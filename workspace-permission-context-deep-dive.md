# Workspace 权限上下文深度分析

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

## 二、主线二：私有项目加入约束

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

### 2.2 加入项目的约束检查

**文件**: `apps/api/plane/app/views/project/invite.py:136-143`
```python
projects = Project.objects.filter(id__in=project_ids, workspace__slug=slug).only("id", "network")

for project in projects:
    # Guest 不能自动加入 SECRET 项目
    if project.network == ProjectNetwork.SECRET.value and workspace_member.role != ROLE.ADMIN.value:
        raise serializers.ValidationError(
            f"You cannot join project {project.id} as it is private"
        )
```

**加入约束矩阵**：

| 用户角色 | 可加入 PUBLIC 项目 | 可加入 SECRET 项目 |
|---------|------------------|-------------------|
| Guest | ✅ 是 | ❌ 否（需要显式邀请） |
| Member | ✅ 是 | ❌ 否（需要显式邀请） |
| Admin | ✅ 是 | ✅ 是（自动加入） |

### 2.3 加入后的角色分配

**文件**: `apps/api/plane/app/views/project/invite.py`
```python
# 加入项目时的默认角色
project_member_role = self.getWorkspaceRoleByWorkspaceSlug(workspaceSlug) ?? EUserPermissions.MEMBER;
```

- Workspace Admin 加入项目 → 自动获得 ADMIN 角色
- Workspace Member 加入 PUBLIC 项目 → 自动获得 MEMBER 角色
- Guest 只能通过邀请加入，角色由邀请者指定

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

## 五、代码事实 vs 常见误解

| 陈述 | 真相 | 来源 |
|------|------|------|
| Workspace Admin 可以访问所有项目 | ❌ 必须先成为项目成员，然后自动获得 ADMIN 权限 | `allow_permission` decorator |
| 新项目默认是私有的 | ❌ 默认是 PUBLIC (network=2) | Project model default value |
| Guest 可以加入 PUBLIC 项目 | ✅ 是 | `apps/api/plane/app/views/project/invite.py` |
| Guest 可以加入 SECRET 项目 | ❌ 必须通过显式邀请 | `apps/api/plane/app/views/project/invite.py:139` |
| `allowPermissions` 会调用 API 验证权限 | ❌ 纯本地同步计算，依赖缓存 | `base-permissions.store.ts:191` |
| 切换 workspace 需要手动清理缓存 | ❌ 命名空间设计自动隔离 | fetch-keys.ts 命名空间设计 |
| 403 和 409 由前端判断 | ❌ 后端在 retrieve 接口就返回了不同状态码 | `apps/api/plane/app/views/project/base.py:230-239` |
| 项目级缓存 key 包含 workspaceSlug | ❌ 大多只包含 projectId | fetch-keys.ts |
| 项目角色变化时缓存自动失效 | ✅ 对于包含 projectRole 的 key 是这样 | `PROJECT_LABELS` 等 key 设计 |

---

## 六、完整判定链路：从进入 Workspace 到进入项目页面

### 6.1 阶段一：进入 Workspace

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

### 6.2 阶段二：进入项目页面

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

### 6.3 阶段三：加入项目流程

```
用户点击"加入项目"按钮
        ↓
joinProject(workspaceSlug, projectId)
        ↓
POST /api/users/me/workspaces/{slug}/projects/invitations/
        ↓
后端校验：
        ├─ 项目是 SECRET？
        │   ├─ Yes → 用户是 Admin？
        │   │   ├─ Yes → 允许加入
        │   │   └─ No → 返回错误
        │   └─ No → 允许加入
        └─ 创建 ProjectMember 记录，角色 = workspace 角色（或 MEMBER）
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

### 6.4 阶段四：跨 Workspace 切换

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

## 七、核心文件索引

| 模块 | 文件路径 | 核心职责 |
|------|---------|---------|
| 后端项目可见性 | `apps/api/plane/app/views/project/base.py` | 项目列表过滤、详情访问判定（403/409） |
| 后端权限装饰器 | `apps/api/plane/app/permissions/base.py` | `allow_permission` 装饰器实现 |
| 后端项目加入 | `apps/api/plane/app/views/project/invite.py` | 加入项目约束检查 |
| 后端用户角色 | `apps/api/plane/app/views/project/member.py` | `UserProjectRolesEndpoint` 返回项目角色映射 |
| 前端权限 Store | `apps/web/core/store/user/base-permissions.store.ts` | `allowPermissions`、缓存管理 |
| 前端项目 Wrapper | `apps/web/core/layouts/auth-layout/project-wrapper.tsx` | 项目权限拦截与分流 |
| 前端无权限页面 | `apps/web/core/components/auth-screens/project/project-access-restriction.tsx` | 403/409 页面渲染 |
| 缓存 Key 定义 | `apps/web/core/constants/fetch-keys.ts` | 所有 SWR 缓存 key 命名空间定义 |
