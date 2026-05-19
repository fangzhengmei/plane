# Intake 到 Issue 转换链路分析 - Round 2

## 一、审核阶段权限边界深度分析

### 1.1 权限系统基础

**文件**: `apps/api/plane/app/permissions/base.py:13-17`

```python
class ROLE(Enum):
    ADMIN = 20
    MEMBER = 15
    GUEST = 5
```

角色权限值越大，权限越高：ADMIN(20) > MEMBER(15) > GUEST(5)

### 1.2 后端 API 权限控制矩阵

#### 1.2.1 创建 Intake Issue
**文件**: `apps/api/plane/app/views/intake/base.py:221`

```python
@allow_permission([ROLE.ADMIN, ROLE.MEMBER, ROLE.GUEST])
def create(self, request, slug, project_id):
```

| 角色 | 权限 | 说明 |
|------|------|------|
| ADMIN | ✅ 允许 | 项目管理员 |
| MEMBER | ✅ 允许 | 项目成员 |
| GUEST | ✅ 允许 | 项目访客 |
| 匿名 | ❌ 拒绝 | 需登录 |

#### 1.2.2 更新 Intake Issue（审核操作核心）
**文件**: `apps/api/plane/app/views/intake/base.py:327`

```python
@allow_permission(allowed_roles=[ROLE.ADMIN], creator=True, model=Issue)
def partial_update(self, request, slug, project_id, pk):
```

`@allow_permission` 装饰器参数说明：
- `allowed_roles=[ROLE.ADMIN]`: 只允许 ADMIN 角色
- `creator=True, model=Issue`: 创建者也有权限（即使不是 ADMIN）

**方法内二次权限校验** (340-367行):

```python
# 第一层校验：必须是项目成员或工作区管理员
if not project_member and not is_workspace_admin:
    return Response({"error": "Only admin or creator can update..."}, 403)

# 第二层校验：GUEST 角色只能编辑自己创建的
if ((project_member and project_member.role <= ROLE.GUEST.value) 
    and not is_workspace_admin) 
    and str(intake_issue.created_by_id) != str(request.user.id):
    return Response({"error": "You cannot edit intake issues"}, 400)

# 第三层校验：只有 MEMBER 以上角色才能更新 IntakeIssue 状态
if (project_member and project_member.role > ROLE.MEMBER.value) or is_workspace_admin:
    # 可以更新 status/duplicate_to/snoozed_till 等审核字段
    intake_serializer = IntakeIssueSerializer(intake_issue, data=request.data, partial=True)
```

#### 1.2.3 审核操作权限细分表

| 操作 | 后端 API 字段 | ADMIN | MEMBER | GUEST（创建者） | GUEST（非创建者） | 说明 |
|------|-------------|-------|--------|-----------------|-----------------|------|
| 修改 Issue 标题 | `issue.name` | ✅ | ✅ | ✅ | ❌ |  |
| 修改 Issue 描述 | `issue.description_*` | ✅ | ✅ | ✅ | ❌ |  |
| 修改 Issue 属性（优先级等） | `issue.*` | ✅ | ✅ | ❌ | ❌ | GUEST 被过滤只留 name/description |
| 接受（ACCEPTED） | `status=1` | ✅ | ❌ | ❌ | ❌ | 需 `role > MEMBER.value` (即 ADMIN) |
| 拒绝（REJECTED） | `status=-1` | ✅ | ❌ | ❌ | ❌ | 同上 |
| 暂停（SNOOZED） | `status=0` | ✅ | ❌ | ❌ | ❌ | 同上 |
| 标记重复（DUPLICATE） | `status=2, duplicate_to` | ✅ | ❌ | ❌ | ❌ | 同上 |
| 设为待审核（PENDING） | `status=-2` | ✅ | ❌ | ❌ | ❌ | 同上 |

**关键权限代码** (418行):
```python
if (project_member and project_member.role > ROLE.MEMBER.value) or is_workspace_admin:
    # 只有 >15 即 ADMIN(20) 才能走到这里更新 IntakeIssue 状态
    intake_serializer = IntakeIssueSerializer(intake_issue, data=request.data, partial=True)
```

> **重要发现**: MEMBER 角色（值=15）不满足 `> MEMBER.value` 条件，因此**不能执行任何状态变更操作**。只有 ADMIN（值=20）可以。

#### 1.2.4 删除 Intake Issue
**文件**: `apps/api/plane/app/views/intake/base.py:545`

```python
@allow_permission(allowed_roles=[ROLE.ADMIN], creator=True, model=Issue)
def destroy(self, request, slug, project_id, pk):
```

| 角色 | 权限 | 说明 |
|------|------|------|
| ADMIN | ✅ 允许 |  |
| MEMBER | ❌ 拒绝 | 装饰器只允许 ADMIN |
| GUEST（创建者） | ✅ 允许 | creator=True 绕过 |
| GUEST（非创建者） | ❌ 拒绝 |  |

### 1.3 前端 UI 权限控制

**文件**: `apps/web/core/components/inbox/content/inbox-issue-header.tsx:89-107`

```typescript
// 基础操作权限：ADMIN 或 MEMBER
const isAllowed = allowPermissions(
  [EUserPermissions.ADMIN, EUserPermissions.MEMBER],
  EUserPermissionsLevel.PROJECT,
  workspaceSlug,
  projectId
);

// 状态变更按钮可见性（仅 PENDING 或 SNOOZED 状态显示）
const canMarkAsDuplicate = isAllowed && (inboxIssue?.status === 0 || inboxIssue?.status === -2);
const canMarkAsAccepted = isAllowed && (inboxIssue?.status === 0 || inboxIssue?.status === -2);
const canMarkAsDeclined = isAllowed && (inboxIssue?.status === 0 || inboxIssue?.status === -2);

// 项目管理员权限（执行实际操作时二次校验）
const isProjectAdmin = allowPermissions(
  [EUserPermissions.ADMIN],
  EUserPermissionsLevel.PROJECT,
  workspaceSlug,
  projectId
);
```

**按钮点击时的权限校验** (332-338行):
```typescript
<Button
  onClick={() =>
    handleActionWithPermission(
      isProjectAdmin,  // 必须是项目管理员
      () => setAcceptIssueModal(true),
      t("inbox_issue.errors.accept_permission")
    )
  }
>
```

`handleActionWithPermission` 实现 (215-224行):
```typescript
const handleActionWithPermission = (isAdmin: boolean, action: () => void, errorMessage: string) => {
  if (isAdmin) action();
  else {
    setToast({
      type: TOAST_TYPE.ERROR,
      title: "Permission denied",
      message: errorMessage,
    });
  }
};
```

### 1.4 权限边界总结图

```
┌─────────────────────────────────────────────────────────────────┐
│                    Intake Issue 权限矩阵                         │
├──────────┬──────────┬──────────┬──────────┬────────────────────┤
│   操作   │  ADMIN   │  MEMBER  │  GUEST   │     备注            │
├──────────┼──────────┼──────────┼──────────┼────────────────────┤
│ 提交创建 │    ✅    │    ✅    │    ✅    │ 所有登录用户        │
├──────────┼──────────┼──────────┼──────────┼────────────────────┤
│ 编辑标题 │    ✅    │    ✅    │ 仅自己  │                    │
│ 编辑描述 │    ✅    │    ✅    │ 仅自己  │                    │
├──────────┼──────────┼──────────┼──────────┼────────────────────┤
│ 接受     │    ✅    │    ❌    │    ❌    │ 需 ADMIN 角色       │
│ 拒绝     │    ✅    │    ❌    │    ❌    │ 需 ADMIN 角色       │
│ 暂停     │    ✅    │    ❌    │    ❌    │ 需 ADMIN 角色       │
│ 标记重复 │    ✅    │    ❌    │    ❌    │ 需 ADMIN 角色       │
├──────────┼──────────┼──────────┼──────────┼────────────────────┤
│ 删除     │    ✅    │    ❌    │ 仅自己  │ creator 权限        │
└──────────┴──────────┴──────────┴──────────┴────────────────────┘
```

> **注意**: 前端 MEMBER 角色会看到 Accept/Decline 按钮，但点击时会被 `handleActionWithPermission` 拦截并提示权限不足。后端也有 `role > MEMBER.value` 的硬校验，双重保障。

---

## 二、公开入口路由映射深度分析

### 2.1 Django 根路由配置

**文件**: `apps/api/plane/urls.py:17-24`

```python
urlpatterns = [
    path("api/", include("plane.app.urls")),           # 内部 API
    path("api/public/", include("plane.space.urls")),   # 公开 API（重点！）
    path("api/instances/", include("plane.license.urls")),
    path("api/v1/", include("plane.api.urls")),
    path("auth/", include("plane.authentication.urls")),
    path("", include("plane.web.urls")),
]
```

### 2.2 公开 API 空间路由

**文件**: `apps/api/plane/space/urls/__init__.py:5-11`

```python
from .intake import urlpatterns as intake_urls
from .issue import urlpatterns as issue_urls
from .project import urlpatterns as project_urls
from .asset import urlpatterns as asset_urls

urlpatterns = [*intake_urls, *issue_urls, *project_urls, *asset_urls]
```

### 2.3 Intake 公开路由配置（核心发现）

**文件**: `apps/api/plane/space/urls/intake.py:14-34`

```python
urlpatterns = [
    # 路由1: intake-issues 前缀
    path(
        "anchor/<str:anchor>/intakes/<uuid:intake_id>/intake-issues/",
        IntakeIssuePublicViewSet.as_view({"get": "list", "post": "create"}),
        name="intake-issue",
    ),
    # 路由2: inbox-issues 前缀（别名！）
    path(
        "anchor/<str:anchor>/intakes/<uuid:intake_id>/inbox-issues/",
        IntakeIssuePublicViewSet.as_view({"get": "list", "post": "create"}),
        name="inbox-issue",
    ),
    # 详情路由（只有 intake-issues 前缀）
    path(
        "anchor/<str:anchor>/intakes/<uuid:intake_id>/intake-issues/<uuid:pk>/",
        IntakeIssuePublicViewSet.as_view({"get": "retrieve", "patch": "partial_update", "delete": "destroy"}),
        name="intake-issue",
    ),
    # ...
]
```

### 2.4 完整公开路由映射表

| HTTP 方法 | 完整 URL 路径 | 处理 ViewSet | 说明 |
|----------|-------------|-------------|------|
| GET | `/api/public/anchor/<anchor>/intakes/<intake_id>/intake-issues/` | `IntakeIssuePublicViewSet.list()` | 列出公开 Intake Issues |
| POST | `/api/public/anchor/<anchor>/intakes/<intake_id>/intake-issues/` | `IntakeIssuePublicViewSet.create()` | 提交公开 Intake Issue |
| GET | `/api/public/anchor/<anchor>/intakes/<intake_id>/inbox-issues/` | `IntakeIssuePublicViewSet.list()` | **同上，别名** |
| POST | `/api/public/anchor/<anchor>/intakes/<intake_id>/inbox-issues/` | `IntakeIssuePublicViewSet.create()` | **同上，别名** |
| GET | `/api/public/anchor/<anchor>/intakes/<intake_id>/intake-issues/<pk>/` | `IntakeIssuePublicViewSet.retrieve()` | 获取单个详情 |
| PATCH | `/api/public/anchor/<anchor>/intakes/<intake_id>/intake-issues/<pk>/` | `IntakeIssuePublicViewSet.partial_update()` | 更新（仅创建者） |
| DELETE | `/api/public/anchor/<anchor>/intakes/<intake_id>/intake-issues/<pk>/` | `IntakeIssuePublicViewSet.destroy()` | 删除（仅创建者） |

> **重要发现**: `intake-issues/` 和 `inbox-issues/` 在公开 API 中是**完全等价的别名**，都指向同一个 `IntakeIssuePublicViewSet` 的 `list` 和 `create` 方法。但详情路由（带 pk）只提供了 `intake-issues/` 前缀版本。

### 2.5 内部 API 路由对比（参考）

**文件**: `apps/api/plane/app/urls/intake.py`

| HTTP 方法 | 完整 URL 路径 | 处理 ViewSet |
|----------|-------------|-------------|
| POST | `/api/workspaces/<slug>/projects/<project_id>/intake-issues/` | `IntakeIssueViewSet.create()` |
| POST | `/api/workspaces/<slug>/projects/<project_id>/inbox-issues/` | `IntakeIssueViewSet.create()` |
| PATCH | `/api/workspaces/<slug>/projects/<project_id>/intake-issues/<pk>/` | `IntakeIssueViewSet.partial_update()` |
| PATCH | `/api/workspaces/<slug>/projects/<project_id>/inbox-issues/<pk>/` | `IntakeIssueViewSet.partial_update()` |

内部 API 也保持了同样的双前缀别名策略。

### 2.6 公开 API 处理逻辑差异

**文件**: `apps/api/plane/space/views/intake.py:107-173`

与内部 API (`IntakeIssueViewSet`) 相比，公开 API (`IntakeIssuePublicViewSet`) 有以下差异：

1. **认证方式**: 通过 `anchor` 查找 `DeployBoard` 验证项目权限，无需 workspace slug
   ```python
   project_deploy_board = DeployBoard.objects.get(anchor=anchor, entity_name="project")
   if project_deploy_board.intake is None:
       return Response({"error": "Intake is not enabled..."}, 400)
   ```

2. **Issue 创建方式**: 直接调用 `Issue.objects.create()` 而非使用 serializer（简化流程）
   ```python
   issue = Issue.objects.create(
       name=request.data.get("issue", {}).get("name"),
       description_json=request.data.get("issue", {}).get("description_json", {}),
       description_html=request.data.get("issue", {}).get("description_html", "<p></p>"),
       priority=request.data.get("issue", {}).get("priority", "low"),
       project_id=project_deploy_board.project_id,
       state_id=triage_state.id,
   )
   ```

3. **更新权限**: 仅创建者可编辑，无角色校验
   ```python
   if str(intake_issue.created_by_id) != str(request.user.id):
       return Response({"error": "You cannot edit intake issues"}, 400)
   ```

4. **字段限制**: 公开更新只能修改 name 和 description
   ```python
   issue_data = {
       "name": issue_data.get("name", issue.name),
       "description_html": issue_data.get("description_html", issue.description_html),
       "description_json": issue_data.get("description_json", issue.description_json),
   }
   ```

### 2.7 公开 API 权限

| 操作 | 权限 | 说明 |
|------|------|------|
| GET list | ✅ 公开 | 基于 anchor 访问，无需登录 |
| POST create | ✅ 需登录 | 任何登录用户可提交 |
| GET retrieve | ✅ 公开 | 基于 anchor 访问 |
| PATCH update | ⚠️ 仅创建者 | 只能编辑自己提交的 |
| DELETE destroy | ⚠️ 仅创建者 | 只能删除自己提交的 |

---

## 三、关键代码位置汇总（Round 2 新增）

| 模块 | 文件路径 | 关键代码 |
|------|----------|---------|
| 权限枚举 | `apps/api/plane/app/permissions/base.py` | `ROLE` 枚举类 (ADMIN=20, MEMBER=15, GUEST=5) |
| 权限装饰器 | `apps/api/plane/app/permissions/base.py` | `allow_permission()` 装饰器 |
| 后端更新权限 | `apps/api/plane/app/views/intake/base.py` | `partial_update()` 方法 340-424行 |
| 前端权限控制 | `apps/web/core/components/inbox/content/inbox-issue-header.tsx` | `isAllowed`, `isProjectAdmin`, `handleActionWithPermission` |
| 根路由配置 | `apps/api/plane/urls.py` | `api/public/` 路径映射 |
| 公开路由 | `apps/api/plane/space/urls/intake.py` | `intake-issues` 与 `inbox-issues` 双别名路由 |
| 公开 API 视图 | `apps/api/plane/space/views/intake.py` | `IntakeIssuePublicViewSet` 完整实现 |

---

## 四、补充发现

### 4.1 MEMBER 角色的"看得见摸不着"现象

前端代码中 `canMarkAsAccepted` 等变量的判断条件是 `isAllowed`（包含 MEMBER），但实际点击时会被 `isProjectAdmin` 二次校验拦截。这意味着：
- MEMBER 能看到 Accept/Decline 按钮
- 点击后会弹出 "Permission denied" 提示
- 后端也有 `role > MEMBER.value` 硬校验，确保安全

### 4.2 intake-issues vs inbox-issues 命名由来

从路由配置看，这两个术语在代码中是**完全混用**的：
- 数据库模型用 `IntakeIssue`
- 前端 store 用 `InboxIssueStore`
- API 路径同时支持两种前缀
- 这可能是历史演进中术语变更导致的兼容设计
