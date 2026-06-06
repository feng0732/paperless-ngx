# Paperless-ngx 邮件导入链路分析

## 一、总体架构

邮件导入模块位于 `src/paperless_mail/`，主要组件：

| 文件 | 职责 |
|------|------|
| [models.py](file:///d:/fz/0601/solo-dogfeeding/code/58-paperless-ngx/src/paperless_mail/models.py) | MailAccount / MailRule / ProcessedMail 模型 |
| [mail.py](file:///d:/fz/0601/solo-dogfeeding/code/58-paperless-ngx/src/paperless_mail/mail.py) | 核心处理逻辑（MailAccountHandler） |
| [tasks.py](file:///d:/fz/0601/solo-dogfeeding/code/58-paperless-ngx/src/paperless_mail/tasks.py) | Celery 任务入口 |
| [preprocessor.py](file:///d:/fz/0601/solo-dogfeeding/code/58-paperless-ngx/src/paperless_mail/preprocessor.py) | 邮件预处理器（如 PGP 解密） |
| [oauth.py](file:///d:/fz/0601/solo-dogfeeding/code/58-paperless-ngx/src/paperless_mail/oauth.py) | OAuth 认证管理 |
| [registry.py](file:///d:/fz/0601/solo-dogfeeding/code/58-paperless-ngx/src/paperless/parsers/registry.py) | 解析器注册表（附件 MIME 支持判定依赖此表） |

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
  - `EVERYTHING`（3）邮件 + 附件都处理
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
| TAG（5） | 排除已打标签 | 打 IMAP keyword / Gmail 标签 / Apple Mail 颜色标签 |

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
   └─ 密码认证（ASCII / UTF-8 AUTH=PLAIN）
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
   ├─ rule_seen_messages：本次 fetch 结果去重
   ├─ consumed_messages：已被前面规则处理的邮件跳过
   └─ ProcessedMail：DB 中已有记录跳过
5. _handle_message() 处理单封邮件
6. 若 stop_processing=True 且本次有文档入队，则 break
```

### 4.3 make_criterias — IMAP 查询条件构造

定义位置：[mail.py#L383-L411](file:///d:/fz/0601/solo-dogfeeding/code/58-paperless-ngx/src/paperless_mail/mail.py#L383-L411)

MailAction 与过滤条件：
- MARK_READ：`seen=False`（仅未读）
- FLAG：`flagged=False`（仅未加星标）
- TAG：
  - Apple Mail 颜色：`flagged=False`
  - Gmail 标签：`NOT(gmail_label=X) AND no_keyword=X`
  - 普通 keyword：`no_keyword=X`
- DELETE / MOVE：无过滤（每次都扫描全部）

### 4.4 邮件预处理器

定义位置：[preprocessor.py](file:///d:/fz/0601/solo-dogfeeding/code/58-paperless-ngx/src/paperless_mail/preprocessor.py)

当前注册的预处理器（[mail.py#L454-L456](file:///d:/fz/0601/solo-dogfeeding/code/58-paperless-ngx/src/paperless_mail/mail.py#L454-L456)）：
- `MailMessageDecryptor`：PGP 加密邮件解密（`EMAIL_ENABLE_GPG_DECRYPTOR` 开启）

---

## 五、附件接收与跳过规则

### 5.1 单封邮件处理入口：_handle_message

定义位置：[mail.py#L715-L759](file:///d:/fz/0601/solo-dogfeeding/code/58-paperless-ngx/src/paperless_mail/mail.py#L715-L759)

```
_handle_message(message, rule)
  ├─ _preprocess_message() 预处理（解密等）
  ├─ 若 ATTACHMENTS_ONLY 且无附件 → 直接返回 0
  ├─ consumption_scope 判断（顺序执行，各自独立入队）：
  │   ├─ EML_ONLY / EVERYTHING → _process_eml()  .eml 文件
  │   └─ ATTACHMENTS_ONLY / EVERYTHING → _process_attachments() 逐个附件
  └─ 返回成功处理的文件数
```

**关键**：当 `consumption_scope = EVERYTHING` 时，`_process_eml` 和 `_process_attachments` 各自内部独立调用 `queue_consumption_tasks()`，会向 Celery 提交两个独立的 chord（详见第六节）。

### 5.2 _process_attachments — 附件逐个筛选核心

定义位置：[mail.py#L798-L940](file:///d:/fz/0601/solo-dogfeeding/code/58-paperless-ngx/src/paperless_mail/mail.py#L798-L940)

每个附件经过以下四道关卡，按顺序判断：

#### 关卡 1：Content-Disposition 检查

```python
if (
    att.content_disposition != "attachment"
    and rule.attachment_type == MailRule.AttachmentProcessing.ATTACHMENTS_ONLY
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

定义位置：[mail.py#L780-L796](file:///d:/fz/0601/solo-dogfeeding/code/58-paperless-ngx/src/paperless_mail/mail.py#L780-L796)

```python
def filename_exclusion_matches(filter_exclude, filename):
    # 逗号分隔多个 pattern，大小写不敏感，fnmatch 通配符匹配
    # 任一 pattern 匹配 → True（跳过）
```

- `filter_attachment_filename_exclude` 任一 pattern 匹配即跳过
- 例：`signature*.png, image*` → 跳过签名图片
- 为空时默认通过

#### 关卡 4：MIME 类型支持检测

定义位置：[mail.py#L849-L914](file:///d:/fz/0601/solo-dogfeeding/code/58-paperless-ngx/src/paperless_mail/mail.py#L849-L914)

```python
mime_type = magic.from_buffer(att.payload, mime=True)  # 读取附件二进制内容检测真实类型
if is_mime_type_supported(mime_type):
    # 通过，继续
else:
    跳过 → "Skipping attachment ... since guessed mime type ... is not supported"
```

**注意**：不使用邮件声明的 Content-Type，而是用 python-magic 从实际文件内容嗅探真实 MIME 类型。

### 5.3 解析器注册表：MIME 支持范围的精确核准

`is_mime_type_supported()` 定义：[documents/parsers.py#L25-L29](file:///d:/fz/0601/solo-dogfeeding/code/58-paperless-ngx/src/documents/parsers.py#L25-L29)

```python
def is_mime_type_supported(mime_type: str) -> bool:
    return get_parser_registry().get_parser_for_file(mime_type, "") is not None
```

其内部调用 `ParserRegistry.get_parser_for_file()`（[registry.py#L332-L393](file:///d:/fz/0601/solo-dogfeeding/code/58-paperless-ngx/src/paperless/parsers/registry.py#L332-L393)），判断机制如下：

**注册表内置解析器注册顺序**（[registry.py#L192-L210](file:///d:/fz/0601/solo-dogfeeding/code/58-paperless-ngx/src/paperless/parsers/registry.py#L192-L210)）：

```
TextDocumentParser → RemoteDocumentParser → TikaDocumentParser → MailDocumentParser → RasterisedDocumentParser
```

**遍历与选择规则**：
1. 先遍历第三方外部解析器（entrypoints `paperless_ngx.parsers`），再遍历内置解析器
2. 对每个解析器依次检查：
   - `mime_type in parser.supported_mime_types()`
   - `parser.score(mime_type, filename, path) is not None`（用于条件性启用/禁用）
3. 取 `score` 最高者；分数相同则先遍历到的胜出（外部插件优先于内置）
4. 只要有任一解析器能处理，`is_mime_type_supported()` 就返回 True

**五个内置解析器的精确 MIME 支持**：

| 解析器 | 支持 MIME → 扩展名 | 分数 | 启用条件 |
|------|------|------|------|
| **RasterisedDocumentParser**（Tesseract OCR）<br>[tesseract.py#L47-L95](file:///d:/fz/0601/solo-dogfeeding/code/58-paperless-ngx/src/paperless/parsers/tesseract.py#L47-L95) | `application/pdf` → .pdf<br>`image/jpeg` → .jpg<br>`image/png` → .png<br>`image/tiff` → .tif<br>`image/gif` → .gif<br>`image/bmp` → .bmp<br>`image/webp` → .webp<br>`image/heic` | 10 | 始终启用 |
| **RemoteDocumentParser**（Azure AI 等远程 OCR）<br>[remote.py#L35-L149](file:///d:/fz/0601/solo-dogfeeding/code/58-paperless-ngx/src/paperless/parsers/remote.py#L35-L149) | `application/pdf` → .pdf<br>`image/png` → .png<br>`image/jpeg` → .jpg<br>`image/tiff` → .tiff<br>`image/bmp` → .bmp<br>`image/gif` → .gif<br>`image/webp` | 20 | `REMOTE_OCR_ENGINE` + API Key + Endpoint 必须完整配置；<br>分数比 Tesseract 高，配置生效时自动优先 |
| **TikaDocumentParser**（办公文档，需 Tika + Gotenberg）<br>[tika.py#L42-L135](file:///d:/fz/0601/solo-dogfeeding/code/58-paperless-ngx/src/paperless/parsers/tika.py#L42-L135) | `application/msword` → .doc<br>`application/vnd.openxmlformats-officedocument.wordprocessingml.document` → .docx<br>`application/vnd.ms-excel` → .xls<br>`application/vnd.openxmlformats-officedocument.spreadsheetml.sheet` → .xlsx<br>`application/vnd.ms-powerpoint` → .ppt<br>`application/vnd.openxmlformats-officedocument.presentationml.presentation` → .pptx<br>`application/vnd.openxmlformats-officedocument.presentationml.slideshow` → .ppsx<br>`application/vnd.oasis.opendocument.presentation` → .odp<br>`application/vnd.oasis.opendocument.spreadsheet` → .ods<br>`application/vnd.oasis.opendocument.text` → .odt<br>`application/vnd.oasis.opendocument.graphics` → .odg<br>`text/rtf` → .rtf | 10 | `TIKA_ENABLED=True` 且 Gotenberg 可用 |
| **MailDocumentParser**（.eml 邮件文件）<br>[mail.py#L109-L133](file:///d:/fz/0601/solo-dogfeeding/code/58-paperless-ngx/src/paperless/parsers/mail.py#L109-L133) | `message/rfc822` → .eml | 10 | 始终启用 |
| **TextDocumentParser**（纯文本）<br>[text.py#L34-L105](file:///d:/fz/0601/solo-dogfeeding/code/58-paperless-ngx/src/paperless/parsers/text.py#L34-L105) | `text/plain` → .txt<br>`text/csv` → .csv<br>`application/csv` → .csv | 10 | 始终启用 |

> **结论**：邮件附件能被接收的 MIME 覆盖范围 = 以上五个内置解析器（以及任何已安装的第三方插件解析器）的 MIME 并集，且对应解析器的 `score()` 不返回 `None`（即启用条件满足）。

### 5.4 _process_eml — 整封邮件作为 .eml

定义位置：[mail.py#L942-L1009](file:///d:/fz/0601/solo-dogfeeding/code/58-paperless-ngx/src/paperless_mail/mail.py#L942-L1009)

```
1. 在 SCRATCH_DIR 创建临时 .eml 文件
2. 特殊处理：将 "From" 头部移到最前（解决 magic 识别为 text/plain 而非 message/rfc822）
3. 写入邮件原始字节
4. 直接构造 consume_file 任务并调用 queue_consumption_tasks() 立即入队
```

---

## 六、合格附件进入文档消费队列

### 6.1 附件写入临时目录

定义位置：[mail.py#L860-L876](file:///d:/fz/0601/solo-dogfeeding/code/58-paperless-ngx/src/paperless_mail/mail.py#L860-L876)

```python
settings.SCRATCH_DIR.mkdir(parents=True, exist_ok=True)
temp_dir = Path(tempfile.mkdtemp(prefix="paperless-mail-", dir=settings.SCRATCH_DIR))
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
    source=DocumentSource.MailFetch,     # 来源标记为 MailFetch (3)
    original_file=temp_filename,         # 临时文件绝对路径
    mailrule_id=rule.pk,                 # 关联规则 ID
)
# __post_init__ 中自动：
#   - original_file 转为绝对路径 resolve()
#   - magic.from_file() 再次检测文件 MIME 类型
```

### 6.3 构造 DocumentMetadataOverrides

定义位置：[data_models.py#L13-L37](file:///d:/fz/0601/solo-dogfeeding/code/58-paperless-ngx/src/documents/data_models.py#L13-L37)

```python
doc_overrides = DocumentMetadataOverrides(
    title=title,
    filename=pathvalidate.sanitize_filename(att.filename),
    correspondent_id=correspondent.id if correspondent else None,
    document_type_id=doc_type.id if doc_type else None,
    tag_ids=tag_ids,
    owner_id=rule.owner.id if (rule.assign_owner_from_rule and rule.owner) else None,
)
```

Title 来源（`_get_title`）：[mail.py#L492-L510](file:///d:/fz/0601/solo-dogfeeding/code/58-paperless-ngx/src/paperless_mail/mail.py#L492-L510)
- `FROM_SUBJECT`：邮件主题
- `FROM_FILENAME`：附件文件名（不含扩展名）
- `NONE`：不指定

Correspondent 来源（`_get_correspondent`）：[mail.py#L512-L538](file:///d:/fz/0601/solo-dogfeeding/code/58-paperless-ngx/src/paperless_mail/mail.py#L512-L538)
- `FROM_EMAIL`：发件人邮箱地址（get_or_create）
- `FROM_NAME`：发件人显示名（失败退回邮箱地址）
- `FROM_CUSTOM`：`rule.assign_correspondent` 指定
- `FROM_NOTHING`：不指定

### 6.4 queue_consumption_tasks — Chord 编排与参数传递

定义位置：[mail.py#L334-L358](file:///d:/fz/0601/solo-dogfeeding/code/58-paperless-ngx/src/paperless_mail/mail.py#L334-L358)

`queue_consumption_tasks` 是关键字-only 函数（签名中 `*` 之后所有参数必须以关键字方式传入）：

```python
def queue_consumption_tasks(
    *,
    consume_tasks: list[Signature],   # consume_file.s() 的签名列表
    rule: MailRule,                   # 当前匹配的规则
    message: MailMessage,             # 当前处理的邮件对象
) -> None:
```

#### 参数传递链：从 queue_consumption_tasks 到 apply_mail_action / error_callback

```
queue_consumption_tasks(consume_tasks=..., rule=R, message=M)
  │
  ├─ 构造 apply_mail_action.s() 的签名（body）
  │   apply_mail_action.s(
  │       rule_id          = R.pk,           # int：规则 ID
  │       message_uid      = M.uid,          # str：IMAP UID（来自 message.uid）
  │       message_subject  = M.subject,      # str：邮件主题（来自 message.subject）
  │       message_date     = M.date,         # datetime：邮件日期（来自 message.date）
  │   )
  │   ⚠️  注意：folder 并未作为参数传入 apply_mail_action
  │
  ├─ 构造 error_callback.s() 的签名（on_error 回调）
  │   error_callback.s(
  │       rule_id          = R.pk,
  │       message_uid      = M.uid,
  │       message_subject  = M.subject,
  │       message_date     = M.date,
  │   )
  │
  └─ chord(header=consume_tasks, body=apply_mail_action_sig)
         .on_error(error_callback_sig)
         .delay()
```

#### apply_mail_action 函数签名与内部取 folder 的方式

`apply_mail_action` 定义：[mail.py#L240-L304](file:///d:/fz/0601/solo-dogfeeding/code/58-paperless-ngx/src/paperless_mail/mail.py#L240-L304)

```python
@shared_task
def apply_mail_action(
    result: list,              # Celery Chord body 自动注入：所有 header 任务的返回值列表
    rule_id: int,              # ← 来自 queue_consumption_tasks 的 R.pk
    message_uid: str,          # ← 来自 M.uid
    message_subject: str,      # ← 来自 M.subject
    message_date: datetime.datetime,  # ← 来自 M.date
) -> None:
```

内部通过 `rule_id` 反向查 rule 再取 `rule.folder`，而不是接收显式参数：

```python
rule = MailRule.objects.get(pk=rule_id)           # 查数据库取 rule
...
ProcessedMail.objects.create(
    rule    = rule,
    folder  = rule.folder,        # ⚠️ folder 从 rule 对象上取，不是参数传入
    uid     = message_uid,
    subject = message_subject,
    received= message_date,
    status  = "SUCCESS",
)
```

`error_callback` 定义：[mail.py#L307-L331](file:///d:/fz/0601/solo-dogfeeding/code/58-paperless-ngx/src/paperless_mail/mail.py#L307-L331)

```python
@shared_task
def error_callback(
    request,                    # Celery on_error 自动注入：失败任务的 request
    exc,                        # Celery on_error 自动注入：异常对象
    tb,                         # Celery on_error 自动注入：traceback
    rule_id: int,               # ← 来自 queue_consumption_tasks
    message_uid: str,
    message_subject: str,
    message_date: datetime.datetime,
) -> None:
```

内部同样通过 `rule = MailRule.objects.get(pk=rule_id)` 取 `rule.folder` 写入 ProcessedMail。

#### 单个 Chord 的执行模式：

```
┌────────────────────────────────────────────────────────────────────┐
│  header（并行）：所有 consume_file 任务                       │
│    consume_file(附件1)  consume_file(附件2)  ...                  │
└───────────────────────────┬────────────────────────────────────────┘
                            │ 全部成功完成 → result = [ret1, ret2, ...]
                            ▼
┌────────────────────────────────────────────────────────────────────┐
│  body：apply_mail_action(result, rule_id, message_uid,             │
│                           message_subject, message_date)            │
│    1. rule = MailRule.objects.get(pk=rule_id)                       │
│    2. 重新连接 IMAP，M.folder.set(rule.folder)                      │
│    3. action.post_consume(M, message_uid, rule.action_parameter)    │
│    4. ProcessedMail.objects.create(                                 │
│          folder=rule.folder,  uid=message_uid, ...)                 │
└────────────────────────────────────────────────────────────────────┘
                            │ 任一 header 任务失败
                            ▼
┌────────────────────────────────────────────────────────────────────┐
│  error_callback(request, exc, tb, rule_id, message_uid, ...)       │
│    1. rule = MailRule.objects.get(pk=rule_id)                       │
│    2. ProcessedMail.objects.create(                                 │
│          folder=rule.folder, status="FAILED", error=traceback)      │
└────────────────────────────────────────────────────────────────────┘
```

### 6.5 EVERYTHING 模式下的双 Chord 衔接

这是代码中最容易被误解的关键点。当 `consumption_scope = EVERYTHING`（邮件 + 附件都处理）时：

`_handle_message` 在 [mail.py#L737-L757](file:///d:/fz/0601/solo-dogfeeding/code/58-paperless-ngx/src/paperless_mail/mail.py#L737-L757) **顺序调用**两个处理函数：

```python
# 第一步：处理 .eml
if rule.consumption_scope in (EML_ONLY, EVERYTHING):
    processed_elements += self._process_eml(...)  # ← 内部立即调用 queue_consumption_tasks()

# 第二步：处理附件
if rule.consumption_scope in (ATTACHMENTS_ONLY, EVERYTHING):
    processed_elements += self._process_attachments(...)  # ← 内部再次调用 queue_consumption_tasks()
```

**两个处理函数各自独立入队**，产生两个互不相关的 Celery Chord：

```
_handle_message()
  │
  ├─ _process_eml()
  │   └─ queue_consumption_tasks(consume_tasks=[eml_consume_task], rule=R, message=M)
  │        └─ Chord A：
  │           ├─ header = [consume_file(eml)]
  │           ├─ body   = apply_mail_action(result, rule_id, message_uid, message_subject, message_date)
  │           │            内部通过 rule_id 查 rule 后取 rule.folder
  │           │            ← 写 1 条 ProcessedMail(SUCCESS/FAILED)
  │           └─ .on_error(error_callback(request, exc, tb, rule_id, message_uid, ...))
  │
  └─ _process_attachments()
      ├─ 遍历附件，构造 consume_tasks = [att1_task, att2_task, ...]
      └─ queue_consumption_tasks(consume_tasks=consume_tasks, rule=R, message=M)
           └─ Chord B：
              ├─ header = [consume_file(att1), consume_file(att2), ...]
              ├─ body   = apply_mail_action(result, rule_id, message_uid, message_subject, message_date)
              │            内部通过 rule_id 查 rule 后取 rule.folder
              │            ← 再写 1 条 ProcessedMail(SUCCESS/FAILED)
              └─ .on_error(error_callback(request, exc, tb, rule_id, message_uid, ...))
```

**两个 Chord 之间没有任何同步或依赖关系**，Celery 会并行调度它们各自的 header。

#### 双 Chord 对邮件动作（MailAction）的影响：

| MailAction | 两次执行的实际效果 |
|------|------|
| **MARK_READ** | 幂等；两次 `\Seen` 标记无副作用 |
| **FLAG** | 幂等；两次 `\Flagged` 标记无副作用 |
| **TAG**（keyword / Gmail 标签 / Apple Mail 颜色） | 幂等；重复打标签无副作用 |
| **MOVE** | 第一次移动成功，邮件 UID 离开原文件夹；第二次执行 `post_consume()` 时 IMAP 找不到该 UID，会抛异常并在 error_callback 中记录 `FAILED` |
| **DELETE** | 类似 MOVE，第二次找不到邮件 UID |

#### 双 Chord 对 ProcessedMail 记录的影响：

`ProcessedMail` 模型（[models.py#L316-L375](file:///d:/fz/0601/solo-dogfeeding/code/58-paperless-ngx/src/paperless_mail/models.py#L316-L375)）**没有**对 `(rule, folder, uid)` 做唯一约束。因此：
- 当 Chord A 和 Chord B 的 body（`apply_mail_action`）都成功时，会产生 **两条** `ProcessedMail(status="SUCCESS")` 记录
- 其中任一 chord 任一 header 任务失败时，会额外产生一条 `FAILED` 记录
- 下次取信时，`_handle_mail_rule` 中 `ProcessedMail.objects.filter(rule=rule, uid=message.uid, folder=rule.folder).exists()` 只要存在任意一条就跳过该邮件

#### _process_attachments 中对重复 ProcessedMail 的保护：

当附件全部被过滤跳过（没有任何合格附件）时，`_process_attachments` 的 [mail.py#L922-L938](file:///d:/fz/0601/solo-dogfeeding/code/58-paperless-ngx/src/paperless_mail/mail.py#L922-L938) 会先检查 `ProcessedMail` 是否已经存在，只有不存在时才写入一条 `PROCESSED_WO_CONSUMPTION` 状态的记录——这是为了避免与 `_process_eml` 的 Chord A 已经写入的记录重复。

### 6.6 apply_mail_action — 消费完成后动作

定义位置：[mail.py#L240-L304](file:///d:/fz/0601/solo-dogfeeding/code/58-paperless-ngx/src/paperless_mail/mail.py#L240-L304)

函数签名：
```python
def apply_mail_action(
    result: list,              # Chord body 自动注入：header 任务返回值列表
    rule_id: int,              # ← 来自 queue_consumption_tasks
    message_uid: str,          # ← 来自 message.uid
    message_subject: str,      # ← 来自 message.subject
    message_date: datetime.datetime,  # ← 来自 message.date
) -> None:
```

执行流程：
```
1. rule = MailRule.objects.get(pk=rule_id)   # 用 rule_id 反查 rule，取 rule.folder
2. account = MailAccount.objects.get(pk=rule.account.pk)
3. 重新连接邮箱（与 handle_mail_account 相同逻辑）
4. M.folder.set(rule.folder)                  # folder 来自 rule，非参数传入
5. action.post_consume(M, message_uid, rule.action_parameter)  执行 IMAP 动作
6. ProcessedMail.objects.create(
       owner=rule.owner,
       rule=rule,
       folder=rule.folder,      # folder 来自 rule 对象
       uid=message_uid,         # 来自参数 message_uid
       subject=message_subject, # 来自参数 message_subject
       received=message_date,   # 来自参数 message_date（自动补 timezone）
       status="SUCCESS",
   )
7. 异常时：写入 status="FAILED" 且带上 traceback
```

### 6.7 无合格附件时的处理

定义位置：[mail.py#L922-L938](file:///d:/fz/0601/solo-dogfeeding/code/58-paperless-ngx/src/paperless_mail/mail.py#L922-L938)

若一封邮件所有附件都被过滤跳过（没有可消费的附件），且 `ProcessedMail` 尚无记录，则直接同步写入：
```python
ProcessedMail.objects.create(status="PROCESSED_WO_CONSUMPTION")
```
防止下次取信时重复扫描。

---

## 七、consume_file 消费任务（文档侧）

定义位置：[tasks.py#L123-L220](file:///d:/fz/0601/solo-dogfeeding/code/58-paperless-ngx/src/documents/tasks.py#L123-L220)

邮件来源的文档与普通上传走同一套消费流水线：

```
consume_file(input_doc, overrides)
  └─ 插件链（非版本文档）：
     ├─ ConsumerPreflightPlugin   预检
     ├─ AsnCheckPlugin            ASN 检查
     ├─ CollatePlugin             双面扫描整理
     ├─ BarcodePlugin             条码分割
     ├─ AsnCheckPlugin            再次 ASN 检查（条码后）
     ├─ WorkflowTriggerPlugin     工作流触发
     └─ ConsumerPlugin            实际入库
```

`ConsumerPlugin` 内部调用 `Consumer.try_consume()`（[documents/consumer.py](file:///d:/fz/0601/solo-dogfeeding/code/58-paperless-ngx/src/documents/consumer.py)）完成文档解析、OCR、归档、写入 originals / thumbnails 等操作（与普通上传完全一致）。

---

## 八、完整调用链汇总

```
process_mail_accounts() [Celery Task]
  │
  └─ MailAccountHandler.handle_mail_account(account)
       ├─ get_mailbox()         建立 IMAP 连接
       ├─ mailbox_login()       登录
       │
       └─ 遍历 account.rules.order_by("order"):
            │
            └─ _handle_mail_rule(M, rule, ...)
                 ├─ M.folder.set(rule.folder)     选择文件夹
                 ├─ make_criterias()              构造 IMAP 查询
                 ├─ M.fetch(criterias)            拉取邮件列表
                 │
                 └─ 遍历 messages:
                      │
                      ├─ 去重（rule_seen / consumed_messages / ProcessedMail）
                      │
                      └─ _handle_message(message, rule)
                           ├─ _preprocess_message()     PGP 解密等
                           │
                           ├─ EML_ONLY / EVERYTHING:
                           │    └─ _process_eml()
                           │         ├─ 写临时 .eml
                           │         ├─ ConsumableDocument(MailFetch)
                           │         ├─ DocumentMetadataOverrides
                           │         └─ queue_consumption_tasks(consume_tasks=[eml_task], rule=R, message=M)  ← Chord A 入队
                           │
                           └─ ATTACHMENTS_ONLY / EVERYTHING:
                                └─ _process_attachments()
                                     └─ for att in message.attachments:
                                          ├─ content_disposition 关卡
                                          ├─ filename_include 关卡
                                          ├─ filename_exclude 关卡
                                          ├─ magic.from_buffer → MIME 关卡（查解析器注册表）
                                          │
                                          ├─ 合格：写临时文件
                                          │       ConsumableDocument + Overrides
                                          │       consume_file.s() 加入列表
                                          └─ 全部附件处理完毕：
                                               ├─ 有任务：queue_consumption_tasks(consume_tasks=[att1, att2...], rule=R, message=M)  ← Chord B 入队
                                               └─ 无任务：ProcessedMail(PROCESSED_WO_CONSUMPTION) （需 .eml 侧尚未写入）
```

Chord A 和 Chord B 在 Celery 中独立执行，各自完成后分别：
- 执行 `apply_mail_action(result, rule_id, message_uid, message_subject, message_date)`：内部通过 rule_id 查 rule 得到 folder，再执行 IMAP 动作并写 `ProcessedMail(SUCCESS/FAILED)`
- 出错时通过 `error_callback(request, exc, tb, rule_id, message_uid, message_subject, message_date)` 写 `ProcessedMail(FAILED)`
