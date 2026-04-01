# 工具系统

## 目的

工具系统是定义 Claude Code 如何与外部世界交互的核心抽象层。每个能力 — 文件操作、shell 执行、agent 生成、web 搜索、MCP 集成 — 都被建模为 `Tool`。工具统一类型化、通过 Zod schemas 验证、由功能标志门控、由权限规则过滤，并在终端 UI 中渲染为 React 组件。这种设计实现：

- **统一接口**：无论功能如何，所有工具共享相同的形状
- **可组合权限**：权限规则一致地应用于所有工具
- **功能标志门控**：工具可以有条件地加载，无需代码更改
- **渐进式披露**：通过 ToolSearch 延迟工具加载减少提示令牌使用
- **丰富的 UI 渲染**：每个工具控制自己的使用/结果/进度/错误消息渲染

## 位置

- `restored-src/src/Tool.ts` — 核心 `Tool` 接口、`buildTool` 工厂和共享类型（约 793 行）
- `restored-src/src/tools.ts` — 工具注册表、`getTools()`、`assembleToolPool()`、功能标志门控（约 390 行）
- `restored-src/src/constants/tools.ts` — 每个代理模式的工具允许/拒绝列表（约 113 行）
- `restored-src/src/constants/toolLimits.ts` — 结果大小和令牌限制（约 57 行）
- `restored-src/src/types/permissions.ts` — 权限类型、决策和规则
- `restored-src/src/hooks/useCanUseTool.tsx` — 权限决策 hook
- `restored-src/src/tools/` — 单个工具实现（60+ 工具）
- `restored-src/src/tools/shared/` — 共享工具实用程序（spawn、git 跟踪）
- `restored-src/src/tools/testing/` — 仅测试工具

## 关键导出

### 从 Tool.ts

| 导出 | 描述 |
|--------|-------------|
| `Tool<Input, Output, P>` | 核心工具接口 — 每个工具必须实现此形状 |
| `ToolDef<Input, Output, P>` | `buildTool` 接受的部分工具定义；可默认的方法是可选的 |
| `Tools` | `readonly Tool[]` — 类型化集合，优于原始数组 |
| `ToolInputJSONSchema` | 用于绕过 Zod 转换的 MCP 工具的原始 JSON Schema 格式 |
| `ValidationResult` | `{ result: true }` 或 `{ result: false, message, errorCode }` |
| `ToolResult<T>` | `{ data: T, newMessages?, contextModifier?, mcpMeta? }` |
| `ToolUseContext` | 传递给每个工具调用的巨大上下文对象 — 包含 AppState、中止控制器、JSX 设置器、消息列表等 |
| `ToolPermissionContext` | 深度不可变的权限状态：模式、规则、工作目录 |
| `ToolCallProgress<P>` | 用于流式进度更新的回调类型 |
| `ToolProgressData` | 所有特定工具进度类型的联合 |
| `buildTool(def)` | 用 `TOOL_DEFAULTS` 合并 `def` 以产生完整 `Tool` 的工厂 |
| `toolMatchesName(tool, name)` | 检查工具是否通过主名称或别名匹配 |
| `findToolByName(tools, name)` | 从列表中按名称或别名查找工具 |
| `getEmptyToolPermissionContext()` | 返回带所有默认值的空白权限上下文 |

**重新导出的进度类型**：`AgentToolProgress`、`BashProgress`、`MCPProgress`、`REPLToolProgress`、`SkillToolProgress`、`TaskOutputProgress`、`WebSearchProgress`

### 从 tools.ts

| 导出 | 描述 |
|--------|-------------|
| `getTools(permissionContext)` | 返回过滤后的内置工具，尊重拒绝规则、REPL 模式和 `isEnabled()` |
| `getAllBaseTools()` | 返回所有可能工具的详尽列表（过滤前） |
| `assembleToolPool(permissionContext, mcpTools)` | 组合内置 + MCP 工具，去重和排序 |
| `getMergedTools(permissionContext, mcpTools)` | 内置 + MCP 工具的简单连接 |
| `filterToolsByDenyRules(tools, permissionContext)` | 过滤被权限规则全面拒绝的工具 |
| `TOOL_PRESETS` | 预定义工具预设（目前仅 `'default'`） |
| `getToolsForDefaultPreset()` | 返回默认预设的工具名称 |

**重新导出的常量**：`ALL_AGENT_DISALLOWED_TOOLS`、`CUSTOM_AGENT_DISALLOWED_TOOLS`、`ASYNC_AGENT_ALLOWED_TOOLS`、`COORDINATOR_MODE_ALLOWED_TOOLS`、`REPL_ONLY_TOOLS`

## 工具接口

`Tool` 接口是每个工具必须满足的契约。它由 `Input`（Zod schema）、`Output`（结果类型）和 `P`（进度数据类型）参数化：

```typescript
export type Tool<
  Input extends AnyObject = AnyObject,
  Output = unknown,
  P extends ToolProgressData = ToolProgressData,
> = {
  // 标识
  readonly name: string
  aliases?: string[]
  searchHint?: string          // 3-10 词短语用于 ToolSearch 关键字匹配

  // 必需方法
  call(args, context, canUseTool, parentMessage, onProgress): Promise<ToolResult<Output>>
  description(input, options): Promise<string>
  prompt(options): Promise<string>
  checkPermissions(input, context): Promise<PermissionResult>
  mapToolResultToToolResultBlockParam(content, toolUseID): ToolResultBlockParam
  renderToolUseMessage(input, options): React.ReactNode
  userFacingName(input): string

  // Schemas
  readonly inputSchema: Input
  readonly inputJSONSchema?: ToolInputJSONSchema
  outputSchema?: z.ZodType<unknown>

  // 行为谓词
  isEnabled(): boolean
  isConcurrencySafe(input): boolean
  isReadOnly(input): boolean
  isDestructive?(input): boolean
  interruptBehavior?(): 'cancel' | 'block'
  isOpenWorld?(input): boolean
  requiresUserInteraction?(): boolean

  // 验证
  validateInput?(input, context): Promise<ValidationResult>
  inputsEquivalent?(a, b): boolean

  // 权限匹配
  preparePermissionMatcher?(input): Promise<(pattern) => boolean>

  // UI 渲染（全部可选）
  renderToolResultMessage?(content, progressMessages, options): React.ReactNode
  renderToolUseProgressMessage?(progressMessages, options): React.ReactNode
  renderToolUseQueuedMessage?(): React.ReactNode
  renderToolUseRejectedMessage?(input, options): React.ReactNode
  renderToolUseErrorMessage?(result, options): React.ReactNode
  renderGroupedToolUse?(toolUses, options): React.ReactNode | null
  renderToolUseTag?(input): React.ReactNode
  userFacingNameBackgroundColor?(input): keyof Theme | undefined

  // 摘要和描述
  getToolUseSummary?(input): string | null
  getActivityDescription?(input): string | null
  isSearchOrReadCommand?(input): { isSearch, isRead, isList }
  toAutoClassifierInput(input): unknown

  // 可搜索性
  extractSearchText?(out: Output): string
  isResultTruncated?(output): boolean

  // 加载行为
  readonly shouldDefer?: boolean      // 通过 ToolSearch 延迟加载
  readonly alwaysLoad?: boolean       // 从不延迟，始终包含
  readonly strict?: boolean           // tengu_tool_pear 的严格模式

  // MCP 元数据
  isMcp?: boolean
  isLsp?: boolean
  mcpInfo?: { serverName: string; toolName: string }

  // 输入变更
  backfillObservableInput?(input): void

  // 大小限制
  maxResultSizeChars: number

  // 透明度
  isTransparentWrapper?(): boolean
}
```

## 工具默认值

`buildTool` 工厂为常被省略的方法填充安全默认值：

```typescript
const TOOL_DEFAULTS = {
  isEnabled: () => true,
  isConcurrencySafe: (_input?) => false,    // 失败关闭：假设不安全
  isReadOnly: (_input?) => false,           // 失败关闭：假设写入
  isDestructive: (_input?) => false,
  checkPermissions: (input) =>              // 推迟到通用权限系统
    Promise.resolve({ behavior: 'allow', updatedInput: input }),
  toAutoClassifierInput: (_input?) => '',   // 默认跳过分类器
  userFacingName: (_input?) => '',
}
```

**关键设计**：默认值在重要的地方是失败关闭的 — 工具被假定为写操作、非并发安全、并被自动模式分类器跳过，除非明确覆盖。

## 工具生命周期

```
1. 模块加载
   - 工具模块导入（静态或动态）
   - buildTool() 用 ToolDef -> 完整 Tool 对象调用
   - Zod 输入 schema 定义（通常通过 lazySchema）

2. 注册
   - getAllBaseTools() 组装所有工具实例
   - 功能标志门控条件工具包含
   - getTools() 按拒绝规则 + isEnabled() 过滤

3. 提示构建
   - tool.prompt() 生成模型的系统提示文本
   - tool.description() 生成运行时描述
   - 输入 schemas 序列化为 API 工具定义的 JSON
   - 延迟工具（shouldDefer=true）从初始提示中排除

4. 模型调用
   - 模型返回带名称 + 输入的 tool_use 块

5. 工具解析
   - findToolByName(tools, name) 按名称或别名定位工具
   - 输入根据 Zod inputSchema 验证

6. 验证
   - 如果定义则调用 tool.validateInput(input, context)
   - 返回 ValidationResult（通过/失败及错误代码）

7. 权限检查
   - 调用 canUseTool(tool, input, context, ...)
   - tool.checkPermissions(input, context) 返回 PermissionResult
   - 权限系统评估：规则 -> 模式 -> 钩子 -> 分类器
   - 决策：allow / deny / ask（显示 UI 提示）

8. 执行
   - tool.call(args, context, canUseTool, parentMessage, onProgress)
   - 工具执行其操作（文件 I/O、shell exec、agent 生成等）
   - 进度事件通过 onProgress 回调发出
   - 结果包装在 ToolResult<T> 中

9. 结果处理
   - tool.mapToolResultToToolResultBlockParam() 序列化为 API
   - 结果大小检查与 maxResultSizeChars
   - 超大结果持久化到磁盘并返回预览
   - tool.renderToolResultMessage() 在终端 UI 中渲染
   - 新消息（如果有）注入对话
```

## 工具注册表

### 工具组装管道

```
getAllBaseTools()
  +-- 始终存在的工具（BashTool、FileReadTool、FileEditTool 等）
  +-- 功能标志门控工具（SleepTool、WorkflowTool、WebBrowserTool 等）
  +-- 环境标志门控工具（REPLTool、ConfigTool、TungstenTool）
  +-- 运行时检查门控工具（PowerShellTool、LSPTool、Worktree 工具）
  +--> Tools[]

getTools(permissionContext)
  +-- CLAUDE_CODE_SIMPLE 模式 -> [BashTool、FileReadTool、FileEditTool]
  +-- getAllBaseTools() 过滤：
  |   +-- 移除特殊工具（ListMcpResources、ReadMcpResource、SyntheticOutput）
  |   +-- filterToolsByDenyRules() -> 移除全面拒绝的工具
  |   +-- REPL 模式 -> 隐藏 REPL_ONLY_TOOLS
  |   +-- isEnabled() -> 移除禁用的工具
  +--> Tools[]

assembleToolPool(permissionContext, mcpTools)
  +-- getTools(permissionContext) -> 内置工具
  +-- filterToolsByDenyRules(mcpTools) -> 允许的 MCP 工具
  +-- 按名称排序每个分区（提示缓存稳定性）
  +-- uniqBy(..., 'name') -> 去重（内置工具优先）
  +--> Tools[]
```

### 功能标志门控

工具使用两种机制条件加载：

**1. 来自 `bun:bundle` 的 `feature()`** — 编译时功能标志：

```typescript
const SleepTool =
  feature('PROACTIVE') || feature('KAIROS')
    ? require('./tools/SleepTool/SleepTool.js').SleepTool
    : null

const WorkflowTool = feature('WORKFLOW_SCRIPTS')
  ? (() => {
      require('./tools/WorkflowTool/bundled/index.js').initBundledWorkflows()
      return require('./tools/WorkflowTool/WorkflowTool.js').WorkflowTool
    })()
  : null
```

**2. `process.env` 检查** — 运行时环境标志：

```typescript
const REPLTool =
  process.env.USER_TYPE === 'ant'
    ? require('./tools/REPLTool/REPLTool.js').REPLTool
    : null

const VerifyPlanExecutionTool =
  process.env.CLAUDE_CODE_VERIFY_PLAN === 'true'
    ? require('./tools/VerifyPlanExecutionTool/VerifyPlanExecutionTool.js')
        .VerifyPlanExecutionTool
    : null
```

**3. 运行时工具检查**：

```typescript
...(isPowerShellToolEnabled() ? [getPowerShellTool()] : []),
...(isWorktreeModeEnabled() ? [EnterWorktreeTool, ExitWorktreeTool] : []),
...(isEnvTruthy(process.env.ENABLE_LSP_TOOL) ? [LSPTool] : []),
...(isAgentSwarmsEnabled() ? [getTeamCreateTool(), getTeamDeleteTool()] : []),
```

### 死代码消除

条件 `require()` 调用（而非顶级 `import`）启用 tree-shaking。当功能标志在构建时为 false 时，工具模块永不包含在 bundle 中。ESLint 规则（`custom-rules/no-process-env-top-level`）强制执行此模式。

## 权限系统

### 权限流程

```
模型调用工具
  |
  +- 1. validateInput() -> 失败？向模型返回错误
  |
  +- 2. canUseTool(tool, input, context, ...)
  |     |
  |     +- 提供 forceDecision？-> 使用它
  |     |
  |     +- hasPermissionsToUseTool() -> 评估：
  |     |   +- alwaysAllowRules -> allow
  |     |   +- alwaysDenyRules -> deny
  |     |   +- tool.checkPermissions() -> ask/allow/deny/passthrough
  |     |   +- 权限模式（bypassPermissions -> allow）
  |     |   +- 钩子（PreToolUse/PostToolUse）
  |     |   +- 分类器（自动模式、bash 分类器）
  |     |
  |     +- 决策处理程序：
  |         +- handleCoordinatorPermission() -> 协调器模式
  |         +- handleSwarmWorkerPermission() -> swarm workers
  |         +- handleInteractivePermission() -> REPL/交互式
  |
  +- 3. tool.call() -> 执行
  |
  +- 4. 返回 ToolResult -> 渲染 + 注入消息
```

### PermissionResult 类型

```typescript
type PermissionResult =
  | { behavior: 'allow', updatedInput? }
  | { behavior: 'deny', message, decisionReason? }
  | { behavior: 'ask', message, decisionReason?, suggestions?, pendingClassifierCheck? }
  | { behavior: 'passthrough', message, decisionReason?, suggestions?, blockedPath? }
```

### 权限模式

| 模式 | 行为 |
|------|----------|
| `default` | 在每个工具使用前询问 |
| `bypassPermissions` | 自动批准所有工具（跳过权限） |
| `acceptEdits` | 自动批准只读工具，询问写入 |
| `auto` | 分类器自动批准安全工具，询问风险工具 |
| `plan` | 计划模式 — 仅允许规划工具 |
| `dontAsk` | 从不询问，自动拒绝风险操作 |
| `bubble` | 将权限决策冒泡到父上下文 |

### 工具允许/拒绝列表

在 `constants/tools.ts` 中定义：

- **`ALL_AGENT_DISALLOWED_TOOLS`**：所有子代理阻止的工具（TaskOutput、ExitPlanMode、Agent、AskUserQuestion、TaskStop、Workflow）
- **`ASYNC_AGENT_ALLOWED_TOOLS`**：异步代理的白名单（Read、WebSearch、TodoWrite、Grep、WebFetch、Glob、Shell、Edit、Write、NotebookEdit、Skill、ToolSearch、Worktree）
- **`COORDINATOR_MODE_ALLOWED_TOOLS`**：协调器的工具（Agent、TaskStop、SendMessage、SyntheticOutput）
- **`IN_PROCESS_TEAMMATE_ALLOWED_TOOLS`**：进程内队友的工具（Task CRUD、SendMessage、Cron）

## 工具类别

| 类别 | 工具 | 描述 |
|----------|-------|-------------|
| **Shell 执行** | `BashTool`、`PowerShellTool` | 带沙箱、超时和输出流式处理的 shell 命令运行 |
| **文件读取** | `FileReadTool`、`GlobTool`、`GrepTool` | 读取文件、glob 模式、带二进制/PDF/图像支持的正则搜索 |
| **文件写入** | `FileWriteTool`、`FileEditTool`、`NotebookEditTool` | 创建/覆盖文件、应用编辑、修改 Jupyter notebooks |
| **Agent/任务** | `AgentTool`、`TaskCreateTool`、`TaskGetTool`、`TaskUpdateTool`、`TaskListTool`、`TaskStopTool`、`TaskOutputTool` | 生成子代理、管理任务、检索输出 |
| **团队/Swarm** | `TeamCreateTool`、`TeamDeleteTool`、`SendMessageTool`、`ListPeersTool` | 多代理团队管理和通信 |
| **Web** | `WebFetchTool`、`WebSearchTool`、`WebBrowserTool` | 获取 URL、搜索 web、浏览器自动化 |
| **规划** | `EnterPlanModeTool`、`ExitPlanModeV2Tool` | 在计划和行动模式之间切换 |
| **MCP** | `ListMcpResourcesTool`、`ReadMcpResourceTool` | 模型上下文协议资源访问 |
| **技能** | `SkillTool` | 加载和执行技能定义 |
| **组织** | `TodoWriteTool`、`BriefTool` | 任务跟踪和会话简报 |
| **用户交互** | `AskUserQuestionTool`、`ConfigTool` | 询问澄清问题、管理配置 |
| **搜索** | `ToolSearchTool` | 通过关键字搜索延迟工具发现 |
| **LSP** | `LSPTool` | 语言服务器协议集成 |
| **Worktree** | `EnterWorktreeTool`、`ExitWorktreeTool` | Git worktree 管理 |
| **高级** | `WorkflowTool`、`SleepTool`、`SnipTool`、`CtxInspectTool`、`TerminalCaptureTool` | 功能标志门控的高级能力 |
| **调度** | `CronCreateTool`、`CronDeleteTool`、`CronListTool`、`RemoteTriggerTool` | Cron 作业和远程触发器 |
| **通知** | `PushNotificationTool`、`SendUserFileTool`、`SubscribePRTool` | 推送通知、文件共享、GitHub webhooks |
| **监控** | `MonitorTool` | 主动监控 |
| **测试** | `TestingPermissionTool` | 用于权限对话框测试的仅测试工具 |

## 工具输入/输出 Schemas

### 输入 Schema 模式

工具使用 Zod v4 定义输入 schemas，通常使用 `lazySchema` 进行 tree-shaking：

```typescript
const inputSchema = lazySchema(() => z.object({
  command: z.string().describe('The bash command to execute'),
  timeout: semanticNumber()
    .min(0)
    .max(300)
    .optional()
    .describe('Timeout in seconds'),
}))

type InputSchema = ReturnType<typeof inputSchema>
```

### MCP 的 JSON Schema

MCP 工具可以提供原始 JSON Schema 而非 Zod：

```typescript
readonly inputJSONSchema?: ToolInputJSONSchema
// { type: 'object', properties: { ... } }
```

### 输出 Schema

用于验证工具输出的可选 Zod schema：

```typescript
outputSchema?: z.ZodType<unknown>
```

### 结果序列化

每个工具实现 `mapToolResultToToolResultBlockParam`：

```typescript
mapToolResultToToolResultBlockParam(content: Output, toolUseID: string): ToolResultBlockParam {
  return {
    type: 'tool_result',
    content: String(content),
    tool_use_id: toolUseID,
  }
}
```

## 错误处理

### 验证错误

```typescript
type ValidationResult =
  | { result: true }
  | { result: false; message: string; errorCode: number }
```

工具实现 `validateInput()` 以在权限检查之前捕获错误。验证失败向模型返回错误消息，不执行。

### 工具执行错误

`tool.call()` 期间的错误被查询引擎捕获并：

1. 通过 `mapToolResultToToolResultBlockParam` 序列化
2. 通过 `renderToolUseErrorMessage()`（特定于工具）或 `<FallbackToolUseErrorMessage />` 渲染
3. 作为带错误内容的 `tool_result` 块返回给模型

### 权限拒绝

- **自动模式拒绝**：通过 `recordAutoModeDenial()` 跟踪，显示通知，计数器递增
- **基于规则的拒绝**：向模型返回解释拒绝的消息
- **用户拒绝**：通过 `renderToolUseRejectedMessage()` 自定义拒绝 UI

### 结果大小限制

来自 `toolLimits.ts`：

| 常量 | 值 | 用途 |
|----------|-------|---------|
| `DEFAULT_MAX_RESULT_SIZE_CHARS` | 50,000 | 每个工具结果字符限制 |
| `MAX_TOOL_RESULT_TOKENS` | 100,000 | 基于令牌的限制 |
| `MAX_TOOL_RESULT_BYTES` | 400,000 | 字节限制（100K x 4） |
| `MAX_TOOL_RESULTS_PER_MESSAGE_CHARS` | 200,000 | 每条消息聚合预算 |
| `TOOL_SUMMARY_MAX_LENGTH` | 50 | 紧凑视图摘要长度 |

当工具结果超过 `maxResultSizeChars` 时：
1. 内容通过 `toolResultStorage.ts` 持久化到磁盘
2. 模型收到带文件路径的预览
3. 单个工具可以设置 `maxResultSizeChars: Infinity` 以选择退出（例如 FileReadTool）

## 关键算法

### 按名称/别名匹配工具

```typescript
function toolMatchesName(tool, name): boolean {
  return tool.name === name || (tool.aliases?.includes(name) ?? false)
}

function findToolByName(tools, name): Tool | undefined {
  return tools.find(t => toolMatchesName(t, name))
}
```

工具可以有别名以在重命名时保持向后兼容。模型可以通过其任何名称引用工具。

### 权限规则匹配

工具可以实现 `preparePermissionMatcher()` 用于细粒度权限匹配：

```typescript
preparePermissionMatcher?(input): Promise<(pattern: string) => boolean>
```

这允许像 BashTool 这样的工具将权限规则匹配到子命令（例如 `Bash(git *)` 匹配 `git commit`、`git push`）。

### 拒绝规则过滤

```typescript
function filterToolsByDenyRules<T>(tools, permissionContext): T[] {
  return tools.filter(tool => !getDenyRuleForTool(permissionContext, tool))
}
```

使用与运行时权限检查相同的匹配器，因此像 `mcp__server` 这样的 MCP 服务器前缀规则在模型看到之前剥离该服务器的所有工具。

### 带缓存稳定性的工具池组装

```typescript
function assembleToolPool(permissionContext, mcpTools): Tools {
  const builtInTools = getTools(permissionContext)
  const allowedMcpTools = filterToolsByDenyRules(mcpTools, permissionContext)
  
  const byName = (a, b) => a.name.localeCompare(b.name)
  return uniqBy(
    [...builtInTools].sort(byName).concat(allowedMcpTools.sort(byName)),
    'name',
  )
}
```

工具按名称排序以确保提示缓存稳定性。服务器在最后一个内置工具之后放置缓存断点；交错 MCP 工具会使下游缓存键失效。`uniqBy` 保留插入顺序，因此内置工具在名称冲突时获胜。

### 延迟工具加载

带 `shouldDefer: true` 的工具从初始系统提示中排除。模型必须首先调用 `ToolSearch` 来发现它们。这在有许多可用工具时减少提示令牌使用。带 `alwaysLoad: true` 的工具是例外 — 它们始终出现在初始提示中。

## 依赖

### 内部

| 模块 | 用途 |
|--------|---------|
| `state/AppState.ts` | 应用程序状态管理 |
| `types/message.ts` | 消息类型（UserMessage、AssistantMessage 等） |
| `types/permissions.ts` | 权限类型和规则 |
| `types/hooks.ts` | Hook 类型（PromptRequest、PromptResponse） |
| `utils/permissions/permissions.ts` | 权限评估逻辑 |
| `utils/toolResultStorage.ts` | 大结果持久化到磁盘 |
| `utils/lazySchema.ts` | 延迟 Zod schema 评估 |
| `utils/envUtils.ts` | 环境变量工具 |
| `services/analytics/` | 事件日志和 GrowthBook 功能标志 |
| `tasks/` | 后台代理的任务管理 |
| `components/` | 用于工具渲染的 React UI 组件 |

### 外部

| 包 | 用途 |
|---------|---------|
| `zod/v4` | 输入/输出 schema 验证 |
| `@anthropic-ai/sdk` | API 类型（ToolUseBlockParam、ToolResultBlockParam） |
| `@modelcontextprotocol/sdk` | MCP 协议类型 |
| `react` | UI 渲染 |
| `lodash-es/uniqBy` | 工具去重 |
| `bun:bundle` | 功能标志 API |

## 数据流

```
用户输入
  |
  ▼
查询引擎 / REPL
  |
  +- assembleToolPool(permissionContext, mcpTools)
  |   +- getTools(permissionContext)
  |   |   +- getAllBaseTools() -- 所有可能的工具
  |   |   +- filterToolsByDenyRules() -- 移除拒绝的工具
  |   |   +- isEnabled() -- 移除禁用的工具
  |   |   +- REPL 模式过滤
  |   +- MCP 工具 -- 按拒绝规则过滤
  |
  +- 系统提示构建
  |   +- 每个工具的 tool.prompt()
  |   +- 运行时上下文的 tool.description()
  |   +- 输入 schemas -> API 的 JSON
  |
  +- API 请求（带工具定义）
  |   |
  |   ▼
  |   模型响应（tool_use 块）
  |   |
  |   ▼
  +- 工具执行管道
  |   +- findToolByName() -> 定位工具
  |   +- validateInput() -> schema + 自定义验证
  |   +- canUseTool() -> 权限决策
  |   |   +- checkPermissions() -> 工具特定逻辑
  |   |   +- hasPermissionsToUseTool() -> 规则 + 模式 + 钩子
  |   |   +- 处理程序（协调器/swarm/交互式）
  |   +- tool.call() -> 执行
  |   |   +- onProgress() -> 流式更新
  |   |   +- ToolResult<T> -> 结果
  |   +- mapToolResultToToolResultBlockParam() -> 序列化
  |   +- 结果大小检查 -> 如果过大则持久化
  |   +- renderToolResultMessage() -> UI
  |
  +- 新消息注入对话
      |
      ▼
    下一个 API 请求（带工具结果）
```

## 集成点

### 查询引擎

查询引擎（`query.ts`）是工具的主要消费者：
- 通过 `ToolUseContext.options.tools` 接收工具
- 序列化工具定义用于 API 请求
- 将 tool_use 块路由到正确的工具
- 处理工具结果并反馈给模型

### AppState

工具通过 `ToolUseContext` 与 AppState 交互：
- `getAppState()` / `setAppState()` — 读取/写入应用程序状态
- `toolPermissionContext` — 当前权限模式和规则
- `mcp.tools` — MCP 提供的工具
- `teamContext` — 用于多代理工具的团队/swarm 状态

### 权限系统

- `useCanUseTool.tsx` — 创建 `canUseTool` 函数的 React hook
- `hasPermissionsToUseTool()` — 核心权限评估（规则 -> 模式 -> 钩子 -> 分类器）
- 三个用于不同执行上下文的处理程序：
  - `handleCoordinatorPermission()` — 协调器模式
  - `handleSwarmWorkerPermission()` — Swarm workers
  - `handleInteractivePermission()` — REPL/交互式

### 钩子系统

工具与钩子系统集成：
- `PreToolUse` 钩子 — 在工具执行前运行，可以拒绝/修改
- `PostToolUse` 钩子 — 在工具执行后运行
- 钩子 `if` 条件通过 `preparePermissionMatcher()` 匹配工具名称和输入

### MCP 集成

- MCP 工具从连接的服务器动态加载
- `ListMcpResourcesTool` 和 `ReadMcpResourceTool` 始终可用
- MCP 工具携带 `mcpInfo: { serverName, toolName }` 元数据
- MCP 工具上的 `_meta['anthropic/alwaysLoad']` 防止延迟

### UI 渲染

每个工具控制自己的渲染：
- `renderToolUseMessage()` — 工具被调用时用户看到的内容
- `renderToolUseProgressMessage()` — 执行期间的进度指示器
- `renderToolResultMessage()` — 在 transcript 中的结果显示
- `renderToolUseErrorMessage()` — 自定义错误 UI
- `renderToolUseRejectedMessage()` — 自定义拒绝 UI
- `renderGroupedToolUse()` — 并行实例的分组显示

### Agent 系统

- `AgentTool` 生成带过滤工具可用性的子代理
- `ASYNC_AGENT_ALLOWED_TOOLS` 限制异步代理可以使用的工具
- `COORDINATOR_MODE_ALLOWED_TOOLS` 限制协调器工具
- 进程内队友通过 `IN_PROCESS_TEAMMATE_ALLOWED_TOOLS` 获得额外工具

## 相关模块

- [Main Entrypoint](./main-entrypoint.md)
- [Query Engine](./query-engine.md)
- [Command System](./command-system.md)
- [Permission System](./permission-system.md)
- [Agent System](./agent-system.md)
- [MCP Integration](./mcp-integration.md)
