# Cycle 与 Module API 端点缓存与同步机制全景

## 1. 核心结论

**关键发现**：Cycle 与 Module 相关的所有 API 端点**均未启用响应缓存**（`@cache_response` 装饰器），也**没有配置失效钩子**（`invalidate_cache`）。所有统计数据完全依赖每次请求时的数据库实时聚合计算。

---

## 2. API 端点清单与缓存状态

### 2.1 Cycle 相关端点（18 个）

| 端点路径 | 视图类 | HTTP 方法 | 读/写 | 缓存启用 | 失效钩子 | 代码位置 |
|---------|--------|----------|------|---------|---------|---------|
| `/api/workspaces/<slug>/projects/<project_id>/cycles/` | `CycleViewSet` | GET/POST | 读+写 | ❌ 无 | ❌ 无 | `app/views/cycle/base.py:64` |
| `/api/workspaces/<slug>/projects/<project_id>/cycles/<pk>/` | `CycleViewSet` | GET/PUT/PATCH/DELETE | 读+写 | ❌ 无 | ❌ 无 | `app/views/cycle/base.py:64` |
| `/api/workspaces/<slug>/projects/<project_id>/cycles/<cycle_id>/cycle-issues/` | `CycleIssueViewSet` | GET/POST | 读+写 | ❌ 无 | ❌ 无 | `app/views/cycle/issue.py:40` |
| `/api/workspaces/<slug>/projects/<project_id>/cycles/<cycle_id>/cycle-issues/<issue_id>/` | `CycleIssueViewSet` | GET/PUT/PATCH/DELETE | 读+写 | ❌ 无 | ❌ 无 | `app/views/cycle/issue.py:40` |
| `/api/workspaces/<slug>/projects/<project_id>/cycles/date-check/` | `CycleDateCheckEndpoint` | GET | 读 | ❌ 无 | ❌ 无 | `app/views/cycle/base.py:520` |
| `/api/workspaces/<slug>/projects/<project_id>/user-favorite-cycles/` | `CycleFavoriteViewSet` | GET/POST | 读+写 | ❌ 无 | ❌ 无 | `app/views/cycle/base.py:559` |
| `/api/workspaces/<slug>/projects/<project_id>/user-favorite-cycles/<cycle_id>/` | `CycleFavoriteViewSet` | DELETE | 写 | ❌ 无 | ❌ 无 | `app/views/cycle/base.py:559` |
| `/api/workspaces/<slug>/projects/<project_id>/cycles/<cycle_id>/transfer-issues/` | `TransferCycleIssueEndpoint` | POST | 写 | ❌ 无 | ❌ 无 | `app/views/cycle/base.py:594` |
| `/api/workspaces/<slug>/projects/<project_id>/cycles/<cycle_id>/user-properties/` | `CycleUserPropertiesEndpoint` | GET/PATCH | 读+写 | ❌ 无 | ❌ 无 | `app/views/cycle/base.py:625` |
| `/api/workspaces/<slug>/projects/<project_id>/cycles/<cycle_id>/archive/` | `CycleArchiveUnarchiveEndpoint` | POST | 写 | ❌ 无 | ❌ 无 | `app/views/cycle/archive.py:40` |
| `/api/workspaces/<slug>/projects/<project_id>/archived-cycles/` | `CycleArchiveUnarchiveEndpoint` | GET | 读 | ❌ 无 | ❌ 无 | `app/views/cycle/archive.py:40` |
| `/api/workspaces/<slug>/projects/<project_id>/archived-cycles/<pk>/` | `CycleArchiveUnarchiveEndpoint` | POST/DELETE | 写 | ❌ 无 | ❌ 无 | `app/views/cycle/archive.py:40` |
| `/api/workspaces/<slug>/projects/<project_id>/cycles/<cycle_id>/progress/` | `CycleProgressEndpoint` | GET | 读 | ❌ 无 | ❌ 无 | `app/views/cycle/base.py:658` |
| `/api/workspaces/<slug>/projects/<project_id>/cycles/<cycle_id>/analytics/` | `CycleAnalyticsEndpoint` | GET | 读 | ❌ 无 | ❌ 无 | `app/views/cycle/base.py:786` |
| `/api/v1/workspaces/<slug>/projects/<project_id>/cycles/` | `CycleListCreateAPIEndpoint` | GET/POST | 读+写 | ❌ 无 | ❌ 无 | `api/views/cycle.py:80` |
| `/api/v1/workspaces/<slug>/projects/<project_id>/cycles/<pk>/` | `CycleDetailAPIEndpoint` | GET/PATCH/DELETE | 读+写 | ❌ 无 | ❌ 无 | `api/views/cycle.py:356` |
| `/api/v1/workspaces/<slug>/projects/<project_id>/cycles/<cycle_id>/cycle-issues/` | `CycleIssueListCreateAPIEndpoint` | GET/POST | 读+写 | ❌ 无 | ❌ 无 | `api/views/cycle.py:798` |
| `/api/v1/workspaces/<slug>/projects/<project_id>/cycles/<cycle_id>/cycle-issues/<issue_id>/` | `CycleIssueDetailAPIEndpoint` | GET/PATCH/DELETE | 读+写 | ❌ 无 | ❌ 无 | `api/views/cycle.py:1005` |

### 2.2 Module 相关端点（14 个）

| 端点路径 | 视图类 | HTTP 方法 | 读/写 | 缓存启用 | 失效钩子 | 代码位置 |
|---------|--------|----------|------|---------|---------|---------|
| `/api/workspaces/<slug>/projects/<project_id>/modules/` | `ModuleViewSet` | GET/POST | 读+写 | ❌ 无 | ❌ 无 | `app/views/module/base.py:71` |
| `/api/workspaces/<slug>/projects/<project_id>/modules/<pk>/` | `ModuleViewSet` | GET/PUT/PATCH/DELETE | 读+写 | ❌ 无 | ❌ 无 | `app/views/module/base.py:71` |
| `/api/workspaces/<slug>/projects/<project_id>/issues/<issue_id>/modules/` | `ModuleIssueViewSet` | POST | 写 | ❌ 无 | ❌ 无 | `app/views/module/issue.py:45` |
| `/api/workspaces/<slug>/projects/<project_id>/modules/<module_id>/issues/` | `ModuleIssueViewSet` | GET/POST | 读+写 | ❌ 无 | ❌ 无 | `app/views/module/issue.py:45` |
| `/api/workspaces/<slug>/projects/<project_id>/modules/<module_id>/issues/<issue_id>/` | `ModuleIssueViewSet` | GET/PUT/PATCH/DELETE | 读+写 | ❌ 无 | ❌ 无 | `app/views/module/issue.py:45` |
| `/api/workspaces/<slug>/projects/<project_id>/modules/<module_id>/module-links/` | `ModuleLinkViewSet` | GET/POST | 读+写 | ❌ 无 | ❌ 无 | `app/views/module/base.py:762` |
| `/api/workspaces/<slug>/projects/<project_id>/modules/<module_id>/module-links/<pk>/` | `ModuleLinkViewSet` | GET/PUT/PATCH/DELETE | 读+写 | ❌ 无 | ❌ 无 | `app/views/module/base.py:762` |
| `/api/workspaces/<slug>/projects/<project_id>/user-favorite-modules/` | `ModuleFavoriteViewSet` | GET/POST | 读+写 | ❌ 无 | ❌ 无 | `app/views/module/base.py:791` |
| `/api/workspaces/<slug>/projects/<project_id>/user-favorite-modules/<module_id>/` | `ModuleFavoriteViewSet` | DELETE | 写 | ❌ 无 | ❌ 无 | `app/views/module/base.py:791` |
| `/api/workspaces/<slug>/projects/<project_id>/modules/<module_id>/user-properties/` | `ModuleUserPropertiesEndpoint` | GET/PATCH | 读+写 | ❌ 无 | ❌ 无 | `app/views/module/base.py:825` |
| `/api/workspaces/<slug>/projects/<project_id>/modules/<module_id>/archive/` | `ModuleArchiveUnarchiveEndpoint` | POST | 写 | ❌ 无 | ❌ 无 | `app/views/module/archive.py:42` |
| `/api/workspaces/<slug>/projects/<project_id>/archived-modules/` | `ModuleArchiveUnarchiveEndpoint` | GET | 读 | ❌ 无 | ❌ 无 | `app/views/module/archive.py:42` |
| `/api/v1/workspaces/<slug>/projects/<project_id>/modules/` | `ModuleListCreateAPIEndpoint` | GET/POST | 读+写 | ❌ 无 | ❌ 无 | `api/views/module.py:76` |
| `/api/v1/workspaces/<slug>/projects/<project_id>/modules/<pk>/` | `ModuleDetailAPIEndpoint` | GET/PATCH/DELETE | 读+写 | ❌ 无 | ❌ 无 | `api/views/module.py:279` |

### 2.3 批量操作端点（3 个，影响统计）

| 端点路径 | 视图类 | HTTP 方法 | 读/写 | 缓存启用 | 失效钩子 | 代码位置 |
|---------|--------|----------|------|---------|---------|---------|
| `/api/workspaces/<slug>/projects/<project_id>/bulk-delete-issues/` | `BulkDeleteIssuesEndpoint` | DELETE | 写 | ❌ 无 | ❌ 无 | `app/views/issue/base.py:761` |
| `/api/workspaces/<slug>/projects/<project_id>/bulk-archive-issues/` | `BulkArchiveIssuesEndpoint` | POST | 写 | ❌ 无 | ❌ 无 | `app/views/issue/archive.py:305` |
| `/api/workspaces/<slug>/projects/<project_id>/bulk-operation-issues/` | `IssueBulkOperationsEndpoint` | POST | 写 | ❌ 无 | ❌ 无 | `app/views/issue/operations.py` |

### 2.4 项目中使用了 `@cache_response` 的端点（5 个，均与 Cycle/Module 无关）

| 端点路径 | 视图类 | 缓存时长 | 代码位置 |
|---------|--------|---------|---------|
| `/api/workspaces/<slug>/project-labels/` | `ProjectLabelListEndpoint` | 2小时 | `app/views/workspace/label.py:21` |
| `/api/workspaces/<slug>/project-estimates/` | `ProjectEstimateListEndpoint` | 2小时 | `app/views/workspace/estimate.py:21` |
| `/api/instances/` | 实例信息 | 2小时 | `license/api/views/instance.py:34` |
| `/api/instances/admin/` | 实例管理 | 2小时 | `license/api/views/admin.py:70` |
| `/api/instances/configuration/` | 实例配置 | 2小时 | `license/api/views/configuration.py:35` |

---

## 3. 后端统计计算机制

### 3.1 实时聚合实现

所有 Cycle/Module 列表和详情查询都通过 Django ORM 的 `annotate` 机制在数据库层面实时计算统计数据。

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

### 3.2 读库路由

所有 Cycle/Module 读端点均设置了 `use_read_replica = True`，在配置了读写分离的环境中会自动路由到读库执行查询。

**代码依据**：
- Cycle：`app/views/cycle/base.py:72`, `api/views/cycle.py:87`
- Module：`app/views/module/base.py:79`, `api/views/module.py:83`

### 3.3 进度快照（Cycle 特有）

Cycle 模型有一个 `progress_snapshot` JSON 字段，仅在 **周期转移** 时写入历史快照，不参与实时计算。

**触发时机**：`POST /transfer-issues/` 端点调用 `transfer_cycle_issues()` 时
- 代码：`apps/api/plane/utils/cycle_transfer_issues.py:408-432`

---

## 4. 前端更新策略详解

### 4.1 单工作项操作更新流程

```
用户操作（状态/负责人/标签变更）
        ↓
1. updateParentStats() → 本地增量更新 (updateDistribution)
        ↓
2. API 调用 (patchIssue / deleteIssue 等)
        ↓
3. fetchParentStats() → 回源刷新当前容器数据
```

**代码依据**：`apps/web/core/store/issue/helpers/base-issues.store.ts:554-587`

### 4.2 批量操作更新流程

| 批量操作 | 本地增量更新 | API 调用 | 回源刷新当前容器 | 代码位置 |
|---------|------------|---------|----------------|---------|
| 批量删除 (`removeBulkIssues`) | ❌ 不支持 | `bulkDeleteIssues` | ✅ `fetchParentStats` | `base-issues.store.ts:677-690` |
| 批量归档 (`bulkArchiveIssues`) | ❌ 不支持 | `bulkArchiveIssues` | ❌ 不回源 | `base-issues.store.ts:698-715` |
| 批量更新属性 (`bulkUpdateProperties`) | ❌ 不支持 | `bulkOperations` | ❌ 不回源 | `base-issues.store.ts:721-753` |

### 4.3 批量操作实现细节

**批量删除**：
```typescript
// base-issues.store.ts:677-690
async removeBulkIssues(workspaceSlug, projectId, issueIds) {
  // 1. 调用批量删除 API
  const response = await this.issueService.bulkDeleteIssues(workspaceSlug, projectId, { issue_ids: issueIds });
  // 2. 回源刷新当前容器统计
  this.fetchParentStats(workspaceSlug, projectId);
  // 3. 本地移除工作项
  runInAction(() => {
    issueIds.forEach((issueId) => {
      this.removeIssueFromList(issueId);
      this.rootIssueStore.issues.removeIssue(issueId);
    });
  });
}
```

**批量归档**：
```typescript
// base-issues.store.ts:698-715
bulkArchiveIssues = async (workspaceSlug, projectId, issueIds) => {
  const response = await this.issueService.bulkArchiveIssues(workspaceSlug, projectId, { issue_ids: issueIds });
  runInAction(() => {
    issueIds.forEach((issueId) => {
      // 本地逐个更新（不触发容器统计更新）
      this.issueUpdate(workspaceSlug, projectId, issueId, { archived_at: response.archived_at }, false);
      this.removeIssueFromList(issueId);
    });
  });
  // ❌ 注意：不调用 fetchParentStats，容器统计可能过时
};
```

**批量更新属性**：
```typescript
// base-issues.store.ts:721-753
bulkUpdateProperties = async (workspaceSlug, projectId, data) => {
  await this.issueService.bulkOperations(workspaceSlug, projectId, data);
  runInAction(() => {
    issueIds.forEach((issueId) => {
      // 本地更新 Issue 状态，但不更新容器统计
      this.rootIssueStore.issues.updateIssue(issueId, updatedProperties);
    });
  });
  // ❌ 注意：既不调用 updateParentStats，也不调用 fetchParentStats
};
```

---

## 5. 当前容器 vs 非当前容器更新路径差异

### 5.1 关键概念

- **当前容器**：用户当前正在浏览的 Cycle 或 Module 视图（如 `/cycles/<cycle-id>`）
- **非当前容器**：用户未在浏览的其他 Cycle 或 Module

### 5.2 单工作项操作更新差异

| 场景 | 当前容器更新路径 | 非当前容器更新路径 | 代码位置 |
|-----|----------------|------------------|---------|
| 更新工作项属性 | 1. `updateParentStats()` 本地增量更新<br>2. API 调用<br>3. `fetchParentStats()` 回源刷新 | ❌ 无任何更新，直到下次访问 | `base-issues.store.ts:554-587` |
| 删除工作项 | 1. `updateParentStats()` 本地增量更新<br>2. API 调用<br>3. `fetchParentStats()` 回源刷新 | ❌ 无任何更新，直到下次访问 | `base-issues.store.ts:596-612` |
| 添加到当前 Cycle | 1. `updateParentStats()` 本地增量更新<br>2. API 调用<br>3. `fetchParentStats()` 回源刷新 | ❌ 无任何更新，直到下次访问 | `base-issues.store.ts:808-838` |
| 从当前 Cycle 移除 | 1. `updateParentStats()` 本地增量更新<br>2. API 调用<br>3. `fetchParentStats()` 回源刷新 | ❌ 无任何更新，直到下次访问 | `base-issues.store.ts:847-866` |
| 添加到当前 Module | 1. `updateParentStats()` 本地增量更新（条件）<br>2. API 调用<br>3. `fetchParentStats()` 回源刷新（条件） | ❌ 无任何更新，直到下次访问 | `base-issues.store.ts:971-1000` |

### 5.3 批量操作更新差异

| 批量操作 | 当前容器 | 非当前容器 | 代码位置 |
|---------|---------|-----------|---------|
| 批量删除 | ✅ `fetchParentStats()` 回源刷新 | ❌ 无更新 | `base-issues.store.ts:677-690` |
| 批量归档 | ❌ 无任何更新（只更新 Issue 列表） | ❌ 无更新 | `base-issues.store.ts:698-715` |
| 批量更新属性 | ❌ 无任何更新（只更新 Issue 对象） | ❌ 无更新 | `base-issues.store.ts:721-753` |

### 5.4 跨容器关联变更更新

当工作项在 Issue 详情页修改了 Cycle 或 Module 关联时：

**修改 Cycle 关联**：
```typescript
// base-issues.store.ts:876-918
addCycleToIssue = async (workspaceSlug, projectId, cycleId, issueId) => {
  // 1. 本地更新 Issue 的 cycle_id
  // 2. 如果当前视图是 Cycle 视图且 cycleId 匹配：
  //    - 调用 updateParentStats() 本地增量更新
  //    - 调用 fetchParentStats() 回源刷新
  // 3. API 调用
  // 4. 如果当前视图是 Cycle 视图且 oldCycleId 匹配：
  //    - 调用 updateParentStats() 本地增量更新
  //    - 调用 fetchParentStats() 回源刷新
};
```

> **重要**：只有当前正在浏览的 Cycle 会更新，目标 Cycle 和原 Cycle 如果不是当前视图，都不会更新。

**修改 Module 关联**：
```typescript
// base-issues.store.ts:1075-1134
changeModulesInIssue = async (workspaceSlug, projectId, issueId, addModuleIds, removeModuleIds) => {
  // 1. 本地更新 Issue 的 module_ids
  // 2. 如果当前视图是 Module 视图且 moduleId 在 add/remove 列表中：
  //    - 调用 updateParentStats() 本地增量更新
  //    - 调用 fetchParentStats() 回源刷新
  // 3. API 调用
};
```

> **重要**：只有当前正在浏览的 Module 会更新，其他涉及的 Module 不会更新。

---

## 6. 数据一致性全景

### 6.1 一致性矩阵

| 操作类型 | 当前容器统计 | 非当前容器统计 | 其他用户会话 |
|---------|------------|--------------|-------------|
| 单工作项更新属性 | ✅ 最终一致（乐观+回源） | ❌ 过时，直到下次访问 | ❌ 过时，直到下次访问 |
| 单工作项删除 | ✅ 最终一致（乐观+回源） | ❌ 过时，直到下次访问 | ❌ 过时，直到下次访问 |
| 单工作项添加到容器 | ✅ 最终一致（乐观+回源） | ❌ 过时，直到下次访问 | ❌ 过时，直到下次访问 |
| 批量删除 | ✅ 最终一致（回源） | ❌ 过时，直到下次访问 | ❌ 过时，直到下次访问 |
| 批量归档 | ⚠️ 不一致（仅更新 Issue 列表） | ❌ 过时，直到下次访问 | ❌ 过时，直到下次访问 |
| 批量更新属性 | ⚠️ 不一致（仅更新 Issue 对象） | ❌ 过时，直到下次访问 | ❌ 过时，直到下次访问 |

### 6.2 潜在不一致场景

1. **批量归档后**：当前 Cycle/Module 的统计值不更新，用户看到的进度条是过时的
2. **批量更新属性后**：当前 Cycle/Module 的统计值不更新，进度显示不准确
3. **跨容器关联变更**：非当前视图的容器统计值不更新
4. **多用户协作**：其他用户的操作不会实时反映到当前用户的统计数据

---

## 7. 关键代码索引

| 功能模块 | 文件路径 | 关键行 |
|---------|---------|-------|
| Cycle 视图集 | `apps/api/plane/app/views/cycle/base.py` | 64-518 |
| Module 视图集 | `apps/api/plane/app/views/module/base.py` | 71-759 |
| 批量删除端点 | `apps/api/plane/app/views/issue/base.py` | 761-785 |
| 批量归档端点 | `apps/api/plane/app/views/issue/archive.py` | 305-338 |
| 前端 Issue 操作主流程 | `apps/web/core/store/issue/helpers/base-issues.store.ts` | 554-1134 |
| Cycle Issue Store | `apps/web/core/store/issue/cycle/issue.store.ts` | 140-175 |
| Module Issue Store | `apps/web/core/store/issue/module/issue.store.ts` | 107-123 |
| 本地分布更新工具 | `packages/utils/src/distribution-update.ts` | 206-268 |
| 缓存装饰器定义 | `apps/api/plane/utils/cache.py` | 25-51 |

---

## 8. 总结

### 8.1 后端架构特点

1. **零缓存策略**：所有 Cycle/Module API 端点均不使用响应缓存，每次请求实时计算
2. **读库路由**：所有读操作支持读写分离，减轻主库压力
3. **无失效钩子**：没有配置 `invalidate_cache` 调用，因为根本没有缓存可失效
4. **快照只读**：`progress_snapshot` 仅用于历史归档，不参与实时计算

### 8.2 前端更新策略

1. **乐观更新优先**：单工作项操作先本地增量更新，再 API 调用，最后回源校验
2. **批量操作缺失**：批量归档和批量更新属性完全不更新容器统计
3. **视图隔离**：只有当前浏览的容器会更新统计，非当前容器依赖下次访问时的 API 请求
4. **无后台刷新**：没有轮询或 WebSocket 机制，其他用户的操作不会实时同步

### 8.3 设计权衡

这种设计在**开发简单性**和**数据准确性**之间做出了权衡：
- ✅ 简单可靠，没有缓存一致性问题
- ✅ 读库路由提升了大规模部署时的性能
- ❌ 批量操作后统计数据可能长时间不一致
- ❌ 非当前视图的统计数据存在延迟
- ❌ 大量并发统计查询可能对数据库造成压力
