# SyntheticOutputTool

## Purpose

SyntheticOutputTool (also known as `StructuredOutput`) enables Claude to return final responses as structured JSON that conforms to a user-provided JSON schema. It is used exclusively in non-interactive sessions (SDK/CLI workflows) where the calling code needs machine-parseable output rather than free-form text. The tool is dynamically created with a custom JSON schema per invocation, validated using Ajv, and cached by schema identity to avoid repeated compilation overhead.

## Location

- `restored-src/src/tools/SyntheticOutputTool/SyntheticOutputTool.ts` — Tool definition, schema validation, and factory (164 lines)

## Key Exports

| Export | Description |
|--------|-------------|
| `SyntheticOutputTool` | The base tool definition (without custom schema) |
| `SYNTHETIC_OUTPUT_TOOL_NAME` | Constant: `'StructuredOutput'` |
| `createSyntheticOutputTool(jsonSchema)` | Factory function: creates a tool instance configured with the given JSON schema |
| `isSyntheticOutputToolEnabled(opts)` | Checks if the tool should be enabled (requires `isNonInteractiveSession`) |
| `Output` | Type: `string` (the success message) |

## Input/Output Schemas

### Input Schema

The base tool accepts any object (`z.object({}).passthrough()`). When created via `createSyntheticOutputTool()`, the actual input schema is the user-provided JSON schema, validated by Ajv at creation time and enforced at runtime.

### Output Schema

```typescript
string  // "Structured output provided successfully"
```

The actual structured data is returned in the `structured_output` field of the call result (not part of the Zod output schema).

## Tool Creation

### Factory Function

```typescript
createSyntheticOutputTool(jsonSchema: Record<string, unknown>): CreateResult
```

Returns either:
- `{ tool: Tool<InputSchema> }` — Successfully created tool instance
- `{ error: string }` — Ajv validation error message for invalid schemas

### Creation Pipeline

```
1. CACHE CHECK
   - Look up jsonSchema in WeakMap cache (by object identity)
   - If cached → return cached result

2. SCHEMA VALIDATION (Ajv)
   - Create Ajv instance with allErrors: true
   - Validate schema with ajv.validateSchema()
   - If invalid → return { error: ajv.errorsText() }

3. SCHEMA COMPILATION
   - Compile schema with ajv.compile()
   - Returns a validate function

4. TOOL CONSTRUCTION
   - Clone base SyntheticOutputTool
   - Override inputJSONSchema with user schema
   - Override call() to validate input against compiled schema
   - Throw TelemetrySafeError on schema mismatch

5. CACHE STORAGE
   - Store result in WeakMap (keyed by schema identity)
   - Return { tool }
```

### Schema Caching

A `WeakMap<object, CreateResult>` caches tool instances by schema object identity:

```typescript
const toolCache = new WeakMap<object, CreateResult>()
```

This is critical for performance because workflow scripts call `agent({schema: SCHEMA})` 30-80 times per run with the same schema object reference. Without caching, each call does `new Ajv() + validateSchema() + compile()` (~1.4ms of JIT codegen). The identity cache reduces 80-call workflow Ajv overhead from ~110ms to ~4ms.

## Runtime Validation

When the model calls the tool, the input is validated against the compiled schema:

```typescript
async call(input) {
  const isValid = validateSchema(input)
  if (!isValid) {
    const errors = validateSchema.errors
      ?.map(e => `${e.instancePath || 'root'}: ${e.message}`)
      .join(', ')
    throw new TelemetrySafeError(
      `Output does not match required schema: ${errors}`,
      `StructuredOutput schema mismatch: ${(errors ?? '').slice(0, 150)}`,
    )
  }
  return {
    data: 'Structured output provided successfully',
    structured_output: input,
  }
}
```

Error messages include:
- Instance path (e.g., `/items/0/name`) for precise location
- Ajv error message (e.g., "must be string")
- Telemetry-safe version (errors truncated to 150 chars, no code/file paths)

## Tool Enablement

The tool is only created when `isSyntheticOutputToolEnabled()` returns true:

```typescript
isSyntheticOutputToolEnabled(opts: { isNonInteractiveSession: boolean }): boolean
  → opts.isNonInteractiveSession
```

Once created, the tool is always enabled (`isEnabled() → true`). It is gated at creation time rather than per-call.

## Permission Handling

| Operation | Permission Decision |
|-----------|-------------------|
| All operations | Auto-allow (always allowed) |

The tool is considered safe because it only validates and returns data — it has no side effects.

## UI Rendering

Minimal UI implementations since this tool is for non-interactive use:

| Method | Output |
|--------|--------|
| `renderToolUseMessage(input)` | Shows up to 3 field key-value pairs, or `"N fields: key1, key2, key3…"` |
| `renderToolUseRejectedMessage()` | `"Structured output rejected"` |
| `renderToolUseErrorMessage()` | `"Structured output error"` |
| `renderToolUseProgressMessage()` | `null` |
| `renderToolResultMessage(output)` | Returns the output string directly |

## Tool Result Formatting

`mapToolResultToToolResultBlockParam()` returns a standard tool_result block:

```json
{
  "tool_use_id": "<id>",
  "type": "tool_result",
  "content": "Structured output provided successfully"
}
```

The actual structured data is available in the `structured_output` field of the call result, which the SDK extracts separately.

## Prompt

The tool prompt instructs the model:

```
Use this tool to return your final response in the requested structured format.
You MUST call this tool exactly once at the end of your response to provide the structured output.
```

## Configuration

### Limits

| Limit | Value | Description |
|-------|-------|-------------|
| `maxResultSizeChars` | 100,000 | Maximum result size stored in context |

### Tool Properties

| Property | Value | Description |
|----------|-------|-------------|
| `isMcp` | false | Not an MCP tool |
| `isReadOnly` | true | Does not modify state |
| `isConcurrencySafe` | true | Safe to call concurrently |
| `isOpenWorld` | false | Closed-world tool |
| `shouldDefer` | Not set | Loaded at startup |

## Dependencies

### Internal

| Module | Purpose |
|--------|---------|
| `Tool.js` | Tool building and types |
| `utils/errors.js` | Telemetry-safe error class |
| `utils/slowOperations.js` | JSON stringification |
| `utils/lazySchema.js` | Lazy schema loading |

### External

| Package | Purpose |
|---------|---------|
| `ajv` | JSON Schema validation |
| `zod/v4` | Base input/output schema |

## Data Flow

```
Model Request
    |
    v
SyntheticOutputTool Input { ...user schema fields... }
    |
    v
┌─────────────────────────────────────────────────────────────────┐
│ PERMISSION CHECK                                                │
│  - Auto-allow (always allowed)                                  │
└─────────────────────────────────────────────────────────────────┘
    |
    v
┌─────────────────────────────────────────────────────────────────┐
│ SCHEMA VALIDATION                                               │
│                                                                 │
│  1. Validate input against compiled Ajv schema                  │
│  2. If invalid:                                                 │
│     - Collect error messages with instance paths                │
│     - Throw TelemetrySafeError                                  │
│  3. If valid:                                                   │
│     - Return { data: success message, structured_output: input }│
└─────────────────────────────────────────────────────────────────┘
    |
    v
Output { data: string, structured_output: object }
    |
    v
SDK extracts structured_output for programmatic use
```
