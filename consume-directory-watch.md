# Paperless-ngx Consume Directory 监控机制深度分析

## 一、核心架构概览

Consume Directory 监控功能由以下核心组件协同工作：

| 组件 | 文件 | 职责 |
|------|------|------|
| 命令入口 | [document_consumer.py](file:///d:/fz/0601/solo-dogfeeding/code/69-paperless-ngx/src/documents/management/commands/document_consumer.py) | Django Management Command，协调监控流程 |
| 文件系统监听 | watchfiles 库 | 底层文件变更事件检测（支持原生通知 + 轮询） |
| 稳定性跟踪器 | [FileStabilityTracker](file:///d:/fz/0601/solo-dogfeeding/code/69-paperless-ngx/src/documents/management/commands/document_consumer.py#L75-L179) | 检测文件写入完成状态，并在监控侧做事件级去重 |
| 文件过滤器 | [ConsumerFilter](file:///d:/fz/0601/solo-dogfeeding/code/69-paperless-ngx/src/documents/management/commands/document_consumer.py#L182-L284) | 过滤无效文件类型和系统文件 |
| 消费任务 | [consume_file](file:///d:/fz/0601/solo-dogfeeding/code/69-paperless-ngx/src/documents/tasks.py#L123-L220) | Celery 异步任务，执行实际文档处理 |
| 预检插件 | [ConsumerPreflightPlugin](file:///d:/fz/0601/solo-dogfeeding/code/69-paperless-ngx/src/documents/consumer.py#L954-L1054) | 任务执行前的文件存在性和 SHA256 内容校验 |

---

## 二、各阶段边界精确划分

整个流程可以划分为 **5 个明确阶段**，每个阶段的去重能力和性质完全不同：

```
┌───────────────────────────────────────────────────────────────────┐
│ 阶段 1: 启动扫描 (_process_existing_files)                        │
│   位置: document_consumer.py L452-L477                            │
│   行为: glob 扫描已有文件，直接调用 _consume_file                  │
│   ★ 无去重  ★ 无稳定性等待                                         │
└────────────────────────────┬──────────────────────────────────────┘
                             ↓
┌───────────────────────────────────────────────────────────────────┐
│ 阶段 2: 监控侧事件去重 (FileStabilityTracker)                      │
│   位置: document_consumer.py L75-L179                             │
│   行为: 内存 dict[Path, TrackedFile] 合并同一文件的多次事件        │
│   ★ 真正的"避免重复入队"机制（仅在 watch 模式、同一进程内有效）    │
└────────────────────────────┬──────────────────────────────────────┘
                             ↓
┌───────────────────────────────────────────────────────────────────┐
│ 阶段 3: 任务入队 (_consume_file)                                   │
│   位置: document_consumer.py L308-L353                            │
│   行为: 调用 consume_file.apply_async 提交 Celery 任务             │
│   ★ 仅检查文件存在性，不检查是否已有相同文件的任务在队列中          │
└────────────────────────────┬──────────────────────────────────────┘
                             ↓
┌───────────────────────────────────────────────────────────────────┐
│ 阶段 4: 消费阶段 SHA256 查重 (ConsumerPreflightPlugin)             │
│   位置: consumer.py L979-L1030                                     │
│   行为: 计算 SHA256，查询 Document.global_objects                  │
│   ★ 不是"避免重复入队"，是"避免重复入库"（任务已入队，已在执行）    │
└────────────────────────────┬──────────────────────────────────────┘
                             ↓
┌───────────────────────────────────────────────────────────────────┐
│ 阶段 5: 消费成功后删除源文件                                       │
│   位置: consumer.py L737-L758                                      │
│   行为: self.input_doc.original_file.unlink()                      │
│   ★ 间接去重：文件消失，后续扫描/监控不会再发现它                   │
└───────────────────────────────────────────────────────────────────┘
```

---

## 三、各阶段详细分析与去重能力评估

### 阶段 1：启动扫描（_process_existing_files）

**代码位置**：[document_consumer.py L452-L477](file:///d:/fz/0601/solo-dogfeeding/code/69-paperless-ngx/src/documents/management/commands/document_consumer.py#L452-L477)

```python
def _process_existing_files(self, ...) -> None:
    glob_pattern = "**/*" if recursive else "*"
    for filepath in directory.glob(glob_pattern):
        if not filepath.is_file():
            continue
        if not consumer_filter(Change.added, str(filepath)):
            continue
        _consume_file(...)  # 直接入队，无任何去重
```

**性质分析**：

| 维度 | 说明 |
|------|------|
| 是否去重 | ❌ **完全没有去重逻辑** |
| 是否等待稳定性 | ❌ **没有稳定性等待**，文件立即入队 |
| 与后续 watch 的关系 | 串行执行：先扫描完全部文件，再进入 `_watch_directory()`。所以**同一进程生命周期内**，启动扫描和 watch 不会重复处理同一文件 |
| 风险窗口 | 监控进程重启时：若上次提交的任务还未消费完成（源文件尚未被删除），本次启动扫描会再次发现该文件并**重复入队** |

---

### 阶段 2：监控侧事件去重（FileStabilityTracker）

**代码位置**：[document_consumer.py L75-L179](file:///d:/fz/0601/solo-dogfeeding/code/69-paperless-ngx/src/documents/management/commands/document_consumer.py#L75-L179)

这是 **整个链路中唯一真正"避免重复入队"的机制**，但仅在 watch 模式下生效。

#### 数据结构

```python
@dataclass
class TrackedFile:
    path: Path                    # 绝对路径，作为 dict 的唯一键
    last_event_time: float        # 最后一次事件的 monotonic 时间戳
    last_mtime: float | None      # 文件 st_mtime
    last_size: int | None         # 文件 st_size

class FileStabilityTracker:
    _tracked: dict[Path, TrackedFile]  # 以规范化路径为键的字典
```

#### 去重逻辑：track() 方法

[document_consumer.py L102-L129](file:///d:/fz/0601/solo-dogfeeding/code/69-paperless-ngx/src/documents/management/commands/document_consumer.py#L102-L129)

```python
def track(self, path: Path, change: Change) -> None:
    path = path.resolve()  # 关键：路径规范化，确保同一文件映射到同一键

    match change:
        case Change.deleted:
            self._tracked.pop(path, None)   # 删除事件：立即移除跟踪
        case Change.added | Change.modified:
            if path in self._tracked:
                # 同一路径已在跟踪中 → 只更新时间戳，不创建新条目
                tracked = self._tracked[path]
                tracked.last_event_time = monotonic()
                tracked.update_stats()
            else:
                # 新路径 → 创建 TrackedFile 并存入 dict
                tracked = TrackedFile(path=path, last_event_time=monotonic())
                if tracked.update_stats():
                    self._tracked[path] = tracked
```

**关键设计**：
- `path.resolve()` 规范化路径，避免符号链接、相对路径导致同一文件被跟踪多次
- `dict[Path, TrackedFile]` 以路径为键，天然去重：无论同一文件触发多少次 `added`/`modified` 事件，都只保留一个条目
- `last_event_time` 每次事件都刷新，避免文件持续写入时提前被判定为稳定

#### 去重逻辑：get_stable_files() 方法

[document_consumer.py L131-L170](file:///d:/fz/0601/solo-dogfeeding/code/69-paperless-ngx/src/documents/management/commands/document_consumer.py#L131-L170)

```python
def get_stable_files(self) -> Iterator[Path]:
    for path, tracked in self._tracked.items():
        # ... 稳定性检查 ...

    # 关键：稳定文件 yield 之前先从 _tracked 中 pop 移除
    for path in to_yield:
        self._tracked.pop(path, None)
        yield path
```

**关键设计**：文件一旦被判定为稳定并返回，立即从跟踪字典中移除。即使后续还有该文件的事件（极端情况下），也会被视为新文件重新跟踪（但此时源文件通常已被消费流程删除）。

#### 本阶段去重能力评估

| 维度 | 说明 |
|------|------|
| 是否真正避免重复入队 | ✅ **是** — 在 watch 模式的同一进程生命周期内，同一文件只会被提交一次到 Celery |
| 作用范围 | 仅同一进程、同一 FileStabilityTracker 实例 |
| 失效场景 | 进程重启后 `_tracked` 字典清空，之前跟踪的文件全部丢失 |
| 失效场景 | 启动扫描阶段不走 FileStabilityTracker，不经过此处 |

---

### 阶段 3：任务入队（_consume_file）

**代码位置**：[document_consumer.py L308-L353](file:///d:/fz/0601/solo-dogfeeding/code/69-paperless-ngx/src/documents/management/commands/document_consumer.py#L308-L353)

```python
def _consume_file(filepath: Path, ...) -> None:
    # 检查 1: 文件是否仍然存在
    try:
        if not filepath.is_file():
            logger.debug(f"Not consuming {filepath}: not a file or doesn't exist")
            return
    except OSError as e:
        logger.warning(f"Not consuming {filepath}: {e}")
        return

    # 检查 2: 生成子目录标签（失败不影响主流程）
    tag_ids = ...
    if subdirs_as_tags:
        try:
            tag_ids = _tags_from_path(filepath, consumption_dir)
        except Exception:
            logger.exception(...)

    # 检查 3: 直接提交 Celery 任务，无任何去重检查
    try:
        consume_file.apply_async(
            kwargs={
                "input_doc": ConsumableDocument(source=DocumentSource.ConsumeFolder,
                                                 original_file=filepath),
                "overrides": DocumentMetadataOverrides(tag_ids=tag_ids),
            },
            headers={"trigger_source": PaperlessTask.TriggerSource.FOLDER_CONSUME},
        )
    except Exception:
        logger.exception(f"Error while queuing document {filepath}")
```

#### 本阶段去重能力评估

| 维度 | 说明 |
|------|------|
| 是否避免重复入队 | ❌ **否** — 不查询 PaperlessTask 表，不检查是否已有相同文件的 PENDING/STARTED 任务 |
| `is_file()` 检查的作用 | 仅防止以下竞态：文件在稳定性检查通过后、入队前被用户删除/移动。不是去重 |
| 标签生成异常处理 | 独立 try-except，标签失败不阻止文件入队，属于容错而非去重 |
| 可能的重复入队 | 如果因进程重启导致同一文件两次到达 `_consume_file`，这里不会拦截 |

**重要事实**：`_consume_file` 不查询 `PaperlessTask` 表。虽然 PaperlessTask 记录中包含 `input_data.filename` 和 `trigger_source=FOLDER_CONSUME`，但入队前不做检查。

---

### 阶段 4：消费阶段 SHA256 查重（ConsumerPreflightPlugin）

**代码位置**：[consumer.py L979-L1030](file:///d:/fz/0601/solo-dogfeeding/code/69-paperless-ngx/src/documents/consumer.py#L979-L1030)

**这不是"避免重复入队"的机制 — 任务已经入队并在 Celery Worker 中执行了。** 它避免的是"重复入库"（即同一内容在 Document 表中创建多条记录）。

```python
def pre_check_duplicate(self) -> None:
    checksum = compute_checksum(Path(self.input_doc.original_file))
    existing_doc = Document.global_objects.filter(
        Q(checksum=checksum) | Q(archive_checksum=checksum),
    )
    if existing_doc.exists():
        # 已有相同内容的文档
        if settings.CONSUMER_DELETE_DUPLICATES:
            Path(self.input_doc.original_file).unlink()  # 删除源文件
            raise ConsumeFileDuplicateError(...)
```

#### 查重维度

| 字段 | 说明 |
|------|------|
| `Document.checksum` | 原始源文件的 SHA256 |
| `Document.archive_checksum` | PDF/A 归档文件的 SHA256 |
| `Document.global_objects` | 包括已软删除（回收站）的文档，不遗漏 |

#### 本阶段去重能力评估

| 维度 | 说明 |
|------|------|
| 是否避免重复入队 | ❌ **否** — 任务已经在执行，浪费了一次 Worker 调度和 SHA256 计算 |
| 是否避免重复入库 | ✅ **是** — 防止同一内容创建多条 Document 记录 |
| 生效前提 | 之前相同内容的文件已经被**成功消费并入库**（Document 记录存在） |
| 失效场景 | 如果之前的任务只是 PENDING/STARTED，Document 尚未写入数据库，则 SHA256 查重检测不到。此时多个重复任务会同时通过查重，并行执行解析。最终只有第一条成功写入的任务会留下 Document，其余在写入时可能因后续流程发现冲突或报错 |
| 与 CONSUMER_DELETE_DUPLICATES 的关系 | 设为 True 时，查重不通过会主动删除源文件（间接防止后续重启再次入队）；设为 False 时，仅记录警告，文件仍在 consume 目录中，重启后会再次触发重复入队 |

---

### 阶段 5：消费成功后删除源文件

**代码位置**：[consumer.py L737-L758](file:///d:/fz/0601/solo-dogfeeding/code/69-paperless-ngx/src/documents/consumer.py#L737-L758)

```python
# 仅在成功消费后删除
self.input_doc.original_file.unlink()
self.working_copy.unlink()

# 同时清理 macOS 资源分叉文件
shadow_file = Path(...).parent / f"._{Path(...).name}"
if Path(shadow_file).is_file():
    Path(shadow_file).unlink()
```

#### 本阶段去重能力评估

| 维度 | 说明 |
|------|------|
| 是否避免重复入队 | ⚠️ **间接防止** — 文件被删除后，后续扫描/监控不会再发现它 |
| 时间窗口 | 任务被提交（PENDING）→ Worker 执行完成（SUCCESS） 之间，源文件仍然存在。此时若监控进程重启，启动扫描会重复入队 |
| 与 CONSUMER_DELETE_DUPLICATES 的区别 | 前者是"成功消费后正常清理"；后者是"查重发现重复时立即删除，避免后续再次触发" |

---

## 四、重复入队风险矩阵

以下是各种场景下是否会发生重复入队的精确分析：

| 场景 | 是否重复入队 | 原因 |
|------|-------------|------|
| watch 模式下同一文件触发多次 added/modified 事件 | ❌ 不会 | FileStabilityTracker 以 Path 为键合并事件，稳定后 pop 移除 |
| watch 模式下文件稳定后，用户再次修改 | ⚠️ 会再次入队 | 稳定文件已从 _tracked 中 pop，新的 modified 事件会重新跟踪并在稳定后再次入队。但 SHA256 查重会在消费阶段拦截（若 CONSUMER_DELETE_DUPLICATES=True 则删除源文件） |
| 单次启动，oneshot 模式 | ❌ 不会 | glob 扫描时逐个处理，单次遍历不重复 |
| 消费任务还在 PENDING，监控进程重启 | ✅ **会！** | 启动扫描重新发现源文件（尚未被成功消费删除），再次调用 _consume_file。SHA256 查重此时还未入库，检测不到。会有多个相同任务并行执行 |
| 消费任务已 SUCCESS，但源文件未被删除（极端异常） | ✅ **会！** | 启动扫描再次发现文件。但此时 Document 已存在，SHA256 查重能拦截（若 CONSUMER_DELETE_DUPLICATES=True 会删除文件，避免下次再重复） |
| 消费者进程崩溃，文件停留在 consume 目录 | ✅ **会！** | 每次重启都会重新发现。SHA256 查重能否拦截取决于上次崩溃前 Document 是否已写入 DB |
| 用户手动将同一文件复制两次到 consume 目录 | ✅ **会** | 两个不同路径（即使内容相同）是不同的 TrackedFile。SHA256 查重会在消费阶段拦截第二次 |

---

## 五、文件稳定性检测机制

### 为什么需要稳定性检测？

文件从外部写入 consume 目录时可能存在：
- **网络传输**：SMB/NFS/WebDAV 等协议分段写入
- **扫描设备**：扫描仪多次打开/关闭文件
- **大文件复制**：需要数秒甚至数分钟完成
- **临时文件重命名**：先写 `.tmp` 再重命名

如果在写入完成前处理，会导致解析失败或内容不完整。

### 稳定性判定三条件

文件被认为"稳定"需同时满足：

1. **时间条件**：距最后一次文件系统事件超过 `stability_delay`（默认 5 秒）
2. **内容条件**：文件大小（`st_size`）和修改时间（`st_mtime`）未变化
3. **存在条件**：文件仍存在且 stat 成功

### 稳定性检测流程

[FileStabilityTracker.get_stable_files()](file:///d:/fz/0601/solo-dogfeeding/code/69-paperless-ngx/src/documents/management/commands/document_consumer.py#L131-L170)

```
遍历所有跟踪文件
    ↓
距最后事件 < stability_delay? → 跳过，继续等待
    ↓ 是
调用 TrackedFile.is_unchanged() 二次校验 stat
    ↓
文件不存在或 stat 失败 → 标记为待移除
    ↓
stats (mtime/size) 变化了 → 更新 last_event_time = now，重置计时器再等一轮
    ↓ 未变化
加入返回列表
    ↓
批量移除不存在的文件
    ↓
逐个 yield 稳定文件，同时从 _tracked 中 pop（防止重复返回）
```

### 处理"检查时恰好被修改"的边缘情况

```python
if not tracked.is_unchanged():
    if tracked.update_stats():          # 文件还在
        tracked.last_event_time = current_time  # 重置计时器
        logger.debug(f"File changed during stability check: {path}")
    else:                                # 文件消失
        to_remove.append(path)
    continue
```

### stat 容错处理

[TrackedFile.update_stats()](file:///d:/fz/0601/solo-dogfeeding/code/69-paperless-ngx/src/documents/management/commands/document_consumer.py#L51-L61) 和 [is_unchanged()](file:///d:/fz/0601/solo-dogfeeding/code/69-paperless-ngx/src/documents/management/commands/document_consumer.py#L63-L72) 都捕获 OSError：

```python
def update_stats(self) -> bool:
    try:
        stat = self.path.stat()
        self.last_mtime = stat.st_mtime
        self.last_size = stat.st_size
        return True
    except OSError:  # FileNotFoundError, PermissionError 等
        return False
```

---

## 六、扫描并发与超时控制

### 双模式文件监听

使用 [watchfiles](https://watchfiles.helpmanual.io/) 库：

| 模式 | 触发条件 | 适用场景 |
|------|----------|----------|
| **原生通知** | `CONSUMER_POLLING_INTERVAL = 0`（默认） | 本地文件系统：inotify (Linux) / FSEvents (macOS) / ReadDirectoryChangesW (Windows) |
| **轮询模式** | `CONSUMER_POLLING_INTERVAL > 0` | 网络文件系统（SMB/NFS），原生通知不可靠 |

模式切换在 [_watch_directory()](file:///d:/fz/0601/solo-dogfeeding/code/69-paperless-ngx/src/documents/management/commands/document_consumer.py#L479-L577)：

```python
use_polling = polling_interval > 0
poll_delay_ms = int(polling_interval * 1000) if use_polling else 0
```

### 动态超时策略

监控循环的超时根据是否有待处理文件动态调整：

```python
while not self.stop_flag.is_set():
    for changes in watch(..., rust_timeout=timeout_ms, yield_on_timeout=True, ...):
        # 处理事件
        # 检查稳定文件
        break  # 退出内层循环，重新评估 timeout

    if tracker.has_pending_files():
        timeout_ms = stability_timeout_ms   # 有文件等待稳定 → 以 stability_delay 为周期唤醒
    elif is_testing:
        timeout_ms = testing_timeout_ms     # 测试模式 → 短超时
    else:
        timeout_ms = 0                       # 无待处理文件 → 无限等待（阻塞）
```

**设计意图**：
- 无文件时阻塞等待，不消耗 CPU
- 有文件等待稳定时，周期性唤醒检查
- 每次 watch 调用后 break 退出重新评估超时

### 轮询模式下的超时保护

测试模式下，轮询模式的 timeout 至少为轮询间隔的 3 倍：

```python
if is_testing and use_polling:
    min_polling_timeout_ms = poll_delay_ms * 3
    timeout_ms = max(min_polling_timeout_ms, testing_timeout_ms)
```

防止 Rust 内部轮询线程尚未完成一次扫描就被超时中断。

### 单线程串行处理

整个监控循环为**单线程串行**：
1. `watch()` 阻塞等待文件事件
2. 逐个事件调用 `tracker.track()`
3. 调用 `tracker.get_stable_files()` 逐个处理稳定文件
4. 每个稳定文件调用 `_consume_file()` 提交任务
5. 循环回到第 1 步

**好处**：`_tracked` 字典无锁访问，状态流转清晰；实际文档解析并发由 Celery Worker 承担。

---

## 七、错误隔离与容错设计

### 7.1 文件过滤器（ConsumerFilter）

**位置**：[document_consumer.py L182-L284](file:///d:/fz/0601/solo-dogfeeding/code/69-paperless-ngx/src/documents/management/commands/document_consumer.py#L182-L284)

```
watchfiles 事件流
    ↓
ConsumerFilter.__call__(change, path)
    ↓
DefaultFilter (父类):
  ├─ 隐藏文件/目录（. 开头）
  ├─ ignore_dirs 中的目录（递归忽略其全部内容）
  └─ ignore_entity_patterns 匹配的文件名
    ↓ 未被过滤
是目录 → 通过（目录本身不消费，但需要监视其子内容）
    ↓
是文件 → 检查扩展名是否在 supported_extensions 中（大小写不敏感）
```

**默认忽略列表**：

- 文件正则：`^\.DS_Store$`, `^\._.*`, `^desktop\.ini$`, `^Thumbs\.db$`
- 目录名：`.stfolder`, `.stversions`, `.localized`, `.Spotlight-V100`, `.Trashes`, `__MACOSX`, `@eaDir`

### 7.2 入队阶段错误隔离

[_consume_file()](file:///d:/fz/0601/solo-dogfeeding/code/69-paperless-ngx/src/documents/management/commands/document_consumer.py#L308-L353) 每个环节独立异常处理：

```
文件存在性检查 (OSError) → 记录警告，跳过该文件
    ↓
标签生成 (Exception) → 记录异常，tag_ids=None，继续入队（不阻止主流程）
    ↓
任务提交 apply_async (Exception) → 记录异常，跳过该文件
```

**关键容错**：标签生成失败不阻止文件入队。这确保即使数据库暂时不可用，文件也不会丢失。

### 7.3 监控循环整体容错

外层 while 循环仅捕获 `KeyboardInterrupt`（[document_consumer.py L575-L577](file:///d:/fz/0601/solo-dogfeeding/code/69-paperless-ngx/src/documents/management/commands/document_consumer.py#L575-L577)）：

```python
while not self.stop_flag.is_set():
    try:
        for changes in watch(...):
            ...
            break
        ...
    except KeyboardInterrupt:
        logger.info("Received interrupt, stopping consumer")
        self.stop_flag.set()
```

其他异常（如 watchfiles 内部错误、文件系统错误）会导致进程崩溃，依赖 systemd/supervisor 自动重启。

### 7.4 子目录标签生成的数据库容错

[_tags_from_path()](file:///d:/fz/0601/solo-dogfeeding/code/69-paperless-ngx/src/documents/management/commands/document_consumer.py#L287-L305) 执行前主动关闭旧连接：

```python
def _tags_from_path(...) -> list[int]:
    db.close_old_connections()  # 防止长连接超时失效
    ...
```

---

## 八、启动流程与两种运行模式

### Oneshot 模式（一次性处理）

```
Command.handle()
    ↓
验证消费目录存在
    ↓
_process_existing_files(): glob 扫描 → ConsumerFilter → _consume_file
    ↓
直接退出，不启动监控
```

适用：容器初始化、手动批量导入。**注意：无稳定性等待，无去重。**

### Watch 模式（持续监控）

```
Command.handle()
    ↓
_process_existing_files() 处理启动时已存在的文件（无稳定等待、无去重）
    ↓
_watch_directory() 进入无限循环
    ↓
┌────────────────────────────────────────────────┐
│  watch() 阻塞等待文件系统事件                    │
│  ↓                                              │
│  每个有效事件 → tracker.track()                 │
│  ↓                                              │
│  tracker.get_stable_files() → _consume_file()   │
│  ↓                                              │
│  根据 tracker.has_pending_files() 调整 timeout  │
└────────────────────────────────────────────────┘
```

**关键点**：启动时先处理已有文件（无稳定等待），再进入监控（有稳定性检测和事件去重）。

---

## 九、配置参数汇总

| 配置项 | 环境变量 | 默认值 | 说明 |
|--------|----------|--------|------|
| `CONSUMPTION_DIR` | `PAPERLESS_CONSUMPTION_DIR` | - | 消费目录路径 |
| `CONSUMER_RECURSIVE` | `PAPERLESS_CONSUMER_RECURSIVE` | `False` | 是否递归监视子目录 |
| `CONSUMER_POLLING_INTERVAL` | `PAPERLESS_CONSUMER_POLLING_INTERVAL` | `0` | 轮询间隔秒数，0=使用原生事件通知 |
| `CONSUMER_STABILITY_DELAY` | `PAPERLESS_CONSUMER_STABILITY_DELAY` | `5` | 文件稳定等待时间（秒） |
| `CONSUMER_IGNORE_PATTERNS` | `PAPERLESS_CONSUMER_IGNORE_PATTERNS` | `[]` | 额外忽略文件名正则（JSON 数组） |
| `CONSUMER_IGNORE_DIRS` | `PAPERLESS_CONSUMER_IGNORE_DIRS` | `[]` | 额外忽略目录名（JSON 数组） |
| `CONSUMER_SUBDIRS_AS_TAGS` | `PAPERLESS_CONSUMER_SUBDIRS_AS_TAGS` | `False` | 将子目录名作为标签 |
| `CONSUMER_DELETE_DUPLICATES` | `PAPERLESS_CONSUMER_DELETE_DUPLICATES` | `False` | SHA256 查重命中时是否删除源文件 |

---

## 十、完整流程图（标注去重性质）

```
文件出现在 consume 目录
    ↓
┌─ 阶段 1/2 入口 ────────────────────────────────────────────┐
│  启动扫描（glob）  或  watchfiles 事件                      │
│  [无去重]          [FileStabilityTracker 事件级去重 ✅]     │
└────────────────────────────────────────────────────────────┘
    ↓
ConsumerFilter 过滤
    ├─ 系统文件/隐藏文件/忽略目录 → 丢弃
    ├─ 不支持的扩展名 → 丢弃
    └─ 有效文件 → 继续
    ↓
（仅 watch 模式）FileStabilityTracker.track()
    └─ 等待 stability_delay → is_unchanged() 二次校验
    ↓
┌─ 阶段 3: _consume_file ────────────────────────────────────┐
│  is_file() 检查 [存在性校验，非去重]                        │
│  标签生成 [容错：失败不阻止]                                │
│  consume_file.apply_async() [不查 PaperlessTask，无去重 ❌] │
└────────────────────────────────────────────────────────────┘
    ↓
任务进入 Celery 队列（PENDING）
    ↓
┌─ 阶段 4: Celery Worker 执行 ───────────────────────────────┐
│  ConsumerPreflightPlugin.pre_check_file_exists()           │
│  ConsumerPreflightPlugin.pre_check_duplicate()             │
│    [SHA256 查重：防止重复入库 ✅，不防止重复入队 ❌]         │
└────────────────────────────────────────────────────────────┘
    ↓
ConsumerPlugin.run() 解析、分类、存储 Document
    ↓
┌─ 阶段 5: 收尾 ─────────────────────────────────────────────┐
│  成功 → unlink() 源文件 [间接防止后续重复入队 ⚠️]          │
│  失败 → 源文件保留，重启后会再次被发现                      │
└────────────────────────────────────────────────────────────┘
```
