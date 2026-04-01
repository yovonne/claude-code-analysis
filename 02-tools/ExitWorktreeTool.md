# ExitWorktreeTool

## Purpose

ExitWorktreeTool exits a worktree session created by EnterWorktreeTool and restores the session to its original working directory. It provides two actions — keep or remove the worktree — with safety checks to prevent accidental loss of uncommitted work. The tool handles full session state restoration including CWD, project root, hooks configuration, and CWD-dependent caches.

## Location

- `restored-src/src/tools/ExitWorktreeTool/ExitWorktreeTool.ts` — Main tool definition and cleanup logic (330 lines)
- `restored-src/src/tools/ExitWorktreeTool/prompt.ts` — Tool prompt (33 lines)
- `restored-src/src/tools/ExitWorktreeTool/UI.tsx` — Terminal UI rendering (25 lines)
- `restored-src/src/tools/ExitWorktreeTool/constants.ts` — Tool name constant (2 lines)

## Key Exports

| Export | Description |
|--------|-------------|
| `ExitWorktreeTool` | The complete tool definition built via `buildTool()` |
| `Output` | Output type: `{ action, originalCwd, worktreePath, worktreeBranch?, tmuxSessionName?, discardedFiles?, discardedCommits?, message }` |
| `ENTER_WORKTREE_TOOL_NAME` | Constant string `'ExitWorktree'` |

## Input/Output Schemas

### Input Schema

```typescript
{
  action: 'keep' | 'remove',          // Required: keep or remove the worktree
  discard_changes?: boolean,          // Required true when removing with uncommitted work
}
```

The `action` parameter determines the fate of the worktree:
- `"keep"` — leaves the worktree directory and branch intact on disk
- `"remove"` — deletes the worktree directory and its branch

The `discard_changes` parameter is a safety gate:
- Required to be `true` when `action: "remove"` and the worktree has uncommitted files or commits not on the original branch
- If omitted and changes exist, the tool refuses and lists the changes

### Output Schema

```typescript
{
  action: 'keep' | 'remove',          // The action taken
  originalCwd: string,                // Directory the session returns to
  worktreePath: string,               // Path of the worktree that was exited
  worktreeBranch?: string,            // Branch name of the worktree
  tmuxSessionName?: string,           // Associated tmux session (if any)
  discardedFiles?: number,            // Count of uncommitted files discarded
  discardedCommits?: number,          // Count of commits discarded
  message: string,                    // Human-readable confirmation
}
```

## Worktree Cleanup

### Cleanup Flow

```
1. MODEL CALLS ExitWorktree
   - { action: 'keep' | 'remove', discard_changes?: boolean }

2. VALIDATION (validateInput)
   - Check: getCurrentWorktreeSession() is not null
   - If null: reject as no-op (no active EnterWorktree session)
   - If action === 'remove' && !discard_changes:
     a. Count worktree changes (files + commits)
     b. If changes exist: reject with details
     c. If count fails (null): reject (fail-closed)

3. CHANGE COUNTING (at execution time)
   - Re-count changes (state may have changed since validation)
   - Uses git status --porcelain for file count
   - Uses git rev-list --count for commit count
   - Falls back to 0/0 if git commands fail

4. KEEP ACTION
   - keepWorktree() — preserves worktree on disk
   - restoreSessionToOriginalCwd() — restore session state
   - Log analytics: tengu_worktree_kept
   - Return with tmux reattach instructions if applicable

5. REMOVE ACTION
   - killTmuxSession() — kill associated tmux session
   - cleanupWorktree() — delete worktree directory and branch
   - restoreSessionToOriginalCwd() — restore session state
   - Log analytics: tengu_worktree_removed
   - Return with discard summary
```

## Change Detection

### countWorktreeChanges Function

Counts uncommitted files and commits in the worktree relative to the original state:

```typescript
async function countWorktreeChanges(
  worktreePath: string,
  originalHeadCommit: string | undefined,
): Promise<ChangeSummary | null>
```

**File counting:**
```bash
git -C <worktreePath> status --porcelain
```
Counts non-empty lines in the porcelain output.

**Commit counting:**
```bash
git -C <worktreePath> rev-list --count <originalHeadCommit>..HEAD
```
Counts commits unique to the worktree branch.

### Fail-Closed Design

The function returns `null` (not `0/0`) when state cannot be reliably determined:

| Condition | Returns | Rationale |
|-----------|---------|-----------|
| `git status` exits non-zero | `null` | Lock file, corrupt index, bad ref |
| `git rev-list` exits non-zero | `null` | Bad ref or corrupt repo |
| `originalHeadCommit` is undefined but git status succeeded | `null` | Hook-based worktree — no baseline commit to compare against |

Returning `null` instead of `0/0` prevents accidental data loss. A silent `0/0` would let `cleanupWorktree` destroy real work when the change count is unknown.

## Session State Restoration

### restoreSessionToOriginalCwd Function

Restores all session state to reflect the original directory:

```typescript
function restoreSessionToOriginalCwd(
  originalCwd: string,
  projectRootIsWorktree: boolean,
): void
```

**State changes:**

| State | Action |
|-------|--------|
| `setCwd(originalCwd)` | Restore working directory |
| `setOriginalCwd(originalCwd)` | Reset original CWD (EnterWorktree set it to worktree path) |
| `setProjectRoot(originalCwd)` | Only if `projectRootIsWorktree` is true |
| `updateHooksConfigSnapshot()` | Re-read hooks from original directory (only if projectRoot was worktree) |
| `saveWorktreeState(null)` | Clear worktree session state |
| `clearSystemPromptSections()` | Force system prompt recomputation |
| `clearMemoryFileCaches()` | Clear CWD-dependent memoized caches |
| `getPlansDirectory.cache.clear()` | Clear plans directory cache |

### Project Root Detection

The tool detects whether the project root was set to the worktree:

```typescript
const projectRootIsWorktree = getProjectRoot() === getOriginalCwd()
```

This is true when the session was started with `--worktree` (setup.ts sets both to the worktree path). For mid-session EnterWorktreeTool calls, project root is NOT changed, so this is false and project root restoration is skipped.

This preserves the "stable project identity" contract — the project root should not change based on where the user happened to cd before entering a worktree.

## Branch Management

### Keep Action

When `action: 'keep'`:
- The worktree directory remains on disk
- The branch remains in the git repository
- The tmux session (if any) continues running
- The tmux session name is returned so the user can reattach: `tmux attach -t <name>`

### Remove Action

When `action: 'remove'`:
- The tmux session (if any) is killed via `killTmuxSession()`
- The worktree directory is deleted via `cleanupWorktree()`
- The branch is removed from the git repository
- All uncommitted changes are permanently lost

### Change Refusal

When removing with uncommitted work and `discard_changes` is not `true`:

```
Worktree has 3 uncommitted files and 2 commits on the worktree branch.
Removing will discard this work permanently. Confirm with the user, then
re-invoke with discard_changes: true — or use action: "keep" to preserve
the worktree.
```

The tool lists exactly what would be lost, enabling informed user decisions.

## Context Switching

### Directory Restoration

The tool reverses all context changes made by EnterWorktreeTool:

| State | Before Exit | After Exit |
|-------|-------------|------------|
| `process.cwd()` | Worktree path | Original CWD |
| `getCwd()` | Worktree path | Original CWD |
| `getOriginalCwd()` | Worktree path | Original CWD |
| `getProjectRoot()` | Worktree path (if --worktree startup) | Original CWD (if --worktree startup) |
| `getCurrentWorktreeSession()` | Session object | null |

### Cache Invalidation

The same caches cleared on entry are cleared on exit to ensure the session reflects the original directory:

| Cache | Purpose |
|-------|---------|
| `clearSystemPromptSections()` | Forces `env_info_simple` to recompute with original context |
| `clearMemoryFileCaches()` | Clears memoized file lookups for original CWD |
| `getPlansDirectory.cache.clear()` | Clears plans directory resolution cache |

## UI Rendering

### Tool Use Message

```tsx
'Exiting worktree…'
```

Shows a simple loading message while the worktree is being exited.

### Tool Result Message

```tsx
const actionLabel = output.action === 'keep' ? 'Kept worktree' : 'Removed worktree';

<Box flexDirection="column">
  <Text>
    {actionLabel}
    {output.worktreeBranch ? (
      <>
        {' '}
        (branch <Text bold>{output.worktreeBranch}</Text>)
      </>
    ) : null}
  </Text>
  <Text dimColor>Returned to {output.originalCwd}</Text>
</Box>
```

Displays the action taken, branch name (if any) in bold, and the return directory in dimmed text.

## Tool Properties

| Property | Value | Rationale |
|----------|-------|-----------|
| `shouldDefer` | `true` | Requires user approval before executing |
| `isDestructive` | `input.action === 'remove'` | Only destructive when removing |
| `userFacingName` | `'Exiting worktree'` | Shown in progress UI |
| `toAutoClassifierInput` | `input.action` | Uses action for auto-classification |

## Validation and Error Handling

### Input Validation

| Condition | Result | Error Code |
|-----------|--------|------------|
| No active worktree session | Reject as no-op | 1 |
| Cannot verify worktree state (git failure) | Reject (fail-closed) | 3 |
| Uncommitted changes without discard_changes | Reject with change listing | 2 |

### No-Op Guarantee

When called outside an EnterWorktree session, the tool is a **no-op**:
- Reports that no worktree session is active
- Takes no filesystem action
- State is unchanged

This is a safety design — the tool will not touch:
- Worktrees created manually with `git worktree add`
- Worktrees from a previous session
- Any directory if EnterWorktree was never called

### Race Condition Defense

The `call()` method re-checks `getCurrentWorktreeSession()` even though `validateInput()` already checked:

```typescript
const session = getCurrentWorktreeSession()
if (!session) {
  throw new Error('Not in a worktree session')
}
```

This defends against a race between validation and execution, since `currentWorktreeSession` is module-level mutable state.

## Prompt Guidance

### Scope

This tool ONLY operates on worktrees created by EnterWorktree in this session. It will NOT touch:
- Worktrees created manually with `git worktree add`
- Worktrees from a previous session
- The current directory if EnterWorktree was never called

### When to Use

- User explicitly asks to "exit the worktree", "leave the worktree", "go back"
- Do NOT call proactively — only when the user asks

### Parameters

- `action` (required): `"keep"` or `"remove"`
  - `"keep"` — preserve worktree for later use or if there are changes to save
  - `"remove"` — clean deletion when work is done or abandoned
- `discard_changes` (optional): only meaningful with `action: "remove"`
  - If worktree has uncommitted files or commits, the tool REFUSES without this set to `true`
  - If tool returns error listing changes, confirm with user before re-invoking

### Behavior Summary

- Restores session CWD to pre-EnterWorktree location
- Clears CWD-dependent caches
- tmux session: killed on `remove`, left running on `keep`
- After exit, EnterWorktree can be called again for a fresh worktree

## Integration Points

### `bootstrap/state.js`

- `getOriginalCwd()` — Returns the original working directory
- `getProjectRoot()` — Returns the project root directory
- `setOriginalCwd(path)` — Restores the original CWD
- `setProjectRoot(path)` — Restores the project root (conditionally)

### `utils/worktree.js`

- `getCurrentWorktreeSession()` — Returns current worktree session or null
- `keepWorktree()` — Preserves worktree on disk, clears session state
- `cleanupWorktree()` — Deletes worktree directory and branch
- `killTmuxSession(name)` — Kills the associated tmux session

### `utils/Shell.js`

- `setCwd(path)` — Updates the tracked current working directory

### `utils/sessionStorage.js`

- `saveWorktreeState(null)` — Clears persisted worktree session state

### `utils/execFileNoThrow.js`

- `execFileNoThrow('git', args)` — Runs git commands without throwing on non-zero exit (for change counting)

### `utils/claudemd.js`

- `clearMemoryFileCaches()` — Clears memoized CLAUDE.md file caches

### `utils/plans.js`

- `getPlansDirectory()` — Plans directory resolver (cache cleared on exit)

### `utils/hooks/hooksConfigSnapshot.js`

- `updateHooksConfigSnapshot()` — Re-reads hooks configuration from the restored directory

### `constants/systemPromptSections.js`

- `clearSystemPromptSections()` — Clears cached system prompt sections

### `utils/array.js`

- `count(array, predicate)` — Counts elements matching predicate (used for file counting)

### `services/analytics/index.js`

- `logEvent('tengu_worktree_kept', ...)` — Logs worktree keep action
- `logEvent('tengu_worktree_removed', ...)` — Logs worktree remove action

## Data Flow

```
Model calls ExitWorktree { action, discard_changes? }
    |
    v
┌─────────────────────────────────────────┐
│ VALIDATION                              │
│                                         │
│ - Check: active worktree session exists │
│ - If removing: count changes            │
│   - git status --porcelain (files)      │
│   - git rev-list --count (commits)      │
│ - If changes exist without              │
│   discard_changes: reject               │
│ - If count fails: reject (fail-closed)  │
└─────────────────────────────────────────┘
    |
    v
┌─────────────────────────────────────────┐
│ RE-COUNT AT EXECUTION TIME              │
│ (state may have changed since validate) │
│                                         │
│ - Same git commands as validation       │
│ - Falls back to 0/0 on failure          │
│ - Used for analytics + messaging only   │
└─────────────────────────────────────────┘
    |
    v
┌─────────────────────────────────────────┐
│ ACTION: KEEP or REMOVE                  │
│                                         │
│ KEEP:                                   │
│   - keepWorktree()                      │
│   - tmux session left running           │
│                                         │
│ REMOVE:                                 │
│   - killTmuxSession() if applicable     │
│   - cleanupWorktree()                   │
│   - branch and directory deleted        │
└─────────────────────────────────────────┘
    |
    v
┌─────────────────────────────────────────┐
│ SESSION RESTORATION                     │
│                                         │
│ - setCwd(originalCwd)                   │
│ - setOriginalCwd(originalCwd)           │
│ - setProjectRoot(originalCwd) if needed │
│ - updateHooksConfigSnapshot() if needed │
│ - saveWorktreeState(null)               │
│ - Clear all CWD-dependent caches        │
└─────────────────────────────────────────┘
    |
    v
Session is back in original directory
    |
    v
Result: { action, originalCwd, worktreePath, discardedFiles?, discardedCommits?, message }
```
