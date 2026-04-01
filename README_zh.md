# Claude Code 源代码分析

Claude Code CLI（v2.1.88）源代码的综合文档，从 npm 包源映射重建。

## 概述

本分析涵盖 ~1884 个 TypeScript 文件，分布在 ~200 个目录中，记录了 Claude Code CLI 的架构、工具、命令、服务、工具、UI 组件和高级功能。

## 文档索引

### 基础
- [架构概述](./00-architecture-overview.md) - 高层系统架构

### 01 核心模块
- [主入口点](./01-core-modules/main-entrypoint.md) - CLI 入口、初始化、命令解析
- [工具系统](./01-core-modules/tool-system.md) - 工具抽象和注册表
- [查询引擎](./01-core-modules/query-engine.md) - 查询处理、LLM 交互
- [命令系统](./01-core-modules/command-system.md) - 命令分发
- [状态管理](./01-core-modules/state-management.md) - AppState 和状态流

### 02 工具（48 个工具）
- [AgentTool](./02-tools/AgentTool.md) - 子 agent 编排
- [AskUserQuestionTool](./02-tools/AskUserQuestionTool.md) - 用户提示
- [BashTool](./02-tools/BashTool.md) - Shell 命令执行
- [BriefTool](./02-tools/BriefTool.md) - 上下文摘要
- [ConfigTool](./02-tools/ConfigTool.md) - 配置管理
- [EnterPlanModeTool](./02-tools/EnterPlanModeTool.md) - 计划模式激活
- [EnterWorktreeTool](./02-tools/EnterWorktreeTool.md) - Git worktree 创建
- [ExitPlanModeTool](./02-tools/ExitPlanModeTool.md) - 计划模式停用
- [ExitWorktreeTool](./02-tools/ExitWorktreeTool.md) - Worktree 清理
- [FileEditTool](./02-tools/FileEditTool.md) - 搜索替换编辑
- [FileReadTool](./02-tools/FileReadTool.md) - 文件读取
- [FileWriteTool](./02-tools/FileWriteTool.md) - 文件写入
- [GlobTool](./02-tools/GlobTool.md) - Glob 模式匹配
- [GrepTool](./02-tools/GrepTool.md) - 正则搜索
- [LSPTool](./02-tools/LSPTool.md) - 语言服务器协议
- [ListMcpResourcesTool](./02-tools/ListMcpResourcesTool.md) - MCP 资源列表
- [MCPTool](./02-tools/MCPTool.md) - MCP 工具调用
- [McpAuthTool](./02-tools/McpAuthTool.md) - MCP 认证
- [NotebookEditTool](./02-tools/NotebookEditTool.md) - Jupyter 笔记本编辑
- [PowerShellTool](./02-tools/PowerShellTool.md) - PowerShell 执行
- [ReadMcpResourceTool](./02-tools/ReadMcpResourceTool.md) - MCP 资源读取
- [REPLTool](./02-tools/REPLTool.md) - 交互式 REPL（仅 Ant）
- [RemoteTriggerTool](./02-tools/RemoteTriggerTool.md) - 远程触发处理
- [SendMessageTool](./02-tools/SendMessageTool.md) - Agent 间消息传递
- [SkillTool](./02-tools/SkillTool.md) - 技能调用
- [ScheduleCronTool](./02-tools/ScheduleCronTool.md) - Cron 作业管理
- [SyntheticOutputTool](./02-tools/SyntheticOutputTool.md) - 合成输出生成
- [TaskCreateTool](./02-tools/TaskCreateTool.md) - 任务创建
- [TaskGetTool](./02-tools/TaskGetTool.md) - 任务获取
- [TaskListTool](./02-tools/TaskListTool.md) - 任务列表
- [TaskOutputTool](./02-tools/TaskOutputTool.md) - 任务输出获取
- [TaskStopTool](./02-tools/TaskStopTool.md) - 任务取消
- [TaskUpdateTool](./02-tools/TaskUpdateTool.md) - 任务状态更新
- [TeamCreateTool](./02-tools/TeamCreateTool.md) - 团队创建
- [TeamDeleteTool](./02-tools/TeamDeleteTool.md) - 团队删除
- [TodoWriteTool](./02-tools/TodoWriteTool.md) - 待办列表管理
- [ToolSearchTool](./02-tools/ToolSearchTool.md) - 工具发现
- [TungstenTool](./02-tools/TungstenTool.md) - Tungsten 集成
- [WebFetchTool](./02-tools/WebFetchTool.md) - URL 获取
- [WebSearchTool](./02-tools/WebSearchTool.md) - Web 搜索
- [KAIROS/仅 Ant 工具](./02-tools/kairos-ant-tools.md) - 功能门控工具

### 03 命令（50+ 个命令）
- [会话命令](./03-commands/session-commands.md) - 会话生命周期
- [清除/退出命令](./03-commands/clear-exit-commands.md) - 会话清理
- [配置命令](./03-commands/config-commands.md) - 配置
- [设置命令](./03-commands/settings-commands.md) - 运行时设置
- [认证命令](./03-commands/auth-commands.md) - 认证
- [审查命令](./03-commands/review-commands.md) - 代码审查
- [分支/问题命令](./03-commands/branch-issue-commands.md) - Git 和问题
- [MCP 命令](./03-commands/mcp-commands.md) - MCP 管理
- [功能命令](./03-commands/feature-commands.md) - 功能特定
- [交互命令](./03-commands/interaction-commands.md) - 交互会话
- [UI 命令](./03-commands/ui-commands.md) - 信息显示
- [调试命令](./03-commands/debug-commands.md) - 调试
- [远程命令](./03-commands/remote-commands.md) - 远程会话
- [集成命令](./03-commands/integration-commands.md) - 外部集成
- [杂项命令](./03-commands/misc-commands.md) - 杂项

### 04 服务
- [API 服务](./04-services/api-service.md) - API 客户端
- [分析服务](./04-services/analytics-service.md) - 遥测
- [OAuth 服务](./04-services/oauth-service.md) - 认证
- [MCP 服务](./04-services/mcp-service.md) - MCP 集成
- [LSP 服务](./04-services/lsp-service.md) - 语言服务器协议
- [压缩服务](./04-services/compact-service.md) - 上下文压缩
- [插件服务](./04-services/plugins-service.md) - 插件系统
- [设置同步服务](./04-services/settings-sync-service.md) - 云同步
- [团队内存同步服务](./04-services/team-memory-sync-service.md) - 团队内存
- [策略限制服务](./04-services/policy-limits-service.md) - 速率限制
- [远程托管设置](./04-services/remote-managed-settings.md) - MDM
- [自动梦想服务](./04-services/auto-dream-service.md) - 主动建议
- [提取记忆服务](./04-services/extract-memories-service.md) - 记忆提取
- [提示服务](./04-services/tips-service.md) - 用户指导
- [工具使用摘要服务](./04-services/tool-use-summary-service.md) - 使用跟踪

### 05 工具
- [认证](./05-utils/auth.md) | [配置](./05-utils/config.md) | [Git](./05-utils/git-utils.md)
- [权限](./05-utils/permissions-utils.md) | [设置](./05-utils/settings-utils.md)
- [Shell](./05-utils/shell-utils.md) | [Bash](./05-utils/bash-utils.md)
- [模型](./05-utils/model-utils.md) | [内存](./05-utils/memory-utils.md)
- [MCP](./05-utils/mcp-utils.md) | [技能](./05-utils/skills-utils.md)
- [Swarm](./05-utils/swarm-utils.md) | [遥测](./05-utils/telemetry-utils.md)
- [任务](./05-utils/task-utils.md) | [待办](./05-utils/todo-utils.md)
- [安全存储](./05-utils/secure-storage.md) | [沙箱](./05-utils/sandbox-utils.md)
- [GitHub](./05-utils/github-utils.md) | [文件持久化](./05-utils/file-persistence.md)
- [钩子](./05-utils/hooks-utils.md) | [努力/快速模式](./05-utils/effort-fast-mode.md)
- [传送](./05-utils/teleport-utils.md) | [Ultraplan](./05-utils/ultraplan-utils.md)
- [计算机使用](./05-utils/computer-use.md) | [深度链接](./05-utils/deep-link.md)
- [Worktree 模式](./05-utils/worktree-mode.md) | [顾问/归属](./05-utils/advisor-attribution.md)

### 06 UI
- [Ink 框架](./06-ui/ink-framework.md) - 终端 UI 框架
- [组件概述](./06-ui/components-overview.md) - React 组件
- [设计系统](./06-ui/design-system.md) - 设计令牌
- [消息组件](./06-ui/messages-components.md) - 消息渲染
- [权限组件](./06-ui/permissions-components.md) - 权限对话框
- [键盘绑定系统](./06-ui/keybindings-system.md) - 键盘快捷键
- [Vim 模式](./06-ui/vim-mode.md) - Vim 模拟
- [旋转组件](./06-ui/spinner-components.md) - 加载指示器

### 07 高级功能
- [协调器模式](./07-advanced-features/coordinator-mode.md) - 多 agent 协调
- [助手模式](./07-advanced-features/assistant-mode.md) - KAIROS 助手
- [伙伴系统](./07-advanced-features/buddy-system.md) - AI 伴侣
- [插件系统](./07-advanced-features/plugins-system.md) - 插件架构
- [技能系统](./07-advanced-features/skills-system.md) - 技能框架
- [语音模式](./07-advanced-features/voice-mode.md) - 语音交互

### 08 内部
- [引导](./08-internals/bootstrap.md) - 应用程序启动
- [迁移](./08-internals/migrations.md) - 数据库迁移
- [钩子系统](./08-internals/hooks-system.md) - React 钩子
- [上下文提供者](./08-internals/context-providers.md) - React 上下文
- [原生 TS 模块](./08-internals/native-ts-modules.md) - 原生模块
- [模式](./08-internals/schemas.md) - Zod 模式
- [类型系统](./08-internals/types-system.md) - TypeScript 类型
- [供应商模块](./08-internals/vendor-modules.md) - 供应商代码

### 09 数据流
- [消息流](./09-data-flow/message-flow.md) - 用户输入到 LLM 响应
- [权限流](./09-data-flow/permission-flow.md) - 权限请求生命周期
- [工具执行流](./09-data-flow/tool-execution-flow.md) - 工具执行生命周期
- [会话生命周期](./09-data-flow/session-lifecycle.md) - 会话管理

## 来源

- **原始包**：[@anthropic-ai/claude-code](https://www.npmjs.com/package/@anthropic-ai/claude-code)
- **版本**：2.1.88
- **源文件**：~1884 个 TypeScript 文件
- **重建方法**：从 `cli.js.map` sourcesContent 字段提取
