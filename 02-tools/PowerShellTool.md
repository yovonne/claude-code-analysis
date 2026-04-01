# PowerShellTool

## Purpose

PowerShellTool is the Windows shell command execution engine for Claude Code. It provides a unified interface for running PowerShell commands with comprehensive security controls, sandboxing (on POSIX platforms), output streaming, timeout management, background execution, and PowerShell edition-aware syntax guidance. It mirrors BashTool's architecture adapted for PowerShell semantics.

## Location

- `restored-src/src/tools/PowerShellTool/PowerShellTool.tsx` — Main tool definition and execution logic (~1001 lines)
- `restored-src/src/tools/PowerShellTool/powershellPermissions.ts` — Permission checking
- `restored-src/src/tools/PowerShellTool/powershellSecurity.ts` — Security validators
- `restored-src/src/tools/PowerShellTool/readOnlyValidation.ts` — Read-only command allowlist
- `restored-src/src/tools/PowerShellTool/pathValidation.ts` — Path constraint validation
- `restored-src/src/tools/PowerShellTool/modeValidation.ts` — Mode-specific permission handling
- `restored-src/src/tools/PowerShellTool/commandSemantics.ts` — Exit code interpretation
- `restored-src/src/tools/PowerShellTool/destructiveCommandWarning.ts` — Destructive pattern detection
- `restored-src/src/tools/PowerShellTool/gitSafety.ts` — Git operation safety
- `restored-src/src/tools/PowerShellTool/clmTypes.ts` — Type definitions
- `restored-src/src/tools/PowerShellTool/commonParameters.ts` — Common parameter handling
- `restored-src/src/tools/PowerShellTool/toolName.ts` — Tool name constant
- `restored-src/src/tools/PowerShellTool/prompt.ts` — Tool prompt generation (146 lines)
- `restored-src/src/tools/PowerShellTool/UI.tsx` — Terminal UI rendering

## Key Exports

| Export | Description |
|--------|-------------|
| `PowerShellTool` | The complete tool definition built via `buildTool()` |
| `PowerShellToolInput` | Zod-inferred input type: `{ command, timeout?, description?, run_in_background?, dangerouslyDisableSandbox? }` |
| `Out` | Output type: `{ stdout, stderr, interrupted, returnCodeInterpretation?, isImage?, ... }` |
| `PowerShellProgress` | Progress data type for streaming updates |
| `detectBlockedSleepPattern(command)` | Detects standalone `Start-Sleep N` (N ≥ 2) patterns |

## Input/Output Schemas

### Input Schema

```typescript
{
  command: string,                     // Required: the PowerShell command
  timeout?: number,                    // Optional: timeout in ms (max getMaxTimeoutMs())
  description?: string,                // Optional: human-readable description
  run_in_background?: boolean,         // Optional: run as background task
  dangerouslyDisableSandbox?: boolean, // Optional: bypass sandbox
}
```

When `CLAUDE_CODE_DISABLE_BACKGROUND_TASKS` is set, `run_in_background` is omitted from the schema.

### Output Schema

```typescript
{
  stdout: string,                         // Command standard output
  stderr: string,                         // Shell reset message
  interrupted: boolean,                   // Whether command was interrupted
  returnCodeInterpretation?: string,      // Semantic meaning of non-error exit codes
  isImage?: boolean,                      // True if stdout is a data URI image
  persistedOutputPath?: string,           // Path to full output on disk
  persistedOutputSize?: number,           // Total output size in bytes
  backgroundTaskId?: string,              // ID for background tasks
  backgroundedByUser?: boolean,           // User pressed Ctrl+B
  assistantAutoBackgrounded?: boolean,    // Auto-backgrounded in assistant mode
}
```

## PowerShell Command Classification

### Search Commands (grep equivalents)

| Cmdlet | Description |
|--------|-------------|
| `Select-String` | grep equivalent |
| `Get-ChildItem` (with -Recurse) | find equivalent |
| `findstr` | Native Windows search |
| `where.exe` | Native Windows which |

### Read Commands

| Cmdlet | Description |
|--------|-------------|
| `Get-Content` | cat equivalent |
| `Get-Item` | File info |
| `Test-Path` | test -e equivalent |
| `Resolve-Path` | realpath equivalent |
| `Get-Process` | ps equivalent |
| `Get-Service` | System info |
| `Get-ChildItem` | ls/dir equivalent |
| `Get-Location` | pwd equivalent |
| `Get-FileHash` | Checksum |
| `Get-Acl` | Permissions info |
| `Format-Hex` | Hexdump equivalent |

### Semantic-Neutral Commands

| Cmdlet | Description |
|--------|-------------|
| `Write-Output` | echo equivalent |
| `Write-Host` | Console output |

## PowerShell Edition Support

The tool detects the PowerShell edition and provides edition-specific guidance:

### Windows PowerShell 5.1 (powershell.exe)

- Pipeline chain operators `&&` and `||` are NOT available
- Ternary (`?:`), null-coalescing (`??`), null-conditional (`?.`) operators NOT available
- Avoid `2>&1` on native executables (wraps stderr in ErrorRecord)
- Default file encoding is UTF-16 LE (with BOM)
- `ConvertFrom-Json` returns PSCustomObject, not hashtable

### PowerShell 7+ (pwsh)

- Pipeline chain operators `&&` and `||` ARE available
- Ternary, null-coalescing, and null-conditional operators available
- Default file encoding is UTF-8 without BOM

## Execution Pipeline

```
1. INPUT RECEIVED
   - Model returns PowerShell tool_use with { command, timeout?, description?, ... }

2. VALIDATION
   - validateInput(): checks Windows sandbox policy, blocked sleep patterns
   - Input parsed against Zod schema

3. PERMISSION CHECK
   - powershellToolHasPermission() runs the permission chain
   - Read-only commands auto-allowed via isReadOnlyCommand()
   - Decision: allow / deny / ask

4. EXECUTION (runPowerShellCommand generator)
   - Detect PowerShell path (getCachedPowerShellPath)
   - Determine sandbox mode (platform-dependent)
   - Set up timeout
   - Spawn command via exec()
   - Stream output via onProgress callback
   - Handle backgrounding (explicit, auto, timeout, interrupt)

5. RESULT PROCESSING
   - Output accumulated via EndTruncatingAccumulator
   - Command semantics interpreted (interpretCommandResult)
   - Large output persisted to disk
   - Claude Code hints extracted and stripped
   - Image output resized and capped
   - Git operations tracked
   - Result returned to model
```

## Windows Sandbox Policy

On native Windows, sandboxing is unavailable (bwrap/sandbox-exec are POSIX-only). If enterprise policy requires sandboxing AND forbids unsandboxed commands, PowerShellTool refuses execution:

```
"Enterprise policy requires sandboxing, but sandboxing is not available on native Windows. Shell command execution is blocked on this platform by policy."
```

This check runs in both `validateInput()` and `call()` for defense-in-depth.

## Sleep Pattern Blocking

Standalone `Start-Sleep N` (where N ≥ 2) as the first statement is blocked:

- `Start-Sleep 5` → blocked
- `Start-Sleep 5; Get-Process` → blocked (suggest Monitor tool or run_in_background)
- `Start-Sleep 1` → allowed (sub-2s pacing is fine)
- `Start-Sleep -Milliseconds 500` → allowed
- `Start-Sleep` inside script blocks/subshells → allowed

## Background Execution

PowerShellTool supports the same background execution modes as BashTool:

| Mode | Trigger | Behavior |
|------|---------|----------|
| Explicit | `run_in_background: true` | Spawn background task, return immediately |
| User-initiated | Ctrl+B | Convert foreground to background |
| Timeout | Command exceeds timeout | Auto-background if allowed |
| Assistant mode | 15s blocking budget | Auto-background in KAIROS mode |

Commands disallowed from auto-backgrounding: `Start-Sleep`, `sleep`.

## Interactive Command Prevention

The tool runs with `-NonInteractive` flag. These commands are prohibited:

- `Read-Host`, `Get-Credential`, `Out-GridView`, `$Host.UI.PromptForChoice`, `pause`
- Destructive cmdlets may prompt for confirmation — add `-Confirm:$false`
- Never use `git rebase -i`, `git add -i`, or other interactive editor commands

## Multiline String Handling

For passing multiline strings to native executables (e.g., git commit messages):

```powershell
git commit -m @'
Commit message here.
Second line with $literal dollar signs.
'@
```

- Use single-quoted here-string `@'...'@` to prevent variable expansion
- Closing `'@` MUST be at column 0 (no leading whitespace)
- Use `--%` stop-parsing token for arguments with `-`, `@`, etc.

## Output Handling

- Stderr is NOT merged into stdout (unlike BashTool) — they remain separate
- Leading empty lines stripped from stdout
- Trailing whitespace trimmed
- Large output (> getMaxOutputLength()) persisted to disk
- Image output detected via data URI prefix, resized to cap dimensions
- Claude Code hints extracted and stripped from output

## Git Operation Tracking

Git operations are tracked via `trackGitOperations()` — PowerShell invokes git/gh/glab/curl as external binaries with identical syntax to bash, so the shell-agnostic regex detection works as-is.

## Dependencies

| Module | Purpose |
|--------|---------|
| `utils/Shell.ts` | Shell execution wrapper |
| `utils/shell/powershellDetection.ts` | PowerShell path and edition detection |
| `utils/sandbox/sandbox-adapter.ts` | Sandbox runtime adapter |
| `utils/task/TaskOutput.ts` | Task output streaming |
| `utils/task/diskOutput.ts` | Disk output path management |
| `utils/toolResultStorage.ts` | Large result persistence |
| `utils/claudeCodeHints.ts` | Hint extraction and stripping |
| `utils/stringUtils.ts` | EndTruncatingAccumulator |
| `shared/gitOperationTracking.ts` | Git operation tracking |
| `tasks/LocalShellTask/` | Background task management |
| `BashTool/` | Shared utilities (image handling, CWD reset, utils) |
| `services/analytics/` | Event logging |
