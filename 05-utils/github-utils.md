# GitHub Utils

## Purpose

Single utility function that detects whether the `gh` CLI is installed and authenticated, used for telemetry purposes. Avoids making network requests by using `gh auth token` (reads local config/keyring only) instead of `gh auth status` (which calls GitHub's API).

## Location

- `restored-src/src/utils/github/ghAuthStatus.ts`

## Key Exports

### Types

- `GhAuthStatus`: Discriminated union — `'authenticated' | 'not_authenticated' | 'not_installed'`

### Functions

- `getGhAuthStatus(): Promise<GhAuthStatus>`: Returns the gh CLI install + auth status. Uses `which()` (Bun.which — no subprocess) to detect install, then checks exit code of `gh auth token` to detect auth. Spawns with `stdout: 'ignore'` so the token never enters this process.

## Execution Flow

1. Call `which('gh')` — if not found, return `'not_installed'`
2. Execute `gh auth token` with `stdout: 'ignore'`, `stderr: 'ignore'`, 5s timeout
3. Exit code 0 → `'authenticated'`; non-zero → `'not_authenticated'`

## Design Notes

- Uses `gh auth token` instead of `gh auth status` because the latter makes a network request to GitHub's API, while `auth token` only reads local config/keyring
- Token is never captured — stdout is set to `'ignore'`
- 5-second timeout prevents hanging on a slow keyring
