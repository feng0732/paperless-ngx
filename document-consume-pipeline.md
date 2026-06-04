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

文档有四种来源，对应 `DocumentSource` 枚举（[data_models.py](file:///d:/fz/0601/solo-dogfeeding/code/26-paperless-ngx/src/documents/data_models.py#L150-L158)）：

| 来源 | DocumentSource | 入口代码 | 触发方式 |
|---|---|---|---|
| 消费目录 | `ConsumeFolder (1)` | [document_consumer.py](file:///d:/fz/0601/solo-dogfeeding/code/26-paperless-ngx/src/documents/management/commands/document_consumer.py#L308-L353) | `watchfiles` 监听目录变化 |
| API 上传 | `ApiUpload (2)` | [views.py#L3127](file:///d:/fz/0601/solo-dogfeeding/code/26-paperless-ngx/src/documents/views.py#L3127-L3158) | REST API `POST /api/documents/post_document/` |
| 邮件收取 | `MailFetch (3)` | [mail.py#L878](file:///d:/fz/0601/solo-dogfeeding/code/26-paperless-ngx/src/paperless_mail/mail.py#L878-L905) | Celery 定时任务 `process_mail_accounts` |
| Web UI 上传 | `WebUI (4)` | [views.py#L3127](file:///d:/fz/0601/solo-dogfeeding/code/26-paperless-ngx/src/documents/views.py#L3127-L3158) | 前端拖拽/选择文件上传 |

### 2.1 消费目录监听

`document_consumer` 管理命令（[document_consumer.py](file:///d:/fz/0601/solo-dogfeeding/code/26-paperless-ngx/src/documents/management/commands/document_consumer.py)）是消费目录的守护进程，核心流程：

1. **启动时扫描**：`_process_existing_files()` 遍历目录中已有文件
2. **持续监听**：通过 `watchfiles.watch()` 监听文件系统事件（原生 inotify/FSEvents 或轮询回退）
3. **稳定性追踪**：`FileStabilityTracker` 确保文件写入完成后才消费（防止读到不完整的文件）
4. **扩展名过滤**：`ConsumerFilter` 只接受解析器支持的文件类型，忽略系统文件和配置的忽略模式
5. **子目录作为标签**：如果 `CONSUMER_SUBDIRS_AS_TAGS` 开启，文件所在子目录名会自动创建/匹配为标签

文件稳定后调用 `_consume_file()`，构造 `ConsumableDocument(source=DocumentSource.ConsumeFolder, ...)` 并通过 `consume_file.apply_async()` 投入 Celery 任务队列。

### 2.2 API / Web UI 上传

两个入口共用 `post_document()` 视图（[views.py#L3127](file:///d:/fz/0601/solo-dogfeeding/code/26-paperless-ngx/src/documents/views.py#L3127)）：

1. 上传文件写入 `SCRATCH_DIR` 下的临时文件
2. 从请求参数构造 `DocumentMetadataOverrides`（可含 title、correspondent、tags 等预设元数据）
3. 构造 `ConsumableDocument`（`source=WebUI` 或 `ApiUpload`）
4. 调用 `consume_file.apply_async()` 入队

### 2.3 邮件收取

`MailAccountFetcher.handle_mail_rule()` 处理邮件规则（[mail.py](file:///d:/fz/0601/solo-dogfeeding/code/26-paperless-ngx/src/paperless_mail/mail.py)）：

1. 连接 IMAP 邮箱，按规则过滤邮件
2. 附件模式：逐个附件保存为临时文件，构造 `ConsumableDocument(source=MailFetch)`
3. EML 模式：整封邮件保存为 `.eml` 文件
4. 通过 Celery `chord` 编排：所有附件消费完成后执行邮件动作（标记已读/删除/移动等）

### 2.4 核心数据模型

- **`ConsumableDocument`**（[data_models.py#L162](file:///d:/fz/0601/solo-dogfeeding/code/26-paperless-ngx/src/documents/data_models.py#L162-L188)）：封装待消费文件，包含来源、文件路径、关联邮件规则等。`__post_init__` 中自动解析绝对路径并检测 MIME 类型。
- **`DocumentMetadataOverrides`**（[data_models.py#L13](file:///d:/fz/0601/solo-dogfeeding/code/26-paperless-ngx/src/documents/data_models.py#L13-L99)）：携带元数据覆盖值（标题、通信方、标签、存储路径、自定义字段、权限等）。各插件可通过 `update()` 方法增量合并覆盖。

---

## 三、预处理：插件链逐步处理

`consume_file` 任务（[tasks.py#L124](file:///d:/fz/0601/solo-dogfeeding/code/26-paperless-ngx/src/documents/tasks.py#L124-L220)）是消费流水线的总调度。它按序实例化并执行插件链，每个插件遵循统一接口（[base.py](file:///d:/fz/0601/solo-dogfeeding/code/26-paperless-ngx/src/documents/plugins/base.py#L23-L103)）：

```
able_to_run → setup() → run() → cleanup()
```

插件可以：
- 修改 `metadata`（覆盖值会传递给后续插件和最终入库）
- 抛出 `StopConsumeTaskError` 终止整个消费任务（如条码拆分后原文件不再需要继续消费）
- 抛出 `ConsumeFileDuplicateError` 标记重复

### 3.1 ConsumerPreflightPlugin — 预检

代码：[consumer.py#L954](file:///d:/fz/0601/solo-dogfeeding/code/26-paperless-ngx/src/documents/consumer.py#L954-L1054)

1. **`pre_check_file_exists()`**：确认文件仍然存在
2. **`pre_check_duplicate()`**：计算文件 SHA256，在数据库中查找相同 checksum 的文档。如果 `CONSUMER_DELETE_DUPLICATES` 开启，直接删除重复文件并抛出 `ConsumeFileDuplicateError`
3. **`pre_check_directories()`**：确保 `SCRATCH_DIR`、`THUMBNAIL_DIR`、`ORIGINALS_DIR`、`ARCHIVE_DIR` 都已创建

### 3.2 AsnCheckPlugin — ASN 校验

代码：[consumer.py#L1056](file:///d:/fz/0601/solo-dogfeeding/code/26-paperless-ngx/src/documents/consumer.py#L1056-L1104)

检查 `metadata.asn`（档案序列号）：
- 范围校验：必须在 `[1, 4294967295]` 范围内
- 唯一性校验：不能与已有文档的 ASN 冲突

> 此插件在条码识别前后各执行一次：第一次检查用户指定的 ASN，第二次检查条码中读取到的 ASN。

### 3.3 CollatePlugin — 双面扫描整理

代码：[double_sided.py](file:///d:/fz/0601/solo-dogfeeding/code/26-paperless-ngx/src/documents/double_sided.py)

仅当 `CONSUMER_ENABLE_COLLATE_DOUBLE_SIDED` 开启且文件位于配置的双面子目录时激活：

1. **第一次收到文件**（奇数页/正面）：移入暂存区，抛出 `StopConsumeTaskError` 等待第二份
2. **第二次收到文件**（偶数页/反面）：反面页逆序排列，与正面逐页交错插入，合并为新 PDF
3. 合并后的 PDF 放回消费目录（不含双面子目录），会被消费目录监听器重新拾取

超时机制：暂存文件 30 分钟后自动失效。

### 3.4 BarcodePlugin — 条码识别与文档拆分

代码：[barcodes.py](file:///d:/fz/0601/solo-dogfeeding/code/26-paperless-ngx/src/documents/barcodes.py)

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

代码：[consumer.py#L67](file:///d:/fz/0601/solo-dogfeeding/code/26-paperless-ngx/src/documents/consumer.py#L67-L87)

调用 `run_workflows(trigger_type=CONSUMPTION)`，匹配消费阶段的 Workflow：

- 工作流的 **Assignment 动作** 会修改 `metadata`（设置标题、通信方、标签、存储路径、权限等）
- 工作流的 **Removal 动作** 会移除已有覆盖值
- 这些覆盖值最终在 `ConsumerPlugin._store()` 中应用到文档记录

---

## 四、OCR 与解析：ConsumerPlugin 核心

代码：[consumer.py#L246](file:///d:/fz/0601/solo-dogfeeding/code/26-paperless-ngx/src/documents/consumer.py#L246-L784)

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

通过 `ParserRegistry`（[registry.py](file:///d:/fz/0601/solo-dogfeeding/code/26-paperless-ngx/src/paperless/parsers/registry.py)）选择最合适的解析器：

1. 遍历所有注册解析器（第三方优先，然后内置）
2. 检查 `supported_mime_types()` 是否包含当前 MIME 类型
3. 调用 `score()` 获取优先级分数，分数最高者胜出

内置解析器（[registry.py#L200](file:///d:/fz/0601/solo-dogfeeding/code/26-paperless-ngx/src/paperless/parsers/registry.py#L200-L210)）：

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

代码：[tesseract.py#L494](file:///d:/fz/0601/solo-dogfeeding/code/26-paperless-ngx/src/paperless/parsers/tesseract.py#L494-L659)

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

`construct_ocrmypdf_parameters()`（[tesseract.py#L266](file:///d:/fz/0601/solo-dogfeeding/code/26-paperless-ngx/src/paperless/parsers/tesseract.py#L266-L383)）构建参数字典，包括：

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

`should_produce_archive()`（[consumer.py#L124](file:///d:/fz/0601/solo-dogfeeding/code/26-paperless-ngx/src/documents/consumer.py#L124-L189)）决定是否生成归档 PDF：

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

## 五、入库与归档

### 5.1 创建文档记录

`ConsumerPlugin._store()`（[consumer.py#L815](file:///d:/fz/0601/solo-dogfeeding/code/26-paperless-ngx/src/documents/consumer.py#L815-L875)）创建数据库记录：

1. **确定创建日期**：优先级 `metadata.created > parser.get_date() > 文件修改时间`
2. **确定标题**：优先级 `metadata.title > metadata.filename 的文件名部分 > 原始文件名`。标题支持工作流占位符解析
3. **计算 checksum**：SHA256（如果 qpdf 修复过 PDF，用修复前的原始文件计算）
4. **创建 Document 对象**：写入 title、content（OCR 文本）、mime_type、checksum、page_count 等
5. **应用覆盖**：`apply_overrides()` 设置通信方、文档类型、标签、存储路径、ASN、所有者、权限、自定义字段

如果是已有文档的新版本，走 `_create_version_from_root()` 创建版本记录。

### 5.2 文件写入磁盘

在 `transaction.atomic()` 内，使用 `FileLock(MEDIA_LOCK)` 保证并发安全：

1. **生成文件名**：`generate_unique_filename()` 根据文档元数据和 `FILENAME_FORMAT` 模板生成路径
2. **写入原始文件**：工作副本 → `ORIGINALS_DIR/<generated_filename>`
3. **写入缩略图**：临时缩略图 → `THUMBNAIL_DIR/<pk>-thumbnail.webp`
4. **写入归档文件**（如有）：归档 PDF → `ARCHIVE_DIR/<generated_archive_filename>`
5. **计算归档 checksum**

然后调用 `document.save()` 触发 `post_save` 信号。

### 5.3 清理临时文件

成功后删除：
- `input_doc.original_file`（原始输入文件）
- `self.working_copy`（工作副本）
- `self.unmodified_original`（qpdf 修复前的原始文件，如有）
- macOS 资源分支文件（`._filename`）

### 5.4 Post-consume 脚本

如果配置了 `POST_CONSUME_SCRIPT`，在消费完成后执行。环境变量包含完整的文档信息：

```
DOCUMENT_ID, DOCUMENT_TYPE, DOCUMENT_CREATED, DOCUMENT_MODIFIED,
DOCUMENT_ADDED, DOCUMENT_FILE_NAME, DOCUMENT_SOURCE_PATH,
DOCUMENT_ARCHIVE_PATH, DOCUMENT_THUMBNAIL_PATH, DOCUMENT_DOWNLOAD_URL,
DOCUMENT_THUMBNAIL_URL, DOCUMENT_OWNER, DOCUMENT_CORRESPONDENT,
DOCUMENT_TAGS, DOCUMENT_ORIGINAL_FILENAME, TASK_ID
```

### 5.5 document_consumption_finished 信号

在 `transaction.atomic()` 内发送，以下处理器同步执行（[apps.py#L24](file:///d:/fz/0601/solo-dogfeeding/code/26-paperless-ngx/src/documents/apps.py#L24-L31)）：

| 处理器 | 作用 |
|---|---|
| `add_inbox_tags` | 为文档添加标记为"收件箱"的标签 |
| `set_correspondent` | 用分类器 + 规则匹配自动设置通信方 |
| `set_document_type` | 用分类器 + 规则匹配自动设置文档类型 |
| `set_tags` | 用分类器 + 规则匹配自动设置标签 |
| `set_storage_path` | 用分类器 + 规则匹配自动设置存储路径 |
| `add_to_index` | 写入 Whoosh/Tantivy 搜索索引 |
| `run_workflows_added` | 执行"文档添加"触发的工作流（可含 Assignment/Removal/Email/Webhook/MoveToTrash 动作） |
| `add_or_update_document_in_llm_index` | 写入 LLM 向量索引 |

### 5.6 文件重命名与移动

`document.save()` 触发 `post_save` 信号 → `update_filename_and_move_files()`（[handlers.py#L434](file:///d:/fz/0601/solo-dogfeeding/code/26-paperless-ngx/src/documents/signals/handlers.py#L434-L668)）：

1. 根据 `FILENAME_FORMAT` 和文档最新元数据生成目标文件名
2. 如果文件名有变化，在 `FileLock` 保护下移动文件
3. 同时处理原始文件和归档文件的移动
4. 清理移动后遗留的空目录

---

## 六、关键文件索引

| 文件 | 职责 |
|---|---|
| [tasks.py](file:///d:/fz/0601/solo-dogfeeding/code/26-paperless-ngx/src/documents/tasks.py#L124) | `consume_file` Celery 任务，插件链总调度 |
| [consumer.py](file:///d:/fz/0601/solo-dogfeeding/code/26-paperless-ngx/src/documents/consumer.py) | `ConsumerPlugin`（核心消费）、`ConsumerPreflightPlugin`、`AsnCheckPlugin`、`WorkflowTriggerPlugin` |
| [barcodes.py](file:///d:/fz/0601/solo-dogfeeding/code/26-paperless-ngx/src/documents/barcodes.py) | `BarcodePlugin` 条码识别与文档拆分 |
| [double_sided.py](file:///d:/fz/0601/solo-dogfeeding/code/26-paperless-ngx/src/documents/double_sided.py) | `CollatePlugin` 双面扫描整理 |
| [data_models.py](file:///d:/fz/0601/solo-dogfeeding/code/26-paperless-ngx/src/documents/data_models.py) | `ConsumableDocument`、`DocumentMetadataOverrides` |
| [plugins/base.py](file:///d:/fz/0601/solo-dogfeeding/code/26-paperless-ngx/src/documents/plugins/base.py) | 插件接口定义、`StopConsumeTaskError` |
| [parsers.py](file:///d:/fz/0601/solo-dogfeeding/code/26-paperless-ngx/src/documents/parsers.py) | 旧版 `DocumentParser` 基类、缩略图工具函数 |
| [registry.py](file:///d:/fz/0601/solo-dogfeeding/code/26-paperless-ngx/src/paperless/parsers/registry.py) | `ParserRegistry` 解析器注册与选择 |
| [tesseract.py](file:///d:/fz/0601/solo-dogfeeding/code/26-paperless-ngx/src/paperless/parsers/tesseract.py) | `RasterisedDocumentParser` OCR 核心实现 |
| [text.py](file:///d:/fz/0601/solo-dogfeeding/code/26-paperless-ngx/src/paperless/parsers/text.py) | `TextDocumentParser` 纯文本解析器 |
| [document_consumer.py](file:///d:/fz/0601/solo-dogfeeding/code/26-paperless-ngx/src/documents/management/commands/document_consumer.py) | 消费目录监听管理命令 |
| [mail.py](file:///d:/fz/0601/solo-dogfeeding/code/26-paperless-ngx/src/paperless_mail/mail.py) | 邮件收取与消费任务投递 |
| [handlers.py](file:///d:/fz/0601/solo-dogfeeding/code/26-paperless-ngx/src/documents/signals/handlers.py) | 消费完成后的信号处理器（自动分类、工作流、索引） |
| [apps.py](file:///d:/fz/0601/solo-dogfeeding/code/26-paperless-ngx/src/documents/apps.py) | 信号连接注册 |
