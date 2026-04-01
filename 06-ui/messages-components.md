# Messages Components

## Purpose

The messages component system renders the diverse message types in Claude Code's chat interface. Each message type has a specialized component that handles its specific content format, styling, and interactions — from user prompts and assistant responses to tool use, system notifications, and team collaboration messages.

## Location

`restored-src/src/components/messages/`

## Key Components

### User Message Types

| Component | Description |
|-----------|-------------|
| `UserPromptMessage` | User text input/prompt messages |
| `UserTextMessage` | Plain text user messages |
| `UserCommandMessage` | Slash command messages |
| `UserChannelMessage` | Channel/system messages from user |
| `UserBashInputMessage` | User bash/shell input |
| `UserBashOutputMessage` | Bash command output |
| `UserImageMessage` | User-submitted image messages |
| `UserMemoryInputMessage` | User memory input |
| `UserPlanMessage` | User plan-related messages |
| `UserResourceUpdateMessage` | Resource update notifications |
| `UserToolResultMessage/` | Tool result display (subdirectory) |
| `UserTeammateMessage` | Messages from teammate agents |
| `UserLocalCommandOutputMessage` | Local command output display |

### Assistant Message Types

| Component | Description |
|-----------|-------------|
| `AssistantTextMessage` | Assistant text response |
| `AssistantThinkingMessage` | Extended thinking display |
| `AssistantRedactedThinkingMessage` | Redacted thinking content |
| `AssistantToolUseMessage` | Tool use display with status |

### System & Special Messages

| Component | Description |
|-----------|-------------|
| `SystemTextMessage` | System notification text |
| `SystemAPIErrorMessage` | API error display |
| `ShutdownMessage` | Session shutdown notification |
| `RateLimitMessage` | Rate limit warning |
| `PlanApprovalMessage` | Plan approval request |
| `TaskAssignmentMessage` | Task assignment notification |
| `HookProgressMessage` | Hook execution progress |
| `AdvisorMessage` | Advisor suggestions |

### Content Components

| Component | Description |
|-----------|-------------|
| `GroupedToolUseContent` | Groups multiple tool use entries |
| `HighlightedThinkingText` | Syntax-highlighted thinking text |
| `CollapsedReadSearchContent` | Collapsed search result content |
| `CompactBoundaryMessage` | Compact mode boundary marker |
| `AttachmentMessage` | File attachment display |

### Team Collaboration

| Component | Description |
|-----------|-------------|
| `teamMemCollapsed` | Collapsed teammate message |
| `teamMemSaved` | Saved teammate state indicator |

### Utilities

| Export | Description |
|--------|-------------|
| `nullRenderingAttachments` | Attachment rendering utilities |

## Dependencies

### Internal Dependencies

- `../ink.js` — Ink framework (Box, Text, etc.)
- `../design-system/` — Theme-aware primitives
- `../Spinner/` — Loading indicators for tool messages
- `../HighlightedCode/` — Syntax-highlighted code blocks
- `../StructuredDiff/` — Diff display for file edits
- `../messageActions.tsx` — Message action buttons
- `../Markdown.tsx` / `../MarkdownTable.tsx` — Markdown rendering
- `../FilePathLink.tsx` — File path links
- `../FallbackToolUseErrorMessage.tsx` — Tool error fallback
- `../FallbackToolUseRejectedMessage.tsx` — Tool rejection fallback
- `../FileEditToolDiff.tsx` — File edit diff display
- `../FileEditToolUpdatedMessage.tsx` — File edit update notification
- `../FileEditToolUseRejectedMessage.tsx` — File edit rejection
- `../NotebookEditToolUseRejectedMessage.tsx` — Notebook edit rejection
- `../SandboxViolationExpandedView.tsx` — Sandbox violation details
- `../ToolUseLoader.tsx` — Tool loading indicator
- `../ThinkingToggle.tsx` — Thinking toggle
- `../../utils/` — Formatting, theme, and utility functions
- `../../hooks/` — Shared React hooks

### External Dependencies

- `react` — React 19
- `figures` — Unicode symbols

## Message Architecture

### Message Flow

```
Message (container)
    ↓
MessageResponse (response wrapper)
    ↓
MessageRow (row layout)
    ↓
[Specific message component by type]
    ↓
Content renderers (Markdown, Code, Diff, etc.)
```

### Message Type Routing

Messages are routed to their appropriate component based on:
- **Message role**: user, assistant, system
- **Message subtype**: text, command, tool-use, image, etc.
- **Content format**: plain text, markdown, structured data

### Tool Message Handling

Tool messages have specialized handling:
- **Loading state**: Shows `ToolUseLoader` spinner during execution
- **Success**: Displays formatted output (diff, text, structured data)
- **Error**: Shows `FallbackToolUseErrorMessage` with error details
- **Rejection**: Shows `FallbackToolUseRejectedMessage` with rejection reason
- **File edits**: Uses `StructuredDiff` for visual diff display
- **Grouped tools**: `GroupedToolUseContent` collapses multiple tool calls

### Thinking Display

Extended thinking messages:
- **Active thinking**: `AssistantThinkingMessage` with shimmer animation
- **Completed thinking**: Shows duration and collapsed content
- **Redacted thinking**: `AssistantRedactedThinkingMessage` for hidden content
- **Toggle**: `ThinkingToggle` allows expanding/collapsing

### Team Messages

Teammate agent messages:
- **Collapsed**: `teamMemCollapsed` shows compact view
- **Expanded**: Full message with agent identity and status
- **Saved state**: `teamMemSaved` indicates persisted work

## Key Design Decisions

1. **Type-specific components**: Each message type has its own component for clear separation of concerns
2. **Progressive rendering**: Tool messages show loading state, then transition to result
3. **Markdown support**: Rich text rendering via Markdown component with code highlighting
4. **Diff integration**: File edits use structured diff display for visual clarity
5. **Compact mode**: Boundary messages and collapsed views support compact display
6. **Team awareness**: Distinct rendering for teammate vs. leader agent messages
7. **Error resilience**: Fallback components for malformed or failed tool messages
8. **Attachment support**: Image and file attachments rendered inline with previews
