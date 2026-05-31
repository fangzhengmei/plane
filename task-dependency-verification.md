# Plane 任务依赖机制核实分析

> 本文档基于源码逐行核实，重点澄清两个关键事实：
> 1. `/issues/.../issue-relation/` 接口实际返回的关系类型
> 2. `remove_relation` 的实际删除条件与双向匹配规则
> 3. 以上事实对循环依赖判断的影响

---

## 目录

1. [旧版 list 接口实际返回的关系类型（8 种，非 2 种）](#1-旧版-list-接口实际返回的关系类型8-种非-2-种)
2. [remove_relation 的实际删除条件与双向匹配规则](#2-remove_relation-的实际删除条件与双向匹配规则)
3. [前端 removeRelation 的双向清理逻辑与 bug](#3-前端-removerelation-的双向清理逻辑与-bug)
4. [以上关键点对循环依赖判断的影响](#4-以上关键点对循环依赖判断的影响)
5. [完整调用链路图](#5-完整调用链路图)
6. [关键文件索引](#6-关键文件索引)

---

## 1. 旧版 list 接口实际返回的关系类型（8 种，非 2 种）

### 1.1 先前分析的误判

先前分析中称旧版 `IssueRelationViewSet.list` 只返回阻塞相关的 2 种类型（blocking / blocked_by），**这是错误的**。

### 1.2 实际代码

```python
# apps/api/plane/app/views/issue/relation.py:42-207
def list(self, request, slug, project_id, issue_id):
    issue_relations = IssueRelation.objects.filter(
        Q(issue_id=issue_id) | Q(related_issue=issue_id)
    ).filter(workspace__slug=self.kwargs.get("slug"))

    # ---- 阻塞关系 ----
    blocking_issues = issue_relations.filter(
        relation_type="blocked_by", related_issue_id=issue_id
    ).values_list("issue_id", flat=True)

    blocked_by_issues = issue_relations.filter(
        relation_type="blocked_by", issue_id=issue_id
    ).values_list("related_issue_id", flat=True)

    # ---- 重复关系 ----
    duplicate_issues = issue_relations.filter(
        issue_id=issue_id, relation_type="duplicate"
    ).values_list("related_issue_id", flat=True)
    duplicate_issues_related = issue_relations.filter(
        related_issue_id=issue_id, relation_type="duplicate"
    ).values_list("issue_id", flat=True)

    # ---- 关联关系 ----
    relates_to_issues = issue_relations.filter(
        issue_id=issue_id, relation_type="relates_to"
    ).values_list("related_issue_id", flat=True)
    relates_to_issues_related = issue_relations.filter(
        related_issue_id=issue_id, relation_type="relates_to"
    ).values_list("issue_id", flat=True)

    # ---- 时间依赖关系 ----
    start_after_issues = issue_relations.filter(
        relation_type="start_before", related_issue_id=issue_id
    ).values_list("issue_id", flat=True)
    start_before_issues = issue_relations.filter(
        relation_type="start_before", issue_id=issue_id
    ).values_list("related_issue_id", flat=True)
    finish_after_issues = issue_relations.filter(
        relation_type="finish_before", related_issue_id=issue_id
    ).values_list("issue_id", flat=True)
    finish_before_issues = issue_relations.filter(
        relation_type="finish_before", issue_id=issue_id
    ).values_list("related_issue_id", flat=True)

    # ---- 构建响应 ----
    response_data = {
        "blocking":      queryset.filter(pk__in=blocking_issues)
                          .annotate(relation_type=Value("blocking", ...)),
        "blocked_by":    queryset.filter(pk__in=blocked_by_issues)
                          .annotate(relation_type=Value("blocked_by", ...)),
        "duplicate":     queryset.filter(pk__in=duplicate_issues)
                          | queryset.filter(pk__in=duplicate_issues_related),
        "relates_to":    queryset.filter(pk__in=relates_to_issues)
                          | queryset.filter(pk__in=relates_to_issues_related),
        "start_after":   queryset.filter(pk__in=start_after_issues)
                          .annotate(relation_type=Value("start_after", ...)),
        "start_before":  queryset.filter(pk__in=start_before_issues)
                          .annotate(relation_type=Value("start_before", ...)),
        "finish_after":  queryset.filter(pk__in=finish_after_issues)
                          .annotate(relation_type=Value("finish_after", ...)),
        "finish_before": queryset.filter(pk__in=finish_before_issues)
                          .annotate(relation_type=Value("finish_before", ...)),
    }
    return Response(response_data, status=status.HTTP_200_OK)
```

### 1.3 实际返回类型总结

**旧版 `/issues/.../issue-relation/` GET 返回全部 8 种关系类型**：

| 返回 key | 数据库查询条件 | 语义 |
|---|---|---|
| `blocking` | `relation_type="blocked_by" AND related_issue_id=当前issue` | 谁被我阻塞（反向） |
| `blocked_by` | `relation_type="blocked_by" AND issue_id=当前issue` | 我被谁阻塞 |
| `duplicate` | `relation_type="duplicate" AND (issue_id=当前 OR related_issue_id=当前)` | 双向对称 |
| `relates_to` | `relation_type="relates_to" AND (issue_id=当前 OR related_issue_id=当前)` | 双向对称 |
| `start_after` | `relation_type="start_before" AND related_issue_id=当前issue` | 我之后开始（反向） |
| `start_before` | `relation_type="start_before" AND issue_id=当前issue` | 我之前开始 |
| `finish_after` | `relation_type="finish_before" AND related_issue_id=当前issue` | 我之后完成（反向） |
| `finish_before` | `relation_type="finish_before" AND issue_id=当前issue` | 我之前完成 |

### 1.4 新版与旧版 list 接口的真正差异

| 维度 | 旧版 `IssueRelationViewSet.list` | 新版 `IssueRelationListCreateAPIEndpoint.get` |
|---|---|---|
| **返回类型数量** | 8 种（相同） | 8 种（相同） |
| **返回数据格式** | 返回完整的 Issue 对象（含 state、priority、label 等） | 返回轻量引用 `{project_id, issue_id}` |
| **URL 前缀** | `/issues/` | `/work-items/` |
| **关系推导逻辑** | 完全一致 | 完全一致 |
| **前端调用** | ✅ 正在使用 | ❌ 未接入 |

**结论：旧版和新版 list 接口返回的关系类型范围完全相同，差异仅在返回数据的粒度（完整对象 vs 轻量引用）。**

---

## 2. remove_relation 的实际删除条件与双向匹配规则

### 2.1 先前分析的误判

先前分析称 `remove_relation` 使用 `relation_type + related_issue_id` 精确定位记录并做反向类型转换，**这是错误的**。

### 2.2 实际代码

```python
# apps/api/plane/app/views/issue/relation.py:262-284
def remove_relation(self, request, slug, project_id, issue_id):
    related_issue = request.data.get("related_issue", None)

    issue_relations = IssueRelation.objects.filter(
        workspace__slug=slug,
    ).filter(
        Q(issue_id=related_issue, related_issue_id=issue_id)
        | Q(issue_id=issue_id, related_issue_id=related_issue)
    )
    issue_relations = issue_relations.first()
    current_instance = json.dumps(IssueRelationSerializer(issue_relations).data, ...)
    issue_relations.delete()
    # ... 触发活动记录 ...
    return Response(status=status.HTTP_204_NO_CONTENT)
```

### 2.3 逐行解析

**前端发送的请求体**：
```typescript
// apps/web/core/services/issue/issue_relation.service.ts:41-52
async deleteIssueRelation(workspaceSlug, projectId, issueId, data) {
    return this.post(
        `/api/workspaces/${workspaceSlug}/projects/${projectId}/issues/${issueId}/remove-relation/`,
        data  // data = { relation_type: relationType, related_issue: related_issue }
    )
}
```

**后端实际使用的字段**：
- ✅ `related_issue` — 从 `request.data` 读取
- ❌ `relation_type` — **完全未使用！** 代码中只有 `request.data.get("related_issue")`，没有 `request.data.get("relation_type")`

**过滤条件**：
```python
Q(issue_id=related_issue, related_issue_id=issue_id)   # 方向 A
| Q(issue_id=issue_id, related_issue_id=related_issue)  # 方向 B
```

这是一个**双向 OR 查询**：无论 `related_issue` 在数据库中是 `issue_id` 还是 `related_issue_id`，都能匹配到。然后用 `.first()` 取第一条。

### 2.4 删除匹配规则总结

| 请求参数 | 后端是否使用 | 作用 |
|---|---|---|
| `related_issue` | ✅ 使用 | 定位配对的 Issue ID |
| `relation_type` | ❌ **未使用** | 前端发送但后端忽略 |
| URL 中的 `issue_id` | ✅ 使用 | 定位当前 Issue ID |

**匹配逻辑**：
1. 在工作空间范围内查找 `IssueRelation` 记录
2. 匹配条件：`(issue=A, related_issue=B) OR (issue=B, related_issue=A)`
3. 取第一条匹配结果（`.first()`）
4. 执行删除

### 2.5 关键推论

**同一个 `(A, B)` 对可能存在多条不同 `relation_type` 的记录**。例如：

```
记录1: issue=A, related_issue=B, relation_type="blocked_by"
记录2: issue=A, related_issue=B, relation_type="relates_to"
```

由于 `remove_relation` **不按 `relation_type` 过滤**，调用 `.first()` 只会删除其中一条，**且删除哪条取决于数据库返回顺序（不可预测）**。

这意味着：如果 A 和 B 同时存在 "阻塞" 和 "关联" 两种关系，删除其中一种时可能会误删另一种。

### 2.6 软删除机制

```python
# apps/api/plane/db/mixins.py:72-82
class SoftDeleteModel(models.Model):
    deleted_at = models.DateTimeField(null=True, blank=True)

    def delete(self, using=None, soft=True, *args, **kwargs):
        if soft:
            self.deleted_at = timezone.now()
            self.save(using=using)
            soft_delete_related_objects.delay(...)
        else:
            return super().delete(using=using, *args, **kwargs)
```

`IssueRelation` 继承自 `ProjectBaseModel → BaseModel → AuditModel → SoftDeleteModel`，因此：
- `.delete()` 默认执行**软删除**（设置 `deleted_at`）
- 软删除后的记录不会被 `SoftDeletionManager` 的默认查询返回
- 但 `.all_objects` 管理器仍可查到

---

## 3. 前端 removeRelation 的双向清理逻辑与 bug

### 3.1 前端 Store 代码

```typescript
// apps/web/core/store/issue/issue-details/relation.store.ts:227-267
removeRelation = async (
    workspaceSlug, projectId, issueId, relationType, related_issue, updateLocally = false
) => {
    try {
        // 1. 从正向关系列表中移除
        const relationIndex = this.relationMap[issueId]?.[relationType]?.findIndex(
            (_issueId) => _issueId === related_issue
        );
        if (relationIndex >= 0)
            runInAction(() => {
                this.relationMap[issueId]?.[relationType]?.splice(relationIndex, 1);
            });

        // 2. 调用后端 API
        if (!updateLocally) {
            await this.issueRelationService.deleteIssueRelation(workspaceSlug, projectId, issueId, {
                relation_type: relationType,
                related_issue,
            });
        }

        // 3. 从反向关系列表中移除
        const reverseRelatedType = REVERSE_RELATIONS[relationType];
        const relatedIndex = this.relationMap[related_issue]?.[reverseRelatedType]?.findIndex(
            (_issueId) => _issueId === related_issue   // ⚠️ BUG: 应该是 _issueId === issueId
        );
        if (relationIndex >= 0)   // ⚠️ BUG: 应该判断 relatedIndex >= 0
            runInAction(() => {
                this.relationMap[related_issue]?.[reverseRelatedType]?.splice(relatedIndex, 1);
            });
    } catch (error) {
        this.fetchRelations(workspaceSlug, projectId, issueId);
        throw error;
    }
};
```

### 3.2 两个 bug

**Bug 1：反向关系查找时使用了错误的 ID**

```typescript
// 第 253-254 行
const relatedIndex = this.relationMap[related_issue]?.[reverseRelatedType]?.findIndex(
    (_issueId) => _issueId === related_issue   // ❌ 错误！在 B 的 blocking 列表中查找 B
);
```

正确逻辑：在 `related_issue`（即 B）的 `reverseRelatedType`（如 "blocking"）列表中，应该查找 `issueId`（即 A），因为 A 是 B 的 blocking 关联方。

修正应为：
```typescript
(_issueId) => _issueId === issueId   // ✅ 在 B 的 blocking 列表中查找 A
```

**Bug 2：条件判断使用了错误的变量**

```typescript
// 第 256 行
if (relationIndex >= 0)   // ❌ 应该是 relatedIndex >= 0
```

`relationIndex` 是正向关系的索引，在步骤 1 中可能已经是 `>= 0`；而 `relatedIndex` 才是反向关系的索引。如果正向关系存在但反向关系不存在（数据不一致时），会执行 `splice(relatedIndex=-1, 1)`，虽然不会报错（会删除最后一个元素），但语义错误。

### 3.3 Bug 的实际影响

| 场景 | Bug 影响 |
|---|---|
| 正常创建的单向阻塞关系（A blocked_by B） | 反向清理不会生效：`B.blocking` 列表中查找 `B` 而非 `A`，`findIndex` 返回 `-1`，但由于 Bug 2 检查的是 `relationIndex`（此时 `>=0`），会执行 `splice(-1, 1)` **错误地删除 B.blocking 列表的最后一个元素** |
| 对称关系（A relates_to B） | `relates_to` 的反向仍是 `relates_to`，查找条件变成 `在 B.relates_to 中查找 B`，同样错误 |

**由于后端删除后会触发重新拉取（catch 中 `fetchRelations`），加上页面切换时也会重新加载，这个 bug 在大多数场景下不会导致持久性数据错误，但会在删除操作后的短暂瞬间显示不正确的关系列表。**

---

## 4. 以上关键点对循环依赖判断的影响

### 4.1 remove_relation 的无差别匹配对循环依赖的意义

`remove_relation` 不区分 `relation_type`、只按 `(issue, related_issue)` 对匹配的机制，对循环依赖的解除有**正反两面影响**：

**正面：解除环路更简单**

假设环路：`A blocked_by B → B blocked_by C → C blocked_by A`

要从 A 的视角解除 A 对 B 的关系（不论关系类型），只需：
```
POST /remove-relation/
{ related_issue: "B" }
```

后端会匹配 `(issue=A, related=B) OR (issue=B, related=A)`，找到 A 和 B 之间的所有关系中第一条并删除。**不需要知道关系是 blocked_by 还是其他类型**。

**反面：可能误删非阻塞关系**

如果 A 和 B 之间同时存在两种关系：
```
记录1: issue=A, related_issue=B, relation_type="blocked_by"
记录2: issue=A, related_issue=B, relation_type="relates_to"
```

调用 `remove_relation` 时，`.first()` 只返回一条记录，删除哪条取决于数据库排序。**可能删除 relats_to 而保留 blocked_by**，导致：
- 环路未被真正解除（blocked_by 仍在）
- 非阻塞的关联关系被意外删除

### 4.2 同一对 Issue 间的多关系场景分析

`IssueRelation` 模型有唯一约束：
```python
UniqueConstraint(
    fields=["issue", "related_issue"],
    condition=Q(deleted_at__isnull=True),
    name="issue_relation_unique_issue_related_issue_when_deleted_at_null",
)
```

该约束只限制 `(issue, related_issue)` 对在**未软删除**时唯一。这意味着：
- `issue=A, related_issue=B, relation_type="blocked_by"` 和 `issue=A, related_issue=B, relation_type="relates_to"` **不能同时存在**
- 但 `issue=A, related_issue=B, relation_type="blocked_by"` 和 `issue=B, related_issue=A, relation_type="relates_to"` **可以同时存在**（因为 (A,B) 和 (B,A) 是不同的键）

因此 `.first()` 的不确定性主要出现在**方向相反**的关系对中：
```
记录1: issue=A, related_issue=B, relation_type="blocked_by"  (A 被 B 阻塞)
记录2: issue=B, related_issue=A, relation_type="relates_to"  (B 与 A 关联)
```

此时从 A 发起 `remove_relation(related_issue=B)`：
- 查询条件匹配到两条记录
- `.first()` 删除哪条不确定
- 如果删除了记录2（relates_to），阻塞关系仍然存在

### 4.3 循环依赖的完整解除策略

考虑到 `remove_relation` 的无差别匹配特性，循环依赖的解除需要额外注意：

**场景：`A blocked_by B → B blocked_by C → C blocked_by A`**

数据库中实际存储：
```
记录1: issue=A, related=B, type="blocked_by"
记录2: issue=B, related=C, type="blocked_by"
记录3: issue=C, related=A, type="blocked_by"
```

每一对 `(issue, related_issue)` 只有一条记录，所以 `.first()` 没有歧义。

**从 A 的详情页解除 A 对 B 的 blocked_by 关系**：
```
前端调用: removeRelation(ws, proj, A.id, "blocked_by", B.id)
后端: filter(Q(issue=B, related=A) | Q(issue=A, related=B))
      → 匹配到记录1: issue=A, related=B, type="blocked_by"
      → 删除 ✅ 正确
```

**从 B 的详情页解除 B 对 A 的 blocking 关系**：
```
前端调用: removeRelation(ws, proj, B.id, "blocking", A.id)
后端: filter(Q(issue=A, related=B) | Q(issue=B, related=A))
      → 匹配到记录1: issue=A, related=B, type="blocked_by"
      → 删除 ✅ 正确（虽然前端传的 relation_type="blocking" 被忽略，但结果正确）
```

**结论**：在纯阻塞环路中（每对关系只有一条 blocked_by 记录），`remove_relation` 的无差别匹配反而**降低了误删风险**，因为不存在同一对之间的多类型冲突。

### 4.4 风险场景：混合类型环路

如果环路中混合了不同类型的关系：
```
A blocked_by B → B relates_to C → C blocked_by A
```

数据库：
```
记录1: issue=A, related=B, type="blocked_by"
记录2: issue=B, related=C, type="relates_to"
记录3: issue=C, related=A, type="blocked_by"
```

从 B 的视角删除与 C 的关系：
```
前端调用: removeRelation(ws, proj, B.id, "relates_to", C.id)
后端: filter(Q(issue=C, related=B) | Q(issue=B, related=C))
      → 只有记录2匹配
      → 删除 ✅ 正确
```

**此场景也安全**，因为 (B,C) 之间只有一条记录。

### 4.5 真正的风险场景

只有当**同一对 Issue 以相反方向存在两种关系**时才有风险：
```
记录1: issue=A, related=B, type="blocked_by"  (A 被 B 阻塞)
记录2: issue=B, related=A, type="relates_to"  (B 与 A 关联)
```

从 A 发起删除：
```
前端调用: removeRelation(ws, proj, A.id, "blocked_by", B.id)
后端: filter(Q(issue=B, related=A) | Q(issue=A, related=B))
      → 匹配到记录1和记录2两条！
      → .first() 删除其中一条（不确定哪条）
```

**这是 `remove_relation` 设计上的根本缺陷**：忽略 `relation_type` 导致无法精确删除指定类型的关系。

---

## 5. 完整调用链路图

### 5.1 查询关系链路

```
前端 IssueRelationStore.fetchRelations()
  → IssueRelationService.listIssueRelations()
    → GET /api/workspaces/{slug}/projects/{pid}/issues/{iid}/issue-relation/
      → IssueRelationViewSet.list()
        → 查询 IssueRelation（Q(issue=iid) | Q(related_issue=iid)）
        → 按 8 种类型分组，返回完整 Issue 对象
  ← 前端 Store: 遍历 response 的 8 个 key，写入 relationMap[issueId][relation_key]
```

### 5.2 创建关系链路

```
前端 IssueRelationStore.createRelation()
  → IssueRelationService.createIssueRelations()
    → POST /api/workspaces/{slug}/projects/{pid}/issues/{iid}/issue-relation/
      body: { relation_type: "blocked_by", issues: ["B", "C"] }
      → IssueRelationViewSet.create()
        → 反向类型交换字段（blocking → blocked_by, issue/related 互换）
        → bulk_create + ignore_conflicts=True
        → issue_activity.delay(type="issue_relation.activity.created", notification=True)
      ← 返回 IssueRelationSerializer / RelatedIssueSerializer 序列化结果
  ← 前端 Store: 更新 relationMap[iid][relationType] + relationMap[B][reverseType]
```

### 5.3 删除关系链路

```
前端 IssueRelationStore.removeRelation(ws, pid, issueId, relationType, relatedIssueId)
  │
  ├─ 1. 乐观更新：从 relationMap[issueId][relationType] 中移除 relatedIssueId
  │
  ├─ 2. 调用后端
  │   → IssueRelationService.deleteIssueRelation()
  │     → POST /api/workspaces/{slug}/projects/{pid}/issues/{iid}/remove-relation/
  │       body: { relation_type: "blocked_by", related_issue: "B" }   ← relation_type 被后端忽略！
  │       → IssueRelationViewSet.remove_relation()
  │         → 只读 related_issue
  │         → filter(Q(issue=B, related=A) | Q(issue=A, related=B))   ← 双向 OR 匹配
  │         → .first() → 取第一条
  │         → .delete() → 软删除（设置 deleted_at）
  │         → issue_activity.delay(type="issue_relation.activity.deleted")
  │
  ├─ 3. 清理反向关系（有 bug）
  │   → 在 relationMap[relatedIssueId][reverseType] 中查找 relatedIssueId  ← 应查找 issueId
  │   → 判断 relationIndex >= 0                                            ← 应判断 relatedIndex
  │   → splice 移除
  │
  └─ 4. 失败回滚：fetchRelations() 重新拉取全量
```

---

## 6. 关键文件索引

| 文件 | 行号 | 核心内容 |
|---|---|---|
| `apps/api/plane/app/views/issue/relation.py` | 42-207 | `IssueRelationViewSet.list` — 返回 8 种关系类型 |
| `apps/api/plane/app/views/issue/relation.py` | 209-260 | `IssueRelationViewSet.create` — 反向类型字段交换 |
| `apps/api/plane/app/views/issue/relation.py` | 262-284 | `IssueRelationViewSet.remove_relation` — **忽略 relation_type，双向 OR 匹配** |
| `apps/api/plane/app/urls/issue.py` | 235-244 | 关系 API URL 路由注册 |
| `apps/api/plane/db/mixins.py` | 61-82 | `SoftDeleteModel.delete()` — 默认软删除 |
| `apps/api/plane/utils/issue_relation_mapper.py` | 全文 | `get_inverse_relation` / `get_actual_relation` |
| `apps/web/core/store/issue/issue-details/relation.store.ts` | 227-267 | `removeRelation` — **反向清理 bug（第 253-256 行）** |
| `apps/web/core/services/issue/issue_relation.service.ts` | 41-52 | `deleteIssueRelation` — 请求体含 `relation_type` + `related_issue` |
| `apps/web/core/constants/gantt-chart.ts` | 9-14 | `REVERSE_RELATIONS` — 4 种反向映射 |

---

## 总结：三项核心修正

### 修正 1：旧版 list 接口返回 8 种关系类型

先前误认为旧版 `/issues/.../issue-relation/` 只返回 blocking/blocked_by 两种类型。实际代码中，`IssueRelationViewSet.list` 查询了全部 6 种数据库存储类型（blocked_by、duplicate、relates_to、start_before、finish_before），并推导出 8 种用户视角类型（含 blocking、start_after、finish_after）。**旧版与新版 list 接口在关系类型覆盖范围上完全一致**，差异仅在返回数据粒度。

### 修正 2：remove_relation 忽略 relation_type，使用双向 OR 匹配

先前误认为 `remove_relation` 通过 `relation_type + related_issue_id` 精确定位记录并做反向类型转换。实际代码中：
- **只读取 `related_issue`，完全忽略 `relation_type`**
- 使用 `Q(issue=A, related=B) | Q(issue=B, related=A)` 的双向 OR 查询
- `.first()` 取第一条匹配记录

这在纯阻塞环路中不会出错（每对 Issue 间只有一条记录），但在同一对 Issue 以相反方向存在多种关系时可能误删。

### 修正 3：前端 removeRelation 存在反向清理 bug

第 253-256 行有两个 bug：
1. 在 `related_issue` 的反向关系列表中查找 `related_issue` 自身（应查找 `issueId`）
2. 条件判断使用 `relationIndex` 而非 `relatedIndex`

此 bug 不会导致持久数据错误（后端删除是正确的，catch 中有全量拉取兜底），但会在删除操作后的短暂瞬间显示不一致的关系列表。
