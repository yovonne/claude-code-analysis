# ToolSearchTool

## Purpose

ToolSearchTool is the discovery mechanism for deferred tools in Claude Code. When tools are deferred (not loaded at startup to save context window space), the model only sees their names. ToolSearchTool allows the model to query for and fetch the full schema definitions of these deferred tools so they can be invoked. It supports both direct selection (`select:ToolName`) and keyword-based search with scoring across tool names, search hints, and descriptions. The tool returns matched tools as `tool_reference` blocks that the SDK can expand into full schemas.

## Location

- `restored-src/src/tools/ToolSearchTool/ToolSearchTool.ts` — Main tool definition and search logic (472 lines)
- `restored-src/src/tools/ToolSearchTool/prompt.ts` — Tool prompt and deferred tool classification (122 lines)
- `restored-src/src/tools/ToolSearchTool/constants.ts` — Tool name constant (2 lines)

## Key Exports

### From ToolSearchTool.ts

| Export | Description |
|--------|-------------|
| `ToolSearchTool` | The complete tool definition built via `buildTool()` |
| `inputSchema` | Zod schema for tool input |
| `outputSchema` | Zod schema for tool output |
| `Output` | Zod-inferred output type: `{ matches, query, total_deferred_tools, pending_mcp_servers? }` |
| `clearToolSearchDescriptionCache()` | Clears the memoized tool description cache |

### From prompt.ts

| Export | Description |
|--------|-------------|
| `TOOL_SEARCH_TOOL_NAME` | Constant: `'ToolSearch'` |
| `isDeferredTool(tool)` | Determines if a tool should be deferred (requires ToolSearch to load) |
| `formatDeferredToolLine(tool)` | Formats a deferred tool name for the `<available-deferred-tools>` message |
| `getPrompt()` | Returns the complete tool prompt |

## Input/Output Schemas

### Input Schema

```typescript
{
  query: string,       // Required: Query to find deferred tools
  max_results?: number, // Optional: Maximum results to return (default: 5)
}
```

Query formats:
- `select:Read,Edit,Grep` — Direct selection of exact tool names (comma-separated)
- `notebook jupyter` — Keyword search, returns up to `max_results` best matches
- `+slack send` — Require "slack" in the name, rank by remaining terms

### Output Schema

```typescript
{
  matches: string[],                // Matched tool names
  query: string,                    // The original query
  total_deferred_tools: number,     // Total number of deferred tools available
  pending_mcp_servers?: string[],   // MCP servers still connecting (when no matches)
}
```

## Deferred Tool Concept

### What Are Deferred Tools?

Deferred tools are tools whose full schema definitions are NOT included in the initial system prompt. Only their names appear in an `<available-deferred-tools>` block. This saves context window space when many tools (especially MCP tools) are available.

### Deferred Tool Classification

A tool is deferred if any of these conditions are true:

| Condition | Result |
|-----------|--------|
| `tool.alwaysLoad === true` | **NEVER deferred** (explicit opt-out via `_meta['anthropic/alwaysLoad']`) |
| `tool.isMcp === true` | **ALWAYS deferred** (MCP tools are workflow-specific) |
| `tool.name === 'ToolSearch'` | **NEVER deferred** (needed to load everything else) |
| `tool.name === 'Agent'` + Fork Subagent enabled | **NEVER deferred** (must be available turn 1) |
| `tool.name === 'BriefTool'` + Kairos enabled | **NEVER deferred** (primary communication channel) |
| `tool.name === 'SendUserFile'` + Kairos + Bridge active | **NEVER deferred** (file-delivery channel) |
| `tool.shouldDefer === true` | **Deferred** |

### Tool Location Hint

The prompt tells the model where deferred tool names appear:

| Mode | Message |
|------|---------|
| Delta enabled (A/B test or ant) | "Deferred tools appear by name in `<system-reminder>` messages." |
| Delta disabled (legacy) | "Deferred tools appear by name in `<available-deferred-tools>` messages." |

## Search Execution Pipeline

### Execution Flow

```
1. INPUT RECEIVED
   - Model returns ToolSearch tool_use with { query, max_results? }

2. DEFERRED TOOL COLLECTION
   - Filter all tools by isDeferredTool()
   - Invalidate description cache if deferred tools changed

3. PENDING MCP SERVER CHECK
   - Check AppState.mcp.clients for 'pending' type
   - Include pending server names in results when no matches found

4. QUERY PARSING
   a. Check for select: prefix
   b. If select: parse comma-separated tool names
   c. Otherwise: keyword search

5. SELECT MODE (select:ToolName)
   a. Split by comma, trim each name
   b. Look up each name in deferred tools, then all tools
   c. Collect found names and missing names
   d. If none found → return empty matches
   e. Return found tool names as matches

6. KEYWORD SEARCH MODE
   a. Check exact name match (deferred first, then all tools)
   b. Check MCP prefix match (mcp__server)
   c. Parse query into terms (required +optional)
   d. Pre-filter by required terms
   e. Score remaining tools
   f. Sort by score, take top max_results

7. RESULT FORMATTING
   - Return matches as tool_reference blocks
   - Include pending MCP servers when no matches
   - Log analytics event
```

### Select Mode

Direct tool selection via `select:` prefix:

```
select:Read          → Single tool
select:Read,Edit     → Multiple tools
select:mcp__github__list_issues  → MCP tool
```

- Comma-separated names are supported for multi-select
- Names are looked up in deferred tools first, then all tools
- If a name isn't deferred but IS in the full tool set, it's still returned (harmless no-op)
- Missing names are logged but don't prevent returning found tools

### Keyword Search

Keyword search uses a weighted scoring system:

| Match Type | MCP Score | Regular Score |
|------------|-----------|---------------|
| Exact part match | 12 | 10 |
| Part contains term | 6 | 5 |
| Full name contains term (fallback) | 3 | 3 |
| searchHint match | 4 | 4 |
| Description match (word boundary) | 2 | 2 |

#### Term Parsing

Query terms are parsed into required and optional:

| Syntax | Type |
|--------|------|
| `+term` | Required (must match) |
| `term` | Optional (ranks higher if matches) |

#### Tool Name Parsing

Tool names are split into searchable parts:

| Name Format | Parsing |
|-------------|---------|
| `mcp__server__action` | Split by `__` and `_`: `["server", "action"]` |
| `CamelCaseTool` | Split by camel case: `["camel", "case", "tool"]` |
| `snake_case_tool` | Split by `_`: `["snake", "case", "tool"]` |

#### Required Term Pre-Filtering

When required terms (`+term`) are present, tools are pre-filtered to only those matching ALL required terms in:
- Parsed name parts
- Part substring matches
- Description (word boundary regex)
- searchHint (word boundary regex)

This avoids scoring tools that can't possibly match.

### Caching

Tool descriptions are memoized by tool name to avoid repeated expensive `tool.prompt()` calls:

```typescript
const getToolDescriptionMemoized = memoize(
  async (toolName, tools) => tool.prompt(...),
  (toolName) => toolName  // Cache key
)
```

Cache invalidation:
- `maybeInvalidateCache()` checks if the deferred tool set has changed
- If changed, clears the memoization cache and updates the cache key
- `clearToolSearchDescriptionCache()` provides manual cache clearing

## Result Formatting

### Tool Reference Blocks

When matches are found, the result is returned as `tool_reference` blocks:

```json
{
  "type": "tool_result",
  "tool_use_id": "<id>",
  "content": [
    { "type": "tool_reference", "tool_name": "Read" },
    { "type": "tool_reference", "tool_name": "Edit" }
  ]
}
```

The SDK expands these references into full tool schemas.

### No Matches

When no matches are found:

```
No matching deferred tools found. Some MCP servers are still connecting:
server1, server2. Their tools will become available shortly — try searching again.
```

## Analytics

Each search logs a `tengu_tool_search_outcome` event:

| Field | Description |
|-------|-------------|
| `query` | The search query |
| `queryType` | `'select'` or `'keyword'` |
| `matchCount` | Number of matches returned |
| `totalDeferredTools` | Total deferred tools available |
| `maxResults` | Requested max results |
| `hasMatches` | Whether any matches were found |

## Permission Handling

ToolSearchTool does not define an explicit `checkPermissions` method. It inherits the default permission behavior from the tool framework. Since it's a read-only discovery tool, it effectively operates without permission barriers.

## Tool Availability

| Condition | Effect |
|-----------|--------|
| `isToolSearchEnabledOptimistic()` returns false | Tool is disabled |

## Configuration

### Limits

| Limit | Value | Description |
|-------|-------|-------------|
| `max_results` default | 5 | Default maximum results |
| `maxResultSizeChars` | 100,000 | Maximum result size stored in context |

### Feature Flags

| Flag | Purpose |
|------|---------|
| `KAIROS` / `KAIROS_BRIEF` | Affects Brief tool deferral |
| `KAIROS` | Affects SendUserFile tool deferral |
| `FORK_SUBAGENT` | Affects Agent tool deferral |
| `tengu_glacier_2xr` | Controls deferred tool location hint (system-reminder vs available-deferred-tools) |

## Dependencies

### Internal

| Module | Purpose |
|--------|---------|
| `Tool.js` | Tool building, findToolByName |
| `utils/toolSearch.js` | Tool search enablement check |
| `utils/debug.js` | Debug logging |
| `utils/stringUtils.js` | RegExp escaping |
| `services/analytics/` | Event logging |
| `bootstrap/state.js` | ReplBridge status |
| `services/analytics/growthbook.js` | Feature flags |
| `tools/AgentTool/constants.js` | Agent tool name |

### External

| Package | Purpose |
|---------|---------|
| `@anthropic-ai/sdk` | ToolResultBlockParam type |
| `lodash-es/memoize.js` | Memoization for tool descriptions |
| `zod/v4` | Input/output schema validation |

## Data Flow

```
Model Request
    |
    v
ToolSearchTool Input { query, max_results? }
    |
    v
┌─────────────────────────────────────────────────────────────────┐
│ DEFERRED TOOL COLLECTION                                        │
│  - Filter tools by isDeferredTool()                             │
│  - Invalidate cache if deferred tools changed                   │
└─────────────────────────────────────────────────────────────────┘
    |
    v
┌─────────────────────────────────────────────────────────────────┐
│ QUERY PARSING                                                   │
│                                                                 │
│  select: prefix?                                                │
│    → Split by comma, look up each name                          │
│    → Return found names                                         │
│                                                                 │
│  Otherwise: keyword search                                      │
│    → Parse terms (+required, optional)                          │
│    → Pre-filter by required terms                               │
│    → Score tools (name parts, hint, description)                │
│    → Sort by score, take top max_results                        │
└─────────────────────────────────────────────────────────────────┘
    |
    v
┌─────────────────────────────────────────────────────────────────┐
│ RESULT FORMATTING                                               │
│                                                                 │
│  Matches found:                                                 │
│    → Return tool_reference blocks                               │
│                                                                 │
│  No matches:                                                    │
│    → Include pending MCP server info                            │
│    → Return text message                                        │
│                                                                 │
│  Log analytics event                                            │
└─────────────────────────────────────────────────────────────────┘
    |
    v
Output { matches[], query, total_deferred_tools, pending_mcp_servers? }
    |
    v
Model Response (tool_result with tool_reference blocks)
```
