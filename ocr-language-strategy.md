# OCR 语言与解析策略代码分析

> 本文档所有路径均为仓库相对路径，即相对于项目根目录 `68-paperless-ngx/`。

---

## 1. 整体架构概览

Paperless-ngx 的文档解析流程分为以下几个核心层次：

```
文档消费入口 (documents/consumer.py)
    ↓
MIME 类型检测
    ↓
解析器注册表 (paperless/parsers/registry.py) → 根据 MIME 类型 + 评分选择最佳解析器
    ↓
具体解析器执行
    ├── TextDocumentParser       (纯文本 txt/csv)
    ├── TikaDocumentParser       (Office 文档 docx/xlsx/pptx 等)
    ├── MailDocumentParser       (邮件 .eml)
    ├── RemoteDocumentParser     (云端 OCR，如 Azure AI)
    └── RasterisedDocumentParser (Tesseract OCR，核心)
    ↓
文本提取 / 归档 PDF 生成 / 缩略图
    ↓
日期解析、分类、存储
```

---

## 2. OCR 语言配置全链路（后台配置的生效边界）

> ⚠️ **重要前置说明**：本节描述的「数据库优先，环境变量兜底」机制，**仅适用于 Tesseract OCR 和 dateparser**。
> **NLTK 分词语言和搜索（Tantivy）语言完全不读取数据库后台配置**，只在 Django 启动时从环境变量推导一次，详见第 3 节。

### 2.1 Tesseract/dateparser 共用的配置来源（数据库 → 环境变量）

Tesseract OCR 和 dateparser 共用一套「数据库优先，环境变量兜底」的语言加载机制。

**第一层：环境变量默认值**（`src/paperless/settings/__init__.py:884`）：

```python
OCR_LANGUAGE = os.getenv("PAPERLESS_OCR_LANGUAGE", "eng")
```

默认值为 `"eng"`（英语）。

**第二层：数据库存储的后台配置**（`src/paperless/models.py:114-119`）：

```python
class ApplicationConfiguration(AbstractSingletonModel):
    language = models.CharField(
        verbose_name=_("Do OCR using these languages"),
        null=True,
        blank=True,
        max_length=32,
    )
```

`ApplicationConfiguration` 是单例模型（全局仅有一条记录），允许管理员在后台页面设置语言，`null=True, blank=True` 表示可以为空（为空时使用环境变量兜底）。

**第三层：运行时合并**（`src/paperless/config.py:47-75`）：

```python
@dataclasses.dataclass
class OcrConfig(OutputTypeConfig):
    language: str = dataclasses.field(init=False)
    mode: ModeChoices = dataclasses.field(init=False)

    def __post_init__(self) -> None:
        super().__post_init__()
        app_config = self._get_config_instance()
        # 数据库优先，为空则回退到环境变量
        self.language = app_config.language or settings.OCR_LANGUAGE
        self.mode = app_config.mode or ModeChoices(settings.OCR_MODE)
```

**Tesseract/dateparser 共用的配置加载优先级**：
1. 读取数据库 `ApplicationConfiguration` 单例的 `language` 字段（后台配置）
2. 若为空（`None` 或空字符串），回退到 `settings.OCR_LANGUAGE`（即环境变量 `PAPERLESS_OCR_LANGUAGE`）
3. 环境变量也未设置时，使用硬编码默认值 `"eng"`

**配置合法性检查（生效边界：仅校验环境变量）**（`src/paperless/checks.py:357-386`）：

系统启动时 Django 会运行 `check_default_language_available()` 诊断检查，但该函数**只读取 `settings.OCR_LANGUAGE`（即环境变量 `PAPERLESS_OCR_LANGUAGE`），完全不读取数据库中的 `ApplicationConfiguration.language`**。

```python
@register()
def check_default_language_available(app_configs, **kwargs):
    errs = []
    if not settings.OCR_LANGUAGE:   # ← 只读 settings.OCR_LANGUAGE（环境变量）
        errs.append(Warning("No OCR language has been specified with PAPERLESS_OCR_LANGUAGE..."))
        return errs

    if shutil.which("tesseract") is not None:
        installed_langs = get_tesseract_langs()
        # ← 同样只 split settings.OCR_LANGUAGE，不碰数据库
        specified_langs = [x.strip() for x in settings.OCR_LANGUAGE.split("+")]
        for lang in specified_langs:
            if lang not in installed_langs:
                errs.append(Error(f"The selected ocr language {lang} is not installed..."))
    return errs
```

**校验链路的生效边界问题**：
- 若环境变量 `PAPERLESS_OCR_LANGUAGE=eng`（合法，已安装），但管理员在后台把 language 改成 `chi_sim`（未安装语言包）
- 启动检查 **不会报错**（因为只校验环境变量的 `eng`）
- 问题直到 Tesseract 实际解析文档时才会暴露（OCRmyPDF 会因为找不到 `chi_sim` 训练数据而失败，抛出 `ParseError`）
- dateparser、NLTK、搜索语言各自有独立的取值链路，也均不受启动检查保护（详见第 3 节）

### 2.2 语言代码格式

采用 **Tesseract ISO 639-2 三字母代码**，支持：
- 单语言：`eng`、`deu`、`chi_sim`
- 多语言（`+` 连接）：`eng+fra+deu`
- 带书写脚本变体（`_` 分隔）：`aze_Cyrl`（阿塞拜疆语-西里尔字母）

### 2.3 语言配置在 OCR 参数中的传递

Tesseract 解析器构造 OCRmyPDF 参数时，语言被直接传入（`src/paperless/parsers/tesseract.py:266-286`）：

```python
def construct_ocrmypdf_parameters(self, ...):
    ocrmypdf_args = {
        "language": self.settings.language,  # ← 语言配置在此传递给 OCRmyPDF
        "output_type": self.settings.output_type,
        "use_threads": True,
        # ...
    }
```

---

## 3. 语言配置的下游推导（生效边界详解）

OCR 语言被推导为 **dateparser 语言**、**NLTK 分词语言**和 **搜索（Tantivy）语言**三个下游组件。四个组件对「后台数据库配置」的读取能力完全不同，这是最容易混淆的地方：

| 组件 | 能否读取后台 `ApplicationConfiguration.language` | 能否被后台配置覆盖 |
|------|----------------------------------------------|------------------|
| Tesseract OCR | ✅ 能，每次解析时实时读取 | ✅ 完全生效 |
| dateparser | ✅ 能，每次创建解析器时实时读取 | ✅ 生效（除非显式设了 `PAPERLESS_DATE_PARSER_LANGUAGES`） |
| NLTK | ❌ 不能，只读取 `settings.OCR_LANGUAGE`（环境变量） | ❌ 完全不生效 |
| 搜索 (Tantivy) | ❌ 不能，只读环境变量 | ❌ 完全不生效 |

### 3.1 dateparser 语言处理（与 Tesseract 的异同）

dateparser 和 Tesseract **都能读取后台数据库配置**，但 dateparser 多了一层自己的独立环境变量覆盖。

**两者相同点**：
- 都会通过 `OcrConfig()` 读取数据库 `ApplicationConfiguration.language`，为空时回退到环境变量 `PAPERLESS_OCR_LANGUAGE`
- 后台修改语言后，下一次解析/日期解析就会生效（无需重启服务）

**两者不同点**：
- Tesseract 没有独立的语言环境变量，只能走 OcrConfig 的「数据库 → OCR 环境变量」链路
- dateparser 额外支持 `PAPERLESS_DATE_PARSER_LANGUAGES` 独立配置，设置后**完全绕过** OCR 语言配置（包括后台数据库）

#### dateparser 的两层 fallback

**第一层（最高优先级）：独立环境变量 `PAPERLESS_DATE_PARSER_LANGUAGES`**（`src/paperless/settings/__init__.py:961-967`）：

```python
DATE_PARSER_LANGUAGES = (
    parse_dateparser_languages(
        os.getenv("PAPERLESS_DATE_PARSER_LANGUAGES"),
    )
    if os.getenv("PAPERLESS_DATE_PARSER_LANGUAGES")
    else None
)
```

若设置了 `PAPERLESS_DATE_PARSER_LANGUAGES`，则通过 `parse_dateparser_languages()` 解析：

```python
# src/paperless/settings/custom.py:331-343
def parse_dateparser_languages(languages: str | None) -> list[str]:
    language_list = languages.split("+") if languages else []
    # 中文特殊处理：zh-Hant / zh-Hans 已知在 dateparser 中有 bug
    for index, language in enumerate(language_list):
        if language.startswith("zh-") and "zh" not in language_list:
            logger.warning(
                f"Chinese locale detected: {language}. dateparser might fail to parse"
                f' some dates with this locale, so Chinese ("zh") will be used as a fallback.',
            )
            language_list.append("zh")
    return list(LocaleDataLoader().get_locale_map(locales=language_list))
```

**第二层（兜底）：通过 `OcrConfig()` 从 OCR 语言动态推导**（`src/documents/plugins/date_parsing/__init__.py:64-93`）：

在 `get_date_parser()` 工厂函数中，每次创建日期解析器时都会**实时**执行：

```python
ocr_config = OcrConfig()   # ← 每次都会 new OcrConfig()，实时读取数据库
languages = settings.DATE_PARSER_LANGUAGES or ocr_to_dateparser_languages(
    ocr_config.language,   # ← 这里的 language 已合并了后台数据库配置
)
```

当 `settings.DATE_PARSER_LANGUAGES` 为 `None`（即未显式设置独立环境变量）时，调用 `ocr_to_dateparser_languages()` 从 OCR 语言动态推导（`src/paperless/utils.py:118-169`）：

```python
def ocr_to_dateparser_languages(ocr_languages: str) -> list[str]:
    loader = LocaleDataLoader()
    result = []
    for ocr_language in ocr_languages.split("+"):
        # 如 "aze_Cyrl" → 拆分为 ocr_lang_part="aze", script=["Cyrl"]
        ocr_lang_part, *script = ocr_language.split("_")
        ocr_script_part = script[0] if script else None

        # 1. 通过 OCR_TO_DATEPARSER_LANGUAGES 映射表转码（ISO 639-2 → locale）
        #    如 "aze" → "az", "eng" → "en", "chi" → "zh"
        language_part = OCR_TO_DATEPARSER_LANGUAGES.get(ocr_lang_part)
        if language_part is None:
            continue  # 映射表中不存在则跳过

        loader.get_locale_map(locales=[language_part])  # 验证基础语言

        # 2. 若有脚本变体，尝试组合 locale（如 "az" + "Cyrl" → "az-Cyrl"）
        if ocr_script_part:
            dateparser_language = f"{language_part}-{ocr_script_part.title()}"
            try:
                loader.get_locale_map(locales=[dateparser_language])
            except Exception:
                # 变体不被支持时回退到基础语言
                dateparser_language = language_part
        else:
            dateparser_language = language_part

        if dateparser_language not in result:
            result.append(dateparser_language)
    return result
```

`OCR_TO_DATEPARSER_LANGUAGES` 映射表定义在 `src/paperless/utils.py:7-115`，包含约 90 种语言的 ISO 639-2 → dateparser locale 映射。

**推导失败 fallback**：
- 某语言在映射表中不存在 → 跳过，记录 debug 日志
- 脚本变体（如 Cyrl）dateparser 不支持 → 回退到基础语言，记录 info 日志
- 整个推导过程抛异常 → 返回空列表 `[]`，记录 warning 日志
- 最终结果为空 → 记录 info 日志，dateparser 使用自身默认的多语言模式

### 3.2 NLTK 语言（完全不读取后台数据库配置）

NLTK 分词语言的取值有两个绝对限制：
1. **没有独立的环境变量**，完全从 OCR 语言推导
2. **只读取 `settings.OCR_LANGUAGE`（即环境变量 `PAPERLESS_OCR_LANGUAGE`），完全不读取数据库 `ApplicationConfiguration.language`**

推导逻辑在 `_get_nltk_language_setting()`（`src/paperless/settings/__init__.py:1026-1058`）：

```python
def _get_nltk_language_setting(ocr_lang: str) -> str | None:
    # 只取第一个主语言（"+" 之前的部分）
    ocr_lang = ocr_lang.split("+", maxsplit=1)[0]

    iso_code_to_nltk = {
        "dan": "danish",
        "nld": "dutch",
        "eng": "english",
        "fin": "finnish",
        "fra": "french",
        "deu": "german",
        "ita": "italian",
        "nor": "norwegian",
        "por": "portuguese",
        "rus": "russian",
        "spa": "spanish",
        "swe": "swedish",
    }
    return iso_code_to_nltk.get(ocr_lang)  # 不在表中返回 None
```

在模块加载时调用（`src/paperless/settings/__init__.py:1106`）：

```python
NLTK_LANGUAGE: str | None = _get_nltk_language_setting(OCR_LANGUAGE)
```

**关键限制（生效边界）**：
- 仅支持 13 种欧洲语言的 Snowball 词干还原器 / Punkt 分词器 / 停用词的交集
- 只考虑多语言配置中的 **第一个** 主语言（例如 `eng+fra` 只取 `eng`）
- 未命中映射表时 `NLTK_LANGUAGE = None`，表示不启用语言相关的 NLP 处理
- **推导发生在 Django settings 加载阶段（进程启动时执行一次），此后值固定不变**
- **代码证据**：`NLTK_LANGUAGE: str | None = _get_nltk_language_setting(OCR_LANGUAGE)`
  直接传入的是 `settings.OCR_LANGUAGE`（来自环境变量），**完全绕过了 `OcrConfig`，因此永远不会读取数据库后台配置**
- 管理员在后台修改 language 字段后，NLTK 语言不会发生任何变化，必须修改环境变量 `PAPERLESS_OCR_LANGUAGE` 并重启 Django 服务才能生效

### 3.3 搜索语言（Tantivy stemmer，完全不读取后台数据库配置）

搜索语言支持显式配置，但和 NLTK 一样，**完全不读取数据库后台配置**，只从环境变量读取。

两层取值来源（均不涉及数据库）：
1. **第一优先级**：环境变量 `PAPERLESS_SEARCH_LANGUAGE` 显式设置
2. **第二优先级**：从环境变量 `PAPERLESS_OCR_LANGUAGE` 的第一个主语言推导

推导逻辑在 `_get_search_language_setting()`（`src/paperless/settings/__init__.py:1061-1101`）：

```python
def _get_search_language_setting(ocr_lang: str) -> str | None:
    # 第一层：显式设置 PAPERLESS_SEARCH_LANGUAGE
    explicit = os.environ.get("PAPERLESS_SEARCH_LANGUAGE")
    if explicit is not None:
        from documents.search._tokenizer import SUPPORTED_LANGUAGES
        return get_choice_from_env("PAPERLESS_SEARCH_LANGUAGE", SUPPORTED_LANGUAGES)

    # 第二层：从 OCR 主语言推导（ISO 639-2/T → ISO 639-1 两字母）
    primary = ocr_lang.split("+", maxsplit=1)[0].lower()
    _ocr_to_search: dict[str, str] = {
        "ara": "ar", "dan": "da", "nld": "nl", "eng": "en",
        "fin": "fi", "fra": "fr", "deu": "de", "ell": "el",
        "hun": "hu", "ita": "it", "nor": "no", "por": "pt",
        "ron": "ro", "rus": "ru", "spa": "es", "swe": "sv",
        "tam": "ta", "tur": "tr",
    }
    return _ocr_to_search.get(primary)
```

在模块加载时调用（`src/paperless/settings/__init__.py:1108`）：

```python
SEARCH_LANGUAGE: str | None = _get_search_language_setting(OCR_LANGUAGE)
```

**关键限制（生效边界）**：
- 支持 19 种 Tantivy 内置词干还原器语言
- 只考虑 OCR 多语言配置中的 **第一个** 主语言
- 显式设置 `PAPERLESS_SEARCH_LANGUAGE` 时，会校验是否在 Tantivy `SUPPORTED_LANGUAGES` 中
- 未命中返回 `None`，表示搜索不分词干
- **推导发生在 Django settings 加载阶段（进程启动时执行一次），此后值固定不变**
- **代码证据**：`SEARCH_LANGUAGE: str | None = _get_search_language_setting(OCR_LANGUAGE)`
  直接传入的是 `settings.OCR_LANGUAGE`（来自环境变量），`PAPERLESS_SEARCH_LANGUAGE` 也是直接从 `os.environ.get()` 读取，**完全绕过了 `OcrConfig`，因此永远不会读取数据库后台配置**
- 管理员在后台修改 language 字段后，搜索词干语言不会发生任何变化
  - 如果使用默认推导：必须修改 `PAPERLESS_OCR_LANGUAGE` 环境变量并重启服务，且需要重建搜索索引
  - 如果使用显式配置：必须修改 `PAPERLESS_SEARCH_LANGUAGE` 环境变量并重启服务，且需要重建搜索索引

### 3.4 语言配置链路汇总（取值边界 + 校验边界全表）

| 组件 | 第一优先级 | 第二优先级 | 推导时机 | 是否读取后台配置 | 启动时是否被校验 | 非法配置暴露时机 |
|------|-----------|-----------|---------|----------------|----------------|----------------|
| **Tesseract OCR** | 数据库 `ApplicationConfiguration.language`（每次实时读） | `PAPERLESS_OCR_LANGUAGE`（默认 `eng`） | 运行时每次解析 new `OcrConfig()` | ✅ 实时生效 | ❌ 启动只校验环境变量 | 文档解析时（OCRmyPDF 报错） |
| **dateparser** | `PAPERLESS_DATE_PARSER_LANGUAGES`（设了就完全绕过 OCR 配置） | 数据库 `ApplicationConfiguration.language` + `PAPERLESS_OCR_LANGUAGE`（通过 `OcrConfig` 实时读） | 运行时每次创建解析器 new `OcrConfig()` | ✅ 实时生效 | ❌ 无启动校验 | 日期解析时（locale 映射失败或解析异常） |
| **NLTK** | —（无独立配置） | `PAPERLESS_OCR_LANGUAGE` **第一个** 主语言（只读环境变量，不读数据库） | Django 启动加载 settings 时（仅一次，永久固化） | ❌ 完全不读 | ❌ 无启动校验 | 文档分类时（语言不在支持列表则为 None，无报错但不启用分词） |
| **搜索 (Tantivy)** | `PAPERLESS_SEARCH_LANGUAGE`（只读环境变量） | `PAPERLESS_OCR_LANGUAGE` **第一个** 主语言（只读环境变量，不读数据库） | Django 启动加载 settings 时（仅一次，永久固化） | ❌ 完全不读 | ❌ 仅显式设 `PAPERLESS_SEARCH_LANGUAGE` 时校验 | 搜索索引构建时（语言不在列表则为 None，不分词干） |

> **生效边界总结（取值 + 校验）**：
> 1. **Tesseract 和 dateparser**：通过每次 new `OcrConfig()` 实时读取数据库，管理员在后台改 language **立即生效**，无需重启。但**启动时不会校验数据库里的语言是否合法**，填了未安装的语言包要等到实际解析文档时才会报错。
> 2. **NLTK 和搜索语言**：在 Django 启动时直接从环境变量 `os.environ` 读取，**代码路径完全绕过 `OcrConfig`，因此数据库里的 `ApplicationConfiguration.language` 对这两个组件没有任何影响**。启动时也不会校验从 OCR 语言推导出来的值是否合法。
> 3. **启动校验的盲区**：`check_default_language_available()` 只校验 `settings.OCR_LANGUAGE`（环境变量）。对于四个组件，只要语言最终取值和环境变量不一致（无论是通过后台配置还是通过独立变量），就完全脱离了启动校验的保护。
> 4. 若仅在后台修改 OCR 语言：Tesseract OCR 和日期解析会使用新语言，但文档分类（NLTK）和搜索词干还原仍使用启动时的旧语言，直到修改环境变量并重启 Django 服务（搜索还需重建索引）。

---

## 4. 解析器选择策略

### 4.1 解析器注册表

所有解析器由 `ParserRegistry` 单例统一管理（`src/paperless/parsers/registry.py:71-99`）：

```python
def get_parser_registry() -> ParserRegistry:
    with _lock:
        if _registry is None:
            r = ParserRegistry()
            r.register_defaults()   # 注册内置解析器
            _registry = r
        if not _discovery_complete:
            _registry.discover()    # 发现第三方插件解析器（entrypoint）
            _discovery_complete = True
    return _registry
```

### 4.2 内置解析器注册与评分

注册顺序（`src/paperless/parsers/registry.py:192-210`）：

```python
def register_defaults(self) -> None:
    self.register_builtin(TextDocumentParser)      # 评分: 10
    self.register_builtin(RemoteDocumentParser)    # 评分: 20（若配置有效）
    self.register_builtin(TikaDocumentParser)      # 评分: 10（若 TIKA_ENABLED）
    self.register_builtin(MailDocumentParser)      # 评分: 10
    self.register_builtin(RasterisedDocumentParser) # 评分: 10
```

各解析器详情：

| 解析器类 | 支持的 MIME 类型 | 评分 | 启用条件 | 文件 |
|---------|---------------|-----|---------|------|
| `TextDocumentParser` | `text/plain`, `text/csv`, `application/csv` | 10 | 始终启用 | `src/paperless/parsers/text.py` |
| `TikaDocumentParser` | `application/msword`, `application/vnd.openxmlformats-officedocument.*` 等 Office 格式 | 10 | `settings.TIKA_ENABLED=True` | `src/paperless/parsers/tika.py` |
| `MailDocumentParser` | `message/rfc822` (.eml) | 10 | 始终启用 | `src/paperless/parsers/mail.py` |
| `RasterisedDocumentParser` | `application/pdf`, `image/jpeg`, `image/png`, `image/tiff`, `image/gif`, `image/bmp`, `image/webp`, `image/heic` | 10 | 始终启用 | `src/paperless/parsers/tesseract.py` |
| `RemoteDocumentParser` | `application/pdf`, `image/png`, `image/jpeg`, `image/tiff`, `image/bmp`, `image/gif`, `image/webp` | 20 | 配置了有效远程引擎（如 Azure AI） | `src/paperless/parsers/remote.py` |

**Remote 解析器的评分逻辑**（`src/paperless/parsers/remote.py:112-149`）：

```python
@classmethod
def score(cls, mime_type, filename, path=None):
    config = RemoteEngineConfig(
        engine=settings.REMOTE_OCR_ENGINE,
        api_key=settings.REMOTE_OCR_API_KEY,
        endpoint=settings.REMOTE_OCR_ENDPOINT,
    )
    if not config.engine_is_valid():
        return None   # 未配置时完全退出竞争
    if mime_type not in _SUPPORTED_MIME_TYPES:
        return None
    return 20  # 高于 Tesseract 的 10，优先使用云端 OCR
```

### 4.3 解析器选择算法

核心逻辑在 `get_parser_for_file()`（`src/paperless/parsers/registry.py:332-393`）：

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

**选择规则**（按优先级）：
1. **MIME 类型过滤**：必须出现在 `supported_mime_types()` 返回的字典中
2. **评分过滤**：`score()` 返回 `None` 表示该解析器主动放弃（如 Tika 未启用、远程引擎未配置）
3. **高分优先**：分数最高者获胜（Remote 的 20 分 > 其他内置的 10 分）
4. **外部优先**：同分时，第三方插件解析器（entrypoint 加载）优先于内置
5. **注册顺序兜底**：同类型、同分数时先注册的优先

### 4.4 消费流程中的解析器调用

在 `src/documents/consumer.py:425-524` 中：

```python
# 1. MIME 类型检测
mime_type = magic.from_file(self.working_copy, mime=True)

# 1.5 PDF 修复 fallback：文件名是 .pdf 但 MIME 类型异常时，用 qpdf 尝试修复
if (
    Path(self.filename).suffix.lower() == ".pdf"
    and mime_type in settings.CONSUMER_PDF_RECOVERABLE_MIME_TYPES
):
    try:
        run_subprocess(["qpdf", "--replace-input", self.working_copy], logger=self.log)
        mime_type = magic.from_file(self.working_copy, mime=True)  # 重新检测
    except Exception as e:
        self.log.error(f"Error attempting to clean PDF: {e}")

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
```

### 4.5 是否生成归档 PDF 的决策

`should_produce_archive()`（`src/documents/consumer.py:124-189`）决策表：

| 条件 | 结果 |
|------|------|
| `parser.requires_pdf_rendition=True`（Office/EML 等浏览器无法原生显示的格式） | ✅ 必须生成 |
| `parser.can_produce_archive=False`（如 TextDocumentParser） | ❌ 不生成 |
| `ARCHIVE_FILE_GENERATION=always` | ✅ 总是生成 |
| `ARCHIVE_FILE_GENERATION=never` | ❌ 从不生成 |
| `ARCHIVE_FILE_GENERATION=auto` + 图片 MIME 类型 | ✅ 生成 |
| `ARCHIVE_FILE_GENERATION=auto` + PDF 带结构标签（born-digital） | ❌ 不生成 |
| `ARCHIVE_FILE_GENERATION=auto` + PDF 文本 > 50 字符（born-digital） | ❌ 不生成 |
| `ARCHIVE_FILE_GENERATION=auto` + PDF 文本 ≤ 50 字符（扫描件） | ✅ 生成 |

---

## 5. 文本提取流程（Tesseract 解析器）

### 5.1 OCR 模式（ModeChoices）

在 `src/paperless/models.py:33-42` 定义了四种模式：

| 模式 | 含义 | 对应 OCRmyPDF 参数 |
|-----|------|------------------|
| `auto` | 自动检测：已有文本则跳过，否则做 OCR | 无特殊参数（默认行为） |
| `force` | 强制对所有页面做 OCR，覆盖已有文本层 | `--force-ocr` |
| `redo` | 重做 OCR，保留已有文本层 | `--redo-ocr` |
| `off` | 完全不调用 OCR 引擎 | `--skip-text`（仅需要归档时） |

同样采用数据库优先 + 环境变量兜底：`OcrConfig.mode = app_config.mode or ModeChoices(settings.OCR_MODE)`。

### 5.2 解析主流程

`RasterisedDocumentParser.parse()`（`src/paperless/parsers/tesseract.py:494-658`）是整个 OCR 解析的核心：

```
开始
  │
  ├─ 前置检测：PDF 是否已有文本
  │   ├─ is_tagged_pdf() → 检查 PDF 是否带 /MarkInfo 结构标签
  │   └─ extract_pdf_text() + len() > PDF_TEXT_MIN_LENGTH(50) → 已有文本层
  │
  ├─ 分支 A: OCR_MODE=off（完全不做 OCR）
  │   ├─ 不需要归档 → text = 原文件文本, return
  │   ├─ 图片输入 → _convert_image_to_pdfa()（img2pdf + pikepdf，不调 Tesseract）
  │   └─ PDF 输入 → _convert_pdf_to_pdfa()（Ghostscript，不调 Tesseract）
  │
  ├─ 分支 B: OCR_MODE=auto + 已有文本 + 不需要归档
  │   └─ 完全跳过 OCRmyPDF, text = 原文件文本
  │
  └─ 分支 C: 其他情况 → 运行 OCRmyPDF
      ├─ auto + 已有文本 + 需要归档 → skip_text=True（仅做 PDF/A 转换，不 OCR）
      └─ 其余情况 → 完整 OCR 流程
          ├─ construct_ocrmypdf_parameters() 构造参数（含 language）
          ├─ ocrmypdf.ocr(**args) 调用
          ├─ extract_text(sidecar_file, archive_pdf) 提取文本
          │   ├─ 优先读取 sidecar.txt（OCRmyPDF 输出的纯文本）
          │   └─ sidecar 不完整时 → pdftotext 提取输出 PDF 文本
          └─ post_process_text() 文本后处理
```

### 5.3 文本提取优先级

`extract_text()`（`src/paperless/parsers/tesseract.py:236-264`）按以下优先级获取文本：

```python
def extract_text(self, sidecar_file, pdf_file):
    # 1. 优先使用 OCRmyPDF 生成的 sidecar 文件
    #    REDO 模式下不使用 sidecar（因为 REDO 只做 OCR，sidecar 可能不完整）
    if sidecar_file is not None and sidecar_file.is_file() and self.settings.mode != ModeChoices.REDO:
        text = read_file_handle_unicode_errors(sidecar_file)
        # sidecar 中含 "[OCR skipped on page" 标记 → 某些页跳过了 OCR，文本不完整
        if "[OCR skipped on page" not in text:
            return post_process_text(text)

    # 2. sidecar 不完整或 REDO 模式 → 从生成的归档 PDF 中用 pdftotext 提取
    return post_process_text(extract_pdf_text(Path(pdf_file)))
```

### 5.4 PDF 文本提取实现

底层通过 `pdftotext` 命令行工具（`src/paperless/parsers/utils.py:67-110`）：

```python
def extract_pdf_text(path, log=None):
    run_subprocess([
        "pdftotext", "-q", "-layout", "-enc", "UTF-8",
        str(path), str(out_path),
    ])
    text = read_file_handle_unicode_errors(out_path)
    return text or None
```

### 5.5 文本后处理

`post_process_text()`（`src/paperless/parsers/tesseract.py:661-672`）：

```python
def post_process_text(text):
    collapsed_spaces = re.sub(r"([^\S\r\n]+)", " ", text)        # 压缩非换行空白
    no_leading_whitespace = re.sub(r"([\n\r]+)([^\S\n\r]+)", "\\1", collapsed_spaces)
    no_trailing_whitespace = re.sub(r"([^\S\n\r]+)$", "", no_leading_whitespace)
    return no_trailing_whitespace.strip().replace("\0", " ")      # 去除 NUL 字符（Postgres 兼容）
```

---

## 6. 失败 Fallback 策略

### 6.1 Tesseract 解析器内部的分层 Fallback

`parse()` 方法（`src/paperless/parsers/tesseract.py:599-658`）中的异常处理链：

```
ocrmypdf.ocr() 调用
  │
  ├─ DigitalSignatureError / EncryptedPdfError（加密/签名 PDF）
  │   └─ 若原文件有文本 (original_has_text) → 使用原文件文本，不做 OCR
  │
  ├─ SubprocessOutputError（通常是 Ghostscript PDF/A 渲染失败）
  │   └─ 记录 warning，提示设置 PAPERLESS_OCR_USER_ARGS
  │      ={"continue_on_soft_render_error": true}，然后抛出 ParseError
  │
  ├─ NoTextFoundException / InputFileError / PriorOcrFoundError
  │   └─ Fallback 重试：safe_fallback=True → force_ocr=True
  │       ├─ 强制对所有页面做 OCR，忽略已有文本层
  │       ├─ 成功 → 使用强制 OCR 的文本和归档
  │       └─ 失败 → 抛出 ParseError
  │
  └─ 其他异常 → 抛出 ParseError

最终兜底（所有异常处理完毕后）：
  ├─ 若 self.text 为空但 original_has_text → 回退使用原文件文本
  └─ 若 self.text 仍为空 → self.text = ""，记录 warning 日志
```

`safe_fallback` 触发强制 OCR 的关键代码（`src/paperless/parsers/tesseract.py:293-302`）：

```python
if safe_fallback or self.settings.mode == ModeChoices.FORCE:
    ocrmypdf_args["force_ocr"] = True   # 强制 OCR，忽略已有文本层
```

### 6.2 缩略图生成的三级 Fallback

（`src/documents/parsers.py:174-198` 和 `src/documents/parsers.py:130-171`）：

```
make_thumbnail_from_pdf()
  │
  ├─ 第一层：ImageMagick convert（PDF 首页 → WebP）
  │   └─ 失败 → make_thumbnail_from_pdf_gs_fallback()
  │       │
  │       ├─ 第二层：Ghostscript 提取首页为 PNG，再用 convert 转 WebP
  │       │
  │       └─ 失败 → 第三层：copy_file_with_basic_stats(get_default_thumbnail())
  │           └─ 使用内置默认图 src/documents/resources/document.webp
  │
  └─ 成功 → 返回生成的 WebP
```

### 6.3 Tika 解析器的 Fallback

针对 TIKA-4110 问题（某些文件 multipart 表单上传返回 500）的 Workaround（`src/paperless/parsers/tika.py:246-261`）：

```python
try:
    parsed = self._tika_client.tika.as_text.from_file(document_path, mime_type)
except httpx.HTTPStatusError as err:
    if err.response.status_code == httpx.codes.INTERNAL_SERVER_ERROR:
        # 回退：改用 buffer 方式上传文件内容
        parsed = self._tika_client.tika.as_text.from_buffer(
            document_path.read_bytes(), mime_type,
        )
```

### 6.4 文本读取的 Unicode Fallback

所有文本读取都经过 `read_file_handle_unicode_errors()`（`src/paperless/parsers/utils.py:113-138`）：

```python
def read_file_handle_unicode_errors(filepath, log=None):
    try:
        return filepath.read_text(encoding="utf-8")
    except UnicodeDecodeError as e:
        _log.warning("Unicode error during text reading, continuing: %s", e)
        return filepath.read_bytes().decode("utf-8", errors="replace")
        # errors="replace" → 无效 UTF-8 字节替换为 U+FFFD 替换字符，不抛出异常
```

### 6.5 PDF 消费前的 MIME 类型修复 Fallback

在 `src/documents/consumer.py:431-461`：
- 当文件扩展名为 `.pdf` 但 `libmagic` 检测出异常 MIME 类型（如 `application/octet-stream`）
- 尝试用 `qpdf --replace-input` 修复 PDF 结构
- 修复后重新检测 MIME 类型
- 修复失败记录错误日志，继续使用原始 MIME 类型（可能后续失败）

### 6.6 跨解析器 Fallback 的缺失

当前架构中 **不存在跨解析器的自动 fallback**：
- 每个 MIME 类型只选择一个「最佳解析器」（评分最高者）
- 若该解析器抛出 `ParseError`，消费流程直接失败终止，不会尝试其他解析器
- 示例：PDF 文档若配置了 RemoteDocumentParser（Azure AI 评分 20），Azure 调用失败时不会自动回退到 RasterisedDocumentParser（Tesseract 评分 10）

如需实现跨解析器 fallback，需要修改 `ConsumerPlugin.run()`（`src/documents/consumer.py:408`）中的异常处理逻辑，在解析失败时手动尝试次优解析器。

---

## 7. 关键文件索引

| 文件路径 | 职责 |
|---------|------|
| `src/paperless/settings/__init__.py` | 环境变量配置入口：`OCR_LANGUAGE`、`OCR_MODE`、`NLTK_LANGUAGE`、`SEARCH_LANGUAGE`、`DATE_PARSER_LANGUAGES` |
| `src/paperless/settings/custom.py` | `parse_dateparser_languages()` — 显式 dateparser 语言解析 |
| `src/paperless/config.py` | `OcrConfig` 运行时配置数据类，实现数据库+环境变量双层加载 |
| `src/paperless/models.py` | `ApplicationConfiguration` 后台配置模型、`ModeChoices`/`OutputTypeChoices` 等枚举 |
| `src/paperless/checks.py` | `check_default_language_available()` — 启动时 OCR 语言诊断检查 |
| `src/paperless/utils.py` | `OCR_TO_DATEPARSER_LANGUAGES` 映射表 + `ocr_to_dateparser_languages()` 推导函数 |
| `src/paperless/parsers/__init__.py` | `ParserProtocol` 解析器接口协议定义 |
| `src/paperless/parsers/registry.py` | `ParserRegistry` 解析器注册表，评分选择算法 |
| `src/paperless/parsers/tesseract.py` | `RasterisedDocumentParser` — Tesseract OCR 核心，含 force_ocr fallback |
| `src/paperless/parsers/utils.py` | `extract_pdf_text()`、`is_tagged_pdf()`、`read_file_handle_unicode_errors()` 等共享工具 |
| `src/paperless/parsers/text.py` | `TextDocumentParser` — 纯文本解析器 |
| `src/paperless/parsers/tika.py` | `TikaDocumentParser` — Office 文档解析器，含 TIKA-4110 fallback |
| `src/paperless/parsers/mail.py` | `MailDocumentParser` — 邮件解析器 |
| `src/paperless/parsers/remote.py` | `RemoteDocumentParser` — 云端 OCR（Azure AI）解析器 |
| `src/documents/plugins/date_parsing/__init__.py` | `get_date_parser()` — dateparser 语言两层 fallback 决策点 |
| `src/documents/consumer.py` | 消费入口：MIME 检测、解析器查找、`should_produce_archive()`、qpdf PDF 修复 |
| `src/documents/parsers.py` | 旧版 `DocumentParser` 基类、缩略图生成三级 fallback 链 |
