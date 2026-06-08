# Paperless-ngx Parser Plugin 体系代码分析

> 所有文件引用使用仓库相对路径（以 `src/` 开头），可在 IDE 中直接跳转。

---

## 一、整体架构

Paperless-ngx 的文档处理分为两个层次的插件体系：

1. **ConsumeTaskPlugin 消费任务插件链**：编排整个文档消费流程（预检→ASN→双面整理→条码→ASN→工作流→核心消费）
2. **ParserProtocol 文档解析器体系**：负责具体文件格式的文本提取、OCR、PDF 生成等

```
src/documents/tasks.py
    └── _try_consume_file()
          └── [插件链顺序执行]
                ├── ConsumerPreflightPlugin   (预检)
                ├── AsnCheckPlugin           (ASN 校验)
                ├── CollatePlugin            (双面文档整理)
                ├── BarcodePlugin            (条码识别拆分)
                ├── AsnCheckPlugin           (再次 ASN 校验)
                ├── WorkflowTriggerPlugin    (工作流元数据覆盖)
                └── ConsumerPlugin           (核心消费 → 调用 Parser)
                                          └── src/paperless/parsers/registry.py
                                                └── ParserRegistry.get_parser_for_file()
                                                      ├── score 评分选择
                                                      └── ParserProtocol 实现
```

---

## 二、插件选择逻辑

### 2.1 消费任务插件链编排

插件链的定义和执行位于 `src/documents/tasks.py` L140-L218。

**执行规则**（每个插件）：
1. 实例化时传入 `input_doc / metadata / status_mgr / tmp_dir / task_id`
2. `able_to_run == False` → 跳过
3. 依次调用 `setup() → run() → finally: cleanup()`
4. `run()` 返回值为信息日志；`metadata` 可被修改并传递给下一个插件
5. 抛出 `StopConsumeTaskError` → 终止整个任务并返回 `ConsumeFileStoppedResult`
6. 抛出 `ConsumeFileDuplicateError` → 返回 `ConsumeFileDuplicateResult`

插件基类与 Mixin 定义在 `src/documents/plugins/base.py` L23-L131：
- `AlwaysRunPluginMixin`：`able_to_run` 恒为 True
- `NoSetupPluginMixin`：`setup()` 空实现
- `NoCleanupPluginMixin`：`cleanup()` 空实现

### 2.2 Parser 评分选择机制（含 Remote 与外部解析器优先级）

核心注册中心：`src/paperless/parsers/registry.py`

**协议定义**：`src/paperless/parsers/__init__.py` L116-L434 的 `ParserProtocol` 定义了所有 Parser 必须满足的结构契约（运行时可 `isinstance` 检查）。

**发现机制**：
- 内置 Parser：在 `register_defaults()` 中显式注册（注册顺序：Text → Remote → Tika → Mail → Rasterised(Tesseract)）
- 第三方（外部）Parser：通过 Python entrypoint 组 `paperless_ngx.parsers` 自动发现
  ```toml
  [project.entry-points."paperless_ngx.parsers"]
  my_parser = "my_package.parsers:MyParser"
  ```
- 校验属性：`name / version / author / url / supported_mime_types / score` 缺一不可

**选择算法**（`get_parser_for_file`，`registry.py` L332-L393）：

```
遍历顺序 = [*外部解析器(entrypoint), *内置解析器]
    ↓
对每个 parser_class:
    1. mime_type ∉ supported_mime_types() → 跳过
    2. score = parser_class.score(mime_type, filename, path)
    3. score is None → 跳过
    4. score > best_score → 替换为当前最佳（严格大于才替换）

最终：
    - 同分时，先出现的胜出（first-seen）→ 外部解析器天然优先于内置
    - 若胜出者是外部解析器，记录 INFO 日志
```

**内置解析器的分数与激活条件**：

| Parser 类 | score | 支持 MIME | 激活条件 |
|-----------|-------|-----------|----------|
| `TextDocumentParser` | 10 | text/plain, text/csv, application/csv | 始终激活 |
| `RemoteDocumentParser` | **20** | application/pdf, image/* (png/jpeg/tiff/bmp/gif/webp) | `REMOTE_OCR_ENGINE` 等配置有效 |
| `TikaDocumentParser` | 10 | Office 文档 (doc/docx/xls/xlsx/ppt/pptx/odt/ods/odp/rtf…) | `TIKA_ENABLED=True` |
| `MailDocumentParser` | 10 | message/rfc822 (.eml) | 始终激活 |
| `RasterisedDocumentParser` (Tesseract) | 10 | application/pdf, image/* (含 heic) | 始终激活 |

**Remote 与外部解析器的优先级关系**：

1. **RemoteDocumentParser 是内置解析器**，不是外部 entrypoint 解析器。它位于 `_builtins` 列表中。
2. **外部解析器（entrypoint）先于所有内置解析器被遍历**。如果某个外部解析器对同一种 MIME 类型返回 ≥10 的分数，它会战胜 Tesseract、Tika、Mail、Text。
3. 但 **RemoteDocumentParser 的 score=20**，如果外部解析器返回 ≤19 分，Remote 仍会胜出。如果外部解析器返回 ≥21 分，则外部解析器覆盖 Remote。
4. Remote 激活后覆盖 PDF/图片的 **Tesseract（score=10）**，因为 20 > 10。

> 小结：`外部(score≥21) > Remote(20) > Tesseract/Tika/Mail/Text(10) > 外部(score≤9且不被跳过)`

---

## 三、外部工具调用链细节

### 3.1 Tesseract OCR — `src/paperless/parsers/tesseract.py`

**调用入口**：`parse()` → `ocrmypdf.ocr(**args)`

参数构造由 `construct_ocrmypdf_parameters()`（L266-L383）完成，涉及：
- **OCR 模式**：`force_ocr / redo_ocr / skip_text / auto / off`
- **图像预处理**：`clean / clean_final / deskew / rotate_pages`
- **语言**：`OCR_LANGUAGE`
- **输出类型**：`pdfa / pdf / pdfa-1 / pdfa-2 / pdfa-3`
- **图像专用**：`image_dpi`（读取文件 DPI → A4 宽度估算 → 配置兜底，三级回退），alpha 通道移除

**Tesseract 的后备（Fallback）机制**（`parse()` L494-L659）：

```
OCR_MODE=off（完全不调用 OCR）:
  ├── 不生成归档 + PDF → 直接返回 pdftotext 文本
  ├── 图片输入 → img2pdf 转换 + pikepdf 打 PDF/A 标签
  └── PDF 输入 → Ghostscript (_convert_pdf_to_pdfa) 直接转 PDF/A

OCR_MODE=auto + 原文有文本 + 不生成归档:
  └── 跳过 ocrmypdf，直接返回 pdftotext 结果

完整 OCR 流程（其他情况）:
  ├── 第一次尝试: ocrmypdf.ocr(normal_args)
  │     ├── 成功 → 提取 sidecar.txt / 归档 PDF 文本
  │     └── 失败异常:
  │           ├── DigitalSignatureError / EncryptedPdfError → 若原 PDF 有文本则用原文本
  │           ├── SubprocessOutputError(Ghostscript PDF/A 失败) → 提示 PAPERLESS_OCR_USER_ARGS 并 raise ParseError
  │           └── NoTextFoundException / InputFileError / PriorOcrFoundError → 进入 Fallback
  │
  └── Fallback 重试（safe_fallback=True）:
        ├── construct_ocrmypdf_parameters(..., safe_fallback=True)
        │     └── 仅强制 force_ocr=True，其他全部配置原样保留（见 safe_fallback 详解）
        └── ocrmypdf.ocr(fallback_args)
              ├── 成功 → 提取文本，归档文件
              └── 失败 → raise ParseError

文本提取优先级（extract_text() L236-L264）:
  1. sidecar.txt（ocrmypdf 生成），除非 REDO 模式
  2. sidecar 含 "[OCR skipped on page" → 判定不完整，丢弃
  3. 从归档 PDF 中用 pdftotext 提取
  4. 仍为空 → 若原 PDF 有文本则用原文本，否则空串 + 告警
```

**safe_fallback 实际保留/修改的配置详解**（`construct_ocrmypdf_parameters()` L266-L383）：

`safe_fallback` 的代码影响只有 **一行**（L293）：
```python
if safe_fallback or self.settings.mode == ModeChoices.FORCE:
    ocrmypdf_args["force_ocr"] = True
```

也就是说，`safe_fallback=True` **等价于强制 OCR 模式为 FORCE**，其他所有配置**不做任何简化或屏蔽**，**完整保留**：

| 配置项 | safe_fallback 时是否保留 | 说明 |
|--------|-------------------------|------|
| `force_ocr` | ✅ **被强制设为 True** | 覆盖 redo_ocr/skip_text/auto 的模式判断 |
| `use_threads` / `jobs` | ✅ 保留 | 线程配置不变 |
| `language` | ✅ 保留 | OCR 语言不变 |
| `output_type` | ✅ 保留 | pdfa / pdf / pdfa-1/2/3 不变 |
| `color_conversion_strategy` | ✅ 保留 | 仅在含 "pdfa" 的 output_type 时生效 |
| `clean` / `clean_final` | ✅ 保留 | 图像去斑点预处理不变 |
| `deskew` | ✅ 保留 | 校正倾斜不变（注意 `mode != REDO` 的条件仍有效，但 safe_fallback 下 mode 实际被强制为 FORCE，所以仍保留） |
| `rotate_pages` / `rotate_pages_threshold` | ✅ 保留 | 自动旋转不变 |
| `pages` / `sidecar` | ✅ 保留 | 只处理前 N 页 / 生成 sidecar 文本不变 |
| `image_dpi` / alpha 移除 | ✅ 保留 | 图片 DPI 三级回退、alpha 通道兼容性处理不变 |
| `user_args`（OCR_USER_ARGS） | ✅ 保留 | 用户自定义参数完整合并 |
| `max_image_mpixels` | ✅ 保留 | 像素限制不变 |

因此 safe_fallback 的"安全"含义是：**切换到最保守的 OCR 模式（force_ocr 强制对所有页面跑 OCR，不管有没有已有文本层），而不是减少参数**。第一次失败通常是由于 REDO/AUTO 模式下 ocrmypdf 的文本层检测出问题，强制 force_ocr 绕过这些检测。

### 3.2 Tika + Gotenberg — `src/paperless/parsers/tika.py`

**客户端生命周期**：在 `__enter__` 中通过 `ExitStack` 统一创建 `TikaClient` 和 `GotenbergClient`，`__exit__` 时自动关闭。

**文本提取**（`parse()` L212-L278）：
```python
# 优先 multipart/form-data 文件上传
parsed = tika_client.tika.as_text.from_file(document_path, mime_type)
# 若 Tika 返回 500（TIKA-4110 bug，部分文件 multipart 出错）→ fallback：直接传 bytes buffer
parsed = tika_client.tika.as_text.from_buffer(document_path.read_bytes(), mime_type)
```

**PDF 转换**（`_convert_to_pdf()` L400-L452）：
- 走 Gotenberg 的 LibreOffice 路由
- 根据 `OCR_OUTPUT_TYPE` 设置 `PdfAFormat`（A2b / A3b，PDF/A-1a 降级告警为 A2b）
- 因为 `requires_pdf_rendition=True`，**即使 `produce_archive=False` 也必须生成 PDF**（浏览器无法显示 Office 原生格式）

### 3.3 Remote OCR (Azure AI Vision) — `src/paperless/parsers/remote.py`

配置校验由 `RemoteEngineConfig`（L46-L65）完成：
- `engine ∈ ("azureai",)` + `api_key` + `endpoint`（azureai 必填 endpoint）

**调用流程**（`_azure_ai_vision_parse()` L361-L433）：
```python
client = DocumentIntelligenceClient(endpoint, AzureKeyCredential(api_key))

# 1. 提交分析任务（长轮询，prebuilt-read 模型）
poller = client.begin_analyze_document(
    model_id="prebuilt-read",
    body=AnalyzeDocumentRequest(bytes_source=f.read()),
    output_content_format=DocumentContentFormat.TEXT,
    output=[AnalyzeOutputOption.PDF],   # 请求生成带文本层的 PDF
)
poller.wait()

# 2. 获取文本内容
result = poller.result()
text = result.content

# 3. 流式下载 searchable PDF 归档到 self._archive_path
for chunk in client.get_analyze_result_pdf(model_id="prebuilt-read", result_id=operation_id):
    f.write(chunk)
```

### 3.4 Mail Parser — `src/paperless/parsers/mail.py`（见第六节 ParserContext 详解）

### 3.5 qpdf 的使用 — `src/documents/consumer.py` L431-L461

**触发条件**：
```python
if (
    Path(self.filename).suffix.lower() == ".pdf"
    and mime_type in settings.CONSUMER_PDF_RECOVERABLE_MIME_TYPES
):
    # 默认 CONSUMER_PDF_RECOVERABLE_MIME_TYPES = ("application/octet-stream",)
```

即：**文件后缀是 .pdf，但 libmagic 识别为 application/octet-stream（或其他可恢复类型）**时，认为这是一个损坏/识别失败的 PDF。

**处理流程**：
```
1. qpdf --replace-input working_copy   (原地修复/线性化 PDF)
2. 重新 magic.from_file() 检测 MIME
3. 备份原始文件到 tmpdir/uo/<filename> 作为 unmodified_original（见下方备份条件）
4. 如果 qpdf 失败 → 仅记录错误日志，不中断流程（working_copy 保持原样）
```

**备份（unmodified_original）的精确条件**：

从代码结构看，`unmodified_original` 的赋值位于 qpdf `run_subprocess()` 调用之后、**与 qpdf 处于同一个 try 块中**，且 **qpdf 之后没有任何 MIME 有效性检查的 if 分支**：

```python
try:
    run_subprocess(["qpdf", "--replace-input", working_copy], ...)  # L441
    mime_type = magic.from_file(self.working_copy, mime=True)       # L449
    # 👇 注意：这里没有 if mime_type 有效的判断，直接执行备份
    self.unmodified_original = Path(tmpdir) / Path("uo") / Path(self.filename)  # L452
    self.unmodified_original.parent.mkdir(exist_ok=True)
    copy_file_with_basic_stats(
        self.input_doc.original_file,   # 注意：备份源是 input_doc.original_file
        self.unmodified_original,       #       不是 qpdf 之前的 working_copy 副本
    )
except Exception as e:
    self.log.error(f"Error attempting to clean PDF: {e}")
```

因此备份的唯一前置条件是：**qpdf 子进程退出码为 0 且 magic 重新检测没有抛异常**。具体：

| 场景 | `unmodified_original` 是否被设置 |
|------|--------------------------------|
| qpdf 成功，magic 重新检测得到 `application/pdf` | ✅ 是 |
| qpdf 成功，magic 重新检测仍是 `application/octet-stream` | ✅ **是**（代码无有效性判断） |
| qpdf 返回非零退出码 → `run_subprocess` 抛异常 | ❌ 否（跳转到 except） |
| `magic.from_file` 抛异常（极少） | ❌ 否（跳转到 except） |

**备份的来源文件**：`self.input_doc.original_file`（用户上传的原始文件，位于消费目录或 API 上传临时位置），**不是** qpdf 修改前 working_copy 的副本。因此 `unmodified_original` 始终等同于上传原件，不受 working_copy 之前任何修改影响（当前阶段 working_copy 只是刚从 original_file 拷贝过来的副本，两者内容一致）。

**qpdf 与 Pre-consume 脚本对解析器选择的影响差异**：

```
时序（consumer.py run()）:
  L427  mime_type = magic.from_file(working_copy)       ← 初始检测
  L431  if 条件满足 → qpdf --replace-input working_copy
  L449    mime_type = magic.from_file(working_copy)     ← 重新检测（可覆盖 mime_type）
  L463  parser_class = get_parser_for_file(mime_type, ...)  ← ← ← 解析器选择时刻
  L479  document_consumption_started.send(...)
  L485  self.run_pre_consume_script()                   ← pre-consume 执行
  L488  with parser_class() as document_parser:         ← 解析器已锁定
  L520    document_parser.parse(working_copy, ...)      ← 解析器实际运行
```

关键差异总结：

| 维度 | qpdf | pre-consume 脚本 |
|------|------|-----------------|
| 执行位置 | `get_parser_for_file()` **之前** | `get_parser_for_file()` **之后** |
| 是否修改 `mime_type` 变量 | ✅ 是（L449 重新赋值） | ❌ 否（mime_type 已是局部变量，脚本无权写入 Python 进程变量） |
| 是否改变解析器选择 | ✅ 直接影响（传入新 mime_type） | ❌ 不改变；parser_class 已被选定 |
| 是否修改 working_copy | ✅ 是（`--replace-input` 原地重写） | ✅ 可修改（通过 `DOCUMENT_WORKING_PATH` 环境变量指向的路径） |
| 修改的影响范围 | 解析器选择 + 解析行为 | **仅解析行为**（如输入文件内容、格式），解析器类型不可变 |

**注意**：qpdf 是 **Parser 选择之前**执行的预处理步骤，不属于任何 Parser 内部逻辑。qpdf 的输出 `working_copy` 会传给后续选定的 Parser。

---

## 四、Pre/Post 消费脚本

两者都定义在 `src/documents/consumer.py`：

### 4.1 Pre-consume 脚本（`run_pre_consume_script()` L297-L337）

**执行时机**：`document_consumption_started` 信号之后，`parser.parse()` 之前。

**环境变量注入**：
| 变量 | 内容 |
|------|------|
| `DOCUMENT_SOURCE_PATH` | 原始上传文件的绝对路径 |
| `DOCUMENT_WORKING_PATH` | working_copy（在临时目录中，可被脚本就地修改） |
| `TASK_ID` | Celery 任务 ID |

**行为**：
- 脚本可修改 `DOCUMENT_WORKING_PATH` 指向的文件（例如去水印、解密、格式转换）
- 脚本不存在 → `PRE_CONSUME_SCRIPT_NOT_FOUND` 错误，消费终止
- 脚本非零退出 → `PRE_CONSUME_SCRIPT_ERROR` 错误，消费终止

### 4.2 Post-consume 脚本（`run_post_consume_script()` L339-L406）

**执行时机**：**事务提交之后**（文件已写入 originals/thumbnails/archive，数据库已保存）。注意它在 `ConsumerPlugin.run()` 的 `with tempfile.TemporaryDirectory` 之外（L769），说明临时目录已清理。

**失败时文档持久化与任务状态的关系**：

```
consumer.py run() 关键时序:
  L417  with tempfile.TemporaryDirectory(...) as tmpdir:     ← 临时目录起点
  ...
  L587    with transaction.atomic():                        ← 数据库事务起点
  L589      if root_document_id: _create_version_from_root()
  L644      else: _store()
  ...
  L670      with FileLock(settings.MEDIA_LOCK):             ← 文件写入
  ...
  L760    except Exception: ... _fail()
  L769  self.run_post_consume_script(document)              ← 事务外，临时目录已清理
  L773  self._send_progress(SUCCESS, FINISHED, doc.id)      ← 成功状态
  L784  return ConsumeFileSuccessResult(document_id=...)

tasks.py 异常处理（L194-L216）:
  try:
      msg = plugin.run()                ← ConsumerPlugin.run() 被调用
  except Exception as e:                 ← run_post_consume_script 抛异常会被这里捕获
      status_mgr.send_progress(FAILED, ...)
      raise                             ← Celery 任务标记为失败
  finally:
      plugin.cleanup()
```

**状态矩阵**：

| 阶段 | post-consume 脚本结果 | 文档持久化 | 临时目录 | 任务最终状态 |
|------|----------------------|-----------|---------|-------------|
| L769 之前 | — | 事务已提交，**文件已落盘** | **已清理**（`with` 语句已退出） | — |
| L769 | ✅ 脚本成功 | 已完成 | 已清理 | `SUCCESS`，`ConsumeFileSuccessResult` |
| L769 | ❌ 脚本退出码非 0 | **已完成，不可回滚** | 已清理 | 调用 `self._fail()` → 抛 `ConsumerError` → tasks.py 捕获 → `FAILED` 状态 + Celery 任务异常 |
| L769 | ❌ 脚本路径不存在 | **已完成，不可回滚** | 已清理 | 同上，`POST_CONSUME_SCRIPT_NOT_FOUND` 错误 |

> 关键结论：**post-consume 脚本失败不会导致文档回滚**，因为它在 `transaction.atomic()` 和 `with tempfile.TemporaryDirectory()` 两个上下文管理器都退出之后才执行。失败的唯一影响是 Celery 任务标记为 FAILED 状态 + 记录错误日志，用户在 UI 上看到任务失败，但文档已经存在于库中（可正常搜索、下载）。

**环境变量注入**（完整）：
| 变量 | 内容 |
|------|------|
| `DOCUMENT_ID` | 数据库主键 |
| `DOCUMENT_TYPE` | DocumentType 名称 |
| `DOCUMENT_CREATED` / `MODIFIED` / `ADDED` | 时间戳 |
| `DOCUMENT_FILE_NAME` | 对外显示的文件名 |
| `DOCUMENT_SOURCE_PATH` / `ARCHIVE_PATH` / `THUMBNAIL_PATH` | 最终存储路径 |
| `DOCUMENT_DOWNLOAD_URL` / `THUMBNAIL_URL` | Django reverse 的 URL 路径 |
| `DOCUMENT_OWNER` / `CORRESPONDENT` / `TAGS` | 元数据 |
| `DOCUMENT_ORIGINAL_FILENAME` | 上传时的原始文件名 |
| `TASK_ID` | Celery 任务 ID |

**行为**：脚本失败同样会抛出 `POST_CONSUME_SCRIPT_ERROR`，但此时文档已持久化完成（事务外）。

---

## 五、版本消费分支（root_document_id）

在 `src/documents/tasks.py` L140-L155 中，插件链根据是否存在 `root_document_id` 选择不同的配置：

```python
plugins = (
    # 新版本路径：跳过 Collate、Barcode、WorkflowTrigger 等，直接消费
    [ConsumerPreflightPlugin, ConsumerPlugin]
    if input_doc.root_document_id is not None
    # 新文档路径：完整插件链
    else [
        ConsumerPreflightPlugin,
        AsnCheckPlugin,
        CollatePlugin,
        BarcodePlugin,
        AsnCheckPlugin,           # 条码扫描后可能新增 ASN，再次校验
        WorkflowTriggerPlugin,
        ConsumerPlugin,
    ]
)
```

**ConsumerPlugin 内部的分支逻辑**（`run()` L589-L642）：

### 分支 A：新文档（`root_document_id is None`）
- 调用 `_store(text, date, page_count, mime_type)`
- 新创建 Document，计算 `created` 日期（优先级：metadata.created → parser.get_date() → 文件 st_mtime）
- 应用 `apply_overrides()` 设置 correspondent / tags / 权限 / 自定义字段等

### 分支 B：新版本（`root_document_id is not None`）
- 调用 `_create_version_from_root(root_doc, text, page_count, mime_type)`
- **不使用 Parser 或 date_parsing 插件的 date**，直接继承 `root_doc.created`
- version_index = 现有最大 version_index + 1
- checksum 使用 `unmodified_original`（若存在，即 qpdf 处理前的原始文件）或 `working_copy`
- `content = text or ""`，`page_count`，`mime_type` 来自新文件
- 可选：保存 `version_label`（来自 metadata）
- 若启用 AUDIT_LOG_ENABLED：
  - 在 `set_actor(actor)` 上下文中 `save()` 记录操作人
  - 手动构造 `LogEntry` 记录 "Version Added" 变更

两个分支后续的文件写入（source / thumbnail / archive）是共享的逻辑，唯一区别在于版本分支的 Document 是 `original_document`（指向 root_document），新文档分支是新创建的 `document`。

---

## 六、ParserContext 与邮件布局如何影响解析效果

### 6.1 ParserContext 结构

定义在 `src/paperless/parsers/__init__.py` L78-L113：

```python
@dataclass(frozen=True, slots=True)
class ParserContext:
    mailrule_id: int | None = None     # 当前唯一使用的字段
    # 未来扩展: output_type, ocr_mode, ocr_language 等
```

`frozen=True` 确保 Consumer 传递后 Parser 不能修改；`slots=True` 轻量。

**传递链路**：
```
src/documents/consumer.py L488-L491:
    with parser_class() as document_parser:
        document_parser.configure(
            ParserContext(mailrule_id=self.input_doc.mailrule_id),
        )
```

只有 MailDocumentParser 在 `configure()`（`mail.py` L193-L194）中实际使用：
```python
def configure(self, context: ParserContext) -> None:
    self._mailrule_id = context.mailrule_id
```

### 6.2 邮件布局（PdfLayout）的具体影响

`MailRule.PdfLayout` 定义于 `src/paperless_mail/models.py` L118-L123：

| 值 | 含义 | Gotenberg merge 顺序 |
|----|------|----------------------|
| `DEFAULT` (0) | 系统默认（使用 `EMAIL_PARSE_DEFAULT_LAYOUT` 设置） | 取决于设置 |
| `TEXT_HTML` (1) | 纯文本邮件 + HTML 内容 | `[mail_pdf, html_pdf]` |
| `HTML_TEXT` (2) | HTML 内容 + 纯文本邮件 | `[html_pdf, mail_pdf]` |
| `HTML_ONLY` (3) | 仅 HTML 内容 | `[html_pdf]` |
| `TEXT_ONLY` (4) | 仅纯文本邮件 | `[mail_pdf]` |

**生效链路**（`mail.py` L274-L282 和 `generate_pdf()` L534-L606）：

```
parse() 被调用
  ├── mailrule_id 有值:
  │     └── MailRule.objects.get(pk=mailrule_id)
  │           └── MailRule.PdfLayout(rule.pdf_layout)
  │
  └── mailrule_id 为 None:
        └── MailRule.PdfLayout(settings.EMAIL_PARSE_DEFAULT_LAYOUT)

generate_pdf(mail_message, pdf_layout):
  ├── 无 HTML 内容:
  │     └── 只用 mail_pdf（忽略 layout）
  └── 有 HTML 内容:
        ├── mail_pdf = generate_pdf_from_mail()   邮件头+纯文本的 HTML 模板 → Gotenberg Chromium
        ├── html_pdf = generate_pdf_from_html()   原始 HTML → Gotenberg Chromium（含附件资源）
        └── Gotenberg merge 路由按 pdf_layout 顺序合并
```

**文本提取与 PDF 布局独立**：`build_formatted_text()` 在 `parse()` 开头就已经完成（Subject/From/To/... + HTML Tika 文本 + 纯文本），不受 PdfLayout 影响。PdfLayout **只影响归档 PDF 的页面顺序与组成**，不影响 `content` 搜索索引。

---

## 七、结果合并与持久化

### 7.1 ConsumerPlugin 中的结果采集

结果从 Parser 取出后在 `src/documents/consumer.py` L408-L784 的 `ConsumerPlugin.run()` 中被汇总，核心采集代码位于 L499-L553：

```python
with parser_class() as document_parser:
    document_parser.configure(ParserContext(mailrule_id=...))

    # 1. 决定是否生成归档 PDF
    produce_archive = should_produce_archive(document_parser, mime_type, working_copy)
    document_parser.parse(working_copy, mime_type, produce_archive=produce_archive)

    # 2. 按顺序提取结果
    text         = document_parser.get_text()
    date         = document_parser.get_date()
    thumbnail    = document_parser.get_thumbnail(working_copy, mime_type)
    archive_path = document_parser.get_archive_path()
    page_count   = document_parser.get_page_count(working_copy, mime_type)

    # 3. 日期兜底：Parser 没找到 → 用 date_parsing 插件
    if date is None:
        with get_date_parser() as date_parser:
            date = next(date_parser.parse(filename, text), None)
```

### 7.2 `should_produce_archive` 决策逻辑

`src/documents/consumer.py` L124-L189：
```
优先级从高到低：
1. parser.requires_pdf_rendition → True（必须生成，如 Office/EML）
2. not parser.can_produce_archive → False（如纯文本）
3. ARCHIVE_FILE_GENERATION=always → True / never → False
4. auto 模式：
     - image/* → True
     - application/pdf 且 tagged 结构 → False
     - application/pdf 且文本长度 ≤ 25 → True（扫描件）
     - application/pdf 且文本长度 > 25 → False（原生数字 PDF）
```

### 7.3 文本合并策略

**Tesseract 文本提取**（`extract_text()` L236-L264）：
```
1. 优先读 sidecar.txt（ocrmypdf 生成），除非 REDO 模式
2. sidecar 含 "[OCR skipped on page" → 判定不完整，丢弃
3. 从归档 PDF 中用 pdftotext 提取文本
4. 仍为空 → 若原 PDF 有文本则用原文本，否则空串 + 告警
```

**Mail Parser 文本合并**（`build_formatted_text()` L233-L261）：
```
Subject: ...
From: ...
To: ...
[CC: / BCC: / Attachments: ...]
HTML content: <Tika 从 HTML body 提取的纯文本>
<纯文本正文>
```

### 7.4 Date Parsing 插件兜底

当 Parser 自身没有找到日期时，Consumer 会调用 `src/documents/plugins/date_parsing/__init__.py` 中的机制：

**发现机制**：entrypoint 组 `paperless_ngx.date_parsers`
- 多个插件：按 `ep.name` 字母排序取第一个，其余发出 WARNING
- 无插件：使用内置 `RegexDateParserPlugin`

**配置注入**（`get_date_parser()` L64-L93）：
```python
DateParserConfig(
    languages=DATE_PARSER_LANGUAGES or OCR_LANGUAGE 推导,
    timezone_str=TIME_ZONE,
    ignore_dates=IGNORE_DATES,
    reference_time=timezone.now(),
    filename_date_order=FILENAME_DATE_ORDER,
    content_date_order=DATE_ORDER,
)
```

### 7.5 最终持久化（原子事务）

```
数据库（transaction.atomic() 内）:
  ├── _store() 或 _create_version_from_root()
  ├── apply_overrides()  — correspondent/doctype/tags/storage/asn/owner/permissions/custom_fields
  ├── document_consumption_finished 信号 → 自动匹配 / 分类器 / 索引
  └── FileLock(MEDIA_LOCK) 下写入文件（失败会回滚事务）
        ├── source → originals/
        ├── thumbnail → thumbnails/
        └── archive → archive/ + archive_checksum

事务外:
  └── run_post_consume_script(document)   — 失败不回滚
```

---

## 八、关键文件速查（相对路径）

| 文件 | 作用 |
|------|------|
| `src/paperless/parsers/__init__.py` | ParserProtocol 协议、MetadataEntry、ParserContext |
| `src/paperless/parsers/registry.py` | ParserRegistry、entrypoint 发现、评分选择算法 |
| `src/paperless/parsers/tesseract.py` | OCRmyPDF + Tesseract 本地 OCR（含 fallback 机制） |
| `src/paperless/parsers/remote.py` | Azure AI Document Intelligence 远程 OCR |
| `src/paperless/parsers/tika.py` | Tika 文本提取 + Gotenberg PDF 转换（Office） |
| `src/paperless/parsers/mail.py` | EML 邮件解析（imap_tools + Tika + Gotenberg，PdfLayout） |
| `src/paperless/parsers/text.py` | 纯文本/CSV 解析 |
| `src/paperless/parsers/utils.py` | 共享工具：extract_pdf_text / is_tagged_pdf / get_page_count_for_pdf / extract_pdf_metadata |
| `src/documents/consumer.py` | ConsumerPlugin、ConsumerPreflightPlugin、AsnCheckPlugin、WorkflowTriggerPlugin、qpdf 预处理、pre/post 脚本、should_produce_archive、_store/_create_version_from_root |
| `src/documents/tasks.py` | 消费任务插件链编排、版本分支与新文档分支的插件列表差异 |
| `src/documents/plugins/base.py` | ConsumeTaskPlugin 基类及 Mixin |
| `src/documents/plugins/date_parsing/__init__.py` | 日期解析插件发现与工厂 |
| `src/documents/plugins/date_parsing/base.py` | DateParserPluginBase 基类 |
| `src/documents/parsers.py` | 旧版 DocumentParser ABC（兼容层）+ MIME/缩略图工具函数（make_thumbnail_from_pdf 等） |
| `src/paperless_mail/models.py` | MailRule.PdfLayout 枚举定义 |
