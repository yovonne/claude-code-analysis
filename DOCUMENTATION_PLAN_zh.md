# Claude Code 源代码文档计划

## 概述

本文档概述了 Claude Code CLI 源代码（版本 2.1.88）的综合文档策略，源代码从 npm 包源映射中重建。

### 代码库统计
- **TypeScript 文件**：~1884
- **目录**：~200
- **工具**：35+
- **命令**：50+
- **服务**：15+
- **UI 组件**：50+

---

## 文档结构

```
analysis/
├── DOCUMENTATION_PLAN.md            # 本文件
├── task_breakdown.json              # 任务跟踪
├── README.md                        # 文档索引
│
├── 00-architecture-overview.md      # 高层架构
│
├── 01-core-modules/
│   ├── main-entrypoint.md           # main.tsx 功能部分
│   ├── tool-system.md               # 工具抽象和注册表
│   ├── query-engine.md              # QueryEngine.ts
│   ├── command-system.md            # 命令分发
│   └── state-management.md           # AppState 和状态流
│
├── 02-tools/                        # 单独的工具文档
│   ├── AgentTool.md
│   ├── AskUserQuestionTool.md
│   ├── BashTool.md
│   ├── BriefTool.md
│   ├── ConfigTool.md
│   ├── CronCreateTool.md
│   ├── CronDeleteTool.md
│   ├── CronListTool.md
│   ├── EnterPlanModeTool.md
│   ├── EnterWorktreeTool.md
│   ├── ExitPlanModeTool.md
│   ├── ExitWorktreeTool.md
│   ├── FileEditTool.md
│   ├── FileReadTool.md
│   ├── FileWriteTool.md
│   ├── GlobTool.md
│   ├── GrepTool.md
│   ├── LSPTool.md
│   ├── ListMcpResourcesTool.md
│   ├── MCPTool.md
│   ├── McpAuthTool.md
│   ├── MonitorTool.md
│   ├── NotebookEditTool.md
│   ├── PowerShellTool.md
│   ├── PushNotificationTool.md
│   ├── REPLTool.md
│   ├── ReadMcpResourceTool.md
│   ├── RemoteTriggerTool.md
│   ├── SendMessageTool.md
│   ├── SendUserFileTool.md
│   ├── SkillTool.md
│   ├── SleepTool.md
│   ├── SubscribePRTool.md
│   ├── SuggestBackgroundPRTool.md
│   ├── SyntheticOutputTool.md
│   ├── TaskCreateTool.md
│   ├── TaskGetTool.md
│   ├── TaskListTool.md
│   ├── TaskOutputTool.md
│   ├── TaskStopTool.md
│   ├── TaskUpdateTool.md
│   ├── TeamCreateTool.md
│   ├── TeamDeleteTool.md
│   ├── TodoWriteTool.md
│   ├── ToolSearchTool.md
│   ├── TungstenTool.md
│   ├── VerifyPlanExecutionTool.md
│   ├── WebFetchTool.md
│   └── WebSearchTool.md
│
├── 03-commands/                     # CLI 命令文档
│   ├── add-dir.md
│   ├── agents.md
│   ├── ant-trace.md
│   ├── autofix-pr.md
│   ├── backfill-sessions.md
│   ├── branch.md
│   ├── break-cache.md
│   ├── bridge.md
│   ├── btw.md
│   ├── bughunter.md
│   ├── chrome.md
│   ├── clear.md
│   ├── color.md
│   ├── compact.md
│   ├── config.md
│   ├── context.md
│   ├── copy.md
│   ├── cost.md
│   ├── ctx_viz.md
│   ├── debug-tool-call.md
│   ├── desktop.md
│   ├── diff.md
│   ├── doctor.md
│   ├── effort.md
│   ├── env.md
│   ├── exit.md
│   ├── export.md
│   ├── extra-usage.md
│   ├── fast.md
│   ├── feedback.md
│   ├── files.md
│   ├── good-claude.md
│   ├── heapdump.md
│   ├── help.md
│   ├── hooks.md
│   ├── ide.md
│   ├── install-github-app.md
│   ├── install-slack-app.md
│   ├── issue.md
│   ├── keybindings.md
│   ├── login.md
│   ├── logout.md
│   ├── mcp.md
│   ├── memory.md
│   ├── mobile.md
│   ├── mock-limits.md
│   ├── model.md
│   ├── oauth-refresh.md
│   ├── onboarding.md
│   ├── output-style.md
│   ├── passes.md
│   ├── perf-issue.md
│   ├── permissions.md
│   ├── plan.md
│   ├── plugin.md
│   ├── pr_comments.md
│   ├── privacy-settings.md
│   ├── rate-limit-options.md
│   ├── release-notes.md
│   ├── reload-plugins.md
│   ├── remote-env.md
│   ├── remote-setup.md
│   ├── rename.md
│   ├── reset-limits.md
│   ├── resume.md
│   ├── review.md
│   ├── rewind.md
│   ├── sandbox-toggle.md
│   ├── session.md
│   ├── share.md
│   ├── skills.md
│   ├── stats.md
│   ├── status.md
│   ├── stickers.md
│   ├── summary.md
│   ├── tag.md
│   ├── tasks.md
│   ├── teleport.md
│   ├── terminalSetup.md
│   ├── theme.md
│   ├── thinkback.md
│   ├── thinkback-play.md
│   ├── upgrade.md
│   ├── usage.md
│   ├── vim.md
│   └── voice.md
│
├── 04-services/
│   ├── api-service.md
│   ├── analytics-service.md
│   ├── auth-service.md
│   ├── auto-dream-service.md
│   ├── compact-service.md
│   ├── extract-memories-service.md
│   ├── lsp-service.md
│   ├── mcp-service.md
│   ├── oauth-service.md
│   ├── plugins-service.md
│   ├── policy-limits-service.md
│   ├── remote-managed-settings.md
│   ├── settings-sync-service.md
│   ├── team-memory-sync-service.md
│   ├── tips-service.md
│   └── tool-use-summary-service.md
│
├── 05-utils/
│   ├── advisor.md
│   ├── auth.md
│   ├── bash-utils.md
│   ├── commit-attribution.md
│   ├── computer-use.md
│   ├── config.md
│   ├── deep-link.md
│   ├── effort.md
│   ├── fast-mode.md
│   ├── file-persistence.md
│   ├── file-state-cache.md
│   ├── git-utils.md
│   ├── github-utils.md
│   ├── hooks-utils.md
│   ├── mcp-utils.md
│   ├── memory-utils.md
│   ├── model-utils.md
│   ├── permissions-utils.md
│   ├── sandbox-utils.md
│   ├── secure-storage.md
│   ├── settings-utils.md
│   ├── shell-utils.md
│   ├── skills-utils.md
│   ├── swarm-utils.md
│   ├── task-utils.md
│   ├── telemetry-utils.md
│   ├── teleport-utils.md
│   ├── todo-utils.md
│   ├── ultraplan-utils.md
│   └── worktree-mode.md
│
├── 06-ui/
│   ├── ink-framework.md
│   ├── components-overview.md
│   ├── design-system.md
│   ├── messages-components.md
│   ├── permissions-components.md
│   ├── keybindings-system.md
│   ├── vim-mode.md
│   └── spinner-components.md
│
├── 07-advanced-features/
│   ├── coordinator-mode.md
│   ├── assistant-mode.md
│   ├── buddy-system.md
│   ├── plugins-system.md
│   ├── skills-system.md
│   └── voice-mode.md
│
├── 08-internals/
│   ├── bootstrap.md
│   ├── migrations.md
│   ├── hooks-system.md
│   ├── context-providers.md
│   ├── native-ts-modules.md
│   ├── schemas.md
│   ├── types-system.md
│   └── vendor-modules.md
│
└── 09-data-flow/
    ├── message-flow.md
    ├── permission-flow.md
    ├── tool-execution-flow.md
    └── session-lifecycle.md
```

---

## 执行阶段

### 阶段 1：核心架构（5 个文件）
分析应用程序的基本构建块。

| 文件 | 来源 | 描述 |
|------|--------|-------------|
| 00-architecture-overview.md | All | 高层系统架构 |
| main-entrypoint.md | main.tsx | CLI 入口、初始化、命令解析 |
| tool-system.md | Tool.ts, tools.ts | 工具抽象、注册表、执行 |
| query-engine.md | QueryEngine.ts | 查询处理、LLM 交互 |
| state-management.md | AppState.tsx | 应用状态、React 上下文 |

### 阶段 2：工具（48 个文件）
记录每个工具的单独实现细节。

类别：
- **文件操作**：FileReadTool, FileWriteTool, FileEditTool, GlobTool, GrepTool, NotebookEditTool
- **Shell 执行**：BashTool, PowerShellTool, REPLTool
- **Agent/任务**：AgentTool, TaskCreateTool, TaskGetTool, TaskListTool, TaskOutputTool, TaskStopTool, TaskUpdateTool
- **团队**：TeamCreateTool, TeamDeleteTool, SendMessageTool
- **MCP**：MCPTool, McpAuthTool, ListMcpResourcesTool, ReadMcpResourceTool
- **规划**：EnterPlanModeTool, ExitPlanModeTool, EnterWorktreeTool, ExitWorktreeTool
- **用户交互**：AskUserQuestionTool, SkillTool, BriefTool
- **Web**：WebFetchTool, WebSearchTool
- **系统**：ConfigTool, LSPTool, TodoWriteTool, ToolSearchTool, SyntheticOutputTool
- **调度**：CronCreateTool, CronDeleteTool, CronListTool, RemoteTriggerTool
- **KAIROS/仅 Ant**：SleepTool, MonitorTool, SendUserFileTool, PushNotificationTool, SubscribePRTool, SuggestBackgroundPRTool, VerifyPlanExecutionTool
- **其他**：TungstenTool

### 阶段 3：命令（50 个文件）
记录所有 CLI 命令的用法、选项和实现。

### 阶段 4：服务（16 个文件）
记录外部集成的服务层。

### 阶段 5：工具（30 个文件）
记录工具模块和辅助函数。

### 阶段 6：UI（8 个文件）
记录终端 UI 组件和 Ink 框架集成。

### 阶段 7：高级功能（6 个文件）
记录实验和高级功能模块。

### 阶段 8：内部（8 个文件）
记录内部基础设施模块。

### 阶段 9：数据流（4 个文件）
记录关键数据流和系统交互。

---

## 文档模板

每个模块文档遵循此结构：

```markdown
# 模块名称

## 目的
简要描述此模块的功能及其在系统中的角色。

## 位置
`restored-src/src/path/to/module.ts`

## 关键导出

### 函数
- `functionName`：描述其功能

### 类
- `ClassName`：描述该类

### 类型
- `TypeName`：描述该类型

### 常量
- `CONSTANT_NAME`：描述和值

## 依赖

### 内部依赖
- 模块 A - 用途
- 模块 B - 用途

### 外部依赖
- 包 X - 用途

## 实现细节

### 核心逻辑
主要实现方法的解释。

### 关键算法
使用的任何重要算法或模式。

### 边缘情况
特殊处理或边缘情况。

## 数据流
描述数据如何流经此模块。

## 集成点
此模块如何与系统其他部分集成。

## 配置
任何配置选项或环境变量。

## 错误处理
错误如何处理和报告。

## 测试
测试方法（如果适用）。

## 相关模块
- [模块 A](./path/to/module-a.md)
- [模块 B](./path/to/module-b.md)

## 注意事项
任何其他注意事项或警告。
```

---

## 命名约定

- **文件**：小写加连字符，与模块名称匹配
- **标题**：主标题用首字母大写
- **代码引用**：文件路径、函数名和代码使用反引号
- **链接**：文档文件之间的相对链接

---

## 进度跟踪

进度在 `task_breakdown.json` 中跟踪，状态如下：
- `pending`：尚未开始
- `in_progress`：正在进行
- `completed`：已完成并验证
- `blocked`：等待依赖

---

## 注意事项

- 所有文档均为英文
- main.tsx 文档按功能部分分解
- 每个工具都有自己专属的文件
- 交叉引用链接相关模块
