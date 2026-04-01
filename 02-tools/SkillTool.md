# SkillTool

## Purpose

The SkillTool is the mechanism for Claude to invoke slash-command skills (also called commands or prompts) within the conversation. Skills are reusable, domain-specific instruction sets that extend Claude's capabilities — things like `/commit`, `/review-pr`, `/pdf`, etc. The tool resolves a skill name to its definition, loads its content into the conversation context, and either executes it inline (expanding the skill's messages into the current turn) or in a forked sub-agent (isolated execution with its own token budget). It also supports experimental remote skill loading from cloud storage.

## Location

- `restored-src/src/tools/SkillTool/SkillTool.ts`
- `restored-src/src/tools/SkillTool/prompt.ts`
- `restored-src/src/tools/SkillTool/constants.ts`
- `restored-src/src/tools/SkillTool/UI.tsx`

## Key Exports

### Functions

- `SkillTool`: The main tool definition built via `buildTool()` — handles skill validation, permission checks, loading, and execution
- `getPrompt()`: Memoized function generating the tool's system prompt with skill listings
- `formatCommandsWithinBudget()`: Formats skill descriptions within a character budget for context window inclusion
- `getCharBudget()`: Calculates the character budget for skill listings based on context window size
- `getSkillToolInfo()`, `getSkillInfo()`: Returns skill count metadata
- `getLimitedSkillToolCommands()`: Returns commands included in the SkillTool prompt
- `clearPromptCache()`: Clears the memoized prompt cache

### Types

- `Progress` (aka `SkillToolProgress`): Progress type for forked skill execution, containing the sub-agent message and prompt content
- `Output`: Union of inline and forked skill result schemas
- `InputSchema`: `{ skill: string, args?: string }`

### Constants

- `SKILL_TOOL_NAME`: `'Skill'`
- `SKILL_BUDGET_CONTEXT_PERCENT`: `0.01` — 1% of context window allocated to skill listing
- `CHARS_PER_TOKEN`: `4` — estimation factor for character-to-token conversion
- `DEFAULT_CHAR_BUDGET`: `8,000` — fallback budget (1% of 200K tokens × 4)
- `MAX_LISTING_DESC_CHARS`: `250` — per-entry hard cap on description length

## Dependencies

### Internal Dependencies

- `prompt.ts` — Tool prompt generation with dynamic skill listings
- `constants.ts` — Tool name constant
- `UI.tsx` — React components for rendering skill tool use/results/progress in terminal
- `runAgent.ts` — Core agent execution engine (for forked skill execution)
- `processSlashCommand.js` — Slash command processing and message expansion
- `commands.js` — Command discovery, loading, and management
- `remoteSkillModules` — Conditional modules for remote skill search/loading (ant-only, experimental)
- `agentContext.js` — Agent identity and context tracking
- `messages.js` — Message creation and normalization utilities
- `pluginTelemetry.js` — Plugin/skill usage analytics

### External Dependencies

- `zod/v4` — Schema validation
- `react` — UI component rendering
- `bun:bundle` — Feature flag gating (`EXPERIMENTAL_SKILL_SEARCH`)
- `lodash-es/uniqBy` — Command deduplication
- `path` — Directory resolution for remote skill cache paths

## Implementation Details

### Input Schema

```typescript
{
  skill: string,   // The skill name (e.g., "commit", "review-pr", "pdf")
  args?: string    // Optional arguments passed to the skill
}
```

The skill name is normalized by stripping a leading slash if present (for compatibility with user-typed `/skill` patterns).

### Output Schema

The output is a union of two shapes depending on execution mode:

**Inline execution (default):**
```typescript
{
  success: boolean,
  commandName: string,
  allowedTools?: string[],   // Tools restricted by the skill
  model?: string,            // Model override if specified
  status?: 'inline'
}
```

**Forked execution:**
```typescript
{
  success: boolean,
  commandName: string,
  status: 'forked',
  agentId: string,           // Sub-agent that executed the skill
  result: string             // Result text from the forked execution
}
```

### Skill Discovery and Loading

#### Command Sources

Skills are discovered from multiple sources, merged and deduplicated:

| Source | Description |
|--------|-------------|
| Built-in | Bundled skills shipped with Claude Code |
| User settings | Skills from `.claude/skills/` or user config |
| Project settings | Skills from project-level `.claude/` directories |
| Plugin/Marketplace | Skills installed from the plugin marketplace |
| MCP skills | Skills loaded from MCP servers (`loadedFrom === 'mcp'`) |

The `getAllCommands()` function merges local commands with MCP skills:
```typescript
async function getAllCommands(context: ToolUseContext): Promise<Command[]> {
  const mcpSkills = context
    .getAppState()
    .mcp.commands.filter(cmd => cmd.type === 'prompt' && cmd.loadedFrom === 'mcp')
  if (mcpSkills.length === 0) return getCommands(getProjectRoot())
  const localCommands = await getCommands(getProjectRoot())
  return uniqBy([...localCommands, ...mcpSkills], 'name')
}
```

#### Skill Listing in Prompt

The `getPrompt()` function generates instructions for the LLM along with a formatted listing of available skills. Skills are formatted within a character budget:

1. Calculate budget: `contextWindowTokens × 4 × 0.01` (or env override via `SLASH_COMMAND_TOOL_CHAR_BUDGET`)
2. Try full descriptions first
3. If over budget, preserve bundled skill descriptions fully, truncate others
4. If extremely over budget, show non-bundled skills as names-only
5. Per-entry cap: `MAX_LISTING_DESC_CHARS` (250 chars)

```
Execute a skill within the main conversation

When users ask you to perform tasks, check if any of the available skills match.
...

How to invoke:
- Use this tool with the skill name and optional arguments
- Examples:
  - `skill: "pdf"` - invoke the pdf skill
  - `skill: "commit", args: "-m 'Fix bug'"` - invoke with arguments
  - `skill: "review-pr", args: "123"` - invoke with arguments
  - `skill: "ms-office-suite:pdf"` - invoke using fully qualified name

Important:
- Available skills are listed in system-reminder messages in the conversation
- When a skill matches the user's request, this is a BLOCKING REQUIREMENT
- NEVER mention a skill without actually calling this tool
- Do not invoke a skill that is already running
- If you see a <command-name> tag in the current conversation turn, the skill has ALREADY been loaded
```

### Input Validation

The `validateInput()` function performs several checks:

1. **Format check**: Skill name must be non-empty after trimming
2. **Slash prefix detection**: Logs analytics if skill name starts with `/`
3. **Remote canonical skill handling** (ant-only): Validates `_canonical_<slug>` names against discovered remote skills
4. **Command existence**: Looks up the command in the merged command registry
5. **Model invocation check**: Rejects skills with `disableModelInvocation: true`
6. **Prompt-type check**: Only prompt-based skills (`type === 'prompt'`) are valid

| Error Code | Condition |
|------------|-----------|
| 1 | Empty or whitespace-only skill name |
| 2 | Unknown skill name |
| 4 | Skill has `disableModelInvocation` |
| 5 | Skill is not a prompt-based skill |
| 6 | Remote skill not discovered in session |

### Permission Checking

The `checkPermissions()` function determines whether a skill execution requires user permission:

1. **Deny rules**: Checks `SkillTool` deny rules first (always honored)
2. **Remote canonical skills** (ant-only): Auto-granted (user-authored content not involved)
3. **Allow rules**: Checks `SkillTool` allow rules for exact match or prefix match (`skill:*`)
4. **Safe properties check**: Auto-allows skills that only use safe `PromptCommand` properties
5. **Default**: Ask user for permission with suggestions for creating allow rules

Rule matching supports:
- **Exact match**: `commit` matches skill `commit`
- **Prefix match**: `review:*` matches `review-pr`, `review-doc`, etc.

Safe properties allowlist (`SAFE_SKILL_PROPERTIES`):
```typescript
const SAFE_SKILL_PROPERTIES = new Set([
  'type', 'progressMessage', 'contentLength', 'argNames', 'model', 'effort',
  'source', 'pluginInfo', 'disableNonInteractive', 'skillRoot', 'context',
  'agent', 'getPromptForCommand', 'frontmatterKeys',
  'name', 'description', 'hasUserSpecifiedDescription', 'isEnabled', 'isHidden',
  'aliases', 'isMcp', 'argumentHint', 'whenToUse', 'paths', 'version',
  'disableModelInvocation', 'userInvocable', 'loadedFrom', 'immediate',
  'userFacingName',
])
```

If a skill has any property NOT in this set with a meaningful value, it requires permission.

### Execution Paths

The `call()` method has three execution paths:

#### 1. Remote Skill Execution (ant-only, experimental)

```
EXPERIMENTAL_SKILL_SEARCH + USER_TYPE=ant + _canonical_<slug>
    │
    ▼
stripCanonicalPrefix(slug)
    │
    ▼
loadRemoteSkill(slug, url) — fetches SKILL.md from cloud storage (GCS/S3/HTTP)
    │
    ├── Cache check (local disk cache)
    ├── Parse frontmatter
    ├── Inject base directory header
    ├── Substitute ${CLAUDE_SKILL_DIR} and ${CLAUDE_SESSION_ID}
    ├── Register with addInvokedSkill() for compaction survival
    │
    ▼
Return inline result with skill content as meta user message
```

Remote skills are declarative markdown — no slash-command expansion (`!command` substitution, `$ARGUMENTS` interpolation) is needed.

#### 2. Forked Sub-Agent Execution

```
command.context === 'fork'
    │
    ▼
executeForkedSkill()
    │
    ├── prepareForkedCommandContext(command, args, context)
    │   ├── Process skill prompt with arguments
    │   ├── Build isolated agent definition
    │   └── Return modified getAppState, prompt messages, skill content
    │
    ├── runAgent() with forked context
    │   ├── Own token budget
    │   ├── Progress reporting via onProgress callback
    │   └── Collect agent messages
    │
    ├── extractResultText(agentMessages)
    │
    └── clearInvokedSkillsForAgent(agentId) — cleanup
```

Forked skills run in an isolated sub-agent with:
- Independent token budget
- Progress messages forwarded to parent as `skill_progress` events
- Result text extracted and returned as `status: 'forked'` output
- Skill content registered for compaction survival

#### 3. Inline Execution (default)

```
Standard skill invocation
    │
    ▼
processPromptSlashCommand(skill, args, commands, context)
    │
    ├── Expand skill prompt with arguments
    │   ├── !command substitution
    │   ├── $ARGUMENTS interpolation
    │   └── Frontmatter key injection
    │
    ├── Process allowed tools restriction
    ├── Apply model override if specified
    ├── Apply effort override if specified
    │
    ▼
Return newMessages + contextModifier
    │
    ├── newMessages: skill's prompt messages tagged with toolUseID
    ├── contextModifier:
    │   ├── allowedTools: auto-allow skill's restricted tool set
    │   ├── mainLoopModel: resolve skill model override
    │   └── effortValue: apply skill effort level
```

Inline execution expands the skill's content directly into the current conversation turn. The skill's system prompt and any argument interpolation are processed, and the resulting messages are injected as user messages tagged with the tool use ID.

### Context Modification

Skills can modify the execution context through the `contextModifier` function:

| Modification | Trigger | Effect |
|-------------|---------|--------|
| `allowedTools` | Skill specifies tool restrictions | Auto-allow those tools in permission context |
| `mainLoopModel` | Skill has `model` field | Override the model for subsequent turns |
| `effortValue` | Skill has `effort` field | Set effort level for the session |

Model resolution uses `resolveSkillModelOverride()` to carry over suffixes like `[1m]` (e.g., `opus[1m]` → preserves the effective context window).

### Progress Reporting

For forked skills, progress is reported through the `onProgress` callback:

```typescript
onProgress({
  toolUseID: `skill_${parentMessage.message.id}`,
  data: {
    message: m,           // The sub-agent message
    type: 'skill_progress',
    prompt: skillContent, // The skill's prompt content
    agentId,              // The sub-agent ID
  },
})
```

The UI renders progress messages using `SubAgentProvider` and `MessageComponent` in condensed mode, showing up to 3 recent messages with a "+N more" indicator.

### Telemetry

Every skill invocation logs a `tengu_skill_tool_invocation` event with:

| Field | Description |
|-------|-------------|
| `command_name` | Sanitized name (built-in/bundled/official or "custom") |
| `_PROTO_skill_name` | Full skill name (PII-tagged, unredacted) |
| `execution_context` | `'inline'`, `'fork'`, or `'remote'` |
| `invocation_trigger` | `'claude-proactive'` or `'nested-skill'` |
| `query_depth` | Nesting depth of skill invocations |
| `was_discovered` | Whether skill was discovered via search (experimental) |
| `skill_name`, `skill_source`, `skill_loaded_from`, `skill_kind` | Internal ant-only fields |
| `plugin_name`, `plugin_repository`, `_PROTO_marketplace_name` | Plugin attribution |

Additional events:
- `tengu_skill_tool_slash_prefix`: When model sends skill name with leading `/`
- `tengu_skill_descriptions_truncated`: When skill listing exceeds budget

### UI Rendering

#### renderToolUseMessage

Displays the skill name during execution. Legacy skills from the deprecated `/commands` folder show with a `/` prefix.

#### renderToolUseProgressMessage

For forked skills, shows live sub-agent progress:
- "Initializing…" when no progress messages yet
- Up to 3 most recent messages in non-verbose mode
- All messages in verbose mode
- "+N more tool uses" indicator for hidden messages
- Uses `SubAgentProvider` for expandable sub-agent UI

#### renderToolResultMessage

- **Forked**: Shows "Done" byline
- **Inline**: Shows "Successfully loaded skill" with tool count and model override if applicable

#### renderToolUseRejectedMessage / renderToolUseErrorMessage

Shows progress history followed by fallback rejection/error messages.

## Configuration

### Environment Variables

| Variable | Purpose |
|----------|---------|
| `SLASH_COMMAND_TOOL_CHAR_BUDGET` | Override the skill listing character budget |
| `USER_TYPE=ant` | Enables ant-only features (remote skills, canonical skills, detailed telemetry) |

### Feature Flags

| Flag | Purpose |
|------|---------|
| `EXPERIMENTAL_SKILL_SEARCH` | Enables remote skill search, loading, and canonical skills |

## Error Handling

### Validation Errors

| Error Code | Condition |
|------------|-----------|
| 1 | Empty or invalid skill name format |
| 2 | Unknown skill — not found in command registry |
| 4 | Skill has `disableModelInvocation` flag |
| 5 | Skill is not a prompt-based skill (e.g., a plain CLI command) |
| 6 | Remote skill not discovered in current session |

### Runtime Errors

- **Command processing failure**: Throws `Error('Command processing failed')` if `processedCommand.shouldQuery` is false
- **Remote skill load failure**: Throws with descriptive error including the slug and underlying error message
- **Remote skill not found**: Throws if the skill was not previously discovered via `DiscoverSkills`

## Data Flow

```
Model calls Skill tool with { skill, args }
    │
    ▼
validateInput()
    ├── Strip leading slash
    ├── Check remote canonical (ant-only)
    ├── Look up command in registry
    ├── Check disableModelInvocation
    └── Verify prompt-based skill
    │
    ▼
checkPermissions()
    ├── Check deny rules → deny
    ├── Check remote canonical (ant-only) → allow
    ├── Check allow rules → allow
    ├── Check safe properties → allow
    └── Default → ask with suggestions
    │
    ▼
call() — Execution path selection
    │
    ├── [remote canonical] ──→ executeRemoteSkill()
    │   ├── loadRemoteSkill(slug, url)
    │   ├── Parse frontmatter, inject headers
    │   ├── Substitute ${CLAUDE_SKILL_DIR}, ${CLAUDE_SESSION_ID}
    │   ├── addInvokedSkill() for compaction
    │   └── Return inline result with meta user message
    │
    ├── [fork context] ──→ executeForkedSkill()
    │   ├── prepareForkedCommandContext()
    │   ├── runAgent() in isolated context
    │   ├── Progress reporting via onProgress
    │   ├── extractResultText()
    │   └── Return forked result
    │
    └── [inline] ──→ processPromptSlashCommand()
        ├── Expand skill prompt with args
        ├── Build newMessages
        ├── Build contextModifier (tools, model, effort)
        └── Return inline result with messages + context modifier
    │
    ▼
mapToolResultToToolResultBlockParam()
    ├── Forked: "Skill completed (forked execution). Result: ..."
    └── Inline: "Launching skill: <name>"
```

## Integration Points

- **Command Registry**: Discovers skills from `.claude/`, plugins, MCP servers, and bundled sources
- **Agent System**: Forked skills use `runAgent()` for isolated sub-agent execution
- **Permission System**: Supports allow/deny rules with prefix matching and safe property allowlisting
- **Plugin System**: Tracks plugin attribution for marketplace skills
- **Compaction System**: `addInvokedSkill()` and `clearInvokedSkillsForAgent()` ensure skill content survives context compaction
- **Model Override System**: Skills can override the active model and effort level
- **Tool Restriction**: Skills can restrict which tools are available during their execution
- **SDK Event Queue**: Progress events emitted for forked skill execution

## Related Modules

- [AgentTool](./AgentTool.md) — Sub-agent execution engine used by forked skills
- [ExitPlanModeTool](./ExitPlanModeTool.md) — Plan mode exit (a built-in skill)
- [commands.js](../commands.js) — Command discovery and management system
- [processSlashCommand.js](../utils/processUserInput/processSlashCommand.js) — Slash command processing
- [pluginTelemetry.js](../utils/telemetry/pluginTelemetry.js) — Plugin usage analytics
