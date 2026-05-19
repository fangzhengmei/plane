# Guest 加入按钮不匹配问题分析

## 一、问题概述

**场景**：Guest 用户通过直接输入 URL 访问一个他未加入的 PUBLIC 项目

**预期行为**：要么不显示"加入项目"按钮，要么点击后能成功加入

**实际行为**：
1. 页面显示"加入项目"按钮
2. 用户点击按钮
3. 后端返回 403 Forbidden 错误
4. 用户体验困惑（按钮看起来可用但实际不可用）

---

## 二、完整调用链分析

### 2.1 按钮显示条件（前端）

**文件**: `apps/web/core/components/auth-screens/project/project-access-restriction.tsx:27-28`
```typescript
// Show join project screen if:
// - User lacks project membership (409 Conflict)
// - User lacks permission to access the private project (403 Forbidden) but is a workspace admin (can join any project)
if (errorStatusCode === 409 || (errorStatusCode === 403 && isWorkspaceAdmin))
  return (
    // ... 渲染带加入按钮的空状态
  );
```

**按钮显示的条件表达式**：
```
errorStatusCode === 409 || (errorStatusCode === 403 && isWorkspaceAdmin)
```

**对于 Guest 用户访问 PUBLIC 项目（非成员）**：
- 后端返回 `409 Conflict`（因为是 PUBLIC 项目，非成员返回 409 而不是 403）
- 条件 `errorStatusCode === 409` 为 **true**
- **按钮显示** ✅

### 2.2 按钮点击后的调用链

**步骤 1：ProjectAuthWrapper 中的处理函数**
**文件**: `apps/web/core/layouts/auth-layout/project-wrapper.tsx:140-143`
```typescript
const handleJoinProject = () => {
  setIsJoiningProject(true);
  joinProject(workspaceSlug, projectId).finally(() => setIsJoiningProject(false));
};
```

**步骤 2：前端 store 调用**
**文件**: `apps/web/core/store/user/base-permissions.store.ts:320-334`
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

**步骤 3：API 请求发送**
**文件**: `apps/web/core/services/user.service.ts:253-259`
```typescript
async joinProject(workspaceSlug: string, project_ids: string[]): Promise<any> {
  return this.post(`/api/users/me/workspaces/${workspaceSlug}/projects/invitations/`, { project_ids })
}
```

**步骤 4：后端路由匹配**
**文件**: `apps/api/plane/app/urls/project.py:63-66`
```python
path(
    "users/me/workspaces/<str:slug>/projects/invitations/",
    UserProjectInvitationsViewset.as_view({"get": "list", "post": "create"}),
    name="user-project-invitations",
),
```

**步骤 5：后端装饰器拦截**
**文件**: `apps/api/plane/app/views/project/invite.py:128-129`
```python
class UserProjectInvitationsViewset(BaseViewSet):
    ...
    @allow_permission([ROLE.ADMIN, ROLE.MEMBER], level="WORKSPACE")
    def create(self, request, slug):
```

**步骤 6：权限判定失败**
**文件**: `apps/api/plane/app/permissions/base.py:44-51`
```python
if level == "WORKSPACE":
    if WorkspaceMember.objects.filter(
        member=request.user,
        workspace__slug=kwargs["slug"],
        role__in=allowed_role_values,  # [20, 15] - ADMIN, MEMBER
        is_active=True,
    ).exists():
        return view_func(instance, request, *args, **kwargs)

# 未通过检查，返回 403
return Response(
    {"error": "You don't have the required permissions."},
    status=status.HTTP_403_FORBIDDEN,
)
```

**步骤 7：前端收到错误**
```
HTTP 403 Forbidden
{ "error": "You don't have the required permissions." }
```

### 2.3 问题根因总结

```
前端显示逻辑：409 → 显示按钮
        ↓
后端实际逻辑：Guest 不能调用 joinProject（装饰器只允许 ADMIN/MEMBER）
        ↓
不匹配：按钮对 Guest 可见，但点击后失败
```

---

## 三、调用链可视化

```
Guest 访问 /{workspaceSlug}/projects/{projectId}/ (PUBLIC 项目，非成员)
        ↓
[ProjectAuthWrapper]
  ├─ fetchProjectDetails()
  │   └─ GET /api/workspaces/{slug}/projects/{projectId}/
  │       └─ 后端判定：PUBLIC + 非成员 → 409 Conflict
  ├─ hasPermissionToCurrentProject = allowPermissions([ADMIN, MEMBER, GUEST], PROJECT, ...)
  │   └─ workspaceProjectsPermissions[slug][projectId] 不存在 → false
  └─ isWorkspaceAdmin = allowPermissions([ADMIN], WORKSPACE, slug)
      └─ workspaceUserInfo[slug].role = 5 (GUEST) → false
        ↓
[ProjectAccessRestriction]
  props: errorStatusCode=409, isWorkspaceAdmin=false
  └─ 条件判断：409 === 409 → true → 显示"加入项目"按钮
        ↓
用户点击"加入项目"
        ↓
handleJoinProject() → joinProject(workspaceSlug, projectId)
        ↓
POST /api/users/me/workspaces/{slug}/projects/invitations/
        ↓
[后端 @allow_permission([ADMIN, MEMBER], WORKSPACE)]
  └─ User.role = 5 (GUEST) ∉ [20, 15] → 403 Forbidden
        ↓
前端收到错误 → 用户困惑
```

---

## 四、修正方案一：前端显示门控

### 4.1 方案思路

在显示"加入项目"按钮前，额外检查用户是否为 Guest。如果是 Guest，不显示按钮，而是显示相应的提示文本。

### 4.2 具体改动点

#### 改动 1：ProjectAuthWrapper 传递用户角色信息

**文件**: `apps/web/core/layouts/auth-layout/project-wrapper.tsx:75`
```typescript
// 当前代码
const isWorkspaceAdmin = allowPermissions([EUserPermissions.ADMIN], EUserPermissionsLevel.WORKSPACE, workspaceSlug);

// 新增：获取用户 workspace 角色
const workspaceUserRole = getWorkspaceRoleByWorkspaceSlug(workspaceSlug);
```

#### 改动 2：将角色传递给 ProjectAccessRestriction

**文件**: `apps/web/core/layouts/auth-layout/project-wrapper.tsx:149-157`
```typescript
// 当前代码
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

// 新增：传递 workspaceUserRole
if (!isProjectLoading && hasPermissionToCurrentProject === false) {
  return (
    <ProjectAccessRestriction
      errorStatusCode={projectDetailsError?.status}
      isWorkspaceAdmin={isWorkspaceAdmin}
      workspaceUserRole={workspaceUserRole}
      handleJoinProject={handleJoinProject}
      isJoinButtonDisabled={isJoiningProject}
    />
  );
}
```

#### 改动 3：更新 ProjectAccessRestriction 的 Props 类型

**文件**: `apps/web/core/components/auth-screens/project/project-access-restriction.tsx:12-17`
```typescript
// 当前代码
type TProps = {
  isWorkspaceAdmin: boolean;
  handleJoinProject: () => void;
  isJoinButtonDisabled: boolean;
  errorStatusCode: number | undefined;
};

// 新增
type TProps = {
  isWorkspaceAdmin: boolean;
  workspaceUserRole?: EUserPermissions | EUserWorkspaceRoles;
  handleJoinProject: () => void;
  isJoinButtonDisabled: boolean;
  errorStatusCode: number | undefined;
};
```

#### 改动 4：更新按钮显示条件

**文件**: `apps/web/core/components/auth-screens/project/project-access-restriction.tsx:27-28`
```typescript
// 当前代码
if (errorStatusCode === 409 || (errorStatusCode === 403 && isWorkspaceAdmin))

// 新增：检查不是 Guest
const canSelfJoin = workspaceUserRole !== EUserWorkspaceRoles.GUEST;

if ((errorStatusCode === 409 && canSelfJoin) || (errorStatusCode === 403 && isWorkspaceAdmin))
```

#### 改动 5：为 Guest 显示不同的提示

**文件**: `apps/web/core/components/auth-screens/project/project-access-restriction.tsx:46-61`
```typescript
// 在显示加入按钮的分支之后，新增 Guest 场景
if (errorStatusCode === 409 && !canSelfJoin)
  return (
    <div className="grid h-full w-full place-items-center bg-surface-1">
      <EmptyStateDetailed
        title={t("project_empty_state.no_access.title")}
        description={t("project_empty_state.no_access.guest_join_description")}
        assetKey="no-access"
        assetClassName="size-40"
      />
    </div>
  );
```

#### 改动 6：新增翻译文案

**文件**: `packages/i18n/src/locales/zh-CN/empty-state.json`
```json
{
  "project_empty_state": {
    "no_access": {
      "title": "无项目访问权限",
      "join_description": "点击下方按钮加入该项目。",
      "restricted_description": "请联系管理员申请访问权限，通过后您可以在此继续。",
      "guest_join_description": "作为访客用户，您需要联系项目管理员邀请您加入此项目。"
    }
  }
}
```

### 4.3 方案风险

| 风险点 | 影响 | 缓解措施 |
|--------|------|---------|
| `getWorkspaceRoleByWorkspaceSlug` 返回 `undefined`（数据未加载完成） | 可能误判为非 Guest 而显示按钮 | 在 ProjectAuthWrapper 中确保权限数据加载完成后再渲染 |
| 翻译文案需要多语言支持 | 部分语言可能缺失翻译 | 遵循现有 i18n 流程，同步更新所有语言包 |
| 类型变更可能影响其他调用方 | 如果其他地方也使用了 ProjectAccessRestriction | 搜索所有使用点，确保兼容性 |

### 4.4 改动范围评估

- **文件数量**：6 个文件（1 个组件、1 个 wrapper、1 个类型、3 个语言包）
- **代码行数**：约 30-40 行新增/修改
- **测试影响**：需要针对 Guest 场景新增 UI 测试用例

---

## 五、修正方案二：后端提示优化

### 5.1 方案思路

不在前端隐藏按钮，而是让后端返回更有意义的错误信息，前端根据错误信息显示友好提示。这种方案保留按钮，但改善错误体验。

### 5.2 具体改动点

#### 改动 1：后端 joinProject 接口对 Guest 返回特定错误码

**文件**: `apps/api/plane/app/views/project/invite.py:128-135`
```python
# 当前代码
@allow_permission([ROLE.ADMIN, ROLE.MEMBER], level="WORKSPACE")
def create(self, request, slug):

# 方案 A：放宽装饰器，在方法内检查并返回特定错误
# 移除装饰器，改为方法内检查
def create(self, request, slug):
    workspace_member = WorkspaceMember.objects.get(member=request.user, workspace__slug=slug, is_active=True)
    
    # Guest 用户不能加入项目
    if workspace_member.role == ROLE.GUEST.value:
        return Response(
            {
                "error": "Guest users cannot join projects directly",
                "error_code": "GUEST_CANNOT_JOIN_PROJECT",
                "message": "Please contact the project administrator to request an invitation."
            },
            status=status.HTTP_403_FORBIDDEN,
        )
    
    # ... 原有逻辑
```

#### 改动 2：前端 ProjectAuthWrapper 捕获并处理特定错误

**文件**: `apps/web/core/layouts/auth-layout/project-wrapper.tsx:140-143`
```typescript
// 当前代码
const handleJoinProject = () => {
  setIsJoiningProject(true);
  joinProject(workspaceSlug, projectId).finally(() => setIsJoiningProject(false));
};

// 新增：捕获特定错误并显示友好提示
const handleJoinProject = async () => {
  setIsJoiningProject(true);
  try {
    await joinProject(workspaceSlug, projectId);
  } catch (error: any) {
    if (error?.error_code === "GUEST_CANNOT_JOIN_PROJECT") {
      // 显示 toast 提示
      setJoinError(error.message);
    }
  } finally {
    setIsJoiningProject(false);
  }
};
```

#### 改动 3：在 ProjectAccessRestriction 中显示错误提示

**文件**: `apps/web/core/components/auth-screens/project/project-access-restriction.tsx`
```typescript
// 新增 Props
type TProps = {
  // ... 原有 props
  joinError?: string;
};

// 在按钮下方显示错误
{joinError && (
  <p className="text-error text-sm mt-2">{joinError}</p>
)}
```

### 5.3 方案风险

| 风险点 | 影响 | 缓解措施 |
|--------|------|---------|
| 放宽装饰器可能引入安全漏洞 | 如果方法内的检查逻辑有 bug，Guest 可能成功加入 | 代码审查时重点关注权限检查逻辑，确保在所有分支前执行 |
| 错误码与前端耦合 | 如果后续修改错误码，前端逻辑会失效 | 定义常量或枚举，避免硬编码字符串 |
| 用户仍然看到不可用的按钮 | 体验不够完美（按钮可见但点击后报错） | 这是本方案的固有 trade-off，相比方案一体验稍差 |

### 5.4 改动范围评估

- **文件数量**：3 个文件（1 个后端视图、1 个 wrapper、1 个组件）
- **代码行数**：约 20-30 行新增/修改
- **测试影响**：需要针对错误场景新增 API 测试和 UI 测试

---

## 六、两套方案对比与优先级建议

### 6.1 方案对比矩阵

| 评估维度 | 方案一：前端门控 | 方案二：后端提示优化 |
|---------|-----------------|-------------------|
| **用户体验** | ⭐⭐⭐⭐⭐ 最佳（看不到不可用按钮） | ⭐⭐⭐ 良好（按钮可见但有友好错误提示） |
| **改动复杂度** | 中等（多文件联动） | 简单（集中在后端） |
| **安全风险** | 无（仅影响显示） | 低（需谨慎处理权限检查） |
| **可维护性** | 高（逻辑集中在显示层） | 中（前后端错误码耦合） |
| **向后兼容** | 高（仅新增逻辑） | 高（仅新增错误码） |
| **国际化成本** | 高（需新增多语言文案） | 低（错误信息在后端或前端统一处理） |
| **上线风险** | 低 | 中（涉及权限逻辑变更） |

### 6.2 优先级建议

#### 推荐：方案一（前端显示门控）**优先级：P0（高）**

**理由**：
1. **用户体验最优**：遵循"不要让用户点击不可用的按钮"的 UX 原则
2. **安全无风险**：纯前端显示逻辑，不涉及权限变更
3. **符合现有架构**：权限判定发起点本来就应该在前端做门控
4. **改动范围可控**：虽然涉及多个文件，但都是显示层逻辑，风险低

#### 备选：方案二（后端提示优化）**优先级：P2（低）**

**适用场景**：
- 如果团队认为"Guest 将来可能被允许加入项目"，保留按钮更具前瞻性
- 作为方案一的补充（即使做了方案一，后端返回清晰的错误码也是好的实践）

### 6.3 实施建议

**短期（立即实施）**：
1. 实施方案一（前端门控）
2. 补充 Guest 场景的测试用例

**长期（后续优化）**：
1. 考虑方案二中的后端错误码标准化（不用于此问题，但对整体系统有益）
2. 考虑是否从产品层面允许 Guest 申请加入（而不是必须邀请）

---

## 七、核心文件索引

| 文件 | 作用 | 改动点 |
|------|------|--------|
| `apps/web/core/components/auth-screens/project/project-access-restriction.tsx` | 无权限页面渲染 | 按钮显示条件、Guest 提示文案 |
| `apps/web/core/layouts/auth-layout/project-wrapper.tsx` | 项目权限包装器 | 传递用户角色、错误处理 |
| `apps/api/plane/app/views/project/invite.py` | 后端加入项目接口 | 权限检查、错误码（方案二） |
| `packages/i18n/src/locales/*/empty-state.json` | 多语言文案 | 新增 Guest 提示文案（方案一） |
| `apps/api/plane/app/permissions/base.py` | 权限装饰器 | 放宽装饰器（方案二，不推荐） |
