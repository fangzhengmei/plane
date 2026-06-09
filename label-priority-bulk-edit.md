# Plane Issue 标签与优先级批量编辑流程分析

## 概述

本文基于对仓库全量代码的逐一核实，梳理 Plane 中批量修改 Issue 标签（labels）和优先级（priority）的完整代码路径。核心结论：**仓库中不存在 `bulk-operation-issues` 后端端点的任何实现**——前端预留了调用入口和 Store 方法，但后端 API、路由注册、权限校验、Schema 验证、数据库写入、活动记录和通知推送均缺失。文档首先明确缺失边界，再以单条更新路径和已有批量端点为参照，整理各环节的实际代码证据。

---

## 一、`@/plane-web` 别名指向

### 1.1 tsconfig.json 路径映射

`apps/web/tsconfig.json` 定义了路径别名：

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
- `ce/hooks/use-bulk-operation-status.ts` — 始终返回 `false`
- `ce/components/issues/bulk-operations/root.tsx` — 渲染升级横幅

`ce/` 目录**不是付费版目录**，它就是当前仓库运行时实际使用的代码。付费版的替换目录不在本仓库中。

---

## 二、布局使用分析：IssueBulkOperationsRoot 与 useBulkOperationStatus

### 2.1 三个布局均使用了这两个符号

| 布局 | IssueBulkOperationsRoot 位置 | useBulkOperationStatus 位置 | disabled 控制方式 |
|------|---------------------------|---------------------------|-----------------|
| **Spreadsheet** | `core/components/issues/issue-layouts/spreadsheet/spreadsheet-view.tsx#L124` | `core/components/issues/issue-layouts/spreadsheet/spreadsheet-view.tsx#L68` | `disabled={!isBulkOperationsEnabled \|\| isEpic}` 传给 `MultipleSelectGroup` |
| **List** | `core/components/issues/issue-layouts/list/default.tsx#L175` | `core/components/issues/issue-layouts/list/default.tsx#L86` | `disabled={!isBulkOperationsEnabled \|\| isEpic}` 传给 `MultipleSelectGroup` |
| **Gantt** | `core/components/gantt-chart/chart/main-content.tsx#L241` | `core/components/gantt-chart/chart/main-content.tsx#L98` + `core/components/issues/issue-layouts/gantt/base-gantt-root.tsx#L62` | `disabled` (main-content) + `enableSelection={isBulkOperationsEnabled && isAllowed}` (base-gantt-root) |

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

`apps/api/plane/urls.py` 定义了四个路由前缀：

```python
path("api/", include("plane.app.urls"))           # 主 API（Django ViewSet）
path("api/public/", include("plane.space.urls"))   # 公开 API（Space/公开访问）
path("api/instances/", include("plane.license.urls"))  # 实例管理 API
path("api/v1/", include("plane.api.urls"))         # REST API v1（外部集成）
```

### 3.2 逐前缀排查结果

#### `api/` — `plane.app.urls`

`apps/api/plane/app/urls/issue.py` 中的批量端点（完整列表）：

| URL 路径 | View | 功能 | 是否涉及标签/优先级批量修改 |
|---------|------|------|--------------------------|
| `bulk-create-labels/` | BulkCreateIssueLabelsEndpoint | 批量创建 Label 实体本身 | ❌ 不是给 Issue 分配标签 |
| `bulk-delete-issues/` | BulkDeleteIssuesEndpoint | 批量删除 Issue | ❌ 删除操作，非属性修改 |
| `bulk-archive-issues/` | BulkArchiveIssuesEndpoint | 批量归档 Issue | ❌ 归档操作，非属性修改 |
| `issue-dates/` | IssueBulkUpdateDateEndpoint | 批量更新 Issue 日期 | ❌ 仅处理 start_date/target_date |

**排除 `bulk-operation-issues/` 的理由**：
1. `apps/api/plane/app/urls/issue.py` 完整列出了 286 行路由定义，不包含 `bulk-operation-issues` 路径
2. `from plane.app.views import (...)` 导入列表中无任何含 `BulkOperation` 或 `bulk_operation` 的 View 类
3. 全 `plane/app/views/` 目录 `.py` 文件搜索 `BulkOperation` 零匹配

其他子模块（cycle、module、project、state 等）的路由中也不含标签/优先级批量操作端点，理由：这些子模块不负责 Issue 属性的批量修改。

#### `api/public/` — `plane.space.urls`

`apps/api/plane/space/urls/issue.py` 包含以下端点：

| URL 路径 | View | HTTP 方法 | 说明 |
|---------|------|---------|------|
| `anchor/<anchor>/issues/<id>/` | IssueRetrievePublicEndpoint | GET | 单条 issue 查看 |
| `anchor/<anchor>/issues/<id>/comments/` | IssueCommentPublicViewSet | GET, POST | 评论列表与创建 |
| `anchor/<anchor>/issues/<id>/comments/<pk>/` | IssueCommentPublicViewSet | GET, PATCH, DELETE | 评论详情/修改/删除 |
| `anchor/<anchor>/issues/<id>/reactions/` | IssueReactionPublicViewSet | GET, POST | 反应列表与创建 |
| `anchor/<anchor>/issues/<id>/reactions/<code>/` | IssueReactionPublicViewSet | DELETE | 删除反应 |
| `anchor/<anchor>/comments/<id>/reactions/` | CommentReactionPublicViewSet | GET, POST | 评论反应列表与创建 |
| `anchor/<anchor>/comments/<id>/reactions/<code>/` | CommentReactionPublicViewSet | DELETE | 删除评论反应 |
| `anchor/<anchor>/issues/<id>/votes/` | IssueVotePublicViewSet | GET, POST, DELETE | 投票列表/创建/删除 |

**修正**：`api/public/` 并非纯只读接口——IssueCommentPublicViewSet 具有 `create`、`partial_update`、`destroy` 能力，IssueReactionPublicViewSet 具有 `create`、`destroy` 能力，IssueVotePublicViewSet 具有 `create`、`destroy` 能力。

**排除理由**：虽然 `api/public/` 支持 comments、reactions、votes 的写入操作，但这些是围绕 issue 的子资源（评论、反应、投票）的单条 CRUD，不是标签或优先级的批量修改端点。没有任何端点接受 `issue_ids` 列表和 `properties` 字段来批量修改 issue 属性。

#### `api/instances/` — `plane.license.urls`

`apps/api/plane/license/urls.py` 仅包含实例管理端点：
- `InstanceEndpoint` / `InstanceAdminEndpoint` — 实例配置与管理员
- `InstanceConfigurationEndpoint` — 配置项
- `InstanceWorkSpaceEndpoint` — 工作空间可用性检查
- `EmailCredentialCheckEndpoint` — 邮件凭证检查

**排除理由**：`api/instances/` 仅处理 Plane 实例的注册和配置管理，不涉及任何业务数据操作，无 Issue 相关端点。

#### `api/v1/` — `plane.api.urls`

`apps/api/plane/api/urls/work_item.py` 包含旧版（`issues/`）和新版（`work-items/`）端点：
- `IssueListCreateAPIEndpoint` — 列表/创建
- `IssueDetailAPIEndpoint` — 详情/修改/删除
- `IssueLinkListCreateAPIEndpoint` / `IssueCommentListCreateAPIEndpoint` — 子资源
- `IssueActivityListAPIEndpoint` — 活动记录（只读）
- `IssueRelationListCreateAPIEndpoint` — 关联

`apps/api/plane/api/urls/label.py` 仅包含：
- `LabelListCreateAPIEndpoint` — 标签列表/创建
- `LabelDetailAPIEndpoint` — 标签详情/修改/删除

**排除理由**：`api/v1/` 是面向外部集成的 REST API，只提供单条 CRUD 操作，无任何批量操作端点。全 `plane/api/views/` 搜索 `bulk` 只匹配到 `bulk_create`（ORM 方法调用，非端点）和模块/周期的批量分配（不含标签/优先级）。

### 3.3 排查结论

| 路由前缀 | 标签批量操作端点 | 优先级批量操作端点 | 通用批量属性端点 | 排除依据 |
|---------|---------------|-----------------|---------------|---------|
| `api/` | ❌ 不存在 | ❌ 不存在 | ❌ `bulk-operation-issues/` 不存在 | 完整阅读 urls/issue.py 286 行，无此路由 |
| `api/public/` | ❌ 不存在 | ❌ 不存在 | ❌ 不存在 | 有 comments/reactions/votes 写入能力，但不涉及标签/优先级批量修改 |
| `api/instances/` | ❌ 不存在 | ❌ 不存在 | ❌ 不存在 | 实例管理接口，无业务数据 |
| `api/v1/` | ❌ 不存在 | ❌ 不存在 | ❌ 不存在 | 外部集成 API，仅单条 CRUD |

---

## 四、仓库中的 EE 扩展目录与批量操作的关系

### 4.1 editor 相关的 EE 扩展存根

`ce/` 目录中存在多个与 editor 相关的扩展存根，这些文件由 `core/` 通过 `@/plane-web/` 别名导入：

| CE 存根文件 | 功能 | 批量操作相关 |
|------------|------|------------|
| `ce/hooks/editor/use-extended-editor-config.ts` | 编辑器扩展配置（CE 版返回空配置） | ❌ 无关 |
| `ce/hooks/pages/use-extended-editor-extensions.ts` | 页面编辑器扩展属性（CE 版空实现） | ❌ 无关 |
| `ce/hooks/use-additional-editor-mention.tsx` | 编辑器高级提及功能（CE 版返回空） | ❌ 无关 |
| `ce/hooks/use-editor-flagging.ts` | 编辑器功能开关（CE 版返回默认值） | ❌ 无关 |
| `ce/hooks/use-file-size.ts` | 文件大小限制（CE 版读实例配置） | ❌ 无关 |
| `ce/hooks/use-issue-embed.tsx` | Issue 嵌入编辑器（CE 版渲染升级卡片） | ❌ 无关 |

`ce/types/pages/pane-extensions.ts` 中有注释明确标注：
```ts
// CE re-exports the core navigation pane extension types directly
// EE overrides this with specific extension data types
```

这证实了 `ce/` 是可替换层的设计意图——Plane One 付费版可以覆盖这些存根提供增强实现。但这些 editor 扩展存根与 Issue 批量操作无关。

### 4.2 批量操作相关的 CE 存根

与批量操作直接相关的 CE 存根只有两个：

| CE 存根文件 | 功能 | Plane One 需要覆盖的内容 |
|------------|------|------------------------|
| `ce/hooks/use-bulk-operation-status.ts` | 返回 `false`，禁用选择 | 返回 `true`，启用选择 |
| `ce/components/issues/bulk-operations/root.tsx` | 渲染升级横幅 | 渲染实际操作面板，调用 `bulkUpdateProperties` |

### 4.3 后端 API 的替换实现

前端通过 `@/plane-web/` 别名实现 `ce/` → Plane One 的替换，但后端没有类似的替换机制。`bulk-operation-issues` 端点需要由 Plane One 的后端服务独立实现。当前仓库中：
- 无 `plane-ee/` 或类似目录
- [settings/common.py](apps/api/plane/settings/common.py) 的 `INSTALLED_APPS` 无任何付费版 Django App
- `plane/license/` 模块仅处理实例注册和配置，不含业务 API

**结论**：Issue 批量操作的完整实现（前端操作面板 + 后端 API 端点）均不在本仓库中。

---

## 五、缺失边界确认：`bulk-operation-issues` 端点不存在

### 5.1 前端引用

前端 `core/services/issue/issue.service.ts#L339-L345` 向以下 URL 发送 POST 请求：

```ts
async bulkOperations(workspaceSlug: string, projectId: string, data: TBulkOperationsPayload): Promise<any> {
  return this.post(`/api/workspaces/${workspaceSlug}/projects/${projectId}/bulk-operation-issues/`, data)
    .then(async (response) => response?.data)
    .catch((error) => { throw error?.response?.data; });
}
```

### 5.2 后端搜索结果

| 搜索范围 | 搜索模式 | 结果 |
|---------|---------|------|
| `apps/api/` 全部 `.py` 文件 | `bulk.operation.issues` | 0 匹配 |
| `apps/api/` 全部 `.py` 文件 | `BulkOperation` | 0 匹配 |
| `apps/api/` 全部 `.py` 文件 | `bulk_operation` | 0 匹配 |
| 全仓库 `.py` 文件 | `bulk-operation-issues` | 0 匹配 |

### 5.3 前端 Store 方法的调用者搜索

`core/store/issue/helpers/base-issues.store.ts#L721-L753` 中的 `bulkUpdateProperties` 在 8 个 store 中被声明为接口方法或继承自 `BaseIssuesStore`：

| Store | 位置 |
|-------|------|
| ProjectIssueStore | `core/store/issue/project/issue.store.ts#L52` |
| CycleIssueStore | `core/store/issue/cycle/issue.store.ts#L91` |
| ModuleIssueStore | `core/store/issue/module/issue.store.ts#L61` |
| ProfileIssueStore | `core/store/issue/profile/issue.store.ts#L59` |
| WorkspaceIssueStore | `core/store/issue/workspace/issue.store.ts#L52` |
| ProjectViewsIssueStore | `core/store/issue/project-views/issue.store.ts#L54` |
| WorkspaceDraftIssueStore | `core/store/issue/workspace-draft/issue.store.ts#L112` |
| ArchivedIssueStore | `core/store/issue/archived/issue.store.ts#L41` |

**全仓库搜索 `.bulkUpdateProperties(` 的调用者结果为零**——该方法已实现但无组件调用。

### 5.4 社区版前端行为

- `ce/hooks/use-bulk-operation-status.ts` 始终返回 `false`
- `ce/components/issues/bulk-operations/root.tsx` 仅渲染 `core/components/issues/bulk-operations/upgrade-banner.tsx`，展示"Upgrade to One"按钮

---

## 六、前端：已有的批量编辑代码结构

虽然后端端点不存在，但前端已实现了选择、组装请求和本地状态更新的代码。

### 6.1 Issue 集合选择 — MultipleSelectStore

`core/store/multiple_select.store.ts` 管理选择状态：

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

### 6.2 选择交互 — useMultipleSelect

`core/hooks/use-multiple-select.ts` 封装键盘/鼠标交互：
- **Shift + 点击**：范围选择
- **Shift + 方向键**：逐个扩展
- **组头复选框**：`isGroupSelected()` 返回 `"empty"` | `"partial"` | `"complete"`

### 6.3 选择容器 — MultipleSelectGroup

`core/components/core/multiple-select/select-group.tsx` 接收 `entities: { [groupID]: string[] }` 和 `disabled`：

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

### 6.4 批量操作载荷类型

`packages/types/src/issues/issues/issue.ts#L153-L156`：

```ts
type TBulkOperationsPayload = {
  issue_ids: string[];
  properties: Partial<TBulkIssueProperties>;
};
```

`TBulkIssueProperties` 包含 `state_id`、`priority`、`label_ids`、`assignee_ids`、`start_date`、`target_date`、`module_ids`、`cycle_id`、`estimate_point`。

### 6.5 Store 层 — bulkUpdateProperties 实现

`core/store/issue/helpers/base-issues.store.ts#L721-L753`（完整代码）：

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
- **后端单条更新 `label_ids`**（见第七节）：替换语义，先 `IssueLabel.objects.filter(issue=instance).delete()` 再 `bulk_create`

### 6.6 MobX 本地更新机制

`core/store/issue/issue.store.ts#L108-L116`：

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

## 七、服务端：单条更新路径的直接证据链

由于 `bulk-operation-issues` 端点不存在，以下以 `apps/api/plane/app/views/issue/base.py#L615-L702` 中的 `IssueViewSet.partial_update` 为参照，逐一追踪权限、Schema、DB 写入、通知、事务和并发处理的实际代码。**这些是单条更新的证据，不是批量操作的实现**。

### 7.1 权限验证

```python
@allow_permission(allowed_roles=[ROLE.ADMIN, ROLE.MEMBER], creator=True, model=Issue)
def partial_update(self, request, slug, project_id, pk=None):
```

`apps/api/plane/app/permissions/base.py#L19-L87` 的 `allow_permission` 执行流程：

1. **creator 检查**（第 24-38 行）：如果 `creator=True`，先查 `WorkspaceMember` 确认用户是 workspace 成员，再查 `model.objects.filter(id=kwargs["pk"], created_by=request.user)` 验证是否为创建者
2. **角色检查**（第 44-78 行）：查 `ProjectMember` 验证用户在项目中角色 ≥ `allowed_roles`；若用户不是项目成员但有 workspace ADMIN 角色，也允许
3. 失败返回 `403 {"error": "You don't have the required permissions."}`

### 7.2 数据获取与快照

`partial_update` 用 `ArrayAgg` 注解获取 `label_ids`、`assignee_ids`、`module_ids` 的当前值，用于活动记录对比：

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

### 7.3 Schema 验证 — IssueCreateSerializer.validate()

`apps/api/plane/app/serializers/issue.py#L123-L196`：

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
- `priority` 是 `apps/api/plane/db/models/issue.py#L141-L146` 的 `CharField(max_length=30, choices=PRIORITY_CHOICES)`
- `PRIORITY_CHOICES`（第 107-113 行）：`("urgent","Urgent"), ("high","High"), ("medium","Medium"), ("low","Low"), ("none","None")`
- Serializer 的 `fields = "__all__"` 包含 `priority`，DRF 自动验证 choices 约束
- 非法值返回 `400 BadRequest`

### 7.4 数据库写入 — IssueCreateSerializer.update()

`apps/api/plane/app/serializers/issue.py#L275-L329`：

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

### 7.5 活动记录 — Celery 异步任务

`partial_update` 在 `serializer.save()` 成功后触发：

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

`apps/api/plane/bgtasks/issue_activities_task.py#L1504-L1604` 的执行流程：

1. 根据 `type="issue.activity.updated"` 映射到 `update_issue_activity`（第 594 行）
2. `update_issue_activity` 的 `ISSUE_ACTIVITY_MAPPER`（第 604-622 行）包含：
   - `"priority"` → `issue_activities_task.py#L161-L185` 的 `track_priority`
   - `"label_ids"` → `issue_activities_task.py#L290-L353` 的 `track_labels`
3. `IssueActivity.objects.bulk_create(issue_activities)` 写入活动记录
4. 如果 `notification=True`，触发 `notifications.delay()`

活动记录与主数据不在同一事务中——`serializer.save()` 同步完成后，`issue_activity.delay()` 是异步的。

### 7.6 通知订阅者

`apps/api/plane/bgtasks/notification_task.py#L191-L319`：

1. 获取 `IssueSubscriber` 列表，排除操作者和新提及用户
2. 对每个订阅者查 `UserNotificationPreference`
3. 创建 `Notification` 记录（`bulk_create`），根据偏好决定是否发邮件

通知是二级异步任务（`notifications.delay()` 由 `issue_activity` 内部触发），失败不影响主数据或活动记录。

### 7.7 并发冲突处理

**后端**：
- ❌ 没有 `select_for_update()` 行级锁
- ❌ 没有乐观锁（无 `version` 字段，不基于 `updated_at` 做条件更新）
- ❌ `partial_update` 不检查 `If-Match` / `ETag`

**前端**：
- MobX `issuesMap` 是单数据源，但只反映本客户端最后一次 API 响应
- 不存在冲突检测或合并机制

---

## 八、已有批量端点的实现模式（参照分析）

仓库中存在四个批量端点，以下是它们的实际代码特征。

### 8.1 BulkDeleteIssuesEndpoint

`apps/api/plane/app/views/issue/base.py#L761-L785`：

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

### 8.2 BulkArchiveIssuesEndpoint

`apps/api/plane/app/views/issue/archive.py#L305-L342`：

- ❌ 无 `transaction.atomic()`
- ⚠️ 全量失败语义：任一 issue 状态不合法整批 400，但 `issue_activity.delay()` 已对前面 issue 触发
- ✅ 逐条触发活动记录

### 8.3 BulkCreateIssueLabelsEndpoint

`apps/api/plane/app/views/issue/label.py#L90-L117`：

此端点批量创建 **Label 实体本身**，不是给 Issue 分配标签。

### 8.4 IssueBulkUpdateDateEndpoint

`apps/api/plane/app/views/issue/base.py#L1114-L1171`：

- ❌ 无 `transaction.atomic()`
- ⚠️ 逐条跳过不存在的 issue，前端无法感知
- ⚠️ `issue_activity.delay()` 在 `bulk_update()` 之前调用
- ❌ 无并发冲突处理

### 8.5 参照模式汇总

| 端点 | 事务 | 部分成功 | 活动记录 | 并发处理 |
|------|------|---------|---------|---------|
| BulkDeleteIssues | ❌ | ❌ 全量 | ❌ 无 | ❌ |
| BulkArchiveIssues | ❌ | ❌ 全量失败 | ✅ 逐条 | ❌ |
| BulkCreateIssueLabels | ❌ | ❌ ignore_conflicts | ❌ 无 | ❌ |
| IssueBulkUpdateDate | ❌ | ⚠️ 静默跳过 | ✅ 逐条 | ❌ |

**注意**：以上模式是现有端点的实际代码特征，不能直接推断 `bulk-operation-issues` 端点会遵循相同模式，因为该端点的实现不在本仓库中。

---

## 九、前端：三种更新策略对比

### 9.1 单条 issueUpdate — 乐观更新 + try-catch 回滚

`core/store/issue/helpers/base-issues.store.ts#L554-L588`：

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

### 9.2 批量 bulkUpdateProperties — 先请求后更新

`core/store/issue/helpers/base-issues.store.ts#L721-L753`：

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

### 9.3 日期 updateIssueDates — 乐观更新 + 手动快照回滚

`core/store/issue/helpers/base-issues.store.ts#L755-L798`：

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

### 9.4 对比表

| 方法 | 更新时机 | 回滚机制 | API 端点存在 |
|------|---------|---------|------------|
| `issueUpdate` | 先 store 后 API | try-catch 回滚 | ✅ `PATCH /issues/:id/` |
| `bulkUpdateProperties` | 先 API 后 store | 无（store 未变） | ❌ `POST /bulk-operation-issues/` 不存在 |
| `updateIssueDates` | 先 store 后 API | 手动快照回滚 | ✅ `POST /issue-dates/` |

---

## 十、直接证据链汇总

### 10.1 权限

| 环节 | 证据 | 位置 |
|------|------|------|
| 批量操作端点权限 | ❌ 端点不存在 | — |
| 单条更新权限 | ✅ `@allow_permission([ADMIN, MEMBER], creator=True, model=Issue)` | `app/views/issue/base.py#L615` |
| 权限装饰器实现 | ✅ 查 WorkspaceMember + ProjectMember + creator 检查 | `app/permissions/base.py#L19-L87` |
| 已有批量端点权限 | ✅ BulkDelete: ADMIN; BulkArchive: ADMIN+MEMBER; BulkUpdateDate: ADMIN+MEMBER | `app/views/issue/base.py#L762`, `app/views/issue/archive.py#L308`, `app/views/issue/base.py#L1114` |

### 10.2 Schema 验证

| 环节 | 证据 | 位置 |
|------|------|------|
| 批量操作 Schema | ❌ 端点不存在 | — |
| 标签验证 | ✅ 静默过滤非项目标签 | `app/serializers/issue.py#L158-L165` |
| 优先级验证 | ✅ DRF choices 约束 | `db/models/issue.py#L107-L113` |

### 10.3 数据库写入

| 环节 | 证据 | 位置 |
|------|------|------|
| 批量操作 DB 写入 | ❌ 端点不存在 | — |
| 标签替换写入 | ✅ `delete()` + `bulk_create(batch_size=10, ignore_conflicts=True)` | `app/serializers/issue.py#L306-L325` |
| IntegrityError 静默 | ✅ `except IntegrityError: pass` | `app/serializers/issue.py#L323-L324` |
| 优先级写入 | ✅ 标量字段，`super().update()` 一行 SQL | `app/serializers/issue.py#L327-L329` |
| 已有批量写入 | ✅ `Issue.objects.bulk_update(issues, ["start_date","target_date"])` | `app/views/issue/base.py#L1169` |

### 10.4 通知订阅

| 环节 | 证据 | 位置 |
|------|------|------|
| 批量操作通知 | ❌ 端点不存在 | — |
| 活动记录触发 | ✅ `issue_activity.delay(notification=True)` | `app/views/issue/base.py#L675-L685` |
| priority 活动追踪 | ✅ `track_priority` 对比新旧值 | `bgtasks/issue_activities_task.py#L161-L185` |
| label_ids 活动追踪 | ✅ `track_labels` 计算集合差 | `bgtasks/issue_activities_task.py#L290-L353` |
| 通知推送 | ✅ `notifications.delay()` 查订阅者+偏好 | `bgtasks/notification_task.py#L191-L319` |
| 活动与数据不同事务 | ✅ Celery 异步，独立事务 | `bgtasks/issue_activities_task.py#L1584-L1599` |

### 10.5 事务回滚

| 环节 | 证据 | 位置 |
|------|------|------|
| 批量操作事务 | ❌ 端点不存在 | — |
| 单条更新事务 | ❌ `transaction`/`atomic` 在 views/issue/ 和 serializers/issue.py 中零匹配 | — |
| 标签先删后建 | ⚠️ 不在事务中，delete 后 bulk_create 失败会导致标签丢失 | `app/serializers/issue.py#L306-L325` |
| 已有批量端点事务 | ❌ BulkDelete/BulkArchive/BulkUpdateDate 均无 `transaction.atomic()` | — |

### 10.6 部分成功

| 环节 | 证据 | 位置 |
|------|------|------|
| 批量操作部分成功 | ❌ 端点不存在 | — |
| BulkUpdateDate 静默跳过 | ✅ `if not issue: continue` | `app/views/issue/base.py#L1130-L1131` |
| BulkArchive 全量失败 | ⚠️ 但活动记录已触发 | `app/views/issue/archive.py#L320-L327` |
| 标签验证静默过滤 | ✅ 不属于项目的标签被静默丢弃 | `app/serializers/issue.py#L158-L165` |
| 前端 forEach 中 throw | ⚠️ `throw new Error("Work item not found")` 中断后续更新 | `core/store/issue/helpers/base-issues.store.ts#L729` |

### 10.7 并发冲突

| 环节 | 证据 | 位置 |
|------|------|------|
| 行级锁 | ❌ `select_for_update` 在所有 issue 相关 view 中零匹配 | — |
| 乐观锁 | ❌ 无 version 字段，无 `updated_at` 条件更新 | — |
| ETag/If-Match | ❌ 无 | — |
| 前端冲突检测 | ❌ 无版本号比对 | — |

---

## 十一、结论

### 11.1 当前状态

批量修改 Issue 标签和优先级的完整功能在本仓库中不可用：

1. **后端 API 不存在**：四个 API 路由前缀（`api/`、`api/public/`、`api/instances/`、`api/v1/`）均无 `bulk-operation-issues` 或等效端点
2. **前端代码是预留骨架**：`bulkUpdateProperties` 方法已实现但无组件调用；三个布局（List、Gantt、Spreadsheet）通过 `@/plane-web/` 别名导入 `useBulkOperationStatus`，当前 `ce/` 实现返回 `false`，禁用选择功能
3. **Plane One 的实现不在本仓库中**：`ce/` 是社区版可替换层，Plane One 付费版通过替换 `ce/` 目录提供增强的 `useBulkOperationStatus`（返回 `true`）和 `IssueBulkOperationsRoot`（实际操作面板），后端 API 端点也需要独立实现，但替换实现均不在本仓库中

### 11.2 基于现有代码的观察

基于单条更新路径和已有批量端点的实际代码：

1. **事务边界**：单条更新的标签"先删后建"和已有批量端点均无 `transaction.atomic()`
2. **部分成功**：现有批量端点模式为静默跳过或全量失败
3. **语义一致性**：前端 store 的标签追加语义与后端 Serializer 的替换语义不同
4. **活动记录时序**：BulkArchiveIssuesEndpoint 中 `issue_activity.delay()` 在 `bulk_update()` 之前触发
5. **并发冲突**：全仓库无乐观锁或行级锁机制
6. **前端循环中断**：`bulkUpdateProperties` 中 `throw new Error` 会中断 `forEach` 后续项

---

## 十二、代码索引

| 组件 | 文件路径 |
|------|---------|
| tsconfig 路径别名 | `apps/web/tsconfig.json` |
| CE 批量操作开关 | `apps/web/ce/hooks/use-bulk-operation-status.ts` |
| CE 批量操作根组件 | `apps/web/ce/components/issues/bulk-operations/root.tsx` |
| 升级提示横幅 | `apps/web/core/components/issues/bulk-operations/upgrade-banner.tsx` |
| CE editor 扩展配置存根 | `apps/web/ce/hooks/editor/use-extended-editor-config.ts` |
| CE editor 附加提及存根 | `apps/web/ce/hooks/use-additional-editor-mention.tsx` |
| CE editor flag 存根 | `apps/web/ce/hooks/use-editor-flagging.ts` |
| CE page 扩展编辑器存根 | `apps/web/ce/hooks/pages/use-extended-editor-extensions.ts` |
| CE pane 扩展类型 | `apps/web/ce/types/pages/pane-extensions.ts` |
| Spreadsheet 布局 | `apps/web/core/components/issues/issue-layouts/spreadsheet/spreadsheet-view.tsx#L68` |
| List 布局 | `apps/web/core/components/issues/issue-layouts/list/default.tsx#L86` |
| Gantt 布局 (main-content) | `apps/web/core/components/gantt-chart/chart/main-content.tsx#L98` |
| Gantt 布局 (base-root) | `apps/web/core/components/issues/issue-layouts/gantt/base-gantt-root.tsx#L62` |
| 前端批量操作服务 | `apps/web/core/services/issue/issue.service.ts#L339-L345` |
| 前端批量 Store 方法 | `apps/web/core/store/issue/helpers/base-issues.store.ts#L721-L753` |
| 前端 Store 更新方法 | `apps/web/core/store/issue/issue.store.ts#L108-L116` |
| 批量操作载荷类型 | `packages/types/src/issues/issues/issue.ts#L153-L156` |
| 选择状态管理 | `apps/web/core/store/multiple_select.store.ts` |
| 根 URL 配置 | `apps/api/plane/urls.py` |
| api/ 路由汇总 | `apps/api/plane/app/urls/__init__.py` |
| api/ Issue 路由 | `apps/api/plane/app/urls/issue.py` |
| api/public/ Issue 路由 | `apps/api/plane/space/urls/issue.py` |
| api/instances/ 路由 | `apps/api/plane/license/urls.py` |
| api/v1/ 路由汇总 | `apps/api/plane/api/urls/__init__.py` |
| api/v1/ Work Item 路由 | `apps/api/plane/api/urls/work_item.py` |
| api/v1/ Label 路由 | `apps/api/plane/api/urls/label.py` |
| 单条更新视图 | `apps/api/plane/app/views/issue/base.py#L615-L702` |
| 权限装饰器 | `apps/api/plane/app/permissions/base.py#L19-L87` |
| Issue Serializer | `apps/api/plane/app/serializers/issue.py#L82-L329` |
| Issue 模型（priority choices） | `apps/api/plane/db/models/issue.py#L107-L113` |
| BulkDeleteIssues | `apps/api/plane/app/views/issue/base.py#L761-L785` |
| BulkArchiveIssues | `apps/api/plane/app/views/issue/archive.py#L305-L342` |
| BulkCreateIssueLabels | `apps/api/plane/app/views/issue/label.py#L90-L117` |
| IssueBulkUpdateDate | `apps/api/plane/app/views/issue/base.py#L1114-L1171` |
| 活动记录任务 | `apps/api/plane/bgtasks/issue_activities_task.py#L1504-L1604` |
| priority 活动追踪 | `apps/api/plane/bgtasks/issue_activities_task.py#L161-L185` |
| labels 活动追踪 | `apps/api/plane/bgtasks/issue_activities_task.py#L290-L353` |
| 活动映射表 | `apps/api/plane/bgtasks/issue_activities_task.py#L604-L622` |
| 通知任务 | `apps/api/plane/bgtasks/notification_task.py#L191-L319` |
| Django INSTALLED_APPS | `apps/api/plane/settings/common.py` |
