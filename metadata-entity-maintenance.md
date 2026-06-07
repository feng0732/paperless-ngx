# Correspondent / DocumentType / Tag 维护逻辑详解

本文档详细解析 paperless-ngx 中三类元数据实体（Correspondent 联系人、DocumentType 文档类型、Tag 标签）的创建、删除、合并，以及与 Document 的反向引用机制。

---

## 1. 模型层：数据结构与继承关系

### 1.1 核心类继承层次

```
ModelWithOwner (抽象)
    └── MatchingModel (抽象)
            ├── Correspondent           # 联系人
            ├── Tag                   # 标签（同时继承 TreeNodeModel）
            ├── DocumentType          # 文档类型
            └── StoragePath       # 存储路径（逻辑相同）
```

核心代码位于 [models.py](file:///d:/fz/0601/solo-dogfeeding/code/64-paperless-ngx/src/documents/models.py)：

- **ModelWithOwner** [models.py#L32-L43](file:///d:/fz/0601/solo-dogfeeding/code/64-paperless-ngx/src/documents/models.py#L32-L43)
  - 提供 `owner` 外键（User，SET_NULL）

- **MatchingModel** [models.py#L46-L93](file:///d:/fz/0601/solo-dogfeeding/code/64-paperless-ngx/src/documents/models.py#L46-L93)
  - 核心字段：`name`、`match`、`matching_algorithm`、`is_insensitive`
  - 唯一约束：`(name, owner)` 联合唯一；`name` 在 `owner IS NULL` 时全局唯一

- **Correspondent** [models.py#L96-L99](file:///d:/fz/0601/solo-dogfeeding/code/64-paperless-ngx/src/documents/models.py#L96-L99)
  - 最纯粹的 MatchingModel 子类，无额外字段

- **DocumentType** [models.py#L141-L144](file:///d:/fz/0601/solo-dogfeeding/code/64-paperless-ngx/src/documents/models.py#L141-L144)
  - 纯粹的 MatchingModel 子类，无额外字段

- **Tag** [models.py#L102-L138](file:///d:/fz/0601/solo-dogfeeding/code/64-paperless-ngx/src/documents/models.py#L102-L138)
  - 额外字段：`color`、`is_inbox_tag`
  - 同时继承 `TreeNodeModel`（来自 django-treenode），支持**树形层级**
  - `MAX_NESTING_DEPTH = 5`
  - `clean()` 方法校验：不能自父、环、最大深度

### 1.2 Document 与 Document 的关联关系

```
Document
  ├── correspondent   → ForeignKey(Correspondent, on_delete=SET_NULL, related_name="documents")
  ├── document_type   → ForeignKey(DocumentType, on_delete=SET_NULL, related_name="documents")
  └── tags          → ManyToManyField(Tag, related_name="documents")
```

见 [models.py#L157-L214](file:///d:/fz/0601/solo-dogfeeding/code/64-paperless-ngx/src/documents/models.py#L157-L214)

**关键删除策略：

- **ForeignKey（correspondent / document_type）：`on_delete=SET_NULL`——删除实体**不会级联删除 Document，而是把对应字段置为 NULL
- **ManyToMany（tags）：无 on_delete 语义，删除 Tag 时 Django 会自动清理中间表记录

---

## 2. 创建逻辑

### 2.1 序列化器层

三个实体都使用 `MatchingModelSerializer + OwnedObjectSerializer` 的组合：

- [serialisers.py#L124-L168](file:///d:/fz/0601/solo-dogfeeding/code/64-paperless-ngx/src/documents/serialisers.py#L124-L168) MatchingModelSerializer：

  - 只读字段：`document_count`（注解）、`slug`
  - `validate()`：校验 `(name, owner)` 唯一性
  - `validate_match()`：正则算法时校验正则表达式

[serialisers.py#L262-L469](file:///d:/fz/0601/solo-dogfeeding/code/64-paperless-ngx/src/documents/serialisers.py#L262-L469) OwnedObjectSerializer：

- `create()`：
  - 如果请求中未显式指定 owner，默认将 owner 设为当前用户
  - 处理 `set_permissions` 权限设置
  - 再次校验 name/owner 唯一

- `update()`：
  - 权限校验：只有 superuser / owner / unowned 才能修改 owner 或 permissions

Tag 额外序列化器：[serialisers.py#L556-L682](file:///d:/fz/0601/solo-dogfeeding/code/64-paperless-ngx/src/documents/serialisers.py#L556-L682)

- `parent` 字段映射到 `tn_parent`
- `validate()` 中通过临时修改实例调用 Tag.clean() 校验层级合法性

### 2.2 视图层

三个 ViewSet 均使用标准 DRF ModelViewSet：

- [views.py#L524-L560](file:///d:/fz/0601/solo-dogfeeding/code/64-paperless-ngx/src/documents/views.py#L524-L560) CorrespondentViewSet
- [views.py#L564-L640](file:///d:/fz/0601/solo-dogfeeding/code/64-paperless-ngx/src/documents/views.py#L564-L640) TagViewSet
- [views.py#L643-L660](file:///d:/fz/0601/solo-dogfeeding/code/64-paperless-ngx/src/documents/views.py#L643-L660) DocumentTypeViewSet

都混入 `PermissionsAwareDocumentCountMixin` 自动注解 document_count 注解

创建流程：POST /api/{entity}/ → serializer.is_valid() → serializer.save() → serializer.create()

---

## 3. 删除逻辑

### 3.1 单个删除

标准 DELETE /api/{entity}/{id}/ → ModelViewSet.destroy()

因为 Document 外键 on_delete=SET_NULL，删除实体不会级联删除 Document，而是置空。

### 3.2 批量删除

通过 `/api/{entity}/bulk_edit_objects/` 接口：

- 前端：[management-list.component.ts#L449-L484](file:///d:/fz/0601/solo-dogfeeding/code/64-paperless-ngx/src-ui/src/app/components/manage/document-attributes/management-list/management-list.component.ts#L449-L484)
- 后端：[views.py#L4543-L4644](file:///d:/fz/0601/solo-dogfeeding/code/64-paperless-ngx/src/documents/views.py#L4543-L4644) BulkEditObjectsView

```
POST /api/{entity}/bulk_edit_objects/
{
  "object_type": "tags" | "correspondents" | "document_types",
  "operation": "delete",
  "objects": [id1, id2, ...],
  "all": true | false,
  "filters": { ... }
}
```

处理流程：

1. 若 `all=true` 时，使用 FilterSet 按条件过滤 + 用户权限过滤
2. 对于 tags 额外展开：选中 tag 的**所有可编辑后代也加入删除列表
3. 权限校验（非 superuser）：全局 delete_xxx 对象权限 + 每个对象的 owner/授权
4. `objs.delete()` 执行删除

---

## 4. 合并逻辑

"合并"在本系统中有两种不同的含义，注意区分：

### 4.1 权限合并（Permission Merge）

这是 `set_permissions` 操作的 `merge` 标志：

- 后端代码：[views.py#L4609-L4639](file:///d:/fz/0601/solo-dogfeeding/code/64-paperless-ngx/src/documents/views.py#L4609-L4639)

```python
if operation == "set_permissions":
    merge = serializer.validated_data.get("merge")
    if merge:
        # 仅对 owner 为空的对象才更新 owner
        qs_owner_update = qs.filter(owner__isnull=True)
    else:
        qs_owner_update = qs
    qs_owner_update.update(owner=owner)

    for obj in qs:
        set_permissions_for_object(permissions, object=obj, merge=merge)
```

- `merge=False`（默认）：覆盖原有权限和 owner
- `merge=True`：仅填充空值，不覆盖已有值

### 4.2 实体合并（Entity Merge）

系统中不存在**专门的后端 API 用于将 Correspondent/Tag/DocumentType。所谓"合并多个为一个"在前端通过**两步操作**：

1. **文档批量编辑：Document 引用迁移**
2. **删除旧实体**

具体做法：

```
# 第一步：批量修改文档
POST /api/documents/bulk_edit/
{
  "documents": [...],
  "method": "set_correspondent" | "set_document_type" | "modify_tags",
  "parameters": { ... }
}

# 第二步：删除旧实体
DELETE /api/{entity}/bulk_edit_objects/
```

Document 批量编辑的具体实现：[bulk_edit.py#L111-L173](file:///d:/fz/0601/solo-dogfeeding/code/64-paperless-ngx/src/documents/bulk_edit.py#L111-L173)

- `set_correspondent(doc_ids, correspondent)`
  - `set_document_type(doc_ids, document_type)`
  - `modify_tags(doc_ids, add_tags, remove_tags)`

这些函数最后都会调用 `bulk_update_documents.apply_async()` 触发搜索索引异步更新。

---

## 5. 文档反向引用

### 5.1 document_count 注解

通过 `PermissionsAwareDocumentCountMixin` [views.py#L474-L520](file:///d:/fz/0601/solo-dogfeeding/code/64-paperless-ngx/src/documents/views.py#L474-L520)

```python
# 对于 Tag 用 through 表优化
document_count_through = Document.tags.through  # 中间表
document_count_source_field = "tag_id"

# 其它直接 Count
Count("documents", filter=permission_filter)
```

`annotate_document_count_for_related_queryset()` 会结合用户权限过滤后统计。

### 5.2 从实体访问文档：

```python
correspondent.documents.all()    # Correspondent → Document
tag.documents.all()          # Tag → Document（M2M）
document_type.documents.all()  # DocumentType → Document
```

这些 `related_name="documents"` 定义在 [models.py#L160-L214](file:///d:/fz/0601/solo-dogfeeding/code/64-paperless-ngx/src/documents/models.py#L160-L214)

### 5.3 Correspondent 额外字段 last_correspondence

CorrespondentViewSet 在 list/retrieve 时额外注解：

```python
.annotate(last_correspondence=Max("documents__created")
```

见 [views.py#L549-L560](file:///d:/fz/0601/solo-dogfeeding/code/64-paperless-ngx/src/documents/views.py#L549-L560)

---

## 6. Tag 层级特殊逻辑

Tag 因为继承 TreeNodeModel，行为特殊处理：

### 6.1 添加标签时自动添加所有祖先

- Document.add_nested_tags() [models.py#L494-L501](file:///d:/fz/0601/solo-dogfeeding/code/64-paperless-ngx/src/documents/models.py#L494-L501)

```python
def add_nested_tags(self, tags) -> None:
    tag_ids = set()
    for tag in tags:
        tag_ids.add(tag.id)
        tag_ids.update(tag.get_ancestors_pks())
    tags_to_add = self.tags.model.objects.filter(id__in=tag_ids)
    self.tags.add(*tags_to_add)
```

- bulk_edit.add_tag() [bulk_edit.py#L176-L202](file:///d:/fz/0601/solo-dogfeeding/code/64-paperless-ngx/src/documents/bulk_edit.py#L176-L202)

```python
def add_tag(doc_ids, tag):
    tag_obj = Tag.objects.get(pk=tag)
    tags_to_add = [tag_obj, *tag_obj.get_ancestors()]
    # 批量创建中间表记录
```

- DocumentSerializer.update() [serialisers.py#L1157-L1178](file:///d:/fz/0601/solo-dogfeeding/code/64-paperless-ngx/src/documents/serialisers.py#L1157-L1178)

```python
# 添加子 Tag 时：
final_tags = set(requested_tags)
for t in requested_tags:
    final_tags.update(t.get_ancestors())
```

### 6.2 删除标签时自动删除所有子孙

- bulk_edit.remove_tag() [bulk_edit.py#L205-L223](file:///d:/fz/0601/solo-dogfeeding/code/64-paperless-ngx/src/documents/bulk_edit.py#L205-L223)

```python
def remove_tag(doc_ids, tag):
    tag_obj = Tag.objects.get(pk=tag)
    tag_ids = [tag_obj.id, *tag_obj.get_descendants_pks()]
    # 删除中间表记录
```

- DocumentSerializer.update() 中：

```python
removed_tags = prev_tags - requested_tags
blocked_tags = set(removed_tags)
for t in removed_tags:
    blocked_tags.update(t.get_descendants())
final_tags.difference_update(blocked_tags)
```

### 6.3 修改 Tag 父节点时同步文档

- TagViewSet.perform_update() [views.py#L634-L639](file:///d:/fz/0601/solo-dogfeeding/code/64-paperless-ngx/src/documents/views.py#L634-L639)

```python
def perform_update(self, serializer):
    old_parent = self.get_object().get_parent()
    tag = serializer.save()
    new_parent = tag.get_parent()
    if new_parent and old_parent != new_parent:
        update_document_parent_tags(tag, new_parent)
```

- tasks.update_document_parent_tags() [tasks.py#L576-L624](file:///d:/fz/0601/solo-dogfeeding/code/64-paperless-ngx/src/documents/tasks.py#L576-L624)

查找所有包含该 Tag 的 Document，补充新父 Tag 及其所有祖先

---

## 7. 自动匹配（Matching）算法

在文档创建/更新时，系统可根据配置自动分配实体：

[matching.py](file:///d:/fz/0601/solo-dogfeeding/code/64-paperless-ngx/src/documents/matching.py)

支持的匹配算法 [models.py#L47-L63](file:///d:/fz/0601/solo-dogfeeding/code/64-paperless-ngx/src/documents/models.py#L47-L63)：

| 值 | 常量 | 说明 |
|---|---|---|
| 0 | MATCH_NONE | 不匹配 |
| 1 | MATCH_ANY | 任意单词匹配 |
| 2 | MATCH_ALL | 全部单词匹配 |
| 3 | MATCH_LITERAL | 精确字符串匹配 |
| 4 | MATCH_REGEX | 正则表达式匹配 |
| 5 | MATCH_FUZZY | 模糊匹配（rapidfuzz） |
| 6 | MATCH_AUTO | 机器学习分类器自动预测 |

匹配入口函数：

- `match_correspondents()` [matching.py#L47-L76](file:///d:/fz/0601/solo-dogfeeding/code/64-paperless-ngx/src/documents/matching.py#L47-L76)
- `match_document_types()` [matching.py#L79-L107](file:///d:/fz/0601/solo-dogfeeding/code/64-paperless-ngx/src/documents/matching.py#L79-L107)
- `match_tags()` [matching.py#L110-L134](file:///d:/fz/0601/solo-dogfeeding/code/64-paperless-ngx/src/documents/matching.py#L110-L134)

核心匹配函数 `matches()` [matching.py#L169-L263](file:///d:/fz/0601/solo-dogfeeding/code/64-paperless-ngx/src/documents/matching.py#L169-L263) 根据 matching_algorithm 分别处理。

另外 `is_inbox_tag` 的 Tag 在文档消费时自动附加到所有新文档。

---

## 8. 关键文件索引

| 文件 | 作用 |
|---|---|
| [models.py](file:///d:/fz/0601/solo-dogfeeding/code/64-paperless-ngx/src/documents/models.py) | 数据模型定义 |
| [serialisers.py](file:///d:/fz/0601/solo-dogfeeding/code/64-paperless-ngx/src/documents/serialisers.py) | 序列化器（创建/更新校验 |
| [views.py](file:///d:/fz/0601/solo-dogfeeding/code/64-paperless-ngx/src/documents/views.py) | API 视图（ViewSet + BulkEditObjectsView |
| [bulk_edit.py](file:///d:/fz/0601/solo-dogfeeding/code/64-paperless-ngx/src/documents/bulk_edit.py) | 文档批量编辑（set_correspondent等 |
| [tasks.py](file:///d:/fz/0601/solo-dogfeeding/code/64-paperless-ngx/src/documents/tasks.py) | 异步任务（update_document_parent_tags） |
| [matching.py](file:///d:/fz/0601/solo-dogfeeding/code/64-paperless-ngx/src/documents/matching.py) | 自动匹配算法 |
| [management-list.component.ts](file:///d:/fz/0601/solo-dogfeeding/code/64-paperless-ngx/src-ui/src/app/components/manage/document-attributes/management-list/management-list.component.ts) | 前端管理列表组件 |
| [abstract-name-filter-service.ts](file:///d:/fz/0601/solo-dogfeeding/code/64-paperless-ngx/src-ui/src/app/services/rest/abstract-name-filter-service.ts) | 前端 REST 服务基类 |
