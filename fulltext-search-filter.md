# Paperless-ngx 全文索引与搜索过滤数据流

本文档梳理 Paperless-ngx 中 OCR 文本 → 索引字段 → 过滤条件 → 排序结果的完整数据流。

---

## 整体架构概览

```
文档消费阶段                          搜索查询阶段
──────────────                        ──────────────
原始文件
  │
  ▼
DocumentParser (OCR提取)              HTTP Request
  │                                     │
  ▼                                     ▼
Document.content (DB存储)            views.DocumentViewSet.list()
  │                                     │
  ▼                                     ▼
add_to_index (signal)                1. 解析搜索参数
  │                                     2. TantivyBackend.search_ids()
  ▼                                     3. ORM FilterSet 过滤
TantivyBackend._build_tantivy_doc()    4. intersect_and_order 交集
  │                                     5. highlight_hits 高亮
  ▼                                     6. 序列化 + 分页
Tantivy Index (磁盘)                    ▼
                                       JSON Response
```

核心搜索引擎：**Tantivy**（Rust 实现的全文搜索引擎，类似 Lucene）。

相关代码目录：
- 搜索核心：[src/documents/search/](file:///d:/fz/0601/solo-dogfeeding/code/56-paperless-ngx/src/documents/search)
- 过滤层：[filters.py](file:///d:/fz/0601/solo-dogfeeding/code/56-paperless-ngx/src/documents/filters.py)
- API 视图层：[views.py](file:///d:/fz/0601/solo-dogfeeding/code/56-paperless-ngx/src/documents/views.py)
- 数据模型：[models.py](file:///d:/fz/0601/solo-dogfeeding/code/56-paperless-ngx/src/documents/models.py)

---

## 一、OCR 文本提取 → 数据库存储

### 1.1 解析器提取文本

文档消费阶段，各类 `DocumentParser` 子类（如 Tesseract OCR 解析器）负责从文件中提取文本：

- 基类定义在 [parsers.py](file:///d:/fz/0601/solo-dogfeeding/code/56-paperless-ngx/src/documents/parsers.py#L205-L224)，每个解析器维护 `self.text` 字段存储提取的文本。
- 解析完成后，文本被写入 `Document.content` 数据库字段。

### 1.2 Document 模型中的内容字段

在 [models.py](file:///d:/fz/0601/solo-dogfeeding/code/56-paperless-ngx/src/documents/models.py#L189-L205) 中定义：

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

### 1.3 有效内容（Effective Content）—— 版本机制

Paperless-ngx 支持文档版本（多版本 OCR 结果）。搜索时使用的是"有效内容"而非简单的 `self.content`。

方法 [get_effective_content()](file:///d:/fz/0601/solo-dogfeeding/code/56-paperless-ngx/src/documents/models.py#L363-L400) 逻辑：

1. 如果 QuerySet 已预先注解了 `effective_content`，直接返回。
2. 如果是**版本文档**（`root_document_id` 不为空），返回自身 `content`。
3. 如果是**根文档**：
   - 优先使用预取缓存中最新版本的 `content`
   - 否则查询数据库获取 `versions` 中最新一条的 `content`
   - 没有版本时回退到自身 `content`

这个机制确保了即使重新 OCR 产生新版本，搜索也能使用最新文本。

---

## 二、数据库 → Tantivy 索引构建

### 2.1 索引触发时机

索引更新通过以下路径触发：

**路径 A：Signal 自动触发**
[handlers.py:add_to_index()](file:///d:/fz/0601/solo-dogfeeding/code/56-paperless-ngx/src/documents/signals/handlers.py#L794-L800) 在文档消费流水线中被调用：

```python
def add_to_index(sender, document, **kwargs) -> None:
    from documents.search import get_backend
    get_backend().add_or_update(
        document,
        effective_content=document.get_effective_content(),
    )
```

**路径 B：API 操作触发**
在 [views.py](file:///d:/fz/0601/solo-dogfeeding/code/56-paperless-ngx/src/documents/views.py) 中，文档创建/更新/删除/批量编辑时直接调用 `get_backend().add_or_update()` 或 `.remove()`。

**路径 C：管理命令全量重建**
[document_index.py](file:///d:/fz/0601/solo-dogfeeding/code/56-paperless-ngx/src/documents/management/commands/document_index.py) 的 `reindex` 子命令调用 `get_backend().rebuild()` 从数据库批量重建。

### 2.2 Tantivy Schema 定义

索引字段结构由 [_schema.py:build_schema()](file:///d:/fz/0601/solo-dogfeeding/code/56-paperless-ngx/src/documents/search/_schema.py#L22-L100) 构建。字段分类如下：

| 类别 | 字段名 | 类型 | 说明 |
|------|--------|------|------|
| **主键** | `id` | unsigned | 文档 ID，stored+indexed+fast |
| **全文搜索字段** | `title`, `correspondent`, `document_type`, `storage_path`, `original_filename`, `content` | text | 使用 `paperless_text` 分词器，stored |
| **排序专用字段** | `title_sort`, `correspondent_sort`, `type_sort` | text | 使用 `simple_analyzer`，fast=True（不存储不索引，仅用于排序） |
| **CJK 支持** | `bigram_content` | text | bigram 分词（2-gram），仅索引不存储 |
| **简单子串搜索** | `simple_title`, `simple_content` | text | `simple_search_analyzer`（非空白分词+ASCII折叠），仅索引 |
| **自动补全** | `autocomplete_word` | text | raw 分词器，存储所有归一化后的单词 |
| **标签** | `tag` | text | 标签名 |
| **结构化 JSON** | `notes`, `custom_fields` | json | 支持 `notes.user:alice`、`custom_fields.name:invoice` 语法 |
| **备注纯文本** | `notes_text` | text | notes 的纯文本副本，用于高亮生成（Tantivy 的 SnippetGenerator 不支持 JSON 字段） |
| **过滤用 ID** | `correspondent_id`, `document_type_id`, `storage_path_id`, `tag_id`, `owner_id`, `viewer_id` | unsigned | indexed+fast，用于权限和分类过滤 |
| **日期** | `created`, `modified`, `added` | date | stored+indexed+fast |
| **数值** | `asn`, `page_count`, `num_notes` | unsigned | stored+indexed+fast |

### 2.3 分词器（Tokenizer）注册

分词器由 [_tokenizer.py:register_tokenizers()](file:///d:/fz/0601/solo-dogfeeding/code/56-paperless-ngx/src/documents/search/_tokenizer.py#L52-L76) 注册，共 4 种：

| 分词器 | 用途 | 处理流程 |
|--------|------|----------|
| `paperless_text` | 主全文搜索 | simple → remove_long(65) → lowercase → ascii_fold → stemmer(可选) |
| `simple_analyzer` | 排序影子字段 | simple → lowercase → ascii_fold |
| `bigram_analyzer` | CJK 语言 | ngram(2,2) → lowercase |
| `simple_search_analyzer` | 简单子串搜索 | regex(r"\S+") → remove_long(65) → lowercase → ascii_fold |

归一化函数 `ascii_fold()` 在 [_normalize.py](file:///d:/fz/0601/solo-dogfeeding/code/56-paperless-ngx/src/documents/search/_normalize.py#L6-L8)，使用 Unicode NFD 分解后丢弃非 ASCII 字符。

### 2.4 文档构建：_build_tantivy_doc()

核心构建函数在 [_backend.py:_build_tantivy_doc()](file:///d:/fz/0601/solo-dogfeeding/code/56-paperless-ngx/src/documents/search/_backend.py#L357-L480)。关键处理逻辑：

**OCR 内容多字段写入：**
```python
doc.add_text("content", content)           # 主全文字段
doc.add_text("bigram_content", content)     # CJK 二元语法
doc.add_text("simple_content", content)     # 简单子串搜索
```

**备注双写：**
```python
doc.add_json("notes", {"note": note.note, "user": note.user.username})  # 结构化查询
doc.add_text("notes_text", " ".join(note_texts))                         # 高亮生成
```

**权限字段：**
```python
if document.owner_id:
    doc.add_unsigned("owner_id", document.owner_id)
for user in get_users_with_perms(document, only_with_perms_in=["view_document"]):
    doc.add_unsigned("viewer_id", user.pk)
```

**自动补全单词提取：**
[_extract_autocomplete_words()](file:///d:/fz/0601/solo-dogfeeding/code/56-paperless-ngx/src/documents/search/_backend.py#L58-L80) 从 title、content、correspondent、document_type、tags 中提取所有单词，ASCII 折叠 + 小写化后写入 `autocomplete_word` 字段。

### 2.5 索引写入的并发控制

`WriteBatch` 上下文管理器（[_backend.py:155-235](file:///d:/fz/0601/solo-dogfeeding/code/56-paperless-ngx/src/documents/search/_backend.py#L155-L235)）通过 `filelock.FileLock` 实现进程安全。`add_or_update()` 采用 upsert 语义：先按 ID 删除旧文档，再写入新文档。

---

## 三、搜索查询处理流程

### 3.1 API 入口：DocumentViewSet.list()

搜索请求的处理逻辑在 [views.py:list()](file:///d:/fz/0601/solo-dogfeeding/code/56-paperless-ngx/src/documents/views.py#L2250-L2451)。

**步骤 1：判断是否为搜索请求**
检查 URL 参数中是否包含 `text`、`title_search`、`query`、`more_like_id` 之一。这些参数互斥，同时指定多个会报错。

**步骤 2：解析搜索参数**
`parse_search_params()` 提取：
- `sort_field_name`：排序字段（去掉 `-` 前缀）
- `sort_reverse`：是否降序
- `use_tantivy_sort`：是否使用 Tantivy 原生排序（仅数值/日期字段，文本排序走 ORM）
- 分页参数

**步骤 3：并行获取数据**
```
Tantivy Backend              Django ORM
     │                           │
     ▼                           ▼
search_ids()              filter_queryset()
返回匹配的 ID 列表        返回通过所有结构化过滤条件的 QuerySet
     │                           │
     └───────────┬───────────────┘
                 ▼
          intersect_and_order()
          取交集并保持 Tantivy 返回的顺序
```

**步骤 4：交集与排序**
`intersect_and_order()` 在 [views.py:2299-2323](file:///d:/fz/0601/solo-dogfeeding/code/56-paperless-ngx/src/documents/views.py#L2299-L2323) 实现：
- 如果 `use_tantivy_sort=True`：以 Tantivy 返回的 ID 顺序为主，用 ORM 过滤可见 ID。小结果集（≤5000）用 `pk__in` 查询，大结果集走全表扫描+Python 集合交集（SQLite 的大 IN 子句性能差）。
- 如果 `use_tantivy_sort=False`：以 ORM 排序为准，Tantivy 结果仅作为 ID 过滤条件。

**步骤 5：高亮生成分页**
对当前页的文档 ID 调用 `backend.highlight_hits()` 生成高亮片段。

**步骤 6：包装与序列化**
使用 `TantivyRelevanceList` 适配 DRF 分页接口（实现 `__len__` 和 `__getitem__`），然后正常走序列化流程。

### 3.2 三种搜索模式

搜索模式枚举定义在 [_backend.py:SearchMode](file:///d:/fz/0601/solo-dogfeeding/code/56-paperless-ngx/src/documents/search/_backend.py#L52-L56)：

| 模式 | URL 参数 | 查询解析函数 | 搜索字段 |
|------|----------|-------------|----------|
| `TEXT` | `?text=xxx` | `parse_simple_text_query()` | `simple_title`, `simple_content` |
| `TITLE` | `?title_search=xxx` | `parse_simple_title_query()` | `simple_title` |
| `QUERY` | `?query=xxx` | `parse_user_query()` | `title`, `content`, `correspondent`, `document_type`, `tag` |

参数到模式的映射在 [_get_tantivy_query_and_mode()](file:///d:/fz/0601/solo-dogfeeding/code/56-paperless-ngx/src/documents/views.py#L269-L278)。

### 3.3 查询解析管道（QUERY 模式）

`parse_user_query()` 在 [_query.py:493-548](file:///d:/fz/0601/solo-dogfeeding/code/56-paperless-ngx/src/documents/search/_query.py#L493-L548) 实现完整管道：

```
原始查询字符串
    │
    ▼
rewrite_natural_date_keywords()   ← 日期语法重写
    │  - 14位紧凑日期: YYYYMMDDHHmmss → ISO 8601
    │  - Whoosh相对范围: [-7 days to now] → [ISO TO ISO]
    │  - 8位日期: created:20240115 → created:[ISO TO ISO]
    │  - Tantivy相对范围: [now-1d TO now] → [ISO TO ISO]
    │  - 自然关键词: created:today, added:"previous month"
    │
    ▼
normalize_query()                 ← 语法规范化
    │  - tag:foo,bar → tag:foo AND tag:bar
    │  - 多空格 → 单空格
    │
    ▼
Tantivy parse_query()             ← 引擎原生解析
    │  默认搜索字段: title(2.0x), content, correspondent,
    │                document_type, tag
    │
    ▼
[可选] 模糊查询融合               ← ADVANCED_FUZZY_SEARCH_THRESHOLD
       Should(exact_query, 0.1 * fuzzy_query)
```

**日期关键词处理**在 [_query.py:87-203](file:///d:/fz/0601/solo-dogfeeding/code/56-paperless-ngx/src/documents/search/_query.py#L87-L203) 中区分 `_DATE_ONLY_FIELDS`（`created`，DateField）和 DateTimeField（`added`、`modified`）：前者用 UTC 午夜边界，后者用本地时区午夜转换为 UTC。

支持的日期关键词：`today`, `yesterday`, `previous week`, `this month`, `previous month`, `this year`, `previous year`, `previous quarter`。

### 3.4 简单搜索（TEXT / TITLE 模式）

使用 `parse_simple_query()` 构建正则子串查询：

- 首个词：`.*token.*`（可匹配词中）
- 后续词：`token.*`（必须词首开始，减少误匹配）
- 多词使用 `regex_phrase_query` 按顺序匹配

分词走 `_simple_query_tokens()`：ASCII 折叠 + 小写化。

### 3.5 权限过滤

搜索时的权限过滤在 Tantivy 内部完成（不走 ORM），由 [build_permission_filter()](file:///d:/fz/0601/solo-dogfeeding/code/56-paperless-ngx/src/documents/search/_query.py#L413-L442) 构建：

```python
disjunction_max_query([
    no_owner,     # 公开文档（无 owner_id）
    owned,        # owner_id == user.pk
    shared,       # viewer_id == user.pk
])
```

使用 `Occur.Must` 与用户查询组合：

```python
boolean_query([
    (Must, user_query),
    (Must, permission_filter),
])
```

超级用户跳过权限过滤（`user=None`）。

---

## 四、ORM 结构化过滤层

即使使用 Tantivy 全文搜索，结构化过滤条件仍然通过 Django FilterSet 在关系数据库层执行。

### 4.1 DocumentFilterSet 定义

[filters.py:DocumentFilterSet](file:///d:/fz/0601/solo-dogfeeding/code/56-paperless-ngx/src/documents/filters.py#L776-L867) 支持的过滤维度：

| 分类 | 字段 | 查找表达式 |
|------|------|-----------|
| 基础 | `id`, `title`, `archive_serial_number`, `original_filename`, `checksum` | 精确/包含/前缀/后缀等 |
| 日期 | `created`(Date), `added`(DateTime), `modified`(DateTime) | 年/月/日/大于/小于等 |
| 关联对象 | `correspondent`, `document_type`, `storage_path`, `tags` | ID 精确匹配、名称匹配、是否为空 |
| 所有权 | `owner` | 是否为空、ID 匹配 |
| 标签组合 | `tags__id__all`/`none`/`in` | 全部匹配/排除/任一匹配 |
| 自定义字段 | `custom_field_query` | JSON 表达式（见下文） |
| 特殊 | `is_tagged`, `is_in_inbox`, `has_custom_fields`, `mime_type`, `shared_by__id` | 布尔标志 |

### 4.2 自定义字段查询解析器

`CustomFieldQueryParser` 在 [filters.py:367-748](file:///d:/fz/0601/solo-dogfeeding/code/56-paperless-ngx/src/documents/filters.py#L367-L748) 实现了嵌套逻辑表达式。

**语法示例：**
```json
["AND", [
    ["my_date_field", "gte", "2024-01-01"],
    ["OR", [
        ["amount", "gt", 1000],
        ["status", "exact", "approved"]
    ]]
]]
```

**支持的运算符类别：**
- `basic`: `exact`, `in`, `isnull`, `exists`
- `string`: `icontains`, `istartswith`, `iendswith`
- `arithmetic`: `gt`, `gte`, `lt`, `lte`, `range`
- `containment`: `contains`（仅 DOCUMENTLINK 类型）

**安全限制：** 最大嵌套深度 10，最大原子条件数 20，防止构造恶意复杂 SQL。

实现方式：对每个原子条件使用 `Count()` 注解配合 `filter` 参数，统计匹配的自定义字段数量，然后通过 `> 0` 判断文档是否满足条件。

### 4.3 已废弃的过滤器

以下过滤器保留但已废弃（供旧保存视图兼容）：
- `title_content`：数据库 LIKE 查询，应改用 Tantivy `text` 参数
- `custom_fields__icontains`：全字段模糊搜索，应改用 Tantivy 字段语法或 `custom_field_query`

---

## 五、搜索结果排序

### 5.1 双轨排序机制

排序分两条路径执行，在 `parse_search_params()` 中根据字段决定：

**路径 A：Tantivy 原生排序（`use_tantivy_sort=True`）**
适用字段在 [_backend.py:SORTABLE_FIELDS](file:///d:/fz/0601/solo-dogfeeding/code/56-paperless-ngx/src/documents/search/_backend.py#L268-L277) 中定义：
`created`, `added`, `modified`, `archive_serial_number`, `page_count`, `num_notes`，以及 `score`（相关度，默认）。

字段名映射通过 `SORT_FIELD_MAP` 转换：
```python
SORT_FIELD_MAP = {
    "title": "title_sort",
    "correspondent__name": "correspondent_sort",
    "document_type__name": "type_sort",
    "created": "created",
    ...
}
```

虽然 `title` 等文本字段在映射表中，但**它们不在 SORTABLE_FIELDS 中**，因为 Tantivy 的 tokenized fast field 排序结果与 ORM 的数据库 collation 排序不一致。

**路径 B：ORM 排序（`use_tantivy_sort=False`）**
不适用于 Tantivy 排序的字段（如文本字段、自定义字段）走 Django ORM 的 `order_by()`，此时 Tantivy 结果仅作为 ID 过滤。

### 5.2 相关度排序（score）

无显式排序字段或 `ordering=score` 时使用 Tantivy BM25 相关度：

1. 无排序字段调用 `searcher.search(query)` 返回按分数降序排列的结果。
2. 分数归一化（除以最大分数）到 [0, 1] 区间。
3. 如配置了 `ADVANCED_FUZZY_SEARCH_THRESHOLD`，过滤掉归一化分数低于阈值的结果。
4. `ordering=score`（升序）时在 Python 层反转列表。

### 5.3 自定义字段排序

`DocumentsOrderingFilter` 在 [filters.py:978-1114](file:///d:/fz/0601/solo-dogfeeding/code/56-paperless-ngx/src/documents/filters.py#L978-L1114) 中处理 `ordering=custom_field_{id}` 参数：

- 使用 `Subquery` 注解获取每个文档对应自定义字段的值
- `SELECT` 类型特殊处理：通过 `Case/When` 将选项 ID 映射为按标签排序的序号
- 附加 `has_field` 注解确保有值的文档排在前面

---

## 六、高亮生成

`highlight_hits()` 在 [_backend.py:515-648](file:///d:/fz/0601/solo-dogfeeding/code/56-paperless-ngx/src/documents/search/_backend.py#L515-L648) 中实现，不重新执行搜索排序，只针对给定 ID 生成片段：

1. 构建 `Must(user_query) AND Must(id IN [...])` 的批量查询，一次取回所有需要的文档。
2. 使用 Tantivy 的 `SnippetGenerator` 生成 HTML 高亮。
3. TEXT 模式使用 `parse_simple_text_highlight_query()`（普通 term 查询替代 regex 查询，因为 SnippetGenerator 不支持 regex）。
4. QUERY 模式额外处理备注高亮：去掉查询中的字段前缀（如 `notes.note:`）后在 `notes_text` 字段上生成高亮（因为 SnippetGenerator 不支持 JSON 字段）。

---

## 七、自动补全

`autocomplete()` 在 [_backend.py:714-759](file:///d:/fz/0601/solo-dogfeeding/code/56-paperless-ngx/src/documents/search/_backend.py#L714-L759)：

1. 输入词先 ASCII 折叠+小写化。
2. 调用 Tantivy 的 `terms_with_prefix()` 在 `autocomplete_word` 字段上做前缀扫描。
3. 传入权限过滤 Query 确保不泄漏不可见文档中的词。
4. 结果按文档频率降序、字母升序排列。

这是最热的搜索路径（每次按键调用），代码注释建议后续可加 Redis 缓存。

---

## 八、索引版本管理

索引需要重建的判断逻辑在 [_schema.py:needs_rebuild()](file:///d:/fz/0601/solo-dogfeeding/code/56-paperless-ngx/src/documents/search/_schema.py#L103-L130)：

读取索引目录下 `.index_settings.json`，对比：
- `schema_version`（当前为 1）
- `language`（`settings.SEARCH_LANGUAGE`）

任一不匹配即触发重建。重建时写入新的哨兵文件。

---

## 九、完整调用链总结

### 索引写入调用链
```
consume_file task
  → DocumentParser.parse() 提取 self.text
  → Document.save(content=text)
  → consume() 流水线
    → add_to_index(sender, document)
      → get_backend().add_or_update(document, effective_content=...)
        → WriteBatch.__enter__() (获取文件锁)
        → _build_tantivy_doc(document)  ← 所有字段填充
        → writer.delete_documents_by_query(id=pk)
        → writer.add_document(doc)
        → writer.commit() + index.reload()
```

### 搜索查询调用链
```
GET /api/documents/?query=invoice&tags__id__in=1,2
  → DocumentViewSet.list()
    → parse_search_params() 提取 sort/page
    → run_text_search()
      → backend.search_ids("invoice", sort_field=None, ...)
        → _parse_query()
          → rewrite_natural_date_keywords()
          → normalize_query()
          → parse_user_query() → exact (+ fuzzy)
        → _apply_permission_filter()
        → searcher.search(final_query)
        → 分数归一化 + 阈值过滤
        → fast_field_values("id", ...)
      → intersect_and_order(tantivy_ids, orm_filtered_qs)
        → ORM filter_queryset() 应用 tags__id__in 等
        → 取 ID 交集
      → backend.highlight_hits(query, page_ids)
    → TantivyRelevanceList 包装
    → paginate_queryset() + serializer
    → Response
```
