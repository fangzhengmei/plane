# 搜索过滤条件解析、组合与查询拼装逻辑

## 一、整体架构概览

搜索过滤系统采用**分层设计**，从前端用户交互到后端数据库查询，经过多层转换和组合，最终形成多维度约束叠加的完整查询：

```
┌─────────────────────────────────────────────────────────────────┐
│                        前端请求发起链路                           │
├─────────────────────────────────────────────────────────────────┤
│  用户交互（UI选择过滤条件）                                       │
│      ↓                                                          │
│  FilterInstance 状态管理（表达式树构建）                           │
│      ↓                                                          │
│  Adapter.toExternal() → TWorkItemFilterExpression                │
│      ↓                                                          │
│  computedFilteredParams() → JSON.stringify(richFilters)          │
│      ↓                                                          │
│  IssueService.getIssues() → URL参数: ?filters={...}&...         │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│                        后端查询拼装链路                           │
├─────────────────────────────────────────────────────────────────┤
│  接收请求: request.query_params.get("filters")                   │
│      ↓                                                          │
│  ComplexFilterBackend → 解析JSON → 构建Q对象(rich filters)        │
│      ↓                                                          │
│  issue_filters() → 解析旧版扁平参数 → 构建过滤字典                │
│      ↓                                                          │
│  权限约束检查: 访客用户/项目成员角色过滤                          │
│      ↓                                                          │
│  多维度叠加: QuerySet.filter(rich_q).filter(**legacy_filters)    │
│                    .filter(permission_q)                         │
│      ↓                                                          │
│  数据库查询执行                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 二、核心数据结构

### 2.1 内部表达式树（前端）

定义位置：`packages/types/src/rich-filters/expression.ts`

```typescript
// 节点类型
FILTER_NODE_TYPE = {
  CONDITION: "condition",  // 叶子节点：单个过滤条件
  GROUP: "group",          // 组合节点：多个条件的逻辑组合
}

// 单个条件节点
type TFilterConditionNode = {
  id: string;
  type: "condition";
  property: string;        // 字段名，如 "state_id", "priority"
  operator: string;        // 操作符，如 "exact", "in", "gte"
  value: SingleOrArray<T>; // 过滤值，支持单值或数组
}

// 组合节点（当前只支持AND）
type TFilterAndGroupNode = {
  id: string;
  type: "group";
  logicalOperator: "and";
  children: TFilterExpression[];  // 子节点，可以是条件或嵌套组
}
```

**设计意图**：
- 使用树状结构支持任意深度的逻辑组合
- 每个节点有唯一ID，便于精确查找和更新
- 组合节点当前仅支持AND，为未来扩展OR/NOT预留了接口

---

## 三、前端过滤条件管理

### 3.1 FilterInstance 状态管理

定义位置：`packages/shared-state/src/store/rich-filters/filter.ts`

`FilterInstance` 是前端过滤系统的核心类，使用 MobX 进行响应式状态管理：

```typescript
class FilterInstance {
  // 核心状态
  expression: TFilterExpression | null;  // 当前过滤表达式树
  initialFilterExpression: TFilterExpression | null;  // 初始状态（用于比较变更）
  
  // 计算属性
  get hasActiveFilters(): boolean;       // 是否有激活的过滤器
  get allConditions(): TFilterConditionNode[];  // 提取所有条件节点
  
  // 操作方法
  addCondition(groupOperator, condition, isNegation);  // 添加条件
  updateConditionValue(conditionId, value);            // 更新条件值
  removeCondition(conditionId);                        // 删除条件
  clearFilters();                                      // 清空所有过滤
}
```

### 3.2 表达式树遍历

定义位置：`packages/utils/src/rich-filters/operations/traversal/core.ts`

`traverseExpressionTree` 是核心遍历工具，支持三种模式：

```typescript
enum TreeTraversalMode {
  ALL = "ALL",           // 访问所有节点
  CONDITIONS = "CONDITIONS",  // 只访问条件节点
  GROUPS = "GROUPS",     // 只访问组节点
}

// 典型用法：提取所有条件
const allConditions = traverseExpressionTree(
  expression,
  (node) => isConditionNode(node) ? node : null,
  TreeTraversalMode.CONDITIONS
);
```

**关键特性**：
- 深度优先遍历
- 自动处理递归嵌套
- 支持按节点类型过滤

---

## 四、格式转换（Adapter 层）

### 4.1 Adapter 接口

定义位置：`packages/types/src/rich-filters/adapter.ts`

```typescript
interface IFilterAdapter<P, E> {
  toInternal(externalFilter: E): TFilterExpression<P> | null;  // 外部→内部
  toExternal(internalFilter: TFilterExpression<P> | null): E;  // 内部→外部
}
```

### 4.2 工作项过滤器 Adapter 实现

定义位置：`packages/shared-state/src/store/work-item-filters/adapter.ts`

#### 4.2.1 外部格式 → 内部表达式树（toInternal）

输入格式（后端返回/URL参数）：
```javascript
// 单个条件
{ "state_id__exact": "uuid-123" }

// 多值条件（逗号分隔）
{ "assignee_id__in": "user1,user2,user3" }

// AND组合
{
  "and": [
    { "state_id__exact": "uuid-123" },
    { "priority__in": "high,urgent" }
  ]
}
```

解析逻辑：
1. 检查是否包含逻辑运算符键（`and`）
2. 解析键名：`property__operator` → 拆分属性和操作符
3. 多值操作符（`in`）自动拆分逗号分隔的字符串
4. 递归构建表达式树

#### 4.2.2 内部表达式树 → 外部格式（toExternal）

输出格式（发送给后端）：
```typescript
// 条件节点转换
// property: "state_id", operator: "exact", value: "uuid-123"
// → { "state_id__exact": "uuid-123" }

// 数组值自动用逗号连接
// property: "assignee_id", operator: "in", value: ["user1", "user2"]
// → { "assignee_id__in": "user1,user2" }

// AND组合转换
// group(and) with children [cond1, cond2]
// → { "and": [cond1_external, cond2_external] }
```

**关键转换代码**（`adapter.ts:246-259`）：
```typescript
private _createWorkItemFilterConditionData = (property, operator, value) => {
  const conditionKey = `${property}__${operator}`;
  const stringValue = Array.isArray(value) ? value.join(",") : value;
  return { [conditionKey]: stringValue };
};
```

---

## 五、前端请求序列化与查询触发

### 5.1 过滤器参数构建

定义位置：`apps/web/core/store/issue/helpers/issue-filter-helper.store.ts`

当用户选择过滤条件后，前端需要将内部表达式树转换为URL查询参数。核心函数是 `computedFilteredParams`：

```typescript
computedFilteredParams = (
  richFilters: TWorkItemFilterExpression,    // Adapter转换后的外部格式
  displayFilters: IIssueDisplayFilterOptions | undefined,
  acceptableParamsByLayout: TIssueParams[]
): Partial<Record<TIssueParams, string | boolean>> => {
  // 1. 构建显示过滤参数（分组、排序等）
  const computedDisplayFilters = {
    group_by: displayFilters?.group_by ? EIssueGroupByToServerOptions[displayFilters.group_by] : undefined,
    order_by: displayFilters?.order_by || undefined,
    sub_issue: displayFilters?.sub_issue ?? true,
  };

  const issueFiltersParams: Partial<Record<TIssueParams, boolean | string>> = {};

  // 2. 转换为字符串参数，数组用逗号连接
  Object.keys(computedDisplayFilters).forEach((key) => {
    const _key = key as TIssueParams;
    const _value = computedDisplayFilters[_key];
    if (_value != undefined && acceptableParamsByLayout.includes(_key))
      issueFiltersParams[_key] = Array.isArray(_value)
        ? _value.join(",")
        : _value;
  });

  // 3. 关键：富过滤器JSON序列化
  if (richFilters) 
    issueFiltersParams.filters = JSON.stringify(richFilters);

  // 4. 附加其他参数
  if (displayFilters?.layout) 
    issueFiltersParams.layout = displayFilters?.layout;

  return issueFiltersParams;
};
```

**序列化关键点**：
- 富过滤器使用 `JSON.stringify()` 序列化为JSON字符串
- 多值参数使用逗号分隔的字符串（如 `priority=high,urgent`）
- 布尔值和简单值直接转换为字符串

### 5.2 查询触发流程

定义位置：`apps/web/core/store/issue/project/issue.store.ts`

```typescript
fetchIssues = async (
  workspaceSlug: string,
  projectId: string,
  loadType: TLoader = "init-loader",
  options: IssuePaginationOptions
) => {
  try {
    // 设置加载状态
    runInAction(() => {
      this.setLoader(loadType);
      this.clear(!isExistingPaginationOptions);
    });

    // 1. 从FilterStore获取构建好的查询参数
    const params = this.issueFilterStore?.getFilterParams(
      options, 
      projectId, 
      undefined, 
      undefined, 
      undefined
    );

    // 2. 调用IssueService发送请求
    const response = await this.issueService.getIssues(
      workspaceSlug, 
      projectId, 
      params, 
      { signal: this.controller.signal }
    );

    // 3. 处理响应
    this.onfetchIssues(response, options, workspaceSlug, projectId);
    return response;
  } catch (error) {
    this.setLoader(undefined);
    throw error;
  }
};
```

### 5.3 最终URL示例

假设用户选择了：
- 状态：待办（state_id = backlog-uuid）
- 优先级：高或紧急（priority in [high, urgent]）
- 分组方式：按状态分组
- 每页数量：25

生成的URL查询参数：
```
?filters={"and":[{"state_id__exact":"backlog-uuid"},{"priority__in":"high,urgent"}]}
&group_by=state
&order_by=-created_at
&per_page=25
&cursor=25:0:0
```

---

## 六、后端多维度过滤叠加

### 6.1 视图层过滤管线

定义位置：`apps/api/plane/app/views/issue/base.py`

后端查询构建采用**分层过滤管线**设计，多种过滤维度依次叠加：

```python
class IssueViewSet(BaseViewSet):
    filter_backends = (ComplexFilterBackend,)  # 富过滤器后端
    filterset_class = IssueFilterSet           # 字段白名单与Q对象构建

    def list(self, request, slug, project_id):
        # ========== 基础查询 ==========
        issue_queryset = self.get_queryset()  # 基础: workspace+project过滤

        # ========== 维度1: 富过滤器 (ComplexFilterBackend) ==========
        # 解析 ?filters=... 参数，构建Q对象
        issue_queryset = self.filter_queryset(issue_queryset)

        # ========== 维度2: 旧版扁平过滤器 ==========
        # 解析 ?state=...&priority=... 等参数
        filters = issue_filters(query_params, "GET")
        issue_queryset = issue_queryset.filter(**filters)

        # ========== 维度3: 权限约束 ==========
        # 访客用户只能查看自己创建的issue
        project = Project.objects.get(pk=project_id, workspace__slug=slug)
        if (
            ProjectMember.objects.filter(
                workspace__slug=slug,
                project_id=project_id,
                member=request.user,
                role=5,  # GUEST角色
                is_active=True,
            ).exists()
            and not project.guest_view_all_features
        ):
            issue_queryset = issue_queryset.filter(created_by=request.user)

        # ========== 维度4: 其他特殊过滤 ==========
        # 分组、分页、排序等后续处理
        # ...
```

### 6.2 过滤器后端执行机制

定义位置：`apps/api/plane/api/views/base.py`

```python
class BaseAPIView(GenericAPIView):
    def filter_queryset(self, queryset):
        """遍历所有filter_backends，依次应用过滤"""
        for backend in list(self.filter_backends):
            queryset = backend().filter_queryset(self.request, queryset, self)
        return queryset
```

这是标准的 DRF FilterBackend 机制，支持多个后端串联执行。当前配置：
```python
filter_backends = (ComplexFilterBackend,)  # 仅使用富过滤器
```

### 6.3 旧版过滤器（issue_filters）

定义位置：`apps/api/plane/utils/issue_filters.py`

这是系统演进过程中保留的扁平参数过滤机制，与富过滤器并行工作：

```python
def issue_filters(query_params, method, prefix=""):
    issue_filter = {}

    # 支持的过滤字段映射表
    ISSUE_FILTER = {
        "state": filter_state,           # 状态过滤
        "state_group": filter_state_group,  # 状态组过滤
        "priority": filter_priority,     # 优先级过滤
        "assignees": filter_assignees,   # 处理人过滤
        "labels": filter_labels,         # 标签过滤
        "created_by": filter_created_by, # 创建人过滤
        "cycle": filter_cycle,           # 迭代过滤
        "module": filter_module,         # 模块过滤
        "start_date": filter_start_date, # 开始日期
        "target_date": filter_target_date, # 截止日期
        "sub_issue": filter_sub_issue_toggle, # 子issue开关
        # ... 共20+种过滤字段
    }

    # 遍历查询参数，匹配到的字段调用对应的过滤函数
    for key, value in ISSUE_FILTER.items():
        if key in query_params:
            func = value
            func(query_params, issue_filter, method, prefix)
    
    return issue_filter
```

**典型过滤函数实现**：
```python
def filter_assignees(params, issue_filter, method, prefix=""):
    if method == "GET":
        # GET请求：逗号分隔字符串 → 数组
        assignees = [item for item in params.get("assignees").split(",") if item != "null"]
        if "None" in assignees:
            issue_filter[f"{prefix}assignees__isnull"] = True
        assignees = filter_valid_uuids(assignees)
        if len(assignees):
            issue_filter[f"{prefix}assignees__in"] = assignees
    
    # 自动添加软删除排除条件
    issue_filter[f"{prefix}issue_assignee__deleted_at__isnull"] = True
    return issue_filter
```

### 6.4 权限约束机制

权限过滤是在业务逻辑层直接添加的硬约束，确保数据安全：

```python
# 在 IssueViewSet.list 中
if (
    # 用户是访客角色
    ProjectMember.objects.filter(
        workspace__slug=slug,
        project_id=project_id,
        member=request.user,
        role=5,  # GUEST
        is_active=True,
    ).exists()
    # 项目未开启"访客查看所有内容"
    and not project.guest_view_all_features
):
    # 强制过滤：只能查看自己创建的issue
    issue_queryset = issue_queryset.filter(created_by=request.user)
```

在 `IssueDetailEndpoint` 中权限检查更复杂，使用子查询实现：
```python
permission_subquery = (
    Issue.issue_objects.filter(workspace__slug=slug, project_id=project_id, id=OuterRef("id"))
    .filter(
        # 管理员/成员：查看所有
        Q(
            project__project_projectmember__member=self.request.user,
            project__project_projectmember__is_active=True,
            project__project_projectmember__role__gt=ROLE.GUEST.value,
        )
        # 访客 + 开启查看所有：查看所有
        | Q(
            project__project_projectmember__member=self.request.user,
            project__project_projectmember__is_active=True,
            project__project_projectmember__role=ROLE.GUEST.value,
            project__guest_view_all_features=True,
        )
        # 访客 + 未开启查看所有：仅查看自己创建
        | Q(
            project__project_projectmember__member=self.request.user,
            project__project_projectmember__is_active=True,
            project__project_projectmember__role=ROLE.GUEST.value,
            project__guest_view_all_features=False,
            created_by=self.request.user,
        )
    )
    .values("id")
)

# 使用 EXISTS 子查询进行权限过滤
issue = Issue.issue_objects.filter(Exists(permission_subquery))
```

### 6.5 完整过滤叠加顺序

最终的查询构建遵循严格的叠加顺序，确保安全性和正确性：

| 顺序 | 过滤维度 | 实现方式 | 说明 |
|------|---------|---------|------|
| 1 | 基础范围 | `get_queryset()` | 限定工作区、项目、删除状态 |
| 2 | 富过滤器 | `ComplexFilterBackend` | 解析 `?filters=` JSON参数 |
| 3 | 旧版过滤器 | `issue_filters()` | 解析扁平查询参数 |
| 4 | 权限约束 | 业务逻辑层 | 访客/成员角色过滤 |
| 5 | 分组/排序 | 后续处理 | `order_by`, `group_by` 等 |

**最终生成的SQL伪代码**：
```sql
SELECT * FROM issue
WHERE 
  -- 基础范围
  workspace_id = 'xxx' 
  AND project_id = 'yyy'
  AND deleted_at IS NULL
  
  -- 富过滤器 (AND组合)
  AND (state_id = 'backlog-uuid' AND priority IN ('high', 'urgent'))
  
  -- 旧版过滤器 (如果有)
  AND created_by_id IN ('user1', 'user2')
  
  -- 权限约束 (访客用户)
  AND created_by_id = 'current-user-id'
  
ORDER BY created_at DESC
LIMIT 25 OFFSET 0;
```

---

## 七、后端查询拼装（ComplexFilterBackend）

### 7.1 ComplexFilterBackend

定义位置：`apps/api/plane/utils/filters/filter_backend.py`

这是后端过滤的核心，负责：
1. 解析前端发送的 JSON 过滤条件
2. 验证结构和字段合法性
3. 递归构建 Django Q 对象
4. 应用到 QuerySet

#### 7.1.1 处理流程

```python
def filter_queryset(self, request, queryset, view):
    # 1. 读取并解析 filters 参数
    filter_string = request.query_params.get("filters", None)
    filter_data = json.loads(filter_string)
    
    # 2. 验证结构（深度、语法）
    self._validate_structure(filter_data, max_depth=5)
    
    # 3. 验证字段白名单
    self._validate_fields(filter_data, view)
    
    # 4. 递归构建Q对象
    combined_q = self._evaluate_node(filter_data, view, queryset)
    
    # 5. 应用查询
    return queryset.filter(combined_q)
```

#### 7.1.2 节点求值（_evaluate_node）

这是核心的递归组合逻辑：

```python
def _evaluate_node(self, node, view, queryset):
    if "or" in node:
        # OR组合：Q() | Q() | ...
        combined_q = Q()
        for child in node["or"]:
            child_q = self._evaluate_node(child, view, queryset)
            combined_q |= child_q
        return combined_q
    
    if "and" in node:
        # AND组合：Q() & Q() & ...
        combined_q = Q()
        for child in node["and"]:
            child_q = self._evaluate_node(child, view, queryset)
            combined_q &= child_q
        return combined_q
    
    if "not" in node:
        # NOT组合：~Q()
        child_q = self._evaluate_node(node["not"], view, queryset)
        return ~child_q
    
    # 叶子节点：通过FilterSet构建Q对象
    return self._build_leaf_q(node, view, queryset)
```

**设计亮点**：
- 支持任意深度的逻辑嵌套（默认最大5层）
- 使用 Django Q 对象进行惰性求值，最后一次性应用
- 逻辑运算符与字段条件完全分离

### 7.2 FilterSet 与 Q 对象构建

定义位置：`apps/api/plane/utils/filters/filterset.py`

#### 7.2.1 BaseFilterSet.build_combined_q

```python
def build_combined_q(self):
    combined_q = Q()
    
    for name, value in self.form.cleaned_data.items():
        f = self.filters[name]
        
        if f.method is not None:
            # 自定义过滤方法（处理软删除等特殊逻辑）
            res = f.filter(self.queryset, value)
            q_piece = res if isinstance(res, Q) else Q(pk__in=res.values("pk"))
        else:
            # 标准字段过滤
            lookup = f"{f.field_name}__{f.lookup_expr}"
            q_piece = Q(**{lookup: value})
        
        # 组合Q对象
        if getattr(f, "exclude", False):
            combined_q &= ~q_piece
        else:
            combined_q &= q_piece
    
    return combined_q
```

#### 7.2.2 IssueFilterSet 示例

```python
class IssueFilterSet(BaseFilterSet):
    # 自定义过滤方法（处理关联表软删除）
    assignee_id = filters.UUIDFilter(method="filter_assignee_id")
    assignee_id__in = UUIDInFilter(method="filter_assignee_id_in")
    
    def filter_assignee_id(self, queryset, name, value):
        # 返回Q对象，同时过滤软删除的关联记录
        return Q(
            issue_assignee__assignee_id=value,
            issue_assignee__deleted_at__isnull=True,
        )
    
    # 标准字段过滤
    state_id = filters.UUIDFilter(field_name="state_id")
    priority = filters.CharFilter(field_name="priority")
```

**自定义方法的作用**：
- 处理关联表的软删除排除
- 实现复杂的业务逻辑过滤
- 保持接口一致性，外部调用无需关心内部实现

---

## 八、字段重叠时的叠加语义分析

> **新增章节**：针对同一字段在两套过滤系统同时出现的情况，提供求交规则、互斥场景分析和排查指南。

当同一字段在**富过滤器（rich filters）**和**旧版扁平参数**中同时出现时，理解它们的叠加规则对于定位结果偏差至关重要。

### 8.1 后端执行顺序与求交规则

#### 8.1.1 执行顺序

在 `IssueViewSet.list` 中，两套过滤系统按以下顺序依次应用：

```python
# 代码位置: apps/api/plane/app/views/issue/base.py:265-271
issue_queryset = self.get_queryset()  # 基础查询

# 步骤1: 应用富过滤器 (ComplexFilterBackend)
# 解析 ?filters={"and":[{"state_id__exact":"xxx"},...]}
issue_queryset = self.filter_queryset(issue_queryset)

# 步骤2: 应用旧版过滤器 (issue_filters)
# 解析 ?state=xxx&priority=yyy 等扁平参数
filters = issue_filters(query_params, "GET")
issue_queryset = issue_queryset.filter(**filters)
```

**关键结论**：两套过滤条件是**独立应用、顺序叠加**的关系，通过多次 `.filter()` 调用实现逻辑 AND。

#### 8.1.2 求交规则

Django QuerySet 的多次 `.filter()` 调用遵循以下规则：

| 场景 | 表达式 | 实际SQL行为 |
|------|--------|------------|
| 同一字段等值 | `.filter(state_id=A).filter(state_id=B)` | `WHERE state_id = A AND state_id = B` → 求交（交集） |
| 同一字段范围 | `.filter(priority__gte="high").filter(priority__lte="urgent")` | `WHERE priority >= "high" AND priority <= "urgent"` → 范围交集 |
| 不同字段 | `.filter(state_id=A).filter(priority=B)` | `WHERE state_id = A AND priority = B` → 独立条件 |

**对于重叠字段，最终结果是两套条件的严格交集**。如果条件互斥，结果将为空集。

### 8.2 字段映射对照表

旧版过滤器与富过滤器使用不同的字段命名体系。`LegacyToRichFiltersConverter` 中定义了映射关系：

| 旧版参数名 | 富过滤器字段名 | 说明 |
|-----------|---------------|------|
| `state` | `state_id` | 状态ID |
| `state_group` | `state_group` | 状态分组 |
| `priority` | `priority` | 优先级 |
| `assignees` | `assignee_id` | 处理人 |
| `labels` | `label_id` | 标签 |
| `cycle` | `cycle_id` | 迭代 |
| `module` | `module_id` | 模块 |
| `mentions` | `mention_id` | 提及人 |
| `created_by` | `created_by_id` | 创建人 |
| `project` | `project_id` | 项目 |
| `subscriber` | `subscriber_id` | 订阅人 |
| `start_date` | `start_date` | 开始日期 |
| `target_date` | `target_date` | 截止日期 |

> **重要提示**：旧版参数使用复数形式（如 `assignees`、`labels`），富过滤器使用单数形式（如 `assignee_id`、`label_id`）。

### 8.3 典型互斥场景与结果偏差分析

#### 场景1：同一字段的不同值（最常见）

**请求示例**：
```
?filters={"and":[{"priority__exact":"high"}]}
&priority=low
```

**执行过程**：
1. 富过滤器生成：`Q(priority__exact="high")`
2. 旧版过滤器生成：`{"priority__in": ["low"]}`
3. 叠加后SQL：`WHERE priority = 'high' AND priority IN ('low')`

**结果**：空集（没有issue能同时是high和low）

**偏差表现**：用户看到"无结果"，但单独使用任一条件都有结果。

---

#### 场景2：同一字段的范围冲突

**请求示例**：
```
?filters={"and":[{"target_date__lte":"2024-01-15"}]}
&target_date=2024-01-20;after
```

**执行过程**：
1. 富过滤器：`target_date <= '2024-01-15'`
2. 旧版过滤器：`target_date >= '2024-01-20'`
3. 叠加后：`target_date <= '2024-01-15' AND target_date >= '2024-01-20'`

**结果**：空集（日期范围无交集）

---

#### 场景3：包含与排除的冲突

**请求示例**：
```
?filters={"and":[{"assignee_id__in":["user1","user2"]}]}
&assignees=None   // None在旧版中表示"未分配"
```

**执行过程**：
1. 富过滤器：`assignee_id IN ('user1', 'user2')`
2. 旧版过滤器解析 `None` → `assignees__isnull = True`
3. 叠加后：`assignee_id IN (...) AND assignees__isnull = True`

**结果**：空集（有处理人和无处理人互斥）

---

#### 场景4：多值条件的子集关系

**请求示例**：
```
?filters={"and":[{"state_id__in":["backlog","todo","done"]}]}
&state=backlog,todo
```

**执行过程**：
1. 富过滤器：`state_id IN ('backlog', 'todo', 'done')`
2. 旧版过滤器：`state__in ('backlog', 'todo')`
3. 叠加后：`state_id IN ('backlog', 'todo', 'done') AND state__in ('backlog', 'todo')`

**结果**：等价于 `state_id IN ('backlog', 'todo')`（取交集）

**偏差表现**：结果比用户预期的少（缺少 `done` 状态的issue）

---

#### 场景5：关联表软删除的双重过滤

**请求示例**：
```
?filters={"and":[{"assignee_id__exact":"user1"}]}
&assignees=user1
```

**执行过程**：
1. 富过滤器（IssueFilterSet 自定义方法）：
   ```python
   Q(issue_assignee__assignee_id="user1", issue_assignee__deleted_at__isnull=True)
   ```
2. 旧版过滤器（filter_assignees 函数）：
   ```python
   {
       "assignees__in": ["user1"],
       "issue_assignee__deleted_at__isnull": True
   }
   ```
3. 叠加后：两套条件同时生效，软删除排除被应用两次（不影响结果，但增加查询复杂度）

**结果**：正常，但查询性能略有下降（重复条件）

### 8.4 排查指南：优先核对的参数组合

当遇到结果偏差时，按以下优先级排查：

#### 8.4.1 高风险参数组合（优先检查）

| 优先级 | 参数名 | 风险点 | 检查方法 |
|--------|--------|--------|---------|
| 🔴 最高 | `state` / `state_id` | 状态值冲突 | 查看是否同时传了 `?state=` 和 `?filters` 中的 `state_id` |
| 🔴 最高 | `priority` | 优先级值冲突 | 检查 `?priority=` 和 `filters` 中的 `priority` |
| 🔴 最高 | `assignees` / `assignee_id` | 处理人冲突 / None值 | 注意旧版 `assignees=None` 表示未分配 |
| 🟠 高 | `labels` / `label_id` | 标签ID冲突 | 检查多值组合的交集 |
| 🟠 高 | `cycle` / `cycle_id` | 迭代冲突 | |
| 🟠 高 | `module` / `module_id` | 模块冲突 | |
| 🟡 中 | `start_date` / `target_date` | 日期范围无交集 | 检查日期的先后顺序 |
| 🟡 中 | `created_by` / `created_by_id` | 创建人冲突 | |
| 🟢 低 | `state_group` | 状态组冲突 | 如 `started` 与 `completed` |

#### 8.4.2 后端执行顺序核对清单

1. **查看基础查询**：`get_queryset()` 是否已经隐含了过滤条件
2. **检查富过滤器**：`self.filter_queryset()` 应用了哪些条件
3. **检查旧版过滤器**：`issue_filters()` 返回的字典包含哪些键
4. **检查权限约束**：是否有 `filter(created_by=request.user)` 等强制条件
5. **确认最终SQL**：`print(queryset.query)` 查看实际生成的SQL

#### 8.4.3 快速定位工具

在调试环境中，可以添加以下代码查看各阶段的过滤条件：

```python
# 在 IssueViewSet.list 中添加调试代码
print("=== 过滤条件排查 ===")
print(f"1. 富过滤器参数: {request.query_params.get('filters')}")
print(f"2. 旧版过滤器结果: {filters}")
print(f"3. 最终SQL WHERE: {str(issue_queryset.query)}")
```

### 8.5 规避策略

1. **统一使用一套系统**：新功能优先使用富过滤器，避免混合使用
2. **前端参数清洗**：发送请求前移除重叠的旧版参数
3. **后端参数校验**：检测到重叠时返回警告或优先使用富过滤器
4. **文档明确说明**：在API文档中标明字段映射关系和叠加规则

---

## 九、Endpoint 级执行顺序对照

不同的 Issue 接口在过滤条件的执行顺序上存在细微但重要的差异。理解这些差异对于准确排查结果偏差至关重要。

### 9.1 IssueViewSet.list vs IssueDetailEndpoint.get 对比

#### 9.1.1 IssueViewSet.list 执行顺序

**代码位置**：`apps/api/plane/app/views/issue/base.py:254-352`

```python
def list(self, request, slug, project_id):
    # 1. 解析旧版过滤器（但不应用）
    filters = issue_filters(query_params, "GET")
    
    # 2. 基础查询
    issue_queryset = self.get_queryset()
    
    # 3. 应用富过滤器 (ComplexFilterBackend)
    issue_queryset = self.filter_queryset(issue_queryset)
    
    # 4. 应用旧版过滤器
    issue_queryset = issue_queryset.filter(**filters, **extra_filters)
    
    # 5. 权限约束（访客过滤）
    if (is_guest and not project.guest_view_all_features):
        issue_queryset = issue_queryset.filter(created_by=request.user)
```

**执行顺序**：
```
基础查询 → 富过滤器 → 旧版过滤器 → 权限约束
```

**关键点**：
- 旧版过滤器先解析，后应用（在富过滤器之后）
- 权限约束在**最后**，优先级最高，无法被前面的过滤条件绕过

---

#### 9.1.2 IssueDetailEndpoint.get 执行顺序

**代码位置**：`apps/api/plane/app/views/issue/base.py:1016-1091`

```python
def get(self, request, slug, project_id):
    # 1. 解析旧版过滤器（但不应用）
    filters = issue_filters(request.query_params, "GET")
    
    # 2. 基础查询 + 权限子查询（最优先）
    issue = Issue.issue_objects.filter(workspace__slug=slug, project_id=project_id) \
                                .filter(Exists(permission_subquery))
    
    # 3. 应用富过滤器
    issue = self.filter_queryset(issue)
    
    # 4. 应用旧版过滤器
    issue = issue.filter(**filters)
```

**执行顺序**：
```
基础查询 → 权限约束（嵌入）→ 富过滤器 → 旧版过滤器
```

**关键点**：
- 权限约束**嵌入在基础查询**中，使用 `EXISTS` 子查询实现
- 权限约束在**最前面**，在富过滤器和旧版过滤器之前生效
- 权限逻辑更复杂，支持三种角色场景（管理员/成员、访客+查看所有、访客+仅自己）

---

### 9.2 执行顺序差异对照表

| 维度 | IssueViewSet.list | IssueDetailEndpoint.get | 影响 |
|------|------------------|-------------------------|------|
| **权限约束时机** | 最后（过滤后） | 最前（基础查询中） | list 端点权限检查在分页/统计之后，detail 端点在最前 |
| **权限实现方式** | 简单 `.filter(created_by=user)` | 复杂 `EXISTS` 子查询 | detail 端点权限逻辑更精细，支持三种角色场景 |
| **过滤叠加关系** | 富过滤器 → 旧版 → 权限 | 权限 → 富过滤器 → 旧版 | 权限在 list 中优先级最高，在 detail 中同样最高但实现不同 |
| **过滤拷贝时机** | 旧版过滤器结果被传入 `issue_group_values` 用于分组统计 | 仅用于结果过滤 | list 端点旧版过滤器结果被复用在多个地方 |
| **适用场景** | 列表页、看板、表格等多结果展示 | 详情页、展开视图等单结果展示 | 排查时需根据调用的端点选择不同的分析路径 |

### 9.3 对同字段重叠场景排查的影响

#### 9.3.1 场景1：结果为空但条件单独有效

**在 list 端点排查**：
1. ✅ 检查旧版过滤器是否在富过滤器之后**额外添加**了冲突条件
2. ✅ 检查权限约束是否在**最后**进一步缩小了结果集
3. ❌ 不需要检查权限是否在过滤前排除了数据（权限在最后）

**在 detail 端点排查**：
1. ✅ 检查权限子查询是否在**最前面**就排除了数据
2. ✅ 检查富过滤器是否在权限基础上**进一步**缩小了范围
3. ✅ 检查旧版过滤器是否在**最后**添加了冲突条件

#### 9.3.2 场景2：权限约束导致的结果偏差

**list 端点**：
```python
# 权限在最后，相当于在所有过滤条件上再加一层
queryset = (
    Issue.objects.filter(workspace=slug, project=pid)
    .filter(rich_filters_q)           # 富过滤器
    .filter(**legacy_filters)          # 旧版过滤器
    .filter(created_by=request.user)   # 权限约束（最后）
)
```
→ 即使过滤条件匹配，访客用户也只能看到自己创建的

**detail 端点**：
```python
# 权限嵌入在基础查询中，使用 EXISTS 子查询
queryset = (
    Issue.objects.filter(workspace=slug, project=pid)
    .filter(Exists(permission_subquery))  # 权限约束（最前）
    .filter(rich_filters_q)               # 富过滤器
    .filter(**legacy_filters)             # 旧版过滤器
)
```
→ 权限不通过的 issue 甚至不会进入后续过滤流程

#### 9.3.3 场景3：分页/统计与实际结果不一致

**问题表现**：list 端点返回的 total_count 与实际分页结果数量不符

**排查路径**：
1. 权限约束在 `filtered_issue_queryset` 被拷贝**之后**才应用
2. 分组统计使用的是 `filtered_issue_queryset`（不含权限约束）
3. 最终返回的 `issue_queryset` 包含权限约束
4. 这是设计意图：分组统计显示全量数据，实际返回受权限约束

```python
# 代码位置: base.py:274, 308-309
filtered_issue_queryset = copy.deepcopy(issue_queryset)  # 拷贝时还没加权限

# ... 中间应用了各种过滤 ...

# 权限在拷贝之后才加
issue_queryset = issue_queryset.filter(created_by=request.user)
filtered_issue_queryset = filtered_issue_queryset.filter(created_by=request.user)
```

> ⚠️ **注意**：`filtered_issue_queryset` 在拷贝后也被应用了权限约束，所以实际上统计结果是准确的。但如果代码版本不同，可能存在不一致。

### 9.4 排查时的端点判断方法

当遇到过滤结果偏差时，首先确定调用的是哪个端点：

| 端点特征 | 判定方法 | 排查侧重点 |
|---------|---------|-----------|
| **list 端点** | URL 为 `/api/v1/workspaces/{slug}/projects/{pid}/issues/` | 检查权限是否在最后过滤掉了结果 |
| **detail 列表端点** | URL 为 `/api/v1/workspaces/{slug}/projects/{pid}/issues/` 但使用 `IssueDetailEndpoint` | 检查权限子查询是否在最前面排除了数据 |
| **单个详情** | URL 包含 `issue_id` 或 `issue_identifier` | 通常不涉及过滤条件叠加，主要检查权限 |

**快速判断调用的类**：在 `base.py` 中搜索 URL 路径对应的 `as_view()` 调用，或查看 Django 的 URL 配置。

### 9.5 调试建议

#### 针对 list 端点：
```python
# 在 list 方法中添加调试
print(f"[LIST] 富过滤器后数量: {issue_queryset.count()}")
print(f"[LIST] 旧版过滤器后数量: {issue_queryset.count()}")
print(f"[LIST] 权限约束后数量: {issue_queryset.count()}")
```

#### 针对 detail 端点：
```python
# 在 get 方法中添加调试
print(f"[DETAIL] 权限后数量: {issue.count()}")
print(f"[DETAIL] 富过滤器后数量: {issue.count()}")
print(f"[DETAIL] 旧版过滤器后数量: {issue.count()}")
```

#### 通用调试：
```python
# 查看实际执行的 SQL
print(f"最终SQL: {str(issue_queryset.query)}")
```

---

## 十、完整链路示例

### 场景：用户在UI选择"状态为待办 且 优先级为高或紧急"

#### 步骤1：前端状态
```typescript
// FilterInstance.expression
{
  id: "group-1",
  type: "group",
  logicalOperator: "and",
  children: [
    {
      id: "cond-1",
      type: "condition",
      property: "state_id",
      operator: "exact",
      value: "backlog-uuid"
    },
    {
      id: "cond-2",
      type: "condition",
      property: "priority",
      operator: "in",
      value: ["high", "urgent"]
    }
  ]
}
```

#### 步骤2：Adapter 转换为外部格式
```javascript
// toExternal 输出
{
  "and": [
    { "state_id__exact": "backlog-uuid" },
    { "priority__in": "high,urgent" }
  ]
}
```

#### 步骤3：后端接收并解析
```python
# 前端发送：?filters={"and":[{"state_id__exact":"backlog-uuid"},{"priority__in":"high,urgent"}]}

# _evaluate_node 递归处理
# 1. 遇到 "and" → 遍历子节点
# 2. 子节点1 {"state_id__exact": "backlog-uuid"} → Q(state_id__exact="backlog-uuid")
# 3. 子节点2 {"priority__in": "high,urgent"} → Q(priority__in=["high", "urgent"])
# 4. AND组合 → Q(...) & Q(...)

# 最终生成的Q对象
Q(state_id__exact="backlog-uuid") & Q(priority__in=["high", "urgent"])
```

#### 步骤4：应用到查询
```python
queryset = Issue.objects.filter(
    Q(state_id__exact="backlog-uuid") & Q(priority__in=["high", "urgent"])
)
```

---

## 十一、关键设计决策

### 11.1 为什么使用表达式树而不是扁平字典？
- 支持复杂的逻辑组合（AND/OR/NOT嵌套）
- 每个条件有唯一标识，便于精确更新/删除
- 类型安全，结构清晰

### 11.2 为什么需要 Adapter 层？
- **解耦**：前端内部结构与后端API格式独立演化
- **兼容**：支持不同业务场景的外部格式（工作项、自动化等）
- **转换**：处理多值逗号分隔、特殊字段映射等

### 11.3 后端为什么使用 Q 对象组合？
- **性能**：所有条件一次性生成 SQL，避免多次查询
- **灵活**：支持任意复杂的逻辑组合
- **安全**：通过 FilterSet 白名单验证，防止SQL注入

### 11.4 自定义过滤方法 vs 标准过滤
- **标准过滤**：直接映射数据库字段，性能最优
- **自定义方法**：处理软删除、权限、复杂业务逻辑
- 统一返回 Q 对象，保持组合逻辑一致

---

## 十二、常见问题排查

### 12.1 过滤条件不生效？
1. 检查前端 `expression` 是否正确构建
2. 检查 `toExternal` 转换后的格式是否正确
3. 检查后端 `filters` 参数是否正确接收
4. 查看 FilterSet 中是否声明了该字段

### 12.2 多值条件只匹配第一个？
- 确认操作符是 `__in` 而不是 `__exact`
- 检查逗号分隔是否正确解析（`_parseFilterValue`）

### 12.3 关联表过滤结果重复？
- 检查是否需要 `.distinct()`
- 查看 FilterSet 中是否设置了 `distinct=True`

### 12.4 软删除记录仍然出现？
- 确认使用了自定义过滤方法（如 `filter_assignee_id`）
- 检查 Q 对象中是否包含 `deleted_at__isnull=True` 条件

---

## 十三、代码文件索引

| 层级 | 文件路径 | 职责 |
|------|---------|------|
| 类型定义 | `packages/types/src/rich-filters/expression.ts` | 表达式树数据结构 |
| 类型定义 | `packages/types/src/rich-filters/adapter.ts` | Adapter 接口定义 |
| 工具函数 | `packages/utils/src/rich-filters/operations/traversal/core.ts` | 树遍历工具 |
| 前端状态 | `packages/shared-state/src/store/rich-filters/filter.ts` | FilterInstance 核心类 |
| 前端状态 | `packages/shared-state/src/store/rich-filters/filter-helpers.ts` | 过滤操作辅助类 |
| 格式转换 | `packages/shared-state/src/store/work-item-filters/adapter.ts` | 工作项过滤器 Adapter |
| 前端参数 | `apps/web/core/store/issue/helpers/issue-filter-helper.store.ts` | 过滤器参数序列化 |
| 前端查询 | `apps/web/core/store/issue/project/issue.store.ts` | 查询触发与响应处理 |
| 后端核心 | `apps/api/plane/utils/filters/filter_backend.py` | ComplexFilterBackend |
| 后端核心 | `apps/api/plane/utils/filters/filterset.py` | BaseFilterSet 与 Q 对象构建 |
| 字段映射 | `apps/api/plane/utils/filters/converters.py` | LegacyToRichFiltersConverter 新旧字段映射 |
| 旧版过滤 | `apps/api/plane/utils/issue_filters.py` | 扁平参数过滤器 |
| 业务视图 | `apps/api/plane/app/views/issue/base.py` | IssueViewSet 过滤管线 |
| 基类视图 | `apps/api/plane/api/views/base.py` | BaseAPIView filter_queryset |
