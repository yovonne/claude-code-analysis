# ExitPlanModeTool

## 目的

ExitPlanModeTool（ExitPlanModeV2Tool）向用户呈现完成的计划以供批准，并将会话从计划模式转换回之前的模式。它处理计划文件持久化、基于团队的批准工作流，并恢复进入计划模式时被剥离的完整权限集。该工具支持直接用户批准和多代理团队负责人批准流程。

## 位置

- `restored-src/src/tools/ExitPlanModeTool/ExitPlanModeV2Tool.ts` — 主工具定义和调用逻辑（494 行）
- `restored-src/src/tools/ExitPlanModeTool/prompt.ts` — 工具提示（30 行）
- `restored-src/src/tools/ExitPlanModeTool/UI.tsx` — 终端 UI 渲染（82 行）
- `restored-src/src/tools/ExitPlanModeTool/constants.ts` — 工具名称常量（3 行）

## 关键导出

| 导出 | 描述 |
|--------|-------------|
| `ExitPlanModeV2Tool` | 通过 `buildTool()` 构建的完整工具定义 |
| `Output` | 输出类型：`{ plan, isAgent, filePath, hasTaskTool?, planWasEdited?, awaitingLeaderApproval?, requestId? }` |
| `AllowedPrompt` | 基于提示的权限请求的类型：`{ tool: 'Bash', prompt: string }` |
| `_sdkInputSchema` | 带注入 `plan` 和 `planFilePath` 字段的 SDK 面向 schema |
| `EXIT_PLAN_MODE_TOOL_NAME` | 常量字符串 `'ExitPlanMode'` |
| `EXIT_PLAN_MODE_V2_TOOL_NAME` | 常量字符串 `'ExitPlanMode'`（相同名称，V2 实现） |

## 输入/输出 Schema

### 输入 Schema

```typescript
{
  allowedPrompts?: {
    tool: 'Bash',
    prompt: string  // 语义描述如 "run tests"、"install dependencies"
  }[]
}
```

输入接受可选的基于提示的权限，这些是计划实施所需的。这些描述动作类别而非特定命令，允许用户授予广泛权限桶以供计划执行。

### SDK 输入 Schema（扩展）

```typescript
{
  allowedPrompts?: { tool: 'Bash', prompt: string }[]
  plan?: string          // 通过 normalizeToolInput 从磁盘注入
  planFilePath?: string  // 通过 normalizeToolInput 注入
}
```

SDK 看到通过 `normalizeToolInput` 注入的其他字段 — 计划内容从磁盘读取而非作为参数传递。

### 输出 Schema

```typescript
{
  plan: string | null,                // 呈现的计划
  isAgent: boolean,                   // 是否从代理上下文调用
  filePath?: string,                  // 计划保存的路径
  hasTaskTool?: boolean,              // Agent 工具是否可用
  planWasEdited?: boolean,            // 用户是否编辑了计划（CCR 或 Ctrl+G）
  awaitingLeaderApproval?: boolean,   // 队友发送计划给团队负责人
  requestId?: string,                 // 计划批准请求的唯一 ID
}
```

## 模式转换

### 退出流程

```
1. 模型调用 ExitPlanMode
   - 可选：{ allowedPrompts: [...] }

2. 验证（validateInput）
   - 对于队友：始终通过（负责人处理批准）
   - 对于非队友：验证 mode === 'plan'
   - 如果不在计划模式则用错误代码 1 拒绝

3. 权限检查（checkPermissions）
   - 队友：'allow'（绕过 UI，mailbox 处理批准）
   - 非队友：'ask'（显示确认对话框）

4. 计划检索
   - 通过 getPlan(agentId) 从磁盘读取计划
   - 如果 CCR web UI 通过 input.plan 发送了编辑的计划，则先写入磁盘
   - 为远程上下文持久化文件快照

5. 队友批准流程（如适用）
   - 如果 isTeammate() && isPlanModeRequired()：
     a. 生成唯一请求 ID
     b. 构建 plan_approval_request 消息
     c. 通过 writeToMailbox() 写入团队负责人 mailbox
     d. 更新任务状态显示"等待批准"
     e. 返回带 awaitingLeaderApproval: true

6. 模式恢复（非队友或自愿计划模式）
   - 从状态中检索 prePlanMode
   - 检查自动模式门控状态（断路器防御）
   - 恢复权限（危险规则）
   - 更新状态：mode = prePlanMode，prePlanMode = undefined
   - 设置 hasExitedPlanMode 标志
   - 设置 needsPlanModeExitAttachment 标志

7. 返回结果
   - 计划内容、文件路径和元数据
   - mapToolResultToToolResultBlockParam() 格式化响应
```

## 计划文件处理

### 计划来源优先级

1. **CCR Web UI 编辑**：如果用户通过 CCR web 界面编辑了计划，编辑的内容通过 `permissionResult.updatedInput` 到达，并在处理前写入磁盘
2. **磁盘回退**：如果没有提供编辑的计划，通过 `getPlan(agentId)` 从磁盘读取

### 磁盘同步

当提供编辑的计划时：
```typescript
if (inputPlan !== undefined && filePath) {
  await writeFile(filePath, inputPlan, 'utf-8')
  void persistFileSnapshotIfRemote()
}
```

这确保 `VerifyPlanExecution` 和 `Read` 工具看到用户编辑的版本。快照被重新保存，因为较早的快照（在 `normalizeToolInput` 中捕获）包含旧计划。

### 计划文件验证

对于 `isPlanModeRequired()` 为 true 的队友，工具在继续之前验证计划文件存在：
```typescript
if (!plan) {
  throw new Error(`No plan file found at ${filePath}...`)
}
```

## 基于团队的批准工作流

### 队友检测

工具使用 `isTeammate()` 确定当前会话是否作为团队成员代理运行。队友有与独立会话不同的批准流程。

### 计划批准请求

当具有必需计划模式的队友退出规划时：

```typescript
const approvalRequest = {
  type: 'plan_approval_request',
  from: agentName,
  timestamp: new Date().toISOString(),
  planFilePath: filePath,
  planContent: plan,
  requestId,
}

await writeToMailbox('team-lead', {
  from: agentName,
  text: jsonStringify(approvalRequest),
  timestamp: new Date().toISOString(),
}, teamName)
```

请求被发送到团队负责人的 mailbox。然后队友等待响应再继续实施。

### 任务状态更新

对于进程内队友，任务状态更新以显示批准状态：
```typescript
setAwaitingPlanApproval(agentTaskId, context.setAppState, true)
```

这允许团队负责人看到哪些任务正在等待计划批准。

### 批准响应处理

当团队负责人批准或拒绝时，响应到达队友的 mailbox。队友处理响应，然后要么继续实施，要么完善计划。

## 模式恢复

### 计划前模式恢复

工具恢复进入计划模式之前处于活动状态的模式：

```typescript
let restoreMode = prev.toolPermissionContext.prePlanMode ?? 'default'
```

### 自动模式门控防御

断路器防止通过计划模式绕过自动模式限制：

```typescript
if (restoreMode === 'auto' && !isAutoModeGateEnabled()) {
  restoreMode = 'default'  // 如果自动模式不可用则回退到默认
}
```

没有这个防御，ExitPlanMode 可以通过直接调用 `setAutoModeActive(true)` 来绕过断路器。

### 权限恢复

当恢复到非自动模式时，恢复在计划模式进入时被剥离的危险权限：

```typescript
if (!restoringToAuto && prev.toolPermissionContext.strippedDangerousRules) {
  baseContext = restoreDangerousPermissions(baseContext)
}
```

当恢复到自动模式时，危险权限保持剥离：
```typescript
if (restoringToAuto) {
  baseContext = stripDangerousPermissionsForAutoMode(baseContext)
}
```

### 状态标志

| 标志 | 设置时机 | 用途 |
|------|----------|---------|
| `hasExitedPlanMode` | 任何成功退出 | 跟踪计划模式在此会话中已退出 |
| `needsPlanModeExitAttachment` | 任何成功退出 | 在响应中触发计划模式退出附件 |
| `needsAutoModeExitAttachment` | 自动模式在计划期间使用但恢复到默认 | 信号自动模式转换清理 |

## 结果块格式化

### 队友等待批准

```
Your plan has been submitted to the team lead for approval.

Plan file: ${filePath}

**What happens next:**
1. Wait for the team lead to review your plan
2. You will receive a message in your inbox with approval/rejection
3. If approved, you can proceed with implementation
4. If rejected, refine your plan based on the feedback

**Important:** Do NOT proceed until you receive approval. Check your inbox for response.

Request ID: ${requestId}
```

### 代理上下文

```
User has approved the plan. There is nothing else needed from you now. Please respond with "ok"
```

### 空计划

```
User has approved exiting plan mode. You can now proceed.
```

### 标准计划批准

```
User has approved your plan. You can now start coding. Start with updating your todo list if applicable

Your plan has been saved to: ${filePath}
You can refer back to it if needed during implementation.

## Approved Plan [edited by user]:
${plan}
```

### 团队并行化提示

当 Agent 工具可用时，附加提示：
```
If this plan can be broken down into multiple independent tasks, consider using the TeamCreate tool to create a team and parallelize the work.
```

## UI 渲染

### 工具使用消息

返回 `null` — 不显示中间"using"消息。

### 工具结果消息

基于计划状态的三种变体：

**空计划：**
```tsx
<Box flexDirection="column" marginTop={1}>
  <Box flexDirection="row">
    <Text color={getModeColor('plan')}>{BLACK_CIRCLE}</Text>
    <Text> Exited plan mode</Text>
  </Box>
</Box>
```

**等待负责人批准：**
```tsx
<Box flexDirection="column" marginTop={1}>
  <Box flexDirection="row">
    <Text color={getModeColor('plan')}>{BLACK_CIRCLE}</Text>
    <Text> Plan submitted for team lead approval</Text>
  </Box>
  <MessageResponse>
    <Box flexDirection="column">
      {filePath && <Text dimColor>Plan file: {displayPath}</Text>}
      <Text dimColor>Waiting for team lead to review and approve...</Text>
    </Box>
  </MessageResponse>
</Box>
```

**用户批准计划：**
```tsx
<Box flexDirection="column" marginTop={1}>
  <Box flexDirection="row">
    <Text color={getModeColor('plan')}>{BLACK_CIRCLE}</Text>
    <Text> User approved Claude&apos;s plan</Text>
  </Box>
  <MessageResponse>
    <Box flexDirection="column">
      {filePath && <Text dimColor>Plan saved to: {displayPath} · /plan to edit</Text>}
      <Markdown>{plan}</Markdown>
    </Box>
  </MessageResponse>
</Box>
```

### 工具使用拒绝消息

```tsx
<Box flexDirection="column">
  <RejectedPlanMessage plan={planContent} />
</Box>
```

渲染带计划内容的 `RejectedPlanMessage` 组件，显示用户的拒绝反馈。

## 验证和错误处理

### 输入验证

| 条件 | 结果 | 错误代码 |
|-----------|--------|------------|
| 不在计划模式 | 拒绝并显示消息 | 1 |
| 队友上下文 | 始终通过（负责人处理批准） | — |
| 无法验证 worktree 状态 | 拒绝（失败关闭） | 3 |

### 权限检查

| 上下文 | 行为 | 理由 |
|---------|----------|-----------|
| 队友 | `allow` — 绕过 UI | Mailbox 异步处理批准 |
| 非队友 | `ask` — 显示对话框 | 用户必须明确批准退出计划模式 |

### 模式门控

与 EnterPlanMode 相同的门控 — 当 `--channels` 功能标志激活并配置了通道时被禁用。这防止模型进入在消息平台上无法退出的模式。

## 提示指导

### 使用时机

- 在将计划写入计划文件并准备好用户批准之后
- 仅用于实施规划任务（不是研究/探索）

### 使用之前

- 计划必须完整且明确
- 如果有未解决的方法问题，先使用 AskUserQuestion
- 不要使用 AskUserQuestion 询问"这个计划可以吗？" — 这是 ExitPlanMode 的作用

### 示例

**好：** "Help me implement yank mode for vim" — 在规划实施步骤后使用 ExitPlanMode

**坏：** "Search for and understand the implementation of vim mode" — 不要使用，这是研究而非规划

## 集成点

### `bootstrap/state.js`

- `getAllowedChannels()` — 用于门控的通道列表
- `hasExitedPlanModeInSession()` — 会话级退出跟踪
- `setHasExitedPlanMode(bool)` — 标记计划模式已退出
- `setNeedsPlanModeExitAttachment(bool)` — 触发退出附件
- `setNeedsAutoModeExitAttachment(bool)` — 信号自动模式清理

### `utils/plans.js`

- `getPlan(agentId)` — 从磁盘读取计划内容
- `getPlanFilePath(agentId)` — 解析计划文件路径
- `persistFileSnapshotIfRemote()` — 同步计划到远程存储

### `utils/teammate.js`

- `isTeammate()` — 检查是否作为团队成员运行
- `isPlanModeRequired()` — 检查此代理是否需要计划模式
- `getAgentName()` / `getTeamName()` — 代理/团队标识

### `utils/teammateMailbox.js`

- `writeToMailbox(recipient, message, teamName)` — 发送批准请求给团队负责人

### `utils/inProcessTeammateHelpers.js`

- `findInProcessTeammateTaskId(agentName, state)` — 查找状态更新的任务 ID
- `setAwaitingPlanApproval(taskId, setState, bool)` — 更新任务批准状态

### `utils/agentId.js`

- `formatAgentId(name, team)` — 格式化代理标识符
- `generateRequestId(type, agentId)` — 生成唯一请求 ID

### `utils/permissions/autoModeState.js`（条件）

- `isAutoModeActive()` — 检查当前自动模式状态
- `setAutoModeActive(bool)` — 设置自动模式状态
- `isAutoModeGateEnabled()` — 检查自动模式门控是否激活
- `getAutoModeUnavailableReason()` — 获取自动模式不可用的原因
- `getAutoModeUnavailableNotification(reason)` — 获取用户通知文本
- `stripDangerousPermissionsForAutoMode(context)` — 为自动模式剥离权限
- `restoreDangerousPermissions(context)` — 恢复先前剥离的权限

### `utils/permissions/permissionSetup.js`（条件）

- 当 `TRANSCRIPT_CLASSIFIER` 功能启用时动态加载

## 数据流

```
Model has completed planning
    │
    ▼
ExitPlanMode tool call { allowedPrompts? }
    │
    ▼
┌─────────────────────────────────────────┐
│ 验证                              │
│ - 队友：通过                    │
│ - 非队友：验证 mode === 'plan'  │
└─────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────┐
│ 权限检查                        │
│ - 队友：允许（绕过 UI）           │
│ - 非队友：询问（显示对话框）       │
└─────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────┐
│ 计划检索                          │
│ - 从磁盘读取                        │
│ - 如果提供则写入编辑的计划         │
│ - 为远程持久化快照           │
└─────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────┐
│ 队友流程？                          │
│                                         │
│ 是：                                    │
│   - 构建批准请求              │
│   - 发送到团队负责人 mailbox           │
│   - 更新任务状态                   │
│   - 返回带 awaitingLeaderApproval  │
│                                         │
│ 否：                                     │
│   - 继续模式恢复         │
└─────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────┐
│ 模式恢复                        │
│                                         │
│ - 检索 prePlanMode                  │
│ - 检查自动模式门控（断路器）│
│ - 恢复/剥离权限             │
│ - 更新状态：mode = prePlanMode      │
│ - 设置退出标志                        │
└─────────────────────────────────────────┘
    │
    ▼
Result block with approved plan content
    │
    ▼
Model begins implementation phase
```
