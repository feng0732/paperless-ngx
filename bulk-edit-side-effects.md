# Bulk Edit 与任务副作用协作机制分析

本文档梳理 Paperless-ngx 中批量编辑（Bulk Edit）的协作逻辑，包括：元数据变更 → 后台任务调度 → 副作用执行 → 部分失败边界处理的完整链路。

---

## 一、整体架构分层

```
API 入口层           核心逻辑层          异步任务层        信号副作用层
┌──────────────┐   ┌──────────────┐   ┌──────────────┐   ┌──────────────┐
│  Views.py    │──▶│ bulk_edit.py│──▶│  tasks.py  │──▶│  handlers  │
│ (BulkEditView, │   │ (各 edit fn) │   │ (Celery)   │   │ (signals)  │
│  Rotate/...  )│   │            │   │            │   │            │
└──────────────┘   └──────────────┘   └──────────────┘   └──────────────┘
                       │                 │                 │
                       ▼                 ▼                 ▼
                 DB 同步写         搜索索引/工作流/WebSocket推送/文件重命名
```

---

## 二、API 入口层

### 2.1 主入口：[BulkEditView](file:///d:/fz/0601/solo-dogfeeding/code/61-paperless-ngx/src/documents/views.py#L2797-L2916)

`BulkEditView` 是统一的批量编辑端点。其 `post` 方法流程：

1. **解析请求**：通过 [BulkEditSerializer](file:///d:/fz/0601/solo-dogfeeding/code/61-paperless-ngx/src/documents/serialisers.py#L1696-L1815) 解析 `method` 字段将字符串映射到 `bulk_edit` 模块对应函数（如 `"set_correspondent"` → `bulk_edit.set_correspondent`。

2. **权限校验**：`_has_document_permissions()` 根据操作类型检查 `change_document`、`delete_document`、`add_document` 等权限。

3. **审计日志**：若开启 AUDIT_LOG_ENABLED，执行前快照 old_value，执行后记录 LogEntry。

4. **执行方法**：`result = method(documents, **parameters)` 同步执行核心函数。

5. **返回 200 OK**：无论后台任务是否完成，仅表示「已成功调度」即返回。

> 关键代码 [views.py#L2864-L2911](file:///d:/fz/0601/solo-dogfeeding/code/61-paperless-ngx/src/documents/views.py#L2864-L2911)

```python
try:
    modified_field = self.MODIFIED_FIELD_BY_METHOD.get(method.__name__, None)
    if settings.AUDIT_LOG_ENABLED and modified_field:
        old_documents = {...}
    result = method(documents, **parameters)   # ← 同步写 DB
    if settings.AUDIT_LOG_ENABLED and modified_field:
        # ... 写 LogEntry
    return Response({"result": result})      # ← 立即返回，不等异步任务
except Exception as e:
    return HttpResponseBadRequest(...)
```

### 2.2 其他专用端点

`RotateDocumentsView`、`MergeDocumentsView`、`DeleteDocumentsView`、`ReprocessDocumentsView` 等通过 `_execute_document_action()` 调用，逻辑与 BulkEditView 相似但无审计日志逻辑。见 [views.py#L2733-L2776](file:///d:/fz/0601/solo-dogfeeding/code/61-paperless-ngx/src/documents/views.py#L2733-L2776)。

---

## 三、核心逻辑层（bulk_edit.py）

批量编辑按副作用触发方式分为两大类：**元数据直接修改 + bulk_update_documents 异步任务** vs **新文档生成类操作（rotate/merge/split/edit_pdf/remove_password/delete_pages）**。

### 3.1 第一类：元数据直接修改

这类操作**同步写入数据库**后，**异步调度 `bulk_update_documents`** 触发后续副作用。

典型模式（以 `set_correspondent` 为例 [bulk_edit.py#L111-L131](file:///d:/fz/0601/solo-dogfeeding/code/61-paperless-ngx/src/documents/bulk_edit.py#L111-L131)：

```python
def set_correspondent(doc_ids: list[int], correspondent: Correspondent):
    # 1. 过滤出实际需要修改的文档
    qs = Document.objects.filter(
        Q(id__in=doc_ids) & ~Q(correspondent=correspondent)
    )
    affected_docs = list(qs.values_list("pk", flat=True))

    # 2. 同步批量 UPDATE 数据库
    qs.update(correspondent=correspondent)

    # 3. 异步触发副作用任务
    bulk_update_documents.apply_async(
        kwargs={"document_ids": affected_docs},
        headers={"trigger_source": PaperlessTask.TriggerSource.SYSTEM},
    )
    return "OK"
```

同类方法及关键差异：

| 方法 | 同步 DB 操作 | affected_docs 计算方式 |
|---|---|---|
| [set_correspondent](file:///d:/fz/0601/solo-dogfeeding/code/61-paperless-ngx/src/documents/bulk_edit.py#L111-L131) | `qs.update(correspondent=...) | `~Q(correspondent=...)` 过滤已相同的文档 |
| [set_document_type](file:///d:/fz/0601/solo-dogfeeding/code/61-paperless-ngx/src/documents/bulk_edit.py#L156-L173) | `qs.update(document_type=...) | 同上 |
| [set_storage_path](file:///d:/fz/0601/solo-dogfeeding/code/61-paperless-ngx/src/documents/bulk_edit.py#L134-L153) | `qs.update(storage_path=...) | 同上 |
| [add_tag](file:///d:/fz/0601/solo-dogfeeding/code/61-paperless-ngx/src/documents/bulk_edit.py#L176-L202) | `bulk_create` 多对多关系 | 过滤已存在的不重复创建，**含祖先 tag |
| [remove_tag](file:///d:/fz/0601/solo-dogfeeding/code/61-paperless-ngx/src/documents/bulk_edit.py#L205-L223) | `qs.delete()` 多对多关系 | 删除 tag 及其所有后代 tag |
| [modify_tags](file:///d:/fz/0601/solo-dogfeeding/code/61-paperless-ngx/src/documents/bulk_edit.py#L226-L284) | transaction.atomic 中先删后增 | 所有传入 doc_ids（测试注释说明精确过滤复杂） |
| [modify_custom_fields](file:///d:/fz/0601/solo-dogfeeding/code/61-paperless-ngx/src/documents/bulk_edit.py#L287-L356) | `update_or_create` + 处理 DOCUMENTLINK 对称反射 | 所有传入 doc_ids |
| [set_permissions](file:///d:/fz/0601/solo-dogfeeding/code/61-paperless-ngx/src/documents/bulk_edit.py#L405-L430) | 遍历设置 owner + django-guardian 权限 | 所有传入 doc_ids |

**注意**：`modify_tags` 中使用了 `transaction.atomic()` 保证 add/remove 原子性 [bulk_edit.py#L249-L276](file:///d:/fz/0601/solo-dogfeeding/code/61-paperless-ngx/src/documents/bulk_edit.py#L249-L276)。

### 3.2 第二类：生成新文档/版本的操作

`rotate`、`merge`、`split`、`delete_pages`、`edit_pdf`、`remove_password` 等操作：

1. **同步处理 PDF 文件**（pikepdf 读写临时文件）
2. **调度 `consume_file` 异步任务** 将临时文件通过消费管线生成新文档或新版本
3. 若 `delete_originals=True`，用 **Celery chord** 编排：consume 全部成功 → 执行删除；失败 → 恢复 ASN

典型流程（以 `merge` + `delete_originals=True` 为例 [bulk_edit.py#L594-L608](file:///d:/fz/0601/solo-dogfeeding/code/61-paperless-ngx/src/documents/bulk_edit.py#L594-L608)：

```python
if delete_originals:
    backup = release_archive_serial_numbers(affected_docs)  # 先释放 ASN，存备份
    try:
        consume_task.apply_async(
            link=[delete.si(affected_docs)],                       # 成功后删原文档
            link_error=[restore_archive_serial_numbers_task.s(backup)],  # 失败则恢复 ASN
        )
    except Exception:
        restore_archive_serial_numbers(backup)   # apply_async 抛错立即同步恢复
        raise
```

`split` 使用 chord（多任务并行），见 [bulk_edit.py#L673-L682](file:///d:/fz/0601/solo-dogfeeding/code/61-paperless-ngx/src/documents/bulk_edit.py#L673-L682)：

```python
chord(
    header=consume_tasks,          # 多个 consume_file 并行
    body=delete.si([doc.id]),    # 全部成功后执行删除
).apply_async(
    link_error=[restore_archive_serial_numbers_task.s(backup)]
)
```

### 3.3 第三类：删除和重处理

- **`delete`** [bulk_edit.py#L359-L392](file:///d:/fz/0601/solo-dogfeeding/code/61-paperless-ngx/src/documents/bulk_edit.py#L359-L392)：自身就是 `@shared_task`，异步执行：级联删所有版本 → 搜索索引移除 → WebSocket 推送删除事件。含特殊异常捕获（UUID 列兼容问题）。
- **`reprocess`** [bulk_edit.py#L395-L402](file:///d:/fz/0601/solo-dogfeeding/code/61-paperless-ngx/src/documents/bulk_edit.py#L395-L402)：对每个文档单独调度 `update_document_content_maybe_archive_file`（重新 OCR）。

---

## 四、异步任务层：bulk_update_documents

[bulk_update_documents](file:///d:/fz/0601/solo-dogfeeding/code/61-paperless-ngx/src/documents/tasks.py#L252-L275) 是元数据变更后的「统一副作用触发器」：

```python
@shared_task
def bulk_update_documents(document_ids) -> None:
    documents = Document.objects.filter(id__in=document_ids)

    for doc in documents:
        clear_document_caches(doc.pk)                    # 1. 清缓存
        document_updated.send(                           # 2. 发自定义信号
            sender=None,
            document=doc,
            logging_group=uuid.uuid4(),
        )
        post_save.send(Document, instance=doc, created=False)  # 3. 发 Django post_save

    with get_backend().batch_update() as batch:          # 4. 更新搜索索引
        for doc in documents:
            batch.add_or_update(doc)

    if ai_config.llm_index_enabled:                       # 5. （可选）更新 LLM 索引
        update_llm_index(rebuild=False)
```

这个任务被登记在 `TRACKED_TASKS` 中 [handlers.py#L1013](file:///d:/fz/0601/solo-dogfeeding/code/61-paperless-ngx/src/documents/signals/handlers.py#L1013)，其状态由 Celery 信号追踪（PENDING → STARTED → SUCCESS/FAILURE）。

---

## 五、信号副作用层

信号连接在 [apps.py](file:///d:/fz/0601/solo-dogfeeding/code/61-paperless-ngx/src/documents/apps.py#L10-L37) 的 `ready()` 中注册：

```python
document_updated.connect(run_workflows_updated)
document_updated.connect(send_websocket_document_updated)
```

另有 Django ORM 信号：
- `post_save(sender=Document)` → `update_filename_and_move_files` [handlers.py#L431-L667](file:///d:/fz/0601/solo-dogfeeding/code/61-paperless-ngx/src/documents/signals/handlers.py#L431-L667)
- `post_save(sender=CustomFieldInstance)` → `update_filename_and_move_files`
- `m2m_changed(sender=Document.tags.through)` → `update_filename_and_move_files`

### 5.1 run_workflows_updated → run_workflows

[handlers.py#L819-L829](file:///d:/fz/0601/solo-dogfeeding/code/61-paperless-ngx/src/documents/signals/handlers.py#L819-L829) → [run_workflows](file:///d:/fz/0601/solo-dogfeeding/code/61-paperless-ngx/src/documents/signals/handlers.py#L854-L998)

DOCUMENT_UPDATED 触发类型的工作流执行逻辑：

1. `document.refresh_from_db()` 防止并发覆盖（重要注释见 [handlers.py#L895-L905](file:///d:/fz/0601/solo-dogfeeding/code/61-paperless-ngx/src/documents/signals/handlers.py#L895-L905)）
2. 检查文档是否已被软删除（`is_deleted`），是则跳过
3. 匹配工作流，按顺序执行 action：ASSIGNMENT / REMOVAL / EMAIL / WEBHOOK / PASSWORD_REMOVAL / MOVE_TO_TRASH
4. 保存文档字段（title / correspondent / document_type / storage_path / owner）
5. **注意**：这里的 `document.save(update_fields=[...])` 会再次触发 `post_save` → `update_filename_and_move_files`，可能引起连锁副作用

### 5.2 send_websocket_document_updated

[handlers.py#L832-L851](file:///d:/fz/0601/solo-dogfeeding/code/61-paperless-ngx/src/documents/signals/handlers.py#L832-L851)：通过 WebSocket 向前端推送文档变更通知（modified 时间戳、owner、权限等）。

### 5.3 update_filename_and_move_files

[handlers.py#L434-L667](file:///d:/fz/0601/solo-dogfeeding/code/61-paperless-ngx/src/documents/signals/handlers.py#L434-L667)：根据 storage_path / 文件名模板重新计算文件名，必要时移动磁盘文件。使用 `FileLock(settings.MEDIA_LOCK)` 全局锁保护，失败时尝试回滚文件位置。

---

## 六、完整调用链路时序图（以 set_correspondent 为例）

```
前端 POST /api/documents/bulk_edit/
    │
    ▼
BulkEditView.post()
    │  1. 序列化校验 method="set_correspondent"
    │  2. 权限校验
    │  3. 审计快照
    │  4. 调用 bulk_edit.set_correspondent([1,2,3], correspondent_id=5)
    │
    ▼
bulk_edit.set_correspondent()
    │  a. DB: UPDATE documents SET correspondent_id=5 WHERE id IN (1,2,3) AND correspondent_id != 5
    │  b. 计算 affected_docs = [1,2,3]（假设都变了）
    │  c. bulk_update_documents.apply_async(document_ids=[1,2,3])
    │  d. return "OK"
    │
    ▼
BulkEditView 记录审计日志 → return HTTP 200 {"result": "OK"}
    │
    │  ═══════════ 前端已拿到响应，以下是后台异步 ═══════════
    │
    ▼
Celery Worker: bulk_update_documents([1,2,3])
    │
    ├─ for doc 1,2,3:
    │    ├─ clear_document_caches(doc.pk)
    │    ├─ document_updated.send(document=doc)
    │    │      ├─► run_workflows_updated()
    │    │      │      └─ run_workflows(DOCUMENT_UPDATED)
    │    │      │         ├─ 匹配并执行工作流 actions
    │    │      │         └─ document.save(...)  ← 再次触发 post_save
    │    │      └─► send_websocket_document_updated()
    │    │             └─ WebSocket 推送
    │    └─ post_save.send(Document, instance=doc)
    │           └─► update_filename_and_move_files()
    │                ├─ 生成新文件名
    │                ├─ 移动源文件/归档文件
    │                └─ Document.objects.update(filename=..., modified=...)
    │
    └─ 搜索索引批量更新 batch.add_or_update(doc)
```

---

## 七、部分失败边界处理

### 7.1 边界一：同步操作部分文档失败（元数据类操作）

**策略**：Django ORM 的 `QuerySet.update()` 是**原子单条 SQL**，要么全部成功要么全部失败。`bulk_create`/`delete` 同理。

如果在同步阶段异常：

- `modify_tags` 中的 `transaction.atomic()` 保证 add/remove 同生共死。
- 异常冒泡到 `BulkEditView.post()` 的 `try/except`，返回 400 Bad Request。
- **此时 DB 回滚，没有文档被修改，也不会调度异步任务**。

### 7.2 边界二：PDF 处理阶段部分文档失败（rotate/merge/split 等）

**策略**：**跳过失败文档，继续处理成功的。

典型代码（rotate 中 [bulk_edit.py#L460-L498](file:///d:/fz/0601/solo-dogfeeding/code/61-paperless-ngx/src/documents/bulk_edit.py#L460-L498)：

```python
for pair in docs_by_root_id.values():
    if pair.source_doc.mime_type != "application/pdf":
        logger.warning(f"Document {pair.root_doc.id} is not a PDF, skipping rotation.")
        continue          # ← 非 PDF 跳过
    try:
        # ... pikepdf 处理 ...
        consume_file.apply_async(...)
    except Exception as e:
        logger.exception(f"Error rotating document {pair.root_doc.id}: {e}")
        # ← 单个文档异常，跳过不影响其他文档
```

merge 中 [bulk_edit.py#L544-L547](file:///d:/fz/0601/solo-dogfeeding/code/61-paperless-ngx/src/documents/bulk_edit.py#L544-L547) 类似：

```python
except Exception as e:
    logger.exception(f"Error merging document {doc.id}, it will not be included in the merge: {e}")
```

**最终结果**：HTTP 200 OK，但部分文档未被处理，仅日志记录。前端无法从响应得知哪些失败。

### 7.3 边界三：异步任务 bulk_update_documents 部分失败

`bulk_update_documents` 是一个 Celery 任务。如果在 for 循环中单个文档的信号处理器抛异常：

- **document_updated.send() 中的异常会冒泡，导致整个任务失败。
- Celery 会根据重试策略重试。
- 已经处理完的文档的副作用（如搜索索引在循环之后批量提交，部分文档可能已触发了工作流/WebSocket 等，但后续文档未处理。
- 失败时 `task_failure_handler` 记录 PaperlessTask.Status.FAILURE。

### 7.4 边界四：consume_file 成功但后续 delete 失败（chord/link 编排）

以 merge+delete_originals=True 为例：

```
consume_file 成功 ──link──▶ delete(affected_docs)
           │
           └──link_error──▶ restore_archive_serial_numbers_task(backup)
```

- consume_file 失败 → **自动触发 link_error → 恢复 ASN。原文档保留，ASN 还原。
- consume_file 成功 → delete 执行。delete 是独立任务，若 delete 失败 ASN 已经释放不会自动恢复（delete 是软删除+文件清理，失败可查任务状态）。

### 7.5 边界五：ASN 释放与 apply_async 抛错同步恢复

在 merge/split/edit_pdf 中，release_archive_serial_numbers() 在调度任务前同步执行。如果 `apply_async()` 本身异常（如 broker 不可用）：

```python
try:
    chord(...).apply_async(link_error=[...])
except Exception:
    restore_archive_serial_numbers(backup)   # ← 同步恢复
    raise
```

见 [bulk_edit.py#L600-L606](file:///d:/fz/0601/solo-dogfeeding/code/61-paperless-ngx/src/documents/bulk_edit.py#L600-L606)、[bulk_edit.py#L673-L682](file:///d:/fz/0601/solo-dogfeeding/code/61-paperless-ngx/src/documents/bulk_edit.py#L673-L682)。

### 7.6 边界六：自定义字段 DOCUMENTLINK 对称反射失败

[modify_custom_fields](file:///d:/fz/0601/solo-dogfeeding/code/61-paperless-ngx/src/documents/bulk_edit.py#L325-L327) 中调用 `reflect_doclinks()`，如果目标文档的反射写入失败，异常会冒泡导致整个批量操作回滚（非事务性，已修改的文档的 CF 不会回滚）。

### 7.7 边界七：工作流执行期间文档被删除/软删

在 `run_workflows` 中每轮 workflow 前检查：

```python
document.refresh_from_db()           # 防并发覆盖
except Document.DoesNotExist:       # 硬删
    break
if document.is_deleted:             # 软删
    break
```

见 [handlers.py#L896-L913](file:///d:/fz/0601/solo-dogfeeding/code/61-paperless-ngx/src/documents/signals/handlers.py#L896-L913)。

---

## 八、关键设计总结

| 关注点 | 设计 |
|---|---|
| **同步/异步分界** | DB 元数据修改同步完成；搜索索引、工作流、WebSocket、文件重命名全部异步 |
| **任务编排** | Celery chord（多任务→删原文档）、link（单任务→删原文档）、link_error（失败补偿） |
| **失败原子性** | 元数据修改靠 Django ORM + transaction；PDF 处理逐文档 try/except 部分成功；ASN 有备份+恢复机制 |
| **可观测性** | TRACKED_TASKS 登记 Celery 任务状态（PENDING/STARTED/SUCCESS/FAILURE/REVOKED）；AUDIT_LOG 记录变更前后值 |
| **并发安全** | update_filename_and_move_files 使用全局 FileLock；run_workflows 每次 refresh_from_db；document.save 指定 update_fields 避免回滚并发写 |
