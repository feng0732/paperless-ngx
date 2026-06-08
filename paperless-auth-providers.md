# Paperless-ngx 认证系统代码分析

本文档梳理 paperless-ngx 的双因素认证（2FA/MFA）与社交登录的完整代码处理路径，涵盖认证入口、凭据校验和账号绑定三大核心流程。

---

## 一、整体架构概览

paperless-ngx 基于 **django-allauth** 框架实现认证体系，核心模块关系如下：

```
┌───────────────────────────────────────────────────────────────┐
│                        前端 (Angular)                          │
│  ┌────────────────────┐  ┌──────────────────────────────────┐ │
│  │ ProfileEditDialog  │  │ ProfileService / UserService     │ │
│  └─────────┬──────────┘  └──────────────┬───────────────────┘ │
└────────────┼────────────────────────────┼─────────────────────┘
             │ HTTP/REST API               │ Session/Token
┌────────────▼────────────────────────────▼─────────────────────┐
│                     后端 (Django + DRF)                        │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │                    URL 路由层                             │  │
│  │  /accounts/*    /api/auth/*    /api/profile/*           │  │
│  └───────────────────────────┬─────────────────────────────┘  │
│                              │                                │
│  ┌───────────────────────────▼─────────────────────────────┐  │
│  │                   allauth 视图层                          │  │
│  │  allauth.account.views  allauth.mfa.views               │  │
│  │  allauth.socialaccount.views  allauth.headless          │  │
│  └───────────────────────────┬─────────────────────────────┘  │
│                              │                                │
│  ┌───────────────────────────▼─────────────────────────────┐  │
│  │               paperless 自定义适配器                      │  │
│  │  CustomAccountAdapter  CustomSocialAccountAdapter       │  │
│  │  DrfTokenStrategy                                       │  │
│  └───────────────────────────┬─────────────────────────────┘  │
│                              │                                │
│  ┌───────────────────────────▼─────────────────────────────┐  │
│  │                   认证后端 (AUTHENTICATION_BACKENDS)      │  │
│  │  1. RemoteUserBackend (可选, SSO)                       │  │
│  │  2. ObjectPermissionBackend (guardian)                  │  │
│  │  3. ModelBackend (Django 默认)                           │  │
│  │  4. AuthenticationBackend (allauth)                     │  │
│  └─────────────────────────────────────────────────────────┘  │
└───────────────────────────────────────────────────────────────┘
```

---

## 二、认证入口：URL 路由与视图

### 2.1 核心路由配置

所有认证相关 URL 均定义在 [urls.py](file:///d:/fz/0601/solo-dogfeeding/code/113-paperless-ngx/src/paperless/urls.py)。

#### 2.1.1 常规账号认证入口

```python
# /accounts/login/  - 登录页面 (GET/POST)
path("login/", allauth_account_views.login, name="account_login"),
# /accounts/logout/ - 登出
path("logout/", allauth_account_views.logout, name="account_logout"),
# /accounts/signup/ - 注册
path("signup/", allauth_account_views.signup, name="account_signup"),
```

同时提供 DRF API 风格的同一路由（共享同一个视图函数）：

```python
# /api/auth/login/  和 /api/auth/logout/
path("login/", allauth_account_views.login, name="login"),
path("logout/", allauth_account_views.logout, name="logout"),
```

**验证测试**：见 [test_api_auth.py](file:///d:/fz/0601/solo-dogfeeding/code/113-paperless-ngx/src/paperless/tests/test_api_auth.py#L12-L23) 中的 `test_api_auth_login_uses_same_view_as_account_login`，确认两个路径使用完全相同的视图类。

#### 2.1.2 双因素认证入口

```python
# /accounts/2fa/authenticate/ - MFA 二次验证页面
path("2fa/authenticate/", allauth_mfa_views.authenticate, name="mfa_authenticate"),
```

#### 2.1.3 社交账号入口

```python
# 通用状态页面
# /accounts/3rdparty/login/cancelled/  - 社交登录取消
# /accounts/3rdparty/login/error/      - 社交登录错误
# /accounts/3rdparty/signup/           - 社交登录后补全注册（若 AUTO_SIGNUP=False）

# 各 OAuth 提供商的独立 URL（由 allauth 的 build_provider_urlpatterns() 动态生成）
# 格式为：/accounts/<provider_id>/login/           → 发起 OAuth 授权（登录或绑定）
#          /accounts/<provider_id>/login/callback/  → OAuth 回调地址
# 例如：/accounts/keycloak-test/login/
#       /accounts/keycloak-test/login/callback/
```

**`build_provider_urlpatterns()` 的作用**：allauth 遍历 `SOCIALACCOUNT_PROVIDERS` 中配置的每个提供商，为其动态注册专属的登录发起和回调 URL。`provider_id` 来自 SocialApp 配置（如 OpenID Connect 的 `provider_id` 字段）。

#### 2.1.4 Headless API 入口

```python
# /api/auth/headless/  - allauth 无头模式（供 SPA 前端调用的 JSON API）
re_path("^auth/headless/", include("allauth.headless.urls")),
```

#### 2.1.5 Profile API 入口（账号管理）

```python
# GET/PATCH /api/profile/                           - 获取/更新用户资料
# POST      /api/profile/generate_auth_token/       - 生成 API Token
# POST      /api/profile/disconnect_social_account/ - 解绑社交账号
# GET       /api/profile/social_account_providers/  - 获取可用社交登录提供商
# GET/POST/DELETE /api/profile/totp/                - TOTP 密钥获取/激活/停用
```

### 2.2 登录模板渲染

登录页面模板位于 [login.html](file:///d:/fz/0601/solo-dogfeeding/code/113-paperless-ngx/src/documents/templates/account/login.html)。

**页面逻辑分支**：

1. **首次安装检测**（L18-L23）：若无任何用户且无文档，JS 自动跳转至注册页
2. **常规登录表单**（L24-L43）：受 `DISABLE_REGULAR_LOGIN` 控制，若禁用则不渲染用户名/密码输入框
3. **社交登录按钮列表**（L46-L83）：通过 `{% get_providers %}` 遍历 allauth 注册的提供商
   - OpenID 提供商额外遍历 brands
   - 若 `REDIRECT_LOGIN_TO_SSO=True` 且非登出状态，自动提交第一个社交登录表单

#### 2.2.1 社交登录 URL 的 `process` 参数关键区别

模板中通过 `{% provider_login_url provider process=process ... %}` 生成授权 URL。这里的 `process` 变量由 allauth 的登录视图注入模板上下文，在登录页面场景下默认为 `"login"`。

`process` 参数是 allauth 区分「登录注册」与「账号绑定」的核心机制：

| process 值 | 适用场景 | URL 示例 | 回调后的行为 |
|-----------|---------|---------|------------|
| `"login"` | 未登录用户访问 `/accounts/login/` 时点击社交登录按钮 | `/accounts/keycloak-test/login/?process=login` | 走登录/注册流程：查找已有 SocialAccount 或创建新 User |
| `"connect"` | 已登录用户在用户资料页点击「绑定社交账号」 | `/accounts/keycloak-test/login/?process=connect` | 走绑定流程：将 SocialAccount 直接关联到当前登录用户 |

**证据链**：
- 登录模板 [login.html L57/L61](file:///d:/fz/0601/solo-dogfeeding/code/113-paperless-ngx/src/documents/templates/account/login.html#L57-L61) 使用 `process=process`（来自 allauth 视图上下文，值为 `"login"`）
- 后端 Profile API [views.py L501/L510-L512](file:///d:/fz/0601/solo-dogfeeding/code/113-paperless-ngx/src/paperless/views.py#L501-L512) 明确传入 `process="connect"` 生成绑定用 URL
- 测试用例 [test_api_profile.py L49/L330](file:///d:/fz/0601/solo-dogfeeding/code/113-paperless-ngx/src/documents/tests/test_api_profile.py#L49-L330) 验证返回的 URL 包含 `?process=connect`

MFA 验证页面模板位于 [authenticate.html](file:///d:/fz/0601/solo-dogfeeding/code/113-paperless-ngx/src/documents/templates/mfa/authenticate.html)，仅包含一个 TOTP 验证码输入框和取消按钮。

社交登录确认页位于 [socialaccount/login.html](file:///d:/fz/0601/solo-dogfeeding/code/113-paperless-ngx/src/documents/templates/socialaccount/login.html)，在绑定社交账号前让用户确认（当 `REDIRECT_LOGIN_TO_SSO=True` 时会自动提交该表单）。

---

## 三、凭据校验流程

### 3.1 Django 认证后端链

在 [__init__.py](file:///d:/fz/0601/solo-dogfeeding/code/113-paperless-ngx/src/paperless/settings/__init__.py#L306-L310) 中配置的认证后端按顺序执行：

```python
AUTHENTICATION_BACKENDS = [
    "guardian.backends.ObjectPermissionBackend",       # 对象级权限
    "django.contrib.auth.backends.ModelBackend",        # Django 默认：用户名+密码
    "allauth.account.auth_backends.AuthenticationBackend",  # allauth：含社交登录
]
```

若启用了 HTTP Remote User（SSO），会额外在最前面插入：
```python
"django.contrib.auth.backends.RemoteUserBackend"
```

### 3.2 allauth 适配器校验钩子

paperless-ngx 通过自定义适配器扩展 allauth 的校验逻辑，定义在 [adapter.py](file:///d:/fz/0601/solo-dogfeeding/code/113-paperless-ngx/src/paperless/adapter.py)。

#### 3.2.1 预认证钩子：禁用常规登录

[CustomAccountAdapter.pre_authenticate](file:///d:/fz/0601/solo-dogfeeding/code/113-paperless-ngx/src/paperless/adapter.py#L39-L48) 在凭据校验前被 allauth 调用：

```python
def pre_authenticate(self, request, **credentials):
    if settings.DISABLE_REGULAR_LOGIN:
        raise ValidationError("Regular login is disabled")
    return super().pre_authenticate(request, **credentials)
```

环境变量 `PAPERLESS_DISABLE_REGULAR_LOGIN` 控制此开关。测试验证见 [test_adapter.py](file:///d:/fz/0601/solo-dogfeeding/code/113-paperless-ngx/src/paperless/tests/test_adapter.py#L56-L71)。

#### 3.2.2 注册开放检测

[CustomAccountAdapter.is_open_for_signup](file:///d:/fz/0601/solo-dogfeeding/code/113-paperless-ngx/src/paperless/adapter.py#L23-L37) 决定是否允许注册：
- 全新安装（无用户 + 无文档）→ 始终允许
- 否则受 `ACCOUNT_ALLOW_SIGNUPS`（即 `PAPERLESS_ACCOUNT_ALLOW_SIGNUPS`）控制

社交账号注册由 [CustomSocialAccountAdapter.is_open_for_signup](file:///d:/fz/0601/solo-dogfeeding/code/113-paperless-ngx/src/paperless/adapter.py#L108-L116) 单独控制，对应 `SOCIALACCOUNT_ALLOW_SIGNUPS`（默认 yes）。

#### 3.2.3 安全 URL 校验

[CustomAccountAdapter.is_safe_url](file:///d:/fz/0601/solo-dogfeeding/code/113-paperless-ngx/src/paperless/adapter.py#L50-L66) 防止开放重定向漏洞：
- 以当前请求的 host 为基础，合并 `ALLOWED_HOSTS`
- 通配符 `*` 会被移除（防止从任意 host 跳转）

### 3.3 DRF API Token 认证

在 [__init__.py](file:///d:/fz/0601/solo-dogfeeding/code/113-paperless-ngx/src/paperless/settings/__init__.py#L151-L167) 中配置的 DRF 认证类：

```python
REST_FRAMEWORK = {
    "DEFAULT_AUTHENTICATION_CLASSES": [
        "paperless.auth.PaperlessBasicAuthentication",    # HTTP Basic
        "rest_framework.authentication.TokenAuthentication",  # DRF Token
        "rest_framework.authentication.SessionAuthentication",  # Django Session
    ],
}
```

#### 3.3.1 HTTP Basic 认证的 MFA 强制校验

[PaperlessBasicAuthentication](file:///d:/fz/0601/solo-dogfeeding/code/113-paperless-ngx/src/paperless/auth.py#L77-L93) 覆写了 DRF 的 Basic 认证：

```python
def authenticate(self, request):
    user_tuple = super().authenticate(request)
    user = user_tuple[0] if user_tuple else None
    mfa_adapter = get_mfa_adapter()
    if user and mfa_adapter.is_mfa_enabled(user):
        raise exceptions.AuthenticationFailed("MFA required")
    return user_tuple
```

**重要**：使用 HTTP Basic 访问 API 时，若用户已启用 MFA，将被拒绝。

#### 3.3.2 DRF Token 获取（含 MFA 校验）

`/api/token/` 使用 [PaperlessObtainAuthTokenView](file:///d:/fz/0601/solo-dogfeeding/code/113-paperless-ngx/src/paperless/views.py#L55-L58)，其序列化器为 [PaperlessAuthTokenSerializer](file:///d:/fz/0601/solo-dogfeeding/code/113-paperless-ngx/src/paperless/serialisers.py#L47-L74)。

校验流程：
1. 父类 `AuthTokenSerializer.validate()` 完成用户名/密码校验
2. 若用户已启用 MFA：
   - 未传 `code` → 抛出 "MFA code is required"
   - 传了 `code` → 使用 `allauth.mfa.totp.internal.auth.TOTP.validate_code()` 验证
3. 校验通过后返回 Token

### 3.4 其他认证方式

#### 3.4.1 AutoLoginMiddleware

见 [auth.py](file:///d:/fz/0601/solo-dogfeeding/code/113-paperless-ngx/src/paperless/auth.py#L16-L29)：若设置了 `PAPERLESS_AUTO_LOGIN_USERNAME`，所有请求自动以该用户登录（跳过 /api/token/ POST 请求）。

#### 3.4.2 HttpRemoteUserMiddleware

见 [auth.py](file:///d:/fz/0601/solo-dogfeeding/code/113-paperless-ngx/src/paperless/auth.py#L50-L66)：基于 HTTP 头的 SSO 认证。当仅启用前端 SSO 时，跳过 /api/ 路径的认证。

#### 3.4.3 AngularApiAuthenticationOverride

见 [auth.py](file:///d:/fz/0601/solo-dogfeeding/code/113-paperless-ngx/src/paperless/auth.py#L32-L47)：仅开发环境（DEBUG=True）下，Referer 为 `http://localhost:4200/` 时自动以第一个 staff 用户登录。

---

## 四、双因素认证（2FA/MFA）代码路径

paperless-ngx 使用 allauth 的 MFA 模块，目前仅支持 **TOTP**（基于时间的一次性密码）。

### 4.1 TOTP 启用流程

```
  用户前端操作                         后端 API 处理
  ───────────                         ────────────
  1. 点击"启用 TOTP" ──────────────► GET /api/profile/totp/
                                         │
                                         ▼
                                   [TOTPView.get]
                                   - totp_auth.get_totp_secret(regenerate=True)
                                   - mfa_adapter.build_totp_url(user, secret)
                                   - mfa_adapter.build_totp_svg(url)
                                         │
                                         ▼
  ◄──────── 返回 url, qr_svg, secret
  2. 用户扫码，输入验证码 ────────► POST /api/profile/totp/
                                     {secret, code}
                                         │
                                         ▼
                                   [TOTPView.post]
                                   - totp_auth.validate_totp_code(secret, code)
                                   - 有效 → TOTP.activate(user, secret)
                                   - 发送 authenticator_added 信号
                                   - auto_generate_recovery_codes(request)
                                         │
                                         ▼
  ◄──────── 返回 {success, recovery_codes}
```

后端实现在 [views.py](file:///d:/fz/0601/solo-dogfeeding/code/113-paperless-ngx/src/paperless/views.py#L303-L370) 的 `TOTPView`。

前端调用在 [profile-edit-dialog.component.ts](file:///d:/fz/0601/solo-dogfeeding/code/113-paperless-ngx/src-ui/src/app/components/common/profile-edit-dialog/profile-edit-dialog.component.ts#L262-L306)：
- `gettotpSettings()` → `ProfileService.getTotpSettings()` → `GET /api/profile/totp/`
- `activateTotp()` → `ProfileService.activateTotp(secret, code)` → `POST /api/profile/totp/`

### 4.2 MFA 登录拦截流程

```
  用户输入用户名/密码
         │
         ▼
  allauth.account.auth_backends.AuthenticationBackend.authenticate()
         │
         ▼
  CustomAccountAdapter.pre_authenticate()  ← 检查 DISABLE_REGULAR_LOGIN
         │
         ▼
  用户名/密码校验通过
         │
         ▼
  allauth 检测用户是否已启用 MFA
  (通过 mfa_adapter.is_mfa_enabled(user))
         │
    ┌────┴────┐
    │         │
  未启用    已启用
    │         │
    │         ▼
    │    重定向至 /accounts/2fa/authenticate/
    │    (allauth.mfa.base.views.authenticate)
    │         │
    │         ▼
    │    渲染 mfa/authenticate.html 模板
    │         │
    │         ▼
    │    用户输入 TOTP code ──► POST /accounts/2fa/authenticate/
    │                              │
    │                              ▼
    │                         allauth 内部校验 TOTP
    │                              │
    │                              ▼
    │                         校验通过 → 完成登录
    ▼
  直接完成登录
```

### 4.3 TOTP 停用流程

两种方式停用 TOTP：

1. **用户自行停用**：`DELETE /api/profile/totp/` → [TOTPView.delete](file:///d:/fz/0601/solo-dogfeeding/code/113-paperless-ngx/src/paperless/views.py#L357-L370)
2. **管理员为用户停用**：`POST /api/users/{id}/deactivate_totp/` → [UserViewSet.deactivate_totp](file:///d:/fz/0601/solo-dogfeeding/code/113-paperless-ngx/src/paperless/views.py#L212-L228)，需要 superuser 权限或操作本人

两者都调用 `allauth.mfa.base.internal.flows.delete_and_cleanup()` 删除 `Authenticator` 记录。

### 4.4 MFA 相关设置

```python
# settings/__init__.py
MFA_TOTP_ISSUER = "Paperless-ngx"  # TOTP 应用名，显示在验证器中
```

### 4.5 MFA 状态展示

用户资料序列化器 [ProfileSerializer](file:///d:/fz/0601/solo-dogfeeding/code/113-paperless-ngx/src/paperless/serialisers.py#L179-L209) 和 [UserSerializer](file:///d:/fz/0601/solo-dogfeeding/code/113-paperless-ngx/src/paperless/serialisers.py#L77-L142) 中都包含 `is_mfa_enabled` 字段，通过 `mfa_adapter.is_mfa_enabled(user)` 计算。

---

## 五、社交登录（Social Account）代码路径

### 5.1 社交登录配置

在 [__init__.py](file:///d:/fz/0601/solo-dogfeeding/code/113-paperless-ngx/src/paperless/settings/__init__.py#L322-L338) 中：

```python
SOCIALACCOUNT_ADAPTER = "paperless.adapter.CustomSocialAccountAdapter"
SOCIALACCOUNT_ALLOW_SIGNUPS = ...  # PAPERLESS_SOCIALACCOUNT_ALLOW_SIGNUPS, 默认 yes
SOCIALACCOUNT_AUTO_SIGNUP = ...     # PAPERLESS_SOCIAL_AUTO_SIGNUP
SOCIALACCOUNT_PROVIDERS = ...        # PAPERLESS_SOCIALACCOUNT_PROVIDERS (JSON 格式)
SOCIAL_ACCOUNT_DEFAULT_GROUPS = ...  # PAPERLESS_SOCIAL_ACCOUNT_DEFAULT_GROUPS
SOCIAL_ACCOUNT_SYNC_GROUPS = ...     # PAPERLESS_SOCIAL_ACCOUNT_SYNC_GROUPS
SOCIAL_ACCOUNT_SYNC_GROUPS_CLAIM = "groups"  # 从哪个 claim 取组信息
HEADLESS_TOKEN_STRATEGY = "paperless.adapter.DrfTokenStrategy"
```

### 5.2 社交登录三种入口场景详解

paperless-ngx 中发起社交授权共有 **三条不同的入口路径**，分别对应不同用户状态和业务场景：

| 场景 | 用户状态 | 入口位置 | process 参数 | 目标 |
|-----|---------|---------|-------------|-----|
| **A. 登录页第三方登录** | 未登录 | `/accounts/login/` 页面底部按钮 | `process=login` | 登录或注册新用户 |
| **B. 已登录用户绑定新账号** | 已登录 | 用户资料对话框「绑定社交账号」 | `process=connect` | 为当前用户添加新的 SocialAccount |
| **C. SSO 自动跳转** | 未登录 | 访问任意受保护页面被重定向到登录页 | `process=login` | 同场景 A，但 `REDIRECT_LOGIN_TO_SSO=True` 时自动提交第一个提供商表单 |

---

#### 5.2.1 场景 A：登录页发起第三方授权（process=login）

```
  未登录用户
     │
     ▼
  GET /accounts/login/  或  GET /api/auth/login/
     │
     ▼
  allauth.account.views.login 视图
  → 渲染 account/login.html 模板
  → 注入上下文变量 process="login"
     │
     ▼
  模板渲染社交按钮：
  {% provider_login_url provider process=process scope=scope auth_params=auth_params as href %}
  生成 URL: /accounts/<provider_id>/login/?process=login
     │
     ▼
  用户点击按钮（POST 或 GET 跳转）
     │
     ▼
  POST/GET /accounts/<provider_id>/login/?process=login
     │
     ▼
  allauth.socialaccount 内部处理
  1. 将 process="login" 存入 session
  2. 构建 OAuth 授权 URL
  3. 302 重定向至第三方授权页面
```

**证据**：登录模板 [login.html L61](file:///d:/fz/0601/solo-dogfeeding/code/113-paperless-ngx/src/documents/templates/account/login.html#L61) 中 `process=process`，此处 `process` 由 allauth `LoginView` 上下文提供，值为 `"login"`。

---

#### 5.2.2 场景 B：已登录用户绑定社交账号（process=connect）

```
  已登录用户
     │
     ▼
  打开前端 ProfileEditDialog 对话框
     │
     ▼
  GET /api/profile/social_account_providers/
     │
     ▼
  [SocialAccountProvidersView.get]  [views.py L497-L519]
  - adapter.list_providers(request)
  - 对每个 provider 调用 p.get_login_url(request, process="connect") ← 关键！
  - 返回 [{name, login_url}]
     │
     ▼
  返回的 URL 示例: /accounts/<provider_id>/login/?process=connect
     │
     ▼
  用户点击前端「绑定」按钮 → window.location.href = login_url
     │
     ▼
  GET /accounts/<provider_id>/login/?process=connect
     │
     ▼
  allauth.socialaccount 内部处理
  1. 将 process="connect" 存入 session
  2. 构建 OAuth 授权 URL
  3. 302 重定向至第三方授权页面
```

**证据**：后端明确使用 `process="connect"` —— [views.py L501](file:///d:/fz/0601/solo-dogfeeding/code/113-paperless-ngx/src/paperless/views.py#L501)：
```python
{"name": p.name, "login_url": p.get_login_url(request, process="connect")}
```

测试验证：[test_api_profile.py L329-L332](file:///d:/fz/0601/solo-dogfeeding/code/113-paperless-ngx/src/documents/tests/test_api_profile.py#L329-L332) 断言返回的 URL 含 `"keycloak-test/login/?process=connect"`。

---

#### 5.2.3 场景 C：SSO 自动跳转

当 `REDIRECT_LOGIN_TO_SSO=True`（环境变量 `PAPERLESS_REDIRECT_LOGIN_TO_SSO`）时，登录页渲染后会自动执行 JS 提交第一个社交登录表单：

```javascript
// login.html L68-L79
if (REDIRECT_LOGIN_TO_SSO && forloop.counter0 == 0 && request.GET.loggedout != '1') {
    const form = document.getElementById('social-login');
    if (form) { form.submit(); }
    else { document.getElementsByClassName('oidc-url')[0].click(); }
}
```

行为同场景 A，只是由浏览器自动触发而无需用户点击。

---

### 5.3 OAuth 回调：allauth 内部如何根据 process 分流

第三方 OAuth 完成后，浏览器被重定向回回调 URL：

```
  第三方授权成功
     │
     ▼
  302 回跳 → GET /accounts/<provider_id>/login/callback/
     │
     ▼
  allauth.socialaccount 回调视图
  1. 校验 OAuth state，兑换 access_token
  2. 获取用户信息（uid、email 等）
  3. 从 session 中取出之前存入的 process 参数
     │
     ▼
  ┌─────────────── process 值是？ ───────────────┐
  │                                               │
  │          "login"                     "connect" │
  │             │                           │     │
  │             ▼                           ▼     │
  │    【登录/注册分支】            【绑定分支】    │
  │             │                           │     │
  │             ▼                           ▼     │
  │   SocialAccount 已存在？         当前 request.user 已认证 │
  │        │        │                    │     │
  │       是        否                   │     │
  │        │        │                    ▼     │
  │        ▼        ▼           关联 SocialAccount │
  │    直接登录   当前有用户登录？       到 request.user │
  │             │        │             │     │
  │            是        否             ▼     │
  │             │        │        触发 social_account_updated │
  │             ▼        ▼           信号 → 组同步 │
  │         绑定账号  AUTO_SIGNUP?     │     │
  │                    │    │          ▼     │
  │                   是    否    跳转首页 │
  │                    │    │                │
  │                    ▼    ▼                │
  │               自动创建  跳转             │
  │               用户并登  /accounts/       │
  │               录      3rdparty/signup/   │
  │                    │                     │
  │                    ▼                     │
  │           CustomSocialAccountAdapter     │
  │             .save_user()                 │
  │                    │                     │
  │                    ▼                     │
  │           添加默认组 + 组同步             │
  │                    │                     │
  │                    ▼                     │
  │                 登录成功                  │
  └──────────────────────────────────────────┘
```

**process 参数的持久化**：allauth 在用户访问 `/accounts/<provider_id>/login/` 发起授权时，将 `process` 值存入 Django session；回调时从 session 中读取，因此即使在第三方站点跳转后也能记住最初的意图（登录 vs 绑定）。

---

### 5.4 社交登录完整端到端流程（场景 A：未登录用户 + process=login）

```
  前端                                  后端
  ───                                  ───
  1. 获取可用提供商（仅场景 B 需要，场景 A 直接从登录页按钮发起）
     │
     ▼
  2. 用户点击登录页的社交登录按钮
     → POST /accounts/<provider_id>/login/?process=login
     │
     ▼
  allauth.socialaccount.views.login
  → 将 process="login" 写入 session
  → 重定向至 OAuth 提供商授权页
     │
     ▼
  3. 用户在第三方完成授权
     ← 提供商回调 GET /accounts/<provider_id>/login/callback/
     │
     ▼
  allauth.socialaccount 内部回调处理
  → 校验 OAuth state，兑换 access_token，获取用户信息
  → 从 session 读取 process="login"
     │
     ▼
  4. 判断账号关联情况（按 process="login" 分支处理）
     ┌──────────────────────────────┐
     │ SocialAccount 已存在？       │
     └──┬───────────────┬───────────┘
        │ 已关联         │ 未关联
        ▼                ▼
     直接登录        ┌──────────────────────┐
                     │ 已登录当前用户？      │
                     └──┬──────────┬────────┘
                        │ 是       │ 否
                        ▼          ▼
                     绑定账号   SOCIALACCOUNT_AUTO_SIGNUP?
                               ┌────┴────┐
                               │ 是      │ 否
                               ▼         ▼
                           自动创建   跳转 /accounts/3rdparty/signup/
                           用户并     让用户补充信息或
                           绑定       关联现有账号
                               │
                               ▼
                     [CustomSocialAccountAdapter.save_user]
                     - 调用父类保存（也会触发 AccountAdapter.save_user）
                     - 添加 SOCIAL_ACCOUNT_DEFAULT_GROUPS
                     - 触发 social_account_updated 信号
                     ─────────────────────────────────────────
                               │
                               ▼
                     [handle_social_account_updated 信号处理器]
                     - 从 extra_data.userinfo 或 id_token 读取 groups claim
                     - 若 SOCIAL_ACCOUNT_SYNC_GROUPS=True，同步用户组
                               │
                               ▼
                     登录成功，重定向到首页
```

### 5.5 关键代码位置

#### 5.5.1 社交登录入口与提供商列表

后端：[SocialAccountProvidersView](file:///d:/fz/0601/solo-dogfeeding/code/113-paperless-ngx/src/paperless/views.py#L490-L519)

前端：[profile-edit-dialog.component.ts](file:///d:/fz/0601/solo-dogfeeding/code/113-paperless-ngx/src-ui/src/app/components/common/profile-edit-dialog/profile-edit-dialog.component.ts#L120-L126) → [ProfileService.getSocialAccountProviders](file:///d:/fz/0601/solo-dogfeeding/code/113-paperless-ngx/src-ui/src/app/services/profile.service.ts#L46-L50)

#### 5.5.2 社交账号保存与默认组

[CustomSocialAccountAdapter.save_user](file:///d:/fz/0601/solo-dogfeeding/code/113-paperless-ngx/src/paperless/adapter.py#L126-L142)：

```python
def save_user(self, request, sociallogin, form=None):
    user: User = super().save_user(request, sociallogin, form)
    group_names: list[str] = settings.SOCIAL_ACCOUNT_DEFAULT_GROUPS
    if len(group_names) > 0:
        groups = Group.objects.filter(name__in=group_names)
        user.groups.add(*groups)
        user.save()
    handle_social_account_updated(None, request, sociallogin)
    return user
```

注意：父类 `save_user` 也会调用 `CustomAccountAdapter.save_user`，后者会设置 `ACCOUNT_DEFAULT_GROUPS`。最终用户会获得两组默认组的并集。

#### 5.5.3 社交账号组同步信号

信号注册在 [apps.py](file:///d:/fz/0601/solo-dogfeeding/code/113-paperless-ngx/src/paperless/apps.py#L18-L20)：

```python
from allauth.socialaccount.signals import social_account_updated
social_account_updated.connect(handle_social_account_updated)
```

信号处理器 [handle_social_account_updated](file:///d:/fz/0601/solo-dogfeeding/code/113-paperless-ngx/src/paperless/signals.py#L35-L63)：
- 从 `sociallogin.account.extra_data` 中读取组信息
- 兼容两种结构：直接的 `groups` 字段，或嵌套在 `userinfo`/`id_token` 下
- 若 `SOCIAL_ACCOUNT_SYNC_GROUPS=True`，用 `groups.set(clear=True)` 覆盖用户组

#### 5.5.4 Headless Token 策略

[DrfTokenStrategy](file:///d:/fz/0601/solo-dogfeeding/code/113-paperless-ngx/src/paperless/adapter.py#L167-L172) 供 allauth headless 模式使用，登录后自动返回 DRF Token：

```python
class DrfTokenStrategy(SessionTokenStrategy):
    def create_access_token(self, request: HttpRequest) -> str | None:
        if not request.user.is_authenticated:
            return None
        token, _ = Token.objects.get_or_create(user=request.user)
        return token.key
```

---

## 六、账号绑定逻辑

### 6.1 社交账号与本地账号的关联关系

使用 allauth 的 `SocialAccount` 模型，通过外键关联到 `User`：

```
User (django.contrib.auth)
  └── SocialAccount (allauth.socialaccount)
        - user: ForeignKey(User)
        - provider: str (如 "google", "github")
        - uid: str (第三方用户唯一 ID)
        - extra_data: JSON (OAuth 返回的用户信息)
        - date_joined
        - last_login
```

### 6.2 绑定（Connect）流程详解

绑定社交账号的完整前后端链路（对应第五章场景 B）：

```
  前端 (Angular)                            后端 (Django)
  ────────────────                          ──────────────
  1. 用户点击头像 → 「Edit Profile」
     │
     ▼
  ProfileEditDialog 打开
     │
     ▼
  2. 调用 ProfileService.getProfile()
     GET /api/profile/
     │                                    [ProfileView.get]
     │                                    返回 UserProfile（含已绑定 social_accounts）
     │◄───────────────────────────────────
     │
     ▼
  渲染「Social Accounts」区域
  - 列出已绑定账号（含「Disconnect」按钮）
  - 渲染「Connect」按钮区域
     │
     ▼
  3. 用户点击「Connect」按钮 ────────────────┐
                                              │
  4. ProfileService.getSocialAccountProviders() │
     GET /api/profile/social_account_providers/ │
                                              │
                                              ▼
                                    [SocialAccountProvidersView.get]
                                    - adapter.list_providers(request)
                                    - p.get_login_url(request, process="connect")
                                    返回 [{name, login_url}]
     │◄───────────────────────────────────────┘
     │
     ▼
  前端显示可用提供商列表下拉框
     │
     ▼
  5. 用户选择某个提供商（如 Keycloak）
     │
     ▼
  window.location.href = provider.login_url
  （浏览器离开 Angular，跳转到 Django 社交登录 URL）
     │
     ▼
  GET /accounts/<provider_id>/login/?process=connect
     │
     ▼
  allauth.socialaccount 内部处理
  - session 存入 process="connect"
  - 302 重定向到第三方 OAuth 授权页
     │
     ▼
  6. 用户在第三方完成授权
     │
     ▼
  第三方 302 重定向回 Django
  GET /accounts/<provider_id>/login/callback/
     │
     ▼
  allauth.socialaccount 回调视图
  - 校验 state，兑换 access_token
  - 从 session 读取 process="connect"
  - 确认 request.user.is_authenticated（必须已登录）
  - 创建 SocialAccount 记录，关联到 request.user
  - 触发 social_account_updated 信号 → 组同步
     │
     ▼
  [CustomSocialAccountAdapter.get_connect_redirect_url]
  → 返回 reverse("base")（即 SPA 首页 /）
     │
     ▼
  302 重定向回 Angular 首页
  前端重新加载，用户已绑定新社交账号
```

**后端绑定重定向**实现见 [adapter.py L118-L124](file:///d:/fz/0601/solo-dogfeeding/code/113-paperless-ngx/src/paperless/adapter.py#L118-L124)：

```python
def get_connect_redirect_url(self, request, socialaccount):
    assert is_authenticated(request.user)
    return redirect_by_name("base")
```

---

### 6.3 解绑（Disconnect）流程详解

解绑不经过 OAuth，完全通过 DRF API 完成：

```
  前端 (Angular)                            后端 (Django)
  ────────────────                          ──────────────
  1. ProfileEditDialog 已打开
     显示当前已绑定的 social_accounts
     │
     ▼
  2. 用户点击某个账号的「Disconnect」按钮
     │
     ▼
  3. ProfileService.disconnectSocialAccount(id)
     POST /api/profile/disconnect_social_account/
     Body: { "id": <social_account_id> }
                                              │
                                              ▼
                                    [DisconnectSocialAccountView.post]
                                    1. user = self.request.user
                                    2. account = user.socialaccount_set.get(pk=id)
                                       (通过 user 过滤，防止越权删除他人账号)
                                    3. account.delete()
                                    4. return Response(account_id)
     │◄───────────────────────────────────────┘
     │
     ▼
  4. 前端收到 200，从 social_accounts 数组中移除该 id
     UI 立即更新，无需刷新页面
     │
     ▼
  5. 调用 ProfileService.getSocialAccountProviders()
     刷新可用提供商列表（解绑后可能可再次绑定）
```

后端实现 [DisconnectSocialAccountView](file:///d:/fz/0601/solo-dogfeeding/code/113-paperless-ngx/src/paperless/views.py#L464-L480) 的安全设计：
- 使用 `user.socialaccount_set.get(pk=...)` 而不是 `SocialAccount.objects.get(pk=...)`
- 这确保用户只能删除属于自己的社交账号，防止 IDOR（不安全直接对象引用）攻击

前端调用在 [profile-edit-dialog.component.ts L245-L260](file:///d:/fz/0601/solo-dogfeeding/code/113-paperless-ngx/src-ui/src/app/components/common/profile-edit-dialog/profile-edit-dialog.component.ts#L245-L260) 的 `disconnectSocialAccount()`。

---

### 6.3.1 绑定/解绑路径对比总结

| 操作 | 是否需要 OAuth | 入口 API | 核心参数 | 调用方 |
|-----|--------------|---------|---------|-------|
| **绑定 (Connect)** | 是 | 先 `GET /api/profile/social_account_providers/` 获取 URL，再浏览器跳转 `process=connect` 的社交登录 URL | `process="connect"` | 浏览器（离开 SPA → Django → 第三方 → Django → SPA） |
| **解绑 (Disconnect)** | 否 | `POST /api/profile/disconnect_social_account/` | `{ id: <social_account_id> }` | Angular HTTP 客户端（纯 AJAX） |

### 6.4 用户资料中的账号绑定信息展示

[ProfileSerializer](file:///d:/fz/0601/solo-dogfeeding/code/113-paperless-ngx/src/paperless/serialisers.py#L179-L209) 通过 `social_accounts` 字段（映射 `user.socialaccount_set`）返回当前用户已绑定的社交账号列表：

```python
social_accounts = SocialAccountSerializer(
    many=True,
    read_only=True,
    source="socialaccount_set",
)
```

[SocialAccountSerializer](file:///d:/fz/0601/solo-dogfeeding/code/113-paperless-ngx/src/paperless/serialisers.py#L161-L176) 包含：
- `id`: SocialAccount 主键（用于解绑）
- `provider`: 提供商 ID
- `name`: 通过 `obj.get_provider_account().to_str()` 获取显示名

前端数据模型见 [user-profile.ts](file:///d:/fz/0601/solo-dogfeeding/code/113-paperless-ngx/src-ui/src/app/data/user-profile.ts) 的 `SocialAccount` 和 `SocialAccountProvider` 接口。

### 6.5 新用户注册时的默认组分配

无论是常规注册还是社交账号注册，paperless-ngx 都通过适配器的 `save_user` 方法分配默认组：

| 用户来源 | 默认组配置 | 代码位置 |
|---------|----------|--------|
| 常规注册 | `ACCOUNT_DEFAULT_GROUPS` (`PAPERLESS_ACCOUNT_DEFAULT_GROUPS`) | [adapter.py L82-L105](file:///d:/fz/0601/solo-dogfeeding/code/113-paperless-ngx/src/paperless/adapter.py#L82-L105) |
| 社交注册 | `SOCIAL_ACCOUNT_DEFAULT_GROUPS` (`PAPERLESS_SOCIAL_ACCOUNT_DEFAULT_GROUPS`) + 常规组 | [adapter.py L126-L142](file:///d:/fz/0601/solo-dogfeeding/code/113-paperless-ngx/src/paperless/adapter.py#L126-L142) |
| 全新安装首个用户 | 自动设为 superuser + staff | [adapter.py L88-L96](file:///d:/fz/0601/solo-dogfeeding/code/113-paperless-ngx/src/paperless/adapter.py#L88-L96) |

---

## 七、登录失败日志

所有登录失败均通过 Django 的 `user_login_failed` 信号记录日志，连接在 [apps.py](file:///d:/fz/0601/solo-dogfeeding/code/113-paperless-ngx/src/paperless/apps.py#L14-L16)。

处理器 [handle_failed_login](file:///d:/fz/0601/solo-dogfeeding/code/113-paperless-ngx/src/paperless/signals.py#L10-L32) 记录：
- 尝试登录的用户名
- 客户端 IP（区分公网/私网，通过 `TRUSTED_PROXIES` 正确获取真实 IP）

---

## 八、核心文件速查表

| 文件 | 作用 |
|-----|------|
| [src/paperless/settings/__init__.py](file:///d:/fz/0601/solo-dogfeeding/code/113-paperless-ngx/src/paperless/settings/__init__.py) | 认证后端、allauth、MFA、社交登录全局配置 |
| [src/paperless/urls.py](file:///d:/fz/0601/solo-dogfeeding/code/113-paperless-ngx/src/paperless/urls.py) | 所有认证 URL 路由 |
| [src/paperless/adapter.py](file:///d:/fz/0601/solo-dogfeeding/code/113-paperless-ngx/src/paperless/adapter.py) | CustomAccountAdapter / CustomSocialAccountAdapter / DrfTokenStrategy |
| [src/paperless/auth.py](file:///d:/fz/0601/solo-dogfeeding/code/113-paperless-ngx/src/paperless/auth.py) | AutoLoginMiddleware / PaperlessBasicAuthentication / RemoteUser 支持 |
| [src/paperless/views.py](file:///d:/fz/0601/solo-dogfeeding/code/113-paperless-ngx/src/paperless/views.py) | TOTPView / ProfileView / DisconnectSocialAccountView / SocialAccountProvidersView |
| [src/paperless/serialisers.py](file:///d:/fz/0601/solo-dogfeeding/code/113-paperless-ngx/src/paperless/serialisers.py) | PaperlessAuthTokenSerializer / ProfileSerializer / UserSerializer / SocialAccountSerializer |
| [src/paperless/signals.py](file:///d:/fz/0601/solo-dogfeeding/code/113-paperless-ngx/src/paperless/signals.py) | 登录失败日志 / 社交账号组同步 |
| [src/paperless/apps.py](file:///d:/fz/0601/solo-dogfeeding/code/113-paperless-ngx/src/paperless/apps.py) | 信号连接注册 |
| [src/documents/templates/account/login.html](file:///d:/fz/0601/solo-dogfeeding/code/113-paperless-ngx/src/documents/templates/account/login.html) | 登录页模板 |
| [src/documents/templates/mfa/authenticate.html](file:///d:/fz/0601/solo-dogfeeding/code/113-paperless-ngx/src/documents/templates/mfa/authenticate.html) | MFA 验证页模板 |
| [src/documents/templates/socialaccount/login.html](file:///d:/fz/0601/solo-dogfeeding/code/113-paperless-ngx/src/documents/templates/socialaccount/login.html) | 社交登录确认页 |
| [src-ui/src/app/services/profile.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/113-paperless-ngx/src-ui/src/app/services/profile.service.ts) | 前端 Profile API 调用封装 |
| [src-ui/src/app/components/common/profile-edit-dialog/profile-edit-dialog.component.ts](file:///d:/fz/0601/solo-dogfeeding/code/113-paperless-ngx/src-ui/src/app/components/common/profile-edit-dialog/profile-edit-dialog.component.ts) | 前端资料编辑对话框（TOTP、社交账号管理） |
| [src-ui/src/app/data/user-profile.ts](file:///d:/fz/0601/solo-dogfeeding/code/113-paperless-ngx/src-ui/src/app/data/user-profile.ts) | 前端用户资料、社交账号、TOTP 的 TypeScript 接口 |
