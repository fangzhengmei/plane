# Intake 到 Issue 转换链路分析 - Round 3

## 一、公开入口认证边界深度校准

### 1.1 权限控制基类溯源

**公开 API 基类**: `apps/api/plane/space/views/base.py:45-52`

```python
class BaseViewSet(TimezoneMixin, ModelViewSet, BasePaginator):
    model = None
    permission_classes = [IsAuthenticated]  # ⚠️ 默认要求认证！
    authentication_classes = [BaseSessionAuthentication]
```

> **重要修正**: 公开 API 的基类 `BaseViewSet` 默认设置了 `permission_classes = [IsAuthenticated]`，这意味着**所有继承它的 ViewSet 默认都要求登录认证**。

### 1.2 其他公开 View 的权限模式对比

通过搜索发现，其他公开 View 都**显式重写**了 `permission_classes` 为 `AllowAny`：

| ViewSet/Endpoint | 文件 | 权限设置 |
|-----------------|------|---------|
| `ProjectIssuesPublicEndpoint` | `space/views/issue.py:74` | `permission_classes = [AllowAny]` |
| `ProjectPublicEndpoint` | `space/views/project.py:20` | `permission_classes = [AllowAny]` |
| `ProjectStatePublicEndpoint` | `space/views/state.py:19` | `permission_classes = [AllowAny]` |
| `ProjectLabelPublicEndpoint` | `space/views/label.py:16` | `permission_classes = [AllowAny]` |
| `ProjectCyclePublicEndpoint` | `space/views/cycle.py:16` | `permission_classes = [AllowAny]` |
| `ProjectModulePublicEndpoint` | `space/views/module.py:16` | `permission_classes = [AllowAny]` |
| `ProjectMetaPublicEndpoint` | `space/views/meta.py:17` | `permission_classes = [AllowAny]` |

### 1.3 IntakeIssuePublicViewSet 的权限（核心发现）

**文件**: `apps/api/plane/space/views/intake.py:31-35`

```python
class IntakeIssuePublicViewSet(BaseViewSet):
    serializer_class = IntakeIssueSerializer
    model = IntakeIssue
    filterset_fields = ["status"]
    # ❗ 没有重写 permission_classes！
    # 继承 BaseViewSet 的 [IsAuthenticated]
```

**结论**: `IntakeIssuePublicViewSet` 没有重写 `permission_classes`，因此**所有端点都继承了基类的 `IsAuthenticated` 要求**。这与其他公开 API 的 `AllowAny` 模式不同！

### 1.4 各端点认证要求逐条确认

| HTTP 方法 | URL 路径 | 处理方法 | 登录要求 | 权限控制来源 | 说明 |
|----------|---------|---------|---------|-------------|------|
| GET | `/api/public/anchor/<anchor>/intakes/<intake_id>/intake-issues/` | `list()` | ✅ 必须登录 | `BaseViewSet.permission_classes = [IsAuthenticated]` | 基类默认 |
| GET | `/api/public/anchor/<anchor>/intakes/<intake_id>/inbox-issues/` | `list()` | ✅ 必须登录 | 同上 | 别名路由 |
| POST | `/api/public/anchor/<anchor>/intakes/<intake_id>/intake-issues/` | `create()` | ✅ 必须登录 | 同上 | 基类默认 |
| POST | `/api/public/anchor/<anchor>/intakes/<intake_id>/inbox-issues/` | `create()` | ✅ 必须登录 | 同上 | 别名路由 |
| GET | `/api/public/anchor/<anchor>/intakes/<intake_id>/intake-issues/<pk>/` | `retrieve()` | ✅ 必须登录 | 同上 | 基类默认 |
| PATCH | `/api/public/anchor/<anchor>/intakes/<intake_id>/intake-issues/<pk>/` | `partial_update()` | ✅ 必须登录 | 同上 + 创建者校验 | 方法内额外校验 |
| DELETE | `/api/public/anchor/<anchor>/intakes/<intake_id>/intake-issues/<pk>/` | `destroy()` | ✅ 必须登录 | 同上 + 创建者校验 | 方法内额外校验 |

### 1.5 方法内额外权限校验

#### partial_update (175-234行):
```python
def partial_update(self, request, anchor, intake_id, pk):
    # ...
    # 仅创建者可编辑
    if str(intake_issue.created_by_id) != str(request.user.id):
        return Response({"error": "You cannot edit intake issues"}, 400)
```

#### destroy (258-280行):
```python
def destroy(self, request, anchor, intake_id, pk):
    # ...
    # 仅创建者可删除
    if str(intake_issue.created_by_id) != str(request.user.id):
        return Response({"error": "You cannot delete intake issue"}, 400)
```

### 1.6 公开 API vs 内部 API 认证对比

| 维度 | 公开 API (`/api/public/anchor/...`) | 内部 API (`/api/workspaces/...`) |
|------|-----------------------------------|--------------------------------|
| 基类权限 | `[IsAuthenticated]` (需登录) | `[IsAuthenticated]` (需登录) |
| 装饰器权限 | 无 `@allow_permission` 装饰器 | 有 `@allow_permission` 装饰器做角色校验 |
| list 认证 | ✅ 必须登录 | ✅ 必须登录 + 角色校验 (ADMIN/MEMBER/GUEST) |
| create 认证 | ✅ 必须登录 | ✅ 必须登录 + 角色校验 (ADMIN/MEMBER/GUEST) |
| retrieve 认证 | ✅ 必须登录 | ✅ 必须登录 + 角色校验 + GUEST 可见性限制 |
| patch 认证 | ✅ 必须登录 + 仅创建者 | ✅ 必须登录 + ADMIN 或创建者 + 角色细分 |
| delete 认证 | ✅ 必须登录 + 仅创建者 | ✅ 必须登录 + ADMIN 或创建者 |

> **修正之前的错误结论**: 公开 API 的 list 和 retrieve **不是匿名访问**，必须登录。这是因为 `IntakeIssuePublicViewSet` 没有像其他公开 View 那样显式设置 `permission_classes = [AllowAny]`。

---

## 二、intake-issues 与 inbox-issues 路由覆盖关系复核

### 2.1 公开 API 路由完整清单

**文件**: `apps/api/plane/space/urls/intake.py:14-34`

```python
urlpatterns = [
    # ========== 集合路由（Collection Routes）==========
    
    # 路由 1a: intake-issues 前缀（集合操作）
    path(
        "anchor/<str:anchor>/intakes/<uuid:intake_id>/intake-issues/",
        IntakeIssuePublicViewSet.as_view({"get": "list", "post": "create"}),
        name="intake-issue",
    ),
    # 路由 1b: inbox-issues 前缀（集合操作，别名）
    path(
        "anchor/<str:anchor>/intakes/<uuid:intake_id>/inbox-issues/",
        IntakeIssuePublicViewSet.as_view({"get": "list", "post": "create"}),
        name="inbox-issue",
    ),
    
    # ========== 详情路由（Detail Routes）==========
    
    # 路由 2: intake-issues 前缀（详情操作）
    path(
        "anchor/<str:anchor>/intakes/<uuid:intake_id>/intake-issues/<uuid:pk>/",
        IntakeIssuePublicViewSet.as_view({"get": "retrieve", "patch": "partial_update", "delete": "destroy"}),
        name="intake-issue",
    ),
    
    # ❗ 注意：没有 inbox-issues 前缀的详情路由！
]
```

### 2.2 路由覆盖关系表

| 路由类型 | intake-issues 前缀 | inbox-issues 前缀 | 说明 |
|---------|-------------------|-------------------|------|
| GET list | ✅ 支持 | ✅ 支持 | 双别名，都调用 `list()` |
| POST create | ✅ 支持 | ✅ 支持 | 双别名，都调用 `create()` |
| GET retrieve | ✅ 支持 | ❌ 不支持 | 仅 intake-issues 有详情路由 |
| PATCH partial_update | ✅ 支持 | ❌ 不支持 | 仅 intake-issues 有详情路由 |
| DELETE destroy | ✅ 支持 | ❌ 不支持 | 仅 intake-issues 有详情路由 |

### 2.3 内部 API 路由对比（参考）

**文件**: `apps/api/plane/app/urls/intake.py`

```python
urlpatterns = [
    # 集合路由：双别名
    path(".../intake-issues/", IntakeIssueViewSet.as_view({"get": "list", "post": "create"})),
    path(".../inbox-issues/", IntakeIssueViewSet.as_view({"get": "list", "post": "create"})),
    
    # 详情路由：双别名（与公开 API 不同！）
    path(".../intake-issues/<uuid:pk>/", IntakeIssueViewSet.as_view({"get": "retrieve", "patch": "partial_update", "delete": "destroy"})),
    path(".../inbox-issues/<uuid:pk>/", IntakeIssueViewSet.as_view({"get": "retrieve", "patch": "partial_update", "delete": "destroy"})),
]
```

| 路由类型 | 内部 API intake-issues | 内部 API inbox-issues |
|---------|----------------------|----------------------|
| GET list | ✅ 支持 | ✅ 支持 |
| POST create | ✅ 支持 | ✅ 支持 |
| GET retrieve | ✅ 支持 | ✅ 支持 |
| PATCH partial_update | ✅ 支持 | ✅ 支持 |
| DELETE destroy | ✅ 支持 | ✅ 支持 |

### 2.4 不对称性总结

```
公开 API (/api/public/anchor/...):
  集合路由: intake-issues/ ↔ inbox-issues/  (双别名，对称)
  详情路由: intake-issues/<pk>/  (仅单一路由，不对称)
                     ↳ 没有 inbox-issues/<pk>/ 路由！

内部 API (/api/workspaces/...):
  集合路由: intake-issues/ ↔ inbox-issues/  (双别名，对称)
  详情路由: intake-issues/<pk>/ ↔ inbox-issues/<pk>/  (双别名，对称)
```

> **发现**: 公开 API 的详情路由缺少 `inbox-issues/<pk>/` 别名，这可能是一个遗漏的配置，或者是有意为之的设计。

---

## 三、get_queryset 方法的潜在问题

**文件**: `apps/api/plane/space/views/intake.py:37-54`

```python
def get_queryset(self):
    project_deploy_board = DeployBoard.objects.get(
        workspace__slug=self.kwargs.get("slug"),       # ⚠️ 路由中没有 slug 参数！
        project_id=self.kwargs.get("project_id"),       # ⚠️ 路由中没有 project_id 参数！
    )
    # ...
```

**问题分析**:
- 路由模式: `anchor/<str:anchor>/intakes/<uuid:intake_id>/...`
- 只有 `anchor` 和 `intake_id` 两个 URL 参数
- 但 `get_queryset()` 尝试获取 `slug` 和 `project_id`，这会导致 `DoesNotExist` 异常

**实际影响**:
- `list()`, `create()`, `retrieve()`, `partial_update()`, `destroy()` 方法都**重写了查询逻辑**，直接在方法内部通过 `anchor` 查找 `DeployBoard`
- 因此 `get_queryset()` 实际上**从未被调用**（或者调用会失败但被上层捕获）
- 这是一个遗留的、有 bug 的方法，应该被清理或修复

---

## 四、完整权限矩阵（最终版）

### 4.1 公开 API 权限矩阵

| 操作 | 登录要求 | 额外权限 | 可操作字段 |
|------|---------|---------|-----------|
| GET list | ✅ 必须登录 | 无 | 只读 |
| POST create | ✅ 必须登录 | 无 | name, description_json, description_html, priority |
| GET retrieve | ✅ 必须登录 | 无 | 只读 |
| PATCH partial_update | ✅ 必须登录 | 仅创建者 | 仅 name, description_html, description_json |
| DELETE destroy | ✅ 必须登录 | 仅创建者 | - |

### 4.2 内部 API 权限矩阵

| 操作 | 登录要求 | 额外权限 | 可操作字段 |
|------|---------|---------|-----------|
| GET list | ✅ 必须登录 | ADMIN/MEMBER/GUEST<br>GUEST 只能看自己的 | 只读 |
| POST create | ✅ 必须登录 | ADMIN/MEMBER/GUEST | name, description, priority 等 |
| GET retrieve | ✅ 必须登录 | ADMIN/MEMBER/GUEST<br>GUEST 只能看自己的 | 只读 |
| PATCH partial_update | ✅ 必须登录 | ADMIN 可改所有<br>MEMBER 只能改 Issue 内容<br>GUEST 仅改自己的 name/description | Issue 字段: 按角色<br>IntakeIssue 状态: 仅 ADMIN |
| DELETE destroy | ✅ 必须登录 | ADMIN 或创建者 | - |

### 4.3 关键权限差异对比

| 维度 | 公开 API | 内部 API |
|------|---------|---------|
| list 访问控制 | 无角色过滤，登录即可看所有 | 有角色过滤，GUEST 只能看自己的 |
| create 字段限制 | 仅 name/description/priority | 可设置所有 Issue 字段 |
| patch 角色控制 | 仅创建者可改，无角色细分 | ADMIN/MEMBER/GUEST 权限细分 |
| 状态变更 (status) | ❌ 无法修改（没有对应逻辑） | ✅ ADMIN 可修改状态 |
| 详情路由别名 | 仅 intake-issues | intake-issues 和 inbox-issues 双别名 |

---

## 五、关键代码位置汇总（Round 3 新增/修正）

| 模块 | 文件路径 | 关键代码 | 说明 |
|------|----------|---------|------|
| 公开 API 基类 | `apps/api/plane/space/views/base.py` | `BaseViewSet.permission_classes = [IsAuthenticated]` | 默认要求认证 |
| 公开 Issue 权限 | `apps/api/plane/space/views/issue.py` | `permission_classes = [AllowAny]` | 其他公开 View 显式设置为匿名 |
| 公开 Intake ViewSet | `apps/api/plane/space/views/intake.py` | `IntakeIssuePublicViewSet` 无 permission_classes 重写 | 继承 IsAuthenticated |
| 公开路由配置 | `apps/api/plane/space/urls/intake.py` | 集合路由双别名，详情路由仅 intake-issues | 路由不对称 |
| 内部路由配置 | `apps/api/plane/app/urls/intake.py` | 集合和详情路由都有双别名 | 路由对称 |
| get_queryset bug | `apps/api/plane/space/views/intake.py:37-54` | 引用不存在的 slug/project_id 参数 | 方法未被实际调用 |

---

## 六、修正总结（Round 2 → Round 3）

### 6.1 认证边界修正

| 项 | Round 2 结论 | Round 3 修正结论 |
|----|------------|-----------------|
| list 是否匿名 | ❌ 认为是公开匿名 | ✅ 必须登录（继承 IsAuthenticated） |
| retrieve 是否匿名 | ❌ 认为是公开匿名 | ✅ 必须登录（继承 IsAuthenticated） |
| 与其他公开 API 关系 | ❌ 认为一致 | ✅ 不一致，其他公开 API 显式 AllowAny |

### 6.2 路由覆盖修正

| 项 | Round 2 结论 | Round 3 修正结论 |
|----|------------|-----------------|
| 详情路由别名 | ❌ 未明确 | ✅ 公开 API 仅 intake-issues 有详情路由，inbox-issues 没有 |
| 内部 API 详情路由 | ❌ 未对比 | ✅ 内部 API 双别名都有详情路由 |
