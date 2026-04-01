# OAuth 服务

## 目的

OAuth 服务为 Anthropic 服务实现 OAuth 2.0 授权代码流程与 PKCE，用于认证 Claude Code CLI 用户。它处理完整的认证生命周期：基于浏览器和手动认证流程、令牌交换、令牌刷新、配置文件获取、安全令牌存储以及跨进程令牌同步。它还管理 API 密钥存储和多提供商认证（Anthropic OAuth、API 密钥、AWS、GCP）。

## 位置

- `restored-src/src/services/oauth/index.ts` — OAuthService 类，主要 auth 流程编排
- `restored-src/src/services/oauth/client.ts` — 用于令牌交换、刷新、配置文件、角色、API 密钥创建的 HTTP 客户端
- `restored-src/src/services/oauth/crypto.ts` — PKCE 代码验证器/挑战和状态生成
- `restored-src/src/services/oauth/auth-code-listener.ts` — 用于 OAuth 重定向捕获的本地 HTTP 服务器
- `restored-src/src/services/oauth/getOauthProfile.ts` — 通过 API 密钥或 OAuth 令牌获取配置文件
- `restored-src/src/utils/auth.ts` — Auth 工具：API 密钥管理、OAuth 令牌生命周期、提供商检测
- `restored-src/src/utils/secureStorage/index.ts` — 安全存储工厂
- `restored-src/src/utils/secureStorage/macOsKeychainStorage.ts` — macOS Keychain 实现
- `restored-src/src/utils/secureStorage/macOsKeychainHelpers.ts` — 共享 keychain 助手和缓存状态
- `restored-src/src/utils/secureStorage/keychainPrefetch.ts` — 用于启动优化的并行 keychain 读取预取
- `restored-src/src/utils/secureStorage/fallbackStorage.ts` — 主/辅助存储回退包装器
- `restored-src/src/utils/secureStorage/plainTextStorage.ts` — 用于非 macOS 平台的纯文本文件回退

## 关键导出

### OAuth 服务 (`services/oauth/index.ts`)

#### 类
- `OAuthService`：具有 PKCE 支持的主要 OAuth 2.0 流程编排器

#### 方法
- `startOAuthFlow(authURLHandler, options)`：启动完整 OAuth 流程；返回 `OAuthTokens`
- `handleManualAuthCodeInput(params)`：处理用户粘贴的认证码用于非浏览器环境
- `cleanup()`：关闭认证码监听器并清除资源

### OAuth 客户端 (`services/oauth/client.ts`)

#### 函数
- `buildAuthUrl(options)`：使用 PKCE 参数构建授权 URL
- `exchangeCodeForTokens(code, state, verifier, port, isManual, expiresIn)`：将认证码交换为令牌
- `refreshOAuthToken(refreshToken, options)`：刷新访问令牌；如需要则获取配置文件
- `fetchProfileInfo(accessToken)`：获取用户配置文件（订阅类型、速率限制层级、账单信息）
- `fetchAndStoreUserRoles(accessToken)`：获取并存储组织/工作区角色
- `createAndStoreApiKey(accessToken)`：通过 OAuth 创建 API 密钥并保存
- `isOAuthTokenExpired(expiresAt)`：检查令牌是否过期（带 5 分钟缓冲区）
- `getOrganizationUUID()`：从配置或配置文件端点获取组织 UUID
- `populateOAuthAccountInfoIfNeeded()`：从环境变量或配置文件填充 OAuth 账户信息
- `storeOAuthAccountInfo(info)`：将账户信息保存到全局配置
- `shouldUseClaudeAIAuth(scopes)`：检查是否存在 Claude.ai 认证范围
- `parseScopes(scopeString)`：将空格分隔的范围字符串解析为数组

### Crypto (`services/oauth/crypto.ts`)

#### 函数
- `generateCodeVerifier()`：生成加密随机 PKCE 代码验证器（32 字节，base64url）
- `generateCodeChallenge(verifier)`：从验证器生成 SHA-256 代码挑战
- `generateState()`：生成随机 CSRF 保护状态参数

### 认证码监听器 (`services/oauth/auth-code-listener.ts`)

#### 类
- `AuthCodeListener`：捕获 OAuth 重定向码的临时 localhost HTTP 服务器

#### 方法
- `start(port?)`：在 OS 分配的端口上开始监听
- `waitForAuthorization(state, onReady)`：等待认证码重定向
- `handleSuccessRedirect(scopes, customHandler?)`：将浏览器发送到成功页面
- `handleErrorRedirect()`：将浏览器发送到错误页面
- `close()`：清理服务器和监听器

### Auth 工具 (`utils/auth.ts`)

#### API 密钥管理
- `getAnthropicApiKey()`：从任何来源获取 API 密钥
- `getAnthropicApiKeyWithSource(options)`：获取带来源标识的 API 密钥
- `saveApiKey(apiKey)`：保存 API 密钥到 keychain (macOS) 或配置
- `removeApiKey()`：从所有存储位置移除 API 密钥
- `getApiKeyFromApiKeyHelper(isNonInteractive)`：执行 apiKeyHelper 命令并缓存
- `isCustomApiKeyApproved(apiKey)`：检查自定义 API 密钥是否被批准

#### OAuth 令牌管理
- `getClaudeAIOAuthTokens()`：从环境、文件描述符或安全存储获取 OAuth 令牌（记忆化）
- `getClaudeAIOAuthTokensAsync()`：异步版本，避免阻塞 keychain 读取
- `saveOAuthTokensIfNeeded(tokens)`：将 OAuth 令牌保存到安全存储
- `checkAndRefreshOAuthTokenIfNeeded(retryCount, force)`：自动刷新过期令牌，带 lockfile 去重
- `handleOAuth401Error(failedAccessToken)`：通过强制刷新处理服务器端 401
- `clearOAuthTokenCache()`：清除所有 OAuth 令牌缓存
- `isClaudeAISubscriber()`：检查用户是否是 Claude.ai 订阅者
- `hasProfileScope()`：检查令牌是否有 `user:profile` 范围

#### 提供商检测
- `isAnthropicAuthEnabled()`：检查是否支持直接 Anthropic 认证
- `getAuthTokenSource()`：识别认证令牌从哪里获取
- `isUsing3PServices()`：检查是否使用 Bedrock、Vertex 或 Foundry
- `is1PApiCustomer()`：检查用户是否是直接 API 客户（不是订阅者或 3P）

#### 订阅管理
- `getSubscriptionType()`：获取订阅类型（max、pro、enterprise、team 或 null）
- `getSubscriptionName()`：获取人类可读的订阅名称
- `getRateLimitTier()`：从 OAuth 令牌获取速率限制层级
- `isMaxSubscriber()`、`isProSubscriber()`、`isTeamSubscriber()`、`isEnterpriseSubscriber()`：类型守卫

#### 云提供商认证
- `refreshAndGetAwsCredentials()`：刷新 AWS 认证并获取 STS 凭证（记忆化，1h TTL）
- `refreshGcpCredentialsIfNeeded()`：如需要则刷新 GCP 凭证（记忆化，1h TTL）
- `checkGcpCredentialsValid()`：探测 GCP 凭证有效性

#### 组织验证
- `validateForceLoginOrg()`：验证 OAuth 令牌属于所需组织（托管设置）

### 安全存储 (`utils/secureStorage/`)

#### 工厂 (`index.ts`)
- `getSecureStorage()`：返回平台适当的存储（macOS keychain 带纯文本回退，或仅纯文本）

#### macOS Keychain 存储 (`macOsKeychainStorage.ts`)
- `macOsKeychainStorage`：使用 `security` CLI 的 SecureStorage 实现
  - `read()`：同步读取，带 30s TTL 缓存和陈旧-而-错误
  - `readAsync()`：异步读取，带去重和生成跟踪
  - `update(data)`：通过 `security -i`（stdin）写入以隐藏有效载荷免受进程监控
  - `delete()`：移除 keychain 条目
- `isMacOsKeychainLocked()`：检查 keychain 是否被锁定（退出代码 36），为进程生命周期缓存

#### Keychain 助手 (`macOsKeychainHelpers.ts`)
- `getMacOsKeychainStorageServiceName(serviceSuffix)`：从配置目录哈希构建稳定的服务名称
- `getUsername()`：获取当前 OS 用户名
- `clearKeychainCache()`：使缓存无效，增加生成，清除进行中的 promise
- `primeKeychainCacheFromPrefetch(stdout)`：从预取结果填充缓存

#### Keychain 预取 (`keychainPrefetch.ts`)
- `startKeychainPrefetch()`：在 main.tsx 顶层触发并行 `security` 子进程
- `ensureKeychainPrefetchCompleted()`：在 preAction 中等待预取完成
- `getLegacyApiKeyPrefetchResult()`：获取用于同步读取器绕过的旧 API 密钥预取结果
- `clearLegacyApiKeyPrefetch()`：在缓存无效时清除预取结果

#### 回退存储 (`fallbackStorage.ts`)
- `createFallbackStorage(primary, secondary)`：创建首先尝试主要，次回退到辅助的存储

#### 纯文本存储 (`plainTextStorage.ts`)
- `plainTextStorage`：使用 `.credentials.json` 文件的 SecureStorage 实现（chmod 0o600）

## 依赖

### 内部依赖
- `constants/oauth.js` — OAuth URL、客户端 ID、范围、beta 头
- `utils/config.js` — 全局配置读写、账户信息、信任对话框状态
- `utils/browser.js` — 浏览器打开工具
- `utils/http.js` — 认证头生成
- `utils/user.js` — 用户数据和设备 ID
- `utils/debug.js` — ant 构建的调试日志
- `utils/log.js` — 错误日志
- `utils/envUtils.js` — 配置目录路径、环境检测
- `utils/errors.js` — 错误工具
- `utils/betas.js` — Beta 标志缓存
- `utils/lockfile.js` — 用于令牌刷新去重的跨进程 lockfile
- `utils/sleep.js` — 睡眠工具
- `utils/authFileDescriptor.js` — 基于文件描述符的令牌读取
- `utils/authPortable.js` — 跨平台 API 密钥规范化
- `utils/aws.js` — AWS STS 调用者身份验证
- `utils/awsAuthStatusManager.js` — 云提供商认证状态跟踪
- `utils/settings/settings.js` — 从多个来源加载设置
- `utils/model/model.js` — 模型名称工具
- `utils/model/providers.js` — API 提供商检测
- `services/analytics/index.js` — OAuth 事件的事件日志
- `bootstrap/state.js` — 会话状态（非交互式、第三方认证偏好）

### 外部依赖
- `axios` — 用于 OAuth 令牌交换、配置文件和 API 调用的 HTTP 客户端
- `execa` / `execaSync` — 用于 `security` CLI 和助手命令的进程生成
- `child_process` — 用于 keychain 预取的原生 Node.js 子进程
- `crypto` — 用于 PKCE（SHA-256、随机字节）和 keychain 十六进制编码的 Node.js crypto
- `fs` / `fs/promises` — 用于纯文本存储和凭证文件的文件操作
- `path` / `os` — 路径和 OS 工具
- `chalk` — 用于 auth 错误的终端输出格式化
- `lodash-es/memoize` — 用于缓存的记忆化

## 实现细节

### OAuth 流程实现

OAuth 服务实现**带 PKCE 的授权代码流程**（RFC 7636），支持两个并行授权路径：

#### 自动流程（基于浏览器）

1. **PKCE 生成**：`generateCodeVerifier()` 创建 32 字节随机验证器；`generateCodeChallenge()` 派生 SHA-256 挑战
2. **状态生成**：`generateState()` 创建 CSRF 保护状态参数
3. **本地服务器**：`AuthCodeListener` 在 OS 分配的 localhost 端口上启动 HTTP 服务器
4. **Auth URL 构建**：`buildAuthUrl()` 构建带有以下内容的授权 URL：
   - `client_id`、`response_type=code`、`code_challenge`、`code_challenge_method=S256`
   - `redirect_uri=http://localhost:{port}/callback`
   - `scope`（推理仅为 `user:inference` 或完整的 `ALL_OAUTH_SCOPES`）
   - 可选：`orgUUID`、`login_hint`、`login_method`
5. **浏览器启动**：`openBrowser()` 打开 auth URL
6. **重定向捕获**：本地服务器在 `/callback?code=...&state=...` 接收重定向
7. **状态验证**：将接收到的状态与预期状态进行比较（CSRF 保护）
8. **令牌交换**：`exchangeCodeForTokens()` 使用 auth 代码和代码验证器 POST 到令牌端点
9. **配置文件获取**：`fetchProfileInfo()` 检索订阅类型和速率限制层级
10. **成功重定向**：浏览器重定向到适当的成功页面（Claude.ai 或 Console）

#### 手动流程（非浏览器环境）

1. 与自动流程相同的 PKCE/状态生成
2. Auth URL 使用指向手动重定向 URL 的 `redirect_uri` 构建
3. 向用户显示 URL 以供手动复制
4. 用户通过 `handleManualAuthCodeInput()` 粘贴 auth 代码
5. 令牌交换继续相同

#### 流程选项

- `loginWithClaudeAi`：路由到 Claude.ai 授权 URL vs Console 授权 URL
- `inferenceOnly`：仅请求 `user:inference` 范围（长期令牌，无需完整配置文件访问）
- `orgUUID`：预选组织
- `loginHint`：预填充登录表单上的电子邮件
- `loginMethod`：请求特定登录方法（SSO、魔法链接、Google）
- `skipBrowserOpen`：调用者处理 URL 显示（由 SDK 控制协议使用）
- `expiresIn`：自定义令牌过期

### 令牌管理

#### 令牌结构 (`OAuthTokens`)

```typescript
{
  accessToken: string
  refreshToken: string | null
  expiresAt: number | null
  scopes: string[]
  subscriptionType: 'max' | 'pro' | 'enterprise' | 'team' | null
  rateLimitTier: string | null
  profile?: OAuthProfileResponse
  tokenAccount?: { uuid, emailAddress, organizationUuid }
}
```

#### 令牌存储

令牌存储在**安全存储**中，采用平台适当的后端：

| 平台 | 主要存储 | 回退 |
|----------|----------------|----------|
| macOS | Keychain（`security` CLI） | 纯文本 `.credentials.json` |
| Linux/Windows | 纯文本 `.credentials.json` | 无 |

存储路径是 `~/.claude/.credentials.json`（或 `CLAUDE_CONFIG_DIR` 覆盖）。在 macOS 上，keychain 服务名称是 `Claude Code-credentials`，带有可选的配置目录哈希后缀。

#### 令牌获取优先级

`getClaudeAIOAuthTokens()` 按顺序检查来源：

1. **Bare 模式检查**：在 `--bare` 模式中立即返回 `null`
2. **环境变量**：`CLAUDE_CODE_OAUTH_TOKEN`（仅推理令牌）
3. **文件描述符**：`CLAUDE_CODE_OAUTH_TOKEN_FILE_DESCRIPTOR` 或 CCR OAuth 令牌文件
4. **安全存储**：Keychain (macOS) 或 `.credentials.json`（所有平台）

结果被**记忆化**以避免重复昂贵的 keychain 读取。

### 认证状态

#### 认证来源检测

`getAuthTokenSource()` 识别认证来自哪里：

| 来源 | 描述 |
|--------|----------|
| `apiKeyHelper` | 在设置中配置的外部命令 |
| `ANTHROPIC_AUTH_TOKEN` | 直接 bearer 令牌环境变量 |
| `CLAUDE_CODE_OAUTH_TOKEN` | 来自环境变量的 OAuth 令牌 |
| `CLAUDE_CODE_OAUTH_TOKEN_FILE_DESCRIPTOR` | 来自继承 FD 的 OAuth 令牌 |
| `CCR_OAUTH_TOKEN_FILE` | 来自 CCR 磁盘回退的 OAuth 令牌 |
| `claude.ai` | 来自安全存储的 OAuth 令牌 |
| `none` | 未配置认证 |

#### Anthropic 认证启用

`isAnthropicAuthEnabled()` 在以下情况下返回 `false`：
- Bare 模式（`--bare`）
- 没有 OAuth 令牌占位符的 SSH 远程
- 第三方提供商（Bedrock、Vertex、Foundry）
- 配置了外部 API 密钥或认证令牌（除非在托管 OAuth 上下文中）

托管 OAuth 上下文（CCR 远程、Claude Desktop）防止回退到用户的本地 API 密钥设置。

### 会话处理

#### 令牌刷新

`checkAndRefreshOAuthTokenIfNeeded()` 实现稳健的刷新机制：

1. **缓存无效**：检查 `.credentials.json` 文件是否已被其他进程修改
2. **过期检查**：使用 5 分钟缓冲区（`isOAuthTokenExpired()`）抢先刷新
3. **异步重新读取**：清除记忆化缓存并从安全存储重新读取（另一个进程可能已刷新）
4. **lockfile 去重**：获取文件锁以防止跨进程并发刷新
   - 锁竞争时带抖动的重试（最多 5 次重试，1-2s 退避）
   - 获取锁后再次检查过期（另一个进程可能已刷新）
5. **范围处理**：为 Claude.ai 订阅者省略范围以允许刷新时范围扩展
6. **令牌保存**：将刷新后的令牌存储到安全存储
7. **缓存清除**：刷新后清除所有缓存

#### 401 错误处理

`handleOAuth401Error()` 处理服务器端令牌过期：

1. **去重**：具有相同失败令牌的并发调用被去重
2. **缓存清除**：强制从安全存储重新读取
3. **令牌比较**：如果 keychain 有不同令牌，另一个标签已刷新 — 使用它
4. **强制刷新**：如果相同令牌，强制刷新，绕过本地过期检查

#### 跨进程陈旧

多个 Claude Code 实例可能同时运行。系统通过以下方式处理：

- **文件修改时间跟踪**：`invalidateOAuthCacheIfDiskChanged()` 比较 `.credentials.json` 的 mtime
- **Lockfile 同步**：防止跨进程并发令牌刷新
- **异步重新读取**：`getClaudeAIOAuthTokensAsync()` 避免阻塞同步记忆化缓存
- **401 处理程序去重**：`pending401Handlers` Map 防止重复刷新尝试

### Keychain/安全存储集成

#### macOS Keychain 架构

Keychain 存储使用 `security` CLI 工具，具有多项优化：

1. **服务命名**：`Claude Code-credentials[-{configDirHash}]` — 每个配置目录唯一
2. **十六进制编码**：有效载荷被十六进制编码以避免 shell 转义问题
3. **Stdin 模式**：使用 `security -i`（交互模式），因此进程监控器只看到 `security -i`，看不到有效载荷（INC-3028）
4. **Argv 回退**：当有效载荷超过 4032 字节 stdin 行限制（4096 - 64 缓冲）时回退到 argv
5. **30 秒 TTL 缓存**：限制跨进程陈旧性，而不强制在每次读取时生成阻塞子进程
6. **陈旧-而-错误**：如果读取失败但存在缓存值，则提供陈旧值而不是返回 null
7. **生成跟踪**：`readAsync()` 在生成之前捕获生成； Skip 如果存在更新生成则写入缓存（防止陈旧子进程结果覆盖新 `update()` 数据）
8. **飞行中去重**：并发的 `readAsync()` 调用共享单个子进程

#### Keychain 预取优化

在 `main.tsx` 顶层，两个 `security` 子进程并行触发：
1. OAuth 凭证条目（`Claude Code-credentials`）
2. 旧版 API 密钥条目（`Claude Code`）

这些与约 65ms 的模块导入并行运行。`ensureKeychainPrefetchCompleted()` 在 preAction 中与 MDM 设置加载一起等待 — 由于子进程在导入评估期间完成，几乎是免费的。然后同步 `read()` 和 `getApiKeyFromConfigOrMacOSKeychain()` 命中其缓存而不是生成新子进程。

#### 回退存储策略

`createFallbackStorage(primary, secondary)` 实现弹性存储模式：

- **读取**：首先尝试主要；次回退到辅助
- **更新**：写入主要；如果主要失败则写入辅助
  - 如果主要写入失败但辅助成功，则删除陈旧的主要条目（防止用新鲜数据掩盖）
  - 如果更新前主要为空，则删除辅助（迁移清理）
- **删除**：从主要和辅助删除

### 令牌刷新流程

```
checkAndRefreshOAuthTokenIfNeeded()
  │
  ├─ invalidateOAuthCacheIfDiskChanged()
  │    └─ stat('.credentials.json') → mtime changed? → clearOAuthTokenCache()
  │
  ├─ getClaudeAIOAuthTokens() (memoized)
  │    └─ Not expired? → return false (no refresh needed)
  │
  ├─ getClaudeAIOAuthTokens.cache.clear() + clearKeychainCache()
  ├─ getClaudeAIOAuthTokensAsync() (async re-read)
  │    └─ Not expired? → return false (another process refreshed)
  │
  ├─ lockfile.lock(claudeDir)
  │    └─ ELOCKED? → retry with jitter (max 5)
  │
  ├─ Double-check expiration after lock
  │    └─ Not expired? → return false (race resolved)
  │
  ├─ refreshOAuthToken(refreshToken)
  │    ├─ POST to token endpoint with grant_type=refresh_token
  │    ├─ Parse response (access_token, refresh_token, expires_in, scope)
  │    └─ fetchProfileInfo(accessToken) (if profile fields not cached)
  │
  ├─ saveOAuthTokensIfNeeded(refreshedTokens)
  │    └─ secureStorage.update({ claudeAiOauth: {...} })
  │
  └─ clearOAuthTokenCache() + clearKeychainCache()
```

### 跨应用认证 (XAA)

系统支持跨多个 Claude Code 入口点的认证：

#### Claude Code Remote (CCR)

- 启动器通过文件描述符生成带有 OAuth 令牌的 CLI
- `CLAUDE_CODE_OAUTH_TOKEN` env 变量设置为占位符 iff 本地侧是订阅者
- 远程的本地设置（apiKeyHelper、ANTHROPIC_API_KEY）不得覆盖 — 会导致与认证注入代理的头不匹配
- 无法继承管道 FD 的 CCR 子进程使用磁盘回退文件

#### Claude Desktop (CCD)

- `CLAUDE_CODE_ENTRYPOINT === 'claude-desktop'` 标识托管会话
- 防止回退到用户的终端 CLI 设置（apiKeyHelper、ANTHROPIC_API_KEY）
- OAuth 令牌由 Claude Desktop 管理，而非 CLI

#### SDK / Cowork

- 环境变量直接提供账户信息：`CLAUDE_CODE_ACCOUNT_UUID`、`CLAUDE_CODE_USER_EMAIL`、`CLAUDE_CODE_ORGANIZATION_UUID`
- `skipBrowserOpen` 选项让 SDK 调用者处理 URL 显示
- `CLAUDE_CODE_OAUTH_TOKEN_FILE_DESCRIPTOR` 用于通过 FD 传递令牌

#### SSH Remote

- `ANTHROPIC_UNIX_SOCKET` 表示带有本地认证注入代理的 SSH 隧道
- `CLAUDE_CODE_OAUTH_TOKEN` 中的占位符令牌表示订阅者状态
- 本地侧在建立会话前运行组织验证；远程跳过冗余验证

### API 密钥管理

#### 密钥来源（优先级顺序）

1. `ANTHROPIC_API_KEY` 环境变量（如果在配置中批准）
2. 文件描述符（`CLAUDE_CODE_API_KEY_FILE_DESCRIPTOR`）
3. `apiKeyHelper` 命令（外部脚本，带 TTL 缓存）
4. macOS Keychain（`/login managed key`）
5. 配置文件（`primaryApiKey`）

#### API 密钥助手

`apiKeyHelper` 是用户配置的外部命令，返回 API 密钥：

- **异步执行，同步缓存**：`getApiKeyFromApiKeyHelper()` 异步运行命令，但将结果缓存，可配置 TTL（默认 5 分钟，环境：`CLAUDE_CODE_API_KEY_HELPER_TTL_MS`）
- **基于纪元的失效**：设置更改会增加纪元，孤立进行中的执行
- **陈旧-而-刷新**：陈旧缓存立即返回，而后台刷新运行
- **信任门控**：如果助手来自项目/本地设置，必须首先建立工作区信任
- **错误处理**：失败缓存哨兵值（`' '`）以防止回退到 OAuth

#### 密钥存储 (macOS)

- 通过带 stdin 模式的 `security add-generic-password` 存储在 keychain（进程监控保护）
- 也添加到配置中的 `customApiKeyResponses.approved` 列表以进行批准跟踪
- 保存后清除记忆化缓存

## 配置

### OAuth 常量

| 常量 | 用途 |
|----------|----------|
| `CLIENT_ID` | OAuth 客户端 ID |
| `CONSOLE_AUTHORIZE_URL` | Console 授权端点 |
| `CLAUDE_AI_AUTHORIZE_URL` | Claude.ai 授权端点 |
| `TOKEN_URL` | 令牌交换端点 |
| `MANUAL_REDIRECT_URL` | 手动 auth 流程的重定向 URL |
| `CONSOLE_SUCCESS_URL` | Console 认证后成功页面 |
| `CLAUDEAI_SUCCESS_URL` | Claude.ai 认证后成功页面 |
| `ROLES_URL` | 用户角色端点 |
| `API_KEY_URL` | API 密钥创建端点 |
| `BASE_API_URL` | 用于配置文件获取的基本 API URL |
| `OAUTH_FILE_SUFFIX` | keychain 服务命名的后缀 |
| `ALL_OAUTH_SCOPES` | 完整 OAuth 范围集 |
| `CLAUDE_AI_OAUTH_SCOPES` | Claude.ai 特定范围 |
| `CLAUDE_AI_INFERENCE_SCOPE` | 仅推理范围（`user:inference`） |
| `CLAUDE_AI_PROFILE_SCOPE` | 配置文件范围（`user:profile`） |
| `OAUTH_BETA_HEADER` | OAuth 配置文件端点的 Beta 头 |

### 环境变量

| 变量 | 用途 |
|----------|----------|
| `CLAUDE_CODE_OAUTH_TOKEN` | 强制设置 OAuth 访问令牌（仅推理） |
| `CLAUDE_CODE_OAUTH_TOKEN_FILE_DESCRIPTOR` | 通过文件描述符的 OAuth 令牌 |
| `CLAUDE_CODE_ACCOUNT_UUID` | 账户 UUID（SDK/cowork） |
| `CLAUDE_CODE_USER_EMAIL` | 用户电子邮件（SDK/cowork） |
| `CLAUDE_CODE_ORGANIZATION_UUID` | 组织 UUID（SDK/cowork） |
| `CLAUDE_CODE_API_KEY_HELPER_TTL_MS` | API 密钥助手缓存 TTL 覆盖 |
| `CLAUDE_CODE_ENTRYPOINT` | 入口点标识符（`claude-desktop`、`local-agent` 等） |
| `CLAUDE_CODE_REMOTE` | 远程会话标志 |
| `ANTHROPIC_UNIX_SOCKET` | SSH 认证代理套接字路径 |
| `CLAUDE_CONFIG_DIR` | 自定义配置目录（影响 keychain 服务名称） |

## 错误处理

- **令牌交换失败**：抛出带描述性消息的错误（401 = 无效代码，其他 = 状态 + 状态文本）
- **令牌刷新失败**：记录到分析与错误消息和响应体；重新抛出给调用者
- **401 不带认证重试**：1P 事件日志记录在 401 上不带认证头重试
- **Keychain 失败**：优雅回退到纯文本存储；陈旧-而-错误提供缓存值
- **Lockfile 竞争**：带抖动重试最多 5 次；如果超过最大重试则返回 false
- **API 密钥助手失败**：错误消息打印到控制台；哨兵值缓存以防止 OAuth 回退
- **配置文件获取失败**：静默记录；保留现有缓存配置文件值（不要用 null 覆盖有效订阅类型）
- **组织验证失败**：返回描述性错误消息而不是抛出；当设置 `forceLoginOrgUUID` 时失败关闭

## 相关模块

- [分析服务](./analytics-service.md) — OAuth 事件的事件日志
- [Auth 命令](../03-commands/auth-commands.md) — CLI auth 命令（login、logout）
- [配置工具](../05-utils/config.md) — 全局配置和账户信息存储
- [安全存储](../05-utils/secure-storage.md) — 凭证存储实现
- [引导状态](../08-internals/bootstrap.md) — 会话状态和信任管理
