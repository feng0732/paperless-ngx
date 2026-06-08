# Paperless-ngx 敏感字段加密与数据保护机制分析

## 概述

Paperless-ngx 的敏感数据保护采用**分层保护策略**，而非数据库层面的透明加密（TDE）。保护机制分为两个主要层面：

1. **API 层面的混淆保护**：敏感字段在 API 序列化/反序列化过程中进行掩码处理
2. **导入/导出层面的加密保护**：数据跨系统迁移时使用对称加密保护敏感字段

**重要说明**：数据库内部存储的敏感字段本身为明文，加密保护仅作用于导出文件。

---

## 一、受保护的敏感字段清单

加密保护通过 `CryptMixin.CRYPT_FIELDS` 配置集中定义：

| 模型 | 导出键名 | 敏感字段 | 说明 |
|------|----------|----------|------|
| `paperless_mail.mailaccount` | `mail_accounts` | `password`, `refresh_token` | 邮件账户密码与 OAuth 刷新令牌 |
| `socialaccount.socialtoken` | `social_tokens` | `token`, `token_secret` | 第三方社交登录令牌与密钥 |

配置位置：[mixins.py](file:///d:/fz/0601/solo-dogfeeding/code/116-paperless-ngx/src/documents/management/commands/mixins.py#L54-L75)

---

## 二、核心加密实现：CryptMixin

### 2.1 加密架构

`CryptMixin` 是导入导出命令的共享混入类，基于 Python `cryptography` 库实现：

- **密钥派生算法**：PBKDF2-HMAC-SHA256
- **对称加密算法**：Fernet（内部封装 AES-128-CBC + HMAC-SHA256）
- **盐值长度**：16 字节（加密安全随机数）
- **迭代次数**：1,000,000 次（与 Django 默认密码哈希迭代一致）
- **密钥长度**：32 字节

核心定义位置：[mixins.py](file:///d:/fz/0601/solo-dogfeeding/code/116-paperless-ngx/src/documents/management/commands/mixins.py#L23-L139)

### 2.2 加密流程（数据进入）

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
- [setup_crypto()](file:///d:/fz/0601/solo-dogfeeding/code/116-paperless-ngx/src/documents/management/commands/mixins.py#L102-L126)：初始化加密上下文
- [encrypt_string()](file:///d:/fz/0601/solo-dogfeeding/code/116-paperless-ngx/src/documents/management/commands/mixins.py#L128-L133)：加密单个字符串

### 2.3 解密流程（数据读取）

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
- [decrypt_string()](file:///d:/fz/0601/solo-dogfeeding/code/116-paperless-ngx/src/documents/management/commands/mixins.py#L135-L139)：解密单个字符串

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
- [ObfuscatedPasswordField](file:///d:/fz/0601/solo-dogfeeding/code/116-paperless-ngx/src/paperless_mail/serialisers.py#L16-L25)：API 层序列化字段
- [MailAccountSerializer.update()](file:///d:/fz/0601/solo-dogfeeding/code/116-paperless-ngx/src/paperless_mail/serialisers.py#L51-L58)：处理全星号掩码时跳过更新

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
- [document_exporter.py:handle()](file:///d:/fz/0601/solo-dogfeeding/code/116-paperless-ngx/src/documents/management/commands/document_exporter.py#L302-L359)：命令入口与参数解析
- [document_exporter.py:dump()](file:///d:/fz/0601/solo-dogfeeding/code/116-paperless-ngx/src/documents/management/commands/document_exporter.py#L360-L559)：主导出流程
- [document_exporter.py:_encrypt_record_inline()](file:///d:/fz/0601/solo-dogfeeding/code/116-paperless-ngx/src/documents/management/commands/document_exporter.py#L689-L699)：单条记录加密处理

### 3.3 邮件运行时密码使用

邮件任务（`mail_fetch`）运行时直接从数据库读取明文密码进行 IMAP 登录：

```
MailAccount.objects.get(pk=...)
    │
    ├─> account.password (明文)
    └─> mailbox.login() / mailbox.xoauth2()
```

位置：[mail.py:mailbox_login()](file:///d:/fz/0601/solo-dogfeeding/code/116-paperless-ngx/src/paperless_mail/mail.py#L211-L237)

---

## 四、数据存储：保护机制分层

### 4.1 存储分层概览

```
┌─────────────────────────────────────────────────────┐
│                导出文件 (manifest.json)             │
│  ┌─────────────────────────────────────────────┐    │
│  │  敏感字段: Fernet 加密 + hex 编码            │    │
│  │  元数据: salt/迭代次数/算法 (明文)           │    │
│  └─────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────┐
│                  数据库 (PostgreSQL/SQLite)         │
│  ┌─────────────────────────────────────────────┐    │
│  │  敏感字段: 明文存储                          │    │
│  │  依赖: 文件系统权限 + 数据库访问控制         │    │
│  └─────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────┐
│                  API 响应 (HTTP/JSON)               │
│  ┌─────────────────────────────────────────────┐    │
│  │  敏感字段: 星号掩码 (**********)             │    │
│  │  实现: ObfuscatedPasswordField              │    │
│  └─────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────┘
```

### 4.2 API 层掩码保护

`ObfuscatedPasswordField` 确保 API 响应中不会泄露明文：

- **输出（序列化）**：`to_representation()` 返回 `max(10, len(值))` 个星号
- **输入（反序列化）**：`to_internal_value()` 直接返回用户输入
- **更新保护**：若提交值全为星号（即前端回显的掩码），则跳过该字段更新

位置：[serialisers.py](file:///d:/fz/0601/solo-dogfeeding/code/116-paperless-ngx/src/paperless_mail/serialisers.py#L16-L58)

### 4.3 加密参数存储格式

导出的 `metadata.json` 中加密相关配置键名定义：

| 配置键 | 含义 | 定义位置 |
|--------|------|----------|
| `__crypto__` | 加密设置顶层命名空间 | [settings.py](file:///d:/fz/0601/solo-dogfeeding/code/116-paperless-ngx/src/documents/settings.py#L8) |
| `__salt_hex__` | 盐值（十六进制） | [settings.py](file:///d:/fz/0601/solo-dogfeeding/code/116-paperless-ngx/src/documents/settings.py#L9) |
| `__key_iters__` | PBKDF2 迭代次数 | [settings.py](file:///d:/fz/0601/solo-dogfeeding/code/116-paperless-ngx/src/documents/settings.py#L10) |
| `__key_size__` | 派生密钥长度 | [settings.py](file:///d:/fz/0601/solo-dogfeeding/code/116-paperless-ngx/src/documents/settings.py#L11) |
| `__key_algo__` | KDF 算法标识 | [settings.py](file:///d:/fz/0601/solo-dogfeeding/code/116-paperless-ngx/src/documents/settings.py#L12) |

---

## 五、数据读取：解密协作链

### 5.1 场景一：API 读取

API 读取始终返回掩码，不提供解密接口：

```
数据库 (明文)
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
- [document_importer.py:load_metadata()](file:///d:/fz/0601/solo-dogfeeding/code/116-paperless-ngx/src/documents/management/commands/document_importer.py#L282-L318)：加载并校验加密元数据
- [document_importer.py:decrypt_secret_fields()](file:///d:/fz/0601/solo-dogfeeding/code/116-paperless-ngx/src/documents/management/commands/document_importer.py#L666-L695)：解密并生成临时文件
- [document_importer.py:_decrypt_record_if_needed()](file:///d:/fz/0601/solo-dogfeeding/code/116-paperless-ngx/src/documents/management/commands/document_importer.py#L656-L664)：单条记录解密

---

## 六、模块协作全景图

```
                    ┌──────────────────────┐
                    │  用户/管理员操作     │
                    └──────────┬───────────┘
                               │
           ┌───────────────────┼───────────────────┐
           ▼                   ▼                   ▼
  ┌────────────────┐  ┌────────────────┐  ┌────────────────┐
  │  REST API      │  │ document_exporter │  │ document_importer │
  │  写入/读取     │  │  (导出加密)     │  │  (导入解密)     │
  └────────┬───────┘  └────────┬───────┘  └────────┬───────┘
           │                   │                   │
           │                   ▼                   ▼
           │          ┌─────────────────────────────────┐
           │          │        CryptMixin               │
           │          │  setup_crypto() / encrypt_*()   │
           │          │  decrypt_*() / CRYPT_FIELDS     │
           │          └─────────────────────────────────┘
           │                   │
           ▼                   ▼
  ┌────────────────┐  ┌─────────────────────────────────┐
  │ Obfuscated-    │  │  Fernet (cryptography 库)        │
  │ PasswordField  │  │  PBKDF2-SHA256 + AES-CBC+HMAC   │
  └────────┬───────┘  └─────────────────────────────────┘
           │
           ▼
  ┌─────────────────────────────────┐
  │         数据库 (明文)           │
  │  MailAccount.password           │
  │  MailAccount.refresh_token      │
  │  SocialToken.token / token_secret│
  └─────────────────────────────────┘
```

---

## 七、安全边界总结

| 保护层面 | 机制 | 强度 | 覆盖范围 |
|----------|------|------|----------|
| **API 传输** | HTTPS + 响应掩码 | 中 | 所有敏感字段 API 响应 |
| **跨系统迁移** | Fernet 对称加密 (AES-128-CBC) | 高 | 导出文件中的敏感字段 |
| **数据库静态** | 无加密（依赖 OS/DB 权限） | 低 | - |
| **内存运行时** | 明文处理 | - | IMAP 登录、令牌刷新等 |

### 关键设计决策

1. **加密仅用于导出**：避免了数据库级加密的复杂度与性能开销，专注于跨系统数据迁移的机密性。
2. **盐值独立存储**：盐值与加密数据分离存储在 `metadata.json` 中，同一导出使用统一盐值。
3. **密码派生而非存储**：始终通过用户口令派生密钥，不在系统任何位置持久化密钥。
4. **API 掩码防御**：前端永远无法读取明文密码，提交掩码时不会误覆盖数据库值。
5. **失败安全**：导入时若检测到加密元数据但未提供口令，立即报错终止，避免产生半解密状态。

---

## 八、邮件内容 GPG 解密（补充）

除了上述字段级加密外，系统还提供邮件内容的 GPG 解密预处理器：

- **模块**：`MailMessageDecryptor`
- **触发条件**：`EMAIL_ENABLE_GPG_DECRYPTOR=True` 且配置了 GPG 密钥目录
- **作用范围**：仅在邮件消费流程中对 `multipart/encrypted` 类型邮件进行解密
- **位置**：[preprocessor.py](file:///d:/fz/0601/solo-dogfeeding/code/116-paperless-ngx/src/paperless_mail/preprocessor.py#L37-L103)

这是与字段加密独立的另一条保护路径，面向邮件内容而非配置凭据。
