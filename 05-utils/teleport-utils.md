# Teleport Utils

## Purpose

Utilities for Claude Code's remote session (teleport) feature — managing cloud/BYOC environments, fetching and interacting with remote sessions via the Sessions API, creating git bundles for seeding remote environments, and selecting which environment to use.

## Location

- `restored-src/src/utils/teleport/api.ts` — API client with retry logic, session CRUD, event sending
- `restored-src/src/utils/teleport/environments.ts` — Environment listing and creation
- `restored-src/src/utils/teleport/environmentSelection.ts` — Environment selection logic from settings
- `restored-src/src/utils/teleport/gitBundle.ts` — Git bundle creation and upload for CCR seed-bundle seeding

## Key Exports

### API Client (`api.ts`)

#### Types
- `SessionStatus`: `'requires_action' | 'running' | 'idle' | 'archived'`
- `SessionResource`: Full session object with id, title, status, environment_id, session_context
- `SessionContext`: Sources (git/knowledge_base), cwd, outcomes, model, seed_bundle_file_id, github_pr
- `CodeSession`: Simplified session shape for UI display (Zod-validated)
- `ListSessionsResponse`: Paginated session list
- `RemoteMessageContent`: Plain string or array of content blocks for remote messages

#### Functions
- `prepareApiRequest()`: Validates OAuth access token and organization UUID; throws if not authenticated
- `getOAuthHeaders(accessToken)`: Create standard OAuth headers (Authorization, Content-Type, anthropic-version)
- `axiosGetWithRetry(url, config?)`: GET request with exponential backoff retry (2s, 4s, 8s, 16s — 5 total attempts) for transient errors (network errors, 5xx)
- `isTransientNetworkError(error)`: Check if an axios error is retryable (no response or 5xx)
- `fetchCodeSessionsFromSessionsAPI()`: Fetch code sessions from `/v1/sessions`, transform to CodeSession format
- `fetchSession(sessionId)`: Fetch a single session by ID. Handles 404 (not found) and 401 (expired) specifically
- `getBranchFromSession(session)`: Extract first branch name from session's git repository outcomes
- `sendEventToRemoteSession(sessionId, messageContent, opts?)`: Send a user message event to an existing remote session. Supports UUID for echo dedup. 30s timeout for cold-start containers
- `updateSessionTitle(sessionId, title)`: Update the title of an existing remote session via PATCH

#### Constants
- `CCR_BYOC_BETA`: `'ccr-byoc-2025-07-29'` — beta header for CCR BYOC endpoints

### Environments (`environments.ts`)

#### Types
- `EnvironmentKind`: `'anthropic_cloud' | 'byoc' | 'bridge'`
- `EnvironmentResource`: `{ kind, environment_id, name, created_at, state }`
- `EnvironmentListResponse`: Paginated environment list

#### Functions
- `fetchEnvironments()`: Fetch available environments from `/v1/environment_providers`. Requires OAuth authentication
- `createDefaultCloudEnvironment(name)`: Create a default anthropic_cloud environment with Python 3.11 and Node 20

### Environment Selection (`environmentSelection.ts`)

#### Types
- `EnvironmentSelectionInfo`: `{ availableEnvironments, selectedEnvironment, selectedEnvironmentSource }`

#### Functions
- `getEnvironmentSelectionInfo()`: Fetch environments and determine which one would be used. Prefers non-bridge environments; respects `remote.defaultEnvironmentId` from settings with source tracking

### Git Bundle (`gitBundle.ts`)

#### Types
- `BundleUploadResult`: `{ success, fileId?, bundleSizeBytes?, scope?, hasWip? }` or `{ success: false, error, failReason? }`
- `BundleScope`: `'all' | 'head' | 'squashed'` — bundle scope tiers
- `BundleFailReason`: `'git_error' | 'too_large' | 'empty_repo'`

#### Functions
- `createAndUploadGitBundle(config, opts?)`: Create a git bundle and upload to Files API. Returns file_id for `seed_bundle_file_id` on SessionContext.

#### Bundle Creation Flow (Fallback Chain)
1. **`--all`**: Bundle all refs (including `refs/seed/stash` for WIP). Most complete but largest
2. **`HEAD`**: Bundle current branch only. Drops side branches/tags but keeps full history
3. **`squashed-root`**: Single parentless commit of HEAD's tree (or stash tree if WIP). No history, just the snapshot

#### WIP Handling
- Uses `git stash create` (dangling commit, doesn't touch working tree) → stored as `refs/seed/stash`
- In squashed mode, WIP is baked into the tree directly (can't bundle stash ref separately without dragging history)
- Untracked files are intentionally not captured
- Cleanup: `refs/seed/stash` and `refs/seed/root` are always deleted after upload (even on crash recovery)

#### Size Limits
- Default: 100MB (configurable via `tengu_ccr_bundle_max_bytes` GrowthBook flag)
- Falls through scope tiers when bundle exceeds limit

## Design Notes

- All API calls require OAuth authentication (API key is not sufficient) — users must run `/login`
- API requests use exponential backoff retry for transient errors but NOT for client errors (4xx)
- Git bundle creation cleans up stale refs from crashed prior runs before starting
- Empty repos (no commits) are detected early and return a specific `empty_repo` fail reason
- The bundle upload uses a fixed `relativePath` (`_source_seed.bundle`) so CCR can locate it
- Environment selection prefers non-bridge environments by default and tracks which settings source configured the selection
