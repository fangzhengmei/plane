# Plane 工作区与项目导入 / 迁移代码理解

## 一、整体架构概览

Plane 的导入/迁移体系在代码层面涉及两套 API，但两者的实际可达性存在差异：

| 层级 | 用途 | 后端可达性 | 前端可达性 |
|------|------|------------|------------|
| **API v1 (Public API)** | 通用外部系统集成、批量写入 | 路由已挂载，但 **PUT Upsert 端点未暴露** | N/A（API 层） |
| **App API (内置导入器)** | GitHub / Jira 原生集成向导 | **后端 View/Service 未实现** | **Service 已定义但零调用，无 UI 页面** |

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
| 工作区设置常量（侧边栏菜单定义） | `packages/constants/src/settings/workspace.ts` |
| 工作区设置侧边栏组件 | `apps/web/core/components/settings/workspace/sidebar/item-categories.tsx` |
| Integrations 页面 | `apps/web/app/(all)/[workspaceSlug]/(settings)/settings/(workspace)/integrations/page.tsx` |
| SingleIntegrationCard 组件 | `apps/web/core/components/integration/single-integration-card.tsx` |
| 项目级 IntegrationCard 组件 | `apps/web/core/components/project/integration-card.tsx` |
| 定价/计划对比（含 Importers 描述） | `apps/web/core/constants/plans.tsx` |
| 路由注册入口 | `apps/web/app/routes.ts` |
| 核心路由定义（含 workspace settings 路由列表） | `apps/web/app/routes/core.ts` |
| 扩展路由定义（当前为空） | `apps/web/app/routes/extended.ts` |
| 路由合并辅助函数 | `apps/web/app/routes/helper.ts` |
| React Router 配置 | `apps/web/react-router.config.ts` |
| 工作区设置 Layout（含权限校验） | `apps/web/app/(all)/[workspaceSlug]/(settings)/settings/(workspace)/layout.tsx` |
| 设置页面辅助函数（pathnameToAccessKey） | `apps/web/core/components/settings/helper.ts` |
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

Jira 和 GitHub 导入的成员映射策略定义在前端 TypeScript 类型中，但**无 UI 组件实现**，仅作为接口契约存在：

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

**注意：** 这些映射策略仅存在于类型定义中。前端无导入向导 UI，后端无导入器 View，因此 `invite`/`map`/`false` 的实际执行逻辑在开源仓库中完全缺失。

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

`IssueAttachment` 模型（继承自 `ProjectBaseModel`）和 `FileAsset` 模型均有 `external_source` + `external_id` 字段。附件的迁移链路比评论和链接更复杂，涉及文件二进制上传：

**API v1 附件创建端点**（可达）：

```
POST /api/v1/workspaces/{slug}/projects/{project_id}/work-items/{issue_id}/attachments/
```

这是一个**两阶段上传**流程：

1. **阶段一：创建 FileAsset 记录 + 获取预签名 URL**
   - 请求体：`{ name, type, size, external_id?, external_source? }`
   - 服务端校验 MIME 类型（`ATTACHMENT_MIME_TYPES` 白名单）和文件大小（`FILE_SIZE_LIMIT`）
   - 如果提供 `external_id` + `external_source`，先查询 `FileAsset` 去重：
     - 匹配条件：`project_id` + `workspace__slug` + `external_source` + `external_id` + `issue_id` + `entity_type=ISSUE_ATTACHMENT`
     - 已存在 → 返回 `409 Conflict` + 已有 asset ID
     - 不存在 → 创建 `FileAsset` 记录（含 `external_id`/`external_source`）
   - 生成 S3 预签名 POST URL
   - 返回 `{ upload_data: { url, fields }, asset_id, attachment, asset_url }`
   - **注意：此阶段返回 200（非 201），因为文件尚未上传**

2. **阶段二：客户端直传 S3**
   - 使用预签名 URL 将文件二进制直接上传至 S3
   - 上传完成后 S3 不会通知 Plane 后端

3. **阶段三：确认上传（由其他机制触发）**
   - `FileAsset.is_uploaded` 字段默认 `False`
   - 列表端点 GET 只返回 `is_uploaded=True` 的附件
   - 需要额外的确认机制将 `is_uploaded` 设为 `True`

**迁移注意事项：**
- 附件创建不支持覆盖 `created_at` 和 `created_by`，始终使用当前时间和当前 API Key 用户
- `external_id` 去重范围是 project + issue 级别，同文件挂到不同 issue 不冲突
- 需先从源系统下载文件到本地，再通过预签名 URL 上传至 S3

### 5.4 评论映射

`IssueComment` 模型拥有 `external_source` + `external_id` 字段，且 API v1 端点对迁移场景提供了**最完整的支持**：

**API v1 评论创建端点**（可达）：

```
POST /api/v1/workspaces/{slug}/projects/{project_id}/work-items/{issue_id}/comments/
```

**external_id 去重机制：**

```python
# issue.py L1422-L1444
if request.data.get("external_id") and request.data.get("external_source"):
    if IssueComment.objects.filter(
        project_id=project_id,
        workspace__slug=slug,
        external_source=request.data.get("external_source"),
        external_id=request.data.get("external_id"),
    ).exists():
        return Response(
            {"error": "...", "id": str(issue_comment.id)},
            status=status.HTTP_409_CONFLICT,
        )
```

- 匹配范围：`project_id` + `workspace` + `external_source` + `external_id`（**不限定 issue_id**，即同一 external_id 在不同 issue 下也会冲突）
- 冲突时返回 `409` + 已有 comment ID

**源时间和作者信息保留：**

评论是三种关联资源中**唯一支持覆盖 `created_at`、`created_by` 和 `actor` 的**：

```python
# issue.py L1449-L1454
issue_comment = IssueComment.objects.get(pk=serializer.instance.id)
issue_comment.created_at = request.data.get("created_at", timezone.now())
issue_comment.created_by_id = request.data.get("created_by", request.user.id)
issue_comment.actor_id = request.data.get("created_by", request.user.id)
issue_comment.save(update_fields=["created_at", "created_by"])
```

- `created_at`：可传入源系统的创建时间，默认当前时间
- `created_by` / `actor_id`：可传入源系统用户映射后的 Plane UUID，默认当前 API Key 用户
- 注意：`actor` 字段在 serializer 的 `read_only_fields` 中，但 View 在 `serializer.save()` 之后手动覆盖了 `actor_id`，绕过了 serializer 的只读限制

**PATCH 更新时的 external_id 唯一性校验：**

```python
# issue.py L1575-L1591 (IssueCommentDetailAPIEndpoint.patch)
if (
    request.data.get("external_id")
    and (issue_comment.external_id != str(request.data.get("external_id")))
    and IssueComment.objects.filter(...).exists()
):
    return Response({"error": "...", "id": str(issue_comment.id)}, status=status.HTTP_409_CONFLICT)
```

仅在 `external_id` 值发生变更时校验唯一性，同值更新不触发冲突检查。

**请求体字段（`IssueCommentCreateSerializer`）：**

| 字段 | 可写 | 说明 |
|------|------|------|
| `comment_html` | ✅ | HTML 格式评论内容 |
| `comment_json` | ✅ | JSON 格式评论内容（富文本结构化） |
| `access` | ✅ | `"INTERNAL"` 或 `"EXTERNAL"` |
| `external_source` | ✅ | 来源系统标识 |
| `external_id` | ✅ | 源系统唯一标识 |
| `created_at` | ⚠️ | serializer 只读，但 View 在 save 后手动覆盖 |
| `created_by` | ⚠️ | serializer 只读，但 View 在 save 后手动覆盖 |

### 5.5 链接映射

`IssueLink` 模型**没有** `external_source` 和 `external_id` 字段，是三种关联资源中**唯一不具备外部 ID 追踪能力**的。

**API v1 链接创建端点**（可达）：

```
POST /api/v1/workspaces/{slug}/projects/{project_id}/work-items/{issue_id}/links/
```

**去重机制 — 基于 URL + Issue 唯一约束：**

```python
# serializers/issue.py L428-L431 (IssueLinkCreateSerializer.create)
def create(self, validated_data):
    if IssueLink.objects.filter(url=validated_data.get("url"), issue_id=validated_data.get("issue_id")).exists():
        raise serializers.ValidationError({"error": "URL already exists for this Issue"})
    return IssueLink.objects.create(**validated_data)
```

- 匹配条件：`url` + `issue_id` 组合唯一
- 冲突时抛出 `ValidationError`，返回 `400 Bad Request`（**不是 409**）
- 同一 URL 可以挂到不同 issue 下

**源时间和作者信息保留：**

链接创建**不支持覆盖 `created_at`**，但**支持覆盖 `created_by`**：

```python
# issue.py L1167-L1168
link.created_by_id = request.data.get("created_by", request.user.id)
link.save(update_fields=["created_by"])
```

- `created_by`：可传入源系统用户映射后的 Plane UUID
- `created_at`：始终使用当前时间，无法覆盖
- `created_by` 在 serializer 的 `read_only_fields` 中，View 在 `save()` 后手动覆盖（与 Comment 模式相同）

**请求体字段（`IssueLinkCreateSerializer`）：**

| 字段 | 可写 | 说明 |
|------|------|------|
| `title` | ✅ | 链接标题 |
| `url` | ✅ | 链接 URL（需 http/https 协议） |
| `issue_id` | ✅ | 所属 issue（serializer 字段，但实际由 URL 路径参数覆盖） |
| `external_source` | ❌ | 模型无此字段 |
| `external_id` | ❌ | 模型无此字段 |
| `created_at` | ❌ | 无法覆盖 |
| `created_by` | ⚠️ | serializer 只读，但 View 在 save 后手动覆盖 |

### 5.6 State 映射

State 模型拥有 `external_source` / `external_id` 以及 `group` 字段（backlog/unstarted/started/completed/cancelled）。导入策略通常为：

- 将源系统状态映射到 Plane 的 5 个 `group`
- 同一 group 下可创建多个 State（如 Jira 的 "In Progress" → Plane State group="started", name="In Progress"）
- 通过 `external_id` 避免重复创建

### 5.7 Module / Cycle 映射

Module 和 Cycle 模型均有 `external_source` / `external_id`。Jira 导入时可配置 `epics_to_modules: true`，将 Jira Epic 映射为 Plane Module。

### 5.8 关联资源迁移边界对比

| 维度 | IssueComment | FileAsset / IssueAttachment | IssueLink |
|------|-------------|---------------------------|-----------|
| **模型 external_source** | ✅ 有 | ✅ 有（FileAsset + IssueAttachment 双模型均有） | ❌ 无 |
| **模型 external_id** | ✅ 有 | ✅ 有（FileAsset + IssueAttachment 双模型均有） | ❌ 无 |
| **Serializer 接受 external_id** | ✅ 可写字段 | ✅ `IssueAttachmentUploadSerializer` 可写 | ❌ 模型无此字段 |
| **POST 创建去重** | ✅ 409 + 已有 ID | ✅ 409 + 已有 ID | ⚠️ 400（URL+issue 唯一） |
| **去重键** | project + workspace + external_source + external_id | project + workspace + external_source + external_id + issue_id + entity_type | url + issue_id |
| **PATCH 更新 external_id 校验** | ✅ 变更时校验 409 | ❌ 无 PATCH 外部 ID 校验逻辑 | ❌ 无 external_id |
| **覆盖 created_at** | ✅ View 手动覆盖 | ❌ 无法覆盖 | ❌ 无法覆盖 |
| **覆盖 created_by** | ✅ View 手动覆盖 | ❌ 始终 request.user | ⚠️ View 手动覆盖 |
| **覆盖 actor** | ✅ View 手动覆盖 | N/A | N/A |
| **幂等重入** | ✅ POST + external_id 409 → PATCH by pk | ⚠️ POST + external_id 409 → 无法 PATCH 更新内容 | ❌ 仅 URL 去重，无外部 ID |
| **失败补偿** | 409 时拿 ID → PATCH 更新内容 | 409 时拿 ID → 但文件已存在无需重传 | 400 时需自行判断是 URL 重复还是其他错误 |
| **上传复杂度** | 简单 JSON 提交 | 两阶段：预签名 URL → 直传 S3 → 确认 is_uploaded | 简单 JSON 提交 |

**关键差异总结：**

1. **IssueComment 是迁移友好度最高的关联资源**：有 external_id 去重、支持覆盖 created_at/created_by/actor、409 冲突返回已有 ID 可用于 PATCH 更新。

2. **FileAsset/Attachment 迁移需关注两阶段上传的完整性**：预签名 URL 机制意味着迁移工具需自行处理文件下载→S3 上传→确认上传完成的三步流程。`is_uploaded=False` 的附件不会出现在列表中，静默丢失。此外，附件创建不支持覆盖 `created_at` 和 `created_by`，无法保留源系统的时间和作者。

3. **IssueLink 是迁移最薄弱的关联资源**：无 external_id/external_source 字段，无法通过外部标识去重或追踪来源。去重仅基于 URL + issue_id 组合，返回 400 而非 409（无法区分"URL 重复"和"其他验证错误"），不支持覆盖 created_at。迁移时需在调用方自行维护"源 URL → Plane link ID"映射表。

---

## 六、导入流程详解

### 6.1 内置导入器流程（GitHub / Jira）— 前端 Service/Type 存在但 UI 页面缺失

```
┌─────────────┐    ┌──────────────────┐    ┌─────────────────────┐    ┌───────────────┐
│ 1. 连接源系统 │ →  │ 2. 预览源数据     │ →  │ 3. 用户配置映射策略   │ →  │ 4. 执行导入    │
│ (输入凭据)    │    │ (获取项目/仓库信息) │    │ (成员/标签/状态映射)  │    │ (后台异步)     │
└─────────────┘    └──────────────────┘    └─────────────────────┘    └───────────────┘
   ⚠️ Service定义     ⚠️ Service定义         ❌ 无UI组件                ❌ 后端未实现
   无UI组件           无UI组件               无调用入口
   无调用入口         无调用入口
```

**已有代码层：**

| 层级 | 存在 | 缺失 |
|------|------|------|
| TypeScript 类型 | `IJiraMetadata`, `IJiraResponse`, `IJiraImporterForm`, `IGithubRepoInfo`, `IGithubServiceImportFormData`, `IImporterService` | — |
| Service 方法 | `JiraImporterService.getJiraProjectInfo`, `createJiraImporter`; `GithubIntegrationService.getGithubRepoInfo`, `createGithubServiceImport` | — |
| SWR 缓存键 | `JIRA_IMPORTER_DETAIL`, `GITHUB_REPOSITORY_INFO`, `IMPORTER_SERVICES_LIST` | — |
| UI 页面/组件 | — | 无 Jira/GitHub 导入向导页面，无导入配置表单，无进度展示组件 |
| 调用链 | Service 类已定义 | **无任何页面或组件调用这些 Service 方法** |
| 后端 API | — | 无 View、URL、Celery 任务 |

**各步骤分析：**

**Step 1 — 连接源系统：** `JiraImporterService.getJiraProjectInfo` 和 `GithubIntegrationService.getGithubRepoInfo` 已定义，但无 UI 组件调用它们。用户无法在界面上输入 Jira 凭据或 GitHub 仓库名。

**Step 2 — 预览源数据：** `IJiraResponse` / `IGithubRepoInfo` 类型定义了返回结构（issues 计数、labels、users 列表等），但无 UI 组件展示预览数据。

**Step 3 — 用户配置映射：** `jira-importer.ts` 和 `github-importer.ts` 类型中定义了 `User.import: "invite" | "map" | false`，但无 UI 组件让用户做映射决策。

**Step 4 — 执行导入：** `createJiraImporter` 和 `createGithubServiceImport` 方法已定义，但后端无对应 View。即使前端调用也会收到 404。

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

## 七、前端可见性审计

### 7.1 工作区设置侧边栏 — 无 Importer 入口

工作区设置侧边栏由 `WORKSPACE_SETTINGS` 常量（`packages/constants/src/settings/workspace.ts`）驱动，包含以下菜单项：

| key | href | 分类 |
|-----|------|------|
| `general` | `/settings` | Administration |
| `members` | `/settings/members` | Administration |
| `billing-and-plans` | `/settings/billing` | Administration |
| `export` | `/settings/exports` | Administration |
| `webhooks` | `/settings/webhooks` | Developer |

**关键发现：侧边栏中无 `import` / `importer` / `integrations` 菜单项。** 用户在标准导航中无法找到导入入口。

### 7.2 Integrations 页面 — 文件存在但路由未注册，URL 不可达

尽管文件系统中存在 `apps/web/app/(all)/[workspaceSlug]/(settings)/settings/(workspace)/integrations/page.tsx`，该页面在当前构建中 **不可通过 URL 访问**。原因链如下：

#### 路由注册链

Plane 前端使用 React Router v7（`@react-router/dev/routes`），路由通过代码显式注册，**不支持文件系统自动发现**：

```
routes.ts
  ├─ import { coreRoutes } from "./routes/core"
  ├─ import { extendedRoutes } from "./routes/extended"
  ├─ mergeRoutes(coreRoutes, extendedRoutes)
  └─ [...mergedRoutes, route("*", "./not-found.tsx")]  ← 404 兜底
```

1. **`routes/core.ts`**（L258-L285）— 工作区设置区域注册了 6 条路由：

```typescript
layout("./(all)/[workspaceSlug]/(settings)/settings/(workspace)/layout.tsx", [
  route(":workspaceSlug/settings", ".../page.tsx"),           // general
  route(":workspaceSlug/settings/members", ".../page.tsx"),   // members
  route(":workspaceSlug/settings/billing", ".../page.tsx"),   // billing
  route(":workspaceSlug/settings/exports", ".../page.tsx"),   // exports
  route(":workspaceSlug/settings/webhooks", ".../page.tsx"),  // webhooks
  route(":workspaceSlug/settings/webhooks/:webhookId", "..."), // webhook detail
])
```

**无 `:workspaceSlug/settings/integrations` 路由。**

2. **`routes/extended.ts`** — 扩展路由数组为空：

```typescript
export const extendedRoutes: RouteConfigEntry[] = [];
```

3. **`react-router.config.ts`** — 未启用文件系统路由发现：

```typescript
export default { appDirectory: "app", ssr: false } satisfies Config;
```

#### 访问结果推演

| 访问方式 | 结果 | 原因 |
|----------|------|------|
| 侧边栏点击 | ❌ 不可达 | `WORKSPACE_SETTINGS` 不含 integrations 项 |
| 直接输入 URL `/{slug}/settings/integrations/` | ❌ 404 页面 | 路由未注册，命中 `route("*", "./not-found.tsx")` 兜底 |
| Power K 快捷命令 | ❌ 不可达 | 工作区设置命令仅列举 `WORKSPACE_SETTINGS` 中的页面 |
| 代码内部链接 | ❌ 不可达 | 全仓库 `grep "settings/integrations"` 零匹配 |

#### 页面内容分析（假设可访问）

即使该页面可达，其内容也与 importer 无关：

- 调用 `IntegrationService.getAppIntegrationsList()` → `GET /api/integrations/` 获取 GitHub/Slack OAuth 应用列表
- 使用 `SingleIntegrationCard` 展示安装/卸载按钮
- `SingleIntegrationCard` 通过 `useIntegrationPopup` 触发 OAuth 授权流程
- **这是 OAuth 集成（同步），不是数据导入（importer）**

#### 工作区设置布局的权限校验

`settings/(workspace)/layout.tsx` 使用 `WORKSPACE_SETTINGS_ACCESS` 进行权限校验：

```typescript
const { accessKey } = pathnameToAccessKey(pathname);
isAuthorized = WORKSPACE_SETTINGS_ACCESS[accessKey]?.includes(userWorkspaceRole);
```

`WORKSPACE_SETTINGS_ACCESS` 由 `WORKSPACE_SETTINGS` 自动生成，不含 `/settings/integrations` 键。即使绕过路由注册直接访问，权限校验也会因 `undefined` 而拒绝渲染。

### 7.3 导入器 Service 方法 — 已定义但从未被调用

| Service 方法 | 定义位置 | 调用情况 |
|-------------|----------|----------|
| `JiraImporterService.getJiraProjectInfo` | `jira.service.ts` | ❌ 全仓库零调用 |
| `JiraImporterService.createJiraImporter` | `jira.service.ts` | ❌ 全仓库零调用 |
| `GithubIntegrationService.getGithubRepoInfo` | `github.service.ts` | ❌ 全仓库零调用 |
| `GithubIntegrationService.createGithubServiceImport` | `github.service.ts` | ❌ 全仓库零调用 |
| `IntegrationService.getImporterServicesList` | `integration.service.ts` | ❌ 全仓库零调用 |
| `IntegrationService.deleteImporterService` | `integration.service.ts` | ❌ 全仓库零调用 |

SWR 缓存键 `IMPORTER_SERVICES_LIST`、`JIRA_IMPORTER_DETAIL`、`GITHUB_REPOSITORY_INFO` 同样已定义但从未被任何 `useSWR` 调用引用。

### 7.4 导入进度展示 — 无实现

开源前端中不存在任何导入进度展示组件。对比导出功能的完整实现：

| 功能 | 导出 (Export) | 导入 (Import) |
|------|--------------|--------------|
| 触发表单 | `ExportForm` 组件 | ❌ 无 |
| 历史列表 | `PrevExports` 组件 + Table | ❌ 无 |
| 进度轮询 | 3 秒轮询 `processing` 状态 | ❌ 无 |
| 侧边栏入口 | ✅ `export` 在 `WORKSPACE_SETTINGS` | ❌ 无 |
| 后端 API | `ExportIssuesEndpoint` (GET/POST) | ❌ 无 View |

### 7.5 定价页面中的 Importers 描述

`plans.tsx` 中有一个 `id: "importers"` 的定价对比组，描述 Jira 和 GitHub 导入在所有付费计划（含 free）中可用（"Without custom props" 或 "With custom props"）。这说明导入功能在 Plane 产品规划中属于标准功能，但开源仓库中尚未落地。

### 7.6 权限控制

- **Integrations 页面**（如手动访问 URL）：仅工作区 Admin 可见（`EUserPermissions.ADMIN` + `EUserPermissionsLevel.WORKSPACE`）
- **Exports 页面**：Admin + Member 可见
- **API v1 端点**：需 API Key 认证（`APIKeyAuthentication`），通过 `ProjectEntityPermission` 控制读写

### 7.7 前端可见性总结

| 维度 | 状态 | 说明 |
|------|------|------|
| 路由注册 | ❌ | `routes/core.ts` 不含 `settings/integrations`；`routes/extended.ts` 为空数组 |
| 导航入口 | ❌ | 侧边栏无 Importer/Import/Integrations 菜单项 |
| URL 直接访问 | ❌ | 未注册路由命中 404 兜底；即使绕过，`WORKSPACE_SETTINGS_ACCESS` 也会拒绝 |
| 导入向导组件 | ❌ | 无 Jira/GitHub 导入配置/预览/映射 UI |
| 导入进度展示 | ❌ | 无进度条、状态轮询、历史记录组件 |
| Service 层 | ⚠️ | 类和方法已定义，但零调用 |
| TypeScript 类型 | ✅ | 完整定义了导入相关接口 |
| SWR 缓存键 | ⚠️ | 已定义，但零引用 |
| 后端 API | ❌ | 无 View/URL/Task |

**结论：开源前端不具备可操作的 importers 入口和导入进度展示功能。** `integrations/page.tsx` 是一个死文件（路由未注册、无任何链接指向），且其内容是 OAuth 集成而非数据导入。导入器相关的 Service/Type 仅保留了"接口骨架"，推测为 Plane 云端版本预留的契约，自部署环境无法使用。

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
