# AgentTool

## 目的

AgentTool 是在 Claude Code 中生成子代理的主要机制。它支持将复杂的多步骤任务委托给专门的专业代理，这些代理可以同步运行（阻塞父代理）或异步运行（在后台运行）。该工具支持内置代理、自定义用户/项目代理、插件代理，以及实验性的 fork 子代理模式（继承完整父上下文）。

## 位置

`restored-src/src/tools/AgentTool/AgentTool.tsx`

## 关键导出

### 函数

- `AgentTool`：通过 `buildTool()` 构建的主要工具定义 — 处理提示生成、输入验证和代理生成逻辑
- `inputSchema`：定义工具输入参数的 Zod schema（description、prompt、subagent_type、model、run_in_background、name、team_name、mode、isolation、cwd）
- `outputSchema`：定义工具输出的 Zod schema（completed 或 async_launched 状态）

### 类型

- `AgentToolInput`：结合基础 schema 和多代理参数的类型（name、team_name、mode、isolation、cwd）
- `Progress`：`AgentToolProgress` 和 `ShellProgress` 的联合，用于进度事件转发
- `RemoteLaunchedOutput`：远程启动代理的输出类型（仅限 ant）

### 常量

- `AGENT_TOOL_NAME`：`'Agent'` — 当前工具名称
- `LEGACY_AGENT_TOOL_NAME`：`'Task'` — 向后兼容的旧有线名称
- `ONE_SHOT_BUILTIN_AGENT_TYPES`：`Set(['Explore', 'Plan'])` — 运行一次并返回报告的代理

## 依赖

### 内部依赖

- `runAgent.ts` — 核心代理执行引擎（异步生成器，产生消息）
- `loadAgentsDir.ts` — 代理定义加载、解析和类型系统
- `agentColorManager.ts` — 代理的颜色分配和主题映射
- `forkSubagent.ts` — Fork 子代理实验（上下文继承）
- `agentToolUtils.ts` — 共享工具：工具解析、完成、生命周期、分类
- `prompt.ts` — 工具提示生成
- `UI.tsx` — 在终端中渲染代理工具使用/结果的 React 组件
- `LocalAgentTask/LocalAgentTask.tsx` — 任务注册、进度跟踪、通知
- `RemoteAgentTask/RemoteAgentTask.tsx` — 远程代理任务注册（仅限 ant）

### 外部依赖

- `zod/v4` — Schema 验证
- `react` — UI 组件渲染
- `bun:bundle` — 功能标志门控

## 实现细节

### 子代理生成

AgentTool `call()` 方法通过多个决策分支协调代理生成：

#### 1. 队友生成（多代理团队）

当同时提供 `team_name` 和 `name` 时，工具委托给 `spawnTeammate()` 而非创建子代理。这会创建一个单独的 tmux split-pane 代理，可通过 `SendMessage({to: name})` 寻址。

```
team_name + name → spawnTeammate() → teammate_spawned output
```

验证守卫：
- 队友不能生成其他队友（平面名单）
- 进程内队友不能生成后台代理

#### 2. Fork 子代理路径

当省略 `subagent_type` 且 `FORK_SUBAGENT` 功能启用（且不在协调器或非交互模式）时，工具采用 fork 路径：

```
!subagent_type + fork enabled → FORK_AGENT → inherits parent context
```

关键特性：
- 子代理继承父代理的完整对话上下文和系统提示
- 使用 `useExactTools: true` 以获得缓存相同的 API 请求前缀
- `permissionMode: 'bubble'` 将权限提示带到父终端
- `model: 'inherit'` 保持父代理的模型以保持上下文长度对等
- 所有生成都异步运行以实现统一的 `<task-notification>` 交互
- 递归 fork 守卫：fork 子代理不能生成更多 fork（通过消息中的 `FORK_BOILERPLATE_TAG` 检测）

Fork 消息构建（`buildForkedMessages`）：
1. 保留完整的父 assistant 消息（所有 tool_use 块、thinking、text）
2. 构建一个带有每个 tool_use 占位符工具结果的单用户消息
3. 附加带有严格规则的每子指令（不生成、不评论、报告格式）

#### 3. 正常子代理路径

当指定 `subagent_type` 时（或 fork 关闭时默认为 general-purpose）：

```
subagent_type → find agent definition → validate MCP requirements → run agent
```

步骤：
1. 从 `activeAgents` 列表查找代理定义
2. 检查权限规则（Agent(agentType) 拒绝规则）
3. 验证所需的 MCP 服务器可用（带等待待处理逻辑，最多 30s）
4. 如果定义则初始化代理颜色
5. 构建系统提示和提示消息
6. 确定同步 vs 异步执行
7. 通过 `runAgent()` 异步生成器执行

#### 4. 远程隔离（仅限 Ant）

当设置 `isolation: 'remote'` 时：
1. 检查远程代理资格
2. 通过 `teleportToRemote()` 委托给 CCR
3. 注册远程代理任务
4. 返回带会话 URL 的 `remote_launched` 输出

### 代理配置

#### 输入 Schema 字段

| 字段 | 类型 | 描述 |
|-------|------|-------------|
| `description` | `string` | 简短（3-5 词）任务摘要 |
| `prompt` | `string` | 代理执行的任务 |
| `subagent_type` | `string?` | 代理类型选择器（默认为 general-purpose 或 fork） |
| `model` | `'sonnet' \| 'opus' \| 'haiku'?` | 可选的模型覆盖 |
| `run_in_background` | `boolean?` | 异步运行代理 |
| `name` | `string?` | 可通过 SendMessage 寻址的名称 |
| `team_name` | `string?` | 用于生成队友的团队名称 |
| `mode` | `PermissionMode?` | 生成队友的权限模式 |
| `isolation` | `'worktree' \| 'remote'?` | 隔离模式 |
| `cwd` | `string?` | 工作目录覆盖（仅限 KAIROS） |

#### Schema 门控

schema 根据功能标志动态省略字段：
- `cwd`：当 `KAIROS` 功能关闭时省略
- `run_in_background`：当后台任务禁用（`CLAUDE_CODE_DISABLE_BACKGROUND_TASKS`）或 fork 子代理启用时省略（在 fork 模式下所有生成都是异步的）

### 代理生命周期

#### 同步执行

同步代理阻塞父轮次直到完成：

1. 注册为前台任务（`registerAgentForeground`）
2. 创建进度跟踪器和活动解析器
3. 产生初始进度消息
4. 迭代 `runAgent()` 异步生成器
5. 在下一条消息和后台信号之间竞态（用于执行中后台化）
6. 收到后台信号：清理前台迭代器，继续在后台运行
7. 完成时：完成结果，发送通知
8. 清理：worktree、MCP 服务器、钩子、文件状态缓存、待办事项、shell 任务

#### 异步执行

异步代理独立于父轮次运行：

1. 注册异步代理（`registerAsyncAgent`），带未链接的中止控制器
2. 在 `agentNameRegistry` 中注册名称以进行 SendMessage 路由
3. 在分离闭包（`void`）中启动 `runAsyncAgentLifecycle()`
4. 立即返回带 agentId 和 outputFile 路径的 `async_launched` 输出
5. 父代理继续；完成时发送通知

#### 执行中后台转换

同步代理可以在执行中后台化：

1. `registerAgentForeground` 设置 `backgroundSignal` promise
2. 在 `getAutoBackgroundMs()` 后自动后台化（启用时 120s，通过环境或 GrowthBook）
3. 信号触发时：清理前台迭代器（1s 超时），以后台模式重新迭代
4. 继续使用独立的摘要和进度跟踪

#### 清理（两种路径）

`runAgent()` 中的 `finally` 块处理：
- MCP 服务器清理（仅代理特定的服务器）
- 会话钩子清理（`clearSessionHooks`）
- 提示缓存跟踪清理
- 文件状态缓存释放
- Fork 上下文消息释放
- Perfetto 代理取消注册
- Transcript 子目录映射清理
- 待办事项条目移除
- 代理的 shell 任务终止
- 启用的 MCP 任务终止（启用时）

Worktree 清理：
- 基于钩子的 worktree：始终保留（无法检测 VCS 更改）
- 非钩子 worktree：检查相对于 head 提交的更改
  - 无更改：移除 worktree 并清除元数据
  - 有更改：保留 worktree，在结果中返回路径和分支

### 内置 vs 自定义代理

#### 代理类型系统

```
AgentDefinition = BuiltInAgentDefinition | CustomAgentDefinition | PluginAgentDefinition
```

| 类型 | 来源 | 提示存储 | 加载 |
|------|--------|---------------|---------|
| 内置 | `'built-in'` | 动态 `getSystemPrompt()` 函数 | 硬编码导入 |
| 自定义 | `'userSettings'`、`'projectSettings'`、`'policySettings'`、`'flagSettings'` | 解析内容的闭包 | `.claude/agents/` 中的 Markdown 文件 |
| 插件 | `'plugin'` | 插件内容闭包 | 插件系统 |

#### 内置代理

| 代理 | 类型 | 模型 | 工具 | 关键特性 |
|-------|------|-------|-------|-------------------|
| General Purpose | `general-purpose` | 默认子代理 | `['*']` | 回退代理，所有工具 |
| Explore | `Explore` | `haiku`（外部）/ `inherit`（ant） | 只读（无 Agent、Edit、Write） | 快速文件搜索，`omitClaudeMd: true` |
| Plan | `Plan` | `inherit` | 只读（无 Agent、Edit、Write） | 软件架构师，`omitClaudeMd: true` |
| Statusline Setup | `statusline-setup` | `sonnet` | `['Read', 'Edit']` | 颜色：橙色，配置状态栏 |
| Claude Code Guide | `claude-code-guide` | `haiku` | Bash/Read/WebFetch/WebSearch | 带当前配置上下文的动态提示 |
| Verification | `verification` | `inherit` | 只读 + Bash（无 Agent、Edit、Write） | 颜色：红色，`background: true`，对抗性测试 |

内置代理加载（`builtInAgents.ts`）：
- `GENERAL_PURPOSE_AGENT` 和 `STATUSLINE_SETUP_AGENT` 始终包含
- `EXPLORE_AGENT` 和 `PLAN_AGENT` 由 `BUILTIN_EXPLORE_PLAN_AGENTS` 功能 + `tengu_amber_stoat` 标志门控
- `CLAUDE_CODE_GUIDE_AGENT` 为非 SDK 入口点包含
- `VERIFICATION_AGENT` 由 `VERIFICATION_AGENT` 功能 + `tengu_hive_evidence` 标志门控
- 协调器模式使用 `getCoordinatorAgents()` 而不是

#### 自定义代理加载

`getAgentDefinitionsWithOverrides()`（按 cwd 记忆化）：

1. 从 `.claude/agents/` 目录加载 markdown 文件（用户、项目、策略、本地、标志源）
2. 解析每个文件：提取 frontmatter（name、description、tools、model、color 等）和正文内容
3. 验证必填字段（name、description）
4. 使用 `getSystemPrompt` 闭包构建 `CustomAgentDefinition`
5. 并发加载插件代理
6. 合并所有来源：`[builtInAgents, pluginAgents, customAgents]`
7. 按 agentType 去重（优先级：内置 → 插件 → 用户 → 项目 → 标志 → 托管）
8. 为所有活动代理初始化颜色
9. 为带 `memory: 'user'` 的代理初始化内存快照

#### 代理定义字段

```typescript
type BaseAgentDefinition = {
  agentType: string           // 唯一标识符
  whenToUse: string           // 代理列表中显示的描述
  tools?: string[]            // 允许的工具（'*' 表示所有）
  disallowedTools?: string[]  // 明确阻止的工具
  skills?: string[]           // 要预加载的技能名称
  mcpServers?: AgentMcpServerSpec[]  // 代理特定的 MCP 服务器
  hooks?: HooksSettings       // 会话范围的钩子
  color?: AgentColorName      // UI 颜色（red、blue、green、yellow、purple、orange、pink、cyan）
  model?: string              // 模型覆盖或 'inherit'
  effort?: EffortValue        // 努力级别
  permissionMode?: PermissionMode  // 权限模式覆盖
  maxTurns?: number           // 最大 agentic 轮次
  background?: boolean        // 始终作为后台运行
  isolation?: 'worktree' | 'remote'  // 隔离模式
  memory?: AgentMemoryScope   // 持久内存范围（user、project、local）
  omitClaudeMd?: boolean      // 从上下文中排除 CLAUDE.md
  initialPrompt?: string      // 预置到第一个用户轮次
  requiredMcpServers?: string[]  // 可用性所需的 MCP 服务器模式
}
```

### 代理通信

#### 父到子

父代理通过以下方式与生成的代理通信：

1. **初始提示**：传递给 AgentTool 的 `prompt` 参数成为用户消息
2. **系统提示**：从代理定义的 `getSystemPrompt()` + 环境细节构建
3. **Fork 上下文**：当 fork 路径时，完整父对话作为上下文消息继承

#### 子到父

子代理通过以下方式通信回来：

1. **同步结果**：`finalizeAgentTool()` 提取文本内容、工具使用计数、令牌、持续时间和使用统计
2. **异步通知**：后台代理完成时 `enqueueAgentNotification()` 触发
3. **进度事件**：`onProgress` 回调在执行期间产生 `agent_progress` 消息
4. **SendMessage**：父代理可以通过其名称或 ID 向运行中的代理发送后续消息

#### 交接分类

当 `TRANSCRIPT_CLASSIFIER` 功能启用且模式为 `'auto'` 时：
- 代理的 transcript 在交接前被分类
- 用安全警告标记潜在危险操作
- 警告被预置到返回给父代理的最终消息中

### 代理状态管理

#### 进度跟踪

`createProgressTracker()` 跟踪：
- 令牌计数（来自使用数据）
- 工具使用计数
- 最后活动描述（从工具名称解析）

进度更新通过以下方式流动：
1. `updateProgressFromMessage()` — 从每条产生的消息提取进度数据
2. `updateAsyncAgentProgress()` — 更新 AppState 中的任务状态
3. `emitTaskProgress()` — 发出带 taskId、toolUseId、description 的 SDK 进度事件

#### 代理身份

每个代理通过 `createAgentId()` 提前创建唯一的 `AgentId`（UUID）：
- 用于 worktree slug 生成
- 用于 transcript 录制（`recordSidechainTranscript`）
- 用于元数据持久化（`writeAgentMetadata`）
- 用于分析归因（`runWithAgentContext`）
- 当提供 `name` 参数时在 `agentNameRegistry` 中注册

#### Transcript 录制

代理对话录制到磁盘：
- 在查询循环开始前录制初始消息
- 每条新消息以正确的父 UUID 增量录制（每消息 O(1)）
- 录制的元数据：agentType、worktreePath、description
- 存储在 `subagents/` 目录中（可选地按 `transcriptSubdir` 分组）

#### 恢复支持

`resumeAgentBackground()` 重建先前生成的代理：
1. 从磁盘加载 transcript 和元数据
2. 过滤消息（移除仅空白、孤立的 thinking、未解析的工具使用）
3. 重建工具结果稳定性的内容替换状态
4. 恢复 worktree 路径（带存在性检查）
5. 重新选择代理定义（或回退到 general-purpose）
6. 通过 `runAsyncAgentLifecycle()` 重新运行

### 颜色管理

`agentColorManager.ts` 管理代理识别的 UI 颜色：

#### 颜色调色板

8 个预定义颜色：`red`、`blue`、`green`、`yellow`、`purple`、`orange`、`pink`、`cyan`

#### 主题映射

每个代理颜色映射到特定于主题的颜色：
```
red → red_FOR_SUBAGENTS_ONLY
blue → blue_FOR_SUBAGENTS_ONLY
...
```

#### API

- `setAgentColor(agentType, color)`：在全局颜色映射中为代理类型分配颜色
- `getAgentColor(agentType)`：检索代理类型的主题颜色（`general-purpose` 返回 `undefined`）

#### 颜色分配流程

1. 代理定义声明 `color` 字段（例如 Verification Agent：`'red'`、Statusline Setup：`'orange'`）
2. 代理生成时：`setAgentColor()` 注册颜色
3. 代理列表加载时：为所有活动代理初始化颜色
4. UI 使用 `userFacingNameBackgroundColor()` 在分组显示中为代理标签应用颜色

### 从目录加载代理

#### Markdown 文件格式

自定义代理在 `.claude/agents/*.md` 文件中定义，带 YAML frontmatter：

```markdown
---
name: my-agent
description: What this agent does
tools: [Read, Bash, Glob]
disallowedTools: [FileWrite]
model: sonnet
color: blue
permissionMode: acceptEdits
maxTurns: 50
memory: project
background: false
isolation: worktree
effort: high
hooks:
  SubagentStop: [notify.sh]
mcpServers:
  - slack
  - { my-server: { command: "...", args: [] } }
skills: [my-skill]
initialPrompt: Do this first...
---

Agent system prompt content goes here...
```

#### 解析管道

`parseAgentFromMarkdown()`：
1. 验证必填字段：`name`（字符串）、`description`（字符串）
2. 取消转义 description 中的换行符（`\\n` → `\n`）
3. 解析并验证可选字段：
   - `color`：必须在 `AGENT_COLORS` 列表中
   - `model`：字符串，`'inherit'` 规范化为小写
   - `background`：布尔值或 `'true'`/`'false'` 字符串
   - `memory`：必须是 `'user'`、`'project'` 或 `'local'`
   - `isolation`：`'worktree'`（始终）或 `'remote'`（仅限 ant）
   - `effort`：字符串级别或整数
   - `permissionMode`：必须在 `PERMISSION_MODES` 中
   - `maxTurns`：正整数
   - `tools`、`disallowedTools`：通过 `parseAgentToolsFromFrontmatter()` 解析
   - `skills`：通过 `parseSlashCommandToolsFromFrontmatter()` 解析
   - `mcpServers`：通过 `AgentMcpServerSpecSchema` 验证
   - `hooks`：通过 `HooksSchema` 验证
4. 构建 `getSystemPrompt` 闭包：如果内存启用则返回内容 + 内存提示
5. 返回 `CustomAgentDefinition` 或 `null`（静默跳过非代理 markdown）

#### JSON 代理格式

代理也可以在 JSON 设置中定义：

```json
{
  "my-agent": {
    "description": "What this agent does",
    "prompt": "System prompt content",
    "tools": ["Read", "Bash"],
    "model": "sonnet",
    "memory": "project"
  }
}
```

通过 `AgentJsonSchema` 验证使用 `parseAgentFromJson()` 解析。

#### MCP 服务器要求

代理可以声明必需的 MCP 服务器：
- `requiredMcpServers`：服务器名称模式数组
- 在生成时检查：每个模式必须匹配至少一个可用服务器
- 如果必需的服务器待处理（连接中），最多等待 30s（每 500ms 轮询）
- 不满足要求的代理从可用代理列表中过滤掉

#### 代理过滤

代理在呈现给 LLM 之前被过滤：
1. **MCP 要求**：`filterAgentsByMcpRequirements()` — 移除所需服务器不可用的代理
2. **权限规则**：`filterDeniedAgents()` — 移除被 `Agent(agentType)` 规则拒绝的代理
3. **允许的类型**：使用 `Agent(x,y)` 语法时，限制为指定类型

### 代理内存

#### 内存范围

| 范围 | 目录 | 用例 |
|-------|-----------|-------------|
| `user` | `<memoryBase>/agent-memory/<agentType>/` | 跨项目学习 |
| `project` | `<cwd>/.claude/agent-memory/<agentType>/` | 团队共享项目内存 |
| `local` | `<cwd>/.claude/agent-memory-local/<agentType>/` | 机器特定内存 |

#### 内存集成

当在代理定义中启用 `memory` 时：
1. `loadAgentMemoryPrompt()` 使用范围特定指南构建内存提示
2. 内存提示附加到代理的系统提示（`systemPrompt + '\n\n' + memoryPrompt`）
3. 如果内存启用，FileWrite/Edit/Read 工具被自动注入（用于内存文件访问）

#### 快照系统

项目级内存快照允许跨团队成员共享代理内存：

- 快照存储在 `.claude/agent-memory-snapshots/<agentType>/`
- `checkAgentMemorySnapshot()`：比较快照时间戳与本地同步状态
  - `'initialize'`：不存在本地内存 → 复制快照到本地
  - `'prompt-update'`：存在较新快照 → 通知更新
  - `'none'`：本地是最新的
- `initializeFromSnapshot()`：首次设置，复制快照文件并记录同步元数据
- `replaceFromSnapshot()`：用快照覆盖本地内存（首先删除现有 .md 文件）

### 工具解析

`resolveAgentTools()` 确定代理可以访问哪些工具：

1. **通过 `filterToolsForAgent()` 过滤可用工具**：
   - MCP 工具始终允许
   - 计划模式代理允许 ExitPlanMode
   - `ALL_AGENT_DISALLOWED_TOOLS` 对所有代理阻止（Agent、TaskCreate 等）
   - `CUSTOM_AGENT_DISALLOWED_TOOLS` 仅对自定义代理阻止
   - `ASYNC_AGENT_ALLOWED_TOOLS` 限制后台代理（带队友例外）

2. **应用代理特定限制**：
   - `tools` 字段：如果未定义或 `['*']`，所有过滤工具都可用
   - 否则，仅包含明确列出的工具
   - `disallowedTools` 字段：明确移除列出的工具
   - 带规则内容的代理规范（`Agent(worker, researcher)`）提取 `allowedAgentTypes`

3. **按名称去重**已解析工具

### UI 渲染

`UI.tsx` 为终端显示提供 React 组件：

#### 渲染函数

- `renderToolUseMessage`：在初始化期间显示代理描述
- `renderToolUseProgressMessage`：显示实时进度（小终端的压缩模式，带搜索/读取分组的全模式）
- `renderToolResultMessage`：显示带统计的完成摘要（工具使用、令牌、持续时间）
- `renderToolUseRejectedMessage`：显示带进度历史的拒绝
- `renderToolUseErrorMessage`：显示带进度历史的错误
- `renderGroupedAgentToolUse`：将多个并行代理生成分组为单个显示

#### 进度消息处理

对于 ant 构建，连续搜索/读取操作被分组为摘要：
- `processProgressMessages()`：分组连续 Glob/Grep/Read 操作
- 显示摘要文本如 "3 searches, 5 reads..." 而非单独条目
- 活动组（进行中）vs 完成组区分

#### 代理统计显示

每个代理显示：
- 代理类型（或用于队友生成的 `@name`）
- 描述
- 工具使用计数
- 令牌计数
- 最后工具信息（压缩搜索/读取摘要或工具名称）
- 异步状态指示器
- 基于代理定义的配色

## 配置

### 环境变量

| 变量 | 用途 |
|----------|---------|
| `CLAUDE_CODE_DISABLE_BACKGROUND_TASKS` | 禁用异步代理执行 |
| `CLAUDE_AUTO_BACKGROUND_TASKS` | 启用 120s 后自动后台化 |
| `CLAUDE_CODE_COORDINATOR_MODE` | 启用协调器模式（不同的代理集） |
| `CLAUDE_CODE_SIMPLE` | 跳过自定义代理加载，仅返回内置 |
| `CLAUDE_AGENT_SDK_DISABLE_BUILTIN_AGENTS` | 在 SDK 模式中禁用内置代理 |
| `CLAUDE_CODE_AGENT_LIST_IN_MESSAGES` | 控制代理列表放置（附件 vs 内联） |
| `CLAUDE_CODE_REMOTE_MEMORY_DIR` | 将本地内存重定向到远程挂载 |
| `CLAUDE_COWORK_MEMORY_EXTRA_GUIDELINES` | 额外内存指南 |
| `USER_TYPE=ant` | 启用 ant 专属功能（远程隔离、嵌入式搜索） |

### 功能标志

| 标志 | 用途 |
|------|---------|
| `FORK_SUBAGENT` | 启用 fork 子代理实验 |
| `KAIROS` | 启用助手模式（强制所有代理异步） |
| `COORDINATOR_MODE` | 启用协调器模式 |
| `TRANSCRIPT_CLASSIFIER` | 启用交接安全分类 |
| `VERIFICATION_AGENT` | 启用验证代理 |
| `BUILTIN_EXPLORE_PLAN_AGENTS` | 启用 Explore/Plan 代理 |
| `AGENT_MEMORY_SNAPSHOT` | 启用内存快照系统 |
| `PROACTIVE` / `KAIROS` | 启用主动模块集成 |

## 错误处理

### 生成错误

- **找不到代理类型**：在错误消息中列出可用的代理类型
- **代理被权限拒绝**：显示拒绝规则及其来源
- **MCP 服务器不可用**：列出缺失的服务器和带工具的可用服务器
- **队友限制**：清除无效队友生成尝试的错误消息
- **递归 fork 守卫**："Fork is not available inside a forked worker"
- **团队访问**："Agent Teams is not yet available on your plan"

### 运行时错误

- **无 assistant 消息**：如果代理没有产生 assistant 消息则抛出
- **中止处理**：`AbortError` 触发带部分结果的通知
- **常规错误**：捕获，发送失败通知和错误消息
- **Worktree 回退**：如果恢复时 worktree 不再存在，回退到父 cwd

## 数据流

```
用户/父代理
    │
    ▼
AgentTool.call()
    │
    ├── [team_name + name] ──→ spawnTeammate() ──→ teammate_spawned
    │
    ├── [!subagent_type + fork] ──→ FORK_AGENT
    │   ├── buildForkedMessages() — 克隆父上下文
    │   ├── 继承父系统提示
    │   └── runAgent(useExactTools: true)
    │
    ├── [subagent_type] ──→ find AgentDefinition
    │   ├── validate MCP requirements
    │   ├── build system prompt + memory
    │   ├── resolve tools (filter + allowlist)
    │   ├── create worktree (if isolation: worktree)
    │   │
    │   ├── [async] ──→ registerAsyncAgent()
    │   │   └── runAsyncAgentLifecycle()
    │   │       ├── runAgent() generator
    │   │       ├── progress tracking
    │   │       ├── summarization
    │   │       └── enqueueAgentNotification()
    │   │
    │   └── [sync] ──→ registerAgentForeground()
    │       ├── runAgent() generator
    │       ├── race with background signal
    │       ├── [backgrounded] → transition to async
    │       └── finalizeAgentTool()
    │
    └── [isolation: remote] ──→ teleportToRemote() ──→ remote_launched
```

## 集成点

- **LocalAgentTask**：任务注册、进度跟踪、终止管理、通知
- **SendMessageTool**：通过代理名称/ID 注册表的代理间通信
- **MCP Service**：代理特定的 MCP 服务器连接和清理
- **Hooks System**：SubagentStart/SubagentStop 钩子、frontmatter 钩子注册
- **Session Storage**：Transcript 录制、元数据持久化、代理颜色映射
- **Analytics**：生成、完成、终止、内存加载的事件日志
- **Perfetto Tracing**：性能跟踪中的代理层次结构可视化
- **SDK Event Queue**：面向 SDK 消费者的进度事件发射
- **Worktree System**：Git worktree 创建、更改检测、清理

## 相关模块

- [LocalAgentTask](../tasks/LocalAgentTask.md) — 本地代理的任务管理
- [SendMessageTool](./SendMessageTool.md) — 代理间通信
- [runAgent](./AgentTool/runAgent.md) — 核心代理执行引擎
- [loadAgentsDir](./AgentTool/loadAgentsDir.md) — 代理定义加载
- [forkSubagent](./AgentTool/forkSubagent.md) — Fork 子代理实验
- [agentColorManager](./AgentTool/agentColorManager.md) — 颜色管理
- [agentMemory](./AgentTool/agentMemory.md) — 持久代理内存
- [agentToolUtils](./AgentTool/agentToolUtils.md) — 共享工具
- [prompt](./AgentTool/prompt.md) — 工具提示生成
- [UI](./AgentTool/UI.md) — 终端 UI 渲染
