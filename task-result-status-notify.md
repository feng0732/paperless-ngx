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

所有后台任务定义在 [tasks.py](file:///d:/fz/0601/solo-dogfeeding/code/29-paperless-ngx/src/documents/tasks.py) 中，使用 `@shared_task` 装饰器：

| Task 函数 | PaperlessTask.TaskType | 说明 |
|---|---|---|
| `consume_file` | CONSUME_FILE | 核心：消费文档（OCR/解析/存储） |
| `train_classifier` | TRAIN_CLASSIFIER | 训练自动分类模型 |
| `sanity_check` | SANITY_CHECK | 系统健康检查 |
| `llmindex_index` | LLM_INDEX | LLM 索引构建 |
| `empty_trash` | EMPTY_TRASH | 清空回收站 |
| `check_scheduled_workflows` | CHECK_WORKLOWS | 定时工作流 |
| `bulk_update_documents` | BULK_UPDATE | 批量更新文档 |
| `update_document_content_maybe_archive_file` | REPROCESS_DOCUMENT | 重新处理文档 |
| `build_share_link_bundle` | BUILD_SHARE_LINK | 构建分享链接包 |

此外，[bulk_edit.py](file:///d:/fz/0601/solo-dogfeeding/code/29-paperless-ngx/src/documents/bulk_edit.py) 中 `delete` 对应 `BULK_DELETE`，邮件处理由 `paperless_mail.tasks.process_mail_accounts` 对应 `MAIL_FETCH`。

### 2.2 任务入队方式

以文档消费为例，[PostDocumentView](file:///d:/fz/0601/solo-dogfeeding/code/29-paperless-ngx/src/documents/views.py#L3093-L3160) 在 `post()` 中：

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

定义在 [models.py](file:///d:/fz/0601/solo-dogfeeding/code/29-paperless-ngx/src/documents/models.py#L701-L708)：

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

定义在 [models.py](file:///d:/fz/0601/solo-dogfeeding/code/29-paperless-ngx/src/documents/models.py#L664-L826)，核心字段：

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

定义在 [handlers.py](file:///d:/fz/0601/solo-dogfeeding/code/29-paperless-ngx/src/documents/signals/handlers.py#L1001-L1313)，这是连接 Celery 生命周期与 DB 持久化的核心桥梁：

| Celery Signal | Handler 函数 | 操作 |
|---|---|---|
| `before_task_publish` | `before_task_publish_handler` | 任务发布到 broker 时 **创建** PaperlessTask 记录 (status=PENDING)，提取 input_data、trigger_source、owner_id |
| `task_prerun` | `task_prerun_handler` | Worker 开始执行时更新 status=STARTED、date_started |
| `task_postrun` | `task_postrun_handler` | 任务正常完成时更新 status=SUCCESS、date_done、duration_seconds、result_data。若 result_data 含 `duplicate_of` 则改为 FAILURE |
| `task_failure` | `task_failure_handler` | 任务异常时更新 status=FAILURE、date_done、result_data (含 error_type/error_message/traceback) |
| `task_revoked` | `task_revoked_handler` | 任务被撤销时更新 status=REVOKED、date_done |

**追踪范围**：仅 `TRACKED_TASKS` 字典中列出的任务名才会被记录，其余 Celery task 静默忽略。

### 3.3 任务返回值 → result_data 的映射

`consume_file` 的返回值是 TypedDict：

- `ConsumeFileSuccessResult`: `{"document_id": int}` → 写入 result_data
- `ConsumeFileDuplicateResult`: `{"duplicate_of": int, "duplicate_in_trash": bool}` → 写入 result_data，且 status 被改为 FAILURE
- `ConsumeFileStoppedResult`: `{"reason": str}` → 写入 result_data

`task_postrun_handler` 中判断 `isinstance(retval, dict)` 时自动将返回值写入 `result_data`。

### 3.4 REST API 查询

[TasksViewSet](file:///d:/fz/0601/solo-dogfeeding/code/29-paperless-ngx/src/documents/views.py#L4055-L4244) 提供以下端点：

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
  │  权限过滤 (can_view)
  ▼
前端 WebSocket 连接
```

#### 4.1.2 三种消息类型

定义在 [helpers.py](file:///d:/fz/0601/solo-dogfeeding/code/29-paperless-ngx/src/documents/plugins/helpers.py#L15-L72)：

| type | Payload 结构 | 触发时机 |
|---|---|---|
| `status_update` | `{filename, task_id, current_progress, max_progress, status, message, document_id, owner_id, users_can_view, groups_can_view}` | 文档消费过程中每个阶段 |
| `documents_deleted` | `{documents: [id, ...]}` | 批量删除文档后 |
| `document_updated` | `{document_id, modified, owner_id, users_can_view, groups_can_view}` | 文档更新后（含 workflow 触发的更新） |

#### 4.1.3 进度管理器

[ProgressManager](file:///d:/fz/0601/solo-dogfeeding/code/29-paperless-ngx/src/documents/plugins/helpers.py#L119-L150) 是消费过程中发送实时进度的核心，作为 context manager 使用：

```python
with ProgressManager(filename, self.request.id) as status_mgr:
    # ... 在各 plugin 中通过 status_mgr.send_progress() 发进度
```

[DocumentsStatusManager](file:///d:/fz/0601\solo-dogfeeding\code\29-paperless-ngx\src\documents\plugins\helpers.py#L153-L182) 负责发送 `documents_deleted` 和 `document_updated` 事件。

#### 4.1.4 StatusConsumer (WebSocket 服务端)

定义在 [consumers.py](file:///d:/fz/0601/solo-dogfeeding/code/29-paperless-ngx/src/paperless/consumers.py#L18-L62)：

- 连接时验证认证、加入 `status_updates` channel group
- `status_update` 和 `document_updated` 事件会进行权限过滤 (`_can_view`)：超级用户/所有者/view 权限用户/group 权限用户可收到
- `documents_deleted` 不做权限过滤（所有连接的客户端都收到）

WebSocket 路由定义在 [urls.py](file:///d:/fz/0601/solo-dogfeeding/code/29-paperless-ngx/src/paperless/urls.py#L420-L421)：`ws/status/`

ASGI 入口在 [asgi.py](file:///d:/fz/0601/solo-dogfeeding/code/29-paperless-ngx/src/paperless/asgi.py#L18-L23)，使用 `AuthMiddlewareStack` 确保认证。

#### 4.1.5 进度阶段与消息

[ProgressStatusOptions](file:///d:/fz/0601/solo-dogfeeding/code/29-paperless-ngx/src/documents/plugins/helpers.py#L15-L19)：

| Phase | 含义 |
|---|---|
| STARTED | 任务开始（0%） |
| WORKING | 处理中（20%~95%） |
| SUCCESS | 成功（100%） |
| FAILED | 失败（100%） |

ConsumerPlugin 在消费流程各节点发送具体消息（[consumer.py](file:///d:/fz/0601/solo-dogfeeding/code/29-paperless-ngx/src/documents/consumer.py#L103-L121)）：

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

[signals/__init__.py](file:///d:/fz/0601/solo-dogfeeding/code/29-paperless-ngx/src/documents/signals/__init__.py)：

| Signal | 发送时机 |
|---|---|
| `document_consumption_started` | 文档消费开始解析前 |
| `document_consumption_finished` | 文档消费完成（DB 事务内） |
| `document_updated` | 文档被更新（版本新增/Workflow 修改后） |

#### 4.2.2 Signal 连接 (apps.py)

[apps.py](file:///d:/fz/0601/solo-dogfeeding/code/29-paperless-ngx/src/documents/apps.py#L24-L33) 在 `DocumentsConfig.ready()` 中注册：

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

#### 4.2.3 `send_websocket_document_updated` 的调用路径

1. **消费流程**：`ConsumerPlugin.run()` 完成后，若为新版本，手动调用 `document_updated.send()`（[consumer.py](file:///d:/fz/0601/solo-dogfeeding/code/29-paperless-ngx/src/documents/consumer.py#L732-L735)）
2. **Workflow 执行后**：`document_updated` signal 触发 `run_workflows_updated` → workflow 修改文档 → `document.save()` → 但 `send_websocket_document_updated` 需要额外触发
3. **定时工作流**：`check_scheduled_workflows` 中直接调用 `send_websocket_document_updated()`（[tasks.py](file:///d:/fz/0601/solo-dogfeeding/code/29-paperless-ngx/src/documents/tasks.py#L569-L573)）
4. **批量更新**：`bulk_update_documents` 调用 `document_updated.send()`（[tasks.py](file:///d:/fz/0601/solo-dogfeeding/code/29-paperless-ngx/src/documents/tasks.py#L260-L264)）

#### 4.2.4 文档删除通知

[bulk_edit.py](file:///d:/fz/0601/solo-dogfeeding/code/29-paperless-ngx/src/documents/bulk_edit.py#L383-L384) 中批量删除后：

```python
status_mgr = DocumentsStatusManager()
status_mgr.send_documents_deleted(delete_ids)
```

---

## 五、前端感知与状态更新

### 5.1 两大服务并行工作

| 服务 | 通道 | 用途 |
|---|---|---|
| `WebsocketStatusService` | WebSocket `ws/status/` | 实时进度条、文档消费状态 |
| `TasksService` | HTTP REST `/api/tasks/` | 任务列表查询、确认、手动触发 |

### 5.2 上传流程 (UploadDocumentsService)

[upload-documents.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/29-paperless-ngx/src-ui/src/app/services/upload-documents.service.ts)：

```
1. newFileUpload(filename) → 创建 FileStatus (phase=STARTED)
2. HTTP POST /api/documents/post_document/
   ├─ UploadProgress → FileStatusPhase.UPLOADING (0~20%)
   └─ Response → 拿到 task_id，phase 仍在 UPLOADING
3. 此后由 WebSocket 接管进度更新
```

### 5.3 WebSocket 进度接收 (WebsocketStatusService)

[websocket-status.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/29-paperless-ngx/src-ui/src/app/services/websocket-status.service.ts)：

**连接**：`connect()` 建立 WebSocket 到 `ws/status/`

**消息分发**（`onmessage`）：

```
switch (type):
  case STATUS_UPDATE:
    handleProgressUpdate(data)
    → 更新 FileStatus (phase/progress/message/documentId)
    → 根据新 phase 发出 Subject 通知:
       STARTED  → documentDetectedSubject
       SUCCESS  → documentConsumptionFinishedSubject
       FAILED   → documentConsumptionFailedSubject

  case DOCUMENT_UPDATED:
    handleDocumentUpdated(data)
    → documentUpdatedSubject.next(data)

  case DOCUMENTS_DELETED:
    documentDeletedSubject.next(true)
```

**进度计算**（[FileStatus.getProgress()](file:///d:/fz/0601/solo-dogfeeding/code/29-paperless-ngx/src-ui/src/app/services/websocket-status.service.ts#L61-L75)）：

| Phase | 进度 |
|---|---|
| STARTED | 0.0 |
| UPLOADING | progress/max × 0.2 (0~20%) |
| WORKING | progress/max × 0.8 + 0.2 (20~100%) |
| SUCCESS/FAILED | 1.0 (100%) |

**权限过滤**：前端也做了 `canViewMessage()` 二次过滤（作为后端过滤的 fallback）。

### 5.4 任务列表管理 (TasksService)

[tasks.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/29-paperless-ngx/src-ui/src/app/services/tasks.service.ts)：

- `reload()`: GET `/api/tasks/?acknowledged=false&page_size=1000` 加载未确认的任务
- `list(page, pageSize, extraParams)`: 分页查询
- `dismissTasks(task_ids)`: POST `/api/tasks/acknowledge/` 批量确认
- `run(taskType)`: POST `/api/tasks/run/` 手动触发任务

### 5.5 前端数据模型

[PaperlessTask](file:///d:/fz/0601/solo-dogfeeding/code/29-paperless-ngx/src-ui/src/app/data/paperless-task.ts) — 与后端 TaskSerializerV10 一一对应：

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

[WebsocketProgressMessage](file:///d:/fz/0601/solo-dogfeeding/code/29-paperless-ngx/src-ui/src/app/data/websocket-progress-message.ts)：

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
 │                            │                         │   document.save()   │                             │
 │                            │                         │   document_consumption_finished.send()             │
 │                            │                         │     → add_inbox_tags, set_correspondent, ...       │
 │                            │                         │     → run_workflows_added → workflow actions       │
 │                            │                         │     → add_to_index, add_or_update_document_in_llm │
 │                            │                         │                      │                             │
 │                            │                         │   send_progress(100,100,SUCCESS) → channel_layer ─→│
 │←─────────────────────────────────────────────────────────────────────────────────── status_update ─────│
 │  FileStatus.phase=SUCCESS  │                         │                      │                             │
 │                            │                         │                      │                             │
 │                            │                         │   [task_postrun]     │                             │
 │                            │                         │   PaperlessTask      │                             │
 │                            │                         │   status=SUCCESS     │                             │
 │                            │                         │   result_data={document_id: N}                     │
 │                            │                         │                      │                             │
 │                            │                         │   [document_updated signal → send_websocket_document_updated]
 │                            │                         │                      │                             │
 │←───────────────────────────────────────────────────────────────────────────── document_updated ──────────│
 │  documentUpdatedSubject    │                         │                      │                             │
 │                            │                         │                      │                             │
 │─ GET /api/tasks/ ─────────→│                         │                      │                             │
 │←── PaperlessTask list ────│                         │                      │                             │
```

---

## 七、关键设计要点

### 7.1 双通道并行

- **WebSocket**：细粒度实时进度（STARTED/WORKING/SUCCESS/FAILED + 百分比），只在消费期间推送
- **REST /api/tasks/**：粗粒度任务状态 (PENDING/STARTED/SUCCESS/FAILURE/REVOKED)，可随时查询历史

两者独立工作，WebSocket 断线不影响任务执行，前端仍可通过轮询 tasks API 获取最终状态。

### 7.2 Celery Signal 自动追踪

任务生命周期通过 5 个 Celery signal handler 自动同步到 DB，**业务代码无需手动更新 PaperlessTask**。`TRACKED_TASKS` 白名单控制哪些任务被追踪。

### 7.3 权限过滤

- **后端**：`StatusConsumer._can_view()` 在推送前根据 owner_id / users_can_view / groups_can_view 过滤
- **前端**：`WebsocketStatusService.canViewMessage()` 做二次过滤（防御性编程）
- **REST API**：`TasksViewSet.get_queryset()` 按用户权限过滤查询结果

### 7.4 消费流程 Plugin 化

`consume_file` 通过 Plugin 链顺序执行（ConsumerPreflightPlugin → AsnCheckPlugin → CollatePlugin → BarcodePlugin → AsnCheckPlugin → WorkflowTriggerPlugin → ConsumerPlugin），每个 Plugin 通过 `ProgressManager` 发送进度，实现了关注点分离。
