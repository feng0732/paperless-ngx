# Paperless-ngx 文档消费流水线

本文档沿代码追踪一份文件从进入系统到最终归档的完整路径，把入口识别、预处理、OCR、入库和归档步骤逐一讲清楚。

---

## 一、全局概览

```
  ┌───────────────────────────────────────────────────────────────────┐
  │                        文档进入系统（入口）                        │
  │  消费目录监听 / API 上传 / Web UI 上传 / 邮件收取                  │
  └──────────────────────────┬────────────────────────────────────────┘
                             │ 构造 ConsumableDocument + DocumentMetadataOverrides
                             │ 调用 consume_file.apply_async()
                             ▼
  ┌───────────────────────────────────────────────────────────────────┐
  │                   Celery 异步任务: consume_file                   │
  │                  (src/documents/tasks.py#L124)                    │
  │                                                                   │
  │  依次执行插件链:                                                   │
  │    1. ConsumerPreflightPlugin   — 预检（文件存在/去重/目录就绪）    │
  │    2. AsnCheckPlugin            — ASN 唯一性与范围校验             │
  │    3. CollatePlugin             — 双面扫描整理                     │
  │    4. BarcodePlugin             — 条码识别 & 文档拆分              │
  │    5. AsnCheckPlugin (再次)     — 条码读取后重新校验 ASN           │
  │    6. WorkflowTriggerPlugin     — 消费阶段工作流覆写               │
  │    7. ConsumerPlugin            — 核心消费（MIME→解析器→OCR→入库） │
  └──────────────────────────┬────────────────────────────────────────┘
                             │
                             ▼
  ┌───────────────────────────────────────────────────────────────────┐
  │                    document_consumption_finished 信号             │
  │    → add_inbox_tags       添加收件箱标签                          │
  │    → set_correspondent    自动匹配通信方                           │
  │    → set_document_type    自动匹配文档类型                         │
  │    → set_tags             自动匹配标签                             │
  │    → set_storage_path     自动匹配存储路径                         │
  │    → add_to_index         写入搜索索引                             │
  │    → run_workflows_added  执行"文档添加"触发的工作流               │
  │    → add_or_update_document_in_llm_index  写入 LLM 索引           │
  └───────────────────────────────────────────────────────────────────┘
```

> 如果是已有文档的新版本（`root_document_id` 不为空），插件链简化为 `[ConsumerPreflightPlugin, ConsumerPlugin]`，跳过条码、双面扫描和工作流触发步骤。

---

## 二、入口识别：文档如何进入系统

文档有四种来源，对应 `DocumentSource` 枚举（[data_models.py](src/documents/data_models.py#L150-L158)）：

| 来源 | DocumentSource | 入口代码 | 触发方式 |
|---|---|---|---|
| 消费目录 | `ConsumeFolder (1)` | [document_consumer.py](src/documents/management/commands/document_consumer.py#L308-L353) | `watchfiles` 监听目录变化 |
| API 上传 | `ApiUpload (2)` | [views.py#L3127](src/documents/views.py#L3127-L3158) | REST API `POST /api/documents/post_document/` |
| 邮件收取 | `MailFetch (3)` | [mail.py#L878](src/paperless_mail/mail.py#L878-L905) | Celery 定时任务 `process_mail_accounts` |
| Web UI 上传 | `WebUI (4)` | [views.py#L3127](src/documents/views.py#L3127-L3158) | 前端拖拽/选择文件上传 |

### 2.1 消费目录监听

`document_consumer` 管理命令（[document_consumer.py](src/documents/management/commands/document_consumer.py)）是消费目录的守护进程，核心流程：

1. **启动时扫描**：`_process_existing_files()` 遍历目录中已有文件
2. **持续监听**：通过 `watchfiles.watch()` 监听文件系统事件（原生 inotify/FSEvents 或轮询回退）
3. **稳定性追踪**：`FileStabilityTracker` 确保文件写入完成后才消费（防止读到不完整的文件）
4. **扩展名过滤**：`ConsumerFilter` 只接受解析器支持的文件类型，忽略系统文件和配置的忽略模式
5. **子目录作为标签**：如果 `CONSUMER_SUBDIRS_AS_TAGS` 开启，文件所在子目录名会自动创建/匹配为标签

文件稳定后调用 `_consume_file()`，构造 `ConsumableDocument(source=DocumentSource.ConsumeFolder, ...)` 并通过 `consume_file.apply_async()` 投入 Celery 任务队列。

### 2.2 API / Web UI 上传

两个入口共用 `post_document()` 视图（[views.py#L3127](src/documents/views.py#L3127)）：

1. 上传文件写入 `SCRATCH_DIR` 下的临时文件
2. 从请求参数构造 `DocumentMetadataOverrides`（可含 title、correspondent、tags 等预设元数据）
3. 构造 `ConsumableDocument`（`source=WebUI` 或 `ApiUpload`）
4. 调用 `consume_file.apply_async()` 入队

### 2.3 邮件收取

`MailAccountFetcher.handle_mail_rule()` 处理邮件规则（[mail.py](src/paperless_mail/mail.py)）：

1. 连接 IMAP 邮箱，按规则过滤邮件
2. 附件模式：逐个附件保存为临时文件，构造 `ConsumableDocument(source=MailFetch)`
3. EML 模式：整封邮件保存为 `.eml` 文件
4. 通过 Celery `chord` 编排：所有附件消费完成后执行邮件动作（标记已读/删除/移动等）

### 2.4 核心数据模型

- **`ConsumableDocument`**（[data_models.py#L162](src/documents/data_models.py#L162-L188)）：封装待消费文件，包含来源、文件路径、关联邮件规则等。`__post_init__` 中自动解析绝对路径并检测 MIME 类型。
- **`DocumentMetadataOverrides`**（[data_models.py#L13](src/documents/data_models.py#L13-L99)）：携带元数据覆盖值（标题、通信方、标签、存储路径、自定义字段、权限等）。各插件可通过 `update()` 方法增量合并覆盖。

---

## 三、预处理：插件链逐步处理

`consume_file` 任务（[tasks.py#L124](src/documents/tasks.py#L124-L220)）是消费流水线的总调度。它按序实例化并执行插件链，每个插件遵循统一接口（[base.py](src/documents/plugins/base.py#L23-L103)）：

```
able_to_run → setup() → run() → cleanup()
```

插件可以：
- 修改 `metadata`（覆盖值会传递给后续插件和最终入库）
- 抛出 `StopConsumeTaskError` 终止整个消费任务（如条码拆分后原文件不再需要继续消费）
- 抛出 `ConsumeFileDuplicateError` 标记重复

### 3.1 ConsumerPreflightPlugin — 预检

代码：[consumer.py#L954](src/documents/consumer.py#L954-L1054)

1. **`pre_check_file_exists()`**：确认文件仍然存在
2. **`pre_check_duplicate()`**：计算文件 SHA256，在数据库中查找相同 checksum 的文档。如果 `CONSUMER_DELETE_DUPLICATES` 开启，直接删除重复文件并抛出 `ConsumeFileDuplicateError`
3. **`pre_check_directories()`**：确保 `SCRATCH_DIR`、`THUMBNAIL_DIR`、`ORIGINALS_DIR`、`ARCHIVE_DIR` 都已创建

### 3.2 AsnCheckPlugin — ASN 校验

代码：[consumer.py#L1056](src/documents/consumer.py#L1056-L1104)

检查 `metadata.asn`（档案序列号）：
- 范围校验：必须在 `[1, 4294967295]` 范围内
- 唯一性校验：不能与已有文档的 ASN 冲突

> 此插件在条码识别前后各执行一次：第一次检查用户指定的 ASN，第二次检查条码中读取到的 ASN。

### 3.3 CollatePlugin — 双面扫描整理

代码：[double_sided.py](src/documents/double_sided.py)

仅当 `CONSUMER_ENABLE_COLLATE_DOUBLE_SIDED` 开启且文件位于配置的双面子目录时激活：

1. **第一次收到文件**（奇数页/正面）：移入暂存区，抛出 `StopConsumeTaskError` 等待第二份
2. **第二次收到文件**（偶数页/反面）：反面页逆序排列，与正面逐页交错插入，合并为新 PDF
3. 合并后的 PDF 放回消费目录（不含双面子目录），会被消费目录监听器重新拾取

超时机制：暂存文件 30 分钟后自动失效。

### 3.4 BarcodePlugin — 条码识别与文档拆分

代码：[barcodes.py](src/documents/barcodes.py)

仅当条码功能开启且文件为 PDF/TIFF 时激活。处理流程：

1. **TIFF 转换**：如果是 TIFF 文件，先通过 `convert_from_tiff_to_pdf()` 转为 PDF
2. **条码检测**（`detect()`）：
   - 逐页将 PDF 转为图像（`pdf2image.convert_from_path`）
   - 可选放大图像以提高识别率
   - 使用 `zxingcpp` 读取条码
3. **条码分类**：每个条码可分为分隔符（`is_separator`）、ASN（`is_asn`）、标签（`is_tag`）
4. **标签映射**：条码值通过 `barcode_tag_mapping` 配置映射为标签 ID
5. **文档拆分**（`separate_pages()`）：
   - 找到所有分隔页位置
   - 用 `pikepdf` 按分隔位置将 PDF 拆分为多个子文档
   - 每个子文档构造新的 `ConsumableDocument` 并调用 `consume_file.apply_async()` 独立消费
   - 原文件删除，抛出 `StopConsumeTaskError` 终止当前任务
6. **ASN 应用**：如果未发生拆分，将条码中读取的 ASN 应用到 `metadata.asn`

### 3.5 WorkflowTriggerPlugin — 消费阶段工作流

代码：[consumer.py#L67](src/documents/consumer.py#L67-L87)

调用 `run_workflows(trigger_type=CONSUMPTION)`，匹配消费阶段的 Workflow：

- 工作流的 **Assignment 动作** 会修改 `metadata`（设置标题、通信方、标签、存储路径、权限等）
- 工作流的 **Removal 动作** 会移除已有覆盖值
- 这些覆盖值最终在 `ConsumerPlugin._store()` 中应用到文档记录

---

## 四、OCR 与解析：ConsumerPlugin 核心

代码：[consumer.py#L246](src/documents/consumer.py#L246-L784)

这是消费流水线中工作量最大的插件，完整流程如下：

### 4.1 文件准备

```
原始文件 → copy_file_with_basic_stats() → 工作副本 (SCRATCH_DIR/tmpdir/filename)
```

将原始文件复制到临时目录作为工作副本，避免修改原文件。

### 4.2 MIME 类型检测与 PDF 修复

```python
mime_type = magic.from_file(self.working_copy, mime=True)
```

如果文件扩展名是 `.pdf` 但 MIME 类型不正确（如 `application/octet-stream`），尝试用 `qpdf --replace-input` 修复，保存修复前的原始文件用于后续 checksum 计算。

### 4.3 解析器选择

通过 `ParserRegistry`（[registry.py](src/paperless/parsers/registry.py)）选择最合适的解析器：

1. 遍历所有注册解析器（第三方优先，然后内置）
2. 检查 `supported_mime_types()` 是否包含当前 MIME 类型
3. 调用 `score()` 获取优先级分数，分数最高者胜出

内置解析器（[registry.py#L200](src/paperless/parsers/registry.py#L200-L210)）：

| 解析器 | 类名 | 支持的 MIME 类型 | 说明 |
|---|---|---|---|
| 文本解析器 | `TextDocumentParser` | text/plain, text/csv | 直接读取文本，不做 OCR |
| 远程解析器 | `RemoteDocumentParser` | 可配置 | 将文件发送到远程 OCR 服务 |
| Tika 解析器 | `TikaDocumentParser` | Office 文档等 | 通过 Apache Tika 提取文本 |
| 邮件解析器 | `MailDocumentParser` | message/rfc822 (.eml) | 解析邮件并转为 PDF |
| **OCR 解析器** | **`RasterisedDocumentParser`** | **PDF, JPEG, PNG, TIFF, GIF, BMP, WebP, HEIC** | **使用 OCRmyPDF + Tesseract** |

### 4.4 Pre-consume 脚本

如果配置了 `PRE_CONSUME_SCRIPT`，在解析前执行该脚本，通过环境变量传递文件路径：

- `DOCUMENT_SOURCE_PATH`：原始文件路径
- `DOCUMENT_WORKING_PATH`：工作副本路径
- `TASK_ID`：任务 ID

### 4.5 解析与 OCR（RasterisedDocumentParser）

代码：[tesseract.py#L494](src/paperless/parsers/tesseract.py#L494-L659)

这是最核心的解析器，处理所有栅格化文档（PDF + 图片）。流程取决于 `OCR_MODE` 设置：

#### 模式：OFF

- 图片：直接转换为 PDF/A（`_convert_image_to_pdfa`），使用 `img2pdf` + `pikepdf` 打 PDF/A 元数据，**不调用 Tesseract**
- PDF：通过 Ghostscript 直接转 PDF/A（`_convert_pdf_to_pdfa`），不执行 OCR
- 提取已有文本（`pdftotext`）

#### 模式：AUTO（默认）

1. 先检测 PDF 是否已有文本（`is_tagged_pdf` + `extract_pdf_text`）
2. **已有文本且不需要归档文件**：直接返回已有文本，跳过 OCRmyPDF
3. **已有文本但需要归档文件**：调用 OCRmyPDF 并设置 `skip_text=True`，只做 PDF/A 转换不做 OCR
4. **无文本**：完整 OCR 流程

#### 模式：FORCE / REDO

- FORCE：对所有页面强制重新 OCR（`--force-ocr`）
- REDO：对没有文本的页面重新 OCR（`--redo-ocr`）

#### OCRmyPDF 调用参数

`construct_ocrmypdf_parameters()`（[tesseract.py#L266](src/paperless/parsers/tesseract.py#L266-L383)）构建参数字典，包括：

- 语言、输出类型（pdfa/pdfa-1/pdfa-2/pdfa-3/pdf）
- OCR 模式（force_ocr / redo_ocr / skip_text）
- 图像清理（clean / clean_final）
- 倾斜校正（deskew）
- 页面旋转检测（rotate_pages）
- 限制 OCR 页数（pages）
- sidecar 文本输出
- 图片特殊处理：DPI 检测/估算、Alpha 通道移除
- 用户自定义参数（`OCR_USER_ARGS`）

#### 文本提取

OCRmyPDF 运行完成后，文本提取优先级：

1. **sidecar 文件**：OCRmyPDF 生成的 sidecar.txt 包含纯 OCR 文本
2. **归档 PDF**：从生成的 PDF 中用 `pdftotext` 提取
3. **后处理**：`post_process_text()` 压缩多余空白、去除前导/尾随空格、替换 NULL 字符

#### 降级策略

如果常规 OCR 失败（如 `PriorOcrFoundError`、`NoTextFoundException`），会以 `safe_fallback=True`（即 `--force-ocr`）重新运行一次。如果文件是加密/签名的，直接使用已有文本。

### 4.6 归档文件生成决策

`should_produce_archive()`（[consumer.py#L124](src/documents/consumer.py#L124-L189)）决定是否生成归档 PDF：

| 条件 | 是否生成归档 |
|---|---|
| 解析器 `requires_pdf_rendition=True`（如 Office 文档） | **是** — 前端需要 PDF 来展示原始格式 |
| 解析器 `can_produce_archive=False`（如 TextDocumentParser） | **否** |
| `ARCHIVE_FILE_GENERATION=always` | **是** |
| `ARCHIVE_FILE_GENERATION=never` | **否** |
| `ARCHIVE_FILE_GENERATION=auto` + 图片文件 | **是** |
| `ARCHIVE_FILE_GENERATION=auto` + 标记 PDF（tagged PDF） | **否** — 原生数字 PDF |
| `ARCHIVE_FILE_GENERATION=auto` + 扫描 PDF（文本 ≤ 50 字符） | **是** |
| `ARCHIVE_FILE_GENERATION=auto` + 有文本的 PDF | **否** |

### 4.7 缩略图生成

每个解析器实现 `get_thumbnail()`：
- **OCR 解析器**：对归档 PDF（或原始文件）首页用 ImageMagick 生成 500px 宽的 WebP 图像，失败时回退到 Ghostscript
- **文本解析器**：用 Pillow 将文本渲染到 500×700 白底图像
- 默认缩略图：如果所有方法失败，使用内置的 `document.webp`

---

## 五、消费成功：入库事务与文件处理的精确顺序

核心逻辑在 `src/documents/consumer.py` 的 `ConsumerPlugin.run()` 方法（L408-784）。以下是**精确的执行顺序**，从 OCR 解析完成后开始追踪：

### 5.1 数据库事务开始

在进度 95% 时进入 `transaction.atomic()`（L586-587）：

```python
with transaction.atomic():
    ...  # 所有以下操作都在同一个数据库事务内
```

> **关键**：事务内的任何步骤失败都会全部回滚，包括数据库写入和文件操作。

### 5.2 创建文档记录（_store）

**代码位置**：`src/documents/consumer.py` 的 `_store()`（L815-875）

1. **确定创建日期**：优先级 `metadata.created > parser.get_date() > 文件修改时间`
2. **确定标题**：优先级 `metadata.title > metadata.filename 的文件名部分 > 原始文件名`。标题支持工作流占位符解析
3. **计算 checksum**：SHA256（如果 qpdf 修复过 PDF，用修复前的原始文件计算）
4. **`Document.objects.create()`**：写入 title、content（OCR 文本）、mime_type、checksum、page_count、created、modified、original_filename 等
5. **`apply_overrides()`**：设置通信方、文档类型、标签、存储路径、ASN、所有者、权限、自定义字段
6. **`document.save()`**：保存初始记录

> **注意**：此时 `document.filename` 字段为 `None`，这会导致 `post_save` 信号中的 `update_filename_and_move_files()` 直接返回（L465-474），不执行任何文件操作。

### 5.3 document_consumption_finished 信号发送

**代码位置**：`src/documents/consumer.py` L658-666

在**文件写入磁盘之前**发送信号：

```python
document_consumption_finished.send(
    sender=self.__class__,
    document=document,
    logging_group=self.logging_group,
    classifier=classifier,
    original_file=self.unmodified_original if self.unmodified_original else self.working_copy,
)
```

**信号处理器按以下顺序同步执行**（`src/documents/apps.py` L24-31）：

| 顺序 | 处理器 | 作用 |
|---|---|---|
| 1 | `add_inbox_tags` | 为文档添加标记为"收件箱"的标签 |
| 2 | `set_correspondent` | 用分类器 + 规则匹配自动设置通信方 |
| 3 | `set_document_type` | 用分类器 + 规则匹配自动设置文档类型 |
| 4 | `set_tags` | 用分类器 + 规则匹配自动设置标签 |
| 5 | `set_storage_path` | 用分类器 + 规则匹配自动设置存储路径 |
| 6 | `add_to_index` | 写入 Whoosh/Tantivy 搜索索引 |
| 7 | `run_workflows_added` | 执行"文档添加"触发的工作流 |
| 8 | `add_or_update_document_in_llm_index` | 写入 LLM 向量索引 |

> **关键**：这些处理器会直接修改 `document` 对象的属性（通信方、标签、存储路径等），这些修改会影响**后续的文件名生成**。如果工作流包含 `MoveToTrash` 动作，文档会被移到回收站但消费流程继续。

### 5.4 文件写入磁盘（初次放置）

**代码位置**：`src/documents/consumer.py` L670-725

在 `FileLock(settings.MEDIA_LOCK)` 保护下写入文件。此时 document 已包含信号处理器设置的属性（通信方、标签、存储路径等），文件名基于这些属性生成。

#### 原始文件

1. **`generate_unique_filename(document)`**：生成原始文件的唯一文件名
   - 返回 `Path` 对象，相对路径（不含 `ORIGINALS_DIR` 前缀）
   - 内部调用 `generate_filename(document)` 渲染模板，如果目标路径已有同名文件则加 `_01`、`_02` 后缀
2. **长度检查**：如果文件名超过 `Document.MAX_STORED_FILENAME_LENGTH`（255），回退到 `generate_filename(document, use_format=False)` 即纯数字 ID 命名
3. **`document.filename = generated_filename`**：设置到 Document 对象
4. **创建目录**：`create_source_path_directory(document.source_path)` — 递归创建父目录
5. **写入**：`self._write(source, document.source_path)` — 将工作副本写入 `ORIGINALS_DIR/<generated_filename>`

#### 缩略图

6. **写入**：`self._write(thumbnail, document.thumbnail_path)` — 写入 `THUMBNAIL_DIR/<pk:07>.webp`
   - 缩略图路径由 `Document.thumbnail_path` 属性计算（`src/documents/models.py` L479-484），固定格式 `{pk:07}.webp`，无模板渲染

#### 归档文件（如有）

7. **`generate_unique_filename(document, archive_filename=True)`**：生成归档文件名
   - 优先尝试 `<原始文件名stem>.pdf`（与原始文件同目录结构），仅当目标不存在时使用
   - 冲突时同样加 `_01`、`_02` 后缀
8. **长度检查**：同原始文件的回退策略
9. **`document.archive_filename = generated_archive_filename`**：设置到 Document 对象
10. **创建目录**：`create_source_path_directory(document.archive_path)`
11. **写入**：`self._write(archive_path, document.archive_path)` — 写入 `ARCHIVE_DIR/<generated_archive_filename>`
12. **计算归档 checksum**：`compute_checksum(document.archive_path)` → 存入 `document.archive_checksum`

> **注意**：此时 `document` 对象上 `filename` 和 `archive_filename` 已设置但**尚未 save 到数据库**，文件名尚未"最终确定"。

### 5.5 document.save() 触发文件名重算与移动

**代码位置**：`src/documents/consumer.py` L728-729

```python
# Don't save with the lock active. Saving will cause the file
# renaming logic to acquire the lock as well.
# This triggers things like file renaming
document.save()
```

特意在 `FileLock` 释放**之后**才 save，因为 `post_save` 信号处理器 `update_filename_and_move_files()` 也会获取同一个 `MEDIA_LOCK`，如果在锁内 save 会死锁。

`post_save` 信号触发 `update_filename_and_move_files()`（`src/documents/signals/handlers.py` L434-668），执行**文件名重算和文件移动**：

#### 5.5.1 前置检查

- 如果 `instance.filename` 为空 → 直接返回（消费初期 `_store()` 中的 save 会走此路径，不执行任何移动）

#### 5.5.2 原始文件名重算

1. **获取 FileLock**：防止并发操作
2. **刷新数据**：`instance.refresh_from_db()` — 等锁期间可能有其他更新
3. **计算候选文件名**：`generate_filename(instance)` — 重新渲染模板
4. **长度检查**：超过 255 字符 → 抛出 `CannotMoveFilesException`
5. **冲突判断**（三路分支）：

   | 条件 | 处理 |
   |---|---|
   | `candidate == old_filename` | 文件名未变，不需要移动 |
   | `candidate_source_path 已存在` 且 `!= old_source_path` | 检查旧文件是否已不在 + 目标文件 checksum 是否匹配：若是则 `original_already_moved=True`（文件已就位）；若否则调用 `generate_unique_filename()` 加后缀 |
   | 其他 | 使用候选文件名（正常路径） |

6. **决定是否移动**：`move_original = (old_filename != new_filename) and not original_already_moved`

#### 5.5.3 归档文件名重算（与原始文件逻辑对称）

1. **计算候选文件名**：`generate_filename(instance, archive_filename=True)` — 归档文件扩展名固定 `.pdf`
2. **长度检查**：同上
3. **冲突判断**：与原始文件完全对称的三路分支，包括 `archive_already_moved` 标志
4. **决定是否移动**：`move_archive = (old_archive_filename != new_archive_filename) and not archive_already_moved`

#### 5.5.4 快速路径：无需移动

如果 `move_original` 和 `move_archive` 都为 False：
- 用 `Document.objects.filter(pk=instance.pk).update(**updates)` 直接更新数据库（不用 `save()` 避免无限递归）
- 返回

#### 5.5.5 执行移动

```
if move_original:
    validate_move(instance, old_source_path, instance.source_path, ORIGINALS_DIR)
    create_source_path_directory(instance.source_path)
    shutil.move(old_source_path, instance.source_path)

if move_archive:
    validate_move(instance, old_archive_path, instance.archive_path, ARCHIVE_DIR)
    create_source_path_directory(instance.archive_path)
    shutil.move(old_archive_path, instance.archive_path)
```

`validate_move()` 的三个安全检查（`src/documents/signals/handlers.py` L444-463）：

| 检查 | 条件 | 结果 |
|---|---|---|
| 路径逃逸 | `new_path` 不在 `root` 下 | 抛出 `CannotMoveFilesException` |
| 源文件不存在 | `old_path` 不是文件 | 抛出 `CannotMoveFilesException`（`logger.fatal`） |
| 目标文件已存在 | `new_path` 已是文件 | 抛出 `CannotMoveFilesException` |

移动完成后，用 `Document.global_objects.filter(pk=instance.pk).update(...)` 更新数据库中的 `filename`、`archive_filename` 和 `modified`（不用 `save()` 避免递归），并清除文档缓存。

#### 5.5.6 失败回滚

如果移动过程中抛出 `OSError`、`DatabaseError` 或 `CannotMoveFilesException`：

1. **尝试回移文件**：
   - 如果原始文件已移到新位置且新位置文件存在 → `shutil.move(instance.source_path, old_source_path)`
   - 如果归档文件已移到新位置且新位置文件存在 → `shutil.move(instance.archive_path, old_archive_path)`
   - 回移本身失败则忽略（文件不会丢失，只是留在新位置，由 sanity checker 处理）
2. **恢复内存中的文件名**：`instance.filename = old_filename`，`instance.archive_filename = old_archive_filename`

#### 5.5.7 清理空目录

无论成功还是失败，最后检查旧路径的父目录是否为空，递归向上删除空目录（直到到达 `ORIGINALS_DIR` / `ARCHIVE_DIR` 根目录为止）。

> **关键洞察**：文件名生成了两次！
> - **第一次**：`ConsumerPlugin.run()` 中 `FileLock` 内（L671），调用 `generate_unique_filename()` — 把文件放到初始位置
> - **第二次**：`post_save` 处理器中，调用 `generate_filename()` — 重算最终位置并移动
> - 两次生成的文件名通常相同，但如果信号处理器修改了 document 属性（如 `set_storage_path` 改变了存储路径模板），第二次生成结果可能不同，文件会被移动到新位置

> **归档文件特殊逻辑**：`generate_unique_filename(doc, archive_filename=True)` 优先尝试 `<原始文件stem>.pdf` 作为归档文件名（`src/documents/file_handling.py` L68-82），保持原始文件与归档文件名的一致性，只有当此名称冲突时才走 `generate_filename()` + 计数器后缀

### 5.6 版本更新信号

**代码位置**：`src/documents/consumer.py` L731-735

如果是已有文档的新版本，发送 `document_updated` 信号：
- `run_workflows_updated`：执行"文档更新"触发的工作流
- `send_websocket_document_updated`：发送 WebSocket 通知

### 5.7 删除临时文件

**代码位置**：`src/documents/consumer.py` L737-758

事务内的最后一步：
1. 删除原始输入文件：`self.input_doc.original_file.unlink()`
2. 删除工作副本：`self.working_copy.unlink()`
3. 删除未修改的原始文件（如有）：`self.unmodified_original.unlink()`
4. 删除 macOS 资源分支文件：`._filename`

至此事务结束。如果以上所有步骤成功，数据库事务提交。

### 5.8 Post-consume 脚本（事务外）

**代码位置**：`src/documents/consumer.py` L769

```python
self.run_post_consume_script(document)
```

**关键**：此步骤在 `transaction.atomic()` 块**之外**执行。

- 如果脚本执行失败，`_fail()` 会抛出异常，但**文档已经成功消费**（事务已提交）
- 脚本通过环境变量接收完整文档信息：

```
DOCUMENT_ID, DOCUMENT_TYPE, DOCUMENT_CREATED, DOCUMENT_MODIFIED,
DOCUMENT_ADDED, DOCUMENT_FILE_NAME, DOCUMENT_SOURCE_PATH,
DOCUMENT_ARCHIVE_PATH, DOCUMENT_THUMBNAIL_PATH, DOCUMENT_DOWNLOAD_URL,
DOCUMENT_THUMBNAIL_URL, DOCUMENT_OWNER, DOCUMENT_CORRESPONDENT,
DOCUMENT_TAGS, DOCUMENT_ORIGINAL_FILENAME, TASK_ID
```

### 5.9 流程时序总结

```
transaction.atomic() 开始
├─ _store() → 创建 Document 记录，filename=None, archive_filename=None
│   └─ document.save() → post_save 触发但 filename 为空直接返回
│
├─ document_consumption_finished 信号
│   ├─ add_inbox_tags
│   ├─ set_correspondent  ← 修改 document 属性，影响后续文件名
│   ├─ set_document_type
│   ├─ set_tags
│   ├─ set_storage_path   ← 可能改变存储路径模板
│   ├─ add_to_index
│   ├─ run_workflows_added
│   └─ add_or_update_document_in_llm_index
│
├─ FileLock(MEDIA_LOCK)   ← 第一次获取锁
│   ├─ generate_unique_filename(document)          → 原始文件初始文件名
│   ├─ document.filename = generated_filename
│   ├─ _write(working_copy, document.source_path)  → ORIGINALS_DIR/...
│   ├─ _write(thumbnail, document.thumbnail_path)  → THUMBNAIL_DIR/<pk>.webp
│   ├─ generate_unique_filename(document, archive_filename=True)  → 归档文件初始文件名
│   ├─ document.archive_filename = generated_archive_filename
│   ├─ _write(archive_path, document.archive_path) → ARCHIVE_DIR/...
│   └─ document.archive_checksum = compute_checksum(...)
│
├─ document.save()   ← FileLock 已释放后才 save（避免死锁）
│   └─ post_save → update_filename_and_move_files()
│       ├─ FileLock(MEDIA_LOCK)   ← 第二次获取锁
│       ├─ generate_filename(instance)             → 重算原始文件名
│       ├─ 冲突判断：三路分支（相同/已存在/正常）
│       ├─ generate_filename(instance, archive=True) → 重算归档文件名
│       ├─ 冲突判断：三路分支（相同/已存在/正常）
│       ├─ 如需移动：validate_move() → shutil.move()
│       ├─ Document.objects.filter(pk=...).update()  ← 避免 save() 递归
│       └─ delete_empty_directories()
│
└─ 删除临时文件（original_file, working_copy, unmodified_original, ._shadow）
transaction.atomic() 结束

run_post_consume_script()  ← 事务外执行
```

---

## 六、文件路径与命名机制详解

### 6.1 文件路径属性

`Document` 模型上的路径属性（`src/documents/models.py` L431-484）都是动态计算的，由数据库中存储的相对文件名拼接根目录得到：

| 属性 | 计算方式 | 数据库字段 | 根目录 |
|---|---|---|---|
| `source_path` | `ORIGINALS_DIR / filename` | `filename` | `ORIGINALS_DIR` |
| `archive_path` | `ARCHIVE_DIR / archive_filename` | `archive_filename` | `ARCHIVE_DIR` |
| `thumbnail_path` | `THUMBNAIL_DIR / {pk:07}.webp` | 无（固定规则） | `THUMBNAIL_DIR` |

当 `filename` 或 `archive_filename` 为空时，`source_path` 回退为 `ORIGINALS_DIR / {pk:07}{file_type}`。

### 6.2 文件名生成的两层函数

**`generate_filename()`**（`src/documents/file_handling.py` L125-185）：核心渲染函数

```
输入: document, counter=0, archive_filename=False, use_format=True
输出: Path（相对路径）
```

1. **确定模板来源**（按优先级）：
   - `document.storage_path.path`（文档级存储路径模板）
   - `settings.FILENAME_FORMAT`（全局模板，先转换为新语法）
   - 无模板 → 使用 `{pk:07}` 数字命名
2. **渲染模板**：通过 Jinja2 模板引擎，注入文档上下文（`src/documents/templating/filepath.py` L345-412）：
   - 基础元数据：title、correspondent、document_type、asn、owner_username、original_name、doc_pk
   - 日期：created_year/month/day、added_year/month/day 等
   - 标签：tag_list、tag_name_list
   - 自定义字段：custom_fields（按字段名索引，含 type 和 value）
   - 空值统一替换为 `-none-` 占位符
3. **安全检查**：`_is_safe_relative_path()` — 拒绝绝对路径和 `..` 遍历
4. **后处理**：
   - `FILENAME_FORMAT_REMOVE_NONE=True` 时删除 `-none-` 占位符及多余分隔符
5. **拼装最终路径**：`{rendered_path.stem}{version_suffix}{counter_str}{filetype_str}`
   - `version_suffix`：版本文档添加 `_v{index}`
   - `counter_str`：冲突时添加 `_01`、`_02`
   - `filetype_str`：原始文件用 `doc.file_type`（如 `.pdf`、`.png`），归档文件固定 `.pdf`

**`generate_unique_filename()`**（`src/documents/file_handling.py` L44-99）：冲突解决包装器

```
输入: doc, archive_filename=False
输出: Path（相对路径，保证不与现有文件冲突）
```

1. **归档文件快捷路径**：如果 `archive_filename=True` 且 `doc.filename` 存在，先尝试 `{原始文件stem}.pdf`（保持同名），目标不存在则直接返回
2. **循环检测冲突**：从 `counter=0` 开始调用 `generate_filename(doc, counter=counter)`，检查 `(root / new_filename).exists()`
   - 如果文件名与当前文件名相同 → 返回（未改变）
   - 如果目标文件已存在 → counter++ 重试
   - 如果目标文件不存在 → 返回

### 6.3 原始文件 vs 归档文件的命名差异

| 方面 | 原始文件 | 归档文件 |
|---|---|---|
| 根目录 | `ORIGINALS_DIR` | `ARCHIVE_DIR` |
| 数据库字段 | `filename` | `archive_filename` |
| 扩展名 | 保留原始扩展名（`.pdf`、`.png` 等） | 固定 `.pdf` |
| 快捷命名 | 无 | 优先 `<原始文件stem>.pdf`（`file_handling.py` L68-82） |
| 冲突检测 | `ORIGINALS_DIR / name` 是否存在 | `ARCHIVE_DIR / name` 是否存在 |
| 版本后缀 | `_v{index}`（仅版本文档） | `_v{index}`（仅版本文档） |
| 上下文文档 | 版本文档使用 `root_document` 渲染模板 | 同原始文件 |

### 6.4 两阶段文件名生成对比

| 阶段 | 调用函数 | 目的 | 冲突处理 |
|---|---|---|---|
| 初次放置（5.4） | `generate_unique_filename()` | 把文件写到磁盘 | 循环加 counter 直到文件名不冲突 |
| 重算移动（5.5） | `generate_filename()` | 计算目标位置 | 三路分支判断（相同/已存在/新路径） |

> 两阶段设计的核心原因：初次放置时 `document.filename` 为 None，需要先生成一个文件名才能写入文件；写入后 save 触发 `update_filename_and_move_files()`，此函数**依赖 `filename` 不为空**才执行（L465-474 的守卫条件）。如果文件名在信号处理器后发生了变化（如存储路径改变），文件会被移动到正确位置。

### 6.5 文件移动的完整生命周期

```
消费时：
  临时文件 ──write──→ ORIGINALS_DIR/<第一次生成的文件名>  (5.4)
                       │
                       └──save──→ post_save → 5.5 重算
                                    │
                                    ├─ 文件名相同 → 不移动
                                    ├─ 文件名不同 → validate_move() → shutil.move()
                                    │                ORIGINALS_DIR/<旧> → ORIGINALS_DIR/<新>
                                    └─ 冲突       → generate_unique_filename() 加后缀

后续编辑时（用户修改通信方/标签/存储路径）：
  Document.save() → post_save → update_filename_and_move_files()
                                    │
                                    ├─ 原始文件：ORIGINALS_DIR/<旧> → ORIGINALS_DIR/<新>
                                    └─ 归档文件：ARCHIVE_DIR/<旧>   → ARCHIVE_DIR/<新>
```

---

## 七、关键文件索引

| 文件 | 职责 |
|---|---|
| [tasks.py](src/documents/tasks.py#L124) | `consume_file` Celery 任务，插件链总调度 |
| [consumer.py](src/documents/consumer.py) | `ConsumerPlugin`（核心消费）、`ConsumerPreflightPlugin`、`AsnCheckPlugin`、`WorkflowTriggerPlugin` |
| [barcodes.py](src/documents/barcodes.py) | `BarcodePlugin` 条码识别与文档拆分 |
| [double_sided.py](src/documents/double_sided.py) | `CollatePlugin` 双面扫描整理 |
| [data_models.py](src/documents/data_models.py) | `ConsumableDocument`、`DocumentMetadataOverrides` |
| [plugins/base.py](src/documents/plugins/base.py) | 插件接口定义、`StopConsumeTaskError` |
| [parsers.py](src/documents/parsers.py) | 旧版 `DocumentParser` 基类、缩略图工具函数 |
| [registry.py](src/paperless/parsers/registry.py) | `ParserRegistry` 解析器注册与选择 |
| [tesseract.py](src/paperless/parsers/tesseract.py) | `RasterisedDocumentParser` OCR 核心实现 |
| [text.py](src/paperless/parsers/text.py) | `TextDocumentParser` 纯文本解析器 |
| [document_consumer.py](src/documents/management/commands/document_consumer.py) | 消费目录监听管理命令 |
| [mail.py](src/paperless_mail/mail.py) | 邮件收取与消费任务投递 |
| [handlers.py](src/documents/signals/handlers.py) | 信号处理器：自动分类、工作流、索引、文件名重算与移动 |
| [apps.py](src/documents/apps.py) | 信号连接注册 |
| [file_handling.py](src/documents/file_handling.py) | `generate_filename()`、`generate_unique_filename()` 文件名生成 |
| [filepath.py](src/documents/templating/filepath.py) | Jinja2 模板渲染文件路径、安全校验 |
| [utils.py](src/documents/utils.py#L147-L169) | `compute_checksum()` SHA-256 哈希计算 |
| [models.py](src/documents/models.py#L431-L484) | `Document.source_path`、`archive_path`、`thumbnail_path` 属性 |
