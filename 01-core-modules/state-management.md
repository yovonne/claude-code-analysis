# State Management

## Purpose
The Claude Code CLI uses a centralized, Redux-like store pattern built on top of React's `useSyncExternalStore` for efficient, selective subscriptions. A single `AppState` object holds all application state — settings, permission context, tasks, MCP connections, notifications, bridge status, and more. State updates flow through an immutable updater pattern (`setState(prev => next)`) with a central `onChangeAppState` observer that syncs side effects (persisting to disk, notifying CCR/SDK, clearing caches).

## Location
- `restored-src/src/state/` - Core state management (store, AppState, selectors)
- `restored-src/src/context/` - React context providers (notifications, modals, mailbox, stats, overlays)
- `restored-src/src/hooks/` - Custom React hooks (permission, bridge, settings, etc.)
- `restored-src/src/types/permissions.ts` - Permission type definitions
- `restored-src/src/Tool.ts` - ToolPermissionContext and ToolUseContext types

## Key Exports

### AppState
- `AppState`: The single immutable state tree containing all application state (~80+ fields). Wrapped in `DeepImmutable<>` for type safety.
- `AppStateStore`: The store interface (`Store<AppState>`) with `getState`, `setState`, `subscribe`
- `getDefaultAppState()`: Factory function returning the default initial state
- `AppStoreContext`: React context holding the store instance
- `useAppState(selector)`: Selector-based subscription hook using `useSyncExternalStore` — only re-renders when the selected value changes (Object.is comparison)
- `useSetAppState()`: Returns the stable `setState` updater — never causes re-renders
- `useAppStateStore()`: Returns the full store for non-React code access
- `useAppStateMaybeOutsideOfProvider()`: Safe version that returns `undefined` outside provider

### Context Providers
- `AppStateProvider`: Root provider wrapping the app, creates the store, nests `MailboxProvider` and `VoiceProvider`
- `MailboxProvider`: Provides a singleton `Mailbox` instance for inter-component messaging
- `StatsProvider`: Provides a metrics/stats store with increment, observe, histogram support
- `NotificationContext` (via `useNotifications`): Priority-based notification queue with fold, invalidate, timeout
- `ModalContext`: Tracks modal layout dimensions for proper sizing of overlay content
- `OverlayContext`: Tracks active overlays for Escape key coordination
- `QueuedMessageContext`: Tracks message queue state for brief layout rendering

### Custom Hooks
- `useAppState(selector)`: Subscribe to a slice of AppState with selector-based memoization
- `useSetAppState()`: Get the setState updater without subscribing
- `useAppStateStore()`: Get the full store reference
- `useNotifications()`: Add/remove notifications with priority queue
- `useRegisterOverlay(id)`: Register/unregister an overlay component
- `useIsOverlayActive()`: Check if any overlay is active
- `useCanUseTool()`: Permission checking hook for tool execution
- `useReplBridge()`: Bridge connection lifecycle hook
- `useSettingsChange()`: Listen for settings file changes

## AppState Architecture

The `AppState` type is a large frozen object (~80+ fields) organized into logical groups:

```typescript
export type AppState = DeepImmutable<{
  // Core settings & model
  settings: SettingsJson
  verbose: boolean
  mainLoopModel: ModelSetting
  mainLoopModelForSession: ModelSetting

  // UI state
  statusLineText: string | undefined
  expandedView: 'none' | 'tasks' | 'teammates'
  isBriefOnly: boolean
  selectedIPAgentIndex: number
  coordinatorTaskIndex: number
  viewSelectionMode: 'none' | 'selecting-agent' | 'viewing-agent'
  footerSelection: FooterItem | null

  // Permission state
  toolPermissionContext: ToolPermissionContext

  // Task state (excluded from DeepImmutable — contains functions)
  tasks: { [taskId: string]: TaskState }
  foregroundedTaskId?: string
  viewingAgentTaskId?: string

  // MCP & plugins
  mcp: { clients, tools, commands, resources, pluginReconnectKey }
  plugins: { enabled, disabled, commands, errors, installationStatus, needsRefresh }

  // Notifications & elicitation
  notifications: { current: Notification | null, queue: Notification[] }
  elicitation: { queue: ElicitationRequestEvent[] }

  // Bridge (remote control)
  replBridgeEnabled, replBridgeConnected, replBridgeSessionActive, ...
  replBridgePermissionCallbacks?: BridgePermissionCallbacks

  // Team/swarm context
  teamContext?: { teamName, teammates, leadAgentId, ... }
  inbox: { messages: Array<{ id, from, text, timestamp, status }> }

  // Speculation (pre-computation)
  speculation: SpeculationState
  speculationSessionTimeSavedMs: number

  // ... and many more: todos, fileHistory, attribution, promptSuggestion, etc.
}>
```

Key design decisions:
- **`DeepImmutable<>` wrapper**: Most fields are deeply immutable for predictable updates. `tasks` is excluded because `TaskState` contains function types (AbortController, callbacks).
- **Mutable refs in speculation**: `SpeculationState.active` uses `messagesRef` and `writtenPathsRef` as mutable refs to avoid array spreading per message — a performance optimization for the hot path.
- **Feature-flagged fields**: Many fields are optional and only populated when feature flags are enabled (`BRIDGE_MODE`, `KAIROS`, `CHICAGO_MCP`, etc.).

## React State Management

The store is a minimal external store implementation (`src/state/store.ts:10-34`):

```typescript
export function createStore<T>(initialState: T, onChange?: OnChange<T>): Store<T> {
  let state = initialState
  const listeners = new Set<Listener>()

  return {
    getState: () => state,
    setState: (updater: (prev: T) => T) => {
      const prev = state
      const next = updater(prev)
      if (Object.is(next, prev)) return  // Skip if no change
      state = next
      onChange?.({ newState: next, oldState: prev })
      for (const listener of listeners) listener()
    },
    subscribe: (listener: Listener) => {
      listeners.add(listener)
      return () => listeners.delete(listener)
    },
  }
}
```

Key characteristics:
- **Updater pattern**: All state changes go through `setState(prev => next)` — no direct mutation
- **Object.is equality check**: Skips notifications if the new state is referentially identical to the old
- **onChange callback**: Fires before listeners, used for side effects (persistence, CCR sync)
- **Listener fan-out**: All subscribed listeners are notified synchronously

The `AppStateProvider` (`src/state/AppState.tsx:37-110`) creates the store once via `useState` and provides it through `AppStoreContext`. It also:
1. Prevents nesting (throws if `AppStateProvider` is inside another `AppStateProvider`)
2. Initializes bypass permission mode on mount if disabled by remote settings
3. Subscribes to settings file changes via `useSettingsChange`
4. Wraps children in `MailboxProvider` and `VoiceProvider`

### useAppState — Selector-Based Subscriptions

```typescript
export function useAppState<T>(selector: (state: AppState) => T): T {
  const store = useAppStore()
  const get = () => selector(store.getState())
  return useSyncExternalStore(store.subscribe, get, get)
}
```

Uses React 18's `useSyncExternalStore` for concurrent-safe subscriptions. The selector function extracts a specific field, and React only re-renders when `Object.is` detects a change in the selected value.

**Best practice**: Select individual fields, not the whole state:
```typescript
const verbose = useAppState(s => s.verbose)          // good
const mode = useAppState(s => s.toolPermissionContext.mode)  // good
const state = useAppState(s => s)                    // BAD — re-renders on every change
```

## Context Providers

### AppStateProvider
The root provider. Creates the store, handles mount-time permission initialization, and subscribes to settings changes.

### MailboxProvider (`src/context/mailbox.tsx`)
Provides a singleton `Mailbox` instance for pub/sub messaging between components. Created once with `useMemo(() => new Mailbox(), [])`.

### StatsProvider (`src/context/stats.tsx`)
Provides a metrics store with:
- **Counters**: `increment(name, value)` — cumulative counts
- **Gauges**: `set(name, value)` — current values
- **Histograms**: `observe(name, value)` — reservoir sampling (Algorithm R, size 1024) with p50/p95/p99 percentiles
- **Sets**: `add(name, value)` — unique value tracking

Metrics are flushed to `lastSessionMetrics` in project config on process exit.

### Notification System (`src/context/notifications.tsx`)
Priority-based notification queue with four levels: `immediate`, `high`, `medium`, `low`.

Key features:
- **Priority scheduling**: `getNext()` selects the highest-priority notification from the queue
- **Folding**: Notifications with the same key can be merged via a `fold` function
- **Invalidation**: Notifications can invalidate others via the `invalidates` array
- **Auto-timeout**: Default 8-second timeout, configurable per notification
- **Immediate bypass**: `immediate` priority notifications skip the queue and display instantly

### OverlayContext (`src/context/overlayContext.tsx`)
Tracks active overlays via `activeOverlays: ReadonlySet<string>` in AppState. Used for Escape key coordination — when an overlay is active, Escape dismisses the overlay instead of canceling the running task.

- `useRegisterOverlay(id, enabled?)`: Auto-registers on mount, unregisters on unmount
- `useIsOverlayActive()`: Returns `true` if any overlay is active
- `useIsModalOverlayActive()`: Returns `true` if any modal (non-autocomplete) overlay is active

### ModalContext (`src/context/modalContext.tsx`)
Provides modal layout dimensions (`rows`, `columns`, `scrollRef`) for proper sizing of content inside slash-command dialogs.

## State Persistence

State persistence is handled at multiple levels:

### 1. onChangeAppState Observer (`src/state/onChangeAppState.ts`)
The `onChangeAppState` function fires on every state change and handles:

- **Permission mode sync**: Detects mode changes and notifies CCR via `notifySessionMetadataChanged()` and SDK via `notifyPermissionModeChanged()`. Externalizes internal modes (e.g., `bubble` → `default`) before sending.
- **Model settings**: When `mainLoopModel` changes, persists to user settings via `updateSettingsForSource()`
- **UI preferences**: `expandedView` → persists `showExpandedTodos` and `showSpinnerTree` to global config
- **Verbose flag**: Persists to global config
- **Tungsten panel**: Persists `tungstenPanelVisible` to global config (ant-only)
- **Auth cache clearing**: Clears API key, AWS, and GCP credential caches when settings change
- **Environment variables**: Re-applies config environment variables when `settings.env` changes

### 2. Global Config
UI preferences (verbose, expanded view, tungsten panel) are persisted via `saveGlobalConfig()` to a JSON file on disk.

### 3. Settings System
Model, permission rules, and other user preferences are persisted via `updateSettingsForSource()` to `settings.json` (user-level, project-level, or local).

### 4. Stats Persistence
Session metrics are flushed to `lastSessionMetrics` in project config on process exit via a `process.on('exit')` handler.

## State Update Patterns

### Immutable Updater Pattern
All state changes use the functional updater pattern:
```typescript
setAppState(prev => ({
  ...prev,
  toolPermissionContext: {
    ...prev.toolPermissionContext,
    mode: newMode,
  },
}))
```

### Nested Object Updates
For nested objects, spread at each level:
```typescript
setAppState(prev => ({
  ...prev,
  notifications: {
    ...prev.notifications,
    current: next,
    queue: prev.notifications.queue.filter(n => n !== next),
  },
}))
```

### Set/Map Updates
For `Set` and `Map` fields, create new instances:
```typescript
setAppState(prev => {
  const next = new Set(prev.activeOverlays)
  next.add(id)
  return { ...prev, activeOverlays: next }
})
```

### Batched Updates
Multiple fields can be updated in a single `setAppState` call. The `Object.is` check at the store level prevents unnecessary notifications if the resulting state is referentially identical.

### Selectors for Computed State
The `selectors.ts` file provides pure functions for deriving computed state:
- `getViewedTeammateTask(appState)`: Returns the currently viewed teammate task or `undefined`
- `getActiveAgentForInput(appState)`: Returns a discriminated union indicating where user input should be routed (`leader`, `viewed`, or `named_agent`)

## Message State

Messages are not stored in `AppState` directly — they live in the REPL component's local state and are passed to hooks like `useReplBridge`. However, AppState contains message-related state:

- `initialMessage`: The initial CLI message to process, with optional `clearContext`, `mode`, and `allowedPrompts`
- `promptSuggestion`: AI-generated follow-up suggestion with `text`, `promptId`, `shownAt`, `acceptedAt`
- `notifications`: Notification queue (not conversation messages, but UI notifications)
- `elicitation`: Queue of MCP elicitation requests

Message types are defined in `src/types/message.js` (runtime file, not `.ts`):
- `UserMessage`, `AssistantMessage`, `SystemMessage`, `ProgressMessage`, `AttachmentMessage`
- `SystemLocalCommandMessage`: UI-only messages stripped at the API boundary

## Permission State

Permission state is the most complex part of AppState, centered on `toolPermissionContext`:

```typescript
type ToolPermissionContext = {
  mode: PermissionMode                         // 'default' | 'plan' | 'bypassPermissions' | 'auto' | 'bubble' | ...
  additionalWorkingDirectories: Map<string, AdditionalWorkingDirectory>
  alwaysAllowRules: ToolPermissionRulesBySource  // { userSettings?: string[], projectSettings?: string[], ... }
  alwaysDenyRules: ToolPermissionRulesBySource
  alwaysAskRules: ToolPermissionRulesBySource
  isBypassPermissionsModeAvailable: boolean
  shouldAvoidPermissionPrompts?: boolean
  awaitAutomatedChecksBeforeDialog?: boolean
  prePlanMode?: PermissionMode                   // Stashed mode before entering plan mode
}
```

### Permission Modes
- **`default`**: Ask for permission before each tool use
- **`plan`**: Plan mode — no tool execution, only planning
- **`bypassPermissions`**: Auto-approve all tools (requires `--dangerously-skip-permissions`)
- **`auto`**: AI classifier decides whether to allow or ask (feature-flagged)
- **`bubble`**: Internal mode for subagents

### Permission Decision Flow (`src/hooks/useCanUseTool.tsx`)
1. Tool requests permission via `canUseTool(tool, input, toolUseContext, ...)`
2. `createPermissionContext()` builds a frozen context object with all decision methods
3. `hasPermissionsToUseTool()` checks rules against the current mode
4. Result is `allow`, `deny`, or `ask`:
   - **allow**: Tool executes immediately
   - **deny**: Tool is rejected with a reason
   - **ask**: Enters the permission queue for user interaction
5. For `ask` decisions, the flow routes through handlers:
   - `handleCoordinatorPermission()`: Coordinator worker checks
   - `handleSwarmWorkerPermission()`: Swarm member checks
   - `handleInteractivePermission()`: Interactive UI prompt
   - Bash classifier: Async auto-approval for safe commands

### Permission Persistence
When a user allows/denies permanently, `PermissionUpdate` objects are created:
```typescript
type PermissionUpdate =
  | { type: 'addRules', destination, rules, behavior }
  | { type: 'replaceRules', destination, rules, behavior }
  | { type: 'removeRules', destination, rules, behavior }
  | { type: 'setMode', destination, mode }
  | { type: 'addDirectories', destination, directories }
  | { type: 'removeDirectories', destination, directories }
```

## Tool State

Tool execution state is managed through `ToolUseContext` (`src/Tool.ts:158-250`), which provides:
- `options`: Configuration (commands, model, tools, MCP clients, etc.)
- `abortController`: For canceling tool execution
- `getAppState()` / `setAppState()`: Access to the root state
- `setToolJSX()`: For rendering tool UI
- `addNotification()`: For tool-level notifications
- `appendSystemMessage()`: For injecting system messages
- `updateFileHistoryState()` / `updateAttributionState()`: Side-effect updaters

The `useCanUseTool` hook creates a `CanUseToolFn` that wraps the permission checking pipeline. It returns a `Promise<PermissionDecision>` that resolves when the user (or classifier/hook) makes a decision.

## Session State

Session state spans several AppState fields:

### Bridge State (Remote Control)
The `useReplBridge` hook (`src/hooks/useReplBridge.tsx`) manages the always-on bridge connection:
- `replBridgeEnabled`: User toggle for bridge
- `replBridgeConnected`: Transport is up
- `replBridgeSessionActive`: User is connected on claude.ai
- `replBridgeReconnecting`: In backoff retry
- `replBridgeConnectUrl` / `replBridgeSessionUrl`: URLs for display
- `replBridgePermissionCallbacks`: Bidirectional permission check callbacks
- `replBridgeError`: Error message for display

The hook handles:
1. **Initialization**: Dynamic import of `initReplBridge`, OAuth check, environment registration
2. **Message forwarding**: Writes new messages to the bridge as they arrive
3. **Inbound messages**: Injects claude.ai messages into the REPL queue
4. **State mapping**: Maps bridge lifecycle events (`ready`, `connected`, `reconnecting`, `failed`) to AppState
5. **Permission delegation**: Routes permission requests to CCR for remote approval
6. **Teardown**: Cleans up on disable, handles rapid toggle with teardown promise chaining
7. **Failure recovery**: Consecutive failure counter with fuse (max 3 failures → auto-disable)

### Swarm/Team State
- `teamContext`: Team name, file path, lead agent ID, teammate registry
- `agentNameRegistry`: Map of agent names to AgentIds for routing
- `inbox`: Messages from teammates
- `workerSandboxPermissions`: Queue of worker network access approval requests
- `pendingWorkerRequest` / `pendingSandboxRequest`: Active permission requests

### Task State
- `tasks`: Dictionary of `TaskState` objects keyed by task ID
- `foregroundedTaskId`: Task whose messages are shown in the main view
- `viewingAgentTaskId`: Task being viewed in teammate transcript mode

Task state management helpers (`src/state/teammateViewHelpers.ts`):
- `enterTeammateView(taskId, setAppState)`: Switch to viewing a teammate's transcript
- `exitTeammateView(setAppState)`: Return to leader's view
- `stopOrDismissAgent(taskId, setAppState)`: Abort running agent or dismiss terminal agent

## Key Algorithms

### Store Equality Check
```typescript
if (Object.is(next, prev)) return  // Skip notification
```
Uses `Object.is` for referential equality — if the updater returns the same reference, no listeners fire. This enables early returns in updaters for no-op changes.

### Notification Queue Processing
```typescript
export function getNext(queue: Notification[]): Notification | undefined {
  if (queue.length === 0) return undefined
  return queue.reduce((min, n) =>
    PRIORITIES[n.priority] < PRIORITIES[min.priority] ? n : min
  )
}
```
Selects the highest-priority notification (lowest priority number) from the queue.

### Speculation State
The `SpeculationState` type supports pre-computing AI responses while the user is still reading:
```typescript
type SpeculationState =
  | { status: 'idle' }
  | {
      status: 'active'
      id: string
      abort: () => void
      messagesRef: { current: Message[] }     // Mutable ref — avoids array spreading
      writtenPathsRef: { current: Set<string> }
      boundary: CompletionBoundary | null
      // ...
    }
```
Uses mutable refs (`messagesRef`, `writtenPathsRef`) to avoid the O(n) array copy on each new message during speculation.

### Permission Mode Transition
Mode changes go through `transitionPermissionMode()` which handles:
- Stashing the previous mode when entering plan mode (`prePlanMode`)
- Restoring the previous mode when exiting plan mode
- Auto-mode gate checks before allowing `auto` mode
- Bypass permissions availability checks

## Dependencies

### Internal
- `src/state/store.ts` — Minimal external store implementation
- `src/state/AppStateStore.ts` — AppState type and default state factory
- `src/state/AppState.tsx` — React provider and hooks
- `src/state/onChangeAppState.ts` — Side effect observer
- `src/state/selectors.ts` — Computed state selectors
- `src/state/teammateViewHelpers.ts` — Teammate view state transitions
- `src/types/permissions.ts` — Permission type definitions
- `src/Tool.ts` — Tool context and permission context types
- `src/utils/settings/settings.ts` — Settings management
- `src/utils/config.ts` — Global config persistence
- `src/utils/permissions/PermissionMode.ts` — Permission mode utilities

### External
- `react` — `useSyncExternalStore`, `useContext`, `useState`, `useEffect`, `useLayoutEffect`, `useCallback`, `useMemo`, `useRef`
- `react/compiler-runtime` — React Compiler memoization (`_c`)
- `bun:bundle` — Feature flag system (`feature()`)
- `lodash-es/memoize.js` — Memoization for context functions

## Data Flow

```
CLI Args / Settings File
        │
        ▼
  getDefaultAppState() ────► createStore(initialState, onChangeAppState)
        │                              │
        ▼                              ▼
  AppStateProvider              setState(updater)
        │                              │
        │                        ┌─────┴─────┐
        │                        ▼           ▼
        │                  onChangeAppState  listeners
        │                        │           │
        │                   ┌────┴────┐      │
        │                   ▼         ▼      ▼
        │              Persist    Sync CCR  useSyncExternalStore
        │              to disk    / SDK          │
        │                                        ▼
        └────────────────────────────────── React Components
                                                   │
                                             useAppState(selector)
                                                   │
                                             Selective re-render
```

1. **Initialization**: `getDefaultAppState()` creates the initial state, `createStore()` wraps it
2. **Provider**: `AppStateProvider` creates the store once and provides it via context
3. **Updates**: Components call `useSetAppState()` to get `setState`, then call it with an updater function
4. **Store**: The store runs the updater, checks `Object.is`, fires `onChangeAppState`, then notifies listeners
5. **Side effects**: `onChangeAppState` persists changes, syncs with CCR/SDK, clears caches
6. **Subscriptions**: `useAppState(selector)` uses `useSyncExternalStore` to subscribe — only re-renders when the selected value changes

## Integration Points

### Main Entry (`main.tsx`)
- Creates the initial `AppState` with CLI args
- Computes `kairosEnabled` (assistant mode) before option mutation
- Sets `initialMessage` from CLI prompt or plan mode exit

### Tool System
- `ToolUseContext` carries `getAppState()` and `setAppState()` to every tool
- `useCanUseTool` hook creates the permission checking pipeline
- Tools update AppState for progress, JSX rendering, and notifications

### Query Engine
- Messages flow through the query engine but are stored in REPL local state
- Bridge forwards messages via `useReplBridge`'s message watching effect
- System/init message is sent through the bridge on connection

### Settings System
- `useSettingsChange` hook watches for settings file changes
- `applySettingsChange` updates AppState when settings are modified
- `onChangeAppState` persists AppState changes back to settings

### CCR (Claude Code Remote)
- `onChangeAppState` syncs permission mode changes via `notifySessionMetadataChanged()`
- `useReplBridge` maintains a WebSocket connection for bidirectional communication
- Bridge permission callbacks route remote permission requests to the local UI

### SDK
- `notifyPermissionModeChanged()` fires SDK status stream updates
- SDK mode changes are applied via `externalMetadataToAppState()` on worker restart

## Related Modules
- [Main Entrypoint](./main-entrypoint.md)
- [Tool System](./tool-system.md)
- [Query Engine](./query-engine.md)
