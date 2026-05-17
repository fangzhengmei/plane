# 周期（Cycle）与功能模块（Module）进度计算数据流

## 1. 概述

本报告描述 Plane 系统中周期（Cycle）与功能模块（Module）两类容器的进度数值如何从工作项（Issue）原始数据汇总计算，包括统计口径、缓存刷新机制以及跨实体的派生计算逻辑。

## 2. 统计口径

### 2.1 基础数据维度

两类容器的进度计算均基于工作项的状态分组（state_group），系统定义了 5 种核心状态分组：

| 状态分组 | 说明 |
|---------|------|
| `backlog` | 待办 |
| `unstarted` | 未开始 |
| `started` | 进行中 |
| `completed` | 已完成 |
| `cancelled` | 已取消 |

### 2.2 数据过滤规则

所有统计聚合均应用统一的过滤条件：
- 排除已归档的工作项（`archived_at__isnull=True`）
- 排除草稿工作项（`is_draft=False`）
- 排除已删除的关联关系（`deleted_at__isnull=True`）

### 2.3 进度计算公式

#### Cycle 进度计算
位置：`packages/utils/src/cycle.ts:202-245`

```
进度 = completed / (total - cancelled) * 100
```

- **分子**：`completed_issues` 或 `completed_estimate_points`（可选是否包含进行中）
- **分母**：`total_issues - cancelled_issues`（排除已取消的工作项）
- **可选参数**：
  - `estimateType`：`"issues"`（按工作项数量）或 `"points"`（按故事点）
  - `includeInProgress`：是否将 `started` 状态计入完成度

#### Module 进度计算（排序场景）
位置：`packages/utils/src/module.ts:37-40`

```
进度 = (completed_issues + cancelled_issues) / total_issues
```

- 该公式主要用于模块列表的排序场景

### 2.4 聚合字段

后端通过 Django ORM 的 `annotate` 机制在数据库层面完成聚合：

| 字段 | 说明 |
|------|------|
| `total_issues` | 总工作项数 |
| `completed_issues` | 已完成工作项数 |
| `cancelled_issues` | 已取消工作项数 |
| `started_issues` | 进行中工作项数 |
| `unstarted_issues` | 未开始工作项数 |
| `backlog_issues` | 待办工作项数 |
| `total_estimate_points` | 总故事点 |
| `completed_estimate_points` | 已完成故事点 |
| ... | ... |

Cycle 位置：`apps/api/plane/api/views/cycle.py:100-164`
Module 位置：`apps/api/plane/api/views/module.py:99-169`

## 3. 数据流向

### 3.1 数据流总览

```
工作项原始数据(Issue)
        ↓
[数据库层] Django ORM annotate 聚合
        ↓
[API层] 序列化返回结构化数据
        ↓
[前端状态层] MobX Store 缓存
        ↓
[前端计算层] 本地增量更新 + 进度计算
        ↓
[UI展示层] 进度条、燃尽图、统计卡片
```

### 3.2 后端聚合流程

#### Cycle 聚合查询
位置：`apps/api/plane/api/views/cycle.py:89-167`

```python
Cycle.objects.filter(...)
    .annotate(total_issues=Count("issue_cycle", filter=Q(...)))
    .annotate(completed_issues=Count("issue_cycle__issue__state__group", filter=Q(state_group="completed")))
    .annotate(cancelled_issues=Count(...))
    .annotate(started_issues=Count(...))
    ...
```

每个 `annotate` 操作通过关联 `CycleIssue` 中间表，过滤不同 `state_group` 的工作项进行计数。

#### Module 聚合查询
位置：`apps/api/plane/api/views/module.py:85-171`

与 Cycle 类似，但关联 `ModuleIssue` 中间表。

### 3.3 前端状态管理

#### Cycle Store
位置：`apps/web/core/store/cycle.store.ts`

- 核心存储：`cycleMap: Record<string, ICycle>`
- 加载状态：`loader`、`progressLoader`
- 拉取方法：
  - `fetchAllCycles()`：拉取项目全量周期（含统计数据）
  - `fetchActiveCycleProgress()`：拉取活跃周期进度
  - `fetchCycleDetails()`：拉取单个周期详情

#### Module Store
位置：`apps/web/core/store/module.store.ts`

- 核心存储：`moduleMap: Record<string, IModule>`
- 拉取方法：
  - `fetchModules()`：拉取项目全量模块
  - `fetchModuleDetails()`：拉取单个模块详情

## 4. 缓存机制

### 4.1 后端进度快照（Cycle 特有）

字段：`Cycle.progress_snapshot`（JSONField）
位置：`apps/api/plane/db/models/cycle.py:74`

**触发时机**：周期转移（transfer_cycle_issues）时
位置：`apps/api/plane/utils/cycle_transfer_issues.py:408-432`

快照内容包括：
- 各状态分组的工作项计数
- 分配统计（assignee distribution）
- 标签统计（label distribution）
- 燃尽图数据（completion chart）
- 故事点统计（如启用）

### 4.2 后端响应缓存

装饰器：`@cache_response`
位置：`apps/api/plane/utils/cache.py:25-51`

- 默认超时：1小时
- 缓存键：`{path}:{user_id}`
- 非 DEBUG 模式下生效

### 4.3 前端本地缓存

MobX Store 维护的内存缓存：
- `cycleMap` / `moduleMap`：聚合后的容器数据
- `fetchedMap`：标记项目数据是否已拉取

### 4.4 本地增量更新

函数：`updateDistribution()`
位置：`packages/utils/src/distribution-update.ts:206-268`

当工作项状态变更时，无需重新拉取全量数据，直接在本地更新统计：

```typescript
updateDistribution(cycleOrModule, {
  pathUpdates: [],      // 直接路径更新（如 total_issues、completed_issues）
  assigneeUpdates: [],  // 分配统计更新
  labelUpdates: [],     // 标签统计更新
});
```

更新路径示例：
- `["total_issues"]`：总工作项数 +1/-1
- `["completed_issues"]`：已完成数 +1/-1
- `["distribution", "completion_chart", "2024-01-15"]`：燃尽图特定日期数据

## 5. 跨实体派生计算

### 5.1 双维度进度

支持两种估算维度切换：
- **按工作项数量**：使用 `*_issues` 系列字段
- **按故事点**：使用 `*_estimate_points` 系列字段

切换逻辑：`CycleStore.getEstimateTypeByCycleId()`
位置：`apps/web/core/store/cycle.store.ts:368-374`

### 5.2 燃尽图数据格式化

#### V1 格式（旧版）
位置：`packages/utils/src/cycle.ts:103-129`
- 数据来源：`cycle.distribution.completion_chart`
- 每日数据点：记录当天未完成的工作项数量

#### V2 格式（新版）
位置：`packages/utils/src/cycle.ts:139-178`
- 数据来源：`cycle.progress` 数组
- 包含每日各状态分组的完整快照

### 5.3 分配/标签统计

两类派生统计维度：
- **按负责人**：`distribution.assignees` / `estimate_distribution.assignees`
- **按标签**：`distribution.labels` / `estimate_distribution.labels`

每个维度包含：
- `total_issues` / `total_estimates`：总数
- `completed_issues` / `completed_estimates`：已完成数
- `pending_issues` / `pending_estimates`：待完成数

位置：`apps/api/plane/utils/cycle_transfer_issues.py:286-396`

### 5.4 理想进度线

函数：`ideal()`
位置：`packages/utils/src/cycle.ts:88-93`

```
理想进度 = (已过天数 / 总天数) * 总范围
```

用于燃尽图中与实际进度对比。

## 6. 关键实体关系

```
Project
   ├─→ Cycle (1:N)
   │     └─→ CycleIssue (1:N) ──→ Issue (N:1)
   │           └─ 关联属性: deleted_at
   │
   └─→ Module (1:N)
         └─→ ModuleIssue (1:N) ──→ Issue (N:1)
               └─ 关联属性: deleted_at
```

- 一个工作项可以同时属于多个 Cycle 和 Module
- 关联删除采用软删除（`deleted_at`），不影响历史统计

## 7. 核心代码索引

| 功能 | 文件路径 |
|------|---------|
| Cycle 进度计算 | `packages/utils/src/cycle.ts:202-245` |
| Module 排序进度 | `packages/utils/src/module.ts:37-40` |
| Cycle 后端聚合 | `apps/api/plane/api/views/cycle.py:89-167` |
| Module 后端聚合 | `apps/api/plane/api/views/module.py:85-171` |
| 周期转移与快照 | `apps/api/plane/utils/cycle_transfer_issues.py` |
| 本地分布更新 | `packages/utils/src/distribution-update.ts` |
| Cycle 前端 Store | `apps/web/core/store/cycle.store.ts` |
| Module 前端 Store | `apps/web/core/store/module.store.ts` |
| 缓存装饰器 | `apps/api/plane/utils/cache.py` |
