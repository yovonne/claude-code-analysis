# Hooks 系统

## 目的

Hooks 系统为通知显示、工具权限处理和各种 UI/UX 功能提供 React hooks。它组织成三个区域：通用 hooks（`src/hooks/` 中的 70+ 个 hooks）、通知 hooks（`hooks/notifs/`）和工具权限 hooks（`hooks/toolPermission/`）。

## 位置

`restored-src/src/hooks/`

## 关键子目录

### hooks/notifs/ — 通知 Hooks

用于显示启动和运行时通知的专门 hooks。

| Hook | 用途 |
|------|---------|
| `useStartupNotification` | 挂载时触发一次通知；封装远程模式门控和每会话一次 ref 保护 |
| `useModelMigrationNotifications` | 模型迁移后显示一次性通知（Sonnet 4.6、Opus 4.6） |
| `useDeprecationWarningNotification` | 使用弃用模型时警告 |
| `useFastModeNotification` | 处理快速模式冷却、超额拒绝和组织可用性变化 |
| `useAutoModeUnavailableNotification` | 在 shift-tab 轮播期间自动模式不可用时显示通知 |
| `useRateLimitWarningNotification` | 显示来自 claude.ai 的速率限制警告和超额通知 |
| `useSettingsErrors` | 显示设置验证错误 |
| `usePluginAutoupdateNotification` | 通知插件自动更新状态 |
| `usePluginInstallationStatus` | 跟踪插件安装进度 |
| `useMcpConnectivityStatus` | 监控 MCP 服务器连接 |
| `useLspInitializationNotification` | 通知 LSP 服务器初始化 |
| `useTeammateShutdownNotification` | 群体队友关闭时通知 |
| `useNpmDeprecationNotification` | 警告弃用的 npm 包 |
| `useInstallMessages` | 显示安装相关消息 |
| `useIDEStatusIndicator` | IDE 连接状态指示器 |
| `useCanSwitchToExistingSubscription` | 订阅切换能力检查 |

### hooks/toolPermission/ — 权限流程

处理三种权限流程：协调器、交互式和群体 worker。

#### `PermissionContext.ts`

创建冻结的权限上下文对象，封装单个工具权限请求的所有操作。

**关键导出:**
- `createPermissionContext(tool, input, toolUseContext, assistantMessage, toolUseID, ...)`: 权限上下文的工厂函数
- `createPermissionQueueOps(setToolUseConfirmQueue)`: 将 React 状态桥接到通用队列接口
- `createResolveOnce(resolve)`: 防止多次 promise 解析的原子 guard

**上下文方法:**
- `logDecision(args)`: 通过 `permissionLogging.ts` 记录权限决策
- `logCancelled()`: 记录工具取消分析
- `persistPermissions(updates)`: 将权限更新持久化到设置
- `resolveIfAborted(resolve)`: 检查中止信号并解析为取消决策
- `cancelAndAbort(feedback, isAbort, contentBlocks)`: 构建拒绝决策并可选中止
- `tryClassifier(pendingCheck, updatedInput)`: 运行 bash 分类器自动批准（BASH_CLASSIFIER 标志）
- `runHooks(permissionMode, suggestions, ...)`: 执行 PermissionRequest hooks
- `buildAllow(updatedInput, opts)`: 构建允许决策
- `buildDeny(message, reason)`: 构建拒绝决策
- `handleUserAllow(...)`: 处理用户接受及持久化和日志
- `handleHookAllow(...)`: 处理基于 hook 的接受
- `pushToQueue()`、`removeFromQueue()`、`updateQueueItem()`: 队列操作

#### `handlers/coordinatorHandler.ts`

处理协调器 worker（非交互式）的权限流程。

- 首先运行 hooks（快速，本地），然后分类器（慢速，推理）
- 如果两者都无法解决，则通过交互式对话框
- 返回 `PermissionDecision | null`

#### `handlers/interactiveHandler.ts`

处理主代理交互式权限流程——最复杂的处理程序。

**关键职责:**
- 将 `ToolUseConfirm` 条目推送到确认队列，带回调
- 竞赛 4 个权限来源：本地用户、bridge (CCR)、channel (Telegram/iMessage)、hooks/分类器
- 使用 `claim()` 原子 guard 防止多次解析
- 管理分类器自动批准的复选框转换计时器
- 处理带唯一请求 ID 的 bridge 权限请求
- 通过 MCP 通知处理 channel 权限转发
- 在后台异步运行 hooks 和分类器

**回调生命周期:**
- `onUserInteraction()`: 隐藏分类器指示器（200ms 宽限期）
- `onDismissCheckmark()`: 提前解散复选框转换（Esc 键）
- `onAbort()`: 用户中止——解析为取消决策
- `onAllow()`: 用户接受——持久化并解析
- `onReject()`: 用户拒绝——解析为反馈
- `recheckPermission()`: 重新评估权限（例如模式切换后）

#### `handlers/swarmWorkerHandler.ts`

处理群体 worker 代理的权限流程。

- 首先尝试分类器自动批准
- 通过邮箱将权限请求转发给领导者
- 注册领导者响应的回调
- 等待时显示待处理指示器
- 错误时回退到本地处理

### 通用 Hooks（选录）

| 类别 | Hooks |
|----------|-------|
| **输入** | `useTextInput`、`useVimInput`、`useArrowKeyHistory`、`useHistorySearch`、`useTypeahead`、`usePasteHandler`、`useSearchInput` |
| **状态** | `useAppState`、`useSettings`、`useSettingsChange`、`useDynamicConfig`、`useMemoryUsage` |
| **会话** | `useAssistantHistory`、`useRemoteSession`、`useSessionBackgrounding`、`useTeleportResume`、`useAwaySummary` |
| **工具** | `useCanUseTool`、`useCancelRequest`、`useTurnDiffs`、`useDiffData`、`useDiffInIDE` |
| **导航** | `useBackgroundTaskNavigation`、`useTaskListWatcher`、`useTasksV2` |
| **IDE** | `useIDEIntegration`、`useIdeAtMentioned`、`useIdeConnectionStatus`、`useIdeLogging`、`useIdeSelection` |
| **语音** | `useVoice`、`useVoiceEnabled`、`useVoiceIntegration` |
| **UI** | `useBlink`、`useClipboardImageHint`、`useMinDisplayTime`、`useVirtualScroll`、`useTerminalSize` |
| **功能** | `useManagePlugins`、`useLspPluginRecommendation`、`useSkillImprovementSurvey`、`useSwarmInitialization` |
| **消息** | `useInboxPoller`、`useMailboxBridge`、`useCommandQueue`、`useDeferredHookMessages`、`useLogMessages` |
| **性能** | `useAfterFirstRender`、`useElapsedTime`、`useNotifyAfterTimeout`、`useTimeout` |
| **快捷键** | `useCommandKeybindings`、`useGlobalKeybindings`、`useExitOnCtrlCD` |

## 依赖

### 内部依赖

- `bootstrap/state.js` — 远程模式检查、允许的通道
- `context/notifications.js` — 通知系统
- `state/AppState.js` — 应用状态选择器和设置器
- `Tool.js` — 工具类型和 ToolUseContext
- `utils/permissions/*` — 权限规则、更新、结果
- `utils/hooks.js` — Hook 执行
- `utils/swarm/permissionSync.js` — 群体权限转发
- `services/analytics/index.js` — 分析日志
- `services/mcp/*` — 通道权限转发

### 外部依赖

- `react` — 所有 hooks 都是 React hooks
- `react/compiler-runtime` — React Compiler 优化（`_c` 缓存）
- `bun:bundle` — 功能标志检查
- `@anthropic-ai/sdk` — 内容块类型

## 实现细节

### React Compiler

所有通知 hooks 都使用 React Compiler 编译（证据：`import { c as _c } from "react/compiler-runtime"` 和手动缓存数组 `$[0]`、`$[1]` 等）。这消除了手动 `useMemo`/`useCallback` 使用。

### Resolve-Once 模式

`createResolveOnce()` 工厂提供带三个操作的原子解析 guard：
- `resolve(value)`: 传递值一次（幂等）
- `isResolved()`: 检查是否已解析
- `claim()`: 原子检查和标记——仅当此调用者赢得竞争时返回 true

这防止多个权限来源（用户、hook、分类器、bridge、channel）竞争时的竞争条件。

### 权限流程决策树

```
工具需要权限
    ↓
是群体 worker？ → handleSwarmWorkerPermission()
    ↓ 否
是协调器？ → handleCoordinatorPermission()
    ↓ 否（交互式）
handleInteractivePermission()
    ├── 推入队列（UI 对话框）
    ├── 竞赛: Bridge 响应 (CCR)
    ├── 竞赛: Channel 响应 (Telegram/iMessage)
    ├── 竞赛: Hooks（异步）
    └── 竞赛: 分类器（异步，仅 bash）
    ↓
第一个 claim() 获胜
```

## 相关模块

- [引导](./bootstrap.md) — 权限状态从引导读取
- [Schemas](./schemas.md) — Hook schemas 定义 hook 配置结构
- [上下文提供者](./context-providers.md) — Hooks 消费的通知上下文
- [类型](./types-system.md) — 定义决策形状的权限类型
