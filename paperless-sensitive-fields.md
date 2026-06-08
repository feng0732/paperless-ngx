# Paperless-ngx 敏感字段加密与数据保护机制分析

## 概述

Paperless-ngx 的敏感数据保护采用**分层保护策略**，而非数据库层面的透明加密（TDE）。保护机制分为两个主要层面：

1. **API 层面的混淆保护**：敏感字段在 API 序列化/反序列化过程中进行掩码处理
2. **导入/导出层面的加密保护**：数据跨系统迁移时使用对称加密保护敏感字段

**重要说明**：数据库内部存储的敏感字段本身为明文，加密保护仅作用于导出文件。

---

## 一、三份保护清单：导出加密 vs API 掩码 vs 数据库明文

这是理解系统敏感数据边界的核心。三份清单**并不完全重合**，体现了不同层面保护的差异化设计。

### 1.1 导出加密清单（CRYPT_FIELDS）

仅在 `document_exporter --passphrase` 导出时生效，保护跨系统迁移的数据。配置集中定义在 `CryptMixin.CRYPT_FIELDS`：

| 模型标识 | 导出键名 | 加密字段 | 字段类型 | 来源 |
|----------|----------|----------|----------|------|
| `paperless_mail.mailaccount` | `mail_accounts` | `password`, `refresh_token` | TextField | 项目内模型 |
| `socialaccount.socialtoken` | `social_tokens` | `token`, `token_secret` | — | django-allauth 第三方模型 |

配置位置：[src/documents/management/commands/mixins.py](src/documents/management/commands/mixins.py#L54-L75)

### 1.2 API 掩码清单（ObfuscatedPasswordField 使用处）

在 REST API 响应层生效，阻止前端读取明文。`ObfuscatedPasswordField` 定义于邮件模块，但被**跨模块复用**：

| 序列化器 | 掩码字段 | 所属模块 | 定义位置 |
|----------|----------|----------|----------|
| `MailAccountSerializer` | `password` | paperless_mail | [src/paperless_mail/serialisers.py](src/paperless_mail/serialisers.py#L28-L58) |
| `UserSerializer` | `password` | paperless | [src/paperless/serialisers.py](src/paperless/serialisers.py#L77-L142) |
| `ProfileSerializer` | `password` | paperless | [src/paperless/serialisers.py](src/paperless/serialisers.py#L179-L209) |
| `ApplicationConfigurationSerializer` | `llm_api_key` | paperless | [src/paperless/serialisers.py](src/paperless/serialisers.py#L212-L296) |

**API 掩码复用路径**：`ObfuscatedPasswordField` 定义在 `paperless_mail` 模块，但通过 `from paperless_mail.serialisers import ObfuscatedPasswordField` 被 `paperless` 模块跨包引用，实现了一处定义、多处复用。

引用位置：[src/paperless/serialisers.py](src/paperless/serialisers.py#L25)

### 1.3 数据库明文边界（DB 中实际存储形态）

数据库中的真实存储形态。注意：Django 内建用户密码走独立哈希链路，不参与 ObfuscatedPasswordField 的掩码更新逻辑。

| 模型 | 字段 | DB 存储形态 | 模型定义 | 备注 |
|------|------|-------------|----------|------|
| `MailAccount` | `password` | 明文 TextField | [src/paperless_mail/models.py](src/paperless_mail/models.py#L45) | 邮件 IMAP 密码或 OAuth 访问令牌 |
| `MailAccount` | `refresh_token` | 明文 TextField | [src/paperless_mail/models.py](src/paperless_mail/models.py#L65-L72) | OAuth 刷新令牌 |
| `SocialToken` | `token` | 明文 | django-allauth 第三方 | 第三方登录访问令牌 |
| `SocialToken` | `token_secret` | 明文 | django-allauth 第三方 | OAuth1 令牌密钥 |
| `ApplicationConfiguration` | `llm_api_key` | 明文 CharField | [src/paperless/models.py](src/paperless/models.py#L328-L333) | LLM 服务 API Key |
| `User`（Django 内建） | `password` | PBKDF2 哈希 | Django contrib.auth | **不参与导出加密清单**，由 Django 自行哈希 |

### 1.4 三份清单差异对照表

这是理解保护边界的关键——并非所有敏感字段都在三个层面同时受保护：

| 字段 | 导出加密 | API 掩码 | DB 明文 | 说明 |
|------|:--------:|:--------:|:-------:|------|
| `MailAccount.password` | ✅ | ✅ | ✅ 明文 | 三层全覆盖 |
| `MailAccount.refresh_token` | ✅ | ❌ | ✅ 明文 | DB 与导出受保护，API 未暴露该字段 |
| `SocialToken.token` | ✅ | ❌ | ✅ 明文 | SocialAccountSerializer 不暴露 token 字段 |
| `SocialToken.token_secret` | ✅ | ❌ | ✅ 明文 | 同上 |
| `ApplicationConfiguration.llm_api_key` | ❌ | ✅ | ✅ 明文 | **缺口**：API 掩码但未纳入导出加密 |
| `User.password` | ❌ | ✅ | ❌ 哈希 | Django 自行 PBKDF2 哈希，不参与导出加密 |

**保护缺口说明**：

1. `llm_api_key` 仅做了 API 掩码，未加入 `CRYPT_FIELDS`。使用 `--passphrase` 导出时该字段仍以明文写入 `manifest.json`。
2. `User.password` 本身由 Django 使用 PBKDF2 哈希存储，不暴露明文，故无需额外导出加密。但序列化层仍复用了 `ObfuscatedPasswordField` 做显示掩码。

---

## 二、核心加密实现：CryptMixin

### 2.1 加密架构

`CryptMixin` 是导入导出命令的共享混入类，基于 Python `cryptography` 库实现：

- **密钥派生算法**：PBKDF2-HMAC-SHA256
- **对称加密算法**：Fernet（内部封装 AES-128-CBC + HMAC-SHA256）
- **盐值长度**：16 字节（加密安全随机数）
- **迭代次数**：1,000,000 次（与 Django 默认密码哈希迭代一致）
- **密钥长度**：32 字节

核心定义位置：[src/documents/management/commands/mixins.py](src/documents/management/commands/mixins.py#L23-L139)

### 2.2 加密流程（数据进入导出文件）

```
用户口令 (passphrase)
        │
        ▼
  os.urandom(16) ──► salt (hex)
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
  .hex() ──► 加密后的十六进制字符串
```

关键方法：
- [setup_crypto()](src/documents/management/commands/mixins.py#L102-L126)：初始化加密上下文
- [encrypt_string()](src/documents/management/commands/mixins.py#L128-L133)：加密单个字符串

### 2.3 解密流程（从导出文件读取）

```
加密后的十六进制字符串
        │
        ▼
  bytes.fromhex() ──► Fernet token 字节
        │
        ▼
  Fernet.decrypt()
        │
        ▼
  .decode("utf-8") ──► 原始明文字符串
```

关键方法：
- [decrypt_string()](src/documents/management/commands/mixins.py#L135-L139)：解密单个字符串

---

## 三、数据进入：写入加密的完整协作链

### 3.1 场景一：API 写入敏感字段（数据库存储）

用户通过 REST API 创建/更新邮件账户时：

```
用户请求 (明文密码)
    │
    ▼
[ObfuscatedPasswordField.to_internal_value()]
    │  直接返回原始值，不做加密处理
    ▼
Django ORM 保存
    │
    ▼
数据库 ──► 明文存储
```

关键代码：
- [ObfuscatedPasswordField](src/paperless_mail/serialisers.py#L16-L25)：API 层序列化字段
- [MailAccountSerializer.update()](src/paperless_mail/serialisers.py#L51-L58)：处理全星号掩码时跳过更新

### 3.2 场景二：导出时加密（跨系统保护）

执行 `document_exporter --passphrase` 命令时：

```
1. 参数解析阶段
   └─> 解析 --passphrase 参数
   └─> setup_crypto(passphrase=用户口令)
       └─> 生成随机 salt
       └─> PBKDF2 派生 Fernet 密钥

2. 数据序列化阶段 (dump())
   └─> 遍历所有模型记录
   └─> 对每条记录调用 _encrypt_record_inline()
       ├─> 检查 model 是否在 CRYPT_FIELDS_BY_MODEL 中
       ├─> 若是，遍历该模型的敏感字段列表
       └─> 对每个非空字段执行 encrypt_string() 替换原值

3. 元数据写入阶段
   └─> 将加密参数写入 metadata.json:
       ├─> __crypto__.__salt_hex__: 盐值 (hex)
       ├─> __crypto__.__key_iters__: 迭代次数
       ├─> __crypto__.__key_size__: 密钥长度
       └─> __crypto__.__key_algo__: 算法名称 (pbkdf2_sha256)
```

核心协作：
- [document_exporter.py:handle()](src/documents/management/commands/document_exporter.py#L302-L359)：命令入口与参数解析
- [document_exporter.py:dump()](src/documents/management/commands/document_exporter.py#L360-L559)：主导出流程
- [document_exporter.py:_encrypt_record_inline()](src/documents/management/commands/document_exporter.py#L689-L699)：单条记录加密处理

### 3.3 邮件运行时密码使用

邮件任务（`mail_fetch`）运行时直接从数据库读取明文密码进行 IMAP 登录：

```
MailAccount.objects.get(pk=...)
    │
    ├─> account.password (明文)
    └─> mailbox.login() / mailbox.xoauth2()
```

位置：[src/paperless_mail/mail.py](src/paperless_mail/mail.py#L211-L237)

---

## 四、数据存储：保护机制分层

### 4.1 存储分层概览

```
┌─────────────────────────────────────────────────────┐
│                导出文件 (manifest.json)             │
│  ┌─────────────────────────────────────────────┐    │
│  │  MailAccount.password      : Fernet 密文    │    │
│  │  MailAccount.refresh_token : Fernet 密文    │    │
│  │  SocialToken.token         : Fernet 密文    │    │
│  │  SocialToken.token_secret  : Fernet 密文    │    │
│  │  ApplicationConfig.llm_api_key: 明文 ❗     │    │
│  │  元数据: salt/迭代次数/算法 (明文)           │    │
│  └─────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────┐
│                  数据库 (PostgreSQL/SQLite)         │
│  ┌─────────────────────────────────────────────┐    │
│  │  MailAccount.password      : 明文            │    │
│  │  MailAccount.refresh_token : 明文            │    │
│  │  SocialToken.token         : 明文            │    │
│  │  SocialToken.token_secret  : 明文            │    │
│  │  ApplicationConfig.llm_api_key: 明文         │    │
│  │  User.password             : PBKDF2 哈希     │    │
│  └─────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────┐
│                  API 响应 (HTTP/JSON)               │
│  ┌─────────────────────────────────────────────┐    │
│  │  MailAccount.password      : **********     │    │
│  │  User.password             : **********     │    │
│  │  ApplicationConfig.llm_api_key: **********  │    │
│  │  MailAccount.refresh_token : 不暴露          │    │
│  │  SocialToken.*             : 不暴露          │    │
│  └─────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────┘
```

### 4.2 API 层掩码复用机制

`ObfuscatedPasswordField` 是 API 掩码保护的核心，设计为一处定义跨模块复用：

**定义处**：[src/paperless_mail/serialisers.py](src/paperless_mail/serialisers.py#L16-L25)

**复用路径**：
```
src/paperless_mail/serialisers.py (定义)
        │
        ▼
src/paperless/serialisers.py (import)
        │
        ├─> UserSerializer.password
        ├─> ProfileSerializer.password
        └─> ApplicationConfigurationSerializer.llm_api_key
```

掩码行为细节：
- **输出（序列化 `to_representation`）**：返回 `max(10, len(值))` 个星号，避免泄露密码长度（最小 10 位）
- **输入（反序列化 `to_internal_value`）**：直接返回用户输入，不做变换
- **更新保护（各 Serializer.update / run_validation）**：
  - `MailAccountSerializer`：若提交值全由星号组成 → `pop("password")` 跳过更新
  - `UserSerializer` / `ProfileSerializer`：通过 `PasswordValidationMixin._has_real_password()` 检测，仅含星号时不调用 `set_password()`
  - `ApplicationConfigurationSerializer`：在 `run_validation()` 中 `del data["llm_api_key"]` 跳过

相关代码：
- [PasswordValidationMixin](src/paperless/serialisers.py#L30-L44)：用户密码校验复用
- [ApplicationConfigurationSerializer.run_validation()](src/paperless/serialisers.py#L222-L235)：LLM API Key 掩码跳过逻辑

### 4.3 加密参数存储格式

导出的 `metadata.json` 中加密相关配置键名定义：

| 配置键 | 含义 | 定义位置 |
|--------|------|----------|
| `__crypto__` | 加密设置顶层命名空间 | [src/documents/settings.py](src/documents/settings.py#L8) |
| `__salt_hex__` | 盐值（十六进制） | [src/documents/settings.py](src/documents/settings.py#L9) |
| `__key_iters__` | PBKDF2 迭代次数 | [src/documents/settings.py](src/documents/settings.py#L10) |
| `__key_size__` | 派生密钥长度 | [src/documents/settings.py](src/documents/settings.py#L11) |
| `__key_algo__` | KDF 算法标识 | [src/documents/settings.py](src/documents/settings.py#L12) |

---

## 五、数据读取：解密协作链

### 5.1 场景一：API 读取

API 读取始终返回掩码，不提供解密接口：

```
数据库 (明文/哈希)
    │
    ▼
Django ORM 查询
    │
    ▼
ObfuscatedPasswordField.to_representation()
    │
    ▼
HTTP 响应: "**********"
```

### 5.2 场景二：导入时解密

执行 `document_importer --passphrase` 命令时：

```
1. 元数据加载阶段 (load_metadata())
   ├─> 读取 metadata.json
   ├─> 检查是否存在 __crypto__ 键
   │   ├─> 存在但未提供 passphrase ──► CommandError
   │   └─> 存在且提供 passphrase ──► load_crypt_params()
   └─> 恢复加密参数: salt, key_iterations, key_size, kdf_algorithm

2. 解密处理阶段 (decrypt_secret_fields())
   ├─> setup_crypto(passphrase=用户口令, salt=恢复的盐值)
   │   └─> 使用相同参数重新派生 Fernet 密钥
   ├─> 遍历所有 manifest 文件
   ├─> 逐条记录调用 _decrypt_record_if_needed()
   │   ├─> 检查 model 是否在 CRYPT_FIELDS_BY_MODEL
   │   └─> 对敏感字段执行 decrypt_string()
   └─> 将解密后的内容写入临时文件 .decrypted.json

3. 数据入库阶段 (load_data_to_database())
   └─> 使用解密后的临时 manifest 文件
   └─> 反序列化写入数据库（明文存储）

4. 清理阶段
   └─> 删除所有临时 .decrypted.json 文件
```

核心协作：
- [document_importer.py:load_metadata()](src/documents/management/commands/document_importer.py#L282-L318)：加载并校验加密元数据
- [document_importer.py:decrypt_secret_fields()](src/documents/management/commands/document_importer.py#L666-L695)：解密并生成临时文件
- [document_importer.py:_decrypt_record_if_needed()](src/documents/management/commands/document_importer.py#L656-L664)：单条记录解密

---

## 六、模块协作全景图

```
                    ┌──────────────────────┐
                    │  用户/管理员操作     │
                    └──────────┬───────────┘
                               │
           ┌───────────────────┼───────────────────┐
           ▼                   ▼                   ▼
  ┌────────────────┐  ┌────────────────────┐  ┌────────────────────┐
  │  REST API      │  │ document_exporter  │  │ document_importer  │
  │  写入/读取     │  │  (导出加密)        │  │  (导入解密)        │
  └────────┬───────┘  └─────────┬──────────┘  └─────────┬──────────┘
           │                    │                       │
           │                    ▼                       ▼
           │          ┌─────────────────────────────────┐
           │          │        CryptMixin               │
           │          │  setup_crypto() / encrypt_*()   │
           │          │  decrypt_*() / CRYPT_FIELDS     │
           │          └─────────────────────────────────┘
           │                    │
           ▼                    ▼
  ┌────────────────────────┐  ┌───────────────────────────────┐
  │  ObfuscatedPasswordField│  │  Fernet (cryptography 库)    │
  │  - to_representation()  │  │  PBKDF2-SHA256 + AES-CBC+HMAC│
  │  - to_internal_value()  │  └───────────────────────────────┘
  └────────────┬───────────┘
               │  跨模块复用
               ├──────────────────────────────────────┐
               │                                      │
   ┌─────────────────────────────┐      ┌────────────────────────────┐
   │ MailAccountSerializer       │      │ paperless 模块:            │
   │  .password                  │      │  UserSerializer.password   │
   └─────────────────────────────┘      │  ProfileSerializer.password│
                                        │  AppConfigSerializer       │
                                        │    .llm_api_key            │
                                        └────────────────────────────┘
               │
               ▼
  ┌─────────────────────────────────┐
  │         数据库                  │
  │  MailAccount.password (明文)    │
  │  MailAccount.refresh_token     │
  │  SocialToken.token/secret      │
  │  ApplicationConfig.llm_api_key │
  │  User.password (PBKDF2 哈希)   │
  └─────────────────────────────────┘
```

---

## 七、安全边界总结

| 保护层面 | 机制 | 强度 | 覆盖字段 |
|----------|------|------|----------|
| **API 传输** | HTTPS + `ObfuscatedPasswordField` 响应掩码 | 中 | `MailAccount.password`, `User.password`, `ApplicationConfiguration.llm_api_key` |
| **跨系统迁移** | Fernet 对称加密 (AES-128-CBC + HMAC) | 高 | `MailAccount.password`, `MailAccount.refresh_token`, `SocialToken.token`, `SocialToken.token_secret` |
| **数据库静态** | 无应用层加密（依赖 OS/DB 访问控制） | 低 | 全部明文（User.password 由 Django PBKDF2 哈希） |
| **内存运行时** | 明文处理 | - | IMAP 登录、LLM 调用、令牌刷新等 |

### 关键设计决策

1. **加密仅用于导出**：避免了数据库级加密的复杂度与性能开销，专注于跨系统数据迁移的机密性。
2. **盐值独立存储**：盐值与加密数据分离存储在 `metadata.json` 中，同一导出使用统一盐值。
3. **密码派生而非存储**：始终通过用户口令派生密钥，不在系统任何位置持久化密钥。
4. **API 掩码一处定义、跨模块复用**：`ObfuscatedPasswordField` 在 `paperless_mail` 定义，被 `paperless` 模块的用户、应用配置序列化器引用。
5. **掩码更新保护差异化实现**：不同序列化器根据字段语义实现了不同的掩码跳过逻辑（`MailAccount` 直接 pop、`User` 走 `PasswordValidationMixin`、`AppConfig` 在 `run_validation` 删除）。
6. **失败安全**：导入时若检测到加密元数据但未提供口令，立即报错终止，避免产生半解密状态。

### 已识别的保护缺口

| 缺口 | 影响 | 风险 |
|------|------|------|
| `ApplicationConfiguration.llm_api_key` 未加入 `CRYPT_FIELDS` | 使用 `--passphrase` 导出时，LLM API Key 仍以明文写入导出包 | 导出文件泄露将暴露第三方大模型服务凭据 |
| 数据库层无字段加密 | 所有邮件密码、刷新令牌、API Key 均明文落盘 | 数据库备份泄露或宿主机被入侵将直接暴露全部凭据 |

---

## 八、邮件内容 GPG 解密（补充）

除了上述字段级加密外，系统还提供邮件内容的 GPG 解密预处理器：

- **模块**：`MailMessageDecryptor`
- **触发条件**：`EMAIL_ENABLE_GPG_DECRYPTOR=True` 且配置了 GPG 密钥目录
- **作用范围**：仅在邮件消费流程中对 `multipart/encrypted` 类型邮件进行解密
- **位置**：[src/paperless_mail/preprocessor.py](src/paperless_mail/preprocessor.py#L37-L103)

这是与字段加密独立的另一条保护路径，面向邮件内容而非配置凭据。

---

## 附录：关键文件索引（仓库相对路径）

| 文件 | 作用 |
|------|------|
| `src/documents/management/commands/mixins.py` | `CryptMixin` 核心加密混入类，`CRYPT_FIELDS` 加密清单 |
| `src/documents/management/commands/document_exporter.py` | 导出命令，调用加密逻辑 |
| `src/documents/management/commands/document_importer.py` | 导入命令，调用解密逻辑 |
| `src/documents/settings.py` | 导出加密元数据键名常量 |
| `src/paperless_mail/serialisers.py` | `ObfuscatedPasswordField` 定义，`MailAccountSerializer` |
| `src/paperless_mail/models.py` | `MailAccount` 模型（password / refresh_token 字段） |
| `src/paperless/serialisers.py` | 复用 `ObfuscatedPasswordField`：User / Profile / AppConfig 序列化器 |
| `src/paperless/models.py` | `ApplicationConfiguration` 模型（llm_api_key 字段） |
| `src/paperless_mail/mail.py` | IMAP 登录逻辑，运行时读取明文密码 |
| `src/paperless_mail/preprocessor.py` | 邮件内容 GPG 解密预处理器 |
