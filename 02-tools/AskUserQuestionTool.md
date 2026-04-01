# AskUserQuestionTool

## Purpose

The AskUserQuestionTool is the primary mechanism for Claude to present multiple-choice questions to the user during task execution. It enables interactive dialogs where the model can gather user preferences, clarify ambiguous instructions, get decisions on implementation choices, or offer directional choices. The tool supports 1–4 questions per invocation, each with 2–4 options, optional preview content for visual comparison, multi-select mode, and free-text "Other" input. It is the core of the interactive user prompting experience in the TUI.

## Location

- `restored-src/src/tools/AskUserQuestionTool/AskUserQuestionTool.tsx`
- `restored-src/src/tools/AskUserQuestionTool/prompt.ts`
- `restored-src/src/components/permissions/AskUserQuestionPermissionRequest/` (UI components)

## Key Exports

### Functions

- `AskUserQuestionTool`: The main tool definition built via `buildTool()` — handles question validation, permission checks, and response collection
- `validateHtmlPreview()`: Validates HTML preview fragments for safety (no `<script>`, `<style>`, full documents)

### Types

- `Question`: A single question with `question`, `header`, `options`, and `multiSelect` fields
- `QuestionOption`: An option within a question with `label`, `description`, and optional `preview`
- `Output`: The tool's output schema containing `questions`, `answers` (question text → answer string), and optional `annotations`

### Constants

- `ASK_USER_QUESTION_TOOL_NAME`: `'AskUserQuestion'`
- `ASK_USER_QUESTION_TOOL_CHIP_WIDTH`: `12` — max chars for the header chip/tag
- `DESCRIPTION`: Tool description summarizing its purpose
- `ASK_USER_QUESTION_TOOL_PROMPT`: Instructions for the LLM on when and how to use the tool
- `PREVIEW_FEATURE_PROMPT`: Format-specific guidance for `markdown` and `html` preview modes

## Dependencies

### Internal Dependencies

- `prompt.ts` — Tool prompt, descriptions, and preview feature guidance
- `PermissionRequest.tsx` — Base permission request component
- `AskUserQuestionPermissionRequest/` — Full interactive dialog UI (7 components)
- `use-multiple-choice-state.ts` — State management hook for multi-question navigation
- `CustomSelect/index.js` — `Select` and `SelectMulti` components for option selection
- `PreviewBox.tsx` — Bordered monospace box for rendering preview content
- `QuestionView.tsx` — Standard question view (options list + text input)
- `PreviewQuestionView.tsx` — Side-by-side view (options left, preview right)
- `QuestionNavigationBar.tsx` — Tab bar for navigating between questions
- `SubmitQuestionsView.tsx` — Review-and-submit screen for final answer confirmation

### External Dependencies

- `zod/v4` — Schema validation with lazy schemas
- `react` — UI component rendering with React Compiler
- `bun:bundle` — Feature flag gating (`KAIROS`, `KAIROS_CHANNELS`)

## Implementation Details

### Input Schema

The tool uses Zod strict object validation with the following structure:

```typescript
{
  questions: [
    {
      question: string,       // The complete question (must end with ?)
      header: string,         // Short label for chip/tag (max 12 chars)
      options: [              // 2–4 options per question
        {
          label: string,       // Display text (1–5 words)
          description: string, // Explanation of trade-offs/implications
          preview?: string     // Optional: markdown or HTML preview content
        }
      ],
      multiSelect: boolean     // Allow multiple selections (default: false)
    }
  ],                       // 1–4 questions per invocation
  answers?: Record<string, string>,       // Populated by permission component
  annotations?: Record<string, {          // Per-question annotations
    preview?: string,       // Preview content of selected option
    notes?: string          // Free-text notes added by user
  }>,
  metadata?: {
    source?: string         // e.g., "remember" for /remember command
  }
}
```

**Validation constraints:**
- Questions must be unique (no duplicate question text)
- Option labels must be unique within each question
- Each question must have 2–4 options
- No "Other" option allowed (provided automatically by the UI)
- HTML preview mode: fragments only — no `<html>`, `<body>`, `<script>`, `<style>`, or `<!DOCTYPE>`

### Output Schema

```typescript
{
  questions: Question[],                    // The questions that were asked
  answers: Record<string, string>,          // question text → answer string
  annotations?: Record<string, {            // Optional per-question annotations
    preview?: string,
    notes?: string
  }>
}
```

Multi-select answers are comma-separated strings (e.g., `"Option A, Option B"`).

### Permission Behavior

The tool always uses `'ask'` behavior — it requires explicit user interaction:

```typescript
async checkPermissions(input) {
  return {
    behavior: 'ask',
    message: 'Answer questions?',
    updatedInput: input
  };
}
```

The tool is marked as `requiresUserInteraction() => true` and `shouldDefer: true`, meaning it blocks execution until the user responds. It is also `isReadOnly() => true` and `isConcurrencySafe() => true`.

### Channel Availability

The tool is disabled when running in channel mode (Telegram/Discord) because there is no user at the keyboard to answer:

```typescript
isEnabled() {
  if ((feature('KAIROS') || feature('KAIROS_CHANNELS')) && getAllowedChannels().length > 0) {
    return false;
  }
  return true;
}
```

### Preview Feature

Options can include a `preview` field that renders concrete artifacts for visual comparison. Two formats are supported:

| Format | Rendering | Use Cases |
|--------|-----------|-----------|
| `markdown` | Monospace box with markdown rendering | ASCII mockups, code snippets, config examples |
| `html` | HTML fragment with inline styles | UI mockups, formatted comparisons, diagrams |

When any option has a preview, the UI switches to a side-by-side layout:
- **Left panel**: Vertical option list (30 chars wide)
- **Right panel**: Preview content + notes field

Previews are only supported for single-select questions (not `multiSelect`).

### Response Handling

When the user submits answers, the tool:

1. Collects answers into a `Record<string, string>` keyed by question text
2. Builds annotations from selected option previews and user notes
3. Passes the updated input back through `toolUseConfirm.onAllow()`
4. The `call()` method returns the questions, answers, and annotations
5. `mapToolResultToToolResultBlockParam()` formats the result for the LLM:

```
User has answered your questions: "Question 1"="Answer 1" ..., You can now continue with the user's answers in mind.
```

## UI Components

### AskUserQuestionPermissionRequest

The top-level component that orchestrates the interactive dialog. It:

1. Parses and validates the tool input via `inputSchema.safeParse()`
2. Calculates optimal content dimensions based on terminal size
3. Manages multi-question state via `useMultipleChoiceState()` hook
4. Handles image paste attachments per question
5. Routes between `QuestionView` (standard) and `PreviewQuestionView` (side-by-side) based on whether the current question has preview options
6. Shows `SubmitQuestionsView` when all questions are answered (or when navigating past the last question)

**Key handlers:**
- `handleQuestionAnswer` — Records an answer, auto-advances for single-question single-select
- `handleFinalResponse` — Submit or cancel from the review screen
- `handleRespondToClaude` — Sends answers back with a "clarify these questions" feedback message
- `handleFinishPlanInterview` — In plan mode, signals the interview is complete
- `handleCancel` — Rejects the questions entirely

### QuestionView

The standard question view without previews. Displays:
- Question navigation bar (tabs with answered checkboxes)
- Question title
- `Select` or `SelectMulti` component with options + "Other" text input
- Footer actions: "Chat about this" and (in plan mode) "Skip interview and plan immediately"
- Help text with keyboard shortcuts

Supports:
- Keyboard navigation (up/down, ctrl+p/ctrl+n)
- Text input mode with external editor (ctrl+g)
- Image paste attachments
- Auto-advance on single select

### PreviewQuestionView

Side-by-side layout for questions with preview content:
- **Left panel** (30 chars): Numbered options with focus pointer and selection checkmark
- **Right panel**: `PreviewBox` rendering the focused option's preview + notes field

Navigation:
- Up/down arrows navigate options
- Enter selects the focused option
- `n` key focuses the notes input
- Escape exits notes input or cancels
- Number keys (1–9) jump to specific options

### PreviewBox

A bordered monospace box component for rendering preview content:
- Renders markdown with syntax highlighting
- Truncates content that exceeds `maxLines` with a truncation indicator
- Calculates box dimensions based on available terminal space
- Uses box-drawing characters (`┌`, `┐`, `└`, `┘`, `─`, `│`) for borders

### QuestionNavigationBar

A tab bar displayed above each question showing:
- Question headers as clickable tabs with checkbox indicators (✓ for answered)
- Navigation arrows (← →) for multi-question sets
- A "Submit" tab (hidden for single-question, single-select)
- Dynamic width allocation — current question gets priority width, others share remaining space
- Truncation when terminal is too narrow

### SubmitQuestionsView

The review-and-submit screen shown after answering all questions:
- Lists all answered questions with their selected answers
- Warns if not all questions have been answered
- Shows permission rule explanation
- Provides "Submit answers" or "Cancel" options

### useMultipleChoiceState Hook

A `useReducer`-based state manager for multi-question dialogs:

```typescript
type State = {
  currentQuestionIndex: number;
  answers: Record<string, AnswerValue>;
  questionStates: Record<string, QuestionState>;
  isInTextInput: boolean;
}
```

**Actions:**
- `next-question` — Advance to next question
- `prev-question` — Go back to previous question (clamped to 0)
- `update-question-state` — Update a question's internal state (selected value, text input)
- `set-answer` — Record an answer with optional auto-advance
- `set-text-input-mode` — Toggle text input focus state

**Exposed API:**
- `currentQuestionIndex`, `answers`, `questionStates`, `isInTextInput`
- `nextQuestion()`, `prevQuestion()`
- `updateQuestionState(questionText, updates, isMultiSelect)`
- `setAnswer(questionText, answer, shouldAdvance?)`
- `setTextInputMode(isInInput)`

## Plan Mode Integration

In plan mode, the tool is used during the interview phase to clarify requirements:

- Questions should clarify requirements or choose between approaches **before** finalizing the plan
- The tool must NOT be used for plan approval — `ExitPlanModeTool` handles that
- Questions must not reference "the plan" because the user cannot see it until `ExitPlanModeTool` is called
- The footer provides "Chat about this" (send clarification back to Claude) and "Skip interview and plan immediately" options
- When `metadata.source === "remember"`, analytics events track the `/remember` command flow

## Keyboard Shortcuts

| Key | Action |
|-----|--------|
| `↑` / `Ctrl+P` | Navigate up (options or footer) |
| `↓` / `Ctrl+N` | Navigate down (options or footer) |
| `Enter` | Select focused option / Execute footer action |
| `Escape` | Cancel / Exit notes input |
| `Tab` | Switch between questions (multi-question) |
| `1`–`9` | Jump to numbered option (preview mode) |
| `n` | Focus notes input (preview mode) |
| `Ctrl+G` | Open external editor for notes |

## Error Handling

### Input Validation Errors

| Error Code | Message |
|------------|---------|
| 1 | HTML preview validation failure (invalid fragment, disallowed tags) |

### Schema Validation

- Duplicate question texts → refinement error
- Duplicate option labels within a question → refinement error
- Questions outside 1–4 range → Zod error
- Options outside 2–4 range → Zod error

## Data Flow

```
Model calls AskUserQuestion tool with questions
    │
    ▼
checkPermissions() → behavior: 'ask'
    │
    ▼
AskUserQuestionPermissionRequest renders
    │
    ├── Parses input via inputSchema.safeParse()
    ├── useMultipleChoiceState() initializes state
    │
    ├── For each question:
    │   ├── has preview? → PreviewQuestionView (side-by-side)
    │   └── no preview?  → QuestionView (standard)
    │
    ├── User navigates, selects options, adds notes
    │
    ├── All answered → SubmitQuestionsView (review)
    │
    └── User action:
        ├── Submit → onAllow(updatedInput with answers + annotations)
        ├── Cancel → onReject()
        ├── Chat about this → onReject(feedback, imageBlocks)
        └── Finish interview → onReject(feedback, imageBlocks)
    │
    ▼
call() returns { questions, answers, annotations }
    │
    ▼
mapToolResultToToolResultBlockParam() formats for LLM
```

## Integration Points

- **Permission System**: Uses `requiresUserInteraction()` to block execution until user responds
- **Plan Mode**: Integrated with plan interview phase (`isPlanModeInterviewPhaseEnabled`)
- **Image Paste**: Supports pasting images per question, converted to `ImageBlockParam`
- **External Editor**: Notes can be edited in the user's configured IDE via `editPromptInEditor()`
- **Analytics**: Events logged for accepted, rejected, responded, and finished-plan actions
- **Channel Permissions**: Disabled in channel mode (Telegram/Discord) via `isEnabled()`
- **SDK**: Public schemas exported as `_sdkInputSchema` and `_sdkOutputSchema` for SDK consumers

## Related Modules

- [ExitPlanModeTool](./ExitPlanModeTool.md) — Plan approval (not to be confused with question asking)
- [PermissionRequest](../components/permissions/PermissionRequest.md) — Base permission request component
- [PermissionDialog](../components/permissions/PermissionDialog.md) — Permission dialog orchestration
- [SkillTool](./SkillTool.md) — Skills that may trigger questions during execution
