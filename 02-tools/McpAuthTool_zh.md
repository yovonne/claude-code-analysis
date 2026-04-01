# McpAuthTool

## 用途

McpAuthTool 是为已安装但尚未认证的 MCP 服务器动态创建的伪工具。它在未认证状态下替换服务器的真正工具，允许模型发现服务器存在并代表用户启动 OAuth 流程。调用时，它启动 OAuth 认证过程，返回授权 URL 以便用户在浏览器中打开，并在认证完成后自动重新连接服务器并使用真正的工具。

## 位置

- `restored-src/src/tools/McpAuthTool/McpAuthTool.ts` — 主要工具工厂和 OAuth 编排（216 行）
- `restored-src/src/services/mcp/auth.ts` — OAuth 流程实现、ClaudeAuthProvider、令牌管理（约 1700 行）
- `restored-src/src/services/mcp/client.ts` — 服务器重新连接、缓存管理、认证缓存（约 2800 行）
- `restored-src/src/services/mcp/types.ts` — 服务器配置 schema（259 行）
- `restored-src/src/services/mcp/mcpStringUtils.ts` — 工具名称构建实用工具（107 行）
- `restored-src/src/services/mcp/oauthPort.ts` — OAuth 重定向端口管理（79 行）
- `restored-src/src/services/mcp/xaa.ts` — 跨应用访问令牌交换
- `restored-src/src/services/mcp/xaaIdpLogin.ts` — IdP 登录和 OIDC 发现
- `restored-src/src/utils/secureStorage/` — 平台特定的 secure credential 存储
- `restored-src/src/utils/log.ts` — MCP 调试/错误日志

## 主要导出

### 来自 McpAuthTool.ts

| 导出 | 描述 |
|--------|-------------|
| `createMcpAuthTool(serverName, config)` | 创建认证伪工具的工厂函数 |
| `McpAuthOutput` | 输出类型：`{ status: 'auth_url' | 'unsupported' | 'error', message, authUrl? }` |

### 来自 auth.ts（OAuth 核心）

| 导出 | 描述 |
|--------|-------------|
| `performMCPOAuthFlow(serverName, config, onAuthorizationUrl, abortSignal, options)` | 主要 OAuth 流程编排器 |
| `ClaudeAuthProvider` | 实现 `OAuthClientProvider` 接口的 OAuth 客户端提供程序 |
| `getServerKey(serverName, serverConfig)` | 从名称 + 配置哈希生成唯一存储键 |
| `hasMcpDiscoveryButNoToken(serverName, config)` | 检查服务器是否被探测但没有凭证 |
| `revokeServerTokens(serverName, config, options)` | 在 OAuth 服务器上撤销令牌（RFC 7009） |
| `clearServerTokensFromLocalStorage(serverName, config)` | 清除本地令牌存储 |
| `wrapFetchWithStepUpDetection(baseFetch, provider)` | 用于 403 step-up 检测的 Fetch 包装器 |
| `normalizeOAuthErrorBody(response)` | 规范化非标准 OAuth 错误响应 |
| `fetchAuthServerMetadata(serverName, serverUrl, configuredMetadataUrl, fetchFn, resourceMetadataUrl)` | 发现 OAuth 服务器元数据 |
| `AuthenticationCancelledError` | 用户取消认证的错误 |

### 来自 client.ts（认证缓存）

| 导出 | 描述 |
|--------|-------------|
| `McpAuthError` | 工具调用期间认证失败的错误类 |
| `isMcpAuthCached(serverId)` | 检查 15 分钟认证缓存的 needs-auth 状态 |
| `setMcpAuthCacheEntry(serverId)` | 将 needs-auth 条目写入缓存 |
| `clearMcpAuthCache()` | 清除整个认证缓存文件 |
| `handleRemoteAuthFailure(name, serverRef, transportType)` | 处理连接期间的 401 |
| `createClaudeAiProxyFetch(innerFetch)` | 带 OAuth 令牌重试的 Fetch 包装器 |

## 输入/输出 Schema

### 输入 Schema

```typescript
// 空对象 — 不需要输入参数
{}
```

该工具不需要参数。认证通过使用空输入对象调用工具来触发。

### 输出 Schema

```typescript
{
  status: 'auth_url' | 'unsupported' | 'error',
  message: string,
  authUrl?: string,  // 当 status 为 'auth_url' 且 URL 可用时存在
}
```

| 状态 | 含义 |
|--------|-------------|
| `auth_url` | OAuth 流程成功启动；用户应打开 URL |
| `unsupported` | 服务器类型不支持程序化 OAuth（例如 claudeai-proxy） |
| `error` | OAuth 流程启动失败 |

## MCP 认证流程

### 何时创建 McpAuthTool

```
1. 服务器连接尝试
   - connectToServer() 尝试连接到 MCP 服务器
   - 服务器返回 HTTP 401 (Unauthorized)

2. 认证失败检测
   - SSE: SSEClientTransport 的 UnauthorizedError
   - HTTP: StreamableHTTPClientTransport 的 UnauthorizedError
   - claudeai-proxy: HTTP 401 状态码

3. 需要认证状态
   - 调用 handleRemoteAuthFailure()：
     a. 发出 tengu_mcp_server_needs_auth 分析事件
     b. 写入认证缓存（15 分钟 TTL）
     c. 返回 { name, type: 'needs-auth', config }

4. 认证工具创建
   - createMcpAuthTool(serverName, config) 创建伪工具
   - 工具名称：mcp__<server>__authenticate
   - 在认证完成前替换 appState 中的真正工具

5. 跳过条件（避免不必要的 401 探测）
   - isMcpAuthCached()：服务器在 15 分钟 needs-auth 缓存中
   - hasMcpDiscoveryButNoToken()：之前探测过，没有存储令牌
   - XAA 服务器排除（可以通过缓存的 id_token 静默重新认证）
```

### OAuth 流程执行

```
1. 工具调用
   - 模型使用 {} 调用 mcp__server__authenticate

2. 传输验证
   - claudeai-proxy → 不支持（需要手动 /mcp 认证）
   - stdio → 不支持（不支持从工具进行 OAuth）
   - sse/http → 继续 OAuth 流程

3. OAuth 初始化
   - 使用以下参数调用 performMCPOAuthFlow()：
     a. serverName、serverConfig
     b. onAuthorizationUrl 回调（捕获 URL）
     c. abortSignal 用于取消
     d. { skipBrowserOpen: true }（不自动打开浏览器）

4. 认证 URL 捕获
   - Promise 竞速：
     a. authUrlPromise：当 onAuthorizationUrl 触发时解析
     b. oauthPromise：当完整流程完成时解析
   - 立即返回 authUrl（非阻塞）

5. 后台完成
   - OAuth 流程在后台继续：
     a. 用户在浏览器中打开 URL
     b. 在提供商处授权
     c. 浏览器重定向到 localhost 回调
     d. ClaudeAuthProvider 接收授权码
     e. SDK 交换码为令牌
     f. 令牌保存到安全存储

6. 服务器重新连接
   - oauthPromise.then()：
     a. clearMcpAuthCache() — 移除 needs-auth 条目
     b. reconnectMcpServerImpl(serverName, config)：
        - clearKeychainCache() — 读取新鲜凭证
        - clearServerCache() — 清除备忘录缓存
        - connectToServer() — 使用令牌重新连接
        - fetchToolsForClient() — 发现真正工具
        - fetchCommandsForClient() — 发现命令
        - fetchResourcesForClient() — 发现资源
     c. setAppState()：
        - 在 mcp.clients 中替换客户端
        - 移除旧工具（基于前缀：mcp__<server>__*）
        - 添加新的真正工具
        - 更新资源
     d. 日志："OAuth 完成，使用 N 个工具重新连接"

7. 错误处理
   - oauthPromise.catch()：
     - 使用失败详情记录 MCP 错误
     - 认证工具保留（用户必须重试或运行 /mcp）
```

### XAA（跨应用访问）认证

```
当服务器配置中 oauth.xaa 为 true 时：

1. IdP 配置
   - getXaaIdpSettings() 读取 IdP 连接详情
   - IdP issuer、clientId、callbackPort 来自 settings.xaaIdp
   - 通过配置：claude mcp xaa setup --issuer <url> ...

2. IdP 登录
   - acquireIdpIdToken()：
     a. 检查缓存（getCachedIdpIdToken）
     b. 如果缓存 → 静默（无浏览器）
     c. 如果缺失/过期 → OIDC authorization_code+PKCE 流程
        - 在 IdP 一次浏览器弹出（跨所有 XAA 服务器共享）
        - id_token 按 issuer 缓存在 keychain 中

3. 令牌交换
   - performCrossAppAccess()：
     a. RFC 8693 令牌交换（无浏览器）
     b. RFC 7523 jwt-bearer grant
     c. IdP id_token → MCP 服务器访问/刷新令牌

4. 令牌存储
   - 保存到与正常 OAuth 相同的 keychain 槽
   - 相同的 ClaudeAuthProvider.tokens() 路径不变
   - discoveryState 为刷新/撤销持久化

5. 错误恢复
   - 带有 shouldClearIdToken 的 XaaTokenExchangeError：
     - 4xx / 无效 body → 清除缓存的 id_token
     - 5xx IdP 中断 → 保留缓存的 id_token
   - 如果清除，下次尝试进行新的 IdP 登录
```

## MCP 的 OAuth 处理

### ClaudeAuthProvider

`ClaudeAuthProvider` 类实现 MCP SDK 的 `OAuthClientProvider` 接口：

```typescript
class ClaudeAuthProvider implements OAuthClientProvider {
  // 必需接口方法：
  get redirectUrl(): string           // http://localhost:<port>/callback
  get clientMetadata(): OAuthClientMetadata  // 客户端注册信息
  clientInformation(): OAuthClientInformation | undefined  // 存储的客户端凭证
  saveClientInformation(info: OAuthClientInformationFull): void  // 保存 DCR 结果
  tokens(): OAuthTokens | undefined   // 读取存储的令牌
  saveTokens(tokens: OAuthTokens): void  // 保存令牌到安全存储
  redirectToAuthorization(url: string): void  // 触发浏览器重定向
  onInvalidAuthorizationCode(code: string): Promise<void>  // 处理错误代码

  // 自定义扩展：
  setMetadata(metadata: AuthorizationServerMetadata): void  // 存储发现的元数据
  markStepUpPending(scope: string): void  // 标记升级认证
  state(): Promise<string>  // 获取当前 OAuth 状态（用于 CSRF 验证）
}
```

### OAuth 发现流程

```
1. 元数据发现（fetchAuthServerMetadata）
   尝试顺序：

   a. 配置的元数据 URL（如果在服务器配置中设置）：
      - 必须是 https://
      - 直接获取到 URL
      - 解析为 OAuthMetadataSchema

   b. RFC 9728 发现（首选）：
      - 在 MCP 服务器上探测 /.well-known/oauth-protected-resource
      - 从响应中读取 authorization_servers[0]
      - 针对该授权服务器 URL 进行 RFC 8414 发现

   c. RFC 8414 回退（遗留）：
      - 仅当服务器 URL 有路径组件时
      - 探测 /.well-known/oauth-authorization-server/{path}
      - 覆盖共同托管 auth 元数据的遗留服务器

2. 动态客户端注册（DCR）
   - 如果没有配置 client_id：
     a. POST 到元数据中的 registration_endpoint
     b. 发送 client_metadata（redirect_uri、grant_types 等）
     c. 保存返回的 client_id 和 client_secret

3. 授权码流程（PKCE）
   a. 生成 code_verifier（随机字节）
   b. 计算 code_challenge = SHA256(code_verifier)
   c. 生成 state（随机 UUID）
   d. 构建授权 URL：
      - 来自元数据的 authorization_endpoint
      - response_type=code
      - client_id
      - redirect_uri
      - code_challenge + code_challenge_method=S256
      - state
      - scope（如果从 WWW-Authenticate 或元数据可用）
      - resource_metadata_url（如果可用）
   e. 打开浏览器到授权 URL
   f. 在 localhost:<port>/callback 等待回调

4. 令牌交换
   a. 从回调接收授权码
   b. 验证 state（防止 CSRF）
   c. POST 到 token_endpoint：
      - grant_type=authorization_code
      - code
      - redirect_uri
      - code_verifier
      - client_id（如果是机密还有 client_secret）
   d. 接收 access_token、refresh_token、expires_in、scope
   e. 保存令牌到安全存储

5. 令牌刷新
   - 当 access_token 过期时：
     a. POST 到 token_endpoint：
        - grant_type=refresh_token
        - refresh_token
        - client_id（如果是机密还有 client_secret）
     b. 接收新的 access_token（可能还有新的 refresh_token）
     c. 更新安全存储
```

### 升级认证

```
场景：服务器返回 403 insufficient_scope

1. 检测（wrapFetchWithStepUpDetection）
   - 拦截 403 响应
   - 解析 WWW-Authenticate 头
   - 提取 scope 参数：scope="read:admin"
   - 调用 provider.markStepUpPending(scope)

2. 升级触发
   - tokens() 在设置 stepUpPending 时省略 refresh_token
   - SDK 掉落到 PKCE 流程（不是刷新）
   - 用户使用提升的 scope 重新授权

3. 作用域持久化
   - 新的 scope 保存为 secure storage 中的 stepUpScope
   - 为下一次 performMCPOAuthFlow 调用缓存
   - 避免重新探测服务器的 scope 要求

4. 缓存的升级
   - 下一次 auth 流程：
     a. 从存储读取 cachedStepUpScope
     b. 从存储读取 cachedResourceMetadataUrl
     c. 传递给 WWWAuthenticateParams
     d. 跳过服务器探测（已经知道需要什么）
```

### 令牌撤销

```
revokeServerTokens(serverName, config, { preserveStepUpState })：

1. 读取存储的令牌
   - getSecureStorage().read().mcpOAuth[serverKey]
   - 提取 accessToken、refreshToken、clientId、clientSecret

2. 发现撤销端点
   - fetchAuthServerMetadata() 获取 OAuth 元数据
   - 从元数据读取 revocation_endpoint
   - 确定 auth 方法：
     - revocation_endpoint_auth_methods_supported（首选）
     - token_endpoint_auth_methods_supported（回退）
     - 默认：client_secret_basic

3. 首先撤销刷新令牌（RFC 7009）
   - revokeToken({ token: refreshToken, tokenTypeHint: 'refresh_token' })
   - RFC 7009 兼容：body 中的 client_id，无 Authorization 头
   - 401 时回退：使用 Bearer auth 重试（非兼容服务器）

4. 撤销访问令牌
   - revokeToken({ token: accessToken, tokenTypeHint: 'access_token' })
   - 可能已经被刷新令牌撤销失效

5. 清除本地存储
   - clearServerTokensFromLocalStorage()
   - 删除整个 mcpOAuth[serverKey] 条目

6. 保留升级状态（可选）
   - 如果 preserveStepUpState=true：
     - 保留 stepUpScope 和 discoveryState
     - 仅清除访问/刷新令牌
     - 下一次 auth 流程跳过服务器探测
```

### OAuth 错误规范化

```
normalizeOAuthErrorBody(response)：

问题：某些 OAuth 服务器（尤其是 Slack）在错误时返回 HTTP 200，
通过 JSON body 表示：{"error": "invalid_refresh_token"}

解决方案：
1. 检查响应是否为 2xx
2. 解析 JSON body
3. 如果匹配 OAuthErrorResponseSchema 但不匹配 OAuthTokensSchema：
   a. 规范化非标准错误代码：
      - invalid_refresh_token → invalid_grant
      - expired_refresh_token → invalid_grant
      - token_expired → invalid_grant
   b. 重写为 HTTP 400 响应
   c. SDK 错误映射正确应用（InvalidGrantError）

这确保刷新重试/失效逻辑正确处理错误。
```

## 令牌管理

### 安全存储

```
存储位置：平台特定的安全存储（keychain/credential manager）
存储键：getServerKey(serverName, serverConfig)
  - 格式："<serverName>|<configHash>"
  - 配置哈希：{ type, url, headers } 的 SHA256（前 16 个十六进制字符）
  - 防止具有相同名称的不同配置之间的凭证重用

存储数据结构：
{
  mcpOAuth: {
    "<serverKey>": {
      serverName: string,
      serverUrl: string,
      accessToken: string,
      refreshToken: string,
      expiresAt: number,           // 时间戳（毫秒）
      scope: string,
      clientId: string,
      clientSecret: string,
      stepUpScope?: string,        // 缓存的提升 scope
      discoveryState?: {           // OAuth 发现状态
        authorizationServerUrl: string,
        resourceMetadataUrl: string,
      },
    }
  }
}
```

### 认证缓存（需要认证）

```
缓存文件：mcp-needs-auth-cache.json
TTL：15 分钟（MCP_AUTH_CACHE_TTL_MS）

结构：
{
  "<serverId>": { timestamp: number }
}

操作：
- isMcpAuthCached(serverId)：检查条目是否存在且在 TTL 内
- setMcpAuthCacheEntry(serverId)：写入时间戳（通过 writeChain 序列化）
- clearMcpAuthCache()：删除文件和空读取缓存

写入序列化：
- writeChain：Promise 链防止并发读-修改-写竞争
- 同一批次中的多个 401 不会损坏缓存
- 写入时使读取缓存失效
```

### OAuth 回调服务器

```
端口选择（findAvailablePort）：
1. 尝试 MCP_OAUTH_CALLBACK_PORT 环境变量（如果设置）
2. 在范围内随机选择：
   - Windows：39152-49151（低于动态范围）
   - Unix：49152-65535（动态范围）
3. 回退：3118（如果随机选择失败）
4. 最多 100 次随机尝试

重定向 URI：http://localhost:<port>/callback
  - RFC 8252 Section 7.3：环回 URI 匹配任何端口

回调处理：
1. 在所选端口上创建 HTTP 服务器
2. 监听 GET /callback?code=...&state=...
3. 验证 state（防止 CSRF）
4. 处理错误：
   - error 参数 → 显示错误页面，拒绝 promise
   - state 不匹配 → 拒绝并显示"可能的 CSRF 攻击"
5. 成功时：
   - 显示"认证成功"页面
   - 使用授权码解析 promise
6. 清理：
   - server.close()
   - 移除事件监听器
   - 清除超时（默认 5 分钟）

手动回调 URL 粘贴：
- 用于 localhost 不可达的远程/基于浏览器的环境
- onWaitingForCallback 回调接受粘贴的 URL
- 解析 code 和 state，与 HTTP 回调相同验证
```

### 令牌刷新

```
自动刷新流程（ClaudeAuthProvider.tokens()）：

1. 检查存储的令牌
2. 如果 access_token 未过期 → 原样返回
3. 如果 access_token 已过期且有 refresh_token：
   a. 使用 refresh_token POST 到 token_endpoint
   b. 处理 OAuth 错误：
      - invalid_grant → 使令牌失效，触发重新认证
      - server_error/temporarily_unavailable → 重试（最多 3 次）
      - too_many_requests → 使用退避重试
   c. 保存新令牌
   d. 返回新的 access_token
4. 如果没有 refresh_token → 返回 undefined（触发重新认证）

瞬时错误的重试行为：
- ServerError (5xx)：最多重试 3 次
- TemporarilyUnavailableError：最多重试 3 次
- TooManyRequestsError：使用 Retry-After 头重试
- InvalidGrantError：不重试，立即使令牌失效
```

## 配置

### 服务器 OAuth 配置

```typescript
McpOAuthConfigSchema:
{
  clientId?: string,              // 预配置的客户端 ID
  callbackPort?: number,          // 固定 OAuth 回调端口
  authServerMetadataUrl?: string, // 预配置的元数据 URL（必须是 https://）
  xaa?: boolean,                  // 启用跨应用访问
}
```

### 环境变量

| 变量 | 用途 |
|----------|---------|
| `MCP_OAUTH_CALLBACK_PORT` | 固定 OAuth 重定向端口 |
| `CLAUDE_CODE_ENABLE_XAA` | 启用跨应用访问（必须是 "1"） |

### 敏感 OAuth 参数（从日志中删除）

| 参数 | 原因 |
|-----------|--------|
| `state` | CSRF 预防令牌 |
| `nonce` | 重放攻击预防 |
| `code_challenge` | PKCE 挑战 |
| `code_verifier` | PKCE 验证器（秘密） |
| `code` | 授权码 |

## 依赖

### 内部

| 模块 | 用途 |
|--------|---------|
| `services/mcp/auth.ts` | OAuth 流程、ClaudeAuthProvider、令牌管理 |
| `services/mcp/client.ts` | 服务器重新连接、认证缓存 |
| `services/mcp/types.ts` | 服务器配置 schema |
| `services/mcp/mcpStringUtils.ts` | 工具名称构建 |
| `services/mcp/oauthPort.ts` | 端口选择、重定向 URI |
| `services/mcp/xaa.ts` | 跨应用访问令牌交换 |
| `services/mcp/xaaIdpLogin.ts` | IdP 登录、OIDC 发现 |
| `utils/secureStorage/` | 平台特定的凭证存储 |
| `utils/browser.ts` | 浏览器打开实用工具 |
| `utils/errors.ts` | 错误消息提取 |
| `utils/log.ts` | MCP 调试/错误日志 |
| `utils/lockfile.ts` | 并发访问的文件锁定 |
| `utils/sleep.ts` | 重试的睡眠实用工具 |
| `utils/slowOperations.ts` | JSON 解析/字符串化 |
| `utils/envUtils.ts` | 配置主目录解析 |
| `utils/platform.ts` | 平台检测 |
| `constants/oauth.ts` | OAuth 配置常量 |
| `services/analytics/` | 事件日志 |

### 外部

| 包 | 用途 |
|---------|---------|
| `@modelcontextprotocol/sdk` | OAuth 客户端、auth 类型、schema |
| `axios` | 用于令牌撤销的 HTTP 客户端 |
| `crypto` | 哈希生成、随机字节、UUID |
| `http` | 本地回调服务器 |
| `fs/promises` | 安全存储文件操作 |
| `xss` | 错误页面的 HTML 清理 |
| `url` | URL 解析 |
| `path` | 路径连接 |
| `lodash-es/reject` | 用于工具替换的数组过滤 |
| `zod/v4` | 输入 schema 验证 |

## 数据流

```
Model Calls Auth Tool
    │
    ▼
┌─────────────────────────────────────────────────────────────────┐
│ AUTH TOOL CALL                                                   │
│                                                                 │
│  1. Transport check                                             │
│     - claudeai-proxy → unsupported                              │
│     - stdio → unsupported                                       │
│     - sse/http → proceed                                        │
│                                                                 │
│  2. OAuth flow start                                            │
│     - performMCPOAuthFlow()                                     │
│     - skipBrowserOpen: true                                     │
│     - Capture auth URL via callback                             │
│                                                                 │
│  3. Return auth URL to model                                    │
│     - status: 'auth_url'                                        │
│     - message: instructions for user                            │
│     - authUrl: URL to open                                      │
└─────────────────────────────────────────────────────────────────┘
    │
    ▼ (background)
┌─────────────────────────────────────────────────────────────────┐
│ BACKGROUND OAUTH COMPLETION                                      │
│                                                                 │
│  1. User opens URL in browser                                   │
│  2. Authorizes on provider                                      │
│  3. Browser redirects to localhost callback                     │
│  4. ClaudeAuthProvider receives code                            │
│  5. SDK exchanges code for tokens                               │
│  6. Tokens saved to secure storage                              │
│                                                                 │
│  7. clearMcpAuthCache()                                         │
│  8. reconnectMcpServerImpl():                                   │
│     - Clear keychain cache                                      │
│     - Clear server caches                                       │
│     - Reconnect with tokens                                     │
│     - Fetch tools, commands, resources                          │
│                                                                 │
│  9. setAppState():                                              │
│     - Replace client                                            │
│     - Remove auth tool (prefix match)                           │
│     - Add real tools                                            │
│     - Update resources                                          │
└─────────────────────────────────────────────────────────────────┘
    │
    ▼
Real MCP Tools Available (auth tool automatically removed)
```
