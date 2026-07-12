基于你之前在做的 ERP / 办公系统的 Supabase + Vue3 经验，这套审批工作流引擎其实是可以复用的通用能力层。我按"数据模型 → 引擎逻辑 → Supabase 实现 → 前端模块"这个顺序讲，这样思路是从下往上串起来的。

## 一、核心设计思想

先明确一个关键点：**业务模块和审批引擎要解耦**。请假、采购、报销这些业务表不应该各自实现一套审批逻辑，而是统一通过"业务类型 + 业务ID"挂接到通用的 workflow_instances 上。这样以后加一个新业务模块（比如合同审批），只需要配置流程模板，不需要写新的审批代码。

```
业务表(leave_requests/purchase_orders/...) 
        │ business_type + business_id
        ▼
  workflow_instances (流程实例)
        │
        ▼
  workflow_instance_nodes (节点状态机)
        │
        ▼
  approval_records (审批日志)
```

## 二、数据库设计（Supabase Postgres）

### 1. 流程定义层（配置态，流程设计器产出）

```sql
-- 流程模板（对应"模板管理"）
create table workflow_templates (
  id uuid primary key default gen_random_uuid(),
  business_type text not null,        -- 'leave' | 'purchase' | 'reimburse' ...
  name text not null,
  description text,
  form_schema jsonb,                  -- 表单字段定义（动态表单渲染用）
  is_active boolean default true,
  created_by uuid references auth.users,
  created_at timestamptz default now()
);

-- 流程版本（支持流程变更但不影响运行中实例）
create table workflow_definitions (
  id uuid primary key default gen_random_uuid(),
  template_id uuid references workflow_templates(id),
  version int not null,
  flow_schema jsonb not null,         -- 核心：节点图定义，见下方JSON结构
  is_published boolean default false,
  created_at timestamptz default now()
);
```

**flow_schema 的 JSON 结构**（这是引擎的"程序"，流程设计器就是在编辑这段 JSON）：

```json
{
  "nodes": [
    { "id": "start", "type": "start", "next": "node_leader" },
    {
      "id": "node_leader",
      "type": "approval",
      "name": "直属主管审批",
      "approverStrategy": { "type": "role", "value": "direct_manager" },
      "approvalMode": "any",          // any=或签 all=会签
      "next": "cond_amount"
    },
    {
      "id": "cond_amount",
      "type": "condition",
      "branches": [
        { "expr": "form.days > 3", "next": "node_hr" },
        { "expr": "default", "next": "end" }
      ]
    },
    {
      "id": "node_hr",
      "type": "approval",
      "name": "HR审批",
      "approverStrategy": { "type": "dept", "value": "HR" },
      "approvalMode": "any",
      "next": "end"
    },
    { "id": "end", "type": "end" }
  ]
}
```

节点类型建议至少支持：`start / approval / condition(条件分支) / cc(抄送) / auto(自动通过，比如金额<100自动过) / end`

approverStrategy 至少支持几种常见策略：
- `user`：指定人
- `role`：角色（如"部门经理"）
- `dept`：指定部门负责人
- `manager_chain`：连续上级（逐级往上，可配置层数）
- `initiator_field`：取表单里某字段值作为审批人（比如报销单里"项目负责人"）

### 2. 运行时层（实例态）

```sql
-- 流程实例
create table workflow_instances (
  id uuid primary key default gen_random_uuid(),
  definition_id uuid references workflow_definitions(id),
  business_type text not null,
  business_id uuid not null,          -- 关联到 leave_requests.id 等
  form_data jsonb,                    -- 提交时的表单快照
  initiator_id uuid references auth.users not null,
  current_node_id text,               -- 当前所在节点
  status text not null default 'running', -- running/approved/rejected/cancelled
  created_at timestamptz default now(),
  finished_at timestamptz
);

-- 实例节点状态（一个实例会经过多个节点，需要记录每个节点的状态）
create table workflow_instance_nodes (
  id uuid primary key default gen_random_uuid(),
  instance_id uuid references workflow_instances(id) on delete cascade,
  node_id text not null,
  node_name text,
  status text default 'pending',      -- pending/processing/approved/rejected/skipped
  approvers uuid[],                   -- 该节点计算出的审批人列表
  approval_mode text,                 -- any/all
  created_at timestamptz default now()
);

-- 审批记录（审批日志，"审批统计"数据来源）
create table approval_records (
  id uuid primary key default gen_random_uuid(),
  instance_id uuid references workflow_instances(id) on delete cascade,
  instance_node_id uuid references workflow_instance_nodes(id),
  approver_id uuid references auth.users not null,
  action text not null,               -- approve/reject/transfer/add_sign/withdraw
  comment text,
  attachments jsonb,
  created_at timestamptz default now()
);

-- 待办任务（"我的待办"直接查这张表，性能关键）
create table approval_tasks (
  id uuid primary key default gen_random_uuid(),
  instance_id uuid references workflow_instances(id) on delete cascade,
  instance_node_id uuid references workflow_instance_nodes(id),
  approver_id uuid references auth.users not null,
  status text default 'pending',      -- pending/done/transferred
  created_at timestamptz default now()
);
```

**关键设计点**：单独建 `approval_tasks` 表而不是每次都从 `workflow_instance_nodes` 里的 approvers 数组去反查"我的待办"。原因是数组字段做"我作为审批人有哪些待办"这种查询效率差，而且或签场景下一个人处理了要让同组其他人的待办自动失效，用独立任务表配合状态更新更清晰，索引也好加（`approver_id + status`）。

### 3. 通知中心

```sql
create table notifications (
  id uuid primary key default gen_random_uuid(),
  user_id uuid references auth.users not null,
  type text,                          -- 'todo'/'approved'/'rejected'/'cc'
  title text,
  content text,
  link jsonb,                         -- {business_type, business_id, instance_id}
  is_read boolean default false,
  created_at timestamptz default now()
);
```

## 三、引擎推进逻辑（最核心的部分）

这块不建议用纯前端 JS 跑，因为审批动作涉及并发（两个审批人同时点"同意"）、事务一致性，**强烈建议用 Supabase Edge Function + Postgres 事务**来做,不要在前端直接改状态表。

流转的核心是一个"推进函数"，伪代码：

```
function advanceWorkflow(instanceId, currentNodeId, action, approverId, comment):
  begin transaction
    1. 写入 approval_records
    2. 更新 approval_tasks: 该节点其他待办(如果是或签)标记为 skipped/transferred
    3. 判断 instance_node 是否满足流转条件:
         - any: 一人通过即可
         - all: 需要所有 approvers 都通过
    4. 若未满足 -> 提交事务，等待其他人
    5. 若拒绝 action -> instance.status = rejected, 结束流程
    6. 若通过:
         - 计算 flow_schema 中当前节点的 next
         - 若 next 是 condition 节点 -> 用 form_data 求值找到真正下一个审批节点
         - 若 next 是 end -> instance.status = approved
         - 否则创建新的 instance_node + 计算 approverStrategy 生成审批人 + 插入 approval_tasks
    7. 写入 notifications 通知下一批审批人
  commit transaction
```

**为什么放在 Edge Function**：
1. 审批人计算逻辑（比如"取直属上级"）往往要查组织架构表，用 Postgres function（PL/pgSQL）写在数据库里执行效率最高，也能保证事务原子性。
2. 条件分支表达式求值（`form.days > 3`）如果逻辑简单，可以用 Postgres 的 jsonb 操作符直接判断；复杂表达式的话在 Edge Function（Deno）里用一个安全的表达式求值库处理更灵活。

我建议的分工是：**审批人策略计算和条件分支求值放 Postgres Function**（`get_approvers(strategy jsonb, business_data jsonb)`），**Edge Function 只做编排**（调用这些 function、处理事务、触发通知）。这样以后你就算换前端框架，这套引擎逻辑也不用动。

## 四、Supabase RLS 设计要点

按你之前 ERP 项目的经验，这块要特别注意几个坑：

1. **业务表本身的 RLS** 要结合 workflow_instances 判断，比如"请假单只有发起人、当前审批人、HR 能看"：

```sql
create policy "leave_requests_select" on leave_requests
for select using (
  auth.uid() = employee_id
  or exists (
    select 1 from approval_tasks t
    join workflow_instances wi on wi.id = t.instance_id
    where wi.business_id = leave_requests.id
      and t.approver_id = auth.uid()
  )
  or has_role(auth.uid(), 'hr')
);
```

2. **approval_tasks / approval_records 只允许审批人本人操作**，但**状态流转不要让前端直接 update**，而是走 Edge Function（用 service role），避免有人绕过引擎逻辑直接改数据库状态。

3. 流程设计器、模板管理这类配置操作，RLS 要限制在管理员角色。

## 五、Vue3 前端模块设计

对应你列的 OA 目录结构，建议这样组织：

```
src/modules/workflow/
├── views/
│   ├── MyTodo.vue          // 我的待办 -> approval_tasks where approver_id=me, status=pending
│   ├── MyApplications.vue  // 我的申请/我发起的 -> workflow_instances where initiator_id=me
│   ├── MyApproved.vue      // 我审批的 -> approval_records where approver_id=me
│   ├── FlowCenter.vue      // 流程中心：所有可发起的模板列表
│   ├── FlowDesigner.vue    // 流程设计器
│   ├── TemplateManage.vue  // 模板管理
│   ├── ApprovalAuth.vue    // 审批权限配置
│   ├── FlowMonitor.vue     // 流程监控（运行中实例的实时状态）
│   ├── ApprovalStats.vue   // 审批统计（时长、通过率等）
│   └── SystemLog.vue
├── components/
│   ├── DynamicForm.vue     // 根据 form_schema 渲染业务表单
│   ├── FlowChart.vue       // 流程图可视化（可用 @vue-flow/core）
│   ├── ApprovalTimeline.vue // 审批记录时间线
│   └── ApprovalActionBar.vue // 同意/拒绝/转办/加签按钮
├── composables/
│   ├── useWorkflowEngine.ts // 封装 submit/approve/reject 等 Edge Function 调用
│   └── useRealtimeTodo.ts   // Supabase Realtime 订阅待办数量变化
└── stores/
    └── workflow.ts
```

**几个实现细节建议**：

- **流程设计器**：不用一上来就做拖拽画布那么复杂，可以先做"节点列表 + 表单配置"的简化版（类似钉钉审批模板配置），后期有精力再上 `@vue-flow/core` 做可视化拖拽。
- **动态表单**：`form_schema` 用类似这样的结构，前端写一个 `DynamicForm.vue` 用 `<component :is>` 动态渲染字段类型（input/select/date/user-picker/amount）：
```json
{ "fields": [
  { "key": "days", "label": "请假天数", "type": "number", "required": true },
  { "key": "reason", "label": "事由", "type": "textarea" }
]}
```
- **待办红点/实时刷新**：用 Supabase Realtime 订阅 `approval_tasks` 表的 INSERT/UPDATE（filter: `approver_id=eq.当前用户`），左侧菜单的"我的待办"数字直接响应式更新，不用轮询。
- **审批统计**：这类聚合查询建议不要在前端拉全量数据算，用 Postgres 视图或者定时刷新的物化视图（`refresh materialized view`），比如"各流程平均审批时长"、"各部门通过率"。

## 六、落地建议（结合你现在的技术栈）

给你个渐进式的实施顺序，避免一上来就想把引擎做全：

1. **第一步**：先把 workflow_instances / instance_nodes / approval_tasks / approval_records 这套表打好，硬编码一个最简单的"单人审批"流程跑通请假模块的提交→审批→通知全链路。
2. **第二步**：加入条件分支和或签/会签，把 flow_schema 的 JSON 结构确定下来。
3. **第三步**：做模板管理界面，让流程配置从"硬编码 JSON"变成"界面配置生成 JSON"。
4. **第四步**：把采购、报销等模块接入，验证复用性。
5. **最后**：流程监控、统计报表、流程设计器可视化这些锦上添花的模块。
