# 非当前容器收敛时序与数据刷新全流程

## 1. 核心结论

**非当前容器的统计数据只有在用户主动路由切换到该容器时才会收敛**。整个过程遵循 **"路由驱动 → Store 选择 → 数据回源 → 本地覆盖 → 视图渲染"** 的五阶段流程，没有任何后台自动同步机制。

---

## 2. 完整收敛时序（以 Cycle 为例）

### 2.1 时序图

```
用户操作: 从 Cycle A 切换到 Cycle B
        │
        ▼
┌─────────────────────────────────────────────────────────────┐
│ 阶段 1: 路由解析 (Next.js App Router)                        │
│ - URL 从 /cycles/A 变为 /cycles/B                           │
│ - useParams() 返回 { cycleId: "B" }                          │
│ 代码: cycle-layout-root.tsx:54-57                            │
└─────────────────────────────────────────────────────────────┘
        │
        ▼
┌─────────────────────────────────────────────────────────────┐
│ 阶段 2: Store 类型选择                                      │
│ - useIssueStoreType() 检测到 cycleId="B"                     │
│ - 返回 EIssuesStoreType.CYCLE                                │
│ - IssuesStoreContext.Provider 提供上下文                     │
│ 代码: use-issue-layout-store.ts:14-43                        │
│       cycle-layout-root.tsx:88                               │
└─────────────────────────────────────────────────────────────┘
        │
        ▼
┌─────────────────────────────────────────────────────────────┐
│ 阶段 3: useIssues 获取 Store 实例                            │
│ - 根据 storeType=CYCLE 返回 context.issue.cycleIssues        │
│ - 这是一个**单例实例**，所有 Cycle 视图共享                   │
│ - 内部通过 computed cycleId 区分当前 Cycle                   │
│ 代码: use-issues.ts:122-126                                  │
│       base-issues.store.ts:274-277                           │
└─────────────────────────────────────────────────────────────┘
        │
        ▼
┌─────────────────────────────────────────────────────────────┐
│ 阶段 4: 视图挂载与 useEffect 触发                            │
│ - BaseKanBanRoot / BaseListRoot / BaseGanttRoot 挂载         │
│ - useEffect 检测到 storeType / viewId 变化                    │
│ - 调用 fetchIssues("init-loader", options, viewId="B")       │
│ 代码: base-kanban-root.tsx:98-100                            │
│       base-list-root.tsx:89-91                               │
│       base-gantt-root.tsx:67-69                              │
└─────────────────────────────────────────────────────────────┘
        │
        ▼
┌─────────────────────────────────────────────────────────────┐
│ 阶段 5: 数据回源与本地覆盖                                    │
│ - CycleIssues.fetchIssues() 调用 API: GET /cycles/B/issues  │
│ - 后端实时计算并返回 Cycle B 的 Issue 列表和统计              │
│ - onfetchIssues() 处理响应:                                  │
│   1. clear() 清空旧的 groupedIssueIds (Cycle A 的数据)       │
│   2. addIssue() 更新全局 issueMap                             │
│   3. updateGroupedIssueIds() 设置 Cycle B 的分组数据          │
│   4. fetchParentStats() 拉取 Cycle B 详情（含统计）           │
│ 代码: cycle/issue.store.ts:186-216                           │
│       base-issues.store.ts:460-488                           │
└─────────────────────────────────────────────────────────────┘
        │
        ▼
┌─────────────────────────────────────────────────────────────┐
│ 阶段 6: 视图渲染更新                                         │
│ - MobX 检测到 groupedIssueIds 变化                            │
│ - observer HOC 触发组件 re-render                            │
│ - 看板/列表/甘特视图从 props 获取最新数据渲染                 │
│ 代码: base-kanban-root.tsx:269-291                           │
│       base-list-root.tsx:158-178                             │
│       base-gantt-root.tsx:131-152                            │
└─────────────────────────────────────────────────────────────┘
```

---

## 3. 各阶段详细说明

### 3.1 阶段 1: 路由解析

**关键代码**：
```typescript
// cycle-layout-root.tsx:54-57
const { workspaceSlug: routerWorkspaceSlug, projectId: routerProjectId, cycleId: routerCycleId } = useParams();
const workspaceSlug = routerWorkspaceSlug ? routerWorkspaceSlug.toString() : undefined;
const projectId = routerProjectId ? routerProjectId.toString() : undefined;
const cycleId = routerCycleId ? routerCycleId.toString() : undefined;
```

**作用**：从 URL 中提取当前 Cycle/Module 的 ID，这是整个流程的起点。

### 3.2 阶段 2: Store 类型选择

`useIssueStoreType()` 是连接路由与 Store 的关键枢纽：

```typescript
// use-issue-layout-store.ts:14-43
export const useIssueStoreType = () => {
  const storeType = useContext(IssuesStoreContext);
  const { cycleId, moduleId, ... } = useParams();

  if (storeType) return storeType;
  if (cycleId) return EIssuesStoreType.CYCLE;
  if (moduleId) return EIssuesStoreType.MODULE;
  // ...其他类型
  return EIssuesStoreType.PROJECT;
};
```

**优先级**：
1. 优先使用 `IssuesStoreContext` 中提供的类型（布局根组件设置）
2. 其次根据 URL 参数推断：`cycleId` → CYCLE，`moduleId` → MODULE
3. 最后默认 PROJECT

**代码依据**：
- Cycle 布局：`cycle-layout-root.tsx:88` (`<IssuesStoreContext.Provider value={EIssuesStoreType.CYCLE}>`)
- Module 布局：`module-layout-root.tsx:71` (`<IssuesStoreContext.Provider value={EIssuesStoreType.MODULE}>`)

### 3.3 阶段 3: Store 实例选择

`useIssues()` 根据 `storeType` 返回对应的 Store 实例：

```typescript
// use-issues.ts:122-126
case EIssuesStoreType.CYCLE:
  return merge(defaultStore, {
    issues: context.issue.cycleIssues,      // 单例实例
    issuesFilter: context.issue.cycleIssuesFilter,
  }) as TStoreIssues[T];
```

**重要设计**：
- `cycleIssues` 和 `moduleIssues` 都是**单例 Store 实例**
- 所有 Cycle 视图共享同一个 `CycleIssues` 实例
- 内部通过 computed `cycleId` 区分当前操作的 Cycle
- 切换 Cycle 时复用同一个 Store 实例，只是清空旧数据填充新数据

**当前 Cycle ID 的获取**：
```typescript
// base-issues.store.ts:274-277
get cycleId() {
  return this.rootIssueStore.cycleId;  // 从 root store 获取当前 cycleId
}
```

### 3.4 阶段 4: 视图挂载与数据加载

三种视图根组件都在 `useEffect` 中触发数据加载：

**看板视图**：
```typescript
// base-kanban-root.tsx:98-100
useEffect(() => {
  fetchIssues("init-loader", { canGroup: true, perPageCount: sub_group_by ? 10 : 30 }, viewId);
}, [fetchIssues, storeType, group_by, sub_group_by, viewId]);
```

**列表视图**：
```typescript
// base-list-root.tsx:89-91
useEffect(() => {
  fetchIssues("init-loader", { canGroup: true, perPageCount: group_by ? 50 : 100 }, viewId);
}, [fetchIssues, storeType, group_by, viewId]);
```

**甘特视图**：
```typescript
// base-gantt-root.tsx:67-69
useEffect(() => {
  fetchIssues("init-loader", { canGroup: false, perPageCount: 100 }, viewId);
}, [fetchIssues, storeType, viewId]);
```

**触发条件**：
- 组件首次挂载
- `storeType` 变化（如从 Project 切换到 Cycle）
- `group_by` / `sub_group_by` 变化（分组维度改变）
- `viewId` 变化（切换到另一个 Cycle/Module）

### 3.5 阶段 5: 数据回源与本地覆盖

`fetchIssues` 的完整执行流程：

```typescript
// cycle/issue.store.ts:186-216
fetchIssues = async (workspaceSlug, projectId, loadType, options, cycleId) => {
  try {
    // 1. 设置加载状态，清空旧数据
    runInAction(() => {
      this.setLoader(loadType);
      this.clear(!isExistingPaginationOptions);  // 关键：清空 Cycle A 的数据
    });

    // 2. 调用后端 API: GET /cycles/B/issues
    const params = this.issueFilterStore?.getFilterParams(options, cycleId, ...);
    const response = await this.issueService.getIssues(workspaceSlug, projectId, params, ...);

    // 3. 处理响应，更新 Store
    this.onfetchIssues(response, options, workspaceSlug, projectId, cycleId, ...);
    return response;
  } catch (error) {
    this.setLoader(undefined);
    throw error;
  }
};
```

`onfetchIssues` 的核心处理逻辑：

```typescript
// base-issues.store.ts:460-488
onfetchIssues(issuesResponse, options, workspaceSlug, projectId, id) {
  // 1. 处理响应数据
  const { issueList, groupedIssues, groupedIssueCount } = this.processIssueResponse(issuesResponse);

  // 2. 更新全局 Issue Map（L1 层）
  this.rootIssueStore.issues.addIssue(issueList);

  // 3. 更新分组数据（L2 层）- 关键：覆盖旧数据
  runInAction(() => {
    this.clear(shouldClearPaginationOptions);                    // 清空旧分组
    this.updateGroupedIssueIds(groupedIssues, groupedIssueCount); // 设置新分组
    this.loader[getGroupKey()] = undefined;
  });

  // 4. 拉取容器统计数据（进度、计数等）
  this.fetchParentStats(workspaceSlug, projectId, id);  // 调用 fetchCycleDetails

  // 5. 提取关联关系
  this.rootIssueStore.issueDetail.relation.extractRelationsFromIssues(issueList);

  // 6. 存储分页信息
  this.storePreviousPaginationValues(issuesResponse, options);
}
```

**关键操作**：
- `clear()`：清空当前 Store 的 `groupedIssueIds`，移除 Cycle A 的所有分组数据
- `updateGroupedIssueIds()`：用 Cycle B 的新分组数据覆盖
- `fetchParentStats()`：调用 `fetchCycleDetails` 拉取 Cycle B 的详情和统计

### 3.6 阶段 6: 视图渲染更新

由于所有视图组件都包裹在 `observer` HOC 中，当 `groupedIssueIds` 变化时：

1. MobX 检测到 observable 变化
2. 自动触发组件 re-render
3. 视图从 props 中获取最新的 `groupedIssueIds`
4. 重新渲染对应的卡片/行/甘特块

**代码依据**：
- 看板：`base-kanban-root.tsx:55` (`observer(function BaseKanBanRoot...)`)
- 列表：`base-list-root.tsx:50` (`observer(function BaseListRoot...)`)
- 甘特：`base-gantt-root.tsx:47` (`observer(function BaseGanttRoot...)`)

---

## 4. 本地缓存与数据优先级

### 4.1 缓存层级

| 层级 | 存储位置 | 生命周期 | 刷新时机 |
|-----|---------|---------|---------|
| L1 全局字典 | `rootIssueStore.issues.issuesMap` | 页面生命周期内持久 | Issue 增删改时更新 |
| L2 视图分组 | `cycleIssues.groupedIssueIds` | 单例 Store，跨 Cycle 切换复用 | 切换 Cycle 时被 `clear()` 清空并覆盖 |
| L3 组件状态 | React `useState` | 组件生命周期 | 组件卸载时清空 |

### 4.2 数据优先级

```
API 返回数据 > 本地乐观更新 > 缓存数据
```

**关键行为**：
- 切换到新 Cycle/Module 时，`clear()` 会清空 L2 层的旧数据
- 然后用 API 返回的新数据填充
- L1 层的全局字典不会被清空，只会被更新（新 Issue 的数据会覆盖旧数据）

### 4.3 Store 单例设计的影响

由于 `CycleIssues` 和 `ModuleIssues` 是单例：

✅ **优点**：
- 内存占用低，不需要为每个 Cycle 创建独立 Store
- 切换时只需要更新数据，不需要重新创建 Store 实例

❌ **缺点**：
- 同时只能保存一个 Cycle/Module 的分组数据
- 切换回之前访问过的 Cycle/Module 时，必须重新从 API 拉取
- 没有本地缓存历史 Cycle/Module 的分组数据

---

## 5. 可复现 UI 场景：旧值何时被新值替换

### 5.1 场景设定

```
初始状态：
- Cycle A：包含 10 个 Issue，3 个已完成，进度 30%
- Cycle B：包含 20 个 Issue，10 个已完成，进度 50%

用户操作序列：
T0: 用户在 Cycle A 视图（已加载完成）
T1: 用户批量删除 Cycle A 中的 2 个 Issue
T2: 用户切换到 Cycle B 视图
T3: 用户在 Cycle B 视图中批量更新 5 个 Issue 状态为"已完成"
T4: 用户切换回 Cycle A 视图
```

### 5.2 时序分析

**T0: Cycle A 视图已加载**
```
cycleIssues.groupedIssueIds = {
  "backlog": ["issue-1", "issue-2"],
  "started": ["issue-3", "issue-4", "issue-5"],
  "completed": ["issue-6", "issue-7", "issue-8"]
}
cycleMap["A"].progress = 30%
```

**T1: 批量删除 Cycle A 中的 2 个 Issue**
```
操作: removeBulkIssues(["issue-1", "issue-2"])
结果:
- issueMap 中移除 issue-1, issue-2
- groupedIssueIds 中移除这两个 ID
- fetchParentStats() 回源刷新 Cycle A 统计
- cycleMap["A"].progress = 1/8 = 12.5% (假设删除的是未开始的)
- 视图即时更新，Issue 从列表消失
```

**T2: 切换到 Cycle B 视图**
```
阶段 1: 路由解析 → cycleId = "B"
阶段 2: useIssueStoreType → CYCLE
阶段 3: useIssues → 返回 cycleIssues 单例
阶段 4: useEffect 触发 fetchIssues("init-loader", ..., "B")
阶段 5: 
  - clear() 清空 groupedIssueIds (Cycle A 的数据被清除)
  - API 调用 GET /cycles/B/issues
  - onfetchIssues() 处理响应:
    * groupedIssueIds 设置为 Cycle B 的分组数据
    * fetchParentStats() 拉取 Cycle B 详情
阶段 6: 视图渲染 Cycle B 的 20 个 Issue
结果:
- cycleIssues.groupedIssueIds = { Cycle B 的分组数据 }
- cycleMap["B"].progress = 50%
- cycleMap["A"].progress 仍为 12.5% (保留在内存中)
```

**T3: 在 Cycle B 视图批量更新 5 个 Issue 状态**
```
操作: bulkUpdateProperties([5 个 started → completed])
结果:
- issueMap 中更新这 5 个 Issue 的 state
- updateIssueList() 移动到 completed 分组
- ❌ 不调用 updateParentStats，不调用 fetchParentStats
- cycleMap["B"].progress 仍为 50% (未更新)
- 视图中 Issue 移动到"已完成"列
```

**T4: 切换回 Cycle A 视图**
```
阶段 1: 路由解析 → cycleId = "A"
阶段 2: useIssueStoreType → CYCLE
阶段 3: useIssues → 返回 cycleIssues 单例 (仍是那个实例)
阶段 4: useEffect 检测到 viewId 变化 → fetchIssues("init-loader", ..., "A")
阶段 5:
  - clear() 清空 groupedIssueIds (Cycle B 的数据被清除)
  - API 调用 GET /cycles/A/issues (后端实时计算)
  - onfetchIssues() 处理响应:
    * groupedIssueIds 设置为 Cycle A 的最新分组数据
    * fetchParentStats() 拉取 Cycle A 详情
阶段 6: 视图渲染 Cycle A 的 8 个 Issue
结果:
- cycleIssues.groupedIssueIds = { Cycle A 的最新分组数据 }
- cycleMap["A"].progress 从 API 获取最新值 (仍是 12.5%，如果没有其他修改)
- cycleMap["B"].progress 仍为 50% (内存中的旧值，实际应为 15/20=75%)
```

### 5.3 关键观察点

| 时间点 | Cycle A 进度 (UI 显示) | Cycle B 进度 (UI 显示) | 一致性状态 |
|-------|------------------------|------------------------|-----------|
| T0 | 30% ✅ | 未知 | Cycle A 一致 |
| T1 | 12.5% ✅ | 未知 | Cycle A 一致 |
| T2 | 内存中 12.5% | 50% ✅ | Cycle B 一致，Cycle A 内存中保留 |
| T3 | 内存中 12.5% | 50% ❌ (实际 75%) | Cycle B 统计不一致 |
| T4 | 12.5% ✅ (从 API 刷新) | 内存中 50% ❌ | Cycle A 一致，Cycle B 内存中旧值 |

**旧值被新值替换的精确时机**：
- 在 `T4` 阶段 5 的 `onfetchIssues()` 中，当 `fetchParentStats()` 完成时
- 具体是 `cycle.store.fetchCycleDetails()` API 返回后，`cycleMap["A"]` 被更新
- 此时 UI 上的进度条、计数卡片等会从旧值（如果有的话）跳转到新值

---

## 6. Module 收敛流程的差异

Module 的收敛流程与 Cycle 基本一致，只有以下细微差异：

| 差异点 | Cycle | Module |
|-------|-------|--------|
| Store 单例 | `cycleIssues` | `moduleIssues` |
| 父统计拉取 | `fetchCycleDetails` + 可选 `fetchActiveCycleProgressPro` | 仅 `fetchModuleDetails` |
| 进度快照 | 有 `progress_snapshot` 字段 | 无 |
| 多对多关系 | 一个 Issue 只能属于一个 Cycle | 一个 Issue 可以属于多个 Module |

**关键代码**：
```typescript
// module/issue.store.ts:94-99
fetchParentStats = (workspaceSlug, projectId, id) => {
  const moduleId = id ?? this.moduleId;
  if (projectId && moduleId) {
    this.rootIssueStore.rootStore.module.fetchModuleDetails(workspaceSlug, projectId, moduleId);
  }
};
```

---

## 7. 关键代码索引

| 功能模块 | 文件路径 | 关键行 |
|---------|---------|-------|
| Cycle 布局根组件 | `apps/web/core/components/issues/issue-layouts/roots/cycle-layout-root.tsx` | 53-133 |
| Module 布局根组件 | `apps/web/core/components/issues/issue-layouts/roots/module-layout-root.tsx` | 45-102 |
| Store 类型选择 Hook | `apps/web/core/hooks/use-issue-layout-store.ts` | 14-43 |
| useIssues Hook | `apps/web/core/hooks/store/use-issues.ts` | 88-163 |
| 看板根组件 | `apps/web/core/components/issues/issue-layouts/kanban/base-kanban-root.tsx` | 55-298 |
| 列表根组件 | `apps/web/core/components/issues/issue-layouts/list/base-list-root.tsx` | 50-182 |
| 甘特根组件 | `apps/web/core/components/issues/issue-layouts/gantt/base-gantt-root.tsx` | 47-157 |
| Cycle Issue Store | `apps/web/core/store/issue/cycle/issue.store.ts` | 103-216 |
| Module Issue Store | `apps/web/core/store/issue/module/issue.store.ts` | 73-140 |
| Base Issues Store | `apps/web/core/store/issue/helpers/base-issues.store.ts` | 460-488 |
| useIssuesActions Hook | `apps/web/core/hooks/use-issues-actions.tsx` | 47-807 |

---

## 8. 总结

### 8.1 收敛五部曲

非当前容器的数据收敛必须经过以下五个步骤，缺一不可：

1. **路由驱动**：用户点击导航或输入 URL，触发路由变化
2. **Store 选择**：根据 URL 参数确定 Store 类型（CYCLE/MODULE）
3. **数据回源**：视图组件挂载后通过 `useEffect` 调用 `fetchIssues` 从 API 拉取最新数据
4. **本地覆盖**：`clear()` 清空旧数据，用 API 返回的新数据覆盖 `groupedIssueIds`
5. **视图渲染**：MobX observer 自动检测变化并重新渲染

### 8.2 无后台同步机制

系统**没有**任何后台自动同步机制：
- ❌ 没有轮询（polling）
- ❌ 没有 WebSocket 推送
- ❌ 没有 Service Worker 后台同步
- ❌ 没有跨标签页通信（BroadcastChannel）

所有数据同步完全依赖用户的主动操作（路由切换、手动刷新）。

### 8.3 设计权衡

**优点**：
1. **简单可靠**：拉模式比推模式更容易实现和调试
2. **最终一致**：只要用户访问，就能看到最新数据
3. **节省资源**：不需要维护长连接或定时任务

**缺点**：
1. **数据延迟**：非当前容器的数据可能长时间不一致
2. **用户困惑**：批量操作后切换视图可能看到"数据回滚"的错觉（其实是旧缓存）
3. **无法协作**：多用户场景下看不到其他人的实时修改

这种设计对于项目管理工具来说是合理的权衡——进度统计不需要毫秒级实时性，而简化架构带来的稳定性收益远大于实时性需求。
