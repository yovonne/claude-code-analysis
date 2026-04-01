# Schemas

## Purpose

Zod v4 schema definitions for validating hook configurations. Extracted to a separate file to break circular dependencies between `settings/types.ts` and `plugins/schemas.ts`.

## Location

`restored-src/src/schemas/hooks.ts`

## Key Exports

### Hook Command Schemas

Four discriminated hook command types, united by the `type` field:

#### `BashCommandHookSchema` (`type: 'command'`)
Executes a shell command when a hook event fires.

| Field | Type | Description |
|-------|------|-------------|
| `command` | `string` | Shell command to execute |
| `shell` | `'bash' \| 'powershell'` | Shell interpreter (defaults to bash) |
| `timeout` | `number` | Timeout in seconds |
| `statusMessage` | `string` | Custom spinner message while hook runs |
| `once` | `boolean` | Run once and remove after execution |
| `async` | `boolean` | Run in background without blocking |
| `asyncRewake` | `boolean` | Background run that wakes model on exit code 2 |

#### `PromptHookSchema` (`type: 'prompt'`)
Evaluates an LLM prompt when a hook event fires.

| Field | Type | Description |
|-------|------|-------------|
| `prompt` | `string` | LLM prompt (use `$ARGUMENTS` for hook input JSON) |
| `model` | `string` | Model to use (defaults to small fast model) |
| `timeout` | `number` | Timeout in seconds |
| `statusMessage` | `string` | Custom spinner message |
| `once` | `boolean` | Run once and remove |

#### `HttpHookSchema` (`type: 'http'`)
POSTs hook input JSON to a URL.

| Field | Type | Description |
|-------|------|-------------|
| `url` | `string` (URL) | URL to POST to |
| `headers` | `Record<string, string>` | Additional headers (supports `$VAR` interpolation) |
| `allowedEnvVars` | `string[]` | Whitelist of env vars for header interpolation |
| `timeout` | `number` | Timeout in seconds |
| `statusMessage` | `string` | Custom spinner message |
| `once` | `boolean` | Run once and remove |

#### `AgentHookSchema` (`type: 'agent'`)
Runs an agentic verifier with a verification prompt.

| Field | Type | Description |
|-------|------|-------------|
| `prompt` | `string` | Verification prompt (e.g., "Verify that unit tests ran") |
| `model` | `string` | Model to use (defaults to Haiku) |
| `timeout` | `number` | Timeout in seconds (default 60) |
| `statusMessage` | `string` | Custom spinner message |
| `once` | `boolean` | Run once and remove |

### Shared Fields

#### `IfConditionSchema`
Optional `if` field on all hook types — permission rule syntax to filter when hooks run (e.g., `"Bash(git *)"`). Uses lazy schema to avoid circular imports.

### Composite Schemas

- `HookCommandSchema`: Discriminated union of all four hook command types
- `HookMatcherSchema`: `{ matcher?: string, hooks: HookCommand[] }` — matches events and runs associated hooks
- `HooksSchema`: `Partial<Record<HookEvent, HookMatcher[]>>` — top-level hooks configuration keyed by event

### Inferred Types

- `HookCommand`: Union of all hook command types
- `BashCommandHook`, `PromptHook`, `AgentHook`, `HttpHook`: Extracted variants
- `HookMatcher`: Matcher configuration type
- `HooksSettings`: `Partial<Record<HookEvent, HookMatcher[]>>` — runtime settings type

## Dependencies

### Internal Dependencies

- `entrypoints/agentSdkTypes.js` — `HOOK_EVENTS`, `HookEvent` enum
- `utils/lazySchema.js` — Lazy schema evaluation to break circular imports
- `utils/shell/shellProvider.js` — `SHELL_TYPES` for shell enum

### External Dependencies

- `zod/v4` — Schema validation

## Implementation Details

### Lazy Schema Pattern

All schemas use `lazySchema(() => ...)` to defer evaluation. This breaks circular dependencies because:
1. `settings/types.ts` imports from `schemas/hooks.ts`
2. `plugins/schemas.ts` imports from `schemas/hooks.ts`
3. Without lazy evaluation, these would form a cycle at module load time

### AgentHook Transform Warning

The `AgentHookSchema.prompt` field deliberately does NOT use `.transform()` to wrap the string in a function. A previous transform wrapped the string in `(_msgs) => prompt` for a programmatic construction use case that has since been refactored. The transform caused `updateSettingsForSource` to silently drop the user's prompt from `settings.json` during JSON round-trip (gh-24920, CC-79).

### Permission Rule Syntax

The `if` condition uses permission rule syntax (e.g., `"Bash(git *)"`, `"Read(*.ts)"`) to filter hooks before spawning. This avoids running hooks for non-matching commands.

## Related Modules

- [Hooks System](./hooks-system.md) — Hooks are validated against these schemas
- [Types](./types-system.md) — Hook types in `types/hooks.ts` align with these schemas
- [Vendor Modules](./vendor-modules.md) — Plugin schemas reference hook schemas
