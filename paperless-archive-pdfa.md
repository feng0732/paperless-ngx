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

---

## 4. 失败回退（Fallback）机制

系统在多处设计了降级回退策略，确保「尽量产出可用结果」而不是直接失败。

### 4.1 PDF/A 元数据打标失败回退

位置：[_convert_image_to_pdfa()](file:///d:/fz/0601/solo-dogfeeding/code/115-paperless-ngx/src/paperless/parsers/tesseract.py#L429-L433)

```python
try:
    # pikepdf 打 PDF/A 标签 ...
except Exception as e:
    logger.warning("PDF/A metadata stamping failed; falling back to plain PDF.")
    pdfa_path.write_bytes(plain_pdf_path.read_bytes())
```

**行为**：pikepdf 注入 PDF/A 元数据失败时，直接使用 img2pdf 输出的普通 PDF。**结果仍然是可用的 PDF，只是不是 PDF/A 标准格式。**

### 4.2 OCR 主流程失败 → Force OCR 回退

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

### 4.3 加密/签名 PDF 回退

位置：[tesseract.py:610-616](file:///d:/fz/0601/solo-dogfeeding/code/115-paperless-ngx/src/paperless/parsers/tesseract.py#L610-L616)

捕获 `DigitalSignatureError` 或 `EncryptedPdfError`：
- 记录警告日志
- 不生成归档件
- 如果原文件已有文本，直接使用原文本内容（用户仍可搜索）

### 4.4 Ghostscript PDF/A 渲染失败提示

位置：[_handle_subprocess_output_error()](file:///d:/fz/0601/solo-dogfeeding/code/115-paperless-ngx/src/paperless/parsers/tesseract.py#L478-L492)

当 `SubprocessOutputError` 消息包含 "Ghostscript PDF/A rendering" 时：
- 日志提示用户可配置 `PAPERLESS_OCR_USER_ARGS: {"continue_on_soft_render_error": true}`
- 让 OCRmyPDF 忽略 Ghostscript 的软渲染错误，继续产出 PDF

### 4.5 缩略图生成回退链

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

### 4.6 文件名过长回退

位置：[consumer.py:672-714](file:///d:/fz/0601/solo-dogfeeding/code/115-paperless-ngx/src/documents/consumer.py#L672-L714)

当模板渲染出的文件名超过 `Document.MAX_STORED_FILENAME_LENGTH` 时：
- 记录警告日志
- 回退到默认命名格式（`{document_id:07}.pdf`），不使用用户自定义模板

---

## 5. 归档文件的存储与命名

### 5.1 文件名生成

代码位置：[documents/file_handling.py](file:///d:/fz/0601/solo-dogfeeding/code/115-paperless-ngx/src/documents/file_handling.py)

`generate_unique_filename(archive_filename=True)` 的策略：

1. **优先与原件同名**：尝试 `{原件文件名去掉扩展名}.pdf`，保留模板中的目录结构
2. **冲突追加序号**：若同名文件已存在，追加 `_01`、`_02`...
3. **模板渲染**：使用 `StoragePath.path` 或全局 `FILENAME_FORMAT` 模板（Jinja2 语法）
4. **默认兜底**：无模板时使用 `{document_id:07}.pdf`，如 `0000001.pdf`

版本文档额外追加 `_v{version_index}` 后缀。

### 5.2 归档件落盘流程

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

## 6. 触发归档的入口点

### 6.1 文档消费时自动生成

主消费流程 [ConsumerPlugin.run()](file:///d:/fz/0601/solo-dogfeeding/code/115-paperless-ngx/src/documents/consumer.py#L408-L784) 在文档首次入库时生成归档件。

### 6.2 `document_archiver` 管理命令

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

### 6.3 Parser 能力声明

每个 Parser 通过两个属性声明归档能力：

| 属性 | 含义 | 示例 |
|------|------|------|
| `can_produce_archive` | 能否产出 PDF 归档件 | `RasterisedDocumentParser=True`, `TextDocumentParser=False` |
| `requires_pdf_rendition` | 是否必须产 PDF 才能在浏览器显示 | `TikaDocumentParser=True`（DOCX 等）|

定义在 [ParserProtocol](file:///d:/fz/0601/solo-dogfeeding/code/115-paperless-ngx/src/paperless/parsers/__init__.py#L209-L228)。

---

## 7. 完整行为流程图

```
文档进入消费流程
       │
       ▼
  should_produce_archive()?
       │
       ├─ No → 只提取文本 + 缩略图，结束
       │
       └─ Yes
            │
            ▼
    RasterisedDocumentParser.parse(produce_archive=True)
            │
            ├─ OCR_MODE=off?
            │    ├─ image → img2pdf + pikepdf(_convert_image_to_pdfa)
            │    │           └─ 失败: 退回普通 PDF
            │    └─ pdf   → Ghostscript(_convert_pdf_to_pdfa)
            │
            ├─ OCR_MODE=auto + 已有文本 + 不需归档?
            │    └─ 直接返回 pdftotext 文本
            │
            └─ OCRmyPDF.ocr() 主流程
                 │
                 ├─ 加密/签名 PDF → 不生成归档，只用原文文本
                 │
                 ├─ 成功 → 提取文本 + 归档件
                 │
                 └─ 失败 (NoTextFound/InputFileError/PriorOcrFound)
                      │
                      └─ Force OCR 回退
                           ├─ 成功 → 提取文本 + 归档件
                           └─ 失败 → 抛出 ParseError 终止
```

---

## 8. 关键文件索引

| 文件 | 职责 |
|------|------|
| [documents/consumer.py](file:///d:/fz/0601/solo-dogfeeding/code/115-paperless-ngx/src/documents/consumer.py) | `should_produce_archive()` 决策、消费主流程、文件落盘 |
| [paperless/parsers/tesseract.py](file:///d:/fz/0601/solo-dogfeeding/code/115-paperless-ngx/src/paperless/parsers/tesseract.py) | `RasterisedDocumentParser`：OCR、PDF/A 转换、Force OCR 回退 |
| [paperless/parsers/utils.py](file:///d:/fz/0601/solo-dogfeeding/code/115-paperless-ngx/src/paperless/parsers/utils.py) | `is_tagged_pdf()`、`extract_pdf_text()`、PDF 文本阈值 |
| [documents/tasks.py](file:///d:/fz/0601/solo-dogfeeding/code/115-paperless-ngx/src/documents/tasks.py) | `update_document_content_maybe_archive_file()` 异步任务 |
| [documents/file_handling.py](file:///d:/fz/0601/solo-dogfeeding/code/115-paperless-ngx/src/documents/file_handling.py) | 归档文件名生成、唯一性保证 |
| [documents/parsers.py](file:///d:/fz/0601/solo-dogfeeding/code/115-paperless-ngx/src/documents/parsers.py) | 缩略图三级回退、`ParseError` 定义 |
| [paperless/models.py](file:///d:/fz/0601/solo-dogfeeding/code/115-paperless-ngx/src/paperless/models.py) | `ArchiveFileGenerationChoices`、`OutputTypeChoices`、`ModeChoices` |
| [paperless/parsers/__init__.py](file:///d:/fz/0601/solo-dogfeeding/code/115-paperless-ngx/src/paperless/parsers/__init__.py) | `ParserProtocol`：解析器能力契约 |
| [documents/management/commands/document_archiver.py](file:///d:/fz/0601/solo-dogfeeding/code/115-paperless-ngx/src/documents/management/commands/document_archiver.py) | `document_archiver` 命令行入口 |
