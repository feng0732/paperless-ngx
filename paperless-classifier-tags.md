# Paperless-ngx Classifier 训练与自动 Tag 模型代码阅读笔记

> 可复核的代码阅读笔记。所有路径为仓库根目录相对路径，关键结论均标记了对应的源码位置以便交叉核对。

---

## 一、整体架构：两套分类系统的职责边界

Paperless-ngx 存在 **两套完全独立** 的分类/建议系统，切勿混淆。其中 MATCH_AUTO 传统分类器自身又有**两种运行模式**（自动写入 vs 仅建议），行为不同：

| 维度 | MATCH_AUTO 传统分类器 | paperless_ai LLM 建议接口 |
|---|---|---|
| **技术栈** | scikit-learn MLP + CountVectorizer | 外部 LLM API (Ollama/OpenAI 等) + 可选 RAG |
| **核心文件** | `src/documents/classifier.py`、`src/documents/matching.py` | `src/paperless_ai/ai_classifier.py`、`src/paperless_ai/matching.py` |
| **训练需求** | 需要离线训练，消费/建议时纯推断 | 无需训练，每次推断实时调用 LLM |
| **运行模式与入口** | ① **自动写入模式**（后台自动）：<br>&nbsp;&nbsp;• 文档消费时信号触发<br>&nbsp;&nbsp;• `document_retagger` 管理命令批量执行<br><br>② **仅建议模式**（前端手动）：<br>&nbsp;&nbsp;• `GET /api/documents/<id>/suggestions/` | 仅 **仅建议模式**（前端手动）：<br>&nbsp;&nbsp;• `GET /api/documents/<id>/ai_suggestions/` |
| **写入行为** | 自动写入模式：直接写入数据库（自动打标签）<br>仅建议模式：仅返回建议 JSON，不写入 DB | **仅返回建议 JSON**，不写入 DB，需用户确认后手动应用 |
| **匹配对象** | 只处理 `matching_algorithm == MATCH_AUTO` 的实体（非 AUTO 走规则匹配路径） | LLM 返回名称字符串 → 通过 `paperless_ai/matching.py` 的名称模糊匹配（阈值 0.8）映射到已有实体；未匹配的以 `suggested_*` 字段返回供用户新建 |
| **覆盖维度** | Tags / Correspondent / DocumentType / StoragePath | 上述 4 项 + Title + Dates |
| **依赖条件** | 本地模型文件存在即可 | 需要 `PAPERLESS_AI_ENABLED=true` 及 LLM 后端配置 |

> **澄清**：`suggestions` 与 `ai_suggestions` 是两个不同的 API 端点，分属不同系统。前者复用传统分类器的底层逻辑（`matching.match_*`），后者调用 LLM。两者都**不会自动写入数据库**——自动写入只发生在消费流程的信号处理器和 `document_retagger` 命令中。

---

## 二、核心数据模型：MatchingModel

所有可自动匹配的实体均继承自 `MatchingModel`，定义于 `src/documents/models.py`：

```python
class MatchingModel(ModelWithOwner):
    MATCH_NONE = 0
    MATCH_ANY = 1
    MATCH_ALL = 2
    MATCH_LITERAL = 3
    MATCH_REGEX = 4
    MATCH_FUZZY = 5
    MATCH_AUTO = 6   # ← 只有此值才会被传统分类器处理
```

**可复核点**：在 `src/documents/matching.py` 的 `matches()` 函数中，`MATCH_AUTO` 分支直接 `return False`，即规则匹配路径完全不处理 AUTO 实体，留给分类器路径。

---

## 三、训练数据准备流程（MATCH_AUTO 传统分类器）

### 3.1 训练触发入口

| 触发方式 | 入口位置 | 说明 |
|---|---|---|
| Celery 定时任务 | `src/paperless/settings/custom.py`（`"5 */1 * * *"`，每小时第 5 分钟） | 调用 `documents.tasks.train_classifier` |
| 管理命令 | `src/documents/management/commands/document_create_classifier.py` | `python manage.py document_create_classifier` |
| API 手动触发 | `src/documents/views.py` → `TasksViewSet` | 前端"任务"页面触发 |

### 3.2 任务调度层：前置检查

`src/documents/tasks.py` 中 `train_classifier()` 先做跳过检查：

```python
# 只要有任意一类实体配置了 MATCH_AUTO 就继续；否则删除旧模型并退出
if (not Tag.objects.filter(matching_algorithm=Tag.MATCH_AUTO).exists()
    and not DocumentType.objects.filter(...).exists()
    and not Correspondent.objects.filter(...).exists()
    and not StoragePath.objects.filter(...).exists()):
    if settings.MODEL_FILE.exists():
        settings.MODEL_FILE.unlink()  # 清理历史残留
    return "No automatic matching items, not training"
```

**可复核点**：如果所有实体之前配过 AUTO 并训练过，后来全部改为非 AUTO，模型文件会被主动删除。

### 3.3 训练数据提取（已核对）

`src/documents/classifier.py` → `DocumentClassifier.train()`：

**Step 1：查询训练文档集**

```python
docs_queryset = (
    Document.objects.exclude(tags__is_inbox_tag=True)   # 排除收件箱文档
    .select_related("document_type", "correspondent", "storage_path")
    .prefetch_related("tags")
    .order_by("pk")
)
```

**Step 2：构建标签向量 + 变化检测 Hash（关键细节）**

```python
hasher = sha256()
for doc in docs_queryset:
    # DocumentType: 单标签分类，-1 表示无
    y = dt.pk if (dt and dt.matching_algorithm == MATCH_AUTO) else -1
    hasher.update(y.to_bytes(4, "little", signed=True))  # -1 也计入 hash
    labels_document_type.append(y)

    # Correspondent / StoragePath: 同上逻辑，各更新一次 hash

    # Tags: 多标签分类，仅收集 MATCH_AUTO 的标签
    tags = list(doc.tags.filter(matching_algorithm=MATCH_AUTO)
                    .order_by("pk").values_list("pk", flat=True))
    for tag in tags:
        hasher.update(tag.to_bytes(4, "little", signed=True))  # 逐个标签 ID
    labels_tags.append(tags)
```

**⚠️ 修正说明**：hash 不是"所有 AUTO 标签的 ID 集合"的 hash，而是**按文档主键顺序**遍历，对每个文档的 4 个维度（document_type/correspondent/tags/storage_path）的 AUTO 标签 ID（无则为 -1）**逐个字节累积**的 SHA256。文档顺序变化、某文档的 AUTO 标签增删、甚至 -1 值的存在与否都会影响最终 hash。

**Step 3：重训练必要性检测（增量优化）**

```python
latest_doc_change = docs_queryset.latest("modified").modified
if (self.last_doc_change_time is not None
    and self.last_doc_change_time >= latest_doc_change
) and self.last_auto_type_hash == hasher.digest():
    # 文档未更新 + AUTO 标签配置未变化 → 跳过训练
    cache.set(CLASSIFIER_MODIFIED_KEY, ...)   # 缓存 50 分钟
    cache.set(CLASSIFIER_HASH_KEY, ...)
    return False
```

两个条件**同时满足**才跳过：
1. 最新文档修改时间 ≤ 上次训练时记录的时间
2. 上述累积 hash 与上次训练结果一致

### 3.4 文本预处理

`src/documents/classifier.py` → `preprocess_content()`：

```
原始文本 (doc.content)
  ↓
① 正则 RE_WORD 提取 → 转小写 → 仅保留 [\w]+ 词边界匹配的单词
  ↓
② 可选 NLTK 高级处理（需 PAPERLESS_NLTK_ENABLED + PAPERLESS_NLTK_LANGUAGE）:
   ├─ word_tokenize 分词
   ├─ SnowballStemmer 词干提取 (amazement → amaz)
   └─ 停用词过滤
  ↓
处理后文本
```

**可复核点**：训练时使用的是 `doc.content`（原始 OCR 文本），不是 `suggestion_content`（后者会截断超长文档，仅用于推断）。词干提取结果通过 `StoredLRUCache`（容量 10000，Redis 持久化）跨 worker 缓存。

---

## 四、模型训练与持久化（MATCH_AUTO）

### 4.1 特征向量化

```python
self.data_vectorizer = CountVectorizer(
    analyzer="word",
    ngram_range=(1, 2),    # 1-gram + 2-gram
    min_df=0.01,           # 至少出现在 1% 文档中才纳入词表
)
data_vectorized = self.data_vectorizer.fit_transform(content_generator())
self.data_vectorizer.stop_words_ = None   # 主动清理，减小序列化体积
```

### 4.2 四个独立 MLP 分类器

| 分类器字段 | 任务类型 | Binarizer |
|---|---|---|
| `tags_classifier` | 多标签分类（Multi-label） | `MultiLabelBinarizer`；**仅 1 个 AUTO tag 时退化**为 `LabelBinarizer`（二分类：有/无该标签） |
| `correspondent_classifier` | 多分类（Multi-class） | 无需（MLPClassifier 原生支持） |
| `document_type_classifier` | 多分类 | 无需 |
| `storage_path_classifier` | 多分类 | 无需 |

四个分类器均使用 `MLPClassifier(tol=0.01)`。

**单 Tag 退化逻辑（`src/documents/classifier.py` L355-L364）**：
```python
if num_tags == 1:
    labels_tags = [label[0] if len(label) == 1 else -1 for label in labels_tags]
    self.tags_binarizer = LabelBinarizer()
```

### 4.3 持久化格式

`src/documents/classifier.py` → `save()`：

```
文件结构（字节序列）:
  [0:32]   HMAC-SHA256 签名（key = Django SECRET_KEY）
  [32:]    pickle 序列化的元组:
            (FORMAT_VERSION,         # 当前 = 10
             last_doc_change_time,   # datetime，用于增量检测
             last_auto_type_hash,    # bytes，上述 SHA256 结果
             data_vectorizer,        # CountVectorizer
             tags_binarizer,         # LabelBinarizer / MultiLabelBinarizer
             tags_classifier,        # MLPClassifier
             correspondent_classifier,
             document_type_classifier,
             storage_path_classifier)
```

写入方式：先写 `.pickle.part` 临时文件 → `os.rename` 原子覆盖。

### 4.4 加载时的三层校验

`load()` 方法按顺序执行：
1. **HMAC 签名校验**（防篡改）
2. **FORMAT_VERSION 校验**（代码版本兼容）
3. **scikit-learn 版本校验**（捕获 `InconsistentVersionWarning`）

任意一层失败 → 删除模型文件，触发下次重新训练。

### 4.5 模型更新与建议缓存的衔接（已核对）

训练完成或跳过训练时，`train()` 方法会将三个 Key 写入 Django 缓存（TTL = 50 分钟，略短于每小时一次的训练周期）：

| 缓存 Key | 值 | 来源 |
|---|---|---|
| `CLASSIFIER_VERSION_KEY` (`"classifier_version"`) | `FORMAT_VERSION`（当前 = 10） | `DocumentClassifier.FORMAT_VERSION` |
| `CLASSIFIER_HASH_KEY` (`"classifier_hash"`) | 训练数据 hash 的 hex 字符串（`hasher.hexdigest()`） | 逐文档逐标签 ID 累积的 SHA256 |
| `CLASSIFIER_MODIFIED_KEY` (`"classifier_modified"`) | `latest_doc_change`（datetime） | `docs_queryset.latest("modified").modified` |

这三个 Key 是**传统分类器 suggestions 缓存有效性的根凭据**，通过两条路径控制缓存：

**路径一：浏览器协商缓存（`src/documents/conditionals.py`）**

`suggestions` API 通过 Django `@condition` 装饰器 + 以下两个函数实现 HTTP 级缓存：

- `suggestions_etag()`：返回 `"{hash}:{NUMBER_OF_SUGGESTED_DATES}"`。若分类器版本不匹配或缓存 Key 不存在，返回 `None` → 禁用协商缓存，强制重算。
- `suggestions_last_modified()`：返回 `CLASSIFIER_MODIFIED_KEY` 记录的 datetime。同样在版本不匹配时返回 `None`。

浏览器携带 `If-None-Match` / `If-Modified-Since` 请求时，Django 自动比对，未变化则返回 `304 Not Modified`。

**路径二：服务端建议缓存（`src/documents/caching.py`）**

建议结果以 `SuggestionCacheData(classifier_version, classifier_hash, suggestions)` 结构存入缓存，**传统 `suggestions` 与 LLM `ai_suggestions` 共用同一个缓存 Key**：`doc_{document_id}_suggest`，后写入的结果会**覆盖**先写入的结果。

- **写入**：
  - 传统分类器（`set_suggestions_cache()`）：
    ```python
    SuggestionCacheData(
        classifier_version = FORMAT_VERSION,          # 当前 = 10
        classifier_hash   = hexlify(last_auto_type_hash).decode(),  # 训练数据 hash
        suggestions       = {...},
    )
    ```
  - LLM 建议（`set_llm_suggestions_cache()`）：
    ```python
    SuggestionCacheData(
        classifier_version = LLM_CACHE_CLASSIFIER_VERSION,  # = 1000，仅写入标记
        classifier_hash   = backend,                         # 后端名称，如 "ollama:llama3"
        suggestions       = {...},
    )
    ```
    `LLM_CACHE_CLASSIFIER_VERSION = 1000` **仅用于写入时标识来源，读取时不参与校验**。

- **读取（传统分类器 `get_suggestion_cache()`）**：三层校验，**任一条件不满足则删除整个缓存**：
  ```
  ① 全局缓存中的 CLASSIFIER_VERSION_KEY == DocumentClassifier.FORMAT_VERSION
                       &&
  ② 全局缓存中的 CLASSIFIER_VERSION_KEY == 已缓存建议的 .classifier_version
                       &&
  ③ 全局缓存中的 CLASSIFIER_HASH_KEY    == 已缓存建议的 .classifier_hash
  ```
  不满足 → `cache.delete(doc_key)`，将该文档的建议缓存整条删除（包括可能存在的 LLM 结果）。

- **读取（LLM 建议 `get_llm_suggestion_cache()`）**：仅校验 backend 名称：
  ```python
  if data and data.classifier_hash == backend:
      return data
  return None
  ```
  **完全不校验 `classifier_version`**。只要 `classifier_hash` 等于传入的 backend 字符串即命中。

**两类缓存的相互影响：**

```
场景 1：先请求 suggestions（传统） → 后请求 ai_suggestions（LLM）
  → LLM 结果覆盖传统结果

场景 2：先请求 ai_suggestions（LLM） → 后请求 suggestions（传统）
  → 传统结果覆盖 LLM 结果

场景 3：请求 ai_suggestions 后，分类器重新训练
  → 下次请求 suggestions 时，传统 get_suggestion_cache() 校验 hash 不匹配
  → 整条 doc_{id}_suggest 缓存被删除（LLM 缓存也随之丢失）

场景 4：请求 suggestions 后，分类器重新训练
  → 同上，整条缓存被删除
```

**失效链路总结**：
```
train_classifier 重训（或检测到变化）
  ↓
更新 CLASSIFIER_VERSION_KEY / CLASSIFIER_HASH_KEY / CLASSIFIER_MODIFIED_KEY
  ↓
├─ conditionals.py：ETag / Last-Modified 变化 → 浏览器缓存失效（304 不再命中）
└─ get_suggestion_cache()：version / hash 不匹配 → 整条 doc_{id}_suggest 缓存被删除
                                                          （传统和 LLM 建议一并失效）
```

---

## 五、文档消费 → 自动打标签（已核对）

### 5.1 消费时序

`src/documents/consumer.py` → `ConsumerPlugin.run()`：

```
1. 文件复制到临时目录
2. MIME 类型检测 → 选择 Parser
3. Parser.parse() → 提取 text / date / 缩略图 / page_count
4. ★ classifier = load_classifier()    ← 仅加载一次
5. transaction.atomic():
   a. _store() → Document 写入 DB
   b. ★ document_consumption_finished.send(classifier=classifier, ...)
   c. 文件移动到 originals/ thumbnails/ archive/
   d. document.save() → 触发 post_save → 文件名重排
```

### 5.2 信号分发链（已核对，共 10 个连接）

`src/documents/apps.py` → `DocumentsConfig.ready()`：

```
document_consumption_finished（共 8 个处理器，按注册顺序）:
  ①  add_inbox_tags                       # 收件箱标签
  ②  set_correspondent                    # 自动匹配通信者
  ③  set_document_type                    # 自动匹配文档类型
  ④  set_tags                             # 自动匹配标签（多标签）
  ⑤  set_storage_path                     # 自动匹配存储路径
  ⑥  add_to_index                         # 全文搜索索引
  ⑦  run_workflows_added                  # DOCUMENT_ADDED 工作流
  ⑧  add_or_update_document_in_llm_index  # LLM 向量索引（可选）

document_updated（共 2 个处理器）:
  ①  run_workflows_updated
  ②  send_websocket_document_updated
```

**可复核点**：分类器对象通过信号参数 `classifier=classifier` 传递给所有处理器，避免每个处理器重复加载。

### 5.3 匹配策略：规则 + 分类器 双轨取并集

`src/documents/matching.py` 中 `match_tags()` / `match_correspondents()` 等函数结构一致：

```python
def match_tags(document, classifier, user=None):
    # ① 分类器预测（仅 MATCH_AUTO 实体）
    predicted_tag_ids = (classifier.predict_tags(document.suggestion_content)
                         if classifier else [])
    # ② 返回并集：规则匹配命中 OR（MATCH_AUTO 且分类器命中）
    return list(filter(
        lambda o: (
            matches(o, document)                                # ANY/ALL/LITERAL/REGEX/FUZZY
            or (o.matching_algorithm == MATCH_AUTO
                and o.pk in predicted_tag_ids)                 # 分类器预测
        ),
        tags
    ))
```

推断时使用的是 `document.suggestion_content`（超长文档会截断：头部 800k 字符 + 尾部 200k 字符），而非完整 `content`。

### 5.4 set_tags 的写入策略（已核对）

`src/documents/signals/handlers.py` → `set_tags()`：

| 参数组合 | 行为 |
|---|---|
| `replace=False`（消费流程默认） | 仅追加新匹配到的标签，**不删除**已有标签 |
| `replace=True`（retagger 覆盖模式） | 先删除旧标签，再应用新标签。但以下两类受保护不被删除：<br>• `is_inbox_tag=True`（收件箱标签）<br>• `match=""` 且 `matching_algorithm != MATCH_AUTO`（纯手动添加、无匹配规则的标签） |
| `dry_run=True`（retagger `--suggest` 预览模式） | 只计算变更集合，不写入 DB，不删除 |

`set_correspondent` / `set_document_type` / `set_storage_path` 还有 `use_first` 参数：
- `True`（函数默认值）：多个匹配时取第一个
- `False`：多个匹配时全部跳过，不做赋值

---

## 六、批量重打标：document_retagger（已核对）

`src/documents/management/commands/document_retagger.py`

### 6.1 命令参数

```bash
python manage.py document_retagger \
    -c -t -T -s \           # 处理维度：correspondent / doc_type / tags / storage_path
    --overwrite \            # 对应 replace=True（默认 False=追加）
    --use-first \            # 多匹配时取第一个（默认 False=全部跳过，区别于函数内默认 True）
    --suggest \              # dry_run 预览模式，不写入 DB
    --inbox-only \           # 仅处理含收件箱标签的文档
    --id-range 100 200 \     # 指定文档 ID 范围
    --base-url http://...    # suggest 模式下输出文档链接
```

**⚠️ 修正说明**：`--use-first` 的命令行默认值是 `False`（与 `set_correspondent` 函数内的默认值 `True` 不同），retagger 会显式将命令行值传入函数覆盖默认行为。

### 6.2 执行流程

```
遍历 Document QuerySet
  ↓
对每个文档依次调用:
  set_correspondent(..., replace=overwrite, use_first=use_first, dry_run=suggest)
  set_document_type(...)
  set_tags(..., replace=overwrite, dry_run=suggest)
  set_storage_path(...)
  ↓
累计统计（RetaggerStats）
  ↓
suggest=True → 输出 DocumentSuggestion 表格（仅展示有变更的文档）
suggest=False → 输出 RetaggerSummary 统计表
```

---

## 七、paperless_ai 建议接口（LLM 路线）

### 7.1 LLM 分类流程

`src/paperless_ai/ai_classifier.py` → `get_ai_document_classification()`：

```
Document 对象
  ↓
① build_prompt_without_rag() 或 build_prompt_with_rag()
   (Prompt = filename + content[:4000] + 可选 RAG 相似文档上下文)
  ↓
② AIClient.run_llm_query(prompt) → 调用外部 LLM API
  ↓
③ parse_ai_response() → 解析 JSON:
   {title, tags[], correspondents[], document_types[], storage_paths[], dates[]}
```

RAG 上下文来自 `src/paperless_ai/indexing.py` 的向量检索（最多 5 篇相似文档）。

### 7.2 名称 → 实体映射：fuzzy match 与 suggested_* 的差异

`src/paperless_ai/matching.py` 中的名称映射分为两个阶段，职责完全不同：

**阶段一：`_match_names_to_queryset()` — 名称 → 实体（含 fuzzy match）**

```
LLM 返回的名称字符串列表（如 ["Invoice", "HR Report", "UnknownTag"]）
  ↓
① 名称标准化：lower + 去标点 + strip
  ↓
② 精确名称匹配（完全相等）→ 命中则加入结果，从候选中移除
  ↓
③ fuzzy match 回退：difflib.get_close_matches(cutoff=0.8)
   ├─ 对每个未命中的名称，找相似度 ≥ 0.8 的最近邻
   └─ 命中则加入结果，从候选中移除
  ↓
返回匹配到的实体对象列表（仅包含数据库中已存在的实体）
```

**阶段二：`extract_unmatched_names()` — 从原始名称列表中过滤已匹配项**

```
输入：
  - LLM 返回的原始名称列表（如 ["Invoice", "HR Report", "UnknownTag"]）
  - 阶段一返回的已匹配实体列表（如 [Tag<Invoice>, Tag<"HR-Report">]）
  ↓
逻辑：
  matched_names = {实体.name.lower() for 实体 in matched_objects}
  返回 [name for name in names if name.lower() not in matched_names]
  ↓
输出：未匹配到的名称字符串列表（如 ["UnknownTag"]）
```

**两者的本质差异：**

| 维度 | `_match_names_to_queryset`（fuzzy match） | `extract_unmatched_names` |
|---|---|---|
| **阶段** | 阶段一：匹配 | 阶段二：后处理过滤 |
| **输入** | LLM 返回的名称列表 | LLM 返回的名称列表 + 已匹配实体列表 |
| **输出** | 匹配到的**实体对象列表** | 未匹配到的**名称字符串列表** |
| **fuzzy match 参与** | 是（作为精确匹配失败后的回退） | 否（仅基于已匹配实体的 name 做精确过滤） |
| **API 响应字段** | `tags` / `correspondents` 等（ID 列表） | `suggested_tags` / `suggested_correspondents` 等（字符串列表） |

**⚠️ 关键细节**：`extract_unmatched_names` 的匹配是**精确大小写不敏感匹配**，不走 fuzzy。例如 LLM 返回 `"HR Report"`，数据库中存在 `"HR-Report"`，fuzzy match 阶段已将其匹配到实体；`extract_unmatched_names` 比对的是实体的 `.name`（即 `"HR-Report"`），与 `"HR Report"` 不相等（空格 vs 连字符），因此该名称会**同时出现在** `tags`（已匹配 ID）和 `suggested_tags`（未匹配字符串）中。这是设计允许的行为。

### 7.3 API 响应结构

`src/documents/views.py` → `DocumentViewSet.ai_suggestions()`：

```python
resp_data = {
    "title": "LLM 生成的标题建议",
    "tags": [1, 5, 9],                       # 已匹配到的 Tag ID
    "suggested_tags": ["new_tag_name"],      # 未匹配，建议新建
    "correspondents": [3],                   # 已匹配
    "suggested_correspondents": [...],       # 未匹配
    "document_types": [...],
    "suggested_document_types": [...],
    "storage_paths": [...],
    "suggested_storage_paths": [...],
    "dates": ["2024-01-15", ...],
}
```

**可复核点**：LLM 建议接口仅返回 JSON，**从不自动写入数据库**。自动写入只有传统 MATCH_AUTO 分类器在消费流程 / retagger 中才会发生。

---

## 八、关键文件索引（相对路径）

| 文件 | 职责 |
|---|---|
| `src/documents/classifier.py` | 传统 ML 分类器：训练 / 预测 / 序列化 / 文本预处理 |
| `src/documents/matching.py` | 规则匹配 + 分类器预测的并集合并逻辑；工作流匹配 |
| `src/documents/caching.py` | 建议缓存读写：`get_suggestion_cache` / `set_suggestions_cache` / LLM 建议缓存；缓存 Key 常量定义 |
| `src/documents/conditionals.py` | HTTP 协商缓存：`suggestions_etag` / `suggestions_last_modified`，基于分类器 version/hash/modified |
| `src/documents/tasks.py` | Celery 任务：`train_classifier` / `consume_file` |
| `src/documents/consumer.py` | 文档消费主流程，加载分类器并触发 `document_consumption_finished` |
| `src/documents/signals/handlers.py` | 信号处理器：`set_tags` / `set_correspondent` / `set_document_type` / `set_storage_path` 等 |
| `src/documents/apps.py` | Django AppConfig，注册所有信号连接 |
| `src/documents/models.py` | `MatchingModel` 基类、`Document.suggestion_content` / `get_effective_content()` |
| `src/paperless_ai/ai_classifier.py` | LLM 分类：Prompt 构建 / RAG / 响应解析 |
| `src/paperless_ai/matching.py` | LLM 返回名称 → 实体的两阶段映射：`_match_names_to_queryset`（精确+fuzzy）+ `extract_unmatched_names`（精确过滤） |
| `src/documents/views.py` | `DocumentViewSet.suggestions`（传统分类器建议）<br>`DocumentViewSet.ai_suggestions`（LLM 建议） |
| `src/documents/management/commands/document_create_classifier.py` | 管理命令：手动触发训练 |
| `src/documents/management/commands/document_retagger.py` | 管理命令：批量重打标 |
| `src/paperless/settings/custom.py` | Celery Beat 定时任务配置（每小时训练一次） |

---

## 九、设计要点复核清单

以下为代码中可独立验证的设计决策：

- [x] **增量训练检测**：`last_doc_change_time`（文档修改时间）+ `last_auto_type_hash`（逐文档逐标签 ID 累积的 SHA256）双重校验 — `src/documents/classifier.py` L285-L303
- [x] **原子持久化**：`.pickle.part` 临时文件 + `rename` + HMAC-SHA256 签名 — `src/documents/classifier.py` L196-L219
- [x] **多级缓存**：向量化结果缓存 5 分钟（`_vectorize`）、词干提取 LRU 缓存 10000 条（`StoredLRUCache`）— `src/documents/classifier.py` L518-L534
- [x] **建议缓存失效链路**：`train_classifier` 更新 `CLASSIFIER_VERSION_KEY / HASH_KEY / MODIFIED_KEY`，同时驱动 conditionals.py 的浏览器协商缓存和 caching.py 的服务端建议缓存失效；传统分类器校验不通过时整条 `doc_{id}_suggest` 被删除（含 LLM 缓存） — `src/documents/caching.py` L132-L184, `src/documents/conditionals.py` L19-L68
- [x] **建议缓存共用 Key 且相互覆盖**：传统 `suggestions` 与 LLM `ai_suggestions` 写入同一个 `doc_{id}_suggest`，后写覆盖先写；传统读取三层校验（version/hash 不匹配则删整条缓存），LLM 读取仅校验 `classifier_hash == backend`，`LLM_CACHE_CLASSIFIER_VERSION` 仅写入时作标记 — `src/documents/caching.py` L34-L44, L132-L233
- [x] **规则 + ML 双轨并行**：MATCH_AUTO 走分类器，其余走规则匹配，结果取并集 — `src/documents/matching.py` L110-L134
- [x] **信号解耦**：消费流程通过 `document_consumption_finished` 分发，共 8 个处理器 — `src/documents/apps.py` L24-L31
- [x] **软删除保护**：收件箱标签和纯手动标签（`match=""` 且非 MATCH_AUTO）不会被自动系统删除 — `src/documents/signals/handlers.py` L248-L264
- [x] **建议接口均不自动写入**：`suggestions`（传统分类器）和 `ai_suggestions`（LLM）两个 API 均仅返回 JSON，无 DB 写入逻辑；自动写入仅发生在消费信号处理器和 `document_retagger` 中 — `src/documents/views.py` L1388-L1528
- [x] **LLM suggested_* 与 fuzzy match 的差异**：fuzzy match 是匹配阶段的回退（阈值 0.8），`extract_unmatched_names` 是匹配后的精确过滤（仅比对 `.name.lower()`），因此同一名称可能同时出现在 `tags`（ID 列表）和 `suggested_tags`（字符串列表）中 — `src/paperless_ai/matching.py` L18-L102
