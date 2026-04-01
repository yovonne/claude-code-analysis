# ScheduleCronTool (CronCreate, CronDelete, CronList)

## 用途

ScheduleCronTool 系列提供基于 cron 的提示调度功能，支持 Claude Code 中的循环调度和一次性提醒，以及可选的磁盘持久化以实现跨会话持久性。该系统使用用户本地时区的标准 5 字段 cron 表达式。

## 位置

- `restored-src/src/tools/ScheduleCronTool/CronCreateTool.ts` — 创建 cron 作业（158 行）
- `restored-src/src/tools/ScheduleCronTool/CronDeleteTool.ts` — 取消 cron 作业（96 行）
- `restored-src/src/tools/ScheduleCronTool/CronListTool.ts` — 列出活动的 cron 作业（98 行）
- `restored-src/src/tools/ScheduleCronTool/prompt.ts` — 提示、描述和功能门控（136 行）
- `restored-src/src/tools/ScheduleCronTool/UI.tsx` — 所有三个工具的 UI 渲染

## 主要导出

### CronCreateTool

| 导出 | 描述 |
|--------|-------------|
| `CronCreateTool` | 用于创建 cron 作业的工具 |
| `CreateOutput` | 输出类型：`{ id, humanSchedule, recurring, durable? }` |
| `CRON_CREATE_TOOL_NAME` | 常量：`'CronCreate'` |
| `DEFAULT_MAX_AGE_DAYS` | 循环任务的自动过期时间 |

### CronDeleteTool

| 导出 | 描述 |
|--------|-------------|
| `CronDeleteTool` | 用于取消 cron 作业的工具 |
| `DeleteOutput` | 输出类型：`{ id }` |
| `CRON_DELETE_TOOL_NAME` | 常量：`'CronDelete'` |

### CronListTool

| 导出 | 描述 |
|--------|-------------|
| `CronListTool` | 用于列出 cron 作业的工具 |
| `ListOutput` | 输出类型：`{ jobs: [{ id, cron, humanSchedule, prompt, recurring?, durable? }] }` |
| `CRON_LIST_TOOL_NAME` | 常量：`'CronList'` |

## 功能门控

所有三个工具共享相同的启用门控：

```typescript
isEnabled() {
  return isKairosCronEnabled()
}
```

`isKairosCronEnabled()` 组合了：
- 构建时 `feature('AGENT_TRIGGERS')` 标志
- 运行时 `tengu_kairos_cron` GrowthBook 门控（5 分钟刷新）
- `CLAUDE_CODE_DISABLE_CRON` 环境变量覆盖（关闭整个调度器）

默认为 `true` — /loop 已 GA。GrowthBook 对 Bedrock/Vertex/Foundry 和禁用遥测时禁用。

### 持久化 Cron

`isDurableCronEnabled()` 单独控制磁盘持久化：
- 默认为 `true`
- 比 `isKairosCronEnabled()` 更窄的kill switch
- 禁用时，在调用站点强制 `durable: false`

## CronCreateTool

### 输入 Schema

```typescript
{
  cron: string,           // 5 字段 cron："M H DoM Mon DoW"（例如 "*/5 * * * *"）
  prompt: string,         // 每次触发时排入队列的提示
  recurring?: boolean,    // true（默认）= 循环，false = 一次性
  durable?: boolean,      // true = 持久化到磁盘，false（默认）= 仅会话
}
```

### 输出 Schema

```typescript
{
  id: string,             // 用于 CronDelete 的作业 ID
  humanSchedule: string,  // 人类可读的调度描述
  recurring: boolean,     // 作业是否循环
  durable?: boolean,      // 作业是否持久化到磁盘
}
```

### 验证

| 检查 | 错误代码 | 消息 |
|-------|-----------|---------|
| 无效 cron 表达式 | 1 | "Invalid cron expression. Expected 5 fields: M H DoM Mon DoW." |
| 明年没有匹配日期 | 2 | "Cron expression does not match any calendar date in the next year." |
| 作业太多（最多 50） | 3 | "Too many scheduled jobs (max 50). Cancel one first." |
| 队友的持久化 cron | 4 | "durable crons are not supported for teammates" |

### 执行

1. 解析和验证 cron 表达式
2. 检查作业数量限制（MAX_JOBS = 50）
3. 检查队友持久化限制
4. 调用 `addCronTask(cron, prompt, recurring, durable, agentId)`
5. 启用预定任务（`setScheduledTasksEnabled(true)`）
6. 返回作业 ID 和人类可读的调度

### 持久性

| 模式 | 存储 | 重启后存活 |
|------|---------|-----------------|
| `durable: false`（默认） | 内存会话存储 | 否 |
| `durable: true` | `.claude/scheduled_tasks.json` | 是 |

队友不支持持久化 cron，因为队友不会跨会话持久化——持久化队友 cron 会在重启后孤立。

## CronDeleteTool

### 输入 Schema

```typescript
{
  id: string,    // CronCreate 的作业 ID
}
```

### 输出 Schema

```typescript
{
  id: string,    // 已删除的作业 ID
}
```

### 验证

| 检查 | 错误代码 | 消息 |
|-------|-----------|---------|
| 作业 ID 未找到 | 1 | "No scheduled job with id '...'" |
| 被其他代理拥有 | 2 | "Cannot delete cron job '...': owned by another agent" |

### 队友限制

队友只能删除自己的 cron 作业。团队负责人（无队友上下文）可以删除任何作业。

### 执行

1. 验证作业 ID 存在
2. 验证所有权（队友只能删除自己的作业）
3. 调用 `removeCronTasks([id])`
4. 返回已删除的作业 ID

## CronListTool

### 输入 Schema

```typescript
{}  // 无输入参数
```

### 输出 Schema

```typescript
{
  jobs: [{
    id: string,
    cron: string,
    humanSchedule: string,
    prompt: string,
    recurring?: boolean,
    durable?: boolean,
  }]
}
```

### 属性

- **只读**：是 — `isReadOnly()` 返回 true
- **并发安全**：是
- **团队过滤**：队友只看到自己的作业；团队负责人看到所有

### 输出格式化

作业格式为：
```
{id} — {humanSchedule}{(recurring) | (one-shot)}{[session-only]}: {truncated prompt}
```

如果没有作业：`"No scheduled jobs."`

## 调度器行为

### 抖动

调度器添加确定性抖动以防止 API 请求聚集：

| 任务类型 | 抖动 |
|-----------|--------|
| 循环 | 最多延迟周期的 10%（最多 15 分钟） |
| 一次性在 :00 或 :30 | 最多提前 90 秒 |

### 自动过期

循环任务在 `DEFAULT_MAX_AGE_DAYS` 天后自动过期。它们触发最后一次，然后被删除。这限制了会话生命周期。

### 触发条件

作业仅在 REPL 空闲时触发（不在查询中途）。这防止中断活跃对话。

## Cron 表达式格式

本地时区的标准 5 字段 cron：

```
M H DoM Mon DoW
```

| 字段 | 范围 | 示例 |
|-------|-------|---------|
| 分钟 | 0-59 | `*/5`（每 5 分钟），`30` |
| 小时 | 0-23 | `14`（下午 2 点），`*` |
| 日期 | 1-31 | `28`，`*` |
| 月份 | 1-12 | `2`（2 月），`*` |
| 星期 | 0-7 | `1-5`（工作日），`*` |

### 示例

| 表达式 | 含义 |
|------------|---------|
| `*/5 * * * *` | 每 5 分钟 |
| `30 14 28 2 *` | 2 月 28 日下午 2:30（每年一次） |
| `0 9 * * 1-5` | 工作日上午 9 点 |
| `0 * * * *` | 每小时 |
| `57 8 * * *` | 每天上午 8:57（避免 :00） |

### 非整点指导

为避免 API 请求聚集，当用户的请求是近似时，选择**不是** 0 或 30 的分钟：

- "每天早上 9 点左右" → `"57 8 * * *"` 或 `"3 9 * * *"`（不是 `"0 9 * * *"`）
- "每小时" → `"7 * * * *"`（不是 `"0 * * * *"`）

仅当用户指定确切时间时才使用 0 或 30。

## 一次性 vs 循环

### 一次性（recurring: false）

用于"在 X 提醒我"或"在 <时间> 做 Y"请求：
- 在下一个匹配时间触发一次
- 触发后自动删除
- 将分钟/小时/日期/月份固定到特定值

### 循环（recurring: true，默认）

用于"每 N 分钟"/"每小时"/"工作日上午 9 点"请求：
- 每次 cron 匹配时触发
- 在 DEFAULT_MAX_AGE_DAYS 天后自动过期
- 使用 CronDelete 可以提前取消

## 依赖

| 模块 | 用途 |
|--------|---------|
| `utils/cron.ts` | Cron 表达式解析和人类可读转换 |
| `utils/cronTasks.ts` | Cron 任务 CRUD、文件管理、下次运行计算 |
| `utils/teammateContext.ts` | 用于所有权过滤的队友上下文 |
| `utils/envUtils.ts` | 环境变量检查 |
| `services/analytics/growthbook.ts` | 功能标志评估 |
| `bootstrap/state.js` | 计划任务启用 |
| `utils/lazySchema.ts` | 延迟 Zod schema 定义 |
| `utils/semanticBoolean.ts` | 语义布尔解析 |
