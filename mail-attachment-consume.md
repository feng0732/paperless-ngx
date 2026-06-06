# Paperless-ngx 邮件导入链路分析

## 一、总体架构

邮件导入模块位于 `src/paperless_mail/`，主要组件：

| 文件 | 职责 |
|------|------|
| [models.py](file:///d:/fz/0601/solo-dogfeeding/code/58-paperless-ngx/src/paperless_mail/models.py) | MailAccount / MailRule / ProcessedMail 模型 |
| [mail.py](file:///d:/fz/0601/solo-dogfeeding/code/58-paperless-ngx/src/paperless_mail/mail.py) | 核心处理逻辑（MailAccountHandler） |
| [tasks.py](file:///d:/fz/0601/solo-dogfeeding/code/58-paperless-ngx/src/paperless_mail/tasks.py) | Celery 任务入口 |
| [preprocessor.py](file:///d:/fz/0601/solo-dogfeeding/code/58-paperless-ngx/src/paperless_mail/preprocessor.py) | 邮件预处理器（如 PGP 解密 |
| [oauth.py](file:///d:/fz/0601/solo-dogfeeding/code/58-paperless-ngx/src/paperless-ngx/src/paperless_mail/oauth.py) | OAuth 认证管理 |

---

## 二、任务入口与调度

### 2.1 Celery 任务：process_mail_accounts

定义位置：[tasks.py#L13-L33](file:///d:/fz/0601/solo-dogfeeding/code/58-paperless-ngx/src/paperless_mail/tasks.py#L13-L33)

```python
@shared_task
def process_mail_accounts(account_ids: list[int] | None = None) -> str:
```

处理流程：
1. 筛选账户（全部或指定 ID 列表）
2. 跳过没有启用规则的账户
3. 对每个账户调用 `MailAccountHandler().handle_mail_account(account)`

该任务由 Celery Beat 周期调度，也可手动触发。

---

## 三、邮箱账户与规则模型

### 3.1 MailAccount（邮箱账户）

定义位置：[models.py#L8-L84](file:///d:/fz/0601/solo-dogfeeding/code/58-paperless-ngx/src/paperless_mail/models.py#L8-L84)

关键字段：
- `imap_server` / `imap_port` / `imap_security`（SSL/STARTTLS/NONE）
- `username` / `password`（或 `refresh_token` + `is_token`）
- `account_type`：IMAP / Gmail OAuth / Outlook OAuth
- `character_set`：默认 UTF-8

### 3.2 MailRule（邮件规则）

定义位置：[models.py#L87-L313](file:///d:/fz/0601/solo-dogfeeding/code/58-paperless-ngx/src/paperless_mail/models.py#L87-L313)

#### 过滤规则（筛选邮件）：
| 字段 | 作用 |
|------|------|
| `folder` | IMAP 文件夹，默认 INBOX |
| `maximum_age` | 最大邮件年龄（天），默认 30 |
| `filter_from` / `filter_to` / `filter_subject` / `filter_body` | 发件人/收件人/主题/正文匹配 |
| `filter_attachment_filename_include` | 附件文件名包含匹配（通配符，逗号分隔多个） |
| `filter_attachment_filename_exclude` | 附件文件名排除匹配（通配符，逗号分隔多个） |

#### 处理范围：
- `consumption_scope`：
  - `ATTACHMENTS_ONLY`（1）仅处理附件
  - `EML_ONLY`（2）仅处理整封邮件（.eml）
  - `EVERYTHING`（3）邮件+附件都处理

- `attachment_type`：
  - `ATTACHMENTS_ONLY`（1）仅处理 content-disposition=attachment 的附件（跳过 inline 嵌入图片等）
  - `EVERYTHING`（2）处理所有附件（含 inline）

#### 处理后动作（MailAction）：
| 动作 | IMAP 查询过滤（避免重复处理） | 执行动作 |
|------|------|------|
| DELETE（1） | - | 删除邮件 |
| MOVE（2） | - | 移动到 action_parameter 指定文件夹 |
| MARK_READ（3） | 仅拉取未读邮件 | 标记已读 |
| FLAG（4） | 仅拉取未星标邮件 | 加星标 |
| TAG（5） | 排除已打标签 | 打 IMAP keyword/Gmail 标签/Apple Mail 颜色标签 |

#### 元数据赋值：
| 字段 | 说明 |
|------|------|
| `assign_title_from` | 从邮件主题 / 附件文件名 / 不指定 |
| `assign_correspondent_from` | 发件人邮箱 / 发件人姓名 / 指定往来单位 / 不指定 |
| `assign_tags` | 要添加的标签 |
| `assign_document_type` | 要指定的文档类型 |
| `assign_owner_from_rule` | 是否将规则所有者作为文档所有者 |
| `stop_processing` | 命中后是否停止后续规则 |

---

## 四、邮箱取信逻辑

### 4.1 handle_mail_account — 账户处理入口

定义位置：[mail.py#L540-L608](file:///d:/fz/0601/solo-dogfeeding/code/58-paperless-ngx/src/paperless_mail/mail.py#L540-L608)

```
1. 连接邮箱（get_mailbox）
   ├─ 安全校验：EMAIL_ALLOW_INTERNAL_HOSTS 限制公网IP检查
   ├─ 根据 imap_security 选择 MailBox / MailBoxStartTls / MailBoxUnencrypted
   └─ SSL 证书支持自定义 CA（EMAIL_CERTIFICATE_FILE）
2. OAuth Token 过期则刷新
3. 登录邮箱（mailbox_login）
   ├─ is_token → XOAUTH2 认证
   └─ 密码认证（ASCII / UTF-8（AUTH=PLAIN）
4. 按 order 顺序遍历每条规则
   └─ _handle_mail_rule()
```

### 4.2 _handle_mail_rule — 单条规则处理

定义位置：[mail.py#L615-L713](file:///d:/fz/0601/solo-dogfeeding/code/58-paperless-ngx/src/paperless_mail/mail.py#L615-L713)

```
1. 选择文件夹（含 MOVE 目标文件夹预先验证存在）
2. make_criterias() 构造 IMAP 查询条件
3. M.fetch(criteria, mark_seen=False, bulk=True) 拉取邮件
4. 逐封邮件去重与跳过：
   ├─ rule_seen_messages：本次 fetch 结果去重（同邮件重复跳过
   ├─ consumed_messages：已被前面规则处理的邮件跳过
   └─ ProcessedMail：DB 中已有记录跳过
5. _handle_message() 处理单封邮件
6. 若 stop_processing=True 且本次有文档入队，则 break
```

### 4.3 make_criterias — IMAP 查询条件构造

定义位置：[mail.py#L383-L411](file:///d:/fz/0601/solo-dogfeeding/code/58-paperless-ngx/src/paperless_mail/mail.py#L615-L713)

```python
def make_criterias(rule, *, supports_gmail_labels):
    criterias = {}
    # 邮件年龄过滤：date_gte = today - maximum_age
    # filter_from / filter_to / filter_subject / filter_body
    # 与 MailAction.get_criteria() 组合（AND）
```

MailAction 与过滤条件：
- MARK_READ：`seen=False`（仅未读）
- FLAG：`flagged=False`（仅未加星标）
- TAG：
  - Apple Mail 颜色：`flagged=False`
  - Gmail 标签：`NOT(gmail_label=X) AND no_keyword=X
  - 普通 keyword：`no_keyword=X
- DELETE / MOVE：无过滤（每次都扫描全部）

### 4.4 邮件预处理器

定义位置：[preprocessor.py](file:///d:/fz/0601/solo-dogfeeding/code/58-paperless-ngx/src/paperless_mail/preprocessor.py)

当前注册的预处理器（mail.py#L454-L456](file:///d:/fz/0601/solo-dogfeeding/code/58-paperless-ngx/src/paperless_mail/mail.py#L454-L456)：
- `MailMessageDecryptor`：PGP 加密邮件解密（EMAIL_ENABLE_GPG_DECRYPTOR 开启）

---

## 五、附件接收与跳过规则

### 5.1 单封邮件处理入口：_handle_message

定义位置：[mail.py#L715-L759](file:///d:/fz/0601/solo-dogfeeding/code/58-paperless-ngx/src/paperless_mail/mail.py#L715-L759)

```
_handle_message(message, rule)
  ├─ _preprocess_message() 预处理（解密等）
  ├─ 若 ATTACHMENTS_ONLY 且无附件 → 直接返回 0
  ├─ consumption_scope 判断：
  │   ├─ EML_ONLY / EVERYTHING → _process_eml() 处理 .eml
  │   └─ ATTACHMENTS_ONLY / EVERYTHING → _process_attachments() 处理附件
  └─ 返回成功处理的文件数
```

### 5.2 _process_attachments — 附件逐个筛选核心

定义位置：[mail.py#L798-L940](file:///d:/fz/0601/solo-dogfeeding/code/58-paperless-ngx/src/paperless_mail/mail.py#L798-L940)

每个附件经过以下关卡，按顺序跳过：

#### 关卡 1：Content-Disposition 检查

```python
if (
    att.content_disposition != "attachment"
    and rule.attachment_type == ATTACHMENTS_ONLY
):
    跳过 → "Skipping attachment ... with content disposition inline"
```

- 当规则设为"仅附件"时，跳过所有 `content_disposition != "attachment"` 的附件（如 inline 嵌入图片）
- 当规则设为"全部"时，inline 附件也会被处理

#### 关卡 2：文件名包含匹配

定义位置：[mail.py#L761-L778](file:///d:/fz/0601/solo-dogfeeding/code/58-paperless-ngx/src/paperless_mail/mail.py#L761-L778)

```python
def filename_inclusion_matches(filter_include, filename):
    # 逗号分隔多个 pattern，大小写不敏感，fnmatch 通配符匹配
    # 任一 pattern 匹配 → True；全部不匹配 → False（跳过）
```

- `filter_attachment_filename_include` 不为空时：任一 pattern 匹配才接收
- 例：`*.pdf, *invoice*` → 只收 PDF 和含 invoice 的文件
- 为空时默认通过

#### 关卡 3：文件名排除匹配

定义位置：[mail.py#L780-L796](file:///d:/fz/0601/solo-dogfeeding/code/58-paperless-ngx/src/paperless-ngx/src/paperless_mail/mail.py#L780-L796)

```python
def filename_exclusion_matches(filter_exclude, filename):
    # 逗号分隔多个 pattern，大小写不敏感，fnmatch 通配符匹配
    # 任一 pattern 匹配 → True（跳过）
```

- `filter_attachment_filename_exclude` 任一 pattern 匹配即跳过
- 例：`signature*.png, image*` → 跳过签名图片
- 为空时默认通过

#### 关卡 4：MIME 类型支持检测

定义位置：[mail.py#L851-L853](file:///d:/fz/0601/solo-dogfeeding/code/58-paperless-ngx/src/paperless_mail/mail.py#L851-L853)

```python
mime_type = magic.from_buffer(att.payload, mime=True)  # 读取附件二进制内容检测真实类型
if is_mime_type_supported(mime_type):
    # 通过，继续
else:
    跳过 → "Skipping attachment ... since guessed mime type ... is not supported"
```

**注意：不使用邮件声明的 Content-Type，而是用 python-magic 从实际文件内容嗅探真实 MIME 类型。

`is_mime_type_supported()` 定义：[parsers.py#L25-L29](file:///d:/fz/0601/solo-dogfeeding/code/58-paperless-ngx/src/documents/parsers.py#L25-L29)

各 Parser 支持的 MIME：
- TesseractParser：`application/pdf, image/*
- TikaParser：各种办公文档、图片
- TextParser：text/plain, text/csv 等
- MailParser：`message/rfc822` (.eml)
- RemoteParser：各种文档格式

### 5.3 _process_eml — 整封邮件作为 .eml

定义位置：[mail.py#L942-L1009](file:///d:/fz/0601/solo-dogfeeding/code/58-paperless-ngx/src/paperless_mail/mail.py#L942-L1009)

```
1. 在 SCRATCH_DIR 创建临时 .eml 文件
2. 特殊处理：将 "From" 头部移到最前（解决 magic 识别为 text/plain 而非 message/rfc822）
3. 写入邮件原始字节
4. 直接构造 consume_file 任务入队
```

---

## 六、合格附件进入文档消费队列

### 6.1 附件写入临时目录

定义位置：[mail.py#L860-L876](file:///d:/fz/0601/solo-dogfeeding/code/58-paperless-ngx/src/paperless_mail/mail.py#L860-L876)

```python
settings.SCRATCH_DIR.mkdir(parents=True, exist_ok=True)
temp_dir = Path(tempfile.mkdtemp(prefix="paperless-mail-", dir=settings.SCRATCH_DIR)
attachment_name = pathvalidate.sanitize_filename(att.filename)
temp_filename = temp_dir / attachment_name
temp_filename.write_bytes(att.payload)
```

- 在 `SCRATCH_DIR` 下创建 `paperless-mail-XXXX` 临时目录
- 附件文件名经 `pathvalidate.sanitize_filename` 清理非法字符
- 写入附件二进制内容

### 6.2 构造 ConsumableDocument

定义位置：[data_models.py#L162-L187](file:///d:/fz/0601/solo-dogfeeding/code/58-paperless-ngx/src/documents/data_models.py#L162-L187)

```python
input_doc = ConsumableDocument(
    source=DocumentSource.MailFetch,      # 来源标记为 MailFetch (3)
    original_file=temp_filename,     # 临时文件绝对路径
    mailrule_id=rule.pk,             # 关联规则 ID
)
# __post_init__ 中自动：
#   - original_file 转为绝对路径 resolve()
#   - magic.from_file() 再次检测文件 MIME 类型
```

DocumentSource 枚举：[data_models.py#L150-L158](file:///d:/fz/0601/solo-dogfeeding/code/58-paperless-ngx/src/documents/data_models.py#L150-L158)

### 6.3 构造 DocumentMetadataOverrides

定义位置：[data_models.py#L13-L37](file:///d:/fz/0601/solo-dogfeeding/code/58-paperless-ngx/src/documents/data_models.py#L13-L37)

```python
doc_overrides = DocumentMetadataOverrides(
    title=title,                                     # 主题或附件文件名（由 rule.assign_title_from
    filename=pathvalidate.sanitize_filename(att.filename),
    correspondent_id=correspondent.id if correspondent else None,
    document_type_id=doc_type.id if doc_type else None,
    tag_ids=tag_ids,                                   # rule.assign_tags 全部标签
    owner_id=rule.owner.id if (rule.assign_owner_from_rule and rule.owner else None,
)
```

Title 来源（`_get_title`）：[mail.py#L492-L510](file:///d:/fz/0601/solo-dogfeeding/code/58-paperless-ngx/src/paperless_mail/mail.py#L492-L510)
- FROM_SUBJECT：邮件主题
- FROM_FILENAME：附件文件名（不含扩展名）
- NONE：不指定

Correspondent 来源（`_get_correspondent`）：[mail.py#L512-L538](file:///d:/fz/0601/solo-dogfeeding/code/58-paperless-ngx/src/paperless_mail/mail.py#L512-L538)
- FROM_EMAIL：发件人邮箱地址（get_or_create）
- FROM_NAME：发件人显示名（退回邮箱地址）
- FROM_CUSTOM：rule.assign_correspondent 指定
- FROM_NOTHING：不指定

### 6.4 Celery 任务编排

定义位置：[mail.py#L896-L905](file:///d:/fz/0601/solo-dogfeeding/code/58-paperless-ngx/src/paperless_mail/mail.py#L896-L905)

```python
consume_task = consume_file.s(
    input_doc=input_doc,
    overrides=doc_overrides,
).set(headers={"trigger_source": PaperlessTask.TriggerSource.EMAIL_CONSUME})
```

### 6.5 queue_consumption_tasks — Chord 编排

定义位置：[mail.py#L334-L358](file:///d:/fz/0601/solo-dogfeeding/code/58-paperless-ngx/src/paperless_mail/mail.py#L334-L358)

```python
def queue_consumption_tasks(consume_tasks, rule, message):
    mail_action_task = apply_mail_action.s(rule_id, uid, subject, date)
    chord(header=consume_tasks, body=mail_action_task)
        .on_error(error_callback.s(...))
        .delay()
```

使用 Celery Chord 模式：
```
┌──────────────────────────────────────────────────────────┐
│  header（并行）：                                 │
│    consume_file(task1)  consume_file(task2) ...     │
└───────────────┬───────────────────────────────────────┘
                │ 全部成功完成
                ▼
┌──────────────────────────────────────────────────────────┐
│  body：apply_mail_action                         │
│    - 对邮件执行 IMAP 动作（标记已读/移动/删除等     │
│    - 写入 ProcessedMail 记录（SUCCESS/FAILED）             │
└──────────────────────────────────────────────────────────┘
                │ 任一 header 任务失败
                ▼
┌──────────────────────────────────────────────────────────┐
│  error_callback：error_callback                       │
│    - 记录 ProcessedMail（FAILED + traceback）            │
└──────────────────────────────────────────────────────────┘
```

### 6.6 apply_mail_action — 消费完成后动作

定义位置：[mail.py#L240-L304](file:///d:/fz/0601/solo-dogfeeding/code/58-paperless-ngx/src/paperless_mail/mail.py#L240-L304)

```
1. 重新连接邮箱
2. action.post_consume() 执行 IMAP 操作
3. ProcessedMail.objects.create(status="SUCCESS")
```

ProcessedMail 模型记录：[models.py#L316-L375](file:///d:/fz/0601/solo-dogfeeding/code/58-paperless-ngx/src/paperless_mail/models.py#L316-L375)
- rule / folder / uid（IMAP UID）/ subject / received / processed / status / error

### 6.7 无合格附件时的处理

mail.py#L922-L938](file:///d:/fz/0601/solo-dogfeeding/code/58-paperless-ngx/src/paperless_mail/mail.py#L922-L938)：若一封邮件所有附件都被过滤跳过（但 .eml 也没入队），且 ProcessedMail 尚无记录，则直接写入：
```python
ProcessedMail.objects.create(status="PROCESSED_WO_CONSUMPTION")
```
防止下次重复扫描。

---

## 七、consume_file 消费任务（文档侧）

定义位置：[tasks.py#L123-L220](file:///d:/fz/0601/solo-dogfeeding/code/58-paperless-ngx/src/documents/tasks.py#L123-L220)

邮件来源的文档与普通上传走同一套消费流水线：

```
consume_file(input_doc, overrides)
  └─ 插件链（非版本文档）：
     ├─ ConsumerPreflightPlugin   预检
     ├─ AsnCheckPlugin      ASN 检查
     ├─ CollatePlugin      双面扫描整理
     ├─ BarcodePlugin     条码分割
     ├─ AsnCheckPlugin    再次 ASN 检查（条码后）
     ├─ WorkflowTriggerPlugin  工作流触发
     └─ ConsumerPlugin    实际入库
```

ConsumerPlugin 内部调用 `Consumer.try_consume() → `documents/consumer.py）完成文档解析、OCR、归档、写入 originals / thumbnails / 存储路径计算等（与普通上传一致）。

---

## 八、完整调用链汇总

```
process_mail_accounts() [Celery Task]
  │
  └─ MailAccountHandler.handle_mail_account(account)
       ├─ get_mailbox()        建立 IMAP 连接
       ├─ mailbox_login()         登录
       │
       └─ 遍历 account.rules.order_by("order"):
       │    │
       │    └─ _handle_mail_rule(M, rule, ...)
       │         ├─ M.folder.set(rule.folder)    选择文件夹
       │         ├─ make_criterias()         构造 IMAP 查询
       │         ├─ M.fetch(criterias)          拉取邮件列表
       │         │
       │         └─ 遍历 messages:
       │              │
       │              ├─ 去重（rule_seen / consumed_messages / ProcessedMail）
       │              │
       │              └─ _handle_message(message, rule)
       │                   ├─ _preprocess_message()   PGP 解密等
       │                   │
       │                   ├─ _process_eml()       EML （可选）
       │                   │    └─ 写 .eml → ConsumableDocument → consume_file.s()
       │                   │
       │                   └─ _process_attachments()   逐个附件：
       │                        │
       │                        └─ for att in message.attachments:
       │                             ├─ content_disposition 关卡
       │                             ├─ filename_include 关卡
       │                             ├─ filename_exclude 关卡
       │                             ├─ magic.from_buffer → mime 关卡
       │                             │
       │                             ├─ 写临时文件
       │                             ├─ ConsumableDocument(MailFetch)
       │                             ├─ DocumentMetadataOverrides(title, correspondent, tags...)
       │                             └─ consume_file.s() 加入 consume_tasks 列表
       │
       └─ queue_consumption_tasks(consume_tasks, rule, message)
            │
            └─ chord(header=consume_tasks,
            │            body=apply_mail_action.s(...))
            │      .on_error(error_callback.s(...))
            │      .delay()
            │
            └─ 所有 consume_file 全部完成
                 │
                 └─ apply_mail_action()
                      ├─ IMAP 动作（MARK_READ/MOVE/DELETE/FLAG/TAG
                      └─ ProcessedMail.objects.create(SUCCESS/FAILED)
```
