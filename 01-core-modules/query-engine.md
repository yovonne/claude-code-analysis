# Query Engine

## Purpose

The Query Engine is the core subsystem that manages the complete lifecycle of LLM interactions in Claude Code. It orchestrates message construction, API streaming, tool execution loops, context management (compaction, snipping, collapse), retry logic, cost/token tracking, and result reporting. It serves as the bridge between user input and the Anthropic API, handling multi-turn agentic conversations with tool use.

## Location

- `restored-src/src/QueryEngine.ts` (~1296 lines) — Main query engine class and `ask()` convenience wrapper
- `restored-src/src/query.ts` (~1700+ lines) — Core query loop (`query()`, `queryLoop()`)
- `restored-src/src/query/` — Query sub-modules (token budget, config, deps, stop hooks)
- `restored-src/src/services/api/claude.ts` — Streaming API client (`queryModelWithStreaming`)
- `restored-src/src/services/api/withRetry.ts` — Retry logic for API calls
- `restored-src/src/services/api/errors.ts` — API error classification and handling
- `restored-src/src/services/api/logging.ts` — API telemetry and logging
- `restored-src/src/utils/messages.ts` — Message construction utilities
- `restored-src/src/utils/systemPromptType.ts` — `SystemPrompt` branded type
- `restored-src/src/context.ts` — User and system context builders
- `restored-src/src/cost-tracker.ts` — Cost and token accounting

## Key Exports

### Classes

- **`QueryEngine`** — Owns the query lifecycle and session state for a conversation. One `QueryEngine` per conversation. Each `submitMessage()` call starts a new turn within the same conversation. State (messages, file cache, usage, etc.) persists across turns.

### Functions

- **`ask()`** — Convenience wrapper around `QueryEngine` for one-shot usage. Creates a `QueryEngine`, calls `submitMessage()`, and transfers the read file cache back.
- **`query()`** — Core generator function that runs the query loop. Yields `StreamEvent`, `RequestStartEvent`, `Message`, `TombstoneMessage`, and `ToolUseSummaryMessage`. Returns a `Terminal` result.
- **`queryLoop()`** — Internal implementation of the query loop. Handles compaction, API calls, tool execution, and continuation logic.
- **`handleStopHooks()`** — Executes post-response hooks (Stop, TaskCompleted, TeammateIdle) after the model finishes responding.
- **`buildQueryConfig()`** — Snapshots immutable runtime gates (streaming tool execution, tool use summaries, ant mode, fast mode) once at query entry.

### Types

- **`QueryEngineConfig`** — Configuration object for `QueryEngine` constructor. Includes `cwd`, `tools`, `commands`, `mcpClients`, `agents`, `canUseTool`, `getAppState`, `setAppState`, `initialMessages`, `readFileCache`, `customSystemPrompt`, `appendSystemPrompt`, `userSpecifiedModel`, `fallbackModel`, `thinkingConfig`, `maxTurns`, `maxBudgetUsd`, `taskBudget`, `jsonSchema`, and more.
- **`QueryParams`** — Parameters for the `query()` function. Includes `messages`, `systemPrompt`, `userContext`, `systemContext`, `canUseTool`, `toolUseContext`, `fallbackModel`, `querySource`, `maxOutputTokensOverride`, `maxTurns`, `skipCacheWrite`, `taskBudget`, `deps`.
- **`QueryDeps`** — Dependency injection interface for testing. Contains `callModel`, `microcompact`, `autocompact`, and `uuid`.
- **`QueryConfig`** — Immutable runtime configuration snapshot. Contains `sessionId` and `gates` (streamingToolExecution, emitToolUseSummaries, isAnt, fastModeEnabled).
- **`State`** — Mutable cross-iteration state in the query loop. Contains `messages`, `toolUseContext`, `autoCompactTracking`, `maxOutputTokensRecoveryCount`, `hasAttemptedReactiveCompact`, `maxOutputTokensOverride`, `pendingToolUseSummary`, `stopHookActive`, `turnCount`, `transition`.
- **`TokenBudgetDecision`** — Result of token budget checking. Either `{ action: 'continue', nudgeMessage, ... }` or `{ action: 'stop', completionEvent }`.
- **`BudgetTracker`** — Tracks token budget continuations with `continuationCount`, `lastDeltaTokens`, `lastGlobalTurnTokens`, `startedAt`.
- **`SystemPrompt`** — Branded type (`readonly string[] & { readonly __brand: 'SystemPrompt' }`) for type-safe system prompt arrays.

## Query Construction

Queries are constructed through a multi-stage pipeline:

### 1. System Prompt Assembly

```
QueryEngine.submitMessage()
  → fetchSystemPromptParts()           // Fetch tools, model, MCP, custom prompt parts
  → asSystemPrompt([...parts])         // Wrap as branded SystemPrompt type
  → appendSystemContext()              // Add system context (git status, cache breaker)
```

The system prompt is built from:
- **Static sections** (cacheable with `scope: 'global'`): intro, system rules, doing tasks, actions, tools, tone/style, output efficiency
- **Dynamic boundary marker**: `SYSTEM_PROMPT_DYNAMIC_BOUNDARY` (`__SYSTEM_PROMPT_DYNAMIC_BOUNDARY__`) separates cacheable from dynamic content
- **Dynamic sections** (runtime-computed): session guidance, memory, ant model override, env info, language, output style, MCP instructions, scratchpad, function result clearing, token budget, brief

When `CLAUDE_CODE_SIMPLE` is set, a minimal prompt is used:
```typescript
[`You are Claude Code...`, `CWD: ${cwd}`, `Date: ${date}`]
```

### 2. Message Preparation

```
queryLoop()
  → getMessagesAfterCompactBoundary()  // Only messages after last compact boundary
  → applyToolResultBudget()            // Enforce per-message budget on tool result size
  → snipCompactIfNeeded()              // Feature-gated history snipping
  → microcompactMessages()             // Feature-gated cache-aware micro-compaction
  → applyCollapsesIfNeeded()           // Feature-gated context collapse
  → prependUserContext()               // Add user context (claudeMd, currentDate)
```

### 3. API Request Parameters

The final request to `deps.callModel()` (which wraps `queryModelWithStreaming`) includes:
- `messages`: Prepared message array with user context prepended
- `systemPrompt`: Full system prompt with system context appended
- `thinkingConfig`: Adaptive/enabled/disabled thinking configuration
- `tools`: Available tool definitions converted to API schema
- `signal`: AbortController signal for cancellation
- `options`: Model, fast mode, tool choice, fallback model, query source, agents, task budget, etc.

## Message Building

Messages are constructed using factory functions in `utils/messages.ts`:

### Message Types

| Type | Factory Function | Purpose |
|------|-----------------|---------|
| `UserMessage` | `createUserMessage()` | User input, tool results, meta messages |
| `AssistantMessage` | N/A (from API) | Model responses with text, thinking, tool_use blocks |
| `SystemMessage` | `createSystemMessage()` | Internal notifications, warnings, API retries |
| `AttachmentMessage` | `createAttachmentMessage()` | Structured data attachments (max_turns, hook results) |
| `ProgressMessage` | N/A | Tool execution progress updates |
| `StreamEvent` | N/A (from API) | Raw SSE events (message_start, content_block_start, etc.) |
| `TombstoneMessage` | `{ type: 'tombstone' }` | Signals removal of orphaned messages from UI |
| `ToolUseSummaryMessage` | `createToolUseSummaryMessage()` | Summaries of tool use batches |
| `CompactBoundaryMessage` | `createMicrocompactBoundaryMessage()` | Marks compaction boundaries |

### Thinking Block Rules

The code documents strict rules for thinking blocks (query.ts:151-163):
1. Messages with thinking/redacted_thinking blocks require `max_thinking_length > 0`
2. Thinking blocks may not be the last message in a sequence
3. Thinking blocks must be preserved for the duration of an assistant trajectory

### Message Normalization

- `normalizeMessagesForAPI()` — Converts internal messages to API-compatible format
- `normalizeMessage()` — Normalizes messages for SDK yielding
- `ensureToolResultPairing()` — Strips duplicate tool_use IDs to prevent API errors
- `stripSignatureBlocks()` — Removes model-specific thinking signatures before fallback

## LLM API Interaction

### Streaming Client

The primary API interaction is through `queryModelWithStreaming()` in `services/api/claude.ts`:

```typescript
// QueryEngine.ts → queryLoop() → deps.callModel()
for await (const message of deps.callModel({
  messages: prependUserContext(messagesForQuery, userContext),
  systemPrompt: fullSystemPrompt,
  thinkingConfig: toolUseContext.options.thinkingConfig,
  tools: toolUseContext.options.tools,
  signal: toolUseContext.abortController.signal,
  options: { model, fastMode, fallbackModel, querySource, ... }
})) {
  // Process streamed messages
}
```

### Request Construction

The API request includes:
- **Betas**: Context management, prompt caching, structured outputs, task budgets, effort, fast mode, advisor, tool search
- **Tool definitions**: Converted via `toolToAPISchema()` with JSON schemas
- **Cache breakpoints**: System prompt split at `SYSTEM_PROMPT_DYNAMIC_BOUNDARY` for global vs. session-scoped caching
- **Model parameters**: `max_tokens`, `temperature`, `thinking` config, `effort`

### Response Processing

Streaming events are processed in order:
1. `message_start` — Resets usage tracking for the new message
2. `content_block_start` — New content block (text, thinking, tool_use)
3. `content_block_delta` — Streaming delta for current block
4. `content_block_stop` — Block complete; tool_use blocks are queued for execution
5. `message_delta` — Final stop_reason and usage delta
6. `message_stop` — Accumulates usage into total

## Streaming Responses

The streaming pipeline handles several scenarios:

### Normal Streaming

```
for await (const message of deps.callModel(...)) {
  // Withhold recoverable errors from SDK callers
  if (!withheld) yield yieldMessage
  if (message.type === 'assistant') {
    assistantMessages.push(message)
    // Queue tool_use blocks for streaming executor
    streamingToolExecutor?.addTool(toolBlock, message)
    // Emit completed tool results as they finish
    for (const result of streamingToolExecutor.getCompletedResults()) {
      yield result.message
    }
  }
}
```

### Streaming Fallback

When streaming fails mid-response:
1. `streamingFallbackOccured` flag is set
2. Orphaned assistant messages are tombstoned (removed from UI)
3. Pending tool results are discarded
4. A fresh `StreamingToolExecutor` is created
5. The request is retried (non-streaming fallback)

### Withholding Recoverable Errors

Certain errors are withheld from SDK callers until recovery is attempted:
- **Prompt too long** (413/400): Withheld so context collapse or reactive compact can retry
- **Max output tokens**: Withheld so the escalation/recovery loop can continue
- **Media size errors**: Withheld so reactive compact can strip images and retry

```typescript
let withheld = false
if (contextCollapse?.isWithheldPromptTooLong(message)) withheld = true
if (reactiveCompact?.isWithheldPromptTooLong(message)) withheld = true
if (reactiveCompact?.isWithheldMediaSizeError(message)) withheld = true
if (isWithheldMaxOutputTokens(message)) withheld = true
if (!withheld) yield yieldMessage
```

## Context Management

The query engine employs multiple context management strategies, applied in order:

### 1. Tool Result Budget

`applyToolResultBudget()` enforces per-message budget limits on aggregate tool result size. Replaces oversized tool results with truncated content.

### 2. History Snipping (feature: `HISTORY_SNIP`)

`snipCompactIfNeeded()` removes older message history while preserving a "protected tail." Used primarily in headless/SDK mode to bound memory in long sessions.

### 3. Micro-Compaction (feature: `CACHED_MICROCOMPACT`)

`microcompactMessages()` performs cache-aware micro-compaction that edits the prompt cache rather than summarizing content. Defers boundary message emission until after the API response to use actual `cache_deleted_input_tokens`.

### 4. Context Collapse (feature: `CONTEXT_COLLAPSE`)

`applyCollapsesIfNeeded()` projects a collapsed context view by archiving older messages and replacing them with summaries. Runs before autocompact so that if collapse gets under the threshold, autocompact is a no-op.

### 5. Auto-Compaction

`autoCompactIfNeeded()` triggers automatic summarization when token count approaches the context limit. Tracks:
- Pre/post compact token counts
- Compaction usage (input/output/cache tokens)
- Consecutive failures (circuit breaker)
- Turn counter since last compact

Post-compaction messages are built via `buildPostCompactMessages()`:
```
summaryMessages + attachments + hookResults + preserved tail
```

### 6. Reactive Compaction (feature: `REACTIVE_COMPACT`)

`tryReactiveCompact()` is a recovery mechanism triggered when the API returns a prompt-too-long error. It strips images or summarizes content to fit within limits.

### 7. Blocking Limit Check

Before making an API call, if not already compacted and not in a recovery path:
```typescript
const { isAtBlockingLimit } = calculateTokenWarningState(
  tokenCountWithEstimation(messagesForQuery) - snipTokensFreed,
  toolUseContext.options.mainLoopModel,
)
if (isAtBlockingLimit) {
  yield createAssistantAPIErrorMessage({ content: PROMPT_TOO_LONG_ERROR_MESSAGE })
  return { reason: 'blocking_limit' }
}
```

## Token Tracking

Token usage is tracked at multiple levels:

### Per-Message Tracking

```typescript
// Reset on message_start
currentMessageUsage = EMPTY_USAGE
currentMessageUsage = updateUsage(currentMessageUsage, message.event.message.usage)

// Updated on message_delta
currentMessageUsage = updateUsage(currentMessageUsage, message.event.usage)

// Accumulated on message_stop
this.totalUsage = accumulateUsage(this.totalUsage, currentMessageUsage)
```

### Session-Level Tracking

`cost-tracker.ts` maintains cumulative state:
- `totalCostUSD` — Total API cost in USD
- `totalInputTokens`, `totalOutputTokens` — Token counts
- `totalCacheReadInputTokens`, `totalCacheCreationInputTokens` — Cache token counts
- `totalAPIDuration`, `totalAPIDurationWithoutRetries` — Timing metrics
- `totalLinesAdded`, `totalLinesRemoved` — Code change metrics
- `modelUsage` — Per-model breakdown map

### Token Budget Feature (feature: `TOKEN_BUDGET`)

The `BudgetTracker` in `query/tokenBudget.ts` manages token budget continuations:

```typescript
// Decision logic
if (turnTokens < budget * 0.9 && !isDiminishing) {
  return { action: 'continue', nudgeMessage: '...' }
}
if (isDiminishing || continuationCount > 0) {
  return { action: 'stop', completionEvent: { ... } }
}
```

- **Completion threshold**: 90% of budget
- **Diminishing returns**: Stops if delta tokens < 500 for 3+ continuations
- **Nudge message**: Injected as a meta user message to continue the model

### API Task Budget

The `taskBudget` parameter (`total` + `remaining`) tracks the API-side task budget across compaction boundaries:
```typescript
// Capture pre-compact final context before messages are replaced
const preCompactContext = finalContextTokensFromLastResponse(messagesForQuery)
taskBudgetRemaining = Math.max(0, (taskBudgetRemaining ?? total) - preCompactContext)
```

## Cost Tracking

Costs are calculated in `cost-tracker.ts`:

```typescript
export function addToTotalSessionCost(cost: number, usage: Usage, model: string): number {
  const modelUsage = addToTotalModelUsage(cost, usage, model)
  addToTotalCostState(cost, modelUsage, model)
  
  // OTel metrics
  getCostCounter()?.add(cost, { model, speed })
  getTokenCounter()?.add(usage.input_tokens, { model, type: 'input' })
  // ... output, cacheRead, cacheCreation
}
```

Cost calculation uses `calculateUSDCost(model, usage)` from `utils/modelCost.ts` which applies per-model pricing.

### Cost Persistence

- `saveCurrentSessionCosts()` — Saves costs to project config before session switch
- `restoreCostStateForSession()` — Restores costs when resuming a session
- Costs include advisor tool token usage (separate model pricing)

### Result Reporting

The final `result` message includes:
```typescript
{
  type: 'result',
  subtype: 'success' | 'error_max_turns' | 'error_max_budget_usd' | 
           'error_max_structured_output_retries' | 'error_during_execution',
  duration_ms: number,
  duration_api_ms: number,
  num_turns: number,
  result: string,           // Extracted text from last assistant message
  stop_reason: string | null,
  session_id: string,
  total_cost_usd: number,
  usage: NonNullableUsage,
  modelUsage: { [model]: ModelUsage },
  permission_denials: SDKPermissionDenial[],
  structured_output: unknown,
  fast_mode_state: FastModeState,
  errors: string[],         // Turn-scoped error diagnostics
}
```

## Model Selection

### Initial Model Selection

```typescript
const initialMainLoopModel = userSpecifiedModel
  ? parseUserSpecifiedModel(userSpecifiedModel)
  : getMainLoopModel()
```

### Runtime Model Selection

```typescript
const currentModel = getRuntimeMainLoopModel({
  permissionMode,
  mainLoopModel: toolUseContext.options.mainLoopModel,
  exceeds200kTokens: permissionMode === 'plan' && doesMostRecentAssistantMessageExceed200k(messagesForQuery),
})
```

### Model Fallback

When `FallbackTriggeredError` is caught (after 3 consecutive 529 errors):
1. Switch to `fallbackModel`
2. Clear all assistant messages and tool results
3. Yield tombstones for orphaned messages
4. Strip thinking signature blocks (model-bound)
5. Create fresh `StreamingToolExecutor`
6. Yield system message about the fallback

```typescript
yield createSystemMessage(
  `Switched to ${renderModelName(fallbackModel)} due to high demand for ${renderModelName(originalModel)}`,
  'warning',
)
```

### Thinking Configuration

```typescript
const initialThinkingConfig = thinkingConfig ?? (
  shouldEnableThinkingByDefault() !== false
    ? { type: 'adaptive' }
    : { type: 'disabled' }
)
```

## Error Handling

The error handling pipeline in `services/api/errors.ts` classifies and formats API errors:

### Error Classification (`classifyAPIError`)

Returns standardized error type strings for analytics:
- `aborted`, `api_timeout`, `repeated_529`, `capacity_off_switch`
- `rate_limit`, `server_overload`, `prompt_too_long`
- `pdf_too_large`, `pdf_password_protected`, `image_too_large`
- `tool_use_mismatch`, `unexpected_tool_result`, `duplicate_tool_use_id`
- `invalid_model`, `credit_balance_low`, `invalid_api_key`
- `token_revoked`, `oauth_org_not_allowed`, `auth_error`
- `bedrock_model_access`, `ssl_cert_error`, `connection_error`
- `server_error`, `client_error`, `unknown`

### Error-to-Message Conversion (`getAssistantMessageFromError`)

Maps specific API errors to user-friendly assistant messages:
- **Timeout**: `API_TIMEOUT_ERROR_MESSAGE`
- **Rate limit (429)**: Parses unified rate limit headers, overage status, reset timestamps
- **Prompt too long (413/400)**: Generic message + raw error in `errorDetails` for reactive compact
- **Media size errors**: Per-variant messages (PDF, image, many-image)
- **Auth errors (401/403)**: Login prompts, org disabled, token revoked
- **Model errors (404)**: Model unavailable guidance with fallback suggestions
- **Tool use errors (400)**: Mismatch, unexpected tool result, duplicate IDs
- **Refusal**: Usage policy violation message

## Retry Logic

The retry system in `services/api/withRetry.ts` is comprehensive:

### `withRetry()` Generator

```typescript
export async function* withRetry<T>(
  getClient: () => Promise<Anthropic>,
  operation: (client, attempt, context) => Promise<T>,
  options: RetryOptions,
): AsyncGenerator<SystemAPIErrorMessage, T>
```

### Retry Configuration

| Parameter | Default | Description |
|-----------|---------|-------------|
| `DEFAULT_MAX_RETRIES` | 10 | Maximum retry attempts |
| `MAX_529_RETRIES` | 3 | Consecutive 529 errors before fallback |
| `BASE_DELAY_MS` | 500 | Base delay for exponential backoff |
| `FLOOR_OUTPUT_TOKENS` | 3000 | Minimum output tokens on context overflow |

### Retryable Errors (`shouldRetry`)

- **Always retry**: Mock errors (never), connection errors, 408 (timeout), 409 (lock), 500+ (server), 401 (auth — with cache clear)
- **Conditional retry**: 429 (rate limit — not for ClaudeAI subscribers unless enterprise), overloaded errors (529)
- **Never retry**: `x-should-retry: false` header (except ant 5xx), mock rate limits

### Exponential Backoff

```typescript
export function getRetryDelay(attempt: number, retryAfterHeader?: string | null, maxDelayMs = 32000): number {
  if (retryAfterHeader) return parseInt(retryAfterHeader, 10) * 1000
  const baseDelay = Math.min(BASE_DELAY_MS * Math.pow(2, attempt - 1), maxDelayMs)
  const jitter = Math.random() * 0.25 * baseDelay
  return baseDelay + jitter
}
```

### Persistent Retry (feature: `UNATTENDED_RETRY`)

When `CLAUDE_CODE_UNATTENDED_RETRY` is enabled:
- Retries 429/529 indefinitely
- Max backoff: 5 minutes
- Reset cap: 6 hours
- Heartbeat: Yields `api_retry` system messages every 30 seconds to prevent host idle detection

### Fast Mode Fallback

On 429/529 with fast mode active:
- **Short retry-after** (< 20s): Wait and retry with fast mode still active (preserves prompt cache)
- **Long/unknown retry-after**: Enter cooldown (10 min minimum), switch to standard speed

### 529 Fallback Trigger

After 3 consecutive 529 errors, throws `FallbackTriggeredError` to trigger model fallback (Opus → Sonnet).

### Foreground vs Background Retry

Only foreground query sources retry on 529:
```typescript
const FOREGROUND_529_RETRY_SOURCES = new Set([
  'repl_main_thread', 'sdk', 'agent:custom', 'agent:default', 
  'agent:builtin', 'compact', 'hook_agent', 'hook_prompt',
  'verification_agent', 'side_question', 'auto_mode', 'bash_classifier'
])
```

## Key Algorithms

### Max Output Tokens Recovery

When the model hits its output token limit:
1. **Escalation** (feature: `tengu_otk_slot_v1`): If using default 8k cap, retry at 64k
2. **Multi-turn recovery**: Inject meta message asking model to resume, up to `MAX_OUTPUT_TOKENS_RECOVERY_LIMIT` (3) times
3. **Exhausted**: Surface the withheld error

```typescript
const recoveryMessage = createUserMessage({
  content: `Output token limit hit. Resume directly — no apology, no recap...`,
  isMeta: true,
})
```

### Prompt Too Long Recovery

When the API returns a prompt-too-long error:
1. **Collapse drain**: First try draining staged context collapses (keeps granular context)
2. **Reactive compact**: Strip images or summarize to fit
3. **No recovery**: Surface the error, run stop failure hooks

### Streaming Tool Execution

Two execution modes:
1. **Streaming** (`tengu_streaming_tool_execution2` gate): Tools execute concurrently while the model continues streaming. Uses `StreamingToolExecutor` to manage in-flight tools.
2. **Batch**: After streaming completes, `runTools()` executes all tool_use blocks sequentially.

### Structured Output Retry

When using JSON schema output:
- Tracks `SYNTHETIC_OUTPUT_TOOL_NAME` calls per query
- Max retries: `MAX_STRUCTURED_OUTPUT_RETRIES` (default 5)
- Exceeded: Yields `error_max_structured_output_retries` result

### Tool Use Summary Generation

After a tool batch completes (feature-gated):
1. Extracts last assistant text block for context
2. Collects tool name, input, and output for each tool
3. Fires off Haiku-based summary generation asynchronously
4. Summary is yielded at the start of the next iteration

## Dependencies

### Internal Dependencies

| Module | Purpose |
|--------|---------|
| `services/api/claude.ts` | `queryModelWithStreaming()` — API streaming client |
| `services/api/withRetry.ts` | `withRetry()` — Retry wrapper with exponential backoff |
| `services/api/errors.ts` | Error classification and message generation |
| `services/api/logging.ts` | API telemetry (query, success, error events) |
| `services/compact/autoCompact.ts` | `autoCompactIfNeeded()` — Automatic context compaction |
| `services/compact/microCompact.ts` | `microcompactMessages()` — Cache-aware micro-compaction |
| `services/compact/snipCompact.ts` | Feature-gated history snipping |
| `services/compact/reactiveCompact.ts` | Feature-gated reactive compaction recovery |
| `services/contextCollapse/` | Feature-gated context collapse |
| `services/tools/toolOrchestration.ts` | `runTools()` — Batch tool execution |
| `services/tools/StreamingToolExecutor.ts` | Concurrent tool execution during streaming |
| `services/toolUseSummary/` | Tool use summary generation |
| `utils/messages.ts` | Message factory functions and normalization |
| `utils/systemPromptType.ts` | `SystemPrompt` branded type |
| `utils/attachments.ts` | Attachment message creation and prefetching |
| `utils/model/model.ts` | Model selection and parsing |
| `utils/tokens.ts` | Token counting and estimation |
| `utils/queryContext.ts` | `fetchSystemPromptParts()` |
| `utils/processUserInput/` | User input and slash command processing |
| `utils/sessionStorage.ts` | Transcript recording and flushing |
| `bootstrap/state.ts` | Global state management (cost, tokens, session) |
| `cost-tracker.ts` | Cost calculation and tracking |
| `context.ts` | User and system context builders |
| `constants/prompts.ts` | System prompt section builders |
| `constants/messages.ts` | Message constants |
| `constants/betas.ts` | API beta header constants |
| `constants/apiLimits.ts` | API limit constants |

### External Dependencies

| Package | Purpose |
|---------|---------|
| `@anthropic-ai/sdk` | Anthropic API client, types, error classes |
| `lodash-es/last.js` | Array last element utility |
| `strip-ansi` | ANSI code stripping for SDK output |
| `crypto.randomUUID` | UUID generation |
| `bun:bundle` | Feature flag system (`feature()`) |

## Data Flow

```
User Input
    │
    ▼
┌─────────────────────────────────────────┐
│           QueryEngine.submitMessage()    │
│                                         │
│  1. Fetch system prompt parts           │
│  2. Process user input (slash commands) │
│  3. Push messages to mutableMessages    │
│  4. Record transcript                   │
│  5. Yield system init message           │
│  6. Enter query loop                    │
└──────────────────┬──────────────────────┘
                   │
                   ▼
┌─────────────────────────────────────────┐
│              query() / queryLoop()       │
│                                         │
│  ┌─ Iteration ──────────────────────┐   │
│  │                                  │   │
│  │  1. Apply snip (feature-gated)   │   │
│  │  2. Apply microcompact           │   │
│  │  3. Apply context collapse       │   │
│  │  4. Apply autocomact if needed   │   │
│  │  5. Check blocking limit         │   │
│  │  6. Call model (streaming)       │   │
│  │     - Withhold recoverable errors│   │
│  │     - Queue tool_use blocks      │   │
│  │     - Stream tool results        │   │
│  │  7. Execute tools                │   │
│  │  8. Get attachment messages      │   │
│  │  9. Run stop hooks               │   │
│  │  10. Check token budget          │   │
│  │  11. Continue or return          │   │
│  └──────────────────────────────────┘   │
│                                         │
│  Recovery paths:                        │
│  - Prompt too long → collapse → reactive│
│  - Max output tokens → escalate → retry │
│  - 529 errors → fallback model          │
│  - Abort → synthetic tool_results       │
└──────────────────┬──────────────────────┘
                   │
                   ▼
┌─────────────────────────────────────────┐
│           QueryEngine result yield       │
│                                         │
│  - success / error_* subtype            │
│  - duration, cost, usage, turns         │
│  - permission denials, fast mode state  │
│  - structured output (if applicable)    │
└─────────────────────────────────────────┘
```

## Integration Points

### Tool System

- **Tool Permission**: `canUseTool` callback wraps permission checking and tracks denials
- **Tool Execution**: `StreamingToolExecutor` for concurrent execution; `runTools()` for batch
- **Tool Results**: Normalized via `normalizeMessagesForAPI()` for API compatibility
- **Tool Use Summary**: Async Haiku call generates summaries of tool batches

### State Management

- **AppState**: Read/written via `getAppState()`/`setAppState()` for tool permissions, fast mode, file history, attribution
- **Session State**: Global state in `bootstrap/state.ts` for cost, tokens, duration, session ID
- **File State Cache**: `FileStateCache` tracks read files; cloned per `QueryEngine` instance

### MCP (Model Context Protocol)

- **MCP Clients**: Passed through config; instructions included in system prompt
- **MCP Tools**: Available alongside built-in tools; URL elicitation handling for -32042 errors
- **MCP Resources**: Tracked in `mcpResources` on tool use context

### Hooks System

- **Stop Hooks**: Execute after model response completes
- **TaskCompleted Hooks**: Run for in-progress tasks owned by the current teammate
- **TeammateIdle Hooks**: Run when a teammate becomes idle
- **Post-Sampling Hooks**: Fire-and-forget after sampling (prompt suggestion, memory extraction, auto-dream)

### Memory System

- **Memory Prefetch**: `startRelevantMemoryPrefetch()` runs asynchronously each iteration
- **Memory Loading**: `loadMemoryPrompt()` injected when custom system prompt + memory path override
- **Memory Correction**: Hint appended to rejection messages when auto-memory is enabled

### Transcript/Session Storage

- **Record Transcript**: User messages recorded immediately; assistant messages fire-and-forget
- **Flush Session Storage**: Before yielding final result to prevent data loss on process kill
- **Compact Boundary**: Flushes preserved segment tail before writing boundary

### Analytics

- **Statsig/GrowthBook**: Feature gates, A/B test values
- **Event Logging**: `logEvent()` for query start/stop, compaction, fallback, tool execution, budget
- **OTLP Telemetry**: `logOTelEvent()` for API requests and errors
- **Session Tracing**: Beta tracing with spans for LLM requests

### SDK Integration

- **SDK Messages**: All yields are converted to `SDKMessage` format for external consumers
- **SDK Status**: `setSDKStatus()` callback for lifecycle updates
- **Permission Denials**: Tracked and reported in final result
- **Fast Mode State**: Reported in result for SDK callers

## Related Modules

- [Main Entrypoint](./main-entrypoint.md)
- [Tool System](./tool-system.md)
- [State Management](./state-management.md)
