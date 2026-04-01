# AgentTool

## Purpose

The AgentTool is the primary mechanism for spawning sub-agents within Claude Code. It enables delegation of complex, multi-step tasks to specialized agents that run either synchronously (blocking the parent) or asynchronously (in the background). The tool supports built-in agents, custom user/project agents, plugin agents, and an experimental fork subagent mode that inherits full parent context.

## Location

`restored-src/src/tools/AgentTool/AgentTool.tsx`

## Key Exports

### Functions

- `AgentTool`: The main tool definition built via `buildTool()` — handles prompt generation, input validation, and agent spawning logic
- `inputSchema`: Zod schema defining the tool's input parameters (description, prompt, subagent_type, model, run_in_background, name, team_name, mode, isolation, cwd)
- `outputSchema`: Zod schema defining the tool's output (completed or async_launched status)

### Types

- `AgentToolInput`: Input type combining base schema with multi-agent parameters (name, team_name, mode, isolation, cwd)
- `Progress`: Union of `AgentToolProgress` and `ShellProgress` for progress event forwarding
- `RemoteLaunchedOutput`: Output type for remote-launched agents (ant-only)

### Constants

- `AGENT_TOOL_NAME`: `'Agent'` — the current tool name
- `LEGACY_AGENT_TOOL_NAME`: `'Task'` — legacy wire name for backward compatibility
- `ONE_SHOT_BUILTIN_AGENT_TYPES`: `Set(['Explore', 'Plan'])` — agents that run once and return a report

## Dependencies

### Internal Dependencies

- `runAgent.ts` — Core agent execution engine (async generator yielding messages)
- `loadAgentsDir.ts` — Agent definition loading, parsing, and type system
- `agentColorManager.ts` — Color assignment and theme mapping for agents
- `forkSubagent.ts` — Fork subagent experiment (context inheritance)
- `agentToolUtils.ts` — Shared utilities: tool resolution, finalization, lifecycle, classification
- `prompt.ts` — Tool prompt generation for the LLM
- `UI.tsx` — React components for rendering agent tool use/results in terminal
- `LocalAgentTask/LocalAgentTask.tsx` — Task registration, progress tracking, notifications
- `RemoteAgentTask/RemoteAgentTask.tsx` — Remote agent task registration (ant-only)

### External Dependencies

- `zod/v4` — Schema validation
- `react` — UI component rendering
- `bun:bundle` — Feature flag gating

## Implementation Details

### Sub-Agent Spawning

The AgentTool `call()` method orchestrates agent spawning through several decision branches:

#### 1. Teammate Spawn (Multi-Agent Teams)

When `team_name` and `name` are both provided, the tool delegates to `spawnTeammate()` instead of creating a subagent. This creates a separate tmux split-pane agent that is addressable via `SendMessage({to: name})`.

```
team_name + name → spawnTeammate() → teammate_spawned output
```

Validation guards:
- Teammates cannot spawn other teammates (flat roster)
- In-process teammates cannot spawn background agents

#### 2. Fork Subagent Path

When `subagent_type` is omitted AND the `FORK_SUBAGENT` feature is enabled (and not in coordinator or non-interactive mode), the tool takes the fork path:

```
!subagent_type + fork enabled → FORK_AGENT → inherits parent context
```

Key characteristics:
- Child inherits the parent's full conversation context and system prompt
- Uses `useExactTools: true` for cache-identical API request prefixes
- `permissionMode: 'bubble'` surfaces permission prompts to parent terminal
- `model: 'inherit'` keeps the parent's model for context length parity
- All spawns run async for unified `<task-notification>` interaction
- Recursive fork guard: fork children cannot spawn further forks (detected via `FORK_BOILERPLATE_TAG` in messages)

Fork message construction (`buildForkedMessages`):
1. Keeps the full parent assistant message (all tool_use blocks, thinking, text)
2. Builds a single user message with placeholder tool_results for every tool_use
3. Appends a per-child directive with strict rules (no spawning, no editorializing, report format)

#### 3. Normal Subagent Path

When `subagent_type` is specified (or defaults to `general-purpose` when fork is off):

```
subagent_type → find agent definition → validate MCP requirements → run agent
```

Steps:
1. Look up agent definition from `activeAgents` list
2. Check permission rules (Agent(agentType) deny rules)
3. Validate required MCP servers are available (with wait-for-pending logic, up to 30s)
4. Initialize agent color if defined
5. Build system prompt and prompt messages
6. Determine sync vs async execution
7. Execute via `runAgent()` async generator

#### 4. Remote Isolation (Ant-Only)

When `isolation: 'remote'` is set:
1. Check remote agent eligibility
2. Delegate to CCR via `teleportToRemote()`
3. Register remote agent task
4. Return `remote_launched` output with session URL

### Agent Configuration

#### Input Schema Fields

| Field | Type | Description |
|-------|------|-------------|
| `description` | `string` | Short (3-5 word) task summary |
| `prompt` | `string` | The task for the agent to perform |
| `subagent_type` | `string?` | Agent type selector (defaults to general-purpose or fork) |
| `model` | `'sonnet' \| 'opus' \| 'haiku'?` | Optional model override |
| `run_in_background` | `boolean?` | Run agent asynchronously |
| `name` | `string?` | Addressable name via SendMessage |
| `team_name` | `string?` | Team name for spawning teammates |
| `mode` | `PermissionMode?` | Permission mode for spawned teammate |
| `isolation` | `'worktree' \| 'remote'?` | Isolation mode |
| `cwd` | `string?` | Working directory override (KAIROS only) |

#### Schema Gating

The schema dynamically omits fields based on feature flags:
- `cwd`: Omitted when `KAIROS` feature is off
- `run_in_background`: Omitted when background tasks are disabled (`CLAUDE_CODE_DISABLE_BACKGROUND_TASKS`) or fork subagent is enabled (all spawns are async in fork mode)

### Agent Lifecycle

#### Synchronous Execution

Sync agents block the parent turn until completion:

1. Register as foreground task (`registerAgentForeground`)
2. Create progress tracker and activity resolver
3. Yield initial progress message
4. Iterate over `runAgent()` async generator
5. Race between next message and background signal (for mid-execution backgrounding)
6. On background signal: clean up foreground iterator, continue in background
7. On completion: finalize result, emit notification
8. Cleanup: worktree, MCP servers, hooks, file state cache, todos, shell tasks

#### Asynchronous Execution

Async agents run independently of the parent turn:

1. Register async agent (`registerAsyncAgent`) with unlinked abort controller
2. Register name in `agentNameRegistry` for SendMessage routing
3. Launch `runAsyncAgentLifecycle()` in detached closure (`void`)
4. Return `async_launched` output immediately with agentId and outputFile path
5. Parent continues; notification fires on completion

#### Background Mid-Execution Transition

A sync agent can be backgrounded mid-execution:

1. `registerAgentForeground` sets up `backgroundSignal` promise
2. Auto-background after `getAutoBackgroundMs()` (120s when enabled via env or GrowthBook)
3. When signal fires: clean up foreground iterator (1s timeout), re-iterate in background mode
4. Continue with independent summarization and progress tracking

#### Cleanup (Both Paths)

The `finally` block in `runAgent()` handles:
- MCP server cleanup (agent-specific servers only)
- Session hook cleanup (`clearSessionHooks`)
- Prompt cache tracking cleanup
- File state cache release
- Fork context messages release
- Perfetto agent unregistration
- Transcript subdir mapping cleanup
- Todos entry removal
- Shell task killing for agent
- Monitor MCP task killing (when enabled)

Worktree cleanup:
- Hook-based worktrees: always kept (can't detect VCS changes)
- Non-hook worktrees: check for changes since head commit
  - No changes: remove worktree and clear metadata
  - Has changes: keep worktree, return path and branch in result

### Built-In vs Custom Agents

#### Agent Type System

```
AgentDefinition = BuiltInAgentDefinition | CustomAgentDefinition | PluginAgentDefinition
```

| Type | Source | Prompt Storage | Loading |
|------|--------|---------------|---------|
| Built-in | `'built-in'` | Dynamic `getSystemPrompt()` function | Hardcoded imports |
| Custom | `'userSettings'`, `'projectSettings'`, `'policySettings'`, `'flagSettings'` | Closure over parsed content | Markdown files in `.claude/agents/` |
| Plugin | `'plugin'` | Closure over plugin content | Plugin system |

#### Built-In Agents

| Agent | Type | Model | Tools | Key Characteristics |
|-------|------|-------|-------|-------------------|
| General Purpose | `general-purpose` | Default subagent | `['*']` | Fallback agent, all tools |
| Explore | `Explore` | `haiku` (external) / `inherit` (ant) | Read-only (no Agent, Edit, Write) | Fast file search, `omitClaudeMd: true` |
| Plan | `Plan` | `inherit` | Read-only (no Agent, Edit, Write) | Software architect, `omitClaudeMd: true` |
| Statusline Setup | `statusline-setup` | `sonnet` | `['Read', 'Edit']` | Color: orange, configures status line |
| Claude Code Guide | `claude-code-guide` | `haiku` | Bash/Read/WebFetch/WebSearch | Dynamic prompt with current config context |
| Verification | `verification` | `inherit` | Read-only + Bash (no Agent, Edit, Write) | Color: red, `background: true`, adversarial testing |

Built-in agent loading (`builtInAgents.ts`):
- `GENERAL_PURPOSE_AGENT` and `STATUSLINE_SETUP_AGENT` always included
- `EXPLORE_AGENT` and `PLAN_AGENT` gated by `BUILTIN_EXPLORE_PLAN_AGENTS` feature + `tengu_amber_stoat` flag
- `CLAUDE_CODE_GUIDE_AGENT` included for non-SDK entrypoints
- `VERIFICATION_AGENT` gated by `VERIFICATION_AGENT` feature + `tengu_hive_evidence` flag
- Coordinator mode uses `getCoordinatorAgents()` instead

#### Custom Agent Loading

`getAgentDefinitionsWithOverrides()` (memoized by cwd):

1. Load markdown files from `.claude/agents/` directories (user, project, policy, local, flag sources)
2. Parse each file: extract frontmatter (name, description, tools, model, color, etc.) and body content
3. Validate required fields (name, description)
4. Build `CustomAgentDefinition` with `getSystemPrompt` closure
5. Load plugin agents concurrently
6. Merge all sources: `[builtInAgents, pluginAgents, customAgents]`
7. Deduplicate by agentType (priority: built-in → plugin → user → project → flag → managed)
8. Initialize colors for all active agents
9. Initialize memory snapshots for agents with `memory: 'user'`

#### Agent Definition Fields

```typescript
type BaseAgentDefinition = {
  agentType: string           // Unique identifier
  whenToUse: string           // Description shown in agent list
  tools?: string[]            // Allowed tools ('*' for all)
  disallowedTools?: string[]  // Explicitly blocked tools
  skills?: string[]           // Skill names to preload
  mcpServers?: AgentMcpServerSpec[]  // Agent-specific MCP servers
  hooks?: HooksSettings       // Session-scoped hooks
  color?: AgentColorName      // UI color (red, blue, green, yellow, purple, orange, pink, cyan)
  model?: string              // Model override or 'inherit'
  effort?: EffortValue        // Effort level
  permissionMode?: PermissionMode  // Permission mode override
  maxTurns?: number           // Maximum agentic turns
  background?: boolean        // Always run as background
  isolation?: 'worktree' | 'remote'  // Isolation mode
  memory?: AgentMemoryScope   // Persistent memory scope (user, project, local)
  omitClaudeMd?: boolean      // Exclude CLAUDE.md from context
  initialPrompt?: string      // Prepended to first user turn
  requiredMcpServers?: string[]  // MCP server patterns required for availability
}
```

### Agent Communication

#### Parent-to-Child

The parent communicates with the spawned agent through:

1. **Initial prompt**: The `prompt` parameter passed to AgentTool becomes the user message
2. **System prompt**: Built from agent definition's `getSystemPrompt()` + environment details
3. **Fork context**: When fork path, the full parent conversation is inherited as context messages

#### Child-to-Parent

The child communicates back through:

1. **Sync result**: `finalizeAgentTool()` extracts text content, tool use count, tokens, duration, and usage stats
2. **Async notification**: `enqueueAgentNotification()` fires when the background agent completes
3. **Progress events**: `onProgress` callback yields `agent_progress` messages during execution
4. **SendMessage**: Parent can send follow-up messages to a running agent via its name or ID

#### Handoff Classification

When `TRANSCRIPT_CLASSIFIER` feature is enabled and mode is `'auto'`:
- The agent's transcript is classified before handoff
- Flags potentially dangerous actions with a security warning
- Warning is prepended to the final message returned to parent

### Agent State Management

#### Progress Tracking

`createProgressTracker()` tracks:
- Token count (from usage data)
- Tool use count
- Last activity description (resolved from tool names)

Progress updates flow through:
1. `updateProgressFromMessage()` — extracts progress data from each yielded message
2. `updateAsyncAgentProgress()` — updates task state in AppState
3. `emitTaskProgress()` — emits SDK progress event with taskId, toolUseId, description

#### Agent Identity

Each agent gets a unique `AgentId` (UUID) created early via `createAgentId()`:
- Used for worktree slug generation
- Used for transcript recording (`recordSidechainTranscript`)
- Used for metadata persistence (`writeAgentMetadata`)
- Used for analytics attribution (`runWithAgentContext`)
- Registered in `agentNameRegistry` when `name` parameter is provided

#### Transcript Recording

Agent conversations are recorded to disk:
- Initial messages recorded before query loop starts
- Each new message recorded incrementally with correct parent UUID (O(1) per message)
- Metadata recorded: agentType, worktreePath, description
- Stored in `subagents/` directory (optionally grouped by `transcriptSubdir`)

#### Resume Support

`resumeAgentBackground()` reconstructs a previously spawned agent:
1. Load transcript and metadata from disk
2. Filter messages (remove whitespace-only, orphaned thinking, unresolved tool uses)
3. Reconstruct content replacement state for tool result stability
4. Restore worktree path (with existence check)
5. Re-select agent definition (or fall back to general-purpose)
6. Re-run through `runAsyncAgentLifecycle()`

### Color Management

`agentColorManager.ts` manages UI colors for agent identification:

#### Color Palette

8 predefined colors: `red`, `blue`, `green`, `yellow`, `purple`, `orange`, `pink`, `cyan`

#### Theme Mapping

Each agent color maps to a theme-specific color:
```
red → red_FOR_SUBAGENTS_ONLY
blue → blue_FOR_SUBAGENTS_ONLY
...
```

#### API

- `setAgentColor(agentType, color)`: Assigns a color to an agent type in the global color map
- `getAgentColor(agentType)`: Retrieves the theme color for an agent type (returns `undefined` for `general-purpose`)

#### Color Assignment Flow

1. Agent definition declares `color` field (e.g., Verification Agent: `'red'`, Statusline Setup: `'orange'`)
2. On agent spawn: `setAgentColor()` registers the color
3. On agent list load: colors initialized for all active agents
4. UI uses `userFacingNameBackgroundColor()` to apply color to agent tags in grouped display

### Agent Loading from Directory

#### Markdown File Format

Custom agents are defined in `.claude/agents/*.md` files with YAML frontmatter:

```markdown
---
name: my-agent
description: What this agent does
tools: [Read, Bash, Glob]
disallowedTools: [FileWrite]
model: sonnet
color: blue
permissionMode: acceptEdits
maxTurns: 50
memory: project
background: false
isolation: worktree
effort: high
hooks:
  SubagentStop: [notify.sh]
mcpServers:
  - slack
  - { my-server: { command: "...", args: [] } }
skills: [my-skill]
initialPrompt: Do this first...
---

Agent system prompt content goes here...
```

#### Parsing Pipeline

`parseAgentFromMarkdown()`:
1. Validate required fields: `name` (string), `description` (string)
2. Unescape newlines in description (`\\n` → `\n`)
3. Parse optional fields with validation:
   - `color`: Must be in `AGENT_COLORS` list
   - `model`: String, `'inherit'` normalized to lowercase
   - `background`: Boolean or `'true'`/`'false'` string
   - `memory`: Must be `'user'`, `'project'`, or `'local'`
   - `isolation`: `'worktree'` (always) or `'remote'` (ant-only)
   - `effort`: String level or integer
   - `permissionMode`: Must be in `PERMISSION_MODES`
   - `maxTurns`: Positive integer
   - `tools`, `disallowedTools`: Parsed via `parseAgentToolsFromFrontmatter()`
   - `skills`: Parsed via `parseSlashCommandToolsFromFrontmatter()`
   - `mcpServers`: Validated via `AgentMcpServerSpecSchema`
   - `hooks`: Validated via `HooksSchema`
4. Build `getSystemPrompt` closure: returns content + memory prompt if memory enabled
5. Return `CustomAgentDefinition` or `null` (silently skips non-agent markdown)

#### JSON Agent Format

Agents can also be defined in JSON settings:

```json
{
  "my-agent": {
    "description": "What this agent does",
    "prompt": "System prompt content",
    "tools": ["Read", "Bash"],
    "model": "sonnet",
    "memory": "project"
  }
}
```

Parsed via `parseAgentFromJson()` using `AgentJsonSchema` validation.

#### MCP Server Requirements

Agents can declare required MCP servers:
- `requiredMcpServers`: Array of server name patterns
- Checked at spawn time: each pattern must match at least one available server
- If required servers are pending (connecting), waits up to 30s (polling every 500ms)
- Agents with unmet requirements are filtered out from the available agent list

#### Agent Filtering

Agents are filtered before being presented to the LLM:
1. **MCP requirements**: `filterAgentsByMcpRequirements()` — removes agents whose required servers aren't available
2. **Permission rules**: `filterDeniedAgents()` — removes agents denied by `Agent(agentType)` rules
3. **Allowed types**: When `Agent(x,y)` syntax is used, restricts to specified types

### Agent Memory

#### Memory Scopes

| Scope | Directory | Use Case |
|-------|-----------|----------|
| `user` | `<memoryBase>/agent-memory/<agentType>/` | Cross-project learnings |
| `project` | `<cwd>/.claude/agent-memory/<agentType>/` | Team-shared project memory |
| `local` | `<cwd>/.claude/agent-memory-local/<agentType>/` | Machine-specific memory |

#### Memory Integration

When `memory` is enabled in agent definition:
1. `loadAgentMemoryPrompt()` builds a memory prompt with scope-specific guidelines
2. Memory prompt is appended to the agent's system prompt (`systemPrompt + '\n\n' + memoryPrompt`)
3. Write/Edit/Read tools are auto-injected if memory is enabled (for memory file access)

#### Snapshot System

Project-level memory snapshots allow sharing agent memory across team members:

- Snapshots stored in `.claude/agent-memory-snapshots/<agentType>/`
- `checkAgentMemorySnapshot()`: Compares snapshot timestamp with local sync state
  - `'initialize'`: No local memory exists → copy snapshot to local
  - `'prompt-update'`: Newer snapshot exists → notify for update
  - `'none'`: Local is up to date
- `initializeFromSnapshot()`: First-time setup, copies snapshot files and records sync metadata
- `replaceFromSnapshot()`: Overwrites local memory with snapshot (removes existing .md files first)

### Tool Resolution

`resolveAgentTools()` determines which tools an agent has access to:

1. **Filter available tools** via `filterToolsForAgent()`:
   - MCP tools always allowed
   - ExitPlanMode allowed for plan mode agents
   - `ALL_AGENT_DISALLOWED_TOOLS` blocked for all agents (Agent, TaskCreate, etc.)
   - `CUSTOM_AGENT_DISALLOWED_TOOLS` blocked for custom agents only
   - `ASYNC_AGENT_ALLOWED_TOOLS` restricts background agents (with teammate exceptions)

2. **Apply agent-specific restrictions**:
   - `tools` field: If undefined or `['*']`, all filtered tools are available
   - Otherwise, only explicitly listed tools are included
   - `disallowedTools` field: Explicitly removes listed tools
   - Agent tool spec with rule content (`Agent(worker, researcher)`) extracts `allowedAgentTypes`

3. **Deduplicate** resolved tools by name

### UI Rendering

`UI.tsx` provides React components for terminal display:

#### Render Functions

- `renderToolUseMessage`: Shows agent description during initialization
- `renderToolUseProgressMessage`: Shows live progress (condensed mode for small terminals, full mode with search/read grouping)
- `renderToolResultMessage`: Shows completion summary with stats (tool uses, tokens, duration)
- `renderToolUseRejectedMessage`: Shows rejection with progress history
- `renderToolUseErrorMessage`: Shows error with progress history
- `renderGroupedAgentToolUse`: Groups multiple parallel agent spawns into a single display

#### Progress Message Processing

For ant builds, consecutive search/read operations are grouped into summaries:
- `processProgressMessages()`: Groups consecutive Glob/Grep/Read operations
- Displays summary text like "3 searches, 5 reads..." instead of individual entries
- Active group (in-progress) vs completed group distinction

#### Agent Stats Display

Each agent shows:
- Agent type (or `@name` for teammate spawns)
- Description
- Tool use count
- Token count
- Last tool info (condensed search/read summary or tool name)
- Async status indicator
- Color coding based on agent definition

## Configuration

### Environment Variables

| Variable | Purpose |
|----------|---------|
| `CLAUDE_CODE_DISABLE_BACKGROUND_TASKS` | Disables async agent execution |
| `CLAUDE_AUTO_BACKGROUND_TASKS` | Enables auto-background after 120s |
| `CLAUDE_CODE_COORDINATOR_MODE` | Enables coordinator mode (different agent set) |
| `CLAUDE_CODE_SIMPLE` | Skips custom agent loading, returns built-ins only |
| `CLAUDE_AGENT_SDK_DISABLE_BUILTIN_AGENTS` | Disables built-in agents in SDK mode |
| `CLAUDE_CODE_AGENT_LIST_IN_MESSAGES` | Controls agent list placement (attachment vs inline) |
| `CLAUDE_CODE_REMOTE_MEMORY_DIR` | Redirects local memory to remote mount |
| `CLAUDE_COWORK_MEMORY_EXTRA_GUIDELINES` | Additional memory guidelines |
| `USER_TYPE=ant` | Enables ant-only features (remote isolation, embedded search) |

### Feature Flags

| Flag | Purpose |
|------|---------|
| `FORK_SUBAGENT` | Enables fork subagent experiment |
| `KAIROS` | Enables assistant mode (forces all agents async) |
| `COORDINATOR_MODE` | Enables coordinator mode |
| `TRANSCRIPT_CLASSIFIER` | Enables handoff safety classification |
| `VERIFICATION_AGENT` | Enables verification agent |
| `BUILTIN_EXPLORE_PLAN_AGENTS` | Enables Explore/Plan agents |
| `AGENT_MEMORY_SNAPSHOT` | Enables memory snapshot system |
| `PROACTIVE` / `KAIROS` | Enables proactive module integration |

## Error Handling

### Spawn Errors

- **Agent type not found**: Lists available agent types in error message
- **Agent denied by permission**: Shows the deny rule and its source
- **MCP servers unavailable**: Lists missing servers and available servers with tools
- **Teammate restrictions**: Clear error messages for invalid teammate spawn attempts
- **Recursive fork guard**: "Fork is not available inside a forked worker"
- **Team access**: "Agent Teams is not yet available on your plan"

### Runtime Errors

- **No assistant messages**: Throws if agent produces no assistant messages
- **Abort handling**: `AbortError` triggers killed notification with partial result
- **General errors**: Caught, failed notification with error message
- **Worktree fallback**: If worktree no longer exists on resume, falls back to parent cwd

## Data Flow

```
User/Parent Agent
    │
    ▼
AgentTool.call()
    │
    ├── [team_name + name] ──→ spawnTeammate() ──→ teammate_spawned
    │
    ├── [!subagent_type + fork] ──→ FORK_AGENT
    │   ├── buildForkedMessages() — clone parent context
    │   ├── inherit parent system prompt
    │   └── runAgent(useExactTools: true)
    │
    ├── [subagent_type] ──→ find AgentDefinition
    │   ├── validate MCP requirements
    │   ├── build system prompt + memory
    │   ├── resolve tools (filter + allowlist)
    │   ├── create worktree (if isolation: worktree)
    │   │
    │   ├── [async] ──→ registerAsyncAgent()
    │   │   └── runAsyncAgentLifecycle()
    │   │       ├── runAgent() generator
    │   │       ├── progress tracking
    │   │       ├── summarization
    │   │       └── enqueueAgentNotification()
    │   │
    │   └── [sync] ──→ registerAgentForeground()
    │       ├── runAgent() generator
    │       ├── race with background signal
    │       ├── [backgrounded] → transition to async
    │       └── finalizeAgentTool()
    │
    └── [isolation: remote] ──→ teleportToRemote() ──→ remote_launched
```

## Integration Points

- **LocalAgentTask**: Task registration, progress tracking, kill management, notifications
- **SendMessageTool**: Inter-agent communication via agent name/ID registry
- **MCP Service**: Agent-specific MCP server connection and cleanup
- **Hooks System**: SubagentStart/SubagentStop hooks, frontmatter hook registration
- **Session Storage**: Transcript recording, metadata persistence, agent color map
- **Analytics**: Event logging for spawn, completion, termination, memory loading
- **Perfetto Tracing**: Agent hierarchy visualization in performance traces
- **SDK Event Queue**: Progress event emission for SDK consumers
- **Worktree System**: Git worktree creation, change detection, cleanup

## Related Modules

- [LocalAgentTask](../tasks/LocalAgentTask.md) — Task management for local agents
- [SendMessageTool](./SendMessageTool.md) — Inter-agent communication
- [runAgent](./AgentTool/runAgent.md) — Core agent execution engine
- [loadAgentsDir](./AgentTool/loadAgentsDir.md) — Agent definition loading
- [forkSubagent](./AgentTool/forkSubagent.md) — Fork subagent experiment
- [agentColorManager](./AgentTool/agentColorManager.md) — Color management
- [agentMemory](./AgentTool/agentMemory.md) — Persistent agent memory
- [agentToolUtils](./AgentTool/agentToolUtils.md) — Shared utilities
- [prompt](./AgentTool/prompt.md) — Tool prompt generation
- [UI](./AgentTool/UI.md) — Terminal UI rendering
