# Message Flow

## Purpose

Traces the complete lifecycle of messages through the Claude Code system — from user input, through the query engine and API, to tool execution and transcript persistence.

## Location

Primary sources:
- `restored-src/src/main.tsx` — entry point and session setup
- `restored-src/src/QueryEngine.ts` — message lifecycle orchestration
- `restored-src/src/utils/messages.ts` — message creation and normalization
- `restored-src/src/utils/processUserInput/processUserInput.ts` — user input processing
- `restored-src/src/query.ts` — API query loop (referenced by QueryEngine)

---

## Message Types

The system uses a discriminated union `Message` type with the following variants:

| Type | Subtype (if applicable) | Purpose |
|------|------------------------|---------|
| `user` | — | User prompts, tool results, slash command output |
| `assistant` | — | AI responses with text, thinking, or tool_use blocks |
| `system` | `compact_boundary` | Marks where conversation was compacted |
| `system` | `api_error` / `api_retry` | API error and retry notifications |
| `system` | `local_command` | Local command output (not sent to API) |
| `system` | `informational` | Status/info messages for the UI |
| `system` | `warning` | Warning messages (e.g., permission mode changes) |
| `system` | `permission_retry` | Permission retry markers |
| `progress` | — | Real-time tool execution progress updates |
| `attachment` | — | System attachments (skills, deferred tools, MCP deltas) |
| `stream_event` | — | Raw API streaming events (message_start, delta, stop) |
| `stream_request_start` | — | Marks start of an API request |
| `tombstone` | — | Control signal for message removal |
| `tool_use_summary` | — | Summarized tool use for SDK consumers |

Key message properties:
- `uuid` — unique identifier for transcript deduplication
- `parentUuid` — links child messages to parents (for branching)
- `toolUseResult` — tool execution results on user messages
- `isMeta` — marks synthetic/system-generated user messages
- `isCompactSummary` — marks compaction summary messages

---

## Complete Message Flow

### Phase 1: Session Initialization

```
main.tsx
  │
  ├─► runMigrations()            — apply settings migrations
  ├─► init()                     — load settings, auth, MDM
  ├─► initializeToolPermissionContext() — set up permission rules
  ├─► getTools()                 — build tool registry
  ├─► createStore(initialState)  — create AppState store
  │
  └─► launchRepl() or ask()      — enter interactive or headless mode
```

### Phase 2: User Input Processing

```
User enters prompt (REPL or SDK)
  │
  ▼
QueryEngine.submitMessage(prompt)
  │
  ├─► fetchSystemPromptParts()   — build system prompt + user context
  ├─► processUserInput()         — handle slash commands, attachments
  │    │
  │    ├─► parse slash commands (/compact, /model, etc.)
  │    ├─► resolve file/image attachments
  │    ├─► inject skill discoveries
  │    └─► createUserMessage()   — create typed UserMessage
  │
  ├─► mutableMessages.push(...newMessages)
  │
  └─► recordTranscript(messages) — persist to disk BEFORE API call
```

### Phase 3: Query Loop

```
query({ messages, systemPrompt, tools, canUseTool, ... })
  │
  ├─► normalizeMessagesForAPI()  — filter UI-only messages
  ├─► queryModelWithStreaming()  — call Anthropic API
  │    │
  │    ├─► stream_event: message_start
  │    ├─► stream_event: content_block_start (text/thinking/tool_use)
  │    ├─► stream_event: content_block_delta (streaming text)
  │    ├─► stream_event: content_block_stop
  │    └─► stream_event: message_delta (stop_reason, usage)
  │
  ├─► For each assistant content block:
  │    │
  │    ├─► If tool_use block:
  │    │    │
  │    │    ├─► canUseTool(tool, input, context, toolUseID)
  │    │    │    │
  │    │    │    ├─► validateInput()       — tool-specific validation
  │    │    │    ├─► checkPermissions()    — permission decision
  │    │    │    │    │
  │    │    │    │    ├─► 'allow'  → execute tool
  │    │    │    │    ├─► 'deny'   → return error to API
  │    │    │    │    └─► 'ask'    → show permission dialog
  │    │    │    │                    │
  │    │    │    │                    ├─► user allows → execute
  │    │    │    │                    └─► user denies → error
  │    │    │    │
  │    │    │    └─► tool.call(args, context, canUseTool)
  │    │    │         │
  │    │    │         ├─► onProgress() calls → yield ProgressMessage
  │    │    │         └─► returns ToolResult
  │    │    │
  │    │    ├─► ToolResult → createUserMessage(tool_result)
  │    │    └─► Push to mutableMessages
  │    │
  │    └─► If text/thinking block:
  │         └─► yield AssistantMessage (already streamed)
  │
  ├─► Check termination conditions:
  │    ├─► stop_reason === 'end_turn' → done
  │    ├─► maxTurns exceeded → yield error result
  │    ├─► maxBudgetUsd exceeded → yield error result
  │    └─► structured output retries exceeded → yield error result
  │
  └─► Return final result message
```

### Phase 4: Message Normalization and SDK Mapping

```
QueryEngine receives message from query()
  │
  ├─► switch (message.type):
  │    │
  │    ├─► 'assistant':
  │    │    ├─► Push to mutableMessages
  │    │    └─► yield normalizeMessage() → SDKAssistantMessage
  │    │
  │    ├─► 'user':
  │    │    ├─► Push to mutableMessages
  │    │    └─► yield normalizeMessage() → SDKUserMessage
  │    │
  │    ├─► 'progress':
  │    │    ├─► Push to mutableMessages
  │    │    ├─► recordTranscript() (fire-and-forget)
  │    │    └─► yield normalizeMessage()
  │    │
  │    ├─► 'attachment':
  │    │    ├─► Push to mutableMessages
  │    │    ├─► Handle special types:
  │    │    │    ├─► 'structured_output' → extract data
  │    │    │    ├─► 'max_turns_reached' → yield error result
  │    │    │    └─► 'queued_command' → replay as user message
  │    │    └─► recordTranscript() (fire-and-forget)
  │    │
  │    ├─► 'system':
  │    │    ├─► Handle snip boundary (if HISTORY_SNIP enabled)
  │    │    ├─► Push to mutableMessages
  │    │    ├─► If compact_boundary:
  │    │    │    ├─► Release pre-compaction messages for GC
  │    │    │    └─► yield SDKCompactBoundaryMessage
  │    │    └─► If api_error: yield SDKApiRetryMessage
  │    │
  │    ├─► 'stream_event':
  │    │    ├─► Track usage (message_start, message_delta, message_stop)
  │    │    └─► Yield only if includePartialMessages
  │    │
  │    └─► 'tombstone': skip (control signal)
  │
  └─► Final result message with usage, cost, duration
```

### Phase 5: Transcript Persistence

```
recordTranscript(messages)
  │
  ├─► Build message chain with parentUuid links
  ├─► Handle deduplication (skip already-recorded UUIDs)
  ├─► Write to transcript file (JSONL format)
  │    └─► ~/.claude/projects/<cwd>/.claude/transcripts/<sessionId>.jsonl
  │
  └─► flushSessionStorage() — when EAGER_FLUSH or COWORK mode
```

---

## Key Data Structures

### QueryEngine Internal State

```
QueryEngine {
  mutableMessages: Message[]      — working message array
  permissionDenials: SDKPermissionDenial[]  — tracked denials
  totalUsage: NonNullableUsage    — accumulated token usage
  discoveredSkillNames: Set<string>  — skill tracking per turn
  loadedNestedMemoryPaths: Set<string>  — memory dedup
  abortController: AbortController  — interrupt support
}
```

### Message Flow Through QueryEngine.submitMessage()

```
1. Clear per-turn tracking (discoveredSkillNames)
2. Fetch system prompt parts (tools, context, MCP)
3. Build ProcessUserInputContext
4. Handle orphaned permission (if any)
5. Process user input (slash commands, attachments)
6. Push new messages to mutableMessages
7. Record transcript (before API call)
8. Build system init message (yield first)
9. Enter query() loop
10. For each yielded message:
    - Classify by type
    - Push to mutableMessages
    - Record transcript
    - Yield normalized SDK message
11. Check termination conditions
12. Yield final result message
```

---

## Integration Points

| Component | Role in Message Flow |
|-----------|---------------------|
| `processUserInput` | Transforms raw input into typed messages |
| `query.ts` | Drives the API streaming loop |
| `canUseTool` | Gates tool execution with permission checks |
| `normalizeMessage` | Maps internal messages to SDK format |
| `recordTranscript` | Persists messages to disk |
| `compact.ts` | Replaces messages with compacted summaries |

## Related Documentation

- [Permission Flow](./permission-flow.md)
- [Tool Execution Flow](./tool-execution-flow.md)
- [Session Lifecycle](./session-lifecycle.md)
- [Tool System](../01-core-modules/tool-system.md)
- [Query Engine](../01-core-modules/query-engine.md)
