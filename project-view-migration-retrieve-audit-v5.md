# 项目视图权限与回放：迁移链回填事实与 retrieve 崩溃分支分析

---

## 1. 迁移链 0053 → 0069：`access=0` 视图的 `owned_by` 回填事实

### 1.1 迁移 0053：GlobalView → IssueView 的数据迁移

`0053_auto_20240102_1315.py:51-70`：

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
                access=global_view.access,           # ← 从 GlobalView 继承
                filters=global_view.query_data.get("filters", {}),
                sort_order=global_view.sort_order,
                created_by_id=global_view.created_by_id,
                updated_by_id=global_view.updated_by_id,
            )
        )
    IssueView.objects.bulk_create(updated_issue_views, batch_size=100)
```

**此迁移中未设置的字段**：
- `owned_by_id` — 不在参数中
- `project_id` — 不在参数中（GlobalView 是工作区级别的）
- `is_locked` — 不在参数中（模型尚无此字段）
- `display_filters` — 不在参数中
- `display_properties` — 不在参数中
- `rich_filters` — 不在参数中（模型尚无此字段）

**关键问题**：迁移 0053 创建的 IssueView **没有 `owned_by_id`**，且**没有 `project_id`**（由 GlobalView 的无项目性质决定）。

### 1.2 迁移 0069：添加 `owned_by` 和 `is_locked` 字段 + 回填

`0069_alter_account_provider_and_more.py:9-13`：

```python
def populate_views_owned_by(apps, schema_editor):
    IssueView = apps.get_model("db", "IssueView")
    # update all existing views to be owned by the user who created them
    IssueView.objects.update(owned_by_id=F("created_by_id"))
```

此迁移的操作序列：

```python
operations = [
    # 1. 添加 is_locked 字段（BooleanField, default=False）
    migrations.AddField(
        model_name="issueview",
        name="is_locked",
        field=models.BooleanField(default=False),
    ),
    # 2. 添加 owned_by 字段（ForeignKey, null=True ← 允许 NULL）
    migrations.AddField(
        model_name="issueview",
        name="owned_by",
        field=models.ForeignKey(
            on_delete=django.db.models.deletion.CASCADE,
            related_name="views",
            to=settings.AUTH_USER_MODEL,
            null=True,              # ← 关键：初始允许 NULL
        ),
    ),
    # 3. 回填 owned_by_id = created_by_id
    migrations.RunPython(populate_views_owned_by),
    # 4. 将 owned_by 改为 NOT NULL
    migrations.AlterField(
        model_name="issueview",
        name="owned_by",
        field=models.ForeignKey(
            on_delete=django.db.models.deletion.CASCADE,
            related_name="views",
            to=settings.AUTH_USER_MODEL,
            # ← 注意：无 null=True，即 NOT NULL
        ),
    ),
]
```

### 1.3 回填效果分析

迁移 0069 第 3 步执行 `IssueView.objects.update(owned_by_id=F("created_by_id"))`：

| 视图来源 | `created_by_id` | `owned_by_id` 回填结果 |
|---------|----------------|----------------------|
| 0053 迁移（来自 GlobalView） | `global_view.created_by_id`（来自 GlobalView 的创建者） | ✅ **被回填**为 GlobalView 的创建者 ID |
| 0053 迁移（`created_by_id` 为 NULL） | NULL | `owned_by_id = NULL` → **第 4 步 ALTER 会失败** |

**核心结论**：

1. **0053 迁移确实设置了 `created_by_id`**（第 66 行：`created_by_id=global_view.created_by_id`），因此 0069 的 `F("created_by_id")` 回填**可以正常工作**，不会导致 NULL。
2. **0053 迁移的视图 `owned_by` 被回填为 GlobalView 的创建者**，该用户可以正常访问自己的视图（包括 `access=0` 的私有视图）。
3. **0053 迁移的视图没有 `project_id`**，这些是工作区级别的视图，只能通过 `WorkspaceViewViewSet` 访问。
4. **0053 迁移不设置 `is_locked`**，0069 添加字段时使用默认值 `False`。

### 1.4 `access=0` 视图的可见性影响

迁移 0053 产生的 `access=0` 视图经过 0069 回填后：

- `owned_by_id` = `created_by_id` = GlobalView 的创建者 ID
- `project_id` = NULL（工作区级别）
- 只能通过 `WorkspaceViewViewSet.get_queryset()` 访问

`WorkspaceViewViewSet.get_queryset()` 的过滤条件：

```python
.filter(workspace__slug=slug)
.filter(project__isnull=True)                          # ← 匹配 project_id=NULL
.filter(Q(owned_by=request.user) | Q(access=1))        # ← access=0 只有 owned_by 可匹配
```

因此：
- **GlobalView 的原始创建者**可以看到自己创建的 `access=0` 视图（通过 `Q(owned_by=request.user)` 匹配）
- **其他用户**看不到这些视图（`Q(access=1)` 不匹配 `access=0`）
- `owned_by` 回填是有效的，**不存在"孤儿视图"问题**

### 1.5 0053 之后的新视图

0053 之后创建的视图走 API 路径，`perform_create` 自动注入 `owned_by=request.user`，`access` 默认为 1。这些视图不存在 `owned_by` 缺失问题。

### 1.6 完整迁移时间线

| 迁移 | 日期 | 对 IssueView 的影响 |
|------|------|-------------------|
| 0053 | 2024-01-02 | 从 GlobalView 创建 IssueView，含 `access`、`created_by_id`，但无 `owned_by`、`is_locked`、`project_id` |
| 0069 | 2024-06-03 | 添加 `is_locked`(default=False) + `owned_by`(null) → 回填 `owned_by_id=F("created_by_id")` → 改 `owned_by` 为 NOT NULL |
| 0081 | 2024-10-15 | 删除 `GlobalView` 模型（数据已在 0053 迁移到 IssueView） |
| 0107 | 更晚 | 将 `filters` 迁移到 `rich_filters`，不涉及 `access`/`owned_by` |

---

## 2. retrieve 执行顺序与 Guest 崩溃分支逐行分析

### 2.1 IssueViewViewSet.retrieve 完整执行流程

代码（`base.py:308-341`），逐行标注执行顺序：

```python
@allow_permission(allowed_roles=[ROLE.ADMIN, ROLE.MEMBER, ROLE.GUEST])    # ← ① 装饰器先于方法体执行
def retrieve(self, request, slug, project_id, pk):

    issue_view = self.get_queryset()                                       # ← ② 查询 queryset
        .filter(pk=pk, project_id=project_id)
        .first()                                                           # ← ③ 可能返回 None

    project = Project.objects.get(id=project_id)                           # ← ④ 获取项目（可能 DoesNotExist → 404）

    if (                                                                   # ← ⑤ Guest 检查块开始
        ProjectMember.objects.filter(
            workspace__slug=slug,
            project_id=project_id,
            member=request.user,
            role=5,
            is_active=True,
        ).exists()
        and not project.guest_view_all_features
        and not issue_view.owned_by == request.user                        # ← ⑥ 如果 issue_view=None → AttributeError!
    ):
        return Response(
            {"error": "You are not allowed to view this issue"},
            status=status.HTTP_403_FORBIDDEN,
        )                                                                  # ← ⑦ 如果命中则提前返回 403

    serializer = IssueViewSerializer(issue_view)                           # ← ⑧ 序列化（issue_view 可能为 None）
    recent_visited_task.delay(                                             # ← ⑨ 异步任务入队
        slug=slug,
        project_id=project_id,
        entity_name="view",
        entity_identifier=pk,
        user_id=request.user.id,
    )
    return Response(serializer.data, status=status.HTTP_200_OK)            # ← ⑩ 返回响应
```

### 2.2 关键判断：Guest 崩溃发生在 `recent_visited_task` 之前还是之后

**代码行号对照**：

| 行号 | 代码 | 说明 |
|------|------|------|
| 310 | `issue_view = self.get_queryset().filter(...).first()` | 查询视图，可能为 None |
| 311 | `project = Project.objects.get(id=project_id)` | 获取项目 |
| 317-331 | `if (...) and not issue_view.owned_by == request.user:` | **Guest 崩溃点** |
| 333 | `serializer = IssueViewSerializer(issue_view)` | 在崩溃点之后 |
| 334-340 | `recent_visited_task.delay(...)` | **在崩溃点之后** |
| 341 | `return Response(...)` | 在崩溃点之后 |

**结论**：Guest + `gvaf=False` + `issue_view=None` 时，`AttributeError` 发生在第 326 行，`recent_visited_task.delay()` 在第 334 行。**崩溃在 `recent_visited_task` 之前，该异步任务不会被触发。**

此前分析的"`.delay()` 在 `AttributeError` 之前被调用"的结论是**错误的**。正确结论是：`AttributeError` 在第 326 行发生时，控制流直接跳转到 `BaseViewSet.handle_exception`，第 334 行的 `.delay()` **不会被执行**。

### 2.3 六种场景的精确执行路径

#### 场景 A：非 Guest + 视图不可见（私有视图或不存在）

```
① allow_permission → 通过
②③ get_queryset().filter().first() → None
④ Project.objects.get(id=project_id) → 项目对象
⑤⑥ Guest 检查块 → ProjectMember(role=5).exists() = False → 跳过整个 if
⑧ IssueViewSerializer(None) → 创建序列化器
⑨ recent_visited_task.delay() → ✅ 执行（对不存在的 pk 写入访问记录）
⑩ Response(200, {id: null, name: "", ...})
```

**副作用**：`recent_visited_task` 写入了一条指向不存在视图的访问记录。

#### 场景 B：Guest + `gvaf=False` + 视图不可见

```
① allow_permission → 通过
②③ get_queryset().filter().first() → None
④ Project.objects.get(id=project_id) → 项目对象
⑤ ProjectMember(role=5).exists() → True
  not project.guest_view_all_features → True
⑥ issue_view.owned_by → AttributeError! (None.owned_by)
  ↓ 异常抛出，控制流转到 handle_exception
⑦-⑩ 均不执行
⑨ recent_visited_task.delay() → ❌ 未执行

handle_exception:
  AttributeError 不在已知类型中 → 500 "Something went wrong please try again later"
```

**副作用**：`recent_visited_task` **不会执行**，无无效访问记录。

#### 场景 C：Guest + `gvaf=False` + 公开视图（非所有者）

```
① allow_permission → 通过
②③ get_queryset().filter().first() → 视图对象
④ Project.objects.get(id=project_id) → 项目对象
⑤ ProjectMember(role=5).exists() → True
  not project.guest_view_all_features → True
⑥ not issue_view.owned_by == request.user → True（不是所有者）
⑦ return Response(403)
⑧-⑩ 均不执行
⑨ recent_visited_task.delay() → ❌ 未执行
```

**副作用**：`recent_visited_task` **不会执行**，因为 403 在 `.delay()` 之前返回。

#### 场景 D：Guest + `gvaf=True` + 可见视图

```
① allow_permission → 通过
②③ get_queryset().filter().first() → 视图对象
④ Project.objects.get(id=project_id) → 项目对象
⑤⑥ Guest 检查块 → not project.guest_view_all_features = False → 跳过
⑧ IssueViewSerializer(视图对象) → 正常序列化
⑨ recent_visited_task.delay() → ✅ 执行
⑩ Response(200, 正常数据)
```

**副作用**：`recent_visited_task` 正常写入有效访问记录。

### 2.4 WorkspaceViewViewSet.retrieve 的执行流程

代码（`base.py:102-112`）：

```python
def retrieve(self, request, slug, pk):
    issue_view = self.get_queryset().filter(pk=pk).first()    # ← ① 可能 None
    serializer = IssueViewSerializer(issue_view)              # ← ② 序列化
    recent_visited_task.delay(                                # ← ③ 异步入队
        slug=slug,
        project_id=None,
        entity_name="view",
        entity_identifier=pk,
        user_id=request.user.id,
    )
    return Response(serializer.data, status=200)              # ← ④ 返回
```

- **没有 Guest 检查块**
- **没有 `None` 保护**
- 当 `issue_view=None` 时，步骤 ②③④ 均正常执行
- `recent_visited_task` **总是被触发**，无论视图是否存在

### 2.5 `recent_visited_task` 触发总结

| 场景 | IssueViewViewSet | WorkspaceViewViewSet |
|------|-----------------|---------------------|
| 正常视图 | ✅ 触发 | ✅ 触发 |
| 视图不可见（非 Guest） | ✅ 触发（写无效记录） | ✅ 触发（写无效记录） |
| 视图不可见（Guest 崩溃） | ❌ 不触发（崩溃在 `.delay()` 之前） | — |
| 视图可见但 403 拒绝 | ❌ 不触发（403 在 `.delay()` 之前返回） | — |

---

## 3. `access=0` 视图可见性的最终结论

### 3.1 迁移 0053 产生的 `access=0` 视图

- `owned_by_id` 被 0069 迁移回填为 `created_by_id`（即 GlobalView 的创建者）
- `project_id` = NULL（工作区级别）
- 只通过 `WorkspaceViewViewSet` 访问
- **创建者**：可见（`Q(owned_by=request.user)` 匹配）
- **其他用户**：不可见（`Q(access=1)` 不匹配 `access=0`）
- **不存在孤儿视图问题**

### 3.2 Seed 产生的视图

- `views.json` 当前仅 `access: 1`
- 如果改为 `access: 0`，seed 代码可正常创建私有视图
- `owned_by_id` = `bot_user.id`（BotTypeEnum.WORKSPACE_SEED）
- 由于 bot_user 不是真实用户，**任何人通过 `Q(owned_by=request.user)` 都无法匹配**这些私有视图
- **Seed 路径创建的 `access=0` 视图会成为孤儿**——API 层面无人可见

### 3.3 API 产生的视图

- `access` 始终为 1（Public），`owned_by` 为当前用户
- 不存在 `access=0` 视图

---

## 4. 关键代码位置索引

| 文件 | 行号 | 说明 |
|------|------|------|
| `apps/api/plane/db/migrations/0053_auto_20240102_1315.py` | 51-70 | GlobalView → IssueView 迁移，设置 `access`、`created_by_id` |
| `apps/api/plane/db/migrations/0069_alter_account_provider_and_more.py` | 9-13 | `populate_views_owned_by` — 回填 `owned_by_id=F("created_by_id")` |
| `apps/api/plane/db/migrations/0069_alter_account_provider_and_more.py` | 48-72 | 添加 `is_locked` + `owned_by` 字段的四步操作 |
| `apps/api/plane/db/migrations/0081_remove_globalview_created_by_and_more.py` | 166-171 | 删除 GlobalView 模型 |
| `apps/api/plane/app/views/view/base.py` | 308-341 | `IssueViewViewSet.retrieve` — 含 Guest 检查 |
| `apps/api/plane/app/views/view/base.py` | 317-331 | Guest 检查块（崩溃点在第 326 行） |
| `apps/api/plane/app/views/view/base.py` | 334-340 | `recent_visited_task.delay()`（在崩溃点之后） |
| `apps/api/plane/app/views/view/base.py` | 102-112 | `WorkspaceViewViewSet.retrieve` — 无 Guest 检查 |
| `apps/api/plane/bgtasks/workspace_seed_task.py` | 477-501 | `create_views()` — seed 写入路径 |
| `apps/api/plane/seeds/data/views.json` | 全文 | seed 数据，当前 `access: 1` |
| `apps/api/plane/db/models/base.py` | 17-44 | `BaseModel.save()` — `disable_auto_set_user` 参数 |
