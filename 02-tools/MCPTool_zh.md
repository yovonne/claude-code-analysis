# MCPTool

## 用途

MCPTool 是表示任何连接的模型上下文协议（MCP）服务器提供的工具的通用包装器。它充当伪工具存根，其实际行为（名称、描述、schema、调用逻辑）在 MCP 服务器连接时在运行时被覆盖。存根定义接口契约；`client.ts` 为每个发现的工具填充真实实现。它处理工具调用、通过 MCP SDK 进行服务器通信、进度流式传输、结果转换、截断和二进制内容持久化。

## 位置

- `restored-src/src/tools/MCPTool/MCPTool.ts` — 主要工具存根定义（78 行）
- `restored-src/src/tools/MCPTool/UI.tsx` — 工具使用、进度和结果的终端 UI 渲染（403 行）
- `restored-src/src/tools/MCPTool/prompt.ts` — 占位符提示/描述（运行时覆盖）（4 行）
- `restored-src/src/tools/MCPTool/classifyForCollapse.ts` — 用于 UI 折叠的搜索/读取分类（605 行）
- `restored-src/src/services/mcp/client.ts` — 运行时工具注册、调用逻辑、结果处理（约 2800 行）
- `restored-src/src/services/mcp/auth.ts` — OAuth 认证、令牌管理、ClaudeAuthProvider（约 1700 行）
- `restored-src/src/services/mcp/types.ts` — 服务器配置 schema、连接类型（259 行）
- `restored-src/src/services/mcp/mcpStringUtils.ts` — 工具名称解析、前缀生成（107 行）
- `restored-src/src/utils/mcpOutputStorage.ts` — 二进制内容持久化（190 行）
- `restored-src/src/utils/mcpValidation.ts` — 内容大小估计、截断逻辑
- `restored-src/src/utils/mcpWebSocketTransport.ts` — WebSocket 传输包装器
- `restored-src/src/services/mcp/elicitationHandler.ts` — 启发请求处理
- `restored-src/src/services/mcp/headersHelper.ts` — 动态头生成
- `restored-src/src/services/mcp/normalization.ts` — 名称规范化实用工具
- `restored-src/src/services/mcp/oauthPort.ts` — OAuth 重定向端口管理（79 行）
- `restored-src/src/utils/mcp/dateTimeParser.ts` — MCP 结果的日期/时间解析
- `restored-src/src/utils/mcp/elicitationValidation.ts` — 启发输入验证

## 主要导出

### 来自 MCPTool.ts

| 导出 | 描述 |
|--------|-------------|
| `MCPTool` | 通过 `buildTool()` 构建的完整工具存根定义 |
| `inputSchema` | 延迟 Zod schema：`z.object({}).passthrough()` — 允许任何输入 |
| `outputSchema` | 延迟 Zod schema：`z.string()` — MCP 工具执行结果 |
| `Output` | Zod 推断的输出类型（字符串） |
| `MCPProgress` | 进度数据类型（从 `types/tools.js` 重新导出） |

### 来自 UI.tsx

| 导出 | 描述 |
|--------|-------------|
| `renderToolUseMessage(input, { verbose })` | 渲染工具调用参数 |
| `renderToolUseProgressMessage(progressMessages)` | 使用进度条渲染进度 |
| `renderToolResultMessage(output, _, { verbose, input })` | 使用丰富格式化渲染工具结果 |
| `tryFlattenJson(content)` | 扁平化 JSON 对象为键值显示对 |
| `tryUnwrapTextPayload(content)` | 从 JSON 包装器中提取主导文本 |
| `trySlackSendCompact(output, input)` | 检测 Slack 发送结果以紧凑显示 |

### 来自 classifyForCollapse.ts

| 导出 | 描述 |
|--------|-------------|
| `classifyMcpToolForCollapse(serverName, toolName)` | 返回用于 UI 折叠的 `{ isSearch, isRead }` |
| `SEARCH_TOOLS` | 跨 MCP 服务器的约 120+ 个搜索工具名称集合 |
| `READ_TOOLS` | 跨 MCP 服务器的约 400+ 个读取工具名称集合 |

### 来自 client.ts（MCP 工具执行）

| 导出 | 描述 |
|--------|-------------|
| `fetchToolsForClient(client)` | 通过 `tools/list` 进行 LRU 缓存的工具发现 |
| `callMCPTool(params)` | 带会话重试的核心工具调用 |
| `callMCPToolWithUrlElicitationRetry(params)` | 带 URL 启发回退的工具调用 |
| `transformMCPResult(result, tool, name)` | 将工具结果规范化为 content/type/schema |
| `processMCPResult(result, tool, name)` | 带截断/持久化的完整结果处理 |
| `McpAuthError` | 认证失败错误类 |
| `McpToolCallError` | 工具执行失败错误类（带有 `_meta`） |
| `McpSessionExpiredError` | 过期的 MCP 会话错误 |
| `isMcpSessionExpiredError(error)` | 检测 404 + JSON-RPC -32001 会话过期 |
| `mcpToolInputToAutoClassifierInput(input, toolName)` | 为自动模式安全分类器编码输入 |

## 输入/输出 Schema

### 输入 Schema

```typescript
// 直通 — 允许任何输入对象，因为 MCP 工具定义自己的 schema
{
  // MCP 服务器工具定义的任何内容
  [key: string]: unknown
}
```

实际输入 schema 在运行时由 `fetchToolsForClient` 按工具设置，它从 MCP 服务器的 `tools/list` 响应中读取 `tool.inputSchema` 并将其存储为 `inputJSONSchema`。

### 输出 Schema

```typescript
{
  data: string | ContentBlockParam[] | MCPToolResult,  // 工具结果内容
  mcpMeta?: {
    _meta?: Record<string, unknown>,                   // MCP 规范 _meta
    structuredContent?: unknown,                        // 结构化内容
  },
}
```

输出可以是：
- 纯字符串（文本结果）
- `ContentBlockParam` 数组（text、image、resource 块）
- `MCPToolResult`（字符串或内容块数组的联合）

## MCP 工具调用

### 工具发现管道

```
1. 服务器连接
   - connectToServer() 建立传输（stdio/sse/http/ws）
   - 客户端与服务器协商能力

2. 工具列表
   - fetchToolsForClient() 发送 tools/list 请求
   - 按服务器名称进行 LRU 缓存（最多 20 个条目）
   - 在 onclose 和重新连接时失效

3. 工具注册
   - 每个 MCP 工具被包装为 Tool 对象：
     a. name: mcp__<server>__<tool>（完全限定）
     b. mcpInfo: { serverName, toolName }
     c. isMcp: true
     d. description/prompt: 来自服务器元数据（最多 2048 字符）
     e. inputJSONSchema: 来自服务器的工具定义
     f. checkPermissions: 返回直通（用户可配置）
     g. call: 委托给 callMCPToolWithUrlElicitationRetry
     h. isConcurrencySafe: 来自 tool.annotations.readOnlyHint
     i. isReadOnly: 来自 tool.annotations.readOnlyHint
     j. isSearchOrReadCommand: 来自 classifyMcpToolForCollapse

4. 资源工具
   - 具有资源能力的第一个服务器获取：
     - ListMcpResourcesTool（列出可用资源）
     - ReadMcpResourceTool（读取特定资源）
   - 仅跨所有服务器添加一次（resourceToolsAdded 标志）
```

### 工具调用执行

```
1. 模型请求
   - 模型使用 { args } 调用 mcp__server__tool

2. 会话保证
   - ensureConnectedClient() 检查连接健康
   - 如果 onclose 清除备忘录缓存则重新连接
   - 如果服务器无法连接则抛出

3. 工具调用
   - callMCPToolWithUrlElicitationRetry() 发送：
     { method: "tools/call", params: { name, arguments: args, _meta } }
   - _meta 包括 claudecode/toolUseId 用于进度关联

4. 进度流式传输
   - onProgress 回调触发 MCPProgress：
     { type: 'mcp_progress', status: 'started'|'completed'|'failed', ... }
   - 进度包括 serverName、toolName、elapsedTimeMs

5. 会话重试
   - McpSessionExpiredError 触发一次自动重试
   - MAX_SESSION_RETRIES = 1
   - 通过 ensureConnectedClient 获取新鲜客户端

6. 结果转换
   - transformMCPResult() 规范化响应：
     a. toolResult → 纯文本
     b. structuredContent → JSON 字符串 + 推断的 schema
     c. content array → 转换的 ContentBlockParam[]
   - processMCPResult() 处理大输出：
     a. 通过 getContentSizeEstimate() 进行大小估计
     b. 如果太大：持久化到文件或截断
     c. 图像内容 → 截断（不是持久化）

7. 错误处理
   - MCP SDK 错误包装在 TelemetrySafeError 中
   - McpError 代码保留在消息中（例如 "McpError -32000"）
   - McpToolCallError 为 SDK 消费者带有 _meta
```

### 工具调用代码路径

```typescript
async call(args, context, _canUseTool, parentMessage, onProgress) {
  const connectedClient = await ensureConnectedClient(client)
  const mcpResult = await callMCPToolWithUrlElicitationRetry({
    client: connectedClient,
    tool: tool.name,
    args,
    meta: { 'claudecode/toolUseId': toolUseId },
    signal: context.abortController.signal,
    onProgress,
    handleElicitation: context.handleElicitation,
  })
  return {
    data: mcpResult.content,
    mcpMeta: { _meta, structuredContent },
  }
}
```

## 服务器通信

### 传输类型

| 传输 | 协议 | 认证 | 用例 |
|-----------|----------|------|----------|
| stdio | stdin/stdout | 无 | 本地 MCP 服务器（子进程） |
| sse | SSE + HTTP POST | OAuth (ClaudeAuthProvider) | 远程流式服务器 |
| http | Streamable HTTP | OAuth (ClaudeAuthProvider) | 现代 MCP 服务器（2025-03-26 规范） |
| ws | WebSocket | 自定义头 | IDE 扩展 |
| sse-ide | SSE | 无（IDE 认证令牌） | IDE 扩展 |
| ws-ide | WebSocket | X-Claude-Code-Ide-Authorization | IDE 扩展 |
| claudeai-proxy | 通过代理的 Streamable HTTP | Claude.ai OAuth 令牌 | Claude.ai 连接器服务器 |
| sdk | 进程内 | 无 | SDK 嵌入式服务器 |

### 连接生命周期

```
1. 初始化
   - 根据服务器配置类型创建传输
   - 为 sse/http 附加认证提供程序 (ClaudeAuthProvider)
   - 合并头：User-Agent + 静态 + 动态 + 会话入口

2. 连接
   - 带超时（环境 MCP_TIMEOUT，默认 30s）的 client.connect(transport)
   - 设置能力：roots、elicitation
   - 请求处理程序：ListRoots、ElicitRequest（取消默认）

3. 健康监控
   - onerror：跟踪连续终端错误（最多 3 次）
   - 检测：ECONNRESET、ETIMEDOUT、EPIPE、EHOSTUNREACH、ECONNREFUSED
   - 会话过期：HTTP 404 + JSON-RPC code -32001
   - SSE 重新连接耗尽："Maximum reconnection attempts"

4. 重新连接
   - onclose 清除备忘录缓存：
     - connectToServer.cache
     - fetchToolsForClient.cache
     - fetchResourcesForClient.cache
     - fetchCommandsForClient.cache
   - 下一次 ensureConnectedClient() 触发新连接

5. 清理
   - stdio: SIGINT → 100ms → SIGTERM → 400ms → SIGKILL（总共 500ms）
   - 进程内：server.close() 然后 client.close()
   - 通过 registerCleanup() 注册用于进程关闭
```

### Fetch 包装堆栈

对于 HTTP/SSE 传输，fetch 调用通过层：

```
fetch()
  → wrapFetchWithTimeout()          // 每个请求 60s 超时，Accept 头
    → wrapFetchWithStepUpDetection() // 403 insufficient_scope → markStepUpPending
      → createFetchWithInit()       // 带初始化合并的基础 fetch
```

- **wrapFetchWithTimeout**：每个请求的新鲜超时信号（不是过时的），保证 `Accept: application/json, text/event-stream` 用于 Streamable HTTP
- **wrapFetchWithStepUpDetection**：检测 WWW-Authenticate 头中的 403 `insufficient_scope`，提取作用域，调用 `provider.markStepUpPending(scope)` 以触发升级 OAuth 流程
- GET 请求跳过超时（长期存在的 SSE 流）

### 批量服务器连接

```
本地服务器（stdio/sdk）：并发 = getMcpServerConnectionBatchSize()（默认 3）
远程服务器（sse/http/ws/claudeai-proxy）：并发 = getRemoteMcpServerConnectionBatchSize()（默认 20）

使用基于插槽的 pMap 进行调度（不是刚性批次）：
- 每个插槽在服务器完成时释放
- 慢速服务器占据一个插槽，不会阻塞整个批次
```

### 认证缓存

```
MCP_AUTH_CACHE_TTL_MS = 15 分钟
缓存文件：mcp-needs-auth-cache.json

在连接时出现 401：
1. handleRemoteAuthFailure() 发出 tengu_mcp_server_needs_auth
2. setMcpAuthCacheEntry() 写入时间戳
3. 服务器标记为 needs-auth，创建认证工具

重新连接时：
- isMcpAuthCached() 检查 TTL
- hasMcpDiscoveryButNoToken() 检查安全存储
- 如果两者都满足则跳过连接（防止 401 循环）
```

## 工具结果处理

### 结果转换

```typescript
transformMCPResult(result, tool, name) → { content, type, schema }

输入形状：
  { toolResult: string }           → type: 'toolResult'
  { structuredContent: object }    → type: 'structuredContent', schema: inferred
  { content: ContentBlock[] }      → type: 'contentArray', schema: inferred

Schema 推断 (inferCompactSchema)：
  - 深度限制（depth=2）
  - 对象：{key: type, ...}（最多 10 个键）
  - 数组：[elementType]
  - 原始类型：typeof value
```

### 大输出处理

```
1. 大小估计：getContentSizeEstimate(content) → 令牌计数
2. 如果超过阈值：
   a. ENABLE_MCP_LARGE_OUTPUT_FILES 禁用 → 内联截断
   b. 包含图像 → 截断（保持压缩）
   c. 否则 → 持久化到文件：
      - persistToolResult() 将 JSON 写入 tool-results 目录
      - 返回带有路径和说明的 <persisted-output>
      - 说明包括：格式描述、jq 用法、分块读取

3. 截断：
   - 通过 EndTruncatingAccumulator 使用 truncateMcpContentIfNeeded()
   - 尽可能保持内容结构
```

### 二进制内容持久化

```
对于工具结果中的 blob 内容：
1. persistBinaryContent(bytes, mimeType, persistId)
   - 从 MIME 类型派生扩展（pdf、json、png 等）
   - 写入 tool-results 目录
   - 返回 { filepath, size, ext }

2. getBinaryBlobSavedMessage(filepath, mimeType, size, sourceDescription)
   - 返回："Binary content (mime, size) saved to filepath"

3. 图像内容（jpeg、png、gif、webp）：
   - 通过 maybeResizeAndDownsampleImageBuffer() 调整大小
   - API 限制强制执行最大尺寸
   - 作为 base64 图像块返回
```

### UI 渲染策略

```
renderToolResultMessage 对文本内容使用三种策略：

1. 文本有效载荷展开 (tryUnwrapTextPayload)：
   - 检测带有主导字符串字段的 JSON（例如 {"messages": "..."}）
   - 展开主体，将额外内容显示为暗淡元数据
   - 阈值：string > 200 字符或包含换行符 + > 50 字符

2. JSON 扁平化 (tryFlattenJson)：
   - 小扁平对象（≤ 12 个键，≤ 5000 字符）
   - 渲染为对齐的 key: value 对
   - 嵌套对象 → 单行 JSON（≤ 120 字符）

3. 回退 (OutputLine)：
   - 漂亮打印的 JSON 带截断
   - URL 链接化

特殊情况：
- Slack 发送结果 → 紧凑的 "Sent message to #channel" 显示
- 大响应（> 10,000 令牌）→ 警告横幅
- 图像块 → [Image] 占位符
```

### 进度显示

```
renderToolUseProgressMessage(progressMessages)：
  - 无进度数据 → "Running…"
  - 带总数的进度 → ProgressBar + 百分比
  - 无总数的进度 → "Processing… N" 或自定义消息

进度事件：
  { type: 'mcp_progress', status: 'started'|'completed'|'failed',
    serverName, toolName, elapsedTimeMs }
```

## 用于 UI 折叠的工具分类

### classifyMcpToolForCollapse

```
输入：serverName, toolName
输出：{ isSearch: boolean, isRead: boolean }

规范化：
  - camelCase → snake_case: searchCode → search_code
  - kebab-case → snake_case: search-files → search_files
  - 小写

SEARCH_TOOLS（约 120 个条目）：
  - Slack: slack_search_public, slack_search_channels, ...
  - GitHub: search_code, search_repositories, search_issues, ...
  - Datadog: search_logs, search_spans, find_slow_spans, ...
  - Sentry: search_docs, search_events, find_organizations, ...
  - Brave: brave_web_search, brave_local_search
  - Exa: web_search_exa, deep_search_exa, ...
  - 等等（Notion、Asana、MongoDB、Airtable 等）

READ_TOOLS（约 400 个条目）：
  - GitHub: get_me, get_file_contents, list_branches, ...
  - Linear: list_projects, get_team, list_users, ...
  - Datadog: query_metrics, list_monitors, get_dashboard, ...
  - Filesystem: read_file, list_directory, get_file_info, ...
  - Git: git_status, git_diff, git_log, git_show, ...
  - Kubernetes: kubectl_get, kubectl_describe, pods_list, ...
  - 等等（PagerDuty、Supabase、Stripe 等）

未知工具名称 → { isSearch: false, isRead: false }（保守）
```

## 配置

### 环境变量

| 变量 | 用途 |
|----------|---------|
| `MCP_TOOL_TIMEOUT` | 工具调用超时（毫秒，默认约 27.8 小时） |
| `MCP_TIMEOUT` | 连接超时（毫秒，默认 30000） |
| `MCP_SERVER_CONNECTION_BATCH_SIZE` | 本地服务器并发（默认 3） |
| `MCP_REMOTE_SERVER_CONNECTION_BATCH_SIZE` | 远程服务器并发（默认 20） |
| `MCP_OAUTH_CALLBACK_PORT` | 固定 OAuth 重定向端口 |
| `CLAUDE_CODE_ENABLE_XAA` | 启用跨应用访问认证 |
| `ENABLE_MCP_LARGE_OUTPUT_FILES` | 为大结果启用文件持久化 |
| `CLAUDE_AGENT_SDK_MCP_NO_PREFIX` | 为 SDK 工具跳过 mcp__ 前缀 |

### 特性标志

| 标志 | 用途 |
|------|---------|
| `MCP_RICH_OUTPUT` | 启用丰富的 UI 渲染（文本展开、JSON 扁平化） |
| `MCP_SKILLS` | 从 MCP 资源启用技能发现 |
| `CHICAGO_MCP` | 计算机使用 MCP 服务器支持 |

## 依赖

### 内部

| 模块 | 用途 |
|--------|---------|
| `services/mcp/client.ts` | 核心 MCP 客户端、工具调用、连接管理 |
| `services/mcp/auth.ts` | OAuth 流程、ClaudeAuthProvider、令牌管理 |
| `services/mcp/types.ts` | 服务器配置 schema、连接类型 |
| `services/mcp/mcpStringUtils.ts` | 工具名称构建、前缀生成 |
| `services/mcp/normalization.ts` | MCP 名称规范化 |
| `utils/mcpOutputStorage.ts` | 二进制内容持久化 |
| `utils/mcpValidation.ts` | 内容大小估计、截断 |
| `utils/mcpWebSocketTransport.ts` | WebSocket 传输包装器 |
| `utils/toolResultStorage.ts` | 大结果持久化到磁盘 |
| `utils/imageResizer.ts` | API 限制的图像调整大小 |
| `utils/errors.ts` | 错误处理实用工具 |
| `utils/log.ts` | MCP 调试/错误日志 |
| `utils/slowOperations.ts` | 带限制的 JSON 解析/字符串化 |
| `utils/memoize.ts` | Fetch 缓存的 LRU 记忆化 |
| `utils/sanitization.ts` | 服务器数据的 Unicode 清理 |
| `utils/proxy.ts` | HTTP/WebSocket 代理支持 |
| `utils/mtls.ts` | mTLS WebSocket 选项 |
| `utils/subprocessEnv.ts` | stdio 子进程的环境 |
| `utils/abortController.ts` | Abort 控制器创建 |
| `utils/cleanupRegistry.ts` | 进程关闭清理 |
| `services/analytics/` | 事件日志 |

### 外部

| 包 | 用途 |
|---------|---------|
| `@modelcontextprotocol/sdk` | MCP 协议实现 |
| `@anthropic-ai/sdk` | API 类型（ContentBlockParam 等） |
| `zod/v4` | 输入/输出 schema 验证 |
| `react` | UI 渲染 |
| `lodash-es` | reject、mapValues、memoize、zipObject |
| `p-map` | 带并发限制的并发处理 |
| `figures` | 终端图标（警告符号） |
| `axios` | 用于令牌撤销的 HTTP 客户端 |
| `ws` | WebSocket 客户端（Node.js） |

## 数据流

```
Model Request
    │
    ▼
MCPTool Input { [args from server schema] }
    │
    ▼
┌─────────────────────────────────────────────────────────────────┐
│ TOOL CALL EXECUTION                                             │
│                                                                 │
│  1. ensureConnectedClient()                                     │
│     - Check memo cache                                          │
│     - Reconnect if needed                                       │
│                                                                 │
│  2. callMCPToolWithUrlElicitationRetry()                      │
│     - Send tools/call request                                   │
│     - Handle elicitation (URL confirmation)                     │
│     - Stream progress via onProgress                            │
│                                                                 │
│  3. Session retry (max 1)                                       │
│     - McpSessionExpiredError → fresh client → retry             │
│                                                                 │
│  4. transformMCPResult()                                        │
│     - toolResult → string                                       │
│     - structuredContent → JSON + schema                         │
│     - content array → ContentBlockParam[]                       │
│                                                                 │
│  5. processMCPResult()                                          │
│     - Size estimation                                           │
│     - Large output → persist or truncate                         │
│     - Image content → special handling                          │
└─────────────────────────────────────────────────────────────────┘
    │
    ▼
Output { data: content, mcpMeta?: { _meta, structuredContent } }
    │
    ▼
┌─────────────────────────────────────────────────────────────────┐
│ UI RENDERING                                                    │
│                                                                 │
│  renderToolResultMessage:                                       │
│  - Slack compact → "Sent to #channel"                           │
│  - Large warning → "~N tokens" banner                          │
│  - Text unwrap → body + extras                                  │
│  - JSON flatten → aligned key: value                            │
│  - Fallback → pretty-printed JSON                               │
│                                                                 │
│  renderToolUseProgressMessage:                                  │
│  - Progress bar with percentage                                 │
│  - "Running…" / "Processing…" fallback                          │
└─────────────────────────────────────────────────────────────────┘
    │
    ▼
Model Response (tool_result block)
```
