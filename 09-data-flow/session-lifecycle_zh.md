# 会话生命周期

## 目的

追踪 Claude Code 会话的完整生命周期 — 从进程启动，通过初始化、对话轮次、上下文管理（压缩）和会话终止。

## 位置

主要来源：
- `restored-src/src/main.tsx` — 进程入口、初始化、命令路由
- `restored-src/src/state/AppState.tsx` / `AppStateStore.ts` — 会话状态管理
- `restored-src/src/QueryEngine.ts` — 每对话轮次管理
- `restored-src/src/history.ts` — 命令历史持久化
- `restored-src/src/assistant/sessionHistory.ts` — 远程会话历史获取
- `restored-src/src/services/compact/compact.ts` — 对话压缩
- `restored-src/src/services/compact/autoCompact.ts` — 自动压缩触发器
- `restored-src/src/services/compact/prompt.ts` — 压缩提示构建
- `restored-src/src/services/compact/postCompactCleanup.ts` — 压缩后清理

---

## 会话生命周期状态

```
进程启动
    │
    ▼
┌─────────────────────────────────────────────┐
│ 1. BOOTSTRAP                                │
│   - MDM/raw 读取预取                         │
│   - Keychain 预取                           │
│   - 设置加载（用户、项目、策略）               │
│   - 迁移检查                                │
└─────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────┐
│ 2. INITIALIZATION                           │
│   - init() — 认证、遥测、配置                │
│   - 权限上下文设置                           │
│   - 工具注册表构建                           │
│   - MCP 服务器连接                           │
│   - 插件加载                                │
│   - 技能发现                                │
└─────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────┐
│ 3. SESSION START                            │
│   - 会话 ID 生成                            │
│   - Transcript 文件创建                     │
│   - SessionStart 钩子                       │
│   - AppState store 创建                     │
└─────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────┐
│ 4. CONVERSATION TURNS (loop)                │
│   - 用户输入 → QueryEngine.submitMessage    │
│   - API 调用 → 流式响应                     │
│   - 工具执行（如需要）                       │
│   - Transcript 持久化                      │
│   - 自动压缩检查                            │
│   - 结果 yield                             │
└─────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────┐
│ 5. CONTEXT MANAGEMENT                       │
│   - 令牌使用跟踪                            │
│   - 自动压缩触发                            │
│   - 手动 /compact 命令                      │
│   - 会话内存压缩                            │
│   - 部分压缩（消息选择器）                   │
│   - Snip 压缩（HISTORY_SNIP 功能）           │
└─────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────┐
│ 6. SESSION TERMINATION                      │
│   - 用户退出 / 中断                         │
│   - 优雅关闭                               │
│   - 最终 Transcript 刷新                    │
│   - 历史持久化                              │
│   - 清理注册表执行                          │
└─────────────────────────────────────────────┘
```

---

## 阶段 1：Bootstrap

### 早期启动（< 100ms）

```
main.tsx 模块评估
  │
  ├─► profileCheckpoint('main_tsx_entry')
  │
  ├─► startMdmRawRead()        — 并行 MDM 子进程（macOS）
  ├─► startKeychainPrefetch()  — 并行 keychain 读取（macOS）
  │
  └─► 导入所有模块（约 135ms 并行加载）
```

### Pre-Command 设置

```
main() 函数
  │
  ├─► 安全：NoDefaultCurrentDirectoryInExePath = '1' (Windows)
  ├─► 信号处理器：SIGINT、exit
  ├─► URL/深层链接处理（cc://、lodestone://）
  ├─► 特殊模式检测：
  │    ├─► `claude assistant [sessionId]` — assistant 模式
  │    ├─► `claude ssh <host> [dir]` — SSH 远程模式
  │    └─► `claude open <url>` — 直接连接模式
  │
  ├─► 交互 vs 非交互检测
  │    ├─► -p/--print 标志 → 非交互（SDK 模式）
  │    ├─► --init-only 标志 → 非交互
  │    ├─► --sdk-url 标志 → 非交互
  │    └─► !process.stdout.isTTY → 非交互
  │
  ├─► 客户端类型确定
  │    ├─► cli、sdk-cli、sdk-typescript、sdk-python
  │    ├─► github-action、claude-vscode、claude-desktop
  │    └─► remote、local-agent
  │
  └─► 急切设置加载（--settings 标志）
```

---

## 阶段 2：初始化

### Commander Hook (preAction)

```
program.hook('preAction')
  │
  ├─► await ensureMdmSettingsLoaded()
  ├─► await ensureKeychainPrefetchCompleted()
  │
  ├─► init()                     — 核心初始化
  │    ├─► 加载设置（用户、项目、本地、策略）
  │    ├─► 初始化遥测
  │    ├─► 检查认证状态
  │    └─► 应用环境变量
  │
  ├─► process.title = 'claude'
  ├─► initSinks()                — 附加日志接收器
  ├─► runMigrations()            — 应用配置迁移
  │
  ├─► loadRemoteManagedSettings()  — 企业设置（非阻塞）
  ├─► loadPolicyLimits()          — 策略限制（非阻塞）
  │
  └─► uploadUserSettings()        — 设置同步（非阻塞、功能门控）
```

### 延迟预取（首次渲染后）

```
startDeferredPrefetches()
  │
  ├─► initUser()                 — 用户信息
  ├─► getUserContext()           — 用户上下文
  ├─► prefetchSystemContextIfSafe() — git 状态（仅在信任时）
  ├─► getRelevantTips()          — 上下文提示
  ├─► countFilesRoundedRg()      — 用于上下文估算的文件计数
  ├─► initializeAnalyticsGates() — 功能标志
  ├─► prefetchOfficialMcpUrls()  — MCP 注册表
  ├─► refreshModelCapabilities() — 模型能力
  ├─► settingsChangeDetector.initialize() — 设置热重载
  └─► skillChangeDetector.initialize() — 技能热重载
```

---

## 阶段 3：会话启动

### QueryEngine 创建

```
new QueryEngine(config)
  │
  ├─► config: {
  │     cwd、tools、commands、mcpClients、agents、
  │     canUseTool、getAppState、setAppState、
  │     initialMessages、readFileCache、...
  │   }
  │
  ├─► mutableMessages = initialMessages ?? []
  ├─► abortController = config.abortController ?? createAbortController()
  ├─► permissionDenials = []
  ├─► totalUsage = EMPTY_USAGE
  └─► discoveredSkillNames = new Set()
```

### 会话状态（AppState）

```
getDefaultAppState()
  │
  └─► 返回完整的初始状态：
       ├─► settings: SettingsJson
       ├─► toolPermissionContext: ToolPermissionContext
       ├─► mainLoopModel: ModelSetting
       ├─► mcp: { clients、tools、commands、resources }
       ├─► plugins: { enabled、disabled、errors }
       ├─► tasks: { [taskId]: TaskState }
       ├─► todos: { [agentId]: TodoList }
       ├─► notifications: { current、queue }
       ├─► speculation: SpeculationState
       ├─► activeOverlays: Set<string>
       └─► ...（60+ 字段）
```

---

## 阶段 4：对话轮次

### 轮次生命周期

```
QueryEngine.submitMessage(prompt)
  │
  ├─► 1. 清除每轮跟踪
  │    └─► discoveredSkillNames.clear()
  │
  ├─► 2. 构建系统提示
  │    ├─► fetchSystemPromptParts()
  │    │    ├─► 默认系统提示（或自定义）
  │    │    ├─► 用户上下文（OS、shell、git 等）
  │    │    ├─► 系统上下文（工作目录等）
  │    │    └─► 内存机制提示（如启用）
  │    └─► asSystemPrompt([...parts])
  │
  ├─► 3. 处理用户输入
  │    ├─► processUserInput()
  │    │    ├─► 解析斜杠命令
  │    │    ├─► 解析附件
  │    │    └─► 构建消息数组
  │    └─► 推送到 mutableMessages
  │
  ├─► 4. 记录 transcript（API 之前）
  │    └─► recordTranscript(messages)
  │
  ├─► 5. Yield 系统 init 消息
  │    └─► buildSystemInitMessage(tools、mcp、model、mode、...)
  │
  ├─► 6. 进入 query() 循环
  │    │
  │    ├─► normalizeMessagesForAPI()
  │    ├─► queryModelWithStreaming()
  │    │    ├─► 流式事件（内容块）
  │    │    ├─► 跟踪使用量
  │    │    └─► 处理错误/重试
  │    │
  │    ├─► 对于每个 tool_use：
  │    │    ├─► canUseTool() → 权限检查
  │    │    ├─► tool.call() → 执行
  │    │    └─► 将 tool_result 作为用户消息注入
  │    │
  │    └─► 继续直到 stop_reason === 'end_turn'
  │
  ├─► 7. 检查终止条件
  │    ├─► maxTurns 达到 → 错误结果
  │    ├─► maxBudgetUsd 超出 → 错误结果
  │    └─► structured output 重试超出 → 错误结果
  │
  └─► 8. Yield 最终结果消息
       └─► { type: 'result', subtype: 'success'|'error_*', ... }
```

---

## 阶段 5：上下文管理

### 令牌跟踪

```
每个 API 响应：
  │
  ├─► message_start → 重置 currentMessageUsage
  ├─► message_delta → 累积使用量
  └─► message_stop → 添加到 totalUsage
  │
  ▼
totalUsage = {
  input_tokens、output_tokens、
  cache_read_input_tokens、cache_creation_input_tokens
}
```

### 自动压缩触发

```
每轮之后：
  │
  ▼
autoCompactIfNeeded(messages, context, cacheSafeParams, querySource)
  │
  ├─► 检查抑制条件：
  │    ├─► DISABLE_COMPACT 环境变量
  │    ├─► DISABLE_AUTO_COMPACT 环境变量
  │    ├─► userConfig.autoCompactEnabled === false
  │    ├─► querySource === 'compact'（防止递归）
  │    ├─► querySource === 'session_memory'（防止死锁）
  │    ├─► REACTIVE_COMPACT 功能标志
  │    └─► CONTEXT_COLLAPSE 功能（活动时抑制）
  │
  ├─► 计算令牌计数
  │    └─► tokenCountWithEstimation(messages) - snipTokensFreed
  │
  ├─► 计算阈值
  │    ├─► effectiveContextWindow = contextWindow - MAX_OUTPUT_TOKENS_FOR_SUMMARY
  │    ├─► autoCompactThreshold = effectiveWindow - AUTOCOMPACT_BUFFER_TOKENS
  │    │    └─► AUTOCOMPACT_BUFFER_TOKENS = 13,000
  │    └─► 检查：tokenCount >= autoCompactThreshold
  │
  └─► 如果阈值超出：
       ├─► 先尝试会话内存压缩
       │    └─► trySessionMemoryCompaction()
       │
       └─► 如果没有会话内存压缩：
            └─► compactConversation()
```

### 压缩过程

```
compactConversation(messages, context, cacheSafeParams, ...)
  │
  ├─► 1. 压缩前钩子
  │    └─► executePreCompactHooks()
  │
  ├─► 2. 构建压缩提示
  │    ├─► NO_TOOLS_PREAMBLE（关键：仅用文本响应）
  │    ├─► BASE_COMPACT_PROMPT（9 部分摘要结构）
  │    ├─► 自定义指令（如有）
  │    └─► NO_TOOLS_TRAILER
  │
  ├─► 3. 流式传输压缩摘要
  │    ├─► 尝试分叉代理（缓存共享）
  │    │    └─► runForkedAgent({ maxTurns: 1, skipCacheWrite: true })
  │    └─► 回退：常规流式传输
  │         └─► queryModelWithStreaming()
  │
  ├─► 4. 处理摘要
  │    ├─► 剥离 <analysis> 部分
  │    ├─► 提取 <summary> 部分
  │    └─► formatCompactSummary()
  │
  ├─► 5. 清除缓存
  │    ├─► context.readFileState.clear()
  │    └─► context.loadedNestedMemoryPaths?.clear()
  │
  ├─► 6. 生成压缩后附件
  │    ├─► 文件附件（最近读取的文件）
  │    ├─► 异步代理附件
  │    ├─► 计划附件（如在计划模式）
  │    ├─► 技能附件（如使用了技能）
  │    ├─► 延迟工具增量
  │    ├─► 代理列表增量
  │    └─► MCP 指令增量
  │
  ├─► 7. 会话启动钩子
  │    └─► processSessionStartHooks('compact')
  │
  ├─► 8. 创建边界标记
  │    └─► createCompactBoundaryMessage('auto'|'manual', tokenCount, ...)
  │
  ├─► 9. 压缩后钩子
  │    └─► executePostCompactHooks()
  │
  ├─► 10. 构建压缩后消息
  │     └─► [boundaryMarker, ...summaryMessages, ...attachments, ...hookResults]
  │
  └─► 11. 压缩后清理
       └─► runPostCompactCleanup()
```

### 令牌警告阈值

```
calculateTokenWarningState(tokenUsage, model)
  │
  ├─► warningThreshold = threshold - WARNING_THRESHOLD_BUFFER_TOKENS (20,000)
  ├─► errorThreshold = threshold - ERROR_THRESHOLD_BUFFER_TOKENS (20,000)
  ├─► blockingLimit = effectiveWindow - MANUAL_COMPACT_BUFFER_TOKENS (3,000)
  │
  └─► 返回：
       ├─► percentLeft
       ├─► isAboveWarningThreshold
       ├─► isAboveErrorThreshold
       ├─► isAboveAutoCompactThreshold
       └─► isAtBlockingLimit
```

---

## 阶段 6：会话终止

### 优雅关闭

```
用户退出（Ctrl+C、/exit 或进程信号）
  │
  ▼
gracefulShutdown()
  │
  ├─► 中止进行中的 API 调用
  ├─► 等待待处理的 transcript 刷新
  ├─► 刷新命令历史
  │    └─► flushPromptHistory()
  │
  ├─► 执行清理注册表
  │    └─► registerCleanup() 回调
  │
  ├─► 关闭 MCP 连接
  ├─► 终止子进程
  └─► 重置终端光标
```

### 历史持久化

```
addToHistory(command)
  │
  ├─► 如果 CLAUDE_CODE_SKIP_PROMPT_HISTORY 则跳过
  ├─► 首次使用时注册清理
  ├─► 存储粘贴内容（内联或哈希引用）
  ├─► 推送到 pendingEntries
  └─► flushPromptHistory()（异步）
       │
       ├─► 获取 history.jsonl 上的锁
       ├─► 写入 JSON 行
       └─► 释放锁

removeLastFromHistory()
  └─► 撤销最新条目（用于 Esc 中断倒回）
```

### 远程会话历史

```
fetchLatestEvents(ctx, limit)
  │
  ├─► GET /v1/sessions/{sessionId}/events
  │    └─► ?limit=100&anchor_to_latest=true
  │
  └─► 返回 HistoryPage { events, firstId, hasMore }

fetchOlderEvents(ctx, beforeId, limit)
  └─► GET /v1/sessions/{sessionId}/events
       └─► ?limit=100&before_id={beforeId}
```

---

## 会话中断和恢复

### 中断处理

```
用户按 Ctrl+C 或 Esc
  │
  ▼
abortController.abort()
  │
  ├─► 进行中的 API 调用取消
  ├─► 运行中的工具中断
  │    └─► interruptBehavior() 决定：
  │         ├─► 'cancel' → 丢弃结果
  │         └─► 'block' → 保持运行
  │
  ├─► Transcript 保留到中断点
  └─► 会话可以用 --resume <sessionId> 恢复
```

### 会话恢复

```
--resume <sessionId> 或 --continue
  │
  ├─► 从磁盘加载 transcript
  ├─► processResumedConversation()
  │    ├─► 重建消息链
  │    ├─► 恢复文件状态缓存
  │    └─► 重新注入系统上下文
  │
  └─► 从最后一条 assistant 消息继续
```

---

## 关键配置

### 环境变量

| 变量 | 用途 |
|----------|---------|
| `DISABLE_COMPACT` | 禁用所有压缩 |
| `DISABLE_AUTO_COMPACT` | 仅禁用自动压缩 |
| `CLAUDE_AUTOCOMPACT_PCT_OVERRIDE` | 覆盖自动压缩阈值（%） |
| `CLAUDE_CODE_AUTO_COMPACT_WINDOW` | 覆盖上下文窗口大小 |
| `CLAUDE_CODE_BLOCKING_LIMIT_OVERRIDE` | 覆盖阻塞限制 |
| `CLAUDE_CODE_EAGER_FLUSH` | 强制每条消息后刷新 transcript |
| `CLAUDE_CODE_SKIP_PROMPT_HISTORY` | 跳过命令历史记录 |

### 令牌缓冲区

| 常量 | 值 | 用途 |
|----------|-------|---------|
| `MAX_OUTPUT_TOKENS_FOR_SUMMARY` | 20,000 | 为压缩输出保留 |
| `AUTOCOMPACT_BUFFER_TOKENS` | 13,000 | 自动压缩触发前的缓冲区 |
| `WARNING_THRESHOLD_BUFFER_TOKENS` | 20,000 | 警告阈值缓冲区 |
| `ERROR_THRESHOLD_BUFFER_TOKENS` | 20,000 | 错误阈值缓冲区 |
| `MANUAL_COMPACT_BUFFER_TOKENS` | 3,000 | 手动压缩的最小缓冲区 |
| `MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES` | 3 | 失败压缩的断路器 |

---

## 集成点

| 组件 | 在会话生命周期中的角色 |
|-----------|--------------------------|
| `main.tsx` | 进程入口、初始化、模式检测 |
| `AppState` | 中央会话状态存储 |
| `QueryEngine` | 每对话轮次管理 |
| `compact.ts` | 对话压缩 |
| `autoCompact.ts` | 自动压缩触发器 |
| `history.ts` | 命令历史持久化 |
| `sessionHistory.ts` | 远程会话事件获取 |
| `cleanupRegistry` | 优雅关闭协调 |

## 相关文档

- [Message Flow](./message-flow.md)
- [Permission Flow](./permission-flow.md)
- [Tool Execution Flow](./tool-execution-flow.md)
- [Compaction Service](../04-services/compact-service.md)
- [State Management](../01-core-modules/state-management.md)
