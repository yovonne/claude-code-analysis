# Telemetry Utilities

## Purpose

The telemetry utilities module implements comprehensive observability for Claude Code using OpenTelemetry (OTel). It provides metrics, logs, traces, and events collection with multiple exporter backends (console, OTLP, Prometheus, BigQuery, Perfetto). It also includes plugin-specific telemetry with privacy-preserving redaction and skill loading event tracking.

## Location

- `restored-src/src/utils/telemetry/instrumentation.ts` — OTel SDK initialization and configuration
- `restored-src/src/utils/telemetry/events.ts` — OTel event emission
- `restored-src/src/utils/telemetry/sessionTracing.ts` — Span management for interactions, LLM requests, tools, hooks
- `restored-src/src/utils/telemetry/betaSessionTracing.ts` — Detailed beta tracing with content logging
- `restored-src/src/utils/telemetry/perfettoTracing.ts` — Chrome Trace Event format tracing (ant-only)
- `restored-src/src/utils/telemetry/bigqueryExporter.ts` — BigQuery metrics exporter
- `restored-src/src/utils/telemetry/logger.ts` — OTel diagnostic logger
- `restored-src/src/utils/telemetry/pluginTelemetry.ts` — Plugin lifecycle telemetry with privacy controls
- `restored-src/src/utils/telemetry/skillLoadedEvent.ts` — Skill loading event emission

## Key Exports

### Instrumentation (`instrumentation.ts`)

#### Functions
- `bootstrapTelemetry()`: Early-stage setup that maps ANT_ prefixed env vars to standard OTEL vars and sets delta temporality default
- `parseExporterTypes(value)`: Parses comma-separated exporter type strings, filtering out 'none'
- `isTelemetryEnabled()`: Checks `CLAUDE_CODE_ENABLE_TELEMETRY` env var
- `initializeTelemetry()`: Main initialization — creates MeterProvider, LoggerProvider, TracerProvider with configured exporters. Supports OTLP (grpc, http/json, http/protobuf), console, Prometheus, and BigQuery exporters. Handles proxy, mTLS, and CA certificate configuration
- `flushTelemetry()`: Force-flush all pending telemetry data (used before logout/org switching)

#### Exporter Configuration
- Dynamic protocol selection for OTLP exporters (grpc, http/json, http/protobuf)
- Proxy support via HttpsProxyAgent with mTLS and CA cert configuration
- Dynamic header refresh via `otelHeadersHelper` settings
- BigQuery metrics enabled for API customers, C4E users, and team users

### Events (`events.ts`)

#### Functions
- `logOTelEvent(eventName, metadata)`: Emits an OTel event with attributes (sequence number, timestamp, prompt ID, workspace paths)
- `redactIfDisabled(content)`: Returns content or '<REDACTED>' based on `OTEL_LOG_USER_PROMPTS` env var

### Session Tracing (`sessionTracing.ts`)

#### Functions
- `isEnhancedTelemetryEnabled()`: Checks feature flag + env var + GrowthBook gate
- `startInteractionSpan(userPrompt)`: Root span for user request → Claude response cycle
- `endInteractionSpan()`: Ends interaction span with duration
- `startLLMRequestSpan(model, newContext?, messagesForAPI?, fastMode?)`: Span for API calls
- `endLLMRequestSpan(span?, metadata?)`: Ends LLM span with token counts, TTFT, errors, retry info
- `startToolSpan(toolName, toolAttributes?, toolInput?)`: Span for tool invocations
- `endToolSpan(toolResult?, resultTokens?)`: Ends tool span with result
- `startToolBlockedOnUserSpan()`, `endToolBlockedOnUserSpan(decision?, source?)`: Span for permission wait time
- `startToolExecutionSpan()`, `endToolExecutionSpan(metadata?)`: Span for actual tool execution
- `startHookSpan(hookEvent, hookName, numHooks, hookDefinitions)`, `endHookSpan(span, metadata?)`: Span for hook execution (beta tracing only)
- `addToolContentEvent(eventName, attributes)`: Add span event with tool content (gated on `OTEL_LOG_TOOL_CONTENT`)
- `getCurrentSpan()`: Get current active span
- `executeInSpan(spanName, fn, attributes?)`: Execute async function within a span context

#### Design
- Uses AsyncLocalStorage for interaction and tool context tracking
- WeakRef-based span storage with 30-minute TTL cleanup interval
- Parallel request support: `endLLMRequestSpan` accepts explicit span parameter to avoid mismatched responses

### Beta Session Tracing (`betaSessionTracing.ts`)

#### Functions
- `isBetaTracingEnabled()`: Checks `ENABLE_BETA_TRACING_DETAILED` + `BETA_TRACING_ENDPOINT` + GrowthBook gate
- `clearBetaTracingState()`: Clear hash tracking after compaction
- `truncateContent(content, maxSize?)`: Truncate to 60KB (Honeycomb limit)
- `addBetaInteractionAttributes(span, userPrompt)`: Add new_context to interaction span
- `addBetaLLMRequestAttributes(span, newContext?, messagesForAPI?)`: Log system prompt (once per hash), tool schemas, incremental new_context with hash-based deduplication
- `addBetaLLMResponseAttributes(endAttributes, metadata?)`: Add model_output (all users) and thinking_output (ant-only)
- `addBetaToolInputAttributes(span, toolName, toolInput)`: Add tool input to span
- `addBetaToolResultAttributes(endAttributes, toolName, toolResult)`: Add tool result to span

#### Design
- Hash-based deduplication: system prompts and tool schemas logged once per unique SHA-256 hash
- Incremental context: tracks last reported message hash per querySource (agent) to send only deltas
- System reminders separated from regular content in new_context
- Visibility rules: thinking output is ant-only; everything else is visible to all users

### Perfetto Tracing (`perfettoTracing.ts`)

#### Functions
- `initializePerfettoTracing()`: Initialize from `CLAUDE_CODE_PERFETTO_TRACE` env var
- `isPerfettoTracingEnabled()`: Check if enabled
- `registerAgent(agentId, agentName, parentAgentId?)`: Register agent in trace hierarchy
- `unregisterAgent(agentId)`: Remove agent from trace
- `startLLMRequestPerfettoSpan(args)`, `endLLMRequestPerfettoSpan(spanId, metadata)`: API call spans with TTFT, TTLT, cache stats, retry sub-spans
- `startToolPerfettoSpan(toolName, args?)`, `endToolPerfettoSpan(spanId, metadata?)`: Tool execution spans
- `startUserInputPerfettoSpan(context?)`, `endUserInputPerfettoSpan(spanId, metadata?)`: User input wait spans
- `startInteractionPerfettoSpan(userPrompt?)`, `endInteractionPerfettoSpan(spanId)`: Interaction spans
- `emitPerfettoInstant(name, category, args?)`: Instant marker events
- `emitPerfettoCounter(name, values)`: Counter events
- `getPerfettoEvents()`, `resetPerfettoTracer()`: Testing helpers

#### Design
- Chrome Trace Event format compatible with ui.perfetto.dev
- Agent hierarchy visualization with parent-child relationships
- Derived metrics: ITPS (input tokens/sec), OTPS (output tokens/sec), cache hit rate
- Sub-spans for Request Setup, First Token (TTFT), Sampling, and retry attempts
- Event cap at 100,000 with oldest-half eviction
- Periodic write support via `CLAUDE_CODE_PERFETTO_WRITE_INTERVAL_S`
- Ant-only feature (eliminated from external builds)

### BigQuery Exporter (`bigqueryExporter.ts`)

#### Class
- `BigQueryMetricsExporter`: Implements `PushMetricExporter` for Anthropic's internal metrics API
  - Transforms OTel ResourceMetrics to internal payload format
  - Sends to `api.anthropic.com/api/claude_code/metrics` (or ant endpoint)
  - Includes customer type, subscription type, and resource attributes
  - Uses delta aggregation temporality
  - Respects trust dialog and organization opt-out settings

### Plugin Telemetry (`pluginTelemetry.ts`)

#### Types
- `TelemetryPluginScope`: `'official' | 'org' | 'user-local' | 'default-bundle'`
- `EnabledVia`: `'user-install' | 'org-policy' | 'default-enable' | 'seed-mount'`
- `InvocationTrigger`: `'user-slash' | 'claude-proactive' | 'nested-skill'`
- `SkillExecutionContext`: `'fork' | 'inline' | 'remote'`
- `InstallSource`: `'cli-explicit' | 'ui-discover' | 'ui-suggestion' | 'deep-link'`
- `PluginCommandErrorCategory`: `'network' | 'not-found' | 'permission' | 'validation' | 'unknown'`

#### Functions
- `hashPluginId(name, marketplace?)`: Opaque per-plugin aggregation key (SHA-256, 16 chars)
- `getTelemetryPluginScope(name, marketplace, managedNames)`: Determine plugin origin scope
- `getEnabledVia(plugin, managedNames, seedDirs)`: Determine how plugin was enabled
- `buildPluginTelemetryFields(name, marketplace, managedNames?)`: Build privacy-preserving telemetry fields (hash, scope, redacted name/marketplace)
- `buildPluginCommandTelemetryFields(pluginInfo, managedNames?)`: Build fields for command/skill invocations
- `logPluginsEnabledForSession(plugins, managedNames, seedDirs)`: Emit `tengu_plugin_enabled_for_session` per plugin
- `classifyPluginCommandError(error)`: Map error messages to bounded categories
- `logPluginLoadErrors(errors, managedNames)`: Emit `tengu_plugin_load_failed` per error

#### Design
- Twin-column privacy: raw values go to PII-tagged `_PROTO_*` columns; redacted values show real names only for official/builtin plugins, 'third-party' otherwise
- `plugin_id_hash` provides opaque per-plugin aggregation without exposing names

### Skill Loaded Event (`skillLoadedEvent.ts`)

#### Functions
- `logSkillsLoaded(cwd, contextWindowTokens)`: Emit `tengu_skill_loaded` event for each skill at session startup, including source, loadedFrom, budget, and kind

### Logger (`logger.ts`)

#### Class
- `ClaudeCodeDiagLogger`: Implements OTel `DiagLogger`, routing errors and warnings to the application's error/debug logging system. Info/debug/verbose are no-ops.

## Dependencies

- `@opentelemetry/api`, `@opentelemetry/api-logs` — OTel core APIs
- `@opentelemetry/sdk-metrics`, `@opentelemetry/sdk-logs`, `@opentelemetry/sdk-trace-base` — OTel SDKs
- `@opentelemetry/resources`, `@opentelemetry/semantic-conventions` — Resource detection and conventions
- Dynamic imports of OTLP exporters (grpc, http, proto) and Prometheus exporter
- `axios` — HTTP client for BigQuery exporter
- `https-proxy-agent` — Proxy support for OTLP exporters

## Design Notes

- **Multi-path initialization**: Beta tracing (`BETA_TRACING_ENDPOINT`) follows a separate code path from standard OTEL telemetry, using its own endpoint and enabling detailed content logging
- **Lazy exporter imports**: OTLP protocol exporters are dynamically imported to avoid loading ~1.2MB of unused packages on every startup
- **Console exporter stripping**: In stream-json mode, console exporters are automatically removed from config to prevent breaking the SDK message channel
- **Graceful shutdown**: All providers are flushed and shut down with configurable timeout (`CLAUDE_CODE_OTEL_SHUTDOWN_TIMEOUT_MS`, default 2000ms)
- **Security**: Plugin telemetry uses hash-based aggregation and redacted names to protect user-defined plugin identities while still enabling analytics
