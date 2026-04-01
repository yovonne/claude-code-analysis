# Worktree Mode

## Purpose

Single function that unconditionally enables worktree mode for all users. Previously gated by a GrowthBook feature flag, but the flag's caching pattern caused silent failures on first launch before the cache was populated.

## Location

- `restored-src/src/utils/worktreeModeEnabled.ts`

## Key Exports

### Functions

- `isWorktreeModeEnabled(): boolean`: Always returns `true`. Worktree mode is unconditionally enabled for all users.

## Design Notes

- Previously gated by GrowthBook flag `tengu_worktree_mode`, but the `CACHED_MAY_BE_STALE` pattern returns the default (`false`) on first launch before the cache is populated, silently swallowing `--worktree`
- See GitHub issue: https://github.com/anthropics/claude-code/issues/27044
- This is a 12-line file — the entire implementation is `return true`
