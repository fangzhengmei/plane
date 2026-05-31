# 项目视图权限与回放机制核查报告

---

## 1. `access=0` 权限的来源：三类写入路径的完整证据

### 1.1 API 路径（经过序列化器）

**写入能力：不可设置 `access=0`**

`IssueViewSerializer.Meta.read_only_fields`（`serializers/view.py:62-69`）将 `access` 和 `is_locked` 标记为只读。DRF `ModelSerializer.to_internal_value()` 在处理请求体时跳过 `read_only_fields` 中的字段，无论 `create()` 还是 `update()` 均不会写入。

`perform_create()` 只注入 `project_id` 和 `owned_by`（`base.py:260-261`）：

```python
def perform_create(self, serializer):
    serializer.save(project_id=self.kwargs.get("project_id"), owned_by=self.request.user)
```

模型默认值 `access=1`（Public），`is_locked=False`。因此 API 路径创建的视图**永远是 `access=1, is_locked=False`**。

### 1.2 迁移路径（ORM 直接写入）

**写入能力：可设置 `access=0`**

迁移 `0053_auto_20240102_1315.py:51-70` 从 `GlobalView` 迁移数据到 `IssueView`：

```python
def issue_view(apps, schema_editor):
    GlobalView = apps.get_model("db", "GlobalView")
    IssueView = apps.get_model("db", "IssueView")
    updated_issue_views = []
    for global_view in GlobalView.objects.all():
        updated_issue_views.append(
            IssueView(
                workspace_id=global_view.workspace_id,
                name=global_view.name,
                description=global_view.description,
                query=global_view.query,
                access=global_view.access,     # ← 直接从 GlobalView 继承 access
                filters=global_view.query_data.get("filters", {}),
                sort_order=global_view.sort_order,
                created_by_id=global_view.created_by_id,
                updated_by_id=global_view.updated_by_id,
            )
        )
    IssueView.objects.bulk_create(updated_issue_views, batch_size=100)
```

此迁移**不设置 `owned_by`**（未出现在 `bulk_create` 参数中），也不设置 `is_locked`。如果原始 `GlobalView.access=0`，则迁移后的 `IssueView.access=0`。

但此迁移是**历史性的**（从旧版 GlobalView 模型升级），新部署不再执行。新部署的工作区走 seed 路径。

### 1.3 Seed 路径（`workspace_seed_task` + `views.json`）

**写入能力：取决于 JSON 数据，当前数据为 `access=1`**

`workspace_seed_task.py:477-501` 中的 `create_views()` 函数：

```python
def create_views(workspace, project_map, bot_user):
    view_seeds = read_seed_file("views.json")
    if not view_seeds:
        return
    for view_seed in view_seeds:
        project_id = view_seed.pop("project_id")
        view_seed.pop("id")
        issue_view = IssueView(
            **view_seed,                          # ← 展开 JSON 中所有字段（含 access）
            project_id=project_map[project_id],
            workspace=workspace,
            created_by_id=bot_user.id,
            owned_by_id=bot_user.id,
        )
        issue_view.save(created_by_id=bot_user.id, disable_auto_set_user=True)
```

`seeds/data/views.json` 内容：

```json
[{
    "id": 1,
    "name": "Project Urgent Tasks",
    "description": "Project Urgent Tasks",
    "access": 1,                    // ← 当前为 1 (Public)
    "filters": {},
    "project_id": 1,
    "display_filters": {...},
    "display_properties": {...},
    "sort_order": 75535,
    "rich_filters": {"priority__in": "urgent"}
}]
```

**关键发现**：
1. `views.json` 当前 `access=1`，但 seed 代码使用 `**view_seed` 展开，JSON 中**可以包含 `access: 0`**，会直接写入数据库。
2. `views.json` 中**不包含 `is_locked` 字段**，IssueView 模型默认 `is_locked=False`。
3. Seed 路径通过 ORM `save()` 写入，**绕过序列化器**，不受 `read_only_fields` 约束。
4. Seed 创建的视图的 `owned_by` 是 `bot_user`（BotTypeEnum.WORKSPACE_SEED），而非请求用户。

### 1.4 对比：Page seed 的 `access=0` 实例

`seeds/data/pages.json` 中第一条记录 `"access": 0`（Private），第二条 `"access": 1`（Public）。

`workspace_seed_task.py:345-388` 中的 `create_pages()` 函数：

```python
page = Page(
    workspace_id=workspace.id,
    is_global=False,
    access=page_seed.get("access", Page.PUBLIC_ACCESS),  # ← 显式读取 access
    ...
)
```

Page seed **支持 `access=0`**，证明 seed 路径的设计意图就是可以创建不同可见性的对象。当前 `views.json` 只有 `access=1` 是**数据层面**的决定，不是代码层面的限制。

### 1.5 三类路径写入能力总结

| 字段 | API (序列化器) | 迁移 0053 (ORM) | Seed (ORM) | Django Shell |
|------|---------------|----------------|-----------|-------------|
| `access=0` | ❌ 被丢弃 | ✅ 从 GlobalView 继承 | ✅ 由 `views.json` 数据决定 | ✅ |
| `access=1` | ✅ 模型默认值 | ✅ | ✅ 当前 `views.json` 值 | ✅ |
| `is_locked=True` | ❌ 被丢弃 | ❌ 未设置 | ❌ JSON 中无此字段 | ✅ |
| `is_locked=False` | ✅ 模型默认值 | ✅ 默认值 | ✅ 默认值 | ✅ |

**结论**：`access=0` 视图**可以**通过迁移 0053（历史）或修改 `views.json` 后的 seed 路径（当前可操作）产生。CE 代码库中**不存在将 `is_locked` 设为 `True` 的受控路径**。

---

## 2. IssueView retrieve 空对象分支的返回行为与异常处理

### 2.1 DRF `Serializer.data` 对 `None` 实例的处理

DRF 源码 `rest_framework/serializers.py` 中 `BaseSerializer.data` 属性：

```python
@property
def data(self):
    if hasattr(self, 'initial_data') and not hasattr(self, '_validated_data'):
        raise AssertionError(...)
    if not hasattr(self, '_data'):
        if self.instance is not None and not getattr(self, '_errors', None):
            self._data = self.to_representation(self.instance)
        elif hasattr(self, '_validated_data') and not getattr(self, '_errors', None):
            self._data = self.to_representation(self.validated_data)
        else:
            self._data = self.get_initial()
    return self._data
```

当 `IssueViewSerializer(None)` 被调用时：
- `self.instance is not None` → **False**（`None is not None` 为 False）
- `hasattr(self, '_validated_data')` → **False**（未调用 `is_valid()`）
- 进入 `else` 分支 → `self._data = self.get_initial()`

`Serializer.get_initial()` 的行为：

```python
def get_initial(self):
    if hasattr(self, 'initial_data'):
        if not isinstance(self.initial_data, Mapping):
            return {}
        return {
            field_name: field.get_value(self.initial_data)
            for field_name, field in self.fields.items()
        }
    return {
        field_name: field.get_initial()
        for field_name, field in self.fields.items()
    }
```

由于没有传入 `data=` 参数，`hasattr(self, 'initial_data')` 为 False，走 else 分支：为每个字段调用 `field.get_initial()`，返回该字段类型的默认初始值。

### 2.2 `IssueViewViewSet.retrieve` 完整行为分析

代码（`base.py:308-341`）：

```python
@allow_permission(allowed_roles=[ROLE.ADMIN, ROLE.MEMBER, ROLE.GUEST])
def retrieve(self, request, slug, project_id, pk):
    issue_view = self.get_queryset().filter(pk=pk, project_id=project_id).first()
    project = Project.objects.get(id=project_id)
    if (
        ProjectMember.objects.filter(..., role=5, ...).exists()
        and not project.guest_view_all_features
        and not issue_view.owned_by == request.user    # ← 此处可能崩溃
    ):
        return Response({"error": "..."}, status=403)
    serializer = IssueViewSerializer(issue_view)
    recent_visited_task.delay(...)
    return Response(serializer.data, status=200)
```

**六种场景的完整推演**：

#### 场景 1：非 Guest 用户 + 视图不可见（私有视图或不存在）

```
get_queryset() → Q 过滤排除 → first() = None
→ 跳过 Guest 检查块（非 Guest）
→ IssueViewSerializer(None).data → get_initial()
→ recent_visited_task.delay(pk=pk, ...)
→ Response(200, {id: null, name: "", access: null, ...})
```

**返回 200 + 字段默认初始值**。前端收到有效状态码但内容为空对象。

`recent_visited_task` 会将此 pk 作为 `entity_identifier` 写入 `UserRecentVisit`，即使视图不存在。这是一个副作用——用户"最近访问"列表中会出现一个指向不存在视图的记录。

#### 场景 2：Guest + `gvaf=False` + 视图不可见

```
get_queryset() → Q 过滤排除 → first() = None
→ Guest 检查块进入：
    role=5 → True
    not guest_view_all_features → True
    not issue_view.owned_by == request.user → AttributeError!
        None.owned_by 在 Python 中不存在
```

**`AttributeError` 被 `BaseViewSet.handle_exception` 捕获**（`base.py:70-109`），不在已知异常类型（`IntegrityError`, `ValidationError`, `ObjectDoesNotExist`, `KeyError`）中，最终返回：

```python
return Response(
    {"error": "Something went wrong please try again later"},
    status=status.HTTP_500_INTERNAL_SERVER_ERROR,
)
```

**返回 500**。但 `recent_visited_task` 使用 `.delay()` 异步执行，已在 `AttributeError` 之前被调用，同样会产生无效的最近访问记录。

#### 场景 3：Guest + `gvaf=False` + 公开视图（非所有者）

```
get_queryset() → access=1 包含 → first() = 视图对象
→ Guest 检查块：
    role=5 → True
    not gvaf → True
    not issue_view.owned_by == request.user → True
→ return Response({"error": "..."}, status=403)
```

**返回 403**。语义正确。`recent_visited_task` 未被调用（403 在 `.delay()` 之前返回）。

#### 场景 4：Guest + `gvaf=True` + 可见视图

```
get_queryset() → 包含 → first() = 视图对象
→ Guest 检查块：not gvaf → False → 跳过
→ IssueViewSerializer(视图对象).data → to_representation()
→ Response(200, 正常数据)
```

**返回 200 + 正常数据**。

### 2.3 `WorkspaceViewViewSet.retrieve` 的行为差异

代码（`base.py:102-112`）：

```python
def retrieve(self, request, slug, pk):
    issue_view = self.get_queryset().filter(pk=pk).first()
    serializer = IssueViewSerializer(issue_view)
    recent_visited_task.delay(slug=slug, project_id=None, entity_name="view",
                              entity_identifier=pk, user_id=request.user.id)
    return Response(serializer.data, status=status.HTTP_200_OK)
```

- **没有 Guest 检查块**
- `issue_view` 为 `None` 时走 `get_initial()` → 200 + 空默认值
- 没有项目级成员关系校验
- Guest 可以通过直接 URL retrieve 任何公开工作区视图，即使 list 中看不到

### 2.4 异常处理链总结

| 异常类型 | `handle_exception` 返回 | 出现场景 |
|----------|------------------------|----------|
| `IntegrityError` | 400 "The payload is not valid" | 不在 retrieve 中出现 |
| `ValidationError` | 400 "Please provide valid detail" | 不在 retrieve 中出现 |
| `ObjectDoesNotExist` | 404 "The required object does not exist." | `Project.objects.get(id=project_id)` 找不到项目时 |
| `KeyError` | 400 "The required key does not exist." | 不在 retrieve 中出现 |
| `AttributeError` | **500** "Something went wrong..." | Guest + `gvaf=False` + `issue_view=None` → `None.owned_by` |
| 其他异常 | 500 "Something went wrong..." | 兜底处理 |

注意：`Project.objects.get(id=project_id)` 如果项目不存在会抛 `DoesNotExist`，被转为 404。但 `issue_view` 为 `None` 时不会抛 `DoesNotExist`（因为用了 `.first()` 而非 `.get()`），所以不会被转为 404。

---

## 3. `guest_view_all_features` 在 list/retrieve 的差异链路与一致性风险

### 3.1 IssueViewViewSet 的两层过滤模型

#### list 方法（`base.py:289-306`）

```
第一层：get_queryset()
  ① project__project_projectmember__member=request.user
  ② Q(owned_by=request.user) | Q(access=1)

第二层：list() 内二次过滤
  ③ if Guest(role=5) AND NOT guest_view_all_features:
      queryset.filter(owned_by=request.user)
```

叠加效果：

| 角色 | `gvaf` | 第一层结果 | 第二层结果 | 最终 |
|------|--------|-----------|-----------|------|
| Admin/Member | — | 自己的 + 公开 | — | 自己的 + 公开 |
| Guest | `True` | 自己的 + 公开 | 不触发 | 自己的 + 公开 |
| Guest | `False` | 自己的 + 公开 | 仅自己的 | **仅自己的** |

#### retrieve 方法（`base.py:308-341`）

```
第一层：get_queryset()
  ① project__project_projectmember__member=request.user
  ② Q(owned_by=request.user) | Q(access=1)

第二层：retrieve() 内对象级判断
  ③ if Guest(role=5) AND NOT guest_view_all_features
      AND NOT issue_view.owned_by == request.user:
      return 403
```

叠加效果：

| 角色 | `gvaf` | 视图类型 | 第一层 | 第二层 | 最终 |
|------|--------|---------|--------|--------|------|
| Guest | `False` | 公开 + 非所有者 | ✅ 可查到 | 403 | **403** |
| Guest | `False` | 公开 + 所有者 | ✅ 可查到 | 跳过 | **200** |
| Guest | `False` | 私有 + 非所有者 | ❌ 查不到 | `None.owned_by` 崩溃 | **500** |
| Guest | `False` | 不存在 | ❌ 查不到 | `None.owned_by` 崩溃 | **500** |

### 3.2 一致性风险矩阵

#### 风险 1：list 与 retrieve 对同一公开视图的判断不一致

- **list**：Guest + `gvaf=False` → 公开视图被**二次过滤移除**（不在列表中出现）
- **retrieve**：Guest + `gvaf=False` + 直接访问该公开视图的 URL → **403 拒绝**

结论：两者语义一致（都不可见），但表现不同——list 是静默过滤，retrieve 是显式拒绝。前端体验不一致。

#### 风险 2：WorkspaceViewViewSet 的 list/retrieve 不对称

- **list**：对 Guest 无条件限制为仅自己的视图（不检查 `gvaf`，使用 `WorkspaceMember.role=5`）
- **retrieve**：**完全没有 Guest 检查**

Guest + `gvaf=False` 场景：
- list 返回空列表（工作区视图不出现）
- retrieve 直接访问公开视图 URL → **200 + 正常数据**

**这是真正的安全漏洞**：list 隐藏了视图，但 retrieve 泄露了数据。

#### 风险 3：`issue_view=None` 时 `None.owned_by` 崩溃

仅在 IssueViewViewSet.retrieve 中发生。当 `get_queryset()` 过滤掉视图后：
- 非 Guest 用户：静默返回 200 + 空默认值
- Guest + `gvaf=False`：500 崩溃

同一个"视图不可见"条件，不同角色得到不同错误，且都不返回语义正确的 404。

#### 风险 4：`recent_visited_task` 的副作用

`recent_visited_task.delay()` 在 `AttributeError` 之前被调用（IssueViewViewSet.retrieve 第 334 行），对不可见/不存在的视图 pk 也会写入 `UserRecentVisit` 记录。当 `issue_view=None` 时：
- 非 Guest 场景：`.delay()` 正常执行，写入一条指向不存在视图的访问记录
- Guest 500 场景：`.delay()` 同样在异常前已入队，产生无效记录

### 3.3 完整一致性风险对比

| 维度 | IssueViewViewSet.list | IssueViewViewSet.retrieve | WorkspaceViewViewSet.list | WorkspaceViewViewSet.retrieve |
|------|----------------------|--------------------------|--------------------------|------------------------------|
| Guest 过滤依据 | `ProjectMember.role` + `gvaf` | `ProjectMember.role` + `gvaf` | `WorkspaceMember.role`（无 `gvaf`） | **无** |
| 不可见视图返回 | 不在列表中 | 200 空值 / 500 崩溃 | 不在列表中 | 200 空值 |
| 公开视图（Guest不可看） | 过滤掉 | 403 | 过滤掉 | **200 泄露** |
| `gvaf` 影响 | ✅ 项目级 | ✅ 项目级 | ❌ 仅 WorkspaceMember | ❌ 无 |
| 空值保护 | — | ❌ | — | ❌ |

---

## 4. `access=0` 视图在当前系统中的实际存在可能性

### 4.1 迁移 0053 产生的 `access=0` 视图

迁移 `0053_auto_20240102_1315.py` 从 `GlobalView` 迁移数据，`access=global_view.access`。如果旧版 GlobalView 中存在 `access=0` 的记录，则迁移后 IssueView 中会有 `access=0` 的视图。

但这些视图**没有设置 `owned_by`**（迁移代码未包含该字段），`owned_by_id` 为 NULL。这意味着：
- `Q(owned_by=request.user)` 永远不匹配
- `Q(access=1)` 也不匹配（`access=0`）
- `get_queryset()` 永远过滤掉这些视图
- 即使原始创建者也无法通过 API 看到或操作这些"孤儿视图"

### 4.2 Seed 产生的视图

当前 `views.json` 只有 `access: 1`。但如果修改 `views.json` 加入 `access: 0`，seed 路径可以正常创建私有视图。这些视图的 `owned_by` 是 `bot_user`（BotTypeEnum.WORKSPACE_SEED），不是真实用户，同样无法通过 `Q(owned_by=request.user)` 被找到。

### 4.3 实际结论

在正常部署中，**通过 API 创建的视图永远是 `access=1`**。`access=0` 的视图只可能存在于从旧版 GlobalView 迁移的数据库中，且由于缺少 `owned_by`，这些视图在 API 层面是"不可见的"。

---

## 5. 关键代码位置索引

| 文件 | 行号 | 说明 |
|------|------|------|
| `apps/api/plane/bgtasks/workspace_seed_task.py` | 477-501 | `create_views()` — seed 写入路径 |
| `apps/api/plane/seeds/data/views.json` | 全文 | seed 数据，当前 `access: 1` |
| `apps/api/plane/seeds/data/pages.json` | 全文 | Page seed 数据，包含 `access: 0` 实例 |
| `apps/api/plane/db/migrations/0053_auto_20240102_1315.py` | 51-70 | 迁移写入 `access=global_view.access` |
| `apps/api/plane/app/serializers/view.py` | 62-69 | `read_only_fields` 含 `access`, `is_locked` |
| `apps/api/plane/app/views/view/base.py` | 102-112 | `WorkspaceViewViewSet.retrieve` — 无 Guest 检查 |
| `apps/api/plane/app/views/view/base.py` | 289-306 | `IssueViewViewSet.list` — Guest 二次过滤 |
| `apps/api/plane/app/views/view/base.py` | 308-341 | `IssueViewViewSet.retrieve` — Guest 检查 + 空值崩溃 |
| `apps/api/plane/app/views/view/base.py` | 71-78 | `WorkspaceViewViewSet.list` — 无条件 Guest 过滤 |
| `apps/api/plane/app/views/base.py` | 70-109 | `handle_exception` — 异常映射 |
| `apps/api/plane/bgtasks/recent_visited_task.py` | 17-60 | 异步写入访问记录（不受 retrieve 空值影响） |
