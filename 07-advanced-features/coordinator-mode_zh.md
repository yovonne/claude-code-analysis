# Coordinator 模式

## 目的

Coordinator 模式将 Claude Code 转变为多工作器编排器。协调器不直接执行任务，而是将研究、实现和验证委托给并行异步工作器代理，同时综合结果并与用户通信。

## 位置

`restored-src/src/coordinator/coordinatorMode.ts`

## 主要导出

### 函数

- `isCoordinatorMode()`: 通过 `bun:bundle` 功能标志和 `CLAUDE_CODE_COORDINATOR_MODE` 环境变量检查协调器模式是否激活。
- `matchSessionMode(sessionMode)`: 将当前运行时模式与恢复会话的存储模式对齐。如果不匹配则翻转环境变量并返回警告消息。
- `getCoordinatorUserContext(mcpClients, scratchpadDir)`: 构建注入到协调器用户上下文的 `workerToolsContext` 字符串，列出可用的工作器工具和 MCP 服务器。
- `getCoordinatorSystemPrompt()`: 返回全面的系统提示，教协调器其角色、工具、工作流程和提示编写最佳实践。

### 常量

- `INTERNAL_WORKER_TOOLS`: 工具名称集合（`TeamCreateTool`、`TeamDeleteTool`、`SendMessageTool`、`SyntheticOutputTool`），从工作器工具列表中排除 — 这些是内部编排工具，不面向工作器。

## 依赖

### 内部依赖

- `../constants/tools.js` — 工作器工具枚举的 `ASYNC_AGENT_ALLOWED_TOOLS`
- `../services/analytics/growthbook.js` — Statsig 功能门检查
- `../tools/*/` — 各工具名称常量（AgentTool、BashTool 等）
- `../utils/envUtils.js` — 环境变量解析的 `isEnvTruthy()`

### 外部依赖

- `bun:bundle` — Bun 的构建时切换功能标志系统

## 实现细节

### 核心逻辑

协调器模式由双层门控制：

1. **构建时功能标志** — 来自 `bun:bundle` 的 `feature('COORDINATOR_MODE')`
2. **运行时环境变量** — `CLAUDE_CODE_COORDINATOR_MODE`（通过 `isEnvTruthy` 检查）

两者都必须为 true，`isCoordinatorMode()` 才能返回 `true`。这种双门模式允许功能被编译进去但在运行时被禁用。

### 会话模式匹配

恢复会话时，`matchSessionMode()` 将存储的会话模式（`'coordinator' | 'normal'`）与当前运行时状态进行比较。如果不同，它翻转 `process.env.CLAUDE_CODE_COORDINATOR_MODE` 以匹配存储的模式。这确保在协调器模式下恢复的会话实际作为协调器模式运行，无论新进程如何启动。

### 工作器工具上下文

`getCoordinatorUserContext()` 动态构建关于生成的工作器有权访问哪些工具的描述。两种模式：

- **简单模式**（`CLAURE_CODE_SIMPLE=1`）：工作器仅获得 `Bash`、`Read` 和 `Edit`
- **完整模式**：工作器获得所有 `ASYNC_AGENT_ALLOWED_TOOLS` 减去内部编排工具

当可用时，MCP 服务器名称被附加，如果 `tengu_scratch` 功能门启用，则包含 scratchpad 目录信息。

### 系统提示架构

系统提示（`getCoordinatorSystemPrompt()`）是一份全面的指令文档，涵盖：

1. **角色定义** — 协调器 vs 工作器职责
2. **工具清单** — AgentTool、SendMessageTool、TaskStopTool、PR 订阅工具
3. **任务通知格式** — 带 task-id、status、summary、result 和 usage 的 XML 结构化 `<task-notification>` 块
4. **工作流程阶段** — 研究（并行工作器）→ 综合（协调器）→ 实现（工作器）→ 验证（工作器）
5. **并发管理** — 并行 vs 串行执行指南
6. **提示编写标准** — 反模式（"based on your findings"）vs 好模式（具体文件路径、行号）
7. **继续 vs 生成的决策矩阵** — 何时继续现有工作器 vs 生成新的

## 数据流

```
用户请求
    ↓
协调器（isCoordinatorMode=true）
    ↓
┌─────────────────────────────────────┐
│  阶段 1: 研究（并行）                  │
│  ├── 工作器 A: 调查文件                │
│  └── 工作器 B: 调查测试                │
└─────────────────────────────────────┘
    ↓ <task-notification> XML
协调器综合发现
    ↓
┌─────────────────────────────────────┐
│  阶段 2: 实现                        │
│  └── 工作器（继续或新生成）            │
└─────────────────────────────────────┘
    ↓ <task-notification> XML
┌─────────────────────────────────────┐
│  阶段 3: 验证                        │
│  └── 工作器（新生成、独立）            │
└─────────────────────────────────────┘
    ↓ <task-notification> XML
协调器向用户报告
```

## 集成点

- **AgentTool** — 生成工作器的主要机制（`subagent_type: "worker"`）
- **SendMessageTool** — 继续具有后续指令的现有工作器
- **TaskStopTool** — 停止方向错误的工作器
- **会话持久化** — 模式与会话一起存储并通过 `matchSessionMode()` 恢复
- **MCP 服务器** — 工作器工具上下文包括连接的 MCP 服务器名称
- **Scratchpad** — 跨工作器知识的可选共享目录（由 `tengu_scratch` 门控）

## 配置

| 环境变量 | 用途 |
|---|---|
| `CLAURE_CODE_COORDINATOR_MODE` | 运行时切换（`1` = 启用） |
| `CLAURE_CODE_SIMPLE` | 简化的工作器工具集（仅 Bash/Read/Edit） |

| 功能门 | 用途 |
|---|---|
| `COORDINATOR_MODE` | 构建时功能标志 |
| `tengu_scratch` | Scratchpad 目录支持 |

## 错误处理

- **恢复时模式不匹配**: 使用面向用户的警告消息自动校正
- **工作器失败**: 协调器继续相同的工作器（保留错误上下文）或根据失败类型生成新的
- **循环依赖避免**: `isScratchpadGateEnabled()` 从 `utils/permissions/filesystem.ts` 复制门检查以避免导入会创建循环依赖的文件系统模块

## 测试

- `clearBuiltinPlugins()` 模式表明为测试隔离清除注册表
- `matchSessionMode()` 函数是纯函数，可以用各种会话模式组合进行测试

## 相关模块

- [AgentTool](../02-tools/AgentTool.md) — 工作器生成机制
- [SendMessageTool](../02-tools/SendMessageTool.md) — 工作器继续
- [TaskStopTool](../02-tools/TaskStopTool.md) — 工作器终止
- [Skills System](./skills-system.md) — 委托给工作器的技能

## 备注

- 协调器模式是一种高级编排模式 — 协调器从不直接执行工具（PR 订阅除外）；所有代码工作都通过工作器
- 系统提示是代码库中最长的提示之一（约 250 行），因为它必须教 LLM 一个完整的多代理工作流程
- 工作器结果作为带有 `<task-notification>` XML 标签的用户角色消息到达 — 协调器必须将这些与实际用户消息区分开
- "继续 vs 生成"决策由协调器根据上下文重叠分析自行判断
