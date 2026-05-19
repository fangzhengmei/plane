# Workspace 权限上下文协作分析

## 一、系统架构总览

成员加入 workspace 后能看到的项目和数据由三层机制共同决定：
1. **权限判定发起点** - 角色定义与项目级覆盖规则
2. **缓存层** - SWR + MobX 双层缓存，以 workspaceSlug 为命名空间隔离
3. **界面渲染裁剪** - 基于权限的条件渲染与数据过滤

三者通过 `workspaceSlug` 作为核心上下文标识串联协作，确保跨工作区切换时权限上下文正确隔离与清理。

---

## 二、权限判定发起点

### 2.1 角色定义体系

角色枚举定义在两处：

**权限等级枚举** (`packages/constants/src/user.ts:36-40`)：
```typescript
export enum EUserPermissions {
  ADMIN = 20,
  MEMBER = 15,
  GUEST = 5,
}
```

数值越大权限越高，权限判定采用"包含"逻辑（高权限自动拥有低权限能力）。

### 2.2 项目级覆盖规则

核心判定逻辑在 `BaseUserPermissionStore.getProjectRole` 中实现：

**文件**: `apps/web/core/store/user/base-permissions.store.ts:122-129`
```typescript
protected getProjectRole = computedFn((workspaceSlug: string, projectId?: string): EUserPermissions | undefined => {
  if (!workspaceSlug || !projectId) return undefined;
  const projectRole = this.workspaceProjectsPermissions?.[workspaceSlug]?.[projectId];
  if (!projectRole) return undefined;
  const workspaceRole = this.workspaceUserInfo?.[workspaceSlug]?.role;
  if (workspaceRole === EUserWorkspaceRoles.ADMIN) return EUserPermissions.ADMIN;
  else return projectRole;
});
```

**覆盖规则**：
- Workspace Admin 自动拥有所有项目的 ADMIN 权限，忽略项目级角色设定
- 非 Admin 用户使用项目级角色（`workspaceProjectsPermissions` 中存储）

### 2.3 通用权限判定入口

`allowPermissions` 方法是所有权限检查的统一入口：

**文件**: `apps/web/core/store/user/base-permissions.store.ts:191-229`
```typescript
allowPermissions = (
  allowPermissions: ETempUserRole[],
  level: TUserPermissionsLevel,
  workspaceSlug?: string,
  projectId?: string,
  onPermissionAllowed?: () => boolean
): boolean => {
  // 1. 从 router 获取当前上下文
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

  // 3. 判定是否在允许列表中
  if (currentUserRole && allowPermissions.includes(currentUserRole)) {
    return onPermissionAllowed ? onPermissionAllowed() : true;
  }
  return false;
};
```

### 2.4 权限判定发起点位置

| 层级 | 发起点文件 | 作用 |
|------|-----------|------|
| Workspace 级 | `apps/web/core/layouts/auth-layout/workspace-wrapper.tsx` | 进入 workspace 时拉取权限数据，验证访问权限 |
| Project 级 | `apps/web/core/layouts/auth-layout/project-wrapper.tsx` | 进入项目时拉取项目权限，验证项目访问权限 |
| 组件级 | 各处组件通过 `useUserPermissions()` hook | 按钮可见性、操作权限等细粒度控制 |

---

## 三、缓存层实现机制

系统采用 **SWR（API 层） + MobX Store（应用层）** 双层缓存架构。

### 3.1 SWR 缓存层

**SWR 配置** (`packages/constants/src/swr.ts:7-22`)：
```typescript
export const DEFAULT_SWR_CONFIG = {
  refreshWhenHidden: false,
  revalidateIfStale: false,
  revalidateOnFocus: false,
  revalidateOnMount: true,
  refreshInterval: 600000,
  errorRetryCount: 3,
};
```

**Fetch Key 命名空间设计** (`apps/web/core/constants/fetch-keys.ts`)：

所有与 workspace 相关的缓存 key 都以 `workspaceSlug` 作为命名空间的一部分：
```typescript
export const WORKSPACE_MEMBER_ME_INFORMATION = (workspaceSlug: string) =>
  `WORKSPACE_MEMBER_ME_INFORMATION_${workspaceSlug.toUpperCase()}`;

export const WORKSPACE_PROJECTS_ROLES_INFORMATION = (workspaceSlug: string) =>
  `WORKSPACE_PROJECTS_ROLES_INFORMATION_${workspaceSlug.toUpperCase()}`;

export const PROJECT_LABELS = (projectId: string, projectRole: EUserPermissions | undefined) =>
  `PROJECT_LABELS_${projectId.toString().toUpperCase()}_${projectRole}`;
```

**关键设计**：项目级资源的缓存 key 甚至包含 `projectRole`，确保角色变化时缓存自动失效。

### 3.2 MobX Store 缓存层

`BaseUserPermissionStore` 维护三个核心 observable，均以 `workspaceSlug` 为顶级键：

**文件**: `apps/web/core/store/user/base-permissions.store.ts:69-72`
```typescript
workspaceUserInfo: Record<string, IWorkspaceMemberMe> = {};           // workspaceSlug -> 成员信息
projectUserInfo: Record<string, Record<string, TProjectMembership>> = {}; // workspaceSlug -> projectId -> 成员信息
workspaceProjectsPermissions: Record<string, IUserProjectsRole> = {};  // workspaceSlug -> projectId -> 角色
```

### 3.3 缓存加载时机（WorkspaceWrapper）

**文件**: `apps/web/core/layouts/auth-layout/workspace-wrapper.tsx:76-128`

当 `workspaceSlug` 变化时，通过 useSWR 自动触发：
```typescript
// 1. 拉取当前用户在 workspace 中的角色
useSWR(
  workspaceSlug && currentWorkspace ? WORKSPACE_MEMBER_ME_INFORMATION(workspaceSlug.toString()) : null,
  () => fetchUserWorkspaceInfo(workspaceSlug.toString())
);

// 2. 拉取当前用户在 workspace 所有项目中的角色
useSWR(
  workspaceSlug && currentWorkspace ? WORKSPACE_PROJECTS_ROLES_INFORMATION(workspaceSlug.toString()) : null,
  () => fetchUserProjectPermissions(workspaceSlug.toString())
);

// 3. 拉取项目列表（含 member_role 过滤字段）
useSWR(
  workspaceSlug && currentWorkspace ? WORKSPACE_PARTIAL_PROJECTS(workspaceSlug.toString()) : null,
  () => fetchPartialProjects(workspaceSlug.toString())
);
```

### 3.4 缓存清理机制

**主动清理** - 离开 workspace 时：
```typescript
// base-permissions.store.ts:260-272
leaveWorkspace = async (workspaceSlug: string): Promise<void> => {
  await userService.leaveWorkspace(workspaceSlug);
  runInAction(() => {
    unset(this.workspaceUserInfo, workspaceSlug);
    unset(this.projectUserInfo, workspaceSlug);
    unset(this.workspaceProjectsPermissions, workspaceSlug);
  });
};
```

**被动隔离** - 由于所有缓存 key 都包含 `workspaceSlug`，切换 workspace 时：
1. 新 workspaceSlug 触发新的 SWR key，自动拉取新数据
2. 旧数据仍保留在内存中但不会被访问（因为 router 上下文已变）
3. 内存中可同时存在多个 workspace 的权限数据，切换无需重新加载

---

## 四、界面渲染裁剪

界面层通过"数据过滤 + 条件渲染"两层机制实现权限裁剪。

### 4.1 数据层过滤

**项目可见性过滤** 在 `ProjectStore` 中通过 computed 属性实现：

**文件**: `apps/web/core/store/project/project.store.ts:239-250`
```typescript
get joinedProjectIds() {
  const currentWorkspace = this.rootStore.workspaceRoot.currentWorkspace;
  if (!currentWorkspace) return [];

  let projects = Object.values(this.projectMap ?? {});
  projects = sortBy(projects, "sort_order");

  // 只返回有 member_role 的项目（即用户已加入的项目）
  const projectIds = projects
    .filter((project) => project.workspace === currentWorkspace.id && !!project.member_role && !project.archived_at)
    .map((project) => project.id);
  return projectIds;
}
```

侧边栏项目列表直接使用此 computed 属性，无需额外权限检查：
```typescript
// projects-list.tsx:51
const { joinedProjectIds: joinedProjects } = useProject();
```

### 4.2 组件级条件渲染

**侧边栏导航项** - 无权限时返回 null：

**文件**: `apps/web/core/components/workspace/sidebar/workspace-menu-item.tsx:52-54`
```typescript
if (!allowPermissions(item.access as any, EUserPermissionsLevel.WORKSPACE, workspaceSlug.toString())) {
  return null;
}
```

**导航配置定义** (`packages/constants/src/workspace.ts:201-223`)：
```typescript
export const WORKSPACE_SIDEBAR_DYNAMIC_NAVIGATION_ITEMS: Record<string, IWorkspaceSidebarNavigationItem> = {
  views: {
    key: "views",
    href: `/workspace-views/all-issues/`,
    access: [EUserWorkspaceRoles.ADMIN, EUserWorkspaceRoles.MEMBER, EUserWorkspaceRoles.GUEST],
    // ...
  },
  analytics: {
    key: "analytics",
    href: `/analytics/`,
    access: [EUserWorkspaceRoles.ADMIN, EUserWorkspaceRoles.MEMBER], // Guest 不可见
    // ...
  },
};
```

**操作按钮可见性** - 典型模式：
```typescript
// projects-list.tsx:57-60
const isAuthorizedUser = allowPermissions(
  [EUserPermissions.ADMIN, EUserPermissions.MEMBER],
  EUserPermissionsLevel.WORKSPACE
);

// 条件渲染创建按钮
{isAuthorizedUser && (
  <IconButton icon={PlusIcon} onClick={() => setIsProjectModalOpen(true)} />
)}
```

### 4.3 页面级权限拦截

**Project Wrapper** 拦截无权限访问：

**文件**: `apps/web/core/layouts/auth-layout/project-wrapper.tsx:149-158`
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

---

## 五、三者协作关系

### 5.1 完整数据流

```
用户输入 URL (包含 workspaceSlug)
        ↓
[Next.js 路由] → 解析 workspaceSlug, projectId
        ↓
[WorkspaceAuthWrapper]
  ├─ useSWR(WORKSPACE_MEMBER_ME_INFORMATION)
  │    → fetchUserWorkspaceInfo()
  │    → 存入 workspaceUserInfo[workspaceSlug]
  ├─ useSWR(WORKSPACE_PROJECTS_ROLES_INFORMATION)
  │    → fetchUserProjectPermissions()
  │    → 存入 workspaceProjectsPermissions[workspaceSlug]
  └─ useSWR(WORKSPACE_PARTIAL_PROJECTS)
       → fetchPartialProjects()
       → 存入 projectMap（含 member_role 字段）
        ↓
[MobX computed]
  ├─ currentWorkspace（基于 router.workspaceSlug）
  ├─ joinedProjectIds（过滤 project.member_role）
  └─ getProjectRole（workspace admin 覆盖逻辑）
        ↓
[组件渲染]
  ├─ SidebarWorkspaceMenuItem → allowPermissions → 渲染或 null
  ├─ SidebarProjectsList → joinedProjectIds → 只渲染已加入项目
  └─ 各操作按钮 → allowPermissions → 条件渲染
```

### 5.2 跨工作区切换时的上下文清理

当用户从 Workspace A 切换到 Workspace B 时：

1. **URL 变化** → `workspaceSlug` 路由参数更新
2. **SWR 自动重新获取** → 新的 workspaceSlug 生成新的 cache key，触发数据拉取
3. **MobX store 更新** → 新数据写入 `workspaceUserInfo[newSlug]`，旧数据保留但不访问
4. **computed 重新计算** → `currentWorkspace`、`joinedProjectIds` 自动指向新 workspace
5. **组件重渲染** → 所有基于 `allowPermissions` 和 `joinedProjectIds` 的 UI 自动更新

**关键优势**：无需显式"清理"操作，命名空间设计天然隔离了不同 workspace 的数据。

### 5.3 加入/离开项目时的权限更新

**加入项目** (`base-permissions.store.ts:320-334`)：
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

**离开项目** (`base-permissions.store.ts:342-354`)：
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

---

## 六、核心文件索引

| 模块 | 文件路径 | 核心职责 |
|------|---------|---------|
| 权限常量 | `packages/constants/src/user.ts` | EUserPermissions 枚举定义 |
| 权限 Store | `apps/web/core/store/user/base-permissions.store.ts` | 权限判定核心逻辑、缓存管理 |
| Workspace Wrapper | `apps/web/core/layouts/auth-layout/workspace-wrapper.tsx` | Workspace 级权限初始化入口 |
| Project Wrapper | `apps/web/core/layouts/auth-layout/project-wrapper.tsx` | Project 级权限初始化入口 |
| 项目 Store | `apps/web/core/store/project/project.store.ts` | joinedProjectIds 数据过滤 |
| Fetch Keys | `apps/web/core/constants/fetch-keys.ts` | SWR 缓存命名空间定义 |
| SWR 配置 | `packages/constants/src/swr.ts` | 全局缓存策略配置 |
| 侧边栏项目 | `apps/web/core/components/workspace/sidebar/projects-list.tsx` | 项目列表渲染裁剪 |
| 侧边栏菜单 | `apps/web/core/components/workspace/sidebar/workspace-menu-item.tsx` | 导航项权限裁剪 |

---

## 七、设计亮点

1. **命名空间隔离**：所有缓存 key 包含 workspaceSlug，天然支持多 workspace 上下文共存
2. **计算属性驱动**：权限相关数据全部通过 MobX computed 派生，响应式更新
3. **统一判定入口**：`allowPermissions` 方法封装所有判定逻辑，确保一致性
4. **角色覆盖机制**：Workspace Admin 自动拥有所有项目权限，简化权限管理
5. **渐进式加载**：先加载部分项目列表，再按需加载完整详情，提升首屏速度
