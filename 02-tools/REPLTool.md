# REPLTool

## Purpose

REPLTool is not a standalone tool but a mode configuration that controls which tools are visible to the model during interactive CLI sessions. When REPL mode is enabled, primitive tools (Read, Write, Edit, Glob, Grep, Bash, NotebookEdit, Agent) are hidden from direct model use, forcing the model to communicate through the REPL interface instead. The tools remain accessible internally within the REPL VM context.

## Location

- `restored-src/src/tools/REPLTool/constants.ts` — REPL mode detection and tool list (47 lines)
- `restored-src/src/tools/REPLTool/primitiveTools.ts` — Primitive tool references (40 lines)

## Key Exports

| Export | Description |
|--------|-------------|
| `REPL_TOOL_NAME` | Constant: `'REPL'` |
| `isReplModeEnabled()` | Determines if REPL mode is active |
| `REPL_ONLY_TOOLS` | Set of tool names hidden from direct model use |
| `getReplPrimitiveTools()` | Lazy getter for the actual tool instances |

## REPL Mode Detection

REPL mode is enabled when ALL of the following conditions are met:

```typescript
function isReplModeEnabled(): boolean {
  // 1. CLAUDE_CODE_REPL is not explicitly set to false/0
  if (isEnvDefinedFalsy(process.env.CLAUDE_CODE_REPL)) return false
  
  // 2. Legacy CLAUDE_REPL_MODE=1 forces it on
  if (isEnvTruthy(process.env.CLAUDE_REPL_MODE)) return true
  
  // 3. Default: on for 'ant' users in CLI mode, off for SDK users
  return (
    process.env.USER_TYPE === 'ant' &&
    process.env.CLAUDE_CODE_ENTRYPOINT === 'cli'
  )
}
```

### Environment Variables

| Variable | Effect |
|----------|--------|
| `CLAUDE_CODE_REPL=0` | Disables REPL mode (opt-out) |
| `CLAUDE_REPL_MODE=1` | Forces REPL mode on (legacy) |
| `USER_TYPE=ant` | Enables REPL mode for CLI users (default) |
| `CLAUDE_CODE_ENTRYPOINT=cli` | Required for default REPL mode |

### SDK Users

SDK entrypoints (sdk-ts, sdk-py, sdk-cli) are NOT defaulted to REPL mode. SDK consumers script direct tool calls (Bash, Read, etc.) and REPL mode would hide those tools. The `USER_TYPE` build-time define ensures the ant-native binary doesn't force REPL mode on every SDK subprocess.

## Hidden Tools (REPL_ONLY_TOOLS)

When REPL mode is enabled, these tools are hidden from the model's direct tool list:

| Tool | Purpose |
|------|---------|
| `FileRead` | Read file contents |
| `FileWrite` | Write file contents |
| `FileEdit` | Edit file contents |
| `Glob` | File pattern matching |
| `Grep` | Content search |
| `Bash` | Shell command execution |
| `NotebookEdit` | Jupyter notebook editing |
| `Agent` | Spawn subagents |

These tools remain accessible inside the REPL VM context — the model communicates with them through the REPL interface rather than calling them directly.

## Primitive Tools Reference

`getReplPrimitiveTools()` returns the actual tool instances for the REPL-only tools:

```typescript
export function getReplPrimitiveTools(): readonly Tool[] {
  return (_primitiveTools ??= [
    FileReadTool,
    FileWriteTool,
    FileEditTool,
    GlobTool,
    GrepTool,
    BashTool,
    NotebookEditTool,
    AgentTool,
  ])
}
```

### Lazy Initialization

The getter uses lazy initialization (`??=`) to avoid circular dependency issues. The import chain `collapseReadSearch.ts → primitiveTools.ts → FileReadTool.tsx → ...` loops back through the tool registry, so a top-level const would hit "Cannot access before initialization". Deferring to call time avoids the TDZ (Temporal Dead Zone).

### Usage

The primitive tools are exported so display-side code (`collapseReadSearch`, renderers) can classify and render virtual messages for these tools even when they're absent from the filtered execution tools list.

## REPL Mode Behavior

When REPL mode is active:

1. **Tool list filtered**: The model only sees the REPL tool, not the primitive tools
2. **REPL VM context**: Primitive tools are still available inside the REPL's execution context
3. **Display rendering**: UI components can still classify and render tool messages using `getReplPrimitiveTools()`
4. **User interaction**: Users type commands directly, which are routed to the appropriate primitive tool

## Dependencies

| Module | Purpose |
|--------|---------|
| `utils/envUtils.ts` | Environment variable checking (isEnvDefinedFalsy, isEnvTruthy) |
| `../AgentTool/` | Agent tool reference |
| `../BashTool/` | Bash tool reference |
| `../FileEditTool/` | FileEdit tool reference |
| `../FileReadTool/` | FileRead tool reference |
| `../FileWriteTool/` | FileWrite tool reference |
| `../GlobTool/` | Glob tool reference |
| `../GrepTool/` | Grep tool reference |
| `../NotebookEditTool/` | NotebookEdit tool reference |
