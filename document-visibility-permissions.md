# Paperless-ngx 文档可见性与权限体系深度解析

## 一、身份边界：权限判定的基础层级

### 1.1 用户身份分层

系统通过四层身份边界来确定文档的可见性范围：

| 身份层级 | 判定条件 | 可见性范围 |
|---------|---------|-----------|
| **超级用户** | `user.is_superuser == True` | 所有文档（无过滤） |
| **文档所有者** | `document.owner == user` | 拥有的所有文档 |
| **被授权用户** | 通过 `UserObjectPermission`/`GroupObjectPermission` 显式授权 | 被共享的文档 |
| **普通用户** | 已认证但无特殊权限 | 仅无所有者文档 + 被授权文档 |
| **匿名用户** | 未认证 | 仅无所有者文档（`owner__isnull=True`） |

### 1.2 所有权判定逻辑

在 `has_object_permission` 中，所有权优先于其他权限检查：

```python
# permissions.py
def has_object_permission(self, request, view, obj):
    if hasattr(obj, "owner") and obj.owner is not None:
        if request.user == obj.owner:
            return True  # 所有者直接通过
        else:
            return super().has_object_permission(request, view, obj)
    else:
        return True  # 无所有者 = 公开访问
```

---

## 二、共享规则：权限的精细化控制

### 2.1 权限类型与继承关系

系统支持三种对象级权限，存在隐式继承关系：

| 权限代码 | 授权范围 | 隐式包含 |
|---------|---------|---------|
| `view_document` | 仅查看 | - |
| `change_document` | 编辑文档 | ✓ `view_document` |
| `delete_document` | 删除文档 | - |

**关键实现**：在权限设置中，授予 `change` 权限时会自动追加 `view` 权限：

```python
# permissions.py
if action == "change":
    # change gives view too
    assign_perm(f"view_{object.__class__.__name__.lower()}", user, object)
```

### 2.2 用户与组权限的并集

文档可见性是用户权限和组权限的**并集**：

```python
# permissions.py
user_perm_docs = UserObjectPermission.objects.filter(user=user, **perm_filter)
group_perm_docs = GroupObjectPermission.objects.filter(group__user=user, **perm_filter)
permitted_documents = user_perm_docs.union(group_perm_docs)
```

### 2.3 权限设置的合并与覆盖

| merge 参数 | 行为 | 适用场景 |
|-----------|------|---------|
| `False` | **覆盖模式** - 移除不在列表中的所有用户/组 | 完整权限重设 |
| `True` | **合并模式** - 仅追加新权限 | 增量共享 |

---

## 三、查询过滤：四层过滤流水线

### 3.1 第一层：视图级过滤（DRF Filter Backend）

**入口点**：`DocumentViewSet` 的 `filter_backends` 配置：

```python
# views.py
filter_backends = (
    DjangoFilterBackend,           # 业务条件过滤
    SearchFilter,                  # 全文搜索
    DocumentsOrderingFilter,       # 排序
    ObjectOwnedOrGrantedPermissionsFilter,  # ⭐ 权限过滤
)
```

**核心过滤器**：`ObjectOwnedOrGrantedPermissionsFilter`

```python
# filters.py
def filter_queryset(self, request, queryset, view):
    objects_with_perms = super().filter_queryset(request, queryset, view)  # guardian 权限
    objects_owned = queryset.filter(owner=request.user)                    # 所有者
    objects_unowned = queryset.filter(owner__isnull=True)                  # 无所有者
    return objects_with_perms | objects_owned | objects_unowned
```

### 3.2 第二层：数据库查询优化

为避免复杂的 OR 子句影响性能，系统使用 **子查询 ID 列表** 进行过滤：

**关键函数**：`_permitted_document_ids`

```python
# permissions.py
def _permitted_document_ids(user):
    base_docs = Document.objects.filter(deleted_at__isnull=True)
    
    # 超级用户: 全部返回
    if user.is_superuser:
        return base_docs.values_list("id", flat=True)
    
    # 合并三类可见文档的ID
    return base_docs.filter(
        Q(owner=user) | Q(owner__isnull=True) | Q(id__in=permitted_documents),
    ).values_list("id", flat=True)
```

**使用场景**：
- 关联对象计数
- 自定义字段统计

### 3.3 第三层：搜索索引权限过滤

全文搜索（Tantivy）在**索引层**直接应用权限过滤，避免先搜后过滤的性能问题：

**索引时**：`TantivyBackend._build_tantivy_doc` 预先存储权限信息

```python
# search/_backend.py
# 索引 owner_id
if document.owner_id:
    doc.add_unsigned("owner_id", document.owner_id)

# 索引所有 viewer_id
users_with_perms = get_users_with_perms(document, only_with_perms_in=["view_document"])
for user in users_with_perms:
    doc.add_unsigned("viewer_id", user.pk)
```

**查询时**：`build_permission_filter` 构建权限查询

```python
# search/_query.py
def build_permission_filter(schema, user):
    no_owner = ~exists("owner_id")        # 无所有者
    owned = term("owner_id", user.pk)     # 我拥有的
    shared = term("viewer_id", user.pk)   # 共享给我的
    return disjunction_max([no_owner, owned, shared])
```

### 3.4 第四层：业务条件过滤

`DocumentFilterSet` 提供的过滤条件也隐含权限语义：

| 过滤器 | 权限关联 |
|-------|---------|
| `shared_by__id` | 查找某用户共享出去的文档（需先通过权限过滤） |
| `owner__id__none` | 无所有者的公开文档 |
| 自定义字段过滤 | 仅在可见文档范围内生效 |

---

## 四、搜索结果呈现：双重过滤与交集裁剪

### 4.1 搜索请求的分流机制

**搜索请求与普通列表查询在 `list` 方法中分流：

```python
# views.py
def list(self, request, *args, **kwargs):
    if not self._is_search_request():
        return super().list(request)  # 普通列表查询走 DRF 过滤流水线
    
    # 全文搜索走 Tantivy + ORM 双重过滤
```

**搜索触发条件**：查询参数包含 `text`/`title_search`/`query`/`more_like_id` 任一即为搜索请求

### 4.2 搜索权限的双重保险机制

搜索结果经过**两次权限过滤**，确保可见性边界不被突破：

**第一次过滤（搜索层）：
- Tantivy 索引层权限过滤
- 调用 `backend.search_ids(query, user=user)`
- 利用索引中 `owner_id` 和 `viewer_id` 字段
- 直接在搜索索引中排除不可见文档

**第二次过滤（ORM层）：
- `filtered_qs = self.filter_queryset(self.get_queryset())`
- 经过 `ObjectOwnedOrGrantedPermissionsFilter`
- 确保搜索结果与数据库权限结果取交集

```python
# views.py
# 步骤1: Tantivy 搜索（已带权限过滤）
all_ids = backend.search_ids(
    query_str,
    user=user,  # 这里 user 不是超级用户时应用权限过滤
    ...
)

# 步骤2: ORM 查询集（已通过 DRF 权限过滤
filtered_qs = self.filter_queryset(self.get_queryset())

# 步骤3: 取交集
ordered_ids = intersect_and_order(all_ids, filtered_qs, ...)
```

### 4.3 交集裁剪的核心算法

`intersect_and_order` 是搜索权限边界的关键实现：

```python
# views.py
def intersect_and_order(
    all_ids: list[int],
    filtered_qs: QuerySet[Document],
    *,
    use_tantivy_sort: bool,
) -> list[int]:
    if not all_ids:
        return []
    if use_tantivy_sort:
        if len(all_ids) <= 5000:  # _TANTIVY_INTERSECT_THRESHOLD
            # 小结果集：针对性 IN 查询
            visible_ids = set(
                filtered_qs.filter(pk__in=all_ids).values_list("pk", flat=True),
            )
        else:
            # 大结果集：全表扫描 + Python 交集
            visible_ids = set(
                filtered_qs.values_list("pk", flat=True),
            )
        return [doc_id for doc_id in all_ids if doc_id in visible_ids]
```

**性能优化策略**：
- **小结果集（≤5000）使用 `pk__in` 查询避免全表扫描
- 大结果集（>5000）使用 Python set 交集（SQLite 优化）
- 始终保留 Tantivy 的排序顺序

### 4.4 超级用户的特殊处理

```python
# views.py
user = None if request.user.is_superuser else request.user
```

当用户是超级用户时，`user=None` 传递给 Tantivy，跳过搜索层权限过滤，但仍会经过 ORM 层过滤（虽然超级用户在 ORM 层也不过滤）。

### 4.5 相似文档搜索的权限流程

```python
# views.py
def run_more_like_this(backend, user, filtered_qs):
    # 1. 权限检查：确保用户能看到参考文档
    more_like_doc_id = _get_more_like_id(query_params, user)
    
    # 2. Tantivy 相似搜索（带权限过滤
    all_ids = backend.more_like_this_ids(more_like_doc_id, user=user)
    
    # 3. ORM 交集裁剪
    ordered_ids = intersect_and_order(all_ids, filtered_qs, ...)
```

---

## 五、结果呈现：序列化与数据一致性

### 5.1 搜索结果的桥接：TantivyRelevanceList

`TantivyRelevanceList` 作为搜索结果与 DRF 序列化之间的桥梁：

```python
# search/_backend.py
class TantivyRelevanceList:
    def __init__(self, ordered_ids, page_hits, page_offset):
        self._ordered_ids = ordered_ids  # 完整匹配ID列表
        self._page_hits = page_hits    # 当前页的搜索命中（带高亮）
        self._page_offset = page_offset  # 当前页偏移量

    def __len__(self):
        return len(self._ordered_ids)  # 用于分页计数

    def __getitem__(self, key):
        # 根据分页切片返回 SearchHit 对象
```

**关键特性**：
- 支持 DRF 分页兼容
- 仅当前页有完整高亮信息
- 其他页返回 stub 数据（无高亮）

### 5.2 序列化器中的权限感知

即使查询层已过滤，序列化器仍在特定场景做二次权限检查：

**重复文档检测**：只返回用户可见的重复文档

```python
# serialisers.py
def get_duplicate_documents(self, obj):
    request = self.context.get("request")
    user = request.user if request else None
    duplicates = _get_viewable_duplicates(obj, user)  # 只返回可见的重复文档
    return list(duplicates.values("id", "title", "deleted_at"))
```

### 5.3 关联统计数据的权限一致性

统计数据（如标签文档数、对应方文档数）必须与当前用户的可见范围一致：

```python
# views.py
def _get_selection_data_for_queryset(self, queryset):
    # queryset 已经过权限过滤，统计基于此结果
    correspondents = Correspondent.objects.annotate(
        document_count=Count("documents", filter=Q(documents__in=queryset)),
    )
```

**搜索时统计数据生成**：

```python
# views.py
# 搜索结果统计基于搜索交集后的文档集合
response.data["selection_data"] = self._get_selection_data_for_queryset(
    filtered_qs.filter(pk__in=result.ordered_ids),
)
```

---

## 六、完整请求流程

### 6.1 普通列表查询流程

```
客户端请求（无搜索参数）
    ↓
[认证层] IsAuthenticated
    ↓
[权限层] PaperlessObjectPermissions.has_permission()  # 模型级权限
    ↓
[过滤层] ObjectOwnedOrGrantedPermissionsFilter        # ORM 查询集裁剪
    ↓
[业务层] DocumentFilterSet + 排序       # 业务条件过滤
    ↓
[分页层] DRF 分页
    ↓
[序列化层] DocumentSerializer
    ↓
结果返回
```

### 6.2 全文搜索流程

```
客户端请求（带 text/query 参数
    ↓
[认证层] IsAuthenticated
    ↓
[分流] 检测到搜索参数 → 走搜索分支
    ↓
[Tantivy 搜索]
  ├─ 解析查询 + 构建权限过滤
  ├─ 索引层权限过滤
  └─ 返回匹配文档 ID 列表（按相关性排序）
    ↓
[ORM 权限过滤] filter_queryset()
    ↓
[交集裁剪] intersect_and_order()
    ↓
[分页] DRF 分页（基于 TantivyRelevanceList
    ↓
[高亮] 当前页文档高亮生成
    ↓
[序列化] SearchResultSerializer
    ↓
结果返回（带高亮 + 统计数据）
```

---

## 七、关键设计决策

### 7.1 为什么使用子查询 ID 列表？

**问题**：直接使用 `Q(owner=user) | Q(owner__isnull=True) | Q(permissions__user=user)` 会产生：
- 复杂的 JOIN + OR 条件
- 潜在的性能问题和重复行
- PostgreSQL 查询规划器可能选择低效执行计划

**解决方案**：返回一个扁平的 ID 列表子查询：
```sql
WHERE id IN (SELECT id FROM documents WHERE ...)
```

### 7.2 为什么搜索层也要做权限过滤？

**性能考量**：
- 先从搜索索引返回所有匹配 → 再在 ORM 层过滤 = 分页不准确 + 性能浪费
- 在 Tantivy 索引中存储 `owner_id` 和 `viewer_id`，搜索时直接做布尔查询 = 高效且分页准确

**安全考量**：
- 双重保险：即使搜索层和 ORM 层双重过滤
- 防止索引与数据库权限不一致时的安全边界

### 7.3 无所有者文档的设计意图

`owner__isnull=True` 的文档对**所有用户**可见：
- 兼容旧版本（早期 paperless 无多用户）
- 支持"公共文档库"场景
- 简化权限管理（不需要为每个用户单独授权）

### 7.4 为什么需要两次权限过滤？

**搜索层过滤的优势**：
- 性能：直接在索引中排除，避免大量无效ID传递
- 分页准确：分页基于可见文档集进行

**ORM层过滤的必要性**：
- 安全兜底：索引可能存在权限同步延迟
- 软删除：索引可能包含已删除文档？
- 复杂过滤：业务过滤条件在数据库中生效（如标签、对应方等）

---

## 八、权限检查速查表

| 操作 | 检查位置 | 判定逻辑 |
|-----|---------|---------|
| **普通列表** | `ObjectOwnedOrGrantedPermissionsFilter` | `owner=me OR owner IS NULL OR has_perm(view)` |
| **详情查询** | `PaperlessObjectPermissions.has_object_permission` | `is_owner OR has_perm(view)` |
| **全文搜索** | Tantivy `build_permission_filter` + ORM 交集 | 双重过滤 |
| **创建文档** | Django 模型权限 `add_document` | 无对象级检查 |
| **修改文档** | `has_object_permission` | `is_owner OR has_perm(change)` |
| **删除文档** | `has_object_permission` | `is_owner OR has_perm(delete)` |
| **统计数据** | 基于过滤后的查询集 | 基于可见文档计数 |
| **相似文档** | Tantivy 搜索 + ORM 交集 | 参考文档权限检查 + 结果交集 |

---

## 九、代码路径索引

| 模块 | 核心文件 | 关键函数/类 |
|-----|---------|------------|
| **权限定义** | permissions.py | `PaperlessObjectPermissions`, `_permitted_document_ids`, `set_permissions_for_object` |
| **查询过滤** | filters.py | `ObjectOwnedOrGrantedPermissionsFilter`, `DocumentFilterSet` |
| **搜索过滤** | search/_backend.py, search/_query.py | `_apply_permission_filter`, `build_permission_filter` |
| **搜索交集** | views.py | `intersect_and_order`, `run_text_search`, `run_more_like_this` |
| **结果桥接** | search/_backend.py | `TantivyRelevanceList` |
| **视图层** | views.py | `DocumentViewSet.filter_backends`, `UnifiedSearchViewSet.list` |
| **序列化** | serialisers.py | `DocumentSerializer.get_duplicate_documents` |
| **模型层** | models.py | `ModelWithOwner`, `Document.owner` |
