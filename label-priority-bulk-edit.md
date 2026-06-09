# Plane Issue 标签与优先级批量编辑流程分析

## 概述

本文基于对仓库全量代码的逐一核实，梳理 Plane 中批量修改 Issue 标签（labels）和优先级（priority）的完整代码路径。核心结论：**仓库中不存在 `bulk-operation-issues` 后端端点的任何实现**——前端预留了调用入口和 Store 方法，但后端 API、路由注册、权限校验、Schema 验证、数据库写入、活动记录和通知推送均缺失。文档首先明确缺失边界，再以单条更新路径和已有批量端点为参照，整理各环节的实际代码证据。

---

## 一、`@/plane-web` 别名指向

### 1.1 tsconfig.json 路径映射

[tsconfig.json](file:///d:/fz/0508-3/solo-dogfeeding/code/195-plane/apps/web/tsconfig.json#L4-L11) 定义了路径别名：

```json
{
  "paths": {
    "@/*": ["./core/*"],
    "@/plane-web/*": ["./ce/*"],
    "package.json": ["./package.json"]
  }
}
```

**`@/plane-web/*` 指向 `./ce/*`（即 `apps/web/ce/`）**，不是"付费版"目录。`ce/` 是社区版（Community Edition）的可替换实现层，与 `core/` 共同组成完整的 web 应用。

### 1.2 `ce/` 目录的角色

`ce/` 目录存放社区版的**可替换实现**，`core/` 中的代码通过 `@/plane-web/*` 别名导入这些实现。这种设计允许 Plane One 付费版通过替换 `ce/` 为自己的目录来提供增强功能。当前仓库只包含 `ce/` 实现。

与批量操作相关的 `ce/` 文件：
- [ce/hooks/use-bulk-operation-status.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/195-plane/apps/web/ce/hooks/use-bulk-operation-status.ts) — 始终返回 `false`
- [ce/components/issues/bulk-operations/root.tsx](file:///d:/fz/0508-3/solo-dogfeeding/code/195-plane/apps/web/ce/components/issues/bulk-operations/root.tsx) — 渲染升级横幅

`ce/` 目录**不是付费版目录**，它就是当前仓库运行时实际使用的代码。付费版的替换目录不在本仓库中。

---

## 二、布局使用分析：IssueBulkOperationsRoot 与 useBulkOperationStatus

### 2.1 三个布局均使用了这两个符号

| 布局 | IssueBulkOperationsRoot 位置 | useBulkOperationStatus 位置 | disabled 控制方式 |
|------|---------------------------|---------------------------|-----------------|
| **Spreadsheet** | [spreadsheet-view.tsx#L124](file:///d:/fz/0508-3/solo-dogfeeding/code/195-plane/apps/web/core/components/issues/issue-layouts/spreadsheet/spreadsheet-view.tsx#L124) | [spreadsheet-view.tsx#L68](file:///d:/fz/0508-3/solo-dogfeeding/code/195-plane/apps/web/core/components/issues/issue-layouts/spreadsheet/spreadsheet-view.tsx#L68) | `disabled={!isBulkOperationsEnabled \|\| isEpic}` 传给 `MultipleSelectGroup` |
| **List** | [default.tsx#L175](file:///d:/fz/0508-3/solo-dogfeeding/code/195-plane/apps/web/core/components/issues/issue-layouts/list/default.tsx#L175) | [default.tsx#L86](file:///d:/fz/0508-3/solo-dogfeeding/code/195-plane/apps/web/core/components/issues/issue-layouts/list/default.tsx#L86) | `disabled={!isBulkOperationsEnabled \|\| isEpic}` 传给 `MultipleSelectGroup` |
| **Gantt** | [main-content.tsx#L241](file:///d:/fz/0508-3/solo-dogfeeding/code/195-plane/apps/web/core/components/gantt-chart/chart/main-content.tsx#L241) | [main-content.tsx#L98](file:///d:/fz/0508-3/solo-dogfeeding/code/195-plane/apps/web/core/components/gantt-chart/chart/main-content.tsx#L98) + [base-gantt-root.tsx#L62](file:///d:/fz/0508-3/solo-dogfeeding/code/195-plane/apps/web/core/components/issues/issue-layouts/gantt/base-gantt-root.tsx#L62) | `disabled={!isBulkOperationsEnabled \|\| isEpic}` (main-content) + `enableSelection={isBulkOperationsEnabled && isAllowed}` (base-gantt-root) |

### 2.2 导入路径

三个布局文件中的导入均使用 `@/plane-web/` 别名：

```ts
import { IssueBulkOperationsRoot } from "@/plane-web/components/issues/bulk-operations";
import { useBulkOperationStatus } from "@/plane-web/hooks/use-bulk-operation-status";
```

运行时解析到 `ce/components/issues/bulk-operations/` 和 `ce/hooks/use-bulk-operation-status`。

### 2.3 disabled 的影响

当 `isBulkOperationsEnabled = false`（当前 CE 版的行为）：
- **Spreadsheet / List**：`MultipleSelectGroup` 的 `disabled=true`，`useMultipleSelect` hook 不注册键盘/鼠标选择事件，用户无法选中 issue
- **Gantt**：`enableSelection=false`，甘特图不渲染选择复选框；同时 `MultipleSelectGroup` 的 `disabled=true` 阻止批量操作面板

当 `isBulkOperationsEnabled = true`（Plane One 版的行为）：
- 用户可以选中 issue，底部渲染 `IssueBulkOperationsRoot` 面板
- Plane One 版的 `IssueBulkOperationsRoot` 实现不在本仓库中

---

## 三、后端路由全面排查

### 3.1 四个 API 路由前缀

[urls.py](file:///d:/fz/0508-3/solo-dogfeeding/code/195-plane/apps/api/plane/urls.py#L17-L24) 定义了四个路由前缀：

```python
path("api/", include("plane.app.urls"))           # 主 API（Django ViewSet）
path("api/public/", include("plane.space.urls"))   # 公开 API（Space/公开访问）
path("api/instances/", include("plane.license.urls"))  # 实例管理 API
path("api/v1/", include("plane.api.urls"))         # REST API v1（外部集成）
```

### 3.2 逐前缀排查结果

#### `api/` — `plane.app.urls`

[urls/issue.py](file:///d:/fz/0508-3/solo-dogfeeding/code/195-plane/apps/api/plane/app/urls/issue.py) 中的批量端点（完整列表）：

| URL 路径 | View | 功能 | 是否涉及标签/优先级批量修改 |
|---------|------|------|--------------------------|
| `bulk-create-labels/` | BulkCreateIssueLabelsEndpoint | 批量创建 Label 实体本身 | ❌ 不是给 Issue 分配标签 |
| `bulk-delete-issues/` | BulkDeleteIssuesEndpoint | 批量删除 Issue | ❌ 删除操作，非属性修改 |
| `bulk-archive-issues/` | BulkArchiveIssuesEndpoint | 批量归档 Issue | ❌ 归档操作，非属性修改 |
| `issue-dates/` | IssueBulkUpdateDateEndpoint | 批量更新 Issue 日期 | ❌ 仅处理 start_date/target_date |

**排除 `bulk-operation-issues/` 的理由**：
1. [urls/issue.py](file:///d:/fz/0508-3/solo-dogfeeding/code/195-plane/apps/api/plane/app/urls/issue.py) 完整列出了 286 行路由定义，不包含 `bulk-operation-issues` 路径
2. `from plane.app.views import (...)` 导入列表中无任何含 `BulkOperation` 或 `bulk_operation` 的 View 类
3. 全 `plane/app/views/` 目录 `.py` 文件搜索 `BulkOperation` 零匹配

其他子模块（cycle、module、project、state 等）的路由中也不含标签/优先级批量操作端点，理由：这些子模块不负责 Issue 属性的批量修改。

#### `api/public/` — `plane.space.urls`

[urls/issue.py (space)](file:///d:/fz/0508-3/solo-dogfeeding/code/195-plane/apps/api/plane/space/urls/issue.py) 仅包含公开访问端点：
- `IssueRetrievePublicEndpoint` — 单条 issue 查看
- `IssueCommentPublicViewSet` — 评论 CRUD
- `IssueReactionPublicViewSet` / `CommentReactionPublicViewSet` — 反应
- `IssueVotePublicViewSet` — 投票

**排除理由**：`api/public/` 是只读的公开访问接口，URL 前缀为 `anchor/<str:anchor>/`，不支持写入操作，无批量端点。

#### `api/instances/` — `plane.license.urls`

[urls.py (license)](file:///d:/fz/0508-3/solo-dogfeeding/code/195-plane/apps/api/plane/license/urls.py) 仅包含实例管理端点：
- `InstanceEndpoint` / `InstanceAdminEndpoint` — 实例配置与管理员
- `InstanceConfigurationEndpoint` — 配置项
- `InstanceWorkSpaceEndpoint` — 工作空间可用性检查
- `EmailCredentialCheckEndpoint` — 邮件凭证检查

**排除理由**：`api/instances/` 仅处理 Plane 实例的注册和配置管理，不涉及任何业务数据操作，无 Issue 相关端点。

#### `api/v1/` — `plane.api.urls`

[urls/work_item.py](file:///d:/fz/0508-3/solo-dogfeeding/code/195-plane/apps/api/plane/api/urls/work_item.py) 包含旧版（`issues/`）和新版（`work-items/`）端点：
- `IssueListCreateAPIEndpoint` — 列表/创建
- `IssueDetailAPIEndpoint` — 详情/修改/删除
- `IssueLinkListCreateAPIEndpoint` / `IssueCommentListCreateAPIEndpoint` — 子资源
- `IssueActivityListAPIEndpoint` — 活动记录（只读）
- `IssueRelationListCreateAPIEndpoint` — 关联

[urls/label.py (api/v1)](file:///d:/fz/0508-3/solo-dogfeeding/code/195-plane/apps/api/plane/api/urls/label.py) 仅包含：
- `LabelListCreateAPIEndpoint` — 标签列表/创建
- `LabelDetailAPIEndpoint` — 标签详情/修改/删除

**排除理由**：`api/v1/` 是面向外部集成的 REST API，只提供单条 CRUD 操作，无任何批量操作端点。全 `plane/api/views/` 搜索 `bulk` 只匹配到 `bulk_create`（ORM 方法调用，非端点）和模块/周期的批量分配（不含标签/优先级）。

### 3.3 排查结论

| 路由前缀 | 标签批量操作端点 | 优先级批量操作端点 | 通用批量属性端点 | 排除依据 |
|---------|---------------|-----------------|---------------|---------|
| `api/` | ❌ 不存在 | ❌ 不存在 | ❌ `bulk-operation-issues/` 不存在 | 完整阅读 urls/issue.py 286 行，无此路由 |
| `api/public/` | ❌ 不存在 | ❌ 不存在 | ❌ 不存在 | 公开只读接口，无写入端点 |
| `api/instances/` | ❌ 不存在 | ❌ 不存在 | ❌ 不存在 | 实例管理接口，无业务数据 |
| `api/v1/` | ❌ 不存在 | ❌ 不存在 | ❌ 不存在 | 外部集成 API，仅单条 CRUD |

---

## 四、缺失边界确认：`bulk-operation-issues` 端点不存在

### 4.1 前端引用

前端 [IssueService.bulkOperations](file:///d:/fz/0508-3/solo-dogfeeding/code/195-plane/apps/web/core/services/issue/issue.service.ts#L339-L345) 向以下 URL 发送 POST 请求：

```ts
async bulkOperations(workspaceSlug: string, projectId: string, data: TBulkOperationsPayload): Promise<any> {
  return this.post(`/api/workspaces/${workspaceSlug}/projects/${projectId}/bulk-operation-issues/`, data)
    .then(async (response) => response?.data)
    .catch((error) => { throw error?.response?.data; });
}
```

### 4.2 后端搜索结果

| 搜索范围 | 搜索模式 | 结果 |
|---------|---------|------|
| `apps/api/` 全部 `.py` 文件 | `bulk.operation.issues` | 0 匹配 |
| `apps/api/` 全部 `.py` 文件 | `BulkOperation` | 0 匹配 |
| `apps/api/` 全部 `.py` 文件 | `bulk_operation` | 0 匹配 |
| 全仓库 `.py` 文件 | `bulk-operation-issues` | 0 匹配 |

### 4.3 前端 Store 方法的调用者搜索

[bulkUpdateProperties](file:///d:/fz/0508-3/solo-dogfeeding/code/195-plane/apps/web/core/store/issue/helpers/base-issues.store.ts#L721-L753) 在 8 个 store 中被声明为接口方法或继承自 `BaseIssuesStore`：

| Store | 位置 |
|-------|------|
| ProjectIssueStore | [project/issue.store.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/195-plane/apps/web/core/store/issue/project/issue.store.ts#L52) |
| CycleIssueStore | [cycle/issue.store.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/195-plane/apps/web/core/store/issue/cycle/issue.store.ts#L91) |
| ModuleIssueStore | [module/issue.store.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/195-plane/apps/web/core/store/issue/module/issue.store.ts#L61) |
| ProfileIssueStore | [profile/issue.store.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/195-plane/apps/web/core/store/issue/profile/issue.store.ts#L59) |
| WorkspaceIssueStore | [workspace/issue.store.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/195-plane/apps/web/core/store/issue/workspace/issue.store.ts#L52) |
| ProjectViewsIssueStore | [project-views/issue.store.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/195-plane/apps/web/core/store/issue/project-views/issue.store.ts#L54) |
| WorkspaceDraftIssueStore | [workspace-draft/issue.store.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/195-plane/apps/web/core/store/issue/workspace-draft/issue.store.ts#L112) |
| ArchivedIssueStore | [archived/issue.store.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/195-plane/apps/web/core/store/issue/archived/issue.store.ts#L41) |

**全仓库搜索 `.bulkUpdateProperties(` 的调用者结果为零**——该方法已实现但无组件调用。

### 4.4 社区版前端行为

- [useBulkOperationStatus](file:///d:/fz/0508-3/solo-dogfeeding/code/195-plane/apps/web/ce/hooks/use-bulk-operation-status.ts) 始终返回 `false`
- [IssueBulkOperationsRoot (CE)](file:///d:/fz/0508-3/solo-dogfeeding/code/195-plane/apps/web/ce/components/issues/bulk-operations/root.tsx) 仅渲染 [BulkOperationsUpgradeBanner](file:///d:/fz/0508-3/solo-dogfeeding/code/195-plane/apps/web/core/components/issues/bulk-operations/upgrade-banner.tsx)，展示"Upgrade to One"按钮

---

## 五、前端：已有的批量编辑代码结构

虽然后端端点不存在，但前端已实现了选择、组装请求和本地状态更新的代码。

### 5.1 Issue 集合选择 — MultipleSelectStore

[MultipleSelectStore](file:///d:/fz/0508-3/solo-dogfeeding/code/195-plane/apps/web/core/store/multiple_select.store.ts) 管理选择状态：

```ts
selectedEntityDetails: TEntityDetails[] = [];    // 选中实体列表
lastSelectedEntityDetails: TEntityDetails | null; // 上次选中项（用于 Shift 范围选择）
activeEntityDetails: TEntityDetails | null;       // 当前活跃项
```

核心方法：
- `updateSelectedEntityDetails(entity, "add"|"remove")` — 单个增删
- `bulkUpdateSelectedEntityDetails(list, "add"|"remove")` — 批量增删，用 `differenceWith` 做差集
- `clearSelection()` — 清空

`selectedEntityIds` 是 computed 属性，映射出 ID 列表。

### 5.2 选择交互 — useMultipleSelect

[useMultipleSelect](file:///d:/fz/0508-3/solo-dogfeeding/code/195-plane/apps/web/core/hooks/use-multiple-select.ts) 封装键盘/鼠标交互：
- **Shift + 点击**：范围选择
- **Shift + 方向键**：逐个扩展
- **组头复选框**：`isGroupSelected()` 返回 `"empty"` | `"partial"` | `"complete"`

### 5.3 选择容器 — MultipleSelectGroup

[MultipleSelectGroup](file:///d:/fz/0508-3/solo-dogfeeding/code/195-plane/apps/web/core/components/core/multiple-select/select-group.tsx) 接收 `entities: { [groupID]: string[] }` 和 `disabled`：

```tsx
<MultipleSelectGroup
  entities={{ [SPREADSHEET_SELECT_GROUP]: issueIds }}
  disabled={!isBulkOperationsEnabled || isEpic}
>
  {(helpers) => (
    <>
      <SpreadsheetTable selectionHelpers={helpers} />
      <IssueBulkOperationsRoot selectionHelpers={helpers} />
    </>
  )}
</MultipleSelectGroup>
```

### 5.4 批量操作载荷类型

[TBulkOperationsPayload](file:///d:/fz/0508-3/solo-dogfeeding/code/195-plane/packages/types/src/issues/issues/issue.ts#L153-L156)：

```ts
type TBulkOperationsPayload = {
  issue_ids: string[];
  properties: Partial<TBulkIssueProperties>;
};
```

`TBulkIssueProperties` 包含 `state_id`、`priority`、`label_ids`、`assignee_ids`、`start_date`、`target_date`、`module_ids`、`cycle_id`、`estimate_point`。

### 5.5 Store 层 — bulkUpdateProperties 实现

[bulkUpdateProperties](file:///d:/fz/0508-3/solo-dogfeeding/code/195-plane/apps/web/core/store/issue/helpers/base-issues.store.ts#L721-L753)（完整代码）：

```ts
bulkUpdateProperties = async (workspaceSlug: string, projectId: string, data: TBulkOperationsPayload) => {
  const issueIds = data.issue_ids;
  await this.issueService.bulkOperations(workspaceSlug, projectId, data);
  runInAction(() => {
    issueIds.forEach((issueId) => {
      const issueBeforeUpdate = clone(this.rootIssueStore.issues.getIssueById(issueId));
      if (!issueBeforeUpdate) throw new Error("Work item not found");
      Object.keys(data.properties).forEach((key) => {
        const property = key as keyof TBulkOperationsPayload["properties"];
        const propertyValue = data.properties[property];
        if (Array.isArray(propertyValue)) {
          const existingValue = issueBeforeUpdate[property];
          const newExistingValue = Array.isArray(existingValue) ? existingValue : [];
          this.rootIssueStore.issues.updateIssue(issueId, {
            [property]: uniq([...newExistingValue, ...propertyValue]),
          });
        } else {
          this.rootIssueStore.issues.updateIssue(issueId, {
            [property]: propertyValue,
          });
        }
      });
      const issueDetails = this.rootIssueStore.issues.getIssueById(issueId);
      this.updateIssueList(issueDetails, issueBeforeUpdate);
    });
  });
};
```

**标签语义差异**：
- **前端 store 更新 `label_ids`**：追加语义 `uniq([...existing, ...new])`
- **后端单条更新 `label_ids`**（见第六节）：替换语义，先 `IssueLabel.objects.filter(issue=instance).delete()` 再 `bulk_create`

### 5.6 MobX 本地更新机制

[IssueStore.updateIssue](file:///d:/fz/0508-3/solo-dogfeeding/code/195-plane/apps/web/core/store/issue/issue.store.ts#L108-L116)：

```ts
updateIssue = (issueId: string, issue: Partial<TIssue>) => {
  if (!issue || !issueId || !this.issuesMap[issueId]) return;
  runInAction(() => {
    set(this.issuesMap, [issueId, "updated_at"], getCurrentDateTimeInISO());
    Object.keys(issue).forEach((key) => {
      set(this.issuesMap, [issueId, key], issue[key as keyof TIssue]);
    });
  });
};
```

`issuesMap` 是唯一数据源，所有依赖组件通过 MobX 自动重渲染。

---

## 六、服务端：单条更新路径的直接证据链

由于 `bulk-operation-issues` 端点不存在，以下以 [IssueViewSet.partial_update](file:///d:/fz/0508-3/solo-dogfeeding/code/195-plane/apps/api/plane/app/views/issue/base.py#L615-L702) 为参照，逐一追踪权限、Schema、DB 写入、通知、事务和并发处理的实际代码。**这些是单条更新的证据，不是批量操作的实现**。

### 6.1 权限验证

```python
@allow_permission(allowed_roles=[ROLE.ADMIN, ROLE.MEMBER], creator=True, model=Issue)
def partial_update(self, request, slug, project_id, pk=None):
```

[allow_permission](file:///d:/fz/0508-3/solo-dogfeeding/code/195-plane/apps/api/plane/app/permissions/base.py#L19-L87) 执行流程：

1. **creator 检查**（第 24-38 行）：如果 `creator=True`，先查 `WorkspaceMember` 确认用户是 workspace 成员，再查 `model.objects.filter(id=kwargs["pk"], created_by=request.user)` 验证是否为创建者
2. **角色检查**（第 44-78 行）：查 `ProjectMember` 验证用户在项目中角色 ≥ `allowed_roles`；若用户不是项目成员但有 workspace ADMIN 角色，也允许
3. 失败返回 `403 {"error": "You don't have the required permissions."}`

### 6.2 数据获取与快照

[partial_update](file:///d:/fz/0508-3/solo-dogfeeding/code/195-plane/apps/api/plane/app/views/issue/base.py#L623-L665) 用 `ArrayAgg` 注解获取 `label_ids`、`assignee_ids`、`module_ids` 的当前值，用于活动记录对比：

```python
issue = (
    queryset.annotate(
        label_ids=Coalesce(
            ArrayAgg("labels__id", distinct=True, filter=Q(...)),
            Value([], output_field=ArrayField(UUIDField())),
        ),
        ...
    )
    .filter(pk=pk)
    .first()
)
current_instance = json.dumps(IssueDetailSerializer(issue).data, cls=DjangoJSONEncoder)
```

### 6.3 Schema 验证 — IssueCreateSerializer.validate()

[IssueCreateSerializer.validate()](file:///d:/fz/0508-3/solo-dogfeeding/code/195-plane/apps/api/plane/app/serializers/issue.py#L123-L196)：

**标签验证**（第 158-165 行）：
```python
if attrs.get("label_ids"):
    label_ids = [label.id for label in attrs["label_ids"]]
    attrs["label_ids"] = list(
        Label.objects.filter(
            project_id=self.context.get("project_id"),
            id__in=label_ids,
        ).values_list("id", flat=True)
    )
```
- 静默过滤不属于当前项目的标签，不报错
- `label_ids` 字段定义（第 90-94 行）为 `ListField(child=PrimaryKeyRelatedField(queryset=Label.objects.all()))`，DRF 先验证每个 ID 是否存在于 `Label` 表

**优先级验证**：
- `priority` 是 [Issue 模型](file:///d:/fz/0508-3/solo-dogfeeding/code/195-plane/apps/api/plane/db/models/issue.py#L141-L146) 的 `CharField(max_length=30, choices=PRIORITY_CHOICES)`
- `PRIORITY_CHOICES`（第 107-113 行）：`("urgent","Urgent"), ("high","High"), ("medium","Medium"), ("low","Low"), ("none","None")`
- Serializer 的 `fields = "__all__"` 包含 `priority`，DRF 自动验证 choices 约束
- 非法值返回 `400 BadRequest`

### 6.4 数据库写入 — IssueCreateSerializer.update()

[IssueCreateSerializer.update()](file:///d:/fz/0508-3/solo-dogfeeding/code/195-plane/apps/api/plane/app/serializers/issue.py#L275-L329)：

**标签写入**（第 306-325 行）：
```python
if labels is not None:
    IssueLabel.objects.filter(issue=instance).delete()
    try:
        IssueLabel.objects.bulk_create(
            [IssueLabel(label_id=label_id, issue=instance, ...) for label_id in labels],
            batch_size=10,
            ignore_conflicts=True,
        )
    except IntegrityError:
        pass
```

**优先级写入**：`priority` 是 `Issue` 模型的标量字段，由 `super().update(instance, validated_data)` 直接写入。

**事务边界**：
- ❌ 没有 `transaction.atomic()` — 全仓库搜索 `apps/api/plane/app/views/issue/` 和 `apps/api/plane/app/serializers/issue.py` 中 `transaction` 和 `atomic` 均为零匹配
- ⚠️ 标签的"先删后建"不在事务中 — 如果 `bulk_create` 失败（且 `IntegrityError` 未被捕获），issue 的标签已被删除但新标签未创建
- `ignore_conflicts=True` 可避免重复插入，但也会静默跳过某些行

### 6.5 活动记录 — Celery 异步任务

[partial_update](file:///d:/fz/0508-3/solo-dogfeeding/code/195-plane/apps/api/plane/app/views/issue/base.py#L675-L685) 在 `serializer.save()` 成功后触发：

```python
issue_activity.delay(
    type="issue.activity.updated",
    requested_data=requested_data,
    current_instance=current_instance,
    actor_id=str(request.user.id),
    issue_id=str(pk),
    project_id=str(project_id),
    epoch=int(timezone.now().timestamp()),
    notification=True,
)
```

[issue_activity](file:///d:/fz/0508-3/solo-dogfeeding/code/195-plane/apps/api/plane/bgtasks/issue_activities_task.py#L1504-L1604) 的执行流程：

1. 根据 `type="issue.activity.updated"` 映射到 `update_issue_activity`（第 594 行）
2. `update_issue_activity` 的 `ISSUE_ACTIVITY_MAPPER`（第 604-622 行）包含：
   - `"priority"` → [track_priority](file:///d:/fz/0508-3/solo-dogfeeding/code/195-plane/apps/api/plane/bgtasks/issue_activities_task.py#L161-L185)
   - `"label_ids"` → [track_labels](file:///d:/fz/0508-3/solo-dogfeeding/code/195-plane/apps/api/plane/bgtasks/issue_activities_task.py#L290-L353)
3. `IssueActivity.objects.bulk_create(issue_activities)` 写入活动记录
4. 如果 `notification=True`，触发 `notifications.delay()`

活动记录与主数据不在同一事务中——`serializer.save()` 同步完成后，`issue_activity.delay()` 是异步的。

### 6.6 通知订阅者

[notifications](file:///d:/fz/0508-3/solo-dogfeeding/code/195-plane/apps/api/plane/bgtasks/notification_task.py#L191-L319)：

1. 获取 `IssueSubscriber` 列表，排除操作者和新提及用户
2. 对每个订阅者查 `UserNotificationPreference`
3. 创建 `Notification` 记录（`bulk_create`），根据偏好决定是否发邮件

通知是二级异步任务（`notifications.delay()` 由 `issue_activity` 内部触发），失败不影响主数据或活动记录。

### 6.7 并发冲突处理

**后端**：
- ❌ 没有 `select_for_update()` 行级锁
- ❌ 没有乐观锁（无 `version` 字段，不基于 `updated_at` 做条件更新）
- ❌ `partial_update` 不检查 `If-Match` / `ETag`

**前端**：
- MobX `issuesMap` 是单数据源，但只反映本客户端最后一次 API 响应
- 不存在冲突检测或合并机制

---

## 七、已有批量端点的实现模式（参照分析）

仓库中存在四个批量端点，以下是它们的实际代码特征。

### 7.1 BulkDeleteIssuesEndpoint

[BulkDeleteIssuesEndpoint](file:///d:/fz/0508-3/solo-dogfeeding/code/195-plane/apps/api/plane/app/views/issue/base.py#L761-L785)：

```python
class BulkDeleteIssuesEndpoint(BaseAPIView):
    @allow_permission([ROLE.ADMIN])
    def delete(self, request, slug, project_id):
        issue_ids = request.data.get("issue_ids", [])
        issues = Issue.issue_objects.filter(workspace__slug=slug, project_id=project_id, pk__in=issue_ids)
        total_issues = len(issues)
        CycleIssue.objects.filter(issue_id__in=issue_ids).delete()
        ModuleIssue.objects.filter(issue_id__in=issue_ids).delete()
        issues.delete()
        return Response({"message": f"{total_issues} issues were deleted"}, status=status.HTTP_200_OK)
```

- ❌ 无 `transaction.atomic()`
- ❌ 无逐条权限验证
- 删除前先清理关联表，若中间失败，关联已删除而 issue 未删除

### 7.2 BulkArchiveIssuesEndpoint

[BulkArchiveIssuesEndpoint](file:///d:/fz/0508-3/solo-dogfeeding/code/195-plane/apps/api/plane/app/views/issue/archive.py#L305-L342)：

- ❌ 无 `transaction.atomic()`
- ⚠️ 全量失败语义：任一 issue 状态不合法整批 400，但 `issue_activity.delay()` 已对前面 issue 触发
- ✅ 逐条触发活动记录

### 7.3 BulkCreateIssueLabelsEndpoint

[BulkCreateIssueLabelsEndpoint](file:///d:/fz/0508-3/solo-dogfeeding/code/195-plane/apps/api/plane/app/views/issue/label.py#L90-L117)：

此端点批量创建 **Label 实体本身**，不是给 Issue 分配标签。

### 7.4 IssueBulkUpdateDateEndpoint

[IssueBulkUpdateDateEndpoint](file:///d:/fz/0508-3/solo-dogfeeding/code/195-plane/apps/api/plane/app/views/issue/base.py#L1114-L1171)：

- ❌ 无 `transaction.atomic()`
- ⚠️ 逐条跳过不存在的 issue，前端无法感知
- ⚠️ `issue_activity.delay()` 在 `bulk_update()` 之前调用
- ❌ 无并发冲突处理

### 7.5 参照模式汇总

| 端点 | 事务 | 部分成功 | 活动记录 | 并发处理 |
|------|------|---------|---------|---------|
| BulkDeleteIssues | ❌ | ❌ 全量 | ❌ 无 | ❌ |
| BulkArchiveIssues | ❌ | ❌ 全量失败 | ✅ 逐条 | ❌ |
| BulkCreateIssueLabels | ❌ | ❌ ignore_conflicts | ❌ 无 | ❌ |
| IssueBulkUpdateDate | ❌ | ⚠️ 静默跳过 | ✅ 逐条 | ❌ |

**注意**：以上模式是现有端点的实际代码特征，不能直接推断 `bulk-operation-issues` 端点会遵循相同模式，因为该端点的实现不在本仓库中。

---

## 八、前端：三种更新策略对比

### 8.1 单条 issueUpdate — 乐观更新 + try-catch 回滚

[issueUpdate](file:///d:/fz/0508-3/solo-dogfeeding/code/195-plane/apps/web/core/store/issue/helpers/base-issues.store.ts#L554-L588)：

```ts
async issueUpdate(workspaceSlug, projectId, issueId, data, shouldSync = true) {
  const issueBeforeUpdate = clone(this.rootIssueStore.issues.getIssueById(issueId));
  try {
    this.rootIssueStore.issues.updateIssue(issueId, data);       // 先更新 store
    this.updateIssueList({ ...issueBeforeUpdate, ...data }, issueBeforeUpdate);
    if (shouldSync) {
      await this.issueService.patchIssue(workspaceSlug, projectId, issueId, data);
    }
  } catch (error) {
    this.rootIssueStore.issues.updateIssue(issueId, issueBeforeUpdate ?? {});       // 回滚
    throw error;
  }
}
```

### 8.2 批量 bulkUpdateProperties — 先请求后更新

[bulkUpdateProperties](file:///d:/fz/0508-3/solo-dogfeeding/code/195-plane/apps/web/core/store/issue/helpers/base-issues.store.ts#L721-L753)：

```ts
bulkUpdateProperties = async (workspaceSlug, projectId, data) => {
  const issueIds = data.issue_ids;
  await this.issueService.bulkOperations(workspaceSlug, projectId, data);  // 先请求
  runInAction(() => {
    issueIds.forEach((issueId) => {
      const issueBeforeUpdate = clone(this.rootIssueStore.issues.getIssueById(issueId));
      if (!issueBeforeUpdate) throw new Error("Work item not found");      // ← 中断风险
      // ... 更新属性 ...
    });
  });
};
```

**问题**：
1. API 失败时，本地 store 未变更，无需回滚
2. `throw new Error("Work item not found")` 在 `forEach` 内抛出，会中断后续 issue 的本地更新
3. 如果后端部分成功，前端无法区分

### 8.3 日期 updateIssueDates — 乐观更新 + 手动快照回滚

[updateIssueDates](file:///d:/fz/0508-3/solo-dogfeeding/code/195-plane/apps/web/core/store/issue/helpers/base-issues.store.ts#L755-L798)：

```ts
async updateIssueDates(workspaceSlug, updates, projectId) {
  const issueDatesBeforeChange = [];
  try {
    runInAction(() => {
      for (const update of updates) {
        this.issueUpdate(workspaceSlug, projectId, update.id, dates, false);
        issueDatesBeforeChange.push({ id: update.id, start_date: ..., target_date: ... });
      }
    });
    await this.issueService.updateIssueDates(workspaceSlug, projectId, updates);
  } catch (e) {
    runInAction(() => {
      for (const update of issueDatesBeforeChange) {
        this.issueUpdate(workspaceSlug, projectId, update.id, dates, false);  // 回滚
      }
    });
    throw e;
  }
}
```

### 8.4 对比表

| 方法 | 更新时机 | 回滚机制 | API 端点存在 |
|------|---------|---------|------------|
| `issueUpdate` | 先 store 后 API | try-catch 回滚 | ✅ `PATCH /issues/:id/` |
| `bulkUpdateProperties` | 先 API 后 store | 无（store 未变） | ❌ `POST /bulk-operation-issues/` 不存在 |
| `updateIssueDates` | 先 store 后 API | 手动快照回滚 | ✅ `POST /issue-dates/` |

---

## 九、直接证据链汇总

### 9.1 权限

| 环节 | 证据 | 位置 |
|------|------|------|
| 批量操作端点权限 | ❌ 端点不存在 | — |
| 单条更新权限 | ✅ `@allow_permission([ADMIN, MEMBER], creator=True, model=Issue)` | [base.py#L615](file:///d:/fz/0508-3/solo-dogfeeding/code/195-plane/apps/api/plane/app/views/issue/base.py#L615) |
| 权限装饰器实现 | ✅ 查 WorkspaceMember + ProjectMember + creator 检查 | [base.py#L19-L87](file:///d:/fz/0508-3/solo-dogfeeding/code/195-plane/apps/api/plane/app/permissions/base.py#L19-L87) |
| 已有批量端点权限 | ✅ BulkDelete: ADMIN; BulkArchive: ADMIN+MEMBER; BulkUpdateDate: ADMIN+MEMBER | [base.py#L762](file:///d:/fz/0508-3/solo-dogfeeding/code/195-plane/apps/api/plane/app/views/issue/base.py#L762), [archive.py#L308](file:///d:/fz/0508-3/solo-dogfeeding/code/195-plane/apps/api/plane/app/views/issue/archive.py#L308), [base.py#L1114](file:///d:/fz/0508-3/solo-dogfeeding/code/195-plane/apps/api/plane/app/views/issue/base.py#L1114) |

### 9.2 Schema 验证

| 环节 | 证据 | 位置 |
|------|------|------|
| 批量操作 Schema | ❌ 端点不存在 | — |
| 标签验证 | ✅ 静默过滤非项目标签 | [issue.py#L158-L165](file:///d:/fz/0508-3/solo-dogfeeding/code/195-plane/apps/api/plane/app/serializers/issue.py#L158-L165) |
| 优先级验证 | ✅ DRF choices 约束 | [models/issue.py#L107-L113](file:///d:/fz/0508-3/solo-dogfeeding/code/195-plane/apps/api/plane/db/models/issue.py#L107-L113) |

### 9.3 数据库写入

| 环节 | 证据 | 位置 |
|------|------|------|
| 批量操作 DB 写入 | ❌ 端点不存在 | — |
| 标签替换写入 | ✅ `delete()` + `bulk_create(batch_size=10, ignore_conflicts=True)` | [issue.py#L306-L325](file:///d:/fz/0508-3/solo-dogfeeding/code/195-plane/apps/api/plane/app/serializers/issue.py#L306-L325) |
| IntegrityError 静默 | ✅ `except IntegrityError: pass` | [issue.py#L323-L324](file:///d:/fz/0508-3/solo-dogfeeding/code/195-plane/apps/api/plane/app/serializers/issue.py#L323-L324) |
| 优先级写入 | ✅ 标量字段，`super().update()` 一行 SQL | [issue.py#L327-L329](file:///d:/fz/0508-3/solo-dogfeeding/code/195-plane/apps/api/plane/app/serializers/issue.py#L327-L329) |
| 已有批量写入 | ✅ `Issue.objects.bulk_update(issues, ["start_date","target_date"])` | [base.py#L1169](file:///d:/fz/0508-3/solo-dogfeeding/code/195-plane/apps/api/plane/app/views/issue/base.py#L1169) |

### 9.4 通知订阅

| 环节 | 证据 | 位置 |
|------|------|------|
| 批量操作通知 | ❌ 端点不存在 | — |
| 活动记录触发 | ✅ `issue_activity.delay(notification=True)` | [base.py#L675-L685](file:///d:/fz/0508-3/solo-dogfeeding/code/195-plane/apps/api/plane/app/views/issue/base.py#L675-L685) |
| priority 活动追踪 | ✅ `track_priority` 对比新旧值 | [issue_activities_task.py#L161-L185](file:///d:/fz/0508-3/solo-dogfeeding/code/195-plane/apps/api/plane/bgtasks/issue_activities_task.py#L161-L185) |
| label_ids 活动追踪 | ✅ `track_labels` 计算集合差 | [issue_activities_task.py#L290-L353](file:///d:/fz/0508-3/solo-dogfeeding/code/195-plane/apps/api/plane/bgtasks/issue_activities_task.py#L290-L353) |
| 通知推送 | ✅ `notifications.delay()` 查订阅者+偏好 | [notification_task.py#L191-L319](file:///d:/fz/0508-3/solo-dogfeeding/code/195-plane/apps/api/plane/bgtasks/notification_task.py#L191-L319) |
| 活动与数据不同事务 | ✅ Celery 异步，独立事务 | [issue_activities_task.py#L1584-L1599](file:///d:/fz/0508-3/solo-dogfeeding/code/195-plane/apps/api/plane/bgtasks/issue_activities_task.py#L1584-L1599) |

### 9.5 事务回滚

| 环节 | 证据 | 位置 |
|------|------|------|
| 批量操作事务 | ❌ 端点不存在 | — |
| 单条更新事务 | ❌ `transaction`/`atomic` 在 views/issue/ 和 serializers/issue.py 中零匹配 | — |
| 标签先删后建 | ⚠️ 不在事务中，delete 后 bulk_create 失败会导致标签丢失 | [issue.py#L306-L325](file:///d:/fz/0508-3/solo-dogfeeding/code/195-plane/apps/api/plane/app/serializers/issue.py#L306-L325) |
| 已有批量端点事务 | ❌ BulkDelete/BulkArchive/BulkUpdateDate 均无 `transaction.atomic()` | — |

### 9.6 部分成功

| 环节 | 证据 | 位置 |
|------|------|------|
| 批量操作部分成功 | ❌ 端点不存在 | — |
| BulkUpdateDate 静默跳过 | ✅ `if not issue: continue` | [base.py#L1130-L1131](file:///d:/fz/0508-3/solo-dogfeeding/code/195-plane/apps/api/plane/app/views/issue/base.py#L1130-L1131) |
| BulkArchive 全量失败 | ⚠️ 但活动记录已触发 | [archive.py#L320-L327](file:///d:/fz/0508-3/solo-dogfeeding/code/195-plane/apps/api/plane/app/views/issue/archive.py#L320-L327) |
| 标签验证静默过滤 | ✅ 不属于项目的标签被静默丢弃 | [issue.py#L158-L165](file:///d:/fz/0508-3/solo-dogfeeding/code/195-plane/apps/api/plane/app/serializers/issue.py#L158-L165) |
| 前端 forEach 中 throw | ⚠️ `throw new Error("Work item not found")` 中断后续更新 | [base-issues.store.ts#L729](file:///d:/fz/0508-3/solo-dogfeeding/code/195-plane/apps/web/core/store/issue/helpers/base-issues.store.ts#L729) |

### 9.7 并发冲突

| 环节 | 证据 | 位置 |
|------|------|------|
| 行级锁 | ❌ `select_for_update` 在所有 issue 相关 view 中零匹配 | — |
| 乐观锁 | ❌ 无 version 字段，无 `updated_at` 条件更新 | — |
| ETag/If-Match | ❌ 无 | — |
| 前端冲突检测 | ❌ 无版本号比对 | — |

---

## 十、结论

### 10.1 当前状态

批量修改 Issue 标签和优先级的完整功能在本仓库中不可用：

1. **后端 API 不存在**：四个 API 路由前缀（`api/`、`api/public/`、`api/instances/`、`api/v1/`）均无 `bulk-operation-issues` 或等效端点
2. **前端代码是预留骨架**：`bulkUpdateProperties` 方法已实现但无组件调用；三个布局（List、Gantt、Spreadsheet）通过 `@/plane-web/` 别名导入 `useBulkOperationStatus`，当前 `ce/` 实现返回 `false`，禁用选择功能
3. **Plane One 的实现不在本仓库中**：`ce/` 是社区版可替换层，Plane One 付费版通过替换 `ce/` 目录提供增强的 `useBulkOperationStatus`（返回 `true`）和 `IssueBulkOperationsRoot`（实际操作面板），但替换实现不在本仓库中

### 10.2 基于现有代码的观察

基于单条更新路径和已有批量端点的实际代码：

1. **事务边界**：单条更新的标签"先删后建"和已有批量端点均无 `transaction.atomic()`
2. **部分成功**：现有批量端点模式为静默跳过或全量失败
3. **语义一致性**：前端 store 的标签追加语义与后端 Serializer 的替换语义不同
4. **活动记录时序**：BulkArchiveIssuesEndpoint 中 `issue_activity.delay()` 在 `bulk_update()` 之前触发
5. **并发冲突**：全仓库无乐观锁或行级锁机制
6. **前端循环中断**：`bulkUpdateProperties` 中 `throw new Error` 会中断 `forEach` 后续项

---

## 十一、代码索引

| 组件 | 文件路径 |
|------|---------|
| tsconfig 路径别名 | [tsconfig.json](file:///d:/fz/0508-3/solo-dogfeeding/code/195-plane/apps/web/tsconfig.json#L4-L11) |
| CE 批量操作开关 | [use-bulk-operation-status.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/195-plane/apps/web/ce/hooks/use-bulk-operation-status.ts) |
| CE 批量操作根组件 | [root.tsx (CE)](file:///d:/fz/0508-3/solo-dogfeeding/code/195-plane/apps/web/ce/components/issues/bulk-operations/root.tsx) |
| 升级提示横幅 | [upgrade-banner.tsx](file:///d:/fz/0508-3/solo-dogfeeding/code/195-plane/apps/web/core/components/issues/bulk-operations/upgrade-banner.tsx) |
| Spreadsheet 布局 | [spreadsheet-view.tsx](file:///d:/fz/0508-3/solo-dogfeeding/code/195-plane/apps/web/core/components/issues/issue-layouts/spreadsheet/spreadsheet-view.tsx#L68) |
| List 布局 | [default.tsx](file:///d:/fz/0508-3/solo-dogfeeding/code/195-plane/apps/web/core/components/issues/issue-layouts/list/default.tsx#L86) |
| Gantt 布局 (main-content) | [main-content.tsx](file:///d:/fz/0508-3/solo-dogfeeding/code/195-plane/apps/web/core/components/gantt-chart/chart/main-content.tsx#L98) |
| Gantt 布局 (base-root) | [base-gantt-root.tsx](file:///d:/fz/0508-3/solo-dogfeeding/code/195-plane/apps/web/core/components/issues/issue-layouts/gantt/base-gantt-root.tsx#L62) |
| 前端批量操作服务 | [issue.service.ts#L339-L345](file:///d:/fz/0508-3/solo-dogfeeding/code/195-plane/apps/web/core/services/issue/issue.service.ts#L339-L345) |
| 前端批量 Store 方法 | [base-issues.store.ts#L721-L753](file:///d:/fz/0508-3/solo-dogfeeding/code/195-plane/apps/web/core/store/issue/helpers/base-issues.store.ts#L721-L753) |
| 前端 Store 更新方法 | [issue.store.ts#L108-L116](file:///d:/fz/0508-3/solo-dogfeeding/code/195-plane/apps/web/core/store/issue/issue.store.ts#L108-L116) |
| 批量操作载荷类型 | [issue.ts#L153-L156](file:///d:/fz/0508-3/solo-dogfeeding/code/195-plane/packages/types/src/issues/issues/issue.ts#L153-L156) |
| 选择状态管理 | [multiple_select.store.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/195-plane/apps/web/core/store/multiple_select.store.ts) |
| 根 URL 配置 | [urls.py](file:///d:/fz/0508-3/solo-dogfeeding/code/195-plane/apps/api/plane/urls.py#L17-L24) |
| api/ 路由汇总 | [urls/__init__.py](file:///d:/fz/0508-3/solo-dogfeeding/code/195-plane/apps/api/plane/app/urls/__init__.py) |
| api/ Issue 路由 | [urls/issue.py](file:///d:/fz/0508-3/solo-dogfeeding/code/195-plane/apps/api/plane/app/urls/issue.py) |
| api/public/ Issue 路由 | [urls/issue.py (space)](file:///d:/fz/0508-3/solo-dogfeeding/code/195-plane/apps/api/plane/space/urls/issue.py) |
| api/instances/ 路由 | [urls.py (license)](file:///d:/fz/0508-3/solo-dogfeeding/code/195-plane/apps/api/plane/license/urls.py) |
| api/v1/ 路由汇总 | [urls/__init__.py (api)](file:///d:/fz/0508-3/solo-dogfeeding/code/195-plane/apps/api/plane/api/urls/__init__.py) |
| api/v1/ Work Item 路由 | [urls/work_item.py](file:///d:/fz/0508-3/solo-dogfeeding/code/195-plane/apps/api/plane/api/urls/work_item.py) |
| api/v1/ Label 路由 | [urls/label.py](file:///d:/fz/0508-3/solo-dogfeeding/code/195-plane/apps/api/plane/api/urls/label.py) |
| 单条更新视图 | [base.py#L615-L702](file:///d:/fz/0508-3/solo-dogfeeding/code/195-plane/apps/api/plane/app/views/issue/base.py#L615-L702) |
| 权限装饰器 | [base.py#L19-L87](file:///d:/fz/0508-3/solo-dogfeeding/code/195-plane/apps/api/plane/app/permissions/base.py#L19-L87) |
| Issue Serializer | [issue.py#L82-L329](file:///d:/fz/0508-3/solo-dogfeeding/code/195-plane/apps/api/plane/app/serializers/issue.py#L82-L329) |
| Issue 模型（priority choices） | [models/issue.py#L107-L113](file:///d:/fz/0508-3/solo-dogfeeding/code/195-plane/apps/api/plane/db/models/issue.py#L107-L113) |
| BulkDeleteIssues | [base.py#L761-L785](file:///d:/fz/0508-3/solo-dogfeeding/code/195-plane/apps/api/plane/app/views/issue/base.py#L761-L785) |
| BulkArchiveIssues | [archive.py#L305-L342](file:///d:/fz/0508-3/solo-dogfeeding/code/195-plane/apps/api/plane/app/views/issue/archive.py#L305-L342) |
| BulkCreateIssueLabels | [label.py#L90-L117](file:///d:/fz/0508-3/solo-dogfeeding/code/195-plane/apps/api/plane/app/views/issue/label.py#L90-L117) |
| IssueBulkUpdateDate | [base.py#L1114-L1171](file:///d:/fz/0508-3/solo-dogfeeding/code/195-plane/apps/api/plane/app/views/issue/base.py#L1114-L1171) |
| 活动记录任务 | [issue_activities_task.py#L1504-L1604](file:///d:/fz/0508-3/solo-dogfeeding/code/195-plane/apps/api/plane/bgtasks/issue_activities_task.py#L1504-L1604) |
| priority 活动追踪 | [issue_activities_task.py#L161-L185](file:///d:/fz/0508-3/solo-dogfeeding/code/195-plane/apps/api/plane/bgtasks/issue_activities_task.py#L161-L185) |
| labels 活动追踪 | [issue_activities_task.py#L290-L353](file:///d:/fz/0508-3/solo-dogfeeding/code/195-plane/apps/api/plane/bgtasks/issue_activities_task.py#L290-L353) |
| 活动映射表 | [issue_activities_task.py#L604-L622](file:///d:/fz/0508-3/solo-dogfeeding/code/195-plane/apps/api/plane/bgtasks/issue_activities_task.py#L604-L622) |
| 通知任务 | [notification_task.py#L191-L319](file:///d:/fz/0508-3/solo-dogfeeding/code/195-plane/apps/api/plane/bgtasks/notification_task.py#L191-L319) |
