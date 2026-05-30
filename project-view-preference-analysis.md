# 项目视图用户偏好：序列化、共享与回放机制梳理

---

## 1. 整体架构概览

项目视图（Project View）系统的用户偏好管理横跨 **前端 MobX Store → API Serializer → Django ORM Model** 三层，涉及以下核心概念：

| 概念 | 说明 |
|------|------|
| `rich_filters` | 新版结构化过滤表达式（AST 树形，支持 AND/OR/NOT 逻辑组合） |
| `display_filters` | 展示类过滤项（group_by, order_by, layout, sub_issue 等） |
| `display_properties` | 列/字段可见性控制（assignee, priority, state 等开关） |
| `filters`（legacy） | 旧版扁平过滤字典，已被 `rich_filters` 取代但仍保留兼容 |
| `query` | 服务端由 `filters` 自动派生的 ORM 查询参数，只读 |
| `access` | 视图可见性：`0 = Private` / `1 = Public` |
| `is_locked` | 视图锁定标志，锁定后禁止修改 |
| `owned_by` | 视图所有者（ForeignKey → User） |

---

## 2. 私有视图与共享视图的存储区别

### 2.1 数据模型层

`IssueView` 模型定义于 `apps/api/plane/db/models/view.py:58-98`，所有视图共享同一张 `issue_views` 表，**不区分存储结构**，通过 `access` 字段区分可见性：

```python
access = models.PositiveSmallIntegerField(
    default=1,
    choices=((0, "Private"), (1, "Public"))
)
owned_by = models.ForeignKey(settings.AUTH_USER_MODEL, on_delete=models.CASCADE, related_name="views")
is_locked = models.BooleanField(default=False)
```

关键设计：
- **私有视图**（`access=0`）：仅 `owned_by` 用户可见。
- **公开视图**（`access=1`）：项目/工作区内所有成员可见。
- `is_locked`：公开视图可被锁定，锁定后即使所有者也不能修改（API 返回 `400`）。

### 2.2 API 查询层的可见性过滤

在 `apps/api/plane/app/views/view/base.py` 中，`IssueViewViewSet.get_queryset()` 通过 Q 对象实现访问控制：

```python
.filter(Q(owned_by=self.request.user) | Q(access=1))
```

即：**返回当前用户自己创建的视图 + 所有公开视图**。

对于 `WorkspaceViewViewSet` 也有相同逻辑（第 66 行），额外加上 `.filter(project__isnull=True)` 过滤出全局视图。

### 2.3 Guest 用户的额外限制

当项目成员角色为 Guest（`role=5`）且项目设置 `guest_view_all_features=False` 时：

```python
# IssueViewViewSet.list() — base.py:293-303
if ProjectMember.objects.filter(
    role=5, is_active=True
).exists() and not project.guest_view_all_features:
    queryset = queryset.filter(owned_by=request.user)
```

Guest 用户在 `guest_view_all_features=False` 时**只能看到自己创建的视图**，即使该视图是 Public 的。

### 2.4 修改与删除权限

| 操作 | 权限要求 |
|------|----------|
| **更新视图** | 仅 `owned_by` 用户；且视图不能 `is_locked` |
| **删除视图** | 项目 Admin（`role=20`）或 `owned_by` 用户 |
| **创建视图** | 项目 Admin / Member / Guest 均可 |
| **收藏视图** | Admin / Member |

`partial_update` 中的关键校验（`base.py:344-363`）：
```python
if issue_view.is_locked:
    return Response({"error": "view is locked"}, status=400)
if issue_view.owned_by_id != request.user.id:
    return Response({"error": "Only the owner of the view can update the view"}, status=400)
```

### 2.5 前端权限判断

在 `project-level.tsx` 中，前端对保存/更新视图做了两层权限判断：

```typescript
const canCreateView = projectDetails?.issue_views_view === true
    && enableSaveView && hasProjectMemberLevelPermissions;

const canUpdateView = enableUpdateView
    && !isViewLocked && hasProjectMemberLevelPermissions && isCurrentUserOwner;
```

- 创建视图需要项目开启了 `issue_views_view` 功能特性。
- 更新视图需要当前用户是视图所有者且视图未锁定。

### 2.6 序列化器的只读字段保护

`IssueViewSerializer`（`apps/api/plane/app/serializers/view.py:56-86`）将以下字段设为 `read_only`：

```python
read_only_fields = ["workspace", "project", "query", "owned_by", "access", "is_locked"]
```

- `access` 和 `is_locked` 在创建时可通过请求设定，但创建后不可通过 PATCH 修改。
- `owned_by` 由 `perform_create` 自动注入为 `request.user`。

---

## 3. URL 与服务端状态的同步

### 3.1 路由参数提取

`RouterStore`（`apps/web/core/store/router.store.ts`）通过 MobX observable 存储当前 URL 查询参数：

```typescript
get viewId() {
    return this.query?.viewId?.toString();
}
```

视图 ID 从路由参数 `[viewId]` 中获取，各 Store 通过 `this.rootStore.router.viewId` 访问。

### 3.2 视图详情的 SWR 获取

视图详情页（`apps/web/app/.../[viewId]/page.tsx`）使用 SWR 按需获取：

```typescript
const { error } = useSWR(
    `VIEW_DETAILS_${viewId}`,
    () => fetchViewDetails(workspaceSlug, projectId, viewId)
);
```

获取后数据写入 `ProjectViewStore.viewMap[viewId]`。

### 3.3 过滤器的初始化与回放

当用户进入视图详情页时，`ProjectViewIssuesFilter.fetchFilters()` 被调用：

```typescript
fetchFilters = async (workspaceSlug, projectId, viewId) => {
    const viewDetails = await this.issueFilterService.getViewDetails(
        workspaceSlug, projectId, viewId
    );
    this.mutateFilters(workspaceSlug, viewId, viewDetails);
};
```

`mutateFilters` 从视图详情中提取三层偏好并写入 Store：

```typescript
mutateFilters = action((workspaceSlug, viewId, viewDetails) => {
    const richFilters = viewDetails?.rich_filters;
    const displayFilters = this.computedDisplayFilters(viewDetails?.display_filters);
    const displayProperties = this.computedDisplayProperties(viewDetails?.display_properties);
    // Kanban 折叠状态从 localStorage 读取
    const kanbanFilters = this.handleIssuesLocalFilters.get(
        EIssuesStoreType.PROJECT_VIEW, workspaceSlug, viewId, currentUserId
    );
    runInAction(() => {
        set(this.filters, [viewId, "richFilters"], richFilters);
        set(this.filters, [viewId, "displayFilters"], displayFilters);
        set(this.filters, [viewId, "displayProperties"], displayProperties);
        set(this.filters, [viewId, "kanbanFilters"], kanbanFilters);
    });
});
```

### 3.4 过滤参数转 API 请求参数

`getAppliedFilters` 将 Store 中的过滤器转为 API 请求参数：

```typescript
getAppliedFilters(viewId) {
    const userFilters = this.getIssueFilters(viewId);
    const filteredParams = handleIssueQueryParamsByLayout(
        userFilters?.displayFilters?.layout, "issues"
    );
    return this.computedFilteredParams(
        userFilters?.richFilters,
        userFilters?.displayFilters,
        filteredParams
    );
}
```

`handleIssueQueryParamsByLayout`（`packages/utils/src/work-item/base.ts:104-134`）根据当前布局类型决定哪些参数应该被传递给 API，避免冗余查询。

### 3.5 URL 查询参数工具

`useQueryParams` hook（`apps/web/core/hooks/use-query-params.ts`）提供通用 URL 参数增删能力，但视图页面主要通过 MobX Store + Router Store 同步状态，**不依赖 URL query params 存储过滤器**。过滤器状态完全由服务端持久化 + 客户端 Store 缓存驱动。

### 3.6 全局视图的同步回写

`GlobalViewStore.updateGlobalView()` 中，当 `rich_filters` 发生变化时，会立即同步到 issue 过滤器 Store：

```typescript
if (shouldSyncFilters && !isEqual(currentViewData?.rich_filters, currentView?.rich_filters)) {
    await this.rootStore.issue.workspaceIssuesFilter.updateFilterExpression(
        workspaceSlug, viewId, currentView?.rich_filters || {}
    );
    this.rootStore.issue.workspaceIssues.fetchIssuesWithExistingPagination(
        workspaceSlug, viewId, "mutation"
    );
}
```

---

## 4. 字段权限对过滤选项的影响

### 4.1 项目功能开关

项目模型上的功能开关直接影响过滤器选项的可用性：

| 项目字段 | 影响范围 |
|----------|----------|
| `issue_views_view` | 控制视图功能的可见性，关闭后隐藏视图入口 |
| `cycle_view` | 关闭后 `DisplayFiltersSelection` 禁用 cycle 相关选项 |
| `module_view` | 关闭后 `DisplayFiltersSelection` 禁用 module 相关选项 |
| `guest_view_all_features` | 关闭后 Guest 只能看自己创建的视图和 issue |

前端 Header 中的传递方式（`header.tsx:193-195`）：

```tsx
<DisplayFiltersSelection
    cycleViewDisabled={!currentProjectDetails?.cycle_view}
    moduleViewDisabled={!currentProjectDetails?.module_view}
/>
```

### 4.2 项目级别过滤器 HOC 的字段集合

`ProjectLevelWorkItemFiltersHOC`（`project-level.tsx`）向过滤器组件注入当前项目上下文的 ID 集合：

```typescript
<WorkItemFiltersHOC
    cycleIds={getProjectCycleIds(projectId) ?? undefined}
    labelIds={getProjectLabelIds(projectId)}
    memberIds={getProjectMemberIds(projectId, false) ?? undefined}
    moduleIds={getProjectModuleIds(projectId) ?? undefined}
    stateIds={getProjectStateIds(projectId)}
    saveViewOptions={saveViewOptions}
    updateViewOptions={updateViewOptions}
>
```

这意味着过滤器的下拉选项**随项目成员、状态、标签等动态变化**，用户只能选择项目中实际存在的值。

### 4.3 服务端字段权限过滤

`WorkspaceViewIssuesViewSet._get_project_permission_filters()` 对 Guest 用户施加了额外约束：

```python
Q(
    Q(project__project_projectmember__role=5,
      project__guest_view_all_features=True)
    | Q(project__project_projectmember__role=5,
        project__guest_view_all_features=False,
        created_by=self.request.user)
    | Q(project__project_projectmember__role__gt=5),
    project__project_projectmember__member=self.request.user,
    project__project_projectmember__is_active=True,
)
```

- Guest 且 `guest_view_all_features=True`：可见所有 issue。
- Guest 且 `guest_view_all_features=False`：只可见自己创建的 issue。
- 非 Guest：可见所有 issue。

### 4.4 复杂过滤器后端的字段白名单

`ComplexFilterBackend._validate_fields()` 强制校验过滤字段是否在视图的 `filterset_class.base_filters` 中注册：

```python
def _validate_fields(self, filter_data, view):
    filterset_class = getattr(view, "filterset_class", None)
    allowed_fields = set(filterset_class.base_filters.keys())
    for field in fields:
        if field not in allowed_fields:
            raise DRFValidationError({
                "message": f"Filtering on field '{field}' is not allowed",
                "code": "invalid_filter_field",
            })
```

未在 FilterSet 中声明的字段**禁止用于过滤**，防止了未授权字段暴露。

### 4.5 Rich Filter 表达式与字段类型

前端 `FilterInstance`（`packages/shared-state/src/store/rich-filters/filter.ts`）通过 `FilterConfigManager` 管理每个字段的配置：

- 每个字段有自己的 `operator` 列表（is, is_not, in, not_in, between 等）。
- 每个字段有值类型约束（UUID / choice / date）。
- 字段配置决定了过滤器的 UI 交互方式和可用操作符。

---

## 5. 视图迁移与版本兼容性

### 5.1 Legacy Filters → Rich Filters 迁移

系统经历了从扁平 `filters` 字典到结构化 `rich_filters` AST 表达式的重大迁移。

#### 5.1.1 转换器：LegacyToRichFiltersConverter

`apps/api/plane/utils/filters/converters.py` 实现了完整转换逻辑：

**字段名映射**（旧名 → 新名）：
```
state      → state_id
labels     → label_id
cycle      → cycle_id
module     → module_id
assignees  → assignee_id
created_by → created_by_id
priority   → priority        (不变)
state_group→ state_group     (不变)
start_date → start_date      (不变)
target_date→ target_date     (不变)
```

**值验证规则**：
- UUID 字段（state_id, label_id 等）校验 UUID 格式。
- Choice 字段（priority, state_group）校验枚举值。
- Date 字段（start_date, target_date）支持 `dateutil` 解析。

**输出格式**：
- 单个条件 → `{field__operator: value}`
- 多个条件 → `{"and": [{field__operator: value}, ...]}`

#### 5.1.2 数据迁移脚本

`apps/api/plane/db/migrations/0107_migrate_filters_to_rich_filters.py` 执行数据迁移：

```python
MODEL_NAMES = [
    "IssueView",
    "WorkspaceUserProperties",
    "ModuleUserProperties",
    "IssueUserProperty",
    "CycleUserProperties",
]
```

迁移逻辑（`filter_migrations.py`）：
1. 查找 `filters` 非空但 `rich_filters` 为空的记录。
2. 使用 `LegacyToRichFiltersConverter.convert(strict=False)` 转换。
3. 批量更新 `rich_filters` 字段（`bulk_update`, batch_size=1000）。
4. 支持 reverse migration（清空 `rich_filters`）。

`strict=False` 模式下，无效值会被静默跳过而非抛出异常，保证迁移不会因脏数据中断。

### 5.2 模型兼容性设计

`IssueView` 模型同时保留了旧版和新版字段：

```python
filters = models.JSONField(default=dict)           # 旧版
rich_filters = models.JSONField(default=dict)       # 新版
display_filters = models.JSONField(default=get_default_display_filters)
display_properties = models.JSONField(default=get_default_display_properties)
```

`save()` 方法中仍会从 `filters` 派生 `query`：

```python
def save(self, *args, **kwargs):
    query_params = self.filters
    self.query = issue_filters(query_params, "POST") if query_params else {}
    super().save(*args, **kwargs)
```

### 5.3 UserProperties 模型的统一结构

所有 `*UserProperties` 模型都包含相同的四字段组合：

| 模型 | 位置 |
|------|------|
| `WorkspaceUserProperties` | `workspace.py:310` |
| `CycleUserProperties` | `cycle.py:130` |
| `ModuleUserProperties` | `module.py:190` |

每个都有 `filters`, `display_filters`, `display_properties`, `rich_filters` 四个 JSONField，均纳入了迁移范围。

### 5.4 前端兼容性处理

前端 `ProjectViewIssuesFilter.mutateFilters()` 同时消费 `rich_filters` 和 `display_filters/display_properties`：

```typescript
const richFilters: TWorkItemFilterExpression = viewDetails?.rich_filters;
const displayFilters = this.computedDisplayFilters(viewDetails?.display_filters);
const displayProperties = this.computedDisplayProperties(viewDetails?.display_properties);
```

`computedDisplayFilters` 和 `computedDisplayProperties` 会与默认值合并，保证缺失字段有合理默认值。

### 5.5 Rich Filter 表达式结构

新版过滤表达式采用 AST 树形结构（`packages/types/src/rich-filters/expression.ts`）：

```
TFilterExpression = TFilterConditionNode | TFilterGroupNode

TFilterConditionNode = {
    id: string,
    type: "condition",
    property: string,       // 如 "state", "assignee"
    operator: TSupportedOperators,  // 如 "is", "is_not", "in"
    value: SingleOrArray<TFilterValue>
}

TFilterGroupNode = {
    id: string,
    type: "group",
    logicalOperator: "AND",
    children: TFilterExpression[]
}
```

`FilterInstance`（`packages/shared-state/src/store/rich-filters/filter.ts`）管理完整的过滤生命周期：
- 表达式初始化、变更检测（`hasChanges`）
- 条件增删改（`addCondition`, `updateConditionValue`, `removeCondition`）
- 视图保存/更新（`saveView`, `updateView`）
- 表达式重置（`resetExpression`，用于回放视图）

---

## 6. 已发布视图（Published View）

### 6.1 发布机制

视图通过 `anchor` 字段发布到 Space 应用（公开访问），`PublishViewModal` 在 CE 版为空壳，由 EE 版扩展实现。

`IPublishedProjectView` 类型定义与 `IProjectView` 的区别：

```typescript
export interface IPublishedProjectView extends Omit<IProjectView, "rich_filters"> {
    filters: IIssueFilterOptions;  // 使用旧版 filters 格式
}
```

发布链接生成逻辑（`packages/utils/src/project-views.ts:105-110`）：

```typescript
export const getPublishViewLink = (anchor: string | undefined) => {
    const SPACE_APP_URL = (SPACE_BASE_URL.trim() === "" 
        ? window.location.origin : SPACE_BASE_URL) + SPACE_BASE_PATH;
    return `${SPACE_APP_URL}/views/${anchor}`;
};
```

### 6.2 Space 端的视图回放

Space 应用的 `PublishStore` 存储发布设置，包括 `view_props`（`TProjectPublishViewProps`），控制公开视图的展示方式。

---

## 7. 关键代码位置索引

| 文件 | 说明 |
|------|------|
| `apps/api/plane/db/models/view.py` | IssueView 模型定义 |
| `apps/api/plane/app/serializers/view.py` | 视图序列化器 |
| `apps/api/plane/app/views/view/base.py` | 视图 API ViewSet（含权限逻辑） |
| `apps/web/core/store/project-view.store.ts` | 项目视图 Store |
| `apps/web/core/store/issue/project-views/filter.store.ts` | 视图过滤器 Store |
| `apps/web/core/store/issue/project-views/issue.store.ts` | 视图 Issue Store |
| `apps/web/core/store/global-view.store.ts` | 全局视图 Store |
| `apps/web/core/store/router.store.ts` | 路由参数 Store |
| `apps/web/core/hooks/use-query-params.ts` | URL 查询参数工具 |
| `apps/web/core/components/views/form.tsx` | 视图创建/编辑表单 |
| `apps/web/core/components/work-item-filters/filters-hoc/project-level.tsx` | 项目级过滤器 HOC |
| `apps/web/core/components/views/view-list-item-action.tsx` | 视图列表操作项 |
| `packages/types/src/views.ts` | 视图类型定义 |
| `packages/types/src/rich-filters/expression.ts` | 过滤表达式类型 |
| `packages/shared-state/src/store/rich-filters/filter.ts` | Rich Filter Store |
| `packages/utils/src/project-views.ts` | 视图工具函数 |
| `packages/utils/src/work-item/base.ts` | 布局参数映射 |
| `packages/constants/src/views.ts` | 视图常量 |
| `apps/api/plane/utils/filters/converters.py` | Legacy → Rich 过滤转换器 |
| `apps/api/plane/utils/filters/filter_migrations.py` | 过滤迁移工具 |
| `apps/api/plane/utils/filters/filter_backend.py` | 复杂过滤后端 |
| `apps/api/plane/db/migrations/0107_migrate_filters_to_rich_filters.py` | 数据迁移脚本 |
| `apps/space/store/publish/publish.store.ts` | Space 发布 Store |
