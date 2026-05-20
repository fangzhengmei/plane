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

### 1.2 外链（超链接）节点的数据结构

外链通过 `CustomLinkExtension` 实现，但它是一个 **Mark（标记）** 类型，而非 Node（节点）类型。定义在 `packages/editor/src/core/extensions/custom-link/extension.tsx:97-275`：

```typescript
export const CustomLinkExtension = Mark.create<LinkOptions, CustomLinkStorage>({
  name: CORE_EXTENSIONS.CUSTOM_LINK,

  addAttributes() {
    return {
      href: { default: null },
      target: { default: "_blank" },
      rel: { default: "noopener noreferrer nofollow" },
      class: { default: "text-accent-secondary underline ..." },
    };
  },

  parseHTML() {
    return [{ tag: "a[href]" }];  // 解析标准 <a> 标签
  },

  renderHTML({ HTMLAttributes }) {
    return ["a", mergeAttributes(this.options.HTMLAttributes, HTMLAttributes), 0];
  },
});
```

**关键特性：**
- 渲染为标准 `<a>` 标签，不是自定义组件标签
- 支持 `http`、`https` 协议，可扩展自定义协议
- 内置安全过滤：阻止 `javascript:`、`data:`、`vbscript:` 等危险协议
- 提供三种插件：autolink（自动链接）、clickHandler（点击处理）、pasteHandler（粘贴处理）

### 1.3 工作项嵌入节点

工作项嵌入使用 `<issue-embed-component>` 自定义标签，定义在 `packages/editor/src/core/extensions/work-item-embed/extension-config.ts:11-49`：

```typescript
export const WorkItemEmbedExtensionConfig = Node.create({
  name: CORE_EXTENSIONS.WORK_ITEM_EMBED,
  group: "block",
  atom: true,

  addAttributes() {
    return {
      entity_identifier: { default: undefined },
      project_identifier: { default: undefined },
      workspace_identifier: { default: undefined },
      id: { default: undefined },
      entity_name: { default: undefined },
    };
  },

  parseHTML() {
    return [{ tag: "issue-embed-component" }];
  },

  renderHTML({ HTMLAttributes }) {
    return ["issue-embed-component", mergeAttributes(HTMLAttributes)];
  },
});
```

### 1.4 TipTap 扩展配置汇总

| 扩展类型 | 标签/选择器 | 类型 | 状态 |
|---------|------------|------|------|
| CUSTOM_IMAGE | `<image-component>` | Node (block) | ✅ 完整实现 |
| CUSTOM_LINK | `<a[href]>` | Mark | ✅ 完整实现 |
| WORK_ITEM_EMBED | `<issue-embed-component>` | Node (block) | ✅ 扩展实现 |
| IMAGE | `<img>` | Node | ⚠️ 基础实现 |
| MENTION | `<mention-component>` | Node | ✅ 完整实现 |

### 1.5 资产元数据提取

系统通过 `CORE_ASSETS_META_DATA_RECORD` 注册表从 ProseMirror 节点提取资产元数据，定义在 `packages/editor/src/core/helpers/assets.ts:20-46`：

```typescript
export const CORE_ASSETS_META_DATA_RECORD: Partial<
  Record<CORE_EXTENSIONS | ADDITIONAL_EXTENSIONS, TAssetMetaDataRecord>
> = {
  [CORE_EXTENSIONS.IMAGE]: (attrs) => {
    if (!attrs?.src) return;
    return {
      href: `#${getImageBlockId(attrs?.id ?? "")}`,
      id: attrs?.id,
      name: `image-${attrs?.id}`,
      size: 0,
      src: attrs?.src,
      type: CORE_EXTENSIONS.IMAGE,
    };
  },
  [CORE_EXTENSIONS.CUSTOM_IMAGE]: (attrs) => {
    if (!attrs?.src) return;
    return {
      href: `#${getImageBlockId(attrs?.id ?? "")}`,
      id: attrs?.id,
      name: `image-${attrs?.id}`,
      size: 0,
      src: attrs?.src,
      type: CORE_EXTENSIONS.CUSTOM_IMAGE,
    };
  },
  // 注意：没有 CUSTOM_LINK、WORK_ITEM_EMBED 的注册
  ...ADDITIONAL_ASSETS_META_DATA_RECORD,
};
```

**CE 版本扩展** (`packages/editor/src/ce/constants/assets.ts:12`):
```typescript
export const ADDITIONAL_ASSETS_META_DATA_RECORD: Partial<Record<ADDITIONAL_EXTENSIONS, TAssetMetaDataRecord>> = {};
```
> 🔴 **类型预留但无实现**：CE 版本的资产元数据注册表为空对象，`TAdditionalEditorAsset = never` 表示没有扩展资产类型。

## 二、资产引用追踪机制

### 2.1 数据库模型

#### FileAsset 模型

`FileAsset` 模型 (`apps/api/plane/db/models/asset.py:31-103`) 是资产追踪的核心：

```python
class FileAsset(BaseModel):
    class EntityTypeContext(models.TextChoices):
        ISSUE_ATTACHMENT = "ISSUE_ATTACHMENT"
        ISSUE_DESCRIPTION = "ISSUE_DESCRIPTION"
        COMMENT_DESCRIPTION = "COMMENT_DESCRIPTION"
        PAGE_DESCRIPTION = "PAGE_DESCRIPTION"  # 页面描述中的图片
        USER_COVER = "USER_COVER"
        USER_AVATAR = "USER_AVATAR"
        WORKSPACE_LOGO = "WORKSPACE_LOGO"
        PROJECT_COVER = "PROJECT_COVER"
        DRAFT_ISSUE_ATTACHMENT = "DRAFT_ISSUE_ATTACHMENT"
        DRAFT_ISSUE_DESCRIPTION = "DRAFT_ISSUE_DESCRIPTION"

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

> 🟡 **外链资源不适用此模型**：`FileAsset` 主要用于存储上传的文件资源，外链（URL）资源不会创建 FileAsset 记录。

#### PageLog 模型

`PageLog` 模型 (`apps/api/plane/db/models/page.py:80-117`) 记录页面内容中的资产变更：

```python
class PageLog(BaseModel):
    TYPE_CHOICES = (
        ("to_do", "To Do"),
        ("issue", "issue"),
        ("image", "Image"),
        ("video", "Video"),
        ("file", "File"),
        ("link", "Link"),
        ("cycle", "Cycle"),
        ("module", "Module"),
        ("back_link", "Back Link"),
        ("forward_link", "Forward Link"),
        ("page_mention", "Page Mention"),
        ("user_mention", "User Mention"),
    )
    transaction = models.UUIDField(default=uuid.uuid4)
    page = models.ForeignKey(Page, related_name="page_log", on_delete=models.CASCADE)
    entity_identifier = models.UUIDField(null=True, blank=True)
    entity_name = models.CharField(max_length=30, verbose_name="Transaction Type")
    entity_type = models.CharField(max_length=30, verbose_name="Entity Type", null=True, blank=True)
    workspace = models.ForeignKey("db.Workspace", ...)
```

### 2.2 PageLog 类型与实际追踪对象对照表

| TYPE_CHOICES | 对应组件标签 | COMPONENT_MAP 注册 | 实际追踪状态 |
|-------------|-------------|-------------------|-------------|
| `image` | `<image-component>` | ✅ 已注册 | ✅ 完整追踪 |
| `issue` | `<mention-component>` (entity_name="issue") | ✅ 已注册 | ✅ 完整追踪 |
| `to_do` | 未知 | ❌ 未注册 | 🔴 仅类型预留 |
| `video` | 无对应组件 | ❌ 未注册 | 🔴 仅类型预留 |
| `file` | 无对应组件 | ❌ 未注册 | 🔴 仅类型预留 |
| `link` | 无对应组件（`<a>` 是 Mark） | ❌ 未注册 | 🔴 仅类型预留 |
| `cycle` | 未知 | ❌ 未注册 | 🔴 仅类型预留 |
| `module` | 未知 | ❌ 未注册 | 🔴 仅类型预留 |
| `back_link` | 未知 | ❌ 未注册 | 🔴 仅类型预留 |
| `forward_link` | 未知 | ❌ 未注册 | 🔴 仅类型预留 |
| `page_mention` | `<mention-component>` (entity_name="page") | ✅ 已注册 | ⚠️ 部分追踪 |
| `user_mention` | `<mention-component>` (entity_name="user") | ✅ 已注册 | ⚠️ 部分追踪 |

### 2.3 页面事务处理

`page_transaction` 任务 (`apps/api/plane/bgtasks/page_transaction_task.py:84-142`) 负责追踪页面内容变化：

```python
@shared_task
def page_transaction(new_description_html, old_description_html, page_id):
    page = Page.objects.get(pk=page_id)
    has_existing_logs = PageLog.objects.filter(page_id=page_id).exists()

    # 1. 单次遍历提取所有组件（优化性能）
    old_components = extract_all_components(old_description_html)
    new_components = extract_all_components(new_description_html)

    new_transactions = []
    deleted_transaction_ids = set()

    for component in component_map.keys():
        old_entities = old_components[component]
        new_entities = new_components[component]

        # 2. 比较差异，确定新增和删除
        old_ids = {m.get("id") for m in old_entities if m.get("id")}
        new_ids = {m.get("id") for m in new_entities if m.get("id")}
        deleted_transaction_ids.update(old_ids - new_ids)

        for mention in new_entities:
            mention_id = mention.get("id")
            if not mention_id or (mention_id in old_ids and has_existing_logs):
                continue

            details = get_entity_details(component, mention)
            new_transactions.append(
                PageLog(
                    transaction=mention_id,
                    page_id=page_id,
                    entity_identifier=details["entity_identifier"],
                    entity_name=details["entity_name"],
                    entity_type=details["entity_type"],
                    workspace_id=page.workspace_id,
                )
            )

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
    "mention-component": {
        "attributes": ["id", "entity_identifier", "entity_name", "entity_type"],
        "extract": lambda m: {
            "entity_name": m.get("entity_name"),
            "entity_type": None,
            "entity_identifier": m.get("entity_identifier"),
        },
    },
    "image-component": {
        "attributes": ["id", "src"],
        "extract": lambda m: {
            "entity_name": "image",
            "entity_type": None,
            "entity_identifier": m.get("src"),
        },
    },
    # 🔴 缺失：issue-embed-component、<a> 标签等
}

component_map = {
    **COMPONENT_MAP,
}

def extract_all_components(description_html):
    soup = BeautifulSoup(description_html, "html.parser")
    results = {}
    for component, config in component_map.items():
        attributes = config.get("attributes", ["id"])
        component_tags = soup.find_all(component)
        entities = []
        for tag in component_tags:
            entity = {attr: tag.get(attr) for attr in attributes}
            entities.append(entity)
        results[component] = entities
    return results
```

> 🟡 **外链资源追踪缺失**：`<a>` 标签是 Mark 类型，不是自定义组件，不会被 `extract_all_components` 提取。`<issue-embed-component>` 有 TipTap 扩展但未在 `COMPONENT_MAP` 中注册，也不会被追踪。

### 2.5 调用链路

`page_transaction` 在以下场景被调用 (`apps/api/plane/app/views/page/base.py`):

- **创建页面** (line 144)：`old_description_html=None`
- **更新页面** (line 188)：对比新旧描述
- **从模板创建** (line 561)
- **复制页面** (line 613)

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

> 🔵 **外链资源无需上传流程**：外链是纯URL引用，不涉及文件上传，不会创建 FileAsset 记录。

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

`extractAssetsFromHTMLContent` 的实现：
```typescript
const extractAssetsFromHTMLContent = (htmlContent: string): string[] => {
  const parser = new DOMParser();
  const doc = parser.parseFromString(htmlContent, "text/html");
  const assetSources = new Set<string>();

  // 只提取 image-component 的 src
  const imageComponents = doc.querySelectorAll("image-component");
  imageComponents.forEach((component) => {
    const src = component.getAttribute("src");
    if (src) assetSources.add(src);
  });

  const additionalAssetIds = extractAdditionalAssetsFromHTMLContent(htmlContent);
  return [...Array.from(assetSources), ...additionalAssetIds];
};
```

> 🟡 **外链资源不参与复制**：`extractAssetsFromHTMLContent` 只提取 `<image-component>` 的 `src` 属性，外链（`<a>` 标签）、工作项嵌入（`<issue-embed-component>`）不会被提取和复制。

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
    # Step 1: 从description_html提取资产ID（只处理 image-component）
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

#### PageLog 删除清理

在 `page_transaction` 任务中，当组件从页面中移除时，对应的 PageLog 记录会被硬删除：

```python
if deleted_transaction_ids:
    PageLog.objects.filter(transaction__in=deleted_transaction_ids).delete()
```

> 🟡 **外链资源删除行为**：
> - 外链（`<a>` 标签）不会创建 PageLog 记录，删除时无需清理
> - 从页面移除外链不会触发任何资产清理逻辑
> - 外链是纯URL引用，不占用存储资源，无需回收

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

## 四、外链资源管理协同流程总览

### 4.1 资源类型对比矩阵

| 特性 | 图片附件 (`<image-component>`) | 外链 (`<a>`) | 工作项嵌入 (`<issue-embed-component>`) |
|-----|------------------------------|-------------|------------------------------------|
| 节点类型 | Node (block) | Mark | Node (block) |
| 自定义标签 | ✅ | ❌ (标准 `<a>`) | ✅ |
| FileAsset 记录 | ✅ | ❌ | ❌ |
| PageLog 追踪 | ✅ | ❌ | ❌ |
| 资产元数据注册 | ✅ | ❌ | ❌ |
| 复制时处理 | ✅（复制S3对象） | ✅（原样保留） | ✅（原样保留） |
| 删除时清理 | ✅（软删除+硬删除） | ❌（无需清理） | ❌（无需清理） |
| 存储占用 | ✅（S3存储） | ❌ | ❌ |

### 4.2 外链资源全生命周期

```
用户输入/粘贴URL
    ↓
CustomLinkExtension 解析
    ├─ autolink 插件：自动识别URL并转换为 <a> 标签
    ├─ pasteHandler 插件：粘贴URL时自动创建链接
    └─ 安全过滤：阻止 javascript:/data:/vbscript: 协议
    ↓
渲染为标准 <a href="..." target="_blank" rel="noopener noreferrer nofollow">
    ↓
页面保存
    ├─ 保存到 description_html（包含完整 <a> 标签）
    ├─ page_transaction 任务触发
    │   └─ extract_all_components 只处理 mention-component 和 image-component
    │      └─ ❌ <a> 标签被忽略，不创建 PageLog 记录
    ↓
页面复制
    ├─ getEditorContentWithReplacedAssets
    │   └─ extractAssetsFromHTMLContent 只提取 image-component 的 src
    │      └─ ✅ <a> 标签原样保留，无需特殊处理
    ↓
页面删除外链
    ├─ 直接从 ProseMirror doc 中移除 Mark
    ├─ description_html 更新后不再包含该 <a> 标签
    └─ ❌ 无 PageLog 记录需要删除，无 FileAsset 需要清理
```

### 4.3 图片附件全生命周期

```
用户上传图片
    ↓
EditorAssetStore.uploadEditorAsset
    ├─ 创建 FileAsset (is_uploaded=False)
    ├─ 生成 S3 预签名 URL
    └─ 前端直传 S3
    ↓
插入 <image-component id="..." src="..." status="uploaded">
    ↓
页面保存
    ├─ page_transaction.delay(new_html, old_html, page_id)
    │   ├─ extract_all_components 提取 image-component
    │   ├─ 对比差异，新增 PageLog (entity_name="image")
    │   └─ 批量插入 PageLog 记录
    ↓
页面复制
    ├─ extractAssetsFromHTMLContent 提取所有 image-component src
    ├─ DuplicateAssetEndpoint 批量复制 S3 对象
    ├─ replaceAssetsInHTMLContent 更新 src 属性
    └─ 生成新 FileAsset 记录
    ↓
删除图片
    ├─ 从编辑器中移除 <image-component>
    ├─ page_transaction 检测到删除
    │   └─ PageLog.objects.filter(transaction__in=deleted_ids).delete()
    └─ ⚠️ FileAsset 记录保留（不自动删除）
    ↓
页面删除
    ├─ soft_delete_related_objects 递归软删除
    │   └─ Page 软删除 → 关联 FileAsset 级联软删除
    └─ 30天后 hard_delete 任务彻底删除
```

## 五、已实现功能 vs 类型预留

### 5.1 已完整实现的功能

✅ **图片附件管理**
- 自定义 `<image-component>` 节点
- TipTap 扩展配置（解析+渲染）
- 上传状态追踪（5种状态）
- FileAsset 模型记录
- PageLog 变更追踪
- 资产复制（S3对象复制）
- 软删除+定期硬删除

✅ **超链接（外链）**
- 标准 `<a>` 标签渲染
- 自动链接识别（autolink）
- 粘贴自动链接（pasteHandler）
- 安全协议过滤
- 链接点击处理

✅ **@提及**
- `<mention-component>` 节点
- PageLog 追踪（entity_name 区分）
- 差异检测与清理

### 5.2 仅类型预留、未实际实现的功能

🔴 **PageLog.TYPE_CHOICES 中未实现的类型**
- `video` - 无对应组件，无追踪逻辑
- `file` - 无对应组件，无追踪逻辑
- `link` - 外链是 Mark 类型，未注册追踪
- `to_do` - 无对应组件
- `cycle` - 无对应组件
- `module` - 无对应组件
- `back_link` - 无对应组件
- `forward_link` - 无对应组件

🔴 **工作项嵌入追踪缺失**
- `<issue-embed-component>` 有 TipTap 扩展
- 但未在 `COMPONENT_MAP` 中注册
- page_transaction 不会追踪其变更

🔴 **CE 版本资产扩展**
- `ADDITIONAL_ASSETS_META_DATA_RECORD = {}`（空对象）
- `TAdditionalEditorAsset = never`
- 无扩展资产类型实现

### 5.3 关键设计决策分析

**1. 外链不做追踪的原因**
- 外链是纯 URL 引用，不占用存储资源
- 外链失效不影响系统数据完整性
- `<a>` 是 Mark 类型，不是独立节点，追踪难度大
- 外链数量可能很多，追踪成本高

**2. FileAsset 不自动删除的原因**
- 图片可能被多个页面/实体引用
- 删除图片节点 ≠ 资产一定可以删除
- 软删除+定期清理提供恢复窗口
- 避免误删导致的数据丢失

**3. PageLog 硬删除的原因**
- PageLog 是审计追踪数据，不是核心业务数据
- 删除组件后保留日志无意义
- 定期清理避免表膨胀

## 六、协同结构概览

```
富文本编辑 (TipTap)
    ↓ 节点变更
┌─────────────────────────────────────────┐
│  节点类型                                │
├──────────┬──────────┬──────────────────┤
│ <image-component> │ <a> (Mark) │ <mention-component> │
└──────────┴──────────┴──────────────────┘
    │           │              │
    │           │              └─→ extract_all_components → PageLog (entity_name 区分)
    │           │
    │           └─→ 无追踪，原样保存
    │
    └─→ CORE_ASSETS_META_DATA_RECORD
          │
          ├─→ 上传时: EditorAssetStore → FileService → S3
          │                     → FileAsset (is_uploaded=False)
          │                     → 上传完成 → is_uploaded=True
          │
          ├─→ 保存时: page_transaction 任务
          │           ↓ 解析HTML
          │         extract_all_components
          │           ↓ 差异对比
          │         PageLog (entity_name="image")
          │
          └─→ 复制时: getEditorContentWithReplacedAssets
                       ↓ 提取资产ID
                     duplicateAssetService
                       ↓ S3对象复制
                     replaceAssetsInHTMLContent
                       ↓ 格式转换
                     convertHTMLDocumentToAllFormats

生命周期管理
    ├─ 图片: 软删除(FileAsset) → 30天后硬删除(hard_delete 任务)
    ├─ 外链: 直接移除，无需清理
    └─ 版本清理: delete_page_versions 任务 (保留最近20个版本)
```

## 七、关键设计特点

1. **自定义HTML标签**：使用 `<image-component>` 而非标准 `<img>` 标签，支持丰富的元数据属性

2. **S3直传模式**：后端仅生成预签名URL，前端直接上传到S3，减少服务器带宽压力

3. **异步任务驱动**：资产复制、页面事务处理、历史清理均通过 Celery 异步执行

4. **软删除为主**：所有删除操作先标记软删除，30天后才真正硬删除，提供恢复窗口

5. **差异化追踪策略**：
   - 图片附件：完整追踪（FileAsset + PageLog）
   - 外链：不追踪（纯URL引用，无存储成本）
   - @提及：部分追踪（PageLog 记录）

6. **可扩展架构**：通过 `ADDITIONAL_ASSETS_META_DATA_RECORD` 支持扩展新的资产类型（当前CE版本为空）

7. **批量优化**：使用 `bulk_create`、单次解析多组件提取等优化手段提升性能
