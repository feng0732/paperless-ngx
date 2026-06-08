# Paperless-ngx 文档版本控制与 OCR 重跑代码实现梳理

## 概述

本文档从代码实现角度梳理 Paperless-ngx 中三大核心流程：
1. **版本保存** — 文档上传新版本、旋转、合并、分割等操作时的版本管理
2. **文本重算** — OCR 重跑（Reprocess）时的文本解析与归档文件重建
3. **文件状态更新** — 文档变更后文件名生成、文件移动、搜索索引更新、WebSocket 通知等

所有文件路径均以 `src/documents/` 为基准（项目根目录下）。

---

## 一、核心数据模型

### 1.1 Document 模型中的版本字段

文件位置：`models.py`

```python
class Document(SoftDeleteModel, ModelWithOwner):
    root_document = models.ForeignKey(
        "self",
        blank=True,
        null=True,
        related_name="versions",
        on_delete=models.CASCADE,
    )
    version_index = models.PositiveIntegerField(blank=True, null=True, db_index=True)
    version_label = models.CharField(max_length=64, blank=True, null=True)
```

关键约束：
- `(root_document, version_index)` 组合唯一
- `root_document=None` 表示这是根文档（原始文档）
- `version_index` 从 1 开始递增，单调不重复（即使中间版本被删除，也不会复用其索引）

### 1.2 ConsumableDocument — 消费输入载体

文件位置：`data_models.py`

```python
@dataclasses.dataclass
class ConsumableDocument:
    source: DocumentSource
    original_file: Path
    root_document_id: int | None = None  # 关键：非 None 表示创建新版本
    original_path: Path | None = None
    mailrule_id: int | None = None
```

`root_document_id` 是判断"创建新版本"还是"创建全新文档"的唯一分水岭。

### 1.3 DocumentMetadataOverrides — 元数据覆盖

文件位置：`data_models.py`

```python
@dataclasses.dataclass
class DocumentMetadataOverrides:
    filename: str | None = None
    title: str | None = None
    correspondent_id: int | None = None
    # ...
    version_label: str | None = None  # 版本标签
    actor_id: int | None = None        # 操作者（用于审计日志）
```

### 1.4 PaperlessTask — 任务追踪

文件位置：`models.py`

```python
class TaskType(models.TextChoices):
    CONSUME_FILE = "consume_file"
    REPROCESS_DOCUMENT = "reprocess_document"
    BULK_UPDATE = "bulk_update"
```

---

## 二、PDF 操作分类：哪些会创建新版本

判断原则：**只要 `ConsumableDocument(root_document_id=...)` 传了非空值就创建新版本；不传就创建全新文档。**

### 2.1 操作分类总表

| 操作 | 函数位置 | 创建新版本？ | 说明 |
|------|---------|------------|------|
| **旋转 PDF** | `bulk_edit.py` `rotate()` | ✅ 总是 | 始终 `root_document_id=pair.root_doc.id` |
| **删除页面** | `bulk_edit.py` `delete_pages()` | ✅ 总是 | 始终 `root_document_id=pair.root_doc.id` |
| **编辑 PDF** | `bulk_edit.py` `edit_pdf()` | ✅ 仅当 `update_document=True` | `update_document=False` 时创建全新文档（可选删除原文档） |
| **移除密码** | `bulk_edit.py` `remove_password()` | ✅ 仅当 `update_document=True` | `update_document=False` 时创建全新文档（可选删除原文档） |
| **合并 PDF** | `bulk_edit.py` `merge()` | ❌ 不创建 | 始终创建**全新独立文档**，可选 `delete_originals=True` 删除原文档 |
| **分割 PDF** | `bulk_edit.py` `split()` | ❌ 不创建 | 始终创建**多个全新独立文档**，可选 `delete_originals=True` 删除原文档 |
| **API 上传新版本** | `views.py` `update_version()` | ✅ 总是 | 用户显式上传新版本 |

### 2.2 创建新版本的操作详细说明

#### 2.2.1 旋转（rotate）

`bulk_edit.py` 中 `rotate()` 对每个根文档生成旋转后的临时 PDF，然后：

```python
consume_file.apply_async(
    kwargs={
        "input_doc": ConsumableDocument(
            source=DocumentSource.ConsumeFolder,
            original_file=filepath,
            root_document_id=pair.root_doc.id,  # ← 绑定根文档
        ),
        "overrides": overrides,  # 从根文档继承 title/correspondent/tags 等
    },
    headers={"trigger_source": trigger_source},
)
```

#### 2.2.2 删除页面（delete_pages）

与旋转完全相同的模式：临时文件 + `root_document_id=pair.root_doc.id` + 继承元数据。

#### 2.2.3 编辑 PDF（edit_pdf）

- **`update_document=True`**：单输出文档，创建新版本（`root_document_id=pair.root_doc.id`）
- **`update_document=False`（默认）**：生成一个或多个全新独立文档，通过 `delete_original=True` 可选删除原文档

代码中显式校验：`update_document and len(pdf_docs) > 1` 会抛 `ValueError`——因为多输出无法对应到单一版本链。

#### 2.2.4 移除密码（remove_password）

与 `edit_pdf` 相同的双模式：
- `update_document=True` → 创建新版本
- `update_document=False`（默认）→ 创建全新文档，可选 `delete_original=True`

### 2.3 不创建新版本的操作

#### 2.3.1 合并（merge）

`ConsumableDocument` 构造时**不设置 `root_document_id`**，产生一个与原文档完全独立的新 Document。
如果设置了 `delete_originals=True`，使用 Celery 的 `link=[delete.si(affected_docs)]`——因为合并只有**单个**输出文档，`consume_task` 是单个 signature，消费成功后直接链式执行删除。

#### 2.3.2 分割（split）

每个切片都生成独立的新 Document（`title` 自动追加 `(split N)`），同样不设 `root_document_id`。
`delete_originals=True` 时使用 Celery `chord(header=consume_tasks, body=delete.si([doc.id]))`——因为分割会产生**多个**消费任务，需要等待所有切片消费完成后才执行删除。

#### 2.3.3 各操作删除原文档的 Celery 原语与回滚链路对比

判断原则：**输出只有 1 个文档用 `link`，输出多个文档用 `chord`**。

| 操作 | Celery 原语 | release ASN（backup） | link_error 回滚 | try/except 兜底 | 原因 |
|------|-----------|----------------------|----------------|----------------|------|
| 合并（merge） | `consume_task.apply_async(link=[delete.si(...)])` | ✅ | ✅ | ✅ | 只有 1 个合并输出文档；需把原文档 ASN 转移给新合并文档 |
| 分割（split） | `chord(header=consume_tasks, body=delete.si(...))` | ✅ | ✅ | ✅ | N 个切片输出，需全部成功后再删；需释放原文档 ASN |
| 编辑 PDF（edit_pdf，`update_document=False`） | `chord(header=consume_tasks, body=delete.si(...))` | ✅ | ✅ | ✅ | 可输出多个文档，需全部成功后再删；需释放原文档 ASN |
| 移除密码（remove_password，`update_document=False`） | `chord(header=consume_tasks, body=delete.si(...))` | ❌ | ❌ | ❌ | 不涉及 ASN 变更；即使消费失败原文档也未被修改，无需回滚 |

#### 2.3.4 ASN 回滚链路详解（仅 merge / split / edit_pdf）

三个涉及 ASN 转移/释放的操作共享完全相同的三段式回滚模式：

**第一步：同步释放 ASN 并获取 backup**

```python
backup = release_archive_serial_numbers(affected_docs)
```

- `release_archive_serial_numbers()` 将原文档的 `archive_serial_number` 置为 NULL，
  并返回原始 ASN 列表（"backup"）供回滚时恢复。
- 此操作在当前请求线程**同步执行**——如果后续 Celery 任务投递失败，需要立即手动恢复。

**第二步：配置 Celery link_error（异步失败回滚）**

```python
link_error=[restore_archive_serial_numbers_task.s(backup)]
```

- 如果 Celery worker 侧消费任务执行失败，`link_error` 会触发 `restore_archive_serial_numbers_task`，
  把 backup 中记录的 ASN 重新写回原文档。

**第三步：try/except 兜底（任务投递失败回滚）**

```python
try:
    consume_task.apply_async(link=..., link_error=...)
except Exception:
    restore_archive_serial_numbers(backup)   # 同步恢复
    raise
```

- 如果 `apply_async()` 本身抛异常（例如 Broker 不可达），任务根本没进队列，
  `link_error` 也不会触发，此时由同步的 `restore_archive_serial_numbers(backup)` 兜底。

#### 2.3.5 remove_password 为什么不需要回滚

`remove_password` 删除原文档时既不调 `release_archive_serial_numbers`，也不配置 `link_error`：

- **ASN 不变**：移除密码只是输出一份无密码副本，原文档和新文档的 ASN 没有转移关系，不需要释放/恢复。
- **原文档未改动**：即使消费任务失败，原文档（带密码）依然完整存在于数据库和磁盘，没有任何副作用需要回滚。
- **失败代价**：最坏情况是临时文件残留在 `SCRATCH_DIR`，不影响业务数据一致性。

---

## 三、版本保存流程

### 3.1 版本创建的触发入口

新版本创建均通过构造 `ConsumableDocument(root_document_id=X)` 并调用 `consume_file` 任务实现，主要入口：

| 操作 | 入口文件 | 关键位置 |
|------|---------|---------|
| API 上传新版本 | `views.py` | `update_version()` 方法 |
| 旋转 PDF | `bulk_edit.py` | `rotate()` |
| 删除页面 | `bulk_edit.py` | `delete_pages()` |
| 编辑 PDF（更新模式） | `bulk_edit.py` | `edit_pdf(update_document=True)` |
| 移除密码（更新模式） | `bulk_edit.py` | `remove_password(update_document=True)` |

### 3.2 版本创建核心流程

**第一步：API/操作层 → 构造 ConsumableDocument**

以 `update_version` 为例（`views.py`）：

```python
input_doc = ConsumableDocument(
    source=DocumentSource.ApiUpload,
    original_file=temp_file_path,
    root_document_id=root_doc.pk,  # 关键：指向根文档
)
overrides = DocumentMetadataOverrides()
overrides.version_label = version_label.strip()
overrides.actor_id = request.user.id

consume_file.apply_async(
    kwargs={"input_doc": input_doc, "overrides": overrides},
    headers={"trigger_source": PaperlessTask.TriggerSource.WEB_UI},
)
```

**第二步：consume_file 任务 → 插件链执行**

文件位置：`tasks.py`

```python
@shared_task(bind=True)
def consume_file(self, input_doc, overrides=None):
    # 有 root_document_id 时走精简插件链（跳过条码、ASN、工作流预触发等）
    plugins = (
        [ConsumerPreflightPlugin, ConsumerPlugin]  # 版本模式
        if input_doc.root_document_id is not None
        else [ConsumerPreflightPlugin, AsnCheckPlugin, CollatePlugin,
              BarcodePlugin, AsnCheckPlugin, WorkflowTriggerPlugin, ConsumerPlugin]
    )
```

**第三步：ConsumerPlugin._create_version_from_root — 创建版本 Document**

文件位置：`consumer.py`

```python
def _create_version_from_root(self, root_doc, *, text, page_count, mime_type):
    root_doc_frozen = Document.objects.select_for_update().get(pk=root_doc.pk)
    # 查询当前最大 version_index（含已软删除的版本）
    next_version_index = (
        Document.global_objects.filter(root_document_id=root_doc_frozen.pk)
        .aggregate(max_index=Max("version_index"))["max_index"]
        or 0
    )
    version_doc = Document(
        root_document=root_doc_frozen,
        version_index=next_version_index + 1,
        checksum=compute_checksum(file_for_checksum),
        content=text or "",
        page_count=page_count,
        mime_type=mime_type,
        original_filename=self.filename,
        owner_id=root_doc_frozen.owner_id,
        created=root_doc_frozen.created,     # 继承根文档创建日期
        title=root_doc_frozen.title,         # 继承根文档标题
        added=timezone.now(),
        modified=timezone.now(),
    )
    if self.metadata.version_label is not None:
        version_doc.version_label = self.metadata.version_label
    return version_doc
```

**第四步：ConsumerPlugin.run — 保存版本并写入文件**

文件位置：`consumer.py`

```python
with transaction.atomic():
    if self.input_doc.root_document_id:
        root_doc = Document.objects.get(pk=self.input_doc.root_document_id)
        original_document = self._create_version_from_root(root_doc, text=text, ...)
        # 审计日志：记录操作人
        if settings.AUDIT_LOG_ENABLED and self.metadata.actor_id:
            actor = User.objects.filter(pk=self.metadata.actor_id).first()
            with set_actor(actor):
                original_document.save()
        else:
            original_document.save()
        # 审计日志：在根文档上记录"Version Added"
        LogEntry.objects.log_create(
            instance=root_doc,
            changes={"Version Added": ["None", original_document.id]},
            action=LogEntry.Action.UPDATE,
            additional_data={"reason": "Version added", "version_id": original_document.id},
        )
        document = original_document
    else:
        document = self._store(text=text, ...)  # 新文档路径

    # 发送 document_consumption_finished 信号（触发索引、标签等）
    document_consumption_finished.send(sender=self.__class__, document=document, ...)

    # 写入原文件、缩略图、归档文件到磁盘
    with FileLock(settings.MEDIA_LOCK):
        document.filename = generate_unique_filename(document)
        create_source_path_directory(document.source_path)
        self._write(working_copy, document.source_path)
        self._write(thumbnail, document.thumbnail_path)
        if archive_path:
            document.archive_filename = generate_unique_filename(document, archive_filename=True)
            self._write(archive_path, document.archive_path)
            document.archive_checksum = compute_checksum(document.archive_path)

    document.save()  # 触发 post_save → update_filename_and_move_files

    # 关键：版本保存成功后，在根文档上发送 document_updated 信号
    if document.root_document_id:
        document_updated.send(sender=self.__class__, document=document.root_document)
```

### 3.3 版本读取与解析

文件位置：`versioning.py`

| 函数 | 作用 |
|------|------|
| `get_root_document(doc)` | 获取根文档（doc.root_document_id 为 None 则返回自身） |
| `get_latest_version_for_root(root_doc)` | 按 id 倒序取最新版本 |
| `resolve_requested_version_for_root(root_doc, request)` | 解析 URL `?version=ID` 参数 |
| `resolve_effective_document(request_doc, request)` | 综合逻辑：有 version 参数取指定版本，否则对根文档取最新版本 |

根文档的 `get_effective_content()`（`models.py`）会自动取最新版本的 content 用于搜索和建议。

### 3.4 版本删除

文件位置：`views.py`

```python
def delete_version(self, request, pk=None, version_id=None):
    # 不能删除根文档自身
    if version_doc.id == root_doc.id:
        return HttpResponseBadRequest("Cannot delete the root/original version.")
    _backend.remove(version_doc.pk)     # 从搜索索引移除
    version_doc.delete()                 # 软删除
    _backend.add_or_update(root_doc)     # 重新索引根文档
    # 审计日志：记录"Version Deleted"
    document_updated.send(sender=self.__class__, document=root_doc)
```

---

## 四、OCR 重跑（文本重算）流程

### 4.1 入口

| 入口 | 位置 |
|------|------|
| Web API `/api/documents/reprocess/` | `views.py` `ReprocessDocumentsView` |
| bulk_edit.reprocess() | `bulk_edit.py` |
| 管理命令 `document_archiver` | `management/commands/document_archiver.py` |

### 4.2 bulk_edit.reprocess

```python
def reprocess(doc_ids: list[int]) -> Literal["OK"]:
    for document_id in doc_ids:
        update_document_content_maybe_archive_file.apply_async(
            kwargs={"document_id": document_id},
            headers={"trigger_source": PaperlessTask.TriggerSource.MANUAL},
        )
    return "OK"
```

**注意：** Reprocess **不创建新版本**，它直接在原 Document 记录上更新 content、archive_checksum、archive_filename。

### 4.3 update_document_content_maybe_archive_file 核心实现

文件位置：`tasks.py`

```python
@shared_task
def update_document_content_maybe_archive_file(document_id) -> None:
    document = Document.objects.get(id=document_id)
    mime_type = document.mime_type

    # 1. 获取对应的 parser（Tesseract、PDF 解析器等）
    parser_class = get_parser_registry().get_parser_for_file(
        mime_type, document.original_filename or "", document.source_path,
    )

    with parser_class() as parser:
        parser.configure(ParserContext())
        produce_archive = should_produce_archive(parser, mime_type, document.source_path)

        # 2. 重新解析（OCR/文本提取）
        parser.parse(document.source_path, mime_type, produce_archive=produce_archive)
        thumbnail = parser.get_thumbnail(document.source_path, mime_type)

        with transaction.atomic():
            oldDocument = Document.objects.get(pk=document.pk)
            if parser.get_archive_path():
                # 3a. 有归档文件：更新 content + archive_checksum + archive_filename
                checksum = compute_checksum(parser.get_archive_path())
                document.archive_filename = generate_unique_filename(
                    document, archive_filename=True,
                )
                Document.objects.filter(pk=document.pk).update(
                    archive_checksum=checksum,
                    content=parser.get_text(),
                    archive_filename=document.archive_filename,
                )
                # 审计日志
                LogEntry.objects.log_create(instance=oldDocument, changes={...},
                    additional_data={"reason": "Update document content"}, ...)
            else:
                # 3b. 无归档文件：仅更新 content
                Document.objects.filter(pk=document.pk).update(content=parser.get_text())

            # 4. 替换归档文件和缩略图
            with FileLock(settings.MEDIA_LOCK):
                if parser.get_archive_path():
                    create_source_path_directory(document.archive_path)
                    shutil.move(parser.get_archive_path(), document.archive_path)
                shutil.move(thumbnail, document.thumbnail_path)

        document.refresh_from_db()

        # 5. 更新搜索索引
        from documents.search import get_backend
        get_backend().add_or_update(document)

        # 6. 更新 LLM 索引（如启用）
        ai_config = AIConfig()
        if ai_config.llm_index_enabled:
            llm_index_add_or_update_document(document)

        # 7. 清除缓存
        clear_document_caches(document.pk)
```

### 4.4 OCR 原地更新后的索引、缓存、信号相互关系

这是最关键的细节：**`update_document_content_maybe_archive_file` 刻意避开了 Django 的信号机制，所有更新动作手动执行。**

#### 4.4.1 为什么没有触发 post_save？

代码使用的是 `Document.objects.filter(pk=document.pk).update(...)` 而非 `document.save()`。
Django ORM 的 `QuerySet.update()` **不会**发出 `post_save` / `pre_save` 信号，这是有意为之：
- 避免 `update_filename_and_move_files` 被触发（归档文件和源文件还没落盘，此时重命名会出错）
- 避免 LLM 建议缓存失效逻辑被重复触发

#### 4.4.2 手动更新链路详解

```
update_document_content_maybe_archive_file(document_id)
        │
        ├──► QuerySet.update(...)  ──► 直接更新 DB，不触发任何 Django 信号
        │       │
        │       ├── 更新 content（OCR 重新提取的文本）
        │       ├── 更新 archive_checksum / archive_filename（若生成了归档 PDF）
        │       └── 写审计日志 LogEntry（reason="Update document content"）
        │
        ├──► 文件落盘（FileLock 内）
        │       ├── 归档 PDF 移至 document.archive_path
        │       └── 缩略图移至 document.thumbnail_path
        │
        ├──► Tantivy 全文搜索索引
        │       │   文件：search/_backend.py
        │       │   入口：get_backend().add_or_update(document)
        │       │   实现：先 remove(doc_id) 再 add_document(...) 实现 upsert
        │       └── 索引字段含 title、content、tags、correspondent、type、created 等
        │
        ├──► LLM 向量索引（条件执行）
        │       │   条件：AIConfig().llm_index_enabled == True
        │       │   入口：llm_index_add_or_update_document(document)
        │       └── 基于 document.content 重新计算 embedding 并写入向量库
        │
        └──► 缓存清除
                │   入口：clear_document_caches(document.pk)
                │   文件：caching.py
                └── 删除三个 cache key：
                    ├── doc_{id}_suggest          （LLM 自动建议缓存，对应 get_suggestion_cache_key()）
                    ├── doc_{id}_metadata         （元数据缓存，对应 get_metadata_cache_key()）
                    └── doc_{id}_thumbnail_modified （缩略图时间戳缓存，对应 get_thumbnail_modified_key()）
```

#### 4.4.3 哪些链路**没有**被触发？

由于既不发 `post_save` 也不发 `document_updated`，以下流程在 OCR 重跑后**不会执行**：

| 未触发的动作 | 正常触发位置 | 影响 |
|-------------|------------|------|
| `update_filename_and_move_files` | `Document.post_save` | 文件名不会根据新 content 自动重命名（合理，因为 reprocess 不改 title/tags 等命名模板依赖字段） |
| `update_llm_suggestions_cache` | `Document.post_save` | 该函数同样是调 `invalidate_llm_suggestions_cache()`，但 `clear_document_caches` 已手动清除了建议缓存，**实际效果一致** |
| `run_workflows_updated` | `document_updated` 信号 | **DOCUMENT_UPDATED 类型的工作流不会运行**——这是一个有意的设计选择：OCR 重跑不视为"文档内容业务上的更新" |
| `send_websocket_document_updated` | `document_updated` 信号 | **前端不会收到 WebSocket 推送**；但 Celery 任务完成状态会通过任务追踪通知前端 |

#### 4.4.4 与 bulk_update_documents 的对比

`bulk_update_documents`（批量元数据修改任务）走完整链路：

```python
for doc in documents:
    clear_document_caches(doc.pk)
    document_updated.send(sender=None, document=doc, ...)  # 工作流 + WebSocket
    post_save.send(Document, instance=doc, created=False)  # 重命名 + LLM 缓存
```

而 OCR 重跑走**精简链路**：只更新数据库、索引、缓存，不触发工作流和 WebSocket。

### 4.5 与版本创建的区别

| 维度 | 版本创建 (consume_file + root_document_id) | OCR 重跑 (reprocess) |
|------|------------------------------------------|----------------------|
| 新 Document 记录 | 是，创建新行 | 否，原地更新 |
| version_index | 递增分配 | 不变 |
| content 更新 | 解析新文件得到 | 重新解析原 source_path |
| 归档文件 | 基于新文件生成 | 基于原 source_path 重新生成 |
| DB 更新方式 | `document.save()` → 触发 `post_save` | `QuerySet.update()` → **不触发**任何信号 |
| 搜索索引 | `document_consumption_finished` → `add_to_index` | 手动 `get_backend().add_or_update()` |
| LLM 索引 | `document_consumption_finished` → `add_or_update_document_in_llm_index` | 手动 `llm_index_add_or_update_document()` |
| 缓存清除 | `update_filename_and_move_files` 内 `clear_document_caches` | 手动 `clear_document_caches()` |
| 工作流 | `document_consumption_finished`（DOCUMENT_ADDED）+ `document_updated`（根文档 DOCUMENT_UPDATED） | ❌ 不触发任何工作流 |
| WebSocket 通知 | `document_updated` 触发 | ❌ 不发送（仅 Celery 任务状态通知） |
| 文件名重算 | `post_save` → `update_filename_and_move_files` | ❌ 不重算（合理：reprocess 不改命名模板字段） |
| 审计日志 | 根文档上记录 "Version Added" | 当前文档上记录 "Update document content" |

---

## 五、文件状态更新流程

文档保存/更新后触发一系列级联更新，主要通过 Django `post_save` 信号和自定义 `document_updated` / `document_consumption_finished` 信号驱动。

### 5.1 信号注册

文件位置：`apps.py`

```python
def ready(self) -> None:
    document_consumption_finished.connect(add_inbox_tags)
    document_consumption_finished.connect(set_correspondent)
    document_consumption_finished.connect(set_document_type)
    document_consumption_finished.connect(set_tags)
    document_consumption_finished.connect(set_storage_path)
    document_consumption_finished.connect(add_to_index)           # 搜索索引
    document_consumption_finished.connect(run_workflows_added)    # DOCUMENT_ADDED 工作流
    document_consumption_finished.connect(add_or_update_document_in_llm_index)
    document_updated.connect(run_workflows_updated)               # DOCUMENT_UPDATED 工作流
    document_updated.connect(send_websocket_document_updated)     # WebSocket 通知
```

### 5.2 update_filename_and_move_files — 文件名生成与文件移动

**触发时机**：`Document.post_save`、`Document.tags.m2m_changed`、`CustomFieldInstance.post_save`

文件位置：`signals/handlers.py`

```python
@receiver(models.signals.post_save, sender=CustomFieldInstance, weak=False)
@receiver(models.signals.m2m_changed, sender=Document.tags.through, weak=False)
@receiver(models.signals.post_save, sender=Document, weak=False)
def update_filename_and_move_files(sender, instance, **kwargs):
    # 若 instance.filename 为空（消费中首次 save）则跳过，
    # 等消费流程设置最终 filename 后再触发
    if not instance.filename:
        return

    with FileLock(settings.MEDIA_LOCK):
        instance.refresh_from_db()

        # 1. 根据命名模板生成候选文件名
        candidate_filename = generate_filename(instance)
        # 含版本号会加上 version_index 后缀
        # 超长则回退默认命名

        # 2. 冲突检测与去重
        candidate_source_path = (settings.ORIGINALS_DIR / candidate_filename).resolve()
        if candidate_source_path.exists() and candidate_source_path != old_source_path:
            new_filename = generate_unique_filename(instance)  # 追加数字后缀
        else:
            new_filename = candidate_filename

        # 3. 移动原文件
        if move_original:
            validate_move(instance, old_source_path, instance.source_path, settings.ORIGINALS_DIR)
            create_source_path_directory(instance.source_path)
            shutil.move(old_source_path, instance.source_path)

        # 4. 同样处理归档文件
        if move_archive:
            shutil.move(old_archive_path, instance.archive_path)

        # 5. 直接 UPDATE 数据库（不调用 save()，避免无限递归）
        Document.global_objects.filter(pk=instance.pk).update(
            filename=instance.filename,
            archive_filename=instance.archive_filename,
            modified=timezone.now(),
        )
        clear_document_caches(instance.pk)
```

**版本文件命名**：当 `document.root_document_id is not None` 时，`generate_filename` 会追加 `-v{version_index}` 后缀。

### 5.3 bulk_update_documents — 批量触发更新链

文件位置：`tasks.py`

```python
@shared_task
def bulk_update_documents(document_ids) -> None:
    documents = Document.objects.filter(id__in=document_ids)
    for doc in documents:
        clear_document_caches(doc.pk)
        document_updated.send(sender=None, document=doc, logging_group=uuid.uuid4())
        post_save.send(Document, instance=doc, created=False)  # 触发 update_filename_and_move_files
    # 批量更新搜索索引
    with get_backend().batch_update() as batch:
        for doc in documents:
            batch.add_or_update(doc)
    # 更新 LLM 索引
    if ai_config.llm_index_enabled:
        update_llm_index(rebuild=False)
```

### 5.4 document_updated 信号处理

文件位置：`signals/handlers.py`

```python
def run_workflows_updated(sender, document, logging_group=None, **kwargs):
    run_workflows(trigger_type=WorkflowTrigger.WorkflowTriggerType.DOCUMENT_UPDATED, ...)

def send_websocket_document_updated(sender, document, **kwargs):
    document.refresh_from_db()
    doc_overrides = DocumentMetadataOverrides.from_document(document)
    with DocumentsStatusManager() as status_mgr:
        status_mgr.send_document_updated(
            document_id=document.id,
            modified=DRF_DATETIME_FIELD.to_representation(document.modified),
            owner_id=doc_overrides.owner_id,
            users_can_view=doc_overrides.view_users,
            groups_can_view=doc_overrides.view_groups,
        )
```

### 5.5 document_consumption_finished 信号处理

该信号在新文档/新版本消费完成后触发，依次执行：
1. `add_inbox_tags` — 添加收件箱标签
2. `set_correspondent` — 自动匹配联系人
3. `set_document_type` — 自动匹配文档类型
4. `set_tags` — 自动匹配标签
5. `set_storage_path` — 自动匹配存储路径
6. `add_to_index` — 添加到 Tantivy 全文索引（使用 `get_effective_content()`）
7. `run_workflows_added` — 执行 DOCUMENT_ADDED 工作流
8. `add_or_update_document_in_llm_index` — 添加到 LLM 向量索引

### 5.6 整体状态更新时序图

```
文档保存（Document.save / Document.objects.update）
        │
        ▼
┌─────────────────────────────┐
│  Django post_save 信号      │
└─────────────┬───────────────┘
              │
     ┌────────┴─────────┐
     │                  │
     ▼                  ▼
update_filename_and_move_files   update_llm_suggestions_cache
（生成文件名、移动文件）        （失效 LLM 建议缓存）
     │
     ▼
  Document.save() 再次触发？
  否 — handler 内用 update() 避免递归
     │
     ▼
  消费流程主动发送 ──────► document_consumption_finished
     │                         │
     │                   ┌─────┴──────┐
     │                   │            │
     │                   ▼            ▼
     │              add_inbox_tags  set_correspondent
     │              set_document_type  set_tags
     │              set_storage_path  add_to_index
     │              run_workflows_added
     │              add_or_update_document_in_llm_index
     │
     └──────► 版本保存后主动发送 ──────► document_updated
                                               │
                                          ┌────┴────┐
                                          │         │
                                          ▼         ▼
                                 run_workflows_updated
                                 send_websocket_document_updated
```

---

## 六、Celery 任务追踪

文件位置：`signals/handlers.py`

通过 Celery 的 `before_task_publish`、`task_prerun`、`task_postrun` 信号，将每个异步任务记录到 `PaperlessTask` 模型中，便于前端展示进度。

被追踪的任务：
```python
TRACKED_TASKS = {
    "documents.tasks.consume_file": CONSUME_FILE,
    "documents.tasks.update_document_content_maybe_archive_file": REPROCESS_DOCUMENT,
    "documents.tasks.bulk_update_documents": BULK_UPDATE,
    # ...
}
```

状态流转：`PENDING → STARTED → SUCCESS / FAILURE / REVOKED`

---

## 七、关键文件索引

| 文件 | 作用 |
|------|------|
| `models.py` | Document、PaperlessTask 等数据模型 |
| `versioning.py` | 版本解析、根文档/最新版本获取 |
| `consumer.py` | 文档消费核心（ConsumerPlugin._create_version_from_root） |
| `tasks.py` | consume_file、update_document_content_maybe_archive_file、bulk_update_documents 等 Celery 任务 |
| `bulk_edit.py` | rotate/merge/split/delete_pages/edit_pdf/remove_password/reprocess 等批量操作入口 |
| `views.py` | API 层：update_version、delete_version、ReprocessDocumentsView 等 |
| `signals/handlers.py` | update_filename_and_move_files、工作流执行、WebSocket 通知、任务追踪 |
| `signals/__init__.py` | 自定义信号定义（document_consumption_finished、document_updated） |
| `apps.py` | 信号处理器注册 |
| `data_models.py` | ConsumableDocument、DocumentMetadataOverrides |
| `caching.py` | clear_document_caches 等缓存工具 |
| `search/_backend.py` | Tantivy 搜索索引的 add_or_update 实现 |
