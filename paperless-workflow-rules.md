# Paperless-ngx Workflow 规则引擎与触发器深度分析

## 一、核心数据模型

### 1.1 Workflow（工作流）

定义于 [models.py](file:///d:/fz/0601/solo-dogfeeding/code/112-paperless-ngx/src/documents/models.py#L1828-L1850)

| 字段 | 类型 | 说明 |
|------|------|------|
| `name` | CharField | 工作流名称，唯一 |
| `order` | SmallIntegerField | 执行顺序，默认 0 |
| `triggers` | ManyToManyField(WorkflowTrigger) | 触发器集合，任意一个匹配即可触发 |
| `actions` | ManyToManyField(WorkflowAction) | 动作集合，按 `order, pk` 顺序执行 |
| `enabled` | BooleanField | 是否启用 |

### 1.2 WorkflowTrigger（触发器）

定义于 [models.py](file:///d:/fz/0601/solo-dogfeeding/code/112-paperless-ngx/src/documents/models.py#L1243-L1467)

#### 触发器类型 WorkflowTriggerType

| 值 | 名称 | 触发时机 |
|----|------|----------|
| 1 | `CONSUMPTION` | 文档消费开始时（文件尚未入库） |
| 2 | `DOCUMENT_ADDED` | 文档添加完成后（已入库） |
| 3 | `DOCUMENT_UPDATED` | 文档更新后 |
| 4 | `SCHEDULED` | 定时/周期性触发 |

#### 匹配模式 WorkflowTriggerMatching

| 值 | 算法 | 说明 |
|----|------|------|
| 0 | `NONE` | 不启用内容匹配 |
| 1 | `ANY` | 任意单词出现即匹配 |
| 2 | `ALL` | 所有单词都出现才匹配 |
| 3 | `LITERAL` | 精确字符串匹配 |
| 4 | `REGEX` | 正则表达式匹配 |
| 5 | `FUZZY` | 模糊匹配（rapidfuzz partial_ratio >= 90） |

#### 触发器过滤条件

| 过滤维度 | 字段 | 说明 |
|----------|------|------|
| 文档来源 | `sources` | MultiSelectField：ConsumeFolder / ApiUpload / MailFetch / WebUI |
| 邮件规则 | `filter_mailrule` | 仅匹配来自指定 MailRule 的文档 |
| 文件名 | `filter_filename` | fnmatch 通配符，大小写不敏感 |
| 文件路径 | `filter_path` | fnmatch 通配符匹配路径 |
| 内容 | `match` + `matching_algorithm` | 文档文本内容匹配 |
| 标签（任意） | `filter_has_tags` | 文档含有任一标签即匹配 |
| 标签（全部） | `filter_has_all_tags` | 文档必须含全部标签 |
| 标签（排除） | `filter_has_not_tags` | 文档不能含有任一标签 |
| 文档类型（单一） | `filter_has_document_type` | 指定文档类型 |
| 文档类型（任意） | `filter_has_any_document_types` | 其中之一即可 |
| 文档类型（排除） | `filter_has_not_document_types` | 排除这些类型 |
| 联系人（单一） | `filter_has_correspondent` | 指定联系人 |
| 联系人（任意） | `filter_has_any_correspondents` | 其中之一即可 |
| 联系人（排除） | `filter_has_not_correspondents` | 排除这些联系人 |
| 存储路径（单一） | `filter_has_storage_path` | 指定存储路径 |
| 存储路径（任意） | `filter_has_any_storage_paths` | 其中之一即可 |
| 存储路径（排除） | `filter_has_not_storage_paths` | 排除这些路径 |
| 自定义字段 | `filter_custom_field_query` | JSON 编码的自定义字段查询表达式 |
| 调度日期字段 | `schedule_date_field` | added / created / modified / custom_field |
| 调度偏移天数 | `schedule_offset_days` | 正负整数，相对目标日期的偏移 |
| 是否周期 | `schedule_is_recurring` | 是否重复触发 |
| 周期间隔 | `schedule_recurring_interval_days` | 重复触发的间隔天数 |

### 1.3 WorkflowAction（动作）

定义于 [models.py](file:///d:/fz/0601/solo-dogfeeding/code/112-paperless-ngx/src/documents/models.py#L1556-L1825)

#### 动作类型 WorkflowActionType

| 值 | 名称 | 作用对象 | 说明 |
|----|------|----------|------|
| 1 | `ASSIGNMENT` | Document / Overrides | 分配元数据：标题、标签、类型、联系人、路径、Owner、权限、自定义字段 |
| 2 | `REMOVAL` | Document / Overrides | 移除元数据：可移除全部或指定项 |
| 3 | `EMAIL` | - | 发送通知邮件，可附加文档 |
| 4 | `WEBHOOK` | - | 异步发送 Webhook HTTP 请求 |
| 5 | `PASSWORD_REMOVAL` | Document | 尝试移除 PDF 密码保护 |
| 6 | `MOVE_TO_TRASH` | Document / ConsumableDocument | 软删除（移到回收站）或中断消费 |

### 1.4 WorkflowRun（工作流运行记录）

定义于 [models.py](file:///d:/fz/0601/solo-dogfeeding/code/112-paperless-ngx/src/documents/models.py#L1853-L1886)

记录每次工作流的实际执行，用于：
- 防止非周期性定时工作流重复执行
- 控制周期性工作流的触发间隔

---

## 二、规则匹配流程（Rules Matching）

### 2.1 匹配入口函数

核心匹配函数定义于 [matching.py](file:///d:/fz/0601/solo-dogfeeding/code/112-paperless-ngx/src/documents/matching.py#L642-L709)：`document_matches_workflow(document, workflow, trigger_type)`

**匹配逻辑：OR 语义** —— 工作流的多个触发器中 **任意一个匹配** 即判定工作流匹配（bail early）。

```
document_matches_workflow()
    ├── 过滤 workflow.triggers，仅保留 type == trigger_type 的触发器
    └── 对每个 trigger 调用具体匹配函数：
        ├── CONSUMPTION → consumable_document_matches_workflow()
        ├── DOCUMENT_ADDED / DOCUMENT_UPDATED / SCHEDULED → existing_document_matches_workflow()
        └── 只要有一个 trigger 返回 True，立即返回 True
```

### 2.2 CONSUMPTION 阶段匹配

定义于 [matching.py](file:///d:/fz/0601/solo-dogfeeding/code/112-paperless-ngx/src/documents/matching.py#L285-L356)：`consumable_document_matches_workflow()`

该阶段文档尚未入库，只检查 **消费时可获取的信息**：

1. **文档来源检查**：`document.source` 必须在 `trigger.sources` 中
2. **邮件规则检查**：若 `trigger.filter_mailrule` 非空，`document.mailrule_id` 必须匹配
3. **文件名检查**：`document.original_file.name` 与 `trigger.filter_filename` 做 fnmatch 匹配
4. **路径检查**：优先使用 `document.original_path`，否则用 `document.original_file`，与 `trigger.filter_path` 做 fnmatch 匹配

> **所有条件为 AND 关系**，必须全部满足才返回 True。

### 2.3 DOCUMENT_ADDED / DOCUMENT_UPDATED / SCHEDULED 阶段匹配

定义于 [matching.py](file:///d:/fz/0601/solo-dogfeeding/code/112-paperless-ngx/src/documents/matching.py#L359-L553)：`existing_document_matches_workflow()`

该阶段文档已入库，可访问完整元数据，按以下顺序逐项检查（AND 关系）：

1. **内容匹配**：若 `matching_algorithm > MATCH_NONE`，调用 `matches()` 对 `document.get_effective_content()` 执行算法匹配
2. **标签检查**：
   - `filter_has_tags`：文档标签集合与 trigger 指定集合的交集非空
   - `filter_has_all_tags`：trigger 指定集合是文档标签集合的子集
   - `filter_has_not_tags`：交集必须为空
3. **联系人检查**：`filter_has_correspondent`（精确匹配）、`filter_has_any_correspondents`（任一）、`filter_has_not_correspondents`（排除）
4. **文档类型检查**：同上三种模式
5. **存储路径检查**：同上三种模式
6. **自定义字段查询**：解析 `filter_custom_field_query` 为 Django Q 对象，执行 DB 查询
7. **文件名检查**：`document.original_filename` 与 `trigger.filter_filename` 做 fnmatch

### 2.4 内容匹配算法详解

定义于 [matching.py](file:///d:/fz/0601/solo-dogfeeding/code/112-paperless-ngx/src/documents/matching.py#L169-L263)：`matches()`

| 算法 | 实现方式 |
|------|----------|
| `MATCH_ALL` | 将 match 字符串拆分为单词（支持双引号分组），每个单词用 `\b{word}\b` 正则搜索，全部找到则匹配 |
| `MATCH_ANY` | 同上，任一单词找到即匹配 |
| `MATCH_LITERAL` | `re.escape()` 后用 `\b...\b` 正则搜索 |
| `MATCH_REGEX` | 使用 `safe_regex_search()`，支持 re.IGNORECASE |
| `MATCH_FUZZY` | rapidfuzz `fuzz.partial_ratio()`，阈值 90 |
| `MATCH_AUTO` | 由分类器 ML 模型在别处处理，此处返回 False |
| `MATCH_NONE` | 直接返回 False |

### 2.5 定时工作流的预过滤

定义于 [matching.py](file:///d:/fz/0601/solo-dogfeeding/code/112-paperless-ngx/src/documents/matching.py#L556-L639)：`prefilter_documents_by_workflowtrigger()`

为避免定时工作流对每篇文档逐一调用 `existing_document_matches_workflow()`，先通过 **数据库 QuerySet 过滤** 大幅缩小候选集。支持的预过滤维度：
- 标签（has / has_all / has_not）
- 联系人、文档类型、存储路径（三种模式）
- 自定义字段查询
- 文件名（iregex 基于 fnmatch 翻译而来）

---

## 三、动作派发机制（Actions Dispatch）

### 3.1 执行入口：`run_workflows()`

定义于 [signals/handlers.py](file:///d:/fz/0601/solo-dogfeeding/code/112-paperless-ngx/src/documents/signals/handlers.py#L854-L999)

```
run_workflows(trigger_type, document, workflow_to_run=None, logging_group, overrides, original_file)
    │
    ├── get_workflows_for_trigger()          // 查询匹配的工作流（含预取 + 注解优化）
    │
    └── 遍历每个 workflow：
        ├── 若非 overrides 模式：refresh_from_db() 防止并发写入覆盖
        ├── 检查 document 是否被软删除，若是则 break
        ├── 调用 matching.document_matches_workflow() 判定
        ├── 若匹配：
        │   ├── 遍历 workflow.actions.order_by("order", "pk")
        │   │   ├── ASSIGNMENT  → apply_assignment_to_overrides() / apply_assignment_to_document()
        │   │   ├── REMOVAL     → apply_removal_to_overrides() / apply_removal_to_document()
        │   │   ├── EMAIL       → execute_email_action()
        │   │   ├── WEBHOOK     → execute_webhook_action()
        │   │   ├── PASSWORD_REMOVAL → execute_password_removal_action()
        │   │   └── MOVE_TO_TRASH → has_move_to_trash_action = True（延后执行）
        │   │
        │   ├── 若非 overrides 模式：document.save(update_fields=[title, correspondent, ...])
        │   ├── 创建 WorkflowRun 记录
        │   └── 若 has_move_to_trash_action：execute_move_to_trash_action()
        │
        └── overrides 模式下返回 (overrides, messages)
```

### 3.2 双模式设计

| 模式 | 触发场景 | 操作对象 | 返回值 |
|------|----------|----------|--------|
| **overrides 模式** | CONSUMPTION 阶段 | `DocumentMetadataOverrides` 对象 | `(overrides, messages)` |
| **直接模式** | DOCUMENT_ADDED / DOCUMENT_UPDATED / SCHEDULED | 实际的 `Document` ORM 对象 | None |

overrides 模式下，动作并不直接修改数据库，而是累积到 `DocumentMetadataOverrides` 数据对象中，由后续消费流程统一应用。

### 3.3 ASSIGNMENT 动作详解

定义于 [workflows/mutations.py](file:///d:/fz/0601/solo-dogfeeding/code/112-paperless-ngx/src/documents/workflows/mutations.py#L16-L191)

| 字段 | Document 模式 | Overrides 模式 |
|------|---------------|----------------|
| 标签 | `document.add_nested_tags()`（自动添加祖先标签） | 累积到 `overrides.tag_ids`，合并去重 |
| 联系人 / 文档类型 / 存储路径 / Owner | 直接赋值外键 | 记录 ID 到 overrides |
| 标题 | 支持 Jinja2 模板占位符解析，截断至 128 字符 | 直接保存模板字符串 |
| 权限（view/change users/groups） | `set_permissions_for_object(merge=True)` | 累积 ID 列表，合并去重 |
| 自定义字段 | `CustomFieldInstance.objects.update_or_create` | 累积到 `overrides.custom_fields` 字典 |

### 3.4 REMOVAL 动作详解

定义于 [workflows/mutations.py](file:///d:/fz/0601/solo-dogfeeding/code/112-paperless-ngx/src/documents/workflows/mutations.py#L194-L354)

- **标签**：`remove_all_tags` 清空；否则移除指定标签及其所有后代标签
- **联系人/类型/路径/Owner**：`remove_all_*` 设为 None；否则匹配则移除
- **权限**：`remove_all_permissions` 清空；否则用 `remove_perm()` 逐项移除
- **自定义字段**：`remove_all_custom_fields` hard_delete 全部；否则删除指定字段的实例

### 3.5 EMAIL 动作

定义于 [workflows/actions.py](file:///d:/fz/0601/solo-dogfeeding/code/112-paperless-ngx/src/documents/workflows/actions.py#L87-L187)：`execute_email_action()`

- Subject / Body 支持 Jinja2 占位符解析（`parse_w_workflow_placeholders()`）
- `include_document=True` 时附加文档文件（DOCUMENT_UPDATED / SCHEDULED 用 `document.source_path`，消费阶段用 `original_file`）
- 通过 `documents.mail.send_email()` 同步发送

### 3.6 WEBHOOK 动作

定义于 [workflows/actions.py](file:///d:/fz/0601/solo-dogfeeding/code/112-paperless-ngx/src/documents/workflows/actions.py#L190-L273)：`execute_webhook_action()`

- **异步执行**：通过 `send_webhook.apply_async()` 提交 Celery 任务
- 参数/Body 均支持占位符解析
- 可配置 Headers、是否包含文档附件、是否以 JSON 形式发送
- 安全防护：[workflows/webhooks.py](file:///d:/fz/0601/solo-dogfeeding/code/112-paperless-ngx/src/documents/workflows/webhooks.py) 中 `WebhookTransport` 做 SSRF 防护（DNS rebinding、内网 IP 校验）

### 3.7 PASSWORD_REMOVAL 动作

定义于 [workflows/actions.py](file:///d:/fz/0601/solo-dogfeeding/code/112-paperless-ngx/src/documents/workflows/actions.py#L276-L345)

- 消费阶段（ConsumableDocument）：挂载 `document_consumption_finished` 信号，待文档入库后再执行
- 已入库文档：直接调用 `documents.bulk_edit.remove_password()` 逐个尝试密码

### 3.8 MOVE_TO_TRASH 动作

定义于 [workflows/actions.py](file:///d:/fz/0601/solo-dogfeeding/code/112-paperless-ngx/src/documents/workflows/actions.py#L348-L375)

- **特殊执行时机**：该动作不在遍历 actions 时立即执行，而是先标记 `has_move_to_trash_action = True`，等所有动作完成、document.save()、WorkflowRun 创建后 **最后执行**
- 已入库文档：`document.delete()`（软删除，SoftDeleteModel）
- 消费阶段：删除临时文件 + 抛出 `StopConsumeTaskError` 中断整个消费流程

---

## 四、任务状态变化流程（Task Status Transitions）

### 4.1 PaperlessTask 状态机

定义于 [models.py](file:///d:/fz/0601/solo-dogfeeding/code/112-paperless-ngx/src/documents/models.py#L664-L800)

```
                    before_task_publish
                          │
                          ▼
                 ┌─────────────────┐
                 │    PENDING      │ （任务已发布到 Broker）
                 └────────┬────────┘
                          │ task_prerun
                          ▼
                 ┌─────────────────┐
                 │    STARTED      │ （Worker 开始执行）
                 └───────┬─────────┘
                 ┌───────┴─────────┐
                 │                 │
        task_postrun         task_failure
         SUCCESS/FAILURE      FAILURE
                 │                 │
                 ▼                 ▼
           ┌──────────┐      ┌──────────┐
           │ SUCCESS  │      │ FAILURE  │
           └──────────┘      └──────────┘

     task_revoked 可在 PENDING 或 STARTED 阶段触发 → REVOKED
```

### 4.2 状态跟踪实现

定义于 [signals/handlers.py](file:///d:/fz/0601/solo-dogfeeding/code/112-paperless-ngx/src/documents/signals/handlers.py#L1001-L1320)，基于 **Celery 信号** 驱动：

| Celery 信号 | PaperlessTask 状态变化 | 附加字段 |
|-------------|----------------------|----------|
| `before_task_publish` | → `PENDING` | `task_id`, `task_type`, `trigger_source`, `input_data`, `owner_id` |
| `task_prerun` | → `STARTED` | `date_started` |
| `task_postrun` (state=SUCCESS) | → `SUCCESS` | `date_done`, `duration_seconds`, `wait_time_seconds`, `result_data` |
| `task_postrun` (retval含 duplicate_of) | → `FAILURE` | 标记重复消费为失败 |
| `task_failure` | → `FAILURE` | `result_data` (error_type, error_message, traceback[:5000]) |
| `task_revoked` | → `REVOKED` | `date_done` |

> **注意**：`task_postrun` 对 FAILURE 状态直接 return，避免与 `task_failure` 重复写入。

### 4.3 被跟踪的任务类型

| TaskType | 对应 Celery 任务 |
|----------|-----------------|
| `CONSUME_FILE` | `documents.tasks.consume_file` |
| `TRAIN_CLASSIFIER` | `documents.tasks.train_classifier` |
| `SANITY_CHECK` | `documents.tasks.sanity_check` |
| `LLM_INDEX` | `documents.tasks.llmindex_index` |
| `EMPTY_TRASH` | `documents.tasks.empty_trash` |
| `CHECK_WORKFLOWS` | `documents.tasks.check_scheduled_workflows` |
| `MAIL_FETCH` | `paperless_mail.tasks.process_mail_accounts` |
| `BULK_UPDATE` | `documents.tasks.bulk_update_documents` |
| `REPROCESS_DOCUMENT` | `documents.tasks.update_document_content_maybe_archive_file` |
| `BUILD_SHARE_LINK` | `documents.tasks.build_share_link_bundle` |
| `BULK_DELETE` | `documents.bulk_edit.delete` |

### 4.4 触发来源 TriggerSource

| 值 | 说明 |
|----|------|
| `SCHEDULED` | Celery Beat 定时调度 |
| `WEB_UI` | Web 界面上传 |
| `API_UPLOAD` | REST API 上传 |
| `FOLDER_CONSUME` | Consume Folder 轮询 |
| `EMAIL_CONSUME` | 邮件附件抓取 |
| `MANUAL` | 其他手动触发（默认值） |
| `SYSTEM` | 系统内部触发（如标签层级变更引发的批量更新） |

---

## 五、四种触发场景的完整执行链路

### 5.1 CONSUMPTION（消费开始时）

**入口**：[consumer.py](file:///d:/fz/0601/solo-dogfeeding/code/112-paperless-ngx/src/documents/consumer.py#L67-L87) `WorkflowTriggerPlugin.run()`

```
consume_file Celery 任务
    └── Consumer 插件管线执行
        └── WorkflowTriggerPlugin.run()
            └── run_workflows(trigger_type=CONSUMPTION, overrides=DocumentMetadataOverrides())
                ├── 仅匹配 source / mailrule / filename / path（无内容、标签等）
                ├── ASSIGNMENT / REMOVAL 动作写入 overrides
                ├── EMAIL / WEBHOOK 立即执行
                ├── PASSWORD_REMOVAL 挂载 document_consumption_finished 信号延迟执行
                └── MOVE_TO_TRASH 删除文件 + 抛 StopConsumeTaskError 中断消费
    └── overrides 中的元数据在后续消费流程中被应用
```

### 5.2 DOCUMENT_ADDED（文档添加完成）

**入口**：[apps.py](file:///d:/fz/0601/solo-dogfeeding/code/112-paperless-ngx/src/documents/apps.py#L24-L31) 信号连接

```
document_consumption_finished 信号
    ├── add_inbox_tags
    ├── set_correspondent / set_document_type / set_tags / set_storage_path  // ML 自动分类
    ├── add_to_index
    ├── run_workflows_added()
    │   └── run_workflows(trigger_type=DOCUMENT_ADDED)
    │       ├── 完整匹配：内容 + 所有元数据过滤器
    │       ├── ASSIGNMENT / REMOVAL 直接修改 Document 并 save()
    │       ├── 其他动作同前
    │       └── 保存后触发 post_save → update_filename_and_move_files（可能移动文件）
    └── add_or_update_document_in_llm_index
```

### 5.3 DOCUMENT_UPDATED（文档更新后）

**入口**：[apps.py](file:///d:/fz/0601/solo-dogfeeding/code/112-paperless-ngx/src/documents/apps.py#L32-L33)

```
document_updated 信号
    ├── run_workflows_updated()
    │   └── run_workflows(trigger_type=DOCUMENT_UPDATED)
    └── send_websocket_document_updated()
```

`document_updated` 信号通常由 API 更新、批量编辑等操作显式发出。

### 5.4 SCHEDULED（定时触发）

**入口**：[tasks.py](file:///d:/fz/0601/solo-dogfeeding/code/112-paperless-ngx/src/documents/tasks.py#L435-L573) `check_scheduled_workflows()` Celery Beat 任务

```
Celery Beat（周期性）
    └── check_scheduled_workflows()
        └── 遍历每个 SCHEDULED 类型的 enabled workflow：
            └── 遍历每个 schedule trigger：
                ├── 根据 schedule_date_field 筛选 Document：
                │   ├── ADDED    → Document.objects.filter(added <= now - offset)
                │   ├── CREATED  → Document.objects.filter(created <= now - offset)
                │   ├── MODIFIED → Document.objects.filter(modified <= now - offset)
                │   └── CUSTOM_FIELD → 基于 CustomFieldInstance.value_date 计算
                ├── prefilter_documents_by_workflowtrigger()  // DB 级预过滤
                └── 遍历每篇候选文档：
                    ├── 检查 WorkflowRun 历史：
                    │   ├── 非周期：已运行过则跳过
                    │   └── 周期：距上次运行 < schedule_recurring_interval_days 则跳过
                    ├── run_workflows(trigger_type=SCHEDULED, workflow_to_run=workflow)
                    └── send_websocket_document_updated()
```

---

## 六、关键设计要点

### 6.1 工作流匹配的 OR + AND 组合逻辑

- **Workflow 级 OR**：一个 Workflow 有多个 Trigger，**任意一个 Trigger 匹配即触发**
- **Trigger 级 AND**：一个 Trigger 内的多个过滤条件（source、filename、tags...）**必须全部满足**

### 6.2 MOVE_TO_TRASH 的延迟执行

该动作被刻意安排在 **所有其他动作完成、Document 保存、WorkflowRun 记录创建之后** 才执行，确保：
1. 审计记录（WorkflowRun）不会因文档被删而丢失
2. 其他动作（如发送通知邮件）有机会先执行

### 6.3 document.save() 的字段白名单

[signals/handlers.py](file:///d:/fz/0601/solo-dogfeeding/code/112-paperless-ngx/src/documents/signals/handlers.py#L975-L984) 中显式列出 `update_fields`：
```python
document.save(update_fields=["title", "correspondent", "document_type", "storage_path", "owner", "modified"])
```
刻意排除 `filename` / `archive_filename`，避免与并发的 `update_filename_and_move_files`（由 post_save 触发）产生竞态条件（参考 issue #12386）。

### 6.4 并发安全

- 每轮 workflow 迭代前 `document.refresh_from_db()` 刷新数据
- 检查 `document.is_deleted` 防止已被其他 workflow 删除的文档继续处理
- 文件操作通过 `FileLock(settings.MEDIA_LOCK)` 全局序列化

### 6.5 Webhook 安全

- SSRF 防护：`WebhookTransport` 解析 DNS 后校验 IP 是否为公网（可配置）
- URL 白名单：仅允许配置的 schemes 和 ports
- 自动重试：Celery task 配置 `autoretry_for=(HTTPStatusError,), max_retries=3`
