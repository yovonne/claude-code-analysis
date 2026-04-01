# Components Overview

## Purpose

Comprehensive catalog of Claude Code's React component library organized by domain. Components are built on the Ink framework (custom React terminal renderer) and the design system (theme-aware primitives).

## Location

`restored-src/src/components/` — main component directory with domain subdirectories.

## Component Categories

### Layout Components

| Component | Location | Description |
|-----------|----------|-------------|
| `FullscreenLayout` | `components/FullscreenLayout.tsx` | Root layout for alt-screen mode; manages terminal-size-constrained layout |
| `SessionPreview` | `components/SessionPreview.tsx` | Session preview overlay |
| `VirtualMessageList` | `components/VirtualMessageList.tsx` | Virtualized message list for performance with large message counts |
| `Messages` | `components/Messages.tsx` | Message container orchestrating message rendering |
| `MessageRow` | `components/MessageRow.tsx` | Individual message row wrapper |
| `MessageResponse` | `components/MessageResponse.tsx` | Response message display |
| `MessageModel` | `components/MessageModel.tsx` | Model information display |
| `MessageSelector` | `components/MessageSelector.tsx` | Message rewind/selection UI |
| `MessageTimestamp` | `components/MessageTimestamp.tsx` | Timestamp display |

### Input Components

| Component | Location | Description |
|-----------|----------|-------------|
| `PromptInput/` | `components/PromptInput/` | Main chat input area with autocomplete and action buttons |
| `TextInput` | `components/TextInput.tsx` | Basic text input |
| `BaseTextInput` | `components/BaseTextInput.tsx` | Base text input with common props |
| `VimTextInput` | `components/VimTextInput.tsx` | Text input with vim mode support |
| `SearchBox` | `components/SearchBox.tsx` | Search input with icon |
| `LanguagePicker` | `components/LanguagePicker.tsx` | Language selection dropdown |
| `ModelPicker` | `components/ModelPicker.tsx` | Model selection dialog |
| `OutputStylePicker` | `components/OutputStylePicker.tsx` | Output style selection |
| `CustomSelect/` | `components/CustomSelect/` | Reusable select/list component |

### Dialog Components

| Component | Location | Description |
|-----------|----------|-------------|
| `BridgeDialog` | `components/BridgeDialog.tsx` | Claude Code Bridge connection dialog |
| `ExportDialog` | `components/ExportDialog.tsx` | Session export dialog |
| `GlobalSearchDialog` | `components/GlobalSearchDialog.tsx` | Global file/command search |
| `QuickOpenDialog` | `components/QuickOpenDialog.tsx` | Quick file open dialog |
| `HistorySearchDialog` | `components/HistorySearchDialog.tsx` | Command history search (ctrl+r) |
| `MCPServerDialogCopy` | `components/MCPServerDialogCopy.tsx` | MCP server configuration |
| `MCPServerApprovalDialog` | `components/MCPServerApprovalDialog.tsx` | MCP server approval |
| `MCPServerMultiselectDialog` | `components/MCPServerMultiselectDialog.tsx` | MCP server multi-select |
| `MCPServerDesktopImportDialog` | `components/MCPServerDesktopImportDialog.tsx` | MCP server import from desktop |
| `WorkflowMultiselectDialog` | `components/WorkflowMultiselectDialog.tsx` | Workflow selection |
| `TeleportRepoMismatchDialog` | `components/TeleportRepoMismatchDialog.tsx` | Teleport repo mismatch warning |
| `RemoteEnvironmentDialog` | `components/RemoteEnvironmentDialog.tsx` | Remote environment config |
| `InvalidConfigDialog` | `components/InvalidConfigDialog.tsx` | Invalid configuration warning |
| `InvalidSettingsDialog` | `components/InvalidSettingsDialog.tsx` | Invalid settings warning |
| `ChannelDowngradeDialog` | `components/ChannelDowngradeDialog.tsx` | Channel downgrade confirmation |
| `DevChannelsDialog` | `components/DevChannelsDialog.tsx` | Dev channel selection |
| `CostThresholdDialog` | `components/CostThresholdDialog.tsx` | Cost threshold warning |
| `AutoModeOptInDialog` | `components/AutoModeOptInDialog.tsx` | Auto mode opt-in |
| `BypassPermissionsModeDialog` | `components/BypassPermissionsModeDialog.tsx` | Bypass permissions mode |
| `ClaudeMdExternalIncludesDialog` | `components/ClaudeMdExternalIncludesDialog.tsx` | External CLAUDE.md includes |
| `IdeAutoConnectDialog` | `components/IdeAutoConnectDialog.tsx` | IDE auto-connect onboarding |
| `IdeOnboardingDialog` | `components/IdeOnboardingDialog.tsx` | IDE onboarding |
| `IdleReturnDialog` | `components/IdleReturnDialog.tsx` | Return from idle state |
| `WorktreeExitDialog` | `components/WorktreeExitDialog.tsx` | Worktree exit confirmation |

### Status & Progress Components

| Component | Location | Description |
|-----------|----------|-------------|
| `Spinner/` | `components/Spinner/` | Spinner animations (detailed in separate doc) |
| `AgentProgressLine` | `components/AgentProgressLine.tsx` | Agent progress indicator |
| `BashModeProgress` | `components/BashModeProgress.tsx` | Bash mode progress |
| `CoordinatorAgentStatus` | `components/CoordinatorAgentStatus.tsx` | Coordinator agent status |
| `EffortIndicator` | `components/EffortIndicator.ts` | Effort level indicator |
| `EffortCallout` | `components/EffortCallout.tsx` | Effort level callout |
| `TeleportProgress` | `components/TeleportProgress.tsx` | Teleport operation progress |
| `ToolUseLoader` | `components/ToolUseLoader.tsx` | Tool execution loading indicator |
| `FastIcon` | `components/FastIcon.tsx` | Fast mode indicator |
| `StatusLine` | `components/StatusLine.tsx` | Status line display |
| `StatusNotices` | `components/StatusNotices.tsx` | Status notices |
| `MemoryUsageIndicator` | `components/MemoryUsageIndicator.tsx` | Memory usage display |
| `IdeStatusIndicator` | `components/IdeStatusIndicator.tsx` | IDE connection status |

### Onboarding & Feature Components

| Component | Location | Description |
|-----------|----------|-------------|
| `Onboarding` | `components/Onboarding.tsx` | First-run onboarding flow |
| `ClaudeInChromeOnboarding` | `components/ClaudeInChromeOnboarding.tsx` | Chrome integration onboarding |
| `DesktopHandoff` | `components/DesktopHandoff.tsx` | Desktop app handoff |
| `DesktopUpsell/` | `components/DesktopUpsell/` | Desktop app promotion |
| `PrBadge` | `components/PrBadge.tsx` | PR badge indicator |
| `CtrlOToExpand` | `components/CtrlOToExpand.tsx` | Ctrl+O expand hint |
| `PressEnterToContinue` | `components/PressEnterToContinue.tsx` | Enter to continue prompt |
| `RemoteCallout` | `components/RemoteCallout.tsx` | Remote environment callout |
| `SessionBackgroundHint` | `components/SessionBackgroundHint.tsx` | Session background hint |
| `ShowInIDEPrompt` | `components/ShowInIDEPrompt.tsx` | Show in IDE prompt |

### Dev & Debug Components

| Component | Location | Description |
|-----------|----------|-------------|
| `DevBar` | `components/DevBar.tsx` | Developer toolbar |
| `DiagnosticsDisplay` | `components/DiagnosticsDisplay.tsx` | Diagnostics information display |
| `SentryErrorBoundary` | `components/SentryErrorBoundary.ts` | Sentry error boundary wrapper |
| `KeybindingWarnings` | `components/KeybindingWarnings.tsx` | Keybinding conflict warnings |
| `ValidationErrorsList` | `components/ValidationErrorsList.tsx` | Validation error list |

### Navigation Components

| Component | Location | Description |
|-----------|----------|-------------|
| `TagTabs` | `components/TagTabs.tsx` | Tag-based tab navigation |
| `TaskListV2` | `components/TaskListV2.tsx` | Task list display |
| `LogSelector` | `components/LogSelector.tsx` | Log selection UI |
| `TeammateViewHeader` | `components/TeammateViewHeader.tsx` | Teammate view header |

### Exit & Flow Components

| Component | Location | Description |
|-----------|----------|-------------|
| `ExitFlow` | `components/ExitFlow.tsx` | Session exit flow |
| `InterruptedByUser` | `components/InterruptedByUser.tsx` | User interruption display |
| `ResumeTask` | `components/ResumeTask.tsx` | Task resume prompt |
| `TeleportStash` | `components/TeleportStash.tsx` | Teleport stash display |
| `TeleportResumeWrapper` | `components/TeleportResumeWrapper.tsx` | Teleport resume wrapper |
| `OffscreenFreeze` | `components/OffscreenFreeze.tsx` | Offscreen freeze indicator |

### Theme & Settings Components

| Component | Location | Description |
|-----------|----------|-------------|
| `ThemePicker` | `components/ThemePicker.tsx` | Theme selection UI |
| `Settings/` | `components/Settings/` | Settings panel components |
| `ManagedSettingsSecurityDialog` | `components/ManagedSettingsSecurityDialog/` | Managed settings security |

### Feedback Components

| Component | Location | Description |
|-----------|----------|-------------|
| `Feedback` | `components/Feedback.tsx` | Feedback submission |
| `FeedbackSurvey/` | `components/FeedbackSurvey/` | Feedback survey components |
| `SkillImprovementSurvey` | `components/SkillImprovementSurvey.tsx` | Skill improvement survey |

### Specialized Components

| Component | Location | Description |
|-----------|----------|-------------|
| `ConsoleOAuthFlow` | `components/ConsoleOAuthFlow.tsx` | Console OAuth authentication flow |
| `ApproveApiKey` | `components/ApproveApiKey.tsx` | API key approval dialog |
| `AwsAuthStatusBox` | `components/AwsAuthStatusBox.tsx` | AWS authentication status |
| `ContextVisualization` | `components/ContextVisualization.tsx` | Context window visualization |
| `ContextSuggestions` | `components/ContextSuggestions.tsx` | Context suggestions |
| `TokenWarning` | `components/TokenWarning.tsx` | Token usage warning |
| `Stats` | `components/Stats.tsx` | Session statistics |
| `ThinkingToggle` | `components/ThinkingToggle.tsx` | Extended thinking toggle |
| `AutoUpdater` | `components/AutoUpdater.tsx` | Auto-update checker |
| `AutoUpdaterWrapper` | `components/AutoUpdaterWrapper.tsx` | Auto-update wrapper |
| `NativeAutoUpdater` | `components/NativeAutoUpdater.tsx` | Native auto-updater |
| `PackageManagerAutoUpdater` | `components/PackageManagerAutoUpdater.tsx` | Package manager update checker |
| `ClickableImageRef` | `components/ClickableImageRef.tsx` | Clickable image reference |
| `FilePathLink` | `components/FilePathLink.tsx` | File path link |
| `ConfigurableShortcutHint` | `components/ConfigurableShortcutHint.tsx` | Configurable shortcut display |
| `ScrollKeybindingHandler` | `components/ScrollKeybindingHandler.tsx` | Scroll keybinding handler |
| `SandboxViolationExpandedView` | `components/SandboxViolationExpandedView.tsx` | Sandbox violation details |
| `TeleportError` | `components/TeleportError.tsx` | Teleport error display |

## Domain Subdirectories

| Directory | Purpose |
|-----------|---------|
| `agents/` | Agent-related components |
| `design-system/` | Theme-aware UI primitives (Box, Text, Dialog, Tabs, etc.) |
| `diff/` | Diff display components |
| `grove/` | Grove-related components |
| `hooks/` | Shared React hooks |
| `mcp/` | MCP server UI components |
| `memory/` | Memory-related components |
| `messages/` | Message type components (detailed in separate doc) |
| `permissions/` | Permission request components (detailed in separate doc) |
| `sandbox/` | Sandbox-related components |
| `shell/` | Shell output components |
| `skills/` | Skill-related components |
| `tasks/` | Task-related components |
| `teams/` | Team-related components |
| `ui/` | General UI components |
| `wizard/` | Wizard flow components |
| `ClaudeCodeHint/` | Claude Code hint components |
| `HelpV2/` | Help overlay V2 |
| `HighlightedCode/` | Syntax-highlighted code display |
| `LogoV2/` | Logo V2 component |
| `LspRecommendation/` | LSP recommendation UI |
| `Passes/` | Pass-related components |
| `StructuredDiff/` | Structured diff display |
| `TrustDialog/` | Trust dialog components |

## Dependencies

### Internal Dependencies

- `../ink.js` — Ink framework (Box, Text, useInput, useTheme, etc.)
- `../keybindings/` — Keybinding system (useKeybinding)
- `../utils/` — Utilities (theme, format, platform, etc.)
- `../hooks/` — Shared hooks
- `../bootstrap/` — Bootstrap state
- `../tasks/` — Task types and state
- `../services/` — Service layer

### External Dependencies

- `react` — React 19
- `figures` — Unicode symbols for terminal UI
- `strip-ansi` — ANSI escape removal

## Key Design Patterns

1. **Theme-aware primitives**: All UI components use `ThemedBox` and `ThemedText` from the design system, accepting theme keys instead of raw colors
2. **Keybinding integration**: Dialogs and interactive components use `useKeybinding` for context-aware keyboard shortcuts
3. **Progressive disclosure**: Complex dialogs show basic info first, with expandable details
4. **Virtual rendering**: Large lists (VirtualMessageList) use viewport culling for performance
5. **Animation isolation**: Spinner components separate animation loops from parent renders to minimize re-render scope
6. **Context providers**: Domain-specific contexts (TabsContext, TextHoverColorContext) for cross-component coordination
7. **Feature flags**: Components conditionally render based on `bun:bundle` feature flags and Statsig gates
