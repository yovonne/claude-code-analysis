# Tool System

## Purpose

The Tool system is the core abstraction layer that defines how Claude Code interacts with the external world. Every capability — file operations, shell execution, agent spawning, web searches, MCP integrations — is modeled as a `Tool`. Tools are uniformly typed, validated via Zod schemas, gated by feature flags, filtered by permission rules, and rendered as React components in the terminal UI. This design enables:

- **Uniform interface**: All tools share the same shape regardless of what they do
- **Composable permissions**: Permission rules apply consistently across all tools
- **Feature-flag gating**: Tools can be conditionally loaded without code changes
- **Progressive disclosure**: Deferred tool loading via ToolSearch reduces prompt token usage
- **Rich UI rendering**: Each tool controls its own use/result/progress/error message rendering

## Location

- `restored-src/src/Tool.ts` — Core `Tool` interface, `buildTool` factory, and shared types (~793 lines)
- `restored-src/src/tools.ts` — Tool registry, `getTools()`, `assembleToolPool()`, feature-flag gating (~390 lines)
- `restored-src/src/constants/tools.ts` — Tool allow/deny lists per agent mode (~113 lines)
- `restored-src/src/constants/toolLimits.ts` — Result size and token limits (~57 lines)
- `restored-src/src/types/permissions.ts` — Permission types, decisions, and rules
- `restored-src/src/hooks/useCanUseTool.tsx` — Permission decision hook
- `restored-src/src/tools/` — Individual tool implementations (60+ tools)
- `restored-src/src/tools/shared/` — Shared tool utilities (spawn, git tracking)
- `restored-src/src/tools/testing/` — Testing-only tools

## Key Exports

### From Tool.ts

| Export | Description |
|--------|-------------|
| `Tool<Input, Output, P>` | The core tool interface — every tool must implement this shape |
| `ToolDef<Input, Output, P>` | Partial tool definition accepted by `buildTool`; defaultable methods are optional |
| `Tools` | `readonly Tool[]` — typed collection, preferred over raw arrays |
| `ToolInputJSONSchema` | Raw JSON Schema format for MCP tools that bypass Zod conversion |
| `ValidationResult` | `{ result: true }` or `{ result: false, message, errorCode }` |
| `ToolResult<T>` | `{ data: T, newMessages?, contextModifier?, mcpMeta? }` |
| `ToolUseContext` | Massive context object passed to every tool call — contains AppState, abort controller, JSX setters, message lists, etc. |
| `ToolPermissionContext` | Deep-immutable permission state: mode, rules, working directories |
| `ToolCallProgress<P>` | Callback type for streaming progress updates |
| `ToolProgressData` | Union of all tool-specific progress types |
| `buildTool(def)` | Factory that merges `def` with `TOOL_DEFAULTS` to produce a complete `Tool` |
| `toolMatchesName(tool, name)` | Checks if tool matches by primary name or alias |
| `findToolByName(tools, name)` | Finds a tool by name or alias from a list |
| `getEmptyToolPermissionContext()` | Returns a blank permission context with all defaults |

**Re-exported progress types**: `AgentToolProgress`, `BashProgress`, `MCPProgress`, `REPLToolProgress`, `SkillToolProgress`, `TaskOutputProgress`, `WebSearchProgress`

### From tools.ts

| Export | Description |
|--------|-------------|
| `getTools(permissionContext)` | Returns filtered built-in tools respecting deny rules, REPL mode, and `isEnabled()` |
| `getAllBaseTools()` | Returns the exhaustive list of all possible tools (before filtering) |
| `assembleToolPool(permissionContext, mcpTools)` | Combines built-in + MCP tools with deduplication and sorting |
| `getMergedTools(permissionContext, mcpTools)` | Simple concatenation of built-in + MCP tools |
| `filterToolsByDenyRules(tools, permissionContext)` | Filters out tools blanket-denied by permission rules |
| `TOOL_PRESETS` | Predefined tool presets (currently only `'default'`) |
| `getToolsForDefaultPreset()` | Returns tool names for the default preset |

**Re-exported constants**: `ALL_AGENT_DISALLOWED_TOOLS`, `CUSTOM_AGENT_DISALLOWED_TOOLS`, `ASYNC_AGENT_ALLOWED_TOOLS`, `COORDINATOR_MODE_ALLOWED_TOOLS`, `REPL_ONLY_TOOLS`

## Tool Interface

The `Tool` interface is the contract every tool must satisfy. It is parameterized by `Input` (Zod schema), `Output` (result type), and `P` (progress data type):

```typescript
export type Tool<
  Input extends AnyObject = AnyObject,
  Output = unknown,
  P extends ToolProgressData = ToolProgressData,
> = {
  // Identity
  readonly name: string
  aliases?: string[]
  searchHint?: string          // 3-10 word phrase for ToolSearch keyword matching

  // Required methods
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

  // Behavioral predicates
  isEnabled(): boolean
  isConcurrencySafe(input): boolean
  isReadOnly(input): boolean
  isDestructive?(input): boolean
  interruptBehavior?(): 'cancel' | 'block'
  isOpenWorld?(input): boolean
  requiresUserInteraction?(): boolean

  // Validation
  validateInput?(input, context): Promise<ValidationResult>
  inputsEquivalent?(a, b): boolean

  // Permission matching
  preparePermissionMatcher?(input): Promise<(pattern) => boolean>

  // UI rendering (all optional)
  renderToolResultMessage?(content, progressMessages, options): React.ReactNode
  renderToolUseProgressMessage?(progressMessages, options): React.ReactNode
  renderToolUseQueuedMessage?(): React.ReactNode
  renderToolUseRejectedMessage?(input, options): React.ReactNode
  renderToolUseErrorMessage?(result, options): React.ReactNode
  renderGroupedToolUse?(toolUses, options): React.ReactNode | null
  renderToolUseTag?(input): React.ReactNode
  userFacingNameBackgroundColor?(input): keyof Theme | undefined

  // Summaries and descriptions
  getToolUseSummary?(input): string | null
  getActivityDescription?(input): string | null
  isSearchOrReadCommand?(input): { isSearch, isRead, isList }
  toAutoClassifierInput(input): unknown

  // Searchability
  extractSearchText?(out: Output): string
  isResultTruncated?(output): boolean

  // Loading behavior
  readonly shouldDefer?: boolean      // Deferred loading via ToolSearch
  readonly alwaysLoad?: boolean       // Never defer, always include
  readonly strict?: boolean           // Strict mode for tengu_tool_pear

  // MCP metadata
  isMcp?: boolean
  isLsp?: boolean
  mcpInfo?: { serverName: string; toolName: string }

  // Input mutation
  backfillObservableInput?(input): void

  // Size limits
  maxResultSizeChars: number

  // Transparency
  isTransparentWrapper?(): boolean
}
```

## Tool Defaults

The `buildTool` factory fills in safe defaults for commonly-omitted methods:

```typescript
const TOOL_DEFAULTS = {
  isEnabled: () => true,
  isConcurrencySafe: (_input?) => false,    // Fail-closed: assume not safe
  isReadOnly: (_input?) => false,           // Fail-closed: assume writes
  isDestructive: (_input?) => false,
  checkPermissions: (input) =>              // Defer to general permission system
    Promise.resolve({ behavior: 'allow', updatedInput: input }),
  toAutoClassifierInput: (_input?) => '',   // Skip classifier by default
  userFacingName: (_input?) => '',
}
```

**Key design**: Defaults are fail-closed where it matters — tools are assumed to be write operations, not concurrency-safe, and skipped by the auto-mode classifier unless explicitly overridden.

## Tool Lifecycle

```
1. MODULE LOAD
   - Tool module imported (static or dynamic)
   - buildTool() called with ToolDef -> complete Tool object
   - Zod input schema defined (usually via lazySchema)

2. REGISTRATION
   - getAllBaseTools() assembles all tool instances
   - Feature flags gate conditional tool inclusion
   - getTools() filters by deny rules + isEnabled()

3. PROMPT CONSTRUCTION
   - tool.prompt() generates system prompt text for the model
   - tool.description() generates runtime description
   - Input schemas serialized to JSON for API tool definitions
   - Deferred tools (shouldDefer=true) excluded from initial prompt

4. MODEL INVOCATION
   - Model returns tool_use block with name + input

5. TOOL RESOLUTION
   - findToolByName(tools, name) locates tool by name or alias
   - Input validated against Zod inputSchema

6. VALIDATION
   - tool.validateInput(input, context) called if defined
   - Returns ValidationResult (pass/fail with error code)

7. PERMISSION CHECK
   - canUseTool(tool, input, context, ...) invoked
   - tool.checkPermissions(input, context) returns PermissionResult
   - Permission system evaluates: rules -> mode -> hooks -> classifier
   - Decision: allow / deny / ask (shows UI prompt)

8. EXECUTION
   - tool.call(args, context, canUseTool, parentMessage, onProgress)
   - Tool performs its operation (file I/O, shell exec, agent spawn, etc.)
   - Progress events emitted via onProgress callback
   - Result wrapped in ToolResult<T>

9. RESULT HANDLING
   - tool.mapToolResultToToolResultBlockParam() serializes for API
   - Result size checked against maxResultSizeChars
   - Oversized results persisted to disk with preview returned
   - tool.renderToolResultMessage() renders in terminal UI
   - New messages (if any) injected into conversation
```

## Tool Registry

### Tool Assembly Pipeline

```
getAllBaseTools()
  +-- Always-present tools (BashTool, FileReadTool, FileEditTool, etc.)
  +-- Feature-flag gated tools (SleepTool, WorkflowTool, WebBrowserTool, etc.)
  +-- Environment-flag gated tools (REPLTool, ConfigTool, TungstenTool)
  +-- Runtime-check gated tools (PowerShellTool, LSPTool, Worktree tools)
  +--> Tools[]

getTools(permissionContext)
  +-- CLAUDE_CODE_SIMPLE mode -> [BashTool, FileReadTool, FileEditTool]
  +-- getAllBaseTools() filtered:
  |   +-- Remove special tools (ListMcpResources, ReadMcpResource, SyntheticOutput)
  |   +-- filterToolsByDenyRules() -> remove blanket-denied tools
  |   +-- REPL mode -> hide REPL_ONLY_TOOLS
  |   +-- isEnabled() -> remove disabled tools
  +--> Tools[]

assembleToolPool(permissionContext, mcpTools)
  +-- getTools(permissionContext) -> built-in tools
  +-- filterToolsByDenyRules(mcpTools) -> allowed MCP tools
  +-- Sort each partition by name (prompt-cache stability)
  +-- uniqBy(..., 'name') -> deduplicate (built-ins win)
  +--> Tools[]
```

### Feature Flag Gating

Tools are conditionally loaded using two mechanisms:

**1. `feature()` from `bun:bundle`** — Compile-time feature flags:

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

**2. `process.env` checks** — Runtime environment flags:

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

**3. Runtime utility checks**:

```typescript
...(isPowerShellToolEnabled() ? [getPowerShellTool()] : []),
...(isWorktreeModeEnabled() ? [EnterWorktreeTool, ExitWorktreeTool] : []),
...(isEnvTruthy(process.env.ENABLE_LSP_TOOL) ? [LSPTool] : []),
...(isAgentSwarmsEnabled() ? [getTeamCreateTool(), getTeamDeleteTool()] : []),
```

### Dead Code Elimination

Conditional `require()` calls (not top-level `import`) enable tree-shaking. When a feature flag is false at build time, the tool module is never included in the bundle. ESLint rules (`custom-rules/no-process-env-top-level`) enforce this pattern.

## Permission System

### Permission Flow

```
model calls tool
  |
  +- 1. validateInput() -> fail? return error to model
  |
  +- 2. canUseTool(tool, input, context, ...)
  |     |
  |     +- forceDecision provided? -> use it
  |     |
  |     +- hasPermissionsToUseTool() -> evaluates:
  |     |   +- alwaysAllowRules -> allow
  |     |   +- alwaysDenyRules -> deny
  |     |   +- tool.checkPermissions() -> ask/allow/deny/passthrough
  |     |   +- Permission mode (bypassPermissions -> allow)
  |     |   +- Hooks (PreToolUse/PostToolUse)
  |     |   +- Classifiers (auto-mode, bash classifier)
  |     |
  |     +- Decision handlers:
  |         +- handleCoordinatorPermission() -> coordinator mode
  |         +- handleSwarmWorkerPermission() -> swarm workers
  |         +- handleInteractivePermission() -> REPL/interactive
  |
  +- 3. tool.call() -> execute
  |
  +- 4. Return ToolResult -> render + inject messages
```

### PermissionResult Types

```typescript
type PermissionResult =
  | { behavior: 'allow', updatedInput? }
  | { behavior: 'deny', message, decisionReason? }
  | { behavior: 'ask', message, decisionReason?, suggestions?, pendingClassifierCheck? }
  | { behavior: 'passthrough', message, decisionReason?, suggestions?, blockedPath? }
```

### Permission Modes

| Mode | Behavior |
|------|----------|
| `default` | Ask for permission on each tool use |
| `bypassPermissions` | Auto-approve all tools (skip permissions) |
| `acceptEdits` | Auto-approve read-only tools, ask for writes |
| `auto` | Classifier auto-approves safe tools, asks for risky ones |
| `plan` | Plan mode — only planning tools allowed |
| `dontAsk` | Never ask, auto-deny risky operations |
| `bubble` | Bubble permission decisions to parent context |

### Tool Allow/Deny Lists

Defined in `constants/tools.ts`:

- **`ALL_AGENT_DISALLOWED_TOOLS`**: Tools blocked for all subagents (TaskOutput, ExitPlanMode, Agent, AskUserQuestion, TaskStop, Workflow)
- **`ASYNC_AGENT_ALLOWED_TOOLS`**: Whitelist for async agents (Read, WebSearch, TodoWrite, Grep, WebFetch, Glob, Shell, Edit, Write, NotebookEdit, Skill, ToolSearch, Worktree)
- **`COORDINATOR_MODE_ALLOWED_TOOLS`**: Tools for coordinator (Agent, TaskStop, SendMessage, SyntheticOutput)
- **`IN_PROCESS_TEAMMATE_ALLOWED_TOOLS`**: Tools for in-process teammates (Task CRUD, SendMessage, Cron)

## Tool Categories

| Category | Tools | Description |
|----------|-------|-------------|
| **Shell Execution** | `BashTool`, `PowerShellTool` | Run shell commands with sandbox, timeout, and output streaming |
| **File Read** | `FileReadTool`, `GlobTool`, `GrepTool` | Read files, glob patterns, regex search with binary/PDF/image support |
| **File Write** | `FileWriteTool`, `FileEditTool`, `NotebookEditTool` | Create/overwrite files, apply edits, modify Jupyter notebooks |
| **Agent/Task** | `AgentTool`, `TaskCreateTool`, `TaskGetTool`, `TaskUpdateTool`, `TaskListTool`, `TaskStopTool`, `TaskOutputTool` | Spawn subagents, manage tasks, retrieve output |
| **Team/Swarm** | `TeamCreateTool`, `TeamDeleteTool`, `SendMessageTool`, `ListPeersTool` | Multi-agent team management and communication |
| **Web** | `WebFetchTool`, `WebSearchTool`, `WebBrowserTool` | Fetch URLs, search the web, browser automation |
| **Planning** | `EnterPlanModeTool`, `ExitPlanModeV2Tool` | Switch between plan and act modes |
| **MCP** | `ListMcpResourcesTool`, `ReadMcpResourceTool` | Model Context Protocol resource access |
| **Skills** | `SkillTool` | Load and execute skill definitions |
| **Organization** | `TodoWriteTool`, `BriefTool` | Task tracking and session briefs |
| **User Interaction** | `AskUserQuestionTool`, `ConfigTool` | Ask clarifying questions, manage configuration |
| **Search** | `ToolSearchTool` | Deferred tool discovery via keyword search |
| **LSP** | `LSPTool` | Language Server Protocol integration |
| **Worktree** | `EnterWorktreeTool`, `ExitWorktreeTool` | Git worktree management |
| **Advanced** | `WorkflowTool`, `SleepTool`, `SnipTool`, `CtxInspectTool`, `TerminalCaptureTool` | Feature-flag gated advanced capabilities |
| **Scheduling** | `CronCreateTool`, `CronDeleteTool`, `CronListTool`, `RemoteTriggerTool` | Cron jobs and remote triggers |
| **Notifications** | `PushNotificationTool`, `SendUserFileTool`, `SubscribePRTool` | Push notifications, file sharing, GitHub webhooks |
| **Monitoring** | `MonitorTool` | Proactive monitoring |
| **Testing** | `TestingPermissionTool` | Test-only tool for permission dialog testing |

## Tool Input/Output Schemas

### Input Schema Pattern

Tools define input schemas using Zod v4, typically with `lazySchema` for tree-shaking:

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

### JSON Schema for MCP

MCP tools can provide raw JSON Schema instead of Zod:

```typescript
readonly inputJSONSchema?: ToolInputJSONSchema
// { type: 'object', properties: { ... } }
```

### Output Schema

Optional Zod schema for validating tool outputs:

```typescript
outputSchema?: z.ZodType<unknown>
```

### Result Serialization

Every tool implements `mapToolResultToToolResultBlockParam`:

```typescript
mapToolResultToToolResultBlockParam(content: Output, toolUseID: string): ToolResultBlockParam {
  return {
    type: 'tool_result',
    content: String(content),
    tool_use_id: toolUseID,
  }
}
```

## Error Handling

### Validation Errors

```typescript
type ValidationResult =
  | { result: true }
  | { result: false; message: string; errorCode: number }
```

Tools implement `validateInput()` to catch errors before permission checks. Failed validation returns an error message to the model without executing.

### Tool Execution Errors

Errors during `tool.call()` are caught by the query engine and:

1. Serialized via `mapToolResultToToolResultBlockParam`
2. Rendered via `renderToolUseErrorMessage()` (tool-specific) or `<FallbackToolUseErrorMessage />`
3. Returned to the model as a `tool_result` block with error content

### Permission Denials

- **Auto-mode denials**: Tracked via `recordAutoModeDenial()`, notification shown, counter increments
- **Rule-based denials**: Message returned to model explaining the denial
- **User denials**: Custom rejection UI via `renderToolUseRejectedMessage()`

### Result Size Limits

From `toolLimits.ts`:

| Constant | Value | Purpose |
|----------|-------|---------|
| `DEFAULT_MAX_RESULT_SIZE_CHARS` | 50,000 | Per-tool result character limit |
| `MAX_TOOL_RESULT_TOKENS` | 100,000 | Token-based limit |
| `MAX_TOOL_RESULT_BYTES` | 400,000 | Byte-based limit (100K x 4) |
| `MAX_TOOL_RESULTS_PER_MESSAGE_CHARS` | 200,000 | Per-message aggregate budget |
| `TOOL_SUMMARY_MAX_LENGTH` | 50 | Compact view summary length |

When a tool result exceeds `maxResultSizeChars`:
1. Content is persisted to disk via `toolResultStorage.ts`
2. Model receives a preview with the file path
3. Individual tools can set `maxResultSizeChars: Infinity` to opt out (e.g., FileReadTool)

## Key Algorithms

### Tool Matching by Name/Alias

```typescript
function toolMatchesName(tool, name): boolean {
  return tool.name === name || (tool.aliases?.includes(name) ?? false)
}

function findToolByName(tools, name): Tool | undefined {
  return tools.find(t => toolMatchesName(t, name))
}
```

Tools can have aliases for backwards compatibility when renamed. The model can reference a tool by any of its names.

### Permission Rule Matching

Tools can implement `preparePermissionMatcher()` for fine-grained permission matching:

```typescript
preparePermissionMatcher?(input): Promise<(pattern: string) => boolean>
```

This allows tools like BashTool to match permission rules against subcommands (e.g., `Bash(git *)` matches `git commit`, `git push`).

### Deny Rule Filtering

```typescript
function filterToolsByDenyRules<T>(tools, permissionContext): T[] {
  return tools.filter(tool => !getDenyRuleForTool(permissionContext, tool))
}
```

Uses the same matcher as runtime permission checks, so MCP server-prefix rules like `mcp__server` strip all tools from that server before the model sees them.

### Tool Pool Assembly with Cache Stability

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

Tools are sorted by name for prompt-cache stability. The server places a cache breakpoint after the last built-in tool; interleaving MCP tools would invalidate downstream cache keys. `uniqBy` preserves insertion order so built-in tools win on name conflicts.

### Deferred Tool Loading

Tools with `shouldDefer: true` are excluded from the initial system prompt. The model must first call `ToolSearch` to discover them. This reduces prompt token usage when many tools are available. Tools with `alwaysLoad: true` are exceptions — they always appear in the initial prompt.

## Dependencies

### Internal

| Module | Purpose |
|--------|---------|
| `state/AppState.ts` | Application state management |
| `types/message.ts` | Message types (UserMessage, AssistantMessage, etc.) |
| `types/permissions.ts` | Permission types and rules |
| `types/hooks.ts` | Hook types (PromptRequest, PromptResponse) |
| `utils/permissions/permissions.ts` | Permission evaluation logic |
| `utils/toolResultStorage.ts` | Large result persistence to disk |
| `utils/lazySchema.ts` | Lazy Zod schema evaluation |
| `utils/envUtils.ts` | Environment variable utilities |
| `services/analytics/` | Event logging and GrowthBook feature flags |
| `tasks/` | Task management for background agents |
| `components/` | React UI components for tool rendering |

### External

| Package | Purpose |
|---------|---------|
| `zod/v4` | Input/output schema validation |
| `@anthropic-ai/sdk` | API types (ToolUseBlockParam, ToolResultBlockParam) |
| `@modelcontextprotocol/sdk` | MCP protocol types |
| `react` | UI rendering |
| `lodash-es/uniqBy` | Tool deduplication |
| `bun:bundle` | Feature flag API |

## Data Flow

```
User Input
  |
  v
QueryEngine / REPL
  |
  +- assembleToolPool(permissionContext, mcpTools)
  |   +- getTools(permissionContext)
  |   |   +- getAllBaseTools() -- all possible tools
  |   |   +- filterToolsByDenyRules() -- remove denied tools
  |   |   +- isEnabled() -- remove disabled tools
  |   |   +- REPL mode filtering
  |   +- MCP tools -- filtered by deny rules
  |
  +- System Prompt Construction
  |   +- tool.prompt() for each tool
  |   +- tool.description() for runtime context
  |   +- Input schemas -> JSON for API
  |
  +- API Request (with tool definitions)
  |   |
  |   v
  |   Model Response (tool_use blocks)
  |   |
  |   v
  +- Tool Execution Pipeline
  |   +- findToolByName() -> locate tool
  |   +- validateInput() -> schema + custom validation
  |   +- canUseTool() -> permission decision
  |   |   +- checkPermissions() -> tool-specific logic
  |   |   +- hasPermissionsToUseTool() -> rules + mode + hooks
  |   |   +- Handler (coordinator/swarm/interactive)
  |   +- tool.call() -> execute
  |   |   +- onProgress() -> streaming updates
  |   |   +- ToolResult<T> -> result
  |   +- mapToolResultToToolResultBlockParam() -> serialize
  |   +- Result size check -> persist if oversized
  |   +- renderToolResultMessage() -> UI
  |
  +- New messages injected into conversation
      |
      v
    Next API Request (with tool results)
```

## Integration Points

### Query Engine

The query engine (`query.ts`) is the primary consumer of tools:
- Receives tools via `ToolUseContext.options.tools`
- Serializes tool definitions for API requests
- Routes tool_use blocks to the correct tool
- Handles tool results and feeds them back to the model

### AppState

Tools interact with AppState through `ToolUseContext`:
- `getAppState()` / `setAppState()` — read/write application state
- `toolPermissionContext` — current permission mode and rules
- `mcp.tools` — MCP-provided tools
- `teamContext` — team/swarm state for multi-agent tools

### Permission System

- `useCanUseTool.tsx` — React hook that creates the `canUseTool` function
- `hasPermissionsToUseTool()` — Core permission evaluation (rules -> mode -> hooks -> classifier)
- Three handlers for different execution contexts:
  - `handleCoordinatorPermission()` — Coordinator mode
  - `handleSwarmWorkerPermission()` — Swarm workers
  - `handleInteractivePermission()` — REPL/interactive

### Hooks System

Tools integrate with the hooks system:
- `PreToolUse` hooks — run before tool execution, can deny/modify
- `PostToolUse` hooks — run after tool execution
- Hook `if` conditions match tool names and inputs via `preparePermissionMatcher()`

### MCP Integration

- MCP tools are dynamically loaded from connected servers
- `ListMcpResourcesTool` and `ReadMcpResourceTool` are always available
- MCP tools carry `mcpInfo: { serverName, toolName }` metadata
- `_meta['anthropic/alwaysLoad']` on MCP tools prevents deferral

### UI Rendering

Each tool controls its own rendering:
- `renderToolUseMessage()` — What the user sees when the tool is called
- `renderToolUseProgressMessage()` — Progress indicators during execution
- `renderToolResultMessage()` — Result display in the transcript
- `renderToolUseErrorMessage()` — Custom error UI
- `renderToolUseRejectedMessage()` — Custom rejection UI
- `renderGroupedToolUse()` — Grouped display for parallel instances

### Agent System

- `AgentTool` spawns subagents with filtered tool availability
- `ASYNC_AGENT_ALLOWED_TOOLS` restricts what async agents can use
- `COORDINATOR_MODE_ALLOWED_TOOLS` restricts coordinator tools
- In-process teammates get additional tools via `IN_PROCESS_TEAMMATE_ALLOWED_TOOLS`

## Related Modules

- [Main Entrypoint](./main-entrypoint.md)
- [Query Engine](./query-engine.md)
- [Command System](./command-system.md)
- [Permission System](./permission-system.md)
- [Agent System](./agent-system.md)
- [MCP Integration](./mcp-integration.md)
