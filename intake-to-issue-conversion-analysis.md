# Intake 到 Issue 转换链路分析

## 一、概述

Intake（收件箱）是 Plane 中用于收集外部或内部反馈的功能，用户提交的 Intake Issue 经过运营审核后，可以被接受、拒绝、暂停或标记为重复。被接受的 Intake Issue 会转换为正式的项目 Issue 并落入目标项目中。

整个链路分为三个核心阶段：
1. **表单接入**：用户通过表单提交 Intake Issue
2. **人工审阅判定**：运营人员在收件箱中审核并决策
3. **Issue 创建回写**：接受后 Issue 状态转换并落入目标项目

---

## 二、核心数据模型

### 2.1 Intake（收件箱配置）
**文件**: `apps/api/plane/db/models/intake.py:12-35`

```python
class Intake(ProjectBaseModel):
    name = models.CharField(max_length=255)
    description = models.TextField(blank=True)
    is_default = models.BooleanField(default=False)
    view_props = models.JSONField(default=dict)
    logo_props = models.JSONField(default=dict)
```

每个项目可以有一个默认 Intake，用于收集反馈。

### 2.2 IntakeIssue（收件箱 Issue 关联）
**文件**: `apps/api/plane/db/models/intake.py:50-84`

```python
class IntakeIssue(ProjectBaseModel):
    intake = models.ForeignKey("db.Intake", related_name="issue_intake", on_delete=models.CASCADE)
    issue = models.ForeignKey("db.Issue", related_name="issue_intake", on_delete=models.CASCADE)
    status = models.IntegerField(choices=..., default=-2)  # PENDING
    snoozed_till = models.DateTimeField(null=True)
    duplicate_to = models.ForeignKey("db.Issue", ..., null=True)
    source = models.CharField(max_length=255, default="IN_APP")
    source_email = models.TextField(blank=True, null=True)
    extra = models.JSONField(default=dict)
```

**状态枚举** (IntakeIssueStatus):
- `-2` PENDING (待审核)
- `-1` REJECTED (已拒绝)
- `0` SNOOZED (已暂停)
- `1` ACCEPTED (已接受)
- `2` DUPLICATE (重复)

IntakeIssue 是 Intake 和 Issue 之间的桥接表，记录审核状态和相关元数据。

---

## 三、阶段一：表单接入

### 3.1 前端提交入口

**文件**: `apps/web/core/services/inbox/inbox-issue.service.ts:40-48`

```typescript
async create(workspaceSlug: string, projectId: string, data: Partial<TIssue>): Promise<TInboxIssue> {
  return this.post(`/api/workspaces/${workspaceSlug}/projects/${projectId}/inbox-issues/`, {
    source: EInboxIssueSource.IN_APP,
    issue: data,
  })
  .then((response) => response?.data)
  .catch((error) => { throw error?.response?.data; });
}
```

### 3.2 后端内部 API 处理

**文件**: `apps/api/plane/app/views/intake/base.py:221-325`

**IntakeIssueViewSet.create() 处理流程**:

1. **参数校验** (223-234行):
   - 校验 `issue.name` 必填
   - 校验 `priority` 有效值: low/medium/high/urgent/none

2. **获取或创建 Triage 状态** (238-250行):
   ```python
   triage_state = State.triage_objects.filter(project_id=project_id, ...).first()
   if not triage_state:
       triage_state = State.objects.create(
           name="Triage",
           group=StateGroup.TRIAGE.value,
           ...
       )
   request.data["issue"]["state_id"] = triage_state.id
   ```

3. **创建 Issue** (252-263行):
   ```python
   serializer = IssueCreateSerializer(data=request.data.get("issue"), context={...})
   if serializer.is_valid():
       serializer.save()
   ```

4. **创建 IntakeIssue 关联** (264-271行):
   ```python
   intake_id = Intake.objects.filter(...).first()
   intake_issue = IntakeIssue.objects.create(
       intake_id=intake_id.id,
       project_id=project_id,
       issue_id=serializer.data["id"],
       source=SourceType.IN_APP,
   )
   ```

5. **异步任务触发** (272-291行):
   - `issue_activity.delay()` - 记录 Issue 创建活动
   - `issue_description_version_task.delay()` - 保存描述版本

### 3.3 公开 API 接口（DeployBoard）

**文件**: `apps/api/plane/space/views/intake.py:107-173`

`IntakeIssuePublicViewSet.create()` 用于公开的 DeployBoard 表单提交，流程与内部 API 类似，但通过 anchor 验证项目权限。

---

## 四、阶段二：人工审阅判定

### 4.1 前端审核界面

**文件**: `apps/web/core/components/inbox/content/inbox-issue-header.tsx`

**审核操作按钮** (328-435行):
- **Accept (接受)**: 调用 `handleInboxIssueAccept()` → `updateInboxIssueStatus(ACCEPTED)`
- **Decline (拒绝)**: 调用 `handleInboxIssueDecline()` → `updateInboxIssueStatus(DECLINED)`
- **Snooze (暂停)**: 调用 `handleInboxIssueSnooze(date)` → `updateInboxIssueSnoozeTill(date)`
- **Mark as Duplicate (标记重复)**: 调用 `handleInboxIssueDuplicate(issueId)` → `updateInboxIssueDuplicateTo(issueId)`

**权限控制** (89-107行):
- 仅 ADMIN/MEMBER 角色可执行审核操作
- 标记接受/拒绝需要 PROJECT ADMIN 权限

### 4.2 前端 Store 状态管理

**文件**: `apps/web/core/store/inbox/inbox-issue.store.ts`

**updateInboxIssueStatus()** (99-143行):
```typescript
updateInboxIssueStatus = async (status: TInboxIssueStatus) => {
  const inboxIssue = await this.inboxIssueService.update(
    this.workspaceSlug, 
    this.projectId, 
    this.issue.id, 
    { status: status }
  );
  
  runInAction(() => {
    set(this, "status", inboxIssue?.status);
    // 更新 intake_count 计数
    if (previousStatus === PENDING && inboxIssue.status !== PENDING) {
      // 减少 pending 计数
    }
  });

  // 如果是 ACCEPTED，同步 Issue 到本地 store
  if (status === EInboxIssueStatus.ACCEPTED) {
    const updatedIssue = { ...this.issue, ...inboxIssue.issue };
    this.store.issue.issues.addIssue([updatedIssue]);
  }
};
```

### 4.3 后端审核处理

**文件**: `apps/api/plane/app/views/intake/base.py:327-496`

**IntakeIssueViewSet.partial_update() 处理流程**:

1. **权限校验** (340-367行):
   - 仅 ADMIN 或创建者可更新
   - GUEST 只能编辑自己创建的，且只能修改 name/description

2. **Issue 数据更新** (376-427行):
   - 如果请求包含 `issue` 字段，更新关联的 Issue
   - GUEST 角色只能修改 name 和 description

3. **IntakeIssue 状态更新** (418-423行):
   - 仅 MEMBER 以上角色可更新 IntakeIssue 状态
   - 通过 `IntakeIssueSerializer` 进行验证和保存

---

## 五、阶段三：Issue 创建回写

### 5.1 状态转换逻辑（核心）

**文件**: `apps/api/plane/app/serializers/intake.py:68-84`

`IntakeIssueSerializer.update()` 是状态转换的核心:

```python
def update(self, instance, validated_data):
    instance = super().update(instance, validated_data)

    # 如果状态变为 ACCEPTED (1)
    if validated_data.get("status") == 1:
        issue = instance.issue
        # 如果 Issue 当前在 TRIAGE 状态
        if issue.state and issue.state.group == StateGroup.TRIAGE.value:
            # 获取项目默认状态
            default_state = State.objects.filter(
                workspace=instance.workspace,
                project=instance.project,
                default=True
            ).first()
            if default_state:
                # 将 Issue 从 TRIAGE 移动到默认状态
                issue.state = default_state
                issue.save()

    return instance
```

### 5.2 状态变更前置校验

**文件**: `apps/api/plane/app/serializers/intake.py:43-66`

`IntakeIssueSerializer.validate()` 确保在接受前项目有默认状态:

```python
def validate(self, attrs):
    if attrs.get("status") == 1:  # ACCEPTED
        intake_issue = self.instance
        issue = intake_issue.issue
        
        if issue.state and issue.state.group == StateGroup.TRIAGE.value:
            default_state = State.objects.filter(
                workspace=intake_issue.workspace,
                project=intake_issue.project,
                default=True
            ).first()
            
            if not default_state:
                raise serializers.ValidationError(
                    {"status": "Cannot accept intake issue: No default state found for the project"}
                )
    return attrs
```

### 5.3 删除时的级联处理

**文件**: `apps/api/plane/app/views/intake/base.py:545-562`

```python
def destroy(self, request, slug, project_id, pk):
    intake_issue = IntakeIssue.objects.get(...)
    
    # 如果状态是 PENDING/REJECTED/SNOOZED/DUPLICATE，同时删除关联的 Issue
    if intake_issue.status in [-2, -1, 0, 2]:
        issue = Issue.objects.filter(...).first()
        issue.delete()
    
    intake_issue.delete()
```

---

## 六、API 路由映射

**文件**: `apps/api/plane/app/urls/intake.py`

| 方法 | 路径 | 处理函数 | 说明 |
|------|------|----------|------|
| POST | `/api/workspaces/<slug>/projects/<project_id>/inbox-issues/` | `IntakeIssueViewSet.create()` | 提交 Intake Issue |
| PATCH | `/api/workspaces/<slug>/projects/<project_id>/inbox-issues/<pk>/` | `IntakeIssueViewSet.partial_update()` | 审核/更新 Intake Issue |
| DELETE | `/api/workspaces/<slug>/projects/<project_id>/inbox-issues/<pk>/` | `IntakeIssueViewSet.destroy()` | 删除 Intake Issue |
| POST | `/api/public/deploy-boards/<anchor>/intakes/<intake_id>/inbox-issues/` | `IntakeIssuePublicViewSet.create()` | 公开表单提交 |

---

## 七、完整流程图

```
用户提交表单
    ↓
[前端] InboxIssueService.create()
    ↓
[后端] IntakeIssueViewSet.create()
    ├─→ 校验参数
    ├─→ 获取/创建 TRIAGE 状态
    ├─→ 创建 Issue (状态: TRIAGE)
    ├─→ 创建 IntakeIssue (状态: PENDING)
    └─→ 触发异步任务 (activity, description_version)
    ↓
运营人员审核
    ↓
[前端] InboxIssueStore.updateInboxIssueStatus(ACCEPTED)
    ↓
[后端] IntakeIssueViewSet.partial_update()
    ├─→ 权限校验
    ├─→ IntakeIssueSerializer.validate()
    │   └─→ 检查项目默认状态是否存在
    ├─→ IntakeIssueSerializer.update()
    │   ├─→ 更新 IntakeIssue.status = ACCEPTED
    │   └─→ 若 Issue 在 TRIAGE → 移动到项目默认状态
    └─→ 触发异步任务 (activity)
    ↓
Issue 正式落入项目
```

---

## 八、关键代码位置汇总

| 模块 | 文件路径 | 关键函数/类 |
|------|----------|-------------|
| 数据模型 | `apps/api/plane/db/models/intake.py` | `Intake`, `IntakeIssue`, `IntakeIssueStatus` |
| 后端 API | `apps/api/plane/app/views/intake/base.py` | `IntakeIssueViewSet.create()`, `partial_update()` |
| 公开 API | `apps/api/plane/space/views/intake.py` | `IntakeIssuePublicViewSet.create()` |
| 序列化器 | `apps/api/plane/app/serializers/intake.py` | `IntakeIssueSerializer.update()`, `validate()` |
| 前端服务 | `apps/web/core/services/inbox/inbox-issue.service.ts` | `InboxIssueService.create()`, `update()` |
| 前端状态 | `apps/web/core/store/inbox/inbox-issue.store.ts` | `InboxIssueStore.updateInboxIssueStatus()` |
| 前端 UI | `apps/web/core/components/inbox/content/inbox-issue-header.tsx` | `InboxIssueActionsHeader` |
| 路由配置 | `apps/api/plane/app/urls/intake.py` | URL 映射 |
