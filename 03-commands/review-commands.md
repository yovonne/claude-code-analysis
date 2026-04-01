# Review Commands

## Purpose

Documents the CLI commands related to code review workflows: `/review` (ultrareview), `/diff`, `/autofix-pr`, `/pr_comments`, and `/good-claude`. These commands enable remote code review sessions, diff inspection, PR comment fetching, and automated PR fixing.

---

## `/review` (Ultrareview)

### Purpose

Launch a teleported code review session in the cloud. Ultrareview creates a remote agent session that analyzes code changes against a base branch or pull request, running multiple parallel agents ("bug hunters") to find issues. Results arrive asynchronously via task notifications.

### Location

- `restored-src/src/commands/review/ultrareviewEnabled.ts` — Feature gate
- `restored-src/src/commands/review/ultrareviewCommand.tsx` — Command entry point
- `restored-src/src/commands/review/reviewRemote.ts` — Remote session launch logic
- `restored-src/src/commands/review/UltrareviewOverageDialog.tsx` — Billing confirmation dialog

### Feature Gate

```typescript
// ultrareviewEnabled.ts
export function isUltrareviewEnabled(): boolean {
  const cfg = getFeatureValue_CACHED_MAY_BE_STALE('tengu_review_bughunter_config', null)
  return cfg?.enabled === true
}
```

The command is visibility-gated by the `tengu_review_bughunter_config` GrowthBook feature flag. When `enabled` is false, the command is filtered out of `getCommands()` entirely.

### Command Registration

The ultrareview command is registered as a `local-jsx` command with lazy loading. It is internal-only (listed in `INTERNAL_ONLY_COMMANDS` as `goodClaude` in the command system).

### Key Exports

#### From `ultrareviewCommand.tsx`

| Export | Description |
|--------|-------------|
| `call(onDone, context, args)` | Command entry point. Checks overage gate, shows billing dialog if needed, launches remote review. |
| `contentBlocksToString(blocks)` | Converts `ContentBlockParam[]` to a plain string for display. |
| `launchAndDone(args, context, onDone, billingNote, signal)` | Launches the remote review and calls `onDone` with the result. Handles abort signals. |

#### From `reviewRemote.ts`

| Export | Description |
|--------|-------------|
| `checkOverageGate()` | Determines billing status: `proceed`, `not-enabled`, `low-balance`, or `needs-confirm`. |
| `confirmOverage()` | Sets the session-level overage confirmation flag. |
| `launchRemoteReview(args, context, billingNote)` | Creates a teleported review session. Returns `ContentBlockParam[]` or `null`. |
| `OverageGate` | Discriminated union type for billing gate results. |

#### From `UltrareviewOverageDialog.tsx`

| Export | Description |
|--------|-------------|
| `UltrareviewOverageDialog({ onProceed, onCancel })` | Dialog prompting user to confirm Extra Usage billing or cancel. |

### Implementation Details

#### Billing Gate Logic (`checkOverageGate`)

The gate determines whether the user can launch an ultrareview and under what billing terms:

1. **Team/Enterprise subscribers** — Always proceed (ultrareview included in plan). No free-review quota check.
2. **Free reviews remaining** — Proceed with a billing note indicating which free review number this is.
3. **Free reviews exhausted** — Check Extra Usage setup:
   - **Not enabled** — Return `not-enabled`, directing user to enable Extra Usage at billing settings.
   - **Low balance** (< $10 available) — Return `low-balance` with the available amount.
   - **First overage in session** — Return `needs-confirm` to show the billing dialog.
   - **Already confirmed** — Proceed with a billing note indicating Extra Usage billing.

#### Remote Review Launch (`launchRemoteReview`)

Supports two modes:

**PR Mode** (when args is a numeric PR number):
- Detects the current GitHub repository via `detectCurrentRepositoryWithHost()`.
- Uses `refs/pull/{N}/head` as the branch reference.
- Teleports to a remote session with `BUGHUNTER_PR_NUMBER` and `BUGHUNTER_REPOSITORY` environment variables.
- Orchestrator runs with `--pr {N}` flag.

**Branch Mode** (when args is not a PR number):
- Bundles the local working tree (`useBundle: true`).
- Computes the merge-base SHA against the default branch.
- Validates: merge-base exists, diff is non-empty, bundle is not too large.
- Teleports with `BUGHUNTER_BASE_BRANCH` set to the merge-base SHA.

**Common Environment Variables:**

| Variable | Default | Max | Purpose |
|----------|---------|-----|---------|
| `BUGHUNTER_DRY_RUN` | `1` | — | Always dry run mode |
| `BUGHUNTER_FLEET_SIZE` | 5 | 20 | Number of parallel review agents |
| `BUGHUNTER_MAX_DURATION` | 10 min | 25 min | Maximum duration per agent |
| `BUGHUNTER_AGENT_TIMEOUT` | 600s | 1800s | Timeout for individual agents |
| `BUGHUNTER_TOTAL_WALLCLOCK` | 22 min | 27 min | Total session wallclock (must be under 30min poll timeout) |

#### Precondition Checks

The launch validates several conditions and returns user-facing errors for recoverable failures:

- **Remote agent eligibility** — Checks for blockers (excluding `no_remote_environment` which is non-blocking).
- **Repository detection** — Must be a GitHub repo for PR mode.
- **Merge-base existence** — Must find a common ancestor with the base branch.
- **Non-empty diff** — Bail early on empty diffs to avoid launching a useless session.
- **Bundle size** — If the working tree is too large to bundle, suggests using PR mode instead.

#### Session Registration

After successful teleport, the session is registered as a `RemoteAgentTask` with type `ultrareview`. The polling loop pipes results back into the local session via task-notification. The session URL is provided for tracking.

### Dependencies

#### Internal Dependencies

| Module | Purpose |
|--------|---------|
| `src/services/analytics/growthbook.js` | Feature flag checks |
| `src/services/analytics/index.js` | `logEvent()` for telemetry |
| `src/services/api/ultrareviewQuota.js` | `fetchUltrareviewQuota()` |
| `src/services/api/usage.js` | `fetchUtilization()` |
| `src/tasks/RemoteAgentTask/RemoteAgentTask.js` | `checkRemoteAgentEligibility()`, `registerRemoteAgentTask()`, `getRemoteTaskSessionUrl()` |
| `src/utils/auth.js` | `isTeamSubscriber()`, `isEnterpriseSubscriber()` |
| `src/utils/detectRepository.js` | `detectCurrentRepositoryWithHost()` |
| `src/utils/execFileNoThrow.js` | Git command execution |
| `src/utils/git.js` | `getDefaultBranch()`, `gitExe()` |
| `src/utils/teleport.js` | `teleportToRemote()` |
| `src/components/design-system/Dialog.js` | Dialog wrapper for overage prompt |
| `src/components/CustomSelect/select.js` | Select component for billing dialog |

### Configuration

- **Feature Flag**: `tengu_review_bughunter_config` (GrowthBook) — controls both command visibility and review agent tuning parameters.
- **Billing URL**: `https://claude.ai/settings/billing` — where users enable Extra Usage.
- **Minimum Balance**: $10 — minimum available balance required to launch an overage review.

### Error Handling

| Error Condition | Response |
|----------------|----------|
| Feature flag disabled | Command hidden from listings |
| Free reviews exhausted, Extra Usage not enabled | System message directing to billing settings |
| Balance below $10 | System message with available balance |
| Not a GitHub repo (PR mode) | `null` → falls through to local review |
| No merge-base found | Text error with base branch name |
| Empty diff | Text error suggesting to make commits first |
| Bundle too large | Text error suggesting PR mode |
| Teleport fails | `null` → falls through to local review |
| User cancels during launch | Abort signal respected, no `onDone` called on dead transcript |

---

## `/diff`

### Purpose

View uncommitted changes and per-turn diffs in an interactive dialog. Shows the current state of the working tree compared to the last committed state, as well as diffs introduced by each conversation turn.

### Location

- `restored-src/src/commands/diff/index.ts` — Command registration
- `restored-src/src/commands/diff/diff.tsx` — JSX implementation

### Command Registration

```typescript
{
  type: 'local-jsx',
  name: 'diff',
  description: 'View uncommitted changes and per-turn diffs',
  load: () => import('./diff.js'),
}
```

### Key Exports

| Export | Description |
|--------|-------------|
| `call(onDone, context)` | Entry point. Lazy-loads `DiffDialog` and renders it with the current conversation messages. |

### Implementation Details

The command is a thin wrapper that lazy-loads the `DiffDialog` component from `src/components/diff/DiffDialog.js`. It passes the full message history (`context.messages`) to the dialog, which computes and displays diffs between conversation turns and the current file state.

### Dependencies

| Module | Purpose |
|--------|---------|
| `src/components/diff/DiffDialog.js` | Main diff display component (lazy-loaded) |
| `src/types/command.ts` | `LocalJSXCommandCall` type |

---

## `/autofix-pr`

### Purpose

Automatically fix issues identified in a pull request. Currently implemented as a disabled stub.

### Location

- `restored-src/src/commands/autofix-pr/index.js`

### Command Registration

```javascript
{
  isEnabled: () => false,
  isHidden: true,
  name: 'stub',
}
```

### Implementation Details

This command is a **disabled stub** — `isEnabled` always returns `false` and `isHidden` is `true`. It has no functional implementation and cannot be invoked by users. The autofix PR functionality is likely planned but not yet implemented in this version.

---

## `/pr_comments`

### Purpose

Fetch and display comments from a GitHub pull request. When the marketplace is private, falls back to an AI-generated prompt that uses the `gh` CLI to retrieve PR comments.

### Location

- `restored-src/src/commands/pr_comments/index.ts`

### Command Registration

```typescript
createMovedToPluginCommand({
  name: 'pr-comments',
  description: 'Get comments from a GitHub pull request',
  progressMessage: 'fetching PR comments',
  pluginName: 'pr-comments',
  pluginCommand: 'pr-comments',
  async getPromptWhileMarketplaceIsPrivate(args) { /* ... */ }
})
```

### Implementation Details

This command has been **migrated to a plugin** (`pr-comments`). The `createMovedToPluginCommand` wrapper handles the transition:

1. **Plugin available** — Delegates to the `pr-comments` plugin.
2. **Plugin unavailable (private marketplace)** — Generates a detailed prompt that instructs the AI model to fetch PR comments using the GitHub CLI (`gh`).

#### Fallback Prompt Steps

When the plugin is unavailable, the generated prompt instructs the model to:

1. Run `gh pr view --json number,headRepository` to get PR number and repo info.
2. Run `gh api /repos/{owner}/{repo}/issues/{number}/comments` for PR-level comments.
3. Run `gh api /repos/{owner}/{repo}/pulls/{number}/comments` for review comments, including `body`, `diff_hunk`, `path`, `line` fields.
4. Parse and format all comments with file/line context, diff hunks, and threaded replies.
5. Return only the formatted comments with no additional text.

### Dependencies

| Module | Purpose |
|--------|---------|
| `../createMovedToPluginCommand.js` | Plugin migration wrapper |

---

## `/good-claude`

### Purpose

Internal Anthropic command. Currently implemented as a disabled stub.

### Location

- `restored-src/src/commands/good-claude/index.js`

### Command Registration

```javascript
{
  isEnabled: () => false,
  isHidden: true,
  name: 'stub',
}
```

### Implementation Details

This command is a **disabled stub** — `isEnabled` always returns `false` and `isHidden` is `true`. Despite being listed in `INTERNAL_ONLY_COMMANDS` in the command system, the actual implementation is a placeholder. The intended functionality is not present in this version of the codebase.

---

## Data Flow

```
/review <PR#>
  │
  ├── checkOverageGate()
  │     ├── Team/Enterprise? → proceed
  │     ├── Free reviews remaining? → proceed with note
  │     ├── Extra Usage enabled?
  │     │     ├── Balance >= $10?
  │     │     │     ├── Session confirmed? → proceed
  │     │     │     └── Not confirmed? → needs-confirm → show dialog
  │     │     └── Balance < $10? → low-balance error
  │     └── Extra Usage not enabled? → not-enabled error
  │
  ├── launchRemoteReview()
  │     ├── checkRemoteAgentEligibility()
  │     ├── PR mode or Branch mode?
  │     │     ├── PR mode: detect repo → teleport with refs/pull/N/head
  │     │     └── Branch mode: compute merge-base → validate diff → teleport with bundle
  │     ├── registerRemoteAgentTask({ type: 'ultrareview', ... })
  │     └── Return launch result with session URL
  │
  └── onDone(result) → task-notification delivers findings

/diff
  │
  └── DiffDialog(messages=context.messages)
        └── Computes and displays per-turn and uncommitted diffs

/autofix-pr → disabled stub
/pr_comments → delegates to pr-comments plugin (or gh CLI fallback)
/good-claude → disabled stub
```

## Related Modules

- [Command System](../01-core-modules/command-system.md)
- [Teleport Utils](../05-utils/teleport.md)
- [Git Utils](../05-utils/git-utils.md)
- [Remote Agent Task](../01-core-modules/task-system.md)
