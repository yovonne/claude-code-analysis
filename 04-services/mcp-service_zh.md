# MCP 服务

## 概述

MCP（模型上下文协议）服务管理到 MCP 服务器的连接，使 Claude Code 能够通过标准化的 MCP 协议与外部工具、资源和提示进行交互。它支持多种传输类型（stdio、SSE、HTTP、WebSocket、SDK、进程内）、OAuth 认证、跨多个作用域的服务器配置以及官方 MCP 注册表。

**关键文件：**
- `types.ts` — 类型定义、配置模式、连接状态
- `client.ts` — MCP 客户端创建、传输设置、工具执行
- `config.ts` — 多作用域配置管理、策略过滤
- `auth.ts` — MCP 服务器的 OAuth 认证
- `officialRegistry.ts` — 官方 MCP 服务器 URL 注册表
- `normalization.ts` — 服务器名称规范化
- `utils.ts` — 工具/命令/资源过滤工具
- `MCPConnectionManager.tsx` — 用于连接管理的 React 上下文
- `useManageMCPConnections.ts` — 连接生命周期钩子

---

## MCP 客户端架构

### 客户端创建 (`client.ts`)

`connectToServer()` 函数（按服务器名称 + 配置记忆化）是建立 MCP 连接的核心入口点。它：

1. 根据服务器配置类型选择适当的传输
2. 创建具有能力声明的 MCP `Client` 实例
3. 通过传输连接，带超时保护
4. 注册请求处理程序（ListRoots、Elicitation）
5. 设置连接生命周期监控（错误/关闭处理程序）
6. 返回 `ConnectedMCPServer`、`FailedMCPServer` 或 `NeedsAuthMCPServer`

### 客户端实例

每个 MCP 客户端初始化为：
```typescript
new Client({
  name: 'claude-code',
  title: 'Claude Code',
  version: MACRO.VERSION,
  description: "Anthropic's agentic coding tool",
  websiteUrl: PRODUCT_URL,
}, {
  capabilities: {
    roots: {},        // 支持 listRoots
    elicitation: {},  // 支持 elicitation 请求
  },
})
```

### 连接状态

服务器经历五种状态（定义在 `types.ts`）：

| 状态 | 描述 |
|---|---|
| `connected` | 具有客户端能力的活动连接、服务器信息 |
| `failed` | 连接失败，错误详情 |
| `needs-auth` | 需要 OAuth 认证 |
| `pending` | 连接进行中，重连跟踪 |
| `disabled` | 服务器被用户明确禁用 |

### 记忆化

`connectToServer` 使用 lodash `memoize` 进行记忆化，缓存键为 `${name}-${JSON.stringify(serverRef)}`，防止在同一会话中重复连接到相同服务器配置。

---

## 服务器管理（传输）

### 传输类型

服务支持八种传输类型，每种都有独特的设置：

#### stdio

- 通过 `StdioClientTransport` 生成子进程
- 来自配置的命令，args 与 env 变量合并
- `CLAUDE_CODE_SHELL_PREFIX` env 变量包装命令以执行 shell
- stderr 用于调试（上限 64MB 以防止内存增长）
- 子进程环境从 `subprocessEnv()` + 服务器特定 env 构建

#### SSE（服务器发送事件）

- 使用 `SSEClientTransport` 与 `ClaudeAuthProvider`
- EventSource 连接（GET）**不**包装超时（长寿命流）
- POST 请求包装了 `wrapFetchWithTimeout()`（每个请求 60s）
- 升级认证检测包装最内层以获取 403 可见性
- 来自静态配置的自定义头 + 动态 `headersHelper` 函数

#### SSE-IDE

- 用于 IDE 扩展的变体
- 不需要认证
- 支持代理的 EventSource 设置

#### HTTP（流式 HTTP）

- 使用 `StreamableHTTPClientTransport`
- 根据 MCP 规范需要 `Accept: application/json, text/event-stream` 头
- 带升级检测的 OAuth 认证提供程序
- CCR 代理模式的会话入口令牌支持
- 连接尝试前的 HTTP 连接测试
- 基本连接日志（URL 解析、DNS 解析）

#### WebSocket

- 使用包装 `ws` 或原生 WebSocket 的自定义 `WebSocketTransport`
- 协议：`['mcp']`
- 通过 `getWebSocketTLSOptions()` 的 TLS 选项
- 通过 `getWebSocketProxyAgent()` 的代理支持
- 通过 `X-Claude-Code-Ide-Authorization` 头的会话入口认证令牌

#### WS-IDE

- IDE 特定的 WebSocket 变体
- 来自配置的认证令牌
- IDE 名称跟踪

#### SDK

- SDK 管理传输的占位符
- 工具调用通过 `mcp_tool_call` 路由回 SDK
- 免于企业策略过滤

#### claudeai-proxy

- 通过 Claude.ai MCP 代理路由
- URL 从 OAuth 配置构建：`{MCP_PROXY_URL}{MCP_PROXY_PATH.replace('{server_id}', id)}`
- 自定义 fetch 包装器（`createClaudeAiProxyFetch`），在 401 时刷新 OAuth 令牌
- 会话 ID 头：`X-Mcp-Client-Session-Id`

#### 进程内（特殊 stdio）

- Chrome MCP 和 Computer Use MCP 运行在进程内以避免约 325MB 子进程开销
- 使用带有链接传输对的 `InProcessTransport`
- 服务器通过工厂函数创建，连接到服务器端传输

### 连接超时

- 默认：`MCP_TIMEOUT` env 变量或 30 秒
- 通过 `Promise.race([connectPromise, timeoutPromise])` 强制执行
- 超时清理进程内服务器和传输

### 连接丢失检测

增强的错误监控跟踪：
- 连续连接错误（阈值：手动关闭前 3 次）
- 终端错误检测：`ECONNRESET`、`ETIMEDOUT`、`EPIPE`、`EHOSTUNREACH`、`ECONNREFUSED`、`Body Timeout Error`、`terminated`、SSE 流错误
- 重入保护防止飞行中流中止期间双重关闭
- 手动关闭在下一次工具调用时通过 memo 缓存清除触发重连

### 会话过期

`isMcpSessionExpiredError()` 通过 HTTP 404 + JSON-RPC 代码 `-32001` 检测过期的 MCP 会话。抛出 `McpSessionExpiredError` 以向调用者发出重新连接信号。

---

## 工具发现和注册

### 工具发现

连接后通过 `client.listTools()` 发现工具：

1. 从连接的服务器获取工具
2. 每个工具包装在 `MCPTool` 实例中
3. 工具名称规范化：`mcp__<normalized_server_name>__<tool_name>`
4. 描述上限为 2048 个字符（防止 OpenAPI 生成的膨胀）
5. 工具结果持久化通过 `persistToolResult` 配置

### 工具命名

`buildMcpToolName()` 函数构建工具名称：
```
mcp__<normalized_server_name>__<original_tool_name>
```

规范化（`normalizeNameForMCP`）：
- 用下划线替换无效字符（点、空格）
- 对于 claude.ai 服务器：合并连续下划线，剥离前导/尾随
- API 兼容模式：`^[a-zA-Z0-9_-]{1,64}$`

### 工具执行

`callTool()` 在 MCP 客户端上：
- 默认超时：`MCP_TOOL_TIMEOUT` env 变量或约 27.8 小时（实际上无限）
- 结果内容处理：文本、图像、二进制内容持久化
- 图像内容通过 `maybeResizeAndDownsampleImageBuffer()` 调整大小/下采样
- 二进制内容通过 `persistBinaryContent()` 持久化到磁盘
- 如果内容超出限制则应用截断
- `isError: true` 结果抛出 `McpToolCallError`（为消费者携带 `_meta`）
- 认证失败（401）抛出 `McpAuthError` 并将服务器状态更新为 `needs-auth`

### IDE 工具过滤

对于 IDE MCP 服务器，仅包含特定工具：
- `mcp__ide__executeCode`
- `mcp__ide__getDiagnostics`

---

## 资源处理

### 资源发现

通过 `client.listResources()` 在连接后发现资源：
- 每个资源标记其服务器名称：`ServerResource = Resource & { server: string }`
- 存储在连接状态的每个服务器中

### 资源读取

`ReadMcpResourceTool` 处理资源读取：
- 调用 `client.readResource(uri)`
- 内容处理类似于工具结果（文本、图像、二进制）
- 通过 `capabilities.resources?.subscribe` 检测资源订阅支持

### 资源列表

`ListMcpResourcesTool` 提供 CLI 工具以列出所有连接服务器上的资源。

---

## MCP 协议实现

### 请求处理程序

客户端为服务器到客户端请求注册处理程序：

#### ListRoots

```typescript
client.setRequestHandler(ListRootsRequestSchema, async () => ({
  roots: [{ uri: `file://${getOriginalCwd()}` }],
}))
```

返回项目根作为唯一可用的根。

#### Elicitation

- 在初始化期间默认处理程序返回 `{ action: 'cancel' }`
- 在 `useManageMCPConnections` 中被 `registerElicitationHandler` 覆盖
- 钩子：`runElicitationHooks()` 和 `runElicitationResultHooks()` 管理 elicitation 生命周期
- 通过 `ElicitationWaitingState` 跟踪等待状态

### 能力声明

客户端向服务器声明这些能力：
- `roots: {}` — 支持根列表
- `elicitation: {}` — 支持 elicitation（空对象以避免破坏 Java MCP SDK 服务器）

### 传输协议详情

#### 流式 HTTP Accept 头

所有 HTTP POST 请求包括：
```
Accept: application/json, text/event-stream
```

这由 `wrapFetchWithTimeout()` 强制执行，它规范化头并保证值，因为某些运行时/代理从 SDK 设置的头中删除它。

#### 请求超时包装

`wrapFetchWithTimeout()`：
- 跳过 GET 请求（长寿命 SSE 流）
- 为每个请求创建新的 `AbortController`（避免 Bun 中 `AbortSignal.timeout()` 内存泄漏）
- POST 请求 60 秒超时
- 正确清理计时器和中止监听器

#### 升级认证检测

`wrapFetchWithStepUpDetection()` 包装 fetch 以检测 403 升级认证要求，并在 SDK 的认证处理程序看到错误之前触发重新认证。

---

## OAuth 认证

### ClaudeAuthProvider (`auth.ts`)

为 MCP SDK 认证实现 `OAuthClientProvider`：

1. **元数据发现：** 从 `.well-known/oauth-authorization-server` 或配置的 `authServerMetadataUrl` 获取 OAuth 元数据
2. **动态客户端注册：** 如果没有存储的客户端信息则注册客户端
3. **令牌交换：** 带本地 HTTP 服务器回调的 PKCE 流程
4. **令牌刷新：** 自动刷新，重试瞬态错误
5. **令牌存储：** 通过平台 keychain/安全存储的安全存储

### OAuth 流程

1. 找到回调的可用端口（`findAvailablePort`）
2. 使用 `buildRedirectUri` 构建重定向 URI
3. 打开浏览器进行授权
4. 本地 HTTP 服务器接收回调
5. 交换代码以获取令牌
6. 将令牌存储到安全存储
7. 如果启用则进行跨应用访问（XAA）令牌交换

### 令牌刷新重试

瞬态错误重试刷新失败：
- `ServerError`、`TemporarilyUnavailableError`、`TooManyRequestsError`
- 瞬态错误最大重试次数
- `InvalidGrantError` 触发令牌失效（需要重新认证）

### 非标准 OAuth 服务器

某些服务器（特别是 Slack）返回 HTTP 200 和错误 JSON 体。自定义 fetch 包装器检测这一点并重写为 400，以便 SDK 的错误映射正常工作。非标准 `invalid_grant` 别名被规范化：
- `invalid_refresh_token`
- `expired_refresh_token`
- `token_expired`

### MCP 认证缓存

基于文件的缓存（`mcp-needs-auth-cache.json`）跟踪需要认证的服务器：
- TTL：15 分钟
- 记忆化读取防止批量连接期间的 N 次文件读取
- 序列化写入防止并发读-修改-写入竞争
- 成功认证时清除

### Claude.ai 代理认证

`createClaudeAiProxyFetch()` 为 claude.ai 代理连接包装 fetch：
- 附加 OAuth bearer 令牌
- 通过 `handleOAuth401Error` 在 401 上重试一次（强制刷新）
- 防止 memo 缓存中过时令牌导致所有连接器大量 401

---

## 官方 MCP 注册表

### 概述 (`officialRegistry.ts`)

在启动时火焰和忘记获取 Anthropic 的官方 MCP 服务器注册表：

```
GET https://api.anthropic.com/mcp-registry/v0/servers?version=latest&visibility=commercial
```

### 行为

1. 如果设置了 `CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC` 则跳过
2. 5 秒超时
3. 提取服务器远程 URL，对其进行规范化（剥离查询字符串、尾随斜杠）
4. 存储在 `Set<string>` 中以进行 O(1) 查找
5. `isOfficialMcpUrl(normalizedUrl)` 检查成员资格
6. 失败关闭：未定义的注册表 → `false`

### 用途

用于分析以在遥测事件中区分官方与第三方 MCP 服务器。

---

## 服务器配置

### 配置作用域 (`config.ts`)

MCP 服务器可以在多个作用域中配置，优先级顺序：

| 作用域 | 来源 | 文件/位置 |
|---|---|---|
| `enterprise` | 托管策略 | `{configHome}/managed-mcp.json` |
| `local` | 项目本地（内存中） | 当前项目配置 |
| `project` | `.mcp.json` 文件 | 从 CWD 到根的遍历 |
| `user` | 全局配置 | 全局 Claude 配置 |
| `dynamic` | 运行时注入 | 启动时传递 |
| `claudeai` | Claude.ai 连接器 | 从 API 获取 |
| `managed` | 企业托管 | 与 enterprise 相同 |

### 配置模式验证

所有配置针对 Zod 模式验证：
- `McpStdioServerConfigSchema` — command（必需，非空）、args、env
- `McpSSEServerConfigSchema` — url、headers、headersHelper、oauth
- `McpHTTPServerConfigSchema` — url、headers、headersHelper、oauth
- `McpWebSocketServerConfigSchema` — url、headers、headersHelper
- `McpSSEIDEServerConfigSchema` — url、ideName
- `McpWebSocketIDEServerConfigSchema` — url、ideName、authToken
- `McpSdkServerConfigSchema` — name
- `McpClaudeAIProxyServerConfigSchema` — url、id

### 环境变量扩展

服务器配置支持 `${VAR}` 和 `$VAR` 语法：
- 在加载时通过 `expandEnvVarsInString()` 扩展
- 缺失的变量报告为警告（不是错误）
- 应用于：command、args、env 值、URL、headers

### 企业策略

#### 允许/拒绝列表

企业管理员可以通过以下方式控制 MCP 服务器访问：
- `allowedMcpServers` — 基于名称、命令或 URL 的允许列表（支持通配符 URL 模式）
- `deniedMcpServers` — 基于名称、命令或 URL 的拒绝列表（始终优先）

策略设置可以通过 `allowManagedMcpServersOnly` 锁定为仅托管。

#### 独占控制

当存在企业 MCP 配置（`managed-mcp.json`）时：
- 所有其他作用域都被忽略
- 企业配置具有独占控制
- 用户不能添加自己的服务器

### 去重

#### 插件服务器去重

`dedupPluginMcpServers()` 防止来自插件的重复服务器：
- 基于命令数组（stdio）或 URL（远程）的签名
- 手动服务器优先于插件服务器
- 插件之间，先加载的优先
- 抑制的服务器为 UI 显示跟踪

#### Claude.ai 连接器去重

`dedupClaudeAiMcpServers()` 防止连接器/手动重复：
- 基于 URL/命令的基于内容的签名匹配
- 手动服务器优先于连接器
- 只有已启用的手动服务器计为去重目标

#### CCR 代理 URL 解包

`unwrapCcrProxyUrl()` 从 CCR 代理 URL 中提取原始供应商 URL 以进行签名匹配：
- 检测 `/v2/session_ingress/shttp/mcp/` 或 `/v2/ccr-sessions/` 路径标记
- 提取 `mcp_url` 查询参数

### 服务器启用/禁用

- `disabledMcpServers` — 要禁用的服务器名称列表
- `enabledMcpServers` — 明确的选择加入列表（覆盖禁用）
- 内置服务器（例如 Computer Use）默认为禁用，需要明确选择加入
- 保留名称：`claude-in-chrome`、计算机使用服务器名称

### 配置文件操作

`.mcp.json` 写入使用原子文件操作：
1. 写入临时文件（`{path}.tmp.{pid}.{timestamp}`）
2. `fsync` 刷新到磁盘
3. 原子重命名
4. 保留原始文件权限

---

## 连接管理

### MCPConnectionManager

React 上下文组件（`MCPConnectionManager.tsx`）提供：
- `reconnectMcpServer(serverName)` — 重新连接特定服务器
- `toggleMcpServer(serverName)` — 启用/禁用服务器

由管理以下内容的 `useManageMCPConnections()` 钩子支持：
- 初始连接批次（带可配置批次大小的并行）
- 失败时重新连接
- 服务器启用/禁用切换
- 动态配置更新

### 连接批处理

- 本地服务器：批次大小 3（`MCP_SERVER_CONNECTION_BATCH_SIZE`）
- 远程服务器：批次大小 20（`MCP_REMOTE_SERVER_CONNECTION_BATCH_SIZE`）
- 防止大量连接期间的资源耗尽

### 认证状态检查

`isMcpAuthCached()` 在尝试连接前检查基于文件的认证缓存：
- 15 分钟 TTL
- `needs-auth` 状态的服务器跳过连接尝试
- 成功认证或显式缓存清除时清除

---

## 工具结果存储

### 二进制内容持久化

大型二进制工具结果持久化到磁盘：
- 保存到 Claude 配置目录
- 在工具结果中通过路径引用
- 包含格式描述以供模型上下文
- 包含大小估计以用于截断决策

### 输出截断

`truncateMcpContentIfNeeded()` 在以下情况下应用截断：
- 内容大小超过配置的限制
- 为模型意识追加大型输出指令
- 文本内容带指示器截断

---

## 附加特性

### MCP 技能

来自 MCP 服务器的功能门控（`MCP_SKILLS`）技能获取：
- `fetchMcpSkillsForClient()` 加载与已连接服务器关联的技能
- 技能出现在 `/skills` 命令中，与 prompts 分开

### MCP 指令增量

功能门控（`MCP_INSTRUCTIONS_DELTA`）基于附件的 MCP 服务器指令传递：
- 将 Chrome 工具搜索指令作为客户端块携带
- 避免 Chrome 延迟连接时提示缓存破坏

### 计算机使用 MCP

功能门控（`CHICAGO_MCP`）计算机使用支持：
- 进程内服务器以避免子进程开销
- 通过包装器的本机模块调度
- 保留的服务器名称

### Chrome 中的 Claude MCP

- 通过 `@ant/claude-for-chrome-mcp` 的进程内服务器
- 用于 UI 显示的自定义工具渲染
- 保留的服务器名称：`claude-in-chrome`
