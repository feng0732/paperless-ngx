# Paperless-ngx Trash & Archive 生命周期代码分析

## 1. 核心基础：软删除机制

Paperless-ngx 使用 `django-softdelete` 库实现软删除（回收站）机制。

### 1.1 模型层设计

Document 模型继承自 `SoftDeleteModel`，这是整个回收站机制的基石：

- **位置**：[models.py](file:///d:/fz/0601/solo-dogfeeding/code/62-paperless-ngx/src/documents/models.py#L157)
- **关键导入**：`from django_softdelete.models import SoftDeleteModel`

继承 `SoftDeleteModel` 后获得以下能力：

| Manager | 作用 |
|--------|------|
| `Document.objects` | 默认查询集，自动过滤已删除（软删除）的文档 |
| `Document.deleted_objects` | 仅查询已软删除的文档（回收站内容） |
| `Document.global_objects` | 查询所有文档，包括已删除和未删除的 |
| `doc.is_deleted` | 布尔属性，标记文档是否处于软删除状态 |
| `doc.deleted_at` | 软删除时间戳 |
| `doc.delete()` | 执行软删除（设置 `deleted_at`，不真正从 DB 删除） |
| `doc.restore(strict=False)` | 从回收站恢复文档 |
| `doc.hard_delete()` | 真正从数据库删除记录 |

同样使用软删除的模型还有：
- [Note](file:///d:/fz/0601/solo-dogfeeding/code/62-paperless-ngx/src/documents/models.py#L828) — 文档备注
- [ShareLink](file:///d:/fz/0601/solo-dogfeeding/code/62-paperless-ngx/src/documents/models.py#L868) — 分享链接
- [CustomFieldInstance](file:///d:/fz/0601/solo-dogfeeding/code/62-paperless-ngx/src/documents/models.py#L1093) — 自定义字段实例

### 1.2 Document.delete() 方法重写

Document 重写了 delete 方法，确保删除根文档时其版本文档也一并进入回收站：

```python
# models.py L503-L514
def delete(self, *args, **kwargs):
    # If deleting a root document, move all its versions to trash as well.
    if self.root_document_id is None:
        Document.objects.filter(root_document=self).delete()
    return super().delete(*args, **kwargs)
```

---

## 2. 完整生命周期流程图

```
用户删除文档
    │
    ▼
┌─────────────────────────────────┐
│  软删除（进入回收站）            │
│  - 调用 Document.delete()        │
│  - 设置 deleted_at 时间戳        │
│  - 从搜索索引移除                │
│  - 发送 WebSocket 通知           │
│  - 文件保持不变 ❗                │
└─────────────────────────────────┘
    │
    ├───────────────────────┐
    │                       │
    ▼                       ▼
┌──────────────┐     ┌──────────────────┐
│  恢复文档     │     │  清空回收站        │
│  restore()    │     │  empty_trash()    │
│              │     │                  │
│  - 清除 deleted_at │  - 临时连接信号    │
│  - 重新索引        │  - 硬删除 DB 记录   │
│  - 文件保持不变    │  - 删除磁盘文件     │
└──────────────┘     │  - 清理审计日志    │
                     └──────────────────┘
```

---

## 3. 软删除（移入回收站）流程

### 3.1 API 入口

删除文档有三种触发方式：

#### 方式一：单文档删除
- **API**：`DELETE /api/documents/<id>/`
- **视图**：[DocumentViewSet.destroy()](file:///d:/fz/0601/solo-dogfeeding/code/62-paperless-ngx/src/documents/views.py#L1204-L1218)

```python
def destroy(self, request, *args, **kwargs):
    from documents.search import get_backend
    get_backend().remove(self.get_object().pk)  # 从搜索索引移除
    return super().destroy(request, *args, **kwargs)  # 调用 DRF 的 destroy，最终调用 model.delete()
```

#### 方式二：批量删除
- **API**：`POST /api/documents/delete/`
- **视图**：[DeleteDocumentsView](file:///d:/fz/0601/solo-dogfeeding/code/62-paperless-ngx/src/documents/views.py#L2987-L2997)
- **执行器**：[bulk_edit.delete()](file:///d:/fz/0601/solo-dogfeeding/code/62-paperless-ngx/src/documents/bulk_edit.py#L359-L392)

```python
@shared_task
def delete(doc_ids: list[int]) -> Literal["OK"]:
    # 收集根文档 ID 及其所有版本文档 ID
    root_ids = Document.objects.filter(
        id__in=doc_ids, root_document__isnull=True
    ).values_list("id", flat=True).distinct()
    version_ids = Document.objects.filter(
        root_document_id__in=root_ids
    ).exclude(id__in=doc_ids).values_list("id", flat=True).distinct()
    delete_ids = list({*doc_ids, *version_ids})

    Document.objects.filter(id__in=delete_ids).delete()  # 软删除

    # 从搜索索引批量移除
    from documents.search import get_backend
    with get_backend().batch_update() as batch:
        for id in delete_ids:
            batch.remove(id)

    # 发送 WebSocket 通知
    status_mgr = DocumentsStatusManager()
    status_mgr.send_documents_deleted(delete_ids)
```

#### 方式三：工作流自动删除
- **触发**：工作流中配置了「Move to trash」动作
- **动作类型**：`WorkflowAction.WorkflowActionType.MOVE_TO_TRASH` (值=6)
- **执行器**：[execute_move_to_trash_action()](file:///d:/fz/0601/solo-dogfeeding/code/62-paperless-ngx/src/documents/workflows/actions.py#L348-L374)

```python
def execute_move_to_trash_action(action, document, logging_group):
    if isinstance(document, Document):
        document.delete()  # 已存在的文档，执行软删除
    else:
        # 消费过程中触发：直接删除文件并终止消费
        if document.original_file.exists():
            document.original_file.unlink()
        raise StopConsumeTaskError("Document deleted by workflow action")
```

工作流中触发 move_to_trash 后，该文档所在工作流的后续动作会被跳过（[handlers.py L907-L913](file:///d:/fz/0601/solo-dogfeeding/code/62-paperless-ngx/src/documents/signals/handlers.py#L907-L913)）：

```python
if document.is_deleted:
    logger.info("Document was moved to trash, skipping remaining workflows")
    break
```

### 3.2 软删除的关键点

**文件不会被删除！** 软删除仅操作数据库：
- 设置 `deleted_at` 时间戳
- 文档从 `Document.objects` 查询集中消失
- 出现在 `Document.deleted_objects` 中
- 磁盘上的原始文件、归档文件、缩略图都保留

---

## 4. 回收站管理（TrashView）

### 4.1 API 端点

- **URL**：`/api/trash/`
- **路由**：[urls.py](file:///d:/fz/0601/solo-dogfeeding/code/62-paperless/src/paperless/urls.py#L257-L259)
- **视图**：[TrashView](file:///d:/fz/0601/solo-dogfeeding/code/62-paperlessless62-paperlessless5083-L5123)

### 4.2 GET - 列出回收站内容

```python
queryset = Document.deleted_objects.all()  # 只返回已软删除的文档
```

### 4.3 POST - 恢复或清空

请求体使用 [TrashSerializer](file:///d:/fz/0601/solo-dogfeeding/code/62-paperlessless62-paperlessless3391-L3401)：
- `documents`：可选，指定文档 ID 列表；不提供则操作全部
- `action`：`"restore"` 或 `"empty"`

#### 恢复文档
```python
for doc in Document.deleted_objects.filter(id__in=doc_ids).all():
    doc.restore(strict=False)
```

`restore(strict=False)` 是 django-softdelete 提供的方法，会清除 `deleted_at` 字段。恢复后：
- 文档重新出现在 `Document.objects` 查询集中
- 文件保持原位不变
- 需要重新索引搜索（恢复本身不自动触发）

#### 清空回收站
```python
empty_trash(doc_ids=doc_ids)  # 永久删除
```

---

## 5. 永久删除（清空回收站）流程

### 5.1 触发方式

1. **手动清空**：通过 `/api/trash/` POST `action=empty`
2. **定时任务**：Celery Beat 定期调用 `documents.tasks.empty_trash`（默认每天执行一次，见 [custom.py](file:///d:/fz/0601/solo-dogfeedingless62-paperlessless62-paperlessless/settings/custom.py#L125-L132)）
3. **自动过期**：定时任务执行时，会删除在回收站中超过 `EMPTY_TRASH_DELAY` 天（默认 30 天）的文档

### 5.2 empty_trash 核心实现

位置：[tasks.py L398-L432](file:///d:/fz/0601/solo-dogfeedingless62-paperlessless62-paperlessless/tasks.py#L398-L432)

```python
@shared_task
def empty_trash(doc_ids=None) -> None:
    if doc_ids is None:
        # 自动模式：只删除超过保留期限的
        documents = Document.deleted_objects.filter(
            deleted_at__lt=timezone.localtime(timezone.now())
            - datetime.timedelta(days=settings.EMPTY_TRASH_DELAY),
        )
    else:
        # 手动模式：删除指定 ID
        documents = Document.deleted_objects.filter(id__in=doc_ids)

    try:
        deleted_document_ids = list(documents.values_list("id", flat=True))

        # 关键：临时连接文件清理信号
        models.signals.post_delete.connect(cleanup_document_deletion, sender=Document)

        documents.delete()  # 这是真正的硬删除！

        # 清理审计日志
        if settings.AUDIT_LOG_ENABLED:
            LogEntry.objects.filter(
                content_type=ContentType.objects.get_for_model(Document),
                object_id__in=deleted_document_ids,
            ).delete()
    finally:
        # 断开信号连接，避免影响正常的软删除
        models.signals.post_delete.disconnect(
            cleanup_document_deletion, sender=Document,
        )
```

**设计精妙之处**：
- `cleanup_document_deletion` **不是**默认注册的信号处理器
- 只有在 `empty_trash` 执行时才临时连接，这样正常的软删除（`Document.delete()`）不会触发文件删除
- 使用 `try/finally` 确保信号总是被断开

---

## 6. 文件系统清理逻辑

### 6.1 cleanup_document_deletion

位置：[signals/handlers.py L342-L402](file:///d:/fz/0601/solo-dogfeedingless62-paperlessless62-paperlessless/signals/handlers.py#L342-L402)

当 `empty_trash` 连接此信号后，每个 Document 被硬删除时会触发：

```python
def cleanup_document_deletion(sender, instance, **kwargs) -> None:
    with FileLock(settings.MEDIA_LOCK):  # 文件操作加锁
        if settings.EMPTY_TRASH_DIR:
            # 可选：将原始文件移动到专门的回收站目录（而非直接删除）
            # 处理文件名冲突：test.pdf → test_01.pdf, test_02.pdf ...
            shutil.move(instance.source_path, new_file_path)
        else:
            # 默认：直接删除原始文件
            files = (instance.archive_path, instance.thumbnail_path, instance.source_path)
            for filename in files:
                if filename and filename.is_file():
                    filename.unlink()

        # 清理空目录
        delete_empty_directories(
            Path(instance.source_path).parent, root=settings.ORIGINALS_DIR,
        )
        if instance.has_archive_version:
            delete_empty_directories(
                Path(instance.archive_path).parent, root=settings.ARCHIVE_DIR,
            )
```

### 6.2 删除的文件清单

| 文件 | 对应模型属性 | 说明 |
|------|------------|------|
| 原始文件 | `doc.source_path` | ORIGINALS_DIR 下，可能被移动到 EMPTY_TRASH_DIR |
| 归档文件 | `doc.archive_path` | ARCHIVE_DIR 下的 PDF/A 版本 |
| 缩略图 | `doc.thumbnail_path` | THUMBNAIL_DIR 下的 webp 文件 |

### 6.3 删除 LLM 索引

另外有一个始终注册的信号处理器：[delete_document_from_llm_index()](file:///d:/fz/0601/solo-dogfeedingless62-paperlessless62-paperlessless/signals/handlers.py#L1342-L1355)

```python
@receiver(models.signals.post_delete, sender=Document)
def delete_document_from_llm_index(sender, instance, **kwargs):
    # 从 LLM 向量索引中异步移除文档
    remove_document_from_llm_index.apply_async(kwargs={"document": instance})
```

这个信号**始终**连接，意味着软删除和硬删除都会触发从 LLM 索引移除。

---

## 7. 关联资源的影响分析

### 7.1 数据库外键关系（Document 作为主表）

| 关联对象 | 关系类型 | on_delete 行为 | 软删除影响 | 硬删除影响 |
|--------|--------|--------------|----------|----------|
| [Correspondent](file:///d:/fz/0601/solo-dogfeedingless62-paperlessless62-paperlessless/models.py#L160-L167) | ForeignKey | `SET_NULL` | 不变 | 外键置 NULL，往来人保留 |
| [StoragePath](file:///d:/fz/0601/solo-dogfeedingless62-paperlessless62-paperlessless/models.py#L169-L176) | ForeignKey | `SET_NULL` | 不变 | 外键置 NULL，存储路径保留 |
| [DocumentType](file:///d:/fz/0601/solo-dogfeedingless62-paperlessless62-paperlessless/models.py#L180-L187) | ForeignKey | `SET_NULL` | 不变 | 外键置 NULL，文档类型保留 |
| [Tag](file:///d:/fz/0601/solo-dogfeedingless62-paperlessless62-paperlessless/models.py#L209-L214) | ManyToMany | 通过中间表 | 中间表不变 | 中间表记录级联删除，标签本身保留 |
| [Note](file:///d:/fz/0601/solo-dogfeedingless62-paperlessless62-paperlessless/models.py#L841-L848) | ForeignKey | `CASCADE` | Note 也被软删除 | Note 被硬删除 |
| [ShareLink](file:///d:/fz/0601/solo-dogfeedingless62-paperlessless62-paperlessless/models.py#L896-L902) | ForeignKey | `CASCADE` | ShareLink 也被软删除 | ShareLink 被硬删除 |
| [CustomFieldInstance](file:///d:/fz/0601/solo-dogfeedingless62-paperlessless/models.py#L1119-L1135) | ForeignKey | `CASCADE` | CustomFieldInstance 也被软删除 | CustomFieldInstance 被硬删除 |
| [Version Document](file:///d:/fz/0601/solo-dogfeedingless62-paperlessless/models.py#L312-L319) | ForeignKey (root_document) | `CASCADE` | 版本也被软删除（在 Document.delete() 中显式处理） | 版本被硬删除 |
| [ShareLinkBundle](file:///d:/fz/0601/solo-dogfeedingless62-paperlessless/models.py#L1004-L1008) | ManyToMany | 通过中间表 | 中间表不变 | 中间表记录级联删除，Bundle 保留 |

### 7.2 其他关联影响

- **搜索索引**：软删除时立即移除，恢复后需重新索引
- **LLM 索引**：软删除和硬删除都会触发异步移除
- **审计日志**：仅在 `empty_trash` 时才会清理对应的 LogEntry 记录
- **ShareLinkBundle 文件**：Bundle 自身的 `delete()` 方法会清理生成的 zip 文件，但这与 Document 删除解耦

---

## 8. Archive（归档文件）的特殊说明

注意：Paperless-ngx 中的「archive」有两种含义，不要混淆：

### 8.1 Archive 文件（PDF/A 版本）
- 指 OCR 后生成的 PDF/A 归档副本
- 存储在 `settings.ARCHIVE_DIR`
- 对应字段：`archive_filename`、`archive_checksum`、`archive_path`
- 在清空回收站时与原始文件一起被删除/移动

### 8.2 Archive Serial Number（ASN 归档编号）
- 物理档案盒的编号，`archive_serial_number` 字段
- 在合并/分割文档时，如果删除原文档，ASN 会被「释放」然后转移给新文档
- 相关函数：[release_archive_serial_numbers()](file:///d:/fz/0601/solo-dogfeedingless62-paperlessless/bulk_edit.py#L65-L78)、[restore_archive_serial_numbers()](file:///d:/fz/0601/solo-dogfeedingless62-paperlessless/bulk_edit.py#L81-L88)

---

## 9. 关键配置项

| 配置项 | 作用 | 默认值 |
|--------|------|--------|
| `EMPTY_TRASH_DELAY` | 回收站自动清空延迟（天） | 30 |
| `EMPTY_TRASH_DIR` | 若设置，删除时将文件移动到此目录而非直接删除 | None |
| `ORIGINALS_DIR` | 原始文件存储目录 | - |
| `ARCHIVE_DIR` | 归档文件存储目录 | - |
| `THUMBNAIL_DIR` | 缩略图存储目录 | - |
| `MEDIA_LOCK` | 文件操作锁文件路径 | - |

---

## 10. 常见疑问解答

**Q：软删除后文件还在磁盘上吗？**
A：在。软删除只改数据库 `deleted_at` 字段，不碰磁盘文件。只有 `empty_trash` 执行硬删除时才会清理文件。

**Q：为什么 cleanup_document_deletion 不直接注册到 post_delete 信号？**
A：因为 django-softdelete 的 `delete()` 也会触发 post_delete（用于设置 deleted_at 的链式操作）。如果默认注册，软删除时文件也会被删掉，破坏回收站功能。所以采用「临时连接」的技巧。

**Q：恢复文档后，搜索能搜到吗？**
A：restore() 本身不触发重新索引。实际使用中需要手动重新索引或等待后续更新触发索引刷新。

**Q：版本文档删除时会怎样？**
A：删除根文档时，Document.delete() 显式把所有版本也一并软删除。单独删除版本文档时走正常流程。

**Q：工作流中 move_to_trash 之后的动作还执行吗？**
A：不执行。run_workflows 循环中会检查 `document.is_deleted`，如果为真就 break 跳出。
