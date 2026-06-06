# Paperless-ngx Storage Path 与文件命名规则代码分析

## 一、整体目录结构配置

所有文件存储的根目录配置在 [paperless/settings/__init__.py](src/paperless/settings/__init__.py#L65-L88) 中：

```
MEDIA_ROOT/                    # 媒体根目录 (PAPERLESS_MEDIA_ROOT)
└── documents/
    ├── originals/             # ORIGINALS_DIR - 原始文件目录
    ├── archive/               # ARCHIVE_DIR - 归档PDF/A目录
    └── thumbnails/            # THUMBNAIL_DIR - 缩略图目录
```

相关配置项：
- `ORIGINALS_DIR` = `MEDIA_ROOT / "documents" / "originals"` - 存放用户上传的原始文件
- `ARCHIVE_DIR` = `MEDIA_ROOT / "documents" / "archive"` - 存放转换后的 PDF/A 归档文件
- `THUMBNAIL_DIR` = `MEDIA_ROOT / "documents" / "thumbnails"` - 存放 WebP 格式缩略图
- `MEDIA_LOCK` = `MEDIA_ROOT / "media.lock"` - 文件锁，用于多线程/进程同步
- `FILENAME_FORMAT` - 全局文件名格式模板（环境变量 `PAPERLESS_FILENAME_FORMAT`）
- `FILENAME_FORMAT_REMOVE_NONE` - 是否移除模板中的 `-none-` 占位符

---

## 二、StoragePath 模型

[documents/models.py](src/documents/models.py#L147-L155) 中定义的 `StoragePath` 模型：

```python
class StoragePath(MatchingModel):
    path = models.TextField(_("path"))
```

- 继承自 `MatchingModel`，支持匹配规则（自动分配给文档）
- `path` 字段存储 Jinja2 模板字符串，用于为每个文档生成相对路径
- `Document` 模型通过外键 `storage_path` 关联到 StoragePath

### 2.1 MatchingModel 基类与匹配规则详解

StoragePath 的自动分配能力来源于其基类 `MatchingModel`，定义在 [documents/models.py](src/documents/models.py#L46-L93)：

```python
class MatchingModel(ModelWithOwner):
    MATCH_NONE = 0       # 不匹配
    MATCH_ANY = 1        # 任意词匹配
    MATCH_ALL = 2        # 所有词匹配
    MATCH_LITERAL = 3    # 精确字符串匹配
    MATCH_REGEX = 4      # 正则表达式匹配
    MATCH_FUZZY = 5      # 模糊匹配
    MATCH_AUTO = 6       # 机器学习自动分类

    name = models.CharField(max_length=128)
    match = models.CharField(max_length=256, blank=True)
    matching_algorithm = models.PositiveSmallIntegerField(default=MATCH_ANY)
    is_insensitive = models.BooleanField(default=True)
```

核心匹配函数 `matches()` 位于 [documents/matching.py](src/documents/matching.py#L169-L263)，按匹配算法分别处理：

| 算法 | 匹配逻辑 |
|------|---------|
| `MATCH_NONE` | 直接返回 False |
| `MATCH_ANY` | 关键词按空格/引号分词，文档内容包含**任意一个**词即匹配 |
| `MATCH_ALL` | 文档内容必须**包含所有**分词 |
| `MATCH_LITERAL` | 精确字符串匹配（加 `\b` 词边界） |
| `MATCH_REGEX` | `match` 字段作为正则表达式执行 |
| `MATCH_FUZZY` | rapidfuzz `partial_ratio ≥ 90` 即匹配 |
| `MATCH_AUTO` | 此处返回 False，实际匹配在 `match_storage_paths()` 中结合分类器预测 |

### 2.2 match_storage_paths() — StoragePath 专用匹配函数

位于 [documents/matching.py](src/documents/matching.py#L137-L166)：

```python
def match_storage_paths(document, classifier, user=None):
    pred_id = classifier.predict_storage_path(document.suggestion_content) if classifier else None
    # 按用户权限过滤 StoragePath 列表
    return list(filter(
        lambda o: matches(o, document)
                  or (o.pk == pred_id and o.matching_algorithm == MATCH_AUTO),
        storage_paths
    ))
```

匹配条件是 **OR** 关系：
1. 基于内容/关键词的规则匹配成功（`matches()` 返回 True）
2. 或该 StoragePath 设置为 `MATCH_AUTO` 且分类器预测的 ID 与之匹配

### 2.3 set_storage_path() — 分配执行函数

位于 [documents/signals/handlers.py](src/documents/signals/handlers.py#L280-L338)：

```python
def set_storage_path(sender, document, *, logging_group=None, classifier=None,
                     replace=False, use_first=True, dry_run=False, **kwargs):
    if document.storage_path and not replace:
        return None                           # 已有值且不覆盖则跳过
    potential_storage_paths = match_storage_paths(document, classifier)
    # 多匹配时：use_first=True 取第一个，否则不分配
    selected = potential_storage_paths[0] if potential_storage_paths else None
    if not dry_run:
        document.storage_path = selected
        document.save(update_fields=("storage_path",))   # ← 触发 post_save
    return selected
```

关键点：
- `replace=False`（默认）时，若文档已有关联 StoragePath，**不会被覆盖**
- 多个 StoragePath 同时匹配时，按 `use_first` 决定取第一个还是不分配
- `document.save(update_fields=("storage_path",))` 会**触发 `post_save` 信号**，进而可能触发重命名

---

## 三、StoragePath 的六大分配入口

StoragePath 分配到 Document 有六条独立的代码路径，每条最终都会通过不同机制触发文件重命名：

```
┌─────────────────────────────────────────────────────────────────────┐
│                    StoragePath 分配入口总览                            │
├──────────────┬──────────────────────────┬───────────────────────────┤
│ 入口         │ 分配函数                 │ 触发重命名机制             │
├──────────────┼──────────────────────────┼───────────────────────────┤
│ ① 消费前 Workflow │ apply_assignment_to_overrides │ _store() 前写入 metadata │
│ ② 消费完成信号   │ set_storage_path()     │ document.save() → post_save │
│ ③ 消费后 Workflow │ apply_assignment_to_document │ document.save() → post_save │
│ ④ 手动/API 编辑  │ Document.save()        │ post_save 信号              │
│ ⑤ 批量编辑       │ bulk_edit.set_storage_path() │ bulk_update_documents 任务 │
│ ⑥ retagger 命令  │ set_storage_path()     │ document.save() → post_save │
└──────────────┴──────────────────────────┴───────────────────────────┘
```

---

### 入口 ①：消费前 Workflow（CONSUMPTION 触发器）

**时序位置：文档落盘之前，解析阶段之前**

#### 代码路径

1. 插件注册：`WorkflowTriggerPlugin` 在 [documents/consumer.py](src/documents/consumer.py#L67-L88) 的 `run()` 方法中最先执行
2. 调用 `run_workflows(trigger_type=CONSUMPTION)` → [handlers.py](src/documents/signals/handlers.py#L854-L999)
3. 命中 WorkflowAction.ASSIGNMENT 时执行 [apply_assignment_to_overrides()](src/documents/workflows/mutations.py#L113-L192)：

```python
if action.assign_storage_path:
    overrides.storage_path_id = action.assign_storage_path.pk
```

4. Consumer 的 `_store()` 方法中 [apply_overrides()](src/documents/consumer.py#L877-L938)：

```python
if self.metadata.storage_path_id:
    document.storage_path = StoragePath.objects.get(pk=self.metadata.storage_path_id)
```

5. `document.save()` 时 storage_path 已写入数据库，紧接着进入下一步"消费完成信号"

#### 对命名的影响

由于在 `_store()` 阶段就已设置好 `storage_path`，当 consumer 首次调用 `generate_unique_filename(document)` 时（见 [consumer.py](src/documents/consumer.py#L671)），就能直接用 StoragePath 的模板生成路径——**减少一次文件移动**。

---

### 入口 ②：消费完成信号 document_consumption_finished

**时序位置：`_store()` 创建 Document 之后，文件落盘之前**

#### 信号注册

在 [documents/apps.py](src/documents/apps.py#L10-L33) 的 `ready()` 中按如下顺序连接：

```python
document_consumption_finished.connect(add_inbox_tags)           # 1
document_consumption_finished.connect(set_correspondent)        # 2
document_consumption_finished.connect(set_document_type)        # 3
document_consumption_finished.connect(set_tags)                 # 4
document_consumption_finished.connect(set_storage_path)         # 5 ← 我们关心的
document_consumption_finished.connect(add_to_index)             # 6
document_consumption_finished.connect(run_workflows_added)      # 7
document_consumption_finished.connect(add_or_update_document_in_llm_index)  # 8
```

#### 代码执行路径

Consumer 在 [consumer.py](src/documents/consumer.py#L658-L666) 发送信号：

```python
document_consumption_finished.send(
    sender=self.__class__,
    document=document,
    logging_group=self.logging_group,
    classifier=classifier,      # ← 带训练好的分类器
    ...
)
```

`set_storage_path()` 使用传入的 `classifier` 做 MATCH_AUTO 预测 + 规则匹配，执行 `document.save(update_fields=("storage_path",))`。

#### 对命名的影响

此时文件**尚未写入 ORIGINALS_DIR**（仍在 MEDIA_LOCK 外等待）。consumer 下一步获得锁后才会调用 `generate_unique_filename()`，所以 StoragePath 的模板同样能在首次落盘时生效，无需额外移动。

> 注意：若 入口①（消费前 Workflow）已设置了 storage_path，`set_storage_path()` 因默认 `replace=False` 会直接 `return None`，不会覆盖。

---

### 入口 ③：消费后 Workflow（DOCUMENT_ADDED 触发器）

**时序位置：消费完成信号的最后一个 handler**

信号链第 7 个 `run_workflows_added` 会触发 [run_workflows(DOCUMENT_ADDED)](src/documents/signals/handlers.py#L803-L817)，命中 ASSIGNMENT 动作时执行 [apply_assignment_to_document()](src/documents/workflows/mutations.py#L16-L111)：

```python
if action.assign_storage_path:
    document.storage_path = action.assign_storage_path
```

在 [handlers.py](src/documents/signals/handlers.py#L975-L984)，run_workflows 末尾会显式保存：

```python
document.save(update_fields=[
    "title", "correspondent", "document_type", "storage_path",
    "owner", "modified",
])
```

#### 对命名的影响

**关键点：此时文件可能已经落盘到临时路径。**

整个消费流程在同一个数据库事务中（[consumer.py](src/documents/consumer.py#L587) 的 `with transaction.atomic()`）。run_workflows_added 执行完 `document.save()` 后，consumer 继续执行到获取 MEDIA_LOCK，再调用 `generate_unique_filename(document)` 时读取到的已是最新 storage_path，**仍然可以首次落盘即用正确路径**。

但若 Workflow 触发的是文档**更新**（DOCUMENT_UPDATED 触发器，已有文件），则 save() 触发的 `post_save` → `update_filename_and_move_files()` 会执行真实的文件移动。

---

### 入口 ④：手动编辑 / REST API

**时序位置：任意时刻用户通过前端或 API 修改**

API 层在 [documents/views.py](src/documents/views.py) 中通过标准 DRF serializer 保存 Document。当 `storage_path` 字段被修改时，`Document.save()` 触发 `post_save` 信号。

#### 对命名的影响

由 [update_filename_and_move_files()](src/documents/signals/handlers.py#L434-L668)（已注册为 `post_save` receiver）统一处理：
- 重新计算 `generate_filename()`
- 对比旧路径，必要时 `shutil.move()` 移动原始文件和归档文件
- 清理空目录

---

### 入口 ⑤：批量编辑 Bulk Edit

**时序位置：用户通过批量编辑 API 修改多个文档的 StoragePath**

[documents/bulk_edit.py](src/documents/bulk_edit.py#L134-L153)：

```python
def set_storage_path(doc_ids, storage_path):
    qs = Document.objects.filter(Q(id__in=doc_ids) & ~Q(storage_path=storage_path))
    affected_docs = list(qs.values_list("pk", flat=True))
    qs.update(storage_path=storage_path)             # ← 用 QuerySet.update，不触发 post_save

    bulk_update_documents.apply_async(               # ← 抛到异步任务
        kwargs={"document_ids": affected_docs},
        headers={"trigger_source": PaperlessTask.TriggerSource.SYSTEM},
    )
    return "OK"
```

关键点：`QuerySet.update()` **不发送 Django signals**，所以必须手动触发后续流程——通过 Celery 任务 `bulk_update_documents` 逐个对文档触发信号 → `update_filename_and_move_files()`。

#### 批量编辑完整时序：API → 异步任务 → 两次重命名

**阶段 1：API 层（同步执行）**

[documents/views.py](src/documents/views.py#L2733-L2776) 的 `_execute_document_action()` 处理批量编辑请求：

```
HTTP 请求 → BulkEditViewSet
  └─ _resolve_document_ids()           解析待处理的文档 ID
  └─ _has_document_permissions()       鉴权检查
  └─ method(documents, **parameters)   调用 bulk_edit.set_storage_path()
       └─ QuerySet.update(storage_path=X)   ← 直接写 DB，无信号
       └─ bulk_update_documents.apply_async(document_ids=affected_docs)
```

所有会修改文档元数据的批量操作（`set_correspondent` / `set_storage_path` / `set_document_type` / `add_tag` / `remove_tag` / `modify_tags` / `modify_custom_fields` / `set_permissions`）最终都会调用 `bulk_update_documents.apply_async()`，**每个操作都单独触发一次异步任务**。

**阶段 2：Celery Worker（异步执行 bulk_update_documents）**

[documents/tasks.py](src/documents/tasks.py#L253-L275)：

```python
@shared_task
def bulk_update_documents(document_ids) -> None:
    documents = Document.objects.filter(id__in=document_ids)
    for doc in documents:
        clear_document_caches(doc.pk)                      # ① 清缓存
        document_updated.send(                              # ② 发送变更通知信号
            sender=None, document=doc, logging_group=uuid.uuid4(),
        )
        post_save.send(Document, instance=doc, created=False)  # ③ 手动发 post_save

    with get_backend().batch_update() as batch:             # ④ 批量更新搜索索引
        for doc in documents:
            batch.add_or_update(doc)

    if ai_config.llm_index_enabled:                          # ⑤ 可选：更新 LLM 索引
        update_llm_index(rebuild=False)
```

对**每个文档**按顺序发送两个信号：先 `document_updated`，后 `post_save`。

---

#### ② `document_updated` 信号的接收者与执行顺序

信号接收者在 [documents/apps.py](src/documents/apps.py#L32-L33) 注册，按连接顺序执行：

```
document_updated.connect(run_workflows_updated)          # 第 1 位
document_updated.connect(send_websocket_document_updated) # 第 2 位
```

**接收者 1：run_workflows_updated → run_workflows(DOCUMENT_UPDATED)**

[documents/signals/handlers.py](src/documents/signals/handlers.py#L819-L829) → [run_workflows()](src/documents/signals/handlers.py#L854-L998)：

```
1. document.refresh_from_db()    ← 从 DB 重新拉取，避免并发覆盖
2. matching.document_matches_workflow()  判断哪些 Workflow 命中
3. 对每个命中的 Workflow 按 action.order 执行：
   ├─ ASSIGNMENT → apply_assignment_to_document()  可能再次修改 storage_path
   ├─ REMOVAL    → apply_removal_to_document()     可能清空 storage_path
   ├─ EMAIL / WEBHOOK / PASSWORD_REMOVAL / MOVE_TO_TRASH
   └─ ...
4. document.save(
       update_fields=[
           "title", "correspondent", "document_type",
           "storage_path", "owner", "modified",
       ]                           ← ★ 重要：刻意排除 filename / archive_filename
   )
   └─ 触发 @receiver(models.signals.post_save, sender=Document)
      → update_filename_and_move_files()           ★ 第一次重命名
```

**关键设计细节** [handlers.py#L966-L984](src/documents/signals/handlers.py#L966-L984)：
> `update_fields` 刻意**不包含 `filename` 和 `archive_filename`**——因为这两个字段由 `update_filename_and_move_files` 独占管理。如果这里把内存中旧值写回 DB，会覆盖并发重命名任务刚写入的新路径，导致 DB 指向旧路径而文件已在新路径（issue #12386）。

**接收者 2：send_websocket_document_updated**

[documents/signals/handlers.py](src/documents/signals/handlers.py#L832-L851)：
- 先 `document.refresh_from_db()` 确保拿到 Workflow 执行后最新的 DB 状态
- 通过 DocumentsStatusManager 发送 WebSocket 消息通知前端刷新

---

#### ③ `post_save.send()` 触发第二次重命名

`bulk_update_documents` 在 `document_updated` 的所有接收者执行完毕后，显式调用：

```python
post_save.send(Document, instance=doc, created=False)
```

这个信号匹配 [handlers.py#L431-L433](src/documents/signals/handlers.py#L431-L433) 的装饰器：

```python
@receiver(models.signals.post_save, sender=CustomFieldInstance, weak=False)
@receiver(models.signals.m2m_changed, sender=Document.tags.through, weak=False)
@receiver(models.signals.post_save, sender=Document, weak=False)   # ← 命中这个
def update_filename_and_move_files(sender, instance, **kwargs):
```

触发 **第二次重命名** `update_filename_and_move_files()`，对 Workflow 执行后可能再次变化的 `storage_path` / `correspondent` / `document_type` / `tags` / 自定义字段等做最终兜底计算。

---

#### 批量编辑完整时序总结图

```
用户在前端执行"批量设置 StoragePath"
  │
  ▼
views._execute_document_action()          [API 线程，同步]
  │
  ├─ bulk_edit.set_storage_path(doc_ids, sp)
  │    ├─ QS.update(storage_path=sp)       ── DB 已变更，无信号
  │    └─ bulk_update_documents.apply_async()
  │
  ▼  HTTP 200 返回前端（用户看到"操作成功"）
─────────────────────────────────────────────────────────────
  │
  ▼  [Celery Worker 线程，异步]
tasks.bulk_update_documents(document_ids)
  │
  └─ for each doc:
       │
       ├─ clear_document_caches(doc.pk)
       │
       ├─ document_updated.send(doc)    ── 第 1 个信号
       │    │
       │    ├─ [接收者 1] run_workflows_updated
       │    │    ├─ doc.refresh_from_db()
       │    │    ├─ 执行匹配的 DOCUMENT_UPDATED Workflow
       │    │    │   └─ apply_assignment_to_document()  可能改 storage_path
       │    │    ├─ doc.save(update_fields=[...])   ── 不含 filename!
       │    │    │   └─ post_save → update_filename_and_move_files()  ★ 第 1 次重命名
       │    │    └─ WorkflowRun.objects.create(...)
       │    │
       │    └─ [接收者 2] send_websocket_document_updated
       │         ├─ doc.refresh_from_db()
       │         └─ WebSocket 通知前端
       │
       └─ post_save.send(Document, instance=doc)   ── 第 2 个信号
            └─ update_filename_and_move_files()             ★ 第 2 次重命名（兜底）
```

#### 关键结论

| 问题 | 结论 |
|------|------|
| 为什么重命名执行两次？ | 第一次在 Workflow 内部 save 时（应对 Workflow 对元数据的二次修改），第二次在 bulk_update_documents 末尾兜底（应对 Workflow 不存在或未修改的情况） |
| 为什么不直接在 bulk_edit 里 `doc.save()`？ | 为了**先跑 Workflow**：让 DOCUMENT_UPDATED 触发器的 Workflow 有机会进一步修改元数据，再统一做重命名 |
| `document_updated` 和 `post_save` 谁先执行？ | `document_updated` 先（含 Workflow + WebSocket），全部完成后才发 `post_save` |
| Workflow 能覆盖批量编辑的 StoragePath 吗？ | **可以**——如果 DOCUMENT_UPDATED Workflow 的 ASSIGNMENT 动作设置了 `assign_storage_path`，会覆盖批量编辑设置的值（Workflow 的 order 越靠后越晚生效） |
| 两次重命名会不会冲突？ | 不会——都通过 `MEDIA_LOCK` 互斥，且第二次用 `document.refresh_from_db()` 从 DB 读到的是最新状态 |
| 前端何时拿到通知？ | `document_updated` 的第 2 个接收者发 WebSocket，**在第一次重命名之后、第二次重命名之前**。但前端收到后刷新时第二次重命名通常已完成（同一进程顺序执行） |

---

### 入口 ⑥：document_retagger 管理命令

**时序位置：管理员手动执行重跑匹配**

[documents/management/commands/document_retagger.py](src/documents/management/commands/document_retagger.py#L317-L328)：

```python
if do_storage_path:
    storage_path = set_storage_path(
        None, document, classifier=classifier,
        replace=overwrite, use_first=use_first, dry_run=suggest,
    )
```

带 `--storage_path` 参数运行时，对每个文档调用与信号相同的 `set_storage_path()` 函数，内部 `document.save(update_fields=("storage_path",))` 自然触发 `post_save` → 文件重命名。

---

## 四、消费流程中 StoragePath → 命名 → 重命名的完整时序

以下是一个新文档被消费时，StoragePath 分配与文件命名之间精确到代码行的时序：

```
Consumer.run()
  │
  ├─ [408] WorkflowTriggerPlugin.run()                         入口①
  │    └─ run_workflows(CONSUMPTION)
  │         └─ apply_assignment_to_overrides()
  │              → overrides.storage_path_id = X
  │
  ├─ [500-553] 解析文档，提取文本/日期/缩略图/归档文件
  │
  ├─ transaction.atomic() ─────────────────────────────────────┐
  │                                                              │
  ├─ [644] _store()                                             │
  │    ├─ [860] Document.objects.create(                        │
  │    │       filename=None,  ← 注意：此时 filename 为空        │
  │    │       storage_path=None,  ← 未设置                     │
  │    │       ...)                                             │
  │    ├─ [871] apply_overrides(document)                       │
  │    │    └─ [892] if metadata.storage_path_id:               │
  │    │            document.storage_path = StoragePath(...)    │
  │    └─ [873] document.save()                                 │
  │         └─ post_save → update_filename_and_move_files()     │
  │              └─ [465] if not instance.filename: return      │
  │                 ↑ filename 仍为空，提前返回！不做任何操作     │
  │                                                              │
  ├─ [658] document_consumption_finished.send()                 │ 入口②
  │    ├─ handlers.set_correspondent()                          │
  │    ├─ handlers.set_document_type()                          │
  │    ├─ handlers.set_tags()                                   │
  │    ├─ handlers.set_storage_path()                           │
  │    │    └─ 若 document.storage_path 已有值（入口①设置的）     │
  │    │       且 replace=False → return None                    │
  │    │    └─ 否则匹配并设置 storage_path                       │
  │    │       document.save(update_fields=("storage_path",))   │
  │    │         └─ post_save → update_filename_and_move_files  │
  │    │              └─ filename 仍为空 → return               │
  │    │                                                         │
  │    └─ handlers.run_workflows_added()                        │ 入口③
  │         └─ apply_assignment_to_document()                   │
  │            document.save(update_fields=["storage_path"...]) │
  │              └─ post_save → update_filename_and_move_files  │
  │                 └─ filename 仍为空 → return                  │
  │                                                              │
  ├─ [670] with FileLock(MEDIA_LOCK):                            │
  │    │                                                         │
  │    ├─ [671] generated_filename =                             │
  │    │       generate_unique_filename(document)               │ ★ 首次命名
  │    │    └─ generate_filename()                               │
  │    │         └─ if context_doc.storage_path is not None:    │
  │    │              filename_format = storage_path.path       │ ← 读取模板
  │    │         └─ format_filename() → 渲染 Jinja2             │
  │    │                                                         │
  │    ├─ [684] create_source_path_directory(source_path)       │
  │    ├─ [686] _write(working_copy, source_path)               │ ★ 写入原始文件
  │    │                                                         │
  │    ├─ [693] _write(thumbnail, thumbnail_path)               │ ★ 写入缩略图
  │    │                                                         │
  │    └─ [698] if archive_path:                                │
  │         ├─ generate_unique_filename(archive=True)           │ ★ 归档命名
  │         └─ _write(archive_path, document.archive_path)      │ ★ 写入归档
  │                                                              │
  ├─ [729] document.save()  ← 离开 MEDIA_LOCK 后保存            │
  │    └─ post_save → update_filename_and_move_files()          │ ★ 二次检查/重命名
  │         └─ [465] if not instance.filename: return → NO      │
  │            filename 已不为空，执行完整流程：                   │
  │            1) 从 DB 刷新数据                                 │
  │            2) generate_filename() 重新计算（可能变了）         │
  │            3) validate_move 安全检查                         │
  │            4) shutil.move() 移动文件（如需要）                │
  │            5) delete_empty_directories() 清理                │
  │                                                              │
  └─ transaction.commit() ──────────────────────────────────────┘
```

### 时序关键结论

1. **filename 为空是重要的"门禁"**：`update_filename_and_move_files()` [handlers.py#L465](src/documents/signals/handlers.py#L465) 开头检查 `if not instance.filename: return`。这保证了消费过程中多次 `document.save()`（存储路径、标签等更新）不会反复尝试移动不存在的文件。

2. **StoragePath 越早设置越高效**：入口①（消费前 Workflow）和入口②（消费完成信号）都是在文件落盘前设置 StoragePath，因此 `generate_unique_filename()` 可直接使用模板生成正确路径，**零文件移动**。

3. **document.save() 离开锁后再触发一次重命名**：[consumer.py#L729](src/documents/consumer.py#L729) 的 save 是有意为之——释放 MEDIA_LOCK 后让 `update_filename_and_move_files()` 以"最终状态"（所有元数据都已稳定）做一次最终的文件名计算和必要的移动。

4. **MEDIA_LOCK 保证互斥**：consumer 内落盘和 `update_filename_and_move_files()` 都获取同一个 `MEDIA_LOCK`，防止并发移动冲突。

---

## 五、路径模板（Path Template）实现机制

### 5.1 模板引擎配置

模板系统基于 Jinja2 的沙箱环境，定义在 [documents/templating/environment.py](src/documents/templating/environment.py)：

```python
class JinjaEnvironment(SandboxedEnvironment):
    def is_safe_callable(self, obj):
        # 阻止访问 .save() / .delete() / .update() 方法
        ...

_template_environment = JinjaEnvironment(
    trim_blocks=True,
    lstrip_blocks=True,
    autoescape=False,
    extensions=["jinja2.ext.loopcontrols"],
)
```

### 5.2 自定义 FilePathTemplate 类

在 [documents/templating/filepath.py](src/documents/templating/filepath.py#L34-L52) 中定义：

```python
class FilePathTemplate(Template):
    def render(self, *args, **kwargs) -> str:
        def clean_filepath(value: str) -> str:
            # 1. 移除换行符
            # 2. 移除斜杠前后的多余空格
            # 3. 去除首尾分隔符（确保是相对路径）
            ...
        return clean_filepath(original_render)
```

### 5.3 模板可用变量

模板上下文通过多个函数构建，位于 [documents/templating/filepath.py](src/documents/templating/filepath.py)：

#### 基本元数据上下文 (`get_basic_metadata_context`)

| 变量 | 说明 |
|------|------|
| `title` | 文档标题（已清理非法字符） |
| `correspondent` | 联系人名称，无则为 `-none-` |
| `document_type` | 文档类型名称，无则为 `-none-` |
| `asn` | 归档序列号，无则为 `-none-` |
| `owner_username` | 所有者用户名，无则为 `-none-` |
| `original_name` | 原始文件名（不含扩展名） |
| `doc_pk` | 文档主键（7位补零，如 `0000001`） |

#### 创建日期上下文 (`get_creation_date_context`)

| 变量 | 示例 |
|------|------|
| `created` | ISO 格式完整日期 |
| `created_year` | `2024` |
| `created_year_short` | `24` |
| `created_month` | `06` |
| `created_month_name` | `June` |
| `created_month_name_short` | `Jun` |
| `created_day` | `15` |

#### 添加日期上下文 (`get_added_date_context`)

与创建日期类似，变量前缀为 `added_`。

#### 标签上下文 (`get_tags_context`)

| 变量 | 说明 |
|------|------|
| `tag_list` | 排序后用逗号连接的标签名（已清理） |
| `tag_name_list` | 标签名称列表（可循环） |

#### 自定义字段上下文 (`get_custom_fields_context`)

```
custom_fields.<字段名>.type   - 字段类型
custom_fields.<字段名>.value  - 字段值
```

#### 完整文档对象 (`get_safe_document_context`)

通过 `document` 变量可访问：`id`, `pk`, `title`, `content`, `page_count`, `created`, `added`, `modified`, `archive_serial_number`, `mime_type`, `checksum`, `archive_checksum`, `filename`, `archive_filename`, `original_filename`, `owner`, `tags`, `correspondent`, `document_type`, `storage_path`

### 5.4 可用过滤器

注册在 [documents/templating/filepath.py](src/documents/templating/filepath.py#L101-L107)：

- `get_cf_value(custom_fields, name, default)` - 获取自定义字段值
- `datetime(format)` - 日期格式化（strftime）
- `slugify` - Django slug 化
- `localize_date(format, locale)` - Babel 本地化日期

### 5.5 模板验证与渲染

核心函数 `validate_filepath_template_and_render` 在 [filepath.py](src/documents/templating/filepath.py#L345-L412)：

1. 若无真实文档，使用 `create_dummy_document()` 创建假文档进行验证
2. 构建完整上下文字典
3. 使用 `FilePathTemplate` 渲染模板
4. 安全检查：确保渲染结果是**相对路径**且不包含 `..` 遍历

### 5.6 旧格式兼容

[documents/templating/utils.py](src/documents/templating/utils.py) 中的 `convert_format_str_to_template_format()` 将旧的 Python `{var}` 格式转换为 Jinja2 `{{ var }}` 格式。

---

## 六、文件命名生成流程

### 6.1 核心函数调用链

```
generate_unique_filename()    # file_handling.py - 生成唯一不冲突文件名
  └── generate_filename()     # file_handling.py - 根据模板生成基础文件名
        └── format_filename() # file_handling.py - 应用模板渲染 + 后处理
              └── validate_filepath_template_and_render()  # filepath.py
```

### 6.2 `generate_filename()` 详细逻辑

位于 [documents/file_handling.py](src/documents/file_handling.py#L125-L185)：

**步骤 1：确定格式字符串来源（优先级从高到低）**

1. `Document.storage_path.path` - 文档关联的 StoragePath 模板
2. `settings.FILENAME_FORMAT` - 全局配置模板（自动转换旧格式）
3. None - 使用文档 ID 默认命名

**步骤 2：渲染模板获取基础路径**

调用 `format_filename()` 渲染模板，返回相对路径（可包含子目录）。

**步骤 3：组装最终文件名**

```
最终文件名 = {基础文件名}{版本后缀}{冲突计数器}{扩展名}
```

- 版本后缀：`_v1`, `_v2` 等（仅版本文档）
- 冲突计数器：`_01`, `_02` 等（仅当发生冲突时）
- 扩展名：归档文件固定 `.pdf`，原始文件使用 `doc.file_type`

**步骤 4：无模板时的默认命名**

```
{doc_pk:07}{版本后缀}{冲突计数器}{扩展名}
例如: 0000001.pdf, 0000001_v1_01.pdf
```

### 6.3 `generate_unique_filename()` 冲突避免

位于 [documents/file_handling.py](src/documents/file_handling.py#L44-L99)：

1. 先尝试 `generate_filename(counter=0)`
2. 若目标路径已存在且不等于旧路径，counter++ 重试（`_01`, `_02`, ...）
3. 对于归档文件，优先尝试与原始文件同名的 `.pdf` 版本

### 6.4 `format_filename()` 后处理

位于 [documents/file_handling.py](src/documents/file_handling.py#L102-L122)：

- 若 `FILENAME_FORMAT_REMOVE_NONE=True`：
  - 移除 `/-none-/` 目录段
  - 移除 ` -none-` 后缀
  - 移除孤立的 `-none-`
- 向后兼容：将剩余 `-none-` 替换为 `none`

---

## 七、归档副本（Archive）管理

### 7.1 是否生成归档的判断逻辑

[documents/consumer.py](src/documents/consumer.py#L124-L189) 中的 `should_produce_archive()`：

| 条件 | 结果 |
|------|------|
| 解析器需要 PDF 渲染（如图片格式） | ✅ 生成 |
| 解析器无法生成归档（如纯文本） | ❌ 不生成 |
| `ARCHIVE_FILE_GENERATION=always` | ✅ 生成 |
| `ARCHIVE_FILE_GENERATION=never` | ❌ 不生成 |
| `auto` + 图片 MIME 类型 | ✅ 生成 |
| `auto` + 原生数字 PDF（有结构标签/文本足够） | ❌ 不生成 |
| `auto` + 扫描 PDF（无文本/文本极少） | ✅ 生成 |

### 7.2 归档文件存储路径

Document 模型属性 [models.py](src/documents/models.py#L440-L453)：

```python
@property
def has_archive_version(self) -> bool:
    return self.archive_filename is not None

@property
def archive_path(self) -> Path | None:
    if self.has_archive_version:
        return (settings.ARCHIVE_DIR / Path(str(self.archive_filename))).resolve()
```

- `archive_filename` 存储相对路径（可含子目录，由模板生成）
- `archive_path` 返回绝对路径
- 归档文件扩展名固定为 `.pdf`（PDF/A 格式）
- `archive_checksum` 存储归档文件的校验和

---

## 八、缩略图（Thumbnail）管理

### 8.1 缩略图路径规则

Document 模型属性 [models.py](src/documents/models.py#L478-L488)：

```python
@property
def thumbnail_path(self) -> Path:
    webp_file_name = f"{self.pk:07}.webp"
    webp_file_path = settings.THUMBNAIL_DIR / Path(webp_file_name)
    return webp_file_path.resolve()
```

**命名规则（固定，不使用模板）：**
```
THUMBNAIL_DIR / {doc_pk:07}.webp
例如: .../thumbnails/0000001.webp
```

特点：
- 固定使用 WebP 格式
- 命名仅依赖文档主键，7位数字补零
- **不使用路径模板**，无分级目录结构
- StoragePath 变更**不会触发缩略图移动**（与原始文件和归档文件不同）

### 8.2 缩略图生成

两种生成方式：

**方式 1：消费流程中实时生成**
[consumer.py](src/documents/consumer.py#L526-L536)
```python
thumbnail = document_parser.get_thumbnail(self.working_copy, mime_type)
# 之后写入 document.thumbnail_path
```

**方式 2：管理命令批量重新生成**
[documents/management/commands/document_thumbnails.py](src/documents/management/commands/document_thumbnails.py)
- 命令：`document_thumbnails`
- 支持 `--document <id>` 指定单个文档
- 支持多进程并行处理
- 每个解析器类实现 `get_thumbnail()` 方法

---

## 九、文件自动重命名与移动机制

### 9.1 触发信号

位于 [documents/signals/handlers.py](src/documents/signals/handlers.py#L431-L668) 的 `update_filename_and_move_files()` 通过 `@receiver` 装饰器监听三个 Django 内置信号：

```python
@receiver(models.signals.post_save, sender=CustomFieldInstance, weak=False)
@receiver(models.signals.m2m_changed, sender=Document.tags.through, weak=False)
@receiver(models.signals.post_save, sender=Document, weak=False)
def update_filename_and_move_files(sender, instance, **kwargs):
```

三个触发源：
- `post_save` (Document) - 文档保存后（最主要入口，涵盖手动编辑、retagger、消费流程最终 save、Workflow 内部 save、bulk_update_documents 手动发送）
- `m2m_changed` (Document.tags.through) - 标签增删改时（若模板中使用了 `tag_list` 等变量）
- `post_save` (CustomFieldInstance) - 自定义字段值变更时（仅当模板使用了 custom_fields，通过 `_filename_template_uses_custom_fields()` 判断）

> `weak=False` 是关键：防止 receiver 被 Python GC 过早回收。

#### `document_updated` 信号 vs Django `post_save` 信号

项目中存在两个"文档更新"相关的信号，职责不同：

| 信号 | 定义位置 | 触发方式 | 用途 |
|------|---------|---------|------|
| `document_updated` | [documents/signals/__init__.py](src/documents/signals/__init__.py) | 应用层代码**显式调用** `.send()` | 变更通知（Workflow、WebSocket），不直接触发文件操作 |
| `models.signals.post_save` | Django 内置 | Django ORM 在 `Model.save()` 后**自动发送**，或手动 `.send()` | 文件重命名/移动 |

两者的关系：批量编辑等场景中，`document_updated` 先发送（跑 Workflow + 通知前端），其内部 Workflow 的 `save()` 会自动触发 `post_save` → 第一次重命名；随后 `bulk_update_documents` 再显式发一次 `post_save` → 第二次兜底重命名。

### 9.2 模板是否依赖自定义字段的检测

[handlers.py](src/documents/signals/handlers.py#L417-L427) 避免不必要的重命名：

```python
def _filename_template_uses_custom_fields(doc):
    template = doc.storage_path.path if doc.storage_path else settings.FILENAME_FORMAT
    return template and "custom_fields" in template
```

### 9.3 执行流程

```
1. if isinstance(instance, CustomFieldInstance):
     若模板不含 custom_fields → return
     否则 instance = instance.document

2. if not instance.filename: return   ← 消费过程中的 save 均被此门禁拦截

3. 获取 FileLock(settings.MEDIA_LOCK) 锁
4. 从数据库刷新文档数据（等待锁期间可能被其他进程更新）
5. 调用 generate_filename() 计算新路径
   - 若路径超过 MAX_STORED_FILENAME_LENGTH (1024) → 抛出异常
   - 若目标已存在且校验和匹配 → 视为已移动（original_already_moved=True）
   - 若目标已存在且非同一文件 → 调用 generate_unique_filename()
6. 对原始文件和归档文件分别计算
7. validate_move() 安全检查：
   - 新路径必须在 ORIGINALS_DIR / ARCHIVE_DIR 内（不能越界）
   - 旧文件必须存在
   - 新文件不能已存在
8. create_source_path_directory() 创建父目录
9. shutil.move() 执行文件移动
10. Document.objects.filter(pk=...).update() 更新数据库
    ↑ 用 QuerySet.update 而非 save()，避免触发 post_save 导致无限递归
11. delete_empty_directories() 清理空目录
12. 若为根文档（root_document_id is None）：
    递归调用 update_filename_and_move_files() 同步所有版本文档
```

### 9.4 异常回滚

若移动或保存过程中出现异常：
1. 尝试将文件移回原位置
2. 恢复 instance 的旧 filename / archive_filename 值
3. 记录警告日志（文件不一致由 sanity_checker 后续处理）

### 9.5 目录清理

`delete_empty_directories()` 在 [file_handling.py](src/documents/file_handling.py#L15-L41)：
- 从文件所在目录开始向上遍历
- 遇到空目录则删除
- 到达根目录（ORIGINALS_DIR / ARCHIVE_DIR）停止
- 确保不越界删除根目录外的内容（通过 `is_relative_to(root)` 校验）

---

## 十、文档删除时的文件清理

[documents/signals/handlers.py](src/documents/signals/handlers.py#L342-L402) 的 `cleanup_document_deletion()`：

1. 获取 MEDIA_LOCK
2. 若配置了 EMPTY_TRASH_DIR：
   - 将原始文件移动到回收站目录（自动处理同名冲突，追加 `_01`, `_02` 等）
   - 归档文件和缩略图直接删除
3. 否则三个文件全部直接删除
4. 清理原始文件和归档文件所在的空目录

---

## 十一、核心文件索引

| 文件 | 主要职责 |
|------|---------|
| [documents/matching.py](src/documents/matching.py) | MatchingModel 规则匹配、match_storage_paths() |
| [documents/apps.py](src/documents/apps.py) | 信号连接注册（消费完成信号顺序、document_updated 接收者） |
| [documents/signals/__init__.py](src/documents/signals/__init__.py) | 自定义信号定义（document_consumption_started/finished、document_updated） |
| [documents/tasks.py](src/documents/tasks.py#L253-L275) | bulk_update_documents 异步任务（document_updated + post_save 两次信号发送） |
| [documents/workflows/mutations.py](src/documents/workflows/mutations.py) | Workflow 赋值/移除动作（含 storage_path） |
| [documents/bulk_edit.py](src/documents/bulk_edit.py) | 批量 set_storage_path 等操作 + 异步任务触发 |
| [documents/views.py](src/documents/views.py#L2733-L2776) | 批量编辑 API 入口 _execute_document_action() |
| [documents/file_handling.py](src/documents/file_handling.py) | 文件名生成、目录创建、空目录清理 |
| [documents/templating/filepath.py](src/documents/templating/filepath.py) | 模板上下文构建、渲染、验证 |
| [documents/templating/environment.py](src/documents/templating/environment.py) | Jinja2 沙箱环境配置 |
| [documents/templating/filters.py](src/documents/templating/filters.py) | 自定义模板过滤器 |
| [documents/templating/utils.py](src/documents/templating/utils.py) | 旧格式到新格式的转换 |
| [documents/signals/handlers.py](src/documents/signals/handlers.py) | set_storage_path / update_filename_and_move_files / run_workflows / 删除清理 |
| [documents/consumer.py](src/documents/consumer.py) | 消费流程时序、_store()、落盘逻辑 |
| [documents/models.py](src/documents/models.py) | Document / StoragePath / MatchingModel 定义 |
| [documents/management/commands/document_thumbnails.py](src/documents/management/commands/document_thumbnails.py) | 缩略图重新生成命令 |
| [documents/management/commands/document_retagger.py](src/documents/management/commands/document_retagger.py) | retagger 批量重新匹配命令 |
| [paperless/settings/__init__.py](src/paperless/settings/__init__.py#L65-L88) | 目录配置和全局模板配置 |
