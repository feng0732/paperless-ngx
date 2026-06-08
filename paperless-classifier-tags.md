# Paperless-ngx Classifier 训练与自动 Tag 模型代码分析

## 一、整体架构概览

Paperless-ngx 采用 **双轨分类体系**，包含两套独立的分类器：

| 分类器类型 | 实现位置 | 技术方案 | 适用场景 |
|---|---|---|---|
| 传统 ML 分类器 | [classifier.py](file:///d:/fz/0601/solo-dogfeeding/code/110-paperless-ngx/src/documents/classifier.py) | scikit-learn MLP + CountVectorizer | 离线训练，自动匹配 Tag/Correspondent/DocumentType/StoragePath |
| AI/LLM 分类器 | [ai_classifier.py](file:///d:/fz/0601/solo-dogfeeding/code/110-paperless-ngx/src/paperless_ai/ai_classifier.py) | LLM + RAG (可选向量检索) | 实时调用大模型进行语义理解 |

**核心数据流：**

```
┌──────────────────────────────────────────────────────────────────┐
│                         训练阶段                                   │
│  文档(已有标签) → 文本预处理 → 特征向量化 → MLP模型训练 → 持久化   │
└──────────────────────────────────────────────────────────────────┘
                              ↓  MODEL_FILE (pickle + HMAC)
┌──────────────────────────────────────────────────────────────────┐
│                         推断阶段                                   │
│  新文档 → 文本预处理 → 加载分类器 → 预测 → 信号分发 → 自动打标签     │
└──────────────────────────────────────────────────────────────────┘
```

---

## 二、核心数据模型：MatchingModel 的匹配算法

所有可被自动匹配的实体（Tag、DocumentType、Correspondent、StoragePath）都继承自 [MatchingModel](file:///d:/fz/0601/solo-dogfeeding/code/110-paperless-ngx/src/documents/models.py#L46-L77)，其核心字段 `matching_algorithm` 决定了匹配方式：

```python
class MatchingModel(ModelWithOwner):
    MATCH_NONE = 0       # 不匹配
    MATCH_ANY = 1        # 任意关键词匹配
    MATCH_ALL = 2        # 全部关键词匹配
    MATCH_LITERAL = 3    # 精确字符串匹配
    MATCH_REGEX = 4      # 正则表达式匹配
    MATCH_FUZZY = 5      # 模糊匹配
    MATCH_AUTO = 6       # 自动（机器学习分类器）
```

**关键设计：** 只有 `matching_algorithm == MATCH_AUTO` 的实体才会被纳入分类器的训练和预测范围。非 AUTO 的实体走 [matching.py](file:///d:/fz/0601/solo-dogfeeding/code/110-paperless-ngx/src/documents/matching.py) 中 `matches()` 函数的规则匹配路径。

---

## 三、训练数据准备流程

训练入口由 Celery 定时任务或管理命令触发：

### 3.1 触发入口

1. **定时任务**：Celery Beat 每小时触发，配置在 [custom.py](file:///d:/fz/0601/solo-dogfeeding/code/110-paperless-ngx/src/paperless/settings/custom.py#L91-L101)
   ```
   "5 */1 * * *"  # 每小时第5分钟执行
   ```

2. **管理命令**：[document_create_classifier.py](file:///d:/fz/0601/solo-dogfeeding/code/110-paperless-ngx/src/documents/management/commands/document_create_classifier.py) → 调用 [train_classifier](file:///d:/fz/0601/solo-dogfeeding/code/110-paperless-ngx/src/documents/tasks.py#L88-L120)

3. **API 触发**：通过 `TasksViewSet` 手动触发

### 3.2 任务调度层

[train_classifier](file:///d:/fz/0601/solo-dogfeeding/code/110-paperless-ngx/src/documents/tasks.py#L88-L120) 任务首先做**前置检查**：

```python
# 如果没有任何实体配置为 MATCH_AUTO，则直接跳过并删除旧模型
if (not Tag.objects.filter(matching_algorithm=Tag.MATCH_AUTO).exists()
    and not DocumentType.objects.filter(matching_algorithm=Tag.MATCH_AUTO).exists()
    ...):
    if settings.MODEL_FILE.exists():
        settings.MODEL_FILE.unlink()
    return "No automatic matching items, not training"
```

### 3.3 训练数据提取

[DocumentClassifier.train()](file:///d:/fz/0601/solo-dogfeeding/code/110-paperless-ngx/src/documents/classifier.py#L221-L427) 中的核心步骤：

**Step 1: 查询训练文档**

```python
docs_queryset = (
    Document.objects.exclude(tags__is_inbox_tag=True)  # 排除收件箱文档
    .select_related("document_type", "correspondent", "storage_path")
    .prefetch_related("tags")
    .order_by("pk")
)
```

**Step 2: 构建标签向量 + 变化检测 Hash**

对每个文档，提取四个维度的标签，同时用 SHA256 累积所有 AUTO 类型实体的主键，用于后续判断是否需要重训练：

```python
hasher = sha256()
for doc in docs_queryset:
    # DocumentType: 单标签，-1 表示无
    y = dt.pk if (dt and dt.matching_algorithm == MATCH_AUTO) else -1
    hasher.update(y.to_bytes(4, "little", signed=True))
    labels_document_type.append(y)

    # Correspondent: 同上
    # StoragePath: 同上

    # Tags: 多标签，只收集 MATCH_AUTO 的标签
    tags = list(doc.tags.filter(matching_algorithm=MATCH_AUTO)
                    .order_by("pk").values_list("pk", flat=True))
    for tag in tags:
        hasher.update(tag.to_bytes(4, "little", signed=True))
    labels_tags.append(tags)
```

**Step 3: 重训练必要性检测（增量优化）**

```python
latest_doc_change = docs_queryset.latest("modified").modified
if (self.last_doc_change_time is not None
    and self.last_doc_change_time >= latest_doc_change
) and self.last_auto_type_hash == hasher.digest():
    # 文档无变化 + AUTO 实体配置无变化 → 跳过训练
    logger.info("No updates since last training")
    return False
```

这是一个重要的性能优化：**只有当文档内容或 AUTO 实体配置发生变化时才会真正训练。**

### 3.4 文本预处理

[preprocess_content()](file:///d:/fz/0601/solo-dogfeeding/code/110-paperless-ngx/src/documents/classifier.py#L487-L516) 处理管道：

```
原始文本
  ↓
1. 正则提取单词 → 转小写 → [\w]+ 词边界匹配
  ↓
2. 可选 NLTK 高级处理（需配置 NLTK_ENABLED + NLTK_LANGUAGE）:
   ├─ word_tokenize 分词
   ├─ SnowballStemmer 词干提取 (amazement → amaz)
   └─ 停用词过滤 (stopwords)
  ↓
处理后文本
```

词干结果使用 [StoredLRUCache](file:///d:/fz/0601/solo-dogfeeding/code/110-paperless-ngx/src/documents/caching.py) 做 LRU 缓存（容量 10000），通过 Redis 跨 worker 共享。

---

## 四、模型训练与持久化

### 4.1 特征向量化

使用 scikit-learn `CountVectorizer`：

```python
self.data_vectorizer = CountVectorizer(
    analyzer="word",
    ngram_range=(1, 2),    # 1-gram + 2-gram 组合
    min_df=0.01,           # 词至少出现在 1% 的文档中
)
data_vectorized = self.data_vectorizer.fit_transform(content_generator())
```

注意：`stop_words_` 属性在训练后被显式置为 `None`，以减小序列化体积。

### 4.2 多分类器并行训练

Paperless-ngx 训练**四个独立的 MLP 分类器**，各自负责一个维度：

| 分类器 | 标签类型 | Binarizer | 模型 |
|---|---|---|---|
| `tags_classifier` | 多标签 (Multi-label) | `MultiLabelBinarizer` (单标签时退化 `LabelBinarizer`) | `MLPClassifier(tol=0.01)` |
| `correspondent_classifier` | 多分类 (Multi-class) | 无需 (sklearn 原生支持) | `MLPClassifier(tol=0.01)` |
| `document_type_classifier` | 多分类 | 无需 | `MLPClassifier(tol=0.01)` |
| `storage_path_classifier` | 多分类 | 无需 | `MLPClassifier(tol=0.01)` |

**Tag 分类器的特殊处理**（单标签场景）：
```python
if num_tags == 1:
    # 只有一个 AUTO tag 时，退化到二分类：有/无该标签
    labels_tags = [label[0] if len(label) == 1 else -1 for label in labels_tags]
    self.tags_binarizer = LabelBinarizer()
```

### 4.3 模型持久化与安全

[save()](file:///d:/fz/0601/solo-dogfeeding/code/110-paperless-ngx/src/documents/classifier.py#L196-L219) 方法：

```
序列化对象组成:
(FORMAT_VERSION, last_doc_change_time, last_auto_type_hash,
 data_vectorizer, tags_binarizer, tags_classifier,
 correspondent_classifier, document_type_classifier, storage_path_classifier)
  ↓
pickle.dumps() → 计算 HMAC-SHA256(使用 SECRET_KEY) → [签名(32字节) | 数据]
  ↓
原子写入：先写 .pickle.part → rename 覆盖正式文件
```

**版本管理**：当前 `FORMAT_VERSION = 10`，历史版本演进记录在注释中。

### 4.4 模型加载与兼容性

[load()](file:///d:/fz/0601/solo-dogfeeding/code/110-paperless-ngx/src/documents/classifier.py#L143-L194) 中做多层校验：

1. **HMAC 签名校验**：防止模型文件被篡改
2. **FORMAT_VERSION 校验**：代码与模型版本不一致 → 触发重训练
3. **scikit-learn 版本校验**：检测 `InconsistentVersionWarning` → 视为不兼容

任何校验失败都会删除模型文件，触发下次重新训练。

---

## 五、文档归类（自动打标签）流程

### 5.1 触发时机：消费流程

[ConsumerPlugin.run()](file:///d:/fz/0601/solo-dogfeeding/code/110-paperless-ngx/src/documents/consumer.py#L408-L784) 是文档消费的核心：

```
消费流程时序:
1. 文件复制到临时目录
2. MIME 类型检测 → 选择 Parser
3. Parser.parse() → 提取文本 text、日期 date、缩略图
4. ★ 加载分类器: classifier = load_classifier()
5. transaction.atomic():
   a. _store() 保存 Document 到数据库
   b. ★ document_consumption_finished.send(classifier=classifier)
   c. 文件写入 originals/、thumbnails/、archive/
   d. document.save() 触发 post_save → 文件名重排
```

**关键优化**：分类器只加载一次，通过信号 `classifier` 参数传递给所有处理器，避免重复加载。

### 5.2 信号分发链

[apps.py](file:///d:/fz/0601/solo-dogfeeding/code/110-paperless-ngx/src/documents/apps.py#L10-L37) 中 `ready()` 注册的信号连接：

```
document_consumption_finished
  ├─ add_inbox_tags              # 添加收件箱标签
  ├─ set_correspondent           # 自动匹配通信者
  ├─ set_document_type           # 自动匹配文档类型
  ├─ set_tags                    # 自动匹配标签（多标签）
  ├─ set_storage_path            # 自动匹配存储路径
  ├─ add_to_index                # 添加到全文索引
  ├─ run_workflows_added         # 触发 DOCUMENT_ADDED 工作流
  └─ add_or_update_document_in_llm_index  # 更新 LLM 向量索引
```

### 5.3 匹配策略：规则 + 分类器 双轨合并

以 [match_tags()](file:///d:/fz/0601/solo-dogfeeding/code/110-paperless-ngx/src/documents/matching.py#L110-L134) 为例：

```python
def match_tags(document, classifier, user=None):
    # 1. 分类器预测（仅针对 MATCH_AUTO 的标签）
    predicted_tag_ids = classifier.predict_tags(document.suggestion_content) if classifier else []

    # 2. 返回合并结果：规则匹配 OR 分类器预测命中
    return list(filter(
        lambda o: (
            matches(o, document)                          # 规则匹配路径
            or (o.matching_algorithm == MATCH_AUTO       # 分类器路径
                and o.pk in predicted_tag_ids)
        ),
        tags
    ))
```

**规则匹配** (`matches()` 函数) 支持 ANY/ALL/LITERAL/REGEX/FUZZY，`MATCH_AUTO` 在规则路径直接返回 `False`。

### 5.4 分类器预测内部实现

以 [predict_tags()](file:///d:/fz/0601/solo-dogfeeding/code/110-paperless-ngx/src/documents/classifier.py#L558-L577) 为例：

```python
def predict_tags(self, content: str) -> list[int]:
    X = self._vectorize(content)                          # 文本向量化（带5分钟缓存）
    y = self.tags_classifier.predict(X)                   # MLP 预测
    tags_ids = self.tags_binarizer.inverse_transform(y)[0]  # 反编码为标签 ID 列表

    if type_of_target(y).startswith("multilabel"):
        return list(tags_ids)                              # 多标签场景
    elif type_of_target(y) == "binary" and tags_ids != -1:
        return [tags_ids]                                  # 单标签场景，有该标签
    else:
        return []                                          # 单标签场景，无该标签
```

**向量化缓存** (`_vectorize`)：使用 `read_cache`（Redis）缓存 5 分钟，key 包含内容 hash + 版本号 + NLTK 配置 + vectorizer hash。

### 5.5 标签写入策略

[set_tags()](file:///d:/fz/0601/solo-dogfeeding/code/110-paperless-ngx/src/documents/signals/handlers.py#L215-L277) 的写入逻辑：

- **replace=False**（默认消费流程）：仅追加新标签，不删除已有标签
- **replace=True**（retagger 覆盖模式）：先删除所有非收件箱、非手动的标签，再重新应用
- **保护机制**：收件箱标签 (`is_inbox_tag=True`) 和手动添加的标签 (`match=""` 且非 `MATCH_AUTO`) 永远不会被自动删除

---

## 六、批量重打标：document_retagger

除了消费时的自动匹配，还可通过 [document_retagger.py](file:///d:/fz/0601/solo-dogfeeding/code/110-paperless-ngx/src/documents/management/commands/document_retagger.py) 对历史文档批量应用分类器：

```bash
# 对所有文档重新打标签（覆盖），使用第一个匹配结果
python manage.py document_retagger -T -c -t -s -f --use-first

# 仅预览建议，不实际写入
python manage.py document_retagger -T --suggest
```

核心流程：遍历 Document QuerySet → 依次调用 `set_correspondent / set_document_type / set_tags / set_storage_path` → 统计变更 → 输出报表。

---

## 七、AI 分类器（可选增强）

[paperless_ai/ai_classifier.py](file:///d:/fz/0601/solo-dogfeeding/code/110-paperless-ngx/src/paperless_ai/ai_classifier.py) 提供了基于 LLM 的增强分类，与传统分类器是**互补关系**而非替代：

```
LLM 分类流程:
1. 构建 Prompt (文件名 + 内容前4000字符)
2. 可选 RAG: query_similar_documents() 检索相似文档作为上下文
3. AIClient.run_llm_query(prompt) → 调用外部 LLM API
4. parse_ai_response() → 解析 JSON 输出 (title/tags/correspondents/...)
```

与传统分类器的区别：
- **不需要训练**：直接调用大模型，靠语义理解
- **成本较高**：每次预测消耗 API token
- **用于前端"建议"功能**，而非消费时的自动匹配

---

## 八、关键文件索引

| 文件 | 作用 |
|---|---|
| [classifier.py](file:///d:/fz/0601/solo-dogfeeding/code/110-paperless-ngx/src/documents/classifier.py) | 传统 ML 分类器核心：训练/预测/序列化 |
| [matching.py](file:///d:/fz/0601/solo-dogfeeding/code/110-paperless-ngx/src/documents/matching.py) | 规则匹配 + 分类器预测的合并逻辑 |
| [tasks.py](file:///d:/fz/0601/solo-dogfeeding/code/110-paperless-ngx/src/documents/tasks.py) | Celery 任务：train_classifier、consume_file |
| [consumer.py](file:///d:/fz/0601/solo-dogfeeding/code/110-paperless-ngx/src/documents/consumer.py) | 文档消费流程，加载分类器并触发信号 |
| [signals/handlers.py](file:///d:/fz/0601/solo-dogfeeding/code/110-paperless-ngx/src/documents/signals/handlers.py) | 信号处理器：set_tags/set_correspondent 等 |
| [apps.py](file:///d:/fz/0601/solo-dogfeeding/code/110-paperless-ngx/src/documents/apps.py) | 信号连接注册 |
| [models.py](file:///d:/fz/0601/solo-dogfeeding/code/110-paperless-ngx/src/documents/models.py) | MatchingModel 基类、Document.suggestion_content |
| [ai_classifier.py](file:///d:/fz/0601/solo-dogfeeding/code/110-paperless-ngx/src/paperless_ai/ai_classifier.py) | LLM/RAG 分类器（可选增强） |
| [management/commands/document_create_classifier.py](file:///d:/fz/0601/solo-dogfeeding/code/110-paperless-ngx/src/documents/management/commands/document_create_classifier.py) | 手动训练命令 |
| [management/commands/document_retagger.py](file:///d:/fz/0601/solo-dogfeeding/code/110-paperless-ngx/src/documents/management/commands/document_retagger.py) | 批量重打标命令 |
| [settings/custom.py](file:///d:/fz/0601/solo-dogfeeding/code/110-paperless-ngx/src/paperless/settings/custom.py) | Celery Beat 定时训练配置 |

---

## 九、设计要点总结

1. **增量训练检测**：通过 `last_doc_change_time` + `last_auto_type_hash` 双重校验避免无效训练，节省算力。

2. **原子持久化 + 安全签名**：`.part` 临时文件 + rename 保证原子更新，HMAC 签名防止模型篡改。

3. **多级缓存**：向量化结果缓存 5 分钟、词干提取 LRU 缓存 10000 条，跨 worker 共享。

4. **规则 + ML 双轨并行**：MATCH_AUTO 走分类器，其余走规则匹配，结果取并集，两种机制互不干扰。

5. **信号解耦**：消费流程通过 `document_consumption_finished` 信号分发，新增匹配维度只需连接信号，无需修改消费主流程。

6. **软删除保护**：收件箱标签和手动添加的标签不会被自动系统覆盖删除，确保用户手动操作的权威性。
