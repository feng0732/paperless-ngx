# Paperless-ngx 自动分类机制深度分析

基于代码分析，本文档详细阐述 Paperless-ngx 中文档自动分类的完整流程，包括匹配条件、规则命中、元数据提取、分类决策和冲突处理机制。

---

## 一、自动分类整体架构

### 1.1 核心模块组成

自动分类系统由以下核心模块协同工作：

| 模块 | 核心文件 | 主要职责 |
|------|---------|---------|
| 匹配模型定义 | [models.py](file:///d:/fz/0601/solo-dogfeeding/code/27-paperless-ngx/src/documents/models.py#L46-L93) | 定义7种匹配算法和匹配数据结构 |
| 规则匹配引擎 | [matching.py](file:///d:/fz/0601/solo-dogfeeding/code/27-paperless-ngx/src/documents/matching.py) | 执行具体的规则匹配逻辑 |
| 机器学习分类器 | [classifier.py](file:///d:/fz/0601/solo-dogfeeding/code/27-paperless-ngx/src/documents/classifier.py) | 基于 MLP 神经网络的自动分类 |
| 分类决策处理 | [handlers.py](file:///d:/fz/0601/solo-dogfeeding/code/27-paperless-ngx/src/documents/signals/handlers.py) | 分类结果决策和冲突处理 |
| 消费流程触发 | [consumer.py](file:///d:/fz/0601/solo-dogfeeding/code/27-paperless-ngx/src/documents/consumer.py) | 文档消费时触发分类流程 |
| AI 分类扩展 | [ai_classifier.py](file:///d:/fz/0601/solo-dogfeeding/code/27-paperless-ngx/src/paperless_ai/ai_classifier.py) | LLM 辅助分类和 RAG 增强 |

### 1.2 分类触发时机

自动分类通过 Django 信号机制在以下时机触发：

1. **文档消费完成**：`document_consumption_finished` 信号
2. **文档更新**：`document_updated` 信号
3. **批量重新标记**：`document_retagger` 命令

信号连接配置在 [apps.py](file:///d:/fz/0601/solo-dogfeeding/code/27-paperless-ngx/src/documents/apps.py#L24-L33) 中定义：

```python
document_consumption_finished.connect(add_inbox_tags)
document_consumption_finished.connect(set_correspondent)
document_consumption_finished.connect(set_document_type)
document_consumption_finished.connect(set_tags)
document_consumption_finished.connect(set_storage_path)
document_consumption_finished.connect(add_to_index)
document_consumption_finished.connect(run_workflows_added)
```

---

## 二、匹配条件详解

### 2.1 七种匹配算法

`MatchingModel` 在 [models.py](file:///d:/fz/0601/solo-dogfeeding/code/27-paperless-ngx/src/documents/models.py#L46-L63) 中定义了 7 种匹配算法：

| 算法 | 枚举值 | 描述 | 匹配逻辑 |
|------|--------|------|---------|
| `MATCH_NONE` | 0 | 不匹配 | 始终返回 False |
| `MATCH_ANY` | 1 | 任意词匹配 | 匹配字符串中任意一个词出现在文档中 |
| `MATCH_ALL` | 2 | 全词匹配 | 匹配字符串中所有词都必须出现在文档中 |
| `MATCH_LITERAL` | 3 | 精确匹配 | 整个匹配字符串作为整体精确匹配 |
| `MATCH_REGEX` | 4 | 正则表达式 | 使用正则表达式进行匹配 |
| `MATCH_FUZZY` | 5 | 模糊匹配 | 使用快速模糊匹配算法（阈值 90%） |
| `MATCH_AUTO` | 6 | 自动分类 | 基于机器学习模型自动预测 |

### 2.2 匹配参数配置

每个匹配规则包含以下可配置参数（[models.py](file:///d:/fz/0601/solo-dogfeeding/code/27-paperless-ngx/src/documents/models.py#L65-L76)）：

```python
name = models.CharField(max_length=128)           # 规则名称
match = models.CharField(max_length=256, blank=True)  # 匹配字符串
matching_algorithm = models.PositiveSmallIntegerField(
    choices=MATCHING_ALGORITHMS,
    default=MATCH_ANY,
)
is_insensitive = models.BooleanField(default=True)  # 是否大小写不敏感
```

### 2.3 可配置分类实体

支持自动分类的元数据类型均继承自 `MatchingModel`：

- **Correspondent**（联系人）- [models.py](file:///d:/fz/0601/solo-dogfeeding/code/27-paperless-ngx/src/documents/models.py#L96-L100)
- **Tag**（标签）- [models.py](file:///d:/fz/0601/solo-dogfeeding/code/27-paperless-ngx/src/documents/models.py#L102-L138)
- **DocumentType**（文档类型）- [models.py](file:///d:/fz/0601/solo-dogfeeding/code/27-paperless-ngx/src/documents/models.py#L141-L144)
- **StoragePath**（存储路径）- [models.py](file:///d:/fz/0601/solo-dogfeeding/code/27-paperless-ngx/src/documents/models.py#L147-L154)

---

## 三、规则命中流程

### 3.1 核心匹配函数

规则匹配的核心逻辑在 [matching.py](file:///d:/fz/0601/solo-dogfeeding/code/27-paperless-ngx/src/documents/matching.py#L169-L263) 的 `matches()` 函数中实现。

#### 匹配执行流程：

```
输入: matching_model (规则), document (文档)
输出: True/False (是否匹配)

1. 获取文档有效内容: document.get_effective_content()
2. 检查匹配字符串是否为空，为空返回 False
3. 根据 is_insensitive 设置正则表达式标志
4. 根据 matching_algorithm 执行相应匹配逻辑
5. 匹配成功时记录日志原因
```

### 3.2 各算法具体实现

#### 3.2.1 MATCH_ALL（全词匹配）
[matches() - L184-L198](file:///d:/fz/0601/solo-dogfeeding/code/27-paperless-ngx/src/documents/matching.py#L184-L198)

```python
for word in _split_match(matching_model):
    search_result = re.search(rf"\b{word}\b", document_content, flags=search_flags)
    if not search_result:
        return False
return True
```

**关键点**：
- 使用 `_split_match()` 解析匹配字符串，支持引号分组
- 每个词独立进行单词边界匹配（`\b`）
- 所有词都必须匹配才返回 True

#### 3.2.2 MATCH_ANY（任意词匹配）
[matches() - L200-L205](file:///d:/fz/0601/solo-dogfeeding/code/27-paperless-ngx/src/documents/matching.py#L200-L205)

```python
for word in _split_match(matching_model):
    if re.search(rf"\b{word}\b", document_content, flags=search_flags):
        return True
return False
```

**关键点**：
- 只要有一个词匹配就返回 True
- 支持引号分组（如 `"hello world"` 作为整体匹配）

#### 3.2.3 MATCH_LITERAL（精确匹配）
[matches() - L207-L221](file:///d:/fz/0601/solo-dogfeeding/code/27-paperless-ngx/src/documents/matching.py#L207-L221)

```python
result = bool(
    re.search(
        rf"\b{re.escape(matching_model.match)}\b",
        document_content,
        flags=search_flags,
    ),
)
```

**关键点**：
- 使用 `re.escape()` 转义特殊字符
- 整个匹配字符串作为整体进行单词边界匹配

#### 3.2.4 MATCH_REGEX（正则表达式）
[matches() - L223-L236](file:///d:/fz/0601/solo-dogfeeding/code/27-paperless-ngx/src/documents/matching.py#L223-L236)

```python
match = safe_regex_search(
    matching_model.match,
    document_content,
    flags=search_flags,
)
return bool(match)
```

**关键点**：
- 使用 `safe_regex_search()` 防止 ReDoS 攻击
- 匹配成功时记录匹配的具体字符串

#### 3.2.5 MATCH_FUZZY（模糊匹配）
[matches() - L238-L256](file:///d:/fz/0601/solo-dogfeeding/code/27-paperless-ngx/src/documents/matching.py#L238-L256)

```python
from rapidfuzz import fuzz

match = re.sub(r"[^\w\s]", "", matching_model.match)
text = re.sub(r"[^\w\s]", "", document_content)
if matching_model.is_insensitive:
    match = match.lower()
    text = text.lower()
if fuzz.partial_ratio(match, text, score_cutoff=90):
    return True
```

**关键点**：
- 使用 `rapidfuzz` 库的 `partial_ratio` 算法
- 匹配阈值固定为 90 分
- 匹配前移除标点符号

#### 3.2.6 MATCH_AUTO（自动分类）
[matches() - L258-L260](file:///d:/fz/0601/solo-dogfeeding/code/27-paperless-ngx/src/documents/matching.py#L258-L260)

```python
elif matching_model.matching_algorithm == MatchingModel.MATCH_AUTO:
    # this is done elsewhere.
    return False
```

**关键点**：
- `matches()` 函数中直接返回 False
- 实际匹配由机器学习分类器在 `match_*` 系列函数中完成

### 3.3 匹配字符串解析

`_split_match()` 函数 ([matching.py](file:///d:/fz/0601/solo-dogfeeding/code/27-paperless-ngx/src/documents/matching.py#L266-L282)) 负责解析匹配字符串：

```python
# 示例解析:
'  some random  words "with   quotes  " and   spaces'
=>
["some", "random", "words", "with\\s+quotes", "and", "spaces"]
```

**解析规则**：
1. 用正则表达式提取词或引号分组
2. 规范化空格（多个空格替换为 `\s+`）
3. 转义特殊正则字符

### 3.4 分类匹配集合函数

针对不同元数据类型，提供了独立的匹配函数，结合规则匹配和机器学习预测：

#### match_correspondents()
[matching.py - L47-L76](file:///d:/fz/0601/solo-dogfeeding/code/27-paperless-ngx/src/documents/matching.py#L47-L76)

```python
def match_correspondents(document, classifier, user=None):
    pred_id = classifier.predict_correspondent(document.suggestion_content) if classifier else None
    return list(filter(
        lambda o: matches(o, document) or (
            o.pk == pred_id and o.matching_algorithm == MatchingModel.MATCH_AUTO
        ),
        correspondents,
    ))
```

**匹配逻辑（OR 关系）**：
1. 传统规则匹配成功 (`matches(o, document)`)
2. **或者** 机器学习预测命中且算法为 `MATCH_AUTO`

相同的逻辑适用于：
- `match_document_types()` - [L79-L107](file:///d:/fz/0601/solo-dogfeeding/code/27-paperless-ngx/src/documents/matching.py#L79-L107)
- `match_tags()` - [L110-L134](file:///d:/fz/0601/solo-dogfeeding/code/27-paperless-ngx/src/documents/matching.py#L110-L134)
- `match_storage_paths()` - [L137-L166](file:///d:/fz/0601/solo-dogfeeding/code/27-paperless-ngx/src/documents/matching.py#L137-L166)

---

## 四、元数据提取

### 4.1 文档内容来源

#### 4.1.1 get_effective_content()
[models.py - L363-L400](file:///d:/fz/0601/solo-dogfeeding/code/27-paperless-ngx/src/documents/models.py#L363-L400)

返回文档的有效内容：
- 根文档：优先使用最新版本的内容
- 版本文档：使用自身内容
- 额外拼接：`{content} {archive_serial_number} {correspondent} {title}`

#### 4.1.2 suggestion_content
[models.py - L402-L428](file:///d:/fz/0601/solo-dogfeeding/code/27-paperless-ngx/src/documents/models.py#L402-L428)

用于分类建议的文档文本，针对大文档进行优化：
- 内容长度 ≤ 1,200,000 字符：返回全部内容
- 内容长度 > 1,200,000 字符：取前 800,000 字符 + 后 200,000 字符

### 4.2 文本预处理

`DocumentClassifier.preprocess_content()` ([classifier.py - L487-L516](file:///d:/fz/0601/solo-dogfeeding/code/27-paperless-ngx/src/documents/classifier.py#L487-L516)) 对文档内容进行预处理：

#### 基础预处理（始终执行）：
```python
# 转小写 + 提取单词 + 规范化空格
content = " ".join(match.group().lower() for match in RE_WORD.finditer(content))
```

#### 高级文本处理（NLTK 启用时）：
```python
if ADVANCED_TEXT_PROCESSING_ENABLED:
    words = word_tokenize(content, language=settings.NLTK_LANGUAGE)
    content = stem_and_skip_stop_words(words)
```

**高级处理包含**：
1. **分词**：使用 NLTK `word_tokenize`
2. **停用词过滤**：移除常见停用词（如 "the", "a", "is" 等）
3. **词干还原**：使用 `SnowballStemmer` 将词还原为词干（如 "amazement" → "amaz"）
4. **词干缓存**：LRU 缓存（10,000 条目）共享于多个 worker

### 4.3 特征向量化

使用 `CountVectorizer` 将文本转换为特征向量 ([classifier.py - L335-L343](file:///d:/fz/0601/solo-dogfeeding/code/27-paperless-ngx/src/documents/classifier.py#L335-L343))：

```python
self.data_vectorizer = CountVectorizer(
    analyzer="word",
    ngram_range=(1, 2),      # 提取 1-gram 和 2-gram 特征
    min_df=0.01,             # 忽略出现在少于 1% 文档中的词
)
```

**向量化缓存**：
- 缓存键：`sha256(content + 版本 + NLTK 配置 + 向量化器哈希)`
- 缓存时长：5 分钟
- 缓存位置：`read-cache`

---

## 五、分类决策机制

### 5.1 机器学习分类器

#### 5.1.1 分类器架构

`DocumentClassifier` 使用多层感知器（MLP）神经网络：

| 分类目标 | 分类器类型 | 说明 |
|---------|-----------|------|
| 标签 | `MultiLabelBinarizer` + `MLPClassifier` | 多标签分类 |
| 联系人 | `MLPClassifier` | 多分类 |
| 文档类型 | `MLPClassifier` | 多分类 |
| 存储路径 | `MLPClassifier` | 多分类 |

特殊情况：只有一个标签时，退化为二分类（`LabelBinarizer`）。

#### 5.1.2 模型训练流程

`train()` 方法 ([classifier.py - L221-L427](file:///d:/fz/0601/solo-dogfeeding/code/27-paperless-ngx/src/documents/classifier.py#L221-L427))：

**步骤 1：收集训练数据**
- 查询非收件箱文档
- 提取 `MATCH_AUTO` 类型的标签、联系人、文档类型、存储路径
- 计算训练数据哈希（用于判断是否需要重新训练）

**步骤 2：判断是否需要重新训练**
```python
if (self.last_doc_change_time >= latest_doc_change
    and self.last_auto_type_hash == hasher.digest()):
    return False  # 无需重新训练
```

**步骤 3：特征向量化**
- 使用 `CountVectorizer` 处理所有文档内容

**步骤 4：训练分类器**
- 对每个分类目标独立训练 MLP 模型
- 模型参数：`MLPClassifier(tol=0.01)`

**步骤 5：保存训练状态**
- 保存模型文件（HMAC 签名保护）
- 缓存训练元数据（50 分钟）

#### 5.1.3 模型文件格式

模型文件使用 HMAC-SHA256 签名保护 ([classifier.py - L136-L141](file:///d:/fz/0601/solo-dogfeeding/code/27-paperless-ngx/src/documents/classifier.py#L136-L141))：

```
[HMAC 签名 (32 字节)][Pickle 序列化数据]
```

序列化内容包括：
- 格式版本号（当前 v10）
- 最后文档变更时间
- 自动类型哈希
- 特征向量化器
- 各分类器模型

### 5.2 分类预测实现

#### 5.2.1 predict_correspondent()
[classifier.py - L536-L545](file:///d:/fz/0601/solo-dogfeeding/code/27-paperless-ngx/src/documents/classifier.py#L536-L545)

```python
def predict_correspondent(self, content: str) -> int | None:
    if self.correspondent_classifier:
        X = self._vectorize(content)
        correspondent_id = self.correspondent_classifier.predict(X)
        return correspondent_id if correspondent_id != -1 else None
    return None
```

#### 5.2.2 predict_tags()
[classifier.py - L558-L577](file:///d:/fz/0601/solo-dogfeeding/code/27-paperless-ngx/src/documents/classifier.py#L558-L577)

```python
def predict_tags(self, content: str) -> list[int]:
    if self.tags_classifier:
        X = self._vectorize(content)
        y = self.tags_classifier.predict(X)
        tags_ids = self.tags_binarizer.inverse_transform(y)[0]
        # 处理多标签和二分类的不同情况
        return list(tags_ids) if ... else []
    return []
```

### 5.3 分类决策函数

#### 5.3.1 set_correspondent()
[handlers.py - L93-L151](file:///d:/fz/0601/solo-dogfeeding/code/27-paperless-ngx/src/documents/signals/handlers.py#L93-L151)

**参数说明**：
- `replace`：是否覆盖已有值
- `use_first`：多匹配时的策略
- `dry_run`：是否仅计算不保存

**决策流程**：
```
1. 如果文档已有 correspondent 且 replace=False → 返回 None
2. 调用 matching.match_correspondents() 获取候选列表
3. 处理多匹配情况：
   - use_first=True → 选择第一个匹配
   - use_first=False → 不分配，返回 None
4. 非 dry_run 时保存到数据库
```

相同结构的决策函数：
- `set_document_type()` - [L154-L212](file:///d:/fz/0601/solo-dogfeeding/code/27-paperless-ngx/src/documents/signals/handlers.py#L154-L212)
- `set_storage_path()` - [L280-L338](file:///d:/fz/0601/solo-dogfeeding/code/27-paperless-ngx/src/documents/signals/handlers.py#L280-L338)

#### 5.3.2 set_tags()（特殊处理）
[handlers.py - L215-L277](file:///d:/fz/0601/solo-dogfeeding/code/27-paperless-ngx/src/documents/signals/handlers.py#L215-L277)

标签分类有特殊的替换逻辑：

```python
if replace:
    # 移除自动匹配的标签，但保留：
    # 1. 收件箱标签 (is_inbox_tag=True)
    # 2. 手动添加的标签 (match="" 且非 MATCH_AUTO)
    Document.tags.through.objects.filter(document=document).exclude(
        Q(tag__is_inbox_tag=True),
    ).exclude(
        Q(tag__match="") & ~Q(tag__matching_algorithm=Tag.MATCH_AUTO),
    ).delete()

# 添加新匹配的标签
matched_tags = matching.match_tags(document, classifier)
tags_to_add = set(matched_tags) - current_tags
document.add_nested_tags(tags_to_add)
```

---

## 六、冲突处理机制

### 6.1 多匹配冲突处理

当多个规则同时匹配时，处理策略由 `use_first` 参数控制：

| 元数据类型 | 默认策略 | 冲突处理代码位置 |
|-----------|---------|-----------------|
| 联系人 | `use_first=True` | [handlers.py - L128-L141](file:///d:/fz/0601/solo-dogfeeding/code/27-paperless-ngx/src/documents/signals/handlers.py#L128-L141) |
| 文档类型 | `use_first=True` | [handlers.py - L189-L202](file:///d:/fz/0601/solo-dogfeeding/code/27-paperless-ngx/src/documents/signals/handlers.py#L189-L202) |
| 存储路径 | `use_first=True` | [handlers.py - L315-L328](file:///d:/fz/0601/solo-dogfeeding/code/27-paperless-ngx/src/documents/signals/handlers.py#L315-L328) |
| 标签 | 全部添加 | [handlers.py - L266-L268](file:///d:/fz/0601/solo-dogfeeding/code/27-paperless-ngx/src/documents/signals/handlers.py#L266-L268) |

#### 冲突处理示例（联系人）：
```python
if potential_count > 1:
    if use_first:
        logger.debug(f"Detected {potential_count} potential correspondents, "
                     f"so we've opted for {selected}")
    else:
        logger.debug(f"Detected {potential_count} potential correspondents, "
                     f"not assigning any correspondent")
        return None
```

### 6.2 规则与自动分类的优先级

规则匹配和自动分类是 **OR 关系**，没有显式优先级：

```python
# matching.py 中的匹配逻辑
lambda o: (
    matches(o, document)  # 规则匹配
    or (
        o.pk == pred_id and o.matching_algorithm == MatchingModel.MATCH_AUTO
        # 自动分类预测
    )
)
```

**注意**：如果一个实体同时配置了非 AUTO 匹配算法，规则匹配会优先被检查，但实际上两者是 OR 关系，任一满足即匹配。

### 6.3 已有值保护

默认情况下（`replace=False`），不会覆盖已有的元数据值：

```python
if document.correspondent and not replace:
    return None
```

这意味着：
- 用户手动设置的值会被保留
- 只有未设置的字段才会被自动填充
- `document_retagger` 命令会使用 `replace=True` 进行强制重新分类

---

## 七、AI 分类扩展

### 7.1 LLM 分类流程

`get_ai_document_classification()` ([ai_classifier.py](file:///d:/fz/0601/solo-dogfeeding/code/27-paperless-ngx/src/paperless_ai/ai_classifier.py#L88-L102)) 提供 AI 辅助分类：

```python
def get_ai_document_classification(document, user=None):
    ai_config = AIConfig()
    prompt = build_prompt_with_rag(document, user) if ai_config.llm_embedding_backend \
             else build_prompt_without_rag(document)
    client = AIClient()
    result = client.run_llm_query(prompt)
    return parse_ai_response(result)
```

### 7.2 RAG 增强上下文

`get_context_for_document()` ([ai_classifier.py](file:///d:/fz/0601/solo-dogfeeding/code/27-paperless-ngx/src/paperless_ai/ai_classifier.py#L49-L74)) 检索相似文档提供上下文：

```python
similar_docs = query_similar_documents(document=doc, ...)[:max_docs]
context_blocks = []
for similar in similar_docs:
    text = similar.content[:1000] or ""
    title = similar.title or similar.filename or "Untitled"
    context_blocks.append(f"TITLE: {title}\n{text}")
```

### 7.3 AI 结果匹配

`paperless_ai/matching.py` 提供 AI 返回名称与系统实体的匹配：

- 精确匹配优先
- 模糊匹配备选（阈值 0.8，使用 `difflib.get_close_matches`）
- 归一化处理：小写、去标点、去空格

```python
# _match_names_to_queryset() 核心逻辑
if target in object_names:
    # 精确匹配
else:
    matches = difflib.get_close_matches(target, object_names, n=1, cutoff=0.8)
    # 模糊匹配
```

---

## 八、完整分类流程

### 8.1 文档消费时的分类流程

```
文档消费流程 (consumer.py):
│
├─→ 1. 前置检查 (ConsumerPreflightPlugin)
├─→ 2. ASN 检查
├─→ 3. 分页整理 (CollatePlugin)
├─→ 4. 条形码识别 (BarcodePlugin)
├─→ 5. 工作流触发 (WorkflowTriggerPlugin)
│   └─→ 消费时工作流匹配，预分配元数据
└─→ 6. 核心消费 (ConsumerPlugin)
    ├─→ 解析文档，提取内容
    ├─→ 解析日期
    ├─→ 保存文档到数据库
    ├─→ 加载分类器: classifier = load_classifier()
    └─→ 发送 document_consumption_finished 信号
        │
        ├─→ add_inbox_tags()        # 添加收件箱标签
        ├─→ set_correspondent()     # 自动分配联系人
        ├─→ set_document_type()     # 自动分配文档类型
        ├─→ set_tags()              # 自动分配标签
        ├─→ set_storage_path()      # 自动分配存储路径
        ├─→ add_to_index()          # 添加到搜索索引
        ├─→ run_workflows_added()   # 运行文档添加工作流
        └─→ add_or_update_document_in_llm_index()  # 更新 LLM 索引
```

### 8.2 消费流程中的关键代码位置

- 分类器加载：[consumer.py - L570-L576](file:///d:/fz/0601/solo-dogfeeding/code/27-paperless-ngx/src/documents/consumer.py#L570-L576)
- 信号发送：[consumer.py - L658-L666](file:///d:/fz/0601/solo-dogfeeding/code/27-paperless-ngx/src/documents/consumer.py#L658-L666)

### 8.3 重新标记流程

`document_retagger` 命令使用 `replace=True` 强制重新分类：

```python
# set_correspondent, set_document_type, set_storage_path:
# replace=True, use_first=False
# 意味着：覆盖已有值，但多匹配时不分配（避免错误）

# set_tags:
# replace=True
# 意味着：移除所有自动标签，重新分配
```

---

## 九、关键技术要点总结

### 9.1 匹配条件速查表

| 匹配算法 | 适用场景 | 性能 | 精确度 |
|---------|---------|------|-------|
| MATCH_ANY | 简单关键词过滤 | 高 | 中 |
| MATCH_ALL | 多条件组合过滤 | 中 | 高 |
| MATCH_LITERAL | 精确短语匹配 | 高 | 高 |
| MATCH_REGEX | 复杂模式匹配 | 低 | 高 |
| MATCH_FUZZY | OCR 错误容错 | 中 | 中 |
| MATCH_AUTO | 智能自动分类 | 中 | 取决于训练数据 |

### 9.2 性能优化点

1. **向量化缓存**：5 分钟缓存，避免重复计算
2. **词干缓存**：10,000 条目 LRU 缓存，跨 worker 共享
3. **训练条件判断**：基于文档变更时间和数据哈希，避免不必要的重训练
4. **大文档截断**：超大文档仅取首尾部分内容

### 9.3 安全机制

1. **HMAC 签名**：模型文件防篡改
2. **安全正则**：`safe_regex_search` 防止 ReDoS 攻击
3. **权限过滤**：匹配时考虑用户权限（`get_objects_for_user_owner_aware`）

---

## 十、代码引用汇总

| 功能模块 | 核心文件 | 关键行范围 |
|---------|---------|-----------|
| 匹配算法定义 | [models.py](file:///d:/fz/0601/solo-dogfeeding/code/27-paperless-ngx/src/documents/models.py) | [L46-L93](file:///d:/fz/0601/solo-dogfeeding/code/27-paperless-ngx/src/documents/models.py#L46-L93) |
| 规则匹配实现 | [matching.py](file:///d:/fz/0601/solo-dogfeeding/code/27-paperless-ngx/src/documents/matching.py) | [L169-L263](file:///d:/fz/0601/solo-dogfeeding/code/27-paperless-ngx/src/documents/matching.py#L169-L263) |
| 分类器训练 | [classifier.py](file:///d:/fz/0601/solo-dogfeeding/code/27-paperless-ngx/src/documents/classifier.py) | [L221-L427](file:///d:/fz/0601/solo-dogfeeding/code/27-paperless-ngx/src/documents/classifier.py#L221-L427) |
| 分类预测 | [classifier.py](file:///d:/fz/0601/solo-dogfeeding/code/27-paperless-ngx/src/documents/classifier.py) | [L536-L588](file:///d:/fz/0601/solo-dogfeeding/code/27-paperless-ngx/src/documents/classifier.py#L536-L588) |
| 分类决策处理 | [handlers.py](file:///d:/fz/0601/solo-dogfeeding/code/27-paperless-ngx/src/documents/signals/handlers.py) | [L93-L338](file:///d:/fz/0601/solo-dogfeeding/code/27-paperless-ngx/src/documents/signals/handlers.py#L93-L338) |
| 信号连接配置 | [apps.py](file:///d:/fz/0601/solo-dogfeeding/code/27-paperless-ngx/src/documents/apps.py) | [L24-L33](file:///d:/fz/0601/solo-dogfeeding/code/27-paperless-ngx/src/documents/apps.py#L24-L33) |
| 消费触发分类 | [consumer.py](file:///d:/fz/0601/solo-dogfeeding/code/27-paperless-ngx/src/documents/consumer.py) | [L570-L666](file:///d:/fz/0601/solo-dogfeeding/code/27-paperless-ngx/src/documents/consumer.py#L570-L666) |
| AI 分类扩展 | [ai_classifier.py](file:///d:/fz/0601/solo-dogfeeding/code/27-paperless-ngx/src/paperless_ai/ai_classifier.py) | [L88-L102](file:///d:/fz/0601/solo-dogfeeding/code/27-paperless-ngx/src/paperless_ai/ai_classifier.py#L88-L102) |
| AI 名称匹配 | [matching.py](file:///d:/fz/0601/solo-dogfeeding/code/27-paperless-ngx/src/paperless_ai/matching.py) | [L61-L93](file:///d:/fz/0601/solo-dogfeeding/code/27-paperless-ngx/src/paperless_ai/matching.py#L61-L93) |
