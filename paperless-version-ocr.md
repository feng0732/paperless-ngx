# Paperless-ngx 文档版本控制与 OCR 重跑代码实现梳理

## 概述

本文档从代码实现角度梳理 Paperless-ngx 中三大核心流程：
1. **版本保存** — 文档上传新版本、旋转、合并、分割等操作时的版本管理
2. **文本重算** — OCR 重跑（Reprocess）时的文本解析与归档文件重建
3. **文件状态更新** — 文档变更后文件名生成、文件移动、搜索索引更新、WebSocket 通知等

---

## 一、核心数据模型

### 1.1 Document 模型中的版本字段

文件位置：[models.py](file:///d:/fz/0601/solo-dogfeeding/code/114-paperless-ngx/src/documents/models.py#L157-L516)

```python
class Document(SoftDeleteModel, ModelWithOwner):
    # ...
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

关键约束（[models.py#L341-L349](file:///d:/fz/0601/solo-dogfeeding/code/114-paperless-ngx/src/documents/models.py#L341-L349)）：
- `(root_document, version_index)` 组合唯一
- `root_document=None` 表示这是根文档（原始文档）
- `version_index` 从 1 开始递增，单调不重复（即使中间版本被删除）

### 1.2 ConsumableDocument — 消费输入载体

文件位置：[data_models.py](file:///d:/fz/0601/solo-dogfeeding/code/114-paperless-ngx/src/documents/data_models.py#L161-L188)

```python
@dataclasses.dataclass
class ConsumableDocument:
    source: DocumentSource
    original_file: Path
    root_document_id: int | None = None  # 关键：非 None 表示创建新版本
    original_path: Path | None = None
    mailrule_id: int | None = None
```

### 1.3 DocumentMetadataOverrides — 元数据覆盖

文件位置：[data_models.py](file:///d:/fz/0601/solo-dogfeeding/code/114-paperless-ngx/src/documents/data_models.py#L12-L148)

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

文件位置：[models.py](file:///d:/fz/0601/solo-dogfeeding/code/114-paperless-ngx/src/documents/models.py#L664-L816)

```python
class TaskType(models.TextChoices):
    CONSUME_FILE = "consume_file"
    REPROCESS_DOCUMENT = "reprocess_document"
    BULK_UPDATE = "bulk_update"
    # ...
```

---

## 二、版本保存流程

### 2.1 版本创建的触发入口

新版本创建均通过构造 `ConsumableDocument(root_document_id=X)` 并调用 `consume_file` 任务实现，主要入口：

| 操作 | 入口文件 | 关键位置 |
|------|---------|---------|
| API 上传新版本 | [views.py](file:///d:/fz/0601/solo-dogfeeding/code/114-paperless-ngx/src/documents/views.py#L1883-L1950) | `update_version()` 方法 |
| 旋转 PDF | [bulk_edit.py](file:///d:/fz/0601/solo-dogfeeding/code/114-paperless-ngx/src/documents/bulk_edit.py#L433-L500) | `rotate()` |
| 删除页面 | [bulk_edit.py](file:///d:/fz/0601/solo-dogfeeding/code/114-paperless-ngx/src/documents/bulk_edit.py#L692-L742) | `delete_pages()` |
| 编辑 PDF（更新模式） | [bulk_edit.py](file:///d:/fz/0601/solo-dogfeeding/code/114-paperless-ngx/src/documents/bulk_edit.py#L745-L872) | `edit_pdf(update_document=True)` |
| 移除密码（更新模式） | [bulk_edit.py](file:///d:/fz/0601/solo-dogfeeding/code/114-paperless-ngx/src/documents/bulk_edit.py#L875-L964) | `remove_password(update_document=True)` |

### 2.2 版本创建核心流程

**第一步：API/操作层 → 构造 ConsumableDocument**

以 `update_version` 为例（[views.py#L1890-L1941](file:///d:/fz/0601/solo-dogfeeding/code/114-paperless-ngx/src/documents/views.py#L1890-L1941)）：

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

文件位置：[tasks.py](file:///d:/fz/0601/solo-dogfeeding/code/114-paperless-ngx/src/documents/tasks.py#L123-L220)

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

文件位置：[consumer.py](file:///d:/fz/0601/solo-dogfeeding/code/114-paperless-ngx/src/documents/consumer.py#L256-L295)

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

文件位置：[consumer.py](file:///d:/fz/0601/solo-dogfeeding/code/114-paperless-ngx/src/documents/consumer.py#L408-L784)

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

### 2.3 版本读取与解析

文件位置：[versioning.py](file:///d:/fz/0601/solo-dogfeeding/code/114-paperless-ngx/src/documents/versioning.py)

| 函数 | 作用 |
|------|------|
| `get_root_document(doc)` | 获取根文档（doc.root_document_id 为 None 则返回自身） |
| `get_latest_version_for_root(root_doc)` | 按 id 倒序取最新版本 |
| `resolve_requested_version_for_root(root_doc, request)` | 解析 URL `?version=ID` 参数 |
| `resolve_effective_document(request_doc, request)` | 综合逻辑：有 version 参数取指定版本，否则对根文档取最新版本 |

根文档的 `get_effective_content()`（[models.py#L363-L400](file:///d:/fz/0601/solo-dogfeeding/code/114-paperless-ngx/src/documents/models.py#L363-L400)）会自动取最新版本的 content 用于搜索和建议。

### 2.4 版本删除

文件位置：[views.py](file:///d:/fz/0601/solo-dogfeeding/code/114-paperless-ngx/src/documents/views.py#L1994-L2055)

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

## 三、OCR 重跑（文本重算）流程

### 3.1 入口

| 入口 | 位置 |
|------|------|
| Web API `/api/documents/reprocess/` | [views.py](file:///d:/fz/0601/solo-dogfeeding/code/114-paperless-ngx/src/documents/views.py#L3000-L3024) `ReprocessDocumentsView` |
| bulk_edit.reprocess() | [bulk_edit.py](file:///d:/fz/0601/solo-dogfeeding/code/114-paperless-ngx/src/documents/bulk_edit.py#L395-L402) |
| 管理命令 `document_archiver` | [management/commands/document_archiver.py](file:///d:/fz/0601/solo-dogfeeding/code/114-paperless-ngx/src/documents/management/commands/document_archiver.py) |

### 3.2 bulk_edit.reprocess

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

### 3.3 update_document_content_maybe_archive_file 核心实现

文件位置：[tasks.py](file:///d:/fz/0601/solo-dogfeeding/code/114-paperless-ngx/src/documents/tasks.py#L278-L395)

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
                document.archive_filename = generate_unique_filename(document, archive_filename=True)
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

### 3.4 与版本创建的区别

| 维度 | 版本创建 (consume_file + root_document_id) | OCR 重跑 (reprocess) |
|------|------------------------------------------|----------------------|
| 新 Document 记录 | 是，创建新行 | 否，原地更新 |
| version_index | 递增分配 | 不变 |
| content 更新 | 解析新文件得到 | 重新解析原文件 |
| 归档文件 | 使用新文件生成 | 重新生成（覆盖原路径） |
| 触发信号 | `document_consumption_finished` + `document_updated`（根） | 仅通过 `post_save` 触发后续 |
| 审计日志 | "Version Added" 记录在根文档 | "Update document content" |

---

## 四、文件状态更新流程

文档保存/更新后触发一系列级联更新，主要通过 Django `post_save` 信号和自定义 `document_updated` / `document_consumption_finished` 信号驱动。

### 4.1 信号注册

文件位置：[apps.py](file:///d:/fz/0601/solo-dogfeeding/code/114-paperless-ngx/src/documents/apps.py#L10-L36)

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

### 4.2 update_filename_and_move_files — 文件名生成与文件移动

**触发时机**：`Document.post_save`、`Document.tags.m2m_changed`、`CustomFieldInstance.post_save`

文件位置：[signals/handlers.py](file:///d:/fz/0601/solo-dogfeeding/code/114-paperless-ngx/src/documents/signals/handlers.py#L430-L628)

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

**版本文件命名**：当 `document.root_document_id is not None` 时，`generate_filename` 会追加 `-v{version_index}` 后缀（参考 [tests/test_file_handling.py#L1329-L1414](file:///d:/fz/0601/solo-dogfeeding/code/114-paperless-ngx/src/documents/tests/test_file_handling.py#L1329-L1414)）。

### 4.3 bulk_update_documents — 批量触发更新链

文件位置：[tasks.py](file:///d:/fz/0601/solo-dogfeeding/code/114-paperless-ngx/src/documents/tasks.py#L252-L275)

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

### 4.4 document_updated 信号处理

文件位置：[signals/handlers.py](file:///d:/fz/0601/solo-dogfeeding/code/114-paperless-ngx/src/documents/signals/handlers.py#L819-L851)

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

### 4.5 document_consumption_finished 信号处理

该信号在新文档/新版本消费完成后触发，依次执行：
1. `add_inbox_tags` — 添加收件箱标签
2. `set_correspondent` — 自动匹配联系人
3. `set_document_type` — 自动匹配文档类型
4. `set_tags` — 自动匹配标签
5. `set_storage_path` — 自动匹配存储路径
6. `add_to_index` — 添加到 Tantivy 全文索引（使用 `get_effective_content()`）
7. `run_workflows_added` — 执行 DOCUMENT_ADDED 工作流
8. `add_or_update_document_in_llm_index` — 添加到 LLM 向量索引

### 4.6 整体状态更新时序图

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

## 五、Celery 任务追踪

文件位置：[signals/handlers.py](file:///d:/fz/0601/solo-dogfeeding/code/114-paperless-ngx/src/documents/signals/handlers.py#L1005-L1141)

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

## 六、关键文件索引

| 文件 | 作用 |
|------|------|
| [models.py](file:///d:/fz/0601/solo-dogfeeding/code/114-paperless-ngx/src/documents/models.py) | Document、PaperlessTask 等数据模型 |
| [versioning.py](file:///d:/fz/0601/solo-dogfeeding/code/114-paperless-ngx/src/documents/versioning.py) | 版本解析、根文档/最新版本获取 |
| [consumer.py](file:///d:/fz/0601/solo-dogfeeding/code/114-paperless-ngx/src/documents/consumer.py) | 文档消费核心（ConsumerPlugin._create_version_from_root） |
| [tasks.py](file:///d:/fz/0601/solo-dogfeeding/code/114-paperless-ngx/src/documents/tasks.py) | consume_file、update_document_content_maybe_archive_file 等 Celery 任务 |
| [bulk_edit.py](file:///d:/fz/0601/solo-dogfeeding/code/114-paperless-ngx/src/documents/bulk_edit.py) | rotate/merge/split/delete_pages/edit_pdf/remove_password/reprocess 等批量操作入口 |
| [views.py](file:///d:/fz/0601/solo-dogfeeding/code/114-paperless-ngx/src/documents/views.py) | API 层：update_version、delete_version、ReprocessDocumentsView 等 |
| [signals/handlers.py](file:///d:/fz/0601/solo-dogfeeding/code/114-paperless-ngx/src/documents/signals/handlers.py) | update_filename_and_move_files、工作流执行、WebSocket 通知、任务追踪 |
| [signals/__init__.py](file:///d:/fz/0601/solo-dogfeeding/code/114-paperless-ngx/src/documents/signals/__init__.py) | 自定义信号定义 |
| [apps.py](file:///d:/fz/0601/solo-dogfeeding/code/114-paperless-ngx/src/documents/apps.py) | 信号处理器注册 |
| [data_models.py](file:///d:/fz/0601/solo-dogfeeding/code/114-paperless-ngx/src/documents/data_models.py) | ConsumableDocument、DocumentMetadataOverrides |
