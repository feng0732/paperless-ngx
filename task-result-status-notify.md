# Paperless-ngx 任务结果回传与状态通知串联梳理

## 一、全局概览

整个链路分为 **4 层**，从前端发起 → 后端入队 → Worker 执行 → 结果/通知回传 → 前端感知：

```
前端 (Angular)
  │  HTTP POST /api/documents/post_document/
  │  返回 Celery task_id
  ▼
Django REST API (PostDocumentView)
  │  consume_file.apply_async(...)
  │  Celery broker (Redis)
  ▼
Celery Worker 执行 consume_file
  │  ├─ ProgressManager.send_progress()  ──→  Channel Layer ──→  WebSocket ──→  前端实时进度
  │  ├─ document_consumption_finished signal ──→  分类/匹配/索引/Workflow
  │  └─ 返回 ConsumeFileSuccessResult / ConsumeFileDuplicateResult / ConsumeFileStoppedResult
  ▼
Celery Signal Handlers (before_task_publish / task_prerun / task_postrun / task_failure / task_revoked)
  │  更新 PaperlessTask ORM 记录 (DB 持久化)
  ▼
前端轮询 /api/tasks/  读取 PaperlessTask 列表
```

---

## 二、后台任务定义与调度

### 2.1 Celery Task 注册

所有后台任务定义在 [tasks.py](src/documents/tasks.py) 中，使用 `@shared_task` 装饰器：

| Task 函数 | PaperlessTask.TaskType | 说明 |
|---|---|---|
| `consume_file` | CONSUME_FILE | 核心：消费文档（OCR/解析/存储） |
| `train_classifier` | TRAIN_CLASSIFIER | 训练自动分类模型 |
| `sanity_check` | SANITY_CHECK | 系统健康检查 |
| `llmindex_index` | LLM_INDEX | LLM 索引构建 |
| `empty_trash` | EMPTY_TRASH | 清空回收站 |
| `check_scheduled_workflows` | CHECK_WORKFLOWS | 定时工作流 |
| `bulk_update_documents` | BULK_UPDATE | 批量更新文档 |
| `update_document_content_maybe_archive_file` | REPROCESS_DOCUMENT | 重新处理文档 |
| `build_share_link_bundle` | BUILD_SHARE_LINK | 构建分享链接包 |

此外，[bulk_edit.py](src/documents/bulk_edit.py) 中 `delete` 对应 `BULK_DELETE`，邮件处理由 `paperless_mail.tasks.process_mail_accounts` 对应 `MAIL_FETCH`。

### 2.2 任务入队方式

以文档消费为例，[PostDocumentView.post()](src/documents/views.py#L3098-L3160) 中：

```python
async_task = consume_file.apply_async(
    kwargs={"input_doc": input_doc, "overrides": input_doc_overrides},
    headers={
        "trigger_source": (
            PaperlessTask.TriggerSource.WEB_UI
            if from_webui
            else PaperlessTask.TriggerSource.API_UPLOAD
        ),
    },
)
return Response(async_task.id)
```

关键点：`headers` 中的 `trigger_source` 被传递到 Celery 消息头，用于任务追踪记录。

### 2.3 触发来源 (TriggerSource)

定义在 [models.py](src/documents/models.py#L701-L708)：

| TriggerSource | 含义 |
|---|---|
| SCHEDULED | Celery beat 定时调度 |
| WEB_UI | 用户通过 Web 界面上传 |
| API_UPLOAD | 通过 REST API 上传 |
| FOLDER_CONSUME | 消费目录自动检测 |
| EMAIL_CONSUME | 邮件规则获取 |
| SYSTEM | 系统自动触发（自愈/配置副作用） |
| MANUAL | 用户通过 `/api/tasks/run/` 手动触发 |

---

## 三、任务结果存取机制

### 3.1 PaperlessTask 模型 (DB 持久化)

定义在 [models.py](src/documents/models.py#L664-L826)，核心字段：

| 字段 | 类型 | 说明 |
|---|---|---|
| `task_id` | CharField(72) | Celery UUID，唯一键 |
| `task_type` | CharField(50) | 任务类型枚举 |
| `trigger_source` | CharField(50) | 触发来源枚举 |
| `status` | CharField(30) | PENDING → STARTED → SUCCESS/FAILURE/REVOKED |
| `date_created` | DateTime | 任务创建时间 |
| `date_started` | DateTime | Worker 开始执行时间 |
| `date_done` | DateTime | 完成时间 |
| `duration_seconds` | Float | 执行耗时 |
| `wait_time_seconds` | Float | 等待耗时（从创建到开始） |
| `input_data` | JSONField | 结构化输入参数 |
| `result_data` | JSONField | 结构化结果数据 |
| `acknowledged` | Boolean | 用户是否已确认/已读 |
| `owner` | FK(User) | 任务归属用户 |

**状态流转**：
```
PENDING ──→ STARTED ──→ SUCCESS
                       → FAILURE
         ──→ REVOKED (取消)
```

### 3.2 Celery Signal Handlers — 自动写入 PaperlessTask

定义在 [handlers.py](src/documents/signals/handlers.py#L1001-L1313)，这是连接 Celery 生命周期与 DB 持久化的核心桥梁：

| Celery Signal | Handler 函数 | 操作 |
|---|---|---|
| `before_task_publish` | [before_task_publish_handler](src/documents/signals/handlers.py#L1103-L1141) | 任务发布到 broker 时 **创建** PaperlessTask 记录 (status=PENDING)，提取 input_data、trigger_source、owner_id |
| `task_prerun` | [task_prerun_handler](src/documents/signals/handlers.py#L1144-L1162) | Worker 开始执行时更新 status=STARTED、date_started |
| `task_postrun` | [task_postrun_handler](src/documents/signals/handlers.py#L1165-L1223) | 任务正常完成时更新 status=SUCCESS、date_done、duration_seconds、result_data。若 result_data 含 `duplicate_of` 则改为 FAILURE |
| `task_failure` | [task_failure_handler](src/documents/signals/handlers.py#L1226-L1278) | 任务异常时更新 status=FAILURE、date_done、result_data (含 error_type/error_message/traceback) |
| `task_revoked` | [task_revoked_handler](src/documents/signals/handlers.py#L1281-L1313) | 任务被撤销时更新 status=REVOKED、date_done |

**追踪范围**：仅 [TRACKED_TASKS](src/documents/signals/handlers.py#L1005-L1017) 字典中列出的任务名才会被记录，其余 Celery task 静默忽略。

### 3.3 任务返回值 → result_data 的映射

`consume_file` 的返回值是 TypedDict（定义在 [data_models.py](src/documents/data_models.py#L190-L210)）：

- `ConsumeFileSuccessResult`: `{"document_id": int}` → 写入 result_data
- `ConsumeFileDuplicateResult`: `{"duplicate_of": int, "duplicate_in_trash": bool}` → 写入 result_data，且 status 被改为 FAILURE
- `ConsumeFileStoppedResult`: `{"reason": str}` → 写入 result_data

[task_postrun_handler](src/documents/signals/handlers.py#L1214-L1219) 中判断 `isinstance(retval, dict)` 时自动将返回值写入 `result_data`。

### 3.4 REST API 查询

[TasksViewSet](src/documents/views.py#L4055-L4244) 提供以下端点：

| 端点 | 方法 | 说明 |
|---|---|---|
| `/api/tasks/` | GET | 分页查询任务列表，支持按 task_type/status/acknowledged 过滤 |
| `/api/tasks/acknowledge/` | POST | 批量确认（标记已读） |
| `/api/tasks/run/` | POST | 手动触发任务（仅超级用户，限 TRAIN_CLASSIFIER / SANITY_CHECK / LLM_INDEX） |
| `/api/tasks/summary/` | GET | 聚合统计（按 task_type 分组，含总数/成功/失败/平均耗时） |
| `/api/tasks/active/` | GET | 当前活跃任务（PENDING + STARTED，上限 50） |

**权限**：普通用户只能看到自己的任务 + 无主任务（系统/定时任务）；staff 可看全部。

---

## 四、通知触发机制

通知有 **两条并行通道**：WebSocket 实时推送 + DB 持久化（前端轮询）。

### 4.1 WebSocket 实时推送

#### 4.1.1 Channel Layer 架构

```
ProgressManager / DocumentsStatusManager
  │  channel_layer.group_send("status_updates", payload)
  ▼
Redis PubSub (channels_redis)
  │
  ▼
StatusConsumer (Django Channels ASGI)
  │  权限过滤 (_can_view)
  ▼
前端 WebSocket 连接
```

#### 4.1.2 三种消息类型

定义在 [helpers.py](src/documents/plugins/helpers.py#L15-L72)：

| type | Payload 结构 | 触发时机 |
|---|---|---|
| `status_update` | `{filename, task_id, current_progress, max_progress, status, message, document_id, owner_id, users_can_view, groups_can_view}` | 文档消费过程中每个阶段 |
| `documents_deleted` | `{documents: [id, ...]}` | 批量删除文档后 |
| `document_updated` | `{document_id, modified, owner_id, users_can_view, groups_can_view}` | 文档更新后（含 workflow 触发的更新） |

#### 4.1.3 进度管理器

[ProgressManager](src/documents/plugins/helpers.py#L119-L150) 是消费过程中发送实时进度的核心，作为 context manager 使用：

```python
with ProgressManager(filename, self.request.id) as status_mgr:
    # ... 在各 plugin 中通过 status_mgr.send_progress() 发进度
```

[DocumentsStatusManager](src/documents/plugins/helpers.py#L153-L182) 负责发送 `documents_deleted` 和 `document_updated` 事件。

#### 4.1.4 StatusConsumer (WebSocket 服务端)

定义在 [consumers.py](src/paperless/consumers.py#L18-L62)：

- 连接时验证认证、加入 `status_updates` channel group
- `status_update` 和 `document_updated` 事件会进行权限过滤 (`_can_view`)：超级用户/所有者/view 权限用户/group 权限用户可收到
- `documents_deleted` 不做权限过滤（所有连接的客户端都收到）

WebSocket 路由定义在 [urls.py](src/paperless/urls.py#L420-L421)：`ws/status/`

ASGI 入口在 [asgi.py](src/paperless/asgi.py#L18-L23)，使用 `AuthMiddlewareStack` 确保认证。

#### 4.1.5 进度阶段与消息

[ProgressStatusOptions](src/documents/plugins/helpers.py#L15-L19)：

| Phase | 含义 |
|---|---|
| STARTED | 任务开始（0%） |
| WORKING | 处理中（20%~95%） |
| SUCCESS | 成功（100%） |
| FAILED | 失败（100%） |

ConsumerPlugin 在消费流程各节点发送具体消息（[consumer.py](src/documents/consumer.py#L103-L121)）：

| ConsumerStatusShortMessage | 阶段 | 进度 |
|---|---|---|
| `new_file` | STARTED | 0% |
| `parsing_document` | WORKING | 20% |
| `generating_thumbnail` | WORKING | 70% |
| `parse_date` | WORKING | 90% |
| `save_document` | WORKING | 95% |
| `finished` | SUCCESS | 100% |
| `failed` | FAILED | 100% |
| `document_already_exists` | FAILED | 100% |
| `asn_already_exists` | FAILED | 100% |

### 4.2 Django Signal 驱动的文档级通知

#### 4.2.1 自定义 Signal 定义

[signals/__init__.py](src/documents/signals/__init__.py)：

| Signal | 发送时机 |
|---|---|
| `document_consumption_started` | 文档消费开始解析前 |
| `document_consumption_finished` | 文档消费完成（DB 事务内） |
| `document_updated` | 文档被更新（版本新增/Workflow 修改后） |

#### 4.2.2 Signal 连接 (apps.py)

[apps.py](src/documents/apps.py#L24-L33) 在 `DocumentsConfig.ready()` 中注册：

```
document_consumption_finished → add_inbox_tags
                             → set_correspondent
                             → set_document_type
                             → set_tags
                             → set_storage_path
                             → add_to_index
                             → run_workflows_added
                             → add_or_update_document_in_llm_index

document_updated             → run_workflows_updated
                             → send_websocket_document_updated  ← 关键：触发 WebSocket document_updated 事件
```

**注意**：`document_consumption_finished` 与 `document_updated` 是**两个独立信号**，前者不会触发后者。Django 内置的 `post_save` 也不会触发自定义的 `document_updated` 信号。

#### 4.2.3 `send_websocket_document_updated` 的调用路径

[send_websocket_document_updated](src/documents/signals/handlers.py#L832-L851) 只在 `document_updated` 信号被发送时触发，而该信号需**显式调用** `document_updated.send()`：

1. **新版本消费**：`ConsumerPlugin.run()` 中，`document.save()` 之后，**仅当** `document.root_document_id` 存在时才发送 `document_updated.send()`（[consumer.py#L731-L735](src/documents/consumer.py#L731-L735)）。**新建文档（非版本更新）不会触发此信号。**
2. **定时工作流**：`check_scheduled_workflows` 中直接调用 `send_websocket_document_updated()`（[tasks.py#L569-L573](src/documents/tasks.py#L569-L573)）
3. **批量更新**：`bulk_update_documents` 调用 `document_updated.send()`（[tasks.py#L260-L264](src/documents/tasks.py#L260-L264)）

#### 4.2.4 文档删除通知

[bulk_edit.py](src/documents/bulk_edit.py#L383-L384) 中批量删除后：

```python
status_mgr = DocumentsStatusManager()
status_mgr.send_documents_deleted(delete_ids)
```

---

## 五、前端感知与状态更新（重点修正）

文档消费完毕后，前端通过**三条独立链路**感知结果，它们的触发时机、数据来源和传递路径完全不同：

### 5.1 三条链路总览

| 链路 | 通道 | 触发机制 | 到达时机 | 携带数据 |
|---|---|---|---|---|
| **① 成功进度** | WebSocket `status_update` | `ConsumerPlugin._send_progress(SUCCESS)` 直接推送 | **最先到达**（任务 return 前） | filename, task_id, document_id, phase=SUCCESS |
| **② document_updated** | WebSocket `document_updated` | `document_updated` signal → `send_websocket_document_updated()` | **仅版本更新/批量更新/定时工作流时到达** | document_id, modified, owner_id |
| **③ 任务列表** | HTTP REST `GET /api/tasks/` | Celery `task_postrun` handler 更新 DB，前端主动轮询 | **最晚到达**（任务 return 后） | PaperlessTask 完整记录 |

### 5.2 链路①：成功进度（status_update）

**后端发送路径**（按代码顺序）：

1. [ConsumerPlugin.run()](src/documents/consumer.py#L408-L784) 完成所有消费逻辑后
2. 退出 `with transaction.atomic()` 和文件锁
3. 执行 `run_post_consume_script(document)`（[consumer.py#L769](src/documents/consumer.py#L769)）
4. 调用 [self._send_progress(100, 100, SUCCESS, FINISHED, document.id)](src/documents/consumer.py#L773-L779)
5. `_send_progress` → [ProgressManager.send_progress()](src/documents/plugins/helpers.py#L125-L150) → `channel_layer.group_send("status_updates", payload)`
6. [StatusConsumer.status_update()](src/paperless/consumers.py#L46-L50) 进行权限过滤后推送到前端 WebSocket

**前端接收路径**：

1. [WebsocketStatusService.handleProgressUpdate()](src-ui/src/app/services/websocket-status.service.ts#L228-L270)
2. `canViewMessage()` 权限过滤
3. 更新 `FileStatus`：`status.updateProgress(FileStatusPhase.WORKING, ...)` → 然后 `status.phase = FileStatusPhase[messageData.status]` 将 phase 覆盖为 SUCCESS
4. `switch (status.phase)` 进入 `FileStatusPhase.SUCCESS` 分支 → `documentConsumptionFinishedSubject.next(status)`
5. 订阅此 Subject 的组件得到通知，可获取 `status.documentId`

**关键**：这条链路在 `consume_file` 函数 return 之前就触发，是最先到达前端的通知。

### 5.3 链路②：document_updated 消息

**后端发送路径** — 需要分场景：

**场景 A：新建文档消费** — **不触发 document_updated**

1. `ConsumerPlugin.run()` 中 `document.save()` 后（[consumer.py#L729](src/documents/consumer.py#L729)），条件 `if document.root_document_id` 为 False（新文档无根文档）
2. 因此 `document_updated.send()` **不被调用**
3. `document_consumption_finished` signal 触发的 `run_workflows_added` 中，`document.save()` 是 Django `post_save`，**不会**触发自定义 `document_updated` 信号
4. **结论**：新建文档消费成功后，前端不会收到 `document_updated` WebSocket 消息，仅靠链路①的 status_update(SUCCESS) 感知

**场景 B：新版本消费** — **触发 document_updated**

1. `ConsumerPlugin.run()` 中 `document.save()` 后，`document.root_document_id` 为真
2. [document_updated.send(sender, document=document.root_document)](src/documents/consumer.py#L732-L735) 被调用
3. Signal handler [send_websocket_document_updated()](src/documents/signals/handlers.py#L832-L851) 从 DB 读取最新权限数据，通过 `DocumentsStatusManager.send_document_updated()` 推送
4. 前端通过 [handleDocumentUpdated()](src-ui/src/app/services/websocket-status.service.ts#L272-L279) 接收 → `documentUpdatedSubject.next(data)`

**场景 C：批量更新/定时工作流** — 直接触发

- `bulk_update_documents` 发送 `document_updated` signal（[tasks.py#L260-L264](src/documents/tasks.py#L260-L264)）
- `check_scheduled_workflows` 直接调用 `send_websocket_document_updated()`（[tasks.py#L569-L573](src/documents/tasks.py#L569-L573)）

**前端接收路径**：

1. [WebsocketStatusService.handleDocumentUpdated()](src-ui/src/app/services/websocket-status.service.ts#L272-L279)
2. `canViewMessage()` 权限过滤
3. `documentUpdatedSubject.next(messageData)` → 订阅组件刷新文档列表/详情

### 5.4 链路③：任务列表（REST API）

**后端写入路径**（按代码顺序）：

1. `consume_file` 函数 return `ConsumeFileSuccessResult(document_id=document.pk)`
2. Celery 触发 `task_postrun` signal → [task_postrun_handler](src/documents/signals/handlers.py#L1165-L1223)
3. 更新 `PaperlessTask` 记录：`status=SUCCESS`, `date_done=now`, `duration_seconds=...`, `result_data={"document_id": N}`
4. DB 持久化完成

**前端读取路径**：

1. [TasksService.reload()](src-ui/src/app/services/tasks.service.ts#L69-L86) 发起 `GET /api/tasks/?acknowledged=false&page_size=1000`
2. 响应经 [TaskSerializerV10](src/documents/serialisers.py#L2442-L2484) 序列化后返回
3. 前端更新 `fileTasks` 数组，组件可访问 `completedFileTasks` / `failedFileTasks` 等过滤视图

**关键**：这条链路依赖前端主动轮询，是最晚到达的通知方式，但它提供了最完整的状态数据（包括 result_data 中的 document_id / duplicate_of 等结构化结果）。

### 5.5 上传流程 (UploadDocumentsService)

[upload-documents.service.ts](src-ui/src/app/services/upload-documents.service.ts) 衔接 HTTP 上传与 WebSocket 进度：

```
1. newFileUpload(filename) → 创建 FileStatus (phase=STARTED)
2. HTTP POST /api/documents/post_document/
   ├─ UploadProgress → FileStatusPhase.UPLOADING (0~20%)
   └─ Response → 拿到 task_id，phase 仍在 UPLOADING
3. 此后由 WebSocket 接管进度更新
```

### 5.6 WebSocket 进度接收 (WebsocketStatusService)

[websocket-status.service.ts](src-ui/src/app/services/websocket-status.service.ts)：

**连接**：`connect()` 建立 WebSocket 到 `ws/status/`

**消息分发**（`onmessage`）：

```
switch (type):
  case STATUS_UPDATE:
    handleProgressUpdate(data)
    → canViewMessage() 过滤
    → 更新 FileStatus (phase/progress/message/documentId)
    → 根据新 phase 发出 Subject 通知:
       STARTED  → documentDetectedSubject
       SUCCESS  → documentConsumptionFinishedSubject
       FAILED   → documentConsumptionFailedSubject

  case DOCUMENT_UPDATED:
    handleDocumentUpdated(data)
    → canViewMessage() 过滤
    → documentUpdatedSubject.next(data)

  case DOCUMENTS_DELETED:
    documentDeletedSubject.next(true)
```

**进度计算**（[FileStatus.getProgress()](src-ui/src/app/services/websocket-status.service.ts#L61-L75)）：

| Phase | 进度 |
|---|---|
| STARTED | 0.0 |
| UPLOADING | progress/max × 0.2 (0~20%) |
| WORKING | progress/max × 0.8 + 0.2 (20~100%) |
| SUCCESS/FAILED | 1.0 (100%) |

### 5.7 任务列表管理 (TasksService)

[tasks.service.ts](src-ui/src/app/services/tasks.service.ts)：

- `reload()`: GET `/api/tasks/?acknowledged=false&page_size=1000` 加载未确认的任务
- `list(page, pageSize, extraParams)`: 分页查询
- `dismissTasks(task_ids)`: POST `/api/tasks/acknowledge/` 批量确认
- `run(taskType)`: POST `/api/tasks/run/` 手动触发任务

### 5.8 前端数据模型

[PaperlessTask](src-ui/src/app/data/paperless-task.ts) — 与后端 [TaskSerializerV10](src/documents/serialisers.py#L2442-L2484) 一一对应：

```typescript
interface PaperlessTask {
  task_id: string
  task_type: PaperlessTaskType
  trigger_source: PaperlessTaskTriggerSource
  status: PaperlessTaskStatus       // pending | started | success | failure | revoked
  date_created: Date
  date_started?: Date
  date_done?: Date
  duration_seconds?: number
  wait_time_seconds?: number
  input_data: Record<string, any>
  result_data?: Record<string, any>
  related_document_ids: number[]
  acknowledged: boolean
  owner?: number
}
```

[WebsocketProgressMessage](src-ui/src/app/data/websocket-progress-message.ts)：

```typescript
interface WebsocketProgressMessage {
  filename?: string
  task_id?: string
  current_progress?: number
  max_progress?: number
  status?: string           // STARTED | WORKING | SUCCESS | FAILED
  message?: string
  document_id: number
  owner_id?: number
  users_can_view?: number[]
  groups_can_view?: number[]
}
```

---

## 六、完整时序图（以文档上传消费为例）

### 6.1 新建文档消费

```
前端                        Django API              Celery Broker          Celery Worker                  WebSocket
 │                            │                         │                      │                             │
 │─ POST /documents/post_document/ ─→│                    │                      │                             │
 │                            │── apply_async() ──────→│                      │                             │
 │←── Response(task_id) ─────│                         │                      │                             │
 │                            │                         │                      │                             │
 │                            │   [before_task_publish] │                      │                             │
 │                            │   创建 PaperlessTask    │                      │                             │
 │                            │   status=PENDING        │                      │                             │
 │                            │                         │                      │                             │
 │                            │                         │── 消息分发 ─────────→│                             │
 │                            │                         │                      │                             │
 │                            │                         │   [task_prerun]      │                             │
 │                            │                         │   PaperlessTask      │                             │
 │                            │                         │   status=STARTED     │                             │
 │                            │                         │                      │                             │
 │                            │                         │   ConsumerPreflightPlugin                          │
 │                            │                         │   send_progress(0,100,STARTED)──→ channel_layer ──→│
 │←─────────────────────────────────────────────────────────────────────────────────── status_update ─────│
 │  FileStatus.phase=STARTED  │                         │                      │                             │
 │                            │                         │   ConsumerPlugin                                  │
 │                            │                         │   send_progress(20,100,WORKING)─→ channel_layer ──→│
 │←─────────────────────────────────────────────────────────────────────────────────── status_update ─────│
 │  FileStatus.phase=WORKING  │                         │                      │                             │
 │                            │                         │   ...70%...90%...95%│                             │
 │                            │                         │                      │                             │
 │                            │                         │   [事务内]                                       │
 │                            │                         │   document.save()                                  │
 │                            │                         │   document_consumption_finished.send()             │
 │                            │                         │     → add_inbox_tags, set_correspondent, ...       │
 │                            │                         │     → run_workflows_added → workflow actions       │
 │                            │                         │     → add_to_index, add_or_update_document_in_llm │
 │                            │                         │   （注意：这些 handler 不会触发 document_updated） │
 │                            │                         │                      │                             │
 │                            │                         │   [事务外]                                       │
 │                            │                         │   run_post_consume_script()                        │
 │                            │                         │   send_progress(100,100,SUCCESS) → channel_layer ─→│
 │←─────────────────────────────────────────────────────────────────────────────────── status_update ─────│  ← 链路①
 │  FileStatus.phase=SUCCESS  │                         │                      │                             │
 │  documentConsumptionFinishedSubject.next(status)      │                      │                             │
 │                            │                         │                      │                             │
 │                            │                         │   return ConsumeFileSuccessResult                  │
 │                            │                         │                      │                             │
 │                            │                         │   [task_postrun]     │                             │
 │                            │                         │   PaperlessTask      │                             │
 │                            │                         │   status=SUCCESS     │                             │
 │                            │                         │   result_data={document_id: N}                     │
 │                            │                         │                      │                             │
 │                            │                         │   （新建文档不触发 document_updated 信号）          │
 │                            │                         │   （前端不会收到 document_updated WebSocket 消息）  │
 │                            │                         │                      │                             │
 │─ GET /api/tasks/ ─────────→│                         │                      │                             │
 │←── PaperlessTask list ────│                         │                      │             ← 链路③         │
```

### 6.2 新版本消费

```
Celery Worker                                                            WebSocket
 │                                                                        │
 │   [事务内]                                                             │
 │   document.save()                                                      │
 │   document.root_document_id 为真 →                                     │
 │     document_updated.send(sender, document=document.root_document)     │
 │       → send_websocket_document_updated()                              │
 │         → DocumentsStatusManager.send_document_updated()               │
 │           → channel_layer.group_send() ──────────────────────────────→ │  ← 链路②
 │                                                                        │
 │   [事务外]                                                             │
 │   run_post_consume_script()                                            │
 │   send_progress(100,100,SUCCESS) → channel_layer ────────────────────→ │  ← 链路①
 │                                                                        │
 │   return ConsumeFileSuccessResult                                      │
 │   [task_postrun] → PaperlessTask status=SUCCESS                        │
 │                                                                        │
```

**注意**：对于版本更新，链路②（document_updated）在链路①（SUCCESS 进度）**之前**到达前端，因为它在事务内就发送了。

---

## 七、关键设计要点

### 7.1 双通道并行

- **WebSocket**：细粒度实时进度（STARTED/WORKING/SUCCESS/FAILED + 百分比），只在消费期间推送
- **REST /api/tasks/**：粗粒度任务状态 (PENDING/STARTED/SUCCESS/FAILURE/REVOKED)，可随时查询历史

两者独立工作，WebSocket 断线不影响任务执行，前端仍可通过轮询 tasks API 获取最终状态。

### 7.2 Celery Signal 自动追踪

任务生命周期通过 5 个 Celery signal handler 自动同步到 DB，**业务代码无需手动更新 PaperlessTask**。[TRACKED_TASKS](src/documents/signals/handlers.py#L1005-L1017) 白名单控制哪些任务被追踪。

### 7.3 权限过滤与边界分析

#### 后端过滤逻辑

[StatusConsumer._can_view()](src/paperless/consumers.py#L23-L34)：

```python
async def _can_view(self, data: PermissionsData) -> bool:
    user = self.scope.get("user")
    if user is None: return False
    owner_id = data.get("owner_id")              # 可能是 None
    users_can_view = data.get("users_can_view", [])  # 默认 []
    groups_can_view = data.get("groups_can_view", []) # 默认 []

    if user.is_superuser or user.id == owner_id or user.id in users_can_view:
        return True
    return await user.groups.filter(pk__in=groups_can_view).aexists()
```

#### 前端过滤逻辑

[WebsocketStatusService.canViewMessage()](src-ui/src/app/services/websocket-status.service.ts#L208-L226)：

```typescript
return (
  !messageData.owner_id ||                            // owner_id 为空 → 放行
  user.is_superuser ||
  (messageData.owner_id && messageData.owner_id === user.id) ||
  (messageData.users_can_view && messageData.users_can_view.includes(user.id)) ||
  (messageData.groups_can_view && messageData.groups_can_view.some(...))
)
```

#### 边界场景：owner_id=None, users_can_view=[], groups_can_view=[]

这是无主文档（如消费目录/邮件获取）在未设置权限时的典型场景：

| | 后端 `_can_view` | 前端 `canViewMessage` |
|---|---|---|
| `owner_id=None` | `user.id == None` → False | `!null` → **True**（放行） |
| `users_can_view=[]` | `user.id in []` → False | `[] && [].includes()` → False（空数组为 truthy 但 includes 返回 False） |
| `groups_can_view=[]` | `.filter(pk__in=[])` → False | `[] && [].some(...)` → False |
| **结果** | **仅 superuser 可收到** | **所有用户可收到** |

**前后端行为不一致**：当 `owner_id=None` 时，后端仅允许 superuser 收到消息（因为 `user.id == None` 永远为 False，空列表也不匹配），但前端 `!messageData.owner_id` 为 True 直接放行所有用户。

**实际影响**：由于后端是第一道过滤，消息在后端就被拦截，前端不会收到这些消息，所以**不会导致越权泄露**。前端 `canViewMessage` 只是防御性 fallback，防止后端过滤遗漏。但在无主文档场景下，后端过度严格 —— 非 superuser 的合法消费者无法通过 WebSocket 收到自己上传的无主文档进度（不过 Web/API 上传时 `owner_id` 必有值，此边界仅影响消费目录/邮件等无主场景）。

### 7.4 消费流程 Plugin 化

`consume_file` 通过 Plugin 链顺序执行（ConsumerPreflightPlugin → AsnCheckPlugin → CollatePlugin → BarcodePlugin → AsnCheckPlugin → WorkflowTriggerPlugin → ConsumerPlugin），每个 Plugin 通过 `ProgressManager` 发送进度，实现了关注点分离。

### 7.5 新建文档缺失 document_updated 通知

新建文档消费成功后，`ConsumerPlugin.run()` 只发送 `status_update(SUCCESS)` 进度消息，**不发送** `document_updated` WebSocket 消息（因为 `document.root_document_id` 为 None，跳过了 `document_updated.send()` 调用）。前端文档列表的刷新依赖 `documentConsumptionFinishedSubject` 的订阅者处理，而非 `documentUpdatedSubject`。这是一个有意的设计选择：消费进度通知已经足以告知前端新文档创建成功。
