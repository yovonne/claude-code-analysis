# Tool Use Summary Service

## Purpose
Generates human-readable summaries of completed tool batches using Haiku (Claude's fast model). Used by the SDK to provide high-level progress updates to clients, particularly for mobile app displays where tool execution details need to be condensed into short labels.

## Location
`restored-src/src/services/toolUseSummary/`

### Key Files
| File | Description |
|------|-------------|
| `toolUseSummaryGenerator.ts` | Summary generation using Haiku API |

## Key Exports

### Functions
- `generateToolUseSummary(params)`: Generate a brief summary string from tool execution data

### Types
- `GenerateToolUseSummaryParams`: Input parameters
  - `tools`: Array of `{ name, input, output }` for each executed tool
  - `signal`: AbortSignal for cancellation
  - `isNonInteractiveSession`: Whether this is a non-interactive (SDK/programmatic) session
  - `lastAssistantText`: Optional last assistant message text for context
- `ToolInfo`: `{ name: string, input: unknown, output: unknown }`

## Dependencies

### Internal Dependencies
- `api/claude` — queryHaiku for summary generation
- `errors` — toError for error conversion
- `log` — logError for error logging
- `slowOperations` — jsonStringify for serialization
- `systemPromptType` — asSystemPrompt for prompt formatting
- `constants/errorIds` — Error ID for tracking

### External Dependencies
- None (uses internal API client)

## Implementation Details

### Summary Generation
1. Validate input — return null if no tools
2. Build concise tool summaries:
   - Serialize input and output to JSON (truncated to 300 chars each)
   - Format as `Tool: {name}\nInput: {input}\nOutput: {output}`
3. Optionally prefix with user intent from last assistant message (truncated to 200 chars)
4. Call Haiku API with:
   - System prompt: instructions for short past-tense summary labels
   - User prompt: tool summaries with optional context
   - Prompt caching enabled
   - No MCP tools, no agents, no append system prompt
5. Extract text content from response, trim whitespace
6. Return summary string or null if empty/failed

### System Prompt Design
The system prompt instructs the model to:
- Write a short summary label (git-commit-subject style)
- Truncate around 30 characters (for mobile display)
- Use past tense verb + most distinctive noun
- Drop articles, connectors, and long location context
- Examples: "Searched in auth/", "Fixed NPE in UserService", "Created signup endpoint"

### Input Truncation
- Tool input/output: 300 characters each
- Last assistant text: 200 characters
- Serialization failures: returns `[unable to serialize]`

## Data Flow

```
Tool batch completes
    ↓
generateToolUseSummary({ tools, signal, isNonInteractiveSession, lastAssistantText })
    ↓
Truncate tool input/output (300 chars each)
    ↓
Build tool summaries string
    ↓
Optionally prepend user intent (200 chars)
    ↓
queryHaiku() — fast model, cached prompt
    ↓
Extract text content, trim
    ↓
Return summary string or null
```

## Integration Points
- **SDK**: Called by SDK to generate progress updates for client apps
- **Mobile App**: Summaries displayed as single-line rows (~30 char truncation)
- **Non-Interactive Sessions**: `isNonInteractiveSession` flag passed through to API

## Configuration
- Model: Haiku (fast, cost-effective for summary generation)
- Prompt caching: Enabled (system prompt is static)
- No tool access: Summary generation doesn't need MCP tools or agents

## Error Handling
- **Non-critical**: Summaries are best-effort — failures logged but don't block execution
- **Error tracking**: Uses `E_TOOL_USE_SUMMARY_GENERATION_FAILED` error ID
- **Empty input**: Returns null for empty tool arrays
- **Serialization failures**: Gracefully handles unserializable values
- **API failures**: Catches errors, logs, returns null

## Testing
- No test-specific exports — pure function, easily testable with mocked queryHaiku

## Related Modules
- [Analytics](./analytics-service.md) — API query telemetry
- [Query Engine](../01-core-modules/query-engine.md) — LLM query infrastructure

## Notes
- Designed for mobile app display — summaries must be short and informative
- Uses Haiku specifically for speed and cost (summaries are non-critical)
- Prompt caching reduces cost since system prompt is constant
- The summary format mirrors git commit subjects: past tense verb + distinctive noun
- Last assistant text provides user intent context but is optional
