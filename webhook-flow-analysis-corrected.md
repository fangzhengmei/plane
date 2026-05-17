# 站外回调（Webhook）完整流程分析（勘误版）

> 本版本基于源码逐行复核，修正初版中的乐观推断，明确区分「已实现」与「未实现/需自行实现」的能力边界。

---

## 1. 投递失败分类与重试边界

### 1.1 会自动重试的失败场景

**仅且仅有 `requests.RequestException` 及其子类会触发自动重试**（`webhook_task.py:256`）：

| 失败类型 | 示例 | 重试行为 |
|---------|------|---------|
| 网络连接错误 | TCP 连接超时、连接被拒绝 | ✅ 自动重试 |
| DNS 解析失败 | 域名无法解析 | ✅ 自动重试 |
| HTTP 超时 | 30 秒超时未收到响应 | ✅ 自动重试 |
| SSL 错误 | 证书验证失败 | ✅ 自动重试 |
| 连接池耗尽 | requests 连接池满 | ✅ 自动重试 |

**重试策略**（`webhook_task.py:254-260`）：
- 最大重试次数：5 次
- 指数退避基数：600 秒（10 分钟）
- 退避序列：10min → 20min → 40min → 80min → 160min
- 总重试窗口：约 5 小时 10 分钟
- 启用 `retry_jitter` 避免惊群效应

### 1.2 不会自动重试的失败场景（静默丢失）

以下失败发生时，事件直接丢弃，**既不重试也不通知**：

| 失败阶段 | 触发条件 | 代码位置 | 后果 |
|---------|---------|---------|------|
| 准备阶段 | Webhook 不存在或已删除 | `webhook_task.py:284` | 静默 return |
| 准备阶段 | Payload JSON 序列化失败 | `webhook_task.py:294-296` | 静默 return |
| 准备阶段 | HMAC 签名生成失败 | `webhook_task.py:315-322` | 静默 return |
| 发送前校验 | URL 运行时 SSRF 校验失败 | `webhook_task.py:330-334` | 静默 return（`validate_url()` 抛 `ValueError`） |
| 发送阶段 | 非 `requests.RequestException` 的其他异常 | `webhook_task.py:380-382` | 静默 return |
| 投递成功 | HTTP 200 但接收方业务处理失败 | `webhook_task.py:337` | 视为成功，不做检查 |

> ⚠️ **高危风险**：`validate_url()` 失败（如目标 IP 被识别为内网）不会触发重试，事件直接丢失且无任何告警。

---

## 2. 熔断机制（勘误重写）

### 2.1 熔断触发条件

**仅当 `requests.RequestException` 连续失败达到 `max_retries=5` 次时才会触发熔断**（`webhook_task.py:367`）：

- 重试计数器：`self.request.retries`（Celery 内置）
- 熔断阈值：`self.request.retries >= self.max_retries`

### 2.2 熔断动作

1. **自动禁用**：将 Webhook 的 `is_active` 字段置为 `False`（`webhook_task.py:368`）
2. **邮件通知**：向 Webhook 创建者发送停用邮件，包含管理页面链接（`webhook_task.py:371-376`）
3. **任务终止**：当前任务正常退出，不再重试

### 2.3 熔断边界

- **仅网络类失败触发熔断**：其他失败类型（如 SSRF 校验失败、序列化失败）不会触发熔断，Webhook 保持启用状态，但事件持续丢失
- **无半开/恢复机制**：熔断后需用户手动重新启用（将 `is_active` 改回 `True`）
- **无批量熔断保护**：多个 Webhook 独立计数，无工作空间级别的熔断策略

---

## 3. 签名与防重放：发送端 vs 接收端边界

### 3.1 发送端已实现能力

**请求头构造**（`webhook_task.py:286-291`）：

| 头字段 | 说明 | 实现状态 |
|--------|------|---------|
| `X-Plane-Delivery` | UUID4 唯一投递 ID | ✅ 已实现 |
| `X-Plane-Event` | 事件类型（如 `issue`/`project`） | ✅ 已实现 |
| `X-Plane-Signature` | HMAC-SHA256 签名 | ✅ 已实现 |
| `X-Plane-Timestamp` | 请求时间戳 | ❌ 未实现 |

**签名算法**（`webhook_task.py:315-322`）：
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

### 3.2 接收端需自行实现（仓内未见）

以下能力**发送端仅提供数据支持，验证逻辑完全由接收方实现**：

#### 3.2.1 签名验证（需自行实现）

接收方必须实现的验证步骤：
1. 从请求头获取 `X-Plane-Signature`
2. 读取原始请求 Body（注意：不能是解析后的对象，必须是原始字节）
3. 使用相同的 `secret_key` 和 HMAC-SHA256 计算签名
4. 使用**恒定时间比较**（如 Python 的 `hmac.compare_digest`）验证签名一致性
5. 注意 `json.dumps` 的键顺序和空格必须与发送端完全一致

> ⚠️ **注意**：发送端使用 `json.dumps(payload)` 序列化，接收方必须确保反序列化后再序列化的结果与发送端完全一致，否则签名验证失败。

#### 3.2.2 防重放（需自行实现）

发送端仅提供 `X-Plane-Delivery` 作为幂等键，但**未实现任何防重放逻辑**：

接收方需自行实现：
1. 存储已处理的 `X-Plane-Delivery` ID（建议设置过期时间，如 24 小时）
2. 每次请求先检查该 ID 是否已处理
3. 已处理则直接返回成功，避免重复消费

**限制**：
- 无时间戳支持，无法基于时间窗口做快速拒绝
- 无签名过期机制，理论上有效的请求可以被无限重放（直到接收方清理 ID 记录）

---

## 4. 租户与项目权限边界（勘误）

### 4.1 权限模型（已实现）

所有 Webhook 操作均要求 **`WORKSPACE` 级别的 `ADMIN` 角色**：

| 操作 | 权限要求 | 代码位置 |
|------|---------|---------|
| 创建 Webhook | WORKSPACE ADMIN | `webhook/base.py:21` |
| 查看 Webhook 列表 | WORKSPACE ADMIN | `webhook/base.py:38` |
| 查看 Webhook 详情 | WORKSPACE ADMIN | `webhook/base.py:38` |
| 更新 Webhook | WORKSPACE ADMIN | `webhook/base.py:78` |
| 删除 Webhook | WORKSPACE ADMIN | `webhook/base.py:104` |
| 重新生成密钥 | WORKSPACE ADMIN | `webhook/base.py:112` |
| 查看投递日志 | WORKSPACE ADMIN | `webhook/base.py:122` |

**租户隔离**：
- 所有查询均通过 `workspace__slug=slug` 过滤
- 数据通过 `workspace` 外键强关联
- 唯一性约束：`(workspace, url, deleted_at)` 联合唯一

### 4.2 项目级隔离（未实现）

**关键勘误**：`ProjectWebhook` 模型**仅定义了数据库表结构，投递链路中完全未使用**。

当前实现：
- `webhook_activity` 仅按事件类型布尔字段过滤（`webhook_task.py:428-441`）
- 没有任何代码通过 `ProjectWebhook` 关联表过滤 Webhook
- 所有 Webhook 本质上都是**工作空间级**的，订阅后接收该工作空间下所有项目的事件

> ⚠️ 若需项目级隔离，需扩展 `webhook_activity` 的过滤逻辑，通过 `ProjectWebhook` 表关联查询。

---

## 5. 更新路径下的生效细节与风险

### 5.1 URL 更新

**校验逻辑**（`webhook.py:62-66`）：
- PATCH 更新 URL 时，会调用 `_validate_webhook_url()` 进行完整校验
- 校验规则与创建时完全一致（SSRF 防护、域名黑名单等）

**对在途任务的影响**：
- Celery 任务仅存储 `webhook_id`，不存储 URL
- 每次重试时重新从数据库读取 Webhook 配置（`webhook_task.py:284`）
- ✅ **结论**：URL 更新后，所有在重试队列中的任务都会使用新 URL 进行下一次重试

### 5.2 事件订阅字段更新（`project`/`issue`/`cycle`/`module`/`issue_comment`）

**生效时机**：即时生效
- 更新后，`webhook_activity` 在下一次查询时就会使用新的布尔字段过滤
- 已进入投递队列的任务不受影响（因为已经过了过滤阶段）

**风险**：无确认/延迟机制，误操作可能导致事件漏发或多发。

### 5.3 Secret Key 轮换

**更新方式**：仅能通过 `/regenerate/` 接口重新生成，PATCH 接口不允许修改（`secret_key` 为只读字段）

**对在途任务的影响**：
- 每次重试重新读取 `secret_key`（`webhook_task.py:284`）
- ⚠️ **风险**：密钥轮换后，在重试队列中的任务会使用**新密钥**签名
- 接收方需支持双密钥过渡期，否则在途重试事件会签名验证失败

### 5.4 删除操作

**实现方式**：调用 `webhook.delete()`（`webhook/base.py:107`），基于 `BaseModel` 的软删除机制，设置 `deleted_at` 字段

**投递过滤逻辑**（`webhook_task.py:426`）：
```python
webhooks = Webhook.objects.filter(workspace__slug=slug, is_active=True)
```

- 仅过滤 `is_active=True`，**未过滤 `deleted_at__isnull=True`**
- ⚠️ **潜在风险**：如果删除操作未同时设置 `is_active=False`，已软删除的 Webhook 理论上仍可能被投递（取决于 `BaseModel.delete()` 的具体实现）

---

## 6. 域名限制在更新路径下的生效细节

### 6.1 限制层级

| 限制类型 | 配置来源 | 生效时机 |
|---------|---------|---------|
| Schema 限制 | 硬编码（仅 http/https） | 创建 + 更新 URL 时 |
| Localhost 限制 | 硬编码 | 创建 + 更新 URL 时 |
| SSRF IP 限制 | `WEBHOOK_ALLOWED_IPS` | 创建 + 更新 URL 时 + **每次投递前** |
| Hostname 白名单 | `WEBHOOK_ALLOWED_HOSTS` | 创建 + 更新 URL 时 + 每次投递前 |
| 域名黑名单 | `WEBHOOK_DISALLOWED_DOMAINS` | 创建 + 更新 URL 时 |
| 请求主机回环防护 | 动态获取 `request.get_host()` | 创建 + 更新 URL 时 |

### 6.2 关键差异

| 检查项 | 创建/更新时校验 | 每次投递前校验 |
|--------|----------------|---------------|
| Schema | ✅ | ❌（但 requests 会隐式检查） |
| Localhost | ✅ | ❌ |
| SSRF IP 防护 | ✅ | ✅（`webhook_task.py:330-334`） |
| Hostname 白名单 | ✅ | ✅ |
| 域名黑名单 | ✅ | ❌ |
| 请求主机回环 | ✅ | ❌ |

> ⚠️ **投递时仅重做 SSRF IP 检查**：域名黑名单等检查仅在配置时生效，投递时不重复校验。

---

## 7. 设计要点总结（勘误版）

### ✅ 已实现的保障机制

1. **网络异常至少一次投递**：Celery 持久化任务 + 网络异常自动重试
2. **幂等性支持**：`X-Plane-Delivery` 提供幂等键（需接收方配合）
3. **传输层安全**：HMAC-SHA256 签名（需接收方验证）
4. **SSRF 防护**：配置时 + 投递前双重 IP 校验
5. **可观测性**：MongoDB/PostgreSQL 双写日志，记录完整请求响应

### ⚠️ 关键缺失/风险

1. **静默失败**：非网络类异常直接丢弃事件，无告警
2. **SSRF 校验失败不重试**：内网 IP 检测失败直接丢事件
3. **无死信队列**：重试耗尽或非重试类失败无补救机制
4. **项目级隔离未实现**：`ProjectWebhook` 表空置
5. **无签名时间戳**：防重放完全依赖接收方存储 delivery ID
6. **密钥轮换无过渡**：在途任务即时使用新密钥
7. **HTTP 状态码不检查**：200/400/500 均视为投递成功
8. **软删除过滤不完整**：仅靠 `is_active` 过滤，未检查 `deleted_at`

### 📌 接收方集成必读

1. 必须实现 HMAC-SHA256 签名验证（注意 JSON 序列化一致性）
2. 必须实现基于 `X-Plane-Delivery` 的防重放（建议 TTL ≥ 24 小时）
3. 密钥轮换时需支持双密钥过渡期
4. 需容忍重复投递（至少一次语义）
5. 建议校验 `X-Plane-Event` 与 payload.event 一致性
