# Plane 邮件配置与通知链路分析

## 1. 整体架构概览

Plane 的邮件系统由以下四个核心层组成：

```
管理员配置 (Admin UI)
    ↓ PATCH /api/instances/configurations/
InstanceConfiguration 模型 (DB 持久化 + 缓存)
    ↓ get_email_configuration() 每次调用时实时读取
Celery 异步任务 (bgtasks/*)
    ↓ get_connection() + EmailMultiAlternatives
Django SMTP Backend → 企业邮件网关
```

**关键设计原则**：邮件配置不驻留在 Django `settings` 对象中，而是在每次发送邮件时从数据库实时读取，实现了"配置即生效"的热加载语义。

---

## 2. 配置热加载的边界

### 2.1 配置存储模型

所有 SMTP 配置存储在 [InstanceConfiguration](file:///d:/fz/0508-3/solo-dogfeeding/code/198-plane/apps/api/plane/license/models/instance.py#L72-L84) 模型中，表名为 `instance_configurations`，核心字段：

| 字段 | 说明 |
|------|------|
| `key` | 配置项名称（如 `EMAIL_HOST`），`unique=True` |
| `value` | 配置值（`TextField`），可为空 |
| `category` | 分类标签（如 `SMTP`） |
| `is_encrypted` | 是否加密存储 |

SMTP 相关的 8 个配置项定义在 [smtp_config_variables](file:///d:/fz/0508-3/solo-dogfeeding/code/198-plane/apps/api/plane/utils/instance_config_variables/core.py#L147-L196)：

| Key | 默认值 | 加密 |
|-----|--------|------|
| `ENABLE_SMTP` | `"0"` | 否 |
| `EMAIL_HOST` | `""` | 否 |
| `EMAIL_HOST_USER` | `""` | 否 |
| `EMAIL_HOST_PASSWORD` | `""` | **是** |
| `EMAIL_PORT` | `"587"` | 否 |
| `EMAIL_FROM` | `""` | 否 |
| `EMAIL_USE_TLS` | `"1"` | 否 |
| `EMAIL_USE_SSL` | `"0"` | 否 |

### 2.2 双模配置读取：SKIP_ENV_VAR 开关

[get_configuration_value](file:///d:/fz/0508-3/solo-dogfeeding/code/198-plane/apps/api/plane/license/utils/instance_value.py#L17-L39) 是配置读取的核心函数，由 `settings.SKIP_ENV_VAR` 控制两种模式：

- **`SKIP_ENV_VAR = True`（默认）**：从 `InstanceConfiguration` 数据库表读取，支持加密字段自动解密。管理员在 Admin UI 修改后立即生效。
- **`SKIP_ENV_VAR = False`**：从 `os.environ` 环境变量读取，与管理员 UI 无关，适合纯环境变量部署场景。

`SKIP_ENV_VAR` 本身在 [common.py](file:///d:/fz/0508-3/solo-dogfeeding/code/198-plane/apps/api/plane/settings/common.py#L349) 中定义为：

```python
SKIP_ENV_VAR = os.environ.get("SKIP_ENV_VAR", "1") == "1"
```

默认值为 `"1"`，即默认从数据库读取配置。

### 2.3 配置写入与缓存失效

管理员通过 Admin UI 提交配置变更时：

1. **前端**：[InstanceEmailForm](file:///d:/fz/0508-3/solo-dogfeeding/code/198-plane/apps/admin/app/(all)/(dashboard)/email/email-config-form.tsx#L110-L122) 调用 `updateInstanceConfigurations(payload)`，其中 `ENABLE_SMTP` 会被强制设为 `"1"`。

2. **后端 API**：[InstanceConfigurationEndpoint.patch](file:///d:/fz/0508-3/solo-dogfeeding/code/198-plane/apps/api/plane/license/api/views/configuration.py#L41-L58) 处理 PATCH 请求：
   - 遍历请求中的 key，查找对应的 `InstanceConfiguration` 记录
   - 若 `is_encrypted=True`，调用 `encrypt_data()` 用 Fernet 加密后存入
   - 执行 `bulk_update` 批量更新
   - `@invalidate_cache` 装饰器清除 `/api/instances/configurations/` 和 `/api/instances/` 的缓存（缓存有效期 2 小时，定义于 `@cache_response(60 * 60 * 2)`）

3. **加密机制**：[encryption.py](file:///d:/fz/0508-3/solo-dogfeeding/code/198-plane/apps/api/plane/license/utils/encryption.py#L20-L30) 使用 PBKDF2 + Fernet 对称加密，密钥派生自 `settings.SECRET_KEY`。

### 2.4 热加载边界总结

| 维度 | 说明 |
|------|------|
| **生效时机** | 每次 Celery 任务调用 `get_email_configuration()` 时从 DB 实时读取，无需重启服务 |
| **不涉及的层** | Django `settings.EMAIL_BACKEND` 在 [common.py#L277](file:///d:/fz/0508-3/solo-dogfeeding/code/198-plane/apps/api/plane/settings/common.py#L277) 中硬编码为 `"django.core.mail.backends.smtp.EmailBackend"`，不可热修改 |
| **缓存影响** | GET 请求的配置有 2 小时 Redis 缓存，但写入时自动失效；邮件发送不走缓存，直接读 DB |
| **禁用邮件** | [DisableEmailFeatureEndpoint](file:///d:/fz/0508-3/solo-dogfeeding/code/198-plane/apps/api/plane/license/api/views/configuration.py#L62-L85) 的 DELETE 方法会将 `ENABLE_SMTP` 设为 `"0"`，其余 SMTP 字段清空 |

---

## 3. 模板渲染的上下文

### 3.1 模板文件清单

所有邮件模板位于 `apps/api/templates/emails/` 目录：

| 类别 | 模板路径 | 用途 |
|------|----------|------|
| 认证 | `emails/auth/forgot_password.html` | 重置密码 |
| 认证 | `emails/auth/magic_signin.html` | Magic Link 登录码 |
| 邀请 | `emails/invitations/workspace_invitation.html` | 工作区邀请 |
| 邀请 | `emails/invitations/project_invitation.html` | 项目邀请 |
| 通知 | `emails/notifications/issue-updates.html` | Issue 变更通知 |
| 通知 | `emails/notifications/project_addition.html` | 添加到项目通知 |
| 通知 | `emails/notifications/webhook-deactivate.html` | Webhook 停用通知 |
| 用户 | `emails/user/user_activation.html` | 账号激活 |
| 用户 | `emails/user/user_deactivation.html` | 账号停用 |
| 用户 | `emails/user/email_updated.html` | 邮箱变更确认 |
| 导出 | `emails/exports/analytics.html` | 分析数据导出 |
| 测试 | `emails/test_email.html` | 测试邮件 |

### 3.2 各模板渲染上下文

#### 认证类

**forgot_password** — [forgot_password_task](file:///d:/fz/0508-3/solo-dogfeeding/code/198-plane/apps/api/plane/bgtasks/forgot_password_task.py#L23-L69)

```python
context = {
    "first_name": first_name,
    "forgot_password_url": abs_url,  # {current_site}/accounts/reset-password/?uidb64=...&token=...&email=...
    "email": email,
}
```

**magic_link** — [magic_link_code_task](file:///d:/fz/0508-3/solo-dogfeeding/code/198-plane/apps/api/plane/bgtasks/magic_link_code_task.py#L23-L61)

```python
context = {
    "code": token,   # 一次性登录码
    "email": email,
}
```

**send_email_update_magic_code** — [user_email_update_task](file:///d:/fz/0508-3/solo-dogfeeding/code/198-plane/apps/api/plane/bgtasks/user_email_update_task.py#L22-L63)

```python
context = {"code": token, "email": email}  # 复用 magic_signin.html 模板
```

#### 邀请类

**workspace_invitation** — [workspace_invitation_task](file:///d:/fz/0508-3/solo-dogfeeding/code/198-plane/apps/api/plane/bgtasks/workspace_invitation_task.py#L23-L88)

```python
context = {
    "email": email,
    "first_name": user.first_name or user.display_name or user.email,
    "workspace_name": workspace.name,
    "abs_url": abs_url,  # {current_site}/workspace-invitations/?invitation_id=...&slug=...&token=...
}
```

**project_invitation** — [project_invitation_task](file:///d:/fz/0508-3/solo-dogfeeding/code/198-plane/apps/api/plane/bgtasks/project_invitation_task.py#L24-L83)

```python
context = {
    "email": email,
    "first_name": user.first_name,
    "project_name": project.name,
    "invitation_url": abs_url,
    "current_site": current_site,
}
```

#### 通知类（最复杂）

**issue-updates** — [send_email_notification](file:///d:/fz/0508-3/solo-dogfeeding/code/198-plane/apps/api/plane/bgtasks/email_notification_task.py#L152-L306)

```python
context = {
    "data": template_data,        # [{actor_detail, changes, issue_details, activity_time}, ...]
    "summary": "Updates were made to the issue by",
    "actors_involved": len(set(actors_involved)),
    "issue": {
        "issue_identifier": "PROJ-123",
        "name": issue.name,
        "issue_url": "{base_api}/{workspace_slug}/projects/{project_id}/issues/{issue_id}",
    },
    "receiver": {"email": receiver.email},
    "issue_url": "...",
    "project_url": "...",
    "workspace": str(issue.project.workspace.slug),
    "project": str(issue.project.name),
    "user_preference": "{base_api}/{workspace_slug}/settings/account/notifications/",
    "comments": [{"actor_comments": ..., "actor_detail": ...}, ...],
    "entity_type": "issue",
}
```

其中 `template_data` 的结构通过 [create_payload](file:///d:/fz/0508-3/solo-dogfeeding/code/198-plane/apps/api/plane/bgtasks/email_notification_task.py#L87-L127) 从原始 `notification_data` 转换而来：

```python
# 输入格式: {"actor_id": [ { data }, { data } ]}
# 输出格式: {"actor_id": { "field": { "old_value": [...], "new_value": [...] } }}
```

**project_addition** — [project_add_user_email_task](file:///d:/fz/0508-3/solo-dogfeeding/code/198-plane/apps/api/plane/bgtasks/project_add_user_email_task.py#L25-L88)

```python
context = {
    "project_name": project_name,
    "workspace_name": workspace_name,
    "email": member_email,
    "inviter_first_name": inviter_first_name,
    "project_url": project_url,
}
```

**webhook-deactivate** — [webhook_task](file:///d:/fz/0508-3/solo-dogfeeding/code/198-plane/apps/api/plane/bgtasks/webhook_task.py#L191-L251)

```python
context = {
    "email": receiver.email,
    "message": "Webhook {url} has been deactivated due to failed requests.",
    "webhook_url": "{current_site}/{workspace_slug}/settings/webhooks/{webhook_id}",
}
```

#### 用户管理类

**user_activation / user_deactivation** — 上下文分别为 `{"email", "profile_url"}` 和 `{"email", "login_url"}`

**email_updated** — 上下文为 `{"email": email}`

#### 导出类

**analytics** — [analytic_plot_export](file:///d:/fz/0508-3/solo-dogfeeding/code/198-plane/apps/api/plane/bgtasks/analytic_plot_export.py#L52-L88)，上下文为空 `{}`，CSV 文件作为附件发送。

### 3.3 纯文本生成

所有 HTML 邮件都会通过 [generate_plain_text_from_html](file:///d:/fz/0508-3/solo-dogfeeding/code/198-plane/apps/api/plane/utils/email.py#L19-L41) 生成对应的纯文本版本，作为 `EmailMultiAlternatives` 的 `body`，HTML 作为 `attach_alternative`。处理步骤：

1. 移除 `<style>` 标签及其内容
2. `strip_tags()` 去除所有 HTML 标签
3. 压缩多余空行

### 3.4 Mention 组件处理

Issue 通知中的评论/提及内容可能包含 `<mention-component entity_identifier="..." />` 自定义 HTML 标签。[process_mention](file:///d:/fz/0508-3/solo-dogfeeding/code/198-plane/apps/api/plane/bgtasks/email_notification_task.py#L130-L139) 使用 BeautifulSoup 将其替换为 `@用户名` 格式的纯文本。

---

## 4. 发送失败的重试与回看机制

### 4.1 邮件发送的两种模式

Plane 的邮件发送分为两种截然不同的模式：

#### 模式 A：即时发送（Fire-and-forget）

适用于认证邮件、邀请邮件、用户管理邮件等。调用链路：

```
视图层 → task.delay() → Celery Worker → get_email_configuration() → get_connection() → msg.send()
```

这些任务（如 `forgot_password`、`workspace_invitation`、`magic_link` 等）的特征：
- 使用 `@shared_task` 装饰，无 `autoretry_for` / `max_retries` 配置
- **没有自动重试机制**：发送失败时 `except Exception` 捕获后调用 `log_exception(e)` 然后静默返回
- **没有发送结果记录**：不写入任何数据库表来追踪发送状态

#### 模式 B：批量聚合 + 日志记录（Issue 通知）

适用于 Issue 变更通知，有两阶段处理：

**阶段 1：通知产生** — [notification_task.notifications](file:///d:/fz/0508-3/solo-dogfeeding/code/198-plane/apps/api/plane/bgtasks/notification_task.py#L191-L674)

当 Issue 发生活动时：
1. 创建 `Notification` 记录（应用内通知）
2. 检查 `UserNotificationPreference` 决定是否发送邮件
3. 创建 `EmailNotificationLog` 记录（`processed_at=None`，`sent_at=None`）

**阶段 2：邮件聚合与发送** — [stack_email_notification](file:///d:/fz/0508-3/solo-dogfeeding/code/198-plane/apps/api/plane/bgtasks/email_notification_task.py#L47-L84)

由 Celery Beat 每 5 分钟触发一次（定义在 [celery.py#L31-L34](file:///d:/fz/0508-3/solo-dogfeeding/code/198-plane/apps/api/plane/celery.py#L31-L34)）：

1. 查询 `EmailNotificationLog.objects.filter(processed_at__isnull=True)`
2. 按 `receiver_id` 分组，再按 `entity_identifier` 和 `triggered_by_id` 聚合
3. 标记 `processed_at=timezone.now()`
4. 对每个聚合分组调用 `send_email_notification.delay()`

**阶段 3：实际发送** — [send_email_notification](file:///d:/fz/0508-3/solo-dogfeeding/code/198-plane/apps/api/plane/bgtasks/email_notification_task.py#L152-L306)

1. 通过 Redis 分布式锁 `send_email_notif_{issue_id}_{receiver_id}_{ids_str}` 防止重复发送
2. 读取 `get_email_configuration()` 构建连接
3. 渲染模板并发送
4. **发送成功**：更新 `EmailNotificationLog.sent_at=timezone.now()`
5. **发送失败**：仅 `log_exception(e)`，`sent_at` 保持 `None`
6. 无论如何都释放 Redis 锁

### 4.2 重试策略对比

| 维度 | 即时发送模式 | Issue 通知模式 |
|------|------------|--------------|
| 自动重试 | **无** | **无** |
| 手动重试 | 不支持 | `processed_at=None` 的记录会在下个 5 分钟周期重新被 `stack_email_notification` 捡起（但 `processed_at` 已被标记） |
| 失败标记 | 无 | `sent_at` 保持 `None` |
| 日志记录 | `log_exception(e)` → 标准错误日志 | `log_exception(e)` + `sent_at=None` |

**重要发现**：Issue 通知模式在 `stack_email_notification` 阶段就把 `processed_at` 标记了，即使后续 `send_email_notification` 失败，这些记录也不会被重新处理。因此实际上 **Plane 的邮件系统没有真正的自动重试机制**。

唯一的例外是 **Webhook 发送**（[webhook_send_task](file:///d:/fz/0508-3/solo-dogfeeding/code/198-plane/apps/api/plane/bgtasks/webhook_task.py#L254-L382)），它配置了：

```python
@shared_task(
    bind=True,
    autoretry_for=(requests.RequestException,),
    retry_backoff=600,      # 10分钟退避
    max_retries=5,
    retry_jitter=True,
)
```

但这是 Webhook 调用而非邮件发送，仅当重试耗尽后会发送 `webhook-deactivate` 邮件通知。

### 4.3 日志回看机制

#### EmailNotificationLog 模型

定义在 [notification.py#L121-L149](file:///d:/fz/0508-3/solo-dogfeeding/code/198-plane/apps/api/plane/db/models/notification.py#L121-L149)：

| 字段 | 说明 |
|------|------|
| `receiver` | 接收人（FK → User） |
| `triggered_by` | 触发者（FK → User） |
| `entity_identifier` | 实体 ID（如 Issue UUID） |
| `entity_name` | 实体类型名 |
| `data` | JSON 详细数据（含 issue 和 issue_activity） |
| `processed_at` | 被 `stack_email_notification` 处理的时间 |
| `sent_at` | 实际发送成功的时间 |
| `entity` | 实体类型 |
| `old_value` / `new_value` | 变更前后值 |

**回看状态判断**：
- `processed_at=None` → 尚未处理（等待下一个 5 分钟聚合周期）
- `processed_at≠None, sent_at=None` → 已处理但发送失败
- `sent_at≠None` → 发送成功

#### 日志清理

[delete_email_notification_logs](file:///d:/fz/0508-3/solo-dogfeeding/code/198-plane/apps/api/plane/bgtasks/cleanup_task.py#L434-L443) 由 Celery Beat 每天 UTC 02:45 触发，清理 `sent_at` 超过 `HARD_DELETE_AFTER_DAYS`（默认 30 天）的记录。清理流程会先尝试归档到 MongoDB，再从 PostgreSQL 删除。

### 4.4 测试邮件

管理员可以在 Admin UI 点击"Send test email"按钮，触发 [EmailCredentialCheckEndpoint](file:///d:/fz/0508-3/solo-dogfeeding/code/198-plane/apps/api/plane/license/api/views/configuration.py#L88-L170)，该端点：

1. 读取当前 `get_email_configuration()`
2. 建立连接并发送测试邮件
3. 精细化的异常分类返回给前端：
   - `SMTPAuthenticationError` → "Invalid credentials provided"
   - `SMTPConnectError` → "Could not connect with the SMTP server"
   - `SMTPSenderRefused` → "From address is invalid"
   - `SMTPServerDisconnected` → "SMTP server disconnected unexpectedly"
   - `SMTPRecipientsRefused` → "All recipient addresses were refused"
   - `TimeoutError` → "Timeout error"
   - `ConnectionError` → "Network connection error"

注意：测试邮件是**同步发送**的（不走 Celery），直接在 API 请求线程中执行，因此能即时返回成功/失败结果。

---

## 5. 完整调用链路图

```
┌──────────────────────────────────────────────────────────────────────┐
│                        Admin UI (email page)                        │
│  ENABLE_SMTP Toggle → 禁用: DELETE /api/instances/email-features/  │
│  表单提交 → PATCH /api/instances/configurations/                    │
│  测试邮件 → POST /api/instances/email-credentials-check/            │
└──────────────────────────────┬───────────────────────────────────────┘
                               │
                               ▼
┌──────────────────────────────────────────────────────────────────────┐
│              InstanceConfiguration (DB: instance_configurations)     │
│  EMAIL_HOST / EMAIL_PORT / EMAIL_HOST_USER / EMAIL_HOST_PASSWORD    │
│  EMAIL_USE_TLS / EMAIL_USE_SSL / EMAIL_FROM / ENABLE_SMTP          │
│  (加密字段用 Fernet + PBKDF2 派生自 SECRET_KEY)                      │
└──────────────────────────────┬───────────────────────────────────────┘
                               │
          get_email_configuration()  ← 每次调用实时读 DB
                               │
        ┌──────────────────────┼───────────────────────┐
        ▼                      ▼                       ▼
┌───────────────┐  ┌──────────────────────┐  ┌────────────────────┐
│ 即时发送任务   │  │ Issue 通知（两阶段） │  │ 测试邮件（同步）   │
│               │  │                      │  │                    │
│ forgot_pass   │  │ ① notifications()    │  │ EmailCredential    │
│ magic_link    │  │   → Notification     │  │ CheckEndpoint      │
│ ws_invitation │  │   → EmailNotifLog    │  │   .post()          │
│ proj_invitat  │  │     (processed_at=   │  │                    │
│ proj_add_user │  │      None)           │  │ 直接 get_connection│
│ user_activ    │  │                      │  │ → msg.send()       │
│ user_deactiv  │  │ ② stack_email_       │  │ → 即时返回结果     │
│ email_update  │  │    notification()    │  │                    │
│ webhook_deact │  │   (Beat: */5 min)    │  └────────────────────┘
│               │  │   → processed_at=now │
│ 无重试        │  │   → send_email_      │
│ 无日志记录    │  │     notification()   │
│               │  │     (Redis锁防重)    │
│               │  │     → sent_at=now    │
│               │  │     → 或失败silent   │
│               │  │                      │
│               │  │ 无自动重试           │
│               │  │ 失败: sent_at=None   │
└───────┬───────┘  └──────────┬───────────┘
        │                     │
        ▼                     ▼
┌──────────────────────────────────────────────────────────────────┐
│                    Django SMTP Backend                            │
│  get_connection(host, port, username, password, use_tls/ssl)    │
│  → EmailMultiAlternatives(subject, body, from_email, to, conn)  │
│  → msg.attach_alternative(html, "text/html")                    │
│  → msg.send()                                                    │
└──────────────────────────────┬───────────────────────────────────┘
                               │
                               ▼
┌──────────────────────────────────────────────────────────────────┐
│                     企业邮件网关 (SMTP Server)                    │
└──────────────────────────────────────────────────────────────────┘
```

---

## 6. 运维接入邮件网关的注意事项

1. **配置热加载**：在 Admin UI 修改 SMTP 配置后，下一个 Celery 任务执行时即生效，无需重启任何服务。但 `EMAIL_BACKEND` 本身是硬编码的，不可通过 UI 切换。

2. **ENABLE_SMTP 开关**：虽然配置可以热加载，但代码中**并未检查 `ENABLE_SMTP` 的值**来决定是否发送邮件。`ENABLE_SMTP` 仅影响 Admin UI 的表单显示/隐藏。禁用后需确认所有邮件任务不会被触发。

3. **无重试机制**：所有邮件任务都没有 `autoretry_for` 配置。接入邮件网关时需确保网关本身的高可用性，或在网关前部署重试队列（如 Postfix 的 `deferred` 队列）。

4. **失败不可见**：即时发送模式的邮件失败仅记录在 `log_exception` 输出中，不在数据库留痕。建议在邮件网关侧配置发送日志和 NDR（退信通知）。

5. **密码加密**：`EMAIL_HOST_PASSWORD` 使用 Fernet 加密存储，密钥派生自 Django `SECRET_KEY`。更换 `SECRET_KEY` 会导致已存储的密码无法解密，需重新配置。

6. **TLS/SSL 互斥**：前端 [email-config-form.tsx](file:///d:/fz/0508-3/solo-dogfeeding/code/198-plane/apps/admin/app/(all)/(dashboard)/email/email-config-form.tsx#L132-L145) 强制 TLS 和 SSL 互斥选择，后端在 `get_connection()` 时通过 `use_tls=EMAIL_USE_TLS == "1"` 转换为布尔值。

7. **日志保留**：`EmailNotificationLog` 在 `sent_at` 超过 30 天后由定时任务清理，清理前会尝试归档到 MongoDB。可通过 `HARD_DELETE_AFTER_DAYS` 环境变量调整保留天数。
