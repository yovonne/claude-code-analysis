# 引导

## 目的

引导模块是应用程序的 DAG（有向无环图）叶子——它持有所有全局可变状态，并为会话级数据提供单一真相来源。每个其他模块都从引导导入，但引导不从任何模块导入（除了通过显式旁路的 `src/utils/crypto.js`）。它定义 `State` 类型、一个单例 `STATE` 对象，以及约 120 个控制访问各个状态字段的 getter/setter 函数。

## 位置

`restored-src/src/bootstrap/state.ts`

## 关键导出

### 类型

- `State`: 完整的应用程序状态结构（约 100 个字段，涵盖会话身份、成本、持续时间、遥测、模型设置、代理颜色、hooks、技能、cron 任务、beta 头闩锁等）
- `ChannelEntry`: 带市场信息的已注册通道插件/服务器的联合类型
- `AttributedCounter`: 用于 OpenTelemetry 属性计数器的接口
- `SessionCronTask`: 从不持久化到磁盘的内存中 cron 任务类型
- `InvokedSkillInfo`: 跟踪会话期间调用的技能，以便在压缩后保留

### 核心函数

- `getSessionId()`: 返回当前会话 UUID
- `regenerateSessionId()`: 创建新会话 ID，可选择将当前的设置为父级
- `switchSession(sessionId, projectDir)`: 原子性地同时切换会话 ID 和项目目录
- `onSessionSwitch`: 用于会话更改通知的信号订阅
- `getProjectRoot()`: 返回稳定的项目根目录（中期工作树更改不会更新）
- `getOriginalCwd()`: 返回启动时的原始工作目录

### 成本和持续时间跟踪

- `addToTotalCostState()`: 累积 API 成本和按模型的使用量
- `addToTotalDurationState()`: 累积带/不带重试的 API 持续时间
- `addToToolDuration()`: 累积每轮的工执行持续时间
- `addToTurnHookDuration()`: 累积每轮的 hook 执行持续时间
- `addToTurnClassifierDuration()`: 累积每轮的分类器持续时间
- `getTotalDuration()`: 返回自会话开始以来的挂钟时间
- `snapshotOutputTokensForTurn()`: 在轮次开始时捕获输出 token 计数用于每轮预算

### 遥测状态

- `setMeter()`: 初始化 OpenTelemetry meter 和所有计数器（会话、LOC、PR、提交、成本、token、代码编辑决策、活跃时间）
- `setLoggerProvider()` / `setEventLogger()`: OTel 日志状态
- `setMeterProvider()` / `setTracerProvider()`: OTel 提供者状态
- `getStatsStore()`: 返回统计观察存储

### 模型和功能状态

- `getMainLoopModelOverride()`: 返回 `--model` CLI 标志覆盖
- `setMainLoopModelOverride()`: 设置模型覆盖
- `getSdkBetas()` / `setSdkBetas()`: SDK 提供的 beta 功能标志
- `getKairosActive()` / `setKairosActive()`: KAIROS（Ant 内部）功能状态

### 会话生命周期

- `setSessionTrustAccepted()`: 会话级信任标志（不持久化）
- `setSessionPersistenceDisabled()`: 禁用会话持久化到磁盘
- `hasExitedPlanModeInSession()` / `setHasExitedPlanMode()`: 跟踪计划模式退出
- `handlePlanModeTransition()`: 管理计划模式退出附加标志
- `handleAutoModeTransition()`: 管理自动模式退出附加标志

### Beta 头闩锁

四个"粘性开启"闩锁防止在会话中期切换功能时提示缓存失效：

- `afkModeHeaderLatched`: 自动模式 beta 头
- `fastModeHeaderLatched`: 快速模式 beta 头
- `cacheEditingHeaderLatched`: 缓存编辑 beta 头
- `thinkingClearLatched`: 思考清除 beta 头（空闲 >1 小时后触发）

### 滚动排水

- `markScrollActivity()`: 标记发生了滚动事件（150ms 防抖）
- `getIsScrollDraining()`: 在滚动排水期间返回 true
- `waitForScrollIdle()`: 滚动期间昂贵操作的异步屏障

### Hook 管理

- `registerHookCallbacks()`: 将 SDK/插件 hook 匹配器合并到状态
- `getRegisteredHooks()` / `clearRegisteredHooks()`: 完整的 hook 生命周期
- `clearRegisteredPluginHooks()`: 仅移除插件 hooks，保持 SDK 回调
- `resetSdkInitState()`: 清除 hooks 和 JSON schema

### 技能跟踪

- `addInvokedSkill()`: 记录带有代理作用域键的技能调用
- `getInvokedSkills()`: 返回用于压缩保留的所有调用技能

### 测试

- `resetStateForTests()`: 完全状态重置（仅在 `NODE_ENV=test` 时可调用）
- `resetCostState()`: 重置成本/持续时间计数器用于新会话
- `setCostStateForRestore()`: 从恢复的会话恢复成本状态

## 依赖

### 内部依赖

无 — 这是 DAG 叶子。唯一从 `src/` 的导入是 `src/utils/crypto.js`（通过 eslint-disable 显式允许用于 `randomUUID` 函数）。

### 外部依赖

- `@anthropic-ai/sdk` — `BetaMessageStreamParams` 类型
- `@opentelemetry/api` — `Attributes`、`Meter`、`MetricOptions` 类型
- `@opentelemetry/api-logs` — `logs` 类型
- `@opentelemetry/sdk-logs` — `LoggerProvider` 类型
- `@opentelemetry/sdk-metrics` — `MeterProvider` 类型
- `@opentelemetry/sdk-trace-base` — `BasicTracerProvider` 类型
- `lodash-es/sumBy.js` — Token 聚合
- `fs` — 用于 CWD 解析的 `realpathSync`
- `process` — 用于当前工作目录的 `cwd()`

## 实现细节

### 核心逻辑

该模块使用由 `getInitialState()` 初始化的单一模块作用域 `STATE` 常量。所有变更都通过显式的 setter 函数进行——模块外部没有直接状态访问。这强制执行严格的 API 表面，并使向任何状态变更添加副作用（信号、验证）变得容易。

### 信号系统

`sessionSwitched` 信号（通过 `createSignal` 创建）提供会话更改的发布/订阅机制。调用者通过 `onSessionSwitch` 注册（作为订阅函数导出）。这避免直接将监听器导入引导，保持它作为 DAG 叶子。

### 交互时间批处理

`updateLastInteractionTime()` 使用脏标志模式批处理 `Date.now()` 调用。键入设置 `interactionTimeDirty = true`；`flushInteractionTime()`（在每次 Ink 渲染前调用）执行实际的 `Date.now()` 调用。这避免在每次键入时调用 `Date.now()`。

### 压缩后标志

`markPostCompaction()` / `consumePostCompaction()` 实现单次发射标志模式。压缩后，下一个 API 成功事件被标记为压缩后，以区分缓存未命中和 TTL 过期。标志在消耗后自动重置。

### 滚动排水

最后一次滚动事件后 150ms 防抖窗口阻止昂贵的单次操作（网络、子进程）与滚动帧竞争事件循环。`markScrollActivity()` 重置计时器；`waitForScrollIdle()` 轮询直到标志清除。

## 数据流

```
main.tsx → bootstrap/state.ts (初始化)
    ↓
所有模块 → bootstrap getters (读取状态)
工具 → bootstrap setters (更新成本、持续时间等)
QueryEngine → bootstrap setters (更新模型使用、API 请求等)
```

## 集成点

- **main.tsx**: 在启动时初始化所有状态，设置 meter/logger/tracer 提供者
- **工具执行**: 更新成本、持续时间和行变更计数器
- **QueryEngine**: 记录 API 请求、模型使用和压缩状态
- **会话管理**: `switchSession()` 与并发会话 PID 文件协调
- **遥测**: 所有 OTel 计数器通过引导初始化和访问

## 错误处理

- `resetStateForTests()` 在测试环境外调用时抛出
- `addToInMemoryErrorLog()` 维护有界环形缓冲区（最多 100 个错误）
- 滚动排水使用 `setTimeout(...).unref()` 阻止进程退出阻塞

## 相关模块

- [上下文提供者](./context-providers.md) — 从引导状态读取的 React 上下文
- [Hooks 系统](./hooks-system.md) — Hook 注册流程通过引导状态
- [类型系统](./types-system.md) — `SessionId` 和 `AgentId` 品牌类型在各处使用
