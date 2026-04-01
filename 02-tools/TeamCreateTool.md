# TeamCreateTool

## Purpose

TeamCreateTool creates a new multi-agent swarm team for coordinating parallel work. It establishes both a team configuration file and a corresponding task list directory, enabling the team lead to spawn teammates, assign tasks, and manage collaborative work. The tool is only available when agent swarms are enabled.

## Location

- `restored-src/src/tools/TeamCreateTool/TeamCreateTool.ts` — Main tool definition (241 lines)
- `restored-src/src/tools/TeamCreateTool/prompt.ts` — Tool prompt with team workflow instructions (114 lines)
- `restored-src/src/tools/TeamCreateTool/constants.ts` — Tool name constant
- `restored-src/src/tools/TeamCreateTool/UI.tsx` — UI rendering for tool use messages

## Key Exports

| Export | Description |
|--------|-------------|
| `TeamCreateTool` | The complete tool definition built via `buildTool()` |
| `Input` | Zod-inferred input type: `{ team_name, description?, agent_type? }` |
| `Output` | Output type: `{ team_name, team_file_path, lead_agent_id }` |

## Input/Output Schemas

### Input Schema

```typescript
{
  team_name: string,       // Required: name for the new team
  description?: string,    // Optional: team description/purpose
  agent_type?: string,     // Optional: type/role of the team lead
}
```

### Output Schema

```typescript
{
  team_name: string,         // The final team name (may differ from input if name existed)
  team_file_path: string,    // Path to the team config file
  lead_agent_id: string,     // Deterministic agent ID for the team lead
}
```

## Team Creation Process

### Execution Pipeline

```
1. INPUT RECEIVED
   - Model returns TeamCreate tool_use with { team_name, description?, agent_type? }

2. VALIDATION
   - validateInput(): ensures team_name is non-empty

3. PRE-CHECKS
   - Verify not already leading a team (one team per leader)
   - Check if team name already exists; if so, generate unique name via generateWordSlug()

4. TEAM FILE CREATION
   - Generate deterministic lead agent ID: formatAgentId(TEAM_LEAD_NAME, teamName)
   - Build TeamFile object with lead member, session ID, model, cwd
   - Write team file to ~/.claude/teams/{team-name}/config.json
   - Register team for session-end cleanup

5. TASK LIST SETUP
   - Reset task list (numbering starts at 1)
   - Create task directory at ~/.claude/tasks/{team-name}/
   - Set leader team name so getTaskListId() returns correct directory

6. APP STATE UPDATE
   - Update AppState.teamContext with team name, path, lead ID, and teammate map
   - Assign teammate color via assignTeammateColor()

7. ANALYTICS
   - Log 'tengu_team_created' event with team metadata

8. RETURN
   - Return { team_name, team_file_path, lead_agent_id }
```

### Unique Name Generation

If the requested team name already exists, `generateUniqueTeamName()` generates a new unique name using `generateWordSlug()` instead of failing. This prevents naming collisions while allowing the user to proceed.

## Team File Structure

The team file (`~/.claude/teams/{team-name}/config.json`) contains:

```typescript
{
  name: string,           // Team name
  description?: string,   // Team description
  createdAt: number,      // Timestamp
  leadAgentId: string,    // Deterministic lead agent ID
  leadSessionId: string,  // Session ID for team discovery
  members: [{
    agentId: string,      // Unique agent identifier
    name: string,         // Human-readable name (e.g., "team-lead")
    agentType: string,    // Role/type of the agent
    model: string,        // Model being used
    joinedAt: number,     // Timestamp
    tmuxPaneId: string,   // tmux pane (empty for lead)
    cwd: string,          // Current working directory
    subscriptions: [],    // Event subscriptions
  }]
}
```

## Team Workflow

The tool prompt defines a 7-step team workflow:

1. **Create a team** with TeamCreate — creates both team and task list
2. **Create tasks** using Task tools — automatically use the team's task list
3. **Spawn teammates** using Agent tool with `team_name` and `name` parameters
4. **Assign tasks** using TaskUpdate with `owner` parameter
5. **Teammates work** on assigned tasks and mark completed via TaskUpdate
6. **Teammates go idle** between turns (normal behavior, not an error)
7. **Shutdown team** via SendMessage with `message: {type: "shutdown_request"}`

## Teammate Agent Types

When spawning teammates, the agent type determines available tools:

- **Read-only agents** (Explore, Plan) — cannot edit/write files; only research/search/planning
- **Full-capability agents** (general-purpose) — access to all tools including file editing and bash
- **Custom agents** (from `.claude/agents/`) — may have custom tool restrictions

## Task List Coordination

Teams share a task list at `~/.claude/tasks/{team-name}/`:

1. Teammates check TaskList periodically, especially after completing tasks
2. Claim unassigned, unblocked tasks with TaskUpdate (set `owner`)
3. Prefer tasks in ID order (lowest first) for context setup
4. Create new tasks with TaskCreate when identifying additional work
5. Mark tasks completed with TaskUpdate
6. If all tasks blocked, notify team lead

## Communication Rules

- Messages from teammates are **automatically delivered** — no manual inbox checking needed
- Idle notifications are automatic and normal — do not treat idle as an error
- Always refer to teammates by **name** (not UUID) for messaging and task assignment
- Do NOT send structured JSON status messages — use plain text and TaskUpdate
- Do not use terminal tools to view team activity — always use SendMessage

## Constraints

- Only one team per leader at a time
- Team name must be non-empty
- Team is registered for automatic cleanup on session end
- Task list numbering resets to 1 for each new team
- Team lead is not considered a "teammate" (isTeammate() returns false)

## Dependencies

| Module | Purpose |
|--------|---------|
| `utils/swarm/teamHelpers.ts` | Team file read/write, path resolution, cleanup registration |
| `utils/swarm/teammateLayoutManager.ts` | Teammate color assignment |
| `utils/tasks.ts` | Task list reset, directory creation, leader team name |
| `utils/agentId.ts` | Agent ID formatting |
| `utils/model/model.ts` | Model parsing for team lead |
| `utils/cwd.ts` | Current working directory |
| `utils/words.ts` | Unique word slug generation |
| `services/analytics/` | Event logging |
