# Tool Execution Flow

## Purpose

Traces the complete lifecycle of tool execution — from the model's tool_use API response through validation, permission checking, execution, progress reporting, and result handling.

## Location

Primary sources:
- `restored-src/src/Tool.ts` — `Tool` interface, `buildTool`, `ToolUseContext`, `ToolResult`
- `restored-src/src/QueryEngine.ts` — tool execution orchestration
- `restored-src/src/query.ts` — query loop that dispatches tool calls
- `restored-src/src/services/tools/toolExecution.ts` — execution engine
- `restored-src/src/services/tools/toolOrchestration.ts` — multi-tool coordination

---

## Tool Interface

Every tool in the system implements the `Tool<Input, Output, Progress>` interface:

```typescript
type Tool<Input, Output, Progress> = {
  // Identity
  name: string
  aliases?: string[]
  searchHint?: string           // keyword search hint for ToolSearch

  // Schema
  inputSchema: z.ZodType<Input>
  inputJSONSchema?: ToolInputJSONSchema  // for MCP tools
  outputSchema?: z.ZodType<Output>

  // Core execution
  call(args: Input, context: ToolUseContext, canUseTool, parentMessage, onProgress?)
    : Promise<ToolResult<Output>>

  // Description (dynamic, context-aware)
  description(input: Input, options): Promise<string>
  prompt(options): Promise<string>

  // Permission & validation
  validateInput?(input: Input, context): Promise<ValidationResult>
  checkPermissions(input: Input, context): Promise<PermissionResult>
  preparePermissionMatcher?(input: Input): Promise<(pattern: string) => boolean>

  // Safety & behavior
  isEnabled(): boolean
  isConcurrencySafe(input: Input): boolean
  isReadOnly(input: Input): boolean
  isDestructive?(input: Input): boolean
  interruptBehavior?(): 'cancel' | 'block'
  requiresUserInteraction?(): boolean

  // UI rendering
  renderToolUseMessage(input, options): React.ReactNode
  renderToolUseProgressMessage(progressMessages, options): React.ReactNode
  renderToolResultMessage(content, progressMessages, options): React.ReactNode
  renderToolUseRejectedMessage?(input, options): React.ReactNode
  renderToolUseErrorMessage?(result, options): React.ReactNode
  renderGroupedToolUse?(toolUses, options): React.ReactNode | null

  // Display helpers
  userFacingName(input): string
  userFacingNameBackgroundColor?(input): string
  getActivityDescription?(input): string | null
  getToolUseSummary?(input): string | null
  isResultTruncated?(output): boolean
  renderToolUseTag?(input): React.ReactNode
  renderToolUseQueuedMessage?(): React.ReactNode

  // Data conversion
  mapToolResultToToolResultBlockParam(content: Output, toolUseID: string)
    : ToolResultBlockParam
  toAutoClassifierInput(input: Input): unknown
  extractSearchText?(out: Output): string

  // Advanced
  shouldDefer?: boolean         // lazy-load via ToolSearch
  alwaysLoad?: boolean          // never defer
  strict?: boolean              // strict mode enforcement
  isTransparentWrapper?(): boolean
  isSearchOrReadCommand?(input): { isSearch, isRead, isList? }
  isOpenWorld?(input): boolean
  backfillObservableInput?(input): void
  getPath?(input: Input): string
  isMcp?: boolean
  isLsp?: boolean
  mcpInfo?: { serverName, toolName }

  // Size limits
  maxResultSizeChars: number
}
```

---

## Tool Execution Lifecycle

### Phase 1: Tool Discovery and Loading

```
QueryEngine.start()
  │
  ├─► getTools()                 — build initial tool registry
  │    │
  │    ├─► Built-in tools (Bash, Read, Edit, Write, etc.)
  │    ├─► MCP tools (from connected servers)
  │    ├─► Agent tools (if enabled)
  │    └─► Plugin tools (if installed)
  │
  ├─► Deferred tools (shouldDefer: true)
  │    └─► Not included in initial API call
  │    └─► Discovered via ToolSearch tool
  │
  └─► Tool filtering
       ├─► isEnabled() check per tool
       ├─► MCP permission filtering
       └─► Sandbox command exclusions
```

### Phase 2: Tool Call Dispatch

```
API returns assistant message with tool_use content blocks
  │
  ▼
query.ts: handleToolUse()
  │
  ├─► For each tool_use block (potentially parallel):
  │    │
  │    ├─► 1. Find tool by name (or alias)
  │    │    └─► toolMatchesName(tool, name)
  │    │
  │    ├─► 2. Validate input schema
  │    │    └─► inputSchema.parse(input)
  │    │
  │    ├─► 3. backfillObservableInput(input)
  │    │    └─► Add legacy/derived fields for hooks/transcript
  │    │
  │    ├─► 4. validateInput(input, context)
  │    │    ├─► Tool-specific validation
  │    │    └─► If fails → return error tool_result
  │    │
  │    └─► 5. canUseTool(tool, input, context, toolUseID)
  │         └─► See Permission Flow document
  │
  └─► 6. Execute or deny based on permission result
```

### Phase 3: Tool Execution

```
Permission === 'allow'
  │
  ▼
tool.call(args, context, canUseTool, parentMessage, onProgress)
  │
  ├─► Tool implementation executes:
  │    │
  │    ├─► BashTool: spawn subprocess, stream output
  │    ├─► FileReadTool: read file, apply limits
  │    ├─► FileEditTool: apply diff/replace
  │    ├─► FileWriteTool: write file content
  │    ├─► MCPTool: call MCP server method
  │    ├─► AgentTool: spawn subagent
  │    └─► ... (each tool has unique implementation)
  │
  ├─► Progress reporting (optional):
  │    │
  │    ├─► onProgress({ toolUseID, data: ProgressData })
  │    ├─► ProgressMessage yielded to query loop
  │    ├─► UI updates with progress display
  │    └─► Progress types:
  │         ├─► BashProgress (stdout/stderr chunks)
  │         ├─► WebSearchProgress (search stages)
  │         ├─► MCPProgress (MCP call stages)
  │         ├─► AgentToolProgress (subagent status)
  │         ├─► SkillToolProgress (skill execution)
  │         ├─► TaskOutputProgress (task output)
  │         └─► REPLToolProgress (REPL execution)
  │
  ├─► Interrupt handling:
  │    │
  │    ├─► interruptBehavior() === 'cancel' → abort tool
  │    └─► interruptBehavior() === 'block' → wait for completion
  │
  └─► Returns ToolResult<Output>
       │
       ├─► data: Output
       ├─► newMessages?: Message[]  (additional messages to inject)
       ├─► contextModifier?: (context) => ToolUseContext
       └─► mcpMeta?: { _meta, structuredContent }
```

### Phase 4: Result Processing

```
ToolResult received
  │
  ▼
mapToolResultToToolResultBlockParam(result, toolUseID)
  │
  ├─► Convert tool output to API format
  ├─► Handle oversized results:
  │    ├─► If result > maxResultSizeChars
  │    └─► Persist to temp file, return preview + path
  │
  ▼
createUserMessage({
  content: tool_result_block,
  toolUseResult: true,
  parentUuid: assistantMessage.uuid,
})
  │
  ▼
Push to mutableMessages
  │
  ▼
Record transcript
  │
  └─► Continue query loop (model sees tool result)
```

---

## Tool Execution Patterns

### Sequential vs Parallel Execution

```
Model can return multiple tool_use blocks in one response:
  │
  ├─► Sequential tools (isConcurrencySafe === false):
  │    └─► Execute one at a time in order
  │    └─► Each result visible to next tool
  │
  └─► Parallel tools (isConcurrencySafe === true):
       └─► Execute simultaneously via Promise.all
       └─► Results collected and returned together
```

### Deferred Tool Loading (ToolSearch)

```
Tool with shouldDefer: true
  │
  ├─► Not included in initial API tool list
  ├─► Model calls ToolSearch to find it
  ├─► ToolSearch returns tool name + description
  ├─► Model then calls the tool by name
  └─► Tool is loaded on-demand for that turn
```

### Subagent Tool Execution (AgentTool)

```
AgentTool.call()
  │
  ├─► Create subagent context (clone or fork)
  ├─► Spawn forked agent or subprocess
  ├─► Subagent runs its own query loop
  ├─► Progress streamed back to parent
  ├─► Result returned as ToolResult
  └─► Subagent messages optionally preserved in transcript
```

---

## ToolResult Structure

```typescript
type ToolResult<T> = {
  data: T                                    // The actual output
  newMessages?: Message[]                    // Additional messages to inject
  contextModifier?: (context) => ToolUseContext  // Modify execution context
  mcpMeta?: {                                // MCP protocol passthrough
    _meta?: Record<string, unknown>
    structuredContent?: Record<string, unknown>
  }
}
```

---

## Tool Defaults (via buildTool)

The `buildTool()` factory fills in safe defaults:

| Method | Default | Rationale |
|--------|---------|-----------|
| `isEnabled()` | `true` | Tool is available |
| `isConcurrencySafe()` | `false` | Assume not safe (fail-closed) |
| `isReadOnly()` | `false` | Assume writes (fail-closed) |
| `isDestructive()` | `false` | Assume not destructive |
| `checkPermissions()` | `{ behavior: 'allow' }` | Defer to general permission system |
| `toAutoClassifierInput()` | `''` | Skip classifier (security tools override) |
| `userFacingName()` | `name` | Use tool name |

---

## Error Handling

```
Tool execution error
  │
  ├─► API error → synthetic assistant message with error text
  ├─► Validation error → tool_result with error message
  ├─► Permission denied → tool_result with denial reason
  ├─► Timeout → tool_result with timeout message
  ├─► Subprocess crash → tool_result with stderr
  └─► Unhandled exception → tool_result with error stack
  │
  └─► All errors become user messages with tool_result content
       └─► Model sees error and can retry or adapt
```

---

## Integration Points

| Component | Role in Tool Execution |
|-----------|----------------------|
| `Tool.call()` | Core tool implementation |
| `canUseTool` | Permission gate |
| `ToolUseContext` | Execution context (state, abort, etc.) |
| `onProgress` | Progress reporting callback |
| `mapToolResultToToolResultBlockParam` | API format conversion |
| `maxResultSizeChars` | Result size management |
| `buildTool()` | Default-filling factory |

## Related Documentation

- [Message Flow](./message-flow.md)
- [Permission Flow](./permission-flow.md)
- [Session Lifecycle](./session-lifecycle.md)
- [Tool System](../01-core-modules/tool-system.md)
- [BashTool](../02-tools/BashTool.md)
- [FileEditTool](../02-tools/FileEditTool.md)
