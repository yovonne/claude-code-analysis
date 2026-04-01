# Bootstrap

## Purpose

The bootstrap module is the DAG (Directed Acyclic Graph) leaf of the application — it holds all global mutable state and provides the single source of truth for session-level data. Every other module imports from bootstrap but bootstrap imports from nothing (except `src/utils/crypto.js` via an explicit bypass). It defines the `State` type, a singleton `STATE` object, and ~120 getter/setter functions that control access to individual state fields.

## Location

`restored-src/src/bootstrap/state.ts`

## Key Exports

### Types

- `State`: The complete application state shape (~100 fields covering session identity, costs, durations, telemetry, model settings, agent colors, hooks, skills, cron tasks, beta header latches, and more)
- `ChannelEntry`: Union type for registered channel plugins/servers with marketplace info
- `AttributedCounter`: Interface for OpenTelemetry attributed counters
- `SessionCronTask`: In-memory cron task type (never persisted to disk)
- `InvokedSkillInfo`: Tracks skills invoked during a session for preservation across compaction

### Core Functions

- `getSessionId()`: Returns the current session UUID
- `regenerateSessionId()`: Creates a new session ID, optionally setting current as parent
- `switchSession(sessionId, projectDir)`: Atomically switches session ID and project directory together
- `onSessionSwitch`: Signal subscription for session change notifications
- `getProjectRoot()`: Returns the stable project root (never updated by mid-session worktree changes)
- `getOriginalCwd()`: Returns the original working directory at startup

### Cost & Duration Tracking

- `addToTotalCostState()`: Accumulates API cost and per-model usage
- `addToTotalDurationState()`: Accumulates API duration with/without retries
- `addToToolDuration()`: Accumulates tool execution duration per turn
- `addToTurnHookDuration()`: Accumulates hook execution duration per turn
- `addToTurnClassifierDuration()`: Accumulates classifier duration per turn
- `getTotalDuration()`: Returns wall-clock time since session start
- `snapshotOutputTokensForTurn()`: Captures output token count at turn start for per-turn budgeting

### Telemetry State

- `setMeter()`: Initializes OpenTelemetry meter and all counters (session, LOC, PR, commit, cost, token, code edit decisions, active time)
- `setLoggerProvider()` / `setEventLogger()`: OTel logging state
- `setMeterProvider()` / `setTracerProvider()`: OTel provider state
- `getStatsStore()`: Returns the stats observation store

### Model & Feature State

- `getMainLoopModelOverride()`: Returns the `--model` CLI flag override
- `setMainLoopModelOverride()`: Sets the model override
- `getSdkBetas()` / `setSdkBetas()`: SDK-provided beta feature flags
- `getKairosActive()` / `setKairosActive()`: KAIROS (Ant-internal) feature state

### Session Lifecycle

- `setSessionTrustAccepted()`: Session-only trust flag (not persisted)
- `setSessionPersistenceDisabled()`: Disables session persistence to disk
- `hasExitedPlanModeInSession()` / `setHasExitedPlanMode()`: Track plan mode exit
- `handlePlanModeTransition()`: Manages plan mode exit attachment flags
- `handleAutoModeTransition()`: Manages auto mode exit attachment flags

### Beta Header Latches

Four "sticky-on" latches prevent prompt cache busting when features are toggled mid-session:

- `afkModeHeaderLatched`: Auto mode beta header
- `fastModeHeaderLatched`: Fast mode beta header
- `cacheEditingHeaderLatched`: Cache editing beta header
- `thinkingClearLatched`: Thinking-clear beta header (triggered after >1h idle)

### Scroll Drain

- `markScrollActivity()`: Marks that a scroll event happened (150ms debounce)
- `getIsScrollDraining()`: Returns true during scroll drain period
- `waitForScrollIdle()`: Async barrier for expensive operations during scroll

### Hook Management

- `registerHookCallbacks()`: Merges SDK/plugin hook matchers into state
- `getRegisteredHooks()` / `clearRegisteredHooks()`: Full hook lifecycle
- `clearRegisteredPluginHooks()`: Removes only plugin hooks, keeps SDK callbacks
- `resetSdkInitState()`: Clears both hooks and JSON schema

### Skills Tracking

- `addInvokedSkill()`: Records a skill invocation with agent-scoped key
- `getInvokedSkills()`: Returns all invoked skills for compaction preservation

### Testing

- `resetStateForTests()`: Full state reset (only callable in `NODE_ENV=test`)
- `resetCostState()`: Resets cost/duration counters for a new session
- `setCostStateForRestore()`: Restores cost state from a resumed session

## Dependencies

### Internal Dependencies

None — this is the DAG leaf. The only import from `src/` is `src/utils/crypto.js` (explicitly allowed via eslint-disable for the `randomUUID` function).

### External Dependencies

- `@anthropic-ai/sdk` — `BetaMessageStreamParams` type
- `@opentelemetry/api` — `Attributes`, `Meter`, `MetricOptions` types
- `@opentelemetry/api-logs` — `logs` type
- `@opentelemetry/sdk-logs` — `LoggerProvider` type
- `@opentelemetry/sdk-metrics` — `MeterProvider` type
- `@opentelemetry/sdk-trace-base` — `BasicTracerProvider` type
- `lodash-es/sumBy.js` — Token aggregation
- `fs` — `realpathSync` for CWD resolution
- `process` — `cwd()` for current working directory

## Implementation Details

### Core Logic

The module uses a single module-scoped `STATE` constant initialized by `getInitialState()`. All mutations go through explicit setter functions — there is no direct state access from outside the module. This enforces a strict API surface and makes it easy to add side effects (signals, validation) to any state change.

### Signal System

The `sessionSwitched` signal (created via `createSignal`) provides a pub/sub mechanism for session changes. Callers register via `onSessionSwitch` (exported as the subscribe function). This avoids importing listeners directly into bootstrap, keeping it a DAG leaf.

### Interaction Time Batching

`updateLastInteractionTime()` uses a dirty flag pattern to batch `Date.now()` calls. Keypresses set `interactionTimeDirty = true`; `flushInteractionTime()` (called before each Ink render) performs the actual `Date.now()` call. This avoids calling `Date.now()` on every keypress.

### Post-Compaction Flag

`markPostCompaction()` / `consumePostCompaction()` implements a single-shot flag pattern. After compaction, the next API success event is tagged as post-compaction to distinguish cache misses from TTL expiry. The flag auto-resets after consumption.

### Scroll Drain

A 150ms debounce window after the last scroll event blocks expensive one-shot work (network, subprocess) from competing with scroll frames for the event loop. `markScrollActivity()` resets the timer; `waitForScrollIdle()` polls until the flag clears.

## Data Flow

```
main.tsx → bootstrap/state.ts (initialization)
    ↓
All modules → bootstrap getters (read state)
Tools → bootstrap setters (update cost, duration, etc.)
QueryEngine → bootstrap setters (update model usage, API requests)
```

## Integration Points

- **main.tsx**: Initializes all state at startup, sets meter/logger/tracer providers
- **Tool execution**: Updates cost, duration, and line change counters
- **QueryEngine**: Records API requests, model usage, and compaction state
- **Session management**: `switchSession()` coordinates with concurrent session PID files
- **Telemetry**: All OTel counters are initialized and accessed through bootstrap

## Error Handling

- `resetStateForTests()` throws if called outside test environment
- `addToInMemoryErrorLog()` maintains a bounded ring buffer (max 100 errors)
- Scroll drain uses `setTimeout(...).unref()` to prevent blocking process exit

## Related Modules

- [Context Providers](./context-providers.md) — React contexts that read from bootstrap state
- [Hooks System](./hooks-system.md) — Hook registration flows through bootstrap state
- [Types System](./types-system.md) — `SessionId` and `AgentId` branded types used throughout
