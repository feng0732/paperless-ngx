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

## 五、上传新版本场景下的端到端深度链路分析

这是实时通知系统最复杂、最典型的场景：用户在文档详情页的版本下拉里上传一个新版本文件，后端 Celery 消费完成后，前端**两个组件**会**同时**收到两条独立的 WebSocket 消息并各自刷新。本章节从后端 API 入口到前端两个组件的 UI 刷新，进行逐行代码级别的追踪。

### 5.1 后端 API 入口 → Celery 任务提交

文件：[views.py](file:///d:/fz/0601/solo-dogfeeding/code/111-paperless-ngx/src/documents/views.py#L1890-L1945)

```
HTTP POST /api/documents/{id}/update_version/   (multipart/form-data)
    │
    ├─ DocumentVersionSerializer 校验：document 文件、可选 version_label
    ├─ get_root_document(request_doc) → 获取根文档（防止传入的是版本 id）
    ├─ has_perms_owner_aware("change_document") → 权限校验
    │
    ├─ 将上传文件写入临时目录 SCRATCH_DIR，设置 mtime
    │
    ├─ 构造 ConsumableDocument(
    │      source=ApiUpload,
    │      original_file=临时文件路径,
    │      root_document_id=root_doc.pk,  ← 关键：标记这是版本更新，不是新文档
    │    )
    │
    ├─ 构造 DocumentMetadataOverrides（version_label + actor_id）
    │
    └─ consume_file.apply_async(
           kwargs={"input_doc": ..., "overrides": ...},
           headers={"trigger_source": WEB_UI},
       )
           │
           └─ 返回 Response(async_task.id)  ← 前端收到 taskId
```

**关键差异**：新文档上传 vs 新版本上传的插件链不同。
文件：[tasks.py](file:///d:/fz/0601/solo-dogfeeding/code/111-paperless-ngx/src/documents/tasks.py#L140-L155)

```python
plugins = (
    # 新版本（有 root_document_id）：只跑 2 个插件
    [ConsumerPreflightPlugin, ConsumerPlugin]
    if input_doc.root_document_id is not None
    # 新文档：跑 8 个插件（含条码拆分、工作流触发、ASN 检查等）
    else [ConsumerPreflightPlugin, AsnCheckPlugin, CollatePlugin, BarcodePlugin,
          AsnCheckPlugin, WorkflowTriggerPlugin, ConsumerPlugin]
)
```

### 5.2 ConsumerPlugin.run() —— 两条消息的精确产生位置

文件：[consumer.py](file:///d:/fz/0601/solo-dogfeeding/code/111-paperless-ngx/src/documents/consumer.py#L408-L784)

整个方法的执行顺序和消息产生点如下：

```
ConsumerPlugin.run():
│
├─ ① 创建 working_copy 临时文件副本
├─ ② MIME 类型检测 → 获取 Parser
├─ ③ document_consumption_started.send()  ← 业务信号，不涉及 WebSocket
├─ ④ run_pre_consume_script()
│
├─ ⑤ 解析文档 → _send_progress(20%, WORKING, PARSING_DOCUMENT)  ← status_update
├─ ⑥ 生成缩略图 → _send_progress(70%, WORKING, GENERATING_THUMBNAIL)  ← status_update
├─ ⑦ 提取日期 → _send_progress(90%, WORKING, PARSE_DATE)  ← status_update
├─ ⑧ 加载分类器 classifier
│
├─ ⑨ _send_progress(95%, WORKING, SAVE_DOCUMENT)  ← status_update
│
│  ═══════════════════════ 事务块开始 transaction.atomic() ═══════════════════════
│  │
│  ├─ ⑩ 分支 A：root_document_id 存在（新版本场景）
│  │     ├─ root_doc = Document.objects.get(pk=self.input_doc.root_document_id)
│  │     ├─ version_doc = _create_version_from_root(root_doc, ...)
│  │     │     ├─ select_for_update 锁行
│  │     │     ├─ 计算下一个 version_index = Max(version_index) + 1
│  │     │     └─ 构造新 Document(root_document=root_doc, version_index=N, ...)
│  │     ├─ version_doc.save()  ← 写入数据库，同时记录审计日志（Version Added）
│  │     └─ document = version_doc  ← 后续流程使用 version_doc
│  │
│  └─ ⑩ 分支 B：root_document_id 为空（新文档场景）
│        └─ document = self._store(...)  ← 创建全新文档
│
│  ├─ ⑪ document_consumption_finished.send(document=document, ...)
│  │     └─ 触发 add_inbox_tags / set_correspondent / run_workflows_consumption 等
│  │
│  ├─ ⑫ FileLock(MEDIA_LOCK) 内写入文件：
│  │     ├─ generate_unique_filename(document)
│  │     ├─ document.filename = ...  →  _write(源文件 → source_path)
│  │     ├─ _write(thumbnail → thumbnail_path)
│  │     └─ 如果有归档文件 → 写入 archive_path + 计算 archive_checksum
│  │
│  ├─ ⑬ document.save()  ← 第二次保存，写入 filename、archive_filename、archive_checksum
│  │     │
│  │     │  ═══════════════════════════════════════════════════════════════════
│  │     │  ★ 第 1 条消息产生位置：document_updated (根文档)
│  │     │  ═══════════════════════════════════════════════════════════════════
│  │     └─ ⑭ if document.root_document_id:
│  │            document_updated.send(
│  │                sender=self.__class__,
│  │                document=document.root_document,  ← 注意：发的是根文档，不是新版本
│  │            )
│  │               │
│  │               └─ apps.py 信号绑定:
│  │                  document_updated → send_websocket_document_updated()
│  │                     └─ DocumentsStatusManager.send_document_updated(
│  │                           document_id=根文档.id,
│  │                           modified=根文档.modified,
│  │                           owner_id=..., users_can_view=..., groups_can_view=...
│  │                        )
│  │                        └─ async_to_sync(channel_layer.group_send)(
│  │                              "status_updates",
│  │                              {"type": "document_updated", "data": {...}}
│  │                           )
│  │
│  └─ ⑮ 删除原始输入文件
│
│  ═══════════════════════ 事务块结束 ═══════════════════════
│
├─ ⑯ run_post_consume_script(document)  ← post consume 脚本
│
│  ═══════════════════════════════════════════════════════════════════
│  ★ 第 2 条消息产生位置：status_update (SUCCESS)
│  ═══════════════════════════════════════════════════════════════════
└─ ⑰ self._send_progress(
        100, 100,
        ProgressStatusOptions.SUCCESS,
        ConsumerStatusShortMessage.FINISHED,
        document.id,  ← 注意：这里的 document 是**新版本文档**的 id
    )
       │
       └─ self.status_mgr.send_progress(
              ProgressStatusOptions.SUCCESS, FINISHED, 100, 100,
              document_id=新版本文档.id,
              owner_id=..., users_can_view=..., groups_can_view=...
          )
          └─ async_to_sync(channel_layer.group_send)(
                "status_updates",
                {"type": "status_update", "data": {...}}
             )

└─ ⑱ document.refresh_from_db() → 返回 ConsumeFileSuccessResult(document_id=新版本id)
```

**两条消息的关键差异总结**：

| 对比项 | status_update(SUCCESS) | document_updated(根文档) |
|-------|------------------------|-------------------------|
| 产生位置 | ConsumerPlugin.run() 末尾，事务外 | ConsumerPlugin.run() 事务内，document.save() 之后 |
| `type` 字段 | `"status_update"` | `"document_updated"` |
| `data.document_id` | **新版本文档**的 id | **根文档**的 id |
| 触发路径 | `_send_progress()` → `ProgressManager.send_progress()` | `document_updated.send()` 信号 → `send_websocket_document_updated()` → `DocumentsStatusManager.send_document_updated()` |
| 额外字段 | `status`, `message`, `current_progress`, `max_progress`, `task_id`, `filename` | `modified` (ISO 时间戳) |
| 顺序 | 后发送 | 先发送 |

两条消息分别通过 Redis Pub/Sub 广播到 `status_updates` 组，各自走 Consumer → WebSocket → 前端 Service 的独立链路。

### 5.3 事务时序深度分析 —— 为什么 document_updated 可能导致详情页读到旧数据

本章节深入探讨 `document_updated` 在 `transaction.atomic` 内部发送、未使用 `transaction.on_commit` 时可能引发的跨系统时序问题，以及 `status_update(SUCCESS)` 作为事务外信号的时序可靠性。

#### 5.3.1 精确事务边界代码定位

文件：[consumer.py#L587](file:///d:/fz/0601/solo-dogfeeding/code/111-paperless-ngx/src/documents/consumer.py#L587)

```python
try:
    with transaction.atomic():          # ───── 事务起点 T0
        # ... 创建 version_doc、保存、文件写入 ...
        document.save()                 # ───── T1 (version_doc 第二次 save)

        if document.root_document_id:
            document_updated.send(      # ───── T2 (事务内发送信号！)
                sender=self.__class__,
                document=document.root_document,
            )
            # ┌──────────────────────────────────────────────────────────┐
            # │ 此处信号处理器内部立即调用 async_to_sync(channel_layer.  │
            # │ group_send) → Redis PUBLISH 命令**立刻发出**！          │
            # │ 但此时 DB 事务还未 COMMIT，外部连接看不到新数据。        │
            # └──────────────────────────────────────────────────────────┘

        self.input_doc.original_file.unlink()     # 删除临时文件
        self.working_copy.unlink()
        # ... 更多清理工作 ...
                                         # ───── T3 (with 块结束 → COMMIT)
except Exception as e:
    self._fail(...)

self.run_post_consume_script(document)  # ───── T4 (事务外)

self._send_progress(100, 100, SUCCESS,  # ───── T5 (status_update SUCCESS)
                    FINISHED, document.id)
```

关键时间点：
- **T2**：`document_updated.send()` → 信号处理函数同步执行 → Redis PUBLISH 立刻发出
- **T3**：`transaction.atomic()` 的 `with` 块退出 → DB 才真正 COMMIT
- **T5**：`status_update(SUCCESS)` → 第二次 Redis PUBLISH

T2 与 T3 之间可能相差若干毫秒（取决于文件删除 I/O 耗时），但 Redis 消息传播是微秒级的，因此 WebSocket 消息**极大概率在 DB COMMIT 之前到达浏览器**。

#### 5.3.2 document_updated 信号处理函数的内部可见性

文件：[handlers.py#L832-L851](file:///d:/fz/0601/solo-dogfeeding/code/111-paperless-ngx/src/documents/signals/handlers.py#L832-L851)

```python
def send_websocket_document_updated(sender, document: Document, **kwargs) -> None:
    # At this point, workflows may already have applied additional changes.
    document.refresh_from_db()   # ← 事务内的 refresh，能看到本事务未提交的修改

    doc_overrides = DocumentMetadataOverrides.from_document(document)

    with DocumentsStatusManager() as status_mgr:
        status_mgr.send_document_updated(
            document_id=document.id,
            modified=DRF_DATETIME_FIELD.to_representation(document.modified),
            owner_id=doc_overrides.owner_id,
            users_can_view=doc_overrides.view_users,
            groups_can_view=doc_overrides.view_groups,
        )
        # __exit__ → async_to_sync(self._channel.flush) → Redis 立即发出
```

> **重要细节**：`refresh_from_db()` 在**发送者的事务内部**执行，因此能看到本事务已写入但未提交的数据（如 version_doc 的保存），`document.modified` 能拿到最新值。但这仅限于发送方所在的 DB 连接。

#### 5.3.3 Redis Pub/Sub 与 DB 事务的跨系统割裂

`BaseStatusManager.send()` 的实现：
文件：[helpers.py#L107-L116](file:///d:/fz/0601/solo-dogfeeding/code/111-paperless-ngx/src/documents/plugins/helpers.py#L107-L116)

```python
def send(self, payload: WebsocketPayload) -> None:
    self.open()
    async_to_sync(self._channel.group_send)("status_updates", payload)
    # ↑ 此处同步等待 Redis PUBLISH 命令返回 ACK，即立即发出
```

问题根源：**Redis Pub/Sub 是独立于 DB 事务的外部系统**。Django 的 `transaction.atomic()` 只能控制 PostgreSQL/MySQL 的 COMMIT/ROLLBACK，**无法**回滚或延迟已经发送到 Redis 的消息。

这就产生了经典的**双写不一致窗口**：

```
Celery Worker (同一 DB 连接 #A)           Redis             Daphne/ASGI (DB 连接 #B)      浏览器
        │                                    │                      │                         │
        │  transaction.atomic() 开始         │                      │                         │
        │  version_doc.save()                │                      │                         │
        │                                    │                      │                         │
        │  document_updated.send()           │                      │                         │
        │  └→ refresh_from_db()  (连接 #A)   │                      │                         │
        │  └→ group_send() ────────────────→│  PUBLISH             │                         │
        │     (Redis 立即发出)               │─────────────────────→│  WebSocket.onmessage    │
        │                                    │                      │  document_updated       │
        │  删除临时文件(几毫秒~几十毫秒)       │                      │                         │
        │                                    │                      │  loadDocument() ───────→│
        │                                    │                      │                         │ HTTP GET /api/documents/{id}/
        │                                    │                      │  SELECT ... (连接 #B)   │
        │                                    │                      │  ❌ 此时事务未 COMMIT， │
        │                                    │                      │     version_doc 不可见， │
        │                                    │                      │     versions 数组缺失！ │
        │                                    │                      │                         │
        │  with 块退出 → DB COMMIT ─────────────────────────────────────────────────────────→│ 此时才可见
        │                                    │                      │                         │
        │  run_post_consume_script()         │                      │                         │
        │  _send_progress(SUCCESS) ─────────→│─────────────────────→│────────────────────────→│ 此时 load 才能读到新版本
```

在 T2 到 T3 的窗口内（可能是几毫秒到几十毫秒，取决于临时文件删除 I/O）：
- 前端通过新 HTTP 请求（独立 DB 连接 #B）执行 `loadDocument()`
- PostgreSQL 的 MVCC 隔离级别（默认 READ COMMITTED）保证连接 #B 看不到连接 #A 未提交的写入
- **结果**：详情页 reload 后，versions 数组中**没有新版本**，PDF 预览仍指向旧版本文件（如果新文件还没写入磁盘甚至可能 404）

但由于两条消息（document_updated 和 status_update）都会触发刷新，通常：
1. `document_updated` 先到 → 第一次 `loadDocument()` 可能拿到旧数据
2. 几十毫秒后 `status_update(SUCCESS)` 到达 → DocumentVersionDropdownComponent 再次 `getVersions()` → 此时事务已 COMMIT，拿到新版本
3. 用户感知上就是版本列表出现了延迟（但 UI 会刷新两次，最终状态正确）

在高 I/O 负载下（临时文件很大、磁盘慢），这个窗口可能拉大到数百毫秒甚至秒级，此时用户可能短暂看到"上传完成但列表中没有新版本"的异常状态。

#### 5.3.4 为什么 status_update(SUCCESS) 是更可靠的完成信号

`status_update(SUCCESS)` 的发送位置：
文件：[consumer.py#L769-L779](file:///d:/fz/0601/solo-dogfeeding/code/111-paperless-ngx/src/documents/consumer.py#L769-L779)

```python
# ← transaction.atomic() 已在此前退出（T3 已 COMMIT）

self.run_post_consume_script(document)   # T4: 外部脚本可能产生额外副作用

self._send_progress(                     # T5: 发 SUCCESS
    100, 100, ProgressStatusOptions.SUCCESS,
    ConsumerStatusShortMessage.FINISHED, document.id,
)
```

时序优势：
1. **DB 已提交**：T3 时事务已经 COMMIT，所有 DB 连接都能看到 version_doc 和完整的 versions 数组
2. **文件已落盘**：事务内 `_write()` 已经把 source、thumbnail、archive 三个文件写入磁盘（`fsync` 取决于 OS，但文件路径已存在）
3. **Post Consume 脚本已执行**：`run_post_consume_script()` 完成所有外部系统集成
4. **cleanup 已完成**：临时文件已删除，不会占用磁盘空间

也就是说，`status_update(SUCCESS)` 代表了**端到端的全部完成**，而 `document_updated` 只代表"DB 写入请求已发出"。这也是为什么 DocumentVersionDropdownComponent 选择订阅 `onDocumentConsumptionFinished()`（由 status_update SUCCESS 驱动），而不是 `onDocumentUpdated()`。

#### 5.3.5 如果使用 transaction.on_commit 的正确写法

如果要消除 T2~T3 的竞态窗口，应将 Redis 发送延迟到事务 COMMIT 之后：

```python
# 当前写法（有竞态窗口）：
with transaction.atomic():
    document.save()
    if document.root_document_id:
        document_updated.send(sender=self.__class__, document=document.root_document)
        # ↑ 信号处理函数立即 group_send → Redis 立即发出

# 正确写法（延迟到 COMMIT 之后）：
from django.db import transaction

with transaction.atomic():
    document.save()
    if document.root_document_id:
        root_doc = document.root_document
        def _send_ws_update(root_doc=root_doc):
            # 闭包捕获当前 root_doc 引用
            send_websocket_document_updated(sender=None, document=root_doc)
        transaction.on_commit(_send_ws_update)
        # ↑ 注册回调，等 COMMIT 成功后才真正执行 Redis 发送
```

`transaction.on_commit()` 的保证：
- 只有当外层事务成功 COMMIT 后才执行回调
- 如果事务 ROLLBACK（异常抛出），回调不会被执行
- 多个 `on_commit` 回调按注册顺序执行

这样可以确保：Redis 消息发出时，DB 数据一定对所有连接可见，前端 `loadDocument()` 一定能读到新版本。

> **注意**：`document_updated` 信号被多个处理器消费（`run_workflows_updated`、`send_websocket_document_updated`），其中 `run_workflows_updated` 必须在事务内执行（因为工作流修改要和文档修改在同一事务中原子提交）。因此不能简单地把整个信号改为 `on_commit`，只能对 WebSocket 发送这一个处理器单独延迟。

#### 5.3.6 根文档 modified 字段为什么可能不变 —— 基于代码事实的精确核准

**第一步：modified 的更新机制**
文件：[models.py#L250-L255](file:///d:/fz/0601/solo-dogfeeding/code/111-paperless-ngx/src/documents/models.py#L250-L255)

```python
modified = models.DateTimeField(
    _("modified"),
    auto_now=True,      # ← 只有实例 .save() 时才自动更新
    editable=False,
    db_index=True,
)
```

`auto_now=True` 的语义：仅当对该模型实例调用 `.save()`（且不指定 `update_fields` 排除 modified，或显式包含 modified）时才被更新。**通过外键关联新增子记录、反向查询、ManyToMany 变更都不会触发父实例的 modified 更新。**

**第二步：上传新版本流程中 root_doc 的所有使用点**

逐个核准 consumer.py 中 root_doc 是否被 `.save()`：

| 位置 | 操作 | 是否触发 root_doc.save() |
|-----|------|-------------------------|
| [consumer.py#L593-L595](file:///d:/fz/0601/solo-dogfeeding/code/111-paperless-ngx/src/documents/consumer.py#L593-L595) | `root_doc = Document.objects.get(pk=self.input_doc.root_document_id)` | ❌ 仅查询 |
| [consumer.py#L265](file:///d:/fz/0601/solo-dogfeeding/code/111-paperless-ngx/src/documents/consumer.py#L265) | `root_doc_frozen = Document.objects.select_for_update().get(pk=root_doc.pk)` | ❌ 仅加行锁 + 查询 |
| [consumer.py#L279-L295](file:///d:/fz/0601/solo-dogfeeding/code/111-paperless-ngx/src/documents/consumer.py#L279-L295) | `version_doc = Document(root_document=root_doc_frozen, ...)` | ❌ 只建立 FK 关联，version_doc 是新实例 |
| [consumer.py#L618-L622](file:///d:/fz/0601/solo-dogfeeding/code/111-paperless-ngx/src/documents/consumer.py#L618-L622) | `original_document.save()` | ❌ 保存的是 version_doc（original_document 是版本实例） |
| [consumer.py#L630-L636](file:///d:/fz/0601/solo-dogfeeding/code/111-paperless-ngx/src/documents/consumer.py#L630-L636) | `LogEntry.objects.log_create(instance=root_doc, ...)` | ❌ auditlog 写 LogEntry 表，不改 root_doc 本身 |
| [consumer.py#L729](file:///d:/fz/0601/solo-dogfeeding/code/111-paperless-ngx/src/documents/consumer.py#L729) | `document.save()` | ❌ document 此时是 version_doc（`document = original_document` 在 L665 被赋值），保存的是版本 |
| [consumer.py#L732-L735](file:///d:/fz/0601/solo-dogfeeding/code/111-paperless-ngx/src/documents/consumer.py#L732-L735) | `document_updated.send(sender, document=document.root_document)` | ⚠️ 发送信号，是否 save 取决于信号处理器 |

**结论（无工作流时）**：上传新版本流程本身 **从不直接调用 `root_doc.save()`**，因此 `root_doc.modified` 保持为上传前的旧值。

**第三步：唯一例外 —— DOCUMENT_UPDATED 工作流触发时会显式 save root_doc**

信号处理器连接顺序（apps.py）：
```python
document_updated.connect(run_workflows_updated)           # 先执行
document_updated.connect(send_websocket_document_updated) # 后执行
```

`run_workflows_updated` → `run_workflows(DOCUMENT_UPDATED, document=root_doc)` 的执行路径：
文件：[handlers.py#L875-L880](file:///d:/fz/0601/solo-dogfeeding/code/111-paperless-ngx/src/documents/signals/handlers.py#L875-L880)

```python
if isinstance(document, Document) and document.root_document_id is not None:
    # 传入的是 version_doc → 跳过
    return None
# 传入的是 root_doc（root_document_id 为 None）→ 正常执行工作流
```

当存在匹配的 DOCUMENT_UPDATED 工作流且触发了 ASSIGNMENT/REMOVAL 等修改操作时：
文件：[handlers.py#L963-L984](file:///d:/fz/0601/solo-dogfeeding/code/111-paperless-ngx/src/documents/signals/handlers.py#L963-L984)

```python
if not use_overrides:   # DOCUMENT_UPDATED 触发时 overrides=None → use_overrides=False
    document.title = document.title[:128]
    document.save(
        update_fields=[
            "title", "correspondent", "document_type", "storage_path",
            "owner", "modified",   # ← 显式包含 modified！
        ],
    )
```

> L973-974 的代码注释也明确指出：*modified has auto_now=True but is not auto-added when update_fields is specified, so it must be listed explicitly.*

**结论（有工作流时）**：如果用户配置了 DOCUMENT_UPDATED 类型的工作流且文档匹配，工作流执行时会**显式 save root_doc** 并更新 `modified`。随后 `send_websocket_document_updated()` 中的 `document.refresh_from_db()` 就能拿到更新后的时间戳。

**第四步：send_websocket_document_updated 中 refresh_from_db 的作用**
文件：[handlers.py#L837-L838](file:///d:/fz/0601/solo-dogfeeding/code/111-paperless-ngx/src/documents/signals/handlers.py#L837-L838)

```python
# At this point, workflows may already have applied additional changes.
document.refresh_from_db()
```

代码注释明确说明：工作流可能已对文档施加了额外修改，所以重新从 DB 拉取。但 `refresh_from_db()` 只能看到**同一 DB 连接**上已提交的写入。由于整个调用链在同一事务内，工作流的 save() 对这个连接是可见的；但对其他连接（如前端 HTTP 请求使用的连接），在事务 COMMIT 之前均不可见。

**对前端判定的实际影响**：
- `handleIncomingDocumentUpdated()` 的回声判定对比的是 `lastLocalSaveModified`（仅在用户本地 PATCH 保存成功时设置），正常查看时为 `null`，所以旧时间戳不会被误判为回声而忽略
- 仅当用户恰好在上传新版本的同时，自己也在 PATCH 修改文档并保存时，旧 modified 才可能与本地保存的时间戳碰巧相等而被错误忽略（概率极低）

---

#### 5.3.7 详情页 HTTP GET 序列化 versions 的完整查询路径

**前端 API 调用链**：

| 调用方 | 方法 | URL | 参数 |
|-------|------|-----|------|
| DocumentDetailComponent.loadDocument() | `documentsService.get(documentId)` | `GET /api/documents/{id}/` | `full_perms=true`（全量字段，含 versions） |
| DocumentVersionDropdownComponent（收到 SUCCESS 后） | `documentsService.getVersions(documentId)` | `GET /api/documents/{id}/` | `fields=id,versions`（仅稀疏字段） |

文件：[document.service.ts#L196-L213](file:///d:/fz/0601/solo-dogfeeding/code/111-paperless-ngx/src-ui/src/app/services/rest/document.service.ts#L196-L213)
```typescript
get(id: number, versionID: number = null, fields: string = null): Observable<Document> {
    const params = { full_perms: true }
    if (versionID) params.version = versionID.toString()
    if (fields) params.fields = fields
    return this.http.get<Document>(this.getResourceUrl(id), { params })
}

getVersions(documentId: number): Observable<Document> {
    return this.http.get<Document>(this.getResourceUrl(documentId), {
        params: { fields: 'id,versions' },
    })
}
```

**后端 ViewSet 路径**：

`GET /api/documents/{id}/` → `DocumentViewSet.retrieve()`：
文件：[views.py#L1116-L1140](file:///d:/fz/0601/solo-dogfeeding/code/111-paperless-ngx/src/documents/views.py#L1116-L1140)

```python
def retrieve(self, request, *args, **kwargs):
    response = super().retrieve(request, *args, **kwargs)
    # ↓ RetrieveModelMixin.retrieve() 的默认实现：
    #   instance = self.get_object()  ← get_queryset().get(pk=pk)
    #   serializer = self.get_serializer(instance)  ← DocumentSerializer(instance)
    #   return Response(serializer.data)
    if "version" not in request.query_params or ...:
        return response
    # version 参数时才走额外逻辑（切换预览版本的 content）
```

DocumentViewSet 没有自定义的 `get_queryset()` 覆盖（使用 `Document.objects.all()`），也没有 `prefetch_related('versions')`。因此 versions 字段的查询由 serializer 在序列化时**按需触发**。

**Serializer 中 versions 字段的精确实现**：
文件：[serialisers.py#L1009](file:///d:/fz/0601/solo-dogfeeding/code/111-paperless-ngx/src/documents/serialisers.py#L1009)
```python
versions = SerializerMethodField()  # ← 调用 get_versions()
```

文件：[serialisers.py#L1043-L1079](file:///d:/fz/0601/solo-dogfeeding/code/111-paperless-ngx/src/documents/serialisers.py#L1043-L1079)

```python
@extend_schema_field(DocumentVersionInfoSerializer(many=True))
def get_versions(self, obj):
    # Step 1: 确定根文档
    root_doc = obj if obj.root_document_id is None else obj.root_document
    if root_doc is None:
        return []

    # Step 2: 检查是否已有 prefetch 缓存（DocumentViewSet 没有 prefetch，所以走 else）
    prefetched_cache = getattr(obj, "_prefetched_objects_cache", None)
    prefetched_versions = (
        prefetched_cache.get("versions")
        if isinstance(prefetched_cache, dict) else None
    )

    versions: list[Document]
    if prefetched_versions is not None:
        versions = [*prefetched_versions, root_doc]
    else:
        # Step 3: ★ 执行独立 SQL 查询 —— 这是决定新版本是否可见的关键查询
        versions_qs = Document.objects.filter(root_document=root_doc).only(
            "id", "added", "checksum", "version_label",
        )
        versions = [*versions_qs, root_doc]
        # ↑ 此时 Django ORM 发出：SELECT id, added, checksum, version_label
        #                    FROM documents_document
        #                   WHERE root_document_id = <root_doc.pk>

    # Step 4: 组装 + 按 id 倒序（最新版本在前）
    def build_info(doc: Document) -> _DocumentVersionInfo:
        return {
            "id": doc.id,
            "added": doc.added,
            "version_label": doc.version_label,
            "checksum": doc.checksum,
            "is_root": doc.id == root_doc.id,
        }
    info = [build_info(doc) for doc in versions]
    info.sort(key=lambda item: item["id"], reverse=True)
    return info
```

**核心要点**：versions 列表来自于 `Document.objects.filter(root_document=root_doc)` 的**独立 SQL 查询**，不依赖于 `root_doc` 实例的任何缓存。因此：
- 查询结果完全取决于**执行此 SELECT 时 DB 中已 COMMIT 的数据**
- 不经过 root_doc 的 `_prefetched_objects_cache`（ViewSet 没有 prefetch）
- `*versions_qs` 触发 QuerySet 的实际求值 → 发 SQL

---

#### 5.3.8 事务 COMMIT 前后 versions 查询结果的可见性差异

结合 PostgreSQL 的 MVCC（Multiversion Concurrency Control）隔离级别（默认 `READ COMMITTED`）：

```
时间轴          Celery Worker (连接 #A)             PostgreSQL               Daphne/API (连接 #B)
  │
  T0  transaction.atomic() 开始
  │     BEGIN
  │
  T1  version_doc = Document(root_document=root_doc, ...)
  │     INSERT INTO documents_document (...) VALUES (...)
  │     ← 在连接 #A 的事务快照中新增一行，其他连接不可见
  │
  T2  version_doc.save()
  │     ← 仍在连接 #A 的私有事务空间
  │
  T3  document_updated.send() → Redis PUBLISH ──微秒级──→ 到达连接 #B
  │                                                       WebSocket.onmessage → document_updated
  │                                                                         ↓
  │                                                       loadDocument() → HTTP GET /api/documents/{id}/
  │                                                                         ↓
  │                                                       DocumentSerializer.get_versions():
  │                                                         SELECT ... FROM documents_document
  │                                                         WHERE root_document_id = <pk>
  │                                                         ← 连接 #B 在 READ COMMITTED 下
  │                                                            只能看到 T0 之前 COMMIT 的行
  │                                                            ❌ 新 version_doc 不可见！
  │                                                            versions 数组少一条
  │
  T4  删除临时文件 (I/O 等待 几ms~几十ms)
  │
  T5  transaction.atomic() 退出 → COMMIT
  │     ← 新 version_doc 行对所有连接可见
  │
  T6  run_post_consume_script()
  │
  T7  _send_progress(SUCCESS) → Redis PUBLISH ────────→ 到达连接 #B
                                                          WebSocket.onmessage → status_update
                                                                         ↓
                                                          getVersions() → HTTP GET /api/documents/{id}/?fields=id,versions
                                                                         ↓
                                                          DocumentSerializer.get_versions():
                                                            SELECT ... WHERE root_document_id = <pk>
                                                            ← T5 已 COMMIT
                                                            ✅ 新 version_doc 可见！
                                                            versions 数组完整
```

**实证总结**：
| 消息类型 | DB 状态 | `filter(root_document=root_doc)` 查询结果 | 前端 versions 数组 |
|---------|---------|------------------------------------------|-------------------|
| `document_updated`（事务内发送，通常先到达） | T3，未 COMMIT | 不包含新 version_doc 行 | ❌ 缺失最新版本 |
| `status_update(SUCCESS)`（事务外发送，通常后到达） | T7，已 COMMIT + post consume 完成 | 包含新 version_doc 行 | ✅ 完整，含最新版本 |

这也解释了为什么**两条消息共同存在时最终状态总是正确**：即使第一次 `loadDocument()` 由于事务未 COMMIT 拿到旧 versions，几十毫秒后 `status_update(SUCCESS)` 驱动的第二次 `getVersions()` 一定能拿到完整版本列表。高 I/O 负载下用户可能短暂看到版本列表不完整，但最终会被第二次刷新修正。

### 5.4 后端 WebSocket 分发 —— Consumer 侧

文件：[consumers.py](file:///d:/fz/0601/solo-dogfeeding/code/111-paperless-ngx/src/paperless/consumers.py)

两条消息到达 Consumer 后的处理路径：

```
Redis Pub/Sub → StatusConsumer (每个浏览器连接一个实例)
    │
    ├─ 收到 {"type": "document_updated", "data": {document_id: 根id, modified, owner_id, ...}}
    │     └─ document_updated(event) 方法被 Channels 自动调度
    │        ├─ _authenticated()? → 否则 close()
    │        └─ _can_view(event["data"])?
    │           └─ 是 → send(json.dumps(event))  ← 通过 WebSocket 发给浏览器
    │
    └─ 收到 {"type": "status_update", "data": {task_id, filename, status: SUCCESS, current_progress: 100, document_id: 新版本id, ...}}
          └─ status_update(event) 方法被 Channels 自动调度
             ├─ _authenticated()? → 否则 close()
             └─ _can_view(event["data"])?
                └─ 是 → send(json.dumps(event))  ← 通过 WebSocket 发给浏览器
```

> 注意：两条消息的 `_can_view` 权限检查各自独立，`status_update` 使用新版本的 owner/权限信息，`document_updated` 使用根文档的 owner/权限信息。通常两者一致，但如果版本创建时权限被覆盖可能出现差异。

### 5.5 前端 WebsocketStatusService 分发

文件：[websocket-status.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/111-paperless-ngx/src-ui/src/app/services/websocket-status.service.ts)

```
浏览器 WebSocket.onmessage → handleMessage(ev)
    │
    ├─ JSON.parse(ev.data) → { type, data }
    │
    ├─ switch(type)
    │   │
    │   ├─ case DOCUMENT_UPDATED:
    │   │    └─ handleDocumentUpdated(messageData)
    │   │       ├─ canViewMessage(messageData)? → 否: return
    │   │       └─ documentUpdatedSubject.next(messageData)
    │   │           └─ messageData = { document_id: 根id, modified: "2026-06-08T10:30:00Z", owner_id, users_can_view, groups_can_view }
    │   │
    │   └─ case STATUS_UPDATE:
    │        └─ handleProgressUpdate(messageData)
    │           ├─ canViewMessage(messageData)? → 否: return
    │           ├─ this.get(task_id, filename) → 创建/查找 FileStatus 对象
    │           ├─ status.updateProgress(WORKING, 100, 100)
    │           ├─ status.message = FILE_STATUS_MESSAGES[FINISHED]
    │           ├─ status.documentId = messageData.document_id  ← 新版本id
    │           │
    │           └─ messageData.status === SUCCESS →
    │              ├─ status.phase = FileStatusPhase.SUCCESS
    │              └─ documentConsumptionFinishedSubject.next(status)
    │                  └─ status = FileStatus{ taskId, filename, phase: SUCCESS, documentId: 新版本id, ... }
    │
    └─ 两条消息通过不同 Subject 广播出去，互不干扰
```

### 5.6 前端组件订阅与界面刷新

两条消息分别被不同的组件订阅，以下是两条链路的前端处理细节。

---

#### 链路 A：status_update(SUCCESS) → DocumentVersionDropdownComponent

文件：[document-version-dropdown.component.ts](file:///d:/fz/0601/solo-dogfeeding/code/111-paperless-ngx/src-ui/src/app/components/document-detail/document-version-dropdown/document-version-dropdown.component.ts#L195-L275)

这个组件**不直接订阅** `onDocumentConsumptionFinished()`，而是在用户点击上传时通过 RxJS `switchMap + merge` 在**局部作用域内**临时订阅对应 taskId 的完成事件：

```
用户选择文件 → onVersionFileSelected(event):
    │
    ├─ this.documentsService.uploadVersion(uploadDocumentId, file, label)
    │   └─ HTTP POST → 返回 taskId (字符串)
    │
    ├─ tap(() => {
    │     this.versionUploadState = UploadState.Processing  ← UI 显示"处理中"
    │     this.toastService.showInfo("Uploading new version...")
    │   })
    │
    ├─ switchMap(taskId → merge(
    │     // 只取第一个匹配 taskId 的 SUCCESS 或 FAILED 事件
    │     websocketStatusService.onDocumentConsumptionFinished()
    │       .pipe(filter(s => s.taskId === taskId),
    │             map(() => ({ state: 'success' }))),
    │     websocketStatusService.onDocumentConsumptionFailed()
    │       .pipe(filter(s => s.taskId === taskId),
    │             map(s => ({ state: 'failed', message: s.message }))),
    │   ).pipe(take(1)))
    │
    ├─ switchMap(result →
    │     result.state === 'success'
    │       ? this.documentsService.getVersions(uploadDocumentId)  ← HTTP GET 拉最新版本列表
    │       : of(null))
    │
    └─ subscribe({
         next: (doc) => {
             if (doc?.versions) {
                 this.versionsUpdated.emit(doc.versions)
                 // 自动选中最新版本（最大 id）
                 this.versionSelected.emit(Math.max(...doc.versions.map(v => v.id)))
                 this.clearVersionUploadStatus()  // UI 恢复 Idle
             }
         },
         error: (e) => {
             this.versionUploadState = UploadState.Failed
             this.versionUploadError = e.message
             this.toastService.showError(...)
         }
       })
```

**父子组件事件通信**（模板绑定）：
文件：[document-detail.component.html](file:///d:/fz/0601/solo-dogfeeding/code/111-paperless-ngx/src-ui/src/app/components/document-detail/document-detail.component.html#L27-L34)

```html
<pngx-document-version-dropdown
  [documentId]="documentId"
  [versions]="document?.versions ?? []"
  [selectedVersionId]="selectedVersionId"
  [userIsOwner]="userIsOwner"
  [userCanEdit]="userCanEdit"
  (versionSelected)="onVersionSelected($event)"
  (versionsUpdated)="onVersionsUpdated($event)"
/>
```

父组件 DocumentDetailComponent 对事件的响应：

文件：[document-detail.component.ts](file:///d:/fz/0601/solo-dogfeeding/code/111-paperless-ngx/src-ui/src/app/components/document-detail/document-detail.component.ts#L965-L974)

```typescript
onVersionSelected(versionId: number) {
    this.selectVersion(versionId)
}

onVersionsUpdated(versions: DocumentVersionInfo[]) {
    this.document.versions = versions
    const openDoc = this.openDocumentService.getOpenDocument(this.documentId)
    if (openDoc) {
        openDoc.versions = versions
        this.openDocumentService.save()  // 持久化到 OpenDocumentService
    }
}
```

`selectVersion()` 方法负责切换预览：
文件：[document-detail.component.ts](file:///d:/fz/0601/solo-dogfeeding/code/111-paperless-ngx/src-ui/src/app/components/document-detail/document-detail.component.ts#L907-L963)

```typescript
selectVersion(versionId: number) {
    this.selectedVersionId = versionId
    this.previewLoaded = false
    this.previewUrl = this.documentsService.getPreviewUrl(this.documentId, false, this.selectedVersionId)
    this.updatePdfSource()   // PDF.js 重新加载 PDF
    this.thumbUrl = this.documentsService.getThumbUrl(this.documentId, this.selectedVersionId)
    this.loadMetadataForSelectedVersion()  // 刷新元数据
    // 重新拉取预览文本
    this.http.get(this.previewUrl, { responseType: 'text' }).subscribe({
        next: (res) => (this.previewText = res.toString()),
    })
}
```

**链路 A 的最终 UI 刷新效果**：
1. 版本下拉列表出现新上传的版本条目
2. 自动选中最新版本（高亮）
3. 右侧 PDF 预览区自动加载新版本 PDF
4. 缩略图替换为新版本缩略图
5. 预览文本区更新为新版本的 OCR 文本
6. 上传状态从"Processing"变回"Idle"

---

#### 链路 B：document_updated(根文档) → DocumentDetailComponent

文件：[document-detail.component.ts](file:///d:/fz/0601/solo-dogfeeding/code/111-paperless-ngx/src-ui/src/app/components/document-detail/document-detail.component.ts#L662-L695)

这个组件在 `ngOnInit` 中**全局订阅** `onDocumentUpdated()`：

```typescript
this.websocketStatusService
  .onDocumentUpdated()
  .pipe(takeUntil(this.unsubscribeNotifier))
  .subscribe((data) => this.handleIncomingDocumentUpdated(data))
```

`handleIncomingDocumentUpdated()` 的四级判定逻辑：

```
handleIncomingDocumentUpdated(data):
    │
    ├─ 第 1 关：data.document_id !== this.documentId  → 不是当前文档，直接 return
    │   （上传新版本时 data.document_id = 根文档id，刚好匹配当前详情页的 documentId）
    │
    ├─ 第 2 关：this.networkActive === true
    │   └─ 有 HTTP 请求在处理 → this.pendingIncomingUpdate = data（缓存），等网络空闲再处理
    │
    ├─ 第 3 关：data.modified === this.lastLocalSaveModified
    │   └─ 时间戳对上了，是"自己刚刚保存产生的回声" → this.lastLocalSaveModified = null，return（忽略）
    │
    └─ 第 4 关：this.openDocumentService.isDirty(this.document)
        │
        ├─ 是（表单有未保存修改）：
        │   └─ showIncomingUpdateModal(data.modified)
        │      弹出对话框："This document has been modified elsewhere. Reload to see the latest changes?"
        │      按钮：Reload document / Keep editing
        │      点击 Reload → reloadRemoteVersion() → loadDocument(this.documentId, true)
        │
        └─ 否（表单干净）：
            └─ this.loadDocument(this.documentId, true)
               this.toastService.showInfo("Document reloaded with latest changes.")
```

**`loadDocument(documentId, forceRemote=true)` 的完整刷新流程**：
文件：[document-detail.component.ts](file:///d:/fz/0601/solo-dogfeeding/code/111-paperless-ngx/src-ui/src/app/components/document-detail/document-detail.component.ts#L492-L610)

```
loadDocument(documentId, forceRemote):
    │
    ├─ this.selectedVersionId = documentId
    ├─ this.previewUrl = documentsService.getPreviewUrl(selectedVersionId)
    ├─ this.updatePdfSource()
    ├─ HTTP GET previewUrl → 刷新 previewText
    ├─ this.thumbUrl = documentsService.getThumbUrl(selectedVersionId)
    │
    ├─ HTTP GET documentsService.get(documentId) → 拿到 doc（含 versions[] 数组）
    │   │
    │   ├─ if openDocument && forceRemote:  Object.assign(openDocument, doc)  ← 覆盖本地编辑
    │   ├─ else if openDocument: new Date(doc.modified) > new Date(openDocument.modified)
    │   │    └─ if hasLocalEdits → showIncomingUpdateModal else Object.assign
    │   └─ else openDocumentService.openDocument(doc)
    │
    ├─ this.updateComponent(useDoc)
    │   ├─ this.document = useDoc
    │   ├─ this.selectedVersionId = Math.max(...doc.versions.map(v => v.id)) || doc.id
    │   ├─ this.loadMetadataForSelectedVersion()
    │   ├─ this.title = documentTitlePipe.transform(doc.title)
    │   └─ this.prepareForm(doc)  ← 所有表单字段（标题/标签/对应人/日期/ASN/自定义字段...）重置为最新值
    │
    └─ this.setupDirtyTracking(useDoc, doc)  ← 重建脏值追踪
```

**链路 B 的最终 UI 刷新效果**：
1. 顶部标题栏显示最新标题
2. 所有表单字段（标题、标签、对应人、文档类型、存储路径、日期、ASN、所有者、权限、自定义字段等）更新为最新值
3. 元数据卡片重新加载
4. 版本下拉（通过 `[versions]="document?.versions"` 输入属性绑定）自动拿到最新 versions 数组
5. PDF 预览、缩略图、预览文本全部更新为最新版本内容
6. Toast 显示 "Document reloaded with latest changes."

---

#### 两条链路的时序与协同

由于 `document_updated` 在事务内发送，`status_update(SUCCESS)` 在事务外发送，且各自经过独立的 Redis Pub/Sub 通道和 WebSocket 消息，它们到达浏览器的顺序可能存在微小差异（通常 `document_updated` 先到）。两条链路刷新的内容也互补：

- **链路 A（status_update）**：精确知道"当前上传操作成功了"，负责上传状态 UI 复位、精确触发 `getVersions()` 拉取版本列表并自动选中新版本。
- **链路 B（document_updated）**：泛化的"文档有变化"通知，负责整个详情页所有元数据和预览内容的全量刷新。

> 潜在的优化空间：链路 A 已经调用了 `getVersions()` 并 `emit(versionsUpdated)` 更新了 `document.versions`，随后链路 B 的 `loadDocument()` 又会通过 `documentsService.get(documentId)` 再次拉取完整文档（含 versions），存在一次重复请求。由于两条消息到达存在时差，通常不会被用户感知。

### 5.7 上传新版本端到端完整时序图

```
 用户点击上传新版本
         │
         ▼
┌──────────────────────────────────────────────────────────────────────┐
│  浏览器                                                              │
│  DocumentVersionDropdownComponent.onVersionFileSelected()            │
│    └─ HTTP POST /api/documents/{id}/update_version/ ───────────────┐│
│       (multipart/form-data: file + label)                          ││
└────────────────────────────────────────────────────────────────────┘│
                                                                      │
                              ▲                                       │
                              │  Response: taskId                     │
                              │                                       │
┌─────────────────────────────┴──────────────────────────────────────┐│
│  浏览器                                                              ││
│    └─ versionUploadState = Processing                               ││
│       tap: toast "Uploading new version..."                         ││
│       switchMap: merge onSuccess/onFailed, filter by taskId         ││
│       (RxJS 订阅临时建立，等待对应 taskId 的 SUCCESS/FAILED)          ││
│                                                                     ││
│  同时 DocumentDetailComponent 早已在 ngOnInit 订阅 onDocumentUpdated  ││
└─────────────────────────────────────────────────────────────────────┘
                                                                      │
                              ▲                                       │
                              │  Celery Worker 异步执行                │
                              │                                       │
┌─────────────────────────────┴──────────────────────────────────────┐│
│  Django / Celery Worker                                              ││
│                                                                     ││
│  consume_file(taskId, input_doc={root_document_id: rootId})         ││
│    └─ ConsumerPlugin.run()                                          ││
│        ├─ 解析/缩略图/日期提取 → 多次 status_update(WORKING)        ││
│        ├─ transaction.atomic():                                     ││
│        │   ├─ _create_version_from_root() → version_doc.save()      ││
│        │   ├─ document_consumption_finished.send()                  ││
│        │   ├─ 文件写入 MEDIA 目录                                     ││
│        │   ├─ document.save()                                        ││
│        │   └─ ★ 消息① document_updated.send(root_document) ────────┼┼──→ Redis Pub/Sub
│        │                                                             ││
│        ├─ run_post_consume_script()                                  ││
│        └─ ★ 消息② _send_progress(SUCCESS, document_id=版本id) ─────┼┼──→ Redis Pub/Sub
│                                                                     ││
└─────────────────────────────────────────────────────────────────────┘│
                                                                      │
                              ▲                                       │
                              │  Redis Pub/Sub → Daphne → WebSocket   │
                              │  两条消息各自独立传播                    │
                              │                                       │
┌─────────────────────────────┴──────────────────────────────────────┐│
│  浏览器 (两条并发接收的消息)                                          ││
│                                                                     ││
│  ═══ 消息①到达：type=document_updated ══════════════════════════════││
│  WebsocketStatusService.handleDocumentUpdated()                    ││
│    └─ documentUpdatedSubject.next({document_id: rootId, modified}) ││
│       └─ DocumentDetailComponent.handleIncomingDocumentUpdated()    ││
│          ├─ 匹配当前 documentId ✓                                    ││
│          ├─ isDirty? → 通常为 false（刚上传，用户没开始改）           ││
│          └─ loadDocument(documentId, forceRemote=true)             ││
│             ├─ HTTP GET /api/documents/{rootId}/ → 完整文档+版本    ││
│             └─ updateComponent() → 全页 UI 刷新                     ││
│                                                                     ││
│  ═══ 消息②到达：type=status_update status=SUCCESS ═══════════════════││
│  WebsocketStatusService.handleProgressUpdate()                      ││
│    ├─ status.phase = SUCCESS                                        ││
│    └─ documentConsumptionFinishedSubject.next(status)               ││
│       └─ DocumentVersionDropdownComponent 内的 RxJS merge:          ││
│          ├─ filter: status.taskId === 本地taskId ✓ (take(1) 完成)   ││
│          ├─ switchMap: HTTP GET /api/documents/{rootId}/versions/   ││
│          ├─ versionsUpdated.emit(versions) → 父组件更新 versions    ││
│          ├─ versionSelected.emit(新版本id) → selectVersion()        ││
│          └─ versionUploadState = Idle (上传状态清除)                ││
│                                                                     ││
└─────────────────────────────────────────────────────────────────────┘
```

---

## 六、完整数据流转图

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

## 七、关键设计要点

### 7.1 前后端双权限校验
- **后端**：StatusConsumer 在发送每条消息前检查权限，避免越权信息通过 WebSocket 泄漏
- **前端**：WebsocketStatusService 收到消息后再次检查，作为防御性兜底
- 权限判定逻辑完全一致，防止逻辑漂移

### 7.2 统一 Group + Consumer 端过滤
- 所有客户端共享一个 `status_updates` 组，避免为每个用户创建独立 channel 造成 Redis 资源消耗
- 过滤放在 Consumer 侧执行（异步 `aexists()` 查询用户组），权衡了实现复杂度与隔离性

### 7.3 上下文管理器确保消息可靠发送
- `ProgressManager` 和 `DocumentsStatusManager` 均使用 `__enter__/__exit__` 管理 channel layer 生命周期
- 退出时 flush channel layer，避免在 Celery worker 进程中消息卡在发送缓冲区

### 7.4 异步 Consumer + 同步发送桥接
- Consumer 继承 `AsyncWebsocketConsumer`，所有 I/O 均为 async（channel layer 操作、DB 查询）
- 事件产生方（Celery、API）在同步上下文中运行，通过 `asgiref.sync.async_to_sync` 桥接调用

### 7.5 `QuerySet.update()` 不触发信号的补偿机制
- Django ORM 批量更新 (`qs.update()`) 不会触发 `post_save`，也不走 serializer 的 `save()`
- Paperless-ngx 专门设计了 `bulk_update_documents` Celery 任务，在批量改 DB 后显式发送 `document_updated` 信号，保证 WebSocket 推送不丢失

### 7.6 详情页的乐观并发控制
- DocumentDetailComponent 收到 `document_updated` 时，对比 `modified` 时间戳判断是否为"自己保存产生的回声"
- 如果用户正在编辑 (form dirty)，弹确认对话框让用户决定是否覆盖本地修改，避免丢失编辑内容

### 7.7 定时工作流的特殊处理
- 定时工作流 `check_scheduled_workflows()` **不通过** `document_updated` 信号发送通知，而是直接调用 `send_websocket_document_updated()`
- 代码注释明确说明原因：SCHEDULED 类型的工作流执行后，内部不会自动触发 `document_updated` 信号，需兜底直接推送

### 7.8 上传新版本场景的双消息协同
- `document_updated` 在事务内发送（根文档 id）→ DocumentDetailComponent 负责全页元数据刷新
- `status_update(SUCCESS)` 在事务外发送（新版本 id）→ DocumentVersionDropdownComponent 负责版本列表刷新 + 状态复位
- 两条消息互补但存在一次潜在重复请求（getVersions vs get 都拉 versions）

### 7.9 事务内发送通知的竞态窗口
- **双写割裂问题**：Django `transaction.atomic()` 无法控制 Redis Pub/Sub 的发送时机，`async_to_sync(channel_layer.group_send)` 在事务内调用时会立即发出 Redis 消息，而 DB COMMIT 可能在几毫秒~几十毫秒之后才发生
- **versions 查询的可见性完全取决于 COMMIT 时机**：`DocumentSerializer.get_versions()` 通过独立 SQL `Document.objects.filter(root_document=root_doc)` 拉取版本列表，不经过 prefetch 缓存，查询结果严格遵循 PostgreSQL MVCC，只能看到 SQL 执行时刻已 COMMIT 的行
- **document_updated 消息到达时 versions 数组可能缺失最新版本**：事务未 COMMIT 前 `filter(root_document=root_doc)` 查不到新插入的 version_doc 行，详情页第一次 reload 时版本下拉少一条
- **信号处理器的内部可见性**：`send_websocket_document_updated()` 内的 `refresh_from_db()` 在发送者事务内执行，能看到同一连接上未提交数据，但这对其他连接（前端 HTTP 请求）无效
- **缓解机制**：`status_update(SUCCESS)` 在事务外 + post consume 脚本之后发送，是端到端可靠的完成信号，会触发第二次 `getVersions()` 拉取，此时 COMMIT 已完成，versions 一定完整
- **改进方案**：对 `send_websocket_document_updated` 处理器单独使用 `transaction.on_commit()` 延迟 Redis 发送到 COMMIT 之后，但不能对整个 `document_updated` 信号做此处理（因为 `run_workflows_updated` 必须在事务内原子提交）

### 7.10 根文档 modified 不变的代码核准结论
- **上传新版本流程本身从不 save root_doc**：consumer.py 中 root_doc 的 7 处使用点全部是查询、加行锁、外键关联、LogEntry 写入，没有一处调用 `root_doc.save()`，因此 `auto_now=True` 的 modified 字段保持旧值
- **唯一例外：DOCUMENT_UPDATED 工作流**：如果配置了匹配的工作流，`run_workflows()` 会显式调用 `document.save(update_fields=[..., "modified"])` 更新 root_doc，此时 `send_websocket_document_updated()` 中的 `refresh_from_db()` 能在同一事务内看到新时间戳
- **refresh_from_db 的连接可见性边界**：只能看到同一 DB 连接中的写入，前端新 HTTP 请求使用独立连接，在 COMMIT 之前看不到工作流对 root_doc 的修改
- **对前端回声判定几乎无影响**：`handleIncomingDocumentUpdated()` 只对比 `lastLocalSaveModified`（用户本地 PATCH 保存时才设置），正常查看时为 null，旧时间戳不会被误判为回声

### 7.11 与 SSE 的区别
系统仅在 AI Chat 功能中使用 SSE 风格的 `StreamingHttpResponse`（[views.py](file:///d:/fz/0601/solo-dogfeeding/code/111-paperless-ngx/src/documents/views.py#L2181-L2185)），用于流式输出 LLM 回答。该实现与实时通知系统完全独立，不经过 WebSocket / Channels 层。
