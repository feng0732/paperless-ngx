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

**核心代码参考**：
- 基类定义：[ModelWithOwner](file:///d:/fz/0601/solo-dogfeeding/code/28-paperless-ngx/src/documents/models.py#L32-L43)
- 权限基类：[PaperlessObjectPermissions](file:///d:/fz/0601/solo-dogfeeding/code/28-paperless-ngx/src/documents/permissions.py#L28-L51)

### 1.2 所有权判定逻辑

在 `has_object_permission` 中，所有权优先于其他权限检查：

```python
# permissions.py L44-L51
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

**关键实现**：在 [set_permissions_for_object](file:///d:/fz/0601/solo-dogfeeding/code/28-paperless-ngx/src/documents/permissions.py#L93-L163) 中，授予 `change` 权限时会自动追加 `view` 权限：

```python
# permissions.py L131-L137
if action == "change":
    # change gives view too
    assign_perm(f"view_{object.__class__.__name__.lower()}", user, object)
```

### 2.2 用户与组权限的并集

文档可见性是用户权限和组权限的**并集**：

```python
# permissions.py L188-L204
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

**入口点**：[DocumentViewSet](file:///d:/fz/0601/solo-dogfeeding/code/28-paperless-ngx/src/documents/views.py#L943-L1023) 的 `filter_backends` 配置：

```python
filter_backends = (
    DjangoFilterBackend,           # 业务条件过滤
    SearchFilter,                  # 全文搜索
    DocumentsOrderingFilter,       # 排序
    ObjectOwnedOrGrantedPermissionsFilter,  # ⭐ 权限过滤
)
```

**核心过滤器**：[ObjectOwnedOrGrantedPermissionsFilter](file:///d:/fz/0601/solo-dogfeeding/code/28-paperless-ngx/src/documents/filters.py#L950-L961)

```python
def filter_queryset(self, request, queryset, view):
    objects_with_perms = super().filter_queryset(request, queryset, view)  # guardian 权限
    objects_owned = queryset.filter(owner=request.user)                    # 所有者
    objects_unowned = queryset.filter(owner__isnull=True)                  # 无所有者
    return objects_with_perms | objects_owned | objects_unowned
```

### 3.2 第二层：数据库查询优化

为避免复杂的 OR 子句影响性能，系统使用 **子查询 ID 列表** 进行过滤：

**关键函数**：[_permitted_document_ids](file:///d:/fz/0601/solo-dogfeeding/code/28-paperless-ngx/src/documents/permissions.py#L166-L204)

```python
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
- 关联对象计数：[get_document_count_filter_for_user](file:///d:/fz/0601/solo-dogfeeding/code/28-paperless-ngx/src/documents/permissions.py#L207-L220)
- 自定义字段统计：[annotate_document_count_for_related_queryset](file:///d:/fz/0601/solo-dogfeeding/code/28-paperless-ngx/src/documents/permissions.py#L223-L255)

### 3.3 第三层：搜索索引权限过滤

全文搜索（Tantivy）在**索引层**直接应用权限过滤，避免先搜后过滤的性能问题：

**索引时**：[TantivyBackend._build_tantivy_doc](file:///d:/fz/0601/solo-dogfeeding/code/28-paperless-ngx/src/documents/search/_backend.py#L357-L480) 预先存储权限信息

```python
# 索引 owner_id
if document.owner_id:
    doc.add_unsigned("owner_id", document.owner_id)

# 索引所有 viewer_id
users_with_perms = get_users_with_perms(document, only_with_perms_in=["view_document"])
for user in users_with_perms:
    doc.add_unsigned("viewer_id", user.pk)
```

**查询时**：[build_permission_filter](file:///d:/fz/0601/solo-dogfeeding/code/28-paperless-ngx/src/documents/search/_query.py#L413-L442) 构建权限查询

```python
def build_permission_filter(schema, user):
    no_owner = ~exists("owner_id")        # 无所有者
    owned = term("owner_id", user.pk)     # 我拥有的
    shared = term("viewer_id", user.pk)   # 共享给我的
    return disjunction_max([no_owner, owned, shared])
```

**应用点**：
- 主搜索：[search_ids](file:///d:/fz/0601/solo-dogfeeding/code/28-paperless-ngx/src/documents/search/_backend.py#L650-L712)
- 自动补全：[autocomplete](file:///d:/fz/0601/solo-dogfeeding/code/28-paperless-ngx/src/documents/search/_backend.py#L714-L759)
- 相似文档：[more_like_this_ids](file:///d:/fz/0601/solo-dogfeeding/code/28-paperless-ngx/src/documents/search/_backend.py#L761-L812)

### 3.4 第四层：业务条件过滤

[DocumentFilterSet](file:///d:/fz/0601/solo-dogfeeding/code/28-paperless-ngx/src/documents/filters.py#L776-L867) 提供的过滤条件也隐含权限语义：

| 过滤器 | 权限关联 |
|-------|---------|
| `shared_by__id` | 查找某用户共享出去的文档（需先通过权限过滤） |
| `owner__id__none` | 无所有者的公开文档 |
| 自定义字段过滤 | 仅在可见文档范围内生效 |

**特殊过滤**：[SharedByUser](file:///d:/fz/0601/solo-dogfeeding/code/28-paperless-ngx/src/documents/filters.py#L208-L240) 查找"我共享的文档"

---

## 四、结果呈现：序列化时的二次校验

### 4.1 序列化器中的权限感知

即使查询层已过滤，序列化器仍在特定场景做二次权限检查：

**重复文档检测**：[DocumentSerializer.get_duplicate_documents](file:///d:/fz/0601/solo-dogfeeding/code/28-paperless-ngx/src/documents/serialisers.py#L1033-L1041)

```python
def get_duplicate_documents(self, obj):
    request = self.context.get("request")
    user = request.user if request else None
    duplicates = _get_viewable_duplicates(obj, user)  # 只返回可见的重复文档
    return list(duplicates.values("id", "title", "deleted_at"))
```

### 4.2 关联数据的权限一致性

统计数据（如标签文档数、对应方文档数）必须与当前用户的可见范围一致：

```python
# views.py L980-L1015
def _get_selection_data_for_queryset(self, queryset):
    # queryset 已经过权限过滤，统计基于此结果
    correspondents = Correspondent.objects.annotate(
        document_count=Count("documents", filter=Q(documents__in=queryset)),
    )
```

---

## 五、完整请求流程

```
客户端请求
    ↓
[认证层] IsAuthenticated
    ↓
[权限层] PaperlessObjectPermissions.has_permission()  # 模型级权限
    ↓
[过滤层] ObjectOwnedOrGrantedPermissionsFilter        # 查询集裁剪
    ↓
[业务层] DocumentFilterSet + SearchFilter + 排序       # 业务条件
    ↓
[搜索层] Tantivy search + build_permission_filter     # 全文搜索时的权限过滤
    ↓
[序列化层] DocumentSerializer                          # 关联数据权限校验
    ↓
结果返回
```

---

## 六、关键设计决策

### 6.1 为什么使用子查询 ID 列表？

**问题**：直接使用 `Q(owner=user) | Q(owner__isnull=True) | Q(permissions__user=user)` 会产生：
- 复杂的 JOIN + OR 条件
- 潜在的性能问题和重复行
- PostgreSQL 查询规划器可能选择低效执行计划

**解决方案**：[_permitted_document_ids](file:///d:/fz/0601/solo-dogfeeding/code/28-paperless-ngx/src/documents/permissions.py#L166-L204) 返回一个扁平的 ID 列表子查询：
```sql
WHERE id IN (SELECT id FROM documents WHERE ...)
```

### 6.2 为什么搜索层也要做权限过滤？

**性能考量**：
- 先从搜索索引返回所有匹配 → 再在 ORM 层过滤 = 分页不准确 + 性能浪费
- 在 Tantivy 索引中存储 `owner_id` 和 `viewer_id`，搜索时直接做布尔查询 = 高效且分页准确

### 6.3 无所有者文档的设计意图

`owner__isnull=True` 的文档对**所有用户**可见：
- 兼容旧版本（早期 paperless 无多用户）
- 支持"公共文档库"场景
- 简化权限管理（不需要为每个用户单独授权）

---

## 七、权限检查速查表

| 操作 | 检查位置 | 判定逻辑 |
|-----|---------|---------|
| **列表查询** | `ObjectOwnedOrGrantedPermissionsFilter` | `owner=me OR owner IS NULL OR has_perm(view)` |
| **详情查询** | `PaperlessObjectPermissions.has_object_permission` | `is_owner OR has_perm(view)` |
| **全文搜索** | Tantivy `build_permission_filter` | 索引层直接过滤 |
| **创建文档** | Django 模型权限 `add_document` | 无对象级检查 |
| **修改文档** | `has_object_permission` | `is_owner OR has_perm(change)` |
| **删除文档** | `has_object_permission` | `is_owner OR has_perm(delete)` |
| **统计数据** | `_permitted_document_ids` 子查询 | 基于可见文档计数 |

---

## 八、代码路径索引

| 模块 | 核心文件 | 关键函数/类 |
|-----|---------|------------|
| **权限定义** | [permissions.py](file:///d:/fz/0601/solo-dogfeeding/code/28-paperless-ngx/src/documents/permissions.py) | `PaperlessObjectPermissions`, `_permitted_document_ids`, `set_permissions_for_object` |
| **查询过滤** | [filters.py](file:///d:/fz/0601/solo-dogfeeding/code/28-paperless-ngx/src/documents/filters.py) | `ObjectOwnedOrGrantedPermissionsFilter`, `DocumentFilterSet`, `SharedByUser` |
| **搜索过滤** | [_backend.py](file:///d:/fz/0601/solo-dogfeeding/code/28-paperless-ngx/src/documents/search/_backend.py) [_query.py](file:///d:/fz/0601/solo-dogfeeding/code/28-paperless-ngx/src/documents/search/_query.py) | `_apply_permission_filter`, `build_permission_filter` |
| **视图层** | [views.py](file:///d:/fz/0601/solo-dogfeeding/code/28-paperless-ngx/src/documents/views.py) | `DocumentViewSet.filter_backends` |
| **序列化** | [serialisers.py](file:///d:/fz/0601/solo-dogfeeding/code/28-paperless-ngx/src/documents/serialisers.py) | `DocumentSerializer.get_duplicate_documents` |
| **模型层** | [models.py](file:///d:/fz/0601/solo-dogfeeding/code/28-paperless-ngx/src/documents/models.py) | `ModelWithOwner`, `Document.owner` |
