# LSP 服务

## 目的

语言服务器协议（LSP）服务使 Claude Code 能够与外部语言服务器接口以实现代码智能。它管理 LSP 服务器进程的完整生命周期 — 发现、生成、初始化、请求路由和优雅关闭 — 同时通过 LSPTool 暴露语言特性（转到定义、悬停、诊断、引用、调用层次结构）。服务器仅从插件中发现，而非用户或项目设置。

## 位置

| 文件 | 行数 | 角色 |
|------|-------|------|
| `restored-src/src/services/lsp/LSPClient.ts` | 448 | 包装 vscode-jsonrpc 的低级 JSON-RPC 客户端 |
| `restored-src/src/services/lsp/LSPServerInstance.ts` | 512 | 带状态机和重试的单服务器生命周期 |
| `restored-src/src/services/lsp/LSPServerManager.ts` | 421 | 按文件扩展名路由的多服务器管理器 |
| `restored-src/src/services/lsp/manager.ts` | 290 | 带异步初始化的单例管理器外观 |
| `restored-src/src/services/lsp/config.ts` | 80 | 从插件加载服务器配置 |
| `restored-src/src/services/lsp/LSPDiagnosticRegistry.ts` | 387 | 异步诊断通知注册表 |
| `restored-src/src/services/lsp/passiveFeedback.ts` | 329 | 诊断处理程序注册和格式化 |

## 关键导出

### LSPClient.ts

| 导出 | 描述 |
|--------|-------------|
| `LSPClient`（类型） | 接口：`start`、`initialize`、`sendRequest`、`sendNotification`、`onNotification`、`onRequest`、`stop`、`capabilities`、`isInitialized` |
| `createLSPClient(serverName, onCrash?)` | 工厂函数，生成子进程并通过 stdio 创建 `MessageConnection` |

### LSPServerInstance.ts

| 导出 | 描述 |
|--------|-------------|
| `LSPServerInstance`（类型） | 接口：`name`、`config`、`state`、`startTime`、`lastError`、`restartCount`、`start()`、`stop()`、`restart()`、`isHealthy()`、`sendRequest()`、`sendNotification()`、`onNotification()`、`onRequest()` |
| `createLSPServerInstance(name, config)` | 包装 `LSPClient` 的工厂函数，带状态跟踪、崩溃恢复和瞬态错误重试 |

### LSPServerManager.ts

| 导出 | 描述 |
|--------|-------------|
| `LSPServerManager`（类型） | 接口：`initialize()`、`shutdown()`、`getServerForFile()`、`ensureServerStarted()`、`sendRequest()`、`getAllServers()`、`openFile()`、`changeFile()`、`saveFile()`、`closeFile()`、`isFileOpen()` |
| `createLSPServerManager()` | 管理多个服务器实例的工厂函数，基于扩展名路由 |

### manager.ts

| 导出 | 描述 |
|--------|-------------|
| `getLspServerManager()` | 返回单例管理器实例（如果未初始化/失败则为 `undefined`） |
| `getInitializationStatus()` | 返回 `{ status: 'not-started' | 'pending' | 'success' | 'failed', error? }` |
| `isLspConnected()` | 如果至少有一个服务器不在错误状态则返回 `true` |
| `waitForInitialization()` | 等待异步初始化完成 |
| `initializeLspServerManager()` | 创建单例并启动异步初始化（在启动时调用） |
| `reinitializeLspServerManager()` | 在插件刷新后强制重新初始化 |
| `shutdownLspServerManager()` | 停止所有服务器并清除状态 |

### LSPDiagnosticRegistry.ts

| 导出 | 描述 |
|--------|-------------|
| `PendingLSPDiagnostic`（类型） | `{ serverName, files, timestamp, attachmentSent }` |
| `registerPendingLSPDiagnostic({ serverName, files })` | 存储异步传递的诊断 |
| `checkForLSPDiagnostics()` | 检索并去重的待处理诊断 |
| `clearAllLSPDiagnostics()` | 清除待处理诊断 |
| `resetAllLSPDiagnosticState()` | 清除待处理和已传递跟踪 |
| `clearDeliveredDiagnosticsForFile(fileUri)` | 清除特定文件的跨回合去重 |
| `getPendingLSPDiagnosticCount()` | 返回待处理诊断计数 |

### passiveFeedback.ts

| 导出 | 描述 |
|--------|-------------|
| `formatDiagnosticsForAttachment(params)` | 将 `PublishDiagnosticsParams` 转换为 `DiagnosticFile[]` |
| `registerLSPNotificationHandlers(manager)` | 在所有服务器上注册 `textDocument/publishDiagnostics` 处理程序 |
| `HandlerRegistrationResult`（类型） | `{ totalServers, successCount, registrationErrors, diagnosticFailures }` |

### config.ts

| 导出 | 描述 |
|--------|-------------|
| `getAllLspServers()` | 并行从启用的插件加载所有服务器配置 |

## 依赖

### 内部

| 模块 | 用途 |
|--------|---------|
| `utils/plugins/lspPluginIntegration.ts` | 插件作用域的服务器加载和环境变量解析 |
| `utils/plugins/pluginLoader.ts` | 插件发现 |
| `utils/debug.ts` | 调试日志 |
| `utils/log.ts` | 错误日志 |
| `utils/errors.ts` | 错误格式化 |
| `utils/subprocessEnv.ts` | 生成进程的环境 |
| `utils/cwd.ts` | 当前工作目录 |
| `utils/sleep.ts` | 重试退避延迟工具 |
| `utils/attachments.ts` | 诊断附件格式化 |
| `services/diagnosticTracking.ts` | `DiagnosticFile` 类型 |

### 外部

| 包 | 用途 |
|---------|---------|
| `vscode-jsonrpc` | 通过 stdio 流的 JSON-RPC 消息连接 |
| `vscode-languageserver-protocol` | LSP 协议类型（`InitializeParams`、`ServerCapabilities` 等） |
| `child_process` | 生成 LSP 服务器进程 |
| `lru-cache` | 已传递诊断的有界去重缓存 |
| `path`, `url` | 文件路径和 URI 转换 |

## 实现细节

### 架构概述

LSP 服务使用三层工厂模式与闭包（无类）：

```
┌─────────────────────────────────────────────────┐
│  manager.ts (单例外观)                           │
│  - initializeLspServerManager()               │
│  - getLspServerManager()                        │
│  - 带生成计数器的异步初始化                       │
└──────────────────────┬──────────────────────────┘
                       │
┌──────────────────────▼──────────────────────────┐
│  LSPServerManager (多服务器路由器)               │
│  - 扩展名 → 服务器映射                           │
│  - 文件打开/关闭/更改/保存同步                    │
│  - 按文件路径请求路由                            │
└──────────────────────┬──────────────────────────┘
                       │
┌──────────────────────▼──────────────────────────┐
│  LSPServerInstance (每服务器生命周期)             │
│  - 状态机：stopped→starting→running             │
│  - 带最大重启上限的崩溃恢复                       │
│  - 瞬态错误重试 (ContentModified)               │
└──────────────────────┬──────────────────────────┘
                       │
┌──────────────────────▼──────────────────────────┐
│  LSPClient (JSON-RPC 传输)                      │
│  - 子进程生成                                    │
│  - 通过 stdio 的 StreamMessageReader/Writer     │
│  - 请求/通知分派                                 │
└─────────────────────────────────────────────────┘
```

### 服务器生命周期管理

#### 状态机

每个 `LSPServerInstance` 遵循严格的状态机：

```
stopped ──start()──▶ starting ──init ok──▶ running
  ▲                    │                      │
  │                    │ error                │ error
  │                    ▼                      ▼
  │                  error ◀──── restart() ── error
  │                    │                      │
  │                    │ stop()               │ stop()
  │                    ▼                      ▼
  └────────────── stopped ◀──── stopping ◀── running
```

状态：`stopped`、`starting`、`running`、`stopping`、`error`

#### 初始化流程

1. **manager.ts** 同步创建 `LSPServerManager` 实例，标记状态为 `pending`
2. 异步 `manager.initialize()` 调用 `getAllLspServers()` 加载插件配置
3. 对于每个服务器配置，创建 `LSPServerInstance` 并构建扩展名映射
4. 注册 `workspace/configuration` 请求处理程序（为每个项目返回 `null`）
5. 成功时，`registerLSPNotificationHandlers()` 附加诊断监听器
6. 状态转换为 `success`；管理器实例可用

#### 启动序列（每服务器）

1. `client.start(command, args, options)` — 生成带管道 stdio 的子进程
2. 等待 `spawn` 事件（关键：`spawn()` 在 `error` 异步触发之前返回）
3. 从 `StreamMessageReader(process.stdout)` 和 `StreamMessageWriter(process.stdin)` 创建 `MessageConnection`
4. 在 `connection.listen()` 之前注册错误/关闭处理程序以防止未处理的拒绝
5. 启用详细协议跟踪以进行调试
6. 应用任何排队的通知/请求处理程序（延迟初始化支持）
7. `client.initialize(params)` — 发送带有工作区信息和客户端能力的 `initialize` 请求
8. 发送 `initialized` 通知
9. 状态转换为 `running`

#### 关闭序列

1. `isStopping = true` 以在故意关闭期间抑制错误日志
2. 发送 `shutdown` 请求，然后是 `exit` 通知
3. 处置连接，移除所有进程事件监听器
4. 终止进程（捕获错误 — 进程可能已经死了）
5. 重置 `isInitialized`、`capabilities`、`isStopping`

#### 崩溃恢复

- `onCrash` 回调将状态设置为 `error` 并增加 `crashRecoveryCount`
- `ensureServerStarted()` 检测 `error` 状态并自动重启
- 最大崩溃恢复上限为 `config.maxRestarts`（默认：3）
- 防止持续崩溃的服务器无限制生成子进程

### 语言特性集成

LSP 服务通过 `LSPTool` 支持这些语言特性：

| 特性 | LSP 方法 | 声明的客户端能力 |
|---------|-----------|--------------------------|
| 转到定义 | `textDocument/definition` | `linkSupport: true` |
| 查找引用 | `textDocument/references` | `dynamicRegistration: false` |
| 悬停 | `textDocument/hover` | `contentFormat: ['markdown', 'plaintext']` |
| 文档符号 | `textDocument/documentSymbol` | `hierarchicalDocumentSymbolSupport: true` |
| 工作区符号 | `workspace/symbol` | — |
| 转到实现 | `textDocument/implementation` | — |
| 调用层次结构 | `textDocument/prepareCallHierarchy` → `callHierarchy/incomingCalls` / `callHierarchy/outgoingCalls` | `dynamicRegistration: false` |
| 诊断 | `textDocument/publishDiagnostics`（通知） | `relatedInformation`、`tagSupport`、`codeDescriptionSupport` |

#### 客户端能力

在初始化期间声明：

```typescript
capabilities: {
  workspace: {
    configuration: false,      // 不声明支持（阻止服务器请求）
    workspaceFolders: false,   // 不声明支持（未处理）
  },
  textDocument: {
    synchronization: { willSave: false, willSaveWaitUntil: false, didSave: true },
    publishDiagnostics: { relatedInformation: true, tagSupport: { valueSet: [1, 2] } },
    hover: { contentFormat: ['markdown', 'plaintext'] },
    definition: { linkSupport: true },
    documentSymbol: { hierarchicalDocumentSymbolSupport: true },
  },
  general: { positionEncodings: ['utf-16'] },
}
```

### LSPTool 集成

`LSPTool`（在 `analysis/02-tools/LSPTool.md` 中单独记录）通过以下方式与 LSP 服务集成：

1. **服务器发现**：`getServerForFile(filePath)` 按文件扩展名路由
2. **文件同步**：`openFile()`、`changeFile()`、`saveFile()` 发送 `didOpen`、`didChange`、`didSave` 通知
3. **请求路由**：`sendRequest(filePath, method, params)` 如需要自动启动服务器
4. **诊断传递**：`LSPDiagnosticRegistry` 存储异步诊断用于附件系统
5. **可用性检查**：`isLspConnected()` 控制工具启用

### 服务器发现和配置

#### 基于插件的发现

LSP 服务器仅通过两种机制从插件中发现：

1. **插件目录中的 `.lsp.json` 文件** — 将服务器名称映射到配置的 JSON 对象
2. **`manifest.lspServers` 字段** — 可以是字符串（指向 JSON 的路径）、内联配置对象或两者的数组

#### 配置解析

```
插件加载
    │
    ├── .lsp.json 文件（可选）
    ├── manifest.lspServers（可选）
    │
    ▼
resolvePluginLspEnvironment()
    ├── ${CLAUDE_PLUGIN_ROOT} 替换
    ├── ${user_config.X} 替换
    ├── ${VAR} 通用环境扩展
    └── CLAUDE_PLUGIN_ROOT / CLAUDE_PLUGIN_DATA 注入到 env
    │
    ▼
addPluginScopeToLspServers()
    └── 服务器名称前缀："plugin:${pluginName}:${serverName}"
    │
    ▼
getAllLspServers() — 合并所有插件结果
```

#### ScopedLspServerConfig 字段

| 字段 | 类型 | 必需 | 描述 |
|-------|------|----------|-------------|
| `command` | `string` | 是 | 生成的 executable |
| `args` | `string[]` | 否 | 命令参数 |
| `env` | `Record<string, string>` | 否 | 环境变量 |
| `cwd` | `string` | 否 | 工作目录 |
| `workspaceFolder` | `string` | 否 | LSP 的工作区根目录 |
| `extensionToLanguage` | `Record<string, string>` | 是 | 文件扩展名 → 语言 ID 映射 |
| `initializationOptions` | `object` | 否 | 服务器特定初始化选项（例如 vue-language-server） |
| `startupTimeout` | `number` | 否 | 初始化超时前的毫秒数 |
| `maxRestarts` | `number` | 否 | 最大重启尝试次数（默认：3） |
| `scope` | `'dynamic'` | 自动 | 由插件作用域设置 |
| `source` | `string` | 自动 | 插件名称 |

### 诊断系统

#### 流程

```
LSP 服务器发送 publishDiagnostics
    │
    ▼
passiveFeedback.ts 处理程序
    ├── formatDiagnosticsForAttachment() — 转换 LSP → Claude 格式
    └── registerPendingLSPDiagnostic() — 存储到注册表
    │
    ▼
LSPDiagnosticRegistry
    ├── 去重（批内 + 跨回合通过 LRU 缓存）
    ├── 音量限制（每文件最多 10，总共最多 30）
    └── 严重性排序（Error > Warning > Info > Hint）
    │
    ▼
checkForLSPDiagnostics() — 由附件系统调用
    └── 将去重、有限的诊断作为 Attachment[] 返回
```

#### 去重

- **批内**：按文件 URI 分组，通过内容哈希去重（消息 + 严重性 + 范围 + 源 + 代码）
- **跨回合**：`LRUCache<string, Set<string>>` 跟踪已传递的诊断（最多 500 个文件），在编辑时按文件清除

#### 音量限制

| 限制 | 值 |
|-------|-------|
| 每文件最大诊断数 | 10 |
| 最大总诊断数 | 30 |
| 跟踪的最大文件数（LRU） | 500 |

### 请求重试逻辑

瞬态 `ContentModified` 错误（代码 `-32801`）通过指数退避重试：

- 最大重试次数：3
- 基础延迟：500ms
- 延迟：500ms、1000ms、2000ms
- 在 rust-analyzer 项目索引期间常见

### 文件同步

管理器通过 `openedFiles: Map<URI, serverName>` 跟踪打开的文件：

| 方法 | LSP 通知 | 行为 |
|--------|-----------------|----------|
| `openFile(path, content)` | `textDocument/didOpen` | 如果已打开则跳过；从配置获取 languageId |
| `changeFile(path, content)` | `textDocument/didChange` | 如果尚未打开则回退到 `openFile` |
| `saveFile(path)` | `textDocument/didSave` | 触发服务器端诊断 |
| `closeFile(path)` | `textDocument/didClose` | 从跟踪中移除（TODO：与 compact 集成） |

### 单例初始化

`manager.ts` 使用生成计数器模式来防止过时 promise 竞争：

```typescript
let initializationGeneration = 0

initializeLspServerManager():
  currentGeneration = ++initializationGeneration
  initializationPromise = manager.initialize().then(() => {
    if (currentGeneration === initializationGeneration) {
      initializationState = 'success'
    }
  })
```

重新初始化（在 `/reload-plugins` 后）增加生成，使进行中的 promise 无效。

### 错误处理

| 场景 | 行为 |
|----------|----------|
| 服务器生成失败 | 设置 `startFailed = true`，在下一次操作时抛出 |
| 服务器崩溃（非零退出） | 调用 `onCrash`，设置状态为 `error`，在下一次使用时自动重启 |
| 操作期间连接错误 | 记录，设置状态为 `error` |
| 初始化超时 | 终止进程，设置状态为 `error` |
| 请求失败 | 用上下文包装错误，重新抛出 |
| 通知失败 | 记录但不重新抛出（即发即忘） |
| 关闭失败 | 清理继续，错误记录但不传播 |

### 配置

#### 环境变量

| 变量 | 效果 |
|----------|----------|
| `--bare` 模式 | LSP 完全禁用（不需要编辑器集成） |

#### 功能标志

| 标志 | 效果 |
|------|----------|
| `isLspConnected()` 返回 false | LSPTool 被禁用 |

### 数据流

```
Claude Code 启动
    │
    ▼
initializeLspServerManager()
    ├── 创建 LSPServerManager 实例
    ├── 异步：从插件获取所有 LSP 服务器
    ├── 对于每个服务器：createLSPServerInstance()
    └── 成功时：registerLSPNotificationHandlers()
    │
    ▼
LSPTool 请求（例如 goToDefinition）
    ├── getServerForFile(filePath) — 按扩展名查找服务器
    ├── ensureServerStarted(filePath) — 如需要延迟启动
    ├── openFile(filePath, content) — 将文件同步到服务器
    └── sendRequest(method, params) — 执行 LSP 请求
    │
    ▼
被动诊断（异步）
    ├── LSP 服务器 → publishDiagnostics 通知
    ├── formatDiagnosticsForAttachment() → registerPendingLSPDiagnostic()
    └── checkForLSPDiagnostics() → 作为附件传递
    │
    ▼
关闭
    └── shutdownLspServerManager() → 停止所有服务器
```

## 备注

- LSP 在 `--bare` 模式下禁用（脚本化的 `-p` 调用不需要编辑器集成）
- `vscode-jsonrpc`（约 129KB）仅在实例化服务器时才被延迟需要
- `closeFile()` 存在但尚未与压缩流程集成（标记为 TODO）
- `restartOnCrash` 和 `shutdownTimeout` 配置字段已定义但使用时会抛出（尚未实现）
- 服务器是延迟启动的 — 仅在首次通过 `ensureServerStarted()` 访问时生成
- `workspace/configuration` 请求处理程序为所有项目返回 `null`，满足协议而不提供实际配置
