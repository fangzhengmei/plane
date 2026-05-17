# 站外回调（Webhook）完整流程分析

## 1. 架构总览

站外回调系统采用 **工作空间级配置 + 事件驱动投递 + 异步任务重试 + 双写日志** 的架构设计，实现从配置注册到事件投递的完整闭环。

```
配置注册 → 事件订阅 → 异步投递 → 失败重试 → 审计日志
    ↑           ↑           ↑          ↑          ↑
    │           │           │          │          │
  权限校验   业务触发    签名校验    指数退避   双写存储
```

---

## 2. 配置注册流程

### 2.1 数据模型

核心模型定义于 `apps/api/plane/db/models/webhook.py`，包含三个关键实体：

| 模型 | 核心字段 | 作用 |
|------|---------|------|
| **Webhook** | `workspace`, `url`, `secret_key`, `is_active`, `project/issue/module/cycle/issue_comment` (布尔字段) | 回调配置主表 |
| **WebhookLog** | `webhook`, `event_type`, `request_*`, `response_*`, `retry_count` | 投递审计日志 |
| **ProjectWebhook** | `webhook`, `project` | 项目级回调关联表 |

### 2.2 配置创建与校验

**入口**：`POST /api/workspaces/<slug>/webhooks/`

**权限边界**：仅 `WORKSPACE` 级别的 `ADMIN` 角色可操作（`webhook/base.py:21`）

**校验流程**：

1. **Schema 校验**：仅允许 `http` 和 `https` 协议（`webhook.py:21-24`）
2. **域名校验**：禁止 `localhost` 和 `127.0.0.1`（`webhook.py:27-31`）
3. **SSRF 防护**：`validate_url()` 函数执行深层检查（`ip_address.py:11-60`）
   - 解析主机名并验证 IP 地址
   - 禁止私有网络、回环地址、保留地址、链路本地地址
   - 支持 `WEBHOOK_ALLOWED_IPS` 白名单绕过
   - 支持 `WEBHOOK_ALLOWED_HOSTS` 主机名白名单
4. **域名黑名单**：检查 `WEBHOOK_DISALLOWED_DOMAINS`，自动追加当前请求主机作为回环防护（`webhook.py:48-55`）
5. **唯一性约束**：同一工作空间下 URL 唯一（软删除级别）

### 2.3 密钥管理

- **自动生成**：创建时自动生成 `secret_key`，格式为 `plane_wh_` + UUID4 hex（`webhook.py:17-18`）
- **密钥轮换**：支持通过 `/regenerate/` 接口重新生成（`webhook/base.py:111-118`）
- **只读保护**：序列化器中 `secret_key` 为只读字段，无法通过更新接口修改

---

## 3. 事件订阅与触发

### 3.1 可订阅事件类型

Webhook 模型通过五个布尔字段控制订阅范围：

| 字段 | 覆盖事件 |
|------|---------|
| `project` | 项目创建/更新/删除 |
| `issue` | 工作项创建/更新/删除 |
| `module` | 模块及模块-工作项关联变更 |
| `cycle` | 迭代及迭代-工作项关联变更 |
| `issue_comment` | 工作项评论 |

### 3.2 事件触发链路

**触发点**：分布于各业务视图的 CRUD 操作中，例如：
- Issue 创建：`issue/base.py:461-469`
- Issue 更新：`issue/base.py:686-694`

**调用链**：

```
业务视图 → model_activity.delay() → webhook_activity.delay() → webhook_send_task.delay()
```

#### 阶段一：`model_activity`（`webhook_task.py:471-512`）

- 对比 `requested_data` 与 `current_instance` 的差异
- 为每个变更字段触发独立的 `webhook_activity` 任务
- 新创建实体直接触发 `created` 事件

#### 阶段二：`webhook_activity`（`webhook_task.py:385-468`）

- 根据 `event` 类型过滤匹配的 Webhook（`webhook_task.py:428-441`）
- 仅处理 `is_active=True` 的 Webhook
- 为每个匹配的 Webhook 发起独立的投递任务

### 3.3 事件数据构造

**序列化映射**：`SERIALIZER_MAPPER` 定义各事件类型对应的序列化器（`webhook_task.py:59-69`）

**Payload 结构**：
```python
payload = {
    "event": event,           # 事件类型: project/issue/module/...
    "action": action,         # 操作: create/update/delete
    "webhook_id": str,        # Webhook ID
    "workspace_id": str,      # 工作空间 ID
    "data": event_data,       # 实体完整数据（删除时仅含 id）
    "activity": {             # 变更详情（仅 update 时）
        "field": str,
        "old_value": Any,
        "new_value": Any,
        "actor": UserData,
        "old_identifier": str,
        "new_identifier": str,
    }
}
```

---

## 4. 投递保证与签名校验

### 4.1 请求构造

**核心实现**：`webhook_send_task`（`webhook_task.py:254-382`）

**请求头**：
| 头字段 | 说明 |
|--------|------|
| `Content-Type` | `application/json` |
| `User-Agent` | `Autopilot` |
| `X-Plane-Delivery` | 唯一投递 ID（UUID4），用于幂等识别 |
| `X-Plane-Event` | 事件类型 |
| `X-Plane-Signature` | HMAC-SHA256 签名 |

### 4.2 签名校验机制

**算法**：HMAC-SHA256（`webhook_task.py:315-322`）

```python
hmac_signature = hmac.new(
    webhook.secret_key.encode("utf-8"),
    json.dumps(payload).encode("utf-8"),
    hashlib.sha256,
)
signature = hmac_signature.hexdigest()
```

**接收方验证步骤**：
1. 从请求头获取 `X-Plane-Signature`
2. 使用相同的 `secret_key` 对请求 Body 计算 HMAC-SHA256
3. 采用**恒定时间比较**（`hmac.compare_digest`）验证签名一致性
4. 验证 `X-Plane-Delivery` 防止重放攻击

### 4.3 运行时安全加固

**发送前二次校验**（`webhook_task.py:329-334`）：
- 每次投递前重新调用 `validate_url()` 验证目标地址
- 防止 DNS 重新绑定攻击（DNS rebinding）

**超时控制**：
- HTTP 请求超时：30 秒（`webhook_task.py:337`）

---

## 5. 失败重试与熔断机制

### 5.1 重试策略

**Celery 任务配置**（`webhook_task.py:254-260`）：

| 参数 | 值 | 说明 |
|------|----|------|
| `autoretry_for` | `(requests.RequestException,)` | 网络异常自动重试 |
| `retry_backoff` | 600 | 指数退避基数：10分钟 |
| `max_retries` | 5 | 最大重试次数 |
| `retry_jitter` | True | 添加随机抖动避免惊群效应 |

**退避时间表**（含抖动）：
| 重试次数 | 延迟（约） |
|----------|-----------|
| 第1次 | 10 分钟 |
| 第2次 | 20 分钟 |
| 第3次 | 40 分钟 |
| 第4次 | 80 分钟 |
| 第5次 | 160 分钟 |

> 总计最大重试窗口约 **5 小时 10 分钟**

### 5.2 熔断机制

**触发条件**：重试次数达到 `max_retries`（`webhook_task.py:367-377`）

**动作**：
1. 将 Webhook 的 `is_active` 置为 `False`（禁用）
2. 向 Webhook 创建者发送停用邮件通知
3. 邮件包含 Webhook 管理页面链接，便于排查和重新启用

### 5.3 审计日志

**双写策略**（`webhook_task.py:94-141`）：

1. **主存储**：MongoDB `webhook_logs` 集合（优先）
2. **降级存储**：PostgreSQL `webhook_logs` 表（MongoDB 不可用时）

**日志字段**：
- 请求信息：方法、头、Body、事件类型
- 响应信息：状态码、头、Body
- 元数据：重试次数、工作空间 ID、Webhook ID

---

## 6. 租户与权限边界

### 6.1 租户隔离

- **数据隔离**：所有 Webhook 和日志均通过 `workspace` 外键强关联
- **查询隔离**：所有 API 操作均通过 `workspace__slug=slug` 过滤
- **唯一性约束**：`(workspace, url, deleted_at)` 联合唯一

### 6.2 权限模型

**管理权限**（`webhook/base.py`）：

| 操作 | 所需权限 |
|------|---------|
| 创建 Webhook | WORKSPACE ADMIN |
| 查看 Webhook 列表 | WORKSPACE ADMIN |
| 更新 Webhook | WORKSPACE ADMIN |
| 删除 Webhook | WORKSPACE ADMIN |
| 重新生成密钥 | WORKSPACE ADMIN |
| 查看投递日志 | WORKSPACE ADMIN |

### 6.3 项目级边界

- `ProjectWebhook` 模型支持 Webhook 与项目的关联
- 当前实现中事件过滤仅基于事件类型，未进一步按项目过滤
- 如需项目级隔离，可通过 `ProjectWebhook` 关联表扩展过滤逻辑

---

## 7. 可配置项

通过环境变量配置：

| 变量 | 说明 |
|------|------|
| `WEBHOOK_ALLOWED_IPS` | 允许的 IP/CIDR 白名单，逗号分隔 |
| `WEBHOOK_ALLOWED_HOSTS` | 允许的主机名白名单，逗号分隔 |
| `WEBHOOK_DISALLOWED_DOMAINS` | 禁止的域名黑名单，逗号分隔 |

---

## 8. 设计要点总结

### ✅ 已实现的保障机制

1. **至少一次投递**：Celery 任务持久化 + 失败重试
2. **幂等性支持**：`X-Plane-Delivery` 提供幂等键
3. **端到端安全**：HMAC 签名 + SSRF 防护 + DNS 重绑定防护
4. **可观测性**：双写日志 + 完整请求响应记录
5. **故障自愈**：自动熔断 + 邮件通知

### ⚠️ 潜在优化点

1. **死信队列**：重试耗尽后的事件可进入死信队列人工处理
2. **批量投递**：高频事件可考虑批量合并
3. **项目级过滤**：`ProjectWebhook` 关联表尚未在投递时参与过滤
4. **签名时间戳**：可增加 `X-Plane-Timestamp` 增强重放防护
5. **回调健康检查**：定期探测端点可用性，提前预警
