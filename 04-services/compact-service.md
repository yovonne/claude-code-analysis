# Compact Service

## Purpose

The compact service manages conversation context reduction to prevent the model's context window from being exceeded. It provides multiple compaction strategies — full conversation summarization, partial compaction around a pivot point, session memory-based compaction, and micro-level tool result pruning — each triggered by different conditions. The service handles the complete lifecycle: pre-compaction hooks, API-based summarization, post-compaction context restoration, and cleanup of invalidated caches.

## Location

| File | Lines | Role |
|------|-------|------|
| `restored-src/src/services/compact/compact.ts` | 1706 | Core compaction: full and partial conversation summarization |
| `restored-src/src/services/compact/autoCompact.ts` | 352 | Automatic compaction triggering based on token thresholds |
| `restored-src/src/services/compact/microCompact.ts` | 531 | Tool result pruning (cached and time-based) |
| `restored-src/src/services/compact/sessionMemoryCompact.ts` | 631 | Session memory-based compaction alternative |
| `restored-src/src/services/compact/prompt.ts` | 375 | Summary prompt templates and formatting |
| `restored-src/src/services/compact/grouping.ts` | 64 | API-round message grouping for compaction |
| `restored-src/src/services/compact/timeBasedMCConfig.ts` | 44 | Time-based microcompact configuration |
| `restored-src/src/services/compact/postCompactCleanup.ts` | 78 | Post-compaction cache and state cleanup |
| `restored-src/src/services/compact/compactWarningState.ts` | 19 | Compact warning suppression state |
| `restored-src/src/services/compact/compactWarningHook.ts` | 17 | React hook for compact warning suppression |
| `restored-src/src/services/compact/apiMicrocompact.ts` | 154 | Server-side context management API strategies |

## Key Exports

### compact.ts

| Export | Description |
|--------|-------------|
| `compactConversation(messages, context, ...)` | Full conversation compaction — summarizes older messages, preserves recent |
| `partialCompactConversation(allMessages, pivotIndex, context, direction)` | Partial compaction around a pivot point (`'from'` or `'up_to'`) |
| `CompactionResult` (type) | `{ boundaryMarker, summaryMessages, attachments, hookResults, messagesToKeep?, ... }` |
| `RecompactionInfo` (type) | Context for analytics: `isRecompactionInChain`, `turnsSincePreviousCompact`, etc. |
| `buildPostCompactMessages(result)` | Constructs consistent post-compact message array |
| `annotateBoundaryWithPreservedSegment(boundary, anchorUuid, messagesToKeep)` | Adds relink metadata for preserved message segments |
| `mergeHookInstructions(userInstructions, hookInstructions)` | Merges user and hook-provided custom instructions |
| `stripImagesFromMessages(messages)` | Removes image/document blocks before compaction |
| `stripReinjectedAttachments(messages)` | Removes skill_discovery/skill_listing attachments |
| `truncateHeadForPTLRetry(messages, ptlResponse)` | Emergency truncation when compact request itself hits prompt-too-long |
| `createPostCompactFileAttachments(readFileState, context, maxFiles)` | Re-injects recently accessed files post-compaction |
| `createPlanAttachmentIfNeeded(agentId)` | Preserves plan file across compaction |
| `createSkillAttachmentIfNeeded(agentId)` | Preserves invoked skill content across compaction |
| `createAsyncAgentAttachmentsIfNeeded(context)` | Preserves async agent state across compaction |
| `createPlanModeAttachmentIfNeeded(context)` | Preserves plan mode instructions across compaction |

### autoCompact.ts

| Export | Description |
|--------|-------------|
| `shouldAutoCompact(messages, model, querySource?, snipTokensFreed)` | Determines if auto-compact should fire |
| `autoCompactIfNeeded(messages, toolUseContext, cacheSafeParams, ...)` | Executes auto-compact with session memory fallback and circuit breaker |
| `isAutoCompactEnabled()` | Checks env vars and user config for auto-compact enablement |
| `getAutoCompactThreshold(model)` | Calculates token threshold for auto-compact trigger |
| `getEffectiveContextWindowSize(model)` | Context window minus reserved summary tokens |
| `calculateTokenWarningState(tokenUsage, model)` | Returns warning/error/auto-compact threshold states |
| `AutoCompactTrackingState` (type) | Tracks `compacted`, `turnCounter`, `turnId`, `consecutiveFailures` |

### microCompact.ts

| Export | Description |
|--------|-------------|
| `microcompactMessages(messages, toolUseContext?, querySource?)` | Main entry — tries time-based, then cached MC, returns unchanged if neither fires |
| `estimateMessageTokens(messages)` | Rough token estimation for messages (pads by 4/3) |
| `evaluateTimeBasedTrigger(messages, querySource)` | Checks if time-based trigger should fire |
| `consumePendingCacheEdits()` | Returns and clears pending cache edits for API layer |
| `getPinnedCacheEdits()` | Returns previously-pinned cache edits for cache hits |
| `pinCacheEdits(userMessageIndex, block)` | Pins a cache_edits block to a message position |
| `markToolsSentToAPIState()` | Marks all registered tools as sent to API |
| `resetMicrocompactState()` | Resets all microcompact state |

### sessionMemoryCompact.ts

| Export | Description |
|--------|-------------|
| `trySessionMemoryCompaction(messages, agentId?, autoCompactThreshold?)` | Attempts session memory-based compaction; returns `null` if not applicable |
| `shouldUseSessionMemoryCompaction()` | Checks feature flags and env vars |
| `calculateMessagesToKeepIndex(messages, lastSummarizedIndex)` | Calculates start index for preserved messages |
| `adjustIndexToPreserveAPIInvariants(messages, startIndex)` | Ensures tool_use/tool_result pairs aren't split |
| `hasTextBlocks(message)` | Checks if message contains text content |
| `SessionMemoryCompactConfig` (type) | `{ minTokens, minTextBlockMessages, maxTokens }` |

### prompt.ts

| Export | Description |
|--------|-------------|
| `getCompactPrompt(customInstructions?)` | Returns full compaction prompt template |
| `getPartialCompactPrompt(customInstructions?, direction)` | Returns partial compaction prompt (direction-aware) |
| `formatCompactSummary(summary)` | Strips `<analysis>` tags, formats `<summary>` tags into headers |
| `getCompactUserSummaryMessage(summary, suppressFollowUpQuestions?, transcriptPath?, recentMessagesPreserved?)` | Wraps summary for injection into context |

### grouping.ts

| Export | Description |
|--------|-------------|
| `groupMessagesByApiRound(messages)` | Groups messages at API-round boundaries (one group per API round-trip) |

### postCompactCleanup.ts

| Export | Description |
|--------|-------------|
| `runPostCompactCleanup(querySource?)` | Clears caches, tracking state, and module-level state after compaction |

### timeBasedMCConfig.ts

| Export | Description |
|--------|-------------|
| `getTimeBasedMCConfig()` | Returns GrowthBook config for time-based microcompact |
| `TimeBasedMCConfig` (type) | `{ enabled, gapThresholdMinutes, keepRecent }` |

### apiMicrocompact.ts

| Export | Description |
|--------|-------------|
| `getAPIContextManagement(options?)` | Returns server-side context management strategies |
| `ContextEditStrategy` (type) | `clear_tool_uses_20250919` or `clear_thinking_20251015` strategies |
| `ContextManagementConfig` (type) | `{ edits: ContextEditStrategy[] }` |

## Dependencies

### Internal

| Module | Purpose |
|--------|---------|
| `services/api/claude.ts` | `queryModelWithStreaming`, `getMaxOutputTokensForModel` |
| `services/api/errors.ts` | Prompt-too-long detection and token gap extraction |
| `services/api/promptCacheBreakDetection.ts` | Cache break notifications |
| `services/SessionMemory/prompts.ts` | Session memory content and truncation |
| `services/SessionMemory/sessionMemoryUtils.ts` | Session memory state management |
| `services/analytics/growthbook.ts` | Feature flag gates |
| `services/analytics/index.ts` | Event logging |
| `services/tokenEstimation.ts` | Rough token counting |
| `services/contextCollapse/` | Context collapse suppression (when enabled) |
| `utils/forkedAgent.ts` | Forked agent for cache-sharing compaction |
| `utils/messages.ts` | Message creation, boundary detection, normalization |
| `utils/tokens.ts` | Token counting from API responses |
| `utils/hooks.ts` | Pre/post compact hook execution |
| `utils/attachments.ts` | Post-compact file restoration |
| `utils/context.ts` | Context window sizes |
| `utils/contextAnalysis.ts` | Token breakdown for analytics |
| `utils/plans.ts` | Plan file access |
| `utils/sessionStorage.ts` | Transcript path, session metadata |
| `utils/sessionStart.ts` | Session start hook processing |
| `utils/config.ts` | Global config access |
| `utils/toolSearch.ts` | Tool discovery post-compaction |
| `tools/FileReadTool/` | Post-compact file re-reading |

### External

| Package | Purpose |
|---------|---------|
| `@anthropic-ai/sdk` | API client, `APIUserAbortError` |
| `lodash-es/uniqBy` | Tool deduplication |
| `react` | Compact warning hook (isolated file) |

## Implementation Details

### Context Compaction Algorithms

#### Full Compaction (`compactConversation`)

The primary compaction strategy summarizes the entire conversation while preserving the most recent context:

```
1. Execute PreCompact hooks
2. Build compact prompt with analysis instructions
3. Stream summary via API (forked agent with cache sharing, or direct streaming)
4. Handle prompt-too-long retries (truncate oldest groups)
5. Clear file read state and loaded memory
6. Generate post-compact attachments (files, agents, plan, skills, tools, MCP)
7. Execute SessionStart hooks
8. Create boundary marker and summary messages
9. Execute PostCompact hooks
10. Return CompactionResult with all components
```

**Cache Sharing Optimization**: When `tengu_compact_cache_prefix` is enabled (default: true), uses a forked agent that reuses the main conversation's cached prompt prefix (system prompt, tools, context messages). Falls back to direct streaming on failure.

**Prompt-Too-Long Retry**: If the compact request itself exceeds the prompt limit:
- Strips oldest API-round groups progressively until under limit
- Max 3 retries
- Prepends synthetic user marker if first remaining message is assistant
- Falls back to 20% group drop when token gap is unparseable

#### Partial Compaction (`partialCompactConversation`)

Summarizes a portion of the conversation around a user-selected pivot point:

**Direction `'from'`** (suffix-preserving):
- Summarizes messages AFTER the pivot index
- Keeps messages BEFORE the pivot intact
- Prompt cache for kept messages is preserved
- Summary appears after kept messages

**Direction `'up_to'`** (prefix-preserving):
- Summarizes messages BEFORE the pivot index
- Keeps messages AFTER the pivot intact
- Prompt cache is invalidated (summary precedes kept messages)
- Strips old compact boundaries from kept messages
- Summary appears before kept messages

#### Session Memory Compaction (`trySessionMemoryCompaction`)

An alternative to API-based summarization that uses pre-extracted session memory:

```
1. Check feature flags (tengu_session_memory + tengu_sm_compact)
2. Wait for in-progress session memory extraction
3. Get session memory content and lastSummarizedMessageId
4. Calculate messages to keep (expands to meet minimums):
   - At least minTokens (default: 10,000)
   - At least minTextBlockMessages (default: 5)
   - Stops at maxTokens (default: 40,000)
5. Adjust index to preserve tool_use/tool_result pairs
6. Filter out old compact boundaries from kept messages
7. Create CompactionResult using session memory as summary
8. Check post-compact size against auto-compact threshold
```

**Two Scenarios**:
- **Normal**: `lastSummarizedMessageId` is set — keeps only messages after that ID
- **Resumed session**: No `lastSummarizedMessageId` but session memory has content — keeps all messages but uses session memory as summary

#### Microcompact (`microcompactMessages`)

Lightweight, per-turn tool result pruning that runs before every API call:

**Time-Based Trigger** (runs first, short-circuits):
- Fires when gap since last assistant message exceeds `gapThresholdMinutes` (default: 60 min)
- Content-clears all but the most recent N compactable tool results
- Resets cached MC state (cache is cold after content change)
- Mutates message content directly

**Cached Microcompact** (ant-only, feature-gated):
- Uses cache editing API to remove tool results without invalidating cached prefix
- Does NOT modify local message content — adds `cache_reference` and `cache_edits` at API layer
- Count-based trigger: when registered tool results exceed threshold, oldest are marked for deletion
- Tracks tool results grouped by user message
- Returns `pendingCacheEdits` to be included in next API request

**Compactable Tools**: FileRead, Bash, PowerShell, REPL, Grep, Glob, WebSearch, WebFetch, FileEdit, FileWrite

### Message Summarization

#### Prompt Structure

The compact prompt instructs the model to produce a structured summary with 9 sections:

1. **Primary Request and Intent** — User's explicit requests
2. **Key Technical Concepts** — Technologies, frameworks, patterns
3. **Files and Code Sections** — Examined/modified files with snippets
4. **Errors and Fixes** — Errors encountered and resolutions
5. **Problem Solving** — Solved problems and troubleshooting
6. **All User Messages** — Non-tool-result user messages
7. **Pending Tasks** — Outstanding work items
8. **Current Work** — What was being worked on at compaction time
9. **Optional Next Step** — Next action with verbatim quotes

#### Analysis Phase

The prompt includes an `<analysis>` section as a drafting scratchpad that is stripped before the summary reaches context. This improves summary quality by forcing the model to organize its thoughts before producing the final output.

#### No-Tools Enforcement

Both preamble and trailer explicitly forbid tool calls during compaction:
- Preamble: "CRITICAL: Respond with TEXT ONLY. Do NOT call any tools."
- Trailer: "REMINDER: Do NOT call any tools. Respond with plain text only."

This prevents the model from wasting its single turn on tool calls, which would result in no text output.

#### Summary Formatting

`formatCompactSummary()` processes the raw API response:
1. Strips `<analysis>...</analysis>` block entirely
2. Replaces `<summary>...</summary>` tags with `Summary:\n` header
3. Cleans up extra whitespace between sections

### Token Optimization Strategies

#### Post-Compact Context Restoration

After compaction, the following attachments are re-injected to restore model context:

| Attachment | Purpose | Token Budget |
|-----------|---------|-------------|
| Recent files (max 5) | Re-reads recently accessed files | 50,000 total, 5,000 per file |
| Plan file | Preserves current plan | Unbounded |
| Invoked skills | Preserves skill guidelines | 25,000 total, 5,000 per skill |
| Plan mode | Preserves plan mode state | Unbounded |
| Async agents | Preserves agent state | Unbounded |
| Deferred tools delta | Re-announces available tools | Computed |
| Agent listing delta | Re-announces available agents | Computed |
| MCP instructions delta | Re-announces MCP tools | Computed |

**File Selection**: Files sorted by recency, filtered to exclude plan files and CLAUDE.md files, deduplicated against Read tool results already in preserved messages, then budget-constrained.

**Skill Truncation**: Per-skill truncation keeps the head of each skill file (setup/usage instructions) rather than dropping whole skills. Uses `~4 chars/token` estimation.

#### Image Stripping

Before sending to the summarization API:
- Image blocks replaced with `[image]` text marker
- Document blocks replaced with `[document]` text marker
- Images within `tool_result` content also stripped
- Prevents the compaction API call itself from hitting prompt-too-long

#### Reinjected Attachment Stripping

`skill_discovery` and `skill_listing` attachments are removed before compaction because they are re-surfaced by `resetSentSkillNames()` and the next turn's discovery signal. Feeding them to the summarizer wastes tokens and pollutes the summary with stale suggestions.

### When Compaction Is Triggered

#### Auto-Compact

Triggered when token usage exceeds the auto-compact threshold:

```
threshold = effectiveContextWindow - AUTOCOMPACT_BUFFER_TOKENS
effectiveContextWindow = contextWindow - MAX_OUTPUT_TOKENS_FOR_SUMMARY
AUTOCOMPACT_BUFFER_TOKENS = 13,000
```

**Suppression Conditions** (auto-compact will NOT fire):
- `DISABLE_COMPACT` or `DISABLE_AUTO_COMPACT` env vars
- User config has `autoCompactEnabled: false`
- `querySource` is `session_memory`, `compact`, or `marble_origami` (prevents deadlocks)
- `REACTIVE_COMPACT` feature flag with `tengu_cobalt_raccoon` enabled
- `CONTEXT_COLLAPSE` feature enabled (collapse owns context management)
- Circuit breaker: 3+ consecutive failures

**Execution Flow**:
```
1. shouldAutoCompact() — check token count vs threshold
2. trySessionMemoryCompaction() — try SM-first (returns null if not applicable)
3. compactConversation() — fall back to full API-based compaction
4. runPostCompactCleanup() — clear caches and tracking state
5. Track consecutiveFailures for circuit breaker
```

#### Manual Compact

Triggered by the `/compact` command. Same `compactConversation()` path but:
- Does NOT suppress user questions
- Shows error notifications on failure (auto-compact does not)
- No recompaction info tracking

#### Partial Compact

Triggered by message selector UI. User picks a pivot message and direction.

#### Reactive Compact

Triggered when the API returns a prompt-too-long error. Falls back to auto-compact if enabled.

#### Microcompact (Per-Turn)

Runs before every API call on the main thread:
- Time-based: fires after configured idle gap (default: 60 min)
- Cached MC: fires when tool result count exceeds threshold

### Compaction Modes

| Mode | Trigger | What's Summarized | Cache Impact |
|------|---------|-------------------|-------------|
| **Full** | Auto or manual | All messages | Cache invalidated (new system prefix) |
| **Partial (from)** | Message selector | Messages after pivot | Cache preserved for kept prefix |
| **Partial (up_to)** | Message selector | Messages before pivot | Cache invalidated (summary precedes kept) |
| **Session Memory** | Auto (SM-first) | Messages before lastSummarizedId | Cache invalidated |
| **Time-based MC** | Idle gap (60 min) | Old tool result content | Cache invalidated (content changed) |
| **Cached MC** | Tool count threshold | Tool results via cache edits | Cache preserved (API-layer edits) |
| **API Context Mgmt** | Server-side token limit | Thinking blocks, tool uses | Server-managed |

### Post-Compact Cleanup

`runPostCompactCleanup()` invalidates all state that depends on pre-compaction context:

| Cleanup | Condition | Purpose |
|---------|-----------|---------|
| `resetMicrocompactState()` | Always | Clears stale tool registration |
| `resetContextCollapse()` | Main thread only | Resets collapse tracking |
| `getUserContext.cache.clear()` | Main thread only | Forces CLAUDE.md reload |
| `resetGetMemoryFilesCache()` | Main thread only | Forces memory file re-read |
| `clearSystemPromptSections()` | Always | Forces system prompt rebuild |
| `clearClassifierApprovals()` | Always | Clears permission classifier cache |
| `clearSpeculativeChecks()` | Always | Clears speculative permission checks |
| `clearBetaTracingState()` | Always | Resets beta tracing |
| `sweepFileContentCache()` | Attribution enabled | Clears attribution file cache |
| `clearSessionMessagesCache()` | Always | Clears session message cache |

**Intentionally NOT cleared**: `sentSkillNames` — re-injecting full skill_listing (~4K tokens) post-compact is pure cache_creation waste. The model still has SkillTool in its schema, and invoked_skills attachment preserves used skill content.

### Boundary Messages

Each compaction creates a `SystemCompactBoundaryMessage` with metadata:

```typescript
{
  type: 'system',
  system: 'compact_boundary',
  compactMetadata: {
    compactType: 'auto' | 'manual',
    preCompactTokenCount: number,
    lastPreCompactUuid: UUID,
    preCompactDiscoveredTools: string[],  // Tools discovered before compaction
    preservedSegment?: {                  // For partial compaction
      headUuid: UUID,
      anchorUuid: UUID,
      tailUuid: UUID,
    },
  },
}
```

The boundary marker enables the message loader to reconstruct the message chain across compaction discontinuities, including relinking preserved segments.

### Error Handling

| Error | Handling |
|-------|----------|
| Not enough messages | Throws `ERROR_MESSAGE_NOT_ENOUGH_MESSAGES` |
| Prompt too long | Retries with truncation (max 3), then throws |
| API error | Throws with API error prefix |
| User abort (ESC) | Throws `ERROR_MESSAGE_USER_ABORT`, no notification |
| Incomplete response | Throws `ERROR_MESSAGE_INCOMPLETE_RESPONSE` |
| No summary text | Throws "Failed to generate conversation summary" |
| Auto-compact failure | Logged, retried next turn, no user notification |
| Manual compact failure | Error notification shown to user |
| Session memory error | Falls back to legacy compact |

### Configuration

#### Environment Variables

| Variable | Effect |
|----------|--------|
| `DISABLE_COMPACT` | Disables all compaction |
| `DISABLE_AUTO_COMPACT` | Disables only auto-compact (manual still works) |
| `CLAUDE_CODE_AUTO_COMPACT_WINDOW` | Override context window for auto-compact calculation |
| `CLAUDE_AUTOCOMPACT_PCT_OVERRIDE` | Override auto-compact threshold as percentage |
| `CLAUDE_CODE_BLOCKING_LIMIT_OVERRIDE` | Override blocking token limit |
| `ENABLE_CLAUDE_CODE_SM_COMPACT` | Force-enable session memory compaction |
| `DISABLE_CLAUDE_CODE_SM_COMPACT` | Force-disable session memory compaction |
| `USE_API_CLEAR_TOOL_RESULTS` | Enable server-side tool result clearing (ant-only) |
| `USE_API_CLEAR_TOOL_USES` | Enable server-side tool use clearing (ant-only) |
| `API_MAX_INPUT_TOKENS` | Override trigger threshold for API context management |
| `API_TARGET_INPUT_TOKENS` | Override target token count for API context management |

#### Token Constants

| Constant | Value | Purpose |
|----------|-------|---------|
| `MAX_OUTPUT_TOKENS_FOR_SUMMARY` | 20,000 | Reserved for compaction output |
| `AUTOCOMPACT_BUFFER_TOKENS` | 13,000 | Buffer below context window for auto-compact |
| `WARNING_THRESHOLD_BUFFER_TOKENS` | 20,000 | Warning threshold buffer |
| `ERROR_THRESHOLD_BUFFER_TOKENS` | 20,000 | Error threshold buffer |
| `MANUAL_COMPACT_BUFFER_TOKENS` | 3,000 | Blocking limit buffer |
| `POST_COMPACT_MAX_FILES_TO_RESTORE` | 5 | Max files re-injected post-compact |
| `POST_COMPACT_TOKEN_BUDGET` | 50,000 | Total file restoration budget |
| `POST_COMPACT_MAX_TOKENS_PER_FILE` | 5,000 | Per-file token limit |
| `POST_COMPACT_MAX_TOKENS_PER_SKILL` | 5,000 | Per-skill token limit |
| `POST_COMPACT_SKILLS_TOKEN_BUDGET` | 25,000 | Total skill restoration budget |
| `MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES` | 3 | Circuit breaker threshold |
| `MAX_PTL_RETRIES` | 3 | Prompt-too-long retry limit |
| `MAX_COMPACT_STREAMING_RETRIES` | 2 | Streaming retry limit (when enabled) |
| `IMAGE_MAX_TOKEN_SIZE` | 2,000 | Estimated tokens per image/document |

#### Session Memory Compact Config

| Field | Default | Description |
|-------|---------|-------------|
| `minTokens` | 10,000 | Minimum tokens to preserve after compaction |
| `minTextBlockMessages` | 5 | Minimum text-block messages to keep |
| `maxTokens` | 40,000 | Maximum tokens to preserve (hard cap) |

#### Time-Based Microcompact Config

| Field | Default | Description |
|-------|---------|-------------|
| `enabled` | false | Master switch |
| `gapThresholdMinutes` | 60 | Trigger when idle gap exceeds this |
| `keepRecent` | 5 | Keep this many most-recent tool results |

### Data Flow

```
User Query / API Call
    │
    ▼
microcompactMessages() — per-turn pruning
    ├── Time-based trigger (idle gap > threshold)
    │   └── Content-clear old tool results
    └── Cached MC trigger (tool count > threshold)
        └── Queue cache_edits for API layer
    │
    ▼
shouldAutoCompact() — check token threshold
    │
    ├── Below threshold → proceed normally
    │
    └── Above threshold → autoCompactIfNeeded()
        │
        ├── trySessionMemoryCompaction()
        │   ├── Session memory exists and not empty?
        │   │   ├── Calculate messages to keep
        │   │   ├── Create boundary + summary from SM
        │   │   └── Return CompactionResult
        │   └── Fall through to full compact
        │
        └── compactConversation()
            ├── PreCompact hooks
            ├── Stream summary (forked agent or direct)
            │   ├── PTL retry: truncate oldest groups
            │   └── Streaming retry (if enabled)
            ├── Clear file state
            ├── Generate post-compact attachments
            │   ├── Recent files (budget-constrained)
            │   ├── Async agents
            │   ├── Plan file
            │   ├── Plan mode
            │   ├── Invoked skills
            │   ├── Deferred tools delta
            │   ├── Agent listing delta
            │   └── MCP instructions delta
            ├── SessionStart hooks
            ├── Create boundary marker
            ├── PostCompact hooks
            └── Return CompactionResult
    │
    ▼
runPostCompactCleanup()
    ├── Reset microcompact state
    ├── Clear context collapse
    ├── Clear user context cache
    ├── Clear system prompt sections
    ├── Clear classifier approvals
    └── Clear session messages cache
    │
    ▼
REPL replaces messages with post-compact result
```

## Notes

- `stripImagesFromMessages` handles images nested inside `tool_result` content arrays, not just top-level message content
- The forked agent path for cache sharing does NOT set `maxOutputTokens` — doing so would clamp `budget_tokens` and create a thinking config mismatch that invalidates the cache
- `partialCompactConversation` with direction `'up_to'` must strip old compact boundaries from kept messages; `'from'` keeps them because backward scan still works
- `adjustIndexToPreserveAPIInvariants` handles two scenarios: tool_use/tool_result pair preservation AND thinking blocks that share message.id with kept assistant messages (streaming yields separate messages per content block)
- Session memory compaction tracks `lastSummarizedMessageId` to know which messages have already been summarized; on resumed sessions this ID is missing and the system falls back to keeping all messages with SM as summary
- The `PTL_RETRY_MARKER` synthetic user message is prepended when truncation leaves an assistant-first sequence (API requires user-first); it is stripped on subsequent retries to prevent stalling
- Microcompact returns messages unchanged when no compaction is needed — callers should not assume mutation
- Cached MC is ant-only and requires the `CACHED_MICROCOMPACT` feature flag plus model support for cache editing
- API context management (`apiMicrocompact.ts`) is a server-side strategy that sends edit instructions to the API rather than modifying client-side messages
- The compact service integrates with the LSP diagnostic system — diagnostics are delivered as attachments post-compaction and the LSP `closeFile()` integration is marked as TODO
