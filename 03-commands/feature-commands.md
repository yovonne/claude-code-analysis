# Feature Commands

## Purpose

Documents the CLI commands that manage Claude Code's extensibility features: agents, skills, memory, background tasks, and hooks. These commands provide interactive UIs for configuring and monitoring the components that extend Claude Code's capabilities.

---

## /agents (Agent Management)

### Purpose
Opens an interactive menu for managing agent configurations — custom agent definitions, permissions, and behavior settings.

### Location
`restored-src/src/commands/agents/index.ts`
`restored-src/src/commands/agents/agents.tsx`

### Command Definition
| Property | Value |
|----------|-------|
| Name | `agents` |
| Type | `local-jsx` |
| Description | Manage agent configurations |

### Key Exports

#### Functions
- `call(onDone, context)`: Entry point; renders `AgentsMenu` with available tools

### Dependencies

#### Internal
- `../../components/agents/AgentsMenu.js` — Interactive agent management UI
- `../../tools.js` — `getTools` for retrieving available tool list
- `../../Tool.js` — `ToolUseContext` type
- `../../types/command.js` — `LocalJSXCommandOnDone` type

#### External
- `react` — Component rendering

### Implementation Details

#### Core Logic
1. Retrieves current app state and extracts `toolPermissionContext`
2. Gets the full list of available tools via `getTools(permissionContext)`
3. Renders `AgentsMenu` component with tools and exit callback
4. The menu allows users to view, configure, and manage agent definitions

### Data Flow
```
User runs /agents
  → context.getAppState() → toolPermissionContext
  → getTools(permissionContext) → tool list
  → Render AgentsMenu(tools, onExit)
  → User interacts with agent management UI
  → User exits → onDone() closes
```

---

## /skills (Skill Listing)

### Purpose
Displays a menu of available skills (command skills) that can be invoked during a session.

### Location
`restored-src/src/commands/skills/index.ts`
`restored-src/src/commands/skills/skills.tsx`

### Command Definition
| Property | Value |
|----------|-------|
| Name | `skills` |
| Type | `local-jsx` |
| Description | List available skills |

### Key Exports

#### Functions
- `call(onDone, context)`: Entry point; renders `SkillsMenu` with available commands

### Dependencies

#### Internal
- `../../components/skills/SkillsMenu.js` — Skills listing UI component
- `../../commands.js` — `LocalJSXCommandContext` type
- `../../types/command.js` — `LocalJSXCommandOnDone` type

#### External
- `react` — Component rendering

### Implementation Details

#### Core Logic
1. Renders `SkillsMenu` component passing:
   - `onExit`: Callback to close the menu
   - `commands`: Available commands from `context.options.commands`
2. The menu displays all registered skills/command plugins available to the user

### Data Flow
```
User runs /skills
  → Render SkillsMenu(onExit, commands=context.options.commands)
  → User browses available skills
  → User exits → onDone() closes
```

---

## /memory (Memory File Editing)

### Purpose
Opens a dialog for selecting and editing Claude memory files (CLAUDE.md files). Memory files provide persistent context across sessions.

### Location
`restored-src/src/commands/memory/index.ts`
`restored-src/src/commands/memory/memory.tsx`

### Command Definition
| Property | Value |
|----------|-------|
| Name | `memory` |
| Type | `local-jsx` |
| Description | Edit Claude memory files |

### Key Exports

#### Functions
- `call(onDone)`: Entry point; clears caches, primes memory files, renders `MemoryCommand`

#### Components
- `MemoryCommand`: React component that presents the memory file selector and handles editing

### Dependencies

#### Internal
- `../../components/memory/MemoryFileSelector.js` — File selection UI
- `../../components/memory/MemoryUpdateNotification.js` — `getRelativeMemoryPath` for display
- `../../components/design-system/Dialog.js` — Dialog container
- `../../utils/claudemd.js` — `clearMemoryFileCaches`, `getMemoryFiles` for memory file management
- `../../utils/envUtils.js` — `getClaudeConfigHomeDir` for directory path
- `../../utils/promptEditor.js` — `editFileInEditor` for opening files in external editor
- `../../utils/errors.js` — `getErrnoCode` for error handling
- `../../utils/log.js` — `logError` for error logging
- `../../ink.js` — `Box`, `Text`, `Link` for UI rendering

#### External
- `fs/promises` — `mkdir`, `writeFile` for file creation
- `react` — Component rendering with `Suspense`

### Implementation Details

#### Core Logic
1. **Preparation**: Clears memory file caches and awaits `getMemoryFiles()` to prime data before rendering
2. **File Selection**: Renders `MemoryFileSelector` dialog listing available memory files
3. **File Handling** (on selection):
   - Creates the `claude` config directory if it doesn't exist (idempotent with `recursive: true`)
   - Creates the file if it doesn't exist (using `wx` flag to avoid overwriting existing content)
   - Opens the file in the user's configured external editor (`$VISUAL` or `$EDITOR`)
4. **Editor Detection**: Checks `$VISUAL` first, then `$EDITOR`, reports which editor is being used
5. **Completion**: Shows the relative path and editor hint, or error message on failure

#### Edge Cases
- File already exists: `wx` flag throws `EEXIST`, caught and ignored to preserve content
- Directory doesn't exist: Created automatically with `mkdir({ recursive: true })`
- Editor not configured: Falls back to system default editor
- Error during open: Logged via `logError` and reported to user

### Data Flow
```
User runs /memory
  → clearMemoryFileCaches() → await getMemoryFiles()
  → Render MemoryCommand dialog
  → User selects a memory file
  → Ensure directory exists (mkdir recursive)
  → Create file if needed (wx flag, ignore EEXIST)
  → editFileInEditor(memoryPath)
  → Detect editor ($VISUAL or $EDITOR)
  → Display: "Opened memory file at <relative-path>"
```

---

## /tasks (Background Task Management)

### Purpose
Opens a dialog for viewing and managing background tasks (bash processes running in the background).

### Location
`restored-src/src/commands/tasks/index.ts`
`restored-src/src/commands/tasks/tasks.tsx`

### Command Definition
| Property | Value |
|----------|-------|
| Name | `tasks` |
| Aliases | `bashes` |
| Type | `local-jsx` |
| Description | List and manage background tasks |

### Key Exports

#### Functions
- `call(onDone, context)`: Entry point; renders `BackgroundTasksDialog`

### Dependencies

#### Internal
- `../../components/tasks/BackgroundTasksDialog.js` — Background task management UI
- `../../commands.js` — `LocalJSXCommandContext` type
- `../../types/command.js` — `LocalJSXCommandOnDone` type

#### External
- `react` — Component rendering

### Implementation Details

#### Core Logic
1. Renders `BackgroundTasksDialog` component with:
   - `toolUseContext`: Full context for task operations
   - `onDone`: Callback to close the dialog
2. The dialog displays all running background tasks with their status, output, and management options (view output, stop, etc.)

### Data Flow
```
User runs /tasks (or /bashes)
  → Render BackgroundTasksDialog(toolUseContext, onDone)
  → User views/manages background tasks
  → User exits → onDone() closes
```

---

## /hooks (Hook Configuration)

### Purpose
Opens a menu for configuring hook behaviors — custom scripts that run in response to tool events (pre/post hooks).

### Location
`restored-src/src/commands/hooks/index.ts`
`restored-src/src/commands/hooks/hooks.tsx`

### Command Definition
| Property | Value |
|----------|-------|
| Name | `hooks` |
| Type | `local-jsx` |
| Description | View hook configurations for tool events |
| Immediate | `true` |

### Key Exports

#### Functions
- `call(onDone, context)`: Entry point; logs analytics event, renders `HooksConfigMenu`

### Dependencies

#### Internal
- `../../components/hooks/HooksConfigMenu.js` — Hook configuration UI
- `../../services/analytics/index.js` — `logEvent` for usage tracking
- `../../tools.js` — `getTools` for retrieving tool names
- `../../types/command.js` — `LocalJSXCommandCall` type

#### External
- `react` — Component rendering

### Implementation Details

#### Core Logic
1. Logs analytics event `tengu_hooks_command`
2. Retrieves app state and extracts `toolPermissionContext`
3. Gets list of all tool names via `getTools(permissionContext).map(tool => tool.name)`
4. Renders `HooksConfigMenu` with tool names and exit callback
5. The menu allows users to configure pre/post hooks for each tool event

### Data Flow
```
User runs /hooks
  → logEvent('tengu_hooks_command')
  → context.getAppState() → toolPermissionContext
  → getTools(permissionContext).map(tool => tool.name) → tool name list
  → Render HooksConfigMenu(toolNames, onExit)
  → User configures hooks for tool events
  → User exits → onDone() closes
```

---

## Feature Commands Overview

These five commands provide the primary interface for managing Claude Code's extensibility system:

```
┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐
│ /agents  │  │ /skills  │  │ /memory  │  │ /tasks   │  │ /hooks   │
│ (manage  │  │ (list    │  │ (edit    │  │ (manage  │  │ (config  │
│  agents) │  │  skills) │  │  memory) │  │  tasks)  │  │  hooks)  │
└──────────┘  └──────────┘  └──────────┘  └──────────┘  └──────────┘
      │             │             │             │             │
      └─────────────┴─────────────┴─────────────┴─────────────┘
                              │
                    ┌─────────┴─────────┐
                    │  Plugin System    │
                    │  (plugins, MCP,   │
                    │   skills, agents) │
                    └───────────────────┘
```

### Relationships
- **Agents** and **Skills** are both plugin-provided extensions managed through the plugin system
- **Memory** files (CLAUDE.md) provide persistent context that agents and skills can reference
- **Tasks** are background processes that agents may spawn
- **Hooks** intercept tool events and can trigger custom behavior for any tool, including agent tools
