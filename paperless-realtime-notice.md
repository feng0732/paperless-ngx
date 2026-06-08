# Paperless-ngx 实时通知机制完整分析

## 概述

Paperless-ngx 采用 **Django Channels + WebSocket** 方案实现实时通知，消息中间件使用 **Redis Pub/Sub**。系统不使用 SSE（Server-Sent Events），唯一的流式响应是 AI Chat 功能使用的 `StreamingHttpResponse`。

实时通知支持三种事件类型：
| 事件类型 | 用途 | 权限检查 |
|---------|------|---------|
| `status_update` | 文档消费/处理进度推送 | ✅ 按用户/组过滤 |
| `document_updated` | 文档元数据更新通知 | ✅ 按用户/组过滤 |
| `documents_deleted` | 文档批量删除通知 | ❌ 所有已认证用户 |

---

## 一、配置与路由层

### 1.1 ASGI 入口配置

文件：[asgi.py](file:///d:/fz/0601/solo-dogfeeding/code/111-paperless-ngx/src/paperless/asgi.py)

```python
application = ProtocolTypeRouter(
    {
        "http": get_asgi_application(),
        "websocket": AuthMiddlewareStack(URLRouter(websocket_urlpatterns)),
    },
)
```

- `ProtocolTypeRouter` 根据协议类型分发：HTTP 请求走传统 Django，WebSocket 请求走 Channels
- `AuthMiddlewareStack` 将 Django session 认证注入 WebSocket scope，使 consumer 能获取当前用户
- 启动顺序至关重要：先初始化 Django ASGI app（加载 AppRegistry），再导入 consumers

### 1.2 WebSocket URL 路由

文件：[urls.py](file:///d:/fz/0601/solo-dogfeeding/code/111-paperless-ngx/src/paperless/urls.py#L420-L422)

```python
websocket_urlpatterns = [
    path("ws/status/", StatusConsumer.as_asgi()),
]
```

前端连接地址由环境配置动态生成：
- 开发环境：`ws://localhost:8000/ws/status/`
- 生产环境：根据当前页面协议自动选择 `ws:` 或 `wss:`

文件：[environment.prod.ts](file:///d:/fz/0601/solo-dogfeeding/code/111-paperless-ngx/src-ui/src/environments/environment.prod.ts#L10-L12)

```typescript
webSocketHost: window.location.host,
webSocketProtocol: window.location.protocol == 'https:' ? 'wss:' : 'ws:',
webSocketBaseUrl: base_url.pathname + 'ws/',
```

### 1.3 Channel Layer 配置（Redis Pub/Sub）

文件：[settings/__init__.py](file:///d:/fz/0601/solo-dogfeeding/code/111-paperless-ngx/src/paperless/settings/__init__.py#L244-L281)

```python
_CHANNELS_BACKEND = os.environ.get(
    "PAPERLESS_CHANNELS_BACKEND",
    "channels_redis.pubsub.RedisPubSubChannelLayer",
)
CHANNEL_LAYERS = {
    "default": {
        "BACKEND": _CHANNELS_BACKEND,
    },
}
if _CHANNELS_BACKEND.startswith("channels_redis."):
    CHANNEL_LAYERS["default"]["CONFIG"] = {
        "hosts": [_CHANNELS_REDIS_URL],
        "capacity": 2000,   # 单频道消息队列容量
        "expiry": 15,       # 消息过期时间（秒）
    }
```

Redis URL 解析支持多种格式：普通 TCP、Unix Socket，以及 Celery/Channels 两种风格的互转，详见 [custom.py](file:///d:/fz/0601/solo-dogfeeding/code/111-paperless-ngx/src/paperless/settings/custom.py#L29-L64)。

---

## 二、事件产生层（所有事件入口完整梳理）

### 2.1 事件发送管理器

文件：[helpers.py](file:///d:/fz/0601/solo-dogfeeding/code/111-paperless-ngx/src/documents/plugins/helpers.py)

系统提供两个上下文管理器类封装事件发送：

#### `BaseStatusManager` — 基类
- `__enter__` / `__exit__` 自动管理 channel layer 生命周期
- `send(payload)` 调用 `async_to_sync(channel_layer.group_send)("status_updates", payload)` 将消息广播到 `status_updates` 组
- `__exit__` 时 flush channel layer，确保消息发出

#### `ProgressManager` — 消费进度
- 构造时接收 `filename` 和 `task_id`
- `send_progress(status, message, current_progress, max_progress, ...)` 发送 `status_update` 事件
- 自动附带权限信息：`owner_id`、`users_can_view`、`groups_can_view`

#### `DocumentsStatusManager` — 文档变更
- `send_documents_deleted(documents: list[int])` 发送 `documents_deleted` 事件
- `send_document_updated(document_id, modified, ...)` 发送 `document_updated` 事件

### 2.2 事件源一：文档消费进度（status_update）

**触发入口**：Celery 任务 `consume_file`

文件：[tasks.py](file:///d:/fz/0601/solo-dogfeeding/code/111-paperless-ngx/src/documents/tasks.py#L123-L220)

```python
@shared_task(bind=True)
def consume_file(self: Task, input_doc: ConsumableDocument, overrides=None):
    with ProgressManager(
        overrides.filename or input_doc.original_file.name,
        self.request.id,
    ) as status_mgr:
        for plugin_class in plugins:
            plugin = plugin_class(input_doc, overrides, status_mgr, tmp_dir, self.request.id)
            plugin.setup()
            plugin.run()  # 内部会多次调用 _send_progress
```

**实际发送位置**：`ConsumerPlugin._send_progress()`

文件：[consumer.py](file:///d:/fz/0601/solo-dogfeeding/code/111-paperless-ngx/src/documents/consumer.py#L213-L232)

```python
def _send_progress(self, current_progress, max_progress, status, message=None, document_id=None):
    self.status_mgr.send_progress(
        status, message, current_progress, max_progress,
        document_id=document_id,
        owner_id=self.metadata.owner_id,
        users_can_view=(self.metadata.view_users or []) + (self.metadata.change_users or []),
        groups_can_view=(self.metadata.view_groups or []) + (self.metadata.change_groups or []),
    )
```

**典型进度节点**：
| 进度 | 阶段 | 消息 |
|-----|------|------|
| 20% | WORKING | `parsing_document` — 解析文档 |
| 70% | WORKING | `generating_thumbnail` — 生成缩略图 |
| 90% | WORKING | `parse_date` — 提取日期 |
| 95% | WORKING | `save_document` — 保存文档 |
| 100% | SUCCESS/FAILED | 完成或失败 |

**前端本地上传阶段**（在发送到后端之前）：
文件：[upload-documents.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/111-paperless-ngx/src-ui/src/app/services/upload-documents.service.ts)

```typescript
public uploadFile(file: File) {
    let status = this.websocketStatusService.newFileUpload(file.name)
    this.documentService.uploadDocument(formData).subscribe({
        next: (event) => {
            if (event.type == HttpEventType.UploadProgress) {
                status.updateProgress(FileStatusPhase.UPLOADING, event.loaded, event.total)
            } else if (event.type == HttpEventType.Response) {
                status.taskId = event.body['task_id']
            }
        },
        error: (error) => { this.websocketStatusService.fail(status, ...) }
    })
}
```

### 2.3 事件源二：文档更新通知（document_updated）—— 全部 7 个入口

**核心信号机制**：Django 自定义信号 `document_updated`

信号定义：[signals/__init__.py](file:///d:/fz/0601/solo-dogfeeding/code/111-paperless-ngx/src/documents/signals/__init__.py)

```python
document_consumption_started = Signal()
document_consumption_finished = Signal()
document_updated = Signal()
```

**信号连接**：AppConfig.ready() 中绑定

文件：[apps.py](file:///d:/fz/0601/solo-dogfeeding/code/111-paperless-ngx/src/documents/apps.py#L10-L33)

```python
def ready(self):
    document_consumption_finished.connect(add_inbox_tags)
    document_consumption_finished.connect(set_correspondent)
    # ... 更多业务处理器
    document_updated.connect(run_workflows_updated)
    document_updated.connect(send_websocket_document_updated)  # ← WebSocket 发送
```

**信号处理器实现**：

文件：[handlers.py](file:///d:/fz/0601/solo-dogfeeding/code/111-paperless-ngx/src/documents/signals/handlers.py#L832-L851)

```python
def send_websocket_document_updated(sender, document: Document, **kwargs):
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

---

#### 入口 ①：REST 单文档更新接口 `DocumentViewSet.update()`

文件：[views.py](file:///d:/fz/0601/solo-dogfeeding/code/111-paperless-ngx/src/documents/views.py#L1135-L1183)

```
HTTP PATCH /api/documents/{id}/
    ↓
DocumentViewSet.update(request, *args, **kwargs)
    ├─ serializer.is_valid() + perform_update()  ← 保存 DB
    ├─ 如果 content 更新在不同 version doc 上 → content_doc.save()
    ├─ get_backend().add_or_update(refreshed_doc)  ← 更新搜索索引
    └─ document_updated.send(sender=self.__class__, document=refreshed_doc)
            ↓
        send_websocket_document_updated() → WebSocket 广播
```

同时触发 `run_workflows_updated()` 工作流执行，工作流修改后也会再次触发 `document_updated`。

---

#### 入口 ②：REST 版本删除接口 `DocumentViewSet.delete_version()`

文件：[views.py](file:///d:/fz/0601/solo-dogfeeding/code/111-paperless-ngx/src/documents/views.py#L2000-L2055)

```
HTTP POST /api/documents/{id}/versions/{version_id}/delete/
    ↓
校验权限 → version_doc.delete() → 更新搜索索引 → 写审计日志
    ↓
document_updated.send(sender=self.__class__, document=root_doc)
    ↓
send_websocket_document_updated()
```

---

#### 入口 ③：REST 版本标签更新接口 `DocumentViewSet.update_version_label()`

文件：[views.py](file:///d:/fz/0601/solo-dogfeeding/code/111-paperless-ngx/src/documents/views.py#L2084-L2122)

```
HTTP PATCH /api/documents/{id}/versions/{version_id}/
    ↓
更新 version_doc.version_label → 写审计日志
    ↓
document_updated.send(sender=self.__class__, document=root_doc)
    ↓
send_websocket_document_updated()
```

---

#### 入口 ④：REST 批量编辑接口 `BulkEditView` → `bulk_update_documents` 任务

批量编辑视图层：[views.py](file:///d:/fz/0601/solo-dogfeeding/code/111-paperless-ngx/src/documents/views.py#L2797-L2970)

```
HTTP POST /api/documents/bulk_edit/
    ↓
BulkEditView._execute_document_action()
    ↓ （根据 method 分发）
    ├─ set_correspondent() / set_document_type() / set_storage_path()
    ├─ add_tag() / remove_tag() / modify_tags()
    ├─ modify_custom_fields() / set_permissions() / modify_asns()
    └─ ... 其他方法
            ↓ 内部异步调用
bulk_update_documents.apply_async(kwargs={"document_ids": affected_docs})
```

Celery 任务 `bulk_update_documents` 实现：
文件：[tasks.py](file:///d:/fz/0601/solo-dogfeeding/code/111-paperless-ngx/src/documents/tasks.py#L252-L275)

```python
@shared_task
def bulk_update_documents(document_ids) -> None:
    documents = Document.objects.filter(id__in=document_ids)
    for doc in documents:
        clear_document_caches(doc.pk)
        document_updated.send(
            sender=None,
            document=doc,
            logging_group=uuid.uuid4(),
        )
        post_save.send(Document, instance=doc, created=False)
    # 批量更新搜索索引 + AI 索引
    with get_backend().batch_update() as batch:
        for doc in documents:
            batch.add_or_update(doc)
```

每个被修改的文档都会发送一次 `document_updated` 信号，对应一条 WebSocket 推送。

批量编辑函数（以 `set_correspondent` 为例）：
文件：[bulk_edit.py](file:///d:/fz/0601/solo-dogfeeding/code/111-paperless-ngx/src/documents/bulk_edit.py#L111-L131)

```python
def set_correspondent(doc_ids: list[int], correspondent: Correspondent):
    # qs.update() 直接批量改 DB，不发 post_save 信号
    affected_docs = list(qs.values_list("pk", flat=True))
    qs.update(correspondent=correspondent)
    # 显式异步发送 document_updated 信号
    bulk_update_documents.apply_async(kwargs={"document_ids": affected_docs})
```

> **设计要点**：Django ORM 的 `QuerySet.update()` 不会触发 `post_save` 信号，因此 paperless-ngx 额外封装了 `bulk_update_documents` 任务来显式发送 `document_updated` 信号，保证实时通知不丢失。

---

#### 入口 ⑤：REST 上传新版本接口 `DocumentViewSet.update_version()`

文件：[views.py](file:///d:/fz/0601/solo-dogfeeding/code/111-paperless-ngx/src/documents/views.py#L1890-L1945)

```
HTTP POST /api/documents/{id}/update_version/ (multipart/form-data)
    ↓
consume_file.apply_async(
    input_doc=ConsumableDocument(..., root_document_id=root_doc.id),
    overrides=overrides,
    headers={"trigger_source": PaperlessTask.TriggerSource.WEB_UI},
)
    ↓
→ 消费过程中持续发 status_update 进度
→ 消费完成后发 status_update SUCCESS 事件
```

这是通过 `consume_file` 任务走消费流程，不是直接 `document_updated`，见 2.2 节。

---

#### 入口 ⑥：标签层级变更 `TagViewSet.perform_update()` → `update_document_parent_tags()`

文件：[views.py](file:///d:/fz/0601/solo-dogfeeding/code/111-paperless-ngx/src/documents/views.py#L634-L639)

```python
def perform_update(self, serializer):
    old_parent = self.get_object().get_parent()
    tag = serializer.save()
    new_parent = tag.get_parent()
    if new_parent and old_parent != new_parent:
        update_document_parent_tags(tag, new_parent)
```

文件：[tasks.py](file:///d:/fz/0601/solo-dogfeeding/code/111-paperless-ngx/src/documents/tasks.py#L576-L590)

```python
def update_document_parent_tags(tag: Tag, new_parent: Tag):
    doc_ids = list(Document.objects.filter(tags=tag).values_list("pk", flat=True))
    # ... 给文档补齐父标签层级 ...
    if affected_ids:
        bulk_update_documents.apply_async(kwargs={"document_ids": affected_ids})
```

---

#### 入口 ⑦：定时工作流 `check_scheduled_workflows()` —— 直接推送，跳过信号

文件：[tasks.py](file:///d:/fz/0601/solo-dogfeeding/code/111-paperless-ngx/src/documents/tasks.py#L436-L573)

这是**唯一不通过 `document_updated` 信号**的 `document_updated` 事件源，直接调用信号处理函数：

```python
@shared_task
def check_scheduled_workflows() -> None:
    scheduled_workflows = get_workflows_for_trigger(
        WorkflowTrigger.WorkflowTriggerType.SCHEDULED,
    )
    for workflow in scheduled_workflows:
        for trigger in workflow.triggers.filter(type=SCHEDULED):
            # 1. 按日期字段匹配文档（added / created / modified / custom_date_field）
            documents = Document.objects.filter(root_document__isnull=True, ...)
            # 2. 去重：非 recurring 只跑一次，recurring 按间隔限制
            for document in documents:
                # 3. 执行工作流动作（改标题/标签/对应人/权限/删除等）
                run_workflows(
                    trigger_type=WorkflowTrigger.WorkflowTriggerType.SCHEDULED,
                    workflow_to_run=workflow,
                    document=document,
                )
                # 4. 直接调用 WebSocket 推送（注意：不经过 document_updated 信号）
                send_websocket_document_updated(
                    sender=None,
                    document=document,
                )
```

> **设计要点**：代码注释明确说明 —— *Scheduled workflows dont send document_updated signal, so send a websocket update here to ensure clients are updated*。因为 `run_workflows(SCHEDULED)` 内部工作流修改文档后不会自动触发 `document_updated` 信号，所以此处兜底直接调用信号处理函数发送 WebSocket 推送。

该任务由 Celery Beat 定时调度触发（`crontab(minute="*/5")` 每 5 分钟执行一次）。

### 2.4 事件源三：文档删除通知（documents_deleted）

**触发位置 ①**：批量删除操作 `bulk_edit.delete()`

文件：[bulk_edit.py](file:///d:/fz/0601/solo-dogfeeding/code/111-paperless-ngx/src/documents/bulk_edit.py#L359-L392)

```python
@shared_task
def delete(doc_ids: list[int]) -> Literal["OK"]:
    root_ids = Document.objects.filter(id__in=doc_ids, root_document__isnull=True).values_list("id", flat=True)
    version_ids = Document.objects.filter(root_document_id__in=root_ids).exclude(id__in=doc_ids).values_list("id", flat=True)
    delete_ids = list({*doc_ids, *version_ids})

    Document.objects.filter(id__in=delete_ids).delete()

    with get_backend().batch_update() as batch:
        for id in delete_ids:
            batch.remove(id)

    status_mgr = DocumentsStatusManager()
    status_mgr.send_documents_deleted(delete_ids)  # ← 发送删除事件
```

REST 入口：`DeleteDocumentsView` 调用 `bulk_edit.delete`
文件：[views.py](file:///d:/fz/0601/solo-dogfeeding/code/111-paperless-ngx/src/documents/views.py#L2987-L2997)

```
HTTP POST /api/documents/delete/
    → DeleteDocumentsView._execute_document_action()
    → bulk_edit.delete(doc_ids)
    → DocumentsStatusManager.send_documents_deleted()
```

**触发位置 ②**：单文档删除 `DocumentViewSet.destroy()`

文件：[views.py](file:///d:/fz/0601/solo-dogfeeding/code/111-paperless-ngx/src/documents/views.py#L1204-L1218)

> 注意：单文档删除不直接发送 `documents_deleted` WebSocket 事件，只走 `post_delete` 信号清理文件。REST 批量删除接口会触发。删除标签/对应人时的级联文档刷新通过 `bulk_update_documents.apply_async()` 触发 `document_updated`。

**触发位置 ③**：Tag 树删除等会触发 `bulk_edit.bulk_update_documents.apply_async()`

文件：[views.py](file:///d:/fz/0601/solo-dogfeeding/code/111-paperless-ngx/src/documents/views.py#L3816-L3824)

```python
# 例如删除 Tag 时，其下所有文档需要被标记更新
response = super().destroy(request, *args, **kwargs)
if doc_ids:
    bulk_edit.bulk_update_documents.apply_async(kwargs={"document_ids": doc_ids})
```

注意：此事件不携带权限信息，consumer 直接向所有已认证连接广播（见下节）。

---

## 三、连接维护与消息分发层（Consumer）

文件：[consumers.py](file:///d:/fz/0601/solo-dogfeeding/code/111-paperless-ngx/src/paperless/consumers.py)

### 3.1 连接生命周期

```python
class StatusConsumer(AsyncWebsocketConsumer):
    async def connect(self) -> None:
        if not self._authenticated():          # 未认证直接关闭
            await self.close()
            return
        await self.channel_layer.group_add(     # 加入 status_updates 组
            "status_updates", self.channel_name
        )
        await self.accept()

    async def disconnect(self, code: int) -> None:
        await self.channel_layer.group_discard(  # 离开组
            "status_updates", self.channel_name
        )
```

所有 WebSocket 客户端共享同一个频道组 `status_updates`，权限过滤在 consumer 接收消息时进行。

### 3.2 权限检查机制

```python
async def _can_view(self, data: PermissionsData) -> bool:
    user = self.scope.get("user")
    if user is None:
        return False
    owner_id = data.get("owner_id")
    users_can_view = data.get("users_can_view", [])
    groups_can_view = data.get("groups_can_view", [])

    if user.is_superuser or user.id == owner_id or user.id in users_can_view:
        return True

    return await user.groups.filter(pk__in=groups_can_view).aexists()
```

权限检查通过条件（任一满足即可）：
1. 用户是超级用户
2. 用户是文档所有者
3. 用户 ID 在 `users_can_view` 列表中
4. 用户所属任一 group ID 在 `groups_can_view` 列表中

### 3.3 消息分发方法

Channels 的约定：当 group_send 的 `type` 字段为 `"status_update"` 时，自动调用 consumer 的 `status_update()` 方法（下划线替换点号）。

```python
async def status_update(self, event: StatusUpdatePayload) -> None:
    if not self._authenticated():
        await self.close()
    elif await self._can_view(event["data"]):    # 权限过滤
        await self.send(json.dumps(event))

async def document_updated(self, event: DocumentUpdatedPayload) -> None:
    if not self._authenticated():
        await self.close()
    elif await self._can_view(event["data"]):    # 权限过滤
        await self.send(json.dumps(event))

async def documents_deleted(self, event: DocumentsDeletedPayload) -> None:
    if not self._authenticated():
        await self.close()
    else:
        await self.send(json.dumps(event))       # 无权限过滤，全广播
```

---

## 四、前端接收与处理层

### 4.1 WebSocket 服务核心

文件：[websocket-status.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/111-paperless-ngx/src-ui/src/app/services/websocket-status.service.ts)

#### 连接管理

```typescript
@Injectable({ providedIn: 'root' })
export class WebsocketStatusService {
  private statusWebSocket: WebSocket
  private readonly connectionStatusSubject = new Subject<boolean>()

  connect() {
    this.disconnect()
    this.statusWebSocket = new WebSocket(
      `${environment.webSocketProtocol}//${environment.webSocketHost}${environment.webSocketBaseUrl}status/`
    )
    this.statusWebSocket.onopen = () => this.connectionStatusSubject.next(true)
    this.statusWebSocket.onclose = () => this.connectionStatusSubject.next(false)
    this.statusWebSocket.onerror = () => this.connectionStatusSubject.next(false)
    this.statusWebSocket.onmessage = (ev: MessageEvent) => this.handleMessage(ev)
  }
```

#### 消息路由（onmessage）

```typescript
  this.statusWebSocket.onmessage = (ev: MessageEvent) => {
    const { type, data: messageData } = JSON.parse(ev.data)
    switch (type) {
      case WebsocketStatusType.DOCUMENTS_DELETED:
        this.documentDeletedSubject.next(true)
        break
      case WebsocketStatusType.DOCUMENT_UPDATED:
        this.handleDocumentUpdated(messageData)
        break
      case WebsocketStatusType.STATUS_UPDATE:
        this.handleProgressUpdate(messageData)
        break
    }
  }
```

#### `handleProgressUpdate` — status_update 处理细节

```typescript
handleProgressUpdate(messageData: WebsocketProgressMessage) {
    if (!this.canViewMessage(messageData)) return  // 前端二次权限校验

    let { status, created } = this.get(messageData.task_id, messageData.filename)
    status.updateProgress(FileStatusPhase.WORKING, messageData.current_progress, messageData.max_progress)
    status.message = FILE_STATUS_MESSAGES[messageData.message] || messageData.message
    status.documentId = messageData.document_id

    if (created) {
        this.documentDetectedSubject.next(status)  // ← 新文件首次出现
    }
    if (messageData.status == FileStatusPhase.SUCCESS) {
        status.phase = FileStatusPhase.SUCCESS
        this.documentConsumptionFinishedSubject.next(status)  // ← 成功完成
    } else if (messageData.status == FileStatusPhase.FAILED) {
        status.phase = FileStatusPhase.FAILED
        this.documentConsumptionFailedSubject.next(status)    // ← 失败
    }
}
```

#### `handleDocumentUpdated` — document_updated 处理细节

```typescript
private handleDocumentUpdated(messageData: WebsocketDocumentUpdatedMessage) {
    if (!this.canViewMessage(messageData)) return  // 前端二次权限校验
    this.documentUpdatedSubject.next(messageData)
}
```

#### 前端二次权限校验（兜底）

```typescript
private canViewMessage(messageData: {
  owner_id?: number; users_can_view?: number[]; groups_can_view?: number[]
}): boolean {
  const user: User = this.settingsService.currentUser
  return (
    !messageData.owner_id ||
    user.is_superuser ||
    messageData.owner_id === user.id ||
    messageData.users_can_view?.includes(user.id) ||
    messageData.groups_can_view?.some((groupId) => user.groups?.includes(groupId))
  )
}
```

前后端权限逻辑完全对称，确保即使后端因某种原因漏过滤，前端也不会展示越权信息。

#### 进度状态机

```typescript
export enum FileStatusPhase {
  STARTED = 0,
  UPLOADING = 1,
  WORKING = 2,
  SUCCESS = 3,
  FAILED = 4,
}
```

`FileStatus.getProgress()` 将阶段映射到 0.0–1.0 进度值：
- STARTED → 0%
- UPLOADING → 0%–20%
- WORKING → 20%–100%
- SUCCESS/FAILED → 100%

#### RxJS 事件暴露

组件通过订阅以下 Observable 接收实时事件：
| 方法 | Subject 类型 | 触发时机 |
|-----|-------------|---------|
| `onDocumentDetected()` | `Subject<FileStatus>` | 新文件首次出现在后端处理队列（STARTED） |
| `onDocumentConsumptionFinished()` | `Subject<FileStatus>` | 后端处理成功（SUCCESS） |
| `onDocumentConsumptionFailed()` | `Subject<FileStatus>` | 后端处理失败（FAILED） |
| `onDocumentDeleted()` | `Subject<boolean>` | 任意文档被删除 |
| `onDocumentUpdated()` | `Subject<WebsocketDocumentUpdatedMessage>` | 文档元数据变更（含 document_id、modified） |
| `onConnectionStatus()` | `Observable<boolean>` | WebSocket 连接/断开 |

---

### 4.2 前端所有订阅组件 + 界面刷新逻辑完整列表

WebSocket 连接由根组件启动：

文件：[app.component.ts](file:///d:/fz/0601/solo-dogfeeding/code/111-paperless-ngx/src-ui/src/app/app.component.ts)

```typescript
export class AppComponent implements OnInit, OnDestroy {
  ngOnInit(): void {
    this.websocketStatusService.connect()   // ← 全局唯一连接点
    // 订阅三个消费状态事件用于 toast 通知
    this.successSubscription = this.websocketStatusService
      .onDocumentConsumptionFinished().subscribe((status) => {
        this.tasksService.reload()
        this.toastService.show({          // 成功 toast：Document xxx was added
          content: `Document ${status.filename} was added...`,
          actionName: "Open document",
          action: () => this.router.navigate(['documents', status.documentId]),
        })
      })
    this.failedSubscription = this.websocketStatusService
      .onDocumentConsumptionFailed().subscribe((status) => {
        this.tasksService.reload()
        this.toastService.showError(`Could not add ${status.filename}: ${status.message}`)
      })
    this.newDocumentSubscription = this.websocketStatusService
      .onDocumentDetected().subscribe((status) => {
        this.tasksService.reload()
        this.toastService.show({ content: `Document ${status.filename} is being processed...` })
      })
  }
  ngOnDestroy(): void {
    this.websocketStatusService.disconnect()  // 应用卸载时断开
  }
}
```

---

#### 订阅组件 ①：DocumentListComponent（文档列表页）

文件：[document-list.component.ts](file:///d:/fz/0601/solo-dogfeeding/code/111-paperless-ngx/src-ui/src/app/components/document-list/document-list.component.ts#L262-L272)

```typescript
ngOnInit(): void {
    this.websocketStatusService
      .onDocumentConsumptionFinished()
      .pipe(takeUntil(this.unsubscribeNotifier))
      .subscribe(() => { this.list.reload() })      // ← 消费完成 → 重新加载列表

    this.websocketStatusService.onDocumentDeleted()
      .subscribe(() => { this.list.reload() })      // ← 删除文档 → 重新加载列表
}
```

**刷新效果**：当前页列表数据（分页、排序、过滤条件保留）从后端重新拉取，UI 无感知刷新。

---

#### 订阅组件 ②：DocumentDetailComponent（文档详情页）

文件：[document-detail.component.ts](file:///d:/fz/0601/solo-dogfeeding/code/111-paperless-ngx/src-ui/src/app/components/document-detail/document-detail.component.ts#L764-L695)

```typescript
ngOnInit(): void {
    this.websocketStatusService
      .onDocumentUpdated()
      .pipe(takeUntil(this.unsubscribeNotifier))
      .subscribe((data) => this.handleIncomingDocumentUpdated(data))
}

private handleIncomingDocumentUpdated(data: IncomingDocumentUpdate): void {
    // 只关心当前打开的文档
    if (data.document_id !== this.documentId) return

    // 1. 如果当前正在编辑（网络请求中），先缓存，等网络空闲后再处理
    if (this.networkActive) {
        this.pendingIncomingUpdate = data
        return
    }
    // 2. 如果收到的 modified 时间戳与上次本地保存一致，认为是自己保存的回声，忽略
    if (incomingModified === this.lastLocalSaveModified) {
        this.lastLocalSaveModified = null
        return
    }
    // 3. 如果表单有未保存修改 → 弹确认对话框让用户选择是否覆盖
    if (this.openDocumentService.isDirty(this.document)) {
        this.showIncomingUpdateModal(data.modified)
    } else {
        // 4. 无未保存修改 → 静默重新加载文档 + toast 提示
        this.loadDocument(this.documentId, true)
        this.toastService.showInfo("Document reloaded with latest changes.")
    }
}
```

**刷新效果**：
- 正在编辑时：弹出"文档已被他人修改"确认框，防止覆盖用户编辑
- 空闲时：自动从后端拉取最新数据渲染，所有字段（标题/标签/对应人/自定义字段/笔记等）即时更新

---

#### 订阅组件 ③：SavedViewWidgetComponent（仪表盘保存视图小组件）

文件：[saved-view-widget.component.ts](file:///d:/fz/0601/solo-dogfeeding/code/111-paperless-ngx/src-ui/src/app/components/dashboard/widgets/saved-view-widget/saved-view-widget.component.ts#L128-L133)

```typescript
ngOnInit(): void {
    this.reload()
    this.websocketStatusService
      .onDocumentConsumptionFinished()
      .pipe(takeUntil(this.unsubscribeNotifier))
      .subscribe(() => { this.reload() })    // 新文档消费完成 → 刷新小组件
}
```

**刷新效果**：仪表盘卡片中的文档列表和计数重新拉取。

---

#### 订阅组件 ④：StatisticsWidgetComponent（仪表盘统计小组件）

文件：[statistics-widget.component.ts](file:///d:/fz/0601/solo-dogfeeding/code/111-paperless-ngx/src-ui/src/app/components/dashboard/widgets/statistics-widget/statistics-widget.component.ts#L111-L118)

```typescript
ngOnInit(): void {
    this.reload()
    this.subscription = this.websocketConnectionService
      .onDocumentConsumptionFinished()
      .subscribe(() => { this.reload() })    // 新文档完成 → 刷新统计数据
}
```

**刷新效果**：总文档数、收件箱数、MIME 类型分布、字符数、ASN 当前值等统计指标实时更新。

---

#### 订阅组件 ⑤：UploadFileWidgetComponent（仪表盘上传小组件）

文件：[upload-file-widget.component.ts](file:///d:/fz/0601/solo-dogfeeding/code/111-paperless-ngx/src-ui/src/app/components/dashboard/widgets/upload-file-widget/upload-file-widget.component.ts)

```typescript
// 不使用 .subscribe()，而是通过 getter 直接从 service 读取实时状态
getStatus() { return this.websocketStatusService.getConsumerStatus() }
getStatusUploading() { return this.websocketStatusService.getConsumerStatus(FileStatusPhase.UPLOADING) }
getStatusFailed() { return this.websocketStatusService.getConsumerStatus(FileStatusPhase.FAILED) }
getStatusSuccess() { return this.websocketStatusService.getConsumerStatus(FileStatusPhase.SUCCESS) }
getTotalUploadProgress() { ... }  // 计算所有 UPLOADING 状态的综合进度
```

**刷新效果**：Angular 变更检测自动触发，组件显示：
- 正在上传/处理的文件列表（带进度条）
- 成功 / 失败的文件列表
- 综合进度条
- 汇总摘要（Processing: N, Failed: M, Added: K）

---

#### 订阅组件 ⑥：DocumentVersionDropdownComponent（文档详情版本下拉）

文件：[document-version-dropdown.component.ts](file:///d:/fz/0601/solo-dogfeeding/code/111-paperless-ngx/src-ui/src/app/components/document-detail/document-version-dropdown/document-version-dropdown.component.ts#L195-L275)

```typescript
onVersionFileSelected(event: Event): void {
    // 调用后端上传新版本 API → 返回 task_id
    this.documentsService.uploadVersion(uploadDocumentId, file, label)
      .pipe(
        tap(() => { this.versionUploadState = UploadState.Processing }),
        switchMap((taskId) => merge(
          // 监听对应 task_id 的成功/失败事件（只取第一个）
          this.websocketStatusService.onDocumentConsumptionFinished()
            .pipe(filter((status) => status.taskId === taskId),
                  map(() => ({ state: 'success' }))),
          this.websocketStatusService.onDocumentConsumptionFailed()
            .pipe(filter((status) => status.taskId === taskId),
                  map((status) => ({ state: 'failed', message: status.message }))),
        ).pipe(take(1))),
        switchMap((result) => result.state === 'success'
          ? this.documentsService.getVersions(uploadDocumentId)
          : of(null)),
      )
      .subscribe({
        next: (doc) => {
            // 成功：重新加载版本列表，自动选中最新版本
            this.versionsUpdated.emit(doc.versions)
            this.versionSelected.emit(Math.max(...doc.versions.map(v => v.id)))
        },
        error: (error) => { this.versionUploadState = UploadState.Failed }
      })
}
```

**刷新效果**：上传新版本后，不需要用户手动刷新，版本列表自动更新并选中最新版本。

---

#### 订阅组件 ⑦：SystemStatusDialogComponent（系统状态对话框）

文件：[system-status-dialog.component.ts](file:///d:/fz/0601/solo-dogfeeding/code/111-paperless-ngx/src-ui/src/app/components/common/system-status-dialog/system-status-dialog.component.ts#L79-L89)

```typescript
ngOnInit() {
    this.status.websocket_connected = this.websocketStatusService.isConnected()
      ? SystemStatusItemStatus.OK : SystemStatusItemStatus.ERROR
    this.websocketStatusService
      .onConnectionStatus()
      .pipe(takeUntil(this.unsubscribeNotifier))
      .subscribe((connected) => {
        this.status.websocket_connected = connected
          ? SystemStatusItemStatus.OK : SystemStatusItemStatus.ERROR
      })
}
```

**刷新效果**：系统状态弹窗中实时显示 WebSocket 连接状态（OK / ERROR），连接断开自动变红，恢复自动变绿。

---

#### 前端组件订阅总览表

| 组件 | 所在页面 | 订阅事件 | UI 刷新行为 |
|-----|---------|---------|------------|
| **AppComponent** | 全局根组件 | `onDocumentConsumptionFinished` / `Failed` / `Detected` | 显示 Toast 通知、刷新 TasksService |
| **DocumentListComponent** | `/documents`、`/view/:id` | `onDocumentConsumptionFinished`、`onDocumentDeleted` | 整页列表数据 reload |
| **DocumentDetailComponent** | `/documents/:id` | `onDocumentUpdated` | 重新拉取当前文档；编辑中则弹冲突确认框 |
| **SavedViewWidgetComponent** | Dashboard | `onDocumentConsumptionFinished` | 小组件文档列表 + 计数 reload |
| **StatisticsWidgetComponent** | Dashboard | `onDocumentConsumptionFinished` | 统计指标（总数/Inbox/类型分布）reload |
| **UploadFileWidgetComponent** | Dashboard + Sidebar | 直接读取 `getConsumerStatus*()` getters | 进度条、上传/失败/成功列表实时渲染 |
| **DocumentVersionDropdownComponent** | 文档详情页内 | `onDocumentConsumptionFinished` / `Failed`（按 taskId 过滤） | 上传新版本后自动刷新版本列表 |
| **SystemStatusDialogComponent** | 系统状态弹窗 | `onConnectionStatus` | WebSocket 连接状态图标实时变绿/变红 |

---

## 五、完整数据流转图

```
┌───────────────────────────────────────────────────────────────────────────────┐
│                        事件产生层（Celery Worker / Django API / Beat）            │
│                                                                               │
│  ┌─────────────────────────┐  ┌──────────────────────────────────────────────┐ │
│  │  status_update 事件源    │  │      document_updated 事件源（7 个入口）       │ │
│  │                         │  │                                              │ │
│  │ ① consume_file(Task)    │  │ ① DocumentViewSet.update()                  │ │
│  │   └→ ConsumerPlugin     │  │ ② DocumentViewSet.delete_version()          │ │
│  │      └→ _send_progress  │  │ ③ DocumentViewSet.update_version_label()    │ │
│  │                         │  │ ④ BulkEditView → bulk_update_documents()    │ │
│  │ ② upload-documents.ts   │  │    (set_correspondent/set_tags/...)         │ │
│  │   (前端本地 UPLOADING)  │  │ ⑤ TagViewSet → update_document_parent_tags()│ │
│  │                         │  │ ⑥ check_scheduled_workflows() 【直推】       │ │
│  │ ③ update_version API    │  │ ⑦ DocumentViewSet.update_version()          │ │
│  │   └→ consume_file       │  │    └→ consume_file（消费完成后另走 status）  │ │
│  └───────────┬─────────────┘  └──────────────────┬───────────────────────────┘ │
│              │ documents_deleted 事件源            │                             │
│              │  ① DeleteDocumentsView             │                             │
│              │     └→ bulk_edit.delete()          │                             │
│              │  ② Tag/对应人删除后级联刷新          │                             │
│              └───────────────┬────────────────────┘                             │
│                              │                                                  │
│                              ▼                                                  │
│  ┌─────────────────────────────────────────────────────────────────────────┐   │
│  │         ProgressManager / DocumentsStatusManager (上下文管理器)          │   │
│  │  async_to_sync(channel_layer.group_send("status_updates", {type, data}))│   │
│  └────────────────────────────┬────────────────────────────────────────────┘   │
└─────────────────────────────┼──────────────────────────────────────────────────┘
                              │
                              ▼
┌───────────────────────────────────────────────────────────────────────────────┐
│                       消息中间件（Redis Pub/Sub）                                 │
│                    CHANNEL_LAYERS → "status_updates" 组                          │
└─────────────────────────────┬───────────────────────────────────────────────────┘
                              │
                              ▼
┌───────────────────────────────────────────────────────────────────────────────┐
│                    连接维护层（Daphne / ASGI Worker）                             │
│                                                                               │
│  StatusConsumer (AsyncWebsocketConsumer) 每个浏览器连接一个实例                   │
│  ┌──────────────────────────────────────────────────────────────────────────┐  │
│  │ connect() → _authenticated()? → group_add("status_updates")            │  │
│  │                                                                        │  │
│  │ status_update(event)  → _can_view(data)? → send(json.dumps(event))     │  │
│  │ document_updated(event)→ _can_view(data)? → send(json.dumps(event))     │  │
│  │ documents_deleted(event)→              → send(json.dumps(event))        │  │
│  │                                                                        │  │
│  │ disconnect() → group_discard("status_updates")                         │  │
│  └────────────────────────────┬───────────────────────────────────────────┘  │
└─────────────────────────────┼───────────────────────────────────────────────────┘
                              │  WebSocket (ws://host/ws/status/)
                              ▼
┌───────────────────────────────────────────────────────────────────────────────┐
│                         前端层（Angular Browser）                                 │
│                                                                               │
│  WebsocketStatusService (providedIn: 'root', 由 AppComponent.ngOnInit connect)│
│  ┌──────────────────────────────────────────────────────────────────────────┐  │
│  │ onmessage → JSON.parse → switch(type)                                   │  │
│  │   ├─ status_update    → handleProgressUpdate → canViewMessage?          │  │
│  │   │                    → consumerStatus[]                                │  │
│  │   │                    → documentDetectedSubject                        │  │
│  │   │                    → documentConsumptionFinishedSubject             │  │
│  │   │                    → documentConsumptionFailedSubject               │  │
│  │   │                                                                    │  │
│  │   ├─ document_updated → handleDocumentUpdated → canViewMessage?         │  │
│  │   │                    → documentUpdatedSubject                         │  │
│  │   │                                                                    │  │
│  │   └─ documents_deleted → documentDeletedSubject                         │  │
│  └────────────────────────────┬───────────────────────────────────────────┘  │
│                              │                                                │
│                              ▼                                                │
│  ┌──────────────────────────────────────────────────────────────────────────┐  │
│  │                         8 个组件订阅 RxJS Subject                          │  │
│  │                                                                          │  │
│  │  AppComponent         → Toast 通知 + TasksService.reload                 │  │
│  │  DocumentList         → list.reload()（消费完成 / 删除）                  │  │
│  │  DocumentDetail       → loadDocument(id) + 冲突处理对话框                  │  │
│  │  SavedViewWidget      → reload()（消费完成）                              │  │
│  │  StatisticsWidget     → reload()（消费完成）                              │  │
│  │  UploadFileWidget     → 进度条/状态列表（变更检测自动刷新）                 │  │
│  │  VersionDropdown      → 按 taskId 过滤 → 版本列表自动刷新                 │  │
│  │  SystemStatusDialog   → WebSocket 连接图标绿/红切换                       │  │
│  └──────────────────────────────────────────────────────────────────────────┘  │
└───────────────────────────────────────────────────────────────────────────────┘
```

---

## 六、关键设计要点

### 6.1 前后端双权限校验
- **后端**：StatusConsumer 在发送每条消息前检查权限，避免越权信息通过 WebSocket 泄漏
- **前端**：WebsocketStatusService 收到消息后再次检查，作为防御性兜底
- 权限判定逻辑完全一致，防止逻辑漂移

### 6.2 统一 Group + Consumer 端过滤
- 所有客户端共享一个 `status_updates` 组，避免为每个用户创建独立 channel 造成 Redis 资源消耗
- 过滤放在 Consumer 侧执行（异步 `aexists()` 查询用户组），权衡了实现复杂度与隔离性

### 6.3 上下文管理器确保消息可靠发送
- `ProgressManager` 和 `DocumentsStatusManager` 均使用 `__enter__/__exit__` 管理 channel layer 生命周期
- 退出时 flush channel layer，避免在 Celery worker 进程中消息卡在发送缓冲区

### 6.4 异步 Consumer + 同步发送桥接
- Consumer 继承 `AsyncWebsocketConsumer`，所有 I/O 均为 async（channel layer 操作、DB 查询）
- 事件产生方（Celery、API）在同步上下文中运行，通过 `asgiref.sync.async_to_sync` 桥接调用

### 6.5 `QuerySet.update()` 不触发信号的补偿机制
- Django ORM 批量更新 (`qs.update()`) 不会触发 `post_save`，也不走 serializer 的 `save()`
- Paperless-ngx 专门设计了 `bulk_update_documents` Celery 任务，在批量改 DB 后显式发送 `document_updated` 信号，保证 WebSocket 推送不丢失

### 6.6 详情页的乐观并发控制
- DocumentDetailComponent 收到 `document_updated` 时，对比 `modified` 时间戳判断是否为"自己保存产生的回声"
- 如果用户正在编辑 (form dirty)，弹确认对话框让用户决定是否覆盖本地修改，避免丢失编辑内容

### 6.7 定时工作流的特殊处理
- 定时工作流 `check_scheduled_workflows()` **不通过** `document_updated` 信号发送通知，而是直接调用 `send_websocket_document_updated()`
- 代码注释明确说明原因：SCHEDULED 类型的工作流执行后，内部不会自动触发 `document_updated` 信号，需兜底直接推送

### 6.8 与 SSE 的区别
系统仅在 AI Chat 功能中使用 SSE 风格的 `StreamingHttpResponse`（[views.py](file:///d:/fz/0601/solo-dogfeeding/code/111-paperless-ngx/src/documents/views.py#L2181-L2185)），用于流式输出 LLM 回答。该实现与实时通知系统完全独立，不经过 WebSocket / Channels 层。
