# Paperless-ngx Workflow 规则引擎与触发器深度分析

## 一、核心数据模型

### 1.1 Workflow（工作流）

定义于 [models.py](file:///d:/fz/0601/solo-dogfeeding/code/112-paperless-ngx/src/documents/models.py#L1828-L1850)

| 字段 | 类型 | 说明 |
|------|------|------|
| `name` | CharField | 工作流名称，唯一 |
| `order` | SmallIntegerField | 执行顺序，默认 0；`get_workflows_for_trigger()` 查询时按 `order` 排序 |
| `triggers` | ManyToManyField(WorkflowTrigger) | 触发器集合，任意一个匹配即可触发（OR 语义） |
| `actions` | ManyToManyField(WorkflowAction) | 动作集合，按 `order, pk` 升序执行 |
| `enabled` | BooleanField | 是否启用；查询时 `enabled=True` 过滤 |

### 1.2 WorkflowTrigger（触发器）

定义于 [models.py](file:///d:/fz/0601/solo-dogfeeding/code/112-paperless-ngx/src/documents/models.py#L1243-L1467)

#### 触发器类型 WorkflowTriggerType

| 值 | 名称 | 触发时机 | 匹配对象类型 |
|----|------|----------|-------------|
| 1 | `CONSUMPTION` | 文档消费开始时（文件尚未入库） | `ConsumableDocument` |
| 2 | `DOCUMENT_ADDED` | 文档消费完成并入库后 | `Document` |
| 3 | `DOCUMENT_UPDATED` | 文档元数据更新后 | `Document` |
| 4 | `SCHEDULED` | Celery Beat 定时调度 | `Document` |

#### 匹配模式 WorkflowTriggerMatching

| 值 | 算法 | 说明 |
|----|------|------|
| 0 | `NONE` | 不启用内容匹配（默认） |
| 1 | `ANY` | 任意单词出现即匹配 |
| 2 | `ALL` | 所有单词都出现才匹配 |
| 3 | `LITERAL` | 精确字符串匹配 |
| 4 | `REGEX` | 正则表达式匹配 |
| 5 | `FUZZY` | 模糊匹配（rapidfuzz partial_ratio >= 90） |

#### 触发器过滤条件总表

| 过滤维度 | 字段 | CONSUMPTION | DOCUMENT_ADDED/UPDATED | SCHEDULED |
|----------|------|:-----------:|:---------------------:|:---------:|
| 文档来源 | `sources` | ✅ | ❌ | ❌ |
| 邮件规则 | `filter_mailrule` | ✅ | ❌ | ❌ |
| 文件名 | `filter_filename` | ✅ | ✅ | ✅(预过滤) |
| 文件路径 | `filter_path` | ✅ | ❌ | ❌ |
| 内容文本 | `match` + `matching_algorithm` | ❌ | ✅ | ✅ |
| 标签（任意） | `filter_has_tags` | ❌ | ✅ | ✅(预过滤) |
| 标签（全部） | `filter_has_all_tags` | ❌ | ✅ | ✅(预过滤) |
| 标签（排除） | `filter_has_not_tags` | ❌ | ✅ | ✅(预过滤) |
| 文档类型（单一/任意/排除） | 3 个字段 | ❌ | ✅ | ✅(预过滤) |
| 联系人（单一/任意/排除） | 3 个字段 | ❌ | ✅ | ✅(预过滤) |
| 存储路径（单一/任意/排除） | 3 个字段 | ❌ | ✅ | ✅(预过滤) |
| 自定义字段查询 | `filter_custom_field_query` | ❌ | ✅ | ✅(预过滤) |
| 调度日期字段 | `schedule_date_field` | ❌ | ❌ | ✅ |
| 调度偏移天数 | `schedule_offset_days` | ❌ | ❌ | ✅ |
| 是否周期 | `schedule_is_recurring` | ❌ | ❌ | ✅ |
| 周期间隔 | `schedule_recurring_interval_days` | ❌ | ❌ | ✅ |

### 1.3 WorkflowAction（动作）

定义于 [models.py](file:///d:/fz/0601/solo-dogfeeding/code/112-paperless-ngx/src/documents/models.py#L1556-L1825)

#### 动作类型 WorkflowActionType

| 值 | 名称 | 作用对象 | 执行方式 | 是否创建 PaperlessTask |
|----|------|----------|----------|:---------------------:|
| 1 | `ASSIGNMENT` | Document / Overrides | 同步（当前线程） | ❌（随宿主任务） |
| 2 | `REMOVAL` | Document / Overrides | 同步（当前线程） | ❌（随宿主任务） |
| 3 | `EMAIL` | - | 同步（当前线程） | ❌（随宿主任务） |
| 4 | `WEBHOOK` | - | **异步**（独立 Celery Task） | ❌（不在 TRACKED_TASKS 中） |
| 5 | `PASSWORD_REMOVAL` | Document | 同步 / 信号延迟 | ❌（随宿主任务） |
| 6 | `MOVE_TO_TRASH` | Document / ConsumableDocument | **延迟到所有动作完成后** | ❌（随宿主任务） |

### 1.4 WorkflowRun（工作流运行记录）

定义于 [models.py](file:///d:/fz/0601/solo-dogfeeding/code/112-paperless-ngx/src/documents/models.py#L1853-L1886)

**与 Workflow 规则的精确关联字段**：

| 字段 | 类型 | 作用 |
|------|------|------|
| `workflow` | ForeignKey(Workflow) | 关联到具体哪条 Workflow 规则 |
| `type` | PositiveSmallIntegerField | 以 `WorkflowTrigger.WorkflowTriggerType` 记录本次是通过哪种触发类型执行的（1=CONSUMPTION 等） |
| `document` | ForeignKey(Document) | 关联到被处理的文档；**CONSUMPTION 模式下为 NULL**（因为此时文档尚未入库） |
| `run_at` | DateTimeField | 执行时间，默认 `timezone.now()`，db_index 加速定时去重查询 |
| （继承） | SoftDeleteModel | 支持软删除 |

**核心用途（仅 SCHEDULED 场景使用）**：在 [tasks.py](file:///d:/fz/0601/solo-dogfeeding/code/112-paperless-ngx/src/documents/tasks.py#L537-L568) `check_scheduled_workflows()` 中做去重判断：

```python
# 非周期工作流：只要存在任意一条 WorkflowRun 就不再执行
if not trigger.schedule_is_recurring and workflow_runs.exists():
    continue

# 周期工作流：距上次 WorkflowRun.run_at 的时间 < 间隔天数则跳过
if trigger.schedule_is_recurring and workflow_runs.exists() and (
    workflow_runs.first().run_at
    > now - datetime.timedelta(days=trigger.schedule_recurring_interval_days)
):
    continue
```

> **重要**：`run_workflows()` 在每次成功匹配并执行完所有动作后都会创建 WorkflowRun 记录（见 [signals/handlers.py](file:///d:/fz/0601/solo-dogfeeding/code/112-paperless-ngx/src/documents/signals/handlers.py#L986-L990)）。

**四种触发类型下 WorkflowRun 的写入与查询对照**：

| 触发类型 | 动作全部成功时是否写入 WorkflowRun | 仅 EMAIL/WEBHOOK/PASSWORD 失败（被内部吞掉） | ASSIGNMENT/REMOVAL/MOVE_TO_TRASH 异常向上抛出时 | WorkflowRun.document | 是否查询 WorkflowRun 去重 | 去重字段 |
|----------|:---------------------------------:|:------------------------------------------:|:---------------------------------------------:|:--------------------:|:------------------------:|---------|
| CONSUMPTION | ✅ | ✅（有审计记录，但日志有错误） | ❌ | `NULL`（文档未入库） | ❌ | — |
| DOCUMENT_ADDED | ✅ | ✅ | ❌ | Document FK | ❌ | — |
| DOCUMENT_UPDATED | ✅ | ✅ | ❌ | Document FK | ❌ | — |
| SCHEDULED | ✅ | ✅ | ❌ | Document FK | ✅ | `(document_id, type=SCHEDULED, workflow_id)` 三元组 |

- **只写不查（CONSUMPTION/ADDED/UPDATED）**：这三种由事件驱动，本身天然"一次性"，无需去重，WorkflowRun 仅作为审计记录存在
- **又写又查（SCHEDULED）**：由 Celery Beat 周期性重复触发，必须靠 WorkflowRun 防止非周期工作流重复执行、以及控制周期工作流的触发间隔
- **EMAIL/WEBHOOK/PASSWORD_REMOVAL 失败 = 部分失败但仍算"成功完成"**：因为这些动作的异常被内部 try/except 吞掉，run_workflows() 动作循环继续、document.save() 正常执行、WorkflowRun 正常创建——只有 WorkflowRun **不创建**的唯一情况是 ASSIGNMENT（非标题部分）、REMOVAL、MOVE_TO_TRASH 这三类动作抛出了未捕获异常

---

## 二、规则匹配流程（Rules Matching）

### 2.1 工作流查询与预取优化

定义于 [workflows/utils.py](file:///d:/fz/0601/solo-dogfeeding/code/112-paperless-ngx/src/documents/workflows/utils.py#L14-L128)：`get_workflows_for_trigger()`

这是每次 `run_workflows()` 调用的第一步，做了大量 ORM 级优化：

```python
Workflow.objects.filter(enabled=True, triggers__type=trigger_type)
    .prefetch_related(
        Prefetch("actions",
            queryset=WorkflowAction.objects
                .select_related(assign_correspondent, assign_document_type,
                                assign_storage_path, assign_owner, email, webhook)
                .prefetch_related(assign_tags, assign_view_users, ...  # 共 15 个 M2M
                )
                .annotate(has_assign_tags=Exists(...),        # 11 个布尔注解
                          has_assign_view_users=Exists(...), ...)
                .order_by("order", "pk")
        ),
        "triggers",
    )
    .order_by("order")
    .distinct()
```

- **select_related**：一次 JOIN 取出所有单值关联（email、webhook 配置等）
- **prefetch_related**：批量取出所有 M2M 关联（标签、用户、组等），避免 N+1
- **annotate(has_assign_tags=Exists(...))**：通过子查询生成布尔字段，让 `apply_assignment_to_document()` 无需再查 M2M 计数即可判断是否需要处理
- **distinct()**：`triggers__type=trigger_type` 是反向 JOIN 过滤，可能产生重复行

### 2.2 匹配入口函数

定义于 [matching.py](file:///d:/fz/0601/solo-dogfeeding/code/112-paperless-ngx/src/documents/matching.py#L642-L709)：`document_matches_workflow(document, workflow, trigger_type)`

**Workflow 级：OR 语义**，Trigger 级：AND 语义。

```
document_matches_workflow()
    ├── workflow.triggers.filter(type=trigger_type)
    │      .select_related(filter_mailrule, filter_has_document_type,
    │                         filter_has_correspondent, filter_has_storage_path,
    │                         schedule_date_custom_field)
    │      .prefetch_related(filter_has_tags, filter_has_all_tags, ...)  // 8 个 M2M
    │
    └── 对每个 trigger：
        ├── CONSUMPTION        → consumable_document_matches_workflow()
        ├── DOCUMENT_ADDED     → existing_document_matches_workflow()
        ├── DOCUMENT_UPDATED   → existing_document_matches_workflow()
        ├── SCHEDULED          → existing_document_matches_workflow()
        └── 任一返回 True → 立即 return True（bail early）
```

### 2.3 CONSUMPTION 阶段匹配

定义于 [matching.py](file:///d:/fz/0601/solo-dogfeeding/code/112-paperless-ngx/src/documents/matching.py#L285-L356)

按 **顺序逐一检查，全部通过才算匹配**（AND）：

1. **source 检查**：`int(document.source)` 必须在 `trigger.sources` 列表中
   - `sources` 是 MultiSelectField，存储为逗号分隔字符串如 `"1,2,3,4"`
2. **mailrule 检查**：若 `trigger.filter_mailrule_id` 非空，`document.mailrule_id` 必须等于它
   - mailrule_id 由 paperless_mail 模块在抓取附件时注入到 `ConsumableDocument`
3. **filename 检查**：`fnmatch(document.original_file.name.lower(), trigger.filter_filename.lower())`
   - 通配符 `*`、`?` 等，大小写不敏感
4. **path 检查**：优先 `document.original_path`（消费目录内的相对/绝对路径），否则用 `document.original_file`
   - 同样 fnmatch 匹配，但**区分大小写**

### 2.4 DOCUMENT_ADDED / DOCUMENT_UPDATED / SCHEDULED 阶段匹配

定义于 [matching.py](file:///d:/fz/0601/solo-dogfeeding/code/112-paperless-ngx/src/documents/matching.py#L359-L553)

按以下顺序逐项检查（AND 关系，任一不满足即短路返回）：

1. **内容算法匹配**：`trigger.matching_algorithm > MATCH_NONE` 时调用 `matches(trigger, document)`
2. **标签三维度检查**：先一次性 `set(document.tags.values_list("id", flat=True))` 取出文档标签 ID 集合，再分别做：
   - `filter_has_tags`：集合交集非空 → `bool(doc_ids & trigger_ids)`
   - `filter_has_all_tags`：trigger 集合是文档集合的子集 → `required_ids.issubset(doc_ids)`
   - `filter_has_not_tags`：交集为空 → `not (doc_ids & excluded_ids)`
3. **联系人三维度**：精确匹配 / 任意之一 / 排除
4. **文档类型三维度**：同上
5. **存储路径三维度**：同上
6. **自定义字段查询**：`CustomFieldQueryParser` 将 JSON 表达式编译为 Django Q + annotations，做 `Document.objects.filter(...).exists()`
7. **文件名 fnmatch**

### 2.5 内容匹配算法详解

定义于 [matching.py](file:///d:/fz/0601/solo-dogfeeding/code/112-paperless-ngx/src/documents/matching.py#L169-L263)：`matches()`

| 算法 | 正则 / 实现 |
|------|-------------|
| `MATCH_ALL` | 每个词 `re.search(r"\b{word}\b", content)`，全找到才 True |
| `MATCH_ANY` | 同上，任一找到即 True |
| `MATCH_LITERAL` | `re.search(r"\b{re.escape(match)}\b", content)` |
| `MATCH_REGEX` | `safe_regex_search()`（安全包装，防 ReDoS） |
| `MATCH_FUZZY` | `rapidfuzz.fuzz.partial_ratio(match, text, score_cutoff=90)` |
| `MATCH_AUTO` | 直接返回 False — 由 ML 分类器在信号 handler `set_correspondent()` 等处处理 |

**单词拆分器** `_split_match()` 支持双引号分组，如：
```
'  some random  words "with   quotes  " and   spaces'
→ ["some", "random", "words", "with\s+quotes", "and", "spaces"]
```
组内空格被替换为 `\s+` 以匹配任意空白。

### 2.6 定时工作流的预过滤

定义于 [matching.py](file:///d:/fz/0601/solo-dogfeeding/code/112-paperless-ngx/src/documents/matching.py#L556-L639)：`prefilter_documents_by_workflowtrigger()`

为避免对每篇文档逐一执行 Python 级的 `existing_document_matches_workflow()`，先在 SQL 层做 QuerySet 过滤：

- `filter_has_tags`：`documents.filter(tags__in=trigger_tags).distinct()`
- `filter_has_all_tags`：对每个 tag 做一次 `documents.filter(tags=tag)`，叠加多次 JOIN
- `filter_has_not_tags`：`documents.exclude(tags__in=excluded_tags)`
- 联系人/类型/路径：三种模式同上
- 自定义字段：同前 annotations + Q
- 文件名：`fnmatch_translate(pattern)` 转为正则后用 `original_filename__iregex=regex`

---

## 三、动作派发机制（Actions Dispatch）

### 3.1 执行入口：`run_workflows()`

定义于 [signals/handlers.py](file:///d:/fz/0601/solo-dogfeeding/code/112-paperless-ngx/src/documents/signals/handlers.py#L854-L999)

**函数签名**：
```python
run_workflows(
    trigger_type,             # 本次执行的触发类型
    document,                 # Document 或 ConsumableDocument
    workflow_to_run=None,     # 可选：跳过查询，直接跑指定 workflow（SCHEDULED 用）
    logging_group=None,       # UUID，用于日志关联
    overrides=None,           # 非 None 即进入 overrides 模式
    original_file=None,       # EMAIL / WEBHOOK 附件的文件路径
)
```

**完整执行流**：

```
run_workflows()
    │
    ├── 跳过版本文档（root_document_id is not None）
    ├── original_file 默认填充：
    │     overrides 模式 → document.original_file
    │     直接模式    → document.source_path
    │
    ├── workflows = get_workflows_for_trigger(trigger_type, workflow_to_run)
    │
    └── for workflow in workflows.order_by("order"):
          ├── 直接模式下：document.refresh_from_db()  // 刷新防止并发覆盖
          ├── 直接模式下：if document.is_deleted → break  // 已被其他 workflow 删除
          │
          ├── if matching.document_matches_workflow(document, workflow, trigger_type):
          │   │
          │   ├── has_move_to_trash_action = False
          │   │
          │   ├── for action in workflow.actions.order_by("order", "pk"):
          │   │   │
          │   │   ├── ASSIGNMENT:
          │   │   │     overrides 模式 → apply_assignment_to_overrides(action, overrides)
          │   │   │     直接模式    → apply_assignment_to_document(action, document, logging_group)
          │   │   │
          │   │   ├── REMOVAL:
          │   │   │     overrides 模式 → apply_removal_to_overrides(action, overrides)
          │   │   │     直接模式    → apply_removal_to_document(action, document)
          │   │   │
          │   │   ├── EMAIL:
          │   │   │     context = build_workflow_action_context(document, overrides)
          │   │   │     execute_email_action(action, document, context, logging_group,
          │   │   │                                    original_file, trigger_type)
          │   │   │
          │   │   ├── WEBHOOK:
          │   │   │     context = build_workflow_action_context(document, overrides)
          │   │   │     execute_webhook_action(action, document, context, logging_group,
          │   │   │                                      original_file)
          │   │   │       └── send_webhook.apply_async(...)  // 独立 Celery 任务
          │   │   │
          │   │   ├── PASSWORD_REMOVAL:
          │   │   │     ConsumableDocument → 挂 document_consumption_finished 信号
          │   │   │     Document          → 立即 bulk_edit.remove_password()
          │   │   │
          │   │   └── MOVE_TO_TRASH:
          │   │         has_move_to_trash_action = True  // 标记，延后执行
          │   │
          │   ├── 直接模式下：
          │   │     document.title = document.title[:128]
          │   │     document.save(update_fields=[
          │   │         "title", "correspondent", "document_type",
          │   │         "storage_path", "owner", "modified"
          │   │     ])
          │   │       └── 触发 post_save → update_filename_and_move_files()
          │   │
          │   ├── WorkflowRun.objects.create(  // ★ 创建运行记录
          │   │       workflow=workflow,
          │   │       type=trigger_type,
          │   │       document=document if not use_overrides else None
          │   │   )
          │   │
          │   └── if has_move_to_trash_action:  // ★ 最后执行
          │         execute_move_to_trash_action(action, document, logging_group)
          │
          └── overrides 模式下 return (overrides, "\n".join(messages))
```

### 3.2 双模式设计的差异汇总

| 维度 | overrides 模式 (CONSUMPTION) | 直接模式 (ADDED/UPDATED/SCHEDULED) |
|------|------------------------------|-----------------------------------|
| 判断标志 | `overrides is not None` | `overrides is None` |
| document 类型 | `ConsumableDocument` | `Document` (ORM 对象) |
| ASSIGNMENT/REMOVAL 操作对象 | `DocumentMetadataOverrides` 数据类 | 直接修改 `Document` 实例字段 |
| `refresh_from_db()` | ❌ | ✅ 每轮 workflow 前 |
| 软删除检查 | ❌ | ✅ 检查 `document.is_deleted` |
| 最终持久化 | overrides 返回给 Consumer 管线，后续由 `ConsumerPlugin` 统一写入 DB | `document.save(update_fields=[...])` 仅保存 6 个白名单字段 |
| WorkflowRun.document | `None`（文档未入库） | 实际 Document FK |
| MOVE_TO_TRASH 行为 | 删除临时文件 + `raise StopConsumeTaskError` | `document.delete()`（软删除） |
| 返回值 | `(overrides, messages_str)` | `None` |

### 3.3 ASSIGNMENT 动作详解

定义于 [workflows/mutations.py](file:///d:/fz/0601/solo-dogfeeding/code/112-paperless-ngx/src/documents/workflows/mutations.py#L16-L191)

**两种模式的关键差异**：

| 字段 | Document 模式 | Overrides 模式 |
|------|---------------|----------------|
| 标签 | `document.add_nested_tags()` → 自动添加祖先标签，立即写入 M2M 表 | `set(overrides.tag_ids) ∪ tag_ids ∪ ancestor_ids`，仅写内存列表 |
| 联系人/类型/路径/Owner | 直接赋值 FK，随 document.save() 落库 | 仅记录 ID 到 overrides.*_id 字段 |
| 标题 | `parse_w_workflow_placeholders()` 解析 Jinja2 模板，立即截取 [:128] | 原样保存模板字符串，Consumer 阶段再解析 |
| 权限 (view/change) | `set_permissions_for_object(merge=True)` → 立即 guardian 表写入 | 累积 user_id / group_id 列表到 overrides |
| 自定义字段 | `CustomFieldInstance.objects.update_or_create` 逐表写入 | 累积到 `overrides.custom_fields` dict |

> 标题模板解析时故意传空字符串给 title 参数，防止**递归替换**（用 title 解析出 title）。

### 3.4 REMOVAL 动作详解

定义于 [workflows/mutations.py](file:///d:/fz/0601/solo-dogfeeding/code/112-paperless-ngx/src/documents/workflows/mutations.py#L194-L354)

- **标签**：
  - `remove_all_tags=True` → `Document.tags.through.objects.filter(document=doc).clear()`（直接模式）；`overrides.tag_ids = None`（overrides 模式）
  - 否则：**同时移除标签及其所有后代标签**（`tag.get_descendants_pks()`）
- **联系人/类型/路径/Owner**：
  - `remove_all_*=True` → 设为 None
  - 否则：当 `document.correspondent` 在 `action.remove_correspondents` 中时才清空
- **权限**：
  - `remove_all_permissions=True` → 传空列表给 `set_permissions_for_object(merge=False)` 全清空
  - 否则：`remove_perm("view_document", user, document)` 逐条移除
- **自定义字段**：
  - `remove_all_custom_fields=True` → `CustomFieldInstance.objects.filter(document=doc).hard_delete()`（软删除模型的硬删）
  - 否则：按 `action.remove_custom_fields` 删除指定字段

### 3.5 EMAIL 动作

定义于 [workflows/actions.py](file:///d:/fz/0601/solo-dogfeeding/code/112-paperless-ngx/src/documents/workflows/actions.py#L87-L187)

**占位符解析器** `parse_w_workflow_placeholders()` 可识别的 Jinja2 变量：
`correspondent`, `document_type`, `owner_username`, `added`, `filename`, `current_filename`, `created`, `title`, `doc_url`, `id`

**附件选择逻辑**：
- `trigger_type ∈ {DOCUMENT_UPDATED, SCHEDULED}` 且 `isinstance(document, Document)` → 用 `document.source_path`
- 否则 → 用 `original_file` 参数（消费阶段传入）
- 两者都缺失 → 不附加

邮件通过 `documents.mail.send_email()` **同步**发送（阻塞当前工作流执行线程）。

### 3.6 动作异常处理与失败传播、WEBHOOK 异步任务

#### 核心事实：每个动作函数内部的 try/except 差异

`run_workflows()` 的动作循环本身**没有 try/catch**（见 [signals/handlers.py:L918-L993](file:///d:/fz/0601/solo-dogfeeding/code/112-paperless-ngx/src/documents/signals/handlers.py#L918-L993)），但**各动作函数内部的异常处理策略完全不同**，这决定了哪些动作失败会中止工作流、哪些只会静默写日志：

| 动作类型 | 内部是否有 try/except | 异常是否向上抛出 | 对 run_workflows() 的影响 |
|----------|:---------------------:|:---------------:|--------------------------|
| ASSIGNMENT | 仅标题解析处有（mutations.py:L42-L60） | **部分抛出** | 标题解析异常只写日志；标签/联系人/权限/自定义字段等 DB 操作异常会向上抛出 |
| REMOVAL | 无 | ✅ 全部向上抛出 | 任一部分异常都会中止工作流 |
| EMAIL | **整体有 try/except**（actions.py:L141-L187） | ❌ **只写 logger.exception，不抛出** | 失败不影响后续动作，不中止工作流 |
| WEBHOOK（同步阶段） | **整体有 try/except**（actions.py:L197-L273） | ❌ **只写 logger.exception，不抛出** | params/headers/读文件/Broker 入队失败全不影响宿主 |
| PASSWORD_REMOVAL | 每个密码 try/except ValueError；全部失败只 logger.error | ❌ 不抛出 | 失败不影响后续动作 |
| MOVE_TO_TRASH | 无；对 ConsumableDocument 明确 `raise StopConsumeTaskError` | ✅ 全部向上抛出 | 执行必然中止（软删 or 抛异常中断消费） |

> **代码证据**：
> - EMAIL 顶层 [actions.py:L141-L187](file:///d:/fz/0601/solo-dogfeeding/code/112-paperless-ngx/src/documents/workflows/actions.py#L141-L187)：`try: ... except Exception as e: logger.exception(...)` —— 吞掉所有异常
> - WEBHOOK 顶层 [actions.py:L197-L273](file:///d:/fz/0601/solo-dogfeeding/code/112-paperless-ngx/src/documents/workflows/actions.py#L197-L273)：同样结构，含 params 解析（L201-L220）和 headers 解析（L237-L243）的内层 try/except
> - PASSWORD_REMOVAL [actions.py:L318-L345](file:///d:/fz/0601/solo-dogfeeding/code/112-paperless-ngx/src/documents/workflows/actions.py#L318-L345)：只 warning/error 日志，从不 raise

#### 对工作流整体状态的影响矩阵

只有 **ASSIGNMENT（非标题部分）、REMOVAL、MOVE_TO_TRASH** 三类动作的失败会触发以下连锁反应：

```
动作异常抛出
    ├── run_workflows() 动作循环立即中止，后续动作不再执行
    ├── document.save(update_fields=[6 个白名单])  ← 若已执行到此处之前则已落库，否则不执行
    ├── WorkflowRun.objects.create()  ← 永远不执行（位于动作循环之后）→ 本次执行无审计记录
    └── 异常向上抛到调用者：
          ├── Celery 宿主 → task_failure 信号 → PaperlessTask.status=FAILURE，result_data 记录 traceback
          └── 同步 API 线程 → Django 中间件捕获 → HTTP 500（无 PaperlessTask）
```

而 **EMAIL / WEBHOOK（同步阶段）/ PASSWORD_REMOVAL** 三类动作失败：

```
动作异常被内部 try/except 捕获
    ├── logger.exception() / logger.error() 写日志
    ├── run_workflows() 动作循环继续，后续动作正常执行
    ├── document.save() 正常执行
    ├── WorkflowRun.objects.create() 正常创建 → 有审计记录，但日志里能看到错误
    └── 宿主 PaperlessTask 状态完全不受影响（仍为 SUCCESS）
```

#### WEBHOOK 动作定义与异步执行链路

定义于 [workflows/actions.py](file:///d:/fz/0601/solo-dogfeeding/code/112-paperless-ngx/src/documents/workflows/actions.py#L190-L273) 与 [workflows/webhooks.py](file:///d:/fz/0601/solo-dogfeeding/code/112-paperless-ngx/src/documents/workflows/webhooks.py)

`send_webhook` Celery 任务配置：
```python
@shared_task(
    retry_backoff=True,
    autoretry_for=(httpx.HTTPStatusError,),  # 仅对 HTTP 状态码错误自动重试（4xx/5xx）
    max_retries=3,
    throws=(httpx.HTTPError,),               # HTTPError 标记为"预期异常"，Celery 不记 FAILURE
)
```

**WEBHOOK 完整执行链路（三个阶段，各阶段失败归属完全独立）**：

```
  ┌───────────────────────────────────────────────────────────────────────┐
  │  阶段 1：同步准备阶段（在 run_workflows() 内部，宿主任务线程内）           │
  │  execute_webhook_action() [顶层 try/except 吞掉所有异常]                  │
  │    ├── params 占位符解析 → 内层 try/except → logger.error                │
  │    ├── body 占位符解析 → 外层 try/except → logger.exception              │
  │    ├── headers 解析 → 内层 try/except → logger.error                     │
  │    ├── include_document 时 original_file.open("rb") 读文件                │
  │    │                                    → 外层 try/except → logger.exception
  │    └── send_webhook.apply_async(...) 入队                                │
  │         └── Broker 不可用等 → 外层 try/except → logger.exception         │
  │                                                                           │
  │  ✅ 本阶段**所有异常只写日志，不向上抛出**                                   │
  │  ✅ 不影响宿主 PaperlessTask 状态，不中止后续动作                            │
  │  ✅ WorkflowRun 正常创建                                                   │
  └───────────────────────────────┬───────────────────────────────────────┘
                                  │
                                  ▼ (入队成功，异步到独立 Worker 线程)
  ┌───────────────────────────────────────────────────────────────────────┐
  │  阶段 2：异步投递阶段（独立 send_webhook 任务，与宿主完全解耦）             │
  │  send_webhook()                                                           │
  │    ├── validate_outbound_http_url() → URL 不合法 → logger.warning + raise │
  │    ├── WebhookTransport → DNS 解析/公网 IP 校验 → ConnectError → raise    │
  │    ├── httpx.Client(timeout=5s).post(...).raise_for_status()              │
  │    │     ├── HTTPStatusError (4xx/5xx) → autoretry_for 触发，最多 3 次     │
  │    │     ├── 其他 httpx.HTTPError → throws= 指定，Celery 不记 FAILURE      │
  │    │     └── 其他异常 → task_failure 信号，但因不在 TRACKED_TASKS          │
  │    │                           → 无 PaperlessTask 记录                     │
  │    └── 所有错误只写入 paperless.workflows.webhooks logger                  │
  │                                                                           │
  │  ❌ 不在 TRACKED_TASKS → 不创建独立 PaperlessTask                          │
  │  ❌ 与宿主 PaperlessTask 状态完全无关（宿主早已继续前进）                    │
  │  ❌ 前端任务列表不可见                                                      │
  └───────────────────────────────────────────────────────────────────────┘

  ┌───────────────────────────────────────────────────────────────────────┐
  │  阶段 3：对 WorkflowRun / PaperlessTask 的最终影响                        │
  │  ┌─────────────────────────────────────────────────────────────────┐   │
  │  │ 失败阶段            WorkflowRun?   宿主 PaperlessTask?   独立Task?│   │
  │  ├─────────────────────────────────────────────────────────────────┤   │
  │  │ 阶段 1 同步准备         ✅ 创建         不受影响(SUCCESS)     ❌    │   │
  │  │ 阶段 2 异步投递失败     ✅ 创建(已在阶段1后)  完全无关         ❌    │   │
  │  │ 全部成功               ✅ 创建         ✅ SUCCESS           ❌    │   │
  │  └─────────────────────────────────────────────────────────────────┘   │
  └───────────────────────────────────────────────────────────────────────┘
```

> **关键事实修正**：之前认为"WEBHOOK 同步阶段异常会让宿主 PaperlessTask FAILURE"是错误的——`execute_webhook_action()` 顶层 try/except 吞掉所有异常，宿主任务完全感知不到 webhook 同步阶段的失败。
>
> 同样，EMAIL 发送失败、PASSWORD_REMOVAL 密码不对也只会写日志，不影响宿主任务状态。
>
> **唯一能让宿主任务 FAILURE 的工作流动作异常来源**：ASSIGNMENT 的标签/权限/自定义字段 DB 操作失败、REMOVAL 的任意失败、MOVE_TO_TRASH 对消费阶段抛 StopConsumeTaskError。

### 3.7 PASSWORD_REMOVAL 动作

定义于 [workflows/actions.py](file:///d:/fz/0601/solo-dogfeeding/code/112-paperless-ngx/src/documents/workflows/actions.py#L276-L345)

**两阶段执行设计**：

```
CONSUMPTION 阶段（isinstance(document, ConsumableDocument)）:
    └── document_consumption_finished.connect(handler)
           handler 里用真正入库的 Document 再执行一次本函数
           执行完立即 signal.disconnect(handler) 避免泄漏

DOCUMENT_ADDED/UPDATED/SCHEDULED 阶段（isinstance(document, Document)）:
    └── 按逗号/换行分割 passwords 列表，依次尝试：
          documents.bulk_edit.remove_password([doc_id], password=pw, ...)
          成功 return，失败尝试下一个；全部失败记 error 日志
```

### 3.8 MOVE_TO_TRASH 动作的延迟执行

定义于 [workflows/actions.py](file:///d:/fz/0601/solo-dogfeeding/code/112-paperless-ngx/src/documents/workflows/actions.py#L348-L375)

**为什么不跟其他动作一起顺序执行？**

在 [signals/handlers.py](file:///d:/fz/0601/solo-dogfeeding/code/112-paperless-ngx/src/documents/signals/handlers.py#L917-L993) 中故意只设标志位，等所有动作、save()、WorkflowRun 都完成后才执行：

```python
for action in workflow.actions.order_by("order", "pk"):
    ...
    elif action.type == WorkflowAction.WorkflowActionType.MOVE_TO_TRASH:
        has_move_to_trash_action = True  # 仅标记

# 所有动作执行完毕
if not use_overrides:
    document.save(update_fields=[...])  # 先把 ASSIGNMENT/REMOVAL 改动落库

WorkflowRun.objects.create(...)  # 再记审计（如果先删文档，这步 FK 约束会失败）

if has_move_to_trash_action:
    execute_move_to_trash_action(action, document, logging_group)  # 最后删
```

---

## 四、任务状态变化流程（Task Status Transitions）

### 4.1 PaperlessTask 状态机

定义于 [models.py](file:///d:/fz/0601/solo-dogfeeding/code/112-paperless-ngx/src/documents/models.py#L664-L800)

```
                    before_task_publish
                          │   (Celery 任务发布到 Broker 时)
                          ▼
                 ┌─────────────────┐
                 │    PENDING      │ （task_id, task_type, trigger_source,
                 └────────┬────────┘    input_data, owner_id 已入库）
                          │ task_prerun
                          │   (Worker 取到任务、开始执行前)
                          ▼
                 ┌─────────────────┐
                 │    STARTED      │ （date_started 入库）
                 └───────┬─────────┘
                 ┌───────┴─────────┐
                 │                 │
        task_postrun         task_failure
         SUCCESS/FAILURE      FAILURE
                 │                 │
                 ▼                 ▼
           ┌──────────┐      ┌──────────┐
           │ SUCCESS  │      │ FAILURE  │ date_done, duration_seconds,
           └──────────┘      └──────────┘ wait_time_seconds, result_data

     task_revoked 可在 PENDING 或 STARTED 阶段触发 → REVOKED（date_done）
```

### 4.2 TRACKED_TASKS：哪些 Celery 任务会被跟踪

定义于 [signals/handlers.py](file:///d:/fz/0601/solo-dogfeeding/code/112-paperless-ngx/src/documents/signals/handlers.py#L1005-L1017)

```python
TRACKED_TASKS: dict[str, PaperlessTask.TaskType] = {
    "documents.tasks.consume_file":                                PaperlessTask.TaskType.CONSUME_FILE,
    "documents.tasks.train_classifier":                            PaperlessTask.TaskType.TRAIN_CLASSIFIER,
    "documents.tasks.sanity_check":                                PaperlessTask.TaskType.SANITY_CHECK,
    "documents.tasks.llmindex_index":                              PaperlessTask.TaskType.LLM_INDEX,
    "documents.tasks.empty_trash":                                 PaperlessTask.TaskType.EMPTY_TRASH,
    "documents.tasks.check_scheduled_workflows":                   PaperlessTask.TaskType.CHECK_WORKFLOWS,
    "paperless_mail.tasks.process_mail_accounts":                  PaperlessTask.TaskType.MAIL_FETCH,
    "documents.tasks.bulk_update_documents":                       PaperlessTask.TaskType.BULK_UPDATE,
    "documents.tasks.update_document_content_maybe_archive_file":  PaperlessTask.TaskType.REPROCESS_DOCUMENT,
    "documents.tasks.build_share_link_bundle":                     PaperlessTask.TaskType.BUILD_SHARE_LINK,
    "documents.bulk_edit.delete":                                  PaperlessTask.TaskType.BULK_DELETE,
}
```

> **关键观察**：
> - `send_webhook` **不在**此表中 → webhook 执行没有独立的 PaperlessTask 跟踪
> - Workflow 本身不是 Celery 任务，它的执行依附于宿主任务（CONSUME_FILE / CHECK_WORKFLOWS / BULK_UPDATE 等）

### 4.3 状态跟踪实现细节

定义于 [signals/handlers.py](file:///d:/fz/0601/solo-dogfeeding/code/112-paperless-ngx/src/documents/signals/handlers.py#L1103-L1313)

| Celery 信号 | 触发时机 | PaperlessTask 操作 |
|-------------|----------|-------------------|
| `before_task_publish` | Publisher 端，任务入 Broker 前 | `PaperlessTask.objects.create()` → 状态 PENDING；解析 task_kwargs 填充 input_data（filename、mime_type、overrides 等）和 owner_id |
| `task_prerun` | Worker 端，任务执行前 | `update(status=STARTED, date_started=now)`；先 `close_old_connections()` |
| `task_postrun` | Worker 端，任务正常/异常结束后 | 若新状态是 FAILURE 直接 return（交给 task_failure）；否则写入 SUCCESS、date_done、duration_seconds、wait_time_seconds、result_data；特殊：若 retval 含 `duplicate_of` 键，则覆写状态为 FAILURE |
| `task_failure` | Worker 端，任务抛出异常 | 写入 FAILURE + result_data{error_type, error_message, traceback[:5000]} |
| `task_revoked` | 任务被取消（不论执行中与否） | 写入 REVOKED + date_done |

`_CELERY_STATE_TO_STATUS` 映射：`SUCCESS → SUCCESS`, `FAILURE → FAILURE`, `REVOKED → REVOKED`（其余都视作 FAILURE）。

### 4.3.1 非 Celery 场景：同步 API 线程没有 PaperlessTask

PaperlessTask 的创建完全依赖 Celery 信号链（`before_task_publish → task_prerun → task_postrun / task_failure`）。如果 `run_workflows()` 在 Django 请求/响应线程中同步执行（DOCUMENT_UPDATED 的 views.py 入口），则：

- ❌ 没有 `before_task_publish` 信号 → **没有 PaperlessTask 行被创建**
- ❌ 没有 `task_prerun` / `task_postrun` / `task_failure`
- **ASSIGNMENT（非标题部分）/ REMOVAL / MOVE_TO_TRASH 抛异常** → Django 中间件捕获 → HTTP 500 + Django `paperless.*` logger
- **EMAIL / WEBHOOK（同步阶段）/ PASSWORD_REMOVAL / ASSIGNMENT 标题解析失败** → 被内部 try/except 吞掉只写日志，HTTP 返回正常（通常 200）
- 前端任务列表**完全看不到这次工作流执行的任何痕迹**（无论成功还是失败）
- 工作流执行成功或"部分失败但异常被吞"的唯一证据：WorkflowRun 记录

> 同步 API 线程的工作流执行可观测性是系统的一处空白：无论成功/部分失败，都无独立 PaperlessTask 记录，需结合 WorkflowRun 表与应用日志（`paperless.workflows.actions` / `paperless.workflows.mutations` / `paperless.workflows.webhooks`）综合排查。

---

### 4.4 TriggerSource：七种触发来源与代码位置

定义于 [models.py](file:///d:/fz/0601/solo-dogfeeding/code/112-paperless-ngx/src/documents/models.py#L701-L710)

**所有 consume_file / 其他任务的 `apply_async()` 调用必须通过 `headers={"trigger_source": ...}` 传递来源**，由 `before_task_publish_handler._determine_trigger_source()` 读取；缺失则默认 `MANUAL`。

| TriggerSource | 设置代码位置 | 说明 |
|---------------|-------------|------|
| `SCHEDULED` | Celery Beat 配置 `parse_beat_schedule()` 自动注入（任务本身由 Beat 发出） | check_scheduled_workflows / process_mail_accounts 等 |
| `WEB_UI` | [views.py:1940](file:///d:/fz/0601/solo-dogfeeding/code/112-paperless-ngx/src/documents/views.py#L1938-L1941)（文档新版本），[views.py:3153](file:///d:/fz/0601/solo-dogfeeding/code/112-paperless-ngx/src/documents/views.py#L3149-L3157)（from_webui=True 的文档上传） | 前端 Web 界面操作 |
| `API_UPLOAD` | [views.py:3155](file:///d:/fz/0601/solo-dogfeeding/code/112-paperless-ngx/src/documents/views.py#L3149-L3157)（from_webui=False） | `/api/documents/post_document/` 直接上传 |
| `FOLDER_CONSUME` | [document_consumer.py:350](file:///d:/fz/0601/solo-dogfeeding/code/112-paperless-ngx/src/documents/management/commands/document_consumer.py#L342-L351) | `document_consumer` 管理命令轮询消费目录 |
| `EMAIL_CONSUME` | [mail.py:901](file:///d:/fz/0601/solo-dogfeeding/code/112-paperless-ngx/src/paperless_mail/mail.py#L896-L903)（附件），[mail.py:1001](file:///d:/fz/0601/solo-dogfeeding/code/112-paperless-ngx/src/paperless_mail/mail.py#L998-L1001)（.eml 原信） | 邮件抓取任务产出的文档 |
| `MANUAL` | `_determine_trigger_source()` 默认值 | 未显式设置 headers 的调用 |
| `SYSTEM` | [bulk_edit.py](file:///d:/fz/0601/solo-dogfeeding/code/112-paperless-ngx/src/documents/bulk_edit.py) 所有 `bulk_update_documents.apply_async(headers={"trigger_source": TriggerSource.SYSTEM})`（见 127、149、169、198 行等），以及 [tasks.py:623](file:///d:/fz/0601/solo-dogfeeding/code/112-paperless-ngx/src/documents/tasks.py#L621-L624) `update_document_parent_tags()` | 系统内部触发的批量更新 |

---

## 五、五种触发场景的完整执行链路

### 5.1 CONSUMPTION（消费开始时）

**入口**：[consumer.py](file:///d:/fz/0601/solo-dogfeeding/code/112-paperless-ngx/src/documents/consumer.py#L67-L87) `WorkflowTriggerPlugin.run()`（Consumer 插件管线的固定执行位置）

**宿主任务**：`documents.tasks.consume_file`（PaperlessTask.TaskType.CONSUME_FILE）

```
┌───────────────────────────────────────────────────────────────────────┐
│  0. 任务提交（按来源不同有 5 处）                                        │
│     ├── FOLDER_CONSUME: document_consumer.py 管理命令                  │
│     ├── WEB_UI / API_UPLOAD: views.py /api/documents/post_document/   │
│     ├── EMAIL_CONSUME: paperless_mail/mail.py handle_message          │
│     └── WEB_UI: views.py 文档新版本上传                                │
│         consume_file.apply_async(headers={"trigger_source": ...})      │
│             │
│             ▼ before_task_publish 信号                                 │
│         PaperlessTask(status=PENDING) 入库                             │
└───────────────────────────────┬───────────────────────────────────────┘
                                │ task_prerun
                                ▼
                    Worker 端开始执行 consume_file()
                                │
                                ▼
                    ConsumerPlugin.setup() → 插件管线初始化
                                │
                                ▼
               ┌──────────────────────────────────────┐
               │ WorkflowTriggerPlugin.run()          │  ← 固定第一个执行
               │   overrides, msg = run_workflows(    │
               │       trigger_type=CONSUMPTION,      │
               │       overrides=DocumentMetadataOverrides())
               └──────────────┬───────────────────────┘
                              │
                              ▼
              run_workflows(trigger_type=CONSUMPTION)
                  ├── 仅检查 source / mailrule / filename / path
                  ├── ASSIGNMENT / REMOVAL 写入 overrides 内存对象
                  ├── EMAIL 同步发送
                  ├── WEBHOOK → send_webhook.apply_async()
                  │              └── 独立任务，不创建 PaperlessTask
                  ├── PASSWORD_REMOVAL → 挂 document_consumption_finished 信号
                  ├── MOVE_TO_TRASH → 删文件 + raise StopConsumeTaskError
                  └── return (overrides, messages)
                              │
                              ▼
                    Consumer 后续流程（分类、OCR、归档等）
                    将 overrides 中的元数据应用到入库 Document
                              │
                              ▼
                    Document 保存到 DB（transaction 内）
                              │
                              ▼
                    document_consumption_finished.send(...)
                              │
              ┌───────────────┴───────────────┐
              │ 见下一节 5.2 DOCUMENT_ADDED    │
              └───────────────────────────────┘
```

### 5.2 DOCUMENT_ADDED（文档添加完成）

**入口**：[apps.py](file:///d:/fz/0601/solo-dogfeeding/code/112-paperless-ngx/src/documents/apps.py#L24-L31) `ready()` 中建立的信号连接

**宿主任务**：仍然是 `consume_file`（同一 Celery 任务内同步执行）

```
document_consumption_finished.send(sender=ConsumerPlugin, document=doc, ...)
    │
    ├── [1] add_inbox_tags           → 添加所有 is_inbox_tag=True 的标签
    ├── [2] set_correspondent        → ML 分类器 + 规则匹配
    ├── [3] set_document_type        → ML 分类器 + 规则匹配
    ├── [4] set_tags                 → ML 分类器 + 规则匹配
    ├── [5] set_storage_path         → ML 分类器 + 规则匹配
    ├── [6] add_to_index             → 全文搜索引擎写入
    │
    ├── [7] run_workflows_added()
    │     └── run_workflows(trigger_type=DOCUMENT_ADDED, document=doc)
    │           ├── existing_document_matches_workflow() 完整匹配
    │           ├── ASSIGNMENT/REMOVAL 直接改 Document + save()
    │           ├── EMAIL / WEBHOOK / PASSWORD_REMOVAL / MOVE_TO_TRASH
    │           └── WorkflowRun.objects.create(document=doc, type=DOCUMENT_ADDED)
    │
    └── [8] add_or_update_document_in_llm_index
```

**信号连接顺序即执行顺序**（apps.py 中 `connect()` 的先后决定）。

### 5.3 DOCUMENT_UPDATED（文档更新后）

**入口**：[apps.py](file:///d:/fz/0601/solo-dogfeeding/code/112-paperless-ngx/src/documents/apps.py#L32-L33)

**宿主任务**：
- `documents.tasks.bulk_update_documents`（最常见，SYSTEM 来源）
- 或直接在 views.py 处理 API 请求的线程（同步发送）

`document_updated` 信号的发出点（SCHEDULED **不**发此信号，见 5.4 节说明）：

| 发出位置 | 场景 |
|---------|------|
| [tasks.py:260](file:///d:/fz/0601/solo-dogfeeding/code/112-paperless-ngx/src/documents/tasks.py#L258-L265) `bulk_update_documents()` | 批量编辑后（set_correspondent、set_tags、rotate、merge/split 等都会先 DB update，再 apply_async 本任务） |
| [views.py:1178](file:///d:/fz/0601/solo-dogfeeding/code/112-paperless-ngx/src/documents/views.py#L1178-L1181) | PATCH /api/documents/{id}/ 后 |
| [views.py:2046](file:///d:/fz/0601/solo-dogfeeding/code/112-paperless-ngx/src/documents/views.py#L2046-L2049) | 删除版本后 |
| [views.py:2119](file:///d:/fz/0601/solo-dogfeeding/code/112-paperless-ngx/src/documents/views.py#L2119-L2122) | 修改版本标签后 |

执行流：
```
document_updated.send(sender=..., document=doc)
    ├── run_workflows_updated()
    │     └── run_workflows(trigger_type=DOCUMENT_UPDATED, document=doc)
    │           └── (同 DOCUMENT_ADDED 逻辑)
    └── send_websocket_document_updated()
          └── DocumentsStatusManager 通过 Channels WebSocket 推前端
```

**bulk_edit → workflow 的完整链路示例**（以 `set_correspondent` 为例）：

```
API PATCH /api/documents/bulk_edit/ {method: "set_correspondent", ...}
    │
    ▼ views.py BulkEditViewSet._execute_document_action()
    │
    ▼ bulk_edit.set_correspondent(doc_ids, correspondent)
        ├── Document.objects.filter(...).update(correspondent=correspondent)  // 直写 DB
        └── bulk_update_documents.apply_async(
                kwargs={"document_ids": affected_docs},
                headers={"trigger_source": PaperlessTask.TriggerSource.SYSTEM}
            )
            │
            ▼ before_task_publish
                PaperlessTask(type=BULK_UPDATE, status=PENDING, trigger_source=SYSTEM)
            │
            ▼ Worker 执行 bulk_update_documents(document_ids)
                ├── for doc in documents:
                │     clear_document_caches(doc.pk)
                │     document_updated.send(sender=None, document=doc)  ← 触发 workflow
                │     post_save.send(Document, instance=doc)
                │         └── update_filename_and_move_files()
                └── 批量更新 search index + LLM index
```

### 5.4 SCHEDULED（定时触发）

**入口**：Celery Beat 调度 → [tasks.py](file:///d:/fz/0601/solo-dogfeeding/code/112-paperless-ngx/src/documents/tasks.py#L435-L573) `check_scheduled_workflows()`

**宿主任务**：`documents.tasks.check_scheduled_workflows`（PaperlessTask.TaskType.CHECK_WORKFLOWS）

**Beat 配置**：定义于 [custom.py](file:///d:/fz/0601/solo-dogfeeding/code/112-paperless-ngx/src/paperless/settings/custom.py#L135-L145) `parse_beat_schedule()`：
```python
{
    "name": "Check and run scheduled workflows",
    "env_key": "PAPERLESS_WORKFLOW_SCHEDULED_TASK_CRON",
    "env_default": "5 */1 * * *",  # 默认每小时第 5 分钟执行
    "task": "documents.tasks.check_scheduled_workflows",
    "options": {"expires": 59 * 60},  # 下一次调度前 1 分钟过期，避免堆积
}
```

**完整执行链路**：

```
Celery Beat（按 cron "5 */1 * * *"）
    │
    ▼ 发布 check_scheduled_workflows 任务（trigger_source=SCHEDULED）
    │   before_task_publish → PaperlessTask(PENDING, type=CHECK_WORKFLOWS)
    │
    ▼ Worker 执行 check_scheduled_workflows()
        │
        ├── scheduled_workflows = get_workflows_for_trigger(SCHEDULED)
        │
        └── for workflow in scheduled_workflows:
            └── for trigger in workflow.triggers.filter(type=SCHEDULED):
                │
                ├── ① 根据 schedule_date_field 选出 "日期已到达阈值" 的文档
                │   threshold = now - timedelta(days=schedule_offset_days)
                │   ├── ADDED    → Document.objects.filter(root_document__isnull=True, added__lte=threshold)
                │   ├── CREATED  → Document.objects.filter(root_document__isnull=True, created__lte=threshold)
                │   ├── MODIFIED → Document.objects.filter(root_document__isnull=True, modified__lte=threshold)
                │   └── CUSTOM_FIELD:
                │         CustomFieldInstance.objects.filter(
                │             field=schedule_date_custom_field,
                │             value_date__lte=threshold,
                │             value_date__gte=now - 365days  // 扫描上限 1 年防全表
                │         )
                │         再用 Python 校验 make_aware(value_date + offset) <= now
                │
                ├── ② prefilter_documents_by_workflowtrigger(documents, trigger)
                │      （SQL 层预过滤：标签、联系人、类型、路径、CF 查询、文件名）
                │
                └── ③ for document in documents:
                      │
                      ├── ④ WorkflowRun 去重检查（精确查询条件）：
                      │   workflow_runs = WorkflowRun.objects.filter(
                      │       document=document,
                      │       type=SCHEDULED,
                      │       workflow=workflow
                      │   ).order_by("-run_at")
                      │
                      │   ├── 非周期 (schedule_is_recurring=False):
                      │   │     if workflow_runs.exists() → continue
                      │   │
                      │   └── 周期 (schedule_is_recurring=True):
                      │         if workflow_runs.first().run_at
                      │            > now - timedelta(days=schedule_recurring_interval_days)
                      │         → continue
                      │
                      ├── ⑤ run_workflows(trigger_type=SCHEDULED,
                      │                    workflow_to_run=workflow,  // 跳过查询
                      │                    document=document)
                      │     └── existing_document_matches_workflow() 最终判定
                      │         → 依次执行动作（ASSIGNMENT/REMOVAL/EMAIL/WEBHOOK/PASSWORD_REMOVAL）
                      │         → document.save(update_fields=[6 个白名单字段])
                      │         → WorkflowRun.objects.create(workflow=workflow,
                      │                                       type=SCHEDULED,
                      │                                       document=document)
                      │         → 若有 MOVE_TO_TRASH 则最后执行
                      │
                      └── ⑥ send_websocket_document_updated(sender=None, document=doc)
                            （代码注释明确："Scheduled workflows dont send document_updated signal,
                             so send a websocket update here to ensure clients are updated"
                             —— 只做纯前端 WebSocket 通知，不触发任何后端逻辑，
                             不会引发 DOCUMENT_UPDATED 工作流的二次执行）
```

**关键去重逻辑说明**：
- WorkflowRun 的过滤字段是 **`(document_id, type=SCHEDULED, workflow_id)` 三元组**，同一文档通过不同 workflow 的 SCHEDULED 触发互不影响
- 非周期：只要历史上**有过任意一次**成功执行就永久跳过
- 周期：上次成功执行的 `run_at` 距离现在小于 `schedule_recurring_interval_days` 才跳过；超过则再次执行（允许多次）

**SCHEDULED 与 DOCUMENT_UPDATED 的边界**：
SCHEDULED 工作流通过 `run_workflows(workflow_to_run=workflow)` 传入单条 workflow，不会调用 get_workflows_for_trigger 扫描其他类型工作流；收尾只调 send_websocket_document_updated 纯前端通知，不发 document_updated 信号——**两条链路完全隔离，不会因 SCHEDULED 的执行间接触发 DOCUMENT_UPDATED 工作流**。

### 5.5 四种触发类型的统一总览（含执行上下文与失败可见性）

| 维度 | CONSUMPTION | DOCUMENT_ADDED | DOCUMENT_UPDATED (Celery 宿主) | DOCUMENT_UPDATED (同步 API 线程) | SCHEDULED |
|------|:-----------:|:--------------:|:-------------------------------:|:--------------------------------:|:---------:|
| **触发入口** | Consumer 插件 `WorkflowTriggerPlugin.run()` | `document_consumption_finished` 信号 | `bulk_update_documents` 任务内 `document_updated.send()` | views.py PATCH/删版本/改版本标签内 `document_updated.send()` | Celery Beat `check_scheduled_workflows()` |
| **执行线程/宿主** | Celery Worker | Celery Worker（同一 consume_file 任务内同步执行） | Celery Worker（bulk_update_documents 任务内同步执行） | Django 请求/响应线程 | Celery Worker |
| **宿主 PaperlessTask 类型** | `CONSUME_FILE` | `CONSUME_FILE` | `BULK_UPDATE` | **无**（不走 Celery） | `CHECK_WORKFLOWS` |
| **典型 TriggerSource** | `FOLDER_CONSUME`、`EMAIL_CONSUME`、`WEB_UI`、`API_UPLOAD` | 同上 | `SYSTEM` | 无 | `SCHEDULED` |
| **匹配对象** | `ConsumableDocument` | `Document` | `Document` | `Document` | `Document` |
| **run_workflows 模式** | overrides 模式 | 直接模式 | 直接模式 | 直接模式 | 直接模式 |
| **WorkflowRun.document** | `NULL` | Document FK | Document FK | Document FK | Document FK |
| **WorkflowRun 去重** | ❌ | ❌ | ❌ | ❌ | ✅（三元组去重 + 周期性间隔） |
| **ASSIGNMENT(非标题)/REMOVAL/MOVE 异常** | 宿主 FAILURE + result_data | 同左 | 同左 | HTTP 500 + Django 日志 | 宿主 FAILURE + result_data |
| **WorkflowRun 是否创建（上述异常时）** | ❌ | ❌ | ❌ | ❌ | ❌ |
| **EMAIL/WEBHOOK(同步)/PASSWORD/标题解析失败** | ✅ 宿主 SUCCESS（异常被吞只写日志） | 同左 | 同左 | ✅ HTTP 正常返回（异常被吞只写日志） | 同左 |
| **WorkflowRun 是否创建（上述失败时）** | ✅（有审计记录） | ✅ | ✅ | ✅ | ✅ |
| **WEBHOOK 异步投递阶段失败** | logger only（不影响宿主，无独立 PaperlessTask） | 同左 | 同左 | 同左（宿主早已返回响应） | 同左 |
| **收尾是否发 document_updated** | 否 | 否 | 否（自身就是触发源） | 否（自身就是触发源） | **否**（只发 send_websocket_document_updated 纯前端通知） |
| **间接触发下一工作流类型** | DOCUMENT_ADDED | 无 | 无 | 无 | 无（明确不触发） |

#### DOCUMENT_UPDATED 的两个执行上下文对比

DOCUMENT_UPDATED 工作流的唯一触发入口是 `document_updated` 信号，但该信号有两大类发出点，执行上下文完全不同：

- **Celery 宿主（SYSTEM 来源）**：所有 `bulk_edit.*` 函数（set_correspondent / set_tags / rotate / merge / split 等）先 `QuerySet.update()` 直写 DB，再 `bulk_update_documents.apply_async(headers={"trigger_source": SYSTEM})`，Worker 收到任务后在 `bulk_update_documents()` 内部对每篇文档 `document_updated.send()`。此时工作流运行在 Celery Worker 线程，ASSIGNMENT/REMOVAL/MOVE 失败会写 PaperlessTask.FAILURE；EMAIL/WEBHOOK/PASSWORD 失败只写日志，宿主仍为 SUCCESS。
- **同步 API 线程（用户操作）**：[views.py:L1178](file:///d:/fz/0601/solo-dogfeeding/code/112-paperless-ngx/src/documents/views.py#L1178-L1181) 单文档 PATCH、[views.py:L2046](file:///d:/fz/0601/solo-dogfeeding/code/112-paperless-ngx/src/documents/views.py#L2046-L2049) 删版本、[views.py:L2119](file:///d:/fz/0601/solo-dogfeeding/code/112-paperless-ngx/src/documents/views.py#L2119-L2122) 改版本标签。此时工作流运行在 Django 请求/响应线程，ASSIGNMENT/REMOVAL/MOVE 失败导致 HTTP 500，**不会创建任何 PaperlessTask**；EMAIL/WEBHOOK/PASSWORD 失败被内部吞掉，HTTP 正常返回。

---

## 六、关键设计要点

### 6.1 工作流匹配的 OR + AND 组合逻辑

- **Workflow 级 OR**：`document_matches_workflow()` 中只要有一个 trigger 返回 True 就 return True（bail early）
- **Trigger 级 AND**：`consumable_document_matches_workflow()` 和 `existing_document_matches_workflow()` 中所有过滤条件依次检查，任一不满足立即返回 False

### 6.2 get_workflows_for_trigger 的预取优化

见 [workflows/utils.py](file:///d:/fz/0601/solo-dogfeeding/code/112-paperless-ngx/src/documents/workflows/utils.py#L23-L103)：

- `annotate(has_assign_tags=Exists(WorkflowAction.assign_tags.through.objects.filter(...)))`：在 SQL 层为每个 action 生成 11 个布尔字段，让 mutations 层无需额外查询即可判断"这个 action 有没有 assign_tags 需要处理"
- `Prefetch("actions", queryset=annotated_actions.order_by("order", "pk"))`：保证 action 执行顺序与 UI 配置一致
- 全部 prefetch + select_related：一次查询取出 workflow 全部关联数据，整个工作流执行过程不会再产生额外 SQL

### 6.3 MOVE_TO_TRASH 的延迟执行

该动作安排在**所有其他动作完成、Document 保存、WorkflowRun 创建之后**才执行：
1. WorkflowRun 先落库（FK 约束要求 document 存在），审计记录不会因文档删除而丢失
2. EMAIL / WEBHOOK 等通知类动作先执行
3. ASSIGNMENT / REMOVAL 的变更先持久化

### 6.4 document.save() 的字段白名单

见 [signals/handlers.py](file:///d:/fz/0601/solo-dogfeeding/code/112-paperless-ngx/src/documents/signals/handlers.py#L975-L984)：
```python
document.save(update_fields=[
    "title", "correspondent", "document_type",
    "storage_path", "owner", "modified"
])
```
刻意排除 `filename` / `archive_filename`。因为 `document.save()` 会触发 `post_save` → `update_filename_and_move_files()`，后者可能已在 DB 中写入了新的 filename。如果此处带上内存中的旧值，会覆盖新值造成 DB 指向旧路径而文件已在新位置（见 issue #12386）。

### 6.5 并发安全

- 每轮 workflow 迭代前 `document.refresh_from_db()` 刷新数据（bulk_update_documents 等场景可能多进程并行处理同一文档）
- 检查 `document.is_deleted`：若文档已被上一轮或另一任务软删，立即 break 退出整个循环
- 文件操作通过 `FileLock(settings.MEDIA_LOCK)` 全局序列化
- `run_workflows()` 入口处就跳过 `root_document_id is not None` 的版本文档，避免对同一逻辑文档重复执行

### 6.6 Webhook 安全与可观测性权衡

- **安全**：`WebhookTransport` 做 DNS rebinding + 内网 IP 校验，URL scheme/port 白名单，禁止 302 跳转
- **自动重试**：`autoretry_for=(HTTPStatusError,), max_retries=3, retry_backoff=True`
- **可观测性缺失**：`send_webhook` 不在 `TRACKED_TASKS`，前端看不到它的执行状态；只能通过 `paperless.workflows.webhooks` logger 的日志排查

### 6.7 PASSWORD_REMOVAL 的两阶段设计

消费阶段文档尚未入库，PDF 文件还在临时目录、`Document.id` 不存在，无法调用 `bulk_edit.remove_password()`。因此通过挂接 `document_consumption_finished` 信号延迟到文档入库后同一触发链里再执行，执行完立即 disconnect 避免信号泄漏。

### 6.8 SCHEDULED 的三层过滤性能优化

1. **日期初筛**（SQL，按索引）：`added/created/modified <= threshold`
2. **prefilter_documents_by_workflowtrigger**（SQL）：标签、联系人、类型、路径、CF、文件名等全部用 QuerySet 过滤
3. **existing_document_matches_workflow**（Python 级，含内容算法匹配）：仅对候选集逐篇执行

加上 WorkflowRun 去重，避免对同一文档反复执行正则/模糊匹配等高成本操作。
