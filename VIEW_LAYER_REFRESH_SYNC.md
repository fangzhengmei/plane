# 视图层刷新同步机制全景：看板 / 列表 / 甘特

## 1. 核心架构概览

Plane 系统的视图层采用 **"单一数据源 + 多视图派生"** 的架构设计：

```
┌─────────────────────────────────────────────────────────────────┐
│                      Root Issue Store                          │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │  issueMap (全局 Issue 数据字典)                           │  │
│  │  issues: Record<string, TIssue>                           │  │
│  └───────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
                              ▲
                              │ 共享数据源
              ┌───────────────┼───────────────┐
              │               │               │
              ▼               ▼               ▼
  ┌─────────────────┐ ┌─────────────────┐ ┌─────────────────┐
  │ CycleIssues     │ │ ModuleIssues    │ │ ProjectIssues   │
  │  - groupedIssue │ │  - groupedIssue │ │  - groupedIssue │
  │  - view-specific│ │  - view-specific│ │  - view-specific│
  └─────────────────┘ └─────────────────┘ └─────────────────┘
              │               │               │
              ▼               ▼               ▼
  ┌─────────────────────────────────────────────────────────┐
  │              BaseIssuesStore (基类)                     │
  │  - updateIssueList()    // 更新分组数据                  │
  │  - removeIssueFromList() // 从分组移除                   │
  │  - updateParentStats()  // 更新容器统计                  │
  │  - fetchParentStats()   // 回源刷新统计                  │
  └─────────────────────────────────────────────────────────┘
              │               │               │
              ▼               ▼               ▼
  ┌─────────────────┐ ┌─────────────────┐ ┌─────────────────┐
  │  看板视图       │ │  列表视图       │ │  甘特视图       │
  │  Kanban Group   │ │  List Block     │ │  Gantt Block    │
  │  横向拖拽更新    │ │  纵向列表更新    │ │  时间轴更新     │
  └─────────────────┘ └─────────────────┘ └─────────────────┘
```

---

## 2. 数据流向总览

### 2.1 三层数据结构

| 层级 | 存储位置 | 内容 | 代码位置 |
|-----|---------|------|---------|
| L1 全局字典 | `RootStore.issue.issues.issuesMap` | 所有 Issue 的完整数据 | `apps/web/core/hooks/store/use-issues.ts:93` |
| L2 视图分组 | `BaseIssuesStore.groupedIssueIds` | 按分组维度组织的 Issue ID 列表 | `apps/web/core/store/issue/helpers/base-issues.store.ts:61` |
| L3 渲染层 | 各视图组件的 props | 从 Store 派生的展示数据 | `base-kanban-root.tsx:269-291` |

### 2.2 数据流动方向

```
用户操作 → BaseIssuesStore 方法 → 更新 L2 groupedIssueIds
              ↓ (MobX observable 自动响应)
        视图组件 re-render → 渲染更新
              ↓ (可选)
        API 调用 → 后端持久化 → fetchParentStats() 回源校验
```

---

## 3. 批量操作刷新链路

### 3.1 批量删除 (`removeBulkIssues`)

**执行流程**：

```
1. API 调用: bulkDeleteIssues()
   代码: base-issues.store.ts:679

2. 回源刷新当前容器统计: fetchParentStats()
   代码: base-issues.store.ts:681

3. 本地 Issue 列表清理:
   ├─ removeIssueFromList(issueId) → 触发 updateIssueList(undefined, issue, DELETE)
   │  代码: base-issues.store.ts:685
   └─ rootIssueStore.issues.removeIssue(issueId) → 更新 L1 全局字典
      代码: base-issues.store.ts:686
```

**刷新路径详解**：

| 视图 | 刷新触发点 | 数据更新机制 | UI 可见性 |
|-----|-----------|-------------|----------|
| 看板 | `removeIssueFromList` → `updateIssueList` → 更新 `groupedIssueIds` | MobX 响应式更新，`KanBan` 组件从 props 获取 `groupedIssueIds` 重新渲染 | ✅ 即时从看板列中消失 |
| 列表 | `removeIssueFromList` → `updateIssueList` → 更新 `groupedIssueIds` | `List` 组件重新渲染，Issue Block 从列表移除 | ✅ 即时从列表中消失 |
| 甘特 | `removeIssueFromList` → `updateIssueList` → 更新 `groupedIssueIds[ALL_ISSUES]` | `GanttChartRoot` 组件的 `blockIds` props 更新，甘特块消失 | ✅ 即时从时间轴中消失 |

**代码依据**：
```typescript
// base-issues.store.ts:677-690
async removeBulkIssues(workspaceSlug, projectId, issueIds) {
  const response = await this.issueService.bulkDeleteIssues(workspaceSlug, projectId, { issue_ids: issueIds });
  this.fetchParentStats(workspaceSlug, projectId);  // ✅ 回源刷新
  runInAction(() => {
    issueIds.forEach((issueId) => {
      this.removeIssueFromList(issueId);  // 从视图分组移除
      this.rootIssueStore.issues.removeIssue(issueId);  // 从全局字典移除
    });
  });
}
```

### 3.2 批量归档 (`bulkArchiveIssues`)

**执行流程**：

```
1. API 调用: bulkArchiveIssues()
   代码: base-issues.store.ts:699

2. 本地逐个更新 Issue (shouldSync=false):
   ├─ issueUpdate(workspaceSlug, projectId, issueId, { archived_at }, false)
   │  ├─ updateIssueList(issue, issueBeforeUpdate)
   │  └─ ❌ shouldSync=false: 跳过 updateParentStats 和 fetchParentStats
   │     代码: base-issues.store.ts:569
   └─ removeIssueFromList(issueId) → 从分组移除
      代码: base-issues.store.ts:712

3. ❌ 不调用 fetchParentStats()
   代码: base-issues.store.ts:698-715 (无调用)
```

**刷新路径详解**：

| 视图 | 刷新触发点 | 数据更新机制 | UI 可见性 | 容器统计 |
|-----|-----------|-------------|----------|---------|
| 看板 | `issueUpdate` → `updateIssueList` → 从分组移除 | Issue 卡片从看板列中消失 | ✅ 即时消失 | ❌ 不更新 |
| 列表 | `issueUpdate` → `updateIssueList` → 从分组移除 | Issue 行从列表中消失 | ✅ 即时消失 | ❌ 不更新 |
| 甘特 | `issueUpdate` → `updateIssueList` → 从 `ALL_ISSUES` 分组移除 | 甘特块从时间轴消失 | ✅ 即时消失 | ❌ 不更新 |

**关键问题**：批量归档后，Issue 从视图中消失，但 Cycle/Module 的进度统计、计数等不会更新，直到用户刷新页面或切换视图。

**代码依据**：
```typescript
// base-issues.store.ts:698-715
bulkArchiveIssues = async (workspaceSlug, projectId, issueIds) => {
  const response = await this.issueService.bulkArchiveIssues(workspaceSlug, projectId, { issue_ids: issueIds });
  runInAction(() => {
    issueIds.forEach((issueId) => {
      this.issueUpdate(
        workspaceSlug,
        projectId,
        issueId,
        { archived_at: response.archived_at },
        false  // ❌ shouldSync=false，跳过统计更新
      );
      this.removeIssueFromList(issueId);
    });
  });
  // ❌ 注意：不调用 fetchParentStats
};
```

### 3.3 批量更新属性 (`bulkUpdateProperties`)

**执行流程**：

```
1. API 调用: bulkOperations()
   代码: base-issues.store.ts:724

2. 本地逐个更新 Issue 对象:
   ├─ rootIssueStore.issues.updateIssue(issueId, updatedProperties)
   │  → 仅更新 L1 全局字典
   │  代码: base-issues.store.ts:734-746
   ├─ updateIssueList(issueDetails, issueBeforeUpdate)
   │  → 更新 L2 分组数据（如状态变更则移动到对应分组）
   │  代码: base-issues.store.ts:750
   └─ ❌ 不调用 updateParentStats
      代码: base-issues.store.ts:721-753 (无调用)

3. ❌ 不调用 fetchParentStats()
   代码: base-issues.store.ts:721-753 (无调用)
```

**刷新路径详解**：

| 视图 | 刷新触发点 | 数据更新机制 | UI 可见性 | 容器统计 |
|-----|-----------|-------------|----------|---------|
| 看板 | `updateIssueList` → 重新计算 Issue 的分组路径 | 如果分组字段被修改（如状态），Issue 卡片会移动到对应的看板列 | ✅ 即时移动/更新 | ❌ 不更新 |
| 列表 | `updateIssueList` → 重新排序/重分组 | 如果排序或分组字段被修改，列表会重新排序或移动 Issue 行 | ✅ 即时更新 | ❌ 不更新 |
| 甘特 | `updateIssueList` → 更新 `ALL_ISSUES` 分组 | 如果日期被修改，甘特块的位置和长度会更新 | ✅ 即时更新 | ❌ 不更新 |

**关键问题**：
- 如果修改的是状态、负责人等字段，Issue 在视图中的位置会正确更新
- 但 Cycle/Module 的进度统计（如已完成数、进行中数）不会更新
- 如果修改的字段不影响分组（如标题、描述），视图中只更新对应 Issue 的显示内容

**代码依据**：
```typescript
// base-issues.store.ts:721-753
bulkUpdateProperties = async (workspaceSlug, projectId, data) => {
  const issueIds = data.issue_ids;
  await this.issueService.bulkOperations(workspaceSlug, projectId, data);
  runInAction(() => {
    issueIds.forEach((issueId) => {
      const issueBeforeUpdate = clone(this.rootIssueStore.issues.getIssueById(issueId));
      // 更新 L1 全局字典
      this.rootIssueStore.issues.updateIssue(issueId, updatedProperties);
      const issueDetails = this.rootIssueStore.issues.getIssueById(issueId);
      // 更新 L2 分组数据
      this.updateIssueList(issueDetails, issueBeforeUpdate);
    });
  });
  // ❌ 注意：既不调用 updateParentStats，也不调用 fetchParentStats
};
```

---

## 4. `updateIssueList` 核心机制

### 4.1 分组数据更新流程

`updateIssueList` 是连接 Issue 数据变更与视图刷新的核心枢纽：

```typescript
// base-issues.store.ts:1196-1248
updateIssueList(issue?, issueBeforeUpdate?, action?) {
  // 1. 获取 Issue ID
  const issueId = issue?.id ?? issueBeforeUpdate?.id;
  
  // 2. 计算更新路径（从哪些分组移除，添加到哪些分组）
  const issueUpdates = this.getUpdateDetails(issue, issueBeforeUpdate, action);
  
  runInAction(() => {
    for (const issueUpdate of issueUpdates) {
      // 3. 执行 ADD/DELETE/REORDER 操作
      if (issueUpdate.action === EIssueGroupedAction.ADD) {
        update(this, ["groupedIssueIds", ...issueUpdate.path], (issueIds) =>
          this.issuesSortWithOrderBy(uniq(concat(issueIds, issueId)), this.orderBy)
        );
      }
      if (issueUpdate.action === EIssueGroupedAction.DELETE) {
        update(this, ["groupedIssueIds", ...issueUpdate.path], (issueIds) =>
          pull(issueIds, issueId)
        );
      }
      if (issueUpdate.action === EIssueGroupedAction.REORDER) {
        update(this, ["groupedIssueIds", ...issueUpdate.path], (issueIds) =>
          this.issuesSortWithOrderBy(issueIds, this.orderBy)
        );
      }
      
      // 4. 累计计数更新
      this.accumulateIssueUpdates(accumulatedUpdatesForCount, issueUpdate.path, issueUpdate.action);
    }
    
    // 5. 更新分组计数
    this.updateIssueCount(accumulatedUpdatesForCount);
  });
}
```

### 4.2 视图如何响应 `groupedIssueIds` 变化

三种视图都通过 MobX `observer` HOC 响应 `groupedIssueIds` 的变化：

**看板视图**：
```typescript
// base-kanban-root.tsx:69
const { issueMap, issuesFilter, issues } = useIssues(storeType);
const groupedIssueIds = issues?.groupedIssueIds;  // observable

// 传递给 KanBan 组件
<KanBanView
  groupedIssueIds={groupedIssueIds ?? {}}  // 响应式 props
  ...
/>
```
**代码**：`apps/web/core/components/issues/issue-layouts/kanban/base-kanban-root.tsx:69,269-291`

**列表视图**：
```typescript
// base-list-root.tsx:62
const { issuesFilter, issues } = useIssues(storeType);
const groupedIssueIds = issues?.groupedIssueIds as TGroupedIssues | undefined;

// 传递给 List 组件
<List
  groupedIssueIds={groupedIssueIds ?? {}}  // 响应式 props
  ...
/>
```
**代码**：`apps/web/core/components/issues/issue-layouts/list/base-list-root.tsx:62,158-178`

**甘特视图**：
```typescript
// base-gantt-root.tsx:54
const { issues, issuesFilter } = useIssues(storeType);
const issuesIds = (issues.groupedIssueIds?.[ALL_ISSUES] as string[]) ?? [];

// 传递给 GanttChartRoot 组件
<GanttChartRoot
  blockIds={issuesIds}  // 响应式 props
  ...
/>
```
**代码**：`apps/web/core/components/issues/issue-layouts/gantt/base-gantt-root.tsx:54,75,131-152`

### 4.3 MobX 响应式更新机制

由于所有视图组件都包裹在 `observer` HOC 中，当 `groupedIssueIds` 发生变化时：

1. MobX 检测到 observable 变化
2. 自动触发组件 re-render
3. 视图从 props 中获取最新的 `groupedIssueIds`
4. 重新渲染对应的卡片/行/甘特块

**代码依据**：
- 看板：`base-kanban-root.tsx:55` (`observer(function BaseKanBanRoot...)`)
- 列表：`base-list-root.tsx:50` (`observer(function BaseListRoot...)`)
- 甘特：`base-gantt-root.tsx:47` (`observer(function BaseGanttRoot...)`)

---

## 5. 当前容器 vs 非当前容器：收敛差异

### 5.1 关键概念

- **当前容器**：用户当前正在浏览的 Cycle/Module 视图（URL 中包含 `cycleId` 或 `moduleId`）
- **非当前容器**：用户未在浏览的其他 Cycle/Module

### 5.2 Store 实例隔离机制

系统为每种上下文创建独立的 Store 实例：

```typescript
// use-issues.ts:122-131
case EIssuesStoreType.CYCLE:
  return merge(defaultStore, {
    issues: context.issue.cycleIssues,      // 独立的 CycleIssues Store 实例
    issuesFilter: context.issue.cycleIssuesFilter,
  }) as TStoreIssues[T];

case EIssuesStoreType.MODULE:
  return merge(defaultStore, {
    issues: context.issue.moduleIssues,     // 独立的 ModuleIssues Store 实例
    issuesFilter: context.issue.moduleIssuesFilter,
  }) as TStoreIssues[T];
```
**代码**：`apps/web/core/hooks/store/use-issues.ts:122-131`

### 5.3 操作影响范围

| 操作类型 | 当前容器 (Current Store) | 非当前容器 (Other Stores) |
|---------|------------------------|--------------------------|
| **单 Issue 更新** | ✅ `updateParentStats()` 乐观更新<br>✅ `fetchParentStats()` 回源刷新<br>✅ `updateIssueList()` 视图刷新 | ❌ 无任何操作 |
| **批量删除** | ✅ `fetchParentStats()` 回源刷新<br>✅ `removeIssueFromList()` 视图刷新 | ❌ 无任何操作 |
| **批量归档** | ⚠️ `removeIssueFromList()` 视图刷新<br>❌ 不更新容器统计 | ❌ 无任何操作 |
| **批量更新属性** | ⚠️ `updateIssueList()` 视图刷新<br>❌ 不更新容器统计 | ❌ 无任何操作 |

### 5.4 UI 可见差异

#### 场景 1：用户在 Cycle A 视图中批量删除 5 个 Issue

| 位置 | UI 表现 | 数据状态 |
|-----|---------|---------|
| Cycle A 视图 | ✅ 5 个 Issue 即时从视图消失<br>✅ 进度条即时更新（回源刷新） | 完全一致 |
| Cycle B 视图（用户未访问） | ⚠️ 进度条显示旧数据（5 个 Issue 仍计入）<br>⚠️ 列表中仍显示这 5 个 Issue（如果缓存了） | 不一致，直到用户访问 Cycle B |
| 全局 Issue 列表 | ✅ 5 个 Issue 即时消失 | 完全一致 |

#### 场景 2：用户在 Module X 视图中批量更新 10 个 Issue 的状态为"已完成"

| 位置 | UI 表现 | 数据状态 |
|-----|---------|---------|
| Module X 视图 | ✅ 10 个 Issue 移动到"已完成"分组<br>❌ 进度条不更新（仍显示旧的完成率） | 视图数据一致，统计数据不一致 |
| Module Y 视图（用户未访问） | ⚠️ 进度条和列表都显示旧数据 | 完全不一致，直到用户访问 Module Y |
| Cycle A 视图（包含这些 Issue） | ⚠️ 如果用户在 Cycle A 视图，Issue 状态更新但进度条不更新 | 部分不一致 |

### 5.5 非当前容器何时收敛

非当前容器的统计数据只有在以下时机才会更新：

1. **用户主动访问**：切换到该 Cycle/Module 视图时触发 `fetchIssues`
   - 代码：`base-kanban-root.tsx:98-100` (`useEffect` 调用 `fetchIssues`)
   - 代码：`base-list-root.tsx:89-91`
   - 代码：`base-gantt-root.tsx:67-69`

2. **全局刷新**：用户刷新浏览器页面，所有 Store 重新初始化

3. **相关操作触发**：在 Issue 详情页修改该 Issue 的 Cycle/Module 关联时，如果目标容器是当前视图则更新

---

## 6. 容器统计刷新链路

### 6.1 Cycle 统计更新

```typescript
// cycle.store.ts:564-571
updateCycleDistribution = (distributionUpdates, cycleId) => {
  const cycle = this.cycleMap[cycleId];
  if (!cycle) return;
  
  updateDistribution(cycle, distributionUpdates);
};
```
**代码**：`apps/web/core/store/cycle.store.ts:564-571`

`fetchParentStats` 在 Cycle 视图中的实现：
```typescript
// cycle/issue.store.ts:140-157
fetchParentStats = (workspaceSlug, projectId) => {
  if (!this.cycleId) return;
  // 1. 拉取 Cycle 详情（包含统计数据）
  this.rootIssueStore.rootStore.cycle.fetchCycleDetails(workspaceSlug, projectId, this.cycleId);
  // 2. 如果是活跃周期，额外拉取进度数据
  if (this.isActiveCycle) {
    this.rootIssueStore.rootStore.cycle.fetchActiveCycleProgressPro(workspaceSlug, projectId);
  }
};
```
**代码**：`apps/web/core/store/issue/cycle/issue.store.ts:140-157`

### 6.2 Module 统计更新

```typescript
// module.store.ts:383-391
updateModuleDistribution = (distributionUpdates, moduleId) => {
  const module = this.moduleMap[moduleId];
  if (!module) return;
  
  updateDistribution(module, distributionUpdates);
};
```
**代码**：`apps/web/core/store/module.store.ts:383-391`

`fetchParentStats` 在 Module 视图中的实现：
```typescript
// module/issue.store.ts:94-99
fetchParentStats = (workspaceSlug, projectId) => {
  if (!this.moduleId) return;
  // 拉取 Module 详情（包含统计数据）
  this.rootIssueStore.rootStore.module.fetchModuleDetails(workspaceSlug, projectId, this.moduleId);
};
```
**代码**：`apps/web/core/store/issue/module/issue.store.ts:94-99`

---

## 7. 关键代码索引

| 功能模块 | 文件路径 | 关键行 |
|---------|---------|-------|
| BaseIssuesStore 基类 | `apps/web/core/store/issue/helpers/base-issues.store.ts` | 1-1248 |
| 批量删除 | `base-issues.store.ts` | 677-690 |
| 批量归档 | `base-issues.store.ts` | 698-715 |
| 批量更新属性 | `base-issues.store.ts` | 721-753 |
| updateIssueList 核心 | `base-issues.store.ts` | 1196-1248 |
| useIssues Hook | `apps/web/core/hooks/store/use-issues.ts` | 88-163 |
| useIssuesActions Hook | `apps/web/core/hooks/use-issues-actions.tsx` | 47-807 |
| 看板根组件 | `apps/web/core/components/issues/issue-layouts/kanban/base-kanban-root.tsx` | 55-298 |
| 列表根组件 | `apps/web/core/components/issues/issue-layouts/list/base-list-root.tsx` | 50-182 |
| 甘特根组件 | `apps/web/core/components/issues/issue-layouts/gantt/base-gantt-root.tsx` | 47-157 |
| Cycle 统计更新 | `apps/web/core/store/cycle.store.ts` | 564-571 |
| Module 统计更新 | `apps/web/core/store/module.store.ts` | 383-391 |
| Cycle Issue Store | `apps/web/core/store/issue/cycle/issue.store.ts` | 140-175 |
| Module Issue Store | `apps/web/core/store/issue/module/issue.store.ts` | 94-123 |

---

## 8. 总结

### 8.1 视图刷新一致性矩阵

| 操作 | 视图刷新 (当前容器) | 统计更新 (当前容器) | 非当前容器 |
|-----|-------------------|-------------------|-----------|
| 单 Issue 更新 | ✅ 即时 | ✅ 最终一致 | ❌ 不更新 |
| 批量删除 | ✅ 即时 | ✅ 最终一致 | ❌ 不更新 |
| 批量归档 | ✅ 即时 | ❌ 不更新 | ❌ 不更新 |
| 批量更新属性 | ✅ 即时（分组字段变更时） | ❌ 不更新 | ❌ 不更新 |

### 8.2 架构权衡

**设计优势**：
1. **单一数据源**：`issuesMap` 作为唯一真实源，避免多 Store 数据不一致
2. **响应式更新**：MobX observer 自动处理视图刷新，无需手动事件触发
3. **上下文隔离**：每个 Cycle/Module 有独立的 Store 实例，互不干扰
4. **乐观更新**：单 Issue 操作先本地更新再 API 调用，用户体验流畅

**存在的问题**：
1. **批量操作统计断层**：批量归档和批量更新属性后，容器统计数据不更新
2. **跨容器同步延迟**：非当前容器的数据可能长时间不一致
3. **无后台同步**：没有轮询或 WebSocket 机制，多用户协作时数据延迟明显

### 8.3 UI 差异表现

用户在批量操作后会观察到：
- ✅ **视图层面**：Issue 正确地从列表/看板/甘特中消失或移动
- ⚠️ **统计层面**：进度条、计数卡片等可能显示过时数据，直到刷新页面
- ❌ **跨容器**：在其他 Cycle/Module 中完全看不到变化，直到用户访问

这种设计在**开发简单性**和**数据实时性**之间做出了明显的权衡，优先保证了视图层面的即时反馈，而牺牲了统计数据的强一致性。
