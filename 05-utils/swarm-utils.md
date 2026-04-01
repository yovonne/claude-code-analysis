# Swarm Utilities

## Purpose

The swarm utilities module provides the complete infrastructure for Claude Code's agent team (swarm) feature. It handles team configuration, teammate spawning (both process-based via tmux/iTerm2 and in-process), permission synchronization, reconnection, layout management, and backend detection for different terminal environments.

## Location

- `restored-src/src/utils/swarm/constants.ts` — Swarm constants and environment variables
- `restored-src/src/utils/swarm/teamHelpers.ts` — Team file CRUD, member management, cleanup
- `restored-src/src/utils/swarm/teammateModel.ts` — Default model fallback for teammates
- `restored-src/src/utils/swarm/teammateInit.ts` — Teammate hook initialization
- `restored-src/src/utils/swarm/teammateLayoutManager.ts` — Color assignment, pane creation, backend delegation
- `restored-src/src/utils/swarm/teammatePromptAddendum.ts` — System prompt addendum for teammates
- `restored-src/src/utils/swarm/spawnUtils.ts` — CLI flag and env var inheritance for spawned teammates
- `restored-src/src/utils/swarm/spawnInProcess.ts` — In-process teammate spawning
- `restored-src/src/utils/swarm/inProcessRunner.ts` — In-process teammate execution loop
- `restored-src/src/utils/swarm/reconnection.ts` — Session reconnection for teammates
- `restored-src/src/utils/swarm/permissionSync.ts` — Permission request/response synchronization
- `restored-src/src/utils/swarm/leaderPermissionBridge.ts` — Bridge between REPL and in-process teammates
- `restored-src/src/utils/swarm/It2SetupPrompt.tsx` — iTerm2 setup UI component

### Backend Submodule (`backends/`)
- `restored-src/src/utils/swarm/backends/types.ts` — Backend type definitions
- `restored-src/src/utils/swarm/backends/detection.ts` — Backend environment detection
- `restored-src/src/utils/swarm/backends/registry.ts` — Backend registry
- `restored-src/src/utils/swarm/backends/ITermBackend.ts` — iTerm2 pane backend
- `restored-src/src/utils/swarm/backends/TmuxBackend.ts` — tmux pane backend
- `restored-src/src/utils/swarm/backends/InProcessBackend.ts` — In-process backend
- `restored-src/src/utils/swarm/backends/PaneBackendExecutor.ts` — Pane execution
- `restored-src/src/utils/swarm/backends/it2Setup.ts` — iTerm2 CLI setup
- `restored-src/src/utils/swarm/backends/teammateModeSnapshot.ts` — Mode snapshot

## Key Exports

### Constants (`constants.ts`)
- `TEAM_LEAD_NAME` (`'team-lead'`), `SWARM_SESSION_NAME`, `SWARM_VIEW_WINDOW_NAME`, `TMUX_COMMAND`, `HIDDEN_SESSION_NAME`
- `getSwarmSocketName()`: Generates unique socket name with PID for external swarm sessions
- `TEAMMATE_COMMAND_ENV_VAR`, `TEAMMATE_COLOR_ENV_VAR`, `PLAN_MODE_REQUIRED_ENV_VAR`: Environment variable names

### Team Helpers (`teamHelpers.ts`)

#### Types
- `TeamFile`: Full team configuration including members, hidden panes, allowed paths
- `TeamAllowedPath`: Directory permission rule with tool, agent, and timestamp
- `SpawnTeamOutput`, `CleanupOutput`: Operation result types

#### Functions
- `sanitizeName(name)`: Sanitizes names for tmux/file paths (alphanumeric + hyphens)
- `sanitizeAgentName(name)`: Sanitizes agent names (replaces @ with -)
- `getTeamDir(teamName)`, `getTeamFilePath(teamName)`: Team directory path helpers
- `readTeamFile(teamName)`, `readTeamFileAsync(teamName)`: Read team config (sync/async)
- `writeTeamFile(teamName, teamFile)`, `writeTeamFileAsync(teamName, teamFile)`: Write team config
- `removeTeammateFromTeamFile(teamName, identifier)`: Remove teammate by agentId or name
- `addHiddenPaneId(teamName, paneId)`, `removeHiddenPaneId(teamName, paneId)`: Manage hidden panes
- `removeMemberFromTeam(teamName, tmuxPaneId)`, `removeMemberByAgentId(teamName, agentId)`: Remove members
- `setMemberMode(teamName, memberName, mode)`, `setMultipleMemberModes(teamName, modeUpdates)`: Update permission modes
- `syncTeammateMode(mode, teamNameOverride?)`: Sync current teammate's mode to config
- `setMemberActive(teamName, memberName, isActive)`: Update active/idle status
- `registerTeamForSessionCleanup(teamName)`, `unregisterTeamForSessionCleanup(teamName)`: Session lifecycle tracking
- `cleanupSessionTeams()`: Clean up orphaned teams on exit
- `cleanupTeamDirectories(teamName)`: Remove team and task directories, destroy git worktrees
- `destroyWorktree(worktreePath)`: Safely remove git worktrees

### Spawn Utilities (`spawnUtils.ts`)
- `getTeammateCommand()`: Get command for spawning teammates (env override or current executable)
- `buildInheritedCliFlags(options?)`: Build CLI flags to propagate to teammates (permissions, model, settings, plugins, mode, chrome)
- `buildInheritedEnvVars()`: Build environment variables to forward (API provider, proxy, config dir, CCR marker)

### In-Process Spawning (`spawnInProcess.ts`)
- `spawnInProcessTeammate(config, context)`: Creates and registers an in-process teammate task with AsyncLocalStorage context isolation
- `killInProcessTeammate(taskId, setAppState)`: Aborts and cleans up an in-process teammate

### In-Process Runner (`inProcessRunner.ts`)
- `runInProcessTeammate(config)`: Main execution loop for in-process teammates. Wraps `runAgent()` with AsyncLocalStorage isolation, progress tracking, mailbox-based communication, plan mode support, and idle notification
- `createInProcessCanUseTool(identity, abortController, onPermissionWaitMs?)`: Creates a `canUseTool` function that resolves permissions via the leader's ToolUseConfirm dialog (with worker badge) or falls back to mailbox polling

### Permission Sync (`permissionSync.ts`)
- `SwarmPermissionRequestSchema`: Zod schema for permission requests
- `createPermissionRequest(params)`: Create a new permission request
- `writePermissionRequest(request)`: Write request to pending directory with file locking
- `readPendingPermissions(teamName?)`: Read all pending requests for leader
- `readResolvedPermission(requestId, teamName?)`: Check if request has been resolved
- `resolvePermission(requestId, resolution, teamName?)`: Resolve a request (move pending → resolved)
- `cleanupOldResolutions(teamName?, maxAgeMs?)`: Clean up old resolved files
- `pollForResponse(requestId, agentName?, teamName?)`: Worker-side polling for response
- `isTeamLeader(teamName?)`, `isSwarmWorker()`: Role detection
- `sendPermissionRequestViaMailbox(request)`: Send request via mailbox (new approach)
- `sendPermissionResponseViaMailbox(workerName, resolution, requestId, teamName?)`: Send response via mailbox
- `sendSandboxPermissionRequestViaMailbox(host, requestId, teamName?)`: Sandbox network access request
- `sendSandboxPermissionResponseViaMailbox(workerName, requestId, host, allow, teamName?)`: Sandbox network access response

### Leader Permission Bridge (`leaderPermissionBridge.ts`)
- `registerLeaderToolUseConfirmQueue(setter)`, `getLeaderToolUseConfirmQueue()`: Bridge for REPL's queue setter
- `registerLeaderSetToolPermissionContext(setter)`, `getLeaderSetToolPermissionContext()`: Bridge for permission context setter

### Teammate Layout Manager (`teammateLayoutManager.ts`)
- `assignTeammateColor(teammateId)`: Assign unique color (round-robin)
- `getTeammateColor(teammateId)`: Get assigned color
- `clearTeammateColors()`: Reset all color assignments
- `isInsideTmux()`: Check if running inside tmux
- `createTeammatePaneInSwarmView(teammateName, teammateColor)`: Create pane using detected backend
- `enablePaneBorderStatus(windowTarget?, useSwarmSocket?)`: Enable pane titles
- `sendCommandToPane(paneId, command, useSwarmSocket?)`: Send command to specific pane

### Reconnection (`reconnection.ts`)
- `computeInitialTeamContext()`: Compute teamContext for AppState from CLI args (sync, before first render)
- `initializeTeammateContextFromSession(setAppState, teamName, agentName)`: Initialize context from resumed session

### Teammate Init (`teammateInit.ts`)
- `initializeTeammateHooks(setAppState, sessionId, teamInfo)`: Register Stop hook for idle notification and apply team-wide allowed paths

### Teammate Prompt Addendum (`teammatePromptAddendum.ts`)
- `TEAMMATE_SYSTEM_PROMPT_ADDendum`: System prompt text appended to teammates explaining communication via SendMessage tool

### Teammate Model (`teammateModel.ts`)
- `getHardcodedTeammateModelFallback()`: Returns default model (Opus 4.6) with provider awareness

### iTerm2 Setup Prompt (`It2SetupPrompt.tsx`)
- `It2SetupPrompt`: React component guiding users through iTerm2 `it2` CLI installation with steps: install → API instructions → verify → success/failure

## Dependencies

- `@anthropic-ai/sandbox-runtime` — Sandbox runtime (adapter)
- `chokidar`, `zod/v4`, `lodash-es` — Third-party libraries
- `../teammate.js`, `../teammateMailbox.js`, `../teammateContext.js` — Teammate core utilities
- `../permissions/` — Permission system
- `../telemetry/perfettoTracing.js` — Performance tracing
- `../../tasks/` — Task system

## Design Notes

- **Dual spawning**: Teammates can run as separate processes (tmux/iTerm2 panes) or in-process (AsyncLocalStorage isolation). In-process teammates share the same process but have independent identity, abort controllers, and conversation context.
- **Mailbox communication**: Teammates communicate via file-based mailboxes. The leader polls for messages; teammates write to the leader's mailbox and poll their own for responses.
- **Permission forwarding**: When a teammate needs permission for a tool use, it forwards the request to the leader who displays the UI. Two mechanisms exist: file-based (pending/resolved directories with locking) and mailbox-based (newer, replaces file-based).
- **Backend abstraction**: The backend system (tmux, iTerm2, in-process) is abstracted behind a registry with auto-detection, allowing the same high-level code to work across different terminal environments.
- **Session cleanup**: Teams created during a session are tracked and cleaned up on exit, including killing orphaned panes, removing directories, and destroying git worktrees.
- **Color assignment**: Teammates get unique colors from a palette in round-robin order for visual distinction in the UI.
