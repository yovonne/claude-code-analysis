# API 服务

## 概述

API 服务是 Claude Code 与 Anthropic API 基础设施之间的核心通信层。它管理客户端创建、认证、请求/响应处理、重试逻辑、错误分类、文件操作和引导数据获取，支持多个 API 提供商（第一方、AWS Bedrock、Azure Foundry、GCP Vertex AI）。

**关键文件：**
- `client.ts` — Anthropic SDK 客户端工厂
- `claude.ts` — 查询执行、请求构建、响应流
- `withRetry.ts` — 带指数退避的重试逻辑
- `errors.ts` — 错误分类和用户面向的消息
- `errorUtils.ts` — 连接错误详情提取
- `bootstrap.ts` — 引导数据获取和缓存
- `filesApi.ts` — 文件上传/下载操作
- `usage.ts` — 利用/配额获取
- `logging.ts` — API 请求/响应日志和指标

---

## API 客户端架构

### 客户端工厂 (`client.ts`)

`getAnthropicClient()` 函数是创建 Anthropic SDK 客户端实例的核心工厂。它根据环境配置动态选择适当的 SDK 变体：

| 提供商 | 环境标志 | SDK 类 | 认证方法 |
|---|---|---|---|
| 第一方 API | （默认） | `Anthropic` | API 密钥或 OAuth 令牌 |
| AWS Bedrock | `CLAUDE_CODE_USE_BEDROCK` | `AnthropicBedrock` | AWS 凭证或 bearer 令牌 |
| Azure Foundry | `CLAUDE_CODE_USE_FOUNDRY` | `AnthropicFoundry` | API 密钥或 Azure AD 令牌 |
| GCP Vertex AI | `CLAUDE_CODE_USE_VERTEX` | `AnthropicVertex` | GoogleAuth (ADC) |

### 客户端配置

每个客户端实例配置如下：

- **默认头：** `x-app: cli`、`User-Agent`、`X-Claude-Code-Session-Id`、可选 `x-claude-remote-container-id`、`x-claude-remote-session-id`、`x-client-app`
- **超时：** `API_TIMEOUT_MS` 环境变量（默认：600 秒）
- **最大重试次数：** 每次调用可配置
- **自定义头：** 从 `ANTHROPIC_CUSTOM_HEADERS` 解析（curl 风格的 `Name: Value` 格式，逗号分隔）
- **代理支持：** 通过 `getProxyFetchOptions()`
- **客户端请求 ID：** UUID 通过 `x-client-request-id` 头注入，仅适用于第一方 API，实现服务器端日志关联

### Fetch 包装器

`buildFetch()` 函数包装全局 `fetch` 以：
1. 为第一方 API 请求注入 `x-client-request-id` UUID
2. 记录每个请求的路径和请求 ID 以进行调试
3. 日志记录错误时永远不会崩溃 fetch

---

## 认证

### 多提供商认证策略

| 提供商 | 认证 |
|---|---|
| 第一方 | `ANTHROPIC_API_KEY` 环境变量、`ANTHROPIC_AUTH_TOKEN` 或 Claude.ai OAuth 令牌 |
| AWS Bedrock | AWS SDK 默认凭证、`AWS_BEARER_TOKEN_BEDROCK` 或通过 `refreshAndGetAwsCredentials()` 刷新的凭证 |
| Azure Foundry | `ANTHROPIC_FOUNDRY_API_KEY` 或 `DefaultAzureCredential` (Azure AD) |
| GCP Vertex AI | `google-auth-library` 带 ADC，通过 `refreshGcpCredentialsIfNeeded()` 刷新 |

### OAuth 令牌管理

- OAuth 令牌在每次 API 调用前通过 `checkAndRefreshOAuthTokenIfNeeded()` 自动检查和刷新
- Claude.ai 订阅者使用 `authToken` 而不是 `apiKey`
- 令牌撤销（403）触发刷新-重试循环
- `ANTHROPIC_AUTH_TOKEN` 环境变量优先于密钥助手解析

### CCR 模式

在 Claude Code Remote (CCR) 模式下，认证通过基础设施提供的 JWT 处理，而不是 `/login`。临时认证错误（401/403）被视为网络问题并重试。

---

## 请求/响应处理

### 查询执行 (`claude.ts`)

支持两种查询模式：

1. **流式：** `queryModelWithStreaming()` — 当令牌到达时产生 `StreamEvent` 块
2. **非流式：** `queryModelWithoutStreaming()` — 等待完整响应

两者都包装核心 `queryModel()` 生成器，它：

1. 解析模型名称（包括 Bedrock 推理配置文件支持模型）
2. 通过 `getMergedBetas()` 确定 beta 头
3. 评估工具搜索资格和延迟工具加载
4. 使用 `toolToAPISchema()` 构建工具模式（包括延迟工具的 `defer_loading`）
5. 规范化消息以实现 API 兼容性
6. 确保 tool_use/tool_result 配对
7. 剥离多余媒体项（每次请求最多 100 个）
8. 计算指纹以进行归因
9. 使用缓存控制标记构建系统提示块
10. 配置 effort、任务预算和输出格式参数
11. 通过 `withRetry()` 执行，流式或非流式回退

### 请求参数

- **模型：** 通过 `normalizeModelStringForAPI()` 规范化
- **系统提示：** 构建为带有 `cache_control` 标记的内容块
- **工具：** 带可选 `defer_loading: true` 的完整模式数组，用于延迟工具
- **最大令牌数：** 来自 `getModelMaxOutputTokens()` 的模型特定默认值
- **温度：** 可配置覆盖
- **Betas：** 从特定于模型、仅限第一方和功能门控头合并
- **额外主体：** 来自 `CLAUDE_CODE_EXTRA_BODY` 环境变量
- **元数据：** 用户 ID、设备 ID、账户 UUID、会话 ID

### Beta 头

粘性开启门闩防止会话中期缓存破坏：
- `afk-mode` — 自动模式（TRANSCRIPT_CLASSIFIER 功能）
- `fast-mode` — 快速模式请求
- `cache-editing` — 缓存微压缩（CACHED_MICROCOMPACT 功能）
- `thinking-clear` — 1 小时缓存 TTL 重置检测

---

## 速率限制

### 速率限制检测

速率限制通过以下方式检测：
- HTTP 429 状态码
- `anthropic-ratelimit-unified-representative-claim` 头（`five_hour`、`seven_day`、`seven_day_opus`）
- `anthropic-ratelimit-unified-overage-status` 头（`allowed`、`allowed_warning`、`rejected`）
- `anthropic-ratelimit-unified-reset` 头（Unix 时间戳）
- HTTP 529 过载错误（消息中包括 `"type":"overloaded_error"`）

### 速率限制响应

重试处理程序（`withRetry.ts`）实现多层级响应：

1. **快速模式短重试：** 如果 `Retry-After < 20s`，等待并重试，保留快速模式
2. **快速模式冷却：** 如果 `Retry-After >= 20s`，进入冷却（切换到标准速度，最少 10 分钟）
3. **超额拒绝：** 如果超额被禁用，永久禁用快速模式
4. **持续重试：** 在无人值守模式（`CLAUDE_CODE_UNATTENDED_RETRY`）中，无限重试：
   - 指数退避，上限为 5 分钟
   - 在 6 小时重置
   - 每 30 秒产生一次心跳以防止空闲检测
5. **前台与后台：** 只有前台查询源（主线程、SDK、代理）在 529 上重试；后台源立即退出

### 重试决策逻辑

`shouldRetry()` 函数考虑：
- 服务器的 `x-should-retry` 头
- 错误类型（连接、超时、速率限制、认证、服务器错误）
- 订阅者类型（Claude.ai 订阅者不重试速率限制；企业用户会）
- CCR 模式（始终重试认证错误）
- 模拟速率限制（永不重试）

---

## 错误处理

### 错误分类 (`errors.ts`)

`classifyAPIError()` 函数将错误映射到分析安全类别：

| 类别 | 触发器 |
|---|---|
| `aborted` | 请求中止 |
| `api_timeout` | 连接超时 |
| `rate_limit` | 429 状态 |
| `server_overload` | 529 状态或 overloaded_error |
| `prompt_too_long` | "prompt is too long" 消息 |
| `pdf_too_large` | PDF 页数超限 |
| `image_too_large` | 图像大小/尺寸超限 |
| `tool_use_mismatch` | tool_use 没有 tool_result |
| `invalid_model` | 无效的模型名称 |
| `credit_balance_low` | 信用余额过低 |
| `invalid_api_key` | x-api-key 错误 |
| `token_revoked` | OAuth 令牌撤销 |
| `ssl_cert_error` | SSL/TLS 证书错误 |
| `connection_error` | 网络连接失败 |
| `server_error` | 5xx 状态码 |
| `client_error` | 4xx 状态码 |

### 用户面向的错误消息

`getAssistantMessageFromError()` 函数将原始 API 错误转换为用户友好的 `AssistantMessage` 对象，包括：
- 上下文感知消息（CLI vs SDK/非交互式）
- 恢复建议（`/login`、`/model`、`/rewind`、`/compact`）
- 用于 UI 渲染的错误类型标签
- 提示过长错误的令牌计数详情
- 3P 模型回退建议

### 连接错误详情 (`errorUtils.ts`)

`extractConnectionErrorDetails()` 函数遍历错误原因链以查找根错误代码，对 SSL/TLS 错误（23 个已知 SSL 错误代码）有特殊处理。`formatAPIError()` 函数生成用户友好的连接错误消息，包含 SSL 特定提示。

### 重试逻辑 (`withRetry.ts`)

`withRetry()` 生成器实现：
- 可配置的最大重试次数（默认：10，通过 `CLAUDE_CODE_MAX_RETRIES` 覆盖）
- 带抖动的指数退避：`BASE_DELAY_MS * 2^(attempt-1) + random(0-25%)`
- 尊重 `Retry-After` 头
- 连续 529 跟踪（Opus 用户回退到 Sonnet 之前最多 3 次）
- 快速模式冷却管理
- 上下文溢出自动调整（400 上减少 `max_tokens`）
- 认证错误时清除 AWS/GCP 凭证缓存
- 中止信号支持
- 用于重试进度的系统消息产生

---

## 引导数据获取

### 目的

引导端点（`/api/claude_cli/bootstrap`）在启动时获取服务器端配置数据：
- `client_data` — 客户端特定配置
- `additional_model_options` — 来自服务器的额外模型选择

### 行为

1. 在仅基本流量模式下跳过
2. 为非第一方提供商跳过
3. 需要 OAuth 令牌（带 `user:profile` 范围）或 API 密钥
4. 使用 `withOAuth401Retry()` 进行令牌刷新-重试
5. 针对 Zod 模式验证响应
6. 仅在数据实际更改时持久化到磁盘缓存（避免每次启动时配置写入）
7. 缓存在全局配置中为 `clientDataCache` 和 `additionalModelOptionsCache`

---

## 文件 API

### 概述 (`filesApi.ts`)

文件 API 客户端管理到 Anthropic 公共文件 API（`/v1/files`）的文件上传和下载。用于：
- 在会话启动时下载文件附件
- 在 BYOC（自带计算）模式下上传文件
- 列出创建后时间戳后创建的文件（1P/云模式）

### 配置

```typescript
type FilesApiConfig = {
  oauthToken: string    // 来自会话 JWT 的 OAuth 令牌
  baseUrl?: string      // 默认: https://api.anthropic.com
  sessionId: string     // 会话特定目录
}
```

### 下载操作

- **`downloadFile(fileId, config)`** — 通过指数退避（3 次重试，500ms 基础延迟）按 ID 下载单个文件
- **`downloadAndSaveFile(attachment, config)`** — 下载并保存到 `{cwd}/{sessionId}/uploads/{relativePath}`
- **`downloadSessionFiles(files, config, concurrency=5)`** — 带并发限制的并行下载

### 上传操作

- **`uploadFile(filePath, relativePath, config)`** — 通过 multipart/form-data 上传单个文件
  - 最大文件大小：500MB
  - 超时：120s
  - 不可重试错误：401、403、413、cancel
- **`uploadSessionFiles(files, config, concurrency=5)`** — 带并发限制的并行上传

### 列表操作

- **`listFilesCreatedAfter(afterCreatedAt, config)`** — 通过游标分页列出创建后时间戳后的文件 `after_id`

### 路径安全

`buildDownloadPath()` 通过以下方式防止路径遍历攻击：
- 规范化路径
- 拒绝以 `..` 开头的路径
- 剥离多余的上传前缀

### Beta 头

文件 API 使用 beta 头：`files-api-2025-04-14,oauth-2025-04-20`

---

## 利用率 API

### 概述 (`usage.ts`)

从 `/api/oauth/usage` 获取 Claude.ai 订阅者的速率限制利用率数据：

```typescript
type Utilization = {
  five_hour?: RateLimit | null
  seven_day?: RateLimit | null
  seven_day_opus?: RateLimit | null
  seven_day_sonnet?: RateLimit | null
  extra_usage?: ExtraUsage | null
}

type RateLimit = {
  utilization: number | null  // 0-100 百分比
  resets_at: string | null    // ISO 8601 时间戳
}
```

如果 OAuth 令牌过期或用户缺少 profile 范围，则跳过 API 调用。

---

## 日志和指标

### API 日志 (`logging.ts`)

- **`logAPIQuery()`** — 记录带模型、工具、缓存策略的请求开始
- **`logAPISuccessAndDuration()`** — 记录带令牌使用、成本、持续时间、缓存统计的响应
- **`logAPIError()`** — 记录带分类和连接详情的错误

### 令牌使用跟踪

- 来自 API 响应的输入/输出/缓存令牌计数
- 通过 `calculateUSDCost()` 计算成本
- 通过 `addToTotalSessionCost()` 跟踪会话成本
- 引导状态中的持续时间跟踪

### 网关检测

检查响应头以检测 AI 网关代理（LiteLLM、Helicone、Portkey、Cloudflare AI Gateway、Kong、Braintrust、Databricks）以进行准确的提供商识别。

### 提示缓存中断检测

跟踪缓存命中/未命中模式并检测缓存中断（由于内容更改），从而优化可缓存提示部分。
