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

在 `has_object_permission` 方法中，所有权的检查优先级高于其他权限检查：

```python
# permissions.py
def has_object_permission(self, request, view, obj):
    if hasattr(obj, "owner") and obj.owner is not None:
        if request.user == obj.owner:
            return True  # 文档所有者直接通过校验
        else:
            return super().has_object_permission(request, view, obj)
    else:
        return True  # 无所有者的文档对所有用户公开
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

**关键实现**：在权限设置中，授予 `change` 权限时会自动追加 `view` 权限，确保编辑者一定能看到文档：

```python
# permissions.py
if action == "change":
    assign_perm(f"view_{object.__class__.__name__.lower()}", user, object)
```

### 2.2 用户与组权限的并集

文档可见性是用户直接权限和用户所属组权限的**并集**，只要其中一个路径授权即可访问：

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
| `True` | **合并模式** - 仅追加新权限，不影响已有授权 | 增量共享 |

---

## 三、查询过滤：四层过滤流水线

### 3.1 第一层：视图级过滤（DRF Filter Backend）

**入口点**：`DocumentViewSet` 的 `filter_backends` 配置定义了过滤执行顺序：

```python
# views.py
filter_backends = (
    DjangoFilterBackend,           # 业务条件过滤
    SearchFilter,                  # 全文搜索
    DocumentsOrderingFilter,       # 排序
    ObjectOwnedOrGrantedPermissionsFilter,  # ⭐ 权限过滤
)
```

**核心过滤器**：`ObjectOwnedOrGrantedPermissionsFilter` 实现了三类可见文档的并集查询：

```python
# filters.py
def filter_queryset(self, request, queryset, view):
    objects_with_perms = super().filter_queryset(request, queryset, view)  # guardian 对象权限
    objects_owned = queryset.filter(owner=request.user)                    # 用户拥有的文档
    objects_unowned = queryset.filter(owner__isnull=True)                  # 无所有者的公开文档
    return objects_with_perms | objects_owned | objects_unowned
```

### 3.2 第二层：数据库查询优化

为避免复杂的 JOIN + OR 子句影响性能，系统使用 **子查询 ID 列表** 进行过滤，生成更简洁高效的 SQL：

**关键函数**：`_permitted_document_ids`

```python
# permissions.py
def _permitted_document_ids(user):
    base_docs = Document.objects.filter(deleted_at__isnull=True)
    
    # 超级用户直接返回全部文档ID
    if user.is_superuser:
        return base_docs.values_list("id", flat=True)
    
    # 合并三类可见文档的ID
    return base_docs.filter(
        Q(owner=user) | Q(owner__isnull=True) | Q(id__in=permitted_documents),
    ).values_list("id", flat=True)
```

**使用场景**：
- 关联对象计数（如标签文档数、对应方文档数）
- 自定义字段统计

### 3.3 第三层：搜索索引权限过滤

全文搜索（Tantivy）在**索引层**直接应用权限过滤，避免先搜后过滤导致的分页不准确和性能浪费：

**索引时**：`TantivyBackend._build_tantivy_doc` 预先存储权限信息到索引：

```python
# search/_backend.py
# 索引 owner_id 字段
if document.owner_id:
    doc.add_unsigned("owner_id", document.owner_id)

# 索引所有具有查看权限的用户ID
users_with_perms = get_users_with_perms(document, only_with_perms_in=["view_document"])
for user in users_with_perms:
    doc.add_unsigned("viewer_id", user.pk)
```

**查询时**：`build_permission_filter` 构建权限查询，在索引层直接过滤：

```python
# search/_query.py
def build_permission_filter(schema, user):
    no_owner = ~exists("owner_id")        # 无所有者的公开文档
    owned = term("owner_id", user.pk)     # 当前用户拥有的文档
    shared = term("viewer_id", user.pk)   # 共享给当前用户的文档
    return disjunction_max([no_owner, owned, shared])
```

### 3.4 第四层：业务条件过滤

`DocumentFilterSet` 提供的过滤条件也隐含权限语义，它们都在权限过滤之后的可见范围内生效：

| 过滤器 | 权限关联 |
|-------|---------|
| `shared_by__id` | 查找某用户共享出去的文档（需先通过权限过滤） |
| `owner__id__none` | 无所有者的公开文档 |
| 自定义字段过滤 | 仅在可见文档范围内生效 |

---

## 四、搜索结果呈现：双重过滤与结果裁剪

### 4.1 搜索请求的分流机制

请求进入 `list` 方法后，首先判断是否为搜索请求，然后分流到不同的处理路径：

```python
# views.py
def list(self, request, *args, **kwargs):
    if not self._is_search_request():
        return super().list(request)  # 普通列表查询走 DRF 标准过滤流水线
    
    # 全文搜索走 Tantivy + ORM 双重过滤路径
```

**搜索触发条件**：查询参数包含 `text`、`title_search`、`query`、`more_like_id` 任一即为搜索请求。

### 4.2 双重过滤的执行流程

搜索结果经过**两次独立的权限过滤**后取交集，确保可见性边界不被突破：

**第一次过滤在搜索层执行**：
- 调用 `backend.search_ids(query, user=user)` 触发 Tantivy 搜索
- 利用索引中预存的 `owner_id` 和 `viewer_id` 字段
- 在搜索索引中直接排除不可见文档
- 返回按相关性排序的匹配文档ID列表

**第二次过滤在 ORM 层执行**：
- 调用 `self.filter_queryset(self.get_queryset())`
- 经过 `ObjectOwnedOrGrantedPermissionsFilter` 过滤器
- 得到数据库层面的可见文档查询集

**两次结果取交集**，确保最终返回的文档同时满足搜索索引和数据库的权限判定：

```python
# views.py
# 步骤1：Tantivy 搜索（已应用搜索层权限过滤）
all_ids = backend.search_ids(
    query_str,
    user=user,  # 非超级用户在此处应用权限过滤
    ...
)

# 步骤2：ORM 查询集（已通过 DRF 权限过滤）
filtered_qs = self.filter_queryset(self.get_queryset())

# 步骤3：取交集，保留搜索排序
ordered_ids = intersect_and_order(all_ids, filtered_qs, ...)
```

### 4.3 交集裁剪的核心算法

`intersect_and_order` 是搜索权限边界的关键实现，它负责将搜索结果与 ORM 可见范围取交集，同时保留搜索排序顺序：

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
        if len(all_ids) <= _TANTIVY_INTERSECT_THRESHOLD:  # 阈值 = 5000
            # 小结果集：使用针对性 IN 查询避免全表扫描
            visible_ids = set(
                filtered_qs.filter(pk__in=all_ids).values_list("pk", flat=True),
            )
        else:
            # 大结果集：全表扫描 + Python 交集（SQLite 优化）
            visible_ids = set(
                filtered_qs.values_list("pk", flat=True),
            )
        # 按搜索顺序过滤，保留原始排序
        return [doc_id for doc_id in all_ids if doc_id in visible_ids]
    # 非搜索排序时，直接让数据库处理排序
    return list(
        filtered_qs.filter(id__in=all_ids).values_list("pk", flat=True),
    )
```

**性能优化策略**：
- 小结果集（≤5000）使用 `pk__in` 查询，避免全表扫描
- 大结果集（>5000）使用 Python set 交集，避免 SQLite 大 IN 子句性能下降
- 始终保留 Tantivy 的搜索排序顺序

### 4.4 超级用户的特殊处理

```python
# views.py
user = None if request.user.is_superuser else request.user
```

当用户是超级用户时，`user=None` 传递给 Tantivy，跳过搜索层权限过滤。虽然 ORM 层仍然执行过滤，但超级用户在 ORM 层也不会被过滤，因此实际效果是返回全部匹配结果。

### 4.5 相似文档搜索的权限流程

相似文档搜索在双重过滤的基础上，还增加了对参考文档本身的权限检查：

```python
# views.py
def run_more_like_this(backend, user, filtered_qs):
    # 1. 首先检查用户是否有权限查看参考文档
    more_like_doc_id = _get_more_like_id(request.query_params, user)
    
    # 2. Tantivy 相似搜索（带权限过滤）
    all_ids = backend.more_like_this_ids(more_like_doc_id, user=user)
    
    # 3. 与 ORM 可见范围取交集
    ordered_ids = intersect_and_order(all_ids, filtered_qs, use_tantivy_sort=True)
```

---

## 五、结果呈现：分页、序列化与数据一致性

### 5.1 搜索结果桥接：TantivyRelevanceList

`TantivyRelevanceList` 作为搜索结果与 DRF 序列化之间的桥梁，它实现了 `__len__` 和 `__getitem__` 接口，使 DRF 分页器能够正常工作：

```python
# search/_backend.py
class TantivyRelevanceList:
    def __init__(self, ordered_ids, page_hits, page_offset):
        self._ordered_ids = ordered_ids    # 完整匹配ID列表，用于计数
        self._page_hits = page_hits        # 当前页的搜索命中（带高亮信息）
        self._page_offset = page_offset    # 当前页在全结果中的偏移量

    def __len__(self):
        return len(self._ordered_ids)      # 用于分页总计数

    def __getitem__(self, key):
        # 根据分页切片返回 SearchHit 对象
        # 当前页返回完整高亮数据，其他页返回 stub 数据
```

**关键特性**：
- 支持 DRF 标准分页协议
- 仅当前页文档有完整高亮信息
- 其他页返回 stub 数据（无高亮）以节省性能

### 5.2 分页与高亮生成

在 `run_text_search` 中，交集完成后进行分页切片和高亮生成：

```python
# views.py
# 计算分页偏移
page_offset = (page_num - 1) * page_size
page_ids = ordered_ids[page_offset : page_offset + page_size]

# 仅为当前页生成高亮
page_hits = backend.highlight_hits(
    query_str,
    page_ids,
    search_mode=search_mode,
    rank_start=page_offset + 1,
)

# 构造结果页
return SearchResultPage(
    ordered_ids=ordered_ids,
    hits=page_hits,
    page_offset=page_offset,
)
```

### 5.3 序列化器中的权限感知

即使查询层已过滤，序列化器仍在特定场景做二次权限检查，确保关联数据的可见性：

**重复文档检测**：只返回用户可见的重复文档：

```python
# serialisers.py
def get_duplicate_documents(self, obj):
    request = self.context.get("request")
    user = request.user if request else None
    duplicates = _get_viewable_duplicates(obj, user)  # 过滤不可见的重复文档
    return list(duplicates.values("id", "title", "deleted_at"))
```

### 5.4 关联统计数据的权限一致性

统计数据（如标签文档数、对应方文档数）必须与当前用户的可见范围一致，避免出现"标签显示有10个文档但用户只能看到3个"的不一致情况：

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
# 搜索结果的统计数据基于搜索交集后的文档集合
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
[认证层] IsAuthenticated 检查登录状态
    ↓
[权限层] PaperlessObjectPermissions.has_permission() 检查模型级权限
    ↓
[过滤层] ObjectOwnedOrGrantedPermissionsFilter 执行 ORM 查询集裁剪
    ↓
[业务层] DocumentFilterSet + 排序 应用业务条件过滤
    ↓
[分页层] DRF 标准分页
    ↓
[序列化层] DocumentSerializer 序列化结果
    ↓
结果返回
```

### 6.2 全文搜索流程

```
客户端请求（带 text/query 搜索参数
    ↓
[认证层] IsAuthenticated 检查登录状态
    ↓
[分流] 检测到搜索参数 → 走搜索分支
    ↓
[参数解析] 提取查询条件、排序方式、分页参数
    ↓
[ORM 可见范围] 调用 filter_queryset() 获取数据库层面可见文档
    ↓
[Tantivy 搜索]
  ├─ 解析用户查询
  ├─ 构建权限过滤条件
  ├─ 索引层权限过滤
  └─ 返回按相关性排序的匹配文档 ID 列表
    ↓
[交集裁剪] intersect_and_order() 取搜索结果与 ORM 可见范围的交集
    ↓
[分页切片] 计算当前页文档 ID
    ↓
[高亮生成] 为当前页文档生成搜索高亮
    ↓
[桥接封装] TantivyRelevanceList 封装搜索结果
    ↓
[DRF 分页] 标准分页处理
    ↓
[序列化] SearchResultSerializer 序列化结果
    ↓
[统计数据] 基于交集结果生成关联统计（可选）
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

**解决方案**：返回一个扁平的 ID 列表子查询，生成更简洁高效的 SQL：
```sql
WHERE id IN (SELECT id FROM documents WHERE ...)
```

### 7.2 为什么搜索层也要做权限过滤？

**性能考量**：
- 如果先从搜索索引返回所有匹配，再在 ORM 层过滤，会导致分页不准确和性能浪费
- 在 Tantivy 索引中存储 `owner_id` 和 `viewer_id`，搜索时直接做布尔查询，既高效又能保证分页准确

**安全考量**：
- 双重保险机制，即使搜索索引与数据库权限不同步，ORM 层仍能兜底
- 防止索引权限数据延迟导致的安全边界突破

### 7.3 无所有者文档的设计意图

`owner__isnull=True` 的文档对**所有用户**可见：
- 兼容旧版本（早期 paperless 无多用户功能）
- 支持"公共文档库"使用场景
- 简化权限管理（不需要为每个用户单独授权公开文档）

### 7.4 为什么需要两次权限过滤？

**搜索层过滤的优势**：
- 性能：直接在索引中排除不可见文档，避免大量无效ID传递
- 分页准确：分页总数基于可见文档集计算，不会出现"某页全被过滤掉"的情况

**ORM层过滤的必要性**：
- 安全兜底：搜索索引可能存在权限同步延迟
- 软删除过滤：索引可能包含已删除文档，ORM 层会排除
- 业务过滤：标签、对应方、日期等复杂过滤条件在数据库中生效

---

## 八、权限检查速查表

| 操作 | 检查位置 | 判定逻辑 |
|-----|---------|---------|
| **普通列表** | `ObjectOwnedOrGrantedPermissionsFilter` | `owner=me OR owner IS NULL OR has_perm(view)` |
| **详情查询** | `PaperlessObjectPermissions.has_object_permission` | `is_owner OR has_perm(view)` |
| **全文搜索** | Tantivy `build_permission_filter` + ORM 交集 | 双重过滤 + 交集裁剪 |
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
