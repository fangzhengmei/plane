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

前端通过 `useCoreOAuthConfig` Hook 读取实例配置并动态渲染 OAuth 按钮 (`apps/web/core/hooks/oauth/core.tsx:21`):

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
        window.location.assign(`${API_BASE_URL}/auth/google/`);
      },
      enabled: config?.is_google_enabled,
    },
    // ... 其他提供方
  ];
};
```

## 四、双部署形态路径差异

Plane 支持两种部署形态：**App 模式**（单租户）和 **Space 模式**（多租户），对应两套独立的 OAuth 路由。

### 4.1 URL 路由差异

`authentication/urls.py` 中定义了两套路径：

```python
# ===== App 模式路径 =====
path("google/", GoogleOauthInitiateEndpoint.as_view()),
path("google/callback/", GoogleCallbackEndpoint.as_view()),
path("github/", GitHubOauthInitiateEndpoint.as_view()),
path("github/callback/", GitHubCallbackEndpoint.as_view()),
# ...

# ===== Space 模式路径 =====
path("spaces/google/", GoogleOauthInitiateSpaceEndpoint.as_view()),
path("spaces/google/callback/", GoogleCallbackSpaceEndpoint.as_view()),
path("spaces/github/", GitHubOauthInitiateSpaceEndpoint.as_view()),
path("spaces/github/callback/", GitHubCallbackSpaceEndpoint.as_view()),
# ...
```

### 4.2 视图层差异对比

| 维度 | App 模式 (`views/app/github.py`) | Space 模式 (`views/space/github.py`) |
|------|---------------------------------|--------------------------------------|
| 重定向 Host | `base_host(is_app=True)` | `base_host(is_space=True)` |
| Host 存储 | `request.session["host"]` | `request.session["host"]` |
| 登录调用 | `user_login(is_app=True)` | `user_login(is_space=True)` |
| 回调处理 | `callback=post_user_auth_workflow` | 无 callback |
| 重定向逻辑 | `get_redirection_path(user)` | `validate_next_path(next_path)` |
| 跳转路径 | `/` 或 workspace | 保持 `next_path` |

**App 模式回调** (`views/app/github.py:60`):
```python
class GitHubCallbackEndpoint(View):
    def get(self, request):
        # ... 验证 state 和 code
        provider = GitHubOAuthProvider(
            request=request, 
            code=code, 
            callback=post_user_auth_workflow  # App 模式有后处理
        )
        user = provider.authenticate()
        user_login(request=request, user=user, is_app=True)
        # 自动重定向到用户 workspace
        path = get_redirection_path(user=user)
        return HttpResponseRedirect(url)
```

**Space 模式回调** (`views/space/github.py:57`):
```python
class GitHubCallbackSpaceEndpoint(View):
    def get(self, request):
        # ... 验证 state 和 code
        provider = GitHubOAuthProvider(request=request, code=code)  # 无 callback
        user = provider.authenticate()
        user_login(request=request, user=user, is_space=True)
        # 保持原始 next_path
        next_path = validate_next_path(next_path=next_path)
        url = f"{base_host(...).rstrip('/')}{next_path}"
        return HttpResponseRedirect(url)
```

### 4.3 Host 解析逻辑

`base_host` 函数根据部署形态返回不同的基础 URL (`authentication/utils/host.py:16`):

```python
def base_host(request, is_admin=False, is_space=False, is_app=False):
    base_origin = settings.WEB_URL or settings.APP_BASE_URL
    
    if is_admin:
        return settings.ADMIN_BASE_URL or (base_origin + "/god-mode/")
    
    if is_space:
        return settings.SPACE_BASE_URL or (base_origin + "/spaces/")
    
    if is_app:
        return settings.APP_BASE_URL or base_origin
    
    return base_origin
```

## 五、会话建立接力步骤

### 5.1 完整 OAuth 登录流程

```
1. 前端点击 OAuth 按钮
   ↓ (window.location.assign)
2. 访问 /auth/github/  → GitHubOauthInitiateEndpoint
   ├─ 生成 state 存入 session
   ├─ 构造 GitHub 授权 URL
   └─ 重定向到 GitHub
   ↓ (用户在 GitHub 授权)
3. GitHub 回调到 /auth/github/callback/
   ↓
4. GitHubCallbackEndpoint
   ├─ 验证 state 匹配
   ├─ 用 code 换取 access_token
   ├─ 用 access_token 获取用户信息
   ├─ 创建/更新 User 和 Account
   ├─ 调用 user_login() 建立会话
   └─ 重定向到目标页面
```

### 5.2 登录与会话建立

`user_login` 函数 (`authentication/utils/login.py:14`) 负责建立 Django 会话：

```python
def user_login(request, user, is_app=False, is_admin=False, is_space=False):
    login(request=request, user=user)  # Django 标准登录
    
    # Admin 会话有独立过期时间
    if is_admin:
        request.session.set_expiry(settings.ADMIN_SESSION_COOKIE_AGE)
    
    # 记录设备信息
    device_info = {
        "user_agent": request.META.get("HTTP_USER_AGENT", ""),
        "ip_address": get_client_ip(request=request),
        "domain": base_host(request, is_app, is_admin, is_space),
    }
    request.session["device_info"] = device_info
    request.session.save()
```

### 5.3 Session 中间件

`SessionMiddleware` (`authentication/middleware/session.py:16`) 支持双 Cookie 机制：

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
        # 根据路径设置不同的 cookie
        is_admin_path = "instances" in request.path
        cookie_name = settings.ADMIN_SESSION_COOKIE_NAME if is_admin_path else settings.SESSION_COOKIE_NAME
        # ... 设置 cookie
```

### 5.4 用户数据同步

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

## 六、关键代码位置速查表

| 功能 | 文件路径 |
|------|----------|
| OAuth 适配器基类 | `apps/api/plane/authentication/adapter/oauth.py` |
| 认证适配器基类 | `apps/api/plane/authentication/adapter/base.py` |
| GitHub 提供方 | `apps/api/plane/authentication/provider/oauth/github.py` |
| Google 提供方 | `apps/api/plane/authentication/provider/oauth/google.py` |
| App 视图层 | `apps/api/plane/authentication/views/app/*.py` |
| Space 视图层 | `apps/api/plane/authentication/views/space/*.py` |
| URL 路由 | `apps/api/plane/authentication/urls.py` |
| 会话中间件 | `apps/api/plane/authentication/middleware/session.py` |
| 登录工具 | `apps/api/plane/authentication/utils/login.py` |
| Host 工具 | `apps/api/plane/authentication/utils/host.py` |
| 配置读取 | `apps/api/plane/license/utils/instance_value.py` |
| 实例配置模型 | `apps/api/plane/license/models/instance.py` |
| 前端 OAuth Hook | `apps/web/core/hooks/oauth/core.tsx` |
| 认证类型定义 | `packages/types/src/instance/auth.ts` |
| OAuth UI 组件 | `packages/ui/src/oauth/oauth-options.tsx` |
