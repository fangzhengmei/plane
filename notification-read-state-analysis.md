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

### SWR全局配置

**配置定义**：`packages/constants/src/swr.ts:16-22`

```typescript
export const WEB_SWR_CONFIG = {
  refreshWhenHidden: false,       // 页面隐藏时不刷新
  revalidateIfStale: true,        // 数据过期时重新验证
  revalidateOnFocus: true,        // 窗口获得焦点时重新验证
  revalidateOnMount: true,        // 组件挂载时重新验证
  errorRetryCount: 3,             // 错误重试3次
};
```

**配置注入**：`apps/web/app/provider.tsx:50`

```typescript
<SWRConfig value={WEB_SWR_CONFIG}>{children}</SWRConfig>
```

---

### 通知拉取的共同前置链路：未读计数请求

**所有通知拉取路径都会先请求未读计数**，这是一个被遗漏的关键前置步骤。

**代码位置**：`apps/web/core/store/notifications/workspace-notifications.store.ts:337-363`

```typescript
getNotifications = async (
  workspaceSlug: string,
  loader: TNotificationLoader = ENotificationLoader.INIT_LOADER,
  queryParamType: TNotificationQueryParamType = ENotificationQueryParamType.INIT
): Promise<TNotificationPaginatedInfo | undefined> => {
  this.loader = loader;
  try {
    const queryParams = this.generateNotificationQueryParams(queryParamType);
    // ========== 关键前置步骤：先拉取未读计数 ==========
    await this.getUnreadNotificationsCount(workspaceSlug);
    // ========== 然后才拉取通知列表 ==========
    const notificationResponse = await workspaceNotificationService.fetchNotifications(workspaceSlug, queryParams);
    // 更新本地状态...
  } catch (error) {
    console.error(error);
    throw error;
  } finally {
    runInAction(() => (this.loader = undefined));
  }
};
```

**未读计数请求实现**：`workspace-notifications.store.ts:317-329`

```typescript
getUnreadNotificationsCount = async (workspaceSlug: string): Promise<TUnreadNotificationsCount | undefined> => {
  try {
    const unreadNotificationCount = await workspaceNotificationService.fetchUnreadNotificationsCount(workspaceSlug);
    if (unreadNotificationCount)
      runInAction(() => {
        set(this, "unreadNotificationsCount", unreadNotificationCount);
      });
    return unreadNotificationCount || undefined;
  } catch (error) {
    console.error(error);
    throw error;
  }
};
```

**后端未读计数查询**：`apps/api/plane/app/views/notification/base.py:196-229`

```python
class UnreadNotificationEndpoint(BaseAPIView):
    use_read_replica = True  # 使用读库，不影响主库性能

    def get(self, request, slug):
        # 普通未读通知数（排除提及）
        unread_notifications_count = (
            Notification.objects.filter(
                workspace__slug=slug,
                receiver_id=request.user.id,
                read_at__isnull=True,
                archived_at__isnull=True,
                snoozed_till__isnull=True,
            )
            .exclude(sender__icontains="mentioned")
            .count()
        )

        # 提及未读通知数
        mention_notifications_count = Notification.objects.filter(
            workspace__slug=slug,
            receiver_id=request.user.id,
            read_at__isnull=True,
            archived_at__isnull=True,
            snoozed_till__isnull=True,
            sender__icontains="mentioned",
        ).count()

        return Response(
            {
                "total_unread_notifications_count": int(unread_notifications_count),
                "mention_unread_notifications_count": int(mention_notifications_count),
            },
            status=status.HTTP_200_OK,
        )
```

**前置链路时序图**：

```
任何触发getNotifications的操作
       ↓
[前端] 设置loader状态
       ↓
      生成查询参数
       ↓
      1. GET /api/workspaces/{slug}/users/notifications/unread/
       │    ↓
       │  [后端] 读库查询两个count值（普通+提及）
       │    ↓
       └─ 更新本地 unreadNotificationsCount
       ↓
      2. GET /api/workspaces/{slug}/users/notifications
       │    ↓
       │  [后端] 按筛选条件查询通知列表
       │    ↓
       └─ 更新本地 notifications 列表
       ↓
      清除loader状态
```

**落库点**：两次都是只读操作，分别读取 `notifications` 表的 count 和列表数据。

---

### 通知列表拉取的真实触发条件

通知列表拉取共有 **5 条独立触发路径**（排除"全部已读"，因其是写操作），每条路径都遵循上述"先未读计数、后通知列表"的共同前置链路：

#### 路径1：页面挂载（自动触发）

**触发时机**：通知页面组件挂载时，由useSWR自动触发

**代码位置**：`apps/web/core/components/workspace-notifications/root.tsx:50-63`

```typescript
// 根据本地是否已有数据决定加载模式
const notificationMutation =
  currentWorkspace && notificationIdsByWorkspaceId(currentWorkspace.id)
    ? ENotificationLoader.MUTATION_LOADER      // 已有本地数据，增量加载
    : ENotificationLoader.INIT_LOADER;          // 无本地数据，首次加载

const notificationLoader =
  currentWorkspace && notificationIdsByWorkspaceId(currentWorkspace.id)
    ? ENotificationQueryParamType.CURRENT       // 拉取当前最新数据
    : ENotificationQueryParamType.INIT;         // 从头开始拉取

useSWR(
  currentWorkspace?.slug ? `WORKSPACE_NOTIFICATION_${currentWorkspace?.slug}` : null,
  currentWorkspace?.slug
    ? () => getNotifications(currentWorkspace?.slug, notificationMutation, notificationLoader)
    : null
);
```

**SWR自动重验证触发条件**（由WEB_SWR_CONFIG配置）：
- 组件首次挂载（`revalidateOnMount: true`）
- 浏览器窗口重新获得焦点（`revalidateOnFocus: true`）
- 缓存数据过期时（`revalidateIfStale: true`）

**落库点**：只读 `notifications` 表（未读计数 + 通知列表）

---

#### 路径2：筛选切换（主动触发）

**触发时机**：用户切换通知筛选条件时

**代码位置**：`apps/web/core/store/notifications/workspace-notifications.store.ts:234-241`

```typescript
updateFilters = <T extends keyof TNotificationFilter>(key: T, value: TNotificationFilter[T]) => {
  set(this.filters, key, value);
  const { workspaceSlug } = this.store.router;
  if (!workspaceSlug) return;

  set(this, "notifications", {});  // 清空本地缓存
  // 触发重新拉取，使用INIT_LOADER + INIT
  this.getNotifications(workspaceSlug, ENotificationLoader.INIT_LOADER, ENotificationQueryParamType.INIT);
};
```

**同样的触发模式也用于**：
- Tab切换（ALL ↔ MENTIONS）：`setCurrentNotificationTab` L264-272
- 批量筛选更新：`updateBulkFilters` L247-257

**落库点**：只读 `notifications` 表（未读计数 + 通知列表）

---

#### 路径3：主动刷新（用户点击）

**触发时机**：用户点击刷新按钮

**代码位置**：`apps/web/core/components/workspace-notifications/sidebar/header/options/root.tsx:35-42`

```typescript
const refreshNotifications = async () => {
  if (loader) return;  // 防重复提交
  try {
    // 使用MUTATION_LOADER + CURRENT，拉取最新数据但不清空缓存
    await getNotifications(workspaceSlug, ENotificationLoader.MUTATION_LOADER, ENotificationQueryParamType.CURRENT);
  } catch (error) {
    console.error(error);
  }
};
```

**落库点**：只读 `notifications` 表（未读计数 + 通知列表）

---

#### 路径4：分页加载（滚动到底部）

**触发时机**：用户点击"加载更多"按钮

**代码位置**：`apps/web/ce/components/workspace-notifications/notification-card/root.tsx:28-34`

```typescript
const getNextNotifications = async () => {
  try {
    // 使用PAGINATION_LOADER + NEXT，拉取下一页数据
    await getNotifications(workspaceSlug, ENotificationLoader.PAGINATION_LOADER, ENotificationQueryParamType.NEXT);
  } catch (error) {
    console.error(error);
  }
};
```

**分页游标逻辑**：`workspace-notifications.store.ts:180-211`

```typescript
generateNotificationQueryParams = (paramType: TNotificationQueryParamType) => {
  const queryCursorNext =
    paramType === ENotificationQueryParamType.INIT
      ? `${this.paginatedCount}:0:0`                          // 从头开始
      : paramType === ENotificationQueryParamType.CURRENT
        ? `${this.paginatedCount}:${0}:0`                     // 拉取最新
        : paramType === ENotificationQueryParamType.NEXT && this.paginationInfo
          ? this.paginationInfo?.next_cursor                  // 使用后端返回的下一页游标
          : `${this.paginatedCount}:${0}:0`;
  // ...
};
```

**落库点**：只读 `notifications` 表（未读计数 + 通知列表）

---

### 状态回写时序（两类不同模式）

通知系统的状态回写分为 **两种完全不同的模式**：
1. **单条操作模式**：乐观更新 + 失败回滚（标记已读/未读/归档/延后）
2. **批量操作模式**：后端优先 + 无回滚（全部已读）

---

#### 模式一：单条操作（乐观更新 + 失败回滚）

适用于：标记已读、标记未读、归档、取消归档、延后、取消延后

##### 触发入口

**用户点击通知卡片**：`apps/web/core/components/workspace-notifications/sidebar/notification-card/item.tsx:46-66`

```typescript
const handleNotificationIssuePeekOverview = async () => {
  if (workspaceSlug && projectId && issueId && !isSnoozeStateModalOpen && !customSnoozeModal) {
    setPeekIssue(undefined);
    setCurrentSelectedNotificationId(notificationId);

    // 仅当未读时才触发已读标记
    if (notification.read_at === null) {
      try {
        await markNotificationAsRead(workspaceSlug);
      } catch (error) {
        console.error(error);
      }
    }
    // ...
  }
};
```

##### 时序图（乐观更新模式）

```
用户点击通知卡片
       ↓
[前端] markNotificationAsRead() 开始
       ├─ 保存当前 read_at 值用于回滚
       ├─ 乐观更新1：未读数 -1 (setUnreadNotificationsCount("decrement"))
       ├─ 乐观更新2：本地状态 read_at = 当前时间 (mutateNotification)
       ├─ 发送 POST /api/workspaces/{slug}/users/notifications/{id}/read/
       │    ↓
       │  [后端] mark_read() 处理
       │    ├─ 查询 Notification 表验证权限
       │    ├─ notification.read_at = timezone.now()
       │    ├─ notification.save()  ← 落库点
       │    └─ 返回更新后的通知数据
       │
       ├─ 后端返回成功
       │    └─ 同步后端返回的 read_at 时间（覆盖乐观更新值）
       └─ 完成
            ↓
       （如失败）
            ├─ 回滚1：未读数 +1 (setUnreadNotificationsCount("increment"))
            ├─ 回滚2：恢复原始 read_at 值
            └─ 抛出异常
```

##### 详细代码分析

**前端Store层**：`apps/web/core/store/notifications/notification.ts:194-212`

```typescript
markNotificationAsRead = async (workspaceSlug: string): Promise<TNotification | undefined> => {
  const currentNotificationReadAt = this.read_at;  // 保存原始值用于回滚
  try {
    // ========== 乐观更新阶段 ==========
    const payload: Partial<TNotification> = { read_at: new Date().toISOString() };
    this.store.workspaceNotification.setUnreadNotificationsCount("decrement");  // 未读数-1
    runInAction(() => this.mutateNotification(payload));  // 本地立即标记为已读
    
    // ========== 后端请求阶段 ==========
    const notification = await workspaceNotificationService.markNotificationAsRead(workspaceSlug, this.id);
    
    // ========== 后端确认阶段 ==========
    if (notification) {
      runInAction(() => this.mutateNotification(notification));  // 用后端返回值覆盖
    }
    return notification;
  } catch (error) {
    // ========== 失败回滚阶段 ==========
    runInAction(() => this.mutateNotification({ read_at: currentNotificationReadAt }));  // 恢复原始值
    this.store.workspaceNotification.setUnreadNotificationsCount("increment");  // 未读数+1
    throw error;
  }
};
```

**后端API层**：`apps/api/plane/app/views/notification/base.py:164-169`

```python
def mark_read(self, request, slug, pk):
    # 权限校验：只能标记自己收到的通知
    notification = Notification.objects.get(receiver=request.user, workspace__slug=slug, pk=pk)
    notification.read_at = timezone.now()  # 关键状态变更
    notification.save()  # 落库点
    serializer = NotificationSerializer(notification)
    return Response(serializer.data, status=status.HTTP_200_OK)
```

**服务层API**：`apps/web/core/services/workspace-notification.service.ts:64-71`

```typescript
async markNotificationAsRead(workspaceSlug: string, notificationId: string): Promise<TNotification | undefined> {
  try {
    const { data } = await this.post(
      `/api/workspaces/${workspaceSlug}/users/notifications/${notificationId}/read/`
    );
    return data || undefined;
  } catch (error) {
    throw error;
  }
}
```

##### 所有单条操作的回写模式对比

| 操作 | 状态字段 | 前端乐观更新 | 后端落库点 | 失败回滚 |
|-----|---------|------------|-----------|---------|
| 标记已读 | `read_at` | 设为当前时间 | base.py:166 | 恢复原值，未读数+1 |
| 标记未读 | `read_at` | 设为 `undefined` | base.py:174 | 恢复原值，未读数-1 |
| 归档 | `archived_at` | 设为当前时间 | base.py:182 | 恢复原值 |
| 取消归档 | `archived_at` | 设为 `undefined` | base.py:190 | 恢复原值 |
| 延后 | `snoozed_till` | 设为目标时间 | base.py:156 | 恢复原值 |
| 取消延后 | `snoozed_till` | 设为 `undefined` | base.py:156 | 恢复原值 |

**代码示例（归档）**：`notification.ts:244-259`

```typescript
archiveNotification = async (workspaceSlug: string): Promise<TNotification | undefined> => {
  const currentNotificationArchivedAt = this.archived_at;
  try {
    const payload: Partial<TNotification> = { archived_at: new Date().toISOString() };
    runInAction(() => this.mutateNotification(payload));  // 乐观更新
    const notification = await workspaceNotificationService.markNotificationAsArchived(workspaceSlug, this.id);
    if (notification) {
      runInAction(() => this.mutateNotification(notification));
    }
    return notification;
  } catch (error) {
    runInAction(() => this.mutateNotification({ archived_at: currentNotificationArchivedAt }));  // 回滚
    throw error;
  }
};
```

---

#### 模式二：批量操作（后端优先 + 无回滚）

仅适用于：全部已读

##### 触发入口

**用户点击"全部已读"按钮**：`apps/web/core/components/workspace-notifications/sidebar/header/options/root.tsx:44-52`

```typescript
const handleMarkAllNotificationsAsRead = async () => {
  if (loader) return;
  try {
    await markAllNotificationsAsRead(workspaceSlug);
  } catch (error) {
    console.error(error);
  }
};
```

##### 时序图（后端优先模式，无回滚）

```
用户点击全部已读按钮
       ↓
[前端] markAllNotificationsAsRead() 开始
       ├─ 设置 loader = MARK_ALL_AS_READY
       ├─ 生成当前筛选参数
       ├─ 发送 POST /api/workspaces/{slug}/users/notifications/mark-all-read/
       │    ↓
       │  [后端] MarkAllReadNotificationViewSet.create() 处理
       │    ├─ 按筛选条件查询未读通知列表
       │    ├─ 遍历设置 notification.read_at = timezone.now()
       │    ├─ Notification.objects.bulk_update()  ← 落库点
       │    └─ 返回 {"message": "Successful"}
       │
       ├─ 后端返回成功
       │    ├─ 本地更新1：当前Tab未读数清零
       │    └─ 本地更新2：所有本地通知 read_at = 当前时间
       └─ 清除 loader 状态
            ↓
       （如失败）
            ├─ 仅打错误日志 console.error(error)
            ├─ 清除 loader 状态
            └─ 抛出异常（无状态回滚）
```

> **关键区别**：与单条操作不同，"全部已读"是**先请求后端，成功后才更新本地状态**，且失败时**不做任何回滚**。

##### 详细代码分析

**前端Store层**：`apps/web/core/store/notifications/workspace-notifications.store.ts:370-401`

```typescript
markAllNotificationsAsRead = async (workspaceSlug: string): Promise<void> => {
  try {
    this.loader = ENotificationLoader.MARK_ALL_AS_READY;
    const queryParams = this.generateNotificationQueryParams(ENotificationQueryParamType.INIT);
    const params = {
      type: queryParams.type,
      snoozed: queryParams.snoozed,
      archived: queryParams.archived,
      read: queryParams.read,
    };
    // ========== 先请求后端 ==========
    await workspaceNotificationService.markAllNotificationsAsRead(workspaceSlug, params);
    
    // ========== 后端成功后才更新本地 ==========
    runInAction(() => {
      update(
        this.unreadNotificationsCount,
        this.currentNotificationTab === ENotificationTab.ALL
          ? "total_unread_notifications_count"
          : "mention_unread_notifications_count",
        () => 0  // 未读数清零
      );
      Object.values(this.notifications).forEach((notification) =>
        notification.mutateNotification({
          read_at: new Date().toUTCString(),  // 所有本地通知标记为已读
        })
      );
    });
  } catch (error) {
    // ========== 失败时仅打日志，无回滚 ==========
    console.error("WorkspaceNotificationStore -> markAllNotificationsAsRead -> error", error);
    throw error;
  } finally {
    runInAction(() => (this.loader = undefined));
  }
};
```

**后端处理**：`apps/api/plane/app/views/notification/base.py:232-288`

```python
class MarkAllReadNotificationViewSet(BaseViewSet):
    def create(self, request, slug):
        snoozed = request.data.get("snoozed", False)
        archived = request.data.get("archived", False)
        type = request.data.get("type", "all")

        # 按筛选条件查询未读通知
        notifications = (
            Notification.objects.filter(workspace__slug=slug, receiver_id=request.user.id, read_at__isnull=True)
            .select_related("workspace", "project", "triggered_by", "receiver")
            .order_by("snoozed_till", "-created_at")
        )

        # 应用各种筛选条件（snoozed/archived/type）...

        # 批量更新
        updated_notifications = []
        for notification in notifications:
            notification.read_at = timezone.now()
            updated_notifications.append(notification)
        Notification.objects.bulk_update(updated_notifications, ["read_at"], batch_size=100)  # 落库点
        return Response({"message": "Successful"}, status=status.HTTP_200_OK)
```

**服务层API**：`apps/web/core/services/workspace-notification.service.ts:109-119`

```typescript
async markAllNotificationsAsRead(
  workspaceSlug: string,
  payload: TNotificationPaginatedInfoQueryParams
): Promise<TNotification | undefined> {
  try {
    const { data } = await this.post(
      `/api/workspaces/${workspaceSlug}/users/notifications/mark-all-read/`,
      payload
    );
    return data || undefined;
  } catch (error) {
    throw error;
  }
}
```

---

### 拉取路径汇总表

| 路径 | 触发源 | Loader类型 | QueryParam类型 | 清空缓存 | 落库操作 | 模式 |
|-----|-------|-----------|---------------|---------|---------|------|
| 页面挂载 | useSWR自动 | INIT_LOADER / MUTATION_LOADER | INIT / CURRENT | 否 | 读（未读计数+通知列表） | 只读 |
| 筛选切换 | updateFilters | INIT_LOADER | INIT | 是 | 读（未读计数+通知列表） | 只读 |
| Tab切换 | setCurrentNotificationTab | INIT_LOADER | INIT | 是 | 读（未读计数+通知列表） | 只读 |
| 主动刷新 | 刷新按钮 | MUTATION_LOADER | CURRENT | 否 | 读（未读计数+通知列表） | 只读 |
| 分页加载 | 加载更多 | PAGINATION_LOADER | NEXT | 否 | 读（未读计数+通知列表） | 只读 |
| 单条已读 | 点击通知卡片 | - | - | 否 | 单条更新read_at | 乐观更新+回滚 |
| 全部已读 | 全部已读按钮 | MARK_ALL_AS_READY | - | 否 | 批量更新read_at | 后端优先+无回滚 |

---

## 关键一致性分析：未读计数与列表的时序及并发场景

### 两次请求的严格时间顺序

**代码证据**：`workspace-notifications.store.ts:337-363`

```typescript
getNotifications = async (workspaceSlug, loader, queryParamType) => {
  this.loader = loader;
  try {
    const queryParams = this.generateNotificationQueryParams(queryParamType);
    // ========== 第一步：未读计数请求（串行等待） ==========
    await this.getUnreadNotificationsCount(workspaceSlug);
    // ========== 第二步：通知列表请求（必须等第一步完成） ==========
    const notificationResponse = await workspaceNotificationService.fetchNotifications(workspaceSlug, queryParams);
    // 更新本地状态...
  } catch (error) {
    console.error(error);
    throw error;
  } finally {
    runInAction(() => (this.loader = undefined));
  }
};
```

**关键结论**：两次请求是**串行执行**而非并行，`await` 关键字确保了：
1. 必须等未读计数请求完成（成功或失败）后，才会发起通知列表请求
2. 未读计数更新早于通知列表更新，两者存在时间差（网络RTT + 后端处理时间）

---

### 前端本地状态更新的两条独立路径

未读计数的更新有 **两条互不相关的路径**：

#### 路径A：拉取时的全量更新

```
getNotifications 触发
       ↓
await getUnreadNotificationsCount() → GET /unread/
       ↓
[后端] 读库查询最新count值
       ↓
set(this, "unreadNotificationsCount", unreadNotificationCount)
       ↓
      GET /notifications
```

**代码位置**：`workspace-notifications.store.ts:317-329`

```typescript
getUnreadNotificationsCount = async (workspaceSlug) => {
  const unreadNotificationCount = await workspaceNotificationService.fetchUnreadNotificationsCount(workspaceSlug);
  if (unreadNotificationCount)
    runInAction(() => {
      set(this, "unreadNotificationsCount", unreadNotificationCount);  // 全量覆盖
    });
  return unreadNotificationCount || undefined;
};
```

#### 路径B：操作时的增量更新

```
用户执行单条操作（标记已读/未读）
       ↓
setUnreadNotificationsCount("decrement" | "increment")
       ↓
update(this.unreadNotificationsCount, countKey, 
  (count) => +Math.max(0, type === "increment" ? count + 1 : count - 1)
)
```

**代码位置**：`workspace-notifications.store.ts:288-309`

```typescript
setUnreadNotificationsCount = (type: "increment" | "decrement", newCount: number = 1): void => {
  const validCount = Math.max(0, Math.abs(newCount));
  switch (this.currentNotificationTab) {
    case ENotificationTab.ALL:
      update(this.unreadNotificationsCount, "total_unread_notifications_count",
        (count) => +Math.max(0, type === "increment" ? count + validCount : count - validCount)
      );
      break;
    case ENotificationTab.MENTIONS:
      update(this.unreadNotificationsCount, "mention_unread_notifications_count",
        (count) => +Math.max(0, type === "increment" ? count + validCount : count - validCount)
      );
      break;
  }
};
```

**关键区别**：
- 路径A（拉取）：**全量覆盖**，直接用后端返回值替换本地状态
- 路径B（操作）：**增量修改**，在当前值基础上 ±1

---

### 并发场景下的不一致性分析

由于存在两条独立的更新路径，在并发操作时可能出现 **短暂的计数与列表不一致**。

#### 场景1：拉取过程中并发标记已读

**时序图**：

```
时间轴 →
  │
  ├─ T0: 触发 getNotifications()
  │    ├─ 发起 GET /unread/
  │    │  [后端查询T0时刻count = 10]
  │    │
  ├─ T1: 用户点击通知卡片（并发）
  │    ├─ 乐观更新：count = 10 - 1 = 9
  │    ├─ 发起 POST /read/
  │    │  [后端更新DB，read_at = now()]
  │    │
  │    ├─ T0的 /unread/ 返回，count = 10
  │    ├─ 全量覆盖：count = 10  ← 不一致！
  │    │
  │    ├─ 发起 GET /notifications/
  │    │  [后端返回列表，该通知已标记已读]
  │    │
  │    └─ 更新列表：通知显示已读
  │
  └─ T2: 状态呈现
       ├─ 未读计数显示：10（错误，应为9）
       └─ 通知列表显示：该通知已读（正确）
```

**不一致表现**：
- 未读计数：10（来自T0时刻的查询结果，已过期）
- 通知列表：该通知已读（来自最新的列表查询）
- 视觉上：用户看到"10条未读"但有一条通知显示已读

**代码证据**：`notification.ts:194-212` 中乐观更新在 POST 请求之前，而 `getNotifications` 中未读计数请求也在进行中，两者竞争更新同一个 `unreadNotificationsCount` 对象。

---

#### 场景2：筛选切换与全部已读并发

**时序图**：

```
时间轴 →
  │
  ├─ T0: 用户点击"全部已读"
  │    ├─ 发起 POST /mark-all-read/
  │    │  [后端批量更新所有通知read_at]
  │    │
  ├─ T1: 用户切换筛选条件（并发）
  │    ├─ 清空本地缓存：set(this, "notifications", {})
  │    ├─ 发起 getNotifications()
  │    │  ├─ GET /unread/  [后端查询，可能还在更新中]
  │    │  ├─ GET /notifications/
  │    │
  │    ├─ "全部已读"的POST返回成功
  │    ├─ 本地更新：count = 0
  │    │
  │    ├─ /unread/ 返回 count = 5（读库延迟）
  │    ├─ 全量覆盖：count = 5  ← 不一致！
  │
  └─ T2: 状态呈现
       ├─ 未读计数显示：5（错误，应为0）
       └─ 通知列表显示：已按新筛选条件加载
```

**不一致表现**：
- 未读计数：5（来自读库的延迟数据）
- 全部已读操作已完成，实际应为0
- 通知列表：已按新筛选条件加载

**代码证据**：`workspace-notifications.store.ts:370-401` 中"全部已读"失败时无回滚，而 `updateFilters` L234-241 会清空缓存并重新拉取，两者的更新顺序不确定。

---

#### 场景3：多次快速标记已读

**时序图**：

```
时间轴 →
  │
  ├─ T0: 初始 count = 10
  │
  ├─ T1: 点击通知A → 乐观更新 count = 9 → POST /read/A
  │
  ├─ T2: 点击通知B → 乐观更新 count = 8 → POST /read/B
  │
  ├─ T3: 触发SWR重验证（窗口聚焦）
  │    ├─ GET /unread/  [后端查询count = 9，A已更新但B还在处理]
  │    ├─ 全量覆盖：count = 9  ← 不一致！
  │
  ├─ T4: POST /read/B 返回成功
  │    └─ 同步状态，但count已被覆盖为9
```

**不一致表现**：
- 未读计数：9（应为8）
- 两条通知都已标记已读

---

### 不一致的纠正机制

前端通过 **多层纠正机制** 确保不一致是**短暂的**，最终会收敛到一致状态：

#### 机制1：下一次拉取的全量覆盖

任何触发 `getNotifications` 的操作都会先调用 `getUnreadNotificationsCount`，而后者是**全量覆盖**本地状态：

```typescript
// getUnreadNotificationsCount 内部
set(this, "unreadNotificationsCount", unreadNotificationCount);  // 直接替换，不是增量
```

**触发时机**：
- 页面挂载（useSWR自动）
- 筛选切换（updateFilters）
- Tab切换（setCurrentNotificationTab）
- 主动刷新（刷新按钮）
- 分页加载（加载更多）
- SWR自动重验证（窗口聚焦、缓存过期）

#### 机制2：后端返回值覆盖乐观更新

单条操作中，后端返回成功后会用后端的真实数据覆盖乐观更新值：

```typescript
// markNotificationAsRead 内部
const notification = await workspaceNotificationService.markNotificationAsRead(workspaceSlug, this.id);
if (notification) {
  runInAction(() => this.mutateNotification(notification));  // 用后端返回值覆盖
}
```

**关键**：通知列表的状态会被后端确认后的值覆盖，确保列表数据的准确性。

#### 机制3：SWR自动重验证

SWR配置 `revalidateOnFocus: true` 和 `revalidateIfStale: true` 确保：
- 窗口重新获得焦点时自动重拉取
- 缓存过期后自动重拉取
- 每次重拉取都会先拉未读计数，再拉通知列表

**代码位置**：`packages/constants/src/swr.ts:16-22`

```typescript
export const WEB_SWR_CONFIG = {
  revalidateIfStale: true,
  revalidateOnFocus: true,
  revalidateOnMount: true,
  errorRetryCount: 3,
};
```

#### 机制4：操作成功后的隐式同步

虽然代码中没有显式在操作成功后调用 `getNotifications`，但：
1. 单条操作成功后，后端返回的通知数据会更新本地列表（`mutateNotification`）
2. 下一次用户交互（如滚动、切换Tab）会触发拉取
3. SWR的自动重验证会在后台静默纠正

---

### 一致性保证分析

| 场景 | 不一致持续时间 | 最终一致性保证 | 关键机制 |
|-----|--------------|--------------|---------|
| 拉取中标记已读 | 短暂（直到下一次拉取） | ✓ 是 | 全量覆盖 + SWR重验证 |
| 筛选与全部已读并发 | 短暂（直到下一次拉取） | ✓ 是 | 全量覆盖 + SWR重验证 |
| 多次快速标记已读 | 短暂（直到下一次拉取） | ✓ 是 | 全量覆盖 + 后端返回覆盖 |
| 未读计数与列表数据差 | 两次请求之间的RTT | ✓ 是 | 串行请求确保计数先更新 |

**设计权衡**：
- 牺牲了**强一致性**（允许短暂不一致）
- 获得了**更好的用户体验**（乐观更新无等待）
- 通过**最终一致性**（多层纠正机制）确保数据最终正确
- 不一致仅影响未读计数的数字显示，不影响通知列表的实际状态

---

### 状态字段变更矩阵

| 操作 | 字段变更 | 代码位置 | 落库表 | 模式 |
|-----|---------|---------|-------|------|
| 标记已读 | `read_at = now()` | base.py:166 | notifications | 乐观更新+回滚 |
| 标记未读 | `read_at = None` | base.py:174 | notifications | 乐观更新+回滚 |
| 归档 | `archived_at = now()` | base.py:182 | notifications | 乐观更新+回滚 |
| 取消归档 | `archived_at = None` | base.py:190 | notifications | 乐观更新+回滚 |
| 延后 | `snoozed_till = date` | base.py:156 | notifications | 乐观更新+回滚 |
| 取消延后 | `snoozed_till = None` | base.py:156 | notifications | 乐观更新+回滚 |
| 全部已读 | 批量更新 `read_at = now()` | base.py:287 | notifications | 后端优先+无回滚 |
| 未读计数查询 | count查询 | base.py:202-221 | notifications（读） | 前置只读 |

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
│  ┌───────────────────────────────────────────────────────────────────────────┐  │
│  │                      共同前置：未读计数请求链路                             │  │
│  ├───────────────────────────────────────────────────────────────────────────┤  │
│  │  任何拉取触发 → GET /unread/ → 更新unreadNotificationsCount → GET /notifications │  │
│  └───────────────────────────────────┬───────────────────────────────────────┘  │
│                                      │                                          │
│  ┌───────────────────────────────────▼───────────────────────────────────────┐  │
│  │                           通知列表拉取路径                                 │  │
│  ├───────────────────────────────────────────────────────────────────────────┤  │
│  │  1. 页面挂载: useSWR自动 → GET /notifications → MobX状态更新               │  │
│  │  2. 筛选切换: updateFilters → 清空缓存 → GET /notifications                 │  │
│  │  3. 主动刷新: 刷新按钮 → GET /notifications (CURRENT模式)                   │  │
│  │  4. 分页加载: 加载更多 → GET /notifications (NEXT游标)                      │  │
│  └───────────────────────────────────┬───────────────────────────────────────┘  │
│                                      │                                          │
│  ┌───────────────────────────────────▼───────────────────────────────────────┐  │
│  │                        状态回写（两类不同模式）                              │  │
│  ├───────────────────────────────────────────────────────────────────────────┤  │
│  │  单条操作(乐观更新+回滚)：                                                 │  │
│  │    点击通知 → 乐观更新 → POST /read/ → 后端save() → 同步状态                │  │
│  │                              ↓(失败)                                        │  │
│  │                         回滚原始状态                                        │  │
│  │                                                                           │  │
│  │  全部已读(后端优先+无回滚)：                                                │  │
│  │    点击全部已读 → POST /mark-all-read/ → 后端bulk_update → 本地更新          │  │
│  │                              ↓(失败)                                        │  │
│  │                         仅打日志，无回滚                                     │  │
│  └───────────────────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────────────┘
```

---

## 关键设计要点

1. **异步处理**：通知生成通过Celery异步任务处理，不阻塞主流程
2. **批量操作**：使用 `bulk_create` 和 `bulk_update` 提高数据库性能
3. **两种回写模式**：
   - 单条操作：乐观更新 + 失败回滚，提升用户体验
   - 批量操作：后端优先 + 无回滚，确保数据一致性
4. **共同前置链路**：所有通知拉取前必先请求未读计数，确保计数与列表数据一致
5. **读库分离**：未读计数查询使用读库（`use_read_replica = True`），不影响主库性能
6. **多级过滤**：订阅者→用户偏好→活动类型，三级过滤确保通知相关性
7. **状态分离**：`read_at`/`snoozed_till`/`archived_at` 三个时间戳字段独立控制通知状态
8. **SWR自动重验证**：利用SWR的revalidateOnFocus/revalidateOnMount特性实现静默刷新
9. **防重复提交**：所有用户操作都通过 `loader` 状态防止重复请求
10. **游标分页**：使用游标模式而非页码分页，提升大数据量下的查询性能
11. **最终一致性设计**：
    - 未读计数采用两条独立更新路径（全量覆盖 + 增量修改）
    - 允许短暂的计数与列表不一致，通过多层纠正机制最终收敛
    - 牺牲强一致性换取更好的用户体验（乐观更新无等待）
12. **串行请求保证**：未读计数与通知列表请求串行执行（`await` 关键字），确保计数先更新
