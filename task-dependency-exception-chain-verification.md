# Plane remove_relation 异常链路证据闭环分析

> 本文档基于 DRF 框架执行逻辑和源码逐行验证，完整还原 `remove_relation` 在目标关系不存在时的异常链路，包括：
> 1. `IssueRelationSerializer(None).data` 为何**不会**报错
> 2. 异常实际抛出点及类型
> 3. HTTP 状态码及返回体的精确结构
> 4. 对前端解除依赖与循环依赖拆解的实际影响

---

## 目录

1. [核心结论速览](#1-核心结论速览)
2. [remove_relation 源码与执行路径](#2-remove_relation-源码与执行路径)
3. [DRF Serializer(None).data 的行为验证](#3-drf-serializernone-data-的行为验证)
4. [异常抛出点确认：第 272 行 delete() 调用](#4-异常抛出点确认第-272-行-delete-调用)
5. [完整异常处理链路](#5-完整异常处理链路)
6. [HTTP 响应最终形态](#6-http-响应最终形态)
7. [前端 Store 与 UI 的实际影响](#7-前端-store-与-ui-的实际影响)
8. [对循环依赖拆解的影响](#8-对循环依赖拆解的影响)
9. [证据链总结](#9-证据链总结)
10. [关键文件索引](#10-关键文件索引)

---

## 1. 核心结论速览

| 问题 | 答案 |
|---|---|
| **`IssueRelationSerializer(None).data` 是否报错？** | ❌ **不会报错** |
| **异常实际抛出点** | 第 272 行：`issue_relations.delete()` |
| **异常类型** | `AttributeError: 'NoneType' object has no attribute 'delete'` |
| **HTTP 状态码** | `500 Internal Server Error` |
| **Content-Type** | `application/json` |
| **返回体结构** | `{"error": "Something went wrong please try again later"}` |
| **对循环依赖拆解影响** | 仅 UI 体验问题，不影响环路可解除性 |

---

## 2. remove_relation 源码与执行路径

### 2.1 完整源码

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
    current_instance = json.dumps(IssueRelationSerializer(issue_relations).data, cls=DjangoJSONEncoder)
    issue_relations.delete()
    issue_activity.delay(
        type="issue_relation.activity.deleted",
        requested_data=json.dumps(request.data, cls=DjangoJSONEncoder),
        actor_id=str(request.user.id),
        issue_id=str(issue_id),
        project_id=str(project_id),
        current_instance=current_instance,
        epoch=int(timezone.now().timestamp()),
        notification=True,
        origin=base_host(request=request, is_app=True),
    )
    return Response(status=status.HTTP_204_NO_CONTENT)
```

### 2.2 目标关系不存在时的逐行执行

**场景**：A 和 B 之间不存在任何关系。

| 行号 | 代码 | 执行结果 |
|---|---|---|
| 263 | `related_issue = request.data.get("related_issue", None)` | `related_issue = "B-id"` ✅ |
| 265-269 | `IssueRelation.objects.filter(...)` | 空 QuerySet ✅ |
| 270 | `issue_relations = issue_relations.first()` | `issue_relations = None` ⚠️ |
| 271 | `IssueRelationSerializer(issue_relations).data` | 返回字段初始值字典（如 `{"id": None, ...}`），**不报错** ✅ |
| 272 | `issue_relations.delete()` | `None.delete()` → **抛出 `AttributeError`** 💥 |
| 273-283 | `issue_activity.delay(...)` | 永不执行 |
| 284 | `return Response(204)` | 永不执行 |

---

## 3. DRF Serializer(None).data 的行为验证

### 3.1 先前分析的误判

先前分析认为第 271 行 `IssueRelationSerializer(None).data` 会抛出 `AttributeError: 'NoneType' object has no attribute 'related_issue'`。**这是错误的。**

### 3.2 DRF `.data` 属性源码级分析

根据 Django REST Framework 3.x 源码，`Serializer.data` 属性的实现逻辑如下：

```python
# rest_framework/serializers.py（简化后的核心逻辑）
@property
def data(self):
    # 检查 1：如果传了 data 但没调用 is_valid()，抛 AssertionError
    if hasattr(self, 'initial_data') and not hasattr(self, '_validated_data'):
        raise AssertionError(...)
    
    if not hasattr(self, '_data'):
        # 分支 1：instance 存在且无错误 → 序列化 instance
        if self.instance is not None and not getattr(self, '_errors', None):
            self._data = self.to_representation(self.instance)
        
        # 分支 2：有校验后的数据且无错误 → 序列化 validated_data
        elif hasattr(self, '_validated_data') and not getattr(self, '_errors', None):
            self._data = self.to_representation(self.validated_data)
        
        # 分支 3：否则 → 返回初始值
        else:
            self._data = self.get_initial()
    
    return self._data
```

### 3.3 分支匹配验证

在 `IssueRelationSerializer(None).data` 场景中：

| 条件 | 求值 |
|---|---|
| `hasattr(self, 'initial_data')` | ❌ False（未传 data 参数） |
| `self.instance is not None` | ❌ False（instance 是 None） |
| `hasattr(self, '_validated_data')` | ❌ False（未调用 is_valid()） |

**匹配「分支 3」：调用 `get_initial()`**

### 3.4 `get_initial()` 的实现

```python
# rest_framework/serializers.py
def get_initial(self):
    if hasattr(self, 'initial_data'):
        return dict(to_primitive(self.initial_data))
    return OrderedDict(
        [
            (field_name, field.get_initial())
            for field_name, field in self.fields.items()
            if field_name not in self.read_only_fields
        ]
    )
```

**关键特性**：
- 只遍历字段定义，**不访问 `instance`**
- 过滤掉 `Meta.read_only_fields` 中的字段
- 调用 `field.get_initial()` 返回字段初始值（未指定则为 `None`）

### 3.5 `IssueRelationSerializer` 的字段定义

```python
# apps/api/plane/app/serializers/issue.py:401-438
class IssueRelationSerializer(BaseSerializer):
    id = serializers.UUIDField(source="related_issue.id", read_only=True)
    project_id = serializers.PrimaryKeyRelatedField(source="related_issue.project_id", read_only=True)
    sequence_id = serializers.IntegerField(source="related_issue.sequence_id", read_only=True)
    name = serializers.CharField(source="related_issue.name", read_only=True)
    relation_type = serializers.CharField(read_only=True)
    state_id = serializers.UUIDField(source="related_issue.state.id", read_only=True)
    priority = serializers.CharField(source="related_issue.priority", read_only=True)
    assignee_ids = serializers.ListField(..., write_only=True, required=False)

    class Meta:
        model = IssueRelation
        fields = [
            "id", "project_id", "sequence_id", "relation_type", "name", 
            "state_id", "priority", "assignee_ids", 
            "created_by", "created_at", "updated_at", "updated_by"
        ]
        read_only_fields = [
            "workspace", "project", "created_by", "created_at", "updated_by", "updated_at"
        ]
```

### 3.6 `get_initial()` 的实际返回

| 字段 | `read_only` | 是否在 `Meta.read_only_fields` | 是否包含在结果中 | 值 |
|---|---|---|---|---|
| `id` | ✅ 显式 | ❌ | ✅ | `None` |
| `project_id` | ✅ 显式 | ❌ | ✅ | `None` |
| `sequence_id` | ✅ 显式 | ❌ | ✅ | `None` |
| `name` | ✅ 显式 | ❌ | ✅ | `None` |
| `relation_type` | ✅ 显式 | ❌ | ✅ | `None` |
| `state_id` | ✅ 显式 | ❌ | ✅ | `None` |
| `priority` | ✅ 显式 | ❌ | ✅ | `None` |
| `assignee_ids` | ❌（write_only） | ❌ | ✅ | `[]`（空列表） |
| `created_by` | ❌ | ✅ | ❌ | - |
| `created_at` | ❌ | ✅ | ❌ | - |
| `updated_by` | ❌ | ✅ | ❌ | - |
| `updated_at` | ❌ | ✅ | ❌ | - |

**实际返回值**：
```python
OrderedDict([
    ('id', None),
    ('project_id', None),
    ('sequence_id', None),
    ('name', None),
    ('relation_type', None),
    ('state_id', None),
    ('priority', None),
    ('assignee_ids', []),
])
```

### 3.7 `json.dumps()` 的结果

```python
json.dumps(
    OrderedDict([('id', None), ..., ('assignee_ids', [])]),
    cls=DjangoJSONEncoder
)
# → '{"id": null, "project_id": null, ..., "assignee_ids": []}'
```

✅ **完全合法，不抛出任何异常**

---

## 4. 异常抛出点确认：第 272 行 delete() 调用

### 4.1 异常类型

```python
issue_relations = None
issue_relations.delete()
# ↑ 抛出: AttributeError: 'NoneType' object has no attribute 'delete'
```

### 4.2 调用栈

```
AttributeError: 'NoneType' object has no attribute 'delete'
  File "plane/app/views/issue/relation.py", line 272, in remove_relation
    issue_relations.delete()
```

### 4.3 为何不在第 271 行抛出？

**`to_representation` 与 `get_initial` 的本质区别**：

| 方法 | 调用时机 | 行为 | 是否访问 instance |
|---|---|---|---|
| `to_representation(instance)` | instance 存在时序列化 | 遍历字段，调用 `field.get_attribute(instance)` → 最终 `getattr(instance, attr)` | ✅ 访问 |
| `get_initial()` | instance 不存在时返回初始值 | 遍历字段定义，调用 `field.get_initial()` | ❌ 不访问 |

只有 `to_representation` 会通过 `get_attribute` → `getattr` 链访问 `instance` 的属性。`get_initial()` 纯基于字段定义，与 `instance` 无关。

---

## 5. 完整异常处理链路

### 5.1 三层异常处理器

异常从抛出到 HTTP 响应经过三层处理：

```
AttributeError 抛出（第 272 行）
    ↓
Layer 1: DRF APIView.handle_exception()
    ↓
Layer 2: Plane BaseViewSet.handle_exception()
    ↓
Layer 3: DRF auth_exception_handler()
    ↓
HTTP 500 响应返回
```

### 5.2 Layer 1：DRF `APIView.handle_exception()`

```python
# rest_framework/views.py（简化）
def handle_exception(self, exc):
    # 转换内置异常为 DRF APIException
    if isinstance(exc, Http404):
        exc = exceptions.NotFound()
    elif isinstance(exc, PermissionDenied):
        exc = exceptions.PermissionDenied()
    
    # 如果是 APIException，返回结构化 Response
    if isinstance(exc, exceptions.APIException):
        ...
        return Response(data, status=exc.status_code, headers=headers)
    # 否则重新抛出
    else:
        self.set_rollback()
        raise  # ← AttributeError 从这里重新抛出！
```

`AttributeError` 不是 `APIException` 子类 → **重新抛出**。

### 5.3 Layer 2：Plane `BaseViewSet.handle_exception()`

```python
# apps/api/plane/app/views/base.py:70-109
def handle_exception(self, exc):
    try:
        # 调用 DRF 的 handle_exception，它会重新抛出 AttributeError
        response = super().handle_exception(exc)
        return response
    except Exception as e:
        # ← AttributeError 在这里被捕获！
        (print(e, traceback.format_exc()) if settings.DEBUG else print("Server Error"))
        
        if isinstance(e, IntegrityError):
            return Response({"error": "The payload is not valid"}, status=400)
        if isinstance(e, ValidationError):
            return Response({"error": "Please provide valid detail"}, status=400)
        if isinstance(e, ObjectDoesNotExist):
            return Response({"error": "The required object does not exist."}, status=404)
        if isinstance(e, KeyError):
            log_exception(e)
            return Response({"error": "The required key does not exist."}, status=400)
        
        # ← AttributeError 不属于以上任何类型，进入兜底分支
        log_exception(e)
        return Response(
            {"error": "Something went wrong please try again later"},
            status=status.HTTP_500_INTERNAL_SERVER_ERROR,
        )
```

### 5.4 Layer 3：`auth_exception_handler()`

```python
# apps/api/plane/authentication/adapter/exception.py:17-34
def auth_exception_handler(exc, context):
    response = exception_handler(exc, context)
    
    # 只处理 NotAuthenticated 和 Throttled
    if isinstance(exc, NotAuthenticated):
        response.status_code = 401
    if isinstance(exc, Throttled):
        ...
    
    return response  # ← 对于 500 响应，直接原样返回
```

这一层不会修改 500 响应。

### 5.5 异常链路图

```
┌─────────────────────────────────────────────────────────┐
│  remove_relation()                                       │
│  Line 272: issue_relations.delete()                      │
│  → AttributeError: 'NoneType' object has no attribute 'delete' │
└────────────────────────────────┬────────────────────────┘
                                 │ 抛出
                                 ▼
┌─────────────────────────────────────────────────────────┐
│  DRF APIView.handle_exception()                          │
│  isinstance(exc, APIException) → False                   │
│  → re-raise                                              │
└────────────────────────────────┬────────────────────────┘
                                 │ 重新抛出
                                 ▼
┌─────────────────────────────────────────────────────────┐
│  BaseViewSet.handle_exception()                          │
│  try: super().handle_exception() → 抛出                  │
│  except Exception as e:                                  │
│    isinstance(e, IntegrityError) → False                 │
│    isinstance(e, ValidationError) → False                │
│    isinstance(e, ObjectDoesNotExist) → False             │
│    isinstance(e, KeyError) → False                       │
│    → 兜底分支                                             │
│    log_exception(e)                                      │
│    return Response({"error": "Something went wrong..."}, │
│                    status=500)                           │
└────────────────────────────────┬────────────────────────┘
                                 │ 返回 500 Response
                                 ▼
┌─────────────────────────────────────────────────────────┐
│  auth_exception_handler()                                │
│  不修改 500 响应，直接返回                                │
└────────────────────────────────┬────────────────────────┘
                                 ▼
                        HTTP 500 Response
```

---

## 6. HTTP 响应最终形态

### 6.1 开发环境（`DEBUG=True`）

```
HTTP/1.1 500 Internal Server Error
Content-Type: application/json
Vary: Accept, Cookie
Allow: POST, OPTIONS

{
  "error": "Something went wrong please try again later"
}
```

**附加特征**：
- 服务端控制台打印完整 traceback
- `log_exception(e)` 记录错误日志

### 6.2 生产环境（`DEBUG=False`）

```
HTTP/1.1 500 Internal Server Error
Content-Type: application/json
Vary: Accept, Cookie
Allow: POST, OPTIONS

{
  "error": "Something went wrong please try again later"
}
```

**附加特征**：
- 服务端控制台仅打印 `"Server Error"`
- `log_exception(e)` 记录错误日志

### 6.3 与先前误判的对比

| 项 | 先前误判 | 核实后 |
|---|---|---|
| 异常抛出点 | 第 271 行（序列化） | 第 272 行（delete()） |
| HTTP 状态码 | 500 | 500 ✅（一致） |
| Content-Type | `text/html`（Django 默认错误页） | `application/json` ✅（结构化 JSON） |
| 返回体 | HTML 错误页字符串 | `{"error": "Something went wrong please try again later"}` |
| 全局异常处理器 | 不处理 AttributeError | ✅ BaseViewSet.handle_exception 处理并返回结构化 JSON |

---

## 7. 前端 Store 与 UI 的实际影响

### 7.1 前端 Store 错误处理

```typescript
// apps/web/core/store/issue/issue-details/relation.store.ts:227-267
removeRelation = async (workspaceSlug, projectId, issueId, relationType, related_issue, updateLocally = false) => {
    try {
        // 1. 乐观更新
        const relationIndex = this.relationMap[issueId]?.[relationType]?.findIndex(
            (_issueId) => _issueId === related_issue
        );
        if (relationIndex >= 0)
            runInAction(() => {
                this.relationMap[issueId]?.[relationType]?.splice(relationIndex, 1);
            });

        // 2. 调用后端
        if (!updateLocally) {
            await this.issueRelationService.deleteIssueRelation(workspaceSlug, projectId, issueId, {
                relation_type: relationType,
                related_issue,
            });
        }

        // 3. 清理反向关系（有 bug）
        ...

        // 4. 刷新活动记录
        this.rootIssueDetailStore.activity.fetchActivities(workspaceSlug, projectId, issueId);
    } catch (error) {
        // ← 500 错误在这里被捕获
        this.fetchRelations(workspaceSlug, projectId, issueId);  // ✅ 重新拉取，恢复状态
        throw error;  // ↑ 向上抛出
    }
};
```

### 7.2 Axios 错误对象结构

由于返回体是合法的 JSON，`error.response.data` 是一个对象而非 HTML 字符串：

```typescript
// Axios error 对象结构
{
  response: {
    status: 500,
    statusText: "Internal Server Error",
    headers: { ... },
    data: {
      error: "Something went wrong please try again later"
    }
  },
  message: "Request failed with status code 500",
  ...
}
```

这意味着前端可以安全地访问 `error.response?.data?.error` 来显示错误信息。

### 7.3 UI 层表现

```typescript
// apps/web/core/components/issues/relations/issue-list-item.tsx:116-120
const handleRemoveRelation = (e: React.MouseEvent) => {
    e.preventDefault();
    e.stopPropagation();
    removeRelation(workspaceSlug, projectId, issueId, relationKey, relationIssueId);
    // ❌ 无 try-catch
    // ❌ 无 toast 错误提示
    // ❌ 无 loading 状态
};
```

**用户体验链**：
```
用户点击"Remove relation"
    ↓
handleRemoveRelation 调用 removeRelation()
    ↓
Store 乐观更新：relationMap 中移除 B → UI 立即"消失"
    ↓
API 请求发出...
    ↓
HTTP 500 返回：{"error": "Something went wrong..."}
    ↓
catch 块执行：
    ├─ fetchRelations() → 重新拉取全量数据
    └─ throw error 向上抛出（无人捕获）
            ↓
UI 重新渲染：
    └─ relationMap 被 fetchRelations 恢复 → UI 显示关系"回来"
            ↓
用户体验：
    - 关系先消失，~500ms 后又出现
    - 控制台有未捕获的 Promise 错误
    - 无任何用户可见的错误提示
    - 但最终数据与服务端一致
```

---

## 8. 对循环依赖拆解的影响

### 8.1 正常环路拆解

```
初始状态: A blocked_by B → B blocked_by C → C blocked_by A
数据库：
  记录1: issue=A, related=B, type="blocked_by"
  记录2: issue=B, related=C, type="blocked_by"
  记录3: issue=C, related=A, type="blocked_by"

用户在 A 的详情页删除与 B 的 blocked_by 关系
    ↓
POST /remove-relation/ {related_issue: "B"}
    ↓
后端：
  .filter(issue=A, related=B) → 匹配记录1
  .first() → 记录1
  序列化 → {"id": "B-id", ...}
  .delete() → 软删除记录1
  issue_activity.delay(...) → 创建删除活动
  返回 204
    ↓
✅ 成功：环路断裂，A 不再被 B 阻塞
```

### 8.2 并发删除场景

```
初始状态: A↔B↔C↔A
    ↓
用户1: 在 A 的详情页删除与 B 的关系
用户2: 也在 A 的详情页删除与 B 的关系
    ↓
请求1先到达：
  匹配记录1 → 删除 → 204 返回
    ↓
请求2后到达：
  .filter(...) → 空 QuerySet（记录1已被软删除）
  .first() → None
  序列化 → {"id": null, ...}
  .delete() → AttributeError
  返回 500 {"error": "Something went wrong..."}
    ↓
用户1体验：✅ 成功，关系消失
用户2体验：⚠️ 关系先消失，后复现（无错误提示）
    ↓
最终结果：✅ 记录1已被删除，环路已断裂
```

### 8.3 双向多类型关系场景

```
记录1: issue=A, related=B, type="blocked_by"  (A 被 B 阻塞)
记录2: issue=B, related=A, type="relates_to"  (B 与 A 关联)

用户意图：删除 A 被 B 阻塞的关系
    ↓
POST /remove-relation/ {related_issue: "B"}
    ↓
.filter(
    Q(issue=B, related=A)   # 匹配记录2
    | Q(issue=A, related=B)  # 匹配记录1
)
    ↓
.first() → 取决于数据库返回顺序
    ↓
可能结果1：返回记录1（blocked_by）
  → ✅ 删除正确，环路断裂
    ↓
可能结果2：返回记录2（relates_to）
  → ❌ 删除了 relates_to，阻塞关系仍在！
  → 但不会抛出异常（关系存在）
  → 关系仍在，UI 显示"成功"（因为 A.relates_to 列表中确实移除了 B）
  → **用户以为删除了阻塞关系，实际没有**
```

**这才是真正的风险**：静默删除了错误的关系类型，用户无感知。

### 8.4 风险矩阵

| 场景 | 是否抛异常 | HTTP 状态 | 最终数据一致性 | 用户体验 |
|---|---|---|---|---|
| 正常删除（关系存在） | ❌ 否 | 204 | ✅ 一致 | ✅ 正常 |
| 并发删除（后到的请求） | ✅ 是 | 500 | ✅ 一致（有 fetchRelations 兜底） | ⚠️ 困惑（先消失后复现） |
| 关系不存在（用户误操作） | ✅ 是 | 500 | ✅ 一致 | ⚠️ 困惑 |
| 双向多类型（误删错误类型） | ❌ 否 | 204 | ❌ 不一致（阻塞关系仍在） | ✅ 显示"成功"（但实际未达预期） |

---

## 9. 证据链总结

### 9.1 关键证据节点

| 结论 | 证据来源 |
|---|---|
| `Serializer(None).data` 不会报错 | DRF 源码 `.data` 属性逻辑：`instance=None` 时调用 `get_initial()` |
| `get_initial()` 不访问 instance | DRF 源码 `get_initial()` 实现 |
| 异常在 `delete()` 抛出 | `None.delete()` → `AttributeError` |
| 返回结构化 JSON 而非 HTML | `BaseViewSet.handle_exception()` 兜底分支返回 `Response({"error": ...}, 500)` |
| Content-Type 是 application/json | DRF `Response` 默认 Content-Type |
| 前端有数据一致性兜底 | `catch` 块中 `fetchRelations()` 重新拉取 |
| 双向多类型场景存在静默误删 | `remove_relation` 忽略 `relation_type`，只用双向 OR 查询 + `.first()` |

### 9.2 先前分析的修正点

| 先前结论 | 修正后 | 证据 |
|---|---|---|
| 第 271 行抛出 AttributeError | ❌ 第 271 行不报错 | DRF `.data` 属性调用 `get_initial()` 不访问 instance |
| 返回 HTML 错误页 | ❌ 返回结构化 JSON | `BaseViewSet.handle_exception()` 返回 `Response({"error": ...}, 500)` |
| 全局异常处理器不处理 | ❌ 被 BaseViewSet 处理 | `BaseViewSet.handle_exception()` 兜底分支 |
| 异常链路不影响数据一致性 | ✅ 保留，但补充：**双向多类型场景会静默误删** | `.first()` 返回不确定的记录 |

---

## 10. 关键文件索引

| 文件 | 行号 | 核心内容 |
|---|---|---|
| `apps/api/plane/app/views/issue/relation.py` | 270-272 | 异常抛出点：`.first()` → None → `delete()` → AttributeError |
| `apps/api/plane/app/views/base.py` | 70-109 | `BaseViewSet.handle_exception()` — 兜底分支返回 500 + JSON |
| `apps/api/plane/app/serializers/issue.py` | 401-438 | `IssueRelationSerializer` 字段定义（均为 read_only=True） |
| `apps/api/plane/authentication/adapter/exception.py` | 17-34 | `auth_exception_handler()` — 不修改 500 响应 |
| `apps/web/core/store/issue/issue-details/relation.store.ts` | 235-266 | 前端 Store：乐观更新 + catch 中 `fetchRelations` 兜底 |
| `apps/web/core/components/issues/relations/issue-list-item.tsx` | 116-120 | UI 层：无 try-catch，无错误提示 |
| `apps/api/plane/db/mixins.py` | 61-82 | `SoftDeleteModel.delete()` — 默认软删除 |

---

## 总结

### 三项核心修正

1. **`IssueRelationSerializer(None).data` 不会报错**
   - DRF `.data` 属性在 `instance=None` 时调用 `get_initial()` 而非 `to_representation()`
   - `get_initial()` 仅基于字段定义返回初始值，不访问 instance
   - 实际返回：`{"id": null, "project_id": null, ..., "assignee_ids": []}`

2. **异常实际抛出点是第 272 行 `issue_relations.delete()`**
   - 类型：`AttributeError: 'NoneType' object has no attribute 'delete'`
   - HTTP 响应：`500 Internal Server Error` + `{"error": "Something went wrong please try again later"}`
   - Content-Type 是 `application/json`，可被前端安全解析

3. **真正的风险是双向多类型场景下的静默误删**
   - `remove_relation` 忽略 `relation_type`，只用双向 OR 查询
   - `.first()` 返回哪条记录取决于数据库排序
   - 如果返回了错误类型的关系，会**静默删除**（返回 204），用户以为成功了，实际阻塞关系仍在
   - 这比 500 错误更危险，因为用户无感知

### 对循环依赖拆解的影响

- **500 错误不影响环路可解除性**：环路是多条边的组合，只要成功删除一条边就会断裂。某条删除失败只是 UI 体验问题。
- **真正的风险是静默误删**：同一对 Issue 存在双向多类型关系时，可能删除了错误的关系类型，用户以为解除了阻塞，实际没有。
- **前端有数据一致性兜底**：`catch` 块中 `fetchRelations()` 会重新拉取全量数据，最终 UI 与服务端一致。
