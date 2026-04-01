# 分析服务

## 目的

分析服务为 Claude Code CLI 提供事件追踪遥测收集、功能开关管理（GrowthBook）以及多后端事件路由。它支持两个主要分析后端：Datadog（第三方）和 Anthropic 的第一方事件日志 API，并提供统一的公共 API，在接收器附加之前对事件进行排队。

## 位置

- `restored-src/src/services/analytics/index.ts` — 公共 API、事件队列、接收器接口
- `restored-src/src/services/analytics/sink.ts` — 接收器初始化和事件路由
- `restored-src/src/services/analytics/config.ts` — 共享的分析禁用逻辑
- `restored-src/src/services/analytics/growthbook.ts` — GrowthBook 功能开关集成
- `restored-src/src/services/analytics/sinkKillswitch.ts` — 每个接收器的 killswitch 配置
- `restored-src/src/services/analytics/metadata.ts` — 事件元数据丰富和清理
- `restored-src/src/services/analytics/datadog.ts` — Datadog 日志导出器
- `restored-src/src/services/analytics/firstPartyEventLogger.ts` — 通过 OpenTelemetry 的 1P 事件日志
- `restored-src/src/services/analytics/firstPartyEventLoggingExporter.ts` — 带重试的 1P 事件 HTTP 导出器

## 主要导出

### 公共 API (`index.ts`)

#### 函数
- `logEvent(eventName, metadata)`：同步事件日志记录；如果没有附加接收器则排队事件
- `logEventAsync(eventName, metadata)`：异步事件日志记录
- `attachAnalyticsSink(sink)`：附加分析后端接收器；通过 `queueMicrotask` 排出排队的事件
- `stripProtoFields(metadata)`：从发送到通用存储的有效载荷中剥离 `_PROTO_*` 键

#### 类型
- `AnalyticsSink`：具有 `logEvent` 和 `logEventAsync` 方法的接口
- `AnalyticsMetadata_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS`：标记类型（`never`），强制明确验证字符串值不包含代码/文件路径
- `AnalyticsMetadata_I_VERIFIED_THIS_IS_PII_TAGGED`：路由到 PII 标记的 proto 列的值的标记类型

### 接收器 (`sink.ts`)

#### 函数
- `initializeAnalyticsSink()`：将接收器实现附加到公共 API
- `initializeAnalyticsGates()`：在启动期间读取功能门值

### GrowthBook (`growthbook.ts`)

#### 函数
- `initializeGrowthBook()`：使用远程评估创建和初始化 GrowthBook 客户端
- `getFeatureValue_CACHED_MAY_BE_STALE(feature, defaultValue)`：非阻塞缓存读取（首选方法）
- `getFeatureValue_DEPRECATED(feature, defaultValue)`：阻塞读取（已弃用）
- `checkSecurityRestrictionGate(gate)`：等待重新初始化的安全关键门检查
- `checkGate_CACHED_OR_BLOCKING(gate)`：具有快速路径缓存和慢速路径阻塞的权利门
- `checkStatsigFeatureGate_CACHED_MAY_BE_STALE(gate)`：Statsig→GrowthBook 迁移助手
- `refreshGrowthBookAfterAuthChange()`：在登录/注销后销毁并重新创建客户端
- `onGrowthBookRefresh(listener)`：订阅功能值刷新事件
- `setGrowthBookConfigOverride(feature, value)`：设置/清除单个配置覆盖（仅限 ant）
- `getAllGrowthBookFeatures()`：枚举所有已知功能和当前值
- `resetGrowthBook()`：完全客户端重置（测试/清理）

#### 类型
- `GrowthBookUserAttributes`：发送到 GrowthBook 用于定向的用户属性（id、sessionId、deviceID、platform、org、subscription 等）

### 事件采样 (`firstPartyEventLogger.ts`)

#### 函数
- `getEventSamplingConfig()`：返回每个事件的采样配置
- `shouldSampleEvent(eventName)`：确定是否应对事件进行采样；返回采样率或 null
- `logEventTo1P(eventName, metadata)`：记录第一方事件
- `logGrowthBookExperimentTo1P(data)`：记录 GrowthBook 实验分配事件
- `initialize1PEventLogging()`：设置 OpenTelemetry LoggerProvider 和导出器
- `reinitialize1PEventLoggingIfConfigChanged()`：当批次配置更改时重建管道
- `shutdown1PEventLogging()`：刷新并关闭 1P 事件日志记录器
- `is1PEventLoggingEnabled()`：检查 1P 事件日志记录是否启用

### 元数据 (`metadata.ts`)

#### 函数
- `getEventMetadata(options)`：构建核心事件元数据（模型、会话、环境上下文等）
- `to1PEventFormat(metadata, userMetadata, additionalMetadata)`：转换为 snake_case 1P API 格式
- `sanitizeToolNameForAnalytics(toolName)`：将 MCP 工具名称编辑为 'mcp_tool' 以进行 PII 保护
- `extractMcpToolDetails(toolName)`：从 `mcp__<server>__<tool>` 格式解析 MCP 服务器/工具名称
- `extractSkillName(toolName, input)`：从 Skill 工具输入中提取技能名称
- `extractToolInputForTelemetry(input)`：序列化和截断工具输入以用于 OTel 事件
- `getFileExtensionForAnalytics(filePath)`：提取和清理文件扩展名
- `getFileExtensionsFromBashCommand(command)`：从 bash 命令中提取文件扩展名
- `mcpToolDetailsForAnalytics(toolName, serverType, baseUrl)`：条件性 MCP 详细信息提取

#### 类型
- `EventMetadata`：跨所有分析系统共享的核心事件元数据
- `EnvContext`：环境上下文（平台、架构、版本、CI 状态等）
- `ProcessMetrics`：进程资源使用情况（内存、CPU、正常运行时间）
- `FirstPartyEventLoggingMetadata`：包含环境、核心、认证和附加字段的完整 1P 事件格式

### Datadog (`datadog.ts`)

#### 函数
- `trackDatadogEvent(eventName, properties)`：向 Datadog 发送事件（批量处理）
- `initializeDatadog()`：记忆化初始化
- `shutdownDatadog()`：刷新剩余日志并清除计时器

### Killswitch (`sinkKillswitch.ts`)

#### 函数
- `isSinkKilled(sink)`：检查特定分析接收器是否通过 GrowthBook 配置禁用

## 依赖

### 内部依赖
- `bootstrap/state.js` — 会话状态（sessionId、isInteractive、信任状态）
- `utils/config.js` — 全局配置读写、信任对话框状态
- `utils/auth.js` — OAuth 令牌检查、订阅类型、Claude.ai 订阅者检测
- `utils/http.js` — 认证头生成
- `utils/user.js` — 用户数据（deviceId、email、organization UUID）
- `utils/env.js` — 环境检测（平台、CI、运行时）
- `utils/envUtils.js` — 环境变量助手、配置目录路径
- `utils/debug.js` — ant 构建的调试日志
- `utils/errors.js` — 错误工具
- `utils/log.js` — 错误日志
- `utils/signal.js` — GrowthBook 刷新监听器的信号/发布订阅
- `constants/keys.js` — GrowthBook 客户端密钥
- `constants/oauth.js` — OAuth 配置常量

### 外部依赖
- `@growthbook/growthbook` — GrowthBook SDK，用于功能开关和实验
- `@opentelemetry/api-logs` — OpenTelemetry Logs API
- `@opentelemetry/sdk-logs` — OpenTelemetry SDK，用于日志批处理
- `@opentelemetry/resources` — OpenTelemetry 资源配置
- `@opentelemetry/core` — OpenTelemetry 导出结果类型
- `@opentelemetry/semantic-conventions` — 标准语义属性名称
- `axios` — Datadog 和 1P API 调用的 HTTP 客户端
- `lodash-es` — `memoize`、`isEqual`，用于缓存和比较
- `bun:bundle` — 功能开关检测（Bun 特定）

## 实现细节

### 事件追踪系统

分析系统使用**排队然后排出**模式来处理接收器初始化之前记录的事件：

1. **事件队列**：在 `attachAnalyticsSink()` 之前记录的事件存储在内存数组中（`eventQueue`）
2. **接收器附加**：当接收器附加时，排队的事件通过 `queueMicrotask()` 异步排出，以避免阻塞启动
3. **幂等附加**：`attachAnalyticsSink()` 可以安全地多次调用 — 后续调用是空操作
4. **双模式**：同步（`logEvent`）和异步（`logEventAsync`）日志记录都支持

接收器实现（`sink.ts`）将每个事件路由到：
- **Datadog**（如果 `tengu_log_datadog_events` 门启用）
- **1P 事件日志记录**（始终启用，除非被 killswitch 杀死）

### 遥测收集

#### 元数据丰富

每个事件都通过 `getEventMetadata()` 丰富了核心元数据：

- **模型信息**：当前模型名称和 beta 标志
- **会话标识**：会话 ID、用户类型、客户端类型
- **环境上下文**：平台、架构、Node.js 版本、终端、包管理器、运行时、CI 状态、版本、构建时间
- **进程指标**：内存使用（RSS、堆）、CPU 使用率（带增量跟踪）、正常运行时间
- **代理标识**：Swarm/团队代理 ID、父会话 ID、代理类型（teammate/subagent/standalone）
- **订阅层级**：用于按层级分析 DAU 的 OAuth 订阅类型
- **仓库远程哈希**：用于服务器端连接的 SHA256 哈希

#### PII 保护

多层 PII 保护：

1. **类型级守卫**：`AnalyticsMetadata_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS`（`never` 类型）需要显式转换
2. **工具名称清理**：MCP 工具名称（`mcp__<server>__<tool>`）被编辑为 `'mcp_tool'`，除非日志门通过
3. **文件扩展名限制**：超过 10 个字符的扩展名替换为 `'other'`，以防止基于哈希的文件名泄漏
4. **工具输入截断**：字符串截断为 128 个字符，对象限制为 20 个键，最大深度 2，JSON 上限为 4KB
5. **Proto 字段剥离**：`_PROTO_*` 键从 Datadog 有效载荷中剥离（通用访问后端），但为 1P（特权 BQ 列）保留

#### 采样

事件可以通过 `tengu_event_sampling_config` GrowthBook 动态配置进行采样：
- 采样率 0：事件完全丢弃
- 采样率 0-1：随机采样，采样率添加到元数据
- 采样率 ≥1 或无配置：100% 日志记录

### GrowthBook 功能开关

#### 客户端架构

GrowthBook 使用**远程评估** — 功能值在服务器端计算并发送到客户端，而不是在本地评估：

1. **客户端创建**：`getGrowthBookClient()` 创建一个带有认证头、用户属性的记忆化 GrowthBook 实例，`remoteEval: true`
2. **初始化**：`initializeGrowthBook()` 等待客户端初始化（5 秒超时），处理远程评估有效载荷，并设置定期刷新
3. **认证感知**：客户端在认证更改后（登录/注销）被销毁并重新创建，因为 `apiHostRequestHeaders` 无法在创建后更新

#### 功能值解析顺序

1. **环境变量覆盖**（`CLAUDE_INTERNAL_FC_OVERRIDES`）— 仅限 ant，用于评估工具
2. **配置覆盖**（全局配置中的 `growthBookOverrides`）— 仅限 ant，通过 `/config Gates` 选项卡设置
3. **内存远程评估有效载荷**（`remoteEvalFeatureValues` Map）— 初始化后的权威值
4. **磁盘缓存**（`~/.claude.json` 中的 `cachedGrowthBookFeatures`）— 在进程重启后保留

#### API 方法

| 方法 | 阻塞？ | 用例 |
|--------|--------|----------|
| `getFeatureValue_CACHED_MAY_BE_STALE` | 否 | 启动关键路径、同步上下文、渲染循环 |
| `getFeatureValue_DEPRECATED` | 是 | 旧代码（正在迁移） |
| `checkSecurityRestrictionGate` | 异步（等待重新初始化） | 认证更改后的安全关键门 |
| `checkGate_CACHED_OR_BLOCKING` | 异步（快速缓存、慢速阻塞） | 用户调用的功能根据订阅/组织进行门控 |
| `checkStatsigFeatureGate_CACHED_MAY_BE_STALE` | 否 | 仅用于 Statsig→GrowthBook 迁移 |

#### 定期刷新

- **外部构建**：每 6 小时（与 Statsig 的间隔匹配）
- **Ant 构建**：每 20 分钟
- 使用 `client.refreshFeatures()`（轻量刷新，保留客户端状态）
- 成功刷新后通知 `onGrowthBookRefresh` 订阅者

#### 变通方案

代码包含多个 SDK 变通方案：
- **格式错误的 API 响应**：API 返回 `{ "value": ... }` 但 SDK 期望 `{ "defaultValue": ... }` — 在 `processRemoteEvalPayload()` 中转换
- **setForcedFeatures 不可靠**：使用自定义缓存（`remoteEvalFeatureValues` Map）而不是 SDK 的强制功能
- **空有效载荷防护**：防止 `{features: {}}` 服务器 bug 清除磁盘上的所有标志

### 分析配置

#### 禁用条件（`isAnalyticsDisabled()`）

当以下任何条件为真时，分析被禁用：
- `NODE_ENV === 'test'`
- `CLAUDE_CODE_USE_BEDROCK` 为真
- `CLAUDE_CODE_USE_VERTEX` 为真
- `CLAUDE_CODE_USE_FOUNDRY` 为真
- 隐私级别禁用遥测（`isTelemetryDisabled()`）

#### 反馈调查

`isFeedbackSurveyDisabled()` 范围更窄 — 它不会阻止 3P 提供商（Bedrock/Vertex/Foundry），因为调查是本地 UI 提示，没有脚本数据。

### 事件接收器

#### Datadog 接收器

- **端点**：`https://http-intake.logs.us5.datadoghq.com/api/v2/logs`
- **批处理**：每个批次最多 100 个事件，15 秒刷新间隔
- **允许的事件**：约 50 个事件名称的白名单（API 错误、工具使用、OAuth 事件等）
- **基数减少**：模型名称规范化为规范名称，MCP 工具名称规范化为 'mcp'，开发版本截断
- **用户桶**：用户 ID 的 SHA256 哈希到 30 个桶，用于唯一用户估计而不暴露 ID
- **标签**：高基数字段（模型、平台、版本、userType 等）作为 Datadog 标签发送以进行过滤

#### 第一方事件日志接收器

- **端点**：`api.anthropic.com`（或暂存）上的 `/api/event_logging/batch`
- **协议**：OpenTelemetry Logs API，带自定义 `FirstPartyEventLoggingExporter`
- **批处理**：通过 GrowthBook 动态配置（`tengu_1p_event_batch_config`）可配置
  - 默认值：每个批次 200 个事件，10 秒导出间隔，8192 最大队列大小
- **弹性**：
  - 失败事件的仅追加 JSONL 文件存储（`~/.claude/telemetry/1p_failed_events.*`）
  - 二次退避重试（base * attempts²，最大 30 秒，最多 8 次重试）
  - 启动时重试上一会话的失败批次
  - 大事件集的分块
  - 认证回退：401 错误时重试不带认证
- **事件类型**：`ClaudeCodeInternalEvent`（标准事件）和 `GrowthbookExperimentEvent`（实验暴露）
- **Proto 序列化**：使用 protobuf 生成的 `ClaudeCodeInternalEvent.toJSON()` 进行有线格式

#### 接收器 Killswitch

通过 GrowthBook 配置 `tengu_frond_boric` 进行每个接收器的 killswitch：
```json
{ "datadog": true, "firstParty": false }
```
- 在每个事件调度时检查（不是在初始化时），以允许运行时切换
- 也在 1P 导出器的 `sendBatchWithRetry` 内部检查，以在杀死时停止重试

### 隐私注意事项

1. **选择退出支持**：通过隐私级别设置尊重全局遥测选择退出
2. **第三方提供商隔离**：使用 Bedrock、Vertex 或 Foundry 时不发送分析
3. **PII 标记字段**：`_PROTO_*` 键路由到具有访问控制的特权 BigQuery 列；在 Datadog 分发之前被剥离
4. **MCP 工具名称编辑**：用户配置的 MCP 服务器名称是 PII 中等风险，默认情况下被编辑
5. **文件路径保护**：类型系统防止意外记录文件路径；文件扩展名被清理
6. **工具输入截断**：深度嵌套和长字符串被截断以限制输出大小
7. **用户分桶**：Datadog 使用哈希用户桶（30 个桶）而不是原始用户 ID
8. **模型名称规范化**：外部用户模型名称规范化以减少基数并防止泄漏自定义模型标识符
9. **信任门控认证头**：在工作区信任建立之前，认证头不会发送到 GrowthBook 或 1P 端点

## 数据流

```
调用站点 (logEvent/logEventAsync)
  │
  ├─ 没有接收器附加？→ 排队事件
  │
  ├─ 接收器附加 → sink.logEvent()
  │    │
  │    ├─ shouldSampleEvent() → 如果采样则丢弃
  │    │
  │    ├─ isSinkKilled('datadog')？→ 跳过 Datadog
  │    │    └─ trackDatadogEvent() → 批处理 → Datadog HTTP API
  │    │
  │    └─ logEventTo1P()
  │         │
  │         ├─ getEventMetadata() → 用环境、会话、进程信息丰富
  │         ├─ OpenTelemetry LoggerProvider → BatchLogRecordProcessor
  │         └─ FirstPartyEventLoggingExporter.export()
  │              │
  │              ├─ 将日志转换为 proto 格式 (ClaudeCodeInternalEvent.toJSON)
  │              ├─ sendBatchWithRetry() → POST /api/event_logging/batch
  │              │    ├─ 认证头（带 401 回退）
  │              │    └─ 分块为 maxBatchSize 批次
  │              ├─ 成功 → 重置退避，重试排队失败
  │              └─ 失败 → 附加到磁盘 JSONL，安排退避重试
```

## 配置

### 环境变量

| 变量 | 用途 |
|----------|----------|
| `CLAUDE_INTERNAL_FC_OVERRIDES` | 覆盖 GrowthBook 功能的 JSON 对象（仅限 ant，USER_TYPE='ant'） |
| `CLAUDE_CODE_GB_BASE_URL` | 自定义 GrowthBook API 基础 URL（仅限 ant） |
| `ANTHROPIC_BASE_URL` | API 基础 URL；主机用作 GrowthBook 属性以进行企业代理定位 |
| `OTEL_LOGS_EXPORT_INTERVAL` | 1P 日志导出间隔覆盖（毫秒） |
| `OTEL_LOG_TOOL_DETAILS` | 为 OTel 事件启用详细工具名称日志记录 |
| `CLAUDE_CODE_DATADOG_FLUSH_INTERVAL_MS` | Datadog 刷新间隔覆盖（毫秒） |
| `NODE_ENV` | 'test' 禁用所有分析 |
| `CLAUDE_CODE_USE_BEDROCK/VERTEX/FOUNDRY` | 为第三方提供商禁用分析 |

### GrowthBook 动态配置

| 配置名称 | 用途 |
|-------------|-------------|
| `tengu_event_sampling_config` | 每个事件的采样率 |
| `tengu_1p_event_batch_config` | 1P 导出器批次大小、延迟、端点、认证设置 |
| `tengu_frond_boric` | 每个接收器的 killswitch（datadog/firstParty） |
| `tengu_log_datadog_events` | 启用 Datadog 事件调度的门 |

## 错误处理

- **GrowthBook 初始化失败**：捕获并记录；使用磁盘缓存作为回退
- **1P 导出失败**：事件写入磁盘 JSONL，以二次退避重试，最多尝试次数后丢弃
- **Datadog 失败**：静默记录；无重试（即发即忘批处理）
- **认证失败**：1P 导出器在 401 时不带认证重试；过期的 OAuth 令牌主动跳过认证
- **元数据丰富失败**：吞下并记录错误；事件可能以部分元数据发出

## 相关模块

- [OAuth 服务](./oauth-service.md) — 认证和令牌管理
- [Auth 工具](../05-utils/auth.md) — API 密钥和 OAuth 令牌来源
- [安全存储](../05-utils/secure-storage.md) — 凭证存储后端
- [配置工具](../05-utils/config.md) — 全局配置管理
