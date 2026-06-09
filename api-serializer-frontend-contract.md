# Plane 序列化层与前端契约对齐分析

## 一、整体架构概览

Plane 的序列化-前端契约横跨三个独立 API 入口和一套前端类型系统：

```
后端三套 API 入口                    前端
┌──────────────────────┐
│  app/  (Web 前端用)   │──────────▶ packages/types/src/
│  api/  (开放 API 用)  │──────────▶ (共享同一套类型)
│  space/(公共页面用)   │──────────▶
└──────────────────────┘
```

- **app 层** ([app/serializers/](file:///d:/fz/0508-3/solo-dogfeeding/code/199-plane/apps/api/plane/app/serializers)): 面向 Plane Web 前端，使用 `DynamicBaseSerializer`，支持 `fields` 裁剪与 `expand` 展开
- **api 层** ([api/serializers/](file:///d:/fz/0508-3/solo-dogfeeding/code/199-plane/apps/api/plane/api/serializers)): 面向开放 API / SDK 消费者，使用独立 `BaseSerializer`，也支持 `fields`/`expand`
- **space 层** ([space/serializer/](file:///d:/fz/0508-3/solo-dogfeeding/code/199-plane/apps/api/plane/space/serializer)): 面向公共分享页面，轻量化序列化
- **前端类型** ([packages/types/src/](file:///d:/fz/0508-3/solo-dogfeeding/code/199-plane/packages/types/src)): 纯 TypeScript 接口定义，手动维护，与后端无自动同步

---

## 二、序列化基础机制

### 2.1 两个 BaseSerializer 的差异

| 特性 | app BaseSerializer ([base.py](file:///d:/fz/0508-3/solo-dogfeeding/code/199-plane/apps/api/plane/app/serializers/base.py)) | api BaseSerializer ([base.py](file:///d:/fz/0508-3/solo-dogfeeding/code/199-plane/apps/api/plane/api/serializers/base.py)) |
|------|------|------|
| 继承链 | `serializers.ModelSerializer` | `serializers.ModelSerializer` |
| fields 裁剪 | 不内置（由 `DynamicBaseSerializer` 提供） | 内置 `_filter_fields()` |
| expand 展开 | 不内置（由 `DynamicBaseSerializer` 提供） | 内置 `to_representation()` |
| 裁剪方式 | N/A | **白名单**：移除不在 `allowed` 集合中的字段 |

### 2.2 DynamicBaseSerializer (app 专用)

定义在 [app/serializers/base.py#L12](file:///d:/fz/0508-3/solo-dogfeeding/code/199-plane/apps/api/plane/app/serializers/base.py#L12-L120)。

核心逻辑：

```python
class DynamicBaseSerializer(BaseSerializer):
    def __init__(self, *args, **kwargs):
        fields = kwargs.pop("fields", [])
        self.expand = kwargs.pop("expand", []) or []
        fields = self.expand          # ← 关键：fields 被 expand 覆盖
        super().__init__(*args, **kwargs)
        if fields is not None:
            self.fields = self._filter_fields(fields)
```

**关键设计决策**：`fields = self.expand` 意味着当传入 `expand` 参数时，最终字段集合由 expand 决定，而非原始 fields。这是一个容易让人困惑的设计——传了 `fields` 但实际被 expand 覆盖。

### 2.3 fields 与 expand 的请求参数解析

在 [app/views/base.py#L139-L146](file:///d:/fz/0508-3/solo-dogfeeding/code/199-plane/apps/api/plane/app/views/base.py#L139-L146)：

```python
@property
def fields(self):
    fields = [field for field in self.request.GET.get("fields", "").split(",") if field]
    return fields if fields else None

@property
def expand(self):
    expand = [expand for expand in self.request.GET.get("expand", "").split(",") if expand]
    return expand if expand else None
```

前端通过 URL query 参数 `?fields=state_id,name&expand=assignees` 控制返回字段。api 层的 `BaseAPIView` 有相同实现。

---

## 三、字段裁剪机制详解

### 3.1 app 层裁剪（DynamicBaseSerializer._filter_fields）

路径: [app/serializers/base.py#L26-L120](file:///d:/fz/0508-3/solo-dogfeeding/code/199-plane/apps/api/plane/app/serializers/base.py#L26-L120)

行为：
1. 遍历 fields 列表，处理嵌套字典 `{key: [sub_fields]}`
2. 构建 `allowed` 列表
3. 对不在 allowed 中的字段，**不删除**，而是检查 expansion mapper
4. 如果字段在 expansion mapper 中，动态添加对应的 Lite Serializer
5. **不删除多余字段**（与 api 层的关键区别）

### 3.2 api 层裁剪（BaseSerializer._filter_fields）

路径: [api/serializers/base.py#L32-L70](file:///d:/fz/0508-3/solo-dogfeeding/code/199-plane/apps/api/plane/api/serializers/base.py#L32-L70)

行为：
1. 同样处理嵌套字典
2. 构建 allowed 集合
3. **显式删除**不在 allowed 中的字段：`self.fields.pop(field_name)`
4. 这是严格的白名单模式

### 3.3 列表视图的"裸 .values()"裁剪

最激进的裁剪出现在 [app/views/issue/base.py#L160-L193](file:///d:/fz/0508-3/solo-dogfeeding/code/199-plane/apps/api/plane/app/views/issue/base.py#L160-L193)：

```python
if self.fields or self.expand:
    issues = IssueSerializer(issue_queryset, many=True, fields=self.fields, expand=self.expand).data
else:
    issues = issue_queryset.values(
        "id", "name", "state_id", "sort_order", "completed_at", "estimate_point",
        "priority", "start_date", "target_date", "sequence_id", "project_id",
        "parent_id", "cycle_id", "module_ids", "label_ids", "assignee_ids",
        "sub_issues_count", "created_at", "updated_at", "created_by", "updated_by",
        "attachment_count", "link_count", "is_draft", "archived_at", "deleted_at",
    )
```

**当没有 fields/expand 参数时，列表接口直接用 Django `.values()` 返回原始字典**，完全绕过序列化器。这是性能优化但也是契约风险点——字段名由手写的列表决定，不经过任何 serializer 验证。

同样，[utils/grouper.py#L93-L141](file:///d:/fz/0508-3/solo-dogfeeding/code/199-plane/apps/api/plane/plane/utils/grouper.py#L93-L141) 中 `issue_on_results` 也手写了 `required_fields` 列表：

```python
required_fields = [
    "id", "name", "state_id", "sort_order", "completed_at", "estimate_point",
    "priority", "start_date", "target_date", "sequence_id", "project_id",
    "parent_id", "cycle_id", "sub_issues_count", "created_at", "updated_at",
    "created_by", "updated_by", "attachment_count", "link_count", "is_draft",
    "archived_at", "state__group",
]
```

**风险**：三处手写字段列表（serializer、`.values()`、grouper）需要保持同步，否则同一字段在不同接口返回不同结果。

---

## 四、嵌套关系展开机制

### 4.1 Expansion Mapper

app 层的 expansion mapper 定义在 [app/serializers/base.py#L74-L96](file:///d:/fz/0508-3/solo-dogfeeding/code/199-plane/apps/api/plane/app/serializers/base.py#L74-L96)：

| 字段名 | 展开序列化器 | many |
|--------|-------------|------|
| user | UserLiteSerializer | False |
| workspace | WorkspaceLiteSerializer | False |
| project | ProjectLiteSerializer | False |
| state | StateLiteSerializer | False |
| assignees | UserLiteSerializer | True |
| labels | LabelSerializer | True |
| parent | IssueLiteSerializer | False |
| sub_issues | IssueLiteSerializer | True |
| issue_cycle | CycleIssueSerializer | True |
| issue_relation | IssueRelationSerializer | True |
| issue_reactions | IssueReactionLiteSerializer | True |
| issue_link | IssueLinkLiteSerializer | True |
| issue_attachment | IssueAttachmentLiteSerializer | True |
| default_assignee / project_lead / created_by / actor / owned_by | UserLiteSerializer | False |

api 层的 mapper ([api/serializers/base.py#L91-L106](file:///d:/fz/0508-3/solo-dogfeeding/code/199-plane/apps/api/plane/api/serializers/base.py#L91-L106)) 略有不同：

| 差异字段 | app | api |
|---------|-----|-----|
| updated_by | ❌ | UserLiteSerializer |
| estimate_point | ❌ | EstimatePointSerializer |
| labels | LabelSerializer | ❌ (在 IssueExpandSerializer 单独处理) |
| issue_cycle / issue_relation / issue_reactions / issue_link / sub_issues | 各自 Lite Serializer | ❌ |

### 4.2 展开的两阶段处理

**阶段一：`__init__` 中的 `_filter_fields`**
- 动态添加 expand 对应的 serializer 字段到 `self.fields`
- 此时决定字段是否出现在输出中

**阶段二：`to_representation` 中的实际展开**
- 遍历 `self.expand`
- 对每个 expand 字段，用对应的 Lite Serializer 替换原始 FK id 值
- 特殊处理 `issue_attachments`：手动查询 `FileAsset` 模型

### 4.3 api 层 IssueSerializer 的特殊展开

api 层 [IssueSerializer.to_representation](file:///d:/fz/0508-3/solo-dogfeeding/code/199-plane/apps/api/plane/api/serializers/issue.py#L290-L320) 自己实现了 assignees/labels 的展开逻辑：

```python
def to_representation(self, instance):
    data = super().to_representation(instance)
    if "assignees" in self.fields:
        if "assignees" in self.expand:
            data["assignees"] = UserLiteSerializer(..., many=True).data
        else:
            data["assignees"] = [str(a) for a in IssueAssignee.objects.filter(...)]
    if "labels" in self.fields:
        if "labels" in self.expand:
            data["labels"] = LabelSerializer(..., many=True).data
        else:
            data["labels"] = [str(l) for l in IssueLabel.objects.filter(...)]
```

**此处有类型差异风险**：assignees/labels 在未展开时返回 `string[]`（UUID 列表），展开时返回 `object[]`。前端类型 `TBaseIssue` 定义了 `assignee_ids: string[]` 和 `label_ids: string[]`，但 api 层序列化器用的是 `assignees`/`labels` 字段名而非 `assignee_ids`/`label_ids`。

---

## 五、列表与详情视图的字段差异

### 5.1 app 层 Issue 的序列化器层级

```
IssueFlatSerializer        → 极简：id, name, description_json, description_html, priority, start_date, target_date, sequence_id, sort_order, is_draft
IssueLiteSerializer        → 轻量：id, sequence_id, project_id
IssueSerializer            → 列表级：+ state_id, estimate_point, parent_id, cycle_id, module_ids, label_ids, assignee_ids, sub_issues_count, attachment_count, link_count, created_at, updated_at, created_by, updated_by, is_draft, archived_at
IssueDetailSerializer      → 详情级：= IssueSerializer + description_html, is_subscribed, is_intake
IssuePublicSerializer      → 公开级：+ state_detail, project_detail, reactions, votes
```

### 5.2 列表接口字段

app 层 `IssueViewSet.list` ([base.py#L254](file:///d:/fz/0508-3/solo-dogfeeding/code/199-plane/apps/api/plane/app/views/issue/base.py#L254)) 在无 expand/fields 时通过 `issue_on_results` → `.values()` 返回，包含 `state__group` 字段但**不包含** `type_id`。

api 层 `IssueListCreateAPIEndpoint.get` 通过 `IssueSerializer(issues, many=True, fields=self.fields, expand=self.expand)` 返回。

### 5.3 详情接口字段

app 层 `IssueViewSet.retrieve` ([base.py#L612](file:///d:/fz/0508-3/solo-dogfeeding/code/199-plane/apps/api/plane/app/views/issue/base.py#L612)) 使用 `IssueDetailSerializer(issue, expand=self.expand)`，额外包含：

| 字段 | 列表 | 详情 |
|------|------|------|
| description_html | ❌ | ✅ |
| is_subscribed | ❌ | ✅ (数据库 annotate) |
| is_intake | ❌ | ✅ (数据库 annotate) |
| issue_reactions | ❌ | ✅ (prefetch_related, expand 时) |
| issue_link | ❌ | ✅ (prefetch_related, expand 时) |
| issue_attachments | ❌ | ✅ (expand 时手动查询) |

### 5.4 IssueListDetailSerializer — 列表手动构造器

[app/serializers/issue.py#L814-L914](file:///d:/fz/0508-3/solo-dogfeeding/code/199-plane/apps/api/plane/app/serializers/issue.py#L814-L914) 中有一个 `IssueListDetailSerializer(serializers.Serializer)`——不继承任何 ModelSerializer，直接在 `to_representation` 中手写所有字段映射：

```python
def to_representation(self, instance):
    data = {
        "id": instance.id,
        "name": instance.name,
        "estimate_point": instance.estimate_point_id,  # ← 注意：手动取 _id
        "created_by": instance.created_by_id,          # ← 手动取 _id
        ...
    }
```

这个序列化器将 FK 字段的 id 值手动映射为前端契约所需的格式，避开了 DRF 默认的 FK → nested object 行为。

---

## 六、前后端契约对齐的代码路径

### 6.1 前端类型层级

定义在 [packages/types/src/issues/issue.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/199-plane/packages/types/src/issues/issue.ts)：

```typescript
type TBaseIssue = {
  id: string;
  sequence_id: number;
  name: string;
  sort_order: number;
  state_id: string | null;
  priority: TIssuePriorities | null;
  label_ids: string[];
  assignee_ids: string[];
  estimate_point: string | null;
  sub_issues_count: number;
  attachment_count: number;
  link_count: number;
  project_id: string | null;
  parent_id: string | null;
  cycle_id: string | null;
  module_ids: string[] | null;
  type_id: string | null;
  created_at: string;
  updated_at: string;
  start_date: string | null;
  target_date: string | null;
  completed_at: string | null;
  archived_at: string | null;
  created_by: string;
  updated_by: string;
  is_draft: boolean;
  is_epic?: boolean;
  is_intake?: boolean;
};

type TIssue = TBaseIssue & {
  description_html?: string;
  is_subscribed?: boolean;
  parent?: Partial<TBaseIssue>;
  issue_reactions?: TIssueReaction[];
  issue_attachments?: TIssueAttachment[];
  issue_link?: TIssueLink[];
  issue_relation?: IssueRelation[];
  issue_related?: IssueRelation[];
};
```

### 6.2 已知的契约不一致

| 字段 | 后端 app 层 | 后端 api 层 | 前端 TBaseIssue | 风险 |
|------|-----------|-----------|----------------|------|
| assignees / assignee_ids | `assignee_ids: string[]` (IssueSerializer) | `assignees: string[]` (展开时 `object[]`) | `assignee_ids: string[]` | **api 层字段名不一致** |
| labels / label_ids | `label_ids: string[]` (IssueSerializer) | `labels: string[]` (展开时 `object[]`) | `label_ids: string[]` | **api 层字段名不一致** |
| type_id | 不在 IssueSerializer.fields | 在 IssueSerializer (api) via `type_id` | `type_id: string \| null` | **app 层缺失** |
| estimate_point | `estimate_point` (FK id, DynamicBaseSerializer) | `estimate_point` (FK id) | `estimate_point: string \| null` | 字段名一致但语义模糊 |
| created_by | `created_by: string` (User PK) | `created_by: string` (User PK) | `created_by: string` | 一致 |
| state__group | 仅 grouper 返回 | 不返回 | `state__group` 在 TIssue 上可选 | 仅分组列表可用 |
| is_epic | 不在 app IssueSerializer | 不在 api IssueSerializer | `is_epic?: boolean` | **前端定义了但后端不返回** |
| description_html | 仅 IssueDetailSerializer | IssueSerializer.exclude 排除了 description_json/stripped | `description_html?: string` | 仅详情视图返回 |
| issue_reactions | 仅 IssueDetailSerializer + expand | 不在 IssueSerializer | `issue_reactions?: TIssueReaction[]` | 仅详情 + expand |
| issue_attachments | 仅 expand (手动查询 FileAsset) | 不在 IssueSerializer | `issue_attachments?: TIssueAttachment[]` | 仅 expand |

### 6.3 Lite Serializer 与前端类型的对齐

| Serializer | 后端字段 | 前端类型 | 一致性 |
|-----------|---------|---------|-------|
| UserLiteSerializer (app) | id, first_name, last_name, avatar, avatar_url, is_bot, display_name | IUserLite: id, first_name, last_name, avatar_url, display_name, is_bot + email?, joining_date? | **后端缺少 email，前端多 email/joining_date** |
| UserLiteSerializer (api) | id, first_name, last_name, email, avatar, avatar_url, display_name | IUserLite | **api 有 email，app 没有** |
| StateLiteSerializer | id, name, color, group | — (前端无独立 StateLite 类型) | 前端用完整 State 类型 |
| LabelSerializer (app) | parent, name, color, id, project_id, workspace_id, sort_order | — | — |
| LabelLiteSerializer | id, name, color | — | — |
| IssueLiteSerializer | id, sequence_id, project_id | — | 嵌套展开用 |

---

## 七、字段新增/重命名时的传导路径

### 7.1 新增字段的传导

当后端新增一个模型字段时，传导链路如下：

```
1. Django Model 新增字段
   ↓
2. 数据库迁移 (makemigrations / migrate)
   ↓
3. 序列化器更新 (需要手动同步到多个位置)
   ├── app/serializers/issue.py  → IssueSerializer.Meta.fields
   ├── app/serializers/issue.py  → IssueDetailSerializer.Meta.fields
   ├── app/serializers/issue.py  → IssueListDetailSerializer.to_representation()
   ├── app/views/issue/base.py   → .values() 字段列表
   ├── app/utils/grouper.py      → issue_on_results required_fields
   ├── api/serializers/issue.py  → IssueSerializer.Meta.fields/exclude
   └── api/serializers/base.py   → expansion mapper (如需展开)
   ↓
4. 前端类型更新 (需要手动同步)
   ├── packages/types/src/issues/issue.ts  → TBaseIssue / TIssue
   ├── packages/types/src/issues/issue_attachment.ts  (如涉及)
   └── packages/types/src/issues/issue_relation.ts    (如涉及)
   ↓
5. 前端视图/组件消费 (无编译时保障)
```

**核心问题**：步骤 3→4 之间**没有自动同步机制**。后端序列化器新增了字段，前端类型不会自动更新。前端用 TypeScript strict mode，但后端返回的 JSON 不受 TypeScript 编译检查。

### 7.2 字段重命名的传导

假设将 `estimate_point` 重命名为 `story_point`：

1. **后端 Model** → 新字段名
2. **后端序列化器** → 必须同步更新所有 Meta.fields 列表
3. **手写 .values() 列表** → 容易遗漏，导致列表接口返回旧字段名
4. **grouper.py** → 容易遗漏
5. **前端类型** → 字段名必须同步修改
6. **前端所有引用点** → 所有 `issue.estimate_point` 的地方都会编译报错

**危险场景**：如果只改了 Model 和序列化器，没改 `.values()` 或 grouper，则列表接口返回的字段名与详情接口不同，前端部分场景正常、部分场景崩溃。

### 7.3 类型变更的传导

当后端字段类型改变（如 `IntegerField` → `CharField`）：

1. **后端序列化器** → DRF 的 serializer field 类型自动跟随 model field
2. **前端类型** → 不会自动变更。前端 `number` 还是 `string` 由手动维护
3. **运行时崩溃** → 前端做 `===` 比较或数学运算时，`"3"` vs `3` 导致判断失败

---

## 八、崩溃风险的典型路径

### 8.1 场景：后端将 FK 从 UUID 改为嵌套对象

如果后端新增了 expand 且前端未处理：

```
GET /api/workspaces/{slug}/projects/{id}/issues/?expand=state
```

返回的 `state` 字段从 `string` (UUID) 变为 `object` (StateLiteSerializer data)。前端如果做 `issue.state_id === someUUID`，改为 `issue.state` 后类型变成 object，比较失败。

### 8.2 场景：app 层与 api 层返回不同字段名

- app 层 `IssueSerializer` 返回 `assignee_ids: string[]`
- api 层 `IssueSerializer` 返回 `assignees: string[]`（未展开）或 `object[]`（展开）

前端 `TBaseIssue` 只定义了 `assignee_ids`。如果前端误用 api 层接口，取 `data.assignee_ids` 会得到 `undefined`。

### 8.3 场景：grouper 返回额外字段

`issue_on_results` 的 `required_fields` 包含 `state__group`，这个字段不在任何 serializer 的 Meta.fields 中。前端 `TIssue` 类型中有可选的 `state__group?: TStateGroups | null`，但如果后端重命名了 State 的 group 字段，此处会静默返回 None 或报错。

### 8.4 场景：is_epic 前端定义但后端不返回

前端 `TBaseIssue` 定义了 `is_epic?: boolean`，但 app 层 `IssueSerializer` 的 `Meta.fields` 不包含 `is_epic`。api 层的 `RelatedIssueSerializer` 有 `is_epic`（source=`issue.type.is_epic`），但主序列化器没有。前端代码如果依赖 `is_epic` 做判断，在列表视图中永远拿到 `undefined`。

---

## 九、契约同步的改进建议

### 9.1 单一字段定义源

将手写的三处字段列表统一为一处：

- `IssueSerializer.Meta.fields` → 权威源
- `.values()` 调用 → 从 serializer 自动推导
- `grouper.required_fields` → 从 serializer 自动推导

### 9.2 app/api 序列化器字段名统一

建议将 api 层 `IssueSerializer` 的 `assignees`/`labels` 改为 `assignee_ids`/`label_ids`，与 app 层和前端 `TBaseIssue` 对齐。

### 9.3 前端类型与后端 OpenAPI Schema 自动校验

api 层已使用 `drf-spectacular` 生成 OpenAPI schema，可通过 CI 流程将 schema 与前端 TypeScript 类型做 diff 校验，在 CI 阶段发现字段不一致。

### 9.4 字段变更检查清单

每次后端字段变更时，必须检查以下位置：

| 检查项 | 文件路径 |
|-------|---------|
| Model 字段 | `plane/db/models/` |
| app 序列化器 | [app/serializers/issue.py](file:///d:/fz/0508-3/solo-dogfeeding/code/199-plane/apps/api/plane/app/serializers/issue.py) |
| api 序列化器 | [api/serializers/issue.py](file:///d:/fz/0508-3/solo-dogfeeding/code/199-plane/apps/api/plane/api/serializers/issue.py) |
| expansion mapper | [app/serializers/base.py#L74](file:///d:/fz/0508-3/solo-dogfeeding/code/199-plane/apps/api/plane/app/serializers/base.py#L74-L96) / [api/serializers/base.py#L91](file:///d:/fz/0508-3/solo-dogfeeding/code/199-plane/apps/api/plane/api/serializers/base.py#L91-L106) |
| .values() 字段列表 | [app/views/issue/base.py#L163](file:///d:/fz/0508-3/solo-dogfeeding/code/199-plane/apps/api/plane/app/views/issue/base.py#L163-L190) |
| grouper required_fields | [utils/grouper.py#L106](file:///d:/fz/0508-3/solo-dogfeeding/code/199-plane/apps/api/plane/plane/utils/grouper.py#L106-L130) |
| IssueListDetailSerializer | [app/serializers/issue.py#L831](file:///d:/fz/0508-3/solo-dogfeeding/code/199-plane/apps/api/plane/app/serializers/issue.py#L831-L860) |
| 前端 TBaseIssue | [packages/types/src/issues/issue.ts#L45](file:///d:/fz/0508-3/solo-dogfeeding/code/199-plane/packages/types/src/issues/issue.ts#L45-L80) |
| 前端 TIssue | [packages/types/src/issues/issue.ts#L90](file:///d:/fz/0508-3/solo-dogfeeding/code/199-plane/packages/types/src/issues/issue.ts#L90-L104) |
| 前端 IUserLite | [packages/types/src/users.ts#L25](file:///d:/fz/0508-3/solo-dogfeeding/code/199-plane/packages/types/src/users.ts#L25-L34) |
| **前端 addIssueToStore** | [store/issue/issue-details/issue.store.ts#L142](file:///d:/fz/0508-3/solo-dogfeeding/code/199-plane/apps/web/core/store/issue/issue-details/issue.store.ts#L142-L179) |
| **前端 ISSUE_ORDERBY_KEY** | [store/issue/helpers/base-issues.store.ts#L145](file:///d:/fz/0508-3/solo-dogfeeding/code/199-plane/apps/web/core/store/issue/helpers/base-issues.store.ts#L145-L175) |
| **前端 ISSUE_GROUP_BY_KEY** | [store/issue/helpers/base-issues.store.ts#L116](file:///d:/fz/0508-3/solo-dogfeeding/code/199-plane/apps/web/core/store/issue/helpers/base-issues.store.ts#L116-L128) |
| **前端 getFilterParams expand 逻辑** | [store/issue/helpers/issue-filter-helper.store.ts#L92](file:///d:/fz/0508-3/solo-dogfeeding/code/199-plane/apps/web/core/store/issue/helpers/issue-filter-helper.store.ts#L92-L126) |
| **前端 IssueService getIssuesFromServer 路由** | [services/issue/issue.service.ts#L40](file:///d:/fz/0508-3/solo-dogfeeding/code/199-plane/apps/web/core/services/issue/issue.service.ts#L40-L61) |

---

## 十一、前端 SDK 与服务层承接路径

### 11.1 服务层总览

前端 Issue 数据流经三层代码：

```
组件/hooks
   ↓ 调用
Store 层 (MobX)               ← 状态管理、查询参数组装
   ↓ 委托
Service 层 (IssueService)     ← HTTP 请求、URL 路由
   ↓ axios
后端 API
```

核心服务类为 [IssueService](file:///d:/fz/0508-3/solo-dogfeeding/code/199-plane/apps/web/core/services/issue/issue.service.ts)，继承自 [APIService](file:///d:/fz/0508-3/solo-dogfeeding/code/199-plane/packages/services/src/api.service.ts)（axios 封装）。

### 11.2 EIssueServiceType — 服务类型路由

定义在 [packages/types/src/issues/issue.ts#L23-L27](file:///d:/fz/0508-3/solo-dogfeeding/code/199-plane/packages/types/src/issues/issue.ts#L23-L27)：

```typescript
export enum EIssueServiceType {
  ISSUES = "issues",
  EPICS = "epics",
  WORK_ITEMS = "work-items",
}
```

`IssueService` 构造时接收 `serviceType`，拼接为 URL 路径段：

```typescript
constructor(serviceType: TIssueServiceType = EIssueServiceType.ISSUES) {
    this.serviceType = serviceType;
}
// 列表请求 → /api/workspaces/{slug}/projects/{id}/${this.serviceType}/
// 详情请求 → /api/workspaces/{slug}/projects/{id}/${this.serviceType}/{issueId}/
```

**serviceType 到后端 URL 的映射**：

| serviceType | 列表 URL 后缀 | 详情 URL 后缀 |
|------------|--------------|-------------|
| `issues` | `/issues/` | `/issues/{id}/` |
| `epics` | `/epics/` | `/epics/{id}/` |
| `work-items` | `/work-items/` | `/work-items/{id}/` |

### 11.3 列表请求的两条路径

[IssueService.getIssuesFromServer](file:///d:/fz/0508-3/solo-dogfeeding/code/199-plane/apps/web/core/services/issue/issue.service.ts#L40-L61) 根据 `expand` 参数动态选择端点：

```typescript
async getIssuesFromServer(workspaceSlug, projectId, queries?, config = {}) {
    const path =
      (queries.expand as string)?.includes("issue_relation") && !queries.group_by
        ? `/api/workspaces/${workspaceSlug}/projects/${projectId}/${this.serviceType}-detail/`
        : `/api/workspaces/${workspaceSlug}/projects/${projectId}/${this.serviceType}/`;
    return this.get(path, { params: queries }, config);
}
```

**关键逻辑**：当 `expand` 包含 `issue_relation` 且无 `group_by` 时，路由到 `-detail/` 端点（[IssueDetailEndpoint](file:///d:/fz/0508-3/solo-dogfeeding/code/199-plane/apps/api/plane/app/views/issue/base.py#L963)），该端点使用 `IssueListDetailSerializer`；否则路由到普通列表端点，使用 `.values()` 或 `IssueSerializer`。

同样，[WorkspaceService.getViewIssues](file:///d:/fz/0508-3/solo-dogfeeding/code/199-plane/apps/web/core/services/workspace.service.ts#L272-L275) 有相同逻辑：

```typescript
async getViewIssues(workspaceSlug, params, config = {}) {
    const path = params.expand?.includes("issue_relation")
      ? `/api/workspaces/${workspaceSlug}/issues-detail/`
      : `/api/workspaces/${workspaceSlug}/issues/`;
    ...
}
```

### 11.4 详情请求的 expand 参数

[IssueStore.fetchIssue](file:///d:/fz/0508-3/solo-dogfeeding/code/199-plane/apps/web/core/store/issue/issue-details/issue.store.ts#L87-L140) 固定传入：

```typescript
fetchIssue = async (workspaceSlug, projectId, issueId) => {
    const query = {
      expand: "issue_reactions,issue_attachments,issue_link,parent",
    };
    const issue = await this.issueService.retrieve(workspaceSlug, projectId, issueId, query);
    ...
};
```

[fetchIssueWithIdentifier](file:///d:/fz/0508-3/solo-dogfeeding/code/199-plane/apps/web/core/store/issue/issue-details/issue.store.ts#L270-L273) 使用相同的 expand 参数。

### 11.5 列表请求的 expand 参数

[IssueFilterHelperStore.computedFilteredParams](file:///d:/fz/0508-3/solo-dogfeeding/code/199-plane/apps/web/core/store/issue/helpers/issue-filter-helper.store.ts#L92-L126) 在 Gantt 布局下自动添加 expand：

```typescript
if (ENABLE_ISSUE_DEPENDENCIES && displayFilters?.layout === EIssueLayoutTypes.GANTT)
    issueFiltersParams["expand"] = "issue_relation,issue_related";
```

其他布局（Kanban、List、Calendar）**不传 expand 参数**，因此走 `.values()` 快速路径。

### 11.6 Inbox 请求的 expand 参数

[InboxIssueService.retrieve](file:///d:/fz/0508-3/solo-dogfeeding/code/199-plane/apps/web/core/services/inbox/inbox-issue.service.ts#L30-L33) 固定传 `expand=issue_inbox`：

```typescript
async retrieve(workspaceSlug, projectId, inboxIssueId) {
    return this.get(
      `/api/workspaces/${workspaceSlug}/projects/${projectId}/inbox-issues/${inboxIssueId}/?expand=issue_inbox`
    );
}
```

---

## 十二、前端 Store 层对 TIssue/TIssuesResponse 的消费链路

### 12.1 Store 层级结构

```
IssueRootStore (root.store.ts)
├── serviceType: TIssueServiceType              ← 决定 URL 路由
├── issues: IssueStore                          ← 全局 issuesMap (TIssue 字典)
│
├── issueDetail: IssueDetail (EPICS=epicDetail) ← 详情专用
│   ├── issue: IssueStore (fetchIssue)          ← 详情 fetch + expand
│   ├── relation: IssueRelationStore
│   ├── link: IssueLinkStore
│   ├── attachment: IssueAttachmentStore
│   ├── reaction: IssueReactionStore
│   ├── subscription: IssueSubscriptionStore
│   ├── subIssues: SubIssuesStore
│   ├── comment: IssueCommentStore
│   └── activity: IssueActivityStore
│
├── projectIssues: ProjectIssues                ← 项目级列表
│   └── issueFilterStore: ProjectIssuesFilter   ← getFilterParams()
├── cycleIssues: CycleIssues                    ← Cycle 级列表
├── moduleIssues: ModuleIssues                  ← Module 级列表
├── projectViewIssues: ProjectViewIssues        ← 视图级列表
├── workspaceIssues: WorkspaceIssues            ← 工作区级列表
├── profileIssues: ProfileIssues                ← 用户档案级列表
├── archivedIssues: ArchivedIssues              ← 归档列表
└── projectEpics: ProjectEpics                  ← Epic 列表
```

### 12.2 TIssuesResponse 的处理流程

[BaseIssuesStore.processIssueResponse](file:///d:/fz/0508-3/solo-dogfeeding/code/199-plane/apps/web/core/store/issue/helpers/base-issues.store.ts#L1257-L1343) 解析 `TIssuesResponse`：

```
TIssuesResponse
  ├── results: TIssueResponseResults
  │     ├── TBaseIssue[]          → 无分组
  │     ├── { [groupId]: { results: TBaseIssue[], total_results } }  → 单层分组
  │     └── { [groupId]: { [subGroupId]: { results: TBaseIssue[], total_results } } } → 双层分组
  ├── total_count
  ├── next_cursor / prev_cursor
  └── next_page_results / prev_page_results
```

**关键类型约束**：`TIssueResponseResults` 中 results 的元素类型是 `TBaseIssue`，而非 `TIssue`。这意味着 TypeScript 编译器不知道 expand 后返回的嵌套字段（如 `parent`、`issue_reactions`）。实际运行时返回的是 `TIssue`（包含展开字段），但类型系统只保证了 `TBaseIssue` 的字段存在。

### 12.3 列表数据入库路径

```
ProjectIssues.fetchIssues()
  → issueFilterStore.getFilterParams()       ← 组装查询参数（含 expand）
  → issueService.getIssues()                 ← HTTP 请求
  → onfetchIssues(response, options)         ← 处理响应
      → processIssueResponse(response)       ← 从 results 提取 TIssue[]
      → rootIssueStore.issues.addIssue(list) ← 写入全局 issuesMap
      → updateGroupedIssueIds()              ← 更新分组 ID 列表
      → issueDetail.relation.extractRelationsFromIssues()  ← 提取关系
```

[IssueStore.addIssue](file:///d:/fz/0508-3/solo-dogfeeding/code/199-plane/apps/web/core/store/issue/issue.store.ts#L61-L75) 对已存在的 issue 做 merge 更新：

```typescript
addIssue = (issues: TIssue[]) => {
    issues.forEach((issue) => {
        if (!this.issuesMap[issue.id]) set(this.issuesMap, issue.id, issue);
        else update(this.issuesMap, issue.id, (prevIssue) => ({ ...prevIssue, ...issue }));
    });
};
```

**这意味着列表接口返回的 `TBaseIssue` 数据会覆盖详情接口已写入的 `TIssue` 展开字段（如 `issue_reactions`），如果列表接口不返回这些字段，展开数据会丢失。** 但由于 `update` 使用 spread，`undefined` 值不会覆盖已有值——只有当列表接口显式返回了某个字段的 `null` 时才会丢失。

### 12.4 详情数据入库路径

```
IssueDetail.issue.fetchIssue()
  → issueService.retrieve(id, { expand: "issue_reactions,issue_attachments,issue_link,parent" })
  → addIssueToStore(issue)                   ← 手动构造 TIssue payload
  → rootIssueStore.issues.addIssue([payload])
  → 分发到各子 store:
      ├── issue_reactions → addReactions()
      ├── issue_link → addLinks()
      ├── issue_attachments → addAttachments()
      ├── is_subscribed → addSubscription()
      └── 子请求: fetchActivities, fetchComments, fetchSubIssues, fetchRelations
```

[addIssueToStore](file:///d:/fz/0508-3/solo-dogfeeding/code/199-plane/apps/web/core/store/issue/issue-details/issue.store.ts#L142-L179) 手动提取 TIssue 字段：

```typescript
addIssueToStore = (issue: TIssue) => {
    const issuePayload: TIssue = {
      id: issue?.id,
      sequence_id: issue?.sequence_id,
      name: issue?.name,
      description_html: issue?.description_html,
      sort_order: issue?.sort_order,
      state_id: issue?.state_id,
      priority: issue?.priority,
      label_ids: issue?.label_ids,
      assignee_ids: issue?.assignee_ids,
      estimate_point: issue?.estimate_point,
      sub_issues_count: issue?.sub_issues_count,
      attachment_count: issue?.attachment_count,
      link_count: issue?.link_count,
      project_id: issue?.project_id,
      parent_id: issue?.parent_id,
      cycle_id: issue?.cycle_id,
      module_ids: issue?.module_ids,
      type_id: issue?.type_id,
      created_at: issue?.created_at,
      updated_at: issue?.updated_at,
      start_date: issue?.start_date,
      target_date: issue?.target_date,
      completed_at: issue?.completed_at,
      archived_at: issue?.archived_at,
      created_by: issue?.created_by,
      updated_by: issue?.updated_by,
      is_draft: issue?.is_draft,
      is_subscribed: issue?.is_subscribed,
      is_epic: issue?.is_epic,
    };
    ...
};
```

**注意**：`addIssueToStore` 没有提取 `parent`、`issue_reactions`、`issue_attachments`、`issue_link` 等展开字段到 `issuePayload`。这些数据是通过后续的 `addReactions`、`addLinks`、`addAttachments` 调用分别存入子 store，而非合并到 `issuesMap` 中的 issue 对象。

**另一个注意**：`is_epic` 字段在此被手动写入 store，但来源是 [retrieve 方法](file:///d:/fz/0508-3/solo-dogfeeding/code/199-plane/apps/web/core/services/issue/issue.service.ts#L117-L122) 中的客户端注入：

```typescript
if (response.data && this.serviceType === EIssueServiceType.EPICS) {
    response.data.is_epic = true;
}
```

`is_epic` 不是后端返回的字段，而是前端在 Epic serviceType 下手动设为 `true`。这解释了为什么 `TBaseIssue` 定义了 `is_epic?` 但后端序列化器不返回它。

### 12.5 TIssueParams — 查询参数类型

定义在 [packages/types/src/view-props.ts#L63-L89](file:///d:/fz/0508-3/solo-dogfeeding/code/199-plane/packages/types/src/view-props.ts#L63-L89)：

```typescript
export type TIssueParams =
  | "priority" | "state_group" | "state" | "assignees" | "mentions"
  | "created_by" | "subscriber" | "labels" | "cycle" | "module"
  | "start_date" | "target_date" | "project" | "team_project"
  | "group_by" | "sub_group_by" | "order_by" | "type" | "sub_issue"
  | "show_empty_groups" | "cursor" | "per_page" | "issue_type"
  | "layout" | "expand" | "filters";
```

`expand` 和 `filters` 都是合法的查询参数，在 `computedFilteredParams` 中被设置。

---

## 十三、前端 expand 请求字段名与后端 expansion mapper 的一致性核对

### 13.1 前端发出的 expand 值汇总

| 发起位置 | expand 值 | 请求路径 |
|---------|----------|---------|
| 详情 fetchIssue | `issue_reactions,issue_attachments,issue_link,parent` | `/issues/{id}/` |
| 详情 fetchIssueWithIdentifier | `issue_reactions,issue_attachments,issue_link,parent` | `/work-items/{identifier}/` |
| 列表 Gantt 布局 | `issue_relation,issue_related` | `/issues/` 或 `/issues-detail/` |
| Inbox retrieve | `issue_inbox` | `/inbox-issues/{id}/` |

### 13.2 后端 expansion mapper 支持的 expand 值

app 层 mapper（[app/serializers/base.py#L74-L96](file:///d:/fz/0508-3/solo-dogfeeding/code/199-plane/apps/api/plane/app/serializers/base.py#L74-L96)）支持的完整列表：

`user`, `workspace`, `project`, `state`, `assignees`, `labels`, `parent`, `sub_issues`, `issue_cycle`, `issue_relation`, `issue_reactions`, `issue_link`, `issue_attachment`, `default_assignee`, `project_lead`, `created_by`, `actor`, `owned_by`, `updated_by`, `issue_related`

### 13.3 逐项核对

| 前端 expand 值 | 后端 mapper 是否支持 | 后端 view 是否 prefetch | 一致性 |
|---------------|-------------------|----------------------|-------|
| `issue_reactions` | ✅ IssueReactionLiteSerializer | ✅ retrieve 有 prefetch_related | ✅ 一致 |
| `issue_attachments` | ✅ IssueAttachmentLiteSerializer | ✅ to_representation 手动查 FileAsset | ✅ 一致 |
| `issue_link` | ✅ IssueLinkLiteSerializer | ✅ retrieve 有 prefetch_related | ✅ 一致 |
| `parent` | ✅ IssueLiteSerializer | ✅ retrieve 有 select_related | ✅ 一致 |
| `issue_relation` | ✅ IssueRelationSerializer | ✅ IssueDetailEndpoint 有 prefetch_related | ✅ 一致 |
| `issue_related` | ✅ RelatedIssueSerializer | ✅ IssueDetailEndpoint 有 prefetch_related | ✅ 一致 |
| `issue_inbox` | ❌ **不在 mapper 中** | N/A (Inbox 有独立序列化) | ⚠️ 见下文 |

### 13.4 `issue_inbox` 的特殊情况

前端 `InboxIssueService.retrieve` 发送 `expand=issue_inbox`，但 app 层 expansion mapper 中**没有** `issue_inbox` 条目。这是因为 inbox 请求走的是独立的 `InboxIssueViewSet`，其序列化器是 `InboxIssueSerializer`，不是通用的 `IssueSerializer`。`issue_inbox` 在 `InboxIssueSerializer` 内部处理，不走通用 expansion mapper。

**风险**：如果有人误将 `expand=issue_inbox` 传给普通 issue 端点，`DynamicBaseSerializer._filter_fields` 不会识别这个 expand 值，会静默忽略。

### 13.5 `issue_attachments` vs `issue_attachment` 命名差异

前端请求 `expand=issue_attachments`（复数），后端 mapper 注册的是 `issue_attachment`（单数）。但在 `DynamicBaseSerializer.to_representation` 中有特殊处理：

```python
if "issue_attachments" in self.expand:
    # 手动查询 FileAsset
```

所以 mapper 中的 key 是 `issue_attachment`（用于 `_filter_fields` 阶段添加字段），但 `to_representation` 阶段检查的是 `issue_attachments`（复数）。**两者同时存在且必须同时修改**，否则会出现字段被添加但数据不被展开的情况。

### 13.6 列表接口不支持 expand 的路径

当列表请求**不含** expand 时（普通 Kanban/List/Calendar 布局），请求走 `IssueViewSet.list` → `issue_on_results` → `.values()` 路径，**完全绕过序列化器和 expansion mapper**。此时前端传入的任何 expand 参数都会被忽略（因为 `.values()` 不处理 expand）。

**只有在 Gantt 布局或包含 `issue_relation` 的 expand 请求时**，才会路由到 `IssueDetailEndpoint`，该端点使用 `IssueListDetailSerializer` 处理 expand。

---

## 十四、类型变更进入列表和详情视图的完整传导路径

### 14.1 新增字段的完整传导

假设后端新增 `story_point` 字段（Model → Serializer → Frontend）：

```
1. Django Model: Issue.story_point = models.IntegerField(null=True)
   ↓
2. 后端序列化器（5 处手动同步）:
   ├── IssueSerializer.Meta.fields += ("story_point",)
   ├── IssueDetailSerializer.Meta.fields += ("story_point",)
   ├── IssueListDetailSerializer.to_representation() += "story_point": instance.story_point
   ├── IssueViewSet.list .values() += "story_point"
   └── grouper.issue_on_results required_fields += "story_point"
   ↓
3. 后端 expansion mapper（可选，如需展开）:
   └── base.py mapper += "story_point": StoryPointSerializer
   ↓
4. 前端类型更新:
   └── TBaseIssue += story_point: number | null
   ↓
5. 前端 Store 层（关键！两处手动同步）:
   ├── IssueDetail.addIssueToStore() += story_point: issue?.story_point
   └── BaseIssuesStore.ISSUE_ORDERBY_KEY += story_point: "story_point"（如需排序）
   ↓
6. 前端视图组件:
   └── 各显示/编辑组件引用 issue.story_point
```

**步骤 5 是前次分析中遗漏的关键环节**：`addIssueToStore` 手动枚举了 TIssue 的每个字段，新增字段必须同步添加到此方法，否则详情接口返回的数据写入 `issuesMap` 后该字段会丢失。

### 14.2 字段类型变更的传导

假设后端将 `estimate_point` 从 `ForeignKey` 改为 `IntegerField`：

```
1. Django Model: estimate_point = models.IntegerField(null=True)  (原 ForeignKey → Point)
   ↓
2. 后端序列列化器:
   ├── IssueSerializer: estimate_point 自动从 PK 变为 int
   ├── IssueListDetailSerializer: estimate_point_id 不再存在（手动取 _id 报错）
   └── DynamicBaseSerializer: expansion mapper 中 estimate_point 展开逻辑失效
   ↓
3. 前端:
   ├── TBaseIssue.estimate_point: string | null → number | null（需手动改）
   ├── ISSUE_ORDERBY_KEY: estimate_point__key 排序失效（后端字段不存在）
   └── ISSUE_FILTER_DEFAULT_DATA: 无影响
```

**`estimate_point` 的特殊风险**：当前 `estimate_point` 在 Model 中是 FK 到 `Estimate` 模型。`IssueSerializer` 中 `estimate_point = serializers.PrimaryKeyRelatedField(read_only=True)` 返回 UUID string。如果后端改为返回整数 key，前端 `TBaseIssue.estimate_point: string | null` 就会收到 `number`，类型不匹配但运行时不报错（TypeScript 类型只在编译时检查）。

### 14.3 前端 is_epic 的特殊传导

`is_epic` 字段的传导路径与普通字段完全不同：

```
1. 后端: IssueSerializer 不包含 is_epic（不返回）
   ↓
2. IssueService.retrieve():
   ├── serviceType === EPICS → response.data.is_epic = true  (客户端注入)
   └── serviceType !== EPICS → 不设置 is_epic (undefined)
   ↓
3. IssueDetail.addIssueToStore():
   └── is_epic: issue?.is_epic  (存入 issuesMap)
   ↓
4. 组件消费: issue.is_epic === true 判断是否为 Epic
```

**风险**：如果后端决定新增 `is_epic` 字段到序列化器，前端客户端注入逻辑仍会执行，可能导致 `is_epic` 被覆盖为 `true`，即使后端返回 `false`。

---

## 十五、关键文件索引

| 文件 | 作用 |
|------|------|
| [app/serializers/base.py](file:///d:/fz/0508-3/solo-dogfeeding/code/199-plane/apps/api/plane/app/serializers/base.py) | DynamicBaseSerializer + expansion mapper |
| [api/serializers/base.py](file:///d:/fz/0508-3/solo-dogfeeding/code/199-plane/apps/api/plane/api/serializers/base.py) | api BaseSerializer + _filter_fields + expansion |
| [app/serializers/issue.py](file:///d:/fz/0508-3/solo-dogfeeding/code/199-plane/apps/api/plane/app/serializers/issue.py) | app 层所有 Issue 序列化器 |
| [api/serializers/issue.py](file:///d:/fz/0508-3/solo-dogfeeding/code/199-plane/apps/api/plane/api/serializers/issue.py) | api 层所有 Issue 序列化器 |
| [app/views/base.py](file:///d:/fz/0508-3/solo-dogfeeding/code/199-plane/apps/api/plane/app/views/base.py) | fields/expand 参数解析 |
| [app/views/issue/base.py](file:///d:/fz/0508-3/solo-dogfeeding/code/199-plane/apps/api/plane/app/views/issue/base.py) | Issue 列表/详情视图 |
| [api/views/issue.py](file:///d:/fz/0508-3/solo-dogfeeding/code/199-plane/apps/api/plane/api/views/issue.py) | api 层 Issue 端点 |
| [utils/grouper.py](file:///d:/fz/0508-3/solo-dogfeeding/code/199-plane/apps/api/plane/plane/utils/grouper.py) | issue_on_results 手写字段列表 |
| [packages/types/src/issues/issue.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/199-plane/packages/types/src/issues/issue.ts) | 前端 TBaseIssue / TIssue / EIssueServiceType 类型 |
| [packages/types/src/view-props.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/199-plane/packages/types/src/view-props.ts) | 前端 TIssueParams 查询参数类型 |
| [packages/types/src/users.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/199-plane/packages/types/src/users.ts) | 前端 IUserLite / IUser 类型 |
| [packages/types/src/issues/issue_relation.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/199-plane/packages/types/src/issues/issue_relation.ts) | 前端 TIssueRelation 类型 |
| [packages/types/src/issues/issue_attachment.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/199-plane/packages/types/src/issues/issue_attachment.ts) | 前端 TIssueAttachment 类型 |
| [packages/types/src/issues/issue_link.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/199-plane/packages/types/src/issues/issue_link.ts) | 前端 TIssueLink 类型 |
| [packages/types/src/issues/issue_reaction.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/199-plane/packages/types/src/issues/issue_reaction.ts) | 前端 TIssueReaction 类型 |
| [packages/services/src/api.service.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/199-plane/packages/services/src/api.service.ts) | 基础 HTTP 服务 (axios 封装) |
| [services/issue/issue.service.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/199-plane/apps/web/core/services/issue/issue.service.ts) | 前端 IssueService (列表/详情/CRUD) |
| [services/inbox/inbox-issue.service.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/199-plane/apps/web/core/services/inbox/inbox-issue.service.ts) | 前端 InboxIssueService |
| [store/issue/root.store.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/199-plane/apps/web/core/store/issue/root.store.ts) | IssueRootStore (顶层状态管理) |
| [store/issue/issue.store.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/199-plane/apps/web/core/store/issue/issue.store.ts) | 全局 issuesMap (TIssue 字典) |
| [store/issue/helpers/base-issues.store.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/199-plane/apps/web/core/store/issue/helpers/base-issues.store.ts) | BaseIssuesStore (列表逻辑 + processIssueResponse) |
| [store/issue/helpers/issue-filter-helper.store.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/199-plane/apps/web/core/store/issue/helpers/issue-filter-helper.store.ts) | 查询参数组装 + expand 逻辑 |
| [store/issue/project/issue.store.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/199-plane/apps/web/core/store/issue/project/issue.store.ts) | ProjectIssues (项目级列表) |
| [store/issue/project/filter.store.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/199-plane/apps/web/core/store/issue/project/filter.store.ts) | ProjectIssuesFilter (getFilterParams) |
| [store/issue/issue-details/issue.store.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/199-plane/apps/web/core/store/issue/issue-details/issue.store.ts) | IssueDetail.fetchIssue + addIssueToStore |
| [store/issue/issue-details/root.store.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/199-plane/apps/web/core/store/issue/issue-details/root.store.ts) | IssueDetail (详情子 store 组合) |
