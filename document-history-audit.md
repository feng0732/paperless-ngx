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

#### 2.3.1 普通字段：只展示新值，不展示旧值

这是一个需要注意的关键边界：**普通字段的 UI 只展示 `change.value[1]`（新值），完全不展示 `change.value[0]`（旧值）**。

```html
<!-- 普通字段分支 -->
@else {
    <li>
        <span>{{ change.key | titlecase }}</span>:&nbsp;
        @if (change.key === 'content') {
            <code class="text-primary">{{ change.value[1]?.substring(0,100) }}...</code>
        } @else {
            <code class="text-primary">{{ getPrettyName(change.key, change.value[1]) | async }}</code>
        }
    </li>
}
```

具体行为：
- **content 字段**：`change.value[1]?.substring(0,100)` — 取新值的前 100 个字符
- **其他普通字段**：`getPrettyName(change.key, change.value[1])` — 对**新值**的 ID 进行名称解析

`getPrettyName()` 方法在 [document-history.component.ts](file:///d:/fz/0601/solo-dogfeeding/code/70-paperless-ngx/src-ui/src/app/components/document-detail/document-history/document-history.component.ts#L66-L113) 中的解析规则：

| 字段类型（change.key） | 解析方式 | 回退 |
|------|----------|------|
| `correspondent` | `CorrespondentService.getCached(id).name` | 直接显示 ID |
| `document_type` | `DocumentTypeService.getCached(id).name` | 直接显示 ID |
| `storage_path` | `StoragePathService.getCached(id).path` | 直接显示 ID |
| `owner` | `UserService.getCached(id).username` | 直接显示 ID |
| **其他所有**（title、tags、custom_fields、deleted_at 等） | 直接原值显示 | - |

注意：`DataType.Tag` 和 `DataType.CustomField` 在 `getPrettyName()` 中**没有分支处理**，因为这两类变更走的是下方的 `type === 'm2m'` 和 `type === 'custom_field'` 特殊分支。

#### 2.3.2 M2M 字段与自定义字段的特殊展示

- **M2M 字段（tags）**：`change.value["type"] === 'm2m'`，显示 `operation`（add/remove，首字母大写）+ 字段名 + `objects.join(', ')`（直接是 tag 名称字符串，无需二次解析）
- **自定义字段**：`change.value["type"] === 'custom_field'`，直接显示 `field`（字段名）+ `value`（字段值）

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

       LogEntry.objects.log_create(...)
   ```

#### 4.2.1 快照取值的关键边界：tags 与 custom_fields

`.values()` 对不同类型字段返回的数据形态不同，这是理解批量编辑快照的核心：

| 字段类型 | 示例字段 | `.values()` 返回值形态 | 示例 |
|------|------|----------|------|
| 普通字段 / FK | `correspondent`, `deleted_at`, `checksum` | 字段实际值（FK 为 pk） | `{"correspondent": 5}` |
| **M2M 字段** | **`tags`** | **单个关联对象的 pk（见下方详述）** | `{"tags": 3}` |
| **反向 OneToMany** | **`custom_fields`** | **单个关联对象的 pk（见下方详述）** | `{"custom_fields": 12}` |

**Django `.values()` 对 M2M 和反向关联的展开行为**：

Django 的 `QuerySet.values()` 在遇到 M2M 字段或反向 OneToMany 字段时，会执行 **SQL JOIN 展开**——一个文档如果关联了 N 个对象，就会产生 N 行结果，每行包含一个关联对象的 pk。

例如某文档关联了 tag_id=3、tag_id=7、tag_id=9 三个标签：

```
.values("pk", "tags") 的原始返回：
[
    {"pk": 1, "tags": 3},   // 第 1 行，对应 tag #3
    {"pk": 1, "tags": 7},   // 第 2 行，对应 tag #7
    {"pk": 1, "tags": 9},   // 第 3 行，对应 tag #9
]
```

但外层使用 **字典推导式** `{obj["pk"]: obj ...}`，相同 pk 会发生键冲突**覆盖**，最终只保留**最后一行**的单个 pk：

```python
old_documents = {
    1: {"pk": 1, "tags": 9, ...}   // 只保留了最后一个 tag_id=9
}
```

#### 4.2.2 快照值 vs 新值的数据结构不一致

快照取值和新值取值使用了完全不同的方式，导致 `changes` 数组的前后两项数据结构不一致：

```
old_value (来自 .values() + dict 覆盖)   →  单个 pk 整数  例如: 9
new_value (来自 Manager.values_list())    →  pk 整数列表    例如: [3, 7, 9, 11]
```

最终写入 `LogEntry.changes` 的结构为：

```python
{
    "tags": [9, [3, 7, 9, 11]]           // old: 单个 int, new: list[int]
    "custom_fields": [12, [12, 15, 18]]  // 同理
}
```

**对前端展示的影响**：

普通字段分支只读取 `change.value[1]`（新值，列表形式），然后调用 `getPrettyName("tags", [3, 7, 9, 11])`。由于 `tags` 不在 `getPrettyName()` 的 switch 分支中，且 `parseInt("[3, 7, 9, 11]")` 返回 `NaN`，最终**回退为直接显示原始字符串**，例如显示为 `3,7,9,11` 或数组的字符串形式。

#### 4.2.3 普通 FK 字段的取值一致性

对比而言，普通 FK 字段的快照和新值是一致的：

- 快照：`old_documents[doc.pk]["correspondent"]` → FK 的 pk 整数
- 新值：`doc.correspondent` → `Correspondent` 对象 → `.pk` → 整数

两者均为单个整数，前后结构统一。

#### 4.2.4 测试覆盖的缺失

在 [test_api_bulk_edit.py](file:///d:/fz/0601/solo-dogfeeding/code/70-paperless-ngx/src/documents/tests/test_api_bulk_edit.py#L1782-L1838) 中，`test_bulk_edit_audit_log_enabled_tags` 和 `test_bulk_edit_audit_log_enabled_custom_fields` **仅断言了 LogEntry 的条数**（1 条、2 条），并未对 `changes` 字段的实际内容做断言，因此快照取值的结构不一致问题未被测试捕获。

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

## 六、tags 与 custom_fields 历史记录的三条分流路径

tags 和 custom_fields 由于涉及多对多关联和独立实例模型，其历史记录来源不止一条，数据形态差异显著。本节对照代码逐一拆解。

### 6.1 总览：三条分流的触发场景

| 分流路径 | 触发场景 | 记录对象 | LogEntry.content_type |
|------|------|------|------|
| **A. django-auditlog 自动 M2M 审计** | 普通 API 保存（触发 `m2m_changed` signal） | Document | documents.Document |
| **B. 批量编辑手动日志** | `POST /api/documents/bulk_edit/`（modify_tags / modify_custom_fields） | Document | documents.Document |
| **C. 字段实例自动审计** | 任意修改 CustomFieldInstance 的操作 | CustomFieldInstance | documents.CustomFieldInstance |

history API 会从 **A + B** 中查询 Document 的 LogEntry，再单独从 **C** 中查询该文档所有 CustomFieldInstance 的 LogEntry，最后按 timestamp 合并排序返回。

### 6.2 分流 A：django-auditlog 自动 M2M 审计（仅 tags）

#### 6.2.1 触发机制

在 [models.py](file:///d:/fz/0601/solo-dogfeeding/code/70-paperless-ngx/src/documents/models.py#L1229-L1240) 中注册：

```python
auditlog.register(Document, m2m_fields={"tags"}, exclude_fields=["content_length", "modified"])
```

django-auditlog 通过 `m2m_changed` signal 监听，内部调用 `LogEntryManager.log_m2m_changes()` 方法。

#### 6.2.2 changes 数据形态

根据 django-auditlog 3.4.1 源码（`auditlog.models.LogEntryManager.log_m2m_changes`）：

```python
objects = [smart_str(instance) for instance in changed_queryset]
kwargs["changes"] = {
    field_name: {
        "type": "m2m",
        "operation": operation,   # "add" 或 "delete"
        "objects": objects,       # Tag.__str__() 的结果列表
    }
}
```

实际示例：

```python
{
    "tags": {
        "type": "m2m",
        "operation": "add",
        "objects": ["Invoice", "Important"]   # Tag 的 name，不是 ID
    }
}
```

#### 6.2.3 ID 含义

- `objects` 中的每一项是 `Tag.__str__()` 的结果。Tag 继承自 [MatchingModel](file:///d:/fz/0601/solo-dogfeeding/code/70-paperless-ngx/src/documents/models.py#L46-L93)，其 `__str__` 返回 `self.name`。
- **注意**：`objects` 存储的是 tag 名称字符串，**不是 tag 的 ID**。这意味着如果 tag 后续被重命名，历史记录中显示的仍是变更当时的名称。

#### 6.2.4 前端显示结果

走 `change.value["type"] === 'm2m'` 分支，在 [document-history.component.html](file:///d:/fz/0601/solo-dogfeeding/code/70-paperless-ngx/src-ui/src/app/components/document-detail/document-history/document-history.component.html#L32-L37) 中渲染：

```html
<li>
    <span class="fst-italic">{{ change.value["operation"] | titlecase }}</span>&nbsp;
    <span>{{ change.key | titlecase }}</span>:&nbsp;
    <code class="text-primary">{{ change.value["objects"].join(', ') }}</code>
</li>
```

显示效果：`Add Tags: Invoice, Important` 或 `Delete Tags: Old`

---

### 6.3 分流 B：批量编辑手动日志（tags 和 custom_fields）

#### 6.3.1 触发机制

在 [BulkEditView.post()](file:///d:/fz/0601/solo-dogfeeding/code/70-paperless-ngx/src/documents/views.py#L2864-L2909) 中，批量编辑不调用 `document.save()`，而是直接执行 SQL UPDATE 或 M2M Manager 操作（绕过 django-auditlog 的 signal），因此手动构造 LogEntry。

#### 6.3.2 changes 数据形态

```python
changes={modified_field: [old_value, new_value]}
```

##### tags 的实际 changes 示例：

```python
{
    "tags": [9, [3, 7, 9, 11]]
    #        ↑    └───────────┘
    #   old_value         new_value
    #   (单个 int)     (int 列表)
}
```

##### custom_fields 的实际 changes 示例：

```python
{
    "custom_fields": [12, [12, 15, 18]]
    #                  ↑    └────────┘
    #             old_value      new_value
    #             (单个 int)   (int 列表)
}
```

#### 6.3.3 ID 含义

- **old_value**：来自 `.values()` + 字典推导式覆盖后的结果，是**最后一个**关联对象的 pk（Tag pk 或 CustomFieldInstance pk），**并非完整列表**。详见 4.2.1 节关于 Django `.values()` 对 M2M/反向关联的展开行为分析。
- **new_value**：来自 `Manager.values_list("pk", flat=True)`，是完整的 pk 列表。
- 两者**均为数据库主键 ID**，不是名称。

#### 6.3.4 前端显示结果

由于 `change.value` 是一个二元数组（不是带 `"type": "m2m"` 的对象），走**普通字段分支**：

```html
<code class="text-primary">{{ getPrettyName(change.key, change.value[1]) | async }}</code>
```

即调用 `getPrettyName("tags", "[3, 7, 9, 11]")` 或 `getPrettyName("custom_fields", "[12, 15, 18]")`。

在 [getPrettyName()](file:///d:/fz/0601/solo-dogfeeding/code/70-paperless-ngx/src-ui/src/app/components/document-detail/document-history/document-history.component.ts#L66-L113) 中：

```typescript
const idInt = parseInt(id, 10)  // parseInt("[3, 7, 9, 11]") → NaN
if (!Number.isFinite(idInt)) {
    result$ = fallback$         // of(id) → 直接返回原始字符串
}
switch (type) {
    // "tags" 和 "custom_fields" 不在 switch 分支中
    default:
        result$ = fallback$
}
```

显示效果：
- `Tags: 3,7,9,11`（数组的 toString() 结果，或 JSON 字符串形式）
- `Custom Fields: 12,15,18`

**关键问题**：显示的是 pk ID 列表而非名称，用户无法直接识别。

---

### 6.4 分流 C：CustomFieldInstance 自动审计日志（仅 custom_fields）

#### 6.4.1 触发机制

CustomFieldInstance 模型在 [models.py](file:///d:/fz/0601/solo-dogfeeding/code/70-paperless-ngx/src/documents/models.py#L1240) 中通过 `auditlog.register(CustomFieldInstance)` 注册。任何对 CustomFieldInstance 的增删改（包括通过普通 API、批量编辑、工作流等）都会自动生成独立的 LogEntry，其 `content_type` 指向 `documents.CustomFieldInstance`。

#### 6.4.2 原始 changes 形态

CustomFieldInstance 自身的字段变更被 django-auditlog 自动记录，例如：

```python
# 新增 CustomFieldInstance 时（action=create）
changes = {
    "document": ["None", 42],
    "field": ["None", 7],
    "value_text": ["None", "invoice-2024-001"],
}

# 修改值时（action=update）
changes = {
    "value_text": ["old-value", "new-value"],
}
```

各字段的 ID 含义：
- `document`：Document 的 pk
- `field`：CustomField 的 pk
- `value_*` 系列：实际存储的字段值

#### 6.4.3 history API 的二次转换

history API **不会原样返回** CustomFieldInstance 的 changes。在 [views.py history action](file:///d:/fz/0601/solo-dogfeeding/code/70-paperless-ngx/src/documents/views.py#L1778-L1800) 中，这些日志被重新格式化：

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
        "id": entry.id,
        "timestamp": entry.timestamp,
        "action": entry.get_action_display(),
        "actor": ...,
    })
```

**关键转换逻辑**：使用 `entry.object_repr`（即 CustomFieldInstance 的 `__str__()` 结果）按冒号拆分为 field 和 value。

CustomFieldInstance 的 `__str__()` 定义在 [models.py](file:///d:/fz/0601/solo-dogfeeding/code/70-paperless-ngx/src/documents/models.py#L1190-L1191)：

```python
def __str__(self) -> str:
    return str(self.field.name) + f" : {self.value_for_search}"
```

`value_for_search` 属性在 [models.py](file:///d:/fz/0601/solo-dogfeeding/code/70-paperless-ngx/src/documents/models.py#L1210-L1226) 中定义，对 SELECT 类型会解析出 label，其余类型直接 `str(self.value)`。

转换后的 changes 形态：

```python
{
    "custom_fields": {
        "type": "custom_field",
        "field": "Invoice Number",   # CustomField.name
        "value": "INV-2024-001",      # value_for_search 的结果
    }
}
```

#### 6.4.4 ID 含义

- `field`：已被转换为 CustomField 的**名称**（不是 ID）
- `value`：已被转换为展示值（对于 SELECT 是 label，不是 select_options 中的 ID）
- 原始 LogEntry 中的 `changes`（包含各字段 ID 的 [old, new] 二元组）在转换过程中被**完全丢弃**，无法从 API 获取

#### 6.4.5 前端显示结果

走 `change.value["type"] === 'custom_field'` 分支，在 [document-history.component.html](file:///d:/fz/0601/solo-dogfeeding/code/70-paperless-ngx/src-ui/src/app/components/document-detail/document-history/document-history.component.html#L39-L43) 中渲染：

```html
<li>
    <span>{{ change.value["field"] }}</span>:&nbsp;
    <code class="text-primary">{{ change.value["value"] }}</code>
</li>
```

显示效果：`Invoice Number: INV-2024-001`

**注意**：该分支同样**不显示旧值**，只显示变更发生时 CustomFieldInstance 的 `object_repr` 快照。如果是删除操作，显示的是被删除前的 field:value；如果是修改操作，显示的是修改后的新值（因为 `object_repr` 取自 `smart_str(instance)`，为变更后的当前状态）。

---

### 6.5 三端分流对比总表

| 维度 | A. 自动 M2M 审计（tags） | B. 批量手动日志 | C. 字段实例审计（custom_fields） |
|------|------|------|------|
| **触发方式** | `m2m_changed` signal | `log_create()` 手动调用 | `post_save`/`pre_delete` signal |
| **changes 顶层 key** | `tags` | `tags` 或 `custom_fields` | `custom_fields`（转换后） |
| **changes 结构** | `{type: "m2m", operation, objects}` | `[old_value, new_value]` 二元组 | `{type: "custom_field", field, value}` |
| **ID/值形态** | objects = Tag 名称列表 | old = 单个 pk int；new = pk 列表 | field = 名称；value = 展示值 |
| **能否显示旧值** | 通过 operation 语义表达 add/delete | 存储了但前端不显示 | 不存储（转换时丢弃原始 changes） |
| **前端显示分支** | `type === 'm2m'` | 普通字段分支 | `type === 'custom_field'` |
| **前端显示效果** | `Add Tags: Invoice, Important` | `Tags: 3,7,9,11`（pk 列表） | `Invoice Number: INV-2024-001` |
| **是否区分增/删** | 是（operation 字段） | 否（只显示最终状态） | 否（只显示最终值） |
| **仅 tags 有** | ✅ | ✅（modify_tags） | ❌ |
| **仅 custom_fields 有** | ❌ | ✅（modify_custom_fields） | ✅ |
| **两者共有** | ❌ | ✅ | ❌ |

---

## 七、完整调用链总览

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
