# 远程托管设置服务

## 目的

管理为企业客户获取、缓存和验证远程托管设置。使用基于校验和的验证来最小化网络流量，并在失败时提供优雅降级。设置从服务器推送并作为策略层应用到设置合并管道。

## 位置

`restored-src/src/services/remoteManagedSettings/`

### 关键文件

| 文件 | 描述 |
|------|-------------|
| `index.ts` | 主服务 — 获取、缓存、轮询、安全检查集成 |
| `syncCache.ts` | 资格检查（auth-touching）和缓存重置包装器 |
| `syncCacheState.ts` | 叶子状态模块 — 会话缓存、资格镜像、同步文件加载 |
| `securityCheck.tsx` | 危险设置更改的阻止对话框 |
| `types.ts` | 远程设置 API 响应的 Zod 模式和类型 |

## 关键导出

### 函数 (index.ts)
- `loadRemoteManagedSettings()`：在 CLI 初始化期间加载设置，开始后台轮询
- `refreshRemoteManagedSettings()`：异步刷新设置（用于 auth 状态更改）
- `isEligibleForRemoteManagedSettings()`：资格检查的公共 API
- `waitForRemoteManagedSettingsToLoad()`：等待初始加载完成
- `clearRemoteManagedSettingsCache()`：清除所有缓存（会话、持久化、停止轮询）
- `startBackgroundPolling()`：开始每小时后台轮询
- `stopBackgroundPolling()`：停止后台轮询
- `initializeRemoteManagedSettingsLoadingPromise()`：提前初始化加载承诺
- `computeChecksumFromSettings(settings)`：从设置计算 SHA-256 校验和（为测试导出）

### 函数 (syncCache.ts)
- `isRemoteManagedSettingsEligible()`：资格检查（缓存，auth-touching）
- `resetSyncCache()`：清除资格缓存和叶子状态

### 函数 (syncCacheState.ts)
- `getRemoteManagedSettingsSyncFromCache()`：同步缓存读取（会话 → 文件）
- `setSessionCache(value)`：更新会话缓存
- `setEligibility(v)`：在叶子模块中镜像资格结果
- `resetSyncCache()`：清除会话缓存和资格
- `getSettingsPath()`：remote-settings.json 缓存文件的路径

### 函数 (securityCheck.tsx)
- `checkManagedSettingsSecurity(cachedSettings, newSettings)`：如果危险设置更改则显示阻止对话框
- `handleSecurityCheckResult(result)`：如果拒绝则退出，否则继续

### 类型 (types.ts)
- `RemoteManagedSettingsResponse`：API 响应（uuid、checksum、settings）
- `RemoteManagedSettingsFetchResult`：获取结果，带 success、settings（null = 304）、checksum、error、skipRetry

### 常量
- `SETTINGS_TIMEOUT_MS`：10 秒
- `DEFAULT_MAX_RETRIES`：5
- `POLLING_INTERVAL_MS`：1 小时
- `LOADING_PROMISE_TIMEOUT_MS`：30 秒

## 依赖

### 内部依赖
- `auth` — API 密钥和 OAuth 令牌检索
- `model/providers` — API 提供商检测
- `settings/types` — 设置模式验证
- `settings/changeDetector` — 热重载通知
- `syncCacheState` — 叶子状态模块（避免循环依赖）
- `securityCheck` — 危险设置对话框

### 外部依赖
- `axios` — HTTP 客户端
- `crypto` — SHA-256 校验和计算
- `fs/promises` — 异步文件 I/O
- `react` — 安全对话框渲染
- `ink` — 终端 UI 渲染

## 实现细节

### 资格
- **控制台用户（API 密钥）**：如果有实际密钥，所有人都有资格
- **OAuth 用户（Claude.ai）**：Enterprise/C4E 和 Team 订阅者
- **外部注入的令牌**（CCD、CCR、Agent SDK、CI）：有资格 — API 为不合资格的 org 返回空
- **第三方提供商**：没有资格
- **自定义基本 URL 用户**：没有资格
- **Cowork（local-agent 入口点）**：没有资格 — 服务器管理的设置不适用

### 获取逻辑

1. 检查资格 — 如果没有资格则返回 null
2. 从文件加载缓存设置（syncCacheState）
3. 从缓存设置计算校验和（排序键，SHA-256）
4. 使用 If-None-Match 头从 API 获取（ETag 缓存）
5. 处理响应：
   - 304：缓存版本仍然有效
   - 204/404：不存在设置，删除缓存文件
   - 200：使用 SettingsSchema 验证，检查安全性，保存
6. 失败时：如果有可用则使用陈旧缓存，否则失败开放

### 安全检查
- 当新设置包含与缓存设置不同的危险设置时
- 在交互模式下显示阻止对话框
- 用户可以接受（应用设置）或拒绝（保留缓存，如果是首次运行则关闭）
- 在非交互模式下跳过

### 后台轮询
- 通过 `setInterval` 每小时轮询（unref'd 以不阻止退出）
- 比较之前和新的设置，更改时触发热重载
- 调用 `settingsChangeDetector.notifyChange('policySettings')`

### 缓存架构（避免循环依赖）

模块拆分以打破循环依赖：
- `syncCacheState.ts`：叶子模块 — 无 auth 导入，仅 path/env/file/json
- `syncCache.ts`：包含 `isRemoteManagedSettingsEligible()`（auth-touching 部分）
- `index.ts`：主服务，从两者导入

资格在叶子中是三态的：undefined（未确定）、false（不合格）、true（继续）。计算一次并通过 `setEligibility()` 镜像。

### 校验和计算
- 递归排序所有键以匹配 Python 的 `json.dumps(sort_keys=True)`
- 使用紧凑分隔符（无空格）以匹配 Python 的 `separators=(",", ":")`
- 格式：`sha256:<hex>`

## 数据流

```
CLI 启动
    ↓
initializeRemoteManagedSettingsLoadingPromise()
    ↓
loadRemoteManagedSettings()
    ↓
getRemoteManagedSettingsSyncFromCache() — 缓存优先快速路径
    ↓
fetchAndLoadRemoteManagedSettings()
    ↓
fetchWithRetry() — 带 ETag 的 API 调用，最多 5 次重试
    ↓
checkManagedSettingsSecurity() — 危险更改时对话框
    ↓
saveSettings() → setSessionCache() → notifyChange('policySettings')
    ↓
startBackgroundPolling() — 每小时刷新
```

## 集成点

- **设置管道**：缓存设置合并为 `policySettings` 层
- **CLI 初始化**：`loadRemoteManagedSettings()` 在启动期间调用
- **Auth 更改**：`refreshRemoteManagedSettings()` 在登录/注销时调用
- **热重载**：`settingsChangeDetector.notifyChange('policySettings')` 触发重新合并
- **安全对话框**：危险设置更改的阻止 UI

## 配置

- API 端点：`/api/claude_code/settings`
- 缓存文件：`~/.claude/remote-settings.json`（模式 0o600）
- 轮询间隔：1 小时
- 获取超时：10 秒
- 最大重试次数：5，带指数退避

## 错误处理

- **失败开放**：API 失败继续而不使用远程设置
- **陈旧缓存**：获取失败时，使用缓存设置（如果有）
- **Auth 错误**：跳过重试（401/403 不会成功）
- **空设置**：204/404 删除缓存文件以防止陈旧设置
- **验证错误**：无效设置结构被拒绝，视为获取失败
- **安全拒绝**：用户可以拒绝危险设置，保留缓存版本

## 测试

- 未暴露 `_reset*ForTesting` 助手 — 测试使用 `clearRemoteManagedSettingsCache()`
- `computeChecksumFromSettings()` 为测试服务器兼容性导出

## 相关模块

- [策略限制](./policy-limits-service.md) — 类似模式（ETag、轮询、失败开放）
- [设置同步](./settings-sync-service.md) — 用户设置同步（不同系统）

## 备注

- 设置在初始宽容解析后使用完整的 `SettingsSchema` 验证
- 加载承诺有 30 秒超时以防止死锁
- 缓存优先启动：缓存设置立即应用，获取在后台运行
- gh-23085：首次资格确定时重置设置缓存可防止中毒的合并缓存
- Cowork (local-agent) 明确排除 — MDM/基于文件的托管设置仍然适用
