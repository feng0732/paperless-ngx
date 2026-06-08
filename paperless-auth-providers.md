# Paperless-ngx 认证系统代码分析

基于对 paperless-ngx 源代码的逐条核实，梳理双因素认证（2FA/MFA）与社交登录的代码处理路径。所有结论均可在引用的代码位置直接验证。

---

## 一、认证入口：URL 路由

所有认证相关 URL 均定义在 [urls.py](file:///d:/fz/0601/solo-dogfeeding/code/113-paperless-ngx/src/paperless/urls.py)。

### 1.1 常规账号认证

```python
# /accounts/login/   (GET/POST)
path("login/", allauth_account_views.login, name="account_login"),    # [urls.py L337]
# /accounts/logout/  (GET, ACCOUNT_LOGOUT_ON_GET=True)
path("logout/", allauth_account_views.logout, name="account_logout"), # [urls.py L338]
# /accounts/signup/  (GET/POST)
path("signup/", allauth_account_views.signup, name="account_signup"), # [urls.py L339]
```

DRF API 风格的同一路由（共享同一个 allauth 视图函数）：

```python
# /api/auth/login/   (GET/POST)  — [urls.py L103-L107]
# /api/auth/logout/  (GET/POST)  — [urls.py L108-L112]
```

**验证测试**：[test_api_auth.py L12-L23](file:///d:/fz/0601/solo-dogfeeding/code/113-paperless-ngx/src/paperless/tests/test_api_auth.py#L12-L23) 确认两个路径使用完全相同的视图类。

### 1.2 双因素认证入口

```python
# /accounts/2fa/authenticate/  (GET/POST)
path("2fa/authenticate/", allauth_mfa_views.authenticate, name="mfa_authenticate"),  # [urls.py L404]
```

### 1.3 社交账号入口

**通用状态页面**（固定在 `/accounts/3rdparty/` 下）— [urls.py L378-L399]：

```python
# /accounts/3rdparty/login/cancelled/
# /accounts/3rdparty/login/error/
# /accounts/3rdparty/signup/
```

**各 OAuth 提供商的独立 URL**（由 `build_provider_urlpatterns()` 动态生成，直接挂载在 `/accounts/` 下）— [urls.py L401]：

```
/accounts/<provider_id>/login/           → 发起 OAuth 授权
/accounts/<provider_id>/login/callback/  → OAuth 回调地址
```

例：`/accounts/keycloak-test/login/?process=connect`

**证据**：测试 Mock 返回 `f"{self.app.provider_id}/login/?process=connect"`，断言包含 `"keycloak-test/login/?process=connect"` — [test_api_profile.py L49/L330](file:///d:/fz/0601/solo-dogfeeding/code/113-paperless-ngx/src/documents/tests/test_api_profile.py#L49-L330)。

### 1.4 Headless API 入口

```python
# /api/auth/headless/  — allauth 无头模式（供 SPA 前端调用的 JSON API）
re_path("^auth/headless/", include("allauth.headless.urls")),  # [urls.py L283]
```

### 1.5 Profile API 入口（账号管理）— [urls.py L223-L249]

```python
# GET/PATCH /api/profile/                           获取/更新用户资料            [urls.py L227]
# POST      /api/profile/generate_auth_token/       生成 API Token                [urls.py L232]
# POST      /api/profile/disconnect_social_account/ 解绑社交账号                  [urls.py L236]
# GET       /api/profile/social_account_providers/  获取可用社交登录提供商        [urls.py L240]
# GET/POST/DELETE /api/profile/totp/                TOTP 密钥获取/激活/停用      [urls.py L244]
```

### 1.6 DRF Token 获取入口

```python
# POST /api/token/  — 获取 DRF Auth Token（含 MFA 校验）
path("token/", PaperlessObtainAuthTokenView.as_view()),  # [urls.py L219]
```

---

## 二、登录模板与社交授权入口

登录页面模板：[login.html](file:///d:/fz/0601/solo-dogfeeding/code/113-paperless-ngx/src/documents/templates/account/login.html)

### 2.1 页面逻辑分支

1. **首次安装检测** [login.html L18-L23]：无任何用户且无文档时，JS 自动跳转 `signup_url`
2. **常规登录表单** [login.html L24-L43]：受 `DISABLE_REGULAR_LOGIN` 控制，禁用则不渲染
3. **社交登录按钮列表** [login.html L46-L83]：`{% get_providers %}` 遍历 allauth 注册的提供商
   - OpenID 提供商额外遍历 brands [login.html L55-L59]
   - `REDIRECT_LOGIN_TO_SSO=True` 且非登出状态时，JS 自动提交第一个社交登录表单 [login.html L68-L79]

### 2.2 社交登录 URL 的 `process` 参数

模板中生成授权 URL 的方式 [login.html L57/L61]：

```django
{% provider_login_url provider process=process scope=scope auth_params=auth_params as href %}
```

其中 `process` 变量由 allauth 登录视图注入模板上下文。后端也可显式指定：

| process 值 | 使用位置 | 代码证据 |
|-----------|---------|---------|
| `"login"` | 登录页模板上下文（allauth 默认） | [login.html L61] `process=process` |
| `"connect"` | Profile API 生成绑定用 URL | [views.py L501](file:///d:/fz/0601/solo-dogfeeding/code/113-paperless-ngx/src/paperless/views.py#L501) `p.get_login_url(request, process="connect")` |

### 2.3 其他模板

- MFA 验证页：[mfa/authenticate.html](file:///d:/fz/0601/solo-dogfeeding/code/113-paperless-ngx/src/documents/templates/mfa/authenticate.html) — TOTP 验证码输入框 + 取消按钮
- 社交登录确认页：[socialaccount/login.html](file:///d:/fz/0601/solo-dogfeeding/code/113-paperless-ngx/src/documents/templates/socialaccount/login.html)

---

## 三、凭据校验

### 3.1 Django 认证后端链

[__init__.py L306-L310](file:///d:/fz/0601/solo-dogfeeding/code/113-paperless-ngx/src/paperless/settings/__init__.py#L306-L310)：

```python
AUTHENTICATION_BACKENDS = [
    "guardian.backends.ObjectPermissionBackend",
    "django.contrib.auth.backends.ModelBackend",
    "allauth.account.auth_backends.AuthenticationBackend",
]
```

启用 HTTP Remote User SSO 时，额外在最前面插入 `"django.contrib.auth.backends.RemoteUserBackend"`。

### 3.2 allauth 自定义适配器

定义在 [adapter.py](file:///d:/fz/0601/solo-dogfeeding/code/113-paperless-ngx/src/paperless/adapter.py)。

#### 3.2.1 预认证钩子：禁用常规登录

[CustomAccountAdapter.pre_authenticate](file:///d:/fz/0601/solo-dogfeeding/code/113-paperless-ngx/src/paperless/adapter.py#L39-L48)：

```python
def pre_authenticate(self, request, **credentials):
    if settings.DISABLE_REGULAR_LOGIN:
        raise ValidationError("Regular login is disabled")
    return super().pre_authenticate(request, **credentials)
```

测试：[test_adapter.py L56-L71](file:///d:/fz/0601/solo-dogfeeding/code/113-paperless-ngx/src/paperless/tests/test_adapter.py#L56-L71)。

#### 3.2.2 注册开放检测

- [CustomAccountAdapter.is_open_for_signup](file:///d:/fz/0601/solo-dogfeeding/code/113-paperless-ngx/src/paperless/adapter.py#L23-L37)：全新安装（无用户+无文档）始终允许，否则受 `ACCOUNT_ALLOW_SIGNUPS` 控制
- [CustomSocialAccountAdapter.is_open_for_signup](file:///d:/fz/0601/solo-dogfeeding/code/113-paperless-ngx/src/paperless/adapter.py#L108-L116)：受 `SOCIALACCOUNT_ALLOW_SIGNUPS` 控制（默认 yes）

#### 3.2.3 安全 URL 校验

[CustomAccountAdapter.is_safe_url](file:///d:/fz/0601/solo-dogfeeding/code/113-paperless-ngx/src/paperless/adapter.py#L50-L66)：合并当前请求 host 与 `ALLOWED_HOSTS`，移除通配符 `*` 后校验。

#### 3.2.4 社交绑定完成后重定向

[CustomSocialAccountAdapter.get_connect_redirect_url](file:///d:/fz/0601/solo-dogfeeding/code/113-paperless-ngx/src/paperless/adapter.py#L118-L124)：

```python
def get_connect_redirect_url(self, request, socialaccount):
    url = reverse("base")
    return url   # 返回字符串 "/"
```

#### 3.2.5 新用户保存与默认组分配

- 常规注册：[CustomAccountAdapter.save_user](file:///d:/fz/0601/solo-dogfeeding/code/113-paperless-ngx/src/paperless/adapter.py#L82-L105)
  - 全新安装首个用户设为 superuser + staff [adapter.py L88-L96]
  - 分配 `ACCOUNT_DEFAULT_GROUPS` [adapter.py L99-L104]

- 社交注册：[CustomSocialAccountAdapter.save_user](file:///d:/fz/0601/solo-dogfeeding/code/113-paperless-ngx/src/paperless/adapter.py#L126-L142)
  - 先调用父类（会触发 AccountAdapter.save_user，分配 `ACCOUNT_DEFAULT_GROUPS`）
  - 再额外分配 `SOCIAL_ACCOUNT_DEFAULT_GROUPS` [adapter.py L133-L140]
  - 手动触发 `handle_social_account_updated` 信号 [adapter.py L141]

### 3.3 DRF 认证配置

[__init__.py L151-L167](file:///d:/fz/0601/solo-dogfeeding/code/113-paperless-ngx/src/paperless/settings/__init__.py#L151-L167)：

```python
REST_FRAMEWORK = {
    "DEFAULT_AUTHENTICATION_CLASSES": [
        "paperless.auth.PaperlessBasicAuthentication",
        "rest_framework.authentication.TokenAuthentication",
        "rest_framework.authentication.SessionAuthentication",
    ],
}
```

DEBUG 时追加 `"paperless.auth.AngularApiAuthenticationOverride"`。

#### 3.3.1 HTTP Basic 认证的 MFA 强制校验

[PaperlessBasicAuthentication.authenticate](file:///d:/fz/0601/solo-dogfeeding/code/113-paperless-ngx/src/paperless/auth.py#L77-L85)：MFA 用户使用 HTTP Basic 访问 API 会被拒绝。

#### 3.3.2 DRF Token 获取（含 MFA 校验）

视图：[PaperlessObtainAuthTokenView](file:///d:/fz/0601/solo-dogfeeding/code/113-paperless-ngx/src/paperless/views.py#L55-L58)

序列化器：[PaperlessAuthTokenSerializer.validate](file:///d:/fz/0601/solo-dogfeeding/code/113-paperless-ngx/src/paperless/serialisers.py#L47-L74)
- 父类校验用户名/密码
- MFA 用户必须传 `code`，否则抛 `"MFA code is required"`
- 传了 `code` 则用 `TOTP(instance=authenticator).validate_code(code)` 验证

### 3.4 其他认证中间件/后端

| 名称 | 文件位置 | 说明 |
|-----|---------|------|
| AutoLoginMiddleware | [auth.py L16-L29](file:///d:/fz/0601/solo-dogfeeding/code/113-paperless-ngx/src/paperless/auth.py#L16-L29) | `PAPERLESS_AUTO_LOGIN_USERNAME` 设置时，所有请求自动登录（跳过 `/api/token/` POST） |
| HttpRemoteUserMiddleware | [auth.py L50-L66](file:///d:/fz/0601/solo-dogfeeding/code/113-paperless-ngx/src/paperless/auth.py#L50-L66) | HTTP_REMOTE_USER SSO；仅前端 SSO 时跳过 `/api/` 路径 |
| AngularApiAuthenticationOverride | [auth.py L32-L47](file:///d:/fz/0601/solo-dogfeeding/code/113-paperless-ngx/src/paperless/auth.py#L32-L47) | DEBUG + Referer `http://localhost:4200/` 时自动以第一个 staff 用户登录 |
| PaperlessRemoteUserAuthentication | [auth.py L69-L74](file:///d:/fz/0601/solo-dogfeeding/code/113-paperless-ngx/src/paperless/auth.py#L69-L74) | DRF 的 REMOTE_USER 认证，覆盖默认 header |
| DrfTokenStrategy | [adapter.py L167-L172](file:///d:/fz/0601/solo-dogfeeding/code/113-paperless-ngx/src/paperless/adapter.py#L167-L172) | allauth headless 模式登录后自动返回 DRF Token |

---

## 四、双因素认证（TOTP）

### 4.1 TOTP 启用

视图：[TOTPView](file:///d:/fz/0601/solo-dogfeeding/code/113-paperless-ngx/src/paperless/views.py#L303-L370)

| 方法 | 路径 | 行为 | 代码位置 |
|-----|-----|------|---------|
| GET | `/api/profile/totp/` | 返回新 secret、TOTP URL、QR SVG | [views.py L310-L325](file:///d:/fz/0601/solo-dogfeeding/code/113-paperless-ngx/src/paperless/views.py#L310-L325) |
| POST | `/api/profile/totp/` | Body: `{secret, code}`；验证通过则激活 TOTP 并返回恢复码 | [views.py L327-L355](file:///d:/fz/0601/solo-dogfeeding/code/113-paperless-ngx/src/paperless/views.py#L327-L355) |

POST 时额外操作：发送 `authenticator_added` 信号、调用 `auto_generate_recovery_codes()`。

### 4.2 TOTP 停用

两种方式：

1. **用户自行停用**：`DELETE /api/profile/totp/` → [TOTPView.delete](file:///d:/fz/0601/solo-dogfeeding/code/113-paperless-ngx/src/paperless/views.py#L357-L370)
2. **管理员为用户停用**：`POST /api/users/{id}/deactivate_totp/` → [UserViewSet.deactivate_totp](file:///d:/fz/0601/solo-dogfeeding/code/113-paperless-ngx/src/paperless/views.py#L212-L228)（需 superuser 或操作本人）

两者均调用 `delete_and_cleanup(request, authenticator)`。

### 4.3 MFA 状态展示

- [ProfileSerializer.get_is_mfa_enabled](file:///d:/fz/0601/solo-dogfeeding/code/113-paperless-ngx/src/paperless/serialisers.py#L191-L193)
- [UserSerializer.get_is_mfa_enabled](file:///d:/fz/0601/solo-dogfeeding/code/113-paperless-ngx/src/paperless/serialisers.py#L88-L90)

均通过 `mfa_adapter.is_mfa_enabled(user)` 计算。

### 4.4 MFA 相关设置

```python
MFA_TOTP_ISSUER = "Paperless-ngx"  # [__init__.py L342]
```

---

## 五、社交登录

### 5.1 全局配置

[__init__.py L322-L340](file:///d:/fz/0601/solo-dogfeeding/code/113-paperless-ngx/src/paperless/settings/__init__.py#L322-L340)：

```python
SOCIALACCOUNT_ADAPTER = "paperless.adapter.CustomSocialAccountAdapter"
SOCIALACCOUNT_ALLOW_SIGNUPS = get_bool_from_env("PAPERLESS_SOCIALACCOUNT_ALLOW_SIGNUPS", "yes")
SOCIALACCOUNT_AUTO_SIGNUP = get_bool_from_env("PAPERLESS_SOCIAL_AUTO_SIGNUP")
SOCIALACCOUNT_PROVIDERS = json.loads(os.getenv("PAPERLESS_SOCIALACCOUNT_PROVIDERS", "{}"))
SOCIAL_ACCOUNT_DEFAULT_GROUPS = get_list_from_env("PAPERLESS_SOCIAL_ACCOUNT_DEFAULT_GROUPS")
SOCIAL_ACCOUNT_SYNC_GROUPS = get_bool_from_env("PAPERLESS_SOCIAL_ACCOUNT_SYNC_GROUPS")
SOCIAL_ACCOUNT_SYNC_GROUPS_CLAIM = os.getenv("PAPERLESS_SOCIAL_ACCOUNT_SYNC_GROUPS_CLAIM", "groups")
HEADLESS_TOKEN_STRATEGY = "paperless.adapter.DrfTokenStrategy"
DISABLE_REGULAR_LOGIN = get_bool_from_env("PAPERLESS_DISABLE_REGULAR_LOGIN")
REDIRECT_LOGIN_TO_SSO = get_bool_from_env("PAPERLESS_REDIRECT_LOGIN_TO_SSO")
```

### 5.2 发起社交授权的三个入口

| 场景 | 用户状态 | 入口 | process 参数 | 代码证据 |
|-----|---------|-----|-------------|---------|
| **登录页第三方登录** | 未登录 | `/accounts/login/` 页面按钮 | `process=login`（allauth 视图上下文默认值） | [login.html L61](file:///d:/fz/0601/solo-dogfeeding/code/113-paperless-ngx/src/documents/templates/account/login.html#L61) |
| **已登录用户绑定新账号** | 已登录 | 用户资料对话框「Connect new social account」链接 | `process=connect`（后端显式指定） | [views.py L501](file:///d:/fz/0601/solo-dogfeeding/code/113-paperless-ngx/src/paperless/views.py#L501) |
| **SSO 自动跳转** | 未登录 | 登录页 JS 自动提交第一个社交表单 | `process=login` | [login.html L68-L79](file:///d:/fz/0601/solo-dogfeeding/code/113-paperless-ngx/src/documents/templates/account/login.html#L68-L79) |

#### 5.2.1 登录页社交按钮（process=login）

非 OpenID 提供商渲染为 POST 表单 [login.html L61-L67]：

```django
<form id="social-login" method="POST" action="{{ href }}">
    {% csrf_token %}
    <button type="submit">{{ provider.name }}</button>
</form>
```

OpenID 提供商渲染为 `<a>` 超链接 [login.html L57-L59]：

```django
<a class="btn btn-secondary oidc-url" href="{{ href }}">{{ brand.name }}</a>
```

`REDIRECT_LOGIN_TO_SSO=True` 时自动提交第一个表单或点击第一个 OIDC 链接 [login.html L68-L79]。

#### 5.2.2 已登录用户绑定入口（process=connect）

后端 API：[SocialAccountProvidersView.get](file:///d:/fz/0601/solo-dogfeeding/code/113-paperless-ngx/src/paperless/views.py#L497-L519)

```python
resp = [
    {"name": p.name, "login_url": p.get_login_url(request, process="connect")}
    for p in providers
    if p.id != "openid"
]
# OpenID 提供商额外展开 brands，同样传 process="connect"
```

前端调用：
- [profile.service.ts L46-L50](file:///d:/fz/0601/solo-dogfeeding/code/113-paperless-ngx/src-ui/src/app/services/profile.service.ts#L46-L50)：`GET profile/social_account_providers/`
- [profile-edit-dialog.component.ts L120-L125](file:///d:/fz/0601/solo-dogfeeding/code/113-paperless-ngx/src-ui/src/app/components/common/profile-edit-dialog/profile-edit-dialog.component.ts#L120-L125)：Dialog 打开时并行请求 providers
- [profile-edit-dialog.component.html L87-L98](file:///d:/fz/0601/solo-dogfeeding/code/113-paperless-ngx/src-ui/src/app/components/common/profile-edit-dialog/profile-edit-dialog.component.html#L87-L98)：渲染为 `<a href="{{ provider.login_url }}"` 超链接列表

用户点击超链接即离开 Angular SPA，浏览器整页导航到 Django 社交授权 URL。

### 5.3 社交账号组同步信号

信号注册：[apps.py L18-L20](file:///d:/fz/0601/solo-dogfeeding/code/113-paperless-ngx/src/paperless/apps.py#L18-L20)

```python
from allauth.socialaccount.signals import social_account_updated
social_account_updated.connect(handle_social_account_updated)
```

处理器：[handle_social_account_updated](file:///d:/fz/0601/solo-dogfeeding/code/113-paperless-ngx/src/paperless/signals.py#L35-L63)

- 从 `sociallogin.account.extra_data` 读取组 claim
- 兼容三种结构：直接 `groups` 字段、`userinfo.groups`、`id_token.groups`
- `SOCIAL_ACCOUNT_SYNC_GROUPS=True` 时，`groups.set(groups, clear=True)` 覆盖用户组

该信号在两种情况下被触发：
1. allauth 框架在社交账号更新/绑定时自动发出
2. [CustomSocialAccountAdapter.save_user](file:///d:/fz/0601/solo-dogfeeding/code/113-paperless-ngx/src/paperless/adapter.py#L141) 中手动调用 `handle_social_account_updated(None, request, sociallogin)`

### 5.4 社交登录错误日志

[CustomSocialAccountAdapter.on_authentication_error](file:///d:/fz/0601/solo-dogfeeding/code/113-paperless-ngx/src/paperless/adapter.py#L144-L164) 记录 warning 级别日志。

---

## 六、账号绑定与解绑

### 6.1 数据模型

`SocialAccount`（allauth 提供）通过外键关联 `User`：

```
User
  └── SocialAccount
        - user: ForeignKey(User)
        - provider: str
        - uid: str
        - extra_data: JSON
```

展示层：
- [ProfileSerializer](file:///d:/fz/0601/solo-dogfeeding/code/113-paperless-ngx/src/paperless/serialisers.py#L179-L209) 通过 `social_accounts`（source=`socialaccount_set`）返回已绑定账号列表
- [SocialAccountSerializer](file:///d:/fz/0601/solo-dogfeeding/code/113-paperless-ngx/src/paperless/serialisers.py#L161-L176) 字段：`id`、`provider`、`name`（`obj.get_provider_account().to_str()`）

### 6.2 绑定（Connect）

绑定流程中 paperless-ngx 代码可直接验证的步骤：

1. 已登录用户打开 ProfileEditDialog
2. `ngOnInit()` 并行请求：
   - `ProfileService.get()` → `GET /api/profile/`（取回已绑定 `social_accounts`）[profile-edit-dialog.component.ts L99-L118](file:///d:/fz/0601/solo-dogfeeding/code/113-paperless-ngx/src-ui/src/app/components/common/profile-edit-dialog/profile-edit-dialog.component.ts#L99-L118)
   - `ProfileService.getSocialAccountProviders()` → `GET /api/profile/social_account_providers/`（取回可绑定列表）[profile-edit-dialog.component.ts L120-L125](file:///d:/fz/0601/solo-dogfeeding/code/113-paperless-ngx/src-ui/src/app/components/common/profile-edit-dialog/profile-edit-dialog.component.ts#L120-L125)
3. 后端生成 `process=connect` 的授权 URL [views.py L501](file:///d:/fz/0601/solo-dogfeeding/code/113-paperless-ngx/src/paperless/views.py#L501)
4. 前端渲染 `<a href="{{ provider.login_url }}"` 超链接 [profile-edit-dialog.component.html L91-L94](file:///d:/fz/0601/solo-dogfeeding/code/113-paperless-ngx/src-ui/src/app/components/common/profile-edit-dialog/profile-edit-dialog.component.html#L91-L94)
5. 用户点击 → 浏览器跳转到 Django → allauth 发起 OAuth → 第三方回调 → allauth 处理绑定
6. 绑定完成后调用 `get_connect_redirect_url()` 返回 `reverse("base")`（即 `"/"`）[adapter.py L118-L124](file:///d:/fz/0601/solo-dogfeeding/code/113-paperless-ngx/src/paperless/adapter.py#L118-L124)，浏览器重定向到 SPA 首页

### 6.3 解绑（Disconnect）

**前置条件**：用户必须有可用密码（`has_usable_password=true`），否则解绑按钮禁用并弹出提示 — [profile-edit-dialog.component.html L66-L79](file:///d:/fz/0601/solo-dogfeeding/code/113-paperless-ngx/src-ui/src/app/components/common/profile-edit-dialog/profile-edit-dialog.component.html#L66-L79)。

后端：[DisconnectSocialAccountView.post](file:///d:/fz/0601/solo-dogfeeding/code/113-paperless-ngx/src/paperless/views.py#L464-L480)

```python
def post(self, request, *args, **kwargs):
    user = self.request.user
    try:
        account = user.socialaccount_set.get(pk=request.data["id"])  # 通过 user 过滤，防越权
        account_id = account.id
        account.delete()
        return Response(account_id)
    except SocialAccount.DoesNotExist:
        return HttpResponseBadRequest("Social account not found")
```

前端：
- Service：[profile.service.ts L39-L44](file:///d:/fz/0601/solo-dogfeeding/code/113-paperless-ngx/src-ui/src/app/services/profile.service.ts#L39-L44) → `POST profile/disconnect_social_account/` with `{ id }`
- Component：[profile-edit-dialog.component.ts L245-L260](file:///d:/fz/0601/solo-dogfeeding/code/113-paperless-ngx/src-ui/src/app/components/common/profile-edit-dialog/profile-edit-dialog.component.ts#L245-L260) — 解绑成功后只做本地 `this.socialAccounts = this.socialAccounts.filter((a) => a.id != id)`，**不重新请求任何 API**。

### 6.4 绑定与解绑对比

| 操作 | 是否经 OAuth | 调用方 | 入口 |
|-----|------------|-------|-----|
| **绑定 (Connect)** | 是 | 浏览器（整页导航，离开 SPA） | 1. `GET /api/profile/social_account_providers/` 获取 URL 2. 点击 `<a>` 跳转 → Django → 第三方 → Django → 重定向回 `/` |
| **解绑 (Disconnect)** | 否 | Angular HTTP 客户端（纯 AJAX） | `POST /api/profile/disconnect_social_account/` 带 `{ id }` |

---

## 七、登录失败日志

信号注册：[apps.py L14-L16](file:///d:/fz/0601/solo-dogfeeding/code/113-paperless-ngx/src/paperless/apps.py#L14-L16)

```python
from django.contrib.auth.signals import user_login_failed
user_login_failed.connect(handle_failed_login)
```

处理器：[handle_failed_login](file:///d:/fz/0601/solo-dogfeeding/code/113-paperless-ngx/src/paperless/signals.py#L10-L32) 记录尝试登录的用户名和客户端 IP（通过 `TRUSTED_PROXIES` 正确区分公网/私网）。

---

## 八、核心文件速查

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
| [src-ui/src/app/components/common/profile-edit-dialog/profile-edit-dialog.component.html](file:///d:/fz/0601/solo-dogfeeding/code/113-paperless-ngx/src-ui/src/app/components/common/profile-edit-dialog/profile-edit-dialog.component.html) | 前端资料编辑对话框模板 |
| [src-ui/src/app/data/user-profile.ts](file:///d:/fz/0601/solo-dogfeeding/code/113-paperless-ngx/src-ui/src/app/data/user-profile.ts) | 前端用户资料 TypeScript 接口 |
