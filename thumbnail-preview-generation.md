# Paperless-ngx 缩略图（Thumbnail）与预览（Preview）生成机制详解

本文档从代码层面全面解读 Paperless-ngx 中缩略图与预览图的生成、缓存读取、失败处理以及状态展示的完整链路。

---

## 一、核心文件速览

| 模块 | 文件 | 作用 |
|------|------|------|
| 后端消费流程 | [consumer.py](file:///d:/fz/0601/solo-dogfeeding/code/63-paperless-ngx/src/documents/consumer.py) | 文档消费核心，调用解析器生成缩略图 |
| 后端视图层 | [views.py](file:///d:/fz/0601/solo-dogfeeding/code/63-paperless-ngx/src/documents/views.py) | 提供 `/thumb/` 和 `/preview/` API 端点 |
| 后端缓存层 | [caching.py](file:///d:/fz/0601/solo-dogfeeding/code/63-paperless-ngx/src/documents/caching.py) | 缩略图修改时间缓存 key 管理 |
| 后端条件判断 | [conditionals.py](file:///d:/fz/0601/solo-dogfeeding/code/63-paperless-ngx/src/documents/conditionals.py) | HTTP 缓存的 ETag / Last-Modified 计算 |
| 后端模型 | [models.py](file:///d:/fz/0601/solo-dogfeeding/code/63-paperless-ngx/src/documents/models.py) | `Document.thumbnail_path` 等属性定义 |
| 后端任务 | [tasks.py](file:///d:/fz/0601/solo-dogfeeding/code/63-paperless-ngx/src/documents/tasks.py) | `update_document_content_maybe_archive_file` 等异步任务中的缩略图重生成 |
| 后端管理命令 | [document_thumbnails.py](file:///d:/fz/0601/solo-dogfeeding/code/63-paperless-ngx/src/documents/management/commands/document_thumbnails.py) | `document_thumbnails` 命令：批量重建缩略图 |
| 后端进度管理 | [helpers.py](file:///d:/fz/0601/solo-dogfeeding/code/63-paperless-ngx/src/documents/plugins/helpers.py) | WebSocket 进度推送（ProgressManager） |
| 前端服务 | [document.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/63-paperless-ngx/src-ui/src/app/services/rest/document.service.ts) | `getPreviewUrl()` / `getThumbUrl()` 构造 URL |
| 前端 WebSocket | [websocket-status.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/63-paperless-ngx/src-ui/src/app/services/websocket-status.service.ts) | 状态消息接收与翻译 |
| 前端预览弹窗 | [preview-popup.component.ts](file:///d:/fz/0601/solo-dogfeeding/code/63-paperless-ngx/src-ui/src/app/components/common/preview-popup/preview-popup.component.ts) | 列表页鼠标悬停预览弹窗 |
| 前端详情页 | [document-detail.component.ts](file:///d:/fz/0601/solo-dogfeeding/code/63-paperless-ngx/src-ui/src/app/components/document-detail/document-detail.component.ts) | 文档详情页的预览加载 |

---

## 二、缩略图（Thumbnail）生成流程

### 2.1 文档消费时生成（主路径）

缩略图的生成发生在文档消费（consume）流程中，由 `ConsumerPlugin.run()` 驱动。关键代码位于 [consumer.py:L526-L536](file:///d:/fz/0601/solo-dogfeeding/code/63-paperless-ngx/src/documents/consumer.py#L526-L536)：

```python
self.log.debug(f"Generating thumbnail for {self.filename}...")
self._send_progress(
    70,
    100,
    ProgressStatusOptions.WORKING,
    ConsumerStatusShortMessage.GENERATING_THUMBNAIL,  # "generating_thumbnail"
)
thumbnail = document_parser.get_thumbnail(
    self.working_copy,
    mime_type,
)
```

进度说明：
- `20/100` → 开始解析文档（`PARSING_DOCUMENT`）
- `70/100` → 开始生成缩略图（`GENERATING_THUMBNAIL`）
- `90/100` → 解析日期（`PARSE_DATE`）
- `95/100` → 保存文档（`SAVE_DOCUMENT`）
- `100/100` → 完成（`FINISHED`）

### 2.2 缩略图的存储路径

定义在 [models.py:L479-L484](file:///d:/fz/0601/solo-dogfeeding/code/63-paperless-ngx/src/documents/models.py#L479-L484)：

```python
@property
def thumbnail_path(self) -> Path:
    webp_file_name = f"{self.pk:07}.webp"
    webp_file_path = settings.THUMBNAIL_DIR / Path(webp_file_name)
    return webp_file_path.resolve()
```

规则：
- 格式固定为 **WebP**
- 文件名 = 文档 ID 的 7 位零填充，如文档 ID `123` → `0000123.webp`
- 存储目录由 `settings.THUMBNAIL_DIR` 决定

### 2.3 缩略图写入磁盘

在消费流程的文件写入阶段，通过 `_write()` 方法将临时生成的缩略图移动到最终位置 [consumer.py:L693-L696](file:///d:/fz/0601/solo-dogfeeding/code/63-paperless-ngx/src/documents/consumer.py#L693-L696)：

```python
with FileLock(settings.MEDIA_LOCK):
    # ... 原始文件写入 ...
    self._write(
        thumbnail,
        document.thumbnail_path,
    )
    # ... 归档文件写入 ...
```

注意：使用 `FileLock(settings.MEDIA_LOCK)` 做文件级互斥锁，避免多进程并发写入冲突。

### 2.4 通过管理命令批量重建

`document_thumbnails` 管理命令提供了手动重建能力，见 [document_thumbnails.py:L11-L30](file:///d:/fz/0601/solo-dogfeeding/code/63-paperless-ngx/src/documents/management/commands/document_thumbnails.py#L11-L30)：

```python
def _process_document(doc_id: int) -> None:
    document: Document = Document.objects.get(id=doc_id)
    parser_class = get_parser_registry().get_parser_for_file(...)
    with parser_class() as parser:
        thumb = parser.get_thumbnail(document.source_path, document.mime_type)
        shutil.move(thumb, document.thumbnail_path)
```

支持 `--document <id>` 指定单文档，或全量重建；支持多进程并行处理。

### 2.5 异步任务中的重生成

当文档内容需要重新解析时（例如 OCR 重新处理），`update_document_content_maybe_archive_file` 任务会重新生成缩略图，见 [tasks.py:L316-L376](file:///d:/fz/0601/solo-dogfeeding/code/63-paperless-ngx/src/documents/tasks.py#L316-L376)：

```python
thumbnail = parser.get_thumbnail(document.source_path, mime_type)
# ...
with FileLock(settings.MEDIA_LOCK):
    shutil.move(thumbnail, document.thumbnail_path)
```

---

## 三、预览（Preview）生成机制

**关键点：预览并没有单独的"生成"步骤，而是直接返回原始文件或归档 PDF 文件。**

### 3.1 API 端点

两个端点都定义在 `DocumentViewSet` 中：

**预览端点** [views.py:L1544-L1563](file:///d:/fz/0601/solo-dogfeeding/code/63-paperless-ngx/src/documents/views.py#L1544-L1563)：
- 路径：`/api/documents/<id>/preview/`
- 参数：`?original=true`（强制使用原始文件）、`?version=<id>`（指定版本）

```python
def preview(self, request, pk=None):
    # ... 权限检查、版本解析 ...
    return serve_file(
        doc=file_doc,
        use_archive=not self.original_requested(request) and file_doc.has_archive_version,
        disposition="inline",  # 浏览器内联展示
    )
```

**缩略图端点** [views.py:L1568-L1583](file:///d:/fz/0601/solo-dogfeeding/code/63-paperless-ngx/src/documents/views.py#L1568-L1583)：
- 路径：`/api/documents/<id>/thumb/`
- 返回 WebP 图片流

```python
def thumb(self, request, pk=None):
    # ... 权限检查、版本解析 ...
    handle = file_doc.thumbnail_file
    return FileResponse(handle, content_type="image/webp")
```

### 3.2 serve_file() 核心逻辑

[views.py:L4473-L4522](file:///d:/fz/0601/solo-dogfeeding/code/63-paperless-ngx/src/documents/views.py#L4473-L4522) 的 `serve_file()` 是预览的核心分发函数：

```python
def serve_file(*, doc, use_archive, disposition, follow_formatting=False):
    if use_archive:
        file_handle = doc.archive_file        # 归档 PDF
        mime_type = "application/pdf"
    else:
        file_handle = doc.source_file         # 原始文件
        mime_type = doc.mime_type
        # CSV 预览特判：转为 text/plain 方便浏览器内联展示
        if mime_type in {"application/csv", "text/csv"} and disposition == "inline":
            mime_type = "text/plain"
    # ... 构造带 Unicode 安全文件名的 Content-Disposition 头 ...
    return FileResponse(file_handle, content_type=mime_type)
```

选择策略（`use_archive`）：
1. 用户未显式传 `?original=true`
2. 文档存在归档版本（`has_archive_version`）
3. 满足以上两条 → 使用归档 PDF；否则使用原始文件

### 3.3 前端 URL 构造

[document.service.ts:L215-L237](file:///d:/fz/0601/solo-dogfeeding/code/63-paperless-ngx/src-ui/src/app/services/rest/document.service.ts#L215-L237)：

```typescript
getPreviewUrl(id, original = false, versionID = null): string {
  let url = new URL(this.getResourceUrl(id, 'preview'))
  if (this._searchQuery) url.hash = `#search="${this.searchQuery}"`
  if (original)   url.searchParams.append('original', 'true')
  if (versionID)  url.searchParams.append('version', versionID.toString())
  return url.toString()
}

getThumbUrl(id, versionID = null): string {
  let url = new URL(this.getResourceUrl(id, 'thumb'))
  if (versionID) url.searchParams.append('version', versionID.toString())
  return url.toString()
}
```

---

## 四、缓存读取机制

Paperless-ngx 使用 **两层缓存**：Django 缓存（内存/Redis）+ HTTP 浏览器缓存。

### 4.1 Django 缓存层

**缩略图修改时间缓存**，定义在 [caching.py:L329-L346](file:///d:/fz/0601/solo-dogfeeding/code/63-paperless-ngx/src/documents/caching.py#L329-L346)：

```python
def get_thumbnail_modified_key(document_id: int) -> str:
    return f"doc_{document_id}_thumbnail_modified"
```

实际使用在 [conditionals.py:L120-L146](file:///d:/fz/0601/solo-dogfeeding/code/63-paperless-ngx/src/documents/conditionals.py#L120-L146) 的 `thumbnail_last_modified()`：

```python
def thumbnail_last_modified(request, pk: int) -> datetime | None:
    doc_key = get_thumbnail_modified_key(doc.id)
    cache_hit = cache.get(doc_key)
    if cache_hit is not None:
        cache.touch(doc_key, CACHE_50_MINUTES)  # 命中则刷新 TTL
        return cache_hit

    # 未命中：读文件系统 mtime
    last_modified = datetime.fromtimestamp(doc.thumbnail_path.stat().st_mtime, tz=UTC)
    cache.set(doc_key, last_modified, CACHE_50_MINUTES)
    return last_modified
```

TTL 常量在 [caching.py:L46-L48](file:///d:/fz/0601/solo-dogfeeding/code/63-paperless-ngx/src/documents/caching.py#L46-L48)：
- `CACHE_1_MINUTE = 60`
- `CACHE_5_MINUTES = 5 * 60`
- `CACHE_50_MINUTES = 50 * 60`

**缓存失效**：文档更新时统一清理，见 [caching.py:L336-L345](file:///d:/fz/0601/solo-dogfeeding/code/63-paperless-ngx/src/documents/caching.py#L336-L345)：

```python
def clear_document_caches(document_id: int) -> None:
    cache.delete_many([
        get_suggestion_cache_key(document_id),
        get_metadata_cache_key(document_id),
        get_thumbnail_modified_key(document_id),  # ← 缩略图时间戳缓存
    ])
```

### 4.2 HTTP 缓存层（浏览器）

通过 Django 的 `@condition` / `@last_modified` 装饰器实现 HTTP 协商缓存。

| 端点 | 装饰器 | ETag 来源 | Last-Modified 来源 |
|------|--------|-----------|-------------------|
| `/preview/` | `@condition(etag_func=preview_etag, last_modified_func=preview_last_modified)` | 文档 checksum（original）或 archive_checksum | `doc.modified` |
| `/thumb/` | `@last_modified(thumbnail_last_modified)` | （无） | 缩略图文件 mtime（优先从 Django 缓存读） |
| `/metadata/` | `@condition(etag_func=metadata_etag, ...)` | `doc.checksum` | `doc.modified` |

`preview_etag` 实现 [conditionals.py:L94-L106](file:///d:/fz/0601/solo-dogfeeding/code/63-paperless-ngx/src/documents/conditionals.py#L94-L106)：

```python
def preview_etag(request, pk: int) -> str | None:
    use_original = request.query_params.get("original") == "true"
    return doc.checksum if use_original else doc.archive_checksum
```

此外所有端点都加了 `@cache_control(no_cache=True)`，表示浏览器每次都做协商验证（304），但不直接使用过期缓存。

---

## 五、失败处理与重试

### 5.1 缩略图生成失败

在消费流程中，缩略图生成被包裹在 `try/except` 中，任何异常都会导致整个消费任务失败，见 [consumer.py:L555-L568](file:///d:/fz/0601/solo-dogfeeding/code/63-paperless-ngx/src/documents/consumer.py#L555-L568)：

```python
except ParseError as e:
    self._fail(str(e), "...", exc_info=True, exception=e)
except Exception as e:
    self._fail(str(e), "...", exc_info=True, exception=e)
```

`_fail()` 方法会：
1. 发送进度 `100/100 FAILED`，附带错误消息
2. 记录错误日志
3. 抛出 `ConsumerError` 终止消费流程

**结论：缩略图生成没有独立的重试机制，它和文档消费是同一个原子事务，失败则整个消费失败。**

### 5.2 前端预览加载失败

**预览弹窗组件** [preview-popup.component.ts:L109-L115](file:///d:/fz/0601/solo-dogfeeding/code/63-paperless-ngx/src-ui/src/app/components/common/preview-popup/preview-popup.component.ts#L109-L115)：

```typescript
onError(event: any) {
  if (event.name == 'PasswordException') {
    this.requiresPassword = true   // PDF 需要密码的特殊标记
  } else {
    this.error = true              // 通用错误标记
  }
}
```

**文档详情页** [document-detail.component.ts:L509-L514](file:///d:/fz/0601/solo-dogfeeding/code/63-paperless-ngx/src-ui/src/app/components/document-detail/document-detail.component.ts#L509-L514)：

```typescript
.subscribe({
  next: (res) => (this.previewText = res.toString()),
  error: (err) =>
    (this.previewText = `An error occurred loading content: ${err.message ?? err.toString()}`),
})
```

文本类文档预览失败时直接在页面显示错误信息，不中断用户操作。

### 5.3 后端 API 端点异常

两个端点都捕获 `FileNotFoundError` 并转为 `Http404`：
- `/thumb/` → 缩略图文件不存在
- `/preview/` → 原始文件或归档文件不存在

### 5.4 手动修复：重建缩略图

由于消费失败时文档并未入库，用户能看到的"缩略图缺失"通常是文件系统损坏或迁移导致。此时使用管理命令修复：

```bash
document_thumbnails                # 重建所有
document_thumbnails --document 42  # 只重建 ID=42
```

这是事实上的"重试"方式。

---

## 六、状态展示链路（WebSocket 实时进度）

整个状态推送是 **后端 ProgressManager → Django Channels (WebSocket) → 前端 WebsocketStatusService** 的三层架构。

### 6.1 后端：ProgressManager 发送消息

[helpers.py:L119-L150](file:///d:/fz/0601/solo-dogfeeding/code/63-paperless-ngx/src/documents/plugins/helpers.py#L119-L150)：

```python
class ProgressManager(BaseStatusManager):
    def send_progress(self, status, message, current_progress, max_progress, *, document_id=None, ...):
        data: ProgressUpdateData = {
            "filename": self.filename,
            "task_id": self.task_id,
            "current_progress": current_progress,   # e.g. 70
            "max_progress": max_progress,           # e.g. 100
            "status": status,                       # "WORKING"
            "message": message,                     # "generating_thumbnail"
            "document_id": document_id,
            # ... 权限字段 ...
        }
        payload = {"type": "status_update", "data": data}
        self.send(payload)   # 通过 channel layer 发到 "status_updates" group
```

状态值枚举 [helpers.py:L15-L19](file:///d:/fz/0601/solo-dogfeeding/code/63-paperless-ngx/src/documents/plugins/helpers.py#L15-L19)：
- `STARTED` / `WORKING` / `SUCCESS` / `FAILED`

### 6.2 消费流程中的状态节点

[consumer.py:L117](file:///d:/fz/0601/solo-dogfeeding/code/63-paperless-ngx/src/documents/consumer.py#L117) 定义了生成缩略图时的短消息：

```python
class ConsumerStatusShortMessage(StrEnum):
    GENERATING_THUMBNAIL = "generating_thumbnail"
    # ... 其他状态 ...
```

完整状态流转：

| 阶段 | current/max | status | message |
|------|------------|--------|---------|
| 开始解析 | 20/100 | WORKING | `parsing_document` |
| 生成缩略图 | 70/100 | WORKING | `generating_thumbnail` |
| 解析日期 | 90/100 | WORKING | `parse_date` |
| 保存文档 | 95/100 | WORKING | `save_document` |
| 成功 | 100/100 | SUCCESS | `finished` |
| 失败 | 100/100 | FAILED | 具体错误信息 |

### 6.3 前端：状态接收与翻译

[websocket-status.service.ts:L25-L42](file:///d:/fz/0601/solo-dogfeeding/code/63-paperless-ngx/src-ui/src/app/services/websocket-status.service.ts#L25-L42) 维护了消息字典：

```typescript
export const FILE_STATUS_MESSAGES = {
  parsing_document:    $localize`Processing document...`,
  generating_thumbnail: $localize`Generating thumbnail...`,
  parse_date:          $localize`Retrieving date from document...`,
  save_document:       $localize`Saving document...`,
  finished:            $localize`Finished.`,
  // ...
}
```

前端进度换算 [websocket-status.service.ts:L61-L74](file:///d:/fz/0601/solo-dogfeeding/code/63-paperless-ngx/src-ui/src/app/services/websocket-status.service.ts#L61-L74)：

```typescript
getProgress(): number {
  switch (this.phase) {
    case FileStatusPhase.STARTED:   return 0.0
    case FileStatusPhase.UPLOADING: return (current / max) * 0.2          // 上传占 20%
    case FileStatusPhase.WORKING:   return (current / max) * 0.8 + 0.2    // 服务端处理占 80%
    case FileStatusPhase.SUCCESS:
    case FileStatusPhase.FAILED:    return 1.0
  }
}
```

因此当后端发送 `generating_thumbnail`（70/100 WORKING）时，前端显示的总进度是：
`0.2 + (70/100) * 0.8 = 0.76` → **76%**

### 6.4 PaperlessTask 数据库记录

除了实时 WebSocket 推送，每个消费任务还会持久化到 `PaperlessTask` 表中，定义在 [models.py:L664-L741](file:///d:/fz/0601/solo-dogfeeding/code/63-paperless-ngx/src/documents/models.py#L664-L741)：

```python
class PaperlessTask(ModelWithOwner):
    class Status(models.TextChoices):
        PENDING = "pending"
        STARTED = "started"
        SUCCESS = "success"
        FAILURE = "failure"
        REVOKED = "revoked"

    task_id        # Celery task ID
    task_type      # "consume_file" 等
    trigger_source # web_ui / api_upload / folder_consume / email_consume / system / manual
    status         # 当前状态
    # ... 时间戳、错误信息快照 ...
```

用户可以通过 `/api/tasks/` 接口查询历史任务（包括缩略图生成阶段的失败）。

---

## 七、整体架构时序图

```
用户上传文档
    │
    ▼
┌──────────────────────────────────────────────────────────┐
│                    后端 consume_file 任务                 │
│                                                          │
│  1. PARSING_DOCUMENT (进度 20/100)                       │
│     └─ parser.parse() 提取文本/OCR                        │
│                                                          │
│  2. GENERATING_THUMBNAIL (进度 70/100)  ◄───────┐        │
│     └─ parser.get_thumbnail() → WebP 临时文件   │        │
│         WebSocket 推送 "generating_thumbnail" ──┘        │
│                                                          │
│  3. PARSE_DATE (进度 90/100)                             │
│  4. SAVE_DOCUMENT (进度 95/100)                          │
│     └─ transaction + FileLock 写入:                      │
│        • 原始文件 → source_path                          │
│        • 缩略图   → thumbnail_path (0000123.webp)        │
│        • 归档 PDF → archive_path (可选)                  │
│                                                          │
│  5. FINISHED (进度 100/100)  SUCCESS                     │
│     或 FAILED (整个事务回滚)                              │
└──────────────────────────────────────────────────────────┘
    │
    ▼
用户访问 /api/documents/<id>/thumb/
    │
    ├─ @last_modified(thumbnail_last_modified)
    │     └─ Django cache → doc_{id}_thumbnail_modified
    │        (命中: 刷新 TTL 50min; 未命中: 读 st_mtime 回填)
    │
    └─ FileResponse(doc.thumbnail_file, "image/webp")


用户访问 /api/documents/<id>/preview/
    │
    ├─ @condition(etag_func=preview_etag, ...)
    │     └─ ETag = archive_checksum or checksum
    │
    └─ serve_file(use_archive=has_archive_version && !?original=true)
           ├─ True  → doc.archive_file   (application/pdf)
           └─ False → doc.source_file    (原始 mime_type)
```

---

## 八、设计要点总结

1. **缩略图格式统一为 WebP**：文件名仅由文档 ID 决定，便于反向查找。
2. **预览 = 文件直出**：没有额外的渲染服务，依赖浏览器对 PDF/图片/文本的原生支持，架构简洁。
3. **HTTP 协商缓存 + Django 应用缓存双层设计**：既降低浏览器重复请求，又避免频繁读取文件系统 mtime。
4. **缩略图生成与消费事务绑定**：失败则整体失败，通过 `document_thumbnails` 命令做事后补偿。
5. **WebSocket + 数据库双通道状态**：实时进度走 Channels，持久化查询走 `PaperlessTask` 表。
6. **文件写入用 FileLock 保护**：`settings.MEDIA_LOCK` 防止多 worker 并发写同一文件。
