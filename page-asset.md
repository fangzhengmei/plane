# 文档型页面图片附件与外链资源管理机制分析

## 一、富文本节点解析机制

### 1.1 图片节点的数据结构

Plane 使用自定义 HTML 标签 `<image-component>` 来表示富文本中的图片节点。该节点的属性定义在 `packages/editor/src/core/extensions/custom-image/types.ts:11-49`：

```typescript
enum ECustomImageAttributeNames {
  ID = "id",
  WIDTH = "width",
  HEIGHT = "height",
  ASPECT_RATIO = "aspectRatio",
  SOURCE = "src",
  ALIGNMENT = "alignment",
  STATUS = "status",
}

type TCustomImageAttributes = {
  id: string | null;
  width: PixelAttribute<"35%" | number> | null;
  height: PixelAttribute<"auto" | number> | null;
  aspectRatio: number | null;
  src: string | null;
  alignment: "left" | "center" | "right";
  status: ECustomImageStatus;
};
```

**图片状态机** (`ECustomImageStatus`):
- `pending` - 待上传
- `uploading` - 上传中
- `uploaded` - 已上传
- `duplicating` - 复制中
- `duplication-failed` - 复制失败

### 1.2 TipTap 扩展配置

图片节点通过 TipTap 扩展进行解析和渲染，配置在 `packages/editor/src/core/extensions/custom-image/extension-config.ts:33-69`：

```typescript
export const CustomImageExtensionConfig = BaseImageExtension.extend({
  name: CORE_EXTENSIONS.CUSTOM_IMAGE,
  group: "block",
  atom: true,

  parseHTML() {
    return [{ tag: "image-component" }];
  },

  renderHTML({ HTMLAttributes }) {
    return ["image-component", mergeAttributes(HTMLAttributes)];
  },
});
```

### 1.3 资产元数据提取

系统通过 `CORE_ASSETS_META_DATA_RECORD` 注册表从 ProseMirror 节点提取资产元数据，定义在 `packages/editor/src/core/helpers/assets.ts:20-46`：

```typescript
export const CORE_ASSETS_META_DATA_RECORD: Partial<
  Record<CORE_EXTENSIONS | ADDITIONAL_EXTENSIONS, TAssetMetaDataRecord>
> = {
  [CORE_EXTENSIONS.CUSTOM_IMAGE]: (attrs) => {
    if (!attrs?.src) return;
    return {
      href: `#${getImageBlockId(attrs?.id ?? "")}`,
      id: attrs?.id,
      name: `image-${attrs?.id}`,
      src: attrs?.src,
      type: CORE_EXTENSIONS.CUSTOM_IMAGE,
    };
  },
  // ... 其他资产类型
};
```

## 二、资产引用追踪机制

### 2.1 数据库模型

`FileAsset` 模型 (`apps/api/plane/db/models/asset.py:31-103`) 是资产追踪的核心：

```python
class FileAsset(BaseModel):
    class EntityTypeContext(models.TextChoices):
        ISSUE_ATTACHMENT = "ISSUE_ATTACHMENT"
        ISSUE_DESCRIPTION = "ISSUE_DESCRIPTION"
        COMMENT_DESCRIPTION = "COMMENT_DESCRIPTION"
        PAGE_DESCRIPTION = "PAGE_DESCRIPTION"  # 页面描述中的图片
        # ... 其他类型

    # 核心字段
    asset = models.FileField(upload_to=get_upload_path, max_length=800)
    entity_type = models.CharField(max_length=255)  # 资产类型
    entity_identifier = models.CharField(max_length=255)  # 实体ID
    is_deleted = models.BooleanField(default=False)
    is_uploaded = models.BooleanField(default=False)
    storage_metadata = models.JSONField(default=dict)

    # 关联字段
    workspace = models.ForeignKey("db.Workspace", ...)
    project = models.ForeignKey("db.Project", ...)
    page = models.ForeignKey("db.Page", ...)
    issue = models.ForeignKey("db.Issue", ...)
```

**关键索引设计** (`asset.py:72-77`):
- `asset_entity_type_idx` - 按实体类型查询
- `asset_entity_identifier_idx` - 按实体标识查询
- `asset_entity_idx` - 复合索引（类型+标识）

### 2.2 页面内容变更追踪

`PageLog` 模型 (`apps/api/plane/db/models/page.py:80-117`) 记录页面内容中的资产变更：

```python
class PageLog(BaseModel):
    TYPE_CHOICES = (
        ("image", "Image"),
        ("file", "File"),
        ("link", "Link"),
        # ... 其他类型
    )
    transaction = models.UUIDField(default=uuid.uuid4)
    page = models.ForeignKey(Page, ...)
    entity_identifier = models.UUIDField(null=True)  # 资产ID
    entity_name = models.CharField(max_length=30)    # 实体类型名称
    entity_type = models.CharField(max_length=30)    # 实体类型
```

### 2.3 页面事务处理

`page_transaction` 任务 (`apps/api/plane/bgtasks/page_transaction_task.py:84-142`) 负责追踪页面内容变化：

```python
@shared_task
def page_transaction(new_description_html, old_description_html, page_id):
    # 1. 单次遍历提取所有组件（优化性能）
    old_components = extract_all_components(old_description_html)
    new_components = extract_all_components(new_description_html)

    # 2. 比较差异，确定新增和删除
    old_ids = {m.get("id") for m in old_entities if m.get("id")}
    new_ids = {m.get("id") for m in new_entities if m.get("id")}
    deleted_transaction_ids = old_ids - new_ids

    # 3. 批量插入新增记录
    if new_transactions:
        PageLog.objects.bulk_create(new_transactions, batch_size=50, ignore_conflicts=True)

    # 4. 清理已删除记录
    if deleted_transaction_ids:
        PageLog.objects.filter(transaction__in=deleted_transaction_ids).delete()
```

### 2.4 组件提取机制

`extract_all_components` 函数 (`page_transaction_task.py:45-71`) 使用 BeautifulSoup 单次解析提取所有组件：

```python
COMPONENT_MAP = {
    "image-component": {
        "attributes": ["id", "src"],
        "extract": lambda m: {
            "entity_name": "image",
            "entity_identifier": m.get("src"),
        },
    },
    "mention-component": { ... },
}

def extract_all_components(description_html):
    soup = BeautifulSoup(description_html, "html.parser")
    results = {}
    for component, config in component_map.items():
        component_tags = soup.find_all(component)
        # ... 提取属性
    return results
```

## 三、资产生命周期管理

### 3.1 上传流程

#### 前端上传管理
`EditorAssetStore` (`apps/web/core/store/editor/asset.store.ts:51-148`) 管理前端资产上传状态：

```typescript
export class EditorAssetStore implements IEditorAssetStore {
  assetsUploadStatus: Record<string, TAttachmentUploadStatus> = {};

  async uploadEditorAsset(args: {
    blockId: string;
    data: TFileEntityInfo;
    file: File;
    projectId?: string;
    workspaceSlug: string;
  }): Promise<TFileSignedURLResponse> {
    // 1. 更新上传状态（进度追踪）
    // 2. 调用 fileService.uploadProjectAsset 或 uploadWorkspaceAsset
    // 3. 监听上传进度，debounce 更新UI
  }
}
```

#### 后端上传接口
`ProjectAssetEndpoint.post` (`apps/api/plane/app/views/asset/v2.py:517-582`) 处理资产上传：

```python
def post(self, request, slug, project_id):
    # 1. 生成 S3 存储路径: {workspace_id}/{uuid4}-{filename}
    asset_key = f"{workspace.id}/{uuid.uuid4().hex}-{name}"

    # 2. 创建 FileAsset 记录（is_uploaded=False）
    asset = FileAsset.objects.create(
        asset=asset_key,
        entity_type=entity_type,
        project_id=project_id,
        **self.get_entity_id_field(entity_type, entity_identifier),
    )

    # 3. 生成预签名URL供前端直接上传到S3
    storage = S3Storage(request=request)
    presigned_url = storage.generate_presigned_post(...)

    # 4. 返回 asset_id 和 presigned_url
```

### 3.2 资产复制机制

#### 前端资产提取与替换
`getEditorContentWithReplacedAssets` (`packages/editor/src/core/helpers/parser.ts:66-100`) 处理页面复制时的资产复制：

```typescript
export const getEditorContentWithReplacedAssets = async (props) => {
  // Step 1: 从HTML中提取所有资产ID
  const assetIds = extractAssetsFromHTMLContent(descriptionHTML);

  // Step 2: 调用后端API批量复制资产
  const duplicateAssetsResponse = await duplicateAssetService({
    entity_id: entityId,
    entity_type: entityType,
    project_id: projectId,
    asset_ids: assetIds,
  });

  // Step 3: 替换HTML中的资产引用
  replacedDescription = replaceAssetsInHTMLContent({
    htmlContent: descriptionHTML,
    assetMap: duplicateAssetsResponse,
  });

  // Step 4: 转换为完整文档格式（JSON + Binary）
  return convertHTMLDocumentToAllFormats(...);
};
```

#### 后端资产复制
`DuplicateAssetEndpoint.post` (`apps/api/plane/app/views/asset/v2.py:705-795`) 处理资产复制：

```python
def post(self, request, slug, asset_id):
    # 1. 查找原始资产
    original_asset = FileAsset.objects.filter(
        id=asset_id, is_uploaded=True, workspace_id__in=user_workspace_ids
    ).first()

    # 2. 创建新资产记录
    duplicated_asset = FileAsset.objects.create(
        asset=destination_key,
        entity_type=entity_type,
        project_id=project_id,
        storage_metadata=original_asset.storage_metadata,
        **self.get_entity_id_field(entity_type, entity_id),
    )

    # 3. 异步复制S3对象
    storage.copy_object(original_asset.asset, destination_key)

    # 4. 立即标记为已上传（因为S3复制是异步的）
    FileAsset.objects.filter(id=duplicated_asset.id).update(is_uploaded=True)
```

#### 后台任务复制
`copy_s3_objects_of_description_and_assets` (`apps/api/plane/bgtasks/copy_s3_object.py:124-155`) 处理批量复制：

```python
@shared_task
def copy_s3_objects_of_description_and_assets(...):
    # Step 1: 从description_html提取资产ID
    asset_ids = extract_asset_ids(entity.description_html, "image-component")

    # Step 2: 复制所有资产
    duplicated_assets = copy_assets(entity, entity_identifier, project_id, asset_ids, user_id)

    # Step 3: 更新HTML中的资产引用
    updated_html = update_description(entity, duplicated_assets, "image-component")

    # Step 4: 请求live-server重新生成二进制和JSON格式
    external_data = sync_with_external_service(entity_name, updated_html)
```

### 3.3 删除与回收机制

#### 软删除策略
所有资产删除均采用软删除方式，通过 `is_deleted` 和 `deleted_at` 字段标记：

```python
# asset/v2.py:403-411
def delete(self, request, slug, asset_id):
    asset = FileAsset.objects.get(id=asset_id, workspace__slug=slug)
    asset.is_deleted = True
    asset.deleted_at = timezone.now()
    asset.save(update_fields=["is_deleted", "deleted_at"])
```

#### 关联删除
`soft_delete_related_objects` (`apps/api/plane/bgtasks/deletion_task.py:18-106`) 递归处理级联删除：

```python
@shared_task
def soft_delete_related_objects(app_label, model_name, instance_pk):
    instance = model_class.all_objects.get(pk=instance_pk)

    for relation in all_related:
        if on_delete_name == "CASCADE":
            # 递归软删除关联对象
            related_obj.deleted_at = timezone.now()
            related_obj.save()
            soft_delete_related_objects(...)
```

#### 硬删除清理
`hard_delete` 任务 (`deletion_task.py:114-192`) 定期清理超过保留期的软删除记录：

```python
@shared_task
def hard_delete():
    days = settings.HARD_DELETE_AFTER_DAYS  # 默认30天

    # 按模型批量硬删除
    _ = Workspace.all_objects.filter(
        deleted_at__lt=timezone.now() - timezone.timedelta(days=days)
    ).delete()

    # Page 及其关联资产
    _ = Page.all_objects.filter(deleted_at__lt=...).delete()

    # 最后遍历所有带deleted_at字段的模型进行清理
    for model in apps.get_models():
        if hasattr(model, "deleted_at"):
            _ = model.all_objects.filter(deleted_at__lt=...).delete()
```

### 3.4 版本历史清理

`delete_page_versions` 任务 (`apps/api/plane/bgtasks/cleanup_task.py:447-455`) 控制页面版本保留数量：

```python
def get_page_versions_queryset():
    # 使用窗口函数，每个页面只保留最近20个版本
    subq = (
        PageVersion.all_objects.annotate(
            row_num=Window(
                expression=RowNumber(),
                partition_by=[F("page_id")],
                order_by=F("created_at").desc(),
            )
        )
        .filter(row_num__gt=20)
        .values("id")
    )
    return PageVersion.all_objects.filter(id__in=Subquery(subq))
```

## 四、协同结构概览

```
富文本编辑 (TipTap)
    ↓ 节点变更
图片节点 <image-component>
    │  (id, src, status, ...)
    ↓ 提取元数据
CORE_ASSETS_META_DATA_RECORD
    │
    ├─→ 上传时: EditorAssetStore → FileService → S3
    │                          → FileAsset (is_uploaded=False)
    │                          → 上传完成 → is_uploaded=True
    │
    ├─→ 保存时: page_transaction 任务
    │            ↓ 解析HTML
    │          extract_all_components
    │            ↓ 差异对比
    │          PageLog (新增/删除记录)
    │
    └─→ 复制时: getEditorContentWithReplacedAssets
                 ↓ 提取资产ID
               duplicateAssetService
                 ↓ S3对象复制
               replaceAssetsInHTMLContent
                 ↓ 格式转换
               convertHTMLDocumentToAllFormats

生命周期管理
    ├─ 软删除: is_deleted = True
    ├─ 定期清理: hard_delete 任务 (30天后硬删除)
    └─ 版本清理: delete_page_versions 任务 (保留最近20个版本)
```

## 五、关键设计特点

1. **自定义HTML标签**：使用 `<image-component>` 而非标准 `<img>` 标签，支持丰富的元数据属性

2. **S3直传模式**：后端仅生成预签名URL，前端直接上传到S3，减少服务器带宽压力

3. **异步任务驱动**：资产复制、页面事务处理、历史清理均通过 Celery 异步执行

4. **软删除为主**：所有删除操作先标记软删除，30天后才真正硬删除，提供恢复窗口

5. **批量优化**：使用 `bulk_create`、单次解析多组件提取等优化手段提升性能

6. **可扩展架构**：通过 `ADDITIONAL_ASSETS_META_DATA_RECORD` 支持扩展新的资产类型
