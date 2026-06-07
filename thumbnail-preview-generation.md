# Paperless-ngx 缩略图与预览生成：代码证据与链路详解

> 代码引用规范：每个证据点标注三层信息，全部使用仓库相对路径，不包含任何本机绝对路径。
> - 仓库相对路径：从项目根目录出发的文件路径
> - 稳定位置：函数名 / 类名 / 变量名（抗行号漂移的语义锚点）
> - 行号范围：辅助定位（基于当前仓库快照）

---

## 一、核心文件索引（仓库相对路径）

| 层级 | 仓库相对路径 | 角色 |
|------|-------------|------|
| 后端消费核心 | `src/documents/consumer.py` | 调用解析器生成缩略图、失败处理、WebSocket 进度推送 |
| 后端 API 视图 | `src/documents/views.py` | `/preview/` 与 `/thumb/` 端点、HTTP 缓存装饰器、`serve_file()` |
| 后端缓存层 | `src/documents/caching.py` | 缩略图修改时间缓存 key、TTL 常量、缓存失效入口 |
| 后端缓存条件 | `src/documents/conditionals.py` | HTTP 协商缓存的 ETag / Last-Modified 计算函数 |
| 后端模型 | `src/documents/models.py` | `Document.thumbnail_path`、`PaperlessTask` 状态记录 |
| 后端异步任务 | `src/documents/tasks.py` | `update_document_content_maybe_archive_file`（重处理时重生成缩略图） |
| 后端批量编辑 | `src/documents/bulk_edit.py` | `reprocess()` 函数，驱动前端"Reprocess"按钮 |
| 后端管理命令 | `src/documents/management/commands/document_thumbnails.py` | `document_thumbnails` CLI：批量重建缩略图 |
| 后端命令基类 | `src/documents/management/commands/base.py` | `PaperlessCommand`：多进程 + 进度条基础设施 |
| 后端进度推送 | `src/documents/plugins/helpers.py` | `ProgressManager`、`ProgressStatusOptions`、`BaseStatusManager._fail()`、`group_send("status_updates")` |
| 后端信号处理 | `src/documents/signals/handlers.py` | Celery 信号：`before_task_publish`、`task_prerun`、`task_postrun`、`task_failure`、`task_revoked`，驱动 `PaperlessTask` 全生命周期 |
| 后端 Celery 初始化 | `src/paperless/celery.py` | Celery App 初始化、signed-pickle 序列化、worker_process_init 钩子 |
| 前端根组件 | `src-ui/src/app/app.component.ts` | `AppComponent.failedSubscription` / `successSubscription` 订阅消费结果、触发 Toast |
| 前端 Toast 服务 | `src-ui/src/app/services/toast.service.ts` | `ToastService.showError()` / `show()` 分发 Toast 消息 |
| 前端 Toast 组件 | `src-ui/src/app/components/common/toast/toast.component.ts` | Toast 渲染、错误详情展示、复制剪贴板 |
| 前端 Toast 样式 | `src-ui/src/app/components/common/toast/toast.component.scss` | `.toast.error` 红色错误样式 |
| 前端 REST 服务 | `src-ui/src/app/services/rest/document.service.ts` | `getThumbUrl()`、`getPreviewUrl()`、`reprocessDocuments()` |
| 前端 WebSocket 状态 | `src-ui/src/app/services/websocket-status.service.ts` | `FileStatus`、`FILE_STATUS_MESSAGES`、`documentConsumptionFailedSubject`、进度换算 |
| 前端预览弹窗 | `src-ui/src/app/components/common/preview-popup/preview-popup.component.ts` | 列表页悬停预览 + `onError()` 处理 |
| 前端预览弹窗模板 | `src-ui/src/app/components/common/preview-popup/preview-popup.component.html` | 错误态、密码锁、PDF Viewer 渲染 |
| 前端文档详情 | `src-ui/src/app/components/document-detail/document-detail.component.ts` | `reprocess()`、`onError()`、`pdfPreviewLoaded()`、`tiffError`、`previewText` |
| 前端文档详情模板 | `src-ui/src/app/components/document-detail/document-detail.component.html` | `#previewContent` 模板、TIFF 错误、密码输入、缩略图覆盖层 |
| 前端文档卡片 | `src-ui/src/app/components/document-list/document-card-small/document-card-small.component.html` | 列表缩略图 `<img>` 渲染 |
| 前端文档卡片 | `src-ui/src/app/components/document-list/document-card-large/document-card-large.component.html` | 列表缩略图 `<img>` 渲染 |
| 前端任务类型 | `src-ui/src/app/data/paperless-task.ts` | `PaperlessTaskType.ReprocessDocument` 等枚举 |

---

## 二、缩略图生成链路（带代码证据位置）

### 2.1 主路径：消费流程中生成

**调用入口** `ConsumerPlugin.run()` 方法内

- 仓库相对路径：`src/documents/consumer.py`
- 稳定位置：`ConsumerPlugin.run()` → `document_parser.get_thumbnail()` 调用点
- 行号范围：L526-L536

```python
# ConsumerPlugin.run() 内部
self.log.debug(f"Generating thumbnail for {self.filename}...")
self._send_progress(
    70,
    100,
    ProgressStatusOptions.WORKING,
    ConsumerStatusShortMessage.GENERATING_THUMBNAIL,  # 值 = "generating_thumbnail"
)
thumbnail = document_parser.get_thumbnail(
    self.working_copy,
    mime_type,
)
```

**状态枚举定义**

- 仓库相对路径：`src/documents/consumer.py`
- 稳定位置：`ConsumerStatusShortMessage.GENERATING_THUMBNAIL`
- 行号范围：L113-L122

### 2.2 存储路径计算

- 仓库相对路径：`src/documents/models.py`
- 稳定位置：`Document.thumbnail_path` 属性
- 行号范围：L479-L484

```python
@property
def thumbnail_path(self) -> Path:
    webp_file_name = f"{self.pk:07}.webp"
    webp_file_path = settings.THUMBNAIL_DIR / Path(webp_file_name)
    return webp_file_path.resolve()
```

规则：固定 WebP 格式，7 位零填充 ID，如文档 123 → `0000123.webp`。

### 2.3 写入磁盘

- 仓库相对路径：`src/documents/consumer.py`
- 稳定位置：`ConsumerPlugin._write()` 调用点，`FileLock(settings.MEDIA_LOCK)` 块内
- 行号范围：L689-L700

```python
with FileLock(settings.MEDIA_LOCK):
    # ... 原始文件写入 ...
    self._write(thumbnail, document.thumbnail_path)
    # ... 归档文件写入 ...
```

### 2.4 重处理任务中重生成

- 仓库相对路径：`src/documents/tasks.py`
- 稳定位置：`update_document_content_maybe_archive_file()` 函数
- 行号范围：L316-L376

```python
thumbnail = parser.get_thumbnail(document.source_path, mime_type)
# ...
with FileLock(settings.MEDIA_LOCK):
    shutil.move(thumbnail, document.thumbnail_path)
```

---

## 三、缩略图生成失败后的用户可见状态（完整端到端链路）

本章节梳理缩略图生成异常时，从 Celery 任务发布 → 后端 WebSocket 推送 → 前端 Toast 显示 → PaperlessTask 失败入库的完整链路。

```
                                  ┌─────────────────────────────────────────────┐
                                  │  用户上传/触发 consume_file                  │
                                  └──────────────────────┬──────────────────────┘
                                                         │
                                            ┌────────────▼────────────┐
                                            │ Celery Broker           │
                                            │ before_task_publish     │
                                            │ → PaperlessTask(PENDING)│
                                            └────────────┬────────────┘
                                                         │
                                            ┌────────────▼────────────┐
                                            │ Celery Worker           │
                                            │ task_prerun             │
                                            │ → PaperlessTask(STARTED)│
                                            └────────────┬────────────┘
                                                         │
                                         ┌───────────────▼───────────────┐
                                         │ ConsumerPlugin.run()          │
                                         │   parser.get_thumbnail() 异常 │
                                         │   except Exception → _fail()  │
                                         └───────┬───────────────────┬───┘
                                                 │                   │
                              ┌──────────────────▼──┐        ┌───────▼──────────────────┐
                              │ ProgressManager       │        │ task_failure signal       │
                              │ send_progress(        │        │ → PaperlessTask(FAILURE)  │
                              │   100/100 FAILED)     │        │   result_data(error_type, │
                              │ group_send("status   │        │   error_message, tb)      │
                              │   _updates")          │        └──────────────────────────┘
                              └──────────┬────────────┘
                                         │ WebSocket
                                         ▼
                              ┌──────────────────────────────────┐
                              │ WebsocketStatusService           │
                              │ handleProgressUpdate()            │
                              │ status.phase = FAILED            │
                              │ documentConsumptionFailedSubject  │
                              │          .next(status)            │
                              └──────────┬───────────────────────┘
                                         │ subscribe
                                         ▼
                              ┌──────────────────────────────────┐
                              │ AppComponent                      │
                              │ failedSubscription                │
                              │ → toastService.showError(...)     │
                              └──────────┬───────────────────────┘
                                         │
                                         ▼
                              ┌──────────────────────────────────┐
                              │ ToastComponent (.toast.error)     │
                              │ 红色背景 + 错误消息 + 10s 自动消失 │
                              └──────────────────────────────────┘
```

### 3.1 阶段一：任务发布 → PaperlessTask(PENDING)

Celery 任务发布到 Broker 前，`before_task_publish` 信号创建 PaperlessTask 记录。

- 仓库相对路径：`src/documents/signals/handlers.py`
- 稳定位置：`TRACKED_TASKS` 字典、`before_task_publish_handler()` 函数
- 行号范围：L1005-L1017（TRACKED_TASKS）、L1103-L1141（handler）

```python
TRACKED_TASKS: dict[str, PaperlessTask.TaskType] = {
    "documents.tasks.consume_file": PaperlessTask.TaskType.CONSUME_FILE,
    "documents.tasks.update_document_content_maybe_archive_file": PaperlessTask.TaskType.REPROCESS_DOCUMENT,
    # ... 其他任务
}

@before_task_publish.connect
def before_task_publish_handler(sender=None, headers=None, body=None, **kwargs):
    task_name = headers.get("task", "")
    task_type = TRACKED_TASKS.get(task_name)
    if task_type is None:
        return
    _, task_kwargs, _ = body
    task_id = headers["id"]
    input_data = _extract_input_data(task_type, task_kwargs)   # 含 filename, mime_type
    trigger_source = _determine_trigger_source(headers)
    owner_id = _extract_owner_id(task_type, task_kwargs)
    PaperlessTask.objects.create(
        task_id=task_id,
        task_type=task_type,
        trigger_source=trigger_source,
        status=PaperlessTask.Status.PENDING,    # 初始状态
        input_data=input_data,
        owner_id=owner_id,
    )
```

- 仓库相对路径：`src/paperless/celery.py`
- 稳定位置：Celery App 初始化（`app = Celery("paperless")`）、信号自动发现（`app.autodiscover_tasks()`）
- 行号范围：L57-L66

### 3.2 阶段二：Worker 执行 → PaperlessTask(STARTED)

Worker 取到任务开始执行时，`task_prerun` 信号更新状态为 STARTED。

- 仓库相对路径：`src/documents/signals/handlers.py`
- 稳定位置：`task_prerun_handler()` 函数
- 行号范围：L1144-L1162

```python
@task_prerun.connect
def task_prerun_handler(sender=None, task_id=None, task=None, **kwargs):
    close_old_connections()
    PaperlessTask.objects.filter(task_id=task_id).update(
        status=PaperlessTask.Status.STARTED,
        date_started=timezone.now(),
    )
```

### 3.3 阶段三：缩略图生成异常 → `_fail()` 触发

缩略图生成异常被 `run()` 的最外层 `try/except` 捕获，调用 `_fail()` 方法。

- 仓库相对路径：`src/documents/consumer.py`
- 稳定位置：`ConsumerPlugin.run()` 的 `except ParseError` / `except Exception` 分支
- 行号范围：L555-L568

```python
except ParseError as e:
    self._fail(str(e), "...", exc_info=True, exception=e)
except Exception as e:
    self._fail(str(e), "...", exc_info=True, exception=e)
```

### 3.4 阶段四：`_fail()` 三副作用 + WebSocket 推送

- 仓库相对路径：`src/documents/plugins/helpers.py`
- 稳定位置：`BaseStatusManager._fail()` 方法
- 行号范围：L82-L105

`_fail()` 产生三个副作用：
1. **WebSocket 推送**：调用 `send_progress()` → `100/100 FAILED` + 错误消息
2. **错误日志**：`logger.exception(...)` 写入日志文件
3. **抛出 ConsumerError**：终止 Celery 任务，触发 `task_failure` 信号

**WebSocket 推送的具体实现：**

- 仓库相对路径：`src/documents/plugins/helpers.py`
- 稳定位置：`ProgressManager.send_progress()` → `BaseStatusManager.send()` → `self._channel.group_send("status_updates", payload)`
- 行号范围：L107-L116（send）、L125-L150（send_progress）

```python
def send(self, payload: WebsocketPayload) -> None:
    self.open()
    async_to_sync(self._channel.group_send)("status_updates", payload)

def send_progress(self, status, message, current_progress, max_progress, *, document_id=None, ...):
    data: ProgressUpdateData = {
        "filename": self.filename,
        "task_id": self.task_id,
        "current_progress": current_progress,  # 100
        "max_progress": max_progress,          # 100
        "status": status,                      # "FAILED"
        "message": message,                    # 错误消息字符串
        "document_id": document_id,
        "owner_id": owner_id,
        "users_can_view": users_can_view or [],
        "groups_can_view": groups_can_view or [],
    }
    payload: StatusUpdatePayload = {"type": "status_update", "data": data}
    self.send(payload)
```

### 3.5 阶段五：前端 WebSocket 接收 → Subject 分发

- 仓库相对路径：`src-ui/src/app/services/websocket-status.service.ts`
- 稳定位置：`connect()` 中 `onmessage` 回调 → `handleProgressUpdate()` → `case FileStatusPhase.FAILED` → `documentConsumptionFailedSubject.next(status)`
- 行号范围：L163-L206（connect/onmessage）、L228-L269（handleProgressUpdate）、L318-L324（onDocumentConsumptionFailed）

```typescript
// connect() 建立 WebSocket 连接
this.statusWebSocket = new WebSocket(`${environment.webSocketProtocol}//.../status/`)
this.statusWebSocket.onmessage = (ev: MessageEvent) => {
  const { type, data: messageData } = JSON.parse(ev.data)
  switch (type) {
    case WebsocketStatusType.STATUS_UPDATE:
      this.handleProgressUpdate(messageData as WebsocketProgressMessage)
      break
  }
}

// handleProgressUpdate() 分发状态
handleProgressUpdate(messageData: WebsocketProgressMessage) {
  let status = this.get(messageData.task_id, messageData.filename).status
  status.updateProgress(FileStatusPhase.WORKING, messageData.current_progress, messageData.max_progress)
  if (messageData.status in FileStatusPhase) {
    status.phase = FileStatusPhase[messageData.status]  // "FAILED" → FileStatusPhase.FAILED
  }
  switch (status.phase) {
    case FileStatusPhase.FAILED:
      this.documentConsumptionFailedSubject.next(status)  // 发布失败事件
      break
  }
}

// 对外暴露订阅接口
onDocumentConsumptionFailed() {
  return this.documentConsumptionFailedSubject
}
```

**进度换算**
- 仓库相对路径：`src-ui/src/app/services/websocket-status.service.ts`
- 稳定位置：`FileStatus.getProgress()` 中 `case FileStatusPhase.FAILED: return 1.0`
- 行号范围：L61-L78

### 3.6 阶段六：AppComponent 订阅 → ToastService.showError

- 仓库相对路径：`src-ui/src/app/app.component.ts`
- 稳定位置：`AppComponent.ngOnInit()` 中 `failedSubscription`、`successSubscription`、`newDocumentSubscription`
- 行号范围：L76-L136（ngOnInit）、L38-L41（subscription 声明）、L51-L61（ngOnDestroy 取消订阅）

```typescript
export class AppComponent implements OnInit, OnDestroy {
  newDocumentSubscription: Subscription
  successSubscription: Subscription
  failedSubscription: Subscription     // 失败订阅声明

  ngOnInit(): void {
    this.websocketStatusService.connect()

    // 成功订阅
    this.successSubscription = this.websocketStatusService
      .onDocumentConsumptionFinished()
      .subscribe((status) => {
        this.tasksService.reload()
        if (this.showNotification(SETTINGS_KEYS.NOTIFICATIONS_CONSUMER_SUCCESS)) {
          this.toastService.show({
            content: $localize`Document ${status.filename} was added to Paperless-ngx.`,
            delay: 10000,
            actionName: $localize`Open document`,
            action: () => { this.router.navigate(['documents', status.documentId]) },
          })
        }
      })

    // ── 失败订阅（核心） ──────────────────────────────────────
    this.failedSubscription = this.websocketStatusService
      .onDocumentConsumptionFailed()
      .subscribe((status) => {
        this.tasksService.reload()
        if (this.showNotification(SETTINGS_KEYS.NOTIFICATIONS_CONSUMER_FAILED)) {
          this.toastService.showError(
            $localize`Could not add ${status.filename}\: ${status.message}`
          )
        }
      })
    // ─────────────────────────────────────────────────────────

    // 新文档检测订阅
    this.newDocumentSubscription = this.websocketStatusService
      .onDocumentDetected()
      .subscribe((status) => {
        this.tasksService.reload()
        if (this.showNotification(SETTINGS_KEYS.NOTIFICATIONS_CONSUMER_NEW_DOCUMENT)) {
          this.toastService.show({
            content: $localize`Document ${status.filename} is being processed...`,
            delay: 5000,
          })
        }
      })
  }

  ngOnDestroy(): void {
    this.websocketStatusService.disconnect()
    if (this.successSubscription) this.successSubscription.unsubscribe()
    if (this.failedSubscription)  this.failedSubscription.unsubscribe()
    if (this.newDocumentSubscription) this.newDocumentSubscription.unsubscribe()
  }

  // 仪表板页可配置是否抑制通知
  private showNotification(key) {
    if (this.router.url == '/dashboard' &&
        this.settings.get(SETTINGS_KEYS.NOTIFICATIONS_CONSUMER_SUPPRESS_ON_DASHBOARD)) {
      return false
    }
    return this.settings.get(key)
  }
}
```

### 3.7 阶段七：ToastService → ToastsComponent → ToastComponent 完整容器层链路

本阶段梳理 Toast 从触发到屏幕渲染的完整数据流：
```
AppComponent.failedSubscription
    │
    ▼
toastService.showError(content)           ← 3.7.1  错误 Toast 入口
    │  classname: 'error', delay: 10000
    ▼
toastService.show(toast)                  ← 3.7.2  Toast 对象入列
    │  this.showToast.next(toast)
    ▼
ToastsComponent (pngx-toasts)            ← 3.7.3  容器订阅
    │  ngOnInit: showToast.subscribe → this.toasts = [toast]
    │  template: <pngx-toast [toast]="toast" [autohide]="true">
    ▼
ToastComponent (pngx-toast)              ← 3.7.4  单个 Toast 渲染
    │  <ngb-toast [autohide]="autohide"
    │             [delay]="toast.delay"
    │             [class]="toast.classname">   ← 'error' class 从这里传入
    │
    ├─ 图标：exclamation-triangle (toast.error 为真)
    ├─ 内容：{{ toast.content }}
    ├─ 错误详情：<details> + 复制剪贴板按钮
    └─ 进度条：ngb-progressbar 显示 delayRemaining
    │
    ▼
toast.component.scss                     ← 3.7.5  错误样式
    ::ng-deep .toast.error {
        border-color: hsla(350, 79%, 40%, 0.4);
    }
    ::ng-deep .toast.error .toast-body {
        background-color: hsla(350, 79%, 40%, 0.8);
    }
```

#### 3.7.1 ToastService.showError() — 错误 Toast 入口

- 仓库相对路径：`src-ui/src/app/services/toast.service.ts`
- 稳定位置：`Toast` 接口、`ToastService.showError()` 方法
- 行号范围：L1-L21（Toast 接口）、L57-L64（showError）

```typescript
export interface Toast {
  id?: string
  content: string
  delay: number
  delayRemaining?: number
  action?: any
  actionName?: string
  classname?: string   // ← 传入 'error' 控制红色样式
  error?: any
}

showError(content: string, error: any = null, delay: number = 10000) {
  this.show({
    content: content,
    delay: delay,          // 默认 10 秒
    classname: 'error',    // ← 关键：将 class 注入 Toast 对象
    error,
  })
}
```

#### 3.7.2 ToastService.show() — 推入 Subject

- 仓库相对路径：`src-ui/src/app/services/toast.service.ts`
- 稳定位置：`ToastService.showToast` Subject、`ToastService.show()` 方法
- 行号范围：L37-L39（Subject 声明）、L41-L55（show 方法）

```typescript
export class ToastService {
  private toasts: Toast[] = []
  private toastsSubject: Subject<Toast[]> = new Subject()
  public showToast: Subject<Toast> = new Subject()  // ← 容器订阅此 Subject

  show(toast: Toast) {
    if (!toast.id) {
      toast.id = uuidv4()
    }
    if (typeof toast.error === 'string') {
      try {
        toast.error = JSON.parse(toast.error)
      } catch (e) {}
    }
    this.toasts.unshift(toast)
    if (!this._suppressPopupToasts) {
      this.showToast.next(toast)   // ← 推送给容器层
    }
    this.toastsSubject.next(this.toasts)
  }
}
```

#### 3.7.3 AppComponent 挂载点 → ToastsComponent 容器

**AppComponent 模板中注册全局容器：**

- 仓库相对路径：`src-ui/src/app/app.component.html`
- 稳定位置：`<pngx-toasts></pngx-toasts>`（全局唯一 Toast 容器）
- 行号：L1

```html
<pngx-toasts></pngx-toasts>     <!-- 全局 Toast 容器 -->
<pngx-file-drop>
  <ng-container content>
    <router-outlet></router-outlet>
  </ng-container>
</pngx-file-drop>
```

**ToastsComponent 订阅 showToast Subject：**

- 仓库相对路径：`src-ui/src/app/components/common/toasts/toasts.component.ts`
- 稳定位置：`ToastsComponent.ngOnInit()` 中 `toastService.showToast.subscribe(...)`
- 行号范围：L33-L37（订阅）、L39-L42（closeToast）

```typescript
export class ToastsComponent implements OnInit, OnDestroy {
  toastService = inject(ToastService)
  private subscription: Subscription
  public toasts: Toast[] = []   // 数组驱动变更检测

  ngOnInit(): void {
    this.subscription = this.toastService.showToast.subscribe((toast) => {
      this.toasts = toast ? [toast] : []   // 只保留最新一条
    })
  }

  closeToast() {
    this.toastService.closeToast(this.toasts[0])
    this.toasts = []
  }
}
```

**ToastsComponent 模板向子组件传递输入：**

- 仓库相对路径：`src-ui/src/app/components/common/toasts/toasts.component.html`
- 稳定位置：`<pngx-toast [toast]="toast" [autohide]="true" (closed)="closeToast()">`
- 行号范围：L1-L3

```html
@for (toast of toasts; track toast.id) {
  <pngx-toast [toast]="toast" [autohide]="true" (closed)="closeToast()"></pngx-toast>
}
```

关键输入：
- `[toast]="toast"`：传递完整 Toast 对象（含 content、delay、classname='error'、error）
- `[autohide]="true"`：开启自动消失

#### 3.7.4 ToastComponent — 单个 Toast 渲染

**组件接收输入：**

- 仓库相对路径：`src-ui/src/app/components/common/toast/toast.component.ts`
- 稳定位置：`@Input() toast`、`@Input() autohide`
- 行号范围：L26-L32（@Input 声明）

```typescript
export class ToastComponent {
  @Input() toast: Toast
  @Input() autohide: boolean = true
  @Output() hidden: EventEmitter<Toast> = new EventEmitter<Toast>()
  @Output() closed: EventEmitter<Toast> = new EventEmitter<Toast>()
}
```

**模板传递到 ngb-toast（bootstrap-ng 的 Toast 组件）：**

- 仓库相对路径：`src-ui/src/app/components/common/toast/toast.component.html`
- 稳定位置：`<ngb-toast [autohide]="autohide" [delay]="toast.delay" [class]="toast.classname">`
- 行号范围：L1-L7

```html
<ngb-toast
    [autohide]="autohide"           <!-- 来自 ToastsComponent 传入的 true -->
    [delay]="toast.delay"           <!-- 来自 Toast.delay（10000ms） -->
    [class]="toast.classname"       <!-- 来自 Toast.classname（'error'）→ 控制红色样式 -->
    [class.mb-2]="true"
    (shown)="onShown(toast)"
    (hidden)="hidden.emit(toast)">
```

**内部渲染细节：**

- 仓库相对路径：`src-ui/src/app/components/common/toast/toast.component.html`
- 稳定位置：图标分支、错误详情 `<details>`、复制按钮、进度条
- 行号范围：L8-L55

```html
<!-- 自动消失进度条 -->
@if (autohide) {
    <ngb-progressbar ... type="dark"
        [max]="toast.delay"
        [value]="toast.delayRemaining"></ngb-progressbar>
}

<div class="d-flex align-items-top">
    <!-- 图标分支 -->
    @if (!toast.error) {
        <i-bs ... name="info-circle"></i-bs>            <!-- 普通信息 -->
    }
    @if (toast.error) {
        <i-bs ... name="exclamation-triangle"></i-bs>   <!-- 错误警告三角 -->
    }

    <div>
        <!-- 主内容 -->
        <p class="ms-2 mb-0 text-break">{{toast.content}}</p>

        <!-- 错误详情折叠区（仅 toast.error 为真时显示） -->
        @if (toast.error) {
        <details class="ms-2">
            <div class="mt-2 ms-n4 me-n2 small">
            @if (isDetailedError(toast.error)) {
                <dl class="row mb-0">
                    <dt>URL</dt>      <dd>{{ toast.error.url }}</dd>
                    <dt>Status</dt>   <dd>{{ toast.error.status }} <em>{{ toast.error.statusText }}</em></dd>
                    <dt>Error</dt>    <dd>{{ getErrorText(toast.error) }}</dd>
                </dl>
            }
            <!-- 复制原始错误按钮 -->
            <button (click)="copyError(toast.error)">
                <i-bs name="clipboard"></i-bs> Copy Raw Error
            </button>
            </div>
        </details>
        }

        <!-- 操作按钮（如"Open document"） -->
        @if (toast.action) {
            <button (click)="closed.emit(toast); toast.action()">{{toast.actionName}}</button>
        }
    </div>

    <button type="button" class="btn-close ..." (click)="closed.emit(toast);"></button>
</div>
```

**错误工具方法：**

- 仓库相对路径：`src-ui/src/app/components/common/toast/toast.component.ts`
- 稳定位置：`isDetailedError()`、`getErrorText()`（截断 200 字符）、`copyError()`（复制剪贴板）
- 行号范围：L52-L75

```typescript
public isDetailedError(error: any): boolean {
  return (
    typeof error === 'object' &&
    'status' in error && 'statusText' in error &&
    'url' in error && 'message' in error && 'error' in error
  )
}
getErrorText(error: any) {
  let text: string = error.error?.detail ?? error.error ?? ''
  if (typeof text === 'object') text = JSON.stringify(text)
  return `${text.slice(0, 200)}${text.length > 200 ? '...' : ''}`   // 截断 200 字符
}
copyError(error: any) {
  this.clipboard.copy(JSON.stringify(error))
}
```

#### 3.7.5 错误样式（CSS）

- 仓库相对路径：`src-ui/src/app/components/common/toast/toast.component.scss`
- 稳定位置：`::ng-deep .toast.error` 红色错误样式规则
- 行号范围：L5-L15

```scss
::ng-deep .toast.error {
    border-color: hsla(350, 79%, 40%, 0.4);       // 红色边框（半透明）
}
::ng-deep .toast.error .toast-body {
    background-color: hsla(350, 79%, 40%, 0.8);   // 红色半透明背景
    border-top-left-radius: inherit;
    border-top-right-radius: inherit;
    border-bottom-left-radius: inherit;
    border-bottom-right-radius: inherit;
}
.progress {
    background-color: var(--pngx-primary);
    opacity: .07;
}
```

**用户可见表现**：页面右下角弹出红色 Toast（红框 + 红色半透明背景 + 黄色警告三角图标），显示 "Could not add <文件名>: <错误消息>"，10 秒内自动消失，底部进度条动态显示剩余时间；若附带 error 对象，可展开 `<details>` 查看 URL / Status / Error 详情并一键复制原始错误到剪贴板。

### 3.8 阶段八：Celery 失败入库 → PaperlessTask(FAILURE)

ConsumerError 抛出后，Celery 的 `task_failure` 信号触发，更新 PaperlessTask 记录。

- 仓库相对路径：`src/documents/signals/handlers.py`
- 稳定位置：`task_failure_handler()` 函数
- 行号范围：L1226-L1278

```python
@task_failure.connect
def task_failure_handler(sender=None, task_id=None, exception=None, traceback=None, **kwargs):
    close_old_connections()

    result_data: dict = {
        "error_type": type(exception).__name__ if exception else "Unknown",
        "error_message": str(exception) if exception else "Unknown error",
    }
    if traceback:
        tb_str = "".join(_tb.format_tb(traceback))
        result_data["traceback"] = tb_str[:5000]   # 截断到 5000 字符

    now = timezone.now()
    update_fields: dict = {
        "status": PaperlessTask.Status.FAILURE,
        "result_data": result_data,
        "date_done": now,
    }

    task_qs = PaperlessTask.objects.filter(task_id=task_id)
    task_instance = task_qs.values("date_started", "date_created").first()
    if task_instance:
        date_started = task_instance["date_started"]
        if date_started:
            update_fields["duration_seconds"] = (now - date_started).total_seconds()
        date_created = task_instance["date_created"]
        if date_started and date_created:
            update_fields["wait_time_seconds"] = (date_started - date_created).total_seconds()
        task_qs.update(**update_fields)
```

**状态映射表**
- 仓库相对路径：`src/documents/signals/handlers.py`
- 稳定位置：`_CELERY_STATE_TO_STATUS` 字典
- 行号范围：L1019-L1023

```python
_CELERY_STATE_TO_STATUS: dict[str, PaperlessTask.Status] = {
    "SUCCESS": PaperlessTask.Status.SUCCESS,
    "FAILURE": PaperlessTask.Status.FAILURE,
    "REVOKED": PaperlessTask.Status.REVOKED,
}
```

**PaperlessTask 状态枚举**
- 仓库相对路径：`src/documents/models.py`
- 稳定位置：`PaperlessTask.Status` 枚举类
- 行号范围：L670-L679

```python
class Status(models.TextChoices):
    PENDING = "pending"
    STARTED = "started"
    SUCCESS = "success"
    FAILURE = "failure"
    REVOKED = "revoked"
```

**PaperlessTask 关键字段**
- 仓库相对路径：`src/documents/models.py`
- 稳定位置：`PaperlessTask.task_id`、`task_type`、`status`、`input_data`、`result_data`、`date_created`、`date_started`、`date_done`、`duration_seconds`、`wait_time_seconds`
- 行号范围：L740-L800

**用户可见表现**：
- 失败后文档**不会入库**（消费事务整体回滚）
- `/api/tasks/` 任务列表（系统状态对话框）中看到历史失败记录，包含 `task_type=consume_file`、`status=failure`
- `result_data` 含 `error_type`（如 `ConsumerError`）、`error_message`、`traceback`（最多 5000 字符）

### 3.9 补充：任务撤销（REVOKED）路径

- 仓库相对路径：`src/documents/signals/handlers.py`
- 稳定位置：`task_revoked_handler()` 函数
- 行号范围：L1281-L1313

任务在执行前/执行中被撤销时触发，更新状态为 `PaperlessTask.Status.REVOKED`。

### 3.10 已入库文档的缩略图缺失

如果缩略图文件因磁盘损坏/迁移丢失（文档已入库），用户可见表现：

**列表页卡片缩略图**：
- 仓库相对路径：`src-ui/src/app/components/document-list/document-card-small/document-card-small.component.html`
- 稳定位置：`<img class="card-img doc-img" [src]="getThumbUrl()">`
- 行号：L5

- 仓库相对路径：`src-ui/src/app/components/document-list/document-card-large/document-card-large.component.html`
- 稳定位置：`<img [src]="getThumbUrl()" class="card-img doc-img">`
- 行号：L5

**表现**：浏览器 `<img>` 加载 `/api/documents/<id>/thumb/` 返回 404，图片位置显示为空白破图图标（浏览器默认行为）。前端未绑定 `(error)` 事件做额外兜底。

**详情页缩略图覆盖层**：
- 仓库相对路径：`src-ui/src/app/components/document-detail/document-detail.component.html`
- 稳定位置：`<img [src]="thumbUrl" ... alt="Document loading...">`
- 行号范围：L458-L467

**表现**：同样是浏览器默认破图标，覆盖层在 `previewLoaded` 为 true 前持续显示。

---

## 四、缩略图重建入口（两条路径）

### 4.1 路径一：CLI 管理命令 `document_thumbnails`

**命令实现**
- 仓库相对路径：`src/documents/management/commands/document_thumbnails.py`
- 稳定位置：`Command` 类、`_process_document()` 函数
- 行号范围：L1-L70

```python
def _process_document(doc_id: int) -> None:
    document: Document = Document.objects.get(id=doc_id)
    parser_class = get_parser_registry().get_parser_for_file(
        document.mime_type, document.original_filename or "", document.source_path,
    )
    if parser_class is None:
        logger.warning("%s: No parser for mime type %s", document, document.mime_type)
        return
    with parser_class() as parser:
        thumb = parser.get_thumbnail(document.source_path, document.mime_type)
        shutil.move(thumb, document.thumbnail_path)

class Command(PaperlessCommand):
    supports_progress_bar = True
    supports_multiprocessing = True

    def add_arguments(self, parser) -> None:
        super().add_arguments(parser)
        parser.add_argument("-d", "--document", type=int, default=None, ...)

    def handle(self, *args, **options):
        if options["document"]:
            documents = Document.objects.filter(pk=options["document"])
        else:
            documents = Document.objects.all()
        ids = list(documents.values_list("id", flat=True))
        for result in self.process_parallel(
            _process_document, ids, description="Regenerating thumbnails...",
        ):
            if result.error:
                self.console.print(f"[red]Failed document {result.item}: {result.error}[/red]")
```

**使用方式**
```bash
# 全部重建
python manage.py document_thumbnails

# 指定单文档
python manage.py document_thumbnails --document 42

# 多进程（默认 = cpu_count//4）
python manage.py document_thumbnails --processes 4

# 关闭进度条
python manage.py document_thumbnails --no-progress-bar
```

**命令基类提供的能力**
- 仓库相对路径：`src/documents/management/commands/base.py`
- 稳定位置：`PaperlessCommand.process_parallel()`、`PaperlessCommand.track()`
- 行号范围：L464-L550

`process_parallel()` 行为：
- `--processes 1` 时主进程顺序执行（方便测试和调试）
- `--processes > 1` 时 fork 子进程，fork 前调用 `db.connections.close_all()`（PostgreSQL 兼容性要求）
- 每个文档的结果包装为 `ProcessResult(item, result, error)`，失败不中断整体流程

### 4.2 路径二：Web UI "Reprocess" 按钮

**前端入口（详情页）**
- 仓库相对路径：`src-ui/src/app/components/document-detail/document-detail.component.ts`
- 稳定位置：`DocumentDetailComponent.reprocess()` 方法
- 行号范围：L1373-L1406

```typescript
reprocess() {
  let modal = this.modalService.open(ConfirmDialogComponent, { backdrop: 'static' })
  modal.componentInstance.title = $localize`Reprocess confirm`
  modal.componentInstance.messageBold = $localize`This operation will permanently recreate the archive file for this document.`
  modal.componentInstance.message = $localize`The archive file will be re-generated with the current settings.`
  modal.componentInstance.btnClass = 'btn-danger'
  modal.componentInstance.btnCaption = $localize`Proceed`
  modal.componentInstance.confirmClicked.subscribe(() => {
    this.documentsService.reprocessDocuments({ documents: [this.document.id] })
      .subscribe({ next: () => { toast.info("...will begin in the background."); modal.close() },
                   error: (err) => { toast.showError("Error executing operation", err) } })
  })
}
```

**前端入口（列表页批量）**
- 仓库相对路径：`src-ui/src/app/components/document-list/bulk-editor/bulk-editor.component.ts`
- 稳定位置：`BulkEditorComponent.reprocessSelected()` 方法
- 行号范围：L891-L906

**HTTP 请求**
- 仓库相对路径：`src-ui/src/app/services/rest/document.service.ts`
- 稳定位置：`DocumentService.reprocessDocuments()`
- 行号范围：L352-L356

```typescript
reprocessDocuments(selection: DocumentSelectionQuery) {
  return this.http.post(this.getResourceUrl(null, 'reprocess'), { ...selection })
}
```

请求路径：`POST /api/documents/reprocess/`

**后端 API 路由**
- 仓库相对路径：`src/documents/views.py`
- 稳定位置：`DocumentViewSet` 中 `reprocess` action 注册
- 行号范围：L3002-L3023

**后端执行**
- 仓库相对路径：`src/documents/bulk_edit.py`
- 稳定位置：`reprocess()` 函数
- 行号范围：L395-L402

```python
def reprocess(doc_ids: list[int]) -> Literal["OK"]:
    for document_id in doc_ids:
        update_document_content_maybe_archive_file.apply_async(
            kwargs={"document_id": document_id},
            headers={"trigger_source": PaperlessTask.TriggerSource.MANUAL},
        )
    return "OK"
```

**任务类型枚举**
- 仓库相对路径：`src/documents/models.py`
- 稳定位置：`PaperlessTask.TaskType.REPROCESS_DOCUMENT = "reprocess_document"`
- 行号：L691

- 仓库相对路径：`src-ui/src/app/data/paperless-task.ts`
- 稳定位置：`PaperlessTaskType.ReprocessDocument = 'reprocess_document'`
- 行号：L12

**用户可见表现**：点击 Reprocess → 弹出确认框 → 确认后 Toast 提示 "will begin in the background" → 任务列表中出现 `task_type=reprocess_document` 的新任务 → 执行过程中缩略图被重新生成。

---

## 五、预览加载失败的前端展示（四种场景）

### 5.1 场景一：PDF 需要密码

**详情页处理逻辑**
- 仓库相对路径：`src-ui/src/app/components/document-detail/document-detail.component.ts`
- 稳定位置：`DocumentDetailComponent.onError()` 方法
- 行号范围：L1502-L1507

```typescript
onError(event) {
  if (event.name == 'PasswordException') {
    this.requiresPassword = true
    this.previewLoaded = true
  }
}
```

**详情页模板渲染**
- 仓库相对路径：`src-ui/src/app/components/document-detail/document-detail.component.html`
- 稳定位置：`@if (requiresPassword)` 块 + `<div class="password-prompt">`
- 行号范围：L509-L515

```html
@if (requiresPassword) {
  <div class="password-prompt">
    <form>
      <input autocomplete="" autofocus="true" class="form-control"
             placeholder="Enter Password" type="password" (keyup)="onPasswordKeyUp($event)" />
    </form>
  </div>
}
```

**预览弹窗处理逻辑**
- 仓库相对路径：`src-ui/src/app/components/common/preview-popup/preview-popup.component.ts`
- 稳定位置：`PreviewPopupComponent.onError()` 方法
- 行号范围：L109-L115

**预览弹窗模板**
- 仓库相对路径：`src-ui/src/app/components/common/preview-popup/preview-popup.component.html`
- 稳定位置：`@if (requiresPassword)` → `file-earmark-lock` 图标
- 行号范围：L20-L24

```html
@if (requiresPassword) {
  <div class="w-100 h-100 position-relative">
    <i-bs width="2em" height="2em" class="position-absolute top-50 start-50 translate-middle" name="file-earmark-lock"></i-bs>
  </div>
}
```

**用户可见表现**：详情页显示密码输入框；列表悬停弹窗显示居中的锁图标。

### 5.2 场景二：PDF Viewer 通用加载错误（非密码）

**预览弹窗处理逻辑**
- 仓库相对路径：`src-ui/src/app/components/common/preview-popup/preview-popup.component.ts`
- 稳定位置：`PreviewPopupComponent.onError()` 的 else 分支 → `this.error = true`
- 行号范围：L109-L115

```typescript
onError(event: any) {
  if (event.name == 'PasswordException') {
    this.requiresPassword = true
  } else {
    this.error = true
  }
}
```

**预览弹窗模板渲染**
- 仓库相对路径：`src-ui/src/app/components/common/preview-popup/preview-popup.component.html`
- 稳定位置：`@if (error)` → "Error loading preview" 斜体文本
- 行号范围：L8-L11

```html
@if (error) {
  <div class="w-100 h-100 position-relative">
    <p class="fst-italic position-absolute top-50 start-50 translate-middle">Error loading preview</p>
  </div>
}
```

**用户可见表现**：悬停弹窗中央显示斜体灰色文字 "Error loading preview"。

### 5.3 场景三：TIFF 渲染失败

**详情页 TIFF 渲染逻辑**
- 仓库相对路径：`src-ui/src/app/components/document-detail/document-detail.component.ts`
- 稳定位置：`DocumentDetailComponent.tryRenderTiff()` 方法中的两个错误处理分支（HTTP 请求错误 + UTIF.js 解码异常）
- 行号范围：L1940-L1984

```typescript
// HTTP 请求失败分支
error: (err) => {
  this.tiffError = $localize`An error occurred loading tiff: ${err.toString()}`
}

// UTIF.js 解码异常分支
catch (err) {
  this.tiffError = $localize`An error occurred loading tiff: ${err.toString()}`
}
```

**详情页模板渲染**
- 仓库相对路径：`src-ui/src/app/components/document-detail/document-detail.component.html`
- 稳定位置：`@case (ContentRenderType.TIFF)` 中的 `@if (!tiffError) / @else`
- 行号范围：L496-L503

```html
@case (ContentRenderType.TIFF) {
  @if (!tiffError) {
    <div class="preview-sticky">
      <img [src]="tiffURL" width="100%" height="100%" alt="{{title}}" />
    </div>
  } @else {
    <div class="preview-sticky bg-light p-3 overflow-auto whitespace-preserve" width="100%">{{tiffError}}</div>
  }
}
```

**元数据加载时重置 tiffError**
- 仓库相对路径：`src-ui/src/app/components/document-detail/document-detail.component.ts`
- 稳定位置：`loadMetadataForSelectedVersion()` 中 `this.tiffError = null`
- 行号范围：L384-L416

**用户可见表现**：预览区域显示浅灰背景、带滚动条的错误文本，包含具体异常信息（如 "An error occurred loading tiff: TypeError: ..."）。

### 5.4 场景四：文本类文档内容获取失败

**详情页预览文本加载逻辑**
- 仓库相对路径：`src-ui/src/app/components/document-detail/document-detail.component.ts`
- 稳定位置：加载 `content` 字段时的 `.subscribe({ error: ... })` 回调
- 行号范围：L509-L514

```typescript
.subscribe({
  next: (res) => (this.previewText = res.toString()),
  error: (err) =>
    (this.previewText = `An error occurred loading content: ${err.message ?? err.toString()}`),
})
```

**详情页模板渲染**
- 仓库相对路径：`src-ui/src/app/components/document-detail/document-detail.component.html`
- 稳定位置：`@case (ContentRenderType.Text)` → `<div>{{previewText}}</div>`
- 行号范围：L488-L489

```html
@case (ContentRenderType.Text) {
  <div class="preview-sticky bg-light p-3 overflow-auto whitespace-preserve" width="100%">{{previewText}}</div>
}
```

**预览弹窗模板渲染**
- 仓库相对路径：`src-ui/src/app/components/common/preview-popup/preview-popup.component.html`
- 稳定位置：`@if (previewText)` → 同样式文本容器
- 行号范围：L14-L15

**用户可见表现**：预览区域直接显示 "An error occurred loading content: ..." 错误信息，不打断其他操作。

---

## 六、缓存读取机制（双层）

### 6.1 Django 应用缓存层：缩略图修改时间

**缓存 Key 构造**
- 仓库相对路径：`src/documents/caching.py`
- 稳定位置：`get_thumbnail_modified_key()` 函数
- 行号范围：L329-L331

```python
def get_thumbnail_modified_key(document_id: int) -> str:
    return f"doc_{document_id}_thumbnail_modified"
```

**TTL 常量**
- 仓库相对路径：`src/documents/caching.py`
- 稳定位置：`CACHE_50_MINUTES = 50 * 60`
- 行号范围：L46-L48

**读缓存逻辑**
- 仓库相对路径：`src/documents/conditionals.py`
- 稳定位置：`thumbnail_last_modified()` 函数
- 行号范围：L120-L146

```python
def thumbnail_last_modified(request, pk: int) -> datetime | None:
    doc_key = get_thumbnail_modified_key(doc.id)
    cache_hit = cache.get(doc_key)
    if cache_hit is not None:
        cache.touch(doc_key, CACHE_50_MINUTES)  # 命中刷新 TTL
        return cache_hit
    # 未命中 → 读文件 mtime → 回填缓存
    last_modified = datetime.fromtimestamp(doc.thumbnail_path.stat().st_mtime, tz=UTC)
    cache.set(doc_key, last_modified, CACHE_50_MINUTES)
    return last_modified
```

**缓存失效**
- 仓库相对路径：`src/documents/caching.py`
- 稳定位置：`clear_document_caches()` 函数
- 行号范围：L336-L345

```python
def clear_document_caches(document_id: int) -> None:
    cache.delete_many([
        get_suggestion_cache_key(document_id),
        get_metadata_cache_key(document_id),
        get_thumbnail_modified_key(document_id),
    ])
```

### 6.2 HTTP 浏览器缓存层：协商缓存

**装饰器挂载**
- 仓库相对路径：`src/documents/views.py`
- 稳定位置：`DocumentViewSet.thumb()` 的 `@last_modified` 装饰器
- 行号范围：L1565-L1583

- 仓库相对路径：`src/documents/views.py`
- 稳定位置：`DocumentViewSet.preview()` 的 `@condition(etag_func=..., last_modified_func=...)` 装饰器
- 行号范围：L1538-L1563

**preview ETag 计算**
- 仓库相对路径：`src/documents/conditionals.py`
- 稳定位置：`preview_etag()` 函数
- 行号范围：L94-L106

```python
def preview_etag(request, pk: int) -> str | None:
    use_original = request.query_params.get("original") == "true"
    return doc.checksum if use_original else doc.archive_checksum
```

所有端点同时有 `@cache_control(no_cache=True)`，表示浏览器每次协商验证（304 Not Modified）而不直接使用过期缓存。

---

## 七、预览生成链路（serve_file 分发）

预览没有独立的"生成"过程，直接返回原始文件或归档文件。

### 7.1 API 端点

**预览端点**
- 仓库相对路径：`src/documents/views.py`
- 稳定位置：`DocumentViewSet.preview()` 方法
- 行号范围：L1544-L1563

路由：`GET /api/documents/<id>/preview/?original=true&version=<id>`

**缩略图端点**
- 仓库相对路径：`src/documents/views.py`
- 稳定位置：`DocumentViewSet.thumb()` 方法
- 行号范围：L1568-L1583

路由：`GET /api/documents/<id>/thumb/?version=<id>`

### 7.2 serve_file() 分发逻辑

- 仓库相对路径：`src/documents/views.py`
- 稳定位置：`serve_file()` 函数
- 行号范围：L4473-L4522

```python
def serve_file(*, doc, use_archive, disposition, follow_formatting=False):
    if use_archive:
        file_handle = doc.archive_file
        mime_type = "application/pdf"
    else:
        file_handle = doc.source_file
        mime_type = doc.mime_type
        if mime_type in {"application/csv", "text/csv"} and disposition == "inline":
            mime_type = "text/plain"   # CSV 转纯文本，便于浏览器内联显示
    # ... 构造 Content-Disposition 头（Unicode 安全文件名） ...
    return FileResponse(file_handle, content_type=mime_type)
```

`use_archive` 决策：用户未传 `?original=true` **且**文档有归档版本 → True（返回 PDF），否则 False（返回原始文件）。

### 7.3 前端 URL 构造

- 仓库相对路径：`src-ui/src/app/services/rest/document.service.ts`
- 稳定位置：`DocumentService.getThumbUrl()` / `getPreviewUrl()`
- 行号范围：L215-L237

---

## 八、状态展示链路（WebSocket 实时进度）

### 8.1 后端推送

- 仓库相对路径：`src/documents/plugins/helpers.py`
- 稳定位置：`ProgressManager.send_progress()` 方法
- 行号范围：L119-L150

推送数据结构包含：`filename`、`task_id`、`current_progress`、`max_progress`、`status`（STARTED/WORKING/SUCCESS/FAILED）、`message`（如 `"generating_thumbnail"`）、`document_id`。

### 8.2 前端接收与翻译

- 仓库相对路径：`src-ui/src/app/services/websocket-status.service.ts`
- 稳定位置：`FILE_STATUS_MESSAGES` 常量
- 行号范围：L25-L42

```typescript
export const FILE_STATUS_MESSAGES = {
  parsing_document:     $localize`Processing document...`,
  generating_thumbnail: $localize`Generating thumbnail...`,
  parse_date:           $localize`Retrieving date from document...`,
  save_document:        $localize`Saving document...`,
  finished:             $localize`Finished.`,
  // ...
}
```

进度换算：上传阶段占 20%，后端处理阶段占 80%。`generating_thumbnail`（70/100 WORKING）→ `0.2 + 0.7 * 0.8 = 76%`。

---

## 九、缩略图/预览链路总览

```
文档入库阶段（一次性）
│
├─ consume_file (Celery Task)
│   ├─ 20%  parser.parse() → 文本/OCR
│   ├─ 70%  parser.get_thumbnail() → 临时 WebP  [generating_thumbnail]
│   ├─ 90%  日期匹配
│   └─ 95%  FileLock → 写入 source_path / thumbnail_path / archive_path
│
└─ 失败 → _fail() → WebSocket FAILED + PaperlessTask(status=FAILURE)


用户访问阶段（反复发生）
│
├─ GET /api/documents/<id>/thumb/
│   └─ @last_modified(thumbnail_last_modified)
│        ├─ Django cache: doc_{id}_thumbnail_modified (TTL 50min, 命中刷新)
│        └─ FileResponse(thumbnail_file, "image/webp")
│
├─ GET /api/documents/<id>/preview/
│   └─ @condition(etag=archive_checksum|checksum)
│        └─ serve_file(use_archive=has_archive && !?original)
│             ├─ True  → archive_file  (application/pdf)
│             └─ False → source_file   (原始 MIME，CSV→text/plain)
│
└─ 前端
    ├─ 列表卡片 <img [src]="getThumbUrl()">  → 失败=浏览器默认破图
    ├─ 悬停弹窗 pngx-preview-popup            → 失败=Error loading preview / 锁图标
    └─ 详情页 #previewContent
         ├─ PDF  → PasswordException → 密码输入框
         ├─ TIFF → tiffError → 浅灰底错误文本
         ├─ Text → previewText → 浅灰底错误文本
         └─ Img  → <img src> → 浏览器默认破图


事后修复（手动触发）
│
├─ CLI:  document_thumbnails [--document N] [--processes N] [--no-progress-bar]
│   └─ PaperlessCommand.process_parallel() → _process_document()
│        └─ parser.get_thumbnail() → shutil.move()
│
└─ Web:  Reprocess 按钮
    └─ POST /api/documents/reprocess/ → bulk_edit.reprocess()
         └─ update_document_content_maybe_archive_file.apply_async()
              └─ 同 consume_file 的缩略图生成逻辑
```
