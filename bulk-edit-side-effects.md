# Bulk Edit 与任务副作用协作机制分析

本文档精确梳理 Paperless-ngx 中批量编辑（Bulk Edit）的协作逻辑：元数据变更 → 后台任务调度 → 副作用执行 → 部分失败边界，重点纠正事务边界、异常冒泡、重试策略与已生效副作用之间的真实关系。

---

## 一、整体架构分层

```
API 入口层           核心逻辑层              异步任务层              信号副作用层
┌──────────────┐   ┌──────────────┐       ┌──────────────┐       ┌──────────────────┐
│  Views.py    │──▶│ bulk_edit.py│──────▶│  tasks.py  │──────▶│  signals/      │
│ (BulkEditView, │   │ (各 edit fn) │       │ (Celery)   │       │  handlers.py   │
│  Rotate/...  )│   │            │       │            │       │                │
└──────────────┘   └──────────────┘       └──────────────┘       └──────────────────┘
                       │                      │                      │
                       ▼                      ▼                      ▼
              DB 同步写（多数无事务）    搜索索引/LLM索引          工作流/WebSocket/文件重命名
```

核心事实：**请求层未开启 `ATOMIC_REQUESTS`，BulkEditView.post() 外部无事务包裹。**

---

## 二、API 入口层

### 2.1 主入口：[BulkEditView.post()](file:///d:/fz/0601/solo-dogfeeding/code/61-paperless-ngx/src/documents/views.py#L2821-L2916)

流程：序列化校验 → 权限校验 → 审计快照 → `method(documents, **parameters)` → 审计写 LogEntry → 返回 200。

**关键事实：外部没有 `@transaction.atomic`，settings 中也没有开启 `ATOMIC_REQUESTS`。** 所以 `method()` 内部每一条独立的 ORM 语句都是**隐式自动提交**的。如果 method 内部中途抛异常：
- 已执行的 SQL 语句**不会回滚**
- `bulk_update_documents.apply_async()` 如果还没执行就不会被调度
- 异常冒泡到外层 `try/except`，返回 HTTP 400

```python
# views.py 中的异常捕获（无外层事务）
try:
    modified_field = self.MODIFIED_FIELD_BY_METHOD.get(method.__name__, None)
    if settings.AUDIT_LOG_ENABLED and modified_field:
        old_documents = {...}
    result = method(documents, **parameters)   # ← 内部语句各自提交
    if settings.AUDIT_LOG_ENABLED and modified_field:
        for doc in new_documents:
            LogEntry.objects.log_create(...)   # ← 如果 method 抛错，这里也不会执行
    return Response({"result": result})
except Exception as e:
    return HttpResponseBadRequest(...)          # ← 已提交的 SQL 不回滚
```

### 2.2 其他专用端点（Rotate/Merge/Delete/Reprocess 等）

通过 `_execute_document_action()` 调用，结构相同，同样无外层事务。见 [views.py#L2733-L2776](file:///d:/fz/0601/solo-dogfeeding/code/61-paperless-ngx/src/documents/views.py#L2733-L2776)。

---

## 三、核心逻辑层（bulk_edit.py）——精确的事务边界

各函数内部事务情况差异极大，下面逐函数说明。

### 3.1 modify_tags：唯一使用 transaction.atomic() + ignore_conflicts=True 的元数据操作

[modify_tags](file:///d:/fz/0601/solo-dogfeeding/code/61-paperless-ngx/src/documents/bulk_edit.py#L226-L284) 是所有元数据编辑中**唯一**使用 `transaction.atomic()` 的函数，同时其 `bulk_create` 开启了 `ignore_conflicts=True`：

```python
with transaction.atomic():
    if expanded_remove_tags:
        DocumentTagRelationship.objects.filter(...).delete()     # ①
    if expanded_add_tags:
        existing_pairs = set(DocumentTagRelationship.objects.filter(...).values_list(...))
        to_create = [DocumentTagRelationship(...) for (doc, tag) not in existing_pairs]
        if to_create:
            DocumentTagRelationship.objects.bulk_create(
                to_create,
                ignore_conflicts=True,   # ② 静默跳过唯一约束冲突的行
            )
# 事务提交后才调度异步任务
if affected_docs:
    bulk_update_documents.apply_async(...)
```

**关键事实**：
- `ignore_conflicts=True` 意味着即使并发请求已经插入了相同的 (doc, tag) 对，数据库层面不会抛 IntegrityError，重复行被静默跳过，`bulk_create` 返回成功。
- 只有非唯一约束类的数据库错误才会抛异常 → 触发 `transaction.atomic()` 回滚 → DELETE 和 INSERT 全部撤销，`apply_async` 也不会调度。
- 这是所有批量元数据编辑中**最干净的回滚保证**。

与 `add_tag`（单标签接口）的关键对比：`add_tag` 的 `bulk_create` **没有** `ignore_conflicts=True`，并发冲突会抛 IntegrityError（虽然由于单条 SQL 原子性也零行插入，但异常会冒泡到前端）。

### 3.2 set_correspondent / set_document_type / set_storage_path：单条 SQL 原子

以 [set_correspondent](file:///d:/fz/0601/solo-dogfeeding/code/61-paperless-ngx/src/documents/bulk_edit.py#L111-L131) 为例：

```python
qs = Document.objects.filter(Q(id__in=doc_ids) & ~Q(correspondent=correspondent))
affected_docs = list(qs.values_list("pk", flat=True))   # SELECT
qs.update(correspondent=correspondent)                   # ② 单条 UPDATE，原子
bulk_update_documents.apply_async(...)                   # ③
return "OK"
```

**异常行为**：
- 若 ② UPDATE 抛错（数据库层），没有已修改数据，返回 400。
- 若 ② 成功、③ `apply_async` 抛错（如 broker 不可用）：**DB 中 correspondent 已更新且提交，但后台副作用任务没有被调度**。前端拿到 400，但文档属性实际已变。

### 3.3 add_tag / remove_tag：一次 bulk_create（无 ignore_conflicts）+ 无事务

以 [add_tag](file:///d:/fz/0601/solo-dogfeeding/code/61-paperless-ngx/src/documents/bulk_edit.py#L176-L202) 为例：

```python
tag_obj = Tag.objects.get(pk=tag)
tags_to_add = [tag_obj, *tag_obj.get_ancestors()]   # tag 本身 + 所有祖先

DocumentTagRelationship = Document.tags.through
to_create = []
affected_docs: set[int] = set()

for t in tags_to_add:                        # 每个 tag（仅 SELECT，无写操作）
    qs = Document.objects.filter(Q(id__in=doc_ids) & ~Q(tags__id=t.id))
    doc_ids_missing_tag = list(qs.values_list("pk", flat=True))
    affected_docs.update(doc_ids_missing_tag)
    to_create.extend(                         # 仅构建 Python 列表
        DocumentTagRelationship(document_id=doc, tag_id=t.id)
        for doc in doc_ids_missing_tag
    )

if to_create:
    DocumentTagRelationship.objects.bulk_create(to_create)   # ① 一次 bulk INSERT，无 ignore_conflicts

if affected_docs:
    bulk_update_documents.apply_async(...)                    # ②
```

**关键事实**：
- 整个 for 循环只做 SELECT 和 Python 列表构建，**不写 DB**。
- 只有一次 `bulk_create(to_create)`，把 tag 及其所有祖先的 (doc, tag) 对**一次性**插入。
- **没有 `ignore_conflicts=True`**。单条 INSERT 语句在数据库层面原子：任何一行违反唯一约束（如并发冲突），整条语句失败，**零行插入**。

**异常行为**：
- 若 ① `bulk_create` 抛 IntegrityError（如并发请求在 SELECT 和 INSERT 之间插入了相同的 (doc, tag)）：**零行插入**，DB 状态与调用前完全一致（无事务，也不需要回滚）。异常冒泡到视图，返回 400。
- 若 ① 成功、② `apply_async` 抛错：所有 (doc, tag) 对已写入提交，但后台副作用任务未调度。

对比 [modify_tags](file:///d:/fz/0601/solo-dogfeeding/code/61-paperless-ngx/src/documents/bulk_edit.py#L226-L284) 的 `bulk_create(to_create, ignore_conflicts=True)`：modify_tags 在事务中且开启 ignore_conflicts，重复行会被静默跳过不抛错。

[remove_tag](file:///d:/fz/0601/solo-dogfeeding/code/61-paperless-ngx/src/documents/bulk_edit.py#L205-L223) 只有一次 `qs.delete()`（单条 DELETE SQL，原子）+ 一次 `apply_async`。

### 3.4 modify_custom_fields：最复杂的失败边界——逐文档逐字段 update_or_create + DOCUMENTLINK 对称反射

[modify_custom_fields](file:///d:/fz/0601/solo-dogfeeding/code/61-paperless-ngx/src/documents/bulk_edit.py#L287-L356) 是所有批量操作中**失败边界最复杂**的。在分析之前，先明确一个之前完全遗漏的事实：

#### 前置知识：哪些 ORM 操作触发同步信号副作用

```python
# signals/handlers.py 注册
@receiver(post_save, sender=CustomFieldInstance)  # ← CustomFieldInstance 保存触发
def update_filename_and_move_files(sender, instance, **kwargs):
    ...  # 可能同步执行文件重命名/移动（如果文件名模板用到了 custom_fields）
```

不同 ORM 操作的信号触发与原子性对照表：

| ORM 操作 | 是否触发 `post_save(CustomFieldInstance)` | 是否触发同步文件重命名 | SQL 原子性 |
|---|---|---|---|
| `Model.save()`（单实例） | **是** | 如果模板用到 custom_fields 则**是** | 单条 UPDATE/INSERT，原子（零行失败） |
| `update_or_create()` | **是**（底层调 `.save()`） | **是** | SELECT + 单条 INSERT/UPDATE（两条语句，非原子） |
| `QuerySet.bulk_create(list)` | **否** | **否** | 单条 INSERT（多 VALUES），原子（零行失败） |
| `QuerySet.bulk_update(list, fields)` | **否** | **否** | 单条 UPDATE（CASE WHEN），原子（零行失败） |
| `QuerySet.update(field=value)` | **否** | **否** | 单条 UPDATE，原子（零行失败） |
| `QuerySet.hard_delete()` | **否**（不触发 per-instance 信号） | **否** | 单条 DELETE，原子（零行失败） |

`update_filename_and_move_files` 内部的异常处理：
- 捕获 `(OSError, DatabaseError, CannotMoveFilesException)` 并尝试回滚文件位置，**不冒泡**。
- 其他异常（如 `TypeError`、`AttributeError`、`ValueError` 等）**会冒泡**，终止整个批量操作。
- 执行在全局 `FileLock(settings.MEDIA_LOCK)` 内。

---

#### 3.4.1 modify_custom_fields 完整执行路径与原子性逐行分析

```python
def modify_custom_fields(doc_ids, add_custom_fields, remove_custom_fields):
    qs = Document.objects.filter(id__in=doc_ids).only("pk")
    affected_docs = list(qs.values_list("pk", flat=True))   # 纯 SELECT
    # ... 处理 add_custom_fields 格式转换 ...
    custom_fields = CustomField.objects.filter(...)            # 纯 SELECT

    # ═══════════════ 第一阶段：add_custom_fields ═══════════════
    for field_id, value in add_custom_fields:                  # 外层：每个字段
        for doc_id in affected_docs:                            # 中层：每个文档
            # ── 步骤 ①：update_or_create 写源文档的 CustomFieldInstance ──
            CustomFieldInstance.objects.update_or_create(     # 两条 SQL：SELECT + UPSERT
                document_id=doc_id, field_id=field_id, defaults=defaults
            )
            #  ↑ 触发 post_save(CustomFieldInstance) → 同步执行 update_filename_and_move_files
            #    可能同步移动 doc_id 对应的磁盘文件（如果文件名模板用了 custom_fields）
            #    如果同步文件重命名抛非预期异常 → 冒泡终止整个批量操作

            # ── 步骤 ②：DOCUMENTLINK 类型的对称反射 ──
            if custom_field.data_type == DOCUMENTLINK and value and doc_id not in value:
                doc = Document.objects.get(id=doc_id)          # 纯 SELECT
                reflect_doclinks(doc, custom_field, value)     # 见 3.4.2

    # ═══════════════ 第二阶段：remove_custom_fields 的 DOCUMENTLINK 对称删除 ═══════════════
    for doclink_being_removed_instance in CustomFieldInstance.objects.filter(  # 纯 SELECT
        document_id__in=affected_docs,
        field__id__in=remove_custom_fields,
        field__data_type=DOCUMENTLINK,
        value_document_ids__isnull=False,
    ):
        for target_doc_id in doclink_being_removed_instance.value:
            # ── 步骤 ③：对每个目标文档调用 remove_doclink ──
            remove_doclink(                                    # 见 3.4.3
                document=Document.objects.get(id=doclink_being_removed_instance.document.id),
                field=doclink_being_removed_instance.field,
                target_doc_id=target_doc_id,
            )
            #  ↑ 每次 remove_doclink 内部的 .save() 触发 post_save → 同步文件重命名（目标文档）

    # ═══════════════ 第三阶段：批量硬删除被移除的 CustomFieldInstance ═══════════════
    # ── 步骤 ④：hard_delete ──
    CustomFieldInstance.objects.filter(
        document_id__in=affected_docs,
        field_id__in=remove_custom_fields,
    ).hard_delete()                                          # 单条 DELETE SQL，原子
    #  ↑ 不触发 post_save/post_delete（QuerySet 批量操作不触发 per-instance 信号）

    # ═══════════════ 第四阶段：调度异步副作用 ═══════════════
    # ── 步骤 ⑤：apply_async ──
    bulk_update_documents.apply_async(...)
    return "OK"
```

---

#### 3.4.2 reflect_doclinks 内部（[bulk_edit.py#L967-L1027](file:///d:/fz/0601/solo-dogfeeding/code/61-paperless-ngx/src/documents/bulk_edit.py#L967-L1027)）

对单个源文档 + 单个 DOCUMENTLINK 字段，向目标文档添加/移除对称链接：

```python
# Phase A：移除「旧值有、新值没有」的对称链接
current_field_instance = CustomFieldInstance.objects.filter(...).first()  # SELECT
if current_field_instance is not None and current_field_instance.value is not None:
    for doc_id in current_field_instance.value:                          # 逐个旧目标
        if doc_id not in target_doc_ids:
            remove_doclink(document, field, target_doc_id=doc_id)        # 见 3.4.3
            #  ↑ 每次循环：2 条独立 UPDATE（CF.save + Document.modified）
            #    .save() 触发同步 post_save → 可能同步移动该目标文档的磁盘文件
            #    某次循环抛异常 → 之前的循环已提交不回滚

# Phase B：构建「新目标文档」的 CF 实例列表（纯内存操作）
existing_custom_field_instances = { ... }                                 # SELECT
custom_field_instances_to_create = []
custom_field_instances_to_update = []
for target_doc_id in target_doc_ids:
    ...  # 决定是 create 还是 update，填入 Python 列表

# Phase C：批量写入（3 条独立 SQL，均不触发信号）
CustomFieldInstance.objects.bulk_create(custom_field_instances_to_create)       # ① 单条 INSERT，原子
CustomFieldInstance.objects.bulk_update(                                       # ② 单条 UPDATE（CASE WHEN），原子
    custom_field_instances_to_update, ["value_document_ids"]
)
Document.objects.filter(id__in=target_doc_ids).update(modified=timezone.now()) # ③ 单条 UPDATE，原子
# ↑ 以上 3 条 SQL 各自原子（零行失败），但互相不在事务中
# ↑ 如果 ① 成功，② 失败：create 的已提交，update 的零行，modified 不更新
# ↑ 如果 ①② 成功，③ 失败：CF 已更新，目标文档 modified 未更新
# ↑ 以上 3 条均不触发 post_save，所以不会同步重命名目标文档文件
```

---

#### 3.4.3 remove_doclink 内部（[bulk_edit.py#L1030-L1048](file:///d:/fz/0601/solo-dogfeeding/code/61-paperless-ngx/src/documents/bulk_edit.py#L1030-L1048)）

单条对称链接的删除，**两条独立 SQL 语句**：

```python
def remove_doclink(document, field, target_doc_id):
    target_doc_field_instance = CustomFieldInstance.objects.filter(
        document_id=target_doc_id, field=field,
    ).first()                                                         # SELECT
    if target_doc_field_instance is not None and document.id in target_doc_field_instance.value:
        target_doc_field_instance.value.remove(document.id)           # Python 列表修改
        target_doc_field_instance.save()                               # ① UPDATE，原子
        #  ↑ 触发 post_save(CustomFieldInstance) → 同步 update_filename_and_move_files
        #    可能同步移动 target_doc_id 对应的磁盘文件
    Document.objects.filter(id=target_doc_id).update(modified=timezone.now())  # ② UPDATE，原子
```

**精确失败边界（每次 remove_doclink 调用）**：

| 抛错位置 | 已生效内容 |
|---|---|
| `.save()` 本身（数据库层） | 零行提交（单条 UPDATE 原子失败） |
| `.save()` 后的 post_save 信号（update_filename_and_move_files 抛非预期异常） | **`.save()` 已提交**，CF value 已更新；但 modified 未更新；同步文件重命名部分执行（取决于异常位置） |
| ② `Document.update(modified=...)` | `.save()` 已提交；同步文件重命名已执行；modified 未更新 |

---

#### 3.4.4 modify_custom_fields 总览：逐步骤失败边界表

假设输入：2 个文档（doc1, doc2）× 1 个 DOCUMENTLINK 字段，目标文档为 doc3, doc4；同时 remove 另一个 DOCUMENTLINK 字段含 doc5 目标。

| 抛错位置 | 已生效（已提交，不回滚） | 未生效 | 同步副作用已发生 |
|---|---|---|---|
| 第 1 次 `update_or_create(doc1, field)` 的 SELECT 阶段 | 零 | 全部 | 零 |
| 第 1 次 `update_or_create(doc1, field)` 的 UPSERT 成功，但 post_save 抛异常 | **doc1 的 CustomFieldInstance 已写入** | doc2 的 CF；所有对称反射；remove 阶段；hard_delete；bulk_update_documents | **doc1 的同步文件重命名部分执行（异常位置决定）** |
| doc1 的 `reflect_doclinks` Phase A 第 1 次 `remove_doclink` 的 `.save()` 抛 DB 异常 | doc1 的 CF 已写入；之前的 remove_doclink 调用（如果有多目标移除）已提交 | doc1 其余目标；doc2 全部；bulk_create/bulk_update；remove 阶段；hard_delete | 之前 remove_doclink 触发的目标文档同步重命名 |
| doc1 的 `reflect_doclinks` Phase C `bulk_create` 抛 IntegrityError（并发冲突） | doc1 的 CF；Phase A 中所有 remove_doclink 提交；同步重命名 | doc3, doc4 的 CF 实例（零行插入）；doc3, doc4 的 modified 更新；doc2 全部；remove 阶段；hard_delete | Phase A 中目标文档的同步重命名 |
| doc1 的 `reflect_doclinks` Phase C `bulk_update` 失败 | doc1 的 CF；Phase A 所有提交；bulk_create 已写入（doc3, doc4 新增的 CF） | doc3, doc4 已有 CF 的 value 更新（零行）；doc3, doc4 的 modified；doc2 全部；remove 阶段；hard_delete | 同上 |
| doc1 的 `reflect_doclinks` Phase C `Document.update(modified)` 失败 | doc1 的 CF；Phase A 所有提交；bulk_create；bulk_update | doc3, doc4 的 modified；doc2 全部；remove 阶段；hard_delete | 同上 |
| doc2 的 `update_or_create` 抛异常 | doc1 的 CF；doc1 的全部 reflect_doclinks（含同步重命名） | doc2 的 CF；doc2 的 reflect_doclinks；remove 阶段；hard_delete | doc1 及其目标文档的同步重命名 |
| remove 阶段第 2 次 `remove_doclink(doc5_target)` 抛异常 | 整个 add 阶段（doc1, doc2 的 CF，所有对称反射，所有同步重命名）；remove 阶段第 1 次 remove_doclink 已提交 | remove 阶段剩余目标；hard_delete；bulk_update_documents | 整个 add 阶段的同步重命名；remove 阶段已成功 remove_doclink 触发的同步重命名 |
| `hard_delete()` 抛 DB 异常 | 整个 add 阶段；整个 remove 阶段（所有对称删除，所有同步重命名） | 被 remove 的 CustomFieldInstance 仍存在（未硬删）；bulk_update_documents | 同上 |
| `bulk_update_documents.apply_async()` 抛错（broker 不可用） | **所有 DB 变更已全部提交**；所有同步副作用已发生 | bulk_update_documents 未调度 → 搜索索引不更新；工作流不执行；WebSocket 不推送；异步文件重命名不执行（但同步阶段已执行过的不会再执行） | 所有同步文件重命名已发生 |

---

**关键结论**：
- `modify_custom_fields` 的每个 `update_or_create`、每个 `remove_doclink`、`reflect_doclinks` Phase A 的每次循环都是**独立提交点**，中途任何异常都会留下部分提交的状态。
- `bulk_create`、`bulk_update`、`hard_delete()`、`QuerySet.update()` 各自是**原子单条 SQL**（零行失败），但它们之间不在事务中。
- **同步副作用（文件重命名/移动）在 `.save()` 和 `update_or_create` 之后立即发生**，不等待批量操作完成，也不参与任何事务回滚。
- 越往后期失败，已生效的变更越多；apply_async 失败时所有 DB 和文件系统变更已不可逆转，仅后台异步任务缺失。

### 3.5 set_permissions：无事务

[set_permissions](file:///d:/fz/0601/solo-dogfeeding/code/61-paperless-ngx/src/documents/bulk_edit.py#L405-L430)：

```python
qs = Document.objects.filter(id__in=doc_ids).select_related("owner")
if merge:
    qs.filter(owner__isnull=True).update(owner=owner)  # ① UPDATE，自动提交
else:
    qs.update(owner=owner)                              # ① UPDATE，自动提交

for doc in qs:                                           # ② 每个文档独立写 django-guardian 权限表
    set_permissions_for_object(permissions=set_permissions, object=doc, merge=merge)

bulk_update_documents.apply_async(...)                    # ③
```

**异常行为**：如果第 3 个文档的 `set_permissions_for_object` 抛错，前 2 个文档的 owner 和权限都已写入提交，第 3 个的 owner 已更新但 guardian 权限可能部分写入。

### 3.6 生成新文档/版本类操作：rotate/merge/split/delete_pages/edit_pdf/remove_password

这些操作的同步部分是 pikepdf 处理临时文件，然后调度 `consume_file`，本身不写业务 DB。`delete_originals=True` 时有 ASN 释放/恢复机制。

以 merge 为例 [bulk_edit.py#L594-L608](file:///d:/fz/0601/solo-dogfeeding/code/61-paperless-ngx/src/documents/bulk_edit.py#L594-L608)：

```python
if delete_originals:
    backup = release_archive_serial_numbers(affected_docs)  # ① UPDATE，同步提交
    try:
        consume_task.apply_async(
            link=[delete.si(affected_docs)],
            link_error=[restore_archive_serial_numbers_task.s(backup)],
        )                                                  # ②
    except Exception:
        restore_archive_serial_numbers(backup)             # ② 抛错时同步恢复 ASN
        raise
```

**精确异常行为**：
- ① 成功，② `apply_async` 同步抛错 → ASN 被同步恢复。
- ② 成功（任务进入队列），之后 consume_file 在 worker 中失败 → Celery 自动触发 `link_error`，异步执行 restore_archive_serial_numbers_task 恢复 ASN。
- consume_file 成功 → Celery 触发 `link`，异步执行 `delete` 删原文档。delete 任务若失败，ASN 已经释放**不会自动恢复**，只能查 PaperlessTask 发现 BULK_DELETE 失败。

chord 编排（split / edit_pdf + delete_original）同理，见 [bulk_edit.py#L673-L682](file:///d:/fz/0601/solo-dogfeeding/code/61-paperless-ngx/src/documents/bulk_edit.py#L673-L682)。

### 3.7 delete（bulk_edit.delete）：自身是 async task，且在 TRACKED_TASKS 中登记

[delete](file:///d:/fz/0601/solo-dogfeeding/code/61-paperless-ngx/src/documents/bulk_edit.py#L359-L392) 被 `@shared_task` 装饰，任务名 `"documents.bulk_edit.delete"` 在 [TRACKED_TASKS](file:///d:/fz/0601/solo-dogfeeding/code/61-paperless-ngx/src/documents/signals/handlers.py#L1005-L1017) 中登记为 TaskType.BULK_DELETE。内部 try/except 吞掉所有异常：

```python
try:
    Document.objects.filter(id__in=delete_ids).delete()   # ① Django ORM delete，级联清理
    with get_backend().batch_update() as batch:           # ② Tantivy 搜索索引批量移除
        for id in delete_ids:
            batch.remove(id)
    status_mgr.send_documents_deleted(delete_ids)          # ③ WebSocket 推送删除事件
except Exception as e:
    if "Data too long for column" in str(e):
        logger.warning(...)
    logger.error(f"Error deleting documents: {e!s}")       # ④ 吞掉异常
return "OK"                                                  # ⑤ 正常返回
```

**delete 任务的精确状态流转**（从 Celery 信号到 PaperlessTask）：

| 阶段 | Celery 信号 | PaperlessTask 状态 | 触发条件 |
|---|---|---|---|
| 调度 | `before_task_publish` | **PENDING** | 调用 `delete.apply_async()` 或 `delete.si()` 时 |
| 开始执行 | `task_prerun` | **STARTED** | Celery worker 开始执行 |
| 执行中抛异常被 try/except 捕获 | — | STARTED（不变） | ①②③ 任何一步异常，走到 ④ |
| 函数 return "OK" | `task_postrun` (state=SUCCESS) | **SUCCESS** | 函数正常返回（无论是否吞过异常） |
| （理论路径）函数外抛异常 | `task_failure` | FAILURE | try/except 之外抛异常（当前代码不会发生） |

**关键矛盾点澄清**：
- 即便 ① 或 ② 或 ③ 内部抛异常并被 ④ 捕获，函数仍然走到 ⑤ `return "OK"`。Celery 看到的是 task 正常返回，state=SUCCESS。
- `task_failure_handler` **不会**被触发（因为 Celery 只在 task 向外层抛异常时才发 task_failure 信号）。
- `task_postrun_handler` 被触发，`_CELERY_STATE_TO_STATUS["SUCCESS"]` → `PaperlessTask.Status.SUCCESS`。
- retval 是字符串 `"OK"`，不是 dict，所以不会触发 `task_postrun_handler` 中的 `isinstance(retval, dict)` 分支，也不会被改成 FAILURE。
- 最终：**即使删除操作实际失败了，PaperlessTask.Status = SUCCESS**，失败信息仅在日志中。

**部分失败的真实状态**：如果 ① `Document.objects.delete()` 只删除了部分文档（Django ORM delete 本身是事务性的，要么全部删除要么零个——因为级联删除在同一事务中；但如果 `batch.remove()` 中部分文档的索引删除失败，或 WebSocket 推送失败，这些都在 try 块内），被吞异常后任务仍返回 "OK"，PaperlessTask 仍为 SUCCESS。

### 3.8 reprocess：逐文档独立调度

[reprocess](file:///d:/fz/0601/solo-dogfeeding/code/61-paperless-ngx/src/documents/bulk_edit.py#L395-L402) 对每个文档单独调用 `update_document_content_maybe_archive_file.apply_async()`。某个 `apply_async` 抛错不影响其他文档已调度的任务。

---

## 四、异步任务层：精确配置与状态

### 4.1 各任务的重试/异常配置

| 任务 | 装饰器 | autoretry | max_retries | 内部 try/except |
|---|---|---|---|---|
| [bulk_update_documents](file:///d:/fz/0601/solo-dogfeeding/code/61-paperless-ngx/src/documents/tasks.py#L252-L275) | `@shared_task` | 无 | 无 | **无** |
| [consume_file](file:///d:/fz/0601/solo-dogfeeding/code/61-paperless-ngx/src/documents/tasks.py#L123-L220) | `@shared_task(bind=True)` | 无 | 无 | 各 plugin 级 try/except，Exception 重新 raise；外层 try/finally |
| [bulk_edit.delete](file:///d:/fz/0601/solo-dogfeeding/code/61-paperless-ngx/src/documents/bulk_edit.py#L359-L392) | `@shared_task` | 无 | 无 | **有**，吞异常返回 OK |
| [update_document_content_maybe_archive_file](file:///d:/fz/0601/solo-dogfeeding/code/61-paperless-ngx/src/documents/tasks.py#L278-L396) | `@shared_task` | 无 | 无 | 有 try/except（逐文档） |
| [workflows.webhooks.send_webhook](file:///d:/fz/0601/solo-dogfeeding/code/61-paperless-ngx/src/documents/workflows/webhooks.py#L71-L76) | `@shared_task(retry_backoff=True, autoretry_for=(HTTPStatusError,), max_retries=3, throws=(HTTPError,))` | HTTPStatusError | 3 | 无 |

**核心事实：bulk_update_documents / consume_file / delete / reprocess 都没有应用级自动重试。** 只有 webhook 任务有 `autoretry_for + max_retries=3`。

Celery 框架级（broker 连接失败等）的重试与应用无关。

### 4.2 bulk_update_documents 内部执行顺序与异常冒泡

[bulk_update_documents](file:///d:/fz/0601/solo-dogfeeding/code/61-paperless-ngx/src/documents/tasks.py#L252-L275)：

```python
@shared_task
def bulk_update_documents(document_ids) -> None:
    documents = Document.objects.filter(id__in=document_ids)

    for doc in documents:                 # ← 外层无 try/except
        clear_document_caches(doc.pk)     # ①
        document_updated.send(            # ② Django Signal.send()，不是 send_robust()
            sender=None,
            document=doc,
            logging_group=uuid.uuid4(),
        )
        post_save.send(Document, instance=doc, created=False)  # ③

    with get_backend().batch_update() as batch:   # ④
        for doc in documents:
            batch.add_or_update(doc)

    if ai_config.llm_index_enabled:                # ⑤
        update_llm_index(rebuild=False)
```

**关键点**：

1. **Django Signal.send()**（不是 `.send_robust()`）的行为：依次调用所有 receiver，任何一个 receiver 抛异常，**立即终止分发**，异常**直接冒泡**给 send() 的调用者。

2. **document_updated 信号 receiver 的注册顺序**（[apps.py#L32-L33](file:///d:/fz/0601/solo-dogfeeding/code/61-paperless-ngx/src/documents/apps.py#L32-L33)）：
   - 先 `run_workflows_updated` → `run_workflows`（触发工作流，内部可能再做 document.save()）
   - 后 `send_websocket_document_updated`（WebSocket 推送）

   如果 `run_workflows_updated` 抛错，`send_websocket_document_updated` **永远不会执行**。

3. **post_save 信号**触发 `update_filename_and_move_files`，该函数内部有 try/except 吞掉 `OSError / DatabaseError / CannotMoveFilesException` 并尝试回滚文件位置（见 [handlers.py#L613-L644](file:///d:/fz/0601/solo-dogfeeding/code/61-paperless-ngx/src/documents/signals/handlers.py#L613-L644)），但该异常**不会冒泡**，因为被 handler 内部捕获。

#### bulk_update_documents 失败时的精确已生效副作用（假设 3 个文档，在 doc2 的 run_workflows 中抛错）：

```
doc1:
  ① clear_document_caches(doc1.pk)          ✓ 已生效（缓存被清，不可逆）
  ② document_updated.send(doc1):
     - run_workflows_updated(doc1)          ✓ 可能已执行完，工作流 mutation 已写入 DB
     - send_websocket_document_updated(doc1) ✓ 可能已推送
  ③ post_save.send(doc1):
     - update_filename_and_move_files(doc1) ✓ 文件可能已移动
  （循环继续）
doc2:
  ① clear_document_caches(doc2.pk)          ✓ 已生效
  ② document_updated.send(doc2):
     - run_workflows_updated(doc2)          ✗ 抛错
     - send_websocket_document_updated(doc2) 未执行
  ③ post_save.send(doc2)                    未执行
doc3:
  全部未执行
④ 搜索索引 batch_update                     未执行（循环未结束就抛了）
⑤ LLM 索引更新                              未执行
```

最终结果：
- PaperlessTask 状态为 **FAILURE**（由 [task_failure_handler](file:///d:/fz/0601/solo-dogfeeding/code/61-paperless-ngx/src/documents/signals/handlers.py#L1226-L1278) 写入）
- doc1 的所有副作用已经发生且不可逆
- doc2 缓存被清，部分工作流可能已写入
- doc3 完全未处理
- 搜索索引未更新（与 DB 状态不一致）
- **不会自动重试**（任务未配置 autoretry）

### 4.3 PaperlessTask 状态流转

[PaperlessTask](file:///d:/fz/0601/solo-dogfeeding/code/61-paperless-ngx/src/documents/models.py#L664-L693) 的状态机：

```
PENDING ──▶ STARTED ──▶ SUCCESS
   │          │
   │          └──────▶ FAILURE
   └─────────────────▶ REVOKED  (任务在启动前被取消)
```

由 Celery 信号驱动（[handlers.py#L1005-L1313](file:///d:/fz/0601/solo-dogfeeding/code/61-paperless-ngx/src/documents/signals/handlers.py#L1005-L1313)）：

| Celery 信号 | PaperlessTask 写入 |
|---|---|
| `before_task_publish` | 创建记录，Status = PENDING |
| `task_prerun` | Status = STARTED，date_started |
| `task_postrun` (state=SUCCESS) | Status = SUCCESS，date_done，duration，wait_time |
| `task_postrun` (state=FAILURE) | **跳过**（由 task_failure 全权处理） |
| `task_failure` | Status = FAILURE，result_data（含 error_type / error_message / traceback）|
| `task_revoked` | Status = REVOKED，date_done |

被登记追踪的任务（[TRACKED_TASKS](file:///d:/fz/0601/solo-dogfeeding/code/61-paperless-ngx/src/documents/signals/handlers.py#L1005-L1017)：`consume_file`、`train_classifier`、`sanity_check`、`llmindex_index`、`empty_trash`、`check_scheduled_workflows`、**`bulk_update_documents`**（TaskType.BULK_UPDATE）、**`reprocess_document`**、`build_share_link_bundle`、**`bulk_edit.delete`**（TaskType.BULK_DELETE，任务名 `"documents.bulk_edit.delete"`）。

**关键事实**：
- `delete.si(affected_docs)（merge/split/edit_pdf 中 link/chord 触发的删除子任务）使用的是**同一个** `bulk_edit.delete` 函数，Celery 任务名完全相同，**会被追踪**，会产生 PaperlessTask 记录（TaskType.BULK_DELETE）。
- 真正**不在** TRACKED_TASKS 中的是 `restore_archive_serial_numbers_task（任务名 `"documents.bulk_edit.restore_archive_serial_numbers_task"`），它作为 link_error 回调失败补偿任务，失败时没有 PaperlessTask 记录，无法通过 PaperlessTask 追踪。

---

## 五、信号副作用层——执行顺序与异常

### 5.1 信号连接注册（[apps.py#L24-L33](file:///d:/fz/0601/solo-dogfeeding/code/61-paperless-ngx/src/documents/apps.py#L24-L33)）

```python
# document_consumption_finished（仅消费流程，与 bulk edit 无关）
document_consumption_finished.connect(add_inbox_tags)
document_consumption_finished.connect(set_correspondent)
document_consumption_finished.connect(set_document_type)
document_consumption_finished.connect(set_tags)
document_consumption_finished.connect(set_storage_path)
document_consumption_finished.connect(add_to_index)
document_consumption_finished.connect(run_workflows_added)
document_consumption_finished.connect(add_or_update_document_in_llm_index)

# document_updated（bulk_update_documents 触发）
document_updated.connect(run_workflows_updated)          # ① 先执行
document_updated.connect(send_websocket_document_updated)  # ② 后执行
```

Django ORM 信号（与 `.send()` 同样行为）：
- `post_save(sender=Document)` → `update_filename_and_move_files`
- `post_save(sender=CustomFieldInstance)` → `update_filename_and_move_files`
- `m2m_changed(sender=Document.tags.through)` → `update_filename_and_move_files`

### 5.2 run_workflows_updated → run_workflows（DOCUMENT_UPDATED）

[run_workflows](file:///d:/fz/0601/solo-dogfeeding/code/61-paperless-ngx/src/documents/signals/handlers.py#L854-L998) 在每个 workflow 执行前检查：

```python
document.refresh_from_db()          # ① 防并发覆盖
except Document.DoesNotExist:       # 硬删则跳过剩余 workflow
    break
if document.is_deleted:             # 软删则跳过剩余 workflow
    break
# 匹配工作流，执行 actions
document.save(
    update_fields=["title", "correspondent", "document_type", "storage_path", "owner", "modified"]
)
```

**注意**：这里的 `document.save(update_fields=[...])` 会再次触发 `post_save` → `update_filename_and_move_files`（连锁副作用）。但该 save 只写白名单字段，不会回滚并发写入的 `filename` / `archive_filename`（见注释 [handlers.py#L968-L984](file:///d:/fz/0601/solo-dogfeeding/code/61-paperless-ngx/src/documents/signals/handlers.py#L968-L984)）。

### 5.3 send_websocket_document_updated

[handlers.py#L832-L851](file:///d:/fz/0601/solo-dogfeeding/code/61-paperless-ngx/src/documents/signals/handlers.py#L832-L851)：刷新文档后 WebSocket 推送 modified / owner_id / 权限。

### 5.4 update_filename_and_move_files

[handlers.py#L434-L667](file:///d:/fz/0601/solo-dogfeeding/code/61-paperless-ngx/src/documents/signals/handlers.py#L434-L667)：
- 使用 `FileLock(settings.MEDIA_LOCK)` 全局互斥锁
- 失败时尝试回滚文件到原位置（[handlers.py#L621-L639](file:///d:/fz/0601/solo-dogfeeding/code/61-paperless-ngx/src/documents/signals/handlers.py#L621-L639)）
- 异常被内部捕获，**不会冒泡**给调用者

---

## 六、完整调用链路时序图（以 modify_custom_fields 为例，精确标注事务边界）

```
前端 POST /api/documents/bulk_edit/
    │
    ▼
BulkEditView.post()
    │  （无外层事务）
    │  1. 序列化 method="modify_custom_fields"
    │  2. 权限校验
    │  3. 审计快照
    │  4. 调用 bulk_edit.modify_custom_fields(
    │        [doc1, doc2],
    │        add_custom_fields={cf_doclink_id: [doc3.id]},
    │        remove_custom_fields=[]
    │     )
    │
    ▼
bulk_edit.modify_custom_fields()   （无 transaction.atomic）
    │
    ├─ FOR field_id IN add_custom_fields:
    │    ├─ FOR doc_id IN [doc1, doc2]:
    │    │    ├─ CustomFieldInstance.objects.update_or_create(doc1, cf_doclink)  ← DB 提交
    │    │    ├─ reflect_doclinks(doc1, cf_doclink, [doc3.id])
    │    │    │    ├─ remove_doclink() ...                                    ← DB 提交（无则跳过）
    │    │    │    ├─ CustomFieldInstance.objects.bulk_create(doc3 侧链接)     ← DB 提交
    │    │    │    └─ Document.objects.filter(id=doc3).update(modified=now)    ← DB 提交
    │    │    ├─ CustomFieldInstance.objects.update_or_create(doc2, cf_doclink)  ← DB 提交
    │    │    └─ reflect_doclinks(doc2, cf_doclink, [doc3.id])                  ← DB 提交
    │
    ├─ FOR remove_custom_fields 的对称反射:
    │    └─ remove_doclink(...) ...                                             ← DB 提交
    │
    ├─ CustomFieldInstance.objects.filter(...).hard_delete()                    ← DB 提交（无删除则跳过）
    │
    ├─ bulk_update_documents.apply_async([doc1.id, doc2.id])                  ← 入队，返回任务 ID
    │
    └─ return "OK"
    │
    ▼
BulkEditView: 写审计日志 → return HTTP 200 {"result": "OK"}
    │
    │  ═══════════ 前端已拿到响应，以下后台异步 ═══════════
    │
    ▼
Celery Worker: bulk_update_documents([doc1, doc2])
    │
    ├─ doc1:
    │    ├─ clear_document_caches(doc1.pk)                         ✓ 缓存失效
    │    ├─ document_updated.send(doc1):
    │    │    ├─ run_workflows_updated(doc1) → run_workflows():
    │    │    │    ├─ refresh_from_db
    │    │    │    ├─ 匹配并执行 DOCUMENT_UPDATED 工作流 actions
    │    │    │    └─ document.save(update_fields=[...])            ← 触发 post_save
    │    │    │           └─► update_filename_and_move_files(doc1)
    │    │    │                 └─ 文件可能被移动 / 文件名更新
    │    │    └─ send_websocket_document_updated(doc1)              ← WebSocket 推送
    │    └─ post_save.send(Document, instance=doc1)
    │         └─► update_filename_and_move_files(doc1) （若工作流没触发则这里触发）
    │
    ├─ doc2:
    │    └─ ... 同上 ...
    │
    └─ 搜索索引批量 batch.add_or_update(doc1, doc2)                ← 索引刷新
```

---

## 七、部分失败边界——精确对照表

### 7.1 同步阶段各函数异常时的真实状态

| 函数 | 事务包裹 | 中途抛错时已生效内容 | 异步任务是否调度 |
|---|---|---|---|
| set_correspondent | 无（单条 UPDATE 原子） | UPDATE 成功则 correspondent 已写入；否则无 | UPDATE 成功但 apply_async 失败则任务未调度 |
| set_document_type | 无（单条 UPDATE 原子） | 同上 | 同上 |
| set_storage_path | 无（单条 UPDATE 原子） | 同上 | 同上 |
| **add_tag** | **无** | **零行写入（bulk_create 原子失败，无 ignore_conflicts，零行插入）** | **bulk_create 前抛错（Tag.objects.get 失败）则不调度；bulk_create 成功但 apply_async 失败则不调度** |
| remove_tag | 无（单条 DELETE 原子） | DELETE 成功则 tag 关系已移除 | DELETE 成功但 apply_async 失败则任务未调度 |
| **modify_tags** | **transaction.atomic()** | **全部回滚** | 不调度 |
| **modify_custom_fields** | **无** | **每个 `update_or_create` / 每个 `remove_doclink` / reflect_doclinks Phase A 每次循环都是独立提交点；`bulk_create`/`bulk_update`/`hard_delete`各自原子（零行失败）但互相不在事务中；同步文件重命名在每次 `.save()`/`update_or_create` 后立即发生且不可逆** | apply_async 前抛则不调度 |
| set_permissions | 无 | 前面文档的 owner 已 UPDATE；guardian 权限表逐文档写入到抛错点 | apply_async 前抛则不调度 |
| merge/split/edit_pdf (delete_originals=False) | 无（只写临时文件） | 临时文件可能残留（OS 级） | apply_async 前抛则不调度 consume_file |
| merge/split/edit_pdf (delete_originals=True) | ASN 释放+恢复保护 | ASN 可能已释放；apply_async 抛错时同步恢复 | apply_async 成功后 consume 失败会 link_error 恢复 ASN |

### 7.2 bulk_update_documents 异步任务中途失败

| 抛错位置 | 已生效副作用（不可逆） | PaperlessTask 状态 | 自动重试 |
|---|---|---|---|
| doc1 的 clear_document_caches | doc1 缓存被清 | FAILURE | 否 |
| doc1 的 run_workflows_updated | doc1 缓存被清；工作流 mutations 可能已写入 DB；WebSocket 未推送 | FAILURE | 否 |
| doc1 的 send_websocket_document_updated | doc1 缓存被清；工作流全部执行完；WebSocket 推送（部分？） | FAILURE | 否 |
| doc2 循环开始时 doc1 已完整处理；doc2 缓存被清 | doc1 的所有副作用、doc2 缓存清、doc2 部分工作流 | FAILURE | 否 |
| 搜索索引 batch_update 内 | 所有文档的缓存清、信号链已走完 | FAILURE | 否 |
| LLM 索引 update_llm_index | 所有文档的缓存清、信号链、搜索索引已走完 | FAILURE | 否 |

### 7.3 chord/link 编排失败（delete_originals=True 时）

```
                    consume_file 成功
                   ┌───────────────────────────────▶ delete.si(原文档) = bulk_edit.delete
                   │                                (独立 Celery 任务，任务名 "documents.bulk_edit.delete"，
                   │                                 **在 TRACKED_TASKS 中** → 有独立 PaperlessTask 记录)
                   │                                 若该任务内部 try/except 吞掉异常 → Status = SUCCESS
                   │                                 若该任务抛异常（理论上不会，因为被内部 try/except 捕获）→ Status = FAILURE
                   │                                 任务失败时 ASN 不会自动恢复（因为恢复 ASN 的 link_error 只挂在 consume_file 上）
                   │
chord(header=consume_tasks)
                   │
                   │ consume_file 失败
                   └───────────────────────────────▶ restore_archive_serial_numbers_task.s(backup)
                                                     (独立 Celery 任务，任务名 "documents.bulk_edit.restore_archive_serial_numbers_task"，
                                                      **不在 TRACKED_TASKS 中** → 没有 PaperlessTask 记录)
```

**任务追踪精确对照表**（以 merge delete_originals=True 为例）：

| 任务 | 任务名 | 是否在 TRACKED_TASKS | 是否有 PaperlessTask |
|---|---|---|---|
| consume_file（主任务） | `documents.tasks.consume_file` | 是（CONSUME_FILE） | 是 |
| delete.si(原文档)（link 回调） | `documents.bulk_edit.delete` | 是（BULK_DELETE） | 是 |
| restore_archive_serial_numbers_task（link_error 回调） | `documents.bulk_edit.restore_archive_serial_numbers_task` | **否** | **否** |

所以：如果 delete 子任务部分删除失败，PaperlessTask 状态仍为 SUCCESS（因为内部 try/except 吞异常），且 ASN 不会恢复，只能通过日志发现。如果 restore_archive_serial_numbers_task 本身执行失败，由于不在 TRACKED_TASKS，没有任何 PaperlessTask 记录，完全不可观测，只能看日志。

### 7.4 bulk_edit.delete（async task）失败可观测性

详见 3.7 节完整分析。此处列出**前后矛盾的修正点**：

**之前的矛盾**：文档一方面说 delete 任务在 TRACKED_TASKS 中（有 PaperlessTask 记录），另一方面又说 link/chord 中的 delete 子任务"无独立 PaperlessTask 记录"。

**修正后的事实**：
- `bulk_edit.delete` 的任务名是 `"documents.bulk_edit.delete"`，明确在 [TRACKED_TASKS](file:///d:/fz/0601/solo-dogfeeding/code/61-paperless-ngx/src/documents/signals/handlers.py#L1005-L1017) 中登记（TaskType.BULK_DELETE）。
- 无论是通过 `BulkEditView.post()` 直接调度的 `delete.apply_async()`，还是通过 Celery `link=[delete.si(affected_docs)]` / `chord(body=delete.si([...]))` 调度的子任务，使用的都是**同一个** `delete` 函数，任务名相同，**都会**被追踪，都会产生 PaperlessTask 记录。
- 真正不在 TRACKED_TASKS 中的是 `restore_archive_serial_numbers_task`（任务名 `"documents.bulk_edit.restore_archive_serial_numbers_task"`）。

**delete 任务的最终状态**（无论哪条调度路径）：
- 内部 try/except 吞掉所有异常 → `return "OK"` → Celery state = SUCCESS
- `task_postrun_handler` 触发 → PaperlessTask.Status = **SUCCESS**
- `task_failure_handler` 不触发
- retval 是字符串 "OK"（不是 dict），不会被特殊分支改为 FAILURE
- **所以即便删除部分失败，PaperlessTask 仍为 SUCCESS**，失败信息仅出现在日志中

---

## 八、关键设计总结（三次修正版）

| 关注点 | 真实设计（三次修正后） |
|---|---|
| **同步/异步分界** | DB 元数据修改同步完成；搜索索引、工作流、WebSocket 全部异步（bulk_update_documents）；**文件重命名/移动分同步+异步两路**：`.save()`/`update_or_create` 触发 CustomFieldInstance.post_save → 同步重命名；Document.post_save 在 bulk_update_documents 中异步触发 |
| **事务策略** | 仅 `modify_tags` 用 `transaction.atomic()` + `bulk_create(ignore_conflicts=True)`；其他元数据操作全部隐式自动提交，逐语句独立 |
| **add_tag vs modify_tags** | add_tag：一次 bulk_create，无 ignore_conflicts，并发冲突抛 IntegrityError（零行插入）；modify_tags：transaction + ignore_conflicts，并发冲突静默跳过，失败全回滚 |
| **modify_custom_fields 失败边界** | 每个 `update_or_create` / 每个 `remove_doclink` / reflect_doclinks Phase A 每次循环都是**独立提交点**；`bulk_create`/`bulk_update`/`hard_delete()`/`QuerySet.update()`各自是**原子单条 SQL**（零行失败），但互相不在事务中 |
| **ORM 操作 vs 信号触发** | `.save()`/`update_or_create` 触发 post_save → 同步文件重命名；`bulk_create`/`bulk_update`/`QuerySet.update()`/`hard_delete()` **不触发** post_save，无同步副作用 |
| **同步副作用（文件重命名）** | CustomFieldInstance post_save 同步触发 update_filename_and_move_files；捕获 `OSError/DatabaseError/CannotMoveFilesException` 不冒泡，其余异常冒泡终止整个批量；同步文件移动无事务回滚 |
| **请求级事务** | BulkEditView 外层无事务；未开启 ATOMIC_REQUESTS；同步阶段抛错不回滚已提交 SQL |
| **任务重试** | 仅 webhook 任务配置 `autoretry_for + max_retries=3`；bulk_update_documents / consume_file / delete / reprocess 均无应用级重试 |
| **信号异常冒泡** | 用 `Signal.send()` 非 `send_robust()`；第一个 receiver 抛异常立即终止分发并冒泡；receiver 注册顺序：run_workflows_updated → send_websocket_document_updated |
| **bulk_update_documents 失败** | 外层 for 无 try/except；单个文档的信号异常导致整个任务失败，已处理文档的副作用不回滚 |
| **delete 任务追踪** | `"documents.bulk_edit.delete"` 明确登记在 TRACKED_TASKS（TaskType.BULK_DELETE），包括 BulkEditView 直接调度和 Celery link/chord 子任务调度的两种场景；均会产生 PaperlessTask 记录 |
| **delete 任务状态** | 内部 try/except 吞掉所有异常，return "OK" → Celery state=SUCCESS → PaperlessTask.Status=SUCCESS；task_failure_handler 不触发；即便实际删除失败也显示 SUCCESS |
| **ASN 恢复任务追踪** | `restore_archive_serial_numbers_task`（link_error 回调）**不在** TRACKED_TASKS 中，失败不可观测，无 PaperlessTask 记录 |
| **ASN 恢复机制** | apply_async 同步抛错 → 同步恢复；consume_file 异步失败 → Celery link_error 异步恢复；delete 子任务失败 → 不恢复 ASN |
| **并发安全** | 文件操作使用全局 FileLock；工作流每次 refresh_from_db；工作流保存时指定 update_fields 白名单避免回滚 filename 字段 |

---

## 附：历次修正的前后矛盾对照表

| 轮次 | 位置 | 之前的说法（矛盾/错误） | 修正后的事实 | 代码依据 |
|---|---|---|---|---|
| 2 | 3.3 add_tag 异常行为 | "某个祖先 tag 的 bulk_create 中途失败，前面 tag 的行已经插入" | 一次 bulk_create（包含所有 tag+祖先），无 ignore_conflicts，失败零行插入 | [bulk_edit.py#L193-L194](file:///d:/fz/0601/solo-dogfeeding/code/61-paperless-ngx/src/documents/bulk_edit.py#L193-L194) |
| 2 | 4.3 TRACKED_TASKS | "delete 子任务不在 TRACKED_TASKS 中" | delete 任务名 `"documents.bulk_edit.delete"` 明确登记；子任务与主任务是同一个函数，都被追踪 | [handlers.py#L1016](file:///d:/fz/0601/solo-dogfeeding/code/61-paperless-ngx/src/documents/signals/handlers.py#L1016) |
| 2 | 7.3 chord 图 | "delete 子任务无独立 PaperlessTask 记录" | delete.si() 有独立 PaperlessTask 记录（TaskType.BULK_DELETE）；真正无记录的是 `restore_archive_serial_numbers_task` | [handlers.py#L1005-L1017](file:///d:/fz/0601/solo-dogfeeding/code/61-paperless-ngx/src/documents/signals/handlers.py#L1005-L1017) |
| 2 | 7.1 对照表 add_tag | "部分祖先 tag 的 DocumentTagRelationship 可能已插入" | "零行写入（bulk_create 原子失败，零行插入）" | [bulk_edit.py#L193-L194](file:///d:/fz/0601/solo-dogfeeding/code/61-paperless-ngx/src/documents/bulk_edit.py#L193-L194) |
| 2 | 3.1 modify_tags | 只提了 transaction.atomic，没提 ignore_conflicts | transaction.atomic + bulk_create(ignore_conflicts=True)，并发冲突静默跳过 | [bulk_edit.py#L272-L276](file:///d:/fz/0601/solo-dogfeeding/code/61-paperless-ngx/src/documents/bulk_edit.py#L272-L276) |
| 2 | 3.7 delete | 只说"被 @shared_task 装饰"，未明确是否追踪 | 任务名 `"documents.bulk_edit.delete"` 在 TRACKED_TASKS 中；含精确状态流转表 | [handlers.py#L1016](file:///d:/fz/0601/solo-dogfeeding/code/61-paperless-ngx/src/documents/signals/handlers.py#L1016) |
| **3** | **3.4 modify_custom_fields** | **笼统说"三层嵌套循环均无 transaction"，未区分各 ORM 操作的原子性差异，完全遗漏同步副作用** | **`update_or_create`/`.save()` 触发 post_save → 同步文件重命名；`bulk_create`/`bulk_update`/`QuerySet.update()`/`hard_delete()`不触发信号；循环内每次迭代是独立提交点；Phase C 三条 SQL 各自原子但互不在事务中** | [bulk_edit.py#L287-L356](file:///d:/fz/0601/solo-dogfeeding/code/61-paperless-ngx/src/documents/bulk_edit.py#L287-L356)、[handlers.py#L431-L442](file:///d:/fz/0601/solo-dogfeeding/code/61-paperless-ngx/src/documents/signals/handlers.py#L431-L442) |
| **3** | **reflect_doclinks Phase C** | **未区分 bulk_create、bulk_update、modified 更新三条 SQL 的原子性和信号差异** | **三条 SQL 各自原子（零行失败）但互不在事务中；均不触发 post_save，无同步副作用；① 成功② 失败时 create 已提交 update 零行；①② 成功③ 失败时 modified 未更新** | [bulk_edit.py#L256-L265](file:///d:/fz/0601/solo-dogfeeding/code/61-paperless-ngx/src/documents/bulk_edit.py#L256-L265) |
| **3** | **remove_doclink** | **只说"独立 save()，自动提交"，未区分两条 SQL 和同步副作用** | **两条独立 UPDATE（CF.save + Document.modified）均原子但互不在事务中；.save() 触发 post_save → 同步文件重命名；save 成功但 modified 失败时 CF 已提交 modified 未更新** | [bulk_edit.py#L1030-L1048](file:///d:/fz/0601/solo-dogfeeding/code/61-paperless-ngx/src/documents/bulk_edit.py#L1030-L1048) |
