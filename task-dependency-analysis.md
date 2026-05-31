# Plane 任务依赖与阻塞机制深度分析

## 目录

1. [依赖关系存储模型](#1-依赖关系存储模型)
2. [关系映射与双向传播](#2-关系映射与双向传播)
3. [两套关系接口的差异：/work-items vs /issues](#3-两套关系接口的差异work-items-vs-issues)
4. [依赖修改对任务状态和提醒的影响](#4-依赖修改对任务状态和提醒的影响)
5. [阻塞链路的循环检测与解除](#5-阻塞链路的循环检测与解除)
6. [start_before/finish_before 的后端支持与前端展示](#6-start_beforefinish_before-的后端支持与前端展示)
7. [视图层展示与编辑路径](#7-视图层展示与编辑路径)
8. [关键文件索引](#8-关键文件索引)

---

## 1. 依赖关系存储模型

### 1.1 数据库模型

Plane 使用两种数据库模型来表达任务间关系，分别是**旧版 `IssueBlocker`** 和**新版 `IssueRelation`**。

#### IssueBlocker（旧版，已废弃但表仍存在）

```python
# apps/api/plane/db/models/issue.py:258-269
class IssueBlocker(ProjectBaseModel):
    block = models.ForeignKey(Issue, related_name="blocker_issues", on_delete=models.CASCADE)
    blocked_by = models.ForeignKey(Issue, related_name="blocked_issues", on_delete=models.CASCADE)
```

- 仅支持 `block → blocked_by` 一种方向性关系
- 字段语义：`block` 是阻塞方，`blocked_by` 是被阻塞方
- 该模型在迁移 `0043` 中被 `IssueRelation` 替代，旧数据已迁移

#### IssueRelation（当前使用）

```python
# apps/api/plane/db/models/issue.py:296-320
class IssueRelation(ProjectBaseModel):
    issue = models.ForeignKey(Issue, related_name="issue_relation", on_delete=models.CASCADE)
    related_issue = models.ForeignKey(Issue, related_name="issue_related", on_delete=models.CASCADE)
    relation_type = models.CharField(max_length=20, default=IssueRelationChoices.BLOCKED_BY)
```

- 采用**通用化设计**：通过 `relation_type` 字段区分关系类型
- 支持的关系类型由 `IssueRelationChoices` 枚举定义：

```python
# apps/api/plane/db/models/issue.py:272-293
class IssueRelationChoices(models.TextChoices):
    DUPLICATE = "duplicate", "Duplicate"
    RELATES_TO = "relates_to", "Relates To"
    BLOCKED_BY = "blocked_by", "Blocked By"
    START_BEFORE = "start_before", "Start Before"
    FINISH_BEFORE = "finish_before", "Finish Before"
    IMPLEMENTED_BY = "implemented_by", "Implemented By"
```

- **唯一约束**：同一对 `(issue, related_issue)` 在未软删除时只能存在一条关系记录

```python
constraints = [
    models.UniqueConstraint(
        fields=["issue", "related_issue"],
        condition=Q(deleted_at__isnull=True),
        name="issue_relation_unique_issue_related_issue_when_deleted_at_null",
    )
]
```

### 1.2 关系存储的语义约定

数据库中**只存储正向关系类型**（即枚举中定义的值），反向关系通过映射推导。例如：

| 存储的 `relation_type` | 正向语义（issue → related_issue） | 反向语义（related_issue → issue） |
|---|---|---|
| `blocked_by` | issue 被 related_issue 阻塞 | related_issue 阻塞了 issue（即 "blocking"） |
| `start_before` | issue 必须在 related_issue 之前开始 | related_issue 在 issue 之后开始（即 "start_after"） |
| `finish_before` | issue 必须在 related_issue 之前完成 | related_issue 在 issue 之后完成（即 "finish_after"） |
| `relates_to` | issue 与 related_issue 相关 | related_issue 与 issue 相关（对称） |
| `duplicate` | issue 与 related_issue 重复 | related_issue 与 issue 重复（对称） |
| `implemented_by` | issue 由 related_issue 实现 | related_issue 实现了 issue（即 "implements"） |

### 1.3 前端类型定义

```typescript
// packages/types/src/issues/issue_relation.ts
export type TIssueRelationTypes = "blocking" | "blocked_by" | "duplicate" | "relates_to";
export type TIssueRelationIdMap = Record<TIssueRelationTypes, string[]>;
export type TIssueRelationMap = {
  [issue_id: string]: Record<TIssueRelationTypes, string[]>;
};
export type TIssueRelation = Record<TIssueRelationTypes, TIssue[]>;
```

前端仅暴露四种关系类型给用户：`blocking`、`blocked_by`、`duplicate`、`relates_to`。其中 `blocking` 是 `blocked_by` 的反向视角，不作为独立类型存入数据库。

---

## 2. 关系映射与双向传播

### 2.1 后端映射工具

```python
# apps/api/plane/utils/issue_relation_mapper.py
def get_inverse_relation(relation_type):
    """根据存储的关系类型获取反向关系"""
    relation_mapping = {
        "start_after": "start_before",
        "finish_after": "finish_before",
        "blocked_by": "blocking",
        "blocking": "blocked_by",
        "start_before": "start_after",
        "finish_before": "finish_after",
        "implemented_by": "implements",
        "implements": "implemented_by",
    }
    return relation_mapping.get(relation_type, relation_type)

def get_actual_relation(relation_type):
    """将前端传入的关系类型转换为数据库实际存储的类型"""
    actual_relation = {
        "start_after": "start_before",
        "finish_after": "finish_before",
        "blocking": "blocked_by",       # 前端传"blocking"→数据库存"blocked_by"，但 issue/related_issue 字段互换
        "blocked_by": "blocked_by",
        "start_before": "start_before",
        "finish_before": "finish_before",
        "implemented_by": "implemented_by",
        "implements": "implemented_by",
    }
    return actual_relation.get(relation_type, relation_type)
```

### 2.2 前端映射常量

```typescript
// apps/web/core/constants/gantt-chart.ts
export const REVERSE_RELATIONS: { [key in TIssueRelationTypes]: TIssueRelationTypes } = {
  blocked_by: "blocking",
  blocking: "blocked_by",
  relates_to: "relates_to",
  duplicate: "duplicate",
};
```

### 2.3 创建关系时的字段交换逻辑

在 `IssueRelationViewSet.create` 中，当关系类型是反向视角（如 `blocking`、`start_after`、`finish_after`）时，`issue_id` 和 `related_issue_id` 会互换：

```python
# apps/api/plane/app/views/issue/relation.py:220-237
IssueRelation(
    issue_id=(issue if relation_type in ["blocking", "start_after", "finish_after"] else issue_id),
    related_issue_id=(
        issue_id if relation_type in ["blocking", "start_after", "finish_after"] else issue
    ),
    relation_type=(get_actual_relation(relation_type)),
    ...
)
```

**示例**：用户在 Issue A 上添加"blocking"关系到 Issue B：
- 前端发送：`relation_type="blocking"`, `issues=["B"]`
- 后端存储：`issue_id=B`, `related_issue_id=A`, `relation_type="blocked_by"`
- 语义：B 被 A 阻塞（A 阻塞了 B）

### 2.4 前端 Store 中的双向更新

`IssueRelationStore.createRelation` 在创建关系后，会同时更新本端和远端的 `relationMap`：

```typescript
// apps/web/core/store/issue/issue-details/relation.store.ts:147-165
const reverseRelatedType = REVERSE_RELATIONS[relationType];
response.forEach((issue) => {
    const issuesOfRelated = get(this.relationMap, [issue.id, reverseRelatedType]);
    // 更新反向关系
    set(this.relationMap, [issue.id, reverseRelatedType], uniq([...issuesOfRelated, issueId]));
});
// 更新正向关系
set(this.relationMap, [issueId, relationType], uniq(issuesOfRelation));
```

---

## 3. 两套关系接口的差异：/work-items vs /issues

Plane 实现了两套关系 API，分别服务于不同的调用场景。

### 3.1 接口对比总览

| 维度 | 旧版：`/issues/.../issue-relation/` | 新版：`/work-items/.../relations/` |
|---|---|---|
| **前缀** | `/api/workspaces/{slug}/projects/{project_id}/issues/` | `/api/workspaces/{slug}/projects/{project_id}/work-items/` |
| **ViewSet/View** | `IssueRelationViewSet` | `IssueRelationListCreateAPIEndpoint` |
| **支持的关系类型** | 仅阻塞相关（blocking/blocked_by） | 全类型（含 start_before/finish_before 等时间依赖） |
| **查询方法** | `GET /issue-relation/` → 返回分组 JSON | `GET /relations/` → 返回分组 JSON |
| **创建方法** | `POST /issue-relation/` → 单条创建 | `POST /relations/` → 批量创建 |
| **删除方法** | `POST /remove-relation/` → 专用端点 | ❌ **无 DELETE 端点** |
| **文件位置** | `plane/app/views/issue/relation.py` | `plane/api/views/issue.py:2264-2542` |
| **当前调用方** | 前端旧版组件 | 无前端调用（后端预留） |

### 3.2 旧版接口：`IssueRelationViewSet`

```python
# apps/api/plane/app/views/issue/relation.py
class IssueRelationViewSet(GenericViewSet):
    @action(detail=True, methods=["get"], url_path="issue-relation")
    def list(self, request, slug, project_id, issue_id):
        """查询关系列表（仅阻塞相关）"""
        return Response({
            "blocking": ...,
            "blocked_by": ...,
        })

    @action(detail=True, methods=["post"], url_path="issue-relation")
    def create(self, request, slug, project_id, issue_id):
        """创建关系（单条）"""
        # 1. 验证 relation_type 和 issues
        # 2. 执行字段交换（blocking 等反向类型）
        # 3. 写入 IssueRelation
        # 4. 触发活动记录 issue_activity.delay

    @action(detail=True, methods=["post"], url_path="remove-relation")
    def remove(self, request, slug, project_id, issue_id):
        """删除关系"""
        # 1. 通过 relation_type + related_issue_id 定位记录
        # 2. 软删除（设置 deleted_at）
        # 3. 触发活动记录
```

**调用入口（前端）**：
- `apps/web/core/services/issue/issue_relation.service.ts` 的 `listIssueRelations`、`createIssueRelations`、`deleteIssueRelation` 全部调用旧版 `/issues/` 接口
- 前端 Store（`relation.store.ts`）通过该 Service 与后端交互

### 3.3 新版接口：`IssueRelationListCreateAPIEndpoint`

```python
# apps/api/plane/api/views/issue.py:2264-2542
class IssueRelationListCreateAPIEndpoint(BaseAPIView):
    serializer_class = IssueRelationSerializer
    http_method_names = ["get", "post"]  # ⚠️ 仅支持 GET 和 POST！

    def get(self, request, slug, project_id, issue_id):
        """查询关系列表（支持全类型）"""
        return Response({
            "blocking": [...],
            "blocked_by": [...],
            "duplicate": [...],
            "relates_to": [...],
            "start_after": [...],
            "start_before": [...],
            "finish_after": [...],
            "finish_before": [...],  # 8 种类型完整返回
        })

    def post(self, request, slug, project_id, issue_id):
        """创建关系（支持批量，使用 bulk_create）"""
        # 1. 使用 IssueRelationCreateSerializer 验证
        # 2. 支持全部 8 种 RELATION_TYPE_CHOICES
        # 3. bulk_create + ignore_conflicts=True
        # 4. 触发活动记录 issue_activity.delay
```

**关键特性**：
- GET 方法返回 8 种关系类型的完整分组（而旧版只返回 2 种阻塞相关）
- POST 使用 `bulk_create` 批量创建（旧版是单条循环创建）
- **但没有实现 DELETE 方法**——删除仍然只能通过旧版 `/remove-relation/` 端点

**当前调用状态**：
- 前端 Service 中 `IssueRelationService` 仍调用旧版 `/issues/.../issue-relation/`
- 新版 `/work-items/.../relations/` 未被前端调用，处于"后端已实现、前端未接入"状态

### 3.4 创建和删除的调用路径对比

**创建路径**：
```
旧版：
  IssueRelationStore.createRelation()
    → IssueRelationService.createIssueRelations()
      → POST /api/workspaces/{slug}/projects/{project_id}/issues/{issue_id}/issue-relation/
        → IssueRelationViewSet.create()

新版（未使用）：
  POST /api/workspaces/{slug}/projects/{project_id}/work-items/{issue_id}/relations/
    → IssueRelationListCreateAPIEndpoint.post()
```

**删除路径**（仅一条）：
```
IssueRelationStore.removeRelation()
  → IssueRelationService.deleteIssueRelation()
    → POST /api/workspaces/{slug}/projects/{project_id}/issues/{issue_id}/remove-relation/
      → IssueRelationViewSet.remove() （软删除）
```

---

## 4. 依赖修改对任务状态和提醒的影响

### 4.1 活动记录（Activity）的生成

依赖关系的创建和删除均通过 Celery 异步任务生成活动记录：

```python
# apps/api/plane/bgtasks/issue_activities_task.py:1277-1377

def create_issue_relation_activity(...):
    """创建关系时，为正反两端 Issue 各生成一条 Activity"""
    for related_issue in requested_data.get("issues"):
        # 正向 Activity
        issue_activities.append(IssueActivity(
            issue_id=issue_id,
            verb="updated",
            old_value="",
            new_value=f"{issue.project.identifier}-{issue.sequence_id}",
            field=requested_data.get("relation_type"),  # 如 "blocked_by"
            comment=f"added {requested_data.get('relation_type')} relation",
        ))
        # 反向 Activity
        inverse_relation = get_inverse_relation(requested_data.get("relation_type"))
        issue_activities.append(IssueActivity(
            issue_id=related_issue,
            field=inverse_relation,  # 如 "blocking"
            ...
        ))

def delete_issue_relation_activity(...):
    """删除关系时同样生成双向 Activity"""
    # 正向：old_value 填充被删除关系的标识，new_value 为空
    # 反向：field 做阻塞关系的互换（blocked_by ↔ blocking）
```

### 4.2 通知的触发

活动创建完成后，若 `notification=True`，会触发 `notifications` Celery 任务：

```python
# apps/api/plane/bgtasks/issue_activities_task.py:1586-1599
if notification:
    notifications.delay(
        type=type,
        issue_id=issue_id,
        actor_id=actor_id,
        project_id=project_id,
        subscriber=subscriber,
        issue_activities_created=...,
    )
```

通知任务的处理逻辑（`apps/api/plane/bgtasks/notification_task.py:191`）：
- `issue_relation.activity.created` 和 `issue_relation.activity.deleted` **不在排除列表中**，因此会触发通知
- 通知会发送给 Issue 的订阅者（subscribers）和被 @提及 的项目成员
- 支持站内通知和邮件通知两种渠道

### 4.3 对任务状态的直接影响

**关键发现：当前代码中，依赖关系的建立和解除不会自动修改任务的状态（State）。

- `IssueRelation` 模型本身没有 `save` 钩子或信号监听器来自动更新关联 Issue 的状态
- 任务的 `state` 字段变更仅由 `Issue.save()` 中的 `_sync_completed_at` 处理，该逻辑仅关注 `state_id` 自身的变化
- 换言之：一个被阻塞的任务不会自动变为"Blocked"状态，用户需要手动操作

这意味着依赖关系与任务状态之间是**松耦合**的——依赖关系是纯关系数据，不直接驱动状态流转。

---

## 5. 阻塞链路的循环检测与解除

### 5.1 循环检测现状：未实现

**经过对整个代码库的搜索，Plane 当前没有实现阻塞链路的循环检测（cycle detection）机制。**

具体证据：
- 后端无任何 `circular`、`cycle detection`、`topological sort`、`DFS`、`graph traversal` 相关代码
- 前端无循环检测逻辑
- `IssueRelationViewSet.create` 和 `IssueRelationListCreateAPIEndpoint.post` 在创建关系时均未做环路校验
- 数据库层面也无相关约束或存储过程

这意味着用户完全可以创建如下环路：

```
A blocked_by B → B blocked_by C → C blocked_by A
```

### 5.2 循环依赖能否解除？

**答案：可以解除，但必须手动逐环拆解。**

由于 Plane 采用**软删除**模式（设置 `deleted_at` 而非物理删除），且删除操作仅通过 `relation_type + related_issue_id` 定位单条记录，因此：

1. **环路不会阻止删除操作**：删除单条关系不需要遍历整个图，只需要匹配两个字段
2. **但环路状态下的业务逻辑会受影响**：
   - 任务状态流转需手动控制，系统不会自动识别环路
   - 甘特图依赖连线（如实现）可能出现视觉循环
   - 未来的自动调度算法可能陷入死循环

### 5.3 循环依赖的实际解除路径

假设存在环路：**A blocked_by B → B blocked_by C → C blocked_by A**

**解除路径 1：通过 A 的详情页解除 A→B 的关系**

```
1. 打开 Issue A 的详情页
2. 展开 Relations Widget
3. 在 "Blocked by" 分组中找到 Issue B
4. 点击菜单 → "Remove relation"
   → 调用 removeRelation(workspaceSlug, projectId, A.id, "blocked_by", B.id)
   → 后端定位记录：issue_id=A, related_issue_id=B, relation_type="blocked_by"
   → 设置 deleted_at = NOW()（软删除）
5. 环路断裂：A 不再被 B 阻塞
```

**解除路径 2：通过 B 的详情页解除 B 对 A 的阻塞**

```
1. 打开 Issue B 的详情页
2. 展开 Relations Widget
3. 在 "Blocking" 分组中找到 Issue A
4. 点击菜单 → "Remove relation"
   → 调用 removeRelation(workspaceSlug, projectId, B.id, "blocking", A.id)
   → 后端会反向查找：实际删除的仍是 issue_id=A, related_issue_id=B 的记录
   → 设置 deleted_at = NOW()
5. 环路断裂：B 不再阻塞 A
```

**解除路径 3：直接删除其中任意一条关系记录**

无论从环路的哪一端入手，只要删除**任意一条边**，整个环路就会断裂。Plane 的删除逻辑是**局部的、点对点的**，不需要遍历整个依赖图。

### 5.4 删除操作的底层逻辑

```python
# apps/api/plane/app/views/issue/relation.py:255-285
def remove(self, request, slug, project_id, issue_id):
    relation_type = request.data.get("relation_type")
    related_issue = request.data.get("related_issue")
    
    # 关键：反向类型时自动互换查询条件
    if relation_type in ["blocking", "start_after", "finish_after"]:
        actual_relation_type = get_actual_relation(relation_type)
        # 反向查找：issue 和 related_issue 互换
        issue_relation = IssueRelation.objects.filter(
            issue_id=related_issue,
            related_issue_id=issue_id,
            relation_type=actual_relation_type,
        ).first()
    else:
        issue_relation = IssueRelation.objects.filter(
            issue_id=issue_id,
            related_issue_id=related_issue,
            relation_type=relation_type,
        ).first()
    
    if issue_relation:
        issue_relation.delete()  # 软删除（通过模型的 delete 方法）
```

**结论**：循环依赖的解除是**线性可解**的，因为每条关系的删除都只依赖于自身的 `(issue_id, related_issue_id, relation_type)` 三元组，与图的其他部分无关。

---

## 6. start_before/finish_before 的后端支持与前端展示

### 6.1 后端支持的完整关系类型

后端 `IssueRelationChoices` 枚举定义了 6 种关系类型：

```python
# apps/api/plane/db/models/issue.py:272-293
class IssueRelationChoices(models.TextChoices):
    DUPLICATE = "duplicate"
    RELATES_TO = "relates_to"
    BLOCKED_BY = "blocked_by"
    START_BEFORE = "start_before"
    FINISH_BEFORE = "finish_before"
    IMPLEMENTED_BY = "implemented_by"
```

新版 API 的 `IssueRelationCreateSerializer` 则支持全部 **8 种用户视角的关系类型**（含反向类型）：

```python
# apps/api/plane/api/serializers/issue.py:540-549
RELATION_TYPE_CHOICES = [
    ("blocking", "Blocking"),
    ("blocked_by", "Blocked By"),
    ("duplicate", "Duplicate"),
    ("relates_to", "Relates To"),
    ("start_before", "Start Before"),
    ("start_after", "Start After"),
    ("finish_before", "Finish Before"),
    ("finish_after", "Finish After"),
]
```

### 6.2 前端展示的范围对比

| 关系类型 | 数据库支持 | 新版 API GET 返回 | 前端类型定义 | UI 可操作 |
|---|---|---|---|---|
| `blocked_by` | ✅ | ✅ | ✅ `TIssueRelationTypes` | ✅ 可添加/删除 |
| `blocking` | ← 反向 | ✅ | ✅ `TIssueRelationTypes` | ✅ 可添加/删除 |
| `duplicate` | ✅ | ✅ | ✅ `TIssueRelationTypes` | ✅ 可添加/删除 |
| `relates_to` | ✅ | ✅ | ✅ `TIssueRelationTypes` | ✅ 可添加/删除 |
| `start_before` | ✅ | ✅ | ❌ | ❌ |
| `start_after` | ← 反向 | ✅ | ❌ | ❌ |
| `finish_before` | ✅ | ✅ | ❌ | ❌ |
| `finish_after` | ← 反向 | ✅ | ❌ | ❌ |
| `implemented_by` | ✅ | ❌ | ❌ | ❌ |

### 6.3 为什么前端不展示时间依赖类型？

**前端类型定义仅包含 4 种关系**：

```typescript
// packages/types/src/issues/issue_relation.ts
export type TIssueRelationTypes = "blocking" | "blocked_by" | "duplicate" | "relates_to";
```

原因分析：
1. **`ISSUE_RELATION_OPTIONS` 配置仅定义了 4 项**：`apps/web/ce/components/relations/index.tsx` 中的配置对象只有 blocking、blocked_by、duplicate、relates_to 四个 key
2. **甘特图功能未完成**：`start_before/finish_before` 这类时间依赖本应在甘特图中通过拖拽连线创建，但甘特图依赖层目前是空壳（`TimelineDependencyPaths` 返回空 JSX）
3. **历史演进**：旧版 `IssueRelationViewSet` 只返回阻塞相关的两种类型，前端代码基于旧版接口编写，尚未适配新版 API 的全类型

### 6.4 时间依赖类型的实际用途

`start_before` 和 `finish_before` 是**前置依赖约束**：

| 存储类型 | 语义（issue → related_issue） | 典型场景 |
|---|---|---|
| `start_before` | issue 的开始时间必须在 related_issue 的开始时间之前 | 任务 A 开始后，任务 B 才能开始 |
| `finish_before` | issue 的完成时间必须在 related_issue 的完成时间之前 | 任务 A 完成后，任务 B 才能完成 |

这类关系在项目管理工具中通常用于：
- 甘特图上绘制依赖箭头
- 自动调整任务日期（前置任务推迟时，后置任务自动顺延）
- 检测时间冲突

但在 Plane 当前版本中，**这些功能均未实现**，时间依赖类型仅存在于数据库模型和 API 层面，未接入前端。

### 6.5 如何启用时间依赖的前端展示

如要在现有 Relations Widget 中展示时间依赖，需要修改以下几处：

1. **扩展类型定义**：在 `TIssueRelationTypes` 中添加 `start_before`、`start_after`、`finish_before`、`finish_after`
2. **扩展 `ISSUE_RELATION_OPTIONS`**：添加这四种类型的图标、颜色、文案配置
3. **切换到新版 API**：将 `IssueRelationService` 从 `/issues/.../issue-relation/` 切换到 `/work-items/.../relations/`
4. **扩展 Store**：`relationMap` 需要支持新的 relation_type key
5. **实现 DELETE 端点**：新版 API 目前无删除方法，需要补充或继续使用旧版 `/remove-relation/`

---

## 7. 视图层展示与编辑路径

### 7.1 任务详情页中的关系 Widget

关系展示和编辑的核心入口是任务详情页的 **Relations Collapsible** 组件：

```
RelationsCollapsible (root.tsx)
├── RelationsCollapsibleTitle (title.tsx) — 折叠标题栏，显示关系计数 + 添加按钮
│   └── RelationActionButton (quick-action-button.tsx) — 下拉菜单，选择关系类型
└── RelationsCollapsibleContent (content.tsx) — 关系列表内容
    └── RelationIssueList — 按关系类型分组的任务列表
```

#### 添加关系的交互流程

1. 用户点击 **RelationActionButton**（`+` 图标）
2. 弹出下拉菜单，展示四种关系类型：`relates_to`、`duplicate`、`blocked_by`、`blocking`
3. 选择类型后，调用 `toggleRelationModal(issueId, relationKey)` 打开 **ExistingIssuesListModal**
4. 在模态框中搜索并选择要关联的任务
5. 提交后调用 `createRelation()` → 后端 API 创建关系
6. Store 双向更新 `relationMap`，并刷新活动记录

```
RelationActionButton
  → handleOnClick(relationKey)
    → setRelationKey(relationKey)
    → toggleRelationModal(issueId, relationKey)
      → ExistingIssuesListModal (isOpen=true)
        → handleExistingIssueModalOnSubmit(data)
          → createRelation(workspaceSlug, projectId, issueId, relationKey, issueIds)
```

#### 删除关系的交互流程

1. 在 `RelationIssueList` 中，每个关联任务旁有删除图标
2. 点击后调用 `removeRelation()` → 乐观更新 + API 调用
3. 同时更新正向和反向的 `relationMap`

### 7.2 关系选项的配置与样式

```typescript
// apps/web/ce/components/relations/index.tsx
export const ISSUE_RELATION_OPTIONS: Record<TIssueRelationTypes, TRelationObject> = {
  relates_to: {
    key: "relates_to",
    i18n_label: "issue.relation.relates_to",
    className: "bg-layer-1 text-secondary",
    icon: (size) => <RelatedIcon height={size} width={size} />,
    placeholder: "Add related work items",
  },
  blocked_by: {
    key: "blocked_by",
    i18n_label: "issue.relation.blocked_by",
    className: "bg-danger-subtle text-danger-primary",  // 红色，强调阻塞
    icon: (size) => <CircleDot size={size} />,
    placeholder: "None",
  },
  blocking: {
    key: "blocking",
    i18n_label: "issue.relation.blocking",
    className: "bg-yellow-500/20 text-yellow-700",  // 黄色，警示
    icon: (size) => <XCircle size={size} />,
    placeholder: "None",
  },
  duplicate: { ... },
};
```

### 7.3 活动流中的关系展示

关系变更会在活动流（Activity）中展示，由 `activity.tsx` 中的消息模板定义：

```typescript
// apps/web/core/components/core/activity.tsx:601-637
blocking: {
  message: (activity, showIssue) => {
    if (activity.old_value === "")
      return `marked this work item is blocking work item ${activity.new_value}.`;
    else
      return `removed the blocking work item ${activity.old_value}.`;
  },
},
blocked_by: {
  message: (activity, showIssue) => {
    if (activity.old_value === "")
      return `marked this work item is being blocked by ${activity.new_value}.`;
    else
      return `removed this work item being blocked by work item ${activity.old_value}.`;
  },
},
```

### 7.4 甘特图中的依赖路径（未完成）

甘特图已预留了依赖连线的组件架构，但当前为空壳：

```
ce/components/gantt-chart/dependency/
├── index.ts                    — 统一导出
├── dependency-paths.tsx         — 空壳：TimelineDependencyPaths
├── draggable-dependency-path.tsx — 空壳：TimelineDraggablePath
└── blockDraggables/
    ├── index.ts
    ├── left-draggable.tsx       — 左侧拖拽手柄（预留）
    └── right-draggable.tsx      — 右侧拖拽手柄（预留）
```

`BaseTimeLineStore` 中预留了依赖相关的接口：
- `isDependencyEnabled: boolean` — 是否启用依赖连线（当前默认 `false`）
- `getIsCurrentDependencyDragging(blockId)` — 当前块是否正在被依赖拖拽（硬编码 `false`）
- `getUpdatedPositionAfterDrag` 中预留了 `ignoreDependencies` 参数

### 7.5 乐观更新与回滚机制

前端 Store 在修改关系时采用**乐观更新**策略：

**`createCurrentRelation`** — 创建前的乐观写入：

```typescript
// apps/web/core/store/issue/issue-details/relation.store.ts:179-225
try {
    // 1. 先更新本地 Store
    runInAction(() => {
        set(this.relationMap, [issueId, relationType], [...]);
        set(this.relationMap, [relatedIssueId, reverseRelatedType], [...]);
    });
    // 2. 再调用 API
    await this.issueRelationService.createIssueRelations(...);
} catch (e) {
    // 3. API 失败则回滚
    runInAction(() => {
        set(this.relationMap, [issueId, relationType], issuesOfRelation);
        set(this.relationMap, [relatedIssueId, reverseRelatedType], issuesOfRelated);
    });
    throw e;
}
```

**`removeRelation`** — 删除前的乐观移除，失败则重新拉取：

```typescript
// apps/web/core/store/issue/issue-details/relation.store.ts:227-267
try {
    runInAction(() => {
        this.relationMap[issueId]?.[relationType]?.splice(relationIndex, 1);
    });
    await this.issueRelationService.deleteIssueRelation(...);
} catch (error) {
    this.fetchRelations(workspaceSlug, projectId, issueId);  // 回滚：重新拉取全量
    throw error;
}
```

### 7.6 API 端点汇总

| 操作 | HTTP 方法 | URL |
|---|---|---|
| 查询关系列表（旧版） | GET | `/api/workspaces/{slug}/projects/{project_id}/issues/{issue_id}/issue-relation/` |
| 创建关系（旧版） | POST | `/api/workspaces/{slug}/projects/{project_id}/issues/{issue_id}/issue-relation/` |
| 删除关系（唯一） | POST | `/api/workspaces/{slug}/projects/{project_id}/issues/{issue_id}/remove-relation/` |
| 查询关系列表（新版） | GET | `/api/workspaces/{slug}/projects/{project_id}/work-items/{issue_id}/relations/` |
| 创建关系（新版） | POST | `/api/workspaces/{slug}/projects/{project_id}/work-items/{issue_id}/relations/` |

**旧版创建请求体**：
```json
{
  "relation_type": "blocked_by",
  "issues": ["uuid-1", "uuid-2"]
}
```

**新版创建请求体**（相同格式）：
```json
{
  "relation_type": "blocked_by",
  "issues": ["uuid-1", "uuid-2"]
}
```

**删除请求体**（唯一路径）：
```json
{
  "relation_type": "blocked_by",
  "related_issue": "uuid-1"
}
```

---

## 8. 关键文件索引

### 后端

| 文件 | 职责 |
|---|---|
| `apps/api/plane/db/models/issue.py:258-320` | `IssueBlocker` + `IssueRelation` + `IssueRelationChoices` 模型定义 |
| `apps/api/plane/app/views/issue/relation.py` | `IssueRelationViewSet` — 旧版关系 CRUD API（当前使用） |
| `apps/api/plane/api/views/issue.py:2264-2542` | `IssueRelationListCreateAPIEndpoint` — 新版 work-items 关系 API（未使用） |
| `apps/api/plane/app/serializers/issue.py:401-479` | 旧版 `IssueRelationSerializer` |
| `apps/api/plane/api/serializers/issue.py:483-667` | 新版 `IssueRelationCreateSerializer` + `IssueRelationResponseSerializer` |
| `apps/api/plane/utils/issue_relation_mapper.py` | `get_inverse_relation` + `get_actual_relation` 映射工具 |
| `apps/api/plane/app/urls/issue.py:234-244` | 旧版关系 API URL 路由 |
| `apps/api/plane/api/urls/work_item.py:149-154` | 新版关系 API URL 路由 |
| `apps/api/plane/bgtasks/issue_activities_task.py:1277-1377` | 关系活动的创建/删除任务 |
| `apps/api/plane/bgtasks/notification_task.py:191+` | 通知分发任务 |
| `apps/api/plane/db/migrations/0043_*.py` | `IssueBlocker` → `IssueRelation` 数据迁移 |

### 前端

| 文件 | 职责 |
|---|---|
| `packages/types/src/issues/issue_relation.ts` | 关系类型定义（仅 4 种） |
| `apps/web/ce/types/gantt-chart.ts` | `TIssueRelationTypes` 类型 |
| `apps/web/core/constants/gantt-chart.ts` | `REVERSE_RELATIONS` 反向映射 |
| `apps/web/core/services/issue/issue_relation.service.ts` | 关系 API 服务层（调用旧版接口） |
| `apps/web/core/store/issue/issue-details/relation.store.ts` | 关系 MobX Store（含乐观更新） |
| `apps/web/core/components/issues/issue-detail-widgets/relations/` | 关系 Widget 组件目录 |
| `apps/web/core/components/issues/relations/` | 关系通用组件（issue-list-item, properties） |
| `apps/web/ce/components/relations/index.tsx` | `ISSUE_RELATION_OPTIONS` 配置（4 种类型） |
| `apps/web/ce/components/relations/activity.ts` | 活动消息模板 |
| `apps/web/core/components/core/activity.tsx:601-637` | blocking/blocked_by 活动展示 |
| `apps/web/core/components/issues/issue-detail/relation-select.tsx` | 关系选择器组件 |
| `apps/web/ce/components/gantt-chart/dependency/` | 甘特图依赖路径（空壳） |
| `apps/web/ce/store/timeline/base-timeline.store.ts` | 甘特图 Store（依赖相关接口预留） |

---

## 总结

Plane 的任务依赖与阻塞机制采用**单表通用关系模型**（`IssueRelation`），通过 `relation_type` 字段和映射函数实现多种关系类型的统一存储和双向推导。其核心特点：

1. **存储模型**：`IssueRelation` 单表，`issue_id` + `related_issue_id` + `relation_type` 三元组，软删除 + 唯一约束
2. **双向传播**：数据库只存正向类型，反向通过 `get_inverse_relation` / `REVERSE_RELATIONS` 推导；前端 Store 同步维护双向映射
3. **两套 API**：旧版 `/issues/.../issue-relation/`（当前使用）与新版 `/work-items/.../relations/`（后端已实现前端未接入）并存；新版支持全 8 种关系类型，但无 DELETE 端点
4. **状态与提醒**：依赖关系变更不自动修改任务状态，但会触发活动记录和通知；活动记录为双向 Issue 各生成一条
5. **循环检测**：**当前未实现**，存在创建环路依赖的风险；但环路可通过**手动删除任意一条边**解除，因为删除逻辑是点对点的，无需遍历全图
6. **时间依赖**：`start_before/finish_before` 等时间依赖类型在后端完整支持（数据库模型 + 新版 API），但前端类型定义、Store、UI 均未接入，这类关系本应在甘特图中通过拖拽连线创建
7. **视图层**：任务详情页的 Relations Widget 是主要编辑入口；甘特图依赖连线功能处于预留空壳状态
