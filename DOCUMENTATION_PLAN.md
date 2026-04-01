# Claude Code Source Documentation Plan

## Overview

This document outlines the comprehensive documentation strategy for the Claude Code CLI source code (version 2.1.88), reconstructed from npm package source maps.

### Codebase Statistics
- **TypeScript Files**: ~1884
- **Directories**: ~200
- **Tools**: 35+
- **Commands**: 50+
- **Services**: 15+
- **UI Components**: 50+

---

## Documentation Structure

```
analysis/
├── DOCUMENTATION_PLAN.md            # This file
├── task_breakdown.json              # Task tracking
├── README.md                        # Documentation index
│
├── 00-architecture-overview.md      # High-level architecture
│
├── 01-core-modules/
│   ├── main-entrypoint.md           # main.tsx functional sections
│   ├── tool-system.md               # Tool abstraction & registry
│   ├── query-engine.md              # QueryEngine.ts
│   ├── command-system.md            # Command dispatch
│   └── state-management.md          # AppState & state flow
│
├── 02-tools/                        # Individual tool docs
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
├── 03-commands/                     # CLI command docs
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

## Execution Phases

### Phase 1: Core Architecture (5 files)
Analyze the fundamental building blocks of the application.

| File | Source | Description |
|------|--------|-------------|
| 00-architecture-overview.md | All | High-level system architecture |
| main-entrypoint.md | main.tsx | CLI entry, initialization, command parsing |
| tool-system.md | Tool.ts, tools.ts | Tool abstraction, registry, execution |
| query-engine.md | QueryEngine.ts | Query processing, LLM interaction |
| state-management.md | AppState.tsx | Application state, React context |

### Phase 2: Tools (48 files)
Document each tool individually with implementation details.

Categories:
- **File Operations**: FileReadTool, FileWriteTool, FileEditTool, GlobTool, GrepTool, NotebookEditTool
- **Shell Execution**: BashTool, PowerShellTool, REPLTool
- **Agent/Task**: AgentTool, TaskCreateTool, TaskGetTool, TaskListTool, TaskOutputTool, TaskStopTool, TaskUpdateTool
- **Team**: TeamCreateTool, TeamDeleteTool, SendMessageTool
- **MCP**: MCPTool, McpAuthTool, ListMcpResourcesTool, ReadMcpResourceTool
- **Planning**: EnterPlanModeTool, ExitPlanModeTool, EnterWorktreeTool, ExitWorktreeTool
- **User Interaction**: AskUserQuestionTool, SkillTool, BriefTool
- **Web**: WebFetchTool, WebSearchTool
- **System**: ConfigTool, LSPTool, TodoWriteTool, ToolSearchTool, SyntheticOutputTool
- **Scheduled**: CronCreateTool, CronDeleteTool, CronListTool, RemoteTriggerTool
- **KAIROS/Ant-only**: SleepTool, MonitorTool, SendUserFileTool, PushNotificationTool, SubscribePRTool, SuggestBackgroundPRTool, VerifyPlanExecutionTool
- **Other**: TungstenTool

### Phase 3: Commands (50 files)
Document all CLI commands with usage, options, and implementation.

### Phase 4: Services (16 files)
Document the service layer for external integrations.

### Phase 5: Utils (30 files)
Document utility modules and helper functions.

### Phase 6: UI (8 files)
Document terminal UI components and the Ink framework integration.

### Phase 7: Advanced Features (6 files)
Document experimental and advanced feature modules.

### Phase 8: Internals (8 files)
Document internal infrastructure modules.

### Phase 9: Data Flow (4 files)
Document key data flows and system interactions.

---

## Documentation Template

Each module documentation follows this structure:

```markdown
# ModuleName

## Purpose
Brief description of what this module does and its role in the system.

## Location
`restored-src/src/path/to/module.ts`

## Key Exports

### Functions
- `functionName`: Description of what it does

### Classes
- `ClassName`: Description of the class

### Types
- `TypeName`: Description of the type

### Constants
- `CONSTANT_NAME`: Description and value

## Dependencies

### Internal Dependencies
- Module A - Purpose
- Module B - Purpose

### External Dependencies
- Package X - Purpose

## Implementation Details

### Core Logic
Explanation of the main implementation approach.

### Key Algorithms
Any important algorithms or patterns used.

### Edge Cases
Special handling or edge cases.

## Data Flow
Description or diagram of how data flows through this module.

## Integration Points
How this module integrates with other parts of the system.

## Configuration
Any configuration options or environment variables.

## Error Handling
How errors are handled and reported.

## Testing
Testing approach if applicable.

## Related Modules
- [Module A](./path/to/module-a.md)
- [Module B](./path/to/module-b.md)

## Notes
Any additional notes or caveats.
```

---

## Naming Conventions

- **Files**: Lowercase with hyphens, matching the module name
- **Headings**: Title case for main sections
- **Code references**: Use backticks for file paths, function names, and code
- **Links**: Relative links between documentation files

---

## Progress Tracking

Progress is tracked in `task_breakdown.json` with the following states:
- `pending`: Not yet started
- `in_progress`: Currently being worked on
- `completed`: Finished and verified
- `blocked`: Waiting on dependency

---

## Notes

- All documentation is in English
- main.tsx documentation is broken into functional sections
- Each tool gets its own dedicated file
- Cross-references link related modules
