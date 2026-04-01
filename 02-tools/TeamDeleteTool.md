# TeamDeleteTool

## Purpose

TeamDeleteTool cleans up team and task directories when swarm work is complete. It removes the team configuration, task list, and clears the team context from the current session. The tool prevents deletion if active teammates remain, ensuring graceful shutdown before cleanup.

## Location

- `restored-src/src/tools/TeamDeleteTool/TeamDeleteTool.ts` — Main tool definition (140 lines)
- `restored-src/src/tools/TeamDeleteTool/prompt.ts` — Tool prompt (17 lines)
- `restored-src/src/tools/TeamDeleteTool/constants.ts` — Tool name constant
- `restored-src/src/tools/TeamDeleteTool/UI.tsx` — UI rendering for tool use/result messages

## Key Exports

| Export | Description |
|--------|-------------|
| `TeamDeleteTool` | The complete tool definition built via `buildTool()` |
| `Input` | Empty input schema — team name determined from session context |
| `Output` | Output type: `{ success, message, team_name? }` |

## Input/Output Schemas

### Input Schema

```typescript
{}  // No input parameters — team name is auto-detected from session
```

### Output Schema

```typescript
{
  success: boolean,       // Whether cleanup succeeded
  message: string,        // Human-readable result message
  team_name?: string,     // Name of the team that was cleaned up
}
```

## Cleanup Process

### Execution Pipeline

```
1. INPUT RECEIVED
   - Model returns TeamDelete tool_use with empty input

2. TEAM DETECTION
   - Read team name from AppState.teamContext.teamName

3. ACTIVE MEMBER CHECK
   - Read team config file
   - Filter out team lead — only count non-lead members
   - Filter out idle/dead members (isActive === false)
   - If active members remain: return failure with member names

4. DIRECTORY CLEANUP
   - Remove team directory (~/.claude/teams/{team-name}/)
   - Remove task directory (~/.claude/tasks/{team-name}/)
   - Unregister team from session-end cleanup (already cleaned)

5. STATE RESET
   - Clear teammate color assignments (clearTeammateColors)
   - Clear leader team name (clearLeaderTeamName)
   - Clear team context from AppState
   - Clear inbox messages

6. ANALYTICS
   - Log 'tengu_team_deleted' event

7. RETURN
   - Return { success, message, team_name }
```

## Active Member Protection

The tool will **fail** if the team still has active (non-idle) non-lead members:

```
{
  success: false,
  message: "Cannot cleanup team with N active member(s): name1, name2. Use requestShutdown to gracefully terminate teammates first.",
  team_name: "team-name"
}
```

This ensures teammates are gracefully terminated before their resources are cleaned up. Members with `isActive === false` (idle or crashed) are excluded from this check.

## Cleanup Operations

| Operation | Description |
|-----------|-------------|
| `cleanupTeamDirectories(teamName)` | Removes team and task directories from disk |
| `unregisterTeamForSessionCleanup(teamName)` | Prevents duplicate cleanup on session end |
| `clearTeammateColors()` | Resets color assignments for fresh start |
| `clearLeaderTeamName()` | Resets getTaskListId() to fall back to session ID |

## AppState Changes

After successful cleanup, the following AppState fields are cleared:

```typescript
{
  teamContext: undefined,     // No active team
  inbox: { messages: [] },    // Clear any queued messages
}
```

## When to Use

- All teammates have finished their work and shut down
- The swarm project is complete
- You want to clean up persistent team and task directories
- Starting a new team (must delete current team first)

## Constraints

- TeamDelete will fail if active members remain
- Teammates must be gracefully terminated first via SendMessage shutdown_request
- No input parameters — team name is determined from current session context
- UI suppresses cleanup result display (batched shutdown message covers it)

## Dependencies

| Module | Purpose |
|--------|---------|
| `utils/swarm/teamHelpers.ts` | Team file reading, directory cleanup, cleanup registration |
| `utils/swarm/teammateLayoutManager.ts` | Teammate color management |
| `utils/tasks.ts` | Leader team name clearing |
| `utils/swarm/constants.ts` | TEAM_LEAD_NAME constant |
| `services/analytics/` | Event logging |
