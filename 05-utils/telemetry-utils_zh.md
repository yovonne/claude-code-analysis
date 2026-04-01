# 遥测工具

## 目的

遥测工具模块使用 OpenTelemetry（OTel）实现 Claude Code 的全面可观测性。它提供指标、日志、追踪和事件收集，具有多个导出器后端（控制台、OTLP、Prometheus、BigQuery、Perfetto）。还包括带隐私保护重写的插件特定遥测和 skill 加载事件跟踪。

## 位置

- `restored-src/src/utils/telemetry/instrumentation.ts` — OTel SDK 初始化和配置
- `restored-src/src/utils/telemetry/events.ts` — OTel 事件发射
- `restored-src/src/utils/telemetry/sessionTracing.ts` — 交互、LLM 请求、工具、hooks 的跨度管理
- `restored-src/src/utils/telemetry/betaSessionTracing.ts` — 带内容日志的详细 beta 追踪
- `restored-src/src/utils/telemetry/perfettoTracing.ts` — Chrome Trace Event 格式追踪（仅 ant）
- `restored-src/src/utils/telemetry/bigqueryExporter.ts` — BigQuery 指标导出器
- `restored-src/src/utils/telemetry/logger.ts` — OTel 诊断日志
- `restored-src/src/utils/telemetry/pluginTelemetry.ts` — 带隐私控制的插件生命周期遥测
- `restored-src/src/utils/telemetry/skillLoadedEvent.ts` — Skill 加载事件发射

## 主要导出

### 插桩（`instrumentation.ts`）

#### 函数
- `bootstrapTelemetry()`: 早期设置，将 ANT_ 前缀的环境变量映射到标准 OTEL 变量并设置 delta 临时性默认值
- `parseExporterTypes(value)`: 解析逗号分隔的导出器类型字符串，过滤掉 'none'
- `isTelemetryEnabled()`: 检查 `CLAUDE_CODE_ENABLE_TELEMETRY` 环境变量
- `initializeTelemetry()`: 主要初始化 — 使用配置的导出器创建 MeterProvider、LoggerProvider、TracerProvider。支持 OTLP（grpc、http/json、http/protobuf）、控制台、Prometheus 和 BigQuery 导出器。处理代理、mTLS 和 CA 证书配置
- `flushTelemetry()`: 强制刷新所有待处理遥测数据（在注销/组织切换前使用）

#### 导出器配置
- OTLP 导出器的动态协议选择（grpc、http/json、http/protobuf）
- 通过 HttpsProxyAgent 支持代理，带 mTLS 和 CA 证书配置
- 通过 `otelHeadersHelper` 设置动态头刷新
- 为 API 客户、C4E 用户和团队用户启用 BigQuery 指标

### 事件（`events.ts`）

#### 函数
- `logOTelEvent(eventName, metadata)`: 发出带属性的 OTel 事件（序列号、时间戳、提示 ID、工作区路径）
- `redactIfDisabled(content)`: 基于 `OTEL_LOG_USER_PROMPTS` 环境变量返回内容或 '<REDACTED>'

### 会话追踪（`sessionTracing.ts`）

#### 函数
- `isEnhancedTelemetryEnabled()`: 检查特性标志 + 环境变量 + GrowthBook 门控
- `startInteractionSpan(userPrompt)`: 用户请求 → Claude 响应周期的根跨度
- `endInteractionSpan()`: 用持续时间结束交互跨度
- `startLLMRequestSpan(model, newContext?, messagesForAPI?, fastMode?)`: API 调用的跨度
- `endLLMRequestSpan(span?, metadata?)`: 用令牌计数、TTFT、错误、重试信息结束 LLM 跨度
- `startToolSpan(toolName, toolAttributes?, toolInput?)`: 工具调用的跨度
- `endToolSpan(toolResult?, resultTokens?)`: 用结果结束工具跨度
- `startToolBlockedOnUserSpan()`、`endToolBlockedOnUserSpan(decision?, source?)`: 权限等待时间的跨度
- `startToolExecutionSpan()`、`endToolExecutionSpan(metadata?)`: 实际工具执行的跨度
- `startHookSpan(hookEvent, hookName, numHooks, hookDefinitions)`、`endHookSpan(span, metadata?)`: hook 执行的跨度（仅 beta 追踪）
- `addToolContentEvent(eventName, attributes)`: 添加带工具内容的跨度事件（门控在 `OTEL_LOG_TOOL_CONTENT`）
- `getCurrentSpan()`: 获取当前活动跨度
- `executeInSpan(spanName, fn, attributes?)`: 在跨度上下文中执行异步函数

#### 设计
- 使用 AsyncLocalStorage 进行交互和工具上下文跟踪
- WeakRef 基于跨度存储，带 30 分钟 TTL 清理间隔
- 并行请求支持：`endLLMRequestSpan` 接受显式跨度参数以避免不匹配的响应

### Beta 会话追踪（`betaSessionTracing.ts`）

#### 函数
- `isBetaTracingEnabled()`: 检查 `ENABLE_BETA_TRACING_DETAILED` + `BETA_TRACING_ENDPOINT` + GrowthBook 门控
- `clearBetaTracingState()`: 在压缩后清除哈希跟踪
- `truncateContent(content, maxSize?)`: 截断到 60KB（Honeycomb 限制）
- `addBetaInteractionAttributes(span, userPrompt)`: 将 new_context 添加到交互跨度
- `addBetaLLMRequestAttributes(span, newContext?, messagesForAPI?)`: 记录系统提示（每个哈希一次）、工具模式、带基于哈希去重的增量 new_context
- `addBetaLLMResponseAttributes(endAttributes, metadata?)`: 添加 model_output（所有用户）和 thinking_output（仅 ant）
- `addBetaToolInputAttributes(span, toolName, toolInput)`: 将工具输入添加到跨度
- `addBetaToolResultAttributes(endAttributes, toolName, toolResult)`: 将工具结果添加到跨度

#### 设计
- 基于哈希的去重：系统提示和工具模式按唯一 SHA-256 哈希记录一次
- 增量上下文：跟踪每个 querySource（agent）最后报告的消息哈希，以仅发送增量
- 系统提醒与 regular content 在 new_context 中分开
- 可见性规则：thinking 输出仅限 ant；其他所有用户都可见

### Perfetto 追踪（`perfettoTracing.ts`）

#### 函数
- `initializePerfettoTracing()`: 从 `CLAUDE_CODE_PERFETTO_TRACE` 环境变量初始化
- `isPerfettoTracingEnabled()`: 检查是否启用
- `registerAgent(agentId, agentName, parentAgentId?)`: 在追踪中注册 agent
- `unregisterAgent(agentId)`: 从追踪中移除 agent
- `startLLMRequestPerfettoSpan(args)`、`endLLMRequestPerfettoSpan(spanId, metadata)`: 带 TTFT、TTLT、缓存统计、重试子跨度的 API 调用跨度
- `startToolPerfettoSpan(toolName, args?)`、`endToolPerfettoSpan(spanId, metadata?)`: 工具执行跨度
- `startUserInputPerfettoSpan(context?)`、`endUserInputPerfettoSpan(spanId, metadata?)`: 用户输入等待跨度
- `startInteractionPerfettoSpan(userPrompt?)`、`endInteractionPerfettoSpan(spanId)`: 交互跨度
- `emitPerfettoInstant(name, category, args?)`: 即时标记事件
- `emitPerfettoCounter(name, values)`: 计数器事件
- `getPerfettoEvents()`、`resetPerfettoTracer()`: 测试辅助函数

#### 设计
- Chrome Trace Event 格式兼容 ui.perfetto.dev
- 带父子关系的 agent 层次结构可视化
- 派生指标：ITPS（输入令牌/秒）、OTPS（输出令牌/秒）、缓存命中率
- 请求设置、第一个令牌（TTFT）、采样和重试尝试的子跨度
- 100,000 的事件上限，带最旧一半驱逐
- 通过 `CLAUDE_CODE_PERFETTO_WRITE_INTERVAL_S` 的定期写入支持
- 仅 ant 特性（从外部构建中消除）

### BigQuery 导出器（`bigqueryExporter.ts`）

#### 类
- `BigQueryMetricsExporter`: 为 Anthropic 内部指标 API 实现 `PushMetricExporter`
  - 将 OTel ResourceMetrics 转换为内部负载格式
  - 发送到 `api.anthropic.com/api/claude_code/metrics`（或 ant 端点）
  - 包括客户类型、订阅类型和资源属性
  - 使用增量聚合临时性
  - 遵循信任对话框和组织选择退出设置

### 插件遥测（`pluginTelemetry.ts`）

#### 类型
- `TelemetryPluginScope`: `'official' | 'org' | 'user-local' | 'default-bundle'`
- `EnabledVia`: `'user-install' | 'org-policy' | 'default-enable' | 'seed-mount'`
- `InvocationTrigger`: `'user-slash' | 'claude-proactive' | 'nested-skill'`
- `SkillExecutionContext`: `'fork' | 'inline' | 'remote'`
- `InstallSource`: `'cli-explicit' | 'ui-discover' | 'ui-suggestion' | 'deep-link'`
- `PluginCommandErrorCategory`: `'network' | 'not-found' | 'permission' | 'validation' | 'unknown'`

#### 函数
- `hashPluginId(name, marketplace?)`: 不透明的每插件聚合键（SHA-256，16 字符）
- `getTelemetryPluginScope(name, marketplace, managedNames)`: 确定插件来源范围
- `getEnabledVia(plugin, managedNames, seedDirs)`: 确定插件如何启用
- `buildPluginTelemetryFields(name, marketplace, managedNames?)`: 构建隐私保护的遥测字段（哈希、范围、重写的名称/市场）
- `buildPluginCommandTelemetryFields(pluginInfo, managedNames?)`: 为命令/skill 调用构建字段
- `logPluginsEnabledForSession(plugins, managedNames, seedDirs)`: 为每个插件发出 `tengu_plugin_enabled_for_session`
- `classifyPluginCommandError(error)`: 将错误消息映射到有限类别
- `logPluginLoadErrors(errors, managedNames)`: 为每个错误发出 `tengu_plugin_load_failed`

#### 设计
- 双列隐私：原始值进入 PII 标记的 `_PROTO_*` 列；重写值仅对官方/内置插件显示真实名称，否则为 'third-party'
- `plugin_id_hash` 提供不暴露名称的每插件聚合

### Skill 加载事件（`skillLoadedEvent.ts`）

#### 函数
- `logSkillsLoaded(cwd, contextWindowTokens)`: 为会话启动时加载的每个 skill 发出 `tengu_skill_loaded` 事件，包括来源、loadedFrom、budget 和 kind

### 日志记录器（`logger.ts`）

#### 类
- `ClaudeCodeDiagLogger`: 实现 OTel `DiagLogger`，将错误和警告路由到应用程序的错误/调试日志系统。Info/debug/verbose 是空操作。

## 依赖

- `@opentelemetry/api`、`@opentelemetry/api-logs` — OTel 核心 API
- `@opentelemetry/sdk-metrics`、`@opentelemetry/sdk-logs`、`@opentelemetry/sdk-trace-base` — OTel SDK
- `@opentelemetry/resources`、`@opentelemetry/semantic-conventions` — 资源检测和约定
- OTLP 导出器（grpc、http、proto）和 Prometheus 导出器的动态导入
- `axios` — BigQuery 导出器的 HTTP 客户端
- `https-proxy-agent` — OTLP 导出器的代理支持

## 设计注意事项

- **多路径初始化**：Beta 追踪（`BETA_TRACING_ENDPOINT`）遵循与标准 OTEL 遥测不同的代码路径，使用自己的端点并启用详细内容日志记录
- **延迟导出器导入**：OTLP 协议导出器被动态导入，以避免在每次启动时加载 ~1.2MB 的未使用包
- **控制台导出器剥离**：在 stream-json 模式下，控制台导出器自动从配置中移除以防止破坏 SDK 消息通道
- **优雅关闭**：所有提供者在可配置超时（`CLAUDE_CODE_OTEL_SHUTDOWN_TIMEOUT_MS`，默认 2000ms）内刷新和关闭
- **安全**：插件遥测使用基于哈希的聚合和重写名称，以在仍启用分析的同时保护用户定义的插件身份
