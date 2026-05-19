# Intake 到 Issue 转换链路分析 - Round 4

## 一、内部 API partial_update 权限边界深度拆解

### 1.1 装饰器放行条件（第一层）

**文件**: `apps/api/plane/app/views/intake/base.py:327`

```python
@allow_permission(allowed_roles=[ROLE.ADMIN], creator=True, model=Issue)
def partial_update(self, request, slug, project_id, pk):
```

#### `@allow_permission` 装饰器逻辑分析

**文件**: `apps/api/plane/app/permissions/base.py:19-84`

放行路径（满足任意一条即可）：

**路径 A - 创建者放行**:
```
if creator and model:
    1. 检查用户是否是工作区成员（WorkspaceMember）
    2. 检查 Issue.objects.filter(id=pk, created_by=request.user) 是否存在
    3. 如果是创建者 → 直接放行，进入方法体
```

**路径 B - 角色放行**:
```
1. 检查 ProjectMember.role 是否在 [ROLE.ADMIN.value] 中（即 role == 20）
2. 或者：是项目成员 + 工作区 ADMIN
3. 满足 → 放行，进入方法体
```

**装饰器放行总结表**:

| 角色 | 是否创建者 | 装饰器放行 | 说明 |
|------|-----------|-----------|------|
| 项目 ADMIN (role=20) | 是/否 | ✅ 放行 | 满足路径 B |
| 工作区 ADMIN + 项目成员 | 是/否 | ✅ 放行 | 满足路径 B |
| 项目 MEMBER (role=15) | 是 | ✅ 放行 | 满足路径 A（创建者） |
| 项目 MEMBER (role=15) | 否 | ❌ 拒绝 | 路径 A 不满足（非创建者），路径 B 不满足（role=15 ≠ 20） |
| 项目 GUEST (role=5) | 是 | ✅ 放行 | 满足路径 A（创建者） |
| 项目 GUEST (role=5) | 否 | ❌ 拒绝 | 路径 A 不满足（非创建者），路径 B 不满足 |
| 非项目成员 | 是/否 | ❌ 拒绝 | 路径 A 需工作区成员，路径 B 不满足 |

> **关键发现 1**: 普通 MEMBER（非创建者）在装饰器层就被拒绝了，根本进不了方法体！
>
> **关键发现 2**: MEMBER 只有在是**创建者**的情况下才能通过装饰器。

---

### 1.2 方法内二次校验（第二层）

进入方法体后，还有三层校验：

#### 第一层：成员身份校验（340-358行）

```python
project_member = ProjectMember.objects.filter(...).first()
is_workspace_admin = WorkspaceMember.objects.filter(role=ROLE.ADMIN.value).exists()

if not project_member and not is_workspace_admin:
    return Response({"error": "Only admin or creator can update..."}, 403)
```

这一层实际上是冗余的（装饰器已经保证了要么是创建者+工作区成员，要么是 ADMIN），但提供了更明确的错误信息。

#### 第二层：GUEST 身份校验（360-367行）

```python
# Only project members admins and created_by users can access this endpoint
if ((project_member and project_member.role <= ROLE.GUEST.value) 
    and not is_workspace_admin) 
    and str(intake_issue.created_by_id) != str(request.user.id):
    return Response({"error": "You cannot edit intake issues"}, 400)
```

| 角色 | 是否创建者 | 校验结果 |
|------|-----------|---------|
| ADMIN | 是/否 | ✅ 通过（role=20 > GUEST.value=5） |
| MEMBER | 是/否 | ✅ 通过（role=15 > GUEST.value=5） |
| GUEST | 是 | ✅ 通过（是创建者） |
| GUEST | 否 | ❌ 拒绝（role <=5 且非创建者） |

> **注意**: 这一层对 MEMBER 没有限制，MEMBER 无论是否是创建者都能通过。但别忘了装饰器层已经过滤掉了非创建者的 MEMBER。

#### 第三层：字段级权限控制（369-424行）

这是最核心的权限拆分逻辑：

##### Issue 字段更新权限（370-413行）

```python
# Get issue data
issue_data = request.data.pop("issue", False)

if bool(issue_data):
    # 查询 Issue ...
    
    # GUEST 角色只能修改 name 和 description
    if project_member and project_member.role <= ROLE.GUEST.value:
        issue_data = {
            "name": issue_data.get("name", issue.name),
            "description_html": issue_data.get("description_html", issue.description_html),
            "description_json": issue_data.get("description_json", issue.description_json),
        }
    
    # 使用 IssueCreateSerializer 验证并保存
    issue_serializer = IssueCreateSerializer(issue, data=issue_data, partial=True, ...)
```

##### Intake 状态更新权限（414-424行）

```python
# Validate intake issue data if user has permission
if (project_member and project_member.role > ROLE.MEMBER.value) or is_workspace_admin:
    # role > 15 即 ADMIN (20)
    intake_serializer = IntakeIssueSerializer(intake_issue, data=request.data, partial=True)
```

> **核心发现**: `role > ROLE.MEMBER.value` → `role > 15`，只有 ADMIN (20) 满足！
>
> 这意味着：**只有 ADMIN 才能修改 IntakeIssue 的 status、duplicate_to、snoozed_till 等审核字段**。

---

### 1.3 各角色可达条件完整拆解

#### 角色 1：项目 ADMIN (role=20)

| 条件 | 值 |
|------|----|
| 装饰器放行 | ✅ 路径 B（角色匹配） |
| 方法内第一层校验 | ✅ 通过 |
| 方法内第二层校验 | ✅ 通过（role > 5） |
| 可更新 Issue 字段 | ✅ 所有字段（无过滤） |
| 可更新 Intake 状态 | ✅ 是（role > 15） |

**能力**:
- 可以修改 Issue 的所有字段（name、description、priority、assignees、labels 等）
- 可以修改 IntakeIssue 的状态（ACCEPTED/REJECTED/SNOOZED/DUPLICATE/PENDING）
- 可以设置 duplicate_to、snoozed_till 等

#### 角色 2：工作区 ADMIN + 项目成员

| 条件 | 值 |
|------|----|
| 装饰器放行 | ✅ 路径 B（工作区 ADMIN + 项目成员） |
| 方法内第一层校验 | ✅ 通过（is_workspace_admin=True） |
| 方法内第二层校验 | ✅ 通过（is_workspace_admin=True） |
| 可更新 Issue 字段 | ✅ 所有字段 |
| 可更新 Intake 状态 | ✅ 是（is_workspace_admin=True） |

**能力**: 与项目 ADMIN 完全相同。

#### 角色 3：项目 MEMBER (role=15) + 是创建者

| 条件 | 值 |
|------|----|
| 装饰器放行 | ✅ 路径 A（创建者） |
| 方法内第一层校验 | ✅ 通过（是项目成员） |
| 方法内第二层校验 | ✅ 通过（role > 5） |
| 可更新 Issue 字段 | ✅ 所有字段（无过滤） |
| 可更新 Intake 状态 | ❌ 否（role=15 不满足 >15） |

**能力**:
- ✅ 可以修改 Issue 的所有字段（name、description、priority 等）
- ❌ **不能**修改 IntakeIssue 的 status、duplicate_to、snoozed_till
- 只能改 Issue 内容，不能做审核决策

#### 角色 4：项目 MEMBER (role=15) + 非创建者

| 条件 | 值 |
|------|----|
| 装饰器放行 | ❌ 拒绝（路径 A 非创建者，路径 B 角色不匹配） |
| 方法内校验 | - |
| 可更新 Issue 字段 | - |
| 可更新 Intake 状态 | - |

**能力**: 完全无法访问此接口，装饰器层直接返回 403。

#### 角色 5：项目 GUEST (role=5) + 是创建者

| 条件 | 值 |
|------|----|
| 装饰器放行 | ✅ 路径 A（创建者） |
| 方法内第一层校验 | ✅ 通过（是项目成员） |
| 方法内第二层校验 | ✅ 通过（是创建者） |
| 可更新 Issue 字段 | ✅ 仅 name、description_html、description_json |
| 可更新 Intake 状态 | ❌ 否（role=5 不满足 >15） |

**能力**:
- ✅ 可以修改 Issue 的标题和描述
- ❌ 不能修改 Issue 的其他字段（priority、assignees、labels 等）
- ❌ 不能修改 IntakeIssue 的审核状态

#### 角色 6：项目 GUEST (role=5) + 非创建者

| 条件 | 值 |
|------|----|
| 装饰器放行 | ❌ 拒绝 |
| 方法内校验 | - |
| 可更新 Issue 字段 | - |
| 可更新 Intake 状态 | - |

**能力**: 完全无法访问此接口。

---

### 1.4 权限决策流程图

```
请求到达 partial_update
    ↓
@allow_permission(allowed_roles=[ADMIN], creator=True, model=Issue)
    ├─→ 是创建者？→ ✅ 进入方法体
    └─→ 是 ADMIN（项目或工作区）？→ ✅ 进入方法体
    └─→ 其他 → ❌ 403 拒绝
    ↓
进入方法体
    ↓
第一层：是项目成员或工作区 ADMIN？→ ❌ 403 拒绝
    ↓
第二层：是 GUEST 且非创建者？→ ❌ 400 拒绝
    ↓
第三层：字段级权限
    ├─→ 请求包含 issue 字段？
    │   ├─→ 是 GUEST？→ 过滤只保留 name/description
    │   └─→ 更新 Issue
    └─→ 请求包含 intake 状态字段（status/duplicate_to/snoozed_till）？
        ├─→ 是 ADMIN？→ 更新 IntakeIssue 状态
        └─→ 不是 ADMIN？→ 忽略这些字段（intake_serializer 为 None）
    ↓
返回结果
```

---

## 二、各角色操作能力总结表

### 2.1 partial_update 接口能力矩阵

| 角色 | 是创建者 | 可访问接口 | 可改 Issue 字段 | 可改 Intake 状态 | 可改字段详情 |
|------|---------|-----------|---------------|-----------------|-------------|
| 项目 ADMIN | ✅/❌ | ✅ | ✅ 全部 | ✅ 是 | 所有 Issue 字段 + status/duplicate_to/snoozed_till |
| 工作区 ADMIN | ✅/❌ | ✅ | ✅ 全部 | ✅ 是 | 同上 |
| 项目 MEMBER | ✅ | ✅ | ✅ 全部 | ❌ 否 | 所有 Issue 字段，但不能改审核状态 |
| 项目 MEMBER | ❌ | ❌ | - | - | 装饰器层拒绝 |
| 项目 GUEST | ✅ | ✅ | ✅ 受限 | ❌ 否 | 仅 name/description |
| 项目 GUEST | ❌ | ❌ | - | - | 装饰器层拒绝 |
| 非项目成员 | ✅/❌ | ❌ | - | - | 装饰器层拒绝 |

### 2.2 修正之前的错误结论

| 错误结论（之前轮次） | 修正结论（本轮） |
|---------------------|-----------------|
| MEMBER 可以改 Issue 内容 | ✅ 正确，但补充：**只有作为创建者的 MEMBER 才能访问接口**，非创建者 MEMBER 连接口都进不去 |
| MEMBER 不能改状态 | ✅ 正确 |
| GUEST 只能改自己的 name/description | ✅ 正确 |
| ADMIN 可以改所有 | ✅ 正确 |

---

## 三、公开 API 与内部 API 最终权限矩阵

### 3.1 公开 API (`/api/public/anchor/...`)

**基类**: `space/views/base.py` → `permission_classes = [IsAuthenticated]`
**ViewSet**: `IntakeIssuePublicViewSet` 未重写权限 → ✅ 所有端点必须登录

| 操作 | 登录要求 | 额外权限 | 可操作字段 |
|------|---------|---------|-----------|
| GET list | ✅ 必须登录 | 无 | 只读 |
| POST create | ✅ 必须登录 | 无 | name, description_json, description_html, priority |
| GET retrieve | ✅ 必须登录 | 无 | 只读 |
| PATCH partial_update | ✅ 必须登录 | 仅创建者 | 仅 name, description_html, description_json |
| DELETE destroy | ✅ 必须登录 | 仅创建者 | - |

**公开 API 无审核能力**: 没有修改 status/duplicate_to/snoozed_till 的逻辑。

### 3.2 内部 API (`/api/workspaces/...`)

**基类**: `app/views/base.py` → `permission_classes = [IsAuthenticated]`
**ViewSet**: `IntakeIssueViewSet` + `@allow_permission` 装饰器

#### 3.2.1 create 接口

```python
@allow_permission([ROLE.ADMIN, ROLE.MEMBER, ROLE.GUEST])
def create(self, request, slug, project_id):
```

| 角色 | 可访问 | 说明 |
|------|-------|------|
| ADMIN | ✅ |  |
| MEMBER | ✅ |  |
| GUEST | ✅ |  |
| 非成员 | ❌ |  |

#### 3.2.2 list 接口

```python
@allow_permission([ROLE.ADMIN, ROLE.MEMBER, ROLE.GUEST])
def list(self, request, slug, project_id):
```

方法内额外逻辑：
```python
if (ProjectMember.objects.filter(role=ROLE.GUEST.value).exists() 
    and not project.guest_view_all_features):
    intake_issue = intake_issue.filter(created_by=request.user)
```

| 角色 | 可访问 | 可见范围 |
|------|-------|---------|
| ADMIN | ✅ | 全部 |
| MEMBER | ✅ | 全部 |
| GUEST | ✅ | 仅自己创建的（如果 guest_view_all_features=False） |

#### 3.2.3 retrieve 接口

```python
@allow_permission(allowed_roles=[ROLE.ADMIN, ROLE.MEMBER, ROLE.GUEST], creator=True, model=Issue)
def retrieve(self, request, slug, project_id, pk):
```

方法内额外逻辑同 list，GUEST 只能看自己的。

#### 3.2.4 partial_update 接口（已详细拆解）

| 角色 | 是创建者 | 可访问 | 可改 Issue | 可改 Intake 状态 |
|------|---------|-------|-----------|-----------------|
| ADMIN | ✅/❌ | ✅ | ✅ 全部 | ✅ 是 |
| 工作区 ADMIN | ✅/❌ | ✅ | ✅ 全部 | ✅ 是 |
| MEMBER | ✅ | ✅ | ✅ 全部 | ❌ 否 |
| MEMBER | ❌ | ❌ | - | - |
| GUEST | ✅ | ✅ | ✅ 仅 name/description | ❌ 否 |
| GUEST | ❌ | ❌ | - | - |

#### 3.2.5 destroy 接口

```python
@allow_permission(allowed_roles=[ROLE.ADMIN], creator=True, model=Issue)
def destroy(self, request, slug, project_id, pk):
```

方法内额外逻辑：
```python
if intake_issue.status in [-2, -1, 0, 2]:  # PENDING/REJECTED/SNOOZED/DUPLICATE
    issue.delete()  # 级联删除 Issue
```

| 角色 | 是创建者 | 可删除 | 级联删除 Issue |
|------|---------|-------|--------------|
| ADMIN | ✅/❌ | ✅ | 状态非 ACCEPTED 时删除 |
| MEMBER | ✅ | ❌ | - |
| MEMBER | ❌ | ❌ | - |
| GUEST | ✅ | ✅ | 状态非 ACCEPTED 时删除 |
| GUEST | ❌ | ❌ | - |

> **注意**: GUEST 创建者可以删除自己的 Intake Issue，这与之前的理解一致。但 MEMBER 创建者**不能**删除，因为装饰器只允许 ADMIN 或创建者，但 MEMBER 作为创建者应该能通过路径 A... 等一下，让我再看：

```python
@allow_permission(allowed_roles=[ROLE.ADMIN], creator=True, model=Issue)
def destroy(self, request, slug, project_id, pk):
```

是的，`creator=True`，所以 MEMBER 作为创建者应该能通过路径 A 放行。让我验证：

路径 A（创建者）:
1. 是工作区成员 → 是的（MEMBER 是项目成员，必然是工作区成员）
2. Issue 是自己创建的 → 是的
3. → 放行 ✅

所以 MEMBER 创建者**可以**删除自己的 Intake Issue。

**修正后的 destroy 权限表**:

| 角色 | 是创建者 | 可删除 | 说明 |
|------|---------|-------|------|
| ADMIN | ✅/❌ | ✅ | 路径 B（角色匹配） |
| 工作区 ADMIN | ✅/❌ | ✅ | 路径 B |
| MEMBER | ✅ | ✅ | 路径 A（创建者） |
| MEMBER | ❌ | ❌ | 两条路径都不满足 |
| GUEST | ✅ | ✅ | 路径 A（创建者） |
| GUEST | ❌ | ❌ | 两条路径都不满足 |

---

## 四、完整权限体系总览

### 4.1 审核状态变更权限（核心业务能力）

| 操作 | 需要的最低权限 | 说明 |
|------|--------------|------|
| 提交 Intake Issue | GUEST | 任何登录用户 |
| 编辑 Issue 内容（自己的） | GUEST | 仅 name/description |
| 编辑 Issue 内容（所有） | MEMBER（创建者）或 ADMIN | MEMBER 必须是创建者 |
| 标记 ACCEPTED（接受） | ADMIN | 项目或工作区管理员 |
| 标记 REJECTED（拒绝） | ADMIN | 同上 |
| 标记 SNOOZED（暂停） | ADMIN | 同上 |
| 标记 DUPLICATE（重复） | ADMIN | 同上 |
| 删除 Intake Issue | ADMIN 或创建者（任意角色） | 创建者可以是 MEMBER/GUEST |

### 4.2 关键代码位置

| 模块 | 文件路径 | 关键代码 |
|------|----------|---------|
| 装饰器权限 | `apps/api/plane/app/permissions/base.py` | `allow_permission()` 函数 |
| partial_update 装饰器 | `apps/api/plane/app/views/intake/base.py:327` | `@allow_permission(allowed_roles=[ROLE.ADMIN], creator=True, model=Issue)` |
| 方法内字段级权限 | `apps/api/plane/app/views/intake/base.py:369-424` | GUEST 字段过滤、ADMIN 状态更新判断 |
| destroy 装饰器 | `apps/api/plane/app/views/intake/base.py:545` | `@allow_permission(allowed_roles=[ROLE.ADMIN], creator=True, model=Issue)` |
| 公开 API 基类权限 | `apps/api/plane/space/views/base.py:48` | `permission_classes = [IsAuthenticated]` |

---

## 五、修正总结（Round 3 → Round 4）

### 5.1 partial_update 权限修正

| 项 | Round 3 结论 | Round 4 修正结论 |
|----|------------|-----------------|
| MEMBER 可访问性 | 模糊，未区分是否创建者 | ✅ 只有**作为创建者的 MEMBER** 才能访问接口；非创建者 MEMBER 在装饰器层被拒绝 |
| MEMBER 可改内容 | 说 MEMBER 只能改 Issue 内容 | ✅ 正确，但补充：前提是 MEMBER 是创建者 |
| GUEST 可访问性 | 正确 | ✅ 正确，只有创建者 GUEST 可访问 |
| ADMIN 能力 | 正确 | ✅ 正确 |

### 5.2 destroy 权限修正

| 项 | Round 3 结论 | Round 4 修正结论 |
|----|------------|-----------------|
| MEMBER 创建者可删除 | ❌ 说不能 | ✅ 能删除（creator=True 路径放行） |
