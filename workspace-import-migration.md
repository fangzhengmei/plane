# Plane 工作区与项目导入 / 迁移代码理解

## 一、整体架构概览

Plane 的导入/迁移体系在代码层面涉及两套 API，但两者的实际可达性存在差异：

| 层级 | 用途 | 实际可达性 |
|------|------|------------|
| **API v1 (Public API)** | 通用外部系统集成、批量写入 | 路由已挂载，但 **PUT Upsert 端点未暴露** |
| **App API (内置导入器)** | GitHub / Jira 原生集成向导 | **后端 View/Service 未实现**，仅有 Model/Serializer 壳 |

### 关键文件索引

| 类别 | 相对路径 |
|------|----------|
| Importer 模型 | `apps/api/plane/db/models/importer.py` |
| Importer 序列化器 | `apps/api/plane/app/serializers/importer.py` |
| Exporter 模型 | `apps/api/plane/db/models/exporter.py` |
| API v1 Issue View（含 PUT Upsert 代码） | `apps/api/plane/api/views/issue.py` |
| API v1 Issue 序列化器 | `apps/api/plane/api/serializers/issue.py` |
| API v1 URL 路由定义 | `apps/api/plane/api/urls/work_item.py` |
| App API Issue 视图 | `apps/api/plane/app/views/issue/base.py` |
| App API IssueCreateSerializer | `apps/api/plane/app/serializers/issue.py` |
| 导出任务 (Celery) | `apps/api/plane/bgtasks/export_task.py` |
| Porters 导出框架 | `apps/api/plane/utils/porters/exporter.py` |
| Porters 格式化器 (CSV/JSON/XLSX) | `apps/api/plane/utils/porters/formatters.py` |
| Exporter 视图 | `apps/api/plane/app/views/exporter/base.py` |
| Exporter URL 路由 | `apps/api/plane/app/urls/exporter.py` |
| Jira 导入前端服务 | `apps/web/core/services/integrations/jira.service.ts` |
| GitHub 导入前端服务 | `apps/web/core/services/integrations/github.service.ts` |
| 集成服务 (列出/删除导入器) | `apps/web/core/services/integrations/integration.service.ts` |
| Jira 导入类型 | `packages/types/src/importer/jira-importer.ts` |
| GitHub 导入类型 | `packages/types/src/importer/github-importer.ts` |
| 导入器通用类型 | `packages/types/src/importer/index.ts` |
| 导出前端服务 | `apps/web/core/services/project/project-export.service.ts` |
| 导出历史前端 | `apps/web/core/components/exporter/prev-exports.tsx` |
| Issue 模型 (external_id) | `apps/api/plane/db/models/issue.py` |
| Label 模型 (external_id) | `apps/api/plane/db/models/label.py` |
| State 模型 (external_id) | `apps/api/plane/db/models/state.py` |
| Project 模型 (external_id) | `apps/api/plane/db/models/project.py` |
| DraftIssue 模型 (external_id) | `apps/api/plane/db/models/draft.py` |
| Module 模型 (external_id) | `apps/api/plane/db/models/module.py` |
| Cycle 模型 (external_id) | `apps/api/plane/db/models/cycle.py` |
| IntakeIssue 模型 (external_id) | `apps/api/plane/db/models/intake.py` |
| Page 模型 (external_id) | `apps/api/plane/db/models/page.py` |
| IssueType 模型 (external_id) | `apps/api/plane/db/models/issue_type.py` |
| FileAsset 模型 (external_id) | `apps/api/plane/db/models/asset.py` |
| App API 全局 URL 注册 | `apps/api/plane/app/urls/__init__.py` |
| 根 URL 配置 | `apps/api/plane/urls.py` |
| API v1 Base View | `apps/api/plane/api/views/base.py` |

---

## 二、路由可达性审计

### 2.1 API v1 PUT Upsert — 已编码但未挂载

`IssueDetailAPIEndpoint` 类中定义了 `put(self, request, slug, project_id)` 方法（`issue.py` L588-L715），实现了基于 `external_id` + `external_source` 的 Upsert 逻辑。然而在 URL 路由定义中：

```python
# apps/api/plane/api/urls/work_item.py L104-L108
path(
    "workspaces/<str:slug>/projects/<uuid:project_id>/work-items/<uuid:pk>/",
    IssueDetailAPIEndpoint.as_view(http_method_names=["get", "patch", "delete"]),
    name="work-item-detail",
)
```

**问题 1：`http_method_names` 不包含 `"put"`**

DRF 的 `as_view(http_method_names=...)` 参数严格限制允许的 HTTP 方法。当前只允许 `get`/`patch`/`delete`，PUT 请求会收到 `405 Method Not Allowed`。

**问题 2：URL 参数签名不匹配**

URL 模式包含 `<uuid:pk>`，但 `put` 方法签名为 `put(self, request, slug, project_id)`，不接受 `pk` 参数。即使将 `"put"` 加入 `http_method_names`，调用时也会因参数不匹配而报错。该 `put` 方法的设计意图是通过请求体中的 `external_id` + `external_source` 定位资源，而非 URL 路径中的 `pk`。

**问题 3：列表端点同样未开放 PUT**

```python
# apps/api/plane/api/urls/work_item.py L99-L103
path(
    "workspaces/<str:slug>/projects/<uuid:project_id>/work-items/",
    IssueListCreateAPIEndpoint.as_view(http_method_names=["get", "post"]),
    name="work-item-list",
)
```

列表端点仅允许 `get` 和 `post`，且 `IssueListCreateAPIEndpoint` 类中没有 `put` 方法定义。

**结论：PUT Upsert 端点当前不可达。** 如需使用，需要在路由层做以下改动之一：
- 在列表 URL 上为 `IssueDetailAPIEndpoint` 新增一条 PUT 路由（不带 `<uuid:pk>`），并将 `"put"` 加入 `http_method_names`
- 或将 `"put"` 加入 detail URL 的 `http_method_names`，同时调整 `put` 方法签名以接受 `pk` 参数

### 2.2 GitHub / Jira 导入器后端 — Model/Serializer 就绪但 View 未实现

**前端调用的 API 端点：**

| 前端方法 | 调用路径 | 后端状态 |
|----------|----------|----------|
| `getJiraProjectInfo` | `GET /api/workspaces/{slug}/importers/jira` | ❌ 无对应 View |
| `createJiraImporter` | `POST /api/workspaces/{slug}/projects/importers/jira/` | ❌ 无对应 View |
| `getGithubRepoInfo` | `GET /api/workspaces/{slug}/importers/github/` | ❌ 无对应 View |
| `createGithubServiceImport` | `POST /api/workspaces/{slug}/projects/importers/github/` | ❌ 无对应 View |
| `getImporterServicesList` | `GET /api/workspaces/{slug}/importers/` | ❌ 无对应 View |
| `deleteImporterService` | `DELETE /api/workspaces/{slug}/importers/{service}/{importerId}/` | ❌ 无对应 View |

**已存在的代码：**

- `Importer` 模型（`db/models/importer.py`）— 完整定义了 `service`, `status`, `metadata`, `config`, `data`, `token`, `imported_data` 字段
- `ImporterSerializer`（`app/serializers/importer.py`）— 简单的 `BaseSerializer` 子类
- 数据库迁移（`db/migrations/0023_auto_20230316_0040.py`）— `importers` 表已创建
- `external.py` 中有 `name="importer"` 的 URL，但实际指向 GPT AI 助手端点，与导入器无关

**不存在的代码：**

- 无 `importer` 相关的 View 类（在 `app/views/` 和 `api/views/` 目录下均无）
- 无 `importer` 相关的 URL 路由（在所有 `urls/*.py` 中无 `importers/` 路径）
- 无 `importer` 相关的 Celery 后台任务（在 `bgtasks/` 目录下无 import 相关任务）
- 无 `importer` 相关的 Service 层（在 `utils/` 目录下无 import service）

**结论：GitHub/Jira 导入器的后端执行逻辑未在本仓库中实现。** 前端类型定义和服务调用已就绪，但后端处理程序缺失。推测这些端点由 Plane 的云端服务（非开源部分）提供，或属于企业版功能。

### 2.3 导出流程 — 完整实现

作为对比参照，导出功能在后端有完整实现：

```python
# apps/api/plane/app/urls/exporter.py
path("workspaces/<str:slug>/export-issues/", ExportIssuesEndpoint.as_view(), name="export-issues")
```

- `ExportIssuesEndpoint`（`app/views/exporter/base.py`）— POST 触发导出 + GET 分页查询历史
- `issue_export_task`（`bgtasks/export_task.py`）— Celery 异步任务
- `DataExporter` + `IssueExportSerializer` + `formatters` — 数据序列化与格式化

---

## 三、数据模型层 — external_source / external_id 机制

### 3.1 核心字段设计

Plane 在几乎所有可迁移实体上都保留了 `external_source` 和 `external_id` 两个 CharField：

```python
external_source = models.CharField(max_length=255, null=True, blank=True)
external_id     = models.CharField(max_length=255, blank=True, null=True)
```

**拥有此字段的模型清单：**

| 模型 | 文件 | 说明 |
|------|------|------|
| `Issue` | `db/models/issue.py` | 工作项 |
| `IssueVersion` | `db/models/issue.py` | 工作项版本快照 |
| `IssueAttachment` | `db/models/issue.py` | 工作项附件 |
| `DraftIssue` | `db/models/draft.py` | 草稿工作项 |
| `Label` | `db/models/label.py` | 标签 |
| `State` | `db/models/state.py` | 状态 |
| `Project` | `db/models/project.py` | 项目 |
| `Module` | `db/models/module.py` | 模块 |
| `Cycle` | `db/models/cycle.py` | 周期 |
| `IntakeIssue` | `db/models/intake.py` | 收件箱工作项 |
| `Page` | `db/models/page.py` | 页面 |
| `IssueType` | `db/models/issue_type.py` | 工作项类型 |
| `FileAsset` | `db/models/asset.py` | 文件资产 |

**设计意图：** `external_source` 标识来源系统（如 `"jira"`, `"github"`），`external_id` 存储源系统中的唯一标识。两者组合形成跨系统的幂等键。

### 3.2 Importer 模型

`Importer` 继承自 `ProjectBaseModel`（关联 project + workspace），记录单次导入任务：

| 字段 | 类型 | 说明 |
|------|------|------|
| `service` | CharField(choices) | `"github"` 或 `"jira"` |
| `status` | CharField(choices) | `queued → processing → completed / failed`，默认 `queued` |
| `initiated_by` | FK → User | 发起导入的用户 |
| `metadata` | JSONField | 连接凭据（Jira hostname/api_token/email 或 GitHub owner/repo） |
| `config` | JSONField | 导入配置（如 `epics_to_modules`, `sync`） |
| `data` | JSONField | 预览与映射数据（用户映射决策、计数统计） |
| `token` | FK → APIToken | 导入过程使用的 API Token |
| `imported_data` | JSONField(null) | 导入结果摘要 |

数据库表名：`importers`，按 `created_at` 倒序排列。

### 3.3 ExporterHistory 模型

记录导出任务，有完整实现：

| 字段 | 说明 |
|------|------|
| `provider` | `"json"` / `"csv"` / `"xlsx"` |
| `status` | `queued → processing → completed / failed` |
| `token` | 唯一 hex token，用于 S3 路径和查询 |
| `url` | 完成后的预签名下载 URL |
| `key` | S3 对象 key |
| `reason` | 失败原因 |

---

## 四、幂等与重入机制

### 4.1 API v1 — PUT Upsert 逻辑（代码存在但路由未挂载）

这是**设计上最核心的幂等写入路径**，位于 `IssueDetailAPIEndpoint.put`：

```
PUT /api/v1/workspaces/{slug}/projects/{project_id}/work-items/
Body: { external_id, external_source, name, ... }
```

**⚠️ 当前不可达 — 详见第二章路由审计**

**流程：**

1. 从请求体提取 `external_id` + `external_source`（**两者均为必填**，否则返回 400）
2. 尝试 `Issue.objects.get(project_id=..., external_id=..., external_source=...)`
3. **若存在** → 执行更新（partial update），返回 `200 OK`
4. **若不存在** (`DoesNotExist`) → 执行创建，返回 `201 Created`
5. 创建时额外支持覆盖 `created_at` 和 `created_by` 字段（保留源系统时间戳和作者）

**关键代码片段：**

```python
# issue.py L588-L715
def put(self, request, slug, project_id):
    project = Project.objects.get(pk=project_id)
    external_id = request.data.get("external_id")
    external_source = request.data.get("external_source")

    if external_id and external_source:
        try:
            issue = Issue.objects.get(
                project_id=project_id,
                workspace__slug=slug,
                external_id=external_id,
                external_source=external_source,
            )
            # UPDATE path
            current_instance = json.dumps(IssueSerializer(issue).data, cls=DjangoJSONEncoder)
            serializer = IssueSerializer(issue, data=request.data, partial=True, context={...})
            if serializer.is_valid():
                serializer.save()
                issue_activity.delay(type="issue.activity.updated", ...)
                return Response(serializer.data, status=status.HTTP_200_OK)
            return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)
        except Issue.DoesNotExist:
            # CREATE path
            serializer = IssueSerializer(data=request.data, context={...})
            if serializer.is_valid():
                serializer.save()
                issue = Issue.objects.filter(...).first()
                issue.created_at = request.data.get("created_at", timezone.now())
                issue.created_by_id = request.data.get("created_by", request.user.id)
                issue.save(update_fields=["created_at", "created_by"])
                issue_activity.delay(type="issue.activity.created", ...)
                return Response(serializer.data, status=status.HTTP_201_CREATED)
            return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)
    else:
        return Response(
            {"error": "external_id and external_source are required"},
            status=status.HTTP_400_BAD_REQUEST,
        )
```

### 4.2 API v1 — POST 创建时的去重检查

`IssueListCreateAPIEndpoint.post`（可达）：

```python
if request.data.get("external_id") and request.data.get("external_source"):
    if Issue.objects.filter(
        project_id=project_id,
        workspace__slug=slug,
        external_source=request.data.get("external_source"),
        external_id=request.data.get("external_id"),
    ).exists():
        return Response(
            {"error": "Issue with the same external id and external source already exists", "id": str(issue.id)},
            status=status.HTTP_409_CONFLICT,
        )
```

**语义：** POST 是"仅创建"，如果 external_id 已存在则返回 `409 Conflict` 并附带已有 issue ID。

### 4.3 API v1 — PATCH 更新时的 external_id 唯一性校验

`IssueDetailAPIEndpoint.patch`（可达）：

```python
if request.data.get("external_id") and (issue.external_id != str(request.data.get("external_id"))):
    if Issue.objects.filter(
        project_id=project_id,
        workspace__slug=slug,
        external_source=request.data.get("external_source", issue.external_source),
        external_id=request.data.get("external_id"),
    ).exists():
        return Response({"error": "...", "id": str(issue.id)}, status=status.HTTP_409_CONFLICT)
```

### 4.4 API v1 — GET 通过 external_id 查询

`IssueListCreateAPIEndpoint.get`（可达）：

```python
external_id = request.GET.get("external_id")
external_source = request.GET.get("external_source")
if external_id and external_source:
    issue = Issue.objects.get(
        external_id=external_id,
        external_source=external_source,
        workspace__slug=slug,
        project_id=project_id,
    )
    return Response(IssueSerializer(issue).data)
```

### 4.5 Label 的幂等保护

`LabelListCreateAPIEndpoint.post`（可达）同样实现了：
- 创建时检查 `external_id` + `external_source` 去重 → 返回 `409`
- 数据库层面 `IntegrityError` → 按名称去重 → 返回 `409` 并附带已有 label ID

### 4.6 Issue 序列号分配的并发安全

`Issue.save` 在创建新 Issue 时使用 PostgreSQL `pg_advisory_xact_lock` 保证同一项目内 `sequence_id` 递增的原子性：

```python
with transaction.atomic():
    lock_key = convert_uuid_to_integer(self.project.id)
    with connection.cursor() as cursor:
        cursor.execute("SELECT pg_advisory_xact_lock(%s)", [lock_key])
    last_sequence = IssueSequence.objects.filter(project=self.project).aggregate(
        largest=models.Max("sequence")
    )["largest"]
    self.sequence_id = last_sequence + 1 if last_sequence else 1
```

**注意：** 此锁是事务级排他锁，事务结束自动释放，避免死锁风险。

### 4.7 多对多关系的幂等写入

`IssueCreateSerializer.create` 中，`IssueAssignee` 和 `IssueLabel` 的 `bulk_create` 使用 `IntegrityError` 异常捕获：

```python
IssueAssignee.objects.bulk_create([...], batch_size=10)
# IntegrityError 被 try/except 捕获并忽略
```

同时这两张表都有 `UniqueConstraint(condition=Q(deleted_at__isnull=True))`，软删除记录不参与唯一约束，实现"删除后可重建"。

### 4.8 重入策略总结

| 场景 | 策略 | HTTP 状态码 | 端点可达性 |
|------|------|-------------|------------|
| 首次创建 (POST) | 正常创建 | 201 | ✅ 可达 |
| 重复创建 (POST + external_id 冲突) | 拒绝，返回已有 ID | 409 | ✅ 可达 |
| Upsert (PUT + external_id) | 存在则更新，不存在则创建 | 200 / 201 | ❌ **路由未挂载** |
| 局部更新 (PATCH + pk) | 按主键更新，external_id 变更时校验 | 200 / 409 | ✅ 可达 |
| 按 external_id 查询 (GET) | 精确查询单条 | 200 | ✅ 可达 |
| Assignee/Label 关联重复 | IntegrityError 静默跳过 | - | ✅ 可达 |
| Sequence ID 并发 | pg_advisory_xact_lock | - | ✅ 可达 |

---

## 五、跨工作区资源映射策略

### 5.1 成员映射

Jira 和 GitHub 导入流程均采用"预览 + 用户决策"模式，该逻辑体现在前端类型定义中：

**Jira 端类型（`jira-importer.ts`）：**

```typescript
interface User {
  username: string;
  import: "invite" | "map" | false;  // 三种策略
  email: string;
}
```

**GitHub 端类型（`github-importer.ts`）：**

```typescript
interface IGithubServiceImportFormData {
  data: {
    users: {
      username: string;
      import: boolean | "invite" | "map";
      email: string;
    }[];
  };
}
```

三种映射策略：
- `"invite"` — 向该用户发送工作区邀请邮件，邀请加入后自动映射
- `"map"` — 将源系统用户映射到当前工作区已有成员
- `false` — 不映射，该用户创建的内容归属到导入操作发起人

**注意：** 这些映射逻辑的实现依赖于后端导入器视图，当前未在本仓库中实现。

### 5.2 标签映射

标签通过 `Label` 模型的 `external_source` + `external_id` 字段实现跨系统关联。Label 的唯一性约束：

```python
# label.py
class Meta:
    constraints = [
        models.UniqueConstraint(
            fields=["name"],
            condition=Q(project__isnull=True, deleted_at__isnull=True),
            name="unique_name_when_project_null_and_not_deleted",
        ),
        models.UniqueConstraint(
            fields=["project", "name"],
            condition=Q(project__isnull=False, deleted_at__isnull=True),
            name="unique_project_name_when_not_deleted",
        ),
    ]
```

**策略：** 同一项目内标签名唯一。导入时如果同名标签已存在（带不同 external_id），API v1 的 Label 创建端点会返回 `409 Conflict` 并附带已有 label ID，调用方可直接复用该 ID 建立 Issue-Label 关联。

### 5.3 附件映射

`FileAsset` 模型有 `external_source` + `external_id`。附件导入需要：

1. 从源系统下载附件文件
2. 通过 FileAsset 上传接口上传至 Plane 的 S3 存储
3. 记录 `external_source` / `external_id` 便于后续增量同步时去重

### 5.4 State 映射

State 模型拥有 `external_source` / `external_id` 以及 `group` 字段（backlog/unstarted/started/completed/cancelled）。导入策略通常为：

- 将源系统状态映射到 Plane 的 5 个 `group`
- 同一 group 下可创建多个 State（如 Jira 的 "In Progress" → Plane State group="started", name="In Progress"）
- 通过 `external_id` 避免重复创建

### 5.5 Module / Cycle 映射

Module 和 Cycle 模型均有 `external_source` / `external_id`。Jira 导入时可配置 `epics_to_modules: true`，将 Jira Epic 映射为 Plane Module。

---

## 六、导入流程详解

### 6.1 内置导入器流程（GitHub / Jira）— 前端就绪、后端缺失

```
┌─────────────┐    ┌──────────────────┐    ┌─────────────────────┐    ┌───────────────┐
│ 1. 连接源系统 │ →  │ 2. 预览源数据     │ →  │ 3. 用户配置映射策略   │ →  │ 4. 执行导入    │
│ (输入凭据)    │    │ (获取项目/仓库信息) │    │ (成员/标签/状态映射)  │    │ (后台异步)     │
└─────────────┘    └──────────────────┘    └─────────────────────┘    └───────────────┘
      ✅ 前端已实现        ✅ 前端已实现          ✅ 前端已实现            ❌ 后端未实现
```

**Step 1 — 连接源系统（前端已实现）：**

- Jira：调用 `GET /api/workspaces/{slug}/importers/jira` 传入 `cloud_hostname`, `api_token`, `email`, `project_key`
- GitHub：调用 `GET /api/workspaces/{slug}/importers/github/` 传入 `owner`, `repo`

**Step 2 — 预览源数据（前端已实现）：**

返回源系统的 issues 计数、labels 计数、collaborators/users 列表等。前端类型 `IJiraResponse` / `IGithubRepoInfo`。

**Step 3 — 用户配置映射（前端已实现）：**

前端展示预览数据，让用户为每个外部用户选择 `invite` / `map` / `false` 策略。

**Step 4 — 执行导入（后端缺失）：**

- Jira：`POST /api/workspaces/{slug}/projects/importers/jira/`
- GitHub：`POST /api/workspaces/{slug}/projects/importers/github/`

请求体包含 `metadata`（连接凭据）、`config`（导入配置）、`data`（用户映射决策）、`project_id`（目标项目）。

这些端点在当前仓库中无对应的 View 实现，推测由 Plane 云端服务处理。

### 6.2 API v1 批量迁移流程 — 可行方案（需注意 PUT 不可达）

```
┌──────────────┐   ┌──────────────┐   ┌───────────────────┐   ┌───────────────┐
│ 1. 创建项目   │ → │ 2. 创建基础   │ → │ 3. 批量创建 Issues  │ → │ 4. 创建关联    │
│ (带 external  │   │  资源          │   │ (POST + 409 处理)  │   │ (评论/附件/链接)│
│  _id)         │   │ (States/Labels)│   │                   │   │               │
└──────────────┘   └──────────────┘   └───────────────────┘   └───────────────┘
```

**Step 1 — 创建目标项目（App API）：**

```http
POST /api/workspaces/{slug}/projects/
{
  "name": "Migrated Project",
  "identifier": "MIG",
  "external_source": "jira",
  "external_id": "PROJ-123"
}
```

**Step 2 — 创建基础资源（API v1 POST，可达）：**

```http
POST /api/v1/workspaces/{slug}/projects/{id}/states/
{
  "name": "In Progress",
  "group": "started",
  "external_source": "jira",
  "external_id": "jira-state-3"
}
```

```http
POST /api/v1/workspaces/{slug}/projects/{id}/labels/
{
  "name": "bug",
  "color": "#FF0000",
  "external_source": "jira",
  "external_id": "jira-label-bug"
}
```

**Step 3 — 批量创建 Issues（API v1 POST，可达）：**

```http
POST /api/v1/workspaces/{slug}/projects/{id}/work-items/
{
  "external_id": "JIRA-456",
  "external_source": "jira",
  "name": "Fix login bug",
  "state": "<uuid-of-mapped-state>",
  "labels": ["<uuid-of-mapped-label>"],
  "assignees": ["<uuid-of-mapped-user>"],
  "description_html": "<p>...</p>"
}
```

**幂等策略调整：** 由于 PUT 不可达，需使用 POST + 409 处理模式：
- 首次调用 POST → 201 Created
- 重复调用 POST → 409 Conflict + 已有 issue ID → 改用 PATCH 更新

```http
PATCH /api/v1/workspaces/{slug}/projects/{id}/work-items/{issue_id}/
{
  "name": "Updated title",
  "state": "<uuid>"
}
```

**Step 4 — 创建关联资源（API v1 POST，可达）：**

```http
POST /api/v1/workspaces/{slug}/projects/{id}/work-items/{issue_id}/comments/
POST /api/v1/workspaces/{slug}/projects/{id}/work-items/{issue_id}/links/
POST /api/v1/workspaces/{slug}/projects/{id}/work-items/{issue_id}/attachments/
```

**注意：** POST 创建 issue 不支持覆盖 `created_at` 和 `created_by`（这两个字段在 `IssueCreateSerializer` 中为 `read_only_fields`），只有 PUT 端点的 create 路径支持此功能。

### 6.3 通过 external_id 查询已有记录（可达）

```http
GET /api/v1/workspaces/{slug}/projects/{id}/work-items/?external_id=JIRA-456&external_source=jira
```

可用于迁移前检测是否已导入，实现自定义的"查询-创建/更新"两步幂等逻辑。

---

## 七、前端可见性策略

### 7.1 导入器列表

前端通过 `IntegrationService.getImporterServicesList()` 获取当前工作区的导入服务列表：

```typescript
// integration.service.ts
async getImporterServicesList(workspaceSlug: string): Promise<IImporterService[]> {
  return this.get(`/api/workspaces/${workspaceSlug}/importers/`)
}
```

使用 SWR 缓存 key `IMPORTER_SERVICES_LIST`，在集成设置页展示。由于后端未实现，此调用在自部署环境下会返回 404。

### 7.2 导出历史轮询（已实现）

`prev-exports.tsx` 实现了自动轮询机制：

```typescript
useEffect(() => {
  const interval = setInterval(() => {
    if (exporterServices?.results?.some((service) => service.status === "processing")) {
      handleRefresh();
    } else {
      clearInterval(interval);
    }
  }, 3000);
  return () => clearInterval(interval);
}, [exporterServices]);
```

**策略：** 每 3 秒轮询一次，只要列表中存在 `status === "processing"` 的记录就继续刷新，全部完成后停止轮询。

### 7.3 权限控制

- **集成/导入页面**：仅工作区 Admin 可见（`EUserPermissions.ADMIN` + `EUserPermissionsLevel.WORKSPACE`）
- **导出页面**：Admin + Member 可见（`EUserPermissions.ADMIN, EUserPermissions.MEMBER`）
- **API v1 端点**：通过 `ProjectEntityPermission` / `ProjectMemberPermission` 控制读写权限，需 API Key 认证（`APIKeyAuthentication`）

### 7.4 导入状态流转

前端 `IImporterService` 类型中 `status` 为联合类型：

```typescript
status: "processing" | "completed" | "failed"
```

后端 `Importer` 模型额外有 `"queued"` 状态。前端展示策略：
- `processing` → 显示进度指示
- `completed` → 显示成功标记
- `failed` → 显示错误信息

### 7.5 SWR 缓存键

| 键 | 用途 |
|------|------|
| `IMPORTER_SERVICES_LIST` | 导入器列表 |
| `JIRA_IMPORTER_DETAIL` | Jira 项目预览信息 |
| `GITHUB_REPOSITORY_INFO` | GitHub 仓库信息 |
| `EXPORT_SERVICES_LIST` | 导出历史列表 |

---

## 八、导出流程（反向参考 — 完整实现）

导出使用 Celery 异步任务，流程清晰，可作为导入实现的参照模式：

1. 用户发起 `POST /api/workspaces/{slug}/export-issues/`，指定 `provider`（csv/json/xlsx）和 `project` 列表
2. 创建 `ExporterHistory` 记录（status=queued, token=auto_generated_hex）
3. 触发 Celery 任务 `issue_export_task.delay(...)`
4. 任务内：status → processing → 使用 `DataExporter` + `IssueExportSerializer` 序列化 → 格式化 → 打包 ZIP → 上传 S3 → status → completed，写入 presigned URL
5. 失败则 status → failed，写入 reason
6. 前端通过 GET 端点分页查询历史，轮询 processing 状态

---

## 九、分批迁移建议

### 9.1 当前可用路径

由于 PUT Upsert 端点未挂载，GitHub/Jira 导入器后端未实现，当前可用的迁移路径为：

**API v1 POST + GET + PATCH 组合模式：**

1. 通过 `GET + external_id` 检查是否已存在
2. 不存在则 `POST` 创建
3. 已存在则用返回的 ID 调用 `PATCH` 更新

### 9.2 如需启用 PUT Upsert

需进行以下改动：

1. **在 `work_item.py` 中新增路由：**

```python
path(
    "workspaces/<str:slug>/projects/<uuid:project_id>/work-items/upsert/",
    IssueDetailAPIEndpoint.as_view(http_method_names=["put"]),
    name="work-item-upsert",
)
```

2. **确认 `put` 方法签名与新路由匹配**（当前签名 `put(self, request, slug, project_id)` 不含 `pk`，与新路由匹配）

### 9.3 分批策略

1. **按项目分批**：每批次迁移一个完整项目（Project → States → Labels → Issues → Comments/Attachments/Links）
2. **项目内按依赖层级**：先创建 States/Labels 等基础资源，再创建 Issues，最后创建关联
3. **自定义幂等**：维护 external_id → Plane UUID 映射表，迁移前先 GET 查询，避免 409

### 9.4 失败补偿

- 单条失败：API 返回 400/409，记录 external_id 便于重试
- 批量中断：使用 GET + external_id 检查已导入记录，对缺失记录重新 POST
- 建议在调用方维护一个外部 ID → Plane UUID 的映射表，加速后续关联创建

### 9.5 注意事项

- POST 创建 issue 不支持覆盖 `created_at` / `created_by`（仅 PUT 的 create 路径支持）
- `sequence_id` 分配使用 `pg_advisory_xact_lock`，大批量写入时需注意事务持有锁的时间
- Label 同名冲突会返回 409，需根据返回的已有 ID 进行映射
- 成员邀请是异步流程（邮件发送），`map` 策略要求用户已是工作区成员
- GitHub/Jira 导入器后端在当前仓库中无实现，若需使用需自行实现或等待官方补充
