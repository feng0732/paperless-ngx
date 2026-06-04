# Paperless-ngx 自动分类机制深度分析

本文档重点分析：
1. 三个自动分类入口的职责划分（`document_consumption_finished`、`document_updated`、`document_retagger`）
2. 自动分类的输入来源和元数据提取链路
3. `get_effective_content`、`suggestion_content`、训练阶段 `doc.content` 与预测文本之间的差异

---

## 一、三个自动分类入口的职责划分

### 1.1 总览

| 入口 | 触发方式 | 是否执行分类 | 是否执行工作流 | 分类器来源 |
|------|---------|-------------|--------------|-----------|
| `document_consumption_finished` | 信号（消费完成） | **是** | 是（DOCUMENT_ADDED） | 预加载传入 |
| `document_updated` | 信号（文档更新） | **否** | 是（DOCUMENT_UPDATED） | 无 |
| `document_retagger` | 命令行手动调用 | **是** | **否** | 自行加载 |

### 1.2 `document_consumption_finished` — 首次消费分类

**职责**：新文档消费完成后，执行自动分类 + 收件箱标签 + 搜索索引 + 工作流。

**发送位置**：`src/documents/consumer.py`，在文档保存成功后发送：

```python
# 消费流程中预加载分类器
classifier = load_classifier()

# 文档保存后发送信号
document_consumption_finished.send(
    sender=self.__class__,
    document=document,
    logging_group=self.logging_group,
    classifier=classifier,  # 预加载的分类器作为参数传入
    original_file=...,
)
```

**信号处理器执行顺序**（按 `src/documents/apps.py` 中的连接顺序）：

| 序号 | 处理器 | 功能 | 参数特征 |
|------|-------|------|---------|
| 1 | `add_inbox_tags` | 添加收件箱标签 | 无分类器参数 |
| 2 | `set_correspondent` | 自动分配联系人 | `replace=False, use_first=True` |
| 3 | `set_document_type` | 自动分配文档类型 | `replace=False, use_first=True` |
| 4 | `set_tags` | 自动分配标签 | `replace=False` |
| 5 | `set_storage_path` | 自动分配存储路径 | `replace=False, use_first=True` |
| 6 | `add_to_index` | 添加到搜索索引 | 无分类器参数 |
| 7 | `run_workflows_added` | 运行 DOCUMENT_ADDED 工作流 | 可覆盖分类结果 |
| 8 | `add_or_update_document_in_llm_index` | 更新 LLM 索引 | 无分类器参数 |

### 1.3 `document_updated` — 仅触发工作流，不执行分类

**职责**：文档更新后触发工作流和 WebSocket 通知，**不执行任何自动分类**。

**发送位置**：
- `src/documents/views.py` — 用户通过 API 更新文档时
- `src/documents/consumer.py` — 版本文档保存后通知根文档更新
- `src/documents/tasks.py` — 批量更新任务中

**信号处理器**（仅2个）：

| 处理器 | 功能 |
|-------|------|
| `run_workflows_updated` | 运行 DOCUMENT_UPDATED 类型工作流 |
| `send_websocket_document_updated` | 发送 WebSocket 更新通知 |

**关键区别**：`document_updated` **不连接** `set_correspondent`、`set_document_type`、`set_tags`、`set_storage_path`。更新文档时不会重新运行分类逻辑。

### 1.4 `document_retagger` — 手动批量重新分类

**职责**：使用当前分类模型对已有文档重新执行分类，**不执行工作流**。

**实现位置**：`src/documents/management/commands/document_retagger.py`

**与信号驱动的差异**：

| 特性 | `document_consumption_finished` | `document_retagger` |
|------|-------------------------------|-------------------|
| 触发方式 | 自动（信号） | 手动（命令行） |
| 分类器加载 | consumer 预加载传入 | 自行 `load_classifier()` |
| `replace` | `False` | 由 `--overwrite` 参数控制 |
| `use_first` | `True` | 由 `--use-first` 参数控制，默认 `False` |
| `dry_run` | `False` | 由 `--suggest` 参数控制 |
| 工作流 | 执行 | **不执行** |
| 分类目标 | 全部 | 可选（`-c/-T/-t/-s`） |

**命令行参数与函数参数映射**：

```python
set_correspondent(
    None, document,
    classifier=classifier,
    replace=overwrite,    # --overwrite / -f
    use_first=use_first,  # --use-first
    dry_run=suggest,      # --suggest
)
```

---

## 二、自动分类的输入来源与元数据提取链路

自动分类涉及两条独立的输入路径：**规则匹配路径** 和 **机器学习预测路径**。两者使用不同的文本来源。

### 2.1 两条输入路径概览

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
```

### 2.2 `get_effective_content()` — 规则匹配的输入

**定义位置**：`src/documents/models.py`

**核心用途**：为文本规则匹配提供文档内容。

```python
def get_effective_content(self) -> str | None:
    """
    For root documents, this is the latest version's content when available.
    For version documents, this is always the document's own content.
    """
    if hasattr(self, "effective_content"):
        return getattr(self, "effective_content")  # 查询集注解优化

    if self.root_document_id is not None or self.pk is None:
        return self.content  # 版本文档返回自身 content

    # 根文档：优先使用最新版本的 content
    prefetched_cache = getattr(self, "_prefetched_objects_cache", None)
    if prefetched_cache and "versions" in prefetched_cache:
        # 使用预取的版本数据
    else:
        # 查询数据库获取最新版本
        latest_version_content = Document.objects.filter(root_document=self)...first()

    return latest_version_content if latest_version_content is not None else self.content
```

**关键特性**：
- ✅ **考虑文档版本**：根文档优先使用最新版本的内容
- ✅ **查询集优化**：支持 `effective_content` 注解避免 N+1 查询
- ✅ **完整内容**：不做任何截断，返回完整文本
- ✅ **预取缓存**：利用 `_prefetched_objects_cache` 提高性能

**使用场景**：
- `matches()` 函数（`src/documents/matching.py`）：所有非 AUTO 算法的匹配
- MATCH_ANY / MATCH_ALL / MATCH_LITERAL / MATCH_REGEX / MATCH_FUZZY

### 2.3 `suggestion_content` — ML 预测的输入

**定义位置**：`src/documents/models.py`

**核心用途**：为机器学习分类预测提供优化的文档内容。

```python
@property
def suggestion_content(self):
    """
    Returns the document text used to generate suggestions.

    If the document content length exceeds a specified limit,
    the text is cropped to include the start and end segments.
    """
    effective_content = self.get_effective_content()
    if not effective_content or len(effective_content) <= 1200000:
        return effective_content
    else:
        # Use 80% from the start and 20% from the end
        head_len = 800000
        tail_len = 200000
        return " ".join((
            effective_content[:head_len],
            effective_content[-tail_len:],
        ))
```

**关键特性**：
- ✅ **基于 get_effective_content**：同样考虑文档版本
- ✅ **大文档截断**：超过 1,200,000 字符时，取前 800,000 + 后 200,000 字符
- ⚠️ **截断策略**：保留开头和结尾上下文，丢弃中间内容
- ✅ **性能优化**：减少超大文档的处理时间和内存占用

**使用场景**：
- `match_correspondents()`、`match_document_types()`、`match_tags()`、`match_storage_paths()`
- MATCH_AUTO 算法的 ML 预测输入

### 2.4 训练阶段的 `doc.content` — 训练数据输入

**定义位置**：`src/documents/classifier.py` 的 `train()` 方法

**核心用途**：用于 ML 分类器的训练数据向量化。

```python
# src/documents/classifier.py
def train(self, queryset=None) -> bool:
    # Step 1: 收集标签数据
    docs_queryset = Document.objects.exclude(tags__is_inbox_tag=True)
    for doc in docs_queryset:
        # 收集 AUTO 类型的元数据作为训练标签
        dt = doc.document_type
        if dt and dt.matching_algorithm == MatchingModel.MATCH_AUTO:
            y = dt.pk
        ...

    # Step 2: 向量化
    def content_generator() -> Iterator[str]:
        for doc in docs_queryset:
            # ⚠️ 直接使用 doc.content，不调用 get_effective_content()
            yield self.preprocess_content(doc.content, shared_cache=False)

    self.data_vectorizer = CountVectorizer(ngram_range=(1, 2), min_df=0.01)
    data_vectorized = self.data_vectorizer.fit_transform(content_generator())
```

**关键特性**：
- ⚠️ **不调用 get_effective_content()**：直接使用 `doc.content`
- ⚠️ **不考虑文档版本**：只使用根文档自身的 content，忽略版本
- ✅ **完整内容**：不做任何截断
- ✅ **预处理**：经过 `preprocess_content()` 处理

**使用场景**：
- 分类器训练时的特征向量化
- 训练阶段的文本输入

### 2.5 预测阶段的文本预处理

预测阶段的完整链路：

```python
# 1. 获取 suggestion_content（可能截断）
suggestion_content = document.suggestion_content

# 2. 调用预测函数
pred_id = classifier.predict_correspondent(suggestion_content)

# 3. 预测函数内部：向量化（含预处理）
def predict_correspondent(self, content: str) -> int | None:
    if self.correspondent_classifier:
        X = self._vectorize(content)  # ← 内部调用 preprocess_content
        correspondent_id = self.correspondent_classifier.predict(X)
        return correspondent_id if correspondent_id != -1 else None
    return None

# 4. _vectorize 调用 preprocess_content（shared_cache=True 默认）
def _vectorize(self, content: str):
    result = self.data_vectorizer.transform([
        self.preprocess_content(content)  # ← shared_cache=True
    ])
```

---

## 三、四种文本来源的详细对比

### 3.1 对比表

| 特性 | `get_effective_content()` | `suggestion_content` | 训练 `doc.content` | 预测 `suggestion_content` → 预处理 |
|------|--------------------------|---------------------|-------------------|----------------------------------|
| **定义位置** | `src/documents/models.py` | `src/documents/models.py` | `src/documents/classifier.py` | `src/documents/classifier.py` |
| **调用链起点** | `matches()` 函数 | `match_*()` 函数 | `train()` → `content_generator` | `predict_*()` → `_vectorize()` |
| **考虑文档版本** | ✅ 是 | ✅ 是（基于 get_effective_content） | ❌ 否（直接 doc.content） | ✅ 是（基于 suggestion_content） |
| **内容截断** | ❌ 无截断 | ✅ >1.2M 时截断 | ❌ 无截断 | ✅ 继承 suggestion_content 的截断 |
| **文本预处理** | ❌ 无（原始文本） | ❌ 无（原始文本） | ✅ 预处理（`shared_cache=False`） | ✅ 预处理（`shared_cache=True`） |
| **用于规则匹配** | ✅ 是 | ❌ 否 | ❌ 否 | ❌ 否 |
| **用于 ML 预测** | ❌ 否 | ✅ 是 | ❌ 否 | ✅ 是（预测阶段内部） |
| **用于 ML 训练** | ❌ 否 | ❌ 否 | ✅ 是 | ❌ 否 |
| **查询集注解优化** | ✅ 支持 `effective_content` | ❌ 不支持 | ❌ 不支持 | ❌ 不支持 |
| **预取缓存利用** | ✅ `_prefetched_objects_cache` | ✅ 继承 | ❌ 不利用 | ✅ 继承 |

### 3.2 训练与预测的不一致性

**不一致点 1：文档版本处理**

| 阶段 | 版本处理 |
|------|---------|
| 训练 | ❌ 直接使用 `doc.content`，忽略版本 |
| 预测 | ✅ 使用 `suggestion_content` → `get_effective_content()`，考虑版本 |

**影响**：如果根文档有版本，训练使用的是根文档原始 content，预测使用的是版本 content。可能导致训练数据与预测数据不一致。

**不一致点 2：大文档截断**

| 阶段 | 截断策略 |
|------|---------|
| 训练 | ❌ 完整 `doc.content`，无截断 |
| 预测 | ✅ >1.2M 时截断为头 800K + 尾 200K |

**影响**：大文档在训练时使用完整内容，预测时使用截断内容。对于超大文档（>1.2M字符），可能存在训练-预测分布不一致。

### 3.3 预处理的一致性

`preprocess_content()` 函数在训练和预测阶段**逻辑一致**：

```python
def preprocess_content(self, content: str, *, shared_cache=True) -> str:
    # 基础处理：转小写 + 提取单词 + 规范化空格
    content = " ".join(match.group().lower() for match in RE_WORD.finditer(content))

    if ADVANCED_TEXT_PROCESSING_ENABLED:
        # NLTK 处理：分词 → 停用词过滤 → 词干还原
        words = word_tokenize(content, language=settings.NLTK_LANGUAGE)
        content = self.stem_and_skip_stop_words(words, shared_cache=shared_cache)

    return content
```

**唯一区别**：`shared_cache` 参数
- 训练：`shared_cache=False`，不需要跨 worker 缓存
- 预测：`shared_cache=True`，使用共享 LRU 缓存提高性能

**预处理逻辑本身完全一致**，不存在不一致。

---

## 四、文本预处理的详细流程

### 4.1 基础预处理（始终执行）

```python
# src/documents/classifier.py
RE_WORD = re.compile(r"[^\W_]+", re.UNICODE)

content = " ".join(match.group().lower() for match in RE_WORD.finditer(content))
```

**作用**：
- 提取所有单词字符（字母、数字、下划线以外的都丢弃）
- 转为小写
- 规范化空格

### 4.2 高级 NLTK 处理（可选，`NLTK_ENABLED`）

```python
# 1. 分词
words = word_tokenize(content, language=settings.NLTK_LANGUAGE)

# 2. 停用词过滤 + 词干还原
def stem_and_skip_stop_words(self, words, *, shared_cache=True):
    stemmer = SnowballStemmer(settings.NLTK_LANGUAGE)
    result = []
    for word in words:
        if word not in self._stop_words:
            stem = stemmer.stem(word)  # 如 "amazement" → "amaz"
            result.append(stem)
    return " ".join(result)
```

**词干缓存**：
- 训练：`shared_cache=False`，每个训练独立
- 预测：`shared_cache=True`，跨 worker 共享，10,000 条目 LRU 缓存

### 4.3 特征向量化

```python
self.data_vectorizer = CountVectorizer(
    analyzer="word",
    ngram_range=(1, 2),  # 提取 1-gram 和 2-gram
    min_df=0.01,         # 忽略出现在少于 1% 文档中的词
)
```

**向量化缓存**（仅预测阶段）：
- 缓存键：`sha256(content + 版本 + NLTK 配置 + 向量化器哈希)`
- 缓存时长：5 分钟

---

## 五、规则命中与匹配算法

### 5.1 七种匹配算法

`MatchingModel` 在 `src/documents/models.py` 中定义了 7 种匹配算法：

| 枚举值 | 名称 | 输入来源 | 匹配逻辑 |
|-------|------|---------|---------|
| `MATCH_NONE = 0` | 不匹配 | - | 始终返回 False |
| `MATCH_ANY = 1` | 任意词 | `get_effective_content()` | 匹配字符串中任一词 |
| `MATCH_ALL = 2` | 全词 | `get_effective_content()` | 匹配字符串中所有词 |
| `MATCH_LITERAL = 3` | 精确匹配 | `get_effective_content()` | 整个匹配字符串精确匹配 |
| `MATCH_REGEX = 4` | 正则表达式 | `get_effective_content()` | 正则匹配 |
| `MATCH_FUZZY = 5` | 模糊匹配 | `get_effective_content()` | `rapidfuzz.partial_ratio`，阈值 90 |
| `MATCH_AUTO = 6` | 自动分类 | `suggestion_content` | ML 预测 |

### 5.2 核心匹配函数 `matches()`

实现位置：`src/documents/matching.py`

```python
def matches(matching_model: MatchingModel, document: Document):
    document_content = document.get_effective_content() or ""  # ← 使用 get_effective_content

    if not matching_model.match.strip():
        return False

    # 根据算法分发...
    elif matching_model.matching_algorithm == MatchingModel.MATCH_AUTO:
        return False  # ← AUTO 在此直接返回 False，由外层 match_* 函数处理
```

### 5.3 组合匹配逻辑

`match_correspondents()` 等函数结合规则匹配和自动预测：

```python
def match_correspondents(document, classifier, user=None):
    pred_id = (
        classifier.predict_correspondent(document.suggestion_content)  # ← 使用 suggestion_content
        if classifier else None
    )

    return list(filter(
        lambda o: (
            matches(o, document)                              # 规则匹配：get_effective_content
            or (
                o.pk == pred_id                                 # ML 预测：suggestion_content
                and o.matching_algorithm == MatchingModel.MATCH_AUTO
            )
        ),
        correspondents,
    ))
```

**OR 关系**：规则匹配成功 **或** 自动预测命中（且算法为 AUTO）。

### 5.4 匹配真值表

| 规则算法 | 规则输入 | `matches()` 结果 | ML 输入 | ML 预测命中 | 最终匹配 |
|---------|---------|-----------------|---------|-----------|---------|
| `MATCH_ANY` | `get_effective_content` | True | - | - | **True** |
| `MATCH_ANY` | `get_effective_content` | False | - | - | **False** |
| `MATCH_AUTO` | `get_effective_content` | False（恒） | `suggestion_content` | 是 | **True** |
| `MATCH_AUTO` | `get_effective_content` | False（恒） | `suggestion_content` | 否 | **False** |

---

## 六、分类决策与冲突处理

### 6.1 分类决策函数参数

| 函数 | `replace` 默认 | `use_first` 默认 | 特殊处理 |
|------|---------------|-----------------|---------|
| `set_correspondent` | `False` | `True` | 多匹配取首 |
| `set_document_type` | `False` | `True` | 多匹配取首 |
| `set_tags` | `False` | - | 全部添加，无冲突 |
| `set_storage_path` | `False` | `True` | 多匹配取首 |

### 6.2 多匹配冲突决策

```python
potential_count = len(potential_correspondents)
if potential_count > 1:
    if use_first:
        # 返回候选列表第一个（按 name 字母排序）
    else:
        return None  # 多匹配时不分配
```

### 6.3 已有值保护

```python
# 单值属性：联系人、文档类型、存储路径
if document.correspondent and not replace:
    return None  # 不覆盖已有值

# 标签（replace=True 时的保留规则）：
# ✅ 保留 is_inbox_tag=True 的收件箱标签
# ✅ 保留 match="" 且 matching_algorithm != MATCH_AUTO 的手动标签
# ❌ 删除其他所有标签
```

### 6.4 工作流覆盖优先级

`document_consumption_finished` 信号中，工作流在分类之后执行，**具有最终决定权**：

```
执行顺序:
1. set_correspondent     → 自动分类
2. set_document_type     → 自动分类
3. set_tags              → 自动分类
4. set_storage_path      → 自动分类
5. run_workflows_added   → 工作流可覆盖以上所有值
```

---

## 七、代码引用汇总

| 功能模块 | 文件（仓库相对路径） |
|---------|-------------------|
| 信号连接配置 | `src/documents/apps.py` |
| 消费完成信号发送 | `src/documents/consumer.py` |
| `get_effective_content` 定义 | `src/documents/models.py` |
| `suggestion_content` 定义 | `src/documents/models.py` |
| 规则匹配实现（`matches()`） | `src/documents/matching.py` |
| 组合匹配逻辑（`match_*`） | `src/documents/matching.py` |
| 匹配算法定义 | `src/documents/models.py` |
| 分类器训练（`doc.content` 使用） | `src/documents/classifier.py` |
| 文本预处理（`preprocess_content`） | `src/documents/classifier.py` |
| 预测函数（`predict_*`） | `src/documents/classifier.py` |
| 分类决策函数 | `src/documents/signals/handlers.py` |
| 重新标记命令 | `src/documents/management/commands/document_retagger.py` |
| 批量更新信号发送 | `src/documents/tasks.py` |
| API 更新信号发送 | `src/documents/views.py` |
