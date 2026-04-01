# Skills System

## Purpose

The skills system provides a mechanism for defining reusable, invocable workflows that the Claude model can execute. Skills are prompt-based commands that give the model detailed instructions for specific tasks. The system supports multiple skill sources: bundled (shipped with CLI), user-defined (filesystem), project-level, managed (policy), legacy commands, and MCP-discovered skills.

## Location

- `restored-src/src/skills/bundledSkills.ts` — Bundled skill definition type and registry
- `restored-src/src/skills/loadSkillsDir.ts` — Skill loading from directories, parsing, deduplication, dynamic discovery
- `restored-src/src/skills/mcpSkillBuilders.ts` — Write-once registry for MCP skill discovery (avoids circular imports)
- `restored-src/src/skills/bundled/index.ts` — Bundled skill initialization
- `restored-src/src/skills/bundled/` — Individual bundled skill implementations

## Key Exports

### Types (bundledSkills.ts)

- `BundledSkillDefinition`: Definition for a skill that ships with the CLI. Includes name, description, aliases, allowed tools, model, hooks, context, agent, files, and `getPromptForCommand`.

### Functions (bundledSkills.ts)

- `registerBundledSkill(definition)`: Registers a bundled skill. Handles optional file extraction for skills with embedded reference files.
- `getBundledSkills()`: Returns a copy of all registered bundled skills as `Command[]`.
- `clearBundledSkills()`: Clears the registry for testing.
- `getBundledSkillExtractDir(skillName)`: Returns the deterministic extraction directory path for a skill's reference files.

### Functions (loadSkillsDir.ts)

- `getSkillsPath(source, dir)`: Returns the filesystem path for skills/commands directories based on source (userSettings, projectSettings, policySettings, plugin).
- `estimateSkillFrontmatterTokens(skill)`: Estimates token count from skill name, description, and whenToUse.
- `parseSkillFrontmatterFields(frontmatter, markdownContent, resolvedName)`: Parses all shared frontmatter fields for both file-based and MCP skill loading.
- `createSkillCommand(params)`: Creates a `Command` object from parsed skill data, including `getPromptForCommand` with argument substitution and shell execution.
- `getSkillDirCommands(cwd)`: Memoized function that loads all skills from managed, user, project, additional, and legacy command directories. Handles deduplication and conditional skill activation.
- `discoverSkillDirsForPaths(filePaths, cwd)`: Walks up from file paths to discover nested `.claude/skills` directories.
- `addSkillDirectories(dirs)`: Loads skills from discovered directories into the dynamic skills map.
- `activateConditionalSkillsForPaths(filePaths, cwd)`: Activates skills with `paths` frontmatter when matching files are touched.
- `getDynamicSkills()`: Returns all dynamically discovered skills.
- `getConditionalSkillCount()`: Returns count of pending conditional skills.
- `clearSkillCaches()`, `clearDynamicSkills()`: Clear cached and dynamic skill state for testing.

### Functions (mcpSkillBuilders.ts)

- `registerMCPSkillBuilders(builders)`: Registers the two functions (`createSkillCommand`, `parseSkillFrontmatterFields`) that MCP skill discovery needs.
- `getMCPSkillBuilders()`: Retrieves the registered builders. Throws if not yet registered.

### Bundled Skills (bundled/index.ts)

`initBundledSkills()` registers all bundled skills at startup:

| Skill | Purpose |
|---|---|
| `update-config` | Configure settings.json, permissions, hooks, env vars |
| `keybindings-help` | Customize keyboard shortcuts in `~/.claude/keybindings.json` |
| `verify` | Verify code changes by running the app [ANT-only] |
| `debug` | Debug session issues via debug log analysis |
| `lorem-ipsum` | Generate filler text for long context testing [ANT-only] |
| `skillify` | Capture a session's process as a reusable skill [ANT-only] |
| `remember` | Review and organize auto-memory entries [ANT-only] |
| `simplify` | Code review for reuse, quality, and efficiency |
| `batch` | Parallel work orchestration across 5-30 worktree agents |
| `stuck` | Diagnose frozen/slow Claude Code sessions [ANT-only] |
| `claude-in-chrome` | Chrome browser automation via MCP tools |
| `dream` | [KAIROS feature gate] |
| `hunter` | [REVIEW_ARTIFACT feature gate] |
| `loop` | [AGENT_TRIGGERS feature gate] |
| `scheduleRemoteAgents` | [AGENT_TRIGGERS_REMOTE feature gate] |
| `claudeApi` | [BUILDING_CLAUDE_APPS feature gate] |
| `runSkillGenerator` | [RUN_SKILL_GENERATOR feature gate] |

## Dependencies

### Internal Dependencies

- `../types/command.js` — `Command` and `PromptCommand` types
- `../Tool.js` — `ToolUseContext` type
- `../utils/frontmatterParser.js` — YAML frontmatter parsing
- `../utils/argumentSubstitution.js` — `$arg` placeholder replacement
- `../utils/markdownConfigLoader.js` — Legacy command file loading
- `../utils/settings/` — Settings loading, managed paths, plugin-only policy
- `../utils/git/gitignore.js` — Gitignore checking for dynamic discovery
- `../utils/permissions/filesystem.js` — Bundled skills root path
- `../services/analytics/index.js` — Event logging for dynamic skill changes
- `../services/tokenEstimation.js` — Token count estimation

### External Dependencies

- `ignore` — Gitignore-style pattern matching for conditional skills
- `lodash-es/memoize.js` — Memoization for `getSkillDirCommands`
- `fs/promises` — File system operations
- `path` — Path manipulation

## Implementation Details

### Skill Loading Architecture

Skills are loaded from multiple sources in parallel at startup:

```
getSkillDirCommands(cwd) [memoized]
    ↓
Parallel loading:
├── Managed skills (policy) — ~/.claude/managed/.claude/skills/
├── User skills — ~/.claude/skills/
├── Project skills — .claude/skills/ (from project dirs up to home)
├── Additional dir skills — --add-dir paths
└── Legacy commands — ~/.claude/commands/ and .claude/commands/
    ↓
Deduplication by realpath (handles symlinks)
    ↓
Separate conditional skills (with paths frontmatter)
    ↓
Return unconditional skills; store conditionals for later activation
```

### Frontmatter Parsing

Skills use YAML frontmatter in their `SKILL.md` files:

```yaml
---
name: my-skill
description: One-line description
when_to_use: Detailed description of when to auto-invoke
allowed-tools: [Read, Bash(gh:*)]
argument-hint: "<arg1> <arg2>"
arguments: [arg1, arg2]
context: inline | fork
model: sonnet | opus | haiku | inherit
user-invocable: true | false
disable-model-invocation: true | false
effort: low | medium | high | N
paths: [src/**/*.ts, tests/**/*.ts]
hooks: { PreToolUse: [...] }
agent: agent-name
version: 1.0.0
---
```

### Bundled Skill File Extraction

Skills can include embedded reference files via the `files` field in their definition:

1. On first invocation, files are extracted to a deterministic directory under the bundled skills root
2. Extraction is memoized (promise-level) to prevent race conditions from concurrent callers
3. Files are written with `O_EXCL | O_NOFOLLOW` flags and `0o600` permissions for security
4. Path traversal is validated — paths containing `..` or absolute paths throw errors
5. The skill prompt is prefixed with `Base directory for this skill: <dir>` so the model can Read/Grep reference files

### Dynamic Skill Discovery

When files are operated on, the system walks up from the file's parent directory toward `cwd`, looking for `.claude/skills/` directories:

1. Each discovered directory is checked against a `Set` to avoid redundant stat calls
2. Gitignored directories are skipped (via `git check-ignore`)
3. Skills are loaded with deeper paths taking precedence over shallower ones
4. A `skillsLoaded` signal notifies listeners to clear caches

### Conditional Skills

Skills with a `paths` frontmatter field are stored separately and activated when matching files are touched:

1. Path matching uses the `ignore` library (gitignore-style patterns)
2. When a file operation occurs, all conditional skills are checked against the file path
3. Matching skills are moved from `conditionalSkills` to `dynamicSkills`
4. Once activated, a skill stays activated (tracked in `activatedConditionalSkillNames`)

### MCP Skill Discovery

To avoid circular imports between `mcpSkills.ts` and `loadSkillsDir.ts`, the two functions MCP needs (`createSkillCommand`, `parseSkillFrontmatterFields`) are registered at module init time via the `mcpSkillBuilders.ts` leaf module. This module imports nothing but types, so both sides can depend on it without forming a cycle.

### Argument Substitution

Skills support positional arguments via `$arg_name` placeholders:

1. `argument-hint` provides user-facing guidance
2. `arguments` lists the argument names
3. `substituteArguments()` replaces placeholders in the skill prompt at invocation time
4. `${CLAUDE_SKILL_DIR}` is replaced with the skill's directory path
5. `${CLAUDE_SESSION_ID}` is replaced with the current session ID

### Shell Execution in Skills

Skills can include inline shell commands using `!`...`` or `!`...`!` syntax:

- Shell commands are executed via `executeShellCommandsInPrompt()`
- MCP skills are **never** allowed to execute inline shell commands (security boundary)
- Non-MCP skills get `alwaysAllowRules` set to their `allowedTools` for shell command permissions

## Data Flow

```
CLI Startup
    ↓
initBundledSkills() → registerBundledSkill() for each skill
    ↓
getSkillDirCommands(cwd) [memoized]
    ↓
┌─────────────────────────────────────────────────┐
│ Parallel loading from all sources               │
│ ├── Managed (policy) skills                     │
│ ├── User skills (~/.claude/skills/)             │
│ ├── Project skills (.claude/skills/)            │
│ ├── Additional dir skills (--add-dir)           │
│ └── Legacy commands (.claude/commands/)         │
└─────────────────────────────────────────────────┘
    ↓
Deduplication by realpath
    ↓
├── Unconditional skills → returned immediately
└── Conditional skills → stored for path-based activation
    ↓
File operation occurs
    ↓
├── discoverSkillDirsForPaths() → find new .claude/skills/ dirs
├── addSkillDirectories() → load newly discovered skills
└── activateConditionalSkillsForPaths() → activate matching skills
    ↓
Dynamic skills merged with startup skills
    ↓
Available to model via Skill tool
```

## Integration Points

- **Skill tool** — Primary invocation mechanism for skills
- **Command system** — Skills are `Command` objects with `type: 'prompt'`
- **Dynamic discovery** — Skills auto-discovered when relevant files are touched
- **MCP system** — MCP servers can discover and register skills via the builders registry
- **Plugin system** — Built-in plugins can provide skill definitions
- **Bare mode** — `--bare` flag skips auto-discovery, loads only explicit `--add-dir` paths
- **Plugin-only policy** — `isRestrictedToPluginOnly('skills')` can lock skills to plugins only

## Configuration

| Environment Variable | Purpose |
|---|---|
| `CLAUDE_CODE_DISABLE_POLICY_SKILLS` | Disables managed (policy) skill loading |

| Settings | Purpose |
|---|---|
| `projectSettings` enabled | Allows project-level skill discovery |
| `skillsLocked` (plugin-only) | Restricts skills to plugins only |

| Bare Mode | Behavior |
|---|---|
| `--bare` flag | Skips all auto-discovery; only loads `--add-dir` paths and bundled skills |

## Error Handling

- Missing skill directories: Logged as warnings, empty arrays returned
- Invalid frontmatter: Logged via `logForDebugging`, skill skipped
- File extraction failures: Logged, skill continues without base-directory prefix
- Duplicate skills: Detected by realpath, first-wins deduplication with logging
- Path traversal in bundled files: Throws error, skill registration fails
- Circular import avoidance: MCP builders use leaf module pattern

## Testing

- `clearBundledSkills()`, `clearDynamicSkills()`, `clearSkillCaches()` provide test isolation
- `rollWithSeed()` pattern not applicable here, but `getBundledSkillExtractDir()` is deterministic
- `parseSkillFrontmatterFields()` is a pure function, easily testable
- `createSkillCommand()` can be tested with mock parameters

## Related Modules

- [SkillTool](../02-tools/SkillTool.md) — Tool that invokes skills
- [Plugins System](./plugins-system.md) — Plugins can provide skills
- [Hooks Utils](../05-utils/hooks-utils.md) — Skills can define hooks
- [Command System](../01-core-modules/command-system.md) — Skills as commands

## Notes

- The skill system supports two directory formats: the modern `skill-name/SKILL.md` format and the legacy single `.md` file format in `/commands/` directories
- Skills default to `userInvocable: true` — users can type `/skill-name` to invoke them
- Skills with `userInvocable: false` are hidden from users but can be auto-invoked by the model
- The `context: fork` field makes a skill run in a sub-agent with its own context
- Conditional skills (with `paths` frontmatter) are a powerful feature — they activate only when relevant files are touched, keeping context clean
- The memoized `getSkillDirCommands()` uses `lodash-es/memoize` — cache can be cleared with `clearSkillCaches()`
- Several bundled skills are ANT-only (verify, lorem-ipsum, skillify, remember, stuck) and gated by `process.env.USER_TYPE !== 'ant'`
- Feature-gated skills (dream, hunter, loop, etc.) are loaded via `require()` for lazy evaluation — they're not imported at module top level to avoid loading unused code
