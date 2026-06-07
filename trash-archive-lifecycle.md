# Paperless-ngx Trash & Archive 生命周期代码分析

本文档基于对 `src/documents/` 模块源代码的逐行核对，系统梳理 Trash（回收站）和 Archive（归档）相关的完整生命周期处理路径。所有代码引用均采用仓库相对路径格式。

---

## 1. 核心基础：软删除机制

Paperless-ngx 使用 `django-softdelete` 库实现软删除（回收站）机制。

### 1.1 模型层设计

[Document](src/documents/models.py#L157) 继承自 `SoftDeleteModel`，这是整个回收站机制的基石：

```python
# src/documents/models.py L157
class Document(SoftDeleteModel, ModelWithOwner):
```

继承 `SoftDeleteModel` 后获得以下能力：

| Manager / 方法 / 属性 | 作用 |
|----------------------|------|
| `Document.objects` | 默认查询集，自动过滤已软删除的文档 |
| `Document.deleted_objects` | 仅查询已软删除的文档（回收站内容） |
| `Document.global_objects` | 查询所有文档，包括已删除和未删除 |
| `doc.is_deleted` | 布尔属性，标记文档是否处于软删除状态 |
| `doc.deleted_at` | 软删除时间戳（DateTimeField，可空） |
| `doc.delete()` | 执行软删除（设置 `deleted_at`，不真正从 DB 删除） |
| `doc.restore(strict=False)` | 从回收站恢复文档，清除 `deleted_at` |
| `doc.hard_delete()` | 真正从数据库删除记录（不常用） |

### 1.2 同样使用软删除的模型

以下模型也继承 `SoftDeleteModel`，支持软删除/恢复：

- [Note](src/documents/models.py#L828) — 文档备注
- [ShareLink](src/documents/models.py#L868) — 分享链接
- [CustomFieldInstance](src/documents/models.py#L1093) — 自定义字段实例

### 1.3 Document.delete() 方法重写

[Document.delete()](src/documents/models.py#L503-L514) 被重写，确保删除根文档时其所有版本文档也一并进入回收站：

```python
# src/documents/models.py L503-L514
def delete(self, *args, **kwargs):
    # If deleting a root document, move all its versions to trash as well.
    if self.root_document_id is None:
        Document.objects.filter(root_document=self).delete()
    return super().delete(*args, **kwargs)
```

**关键**：只在当前文档是根文档（`root_document_id is None`）时才显式级联软删除版本。这补充了 `root_document` 外键上的 `on_delete=models.CASCADE`（[src/documents/models.py L312-L319](src/documents/models.py#L312-L319)），因为 Django 的 CASCADE 只在真正硬删除时触发。

单元测试验证见 [src/documents/tests/test_document_model.py L105-L125](src/documents/tests/test_document_model.py#L105-L125)。

---

## 2. 完整生命周期流程图

```
用户删除文档（单删 / 批删 / 工作流）
    │
    ▼
┌─────────────────────────────────────────────┐
│  软删除（进入回收站）                         │
│  - 设置 Document.deleted_at                  │
│  - Note / ShareLink / CustomFieldInstance    │
│    由 django-softdelete 级联软删除            │
│  - 从 Tantivy 全文搜索索引移除                │
│  - 从 LLM 向量索引异步移除（若 llm_index_enabled）│
│  - 发送 WebSocket 通知（仅批量删除）          │
│  - ⚠️ 磁盘文件（原文件/归档/缩略图）保持不变   │
└─────────────────────────────────────────────┘
    │
    ├───────────────────────────────┐
    │                               │
    ▼                               ▼
┌───────────────────┐     ┌───────────────────────────────────┐
│  恢复文档          │     │  清空回收站（手动 / 定时自动）       │
│  doc.restore()     │     │  tasks.empty_trash()               │
│                   │     │                                    │
│  - 清除 deleted_at │     │  - 临时连接 post_delete → 清理文件  │
│  - 关联软删除对象   │     │  - 硬删除 Document DB 记录          │
│    一并恢复        │     │  - 删除/移动磁盘文件                │
│  - ⚠️ 不自动重索引  │     │  - 清理审计日志 LogEntry            │
│                   │     │  - 断开信号                         │
└───────────────────┘     └───────────────────────────────────┘
```

---

## 3. 软删除（移入回收站）流程

### 3.1 三条触发路径

#### 路径一：单文档 REST DELETE

- **API**：`DELETE /api/documents/<id>/`
- **视图**：[DocumentViewSet.destroy()](src/documents/views.py#L1204-L1218)

```python
# src/documents/views.py L1204-L1218
def destroy(self, request, *args, **kwargs):
    from documents.search import get_backend
    get_backend().remove(self.get_object().pk)  # ① 从搜索索引移除
    try:
        return super().destroy(request, *args, **kwargs)  # ② DRF destroy → model.delete()
    except Exception as e:
        ...  # 异常处理
```

#### 路径二：批量删除（异步 Celery 任务）

- **API**：`POST /api/documents/delete/`
- **视图**：[DeleteDocumentsView](src/documents/views.py#L2987-L2997)
- **异步执行器**：[bulk_edit.delete()](src/documents/bulk_edit.py#L359-L392)

```python
# src/documents/views.py L2987-L2997
class DeleteDocumentsView(DocumentOperationPermissionMixin):
    serializer_class = DeleteDocumentsSerializer
    def post(self, request, *args, **kwargs):
        serializer = self.get_serializer(data=request.data)
        serializer.is_valid(raise_exception=True)
        return self._execute_document_action(
            method=bulk_edit.delete,
            validated_data=serializer.validated_data,
            operation_label="document delete",
        )
```

```python
# src/documents/bulk_edit.py L359-L392
@shared_task
def delete(doc_ids: list[int]) -> Literal["OK"]:
    # ① 收集根文档及其所有版本文档
    root_ids = Document.objects.filter(id__in=doc_ids, root_document__isnull=True)\
        .values_list("id", flat=True).distinct()
    version_ids = Document.objects.filter(root_document_id__in=root_ids)\
        .exclude(id__in=doc_ids).values_list("id", flat=True).distinct()
    delete_ids = list({*doc_ids, *version_ids})

    Document.objects.filter(id__in=delete_ids).delete()  # ② 批量软删除

    # ③ 从 Tantivy 搜索索引批量移除
    from documents.search import get_backend
    with get_backend().batch_update() as batch:
        for id in delete_ids:
            batch.remove(id)

    # ④ WebSocket 通知前端
    status_mgr = DocumentsStatusManager()
    status_mgr.send_documents_deleted(delete_ids)
    return "OK"
```

#### 路径三：工作流自动删除（Move to trash）

- **动作类型常量**：`WorkflowAction.WorkflowActionType.MOVE_TO_TRASH = 6`（[src/documents/models.py L1578-L1581](src/documents/models.py#L1578-L1581)）
- **执行器**：[execute_move_to_trash_action()](src/documents/workflows/actions.py#L348-L375)

```python
# src/documents/workflows/actions.py L348-L375
def execute_move_to_trash_action(
    action: WorkflowAction,
    document: Document | ConsumableDocument,
    logging_group: uuid.UUID | None,
) -> None:
    if isinstance(document, Document):
        document.delete()  # ① 已入库文档：走正常软删除
    else:
        # ② 消费过程中触发：直接删文件 + 抛出 StopConsumeTaskError 终止
        if document.original_file.exists():
            document.original_file.unlink()
        raise StopConsumeTaskError(
            "Document deleted by workflow action during consumption",
        )
```

**工作流短路逻辑**：在 `run_workflows` 循环中，每次动作后检查 `document.is_deleted`，若为真则 `break` 跳过后续动作（[src/documents/signals/handlers.py L907-L913](src/documents/signals/handlers.py#L907-L913)）：

```python
# src/documents/signals/handlers.py L907-L913
if document.is_deleted:
    logger.info(
        "Document was moved to trash, skipping remaining workflows",
        extra={"group": logging_group},
    )
    break
```

### 3.2 软删除核心事实

**磁盘文件不会被删除**。软删除只做以下数据库和索引层面的操作：
1. 给 Document（以及级联软删除的 Note/ShareLink/CustomFieldInstance）写 `deleted_at` 时间戳
2. 从 Tantivy 全文搜索索引移除（单删和批删都显式调用）
3. 触发 `post_delete` 信号 → 若 `AIConfig.llm_index_enabled` 为真则异步从 LLM 索引移除（详见第 6.3 节）
4. 批量删除额外发送 WebSocket 通知

---

## 4. 回收站管理（TrashView）

### 4.1 API 端点

- **URL**：`/api/trash/`
- **路由注册**：[src/paperless/urls.py L257-L259](src/paperless/urls.py#L257-L259)
- **视图类**：[TrashView](src/documents/views.py#L5083-L5123)

### 4.2 GET — 列出回收站内容

```python
# src/documents/views.py L5091
queryset = Document.deleted_objects.all()  # 仅已软删除的文档
```

### 4.3 POST — 恢复或清空

请求体使用 [TrashSerializer](src/documents/serialisers.py#L3391-L3411)：

| 字段 | 必填 | 说明 |
|------|------|------|
| `documents` | 否 | 文档 ID 列表；不提供则操作全部回收站内容 |
| `action` | 是 | `"restore"` 或 `"empty"` |

```python
# src/documents/serialisers.py L3391-L3411
class TrashSerializer(SerializerWithPerms):
    documents = serializers.ListField(required=False, write_only=True, ...)
    action = serializers.ChoiceField(choices=["restore", "empty"], write_only=True, ...)

    def validate_documents(self, documents):
        # 校验：所有 ID 都必须属于已软删除文档
        count = Document.deleted_objects.filter(id__in=documents).count()
        if not count == len(documents):
            raise serializers.ValidationError(
                "Some documents in the list have not yet been deleted.",
            )
        return documents
```

#### 恢复文档（action=restore）

[src/documents/views.py L5116-L5118](src/documents/views.py#L5116-L5118)：

```python
if action == "restore":
    for doc in Document.deleted_objects.filter(id__in=doc_ids).all():
        doc.restore(strict=False)
```

`restore(strict=False)` 由 django-softdelete 提供，会清除 `deleted_at`，并级联恢复同样被软删除的关联对象（Note/ShareLink/CustomFieldInstance）。

**⚠️ 恢复后不会自动触发 Tantivy 搜索索引和 LLM 向量索引重建**。代码证据：
- 搜索索引 `add_to_index` 仅连接在 `document_consumption_finished` 信号上（[src/documents/apps.py L29](src/documents/apps.py#L29)），restore 不触发该信号
- LLM 索引 `add_or_update_document_in_llm_index` 同样仅连接在 `document_consumption_finished` 上（[src/documents/apps.py L31](src/documents/apps.py#L31)）
- `document_updated` 信号也没有连接任何索引更新函数（[src/documents/apps.py L32-L33](src/documents/apps.py#L32-L33)）
- TrashView 恢复代码中也没有显式调用 `get_backend().add_or_update()`

因此恢复后文档不会立刻出现在搜索结果中，需要等待后续触发索引重建的事件（如修改文档）或手动重建。

#### 清空回收站（action=empty）

[src/documents/views.py L5119-L5122](src/documents/views.py#L5119-L5122)：

```python
elif action == "empty":
    if doc_ids is None:
        doc_ids = [doc.id for doc in docs]
    empty_trash(doc_ids=doc_ids)  # 永久删除
```

---

## 4.4 恢复文档后索引重新加入路径分析

恢复（`doc.restore(strict=False)`）本身**完全不触发任何索引更新**。要让恢复后的文档重新出现在 Tantivy 搜索和 LLM 问答中，需要依赖以下 5 条间接路径。本节逐条分析它们对两种索引的真实影响。

### 4.4.1 信号注册总览（证据基础）

在分析具体路径之前，先明确两个自定义信号的全部处理函数——这是判断每条路径影响的基础。

信号连接全部定义在 [src/documents/apps.py L24-L33](src/documents/apps.py#L24-L33)：

```python
document_consumption_finished.connect(add_inbox_tags)
document_consumption_finished.connect(set_correspondent)
document_consumption_finished.connect(set_document_type)
document_consumption_finished.connect(set_tags)
document_consumption_finished.connect(set_storage_path)
document_consumption_finished.connect(add_to_index)              # ⭐ Tantivy 索引
document_consumption_finished.connect(run_workflows_added)
document_consumption_finished.connect(add_or_update_document_in_llm_index)  # ⭐ LLM 索引
document_updated.connect(run_workflows_updated)                  # 仅工作流
document_updated.connect(send_websocket_document_updated)         # 仅 WebSocket
```

**关键结论**：`document_updated` 信号的两个处理函数都与索引无关，只有 `document_consumption_finished` 同时连接了 Tantivy 和 LLM 索引的添加函数。

### 4.4.2 五条路径逐条分析

#### 路径一：`document_updated` 信号触发

**触发位置**（共 5 处）：

| 场景 | 代码位置 | 触发前是否已更新 Tantivy？ |
|------|---------|--------------------------|
| 单文档 REST API `PUT/PATCH` | [src/documents/views.py L1178](src/documents/views.py#L1178) | ✅ 是（L1176 已显式 `add_or_update`） |
| 删除文档版本 | [src/documents/views.py L2046](src/documents/views.py#L2046) | ✅ 是（L2022 已显式 `add_or_update` 根文档） |
| 更新版本标签 | [src/documents/views.py L2119](src/documents/views.py#L2119) | ❌ 否（此处无显式索引调用） |
| `bulk_update_documents` 内循环 | [src/documents/tasks.py L260](src/documents/tasks.py#L260) | ⭕ 稍后批量处理（见路径二） |
| 消费中生成新版本后 | [src/documents/consumer.py L732-L735](src/documents/consumer.py#L732-L735) | ❌ 否（`document_consumption_finished` 发送的是版本文档，此处 `document_updated` 发送的是**根文档**，用于通知前端变更） |

**`document_updated` 的两个处理函数**：

- [run_workflows_updated()](src/documents/signals/handlers.py L819-L829)：仅运行 `DOCUMENT_UPDATED` 类型的工作流
- [send_websocket_document_updated()](src/documents/signals/handlers.py L832-L851)：仅通过 WebSocket 向前端推送文档变更通知

**对索引的影响**：
- **Tantivy 搜索索引**：❌ 不影响（信号处理函数中无索引操作。索引更新必须在 `send()` 之前显式调用）
- **LLM 向量索引**：❌ 不影响（同理）

这也印证了为什么恢复文档后必须走其他路径——因为 restore 不会发送 `document_updated`，即便发送也不会更新索引。

---

#### 路径二：`bulk_update_documents` 批量编辑异步任务

**触发方式**：所有批量编辑（修改往来人、文档类型、存储路径、标签增删、自定义字段增删、权限变更等）在完成数据库 `update()` 后，都会异步调用此任务。入口见 [src/documents/bulk_edit.py L126-L354](src/documents/bulk_edit.py#L126-L354) 中各处 `bulk_update_documents.apply_async()`。

**核心实现**：[src/documents/tasks.py L253-L275](src/documents/tasks.py#L253-L275)

```python
@shared_task
def bulk_update_documents(document_ids) -> None:
    from documents.search import get_backend

    documents = Document.objects.filter(id__in=document_ids)

    for doc in documents:
        clear_document_caches(doc.pk)
        document_updated.send(              # ① 发工作流 + WebSocket
            sender=None,
            document=doc,
            logging_group=uuid.uuid4(),
        )
        post_save.send(Document, instance=doc, created=False)

    with get_backend().batch_update() as batch:
        for doc in documents:
            batch.add_or_update(doc)          # ② ⭐ 批量更新 Tantivy 索引

    ai_config = AIConfig()
    if ai_config.llm_index_enabled:
        update_llm_index(                    # ③ ⭐ 增量更新 LLM 索引
            rebuild=False,
        )
```

**对索引的影响**：
- **Tantivy 搜索索引**：✅ **会重新加入**。通过 `batch.add_or_update(doc)` 批量写入。由于查询条件是 `Document.objects`（默认 Manager，自动过滤软删除文档），恢复后的文档（`deleted_at=NULL`）会被包含。
- **LLM 向量索引**：✅ **会重新加入**（仅当 `llm_index_enabled=True`）。调用 `update_llm_index(rebuild=False)` 做增量重建。
- 工作流：✅ 触发 `run_workflows_updated`
- WebSocket：✅ 触发前端通知

**⚠️ 注意**：`Document.objects` 会过滤已软删除文档，所以**如果文档仍在回收站中（`is_deleted=True`），它不会出现在 `documents` 查询集中，也就不会被重新加入索引**。只有恢复之后（`deleted_at` 被清空）再触发批量编辑，才会生效。

---

#### 路径三：`document_consumption_finished` 文档消费完成

**触发位置**：[src/documents/consumer.py L658-L666](src/documents/consumer.py#L658-L666)

> ⚠️ **修正之前不准确结论**："仅在新文档首次消费完成时触发"是错误的。实际上，**新文档消费**和**为已有文档新增版本消费**都会触发该信号，但两种场景下 `send()` 的 `document` 参数是不同的对象。

---

##### 场景 A：新文档首次消费（`self.input_doc.root_document_id` 为空）

执行路径见 [src/documents/consumer.py L643-L649](src/documents/consumer.py#L643-L649) 和 [L654-L731](src/documents/consumer.py#L654-L731)：

```python
# L643-L649: 走 _store 分支，创建全新根文档
document = self._store(text=text, date=date, page_count=page_count, mime_type=mime_type)

# L654-L656: document 是新创建的根文档
document = Document.objects.prefetch_related("versions").get(pk=document.pk)

# L658-L666: 发送 document_consumption_finished，参数是根文档
document_consumption_finished.send(sender=self.__class__, document=document, ...)

# L729: 保存根文档
document.save()

# L731: document.root_document_id 为空 → 不发送 document_updated
if document.root_document_id:
    document_updated.send(...)   # 跳过
```

| 信号 | 发送对象 | Tantivy | LLM | 工作流 | WebSocket |
|------|---------|---------|-----|-------|----------|
| `document_consumption_finished` | **根文档** | ✅ 加入 | ✅ 加入（启用时） | ✅ `DOCUMENT_ADDED` | ❌ |
| `document_updated` | —（不发送） | — | — | — | — |

---

##### 场景 B：为已有文档新增版本（`self.input_doc.root_document_id` 不为空）

执行路径见 [src/documents/consumer.py L589-L642](src/documents/consumer.py#L589-L642) 和 [L654-L735](src/documents/consumer.py#L654-L735)：

```python
# L589-L601: 获取根文档，创建版本文档（继承根文档的 title/created/owner 等，
#            但有独立 pk、content、checksum）
root_doc = Document.objects.get(pk=self.input_doc.root_document_id)
original_document = self._create_version_from_root(root_doc, text=text, ...)

# L642: ⭐ 关键：document 被赋值为新创建的「版本文档」，不是根文档！
document = original_document

# L654-L656: 预取后 document 仍然是版本文档
document = Document.objects.prefetch_related("versions").get(pk=document.pk)

# L658-L666: 发送 document_consumption_finished，参数是「版本文档」
document_consumption_finished.send(sender=self.__class__, document=document, ...)

# L729: 保存版本文档
document.save()

# L731-L735: document.root_document_id 不为空 → 额外发送 document_updated，
#            这次发送的是「根文档」
if document.root_document_id:
    document_updated.send(sender=self.__class__, document=document.root_document)
```

**版本文档的结构**（[src/documents/consumer.py L279-L295](src/documents/consumer.py#L279-L295)）：
- `root_document=root_doc_frozen` —— 外键指向根文档
- `version_index` —— 递增的版本号
- 独立的 `pk`、`content`、`checksum`、`page_count`、`mime_type`
- 继承 `title`、`created`、`owner_id`、tags 等元数据

---

##### 两种场景下各处理函数的实际作用

`document_consumption_finished` 共连接 8 个处理函数（[src/documents/apps.py L24-L31](src/documents/apps.py#L24-L31)），两种场景下它们的实际效果：

| 处理函数 | 场景 A（新文档）接收：根文档 | 场景 B（新增版本）接收：版本文档 |
|---------|-------------------------|------------------------------|
| `add_inbox_tags` | ✅ 为新根文档打 inbox 标签 | ⚠️ 为版本文档打标签（版本本身没有独立 tag 管理，通常无实际效果） |
| `set_correspondent` | ✅ 自动匹配根文档往来人 | ⚠️ 为版本文档匹配（版本继承自根文档，通常覆盖无效） |
| `set_document_type` | ✅ 自动匹配根文档类型 | ⚠️ 同上 |
| `set_tags` | ✅ 自动匹配根文档标签 | ⚠️ 同上 |
| `set_storage_path` | ✅ 设置根文档存储路径 | ⚠️ 同上 |
| `add_to_index` | ✅ 根文档按自身 pk 加入 Tantivy | ⚠️ **版本文档按自身 pk 加入 Tantivy**，但 REST API 查询层有 `filter(root_document__isnull=True)`（[src/documents/views.py L1042](src/documents/views.py#L1042)），搜索结果的二次交集过滤会排除版本 ID，用户搜索不到版本 |
| `run_workflows_added` | ✅ 运行根文档 `DOCUMENT_ADDED` 工作流 | ⚠️ 对版本文档运行 `DOCUMENT_ADDED` 工作流（工作流内部通常通过 `document.root_document` 找到根再操作，具体取决于工作流动作） |
| `add_or_update_document_in_llm_index` | ✅ 根文档加入 LLM 向量索引 | ⚠️ **版本文档按自身 pk 加入 LLM 向量索引**（插件内部行为取决于 `paperless_ai` 实现） |

此外场景 B 在文件写入后还会额外发送：

| 信号 | 发送对象 | Tantivy | LLM | 工作流 | WebSocket |
|------|---------|---------|-----|-------|----------|
| `document_updated` | **根文档**（`document.root_document`） | ❌ | ❌ | ✅ `DOCUMENT_UPDATED` | ✅ 推送通知 |

---

##### 对恢复文档路径的启示

恢复文档不会触发消费流程，因此上述两种场景都不会自动发生。如需让恢复的文档重新出现在搜索中，需要依赖其他路径（批量编辑、手动重建索引等）。

---

#### 路径四：后台管理（Django Admin）保存文档

**实现**：[src/documents/admin.py L58-L120](src/documents/admin.py#L58-L120) 中的 `DocumentAdmin`

```python
# src/documents/admin.py L116-L120
def save_model(self, request, obj, form, change):
    from documents.search import get_backend
    get_backend().add_or_update(obj)   # ⭐ 显式更新 Tantivy
    super().save_model(request, obj, form, change)

# src/documents/admin.py L110-L114
def delete_model(self, request, obj):
    from documents.search import get_backend
    get_backend().remove(obj.pk)       # ⭐ 显式从 Tantivy 移除
    super().delete_model(request, obj)
```

**对索引的影响**：
- **Tantivy 搜索索引**：✅ **会重新加入**。`save_model()` 在保存前显式调用 `get_backend().add_or_update(obj)`。
- **LLM 向量索引**：❌ **不会重新加入**。`DocumentAdmin` 中没有任何 LLM 索引相关调用，也不会发送 `document_consumption_finished` 信号。
- 工作流：❌ 不触发 `document_updated`，不运行工作流
- WebSocket：❌ 不推送通知

**注意**：`DocumentAdmin.get_queryset()` 返回 `Document.global_objects.all()`（[src/documents/admin.py L96-L100](src/documents/admin.py#L96-L100)），即管理员在后台可以看到并编辑已软删除（回收站中）的文档。因此在后台直接保存一个回收站中的文档，Tantivy 索引中可能会出现一条 `is_deleted=True` 的记录——不过搜索前端使用 `Document.objects` 查询，通常不会返回这些文档。

---

#### 路径五：手动重建索引（Management Commands）

这是最可靠的恢复方式，不依赖任何触发条件。

##### Tantivy 搜索索引：`document_index reindex`

[src/documents/management/commands/document_index.py L16-L70](src/documents/management/commands/document_index.py#L16-L70)

```bash
# 增量重建（保留现有索引，全量覆盖）
python manage.py document_index reindex

# 先清空再完全重建
python manage.py document_index reindex --recreate

# 仅在索引过期时重建（schema/语言变更时）
python manage.py document_index reindex --if-needed
```

核心逻辑：
```python
documents = Document.objects.select_related(...).prefetch_related(...)  # 默认 Manager
get_backend().rebuild(documents, ...)
```

- **Tantivy 搜索索引**：✅ 使用 `Document.objects`（非软删除）全量重建，所有恢复后的文档会被包含。

##### LLM 向量索引：`document_llmindex rebuild | update`

[src/documents/management/commands/document_llmindex.py L7-L24](src/documents/management/commands/document_llmindex.py#L7-L24)

```bash
# 完全重建 LLM 向量索引
python manage.py document_llmindex rebuild

# 增量更新 LLM 向量索引
python manage.py document_llmindex update
```

最终调用 [llmindex_index()](src/documents/tasks.py L628-L643)，内部先检查 `ai_config.llm_index_enabled`，然后委托给 `paperless_ai.indexing.update_llm_index`。

- **LLM 向量索引**：✅（仅当 `llm_index_enabled=True`）全量重建或增量更新，包含所有非软删除文档。

### 4.4.3 影响矩阵汇总

| 路径 | 触发场景 | Tantivy 搜索索引 | LLM 向量索引 | 工作流 | WebSocket | 对恢复文档是否有效 |
|------|---------|----------------|------------|-------|----------|-----------------|
| `doc.restore()` 本身 | 回收站恢复 | ❌ 不更新 | ❌ 不更新 | ❌ 不触发 | ❌ 不推送 | ❌ 无效 |
| `document_updated` 信号 | 单文档编辑、版本变更等 | ❌ 不更新（仅信号处理） | ❌ 不更新 | ✅ 运行 | ✅ 推送 | ❌ 信号本身不更新索引 |
| `bulk_update_documents` | 批量编辑往来人/标签/类型/字段等 | ✅ `batch.add_or_update` | ✅ `update_llm_index`（启用时） | ✅ 运行 | ✅ 推送 | ✅ 需在恢复后编辑 |
| `document_consumption_finished`（场景 A） | 新文档首次消费，发送对象=根文档 | ✅ `add_to_index`（按根文档 pk） | ✅ `add_or_update_document_in_llm_index`（启用时） | ✅ `DOCUMENT_ADDED` | ❌ 不推送 | ❌ 恢复不会重新消费 |
| `document_consumption_finished`（场景 B） | 为已有文档新增版本，发送对象=版本文档 | ⚠️ `add_to_index`（按版本文档 pk，搜索层过滤排除） | ⚠️ `add_or_update_document_in_llm_index`（按版本 pk） | ⚠️ 对版本运行 `DOCUMENT_ADDED` | ❌ 不推送 | ❌ 恢复不会重新消费 |
| `document_updated`（场景 B 追加） | 新增版本后追加发送，发送对象=根文档 | ❌ | ❌ | ✅ `DOCUMENT_UPDATED` | ✅ 推送 | ❌ |
| Admin `DocumentAdmin.save_model` | 管理员后台编辑保存 | ✅ `add_or_update` | ❌ 不更新 | ❌ 不触发 | ❌ 不推送 | ✅ Tantivy 有效，LLM 无效 |
| `document_index reindex` | 手动管理命令 | ✅ `get_backend().rebuild()` 全量（含版本文档，但搜索层过滤） | — | ❌ | ❌ | ✅ 最可靠 |
| `document_llmindex rebuild` | 手动管理命令 | — | ✅ 全量重建（启用时） | ❌ | ❌ | ✅ 最可靠 |
| REST API 单文档 `PUT/PATCH` | 前端编辑保存 | ✅ 显式 `add_or_update`（信号前调用） | ❌ 不更新 | ✅ 运行 | ✅ 推送 | ✅ Tantivy 有效，LLM 无效 |
| 备注增删 API | 新增/删除文档备注 | ✅ 显式 `add_or_update` | ❌ 不更新 | ❌ | ❌ | ✅ Tantivy 有效，LLM 无效 |

### 4.4.4 恢复后让文档出现在搜索中的推荐做法

根据上表，恢复文档后若需立即生效，有三种可靠方式：

1. **最全面**：执行两个命令
   ```bash
   python manage.py document_index reindex
   python manage.py document_llmindex rebuild   # 如启用 LLM
   ```

2. **前端操作 Tantivy 即可**：对恢复的文档执行任意一次编辑保存（修改任意字段后保存），REST API `PUT` 会触发 `get_backend().add_or_update()`（[src/documents/views.py L1176](src/documents/views.py#L1176)）

3. **批量操作 Tantivy 即可**：对恢复的文档执行一次批量编辑（例如重新设置同一个标签），触发 `bulk_update_documents`，同时更新 Tantivy 和 LLM（启用时）索引。

---

## 5. 永久删除（清空回收站）流程

### 5.1 三种触发方式

| 触发方式 | 入口 | doc_ids 参数 |
|---------|------|-------------|
| 手动清空指定文档 | `POST /api/trash/` `action=empty` + `documents=[...]` | 指定的 ID 列表 |
| 手动清空全部 | `POST /api/trash/` `action=empty`（不提供 documents） | 所有回收站文档 ID |
| **定时任务自动清理** | Celery Beat 每天 01:00 调 `documents.tasks.empty_trash` | `None`（走过期逻辑） |

定时任务配置在 [src/paperless/settings/custom.py L124-L134](src/paperless/settings/custom.py#L124-L134)：

```python
# src/paperless/settings/custom.py L124-L134
{
    "name": "Empty trash",
    "env_key": "PAPERLESS_EMPTY_TRASH_TASK_CRON",
    "env_default": "0 1 * * *",           # 默认每天 01:00
    "task": "documents.tasks.empty_trash",
    "options": {"expires": 23.0 * 60.0 * 60.0},
},
```

### 5.2 empty_trash 核心实现

[src/documents/tasks.py L398-L432](src/documents/tasks.py#L398-L432)：

```python
@shared_task
def empty_trash(doc_ids=None) -> None:
    documents = (
        Document.deleted_objects.filter(id__in=doc_ids)
        if doc_ids is not None
        else Document.deleted_objects.filter(
            deleted_at__lt=timezone.localtime(timezone.now())
            - datetime.timedelta(days=settings.EMPTY_TRASH_DELAY),  # 默认 30 天
        )
    )

    try:
        deleted_document_ids = list(documents.values_list("id", flat=True))
        # ⭐ 关键：临时连接文件清理信号
        models.signals.post_delete.connect(cleanup_document_deletion, sender=Document)
        documents.delete()  # 真正的硬删除
        logger.info(f"Deleted {len(deleted_document_ids)} documents from trash")

        # 清理审计日志
        if settings.AUDIT_LOG_ENABLED:
            LogEntry.objects.filter(
                content_type=ContentType.objects.get_for_model(Document),
                object_id__in=deleted_document_ids,
            ).delete()
    except Exception as e:
        logger.exception(f"Error while emptying trash: {e}")
    finally:
        # ⭐ 关键：无论成功异常都断开信号，避免污染后续软删除
        models.signals.post_delete.disconnect(
            cleanup_document_deletion, sender=Document,
        )
```

### 5.3 设计要点：为什么临时连接信号

这是整个回收站机制中最精妙的设计决策：

1. `django-softdelete` 的软删除 `delete()` 在内部也会触发 Django 的 `post_delete` 信号（用于实现链式级联）
2. 如果在 `apps.py` 的 `ready()` 中默认注册 `cleanup_document_deletion`，那么正常软删除时文件也会被误删
3. 所以选择**只在 `empty_trash` 执行期间临时连接**，用 `try/finally` 确保必定断开

`cleanup_document_deletion` 上方的注释也明确提示了这点：

```python
# see empty_trash in documents/tasks.py for signal handling
def cleanup_document_deletion(sender, instance, **kwargs) -> None:
```

---

## 6. 文件系统与索引清理逻辑

### 6.1 cleanup_document_deletion：磁盘文件清理

[src/documents/signals/handlers.py L342-L402](src/documents/signals/handlers.py#L342-L402)

```python
def cleanup_document_deletion(sender, instance, **kwargs) -> None:
    with FileLock(settings.MEDIA_LOCK):  # 跨进程文件锁
        if settings.EMPTY_TRASH_DIR:
            # A. 配置了 EMPTY_TRASH_DIR：把原文件移动到专用回收站目录
            counter = 0
            old_filename = Path(instance.source_path).name
            old_filebase = Path(old_filename).stem
            old_fileext = Path(old_filename).suffix
            while True:
                new_file_path = settings.EMPTY_TRASH_DIR / (
                    old_filebase + (f"_{counter:02}" if counter else "") + old_fileext
                )
                if new_file_path.exists():
                    counter += 1
                else:
                    break
            try:
                shutil.move(instance.source_path, new_file_path)
            except OSError as e:
                logger.error(f"Failed to move ...: {e}. Skipping cleanup!")
                return

        # B. 组装待删除文件列表
        files = (instance.archive_path, instance.thumbnail_path)
        if not settings.EMPTY_TRASH_DIR:
            # 只有未配置 EMPTY_TRASH_DIR 时才直接删原文件
            files += (instance.source_path,)

        # C. 删除文件（归档文件和缩略图永远直接删除）
        for filename in files:
            if filename and filename.is_file():
                try:
                    filename.unlink()
                except OSError as e:
                    logger.warning(
                        f"While deleting document {instance!s}, "
                        f"the file {filename} could not be deleted: {e}",
                    )

        # D. 递归清理空目录（仅 originals 和 archive 目录）
        delete_empty_directories(
            Path(instance.source_path).parent, root=settings.ORIGINALS_DIR,
        )
        if instance.has_archive_version:
            delete_empty_directories(
                Path(instance.archive_path).parent, root=settings.ARCHIVE_DIR,
            )
```

**文件处理矩阵**：

| 文件 | `EMPTY_TRASH_DIR` 未设置 | `EMPTY_TRASH_DIR` 已设置 |
|------|-------------------------|------------------------|
| `doc.source_path`（原文件） | `unlink()` 直接删除 | `shutil.move()` 移动到回收站目录，重名自动加 `_01/_02` 后缀 |
| `doc.archive_path`（PDF/A 归档） | `unlink()` 直接删除 | `unlink()` 直接删除 |
| `doc.thumbnail_path`（缩略图） | `unlink()` 直接删除 | `unlink()` 直接删除 |

递归清理空目录的实现在 [src/documents/file_handling.py L15-L41](src/documents/file_handling.py#L15-L41)，仅在 `ORIGINALS_DIR` 和 `ARCHIVE_DIR` 两个根目录内生效，不会越界删除。

### 6.2 Tantivy 全文搜索索引

搜索后端使用 Tantivy。删除操作只在**软删除时**显式移除，不在 empty_trash 中重复执行：

- 单删：[DocumentViewSet.destroy()](src/documents/views.py#L1207) → `get_backend().remove(pk)`
- 批删：[bulk_edit.delete()](src/documents/bulk_edit.py#L377-L381) → 批量 `batch.remove(id)`

后端实现在 [src/documents/search/_backend.py L501-L513](src/documents/search/_backend.py#L501-L513)：

```python
def remove(self, doc_id: int) -> None:
    self._ensure_open()
    with self.batch_update(lock_timeout=5.0) as batch:
        batch.remove(doc_id)
```

Tantivy 索引的添加通过多条路径触发，**不只有消费完成**：

1. **文档消费完成信号**（新文档或新增版本都会触发，区分发送对象）：信号连接定义在 [src/documents/apps.py L29](src/documents/apps.py#L29)
   ```python
   document_consumption_finished.connect(add_to_index)
   ```
   - 新文档消费：`document_consumption_finished` 发送**根文档**，`add_to_index` 按根文档 pk 加入索引
   - 新增版本消费：`document_consumption_finished` 发送**版本文档**，`add_to_index` 按版本文档 pk 加入索引（但前端搜索层通过 `filter(root_document__isnull=True)` 过滤排除）

2. **REST API 单文档 PUT/PATCH**：[src/documents/views.py L1176](src/documents/views.py#L1176) 显式调用 `get_backend().add_or_update(doc)`

3. **备注增删 API**：[src/documents/views.py L1660](src/documents/views.py#L1660) 和 [L1704](src/documents/views.py#L1704) 显式调用

4. **`bulk_update_documents` 异步任务**：[src/documents/tasks.py L268-L272](src/documents/tasks.py#L268-L272) 批量 `batch.add_or_update(doc)`

5. **后台管理保存**：[src/documents/admin.py L116-L119](src/documents/admin.py#L116-L119) 显式 `get_backend().add_or_update(obj)`

6. **`document_index reindex` 管理命令**：全量重建

`add_to_index` 的实现见 [src/documents/signals/handlers.py L794-L800](src/documents/signals/handlers.py#L794-L800)。

**恢复文档后**：代码中没有任何地方在 `restore()` 后调用 `get_backend().add_or_update()`，因此恢复后文档不会立刻出现在搜索结果中。

### 6.3 LLM 向量索引

LLM 索引的删除信号**始终注册**，不受 `empty_trash` 控制，但有启用前提条件：

[src/documents/signals/handlers.py L1342-L1355](src/documents/signals/handlers.py#L1342-L1355)：

```python
@receiver(models.signals.post_delete, sender=Document)
def delete_document_from_llm_index(sender, instance, **kwargs) -> None:
    ai_config = AIConfig()
    if ai_config.llm_index_enabled:  # ⭐ 仅当启用时才执行
        from documents.tasks import remove_document_from_llm_index
        remove_document_from_llm_index.apply_async(kwargs={"document": instance})
```

异步任务 `remove_document_from_llm_index` 定义在 [src/documents/tasks.py L652-L653](src/documents/tasks.py#L652-L653)，直接调用 `llm_index_remove_document(document)`。

**注意**：
- 无论软删除还是硬删除，只要触发了 `post_delete` 且 `AIConfig.llm_index_enabled` 为真，就会执行异步移除
- 和 Tantivy 索引一样，`restore()` 后也不会自动重新加入 LLM 索引
- LLM 索引的添加同样只在 `document_consumption_finished` 信号触发（[src/documents/apps.py L31](src/documents/apps.py#L31)），处理函数见 [src/documents/signals/handlers.py L1331-L1339](src/documents/signals/handlers.py#L1331-L1339)

---

## 7. 关联资源影响分析

### 7.1 外键与 M2M 关系全景图

以下模型关系定义在 [src/documents/models.py](src/documents/models.py) 中：

| 关联对象 | 字段定义位置 | 关系类型 | `on_delete` | 软删除影响 | 硬删除影响 |
|---------|------------|---------|------------|----------|----------|
| Correspondent | [L160-L167](src/documents/models.py#L160-L167) | ForeignKey | `SET_NULL` | 不变（只是 Document 软删） | Document 外键置 NULL，往来人本身保留 |
| StoragePath | [L169-L176](src/documents/models.py#L169-L176) | ForeignKey | `SET_NULL` | 不变 | Document 外键置 NULL，存储路径保留 |
| DocumentType | [L180-L187](src/documents/models.py#L180-L187) | ForeignKey | `SET_NULL` | 不变 | Document 外键置 NULL，文档类型保留 |
| Tag | [L209-L214](src/documents/models.py#L209-L214) | ManyToMany | 中间表 | 中间表记录不变 | 中间表记录级联删除，Tag 本身保留 |
| Note | [L841-L848](src/documents/models.py#L841-L848) | ForeignKey | `CASCADE` | django-softdelete 级联软删除 Note | Django ORM CASCADE 硬删除 Note |
| ShareLink | [L896-L902](src/documents/models.py#L896-L902) | ForeignKey | `CASCADE` | 级联软删除 ShareLink | 级联硬删除 ShareLink |
| CustomFieldInstance | [L1119-L1135](src/documents/models.py#L1119-L1135) | ForeignKey | `CASCADE` | 级联软删除 | 级联硬删除 |
| Version Document | [L312-L319](src/documents/models.py#L312-L319) | ForeignKey (`root_document`) | `CASCADE` | Document.delete() 显式级联软删除 | Django ORM CASCADE 硬删除 |
| ShareLinkBundle | [L1004-L1008](src/documents/models.py#L1004-L1008) | ManyToMany | 中间表 | 中间表不变 | 中间表记录级联删除，Bundle 本身保留 |

### 7.2 级联软删除的实现机制与证据

Note / ShareLink / CustomFieldInstance 均继承 `SoftDeleteModel`，且外键声明为 `on_delete=models.CASCADE`。

- **软删除时**：`django-softdelete` 库内部收集所有指向被删对象的、关联模型也继承了 `SoftDeleteModel` 的 ForeignKey 关系，逐个调用关联对象的 `delete()`（也是软删除）。这就是 Document 软删除后，Note/ShareLink/CustomFieldInstance 也会被打上 `deleted_at` 的原因。
- **硬删除时**（empty_trash）：走标准 Django ORM 的 `CASCADE`，直接从数据库级联 DELETE。
- **恢复时**：`doc.restore(strict=False)` 同样会级联恢复所有关联软删除的 Note/ShareLink/CustomFieldInstance。

**代码证据**：测试用例 [test_export_import_soft_deleted_document](src/documents/tests/test_management_exporter.py#L978-L1022) 在 L995 调用 `self.d1.delete()` 软删除文档后，在 L1016-L1022 验证 Note 和 CustomFieldInstance 的 `deleted_at` 均不为 None：

```python
# src/documents/tests/test_management_exporter.py L1012-L1022
reimported_note = Note.global_objects.get(pk=self.note.pk)
self.assertIsNotNone(reimported_note.deleted_at)  # Note 也被软删除

reimported_cfi = CustomFieldInstance.global_objects.get(pk=self.cfi1.pk)
self.assertIsNotNone(reimported_cfi.deleted_at)   # CustomFieldInstance 也被软删除
```

### 7.3 其他关联影响汇总

| 系统 | 触发时机 | 行为 | 代码位置 |
|-----|---------|------|---------|
| Tantivy 搜索索引 | 软删除时（单删/批删） | `get_backend().remove()` / `batch.remove()` | [src/documents/views.py L1207](src/documents/views.py#L1207)、[src/documents/bulk_edit.py L377-L381](src/documents/bulk_edit.py#L377-L381) |
| LLM 向量索引 | 软删除和硬删除（post_delete 信号） | 异步 `remove_document_from_llm_index`（仅 `llm_index_enabled` 为真时） | [src/documents/signals/handlers.py L1342-L1355](src/documents/signals/handlers.py#L1342-L1355) |
| 审计日志 LogEntry | hard_delete（empty_trash 内） | `LogEntry.objects.filter(...).delete()` | [src/documents/tasks.py L420-L425](src/documents/tasks.py#L420-L425) |
| 磁盘文件 | hard_delete（empty_trash 内） | 删除/移动原文件、归档、缩略图，清理空目录 | [src/documents/signals/handlers.py L342-L402](src/documents/signals/handlers.py#L342-L402) |
| WebSocket 通知 | 批量软删除 | `send_documents_deleted(delete_ids)` | [src/documents/bulk_edit.py L383-L384](src/documents/bulk_edit.py#L383-L384) |

---

## 8. Archive（归档）的双含义澄清

Paperless-ngx 中「archive」一词有两种完全不同的含义，切勿混淆：

### 8.1 Archive 文件（PDF/A 归档副本）

- 指 OCR 后生成的 PDF/A 格式长期归档副本
- 存储目录：`settings.ARCHIVE_DIR`
- 对应 Document 字段：`archive_filename`、`archive_checksum`、`archive_path`（property）、`has_archive_version`
- 生命周期：和原文件完全一致，软删除保留，硬删除时直接 `unlink()`（即使配置了 `EMPTY_TRASH_DIR` 也只会移动原文件，归档 PDF/A 始终直接删除）

### 8.2 Archive Serial Number（ASN 物理归档编号）

- 表示文档在物理档案盒中的编号，`Document.archive_serial_number` 字段（PositiveIntegerField，唯一约束）
- 只在合并/分割/替换文档的流程中涉及「释放」和「恢复」，与 Trash 生命周期无直接关系
- 相关函数：
  - [release_archive_serial_numbers()](src/documents/bulk_edit.py#L65-L78)：清空待替换文档的 ASN，返回 `{doc_id: old_asn}` 备份
  - [restore_archive_serial_numbers()](src/documents/bulk_edit.py#L81-L88)：用备份恢复 ASN（替换失败回滚时使用）

---

## 9. 关键配置项

| 配置项 | 作用 | 默认值 |
|--------|------|--------|
| `EMPTY_TRASH_DELAY` | 回收站自动清空的文档保留期限（天）。定时任务触发时仅删除 `deleted_at` 早于 `now - EMPTY_TRASH_DELAY` 的文档 | 30 |
| `EMPTY_TRASH_DIR` | 若设置，则 `empty_trash` 时将原文件移动到此目录（自动重命名避免冲突），而非直接 `unlink()`。归档文件和缩略图仍直接删除 | `None` |
| `PAPERLESS_EMPTY_TRASH_TASK_CRON` | 清空回收站定时任务的 Cron 表达式 | `"0 1 * * *"`（每天 01:00） |
| `ORIGINALS_DIR` | 原始文件存储根目录，用于空目录清理的上边界 | — |
| `ARCHIVE_DIR` | PDF/A 归档文件存储根目录 | — |
| `THUMBNAIL_DIR` | 缩略图存储目录 | — |
| `MEDIA_LOCK` | 文件操作锁文件路径，保证跨进程文件操作原子性 | — |
| `AUDIT_LOG_ENABLED` | 是否启用审计日志；启用时 empty_trash 会同步清理对应 LogEntry | — |

---

## 10. 常见疑问解答（基于代码证据）

**Q：软删除后文件还在磁盘上吗？**
A：在。软删除仅设置 `deleted_at` 时间戳和更新搜索/LLM 索引，完全不触碰磁盘文件。只有 `empty_trash` 执行硬删除时才会在 `post_delete` 信号中调用 `cleanup_document_deletion` 清理文件。单元测试 [test_document_soft_delete](src/documents/tests/test_document_model.py#L65-L103) 显式 mock `Path.unlink` 验证软删除不会调用 unlink。

**Q：为什么 `cleanup_document_deletion` 不直接在 apps.py ready() 中注册到 post_delete？**
A：因为 `django-softdelete` 的软删除内部也会触发 Django 的 `post_delete` 信号（用于实现级联软删除）。如果默认注册，软删除时文件就会被误删，回收站功能就失去了意义。所以采用「在 `empty_trash` 内临时连接、try/finally 确保断开」的技巧。函数上方的注释 `# see empty_trash in documents/tasks.py for signal handling` 也明确了这一约定。

**Q：恢复文档后，搜索能立刻搜到吗？LLM 问答能用到吗？**
A：不能。代码证据：
1. Tantivy 索引添加有 6 条路径（消费完成信号、REST PUT、备注增删、批量编辑、Admin 保存、手动重建命令），但 **`restore()` 本身不触发任何一条。其中 `add_to_index` 处理函数只在 `document_consumption_finished` 信号连接（[src/documents/apps.py L29](src/documents/apps.py#L29)）
2. LLM 索引添加同样主要在 `document_consumption_finished` 信号连接（[src/documents/apps.py L31](src/documents/apps.py#L31)），此外只有 `bulk_update_documents` 会条件性更新 LLM
3. `document_updated` 信号没有连接任何索引更新函数（[src/documents/apps.py L32-L33](src/documents/apps.py#L32-L33)）
4. [TrashView.post()](src/documents/views.py#L5116-L5118) 的 restore 分支中没有任何显式索引调用

需要靠后续触发索引重建的事件（如编辑保存文档、批量编辑）或手动调用索引管理命令。

**Q：删除根文档时，版本文档怎么办？**
A：会被一并软删除。[Document.delete()](src/documents/models.py#L503-L514) 在检测到 `root_document_id is None`（即当前是根文档）时，显式 `Document.objects.filter(root_document=self).delete()` 软删除所有版本。硬删除时则由 `root_document` 外键的 `on_delete=models.CASCADE` 自动级联。测试见 [test_delete_root_deletes_versions](src/documents/tests/test_document_model.py#L105-L125)。

**Q：工作流中执行 move_to_trash 之后的动作还会执行吗？**
A：不会。[run_workflows](src/documents/signals/handlers.py#L907-L913) 循环在每个动作后检查 `document.is_deleted`，若为真则 `break` 跳出，后续动作全部跳过。

**Q：Note / ShareLink / CustomFieldInstance 在文档软删除时会怎样？**
A：会被 `django-softdelete` 级联软删除（同样打上 `deleted_at`）。因为它们都继承了 `SoftDeleteModel`，且外键 `on_delete=CASCADE`。恢复文档时也会被级联恢复。硬删除时则由 Django ORM CASCADE 真正从数据库删除。单元测试 [test_export_import_soft_deleted_document](src/documents/tests/test_management_exporter.py#L978-L1022) 显式验证了软删除后 Note 和 CustomFieldInstance 的 `deleted_at` 均被设置。

**Q：配置了 EMPTY_TRASH_DIR 后，所有文件都会移动到那里吗？**
A：不是。只有 `source_path`（原文件）会被 `shutil.move` 到 `EMPTY_TRASH_DIR`（文件名冲突时自动追加 `_01`、`_02`…）。`archive_path`（PDF/A）和 `thumbnail_path`（缩略图）始终直接 `unlink()` 删除。代码见 [src/documents/signals/handlers.py L344-L379](src/documents/signals/handlers.py#L344-L379)。

**Q：回收站的 API 路由是什么？DELETE 文档和 POST /api/trash/ 有什么区别？**
A：回收站 API 是 `POST /api/trash/`，路由定义在 [src/paperless/urls.py L257-L259](src/paperless/urls.py#L257-L259)，视图类为 [TrashView](src/documents/views.py#L5083-L5123)。区别：
- `DELETE /api/documents/<id>/` → 软删除，文档进入回收站，磁盘文件保留
- `POST /api/trash/` body `{"action":"restore"}` → 从回收站恢复
- `POST /api/trash/` body `{"action":"empty"}` → 永久删除，清理磁盘文件
