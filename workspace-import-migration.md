# Plane 工作区与项目导入 / 迁移代码理解

## 一、整体架构概览

Plane 的导入/迁移体系分为两层：

| 层级 | 用途 | 入口 |
|------|------|------|
| **API v1 (Public API)** | 通用外部系统集成、批量写入、幂等 Upsert | `/api/v1/workspaces/{slug}/projects/{id}/work-items/` |
| **App API (内置导入器)** | GitHub / Jira 原生集成向导 | `/api/workspaces/{slug}/projects/importers/{service}/` |

### 关键文件索引

| 类别 | 文件 |
|------|------|
| Importer 模型 | [importer.py](file:///d:/fz/0508-3/solo-dogfeeding/code/200-plane/apps/api/plane/db/models/importer.py) |
| Importer 序列化器 | [importer.py](file:///d:/fz/0508-3/solo-dogfeeding/code/200-plane/apps/api/plane/app/serializers/importer.py) |
| Exporter 模型 | [exporter.py](file:///d:/fz/0508-3/solo-dogfeeding/code/200-plane/apps/api/plane/db/models/exporter.py) |
| API v1 Issue CRUD + Upsert | [issue.py](file:///d:/fz/0508-3/solo-dogfeeding/code/200-plane/apps/api/plane/api/views/issue.py) |
| API v1 Issue 序列化器 | [issue.py](file:///d:/fz/0508-3/solo-dogfeeding/code/200-plane/apps/api/plane/api/serializers/issue.py) |
| App API Issue 视图 | [base.py](file:///d:/fz/0508-3/solo-dogfeeding/code/200-plane/apps/api/plane/app/views/issue/base.py) |
| App API IssueCreateSerializer | [issue.py](file:///d:/fz/0508-3/solo-dogfeeding/code/200-plane/apps/api/plane/app/serializers/issue.py#L82-L273) |
| 导出任务 (Celery) | [export_task.py](file:///d:/fz/0508-3/solo-dogfeeding/code/200-plane/apps/api/plane/bgtasks/export_task.py) |
| Porters 导出框架 | [exporter.py](file:///d:/fz/0508-3/solo-dogfeeding/code/200-plane/apps/api/plane/utils/porters/exporter.py) |
| Porters 格式化器 (CSV/JSON/XLSX) | [formatters.py](file:///d:/fz/0508-3/solo-dogfeeding/code/200-plane/apps/api/plane/utils/porters/formatters.py) |
| Jira 导入前端服务 | [jira.service.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/200-plane/apps/web/core/services/integrations/jira.service.ts) |
| GitHub 导入前端服务 | [github.service.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/200-plane/apps/web/core/services/integrations/github.service.ts) |
| 集成服务 (列出/删除导入器) | [integration.service.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/200-plane/apps/web/core/services/integrations/integration.service.ts) |
| Jira 导入类型 | [jira-importer.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/200-plane/packages/types/src/importer/jira-importer.ts) |
| GitHub 导入类型 | [github-importer.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/200-plane/packages/types/src/importer/github-importer.ts) |
| 导入器通用类型 | [index.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/200-plane/packages/types/src/importer/index.ts) |
| 导出前端服务 | [project-export.service.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/200-plane/apps/web/core/services/project/project-export.service.ts) |
| 导出历史前端 | [prev-exports.tsx](file:///d:/fz/0508-3/solo-dogfeeding/code/200-plane/apps/web/core/components/exporter/prev-exports.tsx) |
| Issue 模型 (external_id) | [issue.py](file:///d:/fz/0508-3/solo-dogfeeding/code/200-plane/apps/api/plane/db/models/issue.py#L162-L163) |
| Label 模型 (external_id) | [label.py](file:///d:/fz/0508-3/solo-dogfeeding/code/200-plane/apps/api/plane/db/models/label.py#L23-L24) |
| State 模型 (external_id) | [state.py](file:///d:/fz/0508-3/solo-dogfeeding/code/200-plane/apps/api/plane/db/models/state.py#L92-L93) |
| Project 模型 (external_id) | [project.py](file:///d:/fz/0508-3/solo-dogfeeding/code/200-plane/apps/api/plane/db/models/project.py#L118-L120) |
| DraftIssue 模型 (external_id) | [draft.py](file:///d:/fz/0508-3/solo-dogfeeding/code/200-plane/apps/api/plane/db/models/draft.py#L68-L69) |
| IssueAttachment 模型 (external_id) | [issue.py](file:///d:/fz/0508-3/solo-dogfeeding/code/200-plane/apps/api/plane/db/models/issue.py#L402-L403) |
| IntakeIssue 模型 (external_id) | [intake.py](file:///d:/fz/0508-3/solo-dogfeeding/code/200-plane/apps/api/plane/db/models/intake.py#L72-L73) |
| Module 模型 (external_id) | [module.py](file:///d:/fz/0508-3/solo-dogfeeding/code/200-plane/apps/api/plane/db/models/module.py#L96-L97) |
| Cycle 模型 (external_id) | [cycle.py](file:///d:/fz/0508-3/solo-dogfeeding/code/200-plane/apps/api/plane/db/models/cycle.py#L72-L73) |
| Page 模型 (external_id) | [page.py](file:///d:/fz/0508-3/solo-dogfeeding/code/200-plane/apps/api/plane/db/models/page.py#L57-L58) |
| IssueType 模型 (external_id) | [issue_type.py](file:///d:/fz/0508-3/solo-dogfeeding/code/200-plane/apps/api/plane/db/models/issue_type.py#L23-L24) |
| FileAsset 模型 (external_id) | [asset.py](file:///d:/fz/0508-3/solo-dogfeeding/code/200-plane/apps/api/plane/db/models/asset.py#L61-L62) |

---

## 二、数据模型层 — external_source / external_id 机制

### 2.1 核心字段设计

Plane 在几乎所有可迁移实体上都保留了 `external_source` 和 `external_id` 两个 CharField：

```python
external_source = models.CharField(max_length=255, null=True, blank=True)
external_id     = models.CharField(max_length=255, blank=True, null=True)
```

**拥有此字段的模型清单：**

| 模型 | 说明 |
|------|------|
| `Issue` | 工作项 |
| `IssueVersion` | 工作项版本快照 |
| `IssueAttachment` | 工作项附件 |
| `DraftIssue` | 草稿工作项 |
| `Label` | 标签 |
| `State` | 状态 |
| `Project` | 项目 |
| `Module` | 模块 |
| `Cycle` | 周期 |
| `IntakeIssue` | 收件箱工作项 |
| `Page` | 页面 |
| `IssueType` | 工作项类型 |
| `FileAsset` | 文件资产 |

**设计意图：** `external_source` 标识来源系统（如 `"jira"`, `"github"`），`external_id` 存储源系统中的唯一标识。两者组合形成跨系统的幂等键。

### 2.2 Importer 模型

[importer.py](file:///d:/fz/0508-3/solo-dogfeeding/code/200-plane/apps/api/plane/db/models/importer.py) 中 `Importer` 继承自 `ProjectBaseModel`（关联 project + workspace），记录单次导入任务：

| 字段 | 说明 |
|------|------|
| `service` | `"github"` 或 `"jira"` |
| `status` | `queued → processing → completed / failed` |
| `initiated_by` | FK → User |
| `metadata` | JSON，存放连接凭据（如 Jira hostname/api_token） |
| `config` | JSON，导入配置（如 `epics_to_modules`） |
| `data` | JSON，预览数据（用户映射、计数等） |
| `token` | FK → APIToken，导入过程使用的 API Token |
| `imported_data` | JSON（nullable），导入结果摘要 |

### 2.3 ExporterHistory 模型

[exporter.py](file:///d:/fz/0508-3/solo-dogfeeding/code/200-plane/apps/api/plane/db/models/exporter.py) 记录导出任务：

| 字段 | 说明 |
|------|------|
| `provider` | `"json"` / `"csv"` / `"xlsx"` |
| `status` | `queued → processing → completed / failed` |
| `token` | 唯一 hex token，用于 S3 路径和查询 |
| `url` | 完成后的预签名下载 URL |
| `key` | S3 对象 key |
| `reason` | 失败原因 |

---

## 三、幂等与重入机制

### 3.1 API v1 — 基于 external_id 的 Upsert

这是**最核心的幂等写入路径**，位于 [IssueDetailAPIEndpoint.put](file:///d:/fz/0508-3/solo-dogfeeding/code/200-plane/apps/api/plane/api/views/issue.py#L588-L715)：

```
PUT /api/v1/workspaces/{slug}/projects/{project_id}/work-items/
Body: { external_id, external_source, name, ... }
```

**流程：**

1. 从请求体提取 `external_id` + `external_source`（**两者均为必填**，否则返回 400）
2. 尝试 `Issue.objects.get(project_id=..., external_id=..., external_source=...)`
3. **若存在** → 执行更新（partial update），返回 `200 OK`
4. **若不存在** (`DoesNotExist`) → 执行创建，返回 `201 Created`
5. 创建时额外支持覆盖 `created_at` 和 `created_by` 字段（保留源系统时间戳和作者）

**关键代码片段：**

```python
# issue.py L603-L657
if external_id and external_source:
    try:
        issue = Issue.objects.get(
            project_id=project_id,
            workspace__slug=slug,
            external_id=external_id,
            external_source=external_source,
        )
        # UPDATE path
        serializer = IssueSerializer(issue, data=request.data, partial=True, ...)
        ...
        return Response(serializer.data, status=status.HTTP_200_OK)
    except Issue.DoesNotExist:
        # CREATE path
        serializer = IssueSerializer(data=request.data, ...)
        ...
        issue.created_at = request.data.get("created_at", timezone.now())
        issue.created_by_id = request.data.get("created_by", request.user.id)
        return Response(serializer.data, status=status.HTTP_201_CREATED)
```

### 3.2 API v1 — POST 创建时的去重检查

[IssueListCreateAPIEndpoint.post](file:///d:/fz/0508-3/solo-dogfeeding/code/200-plane/apps/api/plane/api/views/issue.py#L423-L494)：

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

**语义：** POST 是"仅创建"，如果 external_id 已存在则返回 `409 Conflict` 并附带已有 issue ID，调用方可据此改用 PUT 或 PATCH。

### 3.3 API v1 — PATCH 更新时的 external_id 唯一性校验

[IssueDetailAPIEndpoint.patch](file:///d:/fz/0508-3/solo-dogfeeding/code/200-plane/apps/api/plane/api/views/issue.py#L739-L795)：

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

### 3.4 Label 的幂等保护

[LabelListCreateAPIEndpoint.post](file:///d:/fz/0508-3/solo-dogfeeding/code/200-plane/apps/api/plane/api/views/issue.py#L885-L935) 同样实现了：
- 创建时检查 `external_id` + `external_source` 去重 → 返回 `409`
- 数据库层面 `IntegrityError` → 按名称去重 → 返回 `409` 并附带已有 label ID

### 3.5 Issue 序列号分配的并发安全

[Issue.save](file:///d:/fz/0508-3/solo-dogfeeding/code/200-plane/apps/api/plane/db/models/issue.py#L180-L214) 在创建新 Issue 时使用 PostgreSQL `pg_advisory_xact_lock` 保证同一项目内 `sequence_id` 递增的原子性：

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

### 3.6 多对多关系的幂等写入

`IssueAssignee` 和 `IssueLabel` 的 `bulk_create` 均使用 `ignore_conflicts=True`：

```python
IssueAssignee.objects.bulk_create([...], batch_size=10, ignore_conflicts=True)
IssueLabel.objects.bulk_create([...], batch_size=10, ignore_conflicts=True)
```

同时这两张表都有 `UniqueConstraint(condition=Q(deleted_at__isnull=True))`，软删除记录不参与唯一约束，实现"删除后可重建"。

### 3.7 重入策略总结

| 场景 | 策略 | HTTP 状态码 |
|------|------|-------------|
| 首次创建 (POST) | 正常创建 | 201 |
| 重复创建 (POST + external_id 冲突) | 拒绝，返回已有 ID | 409 |
| Upsert (PUT + external_id) | 存在则更新，不存在则创建 | 200 / 201 |
| 更新时 external_id 变更冲突 | 拒绝 | 409 |
| Assignee/Label 关联重复 | ignore_conflicts 静默跳过 | - |
| Sequence ID 并发 | pg_advisory_xact_lock | - |

---

## 四、跨工作区资源映射策略

### 4.1 成员映射

Jira 和 GitHub 导入流程均采用"预览 + 用户决策"模式：

**前端类型定义：**

```typescript
// jira-importer.ts
interface User {
  username: string;
  import: "invite" | "map" | false;  // 三种策略
  email: string;
}

// github-importer.ts
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

### 4.2 标签映射

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

### 4.3 附件映射

`IssueAttachment`（实际使用 `FileAsset` 模型）同样有 `external_source` + `external_id`。附件导入需要：

1. 从源系统下载附件文件
2. 通过 FileAsset 上传接口上传至 Plane 的 S3 存储
3. 记录 `external_source` / `external_id` 便于后续增量同步时去重

### 4.4 State 映射

State 模型拥有 `external_source` / `external_id` 以及 `group` 字段（backlog/unstarted/started/completed/cancelled）。导入策略通常为：

- 将源系统状态映射到 Plane 的 5 个 `group`
- 同一 group 下可创建多个 State（如 Jira 的 "In Progress" → Plane State group="started", name="In Progress"）
- 通过 `external_id` 避免重复创建

### 4.5 Module / Cycle 映射

Module 和 Cycle 模型均有 `external_source` / `external_id`。Jira 导入时可配置 `epics_to_modules: true`，将 Jira Epic 映射为 Plane Module。

---

## 五、导入流程详解

### 5.1 内置导入器流程（GitHub / Jira）

```
┌─────────────┐    ┌──────────────────┐    ┌─────────────────────┐    ┌───────────────┐
│ 1. 连接源系统 │ →  │ 2. 预览源数据     │ →  │ 3. 用户配置映射策略   │ →  │ 4. 执行导入    │
│ (输入凭据)    │    │ (获取项目/仓库信息) │    │ (成员/标签/状态映射)  │    │ (后台异步)     │
└─────────────┘    └──────────────────┘    └─────────────────────┘    └───────────────┘
```

**Step 1 — 连接源系统：**

- Jira：调用 `GET /api/workspaces/{slug}/importers/jira` 传入 `cloud_hostname`, `api_token`, `email`, `project_key`
- GitHub：调用 `GET /api/workspaces/{slug}/importers/github/` 传入 `owner`, `repo`

**Step 2 — 预览源数据：**

返回源系统的 issues 计数、labels 计数、collaborators/users 列表等。前端类型 `IJiraResponse` / `IGithubRepoInfo`。

**Step 3 — 用户配置映射：**

前端展示预览数据，让用户为每个外部用户选择 `invite` / `map` / `false` 策略。

**Step 4 — 执行导入：**

- Jira：`POST /api/workspaces/{slug}/projects/importers/jira/`
- GitHub：`POST /api/workspaces/{slug}/projects/importers/github/`

请求体包含 `metadata`（连接凭据）、`config`（导入配置）、`data`（用户映射决策）、`project_id`（目标项目）。

导入任务在后台异步执行，`Importer.status` 从 `queued` → `processing` → `completed` / `failed`。

### 5.2 API v1 批量迁移流程（推荐用于自定义系统集成）

```
┌──────────────┐   ┌──────────────┐   ┌───────────────┐   ┌───────────────┐
│ 1. 创建项目   │ → │ 2. 创建基础   │ → │ 3. 批量 Upsert │ → │ 4. 创建关联    │
│ (带 external  │   │  资源          │   │ Issues         │   │ (评论/附件/链接)│
│  _id)         │   │ (States/Labels)│   │ (PUT 幂等)     │   │               │
└──────────────┘   └──────────────┘   └───────────────┘   └───────────────┘
```

**Step 1 — 创建目标项目：**

```http
POST /api/workspaces/{slug}/projects/
{
  "name": "Migrated Project",
  "identifier": "MIG",
  "external_source": "jira",
  "external_id": "PROJ-123"
}
```

**Step 2 — 创建基础资源（States / Labels）：**

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

**Step 3 — 批量 Upsert Issues：**

```http
PUT /api/v1/workspaces/{slug}/projects/{id}/work-items/
{
  "external_id": "JIRA-456",
  "external_source": "jira",
  "name": "Fix login bug",
  "state": "<uuid-of-mapped-state>",
  "labels": ["<uuid-of-mapped-label>"],
  "assignees": ["<uuid-of-mapped-user>"],
  "created_at": "2024-01-15T10:30:00Z",
  "created_by": "<uuid>",
  "description_html": "<p>...</p>"
}
```

**幂等保证：** 相同 `external_id` + `external_source` 的 PUT 请求可安全重复调用——存在则更新，不存在则创建。

**Step 4 — 创建关联资源：**

```http
POST /api/v1/workspaces/{slug}/projects/{id}/work-items/{issue_id}/comments/
POST /api/v1/workspaces/{slug}/projects/{id}/work-items/{issue_id}/links/
POST /api/v1/workspaces/{slug}/projects/{id}/work-items/{issue_id}/attachments/
```

### 5.3 通过 external_id 查询已有记录

API v1 列表端点支持 `external_id` + `external_source` 查询参数：

```http
GET /api/v1/workspaces/{slug}/projects/{id}/work-items/?external_id=JIRA-456&external_source=jira
```

[IssueListCreateAPIEndpoint.get](file:///d:/fz/0508-3/solo-dogfeeding/code/200-plane/apps/api/plane/api/views/issue.py#L305-L325) 中：

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

---

## 六、前端可见性策略

### 6.1 导入器列表

前端通过 `IntegrationService.getImporterServicesList()` 获取当前工作区的导入服务列表：

```typescript
// integration.service.ts
async getImporterServicesList(workspaceSlug: string): Promise<IImporterService[]> {
  return this.get(`/api/workspaces/${workspaceSlug}/importers/`)
}
```

使用 SWR 缓存 key `IMPORTER_SERVICES_LIST`，在集成设置页展示。

### 6.2 导出历史轮询

[prev-exports.tsx](file:///d:/fz/0508-3/solo-dogfeeding/code/200-plane/apps/web/core/components/exporter/prev-exports.tsx) 实现了自动轮询机制：

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

### 6.3 权限控制

- **集成/导入页面**：仅工作区 Admin 可见（`EUserPermissions.ADMIN` + `EUserPermissionsLevel.WORKSPACE`）
- **导出页面**：Admin + Member 可见（`EUserPermissions.ADMIN, EUserPermissions.MEMBER`）
- **API v1 端点**：通过 `ProjectEntityPermission` / `ProjectMemberPermission` 控制读写权限

### 6.4 导入状态流转

前端 `IImporterService` 类型中 `status` 为联合类型：

```typescript
status: "processing" | "completed" | "failed"
```

后端 `Importer` 模型额外有 `"queued"` 状态。前端展示策略：
- `processing` → 显示进度指示
- `completed` → 显示成功标记
- `failed` → 显示错误信息

### 6.5 SWR 缓存键

| 键 | 用途 |
|------|------|
| `IMPORTER_SERVICES_LIST` | 导入器列表 |
| `JIRA_IMPORTER_DETAIL` | Jira 项目预览信息 |
| `GITHUB_REPOSITORY_INFO` | GitHub 仓库信息 |
| `EXPORT_SERVICES_LIST` | 导出历史列表 |

---

## 七、导出流程（反向参考）

导出使用 Celery 异步任务，流程清晰，可作为导入的参照：

1. 用户发起 `POST /api/workspaces/{slug}/export-issues/`，指定 `provider`（csv/json/xlsx）和 `project` 列表
2. 创建 `ExporterHistory` 记录（status=queued, token=auto_generated_hex）
3. 触发 Celery 任务 `issue_export_task.delay(...)`
4. 任务内：status → processing → 使用 `DataExporter` + `IssueExportSerializer` 序列化 → 格式化 → 打包 ZIP → 上传 S3 → status → completed，写入 presigned URL
5. 失败则 status → failed，写入 reason

---

## 八、分批迁移建议

基于上述代码理解，分批迁移另一套系统的项目时建议：

### 8.1 推荐调用路径

使用 **API v1** 的 `PUT /work-items/` 端点，配合 `external_id` + `external_source` 实现幂等写入，而非内置的 GitHub/Jira 导入器。

### 8.2 分批策略

1. **按项目分批**：每批次迁移一个完整项目（Project → States → Labels → Issues → Comments/Attachments/Links）
2. **项目内按依赖层级**：先创建 States/Labels 等基础资源，再创建 Issues，最后创建关联
3. **Upsert 可重入**：同一批次可安全重试，PUT 端点自动判断 create/update

### 8.3 失败补偿

- 单条失败：API 返回 400/409，记录 external_id 便于重试
- 批量中断：后续重跑 PUT 请求，已成功记录会被更新而非重复创建
- 建议在调用方维护一个外部 ID → Plane UUID 的映射表，加速后续关联创建

### 8.4 注意事项

- `created_at` / `created_by` 字段需在创建后额外更新（PUT 端点的 create 路径会处理）
- `sequence_id` 分配使用 `pg_advisory_xact_lock`，大批量写入时需注意事务持有锁的时间
- Label 同名冲突会返回 409，需根据返回的已有 ID 进行映射
- 成员邀请是异步流程（邮件发送），`map` 策略要求用户已是工作区成员
