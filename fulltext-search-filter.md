# Paperless-ngx 全文索引与搜索过滤数据流

本文档梳理 Paperless-ngx 中 OCR 文本 → 索引字段 → 过滤条件 → 排序结果的完整数据流，所有描述均以代码事实为准。

---

## 整体架构概览

```
文档消费阶段                          搜索查询阶段
──────────────                        ──────────────
原始文件
  │
  ▼
DocumentParser.parse() (OCR提取)      HTTP Request
  │  设置 self.text                      │
  ▼                                     ▼
Document.objects.create(content=text)  DocumentViewSet.list()
  │                                     │
  ▼                                     ▼
document_consumption_finished Signal   1. parse_search_params()
  │  (add_to_index receiver)             2. TantivyBackend.search_ids()
  ▼                                     3. ORM filter_queryset()
TantivyBackend.add_or_update()          4. intersect_and_order()
  │                                     5. highlight_hits()
  ▼                                     6. TantivyRelevanceList + 分页
Tantivy Index (磁盘)                    ▼
                                       JSON Response
```

核心搜索引擎：**Tantivy**（Rust 实现的全文搜索引擎，类 Lucene）。

相关代码目录：
- 搜索核心：`src/documents/search/`
- 过滤层：`src/documents/filters.py`
- API 视图层：`src/documents/views.py`
- 数据模型：`src/documents/models.py`
- 消费流程：`src/documents/consumer.py`
- 信号注册：`src/documents/apps.py`、`src/documents/signals/__init__.py`

---

## 一、OCR 文本提取 → 数据库存储

### 1.1 解析器方法职责边界

`DocumentConsumer.try_consume()` 位于 `src/documents/consumer.py`，是消费流程的驱动入口。解析器基类在 `src/documents/parsers.py`，其各方法有严格的职责划分：

| 方法 | 参数 | 职责 | 产物（内部状态 / 返回值） |
|------|------|------|---------------------------|
| `parse(document_path, mime_type, produce_archive=...)` | 文档路径 + MIME + 是否生成归档 | 核心 OCR / 文本抽取 + 日期识别 + 可选归档 PDF 生成 | 设置内部属性 `self.text`、`self.date`、`self.archive_path`（无返回值） |
| `get_text()` | 无参 | 只读 getter | 返回 `self.text`（parse 的产物） |
| `get_date()` | 无参 | 只读 getter | 返回 `self.date`（parse 的产物，可能为 None） |
| `get_archive_path()` | 无参 | 只读 getter | 返回 `self.archive_path`（parse 的产物，若 produce_archive=False 则为 None） |
| `get_thumbnail(document_path, mime_type)` | **文档路径 + MIME**（独立传参） | 独立生成缩略图（不依赖 parse 的内部状态） | 返回缩略图临时文件路径 |
| `get_page_count(document_path, mime_type)` | **文档路径 + MIME**（独立传参） | 独立计算页数（不依赖 parse 的内部状态） | 返回 int 或 None |

DocumentConsumer 中的**精确调用顺序**（`consumer.py` 第 505-553 行，全部在同一个 `try` 块内）：

```
① produce_archive = should_produce_archive(document_parser, mime_type, working_copy, ...)
② document_parser.parse(working_copy, mime_type, produce_archive=produce_archive)
     └─ 仅设置 self.text / self.date / self.archive_path（若开启归档）
③ thumbnail = document_parser.get_thumbnail(working_copy, mime_type)
     └─ 独立生成缩略图（parse 之后执行，传 working_copy 和 mime_type）
④ text = document_parser.get_text()               ← 无参 getter
⑤ date = document_parser.get_date()               ← 无参 getter；若 None 则回退到文件名解析
⑥ archive_path = document_parser.get_archive_path()  ← 无参 getter
⑦ page_count = document_parser.get_page_count(working_copy, mime_type)
     └─ 独立计算页数（parse 之后执行，传 working_copy 和 mime_type）
```

**关键事实：**
- `parse()` **不负责**生成缩略图和计算页数——这两项由独立方法完成，且都需要重新传入文档路径和 MIME 类型。
- `get_text()` / `get_date()` / `get_archive_path()` 是纯 getter，不执行任何计算。
- 若 `get_date()` 返回 None，Consumer 会进一步调用外部日期解析器（基于文件名和文本内容回退识别）。

### 1.2 Document 模型中的内容字段

`Document.content` 字段在 `src/documents/models.py` 中定义：

```python
content = models.TextField(
    _("content"),
    blank=True,
    help_text=_("The raw, text-only data of the document. This field is primarily used for searching."),
)

content_length = models.GeneratedField(
    expression=Length("content"),
    output_field=PositiveIntegerField(default=0),
    db_persist=True,
    ...
)
```

在消费流程 `_store()` 方法中，文本直接作为参数传入：

```python
document = Document.objects.create(
    title=title[:127],
    content=text,        # ← 来自 parser 的 OCR 结果
    mime_type=mime_type,
    ...
)
```

### 1.3 有效内容（Effective Content）—— 多版本机制

Paperless-ngx 支持同一文档保存多个版本（重新 OCR 会产生新版本）。搜索索引时并不直接使用 `self.content`，而是使用 `get_effective_content()`（`src/documents/models.py`）：

**返回优先级：**
1. 若 QuerySet 预先注解了 `effective_content` 属性，直接返回。
2. 若当前是**版本文档**（`root_document_id is not None`），返回自身 `content`。
3. 若当前是**根文档**：
   - 优先从 `prefetch_related("versions")` 缓存中取最大 ID 版本的 `content`
   - 否则查数据库：`Document.objects.filter(root_document=self).order_by("-id").values_list("content", flat=True).first()`
   - 没有版本时回退到根文档自身的 `content`

这样确保即使文档重新 OCR 生成了新版本，搜索始终能命中最新的文本。

---

## 二、数据库 → Tantivy 索引构建

### 2.1 索引触发时机

索引更新通过以下路径触发：

**路径 A：消费完成自定义 Signal（主路径）**

`src/documents/signals/__init__.py` 定义了三个自定义 Django Signal：
```python
document_consumption_started = Signal()
document_consumption_finished = Signal()
document_updated = Signal()
```

在 `src/documents/apps.py` 的 `ready()` 中按以下顺序注册 receiver（顺序决定执行先后）：
```python
document_consumption_finished.connect(add_inbox_tags)
document_consumption_finished.connect(set_correspondent)
document_consumption_finished.connect(set_document_type)
document_consumption_finished.connect(set_tags)
document_consumption_finished.connect(set_storage_path)
document_consumption_finished.connect(add_to_index)         # ← 第 6 个执行
document_consumption_finished.connect(run_workflows_added)
document_consumption_finished.connect(add_or_update_document_in_llm_index)
```

> **关键时序事实**：`document_consumption_finished.send()` **发生在媒体文件复制、归档文件写入和最终 document.save() 之前**。整个消费流程包裹在单个 `transaction.atomic()` 中，精确顺序如下（`src/documents/consumer.py`）：
>
> ```
> transaction.atomic()
>   │
>   ├─ ① Document.objects.create(...) / version.save()  ← 第一次 DB 保存
>   │     此时已有字段：title / content / mime_type / checksum /
>   │     created / modified / page_count / original_filename
>   │     （filename、archive_filename、correspondent、tags 等仍为空）
>   │
>   ├─ ② document_consumption_finished.send()  ← Signal 在此发送！
>   │     └─ receiver 链按注册顺序执行：
>   │          add_inbox_tags → set_correspondent → set_document_type
>   │          → set_tags → set_storage_path → add_to_index ← 此处写索引
>   │          → run_workflows_added → add_or_update_document_in_llm_index
>   │     注：前 5 个 receiver 会修改并保存 document（set_correspondent 等），
>   │         所以 add_to_index 执行时 correspondent / tags / storage_path 已入库。
>   │
>   ├─ ③ FileLock(MEDIA_LOCK) 内的文件复制
>   │     ├─ 生成唯一文件名 → 设置 document.filename
>   │     ├─ 写入原始文件 → document.source_path
>   │     ├─ 写入缩略图 → document.thumbnail_path
>   │     └─ 写入归档 PDF（若有）→ 设置 archive_filename / archive_checksum
>   │
>   └─ ④ document.save()  ← 最终保存（只持久化 filename / archive_filename / archive_checksum）
>          （不触发 add_to_index，因为它是消费完成 Signal 的 receiver，
>           而非 Django ORM 的 post_save receiver）
> ```
>
> **关于索引完整性的说明**：Tantivy Schema 中只有 `original_filename` 字段，没有 `filename` 和 `archive_filename`（见 2.2 节 Schema 表）。因此第 ④ 步的最终 save 不影响索引内容，索引在第 ② 步 receiver 链中写入即为完整数据。

`add_to_index`（`src/documents/signals/handlers.py`）实现：
```python
def add_to_index(sender, document, **kwargs) -> None:
    from documents.search import get_backend
    get_backend().add_or_update(
        document,
        effective_content=document.get_effective_content(),
    )
```

注意：这**不是** Django ORM 的 `post_save` 信号，而是应用自定义的消费完成信号。

**路径 B：API 操作显式调用**

在 `src/documents/views.py` 中，文档更新、删除、批量编辑、版本上传等操作直接调用：
- `get_backend().add_or_update(doc)`
- `get_backend().remove(doc.pk)`

**路径 C：管理命令全量重建**

`src/documents/management/commands/document_index.py` 的 `reindex` 子命令：
```python
documents = Document.objects.select_related(...).prefetch_related("tags", "notes", "custom_fields", "versions")
get_backend().rebuild(documents, iter_wrapper=...)
```

`rebuild()` 内部对每篇文档也调用 `document.get_effective_content()`。

### 2.2 Tantivy Schema 定义

索引字段结构由 `src/documents/search/_schema.py` 中 `build_schema()` 构建。

**Tantivy API 默认值说明：** `add_text_field()` 和 `add_json_field()` 的 `indexed` 默认为 `True`（自动建倒排索引，可查询），`fast` 默认为 `False`；`add_unsigned_field()` 和 `add_date_field()` 的参数必须显式指定。代码注释和实际参数之间的差异已在校对时注明。

所有字段如下（逐项与代码核对）：

| 类别 | 字段名 | Tantivy 类型 | stored | indexed | fast | 分词器 | 说明 |
|------|--------|-------------|--------|---------|------|--------|------|
| 主键 | `id` | unsigned | ✓ | ✓ | ✓ | — | Django 主键 |
| 校验和 | `checksum` | text | ✓ | ✓ | — | raw | 原始文件 MD5；stored 便于取回；raw 分词器保留完整值；不在任何搜索字段列表中 |
| 全文搜索 | `title` | text | ✓ | ✓ | — | paperless_text | 标题；QUERY 模式默认字段，权重 2.0x |
| 全文搜索 | `correspondent` | text | ✓ | ✓ | — | paperless_text | 往来方名称；QUERY 模式默认字段 |
| 全文搜索 | `document_type` | text | ✓ | ✓ | — | paperless_text | 文档类型名；QUERY 模式默认字段 |
| 全文搜索 | `storage_path` | text | ✓ | ✓ | — | paperless_text | 存储路径名；已建索引，但**不在** `DEFAULT_SEARCH_FIELDS`，需显式字段语法查询 |
| 全文搜索 | `original_filename` | text | ✓ | ✓ | — | paperless_text | 原始文件名；已建索引，但**不在** `DEFAULT_SEARCH_FIELDS`，需显式字段语法查询 |
| 全文搜索 | `content` | text | ✓ | ✓ | — | paperless_text | OCR 正文；QUERY 模式默认字段 |
| 排序影子字段 | `title_sort` | text | — | ✓ | ✓ | simple_analyzer | 仅用于 fast field 排序；代码注释写 "not stored/indexed"，但实际 `indexed` 默认为 True |
| 排序影子字段 | `correspondent_sort` | text | — | ✓ | ✓ | simple_analyzer | 仅用于 fast field 排序 |
| 排序影子字段 | `type_sort` | text | — | ✓ | ✓ | simple_analyzer | 仅用于 fast field 排序 |
| CJK 支持 | `bigram_content` | text | — | ✓ | — | bigram_analyzer | 中日韩文 2-gram 二元语法；代码注释 "not stored, indexed only"，准确；不在默认搜索字段列表，需显式 `bigram_content:xxx` |
| 简单子串搜索 | `simple_title` | text | — | ✓ | — | simple_search_analyzer | TEXT/TITLE 模式 `?text=` / `?title_search=` 用；`SIMPLE_SEARCH_FIELDS` 成员 |
| 简单子串搜索 | `simple_content` | text | — | ✓ | — | simple_search_analyzer | TEXT 模式用；`SIMPLE_SEARCH_FIELDS` 成员 |
| 自动补全 | `autocomplete_word` | text | ✓ | ✓ | — | raw | 归一化后的单词；raw 分词器保留完整 token；代码注释写 "not indexed"，但实际 `indexed` 默认为 True——**必须 indexed 才能被 `terms_with_prefix()` 扫描 term dictionary** 做前缀匹配 |
| 标签 | `tag` | text | ✓ | ✓ | — | paperless_text | 标签名（多值，一个文档多 tag 多次 add）；QUERY 模式默认字段 |
| 结构化 JSON | `notes` | json | ✓ | ✓ | — | paperless_text | 备注结构化查询：`notes.user:alice`、`notes.note:keyword` |
| 备注纯文本 | `notes_text` | text | ✓ | ✓ | — | paperless_text | 备注纯文本副本；Tantivy 的 `SnippetGenerator` 不支持 JSON 字段高亮，所以需要该文本字段做高亮 |
| 结构化 JSON | `custom_fields` | json | ✓ | ✓ | — | paperless_text | 自定义字段结构化查询：`custom_fields.name:invoice`、`custom_fields.value:1000` |
| 过滤用 ID | `correspondent_id` | unsigned | — | ✓ | ✓ | — | 多值，term 查询过滤 |
| 过滤用 ID | `document_type_id` | unsigned | — | ✓ | ✓ | — | 多值 |
| 过滤用 ID | `storage_path_id` | unsigned | — | ✓ | ✓ | — | 多值 |
| 过滤用 ID | `tag_id` | unsigned | — | ✓ | ✓ | — | 多值 |
| 过滤用 ID | `owner_id` | unsigned | — | ✓ | ✓ | — | 单值，权限过滤用 |
| 过滤用 ID | `viewer_id` | unsigned | — | ✓ | ✓ | — | 多值，共享权限过滤用 |
| 日期 | `created` | date | ✓ | ✓ | ✓ | — | 文档创建日期；SORTABLE_FIELDS 成员 |
| 日期 | `modified` | date | ✓ | ✓ | ✓ | — | 最后修改；SORTABLE_FIELDS 成员 |
| 日期 | `added` | date | ✓ | ✓ | ✓ | — | 入库时间；SORTABLE_FIELDS 成员 |
| 数值 | `asn` | unsigned | ✓ | ✓ | ✓ | — | 归档序列号；SORTABLE_FIELDS 成员 |
| 数值 | `page_count` | unsigned | ✓ | ✓ | ✓ | — | 页数；SORTABLE_FIELDS 成员 |
| 数值 | `num_notes` | unsigned | ✓ | ✓ | ✓ | — | 备注数；SORTABLE_FIELDS 成员 |

**查询字段列表对照（`src/documents/search/_query.py`）：**
- `DEFAULT_SEARCH_FIELDS`（QUERY 模式）：`title`, `content`, `correspondent`, `document_type`, `tag`
- `SIMPLE_SEARCH_FIELDS`（TEXT 模式）：`simple_title`, `simple_content`
- `TITLE_SEARCH_FIELDS`（TITLE 模式）：`simple_title`
- `_FIELD_BOOSTS`：`{"title": 2.0}`
- `_SIMPLE_FIELD_BOOSTS`：`{"simple_title": 2.0}`

### 2.3 分词器（Tokenizer）

分词器在 `src/documents/search/_tokenizer.py` 中注册。每个 Index 打开后必须重新注册（Tantivy 的要求）：

| 分词器名 | 处理流水线 | 用途 |
|----------|-----------|------|
| `paperless_text` | simple → remove_long(65) → lowercase → ascii_fold → stemmer(可选) | 主全文字段 |
| `simple_analyzer` | simple → lowercase → ascii_fold | 排序影子字段 + fast field 排序 |
| `bigram_analyzer` | ngram(2,2) → lowercase | CJK 中日韩文 |
| `simple_search_analyzer` | regex(r"\S+") → remove_long(65) → lowercase → ascii_fold | 简单子串搜索 |

`ascii_fold()` 在 `src/documents/search/_normalize.py` 中使用 `unicodedata.normalize("NFD")` + ASCII 忽略编码实现。

### 2.4 文档构建：_build_tantivy_doc()

核心构建函数在 `src/documents/search/_backend.py`，对应 Django Document → Tantivy Document 的转换。要点：

**OCR 正文三份写入：**
```python
doc.add_text("content", content)            # 主全文字段（paperless_text 分词）
doc.add_text("bigram_content", content)      # CJK bigram
doc.add_text("simple_content", content)      # 简单子串搜索（非空白分词）
```

**标题同样三份写入：** `title`、`title_sort`、`simple_title`。

**备注双写（兼容高亮）：**
```python
doc.add_json("notes", {"note": note.note, "user": note.user.username})
doc.add_text("notes_text", " ".join(note_texts))  # Tantivy SnippetGenerator 不支持 JSON
```

**权限字段：**
```python
if document.owner_id:
    doc.add_unsigned("owner_id", document.owner_id)
for user in get_users_with_perms(document, only_with_perms_in=["view_document"]):
    doc.add_unsigned("viewer_id", user.pk)
```

**自动补全词表：**
从 `[title, content, correspondent.name, document_type.name] + tag_names` 中提取所有单词，小写化 + ASCII 折叠后逐条写入 `autocomplete_word`（raw 分词器保留完整值）。

### 2.5 并发控制与 Upsert 语义

`WriteBatch` 上下文管理器（`src/documents/search/_backend.py`）：
- 通过 `filelock.FileLock`（锁文件 `.tantivy.lock`）保证多进程安全。
- `add_or_update()` 语义：先 `delete_documents_by_query(id=pk)` 删除旧文档，再 `add_document()` 写入新文档，确保权限变更等不会遗留脏数据。
- `__exit__` 中无异常则 `commit()` + `reload()`，有异常则丢弃 writer 自动回滚。

---

## 三、搜索查询处理流程

### 3.1 API 入口：DocumentViewSet.list()

搜索请求处理逻辑在 `src/documents/views.py` 的 `list()` 方法内。

**判断是否搜索请求：**
检查 URL 查询参数中是否包含 `text`、`title_search`、`query`、`more_like_id` 中**恰好一个**。同时出现多个会返回 `ValidationError`。

**完整执行步骤：**

```
① parse_search_params()
   │  提取 sort_field_name / sort_reverse / use_tantivy_sort / 分页
   ▼
② backend = get_backend()
③ filtered_qs = self.filter_queryset(self.get_queryset())
   │  应用 DocumentFilterSet（标签、日期、自定义字段等结构化条件）
   ▼
④ user = None if superuser else request.user
   ▼
⑤ ┌─ more_like_id → run_more_like_this()
  └─ 其他        → run_text_search()
       │
       ├─ backend.search_ids(...)           ← Tantivy 匹配 + 权限过滤
       ├─ intersect_and_order(all_ids, qs)  ← 与 ORM 结果取交集
       └─ backend.highlight_hits(...)       ← 当前页高亮
   ▼
⑥ 包装为 TantivyRelevanceList → paginate_queryset() → 序列化 → Response
```

### 3.2 三种搜索模式

搜索模式定义在 `src/documents/search/_backend.py` 的 `SearchMode` 枚举：

| 模式 | URL 参数 | 查询解析函数 | 默认搜索字段 |
|------|----------|-------------|-------------|
| `SearchMode.TEXT` | `?text=xxx` | `parse_simple_text_query()` | `simple_title`, `simple_content` |
| `SearchMode.TITLE` | `?title_search=xxx` | `parse_simple_title_query()` | `simple_title` |
| `SearchMode.QUERY` | `?query=xxx` | `parse_user_query()` | `title`(2.0x), `content`, `correspondent`, `document_type`, `tag` |

参数到模式的映射函数 `_get_tantivy_query_and_mode()` 也在 `views.py` 中。

### 3.3 QUERY 模式查询解析管道

`parse_user_query()`（`src/documents/search/_query.py`）的完整处理流水线：

```
原始查询字符串
    │
    ▼
① rewrite_natural_date_keywords(query, tz)
   │
   ├─ _rewrite_compact_date()         14 位紧凑日期 YYYYMMDDHHmmss → ISO 8601
   ├─ _rewrite_whoosh_relative_range()  [-7 days to now] → [ISO TO ISO]
   ├─ _rewrite_8digit_date()          field:YYYYMMDD → field:[ISO TO ISO]
   ├─ _rewrite_relative_range()       [now-1d TO now+2h] → [ISO TO ISO]
   └─ _FIELD_DATE_RE 替换              field:today / field:"previous quarter"
                                         → field:[ISO TO ISO]
    │
    ▼
② normalize_query(query)
   │
   ├─ tag:foo,bar → tag:foo AND tag:bar
   └─ 多空格 → 单空格
    │
    ▼
③ index.parse_query(query_str, DEFAULT_SEARCH_FIELDS, field_boosts={"title": 2.0})
    │
    ▼
④ [可选] 模糊查询融合（settings.ADVANCED_FUZZY_SEARCH_THRESHOLD 非空时）
   │
   └─ boolean_query([
        Should → exact_query,
        Should → boost_query(fuzzy_query, 0.1),  # 模糊命中权重极低
      ])
```

日期关键词支持：`today`、`yesterday`、`previous week`、`this month`、`previous month`、`this year`、`previous year`、`previous quarter`。

日期字段分两类处理：
- `_DATE_ONLY_FIELDS = {"created"}`（DateField）：直接用 UTC 午夜边界。
- `added`、`modified`（DateTimeField）：用本地时区午夜转换为 UTC，保证"今天"按用户时区计。

### 3.4 TEXT / TITLE 模式（简单子串搜索）

`parse_simple_query()` 使用 `regex_query` / `regex_phrase_query`：

```python
# 第 1 个 token：任意位置子串匹配
patterns[0] = f".*{escaped}.*"
# 后续 token：必须从词首开始（减少误匹配，如不会因 "6" 命中 "16"）
patterns[i] = f"{escaped}.*"

# 单 token → regex_query；多 token → regex_phrase_query（按顺序）
```

### 3.5 权限过滤（Tantivy 内部）

`build_permission_filter()`（`src/documents/search/_query.py`）构建权限 Query，通过 `disjunction_max_query` 组合三分支：

```python
no_owner  = 文档无 owner_id 字段（公开文档）
owned     = owner_id == user.pk
shared    = viewer_id == user.pk

final = boolean_query([
    (Must, user_query),
    (Must, permission_filter),   # 非超级用户必加
])
```

超级用户传 `user=None`，整个权限过滤被跳过。

---

## 四、ORM 结构化过滤层

即使使用 Tantivy 全文搜索，结构化过滤条件仍通过 Django FilterSet 在关系数据库层执行，然后与 Tantivy 结果取交集。

### 4.1 DocumentFilterSet

定义在 `src/documents/filters.py`，主要过滤维度：

| 过滤类别 | 字段 / 参数 | 查找表达式 |
|----------|------------|-----------|
| 基础属性 | `id`, `title`, `original_filename`, `checksum`, `archive_serial_number` | 精确/包含/前缀/后缀等 |
| 日期时间 | `created`(Date), `added`(DateTime), `modified`(DateTime) | 年/月/日/大于/小于等 |
| 关联对象 | `correspondent`, `document_type`, `storage_path`, `tags` | ID 精确、名称匹配、是否为空 |
| 标签组合 | `tags__id__all` / `tags__id__none` / `tags__id__in` | 全部包含/排除/任一包含 |
| 所有权 | `owner`, `owner__id__none`, `shared_by__id` | 归属/共享 |
| 自定义字段 | `custom_field_query` | JSON 表达式（见下） |
| 布尔标志 | `is_tagged`, `is_in_inbox`, `has_custom_fields` | 标签/收件箱/CF 是否存在 |
| 其他 | `mime_type` | MIME 类型包含 |

### 4.2 自定义字段查询解析器

`CustomFieldQueryParser`（`src/documents/filters.py`）解析嵌套 JSON 逻辑表达式：

```json
["AND", [
    ["invoice_date", "gte", "2024-01-01"],
    ["OR", [
        ["amount", "gt", 1000],
        ["status", "exact", "approved"]
    ]],
    ["NOT", ["archived", "exact", true]]
]]
```

**运算符类别：**
- `basic`: `exact`, `in`, `isnull`, `exists`
- `string`: `icontains`, `istartswith`, `iendswith`
- `arithmetic`: `gt`, `gte`, `lt`, `lte`, `range`
- `containment`: `contains`（仅 DOCUMENTLINK 类型）

**安全限制：** 最大嵌套深度 10，最大原子条件数 20。

**实现方式：** 每个原子条件生成 `Count("custom_fields", filter=Q(...))` 注解，再用 `_custom_field_filter_N__gt 0` 判断文档是否满足——绕开不同自定义字段实例不能直接组合过滤的问题。

### 4.3 已废弃但保留的过滤器

- `title_content`：数据库 `LIKE` 查询（deprecated，UI 已改用 Tantivy 的 `text`）
- `custom_fields__icontains`：跨字段模糊搜索（deprecated，改用 Tantivy 或 `custom_field_query`）

---

## 五、搜索结果排序

### 5.1 排序判断逻辑（视图层）

`parse_search_params()` 在 `views.py` 中决定走哪条排序路径：

```python
ordering_param = request.query_params.get("ordering", "")
sort_reverse = ordering_param.startswith("-")
sort_field_name = ordering_param.lstrip("-") or None

use_tantivy_sort = (
    sort_field_name in TantivyBackend.SORTABLE_FIELDS
    or sort_field_name is None
    or sort_field_name == "score"   # ← 视图层特例，不在 SORTABLE_FIELDS 中
)
```

关键事实：**`"score"` 并不在 `TantivyBackend.SORTABLE_FIELDS` 集合里**，它是视图层单独判断的一个特殊分支，用来表示"按相关度排序"。

### 5.2 TantivyBackend.SORTABLE_FIELDS

定义在 `src/documents/search/_backend.py`：

```python
SORTABLE_FIELDS: frozenset[str] = frozenset({
    "created",
    "added",
    "modified",
    "archive_serial_number",
    "page_count",
    "num_notes",
})
```

这些都是数值或日期字段，Tantivy 的 fast field 排序结果与 ORM 排序一致。文本字段（`title`、`correspondent__name`、`document_type__name`）虽然在 `SORT_FIELD_MAP` 中有映射，但被**故意排除**在 `SORTABLE_FIELDS` 之外，因为 tokenized fast field 排序与数据库 collation 排序结果不一致。

字段名 → 索引字段名的映射：
```python
SORT_FIELD_MAP = {
    "title": "title_sort",
    "correspondent__name": "correspondent_sort",
    "document_type__name": "type_sort",
    "created": "created",
    "added": "added",
    "modified": "modified",
    "archive_serial_number": "asn",
    "page_count": "page_count",
    "num_notes": "num_notes",
}
```

### 5.3 双轨排序机制

**路径 A：Tantivy 原生排序（`use_tantivy_sort=True`）**

覆盖三种情况：字段属于 `SORTABLE_FIELDS`、无 `ordering` 参数、`ordering=score`。

在 `run_text_search()` 中的实际调用：
```python
is_score_sort = sort_field_name == "score"
all_ids = backend.search_ids(
    query_str,
    user=user,
    sort_field=(
        None if (not use_tantivy_sort or is_score_sort) else sort_field_name
    ),          # ↑ score 传 None，让 Tantivy 走默认相关度
    sort_reverse=sort_reverse,
    search_mode=search_mode,
)
```

`score` 的特殊性：
1. 传入 `sort_field=None`，Tantivy 按 BM25 分数降序返回。
2. 如果 `ordering=score`（不带 `-`，表示升序，即最差在前），由于 Tantivy 永远按相关度降序输出，需要在 Python 层手动反转：
   ```python
   if is_score_sort and not sort_reverse:
       ordered_ids = list(reversed(ordered_ids))
   ```
3. 相关度搜索还会做分数归一化（除以最高分到 [0,1]），并可按 `ADVANCED_FUZZY_SEARCH_THRESHOLD` 过滤低分结果。

**路径 B：ORM 排序（`use_tantivy_sort=False`）**

文本字段或自定义字段排序时走此路径。Tantivy 结果仅作为 ID 白名单传入 ORM，最终顺序完全由 Django `order_by()` 决定。

### 5.4 交集函数 intersect_and_order()

```python
def intersect_and_order(all_ids, filtered_qs, *, use_tantivy_sort):
    if use_tantivy_sort:
        # 保持 Tantivy 返回的顺序（相关度或数值排序）
        if len(all_ids) <= 5000:
            visible_ids = set(filtered_qs.filter(pk__in=all_ids).values_list("pk", flat=True))
        else:
            # SQLite 的大 IN 子句性能差，回退到全表扫描 + Python 集合
            visible_ids = set(filtered_qs.values_list("pk", flat=True))
        return [doc_id for doc_id in all_ids if doc_id in visible_ids]
    else:
        # ORM 的 order_by 决定顺序，Tantivy 仅过滤 ID
        return list(filtered_qs.filter(id__in=all_ids).values_list("pk", flat=True))
```

阈值常量 `_TANTIVY_INTERSECT_THRESHOLD = 5_000` 定义在 `views.py` 顶部。

### 5.5 自定义字段排序

`DocumentsOrderingFilter`（`src/documents/filters.py`）处理 `ordering=custom_field_{id}` 参数：

- 用 `Subquery` 注解每个文档的自定义字段值（按数据类型选择 `value_text` / `value_int` / `value_float` / `value_date` / `value_monetary_amount` / `value_url` / `value_bool`）。
- `SELECT` 类型特殊：通过 `Case/When` 将选项 ID 映射为按标签字母序排列的整数序号，兼容 SQLite。
- 附加 `has_field` 注解，让有值的文档排在空值前面。

---

## 六、高亮生成

`TantivyBackend.highlight_hits()`（`src/documents/search/_backend.py`）不重新跑搜索排序，只针对给定的 ID 列表按需生成片段：

1. 构建 `Must(user_query) AND Must(id IN [page_ids])` 的批量查询，一次性取回当前页的 Tantivy 文档。
2. 用 `tantivy.SnippetGenerator` 生成 `<b>...</b>` HTML 高亮。
3. `SearchMode.TEXT` 的特殊处理：简单搜索用 regex query 匹配，但 SnippetGenerator 不支持 regex，改用 `parse_simple_text_highlight_query()` 生成普通 term query 在 `content` 字段上高亮。
4. `SearchMode.QUERY` 的备注高亮：JSON 字段不能用 SnippetGenerator，因此先剥掉查询字符串中的字段前缀（如 `notes.note:` → 空串），在纯文本副本 `notes_text` 上重新解析查询再生成片段。

---

## 七、自动补全

`TantivyBackend.autocomplete()` 是最热的搜索路径（每次按键触发）：

1. 输入词先 ASCII 折叠 + 小写化。
2. 调用 Tantivy 的 `terms_with_prefix("autocomplete_word", normalized_term, permission_query, limit)`：
   - `autocomplete_word` 用 raw 分词器，每个词是完整的归一化 token。
   - 前缀扫描利用 Tantivy 的 term dictionary 有序性，性能 O(log N)。
   - 权限 query 传入以过滤不可见文档的词（避免信息泄漏）。
3. 结果按文档频率降序、字母升序排列返回。

---

## 八、索引版本管理

`needs_rebuild()` 在 `src/documents/search/_schema.py` 中读取索引目录下 `.index_settings.json` 比较：
- `schema_version`（当前值 1）
- `language`（`settings.SEARCH_LANGUAGE`）

任一不匹配即触发索引重建，重建后写入新的哨兵文件。

---

## 九、完整调用链

### 索引写入调用链

```
Celery task: documents.tasks.consume_file
  └─ DocumentConsumer.try_consume()
       ├─ produce_archive = should_produce_archive(...)
       ├─ ① document_parser.parse(working_copy, mime_type, produce_archive=produce_archive)
       │     └─ 仅设置内部状态：self.text / self.date / self.archive_path
       ├─ ② thumbnail = document_parser.get_thumbnail(working_copy, mime_type)
       │     └─ 独立生成缩略图（parse 之后，不依赖 parse 内部状态）
       ├─ ③ text = document_parser.get_text()               ← 纯 getter
       ├─ ④ date = document_parser.get_date()
       │     └─ [若 None] 回退：get_date_parser().parse(filename, text)
       ├─ ⑤ archive_path = document_parser.get_archive_path()  ← 纯 getter
       ├─ ⑥ page_count = document_parser.get_page_count(working_copy, mime_type)
       │     └─ 独立计算页数（parse 之后，不依赖 parse 内部状态）
       │
       └─ transaction.atomic()
            │
            ├─ ① 第一次 DB 保存
            │    ├─ 新文档：self._store(text, mime_type, date, page_count)
            │    │     └─ Document.objects.create(
            │    │           title=..., content=text, mime_type=...,
            │    │           checksum=..., created=..., page_count=...,
            │    │           original_filename=self.filename
            │    │        )
            │    └─ 新版本：_create_version_from_root(root_doc, text, ...) + .save()
            │
            ├─ ② document_consumption_finished.send(sender=self.__class__,
            │                                       document=document,
            │                                       classifier=classifier, ...)
            │    │  （注：此时媒体文件、归档文件尚未写入磁盘）
            │    │
            │    └─ apps.py 注册的 8 个 receiver 按序执行：
            │         ├─ add_inbox_tags
            │         ├─ set_correspondent(sender, document)  ← 修改 document.correspondent 并 save
            │         ├─ set_document_type(sender, document)   ← 修改并 save
            │         ├─ set_tags(sender, document)            ← 修改并 save
            │         ├─ set_storage_path(sender, document)    ← 修改并 save
            │         ├─ add_to_index(sender, document)        ← 索引写入在此处
            │         │    └─ get_backend().add_or_update(document,
            │         │              effective_content=document.get_effective_content())
            │         │         └─ WriteBatch
            │         │              ├─ filelock 获取 .tantivy.lock
            │         │              ├─ _build_tantivy_doc(document, effective_content)
            │         │              ├─ writer.delete_documents_by_query(id=pk)  ← upsert
            │         │              ├─ writer.add_document(doc)
            │         │              └─ writer.commit() + index.reload()
            │         ├─ run_workflows_added
            │         └─ add_or_update_document_in_llm_index
            │
            ├─ ③ FileLock(settings.MEDIA_LOCK) 媒体文件落盘
            │    ├─ 生成唯一文件名 generated_filename → document.filename
            │    ├─ 写入原始文件 → document.source_path
            │    ├─ 写入缩略图 → document.thumbnail_path
            │    └─ 归档 PDF 写入（若有）
            │         ├─ 生成唯一归档名 → document.archive_filename
            │         ├─ 写入 archive_path → document.archive_path
            │         └─ document.archive_checksum = compute_checksum(...)
            │
            ├─ ④ document.save()  ← 仅持久化 filename / archive_filename / archive_checksum
            │      （不触发 add_to_index，因为它是消费完成 Signal 的 receiver）
            │
            ├─ [若为新版本] document_updated.send(sender=..., document=root_document)
            │
            └─ 清理临时文件：unlink 原始文件 / working_copy / unmodified_original
```

### 搜索查询调用链

```
GET /api/documents/?query=invoice&tags__id__in=1,2&ordering=-added
  └─ DocumentViewSet.list()
       ├─ _is_search_request() → True（含 query 参数）
       ├─ parse_search_params()
       │    ├─ sort_field_name="added", sort_reverse=True
       │    └─ use_tantivy_sort = "added" in SORTABLE_FIELDS → True
       ├─ backend = get_backend()
       ├─ filtered_qs = filter_queryset(get_queryset())
       │    └─ DocumentFilterSet 应用 tags__id__in=1,2 等结构化条件
       ├─ user = request.user（若非 superuser）
       └─ run_text_search(backend, user, filtered_qs)
            ├─ backend.search_ids("invoice", user, sort_field="added", sort_reverse=True)
            │    ├─ _parse_query("invoice", QUERY)
            │    │    └─ parse_user_query(index, "invoice", tz)
            │    │         ├─ rewrite_natural_date_keywords("invoice") → "invoice"
            │    │         ├─ normalize_query("invoice") → "invoice"
            │    │         └─ index.parse_query("invoice", DEFAULT_SEARCH_FIELDS,
            │    │              field_boosts={"title": 2.0})
            │    ├─ _apply_permission_filter(user_query, user) → Must(query) + Must(perm)
            │    ├─ searcher.search(final_query, order_by_field="added", order=Desc)
            │    └─ fast_field_values("id", doc_addrs) → [doc_id_1, doc_id_2, ...]
            ├─ intersect_and_order(all_ids, filtered_qs, use_tantivy_sort=True)
            │    ├─ visible_ids = set(filtered_qs.filter(pk__in=all_ids)...)
            │    └─ [id for id in all_ids if id in visible_ids]
            └─ backend.highlight_hits("invoice", page_ids, search_mode=QUERY, rank_start=...)
                 ├─ 构建 Must(user_query) AND Must(id IN page_ids)
                 ├─ SnippetGenerator("content") → HTML 片段
                 └─ SnippetGenerator("notes_text") → 备注 HTML 片段
       ├─ TantivyRelevanceList(ordered_ids, page_hits, page_offset)
       ├─ paginate_queryset(rl)
       ├─ serializer(page, many=True)
       └─ get_paginated_response(serializer.data)
```
