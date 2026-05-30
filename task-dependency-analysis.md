# Plane 任务依赖与阻塞机制深度分析

## 目录

1. [依赖关系存储模型](#1-依赖关系存储模型)
2. [关系映射与双向传播](#2-关系映射与双向传播)
3. [依赖修改对任务状态和提醒的影响](#3-依赖修改对任务状态和提醒的影响)
4. [阻塞链路的循环检测](#4-阻塞链路的循环检测)
5. [视图层展示与编辑路径](#5-视图层展示与编辑路径)
6. [关键文件索引](#6-关键文件索引)

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

## 3. 依赖修改对任务状态和提醒的影响

### 3.1 活动记录（Activity）的生成

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

### 3.2 通知的触发

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

### 3.3 对任务状态的直接影响

**关键发现：当前代码中，依赖关系的建立和解除不会自动修改任务的状态（State）。**

- `IssueRelation` 模型本身没有 `save` 钩子或信号监听器来自动更新关联 Issue 的状态
- 任务的 `state` 字段变更仅由 `Issue.save()` 中的 `_sync_completed_at` 处理，该逻辑仅关注 `state_id` 自身的变化
- 换言之：一个被阻塞的任务不会自动变为"Blocked"状态，用户需要手动操作

这意味着依赖关系与任务状态之间是**松耦合**的——依赖关系是纯关系数据，不直接驱动状态流转。

---

## 4. 阻塞链路的循环检测

### 4.1 当前实现状态

**经过对整个代码库的搜索（包括后端 Python 和前端 TypeScript），Plane 当前没有实现阻塞链路的循环检测（cycle detection）机制。**

具体证据：
- 后端无任何 `circular`、`cycle detection`、`topological sort`、`DFS`、`graph traversal` 相关代码
- 前端无循环检测逻辑
- `IssueRelationViewSet.create` 方法在创建关系时未做环路校验
- 数据库层面也无相关约束或存储过程

### 4.2 潜在风险

由于缺少循环检测，用户可能创建如下环路：

```
A blocked_by B → B blocked_by C → C blocked_by A
```

这会导致：
- 阻塞链无限循环，无法被解除
- 甘特图上的依赖路径渲染可能出现异常
- 列表视图中的"被阻塞"标识可能不准确

### 4.3 甘特图依赖层的状态

甘特图的依赖连线组件目前处于**空壳状态**：

```typescript
// apps/web/ce/components/gantt-chart/dependency/dependency-paths.tsx
export function TimelineDependencyPaths(_props: Props) {
  return <></>;
}

// apps/web/ce/components/gantt-chart/dependency/draggable-dependency-path.tsx
export function TimelineDraggablePath() {
  return <></>;
}
```

甘特图 Store 中 `isDependencyEnabled` 默认为 `false`：

```typescript
// apps/web/ce/store/timeline/base-timeline.store.ts:85
isDependencyEnabled = false;
```

`getIsCurrentDependencyDragging` 返回硬编码 `false`：

```typescript
// apps/web/ce/store/timeline/base-timeline.store.ts:345
getIsCurrentDependencyDragging = computedFn((_blockId: string) => false);
```

这表明甘特图上的依赖路径渲染和拖拽创建依赖的功能尚未完全实现。

---

## 5. 视图层展示与编辑路径

### 5.1 任务详情页中的关系 Widget

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

### 5.2 关系选项的配置与样式

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

### 5.3 活动流中的关系展示

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

### 5.4 甘特图中的依赖路径（未完成）

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

### 5.5 乐观更新与回滚机制

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

### 5.6 API 端点汇总

| 操作 | HTTP 方法 | URL |
|---|---|---|
| 查询关系列表 | GET | `/api/workspaces/{slug}/projects/{project_id}/issues/{issue_id}/issue-relation/` |
| 创建关系 | POST | `/api/workspaces/{slug}/projects/{project_id}/issues/{issue_id}/issue-relation/` |
| 删除关系 | POST | `/api/workspaces/{slug}/projects/{project_id}/issues/{issue_id}/remove-relation/` |

**创建请求体**：
```json
{
  "relation_type": "blocked_by",
  "issues": ["uuid-1", "uuid-2"]
}
```

**删除请求体**：
```json
{
  "relation_type": "blocked_by",
  "related_issue": "uuid-1"
}
```

---

## 6. 关键文件索引

### 后端

| 文件 | 职责 |
|---|---|
| `apps/api/plane/db/models/issue.py:258-320` | `IssueBlocker` + `IssueRelation` + `IssueRelationChoices` 模型定义 |
| `apps/api/plane/app/views/issue/relation.py` | `IssueRelationViewSet` — 关系的 CRUD API |
| `apps/api/plane/app/serializers/issue.py:401-479` | `IssueRelationSerializer` + `RelatedIssueSerializer` |
| `apps/api/plane/utils/issue_relation_mapper.py` | `get_inverse_relation` + `get_actual_relation` 映射工具 |
| `apps/api/plane/app/urls/issue.py:234-244` | 关系 API URL 路由 |
| `apps/api/plane/bgtasks/issue_activities_task.py:1277-1377` | 关系活动的创建/删除任务 |
| `apps/api/plane/bgtasks/notification_task.py:191+` | 通知分发任务 |
| `apps/api/plane/db/migrations/0043_*.py` | `IssueBlocker` → `IssueRelation` 数据迁移 |

### 前端

| 文件 | 职责 |
|---|---|
| `packages/types/src/issues/issue_relation.ts` | 关系类型定义 |
| `apps/web/ce/types/gantt-chart.ts` | `TIssueRelationTypes` 类型 |
| `apps/web/core/constants/gantt-chart.ts` | `REVERSE_RELATIONS` 反向映射 |
| `apps/web/core/services/issue/issue_relation.service.ts` | 关系 API 服务层 |
| `apps/web/core/store/issue/issue-details/relation.store.ts` | 关系 MobX Store（含乐观更新） |
| `apps/web/core/components/issues/issue-detail-widgets/relations/` | 关系 Widget 组件目录 |
| `apps/web/ce/components/relations/index.tsx` | `ISSUE_RELATION_OPTIONS` 配置 |
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
3. **状态与提醒**：依赖关系变更不自动修改任务状态，但会触发活动记录和通知；活动记录为双向 Issue 各生成一条
4. **循环检测**：**当前未实现**，存在创建环路依赖的风险
5. **视图层**：任务详情页的 Relations Widget 是主要编辑入口；甘特图依赖连线功能处于预留空壳状态
