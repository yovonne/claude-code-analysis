# 认证命令

## 目的

记录负责用户认证生命周期的 CLI 命令：登录（`/login`）、登出（`/logout`）和令牌刷新（`/oauth-refresh`）。这些命令管理 Anthropic 账户凭证、OAuth 流程、受信任设备注册以及认证后状态同步。

---

## `/login`

### 目的

使用 Anthropic 账户进行认证或在现有账户之间切换。为新用户显示"Sign in with your Anthropic account"，为已认证用户显示"Switch Anthropic accounts"。

### 位置

- `restored-src/src/commands/login/index.ts` — 命令注册
- `restored-src/src/commands/login/login.tsx` — JSX 实现

### 命令注册

```typescript
{
  type: 'local-jsx',
  name: 'login',
  description: hasAnthropicApiKeyAuth()
    ? 'Switch Anthropic accounts'
    : 'Sign in with your Anthropic account',
  isEnabled: () => !isEnvTruthy(process.env.DISABLE_LOGIN_COMMAND),
  load: () => import('./login.js'),
}
```

### 关键导出

#### 函数

- `call(onDone, context)`：入口点。渲染包裹在 `Dialog` 中的 `<Login>` JSX 组件，提供取消/确认处理程序。
- `Login(props)`：React 组件，在标题为"Login"的 `Dialog` 中包装 `ConsoleOAuthFlow`。提供取消/确认处理程序。

### 实现细节

#### 核心逻辑

登录流程使用 `ConsoleOAuthFlow` 组件执行基于 OAuth 的 Anthropic 控制台认证。成功认证后，执行全面的登录后刷新序列：

1. **API 密钥更改通知** — `context.onChangeAPIKey()` 信号密钥轮换。
2. **签名块剥离** — `stripSignatureBlocks()` 移除绑定到旧 API 密钥的 thinking/connector_text 块，防止新密钥拒绝陈旧签名。
3. **成本状态重置** — `resetCostState()` 清除新账户的累积成本跟踪。
4. **远程设置刷新** — `refreshRemoteManagedSettings()` 和 `refreshPolicyLimits()` 获取组织特定配置（非阻塞）。
5. **用户缓存重置** — `resetUserCache()` 在 GrowthBook 刷新之前清除缓存的用户数据，以便功能标志拾取新凭证。
6. **GrowthBook 刷新** — `refreshGrowthBookAfterAuthChange()` 重新加载功能标志（例如 claude.ai MCP）。
7. **受信任设备注册** — `clearTrustedDeviceToken()` 移除陈旧令牌，然后 `enrollTrustedDevice()` 注册设备以进行远程控制（10 分钟会话窗口）。
8. **权限killswitch 重置** — `resetBypassPermissionsCheck()` 和 `checkAndDisableBypassPermissionsIfNeeded()` 使用新组织上下文重新评估权限门。
9. **自动模式门重置** — 当 `TRANSCRIPT_CLASSIFIER` 功能活跃时，`resetAutoModeGateCheck()` 和 `checkAndDisableAutoModeIfNeeded()` 重新评估自动模式资格。
10. **Auth 版本递增** — `authVersion` 递增以触发钩子中依赖认证的数据重新获取（例如 MCP 服务器）。

#### UI 结构

```
Dialog(title="Login", color="permission")
  └── ConsoleOAuthFlow
        └── (OAuth browser flow)
```

对话框包含 `ConfigurableShortcutHint` 用于取消操作和双按退出保护。

### 依赖

#### 内部依赖

| 模块 | 目的 |
|--------|---------|
| `src/utils/auth.js` | `hasAnthropicApiKeyAuth()` — 确定登录 vs 切换措辞 |
| `src/utils/envUtils.js` | `isEnvTruthy()` — 检查 `DISABLE_LOGIN_COMMAND` |
| `src/bootstrap/state.js` | `resetCostState()` — 重置成本跟踪 |
| `src/bridge/trustedDevice.js` | `clearTrustedDeviceToken()`、`enrollTrustedDevice()` |
| `src/components/ConsoleOAuthFlow.js` | OAuth 流程 UI 组件 |
| `src/components/design-system/Dialog.js` | Dialog 包装器 |
| `src/services/analytics/growthbook.js` | `refreshGrowthBookAfterAuthChange()` |
| `src/services/policyLimits/index.js` | `refreshPolicyLimits()` |
| `src/services/remoteManagedSettings/index.js` | `refreshRemoteManagedSettings()` |
| `src/utils/permissions/bypassPermissionsKillswitch.js` | 权限门管理 |
| `src/utils/user.js` | `resetUserCache()` |
| `src/utils/messages.js` | `stripSignatureBlocks()` |

### 配置

| 环境变量 | 效果 |
|---------------------|--------|
| `DISABLE_LOGIN_COMMAND` | 为真时，`/login` 命令被禁用并隐藏 |

### 错误处理

- 如果用户取消 OAuth 流程，使用 `success = false` 调用 `onDone('Login interrupted')`。
- 成功时，调用 `onDone('Login successful')` 并执行完整的登录后刷新。

---

## `/logout`

### 目的

从 Anthropic 账户登出，删除所有存储的凭证，清除认证相关缓存，并可选择重置 onboarding 状态。

### 位置

- `restored-src/src/commands/logout/index.ts` — 命令注册
- `restored-src/src/commands/logout/logout.tsx` — JSX 实现

### 命令注册

```typescript
{
  type: 'local-jsx',
  name: 'logout',
  description: 'Sign out from your Anthropic account',
  isEnabled: () => !isEnvTruthy(process.env.DISABLE_LOGOUT_COMMAND),
  load: () => import('./logout.js'),
}
```

### 关键导出

#### 函数

- `call()`：入口点。调用 `performLogout({ clearOnboarding: true })`，显示成功消息，并触发优雅关闭。
- `performLogout({ clearOnboarding })`：核心登出逻辑。刷新遥测，删除 API 密钥，擦除安全存储，清除缓存，并更新全局配置。
- `clearAuthRelatedCaches()`：使所有认证依赖的记忆化缓存失效。

### 实现细节

#### 核心逻辑

**`performLogout()`** 执行以下序列：

1. **刷新遥测** — `flushTelemetry()` 被延迟导入（避免启动时约 1.1MB OpenTelemetry）并在清除凭证之前调用，以防止向错误组织泄露数据。
2. **删除 API 密钥** — `removeApiKey()` 删除存储的 API 密钥。
3. **擦除安全存储** — `getSecureStorage().delete()` 删除所有安全存储的数据。
4. **清除认证缓存** — `clearAuthRelatedCaches()` 使所有认证依赖缓存失效。
5. **更新全局配置** — `saveGlobalConfig()` 将 `oauthAccount` 重置为 `undefined`。如果 `clearOnboarding` 为 true，还重置 `hasCompletedOnboarding`、`subscriptionNoticeCount`、`hasAvailableSubscription`，并清除批准的自定义 API 密钥响应。

**`clearAuthRelatedCaches()`** 清除：

- OAuth 令牌缓存（`getClaudeAIOAuthTokens.cache.clear()`）
- 受信任设备令牌缓存
- Beta 缓存
- 工具模式缓存
- 用户数据缓存（`resetUserCache()`）
- GrowthBook 功能标志（`refreshGrowthBookAfterAuthChange()`）
- Grove 配置缓存（`getGroveNoticeConfig.cache.clear()`、`getGroveSettings.cache.clear()`）
- 远程托管设置缓存（`clearRemoteManagedSettingsCache()`）
- 策略限制缓存（`clearPolicyLimitsCache()`）

#### 关闭

登出后，通过 `setTimeout(200ms)` 调度 `gracefulShutdownSync(0, 'logout')` 以干净地终止进程。

### 依赖

#### 内部依赖

| 模块 | 目的 |
|--------|---------|
| `src/bridge/trustedDevice.js` | `clearTrustedDeviceTokenCache()` |
| `src/services/analytics/growthbook.js` | `refreshGrowthBookAfterAuthChange()` |
| `src/services/api/grove.js` | `getGroveNoticeConfig()`、`getGroveSettings()` |
| `src/services/policyLimits/index.js` | `clearPolicyLimitsCache()` |
| `src/services/remoteManagedSettings/index.js` | `clearRemoteManagedSettingsCache()` |
| `src/utils/auth.js` | `getClaudeAIOAuthTokens()`、`removeApiKey()` |
| `src/utils/betas.js` | `clearBetasCaches()` |
| `src/utils/config.js` | `saveGlobalConfig()` |
| `src/utils/gracefulShutdown.js` | `gracefulShutdownSync()` |
| `src/utils/secureStorage/index.js` | `getSecureStorage()` |
| `src/utils/toolSchemaCache.js` | `clearToolSchemaCache()` |
| `src/utils/user.js` | `resetUserCache()` |
| `src/utils/telemetry/instrumentation.js` | `flushTelemetry()`（延迟导入） |

### 配置

| 环境变量 | 效果 |
|---------------------|--------|
| `DISABLE_LOGOUT_COMMAND` | 为真时，`/logout` 命令被禁用并隐藏 |

### 错误处理

- 遥测刷新在删除凭证之前发生，以防止数据泄漏到错误的组织。
- 优雅关闭使用 200ms 延迟以允许成功消息在退出前渲染。

---

## `/oauth-refresh`

### 目的

刷新 OAuth 令牌。当前已实现为禁用的存根。

### 位置

- `restored-src/src/commands/oauth-refresh/index.js`

### 命令注册

```javascript
{
  isEnabled: () => false,
  isHidden: true,
  name: 'stub',
}
```

### 实现细节

此命令是一个**禁用存根** — `isEnabled` 始终返回 `false` 且 `isHidden` 为 `true`，意味着它从不出现在命令列表中且无法调用。实际的 OAuth 令牌刷新逻辑由 auth 服务和 GrowthBook 集成在内部处理，而不是通过用户面对的命令。

### 说明

- 该命令作为占位符存在于命令目录中，但没有功能实现。
- OAuth 令牌生命周期管理由 `ConsoleOAuthFlow` 组件和 auth 工具（`getClaudeAIOAuthTokens`）自动处理。

---

## 数据流

```
/login
  │
  ├── ConsoleOAuthFlow (browser-based OAuth)
  │     │
  │     ▼
  │   Success → onChangeAPIKey()
  │     │
  │     ├── stripSignatureBlocks()      ← Remove stale signatures
  │     ├── resetCostState()            ← Reset cost tracking
  │     ├── refreshRemoteManagedSettings() ← Org config
  │     ├── refreshPolicyLimits()       ← Policy limits
  │     ├── resetUserCache()            ← Clear user data
  │     ├── refreshGrowthBookAfterAuthChange() ← Feature flags
  │     ├── clearTrustedDeviceToken()   ← Clear old token
  │     ├── enrollTrustedDevice()       ← Enroll for Remote Control
  │     ├── resetBypassPermissionsCheck() ← Re-evaluate permissions
  │     └── authVersion++               ← Trigger hook re-fetches
  │
  └── onDone('Login successful')

/logout
  │
  ├── flushTelemetry()                  ← Before credential deletion
  ├── removeApiKey()                    ← Delete API key
  ├── secureStorage.delete()            ← Wipe all secure data
  ├── clearAuthRelatedCaches()          ← Invalidate all caches
  │     ├── OAuth token cache
  │     ├── Trusted device token cache
  │     ├── Betas, tool schema, user caches
  │     ├── GrowthBook refresh
  │     ├── Grove config cache
  │     ├── Remote settings cache
  │     └── Policy limits cache
  ├── saveGlobalConfig()                ← Reset oauthAccount, onboarding
  └── gracefulShutdownSync()            ← Clean exit

/oauth-refresh
  │
  └── (disabled stub — no functional implementation)
```

## 相关模块

- [Command System](../01-core-modules/command-system.md)
- [Auth Service](../04-services/auth-service.md)
- [Auth Utils](../05-utils/auth.md)
