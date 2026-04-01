# Claude Code Source Code Analysis

Comprehensive documentation of the Claude Code CLI (v2.1.88) source code, reconstructed from npm package source maps.

## Overview

This analysis covers ~1884 TypeScript files across ~200 directories, documenting the architecture, tools, commands, services, utilities, UI components, and advanced features of the Claude Code CLI.

## Documentation Index

### Foundation
- [Architecture Overview](./00-architecture-overview.md) - High-level system architecture

### 01 Core Modules
- [Main Entrypoint](./01-core-modules/main-entrypoint.md) - CLI entry, initialization, command parsing
- [Tool System](./01-core-modules/tool-system.md) - Tool abstraction & registry
- [Query Engine](./01-core-modules/query-engine.md) - Query processing, LLM interaction
- [Command System](./01-core-modules/command-system.md) - Command dispatch
- [State Management](./01-core-modules/state-management.md) - AppState & state flow

### 02 Tools (48 tools)
- [AgentTool](./02-tools/AgentTool.md) - Sub-agent orchestration
- [AskUserQuestionTool](./02-tools/AskUserQuestionTool.md) - User prompting
- [BashTool](./02-tools/BashTool.md) - Shell command execution
- [BriefTool](./02-tools/BriefTool.md) - Context summarization
- [ConfigTool](./02-tools/ConfigTool.md) - Configuration management
- [EnterPlanModeTool](./02-tools/EnterPlanModeTool.md) - Plan mode activation
- [EnterWorktreeTool](./02-tools/EnterWorktreeTool.md) - Git worktree creation
- [ExitPlanModeTool](./02-tools/ExitPlanModeTool.md) - Plan mode deactivation
- [ExitWorktreeTool](./02-tools/ExitWorktreeTool.md) - Worktree cleanup
- [FileEditTool](./02-tools/FileEditTool.md) - Search-and-replace editing
- [FileReadTool](./02-tools/FileReadTool.md) - File reading
- [FileWriteTool](./02-tools/FileWriteTool.md) - File writing
- [GlobTool](./02-tools/GlobTool.md) - Glob pattern matching
- [GrepTool](./02-tools/GrepTool.md) - Regex search
- [LSPTool](./02-tools/LSPTool.md) - Language Server Protocol
- [ListMcpResourcesTool](./02-tools/ListMcpResourcesTool.md) - MCP resource listing
- [MCPTool](./02-tools/MCPTool.md) - MCP tool invocation
- [McpAuthTool](./02-tools/McpAuthTool.md) - MCP authentication
- [NotebookEditTool](./02-tools/NotebookEditTool.md) - Jupyter notebook editing
- [PowerShellTool](./02-tools/PowerShellTool.md) - PowerShell execution
- [ReadMcpResourceTool](./02-tools/ReadMcpResourceTool.md) - MCP resource reading
- [REPLTool](./02-tools/REPLTool.md) - Interactive REPL (Ant-only)
- [RemoteTriggerTool](./02-tools/RemoteTriggerTool.md) - Remote trigger handling
- [SendMessageTool](./02-tools/SendMessageTool.md) - Inter-agent messaging
- [SkillTool](./02-tools/SkillTool.md) - Skill invocation
- [ScheduleCronTool](./02-tools/ScheduleCronTool.md) - Cron job management
- [SyntheticOutputTool](./02-tools/SyntheticOutputTool.md) - Synthetic output generation
- [TaskCreateTool](./02-tools/TaskCreateTool.md) - Task creation
- [TaskGetTool](./02-tools/TaskGetTool.md) - Task retrieval
- [TaskListTool](./02-tools/TaskListTool.md) - Task listing
- [TaskOutputTool](./02-tools/TaskOutputTool.md) - Task output retrieval
- [TaskStopTool](./02-tools/TaskStopTool.md) - Task cancellation
- [TaskUpdateTool](./02-tools/TaskUpdateTool.md) - Task status updates
- [TeamCreateTool](./02-tools/TeamCreateTool.md) - Team creation
- [TeamDeleteTool](./02-tools/TeamDeleteTool.md) - Team deletion
- [TodoWriteTool](./02-tools/TodoWriteTool.md) - Todo list management
- [ToolSearchTool](./02-tools/ToolSearchTool.md) - Tool discovery
- [TungstenTool](./02-tools/TungstenTool.md) - Tungsten integration
- [WebFetchTool](./02-tools/WebFetchTool.md) - URL fetching
- [WebSearchTool](./02-tools/WebSearchTool.md) - Web search
- [KAIROS/Ant-only Tools](./02-tools/kairos-ant-tools.md) - Feature-gated tools

### 03 Commands (50+ commands)
- [Session Commands](./03-commands/session-commands.md) - Session lifecycle
- [Clear/Exit Commands](./03-commands/clear-exit-commands.md) - Session cleanup
- [Config Commands](./03-commands/config-commands.md) - Configuration
- [Settings Commands](./03-commands/settings-commands.md) - Runtime settings
- [Auth Commands](./03-commands/auth-commands.md) - Authentication
- [Review Commands](./03-commands/review-commands.md) - Code review
- [Branch/Issue Commands](./03-commands/branch-issue-commands.md) - Git & issues
- [MCP Commands](./03-commands/mcp-commands.md) - MCP management
- [Feature Commands](./03-commands/feature-commands.md) - Feature-specific
- [Interaction Commands](./03-commands/interaction-commands.md) - Interactive session
- [UI Commands](./03-commands/ui-commands.md) - Information display
- [Debug Commands](./03-commands/debug-commands.md) - Debugging
- [Remote Commands](./03-commands/remote-commands.md) - Remote sessions
- [Integration Commands](./03-commands/integration-commands.md) - External integrations
- [Misc Commands](./03-commands/misc-commands.md) - Miscellaneous

### 04 Services
- [API Service](./04-services/api-service.md) - API client
- [Analytics Service](./04-services/analytics-service.md) - Telemetry
- [OAuth Service](./04-services/oauth-service.md) - Authentication
- [MCP Service](./04-services/mcp-service.md) - MCP integration
- [LSP Service](./04-services/lsp-service.md) - Language Server Protocol
- [Compact Service](./04-services/compact-service.md) - Context compaction
- [Plugins Service](./04-services/plugins-service.md) - Plugin system
- [Settings Sync Service](./04-services/settings-sync-service.md) - Cloud sync
- [Team Memory Sync Service](./04-services/team-memory-sync-service.md) - Team memory
- [Policy Limits Service](./04-services/policy-limits-service.md) - Rate limiting
- [Remote Managed Settings](./04-services/remote-managed-settings.md) - MDM
- [Auto Dream Service](./04-services/auto-dream-service.md) - Proactive suggestions
- [Extract Memories Service](./04-services/extract-memories-service.md) - Memory extraction
- [Tips Service](./04-services/tips-service.md) - User guidance
- [Tool Use Summary Service](./04-services/tool-use-summary-service.md) - Usage tracking

### 05 Utils
- [Auth](./05-utils/auth.md) | [Config](./05-utils/config.md) | [Git](./05-utils/git-utils.md)
- [Permissions](./05-utils/permissions-utils.md) | [Settings](./05-utils/settings-utils.md)
- [Shell](./05-utils/shell-utils.md) | [Bash](./05-utils/bash-utils.md)
- [Model](./05-utils/model-utils.md) | [Memory](./05-utils/memory-utils.md)
- [MCP](./05-utils/mcp-utils.md) | [Skills](./05-utils/skills-utils.md)
- [Swarm](./05-utils/swarm-utils.md) | [Telemetry](./05-utils/telemetry-utils.md)
- [Task](./05-utils/task-utils.md) | [Todo](./05-utils/todo-utils.md)
- [Secure Storage](./05-utils/secure-storage.md) | [Sandbox](./05-utils/sandbox-utils.md)
- [GitHub](./05-utils/github-utils.md) | [File Persistence](./05-utils/file-persistence.md)
- [Hooks](./05-utils/hooks-utils.md) | [Effort/Fast Mode](./05-utils/effort-fast-mode.md)
- [Teleport](./05-utils/teleport-utils.md) | [Ultraplan](./05-utils/ultraplan-utils.md)
- [Computer Use](./05-utils/computer-use.md) | [Deep Link](./05-utils/deep-link.md)
- [Worktree Mode](./05-utils/worktree-mode.md) | [Advisor/Attribution](./05-utils/advisor-attribution.md)

### 06 UI
- [Ink Framework](./06-ui/ink-framework.md) - Terminal UI framework
- [Components Overview](./06-ui/components-overview.md) - React components
- [Design System](./06-ui/design-system.md) - Design tokens
- [Messages Components](./06-ui/messages-components.md) - Message rendering
- [Permissions Components](./06-ui/permissions-components.md) - Permission dialogs
- [Keybindings System](./06-ui/keybindings-system.md) - Keyboard shortcuts
- [Vim Mode](./06-ui/vim-mode.md) - Vim emulation
- [Spinner Components](./06-ui/spinner-components.md) - Loading indicators

### 07 Advanced Features
- [Coordinator Mode](./07-advanced-features/coordinator-mode.md) - Multi-agent coordination
- [Assistant Mode](./07-advanced-features/assistant-mode.md) - KAIROS assistant
- [Buddy System](./07-advanced-features/buddy-system.md) - AI companion
- [Plugins System](./07-advanced-features/plugins-system.md) - Plugin architecture
- [Skills System](./07-advanced-features/skills-system.md) - Skills framework
- [Voice Mode](./07-advanced-features/voice-mode.md) - Voice interaction

### 08 Internals
- [Bootstrap](./08-internals/bootstrap.md) - Application startup
- [Migrations](./08-internals/migrations.md) - Database migrations
- [Hooks System](./08-internals/hooks-system.md) - React hooks
- [Context Providers](./08-internals/context-providers.md) - React context
- [Native TS Modules](./08-internals/native-ts-modules.md) - Native modules
- [Schemas](./08-internals/schemas.md) - Zod schemas
- [Types System](./08-internals/types-system.md) - TypeScript types
- [Vendor Modules](./08-internals/vendor-modules.md) - Vendor code

### 09 Data Flow
- [Message Flow](./09-data-flow/message-flow.md) - User input to LLM response
- [Permission Flow](./09-data-flow/permission-flow.md) - Permission request lifecycle
- [Tool Execution Flow](./09-data-flow/tool-execution-flow.md) - Tool execution lifecycle
- [Session Lifecycle](./09-data-flow/session-lifecycle.md) - Session management

## Source

- **Original Package**: [@anthropic-ai/claude-code](https://www.npmjs.com/package/@anthropic-ai/claude-code)
- **Version**: 2.1.88
- **Source Files**: ~1884 TypeScript files
- **Restoration Method**: Extracted from `cli.js.map` sourcesContent field
