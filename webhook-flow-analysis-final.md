# 站外回调（Webhook）完整流程分析（最终统一口径）

> 本文档基于源码逐行核对，所有结论与实际代码实现完全对齐，可直接用于排障和安全评估。

---

## 1. 整体架构

站外回调系统采用 **工作空间级配置 + 事件驱动投递 + 异步任务重试 + 双写日志** 的架构设计：

```
配置注册 → 事件订阅 → 异步投递 → 失败重试 → 审计日志
    ↑           ↑           ↑          ↑          ↑
    │           │           │          │          │
  权限校验   业务触发    签名校验    指数退避   双写存储
```

---

## 2. 配置注册流程

### 2.1 数据模型

| 模型 | 核心字段 | 说明 |
|------|---------|------|
| **Webhook** | `workspace`, `url`, `secret_key`, `is_active`, `project/issue/module/cycle/issue_comment` | 回调配置主表，软删除 |
| **WebhookLog** | `webhook`, `event_type`, `request_*`, `response_*`, `retry_count` | 投递审计日志 |
| **ProjectWebhook** | `webhook`, `project` | 项目级回调关联表（仅定义，未使用） |

### 2.2 URL 校验层级

URL 校验分为**模型层**和**序列化器层**两层：

#### 模型层校验（创建和更新 URL 时触发）
由 `validate_schema` 和 `validate_domain` 两个 validator 实现：
- `validate_schema`：仅允许 `http` 和 `https` 协议
- `validate_domain`：禁止 `localhost` 和 `127.0.0.1`

#### 序列化器层校验（`_validate_webhook_url`）
| 校验步骤 | 检查内容 | 依赖 |
|---------|---------|------|
| 1. SSRF 防护 | 调用 `validate_url()` 检查协议、主机名、IP | 无外部依赖 |
| 2. 白名单豁免 | 主机名在 `WEBHOOK_ALLOWED_HOSTS` 中则跳过后续检查 | `WEBHOOK_ALLOWED_HOSTS` 配置 |
| 3. 域名黑名单 | 检查 `WEBHOOK_DISALLOWED_DOMAINS` 配置 | `WEBHOOK_DISALLOWED_DOMAINS` 配置 |
| 4. 回环防护 | 追加当前请求主机到黑名单，防止回环攻击 | `context["request"]` |

### 2.3 创建与更新路径的实现差异

| 环节 | 创建路径（POST） | 更新路径（PATCH） | 差异说明 |
|------|-----------------|-----------------|---------|
| context 传递 | `context={"request": request}` ✅ 正确 | `context={request: request}` ❌ 错误 | PATCH 路径 key 类型错误（对象 vs 字符串） |
| 回环防护（第 4 步） | ✅ 正常工作 | ❌ 失效 | `self.context.get("request")` 返回 None，跳过动态回环防护 |
| 模型层校验 | ✅ 正常工作 | ✅ 正常工作 | 无差异 |
| SSRF 防护（第 1 步） | ✅ 正常工作 | ✅ 正常工作 | 无差异 |
| 白名单豁免（第 2 步） | ✅ 正常工作 | ✅ 正常工作 | 无差异 |
| 静态域名黑名单（第 3 步） | ✅ 正常工作 | ✅ 正常工作 | 无差异 |

> 📌 **统一结论**：PATCH 更新时，**仅动态请求主机回环防护失效**，其他所有校验均正常工作。静态配置的 `WEBHOOK_DISALLOWED_DOMAINS` 仍然生效，只有动态追加的当前请求主机防护失效。

### 2.4 密钥管理

- **自动生成**：创建时自动生成 `secret_key`，格式为 `plane_wh_` + UUID4 hex
- **密钥轮换**：仅能通过 `/regenerate/` 接口重新生成，PATCH 接口不允许修改
- **只读保护**：序列化器中 `secret_key` 为只读字段

---

## 3. 投递前校验范围（统一口径）

每次投递前调用 `validate_url()` 进行完整校验，包含 5 个步骤：

| 步骤 | 校验内容 | 失败处理 |
|------|---------|---------|
| 1 | 主机名存在性检查 | 抛 `ValueError` |
| 2 | 协议校验（仅 http/https） | 抛 `ValueError` |
| 3 | 主机名白名单匹配 | 匹配成功则跳过后续 IP 检查 |
| 4 | DNS 解析获取 IP 列表 | 解析失败抛 `ValueError` |
| 5 | SSRF IP 检查（禁止私有/回环/保留/链路本地地址） | 违规抛 `ValueError`，`WEBHOOK_ALLOWED_IPS` 白名单可豁免 |

> 📌 **统一结论**：投递前校验 = 协议校验 + 主机名校验 + SSRF IP 校验，是三层完整校验，而非仅 IP 检查。

---

## 4. 投递失败分类与重试边界

### 4.1 会自动重试的失败场景

**仅且仅有 `requests.RequestException` 及其子类会触发自动重试**：

| 失败类型 | 示例 | 重试行为 |
|---------|------|---------|
| 网络连接错误 | TCP 连接超时、连接被拒绝 | ✅ 自动重试 |
| DNS 解析失败 | 域名无法解析 | ✅ 自动重试 |
| HTTP 超时 | 30 秒超时未收到响应 | ✅ 自动重试 |
| SSL 错误 | 证书验证失败 | ✅ 自动重试 |
| 连接池耗尽 | requests 连接池满 | ✅ 自动重试 |

**重试策略**：
- 最大重试次数：5 次
- 指数退避基数：600 秒（10 分钟）
- 退避序列：10min → 20min → 40min → 80min → 160min
- 总重试窗口：约 5 小时 10 分钟
- 启用 `retry_jitter` 避免惊群效应

### 4.2 不会自动重试的失败场景（静默丢失）

以下失败发生时，事件直接丢弃，**既不重试也不通知**：

| 失败阶段 | 触发条件 | 代码位置 | 后果 |
|---------|---------|---------|------|
| 准备阶段 | Webhook 不存在或已删除 | `webhook_task.py:284` | 静默 return |
| 准备阶段 | Payload JSON 序列化失败 | `webhook_task.py:294-296` | 静默 return |
| 准备阶段 | HMAC 签名生成失败 | `webhook_task.py:315-322` | 静默 return |
| 发送前校验 | 协议校验失败（非 http/https） | `webhook_task.py:330-334` | 静默 return |
| 发送前校验 | 主机名为空 | `webhook_task.py:330-334` | 静默 return |
| 发送前校验 | Hostname 不在白名单且 IP 解析为内网 | `webhook_task.py:330-334` | 静默 return |
| 发送前校验 | DNS 解析失败 | `webhook_task.py:330-334` | 静默 return |
| 发送阶段 | 非 `requests.RequestException` 的其他异常 | `webhook_task.py:380-382` | 静默 return |
| 投递成功 | HTTP 200 但接收方业务处理失败 | `webhook_task.py:337` | 视为成功，不做检查 |

> ⚠️ **高危风险**：发送前校验的任何失败都不会触发重试，事件直接丢失且无任何告警。

---

## 5. 熔断机制

### 5.1 熔断触发条件

**仅当 `requests.RequestException` 连续失败达到 `max_retries=5` 次时才会触发熔断**：

- 重试计数器：`self.request.retries`（Celery 内置）
- 熔断阈值：`self.request.retries >= self.max_retries`

### 5.2 熔断动作

1. **自动禁用**：将 Webhook 的 `is_active` 字段置为 `False`
2. **邮件通知**：向 Webhook 创建者发送停用邮件，包含管理页面链接
3. **任务终止**：当前任务正常退出，不再重试

### 5.3 熔断边界

- **仅网络类失败触发熔断**：其他失败类型（如发送前校验失败、序列化失败）不会触发熔断，Webhook 保持启用状态，但事件持续丢失
- **无半开/恢复机制**：熔断后需用户手动重新启用
- **无批量熔断保护**：多个 Webhook 独立计数，无工作空间级别的熔断策略

---

## 6. 签名与请求标识：发送端 vs 接收端边界

### 6.1 发送端已实现能力

**请求头构造**（`webhook_task.py:286-291`）：

| 头字段 | 说明 | 实现状态 | 生成时机 |
|--------|------|---------|---------|
| `Content-Type` | `application/json` | ✅ 已实现 | 每次投递 |
| `User-Agent` | `Autopilot` | ✅ 已实现 | 每次投递 |
| `X-Plane-Delivery` | UUID4 投递标识 | ✅ 已实现 | **每次任务执行时重新生成**（含重试） |
| `X-Plane-Event` | 事件类型（如 `issue`/`project`） | ✅ 已实现 | 每次投递 |
| `X-Plane-Signature` | HMAC-SHA256 签名 | ✅ 已实现 | 每次投递 |
| `X-Plane-Timestamp` | 请求时间戳 | ❌ 未实现 | - |

**关键行为**（`webhook_task.py:289`）：
```python
"X-Plane-Delivery": str(uuid.uuid4()),
```

> 📌 **统一结论**：`X-Plane-Delivery` 在**每次任务执行时重新生成**。由于 Celery 重试本质是重新执行任务函数，因此**同一事件的每次重试都会生成不同的 `X-Plane-Delivery` 值**。该字段不能用于识别同一事件的多次重试。

### 6.2 签名算法

```python
hmac_signature = hmac.new(
    webhook.secret_key.encode("utf-8"),
    json.dumps(payload).encode("utf-8"),
    hashlib.sha256,
)
headers["X-Plane-Signature"] = hmac_signature.hexdigest()
```

- 签名范围：**整个 payload JSON 字符串**
- 编码：UTF-8
- 哈希算法：SHA-256

### 6.3 接收端需自行实现（仓内未见）

#### 6.3.1 签名验证（需自行实现）

接收方必须实现的验证步骤：
1. 从请求头获取 `X-Plane-Signature`
2. 读取原始请求 Body（注意：不能是解析后的对象，必须是原始字节）
3. 使用相同的 `secret_key` 和 HMAC-SHA256 计算签名
4. 使用**恒定时间比较**验证签名一致性
5. 注意 `json.dumps` 的键顺序和空格必须与发送端完全一致

#### 6.3.2 防重放（需自行实现，且无可靠幂等键）

**重要勘误**：由于 `X-Plane-Delivery` 每次重试都会重新生成，**该字段不能作为幂等键用于识别重复投递**。

接收方防重放的可选方案：
1. **基于业务主键去重**：使用 payload 中的业务实体 ID（如 `data.id`）+ 事件类型 + 操作类型作为去重键
2. **基于时间窗口 + 内容哈希**：计算 payload 哈希，结合时间窗口（如 10 分钟内）去重
3. **接收端生成幂等键**：如果业务场景允许，由接收方维护已处理的业务事件记录

**当前实现的限制**：
- 无时间戳支持，无法基于时间窗口做快速拒绝
- 无可靠的幂等投递标识，无法精确识别同一事件的多次重试
- 无签名过期机制，理论上有效的请求可以被无限重放

---

## 7. 软删除与投递关系（统一口径）

### 7.1 软删除实现机制

`Webhook` 继承自 `SoftDeleteModel`，默认管理器自动过滤已删除记录：

```python
class SoftDeletionManager(models.Manager):
    def get_queryset(self):
        return SoftDeletionQuerySet(self.model).filter(deleted_at__isnull=True)
```

### 7.2 投递时的过滤逻辑

**事件触发阶段**（`webhook_task.py:426`）：
```python
webhooks = Webhook.objects.filter(workspace__slug=slug, is_active=True)
```
- 使用默认管理器 `objects`，自动附加 `deleted_at__isnull=True` 条件
- 同时过滤 `is_active=True`
- ✅ **结论**：软删除的 Webhook 不会被触发投递

**任务执行阶段**（`webhook_task.py:284`）：
```python
webhook = Webhook.objects.get(id=webhook_id, workspace__slug=slug)
```
- 同样使用默认管理器 `objects`
- 如果 Webhook 在任务入队后被软删除，`get()` 会抛 `DoesNotExist` 异常
- 异常被外层 `except Exception as e` 捕获，任务静默 return

> 📌 **统一结论**：软删除机制在默认查询行为下工作正常，已软删除的 Webhook 不会被投递。前版"软删除过滤不完整"的判断错误，予以修正。

---

## 8. 租户与项目权限边界

### 8.1 权限模型（已实现）

所有 Webhook 操作均要求 **`WORKSPACE` 级别的 `ADMIN` 角色**：

| 操作 | 权限要求 |
|------|---------|
| 创建 Webhook | WORKSPACE ADMIN |
| 查看 Webhook 列表/详情 | WORKSPACE ADMIN |
| 更新 Webhook | WORKSPACE ADMIN |
| 删除 Webhook | WORKSPACE ADMIN |
| 重新生成密钥 | WORKSPACE ADMIN |
| 查看投递日志 | WORKSPACE ADMIN |

**租户隔离**：
- 所有查询均通过 `workspace__slug=slug` 过滤
- 数据通过 `workspace` 外键强关联
- 唯一性约束：`(workspace, url, deleted_at)` 联合唯一

### 8.2 项目级隔离（未实现）

**统一结论**：`ProjectWebhook` 模型**仅定义了数据库表结构，投递链路中完全未使用**。

- `webhook_activity` 仅按事件类型布尔字段过滤
- 没有任何代码通过 `ProjectWebhook` 关联表过滤 Webhook
- 所有 Webhook 本质上都是**工作空间级**的，订阅后接收该工作空间下所有项目的事件

---

## 9. 更新路径下的生效细节（与代码完全对齐）

### 9.1 URL 更新

**校验逻辑**：
- PATCH 更新 URL 时，会调用 `_validate_webhook_url()` 进行完整校验
- 如第 2.3 节所述，仅动态请求主机回环防护失效，其他校验正常

**对在途任务的影响**：
- Celery 任务仅存储 `webhook_id`，不存储 URL
- 每次重试时重新从数据库读取 Webhook 配置
- ✅ **结论**：URL 更新后，所有在重试队列中的任务都会使用新 URL 进行下一次重试

### 9.2 事件订阅字段更新

**生效时机**：即时生效
- 更新后，`webhook_activity` 在下一次查询时就会使用新的布尔字段过滤
- 已进入投递队列的任务不受影响

**风险**：无确认/延迟机制，误操作可能导致事件漏发或多发。

### 9.3 Secret Key 轮换

**更新方式**：仅能通过 `/regenerate/` 接口重新生成，PATCH 接口不允许修改

**对在途任务的影响**：
- 每次重试重新读取 `secret_key`
- ⚠️ **风险**：密钥轮换后，在重试队列中的任务会使用**新密钥**签名
- 接收方需支持双密钥过渡期，否则在途重试事件会签名验证失败

### 9.4 删除操作

**实现方式**：调用 `webhook.delete()`，默认软删除，设置 `deleted_at` 字段

**投递过滤逻辑**：
- 如第 7 章所述，默认管理器自动过滤 `deleted_at__isnull=True`
- 已软删除的 Webhook 不会被触发投递
- 在途任务遇到 Webhook 被删除会静默失败

---

## 10. PATCH 上下文传递异常的影响范围（统一口径）

### 10.1 问题描述

PATCH 更新接口存在 context 传递错误（`webhook/base.py:84`）：
```python
context={request: request}  # ❌ key 应该是字符串 "request"
```

### 10.2 受影响的校验

| 校验项 | 受影响？ | 说明 |
|--------|---------|------|
| 模型层 schema 校验 | ❌ 不受影响 | 由模型 validator 独立执行 |
| 模型层 localhost 校验 | ❌ 不受影响 | 由模型 validator 独立执行 |
| `validate_url()` SSRF 防护 | ❌ 不受影响 | 不依赖 context |
| `WEBHOOK_ALLOWED_HOSTS` 白名单 | ❌ 不受影响 | 不依赖 context |
| `WEBHOOK_DISALLOWED_DOMAINS` 静态黑名单 | ❌ 不受影响 | 不依赖 context |
| 动态请求主机回环防护 | ✅ 受影响 | `self.context.get("request")` 返回 None，跳过 |

### 10.3 具体风险

用户可通过 PATCH 接口将 Webhook URL 更新为**当前 Plane 实例的域名**，绕过回环防护。

**缓解**：创建接口无此问题，可通过创建新 Webhook 替代更新操作。

---

## 11. 设计要点总结（最终统一口径）

### ✅ 已实现的保障机制

1. **网络异常至少一次投递**：Celery 持久化任务 + 网络异常自动重试
2. **传输层安全**：HMAC-SHA256 签名（需接收方验证）
3. **SSRF 防护**：配置时 + 投递前双重校验（协议 + 主机名 + IP）
4. **软删除隔离**：默认管理器自动过滤已删除记录，工作正常
5. **可观测性**：MongoDB/PostgreSQL 双写日志，记录完整请求响应
6. **请求标识**：`X-Plane-Delivery` 用于追踪单次投递（但不可用于幂等）

### ⚠️ 关键缺失/风险

1. **静默失败**：非网络类异常直接丢弃事件，无告警
2. **发送前校验失败不重试**：协议错误、内网 IP、DNS 失败直接丢事件
3. **无死信队列**：重试耗尽或非重试类失败无补救机制
4. **项目级隔离未实现**：`ProjectWebhook` 表空置
5. **无签名时间戳**：无法基于时间窗口快速拒绝重放
6. **无可靠幂等键**：`X-Plane-Delivery` 每次重试重新生成，无法用于去重
7. **密钥轮换无过渡**：在途任务即时使用新密钥
8. **HTTP 状态码不检查**：200/400/500 均视为投递成功
9. **PATCH 接口 context 传递错误**：导致更新 URL 时动态回环防护失效

### 📌 接收方集成必读

1. **必须实现 HMAC-SHA256 签名验证**：注意 JSON 序列化一致性（键顺序、空格）
2. **防重放需基于业务字段**：`X-Plane-Delivery` 不可靠，建议使用 `data.id` + `event` + `action` 作为去重键
3. **密钥轮换支持双密钥**：在途重试事件会使用新密钥，需平滑过渡
4. **容忍重复投递**：至少一次语义，业务操作需保证幂等
5. **校验事件类型一致性**：建议校验 `X-Plane-Event` 与 `payload.event` 是否一致
6. **注意重试的签名变化**：同一事件的多次重试 payload 相同，但 `X-Plane-Delivery` 不同

### 📌 运维安全建议

1. **必须配置 `WEBHOOK_DISALLOWED_DOMAINS`**：将 Plane 实例域名加入黑名单，缓解 PATCH 回环防护失效问题
2. **监控静默失败日志**：重点关注 `webhook_task.py:326`（准备阶段失败）和 `:382`（发送阶段非网络异常）
3. **重要业务建议 ACK 机制**：接收方返回成功后再执行业务操作，避免事件丢失
4. **密钥轮换前暂停 Webhook**：等待在途任务完成后再轮换密钥，避免签名验证失败
5. **定期审计 Webhook 配置**：检查是否有被更新为内部域名的 Webhook
