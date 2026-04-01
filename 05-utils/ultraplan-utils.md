# Ultraplan Utils

## Purpose

Utilities for the `/ultraplan` feature — polling remote CCR (Claude Code Remote) sessions for plan approval, scanning event streams for ExitPlanMode tool results, and detecting "ultraplan" / "ultrareview" keyword triggers in user input.

## Location

- `restored-src/src/utils/ultraplan/ccrSession.ts` — CCR session polling for plan approval
- `restored-src/src/utils/ultraplan/keyword.ts` — Keyword trigger detection for ultraplan/ultrareview

## Key Exports

### CCR Session Polling (`ccrSession.ts`)

#### Types
- `PollFailReason`: `'terminated' | 'timeout_pending' | 'timeout_no_plan' | 'extract_marker_missing' | 'network_or_unknown' | 'stopped'`
- `UltraplanPollError`: Custom error with `reason` (PollFailReason) and `rejectCount` properties
- `ScanResult`: Discriminated union — `{ kind: 'approved', plan } | { kind: 'teleport', plan } | { kind: 'rejected', id } | { kind: 'pending' } | { kind: 'terminated', subtype } | { kind: 'unchanged' }`
- `UltraplanPhase`: `'running' | 'needs_input' | 'plan_ready'` — state machine phases
- `PollResult`: `{ plan, rejectCount, executionTarget: 'local' | 'remote' }`

#### Constants
- `ULTRAPLAN_TELEPORT_SENTINEL`: `'__ULTRAPLAN_TELEPORT_LOCAL__'` — marker in browser PlanModal feedback when user clicks "teleport back to terminal"

#### Classes
- `ExitPlanModeScanner`: Pure stateful classifier for the CCR event stream. Ingests `SDKMessage[]` batches and returns the current ExitPlanMode verdict. No I/O, no timers — feed it synthetic or recorded events for unit tests.
  - `ingest(newEvents)`: Process new events and return ScanResult
  - `hasPendingPlan`: True when an ExitPlanMode tool_use exists with no tool_result yet
  - `rejectCount`: Number of unique rejected plan IDs

#### Functions
- `pollForApprovedExitPlanMode(sessionId, timeoutMs, onPhaseChange?, shouldStop?)`: Poll a remote session for an approved ExitPlanMode result. Returns the approved plan text and execution target. Throws `UltraplanPollError` on failure.
- `extractTeleportPlan(content)`: Extract plan text after the ULTRAPLAN_TELEPORT_SENTINEL marker. Returns null when sentinel is absent.
- `extractApprovedPlan(content)`: Extract plan text from "## Approved Plan:" or "## Approved Plan (edited by user):" markers.

#### Polling Behavior
- Polls every 3 seconds with up to 5 consecutive failure tolerance for transient network errors
- Phase transitions: `running` → `needs_input` (idle with no pending plan) → `plan_ready` (ExitPlanMode emitted, awaiting result)
- Handles both "approved" (execute remotely) and "teleport" (execute locally, archive remote) outcomes
- Normal rejections are tracked and skipped so the user can iterate in the browser
- CCR briefly flips to 'idle' between tool turns — only trusted when no new events arrived

### Keyword Detection (`keyword.ts`)

#### Types
- `TriggerPosition`: `{ word, start, end }` — position of a keyword trigger in text

#### Functions
- `findUltraplanTriggerPositions(text)`: Find "ultraplan" keyword positions that are actual launch directives
- `findUltrareviewTriggerPositions(text)`: Find "ultrareview" keyword positions
- `hasUltraplanKeyword(text)`: Check if text contains a triggerable "ultraplan"
- `hasUltrareviewKeyword(text)`: Check if text contains a triggerable "ultrareview"
- `replaceUltraplanKeyword(text)`: Replace first triggerable "ultraplan" with "plan" (preserves casing of suffix)

#### Filtering Rules (what does NOT trigger)
- Inside paired delimiters: backticks, double quotes, angle brackets, curly braces, square brackets, parentheses
- Single quotes are delimiters only when not an apostrophe (opening quote preceded by non-word char)
- Path/identifier context: preceded or followed by `/`, `\`, or `-`, or followed by `.` + word char (file extension)
- Followed by `?`: a question about the feature shouldn't invoke it
- Slash command input: text starting with `/` is a slash command invocation

## Design Notes

- The `ExitPlanModeScanner` is a pure classifier — no I/O, no timers — making it trivially testable with synthetic or recorded events
- Precedence in scanning: approved > terminated > rejected > pending > unchanged. A batch may contain both an approved tool_result AND a subsequent `{type:'result'}` (user approved, then remote crashed) — the approved plan is real and shouldn't be dropped
- The teleport sentinel allows users to approve a plan but request local execution instead of remote — the plan text is embedded in the rejection feedback
- Keyword detection mirrors `findThinkingTriggerPositions` (thinking.ts) so PromptInput treats both trigger types uniformly
- `replaceUltraplanKeyword` preserves the user's casing of the "plan" suffix — "Ultraplan" → "Plan", "ULTRAPLAN" → "PLAN"
