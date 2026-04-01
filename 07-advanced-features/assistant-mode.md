# Assistant Mode & Session History

## Purpose

The assistant module provides session history retrieval capabilities, enabling the loading and pagination of historical session events from remote storage. This supports session resumption, review, and analysis of past interactions.

## Location

`restored-src/src/assistant/sessionHistory.ts`

## Key Exports

### Types

- `HistoryPage`: Represents a paginated set of session events with `events` (SDKMessage[]), `firstId` (oldest event ID cursor), and `hasMore` (whether older events exist).
- `HistoryAuthCtx`: Authentication context containing `baseUrl` and `headers`, pre-computed once and reused across page fetches.
- `SessionEventsResponse`: Raw API response shape with `data`, `has_more`, `first_id`, and `last_id` fields.

### Constants

- `HISTORY_PAGE_SIZE`: Default page size of 100 events per fetch.

### Functions

- `createHistoryAuthCtx(sessionId)`: Prepares authentication headers and base URL once for a given session. Returns a reusable `HistoryAuthCtx` with OAuth headers, organization UUID, and the beta header `ccr-byoc-2025-07-29`.
- `fetchLatestEvents(ctx, limit)`: Fetches the newest `limit` events, anchored to the latest point. Returns events in chronological order; `hasMore=true` means older events exist.
- `fetchOlderEvents(ctx, beforeId, limit)`: Fetches events immediately before the given `beforeId` cursor, enabling backward pagination through session history.

## Dependencies

### Internal Dependencies

- `../constants/oauth.js` — OAuth configuration (base API URL)
- `../entrypoints/agentSdkTypes.js` — `SDKMessage` type definition
- `../utils/debug.js` — `logForDebugging()` for error logging
- `../utils/teleport/api.js` — `prepareApiRequest()` for auth tokens and `getOAuthHeaders()` for header construction

### External Dependencies

- `axios` — HTTP client for API requests

## Implementation Details

### Core Logic

The module implements a cursor-based pagination system for session events:

1. **Auth context preparation** — `createHistoryAuthCtx()` performs the OAuth token fetch and header construction once, returning a reusable context object. This avoids redundant auth lookups on every page fetch.

2. **Page fetching** — `fetchPage()` is the internal workhorse that makes the HTTP GET request with configurable params. It uses `validateStatus: () => true` to prevent axios from throwing on non-2xx responses, instead returning `null` for error handling upstream.

3. **Newest-first pagination** — `fetchLatestEvents()` uses `anchor_to_latest: true` to get the most recent events. The `has_more` flag indicates whether older events exist beyond this page.

4. **Backward pagination** — `fetchOlderEvents()` uses a `before_id` cursor to fetch the next page of older events, enabling infinite scroll backward through session history.

### Authentication Flow

```
createHistoryAuthCtx(sessionId)
    ↓
prepareApiRequest() → accessToken, orgUUID
    ↓
getOAuthHeaders(accessToken)
    ↓
Build headers:
  - OAuth bearer token
  - anthropic-beta: ccr-byoc-2025-07-29
  - x-organization-uuid: orgUUID
    ↓
Return { baseUrl, headers }
```

### Error Handling

- Network errors are caught by `.catch(() => null)` on the axios call
- Non-200 responses are logged via `logForDebugging()` with the HTTP status code
- Invalid response data (`resp.data.data` not an array) defaults to an empty array
- The `validateStatus: () => true` pattern ensures all HTTP responses are handled gracefully rather than throwing

## Data Flow

```
Session Resume / History Request
    ↓
createHistoryAuthCtx(sessionId)
    ↓ { baseUrl, headers }
    ↓
fetchLatestEvents(ctx, 100)
    ↓
  GET /v1/sessions/{sessionId}/events?limit=100&anchor_to_latest=true
    ↓
HistoryPage { events[], firstId, hasMore }
    ↓
(if hasMore) fetchOlderEvents(ctx, firstId, 100)
    ↓
  GET /v1/sessions/{sessionId}/events?limit=100&before_id={firstId}
    ↓
Next HistoryPage...
```

## Integration Points

- **Session resumption** — History pages are loaded when resuming a previous session to restore conversation context
- **Session review** — Users can browse past session events for analysis
- **Teleport API** — Uses the teleport utility layer for authentication and request preparation
- **OAuth system** — Depends on valid OAuth tokens for API access

## Configuration

| Parameter | Default | Purpose |
|---|---|---|
| `HISTORY_PAGE_SIZE` | 100 | Events per page |
| `timeout` | 15000ms | HTTP request timeout |
| `anchor_to_latest` | true (fetchLatestEvents) | Anchor to newest events |
| `before_id` | cursor (fetchOlderEvents) | Pagination cursor |

## Testing

- The `fetchPage()` function accepts a `label` parameter for clear debug logging, aiding test diagnostics
- Error paths (network failure, non-200 status) return `null` rather than throwing, making them straightforward to test
- The auth context can be mocked independently of the fetch logic

## Related Modules

- [Session Commands](../03-commands/session-commands.md) — Commands that may use session history
- [OAuth Service](../04-services/oauth-service.md) — Authentication backing
- [Teleport Utils](../05-utils/teleport-utils.md) — API request preparation

## Notes

- The module name "assistant" suggests this may be part of a broader assistant mode feature, though only session history is present in the restored source
- The `ccr-byoc-2025-07-29` beta header indicates this uses a specific API version for session event retrieval
- The pagination design (anchor-to-latest + before-id cursor) is optimized for the common case of viewing recent events first, with optional backward traversal
