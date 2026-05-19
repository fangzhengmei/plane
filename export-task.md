# 导出任务流程分析

## 概述

导出功能采用 **"异步任务 + 轮询拉取"** 的架构模式：用户发起导出请求后，API 立即返回，后台通过 Celery 异步处理任务，前端通过轮询获取任务状态和结果。

---

## 一、任务编排：请求分发流程

### 1.1 入口层 - API 视图

**文件**: `apps/api/plane/app/views/exporter/base.py`

```python
class ExportIssuesEndpoint(BaseAPIView):
    def post(self, request, slug):
        # 1. 参数校验（provider: csv/json/xlsx）
        # 2. 创建 ExporterHistory 记录（状态: queued）
        exporter = ExporterHistory.objects.create(
            workspace=workspace,
            project=project_ids,
            initiated_by=request.user,
            provider=provider,
            type="issue_exports",
        )
        # 3. 提交异步任务到 Celery
        issue_export_task.delay(
            provider=exporter.provider,
            workspace_id=workspace.id,
            project_ids=project_ids,
            token_id=exporter.token,
            multiple=multiple,
            slug=slug,
        )
        # 4. 立即返回响应
        return Response({"message": "Once the export is ready..."}, status=200)
```

**关键特性**:
- 非阻塞：API 不等待任务完成，创建记录后立即返回
- 任务标识：通过 `token` 字段关联 ExporterHistory 和后台任务
- 权限控制：仅 WORKSPACE 级别的 ADMIN/MEMBER 可发起

### 1.2 调度层 - Celery 任务

**文件**: `apps/api/plane/bgtasks/export_task.py`

```python
@shared_task
def issue_export_task(provider, workspace_id, project_ids, token_id, multiple, slug):
    # 后台异步执行的核心逻辑
    pass
```

**任务执行链路**:
```
API.post()
    ↓ create ExporterHistory (queued)
    ↓ issue_export_task.delay() ────→ Celery Queue
                                          ↓
                                    Worker  pickup
                                          ↓
                                    issue_export_task() 执行
```

---

## 二、状态推进：任务生命周期

### 2.1 状态模型

**文件**: `apps/api/plane/db/models/exporter.py`

```python
class ExporterHistory(BaseModel):
    status = models.CharField(
        max_length=50,
        choices=(
            ("queued", "Queued"),      # 已入队，等待执行
            ("processing", "Processing"),  # 正在处理
            ("completed", "Completed"),    # 处理完成
            ("failed", "Failed"),      # 处理失败
        ),
        default="queued",
    )
    token = models.CharField(max_length=255, default=generate_token, unique=True)
    url = models.URLField(max_length=800, blank=True, null=True)  # 下载链接
    reason = models.TextField(blank=True)  # 失败原因
```

### 2.2 状态流转图

```
  发起请求
     ↓
  [queued]  ← ExporterHistory.objects.create()
     ↓
  Celery 任务开始执行
     ↓
  [processing]  ← exporter_instance.status = "processing"
     ↓
  数据查询 → 序列化 → 格式化 → 打包ZIP → 上传S3
     ↓
  成功? ──┬── 是 → [completed] ← url = presigned_url
          └── 否 → [failed]    ← reason = 错误信息
```

### 2.3 状态变更关键点

| 阶段 | 代码位置 | 操作 |
|------|---------|------|
| 初始化 | `views/exporter/base.py:41-47` | `status="queued"` |
| 任务开始 | `bgtasks/export_task.py:143-145` | `status="processing"` |
| 上传成功 | `bgtasks/export_task.py:117-120` | `status="completed"`, `url=presigned_url` |
| 格式错误 | `bgtasks/export_task.py:197-200` | `status="failed"`, `reason=str(e)` |
| 异常捕获 | `bgtasks/export_task.py:221-224` | `status="failed"`, `reason=str(e)` |

---

## 三、结果文件回流：从后台到用户

### 3.1 数据处理流水线

**文件**: `apps/api/plane/bgtasks/export_task.py` + `apps/api/plane/utils/porters/`

```
QuerySet (Issue)
    ↓ select_related / prefetch_related (148-190行)
    ↓
DataExporter(IssueExportSerializer, format_type=provider)
    ↓ serialize() → List[Dict]
    ↓
Formatter.encode() → str/bytes
    ├─ CSVFormatter  → CSV 字符串
    ├─ JSONFormatter → JSON 字符串
    └─ XLSXFormatter → Excel 字节流
    ↓
create_zip_file() → ZIP 压缩包
    ↓
upload_to_s3() → S3/MinIO 存储
    ↓
generate_presigned_url() → 7天有效期下载链接
    ↓
ExporterHistory.url = presigned_url
```

### 3.2 文件上传与链接生成

**文件**: `apps/api/plane/bgtasks/export_task.py:42-124`

```python
def upload_to_s3(zip_file, workspace_id, token_id, slug):
    # 1. 生成文件名: {workspace_id}/export-{slug}-{token[:6]}-{date}.zip
    file_name = f"{workspace_id}/export-{slug}-{token_id[:6]}-{str(timezone.now().date())}.zip"
    
    # 2. 上传到 S3/MinIO
    s3.upload_fileobj(zip_file, bucket_name, file_name)
    
    # 3. 生成预签名 URL (有效期7天)
    presigned_url = s3.generate_presigned_url(
        "get_object",
        Params={"Bucket": bucket_name, "Key": file_name},
        ExpiresIn=7 * 24 * 60 * 60,  # 7天
    )
    
    # 4. 回写数据库
    exporter_instance = ExporterHistory.objects.get(token=token_id)
    exporter_instance.url = presigned_url
    exporter_instance.status = "completed"
    exporter_instance.save()
```

### 3.3 前端轮询机制

**文件**: `apps/web/core/components/exporter/prev-exports.tsx`

```typescript
// 1. 使用 SWR 进行数据获取和缓存
const { data: exporterServices } = useSWR(
  EXPORT_SERVICES_LIST(workspaceSlug, cursor, `${per_page}`),
  () => integrationService.getExportsServicesList(workspaceSlug, cursor, per_page)
);

// 2. 自动轮询：只要有 processing 状态的任务，每3秒刷新一次
useEffect(() => {
  const interval = setInterval(() => {
    if (exporterServices?.results?.some((service) => service.status === "processing")) {
      handleRefresh();  // 调用 mutate 刷新数据
    } else {
      clearInterval(interval);  // 全部完成后停止轮询
    }
  }, 3000);  // 3秒轮询间隔

  return () => clearInterval(interval);
}, [exporterServices]);
```

### 3.4 前端 API 调用

**文件**: `apps/web/core/services/integrations/integration.service.ts`

```typescript
// 获取导出历史列表（带分页）
async getExportsServicesList(workspaceSlug, cursor, per_page) {
  return this.get(`/api/workspaces/${workspaceSlug}/export-issues`, {
    params: { per_page, cursor },
  });
}
```

---

## 四、核心模块关系图

```
┌─────────────────────────────────────────────────────────────────┐
│                         前端 (Web)                              │
│  ┌──────────────┐     ┌─────────────────────────────────────┐  │
│  │ Export Form  │────▶│ ProjectExportService.csvExport()    │  │
│  └──────────────┘     └─────────────────┬───────────────────┘  │
│                                        │ POST /export-issues/  │
│  ┌──────────────┐     ┌────────────────▼───────────────────┐  │
│  │ PrevExports  │◀────│ IntegrationService.getExportsList()│  │
│  │ (3s轮询)     │     └─────────────────┬───────────────────┘  │
│  └──────────────┘                       │ GET /export-issues/  │
└──────────────────────────────────────────┼──────────────────────┘
                                           │
┌──────────────────────────────────────────┼──────────────────────┐
│                        后端 (API)                                │
│  ┌────────────────────────▼──────────────────────────────────┐  │
│  │           ExportIssuesEndpoint (API 视图)                 │  │
│  │  POST: 创建 ExporterHistory → 提交 Celery 任务            │  │
│  │  GET:  分页查询 ExporterHistory 列表                       │  │
│  └────────────────────────┬──────────────────────────────────┘  │
│                           │ token_id                            │
│  ┌────────────────────────▼──────────────────────────────────┐  │
│  │              ExporterHistory (DB 模型)                    │  │
│  │  status: queued → processing → completed/failed           │  │
│  │  url: S3 预签名下载链接                                    │  │
│  └────────────────────────┬──────────────────────────────────┘  │
│                           │ .delay()                            │
│  ┌────────────────────────▼──────────────────────────────────┐  │
│  │              Celery Worker (后台进程)                      │  │
│  │  ┌─────────────────────────────────────────────────────┐  │  │
│  │  │ issue_export_task()                                 │  │  │
│  │  │  1. status = processing                             │  │  │
│  │  │  2. QuerySet 数据查询 + prefetch                     │  │  │
│  │  │  3. DataExporter → IssueExportSerializer             │  │  │
│  │  │  4. Formatter (CSV/JSON/XLSX)                        │  │  │
│  │  │  5. ZIP 打包                                        │  │  │
│  │  │  6. S3 上传 + 生成预签名 URL                         │  │  │
│  │  │  7. status = completed, url = presigned_url          │  │  │
│  │  └─────────────────────────────────────────────────────┘  │  │
│  └───────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 五、关键技术点

### 5.1 数据查询优化

`bgtasks/export_task.py:148-190` 中使用了大量的 `select_related` 和 `prefetch_related`：
- `select_related`: 一对一/外键关联（project, workspace, state, created_by 等）
- `prefetch_related`: 多对多/反向关联（labels, assignees, comments, relations 等）
- 使用 `Prefetch` 对象自定义查询集（如按创建时间排序评论）

### 5.2 序列化器设计

`utils/porters/serializers/issue.py` 中的 `IssueExportSerializer`：
- 继承自 `IssueSerializer`，专门为导出优化
- 将 UUID 转换为可读名称（如 `created_by_name`）
- 将关联对象转换为字符串或简化结构（如 `assignees` 转为姓名列表）
- 导出字段经过精心挑选，避免敏感信息

### 5.3 格式化器插件化

`utils/porters/formatters.py` 采用策略模式：
- `BaseFormatter` 抽象基类定义 `encode/decode/extension` 接口
- `CSVFormatter`/`JSONFormatter`/`XLSXFormatter` 具体实现
- `DataExporter` 根据 `format_type` 动态选择格式化器
- 支持注册新的格式化器（`Exporter.register_formatter()`）

### 5.4 多项目导出支持

通过 `multiple` 参数控制：
- `multiple=true`: 每个项目单独导出一个文件，打包到同一个 ZIP
- `multiple=false`: 所有项目合并导出到一个文件

---

## 六、异常处理机制

| 异常场景 | 处理方式 |
|---------|---------|
| 不支持的导出格式 | 任务内捕获，`status="failed"`, `reason="Unsupported format"` |
| 数据查询异常 | 外层 try-catch 捕获，`status="failed"`, `reason=str(e)` |
| S3 上传失败 | `upload_to_s3` 中抛出异常，外层捕获并标记失败 |
| 序列化错误 | 由 DRF serializer 抛出，外层捕获 |

所有异常都会通过 `log_exception(e)` 记录到日志系统。

---

## 七、文件清单

| 文件路径 | 职责 |
|---------|------|
| `apps/api/plane/app/views/exporter/base.py` | API 入口，任务提交与历史查询 |
| `apps/api/plane/app/urls/exporter.py` | 路由配置 |
| `apps/api/plane/db/models/exporter.py` | ExporterHistory 数据模型 |
| `apps/api/plane/bgtasks/export_task.py` | Celery 后台任务核心逻辑 |
| `apps/api/plane/utils/porters/exporter.py` | DataExporter 导出编排器 |
| `apps/api/plane/utils/porters/formatters.py` | CSV/JSON/XLSX 格式化器 |
| `apps/api/plane/utils/porters/serializers/issue.py` | Issue 导出序列化器 |
| `apps/web/core/services/project/project-export.service.ts` | 前端导出请求服务 |
| `apps/web/core/services/integrations/integration.service.ts` | 前端导出历史查询服务 |
| `apps/web/core/components/exporter/prev-exports.tsx` | 导出历史列表 + 轮询组件 |
| `apps/web/core/components/exporter/column.tsx` | 导出列表列定义 + 过期判定 |
| `apps/web/core/components/exporter/single-export.tsx` | 单条导出记录展示组件 |
| `apps/api/plane/bgtasks/exporter_expired_task.py` | 过期导出清理任务 |
| `apps/api/plane/celery.py` | Celery Beat 定时任务配置 |

---

## 八、过期清理机制：后台回收下载地址

### 8.1 定时任务配置

**文件**: `apps/api/plane/celery.py`

```python
app.conf.beat_schedule = {
    # 每天 UTC 01:30 执行一次
    "check-every-day-to-delete_exporter_history": {
        "task": "plane.bgtasks.exporter_expired_task.delete_old_s3_link",
        "schedule": crontab(hour=1, minute=30),  # UTC 01:30
    },
    # 每天 UTC 03:45 再次执行（双重保险）
    "check-every-day-to-delete-exporter-history": {
        "task": "plane.bgtasks.exporter_expired_task.delete_old_s3_link",
        "schedule": crontab(hour=3, minute=45),  # UTC 03:45
    },
}
```

**设计考量**:
- 每天执行两次，确保清理任务不会因为单次执行失败而遗漏
- 选择低峰时段（UTC 凌晨）执行，避免影响用户体验
- 任务注册在 `CELERY_IMPORTS` 中：`"plane.bgtasks.exporter_expired_task"`

### 8.2 清理任务实现

**文件**: `apps/api/plane/bgtasks/exporter_expired_task.py`

```python
@shared_task
def delete_old_s3_link():
    # 找出创建时间 >= 8天 的导出记录（比7天有效期多1天缓冲期）
    expired_exporter_history = ExporterHistory.objects.filter(
        Q(url__isnull=False) & Q(created_at__lte=timezone.now() - timedelta(days=8))
    ).values_list("key", "id")
    
    for file_name, exporter_id in expired_exporter_history:
        # 1. 从 S3/MinIO 物理删除文件
        if file_name:
            s3.delete_object(Bucket=bucket_name, Key=file_name)
        
        # 2. 数据库中将 url 置空（不删除记录，保留历史）
        ExporterHistory.objects.filter(id=exporter_id).update(url=None)
```

### 8.3 清理策略详解

| 维度 | 策略 | 说明 |
|-----|------|------|
| **清理阈值** | 创建时间 >= 8天 | 比预签名URL的7天有效期多1天缓冲 |
| **筛选条件** | `url__isnull=False` | 只处理有下载链接的记录 |
| **物理删除** | S3 文件删除 | 释放存储空间 |
| **数据库处理** | `url=None` | 保留历史记录，仅清空下载链接 |
| **保留历史** | 不删除 ExporterHistory | 保留审计痕迹 |
| **执行频率** | 每天2次 | UTC 01:30 和 03:45 |

### 8.4 清理前后对比

```
清理前:
ExporterHistory {
    status: "completed",
    url: "https://s3.amazonaws.com/.../export.zip?X-Amz-Signature=...",
    key: "workspace_id/export-slug-abc123-2026-05-12.zip",
    created_at: "2026-05-12T10:00:00Z"
}
S3: 存在 export.zip 文件

清理后 (2026-05-20 执行):
ExporterHistory {
    status: "completed",  // 状态不变
    url: null,            // 下载链接被清空
    key: "...",           // key 保留（用于审计）
    created_at: "2026-05-12T10:00:00Z"  // 创建时间不变
}
S3: export.zip 文件已被删除
```

---

## 九、前端过期判定与状态展示

### 9.1 过期判定逻辑

前端通过 **纯客户端计算** 判定链接是否过期，不依赖后端状态字段。

**文件**: `apps/web/core/components/exporter/column.tsx:12-18` 和 `apps/web/core/components/exporter/single-export.tsx:25-31`

```typescript
const checkExpiry = (inputDateString: string) => {
  const currentDate = new Date();
  const expiryDate = getDate(inputDateString);  // 解析 created_at
  if (!expiryDate) return false;
  expiryDate.setDate(expiryDate.getDate() + 7);  // 创建时间 + 7天
  return expiryDate > currentDate;  // 比较当前时间
};
```

**判定公式**:
```
是否过期 = (创建时间 + 7天) <= 当前时间
```

**双重过期判定**:
1. **客户端判定**: 每次渲染时实时计算 `created_at + 7天`
2. **服务端判定**: 后台定时任务将 `url` 置空（8天阈值）

这意味着在第7天到第8天之间，客户端会显示 "Expired"，但 S3 文件还在；第8天后，后台任务清理文件和 `url`。

### 9.2 状态颜色映射

**文件**: `apps/web/core/components/exporter/column.tsx:77-93`

```typescript
className={`rounded-sm px-2 py-1 text-11 capitalize ${
  rowData.status === "completed"
    ? "bg-success-subtle text-success-primary"    // 绿色
    : rowData.status === "processing"
      ? "bg-yellow-500/20 text-yellow-500"       // 黄色
      : rowData.status === "failed"
        ? "bg-danger-subtle text-danger-primary" // 红色
        : rowData.status === "expired"
          ? "bg-orange-500/20 text-orange-500"   // 橙色
          : "bg-gray-500/20 text-gray-500"       // 灰色（queued）
}`}
```

### 9.3 状态展示矩阵

| 后端 status | 前端 checkExpiry() | 展示状态 | 下载按钮 |
|------------|-------------------|---------|---------|
| `queued` | - | 灰色 "queued" | 显示 `-` |
| `processing` | - | 黄色 "processing" | 显示 `-` |
| `completed` | `true` (<7天) | 绿色 "completed" | 可点击 "Download" |
| `completed` | `false` (>=7天) | 绿色 "completed" | 显示 "Expired" |
| `failed` | - | 红色 "failed" | 显示 `-` |

**注意**: `expired` 状态仅在前端逻辑中存在，后端数据库没有 `expired` 这个状态值。后端的 `completed` 记录在超过7天后，前端会把它当作过期处理。

### 9.4 下载按钮渲染逻辑

**文件**: `apps/web/core/components/exporter/column.tsx:98-114`

```typescript
tdRender: (rowData: RowData) =>
  checkExpiry(rowData.created_at) ? (
    // 未过期
    rowData.status == "completed" ? (
      <a target="_blank" href={rowData?.url} rel="noopener noreferrer">
        <button className="flex w-full items-center gap-1 font-medium text-accent-primary">
          <Download className="h-4 w-4" />
          <div>Download</div>
        </button>
      </a>
    ) : (
      "-"  // processing/queued/failed 状态
    )
  ) : (
    // 已过期
    <div className="text-11 text-danger-primary">Expired</div>
  ),
```

---

## 十、轮询机制详解：queued 与 processing 的差异

### 10.1 轮询触发条件

**文件**: `apps/web/core/components/exporter/prev-exports.tsx:54-64`

```typescript
useEffect(() => {
  const interval = setInterval(() => {
    // 只要有任何一条记录处于 processing 状态，就继续轮询
    if (exporterServices?.results?.some((service) => service.status === "processing")) {
      handleRefresh();  // 调用 SWR mutate 刷新
    } else {
      clearInterval(interval);  // 没有 processing 任务，停止轮询
    }
  }, 3000);  // 每3秒轮询一次

  return () => clearInterval(interval);  // 组件卸载时清理定时器
}, [exporterServices]);  // 依赖项：exporterServices 变化时重建定时器
```

### 10.2 queued vs processing 状态对轮询的影响

| 状态 | 是否触发轮询 | 说明 |
|-----|------------|------|
| `queued` | ❌ 不触发 | 轮询条件只检查 `status === "processing"` |
| `processing` | ✅ 触发 | 持续每3秒刷新，直到状态变更 |
| `completed` | ❌ 不触发 | 任务完成，无需再轮询 |
| `failed` | ❌ 不触发 | 任务失败，无需再轮询 |

**关键发现**: `queued` 状态的任务 **不会触发自动轮询**。这意味着：
- 如果任务长时间卡在 `queued` 状态（如 Celery Worker 挂了），前端不会自动刷新
- 用户必须手动点击 "Refresh Status" 按钮才能看到最新状态
- 只有当任务进入 `processing` 状态后，自动轮询才会启动

### 10.3 轮询启停时序图

```
用户发起导出
    ↓
ExporterHistory 创建 (status=queued)
    ↓
页面显示灰色 "queued" 状态
    ↓
[此时不会自动轮询]
    ↓
Celery Worker 开始执行
    ↓
status 变为 processing
    ↓
用户刷新页面 / 手动刷新
    ↓
exporterServices 更新，包含 processing 任务
    ↓
useEffect 检测到 processing，启动 3s 轮询
    ↓
每隔 3s 调用 handleRefresh() → mutate() → GET /export-issues/
    ↓
任务完成，status 变为 completed
    ↓
下一次轮询检测到无 processing 任务
    ↓
clearInterval(interval)，停止轮询
```

### 10.4 手动刷新机制

**文件**: `apps/web/core/components/exporter/prev-exports.tsx:49-52`

```typescript
const handleRefresh = () => {
  setRefreshing(true);
  // 调用 SWR 的 mutate 强制重新获取数据
  mutate(EXPORT_SERVICES_LIST(workspaceSlug, `${cursor}`, `${per_page}`))
    .then(() => setRefreshing(false));
};
```

**手动刷新触发场景**:
1. 用户点击 "Refresh Status" 按钮
2. 自动轮询每隔 3 秒调用一次（当有 processing 任务时）

### 10.5 潜在问题与优化点

**当前设计的局限**:
```
问题: queued 状态的任务不会触发自动轮询
场景: 
  1. 用户发起导出 → status=queued
  2. Celery Worker 因为某些原因延迟 pickup 任务
  3. 前端页面一直显示 queued，不会自动刷新
  4. 用户以为导出卡住了，可能重复发起导出
```

**建议优化**: 轮询条件应该同时检查 `queued` 和 `processing`:
```typescript
// 优化前（当前）
if (exporterServices?.results?.some((service) => service.status === "processing"))

// 优化后
if (exporterServices?.results?.some((service) => 
    service.status === "processing" || service.status === "queued"
))
```

---

## 十一、完整生命周期闭环

### 11.1 全流程时间线

```
T+0s:     用户点击 Export 按钮
          ↓ POST /export-issues/
T+0.1s:   ExporterHistory 创建 (status=queued, token=xxx)
          ↓ issue_export_task.delay() 提交到 Celery 队列
T+0.2s:   API 返回 200 OK，前端显示成功提示
          ↓
T+0.5s:   用户跳转到导出列表页
          ↓ GET /export-issues/?per_page=10&cursor=...
          ↓ 显示 status=queued（灰色）
          ↓ [无自动轮询，因为 queued 不触发]
          ↓
T+2s:     Celery Worker pickup 任务
          ↓ status 更新为 processing
          ↓
T+2.1s:   [用户手动刷新 或 下一次自动轮询]
          ↓ 获取到 processing 状态
          ↓ 启动 3s 自动轮询
          ↓
T+5s:     第一轮轮询，仍在 processing
T+8s:     第二轮轮询，仍在 processing
...       (持续轮询)
          ↓
T+N s:    后台处理完成，status=completed, url=预签名链接
          ↓
T+N+3s:   轮询检测到 completed，无 processing 任务
          ↓ 停止轮询
          ↓ 显示绿色 completed + Download 按钮
          ↓
T+7天:    前端 checkExpiry() 返回 false
          ↓ Download 按钮变为 "Expired"
          ↓
T+8天:    后台定时任务执行
          ↓ S3 文件被删除，url 被置空
```

### 11.2 状态流转完整表

| 阶段 | 后端 status | 前端展示 | 自动轮询 | 下载按钮 |
|-----|------------|---------|---------|---------|
| 刚创建 | `queued` | 灰色 queued | ❌ 停止 | `-` |
| 处理中 | `processing` | 黄色 processing | ✅ 运行中 | `-` |
| 完成 (<7天) | `completed` | 绿色 completed | ❌ 停止 | 可下载 |
| 完成 (>=7天) | `completed` | 绿色 completed | ❌ 停止 | Expired |
| 完成 (>=8天) | `completed` (url=null) | 绿色 completed | ❌ 停止 | Expired |
| 失败 | `failed` | 红色 failed | ❌ 停止 | `-` |

