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

```
User A 点击 StateDropdown → onChange(val)
  → issueOperations.update(workspaceSlug, projectId, issueId, { state_id: val })
    → IssueDetail.issue.updateIssue()
      → ProjectIssues.updateIssue() = BaseIssuesStore.issueUpdate()
        ① clone(getIssueById) 保存快照
        ② rootIssueStore.issues.updateIssue(issueId, data)    ← 乐观写入 issuesMap
        ③ updateIssueList(...)                                ← 更新列表分组 ID
        ④ updateParentStats(...)                              ← 乐观更新父级统计
        ⑤ await this.issueService.patchIssue(...)             ← REST PATCH
        ⑥ fetchParentStats(...)                               ← PATCH 成功后刷新父级统计
        ⑦ [如果 ⑤ 失败] rootIssueStore.issues.updateIssue(issueBeforeUpdate) ← 回滚
```

**关键观察**：
- 步骤 ② 立即修改 `issuesMap` → observer 组件即时重渲染 → UI 无延迟
- 步骤 ⑤ PATCH 成功后 **不会** 重新 fetch 该 issue 来拿服务端计算的字段（如 `updated_at`、`completed_at`），本地保留的是乐观值
- 步骤 ⑤ PATCH 成功后也不会调 `addIssue()` 做服务端数据的浅合并回写
- 只有步骤 ⑥ 刷新的是父级统计，不是当前 issue 本身

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

## 四、Hocuspocus 支持的文档类型

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

## 五、数据流汇总图

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

## 六、关键文件索引

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
| Issue 详情 store | [issue-details/issue.store.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/197-plane/apps/web/core/store/issue/issue-details/issue.store.ts) | Issue fetch/update 路由（委托给 ProjectIssues） |
| Issue 详情 root | [issue-details/root.store.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/197-plane/apps/web/core/store/issue/issue-details/root.store.ts) | IssueDetail 聚合 store（issue+activity+comment+...） |
| Issue 乐观更新 | [base-issues.store.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/197-plane/apps/web/core/store/issue/helpers/base-issues.store.ts) | issueUpdate 乐观更新+回滚逻辑 |
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
