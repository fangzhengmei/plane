# OAuth 第三方登录实现深度分析

## 一、架构总览

Plane 的 OAuth 系统采用**适配器模式** + **提供方注册** + **双路径路由**的三层架构：

```
前端触发 → 视图层(双路径) → Provider层 → Adapter层 → 会话建立
     ↓           ↓            ↓          ↓          ↓
useOAuthHook  app/space   github/google  OauthAdapter  user_login
```

## 二、OAuth 提供方注册机制

### 2.1 核心适配器基类

`Adapter` 基类 (`apps/api/plane/authentication/adapter/base.py:35`) 定义了所有认证方式的统一接口：

```python
class Adapter:
    def authenticate(self): raise NotImplementedError
    def get_user_token(self, data, headers=None): raise NotImplementedError
    def get_user_response(self): raise NotImplementedError
    def complete_login_or_signup(self): # 核心登录/注册逻辑
```

### 2.2 OAuth 适配器

`OauthAdapter` (`apps/api/plane/authentication/adapter/oauth.py:24`) 继承自 `Adapter`，封装了标准 OAuth2 流程：

```python
class OauthAdapter(Adapter):
    def authenticate(self):
        self.set_token_data()      # 1. 用 code 换 token
        self.set_user_data()       # 2. 用 token 换用户信息
        return self.complete_login_or_signup()  # 3. 完成登录/注册
```

### 2.3 具体提供方实现

每个 OAuth 提供方独立注册，继承 `OauthAdapter` 并实现特定逻辑：

| 提供方 | 文件 | 特有逻辑 |
|--------|------|----------|
| GitHub | `provider/oauth/github.py` | 组织成员验证、独立邮箱 API 调用 |
| Google | `provider/oauth/google.py` | 标准 OpenID Connect |
| GitLab | `provider/oauth/gitlab.py` | 支持自托管实例 (`GITLAB_HOST`) |
| Gitea | `provider/oauth/gitea.py` | 支持自托管实例 (`GITEA_HOST`) |

**GitHub 提供方注册示例** (`github.py:23`):
```python
class GitHubOAuthProvider(OauthAdapter):
    token_url = "https://github.com/login/oauth/access_token"
    userinfo_url = "https://api.github.com/user"
    scope = "read:user user:email"
    
    def __init__(self, request, code=None, state=None, callback=None):
        # 读取配置
        GITHUB_CLIENT_ID, GITHUB_CLIENT_SECRET, GITHUB_ORGANIZATION_ID = 
            get_configuration_value([...])
        
        # 构造回调 URL
        redirect_uri = f"{scheme}://{host}/auth/github/callback/"
        
        # 构造授权 URL
        auth_url = f"https://github.com/login/oauth/authorize?{urlencode(params)}"
        
        super().__init__(...)
```

## 三、实例级开关机制

### 3.1 配置分层

系统支持两种配置模式，由 `SKIP_ENV_VAR` 控制：

| 模式 | SKIP_ENV_VAR | 配置来源 | 适用场景 |
|------|-------------|----------|----------|
| 环境变量模式 | False | `os.environ` | Docker 部署、快速启动 |
| 数据库配置模式 | True | `InstanceConfiguration` 表 | 企业部署、需要热更新 |

**配置读取逻辑** (`license/utils/instance_value.py:17`):
```python
def get_configuration_value(keys):
    if settings.SKIP_ENV_VAR:
        # 从数据库 InstanceConfiguration 表读取
        instance_configuration = InstanceConfiguration.objects.values(...)
    else:
        # 从环境变量读取
        for key in keys:
            environment_list.append(os.environ.get(key.get("key"), key.get("default")))
```

### 3.2 认证开关配置

`TInstanceAuthenticationMethodKeys` 类型定义了所有认证方式的开关 (`packages/types/src/instance/auth.ts:27`):

```typescript
export type TInstanceAuthenticationMethodKeys =
  | "ENABLE_SIGNUP"           // 注册开关
  | "ENABLE_MAGIC_LINK_LOGIN" // Magic Link 登录
  | "ENABLE_EMAIL_PASSWORD"   // 邮箱密码登录
  | "IS_GOOGLE_ENABLED"       // Google OAuth
  | "IS_GITHUB_ENABLED"       // GitHub OAuth
  | "IS_GITLAB_ENABLED"       // GitLab OAuth
  | "IS_GITEA_ENABLED";       // Gitea OAuth
```

### 3.3 OAuth 配置键

每个 OAuth 提供方有独立的配置键：

```typescript
// Google
| "GOOGLE_CLIENT_ID" | "GOOGLE_CLIENT_SECRET" | "ENABLE_GOOGLE_SYNC"

// GitHub
| "GITHUB_CLIENT_ID" | "GITHUB_CLIENT_SECRET" 
| "GITHUB_ORGANIZATION_ID" | "ENABLE_GITHUB_SYNC"

// GitLab
| "GITLAB_HOST" | "GITLAB_CLIENT_ID" 
| "GITLAB_CLIENT_SECRET" | "ENABLE_GITLAB_SYNC"

// Gitea
| "GITEA_HOST" | "GITEA_CLIENT_ID" 
| "GITEA_CLIENT_SECRET" | "ENABLE_GITEA_SYNC"
```

### 3.4 前端开关控制

前端通过 `useOAuthConfig` Hook 读取实例配置并动态渲染 OAuth 按钮：

**Hook 组合逻辑** (`apps/web/core/hooks/oauth/index.ts:13`):
```typescript
export const useOAuthConfig = (oauthActionText: string = "Continue"): TOAuthConfigs => {
  const coreOAuthConfig = useCoreOAuthConfig(oauthActionText);
  const extendedOAuthConfig = useExtendedOAuthConfig(oAuthActionText);
  return {
    isOAuthEnabled: coreOAuthConfig.isOAuthEnabled || extendedOAuthConfig.isOAuthEnabled,
    oAuthOptions: [...coreOAuthConfig.oAuthOptions, ...extendedOAuthConfig.oAuthOptions],
  };
};
```

**核心配置读取** (`apps/web/core/hooks/oauth/core.tsx:21`):
```typescript
export const useCoreOAuthConfig = (oauthActionText: string): TOAuthConfigs => {
  const { config } = useInstance();
  
  const isOAuthEnabled =
    (config &&
      (config?.is_google_enabled ||
        config?.is_github_enabled ||
        config?.is_gitlab_enabled ||
        config?.is_gitea_enabled)) || false;

  const oAuthOptions: TOAuthOption[] = [
    {
      id: "google",
      text: `${oauthActionText} with Google`,
      onClick: () => {
        // 注意：前端永远调用不带 spaces 前缀的 App 模式路由
        window.location.assign(`${API_BASE_URL}/auth/google/${next_path ? `?next_path=${next_path}` : ``}`);
      },
      enabled: config?.is_google_enabled,
    },
    // ... 其他提供方
  ];
};
```

### 3.5 注册开关检查

`ENABLE_SIGNUP` 开关在用户首次登录（注册场景）时生效，检查逻辑位于 `Adapter.__check_signup` (`adapter/base.py:102`):

```python
def __check_signup(self, email):
    """Check if sign up is enabled or not and raise exception if not enabled"""
    (ENABLE_SIGNUP,) = get_configuration_value([
        {"key": "ENABLE_SIGNUP", "default": os.environ.get("ENABLE_SIGNUP", "1")}
    ])

    # 注册关闭时，仅允许已有邀请的用户注册
    if ENABLE_SIGNUP == "0" and not WorkspaceMemberInvite.objects.filter(email=email).exists():
        raise AuthenticationException(error_code=AUTHENTICATION_ERROR_CODES["SIGNUP_DISABLED"], ...)

    return True
```

该方法在 `complete_login_or_signup` 中被调用，当用户不存在（新注册）时触发检查。

## 四、双部署形态路径差异

Plane 支持两种部署形态：**App 模式**（主应用单租户）和 **Space 模式**（发布页面/多租户），对应两套独立的 OAuth 路由。

### 4.1 URL 路由差异

`authentication/urls.py` 中定义了两套路径：

```python
# ===== App 模式路径（主应用使用）=====
path("google/", GoogleOauthInitiateEndpoint.as_view()),
path("google/callback/", GoogleCallbackEndpoint.as_view()),
path("github/", GitHubOauthInitiateEndpoint.as_view()),
path("github/callback/", GitHubCallbackEndpoint.as_view()),
# ... 其他提供方

# ===== Space 模式路径（发布页面使用）=====
path("spaces/google/", GoogleOauthInitiateSpaceEndpoint.as_view()),
path("spaces/google/callback/", GoogleCallbackSpaceEndpoint.as_view()),
path("spaces/github/", GitHubOauthInitiateSpaceEndpoint.as_view()),
path("spaces/github/callback/", GitHubCallbackSpaceEndpoint.as_view()),
# ... 其他提供方
```

### 4.2 前端入口与路由对应关系

**关键事实：前端 Web 应用的登录/注册页面只调用 App 模式路由**

- `useCoreOAuthConfig` 生成的 onClick 永远指向 `/auth/google/`、`/auth/github/` 等不带 `spaces/` 前缀的路径
- Space 模式路由 `/auth/spaces/*` 不被前端登录页调用，主要用于：
  - 已发布项目页面（publish-project）的访问控制
  - 外部用户通过公开链接访问时的认证

**发布页面使用 Space 模式示例** (`publish-project/modal.tsx:167`):
```typescript
const SPACE_APP_URL = (SPACE_BASE_URL.trim() === "" ? window.location.origin : SPACE_BASE_URL) + SPACE_BASE_PATH;
const publishLink = `${SPACE_APP_URL}/issues/${projectPublishSettings?.anchor}`;
```

### 4.3 视图层差异对比

| 维度 | App 模式 (`views/app/github.py`) | Space 模式 (`views/space/github.py`) |
|------|---------------------------------|--------------------------------------|
| 路由前缀 | `/auth/github/` | `/auth/spaces/github/` |
| next_path 存储 | initiate 时存入 session | **initiate 时读取但不存入 session** |
| Host 存储 | `request.session["host"] = base_host(is_app=True)` | `request.session["host"] = base_host(is_space=True)` |
| 登录调用 | `user_login(is_app=True)` | `user_login(is_space=True)` |
| 回调处理 | `callback=post_user_auth_workflow` | 无 callback |
| 重定向逻辑 | 优先 next_path，否则 `get_redirection_path(user)` | `validate_next_path(next_path)`（但 next_path 永远为 None） |
| 跳转目标 | 用户 workspace 或 onboarding | 固定返回 space base URL |

**App 模式 Initiate** (`views/app/github.py:26`):
```python
class GitHubOauthInitiateEndpoint(View):
    def get(self, request):
        request.session["host"] = base_host(request=request, is_app=True)
        next_path = request.GET.get("next_path")
        if next_path:
            request.session["next_path"] = str(next_path)  # ✅ App 模式存入 session
        # ... 生成 state，重定向到 GitHub
```

**App 模式 Callback** (`views/app/github.py:60`):
```python
class GitHubCallbackEndpoint(View):
    def get(self, request):
        code = request.GET.get("code")
        state = request.GET.get("state")
        next_path = request.session.get("next_path")  # ✅ 能取到 initiate 时存入的值
        
        provider = GitHubOAuthProvider(
            request=request, 
            code=code, 
            callback=post_user_auth_workflow  # App 模式有后处理
        )
        user = provider.authenticate()
        user_login(request=request, user=user, is_app=True)
        
        # 优先 next_path，否则自动导向用户 workspace
        if next_path:
            path = next_path
        else:
            path = get_redirection_path(user=user)
        return HttpResponseRedirect(url)
```

**Space 模式 Initiate** (`views/space/github.py:25`):
```python
class GitHubOauthInitiateSpaceEndpoint(View):
    def get(self, request):
        request.session["host"] = base_host(request=request, is_space=True)
        next_path = request.GET.get("next_path")  # 读取了但...
        # ❌ Space 模式没有存入 session 的逻辑！
        # ... 生成 state，重定向到 GitHub
```

**Space 模式 Callback** (`views/space/github.py:57`):
```python
class GitHubCallbackSpaceEndpoint(View):
    def get(self, request):
        code = request.GET.get("code")
        state = request.GET.get("state")
        next_path = request.session.get("next_path")  # ❌ 永远为 None，因为 initiate 没存
        
        provider = GitHubOAuthProvider(request=request, code=code)  # 无 callback
        user = provider.authenticate()
        user_login(request=request, user=user, is_space=True)
        
        next_path = validate_next_path(next_path=next_path)  # 验证 None
        # 永远跳转到 space base URL
        url = f"{base_host(request=request, is_space=True).rstrip('/')}{next_path}"
        return HttpResponseRedirect(url)
```

### 4.4 Host 计算参数语义

`base_host` 函数的三个布尔参数是**互斥**的，按优先级判断：`is_admin` > `is_space` > `is_app` (`authentication/utils/host.py:16`):

```python
def base_host(
    request: Request | HttpRequest,
    is_admin: bool = False,    # 最高优先级：管理后台
    is_space: bool = False,    # 次高优先级：发布空间
    is_app: bool = False,      # 最低优先级：主应用
) -> str:
    base_origin = settings.WEB_URL or settings.APP_BASE_URL
    
    if is_admin:
        # 返回 Admin URL + /god-mode/ 路径
        return settings.ADMIN_BASE_URL or (base_origin + "/god-mode/")
    
    if is_space:
        # 返回 Space URL + /spaces/ 路径
        return settings.SPACE_BASE_URL or (base_origin + "/spaces/")
    
    if is_app:
        # 返回 App 基础 URL
        return settings.APP_BASE_URL or base_origin
    
    return base_origin
```

**注意**：不是同时传多个参数，而是根据场景只传一个为 `True`。

## 五、会话建立接力步骤

### 5.1 完整 OAuth 登录流程（App 模式）

```
1. 前端点击 "Sign in with GitHub"
   ↓ (window.location.assign)
2. GET /auth/github/?next_path=/some-path
   → GitHubOauthInitiateEndpoint
   ├─ 存入 session: host, next_path
   ├─ 生成 state 存入 session
   ├─ 构造 GitHub 授权 URL（带 state 和 redirect_uri）
   └─ 302 重定向到 GitHub
   ↓ (用户在 GitHub 授权)
3. GitHub 回调 GET /auth/github/callback/?code=xxx&state=xxx
   → GitHubCallbackEndpoint
   ├─ 验证 state 与 session 中匹配
   ├─ 用 code 向 GitHub 换 access_token
   ├─ 用 access_token 调 /user 和 /user/emails 取用户信息
   ├─ 调用 complete_login_or_signup():
   │  ├─ 查 User 表，不存在则创建（检查 ENABLE_SIGNUP）
   │  ├─ 调用 post_user_auth_workflow() 处理邀请
   │  ├─ 创建/更新 Account 表记录
   │  └─ 更新用户 last_login 等信息
   ├─ 调用 user_login() 建立 Django 会话
   └─ 302 重定向到 next_path 或用户 workspace
```

### 5.2 登录与会话建立

`user_login` 函数 (`authentication/utils/login.py:14`) 负责建立 Django 会话并记录设备信息：

```python
def user_login(request, user, is_app=False, is_admin=False, is_space=False):
    login(request=request, user=user)  # Django 标准登录，设置 session cookie
    
    # Admin 会话有独立过期时间
    if is_admin:
        request.session.set_expiry(settings.ADMIN_SESSION_COOKIE_AGE)
    
    # 记录设备信息到 session
    device_info = {
        "user_agent": request.META.get("HTTP_USER_AGENT", ""),
        "ip_address": get_client_ip(request=request),
        "domain": base_host(request, is_app, is_admin, is_space),
    }
    request.session["device_info"] = device_info
    request.session.save()
```

### 5.3 Session 中间件与双 Cookie 机制

`SessionMiddleware` (`authentication/middleware/session.py:16`) 支持 Admin 和普通用户两套独立的 session cookie：

```python
class SessionMiddleware(MiddlewareMixin):
    def process_request(self, request):
        # Admin 路径使用独立的 session cookie
        if "instances" in request.path:
            session_key = request.COOKIES.get(settings.ADMIN_SESSION_COOKIE_NAME)
        else:
            session_key = request.COOKIES.get(settings.SESSION_COOKIE_NAME)
        request.session = self.SessionStore(session_key)
    
    def process_response(self, request, response):
        is_admin_path = "instances" in request.path
        cookie_name = settings.ADMIN_SESSION_COOKIE_NAME if is_admin_path else settings.SESSION_COOKIE_NAME
        
        # 根据路径设置不同的 cookie
        if modified or settings.SESSION_SAVE_EVERY_REQUEST:
            request.session.save()
            response.set_cookie(
                cookie_name,
                request.session.session_key,
                max_age=settings.ADMIN_SESSION_COOKIE_AGE if is_admin_path else request.session.get_expiry_age(),
                # ... 其他 cookie 属性
            )
```

### 5.4 登录后回调衔接

App 模式在认证成功后会调用 `post_user_auth_workflow` callback，用于处理工作区邀请：

**callback 注册** (`views/app/github.py:89`):
```python
provider = GitHubOAuthProvider(request=request, code=code, callback=post_user_auth_workflow)
```

**callback 执行时机** (`adapter/base.py:351-353`):
```python
def complete_login_or_signup(self):
    # ... 用户创建/登录完成后
    if self.callback:
        self.callback(user, is_signup, self.request)
```

**callback 实现** (`utils/user_auth_workflow.py:8`):
```python
def post_user_auth_workflow(user, is_signup, request):
    process_workspace_project_invitations(user=user)
```

该回调会处理用户邮箱匹配的工作区/项目邀请，自动将用户加入对应工作区。

**注意**：Space 模式没有注册 callback，因此不会自动处理邀请。

### 5.5 用户数据同步

当 `ENABLE_*_SYNC` 开启时，每次登录会同步用户数据 (`adapter/base.py:256`):

```python
def sync_user_data(self, user):
    # 更新姓名、显示名
    user.first_name = self.user_data.get("user", {}).get("first_name", "")
    user.display_name = display_name
    
    # 下载并更新头像
    avatar = self.user_data.get("user", {}).get("avatar", "")
    avatar_asset = self.download_and_upload_avatar(avatar_url=avatar, user=user)
    if avatar_asset:
        user.avatar_asset = avatar_asset
    
    user.save()
    return user
```

同步逻辑仅在非首次登录（`not is_signup`）且同步开关开启时执行。

## 六、关键代码位置速查表

| 功能 | 文件路径 |
|------|----------|
| OAuth 适配器基类 | `apps/api/plane/authentication/adapter/oauth.py` |
| 认证适配器基类（含注册开关检查） | `apps/api/plane/authentication/adapter/base.py` |
| GitHub 提供方 | `apps/api/plane/authentication/provider/oauth/github.py` |
| Google 提供方 | `apps/api/plane/authentication/provider/oauth/google.py` |
| App 模式视图层 | `apps/api/plane/authentication/views/app/*.py` |
| Space 模式视图层 | `apps/api/plane/authentication/views/space/*.py` |
| URL 路由 | `apps/api/plane/authentication/urls.py` |
| 会话中间件 | `apps/api/plane/authentication/middleware/session.py` |
| 登录工具 | `apps/api/plane/authentication/utils/login.py` |
| Host 计算工具 | `apps/api/plane/authentication/utils/host.py` |
| 登录后回调 | `apps/api/plane/authentication/utils/user_auth_workflow.py` |
| 重定向路径计算 | `apps/api/plane/authentication/utils/redirection_path.py` |
| 配置读取 | `apps/api/plane/license/utils/instance_value.py` |
| 实例配置模型 | `apps/api/plane/license/models/instance.py` |
| 前端 OAuth Hook 入口 | `apps/web/core/hooks/oauth/index.ts` |
| 前端核心 OAuth 配置 | `apps/web/core/hooks/oauth/core.tsx` |
| 认证类型定义 | `packages/types/src/instance/auth.ts` |
| OAuth UI 组件 | `packages/ui/src/oauth/oauth-options.tsx` |
