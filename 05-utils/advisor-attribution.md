# Advisor and Commit Attribution

## Purpose

Two independent utility modules: **Advisor** provides a tool that lets Claude consult a stronger reviewer model before substantive work, and **Commit Attribution** tracks and calculates Claude's character-level contribution to files for git commit trailers showing what percentage of changes were AI-generated.

## Location

- `restored-src/src/utils/advisor.ts` — Advisor tool types, configuration, model support, usage extraction
- `restored-src/src/utils/commitAttribution.ts` — File-level attribution tracking, commit calculation, snapshot persistence

## Key Exports

### Advisor (`advisor.ts`)

#### Types
- `AdvisorServerToolUseBlock`: `{ type: 'server_tool_use', id, name: 'advisor', input }` — tool call from the model
- `AdvisorToolResultBlock`: `{ type: 'advisor_tool_result', tool_use_id, content }` — result from the advisor. Content can be `advisor_result` (plain text), `advisor_redacted_result` (encrypted), or `advisor_tool_result_error`.
- `AdvisorBlock`: Union of tool use and tool result blocks

#### Functions
- `isAdvisorBlock(param)`: Type guard to check if a block is an advisor block
- `isAdvisorEnabled()`: Check if advisor tool is enabled. Respects env var `CLAUDE_CODE_DISABLE_ADVISOR_TOOL`, first-party-only beta header, and GrowthBook config `tengu_sage_compass`.
- `canUserConfigureAdvisor()`: Check if the user can configure their own advisor model
- `getExperimentAdvisorModels()`: Get experiment-configured base and advisor model pair (when user cannot configure)
- `modelSupportsAdvisor(model)`: Check if a model supports calling the advisor tool (Opus 4.6, Sonnet 4.6, or ant)
- `isValidAdvisorModel(model)`: Check if a model can serve as an advisor model (same criteria)
- `getInitialAdvisorSetting()`: Get the initial advisor model from settings
- `getAdvisorUsage(usage)`: Extract advisor iteration usage from the API response's `usage.iterations` array

#### Constants
- `ADVISOR_TOOL_INSTRUCTIONS`: Full system prompt instructions for the advisor tool. Describes when to call advisor (before substantive work, when stuck, when done), how to handle advice, and how to resolve conflicts between advisor guidance and empirical evidence.

#### Configuration
- Advisor config comes from GrowthBook feature `tengu_sage_compass` with fields: `enabled`, `canUserConfigure`, `baseModel`, `advisorModel`

### Commit Attribution (`commitAttribution.ts`)

#### Types
- `AttributionState`: Tracks file states, session baselines, surface, starting HEAD SHA, prompt counts, permission prompt counts, and escape counts
- `AttributionSummary`: `{ claudePercent, claudeChars, humanChars, surfaces }` — summary for a commit
- `FileAttribution`: `{ claudeChars, humanChars, percent, surface }` — per-file details
- `AttributionData`: Full attribution data for git notes JSON (version, summary, files, surfaceBreakdown, excludedGenerated, sessions)

#### Constants
- `INTERNAL_MODEL_REPOS`: Allowlist of private repos where internal model names are allowed in git commit trailers. Includes both SSH and HTTPS URL formats for ~25 repos.

#### Repo Classification
- `getAttributionRepoRoot()`: Get repo root for attribution operations (respects agent worktree overrides)
- `isInternalModelRepo`: Async check if current repo is in the internal model allowlist. Memoized per process.
- `isInternalModelRepoCached()`: Synchronous cached result (safe default: false = don't leak)
- `getRepoClassCached()`: Synchronous cached classification: `'internal' | 'external' | 'none' | null`

#### Model Name Sanitization
- `sanitizeModelName(shortName)`: Map internal model variants to public names (e.g., `opus-4-6` → `claude-opus-4-6`)
- `sanitizeSurfaceKey(surfaceKey)`: Sanitize a surface key (e.g., `cli/opus-4-5-fast` → `cli/claude-opus-4-5`)

#### File Tracking
- `createEmptyAttributionState()`: Create initial attribution state for a new session
- `trackFileModification(state, filePath, oldContent, newContent, userModified, mtime?)`: Track a file edit by Claude
- `trackFileCreation(state, filePath, content, mtime?)`: Track a file created by Claude (via bash or other non-tracked mechanism)
- `trackFileDeletion(state, filePath, oldContent)`: Track a file deleted by Claude
- `trackBulkFileChanges(state, changes)`: Track multiple file changes in bulk (avoids O(n²) Map copying for large diffs)
- `getFileMtime(filePath)`: Get file mtimeMs for pre-computation before entering sync callbacks

#### Commit Calculation
- `calculateCommitAttribution(states, stagedFiles)`: Calculate final attribution for staged files. Compares session baseline to committed state. Processes files in parallel. Excludes generated files.
- `getGitDiffSize(filePath)`: Get size of changes from git diff stat output
- `isFileDeleted(filePath)`: Check if a file was deleted in staged changes
- `getStagedFiles()`: Get list of staged files from git

#### Surface Keys
- `getClientSurface()`: Get current client surface from `CLAUDE_CODE_ENTRYPOINT` env var (defaults to `'cli'`)
- `buildSurfaceKey(surface, model)`: Build a surface key including model name (format: `surface/model`)

#### Hashing and Path Normalization
- `computeContentHash(content)`: SHA-256 hash of content
- `normalizeFilePath(filePath)`: Normalize to relative path from repo root. Resolves symlinks for macOS `/tmp` vs `/private/tmp` consistency.
- `expandFilePath(filePath)`: Expand relative path to absolute path from repo root

#### Snapshot Persistence
- `stateToSnapshotMessage(state, messageId)`: Convert attribution state to snapshot message for log persistence
- `restoreAttributionStateFromSnapshots(snapshots)`: Restore state from log snapshots on session resume. Uses only the LAST snapshot (not summing) to avoid quadratic growth bug.
- `attributionRestoreStateFromLog(attributionSnapshots, onUpdateState)`: Wrapper that restores and applies state update
- `incrementPromptCount(attribution, saveSnapshot)`: Increment prompt count and save snapshot. Used to persist count across compaction.

#### Git State
- `isGitTransientState()`: Check if in a transient git state (rebase, merge, cherry-pick, bisect)

## Design Notes

- **Advisor tool**: The SDK doesn't yet have types for advisor blocks — current types are placeholders marked with TODO. The advisor tool takes NO parameters; the entire conversation history is automatically forwarded.
- **Attribution character counting**: Uses common prefix/suffix matching to find the actual changed region, correctly handling same-length replacements (e.g., "Esc" → "esc") where `Math.abs(newLen - oldLen)` would be 0.
- **Snapshot restoration bug fix**: The code uses only the LAST snapshot (not summing across all snapshots) to avoid quadratic growth — 837 snapshots × 280 files was producing 1.15 quadrillion "chars" for a 5KB file over a 5-day session.
- **Internal model name protection**: Model names are sanitized to public equivalents in all repos EXCEPT a specific allowlist of confirmed private repos. This is intentionally a repo allowlist, not an org-wide check, because the anthropics org contains public repos where undercover mode must stay ON.
- **Bulk file changes**: `trackBulkFileChanges` creates ONE copy of the Map and mutates it for each file, avoiding O(n²) cost when processing large git diffs (jj operations touching hundreds of thousands of files).
- **Surface keys** include the model name (e.g., `cli/claude-sonnet-4-6`) so attribution can be broken down by which model made the changes.
