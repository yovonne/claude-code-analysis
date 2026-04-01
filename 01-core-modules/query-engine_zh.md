# 查询引擎

## 目的

查询引擎是管理 Claude Code 中 LLM 交互完整生命周期的核心子系统。它协调消息构建、API 流式处理、工具执行循环、上下文管理（压缩、裁剪、折叠）、重试逻辑、成本/令牌跟踪和结果报告。它作为用户输入与 Anthropic API 之间的桥梁，处理带有工具使用的多轮 agentic 对话。

## 位置

- `restored-src/src/QueryEngine.ts`（约 1296 行）— 主查询引擎类和 `ask()` 便捷包装器
- `restored-src/src/query.ts`（约 1700+ 行）— 核心查询循环（`query()`、`queryLoop()`）
- `restored-src/src/query/` — 查询子模块（令牌预算、配置、依赖、停止钩子）
- `restored-src/src/services/api/claude.ts` — 流式 API 客户端（`queryModelWithStreaming`）
- `restored-src/src/services/api/withRetry.ts` — API 调用重试逻辑
- `restored-src/src/services/api/errors.ts` — API 错误分类和处理
- `restored-src/src/services/api/logging.ts` — API 遥测和日志记录
- `restored-src/src/utils/messages.ts` — 消息构建工具
- `restored-src/src/utils/systemPromptType.ts` — `SystemPrompt` 品牌类型
- `restored-src/src/context.ts` — 用户和系统上下文构建器
- `restored-src/src/cost-tracker.ts` — 成本和令牌计费

## 关键导出

### 类

- **`QueryEngine`** — 拥有对话的查询生命周期和会话状态。每个对话一个 `QueryEngine`。每次 `submitMessage()` 调用开始同一对话内的新轮次。状态（消息、文件缓存、使用量等）在轮次之间保持。

### 函数

- **`ask()`** — 用于一次性使用的 `QueryEngine` 便捷包装器。创建 `QueryEngine`，调用 `submitMessage()`，并传回读取的文件缓存。
- **`query()`** — 运行查询循环的核心生成器函数。生成 `StreamEvent`、`RequestStartEvent`、`Message`、`TombstoneMessage` 和 `ToolUseSummaryMessage`。返回 `Terminal` 结果。
- **`queryLoop()`** — 查询循环的内部实现。处理压缩、API 调用、工具执行和继续逻辑。
- **`handleStopHooks()`** — 在模型完成响应后执行后响应钩子（Stop、TaskCompleted、TeammateIdle）。
- **`buildQueryConfig()`** — 在查询入口处一次性快照不可变的运行时门控（流式工具执行、工具使用摘要、ant 模式、快速模式）。

### 类型

- **`QueryEngineConfig`** — `QueryEngine` 构造函数的配置对象。包括 `cwd`、`tools`、`commands`、`mcpClients`、`agents`、`canUseTool`、`getAppState`、`setAppState`、`initialMessages`、`readFileCache`、`customSystemPrompt`、`appendSystemPrompt`、`userSpecifiedModel`、`fallbackModel`、`thinkingConfig`、`maxTurns`、`maxBudgetUsd`、`taskBudget`、`jsonSchema` 等。
- **`QueryParams`** — `query()` 函数的参数。包括 `messages`、`systemPrompt`、`userContext`、`systemContext`、`canUseTool`、`toolUseContext`、`fallbackModel`、`querySource`、`maxOutputTokensOverride`、`maxTurns`、`skipCacheWrite`、`taskBudget`、`deps`。
- **`QueryDeps`** — 用于测试的依赖注入接口。包含 `callModel`、`microcompact`、`autocompact` 和 `uuid`。
- **`QueryConfig`** — 不可变的运行时配置快照。包含 `sessionId` 和 `gates`（streamingToolExecution、emitToolUseSummaries、isAnt、fastModeEnabled）。
- **`State`** — 查询循环中的可变跨迭代状态。包含 `messages`、`toolUseContext`、`autoCompactTracking`、`maxOutputTokensRecoveryCount`、`hasAttemptedReactiveCompact`、`maxOutputTokensOverride`、`pendingToolUseSummary`、`stopHookActive`、`turnCount`、`transition`。
- **`TokenBudgetDecision`** — 令牌预算检查的结果。`{ action: 'continue', nudgeMessage, ... }` 或 `{ action: 'stop', completionEvent }`。
- **`BudgetTracker`** — 跟踪令牌预算继续，包含 `continuationCount`、`lastDeltaTokens`、`lastGlobalTurnTokens`、`startedAt`。
- **`SystemPrompt`** — 品牌类型（`readonly string[] & { readonly __brand: 'SystemPrompt' }`），用于类型安全的系统提示数组。

## 查询构建

查询通过多阶段管道构建：

### 1. 系统提示组装

```
QueryEngine.submitMessage()
  → fetchSystemPromptParts()           // 获取工具、模型、MCP、自定义提示部分
  → asSystemPrompt([...parts])         // 包装为品牌 SystemPrompt 类型
  → appendSystemContext()              // 添加系统上下文（git 状态、缓存破坏器）
```

系统提示由以下部分构建：
- **静态部分**（可使用 `scope: 'global'` 缓存）：介绍、系统规则、任务执行、操作、工具、语气/风格、输出效率
- **动态边界标记**：`SYSTEM_PROMPT_DYNAMIC_BOUNDARY`（`__SYSTEM_PROMPT_DYNAMIC_BOUNDARY__`）分隔可缓存和动态内容
- **动态部分**（运行时计算）：会话指导、内存、ant 模型覆盖、环境信息、语言、输出风格、MCP 说明、草稿纸、功能结果清除、令牌预算、简报

当设置 `CLAUDE_CODE_SIMPLE` 时，使用最小提示：
```typescript
[`You are Claude Code...`, `CWD: ${cwd}`, `Date: ${date}`]
```

### 2. 消息准备

```
queryLoop()
  → getMessagesAfterCompactBoundary()  // 仅消息压缩边界后的消息
  → applyToolResultBudget()            // 强制执行每条消息的工具结果大小限制
  → snipCompactIfNeeded()              // 功能门控的历史裁剪
  → microcompactMessages()             // 功能门控的缓存感知微压缩
  → applyCollapsesIfNeeded()           // 功能门控的上下文折叠
  → prependUserContext()               // 添加用户上下文（claudeMd、currentDate）
```

### 3. API 请求参数

对 `deps.callModel()`（包装 `queryModelWithStreaming`）的最终请求包括：
- `messages`：准备好的消息数组，预先添加用户上下文
- `systemPrompt`：带有附加系统上下文的完整系统提示
- `thinkingConfig`：自适应/启用/禁用的思考配置
- `tools`：转换为 API 模式的可用工具定义
- `signal`：用于取消的 AbortController 信号
- `options`：模型、快速模式、工具选择、回退模型、查询源、agent、任务预算等。

## 消息构建

消息使用 `utils/messages.ts` 中的工厂函数构建：

### 消息类型

| 类型 | 工厂函数 | 用途 |
|------|-----------------|---------|
| `UserMessage` | `createUserMessage()` | 用户输入、工具结果、元消息 |
| `AssistantMessage` | N/A（来自 API） | 带文本、思考、tool_use 块的模型响应 |
| `SystemMessage` | `createSystemMessage()` | 内部通知、警告、API 重试 |
| `AttachmentMessage` | `createAttachmentMessage()` | 结构化数据附件（max_turns、hook 结果） |
| `ProgressMessage` | N/A | 工具执行进度更新 |
| `StreamEvent` | N/A（来自 API） | 原始 SSE 事件（message_start、content_block_start 等） |
| `TombstoneMessage` | `{ type: 'tombstone' }` | 信号从 UI 中删除孤立消息 |
| `ToolUseSummaryMessage` | `createToolUseSummaryMessage()` | 工具使用批次的摘要 |
| `CompactBoundaryMessage` | `createMicrocompactBoundaryMessage()` | 标记压缩边界 |

### 思考块规则

代码记录了思考块的严格规则（query.ts:151-163）：
1. 带 thinking/redacted_thinking 块的消息需要 `max_thinking_length > 0`
2. 思考块不能是序列中的最后一条消息
3. 思考块必须在 assistant 轨迹期间保留

### 消息规范化

- `normalizeMessagesForAPI()` — 将内部消息转换为 API 兼容格式
- `normalizeMessage()` — 规范化用于 SDK 产出的消息
- `ensureToolResultPairing()` — 剥离重复的 tool_use ID 以防止 API 错误
- `stripSignatureBlocks()` — 在回退前移除特定模型的思考签名

## LLM API 交互

### 流式客户端

主要 API 交互通过 `services/api/claude.ts` 中的 `queryModelWithStreaming()`：

```typescript
// QueryEngine.ts → queryLoop() → deps.callModel()
for await (const message of deps.callModel({
  messages: prependUserContext(messagesForQuery, userContext),
  systemPrompt: fullSystemPrompt,
  thinkingConfig: toolUseContext.options.thinkingConfig,
  tools: toolUseContext.options.tools,
  signal: toolUseContext.abortController.signal,
  options: { model, fastMode, fallbackModel, querySource, ... }
})) {
  // 处理流式消息
}
```

### 请求构建

API 请求包括：
- **Betas**：上下文管理、提示缓存、结构化输出、任务预算、努力、快速模式、advisor、工具搜索
- **工具定义**：通过 `toolToAPISchema()` 转换，带 JSON schemas
- **缓存断点**：系统提示在 `SYSTEM_PROMPT_DYNAMIC_BOUNDARY` 处分割，用于全局与会话范围的缓存
- **模型参数**：`max_tokens`、`temperature`、`thinking` 配置、`effort`

### 响应处理

流式事件按顺序处理：
1. `message_start` — 为新消息重置使用量跟踪
2. `content_block_start` — 新内容块（text、thinking、tool_use）
3. `content_block_delta` — 当前块的流式增量
4. `content_block_stop` — 块完成；tool_use 块排队等待执行
5. `message_delta` — 最终 stop_reason 和使用量增量
6. `message_stop` — 将使用量累积到总数

## 流式响应

流式管道处理几种场景：

### 正常流式处理

```
for await (const message of deps.callModel(...)) {
  // 从 SDK 调用者处保留可恢复的错误
  if (!withheld) yield yieldMessage
  if (message.type === 'assistant') {
    assistantMessages.push(message)
    // 将 tool_use 块排队到流式执行器
    streamingToolExecutor?.addTool(toolBlock, message)
    // 在完成时发出完成工具结果
    for (const result of streamingToolExecutor.getCompletedResults()) {
      yield result.message
    }
  }
}
```

### 流式回退

当流式处理中途失败时：
1. 设置 `streamingFallbackOccured` 标志
2. 将孤立的 assistant 消息 tombstone（从 UI 中移除）
3. 丢弃待处理的工具结果
4. 创建新的 `StreamingToolExecutor`
5. 重试请求（非流式回退）

### 保留可恢复错误

某些错误在尝试恢复之前从 SDK 调用者处保留：
- **提示太长**（413/400）：保留以便上下文折叠或响应式压缩可以重试
- **最大输出令牌**：保留以便升级/恢复循环可以继续
- **媒体大小错误**：保留以便响应式压缩可以剥离图像并重试

```typescript
let withheld = false
if (contextCollapse?.isWithheldPromptTooLong(message)) withheld = true
if (reactiveCompact?.isWithheldPromptTooLong(message)) withheld = true
if (reactiveCompact?.isWithheldMediaSizeError(message)) withheld = true
if (isWithheldMaxOutputTokens(message)) withheld = true
if (!withheld) yield yieldMessage
```

## 上下文管理

查询引擎采用多种上下文管理策略，按顺序应用：

### 1. 工具结果预算

`applyToolResultBudget()` 强制执行每条消息的聚合工具结果大小限制。用截断内容替换过大的工具结果。

### 2. 历史裁剪（功能：`HISTORY_SNIP`）

`snipCompactIfNeeded()` 在保留"受保护尾部"的同时移除较旧的消息历史。主要用于 headless/SDK 模式以在长会话中限制内存。

### 3. 微压缩（功能：`CACHED_MICROCOMPACT`）

`microcompactMessages()` 执行缓存感知的微压缩，编辑提示缓存而非总结内容。推迟边界消息发出直到 API 响应后，以使用实际的 `cache_deleted_input_tokens`。

### 4. 上下文折叠（功能：`CONTEXT_COLLAPSE`）

`applyCollapsesIfNeeded()` 通过将旧消息归档并用摘要替换来投影折叠的上下文视图。在自动压缩之前运行，以便如果折叠低于阈值，自动压缩是一个空操作。

### 5. 自动压缩

`autoCompactIfNeeded()` 在令牌计数接近上下文限制时触发自动总结。跟踪：
- 压缩前/后的令牌计数
- 压缩使用量（输入/输出/缓存令牌）
- 连续失败（断路器）
- 自上次压缩以来的轮次计数

压缩后的消息通过 `buildPostCompactMessages()` 构建：
```
summaryMessages + attachments + hookResults + preserved tail
```

### 6. 响应式压缩（功能：`REACTIVE_COMPACT`）

`tryReactiveCompact()` 是一种恢复机制，当 API 返回提示太长错误时触发。它剥离图像或总结内容以适应限制。

### 7. 阻塞限制检查

在发出 API 调用之前，如果尚未压缩且不在恢复路径上：
```typescript
const { isAtBlockingLimit } = calculateTokenWarningState(
  tokenCountWithEstimation(messagesForQuery) - snipTokensFreed,
  toolUseContext.options.mainLoopModel,
)
if (isAtBlockingLimit) {
  yield createAssistantAPIErrorMessage({ content: PROMPT_TOO_LONG_ERROR_MESSAGE })
  return { reason: 'blocking_limit' }
}
```

## 令牌跟踪

令牌使用在多个级别跟踪：

### 每条消息跟踪

```typescript
// 在 message_start 上重置
currentMessageUsage = EMPTY_USAGE
currentMessageUsage = updateUsage(currentMessageUsage, message.event.message.usage)

// 在 message_delta 上更新
currentMessageUsage = updateUsage(currentMessageUsage, message.event.usage)

// 在 message_stop 上累积
this.totalUsage = accumulateUsage(this.totalUsage, currentMessageUsage)
```

### 会话级别跟踪

`cost-tracker.ts` 维护累积状态：
- `totalCostUSD` — 总 API 成本（美元）
- `totalInputTokens`、`totalOutputTokens` — 令牌计数
- `totalCacheReadInputTokens`、`totalCacheCreationInputTokens` — 缓存令牌计数
- `totalAPIDuration`、`totalAPIDurationWithoutRetries` — 计时指标
- `totalLinesAdded`、`totalLinesRemoved` — 代码更改指标
- `modelUsage` — 按模型细分的映射

### 令牌预算功能（功能：`TOKEN_BUDGET`）

`query/tokenBudget.ts` 中的 `BudgetTracker` 管理令牌预算继续：

```typescript
// 决策逻辑
if (turnTokens < budget * 0.9 && !isDiminishing) {
  return { action: 'continue', nudgeMessage: '...' }
}
if (isDiminishing || continuationCount > 0) {
  return { action: 'stop', completionEvent: { ... } }
}
```

- **完成阈值**：预算的 90%
- **递减返回**：如果连续 3+ 次增量令牌 < 500 则停止
- **提示消息**：作为元用户消息注入以继续模型

### API 任务预算

`taskBudget` 参数（`total` + `remaining`）跨压缩边界跟踪 API 端任务预算：
```typescript
// 在消息替换之前捕获压缩前的最终上下文
const preCompactContext = finalContextTokensFromLastResponse(messagesForQuery)
taskBudgetRemaining = Math.max(0, (taskBudgetRemaining ?? total) - preCompactContext)
```

## 成本跟踪

成本在 `cost-tracker.ts` 中计算：

```typescript
export function addToTotalSessionCost(cost: number, usage: Usage, model: string): number {
  const modelUsage = addToTotalModelUsage(cost, usage, model)
  addToTotalCostState(cost, modelUsage, model)
  
  // OTel 指标
  getCostCounter()?.add(cost, { model, speed })
  getTokenCounter()?.add(usage.input_tokens, { model, type: 'input' })
  // ... output, cacheRead, cacheCreation
}
```

成本计算使用 `utils/modelCost.ts` 中的 `calculateUSDCost(model, usage)`，应用按模型定价。

### 成本持久化

- `saveCurrentSessionCosts()` — 在会话切换前将成本保存到项目配置
- `restoreCostStateForSession()` — 在恢复会话时恢复成本
- 成本包括 advisor 工具令牌使用量（单独模型定价）

### 结果报告

最终 `result` 消息包括：
```typescript
{
  type: 'result',
  subtype: 'success' | 'error_max_turns' | 'error_max_budget_usd' | 
           'error_max_structured_output_retries' | 'error_during_execution',
  duration_ms: number,
  duration_api_ms: number,
  num_turns: number,
  result: string,           // 从最后一条 assistant 消息提取的文本
  stop_reason: string | null,
  session_id: string,
  total_cost_usd: number,
  usage: NonNullableUsage,
  modelUsage: { [model]: ModelUsage },
  permission_denials: SDKPermissionDenial[],
  structured_output: unknown,
  fast_mode_state: FastModeState,
  errors: string[],         // 轮次作用域错误诊断
}
```

## 模型选择

### 初始模型选择

```typescript
const initialMainLoopModel = userSpecifiedModel
  ? parseUserSpecifiedModel(userSpecifiedModel)
  : getMainLoopModel()
```

### 运行时模型选择

```typescript
const currentModel = getRuntimeMainLoopModel({
  permissionMode,
  mainLoopModel: toolUseContext.options.mainLoopModel,
  exceeds200kTokens: permissionMode === 'plan' && doesMostRecentAssistantMessageExceed200k(messagesForQuery),
})
```

### 模型回退

当捕获 `FallbackTriggeredError` 时（连续 3 次 529 错误后）：
1. 切换到 `fallbackModel`
2. 清除所有 assistant 消息和工具结果
3. 为孤立消息 yield tombstones
4. 剥离思考签名块（特定于模型）
5. 创建新的 `StreamingToolExecutor`
6. Yield 关于回退的系统消息

```typescript
yield createSystemMessage(
  `Switched to ${renderModelName(fallbackModel)} due to high demand for ${renderModelName(originalModel)}`,
  'warning',
)
```

### 思考配置

```typescript
const initialThinkingConfig = thinkingConfig ?? (
  shouldEnableThinkingByDefault() !== false
    ? { type: 'adaptive' }
    : { type: 'disabled' }
)
```

## 错误处理

`services/api/errors.ts` 中的错误处理管道对 API 错误进行分类和格式化：

### 错误分类（`classifyAPIError`）

返回分析的标准错误类型字符串：
- `aborted`、`api_timeout`、`repeated_529`、`capacity_off_switch`
- `rate_limit`、`server_overload`、`prompt_too_long`
- `pdf_too_large`、`pdf_password_protected`、`image_too_large`
- `tool_use_mismatch`、`unexpected_tool_result`、`duplicate_tool_use_id`
- `invalid_model`、`credit_balance_low`、`invalid_api_key`
- `token_revoked`、`oauth_org_not_allowed`、`auth_error`
- `bedrock_model_access`、`ssl_cert_error`、`connection_error`
- `server_error`、`client_error`、`unknown`

### 错误转消息转换（`getAssistantMessageFromError`）

将特定 API 错误映射到用户友好的 assistant 消息：
- **超时**：`API_TIMEOUT_ERROR_MESSAGE`
- **速率限制（429）**：解析统一速率限制头、超额状态、重置时间戳
- **提示太长（413/400）**：通用消息 + 原始错误在 `errorDetails` 中用于响应式压缩
- **媒体大小错误**：每种变体的消息（PDF、图像、多图像）
- **认证错误（401/403）**：登录提示、组织禁用、令牌撤销
- **模型错误（404）**：带回退建议的模型不可用指导
- **工具使用错误（400）**：不匹配、意外工具结果、重复 ID
- **拒绝**：使用策略违规消息

## 重试逻辑

`services/api/withRetry.ts` 中的重试系统是全面的：

### `withRetry()` 生成器

```typescript
export async function* withRetry<T>(
  getClient: () => Promise<Anthropic>,
  operation: (client, attempt, context) => Promise<T>,
  options: RetryOptions,
): AsyncGenerator<SystemAPIErrorMessage, T>
```

### 重试配置

| 参数 | 默认值 | 描述 |
|-----------|---------|---------|
| `DEFAULT_MAX_RETRIES` | 10 | 最大重试次数 |
| `MAX_529_RETRIES` | 3 | 回退前的连续 529 错误 |
| `BASE_DELAY_MS` | 500 | 指数退避的基础延迟 |
| `FLOOR_OUTPUT_TOKENS` | 3000 | 上下文溢出时的最小输出令牌 |

### 可重试错误（`shouldRetry`）

- **始终重试**：模拟错误（永不）、连接错误、408（超时）、409（锁）、500+（服务器）、401（认证 — 带缓存清除）
- **条件重试**：429（速率限制 — 企业除外 claude.ai 订阅者）、过载错误（529）
- **永不重试**：`x-should-retry: false` 头（除 ant 5xx 外）、模拟速率限制

### 指数退避

```typescript
export function getRetryDelay(attempt: number, retryAfterHeader?: string | null, maxDelayMs = 32000): number {
  if (retryAfterHeader) return parseInt(retryAfterHeader, 10) * 1000
  const baseDelay = Math.min(BASE_DELAY_MS * Math.pow(2, attempt - 1), maxDelayMs)
  const jitter = Math.random() * 0.25 * baseDelay
  return baseDelay + jitter
}
```

### 持久重试（功能：`UNATTENDED_RETRY`）

当启用 `CLAUDE_CODE_UNATTENDED_RETRY` 时：
- 无限重试 429/529
- 最大退避：5 分钟
- 重置上限：6 小时
- 心跳：每 30 秒 yield `api_retry` 系统消息以防止主机空闲检测

### 快速模式回退

在快速模式激活时出现 429/529：
- **短 retry-after**（< 20s）：等待并重试，保持快速模式激活（保留提示缓存）
- **长/未知 retry-after**：进入冷却（至少 10 分钟），切换到标准速度

### 529 回退触发器

连续 3 次 529 错误后，抛出 `FallbackTriggeredError` 以触发模型回退（Opus → Sonnet）。

### 前台与后台重试

只有前台查询源在 529 上重试：
```typescript
const FOREGROUND_529_RETRY_SOURCES = new Set([
  'repl_main_thread', 'sdk', 'agent:custom', 'agent:default', 
  'agent:builtin', 'compact', 'hook_agent', 'hook_prompt',
  'verification_agent', 'side_question', 'auto_mode', 'bash_classifier'
])
```

## 关键算法

### 最大输出令牌恢复

当模型达到其输出令牌限制时：
1. **升级**（功能：`tengu_otk_slot_v1`）：如果使用默认 8k 上限，在 64k 重试
2. **多轮恢复**：注入元消息请求模型恢复，最多 `MAX_OUTPUT_TOKENS_RECOVERY_LIMIT`（3）次
3. **耗尽**：显示保留的错误

```typescript
const recoveryMessage = createUserMessage({
  content: `Output token limit hit. Resume directly — no apology, no recap...`,
  isMeta: true,
})
```

### 提示太长恢复

当 API 返回提示太长错误时：
1. **折叠排水**：首先尝试排空分阶段的上下文折叠（保持细粒度上下文）
2. **响应式压缩**：剥离图像或总结以适应
3. **无恢复**：显示错误，运行停止失败钩子

### 流式工具执行

两种执行模式：
1. **流式**（`tengu_streaming_tool_execution2` 门控）：工具在模型继续流式处理时并发执行。使用 `StreamingToolExecutor` 管理进行中的工具。
2. **批处理**：流式处理完成后，`runTools()` 顺序执行所有 tool_use 块。

### 结构化输出重试

当使用 JSON schema 输出时：
- 跟踪每次查询的 `SYNTHETIC_OUTPUT_TOOL_NAME` 调用
- 最大重试次数：`MAX_STRUCTURED_OUTPUT_RETRIES`（默认 5）
- 超出：yield `error_max_structured_output_retries` 结果

### 工具使用摘要生成

工具批次完成后（功能门控）：
1. 提取最后一条 assistant 文本块作为上下文
2. 收集每个工具的工具名称、输入和输出
3. 异步触发基于 Haiku 的摘要生成
4. 摘要在下一次迭代开始时 yield

## 依赖

### 内部依赖

| 模块 | 用途 |
|--------|---------|
| `services/api/claude.ts` | `queryModelWithStreaming()` — API 流式客户端 |
| `services/api/withRetry.ts` | `withRetry()` — 带指数退避的重试包装器 |
| `services/api/errors.ts` | 错误分类和消息生成 |
| `services/api/logging.ts` | API 遥测（查询、成功、错误事件） |
| `services/compact/autoCompact.ts` | `autoCompactIfNeeded()` — 自动上下文压缩 |
| `services/compact/microCompact.ts` | `microcompactMessages()` — 缓存感知微压缩 |
| `services/compact/snipCompact.ts` | 功能门控的历史裁剪 |
| `services/compact/reactiveCompact.ts` | 功能门控的响应式压缩恢复 |
| `services/contextCollapse/` | 功能门控的上下文折叠 |
| `services/tools/toolOrchestration.ts` | `runTools()` — 批处理工具执行 |
| `services/tools/StreamingToolExecutor.ts` | 流式处理期间的并发工具执行 |
| `services/toolUseSummary/` | 工具使用摘要生成 |
| `utils/messages.ts` | 消息工厂函数和规范化 |
| `utils/systemPromptType.ts` | `SystemPrompt` 品牌类型 |
| `utils/attachments.ts` | 附件消息创建和预取 |
| `utils/model/model.ts` | 模型选择和解析 |
| `utils/tokens.ts` | 令牌计数和估计 |
| `utils/queryContext.ts` | `fetchSystemPromptParts()` |
| `utils/processUserInput/` | 用户输入和斜杠命令处理 |
| `utils/sessionStorage.ts` | Transcript 录制和刷新 |
| `bootstrap/state.ts` | 全局状态管理（成本、令牌、会话） |
| `cost-tracker.ts` | 成本计算和跟踪 |
| `context.ts` | 用户和系统上下文构建器 |
| `constants/prompts.ts` | 系统提示部分构建器 |
| `constants/messages.ts` | 消息常量 |
| `constants/betas.ts` | API beta 头常量 |
| `constants/apiLimits.ts` | API 限制常量 |

### 外部依赖

| 包 | 用途 |
|---------|---------|
| `@anthropic-ai/sdk` | Anthropic API 客户端、类型、错误类 |
| `lodash-es/last.js` | 数组最后元素工具 |
| `strip-ansi` | SDK 输出的 ANSI 代码剥离 |
| `crypto.randomUUID` | UUID 生成 |
| `bun:bundle` | 功能标志系统（`feature()`） |

## 数据流

```
用户输入
    │
    ▼
┌─────────────────────────────────────────┐
│           QueryEngine.submitMessage()    │
│                                         │
│  1. 获取系统提示部分                      │
│  2. 处理用户输入（斜杠命令）               │
│  3. 将消息推送到可变消息                   │
│  4. 录制 transcript                      │
│  5. Yield 系统初始化消息                  │
│  6. 进入查询循环                          │
└──────────────────┬──────────────────────┘
                   │
                   ▼
┌─────────────────────────────────────────┐
│              query() / queryLoop()       │
│                                         │
│  ┌─ 迭代 ──────────────────────────────┐ │
│  │                                      │ │
│  │  1. 应用裁剪（功能门控）              │ │
│  │  2. 应用微压缩                       │ │
│  │  3. 应用上下文折叠                   │ │
│  │  4. 需要时应用自动压缩               │ │
│  │  5. 检查阻塞限制                     │ │
│  │  6. 调用模型（流式）                 │ │
│  │     - 保留可恢复错误                 │ │
│  │     - 排队 tool_use 块               │ │
│  │     - 流式处理工具结果               │ │
│  │  7. 执行工具                        │ │
│  │  8. 获取附件消息                    │ │
│  │  9. 运行停止钩子                    │ │
│  │  10. 检查令牌预算                   │ │
│  │  11. 继续或返回                     │ │
│  └──────────────────────────────────┘  │
│                                         │
│  恢复路径：                              │
│  - 提示太长 → 折叠 → 响应式               │
│  - 最大输出令牌 → 升级 → 重试             │
│  - 529 错误 → 回退模型                   │
│  - 中止 → 合成 tool_results              │
└──────────────────┬──────────────────────┘
                   │
                   ▼
┌─────────────────────────────────────────┐
│           QueryEngine result yield       │
│                                         │
│  - success / error_* 子类型              │
│  - duration、cost、usage、turns         │
│  - permission denials、fast mode state  │
│  - structured output（如果适用）         │
└─────────────────────────────────────────┘
```

## 集成点

### 工具系统

- **工具权限**：`canUseTool` 回调包装权限检查并跟踪拒绝
- **工具执行**：用于并发执行的 `StreamingToolExecutor`；用于批处理的 `runTools()`
- **工具结果**：通过 `normalizeMessagesForAPI()` 规范化以确保 API 兼容性
- **工具使用摘要**：异步 Haiku 调用生成工具批次摘要

### 状态管理

- **AppState**：通过 `getAppState()`/`setAppState()` 读取/写入，用于工具权限、快速模式、文件历史、归属
- **会话状态**：`bootstrap/state.ts` 中的全局状态，用于成本、令牌、持续时间、会话 ID
- **文件状态缓存**：`FileStateCache` 跟踪读取的文件；每个 `QueryEngine` 实例克隆

### MCP（模型上下文协议）

- **MCP 客户端**：通过配置传递；说明包含在系统提示中
- **MCP 工具**：与内置工具一起可用；用于 -32042 错误的 URL 引出处理
- **MCP 资源**：在 tool use 上下文上的 `mcpResources` 中跟踪

### 钩子系统

- **停止钩子**：在模型响应完成后执行
- **TaskCompleted 钩子**：为当前队友拥有的进行中任务运行
- **TeammateIdle 钩子**：在队友空闲时运行
- **采样后钩子**：在采样后触发并忘记（提示建议、内存提取、自动做梦）

### 内存系统

- **内存预取**：`startRelevantMemoryPrefetch()` 在每次迭代时异步运行
- **内存加载**：当自定义系统提示 + 内存路径覆盖时注入 `loadMemoryPrompt()`
- **内存纠正**：当启用自动内存时，在拒绝消息中附加提示

### Transcript/会话存储

- **录制 Transcript**：用户消息立即录制；assistant 消息触发并忘记
- **刷新会话存储**：在 yield 最终结果之前刷新以防止进程杀死时的数据丢失
- **压缩边界**：在写入边界之前刷新保留的段尾部

### 分析

- **Statsig/GrowthBook**：功能门控、A/B 测试值
- **事件日志**：`logEvent()` 用于查询开始/停止、压缩、回退、工具执行、预算
- **OTLP 遥测**：`logOTelEvent()` 用于 API 请求和错误
- **会话跟踪**：带 LLM 请求 spans 的 Beta 跟踪

### SDK 集成

- **SDK 消息**：所有 yield 都转换为 `SDKMessage` 格式供外部消费者使用
- **SDK 状态**：`setSDKStatus()` 回调用于生命周期更新
- **权限拒绝**：跟踪并在最终结果中报告
- **快速模式状态**：报告给 SDK 调用者

## 相关模块

- [Main Entrypoint](./main-entrypoint.md)
- [Tool System](./tool-system.md)
- [State Management](./state-management.md)
