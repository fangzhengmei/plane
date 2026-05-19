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
