# Secure Storage Utilities

## Purpose

The secure storage utilities module provides a platform-aware credential storage system for Claude Code. It abstracts over multiple storage backends (macOS Keychain, plaintext file) with automatic fallback, startup prefetch optimization, and stale-while-error caching to ensure reliable credential access across different environments.

## Location

- `restored-src/src/utils/secureStorage/index.ts` — Factory for platform-appropriate storage
- `restored-src/src/utils/secureStorage/types.ts` — Shared type definitions
- `restored-src/src/utils/secureStorage/macOsKeychainStorage.ts` — macOS Keychain implementation
- `restored-src/src/utils/secureStorage/macOsKeychainHelpers.ts` — Lightweight helpers (no heavy imports)
- `restored-src/src/utils/secureStorage/keychainPrefetch.ts` — Startup prefetch for parallel keychain reads
- `restored-src/src/utils/secureStorage/fallbackStorage.ts` — Primary/secondary fallback wrapper
- `restored-src/src/utils/secureStorage/plainTextStorage.ts` — Plaintext file fallback (`.credentials.json`)

## Key Exports

### Factory (`index.ts`)

#### Functions
- `getSecureStorage()`: Returns the appropriate storage implementation:
  - macOS: `createFallbackStorage(macOsKeychainStorage, plainTextStorage)` — tries Keychain first, falls back to plaintext
  - Other platforms: `plainTextStorage` directly
  - TODO: Add libsecret support for Linux

### macOS Keychain Storage (`macOsKeychainStorage.ts`)

#### Object: `macOsKeychainStorage`
Implements `SecureStorage` interface using the macOS `security` CLI:

- `read()`: Sync read with TTL cache (30s) and stale-while-error — serves stale value on transient failures
- `readAsync()`: Async read with deduplicated in-flight requests (generation-based staleness detection)
- `update(data)`: Write via `security -i` (stdin) to avoid process monitors seeing payload. Falls back to argv when payload exceeds 4032-byte stdin line limit. Data stored as hex-encoded JSON.
- `delete()`: Remove keychain entry via `security delete-generic-password`

#### Functions
- `isMacOsKeychainLocked()`: Check if keychain is locked (exit code 36 from `security show-keychain-info`). Cached for process lifetime since lock state doesn't change during a session.

### macOS Keychain Helpers (`macOsKeychainHelpers.ts`)

**Critical constraint**: This module MUST NOT import `execa`, `execFileNoThrow`, or similar heavy modules. It is loaded at the very top of `main.tsx` before ~65ms of module evaluation, and Bun's `__esm` wrapper evaluates the entire module when any symbol is accessed.

#### Constants
- `CREDENTIALS_SERVICE_SUFFIX` (`'-credentials'`): Suffix for OAuth credentials keychain entry
- `KEYCHAIN_CACHE_TTL_MS` (30,000): Cache TTL to balance cross-process staleness with avoiding repeated sync spawns

#### Functions
- `getMacOsKeychainStorageServiceName(serviceSuffix?)`: Build service name from config dir hash (unique per custom config directory)
- `getUsername()`: Get current username from env or `os.userInfo()`
- `clearKeychainCache()`: Invalidate cache (increments generation, clears in-flight promise)
- `primeKeychainCacheFromPrefetch(stdout)`: Populate cache from prefetch result (only if cache untouched)

#### State
- `keychainCacheState`: Shared mutable object with `cache`, `generation`, and `readInFlight` fields. Lives here (not in storage module) so `keychainPrefetch.ts` can prime it without pulling in `execa`.

### Keychain Prefetch (`keychainPrefetch.ts`)

**Purpose**: Fire both keychain reads (OAuth credentials + legacy API key) in parallel at startup, overlapping the ~65ms subprocess cost with main.tsx module evaluation.

#### Functions
- `startKeychainPrefetch()`: Fire both `security find-generic-password` subprocesses immediately (non-blocking). Non-darwin and bare mode are no-ops.
- `ensureKeychainPrefetchCompleted()`: Await prefetch completion. Called in main.tsx preAction — nearly free since subprocesses finish during import evaluation.
- `getLegacyApiKeyPrefetchResult()`: Get legacy API key prefetch result for auth.ts to skip its sync spawn.
- `clearLegacyApiKeyPrefetch()`: Clear prefetch result on cache invalidation.

#### Design
- Distinguishes "not started" (`null`) from "completed with no key" (`{ stdout: null }`) so sync readers only trust completed prefetches.
- Timed-out prefetches don't prime null — sync path retries with its own longer timeout.

### Fallback Storage (`fallbackStorage.ts`)

#### Functions
- `createFallbackStorage(primary, secondary)`: Creates a storage wrapper that tries primary first, falls back to secondary. On successful primary write after previously being empty, deletes secondary entry (migration). On primary write failure with existing data, deletes stale primary entry to prevent shadowing fresh secondary data.

### Plaintext Storage (`plainTextStorage.ts`)

#### Object: `plainTextStorage`
Implements `SecureStorage` using a `.credentials.json` file in the Claude config directory:

- `read()`, `readAsync()`: Read and parse JSON file
- `update(data)`: Write JSON with `chmod 0o600` (owner read/write only). Returns warning about plaintext storage.
- `delete()`: Unlink the credentials file

## Dependencies

- `execa` — Process execution (keychain storage only, not helpers/prefetch)
- `../execFileNoThrow.js`, `../execFileNoThrowPortable.js` — Process execution wrappers
- `../slowOperations.js` — JSON operations with lodash cloneDeep
- `../envUtils.js` — Config directory detection
- `../fsOperations.js` — Filesystem abstraction
- `src/constants/oauth.js` — OAuth configuration

## Design Notes

- **Startup optimization**: The keychain prefetch pattern fires subprocesses at the very top of `main.tsx`, parallelizing the ~65ms of sync `security` spawns with module evaluation. The `macOsKeychainHelpers.ts` module is carefully kept free of heavy imports to avoid defeating this optimization.
- **Stale-while-error caching**: Keychain reads cache results for 30s and serve stale values on transient failures. This prevents "Not logged in" flashes during startup storms when 50+ MCP connectors authenticate simultaneously.
- **Generation-based staleness**: `readAsync()` captures the generation number before spawning and skips its cache write if a newer generation exists, preventing stale subprocess results from overwriting fresh `update()` writes.
- **Process monitor evasion**: Keychain writes use `security -i` (stdin) so process monitors (CrowdStrike, etc.) see only `security -i`, not the hex-encoded payload. Falls back to argv when payload exceeds the 4032-byte stdin line limit.
- **Migration safety**: When migrating from plaintext to Keychain, the fallback storage deletes the plaintext entry after successful Keychain write. When primary write fails, it deletes the stale primary entry to prevent it from shadowing the fresh secondary data (fixes /login loops from rotated refresh tokens).
