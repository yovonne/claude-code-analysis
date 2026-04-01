# Authentication Commands

## Purpose

Documents the CLI commands responsible for user authentication lifecycle: signing in (`/login`), signing out (`/logout`), and token refresh (`/oauth-refresh`). These commands manage Anthropic account credentials, OAuth flows, trusted device enrollment, and post-auth state synchronization.

---

## `/login`

### Purpose

Authenticate with an Anthropic account or switch between existing accounts. Displays "Sign in with your Anthropic account" for new users or "Switch Anthropic accounts" when already authenticated.

### Location

- `restored-src/src/commands/login/index.ts` — Command registration
- `restored-src/src/commands/login/login.tsx` — JSX implementation

### Command Registration

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

### Key Exports

#### Functions

- `call(onDone, context)`: Entry point. Renders the `<Login>` JSX component wrapped in a `Dialog`.
- `Login(props)`: React component that wraps `ConsoleOAuthFlow` inside a `Dialog` titled "Login". Provides cancel/confirm handlers.

### Implementation Details

#### Core Logic

The login flow uses the `ConsoleOAuthFlow` component to perform an OAuth-based authentication with the Anthropic console. After successful authentication, a comprehensive post-login refresh sequence executes:

1. **API Key Change Notification** — `context.onChangeAPIKey()` signals the key rotation.
2. **Signature Block Stripping** — `stripSignatureBlocks()` removes thinking/connector_text blocks bound to the old API key, preventing the new key from rejecting stale signatures.
3. **Cost State Reset** — `resetCostState()` clears accumulated cost tracking for the new account.
4. **Remote Settings Refresh** — `refreshRemoteManagedSettings()` and `refreshPolicyLimits()` fetch org-specific configuration (non-blocking).
5. **User Cache Reset** — `resetUserCache()` clears cached user data *before* GrowthBook refresh so feature flags pick up fresh credentials.
6. **GrowthBook Refresh** — `refreshGrowthBookAfterAuthChange()` reloads feature flags (e.g., claude.ai MCPs).
7. **Trusted Device Enrollment** — `clearTrustedDeviceToken()` removes stale tokens, then `enrollTrustedDevice()` enrolls the device for Remote Control (10-minute session window).
8. **Permission Killswitch Reset** — `resetBypassPermissionsCheck()` and `checkAndDisableBypassPermissionsIfNeeded()` re-evaluate permission gates with the new org context.
9. **Auto Mode Gate Reset** — When `TRANSCRIPT_CLASSIFIER` feature is active, `resetAutoModeGateCheck()` and `checkAndDisableAutoModeIfNeeded()` re-evaluate auto-mode eligibility.
10. **Auth Version Bump** — `authVersion` is incremented to trigger re-fetching of auth-dependent data in hooks (e.g., MCP servers).

#### UI Structure

```
Dialog(title="Login", color="permission")
  └── ConsoleOAuthFlow
        └── (OAuth browser flow)
```

The dialog includes a `ConfigurableShortcutHint` for the cancel action and a double-press-to-exit guard.

### Dependencies

#### Internal Dependencies

| Module | Purpose |
|--------|---------|
| `src/utils/auth.js` | `hasAnthropicApiKeyAuth()` — determines login vs. switch wording |
| `src/utils/envUtils.js` | `isEnvTruthy()` — checks `DISABLE_LOGIN_COMMAND` |
| `src/bootstrap/state.js` | `resetCostState()` — resets cost tracking |
| `src/bridge/trustedDevice.js` | `clearTrustedDeviceToken()`, `enrollTrustedDevice()` |
| `src/components/ConsoleOAuthFlow.js` | OAuth flow UI component |
| `src/components/design-system/Dialog.js` | Dialog wrapper |
| `src/services/analytics/growthbook.js` | `refreshGrowthBookAfterAuthChange()` |
| `src/services/policyLimits/index.js` | `refreshPolicyLimits()` |
| `src/services/remoteManagedSettings/index.js` | `refreshRemoteManagedSettings()` |
| `src/utils/permissions/bypassPermissionsKillswitch.js` | Permission gate management |
| `src/utils/user.js` | `resetUserCache()` |
| `src/utils/messages.js` | `stripSignatureBlocks()` |

### Configuration

| Environment Variable | Effect |
|---------------------|--------|
| `DISABLE_LOGIN_COMMAND` | When truthy, the `/login` command is disabled and hidden |

### Error Handling

- If the user cancels the OAuth flow, `onDone('Login interrupted')` is called with `success = false`.
- On success, `onDone('Login successful')` is called and the full post-login refresh executes.

---

## `/logout`

### Purpose

Sign out from the Anthropic account, removing all stored credentials, clearing auth-related caches, and optionally resetting onboarding state.

### Location

- `restored-src/src/commands/logout/index.ts` — Command registration
- `restored-src/src/commands/logout/logout.tsx` — JSX implementation

### Command Registration

```typescript
{
  type: 'local-jsx',
  name: 'logout',
  description: 'Sign out from your Anthropic account',
  isEnabled: () => !isEnvTruthy(process.env.DISABLE_LOGOUT_COMMAND),
  load: () => import('./logout.js'),
}
```

### Key Exports

#### Functions

- `call()`: Entry point. Calls `performLogout({ clearOnboarding: true })`, displays a success message, and triggers graceful shutdown.
- `performLogout({ clearOnboarding })`: Core logout logic. Flushes telemetry, removes API key, wipes secure storage, clears caches, and updates global config.
- `clearAuthRelatedCaches()`: Invalidates all auth-dependent memoized caches.

### Implementation Details

#### Core Logic

**`performLogout()`** executes the following sequence:

1. **Flush Telemetry** — `flushTelemetry()` is imported lazily (avoids ~1.1MB OpenTelemetry at startup) and called *before* clearing credentials to prevent org data leakage.
2. **Remove API Key** — `removeApiKey()` deletes the stored API key.
3. **Wipe Secure Storage** — `getSecureStorage().delete()` removes all securely stored data.
4. **Clear Auth Caches** — `clearAuthRelatedCaches()` invalidates all auth-dependent caches.
5. **Update Global Config** — `saveGlobalConfig()` resets `oauthAccount` to `undefined`. If `clearOnboarding` is true, also resets `hasCompletedOnboarding`, `subscriptionNoticeCount`, `hasAvailableSubscription`, and clears approved custom API key responses.

**`clearAuthRelatedCaches()`** clears:

- OAuth token cache (`getClaudeAIOAuthTokens.cache.clear()`)
- Trusted device token cache
- Betas caches
- Tool schema cache
- User data cache (`resetUserCache()`)
- GrowthBook feature flags (`refreshGrowthBookAfterAuthChange()`)
- Grove config cache (`getGroveNoticeConfig.cache.clear()`, `getGroveSettings.cache.clear()`)
- Remote managed settings cache (`clearRemoteManagedSettingsCache()`)
- Policy limits cache (`clearPolicyLimitsCache()`)

#### Shutdown

After logout, `gracefulShutdownSync(0, 'logout')` is scheduled via `setTimeout(200ms)` to cleanly terminate the process.

### Dependencies

#### Internal Dependencies

| Module | Purpose |
|--------|---------|
| `src/bridge/trustedDevice.js` | `clearTrustedDeviceTokenCache()` |
| `src/services/analytics/growthbook.js` | `refreshGrowthBookAfterAuthChange()` |
| `src/services/api/grove.js` | `getGroveNoticeConfig()`, `getGroveSettings()` |
| `src/services/policyLimits/index.js` | `clearPolicyLimitsCache()` |
| `src/services/remoteManagedSettings/index.js` | `clearRemoteManagedSettingsCache()` |
| `src/utils/auth.js` | `getClaudeAIOAuthTokens()`, `removeApiKey()` |
| `src/utils/betas.js` | `clearBetasCaches()` |
| `src/utils/config.js` | `saveGlobalConfig()` |
| `src/utils/gracefulShutdown.js` | `gracefulShutdownSync()` |
| `src/utils/secureStorage/index.js` | `getSecureStorage()` |
| `src/utils/toolSchemaCache.js` | `clearToolSchemaCache()` |
| `src/utils/user.js` | `resetUserCache()` |
| `src/utils/telemetry/instrumentation.js` | `flushTelemetry()` (lazy import) |

### Configuration

| Environment Variable | Effect |
|---------------------|--------|
| `DISABLE_LOGOUT_COMMAND` | When truthy, the `/logout` command is disabled and hidden |

### Error Handling

- Telemetry flush happens before credential deletion to prevent data leakage to the wrong org.
- The graceful shutdown uses a 200ms delay to allow the success message to render before exit.

---

## `/oauth-refresh`

### Purpose

Refresh OAuth tokens. Currently implemented as a disabled stub.

### Location

- `restored-src/src/commands/oauth-refresh/index.js`

### Command Registration

```javascript
{
  isEnabled: () => false,
  isHidden: true,
  name: 'stub',
}
```

### Implementation Details

This command is a **disabled stub** — `isEnabled` always returns `false` and `isHidden` is `true`, meaning it never appears in command listings and cannot be invoked. The actual OAuth token refresh logic is handled internally by the auth service and GrowthBook integration, not through a user-facing command.

### Notes

- The command exists as a placeholder in the commands directory but has no functional implementation.
- OAuth token lifecycle management is handled automatically by the `ConsoleOAuthFlow` component and the auth utilities (`getClaudeAIOAuthTokens`).

---

## Data Flow

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

## Related Modules

- [Command System](../01-core-modules/command-system.md)
- [Auth Service](../04-services/auth-service.md)
- [Auth Utils](../05-utils/auth.md)
