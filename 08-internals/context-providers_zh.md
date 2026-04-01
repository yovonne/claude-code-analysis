# 上下文提供者

## 目的

React 上下文提供者，向组件树提供共享状态和工具函数。这些上下文实现跨组件通信，无需属性穿透，涵盖通知、模态框、覆盖层、语音状态、邮箱、统计和 FPS 指标。

## 位置

`restored-src/src/context/`

## 关键提供者

### `notifications.tsx` — 通知系统

管理带优先级折叠、失效和超时支持的基于优先级的通知队列。

**类型:**
- `Priority`: `'low' | 'medium' | 'high' | 'immediate'`
- `Notification`: 要么是 `TextNotification`（文本 + 颜色），要么是 `JSXNotification`（React 节点）
- `BaseNotification`: 共享字段 — `key`、`invalidates`、`priority`、`timeoutMs`、`fold`

**关键导出:**
- `useNotifications()`: 返回 `{ addNotification, removeNotification }`
- `getNext(queue)`: 从队列中选择最高优先级通知（immediate > high > medium > low）

**功能:**
- **优先级队列**: 通知按优先级出队，而非 FIFO
- **立即优先级**: 绕过队列——立即显示并清除当前超时
- **折叠**: 具有相同 `key` 的通知可以通过 `fold` 函数合并（如 `Array.reduce`）
- **失效**: 通知可以通过 `key` 使其他通知失效，立即从队列移除
- **超时**: `timeoutMs` 后自动关闭（默认 8000ms）
- **去重**: 阻止具有相同 key 的重复通知

### `modalContext.tsx` — 模态槽上下文

为在模态槽（斜杠命令对话框的底部锚定窗格）内渲染的内容提供尺寸和滚动信息。

**类型 `ModalCtx`:**
- `rows`: 可用内容行
- `columns`: 可用内容列
- `scrollRef`: 模态 ScrollBox 的 ref

**关键导出:**
- `useIsInsideModal()`: 当组件在模态槽内时返回 true
- `useModalOrTerminalSize(fallback)`: 返回模态尺寸或回退到终端尺寸
- `useModalScrollRef()`: 返回模态滚动 ref 用于标签切换时滚动重置

**用例:**
- 抑制顶层框架（Pane 跳过其 Divider）
- 大小选择分页到可用行（非完整终端）
- 通过 ScrollBox keying 在标签切换时重置滚动

### `overlayContext.tsx` — 覆盖层跟踪

跟踪活动覆盖层用于 Escape 键协调。

**关键导出:**
- `useRegisterOverlay(id, enabled?)`: 在挂载时自动注册，在卸载时取消注册
- `useIsOverlayActive()`: 当有任何覆盖层活动时返回 true
- `useIsModalOverlayActive()`: 当有任何模态（非自动完成）覆盖层活动时返回 true

**非模态覆盖层:** `autocomplete` 被排除在模态覆盖层检测之外，这样它不会禁用 TextInput 焦点。

### `promptOverlayContext.tsx` — 浮动提示覆盖层

用于浮动在提示上方内容的端口，逃逸 FullscreenLayout 的 `overflowY:hidden` 裁剪。

**两个通道:**
- **建议数据**: 结构化斜杠命令建议数据（由 PromptInputFooter 写入）
- **对话框节点**: 任意对话框节点如 AutoModeOptInDialog（由 PromptInput 写入）

**关键导出:**
- `usePromptOverlay()`: 返回建议数据
- `usePromptOverlayDialog()`: 返回对话框 ReactNode
- `useSetPromptOverlay(data)`: 注册建议数据（卸载时清除）
- `useSetPromptOverlayDialog(node)`: 注册对话框节点（卸载时清除）

**设计:** 拆分为数据/设置器上下文对，这样写入者永远不会在自己的写入时重新渲染。

### `voice.tsx` — 语音状态

使用 `useSyncExternalStore` 的自定义存储模式管理语音输入状态。

**类型 `VoiceState`:**
- `voiceState`: `'idle' | 'recording' | 'processing'`
- `voiceError`: 错误消息或 null
- `voiceInterimTranscript`: 当前转录
- `voiceAudioLevels`: 用于可视化的音频电平数组
- `voiceWarmingUp`: 预热指示器

**关键导出:**
- `useVoiceState(selector)`: 订阅语音状态的切片（仅在所选值变化时重新渲染）
- `useSetVoiceState()`: 获取状态设置器（稳定引用，从不导致重新渲染）
- `useGetVoiceState()`: 在回调内获取新鲜状态的同步读取器

### `mailbox.tsx` — 邮箱上下文

提供用于代理间通信的单例 `Mailbox` 实例。

**关键导出:**
- `useMailbox()`: 返回邮箱实例（在提供者外使用时抛出）

### `stats.tsx` — 统计存储

带计数器、仪表、直方图和集合的内存指标存储。

**类型 `StatsStore`:**
- `increment(name, value?)`: 添加到计数器
- `set(name, value)`: 设置仪表值
- `observe(name, value)`: 记录直方图观察（水库采样，大小 1024）
- `add(name, value)`: 添加到字符串集合
- `getAll()`: 以扁平记录形式返回所有指标

**直方图功能:**
- 水库采样（Algorithm R）用于有界内存
- 计算 count、min、max、avg、p50、p95、p99
- 在进程退出时刷新到项目配置

**关键导出:**
- `createStatsStore()`: 工厂函数
- `useStats()`: 返回统计存储
- `useCounter(name)`: 返回命名计数器的递增函数
- `useGauge(name)`: 返回命名仪表的设置函数
- `useTimer(name)`: 返回命名计时器的观察函数
- `useSet(name)`: 返回命名集合的添加函数

### `fpsMetrics.tsx` — FPS 指标

提供 FPS 跟踪指标的访问。

**关键导出:**
- `useFpsMetrics()`: 返回 FPS 指标的 getter 函数

### `QueuedMessageContext.tsx` — 队列消息布局

在多代理场景中为队列消息提供布局上下文。

**类型 `QueuedMessageContextValue`:**
- `isQueued`: 对子项始终为 true
- `isFirst`: 这是否是第一个队列消息
- `paddingWidth`: 容器填充的宽度减少

**关键导出:**
- `useQueuedMessage()`: 返回上下文值
- `QueuedMessageProvider`: 用填充和上下文包装子项

## 依赖

### 内部依赖

- `state/AppState.js` — 应用状态存储和设置器
- `utils/config.js` — 项目配置持久化（统计刷新）
- `utils/mailbox.js` — 邮箱实现
- `utils/fpsTracker.js` — FPS 跟踪
- `ink/components/ScrollBox.js` — ScrollBox 处理程序类型
- `ink.js` — 用于布局的 Box 组件

### 外部依赖

- `react` — 上下文创建和 hooks
- `react/compiler-runtime` — React Compiler 优化

## 实现细节

### 存储模式（语音、统计）

语音和统计上下文都使用自定义存储模式而非 React 状态：
- `createStore(initialState)` 创建带 `getState`、`setState`、`subscribe` 的外部存储
- `useSyncExternalStore` 将组件订阅到特定切片
- 设置器是稳定引用，从不导致重新渲染

### 通知队列算法

```
addNotification(notif):
  if priority == 'immediate':
    清除当前超时
    立即显示，重新排队非立即项
    设置新超时
  else:
    检查折叠（相同 key）→ 合并
    检查失效 → 移除失效项
    检查重复 → 跳过
    添加到队列
  processQueue()

processQueue():
  如果当前没有内容:
    从队列获取最高优先级 (getNext)
    设置为当前
    开始超时
```

## 相关模块

- [引导](./bootstrap.md) — 引导状态中存储的统计存储引用
- [Hooks 系统](./hooks-system.md) — 通知 hooks 消费通知上下文
- [Schemas](./schemas.md) — Hook schemas 定义 hook 配置结构
