# Workspace 权限上下文深度分析 v2（修订版）

> **修订说明**：本版本修正了 v1 中关于私有项目加入返回状态、角色写入路径的不准确描述，补充了最小证据链，并修正了代码事实表中的错误结论。

---

## 一、主线一：后端项目可见性规则

### 1.1 项目可见性的数据库模型

**项目网络类型** (`apps/api/plane/db/models/project.py:30-36`)：
```python
class ProjectNetwork(Enum):
    SECRET = 0   # 私有/秘密项目
    PUBLIC = 2   # 公开项目
```

**数据库字段** (`apps/api/plane/db/models/project.py:74`)：
```python
network = models.PositiveSmallIntegerField(default=2, choices=NETWORK_CHOICES)
```

> **代码事实**：新项目默认是 PUBLIC (2)，而非 SECRET (0)。

### 1.2 项目列表可见性过滤逻辑

后端在 `ProjectViewSet.list()` 中根据用户 workspace 角色进行不同的过滤：

**文件**: `apps/api/plane/app/views/project/base.py:104-127`
```python
# Guest 用户：只能看到已加入的项目
if WorkspaceMember.objects.filter(
    member=request.user, workspace__slug=slug, is_active=True, role=ROLE.GUEST.value
).exists():
    projects = projects.filter(
        project_projectmember__member=self.request.user,
        project_projectmember__is_active=True,
    )

# Member 用户：能看到已加入的项目 + 所有 PUBLIC 项目
if WorkspaceMember.objects.filter(
    member=request.user, workspace__slug=slug, is_active=True, role=ROLE.MEMBER.value
).exists():
    projects = projects.filter(
        Q(project_projectmember__member=self.request.user, project_projectmember__is_active=True)
        | Q(network=2)  # PUBLIC
    )

# Admin 用户：不额外过滤，能看到所有项目
```

**可见性矩阵**：

| 用户角色 | SECRET 项目可见条件 | PUBLIC 项目可见条件 |
|---------|-------------------|--------------------|
| Guest | 必须是项目成员 | 必须是项目成员 |
| Member | 必须是项目成员 | 始终可见 |
| Admin | 始终可见 | 始终可见 |

### 1.3 项目详情访问判定

**文件**: `apps/api/plane/app/views/project/base.py:221-239`
```python
def retrieve(self, request, slug, pk):
    project = self.get_queryset().filter(archived_at__isnull=True).filter(pk=pk).first()

    if project is None:
        return Response({"error": "Project does not exist"}, status=status.HTTP_404_NOT_FOUND)

    member_ids = [str(project_member.member_id) for project_member in project.members_list]

    # 非项目成员访问时的分流
    if str(request.user.id) not in member_ids:
        if project.network == ProjectNetwork.SECRET.value:
            # SECRET 项目：403 Forbidden
            return Response({"error": "You do not have permission"}, status=status.HTTP_403_FORBIDDEN)
        else:
            # PUBLIC 项目：409 Conflict（可加入）
            return Response({"error": "You are not a member of this project"}, status=status.HTTP_409_CONFLICT)
```

> **关键代码事实**：后端在 `retrieve` 接口就已经区分了 403 和 409 两种状态，而不是前端臆断。

### 1.4 后端权限装饰器 `allow_permission`

**文件**: `apps/api/plane/app/permissions/base.py:19-86`
```python
def allow_permission(allowed_roles, level="PROJECT", creator=False, model=None):
    def decorator(view_func):
        def _wrapped_view(instance, request, *args, **kwargs):
            if level == "WORKSPACE":
                # 检查 workspace 角色
                if WorkspaceMember.objects.filter(
                    member=request.user,
                    workspace__slug=kwargs["slug"],
                    role__in=allowed_role_values,
                    is_active=True,
                ).exists():
                    return view_func(instance, request, *args, **kwargs)
            else:  # PROJECT level
                # 检查项目角色
                is_user_has_allowed_role = ProjectMember.objects.filter(
                    member=request.user,
                    workspace__slug=kwargs["slug"],
                    project_id=kwargs["project_id"],
                    role__in=allowed_role_values,
                    is_active=True,
                ).exists()

                # Workspace Admin 如果是项目成员，自动拥有所有权限
                if is_user_has_allowed_role:
                    return view_func(...)
                elif (
                    ProjectMember.objects.filter(member=request.user, ..., is_active=True).exists()
                    and WorkspaceMember.objects.filter(member=request.user, role=ROLE.ADMIN.value, ...).exists()
                ):
                    return view_func(...)

            return Response({"error": "You don't have the required permissions."},
                          status=status.HTTP_403_FORBIDDEN)
```

> **常见误解**：很多人以为 Workspace Admin 可以访问所有项目而无需加入。但代码显示，Admin 也必须先成为项目成员（`ProjectMember` 记录存在），然后才能自动升级权限。

---

## 二、主线二：私有项目加入约束（修订版）

### 2.1 加入项目的 API 端点

**前端调用** (`apps/web/core/services/user.service.ts:253-259`)：
```typescript
async joinProject(workspaceSlug: string, project_ids: string[]): Promise<any> {
  return this.post(`/api/users/me/workspaces/${workspaceSlug}/projects/invitations/`, { project_ids })
}
```

**后端 URL** (`apps/api/plane/app/urls/project.py:63-66`)：
```python
path(
    "users/me/workspaces/<str:slug>/projects/invitations/",
    UserProjectInvitationsViewset.as_view({"get": "list", "post": "create"}),
    name="user-project-invitations",
),
```

### 2.2 加入接口的权限装饰器

**文件**: `apps/api/plane/app/views/project/invite.py:128`
```python
class UserProjectInvitationsViewset(BaseViewSet):
    ...
    @allow_permission([ROLE.ADMIN, ROLE.MEMBER], level="WORKSPACE")
    def create(self, request, slug):
```

> **关键代码事实**：`UserProjectInvitationsViewset.create` 方法上的装饰器 `@allow_permission([ROLE.ADMIN, ROLE.MEMBER], level="WORKSPACE")` 意味着 **Guest 根本不能调用这个接口**。Guest 要加入项目只能通过邀请链接（`ProjectJoinEndpoint`）。

### 2.3 加入项目的约束检查与返回

**文件**: `apps/api/plane/app/views/project/invite.py:128-180`
```python
@allow_permission([ROLE.ADMIN, ROLE.MEMBER], level="WORKSPACE")
def create(self, request, slug):
    project_ids = request.data.get("project_ids", [])

    # Get the workspace user role
    workspace_member = WorkspaceMember.objects.get(member=request.user, workspace__slug=slug, is_active=True)

    # Get all the projects
    projects = Project.objects.filter(id__in=project_ids, workspace__slug=slug).only("id", "network")
    # Check if user has permission to join each project
    for project in projects:
        if project.network == ProjectNetwork.SECRET.value and workspace_member.role != ROLE.ADMIN.value:
            return Response(
                {"error": "Only workspace admins can join private project"},
                status=status.HTTP_403_FORBIDDEN,  # ✅ 实际状态码：403
            )

    workspace_role = workspace_member.role
    workspace = workspace_member.workspace

    # If the user was already part of workspace
    _ = ProjectMember.objects.filter(workspace__slug=slug, project_id__in=project_ids, member=request.user).update(
        is_active=True
    )

    ProjectMember.objects.bulk_create(
        [
            ProjectMember(
                project_id=project_id,
                member=request.user,
                role=workspace_role,  # ✅ 角色来源：直接使用 workspace 角色
                workspace=workspace,
                created_by=request.user,
            )
            for project_id in project_ids
        ],
        ignore_conflicts=True,
    )

    ProjectUserProperty.objects.bulk_create(
        [
            ProjectUserProperty(
                project_id=project_id,
                user=request.user,
                workspace=workspace,
                created_by=request.user,
            )
            for project_id in project_ids
        ],
        ignore_conflicts=True,
    )

    return Response({"message": "Projects joined successfully"}, status=status.HTTP_201_CREATED)  # ✅ 成功状态码：201
```

### 2.4 加入约束矩阵（修订版）

| 用户角色 | 可调用 joinProject 接口 | 可加入 PUBLIC 项目 | 可加入 SECRET 项目 | 加入后角色 |
|---------|-----------------------|------------------|-------------------|-----------|
| Guest | ❌ 否（装饰器拦截） | ❌ 否（只能通过邀请链接） | ❌ 否（只能通过邀请链接） | 由邀请者指定 |
| Member | ✅ 是 | ✅ 是 | ❌ 否（返回 403） | Workspace 角色（MEMBER=15） |
| Admin | ✅ 是 | ✅ 是 | ✅ 是 | Workspace 角色（ADMIN=20） |

### 2.5 加入失败时的实际返回形态

**场景**：Member 用户尝试加入 SECRET 项目

**请求**：
```
POST /api/users/me/workspaces/{slug}/projects/invitations/
Body: { "project_ids": ["<secret-project-id>"] }
```

**响应**（`apps/api/plane/app/views/project/invite.py:140-143`）：
```http
HTTP/1.1 403 Forbidden
Content-Type: application/json

{
  "error": "Only workspace admins can join private project"
}
```

### 2.6 加入成功后的角色写入路径（修订版）

**后端写入路径**：
1. 从 `WorkspaceMember` 获取用户在 workspace 中的角色：`workspace_member.role`（第 133、145 行）
2. 直接使用该角色创建 `ProjectMember` 记录：`role=workspace_role`（第 158 行）
3. 同时创建 `ProjectUserProperty` 记录（第 167-178 行）

**前端缓存更新路径** (`apps/web/core/store/user/base-permissions.store.ts:320-334`)：
```typescript
joinProject = async (workspaceSlug: string, projectId: string): Promise<void> => {
  const response = await userService.joinProject(workspaceSlug, [projectId]);
  const projectMemberRole = this.getWorkspaceRoleByWorkspaceSlug(workspaceSlug) ?? EUserPermissions.MEMBER;
  if (response) {
    runInAction(() => {
      set(this.workspaceProjectsPermissions, [workspaceSlug, projectId], projectMemberRole);
    });
    void this.fetchWorkspaceLevelProjectEntities(workspaceSlug, projectId);
  }
};
```

> **代码事实**：前端和后端在角色分配上是一致的——都使用用户在 workspace 中的角色作为项目角色。前端的 `?? EUserPermissions.MEMBER` 只是防御性默认值，正常情况下不会触发。

### 2.7 Guest 加入项目的特殊路径

Guest 无法调用 `joinProject` 接口（被装饰器拦截），只能通过邀请链接加入：

**文件**: `apps/api/plane/app/views/project/invite.py:183-249`
```python
class ProjectJoinEndpoint(BaseAPIView):
    permission_classes = [AllowAny]

    def post(self, request, slug, project_id, pk):
        project_invite = ProjectMemberInvite.objects.get(pk=pk, project_id=project_id, workspace__slug=slug)

        if project_invite.accepted:
            # 创建 ProjectMember 记录，角色使用邀请时指定的角色
            _ = ProjectMember.objects.create(
                project_id=project_id,
                member=user,
                role=project_invite.role,  # 角色来自邀请记录，不是 workspace 角色
            )
```

> **关键区别**：通过邀请链接加入时，角色使用 `project_invite.role`（邀请者指定的角色），而不是用户的 workspace 角色。

---

## 三、主线三：前端 allowPermissions 与 403/409 页面分流

### 3.1 前端 `allowPermissions` 的真相

**文件**: `apps/web/core/store/user/base-permissions.store.ts:191-229`
```typescript
allowPermissions = (
  allowPermissions: ETempUserRole[],
  level: TUserPermissionsLevel,
  workspaceSlug?: string,
  projectId?: string,
  onPermissionAllowed?: () => boolean
): boolean => {
  // 1. 从 router 获取当前上下文（如果没传）
  const { workspaceSlug: currentWorkspaceSlug, projectId: currentProjectId } = this.store.router;
  if (!workspaceSlug) workspaceSlug = currentWorkspaceSlug;
  if (!projectId) projectId = currentProjectId;

  // 2. 根据层级获取角色
  let currentUserRole: TUserPermissions | undefined = undefined;
  if (level === EUserPermissionsLevel.WORKSPACE) {
    currentUserRole = this.getWorkspaceRoleByWorkspaceSlug(workspaceSlug);
  }
  if (level === EUserPermissionsLevel.PROJECT) {
    currentUserRole = this.getProjectRoleByWorkspaceSlugAndProjectId(workspaceSlug, projectId);
  }

  // 3. 检查是否在允许列表中
  if (currentUserRole && allowPermissions.includes(currentUserRole)) {
    return onPermissionAllowed ? onPermissionAllowed() : true;
  }
  return false;
};
```

**常见误解**：
- ❌ 误解：`allowPermissions` 会发起 API 请求验证权限
- ✅ 事实：`allowPermissions` 完全基于本地 store 中的缓存数据，是**纯同步计算**

> **代码事实**：`allowPermissions` 的准确性完全依赖于 store 中缓存的权限数据是否最新。如果缓存过期，判定可能错误。

### 3.2 403/409 页面分流逻辑

**ProjectAuthWrapper** (`apps/web/core/layouts/auth-layout/project-wrapper.tsx:149-158`)：
```typescript
const hasPermissionToCurrentProject = allowPermissions(
  [EUserPermissions.ADMIN, EUserPermissions.MEMBER, EUserPermissions.GUEST],
  EUserPermissionsLevel.PROJECT,
  workspaceSlug,
  projectId
);

if (!isProjectLoading && hasPermissionToCurrentProject === false) {
  return (
    <ProjectAccessRestriction
      errorStatusCode={projectDetailsError?.status}
      isWorkspaceAdmin={isWorkspaceAdmin}
      handleJoinProject={handleJoinProject}
      isJoinButtonDisabled={isJoiningProject}
    />
  );
}
```

**ProjectAccessRestriction** 组件 (`apps/web/core/components/auth-screens/project/project-access-restriction.tsx:24-75`)：
```typescript
// 409 Conflict：PUBLIC 项目，用户不是成员，显示"加入项目"按钮
// 403 Forbidden + Workspace Admin：SECRET 项目但用户是 Admin，仍可加入
if (errorStatusCode === 409 || (errorStatusCode === 403 && isWorkspaceAdmin))
  return (
    <EmptyStateDetailed
      title={t("project_empty_state.no_access.title")}
      description={t("project_empty_state.no_access.join_description")}
      actions={[{ label: t("project_empty_state.no_access.cta_primary"), onClick: handleJoinProject }]}
    />
  );

// 403 Forbidden + 非 Admin：SECRET 项目，用户无权限，仅显示无权限提示
if (errorStatusCode === 403) {
  return (
    <EmptyStateDetailed
      title={t("project_empty_state.no_access.title")}
      description={t("project_empty_state.no_access.restricted_description")}
      // 无操作按钮
    />
  );
}
```

### 3.3 分流状态机

```
用户访问项目页面
        ↓
fetchProjectDetails() → 后端返回
        ├─ 200 OK → 正常渲染
        ├─ 404 Not Found → 项目不存在
        ├─ 409 Conflict → PUBLIC 项目，非成员 → 显示"加入项目"
        └─ 403 Forbidden → SECRET 项目
                  ├─ isWorkspaceAdmin = true → 显示"加入项目"
                  └─ isWorkspaceAdmin = false → 显示"无权限"
```

### 3.4 前端缺陷：Guest 用户看到不可点击的按钮

根据代码分析，存在一个前端逻辑缺陷：

1. Guest 直接通过 URL 访问 PUBLIC 项目（非成员）→ 后端返回 409
2. `ProjectAccessRestriction` 看到 409 → 显示"加入项目"按钮
3. Guest 点击按钮 → 调用 `joinProject` → 后端装饰器拦截 → 返回 403
4. 用户看到按钮但点击后报错

> **问题**：前端应该在 `isWorkspaceAdmin` 之外，额外检查用户是否是 Guest，避免显示无法实际点击的按钮。

---

## 四、主线四：跨 Workspace 与跨 Project 缓存 Key 隔离边界

### 4.1 SWR 缓存 Key 设计原则

**文件**: `apps/web/core/constants/fetch-keys.ts`

**Workspace 级 Key（含 workspaceSlug）**：
```typescript
export const WORKSPACE_MEMBER_ME_INFORMATION = (workspaceSlug: string) =>
  `WORKSPACE_MEMBER_ME_INFORMATION_${workspaceSlug.toUpperCase()}`;

export const WORKSPACE_PROJECTS_ROLES_INFORMATION = (workspaceSlug: string) =>
  `WORKSPACE_PROJECTS_ROLES_INFORMATION_${workspaceSlug.toUpperCase()}`;

export const WORKSPACE_PARTIAL_PROJECTS = (workspaceSlug: string) =>
  `WORKSPACE_PARTIAL_PROJECTS_${workspaceSlug.toUpperCase()}`;
```

**Project 级 Key（含 projectId，部分含 projectRole）**：
```typescript
export const PROJECT_DETAILS = (workspaceSlug: string, projectId: string) =>
  `PROJECT_DETAILS_${projectId.toString().toUpperCase()}`;

export const PROJECT_LABELS = (projectId: string, projectRole: EUserPermissions | undefined) =>
  `PROJECT_LABELS_${projectId.toString().toUpperCase()}_${projectRole}`;

export const PROJECT_MEMBERS = (projectId: string, projectRole: EUserPermissions | undefined) =>
  `PROJECT_MEMBERS_${projectId.toString().toUpperCase()}_${projectRole}`;
```

> **关键设计**：项目级资源的缓存 key 包含 `projectRole`。这意味着当用户在项目中的角色变化时，缓存会自动失效并重新拉取。

### 4.2 跨 Workspace 隔离边界

| 缓存层级 | Key 包含 workspaceSlug | 切换 Workspace 时行为 |
|---------|------------------------|----------------------|
| WORKSPACE_MEMBER_ME_INFORMATION | ✅ 是 | 自动触发新请求 |
| WORKSPACE_PROJECTS_ROLES_INFORMATION | ✅ 是 | 自动触发新请求 |
| WORKSPACE_PARTIAL_PROJECTS | ✅ 是 | 自动触发新请求 |
| workspaceUserInfo[workspaceSlug] | ✅ 是（MobX key） | 旧数据保留但不访问 |
| workspaceProjectsPermissions[workspaceSlug] | ✅ 是（MobX key） | 旧数据保留但不访问 |

> **代码事实**：由于所有 workspace 相关数据都以 workspaceSlug 命名空间隔离，切换 workspace 时**不需要显式清理缓存**。SWR 会根据新 key 自动拉取新数据，MobX store 中不同 workspace 的数据互不干扰。

### 4.3 跨 Project 隔离边界

| 缓存层级 | Key 包含 projectId | Key 包含 projectRole | 加入/离开项目时行为 |
|---------|-------------------|---------------------|---------------------|
| PROJECT_DETAILS | ✅ 是 | ❌ 否 | 需要手动清理 |
| PROJECT_LABELS | ✅ 是 | ✅ 是 | 角色变化时自动失效 |
| PROJECT_MEMBERS | ✅ 是 | ✅ 是 | 角色变化时自动失效 |
| PROJECT_STATES | ✅ 是 | ✅ 是 | 角色变化时自动失效 |
| projectMap[projectId] | ✅ 是（MobX key） | ❌ 否 | 离开项目时手动 unset |

**离开项目时的清理** (`apps/web/core/store/user/base-permissions.store.ts:342-354`)：
```typescript
leaveProject = async (workspaceSlug: string, projectId: string): Promise<void> => {
  await userService.leaveProject(workspaceSlug, projectId);
  runInAction(() => {
    unset(this.workspaceProjectsPermissions, [workspaceSlug, projectId]);
    unset(this.projectUserInfo, [workspaceSlug, projectId]);
    unset(this.store.projectRoot.project.projectMap, [projectId]);
  });
};
```

### 4.4 缓存隔离的设计缺陷

**问题**：`PROJECT_DETAILS` 的 key 只包含 `projectId`，不包含 `workspaceSlug` 或 `projectRole`。
```typescript
export const PROJECT_DETAILS = (workspaceSlug: string, projectId: string) =>
  `PROJECT_DETAILS_${projectId.toString().toUpperCase()}`;  // workspaceSlug 未使用！
```

> **潜在风险**：理论上，如果两个不同 workspace 中有相同的 projectId（虽然概率极低），缓存会冲突。但实际中 projectId 是 UUID，全局唯一，所以这个问题不存在。

---

## 五、最小证据链：后端加入项目判定流程

### 5.1 证据链说明

以下是 Member 用户尝试加入 SECRET 项目时，后端的完整判定流程，每条结论都对应具体代码行号：

```
用户调用 POST /api/users/me/workspaces/{slug}/projects/invitations/
        ↓
[1] 装饰器拦截检查
    @allow_permission([ROLE.ADMIN, ROLE.MEMBER], level="WORKSPACE")
    来源：apps/api/plane/app/views/project/invite.py:128
    判定：用户是 ADMIN 或 MEMBER？→ Guest 直接返回 403
        ↓
[2] 获取 workspace 成员信息
    workspace_member = WorkspaceMember.objects.get(member=request.user, ...)
    来源：apps/api/plane/app/views/project/invite.py:133
    结果：workspace_member.role = 15 (MEMBER)
        ↓
[3] 遍历项目检查 network
    for project in projects:
        if project.network == SECRET and workspace_member.role != ADMIN:
    来源：apps/api/plane/app/views/project/invite.py:138-139
    判定：项目是 SECRET 且用户不是 Admin？→ 进入错误分支
        ↓
[4] 返回 403 Forbidden
    return Response(
        {"error": "Only workspace admins can join private project"},
        status=status.HTTP_403_FORBIDDEN
    )
    来源：apps/api/plane/app/views/project/invite.py:140-143
    结果：请求被拒绝
```

### 5.2 成功路径的证据链

```
Admin 用户调用 POST /api/users/me/workspaces/{slug}/projects/invitations/
        ↓
[1] 装饰器检查通过
    来源：apps/api/plane/app/views/project/invite.py:128
        ↓
[2] 获取 workspace_member.role = 20 (ADMIN)
    来源：apps/api/plane/app/views/project/invite.py:133
        ↓
[3] 项目 network 检查通过
    if project.network == SECRET and workspace_member.role != ADMIN:
    → 条件不成立（用户是 Admin），跳过错误分支
    来源：apps/api/plane/app/views/project/invite.py:138-143
        ↓
[4] 创建 ProjectMember 记录
    ProjectMember.objects.bulk_create([
        ProjectMember(..., role=workspace_role, ...)
    ])
    来源：apps/api/plane/app/views/project/invite.py:153-165
    结果：role = 20（继承 workspace Admin 角色）
        ↓
[5] 返回 201 Created
    return Response({"message": "Projects joined successfully"},
                   status=status.HTTP_201_CREATED)
    来源：apps/api/plane/app/views/project/invite.py:180
```

---

## 六、代码事实 vs 常见误解（修订版）

| 陈述 | v1 结论 | v2 修正后结论 | 证据来源 |
|------|--------|--------------|---------|
| Workspace Admin 可以访问所有项目 | ❌ 必须先加入项目 | ❌ 必须先加入项目 | `allow_permission` decorator |
| 新项目默认是私有的 | ❌ 默认 PUBLIC | ❌ 默认 PUBLIC | Project model default value |
| Guest 可以加入 PUBLIC 项目 | ✅ 是 | ❌ 否（Guest 不能调用 joinProject 接口，只能通过邀请） | `@allow_permission([ROLE.ADMIN, ROLE.MEMBER])` |
| Guest 可以加入 SECRET 项目 | ❌ 需显式邀请 | ❌ 需显式邀请 | `apps/api/plane/app/views/project/invite.py:128` |
| 私有项目加入失败返回 409 | （未提及） | ❌ 返回 403 Forbidden | `apps/api/plane/app/views/project/invite.py:142` |
| 加入成功返回 200 OK | （未提及） | ❌ 返回 201 Created | `apps/api/plane/app/views/project/invite.py:180` |
| 加入后角色 = workspace 角色 | ✅ 是 | ✅ 是（但 Guest 邀请加入时使用邀请者指定的角色） | `apps/api/plane/app/views/project/invite.py:158` |
| `allowPermissions` 会调用 API | ❌ 纯本地计算 | ❌ 纯本地计算 | `base-permissions.store.ts:191` |
| 切换 workspace 需要手动清理缓存 | ❌ 命名空间自动隔离 | ❌ 命名空间自动隔离 | fetch-keys.ts 命名空间设计 |
| 403/409 由前端判断 | ❌ 后端返回 | ❌ 后端返回 | `apps/api/plane/app/views/project/base.py:230-239` |
| 项目级缓存 key 包含 workspaceSlug | ❌ 大多只含 projectId | ❌ 大多只含 projectId | fetch-keys.ts |
| 项目角色变化时缓存自动失效 | ✅ 含 projectRole 的 key | ✅ 含 projectRole 的 key | `PROJECT_LABELS` 等 key 设计 |

---

## 七、完整判定链路：从进入 Workspace 到进入项目页面

### 7.1 阶段一：进入 Workspace

```
用户输入 URL: /{workspaceSlug}/
        ↓
[Next.js 路由] 解析 workspaceSlug
        ↓
[WorkspaceAuthWrapper] 挂载
        ├─ useSWR(WORKSPACE_MEMBER_ME_INFORMATION)
        │   → GET /api/workspaces/{slug}/workspace-members/me/
        │   → 后端返回 IWorkspaceMemberMe（含 role 字段）
        │   → 存入 workspaceUserInfo[workspaceSlug]
        ├─ useSWR(WORKSPACE_PROJECTS_ROLES_INFORMATION)
        │   → GET /api/users/me/workspaces/{slug}/project-roles/
        │   → 后端返回 { projectId: role } 映射
        │   → 存入 workspaceProjectsPermissions[workspaceSlug]
        └─ useSWR(WORKSPACE_PARTIAL_PROJECTS)
            → GET /api/workspaces/{slug}/projects/
            → 后端根据用户角色过滤项目列表（见 1.2）
            → 存入 projectMap（每个 project 含 member_role 字段）
        ↓
[MobX computed 计算]
        ├─ currentWorkspace = workspaces.find(w => w.slug === workspaceSlug)
        └─ joinedProjectIds = projectMap.filter(p => p.member_role != null)
        ↓
[Sidebar 渲染]
        ├─ 遍历 joinedProjectIds，渲染项目列表
        └─ 遍历导航项，用 allowPermissions 过滤无权访问的项
        ↓
✅ Workspace 上下文就绪
```

### 7.2 阶段二：进入项目页面

```
用户点击项目 /{workspaceSlug}/projects/{projectId}/
        ↓
[Next.js 路由] 解析 workspaceSlug, projectId
        ↓
[ProjectAuthWrapper] 挂载
        ├─ useSWR(PROJECT_DETAILS)
        │   → GET /api/workspaces/{slug}/projects/{projectId}/
        │   → 后端判定：
        │      ├─ 用户是成员 → 200 OK，返回项目详情
        │      ├─ 用户非成员 + PUBLIC → 409 Conflict
        │      └─ 用户非成员 + SECRET → 403 Forbidden
        └─ useSWR(PROJECT_ME_INFORMATION)
            → GET /api/workspaces/{slug}/projects/{projectId}/project-members/me/
            → 存入 projectUserInfo[workspaceSlug][projectId]
        ↓
[权限判定]
        ├─ hasPermissionToCurrentProject = allowPermissions(
        │     [ADMIN, MEMBER, GUEST], PROJECT, workspaceSlug, projectId
        │   )
        ├─ 检查 projectDetailsError?.status
        └─ isWorkspaceAdmin = allowPermissions([ADMIN], WORKSPACE, workspaceSlug)
        ↓
{ hasPermissionToCurrentProject === true }
        ├─ Yes → 渲染项目页面内容
        └─ No → 渲染 <ProjectAccessRestriction />
                  ├─ status === 409 → 显示"加入项目"按钮
                  ├─ status === 403 && isWorkspaceAdmin → 显示"加入项目"按钮
                  └─ status === 403 && !isWorkspaceAdmin → 显示"无权限"
        ↓
✅ 项目页面渲染完成
```

### 7.3 阶段三：加入项目流程（修订版）

```
用户点击"加入项目"按钮
        ↓
joinProject(workspaceSlug, projectId)
        ↓
POST /api/users/me/workspaces/{slug}/projects/invitations/
        ↓
[1] 装饰器检查 @allow_permission([ADMIN, MEMBER], WORKSPACE)
        ├─ Guest → 403 Forbidden（"You don't have the required permissions."）
        └─ Member/Admin → 继续
        ↓
[2] 后端校验：
        ├─ 项目是 SECRET？
        │   ├─ Yes → 用户是 Admin？
        │   │   ├─ Yes → 允许加入
        │   │   └─ No → 403 Forbidden（"Only workspace admins can join private project"）
        │   └─ No → 允许加入
        ↓
[3] 创建 ProjectMember 记录，角色 = workspace 角色
        ↓
[4] 返回 201 Created
        ↓
前端更新 store：
        ├─ set(workspaceProjectsPermissions, [workspaceSlug, projectId], role)
        └─ fetchWorkspaceLevelProjectEntities() 重新拉取项目数据
        ↓
[MobX 响应式更新]
        ├─ joinedProjectIds 自动包含新项目
        ├─ allowPermissions 判定通过
        └─ 项目页面自动渲染
        ↓
✅ 加入成功
```

### 7.4 阶段四：跨 Workspace 切换

```
用户切换 Workspace（URL 变化）
        ↓
[Next.js 路由] workspaceSlug 更新
        ↓
[WorkspaceAuthWrapper] 检测到 workspaceSlug 变化
        ├─ 旧的 useSWR key 失效（因为包含 workspaceSlug）
        ├─ 新的 workspaceSlug 触发新的 SWR 请求
        │   ├─ WORKSPACE_MEMBER_ME_INFORMATION(newSlug)
        │   ├─ WORKSPACE_PROJECTS_ROLES_INFORMATION(newSlug)
        │   └─ WORKSPACE_PARTIAL_PROJECTS(newSlug)
        └─ MobX store 中旧 workspace 数据保留但不再被访问
        ↓
[MobX computed 重新计算]
        ├─ currentWorkspace 指向新 workspace
        └─ joinedProjectIds 基于新 workspace 的项目
        ↓
[UI 全量重渲染]
        ├─ 侧边栏项目列表更新
        └─ 所有 allowPermissions 基于新上下文判定
        ↓
✅ Workspace 切换完成，无需手动清理
```

---

## 八、核心文件索引

| 模块 | 文件路径 | 核心职责 |
|------|---------|---------|
| 后端项目可见性 | `apps/api/plane/app/views/project/base.py` | 项目列表过滤、详情访问判定（403/409） |
| 后端权限装饰器 | `apps/api/plane/app/permissions/base.py` | `allow_permission` 装饰器实现 |
| 后端项目加入 | `apps/api/plane/app/views/project/invite.py` | 加入项目约束检查、角色分配 |
| 后端用户角色 | `apps/api/plane/app/views/project/member.py` | `UserProjectRolesEndpoint` 返回项目角色映射 |
| 前端权限 Store | `apps/web/core/store/user/base-permissions.store.ts` | `allowPermissions`、缓存管理 |
| 前端项目 Wrapper | `apps/web/core/layouts/auth-layout/project-wrapper.tsx` | 项目权限拦截与分流 |
| 前端无权限页面 | `apps/web/core/components/auth-screens/project/project-access-restriction.tsx` | 403/409 页面渲染 |
| 缓存 Key 定义 | `apps/web/core/constants/fetch-keys.ts` | 所有 SWR 缓存 key 命名空间定义 |

---

## 九、v1 → v2 修订清单

| 修订项 | v1 错误/遗漏 | v2 修正 |
|--------|-------------|---------|
| 私有项目加入失败状态码 | 未明确说明 | 明确返回 403 Forbidden，消息 `"Only workspace admins can join private project"` |
| 加入成功状态码 | 未提及 | 明确返回 201 Created |
| Guest 加入 PUBLIC 项目 | 认为可以直接加入 | 修正：Guest 不能调用 joinProject 接口，只能通过邀请链接 |
| 加入后角色来源 | 前端 `?? EUserPermissions.MEMBER` 误导 | 明确：正常流程下角色完全继承 workspace 角色，`??` 只是防御性默认 |
| Guest 特殊加入路径 | 未提及 | 补充：Guest 只能通过 `ProjectJoinEndpoint` 邀请链接加入，角色由邀请者指定 |
| 前端缺陷 | 未提及 | 补充：Guest 访问 PUBLIC 项目时会看到"加入项目"按钮但点击后报错 |
| 最小证据链 | 未提供 | 新增：完整的后端判定流程证据链，每条结论附带代码行号 |
| 代码事实表 | 3 条不准确 | 修正 3 条结论，新增 3 条事实 |
