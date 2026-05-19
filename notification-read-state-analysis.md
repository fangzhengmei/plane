# 通知系统状态链路分析

## 概述

通知系统的完整状态链路分为三个核心阶段：
1. **事件触发与订阅者筛选** - 后端事件驱动
2. **通知条目生成与落库** - 后端批量处理
3. **客户端轮询与已读标记** - 前后端交互

---

## 阶段一：事件触发与订阅者筛选

### 状态机流转

```
事件发生 → 活动记录(IssueActivity) → 通知任务触发 → 订阅者筛选 → 合格接收者集合
```

### 核心代码路径

**触发入口**：`apps/api/plane/bgtasks/issue_activities_task.py:1504-1604`

```python
@shared_task
def issue_activity(type, requested_data, current_instance, issue_id, actor_id, project_id, epoch, ...):
    # 1. 根据活动类型映射到具体处理函数
    ACTIVITY_MAPPER = {
        "issue.activity.created": create_issue_activity,
        "issue.activity.updated": update_issue_activity,
        "comment.activity.created": create_comment_activity,
        # ... 共20+种活动类型
    }
    
    # 2. 批量创建活动记录
    issue_activities_created = IssueActivity.objects.bulk_create(issue_activities)
    
    # 3. 触发通知任务
    if notification:
        notifications.delay(
            type=type,
            issue_id=issue_id,
            issue_activities_created=json.dumps(...),
            ...
        )
```

**订阅者筛选逻辑**：`apps/api/plane/bgtasks/notification_task.py:190-670`

筛选流程（按优先级）：

| 筛选类型 | 代码位置 | 筛选条件 |
|---------|---------|---------|
| **项目成员过滤** | L231-233 | 只考虑活跃的项目成员 |
| **新提及用户** | L236-238 | 描述中新提及且之前未提及的用户 |
| **评论提及用户** | L249-265 | 评论中新提及的用户 |
| **问题订阅者** | L280-288 | 订阅者 - 新提及用户 - 评论提及用户 - 操作者 |
| **问题负责人** | L303-307 | 单独获取负责人列表用于sender字段区分 |

### 关键排除规则

```python
issue_subscribers = list(
    IssueSubscriber.objects.filter(...)
    .exclude(subscriber_id__in=list(new_mentions + comment_mentions + [actor_id]))
    .values_list("subscriber", flat=True)
)
```

- 已被提及的用户不重复发送订阅通知
- 操作者本人不接收通知
- 非项目成员不接收通知

---

## 阶段二：通知条目生成与落库

### 状态机流转

```
合格接收者集合 → 个性化通知构建 → 批量落库(Notification表) → 邮件日志记录(EmailNotificationLog)
```

### 数据模型

**通知主表**：`apps/api/plane/db/models/notification.py:13-69`

| 字段 | 状态含义 | 初始值 |
|-----|---------|-------|
| `read_at` | 是否已读 | `None`（未读） |
| `snoozed_till` | 延后提醒时间 | `None` |
| `archived_at` | 归档时间 | `None` |
| `sender` | 通知类型标识 | `in_app:issue_activities:{created/assigned/subscribed/mentioned}` |

### 通知构建与落库

**构建逻辑**：`notification_task.py:311-406`

```python
for subscriber in issue_subscribers:
    # 根据用户角色区分sender
    if issue.created_by_id == subscriber:
        sender = "in_app:issue_activities:created"
    elif subscriber in issue_assignees:
        sender = "in_app:issue_activities:assigned"
    else:
        sender = "in_app:issue_activities:subscribed"
    
    # 检查用户通知偏好
    preference = UserNotificationPreference.objects.get(user_id=subscriber)
    
    for issue_activity in issue_activities_created:
        # 根据活动字段和用户偏好决定是否发邮件
        send_email = determine_send_email(issue_activity, preference)
        
        # 构建应用内通知
        bulk_notifications.append(Notification(...))
        
        # 构建邮件通知日志
        if send_email:
            bulk_email_logs.append(EmailNotificationLog(...))
```

**批量落库**：`notification_task.py:669-670`

```python
Notification.objects.bulk_create(bulk_notifications, batch_size=100)
EmailNotificationLog.objects.bulk_create(bulk_email_logs, batch_size=100, ignore_conflicts=True)
```

### 落库点汇总

| 操作 | 表名 | 时机 |
|-----|------|------|
| 应用内通知 | `notifications` | L669 批量创建 |
| 邮件通知日志 | `email_notification_logs` | L670 批量创建 |
| 提及订阅关系 | `issue_subscribers` | L455-459 批量创建 |
| 提及记录 | `issue_mentions` | L662-667 更新 |

---

## 阶段三：客户端轮询与已读标记

### 状态机流转

```
客户端挂载 → useSWR轮询 → 获取通知列表 → 本地状态更新 → 用户操作 → 标记已读/归档/延后
```

### 客户端轮询机制

**轮询入口**：`apps/web/core/components/workspace-notifications/root.tsx:58-63`

```typescript
useSWR(
    currentWorkspace?.slug ? `WORKSPACE_NOTIFICATION_${currentWorkspace?.slug}` : null,
    currentWorkspace?.slug
        ? () => getNotifications(currentWorkspace?.slug, notificationMutation, notificationLoader)
        : null
);
```

使用 `useSWR` 进行自动轮询（默认配置），每次切换tab或过滤器时重新拉取。

**服务层API**：`apps/web/core/services/workspace-notification.service.ts:34-46`

```typescript
async fetchNotifications(workspaceSlug: string, params: ...): Promise<TNotificationPaginatedInfo | undefined> {
    return this.get(`/api/workspaces/${workspaceSlug}/users/notifications`, { params });
}
```

### 后端列表查询

**查询接口**：`apps/api/plane/app/views/notification/base.py:48-149`

```python
def list(self, request, slug):
    # 支持的过滤参数
    # - snoozed: true/false
    # - archived: true/false
    # - read: true/false/null
    # - type: subscribed/assigned/created
    # - mentioned: true/false
    
    notifications = Notification.objects.filter(
        workspace__slug=slug, 
        receiver_id=request.user.id
    )
    # 应用各种过滤器...
```

### 已读标记流程

**前端操作**：`apps/web/core/store/notifications/notification.ts:194-212`

```typescript
markNotificationAsRead = async (workspaceSlug: string): Promise<TNotification | undefined> => {
    const currentNotificationReadAt = this.read_at;
    try {
        // 1. 乐观更新：本地立即标记为已读
        const payload: Partial<TNotification> = { read_at: new Date().toISOString() };
        this.store.workspaceNotification.setUnreadNotificationsCount("decrement");
        runInAction(() => this.mutateNotification(payload));
        
        // 2. 后端请求
        const notification = await workspaceNotificationService.markNotificationAsRead(workspaceSlug, this.id);
        
        // 3. 后端确认后再次同步
        if (notification) {
            runInAction(() => this.mutateNotification(notification));
        }
        return notification;
    } catch (error) {
        // 失败回滚
        runInAction(() => this.mutateNotification({ read_at: currentNotificationReadAt }));
        this.store.workspaceNotification.setUnreadNotificationsCount("increment");
        throw error;
    }
};
```

**后端处理**：`apps/api/plane/app/views/notification/base.py:164-169`

```python
def mark_read(self, request, slug, pk):
    notification = Notification.objects.get(receiver=request.user, workspace__slug=slug, pk=pk)
    notification.read_at = timezone.now()  # 关键状态变更点
    notification.save()
    return Response(serializer.data, status=status.HTTP_200_OK)
```

### 状态字段变更矩阵

| 操作 | 字段变更 | 代码位置 |
|-----|---------|---------|
| 标记已读 | `read_at = now()` | base.py:166 |
| 标记未读 | `read_at = None` | base.py:174 |
| 归档 | `archived_at = now()` | base.py:182 |
| 取消归档 | `archived_at = None` | base.py:190 |
| 延后 | `snoozed_till = date` | base.py:156 |
| 取消延后 | `snoozed_till = None` | base.py:156 |
| 全部已读 | 批量更新 `read_at = now()` | base.py:285 |

---

## 完整状态链路图

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                               事件触发阶段 (后端)                                │
├─────────────────────────────────────────────────────────────────────────────────┤
│  Issue变更/评论创建 → IssueActivity.bulk_create → notifications.delay()         │
│                            ↓                                                    │
│                    订阅者筛选逻辑                                                │
│                    ├─ 项目成员过滤                                              │
│                    ├─ 提及用户识别（描述+评论）                                  │
│                    ├─ 订阅者排除（提及+操作者）                                  │
│                    └─ 负责人识别                                                │
└───────────────────────────────────┬─────────────────────────────────────────────┘
                                    │
┌───────────────────────────────────▼─────────────────────────────────────────────┐
│                              通知生成阶段 (后端)                                 │
├─────────────────────────────────────────────────────────────────────────────────┤
│  遍历接收者 → 构建Notification对象 → 检查UserNotificationPreference              │
│                    ↓                                                           │
│          Notification.bulk_create()  →  EmailNotificationLog.bulk_create()      │
│                    ↓                                                           │
│          落库表: notifications, email_notification_logs                         │
└───────────────────────────────────┬─────────────────────────────────────────────┘
                                    │
┌───────────────────────────────────▼─────────────────────────────────────────────┐
│                            客户端轮询与已读阶段 (前后端)                          │
├─────────────────────────────────────────────────────────────────────────────────┤
│  前端挂载 → useSWR自动轮询 → GET /api/workspaces/{slug}/users/notifications     │
│                    ↓                                                           │
│          本地MobX状态更新（WorkspaceNotificationStore）                          │
│                    ↓                                                           │
│          用户点击已读 → 乐观更新(read_at=now) → POST /read/ → 后端确认           │
│                    ↓                                                           │
│          数据库更新: notifications.read_at = timezone.now()                     │
└─────────────────────────────────────────────────────────────────────────────────┘
```

---

## 关键设计要点

1. **异步处理**：通知生成通过Celery异步任务处理，不阻塞主流程
2. **批量操作**：使用 `bulk_create` 和 `bulk_update` 提高数据库性能
3. **乐观更新**：前端已读操作先本地更新，后端确认失败时回滚
4. **多级过滤**：订阅者→用户偏好→活动类型，三级过滤确保通知相关性
5. **状态分离**：`read_at`/`snoozed_till`/`archived_at` 三个时间戳字段独立控制通知状态
