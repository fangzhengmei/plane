# Plane 实时协作通道 — 状态同步机制梳理

## 一、整体架构概览

Plane 的"实时协作"分为**两条独立通道**，服务于不同的业务场景：

| 通道 | 覆盖范围 | 传输协议 | 数据模型 |
|------|----------|----------|----------|
| **Hocuspocus/Y.js 通道** | Page 文档编辑（富文本 + 标题） | WebSocket → Y.js CRDT | Y.Doc 二进制增量 |
| **MobX 全局 store + 按需拉取通道** | Issue 详情字段（state/priority/assignee/label 等） | HTTP PATCH → MobX 乐观更新 + SWR 按需 revalidation | JSON 字段级覆盖 + 浅合并 |

> **关键发现：Issue 详情的元数据字段（state、priority、assignee 等）目前不走 WebSocket 实时通道。** 客户端以 **MobX 全局 `issuesMap`** 为单一数据源，字段的读取和写入全部经过它；SWR 仅充当取数调度器，组件不直接消费 SWR 的 `data`。**这是字段闪烁问题的核心上下文。**

---

## 二、Hocuspocus/Y.js 通道（Page 文档协作）

### 2.1 服务端架构

服务端入口在 `apps/live/` 目录，核心组件：

```
Server (express + express-ws)
  └─ HocusPocusServerManager (单例)
       └─ Hocuspocus Server
            ├─ onAuthenticate       ← 认证
            ├─ onStateless          ← 无状态消息转发
            └─ Extensions:
                 ├─ Logger          ← 日志
                 ├─ Database        ← 文档持久化 (fetch/store)
                 ├─ Redis           ← 跨节点消息广播
                 ├─ TitleSyncExtension  ← 标题变更检测 & 持久化
                 └─ ForceCloseHandler  ← 管理员强制关闭连接
```

相关文件：
- [server.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/197-plane/apps/live/src/server.ts)
- [hocuspocus.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/197-plane/apps/live/src/hocuspocus.ts)
- [extensions/index.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/197-plane/apps/live/src/extensions/index.ts)

### 2.2 客户端连接建立流程

1. **构造 WebSocket URL**：在 [editor-body.tsx](file:///d:/fz/0508-3/solo-dogfeeding/code/197-plane/apps/web/core/components/pages/editor/editor-body.tsx#L190-L214) 中，根据 `LIVE_BASE_URL` 拼接 `ws(s)://<host>/collaboration` 路径，附加 `webhookConnectionParams`（workspaceSlug、projectId、documentType 等）作为 query 参数。

2. **传入认证信息**：`userConfig` 包含 `id`、`name`、`color`，作为 WebSocket token 传给 Hocuspocus 的 `onAuthenticate` 钩子。

3. **服务端认证**：[auth.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/197-plane/apps/live/src/lib/auth.ts) 中解析 token 获取 `userId` 和 `cookie`，通过 `UserService.currentUser(cookie)` 验证身份，校验 `userId` 一致性后放行连接。

4. **认证成功后**，Hocuspocus 触发 `onLoadDocument`：
   - [database.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/197-plane/apps/live/src/extensions/database.ts#L26-L70) 调用 `PageCoreService.fetchDescriptionBinary()` 获取文档二进制数据。
   - 若二进制为空，回退到 HTML 格式并自动转换。
   - 文档加载后，Y.js 协议接管增量同步。

5. **连接状态回调**：[editor-body.tsx](file:///d:/fz/0508-3/solo-dogfeeding/code/197-plane/apps/web/core/components/pages/editor/editor-body.tsx#L169-L188) 通过 `serverHandler.onStateChange` 将协作阶段映射为 UI 状态：
   - `disconnected` → `error`
   - `synced` → `synced`
   - 其他（`initial`/`connecting`/`awaiting-sync`/`reconnecting`）→ `syncing`

### 2.3 事件粒度

#### 2.3.1 文档内容（description）

Y.js 通过 CRDT 增量同步，粒度细到**字符级别**。Hocuspocus 的 `debounce: 10000`（[hocuspocus.ts#L59](file:///d:/fz/0508-3/solo-dogfeeding/code/197-plane/apps/live/src/hocuspocus.ts#L59)）控制的是**文档持久化到数据库的间隔**（10 秒），而非客户端间同步的间隔——客户端间同步是实时的。

#### 2.3.2 标题（title）

标题通过 Y.js 的 `XmlFragment("title")` 共享，变更粒度也是字符级。

[TitleSyncExtension](file:///d:/fz/0508-3/solo-dogfeeding/code/197-plane/apps/live/src/extensions/title-sync.ts) 在 `afterLoadDocument` 中对 `title` 字段注册 `observeDeep` 监听器，当标题变更时：
1. 提取文本内容
2. 如果有父页面，向父页面广播 `property_updated` 事件
3. 通过 [TitleUpdateManager](file:///d:/fz/0508-3/solo-dogfeeding/code/197-plane/apps/live/src/extensions/title-update/title-update-manager.ts) 以 **5 秒 debounce** 调用 REST API 持久化标题

#### 2.3.3 Stateless 消息（协作事件）

[DocumentCollaborativeEvents](file:///d:/fz/0508-3/solo-dogfeeding/code/197-plane/packages/editor/src/core/constants/document-collaborative-events.ts) 定义了 Page 级别的无状态协作事件：

| 事件键 | 客户端名 | 服务端名 | Payload |
|--------|----------|----------|---------|
| `lock` | `locked` | `lock` | `{ is_locked }` |
| `unlock` | `unlocked` | `unlock` | — |
| `archive` | `archived` | `archive` | `{ archived_at }` |
| `unarchive` | `unarchived` | `unarchive` | — |
| `make-public` | `made-public` | `make-public` | `{ access }` |
| `make-private` | `made-private` | `make-private` | `{ access }` |
| `delete` | `deleted` | `delete` | `{ deleted_at }` |
| `move` | `moved` | `move` | `{ new_project_id, new_page_id }` |
| `duplicate` | `duplicated` | `duplicate` | `{ new_page_id }` |
| `property_update` | `property_updated` | `property_update` | `Partial<TPage>` |
| `restore` | `restored` | `restore` | `{ deleted_page_ids }` |
| `error` | `error` | `error` | `{ error_message, error_type, error_code }` |

这些事件的流转路径：

```
客户端 A 操作 → emitRealTimeUpdate(serverEvent) → WebSocket → 服务端 onStateless
  → DocumentCollaborativeEvents[payload].client → document.broadcastStateless(clientEvent)
  → 客户端 B/C 收到 stateless 消息 → useCollaborativePageActions 处理
```

[stateless.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/197-plane/apps/live/src/lib/stateless.ts) 中的逻辑：将客户端发送的 server 端事件名映射为 client 端事件名，然后 `broadcastStateless` 给所有连接。

### 2.4 冲突合并策略（Page 文档）

Y.js CRDT 天然支持无冲突合并——所有操作都带有 Lamport 时间戳和向量时钟，保证最终一致性。具体表现：

- **文档内容**：Y.js 自动合并，无需额外冲突处理。
- **标题**：同上，Y.js CRDT 合并。
- **无状态事件**（lock/archive/delete 等）：**不合并，直接执行**。由 [useCollaborativePageActions](file:///d:/fz/0508-3/solo-dogfeeding/code/197-plane/apps/web/core/hooks/use-collaborative-page-actions.tsx) 处理——收到消息后直接调用 page store 的对应方法（`page.lock()`、`page.archive()` 等），没有冲突检测。

  > 注意：`currentActionBeingProcessed` 机制仅用于**跳过自己发出的回声事件**，不处理两人同时操作的冲突。

### 2.5 断线重连后的状态补齐

Hocuspocus 内置了断线重连机制：

1. **连接断开时**：`serverHandler.onStateChange` 报告 `stage.kind === "disconnected"`，UI 显示 `error` 状态。
2. **重连时**：Hocuspocus 客户端自动重新连接 WebSocket，进入 `reconnecting` → `awaiting-sync` 阶段。
3. **状态补齐**：重连后 Hocuspocus 执行 Y.js 的**状态同步协议**：
   - 客户端发送本地 `sv`（state vector）
   - 服务端计算差异并返回缺失的更新
   - 客户端应用差异后进入 `synced` 状态

4. **文档级别**：如果服务端内存中已无该文档（所有连接都已断开，文档被 unload），重连时会触发 `onLoadDocument` 重新从数据库加载最新二进制，然后与客户端本地状态合并。

> **局限**：断线期间的 **stateless 事件**（如别人删除/归档了页面）不会被补发。重连后只能通过 Y.js 同步文档内容变更，无法恢复断线期间丢失的 stateless 消息。

### 2.6 跨节点广播

[Redis Extension](file:///d:/fz/0508-3/solo-dogfeeding/code/197-plane/apps/live/src/extensions/redis.ts) 继承 `@hocuspocus/extension-redis`，用于多 live-server 节点间的消息同步：

- **Y.js 更新**：由 `@hocuspocus/extension-redis` 原生处理，通过 Redis Pub/Sub 在节点间转发。
- **Stateless 广播**：自定义 `broadcastToDocument` 方法（[redis.ts#L126-L140](file:///d:/fz/0508-3/solo-dogfeeding/code/197-plane/apps/live/src/extensions/redis.ts#L126-L140)），通过 Redis channel 将消息发送给所有持有该文档的节点。
- **管理命令**：`hocuspocus:admin` channel 用于 `FORCE_CLOSE` 等管理操作，由 [ForceCloseHandler](file:///d:/fz/0508-3/solo-dogfeeding/code/197-plane/apps/live/src/extensions/force-close-handler.ts) 处理。

---

## 三、MobX 全局 store + 按需拉取通道（Issue 详情字段）

### 3.1 核心数据源：`issuesMap`

Issue 详情的所有元数据字段（state、priority、assignee、labels、start_date、target_date 等）以 **MobX 全局 `issuesMap`** 为单一数据源。[IssueStore](file:///d:/fz/0508-3/solo-dogfeeding/code/197-plane/apps/web/core/store/issue/issue.store.ts) 维护 `issuesMap: Record<issue_id, TIssue>`，所有组件（列表、详情、Peek 概览）通过 `getIssueById()` 从同一个 map 读取。

写入 `issuesMap` 的路径有以下几种：

| 写入路径 | 触发方 | 合并方式 | 代码位置 |
|----------|--------|----------|----------|
| `addIssue(issues)` | 页面加载、SWR revalidation、列表刷新 | 浅合并 `{ ...prevIssue, ...issue }`（已存在时） | [issue.store.ts#L61-L75](file:///d:/fz/0508-3/solo-dogfeeding/code/197-plane/apps/web/core/store/issue/issue.store.ts#L61-L75) |
| `updateIssue(issueId, partial)` | 乐观更新（字段级覆盖） | 逐字段 `set(this.issuesMap, [issueId, key], value)` | [issue.store.ts#L108-L116](file:///d:/fz/0508-3/solo-dogfeeding/code/197-plane/apps/web/core/store/issue/issue.store.ts#L108-L116) |
| `removeIssue(issueId)` | 删除 issue | 直接 delete | [issue.store.ts#L123-L128](file:///d:/fz/0508-3/solo-dogfeeding/code/197-plane/apps/web/core/store/issue/issue.store.ts#L123-L128) |

**关键细节**：`addIssue` 对已存在的 issue 做 **浅合并** `{ ...prevIssue, ...issue }`，这会直接用服务端返回的每个字段覆盖本地值。而 `updateIssue` 只覆盖传入的 **部分字段**，其余字段不变。两种合并方式的不一致是闪烁的潜在来源之一。

### 3.2 SWR 的角色：取数调度器，不是状态管理器

SWR 在 Issue 详情中 **不直接持有组件消费的数据**，它仅作为取数调度器——触发 MobX store 的 fetch 方法，fetch 返回的数据写入 `issuesMap` 后由 MobX 驱动 UI 更新。

各页面的 SWR 配置：

| 页面 | SWR Key | revalidateOnFocus | revalidateOnReconnect | revalidateIfStale | refreshInterval |
|------|---------|-------------------|----------------------|-------------------|-----------------|
| Issue 详情页 (browse) | `ISSUE_DETAIL_...` | **true**（默认） | **true**（默认） | true（默认） | 无 |
| 归档 Issue 详情 | `ARCHIVED_ISSUE_DETAIL_...` | **true**（默认） | **true**（默认） | true（默认） | 无 |
| Peek 概览 (列表内弹窗) | `peek-issue-...` | **false** | **false** | **false** | 无 |
| 列表布局 (project/cycle/module) | 各自 key | **false** | — | **false** | 无 |

**要点**：
- 只有 Issue 详情页的 SWR 允许 `revalidateOnFocus` 和 `revalidateOnReconnect`——用户切换标签页回来或网络恢复时会自动触发 `fetchIssueWithIdentifier()`。
- Peek 概览和列表布局显式关闭了所有 SWR revalidation，加载后不再主动拉取。
- **没有 `refreshInterval`**，即没有定时轮询。SWR 仅在 focus/reconnect 事件时触发拉取。

### 3.3 字段修改的完整数据流

#### 3.3.1 编辑用户（User A）的操作路径

`IssueDetail.updateIssue`（[issue.store.ts#L181-L191](file:///d:/fz/0508-3/solo-dogfeeding/code/197-plane/apps/web/core/store/issue/issue-details/issue.store.ts#L181-L191)）使用 **`Promise.all` 并行发起**两个字操作：

```typescript
updateIssue = async (workspaceSlug, projectId, issueId, data) => {
  const currentStore = ...; // ProjectIssues 或 ProjectEpics
  await Promise.all([
    currentStore.updateIssue(workspaceSlug, projectId, issueId, data),  // 分支 A
    this.rootIssueDetailStore.activity.fetchActivities(workspaceSlug, projectId, issueId),  // 分支 B
  ]);
};
```

**JavaScript 微任务层面的精确调用发起顺序**：`Promise.all` 接收两个 Promise，它们的执行函数在**同一个微任务**中依次启动：

```
① 分支 A 同步执行至第一个 await：
   A-1 clone(getIssueById) 保存快照                              ← 同步
   A-2 rootIssueStore.issues.updateIssue(issueId, data)          ← 乐观写入 issuesMap（同步，UI 立即重渲染）
   A-3 updateIssueList(...)                                      ← 更新列表分组 ID（同步）
   A-4 updateParentStats(...)                                    ← 乐观更新 projectMap 统计（同步）
   A-5 await this.issueService.patchIssue(...)                   ← 遇到 await，挂起分支 A

② 分支 B 同步执行至第一个 await：
   B-1 检查已有活动记录，取最后一条的 created_at                  ← 同步
   B-2 await this.issueActivityService.getIssueActivities(...)   ← 遇到 await，挂起分支 B

③ 此时分支 A 的 PATCH 请求和分支 B 的 GET/activities 请求
   已在浏览器网络栈中排队，**几乎同时发出**（无先后保证）
```

**分支 A 剩余部分**（PATCH 返回 204 后恢复执行）：

```
A-6 fetchParentStats(...)                                       ← PATCH 成功后才执行，不 await（fire-and-forget）
A-7 [如果 A-5 失败] rootIssueStore.issues.updateIssue(issueBeforeUpdate) ← 回滚
```

**关键观察**：
- 步骤 A-2 立即修改 `issuesMap` → observer 组件即时重渲染 → UI 无延迟
- 步骤 A-5 PATCH 成功后 **不会** 重新 fetch 该 issue 来拿服务端计算的字段（如 `updated_at`、`completed_at`），本地保留的是乐观值
- 步骤 A-5 PATCH 成功后也不会调 `addIssue()` 做服务端数据的浅合并回写
- 步骤 A-6 `fetchParentStats` 调用 [project.fetchProjectDetails](file:///d:/fz/0508-3/solo-dogfeeding/code/197-plane/apps/web/core/store/project/project.store.ts#L364-L375)，发出 `GET /api/.../projects/:id/`，返回值写入 **`projectMap`**（非 `issuesMap`），只修正项目的 `total_issues`/`cancelled_issues`/`completed_issues` 等统计字段
- 步骤 B-2 的 `fetchActivities` 与 PATCH **几乎同时发出**——详见第四章时序分析

#### 3.3.2 其他用户（User B）看到变更的路径

User B 看到 User A 的修改，**只能通过以下时机触发数据拉取**：

| 触发时机 | 机制 | 覆盖页面 |
|----------|------|----------|
| 切换标签页回来 | SWR `revalidateOnFocus` | 仅 Issue 详情页 |
| 网络恢复 | SWR `revalidateOnReconnect` | 仅 Issue 详情页 |
| 列表滚动/翻页 | `fetchNextIssues` → `addIssue()` | 列表布局 |
| 手动刷新 | 整页重新加载 | 所有页面 |
| 其他组件触发 fetch | 如 activity、comment 等 fetch 后不更新 issuesMap | 不影响字段 |

拉取后的写入路径：

```
SWR revalidation → fetchIssueWithIdentifier()
  → this.issueService.retrieveWithIdentifier()          ← REST GET
  → addIssueToStore(issue) → rootIssueStore.issues.addIssue([issuePayload])
    → issuesMap[id] 已存在 → { ...prevIssue, ...serverIssue }  ← 浅合并覆盖
```

**关键**：`addIssue` 的浅合并会用服务端返回的 **全部字段** 覆盖本地值，包括 User B 当前正在编辑但尚未提交的字段——这是并发覆盖的根因。

#### 3.3.3 Issue 描述（description_html）的特殊路径

Issue 描述不走 Hocuspocus 协作编辑，而是使用 [DescriptionInput](file:///d:/fz/0508-3/solo-dogfeeding/code/197-plane/apps/web/core/components/editor/rich-text/description-input/root.tsx) 组件：

- 使用 `RichTextEditor`（非协作编辑器），纯本地编辑
- **1500ms debounce** 自动保存：`debouncedFormSave = debounce(async () => { handleSubmit(...) }, 1500)`
- 保存时调用 `issueOperations.update()` → REST PATCH → `description_html` 字段
- 组件卸载时如有未保存变更，会立即保存
- `swrDescription` prop 存在但未被 Issue 详情页使用——该 prop 用于外部注入实时描述内容，当前 Issue 详情的 `DescriptionInput` 并未传入

**描述与 `issuesMap` 的交互**：
1. `DescriptionInput` 的 `initialValue` 读取自 `issue.description_html`（来自 `issuesMap`）
2. `useEffect` 监听 `initialValue` 变化 → 调用 `reset()` 重置表单 + `setLocalDescription()` 更新本地状态
3. 每次 debounce 保存后，乐观更新会修改 `issuesMap[id].description_html`，触发 `initialValue` 变化
4. 但由于值与编辑器当前内容一致，`useEffect` 的 reset 不会产生视觉变化
5. **风险场景**：如果此时恰好有 SWR revalidation 拉回了不同的 `description_html`，`initialValue` 变化会触发 reset → 编辑器内容被服务端数据覆盖 → 正在编辑的内容丢失

### 3.4 字段闪烁的精确原因分析

基于以上数据流，Issue 详情页字段闪烁来源于以下三种场景：

#### 场景 1：乐观更新与 SWR revalidation 的竞态

```
时间线：
  t0: User A 修改 state_id → 乐观更新 issuesMap[id].state_id = "closed"
  t1: User A 切换标签页后切回 → SWR revalidateOnFocus 触发
  t2: SWR 调用 fetchIssueWithIdentifier() → REST GET
  t3: 如果 t0 的 PATCH 尚未到达服务端，GET 返回 state_id = "open"
  t4: addIssue() 浅合并 → issuesMap[id].state_id 被覆盖为 "open" → 闪烁：closed → open
  t5: PATCH 完成 → 下次 SWR revalidation 返回 state_id = "closed" → 再闪烁：open → closed
```

这是因为 `issueUpdate()` 在 PATCH 成功后 **没有** 回写服务端响应到 `issuesMap`，导致 SWR revalidation 在 PATCH 完成前可能拉回旧值。

#### 场景 2：跨用户状态不一致窗口

```
时间线：
  t0: User A 修改 state_id → PATCH → 服务端写入 "closed"
  t1: User B 仍看到 issuesMap 中的旧值 "open"
  t2: User B 切换标签页回来 → SWR revalidateOnFocus
  t3: GET 返回 "closed" → addIssue() 浅合并 → state_id 跳变 "open" → "closed"
```

这不是"闪烁"而是"跳变"——用户 B 的 UI 在 revalidation 前一直显示旧值，revalidation 后突然更新。由于没有实时推送，跳变的时机取决于用户何时触发 SWR revalidation。

#### 场景 3：`addIssue` 浅合并覆盖正在编辑的字段

```
时间线：
  t0: User B 正在修改 assignee_ids（下拉框已打开）
  t1: SWR revalidation 拉回服务端最新数据
  t2: addIssue() 浅合并 → issuesMap[id].assignee_ids 被服务端值覆盖
  t3: observer 组件重渲染 → 下拉框中显示的 assignee 瞬间跳变
```

### 3.5 冲突合并策略（Issue 字段）

**当前没有任何冲突合并策略。** 具体表现为：

- Issue 字段是 **last-write-wins**（最后写入胜出），由服务端 PATCH 直接覆盖。
- 如果用户 A 和用户 B 同时修改同一字段，后提交的覆盖先提交的，没有冲突检测或合并。
- `addIssue` 的浅合并 `{ ...prevIssue, ...serverIssue }` 会在 SWR revalidation 时无条件用服务端数据覆盖本地值，包括用户正在编辑但尚未提交的字段。
- 乐观更新 + 回滚机制仅处理网络错误，不处理并发冲突。
- `IssueVersion` 机制（[issue_version_sync.py](file:///d:/fz/0508-3/solo-dogfeeding/code/197-plane/apps/api/plane/bgtasks/issue_version_sync.py)）仅用于版本记录和审计，不参与实时冲突解决。

### 3.6 断线重连后的状态补齐（Issue）

**没有专门的状态补齐机制。** 补齐依赖以下条件的组合：

| 条件 | 补齐行为 | 覆盖范围 |
|------|----------|----------|
| 用户在 Issue 详情页 | SWR `revalidateOnReconnect: true`（默认）触发 `fetchIssueWithIdentifier()` | 仅详情页，仅当前 issue |
| 用户在列表页 | 列表页 SWR 关闭了 revalidateOnReconnect，**不会自动补齐** | — |
| 用户手动刷新 | 整页重新加载，`useSWR` 重新 fetch | 当前页面所有数据 |
| 用户翻页/滚动 | `fetchNextIssues` → `addIssue()` 更新 `issuesMap` | 仅新增的 issue |

**补齐边界**：
- 断线期间其他人修改的字段，只有在重连后 **且用户恰好在 Issue 详情页** 时才会通过 SWR revalidation 补齐
- 列表页不会自动补齐——用户必须手动刷新或触发翻页
- 没有断线期间变更的增量补发能力，每次 revalidation 都是全量 GET + 浅合并
- 子资源（activity、comment、reaction、link、attachment、subIssue、relation）在 `fetchIssue` 时会一并拉取，但列表页的 peek 概览关闭了 revalidation，不会自动刷新子资源

### 3.7 `issueUpdate` 完成后的数据一致性缺口

`issueUpdate()` 在 PATCH 成功后：

1. ✅ 调用 `fetchParentStats()` 更新父级统计
2. ✅ 调用 `fetchActivities()` 更新活动流（通过 `IssueDetail.updateIssue()` 中的 `Promise.all`）
3. ❌ **没有** 回写服务端响应到 `issuesMap`——PATCH 的返回值被丢弃
4. ❌ **没有** 重新 fetch 当前 issue 来获取服务端计算的字段（`updated_at`、`completed_at` 等）

这意味着 PATCH 成功后，`issuesMap` 中的数据仍然是乐观更新的值，可能与服务端实际值有细微差异（如 `updated_at` 时间戳）。这不会直接导致闪烁，但会影响后续 SWR revalidation 的比较基准。

---

## 四、服务端 PATCH 保存后的事件链路

### 4.1 PATCH 入口与响应

Issue 字段修改有**两条 API 入口**，行为不同：

| 入口 | 文件 | 返回值 | 前端调用方 |
|------|------|--------|-----------|
| `IssueViewSet.partial_update` | [base.py#L616-L701](file:///d:/fz/0508-3/solo-dogfeeding/code/197-plane/apps/api/plane/app/views/issue/base.py#L616-L701) | **`204 No Content`**（无 body） | web 前端主流程 |
| `IssueDetailAPIEndpoint.patch` | [issue.py#L739-L795](file:///d:/fz/0508-3/solo-dogfeeding/code/197-plane/apps/api/plane/api/views/issue.py#L739-L795) | **`200 serializer.data`**（完整序列化） | 外部 API / PUT 接口 |

**前端 web 走的是 `IssueViewSet`**，返回 `204 No Content`——这意味着：

1. PATCH 响应**不携带**服务端计算后的字段值（如 `updated_at`、`completed_at`、`sequence_id`）
2. 前端 `issueUpdate()` 的 `await this.issueService.patchIssue(...)` 仅获知成功/失败，**无法**用响应体回写 `issuesMap`
3. `issuesMap` 中的数据一直是步骤 ② 乐观写入的值，直到 SWR revalidation 或手动刷新拉回最新数据

> 注意：外部 API 入口 `IssueDetailAPIEndpoint.patch` 返回完整的 `serializer.data`，但前端 web 不调用此接口。

### 4.2 PATCH 保存后的四条异步链路

`IssueViewSet.partial_update` 在 `serializer.save()` 成功后，触发以下四条 Celery 异步任务链路（`skip_activity=True` 且为描述更新时跳过全部链路）：

```
serializer.save()
  │
  ├─① issue_activity.delay(type="issue.activity.updated", ..., notification=True, origin=base_host)
  │    └─ Celery 异步
  │
  ├─② model_activity.delay(model_name="issue", ..., origin=base_host)
  │    └─ Celery 异步
  │
  └─③ issue_description_version_task.delay(updated_issue=current_instance, ...)
       └─ Celery 异步
```

#### 链路 ①：`issue_activity` — 活动记录 + 通知投递

**代码**：[issue_activities_task.py#L1503-L1603](file:///d:/fz/0508-3/solo-dogfeeding/code/197-plane/apps/api/plane/bgtasks/issue_activities_task.py#L1503-L1603)

执行步骤：

1. **Redis 写入**：如果传了 `origin`，将 `issue_id → origin` 写入 Redis，**TTL=600s**（10 分钟）。用途：后续 email 通知中的链接需要知道请求来源域名（`origin`），存在 Redis 里供 `email_notification_task` 读取。
2. **更新 `issue.updated_at`**：直接 `issue.updated_at = timezone.now(); issue.save(update_fields=["updated_at"])`。这是服务端对 `updated_at` 的权威刷新。
3. **字段级 diff → 创建 `IssueActivity` 记录**：通过 `update_issue_activity` 逐字段比对 `requested_data` 和 `current_instance`，为每个变更字段创建一条 `IssueActivity`。比对的字段有：`name`、`parent_id`、`priority`、`state_id`、`description_html`、`target_date`、`start_date`、`label_ids`、`assignee_ids`、`estimate_point`、`archived_at`、`closed_to`。
4. **`bulk_create` 写入**：一次性写入所有 `IssueActivity` 记录到数据库。
5. **触发通知任务**（`notification=True` 时）：调用 `notifications.delay(...)` → 链路 ①-①。

**链路 ①-①：`notifications` — 通知投递**

**代码**：[notification_task.py#L190-L671](file:///d:/fz/0508-3/solo-dogfeeding/code/197-plane/apps/api/plane/bgtasks/notification_task.py#L190-L671)

执行步骤：

1. **提取 @mention**：从 `description_html` 中解析 `<mention-component>` 标签，diff 出新增和移除的 mention。
2. **mention 自动订阅**：被 mention 的用户自动成为 `IssueSubscriber`。
3. **订阅者通知**：排除 actor 自己和已被 mention 的用户，向剩余订阅者批量创建 `Notification` 记录（in-app 通知）。
4. **mention 通知**：向被 mention 的用户创建 `Notification` 记录，sender 为 `in_app:issue_activities:mentioned`。
5. **邮件通知**：根据 `UserNotificationPreference` 配置（`state_change`、`issue_completed`、`comment`、`property_change`、`mention`），批量创建 `EmailNotificationLog` 记录，供 `email_notification_task` 异步发送。
6. **`bulk_create`**：一次性写入 `Notification` 和 `EmailNotificationLog`。

**对前端实时同步的影响**：

- `Notification` 记录会出现在前端的通知铃铛（inbox）中，但这是**独立的拉取通道**，不会触发 Issue 详情页字段刷新。
- `IssueActivity` 记录会被前端 `fetchActivities()` 拉取并显示在活动流中。`IssueDetail.updateIssue()` 在 PATCH 成功后会 `Promise.all([issueUpdate, fetchActivities])`，所以编辑者自己能看到活动流更新。但其他用户的 `fetchActivities` 不受触发——他们需要 SWR revalidation。
- `issue.updated_at` 的更新不会推送到前端，只有 GET 请求才能拿到新值。

#### 链路 ②：`model_activity` — Webhook 投递

**代码**：[webhook_task.py#L471-L513](file:///d:/fz/0508-3/solo-dogfeeding/code/197-plane/apps/api/plane/bgtasks/webhook_task.py#L471-L513)

执行步骤：

1. **逐字段 diff**：遍历 `requested_data` 中的每个 key，与 `current_instance` 比对。
2. **每个变更字段触发一个 `webhook_activity.delay()`**：如 `state_id` 变更会触发一个 `webhook_activity`，`priority` 变更再触发一个。

**链路 ②-①：`webhook_activity` — Webhook 筛选与分发**

**代码**：[webhook_task.py#L385-L468](file:///d:/fz/0508-3/solo-dogfeeding/code/197-plane/apps/api/plane/bgtasks/webhook_task.py#L385-L468)

执行步骤：

1. **筛选 Webhook**：查询 `Webhook.objects.filter(workspace__slug=slug, is_active=True, issue=True)` 找到所有订阅了 issue 事件的 webhook。
2. **对每个 webhook 触发 `webhook_send_task.delay()`**。

**链路 ②-②：`webhook_send_task` — HTTP POST 投递**

**代码**：[webhook_task.py#L254-L382](file:///d:/fz/0508-3/solo-dogfeeding/code/197-plane/apps/api/plane/bgtasks/webhook_task.py#L254-L382)

执行步骤：

1. **构建 payload**：包含 `event`、`action`、`webhook_id`、`workspace_id`、`data`（完整 issue 序列化）、`activity`（字段变更详情）。
2. **HMAC 签名**：如果 webhook 有 `secret_key`，用 HMAC-SHA256 签名放入 `X-Plane-Signature` header。
3. **HTTP POST**：向 webhook URL 发送请求，超时 30s。
4. **记录日志**：将请求/响应日志写入 MongoDB（`webhook_logs` 集合），失败回退到 PostgreSQL `WebhookLog` 表。
5. **重试与停用**：最多重试 5 次（`max_retries=5`，`retry_backoff=600s`），全部失败后停用 webhook 并发邮件通知创建者。

**对前端实时同步的影响**：

- ❌ **完全不影响前端**。Webhook 是外部投递机制，向第三方系统推送变更事件。
- Webhook 的 `data` 字段包含完整 issue 序列化数据，但只发给外部 HTTP 端点，不发给 Plane 前端。
- Webhook 投递失败不会回滚 issue 更新——issue 已经持久化成功。

#### 链路 ③：`issue_description_version_task` — 描述版本记录

**代码**：[issue_description_version_task.py#L43-L78](file:///d:/fz/0508-3/solo-dogfeeding/code/197-plane/apps/api/plane/bgtasks/issue_description_version_task.py#L43-L78)

执行步骤：

1. **比较 `description_html`**：如果 `description_html` 没有实际变化则跳过。
2. **合并或新建版本**：如果最新版本是同一用户在 600 秒内编辑的，则更新该版本；否则创建新的 `IssueDescriptionVersion` 记录。
3. **写入数据库**：保存 `description_json`、`description_html`、`description_binary`、`description_stripped`。

**对前端实时同步的影响**：

- ❌ **完全不影响前端**。这是纯粹的审计/版本历史功能，前端通过独立的"历史版本"页面拉取 `IssueDescriptionVersion`，不参与实时同步。

### 4.3 Redis 在 Issue 事件中的角色

Redis 在 API 服务端的 Issue 链路中**仅做一件事**：存储 `issue_id → origin` 映射（TTL=600s），供 `email_notification_task` 生成邮件中的链接。

```python
# issue_activities_task.py#L1528-L1531
if origin:
    ri = redis_instance()
    ri.set(str(issue_id), origin, ex=600)
```

这与 live server 中的 Redis 用途完全不同：
- **API 服务端 Redis**：仅存储 `issue_id → origin`（通知邮件用），是 key-value 缓存。
- **Live server Redis**：Hocuspocus 扩展用 Redis Pub/Sub 做跨节点 Y.js 更新和 stateless 消息广播。

两者共享同一个 Redis 实例，但使用不同的 key 空间和机制（一个用 `SET/GET`，一个用 `Pub/Sub`），互不干扰。

### 4.4 事件链路对前端实时同步的影响汇总

| 链路 | 目的 | 写入目标 | 影响前端实时同步？ | 影响方式 |
|------|------|----------|-------------------|----------|
| `issue_activity` → `IssueActivity` | 活动记录（审计） | PostgreSQL | **间接** | 编辑者的 `fetchActivities()` 可拉到新记录；其他用户需等 SWR revalidation |
| `issue_activity` → `notifications` | 通知投递 | PostgreSQL `Notification` + `EmailNotificationLog` | **间接** | 通知铃铛可拉到新通知，但不触发 Issue 详情刷新 |
| `issue_activity` → `issue.updated_at` 更新 | 时间戳刷新 | PostgreSQL | **间接** | 仅通过后续 GET 才能拿到新 `updated_at` |
| `issue_activity` → Redis `issue_id:origin` | 邮件链接域名 | Redis | ❌ | 仅通知邮件用 |
| `model_activity` → `webhook_activity` → `webhook_send_task` | Webhook 外部投递 | MongoDB/PostgreSQL 日志 | ❌ | 外部系统推送 |
| `issue_description_version_task` | 描述版本历史 | PostgreSQL | ❌ | 独立版本页面拉取 |

### 4.5 PATCH 响应、后续拉取与其他客户端本地状态的关系

#### 4.5.1 编辑者（User A）的完整时序

##### 客户端发起阶段

`Promise.all` 在同一个微任务中依次启动分支 A 和分支 B，两个 HTTP 请求几乎同时进入浏览器网络栈：

```
[微任务内]
  A-1 ~ A-4 同步执行：乐观更新 issuesMap + projectMap + 列表分组
  A-5 await patchIssue(...)     → PATCH HTTP 请求排队
  B-1 同步执行：检查 activityMap 增量锚点
  B-2 await getIssueActivities(...) → GET /activities/ HTTP 请求排队
  [微任务结束，浏览器发出两个 HTTP 请求]
```

##### 服务端处理 PATCH 请求

[IssueViewSet.partial_update](file:///d:/fz/0508-3/solo-dogfeeding/code/197-plane/apps/api/plane/app/views/issue/base.py#L616-L701) 在**一个同步请求周期**内完成以下操作：

```
[PATCH 请求到达服务端]
  S-1 queryset.annotate(label_ids, assignee_ids, module_ids).filter(pk=pk).first()
      ← 同步 DB 查询，获取更新前的 issue 实例
  S-2 current_instance = json.dumps(IssueDetailSerializer(issue).data)
      ← 同步，序列化更新前快照（用于后续 Celery diff）
  S-3 requested_data = json.dumps(request.data)
      ← 同步，序列化请求体
  S-4 serializer = IssueCreateSerializer(issue, data=request.data, partial=True)
  S-5 serializer.save()
      ← 同步 DB 写入，事务提交（Django 默认 autocommit）
      ← 此时 issue 已持久化到 DB
  S-6 issue_activity.delay(
        type="issue.activity.updated",
        requested_data=requested_data,
        current_instance=current_instance, ...)
      ← 同步，但仅入队（Celery delay = apply_async = broker LPUSH/PUBLISH）
      ← 不等 worker 执行，耗时 ~1-5ms（Redis 写入）
  S-7 model_activity.delay(model_name="issue", ...)
      ← 同步，仅入队
  S-8 issue_description_version_task.delay(updated_issue=current_instance, ...)
      ← 同步，仅入队
  S-9 return Response(status=status.HTTP_204_NO_CONTENT)
      ← 返回 204，此时 DB 已写入但 Celery 任务尚未执行
```

**关键区分**：`serializer.save()`（S-5）是**同步 DB 写入**，在 204 返回前已提交。`issue_activity.delay()`（S-6）是**同步入队**，只将任务消息推入 Redis broker，worker 尚未开始执行。204 响应发出时，DB 中只有更新后的 issue 行，**没有任何 IssueActivity、Notification 记录**。

##### 客户端收到 204 后

```
[浏览器收到 204]
  A-5 await 结束 → 分支 A 恢复执行
  A-6 fetchParentStats() → GET /api/.../projects/:id/  ← fire-and-forget
      写入目标：projectMap（非 issuesMap）

[浏览器收到 GET /activities/ 响应]
  B-2 await 结束 → 分支 B 恢复执行
  B-3 runInAction → 将活动记录写入 activityMap + activities
```

##### Celery worker 执行（独立进程，时序不确定）

```
[Worker 从 broker 取出任务]
  C-1 issue_activity 任务执行：
      a. Redis SET issue_id→origin (TTL=600s)
      b. issue.updated_at = timezone.now(); issue.save(update_fields=["updated_at"])
         ← 再次写入 DB，更新 updated_at
      c. update_issue_activity: 逐字段 diff → bulk_create IssueActivity
         ← 此时 IssueActivity 记录才落库
      d. notifications.delay(...) → 入队
  C-2 model_activity 任务执行：
      → webhook_activity.delay(...) → 入队
  C-3 notifications 任务执行：
      → bulk_create Notification + EmailNotificationLog
         ← 此时通知记录才落库
  C-4 webhook_activity → webhook_send_task → HTTP POST 外部端点
  C-5 issue_description_version_task 执行：
      → 创建/更新 IssueDescriptionVersion
```

##### 竞态来源分类

| 竞态 | 来源 | 是否确定性 | 说明 |
|------|------|-----------|------|
| PATCH vs GET/activities 的网络到达顺序 | **网络延迟** | 不确定 | 两个请求几乎同时发出，到达服务端的顺序取决于 TCP 连接复用、网络抖动等 |
| GET/activities 到达时 IssueActivity 是否已落库 | **Celery worker 执行延迟** | 不确定 | S-6 入队到 C-1c 落库之间有：broker 传输 + worker 取出 + diff 计算 + bulk_create。通常 50-200ms，繁忙时可达秒级 |
| `fetchParentStats` 到达时 project 统计是否已反映 PATCH | **无竞态** | 确定 | S-5 已同步写入 DB，project 统计查询基于 DB 聚合，一定能读到最新值 |
| SWR revalidation 到达时 PATCH 是否已落库 | **网络延迟** | 不确定 | 取决于用户切换标签页的时机 vs PATCH 往返时间 |

**结论**：
- **网络级竞态**（PATCH vs GET/activities 的到达顺序）在实际中几乎不影响结果——因为即使 PATCH 先到达并完成 S-5 写入，GET/activities 查询的也是 `IssueActivity` 表而非 `Issue` 表，而 `IssueActivity` 的落库取决于 Celery worker
- **真正的竞态是 Celery worker 执行延迟**：无论 GET/activities 在 PATCH 之前还是之后到达，都**极可能早于** C-1c（IssueActivity 落库），因为 C-1c 需要经过 broker 传输 + worker 取出 + diff + bulk_create
- `fetchParentStats` 不存在竞态——S-5 已同步写入 DB，后续的 project 查询一定能读到最新 issue 数据

#### 4.5.2 各拉取操作能更新的数据范围

| 拉取操作 | 触发时机 | HTTP 请求 | 写入 store | 写入 issuesMap？ | 能修正的数据 |
|----------|----------|-----------|-----------|-----------------|-------------|
| `fetchActivities`（分支 B） | `Promise.all` 与 PATCH 并行 | `GET /activities/?created_at__gt=...` | `activityMap` + `activities` | ❌ | 活动流记录（field/old_value/new_value/actor），不含 issue 字段 |
| `fetchParentStats`（A-6） | PATCH 204 后 fire-and-forget | `GET /projects/:id/` | `projectMap` | ❌ | 项目统计（total_issues/completed_issues/cancelled_issues 等），不含任何 issue 字段 |
| SWR revalidation | 标签页焦点 / 网络恢复 | `GET /issues/:id/` | `issuesMap`（via `addIssue`） | ✅ 浅合并覆盖 | **issuesMap 中的全部字段**（含 `updated_at`、`completed_at` 等服务端计算字段），但也会覆盖本地未提交修改 |
| `fetchComments` | 手动/组件挂载 | `GET /issues/:id/comments/` | `commentMap` | ❌ | 评论列表 |
| `fetchReactions` | 手动/组件挂载 | `GET /issues/:id/reactions/` | `reactionMap` | ❌ | 表情回应 |
| `fetchLinks` | 手动/组件挂载 | `GET /issues/:id/links/` | `linkMap` | ❌ | 链接列表 |
| `fetchAttachments` | 手动/组件挂载 | `GET /issues/:id/attachments/` | `attachmentMap` | ❌ | 附件列表 |
| `fetchSubIssues` | 手动/组件挂载 | `GET /issues/:id/sub-issues/` | `subIssueMap` | ❌ | 子 issue 列表 |
| `fetchRelations` | 手动/组件挂载 | `GET /issues/:id/relations/` | `relationMap` | ❌ | 关联 issue 列表 |

**核心结论**：
- 编辑者的 `issuesMap` 在 PATCH 成功后**没有任何拉取操作主动修正**——`fetchActivities` 只修活动流，`fetchParentStats` 只修 `projectMap`，`issuesMap` 中的 `updated_at` 等服务端计算字段永远停留在乐观值
- 其他客户端的 `issuesMap` **只能通过 `fetchIssueWithIdentifier`（SWR revalidation）修正**
- 所有子资源拉取（comments/reactions/links/attachments/subIssues/relations）都不触及 `issuesMap`

#### 4.5.3 `fetchActivities` 与 Celery 落库的竞态

`fetchActivities`（分支 B）的 GET/activities 请求与 Celery `issue_activity` 任务的执行存在**确定性竞态**：

```
客户端时间线                        服务端时间线
──────────                        ──────────
A-5 PATCH 请求发出
B-2 GET /activities/ 请求发出
  │                                PATCH 到达 → S-1~S-5 同步执行（~20ms）
  │                                S-5 serializer.save() → DB 写入 "closed"  ← 已提交
  │                                S-6 issue_activity.delay(...) → Redis 入队   ← ~1-5ms
  │                                S-7 model_activity.delay(...) → Redis 入队
  │                                S-8 issue_description_version_task.delay(...)
  │                                S-9 返回 204 ─────────────────────────────→ A-5 await 结束
  │
  │                                GET /activities/ 到达 → DB 查询
  │                                此时 IssueActivity 表中无新记录          ← C-1c 尚未执行
  │                                返回空列表 ─────────────────────────────→ B-2 await 结束
  │
  │                                [Celery worker 从 broker 取出任务]
  │                                C-1a Redis SET
  │                                C-1b issue.updated_at = now(); save()
  │                                C-1c diff → bulk_create IssueActivity    ← 此时才落库
  │                                C-1d notifications.delay(...)
```

**竞态分析**：
- GET/activities 查询的是 `IssueActivity` 表，不是 `Issue` 表
- `IssueActivity` 记录在 C-1c 才落库，这需要经过：broker 传输（~1ms）+ worker 取出（取决于队列深度，0ms~数秒）+ diff 计算（~5ms）+ bulk_create（~5ms）
- 204 响应返回后（S-9），GET/activities 的到达和 C-1c 的执行**完全独立**
- **通常情况下 GET/activities 会早于 C-1c**——因为 GET 只需 ~50ms 往返，而 Celery 通常需要 ~100-300ms 完成 C-1a~C-1c
- 但如果 Celery worker 空闲且恰好在同一进程，也可能 C-1c 先完成

**增量拉取的盲区**：`fetchActivities` 使用 `created_at__gt=lastActivity.created_at` 增量过滤。如果本次 GET 没拉到新记录（Celery 还没落库），后续的 `fetchActivities` 调用仍会用同一个锚点——**但只有在 `Promise.all` 的 `updateIssue` 中才会再次触发**。用户手动操作（如切换 state）会触发新的 `Promise.all`，其中包含新的 `fetchActivities`，此时 C-1c 可能已经完成，增量拉取可以补回漏掉的记录。

#### 4.5.4 其他用户（User B）的感知时序

```
t1: [User A] PATCH 到达服务端 → S-5 DB 写入 "closed" → S-9 返回 204
t2: [Celery] C-1c IssueActivity 落库
t3: [Celery] C-3 Notification 落库
t?: [User B] 切换标签页回来 → SWR revalidateOnFocus
    → fetchIssueWithIdentifier() → GET /api/.../issues/:id/
    → 服务端返回最新数据（state_id="closed", updated_at=C-1b的时间戳）
    → addIssue() → { ...prevIssue, ...serverIssue } 浅合并覆盖
    → UI 更新：state_id 从 "open" 跳变为 "closed"
```

**关键问题**：
- User B 感知 User A 的修改**完全依赖** SWR revalidation 或手动刷新，没有推送通道
- 如果 User B 也正在修改同一 issue，`addIssue` 的浅合并会用服务端全量数据覆盖 User B 的本地未提交修改
- User B 的通知铃铛可能在 SWR revalidation 之前就显示"state 变更"通知（C-3 落库后），但 Issue 详情页仍显示旧值——**通知和详情不一致**
- User B 的活动流不会自动更新——`fetchActivities` 只在编辑者 User A 的 `Promise.all` 中被触发，User B 需要手动刷新
- User B 通过 SWR revalidation 拿到的 `updated_at` 是 C-1b 写入的值（Celery 任务中的 `timezone.now()`），不是 S-5 写入的值——这解释了为什么 `updated_at` 会有细微延迟

#### 4.5.5 PATCH 响应与 SWR revalidation 的竞态

```
时间线（User A）：
  t0: A-2 乐观更新 issuesMap[id].state_id = "closed"
  t1: A-5 PATCH 请求发出
  t2: [网络延迟] PATCH 尚未到达服务端
  t3: User A 切换标签页后切回 → SWR revalidateOnFocus
  t4: GET /api/.../issues/:id/ → 返回旧值 state_id = "open"（S-5 尚未执行）
  t5: addIssue() 浅合并 → issuesMap 被覆盖为 "open" → 闪烁！
  t6: PATCH 到达服务端 → S-5 DB 写入 "closed"
  t7: 下次 SWR revalidation → GET 返回 "closed" → 再次闪烁！
```

**根因**：乐观更新写入 `issuesMap` 后，如果 SWR revalidation 在 PATCH 落库前触发 GET，会拉回旧值覆盖乐观更新。

可能的缓解策略（代码中尚未实现）：
1. PATCH 成功后立即用服务端响应回写 `issuesMap`——但当前返回 204，无 body 可用
2. PATCH 进行中时暂时禁用 SWR revalidation——需要修改 SWR 配置
3. 将 `IssueViewSet.partial_update` 改为返回 `200 serializer.data`——让前端能用响应体修正本地状态

---

## 五、Hocuspocus 支持的文档类型

当前 Hocuspocus live server 仅支持 `project_page` 类型（[handler.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/197-plane/apps/live/src/services/page/handler.ts)）：

```typescript
export const getPageService = (documentType: TDocumentTypes, context: HocusPocusServerContext) => {
  if (documentType === "project_page") {
    return new ProjectPageService({ ... });
  }
  throw new AppError(`Invalid document type ${documentType} provided.`);
}
```

`TDocumentTypes = "project_page"`（[types/index.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/197-plane/apps/live/src/types/index.ts#L26)），Issue 描述不在支持范围内。

---

## 六、数据流汇总图

### Page 文档协作（有实时通道）

```
用户 A 编辑
    │
    ▼
CollaborativeDocumentEditorWithRef (Tiptap + Y.js)
    │ Y.js 本地更新
    ▼
WebSocket ──────────────────────────────────────► Hocuspocus Server
    │                                                  │
    │                                          Y.js 广播给同文档
    │                                          其他连接 (含跨节点 via Redis)
    │                                                  │
    │ ◄── Y.js 增量同步 ───────────────────────────────┘
    │
    ▼
用户 B 的编辑器实时更新 (CRDT 合并)

额外路径：
  - 文档持久化: debounce 10s → Database.store → REST API
  - 标题持久化: debounce 5s → TitleUpdateManager → REST API
  - 标题变更广播: TitleSyncExtension → broadcastMessageToPage (parent page)
  - Stateless 事件: lock/unlock/archive/delete 等 → onStateless → broadcastStateless
```

### Issue 详情字段（无实时通道，MobX 全局 store 驱动）

```
用户 A 修改字段 (state/priority/etc.)
    │
    ▼
issueOperations.update() → BaseIssuesStore.issueUpdate()
    │
    ├─ ① 乐观更新: issuesMap.updateIssue(issueId, data)  ← MobX observer 立即重渲染
    ├─ ② REST PATCH /api/.../issues/:id/
    │      ├─ 成功: fetchParentStats() + fetchActivities()
    │      └─ 失败: issuesMap.updateIssue(issueBeforeUpdate) ← 回滚
    │
    │  (无推送给其他客户端)
    │
    ▼
用户 B 感知变更的路径:
    ├─ 切换标签页回来 → SWR revalidateOnFocus → fetchIssueWithIdentifier()
    │      → REST GET → addIssue() → issuesMap 浅合并 → observer 重渲染
    ├─ 网络恢复 → SWR revalidateOnReconnect → 同上
    ├─ 列表翻页 → fetchNextIssues → addIssue() → issuesMap 浅合并
    └─ 手动刷新 → 整页重新加载
```

---

## 七、关键文件索引

| 模块 | 文件 | 作用 |
|------|------|------|
| Live Server | [server.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/197-plane/apps/live/src/server.ts) | Express + WebSocket 入口 |
| Hocuspocus | [hocuspocus.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/197-plane/apps/live/src/hocuspocus.ts) | Hocuspocus 服务器配置 (debounce=10s) |
| 认证 | [auth.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/197-plane/apps/live/src/lib/auth.ts) | WebSocket 连接认证 |
| 文档持久化 | [database.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/197-plane/apps/live/src/extensions/database.ts) | fetch/store 文档二进制 |
| Redis 广播 | [redis.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/197-plane/apps/live/src/extensions/redis.ts) | 跨节点消息同步 |
| 标题同步 | [title-sync.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/197-plane/apps/live/src/extensions/title-sync.ts) | 标题变更检测+持久化+广播 |
| 标题防抖 | [title-update-manager.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/197-plane/apps/live/src/extensions/title-update/title-update-manager.ts) | 标题更新 debounce (5s) |
| 防抖管理 | [debounce.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/197-plane/apps/live/src/extensions/title-update/debounce.ts) | 通用 debounce + abort 支持 |
| 强制关闭 | [force-close-handler.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/197-plane/apps/live/src/extensions/force-close-handler.ts) | 管理命令跨节点关闭连接 |
| Stateless 消息 | [stateless.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/197-plane/apps/live/src/lib/stateless.ts) | 无状态事件转发 |
| 消息广播 | [broadcast-message.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/197-plane/apps/live/src/utils/broadcast-message.ts) | 向文档所有连接广播 |
| 错误广播 | [broadcast-error.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/197-plane/apps/live/src/utils/broadcast-error.ts) | 向客户端广播错误 |
| 协作事件定义 | [document-collaborative-events.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/197-plane/packages/editor/src/core/constants/document-collaborative-events.ts) | Page 协作事件类型映射 |
| 协作事件类型 | [document-collaborative-events.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/197-plane/packages/editor/src/core/types/document-collaborative-events.ts) | 事件 payload 类型定义 |
| WS 装饰器 | [websocket.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/197-plane/packages/decorators/src/websocket.ts) | @WebSocket 路由装饰器 |
| 协作控制器 | [collaboration.controller.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/197-plane/apps/live/src/controllers/collaboration.controller.ts) | WS /collaboration 路由 |
| Page 编辑器 | [editor-body.tsx](file:///d:/fz/0508-3/solo-dogfeeding/code/197-plane/apps/web/core/components/pages/editor/editor-body.tsx) | Page 协作文档编辑器组件 |
| Page 实时事件 | [use-realtime-page-events.tsx](file:///d:/fz/0508-3/solo-dogfeeding/code/197-plane/apps/web/core/hooks/use-realtime-page-events.tsx) | Page 实时事件处理 hook |
| Page 协作动作 | [use-collaborative-page-actions.tsx](file:///d:/fz/0508-3/solo-dogfeeding/code/197-plane/apps/web/core/hooks/use-collaborative-page-actions.tsx) | Page 协作操作 hook |
| Issue 全局 store | [issue.store.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/197-plane/apps/web/core/store/issue/issue.store.ts) | issuesMap 单一数据源、addItem/updateIssue/removeIssue |
| Issue 详情 store | [issue-details/issue.store.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/197-plane/apps/web/core/store/issue/issue-details/issue.store.ts) | Issue fetch/update 路由（Promise.all 并行 issueUpdate + fetchActivities） |
| Issue 活动流 store | [activity.store.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/197-plane/apps/web/ce/store/issue/issue-details/activity.store.ts) | fetchActivities 增量拉取 + activityMap 管理 |
| Issue 详情 root | [issue-details/root.store.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/197-plane/apps/web/core/store/issue/issue-details/root.store.ts) | IssueDetail 聚合 store（issue+activity+comment+...） |
| Issue 乐观更新 | [base-issues.store.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/197-plane/apps/web/core/store/issue/helpers/base-issues.store.ts) | issueUpdate 乐观更新+回滚+fetchParentStats 逻辑 |
| Project store | [project.store.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/197-plane/apps/web/core/store/project/project.store.ts) | fetchProjectDetails（fetchParentStats 的写入目标：projectMap） |
| Project issues store | [project/issue.store.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/197-plane/apps/web/core/store/issue/project/issue.store.ts) | fetchParentStats → project.fetchProjectDetails |
| Issue 描述编辑 | [description-input/root.tsx](file:///d:/fz/0508-3/solo-dogfeeding/code/197-plane/apps/web/core/components/editor/rich-text/description-input/root.tsx) | Issue 描述编辑器 (非协作, 1500ms debounce) |
| Issue 主内容 | [main-content.tsx](file:///d:/fz/0508-3/solo-dogfeeding/code/197-plane/apps/web/core/components/issues/issue-detail/main-content.tsx) | Issue 详情页主内容区 |
| Issue 详情页 | [browse/page.tsx](file:///d:/fz/0508-3/solo-dogfeeding/code/197-plane/apps/web/app/(all)/[workspaceSlug]/(projects)/projects/(detail)/browse/[workItem]/page.tsx) | SWR+fetchIssueWithIdentifier 加载入口 |
| Issue 详情根组件 | [issue-detail/root.tsx](file:///d:/fz/0508-3/solo-dogfeeding/code/197-plane/apps/web/core/components/issues/issue-detail/root.tsx) | IssueDetailRoot + TIssueOperations 定义 |
| Issue 侧边栏 | [issue-detail/sidebar.tsx](file:///d:/fz/0508-3/solo-dogfeeding/code/197-plane/apps/web/core/components/issues/issue-detail/sidebar.tsx) | 字段下拉框 (state/priority/assignee/date) |
| Peek 概览 | [peek-overview/root.tsx](file:///d:/fz/0508-3/solo-dogfeeding/code/197-plane/apps/web/core/components/issues/peek-overview/root.tsx) | 列表内弹窗 (SWR revalidation 已关闭) |
| WorkItem 详情 | [workItem-detail.tsx](file:///d:/fz/0508-3/solo-dogfeeding/code/197-plane/apps/web/ce/components/browse/workItem-detail.tsx) | CE 版 WorkItemDetailRoot 包装 |
| Page 服务 | [core.service.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/197-plane/apps/live/src/services/page/core.service.ts) | Page CRUD 服务 (live server 端) |
| 类型定义 | [types/index.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/197-plane/apps/live/src/types/index.ts) | HocusPocus 上下文类型 |
| Issue 版本 | [issue_version_sync.py](file:///d:/fz/0508-3/solo-dogfeeding/code/197-plane/apps/api/plane/bgtasks/issue_version_sync.py) | Issue 版本记录 (非实时) |
| Redis 管理 | [redis.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/197-plane/apps/live/src/redis.ts) | Redis 连接管理 (reconnect策略) |
| **服务端 Issue PATCH** | [base.py](file:///d:/fz/0508-3/solo-dogfeeding/code/197-plane/apps/api/plane/app/views/issue/base.py) | `IssueViewSet.partial_update` (返回 204) |
| **服务端 Issue PATCH (API)** | [issue.py](file:///d:/fz/0508-3/solo-dogfeeding/code/197-plane/apps/api/plane/api/views/issue.py) | `IssueDetailAPIEndpoint.patch` (返回 200+body) |
| **活动记录任务** | [issue_activities_task.py](file:///d:/fz/0508-3/solo-dogfeeding/code/197-plane/apps/api/plane/bgtasks/issue_activities_task.py) | 字段级 diff + IssueActivity 创建 + 通知触发 |
| **Webhook 任务** | [webhook_task.py](file:///d:/fz/0508-3/solo-dogfeeding/code/197-plane/apps/api/plane/bgtasks/webhook_task.py) | model_activity → webhook_activity → webhook_send_task |
| **通知任务** | [notification_task.py](file:///d:/fz/0508-3/solo-dogfeeding/code/197-plane/apps/api/plane/bgtasks/notification_task.py) | Notification + EmailNotificationLog 创建 |
| **描述版本任务** | [issue_description_version_task.py](file:///d:/fz/0508-3/solo-dogfeeding/code/197-plane/apps/api/plane/bgtasks/issue_description_version_task.py) | IssueDescriptionVersion 审计记录 |
| **API Redis** | [redis.py](file:///d:/fz/0508-3/solo-dogfeeding/code/197-plane/apps/api/plane/settings/redis.py) | API 端 Redis 连接 (issue_id→origin 缓存) |
