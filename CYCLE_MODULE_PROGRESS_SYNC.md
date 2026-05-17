# Cycle 与 Module 进度同步机制详解

## 1. 核心设计原则

Plane 系统中 Cycle 与 Module 的进度同步采用 **"后端实时计算 + 前端乐观更新 + 最终回源校验"** 的三层架构，确保统计数据的准确性与用户体验的响应性。

---

## 2. 后端统计计算机制

### 2.1 实时聚合，无主动缓存失效

**关键结论**：后端不维护实时更新的统计缓存，每次 API 请求都通过 Django ORM 的 `annotate` 机制实时计算。

| 组件 | 实现方式 | 代码依据 |
|------|---------|---------|
| Cycle 统计 | 通过 `Count` 聚合 `CycleIssue` 中间表，按 `state__group` 过滤 | `apps/api/plane/app/views/cycle/base.py:114-151` |
| Module 统计 | 通过子查询 `Subquery` 分别计算各状态分组的 Issue 数量 | `apps/api/plane/app/views/module/base.py:86-292` |

**Cycle 聚合示例**：
```python
# apps/api/plane/app/views/cycle/base.py:114-151
.annotate(total_issues=Count(
    "issue_cycle__issue__id",
    distinct=True,
    filter=Q(
        issue_cycle__issue__archived_at__isnull=True,
        issue_cycle__issue__is_draft=False,
        issue_cycle__deleted_at__isnull=True,
        issue_cycle__issue__deleted_at__isnull=True,
    ),
))
.annotate(completed_issues=Count(
    "issue_cycle__issue__id",
    distinct=True,
    filter=Q(issue_cycle__issue__state__group="completed", ...),
))
```

**Module 聚合示例**：
```python
# apps/api/plane/app/views/module/base.py:86-105
completed_issues = (
    Issue.issue_objects.filter(
        state__group="completed",
        issue_module__module_id=OuterRef("pk"),
        issue_module__deleted_at__isnull=True,
    )
    .values("issue_module__module_id")
    .annotate(cnt=Count("pk"))
    .values("cnt")
)
```

### 2.2 后端缓存机制

| 缓存类型 | 适用场景 | 刷新时机 | 代码依据 |
|---------|---------|---------|---------|
| `@cache_response` 装饰器 | API 响应缓存 | 1小时超时自动失效 | `apps/api/plane/utils/cache.py:25-51` |
| `progress_snapshot` (Cycle 特有) | 周期转移时的历史快照 | 仅在 `transfer_cycle_issues` 时写入 | `apps/api/plane/utils/cycle_transfer_issues.py:408-432` |

> **重要说明**：`progress_snapshot` 是只读历史快照，用于已结束周期的数据分析，不参与实时进度计算。

---

## 3. 前端更新策略

### 3.1 双阶段更新流程

工作项变更触发的进度更新遵循 **"乐观本地更新 → 后端API调用 → 回源校验"** 的三阶段流程：

```
用户操作触发工作项变更
        ↓
1. updateParentStats() → 乐观本地更新 (updateDistribution)
        ↓
2. API 调用 (patchIssue / deleteIssue 等)
        ↓
3. fetchParentStats() → 从后端回源刷新最新数据
```

**代码依据**：`apps/web/core/store/issue/helpers/base-issues.store.ts:554-587`

```typescript
async issueUpdate(workspaceSlug, projectId, issueId, data, shouldSync = true) {
  const issueBeforeUpdate = clone(this.rootIssueStore.issues.getIssueById(issueId));
  try {
    // 1. 本地乐观更新
    this.rootIssueStore.issues.updateIssue(issueId, data);
    this.updateIssueList({ ...issueBeforeUpdate, ...data }, issueBeforeUpdate);
    this.updateParentStats(issueBeforeUpdate, { ...issueBeforeUpdate, ...data });

    // 2. 后端API调用
    await this.issueService.patchIssue(workspaceSlug, projectId, issueId, data);

    // 3. 回源刷新
    this.fetchParentStats(workspaceSlug, projectId);
  } catch (error) {
    // 失败回滚
    this.rootIssueStore.issues.updateIssue(issueId, issueBeforeUpdate ?? {});
    throw error;
  }
}
```

### 3.2 本地增量更新 (updateDistribution)

**适用场景**：工作项状态变更、负责人变更、标签变更等可预测的统计变化。

**实现机制**：
```typescript
// apps/web/core/store/issue/cycle/issue.store.ts:159-175
updateParentStats = (prevIssueState, nextIssueState, id) => {
  const distributionUpdates = getDistributionPathsPostUpdate(
    prevIssueState,
    nextIssueState,
    this.rootIssueStore.rootStore.state.stateMap,
    this.rootIssueStore.rootStore.projectEstimate?.currentActiveEstimate?.estimatePointById
  );
  const cycleId = id ?? this.cycleId;
  if (cycleId) {
    this.rootIssueStore.rootStore.cycle.updateCycleDistribution(distributionUpdates, cycleId);
  }
};
```

**代码依据**：
- Cycle：`apps/web/core/store/cycle.store.ts:564-571`
- Module：`apps/web/core/store/module.store.ts:383-391`

### 3.3 回源刷新 (fetchParentStats)

**适用场景**：API 调用完成后，确保前端数据与后端一致。

| 容器类型 | 回源方法 | 代码依据 |
|---------|---------|---------|
| Cycle | `fetchCycleDetails` + 条件性 `fetchActiveCycleProgressPro` | `apps/web/core/store/issue/cycle/issue.store.ts:140-157` |
| Module | `fetchModuleDetails` | `apps/web/core/store/issue/module/issue.store.ts:94-99` |

---

## 4. 触发场景与更新策略对照表

| 操作场景 | 本地增量更新 | 后端API | 回源刷新 | 代码位置 |
|---------|------------|---------|---------|---------|
| **创建工作项** | ❌ 不适用 | `createIssue` | ✅ `fetchParentStats` | `base-issues.store.ts:526-543` |
| **更新工作项属性** (状态/负责人/标签等) | ✅ `updateParentStats` | `patchIssue` | ✅ `fetchParentStats` | `base-issues.store.ts:554-587` |
| **删除工作项** | ✅ `updateParentStats(prev, undefined)` | `deleteIssue` | ✅ `fetchParentStats` | `base-issues.store.ts:596-612` |
| **归档工作项** | ✅ `updateParentStats(prev, undefined)` | `archiveIssue` | ✅ `fetchParentStats` | `base-issues.store.ts:620-636` |
| **批量删除** | ❌ 不支持 | `bulkDeleteIssues` | ✅ `fetchParentStats` | `base-issues.store.ts:677-690` |
| **批量归档** | ❌ 不支持 | `bulkArchiveIssues` | ❌ 不回源 | `base-issues.store.ts:698-715` |
| **批量更新属性** | ❌ 不支持 | `bulkOperations` | ❌ 不回源 | `base-issues.store.ts:721-753` |
| **添加到 Cycle** | ✅ `updateParentStats` (条件) | `addIssueToCycle` | ✅ `fetchParentStats` (条件) | `base-issues.store.ts:808-838` |
| **从 Cycle 移除** | ✅ `updateParentStats` | `removeIssueFromCycle` | ✅ `fetchParentStats` | `base-issues.store.ts:847-866` |
| **添加到 Module** | ❌ 不支持 | `addIssuesToModule` | ✅ `fetchParentStats` (条件) | `base-issues.store.ts:971-1000` |
| **从 Module 移除** | ❌ 不支持 | `removeIssuesFromModuleBulk` | ✅ `fetchParentStats` (条件) | `base-issues.store.ts:1010-1032` |
| **工作项内修改 Cycle** | ✅ `updateParentStats` (条件) | `addIssueToCycle` | ✅ `fetchParentStats` (条件) | `base-issues.store.ts:876-918` |
| **工作项内移除 Cycle** | ✅ `updateParentStats` | `removeIssueFromCycle` | ✅ `fetchParentStats` | `base-issues.store.ts:927-961` |
| **工作项内修改 Module** | ✅ `updateParentStats` (条件) | `addModulesToIssue` | ✅ `fetchParentStats` (条件) | `base-issues.store.ts:1075-1134` |

---

## 5. Cycle 与 Module 统计同步机制

### 5.1 实体关系与数据隔离

```
Issue (工作项)
   ├─ cycle_id (单个 Cycle)
   └─ module_ids (多个 Module)
        ↓
CycleIssue 中间表 (Cycle 1:N Issue)
ModuleIssue 中间表 (Module 1:N Issue)
```

- 一个工作项只能属于 **一个** Cycle
- 一个工作项可以属于 **多个** Module
- Cycle 与 Module 的统计完全独立，通过各自的中间表聚合

### 5.2 状态变更时的同步

当工作项状态变更时：

1. **Cycle 统计更新**：
   - 触发位置：`CycleIssues.updateParentStats()`
   - 代码：`apps/web/core/store/issue/cycle/issue.store.ts:159-175`

2. **Module 统计更新**：
   - 触发位置：`ModuleIssues.updateParentStats()`
   - 代码：`apps/web/core/store/issue/module/issue.store.ts:107-123`

> **关键区别**：在 Cycle 视图下修改工作项，只会更新当前 Cycle 的统计；在 Module 视图下修改，只会更新当前 Module 的统计。跨容器的统计同步依赖于下次访问时的 API 回源。

### 5.3 关联关系变更时的同步

**添加/移除 Cycle**：
```typescript
// apps/web/core/store/issue/helpers/base-issues.store.ts:876-918
addCycleToIssue = async (workspaceSlug, projectId, cycleId, issueId) => {
  // 1. 本地乐观更新
  // 2. updateParentStats (如果影响当前 Cycle)
  // 3. API 调用
  // 4. fetchParentStats (如果影响当前 Cycle)
};
```

**添加/移除 Module**：
```typescript
// apps/web/core/store/issue/helpers/base-issues.store.ts:1075-1134
changeModulesInIssue = async (workspaceSlug, projectId, issueId, addModuleIds, removeModuleIds) => {
  // 1. 本地乐观更新
  // 2. updateParentStats (如果影响当前 Module)
  // 3. API 调用
  // 4. fetchParentStats (如果影响当前 Module)
};
```

---

## 6. 数据一致性保证

### 6.1 一致性层级

| 层级 | 保证方式 | 延迟 |
|-----|---------|-----|
| 当前视图容器 | 乐观本地更新 + 回源校验 | 即时（本地）→ API 往返时间（最终一致） |
| 非当前视图容器 | 依赖下次 API 请求 | 直到用户访问该容器页面 |
| 其他用户会话 | 依赖前端轮询或 WebSocket（如有） | 取决于刷新策略 |

### 6.2 失败回滚机制

所有乐观更新都包含错误处理，API 失败时自动回滚本地状态：

```typescript
// apps/web/core/store/issue/helpers/base-issues.store.ts:582-586
catch (error) {
  this.rootIssueStore.issues.updateIssue(issueId, issueBeforeUpdate ?? {});
  this.updateIssueList(issueBeforeUpdate, { ...issueBeforeUpdate, ...data });
  throw error;
}
```

---

## 7. 关键代码索引

| 功能模块 | 文件路径 | 关键行 |
|---------|---------|-------|
| 工作项更新主流程 | `apps/web/core/store/issue/helpers/base-issues.store.ts` | 554-587 |
| Cycle 本地分布更新 | `apps/web/core/store/cycle.store.ts` | 564-571 |
| Module 本地分布更新 | `apps/web/core/store/module.store.ts` | 383-391 |
| Cycle 问题子 Store | `apps/web/core/store/issue/cycle/issue.store.ts` | 140-175 |
| Module 问题子 Store | `apps/web/core/store/issue/module/issue.store.ts` | 94-123 |
| 分布更新工具函数 | `packages/utils/src/distribution-update.ts` | 206-268 |
| Cycle 后端聚合 | `apps/api/plane/app/views/cycle/base.py` | 114-181 |
| Module 后端聚合 | `apps/api/plane/app/views/module/base.py` | 86-292 |
| 周期转移快照 | `apps/api/plane/utils/cycle_transfer_issues.py` | 408-432 |
| 后端缓存装饰器 | `apps/api/plane/utils/cache.py` | 25-51 |

---

## 8. 总结

Plane 系统采用的进度同步策略体现了典型的 **"最终一致性"** 架构：

1. **后端**：不维护实时统计缓存，每次请求实时计算，简单可靠但牺牲了极致性能
2. **前端**：乐观更新提供即时反馈，回源刷新保证数据准确
3. **跨容器同步**：基于用户访问驱动，非当前视图容器的统计数据可能存在延迟

这种设计在性能和一致性之间取得了平衡，对于项目管理工具来说是合理的权衡。
