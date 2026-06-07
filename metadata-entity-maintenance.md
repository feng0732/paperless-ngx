# Correspondent / DocumentType / Tag 维护逻辑详解

本文档详细解析 paperless-ngx 中三类元数据实体（Correspondent 联系人、DocumentType 文档类型、Tag 标签）的创建、删除、合并，以及与 Document 的反向引用机制。

---

## 1. 模型层：数据结构与继承关系

### 1.1 核心类继承层次

```
ModelWithOwner (抽象)
    └── MatchingModel (抽象)
            ├── Correspondent          # 联系人
            ├── Tag                    # 标签（同时继承 TreeNodeModel）
            ├── DocumentType           # 文档类型
            └── StoragePath            # 存储路径（逻辑相同）
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
  - `clean()` 方法校验：不能自父、成环、超过最大深度

### 1.2 Document 与三类实体的关联关系

```
Document
  ├── correspondent   → ForeignKey(Correspondent, on_delete=SET_NULL, related_name="documents")
  ├── document_type   → ForeignKey(DocumentType, on_delete=SET_NULL, related_name="documents")
  └── tags            → ManyToManyField(Tag, related_name="documents")
```

见 [models.py#L157-L214](file:///d:/fz/0601/solo-dogfeeding/code/64-paperless-ngx/src/documents/models.py#L157-L214)

**关键删除策略：**

- **ForeignKey（correspondent / document_type）**：`on_delete=SET_NULL`，删除实体**不会级联删除 Document**，而是把对应字段置为 NULL
- **ManyToMany（tags）**：无 on_delete 语义，删除 Tag 时 Django 会自动清理中间表记录，但 Document 本身不受影响

---

## 2. 创建逻辑

### 2.1 序列化器层

三个实体都使用 `MatchingModelSerializer + OwnedObjectSerializer` 的组合：

- [serialisers.py#L124-L168](file:///d:/fz/0601/solo-dogfeeding/code/64-paperless-ngx/src/documents/serialisers.py#L124-L168) MatchingModelSerializer：
  - 只读字段：`document_count`（注解）、`slug`
  - `validate()`：校验 `(name, owner)` 唯一性
  - `validate_match()`：正则算法时校验正则表达式合法性

- [serialisers.py#L262-L469](file:///d:/fz/0601/solo-dogfeeding/code/64-paperless-ngx/src/documents/serialisers.py#L262-L469) OwnedObjectSerializer：
  - `create()`：如果请求中未显式指定 owner，默认将 owner 设为当前用户；处理 `set_permissions` 权限设置；再次校验 name/owner 唯一
  - `update()`：权限校验，只有 superuser / owner / unowned 才能修改 owner 或 permissions

- **Tag 额外序列化器** [serialisers.py#L556-L682](file:///d:/fz/0601/solo-dogfeeding/code/64-paperless-ngx/src/documents/serialisers.py#L556-L682)
  - `parent` 字段映射到 `tn_parent`
  - `validate()` 中通过临时修改实例调用 Tag.clean() 校验层级合法性

### 2.2 视图层

三个 ViewSet 均使用标准 DRF ModelViewSet：

- [views.py#L524-L560](file:///d:/fz/0601/solo-dogfeeding/code/64-paperless-ngx/src/documents/views.py#L524-L560) CorrespondentViewSet
- [views.py#L564-L640](file:///d:/fz/0601/solo-dogfeeding/code/64-paperless-ngx/src/documents/views.py#L564-L640) TagViewSet
- [views.py#L643-L660](file:///d:/fz/0601/solo-dogfeeding/code/64-paperless-ngx/src/documents/views.py#L643-L660) DocumentTypeViewSet

都混入 `PermissionsAwareDocumentCountMixin` 自动注解 document_count。

创建流程：`POST /api/{entity}/ → serializer.is_valid() → serializer.save() → serializer.create()`

---

## 3. 删除逻辑

### 3.1 单个删除

标准 `DELETE /api/{entity}/{id}/ → ModelViewSet.destroy()`

因为 Document 外键 `on_delete=SET_NULL`，删除实体不会级联删除 Document，而是把对应字段置空。

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

1. 若 `all=true`，使用 FilterSet 按条件过滤 + 用户权限过滤
2. **Tag 特殊展开**：选中 tag 的**所有可编辑后代**也自动加入删除列表
3. 权限校验（非 superuser）：全局 `delete_xxx` 对象权限 + 每个对象的 owner/授权
4. `objs.delete()` 执行删除

---

## 4. 合并逻辑（深度解析）

"合并"在本系统中有**两种完全不同**的含义，必须严格区分。

### 4.1 权限合并（Permission Merge）

这是 `set_permissions` 操作的 `merge` 布尔标志，作用于**实体自身的 owner 和对象级权限**，与实体合并无关。

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

### 4.2 实体合并（Entity Merge）——没有专用 API，必须手动两步法

系统**不存在**专门的后端 API 用于"将 Correspondent A 和 Correspondent B 合并成一个新实体 C"。所谓"合并多个实体为一个"在前端/用户操作层面是通过**两个独立操作**完成的：

**第一步：迁移 Document 引用（把引用旧实体的文档改成引用目标实体）**
**第二步：删除旧实体**

#### 4.2.1 用户操作完整路径

以"把 Correspondent A 合并到 Correspondent B"为例：

```
用户视角操作流程：
  1. 在管理列表中点击 Correspondent A 旁边的"文档数量"链接
     → 跳转到文档列表页面，自动带上过滤条件 rule_type=3 (correspondent is) value=A.id
  2. 在文档列表中全选
  3. 打开批量编辑器 (Bulk Editor)
  4. 在 Correspondent 下拉中选择 Correspondent B
  5. 确认执行 "set_correspondent"
     → 所有选中文档的 correspondent 字段从 A 改为 B
  6. 回到 Correspondent 管理列表
  7. 选中 Correspondent A，点击删除
     → Correspondent A 被删除（此时已无文档引用它）
```

#### 4.2.2 前端代码路径

- **管理列表跳转文档过滤**：
  [management-list.component.ts#L262-L266](file:///d:/fz/0601/solo-dogfeeding/code/64-paperless-ngx/src-ui/src/app/components/manage/document-attributes/management-list/management-list.component.ts#L262-L266)
  ```typescript
  getDocumentFilterUrl(object: MatchingModel) {
    return this.documentListViewService.getQuickFilterUrl([
      { rule_type: this.filterRuleType, value: object.id.toString() },
    ])
  }
  ```

- **文档列表批量编辑器执行引用迁移**：
  [bulk-editor.component.ts#L553-L588](file:///d:/fz/0601/solo-dogfeeding/code/64-paperless-ngx/src-ui/src/app/components/document-list/bulk-editor/bulk-editor.component.ts#L553-L588)
  `setCorrespondents()` → `executeBulkEditMethod(modal, 'set_correspondent', { correspondent })`

  [bulk-editor.component.ts#L591-L627](file:///d:/fz/0601/solo-dogfeeding/code/64-paperless-ngx/src-ui/src/app/components/document-list/bulk-editor/bulk-editor.component.ts#L591-L627)
  `setDocumentTypes()` → `executeBulkEditMethod(modal, 'set_document_type', { document_type })`

  [bulk-editor.component.ts#L489-L550](file:///d:/fz/0601/solo-dogfeeding/code/64-paperless-ngx/src-ui/src/app/components/document-list/bulk-editor/bulk-editor.component.ts#L489-L550)
  `setTags()` → `executeBulkEditMethod(modal, 'modify_tags', { add_tags, remove_tags })`

#### 4.2.3 后端 Document 引用迁移代码路径

通过 `POST /api/documents/bulk_edit/` 执行：

| 实体类型 | method 名称 | 后端实现函数 | 代码位置 |
|---------|-----------|-----------|---------|
| Correspondent | `set_correspondent` | `bulk_edit.set_correspondent()` | [bulk_edit.py#L111-L131](file:///d:/fz/0601/solo-dogfeeding/code/64-paperless-ngx/src/documents/bulk_edit.py#L111-L131) |
| DocumentType | `set_document_type` | `bulk_edit.set_document_type()` | [bulk_edit.py#L156-L173](file:///d:/fz/0601/solo-dogfeeding/code/64-paperless-ngx/src/documents/bulk_edit.py#L156-L173) |
| Tag | `modify_tags` | `bulk_edit.modify_tags()` | [bulk_edit.py#L226-L284](file:///d:/fz/0601/solo-dogfeeding/code/64-paperless-ngx/src/documents/bulk_edit.py#L226-L284) |

**set_correspondent / set_document_type 核心逻辑：**

```python
def set_correspondent(doc_ids, correspondent):
    if correspondent:
        correspondent = Correspondent.objects.only("pk").get(id=correspondent)

    qs = (
        Document.objects.filter(Q(id__in=doc_ids) & ~Q(correspondent=correspondent))
        .select_related("correspondent")
        .only("pk", "correspondent__id")
    )
    affected_docs = list(qs.values_list("pk", flat=True))
    qs.update(correspondent=correspondent)  # 一条 SQL UPDATE

    bulk_update_documents.apply_async(
        kwargs={"document_ids": affected_docs},
        headers={"trigger_source": PaperlessTask.TriggerSource.SYSTEM},
    )
    return "OK"
```

关键特性：
- 使用 `~Q(correspondent=correspondent)` 过滤掉已经是目标值的文档，避免不必要的 update
- 使用 Django ORM `.update()`，单条 SQL 批量更新
- 最后异步调用 `bulk_update_documents` 重新索引搜索（Whoosh/Elasticsearch）

**modify_tags 核心逻辑（Tag 层级感知）：**

```python
def modify_tags(doc_ids, add_tags, remove_tags):
    # add 时自动展开祖先
    expanded_add_tags: set[int] = set()
    for t in Tag.objects.filter(pk__in=add_tags):
        expanded_add_tags.add(int(t.id))
        expanded_add_tags.update(int(pk) for pk in t.get_ancestors_pks())

    # remove 时自动展开子孙
    expanded_remove_tags: set[int] = set()
    for t in Tag.objects.filter(pk__in=remove_tags):
        expanded_remove_tags.add(int(t.id))
        expanded_remove_tags.update(int(pk) for pk in t.get_descendants_pks())

    with transaction.atomic():
        if expanded_remove_tags:
            DocumentTagRelationship.objects.filter(
                document_id__in=affected_docs,
                tag_id__in=expanded_remove_tags,
            ).delete()

        # 批量 insert 中间表，忽略已存在的行
        to_create = [
            DocumentTagRelationship(document_id=doc, tag_id=tag)
            for doc in affected_docs
            for tag in expanded_add_tags
            if (doc, tag) not in existing_pairs
        ]
        DocumentTagRelationship.objects.bulk_create(to_create, ignore_conflicts=True)
```

#### 4.2.4 引用迁移边界（非常重要）

**Document 批量编辑只迁移 Document 表中的引用**。系统中还有大量其他表引用了这三类实体，这些引用**不会被自动迁移**：

##### A. 会被 Django on_delete=SET_NULL 自动置空的 ForeignKey 引用（删除实体时自动处理）

| 表 | 字段 | on_delete | 说明 |
|---|---|---|---|
| Document | correspondent | SET_NULL | 文档联系人 |
| Document | document_type | SET_NULL | 文档类型 |
| WorkflowTrigger | filter_has_correspondent | SET_NULL | 工作流触发器：匹配此联系人 |
| WorkflowTrigger | filter_has_document_type | SET_NULL | 工作流触发器：匹配此文档类型 |
| WorkflowTrigger | filter_has_storage_path | SET_NULL | 工作流触发器：匹配此存储路径 |
| WorkflowAction | assign_correspondent | SET_NULL | 工作流动作：分配此联系人 |
| WorkflowAction | assign_document_type | SET_NULL | 工作流动作：分配此文档类型 |
| WorkflowAction | assign_storage_path | SET_NULL | 工作流动作：分配此存储路径 |
| PaperlessMailRule | assign_document_type | SET_NULL | 邮件规则：分配此文档类型 |
| PaperlessMailRule | assign_correspondent | SET_NULL | 邮件规则：分配此联系人 |

见：
- Workflow 模型 [models.py#L1345-L1632](file:///d:/fz/0601/solo-dogfeeding/code/64-paperless-ngx/src/documents/models.py#L1345-L1632)
- 邮件规则 [paperless_mail/models.py#L271-L297](file:///d:/fz/0601/solo-dogfeeding/code/64-paperless-ngx/src/paperless_mail/models.py#L271-L297)

##### B. ManyToMany 引用（Django 自动清理中间表行，不做迁移）

| 表 | 字段 | 说明 |
|---|---|---|
| Document | tags | 文档标签 |
| WorkflowTrigger | filter_has_tags / filter_has_all_tags / filter_has_not_tags | 触发器条件：包含/不包含标签 |
| WorkflowTrigger | filter_has_any_document_types / filter_has_not_document_types | 触发器条件：包含/不包含文档类型 |
| WorkflowTrigger | filter_has_any_correspondents / filter_has_not_correspondents | 触发器条件：包含/不包含联系人 |
| WorkflowAction | assign_tags | 工作流动作：分配标签 |
| WorkflowAction | remove_tags / remove_document_types / remove_correspondents | 工作流动作：移除 |
| PaperlessMailRule | assign_tags | 邮件规则：分配标签 |

##### C. 存储为字符串/数字 ID 的引用（完全不会被自动检测或迁移）

| 表 | 字段 | 说明 |
|---|---|---|
| SavedViewFilterRule | `rule_type` + `value` | 保存视图的过滤规则，value 存的是实体 ID 的字符串形式，完全无外键约束 |

见 [models.py#L578-L648](file:///d:/fz/0601/solo-dogfeeding/code/64-paperless-ngx/src/documents/models.py#L578-L648)

RULE_TYPES 中对应实体的枚举：
- `3 = correspondent is`
- `4 = document type is`
- `6 = has tag`
- `7 = has any tag`
- `17 = does not have tag`
- `22 = has tags in`
- `26 = has correspondent in`
- `27 = does not have correspondent in`
- `28 = has document type in`
- `29 = does not have document type in`

##### D. django-guardian 对象级权限表

| 表 | 说明 |
|---|---|
| UserObjectPermission | 用户对实体的对象级权限（通过 content_type + object_pk 关联） |
| GroupObjectPermission | 用户组对实体的对象级权限（通过 content_type + object_pk 关联） |

删除实体时 django-guardian 会级联清理这些行，但实体合并时这些权限**不会自动迁移到目标实体**。

#### 4.2.5 限制条件总结

实体合并操作存在以下限制：

1. **没有原子性保证**：迁移引用和删除旧实体是两个独立的 HTTP 请求，中间可能失败
2. **只迁移 Document 引用**：Workflow、MailRule、SavedView、Guardian 权限等其他引用不会被自动迁移
3. **Tag 合并需要两次 modify_tags 操作**：
   - 第一次：`add_tags = [target_tag_id]`（把目标 Tag 加到所有文档）
   - 第二次：`remove_tags = [source_tag_id]`（把源 Tag 从所有文档移除）
   - 注意：Tag 层级会自动展开祖先/子孙
4. **Correspondent 和 DocumentType 是单值字段**：`set_correspondent` 直接覆盖原值，天然等于"合并"
5. **SavedViewFilterRule 无任何保护**：删除实体后，引用该实体 ID 的保存视图规则会变成"指向不存在实体"的无效规则，不会报错但过滤结果为空

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

### 5.2 从实体反向访问文档

```python
correspondent.documents.all()    # Correspondent → Document
tag.documents.all()              # Tag → Document（M2M）
document_type.documents.all()    # DocumentType → Document
```

这些 `related_name="documents"` 定义在 [models.py#L160-L214](file:///d:/fz/0601/solo-dogfeeding/code/64-paperless-ngx/src/documents/models.py#L160-L214)

### 5.3 Correspondent 额外字段 last_correspondence

CorrespondentViewSet 在 list/retrieve 时额外注解：

```python
.annotate(last_correspondence=Max("documents__created"))
```

见 [views.py#L549-L560](file:///d:/fz/0601/solo-dogfeeding/code/64-paperless-ngx/src/documents/views.py#L549-L560)

---

## 6. Tag 层级特殊逻辑

Tag 因为继承 TreeNodeModel，行为有特殊处理：

### 6.1 添加标签时自动添加所有祖先

- `Document.add_nested_tags()` [models.py#L494-L501](file:///d:/fz/0601/solo-dogfeeding/code/64-paperless-ngx/src/documents/models.py#L494-L501)

```python
def add_nested_tags(self, tags) -> None:
    tag_ids = set()
    for tag in tags:
        tag_ids.add(tag.id)
        tag_ids.update(tag.get_ancestors_pks())
    tags_to_add = self.tags.model.objects.filter(id__in=tag_ids)
    self.tags.add(*tags_to_add)
```

- `bulk_edit.add_tag()` [bulk_edit.py#L176-L202](file:///d:/fz/0601/solo-dogfeeding/code/64-paperless-ngx/src/documents/bulk_edit.py#L176-L202)

```python
def add_tag(doc_ids, tag):
    tag_obj = Tag.objects.get(pk=tag)
    tags_to_add = [tag_obj, *tag_obj.get_ancestors()]
    # 批量创建中间表记录
```

- `DocumentSerializer.update()` [serialisers.py#L1157-L1178](file:///d:/fz/0601/solo-dogfeeding/code/64-paperless-ngx/src/documents/serialisers.py#L1157-L1178)

```python
# 添加子 Tag 时：
final_tags = set(requested_tags)
for t in requested_tags:
    final_tags.update(t.get_ancestors())
```

### 6.2 删除标签时自动删除所有子孙

- `bulk_edit.remove_tag()` [bulk_edit.py#L205-L223](file:///d:/fz/0601/solo-dogfeeding/code/64-paperless-ngx/src/documents/bulk_edit.py#L205-L223)

```python
def remove_tag(doc_ids, tag):
    tag_obj = Tag.objects.get(pk=tag)
    tag_ids = [tag_obj.id, *tag_obj.get_descendants_pks()]
    # 删除中间表记录
```

- `DocumentSerializer.update()` 中：

```python
removed_tags = prev_tags - requested_tags
blocked_tags = set(removed_tags)
for t in removed_tags:
    blocked_tags.update(t.get_descendants())
final_tags.difference_update(blocked_tags)
```

### 6.3 修改 Tag 父节点时同步文档

- `TagViewSet.perform_update()` [views.py#L634-L639](file:///d:/fz/0601/solo-dogfeeding/code/64-paperless-ngx/src/documents/views.py#L634-L639)

```python
def perform_update(self, serializer):
    old_parent = self.get_object().get_parent()
    tag = serializer.save()
    new_parent = tag.get_parent()
    if new_parent and old_parent != new_parent:
        update_document_parent_tags(tag, new_parent)
```

- `tasks.update_document_parent_tags()` [tasks.py#L576-L624](file:///d:/fz/0601/solo-dogfeeding/code/64-paperless-ngx/src/documents/tasks.py#L576-L624)

查找所有包含该 Tag 的 Document，补充新父 Tag 及其所有祖先。

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
| [serialisers.py](file:///d:/fz/0601/solo-dogfeeding/code/64-paperless-ngx/src/documents/serialisers.py) | 序列化器（创建/更新校验） |
| [views.py](file:///d:/fz/0601/solo-dogfeeding/code/64-paperless-ngx/src/documents/views.py) | API 视图（ViewSet + BulkEditObjectsView） |
| [bulk_edit.py](file:///d:/fz/0601/solo-dogfeeding/code/64-paperless-ngx/src/documents/bulk_edit.py) | 文档批量编辑（set_correspondent 等） |
| [tasks.py](file:///d:/fz/0601/solo-dogfeeding/code/64-paperless-ngx/src/documents/tasks.py) | 异步任务（update_document_parent_tags） |
| [matching.py](file:///d:/fz/0601/solo-dogfeeding/code/64-paperless-ngx/src/documents/matching.py) | 自动匹配算法 |
| [paperless_mail/models.py](file:///d:/fz/0601/solo-dogfeeding/code/64-paperless-ngx/src/paperless_mail/models.py) | 邮件规则对实体的引用 |
| [management-list.component.ts](file:///d:/fz/0601/solo-dogfeeding/code/64-paperless-ngx/src-ui/src/app/components/manage/document-attributes/management-list/management-list.component.ts) | 前端管理列表组件 |
| [bulk-editor.component.ts](file:///d:/fz/0601/solo-dogfeeding/code/64-paperless-ngx/src-ui/src/app/components/document-list/bulk-editor/bulk-editor.component.ts) | 前端文档批量编辑器（引用迁移入口） |
| [abstract-name-filter-service.ts](file:///d:/fz/0601/solo-dogfeeding/code/64-paperless-ngx/src-ui/src/app/services/rest/abstract-name-filter-service.ts) | 前端 REST 服务基类 |
