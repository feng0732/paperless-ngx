# Paperless-ngx 自动分类机制深度分析

本文档重点分析自动分类的三个入口及其职责划分：`document_consumption_finished`、`document_updated`、`document_retagger`，并深入阐述规则命中、分类决策、多匹配冲突处理和已有值保护机制。

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
# consumer.py 约第570行
classifier = load_classifier()  # 预加载分类器

# 约第658行
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

**关键设计**：
- 分类器在信号发送前预加载，通过 `classifier` 参数传递给所有处理器，避免多次 I/O
- 分类处理器（2-5）的 `replace` 均为 `False`，不会覆盖用户手动设置的值
- 工作流（序号7）在分类之后执行，具有最终决定权，可覆盖分类结果

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

**与 `document_consumption_finished` 的本质区别**：
- `document_updated` **不连接** `set_correspondent`、`set_document_type`、`set_tags`、`set_storage_path`
- 更新文档时不会重新运行分类逻辑
- 如需对已有文档重新分类，必须使用 `document_retagger` 命令

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
| 工作流 | 执行 `run_workflows_added` | **不执行** |
| 收件箱标签 | 执行 `add_inbox_tags` | **不执行** |
| 搜索索引 | 执行 `add_to_index` | **不执行** |
| 分类目标 | 全部（联系人/类型/标签/路径） | 可选（`-c/-T/-t/-s`） |
| 文档范围 | 当前消费的单个文档 | 可选全部/收件箱/ID范围 |

**命令行参数与函数参数映射**：

```python
# document_retagger.py 约第278行
set_correspondent(
    None, document,
    classifier=classifier,
    replace=overwrite,    # --overwrite / -f
    use_first=use_first,  # --use-first
    dry_run=suggest,      # --suggest
)
```

**覆盖模式组合**：

| `--overwrite` | `--use-first` | 行为 |
|--------------|--------------|------|
| `False`（默认） | `False`（默认） | 不覆盖已有值；多匹配不分配 |
| `False` | `True` | 不覆盖已有值；多匹配取第一个 |
| `True` | `False` | 覆盖已有值；多匹配不分配 |
| `True` | `True` | 覆盖已有值；多匹配取第一个 |

---

## 二、规则命中与匹配算法

### 2.1 七种匹配算法

`MatchingModel` 在 `src/documents/models.py` 中定义了 7 种匹配算法：

| 枚举值 | 名称 | 匹配逻辑 | 核心实现 |
|-------|------|---------|---------|
| `MATCH_NONE = 0` | 不匹配 | 始终返回 False | - |
| `MATCH_ANY = 1` | 任意词 | 匹配字符串中任一词出现在文档中 | 单词边界 `\b` 匹配 |
| `MATCH_ALL = 2` | 全词 | 匹配字符串中所有词都出现在文档中 | 逐词 `\b` 匹配 |
| `MATCH_LITERAL = 3` | 精确匹配 | 整个匹配字符串作为整体精确匹配 | `re.escape()` + `\b` |
| `MATCH_REGEX = 4` | 正则表达式 | 正则表达式匹配 | `safe_regex_search()` |
| `MATCH_FUZZY = 5` | 模糊匹配 | 部分相似度匹配 | `rapidfuzz.fuzz.partial_ratio`，阈值 90 |
| `MATCH_AUTO = 6` | 自动分类 | 机器学习模型预测 | `matches()` 返回 False，由 `match_*` 函数处理 |

### 2.2 匹配参数

每个匹配规则（Tag / Correspondent / DocumentType / StoragePath）包含：

| 字段 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `match` | CharField(256) | `""` | 匹配字符串，空值不匹配 |
| `matching_algorithm` | PositiveSmallIntegerField | `MATCH_ANY` | 匹配算法选择 |
| `is_insensitive` | BooleanField | `True` | 是否大小写不敏感 |

### 2.3 核心匹配函数 `matches()`

实现位置：`src/documents/matching.py`

```python
def matches(matching_model: MatchingModel, document: Document):
    document_content = document.get_effective_content() or ""

    if not matching_model.match.strip():
        return False

    if matching_model.is_insensitive:
        search_flags = re.IGNORECASE

    # 根据算法分发...
    elif matching_model.matching_algorithm == MatchingModel.MATCH_AUTO:
        # this is done elsewhere.
        return False  # ← AUTO 在此直接返回 False
```

**关键设计**：`MATCH_AUTO` 在 `matches()` 中返回 `False`，自动预测由外层 `match_*` 函数独立处理。

### 2.4 匹配字符串解析 `_split_match()`

`src/documents/matching.py` 中的 `_split_match()` 将匹配字符串拆分为独立关键词：

```
输入: '  some random  words "with   quotes  " and   spaces'
输出: ["some", "random", "words", "with\s+quotes", "and", "spaces"]
```

- 引号内的词作为整体，内部空格转为 `\s+`
- 特殊正则字符被转义

---

## 三、规则匹配与自动预测的关联

### 3.1 组合匹配逻辑

`match_correspondents`、`match_document_types`、`match_tags`、`match_storage_paths` 四个函数使用相同的组合逻辑：

```python
# src/documents/matching.py — 以 match_correspondents 为例
def match_correspondents(document, classifier, user=None):
    pred_id = (
        classifier.predict_correspondent(document.suggestion_content)
        if classifier else None
    )

    return list(filter(
        lambda o: (
            matches(o, document)                              # 规则匹配
            or (
                o.pk == pred_id                                 # ML 预测命中
                and o.matching_algorithm == MatchingModel.MATCH_AUTO
                # 且该规则的算法为 AUTO
            )
        ),
        correspondents,
    ))
```

**OR 关系**：规则匹配成功 **或** 自动预测命中（且算法为 AUTO），任一满足即匹配。

### 3.2 匹配真值表

| 规则算法 | `matches()` 结果 | ML 预测命中 pk | 最终匹配 | 说明 |
|---------|-----------------|---------------|---------|------|
| `MATCH_ANY` | True | 任意 | **True** | 规则匹配成功，短路 OR |
| `MATCH_ANY` | False | 是 | **False** | 非 AUTO 规则，ML 预测无效 |
| `MATCH_AUTO` | False（恒） | 是 | **True** | ML 预测命中且算法为 AUTO |
| `MATCH_AUTO` | False（恒） | 否 | **False** | ML 预测未命中 |
| 其他算法 | True | 任意 | **True** | 规则匹配成功 |
| 其他算法 | False | 是 | **False** | 非 AUTO 规则，ML 预测无效 |

**核心结论**：ML 预测仅对 `matching_algorithm == MATCH_AUTO` 的规则生效。非 AUTO 规则完全依赖 `matches()` 的文本匹配结果。

### 3.3 四种元数据类型的匹配函数

| 函数 | 文件位置 | ML 预测函数 | 预测返回类型 |
|------|---------|------------|------------|
| `match_correspondents` | `src/documents/matching.py` | `predict_correspondent` | `int | None` |
| `match_document_types` | `src/documents/matching.py` | `predict_document_type` | `int | None` |
| `match_tags` | `src/documents/matching.py` | `predict_tags` | `list[int]` |
| `match_storage_paths` | `src/documents/matching.py` | `predict_storage_path` | `int | None` |

**标签的特殊性**：标签使用 `in` 判断（`o.pk in predicted_tag_ids`），因为标签预测返回列表；其余三个使用 `==` 判断（`o.pk == pred_id`），因为预测返回单个 ID 或 None。

### 3.4 ML 预测的实现

`src/documents/classifier.py` 中，预测值 `-1` 表示"无匹配"：

```python
def predict_correspondent(self, content: str) -> int | None:
    if self.correspondent_classifier:
        X = self._vectorize(content)
        correspondent_id = self.correspondent_classifier.predict(X)
        return correspondent_id if correspondent_id != -1 else None
    return None
```

---

## 四、分类决策与多匹配冲突处理

### 4.1 四个分类决策函数

所有分类决策函数在 `src/documents/signals/handlers.py` 中实现：

| 函数 | 参数签名 | 特殊处理 |
|------|---------|---------|
| `set_correspondent` | `replace=False, use_first=True, dry_run=False` | 多匹配取首 |
| `set_document_type` | `replace=False, use_first=True, dry_run=False` | 多匹配取首 |
| `set_tags` | `replace=False, dry_run=False` | 无 `use_first`，多标签全部添加 |
| `set_storage_path` | `replace=False, use_first=True, dry_run=False` | 多匹配取首 |

### 4.2 多匹配冲突决策逻辑

以 `set_correspondent` 为例：

```python
# src/documents/signals/handlers.py
potential_correspondents = matching.match_correspondents(document, classifier)
potential_count = len(potential_correspondents)
selected = potential_correspondents[0] if potential_correspondents else None

if potential_count > 1:
    if use_first:
        logger.debug(f"Detected {potential_count} potential correspondents, "
                     f"so we've opted for {selected}")
    else:
        logger.debug(f"Detected {potential_count} potential correspondents, "
                     f"not assigning any correspondent")
        return None
```

**决策流程**：
```
候选列表长度 N:
  N == 0 → 返回 None，不分配
  N == 1 → 返回该候选
  N > 1  →
    use_first=True  → 返回候选列表第一个
    use_first=False → 返回 None，不分配
```

**"第一个"的定义**：按 `MatchingModel.Meta.ordering = ("name",)` 字母顺序排序，而非匹配质量。

### 4.3 `use_first` 参数在不同场景下的值

| 场景 | `use_first` | 说明 |
|------|------------|------|
| 文档消费（`document_consumption_finished`） | `True` | 默认值，快速分配 |
| 重新标记默认（`document_retagger` 无 `--use-first`） | `False` | 保守策略，多匹配跳过 |
| 重新标记指定（`document_retagger --use-first`） | `True` | 强制取第一个 |

### 4.4 标签的特殊处理

标签支持多值，不存在"冲突"概念：

```python
# src/documents/signals/handlers.py
matched_tags = matching.match_tags(document, classifier)
tags_to_add = set(matched_tags) - current_tags  # 增量添加
```

所有匹配到的标签都会被添加，不需要 `use_first` 参数。

---

## 五、已有值冲突处理方式

### 5.1 已有值保护（`replace` 参数）

联系人、文档类型、存储路径三个单值属性使用相同的保护逻辑：

```python
# src/documents/signals/handlers.py — set_correspondent
if document.correspondent and not replace:
    return None  # 已有值且不覆盖，直接返回
```

| 元数据类型 | 冲突检查代码 | `replace=False` | `replace=True` |
|-----------|------------|----------------|---------------|
| 联系人 | `document.correspondent and not replace` | 保留原值 | 覆盖 |
| 文档类型 | `document.document_type and not replace` | 保留原值 | 覆盖 |
| 存储路径 | `document.storage_path and not replace` | 保留原值 | 覆盖 |

### 5.2 标签的覆盖逻辑

标签是多值属性，覆盖逻辑更复杂：

```python
# src/documents/signals/handlers.py — set_tags
if replace:
    # 移除标签，但保留以下两种：
    Document.tags.through.objects.filter(document=document).exclude(
        Q(tag__is_inbox_tag=True),              # 1. 收件箱标签
    ).exclude(
        Q(tag__match="") & ~Q(tag__matching_algorithm=Tag.MATCH_AUTO),
        # 2. 手动标签（match为空 且 非AUTO算法）
    ).delete()

# 添加新匹配标签
matched_tags = matching.match_tags(document, classifier)
tags_to_add = set(matched_tags) - current_tags
```

**标签保留规则**（`replace=True` 时）：

| 标签类型 | 条件 | 是否保留 |
|---------|------|---------|
| 收件箱标签 | `is_inbox_tag=True` | ✅ 保留 |
| 手动标签 | `match=""` 且 `matching_algorithm != MATCH_AUTO` | ✅ 保留 |
| 规则匹配标签 | `match != ""` 且算法非 AUTO | ❌ 删除 |
| 自动分类标签 | `matching_algorithm == MATCH_AUTO` | ❌ 删除 |

### 5.3 `dry_run` 参数

`dry_run=True` 时只计算不写入数据库，用于 `document_retagger --suggest` 预览模式：

```python
# src/documents/signals/handlers.py — set_correspondent
if (selected or replace) and not dry_run:
    document.correspondent = selected
    document.save(update_fields=("correspondent",))
```

### 5.4 工作流覆盖优先级

在 `document_consumption_finished` 信号中，工作流在分类之后执行：

```
执行顺序:
1. set_correspondent     → 自动分配联系人
2. set_document_type     → 自动分配文档类型
3. set_tags              → 自动分配标签
4. set_storage_path      → 自动分配存储路径
5. run_workflows_added   → 工作流可覆盖以上所有值
```

**工作流具有最终决定权**，可通过 Assignment Action 覆盖自动分类的结果。

---

## 六、ML 分类器训练与预测

### 6.1 训练数据收集

`src/documents/classifier.py` 的 `train()` 方法：

```python
# 仅使用非收件箱文档
docs_queryset = Document.objects.exclude(tags__is_inbox_tag=True)

# 仅收集 MATCH_AUTO 类型的元数据作为训练标签
dt = doc.document_type
if dt and dt.matching_algorithm == MatchingModel.MATCH_AUTO:
    y = dt.pk
```

**关键限制**：只有 `matching_algorithm == MATCH_AUTO` 的实体才会参与训练。手动匹配规则的实体对分类器不可见。

### 6.2 重训练条件判断

```python
# src/documents/classifier.py
if (self.last_doc_change_time >= latest_doc_change
    and self.last_auto_type_hash == hasher.digest()):
    return False  # 无需重训练
```

条件：文档最后修改时间未变化 **且** AUTO 类型的数据哈希未变化。

### 6.3 文本预处理

`src/documents/classifier.py` 的 `preprocess_content()`：

1. **基础处理**（始终执行）：转小写 + 提取单词（`RE_WORD`）
2. **高级处理**（`NLTK_ENABLED` 时）：分词 → 停用词过滤 → SnowballStemmer 词干还原

### 6.4 特征向量化

```python
self.data_vectorizer = CountVectorizer(
    analyzer="word",
    ngram_range=(1, 2),   # 1-gram + 2-gram
    min_df=0.01,           # 忽略频率低于 1% 的词
)
```

### 6.5 分类模型

| 目标 | 模型 | 特殊处理 |
|------|------|---------|
| 标签 | `MultiLabelBinarizer` + `MLPClassifier` | 单标签时退化为 `LabelBinarizer` |
| 联系人 | `MLPClassifier` | 多分类 |
| 文档类型 | `MLPClassifier` | 多分类 |
| 存储路径 | `MLPClassifier` | 多分类 |

---

## 七、代码引用汇总

| 功能模块 | 文件（仓库相对路径） |
|---------|-------------------|
| 信号连接配置 | `src/documents/apps.py` |
| 消费完成信号发送 | `src/documents/consumer.py` |
| 规则匹配实现 | `src/documents/matching.py` |
| 匹配算法定义 | `src/documents/models.py` |
| 分类器训练与预测 | `src/documents/classifier.py` |
| 分类决策函数 | `src/documents/signals/handlers.py` |
| 重新标记命令 | `src/documents/management/commands/document_retagger.py` |
| 批量更新信号发送 | `src/documents/tasks.py` |
| API 更新信号发送 | `src/documents/views.py` |
