# Paperless-ngx 归档文件（Archive）与 PDF/A 处理深度解析

本文档对照源代码，系统梳理 Paperless-ngx 中归档文件的生成决策、PDF/A 格式转换以及失败回退的完整行为链条。

---

## 1. 核心概念

### 1.1 Archive 文件是什么

Paperless-ngx 为每份文档维护两份文件：

| 文件 | 存储位置 | 说明 |
|------|---------|------|
| **Original（原件）** | `ORIGINALS_DIR` | 用户上传的原始文件，格式不变 |
| **Archive（归档件）** | `ARCHIVE_DIR` | 可选的 PDF/A 格式副本，含 OCR 文本层，便于长期归档和全文检索 |

归档件的核心价值：
- 统一为 PDF 格式，便于前端预览
- 转为 PDF/A 标准（ISO 19005），适合长期数字保存
- 嵌入 OCR 识别的文本层，使扫描件变为可搜索 PDF

### 1.2 关键配置枚举

定义在 [paperless/models.py](file:///d:/fz/0601/solo-dogfeeding/code/115-paperless-ngx/src/paperless/models.py)：

**ArchiveFileGenerationChoices** — 是否生成归档件
- `auto`（默认）：智能判断
- `always`：始终生成
- `never`：永不生成

**OutputTypeChoices** — 归档件输出格式
- `pdf`：普通 PDF（不做 PDF/A 转换）
- `pdfa`：PDF/A-2b（默认）
- `pdfa-1`：PDF/A-1b
- `pdfa-2`：PDF/A-2b
- `pdfa-3`：PDF/A-3b

**ModeChoices** — OCR 运行模式
- `auto`（默认）：有文本则跳过 OCR，无文本则 OCR
- `force`：强制对所有页面重新 OCR
- `redo`：重做已有 OCR 的页面
- `off`：完全禁用 OCR 引擎

---

## 2. 归档文件生成决策：`should_produce_archive()`

这是整个归档流程的入口阀门，决定「这份文档要不要生成归档件」。

代码位置：[documents/consumer.py:124-189](file:///d:/fz/0601/solo-dogfeeding/code/115-paperless-ngx/src/documents/consumer.py#L124-L189)

### 2.1 决策优先级（从高到低）

```
1. parser.requires_pdf_rendition == True   → 必须生成（浏览器无法直接显示的格式，如 DOCX）
2. parser.can_produce_archive == False     → 不能生成（如纯文本解析器）
3. ARCHIVE_FILE_GENERATION = "always"      → 始终生成
4. ARCHIVE_FILE_GENERATION = "never"       → 永不生成
5. ARCHIVE_FILE_GENERATION = "auto"        → 智能判断（见下文）
```

### 2.2 `auto` 模式的智能判断逻辑

```
输入是 image/*        → 生成归档件（图片必须包装成 PDF 才能预览）
输入是 application/pdf → 进一步判断：
    ├─ is_tagged_pdf() == True         → 不生成（带结构标签的原生数字 PDF）
    ├─ extract_pdf_text() 文本 ≤ 50 字符 → 生成（扫描件，文本极少或无）
    └─ 文本 > 50 字符                   → 不生成（原生数字 PDF 已有充足文本）
其他 MIME 类型         → 不生成
```

关键阈值常量 `PDF_TEXT_MIN_LENGTH = 50` 定义在 [paperless/parsers/utils.py:25](file:///d:/fz/0601/solo-dogfeeding/code/115-paperless-ngx/src/paperless/parsers/utils.py#L25)。

### 2.3 `is_tagged_pdf()` 判断原理

代码位置：[paperless/parsers/utils.py:28-64](file:///d:/fz/0601/solo-dogfeeding/code/115-paperless-ngx/src/paperless/parsers/utils.py#L28-L64)

使用 `pikepdf` 检查 PDF 根目录是否存在 `/MarkInfo` 字典且 `/Marked` 为 `true`。这是 Word、LibreOffice 等导出的「结构化 PDF」的标志——这类文档已有完整文本层，无需 OCR。

### 2.4 决策调用链

文档消费流程中调用点：

1. **消费阶段**：[documents/consumer.py:514-519](file:///d:/fz/0601/solo-dogfeeding/code/115-paperless-ngx/src/documents/consumer.py#L514-L519) — 在 `ConsumerPlugin.run()` 中调用
2. **归档命令阶段**：[documents/tasks.py:305-309](file:///d:/fz/0601/solo-dogfeeding/code/115-paperless-ngx/src/documents/tasks.py#L305-L309) — 在 `update_document_content_maybe_archive_file()` 中调用

---

## 3. PDF/A 格式转换实现

Tesseract 解析器（`RasterisedDocumentParser`）是实际执行 PDF/A 转换的主力。

代码位置：[paperless/parsers/tesseract.py](file:///d:/fz/0601/solo-dogfeeding/code/115-paperless-ngx/src/paperless/parsers/tesseract.py)

### 3.1 三条转换路径

根据 `OCR_MODE` 和输入文件类型，`parse()` 方法选择不同路径：

```
parse()
  │
  ├─ OCR_MODE = "off" （完全不跑 OCR 引擎）
  │    ├─ 输入是 image → _convert_image_to_pdfa()  （img2pdf + pikepdf）
  │    └─ 输入是 pdf   → _convert_pdf_to_pdfa()     （Ghostscript 直接转换）
  │
  ├─ OCR_MODE = "auto" + 已有文本 + 无需归档 → 直接返回 pdftotext 内容，跳过 OCRmyPDF
  │
  └─ 其他情况 → 调用 ocrmypdf.ocr()
       ├─ skip_text=True （已有文本，只做 PDF/A 转换，不做 OCR）
       └─ skip_text=False（完整 OCR + PDF/A 转换）
```

### 3.2 路径一：图片 → PDF/A（无 OCR）

方法：[_convert_image_to_pdfa()](file:///d:/fz/0601/solo-dogfeeding/code/115-paperless-ngx/src/paperless/parsers/tesseract.py#L385-L435)

**步骤：**

1. **img2pdf 包装**：使用 `img2pdf.convert()` 将图片无损包装为普通 PDF（不重新编码像素）
   - 若配置了 `OCR_IMAGE_DPI`，使用固定 DPI 布局函数
2. **pikepdf 打 PDF/A 标签**：
   - 注入 sRGB ICC 色彩配置文件（`ocrmypdf.data.sRGB.icc`）
   - 创建 `/OutputIntents` 字典，类型为 `/GTS_PDFA1`
   - 设置 XMP 元数据：`pdfaid:part = "2"`，`pdfaid:conformance = "B"`
   - 即 PDF/A-2b 级别

**特点**：不调用 Tesseract、不调用 Ghostscript，速度快，资源占用低。

### 3.3 路径二：PDF → PDF/A（无 OCR）

方法：[_convert_pdf_to_pdfa()](file:///d:/fz/0601/solo-dogfeeding/code/115-paperless-ngx/src/paperless/parsers/tesseract.py#L437-L476)

**步骤：**

1. 若 `output_type = "pdf"`，直接拷贝原文件，不做 PDF/A 转换
2. 否则调用 `ocrmypdf._exec.ghostscript.generate_pdfa()`：
   - 先由 `generate_pdfa_ps()` 生成 PDF/A 定义 PostScript 文件（`pdfa.ps`）
   - Ghostscript 将 `[pdfa.ps, input.pdf]` 合并渲染，输出 PDF/A
   - PDF/A 版本映射：`pdfa→2`，`pdfa-1→1`，`pdfa-2→2`，`pdfa-3→3`
   - 色彩转换策略使用 `color_conversion_strategy`（默认 `RGB`）

### 3.4 路径三：OCRmyPDF 完整处理

OCRmyPDF 负责最复杂的场景：扫描件 OCR + 生成 PDF/A。

参数构造方法：[construct_ocrmypdf_parameters()](file:///d:/fz/0601/solo-dogfeeding/code/115-paperless-ngx/src/paperless/parsers/tesseract.py#L266-L383)

**关键参数映射：**

| Paperless 配置 | OCRmyPDF 参数 | 说明 |
|---------------|--------------|------|
| `OCR_MODE=force` | `--force-ocr` | 强制所有页面重新 OCR |
| `OCR_MODE=redo` | `--redo-ocr` | 重做已有 OCR 层的页面 |
| `OCR_MODE=auto/off` + 已有文本 | `--skip-text` | 跳过含文本页面，只做 PDF/A 转换 |
| `OCR_OUTPUT_TYPE` | `--output-type` | pdf / pdfa / pdfa-N |
| `OCR_CLEAN=clean` | `--clean` | 用 unpaper 清理图像 |
| `OCR_CLEAN=clean-final` | `--clean-final` | 最终输出清理（与 redo-ocr 不兼容）|
| `OCR_DESKEW` | `--deskew` | 自动纠偏（与 redo-ocr 不兼容）|
| `OCR_ROTATE_PAGES` | `--rotate-pages` | 自动旋转 |
| 图片输入 | `--image-dpi` | 图像 DPI（从 EXIF 读取，或估算 A4 宽度，或使用配置）|

**图片额外处理**：
- 含 Alpha 通道的图片（RGBA/LA）先用 ImageMagick `-alpha off` 移除透明度（img2pdf 不支持）
- DPI 不足时抛出 `ParseError`，要求用户配置 `OCR_IMAGE_DPI`

**PDF 文本层 vs 数据库 content 字段（Tesseract 路径）**：

| 维度 | 值 | 来源 |
|------|---|------|
| `Document.content`（DB 字段） | OCR 识别的纯文本 | `self.extract_text()`：优先从 OCRmyPDF sidecar.txt 读取，否则从归档 PDF 本身用 pdftotext 提取 |
| 归档 PDF 是否含文本层 | **有（OCR 结果嵌入）** | OCRmyPDF 将 Tesseract 识别的文本以隐形方式叠加在 PDF 图像之上，形成可搜索 PDF |
| 两者一致性 | **高度一致** | 两者均来自同一次 Tesseract 识别结果；sidecar.txt 与 PDF 文本层是同一份数据的两种输出格式 |

具体代码流程（[tesseract.py](file:///d:/fz/0601/solo-dogfeeding/code/115-paperless-ngx/src/paperless/parsers/tesseract.py)）：
1. OCRmyPDF 执行 `ocrmypdf.ocr()` → 同时产出 `archive.pdf`（含隐形文本层）和 `sidecar.txt`（纯文本副本）
2. `extract_text(sidecar_file, archive_path)` 方法（第 236-264 行）优先读取 sidecar.txt，若不完整则 fallback 到 `extract_pdf_text(archive.pdf)`
3. Consumer 通过 `parser.get_text()` 拿到文本，写入 `Document.content`
4. Consumer 通过 `parser.get_archive_path()` 拿到 PDF，落盘到 ARCHIVE_DIR
5. **两条管道共享数据源**，DB content 与 PDF 文本层本质相同

特殊分支（OCR_MODE=off + 图片）：
- `self.text = ""`（第 546 行），DB content 为空
- 归档 PDF 只是图片包装，**无文本层**，无法在 PDF 内搜索

### 3.5 路径四：Office 文档（Tika + Gotenberg）

处理 DOCX/ODT/XLSX/PPTX/RTF 等格式，由 `TikaDocumentParser` 实现。

代码位置：[paperless/parsers/tika.py](file:///d:/fz/0601/solo-dogfeeding/code/115-paperless-ngx/src/paperless/parsers/tika.py)

**前置条件**：
- `PAPERLESS_TIKA_ENABLED=true`（否则 `score()` 返回 `None`，该解析器不注册）
- 需要同时运行 **Apache Tika**（文本提取）和 **Gotenberg**（PDF 转换）两个外部服务

**Parser 能力声明**：
```
can_produce_archive  = False   ← 不认为是"可选的 OCR 归档"
requires_pdf_rendition = True  ← 浏览器无法直接显示 Office 格式，必须生成 PDF
```
因此 `should_produce_archive()` 永远返回 `True`，`produce_archive` 参数在 `parse()` 中被**完全忽略**——PDF 总是生成。

**完整流程**：

```
parse(document_path, mime_type, produce_archive=...)
  │
  ├─ 1. Tika 文本提取（HTTP 请求 TIKA_ENDPOINT）
  │    ├─ 优先: multipart/form 上传文件
  │    └─ 500 错误回退: 以二进制 buffer 方式重新提交（Tika TIKA-4110 缺陷的 workaround）
  │    ├─ 失败 → ParseError 终止
  │    └─ 得到 self._text + self._date（文档创建日期）
  │
  └─ 2. Gotenberg PDF 转换（_convert_to_pdf()）
       │
       ├─ 读取 OutputTypeConfig().output_type
       │
       ├─ 映射到 Gotenberg PdfAFormat：
       │    ├─ pdfa / pdfa-2  → PdfAFormat.A2b
       │    ├─ pdfa-1         → 不支持！日志警告，降级为 A2b
       │    ├─ pdfa-3         → PdfAFormat.A3b
       │    └─ pdf           → 不调用 route.pdf_format()，输出普通 PDF
       │
       └─ Gotenberg LibreOffice 路由 (libre_office.to_pdf())
            └─ 返回 convert.pdf
```

**关键方法**：
- [_convert_to_pdf()](file:///d:/fz/0601/solo-dogfeeding/code/115-paperless-ngx/src/paperless/parsers/tika.py#L400-L452) — LibreOffice 转 PDF + PDF/A 版本映射

**PDF 文本层 vs 数据库 content 字段（Tika 路径）**：

| 维度 | 值 | 来源 |
|------|---|------|
| `Document.content`（DB 字段） | Tika 服务器提取的纯文本 | `self._tika_client.tika.as_text.from_file()` → `self._text` → `parser.get_text()` |
| 归档 PDF 是否含文本层 | 视 LibreOffice 而定（通常有） | LibreOffice 导出 PDF 时自动嵌入的可复制文本 |
| 两者一致性 | **不一定一致** | Tika 与 LibreOffice 是两个独立引擎，提取/嵌入文本的顺序、格式、换行符可能不同 |

具体流程：
1. `parse()` 中 `self._text = parsed.content`（第 268 行）— 这是 Tika 文本提取结果
2. `parse()` 中 `self._archive_path = self._convert_to_pdf()`（第 278 行）— 这是 Gotenberg/LibreOffice 渲染的 PDF
3. Consumer 层通过 `parser.get_text()` 获取第 1 步的文本，写入 `Document.content`
4. Consumer 层通过 `parser.get_archive_path()` 获取第 2 步的 PDF，落盘到 ARCHIVE_DIR
5. 两条管道完全独立，Tika 的文本不会被注入到 PDF 中

---

### 3.6 路径五：邮件 EML（Mail Parser + Gotenberg）

处理 `message/rfc822`（.eml）邮件文件，由 `MailDocumentParser` 实现。

代码位置：[paperless/parsers/mail.py](file:///d:/fz/0601/solo-dogfeeding/code/115-paperless-ngx/src/paperless/parsers/mail.py)

**Parser 能力声明**（与 Tika 完全相同）：
```
can_produce_archive  = False
requires_pdf_rendition = True
```
PDF 始终生成，与 `produce_archive` 无关。

**完整流程**：

```
parse(document_path, mime_type)
  │
  ├─ 1. imap_tools 解析 EML → MailMessage 对象
  │    └─ parse_file_to_message()，缺少 From 头则 ParseError
  │
  ├─ 2. 组装格式化文本（Subject/From/To/CC/BCC/附件列表 + HTML 文本 + 纯文本）
  │    └─ HTML 内容经 Tika 服务器提取纯文本（tika_parse()）
  │
  ├─ 3. 生成 PDF — generate_pdf()
  │    │
  │    ├─ A. 邮件正文 → HTML 模板渲染 (mail_to_html())
  │    │    └─ Gotenberg Chromium HTML→PDF（chromium.html_to_pdf()）
  │    │         └─ 应用 A4 纸张、0.1 英寸页边距、email_msg_template.html + output.css
  │    │
  │    ├─ B. 邮件 HTML 正文（如果有）→ generate_pdf_from_html()
  │    │    ├─ <script> 标签替换为 <div hidden>（安全清洗）
  │    │    ├─ 附件以 cid: 引用写入临时文件并注册为资源
  │    │    └─ Gotenberg Chromium HTML→PDF
  │    │
  │    └─ C. 按 MailRule.PdfLayout 合并（Gotenberg merge 路由）
  │         ├─ TEXT_HTML（默认）: [正文PDF, HTML内容PDF]
  │         ├─ HTML_TEXT:         [HTML内容PDF, 正文PDF]
  │         ├─ HTML_ONLY:         仅 HTML 内容
  │         └─ TEXT_ONLY:         仅正文
  │
  └─ 4. 每一步 Gotenberg 调用都应用 PDF/A 格式化（_settings_to_gotenberg_pdfa()）
```

**PDF/A 版本映射**（[_settings_to_gotenberg_pdfa()](file:///d:/fz/0601/solo-dogfeeding/code/115-paperless-ngx/src/paperless/parsers/mail.py#L452-L466)，与 Tika 逻辑一致）：

| `OCR_OUTPUT_TYPE` | Gotenberg `PdfAFormat` | 说明 |
|-------------------|------------------------|------|
| `pdfa` / `pdfa-2` | `A2b` | 默认 PDF/A-2b |
| `pdfa-1` | `A2b` | Gotenberg 不支持 A1，日志警告后降级 |
| `pdfa-3` | `A3b` | PDF/A-3b |
| `pdf` | `None` | 不设置 `pdf_format()`，输出普通 PDF |

**注意**：邮件解析器在三处独立调用 Gotenberg（正文 HTML→PDF、邮件 HTML→PDF、合并），每处都会单独应用一次 PDF/A 设置。

**邮件有/无 HTML 分支的 Gotenberg 调用差异**：

代码位置：[generate_pdf()](file:///d:/fz/0601/solo-dogfeeding/code/115-paperless-ngx/src/paperless/parsers/mail.py#L534-L606)

```
if not mail_message.html:          # 无 HTML 正文（纯文本邮件）
    archive_path = mail_pdf_file   # 直接使用正文模板渲染的 PDF
    └─ Gotenberg 调用次数 = 1 次
       (仅 generate_pdf_from_mail → Chromium HTML→PDF)
    └─ PDF/A 应用次数 = 1 次
       (仅在 Chromium 阶段，无 Merge 二次处理)
    └─ 不调用 Gotenberg Merge 路由

else:                              # 有 HTML 正文（富文本邮件）
    pdf_of_html_content = generate_pdf_from_html(...)
    Gotenberg Merge 合并 [mail_pdf_file, pdf_of_html_content]
    └─ Gotenberg 调用次数 = 3 次
       1. generate_pdf_from_mail     → Chromium HTML→PDF（正文模板）
       2. generate_pdf_from_html     → Chromium HTML→PDF（邮件 HTML 正文）
       3. merge                      → Gotenberg Merge 按 PdfLayout 合并
    └─ PDF/A 应用次数 = 3 次
       (每次 Gotenberg 调用都独立调用 _settings_to_gotenberg_pdfa())
    └─ Merge 阶段按 PdfLayout 决定最终组成：
         TEXT_HTML : [正文PDF, HTML内容PDF]
         HTML_TEXT : [HTML内容PDF, 正文PDF]
         HTML_ONLY : [HTML内容PDF]           ← 不包含正文模板
         TEXT_ONLY : [正文PDF]                ← 不包含邮件 HTML 正文
```

**HTML 正文安全处理**（[generate_pdf_from_html()](file:///d:/fz/0601/solo-dogfeeding/code/115-paperless-ngx/src/paperless/parsers/mail.py#L739-L834)）：
- `<script>` 标签整体替换为 `<div hidden>`，防止邮件中恶意脚本在 Chromium 渲染时执行
- 内联附件 `cid:xxx` 引用写入临时文件并通过 `route.resource()` 注册给 Chromium
- 文件名只保留字母数字，防止路径注入

**PDF 文本层 vs 数据库 content 字段（Mail 路径）**：

| 维度 | 值 | 来源 |
|------|---|------|
| `Document.content`（DB 字段） | `build_formatted_text()` 拼接结果 | `Subject:`/`From:`/`To:`/`CC:`/`BCC:`/`Attachments:` 头 + HTML 内容经 Tika 提取的纯文本 + mail.text 纯文本 |
| 归档 PDF 是否含文本层 | 通常有 | Chromium 渲染 HTML 时自动嵌入的可复制文本（含 HTML 标签残留、排版格式字符） |
| 两者一致性 | **差异较大** | DB content 是结构化的头字段 + 清洗后的纯文本；PDF 文本层是 HTML 渲染的视觉输出，顺序、格式完全不同 |

关键代码：[build_formatted_text()](file:///d:/fz/0601/solo-dogfeeding/code/115-paperless-ngx/src/paperless/parsers/mail.py#L233-L261) 在 `parse()` 内联定义，拼接顺序：
```
Subject: {subject}
From: {from}
To: {to_list}
CC: {cc_list}        [可选]
BCC: {bcc_list}      [可选]
Attachments: ...     [可选]
HTML content: {tika_parse(mail.html)}   [仅当有 HTML 正文时]
{mail.text}                         [纯文本正文]
```

### 3.7 Gotenberg PDF/A 版本映射与能力限制

Gotenberg 的 `PdfAFormat` 枚举仅支持 `A2b` 和 `A3b`，**不支持 PDF/A-1**。

这导致一个重要的"静默降级"行为：当用户将 `PAPERLESS_OCR_OUTPUT_TYPE` 设为 `pdfa-1`（PDF/A-1b）时：

- **Tesseract 路径**：OCRmyPDF/Ghostscript 原生支持 pdfa-1，正常输出 PDF/A-1b
- **Tika/Mail 路径**：Gotenberg 不支持，日志打印警告：
  ```
  Gotenberg does not support PDF/A-1a, choosing PDF/A-2b instead
  ```
  实际输出 **PDF/A-2b**，与用户配置不一致

---

## 4. 三条归档路径的核心差异对比

| 维度 | Tesseract（OCR） | Tika（Office） | Mail（EML） |
|------|------------------|----------------|-------------|
| **适用格式** | PDF, JPEG, PNG, TIFF, GIF, BMP, WebP, HEIC | DOCX, ODT, XLSX, PPTX, RTF 等 | `.eml` (message/rfc822) |
| **能力声明** | `can_produce_archive=True`, `requires_pdf_rendition=False` | `can_produce_archive=False`, `requires_pdf_rendition=True` | 同 Tika |
| **`should_produce_archive()` 影响** | 完全服从（受 auto/always/never 控制） | 总是 True（`requires_pdf_rendition` 强制） | 总是 True |
| **`produce_archive` 参数** | 被遵守：False 时不生成归档 PDF | 被忽略：PDF 始终生成 | 被忽略 |
| **PDF 转换引擎** | OCRmyPDF（底层 Ghostscript + Tesseract） | Gotenberg → LibreOffice | Gotenberg → Chromium（HTML→PDF）+ Merge |
| **是否需要外部服务** | 否（全部本地二进制） | 是（Tika + Gotenberg，两个 HTTP 服务） | 是（Gotenberg，可选 Tika 用于 HTML 文本提取） |
| **PDF/A-1 支持** | 是（Ghostscript 原生） | **否**（降级为 A2b + 警告） | **否**（降级为 A2b + 警告） |
| **PDF/A 版本来源** | `output_type` → pdfa_part 字符串（`pdfa→2` 等） | `OutputTypeConfig().output_type` → `PdfAFormat` 枚举 | `settings.OCR_OUTPUT_TYPE` → `PdfAFormat` 枚举 |
| **配置命名空间** | `OcrConfig`（含 mode/language/deskew 等 OCR 选项） | `OutputTypeConfig`（仅 output_type） | 直接读 `settings.OCR_OUTPUT_TYPE` |
| **DB `Document.content` 来源** | `self.extract_text()`：优先 OCRmyPDF sidecar.txt，fallback 到 pdftotext 提取归档 PDF | Tika 服务器 `tika.as_text.from_file()` 纯文本提取 | `build_formatted_text()`：邮件头 + HTML 经 Tika 提取 + mail.text 的拼接结果 |
| **归档 PDF 是否含文本层** | **有**（OCRmyPDF 将 Tesseract 识别的隐形文本叠加在图像上） | **视 LibreOffice 而定**（通常有，是 LibreOffice 导出 PDF 时自带的） | **通常有**（Chromium 渲染 HTML 时自动嵌入，可能含 HTML 标签残留） |
| **DB content 与 PDF 文本层是否同源** | **是** — 同一次 Tesseract 识别，两条输出管道 | **否** — 独立引擎：Tika 提取 vs LibreOffice 嵌入 | **否** — 结构化头字段+纯文本拼接 vs Chromium HTML 渲染输出 |
| **DB content 与 PDF 文本层一致性** | **高度一致** | **不一定一致**（顺序、换行、格式可能不同） | **差异较大**（结构化 vs 视觉化） |
| **Gotenberg 调用次数** | 0（不使用 Gotenberg） | 1 次（LibreOffice 路由） | 无 HTML：1 次（仅 Chromium 正文）；有 HTML：3 次（正文 Chromium + HTML Chromium + Merge） |
| **PDF/A 应用次数** | 1 次（OCRmyPDF/Ghostscript 输出阶段） | 1 次（LibreOffice 路由） | 无 HTML：1 次；有 HTML：3 次（每次 Gotenberg 调用独立应用） |
| **失败回退机制** | Force OCR 重试、加密 PDF 跳过、元数据打标降级为普通 PDF、Ghostscript 软错误提示 | Tika 500 重试 buffer 模式；Gotenberg 失败直接 ParseError | 无特殊回退，Gotenberg 失败直接 ParseError；PDF/A-1 降级为 A2b |

---

## 5. 失败回退（Fallback）机制

系统在多处设计了降级回退策略，确保「尽量产出可用结果」而不是直接失败。

### 5.1 Tesseract：PDF/A 元数据打标失败回退

位置：[_convert_image_to_pdfa()](file:///d:/fz/0601/solo-dogfeeding/code/115-paperless-ngx/src/paperless/parsers/tesseract.py#L429-L433)

```python
try:
    # pikepdf 打 PDF/A 标签 ...
except Exception as e:
    logger.warning("PDF/A metadata stamping failed; falling back to plain PDF.")
    pdfa_path.write_bytes(plain_pdf_path.read_bytes())
```

**行为**：pikepdf 注入 PDF/A 元数据失败时，直接使用 img2pdf 输出的普通 PDF。**结果仍然是可用的 PDF，只是不是 PDF/A 标准格式。**

### 5.2 Tesseract：OCR 主流程失败 → Force OCR 回退

位置：[RasterisedDocumentParser.parse()](file:///d:/fz/0601/solo-dogfeeding/code/115-paperless-ngx/src/paperless/parsers/tesseract.py#L619-L646)

触发条件：捕获以下异常时
- `NoTextFoundException`：OCR 后未提取到任何文本
- `InputFileError`：OCRmyPDF 认为输入文件损坏
- `PriorOcrFoundError`：检测到已有 OCR 层但 redo 未开启

**回退动作**：
1. 创建新的输出文件 `archive-fallback.pdf` 和 `sidecar-fallback.txt`
2. 用 `safe_fallback=True` 重新构造参数 → 强制启用 `--force-ocr`
3. 再次调用 `ocrmypdf.ocr()`
4. 若仍失败，抛出 `ParseError` 终止

### 5.3 Tesseract：加密/签名 PDF 回退

位置：[tesseract.py:610-616](file:///d:/fz/0601/solo-dogfeeding/code/115-paperless-ngx/src/paperless/parsers/tesseract.py#L610-L616)

捕获 `DigitalSignatureError` 或 `EncryptedPdfError`：
- 记录警告日志
- 不生成归档件
- 如果原文件已有文本，直接使用原文本内容（用户仍可搜索）

### 5.4 Tesseract：Ghostscript PDF/A 渲染失败提示

位置：[_handle_subprocess_output_error()](file:///d:/fz/0601/solo-dogfeeding/code/115-paperless-ngx/src/paperless/parsers/tesseract.py#L478-L492)

当 `SubprocessOutputError` 消息包含 "Ghostscript PDF/A rendering" 时：
- 日志提示用户可配置 `PAPERLESS_OCR_USER_ARGS: {"continue_on_soft_render_error": true}`
- 让 OCRmyPDF 忽略 Ghostscript 的软渲染错误，继续产出 PDF

### 5.5 Tika：Tika 服务器 500 错误回退

位置：[TikaDocumentParser.parse()](file:///d:/fz/0601/solo-dogfeeding/code/115-paperless-ngx/src/paperless/parsers/tika.py#L247-L261)

当 Tika 服务器返回 HTTP 500（Apache TIKA-4110 缺陷：某些文件以 multipart/form 上传会失败）时：
- 自动以二进制 buffer 方式（`from_buffer`）重新提交请求
- 这是特定已知缺陷的专门 workaround，不处理其他 HTTP 错误

### 5.6 Tika/Mail：PDF/A-1 请求降级为 A-2b

位置：
- Tika: [tika.py:436-439](file:///d:/fz/0601/solo-dogfeeding/code/115-paperless-ngx/src/paperless/parsers/tika.py#L436-L439)
- Mail: [mail.py:459-463](file:///d:/fz/0601/solo-dogfeeding/code/115-paperless-ngx/src/paperless/parsers/mail.py#L459-L463)

当 `OCR_OUTPUT_TYPE=pdfa-1` 时，Gotenberg 不支持该格式：
- 记录 WARNING 级别日志
- 静默降级输出 PDF/A-2b
- 整个流程不报错，用户只有查看日志才能发现不一致

### 5.7 通用：缩略图生成回退链

位置：[documents/parsers.py:130-198](file:///d:/fz/0601/solo-dogfeeding/code/115-paperless-ngx/src/documents/parsers.py#L130-L198)

```
make_thumbnail_from_pdf()
  │
  ├─ ImageMagick convert 转换首页为 WebP
  │    │
  │    └─ 失败 → make_thumbnail_from_pdf_gs_fallback()
  │              │
  │              ├─ Ghostscript 输出 PNG（gs 不支持 WebP）
  │              │    │
  │              │    └─ 失败 → 拷贝内置默认缩略图 document.webp
  │              │
  │              └─ ImageMagick 将 PNG 转为 WebP
  │
  └─ 返回 WebP 缩略图
```

三级回退：ImageMagick → Ghostscript → 内置默认图，保证 100% 能拿到缩略图。

### 5.8 通用：文件名过长回退

位置：[consumer.py:672-714](file:///d:/fz/0601/solo-dogfeeding/code/115-paperless-ngx/src/documents/consumer.py#L672-L714)

当模板渲染出的文件名超过 `Document.MAX_STORED_FILENAME_LENGTH` 时：
- 记录警告日志
- 回退到默认命名格式（`{document_id:07}.pdf`），不使用用户自定义模板

---

## 6. 归档文件的存储与命名

### 6.1 文件名生成

代码位置：[documents/file_handling.py](file:///d:/fz/0601/solo-dogfeeding/code/115-paperless-ngx/src/documents/file_handling.py)

`generate_unique_filename(archive_filename=True)` 的策略：

1. **优先与原件同名**：尝试 `{原件文件名去掉扩展名}.pdf`，保留模板中的目录结构
2. **冲突追加序号**：若同名文件已存在，追加 `_01`、`_02`...
3. **模板渲染**：使用 `StoragePath.path` 或全局 `FILENAME_FORMAT` 模板（Jinja2 语法）
4. **默认兜底**：无模板时使用 `{document_id:07}.pdf`，如 `0000001.pdf`

版本文档额外追加 `_v{version_index}` 后缀。

### 6.2 归档件落盘流程

位置：[consumer.py:698-724](file:///d:/fz/0601/solo-dogfeeding/code/115-paperless-ngx/src/documents/consumer.py#L698-L724)

```
1. parser.get_archive_path() 返回临时目录中的 archive.pdf
2. generate_unique_filename() 生成目标路径
3. create_source_path_directory() 创建父目录（parents=True）
4. 拷贝文件到 ARCHIVE_DIR 下
5. compute_checksum() 计算归档文件 SHA256
6. 数据库保存 archive_filename 和 archive_checksum
```

所有文件操作在 `transaction.atomic()` + `FileLock(MEDIA_LOCK)` 保护下执行，文件移动失败则数据库回滚。

---

## 7. 触发归档的入口点

### 7.1 文档消费时自动生成

主消费流程 [ConsumerPlugin.run()](file:///d:/fz/0601/solo-dogfeeding/code/115-paperless-ngx/src/documents/consumer.py#L408-L784) 在文档首次入库时生成归档件。

### 7.2 `document_archiver` 管理命令

位置：[documents/management/commands/document_archiver.py](file:///d:/fz/0601/solo-dogfeeding/code/115-paperless-ngx/src/documents/management/commands/document_archiver.py)

```bash
# 为所有没有归档件的文档生成归档
python manage.py document_archiver

# 重新生成所有文档的归档件（覆盖已有）
python manage.py document_archiver -f / --overwrite

# 只处理某个文档
python manage.py document_archiver -d <document_id>
```

命令内部调用 [update_document_content_maybe_archive_file()](file:///d:/fz/0601/solo-dogfeeding/code/115-paperless-ngx/src/documents/tasks.py#L279-L396)，支持多进程并行处理。

### 7.3 Parser 能力声明

每个 Parser 通过两个属性声明归档能力：

| 属性 | 含义 | 示例 |
|------|------|------|
| `can_produce_archive` | 能否产出 PDF 归档件 | `RasterisedDocumentParser=True`, `TextDocumentParser=False` |
| `requires_pdf_rendition` | 是否必须产 PDF 才能在浏览器显示 | `TikaDocumentParser=True`（DOCX 等）|

定义在 [ParserProtocol](file:///d:/fz/0601/solo-dogfeeding/code/115-paperless-ngx/src/paperless/parsers/__init__.py#L209-L228)。

---

## 8. 完整行为流程图

```
文档进入消费流程
       │
       ▼
  确定 MIME 类型 → 选择对应 Parser
       │
       ▼
  should_produce_archive()?
       │
       ├─ No → 只提取文本 + 缩略图，结束
       │
       └─ Yes（或 requires_pdf_rendition=True 强制）
            │
            ▼
  ┌─────────────────────────────────────────────────────────────┐
  │                    按 Parser 分流                            │
  ├─────────────────────────────────────────────────────────────┤
  │                                                             │
  │  Tesseract（PDF / 图片）                                    │
  │    ├─ OCR_MODE=off + 图片 → img2pdf + pikepdf                │
  │    │                       └─ 失败: 退回普通 PDF              │
  │    ├─ OCR_MODE=off + PDF  → Ghostscript 直接转 PDF/A          │
  │    ├─ auto + 已有文本 + 不需归档 → 仅 pdftotext              │
  │    └─ OCRmyPDF.ocr() 主流程                                   │
  │         ├─ 加密/签名 → 不生成归档，只用原文文本                │
  │         ├─ 成功 → 文本 + 归档件                                │
  │         └─ 失败 → Force OCR 回退                              │
  │              ├─ 成功 → 文本 + 归档件                           │
  │              └─ 失败 → ParseError 终止                        │
  │                                                             │
  │  Tika（Office: DOCX/XLSX/PPTX/ODT/RTF…）                    │
  │    ├─ Tika 文本提取                                           │
  │    │    └─ 500 错误: from_buffer 重试                          │
  │    └─ Gotenberg LibreOffice → PDF                             │
  │         └─ PDF/A-1 请求 → 静默降级为 A2b + 警告日志            │
  │                                                             │
  │  Mail（.eml / message/rfc822）                               │
  │    ├─ imap_tools 解析 EML                                     │
  │    ├─ 邮件正文 HTML 模板 → Gotenberg Chromium HTML→PDF         │
  │    ├─ 邮件 HTML 正文（如有）→ Gotenberg Chromium HTML→PDF      │
  │    ├─ Gotenberg Merge 按布局合并（TEXT_HTML/HTML_ONLY…）       │
  │    └─ 每步均应用 PDF/A 格式化（A-1 降级为 A2b）                 │
  │                                                             │
  └─────────────────────────────────────────────────────────────┘
       │
       ▼
  落盘到 ARCHIVE_DIR + 写入 DB（archive_filename + archive_checksum）
```

---

## 9. 关键文件索引

| 文件 | 职责 |
|------|------|
| [documents/consumer.py](file:///d:/fz/0601/solo-dogfeeding/code/115-paperless-ngx/src/documents/consumer.py) | `should_produce_archive()` 决策、消费主流程、文件落盘 |
| [paperless/parsers/tesseract.py](file:///d:/fz/0601/solo-dogfeeding/code/115-paperless-ngx/src/paperless/parsers/tesseract.py) | `RasterisedDocumentParser`：OCR、PDF/A 转换、Force OCR 回退 |
| [paperless/parsers/tika.py](file:///d:/fz/0601/solo-dogfeeding/code/115-paperless-ngx/src/paperless/parsers/tika.py) | `TikaDocumentParser`：Office 文档，Tika 文本提取 + Gotenberg LibreOffice 转 PDF |
| [paperless/parsers/mail.py](file:///d:/fz/0601/solo-dogfeeding/code/115-paperless-ngx/src/paperless/parsers/mail.py) | `MailDocumentParser`：EML 邮件，Gotenberg Chromium HTML→PDF + Merge |
| [paperless/parsers/utils.py](file:///d:/fz/0601/solo-dogfeeding/code/115-paperless-ngx/src/paperless/parsers/utils.py) | `is_tagged_pdf()`、`extract_pdf_text()`、PDF 文本阈值 |
| [paperless/config.py](file:///d:/fz/0601/solo-dogfeeding/code/115-paperless-ngx/src/paperless/config.py) | `OutputTypeConfig`、`OcrConfig`：各解析器读取配置的入口 |
| [documents/tasks.py](file:///d:/fz/0601/solo-dogfeeding/code/115-paperless-ngx/src/documents/tasks.py) | `update_document_content_maybe_archive_file()` 异步任务 |
| [documents/file_handling.py](file:///d:/fz/0601/solo-dogfeeding/code/115-paperless-ngx/src/documents/file_handling.py) | 归档文件名生成、唯一性保证 |
| [documents/parsers.py](file:///d:/fz/0601/solo-dogfeeding/code/115-paperless-ngx/src/documents/parsers.py) | 缩略图三级回退、`ParseError` 定义 |
| [paperless/models.py](file:///d:/fz/0601/solo-dogfeeding/code/115-paperless-ngx/src/paperless/models.py) | `ArchiveFileGenerationChoices`、`OutputTypeChoices`、`ModeChoices` |
| [paperless/parsers/__init__.py](file:///d:/fz/0601/solo-dogfeeding/code/115-paperless-ngx/src/paperless/parsers/__init__.py) | `ParserProtocol`：解析器能力契约 |
| [documents/management/commands/document_archiver.py](file:///d:/fz/0601/solo-dogfeeding/code/115-paperless-ngx/src/documents/management/commands/document_archiver.py) | `document_archiver` 命令行入口 |
