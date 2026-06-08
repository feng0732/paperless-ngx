# Paperless-ngx Parser Plugin 体系代码分析

## 一、整体架构

Paperless-ngx 的文档处理分为两个层次的插件体系：

1. **ConsumeTaskPlugin 消费任务插件链**：编排整个文档消费流程（预检→条码→ASN→工作流→核心消费）
2. **ParserProtocol 文档解析器体系**：负责具体文件格式的文本提取、OCR、PDF 生成等

```
documents/tasks.py
    └── _try_consume_file()
          └── [插件链顺序执行]
                ├── ConsumerPreflightPlugin   (预检：文件存在/重复/目录)
                ├── AsnCheckPlugin           (ASN 校验)
                ├── CollatePlugin            (双面文档整理)
                ├── BarcodePlugin            (条码识别拆分)
                ├── AsnCheckPlugin           (再次 ASN 校验)
                ├── WorkflowTriggerPlugin    (工作流元数据覆盖)
                └── ConsumerPlugin           (核心消费 → 调用 Parser)
                                          └── paperless/parsers/registry.py
                                                └── ParserRegistry.get_parser_for_file()
                                                      ├── score 评分选择
                                                      └── ParserProtocol 实现
```

---

## 二、插件选择逻辑

### 2.1 消费任务插件链编排

插件链的定义和执行位于 [tasks.py](file:///d:/fz/0601/solo-dogfeeding/code/117-paperless-ngx/src/documents/tasks.py#L140-L218)。

**执行规则**（每个插件）：
1. 实例化时传入 `input_doc / metadata / status_mgr / tmp_dir / task_id`
2. `able_to_run == False` → 跳过
3. 依次调用 `setup() → run() → finally: cleanup()`
4. `run()` 返回值为信息日志；`metadata` 可被修改并传递给下一个插件
5. 抛出 `StopConsumeTaskError` → 终止整个任务并返回 `ConsumeFileStoppedResult`
6. 抛出 `ConsumeFileDuplicateError` → 返回 `ConsumeFileDuplicateResult`

插件基类与 Mixin 定义在 [base.py](file:///d:/fz/0601/solo-dogfeeding/code/117-paperless-ngx/src/documents/plugins/base.py#L23-L131)：
- `AlwaysRunPluginMixin`：`able_to_run` 恒为 True
- `NoSetupPluginMixin`：`setup()` 空实现
- `NoCleanupPluginMixin`：`cleanup()` 空实现

### 2.2 Parser 评分选择机制

核心注册中心 [registry.py](file:///d:/fz/0601/solo-dogfeeding/code/117-paperless-ngx/src/paperless/parsers/registry.py)。

**协议定义**：[ParserProtocol](file:///d:/fz/0601/solo-dogfeeding/code/117-paperless-ngx/src/paperless/parsers/__init__.py#L116-L434) 定义了所有 Parser 必须满足的结构契约（运行时可检查）。

**发现机制**：
- 内置 Parser：在 `register_defaults()` 中显式注册
- 第三方 Parser：通过 Python entrypoint 组 `paperless_ngx.parsers` 自动发现
  ```toml
  [project.entry-points."paperless_ngx.parsers"]
  my_parser = "my_package.parsers:MyParser"
  ```
- 校验属性：`name / version / author / url / supported_mime_types / score` 缺一不可

**选择算法**（[get_parser_for_file](file:///d:/fz/0601/solo-dogfeeding/code/117-paperless-ngx/src/paperless/parsers/registry.py#L332-L393)）：
1. 遍历顺序：`外部插件 → 内置插件`（同分时外部优先，first-seen policy）
2. 过滤条件：`mime_type in parser.supported_mime_types()` 且 `score() is not None`
3. 胜者：最高 `score()` 的 Parser

**各内置 Parser 分数与 MIME 覆盖**：

| Parser 类 | score | 支持 MIME | 激活条件 |
|-----------|-------|-----------|----------|
| [TextDocumentParser](file:///d:/fz/0601/solo-dogfeeding/code/117-paperless-ngx/src/paperless/parsers/text.py) | 10 | text/plain, text/csv, application/csv | 始终激活 |
| [RemoteDocumentParser](file:///d:/fz/0601/solo-dogfeeding/code/117-paperless-ngx/src/paperless/parsers/remote.py) | **20** | application/pdf, image/* (png/jpeg/tiff/bmp/gif/webp) | 配置了 REMOTE_OCR_ENGINE 等 |
| [TikaDocumentParser](file:///d:/fz/0601/solo-dogfeeding/code/117-paperless-ngx/src/paperless/parsers/tika.py) | 10 | Office 文档 (doc/docx/xls/xlsx/ppt/pptx/odt/ods/odp/rtf...) | TIKA_ENABLED=True |
| [MailDocumentParser](file:///d:/fz/0601/solo-dogfeeding/code/117-paperless-ngx/src/paperless/parsers/mail.py) | 10 | message/rfc822 (.eml) | 始终激活 |
| [RasterisedDocumentParser](file:///d:/fz/0601/solo-dogfeeding/code/117-paperless-ngx/src/paperless/parsers/tesseract.py) | 10 | application/pdf, image/* (含 heic) | 始终激活 |

> 注意：**RemoteDocumentParser** 的 score=20，高于 Tesseract 的 10，因此只要配置了 Azure AI，它就会接管 PDF 和图片的 OCR。

---

## 三、外部工具调用

### 3.1 Tesseract OCR — [tesseract.py](file:///d:/fz/0601/solo-dogfeeding/code/117-paperless-ngx/src/paperless/parsers/tesseract.py)

**调用路径**：`parse()` → `ocrmypdf.ocr(**args)`

参数构造由 [construct_ocrmypdf_parameters()](file:///d:/fz/0601/solo-dogfeeding/code/117-paperless-ngx/src/paperless/parsers/tesseract.py#L266-L383) 完成，涉及：
- **OCR 模式**：`force_ocr / redo_ocr / skip_text / auto`
- **图像预处理**：`clean / clean_final / deskew / rotate_pages`
- **语言**：`OCR_LANGUAGE`
- **输出类型**：`pdfa / pdf / pdfa-1 / pdfa-2 / pdfa-3`
- **图像专用**：`image_dpi`（读取文件 DPI→A4 估算→配置兜底），alpha 通道移除

**OCR 模式决策路径**（[parse()](file:///d:/fz/0601/solo-dogfeeding/code/117-paperless-ngx/src/paperless/parsers/tesseract.py#L494-L659)）：
```
OCR_MODE=off
  ├── 不生成归档 + PDF → 直接返回 pdftotext 文本
  ├── 图片输入 → img2pdf + pikepdf 打 PDF/A 标签（不调用 OCR）
  └── PDF 输入 → Ghostscript 直接转 PDF/A（不调用 OCR）

OCR_MODE=auto + 原文有文本 + 不生成归档
  └── 跳过 ocrmypdf，直接返回 pdftotext 结果

其他情况（完整 OCR 流程）：
  ├── 先调用 ocrmypdf.ocr()
  ├── 失败时 fallback：force_ocr 重试一次
  └── 文本提取优先 sidecar.txt → 否则提取 PDF 文本
```

**外部二进制**（由 ocrmypdf 间接调用）：`tesseract`、`gs` (Ghostscript)、`convert` (ImageMagick)、`unpaper`

### 3.2 Tika + Gotenberg — [tika.py](file:///d:/fz/0601/solo-dogfeeding/code/117-paperless-ngx/src/paperless/parsers/tika.py)

**客户端生命周期**：在 `__enter__` 中通过 `ExitStack` 统一创建 `TikaClient` 和 `GotenbergClient`，`__exit__` 时自动关闭。

**文本提取**（[parse()](file:///d:/fz/0601/solo-dogfeeding/code/117-paperless-ngx/src/paperless/parsers/tika.py#L212-L278)）：
```python
# 优先 multipart/form-data 上传
parsed = tika_client.tika.as_text.from_file(document_path, mime_type)
# 500 错误 fallback：直接传 bytes buffer
parsed = tika_client.tika.as_text.from_buffer(bytes, mime_type)
```

**PDF 转换**（[_convert_to_pdf()](file:///d:/fz/0601/solo-dogfeeding/code/117-paperless-ngx/src/paperless/parsers/tika.py#L400-L452)）：
- 走 Gotenberg 的 LibreOffice 路由
- 根据 `OCR_OUTPUT_TYPE` 设置 `PdfAFormat`（A2b / A3b，PDF/A-1a 降级为 A2b）
- 因为 `requires_pdf_rendition=True`，**即使 `produce_archive=False` 也必须生成 PDF**（浏览器无法显示 Office 原生格式）

### 3.3 Remote OCR (Azure AI Vision) — [remote.py](file:///d:/fz/0601/solo-dogfeeding/code/117-paperless-ngx/src/paperless/parsers/remote.py)

配置校验由 [RemoteEngineConfig](file:///d:/fz/0601/solo-dogfeeding/code/117-paperless-ngx/src/paperless/parsers/remote.py#L46-L65) 完成：
- `engine ∈ ("azureai",)` + `api_key` + `endpoint`（azureai 需要）

**调用流程**（[_azure_ai_vision_parse()](file:///d:/fz/0601/solo-dogfeeding/code/117-paperless-ngx/src/paperless/parsers/remote.py#L361-L433)）：
```python
client = DocumentIntelligenceClient(endpoint, AzureKeyCredential(api_key))

# 1. 提交分析任务（长轮询）
poller = client.begin_analyze_document(
    model_id="prebuilt-read",
    body=AnalyzeDocumentRequest(bytes_source=f.read()),
    output_content_format=DocumentContentFormat.TEXT,
    output=[AnalyzeOutputOption.PDF],       # 请求生成 searchable PDF
)
poller.wait()

# 2. 获取文本
result = poller.result()
text = result.content

# 3. 流式下载 searchable PDF 归档
for chunk in client.get_analyze_result_pdf(model_id="prebuilt-read", result_id=operation_id):
    f.write(chunk)
```

### 3.4 Mail Parser — [mail.py](file:///d:/fz/0601/solo-dogfeeding/code/117-paperless-ngx/src/paperless/parsers/mail.py)

EML 解析涉及多个外部组件协作：

```
imap_tools.MailMessage.from_bytes()
  └── 提取邮件头、纯文本、HTML、附件

文本组装（build_formatted_text）：
  ├── 邮件头 (Subject/From/To/CC/BCC/Attachments)
  ├── HTML 正文 → Tika 服务器提取纯文本 ([tika_parse()](file:///d:/fz/0601/solo-dogfeeding/code/117-paperless-ngx/src/paperless/parsers/mail.py#L501-L532))
  └── 纯文本正文

PDF 生成（generate_pdf）：
  ├── 邮件头+纯文本 → Django 模板渲染 HTML → Gotenberg Chromium HTML→PDF
  ├── HTML 正文 → Gotenberg Chromium HTML→PDF（带附件资源）
  └── 两者按 MailRule.PdfLayout 合并（TEXT_HTML / HTML_TEXT / HTML_ONLY / TEXT_ONLY）
       └── Gotenberg merge 路由
```

---

## 四、结果合并逻辑

### 4.1 ConsumerPlugin 中的结果采集

结果从 Parser 取出后在 [ConsumerPlugin.run()](file:///d:/fz/0601/solo-dogfeeding/code/117-paperless-ngx/src/documents/consumer.py#L408-L784) 中被汇总，核心采集代码位于 L499-L553：

```python
with parser_class() as document_parser:
    document_parser.configure(ParserContext(mailrule_id=...))

    # 1. 决定是否生成归档 PDF
    produce_archive = should_produce_archive(document_parser, mime_type, working_copy)
    document_parser.parse(working_copy, mime_type, produce_archive=produce_archive)

    # 2. 按顺序提取结果
    text        = document_parser.get_text()
    date        = document_parser.get_date()
    thumbnail   = document_parser.get_thumbnail(working_copy, mime_type)
    archive_path = document_parser.get_archive_path()
    page_count  = document_parser.get_page_count(working_copy, mime_type)

    # 3. 日期兜底：Parser 没找到 → 用 date_parsing 插件
    if date is None:
        with get_date_parser() as date_parser:
            date = next(date_parser.parse(filename, text), None)
```

### 4.2 `should_produce_archive` 决策逻辑

[should_produce_archive()](file:///d:/fz/0601/solo-dogfeeding/code/117-paperless-ngx/src/documents/consumer.py#L124-L189)：
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

### 4.3 文本合并的特殊策略

**Tesseract 文本提取**（[extract_text()](file:///d:/fz/0601/solo-dogfeeding/code/117-paperless-ngx/src/paperless/parsers/tesseract.py#L236-L264)）：
```
1. 优先读 sidecar.txt（ocrmypdf 生成的纯文本），除非 REDO 模式
2. sidecar 不完整（含 "[OCR skipped on page"）→ 丢弃
3. 兜底：从生成的归档 PDF 中提取文本
4. 最终仍为空 → 若原 PDF 有文本则用原文，否则置空并告警
```

**Mail Parser 文本合并**（[build_formatted_text()](file:///d:/fz/0601/solo-dogfeeding/code/117-paperless-ngx/src/paperless/parsers/mail.py#L233-L261)）：
```
邮件头 (Subject/From/To/CC/BCC/Attachments)
+ HTML 内容: (Tika 提取的纯文本)
+ 纯文本正文
```

### 4.4 Date Parsing 插件兜底

当 Parser 自身没有找到日期时，Consumer 会调用 [date_parsing 插件](file:///d:/fz/0601/solo-dogfeeding/code/117-paperless-ngx/src/documents/plugins/date_parsing/__init__.py)：

**发现机制**：entrypoint 组 `paperless_ngx.date_parsers`
- 多个插件：按名称排序取第一个，其余告警
- 无插件：使用内置 `RegexDateParserPlugin`

**配置注入**（[get_date_parser()](file:///d:/fz/0601/solo-dogfeeding/code/117-paperless-ngx/src/documents/plugins/date_parsing/__init__.py#L64-L93)）：
```python
DateParserConfig(
    languages=...,                 # DATE_PARSER_LANGUAGES 或从 OCR_LANGUAGE 推导
    timezone_str=settings.TIME_ZONE,
    ignore_dates=settings.IGNORE_DATES,
    reference_time=timezone.now(),
    filename_date_order=settings.FILENAME_DATE_ORDER,
    content_date_order=settings.DATE_ORDER,
)
```

### 4.5 最终持久化（原子事务）

在 [_store()](file:///d:/fz/0601/solo-dogfeeding/code/117-paperless-ngx/src/documents/consumer.py#L815-L875) 和后续的文件写入中：

```
数据库（事务内）：
  ├── Document 记录 (title / content / mime_type / checksum / created / page_count)
  ├── 应用元数据覆盖 (correspondent / document_type / tags / storage_path / asn / owner / permissions / custom_fields)
  └── 发送 document_consumption_finished 信号 → 触发自动匹配、分类器、索引等

文件系统（事务内，FileLock 保护）：
  ├── original → source_path
  ├── thumbnail.webp → thumbnail_path
  └── archive.pdf → archive_path (若存在) + archive_checksum
```

---

## 五、关键文件速查

| 文件 | 作用 |
|------|------|
| [src/paperless/parsers/__init__.py](file:///d:/fz/0601/solo-dogfeeding/code/117-paperless-ngx/src/paperless/parsers/__init__.py) | ParserProtocol 协议、MetadataEntry、ParserContext |
| [src/paperless/parsers/registry.py](file:///d:/fz/0601/solo-dogfeeding/code/117-paperless-ngx/src/paperless/parsers/registry.py) | ParserRegistry、entrypoint 发现、评分选择 |
| [src/paperless/parsers/tesseract.py](file:///d:/fz/0601/solo-dogfeeding/code/117-paperless-ngx/src/paperless/parsers/tesseract.py) | OCRmyPDF + Tesseract 本地 OCR |
| [src/paperless/parsers/remote.py](file:///d:/fz/0601/solo-dogfeeding/code/117-paperless-ngx/src/paperless/parsers/remote.py) | Azure AI Document Intelligence 远程 OCR |
| [src/paperless/parsers/tika.py](file:///d:/fz/0601/solo-dogfeeding/code/117-paperless-ngx/src/paperless/parsers/tika.py) | Tika 文本提取 + Gotenberg PDF 转换（Office） |
| [src/paperless/parsers/mail.py](file:///d:/fz/0601/solo-dogfeeding/code/117-paperless-ngx/src/paperless/parsers/mail.py) | EML 邮件解析（imap_tools + Tika + Gotenberg） |
| [src/paperless/parsers/text.py](file:///d:/fz/0601/solo-dogfeeding/code/117-paperless-ngx/src/paperless/parsers/text.py) | 纯文本/CSV 解析 |
| [src/documents/consumer.py](file:///d:/fz/0601/solo-dogfeeding/code/117-paperless-ngx/src/documents/consumer.py) | ConsumerPlugin / 预检 / ASN / 工作流触发插件 |
| [src/documents/tasks.py](file:///d:/fz/0601/solo-dogfeeding/code/117-paperless-ngx/src/documents/tasks.py#L140-L218) | 消费任务插件链编排执行 |
| [src/documents/plugins/base.py](file:///d:/fz/0601/solo-dogfeeding/code/117-paperless-ngx/src/documents/plugins/base.py) | ConsumeTaskPlugin 基类 |
| [src/documents/plugins/date_parsing/__init__.py](file:///d:/fz/0601/solo-dogfeeding/code/117-paperless-ngx/src/documents/plugins/date_parsing/__init__.py) | 日期解析插件发现与工厂 |
| [src/documents/parsers.py](file:///d:/fz/0601/solo-dogfeeding/code/117-paperless-ngx/src/documents/parsers.py) | 旧版 DocumentParser ABC（兼容层）+ MIME/缩略图工具函数 |
