# 策略限制服务

## 目的

从 API 获取组织级策略限制并用于禁用 CLI 功能。遵循与远程托管设置相同的模式：失败开放、ETag 缓存、后台轮询和重试逻辑。

## 位置

`restored-src/src/services/policyLimits/`

### 关键文件

| 文件 | 描述 |
|------|-------------|
| `index.ts` | 主服务 — 获取、缓存、轮询和策略评估 |
| `types.ts` | 策略限制 API 响应的 Zod 模式和类型 |

## 关键导出

### 函数 (index.ts)
- `loadPolicyLimits()`：在 CLI 初始化期间加载策略限制，启动后台轮询
- `refreshPolicyLimits()`：异步刷新策略限制（用于 auth 状态更改）
- `isPolicyAllowed(policy)`：检查特定策略是否允许（同步）
- `isPolicyLimitsEligible()`：检查当前用户是否有资格获得策略限制
- `waitForPolicyLimitsToLoad()`：等待初始加载完成
- `clearPolicyLimitsCache()`：清除所有缓存（会话、持久化、停止轮询）
- `startBackgroundPolling()`：开始每小时后台轮询
- `stopBackgroundPolling()`：停止后台轮询
- `initializePolicyLimitsLoadingPromise()`：提前初始化加载承诺（从 init.ts 调用）
- `_resetPolicyLimitsForTesting()`：测试专用同步重置模块状态

### 类型 (types.ts)
- `PolicyLimitsResponse`：带 `restrictions` 记录的 API 响应（策略键 → { allowed: boolean }）
- `PolicyLimitsFetchResult`：获取结果，带 success、restrictions（null = 304 缓存有效）、etag、error、skipRetry

### 常量
- `CACHE_FILENAME`：'policy-limits.json'
- `FETCH_TIMEOUT_MS`：10 秒
- `DEFAULT_MAX_RETRIES`：5
- `POLLING_INTERVAL_MS`：1 小时
- `LOADING_PROMISE_TIMEOUT_MS`：30 秒
- `ESSENTIAL_TRAFFIC_DENY_ON_MISS`：包含在 HIPAA 模式下失败关闭的策略集（当前 `allow_product_feedback`）

## 依赖

### 内部依赖
- `auth` — API 密钥和 OAuth 令牌检索
- `model/providers` — API 提供商检测
- `privacyLevel` — 仅基本流量模式检查
- `api/withRetry` — 重试延迟计算
- `cleanupRegistry` — 关闭清理注册

### 外部依赖
- `axios` — HTTP 客户端
- `crypto` — SHA-256 校验和计算
- `fs/promises` — 异步文件 I/O
- `fs` — 同步文件读取（用于同步策略检查）

## 实现细节

### 资格
- **控制台用户（API 密钥）**：如果有实际密钥，所有人都有资格
- **OAuth 用户（Claude.ai）**：仅 Team 和 Enterprise/C4E 订阅者
- **第三方提供商**：没有资格
- **自定义基本 URL 用户**：没有资格

### 获取逻辑

1. 检查资格 — 如果没有资格则返回 null
2. 从文件加载缓存的限制
3. 从缓存限制计算校验和（排序键，SHA-256）
4. 使用 If-None-Match 头从 API 获取（ETag 缓存）
5. 处理响应：
   - 304：缓存版本仍然有效
   - 404：不存在限制，删除缓存文件
   - 200：应用新限制，保存到缓存
6. 失败时：如果有可用则使用陈旧缓存，否则失败开放（返回 null）

### 策略评估（同步）
- `isPolicyAllowed(policy)` 从会话缓存或文件检查限制
- **失败开放**：未知策略或不可用缓存返回 `true`
- **例外**：当启用仅基本流量模式（HIPAA）且缓存不可用时，`ESSENTIAL_TRAFFIC_DENY_ON_MISS` 中的策略失败关闭
- 只有被阻止的策略包含在响应中 — 不存在的键意味着允许

### 后台轮询
- 通过 `setInterval` 每小时轮询（unref'd 以不阻止退出）
- 比较之前和新的缓存状态，记录更改
- 注册关闭注册表

### 校验和计算
- 递归排序所有键以实现一致哈希
- 使用 `jsonStringify`（紧凑，无空格）以匹配服务器端 Python `json.dumps(sort_keys=True, separators=(",", ":"))`
- 格式：`sha256:<hex>`

## 数据流

```
CLI 启动
    ↓
initializePolicyLimitsLoadingPromise() — 提前设置承诺
    ↓
loadPolicyLimits()
    ↓
loadCachedRestrictions() — 从文件同步读取
    ↓
computeChecksum() — 从缓存数据计算
    ↓
fetchWithRetry() — 带 ETag 的 API 调用，最多 5 次重试
    ↓
应用新限制 / 使用陈旧缓存 / 失败开放
    ↓
startBackgroundPolling() — 每小时刷新
    ↓
isPolicyAllowed() — 从会话缓存同步检查
```

## 集成点

- **CLI 初始化**：`loadPolicyLimits()` 在启动期间调用
- **Init 模块**：`initializePolicyLimitsLoadingPromise()` 为其他系统提前调用以等待
- **功能门**：`isPolicyAllowed()` 在代码库中同步调用
- **Auth 更改**：`refreshPolicyLimits()` 在登录/注销时调用
- **HIPAA 模式**：启用仅基本流量时 `ESSENTIAL_TRAFFIC_DENY_ON_MISS` 策略失败关闭

## 配置

- API 端点：`/api/claude_code/policy_limits`
- 缓存文件：`~/.claude/policy-limits.json`（模式 0o600）
- 轮询间隔：1 小时
- 获取超时：10 秒
- 最大重试次数：5，带指数退避

## 错误处理

- **失败开放**：API 失败继续而不限制（无阻塞）
- **陈旧缓存**：获取失败时，使用缓存的限制（如果有）
- **Auth 错误**：跳过重试（401/403 重试不会成功）
- **空限制**：404 响应删除缓存文件以防止陈旧设置
- **文件错误**：保存/加载失败记录但不崩溃

## 测试

- `_resetPolicyLimitsForTesting()` 清除会话缓存、加载承诺并停止轮询
- 比 `clearPolicyLimitsCache()` 更便宜（无文件 I/O）用于测试 beforeEach

## 相关模块

- [远程托管设置](./remote-managed-settings.md) — 类似模式（ETag、轮询、失败开放）
- [分析](./analytics-service.md) — 遥测事件

## 备注

- 只有被阻止的策略包含在 API 响应中 — 不存在意味着允许
- 加载承诺有 30 秒超时以防止如果 loadPolicyLimits() 从未被调用的死锁
- 会话缓存从文件在第一次读取时填充，避免重复磁盘 I/O
- 基本流量拒绝-未命中保护 HIPAA 组织免受缓存未命中静默重新启用功能
