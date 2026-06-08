# Paperless-ngx 敏感字段保护机制精确分析

## 概述

Paperless-ngx 的敏感数据保护**不存在数据库层面的应用层加密**。全部保护机制分布在两个独立边界：

1. **API 响应掩码边界**：部分敏感字段在 REST API 响应中被替换为星号，仅影响 HTTP 响应的显示层
2. **导出文件加密边界**：部分敏感字段在 `document_exporter --passphrase` 导出时使用 Fernet 对称加密，仅影响导出包

**核心事实**：数据库中，除 Django 内建 `User.password` 由框架自行 PBKDF2 哈希外，其余所有敏感字段均为**明文存储**。

---

## 一、三类保护边界的精确状态

以下逐项对照代码，精确说明每个敏感字段在三个边界上的实际状态。

### 1.1 边界一：数据库存储（DB 层）

**DB 层无任何应用层字段加密**。所有项目自定义敏感字段均以原始形态落盘。

| 模型 | 字段 | 字段定义类型 | DB 实际存储 | 证据位置 |
|------|------|-------------|-------------|----------|
| `MailAccount` | `password` | `TextField` | 明文 | [src/paperless_mail/models.py](src/paperless_mail/models.py#L45) |
| `MailAccount` | `refresh_token` | `TextField(blank=True, null=True)` | 明文 | [src/paperless_mail/models.py](src/paperless_mail/models.py#L65-L72) |
| `ApplicationConfiguration` | `llm_api_key` | `CharField(max_length=1024, blank=True, null=True)` | 明文 | [src/paperless/models.py](src/paperless/models.py#L328-L333) |
| `SocialToken`（django-allauth） | `token` | 第三方 CharField | 明文 | 不在项目代码中 |
| `SocialToken`（django-allauth） | `token_secret` | 第三方 CharField | 明文 | 不在项目代码中 |
| `User`（Django 内建） | `password` | Django 内建 | PBKDF2 哈希（非本项目实现） | Django contrib.auth |

**重要修正**：DB 层不存在"受保护"或"加密"的概念。此处的全部状态只有两种——要么明文，要么由框架（Django）自行哈希，均与本项目的保护机制无关。

### 1.2 边界二：API 响应掩码

API 掩码通过 `ObfuscatedPasswordField` 序列化字段实现，**仅作用于 HTTP 响应层**，不影响数据存储形态。

`ObfuscatedPasswordField` 行为精确描述：
- `to_representation(value)` → 返回 `"*" * max(10, len(value))`，替换响应中的明文为星号
- `to_internal_value(data)` → 直接 `return data`，对提交值不做任何变换
- 证据位置：[src/paperless_mail/serialisers.py](src/paperless_mail/serialisers.py#L16-L25)

以下是所有 API 暴露的敏感字段的精确状态：

| 序列化器 | 字段 | 是否使用 `ObfuscatedPasswordField` | 在 `fields` 列表中？ | API 响应状态 | 证据位置 |
|----------|------|:----------------------------------:|:-------------------:|-------------|----------|
| `MailAccountSerializer` | `password` | ✅ | ✅ 是 | 返回星号掩码 | [src/paperless_mail/serialisers.py](src/paperless_mail/serialisers.py#L29-L49) |
| `MailAccountSerializer` | `refresh_token` | — | ❌ 不在 fields 中 | **API 不暴露** | [src/paperless_mail/serialisers.py](src/paperless_mail/serialisers.py#L33-L49) |
| `UserSerializer` | `password` | ✅ | ✅ 是 | 返回星号掩码 | [src/paperless/serialisers.py](src/paperless/serialisers.py#L77-L109) |
| `ProfileSerializer` | `password` | ✅ | ✅ 是 | 返回星号掩码 | [src/paperless/serialisers.py](src/paperless/serialisers.py#L179-L209) |
| `ApplicationConfigurationSerializer` | `llm_api_key` | ✅ | ✅ 是（`fields = "__all__"`） | 返回星号掩码 | [src/paperless/serialisers.py](src/paperless/serialisers.py#L212-L296) |
| `SocialAccountSerializer` | `token` / `token_secret` | — | ❌ 不在 fields 中 | **API 不暴露**（仅序列化 id/provider/name） | [src/paperless/serialisers.py](src/paperless/serialisers.py#L161-L176) |

**API 掩码复用链路**：

```
定义处: src/paperless_mail/serialisers.py (ObfuscatedPasswordField)
    │
    ▼ import
使用处: src/paperless/serialisers.py
    │
    ├─> UserSerializer.password           (L78)
    ├─> ProfileSerializer.password        (L181)
    └─> ApplicationConfigurationSerializer.llm_api_key  (L217)
```

**掩码提交时的跳过逻辑（防止星号覆盖真实值）**：

| 序列化器 | 跳过逻辑实现 | 证据位置 |
|----------|-------------|----------|
| `MailAccountSerializer` | `update()` 中检测 `replace("*", "") == ""` 为真时 `pop("password")` | [src/paperless_mail/serialisers.py](src/paperless_mail/serialisers.py#L51-L58) |
| `UserSerializer` | `update()` 中通过 `PasswordValidationMixin._has_real_password()` 判断，非真实值时不调用 `set_password()` | [src/paperless/serialisers.py](src/paperless/serialisers.py#L30-L44), [L114-L121](src/paperless/serialisers.py#L114-L121) |
| `ProfileSerializer` | 同上，继承 `PasswordValidationMixin` | [src/paperless/serialisers.py](src/paperless/serialisers.py#L179) |
| `ApplicationConfigurationSerializer` | `run_validation()` 中检测 `replace("*", "")` 长度为 0 时 `del data["llm_api_key"]` | [src/paperless/serialisers.py](src/paperless/serialisers.py#L222-L235) |

### 1.3 边界三：导出文件加密

仅在执行 `document_exporter --passphrase <口令>` 时生效。加密清单硬编码于 `CryptMixin.CRYPT_FIELDS`。

加密配置精确列表：

| 模型标识 | 导出 JSON 键名 | 加密字段 | 证据位置 |
|----------|---------------|----------|----------|
| `paperless_mail.mailaccount` | `mail_accounts` | `password`, `refresh_token` | [src/documents/management/commands/mixins.py](src/documents/management/commands/mixins.py#L54-L70) |
| `socialaccount.socialtoken` | `social_tokens` | `token`, `token_secret` | [src/documents/management/commands/mixins.py](src/documents/management/commands/mixins.py#L63-L70) |

**未纳入导出加密的敏感字段**：
- `ApplicationConfiguration.llm_api_key` → 导出时**明文**写入 `manifest.json`
- `User.password` → 不在 `CRYPT_FIELDS` 中，但本身是 PBKDF2 哈希值

---

## 二、三类边界状态总对照表

本表格取代之前容易误导的"三层全覆盖"等表述，精确呈现每个字段在各边界上的真实状态：

| 字段 | DB 存储 | API 响应 | 导出文件（带 passphrase） |
|------|---------|----------|--------------------------|
| `MailAccount.password` | 明文 TextField | ✅ 星号掩码 | ✅ Fernet 加密 |
| `MailAccount.refresh_token` | 明文 TextField | ❌ 不在序列化字段中，API 不暴露 | ✅ Fernet 加密 |
| `SocialToken.token` | 明文（第三方） | ❌ 不在任何序列化字段中，API 不暴露 | ✅ Fernet 加密 |
| `SocialToken.token_secret` | 明文（第三方） | ❌ 不在任何序列化字段中，API 不暴露 | ✅ Fernet 加密 |
| `ApplicationConfiguration.llm_api_key` | 明文 CharField | ✅ 星号掩码 | ❌ 明文（保护缺口） |
| `User.password` | PBKDF2 哈希（Django） | ✅ 星号掩码 | ❌ 不在 CRYPT_FIELDS，但本身是哈希值 |

---

## 三、核心加密实现：CryptMixin（导出加密边界）

### 3.1 加密架构

`CryptMixin` 是导入导出命令的共享混入类，基于 Python `cryptography` 库实现：

- **密钥派生算法**：PBKDF2-HMAC-SHA256
- **对称加密算法**：Fernet（内部封装 AES-128-CBC + HMAC-SHA256）
- **盐值长度**：16 字节（加密安全随机数）
- **迭代次数**：1,000,000 次（与 Django 默认密码哈希迭代一致）
- **密钥长度**：32 字节

核心定义位置：[src/documents/management/commands/mixins.py](src/documents/management/commands/mixins.py#L23-L139)

### 3.2 加密流程（数据写入导出文件）

```
用户口令 (passphrase)
        │
        ▼
  os.urandom(16) ──► salt (hex 字符串，存入 metadata.json)
        │
        ▼
  PBKDF2HMAC (SHA256, 1,000,000 次迭代)
        │
        ▼
  base64.urlsafe_b64encode() ──► Fernet 密钥
        │
        ▼
  Fernet.encrypt(明文.encode("utf-8"))
        │
        ▼
  .hex() ──► 加密后的十六进制字符串（写入 manifest.json）
```

关键方法：
- [setup_crypto()](src/documents/management/commands/mixins.py#L102-L126)：初始化加密上下文，生成盐值并派生 Fernet 密钥
- [encrypt_string()](src/documents/management/commands/mixins.py#L128-L133)：加密单个明文字符串为 hex 字符串

### 3.3 解密流程（从导出文件读取）

```
metadata.json 中恢复的 salt（hex）
        │
        ▼
  PBKDF2HMAC (同参数) ──► 相同 Fernet 密钥
        │
        ▼
  bytes.fromhex() ──► Fernet token 字节
        │
        ▼
  Fernet.decrypt()
        │
        ▼
  .decode("utf-8") ──► 原始明文字符串（写入临时 .decrypted.json，再导入 DB）
```

关键方法：
- [decrypt_string()](src/documents/management/commands/mixins.py#L135-L139)：将 hex 密文解密为明文字符串

---

## 四、数据进入与读取的完整协作链

### 4.1 场景一：API 写入（数据进入数据库）

```
用户请求 (明文密码/密钥)
    │
    ▼
ObfuscatedPasswordField.to_internal_value(data)
    │  return data  ──► 原样返回，不做任何变换
    ▼
Serializer.update() / run_validation()
    │  若提交值全为星号 → 跳过该字段（防止掩码覆盖原值）
    ▼
Django ORM save()
    │
    ▼
数据库 ──► 明文存储（User.password 除外，由 Django set_password() 哈希）
```

### 4.2 场景二：导出加密（数据进入导出包）

仅当传入 `--passphrase` 参数时执行加密分支：

```
1. handle() 解析参数
   └─> --passphrase <口令>
   └─> 调用 setup_crypto(passphrase=口令)
       ├─> 生成 16 字节随机盐
       └─> PBKDF2 派生 Fernet 密钥

2. dump() 遍历所有模型
   └─> 每条记录调用 _encrypt_record_inline(obj, model_name)
       ├─> if model_name not in CRYPT_FIELDS_BY_MODEL → 跳过
       ├─> 遍历 CRYPT_FIELDS_BY_MODEL[model_name] 字段列表
       └─> 对非空字段 value 执行 encrypt_string(value) 替换原值

3. 写入 metadata.json
   └─> __crypto__.__salt_hex__: 盐值 hex
   └─> __crypto__.__key_iters__: 1000000
   └─> __crypto__.__key_size__: 32
   └─> __crypto__.__key_algo__: "pbkdf2_sha256"
```

核心协作：
- [document_exporter.py:handle()](src/documents/management/commands/document_exporter.py#L302-L359)：参数解析与加密初始化
- [document_exporter.py:dump()](src/documents/management/commands/document_exporter.py#L360-L559)：主导出流程
- [document_exporter.py:_encrypt_record_inline()](src/documents/management/commands/document_exporter.py#L689-L699)：单条记录字段级加密

### 4.3 场景三：API 读取（数据从数据库返回前端）

```
数据库 (明文/哈希)
    │
    ▼
Django ORM 查询
    │
    ▼
ObfuscatedPasswordField.to_representation(value)
    │  return "*" * max(10, len(value))
    ▼
HTTP 响应 JSON ──► "**********"
```

### 4.4 场景四：导入解密（数据从导出包进入数据库）

```
1. load_metadata()
   ├─> 读取 metadata.json
   ├─> 存在 __crypto__ 但未传 --passphrase → CommandError 终止
   └─> 存在 __crypto__ 且有 --passphrase → load_crypt_params() 恢复 salt/iter 等

2. decrypt_secret_fields()
   ├─> setup_crypto(passphrase=口令, salt=恢复的盐值) → 重新派生密钥
   ├─> 遍历所有 manifest 文件
   ├─> 逐条记录调用 _decrypt_record_if_needed()
   │   ├─> model_name 不在 CRYPT_FIELDS_BY_MODEL → 跳过
   │   └─> decrypt_string() 逐个解密字段
   └─> 写入临时 <name>.decrypted.json

3. load_data_to_database()
   └─> 使用 .decrypted.json 反序列化
   └─> Django ORM save() → DB 明文存储

4. 清理：删除所有临时 .decrypted.json
```

核心协作：
- [document_importer.py:load_metadata()](src/documents/management/commands/document_importer.py#L282-L318)：加密元数据检测与加载
- [document_importer.py:decrypt_secret_fields()](src/documents/management/commands/document_importer.py#L666-L695)：解密并生成临时文件
- [document_importer.py:_decrypt_record_if_needed()](src/documents/management/commands/document_importer.py#L656-L664)：单条记录字段级解密

### 4.5 场景五：运行时使用（数据从 DB 读取到内存）

敏感字段在业务逻辑运行时以明文内存形态被直接使用：

```python
# src/paperless_mail/mail.py
account = MailAccount.objects.get(...)
mailbox.login(account.username, account.password)  # 明文直接传入 IMAP 库
```

```python
# src/paperless_ai/client.py
OpenAILike(api_key=self.settings.llm_api_key, ...)  # 明文直接传入 LLM SDK
```

证据位置：
- [src/paperless_mail/mail.py](src/paperless_mail/mail.py#L211-L237)：IMAP 登录读取明文密码
- [src/paperless_ai/client.py](src/paperless_ai/client.py#L50-L56)：LLM 调用读取明文 API Key

---

## 五、保护架构图（精确标注各边界状态）

```
┌──────────────────────────────────────────────────────────────────┐
│                      导出文件 (manifest.json)                     │
│  ┌──────────────────────────────────────────────────────────┐    │
│  │  MailAccount.password      : ✅ Fernet 密文 (hex)         │    │
│  │  MailAccount.refresh_token : ✅ Fernet 密文 (hex)         │    │
│  │  SocialToken.token         : ✅ Fernet 密文 (hex)         │    │
│  │  SocialToken.token_secret  : ✅ Fernet 密文 (hex)         │    │
│  │  ApplicationConfig.llm_api_key: ❌ 明文                  │    │
│  │  User.password             : 不在 CRYPT_FIELDS（PBKDF2）  │    │
│  │  metadata.json.__crypto__  : salt/iter/size/algo (明文)   │    │
│  └──────────────────────────────────────────────────────────┘    │
└──────────────────────────────────────────────────────────────────┘
                              ▲
                              │ document_exporter --passphrase
                              │ document_importer --passphrase
                              │
┌──────────────────────────────────────────────────────────────────┐
│                     数据库 (PostgreSQL / SQLite)                  │
│  ┌──────────────────────────────────────────────────────────┐    │
│  │  MailAccount.password      : ❌ 明文 TextField            │    │
│  │  MailAccount.refresh_token : ❌ 明文 TextField            │    │
│  │  SocialToken.token         : ❌ 明文 (第三方)              │    │
│  │  SocialToken.token_secret  : ❌ 明文 (第三方)              │    │
│  │  ApplicationConfig.llm_api_key: ❌ 明文 CharField         │    │
│  │  User.password             : PBKDF2 哈希 (Django 内建)    │    │
│  └──────────────────────────────────────────────────────────┘    │
└──────────────────────────────────────────────────────────────────┘
                              ▲ │
                              │ │
           API 写入(明文) ────┘ │────── ORM 读取(明文) ──► 业务逻辑运行时
                                │
┌──────────────────────────────────────────────────────────────────┐
│                      API 响应 (HTTP/JSON)                         │
│  ┌──────────────────────────────────────────────────────────┐    │
│  │  MailAccount.password      : ✅ "**********" (掩码)       │    │
│  │  MailAccount.refresh_token : ⚫ 不在序列化字段（不暴露）   │    │
│  │  SocialToken.*             : ⚫ 不在序列化字段（不暴露）   │    │
│  │  User.password             : ✅ "**********" (掩码)       │    │
│  │  ApplicationConfig.llm_api_key: ✅ "**********" (掩码)    │    │
│  └──────────────────────────────────────────────────────────┘    │
└──────────────────────────────────────────────────────────────────┘

图例: ✅ 有保护 / ❌ 无保护明文 / ⚫ 不在该边界暴露
```

---

## 六、安全边界总结

| 边界 | 机制 | 实际覆盖字段 |
|------|------|-------------|
| **API 响应** | `ObfuscatedPasswordField` 输出星号 + 提交时跳过掩码更新 | `MailAccount.password`, `User.password`, `Profile.password`, `ApplicationConfiguration.llm_api_key` |
| **跨系统导出** | Fernet (AES-128-CBC + HMAC-SHA256) + PBKDF2 密钥派生 | `MailAccount.password`, `MailAccount.refresh_token`, `SocialToken.token`, `SocialToken.token_secret` |
| **数据库静态** | 无应用层加密，仅依赖 OS 文件权限 + DB 访问控制 | 全部明文（`User.password` 由 Django PBKDF2 哈希） |
| **内存运行时** | 明文处理，直接传入 IMAP/LLM 等第三方库 | 所有敏感字段 |

### 已识别的保护缺口

| 缺口 | 精确描述 | 风险 |
|------|---------|------|
| `llm_api_key` 导出未加密 | `ApplicationConfiguration.llm_api_key` 使用了 `ObfuscatedPasswordField` 做 API 掩码，但未加入 `CryptMixin.CRYPT_FIELDS`，导致 `document_exporter --passphrase` 导出时该字段仍以**明文**写入 `manifest.json` | 导出文件泄露将暴露第三方大模型服务 API Key，攻击者可直接消费对应账户额度 |
| 数据库无字段级加密 | 所有邮件密码、刷新令牌、社交登录令牌、LLM API Key 均以明文落盘 | 数据库备份泄露、宿主机入侵、DBA 权限滥用均可直接获取全部凭据 |
| API 掩码仅防显示不防中间人 | `ObfuscatedPasswordField` 只影响响应体显示，HTTPS 传输层安全由部署环境负责 | 若部署未启用 HTTPS，掩码不提供任何传输层保护 |

---

## 七、关键设计决策澄清

以下针对之前容易误解的表述做精确澄清：

1. **"导出加密" ≠ "数据库加密"**：加密仅发生在导出流程中，加密后的密文只存在于导出包文件内。数据库本身始终是明文。

2. **"API 掩码" ≠ "数据加密"**：掩码只作用于 HTTP 响应序列化阶段，将显示替换为星号。数据库和内存中的值仍是原始明文。

3. **不存在"三层全覆盖"的字段**：`MailAccount.password` 虽然在 API 层有掩码、导出层有加密，但 DB 层是纯明文——三个边界中只有两个有防护，且防护范围完全不重叠。

4. **"DB 与导出受保护"表述不准确**：DB 层没有任何应用层保护；导出层仅在用户显式传入 `--passphrase` 参数时才启用加密，不传口令时导出文件同样是全明文。

5. **掩码复用不等于保护一致**：`ObfuscatedPasswordField` 跨模块复用只说明 API 显示层统一使用了星号策略，不代表这些字段在导出层也有同等保护（`llm_api_key` 即为反例）。

---

## 八、邮件内容 GPG 解密（独立链路补充）

除了上述字段级保护机制外，系统还提供邮件内容的 GPG 解密预处理器，面向邮件内容而非配置凭据，与字段加密是完全独立的链路：

- **模块**：`MailMessageDecryptor`
- **触发条件**：`EMAIL_ENABLE_GPG_DECRYPTOR=True` 且配置了 GPG 密钥目录
- **作用范围**：邮件消费流程中对 `multipart/encrypted` MIME 类型的邮件进行解密
- **位置**：[src/paperless_mail/preprocessor.py](src/paperless_mail/preprocessor.py#L37-L103)

---

## 附录：关键文件索引（仓库相对路径）

| 文件 | 作用 |
|------|------|
| `src/documents/management/commands/mixins.py` | `CryptMixin` 核心加密混入类，`CRYPT_FIELDS` 导出加密清单，加密/解密方法实现 |
| `src/documents/management/commands/document_exporter.py` | 导出命令入口，加密参数解析，单条记录内联加密调用 |
| `src/documents/management/commands/document_importer.py` | 导入命令入口，加密元数据检测，临时解密文件生成与清理 |
| `src/documents/settings.py` | 导出加密元数据键名常量（`__crypto__`、`__salt_hex__` 等） |
| `src/paperless_mail/serialisers.py` | `ObfuscatedPasswordField` 定义，`MailAccountSerializer`（仅 password 在 fields 中） |
| `src/paperless_mail/models.py` | `MailAccount` 模型，`password` 与 `refresh_token` 均为明文 TextField |
| `src/paperless/serialisers.py` | 跨模块复用 `ObfuscatedPasswordField`：User / Profile / AppConfig 序列化器，`PasswordValidationMixin`，`SocialAccountSerializer`（不暴露 SocialToken 字段） |
| `src/paperless/models.py` | `ApplicationConfiguration` 模型，`llm_api_key` 为明文 CharField |
| `src/paperless_mail/mail.py` | IMAP 登录逻辑，运行时直接读取 DB 明文密码 |
| `src/paperless_mail/preprocessor.py` | 邮件内容 GPG 解密预处理器（独立链路） |
| `src/paperless_ai/client.py` | LLM 客户端，运行时直接读取 DB 明文 llm_api_key |
