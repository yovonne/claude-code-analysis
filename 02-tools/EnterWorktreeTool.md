# EnterWorktreeTool

## Purpose

EnterWorktreeTool creates an isolated git worktree (or VCS-agnostic worktree via hooks) and switches the current session into it. This provides a clean, isolated working directory for parallel development without affecting the main branch or working tree. The tool manages the full lifecycle of worktree creation including branch setup, CWD switching, cache invalidation, and session state persistence.

## Location

- `restored-src/src/tools/EnterWorktreeTool/EnterWorktreeTool.ts` — Main tool definition and creation logic (128 lines)
- `restored-src/src/tools/EnterWorktreeTool/prompt.ts` — Tool prompt (31 lines)
- `restored-src/src/tools/EnterWorktreeTool/UI.tsx` — Terminal UI rendering (20 lines)
- `restored-src/src/tools/EnterWorktreeTool/constants.ts` — Tool name constant (2 lines)

## Key Exports

| Export | Description |
|--------|-------------|
| `EnterWorktreeTool` | The complete tool definition built via `buildTool()` |
| `Output` | Output type: `{ worktreePath, worktreeBranch?, message }` |
| `ENTER_WORKTREE_TOOL_NAME` | Constant string `'EnterWorktree'` |

## Input/Output Schemas

### Input Schema

```typescript
{
  name?: string  // Optional worktree name
}
```

The `name` parameter is validated against worktree naming rules:
- Each `/`-separated segment may contain only letters, digits, dots, underscores, and dashes
- Maximum 64 characters total
- If not provided, a random name is generated via `getPlanSlug()`

### Output Schema

```typescript
{
  worktreePath: string,   // Absolute path to the new worktree directory
  worktreeBranch?: string, // Branch name created for this worktree
  message: string,        // Human-readable confirmation
}
```

## Worktree Creation

### Creation Flow

```
1. MODEL CALLS EnterWorktree
   - Optional: { name: "feature-auth" }

2. SESSION VALIDATION
   - Check: not already in a worktree session
   - getCurrentWorktreeSession() must return null
   - Throws if already in a worktree session

3. REPO ROOT RESOLUTION
   - findCanonicalGitRoot(getCwd()) — find the main git repository root
   - If different from current CWD, chdir to repo root
   - This ensures worktree creation works from within nested directories

4. WORKTREE NAME RESOLUTION
   - slug = input.name ?? getPlanSlug()
   - getPlanSlug() generates a random name if none provided

5. WORKTREE CREATION
   - createWorktreeForSession(sessionId, slug)
   - Creates git worktree inside .claude/worktrees/
   - Creates a new branch based on HEAD
   - Returns { worktreePath, worktreeBranch, originalCwd, ... }

6. SESSION STATE SWITCH
   - process.chdir(worktreePath) — change process working directory
   - setCwd(worktreePath) — update tracked CWD
   - setOriginalCwd(worktreePath) — set original CWD to worktree path
   - saveWorktreeState(worktreeSession) — persist worktree session state

7. CACHE INVALIDATION
   - clearSystemPromptSections() — force env_info_simple recomputation
   - clearMemoryFileCaches() — clear CWD-dependent memoized caches
   - getPlansDirectory.cache.clear() — clear plans directory cache

8. ANALYTICS
   - logEvent('tengu_worktree_created', { mid_session: true })

9. RESULT RETURNED
   - worktreePath, worktreeBranch, confirmation message
```

## Branch Management

### Branch Creation

The worktree is created with a new branch based on the current HEAD:

```typescript
const worktreeSession = await createWorktreeForSession(getSessionId(), slug)
```

The `createWorktreeForSession` function (in `utils/worktree.ts`):
1. Creates a new branch from HEAD with a unique name
2. Sets up the git worktree at `.claude/worktrees/<slug>/`
3. Optionally creates a tmux session for terminal isolation
4. Records the original head commit for change tracking on exit

### Branch Naming

Branch names are derived from the worktree slug:
- User-provided name: `feature-auth` → branch `feature-auth`
- Auto-generated name: `plan-abc123` → branch `plan-abc123`

The branch is created fresh from HEAD, so it starts with no unique commits.

## Context Switching

### Directory State Changes

The tool performs a complete context switch to the worktree:

| State | Before | After |
|-------|--------|-------|
| `process.cwd()` | Original directory | Worktree path |
| `getCwd()` | Original directory | Worktree path |
| `getOriginalCwd()` | Original directory | Worktree path (intentional) |
| `getCurrentWorktreeSession()` | null | Worktree session object |

**Note:** `setOriginalCwd` is set to the worktree path intentionally. This is because `getProjectRoot` uses `getOriginalCwd` as a fallback, and during worktree sessions the project root should resolve to the worktree location. See `state.ts` comments for rationale.

### Cache Invalidation

Three cache layers are cleared to ensure the session reflects the worktree context:

| Cache | Purpose |
|-------|---------|
| `clearSystemPromptSections()` | Forces `env_info_simple` to recompute with worktree context |
| `clearMemoryFileCaches()` | Clears memoized file lookups that depend on CWD |
| `getPlansDirectory.cache.clear()` | Clears plans directory resolution cache |

### Session State Persistence

The worktree session state is saved via `saveWorktreeState(worktreeSession)`. This persists:
- `worktreePath` — the worktree directory
- `worktreeBranch` — the branch name
- `originalCwd` — where the session was before entering
- `originalHeadCommit` — HEAD commit at creation time (for change tracking)
- `tmuxSessionName` — associated tmux session if created

This state is used by `ExitWorktreeTool` to restore the session correctly.

## VCS-Agnostic Support

### Hook-Based Worktrees

Outside a git repository, the tool delegates to `WorktreeCreate`/`WorktreeRemove` hooks configured in `settings.json`:

```json
{
  "hooks": {
    "WorktreeCreate": "custom-script.sh",
    "WorktreeRemove": "cleanup-script.sh"
  }
}
```

This allows worktree-like isolation for non-git version control systems or custom workflows. The `createWorktreeForSession` function detects whether it's in a git repository and routes accordingly.

## UI Rendering

### Tool Use Message

```tsx
'Creating worktree…'
```

Shows a simple loading message while the worktree is being created.

### Tool Result Message

```tsx
<Box flexDirection="column">
  <Text>
    Switched to worktree on branch <Text bold>{output.worktreeBranch}</Text>
  </Text>
  <Text dimColor>{output.worktreePath}</Text>
</Box>
```

Displays the branch name in bold and the worktree path in dimmed text.

## Tool Properties

| Property | Value | Rationale |
|----------|-------|-----------|
| `shouldDefer` | `true` | Requires user approval before executing |
| `isDestructive` | Not defined | Worktree creation is non-destructive |
| `userFacingName` | `'Creating worktree'` | Shown in progress UI |
| `toAutoClassifierInput` | `input.name ?? ''` | Uses worktree name for auto-classification |

## Prompt Guidance

### When to Use

Only when the user explicitly mentions "worktree":
- "start a worktree"
- "work in a worktree"
- "create a worktree"
- "use a worktree"

### When NOT to Use

- User asks to create a branch — use git commands instead
- User asks to switch branches — use git checkout/switch
- User asks to fix a bug or work on a feature — use normal git workflow
- Never use unless "worktree" is explicitly mentioned

### Requirements

- Must be in a git repository, OR have WorktreeCreate/WorktreeRemove hooks configured
- Must not already be in a worktree

### Behavior Summary

- In git repo: creates worktree in `.claude/worktrees/` with new branch from HEAD
- Outside git repo: delegates to configured hooks
- Switches session CWD to new worktree
- Use ExitWorktree to leave mid-session (keep or remove)
- On session exit, user is prompted to keep or remove the worktree

## Integration Points

### `bootstrap/state.js`

- `getSessionId()` — Returns the current session identifier
- `setOriginalCwd(path)` — Sets the original working directory for the session

### `utils/worktree.js`

- `createWorktreeForSession(sessionId, slug)` — Creates the worktree and branch
- `getCurrentWorktreeSession()` — Returns current worktree session or null
- `validateWorktreeSlug(slug)` — Validates worktree name format

### `utils/git.js`

- `findCanonicalGitRoot(cwd)` — Resolves to the root of the canonical git repository (handles worktrees)

### `utils/Shell.js`

- `setCwd(path)` — Updates the tracked current working directory

### `utils/sessionStorage.js`

- `saveWorktreeState(session)` — Persists worktree session state for later restoration

### `utils/claudemd.js`

- `clearMemoryFileCaches()` — Clears memoized CLAUDE.md file caches

### `utils/plans.js`

- `getPlanSlug()` — Generates a random plan slug for default worktree name
- `getPlansDirectory()` — Plans directory resolver (cache cleared on entry)

### `constants/systemPromptSections.js`

- `clearSystemPromptSections()` — Clears cached system prompt sections

### `services/analytics/index.js`

- `logEvent('tengu_worktree_created', ...)` — Logs worktree creation

## Data Flow

```
Model calls EnterWorktree { name? }
    |
    v
┌─────────────────────────────────────────┐
│ VALIDATION                              │
│ - Check: not already in worktree session│
│ - Validate name format if provided      │
└─────────────────────────────────────────┘
    |
    v
┌─────────────────────────────────────────┐
│ REPO RESOLUTION                         │
│ - Find canonical git root               │
│ - chdir to root if needed               │
└─────────────────────────────────────────┘
    |
    v
┌─────────────────────────────────────────┐
│ WORKTREE CREATION                       │
│                                         │
│ createWorktreeForSession(sessionId, slug)│
│   - Create branch from HEAD             │
│   - Create git worktree directory       │
│   - Record original head commit         │
│   - Optionally create tmux session      │
│                                         │
│ Returns: {                              │
│   worktreePath, worktreeBranch,         │
│   originalCwd, originalHeadCommit,      │
│   tmuxSessionName                       │
│ }                                       │
└─────────────────────────────────────────┘
    |
    v
┌─────────────────────────────────────────┐
│ CONTEXT SWITCH                          │
│                                         │
│ - process.chdir(worktreePath)           │
│ - setCwd(worktreePath)                  │
│ - setOriginalCwd(worktreePath)          │
│ - saveWorktreeState(session)            │
│ - Clear system prompt caches            │
│ - Clear memory file caches              │
│ - Clear plans directory cache           │
└─────────────────────────────────────────┘
    |
    v
Session is now in worktree context
    |
    v
Result: { worktreePath, worktreeBranch, message }
```
