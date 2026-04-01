# 状态管理

## 目的
Claude Code CLI 使用基于 React 的 `useSyncExternalStore` 构建的集中式、类似 Redux 的存储模式。一个单一的 `AppState` 对象持有所有应用程序状态 — 设置、权限上下文、任务、MCP 连接、通知、桥接状态等。状态更新通过不可变的更新器模式（`setState(prev => next)`）流动，具有同步副作用（持久化到磁盘、通知 CCR/SDK、清除缓存）的中央 `onChangeAppState` 观察器。

## 位置
- `restored-src/src/state/` - 核心状态管理（store、AppState、selectors）
- `restored-src/src/context/` - React 上下文提供者（notifications、modals、mailbox、stats、overlays）
- `restored-src/src/hooks/` - 自定义 React hooks（permission、bridge、settings 等）
- `restored-src/src/types/permissions.ts` - 权限类型定义
- `restored-src/src/Tool.ts` - ToolPermissionContext 和 ToolUseContext 类型

## 关键导出

### AppState
- `AppState`：包含所有应用程序状态的单一不可变状态树（~80+ 字段）。用 `DeepImmutable<>` 包装以确保类型安全。
- `AppStateStore`：存储接口（`Store<AppState>`），具有 `getState`、`setState`、`subscribe`
- `getDefaultAppState()`：返回默认初始状态的工厂函数
- `AppStoreContext`：持有存储实例的 React 上下文
- `useAppState(selector)`：基于选择器的订阅 hook，使用 `useSyncExternalStore` — 仅在所选值更改时重新渲染（Object.is 比较）
- `useSetAppState()`：返回稳定的 `setState` 更新器 — 从不导致重新渲染
- `useAppStateStore()`：返回用于非 React 代码访问的完整存储
- `useAppStateMaybeOutsideOfProvider()`：安全版本，在提供者外部返回 `undefined`

### 上下文提供者
- `AppStateProvider`：包装应用程序的根提供者，创建存储，嵌套 `MailboxProvider` 和 `VoiceProvider`
- `MailboxProvider`：提供用于组件间消息传递的单例 `Mailbox` 实例
- `StatsProvider`：提供带 increment、observe、histogram 支持的指标/统计存储
- `NotificationContext`（通过 `useNotifications`）：带 fold、invalidate、timeout 的基于优先级的通知队列
- `ModalContext`：跟踪模态布局维度以正确调整覆盖内容的大小
- `OverlayContext`：跟踪活动覆盖以协调 Escape 键
- `QueuedMessageContext`：跟踪消息队列状态以进行简要布局渲染

### 自定义 Hooks
- `useAppState(selector)`：通过基于选择器的记忆化订阅 AppState 的一个切片
- `useSetAppState()`：获取 setState 更新器而不订阅
- `useAppStateStore()`：获取完整存储引用
- `useNotifications()`：添加/删除带优先级队列的通知
- `useRegisterOverlay(id)`：注册/取消注册覆盖组件
- `useIsOverlayActive()`：检查是否有任何覆盖处于活动状态
- `useCanUseTool()`：用于工具执行的权限检查 hook
- `useReplBridge()`：桥接连接生命周期 hook
- `useSettingsChange()`：监听设置文件更改

## AppState 架构

`AppState` 类型是一个大型冻结对象（~80+ 字段），组织成逻辑组：

```typescript
export type AppState = DeepImmutable<{
  // 核心设置和模型
  settings: SettingsJson
  verbose: boolean
  mainLoopModel: ModelSetting
  mainLoopModelForSession: ModelSetting

  // UI 状态
  statusLineText: string | undefined
  expandedView: 'none' | 'tasks' | 'teammates'
  isBriefOnly: boolean
  selectedIPAgentIndex: number
  coordinatorTaskIndex: number
  viewSelectionMode: 'none' | 'selecting-agent' | 'viewing-agent'
  footerSelection: FooterItem | null

  // 权限状态
  toolPermissionContext: ToolPermissionContext

  // 任务状态（从 DeepImmutable 排除 — 包含函数）
  tasks: { [taskId: string]: TaskState }
  foregroundedTaskId?: string
  viewingAgentTaskId?: string

  // MCP 和插件
  mcp: { clients, tools, commands, resources, pluginReconnectKey }
  plugins: { enabled, disabled, commands, errors, installationStatus, needsRefresh }

  // 通知和引出
  notifications: { current: Notification | null, queue: Notification[] }
  elicitation: { queue: ElicitationRequestEvent[] }

  // 桥接（远程控制）
  replBridgeEnabled, replBridgeConnected, replBridgeSessionActive, ...
  replBridgePermissionCallbacks?: BridgePermissionCallbacks

  // 团队/swarm 上下文
  teamContext?: { teamName, teammates, leadAgentId, ... }
  inbox: { messages: Array<{ id, from, text, timestamp, status }> }

  // 推测（预计算）
  speculation: SpeculationState
  speculationSessionTimeSavedMs: number

  // ... 还有更多：todos、fileHistory、attribution、promptSuggestion 等
}>
```

关键设计决策：
- **`DeepImmutable<>` 包装器**：大多数字段是深度不可变的，以实现可预测的更新。`tasks` 被排除，因为 `TaskState` 包含函数类型（AbortController、callbacks）。
- **推测中的可变 refs**：`SpeculationState.active` 使用 `messagesRef` 和 `writtenPathsRef` 作为可变 refs，以避免每条消息的数组展开 — 热路径的性能优化。
- **功能标志字段**：许多字段是可选的，仅在功能标志启用时填充（`BRIDGE_MODE`、`KAIROS`、`CHICAGO_MCP` 等）。

## React 状态管理

存储是一个最小的外部存储实现（`src/state/store.ts:10-34`）：

```typescript
export function createStore<T>(initialState: T, onChange?: OnChange<T>): Store<T> {
  let state = initialState
  const listeners = new Set<Listener>()

  return {
    getState: () => state,
    setState: (updater: (prev: T) => T) => {
      const prev = state
      const next = updater(prev)
      if (Object.is(next, prev)) return  // 如果没有变化则跳过
      state = next
      onChange?.({ newState: next, oldState: prev })
      for (const listener of listeners) listener()
    },
    subscribe: (listener: Listener) => {
      listeners.add(listener)
      return () => listeners.delete(listener)
    },
  }
}
```

关键特性：
- **更新器模式**：所有状态更改通过 `setState(prev => next)` 进行 — 无直接变更
- **Object.is 相等性检查**：如果新状态与旧状态是引用相同的，则跳过通知
- **onChange 回调**：在监听器之前触发，用于副作用（持久化、CCR 同步）
- **监听器扇出**：所有订阅的监听器同步通知

`AppStateProvider`（`src/state/AppState.tsx:37-110`）通过 `useState` 创建一次存储并通过 `AppStoreContext` 提供。它还：
1. 防止嵌套（如果在另一个 `AppStateProvider` 内部则抛出）
2. 如果远程设置禁用，则在挂载时初始化旁路权限模式
3. 通过 `useSettingsChange` 订阅设置文件更改
4. 用 `MailboxProvider` 和 `VoiceProvider` 包装子级

### useAppState — 基于选择器的订阅

```typescript
export function useAppState<T>(selector: (state: AppState) => T): T {
  const store = useAppStore()
  const get = () => selector(store.getState())
  return useSyncExternalStore(store.subscribe, get, get)
}
```

使用 React 18 的 `useSyncExternalStore` 进行并发安全订阅。选择器函数提取特定字段，React 仅在 `Object.is` 检测到所选值更改时重新渲染。

**最佳实践**：选择单个字段，而非整个状态：
```typescript
const verbose = useAppState(s => s.verbose)          // 好
const mode = useAppState(s => s.toolPermissionContext.mode)  // 好
const state = useAppState(s => s)                    // 坏 — 每次更改都重新渲染
```

## 上下文提供者

### AppStateProvider
根提供者。创建存储，处理挂载时权限初始化，订阅设置更改。

### MailboxProvider（`src/context/mailbox.tsx`）
提供用于组件间发布/订阅消息的单例 `Mailbox` 实例。通过 `useMemo(() => new Mailbox(), [])` 创建一次。

### StatsProvider（`src/context/stats.tsx`）
提供带以下功能的指标存储：
- **计数器**：`increment(name, value)` — 累积计数
- **仪表**：`set(name, value)` — 当前值
- **直方图**：`observe(name, value)` — 水库采样（算法 R，大小 1024），带 p50/p95/p99 百分位数
- **集合**：`add(name, value)` — 唯一值跟踪

指标在进程退出时刷新到项目配置中的 `lastSessionMetrics`。

### 通知系统（`src/context/notifications.tsx`）
带四个级别的基于优先级的通知队列：`immediate`、`high`、`medium`、`low`。

关键特性：
- **优先级调度**：`getNext()` 从队列中选择最高优先级通知
- **折叠**：具有相同键的通知可以通过 `fold` 函数合并
- **使无效**：通知可以通过 `invalidates` 数组使其他通知无效
- **自动超时**：默认 8 秒超时，可按通知配置
- **即时绕过**：`immediate` 优先级通知跳过队列并立即显示

### OverlayContext（`src/context/overlayContext.tsx`）
通过 AppState 中的 `activeOverlays: ReadonlySet<string>` 跟踪活动覆盖。用于 Escape 键协调 — 当覆盖处于活动状态时，Escape 关闭覆盖而非取消运行中的任务。

- `useRegisterOverlay(id, enabled?)`：在挂载时自动注册，在卸载时取消注册
- `useIsOverlayActive()`：如果任何覆盖处于活动状态则返回 `true`
- `useIsModalOverlayActive()`：如果任何模态（非自动完成）覆盖处于活动状态则返回 `true`

### ModalContext（`src/context/modalContext.tsx`）
提供模态布局维度（`rows`、`columns`、`scrollRef`），以正确调整斜杠命令对话框内容的大小。

## 状态持久化

状态持久化在多个级别处理：

### 1. onChangeAppState 观察器（`src/state/onChangeAppState.ts`）
`onChangeAppState` 函数在每次状态更改时触发，处理：

- **权限模式同步**：检测模式更改并通过 `notifySessionMetadataChanged()` 通知 CCR，通过 `notifyPermissionModeChanged()` 通知 SDK。发送前将内部模式外部化（例如 `bubble` → `default`）。
- **模型设置**：当 `mainLoopModel` 更改时，通过 `updateSettingsForSource()` 持久化到用户设置
- **UI 偏好**：`expandedView` → 将 `showExpandedTodos` 和 `showSpinnerTree` 持久化到全局配置
- **详细标志**：持久化到全局配置
- **Tungsten 面板**：将 `tungstenPanelVisible` 持久化到全局配置（仅限 ant）
- **认证缓存清除**：当设置更改时清除 API 密钥、AWS 和 GCP 凭证缓存
- **环境变量**：当 `settings.env` 更改时重新应用配置环境变量

### 2. 全局配置
UI 偏好（详细、扩展视图、tungsten 面板）通过 `saveGlobalConfig()` 持久化到磁盘上的 JSON 文件。

### 3. 设置系统
模型、权限规则和其他用户偏好通过 `updateSettingsForSource()` 持久化到 `settings.json`（用户级、项目级或本地）。

### 4. 统计持久化
会话指标在进程退出时通过 `process.on('exit')` 处理程序刷新到项目配置中的 `lastSessionMetrics`。

## 状态更新模式

### 不可变更新器模式
所有状态更改使用函数更新器模式：
```typescript
setAppState(prev => ({
  ...prev,
  toolPermissionContext: {
    ...prev.toolPermissionContext,
    mode: newMode,
  },
}))
```

### 嵌套对象更新
对于嵌套对象，在每个层级展开：
```typescript
setAppState(prev => ({
  ...prev,
  notifications: {
    ...prev.notifications,
    current: next,
    queue: prev.notifications.queue.filter(n => n !== next),
  },
}))
```

### Set/Map 更新
对于 `Set` 和 `Map` 字段，创建新实例：
```typescript
setAppState(prev => {
  const next = new Set(prev.activeOverlays)
  next.add(id)
  return { ...prev, activeOverlays: next }
})
```

### 批量更新
多个字段可以在单个 `setState` 调用中更新。存储级别的 `Object.is` 检查可防止结果状态引用相同时的不必要通知。

### 计算状态的选择器
`selectors.ts` 文件提供用于派生计算状态的纯函数：
- `getViewedTeammateTask(appState)`：返回当前查看的队友任务或 `undefined`
- `getActiveAgentForInput(appState)`：返回指示用户输入应路由到哪里的区分联合（`leader`、`viewed` 或 `named_agent`）

## 消息状态

消息不直接存储在 `AppState` 中 — 它们存在于 REPL 组件的本地状态中，并通过 `useReplBridge` 等 hook 传递。然而，AppState 包含消息相关状态：

- `initialMessage`：要处理初始 CLI 消息，带可选的 `clearContext`、`mode` 和 `allowedPrompts`
- `promptSuggestion`：带 `text`、`promptId`、`shownAt`、`acceptedAt` 的 AI 生成的后续建议
- `notifications`：通知队列（不是对话消息，而是 UI 通知）
- `elicitation`：MCP 引出请求队列

消息类型在 `src/types/message.js`（运行时文件，非 `.ts`）中定义：
- `UserMessage`、`AssistantMessage`、`SystemMessage`、`ProgressMessage`、`AttachmentMessage`
- `SystemLocalCommandMessage`：在 API 边界剥离的仅 UI 消息

## 权限状态

权限状态是 AppState 中最复杂的部分，围绕 `toolPermissionContext`：

```typescript
type ToolPermissionContext = {
  mode: PermissionMode                         // 'default' | 'plan' | 'bypassPermissions' | 'auto' | 'bubble' | ...
  additionalWorkingDirectories: Map<string, AdditionalWorkingDirectory>
  alwaysAllowRules: ToolPermissionRulesBySource  // { userSettings?: string[], projectSettings?: string[], ... }
  alwaysDenyRules: ToolPermissionRulesBySource
  alwaysAskRules: ToolPermissionRulesBySource
  isBypassPermissionsModeAvailable: boolean
  shouldAvoidPermissionPrompts?: boolean
  awaitAutomatedChecksBeforeDialog?: boolean
  prePlanMode?: PermissionMode                   // 进入计划模式前暂存的模式
}
```

### 权限模式
- **`default`**：在每个工具使用前请求权限
- **`plan`**：计划模式 — 仅允许规划，无工具执行
- **`bypassPermissions`**：自动批准所有工具（需要 `--dangerously-skip-permissions`）
- **`auto`**：AI 分类器决定允许还是询问（功能标志）
- **`bubble`**：用于子代理的内部模式

### 权限决策流程（`src/hooks/useCanUseTool.tsx`）
1. 工具通过 `canUseTool(tool, input, toolUseContext, ...)` 请求权限
2. `createPermissionContext()` 构建带所有决策方法的冻结上下文对象
3. `hasPermissionsToUseTool()` 根据当前模式检查规则
4. 结果是 `allow`、`deny` 或 `ask`：
   - **allow**：工具立即执行
   - **deny**：工具被拒绝并附带原因
   - **ask**：进入用户交互的权限队列
5. 对于 `ask` 决策，流程通过处理程序路由：
   - `handleCoordinatorPermission()`：协调器 worker 检查
   - `handleSwarmWorkerPermission()`：swarm 成员检查
   - `handleInteractivePermission()`：交互式 UI 提示
   - Bash 分类器：安全命令的异步自动批准

### 权限持久化
当用户永久允许/拒绝时，创建 `PermissionUpdate` 对象：
```typescript
type PermissionUpdate =
  | { type: 'addRules', destination, rules, behavior }
  | { type: 'replaceRules', destination, rules, behavior }
  | { type: 'removeRules', destination, rules, behavior }
  | { type: 'setMode', destination, mode }
  | { type: 'addDirectories', destination, directories }
  | { type: 'removeDirectories', destination, directories }
```

## 工具状态

工具执行状态通过 `ToolUseContext`（`src/Tool.ts:158-250`）管理，提供：
- `options`：配置（命令、模型、工具、MCP 客户端等）
- `abortController`：用于取消工具执行
- `getAppState()` / `setAppState()`：访问根状态
- `setToolJSX()`：用于渲染工具 UI
- `addNotification()`：用于工具级通知
- `appendSystemMessage()`：用于注入系统消息
- `updateFileHistoryState()` / `updateAttributionState()`：副作用更新器

`useCanUseTool` hook 创建一个包装权限检查管道的 `CanUseToolFn`。它返回一个解析用户（或分类器/hook）做出决策的 `Promise<PermissionDecision>`。

## 会话状态

会话状态跨越多个 AppState 字段：

### 桥接状态（远程控制）
`useReplBridge` hook（`src/hooks/useReplBridge.tsx`）管理始终开启的桥接连接：
- `replBridgeEnabled`：用户切换桥接
- `replBridgeConnected`：传输已启动
- `replBridgeSessionActive`：用户在 claude.ai 上已连接
- `replBridgeReconnecting`：处于退避重试中
- `replBridgeConnectUrl` / `replBridgeSessionUrl`：显示的 URL
- `replBridgePermissionCallbacks`：双向权限检查回调
- `replBridgeError`：显示的错误消息

该 hook 处理：
1. **初始化**：`initReplBridge` 的动态导入、OAuth 检查、环境注册
2. **消息转发**：在新消息到达时写入桥接
3. **入站消息**：将 claude.ai 消息注入 REPL 队列
4. **状态映射**：将桥接生命周期事件（`ready`、`connected`、`reconnecting`、`failed`）映射到 AppState
5. **权限委托**：将远程权限请求路由到 CCR 进行本地审批
6. **拆卸**：在禁用时清理，处理带拆卸承诺链的快速切换
7. **故障恢复**：连续失败计数器与保险丝（3 次失败 → 自动禁用）

### Swarm/团队状态
- `teamContext`：团队名称、文件路径、lead agent ID、队友注册表
- `agentNameRegistry`：代理名称到 AgentId 的映射用于路由
- `inbox`：来自队友的消息
- `workerSandboxPermissions`：worker 网络访问批准请求队列
- `pendingWorkerRequest` / `pendingSandboxRequest`：活动权限请求

### 任务状态
- `tasks`：按键任务 ID 键控的 `TaskState` 对象字典
- `foregroundedTaskId`：其消息显示在主视图中的任务
- `viewingAgentTaskId`：在队友 transcript 模式下查看的任务

任务状态管理助手（`src/state/teammateViewHelpers.ts`）：
- `enterTeammateView(taskId, setAppState)`：切换到查看队友的 transcript
- `exitTeammateView(setAppState)`：返回到 leader 的视图
- `stopOrDismissAgent(taskId, setAppState)`：中止运行中的代理或关闭终端代理

## 关键算法

### 存储相等性检查
```typescript
if (Object.is(next, prev)) return  // 跳过通知
```
使用 `Object.is` 进行引用相等性 — 如果更新器返回相同引用，则不触发监听器。这使得在更新器中进行无操作更改的早期返回成为可能。

### 通知队列处理
```typescript
export function getNext(queue: Notification[]): Notification | undefined {
  if (queue.length === 0) return undefined
  return queue.reduce((min, n) =>
    PRIORITIES[n.priority] < PRIORITIES[min.priority] ? n : min
  )
}
```
从队列中选择最高优先级通知（最低优先级数字）。

### 推测状态
`SpeculationState` 类型支持在用户仍在阅读时预计算 AI 响应：
```typescript
type SpeculationState =
  | { status: 'idle' }
  | {
      status: 'active'
      id: string
      abort: () => void
      messagesRef: { current: Message[] }     // 可变 ref — 避免数组展开
      writtenPathsRef: { current: Set<string> }
      boundary: CompletionBoundary | null
      // ...
    }
```
使用可变 refs（`messagesRef`、`writtenPathsRef`）来避免在推测期间每条新消息的 O(n) 数组复制。

### 权限模式转换
模式更改通过 `transitionPermissionMode()` 处理：
- 进入计划模式时暂存先前模式（`prePlanMode`）
- 退出计划模式时恢复先前模式
- 允许 `auto` 模式前的自动模式门控检查
- 旁路权限可用性检查

## 依赖

### 内部
- `src/state/store.ts` — 最小的外部存储实现
- `src/state/AppStateStore.ts` — AppState 类型和默认状态工厂
- `src/state/AppState.tsx` — React 提供者和 hooks
- `src/state/onChangeAppState.ts` — 副作用观察器
- `src/state/selectors.ts` — 计算状态选择器
- `src/state/teammateViewHelpers.ts` — 队友视图状态转换
- `src/types/permissions.ts` — 权限类型定义
- `src/Tool.ts` — 工具上下文和权限上下文类型
- `src/utils/settings/settings.ts` — 设置管理
- `src/utils/config.ts` — 全局配置持久化
- `src/utils/permissions/PermissionMode.ts` — 权限模式工具

### 外部
- `react` — `useSyncExternalStore`、`useContext`、`useState`、`useEffect`、`useLayoutEffect`、`useCallback`、`useMemo`、`useRef`
- `react/compiler-runtime` — React 编译器记忆化（`_c`）
- `bun:bundle` — 功能标志系统（`feature()`）
- `lodash-es/memoize.js` — 上下文函数的记忆化

## 数据流

```
CLI 参数 / 设置文件
        │
        ▼
  getDefaultAppState() ────► createStore(initialState, onChangeAppState)
        │                              │
        ▼                              ▼
  AppStateProvider              setState(updater)
        │                              │
        │                        ┌─────┴─────┐
        │                        ▼           ▼
        │                  onChangeAppState  listeners
        │                        │           │
        │                   ┌────┴────┐      │
        │                   ▼         ▼      ▼
        │              持久化    同步 CCR  useSyncExternalStore
        │              到磁盘   / SDK          │
        │                                        ▼
        └────────────────────────────────── React 组件
                                                   │
                                             useAppState(selector)
                                                   │
                                             选择性重新渲染
```

1. **初始化**：`getDefaultAppState()` 创建初始状态，`createStore()` 包装它
2. **提供者**：`AppStateProvider` 创建一次存储并通过上下文提供
3. **更新**：组件调用 `useSetAppState()` 获取 `setState`，然后用更新器函数调用它
4. **存储**：存储运行更新器，检查 `Object.is`，触发 `onChangeAppState`，然后通知监听器
5. **副作用**：`onChangeAppState` 持久化更改，同步到 CCR/SDK，清除缓存
6. **订阅**：`useAppState(selector)` 使用 `useSyncExternalStore` 订阅 — 仅在所选值更改时重新渲染

## 集成点

### 主入口（`main.tsx`）
- 使用 CLI 参数创建初始 `AppState`
- 在选项变更前计算 `kairosEnabled`（助手模式）
- 从 CLI 提示或计划模式退出设置 `initialMessage`

### 工具系统
- `ToolUseContext` 将 `getAppState()` 和 `setAppState()` 带到每个工具
- `useCanUseTool` hook 创建权限检查管道
- 工具更新 AppState 以进行进度、JSX 渲染和通知

### 查询引擎
- 消息通过查询引擎流动但存储在 REPL 本地状态中
- 桥接通过 `useReplBridge` 的消息监视效果转发消息
- 系统/初始化消息在连接时通过桥接发送

### 设置系统
- `useSettingsChange` hook 监视设置文件更改
- `applySettingsChange` 在修改设置时更新 AppState
- `onChangeAppState` 将 AppState 更改持久化回设置

### CCR（Claude Code Remote）
- `onChangeAppState` 通过 `notifySessionMetadataChanged()` 同步权限模式更改
- `useReplBridge` 维护用于双向通信的 WebSocket 连接
- 桥接权限回调将远程权限请求路由到本地 UI

### SDK
- `notifyPermissionModeChanged()` 触发 SDK 状态流更新
- SDK 模式更改通过 `externalMetadataToAppState()` 在 worker 重启时应用

## 相关模块
- [Main Entrypoint](./main-entrypoint.md)
- [Tool System](./tool-system.md)
- [Query Engine](./query-engine.md)
