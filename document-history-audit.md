# 文档变更历史与审计日志代码实现分析

## 一、整体架构概述

Paperless-ngx 的文档变更历史基于第三方库 **django-auditlog** 实现，结合自定义手动日志记录构成完整的审计体系。

### 1.1 核心依赖与启用方式

审计日志功能通过 `AUDIT_LOG_ENABLED` 配置开关控制，在 [settings/__init__.py](file:///d:/fz/0601/solo-dogfeeding/code/70-paperless-ngx/src/paperless/settings/__init__.py#L1004-L1005) 中动态加载：

```python
if settings.AUDIT_LOG_ENABLED:
    INSTALLED_APPS.append("auditlog")
    MIDDLEWARE.append("auditlog.middleware.AuditlogMiddleware")
```

`AuditlogMiddleware` 负责在 HTTP 请求上下文中自动关联操作用户（actor）。

### 1.2 注册审计的模型

在 [models.py](file:///d:/fz/0601/solo-dogfeeding/code/70-paperless-ngx/src/documents/models.py#L1229-L1240) 中，通过 `auditlog.register()` 注册需要自动追踪变更的模型：

| 模型 | 特殊配置 |
|------|----------|
| `Document` | `m2m_fields={"tags"}`, `exclude_fields=["content_length", "modified"]` |
| `Correspondent` | - |
| `Tag` | - |
| `DocumentType` | - |
| `Note` | - |
| `CustomField` | - |
| `CustomFieldInstance` | - |

**注意**：Document 显式排除了 `modified` 字段（该字段 `auto_now=True` 每次保存都会变更，无审计价值）和 `content_length`（自动生成字段）。

---

## 二、元数据差异的计算逻辑

元数据差异记录分为 **自动追踪** 和 **手动记录** 两种模式。

### 2.1 自动追踪（django-auditlog）

django-auditlog 通过 Django signals（`post_save`、`m2m_changed`、`pre_delete`）自动捕获字段变化，存储在 `LogEntry.changes` 字段中，格式为 JSON：

```python
{
    "field_name": ["old_value", "new_value"]
}
```

对于多对多字段（如 Document.tags），django-auditlog 使用特殊结构：

```python
{
    "tags": {
        "type": "m2m",
        "operation": "add",  # 或 "remove"
        "objects": ["tag_name1", "tag_name2"]
    }
}
```

### 2.2 手动记录的变更场景

以下场景由于涉及跨模型关联或语义化操作，通过 `LogEntry.objects.log_create()` 手动记录：

#### 2.2.1 Notes 变更

在 [views.py](file:///d:/fz/0601/solo-dogfeeding/code/70-paperless-ngx/src/documents/views.py#L1644-L1695) 中，Note 的增删直接挂在 Document 的审计日志下：

```python
# Note Added
LogEntry.objects.log_create(
    instance=doc,
    changes={"Note Added": ["None", note.id]},
    action=LogEntry.Action.UPDATE,
)

# Note Deleted
LogEntry.objects.log_create(
    instance=doc,
    changes={"Note Deleted": [note.id, "None"]},
    action=LogEntry.Action.UPDATE,
)
```

#### 2.2.2 文档版本操作

在 [consumer.py](file:///d:/fz/0601/solo-dogfeeding/code/70-paperless-ngx/src/documents/consumer.py#L624-L641) 和 [views.py](file:///d:/fz/0601/solo-dogfeeding/code/70-paperless-ngx/src/documents/views.py#L2023-L2117) 中记录版本的增删与标签变更：

```python
# Version Added
LogEntry.objects.log_create(
    instance=root_doc,
    changes={"Version Added": ["None", version_doc.id]},
    action=LogEntry.Action.UPDATE,
    additional_data={"reason": "Version added", "version_id": version_doc.id},
)

# Version Deleted
changes={"Version Deleted": ["None", version_doc_id]}

# Version Label Updated
changes={"Version Label": [old_label, new_label]}
```

#### 2.2.3 自定义字段变更的视图层聚合

CustomFieldInstance 有自己独立的审计日志。但在 [views.py 的 history 端点](file:///d:/fz/0601/solo-dogfeeding/code/70-paperless-ngx/src/documents/views.py#L1778-L1800) 中，这些独立的日志被**重新格式化聚合**到 Document 的历史流中：

```python
for entry in LogEntry.objects.get_for_objects(doc.custom_fields.all()):
    entries.append({
        "changes": {
            "custom_fields": {
                "type": "custom_field",
                "field": str(entry.object_repr).split(":")[0].strip(),
                "value": str(entry.object_repr).split(":")[1].strip(),
            },
        },
        # ...
    })
```

### 2.3 前端差异展示

前端在 [document-history.component.html](file:///d:/fz/0601/solo-dogfeeding/code/70-paperless-ngx/src-ui/src/app/components/document-detail/document-history/document-history.component.html#L30-L56) 中根据 `changes` 的结构分支渲染：

- **普通字段**：`change.value` 是一个二元数组 `[old, new]`，使用 `getPrettyName()` 对关联对象 ID 进行名称解析
- **M2M 字段**：`change.value["type"] === 'm2m'`，显示 operation + objects
- **自定义字段**：`change.value["type"] === 'custom_field'`，直接显示 field + value
- **content 字段**：截取前 100 字符避免 UI 爆炸

---

## 三、操作者（Actor）记录的实现方式

操作者记录通过三种机制实现，覆盖不同运行上下文。

### 3.1 AuditlogMiddleware（HTTP 请求上下文）

`auditlog.middleware.AuditlogMiddleware` 在每个 HTTP 请求开始时自动将 `request.user` 设置为线程/协程本地的 actor。这是最常见的场景——用户通过 Web UI 或 API 修改文档时，操作者自动被记录。

### 3.2 `set_actor` 上下文管理器（非 HTTP 上下文）

在没有 `request` 对象的场景（如 Celery 任务、后台消费、序列化器更新），使用 `auditlog.context.set_actor` 显式设置：

#### 3.2.1 序列化器更新

在 [serialisers.py](file:///d:/fz/0601/solo-dogfeeding/code/70-paperless-ngx/src/documents/serialisers.py#L1204-L1206) 中：

```python
if settings.AUDIT_LOG_ENABLED:
    with set_actor(self.user):
        super().update(instance, validated_data)
```

#### 3.2.2 消费流程中保存原始文档

在 [consumer.py](file:///d:/fz/0601/solo-dogfeeding/code/70-paperless-ngx/src/documents/consumer.py#L610-L618) 中：

```python
actor = User.objects.filter(pk=self.metadata.actor_id).first()
if actor is not None:
    with set_actor(actor):
        original_document.save()
```

### 3.3 `log_create` 的 `actor` 参数（手动记录）

对于所有通过 `LogEntry.objects.log_create()` 手动创建的日志，actor 通过参数直接传入：

```python
LogEntry.objects.log_create(
    instance=doc,
    changes={...},
    action=LogEntry.Action.UPDATE,
    actor=user,  # request.user 或其他 User 对象
    additional_data={...},
)
```

### 3.4 特殊情况：缺失 actor

Notes 变更（[views.py](file:///d:/fz/0601/solo-dogfeeding/code/70-paperless-ngx/src/documents/views.py#L1644-L1653)）在调用 `log_create` 时**未传入 actor 参数**，此时日志条目的 actor 为 None，前端显示为 "System"。

---

## 四、批量编辑的粒度控制

### 4.1 批量编辑方法与追踪字段映射

在 [views.py 的 BulkEditView](file:///d:/fz/0601/solo-dogfeeding/code/70-paperless-ngx/src/documents/views.py#L2798-L2817) 中，`MODIFIED_FIELD_BY_METHOD` 定义了每种批量方法对应的追踪粒度：

| 方法 | 追踪字段 | 说明 |
|------|----------|------|
| `set_correspondent` | `correspondent` | FK 字段 |
| `set_document_type` | `document_type` | FK 字段 |
| `set_storage_path` | `storage_path` | FK 字段 |
| `add_tag` / `remove_tag` / `modify_tags` | `tags` | M2M 字段 |
| `modify_custom_fields` | `custom_fields` | 反向关联 |
| `set_permissions` | `None` | 不记录变更 |
| `delete` | `deleted_at` | 软删除标记 |
| `rotate` / `delete_pages` / `split` / `merge` / `edit_pdf` / `remove_password` | `None` | 创建新文档/版本，不直接修改原文档字段 |
| `reprocess` | `checksum` | 文件内容哈希 |

**设计意图**：标记为 `None` 的操作本质上创建新的 Document 记录或版本，旧字段不变，因此无需在原文档上挂差异日志。

### 4.2 批量审计日志的执行流程

在 [BulkEditView.post()](file:///d:/fz/0601/solo-dogfeeding/code/70-paperless-ngx/src/documents/views.py#L2864-L2909) 中：

1. **编辑前快照**：查询并缓存旧值
   ```python
   old_documents = {
       obj["pk"]: obj
       for obj in Document.objects.filter(pk__in=documents).values(
           "pk", "correspondent", "document_type", "storage_path",
           "tags", "custom_fields", "deleted_at", "checksum",
       )
   }
   ```

2. **执行编辑**：调用 `method(documents, **parameters)`

3. **编辑后逐文档对比记录**：
   ```python
   for doc in new_documents:
       old_value = old_documents[doc.pk][modified_field]
       new_value = getattr(doc, modified_field)
       # FK 字段取 pk，Manager 取 pk 列表
       if isinstance(new_value, Model):
           new_value = new_value.pk
       elif isinstance(new_value, Manager):
           new_value = list(new_value.values_list("pk", flat=True))

       LogEntry.objects.log_create(
           instance=doc,
           changes={modified_field: [old_value, new_value]},
           action=LogEntry.Action.UPDATE,
           actor=user,
           additional_data={"reason": f"Bulk edit: {method.__name__}"},
       )
   ```

### 4.3 粒度特征：文档级独立日志

批量编辑 **不会** 创建一条"批量变更"的总日志，而是为**每个受影响的文档各生成一条独立的 LogEntry**。这样做的好处：

- 每条日志天然关联到具体 Document，查询单文档历史时无需额外过滤
- `additional_data.reason` 字段保留批量溯源信息

### 4.4 CustomField 批量编辑的双重日志

`modify_custom_fields` 操作会产生两类日志：
1. **Document 级**：通过 BulkEditView 的手动记录，`changes={"custom_fields": [...]}` 挂在 Document 上
2. **CustomFieldInstance 级**：通过 django-auditlog 自动注册，每条 CustomFieldInstance 的增删改各自独立记录

在 history API 返回时，CustomFieldInstance 级别的日志会被二次查询并格式化为统一结构聚合到文档历史流中（参见 2.2.3 节）。

---

## 五、历史记录 API 端点

### 5.1 端点定义

`GET /api/documents/<pk>/history/` — [views.py history action](file:///d:/fz/0601/solo-dogfeeding/code/70-paperless-ngx/src/documents/views.py#L1743-L1802)

### 5.2 权限控制

```python
if not request.user.has_perm("auditlog.view_logentry") or (
    doc.owner is not None
    and doc.owner != request.user
    and not request.user.is_superuser
):
    return HttpResponseForbidden("Insufficient permissions")
```

需要同时满足：拥有 `auditlog.view_logentry` 全局权限 + 是文档 owner 或 superuser。

### 5.3 返回数据结构

```python
[
    {
        "id": 123,
        "timestamp": "2024-01-01T12:00:00Z",
        "action": "update",  # create / update / delete
        "changes": {...},    # 差异结构（见第二节）
        "actor": {"id": 1, "username": "admin"}  # 或 None
    },
    ...
]
```

所有条目按 `timestamp` 倒序排列。前端通过 [document.service.ts getHistory()](file:///d:/fz/0601/solo-dogfeeding/code/70-paperless-ngx/src-ui/src/app/services/rest/document.service.ts#L413-L415) 调用。

---

## 六、完整调用链总览

```
用户操作
  │
  ├─ HTTP API (PATCH/PUT /api/documents/<id>/)
  │     └─ DocumentSerializer.update()
  │          └─ with set_actor(user): super().update()
  │               └─ django-auditlog post_save signal → LogEntry (自动)
  │
  ├─ HTTP API (POST /api/documents/bulk_edit/)
  │     └─ BulkEditView.post()
  │          ├─ 快照 old_documents
  │          ├─ 调用 bulk_edit.* 方法 (直接UPDATE，绕过 save())
  │          └─ 遍历: LogEntry.objects.log_create(actor=user, additional_data={"reason": "Bulk edit: ..."})
  │
  ├─ HTTP API (Notes 增删)
  │     └─ views.py notes action
  │          └─ LogEntry.objects.log_create()  (注意: 无 actor)
  │
  ├─ HTTP API (版本操作: 删除/标签更新)
  │     └─ views.py version actions
  │          └─ LogEntry.objects.log_create(actor=request.user)
  │
  └─ Celery 任务 (消费/重处理等)
        └─ consumer.py / tasks.py
             └─ with set_actor(actor): document.save()
                  └─ django-auditlog signal → LogEntry
             └─ LogEntry.objects.log_create(actor=actor, additional_data={"reason": "Version added"})
```
