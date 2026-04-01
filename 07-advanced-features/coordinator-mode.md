# Coordinator Mode

## Purpose

Coordinator Mode transforms Claude Code into a multi-worker orchestrator. Instead of executing tasks directly, the coordinator delegates research, implementation, and verification to parallel async worker agents while synthesizing results and communicating with the user.

## Location

`restored-src/src/coordinator/coordinatorMode.ts`

## Key Exports

### Functions

- `isCoordinatorMode()`: Checks if coordinator mode is active via `bun:bundle` feature flag and `CLAUDE_CODE_COORDINATOR_MODE` env var.
- `matchSessionMode(sessionMode)`: Aligns the current runtime mode with a resumed session's stored mode. Flips the env var if mismatched and returns a warning message.
- `getCoordinatorUserContext(mcpClients, scratchpadDir)`: Builds a `workerToolsContext` string injected into the coordinator's user context, listing available worker tools and MCP servers.
- `getCoordinatorSystemPrompt()`: Returns the comprehensive system prompt that teaches the coordinator its role, tools, workflow, and prompt-writing best practices.

### Constants

- `INTERNAL_WORKER_TOOLS`: Set of tool names (`TeamCreateTool`, `TeamDeleteTool`, `SendMessageTool`, `SyntheticOutputTool`) excluded from worker tool listings — these are internal orchestration tools, not worker-facing.

## Dependencies

### Internal Dependencies

- `../constants/tools.js` — `ASYNC_AGENT_ALLOWED_TOOLS` for worker tool enumeration
- `../services/analytics/growthbook.js` — Statsig feature gate checks
- `../tools/*/` — Individual tool name constants (AgentTool, BashTool, etc.)
- `../utils/envUtils.js` — `isEnvTruthy()` for env var parsing

### External Dependencies

- `bun:bundle` — Bun's feature flag system for build-time toggles

## Implementation Details

### Core Logic

Coordinator mode is controlled by a two-layer gate:

1. **Build-time feature flag** — `feature('COORDINATOR_MODE')` from `bun:bundle`
2. **Runtime env var** — `CLAUDE_CODE_COORDINATOR_MODE` (checked via `isEnvTruthy`)

Both must be true for `isCoordinatorMode()` to return `true`. This dual-gate pattern allows the feature to be compiled in but disabled at runtime.

### Session Mode Matching

When resuming a session, `matchSessionMode()` compares the stored session mode (`'coordinator' | 'normal'`) against the current runtime state. If they differ, it flips `process.env.CLAUDE_CODE_COORDINATOR_MODE` to match the stored mode. This ensures a session resumed in coordinator mode actually operates as coordinator mode, regardless of how the new process was launched.

### Worker Tool Context

`getCoordinatorUserContext()` dynamically constructs a description of what tools spawned workers have access to. Two modes exist:

- **Simple mode** (`CLAUDE_CODE_SIMPLE=1`): Workers get only `Bash`, `Read`, and `Edit`
- **Full mode**: Workers get all `ASYNC_AGENT_ALLOWED_TOOLS` minus internal orchestration tools

MCP server names are appended when available, and scratchpad directory info is included if the `tengu_scratch` feature gate is enabled.

### System Prompt Architecture

The system prompt (`getCoordinatorSystemPrompt()`) is a comprehensive instruction document covering:

1. **Role definition** — Coordinator vs worker responsibilities
2. **Tool inventory** — AgentTool, SendMessageTool, TaskStopTool, PR subscription tools
3. **Task notification format** — XML-structured `<task-notification>` blocks with task-id, status, summary, result, and usage
4. **Workflow phases** — Research (parallel workers) → Synthesis (coordinator) → Implementation (workers) → Verification (workers)
5. **Concurrency management** — Guidelines for parallel vs serial execution
6. **Prompt-writing standards** — Anti-patterns ("based on your findings") vs good patterns (specific file paths, line numbers)
7. **Continue vs spawn decision matrix** — When to continue an existing worker vs spawn fresh

## Data Flow

```
User Request
    ↓
Coordinator (isCoordinatorMode=true)
    ↓
┌─────────────────────────────────────┐
│  Phase 1: Research (parallel)       │
│  ├── Worker A: investigate files    │
│  └── Worker B: investigate tests    │
└─────────────────────────────────────┘
    ↓ <task-notification> XML
Coordinator synthesizes findings
    ↓
┌─────────────────────────────────────┐
│  Phase 2: Implementation            │
│  └── Worker (continued or fresh)    │
└─────────────────────────────────────┘
    ↓ <task-notification> XML
┌─────────────────────────────────────┐
│  Phase 3: Verification              │
│  └── Worker (fresh, independent)    │
└─────────────────────────────────────┘
    ↓ <task-notification> XML
Coordinator reports to user
```

## Integration Points

- **AgentTool** — Primary mechanism for spawning workers (`subagent_type: "worker"`)
- **SendMessageTool** — Continuing existing workers with follow-up instructions
- **TaskStopTool** — Stopping misdirected workers
- **Session persistence** — Mode is stored with sessions and restored via `matchSessionMode()`
- **MCP servers** — Worker tool context includes connected MCP server names
- **Scratchpad** — Optional shared directory for cross-worker knowledge (gated by `tengu_scratch`)

## Configuration

| Environment Variable | Purpose |
|---|---|
| `CLAUDE_CODE_COORDINATOR_MODE` | Runtime toggle (`1` = enabled) |
| `CLAUDE_CODE_SIMPLE` | Simplified worker tool set (Bash/Read/Edit only) |

| Feature Gate | Purpose |
|---|---|
| `COORDINATOR_MODE` | Build-time feature flag |
| `tengu_scratch` | Scratchpad directory support |

## Error Handling

- **Mode mismatch on resume**: Automatically corrected with a user-facing warning message
- **Worker failures**: Coordinator continues the same worker (preserving error context) or spawns fresh depending on the failure type
- **Circular dependency avoidance**: `isScratchpadGateEnabled()` duplicates the gate check from `utils/permissions/filesystem.ts` to avoid importing filesystem modules that would create a circular dependency

## Testing

- `clearBuiltinPlugins()` pattern suggests registry clearing for test isolation
- The `matchSessionMode()` function is pure and testable with various session mode combinations

## Related Modules

- [AgentTool](../02-tools/AgentTool.md) — Worker spawning mechanism
- [SendMessageTool](../02-tools/SendMessageTool.md) — Worker continuation
- [TaskStopTool](../02-tools/TaskStopTool.md) — Worker termination
- [Skills System](./skills-system.md) — Skills delegated to workers in coordinator mode

## Notes

- Coordinator mode is an advanced orchestration pattern — the coordinator never executes tools directly (except for PR subscriptions); all code work goes through workers
- The system prompt is one of the longest in the codebase (~250 lines) because it must teach the LLM a complete multi-agent workflow
- Worker results arrive as user-role messages with `<task-notification>` XML tags — the coordinator must distinguish these from actual user messages
- The "continue vs spawn" decision is left to the coordinator's judgment based on context overlap analysis
