# Paperless-ngx 自动分类机制深度分析

本文档重点分析自动分类的**触发信号**、**重新标记命令覆盖情况**、**多匹配参数**、**规则匹配与自动预测的关联**以及**已有值冲突处理方式**。

---

## 一、自动分类触发信号

### 1.1 信号定义与连接

自动分类通过 Django 信号机制触发，信号连接在 `src/documents/apps.py` 中配置：

[apps.py#L24-L33](file:///d:/fz/0601/solo-dogfeeding/code/27-paperless-ngx/src/documents/apps.py#L24-L33)

```python
# 消费完成信号 - 按顺序执行以下处理器
document_consumption_finished.connect(add_inbox_tags)
document_consumption_finished.connect(set_correspondent)
document_consumption_finished.connect(set_document_type)
document_consumption_finished.connect(set_tags)
document_consumption_finished.connect(set_storage_path)
document_consumption_finished.connect(add_to_index)
document_consumption_finished.connect(run_workflows_added)
document_consumption_finished.connect(add_or_update_document_in_llm_index)

# 文档更新信号
document_updated.connect(run_workflows_updated)
document_updated.connect(send_websocket_document_updated)
```

### 1.2 触发时机

#### 1.2.1 文档消费时触发

在 `src/documents/consumer.py` 的消费流程中，文档保存后发送信号：

[consumer.py#L570-L666](file:///d:/fz/0601/solo-dogfeeding/code/27-paperless-ngx/src/documents/consumer.py#L570-L666)

```python
# 分类器在消费流程中预加载，避免多次加载
classifier = load_classifier()

# 文档保存完成后发送信号
document_consumption_finished.send(
    sender=self.__class__,
    document=document,
    logging_group=self.logging_group,
    classifier=classifier,              # 预加载的分类器实例
    original_file=self.unmodified_original
    if self.unmodified_original
    else self.working_copy,
)
```

**关键设计**：分类器在信号发送前预加载，通过参数传递给所有信号处理器，避免重复加载开销。

#### 1.2.2 文档更新时触发

通过 `document_updated` 信号触发，主要用于工作流执行：

[handlers.py#L819-L829](file:///d:/fz/0601/solo-dogfeeding/code/27-paperless-ngx/src/documents/signals/handlers.py#L819-L829)

```python
def run_workflows_updated(
    sender,
    document: Document,
    logging_group: uuid.UUID | None = None,
    **kwargs,
) -> None:
    run_workflows(
        trigger_type=WorkflowTrigger.WorkflowTriggerType.DOCUMENT_UPDATED,
        document=document,
        logging_group=logging_group,
    )
```

#### 1.2.3 批量更新时触发

`bulk_update_documents` 任务中显式发送信号：

[tasks.py#L253-L276](file:///d:/fz/0601/solo-dogfeeding/code/27-paperless-ngx/src/documents/tasks.py#L253-L276)

```python
@shared_task
def bulk_update_documents(document_ids) -> None:
    documents = Document.objects.filter(id__in=document_ids)
    for doc in documents:
        clear_document_caches(doc.pk)
        document_updated.send(
            sender=None,
            document=doc,
            logging_group=uuid.uuid4(),
        )
        post_save.send(Document, instance=doc, created=False)
```

#### 1.2.4 手动重新标记命令

通过 `document_retagger` 命令行工具手动触发，不依赖信号机制，直接调用分类函数：

[document_retagger.py#L278-L328](file:///d:/fz/0601/solo-dogfeeding/code/27-paperless-ngx/src/documents/management/commands/document_retagger.py#L278-L328)

### 1.3 信号处理器执行顺序

`document_consumption_finished` 信号的处理器按连接顺序执行：

| 顺序 | 处理器 | 功能 | 代码位置 |
|------|-------|------|---------|
| 1 | `add_inbox_tags` | 添加收件箱标签 | [handlers.py#L80-L91](file:///d:/fz/0601/solo-dogfeeding/code/27-paperless-ngx/src/documents/signals/handlers.py#L80-L91) |
| 2 | `set_correspondent` | 自动分配联系人 | [handlers.py#L93-L151](file:///d:/fz/0601/solo-dogfeeding/code/27-paperless-ngx/src/documents/signals/handlers.py#L93-L151) |
| 3 | `set_document_type` | 自动分配文档类型 | [handlers.py#L154-L212](file:///d:/fz/0601/solo-dogfeeding/code/27-paperless-ngx/src/documents/signals/handlers.py#L154-L212) |
| 4 | `set_tags` | 自动分配标签 | [handlers.py#L215-L277](file:///d:/fz/0601/solo-dogfeeding/code/27-paperless-ngx/src/documents/signals/handlers.py#L215-L277) |
| 5 | `set_storage_path` | 自动分配存储路径 | [handlers.py#L280-L338](file:///d:/fz/0601/solo-dogfeeding/code/27-paperless-ngx/src/documents/signals/handlers.py#L280-L338) |
| 6 | `add_to_index` | 添加到搜索索引 | [handlers.py#L794-L800](file:///d:/fz/0601/solo-dogfeeding/code/27-paperless-ngx/src/documents/signals/handlers.py#L794-L800) |
| 7 | `run_workflows_added` | 运行文档添加工作流 | [handlers.py#L803-L816](file:///d:/fz/0601/solo-dogfeeding/code/27-paperless-ngx/src/documents/signals/handlers.py#L803-L816) |
| 8 | `add_or_update_document_in_llm_index` | 更新 LLM 索引 | - |

**重要**：分类处理器（2-5）之间没有依赖关系，按顺序独立执行。

---

## 二、重新标记命令的覆盖情况

### 2.1 命令参数概览

`document_retagger` 命令在 `src/documents/management/commands/document_retagger.py` 中实现，支持以下覆盖相关参数：

[document_retagger.py#L186-L228](file:///d:/fz/0601/solo-dogfeeding/code/27-paperless-ngx/src/documents/management/commands/document_retagger.py#L186-L228)

```python
def add_arguments(self, parser) -> None:
    parser.add_argument("-c", "--correspondent", default=False, action="store_true")
    parser.add_argument("-T", "--tags", default=False, action="store_true")
    parser.add_argument("-t", "--document_type", default=False, action="store_true")
    parser.add_argument("-s", "--storage_path", default=False, action="store_true")
    parser.add_argument("-i", "--inbox-only", default=False, action="store_true")
    parser.add_argument("--use-first", default=False, action="store_true",
        help="By default this command will not try to assign a correspondent "
             "if more than one matches the document. Use this flag to pick "
             "the first match instead.")
    parser.add_argument("-f", "--overwrite", default=False, action="store_true",
        help="Overwrite any previously set correspondent, document type, and "
             "remove tags that no longer match due to changed rules.")
    parser.add_argument("--suggest", default=False, action="store_true",
        help="Show what would be changed without applying anything.")
    parser.add_argument("--base-url", help="Base URL for document links in suggest output.")
    parser.add_argument("--id-range", nargs=2, type=int,
        help="Restrict retagging to documents within this ID range (inclusive).")
```

### 2.2 参数映射与执行逻辑

[document_retagger.py#L278-L328](file:///d:/fz/0601/solo-dogfeeding/code/27-paperless-ngx/src/documents/management/commands/document_retagger.py#L278-L328)

```python
# 命令行参数 -> 函数参数映射
for document in documents:
    if do_correspondent:
        correspondent = set_correspondent(
            None,
            document,
            classifier=classifier,
            replace=overwrite,       # --overwrite
            use_first=use_first,     # --use-first
            dry_run=suggest,         # --suggest
        )

    if do_document_type:
        document_type = set_document_type(
            None,
            document,
            classifier=classifier,
            replace=overwrite,       # --overwrite
            use_first=use_first,     # --use-first
            dry_run=suggest,         # --suggest
        )

    if do_tags:
        tags_to_add, tags_to_remove = set_tags(
            None,
            document,
            classifier=classifier,
            replace=overwrite,       # --overwrite
            dry_run=suggest,         # --suggest
            # 注意：set_tags 没有 use_first 参数
        )

    if do_storage_path:
        storage_path = set_storage_path(
            None,
            document,
            classifier=classifier,
            replace=overwrite,       # --overwrite
            use_first=use_first,     # --use-first
            dry_run=suggest,         # --suggest
        )
```

### 2.3 覆盖模式对比

| 场景 | `--overwrite` | `--use-first` | 行为描述 |
|------|--------------|--------------|---------|
| 默认消费 | `False` | `True` | 不覆盖已有值；多匹配时取第一个 |
| 重新标记默认 | `False` | `False` | 不覆盖已有值；多匹配时**不分配**（安全默认） |
| 强制覆盖 | `True` | `False` | 覆盖已有值；多匹配时不分配 |
| 强制覆盖+首选 | `True` | `True` | 覆盖已有值；多匹配时取第一个 |
| 仅建议模式 | 任意 | 任意 | 仅计算，不写入数据库 |

### 2.4 标签的特殊覆盖逻辑

`set_tags` 函数在 `replace=True` 时有特殊的标签保留规则：

[handlers.py#L248-L264](file:///d:/fz/0601/solo-dogfeeding/code/27-paperless-ngx/src/documents/signals/handlers.py#L248-L264)

```python
if replace:
    # 移除标签，但保留以下两种：
    Document.tags.through.objects.filter(document=document).exclude(
        Q(tag__is_inbox_tag=True),              # 1. 收件箱标签
    ).exclude(
        Q(tag__match="") & ~Q(tag__matching_algorithm=Tag.MATCH_AUTO),
        # 2. 手动添加的标签（match为空且非自动分类）
    ).delete()
```

**标签覆盖时的保留规则**：
- ✅ 保留 `is_inbox_tag=True` 的标签（收件箱标签）
- ✅ 保留 `match=""` 且 `matching_algorithm != MATCH_AUTO` 的标签（手动标签）
- ❌ 删除其他所有标签（规则匹配标签和自动分类标签）

---

## 三、多匹配参数（use_first）

### 3.1 参数定义与默认值

四个分类决策函数中，三个支持 `use_first` 参数：

| 函数 | 是否支持 `use_first` | 默认值 | 代码位置 |
|------|---------------------|--------|---------|
| `set_correspondent` | 是 | `True` | [handlers.py#L100](file:///d:/fz/0601/solo-dogfeeding/code/27-paperless-ngx/src/documents/signals/handlers.py#L100) |
| `set_document_type` | 是 | `True` | [handlers.py#L161](file:///d:/fz/0601/solo-dogfeeding/code/27-paperless-ngx/src/documents/signals/handlers.py#L161) |
| `set_storage_path` | 是 | `True` | [handlers.py#L288](file:///d:/fz/0601/solo-dogfeeding/code/27-paperless-ngx/src/documents/signals/handlers.py#L288) |
| `set_tags` | 否 | - | 标签支持多值，无冲突问题 |

### 3.2 冲突处理实现

以 `set_correspondent` 为例：

[handlers.py#L128-L141](file:///d:/fz/0601/solo-dogfeeding/code/27-paperless-ngx/src/documents/signals/handlers.py#L128-L141)

```python
potential_correspondents = matching.match_correspondents(document, classifier)
potential_count = len(potential_correspondents)
selected = potential_correspondents[0] if potential_correspondents else None

if potential_count > 1:
    if use_first:
        logger.debug(
            f"Detected {potential_count} potential correspondents, "
            f"so we've opted for {selected}",
            extra={"group": logging_group},
        )
    else:
        logger.debug(
            f"Detected {potential_count} potential correspondents, "
            f"not assigning any correspondent",
            extra={"group": logging_group},
        )
        return None
```

**决策逻辑**：
```
候选列表长度 N：
  N == 0 → 返回 None，不分配
  N == 1 → 返回该候选
  N > 1  → 
    use_first=True  → 返回候选列表第一个
    use_first=False → 返回 None，不分配
```

### 3.3 不同调用场景的参数值

| 调用场景 | `use_first` 值 | 说明 |
|---------|---------------|------|
| 文档消费（信号触发） | `True` | 快速分配，即使有多个匹配也选第一个 |
| 重新标记（默认） | `False` | 保守策略，多匹配时跳过，避免错误 |
| 重新标记（`--use-first`） | `True` | 主动选择第一个匹配 |

### 3.4 "第一个"的定义

候选列表的排序由匹配查询的 `order_by("name")` 决定，定义在 `MatchingModel.Meta` 中：

[models.py#L79](file:///d:/fz/0601/solo-dogfeeding/code/27-paperless-ngx/src/documents/models.py#L79)

```python
class Meta(ModelWithOwner.Meta):
    abstract = True
    ordering = ("name",)  # 按名称字母排序
```

**注意**：匹配结果按名称字母顺序排序，"第一个"是名称字母顺序最靠前的那个，而非匹配质量最高的。

---

## 四、规则匹配与自动预测的关联

### 4.1 匹配算法定义

`MatchingModel` 定义了 7 种匹配算法：

[models.py#L46-L63](file:///d:/fz/0601/solo-dogfeeding/code/27-paperless-ngx/src/documents/models.py#L46-L63)

```python
class MatchingModel(ModelWithOwner):
    MATCH_NONE = 0     # 不匹配
    MATCH_ANY = 1      # 任意词
    MATCH_ALL = 2      # 全词
    MATCH_LITERAL = 3  # 精确匹配
    MATCH_REGEX = 4    # 正则表达式
    MATCH_FUZZY = 5    # 模糊匹配
    MATCH_AUTO = 6     # 自动分类

    matching_algorithm = models.PositiveSmallIntegerField(
        choices=MATCHING_ALGORITHMS,
        default=MATCH_ANY,
    )
```

### 4.2 核心匹配函数 `matches()`

传统规则匹配在 `src/documents/matching.py` 的 `matches()` 函数中实现：

[matching.py#L169-L263](file:///d:/fz/0601/solo-dogfeeding/code/27-paperless-ngx/src/documents/matching.py#L169-L263)

```python
def matches(matching_model: MatchingModel, document: Document):
    # 获取文档有效内容
    document_content = document.get_effective_content() or ""

    # 空匹配字符串不匹配
    if not matching_model.match.strip():
        return False

    # 根据 matching_algorithm 分发到不同匹配逻辑
    if matching_model.matching_algorithm == MatchingModel.MATCH_NONE:
        return False
    elif matching_model.matching_algorithm == MatchingModel.MATCH_ALL:
        # ... 全词匹配逻辑
    elif matching_model.matching_algorithm == MatchingModel.MATCH_ANY:
        # ... 任意词匹配逻辑
    # ... 其他算法
    elif matching_model.matching_algorithm == MatchingModel.MATCH_AUTO:
        # this is done elsewhere.
        return False
```

**关键点**：`MATCH_AUTO` 在 `matches()` 中直接返回 `False`，实际匹配在 `match_*` 系列函数中完成。

### 4.3 组合匹配逻辑

`match_correspondents` 等函数结合了**规则匹配**和**自动预测**，使用 **OR 关系**：

[matching.py#L47-L76](file:///d:/fz/0601/solo-dogfeeding/code/27-paperless-ngx/src/documents/matching.py#L47-L76)

```python
def match_correspondents(document: Document, classifier: DocumentClassifier, user=None):
    # 1. 机器学习预测
    pred_id = (
        classifier.predict_correspondent(document.suggestion_content)
        if classifier
        else None
    )

    # 2. 权限过滤
    if user is not None:
        correspondents = get_objects_for_user_owner_aware(
            user, "documents.view_correspondent", Correspondent,
        )
    else:
        correspondents = Correspondent.objects.all()

    # 3. 组合匹配：规则匹配 OR 自动预测
    return list(
        filter(
            lambda o: (
                matches(o, document)                            # 规则匹配
                or (
                    o.pk == pred_id                               # ML预测ID匹配
                    and o.matching_algorithm == MatchingModel.MATCH_AUTO
                    # 仅当规则配置为AUTO时才接受ML预测
                )
            ),
            correspondents,
        ),
    )
```

### 4.4 匹配逻辑真值表

| 规则算法 | 规则匹配结果 | ML 预测命中 | 最终结果 | 说明 |
|---------|-------------|------------|---------|------|
| `MATCH_ANY` | True | 任意 | True | 规则匹配优先，无需 ML |
| `MATCH_ANY` | False | 是 | False | ML 预测仅对 `MATCH_AUTO` 有效 |
| `MATCH_AUTO` | False (恒) | 是 | True | ML 预测命中 |
| `MATCH_AUTO` | False (恒) | 否 | False | ML 预测未命中 |
| 其他算法 | True | 任意 | True | 规则匹配成功 |
| 其他算法 | False | 是 | False | 规则配置非 AUTO，ML 不生效 |

### 4.5 四种元数据类型的匹配逻辑

所有 `match_*` 函数使用相同的 OR 逻辑：

| 函数 | 代码位置 | ML 预测函数 |
|------|---------|------------|
| `match_correspondents` | [matching.py#L47-L76](file:///d:/fz/0601/solo-dogfeeding/code/27-paperless-ngx/src/documents/matching.py#L47-L76) | `predict_correspondent` |
| `match_document_types` | [matching.py#L79-L107](file:///d:/fz/0601/solo-dogfeeding/code/27-paperless-ngx/src/documents/matching.py#L79-L107) | `predict_document_type` |
| `match_tags` | [matching.py#L110-L134](file:///d:/fz/0601/solo-dogfeeding/code/27-paperless-ngx/src/documents/matching.py#L110-L134) | `predict_tags` |
| `match_storage_paths` | [matching.py#L137-L166](file:///d:/fz/0601/solo-dogfeeding/code/27-paperless-ngx/src/documents/matching.py#L137-L166) | `predict_storage_path` |

### 4.6 机器学习预测实现

以 `predict_correspondent` 为例：

[classifier.py#L536-L545](file:///d:/fz/0601/solo-dogfeeding/code/27-paperless-ngx/src/documents/classifier.py#L536-L545)

```python
def predict_correspondent(self, content: str) -> int | None:
    if self.correspondent_classifier:
        X = self._vectorize(content)                   # 文本向量化
        correspondent_id = self.correspondent_classifier.predict(X)
        return correspondent_id if correspondent_id != -1 else None
    return None
```

**预测值 `-1` 的含义**：表示"无匹配"，在训练时作为空标签的占位符。

---

## 五、已有值冲突处理方式

### 5.1 已有值保护机制

所有分类决策函数都通过 `replace` 参数控制是否覆盖已有值：

[handlers.py#L121-L122](file:///d:/fz/0601/solo-dogfeeding/code/27-paperless-ngx/src/documents/signals/handlers.py#L121-L122)

```python
# set_correspondent 中的已有值检查
if document.correspondent and not replace:
    return None
```

相同逻辑适用于：
- `set_document_type` - [handlers.py#L182-L183](file:///d:/fz/0601/solo-dogfeeding/code/27-paperless-ngx/src/documents/signals/handlers.py#L182-L183)
- `set_storage_path` - [handlers.py#L308-L309](file:///d:/fz/0601/solo-dogfeeding/code/27-paperless-ngx/src/documents/signals/handlers.py#L308-L309)

### 5.2 冲突处理策略对比

| 元数据类型 | 冲突点 | `replace=False`（默认消费） | `replace=True`（重新标记） |
|-----------|--------|----------------------------|---------------------------|
| 联系人 | 已有 `correspondent_id` | 直接返回 None，保留原值 | 覆盖为新匹配值 |
| 文档类型 | 已有 `document_type_id` | 直接返回 None，保留原值 | 覆盖为新匹配值 |
| 存储路径 | 已有 `storage_path_id` | 直接返回 None，保留原值 | 覆盖为新匹配值 |
| 标签 | 已有标签集合 | 添加新匹配标签，保留原有标签 | 删除旧自动标签，添加新匹配标签 |

### 5.3 标签的特殊冲突处理

标签是多值属性，冲突处理方式不同：

[handlers.py#L248-L277](file:///d:/fz/0601/solo-dogfeeding/code/27-paperless-ngx/src/documents/signals/handlers.py#L248-L277)

```python
def set_tags(sender, document, *, classifier=None, replace=False, dry_run=False, **kwargs):
    # replace=True 时，先移除旧标签
    if replace:
        tags_to_remove: set[Tag] = set(
            document.tags.exclude(
                is_inbox_tag=True,                    # 保留收件箱标签
            ).exclude(
                Q(match="") & ~Q(matching_algorithm=Tag.MATCH_AUTO),
                # 保留手动标签
            ),
        )
        if not dry_run:
            Document.tags.through.objects.filter(document=document).exclude(
                Q(tag__is_inbox_tag=True),
            ).exclude(
                Q(tag__match="") & ~Q(tag__matching_algorithm=Tag.MATCH_AUTO),
            ).delete()

    # 添加新匹配的标签（增量添加，不删除）
    current_tags = set(document.tags.all())
    matched_tags = matching.match_tags(document, classifier)
    tags_to_add = set(matched_tags) - current_tags

    if tags_to_add and not dry_run:
        document.add_nested_tags(tags_to_add)

    return tags_to_add, tags_to_remove
```

### 5.4 手动设置值的保护

**如何判断"手动设置"的标签**：
- `match=""`（匹配字符串为空）
- `matching_algorithm != MATCH_AUTO`（非自动分类算法）

这种标签被认为是用户手动添加的，在 `replace=True` 时也会被保留。

### 5.5 工作流的优先级

工作流动作在分类信号之后执行，可以覆盖自动分类结果：

[apps.py#L30](file:///d:/fz/0601/solo-dogfeeding/code/27-paperless-ngx/src/documents/apps.py#L30)

```python
document_consumption_finished.connect(run_workflows_added)
```

执行顺序：
1. `set_correspondent` → 自动分配联系人
2. `set_document_type` → 自动分配文档类型
3. `set_tags` → 自动分配标签
4. `set_storage_path` → 自动分配存储路径
5. `run_workflows_added` → 工作流执行，可覆盖以上值

**工作流分配具有最终决定权**。

---

## 六、完整调用链总结

### 6.1 文档消费时的分类流程

```
consumer.py:
  └─ ConsumerPlugin.run()
      ├─ 解析文档，提取内容
      ├─ 保存 Document 到数据库
      ├─ classifier = load_classifier()  ← 预加载分类器
      └─ document_consumption_finished.send(..., classifier=classifier)
          │
          ├─ add_inbox_tags()
          ├─ set_correspondent(replace=False, use_first=True)
          │   └─ matching.match_correspondents(doc, classifier)
          │       ├─ classifier.predict_correspondent(doc.suggestion_content)
          │       └─ filter: matches(o, doc) OR (o.pk == pred_id AND algo == AUTO)
          ├─ set_document_type(replace=False, use_first=True)
          ├─ set_tags(replace=False)
          ├─ set_storage_path(replace=False, use_first=True)
          ├─ add_to_index()
          └─ run_workflows_added()  ← 工作流可覆盖自动分类结果
```

### 6.2 重新标记命令的执行流程

```
document_retagger.py handle():
  ├─ 解析参数: overwrite, use_first, suggest
  ├─ classifier = load_classifier()
  └─ for document in documents:
      ├─ set_correspondent(replace=overwrite, use_first=use_first, dry_run=suggest)
      ├─ set_document_type(replace=overwrite, use_first=use_first, dry_run=suggest)
      ├─ set_tags(replace=overwrite, dry_run=suggest)
      └─ set_storage_path(replace=overwrite, use_first=use_first, dry_run=suggest)
```

---

## 七、代码引用汇总（仓库相对路径）

| 功能模块 | 相对路径 | 关键行范围 |
|---------|---------|-----------|
| 信号连接配置 | `src/documents/apps.py` | [L24-L33](file:///d:/fz/0601/solo-dogfeeding/code/27-paperless-ngx/src/documents/apps.py#L24-L33) |
| 消费时信号触发 | `src/documents/consumer.py` | [L570-L666](file:///d:/fz/0601/solo-dogfeeding/code/27-paperless-ngx/src/documents/consumer.py#L570-L666) |
| 重新标记命令 | `src/documents/management/commands/document_retagger.py` | [L186-L328](file:///d:/fz/0601/solo-dogfeeding/code/27-paperless-ngx/src/documents/management/commands/document_retagger.py#L186-L328) |
| 匹配算法定义 | `src/documents/models.py` | [L46-L79](file:///d:/fz/0601/solo-dogfeeding/code/27-paperless-ngx/src/documents/models.py#L46-L79) |
| 规则匹配实现 | `src/documents/matching.py` | [L169-L263](file:///d:/fz/0601/solo-dogfeeding/code/27-paperless-ngx/src/documents/matching.py#L169-L263) |
| 组合匹配逻辑 | `src/documents/matching.py` | [L47-L166](file:///d:/fz/0601/solo-dogfeeding/code/27-paperless-ngx/src/documents/matching.py#L47-L166) |
| ML 预测实现 | `src/documents/classifier.py` | [L536-L588](file:///d:/fz/0601/solo-dogfeeding/code/27-paperless-ngx/src/documents/classifier.py#L536-L588) |
| 联系人分类决策 | `src/documents/signals/handlers.py` | [L93-L151](file:///d:/fz/0601/solo-dogfeeding/code/27-paperless-ngx/src/documents/signals/handlers.py#L93-L151) |
| 文档类型分类决策 | `src/documents/signals/handlers.py` | [L154-L212](file:///d:/fz/0601/solo-dogfeeding/code/27-paperless-ngx/src/documents/signals/handlers.py#L154-L212) |
| 标签分类决策 | `src/documents/signals/handlers.py` | [L215-L277](file:///d:/fz/0601/solo-dogfeeding/code/27-paperless-ngx/src/documents/signals/handlers.py#L215-L277) |
| 存储路径分类决策 | `src/documents/signals/handlers.py` | [L280-L338](file:///d:/fz/0601/solo-dogfeeding/code/27-paperless-ngx/src/documents/signals/handlers.py#L280-L338) |
| 批量更新信号发送 | `src/documents/tasks.py` | [L253-L276](file:///d:/fz/0601/solo-dogfeeding/code/27-paperless-ngx/src/documents/tasks.py#L253-L276) |
