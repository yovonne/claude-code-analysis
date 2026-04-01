# File Persistence

## Purpose

Orchestrates persisting session output files to the Files API at the end of each turn. Supports two modes: BYOC (Bring Your Own Cloud) which scans local filesystem and uploads modified files, and Cloud (1P) mode which reads file IDs from extended attributes (rclone handles sync). Only active for remote session users with the appropriate environment configuration.

## Location

- `restored-src/src/utils/filePersistence/filePersistence.ts` — Main orchestration logic
- `restored-src/src/utils/filePersistence/outputsScanner.ts` — Outputs directory scanner, modified file detection
- `restored-src/src/utils/filePersistence/types.ts` — Shared types (referenced by both modules)

## Key Exports

### Orchestration (`filePersistence.ts`)

#### Functions

- `runFilePersistence(turnStartTime, signal?)`: Main entry point. Checks environment kind (only runs in BYOC mode), gathers config (session access token, session ID), scans for modified files, uploads to Files API, and returns event data with success/failure lists.
- `executeFilePersistence(turnStartTime, signal, onResult)`: Wrapper that runs persistence and emits result via callback. Handles errors internally.
- `isFilePersistenceEnabled()`: Checks feature flag, environment kind, session access token, and `CLAUDE_CODE_REMOTE_SESSION_ID` to determine if persistence is active.

### Scanner (`outputsScanner.ts`)

#### Functions

- `findModifiedFiles(turnStartTime, outputsDir)`: Recursively scans the outputs directory for files modified since the turn started. Uses parallel `stat()` calls for efficiency. Skips symlinks for security.
- `getEnvironmentKind()`: Reads `CLAUDE_CODE_ENVIRONMENT_KIND` env var; returns `'byoc'`, `'anthropic_cloud'`, or `null`.
- `logDebug(message)`: Shared debug logger for file persistence modules.

## Execution Flow (BYOC Mode)

1. Check environment kind — return `null` if not `'byoc'`
2. Validate session access token and `CLAUDE_CODE_REMOTE_SESSION_ID`
3. Scan `{cwd}/{sessionId}/outputs` directory for files with `mtime >= turnStartTime`
4. Enforce file count limit (skip files resolving outside outputs directory for security)
5. Upload files in parallel via `uploadSessionFiles()` with configurable concurrency
6. Separate successful uploads (with file IDs) from failures
7. Emit analytics events for start/completion/errors

## Design Notes

- File persistence is gated behind feature flag `FILE_PERSISTENCE` and requires a remote session context — normal CLI users never trigger it
- Security: symlinks are skipped during scanning; files resolving outside the outputs directory are rejected
- Cloud mode (1P) is currently a no-op — xattr-based file ID reading is planned but not yet implemented
- File count limit prevents overwhelming the Files API with large batches
