# 项目视图偏好：权限约束与状态同步机制深度分析

---

## 1. 核心修正说明

本文档是对前期分析的修正版本，重点修正了关于 `access` 和 `is_locked` 字段在创建和更新阶段约束的错误结论，并详细阐述了这些约束对私有/共享视图回放操作的具体影响。

---

## 2. `access` 与 `is_locked` 字段的完整约束分析

### 2.1 数据模型层定义

`IssueView` 模型（`apps/api/plane/db/models/view.py:58-98`）：

```python
access = models.PositiveSmallIntegerField(
    default=1,                       # 默认 PUBLIC
    choices=((0, "Private"), (1, "Public"))
)
is_locked = models.BooleanField(default=False)   # 默认未锁定
owned_by = models.ForeignKey(settings.AUTH_USER_MODEL, on_delete=models.CASCADE, related_name="views")
```

### 2.2 序列化器层的只读约束

**关键结论修正**：`access` 和 `is_locked` **在序列化器层面被永久标记为只读**，不仅更新时不能改，创建时也不能通过请求数据设置。

`IssueViewSerializer`（`apps/api/plane/app/serializers/view.py:56-69`）：

```python
class IssueViewSerializer(DynamicBaseSerializer):
    is_favorite = serializers.BooleanField(read_only=True)

    class Meta:
        model = IssueView
        fields = "__all__"
        read_only_fields = [
            "workspace",
            "project",
            "query",
            "owned_by",
            "access",           # ← 只读
            "is_locked",        # ← 只读
        ]
```

`DynamicBaseSerializer` 不修改 `read_only_fields` 逻辑，它只处理字段过滤和展开。DRF 序列化器的 `read_only_fields` 意味着这些字段在 `to_internal_value` 时会被忽略，无论创建还是更新。

### 2.3 创建阶段的实际行为

**创建流程图**：

```
前端发送 {name, access: 0, rich_filters, ...}
        ↓
serializer.is_valid() → access 被忽略（read_only）
        ↓
perform_create() 注入 project_id 和 owned_by
        ↓
模型创建 → access 使用默认值 1 (PUBLIC)
          is_locked 使用默认值 False
```

`IssueViewViewSet.perform_create`（`base.py:260-261`）：

```python
def perform_create(self, serializer):
    serializer.save(project_id=self.kwargs.get("project_id"), owned_by=self.request.user)
```

只注入 `project_id` 和 `owned_by`，没有设置 `access` 或 `is_locked`。

**前端行为**：`ProjectViewForm`（`form.tsx:92-106`）虽然在提交时包含了 `access` 字段：

```typescript
await handleFormSubmit({
    name: formData.name,
    description: formData.description,
    logo_props: formData.logo_props,
    rich_filters: formData.rich_filters,
    display_filters: formData.display_filters,
    display_properties: formData.display_properties,
    access: formData.access,    // ← 会被后端序列化器忽略
} as IProjectView);
```

但由于序列化器的 `read_only_fields`，这个值会被丢弃。

**CE 版限制**：`AccessController` 在 CE 版是空壳组件（`ce/components/views/access-controller.tsx:8-9`）：

```typescript
export function AccessController(props: any) {
    return <></>;
}
```

这意味着 CE 版用户**甚至无法在 UI 上选择视图的访问级别**，表单默认值 `access: EViewAccess.PUBLIC` 也无法改变。

### 2.4 更新阶段的约束检查

`IssueViewViewSet.partial_update`（`base.py:343-363`）执行三层检查：

```python
def partial_update(self, request, slug, project_id, pk):
    with transaction.atomic():
        issue_view = IssueView.objects.select_for_update().get(...)

        # 第一层检查：锁定视图不能修改
        if issue_view.is_locked:
            return Response({"error": "view is locked"}, status=400)

        # 第二层检查：仅所有者可修改
        if issue_view.owned_by_id != request.user.id:
            return Response(
                {"error": "Only the owner of the view can update the view"},
                status=400
            )

        # 第三层：序列化器验证（access 和 is_locked 仍是只读）
        serializer = IssueViewSerializer(issue_view, data=request.data, partial=True)
        if serializer.is_valid():
            serializer.save()  # ← access/is_locked 不会被修改
            return Response(serializer.data, status=200)
```

**重要结论**：即使所有者通过 API 尝试发送 `{"access": 0}` 或 `{"is_locked": true}`，也会被序列化器忽略。视图一旦创建，其 `access` 和 `is_locked` 属性在 CE 版中是**不可改变的**。

### 2.5 关于 `is_locked` 如何设置的疑问

由于 `is_locked` 在序列化器中是只读的，且没有专门的锁定/解锁端点，**CE 版实际上无法将视图设置为锁定状态**。该字段目前只有校验逻辑，没有设置逻辑。这可能是 EE 版的付费功能。

### 2.6 约束矩阵总结

| 操作 | `access` 可改吗？ | `is_locked` 可改吗？ | 权限要求 |
|------|-------------------|---------------------|----------|
| **创建视图** | ❌ 只读，默认 `1 (Public)` | ❌ 只读，默认 `False` | Admin / Member / Guest |
| **更新视图** | ❌ 只读 | ❌ 只读 | 仅 `owned_by`，且 `is_locked=False` |
| **删除视图** | — | — | Admin 或 `owned_by` |
| **锁定视图** | — | ❌ 无端点支持 | — |

---

## 3. URL 与服务端状态同步机制（修正版）

### 3.1 路由参数与视图 ID 提取

`RouterStore`（`apps/web/core/store/router.store.ts`）从路由参数中提取 `viewId`：

```typescript
get viewId() {
    return this.query?.viewId?.toString();
}
```

各 Store 通过 `this.rootStore.router.viewId` 访问当前视图 ID。

### 3.2 视图详情的按需加载

视图详情页（`[viewId]/page.tsx`）使用 SWR 按需加载：

```typescript
const { error } = useSWR(
    `VIEW_DETAILS_${viewId}`,
    () => fetchViewDetails(workspaceSlug, projectId, viewId)
);
```

加载后数据写入 `ProjectViewStore.viewMap[viewId]`。

### 3.3 过滤器状态的回放流程

当用户进入视图详情页时，`ProjectViewIssuesFilter.fetchFilters()` 被调用：

```typescript
fetchFilters = async (workspaceSlug, projectId, viewId) => {
    const viewDetails = await this.issueFilterService.getViewDetails(
        workspaceSlug, projectId, viewId
    );
    this.mutateFilters(workspaceSlug, viewId, viewDetails);
};
```

`mutateFilters`（`filter.store.ts:146-174`）从视图详情中提取三层偏好：

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

**关键点**：过滤器状态**完全由服务端持久化 + MobX Store 缓存驱动，不依赖 URL query params**。URL 仅承载 `viewId`。

### 3.4 过滤器到 API 参数的转换

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

`handleIssueQueryParamsByLayout`（`packages/utils/src/work-item/base.ts:104-134`）根据布局类型动态决定传递给 API 的参数子集，避免冗余查询。

### 3.5 更新时的乐观同步

`ProjectViewStore.updateView`（`project-view.store.ts:224-239`）采用先更新本地缓存再同步服务端的策略：

```typescript
async updateView(workspaceSlug, projectId, viewId, data) {
    const currentView = this.getViewById(viewId);
    
    // 乐观更新本地 Store
    runInAction(() => {
        set(this.viewMap, [viewId], { ...currentView, ...data });
    });
    
    // 同步到服务端
    const response = await this.viewService.patchView(workspaceSlug, projectId, viewId, data);
    
    return response;
}
```

这种策略保证了 UI 响应的即时性，但需要处理服务端失败的回滚（当前代码未显式处理回滚）。

### 3.6 全局视图的同步回写

`GlobalViewStore.updateGlobalView()` 中，当 `rich_filters` 变化时会同步到过滤器 Store：

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

## 4. 约束对视图回放操作的具体影响

### 4.1 视图回放的定义

**视图回放**（View Playback）是指用户通过点击视图链接或导航到视图详情页，从服务端加载视图配置（过滤条件、显示属性、布局设置等），并在前端应用这些配置，展示相应 issue 列表的完整过程。

回放流程：

```
用户访问 /workspace/projects/projectId/views/viewId
        ↓
RouterStore 提取 viewId
        ↓
ProjectViewStore.fetchViewDetails() 加载视图元数据
        ↓
ProjectViewIssuesFilter.fetchFilters() 加载并应用过滤器
        ↓
Issue 列表根据过滤器查询并展示
```

### 4.2 `access` 约束对回放的影响

#### 4.2.1 私有视图（`access=0`）的回放

**谁能回放？**
- `get_queryset()` 中的 `Q(owned_by=self.request.user) | Q(access=1)` 过滤确保只有所有者能看到私有视图。
- 非所有者在列表中看不到该视图，直接访问 URL 会被 `retrieve()` 中的权限检查拒绝（通过 `get_queryset().filter(pk=pk).first()` 返回 `None`，序列化 `None` 会报错）。

**回放内容**：
- 所有者可以完整回放所有过滤条件和显示设置。
- 但 issue 列表仍受 `_get_project_permission_filters` 约束（Guest 用户限制）。

**CE 版限制**：由于 `access` 是只读且默认 Public，CE 版实际上**无法创建私有视图**。

#### 4.2.2 公开视图（`access=1`）的回放

**谁能回放？**
- 项目内所有成员（Admin / Member / Guest）都能在列表中看到并回放。
- Guest 用户在 `guest_view_all_features=False` 时的额外限制：
  - `list()` 会将视图过滤为仅自己创建的（`base.py:293-303`）。
  - `retrieve()` 中如果访问他人创建的公开视图，会返回 403（`base.py:317-331`）。

**回放内容**：
- 所有成员都能回放完整的过滤条件和显示设置。
- 但 issue 列表仍受权限过滤：
  - Guest 且 `guest_view_all_features=True`：可见所有 issue。
  - Guest 且 `guest_view_all_features=False`：只可见自己创建的 issue。
  - 非 Guest：可见所有 issue。

### 4.3 `is_locked` 约束对回放的影响

**谁能回放锁定视图？**
- `get_queryset()` 不对 `is_locked` 做过滤，**锁定视图对所有可见用户都能正常回放**。
- 锁定只是防止修改，不影响查看和回放。

**回放时的交互限制**：
- 回放锁定视图时，前端应禁用保存/更新按钮：

```typescript
// project-level.tsx
const canUpdateView = enableUpdateView
    && !isViewLocked
    && hasProjectMemberLevelPermissions
    && isCurrentUserOwner;
```

- 用户可以调整过滤条件临时查看，但无法保存回视图。
- 如果用户修改了过滤器，`resetFilters()` 可以一键恢复到视图保存的状态：

```typescript
resetFilters: IProjectViewIssuesFilter["resetFilters"] = action((workspaceSlug, viewId) => {
    const viewDetails = this.rootIssueStore.rootStore.projectView.getViewById(viewId);
    if (!viewDetails) return;
    this.mutateFilters(workspaceSlug, viewId, viewDetails);
});
```

**CE 版限制**：由于无法将视图设置为锁定状态，CE 版用户实际上不会遇到被锁定的视图。

### 4.4 `owned_by` 约束对回放的影响

- 非所有者可以回放视图（如果是 Public），但不能修改。
- 所有者可以回放并修改视图（如果未锁定）。
- 修改后，所有访问该视图的用户都会看到更新后的配置（因为从同一条数据库记录读取）。

### 4.5 回放影响矩阵

| 视图类型 | 谁能看见/回放 | 回放时能否修改 | issue 可见性 |
|----------|---------------|----------------|-------------|
| **私有视图**<br>`access=0` | 仅 `owned_by` | 所有者可改（未锁定时） | 所有者权限范围内的所有 issue |
| **公开视图**<br>`access=1` | 所有项目成员 | 仅所有者可改（未锁定时） | 按角色权限过滤 |
| **锁定视图**<br>`is_locked=1` | 同 access 规则 | ❌ 任何人都不能改 | 同 access 规则 |
| **Guest + `guest_view_all_features=False`** | 仅自己创建的视图 | 仅自己创建的视图可改 | 仅自己创建的 issue |

---

## 5. 字段权限对过滤选项的影响

### 5.1 项目功能开关

| 项目字段 | 影响范围 |
|----------|----------|
| `issue_views_view` | 控制视图功能的可见性，关闭后隐藏视图入口 |
| `cycle_view` | 关闭后 `DisplayFiltersSelection` 禁用 cycle 相关选项 |
| `module_view` | 关闭后 `DisplayFiltersSelection` 禁用 module 相关选项 |
| `guest_view_all_features` | 关闭后 Guest 只能看自己创建的视图和 issue |

### 5.2 过滤器下拉选项的动态范围

`ProjectLevelWorkItemFiltersHOC`（`project-level.tsx`）注入项目上下文的 ID 集合：

```typescript
<WorkItemFiltersHOC
    cycleIds={getProjectCycleIds(projectId) ?? undefined}
    labelIds={getProjectLabelIds(projectId)}
    memberIds={getProjectMemberIds(projectId, false) ?? undefined}
    moduleIds={getProjectModuleIds(projectId) ?? undefined}
    stateIds={getProjectStateIds(projectId)}
>
```

过滤器下拉选项**随项目成员、状态、标签等动态变化**，用户只能选择项目中实际存在的值。

### 5.3 服务端字段白名单校验

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

未在 FilterSet 中声明的字段**禁止用于过滤**。

---

## 6. 视图迁移与版本兼容性

### 6.1 Legacy Filters → Rich Filters 迁移

系统经历了从扁平 `filters` 字典到结构化 `rich_filters` AST 表达式的重大迁移。

#### 转换器：LegacyToRichFiltersConverter

`apps/api/plane/utils/filters/converters.py` 实现完整转换：

**字段名映射**（旧 → 新）：
```
state      → state_id
labels     → label_id
cycle      → cycle_id
module     → module_id
assignees  → assignee_id
created_by → created_by_id
priority   → priority
state_group→ state_group
start_date → start_date
target_date→ target_date
```

**输出格式**：
- 单个条件 → `{field__operator: value}`
- 多个条件 → `{"and": [{field__operator: value}, ...]}`

#### 数据迁移脚本

`apps/api/plane/db/migrations/0107_migrate_filters_to_rich_filters.py` 迁移 5 个模型：

```python
MODEL_NAMES = [
    "IssueView",
    "WorkspaceUserProperties",
    "ModuleUserProperties",
    "IssueUserProperty",
    "CycleUserProperties",
]
```

`strict=False` 模式下，无效值会被静默跳过，保证迁移不会因脏数据中断。

### 6.2 模型兼容性设计

`IssueView` 模型同时保留新旧字段：

```python
filters = models.JSONField(default=dict)           # 旧版
rich_filters = models.JSONField(default=dict)       # 新版
display_filters = models.JSONField(default=get_default_display_filters)
display_properties = models.JSONField(default=get_default_display_properties)
```

`save()` 方法仍从 `filters` 派生 `query`（兼容旧 API）。

---

## 7. 关键发现与建议

### 7.1 发现的设计问题

1. **`access` 只读导致 CE 版无法创建私有视图**：
   - 序列化器将 `access` 标记为只读，但模型默认值是 Public。
   - CE 版的 `AccessController` 是空壳，UI 上也无法选择。
   - 这导致 CE 版所有视图都是公开的，与产品宣传可能存在差异。

2. **`is_locked` 字段有校验无设置**：
   - `partial_update` 中检查了 `is_locked`，但没有端点可以设置它。
   - 该字段在 CE 版实际上永远是 `False`。

3. **`updateView` 乐观更新无回滚机制**：
   - 先更新本地 Store，再发送 API 请求。
   - 如果 API 失败，本地 Store 已经变更，可能导致状态不一致。

4. **前端发送 `access` 但后端忽略**：
   - 前端表单明确发送 `access` 字段，但被序列化器丢弃。
   - 这会造成前端开发者困惑，以为可以设置访问级别。

### 7.2 建议

1. **如果 CE 版应支持私有视图**：
   - 从 `read_only_fields` 中移除 `access`。
   - 在 `create` 方法中校验 `access` 值只能是 0 或 1。
   - 实现 CE 版的 `AccessController` 组件。

2. **如果 `is_locked` 是 EE 功能**：
   - 应在代码中明确标注为 EE-only。
   - 或提供明确的 feature flag 控制。

3. **完善 `updateView` 的失败处理**：
   - 在 API 失败时回滚本地 Store 的变更。

4. **前后端字段一致性**：
   - 要么在前端移除对 `access` 的发送，要么在后端支持该字段。
   - 保持前后端契约一致，减少调试困惑。

---

## 8. 关键代码位置索引

| 文件 | 说明 |
|------|------|
| `apps/api/plane/db/models/view.py:58-98` | IssueView 模型定义 |
| `apps/api/plane/app/serializers/view.py:56-86` | 视图序列化器（含 read_only_fields） |
| `apps/api/plane/app/views/view/base.py:343-363` | partial_update 权限检查 |
| `apps/api/plane/app/views/view/base.py:260-261` | perform_create 注入逻辑 |
| `apps/web/ce/components/views/access-controller.tsx` | CE 版空壳 AccessController |
| `apps/web/core/components/views/form.tsx:92-106` | 前端表单提交逻辑 |
| `apps/web/core/store/project-view.store.ts:224-239` | updateView 乐观更新 |
| `apps/web/core/store/issue/project-views/filter.store.ts:146-174` | mutateFilters 回放逻辑 |
| `apps/web/core/store/issue/project-views/filter.store.ts:339-343` | resetFilters 重置逻辑 |
| `apps/api/plane/utils/filters/converters.py` | Legacy → Rich 过滤转换器 |
| `apps/api/plane/db/migrations/0107_migrate_filters_to_rich_filters.py` | 数据迁移脚本 |
