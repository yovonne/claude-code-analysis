# KAIROS 和 Ant 专用功能门控工具

## 目的

本文档涵盖七个基于功能标志或环境变量有条件地打包到 Claude Code 中的工具。这些工具**不在**公共/外部构建中提供 — 它们要么受 `KAIROS`（助手模式）、`PROACTIVE`（主动模式）、`ANT` 专用构建，要么是实验性标志。了解这些工具揭示了内部 Anthropic 构建的完整能力表面。

## 位置

所有工具注册位于 `restored-src/src/tools.ts`。各个工具源文件位于 `restored-src/src/tools/<ToolName>/`。

---

## 功能标志摘要

| 工具 | 门控 | 门控类型 | 构建影响 |
|------|------|-----------|--------------|
| **SleepTool** | `feature('PROACTIVE') \|\| feature('KAIROS')` | Bun 功能标志 | 外部构建中的 DCE |
| **MonitorTool** | `feature('MONITOR_TOOL')` | Bun 功能标志 | 外部构建中的 DCE |
| **SendUserFileTool** | `feature('KAIROS')` | Bun 功能标志 | 外部构建中的 DCE |
| **PushNotificationTool** | `feature('KAIROS') \|\| feature('KAIROS_PUSH_NOTIFICATION')` | Bun 功能标志 | 外部构建中的 DCE |
| **SubscribePRTool** | `feature('KAIROS_GITHUB_WEBHOOKS')` | Bun 功能标志 | 外部构建中的 DCE |
| **SuggestBackgroundPRTool** | `process.env.USER_TYPE === 'ant'` | 运行时 env（构建时常量） | 外部构建中的 DCE |
| **VerifyPlanExecutionTool** | `process.env.CLAUDE_CODE_VERIFY_PLAN === 'true'` | 运行时 env（构建时常量） | 外部构建中的 DCE |

### 门控机制

**Bun 功能标志**（来自 `bun:bundle` 的 `feature('...')`）：
- 在打包时评估。当为 `false` 时，整个 `require()` 调用和依赖代码被 Bun 的死代码消除（DCE）消除。外部构建附带这些标志关闭，因此工具代码有**零占用**。

**环境变量门控**（`process.env.USER_TYPE === 'ant'`、`process.env.CLAUDE_CODE_VERIFY_PLAN === 'true'`）：
- 在外部构建管道中也是构建时常量。`USER_TYPE` 仅对内部构建设置为 `'ant'`；外部构建设置为 `'external'`。Bun DCE 消除这些分支。

### 为什么这些工具被门控

1. **仅内部功能**：`SuggestBackgroundPRTool` 和 `VerifyPlanExecutionTool` 等工具支持 Anthropic 的内部开发工作流，与外部用户无关。

2. **助手/主动模式**：`SleepTool`、`SendUserFileTool` 和 `PushNotificationTool` 是 KAIROS 助手模式的核心 — 一个自主工作的常在 agent。外部构建没有此模式。

3. **实验/预览功能**：`MONITOR_TOOL`、`KAIROS_GITHUB_WEBHOOKS` 和 `KAIROS_PUSH_NOTIFICATION` 是用于逐步推出和测试的功能标志。

4. **包大小**：每个工具都带有自己的源代码、UI 组件、提示和依赖。DCE 确保外部构建不为未使用的代码付费。

---

## 工具详情

### 1. SleepTool

**工具名称：** `Sleep`

**功能门控：** `feature('PROACTIVE') || feature('KAIROS')`

**位置：** `restored-src/src/tools/SleepTool/prompt.ts`

**目的：** 允许模型在指定持续时间内等待而不占用 shell 进程。对于主动/助手模式至关重要，其中 agent 定期通过 `<tick>` 提示检查工作。

**关键导出：**
- `SLEEP_TOOL_NAME`：`'Sleep'`
- `DESCRIPTION`：`'Wait for a specified duration'`
- `SLEEP_TOOL_PROMPT`：LLM 的说明

**实现：**
恢复的源代码中只有 `prompt.ts` 存在 — 主工具定义（`SleepTool.ts`）缺失。提示指示模型：
- 当被告知休息、空闲或等待某事时使用 Sleep
- 通过查找有用工作来响应 `<tick>` 定期检查
- 与其他工具并发调用 Sleep（非阻塞）
- 首选 Sleep 而不是 `Bash(sleep ...)` 以避免占用 shell 进程
- 在 5 分钟提示缓存过期之间平衡唤醒 API 调用

**集成点：**
- `query.ts:1566` — 检测响应块中的 Sleep 工具使用
- `REPL.tsx:1655` — 跟踪仅 Sleep 处于活跃状态时（允许在睡眠期间用户输入）
- `main.tsx:1864` — 在 `getTools()` 之前激活主动模式，以便 `SleepTool.isEnabled()` 通过
- `handlePromptSubmit.ts:320` — 带 `interruptBehavior: 'cancel'` 的用户输入唤醒睡眠工具
- `constants/prompts.ts:872-886` — 系统提示要求空闲时使用 Sleep（从不叙述等待）
- `classifierDecision.ts:87` — 列为安全 YOLO 允许列表工具
- `mcp/channelNotification.ts:9` — Sleep 轮询 `hasCommandsInQueue()` 并在 MCP 通知到达时在 1 秒内唤醒

**为什么门控：** Sleep 是主动/助手模式的心跳机制。没有主动模式，就没有 `<tick>` 提示和自主 agent 循环 — Sleep 将毫无意义。

**与标准工具的区别：** 与 `Bash(sleep N)` 不同，Sleep 不占用 shell 进程，是并发安全的，并与基于 tick 的主动调度系统集成。每次唤醒触发 API 调用，但受益于提示缓存重用。

---

### 2. MonitorTool

**工具名称：** `Monitor`（从模块名称推断）

**功能门控：** `feature('MONITOR_TOOL')`

**位置：** `restored-src/src/tools/MonitorTool/MonitorTool.js`（不在恢复源代码中）

**目的：** 启用对长时间运行的后台命令（Bash/PowerShell）的监控作为专用监控任务。当 `MONITOR_TOOL` 启用时，后台进程作为 `kind: 'monitor'` 任务生成，优先级为 `'next'` 而不是 `'later'`。

**集成点：**
- `tools.ts:39-40` — 通过 `require()` 条件加载
- `BashTool.tsx:525` — 当 `MONITOR_TOOL` 开启且 `run_in_background` 未设置时，Bash 命令可能成为监控任务
- `PowerShellTool.tsx:361` — PowerShell 相同行为
- `BashTool/prompt.ts:312-320` — 标志开启时提示包含监控特定说明
- `AgentTool/runAgent.ts:849` — Agent 执行检查监控工具可用性
- `LocalShellTask.tsx:129` — `kind === 'monitor'` 创建特殊监控任务类型
- `LocalShellTask.tsx:169` — 监控任务获得 `'next'` 优先级 vs 普通后台任务的 `'later'`
- `tasks.ts:12` — `MonitorMcpTask` — MCP 监控的专用任务类型
- `BackgroundTasksDialog.tsx:117-119` — 监控任务显示的 UI 组件
- `PermissionRequest.tsx:40-41,73` — 监控工具的特殊权限请求 UI
- `query.ts:1551` — 标志开启时查询引擎在不 Sleep 的情况下消耗监控任务

**为什么门控：** 监控模式改变了后台任务的调度和显示方式。它是后台任务系统的实验性增强，可能显著改变用户体验。

**与标准工具的区别：** 标准后台任务以 `'later'` 优先级运行并通过通用任务系统管理。监控任务获得 `'next'` 优先级、专用 UI 组件（`MonitorMcpDetailDialog`）和特殊权限请求处理。查询引擎可以在不需要 Sleep 周期的情况下消耗监控任务。

---

### 3. SendUserFileTool

**工具名称：** 导出为 `SEND_USER_FILE_TOOL_NAME`（值来自 prompt.ts）

**功能门控：** `feature('KAIROS')`

**位置：** `restored-src/src/tools/SendUserFileTool/SendUserFileTool.js`（不在恢复源代码中）

**目的：** 在 KAIROS 助手模式中将文件从 agent 发送到用户。通过 BriefTool（`SendUserMessage`）补充，在助手工作流中提供文件传送的专用渠道。

**集成点：**
- `tools.ts:42-43` — 当 `KAIROS` 启用时条件加载
- `conversationRecovery.ts:67-70` — 加载工具名称用于会话恢复；用于在会话重放期间过滤工具（`line 367`）
- `ToolSearchTool/prompt.ts:15-18,100-101` — 当 KAIROS 开启时工具名称包含在工具搜索结果中
- `Messages.tsx:82,509` — 与 BriefTool 名称组合用于消息渲染：`[BRIEF_TOOL_NAME, SEND_USER_FILE_TOOL_NAME]`

**为什么门控：** 此工具专用于 KAIROS 助手模式的文件传送机制。外部构建没有助手模式，因此该工具将无法运行。

**与标准工具的区别：** 与发送文本消息（带可选附件）的 BriefTool 不同，SendUserFileTool 似乎是专用文件传送渠道。它在 Messages 组件中被视为"brief 工具"变体，表明它类似地渲染但具有文件聚焦语义。

---

### 4. PushNotificationTool

**工具名称：** `PushNotification`（从模块名称推断）

**功能门控：** `feature('KAIROS') || feature('KAIROS_PUSH_NOTIFICATION')`

**位置：** `restored-src/src/tools/PushNotificationTool/PushNotificationTool.js`（不在恢复源代码中）

**目的：** 当助手模式不活跃可见时向用户发送推送通知。使助手能够提醒用户重要事件，即使他们不看终端。

**集成点：**
- `tools.ts:45-48` — 当 KAIROS 或 KAIROS_PUSH_NOTIFICATION 启用时条件加载
- `ConfigTool/supportedSettings.ts:164` — 当功能启用时配置设置包含推送通知选项
- `Settings/Config.tsx:658,672` — 当 KAIROS 标志开启时 UI 标签从"通知"更改为"本地通知"，带有附加配置部分

**为什么门控：** 推送通知需要平台特定集成（OS 通知 API、移动推送服务），仅在助手自主运行时才有意义。`KAIROS_PUSH_NOTIFICATION` 子标志允许独立于完整助手模式推送。

**与标准工具的区别：** 这是唯一与 OS 级通知系统交互的工具，而不是终端 UI。它弥合了常在助手和用户注意力之间的差距，实现异步通信。

---

### 5. SubscribePRTool

**工具名称：** `SubscribePR`（从模块名称推断）

**功能门控：** `feature('KAIROS_GITHUB_WEBHOOKS')`

**位置：** `restored-src/src/tools/SubscribePRTool/SubscribePRTool.js`（不在恢复源代码中）

**目的：** 允许助手订阅 GitHub Pull Request webhook，在 PR 更新、评论或合并时获取实时通知。作为 KAIROS GitHub 集成表面的一部分。

**集成点：**
- `tools.ts:50-51` — 当 `KAIROS_GITHUB_WEBHOOKS` 启用时条件加载

**为什么门控：** GitHub webhook 订阅需要：
- 用于 webhook 传递的公开可达端点
- GitHub 应用/机器人凭证
- 接收和处理 webhooks 的持久助手进程

这些是仅在助手作为持久服务运行的内部 Anthropic 部署中才有意义的基础设施要求。

**与标准工具的区别：** 与标准工具（模型启动调用）不同，SubscribePRTool 设置入站事件流。这是一个订阅工具 — 助手注册兴趣，然后接收异步通知，这可能通过 tick 系统触发进一步的工具调用。

---

### 6. SuggestBackgroundPRTool

**工具名称：** `SuggestBackgroundPR`（从模块名称推断）

**功能门控：** `process.env.USER_TYPE === 'ant'`

**位置：** `restored-src/src/tools/SuggestBackgroundPRTool/SuggestBackgroundPRTool.js`（不在恢复源代码中）

**目的：** 建议或创建后台 Pull Request 作为 Anthropic 内部开发工作流的一部分。使助手能够通过 PR 而非直接文件修改来提出代码更改。

**集成点：**
- `tools.ts:20-23` — 当 `USER_TYPE === 'ant'` 时条件加载
- `tools.ts:216` — 可用时添加到工具池

**为什么门控：** 此工具专用于 Anthropic 的内部 GitHub 工作流。它需要：
- 内部 GitHub 组织访问
- 自定义 PR 创建/管理基础设施
- Anthropic 特定的代码审查流程

外部用户无法访问此基础设施，因此该工具将无法运行。

**与标准工具的区别：** 标准工具直接在工作目录中修改文件。SuggestBackgroundPRTool 在 GitHub API 级别操作，作为副作用创建 PR。它是文件编辑 → 提交 → 推送 → PR 创建流程的更高级别工作流抽象。

---

### 7. VerifyPlanExecutionTool

**工具名称：** `VerifyPlanExecution`（推断；常量：`VERIFY_PLAN_EXECUTION_TOOL_NAME`）

**功能门控：** `process.env.CLAUDE_CODE_VERIFY_PLAN === 'true'`

**位置：** `restored-src/src/tools/VerifyPlanExecutionTool/VerifyPlanExecutionTool.js`（不在恢复源代码中）

**目的：** 验证模型的计划执行是否与预期计划匹配。在复杂多步骤操作期间作为 guardrail，确保 agent 保持正轨且不偏离批准的计划。

**关键导出（来自 constants.js）：**
- `VERIFY_PLAN_EXECUTION_TOOL_NAME` — 工具名称字符串

**集成点：**
- `tools.ts:91-94` — 当 `CLAUDE_CODE_VERIFY_PLAN === 'true'` 时条件加载
- `classifierDecision.ts:37-41,91` — 加载工具名称用于权限分类；添加到安全 YOLO 允许列表
- `hooks/execAgentHook.ts:47` — 注释引用："programmatic construction, since refactored into VerifyPlanExecutionTool"
- `schemas/hooks.ts:136` — 模式注释："has since been refactored into VerifyPlanExecutionTool"
- `attachments.ts:291,3900,3922` — `VERIFY_PLAN_REMINDER_CONFIG` 带 `TURNS_BETWEEN_REMINDERS`；用于在配置的回 合间隔提醒模型关于计划验证
- `messages.ts:4241-4244` — 外部构建的死代码消除注释
- `ExitPlanModePermissionRequest.tsx:366` — 外部构建的 DCE 注释

**为什么门控：** 计划验证是增加 agent 循环开销的实验性安全功能。它在 env var 后门控，以便可以：
- 启用进行测试/验证概念
- 在生产中禁用以减少 API 成本和延迟
- 按会话切换以进行调试 agent 行为

**与标准工具的区别：** 这是一个元工具 — 它不对代码库执行工作，而是根据计划验证 agent 自己的行为。它是安全/guardrail 基础设施的一部分，而不是能力表面的一部分。`VERIFY_PLAN_REMINDER_CONFIG` 表明它定期提示模型在配置的回合间隔进行自我验证。

---

## 这些工具与标准工具的区别

| 方面 | 标准工具 | KAIROS/Ant 专用工具 |
|--------|---------------|----------------------|
| **可用性** | 始终打包 | 通过 DCE 有条件打包 |
| **构建占用** | 存在于所有构建中 | 关闭时为零占用 |
| **目标用户** | 外部 + 内部 | 仅内部 Anthropic |
| **模式依赖** | 可在任何模式下工作 | 通常需要主动/助手模式 |
| **基础设施** | 自包含 | 可能需要外部服务（webhooks、推送 API、GitHub 组织） |
| **权限模型** | 标准权限流程 | 一些有特殊权限 UI（MonitorTool） |
| **执行模型** | 模型启动 | 一些设置异步事件流（SubscribePRTool） |

## 注册模式

所有七个工具在 `tools.ts` 中遵循相同的条件注册模式：

```typescript
// 带延迟 require 的构建时门控
const ToolName = feature('FLAG') || process.env.VAR === 'value'
  ? require('./tools/ToolName/ToolName.js').ToolName
  : null

// 工具池中的条件包含
...(ToolName ? [ToolName] : [])
```

此模式确保：
1. **禁用时零运行时成本** — `null` 在 spread 中是无操作的
2. **禁用时零包成本** — Bun DCE 消除整个 `require()` 分支
3. **延迟加载** — 工具代码仅在工具实际可用时加载
4. **类型安全** — TypeScript 从三元组推断 `ToolName | null`

## 相关模块

- [BriefTool](./BriefTool.md) — 主要用户通信工具，也是 KAIROS 门控
- [AgentTool](./AgentTool.md) — 子 agent 执行，与主动模式集成
- [ScheduleCronTool](../01-core-modules/tool-system.md) — Cron 调度，由 `AGENT_TRIGGERS` 门控
- [tool-system](../01-core-modules/tool-system.md) — 核心工具抽象和注册表
