# Plane 工单附件上传与资源权限链路分析

## 1. 整体架构概览

Plane 的附件系统采用 **客户端直传 + 服务端签名 + 元数据关联** 的三段式架构：

```
前端选择文件 → 请求预签名 URL → 直传对象存储(S3/MinIO) → 确认上传状态 → 后端关联元数据
```

核心模型为 `FileAsset`，通过 `entity_type` 区分不同业务场景（工单附件、描述内嵌图片、评论图片等），通过外键关联到 `Issue`、`Comment`、`Page` 等实体。

---

## 2. 前端上传流程

### 2.1 附件上传入口

工单附件的上传入口组件为 [IssueAttachmentUpload](file:///d:/fz/0508-3/solo-dogfeeding/code/194-plane/apps/web/core/components/issues/attachment/attachment-upload.tsx)，它使用 `react-dropzone` 实现拖拽与点击上传：

```tsx
const { getRootProps, getInputProps, isDragActive } = useDropzone({
  onDrop,
  maxSize: maxFileSize,   // 来自 useFileSize()，对应后端 FILE_SIZE_LIMIT
  multiple: false,
  disabled: isLoading || disabled,
});
```

`onDrop` 回调调用 `attachmentOperations.create(currentFile)`，该操作由 [useAttachmentOperations](file:///d:/fz/0508-3/solo-dogfeeding/code/194-plane/apps/web/core/components/issues/issue-detail-widgets/attachments/helper.tsx#L30-L85) 提供，最终调用 MobX Store 的 `createAttachment` 方法。

### 2.2 Store 层上传逻辑

[IssueAttachmentStore.createAttachment](file:///d:/fz/0508-3/solo-dogfeeding/code/194-plane/apps/web/core/store/issue/issue-details/attachment.store.ts#L142-L185) 执行以下步骤：

1. **创建临时上传状态**：生成 `tempId`（UUID v4），在 `attachmentsUploadStatusMap` 中记录 `{id, name, progress, size, type}`，前端据此展示上传进度条。
2. **调用 Service 层**：调用 `IssueAttachmentService.uploadIssueAttachment()`。
3. **更新 Store**：上传成功后，将返回的正式 attachment 写入 `attachmentMap` 和 `attachments` 索引，并更新 issue 的 `attachment_count`。
4. **清理临时状态**：无论成功或失败，均删除 `attachmentsUploadStatusMap` 中的临时条目。

### 2.3 Service 层上传三步曲

[IssueAttachmentService.uploadIssueAttachment](file:///d:/fz/0508-3/solo-dogfeeding/code/194-plane/apps/web/core/services/issue/issue_attachment.service.ts#L43-L69) 完成三步操作：

**步骤 1：获取预签名 URL**

```
POST /api/assets/v2/workspaces/{slug}/projects/{projectId}/{serviceType}/{issueId}/attachments/
Body: { name, size, type }   // type 为 MIME 类型，由 file-type 库从文件签名检测
```

前端通过 [getFileMetaDataForUpload](file:///d:/fz/0508-3/solo-dogfeeding/code/194-plane/packages/services/src/file/helper.ts#L115-L122) 提取文件元数据。该函数内部调用 `validateAndDetectFileType`，基于文件头签名（magic bytes）检测真实 MIME 类型，同时进行文件名校验（禁止双扩展名、隐藏文件、危险扩展名等）。

**步骤 2：直传对象存储**

使用 [FileUploadService.uploadFile](file:///d:/fz/0508-3/solo-dogfeeding/code/194-plane/packages/services/src/file/file-upload.service.ts#L30-L47) 将文件以 `multipart/form-data` 形式 POST 到步骤 1 返回的预签名 URL。请求中 `withCredentials: false`，因为目标是 S3 而非 Plane 后端。

上传 payload 由 [generateFileUploadPayload](file:///d:/fz/0508-3/solo-dogfeeding/code/194-plane/packages/services/src/file/helper.ts#L58-L63) 构建，将预签名响应中的 `fields`（含签名策略、凭证等）与文件本体组装为 `FormData`。

**步骤 3：确认上传完成**

```
PATCH /api/assets/v2/workspaces/{slug}/projects/{projectId}/{serviceType}/{issueId}/attachments/{asset_id}/
```

通知后端文件已成功传输至对象存储，后端将 `is_uploaded` 标记为 `True`，并异步触发元数据提取和活动日志。

### 2.4 编辑器内嵌资源上传

描述和评论中的图片上传走不同的 Service 路径——[FileService.uploadProjectAsset](file:///d:/fz/0508-3/solo-dogfeeding/code/194-plane/apps/web/core/services/file.service.ts#L148-L174)（或 `uploadWorkspaceAsset`），entity_type 为 `ISSUE_DESCRIPTION` / `COMMENT_DESCRIPTION` / `PAGE_DESCRIPTION`。上传流程同样为三步：获取预签名 → 直传 → 确认。编辑器层使用 [useUploader](file:///d:/fz/0508-3/solo-dogfeeding/code/194-plane/packages/editor/src/core/hooks/use-file-upload.ts#L28-L96) hook 进行文件校验与上传编排。

---

## 3. 后端签名与存储层

### 3.1 S3Storage 类

[S3Storage](file:///d:/fz/0508-3/solo-dogfeeding/code/194-plane/apps/api/plane/settings/storage.py#L19-L205) 是核心存储抽象层，基于 `boto3` 实现：

- **初始化**：从环境变量读取 `AWS_ACCESS_KEY_ID`、`AWS_SECRET_ACCESS_KEY`、`AWS_S3_BUCKET_NAME`、`AWS_REGION`，支持 MinIO（`USE_MINIO=1`）和原生 S3 两种后端。
- **预签名上传**（`generate_presigned_post`）：生成 POST 策略，包含 bucket、content-length-range、Content-Type 等条件。
- **预签名下载**（`generate_presigned_url`）：生成 GET 签名 URL，支持 `Content-Disposition` 头（inline 或 attachment）。
- **签名过期时间**：通过 `SIGNED_URL_EXPIRATION` 环境变量控制，默认 **3600 秒（1 小时）**。

### 3.2 FileAsset 模型

[FileAsset](file:///d:/fz/0508-3/solo-dogfeeding/code/194-plane/apps/api/plane/db/models/asset.py#L31-L103) 是所有文件资源的统一模型，关键字段：

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

[IssueAttachmentV2Endpoint.post](file:///d:/fz/0508-3/solo-dogfeeding/code/194-plane/apps/api/plane/app/views/issue/attachment.py#L99-L147) 执行：

1. **文件名校验**：`sanitize_filename` 去除路径分隔符等危险字符。
2. **MIME 类型校验**：`type` 必须在 `settings.ATTACHMENT_MIME_TYPES` 白名单内（涵盖图片、文档、音视频、压缩包、3D 模型、字体等约 50+ 种类型）。
3. **文件大小限制**：`size_limit = min(size, settings.FILE_SIZE_LIMIT)`，默认 `FILE_SIZE_LIMIT = 5242880`（5 MB）。
4. **生成存储键**：`{workspace_id}/{uuid_hex}-{name}`。
5. **创建 FileAsset 记录**：`is_uploaded=False`，关联 `issue_id`、`project_id`、`workspace_id`，`entity_type=ISSUE_ATTACHMENT`。
6. **生成预签名 POST**：返回 `{upload_data, asset_id, attachment, asset_url}`。

### 4.2 上传确认流程

[IssueAttachmentV2Endpoint.patch](file:///d:/fz/0508-3/solo-dogfeeding/code/194-plane/apps/api/plane/app/views/issue/attachment.py#L203-L230) 执行：

1. **权限校验**：仅当 `is_uploaded` 为 False 时触发活动日志（防止重复触发）。
2. **标记上传完成**：`is_uploaded = True`，`created_by = request.user`。
3. **异步元数据提取**：若 `storage_metadata` 为空，通过 Celery 任务 `get_asset_object_metadata.delay()` 异步获取 S3 对象元数据。
4. **活动日志**：发送 `attachment.activity.created` 事件，包含序列化后的附件数据。

### 4.3 编辑器资源的批量关联

编辑器中的图片（ISSUE_DESCRIPTION / COMMENT_DESCRIPTION）在上传时可能尚未关联到具体实体（如新建 issue 时先上传图片）。[ProjectBulkAssetEndpoint.post](file:///d:/fz/0508-3/solo-dogfeeding/code/194-plane/apps/api/plane/app/views/asset/v2.py#L635-L693) 在 issue/comment 创建后，批量将 asset 关联到对应实体：

- `ISSUE_DESCRIPTION` → `assets.update(issue_id=entity_id, project_id=project_id)`
- `COMMENT_DESCRIPTION` → `assets.update(comment_id=entity_id)`
- `PAGE_DESCRIPTION` → `assets.update(page_id=entity_id)`

---

## 5. 下载访问控制

### 5.1 权限装饰器 `allow_permission`

[allow_permission](file:///d:/fz/0508-3/solo-dogfeeding/code/194-plane/apps/api/plane/app/permissions/base.py#L19-L87) 是附件相关端点的核心权限守卫，支持两种级别：

**PROJECT 级别**（默认）：

1. 检查用户是否为项目成员且角色在 `allowed_roles` 中（ADMIN=20, MEMBER=15, GUEST=5）。
2. 额外检查：如果用户是项目成员（任意角色）且是工作区 ADMIN，也允许访问。

**WORKSPACE 级别**：

仅检查用户是否为工作区成员且角色满足要求。

**creator 选项**：若 `creator=True` 且指定了 `model`，则额外允许资源创建者访问（先校验工作区成员身份，再校验 `created_by=request.user`）。

### 5.2 各端点的权限矩阵

| 端点 | HTTP 方法 | 权限级别 | 允许角色 | 特殊规则 |
|------|-----------|----------|----------|----------|
| `IssueAttachmentV2Endpoint.post` | POST | PROJECT | ADMIN, MEMBER, GUEST | — |
| `IssueAttachmentV2Endpoint.get` (列表) | GET | PROJECT | ADMIN, MEMBER, GUEST | — |
| `IssueAttachmentV2Endpoint.get` (单个) | GET | PROJECT | ADMIN, MEMBER, GUEST | 返回预签名下载 URL（302 重定向） |
| `IssueAttachmentV2Endpoint.patch` (确认上传) | PATCH | PROJECT | ADMIN, MEMBER, GUEST | API 层额外校验 `user_has_issue_permission` |
| `IssueAttachmentV2Endpoint.delete` | DELETE | PROJECT | ADMIN | `creator=True`，仅 ADMIN 或创建者可删除 |
| `WorkspaceFileAssetEndpoint.get` | GET | WORKSPACE | ADMIN, MEMBER, GUEST | — |
| `WorkspaceFileAssetEndpoint.post` | POST | WORKSPACE | ADMIN, MEMBER, GUEST | — |
| `WorkspaceAssetDownloadEndpoint.get` | GET | WORKSPACE | ADMIN, MEMBER, GUEST | `is_uploaded=True` 校验 |
| `ProjectAssetDownloadEndpoint.get` | GET | PROJECT | ADMIN, MEMBER, GUEST | `is_uploaded=True` + project_id 匹配 |
| `StaticFileAssetEndpoint.get` | GET | **AllowAny** | 无需认证 | 仅限 LOGO/AVATAR/COVER 类型 |
| `AssetRestoreEndpoint.post` | POST | WORKSPACE | ADMIN, MEMBER, GUEST | — |
| `DuplicateAssetEndpoint.post` | POST | WORKSPACE | ADMIN, MEMBER, GUEST | 源资产查找限制为用户所属工作区 |

### 5.3 下载流程

当用户点击附件下载时：

1. **前端**：通过 [getFileURL](file:///d:/fz/0508-3/solo-dogfeeding/code/194-plane/packages/utils/src/file.ts#L15-L20) 将 `asset_url`（相对路径）拼接为完整 API URL，然后 `window.open(fileURL, "_blank")` 发起 GET 请求。
2. **后端**：对应端点（如 `IssueAttachmentV2Endpoint.get`）进行权限校验后，调用 `S3Storage.generate_presigned_url()` 生成临时签名 URL，并以 `HttpResponseRedirect`（302）重定向浏览器至该 URL。
3. **对象存储**：浏览器通过签名 URL 直接从 S3/MinIO 下载文件。

**关键点**：前端从未直接持有对象存储的永久访问路径，所有下载必须经由后端 API 端点进行权限校验后重定向。

---

## 6. URL 失效策略

### 6.1 签名 URL 过期

所有签名 URL 的有效期由 `SIGNED_URL_EXPIRATION` 环境变量控制，默认 **3600 秒（1 小时）**。过期后 URL 不再可访问，用户需要重新通过 API 端点获取新的签名 URL。

此策略同时适用于：
- **上传预签名 POST URL**（`generate_presigned_post`）
- **下载预签名 GET URL**（`generate_presigned_url`）

### 6.2 软删除与恢复

- **删除**：附件删除为软删除操作（`is_deleted=True, deleted_at=now()`），不立即移除对象存储中的文件。V1 端点会物理删除存储对象（`asset.delete(save=False)`），V2 端点仅标记软删除。
- **恢复**：[AssetRestoreEndpoint](file:///d:/fz/0508-3/solo-dogfeeding/code/194-plane/apps/api/plane/app/views/asset/v2.py#L473-L482) 可将软删除的资产恢复（`is_deleted=False, deleted_at=None`），需要 WORKSPACE 级别权限。
- **硬删除**：由 `HARD_DELETE_AFTER_DAYS`（默认 60 天）环境变量控制，通过定时任务清理软删除超过指定天数的资产。

### 6.3 前端 asset_url 机制

`asset_url` 是一个相对 API 路径（如 `/api/assets/v2/workspaces/.../attachments/{id}/`），本身永不过期。每次前端访问此路径时，后端都会生成新的签名 URL 进行重定向。因此，**前端的下载链接始终有效**（只要用户有权限且资产未被删除），签名 URL 的过期不影响业务层面的可用性。

---

## 7. 跨工作区引用边界

### 7.1 资产的工作区隔离

`FileAsset` 的存储键格式为 `{workspace_id}/{uuid}-{name}`，从存储层面实现了工作区隔离。所有查询均通过 `workspace__slug=slug` 条件限定工作区范围。

### 7.2 DuplicateAssetEndpoint 的跨工作区边界

[DuplicateAssetEndpoint](file:///d:/fz/0508-3/solo-dogfeeding/code/194-plane/apps/api/plane/app/views/asset/v2.py#L705-L795) 用于跨实体复制资产（如 issue 复制时连带复制附件），其安全边界如下：

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

### 7.3 工单附件不可跨工作区引用

`IssueAttachmentV2Endpoint` 的所有操作均在 `workspace__slug + project_id + issue_id` 三重限定下进行，不存在跨工作区引用同一附件的可能性。每个附件记录都严格绑定到其所属工作区和项目。

---

## 8. 前端预览组件的权限继承

### 8.1 附件列表与预览

前端附件展示组件分为两种视图：

1. **详情面板视图**：[IssueAttachmentsDetail](file:///d:/fz/0508-3/solo-dogfeeding/code/194-plane/apps/web/core/components/issues/attachment/attachment-detail.tsx) — 卡片式展示，点击通过 `<Link href={fileURL}>` 在新标签页打开下载。
2. **列表项视图**：[IssueAttachmentsListItem](file:///d:/fz/0508-3/solo-dogfeeding/code/194-plane/apps/web/core/components/issues/attachment/attachment-list-item.tsx) — 紧凑列表，点击通过 `window.open(fileURL, "_blank")` 打开。

两者都通过 `getFileURL(attachment.asset_url)` 构造下载 URL，本质上是向后端 API 端点发起 GET 请求。

### 8.2 权限继承逻辑

前端附件组件本身 **不进行独立的权限判断**，权限完全继承自其父级上下文：

1. **页面级权限**：用户能访问 issue 详情页，意味着已通过了工作区/项目级别的成员校验。
2. **组件级禁用**：通过 `disabled` prop 控制是否允许上传/删除操作，该 prop 由上层组件根据用户角色（如 GUEST 角色可能禁用上传）传入。
3. **下载权限**：点击下载时，浏览器直接请求后端 API（携带认证 Cookie），后端通过 `allow_permission` 再次校验权限。若权限不足，返回 403 而非签名 URL。

### 8.3 Space（公开门户）的附件访问

[EntityAssetEndpoint](file:///d:/fz/0508-3/solo-dogfeeding/code/194-plane/apps/api/plane/space/views/asset.py#L27-L67) 用于 Plane Space 公开工单板：

- **GET 请求**：`AllowAny` 权限，无需认证。通过 `anchor`（部署标识）找到对应的 DeployBoard，再查找其工作区下的 FileAsset。仅允许访问 `ISSUE_DESCRIPTION` 和 `COMMENT_DESCRIPTION` 类型的资产。
- **POST/PATCH/DELETE**：`IsAuthenticated` 权限，且必须属于已发布的 DeployBoard。

**注意**：Space 的 GET 端点不校验 `ISSUE_ATTACHMENT` 类型，意味着工单的显式附件（非描述内嵌图片）**不会**在公开门户中暴露，仅描述和评论中的内嵌图片可被匿名用户访问。

### 8.4 StaticFileAssetEndpoint 的特殊处理

[StaticFileAssetEndpoint](file:///d:/fz/0508-3/solo-dogfeeding/code/194-plane/apps/api/plane/app/views/asset/v2.py#L437-L470) 使用 `AllowAny` 权限，仅允许访问以下类型：

- `USER_AVATAR`
- `USER_COVER`
- `WORKSPACE_LOGO`
- `PROJECT_COVER`

这些资源属于公共展示类资产，不涉及敏感信息，因此无需认证即可访问。

---

## 9. 安全措施总结

| 安全维度 | 实现机制 |
|----------|----------|
| **文件类型校验** | 前端：file-type 库从文件签名检测 MIME；后端：`ATTACHMENT_MIME_TYPES` 白名单 |
| **危险扩展名** | 前端：`DANGEROUS_EXTENSIONS` 列表（exe/bat/sh/php 等），禁止双扩展名和隐藏文件 |
| **文件大小限制** | `FILE_SIZE_LIMIT` 环境变量，默认 5 MB，前后端双重校验 |
| **文件名消毒** | `sanitize_filename` 去除路径分隔符等危险字符 |
| **签名上传** | 预签名 POST 包含 bucket/key/Content-Type/content-length-range 条件约束 |
| **签名下载** | 所有下载通过后端重定向至临时签名 URL，签名默认 1 小时过期 |
| **权限分层** | PROJECT 级（工单附件）和 WORKSPACE 级（工作区资产）两级权限控制 |
| **角色控制** | ADMIN/MEMBER/GUEST 三级角色，删除操作仅 ADMIN 或创建者 |
| **工作区隔离** | 存储键包含 workspace_id，查询均限定工作区范围 |
| **软删除** | 附件删除为软删除，可恢复，硬删除由定时任务在 60 天后执行 |
| **限流** | `AssetRateThrottle` 针对资产操作进行频率限制 |
| **公共资产白名单** | `StaticFileAssetEndpoint` 仅对头像/Logo/封面等公共类型允许匿名访问 |
