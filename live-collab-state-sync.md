# Plane 实时协作通道 — 状态同步机制梳理

## 一、整体架构概览

Plane 的"实时协作"分为**两条独立通道**，服务于不同的业务场景：

| 通道 | 覆盖范围 | 传输协议 | 数据模型 |
|------|----------|----------|----------|
| **Hocuspocus/Y.js 通道** | Page 文档编辑（富文本 + 标题） | WebSocket → Y.js CRDT | Y.Doc 二进制增量 |
| **REST + SWR 轮询通道** | Issue 详情字段（state/priority/assignee/label 等） | HTTP PATCH → SWR revalidation | JSON 全量替换 |

> **关键发现：Issue 详情的元数据字段（state、priority、assignee 等）目前不走 WebSocket 实时通道，而是通过 REST API 更新 + SWR 客户端缓存刷新。** 这是字段闪烁问题的核心上下文。

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

## 三、REST + SWR 通道（Issue 详情字段）

### 3.1 当前机制

Issue 详情的元数据字段（state、priority、assignee、labels、start_date、target_date 等）**没有 WebSocket 实时通道**，走的是传统 REST 模式：

```
客户端 A 修改字段 → PATCH /api/.../issues/:id/ → 服务端写入 DB → 返回 200
                                                           ↓
                                              客户端 B 通过 SWR revalidation 拉取新数据
```

#### 3.1.1 乐观更新

[base-issues.store.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/197-plane/apps/web/core/store/issue/helpers/base-issues.store.ts#L554-L588) 的 `issueUpdate` 方法：

1. **先乐观更新本地 store**：`this.rootIssueStore.issues.updateIssue(issueId, data)` + `this.updateIssueList()`
2. **再调 REST API**：`this.issueService.patchIssue(workspaceSlug, projectId, issueId, data)`
3. **失败回滚**：catch 中恢复为 `issueBeforeUpdate`

```typescript
// 核心逻辑简化
async issueUpdate(workspaceSlug, projectId, issueId, data, shouldSync = true) {
  const issueBeforeUpdate = clone(this.rootIssueStore.issues.getIssueById(issueId));
  try {
    this.rootIssueStore.issues.updateIssue(issueId, data);     // 乐观更新
    this.updateIssueList({ ...issueBeforeUpdate, ...data }, issueBeforeUpdate);
    if (!shouldSync) return;
    await this.issueService.patchIssue(workspaceSlug, projectId, issueId, data); // REST API
  } catch (error) {
    this.rootIssueStore.issues.updateIssue(issueId, issueBeforeUpdate ?? {});    // 回滚
    this.updateIssueList(issueBeforeUpdate, { ...issueBeforeUpdate, ...data });
    throw error;
  }
}
```

#### 3.1.2 Issue 描述（description_html）

Issue 描述不走 Hocuspocus 协作编辑，而是使用 [DescriptionInput](file:///d:/fz/0508-3/solo-dogfeeding/code/197-plane/apps/web/core/components/editor/rich-text/description-input/root.tsx) 组件：

- 使用 `RichTextEditor`（非协作编辑器），纯本地编辑。
- **1500ms debounce** 自动保存：`debouncedFormSave = debounce(async () => { handleSubmit(...) }, 1500)`
- 保存时调用 `issueOperations.update()` → REST PATCH → `description_html` 字段。
- 组件卸载时如有未保存变更，会立即保存。

> **注意**：`swrDescription` prop 存在但未被 Issue 详情页使用——该 prop 用于外部注入实时描述内容，但当前 Issue 详情的 `DescriptionInput` 并未传入 `swrDescription`。

### 3.2 问题分析：为什么 Issue 字段会闪烁

基于以上代码追踪，Issue 详情页的字段闪烁可能来源于：

1. **乐观更新 → 服务端响应 → SWR revalidation 的三重写入**：
   - 乐观更新立即修改本地 MobX store
   - REST API 返回后，SWR 可能触发重新验证，拉回服务端数据覆盖本地
   - 如果其他用户也在编辑，revalidation 拉回的数据可能与乐观更新不同

2. **无实时通道导致的状态不一致窗口**：
   - 用户 A 修改 state → PATCH API → 用户 A 本地乐观更新完成
   - 用户 B 仍看到旧 state，直到 SWR 下次 revalidation
   - 当 B 的 SWR revalidation 拉到新数据时，state 突然跳变 → 视觉闪烁

3. **SWR revalidation 时机不确定**：
   - SWR 的 `revalidateOnFocus`、`revalidateOnReconnect` 等策略可能导致不预期的数据刷新
   - 多个 SWR hook 指向同一 issue 的不同子资源（reactions、comments、links），刷新频率不同步

4. **DescriptionInput 的 debounce 保存**：
   - 编辑中每 1500ms 自动保存一次
   - 保存成功后 MobX store 更新 `description_html`
   - 如果此时 SWR 也在 revalidation，可能拉回一个略微过时的 `description_html`（网络延迟），导致编辑器内容回退

### 3.3 冲突合并策略（Issue 字段）

**当前没有任何冲突合并策略。** 具体表现为：

- Issue 字段是 **last-write-wins**（最后写入胜出），由服务端 PATCH 直接覆盖。
- 如果用户 A 和用户 B 同时修改同一字段，后提交的覆盖先提交的，没有冲突检测或合并。
- `IssueVersion` 机制（[issue_version_sync.py](file:///d:/fz/0508-3/solo-dogfeeding/code/197-plane/apps/api/plane/bgtasks/issue_version_sync.py)）仅用于版本记录和审计，不参与实时冲突解决。
- 乐观更新 + 回滚机制仅处理网络错误，不处理并发冲突。

### 3.4 断线重连后的状态补齐（Issue）

**没有专门的状态补齐机制。** 依赖 SWR 的内置行为：

- `revalidateOnReconnect: true`（SWR 默认）：网络恢复后自动重新验证所有活跃的 SWR key。
- 用户手动刷新页面时重新拉取。
- 没有断线期间变更的增量补发能力。

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

### Issue 详情字段（无实时通道）

```
用户 A 修改字段 (state/priority/etc.)
    │
    ▼
MobX Store 乐观更新
    │
    ▼
REST PATCH /api/.../issues/:id/
    │
    ▼
服务端写入 DB
    │
    │  (无推送)
    │
    ▼
用户 B 的 SWR revalidation (定时/焦点/重连)
    │
    ▼
MobX Store 更新 → UI 刷新 (可能闪烁)
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
| Issue 详情 store | [issue.store.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/197-plane/apps/web/core/store/issue/issue-details/issue.store.ts) | Issue CRUD 操作 |
| Issue 乐观更新 | [base-issues.store.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/197-plane/apps/web/core/store/issue/helpers/base-issues.store.ts) | Issue 乐观更新+回滚逻辑 |
| Issue 描述编辑 | [description-input/root.tsx](file:///d:/fz/0508-3/solo-dogfeeding/code/197-plane/apps/web/core/components/editor/rich-text/description-input/root.tsx) | Issue 描述编辑器 (非协作) |
| Issue 主内容 | [main-content.tsx](file:///d:/fz/0508-3/solo-dogfeeding/code/197-plane/apps/web/core/components/issues/issue-detail/main-content.tsx) | Issue 详情页主内容区 |
| Page 服务 | [core.service.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/197-plane/apps/live/src/services/page/core.service.ts) | Page CRUD 服务 (live server 端) |
| 类型定义 | [types/index.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/197-plane/apps/live/src/types/index.ts) | HocusPocus 上下文类型 |
| Issue 版本 | [issue_version_sync.py](file:///d:/fz/0508-3/solo-dogfeeding/code/197-plane/apps/api/plane/bgtasks/issue_version_sync.py) | Issue 版本记录 (非实时) |
| Redis 管理 | [redis.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/197-plane/apps/live/src/redis.ts) | Redis 连接管理 (reconnect策略) |
