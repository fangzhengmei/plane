# Guest 加入按钮修复落地方案

## 一、修复原则

**最小改动优先**：只修改必要的代码，避免大范围重构，降低回归风险。

**改动顺序**：从"最容易验证、风险最低"的改动开始，逐步推进。

---

## 二、改动路线图（按优先级排序）

### 阶段 1：核心修复（2 个文件，立即实施）

#### 改动 1.1：ProjectAccessRestriction 新增 Guest 门控

**文件**: `apps/web/core/components/auth-screens/project/project-access-restriction.tsx`

**改动要点**：

```typescript
// 1. 更新 Props 类型，新增 canSelfJoin
type TProps = {
  isWorkspaceAdmin: boolean;
  canSelfJoin: boolean;  // 新增：用户是否有权限主动加入项目
  handleJoinProject: () => void;
  isJoinButtonDisabled: boolean;
  errorStatusCode: number | undefined;
};

// 2. 更新显示条件
// 原代码：
// if (errorStatusCode === 409 || (errorStatusCode === 403 && isWorkspaceAdmin))
// 修改为：
if ((errorStatusCode === 409 && canSelfJoin) || (errorStatusCode === 403 && isWorkspaceAdmin))
  return (
    <EmptyStateDetailed
      // ... 原有渲染逻辑不变
    />
  );

// 3. 新增：Guest 访问 PUBLIC 项目场景
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

**Why 先改这个**：
- 这是问题的核心：显示逻辑与实际权限不匹配
- 改动完全在组件内部，不影响其他模块
- 可以独立测试（只需传入不同的 `canSelfJoin` prop）

#### 改动 1.2：ProjectAuthWrapper 计算并传递 canSelfJoin

**文件**: `apps/web/core/layouts/auth-layout/project-wrapper.tsx`

**改动要点**：

```typescript
// 1. 从 store 获取用户 workspace 角色（已有这个方法，直接调用）
const workspaceUserRole = getWorkspaceRoleByWorkspaceSlug(workspaceSlug);

// 2. 计算 canSelfJoin：只有 ADMIN 和 MEMBER 可以主动加入
const canSelfJoin = workspaceUserRole === EUserPermissions.ADMIN || workspaceUserRole === EUserPermissions.MEMBER;

// 3. 传递给 ProjectAccessRestriction
if (!isProjectLoading && hasPermissionToCurrentProject === false) {
  return (
    <ProjectAccessRestriction
      errorStatusCode={projectDetailsError?.status}
      isWorkspaceAdmin={isWorkspaceAdmin}
      canSelfJoin={canSelfJoin}  // 新增
      handleJoinProject={handleJoinProject}
      isJoinButtonDisabled={isJoiningProject}
    />
  );
}
```

**Why 第二个改这个**：
- 依赖改动 1.1 的 Props 变更
- 只涉及 props 传递，逻辑简单
- 端到端可测试

### 阶段 2：文案完善（3 个文件，可与阶段 1 并行）

#### 改动 2.1：中文文案

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

#### 改动 2.2：英文文案

**文件**: `packages/i18n/src/locales/en/empty-state.json`

```json
{
  "project_empty_state": {
    "no_access": {
      "title": "No project access",
      "join_description": "Click the button below to join the project.",
      "restricted_description": "Contact an administrator to request access, and you can continue here once approved.",
      "guest_join_description": "As a guest user, you need to contact the project administrator to request an invitation."
    }
  }
}
```

#### 改动 2.3：其他语言（至少同步繁体中文）

**文件**: `packages/i18n/src/locales/zh-TW/empty-state.json`

```json
{
  "project_empty_state": {
    "no_access": {
      "guest_join_description": "作為訪客使用者，您需要聯繫專案管理員邀請您加入此專案。"
    }
  }
}
```

**Why 可以并行**：
- 文案改动不依赖代码逻辑变更
- 即使文案没改完，代码逻辑也能正常工作（会显示翻译 key 或 fallback 到英文）

### 阶段 3：防御性增强（可选，低优先级）

#### 改动 3.1：joinProject 增加前置检查（推荐但非必须）

**文件**: `apps/web/core/store/user/base-permissions.store.ts`

```typescript
joinProject = async (workspaceSlug: string, projectId: string): Promise<void> => {
  // 新增：前置权限检查，避免不必要的 API 请求
  const workspaceRole = this.getWorkspaceRoleByWorkspaceSlug(workspaceSlug);
  if (workspaceRole === EUserPermissions.GUEST) {
    console.warn("Guest users cannot join projects directly");
    throw new Error("Guest users cannot join projects directly");
  }

  // ... 原有逻辑不变
};
```

**Why 可选**：
- 这是"双重保险"，阶段 1 的改动已经阻止了 Guest 看到按钮
- 但加上可以防止其他调用路径（如通过控制台调用）
- 不影响核心功能修复

---

## 三、改动文件汇总

| 阶段 | 优先级 | 文件路径 | 改动类型 | 代码行数 |
|------|--------|---------|---------|---------|
| 1 | P0 | `apps/web/core/components/auth-screens/project/project-access-restriction.tsx` | 核心逻辑 | ~15 行新增/修改 |
| 1 | P0 | `apps/web/core/layouts/auth-layout/project-wrapper.tsx` | Props 传递 | ~5 行新增 |
| 2 | P1 | `packages/i18n/src/locales/zh-CN/empty-state.json` | 文案 | 1 行新增 |
| 2 | P1 | `packages/i18n/src/locales/en/empty-state.json` | 文案 | 1 行新增 |
| 2 | P1 | `packages/i18n/src/locales/zh-TW/empty-state.json` | 文案 | 1 行新增 |
| 3 | P2 | `apps/web/core/store/user/base-permissions.store.ts` | 防御性检查 | ~5 行新增 |

**总计**：6 个文件，~28 行代码改动

---

## 四、回归检查清单

### 测试矩阵：角色 × 项目类型 × 成员状态

| # | 角色 | 项目类型 | 成员状态 | 操作路径 | 预期结果 |
|---|------|---------|---------|---------|---------|
| 1 | Guest | PUBLIC | 非成员 | 直接访问项目 URL | 显示"作为访客用户，需联系管理员邀请"，**无按钮** |
| 2 | Guest | PUBLIC | 是成员 | 正常进入项目 | 正常渲染项目页面 |
| 3 | Guest | SECRET | 非成员 | 直接访问项目 URL | 显示"无权限"，**无按钮**（后端返回 403） |
| 4 | Guest | SECRET | 是成员 | 正常进入项目 | 正常渲染项目页面 |
| 5 | Member | PUBLIC | 非成员 | 直接访问项目 URL | 显示"加入项目"按钮，点击后成功加入 |
| 6 | Member | PUBLIC | 是成员 | 正常进入项目 | 正常渲染项目页面 |
| 7 | Member | SECRET | 非成员 | 直接访问项目 URL | 显示"无权限"，**无按钮**（后端返回 403） |
| 8 | Member | SECRET | 是成员 | 正常进入项目 | 正常渲染项目页面 |
| 9 | Admin | PUBLIC | 非成员 | 直接访问项目 URL | 显示"加入项目"按钮，点击后成功加入 |
| 10 | Admin | PUBLIC | 是成员 | 正常进入项目 | 正常渲染项目页面 |
| 11 | Admin | SECRET | 非成员 | 直接访问项目 URL | 显示"加入项目"按钮（403 + isWorkspaceAdmin），点击后成功加入 |
| 12 | Admin | SECRET | 是成员 | 正常进入项目 | 正常渲染项目页面 |

### 关键检查点

#### 前端显示检查

- [ ] **#1 通过**：Guest 访问 PUBLIC 非成员项目，不显示加入按钮
- [ ] **#5 通过**：Member 访问 PUBLIC 非成员项目，显示加入按钮
- [ ] **#7 通过**：Member 访问 SECRET 非成员项目，不显示加入按钮
- [ ] **#11 通过**：Admin 访问 SECRET 非成员项目，显示加入按钮

#### 功能操作检查

- [ ] **#5 点击加入**：Member 点击加入 PUBLIC 项目，成功加入并自动进入项目
- [ ] **#9 点击加入**：Admin 点击加入 PUBLIC 项目，成功加入并自动进入项目
- [ ] **#11 点击加入**：Admin 点击加入 SECRET 项目，成功加入并自动进入项目

#### 回归保护检查

- [ ] **#2, 4, 6, 8, 10, 12 通过**：已成员用户访问项目，不受影响，正常进入
- [ ] **侧边栏项目列表**：Guest 只看到已加入的项目（回归可见性过滤逻辑）
- [ ] **项目设置-成员管理**：Admin 仍可以邀请 Guest 加入项目（回归邀请流程）
- [ ] **邀请链接加入**：Guest 通过邀请链接加入项目的流程不受影响（这个路径走的是 `ProjectJoinEndpoint`，不是 `joinProject` 接口）

---

## 五、上线顺序建议

### 第 1 步：本地开发验证（开发者）
1. 改动阶段 1 的 2 个核心文件
2. 启动 `pnpm dev`
3. 用 3 个测试账号（Guest/Member/Admin）验证测试矩阵中的关键场景
4. 确认修复生效且无明显回归

### 第 2 步：文案 PR（可以单独合入）
1. 提交中文/英文/繁体中文的文案改动
2. 这部分可以独立合入，不影响功能

### 第 3 步：核心修复 PR
1. 提交阶段 1 的代码改动
2. 附上测试结果截图（至少覆盖 #1, #5, #7, #11）

### 第 4 步：防御性增强（可选，后续迭代）
1. 如果时间充裕，提交阶段 3 的改动
2. 这是锦上添花，不影响核心修复

### 第 5 步：测试环境验证（QA）
1. QA 按照回归检查清单执行完整测试
2. 重点关注 Guest 场景和 SECRET 项目场景

### 第 6 步：生产发布
1. 合并到主分支
2. 灰度发布或全量发布

---

## 六、风险与回滚

### 风险评估

| 风险 | 概率 | 影响 | 缓解措施 |
|------|------|------|---------|
| 改动导致已成员用户无法访问项目 | 低 | 高 | 回归检查 #2, 4, 6, 8, 10, 12 重点覆盖 |
| 其他使用 ProjectAccessRestriction 的地方受影响 | 低 | 中 | 全局搜索 `<ProjectAccessRestriction`，确认只有 ProjectAuthWrapper 使用 |
| 翻译 key 不存在导致显示异常 | 中 | 低 | i18n 库会 fallback 到英文或显示 key，不影响功能 |

### 回滚方案

如果上线后发现问题：
1. 立即回滚核心修复 PR（阶段 1 的 2 个文件）
2. 文案改动可以保留（只是新增了一个 key，不影响现有逻辑）
3. 分析问题原因后重新提交

---

## 七、改动前后对比

### 改动前（问题场景）
```
Guest → 访问 PUBLIC 项目 URL（非成员）
    ↓
后端返回 409 Conflict
    ↓
前端判断：409 === 409 → true
    ↓
显示"加入项目"按钮
    ↓
用户点击 → 后端 403 → 用户困惑
```

### 改动后（修复后）
```
Guest → 访问 PUBLIC 项目 URL（非成员）
    ↓
后端返回 409 Conflict
    ↓
ProjectAuthWrapper 计算：
  workspaceUserRole = GUEST
  canSelfJoin = GUEST === ADMIN || GUEST === MEMBER → false
    ↓
ProjectAccessRestriction 判断：
  (409 === 409 && false) || ... → false
  409 === 409 && !false → true
    ↓
显示"作为访客用户，需联系管理员邀请"
    ↓
用户理解，无困惑
```
