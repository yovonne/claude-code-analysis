# Command System

## Purpose

The Command System is the central mechanism through which Claude Code CLI handles slash commands (e.g., `/help`, `/config`, `/compact`). It provides a unified registration, dispatch, and execution architecture for three distinct command types: **local** commands (return text), **local-jsx** commands (render interactive Ink UI), and **prompt** commands (expand into model-visible content blocks / skills). Commands are loaded from multiple sources — built-in definitions, plugin directories, skill directories, bundled skills, MCP servers, and workflow scripts — and are dynamically filtered based on auth state, feature flags, execution mode (interactive vs. non-interactive), and remote/bridge context.

## Location

- `restored-src/src/commands.ts` — Command registration, aggregation, availability filtering, and remote-mode gating
- `restored-src/src/commands/` — Individual command implementations (~70+ commands as directories or files)
- `restored-src/src/types/command.ts` — Command type definitions and interfaces
- `restored-src/src/utils/slashCommandParsing.ts` — Slash command string parsing
- `restored-src/src/utils/processUserInput/processSlashCommand.tsx` — Slash command execution pipeline
- `restored-src/src/utils/processUserInput/processUserInput.ts` — Top-level user input routing (slash vs. text vs. bash)

## Key Exports

### From commands.ts

| Export | Description |
|--------|-------------|
| `getCommands(cwd)` | Async function returning all available commands for the current user. Memoized by `cwd`. Applies availability, `isEnabled()`, and deduplication filters. |
| `filterCommandsForRemoteMode(commands)` | Filters a command list to only those in `REMOTE_SAFE_COMMANDS`. Used in `--remote` mode to prevent local-only commands from appearing. |
| `findCommand(commandName, commands)` | Looks up a command by name or alias. Returns `undefined` if not found. |
| `getCommand(commandName, commands)` | Like `findCommand` but throws `ReferenceError` with a list of available commands if not found. |
| `hasCommand(commandName, commands)` | Returns `true` if a command exists by name or alias. |
| `builtInCommandNames()` | Memoized `Set<string>` of all built-in command names and aliases. |
| `getSkillToolCommands(cwd)` | Returns prompt-type commands visible to the SkillTool (model-invocable skills). Filters out builtins and commands without descriptions. |
| `getSlashCommandToolSkills(cwd)` | Returns only skill commands (loaded from `skills/`, `plugin`, `bundled`, or with `disableModelInvocation`). |
| `getMcpSkillCommands(mcpCommands)` | Returns MCP-provided skill commands when `MCP_SKILLS` feature is enabled. |
| `isBridgeSafeCommand(cmd)` | Determines if a command is safe to execute when received over the Remote Control bridge. |
| `formatDescriptionWithSource(cmd)` | Formats a command's description with source annotation (e.g., `(plugin)`, `(workflow)`, `(bundled)`). |
| `clearCommandsCache()` / `clearCommandMemoizationCaches()` | Invalidate memoized command loading caches (used after plugins/skills change). |
| `meetsAvailabilityRequirement(cmd)` | Checks if a command's `availability` constraints match the current auth state (`claude-ai`, `console`). |
| `INTERNAL_ONLY_COMMANDS` | Array of commands only visible to Anthropic internal users (`USER_TYPE === 'ant'`). |
| `REMOTE_SAFE_COMMANDS` | `Set<Command>` of commands safe for remote mode (affect only local TUI state). |
| `BRIDGE_SAFE_COMMANDS` | `Set<Command>` of `local`-type commands safe to execute over the Remote Control bridge. |

### From types/command.ts

| Export | Description |
|--------|-------------|
| `Command` | Union type: `CommandBase & (PromptCommand | LocalCommand | LocalJSXCommand)` |
| `CommandBase` | Shared properties: `name`, `description`, `aliases`, `isEnabled`, `isHidden`, `availability`, `loadedFrom`, etc. |
| `PromptCommand` | Commands that expand into `ContentBlockParam[]` sent to the model. Has `getPromptForCommand()`. |
| `LocalCommand` | Commands that return `LocalCommandResult` (`text`, `compact`, or `skip`). |
| `LocalJSXCommand` | Commands that render Ink JSX UI. Has `load()` returning a `LocalJSXCommandModule`. |
| `LocalCommandResult` | Discriminated union: `{ type: 'text', value }`, `{ type: 'compact', compactionResult }`, `{ type: 'skip' }` |
| `LocalJSXCommandContext` | Context passed to JSX commands: `canUseTool`, `setMessages`, `options`, `resume`, etc. |
| `LocalJSXCommandOnDone` | Callback when a JSX command completes. Controls display mode, query behavior, and follow-up input. |
| `CommandAvailability` | `'claude-ai' | 'console'` — auth/provider gating. |
| `getCommandName(cmd)` | Resolves user-visible name, falling back to `cmd.name`. |
| `isCommandEnabled(cmd)` | Resolves `isEnabled()`, defaulting to `true`. |

## Command Interface

The `Command` type is a discriminated union based on `type`:

```typescript
type Command = CommandBase & (
  | PromptCommand      // type: 'prompt' — expands to content blocks for the model
  | LocalCommand       // type: 'local' — returns text/compact/skip result
  | LocalJSXCommand    // type: 'local-jsx' — renders Ink UI components
)
```

**CommandBase** (shared properties):

```typescript
type CommandBase = {
  name: string                          // Command identifier (e.g., 'compact')
  description: string                   // Shown in help/typeahead
  aliases?: string[]                    // Alternative names (e.g., config → ['settings'])
  isEnabled?: () => boolean             // Dynamic enable check (defaults true)
  isHidden?: boolean                    // Hide from typeahead/help (defaults false)
  availability?: CommandAvailability[]  // Auth gating: 'claude-ai' | 'console'
  argumentHint?: string                 // Gray hint text for args (e.g., '[model]')
  whenToUse?: string                    // Detailed usage scenarios (skills)
  version?: string
  loadedFrom?: 'commands_DEPRECATED' | 'skills' | 'plugin' | 'managed' | 'bundled' | 'mcp'
  kind?: 'workflow'                     // Badged in autocomplete
  immediate?: boolean                   // Execute without waiting for stop point
  isSensitive?: boolean                 // Args redacted from history
  disableModelInvocation?: boolean      // Block model-triggered invocation
  userInvocable?: boolean               // Allow user typing /command (default true)
  userFacingName?: () => string         // Override displayed name
  hasUserSpecifiedDescription?: boolean
}
```

**PromptCommand** (skills / model-invocable commands):

```typescript
type PromptCommand = {
  type: 'prompt'
  progressMessage: string               // "loading..." text shown during execution
  contentLength: number                 // Character count for token estimation
  source: SettingSource | 'builtin' | 'mcp' | 'plugin' | 'bundled'
  argNames?: string[]
  allowedTools?: string[]               // Extra tools granted during execution
  model?: string                        // Override model for this command
  context?: 'inline' | 'fork'           // inline = current conversation, fork = sub-agent
  agent?: string                        // Agent type when forked
  effort?: EffortValue
  paths?: string[]                      // Glob patterns — only visible after matching file touch
  skillRoot?: string                    // Base dir for skill hooks (CLAUDE_PLUGIN_ROOT)
  hooks?: HooksSettings                 // Hook registrations
  pluginInfo?: { pluginManifest, repository }
  disableNonInteractive?: boolean
  getPromptForCommand(args, context): Promise<ContentBlockParam[]>
}
```

**LocalCommand** (pure text/compact results):

```typescript
type LocalCommand = {
  type: 'local'
  supportsNonInteractive: boolean
  load: () => Promise<{ call: (args, context) => Promise<LocalCommandResult> }>
}
```

**LocalJSXCommand** (interactive Ink UI):

```typescript
type LocalJSXCommand = {
  type: 'local-jsx'
  load: () => Promise<{ call: (onDone, context, args) => Promise<React.ReactNode> }>
}
```

## Command Registration

Commands are registered from **six sources**, assembled in `commands.ts`:

### 1. Built-in Commands (static imports)

~70 commands imported directly at module load time and collected into the `COMMANDS()` memoized array:

```typescript
const COMMANDS = memoize((): Command[] => [
  addDir, advisor, agents, branch, btw, chrome, clear, color, compact,
  config, copy, desktop, context, cost, diff, doctor, effort, exit,
  fast, files, help, ide, init, mcp, memory, model, /* ... */
])
```

### 2. Feature-Gated Commands (conditional require)

Commands behind feature flags loaded via `require()` only when the flag is active:

```typescript
const proactive = feature('PROACTIVE') || feature('KAIROS')
  ? require('./commands/proactive.js').default : null
const bridge = feature('BRIDGE_MODE')
  ? require('./commands/bridge/index.js').default : null
const voiceCommand = feature('VOICE_MODE')
  ? require('./commands/voice/index.js').default : null
// ... spread into COMMANDS(): ...(bridge ? [bridge] : [])
```

### 3. Internal-Only Commands

Commands visible only when `USER_TYPE === 'ant'` and `!IS_DEMO`:

```typescript
export const INTERNAL_ONLY_COMMANDS = [
  backfillSessions, breakCache, bughunter, commit, commitPushPr,
  ctx_viz, goodClaude, issue, /* ... */
].filter(Boolean)
// Added to COMMANDS(): ...(process.env.USER_TYPE === 'ant' && !process.env.IS_DEMO ? INTERNAL_ONLY_COMMANDS : [])
```

### 4. Skill Directory Commands (async, disk I/O)

Commands loaded from `skills/` directories in the project:

```typescript
const { skillDirCommands } = await getSkillDirCommands(cwd)
```

### 5. Plugin Commands (async, disk I/O)

Commands loaded from installed plugins:

```typescript
const pluginCommands = await getPluginCommands()
const pluginSkills = await getPluginSkills()
const builtinPluginSkills = getBuiltinPluginSkillCommands()
```

### 6. Workflow Commands (feature-gated)

Commands generated from workflow scripts:

```typescript
const workflowCommands = getWorkflowCommands
  ? await getWorkflowCommands(cwd) : []
```

### 7. Bundled Skills (synchronous)

Skills bundled with Claude Code, registered at startup:

```typescript
const bundledSkills = getBundledSkills()
```

### Assembly Order

`loadAllCommands()` merges all sources in this order:

```typescript
return [
  ...bundledSkills,
  ...builtinPluginSkills,
  ...skillDirCommands,
  ...workflowCommands,
  ...pluginCommands,
  ...pluginSkills,
  ...COMMANDS(),  // built-in commands last
]
```

## Command Dispatch Mechanism

Command dispatch flows through three layers:

### Layer 1: `processUserInput()` → `processUserInputBase()`

The entry point for all user input (`processUserInput.ts`). It determines the input type:

```
input string starting with '/'  →  processSlashCommand()
input with mode 'bash'          →  processBashCommand()
all other input                 →  processTextPrompt()
```

Before dispatch, it handles:
- Bridge-safe command override (mobile/web clients)
- Ultraplan keyword detection (rewrites to `/ultraplan`)
- Image processing and attachment extraction
- Hook execution (`UserPromptSubmit` hooks)

### Layer 2: `processSlashCommand()`

Located in `processSlashCommand.tsx`. The main dispatch pipeline:

```
1. parseSlashCommand(inputString)          → extract commandName + args
2. hasCommand(commandName, commands)       → validate command exists
3. getMessagesForSlashCommand(...)         → execute based on command type
4. Log telemetry (tengu_input_command)
5. Return { messages, shouldQuery, ... }
```

### Layer 3: `getMessagesForSlashCommand()` — Type-Based Routing

Switches on `command.type`:

| Type | Execution Path |
|------|---------------|
| `local-jsx` | Loads module via `command.load()`, calls `mod.call(onDone, context, args)`. Returns a Promise that resolves when `onDone()` is called or when JSX is rendered. |
| `local` | Loads module via `command.load()`, calls `mod.call(args, context)`. Returns `LocalCommandResult` (`text`, `compact`, or `skip`). |
| `prompt` | If `command.context === 'fork'`, runs `executeForkedSlashCommand()` (sub-agent). Otherwise calls `getMessagesForPromptSlashCommand()` which invokes `command.getPromptForCommand(args, context)` to get content blocks. |

## Slash Command Parsing

Parsing is handled by `parseSlashCommand()` in `slashCommandParsing.ts`:

```typescript
export function parseSlashCommand(input: string): ParsedSlashCommand | null {
  // 1. Trim and verify starts with '/'
  // 2. Split by spaces
  // 3. Check for MCP commands (second word is '(MCP)')
  // 4. Return { commandName, args, isMcp }
}
```

**Examples:**
- `/search foo bar` → `{ commandName: 'search', args: 'foo bar', isMcp: false }`
- `/mcp:tool (MCP) arg1 arg2` → `{ commandName: 'mcp:tool (MCP)', args: 'arg1 arg2', isMcp: true }`
- `/` → `null` (invalid)
- `not a command` → `null` (no leading `/`)

Command name validation uses `looksLikeCommand()`:

```typescript
export function looksLikeCommand(commandName: string): boolean {
  return !/[^a-zA-Z0-9:\-_]/.test(commandName)
}
```

This distinguishes command names from file paths — `/var/log` is detected as a file path, not a command.

## Command Categories

Commands can be grouped by function:

| Category | Commands |
|----------|----------|
| **Session Management** | `/clear`, `/compact`, `/resume`, `/rename`, `/session`, `/exit` |
| **Configuration** | `/config` (alias: `/settings`), `/model`, `/theme`, `/color`, `/vim`, `/keybindings`, `/permissions`, `/privacy-settings`, `/output-style`, `/effort`, `/sandbox-toggle`, `/hooks` |
| **Information & Status** | `/help`, `/status`, `/cost`, `/usage`, `/stats`, `/statusline`, `/version`, `/release-notes`, `/diff`, `/context`, `/files`, `/insights` |
| **Development Workflow** | `/review`, `/ultrareview`, `/plan`, `/fast`, `/passes`, `/branch`, `/tag`, `/tasks`, `/commit`, `/commit-push-pr`, `/security-review`, `/autofix-pr`, `/pr_comments` |
| **Plugins & Extensions** | `/plugin`, `/mcp`, `/skills`, `/reload-plugins`, `/install-github-app`, `/install-slack-app` |
| **Authentication** | `/login`, `/logout`, `/oauth-refresh` |
| **Communication** | `/copy`, `/share`, `/feedback`, `/btw`, `/export` |
| **Debugging & Diagnostics** | `/doctor`, `/heapdump`, `/debug-tool-call`, `/ctx_viz`, `/ant-trace`, `/perf-issue`, `/env`, `/remote-env` |
| **IDE Integration** | `/ide`, `/desktop`, `/chrome` |
| **AI Features** | `/memory`, `/thinkback`, `/thinkback-play`, `/advisor`, `/summary`, `/rewind` |
| **Remote/Bridge** | `/bridge`, `/bridge-kick`, `/mobile`, `/remote-setup`, `/remoteControlServer` |
| **Feature-Gated** | `/proactive`, `/brief`, `/assistant`, `/voice`, `/fork`, `/peers`, `/buddy`, `/workflows`, `/torch`, `/ultraplan`, `/subscribe-pr` |
| **Internal (Ant-only)** | `/backfill-sessions`, `/break-cache`, `/bughunter`, `/good-claude`, `/issue`, `/init-verifiers`, `/force-snip`, `/mock-limits`, `/teleport`, `/agents-platform` |

## Command Execution Flow

```
User types: /compact summarize key decisions
                    │
                    ▼
        ┌───────────────────────┐
        │   processUserInput()  │  ← Top-level entry
        │   (processUserInput.ts)│
        └───────────┬───────────┘
                    │ input starts with '/'
                    ▼
        ┌───────────────────────┐
        │ processUserInputBase()│  ← Bridge check, ultraplan rewrite,
        │                       │     image processing, attachments
        └───────────┬───────────┘
                    │
                    ▼
        ┌───────────────────────┐
        │ processSlashCommand() │  ← Main dispatch
        │ (processSlashCommand.)│
        │        tsx            │
        └───────────┬───────────┘
                    │
          ┌─────────┼─────────┐
          ▼         ▼         ▼
    ┌─────────┐ ┌──────┐ ┌──────────┐
    │parseSlash│ │hasCmd│ │getMessages│
    │Command() │ │check │ │ForSlash  │
    └─────────┘ └──────┘ │Command() │
          │         │    └────┬─────┘
          │         │         │
          ▼         ▼    ┌────┴─────┐
    {name, args}   true  │ type switch│
                         │           │
                         ▼           ▼
                    ┌────────┐ ┌──────────┐ ┌──────────┐
                    │local   │ │local-jsx │ │ prompt   │
                    │        │ │          │ │          │
                    │mod.call│ │mod.call  │ │getPrompt │
                    │(args)  │ │(onDone)  │ │ForCommand│
                    └───┬────┘ └────┬─────┘ └────┬─────┘
                        │           │            │
                        ▼           ▼            ▼
                   LocalCommand  Ink JSX    ContentBlock[]
                   Result        UI         → model
```

### Result Processing

After command execution, results are transformed into messages:

- **`local` text result**: Wrapped in `<local-command-stdout>` tags as a system message
- **`local` compact result**: Triggers `buildPostCompactMessages()` to rebuild conversation with summary
- **`local` skip result**: Returns empty messages, no query
- **`local-jsx` result**: JSX is rendered via `setToolJSX()`. `onDone()` callback controls message generation
- **`prompt` inline result**: Content blocks become a user message with `isMeta: true`
- **`prompt` fork result**: Runs sub-agent via `runAgent()`, collects messages, wraps in `<local-command-stdout>`

## Command Options

Command arguments are passed as a raw string to the command's `call()` or `getPromptForCommand()` method. Parsing of arguments is the responsibility of each command implementation.

**Argument hints** are displayed in the typeahead UI:

```typescript
// compact/index.ts
argumentHint: '<optional custom summarization instructions>'

// model/index.ts
argumentHint: '[model]'
```

**Sensitive commands** redact their arguments from history:

```typescript
isSensitive: true  // args shown as '***' in messages
```

**Immediate commands** execute without waiting for a model stop point:

```typescript
immediate: true  // bypasses the command queue
```

## Command Help

The `/help` command is a `local-jsx` command that lazy-loads its implementation:

```typescript
// commands/help/index.ts
const help = {
  type: 'local-jsx',
  name: 'help',
  description: 'Show help and available commands',
  load: () => import('./help.js'),
} satisfies Command
```

Command descriptions are displayed with source annotations via `formatDescriptionWithSource()`:

- Built-in commands: plain description
- Plugin commands: `(PluginName) description`
- Workflow commands: `description (workflow)`
- Bundled skills: `description (bundled)`
- User skills: `description (source-name)`

## Remote Mode Filtering

Two separate filtering mechanisms exist:

### 1. Remote Mode (`--remote`)

`filterCommandsForRemoteMode()` filters to `REMOTE_SAFE_COMMANDS`:

```typescript
export const REMOTE_SAFE_COMMANDS: Set<Command> = new Set([
  session,   // Shows QR code / URL for remote session
  exit,      // Exit the TUI
  clear,     // Clear screen
  help,      // Show help
  theme,     // Change terminal theme
  color,     // Change agent color
  vim,       // Toggle vim mode
  cost,      // Show session cost
  usage,     // Show usage info
  copy,      // Copy last message
  btw,       // Quick note
  feedback,  // Send feedback
  plan,      // Plan mode toggle
  keybindings,
  statusline,
  stickers,
  mobile,    // Mobile QR code
])
```

These commands only affect local TUI state and don't depend on local filesystem, git, shell, IDE, MCP, or other local execution context.

### 2. Bridge Mode (Remote Control from mobile/web)

`isBridgeSafeCommand()` determines if a command received over the bridge should execute:

```typescript
export function isBridgeSafeCommand(cmd: Command): boolean {
  if (cmd.type === 'local-jsx') return false   // Ink UI blocked
  if (cmd.type === 'prompt') return true       // Skills safe (expand to text)
  return BRIDGE_SAFE_COMMANDS.has(cmd)         // Explicit local allowlist
}

export const BRIDGE_SAFE_COMMANDS: Set<Command> = new Set([
  compact, clear, cost, summary, releaseNotes, files,
])
```

In `processUserInputBase()`, bridge-origin input is checked before the slash command gate:

```typescript
if (bridgeOrigin && inputString.startsWith('/')) {
  const parsed = parseSlashCommand(inputString)
  const cmd = findCommand(parsed.commandName, commands)
  if (cmd) {
    if (isBridgeSafeCommand(cmd)) {
      effectiveSkipSlash = false  // Allow execution
    } else {
      return { messages: [...], shouldQuery: false,
        resultText: `/${getCommandName(cmd)} isn't available over Remote Control.` }
    }
  }
}
```

## Key Commands Overview

| Command | Type | Description |
|---------|------|-------------|
| `/help` | local-jsx | Show help and available commands |
| `/config` | local-jsx | Open config panel (alias: `/settings`) |
| `/compact` | local | Clear conversation, keep summary in context |
| `/clear` | local | Clear conversation history (aliases: `/reset`, `/new`) |
| `/model` | local-jsx | Set the AI model (dynamic description shows current model) |
| `/session` | local-jsx | Show remote session URL and QR code (only in remote mode) |
| `/exit` | local-jsx | Exit the REPL (alias: `/quit`, immediate) |
| `/copy` | local-jsx | Copy last message to clipboard |
| `/theme` | local-jsx | Change terminal theme |
| `/vim` | local-jsx | Toggle vim mode |
| `/cost` | local-jsx | Show session cost |
| `/usage` | local-jsx | Show usage information |
| `/plan` | local-jsx | Toggle plan mode |
| `/skills` | local-jsx | Manage skills |
| `/mcp` | local-jsx | Manage MCP servers |
| `/login` | local-jsx | Authenticate with Anthropic |
| `/logout` | local-jsx | Sign out |
| `/review` | local | Review code changes |
| `/status` | local-jsx | Show session status |
| `/doctor` | local-jsx | Diagnose configuration issues |
| `/memory` | local-jsx | Manage long-term memory |
| `/plugin` | local-jsx | Manage plugins |
| `/share` | local-jsx | Share session |
| `/feedback` | local-jsx | Send feedback |
| `/btw` | local-jsx | Quick note to append to context |
| `/diff` | local-jsx | Show current diff |
| `/files` | local-jsx | List tracked files |
| `/resume` | local-jsx | Resume a previous session |
| `/rename` | local-jsx | Rename current session |
| `/keybindings` | local-jsx | Manage keybindings |
| `/permissions` | local-jsx | Manage tool permissions |
| `/privacy-settings` | local-jsx | Privacy configuration |
| `/hooks` | local-jsx | Manage hooks |
| `/output-style` | local-jsx | Configure output formatting |
| `/effort` | local-jsx | Set effort level |
| `/sandbox-toggle` | local-jsx | Toggle sandbox mode |
| `/chrome` | local-jsx | Chrome browser integration |
| `/mobile` | local-jsx | Mobile QR code generation |
| `/stickers` | local-jsx | Sticker management |
| `/statusline` | local | Status line toggle |
| `/release-notes` | local | Show changelog |
| `/extra-usage` | local | Additional usage information |
| `/rate-limit-options` | local | Rate limit configuration |
| `/stats` | local-jsx | Show statistics |
| `/fast` | local-jsx | Fast mode toggle |
| `/passes` | local-jsx | Pass configuration |
| `/branch` | local-jsx | Branch management |
| `/tag` | local-jsx | Tag management |
| `/tasks` | local-jsx | Task management |
| `/agents` | local-jsx | Agent management |
| `/add-dir` | local-jsx | Add directory to context |
| `/context` | local-jsx | Show context information |
| `/desktop` | local-jsx | Desktop integration |
| `/ide` | local-jsx | IDE integration |
| `/terminal-setup` | local-jsx | Terminal configuration |
| `/upgrade` | local-jsx | Upgrade Claude Code |
| `/export` | local-jsx | Export session data |
| `/rewind` | local-jsx | Rewind conversation |
| `/thinkback` | local-jsx | Thinkback feature |
| `/thinkback-play` | local-jsx | Play thinkback recording |
| `/advisor` | local | Advisor feature |
| `/summary` | local | Summarize conversation |
| `/teleport` | local | Teleport feature |
| `/security-review` | local | Security review |
| `/commit` | local | Git commit |
| `/commit-push-pr` | local | Commit, push, and create PR |
| `/autofix-pr` | local | Autofix PR issues |
| `/pr_comments` | local-jsx | PR comments management |
| `/ultrareview` | local | Ultra review mode |
| `/heapdump` | local | Memory heap dump |
| `/mock-limits` | local | Mock limits (internal) |
| `/version` | local | Show version (internal) |
| `/env` | local-jsx | Environment variables |
| `/remote-env` | local-jsx | Remote environment |
| `/oauth-refresh` | local | Refresh OAuth token |
| `/debug-tool-call` | local | Debug tool call |
| `/ant-trace` | local | Ant tracing (internal) |
| `/perf-issue` | local | Performance issue reporting |
| `/init` | local | Initialize project |
| `/init-verifiers` | local | Initialize verifiers (internal) |
| `/backfill-sessions` | local | Backfill sessions (internal) |
| `/break-cache` | local | Break cache (internal) |
| `/bughunter` | local | Bug hunting (internal) |
| `/good-claude` | local | Good Claude (internal) |
| `/issue` | local | Issue creation (internal) |
| `/force-snip` | local | Force history snip (internal) |
| `/bridge-kick` | local | Kick bridge connection |
| `/ctx_viz` | local | Context visualization (internal) |

## Key Algorithms

### Command Lookup

```typescript
export function findCommand(commandName: string, commands: Command[]): Command | undefined {
  return commands.find(
    _ =>
      _.name === commandName ||
      getCommandName(_) === commandName ||
      _.aliases?.includes(commandName),
  )
}
```

Commands are matched by `name`, `userFacingName()`, or any `alias`.

### Availability Filtering

```typescript
export function meetsAvailabilityRequirement(cmd: Command): boolean {
  if (!cmd.availability) return true
  for (const a of cmd.availability) {
    switch (a) {
      case 'claude-ai':
        if (isClaudeAISubscriber()) return true
        break
      case 'console':
        if (!isClaudeAISubscriber() && !isUsing3PServices() && isFirstPartyAnthropicBaseUrl())
          return true
        break
    }
  }
  return false
}
```

Commands without `availability` are universal. Commands with `availability` must match at least one auth type.

### Dynamic Skill Deduplication

When `getCommands()` is called, dynamic skills are deduplicated against base commands:

```typescript
const baseCommandNames = new Set(baseCommands.map(c => c.name))
const uniqueDynamicSkills = dynamicSkills.filter(
  s => !baseCommandNames.has(s.name) && meetsAvailabilityRequirement(s) && isCommandEnabled(s),
)
// Inserted after plugin skills but before built-in commands
const insertIndex = baseCommands.findIndex(c => builtInNames.has(c.name))
```

### Memoization Strategy

- `COMMANDS()` — memoized (no key), returns same array every call
- `builtInCommandNames()` — memoized (no key), returns same Set
- `loadAllCommands(cwd)` — memoized by `cwd`, caches expensive disk I/O
- `getSkillToolCommands(cwd)` — memoized by `cwd`
- `getSlashCommandToolSkills(cwd)` — memoized by `cwd`
- `meetsAvailabilityRequirement()` — **not** memoized (auth state can change mid-session)
- `isCommandEnabled()` — **not** memoized (feature flags can change)

### Lazy Loading Pattern

All commands use lazy loading to minimize startup time:

```typescript
// Command registration (lightweight)
const compact = {
  type: 'local',
  name: 'compact',
  description: 'Clear conversation history but keep a summary...',
  load: () => import('./compact.js'),  // Dynamic import
} satisfies Command

// Implementation (loaded on first use)
// compact/compact.js exports:
export async function call(args: string, context: LocalJSXCommandContext) {
  // Heavy implementation here
}
```

## Dependencies

### Internal Dependencies

| Module | Purpose |
|--------|---------|
| `src/types/command.ts` | Command type definitions |
| `src/types/message.ts` | Message types for command results |
| `src/bootstrap/state.ts` | App state access (session ID, remote mode, etc.) |
| `src/constants/messages.ts` | Error messages (`NO_CONTENT_MESSAGE`) |
| `src/constants/xml.ts` | XML tag constants (`COMMAND_MESSAGE_TAG`, `COMMAND_NAME_TAG`) |
| `src/Tool.ts` | `ToolUseContext`, `SetToolJSXFn` types |
| `src/utils/messages.ts` | Message creation utilities |
| `src/utils/debug.ts` | Debug logging |
| `src/utils/log.ts` | Error logging |
| `src/utils/errors.ts` | Error types (`AbortError`, `MalformedCommandError`) |
| `src/utils/envUtils.ts` | Environment variable utilities |
| `src/utils/permissions/permissions.ts` | Tool permission checks |
| `src/utils/attachments.ts` | Attachment extraction |
| `src/utils/telemetry/events.ts` | Telemetry event logging |
| `src/services/analytics/index.js` | Analytics event tracking |
| `src/services/compact/compact.js` | Compaction logic |
| `src/tools/AgentTool/runAgent.js` | Sub-agent execution for forked commands |
| `src/skills/loadSkillsDir.js` | Skill directory command loading |
| `src/skills/bundledSkills.js` | Bundled skill loading |
| `src/plugins/builtinPlugins.js` | Built-in plugin commands |
| `src/utils/plugins/loadPluginCommands.js` | Plugin command loading |
| `src/utils/slashCommandParsing.ts` | Slash command string parsing |
| `src/utils/processUserInput/*.ts` | Input processing pipeline |

### External Dependencies

| Package | Usage |
|---------|-------|
| `@anthropic-ai/sdk` | `ContentBlockParam`, `TextBlockParam` types |
| `lodash-es/memoize.js` | Memoization for command caching |
| `bun:bundle` | Feature flag detection (`feature()`) |
| `crypto` | UUID generation |

## Data Flow

### Command Registration Flow

```
Module load
    │
    ├── Static imports of ~70 built-in commands
    ├── Conditional require() for feature-gated commands
    │
    ▼
COMMANDS() [memoized array]
    │
    ├── getSkills(cwd) → skillDirCommands, pluginSkills, bundledSkills, builtinPluginSkills
    ├── getPluginCommands() → pluginCommands
    ├── getWorkflowCommands(cwd) → workflowCommands (feature-gated)
    │
    ▼
loadAllCommands(cwd) [memoized by cwd]
    │
    ├── meetsAvailabilityRequirement() filter (per-call, not memoized)
    ├── isCommandEnabled() filter (per-call, not memoized)
    ├── Dynamic skill deduplication
    │
    ▼
getCommands(cwd) → filtered Command[]
```

### Command Execution Flow

```
User input: /command args
    │
    ▼
processUserInput()
    │
    ├── Bridge-safe check (if bridgeOrigin)
    ├── Ultraplan keyword check
    ├── Attachment extraction
    │
    ▼
processUserInputBase()
    │
    ├── input starts with '/'?
    │
    ▼
processSlashCommand()
    │
    ├── parseSlashCommand() → { commandName, args, isMcp }
    ├── hasCommand() check
    │   └── If not found: check if looks like command vs file path
    │       └── Known command → "Unknown skill: X" error
    │       └── Not a command → treat as regular prompt
    │
    ▼
getMessagesForSlashCommand()
    │
    ├── userInvocable check (block if false)
    ├── recordSkillUsage() for prompt commands
    │
    ├── type === 'local-jsx'
    │   ├── command.load() → mod.call(onDone, context, args)
    │   ├── onDone() callback resolves Promise
    │   └── setToolJSX() renders Ink UI
    │
    ├── type === 'local'
    │   ├── command.load() → mod.call(args, context)
    │   └── Returns LocalCommandResult:
    │       ├── 'text' → system message with result
    │       ├── 'compact' → buildPostCompactMessages()
    │       └── 'skip' → empty messages
    │
    └── type === 'prompt'
        ├── context === 'fork'?
        │   └── executeForkedSlashCommand() → runAgent() sub-agent
        │       ├── Wait for MCP settle (up to 10s)
        │       ├── Run agent, collect messages
        │       ├── Extract result text
        │       └── Wrap in <local-command-stdout>
        │
        └── inline (default)
            └── getMessagesForPromptSlashCommand()
                ├── Register skill hooks
                ├── addInvokedSkill() for compaction preservation
                ├── command.getPromptForCommand() → ContentBlockParam[]
                ├── Extract attachments from skill content
                └── Build messages with metadata
    │
    ▼
Return ProcessUserInputBaseResult
    │
    ├── messages: Message[]
    ├── shouldQuery: boolean
    ├── allowedTools?: string[]
    ├── model?: string
    ├── effort?: EffortValue
    ├── resultText?: string
    ├── nextInput?: string
    └── submitNextInput?: boolean
```

## Integration Points

### Main Entrypoint

Commands are loaded during app initialization and passed to the REPL component. In `--remote` mode, commands are pre-filtered through `filterCommandsForRemoteMode()`.

### State Management

Commands access app state through `ToolUseContext` which provides `getAppState()`, `setAppState()`, and session-scoped data.

### Tool System

- **SkillTool**: Uses `getSkillToolCommands()` to show model-invocable skills
- **SlashCommandTool**: Uses `getSlashCommandToolSkills()` for slash command skills
- Commands can grant additional tool permissions via `allowedTools`

### Compaction System

- `/compact` and `/clear` call `clearSystemPromptSections()` to reset memoized system prompt sections
- Invoked skills are tracked via `addInvokedSkill()` for compaction preservation
- `/compact` returns `LocalCommandResult` with `type: 'compact'` triggering full conversation rebuild

### Remote Control Bridge

- Commands from mobile/web are filtered through `isBridgeSafeCommand()`
- `BRIDGE_SAFE_COMMANDS` allowlist controls which `local` commands can execute
- `prompt` commands are always safe (expand to text)
- `local-jsx` commands are always blocked (render terminal UI)

### Analytics & Telemetry

Every command invocation logs:
- `tengu_input_command` — command name, source, plugin metadata
- `tengu_input_slash_invalid` — unknown command attempts
- `tengu_input_slash_missing` — malformed slash commands
- `tengu_input_prompt` — regular prompt detection
- Plugin-specific telemetry via `buildPluginCommandTelemetryFields()`

### Hooks System

Prompt commands can register hooks via the `hooks` property:

```typescript
if (command.hooks && hooksAllowedForThisSkill) {
  registerSkillHooks(context.setAppState, sessionId, command.hooks, command.name, command.skillRoot)
}
```

Hooks are blocked for non-admin-trusted sources when the `hooks` setting is restricted to plugins only.

## Related Modules

- [Main Entrypoint](./main-entrypoint.md)
- [Tool System](./tool-system.md)
- [State Management](./state-management.md)
