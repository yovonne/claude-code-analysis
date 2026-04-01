# WebSearchTool

## Purpose

WebSearchTool enables Claude to perform live web searches and use the results to inform responses. It interfaces with Anthropic's native web search API (`web_search_20250305`), providing up-to-date information for current events and recent data. The tool supports domain filtering (allow/block lists), streams real-time progress updates during search execution, and mandates source citation in responses. It is only available for first-party API, Vertex AI (Claude 4.0+), and Foundry providers.

## Location

- `restored-src/src/tools/WebSearchTool/WebSearchTool.ts` — Main tool definition and search execution logic (436 lines)
- `restored-src/src/tools/WebSearchTool/prompt.ts` — Tool prompt with source citation requirements (35 lines)
- `restored-src/src/tools/WebSearchTool/UI.tsx` — Terminal UI rendering with progress display (101 lines)

## Key Exports

### From WebSearchTool.ts

| Export | Description |
|--------|-------------|
| `WebSearchTool` | The complete tool definition built via `buildTool()` |
| `Input` | Zod-inferred input type: `{ query, allowed_domains?, blocked_domains? }` |
| `Output` | Zod-inferred output type: `{ query, results, durationSeconds }` |
| `SearchResult` | Type: `{ tool_use_id, content: { title, url }[] }` |
| `WebSearchProgress` | Progress data type for streaming updates (re-exported from types/tools.js) |

## Input/Output Schemas

### Input Schema

```typescript
{
  query: string,            // Required: The search query to use (min 2 chars)
  allowed_domains?: string[],  // Optional: Only include results from these domains
  blocked_domains?: string[],  // Optional: Never include results from these domains
}
```

### Output Schema

```typescript
{
  query: string,                    // The search query that was executed
  results: (SearchResult | string)[], // Search results and/or text commentary
  durationSeconds: number,          // Time taken to complete the search
}
```

## Search Execution Pipeline

### Execution Flow

```
1. INPUT RECEIVED
   - Model returns WebSearch tool_use with { query, allowed_domains?, blocked_domains? }

2. VALIDATION
   - validateInput():
     a. Query must not be empty
     b. Cannot specify both allowed_domains AND blocked_domains simultaneously

3. PERMISSION CHECK
   - checkPermissions(): Always returns 'passthrough' (delegated to API layer)
   - Suggests adding allow rule to local settings

4. SEARCH EXECUTION (call)
   a. Create user message with search query
   b. Build tool schema (BetaWebSearchTool20250305)
   c. Select model (Haiku or main model based on feature flag)
   d. Open streaming query with web_search tool
   e. Process streaming events:
      - Track tool use IDs
      - Accumulate partial JSON for progress updates
      - Emit query_update progress events
      - Emit search_results_received progress events
   f. Collect all content blocks

5. RESULT PROCESSING
   a. Parse content blocks into structured output
   b. Extract text summaries and search hits
   c. Calculate duration
   d. Return formatted output
```

### Tool Schema Construction

The tool schema sent to the API:

```typescript
{
  type: 'web_search_20250305',
  name: 'web_search',
  allowed_domains: input.allowed_domains,  // Optional domain allowlist
  blocked_domains: input.blocked_domains,  // Optional domain blocklist
  max_uses: 8,                              // Hardcoded maximum searches per session
}
```

### Model Selection

| Feature Flag | Model Used | Thinking |
|--------------|-----------|----------|
| `tengu_plum_vx3` enabled | `getSmallFastModel()` (Haiku) | Disabled |
| `tengu_plum_vx3` disabled | `context.options.mainLoopModel` | Per context config |

### Streaming Event Processing

The tool processes four types of streaming events:

| Event Type | Handling |
|------------|----------|
| `assistant` message | Accumulates content blocks into final result |
| `content_block_start` (server_tool_use) | Tracks tool use ID for query extraction |
| `content_block_delta` (input_json_delta) | Accumulates partial JSON, extracts query for progress updates |
| `content_block_start` (web_search_tool_result) | Emits `search_results_received` progress with result count |

### Progress Updates

Two progress event types are emitted via the `onProgress` callback:

| Type | Data |
|------|------|
| `query_update` | `{ query }` — When a new search query is detected |
| `search_results_received` | `{ resultCount, query }` — When search results arrive |

## Result Parsing

### Content Block Structure

The API returns a sequence of blocks:

```
- text (introductory text, always present)
[
  - server_tool_use
  - web_search_tool_result
  - text and citation blocks intermingled
]+  (repeated for each search)
```

### Parsing Logic

`makeOutputFromSearchResponse()` processes the blocks:

1. Accumulates text blocks into `textAcc`
2. On `server_tool_use`: flushes accumulated text to results, resets
3. On `web_search_tool_result`: extracts hits (title + url) and adds to results
4. On `text`: appends to accumulator (in current text phase) or starts new text phase
5. Final accumulated text is flushed to results

Error handling: If `web_search_tool_result.content` is not an array (error case), an error message string is pushed to results instead.

## Provider Availability

| Provider | Enabled | Notes |
|----------|---------|-------|
| `firstParty` | Always | Anthropic's direct API |
| `vertex` | Claude 4.0+ only | Requires `claude-opus-4`, `claude-sonnet-4`, or `claude-haiku-4` |
| `foundry` | Always | Models already support web search |
| Other | Disabled | — |

## Prompt Requirements

The tool prompt enforces mandatory source citation:

```
CRITICAL REQUIREMENT - You MUST follow this:
  - After answering the user's question, you MUST include a "Sources:" section
  - List all relevant URLs as markdown hyperlinks: [Title](URL)
  - This is MANDATORY - never skip including sources
```

The prompt also includes the current month/year for accurate search queries and notes that web search is only available in the US.

## Tool Result Formatting

`mapToolResultToToolResultBlockParam()` formats the output for the model:

```
Web search results for query: "<query>"

<text summaries>
Links: [{"title": "...", "url": "..."}]

REMINDER: You MUST include the sources above in your response to the user using markdown hyperlinks.
```

Null/undefined entries in the results array are silently skipped (guards against JSON round-tripping issues from compaction or transcript deserialization).

## Configuration

### Feature Flags

| Flag | Purpose |
|------|---------|
| `tengu_plum_vx3` | When enabled, uses Haiku model with disabled thinking for searches |

### Limits

| Limit | Value | Description |
|-------|-------|-------------|
| `max_uses` | 8 | Maximum web searches per session (hardcoded) |
| `maxResultSizeChars` | 100,000 | Maximum result size stored in context |
| Query min length | 2 characters | Minimum query length |

## Dependencies

### Internal

| Module | Purpose |
|--------|---------|
| `services/api/claude.js` | Model query streaming |
| `services/analytics/growthbook.js` | Feature flag access |
| `utils/model/model.js` | Model selection utilities |
| `utils/slowOperations.js` | JSON parse/stringify |
| `utils/messages.js` | Message creation |
| `utils/systemPromptType.js` | System prompt wrapper |
| `constants/common.js` | Month/year formatting |

### External

| Package | Purpose |
|---------|---------|
| `@anthropic-ai/sdk` | Web search tool types |
| `zod/v4` | Input/output schema validation |
| `react` | UI rendering |

## Data Flow

```
Model Request
    |
    v
WebSearchTool Input { query, allowed_domains?, blocked_domains? }
    |
    v
┌─────────────────────────────────────────────────────────────────┐
│ VALIDATION                                                      │
│ - Query not empty                                               │
│ - Cannot specify both allowed_domains and blocked_domains       │
└─────────────────────────────────────────────────────────────────┘
    |
    v
┌─────────────────────────────────────────────────────────────────┐
│ PERMISSION CHECK                                                │
│  - Always passthrough (delegated to API layer)                  │
└─────────────────────────────────────────────────────────────────┘
    |
    v
┌─────────────────────────────────────────────────────────────────┐
│ SEARCH EXECUTION (streaming)                                    │
│                                                                 │
│  1. Build web_search tool schema                                │
│  2. Select model (Haiku or main)                                │
│  3. Open streaming query                                      │
│  4. Process events:                                             │
│     - Track tool use IDs                                       │
│     - Extract queries from partial JSON                        │
│     - Emit progress updates                                    │
│     - Collect content blocks                                   │
│  5. Parse results into SearchResult | string[]                  │
└─────────────────────────────────────────────────────────────────┘
    |
    v
Output { query, results[], durationSeconds }
    |
    v
Model Response (tool_result with formatted links + source reminder)
```
