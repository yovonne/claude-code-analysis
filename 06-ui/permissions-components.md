# Permissions Components

## Purpose

The permissions component system renders interactive permission request dialogs that allow users to approve, deny, or configure access for various operations (file system, shell commands, network access, sandbox operations, etc.). Each permission type has a specialized component with context-appropriate explanations and action buttons.

## Location

`restored-src/src/components/permissions/`

## Key Components

### Permission Request Components

| Component | Description |
|-----------|-------------|
| `BashPermissionRequest/` | Shell command execution permission |
| `FileEdit.PermissionRequest/` | File edit operation permission |
| `FileWritePermissionRequest/` | File write operation permission |
| `FilesystemPermissionRequest/` | File system access permission |
| `SandboxPermissionRequest.tsx` | Sandbox operation permission |
| `SedEditPermissionRequest/` | Sed edit operation permission |
| `NotebookEditPermissionRequest/` | Notebook edit permission |
| `SkillPermissionRequest/` | Skill execution permission |
| `WebFetchPermissionRequest/` | Web fetch/network permission |
| `AskUserQuestionPermissionRequest/` | Ask user question permission |
| `EnterPlanModePermissionRequest/` | Enter plan mode permission |
| `ExitPlanModePermissionRequest/` | Exit plan mode permission |
| `PowerShellPermissionRequest/` | PowerShell execution permission |
| `ComputerUseApproval/` | Computer use approval |
| `FilePermissionDialog/` | Generic file permission dialog |

### Core Permission Components

| Component | Description |
|-----------|-------------|
| `PermissionPrompt.tsx` | Main permission prompt orchestrator |
| `PermissionDialog.tsx` | Base permission dialog wrapper |
| `PermissionRequest.tsx` | Permission request container |
| `PermissionRequestTitle.tsx` | Permission request title display |
| `PermissionExplanation.tsx` | Permission explanation text |
| `PermissionRuleExplanation.tsx` | Rule-based permission explanation |
| `PermissionDecisionDebugInfo.tsx` | Debug info for permission decisions |
| `FallbackPermissionRequest.tsx` | Fallback for unknown permission types |

### Worker/Teammate Components

| Component | Description |
|-----------|-------------|
| `WorkerBadge.tsx` | Worker identity badge |
| `WorkerPendingPermission.tsx` | Pending permission indicator for workers |

### Hooks & Utilities

| Export | Description |
|--------|-------------|
| `hooks.ts` | Permission-related React hooks |
| `utils.ts` | Permission utility functions |
| `shellPermissionHelpers.tsx` | Shell-specific permission helpers |
| `useShellPermissionFeedback.ts` | Shell permission feedback hook |
| `rules/` | Permission rule definitions |

## Dependencies

### Internal Dependencies

- `../ink.js` — Ink framework (Box, Text, etc.)
- `../design-system/` — Dialog, Pane, and other primitives
- `../TextInput.tsx` / `../VimTextInput.tsx` — Text input for permission responses
- `../ConfigurableShortcutHint.tsx` — Shortcut hints
- `../design-system/KeyboardShortcutHint.tsx` — Keyboard shortcut display
- `../design-system/Byline.tsx` — Action hints
- `../../keybindings/useKeybinding.js` — Keybinding integration
- `../../utils/` — Permission utilities, formatting, theme
- `../../hooks/` — Shared hooks (useExitOnCtrlCDWithKeybindings)
- `../../bootstrap/` — Bootstrap state
- `../../permissions/` — Permission logic and types

### External Dependencies

- `react` — React 19
- `figures` — Unicode symbols

## Permission Architecture

### Permission Flow

```
Tool Execution Request
    ↓
Permission System (determines if permission needed)
    ↓
PermissionPrompt (orchestrator)
    ↓
[Specific PermissionRequest component by type]
    ↓
PermissionDialog (base dialog wrapper)
    ↓
PermissionExplanation (context + reasoning)
    ↓
Action Buttons (Allow, Deny, Allow Always, etc.)
    ↓
User Decision → Tool Execution or Rejection
```

### Permission Request Pattern

Each permission request component follows this pattern:

1. **Title**: Describes what operation is being requested
2. **Explanation**: Why the permission is needed and what it allows
3. **Details**: Operation-specific details (file paths, commands, URLs)
4. **Actions**: Allow, Deny, Allow Always (context-dependent)
5. **Rule info**: Shows if a permission rule applies and why

### Permission Types

- **File operations**: Read, write, edit specific files or directories
- **Shell commands**: Execute bash/PowerShell commands with specific arguments
- **Network access**: Fetch URLs, make HTTP requests
- **Sandbox operations**: Operations requiring sandbox escape
- **Skill execution**: Running skills with specific permissions
- **Plan mode**: Entering/exiting planning mode
- **Computer use**: Screen/keyboard/mouse control approval

### Decision Options

Permission dialogs typically offer:
- **Allow once**: Single execution approval
- **Deny**: Reject this request
- **Allow always**: Create a permanent rule (when applicable)
- **Session allow**: Allow for current session only
- **Remember decision**: Save as a rule for future similar requests

### Shell Permission Handling

Shell permissions have specialized handling:
- **Command display**: Shows the exact command to be executed
- **Argument highlighting**: Highlights potentially dangerous arguments
- **Feedback hook**: `useShellPermissionFeedback` tracks user decisions
- **Helper functions**: `shellPermissionHelpers.tsx` provides common shell permission logic

### Worker Permissions

When worker agents need permissions:
- **WorkerBadge**: Shows which worker is requesting permission
- **WorkerPendingPermission**: Visual indicator of pending worker permissions
- **Delegation**: Permissions can be delegated from leader to workers

## Key Design Decisions

1. **Type-specific components**: Each permission type has its own component for appropriate context and explanation
2. **Rule transparency**: Shows why a permission rule applies or doesn't apply
3. **Progressive disclosure**: Basic info first, expandable details for power users
4. **Debug info**: `PermissionDecisionDebugInfo` helps diagnose permission decisions
5. **Fallback handling**: `FallbackPermissionRequest` handles unknown permission types gracefully
6. **Worker awareness**: Distinct rendering for worker vs. leader permission requests
7. **Keybinding integration**: Permission dialogs use context-aware keyboard shortcuts
8. **Session persistence**: "Allow always" creates persistent rules stored in config
