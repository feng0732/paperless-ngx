# Paperless-ngx 文档可见性与权限体系深度解析

## 一、身份边界：权限判定的基础层级

### 1.1 用户身份分层

系统通过三层身份边界确定文档的可见性范围。文档 API 的 `permission_classes` 配置了 `IsAuthenticated`，因此**未认证用户无法访问任何文档接口**，不存在"匿名用户可见无所有者文档"的运行时路径：

```python
# views.py
permission_classes = (IsAuthenticated, PaperlessObjectPermissions)
```

| 身份层级 | 判定条件 | 可见性范围 |
|---------|---------|-----------|
| **超级用户** | `user.is_superuser == True` | 所有文档（无过滤） |
| **文档所有者** | `document.owner == user` | 自有文档 |
| **被授权用户** | 通过 `UserObjectPermission` 或 `GroupObjectPermission` 显式授权 | 被共享的文档（含组权限继承） |
| **普通已认证用户** | 已登录但非超级用户、非所有者、无显式授权 | 无所有者的公开文档 |

普通已认证用户的可见范围是三类文档的并集：自有文档、无所有者文档、通过用户权限或组权限被授权的文档。上表仅列出每种身份独占的可见范围增量，实际可见范围需叠加。

**关于匿名用户的补充说明**：`_permitted_document_ids` 中确实有对匿名用户的处理分支，返回 `owner__isnull=True` 的文档ID，但这仅用于 drf-spectacular 等 Schema 生成场景，并非运行时访问路径：

```python
# permissions.py
def _permitted_document_ids(user):
    if user is None or not getattr(user, "is_authenticated", False):
        # Just Anonymous user e.g. for drf-spectacular
        return base_docs.filter(owner__isnull=True).values_list("id", flat=True)
```

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
        return True  # 无所有者的文档对所有已认证用户公开
```

辅助函数 `has_perms_owner_aware` 封装了同样的三重判定逻辑，用于非 DRF 上下文的权限检查：

```python
# permissions.py
def has_perms_owner_aware(user, perms, obj):
    checker = ObjectPermissionChecker(user)
    return obj.owner is None or obj.owner == user or checker.has_perm(perms, obj)
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

**关键实现**：在权限设置中，授予 `change` 权限时会自动追加 `view` 权限，确保编辑者一定能看到文档。此逻辑对用户授权和组授权均生效：

```python
# permissions.py
if action == "change":
    assign_perm(f"view_{object.__class__.__name__.lower()}", user, object)
    # 组授权路径同样有此逻辑
    assign_perm(f"view_{object.__class__.__name__.lower()}", group, object)
```

### 2.2 用户与组权限的并集

文档可见性是用户直接权限和用户所属组权限的**并集**，只要其中一个路径授权即可访问：

```python
# permissions.py
user_perm_docs = (
    UserObjectPermission.objects.filter(user=user, **perm_filter)
    .annotate(object_pk_int=Cast("object_pk", IntegerField()))
    .values_list("object_pk_int", flat=True)
)
group_perm_docs = (
    GroupObjectPermission.objects.filter(group__user=user, **perm_filter)
    .annotate(object_pk_int=Cast("object_pk", IntegerField()))
    .values_list("object_pk_int", flat=True)
)
permitted_documents = user_perm_docs.union(group_perm_docs)
```

### 2.3 权限设置的合并与覆盖

| merge 参数 | 行为 | 适用场景 |
|-----------|------|---------|
| `False` | **覆盖模式** - 先计算需移除的用户/组（排除待新增的），再追加新的 | 完整权限重设 |
| `True` | **合并模式** - 仅追加新权限，不移除已有授权 | 增量共享 |

覆盖模式的关键细节：先从当前有权限的用户中排除待新增用户，得到"需移除"集合，避免先加后删的冲突：

```python
# permissions.py
users_to_remove = (
    get_users_with_perms(object, only_with_perms_in=[permission], with_group_users=False)
    if not merge
    else User.objects.none()
)
if users_to_add.exists() and users_to_remove.exists():
    users_to_remove = users_to_remove.exclude(id__in=users_to_add)
```

---

## 三、查询过滤：DRF filter_backends 的叠加机制

### 3.1 filter_backends 的执行方式

`DocumentViewSet` 的 `filter_backends` 配置定义了四个过滤后端：

```python
# views.py
filter_backends = (
    DjangoFilterBackend,           # 业务条件过滤（DocumentFilterSet）
    SearchFilter,                  # DRF 全文搜索（实际未使用，搜索走 Tantivy）
    DocumentsOrderingFilter,       # 排序
    ObjectOwnedOrGrantedPermissionsFilter,  # 权限过滤
)
```

DRF 的 filter_backends 是按顺序**依次叠加**的，每个后端接收上一个后端的输出 queryset 并进一步过滤。但需要注意：**顺序不代表"先业务后权限"的优先级关系**，所有过滤条件最终会合并为一条 SQL 查询，各条件之间是 AND 关系。权限过滤放在最后只是为了代码组织清晰，实际执行时权限条件和业务条件同时生效，不存在"先裁剪再过滤"的先后顺序。

### 3.2 权限过滤：ObjectOwnedOrGrantedPermissionsFilter

`ObjectOwnedOrGrantedPermissionsFilter` 实现了三类可见文档的并集查询：

```python
# filters.py
def filter_queryset(self, request, queryset, view):
    objects_with_perms = super().filter_queryset(request, queryset, view)  # guardian 对象权限
    objects_owned = queryset.filter(owner=request.user)                    # 用户拥有的文档
    objects_unowned = queryset.filter(owner__isnull=True)                  # 无所有者的公开文档
    return objects_with_perms | objects_owned | objects_unowned
```

与之对比，`ObjectOwnedPermissionsFilter` 仅保留自有文档和无所有者文档，不包含 guardian 对象权限，用于不需要共享的场景：

```python
# filters.py
class ObjectOwnedPermissionsFilter(ObjectPermissionsFilter):
    def filter_queryset(self, request, queryset, view):
        if request.user.is_superuser:
            return queryset
        objects_owned = queryset.filter(owner=request.user)
        objects_unowned = queryset.filter(owner__isnull=True)
        return objects_owned | objects_unowned
```

### 3.3 业务条件过滤：DocumentFilterSet

`DocumentFilterSet` 定义了所有业务过滤条件，它们在权限过滤的可见范围内进一步缩小结果集：

| 过滤器 | 作用 |
|-------|------|
| `tags__id__all` / `tags__id__none` / `tags__id__in` | 标签过滤 |
| `correspondent__id__none` / `correspondent__id` | 对应方过滤 |
| `document_type__id__none` | 文档类型过滤 |
| `storage_path__id__none` | 存储路径过滤 |
| `custom_field_query` | 自定义字段结构化查询 |
| `shared_by__id` | 查找某用户共享出去的文档（且该文档至少有一个用户/组权限） |
| `owner__id__none` | 无所有者的公开文档 |
| `mime_type` | MIME 类型过滤 |
| 日期范围过滤 | 创建/修改/添加时间过滤 |

`shared_by__id` 过滤器比较特殊：它筛选的是 `owner_id=value` 且至少存在一条 `UserObjectPermission` 或 `GroupObjectPermission` 记录的文档，即"该用户拥有且已共享出去的文档"：

```python
# filters.py
class SharedByUser(Filter):
    def filter(self, qs, value):
        return (
            qs.filter(owner_id=value)
            .annotate(num_shared_users=Count(...))
            .annotate(num_shared_groups=Count(...))
            .filter(Q(num_shared_users__gt=0) | Q(num_shared_groups__gt=0))
            if value is not None
            else qs
        )
```

### 3.4 计数子查询：_permitted_document_ids 的适用场景

`_permitted_document_ids` 返回用户可见文档ID的子查询，专为**关联模型计数**场景设计，不用于 DRF 过滤流水线（那是 `ObjectOwnedOrGrantedPermissionsFilter` 的职责）：

```python
# permissions.py
def _permitted_document_ids(user):
    base_docs = Document.objects.filter(deleted_at__isnull=True).only("id", "owner")

    if user is None or not getattr(user, "is_authenticated", False):
        return base_docs.filter(owner__isnull=True).values_list("id", flat=True)
    if getattr(user, "is_superuser", False):
        return base_docs.values_list("id", flat=True)

    # 用户权限 + 组权限的并集
    permitted_documents = user_perm_docs.union(group_perm_docs)
    return base_docs.filter(
        Q(owner=user) | Q(owner__isnull=True) | Q(id__in=permitted_documents),
    ).values_list("id", flat=True)
```

**适用场景一：关联模型的文档计数过滤**

`get_document_count_filter_for_user` 返回一个 Q 对象，用于标签、对应方等关联模型在序列化时计算当前用户可见的文档数量：

```python
# permissions.py
def get_document_count_filter_for_user(user):
    if getattr(user, "is_superuser", False):
        return Q(documents__deleted_at__isnull=True)
    permitted_ids = _permitted_document_ids(user)
    return Q(documents__id__in=permitted_ids)
```

**适用场景二：多对多中间表的文档计数注解**

`annotate_document_count_for_related_queryset` 用于自定义字段等通过中间表关联的场景，通过 Subquery 注解计算可见文档数：

```python
# permissions.py
def annotate_document_count_for_related_queryset(queryset, through_model, related_object_field, target_field="document_id", user=None):
    permitted_ids = _permitted_document_ids(user)
    counts = (
        through_model.objects.filter(
            **{related_object_field: OuterRef("pk"), f"{target_field}__in": permitted_ids},
        )
        .values(related_object_field)
        .annotate(c=Count(target_field))
        .values("c")
    )
    return queryset.annotate(document_count=Coalesce(Subquery(counts[:1]), 0))
```

**与 DRF 过滤流水线的关系**：`_permitted_document_ids` 和 `ObjectOwnedOrGrantedPermissionsFilter` 是两套独立的权限过滤机制。前者生成 ID 子查询用于关联计数，后者直接过滤 queryset 用于列表查询。两者使用相同的权限判定逻辑（owner + isnull + guardian），但实现路径不同。

---

## 四、搜索结果呈现：双重过滤与结果裁剪

### 4.1 搜索请求的分流机制

`UnifiedSearchViewSet` 继承自 `DocumentViewSet`，在 `list` 方法中判断请求是否包含搜索参数，分流到不同处理路径：

```python
# views.py
class UnifiedSearchViewSet(DocumentViewSet):
    def _is_search_request(self):
        return bool(self._get_active_search_params())

    def list(self, request, *args, **kwargs):
        if not self._is_search_request():
            return super().list(request)  # 普通列表查询走 DRF 标准过滤流水线
        # 全文搜索走 Tantivy + ORM 双重过滤路径
```

**搜索触发条件**：查询参数包含 `text`、`title_search`、`query`、`more_like_id` 任一即为搜索请求。

搜索请求使用 `SearchResultSerializer`，普通请求使用 `DocumentSerializer`：

```python
# views.py
def get_serializer_class(self):
    if self._is_search_request():
        return SearchResultSerializer
    return DocumentSerializer
```

### 4.2 Tantivy 搜索层与 ORM 层的职责划分

搜索结果经过两个独立的过滤系统后取交集。两层的职责不同：

**Tantivy 搜索层**负责：查询匹配 + 权限过滤
- 解析用户查询字符串，在索引中搜索匹配文档
- 应用权限过滤（owner_id / viewer_id），排除不可见文档
- 返回按相关性排序的匹配文档ID列表
- **不处理业务过滤条件**（标签、对应方、日期等不在索引查询中）

```python
# search/_backend.py
def _apply_permission_filter(self, query, user):
    if user is not None:
        permission_filter = build_permission_filter(self._schema, user)
        return tantivy.Query.boolean_query([
            (tantivy.Occur.Must, query),           # 用户查询
            (tantivy.Occur.Must, permission_filter), # 权限过滤
        ])
    return query  # user=None 时跳过权限过滤（超级用户）
```

**ORM 层**负责：权限兜底 + 业务过滤 + 软删除过滤
- 调用 `self.filter_queryset(self.get_queryset())` 一次性执行所有 filter_backends
- `ObjectOwnedOrGrantedPermissionsFilter` 提供权限兜底
- `DjangoFilterBackend` + `DocumentFilterSet` 应用业务条件
- `Document.objects.all()` 默认排除软删除文档（`deleted_at__isnull=True` 在 `_permitted_document_ids` 中，但不在默认 queryset 中；实际上 `Document.objects.all()` 不含软删除过滤，需要 `filtered_qs` 的其他条件来保证）

```python
# views.py
filtered_qs = self.filter_queryset(self.get_queryset())
user = None if request.user.is_superuser else request.user
```

### 4.3 交集裁剪的核心算法

`intersect_and_order` 将 Tantivy 搜索结果与 ORM 可见范围取交集，保留搜索排序顺序：

```python
# views.py
def intersect_and_order(all_ids, filtered_qs, *, use_tantivy_sort):
    if not all_ids:
        return []
    if use_tantivy_sort:
        if len(all_ids) <= _TANTIVY_INTERSECT_THRESHOLD:  # 阈值 = 5000
            # 小结果集：用搜索ID做 IN 查询，精准匹配
            visible_ids = set(
                filtered_qs.filter(pk__in=all_ids).values_list("pk", flat=True),
            )
        else:
            # 大结果集：加载 ORM 全部可见ID，Python 侧交集
            visible_ids = set(
                filtered_qs.values_list("pk", flat=True),
            )
        # 按 Tantivy 排序顺序过滤
        return [doc_id for doc_id in all_ids if doc_id in visible_ids]
    # 非 Tantivy 排序时，让数据库处理排序
    return list(filtered_qs.filter(id__in=all_ids).values_list("pk", flat=True))
```

**两段策略的取舍**：
- 小结果集（≤5000）：用搜索ID列表做 `pk__in` 查询，数据库只需检查这些ID是否在 ORM 可见范围内，避免全表扫描
- 大结果集（>5000）：加载 ORM 全部可见ID到 Python set，再在 Python 侧做交集。这比在 SQLite 中执行大 IN 子句更快
- `use_tantivy_sort=True` 时保留 Tantivy 的搜索排序，`use_tantivy_sort=False` 时让数据库按指定字段排序

### 4.4 超级用户的特殊处理

```python
# views.py
user = None if request.user.is_superuser else request.user
```

当用户是超级用户时，`user=None` 传递给 Tantivy 的 `_apply_permission_filter`，该方法在 `user is not None` 时才应用权限过滤，因此超级用户在搜索层跳过权限过滤。ORM 层的 `ObjectOwnedOrGrantedPermissionsFilter` 也不过滤超级用户。最终效果：超级用户看到全部匹配结果。

### 4.5 相似文档搜索的权限流程

相似文档搜索在双重过滤的基础上，还增加了对参考文档本身的权限检查：

```python
# views.py
def _get_more_like_id(query_params, user):
    more_like_doc_id = int(query_params["more_like_id"])
    more_like_doc = Document.objects.select_related("owner").get(pk=more_like_doc_id)
    # 权限检查：使用 has_perms_owner_aware 验证用户能否查看参考文档
    if user and not has_perms_owner_aware(user, "view_document", more_like_doc):
        raise PermissionDenied
    return more_like_doc_id
```

完整流程：
1. 检查用户是否有权限查看参考文档（`has_perms_owner_aware`）
2. Tantivy 相似搜索（带权限过滤）
3. 与 ORM 可见范围取交集

### 4.6 为什么需要两层过滤？

**Tantivy 搜索层过滤的必要性**：
- 性能：直接在索引中排除不可见文档，避免将大量无权限的文档ID传递到应用层
- 分页准确：搜索结果的分页总数基于权限过滤后的文档集计算，不会出现"总数显示100条但实际只能看到30条"的不一致

**ORM 层过滤的必要性**：
- 业务条件过滤：标签、对应方、日期等业务过滤条件由 `DjangoFilterBackend` + `DocumentFilterSet` 在数据库层处理，Tantivy 索引查询不包含这些条件
- 权限兜底：如果搜索索引的权限数据（`owner_id`、`viewer_id`）与数据库不同步（索引更新延迟），ORM 层仍能保证安全边界
- 软删除排除：`_permitted_document_ids` 中包含 `deleted_at__isnull=True` 条件，确保索引中残留的已删除文档不会出现

---

## 五、结果呈现：分页、序列化与数据一致性

### 5.1 搜索结果桥接：TantivyRelevanceList

`TantivyRelevanceList` 作为搜索结果与 DRF 分页之间的桥梁，它实现了 `__len__` 和 `__getitem__` 接口，使 DRF 分页器能正常工作：

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

### 5.2 分页与高亮生成

在 `run_text_search` 中，交集完成后进行分页切片和高亮生成。高亮仅对当前页文档生成，避免对全部结果生成高亮的性能浪费：

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
```

相似文档搜索（`run_more_like_this`）不生成高亮，直接构造 stub SearchHit：

```python
# views.py
page_hits = [
    SearchHit(id=doc_id, score=0.0, rank=rank, highlights={})
    for rank, doc_id in enumerate(page_ids, start=page_offset + 1)
]
```

### 5.3 序列化器中的权限感知

即使查询层已过滤，序列化器仍在特定场景做二次权限检查，确保关联数据的可见性：

```python
# serialisers.py
def get_duplicate_documents(self, obj):
    request = self.context.get("request")
    user = request.user if request else None
    duplicates = _get_viewable_duplicates(obj, user)  # 过滤不可见的重复文档
    return list(duplicates.values("id", "title", "deleted_at"))
```

### 5.4 关联统计数据的权限一致性

统计数据（如标签文档数、对应方文档数）必须与当前用户的可见范围一致，避免"标签显示有10个文档但用户只能看到3个"的不一致情况。

**普通列表查询**：统计基于权限过滤后的 queryset：

```python
# views.py
def _get_selection_data_for_queryset(self, queryset):
    correspondents = Correspondent.objects.annotate(
        document_count=Count("documents", filter=Q(documents__in=queryset), distinct=True),
    )
    tags = Tag.objects.annotate(
        document_count=Count("documents", filter=Q(documents__in=queryset), distinct=True),
    )
    document_types = DocumentType.objects.annotate(
        document_count=Count("documents", filter=Q(documents__in=queryset), distinct=True),
    )
```

**全文搜索**：统计基于搜索交集后的文档集合，进一步叠加 `pk__in=result.ordered_ids` 条件：

```python
# views.py
response.data["selection_data"] = self._get_selection_data_for_queryset(
    filtered_qs.filter(pk__in=result.ordered_ids),
)
```

注意这里用的是 `filtered_qs.filter(pk__in=result.ordered_ids)` 而非直接用 `result.ordered_ids`，因为 `_get_selection_data_for_queryset` 需要一个 Document queryset 作为 `documents__in` 的参数，而 `filtered_qs` 本身已包含权限过滤和业务过滤条件。

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
[过滤层] DRF filter_backends 依次叠加
  ├─ DjangoFilterBackend: DocumentFilterSet 业务条件
  ├─ SearchFilter: DRF 全文搜索（实际未使用）
  ├─ DocumentsOrderingFilter: 排序
  └─ ObjectOwnedOrGrantedPermissionsFilter: 权限过滤
  → 所有条件合并为一条 SQL，同时生效
    ↓
[分页层] DRF 标准分页
    ↓
[序列化层] DocumentSerializer 序列化结果
    ↓
[统计层] _get_selection_data_for_queryset() 生成关联统计
    ↓
结果返回
```

### 6.2 全文搜索流程

```
客户端请求（带 text/query 搜索参数）
    ↓
[认证层] IsAuthenticated 检查登录状态
    ↓
[分流] _is_search_request() 检测到搜索参数 → 走搜索分支
    ↓
[参数解析] parse_search_params() 提取查询条件、排序方式、分页参数
    ↓
[ORM 可见范围] filter_queryset() 执行所有 filter_backends
  → 得到权限过滤 + 业务过滤后的 queryset
    ↓
[Tantivy 搜索]
  ├─ _parse_query() 解析用户查询
  ├─ _apply_permission_filter() 叠加权限过滤
  │   └─ build_permission_filter() 构建 owner/viewer 条件
  └─ 返回按相关性排序的匹配文档 ID 列表
    ↓
[交集裁剪] intersect_and_order() 取搜索结果与 ORM 可见范围的交集
  → 搜索层过滤权限，ORM层过滤权限+业务条件，交集确保两者都满足
    ↓
[分页切片] 计算当前页文档 ID
    ↓
[高亮生成] highlight_hits() 仅为当前页生成搜索高亮
    ↓
[桥接封装] TantivyRelevanceList 封装搜索结果
    ↓
[DRF 分页] 标准分页处理
    ↓
[序列化] SearchResultSerializer 序列化结果
    ↓
[统计数据] 基于交集结果生成关联统计（可选，需 include_selection_data=true）
    ↓
结果返回（带高亮 + 统计数据）
```

---

## 七、关键设计决策

### 7.1 为什么使用子查询 ID 列表做关联计数？

**问题**：直接使用 `Q(owner=user) | Q(owner__isnull=True) | Q(permissions__user=user)` 产生复杂的 JOIN + OR 条件，导致 PostgreSQL 查询规划器可能选择低效执行计划。

**解决方案**：`_permitted_document_ids` 预计算一个扁平的 ID 列表子查询，调用方通过 `id__in=permitted_ids` 使用：

```sql
WHERE id IN (SELECT id FROM documents WHERE owner_id = X OR owner_id IS NULL OR id IN (...))
```

**适用边界**：此方案专为关联模型计数场景设计（标签文档数、对应方文档数、自定义字段统计），不用于 DRF 的列表查询过滤。列表查询由 `ObjectOwnedOrGrantedPermissionsFilter` 直接处理 queryset。

### 7.2 为什么搜索层也要做权限过滤？

**性能考量**：
- 如果先从搜索索引返回所有匹配，再在 ORM 层过滤，会导致分页不准确和性能浪费
- 在 Tantivy 索引中存储 `owner_id` 和 `viewer_id`，搜索时直接做布尔查询，既高效又能保证分页准确

**安全考量**：
- 双重保险机制，即使搜索索引与数据库权限不同步，ORM 层仍能兜底
- 防止索引权限数据延迟导致的安全边界突破

### 7.3 无所有者文档的设计意图

`owner__isnull=True` 的文档对**所有已认证用户**可见：
- 兼容旧版本（早期 paperless 无多用户功能）
- 支持"公共文档库"使用场景
- 简化权限管理（不需要为每个用户单独授权公开文档）

### 7.4 为什么搜索层和 ORM 层的过滤职责不同？

**搜索层仅做权限过滤**：Tantivy 索引中存储了 `owner_id` 和 `viewer_id` 字段，权限过滤可以在索引层高效完成，但索引中不包含完整的业务过滤语义（标签、对应方、自定义字段查询等仍需数据库处理）。

**ORM 层做权限兜底 + 业务过滤 + 软删除过滤**：
- 权限兜底：防止索引权限数据不同步
- 业务过滤：标签、对应方、日期等复杂过滤条件在数据库中生效
- 软删除：`_permitted_document_ids` 包含 `deleted_at__isnull=True`，确保索引中残留的已删除文档不会出现

交集的结果是：最终返回的文档**同时满足**搜索层权限、ORM层权限、ORM层业务条件三层约束。

---

## 八、权限检查速查表

| 操作 | 检查位置 | 判定逻辑 |
|-----|---------|---------|
| **普通列表** | `ObjectOwnedOrGrantedPermissionsFilter` | `owner=me OR owner IS NULL OR has_perm(view)` |
| **详情查询** | `PaperlessObjectPermissions.has_object_permission` | `is_owner OR has_perm(view) OR owner IS NULL` |
| **全文搜索** | Tantivy `build_permission_filter` + ORM 交集 | 搜索层权限过滤 + ORM层权限+业务过滤 + 交集裁剪 |
| **创建文档** | Django 模型权限 `add_document` | 无对象级检查 |
| **修改文档** | `has_object_permission` | `is_owner OR has_perm(change)` |
| **删除文档** | `has_object_permission` | `is_owner OR has_perm(delete)` |
| **关联计数** | `_permitted_document_ids` 子查询 | `owner=me OR owner IS NULL OR id IN (permitted_ids)` |
| **相似文档** | Tantivy 搜索 + ORM 交集 | 参考文档权限检查(`has_perms_owner_aware`) + 结果交集 |

---

## 九、代码路径索引

| 模块 | 核心文件 | 关键函数/类 |
|-----|---------|------------|
| **权限定义** | permissions.py | `PaperlessObjectPermissions`, `_permitted_document_ids`, `set_permissions_for_object`, `get_document_count_filter_for_user`, `annotate_document_count_for_related_queryset`, `has_perms_owner_aware` |
| **查询过滤** | filters.py | `ObjectOwnedOrGrantedPermissionsFilter`, `ObjectOwnedPermissionsFilter`, `DocumentFilterSet`, `SharedByUser` |
| **搜索过滤** | search/_backend.py, search/_query.py | `_apply_permission_filter`, `build_permission_filter` |
| **搜索交集** | views.py | `intersect_and_order`, `run_text_search`, `run_more_like_this` |
| **结果桥接** | search/_backend.py | `TantivyRelevanceList` |
| **视图层** | views.py | `UnifiedSearchViewSet`, `DocumentViewSet`, `_is_search_request` |
| **序列化** | serialisers.py | `DocumentSerializer.get_duplicate_documents` |
| **模型层** | models.py | `ModelWithOwner`, `Document.owner` |
