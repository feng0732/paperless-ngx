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

## 二、事件产生层

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
        # ... 多个插件顺序执行，每个插件都可调用 status_mgr.send_progress()
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

### 2.3 事件源二：文档更新通知（document_updated）

**触发机制**：Django 自定义信号 `document_updated`

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
    # ... 更多处理器
    document_updated.connect(run_workflows_updated)
    document_updated.connect(send_websocket_document_updated)  # ← WebSocket 发送
```

**处理器实现**：

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

**触发场景**：
- `bulk_update_documents` Celery 任务（批量编辑）
- 工作流执行后文档属性变更

### 2.4 事件源三：文档删除通知（documents_deleted）

**触发位置**：批量删除操作

文件：[bulk_edit.py](file:///d:/fz/0601/solo-dogfeeding/code/111-paperless-ngx/src/documents/bulk_edit.py#L383-L384)

```python
status_mgr = DocumentsStatusManager()
status_mgr.send_documents_deleted(delete_ids)
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

### 4.1 WebSocket 服务

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

#### 消息路由

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
| `onDocumentDetected()` | `Subject<FileStatus>` | 新文件开始处理（STARTED） |
| `onDocumentConsumptionFinished()` | `Subject<FileStatus>` | 处理成功（SUCCESS） |
| `onDocumentConsumptionFailed()` | `Subject<FileStatus>` | 处理失败（FAILED） |
| `onDocumentDeleted()` | `Subject<boolean>` | 任意文档被删除 |
| `onDocumentUpdated()` | `Subject<WebsocketDocumentUpdatedMessage>` | 文档元数据变更 |
| `onConnectionStatus()` | `Observable<boolean>` | WebSocket 连接/断开 |

---

## 五、完整数据流转图

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        事件产生层（Celery Worker / API）                  │
│                                                                         │
│  ┌──────────────────┐   ┌──────────────────┐   ┌──────────────────┐    │
│  │  consume_file()  │   │ document_updated │   │  bulk_edit.delete│    │
│  │  (Celery Task)   │   │  (Django Signal) │   │  (API Handler)   │    │
│  └────────┬─────────┘   └────────┬─────────┘   └────────┬─────────┘    │
│           │                      │                      │              │
│           ▼                      ▼                      ▼              │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │              ProgressManager / DocumentsStatusManager           │    │
│  │         async_to_sync(channel_layer.group_send("status_updates",│    │
│  │                          {type, data}))                         │    │
│  └────────────────────────────┬────────────────────────────────────┘    │
└─────────────────────────────┼──────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                       消息中间件（Redis Pub/Sub）                         │
│                    CHANNEL_LAYERS → "status_updates" 组                  │
└─────────────────────────────┬──────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                    连接维护层（Daphne / ASGI Worker）                     │
│                                                                         │
│  StatusConsumer (AsyncWebsocketConsumer) 每个连接一个实例                 │
│  ┌──────────────────────────────────────────────────────────────────┐   │
│  │ connect() → _authenticated()? → group_add("status_updates")     │   │
│  │                                                                  │   │
│  │ status_update(event)  → _can_view(data)? → send(json.dumps(event))│  │
│  │ document_updated(event)→ _can_view(data)? → send(json.dumps(event))│  │
│  │ documents_deleted(event) →             → send(json.dumps(event)) │   │
│  │                                                                  │   │
│  │ disconnect() → group_discard("status_updates")                  │   │
│  └────────────────────────────┬─────────────────────────────────────┘   │
└─────────────────────────────┼──────────────────────────────────────────┘
                              │  WebSocket (ws://host/ws/status/)
                              ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                         前端层（Angular Browser）                         │
│                                                                         │
│  WebsocketStatusService (providedIn: 'root')                             │
│  ┌──────────────────────────────────────────────────────────────────┐   │
│  │ connect() → new WebSocket(url)                                   │   │
│  │ onmessage → JSON.parse → switch(type)                            │   │
│  │   ├─ status_update    → handleProgressUpdate → canViewMessage?   │   │
│  │   │                      → consumerStatus[] + RxJS Subjects      │   │
│  │   ├─ document_updated → handleDocumentUpdated → canViewMessage?  │   │
│  │   │                      → documentUpdatedSubject                │   │
│  │   └─ documents_deleted → documentDeletedSubject                  │   │
│  └──────────────────────────────────────────────────────────────────┘   │
│                              │                                          │
│                              ▼                                          │
│  各 Angular 组件订阅 Subject：document-list、toast 通知、进度条等         │
└─────────────────────────────────────────────────────────────────────────┘
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

### 6.5 与 SSE 的区别
系统仅在 AI Chat 功能中使用 SSE 风格的 `StreamingHttpResponse`（[views.py](file:///d:/fz/0601/solo-dogfeeding/code/111-paperless-ngx/src/documents/views.py#L2181-L2185)），用于流式输出 LLM 回答。该实现与实时通知系统完全独立，不经过 WebSocket / Channels 层。
