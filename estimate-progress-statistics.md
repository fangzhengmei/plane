# Plane Estimate 进度统计：全链路代码分析

## 一、项目级 Estimate 配置

### 1.1 三种 Estimate 分制

前端枚举定义在 `packages/constants/src/estimates.ts#L12-L16`：

```ts
export enum EEstimateSystem {
  POINTS = "points",       // 点数（Fibonacci/Linear/Squares/Custom）
  CATEGORIES = "categories", // 分类（T-shirt/Easy-to-hard/Custom）
  TIME = "time",            // 时间（EE 功能）
}
```

后端模型定义在 `apps/api/plane/db/models/estimate.py#L13-L16`：

```python
class EstimateType(models.TextChoices):
    CATEGORIES = "categories", "Categories"
    POINTS = "points", "Points"
```

> **注意**：后端 `EstimateType` 只声明了 `CATEGORIES` 和 `POINTS` 两种，**没有 `TIME`**。`TIME` 是前端 EE（企业版）扩展的类型，后端创建时若传入 `type="time"` 可存入 CharField（Estimate.type 是 `CharField(255)`，不受 choices 约束），但后端统计逻辑 **不识别 time 类型**。

### 1.2 项目绑定 Estimate

Project 模型通过 ForeignKey 关联 Estimate（`apps/api/plane/db/models/project.py#L109`）：

```python
estimate = models.ForeignKey("db.Estimate", on_delete=models.SET_NULL, related_name="projects", null=True)
```

每个项目只能关联一个 Estimate。项目内可有多个 Estimate 方案，但只有 `last_used=True` 的那个是当前激活的。

前端判断项目是否启用了 Estimate（`apps/web/core/store/estimates/project-estimate.store.ts#L135-L140`）：

```ts
areEstimateEnabledByProjectId = computedFn((projectId: string) => {
  const projectDetails = this.store.projectRoot.project.getProjectById(projectId);
  return Boolean(projectDetails.estimate) || false;
});
```

### 1.3 EstimatePoint 结构

每个 Estimate 下挂多个 EstimatePoint（`apps/api/plane/db/models/estimate.py#L43-L47`）：

| 字段       | 类型           | 说明                                                                     |
| ---------- | -------------- | ------------------------------------------------------------------------ |
| `key`      | IntegerField   | 序号（0 起步，MinValueValidator(0)），用于排序                           |
| `value`    | CharField(255) | 显示值：points 型存 "1"/"2"/"5"/"8"；categories 型存 "XS"/"S"/"M"；time 型存 "1"/"2"/"3"（见 1.4 详述） |
| `estimate` | ForeignKey     | 所属 Estimate                                                            |

前端预设模板（`packages/constants/src/estimates.ts#L29-L141`）：

- **Points**: Fibonacci(1,2,3,5,8,13)、Linear(1-6)、Squares(1,4,9,16,25,36)、Custom
- **Categories**: T-Shirt(XS,S,M,L,XL,XXL)、Easy-to-hard(Easy,Medium,Hard,Very Hard)、Custom
- **Time**: Hours(1-6)，**is_ee: true**

### 1.4 TIME 分制的实际存储单位与展示方式

这是容易混淆的部分，需逐层说明：

**模板定义层**：TIME 的 hours 模板 value 值为 `"1"`, `"2"`, `"3"`, `"4"`, `"5"`, `"6"`（`packages/constants/src/estimates.ts#L126-L137`），这些数字代表的是 **小时数**，不是分钟数。

**模板预览层**：`EstimateCreateStageOne` 组件展示模板时，对 TIME 类型调用 `convertMinutesToHoursMinutesString(Number(template.value))`（`apps/web/core/components/estimates/create/stage-one.tsx#L110-L112`）：

```ts
estimateSystem === (EEstimateSystem.TIME as TEstimateSystemKeys)
  ? convertMinutesToHoursMinutesString(Number(template.value)).trim()
  : template.value
```

`convertMinutesToHoursMinutesString` 的输入参数名为 `totalMinutes`（`packages/utils/src/datetime.ts#L367-L371`），内部用 `Math.floor(mins / 60)` 计算小时。但 hours 模板传入的 value 是 `"1"`, `"2"` 等小数字符串，`Number("1")` = 1，被当作 1 **分钟** 处理：

- value `"1"` → `convertMinutesToHoursMinutesString(1)` → `"1m "`（1 分钟，0 小时）
- value `"2"` → `convertMinutesToHoursMinutesString(2)` → `"2m "`（2 分钟）
- value `"6"` → `convertMinutesToHoursMinutesString(6)` → `"6m "`（6 分钟）

**所以模板预览显示的是 "1m, 2m, 3m, 4m, 5m, 6m"，而非 "1h, 2h, 3h, 4h, 5h, 6h"**。这表明 TIME 分制的 `value` 字段在当前模板中的语义实际上是 **分钟**，而非小时——尽管模板名叫 "Hours"。换言之，如果用户想表示 2 小时，需要在 EstimatePoint 的 value 中存入 `"120"`。

**自定义录入层**：TIME 类型使用 `EstimateTimeInput` 组件（`apps/web/ce/components/estimates/inputs/time-input.tsx`），但在 CE 版中该组件是空壳 `<></>`，EE 版的实现不在本仓库中。创建/编辑时，TIME 和 POINTS 共用同一套数值校验逻辑（`apps/web/core/components/estimates/points/create.tsx#L97-L106`）：必须为正数，`Number(value) > 0`。

**展示层**：所有展示 TIME 类型 estimate 值的地方（dropdown、readonly、preview）统一调用 `convertMinutesToHoursMinutesString(Number(estimatePoint.value))`，将 value 当作分钟数格式化为 `"Xh Ym"` 格式。

**后端汇总层**：后端统计只认 `estimate_point__estimate__type="points"`，TIME 类型的 EstimatePoint 完全不参与汇总（详见第三节）。

**总结**：TIME 分制的 value 存储单位在代码约定上是 **分钟**，`convertMinutesToHoursMinutesString` 函数的参数名也明确为 `totalMinutes`。但默认 hours 模板的值 "1"-"6" 被当作 1-6 分钟处理，实际展示效果为 "1m"-"6m"，与 "Hours" 模板名称的语义有偏差。如果用户想表示 N 小时，应存入 N×60 的值。

---

## 二、Issue 上 Estimate 字段的录入

### 2.1 Issue 模型字段

Issue 有两个相关字段（`apps/api/plane/db/models/issue.py#L128-L134`）：

```python
point = models.IntegerField(validators=[MinValueValidator(0), MaxValueValidator(12)], null=True, blank=True)
estimate_point = models.ForeignKey("db.EstimatePoint", on_delete=models.SET_NULL, related_name="issue_estimates", null=True, blank=True)
```

- `point`：**旧版字段**，整数 0-12，已废弃。`DefaultAnalyticsEndpoint` 仍在使用此字段（见第七节）
- `estimate_point`：**当前使用字段**，FK 指向 EstimatePoint，`SET_NULL` 表示 EstimatePoint 被删除时置空

### 2.2 前端录入组件

Issue 的 estimate 通过 `apps/web/core/components/dropdowns/estimate.tsx#L97-L130` 下拉选择：

- 读取项目当前 active estimate（`last_used=True`）的所有 EstimatePoint
- 对于 TIME 类型，显示值通过 `convertMinutesToHoursMinutesString(Number(estimatePoint.value))` 转换（分钟→"Xh Ym"）
- 对于 POINTS / CATEGORIES 类型，直接显示 `estimatePoint.value`
- 下拉第一项为 "No estimate"（value: null），用户可将 estimate 置空

前端只读展示（`apps/web/core/components/readonly/estimate.tsx#L38-L41`）：

```ts
const displayValue = estimatePoint
  ? currentActiveEstimate?.type === EEstimateSystem.TIME
    ? convertMinutesToHoursMinutesString(Number(estimatePoint.value))
    : estimatePoint.value
  : null;
```

> **关键**：`estimate_point` 存的是 EstimatePoint 的 UUID，不是数值。数值需要通过关联查询 `estimate_point__value` 获取。

---

## 三、分制之间是否换算？——**不换算**

### 3.1 核心结论：三种分制之间不做换算

代码中 **不存在** POINTS → CATEGORIES → TIME 之间的换算逻辑。三种分制是完全独立的体系：

1. **POINTS** 型：`value` 存数值字符串（"1", "5", "8"），可直接 `parseFloat` 参与数学运算
2. **CATEGORIES** 型：`value` 存标签字符串（"XS", "S", "M"），**无法转为数值**，`parseFloat("XS")` = NaN
3. **TIME** 型：`value` 存分钟数字符串（约定上为分钟，如 "60"=1h、"120"=2h），前端展示时 `convertMinutesToHoursMinutesString` 将分钟→"Xh Ym"

### 3.2 TIME 类型的 value 约定为分钟

`convertMinutesToHoursMinutesString`（`packages/utils/src/datetime.ts#L367-L371`）：

```ts
export const convertMinutesToHoursMinutesString = (totalMinutes: number): string => {
  const { hours, minutes } = convertMinutesToHoursAndMinutes(totalMinutes);
  return `${hours ? `${hours}h ` : ``}${minutes ? `${minutes}m ` : ``}`;
};
```

这是 **展示层** 的格式化，不是分制间的换算。详见第一节 1.4 对 hours 模板值与分钟格式化之间关联的分析。

### 3.3 后端统计只认 `type="points"`

这是 **最关键的发现**。后端在所有 estimate 汇总查询中，都硬编码了 `estimate_point__estimate__type="points"` 过滤条件：

- **CycleProgressEndpoint**（`apps/api/plane/app/views/cycle/base.py#L665-L666`）：
  ```python
  Issue.issue_objects.filter(
      estimate_point__estimate__type="points",
      ...
  )
  ```

- **ModuleViewSet.get_queryset**（`apps/api/plane/app/views/module/base.py#L147-L148`）：
  ```python
  Issue.issue_objects.filter(
      estimate_point__estimate__type="points",
      ...
  )
  ```

- **CycleArchiveUnarchiveEndpoint.get_queryset**（`apps/api/plane/app/views/cycle/archive.py#L49-L113`）：每个状态组的子查询都带 `estimate_point__estimate__type="points"`

- **CycleAnalyticsEndpoint**（`apps/api/plane/app/views/cycle/base.py#L832-L837`）：
  ```python
  estimate_type = Project.objects.filter(
      workspace__slug=slug,
      pk=project_id,
      estimate__isnull=False,
      estimate__type="points",
  ).exists()
  ```

- **ModuleViewSet.retrieve**（`apps/api/plane/app/views/module/base.py#L417-L422`）：同上模式

- **burndown_plot**（`apps/api/plane/utils/analytics_plot.py#L127-L132`）：
  ```python
  estimate_type = Project.objects.filter(
      workspace__slug=slug,
      pk=project_id,
      estimate__isnull=False,
      estimate__type="points",
  ).exists()
  ```

**后果**：如果项目使用 CATEGORIES（T-shirt）或 TIME 类型，后端的 `total_estimate_points` 等聚合值 **全部为 0**，因为过滤条件排除了非 points 类型的 EstimatePoint。

### 3.4 前端对 CATEGORIES 类型的特殊处理

`EstimateTypeDropdown`（`apps/web/core/components/cycles/dropdowns/estimate-type-dropdown.tsx#L30-L31`）：

```ts
return (getIsPointsDataAvailable(cycleId) || isCurrentProjectEstimateEnabled) &&
  currentProjectEstimateType !== EEstimateSystem.CATEGORIES ? (
    // 显示 "Work items / Estimates" 切换下拉
  ) : ...
```

当 estimate 类型是 CATEGORIES 时，**不显示** "Work items / Estimates" 切换下拉，因为 T-shirt 值无法做数学汇总。

`CycleSidebarDetails`（`apps/web/core/components/cycles/analytics-sidebar/sidebar-details.tsx#L52-L58`）也只对 POINTS 类型显示 estimate 点数：

```ts
const isEstimatePointValid = isEmpty(cycleDetails?.progress_snapshot || {})
  ? estimateType && estimateType?.type == EEstimateSystem.POINTS ? true : false
  : isEmpty(cycleDetails?.progress_snapshot?.estimate_distribution || {}) ? false : true;
```

### 3.5 前端对 TIME 类型的处理现状

TIME 类型虽然是 EE 功能，但后端统计逻辑的 `estimate_point__estimate__type="points"` 过滤会将其排除。这意味着：

- TIME 类型项目的 `total_estimate_points` = 0
- 前端 `getIsPointsDataAvailable` 返回 false
- 不会展示 points 维度的进度
- `EstimateTypeDropdown` 中 `currentProjectEstimateType !== EEstimateSystem.CATEGORIES` 对 TIME 类型为 true，但因为 `getIsPointsDataAvailable` 返回 false 且 `isCurrentProjectEstimateEnabled` 为 true，下拉框可能显示但切换后数据全为 0

前端在展示层面做了 `convertMinutesToHoursMinutesString` 转换（在 dropdown、readonly、preview 组件中），但仅用于单个 issue 的 estimate 显示，不涉及汇总统计。

---

## 四、未填 Estimate 的兜底策略

### 4.1 前端乐观更新（distribution-update.ts）

`getDistributionDataOfIssue`（`packages/utils/src/distribution-update.ts#L104-L109`）：

```ts
const estimatePoint = parseFloat(estimatePointById?.(issue.estimate_point ?? "")?.value ?? "0");

pathUpdates.push({ path: ["total_issues"], value: multiplier });
pathUpdates.push({ path: ["total_estimate_points"], value: multiplier * estimatePoint });
```

当 `issue.estimate_point` 为 null 时：
- `estimatePointById(undefined)` → undefined
- `undefined?.value` → undefined
- `undefined ?? "0"` → "0"
- `parseFloat("0")` → 0

**结果**：未填 estimate 的 issue 对 `total_estimate_points` 贡献为 **0**，但仍然计入 `total_issues`（+1）。

### 4.2 后端聚合

后端使用 `Cast("estimate_point__value", FloatField())` + `Sum(... default=Value(0))`：
- 当 `estimate_point` 为 null 时，`Cast` 结果为 null
- `Sum` 的 `default=Value(0)` 确保 null 不影响聚合
- 但该 issue 仍计入 `total_issues`

### 4.3 兜底策略总结

| 场景 | total_issues | total_estimate_points |
|------|-------------|----------------------|
| issue 有 points 型 estimate | +1 | +parseFloat(value) |
| issue 有 categories 型 estimate | +1 | +0（被 `type="points"` 过滤掉） |
| issue 有 time 型 estimate | +1 | +0（被 `type="points"` 过滤掉） |
| issue 无 estimate | +1 | +0 |

**影响**：未填 estimate 的 issue 会让 `total_estimate_points` 偏小，导致基于 estimate points 的完成百分比偏高（分母变小）。但基于 issues 的完成百分比不受影响。

---

## 五、Cycle 维度 Estimate 汇总与进度计算

### 5.1 数据来源：两套口径

Cycle 的 estimate 统计有 **两套数据来源**：

#### 5.1.1 实时聚合（CycleProgressEndpoint）

`CycleProgressEndpoint`（`apps/api/plane/app/views/cycle/base.py#L658-L783`）每次请求时从数据库实时聚合：

```python
aggregate_estimates = (
    Issue.issue_objects.filter(
        estimate_point__estimate__type="points",
        issue_cycle__cycle_id=cycle_id,
        issue_cycle__deleted_at__isnull=True,
        workspace__slug=slug,
        project_id=project_id,
    )
    .annotate(value_as_float=Cast("estimate_point__value", FloatField()))
    .aggregate(
        backlog_estimate_point=Sum(Case(When(state__group="backlog", then="value_as_float"), ...)),
        unstarted_estimate_point=Sum(Case(When(state__group="unstarted", then="value_as_float"), ...)),
        started_estimate_point=Sum(Case(When(state__group="started", then="value_as_float"), ...)),
        cancelled_estimate_point=Sum(Case(When(state__group="cancelled", then="value_as_float"), ...)),
        completed_estimate_points=Sum(Case(When(state__group="completed", then="value_as_float"), ...)),
        total_estimate_points=Sum("value_as_float", ...),
    )
)
```

返回字段：`total_estimate_points`, `completed_estimate_points`, `backlog_estimate_points`, `started_estimate_points`, `unstarted_estimate_points`, `cancelled_estimate_points`

**关键过滤条件**：
- `estimate_point__estimate__type="points"` → 仅 points 型
- `issue_cycle__deleted_at__isnull=True` → 排除已删除的 cycle-issue 关联
- `Issue.issue_objects` → 自动排除 triage 状态、已归档、草稿 issue（`apps/api/plane/db/models/issue.py#L92-L101`）

#### 5.1.2 快照（progress_snapshot）

Cycle 模型有 `progress_snapshot` JSONField，当 cycle 结束并转移 issues 到新 cycle 时，会保存当前进度快照。快照中包含 `total_estimate_points`, `completed_estimate_points` 等字段。

前端在展示时优先使用快照数据（`apps/web/core/components/cycles/analytics-sidebar/issue-progress.tsx#L45-L58`）：

```ts
export const validateCycleSnapshot = (cycleDetails: ICycle | null): ICycle | null => {
  if (!isEmpty(cycleDetails.progress_snapshot)) {
    Object.keys(cycleDetails.progress_snapshot || {}).forEach((key) => {
      updatedCycleDetails[currentKey as keyof ICycle] = cycleDetails?.progress_snapshot?.[currentKey];
    });
  }
  return updatedCycleDetails;
};
```

### 5.2 前端 Cycle 进度百分比计算

`calculateCycleProgress`（`packages/utils/src/cycle.ts#L202-L245`）：

```ts
export const calculateCycleProgress = (
  cycle: ICycle | undefined,
  estimateType: "issues" | "points" = "issues",
  includeInProgress: boolean = false
): number => {
  let completed, cancelled, total;

  if (estimateType === "points") {
    completed = cycleDetails.completed_estimate_points || 0;
    cancelled = cycleDetails.cancelled_estimate_points || 0;
    total = cycleDetails.total_estimate_points || 0;
  } else {
    completed = cycleDetails.completed_issues || 0;
    cancelled = cycleDetails.cancelled_issues || 0;
    total = cycleDetails.total_issues || 0;
  }

  const adjustedTotal = total - cancelled;  // 从总分母中扣除 cancelled
  const percentage = (completed / adjustedTotal) * 100;
  return Math.round(percentage);
};
```

**公式**：`完成度 = completed / (total - cancelled) × 100%`

- `cancelled` 从分母中扣除（不管 issues 还是 points 模式）
- `includeInProgress` 参数可将 `started` 也加到 completed 中

### 5.3 Cycle 燃尽图数据

`formatV2Data`（`packages/utils/src/cycle.ts#L139-L178`）根据 `estimateType` 切换数据源：

```ts
const pending = isTypeIssue
  ? p.total_issues - p.completed_issues - p.cancelled_issues
  : p.total_estimate_points - p.completed_estimate_points - p.cancelled_estimate_points;
const completed = isTypeIssue ? p.completed_issues : p.completed_estimate_points;
```

后端 burndown_plot（`apps/api/plane/utils/analytics_plot.py#L123-L264`）在 `plot_type="points"` 时：
- 先检查 `estimate_type = Project.objects.filter(estimate__type="points").exists()`
- 如果 `estimate_type` 为 True 且 `plot_type="points"`，查询所有 `estimate_point__isnull=False` 的 issue
- 将 `estimate_point__value` 转为 float 并按日累加
- 累计待完成 = 总 estimate_points - 截至当日的已完成 estimate_points

> **差异**：详见第十节 10.2 对 burndown_plot 与 CycleProgressEndpoint 过滤条件差异的详细对比。

---

## 六、Module 维度 Estimate 汇总与进度计算

### 6.1 Module 列表查询（实时聚合）

`ModuleViewSet.get_queryset`（`apps/api/plane/app/views/module/base.py#L145-L277`）对每个状态组分别做子查询聚合：

```python
completed_estimate_point = (
    Issue.issue_objects.filter(
        estimate_point__estimate__type="points",
        state__group="completed",
        issue_module__module_id=OuterRef("pk"),
        issue_module__deleted_at__isnull=True,
    )
    .annotate(completed_estimate_points=Sum(Cast("estimate_point__value", FloatField())))
    .values("completed_estimate_points")[:1]
)
```

同样硬编码 `estimate_point__estimate__type="points"`。

### 6.2 Module 详情页（retrieve）

`ModuleViewSet.retrieve`（`apps/api/plane/app/views/module/base.py#L417-L428`）先判断项目是否有 points 型 estimate：

```python
estimate_type = Project.objects.filter(
    workspace__slug=slug, pk=project_id,
    estimate__isnull=False, estimate__type="points",
).exists()
```

只有 `estimate_type=True` 时才查询 `estimate_distribution`（按 assignee/label 分组的 estimate 聚合）和 burndown 图。注意这里的 assignee/label 维度 estimate 分布查询 **没有** `estimate_point__estimate__type="points"` 过滤，直接对所有 issue 做 `Sum(Cast("estimate_point__value", FloatField()))`。但因为外层 `if estimate_type:` 已经保证项目存在 points 型 estimate，所以一般不会混入其他类型。

### 6.3 前端 Module 进度

`ModuleAnalyticsProgress`（`apps/web/core/components/modules/analytics-sidebar/issue-progress.tsx#L70-L78`）：

```ts
const progressHeaderPercentage = moduleDetails
  ? plotType === "points"
    ? completedEstimatePoints != 0 && totalEstimatePoints != 0
      ? Math.round((completedEstimatePoints / totalEstimatePoints) * 100)
      : 0
    : completedIssues != 0 && completedIssues != 0
      ? Math.round((completedIssues / totalIssues) * 100)
      : 0
  : 0;
```

> **注意**：Module 的进度百分比计算 **没有扣除 cancelled**！公式是 `completed / total × 100%`，而 Cycle 用的是 `completed / (total - cancelled) × 100%`。这是 **Cycle 与 Module 进度计算口径不一致** 的地方。

Module 的 "points" 切换下拉只在 `isCurrentEstimateTypeIsPoints`（estimate type 为 POINTS）时显示（`apps/web/core/components/modules/analytics-sidebar/issue-progress.tsx#L135`）。

---

## 七、项目维度 Estimate 汇总

### 7.1 后端汇总接口

项目维度有两个 analytics 接口：

#### 7.1.1 AnalyticsEndpoint（通用图表）

`apps/api/plane/app/views/analytic/base.py#L37-L173`：workspace 级别，支持通过 `project` 筛选参数聚焦到单个项目。

```python
class AnalyticsEndpoint(BaseAPIView):
    def get(self, request, slug):
        x_axis = request.GET.get("x_axis", False)
        y_axis = request.GET.get("y_axis", False)   # "issue_count" | "estimate"
        segment = request.GET.get("segment", False)
        filters = issue_filters(request.GET, "GET")
        queryset = Issue.issue_objects.filter(workspace__slug=slug, **filters)
        distribution = build_graph_plot(queryset=queryset, x_axis=x_axis, y_axis=y_axis, segment=segment)
```

当 `y_axis="estimate"` 时，`build_graph_plot`（`apps/api/plane/utils/analytics_plot.py#L110-L111`）：

```python
queryset = queryset.annotate(estimate=Sum(Cast("estimate_point__value", FloatField())))
```

这里 **没有** `estimate_point__estimate__type="points"` 过滤，也 **没有** `estimate_point__isnull=False` 过滤。所有 issue 都参与聚合，没有 estimate_point 的 issue 的 `estimate_point__value` 为 null，Cast 后也为 null，Sum 时被忽略。如果项目使用 CATEGORIES 类型，`Cast("XS", FloatField())` 结果为 null，同样被 Sum 忽略。

#### 7.1.2 DefaultAnalyticsEndpoint（默认统计面板）

`apps/api/plane/app/views/analytic/base.py#L251-L388`：

```python
open_estimate_sum = open_issues_queryset.aggregate(sum=Sum("point"))["sum"]
total_estimate_sum = base_issues.aggregate(sum=Sum("point"))["sum"]
```

**重要**：这里使用的是 **旧版 `point` 字段**（IntegerField），而不是 `estimate_point`（FK）。这是一个独立的、已废弃的统计口径，与当前 estimate 系统完全无关。

#### 7.1.3 ProjectStatsEndpoint

`apps/api/plane/app/views/analytic/base.py#L391-L455`：只统计 issue 数量和成员数，不涉及 estimate。

### 7.2 前端筛选与图表展示流程

1. **Y 轴指标筛选**：`apps/web/core/components/analytics/select/select-y-axis.tsx` 提供 `issue_count` 和 `estimate` 两个选项。当选择 `estimate` 时，前端会检查当前项目的 estimate 是否启用且类型为 POINTS：

```ts
const isEstimateEnabled = (analyticsOption: string) => {
  if (analyticsOption === "estimate") {
    if (
      projectId &&
      currentActiveEstimateId &&
      areEstimateEnabledByProjectId(projectId.toString()) &&
      estimateById(currentActiveEstimateId)?.type === EEstimateSystem.POINTS
    ) {
      return true;
    } else {
      return false;
    }
  }
  return true;
};
```

只有项目 estimate 类型为 POINTS 时，`estimate` 选项才可见。CATEGORIES 和 TIME 类型下该选项被隐藏。

2. **后端请求**：`AnalyticsEndpoint` 通过 `x_axis`、`y_axis`、`segment` 参数控制图表维度。支持的 x_axis 包括 `estimate_point__value`，可以将 estimate 值作为 X 轴分组。

3. **数据渲染**：后端返回的 `distribution` 是按 x_axis 分组的聚合结果，前端用柱状图或折线图展示。

### 7.3 项目维度 analytics 的过滤条件特点

与 Cycle/Module 的 estimate 汇总不同，项目维度 `AnalyticsEndpoint` 的 `build_graph_plot` 在 `y_axis="estimate"` 时：

| 特征 | AnalyticsEndpoint (build_graph_plot) | CycleProgressEndpoint |
|------|--------------------------------------|----------------------|
| estimate type 过滤 | **无** | `estimate_point__estimate__type="points"` |
| estimate_point 非空过滤 | **无**（null 被 Sum 忽略） | 隐含（FK join 自然排除 null） |
| 包含 CATEGORIES 型 | 是（但 value 无法 Cast 为 float，Sum 结果为 null） | 否 |
| 包含 TIME 型 | 是（value 可 Cast 为 float，Sum 结果包含） | 否 |

这意味着：如果一个项目使用 TIME 类型 estimate，在项目维度 Analytics 中选择 `y_axis=estimate` 可以看到汇总值，但在 Cycle/Module 维度中 estimate points 始终为 0。这是又一层统计口径不一致。

---

## 八、关闭 Issue 与子任务对总分母的影响

### 8.1 Cancelled 状态的影响

| 维度   | 分母计算                                                       |
| ------ | -------------------------------------------------------------- |
| Cycle  | `total - cancelled`（扣除 cancelled）                          |
| Module | `total`（**不扣除** cancelled）                                |

Cycle 的 `calculateCycleProgress`（`packages/utils/src/cycle.ts#L234-L235`）：
```ts
const adjustedTotal = total - cancelled;
```

Module 的进度直接用 `completed / total`，不扣除 cancelled。

### 8.2 子任务（Sub-issues）是否计入

**Cycle**：`total_issues` 的 Count 过滤条件（`apps/api/plane/app/views/cycle/base.py#L115-L125`）中 **没有** `parent__isnull=True` 过滤。所以 **子任务计入** `total_issues` 和 `total_estimate_points`。

**Module**：`total_issues` 的子查询（`apps/api/plane/app/views/module/base.py#L136-L144`）也 **没有** `parent__isnull=True` 过滤。子任务同样计入。

**CycleProgressEndpoint** 的 estimate 聚合（`apps/api/plane/app/views/cycle/base.py#L664-L711`）也 **没有** 排除子任务。

> **结论**：子任务在所有维度的统计中都被完整计入——既有 `total_issues`，也有 `total_estimate_points`。如果子任务和父任务都有 estimate，则 **estimate 会被重复计算**。

### 8.3 IssueManager 默认排除项

`IssueManager`（`apps/api/plane/db/models/issue.py#L92-L101`）：

```python
class IssueManager(SoftDeletionManager):
    def get_queryset(self):
        return (
            super().get_queryset()
            .exclude(state__group=StateGroup.TRIAGE.value)
            .exclude(archived_at__isnull=False)
            .exclude(project__archived_at__isnull=False)
            .exclude(is_draft=True)
        )
```

自动排除：TRIAGE 状态、已归档 issue、已归档项目中的 issue、草稿 issue。这些 **不计入** 任何统计。

---

## 九、前端数据流完整路径

### 9.1 Issue 创建/编辑 → 设置 estimate_point

```
用户选择 EstimatePoint
  → EstimateDropdown 组件（dropdowns/estimate.tsx）
  → 调用 IssueService.updateIssue({ estimate_point: estimatePointId })
  → 后端 Issue.estimate_point (FK → EstimatePoint)
```

### 9.2 Cycle 列表 → 获取统计

```
CycleViewSet.list
  → get_queryset()：annotate total_issues/completed_issues/cancelled_issues
  → 注意：列表接口不返回 estimate_points 字段！
  → 前端 CycleStore.fetchAllCycles → cycleMap
```

### 9.3 Cycle 详情 → 获取 estimate 统计

```
方式1: CycleProgressEndpoint.get
  → 实时聚合 estimate_point__estimate__type="points" 的 issues
  → 返回 total/completed/backlog/started/unstarted/cancelled_estimate_points + issues 数量

方式2: CycleAnalyticsEndpoint.get?type=points
  → 先检查项目 estimate type 是否为 points
  → 若是，返回 assignee/label 维度的 estimate 分布 + burndown completion_chart

前端:
  fetchCycleDetails → cycleMap（含 estimate_distribution）
  fetchActiveCycleProgress → 更新 cycle 的 estimate_points 字段
  fetchActiveCycleAnalytics → 更新 cycle 的 distribution/estimate_distribution
```

### 9.4 前端展示切换

```
用户在 Cycle 详情页切换 "Work items / Estimates"
  → EstimateTypeDropdown → setEstimateType(cycleId, "issues"|"points")
  → CycleAnalyticsProgress 组件根据 estimateType 选择数据源:
    - "issues": cycleDetails.distribution + cycleDetails.xxx_issues
    - "points": cycleDetails.estimate_distribution + cycleDetails.xxx_estimate_points
```

### 9.5 前端乐观更新

当 issue 的 state/estimate_point 变更时，前端通过 `updateCycleDistribution` 乐观更新 cycle 的统计字段，无需等后端返回：

```
Issue 更新
  → getDistributionPathsPostUpdate(prevIssue, nextIssue, stateMap, estimatePointById)
  → 计算 pathUpdates（total_issues, total_estimate_points, xxx_issues, xxx_estimate_points）
  → updateDistribution(cycle, distributionUpdates) → 直接修改 cycle 对象字段
```

---

## 十、潜在问题与统计口径差异汇总

### 10.1 Cycle 与 Module 进度计算口径不一致

| 维度   | 进度公式                     | cancelled 处理 |
| ------ | ---------------------------- | -------------- |
| Cycle  | `completed / (total - cancelled)` | 从分母扣除     |
| Module | `completed / total`          | 不扣除         |

### 10.2 burndown_plot 与实时进度接口过滤条件差异

这是最容易导致"数据对不上"的地方，下面逐项对比：

| 对比项 | CycleProgressEndpoint | ModuleViewSet.get_queryset | CycleAnalyticsEndpoint (assignee/label 分布) | burndown_plot |
|--------|----------------------|---------------------------|----------------------------------------------|---------------|
| estimate type 过滤 | `estimate_point__estimate__type="points"` | `estimate_point__estimate__type="points"` | 无（靠外层 `if estimate_type:` 保护） | **无**（靠 `if estimate_type:` 短路） |
| estimate_point 非空 | 隐含（FK join） | 隐含（FK join） | 无（null 被 Sum 忽略） | **`estimate_point__isnull=False`** |
| 无 estimate 的 issue | 不计入 estimate_points | 不计入 estimate_points | 不计入 estimate（Sum null = null） | **不计入**（显式排除） |

关键差异详解：

**burndown_plot 的两层逻辑**（`apps/api/plane/utils/analytics_plot.py#L123-L156`）：

1. 先判断项目 estimate type 是否为 points：`estimate_type = Project.objects.filter(estimate__type="points").exists()`
2. 如果为 True 且 `plot_type="points"`，查询 `estimate_point__isnull=False` 的 issue 并累加 value

**CycleProgressEndpoint** 的逻辑：直接 `filter(estimate_point__estimate__type="points")`，这隐含了 `estimate_point__isnull=False`（因为 FK join 只能匹配到有 estimate_point 的 issue），同时排除了非 points 类型的 estimate_point。

**实际影响**：在正常使用场景下（项目只有一种 estimate type），两者的结果是一致的。但如果出现以下异常情况，数据会产生差异：
- 项目存在多个 Estimate（一个 points、一个 time），且某些 issue 关联了 time 型 estimate_point → burndown_plot 会包含这些 issue 的 value，但 CycleProgressEndpoint 不会
- CycleArchiveUnarchiveEndpoint（`apps/api/plane/app/views/cycle/archive.py#L49-L113`）中的 estimate 分布查询也使用了 `estimate_point__estimate__type="points"`，与 burndown_plot 的宽松过滤不一致

### 10.3 CATEGORIES/TIME 类型项目的 estimate 统计为 0

后端所有 estimate 聚合都硬编码 `estimate_point__estimate__type="points"`，导致非 points 项目的 `total_estimate_points = 0`。前端虽然对 CATEGORIES 做了 UI 隐藏（不显示切换下拉），但 TIME 类型的用户体验可能有落差（录入时有 estimate，汇总时看不到）。

### 10.4 项目维度 Analytics 与 Cycle/Module 维度口径不一致

`build_graph_plot`（项目维度）在 `y_axis="estimate"` 时 **不做** `estimate_point__estimate__type="points"` 过滤，而 Cycle/Module 维度都做。这意味着：
- TIME 类型项目在 Analytics 中可以看到 estimate 汇总值
- 但在 Cycle/Module 中 estimate points 始终为 0
- 项目维度的 `DefaultAnalyticsEndpoint` 甚至使用的是 **旧版 `point` 字段**，与整个 estimate_point 体系无关

### 10.5 子任务 estimate 双重计算

父子 issue 若都设置了 estimate_point，两者的 points 都会被累加到 `total_estimate_points`。这可能导致 estimate 总量虚高。

### 10.6 未填 estimate 的 issue 拉高完成度

未填 estimate 的 issue 计入 `total_issues` 但 `total_estimate_points` 为 0。基于 points 的完成百分比 = `completed_estimate_points / total_estimate_points`，如果部分 issue 无 estimate，分母偏小，完成度偏高。

### 10.7 后端 EstimateType 缺少 TIME

后端 `EstimateType.TextChoices` 只有 CATEGORIES 和 POINTS，TIME 类型虽然可通过 CharField 存入数据库，但不在 choices 中。如果前端 EE 功能传了 `type="time"`，后端创建可成功，但统计时被 `type="points"` 过滤排除。

### 10.8 TIME hours 模板值与分钟格式化的语义偏差

hours 模板的 value 为 "1"-"6"，被 `convertMinutesToHoursMinutesString` 当作 1-6 分钟处理，显示为 "1m"-"6m"，而非 "1h"-"6h"。这与模板名称 "Hours" 的语义不符。如果用户存入 "60" 来表示 1 小时，展示为 "1h" 是正确的，但默认模板并不提供这样的值。
