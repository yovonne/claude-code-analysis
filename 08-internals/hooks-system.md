# Hooks System

## Purpose

The hooks system provides React hooks for notification display, tool permission handling, and various UI/UX features. It is organized into three areas: general hooks (70+ hooks in `src/hooks/`), notification hooks (`hooks/notifs/`), and tool permission hooks (`hooks/toolPermission/`).

## Location

`restored-src/src/hooks/`

## Key Subdirectories

### hooks/notifs/ — Notification Hooks

Specialized hooks for displaying startup and runtime notifications.

| Hook | Purpose |
|------|---------|
| `useStartupNotification` | Fires notification(s) once on mount; encapsulates remote-mode gate and once-per-session ref guard |
| `useModelMigrationNotifications` | Shows one-time notifications after model migrations (Sonnet 4.6, Opus 4.6) |
| `useDeprecationWarningNotification` | Warns when using a deprecated model |
| `useFastModeNotification` | Handles fast mode cooldown, overage rejection, and org availability changes |
| `useAutoModeUnavailableNotification` | Shows notification when auto mode is unavailable during shift-tab carousel |
| `useRateLimitWarningNotification` | Displays rate limit warnings and overage notifications from claude.ai |
| `useSettingsErrors` | Displays settings validation errors |
| `usePluginAutoupdateNotification` | Notifies about plugin auto-update status |
| `usePluginInstallationStatus` | Tracks plugin installation progress |
| `useMcpConnectivityStatus` | Monitors MCP server connectivity |
| `useLspInitializationNotification` | Notifies about LSP server initialization |
| `useTeammateShutdownNotification` | Notifies when a swarm teammate shuts down |
| `useNpmDeprecationNotification` | Warns about deprecated npm packages |
| `useInstallMessages` | Shows installation-related messages |
| `useIDEStatusIndicator` | IDE connection status indicator |
| `useCanSwitchToExistingSubscription` | Subscription switching capability check |

### hooks/toolPermission/ — Permission Flow

Handles the three permission flows: coordinator, interactive, and swarm worker.

#### `PermissionContext.ts`

Creates a frozen permission context object that encapsulates all operations for a single tool permission request.

**Key exports:**
- `createPermissionContext(tool, input, toolUseContext, assistantMessage, toolUseID, ...)`: Factory for permission context
- `createPermissionQueueOps(setToolUseConfirmQueue)`: Bridges React state to generic queue interface
- `createResolveOnce(resolve)`: Atomic guard preventing multiple promise resolutions

**Context methods:**
- `logDecision(args)`: Logs permission decisions via `permissionLogging.ts`
- `logCancelled()`: Logs tool cancellation analytics
- `persistPermissions(updates)`: Persists permission updates to settings
- `resolveIfAborted(resolve)`: Checks abort signal and resolves with cancel decision
- `cancelAndAbort(feedback, isAbort, contentBlocks)`: Builds deny decision and optionally aborts
- `tryClassifier(pendingCheck, updatedInput)`: Runs bash classifier auto-approval (BASH_CLASSIFIER flag)
- `runHooks(permissionMode, suggestions, ...)`: Executes PermissionRequest hooks
- `buildAllow(updatedInput, opts)`: Constructs allow decision
- `buildDeny(message, reason)`: Constructs deny decision
- `handleUserAllow(...)`: Processes user acceptance with persistence and logging
- `handleHookAllow(...)`: Processes hook-based acceptance
- `pushToQueue()`, `removeFromQueue()`, `updateQueueItem()`: Queue operations

#### `handlers/coordinatorHandler.ts`

Handles permission flow for coordinator workers (non-interactive).

- Runs hooks first (fast, local), then classifier (slow, inference)
- Falls through to interactive dialog if neither resolves
- Returns `PermissionDecision | null`

#### `handlers/interactiveHandler.ts`

Handles the main-agent interactive permission flow — the most complex handler.

**Key responsibilities:**
- Pushes `ToolUseConfirm` entry to the confirm queue with callbacks
- Races 4 permission sources: local user, bridge (CCR), channel (Telegram/iMessage), hooks/classifier
- Uses `claim()` atomic guard to prevent multiple resolutions
- Manages checkmark transition timer for classifier auto-approval
- Handles bridge permission requests with unique request IDs
- Handles channel permission relay via MCP notifications
- Runs hooks and classifier asynchronously in the background

**Callback lifecycle:**
- `onUserInteraction()`: Hides classifier indicator (200ms grace period)
- `onDismissCheckmark()`: Dismisses checkmark transition early (Esc key)
- `onAbort()`: User aborts — resolves with cancel decision
- `onAllow()`: User accepts — persists and resolves
- `onReject()`: User denies — resolves with feedback
- `recheckPermission()`: Re-evaluates permissions (e.g., after mode switch)

#### `handlers/swarmWorkerHandler.ts`

Handles permission flow for swarm worker agents.

- Tries classifier auto-approval first
- Forwards permission request to leader via mailbox
- Registers callbacks for leader response
- Shows pending indicator while waiting
- Falls back to local handling on error

### General Hooks (selected)

| Category | Hooks |
|----------|-------|
| **Input** | `useTextInput`, `useVimInput`, `useArrowKeyHistory`, `useHistorySearch`, `useTypeahead`, `usePasteHandler`, `useSearchInput` |
| **State** | `useAppState`, `useSettings`, `useSettingsChange`, `useDynamicConfig`, `useMemoryUsage` |
| **Session** | `useAssistantHistory`, `useRemoteSession`, `useSessionBackgrounding`, `useTeleportResume`, `useAwaySummary` |
| **Tools** | `useCanUseTool`, `useCancelRequest`, `useTurnDiffs`, `useDiffData`, `useDiffInIDE` |
| **Navigation** | `useBackgroundTaskNavigation`, `useTaskListWatcher`, `useTasksV2` |
| **IDE** | `useIDEIntegration`, `useIdeAtMentioned`, `useIdeConnectionStatus`, `useIdeLogging`, `useIdeSelection` |
| **Voice** | `useVoice`, `useVoiceEnabled`, `useVoiceIntegration` |
| **UI** | `useBlink`, `useClipboardImageHint`, `useMinDisplayTime`, `useVirtualScroll`, `useTerminalSize` |
| **Features** | `useManagePlugins`, `useLspPluginRecommendation`, `useSkillImprovementSurvey`, `useSwarmInitialization` |
| **Messaging** | `useInboxPoller`, `useMailboxBridge`, `useCommandQueue`, `useDeferredHookMessages`, `useLogMessages` |
| **Performance** | `useAfterFirstRender`, `useElapsedTime`, `useNotifyAfterTimeout`, `useTimeout` |
| **Keybindings** | `useCommandKeybindings`, `useGlobalKeybindings`, `useExitOnCtrlCD` |

## Dependencies

### Internal Dependencies

- `bootstrap/state.js` — Remote mode check, allowed channels
- `context/notifications.js` — Notification system
- `state/AppState.js` — App state selectors and setters
- `Tool.js` — Tool types and ToolUseContext
- `utils/permissions/*` — Permission rules, updates, results
- `utils/hooks.js` — Hook execution
- `utils/swarm/permissionSync.js` — Swarm permission forwarding
- `services/analytics/index.js` — Analytics logging
- `services/mcp/*` — Channel permission relay

### External Dependencies

- `react` — All hooks are React hooks
- `react/compiler-runtime` — React Compiler optimization (`_c` cache)
- `bun:bundle` — Feature flag checks
- `@anthropic-ai/sdk` — Content block types

## Implementation Details

### React Compiler

All notification hooks are compiled with React Compiler (evidenced by `import { c as _c } from "react/compiler-runtime"` and manual cache arrays `$[0]`, `$[1]`, etc.). This eliminates manual `useMemo`/`useCallback` usage.

### Resolve-Once Pattern

The `createResolveOnce()` factory provides atomic resolution guards with three operations:
- `resolve(value)`: Delivers value once (idempotent)
- `isResolved()`: Checks if already resolved
- `claim()`: Atomic check-and-mark — returns true only if this caller won the race

This prevents race conditions when multiple permission sources (user, hook, classifier, bridge, channel) compete.

### Permission Flow Decision Tree

```
Tool needs permission
    ↓
Is swarm worker? → handleSwarmWorkerPermission()
    ↓ No
Is coordinator? → handleCoordinatorPermission()
    ↓ No (interactive)
handleInteractivePermission()
    ├── Push to queue (UI dialog)
    ├── Race: Bridge response (CCR)
    ├── Race: Channel response (Telegram/iMessage)
    ├── Race: Hooks (async)
    └── Race: Classifier (async, bash only)
    ↓
First to claim() wins
```

## Related Modules

- [Bootstrap](./bootstrap.md) — Permission state reads from bootstrap
- [Schemas](./schemas.md) — Hook schemas define hook configuration structure
- [Context Providers](./context-providers.md) — Notification context consumed by hooks
- [Types](./types-system.md) — Permission types define decision shapes
