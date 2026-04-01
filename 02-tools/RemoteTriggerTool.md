# RemoteTriggerTool

## Purpose

RemoteTriggerTool manages scheduled remote Claude Code agents (triggers) via the claude.ai CCR API. It provides a typed interface for listing, creating, updating, retrieving, and running remote triggers without exposing OAuth tokens to the shell. All authentication is handled in-process through the tool's internal OAuth flow.

## Location

- `restored-src/src/tools/RemoteTriggerTool/RemoteTriggerTool.ts` — Main tool definition (162 lines)
- `restored-src/src/tools/RemoteTriggerTool/prompt.ts` — Tool prompt, description, and constants
- `restored-src/src/tools/RemoteTriggerTool/UI.tsx` — UI rendering for tool use/result messages

## Key Exports

| Export | Description |
|--------|-------------|
| `RemoteTriggerTool` | The complete tool definition built via `buildTool()` |
| `Input` | Zod-inferred input type: `{ action, trigger_id?, body? }` |
| `Output` | Zod-inferred output type: `{ status, json }` |
| `REMOTE_TRIGGER_TOOL_NAME` | Constant: `'RemoteTrigger'` |
| `DESCRIPTION` | Tool description string |
| `PROMPT` | Tool prompt string |

## Input/Output Schemas

### Input Schema

```typescript
{
  action: 'list' | 'get' | 'create' | 'update' | 'run',
  trigger_id?: string,    // Required for: get, update, run. Regex: /^[\w-]+$/
  body?: Record<string, unknown>,  // Required for: create, update
}
```

### Output Schema

```typescript
{
  status: number,    // HTTP status code from API
  json: string,      // JSON response body (stringified)
}
```

## Actions

### list

Lists all remote triggers for the organization.

```json
{ "action": "list" }
```

- **Method**: GET
- **Endpoint**: `/v1/code/triggers`
- **Requires**: `trigger_id` — No, `body` — No

### get

Retrieves a specific trigger by ID.

```json
{ "action": "get", "trigger_id": "my-trigger-id" }
```

- **Method**: GET
- **Endpoint**: `/v1/code/triggers/{trigger_id}`
- **Requires**: `trigger_id` — Yes

### create

Creates a new remote trigger.

```json
{ "action": "create", "body": { "name": "daily-check", "schedule": "0 9 * * *" } }
```

- **Method**: POST
- **Endpoint**: `/v1/code/triggers`
- **Requires**: `body` — Yes

### update

Updates an existing trigger (partial update).

```json
{ "action": "update", "trigger_id": "my-trigger-id", "body": { "schedule": "0 10 * * *" } }
```

- **Method**: POST
- **Endpoint**: `/v1/code/triggers/{trigger_id}`
- **Requires**: `trigger_id` — Yes, `body` — Yes

### run

Manually triggers a scheduled trigger.

```json
{ "action": "run", "trigger_id": "my-trigger-id" }
```

- **Method**: POST
- **Endpoint**: `/v1/code/triggers/{trigger_id}/run`
- **Requires**: `trigger_id` — Yes

## API Communication

### Request Construction

```typescript
const base = `${getOauthConfig().BASE_API_URL}/v1/code/triggers`
const headers = {
  Authorization: `Bearer ${accessToken}`,
  'Content-Type': 'application/json',
  'anthropic-version': '2023-06-01',
  'anthropic-beta': 'ccr-triggers-2026-01-30',
  'x-organization-uuid': orgUUID,
}
```

### Authentication Flow

1. Check and refresh OAuth token if needed (`checkAndRefreshOAuthTokenIfNeeded`)
2. Retrieve access token (`getClaudeAIOAuthTokens`)
3. Resolve organization UUID (`getOrganizationUUID`)
4. If not authenticated: throw error directing user to run `/login`

### HTTP Client

Uses `axios` with:
- 20-second timeout
- Abort controller signal support
- `validateStatus: () => true` (accepts all status codes)
- Response returned as `{ status, json }`

## Feature Gating

The tool is only enabled when BOTH conditions are met:

```typescript
isEnabled() {
  return (
    getFeatureValue_CACHED_MAY_BE_STALE('tengu_surreal_dali', false) &&
    isPolicyAllowed('allow_remote_sessions')
  )
}
```

| Gate | Purpose |
|------|---------|
| `tengu_surreal_dali` | GrowthBook feature flag for remote triggers |
| `allow_remote_sessions` | Organization policy limit |

## Concurrency and Read-Only Classification

- **Concurrency safe**: Yes — all actions are idempotent API calls
- **Read-only**: `list` and `get` actions are classified as read-only
- **Not read-only**: `create`, `update`, `run` actions modify state

## Result Formatting

Results are formatted for the model using `mapToolResultToToolResultBlockParam`:

```
HTTP {status}
{json response}
```

The UI renders result messages showing the HTTP status and line count:

```
HTTP 200 (15 lines)
```

## Error Handling

| Error Condition | Response |
|----------------|----------|
| Not authenticated | "Not authenticated with a claude.ai account. Run /login and try again." |
| Cannot resolve org | "Unable to resolve organization UUID." |
| Missing trigger_id | Throws with specific action requirement message |
| Missing body | Throws with specific action requirement message |
| API error | Returns HTTP status + error JSON (not thrown, validateStatus accepts all) |

## Security Considerations

- **OAuth tokens never exposed to shell**: Tokens are added in-process via headers
- **No shell execution**: Uses axios HTTP client, not shell commands
- **Organization-scoped**: All operations include `x-organization-uuid` header
- **Beta API**: Uses `anthropic-beta: ccr-triggers-2026-01-30` header

## Dependencies

| Module | Purpose |
|--------|---------|
| `utils/auth.ts` | OAuth token management (check, refresh, retrieve) |
| `constants/oauth.ts` | OAuth configuration (BASE_API_URL) |
| `services/oauth/client.ts` | Organization UUID resolution |
| `services/policyLimits/` | Organization policy limit checking |
| `services/analytics/growthbook.ts` | Feature flag evaluation |
| `utils/slowOperations.ts` | jsonStringify for response formatting |
| `axios` | HTTP client for API requests |
| `zod/v4` | Input/output schema validation |
