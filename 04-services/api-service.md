# API Service

## Overview

The API service is the core communication layer between Claude Code and Anthropic's API infrastructure. It manages client creation, authentication, request/response handling, retry logic, error classification, file operations, and bootstrap data fetching across multiple API providers (first-party, AWS Bedrock, Azure Foundry, GCP Vertex AI).

**Key files:**
- `client.ts` — Anthropic SDK client factory
- `claude.ts` — Query execution, request building, response streaming
- `withRetry.ts` — Retry logic with exponential backoff
- `errors.ts` — Error classification and user-facing messages
- `errorUtils.ts` — Connection error detail extraction
- `bootstrap.ts` — Bootstrap data fetching and caching
- `filesApi.ts` — File upload/download operations
- `usage.ts` — Utilization/quota fetching
- `logging.ts` — API request/response logging and metrics

---

## API Client Architecture

### Client Factory (`client.ts`)

The `getAnthropicClient()` function is the central factory for creating Anthropic SDK client instances. It dynamically selects the appropriate SDK variant based on environment configuration:

| Provider | Env Flag | SDK Class | Auth Method |
|---|---|---|---|
| First-party API | (default) | `Anthropic` | API key or OAuth token |
| AWS Bedrock | `CLAUDE_CODE_USE_BEDROCK` | `AnthropicBedrock` | AWS credentials or bearer token |
| Azure Foundry | `CLAUDE_CODE_USE_FOUNDRY` | `AnthropicFoundry` | API key or Azure AD token |
| GCP Vertex AI | `CLAUDE_CODE_USE_VERTEX` | `AnthropicVertex` | GoogleAuth (ADC) |

### Client Configuration

Each client instance is configured with:

- **Default headers:** `x-app: cli`, `User-Agent`, `X-Claude-Code-Session-Id`, optional `x-claude-remote-container-id`, `x-claude-remote-session-id`, `x-client-app`
- **Timeout:** `API_TIMEOUT_MS` env var (default: 600s)
- **Max retries:** Configurable per-call
- **Custom headers:** Parsed from `ANTHROPIC_CUSTOM_HEADERS` (curl-style `Name: Value` format, newline-separated)
- **Proxy support:** Via `getProxyFetchOptions()`
- **Client request IDs:** UUID injected via `x-client-request-id` header for first-party API only, enabling server-side log correlation

### Fetch Wrapper

The `buildFetch()` function wraps the global `fetch` to:
1. Inject `x-client-request-id` UUID for first-party API requests
2. Log each request's path and request ID for debugging
3. Never crash the fetch on logging errors

---

## Authentication

### Multi-Provider Auth Strategy

| Provider | Authentication |
|---|---|
| First-party | `ANTHROPIC_API_KEY` env var, `ANTHROPIC_AUTH_TOKEN`, or Claude.ai OAuth tokens |
| AWS Bedrock | AWS SDK default credentials, `AWS_BEARER_TOKEN_BEDROCK`, or refreshed credentials via `refreshAndGetAwsCredentials()` |
| Azure Foundry | `ANTHROPIC_FOUNDRY_API_KEY` or `DefaultAzureCredential` (Azure AD) |
| GCP Vertex AI | `google-auth-library` with ADC, refreshed via `refreshGcpCredentialsIfNeeded()` |

### OAuth Token Management

- OAuth tokens are checked and refreshed automatically before each API call via `checkAndRefreshOAuthTokenIfNeeded()`
- Claude.ai subscribers use `authToken` instead of `apiKey`
- Token revocation (403) triggers a refresh-and-retry cycle
- The `ANTHROPIC_AUTH_TOKEN` env var takes precedence over key helper resolution

### CCR Mode

In Claude Code Remote (CCR) mode, authentication is handled via infrastructure-provided JWTs rather than `/login`. Transient auth errors (401/403) are treated as network issues and retried.

---

## Request/Response Handling

### Query Execution (`claude.ts`)

Two query modes are supported:

1. **Streaming:** `queryModelWithStreaming()` — yields `StreamEvent` chunks as tokens arrive
2. **Non-streaming:** `queryModelWithoutStreaming()` — waits for complete response

Both wrap the core `queryModel()` generator, which:

1. Resolves the model name (including Bedrock inference profile backing models)
2. Determines beta headers via `getMergedBetas()`
3. Evaluates tool search eligibility and deferred tool loading
4. Builds tool schemas with `toolToAPISchema()` (including `defer_loading` for deferred tools)
5. Normalizes messages for API compatibility
6. Ensures tool_use/tool_result pairing
7. Strips excess media items (max 100 per request)
8. Computes fingerprint for attribution
9. Builds system prompt blocks with cache control markers
10. Configures effort, task budget, and output format parameters
11. Executes via `withRetry()` with streaming or non-streaming fallback

### Request Parameters

- **Model:** Normalized via `normalizeModelStringForAPI()`
- **System prompt:** Built as content blocks with `cache_control` markers
- **Tools:** Full schema array with optional `defer_loading: true` for deferred tools
- **Max tokens:** Model-specific defaults from `getModelMaxOutputTokens()`
- **Temperature:** Configurable override
- **Betas:** Merged from model-specific, first-party-only, and feature-gated headers
- **Extra body:** From `CLAUDE_CODE_EXTRA_BODY` env var
- **Metadata:** User ID, device ID, account UUID, session ID

### Beta Headers

Sticky-on latches prevent mid-session cache busting:
- `afk-mode` — Auto mode (TRANSCRIPT_CLASSIFIER feature)
- `fast-mode` — Fast mode requests
- `cache-editing` — Cached microcompact (CACHED_MICROCOMPACT feature)
- `thinking-clear` — 1h cache TTL reset detection

---

## Rate Limiting

### Rate Limit Detection

Rate limits are detected via:
- HTTP 429 status codes
- `anthropic-ratelimit-unified-representative-claim` header (`five_hour`, `seven_day`, `seven_day_opus`)
- `anthropic-ratelimit-unified-overage-status` header (`allowed`, `allowed_warning`, `rejected`)
- `anthropic-ratelimit-unified-reset` header (Unix timestamp)
- HTTP 529 overloaded errors (including `"type":"overloaded_error"` in message)

### Rate Limit Response

The retry handler (`withRetry.ts`) implements a multi-tier response:

1. **Fast mode short retry:** If `Retry-After < 20s`, wait and retry with fast mode preserved
2. **Fast mode cooldown:** If `Retry-After >= 20s`, enter cooldown (switch to standard speed, min 10 min)
3. **Overage rejection:** If overage is disabled, permanently disable fast mode
4. **Persistent retry:** In unattended mode (`CLAUDE_CODE_UNATTENDED_RETRY`), retry indefinitely with:
   - Exponential backoff capped at 5 minutes
   - Reset-cap at 6 hours
   - Heartbeat yields every 30s to prevent idle detection
5. **Foreground vs background:** Only foreground query sources (main thread, SDK, agents) retry on 529; background sources bail immediately

### Retry Decision Logic

The `shouldRetry()` function considers:
- `x-should-retry` header from server
- Error type (connection, timeout, rate limit, auth, server error)
- Subscriber type (Claude.ai subscribers don't retry rate limits; enterprise users do)
- CCR mode (always retries auth errors)
- Mock rate limits (never retried)

---

## Error Handling

### Error Classification (`errors.ts`)

The `classifyAPIError()` function maps errors to analytics-safe categories:

| Category | Trigger |
|---|---|
| `aborted` | Request aborted |
| `api_timeout` | Connection timeout |
| `rate_limit` | 429 status |
| `server_overload` | 529 status or overloaded_error |
| `prompt_too_long` | "prompt is too long" message |
| `pdf_too_large` | PDF page limit exceeded |
| `image_too_large` | Image size/dimension exceeded |
| `tool_use_mismatch` | tool_use without tool_result |
| `invalid_model` | Invalid model name |
| `credit_balance_low` | Credit balance too low |
| `invalid_api_key` | x-api-key errors |
| `token_revoked` | OAuth token revoked |
| `ssl_cert_error` | SSL/TLS certificate errors |
| `connection_error` | Network connection failures |
| `server_error` | 5xx status codes |
| `client_error` | 4xx status codes |

### User-Facing Error Messages

The `getAssistantMessageFromError()` function converts raw API errors into user-friendly `AssistantMessage` objects with:
- Context-aware messages (CLI vs SDK/non-interactive)
- Recovery suggestions (`/login`, `/model`, `/rewind`, `/compact`)
- Error type tags for UI rendering
- Token count details for prompt-too-long errors
- 3P model fallback suggestions

### Connection Error Details (`errorUtils.ts`)

The `extractConnectionErrorDetails()` function walks the error cause chain to find root error codes, with special handling for SSL/TLS errors (23 known SSL error codes). The `formatAPIError()` function produces user-friendly connection error messages with SSL-specific hints.

### Retry Logic (`withRetry.ts`)

The `withRetry()` generator implements:
- Configurable max retries (default: 10, override via `CLAUDE_CODE_MAX_RETRIES`)
- Exponential backoff with jitter: `BASE_DELAY_MS * 2^(attempt-1) + random(0-25%)`
- `Retry-After` header respect
- Consecutive 529 tracking (max 3 before fallback to Sonnet for Opus users)
- Fast mode cooldown management
- Context overflow auto-adjustment (reduces `max_tokens` on 400)
- AWS/GCP credential cache clearing on auth errors
- Abort signal support
- System message yields for retry progress

---

## Bootstrap Data Fetching

### Purpose

The bootstrap endpoint (`/api/claude_cli/bootstrap`) fetches server-side configuration data at startup:
- `client_data` — Client-specific configuration
- `additional_model_options` — Extra model choices from the server

### Behavior

1. Skipped in essential-traffic-only mode
2. Skipped for non-first-party providers
3. Requires OAuth token (with `user:profile` scope) or API key
4. Uses `withOAuth401Retry()` for token refresh-and-retry
5. Response validated against Zod schema
6. Only persisted to disk cache if data actually changed (avoids config writes on every startup)
7. Cached in global config as `clientDataCache` and `additionalModelOptionsCache`

---

## Files API

### Overview (`filesApi.ts`)

The Files API client manages file uploads and downloads to Anthropic's Public Files API (`/v1/files`). Used for:
- Downloading file attachments at session startup
- Uploading files in BYOC (Bring Your Own Compute) mode
- Listing files created after a timestamp (1P/Cloud mode)

### Configuration

```typescript
type FilesApiConfig = {
  oauthToken: string    // OAuth token from session JWT
  baseUrl?: string      // Default: https://api.anthropic.com
  sessionId: string     // Session-specific directory
}
```

### Download Operations

- **`downloadFile(fileId, config)`** — Downloads a single file by ID with exponential backoff (3 retries, 500ms base delay)
- **`downloadAndSaveFile(attachment, config)`** — Downloads and saves to `{cwd}/{sessionId}/uploads/{relativePath}`
- **`downloadSessionFiles(files, config, concurrency=5)`** — Parallel download with concurrency limit

### Upload Operations

- **`uploadFile(filePath, relativePath, config)`** — Uploads a single file via multipart/form-data
  - Max file size: 500MB
  - Timeout: 120s
  - Non-retriable errors: 401, 403, 413, cancel
- **`uploadSessionFiles(files, config, concurrency=5)`** — Parallel upload with concurrency limit

### List Operations

- **`listFilesCreatedAfter(afterCreatedAt, config)`** — Lists files created after a timestamp with cursor-based pagination via `after_id`

### Path Safety

`buildDownloadPath()` prevents path traversal attacks by:
- Normalizing paths
- Rejecting paths starting with `..`
- Stripping redundant upload prefixes

### Beta Headers

Files API uses beta header: `files-api-2025-04-14,oauth-2025-04-20`

---

## Utilization API

### Overview (`usage.ts`)

Fetches rate limit utilization data from `/api/oauth/usage` for Claude.ai subscribers:

```typescript
type Utilization = {
  five_hour?: RateLimit | null
  seven_day?: RateLimit | null
  seven_day_opus?: RateLimit | null
  seven_day_sonnet?: RateLimit | null
  extra_usage?: ExtraUsage | null
}

type RateLimit = {
  utilization: number | null  // 0-100 percentage
  resets_at: string | null    // ISO 8601 timestamp
}
```

Skips the API call if OAuth token is expired or user lacks profile scope.

---

## Logging and Metrics

### API Logging (`logging.ts`)

- **`logAPIQuery()`** — Logs request start with model, tools, cache strategy
- **`logAPISuccessAndDuration()`** — Logs response with token usage, cost, duration, cache stats
- **`logAPIError()`** — Logs errors with classification and connection details

### Token Usage Tracking

- Input/output/cache token counts from API response
- Cost calculation via `calculateUSDCost()`
- Session cost tracking via `addToTotalSessionCost()`
- Duration tracking in bootstrap state

### Gateway Detection

Response headers are inspected to detect AI gateway proxies (LiteLLM, Helicone, Portkey, Cloudflare AI Gateway, Kong, Braintrust, Databricks) for accurate provider identification.

### Prompt Cache Break Detection

Tracks cache hit/miss patterns and detects cache breaks due to content changes, enabling optimization of cacheable prompt sections.
