# Plane 工单附件上传与资源权限链路分析

## 1. 整体架构概览

Plane 的附件系统采用 **客户端直传 + 服务端签名 + 元数据关联** 的三段式架构：

```
前端选择文件 → 请求预签名 URL → 直传对象存储(S3/MinIO) → 确认上传状态 → 后端关联元数据
```

核心模型为 `FileAsset`，通过 `entity_type` 区分不同业务场景（工单附件、描述内嵌图片、评论图片等），通过外键关联到 `Issue`、`Comment`、`Page` 等实体。

附件相关的后端端点分布在三个层面：

| 层面 | URL 前缀 | 代码位置 | 服务对象 |
|------|----------|----------|----------|
| App 侧 | `/api/assets/v2/...` | `apps/api/plane/app/views/issue/attachment.py` + `apps/api/plane/app/views/asset/v2.py` | 前端 Web/Admin |
| API 侧（旧路径） | `/api/workspaces/.../issue-attachments/` | `apps/api/plane/api/views/issue.py` | 外部 API 消费者 |
| API 侧（新路径） | `/api/workspaces/.../work-items/.../attachments/` | 同上 | 外部 API 消费者 |

---

## 2. 前端上传流程

### 2.1 附件上传入口

工单附件的上传入口组件为 `apps/web/core/components/issues/attachment/attachment-upload.tsx` 中的 `IssueAttachmentUpload`，它使用 `react-dropzone` 实现拖拽与点击上传：

```tsx
const { getRootProps, getInputProps, isDragActive } = useDropzone({
  onDrop,
  maxSize: maxFileSize,   // 来自 useFileSize()，对应后端 FILE_SIZE_LIMIT
  multiple: false,
  disabled: isLoading || disabled,
});
```

`onDrop` 回调调用 `attachmentOperations.create(currentFile)`，该操作由 `apps/web/core/components/issues/issue-detail-widgets/attachments/helper.tsx` 中的 `useAttachmentOperations` 提供，最终调用 MobX Store 的 `createAttachment` 方法。

### 2.2 Store 层上传逻辑

`apps/web/core/store/issue/issue-details/attachment.store.ts` 中的 `IssueAttachmentStore.createAttachment` 执行以下步骤：

1. **创建临时上传状态**：生成 `tempId`（UUID v4），在 `attachmentsUploadStatusMap` 中记录 `{id, name, progress, size, type}`，前端据此展示上传进度条。
2. **调用 Service 层**：调用 `IssueAttachmentService.uploadIssueAttachment()`。
3. **更新 Store**：上传成功后，将返回的正式 attachment 写入 `attachmentMap` 和 `attachments` 索引，并更新 issue 的 `attachment_count`。
4. **清理临时状态**：无论成功或失败，均删除 `attachmentsUploadStatusMap` 中的临时条目。

### 2.3 Service 层上传三步曲

`apps/web/core/services/issue/issue_attachment.service.ts` 中的 `uploadIssueAttachment` 完成三步操作：

**步骤 1：获取预签名 URL**

```
POST /api/assets/v2/workspaces/{slug}/projects/{projectId}/{serviceType}/{issueId}/attachments/
Body: { name, size, type }   // type 为 MIME 类型，由 file-type 库从文件签名检测
```

前端通过 `packages/services/src/file/helper.ts` 中的 `getFileMetaDataForUpload` 提取文件元数据。该函数内部调用 `validateAndDetectFileType`，基于文件头签名（magic bytes）检测真实 MIME 类型，同时进行文件名校验。

> **重要澄清：前端文件名校验仅发出警告，不会阻止上传。** `validateAndDetectFileType` 在检测到不合规文件名时调用 `console.warn()` 输出警告信息，但**不抛出异常、不返回错误、不中断上传流程**。文件名校验的代码路径为：`validateFilename` → 返回错误消息字符串 → `console.warn()` → 继续执行。真正起阻止作用的是后端 `ATTACHMENT_MIME_TYPES` 白名单和 `sanitize_filename` 消毒处理。

**步骤 2：直传对象存储**

使用 `packages/services/src/file/file-upload.service.ts` 中的 `FileUploadService.uploadFile` 将文件以 `multipart/form-data` 形式 POST 到步骤 1 返回的预签名 URL。请求中 `withCredentials: false`，因为目标是 S3 而非 Plane 后端。

上传 payload 由 `packages/services/src/file/helper.ts` 中的 `generateFileUploadPayload` 构建，将预签名响应中的 `fields`（含签名策略、凭证等）与文件本体组装为 `FormData`。

**步骤 3：确认上传完成**

```
PATCH /api/assets/v2/workspaces/{slug}/projects/{projectId}/{serviceType}/{issueId}/attachments/{asset_id}/
```

通知后端文件已成功传输至对象存储，后端将 `is_uploaded` 标记为 `True`，并异步触发元数据提取和活动日志。

### 2.4 编辑器内嵌资源上传

描述和评论中的图片上传走不同的 Service 路径——`apps/web/core/services/file.service.ts` 中的 `FileService.uploadProjectAsset`（或 `uploadWorkspaceAsset`），entity_type 为 `ISSUE_DESCRIPTION` / `COMMENT_DESCRIPTION` / `PAGE_DESCRIPTION`。上传流程同样为三步：获取预签名 → 直传 → 确认。编辑器层使用 `packages/editor/src/core/hooks/use-file-upload.ts` 中的 `useUploader` hook 进行文件校验与上传编排。

---

## 3. 后端签名与存储层

### 3.1 S3Storage 类

`apps/api/plane/settings/storage.py` 中的 `S3Storage` 是核心存储抽象层，基于 `boto3` 实现：

- **初始化**：从环境变量读取 `AWS_ACCESS_KEY_ID`、`AWS_SECRET_ACCESS_KEY`、`AWS_S3_BUCKET_NAME`、`AWS_REGION`，支持 MinIO（`USE_MINIO=1`）和原生 S3 两种后端。
- **预签名上传**（`generate_presigned_post`）：生成 POST 策略，包含 bucket、content-length-range、Content-Type 等条件。
- **预签名下载**（`generate_presigned_url`）：生成 GET 签名 URL，支持 `Content-Disposition` 头（inline 或 attachment）。
- **签名过期时间**：通过 `SIGNED_URL_EXPIRATION` 环境变量控制，默认 **3600 秒（1 小时）**。

### 3.2 FileAsset 模型

`apps/api/plane/db/models/asset.py` 中的 `FileAsset` 是所有文件资源的统一模型，关键字段：

| 字段 | 说明 |
|------|------|
| `asset` | FileField，存储对象存储路径，格式为 `{workspace_id}/{uuid}-{sanitized_name}` |
| `entity_type` | 资源类型枚举：ISSUE_ATTACHMENT / ISSUE_DESCRIPTION / COMMENT_DESCRIPTION / PAGE_DESCRIPTION / USER_AVATAR / USER_COVER / WORKSPACE_LOGO / PROJECT_COVER 等 |
| `is_uploaded` | 是否已完成上传确认，默认 False |
| `is_deleted` | 软删除标记 |
| `attributes` | JSON 字段，存储 `{name, type, size}` |
| `storage_metadata` | JSON 字段，异步提取的对象存储元数据 |
| `workspace` / `project` / `issue` / `comment` / `page` | 外键关联到对应实体 |

`asset_url` 属性根据 `entity_type` 生成不同的 API 访问路径：

- **ISSUE_ATTACHMENT** → `/api/assets/v2/workspaces/{slug}/projects/{project_id}/issues/{issue_id}/attachments/{id}/`
- **ISSUE_DESCRIPTION / COMMENT_DESCRIPTION / PAGE_DESCRIPTION** → `/api/assets/v2/workspaces/{slug}/projects/{project_id}/{id}/`
- **WORKSPACE_LOGO / USER_AVATAR / USER_COVER / PROJECT_COVER** → `/api/assets/v2/static/{id}/`（允许匿名访问）

---

## 4. 元数据记录与实体关联

### 4.1 工单附件创建流程

`apps/api/plane/app/views/issue/attachment.py` 中 `IssueAttachmentV2Endpoint.post` 执行：

1. **文件名校验**：`sanitize_filename` 去除路径分隔符等危险字符。
2. **MIME 类型校验**：`type` 必须在 `settings.ATTACHMENT_MIME_TYPES` 白名单内（涵盖图片、文档、音视频、压缩包、3D 模型、字体等约 50+ 种类型）。
3. **文件大小限制**：`size_limit = min(size, settings.FILE_SIZE_LIMIT)`，默认 `FILE_SIZE_LIMIT = 5242880`（5 MB）。
4. **生成存储键**：`{workspace_id}/{uuid_hex}-{name}`。
5. **创建 FileAsset 记录**：`is_uploaded=False`，关联 `issue_id`、`project_id`、`workspace_id`，`entity_type=ISSUE_ATTACHMENT`。
6. **生成预签名 POST**：返回 `{upload_data, asset_id, attachment, asset_url}`。

### 4.2 上传确认流程

`IssueAttachmentV2Endpoint.patch` 执行：

1. **查找资产**：`FileAsset.objects.get(pk=pk, workspace__slug=slug, project_id=project_id)` — 注意查询**不包含** `issue_id` 过滤条件。
2. **权限守卫**：`@allow_permission([ROLE.ADMIN, ROLE.MEMBER, ROLE.GUEST])` 作用于 PROJECT 级别，校验用户是否为项目成员，**不校验 asset 是否属于当前 issue_id**。
3. **标记上传完成**：仅当 `is_uploaded` 为 False 时触发活动日志和状态更新，防止重复触发。
4. **异步元数据提取**：若 `storage_metadata` 为空，通过 Celery 任务 `get_asset_object_metadata.delay()` 异步获取 S3 对象元数据。

### 4.3 编辑器资源的批量关联

编辑器中的图片（ISSUE_DESCRIPTION / COMMENT_DESCRIPTION）在上传时可能尚未关联到具体实体（如新建 issue 时先上传图片）。`apps/api/plane/app/views/asset/v2.py` 中 `ProjectBulkAssetEndpoint.post` 在 issue/comment 创建后，批量将 asset 关联到对应实体：

- `ISSUE_DESCRIPTION` → `assets.update(issue_id=entity_id, project_id=project_id)`
- `COMMENT_DESCRIPTION` → `assets.update(comment_id=entity_id)`
- `PAGE_DESCRIPTION` → `assets.update(page_id=entity_id)`

---

## 5. 三组附件端点的完整行为对比

Plane 的附件操作涉及三组端点，分别服务于 Web 前端和外部 API 消费者。它们的权限机制和查询逻辑存在显著差异。

### 5.1 端点分组与 URL 路径

**App 侧 issue-attachments（V2）**

代码：`apps/api/plane/app/views/issue/attachment.py` → `IssueAttachmentV2Endpoint`

```
POST   /api/assets/v2/workspaces/{slug}/projects/{project_id}/issues/{issue_id}/attachments/
GET    /api/assets/v2/workspaces/{slug}/projects/{project_id}/issues/{issue_id}/attachments/          (列表)
GET    /api/assets/v2/workspaces/{slug}/projects/{project_id}/issues/{issue_id}/attachments/{pk}/      (下载)
PATCH  /api/assets/v2/workspaces/{slug}/projects/{project_id}/issues/{issue_id}/attachments/{pk}/      (确认上传)
DELETE /api/assets/v2/workspaces/{slug}/projects/{project_id}/issues/{issue_id}/attachments/{pk}/
```

**App 侧 issue-attachments（V1）**

代码：`apps/api/plane/app/views/issue/attachment.py` → `IssueAttachmentEndpoint`

路由注册于 `apps/api/plane/app/urls/issue.py`，URL 模式为 `workspaces/<str:slug>/projects/<uuid:project_id>/issues/<uuid:issue_id>/issue-attachments/` 及其 `/<uuid:pk>/` 子路径，当前仍处于活跃注册状态。

```
POST   /api/workspaces/{slug}/projects/{project_id}/issues/{issue_id}/issue-attachments/
GET    /api/workspaces/{slug}/projects/{project_id}/issues/{issue_id}/issue-attachments/
DELETE /api/workspaces/{slug}/projects/{project_id}/issues/{issue_id}/issue-attachments/{pk}/
```

**API 侧 work-items/attachments**

代码：`apps/api/plane/api/views/issue.py` → `IssueAttachmentListCreateAPIEndpoint` + `IssueAttachmentDetailAPIEndpoint`

旧路径：
```
POST   /api/workspaces/{slug}/projects/{project_id}/issues/{issue_id}/issue-attachments/
GET    /api/workspaces/{slug}/projects/{project_id}/issues/{issue_id}/issue-attachments/
GET    /api/workspaces/{slug}/projects/{project_id}/issues/{issue_id}/issue-attachments/{pk}/
PATCH  /api/workspaces/{slug}/projects/{project_id}/issues/{issue_id}/issue-attachments/{pk}/
DELETE /api/workspaces/{slug}/projects/{project_id}/issues/{issue_id}/issue-attachments/{pk}/
```

新路径：
```
POST   /api/workspaces/{slug}/projects/{project_id}/work-items/{issue_id}/attachments/
GET    /api/workspaces/{slug}/projects/{project_id}/work-items/{issue_id}/attachments/
GET    /api/workspaces/{slug}/projects/{project_id}/work-items/{issue_id}/attachments/{pk}/
PATCH  /api/workspaces/{slug}/projects/{project_id}/work-items/{issue_id}/attachments/{pk}/
DELETE /api/workspaces/{slug}/projects/{project_id}/work-items/{issue_id}/attachments/{pk}/
```

> 旧路径和新路径指向同一个 Endpoint 类，行为完全相同。

### 5.2 上传（POST）行为对比

| 维度 | App V2 (`IssueAttachmentV2Endpoint`) | API 侧 (`IssueAttachmentListCreateAPIEndpoint`) |
|------|--------------------------------------|------------------------------------------------|
| **权限机制** | `@allow_permission([ADMIN, MEMBER, GUEST])`，PROJECT 级别 | `user_has_issue_permission(issue=issue, allow_creator=True, allowed_roles=[ADMIN, MEMBER, GUEST])` |
| **issue 存在性校验** | ❌ 不校验 issue 是否存在 | ✅ 先 `Issue.objects.get(pk=issue_id, ...)` 校验 issue 存在 |
| **creator 规则** | ❌ 不允许非项目成员的 issue 创建者上传 | ✅ `allow_creator=True`：若用户是 issue 创建者，即使不是项目成员也可上传 |
| **MIME 校验** | ✅ `ATTACHMENT_MIME_TYPES` 白名单 | ✅ `ATTACHMENT_MIME_TYPES` 白名单 |
| **外部集成** | ❌ 不支持 | ✅ 支持 `external_id` + `external_source` 去重 |
| **文件名校验** | `sanitize_filename`，空名默认 `"unnamed"` | `sanitize_filename`，空名直接报 400 |

### 5.3 列表（GET 列表）行为对比

| 维度 | App V2 | API 侧 |
|------|--------|---------|
| **权限机制** | `@allow_permission([ADMIN, MEMBER, GUEST])`，PROJECT 级别 | `@allow_permission([ADMIN, MEMBER, GUEST])`，PROJECT 级别 |
| **查询条件** | `issue_id` + `entity_type=ISSUE_ATTACHMENT` + `workspace__slug` + `project_id` + `is_uploaded=True` | 同 App V2 |
| **issue_id 过滤** | ✅ 用于过滤结果 | ✅ 用于过滤结果 |

### 5.4 下载（GET 单个）行为对比

| 维度 | App V2 | API 侧 |
|------|--------|---------|
| **权限机制** | `@allow_permission([ADMIN, MEMBER, GUEST])`，PROJECT 级别 | `user_has_issue_permission(issue=None, allowed_roles=None, allow_creator=False)` |
| **API 侧权限语义** | — | `issue=None` → 不校验具体 issue；`allowed_roles=None` → 不限制角色；`allow_creator=False` → 不检查创建者。实质等价于：**只要是项目成员即可** |
| **查询条件** | `id=pk, workspace__slug=slug, project_id=project_id` | `id=pk, workspace__slug=slug, project_id=project_id` |
| **issue_id 过滤** | ❌ 不含 | ❌ 不含 |
| **is_uploaded 校验** | ✅ 未上传返回 400 | ✅ 未上传返回 400 |

### 5.5 上传确认（PATCH）行为对比

| 维度 | App V2 | API 侧 |
|------|--------|---------|
| **权限机制** | `@allow_permission([ADMIN, MEMBER, GUEST])`，PROJECT 级别 | `user_has_issue_permission(issue=issue, allow_creator=True, allowed_roles=[ADMIN, MEMBER, GUEST])` |
| **issue 存在性校验** | ❌ 不校验 | ✅ 先校验 issue 存在 |
| **查询条件** | `pk=pk, workspace__slug=slug, project_id=project_id` | `pk=pk, workspace__slug=slug, project_id=project_id` |
| **issue_id 过滤** | ❌ 不含 | ❌ 不含 |
| **creator 规则** | ❌ 不允许非项目成员的 issue 创建者确认 | ✅ `allow_creator=True`：issue 创建者可确认上传 |

### 5.6 删除（DELETE）行为对比

| 维度 | App V1 | App V2 | API 侧 |
|------|--------|--------|---------|
| **权限机制** | `@allow_permission([ADMIN], creator=True, model=FileAsset)` | `@allow_permission([ADMIN], creator=True, model=FileAsset)` | `user_has_issue_permission(issue=issue, allow_creator=True, allowed_roles=[ADMIN, MEMBER, GUEST])` |
| **查询条件** | `pk=pk, workspace__slug=slug, project_id=project_id, issue_id=issue_id` | `pk=pk, workspace__slug=slug, project_id=project_id` | `pk=pk, workspace__slug=slug, project_id=project_id` |
| **issue_id 过滤** | ✅ V1 含 issue_id | ❌ V2 不含 | ❌ 不含 |
| **删除方式** | 硬删除（`asset.delete()` + DB 删除） | 软删除（`is_deleted=True`） | 软删除（`is_deleted=True`） |
| **creator 规则** | ADMIN 或 `created_by=request.user` | ADMIN 或 `created_by=request.user` | 项目成员 (ADMIN/MEMBER/GUEST) 或 issue 创建者 |
| **角色范围** | 仅 ADMIN + 创建者 | 仅 ADMIN + 创建者 | **所有角色 (ADMIN/MEMBER/GUEST) + 创建者** — 角色范围更宽 |

### 5.7 关键差异总结

**1. API 侧独有的 `user_has_issue_permission` 函数**

`apps/api/plane/api/views/issue.py` 中的 `user_has_issue_permission` 定义为：

```python
def user_has_issue_permission(user_id, project_id, issue=None, allowed_roles=None, allow_creator=True):
    if allow_creator and issue is not None and user_id == issue.created_by_id:
        return True
    qs = ProjectMember.objects.filter(project_id=project_id, member_id=user_id, is_active=True)
    if allowed_roles is not None:
        qs = qs.filter(role__in=allowed_roles)
    return qs.exists()
```

该函数与 App 侧的 `@allow_permission` 装饰器有两个关键区别：
- **issue 创建者优先**：当 `allow_creator=True` 且 `issue is not None` 时，issue 创建者即使不是项目成员也被放行。这在 `@allow_permission` 中不存在——`allow_permission` 的 creator 规则要求**同时是工作区成员且是资源创建者**。
- **`issue=None` 的降级行为**：下载端点传 `issue=None`，此时 `allow_creator` 分支不生效，退化为纯项目成员校验。

**2. V1 端点是唯一在 DELETE 查询中包含 issue_id 的版本**

```python
# V1 (IssueAttachmentEndpoint.delete) — apps/api/plane/app/views/issue/attachment.py
issue_attachment = FileAsset.objects.filter(
    pk=pk, workspace__slug=slug, project_id=project_id, issue_id=issue_id
).first()

# V2 (IssueAttachmentV2Endpoint.delete) — 同文件
issue_attachment = FileAsset.objects.get(
    pk=pk, workspace__slug=slug, project_id=project_id
)
```

V1 的 DELETE 操作在数据库查询层面限制了 asset 必须属于指定 issue，这在所有版本中是唯一的。V1 同时也是唯一执行硬删除（`asset.delete()` + DB 记录删除）的版本，V2 和 API 侧均为软删除。V1 的路由当前仍处于活跃注册状态。

**3. API 侧 DELETE 的角色范围更宽**

App V2 的 DELETE 仅允许 ADMIN 或资源创建者，API 侧则允许所有项目角色（ADMIN/MEMBER/GUEST）加上 issue 创建者。这意味着 GUEST 用户在 API 侧可以删除附件，但在 App V2 侧不行。

### 5.8 API 侧 issue 创建者规则与 asset 查询不含 issue_id 的组合影响

API 侧的 PATCH（确认上传）和 DELETE 操作存在一个权限与查询不匹配的组合问题：

**代码执行流程**（以 DELETE 为例，PATCH 同理）：

```python
# 步骤 1：从 URL 中的 issue_id 获取 issue 对象
issue = Issue.objects.get(pk=issue_id, workspace__slug=slug, project_id=project_id)

# 步骤 2：权限校验 — 检查用户是否为该 issue 的创建者
if not user_has_issue_permission(
    request.user.id,
    project_id=project_id,
    issue=issue,                    # ← 校验的是 URL 中 issue 的创建者
    allowed_roles=[ROLE.ADMIN.value, ROLE.MEMBER.value, ROLE.GUEST.value],
    allow_creator=True,
):
    return Response({"error": "..."}, status=status.HTTP_403_FORBIDDEN)

# 步骤 3：查询 asset — 不含 issue_id
issue_attachment = FileAsset.objects.get(pk=pk, workspace__slug=slug, project_id=project_id)
#                                ↑ pk 来自请求，可以是项目中任意 asset
```

**组合效果**：步骤 2 校验的是 URL 中 issue 的创建者身份，步骤 3 查询的 asset 却不限定属于该 issue。两者解耦导致：

> **非项目成员的 issue 创建者，可以借自己创建的 issue URL 对同项目其他 issue 的附件执行确认上传（PATCH）或删除（DELETE）操作。**

**具体场景推演**：

假设项目 P 中存在 issue A（由用户 X 创建）和 issue B（由用户 Y 创建），issue B 有附件 asset_Z。

| 用户 | 项目成员身份 | 操作 | 结果 |
|------|-------------|------|------|
| X（issue A 创建者） | ❌ 不是项目成员 | `DELETE /api/.../work-items/{issue_A_id}/attachments/{asset_Z}/` | ✅ 成功 — `user_has_issue_permission(issue=issue_A)` 因 X 是 A 的创建者放行；`FileAsset.objects.get(pk=asset_Z, project_id=P)` 不含 issue_id 过滤，查到 issue B 的附件并删除 |
| X（issue A 创建者） | ❌ 不是项目成员 | `PATCH /api/.../work-items/{issue_A_id}/attachments/{asset_Z}/` | ✅ 成功 — 同上逻辑，X 可确认 issue B 的附件上传 |
| X（issue A 创建者） | ❌ 不是项目成员 | `DELETE /api/.../work-items/{issue_A_id}/attachments/{asset_Z}/`（App V2 侧） | ❌ 失败 — App V2 使用 `@allow_permission([ADMIN], creator=True, model=FileAsset)`，creator 语义为 FileAsset 的 `created_by` 而非 issue 的 `created_by`，X 不是 asset_Z 的创建者也不是项目成员 |

**根因**：API 侧的 `user_has_issue_permission` 以 URL 中的 issue 对象做权限判定，但 FileAsset 查询不以 issue_id 做范围限定，权限校验的对象与实际操作的对象不一致。

**对比 App 侧**：App V2 的 `@allow_permission` 使用 `creator=True, model=FileAsset`，creator 语义为 FileAsset 的 `created_by`，且要求同时是工作区成员。App V1 的 DELETE 查询包含 `issue_id`，即使权限通过，数据库层面也限制了 asset 必须属于指定 issue。因此 App 侧不存在此组合问题。

---

## 6. 通用资产端点的可变更边界

除了工单专属的 `IssueAttachmentV2Endpoint` 和 `IssueAttachmentDetailAPIEndpoint`，Plane 还提供了四个通用资产端点，它们的查询条件不区分 `entity_type`，对 `ISSUE_ATTACHMENT` 类型的资产同样可操作。这些端点构成了一条与 issue URL 平行的附件访问和变更路径。

### 6.1 `WorkspaceFileAssetEndpoint`

代码：`apps/api/plane/app/views/asset/v2.py`

**确认上传（PATCH）**

```python
@allow_permission([ROLE.ADMIN, ROLE.MEMBER, ROLE.GUEST], level="WORKSPACE")
def patch(self, request, slug, asset_id):
    asset = FileAsset.objects.get(id=asset_id, workspace__slug=slug)
    asset.is_uploaded = True
    # ...
    self.entity_asset_save(asset_id=asset_id, entity_type=asset.entity_type, asset=asset, request=request)
    asset.save(update_fields=["is_uploaded", "attributes"])
```

- 查询条件：`id=asset_id, workspace__slug=slug` — **不含 `entity_type`、`project_id`、`issue_id`**
- 权限级别：WORKSPACE，工作区成员即可
- 影响：工作区成员可确认上传同工作区内**任意项目**的 `ISSUE_ATTACHMENT` 资产，包括非成员项目

**删除（DELETE）**

```python
@allow_permission([ROLE.ADMIN, ROLE.MEMBER, ROLE.GUEST], level="WORKSPACE")
def delete(self, request, slug, asset_id):
    asset = FileAsset.objects.get(id=asset_id, workspace__slug=slug)
    asset.is_deleted = True
    asset.deleted_at = timezone.now()
    self.entity_asset_delete(entity_type=asset.entity_type, asset=asset, request=request)
    asset.save(update_fields=["is_deleted", "deleted_at"])
```

- 查询条件：`id=asset_id, workspace__slug=slug` — **不含 `entity_type`、`project_id`、`issue_id`**
- 权限级别：WORKSPACE
- `entity_asset_delete` 副作用：仅处理 `WORKSPACE_LOGO` 和 `PROJECT_COVER` 类型，对 `ISSUE_ATTACHMENT` 无额外副作用
- 影响：工作区成员可软删除同工作区内**任意项目**的 `ISSUE_ATTACHMENT` 资产，包括非成员项目

**下载（GET）**

```python
@allow_permission([ROLE.ADMIN, ROLE.MEMBER, ROLE.GUEST], level="WORKSPACE")
def get(self, request, slug, asset_id):
    asset = FileAsset.objects.get(id=asset_id, workspace__slug=slug)
    if not asset.is_uploaded:
        return Response({"error": "..."}, status=status.HTTP_404_NOT_FOUND)
    # 生成签名 URL 并 302 重定向
```

- 查询条件：`id=asset_id, workspace__slug=slug` — **不含 `entity_type`、`project_id`、`issue_id`**
- `is_uploaded` 校验：✅ 存在
- 影响：工作区成员可下载同工作区内**任意项目**的 `ISSUE_ATTACHMENT`，无需是项目成员

### 6.2 `ProjectAssetEndpoint`

代码：`apps/api/plane/app/views/asset/v2.py`

**确认上传（PATCH）**

```python
@allow_permission([ROLE.ADMIN, ROLE.MEMBER, ROLE.GUEST])
def patch(self, request, slug, project_id, pk):
    asset = FileAsset.objects.get(id=pk, workspace__slug=slug, project_id=project_id)
    asset.is_uploaded = True
    # ...
    asset.save(update_fields=["is_uploaded", "attributes"])
```

- 查询条件：`id=pk, workspace__slug=slug, project_id=project_id` — **不含 `entity_type`、`issue_id`**
- 权限级别：PROJECT，项目成员即可
- 影响：项目成员可确认上传同项目内**任意 issue** 的 `ISSUE_ATTACHMENT` 资产
- 注意：与 `WorkspaceFileAssetEndpoint` 不同，此端点的 PATCH **不调用** `entity_asset_save`，不会触发封面/Logo 更新副作用

**删除（DELETE）**

```python
@allow_permission([ROLE.ADMIN, ROLE.MEMBER, ROLE.GUEST])
def delete(self, request, slug, project_id, pk):
    asset = FileAsset.objects.get(id=pk, workspace__slug=slug, project_id=project_id)
    asset.is_deleted = True
    asset.deleted_at = timezone.now()
    asset.save(update_fields=["is_deleted", "deleted_at"])
```

- 查询条件：`id=pk, workspace__slug=slug, project_id=project_id` — **不含 `entity_type`、`issue_id`**
- 权限级别：PROJECT
- `entity_asset_delete` 副作用：**未调用**（与 WorkspaceFileAssetEndpoint 不同）
- 影响：项目成员可软删除同项目内**任意 issue** 的 `ISSUE_ATTACHMENT`，无需知道 issue_id

**下载（GET）**

```python
@allow_permission([ROLE.ADMIN, ROLE.MEMBER, ROLE.GUEST])
def get(self, request, slug, project_id, pk):
    asset = FileAsset.objects.get(workspace__slug=slug, project_id=project_id, pk=pk)
    if not asset.is_uploaded:
        return Response({"error": "..."}, status=status.HTTP_404_NOT_FOUND)
    # 生成签名 URL 并 302 重定向
```

- 查询条件：`workspace__slug=slug, project_id=project_id, pk=pk` — **不含 `entity_type`、`issue_id`**
- `is_uploaded` 校验：✅ 存在
- 影响：项目成员可下载同项目内**任意 issue** 的 `ISSUE_ATTACHMENT`

### 6.3 `AssetRestoreEndpoint`

代码：`apps/api/plane/app/views/asset/v2.py`

```python
class AssetRestoreEndpoint(BaseAPIView):
    @allow_permission([ROLE.ADMIN, ROLE.MEMBER, ROLE.GUEST], level="WORKSPACE")
    def post(self, request, slug, asset_id):
        asset = FileAsset.all_objects.get(id=asset_id, workspace__slug=slug)
        asset.is_deleted = False
        asset.deleted_at = None
        asset.save(update_fields=["is_deleted", "deleted_at"])
```

**恢复条件分析**：

| 维度 | 行为 |
|------|------|
| 查询管理器 | `FileAsset.all_objects` — **包含已软删除的记录**（默认 `objects` 管理器会过滤 `is_deleted=True` 的记录） |
| 查询条件 | `id=asset_id, workspace__slug=slug` — **不含 `entity_type`、`project_id`、`issue_id`** |
| 权限级别 | WORKSPACE |
| 角色范围 | ADMIN / MEMBER / GUEST |
| 恢复后状态 | `is_deleted=False, deleted_at=None`，资产重新可见 |

**对 `ISSUE_ATTACHMENT` 的影响**：

- 工作区成员可恢复同工作区内**任意项目**的已删除 `ISSUE_ATTACHMENT`，包括非成员项目
- 无需是原附件的创建者
- 无需知道 issue_id 或 project_id
- 恢复后附件重新出现在对应 issue 的列表中（`is_uploaded=True` 且 `is_deleted=False`）
- **不触发活动日志**：恢复操作不发送 `issue_activity`，不会在 issue 活动流中留下记录

### 6.4 `DuplicateAssetEndpoint`

代码：`apps/api/plane/app/views/asset/v2.py`

```python
class DuplicateAssetEndpoint(BaseAPIView):
    throttle_classes = [AssetRateThrottle]

    @allow_permission([ROLE.ADMIN, ROLE.MEMBER, ROLE.GUEST], level="WORKSPACE")
    def post(self, request, slug, asset_id):
        project_id = request.data.get("project_id", None)
        entity_id = request.data.get("entity_id", None)
        entity_type = request.data.get("entity_type", None)
        # ...
```

**来源资产校验规则**：

```python
user_workspace_ids = WorkspaceMember.objects.filter(
    member=request.user, is_active=True,
).values_list("workspace_id", flat=True)
original_asset = FileAsset.objects.filter(
    id=asset_id,
    is_uploaded=True,
    workspace_id__in=user_workspace_ids,
).first()
```

| 维度 | 行为 |
|------|------|
| 工作区范围 | 限定为用户所属的活跃工作区 |
| `entity_type` | ❌ 不限制，可复制任意类型包括 `ISSUE_ATTACHMENT` |
| `project_id` | ❌ 不限制，可跨项目复制 |
| `is_uploaded` | ✅ 必须已上传 |
| `is_deleted` | 默认管理器自动过滤软删除记录 |

**目标实体校验规则**：

```python
if project_id:
    if not Project.objects.filter(id=project_id, workspace=workspace).exists():
        return Response({"error": "Project not found"}, status=status.HTTP_404_NOT_FOUND)
```

| 维度 | 行为 |
|------|------|
| 目标工作区 | URL 中的 `slug`，由 `@allow_permission(level="WORKSPACE")` 校验用户成员身份 |
| 目标项目 | 若指定 `project_id`，校验该项目属于目标工作区 |
| `entity_type` | 由请求体 `entity_type` 字段指定，**不校验是否与来源资产一致** |
| `entity_id` | 由请求体 `entity_id` 字段指定，通过 `get_entity_id_field` 映射到 `issue_id`/`comment_id` 等，**不校验目标实体是否存在** |

**对 `ISSUE_ATTACHMENT` 的影响**：

- 工作区成员可将同工作区内任意项目的 `ISSUE_ATTACHMENT` 复制到目标项目/issue
- 来源资产的 `entity_type` 与目标的 `entity_type` 无需一致（例如可将 `ISSUE_ATTACHMENT` 复制为 `ISSUE_DESCRIPTION`，反之亦然）
- 复制后的资产 `is_uploaded=True`，可立即下载
- 存储层执行 `storage.copy_object()` 产生独立的对象存储副本

### 6.5 通用资产端点对附件权限的影响汇总

下表汇总所有通用资产端点的操作对 `ISSUE_ATTACHMENT` 的影响：

| 端点 | 操作 | 查询含 entity_type? | 查询含 project_id? | 查询含 issue_id? | 权限级别 | 对 ISSUE_ATTACHMENT 的影响 |
|------|------|--------------------|--------------------|------------------|----------|--------------------------|
| `WorkspaceFileAssetEndpoint` | PATCH 确认上传 | ❌ | ❌ | ❌ | WORKSPACE | 工作区成员可确认任意项目的附件上传 |
| `WorkspaceFileAssetEndpoint` | DELETE 删除 | ❌ | ❌ | ❌ | WORKSPACE | 工作区成员可删除任意项目的附件 |
| `WorkspaceFileAssetEndpoint` | GET 下载 | ❌ | ❌ | ❌ | WORKSPACE | 工作区成员可下载任意项目的附件 |
| `ProjectAssetEndpoint` | PATCH 确认上传 | ❌ | ✅ | ❌ | PROJECT | 项目成员可确认任意 issue 的附件上传 |
| `ProjectAssetEndpoint` | DELETE 删除 | ❌ | ✅ | ❌ | PROJECT | 项目成员可删除任意 issue 的附件 |
| `ProjectAssetEndpoint` | GET 下载 | ❌ | ✅ | ❌ | PROJECT | 项目成员可下载任意 issue 的附件 |
| `WorkspaceAssetDownloadEndpoint` | GET 下载 | ❌ | ❌ | ❌ | WORKSPACE | 工作区成员可下载任意项目的附件 |
| `ProjectAssetDownloadEndpoint` | GET 下载 | ❌ | ✅ | ❌ | PROJECT | 项目成员可下载任意 issue 的附件 |
| `AssetRestoreEndpoint` | POST 恢复 | ❌ | ❌ | ❌ | WORKSPACE | 工作区成员可恢复任意项目的已删除附件 |
| `DuplicateAssetEndpoint` | POST 复制 | ❌ | ❌ | ❌ | WORKSPACE | 工作区成员可复制任意项目的附件 |

**对比 `StaticFileAssetEndpoint`**（唯一做了 `entity_type` 限制的通用端点）：

```python
class StaticFileAssetEndpoint(BaseAPIView):
    permission_classes = [AllowAny]
    def get(self, request, asset_id):
        asset = FileAsset.objects.get(id=asset_id)
        if asset.entity_type not in [USER_AVATAR, USER_COVER, WORKSPACE_LOGO, PROJECT_COVER]:
            return Response({"error": "Invalid entity type."}, status=status.HTTP_400_BAD_REQUEST)
```

`StaticFileAssetEndpoint` 是通用资产端点中唯一在代码层面校验 `entity_type` 的，将可访问范围限制在公共展示类资产。其余端点均不做此限制，`ISSUE_ATTACHMENT` 可被任意操作。

### 6.6 通用端点与 issue URL 的权限差异

| 操作 | 通过 issue URL（`IssueAttachmentV2Endpoint`） | 通过通用端点（如 `WorkspaceFileAssetEndpoint`） |
|------|---------------------------------------------|---------------------------------------------|
| 下载附件 | 需项目成员身份 | 需工作区成员身份（可跨项目） |
| 确认上传 | 需项目成员身份 | 需工作区成员身份（可跨项目） |
| 删除附件 | 仅 ADMIN + 资源创建者 | 工作区任意成员（可跨项目） |
| 恢复已删除附件 | 无直接路径（`AssetRestoreEndpoint` 是唯一途径） | 需工作区成员身份 |
| 复制附件 | 无直接路径（`DuplicateAssetEndpoint` 是唯一途径） | 需工作区成员身份 |

**核心结论**：通用资产端点将 `ISSUE_ATTACHMENT` 的变更权限从**项目级**降级为**工作区级**。通过这些端点，非项目成员的工作区成员可对项目内的附件执行下载、确认上传、删除、恢复操作，且无需经过 issue URL 路径。

---

## 7. 下载与访问控制

### 7.1 权限装饰器 `allow_permission`

`apps/api/plane/app/permissions/base.py` 中的 `allow_permission` 是附件相关端点的核心权限守卫，支持两种级别：

**PROJECT 级别**（默认）：

1. 检查用户是否为项目成员且角色在 `allowed_roles` 中（ADMIN=20, MEMBER=15, GUEST=5）。
2. 额外检查：如果用户是项目成员（任意角色）且是工作区 ADMIN，也允许访问。

**WORKSPACE 级别**：

仅检查用户是否为工作区成员且角色满足要求。

**creator 选项**：若 `creator=True` 且指定了 `model`，则额外允许资源创建者访问（先校验工作区成员身份，再校验 `created_by=request.user`）。

### 7.2 `user_has_issue_permission` 函数

`apps/api/plane/api/views/issue.py` 中的 `user_has_issue_permission` 是 API 侧端点使用的权限函数，逻辑如下：

```python
def user_has_issue_permission(user_id, project_id, issue=None, allowed_roles=None, allow_creator=True):
    # 1. 如果允许创建者且 issue 不为空，检查用户是否为 issue 创建者
    if allow_creator and issue is not None and user_id == issue.created_by_id:
        return True
    # 2. 检查项目成员资格，可限制角色范围
    qs = ProjectMember.objects.filter(project_id=project_id, member_id=user_id, is_active=True)
    if allowed_roles is not None:
        qs = qs.filter(role__in=allowed_roles)
    return qs.exists()
```

与 `@allow_permission` 的关键区别：

| 维度 | `@allow_permission` | `user_has_issue_permission` |
|------|---------------------|----------------------------|
| creator 语义 | 资源（FileAsset）的 `created_by` | issue 的 `created_by` |
| creator 前提 | 必须同时是工作区成员 | 无需是工作区/项目成员 |
| 工作区 ADMIN 特权 | 项目成员 + 工作区 ADMIN 自动放行 | 无此逻辑 |
| issue 存在性 | 不校验 | 部分操作校验（POST/PATCH/DELETE） |

### 7.3 issue_id 在附件端点中的实际作用范围

> **核心发现：App V2 和 API 侧的单个资源操作（下载/删除/确认上传）均不基于 `issue_id` 进行权限限制。** 虽然 URL 中包含 `issue_id` 参数，但实际查询只使用 `(pk, workspace__slug, project_id)` 三元组。App V1 的 DELETE 是唯一使用 `(pk, workspace__slug, project_id, issue_id)` 四元组的操作。

各操作的查询条件总结：

**App V2 — `IssueAttachmentV2Endpoint`**

| 操作 | 查询条件 | 含 issue_id? |
|------|----------|-------------|
| POST | N/A（创建时写入） | — |
| GET 列表 | `issue_id + entity_type + workspace__slug + project_id + is_uploaded` | ✅ 用于过滤 |
| GET 单个 | `id + workspace__slug + project_id` | ❌ |
| PATCH | `pk + workspace__slug + project_id` | ❌ |
| DELETE | `pk + workspace__slug + project_id` | ❌ |

**App V1 — `IssueAttachmentEndpoint`**

| 操作 | 查询条件 | 含 issue_id? |
|------|----------|-------------|
| POST | N/A（创建时写入） | — |
| GET 列表 | `issue_id + workspace__slug + project_id` | ✅ 用于过滤 |
| DELETE | `pk + workspace__slug + project_id + issue_id` | ✅ **V1 DELETE 是所有端点中唯一在 asset 查询中含 issue_id 的操作** |

**API 侧 — `IssueAttachmentDetailAPIEndpoint`**

| 操作 | 查询条件 | 含 issue_id? |
|------|----------|-------------|
| POST | N/A（创建时写入） | — |
| GET 列表 | `issue_id + entity_type + workspace__slug + project_id + is_uploaded` | ✅ 用于过滤 |
| GET 单个 | `id + workspace__slug + project_id` | ❌ |
| PATCH | `pk + workspace__slug + project_id` | ❌ |
| DELETE | `pk + workspace__slug + project_id` | ❌ |

### 7.4 项目内跨工单 asset 访问与变更范围

由于单个资源操作不校验 `issue_id`，**项目级边界是附件访问与变更的最小隔离单元**。具体表现为：

| 场景 | 是否可行 | 说明 |
|------|----------|------|
| 同一工单内下载其他附件 | ✅ | 正常行为 |
| 同项目不同工单间，用 asset_id 下载附件 | ✅ | 查询不含 issue_id，仅校验项目成员 |
| 同项目不同工单间，用 asset_id 删除/确认上传附件（API 侧，issue 创建者） | ✅ | 5.8 节详述：issue 创建者可借自己 issue 的 URL 变更其他 issue 的附件 |
| 跨项目用 asset_id 下载附件 | ✅ 可通过 `WorkspaceAssetDownloadEndpoint` 绕过 | 6.3 节详述 |
| 跨项目用 asset_id 通过 issue URL 下载 | ❌ | `project_id` 为查询条件，且权限守卫校验项目成员身份 |
| 跨工作区用 asset_id 下载附件 | ❌ | `workspace__slug` 为查询条件，所有端点均校验工作区范围 |

**影响评估**：

- 同项目成员本就有权访问项目内所有工单，因此通过 issue URL 的跨工单**只读**访问不构成权限越级。
- 但 API 侧的跨工单**变更**（删除、确认上传）可被非项目成员的 issue 创建者利用，如 5.8 节所述，这在设计上是一个权限泄漏点。
- `WorkspaceAssetDownloadEndpoint` 允许工作区成员（无需是项目成员）下载项目内 `ISSUE_ATTACHMENT`，这也是一个潜在的权限泄漏点。

### 7.5 下载流程

当用户点击附件下载时：

1. **前端**：通过 `packages/utils/src/file.ts` 中的 `getFileURL` 将 `asset_url`（相对路径）拼接为完整 API URL，然后 `window.open(fileURL, "_blank")` 发起 GET 请求。
2. **后端**：对应端点（如 `IssueAttachmentV2Endpoint.get`）进行权限校验后，调用 `S3Storage.generate_presigned_url()` 生成临时签名 URL，并以 `HttpResponseRedirect`（302）重定向浏览器至该 URL。
3. **对象存储**：浏览器通过签名 URL 直接从 S3/MinIO 下载文件。

**关键点**：前端从未直接持有对象存储的永久访问路径，所有下载必须经由后端 API 端点进行权限校验后重定向。

---

## 8. URL 失效策略

### 8.1 签名 URL 过期

所有签名 URL 的有效期由 `SIGNED_URL_EXPIRATION` 环境变量控制，默认 **3600 秒（1 小时）**。过期后 URL 不再可访问，用户需要重新通过 API 端点获取新的签名 URL。

此策略同时适用于：
- **上传预签名 POST URL**（`generate_presigned_post`）
- **下载预签名 GET URL**（`generate_presigned_url`）

### 8.2 软删除与恢复

- **删除**：附件删除为软删除操作（`is_deleted=True, deleted_at=now()`），不立即移除对象存储中的文件。V1 端点（`IssueAttachmentEndpoint.delete`）会物理删除存储对象（`asset.delete(save=False)`）并删除数据库记录，V2 端点（`IssueAttachmentV2Endpoint.delete`）和 API 侧端点仅标记软删除。
- **恢复**：`apps/api/plane/app/views/asset/v2.py` 中的 `AssetRestoreEndpoint` 可将软删除的资产恢复（`is_deleted=False, deleted_at=None`），需要 WORKSPACE 级别权限。
- **硬删除**：由 `HARD_DELETE_AFTER_DAYS`（默认 60 天）环境变量控制，通过定时任务清理软删除超过指定天数的资产。

### 8.3 前端 asset_url 机制

`asset_url` 是一个相对 API 路径（如 `/api/assets/v2/workspaces/.../attachments/{id}/`），本身永不过期。每次前端访问此路径时，后端都会生成新的签名 URL 进行重定向。因此，**前端的下载链接始终有效**（只要用户有权限且资产未被删除），签名 URL 的过期不影响业务层面的可用性。

---

## 9. 跨工作区与跨项目引用边界

### 9.1 资产的工作区隔离

`FileAsset` 的存储键格式为 `{workspace_id}/{uuid}-{name}`，从存储层面实现了工作区隔离。所有查询均通过 `workspace__slug=slug` 条件限定工作区范围。

### 9.2 项目级隔离（非工单级）

如第 7.4 节所述，附件访问与变更的最小隔离单元是**项目**而非工单。URL 中虽然包含 `issue_id`，但 App V2 和 API 侧的单个资源操作（下载、确认上传、删除）的数据库查询均不使用 `issue_id` 作为过滤条件。App V1 的 DELETE 是唯一的例外，其查询包含 `issue_id`。权限隔离主要依赖 `workspace__slug + project_id` 双重约束和权限守卫的成员校验。

### 9.3 DuplicateAssetEndpoint 的跨工作区边界

`apps/api/plane/app/views/asset/v2.py` 中的 `DuplicateAssetEndpoint` 用于跨实体复制资产（如 issue 复制时连带复制附件），其安全边界如下：

1. **源资产查找**：限制为用户所属工作区内的资产。
   ```python
   user_workspace_ids = WorkspaceMember.objects.filter(
       member=request.user, is_active=True,
   ).values_list("workspace_id", flat=True)
   original_asset = FileAsset.objects.filter(
       id=asset_id, is_uploaded=True, workspace_id__in=user_workspace_ids,
   ).first()
   ```
2. **目标工作区校验**：目标 workspace 由 URL 中的 `slug` 参数决定，`allow_permission` 装饰器确保用户是目标工作区成员。
3. **项目校验**：若指定了 `project_id`，校验该项目存在于目标工作区中。
4. **存储层复制**：调用 `storage.copy_object()` 在对象存储中复制文件，生成新的 `FileAsset` 记录（新 workspace、新 project、新 entity 关联），共享 `storage_metadata`。

---

## 10. 前端预览组件的权限继承

### 10.1 附件列表与预览

前端附件展示组件分为两种视图：

1. **详情面板视图**：`apps/web/core/components/issues/attachment/attachment-detail.tsx` 中的 `IssueAttachmentsDetail` — 卡片式展示，点击通过 `<Link href={fileURL}>` 在新标签页打开下载。
2. **列表项视图**：`apps/web/core/components/issues/attachment/attachment-list-item.tsx` 中的 `IssueAttachmentsListItem` — 紧凑列表，点击通过 `window.open(fileURL, "_blank")` 打开。

两者都通过 `getFileURL(attachment.asset_url)` 构造下载 URL，本质上是向后端 API 端点发起 GET 请求。

### 10.2 权限继承逻辑

前端附件组件本身 **不进行独立的权限判断**，权限完全继承自其父级上下文：

1. **页面级权限**：用户能访问 issue 详情页，意味着已通过了工作区/项目级别的成员校验。
2. **组件级禁用**：通过 `disabled` prop 控制是否允许上传/删除操作，该 prop 由上层组件根据用户角色（如 GUEST 角色可能禁用上传）传入。
3. **下载权限**：点击下载时，浏览器直接请求后端 API（携带认证 Cookie），后端通过 `allow_permission` 再次校验权限。若权限不足，返回 403 而非签名 URL。

### 10.3 Space（公开门户）的附件访问

`apps/api/plane/space/views/asset.py` 中的 `EntityAssetEndpoint` 用于 Plane Space 公开工单板：

- **GET 请求**：`AllowAny` 权限，无需认证。通过 `anchor`（部署标识）找到对应的 DeployBoard，再查找其工作区下的 FileAsset。查询条件限定 `entity_type__in=[ISSUE_DESCRIPTION, COMMENT_DESCRIPTION]`，即**仅允许访问描述和评论中的内嵌图片**。
- **POST/PATCH/DELETE**：`IsAuthenticated` 权限，且必须属于已发布的 DeployBoard。

**注意**：Space 的 GET 端点通过 `entity_type` 白名单排除了 `ISSUE_ATTACHMENT` 类型，意味着工单的显式附件**不会**在公开门户中暴露，仅描述和评论中的内嵌图片可被匿名用户访问。

### 10.4 StaticFileAssetEndpoint 的特殊处理

`apps/api/plane/app/views/asset/v2.py` 中的 `StaticFileAssetEndpoint` 使用 `AllowAny` 权限，仅允许访问以下类型：

- `USER_AVATAR`
- `USER_COVER`
- `WORKSPACE_LOGO`
- `PROJECT_COVER`

这些资源属于公共展示类资产，不涉及敏感信息，因此无需认证即可访问。

---

## 11. 前端文件名校验行为详析

`packages/services/src/file/helper.ts` 中的文件名校验流程：

```
validateFilename(filename)  →  返回 string | null
                                  ↓
                    validateAndDetectFileType(file)
                                  ↓
                    if (filenameError) {
                      console.warn(...)   // ← 仅警告，不中断
                    }
                                  ↓
                    继续执行 detectMimeTypeFromSignature
                                  ↓
                    返回检测到的 MIME 类型或空字符串
                                  ↓
                    getFileMetaDataForUpload 继续构建 {name, size, type}
```

**校验规则与行为对照**：

| 校验规则 | 校验结果 | 行为 | 是否阻止上传 |
|----------|----------|------|-------------|
| 文件名为空 | `"Filename cannot be empty"` | `console.warn` | ❌ 不阻止（后端会拒绝空名） |
| 隐藏文件（以 `.` 开头） | `"Hidden files (starting with dot) are not allowed"` | `console.warn` | ❌ 不阻止 |
| 文件名含路径分隔符 `/` 或 `\` | `"Filename cannot contain path separators"` | `console.warn` | ❌ 不阻止 |
| 双扩展名含危险中间扩展名 | `"File has suspicious double extension"` | `console.warn` | ❌ 不阻止 |
| 最终扩展名为危险类型 | `"File extension 'xxx' is not allowed"` | `console.warn` | ❌ 不阻止 |

**结论**：前端文件名校验全部为 **仅警告（console.warn），不阻止上传**。真正阻止恶意文件上传的是后端的双重防线：
1. `ATTACHMENT_MIME_TYPES` 白名单校验 MIME 类型
2. `sanitize_filename` 对文件名进行消毒处理
3. S3 预签名 POST 中的 `Content-Type` 条件约束

---

## 12. 安全措施总结

| 安全维度 | 实现机制 |
|----------|----------|
| **文件类型校验** | 前端：file-type 库从文件签名检测 MIME（**仅辅助检测，不阻止**）；后端：`ATTACHMENT_MIME_TYPES` 白名单（**强制阻止**） |
| **危险扩展名** | 前端：`DANGEROUS_EXTENSIONS` 列表（exe/bat/sh/php 等）校验（**仅 console.warn，不阻止**）；后端：`sanitize_filename` 消毒 + MIME 白名单双重防护 |
| **文件大小限制** | `FILE_SIZE_LIMIT` 环境变量，默认 5 MB，前后端双重校验 |
| **文件名消毒** | 后端 `sanitize_filename` 去除路径分隔符等危险字符（**强制**） |
| **签名上传** | 预签名 POST 包含 bucket/key/Content-Type/content-length-range 条件约束 |
| **签名下载** | 所有下载通过后端重定向至临时签名 URL，签名默认 1 小时过期 |
| **权限分层** | PROJECT 级（工单附件）和 WORKSPACE 级（工作区资产）两级权限控制 |
| **权限粒度** | 附件访问与变更隔离粒度为**项目级**，非工单级。同项目内用户可通过 asset_id 访问任意工单的附件 |
| **跨工单变更** | API 侧 `user_has_issue_permission(issue=issue, allow_creator=True)` 与 `FileAsset` 查询不含 `issue_id` 组合，导致非项目成员的 issue 创建者可变更同项目其他 issue 的附件（5.8 节） |
| **通用端点权限降级** | `WorkspaceFileAssetEndpoint`、`ProjectAssetEndpoint`、`AssetRestoreEndpoint`、`DuplicateAssetEndpoint` 均不按 `entity_type` 过滤，将 `ISSUE_ATTACHMENT` 变更权限从项目级降级为工作区级（6.5 节） |
| **Download 端点绕过** | `WorkspaceAssetDownloadEndpoint` 和 `ProjectAssetDownloadEndpoint` 不按 `entity_type` 过滤，工作区/项目成员可绕过 issue URL 下载 `ISSUE_ATTACHMENT` 资产 |
| **角色控制** | ADMIN/MEMBER/GUEST 三级角色，App V2 删除操作仅 ADMIN 或创建者；API 侧删除允许所有角色 + issue 创建者 |
| **工作区隔离** | 存储键包含 workspace_id，查询均限定工作区范围 |
| **软删除** | 附件删除为软删除，可恢复，硬删除由定时任务在 60 天后执行 |
| **限流** | `AssetRateThrottle` 针对资产操作进行频率限制 |
| **公共资产白名单** | `StaticFileAssetEndpoint` 仅对头像/Logo/封面等公共类型允许匿名访问（有 `entity_type` 过滤） |
| **Space 门户隔离** | 公开门户仅暴露 `ISSUE_DESCRIPTION` 和 `COMMENT_DESCRIPTION`，不暴露 `ISSUE_ATTACHMENT` |
