# OCR 语言与解析策略代码分析

## 1. 整体架构概览

Paperless-ngx 的文档解析流程分为以下几个核心层次：

```
文档消费入口 (consumer.py)
    ↓
MIME 类型检测
    ↓
解析器注册表 (registry.py) → 根据 MIME 类型 + 评分选择最佳解析器
    ↓
具体解析器执行
    ├── TextDocumentParser   (纯文本 txt/csv)
    ├── TikaDocumentParser   (Office 文档 docx/xlsx/pptx 等)
    ├── MailDocumentParser   (邮件 .eml)
    ├── RemoteDocumentParser (云端 OCR，如 Azure AI)
    └── RasterisedDocumentParser (Tesseract OCR，核心)
    ↓
文本提取 / 归档 PDF 生成 / 缩略图
    ↓
日期解析、分类、存储
```

---

## 2. OCR 语言配置

### 2.1 配置来源与优先级

OCR 语言配置采用 **数据库优先，环境变量兜底** 的双层机制。

**环境变量定义**（[settings/__init__.py:884](file:///d:/fz/0601/solo-dogfeeding/code/68-paperless-ngx/src/paperless/settings/__init__.py#L884-L884)）：

```python
OCR_LANGUAGE = os.getenv("PAPERLESS_OCR_LANGUAGE", "eng")
```

默认值为 `"eng"`（英语）。

**运行时加载**（[config.py:47-75](file:///d:/fz/0601/solo-dogfeeding/code/68-paperless-ngx/src/paperless/config.py#L47-L75)）：

```python
@dataclasses.dataclass
class OcrConfig(OutputTypeConfig):
    language: str = dataclasses.field(init=False)
    mode: ModeChoices = dataclasses.field(init=False)
    # ... 其他字段

    def __post_init__(self) -> None:
        super().__post_init__()
        app_config = self._get_config_instance()
        self.language = app_config.language or settings.OCR_LANGUAGE
        self.mode = app_config.mode or ModeChoices(settings.OCR_MODE)
```

配置加载顺序：
1. 先读取数据库 `ApplicationConfiguration` 单例中的 `language` 字段
2. 若为空，则回退到 `settings.OCR_LANGUAGE`（来自环境变量 `PAPERLESS_OCR_LANGUAGE`）

### 2.2 语言代码格式

采用 **Tesseract ISO 639-2 三字母代码**，支持：
- 单语言：`eng`、`deu`、`chi_sim`
- 多语言（`+` 连接）：`eng+fra+deu`
- 带书写脚本变体：`aze_Cyrl`（阿塞拜疆语-西里尔字母）

### 2.3 语言配置在 OCR 参数中的传递

在 Tesseract 解析器构造 OCRmyPDF 参数时，语言被直接传入（[tesseract.py:266-286](file:///d:/fz/0601/solo-dogfeeding/code/68-paperless-ngx/src/paperless/parsers/tesseract.py#L266-L286)）：

```python
def construct_ocrmypdf_parameters(self, ...):
    ocrmypdf_args = {
        "language": self.settings.language,  # ← 语言配置在此传递
        "output_type": self.settings.output_type,
        "use_threads": True,
        # ...
    }
```

### 2.4 语言配置的下游影响

OCR 语言不仅用于 Tesseract，还会被推导为其他组件的语言设置：

| 下游组件 | 推导函数 | 位置 |
|---------|---------|------|
| 日期解析 (dateparser) | `ocr_to_dateparser_languages()` | [utils.py:118-169](file:///d:/fz/0601/solo-dogfeeding/code/68-paperless-ngx/src/paperless/utils.py#L118-L169) |
| NLTK 分词 | `_get_nltk_language_setting()` | [settings/__init__.py:1067](file:///d:/fz/0601/solo-dogfeeding/code/68-paperless-ngx/src/paperless/settings/__init__.py#L1067-L1067) |
| 搜索语言 | `_get_search_language_setting()` | [settings/__init__.py:1108](file:///d:/fz/0601/solo-dogfeeding/code/68-paperless-ngx/src/paperless/settings/__init__.py#L1108-L1108) |

**Tesseract → dateparser 语言转换**示例（[utils.py:118-169](file:///d:/fz/0601/solo-dogfeeding/code/68-paperless-ngx/src/paperless/utils.py#L118-L169)）：

```python
def ocr_to_dateparser_languages(ocr_languages: str) -> list[str]:
    # 输入: "eng+fra+aze_Cyrl"
    for ocr_language in ocr_languages.split("+"):
        ocr_lang_part, *script = ocr_language.split("_")
        # "aze" → "az", "Cyrl" → 尝试 "az-Cyrl"，失败则回退 "az"
        language_part = OCR_TO_DATEPARSER_LANGUAGES.get(ocr_lang_part)
        # ... 验证 dateparser 是否支持该 locale
```

---

## 3. 解析器选择策略

### 3.1 解析器注册表

所有解析器由 `ParserRegistry` 单例统一管理（[registry.py:71-99](file:///d:/fz/0601/solo-dogfeeding/code/68-paperless-ngx/src/paperless/parsers/registry.py#L71-L99)）：

```python
def get_parser_registry() -> ParserRegistry:
    with _lock:
        if _registry is None:
            r = ParserRegistry()
            r.register_defaults()   # 注册内置解析器
            _registry = r
        if not _discovery_complete:
            _registry.discover()    # 发现第三方插件解析器
            _discovery_complete = True
    return _registry
```

### 3.2 内置解析器注册

注册顺序（[registry.py:192-210](file:///d:/fz/0601/solo-dogfeeding/code/68-paperless-ngx/src/paperless/parsers/registry.py#L192-L210)）：

```python
def register_defaults(self) -> None:
    self.register_builtin(TextDocumentParser)      # 分数: 10
    self.register_builtin(RemoteDocumentParser)    # 分数: 20 (若配置)
    self.register_builtin(TikaDocumentParser)      # 分数: 10 (若启用)
    self.register_builtin(MailDocumentParser)      # 分数: 10
    self.register_builtin(RasterisedDocumentParser) # 分数: 10
```

### 3.3 各解析器的 MIME 类型与评分

| 解析器 | MIME 类型 | 评分 | 启用条件 |
|-------|----------|------|---------|
| TextDocumentParser | text/plain, text/csv, application/csv | 10 | 始终启用 |
| TikaDocumentParser | doc/docx/xls/xlsx/ppt/pptx/odt/ods/odp/rtf 等 | 10 | `settings.TIKA_ENABLED=True` |
| MailDocumentParser | message/rfc822 (.eml) | 10 | 始终启用 |
| RasterisedDocumentParser | application/pdf, image/jpeg/png/tiff/gif/bmp/webp/heic | 10 | 始终启用 |
| RemoteDocumentParser | application/pdf, image/jpeg/png/tiff/gif/bmp/webp | 20 | 配置了有效的远程引擎 (如 Azure) |

**评分逻辑示例**（[remote.py:112-149](file:///d:/fz/0601/solo-dogfeeding/code/68-paperless-ngx/src/paperless/parsers/remote.py#L112-L149)）：

```python
@classmethod
def score(cls, mime_type, filename, path=None):
    config = RemoteEngineConfig(...)
    if not config.engine_is_valid():
        return None   # 未配置时完全退出竞争
    if mime_type not in _SUPPORTED_MIME_TYPES:
        return None
    return 20  # 高于 Tesseract 的 10，优先使用云端 OCR
```

### 3.4 解析器选择算法

核心逻辑在 `get_parser_for_file()`（[registry.py:332-393](file:///d:/fz/0601/solo-dogfeeding/code/68-paperless-ngx/src/paperless/parsers/registry.py#L332-L393)）：

```python
def get_parser_for_file(self, mime_type, filename, path=None):
    best_score = None
    best_parser = None

    # 外部解析器在前，内置在后 → 同分时外部优先
    for parser_class in (*self._external, *self._builtins):
        if mime_type not in parser_class.supported_mime_types():
            continue
        score = parser_class.score(mime_type, filename, path)
        if score is None:
            continue
        if best_score is None or score > best_score:
            best_score = score
            best_parser = parser_class

    return best_parser
```

**选择规则总结**：
1. **MIME 类型过滤**：必须出现在 `supported_mime_types()` 中
2. **评分过滤**：`score()` 返回 `None` 表示该解析器主动放弃（如未启用 Tika）
3. **高分优先**：分数最高者获胜
4. **外部优先**：同分时，第三方插件解析器优先于内置
5. **注册顺序兜底**：同类型解析器分数相同时，先注册的优先

### 3.5 消费流程中的解析器调用

在 [consumer.py:425-524](file:///d:/fz/0601/solo-dogfeeding/code/68-paperless-ngx/src/documents/consumer.py#L425-L524) 中：

```python
# 1. MIME 类型检测
mime_type = magic.from_file(self.working_copy, mime=True)

# 2. 获取解析器类
parser_class = get_parser_registry().get_parser_for_file(
    mime_type, self.filename, self.working_copy
)

# 3. 实例化并执行解析
with parser_class() as document_parser:
    document_parser.configure(ParserContext(mailrule_id=...))
    produce_archive = should_produce_archive(document_parser, mime_type, ...)
    document_parser.parse(self.working_copy, mime_type, produce_archive=produce_archive)
    text = document_parser.get_text()
    # ...
```

**是否生成归档 PDF** 的决策在 `should_produce_archive()`（[consumer.py:124-189](file:///d:/fz/0601/solo-dogfeeding/code/68-paperless-ngx/src/documents/consumer.py#L124-L189)）：

| 条件 | 结果 |
|------|------|
| `parser.requires_pdf_rendition=True`（如 Office/EML 浏览器无法显示） | ✅ 必须生成 |
| `parser.can_produce_archive=False`（如纯文本） | ❌ 不生成 |
| `ARCHIVE_FILE_GENERATION=always` | ✅ 总是生成 |
| `ARCHIVE_FILE_GENERATION=never` | ❌ 从不生成 |
| `ARCHIVE_FILE_GENERATION=auto` + 图片文档 | ✅ 生成 |
| `ARCHIVE_FILE_GENERATION=auto` + PDF 带结构标签（born-digital） | ❌ 不生成 |
| `ARCHIVE_FILE_GENERATION=auto` + PDF 文本 > 50 字符（born-digital） | ❌ 不生成 |
| `ARCHIVE_FILE_GENERATION=auto` + PDF 文本 ≤ 50 字符（扫描件） | ✅ 生成 |

---

## 4. 文本提取流程（以 Tesseract 解析器为核心）

### 4.1 OCR 模式（ModeChoices）

在 [models.py:33-42](file:///d:/fz/0601/solo-dogfeeding/code/68-paperless-ngx/src/paperless/models.py#L33-L42) 定义了四种 OCR 模式：

| 模式 | 含义 | 对应 OCRmyPDF 参数 |
|-----|------|------------------|
| `auto` | 自动检测：已有文本则跳过，否则做 OCR | 无特殊参数（默认行为） |
| `force` | 强制对所有页面做 OCR，覆盖已有文本 | `--force-ocr` |
| `redo` | 重做 OCR，保留已有文本层 | `--redo-ocr` |
| `off` | 完全不调用 OCR 引擎 | `--skip-text`（仅在需要归档时） |

### 4.2 解析主流程

`RasterisedDocumentParser.parse()` 是整个 OCR 解析的核心（[tesseract.py:494-658](file:///d:/fz/0601/solo-dogfeeding/code/68-paperless-ngx/src/paperless/parsers/tesseract.py#L494-L658)），流程如下：

```
开始
  │
  ├─ 检测 PDF 是否已有文本
  │   ├─ is_tagged_pdf() → 检查 PDF 结构标签
  │   └─ extract_pdf_text() + len() > 50 → 已有文本层
  │
  ├─ 分支 1: OCR_MODE=off
  │   ├─ 不需要归档 → text = 原文件文本, return
  │   ├─ 图片输入 → img2pdf 转 PDF/A, 无 OCR
  │   └─ PDF 输入 → Ghostscript 转 PDF/A, 无 OCR
  │
  ├─ 分支 2: OCR_MODE=auto + 已有文本 + 不需要归档
  │   └─ 完全跳过 OCRmyPDF, text = 原文件文本
  │
  └─ 分支 3: 其他情况（运行 OCRmyPDF）
      ├─ auto + 已有文本 + 需要归档 → skip_text=True（仅转 PDF/A）
      └─ 其余情况 → 完整 OCR
          │
          ├─ 构造参数 construct_ocrmypdf_parameters()
          ├─ 调用 ocrmypdf.ocr(**args)
          ├─ 提取文本 extract_text(sidecar_file, archive_pdf)
          │   ├─ 优先读取 sidecar.txt（OCRmyPDF 输出）
          │   └─ sidecar 不完整时 → pdftotext 提取 PDF 文本
          └─ 文本后处理 post_process_text()
```

### 4.3 文本提取的优先级

`extract_text()` 方法（[tesseract.py:236-264](file:///d:/fz/0601/solo-dogfeeding/code/68-paperless-ngx/src/paperless/parsers/tesseract.py#L236-L264)）按以下优先级获取文本：

```python
def extract_text(self, sidecar_file, pdf_file):
    # 1. 优先使用 OCRmyPDF 生成的 sidecar 文件
    if sidecar_file.is_file() and self.settings.mode != ModeChoices.REDO:
        text = read_file_handle_unicode_errors(sidecar_file)
        if "[OCR skipped on page" not in text:
            return post_process_text(text)  # sidecar 完整

    # 2. sidecar 不完整或 REDO 模式 → 从生成的 PDF 中提取
    return post_process_text(extract_pdf_text(pdf_file))
```

**sidecar 文件特殊判断**：
- 当 sidecar 中包含 `"[OCR skipped on page"` 标记时，说明某些页面跳过了 OCR，sidecar 文本不完整
- 此时回退到对整个输出 PDF 运行 `pdftotext`

### 4.4 PDF 文本提取实现

底层通过 `pdftotext` 命令行工具提取（[utils.py:67-110](file:///d:/fz/0601/solo-dogfeeding/code/68-paperless-ngx/src/paperless/parsers/utils.py#L67-L110)）：

```python
def extract_pdf_text(path, log=None):
    run_subprocess([
        "pdftotext", "-q", "-layout", "-enc", "UTF-8",
        str(path), str(out_path),
    ])
    text = read_file_handle_unicode_errors(out_path)
    return text or None
```

### 4.5 文本后处理

`post_process_text()`（[tesseract.py:661-672](file:///d:/fz/0601/solo-dogfeeding/code/68-paperless-ngx/src/paperless/parsers/tesseract.py#L661-L672)）：

```python
def post_process_text(text):
    collapsed_spaces = re.sub(r"([^\S\r\n]+)", " ", text)       # 压缩多余空格
    no_leading_whitespace = re.sub(r"([\n\r]+)([^\S\n\r]+)", "\\1", collapsed_spaces)
    no_trailing_whitespace = re.sub(r"([^\S\n\r]+)$", "", no_leading_whitespace)
    return no_trailing_whitespace.strip().replace("\0", " ")      # 去除 NUL 字符（Postgres 兼容）
```

---

## 5. 失败 Fallback 机制

### 5.1 Tesseract 解析器内部的分层 Fallback

在 `parse()` 方法中有多层异常处理（[tesseract.py:599-658](file:///d:/fz/0601/solo-dogfeeding/code/68-paperless-ngx/src/paperless/parsers/tesseract.py#L599-L658)）：

```
ocrmypdf.ocr() 调用
  │
  ├─ DigitalSignatureError / EncryptedPdfError
  │   └─ 若原文件有文本 → 使用原文件文本
  │
  ├─ SubprocessOutputError (Ghostscript 渲染失败)
  │   └─ 提示用户设置 PAPERLESS_OCR_USER_ARGS={"continue_on_soft_render_error": true}
  │
  ├─ NoTextFoundException / InputFileError / PriorOcrFoundError
  │   └─ Fallback: safe_fallback=True → force_ocr=True 强制重试
  │       ├─ 成功 → 使用强制 OCR 的结果
  │       └─ 失败 → 抛出 ParseError
  │
  └─ 其他异常 → 抛出 ParseError

最后兜底（所有路径执行完毕后）：
  ├─ 若 text 为空但 original_has_text → 使用原文件文本
  └─ 若 text 仍为空 → text = ""（警告日志）
```

**强制 OCR Fallback 的关键代码**：

```python
except (NoTextFoundException, InputFileError, PriorOcrFoundError) as e:
    # 构造 fallback 参数：safe_fallback=True → force_ocr=True
    args = self.construct_ocrmypdf_parameters(
        ..., safe_fallback=True,
    )
    # 再次调用 OCRmyPDF，强制对所有页面做 OCR
    ocrmypdf.ocr(**args)
```

`safe_fallback` 参数在 `construct_ocrmypdf_parameters()` 中的处理（[tesseract.py:293-302](file:///d:/fz/0601/solo-dogfeeding/code/68-paperless-ngx/src/paperless/parsers/tesseract.py#L293-L302)）：

```python
if safe_fallback or self.settings.mode == ModeChoices.FORCE:
    ocrmypdf_args["force_ocr"] = True   # 强制 OCR，忽略已有文本层
```

### 5.2 缩略图生成的 Fallback 链

缩略图生成有三层 Fallback（[parsers.py:174-198](file:///d:/fz/0601/solo-dogfeeding/code/68-paperless-ngx/src/documents/parsers.py#L174-L198) 和 [parsers.py:130-171](file:///d:/fz/0601/solo-dogfeeding/code/68-paperless-ngx/src/documents/parsers.py#L130-L171)）：

```
make_thumbnail_from_pdf()
  │
  ├─ 尝试 ImageMagick convert
  │   └─ 失败 → make_thumbnail_from_pdf_gs_fallback()
  │       │
  │       ├─ 尝试 Ghostscript (gs) 提取第一页
  │       │   └─ 再用 convert 转 WebP
  │       │
  │       └─ 失败 → copy_file_with_basic_stats(get_default_thumbnail())
  │           └─ 使用内置的默认 document.webp
  │
  └─ 成功 → 返回生成的 WebP
```

### 5.3 Tika 解析器的 Fallback

Tika 解析器处理某些文件时 multipart 表单上传会 500 错误，有针对 TIKA-4110 的 workaround（[tika.py:246-261](file:///d:/fz/0601/solo-dogfeeding/code/68-paperless-ngx/src/paperless/parsers/tika.py#L246-L261)）：

```python
try:
    parsed = self._tika_client.tika.as_text.from_file(document_path, mime_type)
except httpx.HTTPStatusError as err:
    # Workaround: TIKA-4110，改用 buffer 上传
    if err.response.status_code == httpx.codes.INTERNAL_SERVER_ERROR:
        parsed = self._tika_client.tika.as_text.from_buffer(
            document_path.read_bytes(), mime_type,
        )
```

### 5.4 文本读取的 Unicode Fallback

所有文本读取都经过 `read_file_handle_unicode_errors()`（[utils.py:113-138](file:///d:/fz/0601/solo-dogfeeding/code/68-paperless-ngx/src/paperless/parsers/utils.py#L113-L138)）：

```python
def read_file_handle_unicode_errors(filepath, log=None):
    try:
        return filepath.read_text(encoding="utf-8")
    except UnicodeDecodeError as e:
        _log.warning("Unicode error during text reading, continuing: %s", e)
        return filepath.read_bytes().decode("utf-8", errors="replace")
        # errors="replace" → 无效字节替换为 U+FFFD �，不抛出异常
```

### 5.5 跨解析器 Fallback 的缺失

当前架构中 **不存在跨解析器的自动 fallback**：
- 每个 MIME 类型只选择一个「最佳解析器」
- 若该解析器抛出 `ParseError`，消费流程直接失败，不会尝试其他解析器
- 例如：PDF 文档若选了 RemoteDocumentParser（Azure），Azure 失败时不会自动回退到 Tesseract

如需实现跨解析器 fallback，需要修改 `ConsumerPlugin.run()` 中的异常处理逻辑。

---

## 6. 关键文件索引

| 文件 | 职责 |
|------|------|
| [paperless/settings/__init__.py](file:///d:/fz/0601/solo-dogfeeding/code/68-paperless-ngx/src/paperless/settings/__init__.py) | 环境变量配置入口，OCR_LANGUAGE、OCR_MODE 等默认值 |
| [paperless/config.py](file:///d:/fz/0601/solo-dogfeeding/code/68-paperless-ngx/src/paperless/config.py) | OcrConfig 等运行时配置数据类，数据库+环境变量双层加载 |
| [paperless/models.py](file:///d:/fz/0601/solo-dogfeeding/code/68-paperless-ngx/src/paperless/models.py) | ModeChoices、OutputTypeChoices、CleanChoices 等枚举 |
| [paperless/parsers/registry.py](file:///d:/fz/0601/solo-dogfeeding/code/68-paperless-ngx/src/paperless/parsers/registry.py) | ParserRegistry，解析器注册与评分选择 |
| [paperless/parsers/__init__.py](file:///d:/fz/0601/solo-dogfeeding/code/68-paperless-ngx/src/paperless/parsers/__init__.py) | ParserProtocol 协议定义 |
| [paperless/parsers/tesseract.py](file:///d:/fz/0601/solo-dogfeeding/code/68-paperless-ngx/src/paperless/parsers/tesseract.py) | RasterisedDocumentParser，Tesseract OCR 核心逻辑 |
| [paperless/parsers/utils.py](file:///d:/fz/0601/solo-dogfeeding/code/68-paperless-ngx/src/paperless/parsers/utils.py) | extract_pdf_text、is_tagged_pdf 等共享工具 |
| [paperless/utils.py](file:///d:/fz/0601/solo-dogfeeding/code/68-paperless-ngx/src/paperless/utils.py) | ocr_to_dateparser_languages 语言转换 |
| [paperless/parsers/text.py](file:///d:/fz/0601/solo-dogfeeding/code/68-paperless-ngx/src/paperless/parsers/text.py) | TextDocumentParser 纯文本解析器 |
| [paperless/parsers/tika.py](file:///d:/fz/0601/solo-dogfeeding/code/68-paperless-ngx/src/paperless/parsers/tika.py) | TikaDocumentParser Office 文档解析器 |
| [paperless/parsers/mail.py](file:///d:/fz/0601/solo-dogfeeding/code/68-paperless-ngx/src/paperless/parsers/mail.py) | MailDocumentParser 邮件解析器 |
| [paperless/parsers/remote.py](file:///d:/fz/0601/solo-dogfeeding/code/68-paperless-ngx/src/paperless/parsers/remote.py) | RemoteDocumentParser 云端 OCR 解析器 |
| [documents/consumer.py](file:///d:/fz/0601/solo-dogfeeding/code/68-paperless-ngx/src/documents/consumer.py) | 消费入口，should_produce_archive 决策 |
| [documents/parsers.py](file:///d:/fz/0601/solo-dogfeeding/code/68-paperless-ngx/src/documents/parsers.py) | 旧版 DocumentParser 基类、缩略图生成 Fallback |
