# Analytics Service

## Purpose

The analytics service provides event tracking, telemetry collection, feature flag management (GrowthBook), and multi-backend event routing for Claude Code CLI. It supports two primary analytics backends: Datadog (third-party) and Anthropic's first-party event logging API, with a unified public API that queues events before sinks are attached.

## Location

- `restored-src/src/services/analytics/index.ts` — Public API, event queue, sink interface
- `restored-src/src/services/analytics/sink.ts` — Sink initialization and event routing
- `restored-src/src/services/analytics/config.ts` — Shared analytics disable logic
- `restored-src/src/services/analytics/growthbook.ts` — GrowthBook feature flags integration
- `restored-src/src/services/analytics/sinkKillswitch.ts` — Per-sink killswitch configuration
- `restored-src/src/services/analytics/metadata.ts` — Event metadata enrichment and sanitization
- `restored-src/src/services/analytics/datadog.ts` — Datadog log exporter
- `restored-src/src/services/analytics/firstPartyEventLogger.ts` — 1P event logging via OpenTelemetry
- `restored-src/src/services/analytics/firstPartyEventLoggingExporter.ts` — 1P event HTTP exporter with retry

## Key Exports

### Public API (`index.ts`)

#### Functions
- `logEvent(eventName, metadata)`: Synchronous event logging; queues events if no sink attached
- `logEventAsync(eventName, metadata)`: Asynchronous event logging
- `attachAnalyticsSink(sink)`: Attaches the analytics backend sink; drains queued events via `queueMicrotask`
- `stripProtoFields(metadata)`: Strips `_PROTO_*` keys from payloads destined for general-access storage

#### Types
- `AnalyticsSink`: Interface with `logEvent` and `logEventAsync` methods
- `AnalyticsMetadata_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS`: Marker type (`never`) forcing explicit verification that string values don't contain code/filepaths
- `AnalyticsMetadata_I_VERIFIED_THIS_IS_PII_TAGGED`: Marker type for values routed to PII-tagged proto columns

### Sink (`sink.ts`)

#### Functions
- `initializeAnalyticsSink()`: Attaches the sink implementation to the public API
- `initializeAnalyticsGates()`: Reads feature gate values during startup

### GrowthBook (`growthbook.ts`)

#### Functions
- `initializeGrowthBook()`: Creates and initializes the GrowthBook client with remote eval
- `getFeatureValue_CACHED_MAY_BE_STALE(feature, defaultValue)`: Non-blocking cached read (preferred method)
- `getFeatureValue_DEPRECATED(feature, defaultValue)`: Blocking read (deprecated)
- `checkSecurityRestrictionGate(gate)`: Security-critical gate check that waits for re-init
- `checkGate_CACHED_OR_BLOCKING(gate)`: Entitlement gate with fast-path cache and slow-path blocking
- `checkStatsigFeatureGate_CACHED_MAY_BE_STALE(gate)`: Migration helper for Statsig→GrowthBook
- `refreshGrowthBookAfterAuthChange()`: Destroys and recreates client after login/logout
- `onGrowthBookRefresh(listener)`: Subscribe to feature value refresh events
- `setGrowthBookConfigOverride(feature, value)`: Set/clear a single config override (ant-only)
- `getAllGrowthBookFeatures()`: Enumerate all known features and current values
- `resetGrowthBook()`: Full client reset (testing/cleanup)

#### Types
- `GrowthBookUserAttributes`: User attributes sent to GrowthBook for targeting (id, sessionId, deviceID, platform, org, subscription, etc.)

### Event Sampling (`firstPartyEventLogger.ts`)

#### Functions
- `getEventSamplingConfig()`: Returns the per-event sampling configuration
- `shouldSampleEvent(eventName)`: Determines if an event should be sampled; returns sample rate or null
- `logEventTo1P(eventName, metadata)`: Logs a first-party event
- `logGrowthBookExperimentTo1P(data)`: Logs a GrowthBook experiment assignment event
- `initialize1PEventLogging()`: Sets up the OpenTelemetry LoggerProvider and exporter
- `reinitialize1PEventLoggingIfConfigChanged()`: Rebuilds pipeline when batch config changes
- `shutdown1PEventLogging()`: Flushes and shuts down the 1P event logger
- `is1PEventLoggingEnabled()`: Checks if 1P event logging is enabled

### Metadata (`metadata.ts`)

#### Functions
- `getEventMetadata(options)`: Builds core event metadata (model, session, env context, etc.)
- `to1PEventFormat(metadata, userMetadata, additionalMetadata)`: Converts to snake_case 1P API format
- `sanitizeToolNameForAnalytics(toolName)`: Redacts MCP tool names to 'mcp_tool' for PII protection
- `extractMcpToolDetails(toolName)`: Parses MCP server/tool names from `mcp__<server>__<tool>` format
- `extractSkillName(toolName, input)`: Extracts skill name from Skill tool input
- `extractToolInputForTelemetry(input)`: Serializes and truncates tool input for OTel events
- `getFileExtensionForAnalytics(filePath)`: Extracts and sanitizes file extensions
- `getFileExtensionsFromBashCommand(command)`: Extracts file extensions from bash commands
- `mcpToolDetailsForAnalytics(toolName, serverType, baseUrl)`: Conditional MCP detail extraction

#### Types
- `EventMetadata`: Core event metadata shared across all analytics systems
- `EnvContext`: Environment context (platform, arch, version, CI status, etc.)
- `ProcessMetrics`: Process resource usage (memory, CPU, uptime)
- `FirstPartyEventLoggingMetadata`: Complete 1P event format with env, core, auth, and additional fields

### Datadog (`datadog.ts`)

#### Functions
- `trackDatadogEvent(eventName, properties)`: Sends an event to Datadog (batched)
- `initializeDatadog()`: Memoized initialization
- `shutdownDatadog()`: Flushes remaining logs and clears timers

### Killswitch (`sinkKillswitch.ts`)

#### Functions
- `isSinkKilled(sink)`: Checks if a specific analytics sink is disabled via GrowthBook config

## Dependencies

### Internal Dependencies
- `bootstrap/state.js` — Session state (sessionId, isInteractive, trust status)
- `utils/config.js` — Global config read/write, trust dialog state
- `utils/auth.js` — OAuth token checks, subscription type, Claude.ai subscriber detection
- `utils/http.js` — Auth header generation
- `utils/user.js` — User data (deviceId, email, organization UUID)
- `utils/env.js` — Environment detection (platform, CI, runtimes)
- `utils/envUtils.js` — Environment variable helpers, config directory paths
- `utils/debug.js` — Debug logging for ant builds
- `utils/errors.js` — Error utilities
- `utils/log.js` — Error logging
- `utils/signal.js` — Signal/pub-sub for GrowthBook refresh listeners
- `constants/keys.js` — GrowthBook client key
- `constants/oauth.js` — OAuth configuration constants

### External Dependencies
- `@growthbook/growthbook` — GrowthBook SDK for feature flags and experiments
- `@opentelemetry/api-logs` — OpenTelemetry Logs API
- `@opentelemetry/sdk-logs` — OpenTelemetry SDK for log batching
- `@opentelemetry/resources` — OpenTelemetry resource configuration
- `@opentelemetry/core` — OpenTelemetry export result types
- `@opentelemetry/semantic-conventions` — Standard semantic attribute names
- `axios` — HTTP client for Datadog and 1P API calls
- `lodash-es` — `memoize`, `isEqual` for caching and comparison
- `bun:bundle` — Feature flag detection (Bun-specific)

## Implementation Details

### Event Tracking System

The analytics system uses a **queue-then-drain** pattern to handle events logged before the sink is initialized:

1. **Event Queue**: Events logged before `attachAnalyticsSink()` are stored in an in-memory array (`eventQueue`)
2. **Sink Attachment**: When the sink attaches, queued events are drained asynchronously via `queueMicrotask()` to avoid blocking startup
3. **Idempotent Attachment**: `attachAnalyticsSink()` is safe to call multiple times — subsequent calls are no-ops
4. **Dual Mode**: Both synchronous (`logEvent`) and asynchronous (`logEventAsync`) logging are supported

The sink implementation (`sink.ts`) routes every event to:
- **Datadog** (if the `tengu_log_datadog_events` gate is enabled)
- **1P Event Logging** (always, unless killed by killswitch)

### Telemetry Collection

#### Metadata Enrichment

Every event is enriched with core metadata via `getEventMetadata()`:

- **Model information**: Current model name and beta flags
- **Session identification**: Session ID, user type, client type
- **Environment context**: Platform, architecture, Node.js version, terminal, package managers, runtimes, CI status, version, build time
- **Process metrics**: Memory usage (RSS, heap), CPU usage with delta tracking, uptime
- **Agent identification**: Swarm/team agent ID, parent session ID, agent type (teammate/subagent/standalone)
- **Subscription tier**: OAuth subscription type for DAU-by-tier analytics
- **Repo remote hash**: SHA256-based hash for server-side joining

#### PII Protection

Multiple layers of PII protection:

1. **Type-level guards**: `AnalyticsMetadata_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS` (`never` type) requires explicit casting
2. **Tool name sanitization**: MCP tool names (`mcp__<server>__<tool>`) are redacted to `'mcp_tool'` unless the logging gate passes
3. **File extension limits**: Extensions >10 chars are replaced with `'other'` to prevent hash-based filename leakage
4. **Tool input truncation**: Strings truncated to 128 chars, objects limited to 20 keys, max depth 2, JSON capped at 4KB
5. **Proto field stripping**: `_PROTO_*` keys are stripped from Datadog payloads (general-access backend) but preserved for 1P (privileged BQ columns)

#### Sampling

Events can be sampled via the `tengu_event_sampling_config` GrowthBook dynamic config:
- Sample rate 0: Event dropped entirely
- Sample rate 0–1: Random sampling, rate added to metadata
- Sample rate ≥1 or no config: 100% logging

### GrowthBook Feature Flags

#### Client Architecture

GrowthBook uses **remote evaluation** — feature values are computed server-side and sent to the client, not evaluated locally:

1. **Client Creation**: `getGrowthBookClient()` creates a memoized GrowthBook instance with auth headers, user attributes, and `remoteEval: true`
2. **Initialization**: `initializeGrowthBook()` awaits client init (5s timeout), processes the remote eval payload, and sets up periodic refresh
3. **Auth-aware**: Client is destroyed and recreated after auth changes (login/logout) because `apiHostRequestHeaders` cannot be updated post-creation

#### Feature Value Resolution Order

1. **Environment variable overrides** (`CLAUDE_INTERNAL_FC_OVERRIDES`) — ant-only, for eval harnesses
2. **Config overrides** (`growthBookOverrides` in global config) — ant-only, set via `/config Gates` tab
3. **In-memory remote eval payload** (`remoteEvalFeatureValues` Map) — authoritative after init
4. **Disk cache** (`cachedGrowthBookFeatures` in `~/.claude.json`) — survives process restarts

#### API Methods

| Method | Blocking? | Use Case |
|--------|-----------|----------|
| `getFeatureValue_CACHED_MAY_BE_STALE` | No | Startup-critical paths, sync contexts, render loops |
| `getFeatureValue_DEPRECATED` | Yes | Legacy code (being migrated) |
| `checkSecurityRestrictionGate` | Async (waits for re-init) | Security-critical gates after auth changes |
| `checkGate_CACHED_OR_BLOCKING` | Async (fast cache, slow blocking) | User-invoked features gated on subscription/org |
| `checkStatsigFeatureGate_CACHED_MAY_BE_STALE` | No | Statsig→GrowthBook migration only |

#### Periodic Refresh

- **External builds**: Every 6 hours (matches Statsig's interval)
- **Ant builds**: Every 20 minutes
- Uses `client.refreshFeatures()` (light refresh, preserves client state)
- Notifies `onGrowthBookRefresh` subscribers after successful refresh

#### Workarounds

The code contains several SDK workarounds:
- **Malformed API response**: API returns `{ "value": ... }` but SDK expects `{ "defaultValue": ... }` — transformed in `processRemoteEvalPayload()`
- **setForcedFeatures unreliable**: Custom cache (`remoteEvalFeatureValues` Map) used instead of SDK's forced features
- **Empty payload guard**: Prevents `{features: {}}` server bug from clearing all flags on disk

### Analytics Configuration

#### Disable Conditions (`isAnalyticsDisabled()`)

Analytics is disabled when ANY of the following is true:
- `NODE_ENV === 'test'`
- `CLAUDE_CODE_USE_BEDROCK` is truthy
- `CLAUDE_CODE_USE_VERTEX` is truthy
- `CLAUDE_CODE_USE_FOUNDRY` is truthy
- Privacy level disables telemetry (`isTelemetryDisabled()`)

#### Feedback Survey

`isFeedbackSurveyDisabled()` has a narrower scope — it does NOT block on 3P providers (Bedrock/Vertex/Foundry) since the survey is a local UI prompt with no transcript data.

### Event Sinks

#### Datadog Sink

- **Endpoint**: `https://http-intake.logs.us5.datadoghq.com/api/v2/logs`
- **Batching**: Up to 100 events per batch, 15s flush interval
- **Allowed events**: Whitelist of ~50 event names (API errors, tool usage, OAuth events, etc.)
- **Cardinality reduction**: Model names normalized to canonical names, MCP tool names to 'mcp', dev versions truncated
- **User buckets**: SHA256 hash of user ID into 30 buckets for unique user estimation without exposing IDs
- **Tags**: High-cardinality fields (model, platform, version, userType, etc.) sent as Datadog tags for filtering

#### First-Party Event Logging Sink

- **Endpoint**: `/api/event_logging/batch` on `api.anthropic.com` (or staging)
- **Protocol**: OpenTelemetry Logs API with custom `FirstPartyEventLoggingExporter`
- **Batching**: Configurable via GrowthBook dynamic config (`tengu_1p_event_batch_config`)
  - Default: 200 events per batch, 10s export interval, 8192 max queue size
- **Resilience**:
  - Append-only JSONL file storage for failed events (`~/.claude/telemetry/1p_failed_events.*`)
  - Quadratic backoff retry (base * attempts², max 30s, 8 max attempts)
  - Startup retry of previous session's failed batches
  - Chunking of large event sets
  - Auth fallback: retries without auth on 401 errors
- **Event types**: `ClaudeCodeInternalEvent` (standard events) and `GrowthbookExperimentEvent` (experiment exposures)
- **Proto serialization**: Uses protobuf-generated `ClaudeCodeInternalEvent.toJSON()` for wire format

#### Sink Killswitch

Per-sink killswitch via GrowthBook config `tengu_frond_boric`:
```json
{ "datadog": true, "firstParty": false }
```
- Checked at per-event dispatch time (not at init) to allow runtime toggling
- Also checked inside the 1P exporter's `sendBatchWithRetry` to stop retries when killed

### Privacy Considerations

1. **Opt-out support**: Respects global telemetry opt-out via privacy level settings
2. **Third-party provider isolation**: No analytics sent when using Bedrock, Vertex, or Foundry
3. **PII-tagged fields**: `_PROTO_*` keys route to privileged BigQuery columns with access controls; stripped before Datadog fanout
4. **MCP tool name redaction**: User-configured MCP server names are PII-medium and redacted by default
5. **File path protection**: Type system prevents accidental logging of file paths; file extensions sanitized
6. **Tool input truncation**: Deep nesting and long strings truncated to bound output size
7. **User bucketing**: Datadog uses hashed user buckets (30 buckets) instead of raw user IDs
8. **Model name normalization**: External user model names normalized to reduce cardinality and prevent leaking custom model identifiers
9. **Trust-gated auth headers**: Auth headers not sent to GrowthBook or 1P endpoints until workspace trust is established

## Data Flow

```
Call Site (logEvent/logEventAsync)
  │
  ├─ No sink attached? → Queue event
  │
  ├─ Sink attached → sink.logEvent()
  │    │
  │    ├─ shouldSampleEvent() → Drop if sampled out
  │    │
  │    ├─ isSinkKilled('datadog')? → Skip Datadog
  │    │    └─ trackDatadogEvent() → Batch → Datadog HTTP API
  │    │
  │    └─ logEventTo1P()
  │         │
  │         ├─ getEventMetadata() → Enrich with env, session, process info
  │         ├─ OpenTelemetry LoggerProvider → BatchLogRecordProcessor
  │         └─ FirstPartyEventLoggingExporter.export()
  │              │
  │              ├─ Transform logs to proto format (ClaudeCodeInternalEvent.toJSON)
  │              ├─ sendBatchWithRetry() → POST /api/event_logging/batch
  │              │    ├─ Auth headers (with 401 fallback)
  │              │    └─ Chunk into maxBatchSize batches
  │              ├─ Success → Reset backoff, retry queued failures
  │              └─ Failure → Append to disk JSONL, schedule backoff retry
```

## Configuration

### Environment Variables

| Variable | Purpose |
|----------|---------|
| `CLAUDE_INTERNAL_FC_OVERRIDES` | JSON object overriding GrowthBook features (ant-only, USER_TYPE='ant') |
| `CLAUDE_CODE_GB_BASE_URL` | Custom GrowthBook API base URL (ant-only) |
| `ANTHROPIC_BASE_URL` | API base URL; host used as GrowthBook attribute for enterprise proxy targeting |
| `OTEL_LOGS_EXPORT_INTERVAL` | Override for 1P log export interval (ms) |
| `OTEL_LOG_TOOL_DETAILS` | Enable detailed tool name logging for OTel events |
| `CLAUDE_CODE_DATADOG_FLUSH_INTERVAL_MS` | Override for Datadog flush interval (ms) |
| `NODE_ENV` | 'test' disables all analytics |
| `CLAUDE_CODE_USE_BEDROCK/VERTEX/FOUNDRY` | Disables analytics for third-party providers |

### GrowthBook Dynamic Configs

| Config Name | Purpose |
|-------------|---------|
| `tengu_event_sampling_config` | Per-event sampling rates |
| `tengu_1p_event_batch_config` | 1P exporter batch size, delay, endpoint, auth settings |
| `tengu_frond_boric` | Per-sink killswitch (datadog/firstParty) |
| `tengu_log_datadog_events` | Gate enabling Datadog event dispatch |

## Error Handling

- **GrowthBook init failures**: Caught and logged; disk cache used as fallback
- **1P export failures**: Events written to disk JSONL, retried with quadratic backoff, dropped after max attempts
- **Datadog failures**: Silently logged; no retry (fire-and-forget batching)
- **Auth failures**: 1P exporter retries without auth on 401; expired OAuth tokens skip auth proactively
- **Metadata enrichment failures**: Swallowed with error logging; event may be emitted with partial metadata

## Related Modules

- [OAuth Service](./oauth-service.md) — Authentication and token management
- [Auth Utils](../05-utils/auth.md) — API key and OAuth token sourcing
- [Secure Storage](../05-utils/secure-storage.md) — Credential storage backend
- [Config Utils](../05-utils/config.md) — Global configuration management
