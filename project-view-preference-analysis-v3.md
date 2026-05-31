# 项目视图偏好回放机制：权限与状态同步修正分析

---

## 1. 修正总览

本文档基于代码逐行验证，纠正此前分析中关于 `retrieve` 返回行为、`access`/`is_locked` 写入边界、`guest_view_all_features` 可见性链路的错误结论。

| 修正点 | 此前结论 | 修正后结论 |
|--------|----------|------------|
| 视图不可见时 retrieve 行为 | "序列化 None 报错 → 500" | DRF Serializer 对 `None` 实例走 `get_initial()` → 返回 200 + 空默认值字典 |
| Guest + `guest_view_all_features=False` + 私有视图 retrieve | 未分析 | `issue_view` 为 `None` → `None.owned_by` → `AttributeError` → 500 |
| `access` 在 API create 中是否可设置 | 部分版本说"可设" | 序列化器 `read_only_fields` 严格阻止，API 路径创建的视图永远是 `access=1` |
| `is_locked` 是否有 CE 写入路径 | 模糊 | CE 代码库**无任何写入路径**，仅存在两处只读校验 |
| `WorkspaceViewViewSet.retrieve` 的 Guest 保护 | 未提及 | 完全没有 Guest 检查，与 list 行为不一致 |
| `guest_view_all_features` 对 list 与 retrieve 的影响差异 | 等同 | list 施加在 queryset 二次过滤上；retrieve 施加在对象级判断上，且两处 ViewSet 逻辑不同 |

---

## 2. `retrieve` 路径的真实返回行为

### 2.1 DRF Serializer 对 `None` 实例的处理

`IssueViewViewSet.retrieve`（`base.py:308-341`）和 `WorkspaceViewViewSet.retrieve`（`base.py:102-112`）均使用 `.first()` 查询：

```python
# IssueViewViewSet
issue_view = self.get_queryset().filter(pk=pk, project_id=project_id).first()

# WorkspaceViewViewSet
issue_view = self.get_queryset().filter(pk=pk).first()
```

当 `get_queryset()` 中的可见性过滤（`Q(owned_by=request.user) | Q(access=1)`）排除了目标视图时，`.first()` 返回 `None`。

随后执行 `IssueViewSerializer(issue_view)`，即 `IssueViewSerializer(None)`。

DRF `Serializer.data` 属性（`rest_framework/serializers.py`）的关键逻辑：

```python
@property
def data(self):
    if not hasattr(self, '_data'):
        if self.instance is not None and not getattr(self, '_errors', None):
            self._data = self.to_representation(self.instance)
        elif hasattr(self, '_validated_data') and not getattr(self, '_errors', None):
            self._data = self.to_representation(self.validated_data)
        else:
            self._data = self.get_initial()    # ← instance 为 None 时走此分支
    return self._data
```

`self.instance is None` → 不满足第一个 `if` → 不满足 `elif`（未调用 `is_valid()`）→ **走 `else` 分支，返回 `get_initial()`**。

`get_initial()` 为每个字段返回默认值（`null`/空字符串/空字典等），不会抛出异常。

### 2.2 六种场景的真实返回

#### 场景 A：非 Guest 用户访问自己不可见的私有视图（access=0）

```
get_queryset() → Q 过滤排除 → first() = None
→ 跳过 Guest 检查块
→ IssueViewSerializer(None).data → get_initial()
→ Response(200, {id: null, name: "", access: null, ...})
```

**结果：200 + 空默认值字典。** 应返回 404，实际返回了一个"幽灵视图"。

#### 场景 B：非 Guest 用户访问不存在的视图（错误 pk）

同场景 A，`first()` 返回 `None`，最终 200 + 空默认值。

#### 场景 C：Guest + `guest_view_all_features=False` + 访问自己不可见的私有视图

```
get_queryset() → Q 过滤排除 → first() = None
→ Guest 检查块：
    ProjectMember(role=5).exists() → True
    not project.guest_view_all_features → True
    not issue_view.owned_by == request.user → AttributeError!
        （None.owned_by 不存在）
```

**结果：500 Internal Server Error。** `BaseViewSet.handle_exception` 将 `AttributeError` 转为 `{"error": "Something went wrong please try again later"}`。

代码位置：`base.py:317-331`，`issue_view` 可能为 `None` 但代码未做空值保护。

#### 场景 D：Guest + `guest_view_all_features=False` + 访问他人创建的公开视图（access=1）

```
get_queryset() → access=1 包含该视图 → first() = 视图对象
→ Guest 检查块：
    ProjectMember(role=5).exists() → True
    not project.guest_view_all_features → True
    not issue_view.owned_by == request.user → True（不是所有者）
→ return Response({"error": "..."}, status=403)
```

**结果：403 Forbidden。** 这是唯一正确返回语义状态码的场景。

#### 场景 E：Guest + `guest_view_all_features=True` + 访问任何可见视图

```
get_queryset() → 包含该视图 → first() = 视图对象
→ Guest 检查块：not project.guest_view_all_features → False
→ 跳过检查
→ 200 + 正常视图数据
```

**结果：200 OK + 正常视图。**

#### 场景 F：WorkspaceViewViewSet.retrieve 访问不可见视图

```
get_queryset() → Q 过滤排除 → first() = None
→ 无 Guest 检查块
→ IssueViewSerializer(None).data → get_initial()
→ Response(200, {id: null, name: "", ...})
```

**结果：200 + 空默认值字典。** WorkspaceViewViewSet.retrieve 完全没有 Guest 保护。

### 2.3 返回行为汇总

| ViewSet | 用户角色 | 视图状态 | HTTP 响应 |
|---------|----------|----------|-----------|
| IssueView | 非 Guest | 私有视图（不可见） | 200 + 空默认值 |
| IssueView | 非 Guest | 不存在 | 200 + 空默认值 |
| IssueView | Guest + `gvaf=False` | 私有视图（不可见） | **500** (`None.owned_by` 崩溃) |
| IssueView | Guest + `gvaf=False` | 公开视图（非所有者） | 403 |
| IssueView | Guest + `gvaf=True` | 可见视图 | 200 + 正常数据 |
| WorkspaceView | 任意 | 不可见/不存在 | 200 + 空默认值 |

> `gvaf` = `guest_view_all_features`

---

## 3. `access` / `is_locked` 写入路径边界

### 3.1 API 写入路径（经过序列化器）

`IssueViewSerializer`（`serializers/view.py:56-69`）：

```python
read_only_fields = [
    "workspace",
    "project",
    "query",
    "owned_by",
    "access",        # ← 只读
    "is_locked",     # ← 只读
]
```

DRF `read_only_fields` 的行为：在 `to_internal_value()` 阶段移除这些字段，无论 `create()` 还是 `update()` 均不会写入。

**结论：API 路径（POST/PATCH）无法设置 `access` 和 `is_locked`，创建的视图永远是默认值 `access=1, is_locked=False`。**

前端虽然发送了 `access` 字段（`form.tsx:100`），但值被序列化器静默丢弃。

### 3.2 迁移/ORM 写入路径（绕过序列化器）

#### 3.2.1 迁移 `0053_auto_20240102_1315.py`

```python
# base.py:51-70
def issue_view(apps, schema_editor):
    for global_view in GlobalView.objects.all():
        updated_issue_views.append(
            IssueView(
                workspace_id=global_view.workspace_id,
                name=global_view.name,
                ...
                access=global_view.access,     # ← 可传入 0 (Private)
                ...
            )
        )
    IssueView.objects.bulk_create(updated_issue_views, batch_size=100)
```

此迁移将旧的 `GlobalView` 记录转换为 `IssueView`，**直接通过 ORM 写入 `access` 字段**，可以设置 `access=0 (Private)`。

#### 3.2.2 迁移 `0107_migrate_filters_to_rich_filters.py`

仅更新 `rich_filters` 字段，不涉及 `access` 或 `is_locked`。

#### 3.2.3 `dummy_data_task.py`

**不创建 IssueView 对象**。该脚本仅创建 Project、State、Label、Cycle、Module、Page、Issue 等实体，无视图数据。

#### 3.2.4 Django shell / 管理命令

代码库中**不存在** IssueView 的 Django admin 注册或管理命令。通过 Django shell 手动执行 `IssueView.objects.create(access=0, is_locked=True, ...)` 在技术上是可行的，但这不是受控的写入路径。

### 3.3 对比 Page 模型的 `is_locked` 处理

Page 模型有专门的 `lock()` 和 `unlock()` 动作（`page/base.py:246-266`），直接在 ViewSet 中操作 ORM：

```python
def lock(self, request, slug, project_id, page_id):
    page = Page.objects.get(pk=page_id)
    page.is_locked = True
    page.save()

def unlock(self, request, slug, project_id, page_id):
    page = Page.objects.get(pk=page_id)
    page.is_locked = False
    page.save()
```

IssueView **没有对应的 lock/unlock 端点**（URL 配置中无 `lock`/`unlock` action），也没有任何代码将 `is_locked` 设为 `True`。

### 3.4 写入路径边界总结

| 字段 | API (序列化器) | 迁移 (ORM) | Page 对比 |
|------|---------------|-----------|-----------|
| `access=0` | ❌ `read_only_fields` 阻止 | ✅ 迁移 0053 可写入 | Page 的 `access` 同理可由 ORM 写入 |
| `access=1` | ✅ 模型默认值 | ✅ | ✅ |
| `is_locked=True` | ❌ `read_only_fields` 阻止 | ❌ 无迁移设置 | Page 有 `lock()` 动作 |
| `is_locked=False` | ✅ 模型默认值 | ✅ | Page 有 `unlock()` 动作 |

**最终结论**：CE 代码库中，唯一能创建 `access=0` 视图的路径是迁移 0053（从 GlobalView 迁移）。CE 中**不存在任何方式**将 `is_locked` 设为 `True`。

---

## 4. `guest_view_all_features` 可见性差异链路

### 4.1 IssueViewViewSet 的两层过滤

#### `list` 方法（`base.py:289-306`）

```
第一层：get_queryset()
  └─ Q(owned_by=request.user) | Q(access=1)
  └─ project__project_projectmember__member=request.user

第二层：list() 内部二次过滤
  └─ if Guest(role=5) AND NOT guest_view_all_features:
      └─ queryset.filter(owned_by=request.user)
```

两层过滤的叠加效果：

| 角色 | `guest_view_all_features` | list 结果 |
|------|---------------------------|-----------|
| Admin/Member | — | 自己的视图 + 所有公开视图 |
| Guest | `True` | 自己的视图 + 所有公开视图 |
| Guest | `False` | **仅自己的视图**（公开视图也被过滤掉） |

#### `retrieve` 方法（`base.py:308-341`）

```
第一层：get_queryset()
  └─ Q(owned_by=request.user) | Q(access=1)

第二层：retrieve() 内部对象级检查
  └─ if Guest(role=5) AND NOT guest_view_all_features
      AND NOT issue_view.owned_by == request.user:
      └─ return 403
```

两层过滤的叠加效果：

| 角色 | `guest_view_all_features` | 视图类型 | retrieve 结果 |
|------|---------------------------|----------|---------------|
| Admin/Member | — | 任意可见 | 200 + 视图数据 |
| Guest | `True` | 公开 | 200 + 视图数据 |
| Guest | `False` | 公开 + 非所有者 | 403 |
| Guest | `False` | 公开 + 所有者 | 200 + 视图数据 |
| Guest | `False` | 私有（非所有者） | 500（`None.owned_by` 崩溃） |
| Guest | `False` | 私有（所有者） | 200 + 视图数据 |

### 4.2 WorkspaceViewViewSet 的不对称设计

#### `list` 方法（`base.py:71-78`）

```python
if WorkspaceMember.objects.filter(
    workspace__slug=slug, member=request.user, role=5, is_active=True
).exists():
    queryset = queryset.filter(owned_by=request.user)
```

- 对 Guest **无条件**限制为仅自己的视图（不检查 `guest_view_all_features`）
- 但这里检查的是 `WorkspaceMember.role`，不是 `ProjectMember.role`

#### `retrieve` 方法（`base.py:102-112`）

```python
def retrieve(self, request, slug, pk):
    issue_view = self.get_queryset().filter(pk=pk).first()
    serializer = IssueViewSerializer(issue_view)
    recent_visited_task.delay(...)
    return Response(serializer.data, status=status.HTTP_200_OK)
```

- **完全没有 Guest 检查**
- 只依赖 `get_queryset()` 中的 `Q(owned_by=request.user) | Q(access=1)` 过滤
- Guest 可以通过 retrieve 访问**任何公开的工作区视图**，即使 list 中看不到

### 4.3 两个 ViewSet 的 list/retrieve 差异对比

| 维度 | IssueViewViewSet | WorkspaceViewViewSet |
|------|------------------|---------------------|
| list Guest 过滤 | 检查 `ProjectMember.role=5` + `guest_view_all_features` | 检查 `WorkspaceMember.role=5`，无 `guest_view_all_features` |
| retrieve Guest 过滤 | 检查 `ProjectMember.role=5` + `guest_view_all_features` + `owned_by` | **无任何 Guest 检查** |
| retrieve 空值保护 | ❌ `issue_view` 可能为 `None` | ❌ `issue_view` 可能为 `None` |
| 不存在视图的 retrieve 返回 | 200 + 空默认值 | 200 + 空默认值 |

### 4.4 前端回放如何受影响

前端视图回放流程（`[viewId]/page.tsx` + `filter.store.ts`）：

```
1. useSWR(`VIEW_DETAILS_${viewId}`, fetchViewDetails)
   └─ 调用 GET /api/workspaces/{slug}/projects/{project_id}/views/{viewId}/
   └─ 若返回 200 + 空默认值 → viewMap[viewId] 存入空对象
   └─ 若返回 403 → 前端展示权限错误
   └─ 若返回 500 → 前端展示通用错误

2. fetchFilters() → mutateFilters(workspaceSlug, viewId, viewDetails)
   └─ 从 viewDetails 提取 rich_filters / display_filters / display_properties
   └─ 若 viewDetails 为空默认值 → 所有过滤器为 null/undefined
   └─ computedDisplayFilters() 与默认值合并 → 回退到系统默认布局
```

对前端的具体影响：

1. **非 Guest 用户访问不可见视图**：SWR 收到 200 + 空数据，不触发错误状态，但后续 `fetchFilters` 会写入空过滤器，issue 列表查询返回无结果或默认结果。用户看到"空视图"而非 404 提示。

2. **Guest + `gvaf=False` + 访问不可见私有视图**：SWR 收到 500，前端进入错误状态，展示通用错误信息。

3. **Guest + `gvaf=False` + 访问非所有者公开视图**：SWR 收到 403，前端可正确展示权限不足提示。

---

## 5. 关键代码位置索引

| 文件 | 行号 | 说明 |
|------|------|------|
| `apps/api/plane/app/views/view/base.py` | 102-112 | `WorkspaceViewViewSet.retrieve` — 无 Guest 检查 |
| `apps/api/plane/app/views/view/base.py` | 308-341 | `IssueViewViewSet.retrieve` — 含 Guest 检查但无空值保护 |
| `apps/api/plane/app/views/view/base.py` | 289-306 | `IssueViewViewSet.list` — 二次 Guest 过滤 |
| `apps/api/plane/app/views/view/base.py` | 71-78 | `WorkspaceViewViewSet.list` — 无条件 Guest 过滤 |
| `apps/api/plane/app/serializers/view.py` | 62-69 | `read_only_fields` 阻止 `access` / `is_locked` 写入 |
| `apps/api/plane/app/views/view/base.py` | 343-363 | `partial_update` — `is_locked` 检查 + `owned_by` 检查 |
| `apps/api/plane/app/views/base.py` | 70-109 | `handle_exception` — 将 `AttributeError` 转为 500 |
| `apps/api/plane/db/migrations/0053_auto_20240102_1315.py` | 51-70 | 迁移写入 `access=global_view.access`，可创建私有视图 |
| `apps/api/plane/db/models/view.py` | 58-95 | `IssueView` 模型，`access` 默认值 1，`is_locked` 默认值 False |
| `apps/web/core/store/issue/project-views/filter.store.ts` | 146-174 | `mutateFilters` — 从视图详情回放过滤器 |
| `apps/web/core/store/project-view.store.ts` | 224-239 | `updateView` — 乐观更新 |
