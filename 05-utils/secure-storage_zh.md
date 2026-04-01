# 安全存储工具

## 目的

安全存储工具模块为 Claude Code 提供平台感知的凭证存储系统。它抽象了多个存储后端（macOS 密钥链、明文文件），具有自动回退、启动预取优化和过时-错误缓存，确保在不同的环境中可靠地访问凭证。

## 位置

- `restored-src/src/utils/secureStorage/index.ts` — 工厂函数，创建平台适当的存储
- `restored-src/src/utils/secureStorage/types.ts` — 共享类型定义
- `restored-src/src/utils/secureStorage/macOsKeychainStorage.ts` — macOS 密钥链实现
- `restored-src/src/utils/secureStorage/macOsKeychainHelpers.ts` — 轻量级辅助函数（无重导入）
- `restored-src/src/utils/secureStorage/keychainPrefetch.ts` — 并行密钥链读取的启动预取
- `restored-src/src/utils/secureStorage/fallbackStorage.ts` — 主/辅助回退包装器
- `restored-src/src/utils/secureStorage/plainTextStorage.ts` — 明文文件回退（`.credentials.json`）

## 主要导出

### 工厂（`index.ts`）

#### 函数
- `getSecureStorage()`: 返回适当的存储实现：
  - macOS: `createFallbackStorage(macOsKeychainStorage, plainTextStorage)` — 首先尝试密钥链，回退到明文
  - 其他平台: 直接使用 `plainTextStorage`
  - TODO: 为 Linux 添加 libsecret 支持

### macOS 密钥链存储（`macOsKeychainStorage.ts`）

#### 对象：`macOsKeychainStorage`
使用 macOS `security` CLI 实现 `SecureStorage` 接口：

- `read()`: 带 TTL 缓存（30 秒）和过时-错误的同步读取 — 在瞬态失败时提供过时值
- `readAsync()`: 带去重的飞行中请求的异步读取（基于生成的陈旧检测）
- `update(data)`: 通过 `security -i`（stdin）写入以避免进程监视器看到负载。当负载超过 4032 字节 stdin 行限制时回退到 argv。数据存储为十六进制编码的 JSON。
- `delete()`: 通过 `security delete-generic-password` 删除密钥链条目

#### 函数
- `isMacOsKeychainLocked()`: 检查密钥链是否锁定（来自 `security show-keychain-info` 的退出代码 36）。为进程生命周期缓存，因为锁定状态在会话期间不会改变。

### macOS 密钥链辅助函数（`macOsKeychainHelpers.ts`）

**关键约束**：此模块不能导入 `execa`、`execFileNoThrow` 或类似的重模块。它在 `main.tsx` 非常靠前的位置加载，在 ~65ms 的模块评估之前，而 Bun 的 `__esm` 包装器在任何符号被访问时评估整个模块。

#### 常量
- `CREDENTIALS_SERVICE_SUFFIX`（`'-credentials'`）: OAuth 凭证密钥链条目的后缀
- `KEYCHAIN_CACHE_TTL_MS`（30,000）: 缓存 TTL，平衡跨进程陈旧性与避免重复同步生成

#### 函数
- `getMacOsKeychainStorageServiceName(serviceSuffix?)`: 从配置目录哈希构建服务名称（每个自定义配置目录唯一）
- `getUsername()`: 从环境或 `os.userInfo()` 获取当前用户名
- `clearKeychainCache()`: 使缓存失效（增加生成，清除飞行中承诺）
- `primeKeychainCacheFromPrefetch(stdout)`: 从预取结果填充缓存（仅在缓存未被触碰时）

#### 状态
- `keychainCacheState`: 带有 `cache`、`generation` 和 `readInFlight` 字段的共享可变对象。位于此处（不在存储模块中），以便 `keychainPrefetch.ts` 可以在不引入 `execa` 的情况下填充它。

### 密钥链预取（`keychainPrefetch.ts`）

**目的**：在启动时并行触发两个密钥链读取（OAuth 凭证 + 传统 API 密钥），将 ~65ms 子进程成本与 main.tsx 模块评估重叠。

#### 函数
- `startKeychainPrefetch()`: 立即生成两个 `security find-generic-password` 子进程（非阻塞）。非 darwin 和 bare 模式是空操作。
- `ensureKeychainPrefetchCompleted()`: 等待预取完成。在 main.tsx preAction 中调用 — 由于子进程在导入评估期间完成，几乎是免费的。
- `getLegacyApiKeyPrefetchResult()`: 获取传统 API 密钥预取结果，供 auth.ts 跳过其同步生成。
- `clearLegacyApiKeyPrefetch()`: 在缓存失效时清除预取结果。

#### 设计
- 区分"未启动"（`null`）和"已完成但无密钥"（`{ stdout: null }`），以便同步读取器仅信任已完成的预取。
- 超时的预取不会填充 null — 同步路径用其自己的更长超时重试。

### 回退存储（`fallbackStorage.ts`）

#### 函数
- `createFallbackStorage(primary, secondary)`: 创建一个存储包装器，首先尝试主要，失败时回退到次要。在先前为空的主要成功写入后，删除次要条目（迁移）。在主要写入失败但现有数据时，删除过时的主要条目以防止掩盖新的次要数据。

### 明文存储（`plainTextStorage.ts`）

#### 对象：`plainTextStorage`
使用 Claude 配置目录中的 `.credentials.json` 文件实现 `SecureStorage`：

- `read()`、`readAsync()`: 读取并解析 JSON 文件
- `update(data)`: 用 `chmod 0o600`（仅所有者读写）写入 JSON。返回关于明文存储的警告。
- `delete()`: 取消链接凭证文件

## 依赖

- `execa` — 进程执行（仅密钥链存储，不是辅助函数/预取）
- `../execFileNoThrow.js`、`../execFileNoThrowPortable.js` — 进程执行包装器
- `../slowOperations.js` — 带 lodash cloneDeep 的 JSON 操作
- `../envUtils.js` — 配置目录检测
- `../fsOperations.js` — 文件系统抽象
- `src/constants/oauth.js` — OAuth 配置

## 设计注意事项

- **启动优化**：密钥链预取模式在 `main.tsx` 最顶部生成子进程，将 ~65ms 的同步 `security` 生成与模块评估并行化。`macOsKeychainHelpers.ts` 模块被仔细保持无重导入，以避免破坏此优化。
- **过时-错误缓存**：密钥链读取为 30 秒缓存结果，并在瞬态失败时提供过时值。这防止了在 50+ MCP 连接器同时认证时启动风暴期间的"Not logged in"闪烁。
- **基于生成的陈旧性**：`readAsync()` 在生成前捕获生成编号，如果存在更新的生成，则跳过其缓存写入，防止过时的子进程结果覆盖新的 `update()` 写入。
- **进程监视器规避**：密钥链写入使用 `security -i`（stdin），因此进程监视器（CrowdStrike 等）只看到 `security -i`，而不是十六进制编码的负载。当负载超过 4032 字节 stdin 行限制时回退到 argv。
- **迁移安全**：当从明文迁移到密钥链时，回退存储在成功密钥链写入后删除明文条目。当主要写入失败时，它删除过时的主要条目以防止掩盖新的次要数据（修复了轮换刷新令牌的 /login 循环）。
