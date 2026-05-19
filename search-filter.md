# 搜索过滤条件解析、组合与查询拼装逻辑

## 一、整体架构概览

搜索过滤系统采用**分层设计**，从前端用户交互到后端数据库查询，经过多层转换和组合：

```
用户交互（UI选择过滤条件）
    ↓
前端状态管理（FilterInstance）
    ↓
内部表达式树（TFilterExpression）
    ↓
Adapter转换（toExternal）
    ↓
外部格式（field__operator: value, AND/OR组合）
    ↓
后端ComplexFilterBackend
    ↓
Django Q对象组合
    ↓
数据库查询（QuerySet.filter）
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

## 五、后端查询拼装

### 5.1 ComplexFilterBackend

定义位置：`apps/api/plane/utils/filters/filter_backend.py`

这是后端过滤的核心，负责：
1. 解析前端发送的 JSON 过滤条件
2. 验证结构和字段合法性
3. 递归构建 Django Q 对象
4. 应用到 QuerySet

#### 5.1.1 处理流程

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

#### 5.1.2 节点求值（_evaluate_node）

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

### 5.2 FilterSet 与 Q 对象构建

定义位置：`apps/api/plane/utils/filters/filterset.py`

#### 5.2.1 BaseFilterSet.build_combined_q

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

#### 5.2.2 IssueFilterSet 示例

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

## 六、完整链路示例

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

## 七、关键设计决策

### 7.1 为什么使用表达式树而不是扁平字典？
- 支持复杂的逻辑组合（AND/OR/NOT嵌套）
- 每个条件有唯一标识，便于精确更新/删除
- 类型安全，结构清晰

### 7.2 为什么需要 Adapter 层？
- **解耦**：前端内部结构与后端API格式独立演化
- **兼容**：支持不同业务场景的外部格式（工作项、自动化等）
- **转换**：处理多值逗号分隔、特殊字段映射等

### 7.3 后端为什么使用 Q 对象组合？
- **性能**：所有条件一次性生成 SQL，避免多次查询
- **灵活**：支持任意复杂的逻辑组合
- **安全**：通过 FilterSet 白名单验证，防止SQL注入

### 7.4 自定义过滤方法 vs 标准过滤
- **标准过滤**：直接映射数据库字段，性能最优
- **自定义方法**：处理软删除、权限、复杂业务逻辑
- 统一返回 Q 对象，保持组合逻辑一致

---

## 八、常见问题排查

### 8.1 过滤条件不生效？
1. 检查前端 `expression` 是否正确构建
2. 检查 `toExternal` 转换后的格式是否正确
3. 检查后端 `filters` 参数是否正确接收
4. 查看 FilterSet 中是否声明了该字段

### 8.2 多值条件只匹配第一个？
- 确认操作符是 `__in` 而不是 `__exact`
- 检查逗号分隔是否正确解析（`_parseFilterValue`）

### 8.3 关联表过滤结果重复？
- 检查是否需要 `.distinct()`
- 查看 FilterSet 中是否设置了 `distinct=True`

### 8.4 软删除记录仍然出现？
- 确认使用了自定义过滤方法（如 `filter_assignee_id`）
- 检查 Q 对象中是否包含 `deleted_at__isnull=True` 条件

---

## 九、代码文件索引

| 层级 | 文件路径 | 职责 |
|------|---------|------|
| 类型定义 | `packages/types/src/rich-filters/expression.ts` | 表达式树数据结构 |
| 类型定义 | `packages/types/src/rich-filters/adapter.ts` | Adapter 接口定义 |
| 工具函数 | `packages/utils/src/rich-filters/operations/traversal/core.ts` | 树遍历工具 |
| 前端状态 | `packages/shared-state/src/store/rich-filters/filter.ts` | FilterInstance 核心类 |
| 前端状态 | `packages/shared-state/src/store/rich-filters/filter-helpers.ts` | 过滤操作辅助类 |
| 格式转换 | `packages/shared-state/src/store/work-item-filters/adapter.ts` | 工作项过滤器 Adapter |
| 后端核心 | `apps/api/plane/utils/filters/filter_backend.py` | ComplexFilterBackend |
| 后端核心 | `apps/api/plane/utils/filters/filterset.py` | BaseFilterSet 与 Q 对象构建 |
| 业务实现 | `apps/api/plane/utils/filters/filterset.py` | IssueFilterSet 示例 |
