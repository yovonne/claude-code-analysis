# 认证工具

**文件：** `restored-src/src/utils/auth.ts`

## 目的

处理 Claude Code 的所有认证流程，包括 API 密钥管理、OAuth 令牌处理和云提供商（AWS/GCP）凭证管理。

## 主要函数

### API 密钥管理

- `getAnthropicApiKey()` - 返回 API 密钥字符串或 null
- `getAnthropicApiKeyWithSource()` - 返回密钥及其来源（`ANTHROPIC_API_KEY`、`apiKeyHelper`、`/login managed key` 或 `none`）
- `hasAnthropicApiKeyAuth()` - 检查 API 密钥认证是否可用
- `saveApiKey(apiKey)` - 将 API 密钥保存到 macOS 密钥链或配置文件
- `removeApiKey()` - 删除存储的 API 密钥
- `isCustomApiKeyApproved(apiKey)` - 检查密钥是否在已批准列表中

### API 密钥助手

- `getConfiguredApiKeyHelper()` - 从设置中获取配置的助手命令
- `getApiKeyFromApiKeyHelper()` - 带缓存和 TTL 的异步执行助手
- `getApiKeyFromApiKeyHelperCached()` - 同步缓存读取（非阻塞）
- `clearApiKeyHelperCache()` - 清除助手缓存
- `calculateApiKeyHelperTTL()` - 从环境变量计算缓存 TTL（默认 5 分钟）

### OAuth 令牌管理

- `getClaudeAIOAuthTokens()` - 从安全存储返回缓存的 OAuth 令牌
- `getClaudeAIOAuthTokensAsync()` - 从存储异步读取的异步版本
- `saveOAuthTokensIfNeeded(tokens)` - 将 OAuth 令牌保存到安全存储
- `clearOAuthTokenCache()` - 清除所有 OAuth 令牌缓存
- `handleOAuth401Error(failedAccessToken)` - 通过强制令牌刷新处理 401 错误
- `checkAndRefreshOAuthTokenIfNeeded()` - 检查过期并在需要时刷新

### 云提供商认证

- `refreshAndGetAwsCredentials()` - 刷新并缓存 AWS STS 凭证（1小时 TTL）
- `clearAwsCredentialsCache()` - 清除 AWS 凭证缓存
- `refreshGcpCredentialsIfNeeded()` - 在过期时刷新 GCP 凭证（1小时 TTL）
- `clearGcpCredentialsCache()` - 清除 GCP 凭证缓存
- `checkGcpCredentialsValid()` - 用 5 秒超时验证 GCP 凭证
- `prefetchAwsCredentialsAndBedRockInfoIfSafe()` - 在信任检查后预取 AWS 凭证
- `prefetchGcpCredentialsIfSafe()` - 在信任检查后预取 GCP 凭证

### 认证上下文检测

- `isAnthropicAuthEnabled()` - 确定是否启用 Anthropic 认证（与第三方提供商对比）
- `isManagedOAuthContext()` - 检测 CCR/claude-desktop 管理的 OAuth 上下文
- `getAuthTokenSource()` - 返回当前认证令牌的来源

## 安全注意事项

- 来自项目/本地设置的 API 密钥助手命令需要接受信任对话框后才能执行
- 来自项目设置的 AWS/GCP 凭证导出命令需要接受信任对话框
- 使用十六进制编码存储在 macOS 密钥链中的 API 密钥以避免 shell 转义问题
- OAuth 令牌存储在安全存储中并进行正确的作用域验证
- 并发 401 处理程序去重防止多次同时刷新尝试
- 通过文件 mtime 监控进行跨进程陈旧检测

## 缓存策略

- API 密钥助手：带可配置 TTL（默认 5 分钟）的记忆化，基于纪元的失效
- AWS 凭证：带 1 小时 TTL 的记忆化
- GCP 凭证：带 1 小时 TTL 的记忆化
- OAuth 令牌：记忆化，在磁盘更改或 401 错误时失效
- 密钥链读取：带 30 秒 TTL 的记忆化

## 依赖

- `authFileDescriptor.ts` - 基于文件描述符的令牌读取
- `authPortable.ts` - 便携式认证工具
- `config.ts` - 已批准密钥的全局配置
- `secureStorage/` - 平台特定的安全存储
- `settings/settings.ts` - 助手命令的设置
- `aws.ts` / `awsAuthStatusManager.ts` - AWS 特定认证
