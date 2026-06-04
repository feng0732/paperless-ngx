# Paperless-ngx 自动分类机制深度分析

本文档重点分析容易产生误解的链路细节：
1. **RE_WORD 正则表达式**：完整定义与预处理作用
2. **suggestion_content 与 get_effective_content 的继承关联**：完整调用链
3. **训练阶段 doc.content 的版本范围**：训练数据是否包含版本文档

同时保持三个入口的职责划分。

---

## 一、三个自动分类入口的职责划分

### 1.1 总览

| 入口 | 触发方式 | 是否执行分类 | 是否执行工作流 | 分类器来源 |
|------|---------|-------------|--------------|-----------|
| `document_consumption_finished` | 信号（消费完成） | **是** | 是（DOCUMENT_ADDED） | 预加载传入 |
| `document_updated` | 信号（文档更新） | **否** | 是（DOCUMENT_UPDATED） | 无 |
| `document_retagger` | 命令行手动调用 | **是** | **否** | 自行加载 |

### 1.2 `document_consumption_finished` — 首次消费分类

**发送位置**：`src/documents/consumer.py`

```python
classifier = load_classifier()  # 预加载分类器

document_consumption_finished.send(
    sender=self.__class__,
    document=document,
    classifier=classifier,  # 预加载的分类器作为参数传入
    ...
)
```

**信号处理器执行顺序**（按 `src/documents/apps.py` 中的连接顺序）：

| 序号 | 处理器 | 功能 |
|------|-------|------|
| 1 | `add_inbox_tags` | 添加收件箱标签 |
| 2 | `set_correspondent` | 自动分配联系人 |
| 3 | `set_document_type` | 自动分配文档类型 |
| 4 | `set_tags` | 自动分配标签 |
| 5 | `set_storage_path` | 自动分配存储路径 |
| 6 | `add_to_index` | 添加到搜索索引 |
| 7 | `run_workflows_added` | 运行 DOCUMENT_ADDED 工作流 |
| 8 | `add_or_update_document_in_llm_index` | 更新 LLM 索引 |

### 1.3 `document_updated` — 仅触发工作流，不执行分类

**发送位置**：
- `src/documents/views.py` — API 更新
- `src/documents/consumer.py` — 版本文档保存后
- `src/documents/tasks.py` — 批量更新

**信号处理器**：`run_workflows_updated` + `send_websocket_document_updated`

**关键**：`document_updated` **不连接** `set_correspondent`、`set_document_type`、`set_tags`、`set_storage_path`。

### 1.4 `document_retagger` — 手动批量重新分类

**实现位置**：`src/documents/management/commands/document_retagger.py`

**与消费的差异**：

| 特性 | 消费 | 重新标记 |
|------|-----|---------|
| `replace` | `False` | 由 `--overwrite` 控制 |
| `use_first` | `True` | 默认 `False`，`--use-first` 设为 `True` |
| `dry_run` | `False` | 由 `--suggest` 控制 |
| 工作流 | 执行 | **不执行** |

---

## 二、RE_WORD 正则表达式深度解析

### 2.1 正则表达式定义

**定义位置**：`src/documents/classifier.py` 第 42 行

```python
RE_WORD = re.compile(r"\b[\w]+\b")  # words that may contain digits
```

**逐字符解析**：

| 符号 | 含义 | 说明 |
|------|------|------|
| `\b` | 单词边界 | 匹配单词的开始或结束位置 |
| `[` | 字符集开始 | 定义可匹配的字符集合 |
| `\w` | 单词字符 | 等价于 `[a-zA-Z0-9_]`（字母、数字、下划线） |
| `]` | 字符集结束 | - |
| `+` | 一次或多次 | 前面的字符出现至少一次 |
| `\b` | 单词边界 | 匹配单词的结束位置 |

**匹配结果**：匹配被单词边界包围的、由一个或多个单词字符组成的序列。

### 2.2 为什么注释说 "may contain digits"

注释 "words that may contain digits" 指的是 `\w` 包含数字字符 `0-9`，与纯字母匹配不同。

**匹配示例**：
- ✅ `"hello"` → 纯字母单词
- ✅ `"hello123"` → 字母 + 数字
- ✅ `"123"` → 纯数字
- ✅ `"hello_world"` → 含下划线
- ❌ `"hello-world"` → 连字符不是单词字符，会被拆分为 `"hello"` 和 `"world"`
- ❌ `"hello@world"` → `@` 不是单词字符

### 2.3 在预处理中的作用

**定义位置**：`src/documents/classifier.py` 的 `preprocess_content()` 方法（第 503 行）

```python
def preprocess_content(self, content: str, *, shared_cache=True) -> str:
    # 基础处理：转小写 + 提取单词 + 规范化空格
    content = " ".join(match.group().lower() for match in RE_WORD.finditer(content))
    # ...
```

**执行流程**：

```
输入文本: "Hello, World! This is a TEST-document 123."

RE_WORD.finditer() 找到:
  ├─ match.group() = "Hello"
  ├─ match.group() = "World"
  ├─ match.group() = "This"
  ├─ match.group() = "is"
  ├─ match.group() = "a"
  ├─ match.group() = "TEST"
  ├─ match.group() = "document"
  └─ match.group() = "123"

转小写:
  "hello" "world" "this" "is" "a" "test" "document" "123"

空格连接输出:
  "hello world this is a test document 123"
```

**关键效果**：
1. **丢弃非单词字符**：标点符号（`,!-.`）等被完全丢弃
2. **拆分带连字符的词**：`"TEST-document"` → `"TEST"` + `"document"`
3. **保留数字**：纯数字字符串（如 `"123"`）作为独立"单词"保留
4. **转小写**：所有字母转为小写，实现大小写不敏感
5. **规范化空格**：多个空格合并为一个

### 2.4 与 NLTK 分词的关系

```python
if ADVANCED_TEXT_PROCESSING_ENABLED:
    from nltk.tokenize import word_tokenize
    words = word_tokenize(content, language=settings.NLTK_LANGUAGE)
    # 停用词过滤 + 词干还原
```

**注意**：NLTK 的 `word_tokenize` 在 `RE_WORD` 处理之后执行。此时输入已经是：
- 只有单词字符（字母、数字、下划线）
- 全部小写
- 单个空格分隔

### 2.5 对分类结果的影响

- **数字敏感**：发票号、金额等数字信息被保留为"单词"，可能影响分类
- **标点丢失**：`"U.S.A"` → `"U"` + `"S"` + `"A"`，缩写语义丢失
- **连字符拆分**：复合词如 `"state-of-the-art"` 被拆分为独立词
- **下划线保留**：`"hello_world"` 作为单个词

---

## 三、suggestion_content 与 get_effective_content 的继承关联

### 3.1 完整调用链路

```
suggestion_content
    └─→ get_effective_content()
         ├─→ 检查 effective_content 注解
         ├─→ 检查是否为版本文档 (root_document_id is not None)
         │    └─→ 返回 self.content
         └─→ 根文档处理
              ├─→ 检查 _prefetched_objects_cache 中的 versions
              │    ├─→ 空 → 返回 self.content
              │    └─→ 非空 → max(id) 的版本的 .content
              └─→ 无预取 → 查询数据库获取最新版本的 .content
                   └─→ 有版本 → 返回最新版本的 .content
                   └─→ 无版本 → 返回 self.content
```

### 3.2 `get_effective_content()` 的分支逻辑

**定义位置**：`src/documents/models.py` 第 363-400 行

```python
def get_effective_content(self) -> str | None:
    # 分支 1: 查询集注解优化
    if hasattr(self, "effective_content"):
        return getattr(self, "effective_content")

    # 分支 2: 版本文档，直接返回自身 content
    if self.root_document_id is not None or self.pk is None:
        return self.content

    # 分支 3: 根文档，尝试获取最新版本的 content
    prefetched_cache = getattr(self, "_prefetched_objects_cache", None)
    prefetched_versions = (
        prefetched_cache.get("versions")
        if isinstance(prefetched_cache, dict)
        else None
    )
    if prefetched_versions is not None:
        # 分支 3a: 使用预取的版本数据
        if not prefetched_versions:
            return self.content
        latest_prefetched = max(prefetched_versions, key=lambda doc: doc.id)
        return latest_prefetched.content

    # 分支 3b: 查询数据库获取最新版本
    latest_version_content = (
        Document.objects.filter(root_document=self)
        .order_by("-id")
        .values_list("content", flat=True)
        .first()
    )
    return (
        latest_version_content
        if latest_version_content is not None
        else self.content
    )
```

### 3.3 `suggestion_content` 的继承与扩展

**定义位置**：`src/documents/models.py` 第 402-428 行

```python
@property
def suggestion_content(self):
    """
    Returns the document text used to generate suggestions.
    """
    effective_content = self.get_effective_content()  # ← 100% 继承

    if not effective_content or len(effective_content) <= 1200000:
        return effective_content  # 小文档：直接返回
    else:
        # 大文档：截断处理
        head_len = 800000
        tail_len = 200000
        return " ".join((
            effective_content[:head_len],
            effective_content[-tail_len:],
        ))
```

**继承关系的精确描述**：

| 文档类型 | `suggestion_content` 输出 | 与 `get_effective_content()` 的关系 |
|---------|-------------------------|---------------------------------|
| **版本文档** | `self.content` | 完全相同 |
| **根文档（无版本）** | `self.content` | 完全相同 |
| **根文档（有版本）** | 最新版本的 `.content` | 完全相同 |
| **小文档（≤1.2M）** | 完整内容 | 完全相同 |
| **大文档（>1.2M）** | 头 800K + 尾 200K | **截断版本**，不相同 |

**结论**：
- `suggestion_content` **100% 基于** `get_effective_content()` 的结果
- 小文档：`suggestion_content == get_effective_content()`
- 大文档：`suggestion_content = truncate(get_effective_content())`

### 3.4 截断的中间内容丢弃

对于 > 1.2M 字符的文档：

```
原内容（1.5M字符）:
  [ 头 800K ][ 中间 500K ][ 尾 200K ]
                     ↑
                  被丢弃

suggestion_content 输出:
  [ 头 800K ] + " " + [ 尾 200K ]
```

**影响**：超大文档的中间部分内容完全不参与 ML 预测。

---

## 四、训练阶段 doc.content 的版本范围

### 4.1 训练查询集的定义

**定义位置**：`src/documents/classifier.py` 的 `train()` 方法（第 228-234 行）

```python
# Get non-inbox documents
docs_queryset = (
    Document.objects.exclude(
        tags__is_inbox_tag=True,
    )
    .select_related("document_type", "correspondent", "storage_path")
    .prefetch_related("tags")
    .order_by("pk")
)
```

**查询集过滤条件分析**：

| 过滤条件 | 含义 |
|---------|------|
| `.exclude(tags__is_inbox_tag=True)` | 排除带有"收件箱标签"的文档 |
| `.select_related(...)` | 关联查询优化，非过滤 |
| `.prefetch_related("tags")` | 预取标签优化，非过滤 |
| `.order_by("pk")` | 按主键排序，非过滤 |

**关键发现**：查询集 **没有** 任何 `root_document_id is null` 的过滤条件。

### 4.2 训练数据包含哪些文档

Document 表中有两类 Document 对象：

| 类型 | `root_document_id` | 说明 |
|------|-------------------|------|
| 根文档 | `NULL` | 原始文档，可拥有多个版本 |
| 版本文档 | 指向根文档的 ID | 由用户创建的新版本，挂靠在根文档下 |

**训练数据包含**：
- ✅ 非收件箱的**根文档**（`root_document_id IS NULL`）
- ✅ 非收件箱的**版本文档**（`root_document_id IS NOT NULL`）

**两者都**会进入训练数据。

### 4.3 训练时使用的 `doc.content`

**定义位置**：`src/documents/classifier.py` 的 `content_generator()`（第 328-333 行）

```python
def content_generator() -> Iterator[str]:
    for doc in docs_queryset:
        # ⚠️ 直接使用 doc.content，不调用 get_effective_content()
        yield self.preprocess_content(doc.content, shared_cache=False)
```

**对两类文档的不同含义**：

| 文档类型 | 训练时使用的 `doc.content` | 预测时使用的 `get_effective_content()` | 是否一致 |
|---------|---------------------------|--------------------------------------|---------|
| **根文档（无版本）** | 根文档自身的 `.content` | 根文档自身的 `.content` | ✅ 一致 |
| **根文档（有版本）** | 根文档自身的 `.content` | **最新版本**的 `.content` | ❌ **不一致** |
| **版本文档（训练数据中）** | 版本文档自身的 `.content` | 版本文档自身的 `.content`（作为独立文档预测时） | ✅ 一致 |

### 4.4 版本场景下的训练-预测不一致

**场景示例**：

```
根文档 Doc-A (pk=100, root_document_id=NULL):
  content = "原始 OCR 内容，有很多错误..."
  有一个版本 Doc-A-v2 (pk=101, root_document_id=100):
    content = "人工修正后的正确内容..."

训练阶段:
  docs_queryset 包含 Doc-A 和 Doc-A-v2（如果都不是收件箱）
  训练使用 Doc-A.content = "原始 OCR 内容，有很多错误..."
  训练使用 Doc-A-v2.content = "人工修正后的正确内容..."

预测阶段（对 Doc-A 预测）:
  get_effective_content() → Doc-A-v2.content = "人工修正后的正确内容..."
  预测使用版本内容

不一致性:
  训练 Doc-A 使用"原始 OCR 内容"
  预测 Doc-A 使用"人工修正后的正确内容"
```

**影响范围**：
- 只影响**拥有版本的根文档**
- 版本文档自身作为训练样本时，训练-预测是一致的
- 根文档的训练样本是原始内容，预测使用的是版本内容

### 4.5 为什么不调用 `get_effective_content()`

可能的设计考虑：

1. **训练性能**：`get_effective_content()` 对每个根文档执行一次数据库查询获取最新版本，大规模训练时会产生 N+1 查询
2. **数据独立性**：每个 Document 对象（包括版本）作为独立训练样本，保持数据完整性
3. **版本也是有效样本**：版本文档自身也是人工标注/修正的结果，作为训练样本有价值

---

## 五、两条输入路径的完整对比

### 5.1 路径概览

```
                    文档 Document
                          │
         ┌────────────────┴────────────────┐
         │                                 │
  get_effective_content()          suggestion_content
         │                                 │
   用于规则匹配                      用于 ML 预测
         │                                 │
   matches() 函数                predict_*() 函数
（MATCH_ANY/ALL/LITERAL...）    （MATCH_AUTO）
         │                                 │
         │                        preprocess_content()
         │                          RE_WORD 提取单词
         │                          转小写
         │                          NLTK 分词/词干
         │                                 │
         │                         CountVectorizer
         │                          1-gram + 2-gram
         │                                 │
         └─────────────────────────────────┘
                   输入来源不同，但目标一致
```

### 5.2 四种文本来源对比表

| 特性 | `get_effective_content()` | `suggestion_content` | 训练 `doc.content` | 预测预处理后 |
|------|--------------------------|---------------------|-------------------|--------------|
| **定义位置** | `src/documents/models.py` | `src/documents/models.py` | `src/documents/classifier.py` | `src/documents/classifier.py` |
| **调用起点** | `matches()` | `match_*()` → `predict_*()` | `train()` | `_vectorize()` |
| **考虑版本** | ✅ 是 | ✅ 是（继承） | ❌ 否（直接 `.content`） | ✅ 是（继承） |
| **内容截断** | ❌ 无 | ✅ >1.2M 时截断 | ❌ 无 | ✅ 继承截断 |
| **RE_WORD 处理** | ❌ 无 | ❌ 无 | ✅ 有 | ✅ 有 |
| **转小写** | ❌ 无 | ❌ 无 | ✅ 有 | ✅ 有 |
| **NLTK 处理** | ❌ 无 | ❌ 无 | 可选 | 可选 |
| **向量化** | ❌ 无 | ❌ 无 | ✅ CountVectorizer | ✅ CountVectorizer |
| **用于规则匹配** | ✅ 是 | ❌ 否 | ❌ 否 | ❌ 否 |
| **用于 ML 预测** | ❌ 否 | ✅ 是 | ❌ 否 | ✅ 是 |
| **用于 ML 训练** | ❌ 否 | ❌ 否 | ✅ 是 | ❌ 否 |

---

## 六、代码引用汇总

| 功能模块 | 文件（仓库相对路径） |
|---------|-------------------|
| `RE_WORD` 正则定义 | `src/documents/classifier.py` |
| `preprocess_content()` 方法 | `src/documents/classifier.py` |
| `get_effective_content()` 方法 | `src/documents/models.py` |
| `suggestion_content` 属性 | `src/documents/models.py` |
| 训练查询集定义 | `src/documents/classifier.py` |
| 训练 `content_generator()` | `src/documents/classifier.py` |
| 信号连接配置 | `src/documents/apps.py` |
| 消费完成信号发送 | `src/documents/consumer.py` |
| 规则匹配 `matches()` | `src/documents/matching.py` |
| 组合匹配 `match_*()` | `src/documents/matching.py` |
| 匹配算法定义 | `src/documents/models.py` |
| 分类决策函数 | `src/documents/signals/handlers.py` |
| 重新标记命令 | `src/documents/management/commands/document_retagger.py` |
