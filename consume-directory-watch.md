# Paperless-ngx Consume Directory 监控机制深度分析

## 一、核心架构概览

Consume Directory 监控功能由以下核心组件协同工作：

| 组件 | 文件 | 职责 |
|------|------|------|
| 命令入口 | [document_consumer.py](file:///d:/fz/0601/solo-dogfeeding/code/69-paperless-ngx/src/documents/management/commands/document_consumer.py) | Django Management Command，协调监控流程 |
| 文件系统监听 | watchfiles 库 | 底层文件变更事件检测（支持原生通知 + 轮询） |
| 稳定性跟踪器 | [FileStabilityTracker](file:///d:/fz/0601/solo-dogfeeding/code/69-paperless-ngx/src/documents/management/commands/document_consumer.py#L75-L179) | 检测文件写入完成状态 |
| 文件过滤器 | [ConsumerFilter](file:///d:/fz/0601/solo-dogfeeding/code/69-paperless-ngx/src/documents/management/commands/document_consumer.py#L182-L284) | 过滤无效文件类型和系统文件 |
| 消费任务 | [consume_file](file:///d:/fz/0601/solo-dogfeeding/code/69-paperless-ngx/src/documents/tasks.py#L123-L220) | Celery 异步任务，执行实际文档处理 |
| 预检插件 | [ConsumerPreflightPlugin](file:///d:/fz/0601/solo-dogfeeding/code/69-paperless-ngx/src/documents/consumer.py#L954-L1054) | 执行重复文件检查等前置校验 |

---

## 二、避免重复入队的多层防护机制

Paperless-ngx 通过 **四层防御** 来避免同一个文件被重复入队处理：

### 第 1 层：文件系统事件级去重（FileStabilityTracker）

**核心思路**：同一个文件在短时间内可能触发多个文件系统事件（added、modified 等），通过在内存中维护跟踪状态，确保每个文件只被处理一次。

#### TrackedFile 数据结构

```python
@dataclass
class TrackedFile:
    path: Path                    # 文件绝对路径（作为唯一键）
    last_event_time: float        # 最后一次事件的时间戳（monotonic clock）
    last_mtime: float | None      # 记录的文件修改时间
    last_size: int | None         # 记录的文件大小
```

#### 去重逻辑

在 [FileStabilityTracker.track()](file:///d:/fz/0601/solo-dogfeeding/code/69-paperless-ngx/src/documents/management/commands/document_consumer.py#L102-L129) 方法中：

```python
def track(self, path: Path, change: Change) -> None:
    path = path.resolve()  # 关键点：解析为绝对路径，确保同一文件映射到同一键

    match change:
        case Change.deleted:
            self._tracked.pop(path, None)  # 文件删除时立即清除跟踪
        case Change.added | Change.modified:
            current_time = monotonic()
            if path in self._tracked:
                # 文件已在跟踪中 -> 只更新时间戳和状态，不创建新条目
                tracked = self._tracked[path]
                tracked.last_event_time = current_time
                tracked.update_stats()
            else:
                # 新文件 -> 创建跟踪条目
                tracked = TrackedFile(path=path, last_event_time=current_time)
                if tracked.update_stats():
                    self._tracked[path] = tracked
```

**关键设计点**：
- 使用 `path.resolve()` 规范化路径，避免相对路径/符号链接导致的重复跟踪
- 内部存储使用 `dict[Path, TrackedFile]`，以路径为键，天然去重
- 文件一旦被 `get_stable_files()` 返回后立即从 `_tracked` 中移除（pop 操作），防止后续事件再次触发

### 第 2 层：消费前文件存在性二次校验

在 [_consume_file()](file:///d:/fz/0601/solo-dogfeeding/code/69-paperless-ngx/src/documents/management/commands/document_consumer.py#L308-L353) 入队前再次校验：

```python
def _consume_file(filepath: Path, ...) -> None:
    try:
        if not filepath.is_file():  # 二次确认文件仍然存在且是普通文件
            logger.debug(f"Not consuming {filepath}: not a file or doesn't exist")
            return
    except OSError as e:
        logger.warning(f"Not consuming {filepath}: {e}")
        return
    # ... 继续入队
```

这防止了以下竞态条件：
- 文件在稳定性检测通过后、入队前被用户删除
- 文件被移动到其他位置
- 文件因权限问题无法访问

### 第 3 层：任务级 SHA256 内容去重

在实际消费任务中，[ConsumerPreflightPlugin.pre_check_duplicate()](file:///d:/fz/0601/solo-dogfeeding/code/69-paperless-ngx/src/documents/consumer.py#L979-L1030) 通过文件内容哈希进行最终校验：

```python
def pre_check_duplicate(self) -> None:
    checksum = compute_checksum(Path(self.input_doc.original_file))
    existing_doc = Document.global_objects.filter(
        Q(checksum=checksum) | Q(archive_checksum=checksum),
    )
    if existing_doc.exists():
        # 找到重复文档
        if settings.CONSUMER_DELETE_DUPLICATES:
            # 删除源文件并抛出重复错误
            Path(self.input_doc.original_file).unlink()
            raise ConsumeFileDuplicateError(...)
```

**检查维度**：
- `checksum`：原始文件的 SHA256
- `archive_checksum`：归档文件（PDF/A）的 SHA256
- 使用 `Document.global_objects` 查询，包括已软删除（回收站）的文档

### 第 4 层：消费成功后自动删除源文件

在 [ConsumerPlugin.run()](file:///d:/fz/0601/solo-dogfeeding/code/69-paperless-ngx/src/documents/consumer.py#L737-L758) 中，处理成功后会删除原始文件：

```python
# Delete the file only if it was successfully consumed
self.input_doc.original_file.unlink()

# 同时删除 macOS 资源分叉文件
shadow_file = Path(self.input_doc.original_file).parent / f"._{Path(...).name}"
if Path(shadow_file).is_file():
    Path(shadow_file).unlink()
```

这样文件在消费目录中消失，即使监控重启也不会被再次发现。

---

## 三、文件稳定性检测机制

### 为什么需要稳定性检测？

文件从外部写入 consume 目录时，可能存在以下情况：
- **网络传输**：通过 SMB/NFS/WebDAV 等协议分段写入
- **扫描设备**：扫描仪多次打开/关闭文件
- **大文件复制**：需要数秒甚至数分钟才能写入完成
- **临时文件重命名**：应用先写 `.tmp` 文件再重命名

如果在文件写入完成前就开始处理，会导致解析失败或内容不完整。

### 稳定性判定三条件

文件被认为"稳定"（可安全消费）需要同时满足：

1. **时间条件**：距离最后一次文件系统事件超过 `stability_delay`（默认 5 秒）
2. **内容条件**：文件大小（`st_size`）和修改时间（`st_mtime`）未发生变化
3. **存在条件**：文件仍然存在且是普通文件

### 稳定性检测流程

[FileStabilityTracker.get_stable_files()](file:///d:/fz/0601/solo-dogfeeding/code/69-paperless-ngx/src/documents/management/commands/document_consumer.py#L131-L170) 的执行流程：

```
遍历所有跟踪文件
    ↓
距离最后事件 < stability_delay? → 跳过，继续等待
    ↓ 是
调用 is_unchanged() 二次校验
    ↓
文件不存在或 stat 失败 → 标记删除
    ↓
stats (mtime/size) 变化了 → 更新 last_event_time，重新等待一轮
    ↓ 未变化
加入返回列表，从跟踪中移除
    ↓
批量清理删除标记的文件
    ↓
逐个 yield 稳定文件路径
```

### 处理"检查时文件恰好被修改"的边缘情况

```python
# FileStabilityTracker.get_stable_files() 中的关键逻辑
if not tracked.is_unchanged():
    if tracked.update_stats():          # 文件还在
        tracked.last_event_time = current_time  # 重置计时器，再等一轮
        logger.debug(f"File changed during stability check: {path}")
    else:                                # 文件消失了
        to_remove.append(path)           # 直接移除
    continue
```

这个设计确保即使在稳定性检查的瞬间文件被修改，也不会误判为稳定。

### TrackedFile 的 stat 容错处理

[TrackedFile.update_stats()](file:///d:/fz/0601/solo-dogfeeding/code/69-paperless-ngx/src/documents/management/commands/document_consumer.py#L51-L61) 和 [is_unchanged()](file:///d:/fz/0601/solo-dogfeeding/code/69-paperless-ngx/src/documents/management/commands/document_consumer.py#L63-L72) 都对 OSError 进行了捕获：

```python
def update_stats(self) -> bool:
    try:
        stat = self.path.stat()
        self.last_mtime = stat.st_mtime
        self.last_size = stat.st_size
        return True
    except OSError:  # 包括 FileNotFoundError, PermissionError 等
        return False
```

这确保了文件权限变更、临时不可访问等异常情况不会导致监控崩溃。

---

## 四、扫描并发与超时控制

### 双模式文件监听

Paperless-ngx 使用 [watchfiles](https://watchfiles.helpmanual.io/) 库，支持两种监听模式：

| 模式 | 触发条件 | 适用场景 |
|------|----------|----------|
| **原生通知** | `CONSUMER_POLLING_INTERVAL = 0`（默认） | 本地文件系统，使用 inotify (Linux) / FSEvents (macOS) / ReadDirectoryChangesW (Windows) |
| **轮询模式** | `CONSUMER_POLLING_INTERVAL > 0` | 网络文件系统（SMB/NFS），原生通知不可靠的场景 |

模式切换逻辑在 [_watch_directory()](file:///d:/fz/0601/solo-dogfeeding/code/69-paperless-ngx/src/documents/management/commands/document_consumer.py#L479-L577)：

```python
use_polling = polling_interval > 0
poll_delay_ms = int(polling_interval * 1000) if use_polling else 0
```

### 动态超时策略

监控循环的超时时间不是固定的，而是根据是否有待处理文件动态调整：

```python
while not self.stop_flag.is_set():
    for changes in watch(..., rust_timeout=timeout_ms, yield_on_timeout=True, ...):
        # ... 处理事件和稳定性检查
        break  # 退出内层循环，重新计算 timeout

    # 动态决定下一轮的超时
    if tracker.has_pending_files():
        timeout_ms = stability_timeout_ms  # 有待处理文件 → 用稳定性延迟作为超时
    elif is_testing:
        timeout_ms = testing_timeout_ms    # 测试模式 → 短超时
    else:
        timeout_ms = 0                     # 无待处理文件 → 无限等待（阻塞直到有事件）
```

**设计意图**：
- 无文件时阻塞等待，不消耗 CPU
- 有文件正在等待稳定性检测时，以 `stability_delay` 为周期唤醒检查
- 每次 watch 调用后 break 退出，重新评估超时策略

### 轮询模式下的超时保护

在轮询模式下，watchfiles 内部的 Rust 线程需要足够时间完成扫描。测试模式下特别处理：

```python
if is_testing:
    if use_polling:
        min_polling_timeout_ms = poll_delay_ms * 3  # 至少 3 倍轮询间隔
        timeout_ms = max(min_polling_timeout_ms, testing_timeout_ms)
```

这防止了超时时间短于轮询周期导致事件丢失。

### 单线程串行处理

整个监控循环是**单线程串行**的：

1. `watch()` 阻塞等待文件事件
2. 收到事件后逐个调用 `tracker.track()` 更新跟踪状态
3. 调用 `tracker.get_stable_files()` 逐个处理稳定文件
4. 每个稳定文件调用 `_consume_file()` 提交 Celery 任务
5. 循环回到第 1 步

这种设计的好处：
- **无并发竞争**：`_tracked` 字典的访问无需加锁
- **简单可靠**：状态流转清晰，易于推理
- **Celery 负责并发**：实际文档解析在 Celery worker 中并发执行，监控进程只负责任务派发

---

## 五、错误隔离与容错设计

### 5.1 文件过滤器（ConsumerFilter）

在文件事件进入稳定性跟踪之前，[ConsumerFilter](file:///d:/fz/0601/solo-dogfeeding/code/69-paperless-ngx/src/documents/management/commands/document_consumer.py#L182-L284) 负责过滤掉不需要处理的文件。

#### 多层过滤逻辑

```
watchfiles 事件流
    ↓
ConsumerFilter.__call__(change, path)
    ↓
DefaultFilter (父类) 检查：
  - 隐藏文件/目录（以 . 开头）
  - ignore_dirs 中的目录
  - ignore_entity_patterns 匹配的文件
    ↓ 未被过滤
如果是目录 → 直接通过（目录本身不处理，但允许继续监视子内容）
    ↓
如果是文件 → 检查扩展名是否在 supported_extensions 中
```

#### 默认忽略列表

**文件正则**（`DEFAULT_IGNORE_PATTERNS`）：
- `^\.DS_Store$` / `^\.DS_STORE$` — macOS Finder 元数据
- `^\._.*` — macOS 资源分叉文件
- `^desktop\.ini$` — Windows 桌面配置
- `^Thumbs\.db$` — Windows 缩略图缓存

**目录名**（`DEFAULT_IGNORE_DIRS`）：
- `.stfolder` / `.stversions` — Syncthing 同步目录
- `.localized` / `.Spotlight-V100` / `.Trashes` / `__MACOSX` — macOS 系统目录
- `@eaDir` — Synology NAS 索引目录

#### 扩展名检查

```python
def _has_supported_extension(self, path: Path) -> bool:
    suffix = path.suffix.lower()  # 大小写不敏感
    return suffix in self._supported_extensions
```

支持的扩展名来自 [get_supported_file_extensions()](file:///d:/fz/0601/solo-dogfeeding/code/69-paperless-ngx/src/documents/parsers/__init__.py)，由各个 Parser 注册。

### 5.2 入队阶段错误隔离

[_consume_file()](file:///d:/fz/0601/solo-dogfeeding/code/69-paperless-ngx/src/documents/management/commands/document_consumer.py#L308-L353) 中每个环节都有独立的异常处理：

```python
def _consume_file(...) -> None:
    # 1. 文件存在性检查
    try:
        if not filepath.is_file():
            return
    except OSError as e:
        logger.warning(f"Not consuming {filepath}: {e}")
        return

    # 2. 标签生成（独立 try-except，失败不影响主流程）
    tag_ids: list[int] | None = None
    if subdirs_as_tags:
        try:
            tag_ids = _tags_from_path(filepath, consumption_dir)
        except Exception:
            logger.exception(f"Error creating tags from path for {filepath}")
            # 注意：这里不 return，tag_ids 保持 None，继续入队

    # 3. 任务提交
    try:
        consume_file.apply_async(...)
    except Exception:
        logger.exception(f"Error while queuing document {filepath}")
        # 单个文件入队失败不影响其他文件
```

关键设计：**标签生成失败不会阻止文件入队**，只是不带标签处理。这确保了即使数据库暂时不可用，文件也不会丢失。

### 5.3 监控循环整体容错

外层 `while` 循环的异常处理（[document_consumer.py:575-577](file:///d:/fz/0601/solo-dogfeeding/code/69-paperless-ngx/src/documents/management/commands/document_consumer.py#L575-L577)）：

```python
while not self.stop_flag.is_set():
    try:
        for changes in watch(...):
            # ... 处理事件
            break
        # ... 调整超时
    except KeyboardInterrupt:
        logger.info("Received interrupt, stopping consumer")
        self.stop_flag.set()
```

注意：这里只捕获了 `KeyboardInterrupt`。如果 watchfiles 内部抛出其他异常（如文件系统错误），监控进程会崩溃。生产环境中通常由 systemd/supervisor 负责自动重启。

### 5.4 子目录标签生成的数据库容错

[_tags_from_path()](file:///d:/fz/0601/solo-dogfeeding/code/69-paperless-ngx/src/documents/management/commands/document_consumer.py#L287-L305) 在执行前主动关闭旧连接：

```python
def _tags_from_path(filepath: Path, consumption_dir: Path) -> list[int]:
    db.close_old_connections()  # 防止使用失效的数据库连接
    tag_ids: set[int] = set()
    path_parts = filepath.relative_to(consumption_dir).parent.parts
    for part in path_parts:
        tag, _ = Tag.objects.get_or_create(
            name__iexact=part,
            defaults={"name": part},
        )
        tag_ids.add(tag.pk)
    return list(tag_ids)
```

这是因为该函数可能在长时间运行的监控线程中被调用，数据库连接可能已超时断开。

---

## 六、启动流程与两种运行模式

### Oneshot 模式（一次性处理）

```
Command.handle()
    ↓
验证消费目录存在
    ↓
_process_existing_files() 用 glob 扫描现有文件
    ↓
每个文件经 ConsumerFilter 过滤后调用 _consume_file()
    ↓
直接退出，不启动监控
```

适用于容器启动时的初始化、手动触发批量导入等场景。

### Watch 模式（持续监控）

```
Command.handle()
    ↓
先执行 _process_existing_files() 处理启动时已存在的文件
    ↓
_watch_directory() 进入无限循环
    ↓
┌─────────────────────────────────────┐
│  watch() 阻塞等待文件系统事件        │
│  ↓                                   │
│  每个事件 → tracker.track()          │
│  ↓                                   │
│  tracker.get_stable_files()          │
│  ↓                                   │
│  每个稳定文件 → _consume_file()      │
│  ↓                                   │
│  根据 tracker 状态调整 timeout       │
└─────────────────────────────────────┘
```

关键点：**启动时先处理已有文件，再进入监控**，避免遗漏。

---

## 七、配置参数汇总

| 配置项 | 环境变量 | 默认值 | 说明 |
|--------|----------|--------|------|
| `CONSUMPTION_DIR` | `PAPERLESS_CONSUMPTION_DIR` | - | 消费目录路径 |
| `CONSUMER_RECURSIVE` | `PAPERLESS_CONSUMER_RECURSIVE` | `False` | 是否递归监视子目录 |
| `CONSUMER_POLLING_INTERVAL` | `PAPERLESS_CONSUMER_POLLING_INTERVAL` | `0` | 轮询间隔秒数，0=使用原生事件 |
| `CONSUMER_STABILITY_DELAY` | `PAPERLESS_CONSUMER_STABILITY_DELAY` | `5` | 文件稳定等待时间（秒） |
| `CONSUMER_IGNORE_PATTERNS` | `PAPERLESS_CONSUMER_IGNORE_PATTERNS` | `[]` | 额外忽略的文件名正则（JSON 数组） |
| `CONSUMER_IGNORE_DIRS` | `PAPERLESS_CONSUMER_IGNORE_DIRS` | `[]` | 额外忽略的目录名（JSON 数组） |
| `CONSUMER_SUBDIRS_AS_TAGS` | `PAPERLESS_CONSUMER_SUBDIRS_AS_TAGS` | `False` | 是否将子目录名作为标签 |
| `CONSUMER_DELETE_DUPLICATES` | `PAPERLESS_CONSUMER_DELETE_DUPLICATES` | `False` | 是否自动删除检测到的重复文件 |

---

## 八、完整流程图

```
文件出现在 consume 目录
    ↓
┌─ 文件系统事件（watchfiles）────────────────────┐
│  Change.added / Change.modified                │
└────────────────────────────────────────────────┘
    ↓
ConsumerFilter 过滤
    ├─ 系统文件/隐藏文件/忽略目录 → 丢弃
    ├─ 不支持的扩展名 → 丢弃
    └─ 有效文件 → 继续
    ↓
FileStabilityTracker.track()
    ├─ 路径已在跟踪中 → 更新 last_event_time 和 stat
    └─ 新路径 → 创建 TrackedFile 条目
    ↓
等待 stability_delay 时间
    ↓
FileStabilityTracker.get_stable_files()
    ├─ 时间未到 → 继续等待
    ├─ 文件已删除 → 移除跟踪
    ├─ stat 变化了 → 重置计时器重新等待
    └─ 文件稳定 → 从跟踪移除，向下传递
    ↓
_consume_file()
    ├─ 二次检查文件存在 → 不存在则丢弃
    ├─ 生成子目录标签（失败不影响）
    └─ 提交 Celery 任务 consume_file.apply_async()
    ↓
Celery Worker 执行 consume_file 任务
    ↓
ConsumerPreflightPlugin
    ├─ pre_check_file_exists() → 文件仍在？
    └─ pre_check_duplicate() → SHA256 查重
        ├─ 重复 + CONSUMER_DELETE_DUPLICATES → 删除文件，抛异常
        └─ 不重复 → 继续
    ↓
ConsumerPlugin.run() 执行解析、分类、存储
    ↓
消费成功 → 删除 consume 目录中的源文件
```
