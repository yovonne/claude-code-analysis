# KAIROS & Ant-Only Feature-Gated Tools

## Purpose

This document covers seven tools that are conditionally bundled into Claude Code based on feature flags or environment variables. These tools are **not** available in the public/external build — they are either gated behind `KAIROS` (assistant mode), `PROACTIVE` (proactive mode), `ANT`-only builds, or experimental flags. Understanding these tools reveals the full capability surface of the internal Anthropic build.

## Location

All tool registrations live in `restored-src/src/tools.ts`. Individual tool source files are in `restored-src/src/tools/<ToolName>/`.

---

## Feature Flag Summary

| Tool | Gate | Gate Type | Build Impact |
|------|------|-----------|--------------|
| **SleepTool** | `feature('PROACTIVE') \|\| feature('KAIROS')` | Bun feature flag | DCE in external builds |
| **MonitorTool** | `feature('MONITOR_TOOL')` | Bun feature flag | DCE in external builds |
| **SendUserFileTool** | `feature('KAIROS')` | Bun feature flag | DCE in external builds |
| **PushNotificationTool** | `feature('KAIROS') \|\| feature('KAIROS_PUSH_NOTIFICATION')` | Bun feature flag | DCE in external builds |
| **SubscribePRTool** | `feature('KAIROS_GITHUB_WEBHOOKS')` | Bun feature flag | DCE in external builds |
| **SuggestBackgroundPRTool** | `process.env.USER_TYPE === 'ant'` | Runtime env (build-time constant) | DCE in external builds |
| **VerifyPlanExecutionTool** | `process.env.CLAUDE_CODE_VERIFY_PLAN === 'true'` | Runtime env (build-time constant) | DCE in external builds |

### Gate Mechanisms

**Bun feature flags** (`feature('...')` from `bun:bundle`):
- Evaluated at bundle time. When `false`, the entire `require()` call and dependent code is eliminated by Bun's dead code elimination (DCE). External builds ship with these flags off, so the tool code has **zero footprint**.

**Environment variable gates** (`process.env.USER_TYPE === 'ant'`, `process.env.CLAUDE_CODE_VERIFY_PLAN === 'true'`):
- Also build-time constants in the external build pipeline. `USER_TYPE` is set to `'ant'` only for internal builds; external builds set it to `'external'`. Bun DCE eliminates the branches.

### Why These Tools Are Gated

1. **Internal-only features**: Tools like `SuggestBackgroundPRTool` and `VerifyPlanExecutionTool` support Anthropic's internal development workflows and are not relevant to external users.

2. **Assistant/Proactive mode**: `SleepTool`, `SendUserFileTool`, and `PushNotificationTool` are core to the KAIROS assistant mode — an always-on agent that works autonomously. External builds don't have this mode.

3. **Experimental/preview features**: `MONITOR_TOOL`, `KAIROS_GITHUB_WEBHOOKS`, and `KAIROS_PUSH_NOTIFICATION` are feature-flagged for gradual rollout and testing.

4. **Bundle size**: Each tool carries its own source, UI components, prompts, and dependencies. DCE ensures external builds don't pay for unused code.

---

## Tool Details

### 1. SleepTool

**Tool Name:** `Sleep`

**Feature Gate:** `feature('PROACTIVE') || feature('KAIROS')`

**Location:** `restored-src/src/tools/SleepTool/prompt.ts`

**Purpose:** Allows the model to wait for a specified duration without holding a shell process. Essential for proactive/assistant mode where the agent periodically checks for work via `<tick>` prompts.

**Key Exports:**
- `SLEEP_TOOL_NAME`: `'Sleep'`
- `DESCRIPTION`: `'Wait for a specified duration'`
- `SLEEP_TOOL_PROMPT`: Instructions for the LLM

**Implementation:**
Only `prompt.ts` exists in the restored source — the main tool definition (`SleepTool.ts`) is missing. The prompt instructs the model to:
- Use Sleep when told to rest, when idle, or when waiting for something
- Respond to `<tick>` periodic check-ins by looking for useful work
- Call Sleep concurrently with other tools (non-blocking)
- Prefer Sleep over `Bash(sleep ...)` to avoid holding shell processes
- Balance wake-up API calls against the 5-minute prompt cache expiration

**Integration Points:**
- `query.ts:1566` — Detects Sleep tool usage in response blocks
- `REPL.tsx:1655` — Tracks when only Sleep is active (enables user input during sleep)
- `main.tsx:1864` — Proactive mode activated before `getTools()` so `SleepTool.isEnabled()` passes
- `handlePromptSubmit.ts:320` — User input with `interruptBehavior: 'cancel'` wakes a sleeping tool
- `constants/prompts.ts:872-886` — System prompt mandates Sleep when idle (never narrate waiting)
- `classifierDecision.ts:87` — Listed as a safe YOLO-allowlisted tool
- `mcp/channelNotification.ts:9` — Sleep polls `hasCommandsInQueue()` and wakes within 1s when MCP notifications arrive

**Why Gated:** Sleep is the heartbeat mechanism for proactive/assistant mode. Without proactive mode, there are no `<tick>` prompts and no autonomous agent loop — Sleep would be meaningless.

**Differs From Standard Tools:** Unlike `Bash(sleep N)`, Sleep doesn't consume a shell process, is concurrency-safe, and integrates with the tick-based proactive scheduling system. Each wake-up triggers an API call but benefits from prompt cache reuse.

---

### 2. MonitorTool

**Tool Name:** `Monitor` (inferred from module naming)

**Feature Gate:** `feature('MONITOR_TOOL')`

**Location:** `restored-src/src/tools/MonitorTool/MonitorTool.js` (not in restored source)

**Purpose:** Enables monitoring of long-running background commands (Bash/PowerShell) as dedicated monitor tasks. When `MONITOR_TOOL` is enabled, background processes spawn as `kind: 'monitor'` tasks with `'next'` priority instead of `'later'`.

**Integration Points:**
- `tools.ts:39-40` — Conditionally loaded via `require()`
- `BashTool.tsx:525` — When `MONITOR_TOOL` is on and `run_in_background` is not set, Bash commands may become monitor tasks
- `PowerShellTool.tsx:361` — Same behavior for PowerShell
- `BashTool/prompt.ts:312-320` — Prompt includes monitor-specific instructions when flag is on
- `AgentTool/runAgent.ts:849` — Agent execution checks for monitor tool availability
- `LocalShellTask.tsx:129` — `kind === 'monitor'` creates a special monitor task type
- `LocalShellTask.tsx:169` — Monitor tasks get `'next'` priority vs `'later'` for regular background tasks
- `tasks.ts:12` — `MonitorMcpTask` — a dedicated task type for MCP monitoring
- `BackgroundTasksDialog.tsx:117-119` — UI components for monitor task display
- `PermissionRequest.tsx:40-41,73` — Special permission request UI for monitor tool
- `query.ts:1551` — Query engine drains monitor tasks without Sleep when flag is on

**Why Gated:** Monitor mode transforms how background tasks are scheduled and displayed. It's an experimental enhancement to the background task system that may change the user experience significantly.

**Differs From Standard Tools:** Standard background tasks run with `'later'` priority and are managed through the generic task system. Monitor tasks get `'next'` priority, dedicated UI components (`MonitorMcpDetailDialog`), and special permission request handling. The query engine can drain monitor tasks without requiring Sleep cycles.

---

### 3. SendUserFileTool

**Tool Name:** Exported as `SEND_USER_FILE_TOOL_NAME` (value from prompt.ts)

**Feature Gate:** `feature('KAIROS')`

**Location:** `restored-src/src/tools/SendUserFileTool/SendUserFileTool.js` (not in restored source)

**Purpose:** Sends files from the agent to the user in KAIROS assistant mode. Complements BriefTool (`SendUserMessage`) by providing a dedicated channel for file delivery in the assistant workflow.

**Integration Points:**
- `tools.ts:42-43` — Conditionally loaded when `KAIROS` is enabled
- `conversationRecovery.ts:67-70` — Tool name loaded for session recovery; used to filter tools during conversation replay (`line 367`)
- `ToolSearchTool/prompt.ts:15-18,100-101` — Tool name included in tool search results when KAIROS is on
- `Messages.tsx:82,509` — Combined with BriefTool names for message rendering: `[BRIEF_TOOL_NAME, SEND_USER_FILE_TOOL_NAME]`

**Why Gated:** This tool is specific to the KAIROS assistant mode's file delivery mechanism. External builds don't have assistant mode, so the tool would be non-functional.

**Differs From Standard Tools:** Unlike BriefTool which sends text messages (with optional attachments), SendUserFileTool appears to be a dedicated file-delivery channel. It's treated as a "brief tool" variant in the Messages component, suggesting it renders similarly but with file-focused semantics.

---

### 4. PushNotificationTool

**Tool Name:** `PushNotification` (inferred from module naming)

**Feature Gate:** `feature('KAIROS') || feature('KAIROS_PUSH_NOTIFICATION')`

**Location:** `restored-src/src/tools/PushNotificationTool/PushNotificationTool.js` (not in restored source)

**Purpose:** Sends push notifications to the user when the assistant mode is not actively visible. Enables the assistant to alert users about important events even when they're not looking at the terminal.

**Integration Points:**
- `tools.ts:45-48` — Conditionally loaded when either KAIROS or KAIROS_PUSH_NOTIFICATION is enabled
- `ConfigTool/supportedSettings.ts:164` — Configuration settings include push notification options when the feature is enabled
- `Settings/Config.tsx:658,672` — UI label changes from "Notifications" to "Local notifications" when KAIROS flags are on, with additional config sections

**Why Gated:** Push notifications require platform-specific integration (OS notification APIs, mobile push services) and are only meaningful when the assistant runs autonomously. The `KAIROS_PUSH_NOTIFICATION` sub-flag allows shipping push notifications independently of full assistant mode.

**Differs From Standard Tools:** This is the only tool that interacts with OS-level notification systems rather than the terminal UI. It bridges the gap between the always-on assistant and the user's attention, enabling asynchronous communication.

---

### 5. SubscribePRTool

**Tool Name:** `SubscribePR` (inferred from module naming)

**Feature Gate:** `feature('KAIROS_GITHUB_WEBHOOKS')`

**Location:** `restored-src/src/tools/SubscribePRTool/SubscribePRTool.js` (not in restored source)

**Purpose:** Allows the assistant to subscribe to GitHub Pull Request webhooks, enabling real-time notifications when PRs are updated, commented on, or merged. Part of the KAIROS GitHub integration surface.

**Integration Points:**
- `tools.ts:50-51` — Conditionally loaded when `KAIROS_GITHUB_WEBHOOKS` is enabled

**Why Gated:** GitHub webhook subscriptions require:
- A publicly reachable endpoint for webhook delivery
- GitHub app/bot credentials
- Persistent assistant process to receive and process webhooks

These are infrastructure requirements that only make sense in the internal Anthropic deployment where the assistant runs as a persistent service.

**Differs From Standard Tools:** Unlike standard tools that are model-initiated (the LLM decides to call them), SubscribePRTool sets up an inbound event stream. It's a subscription tool — the assistant registers interest and then receives async notifications, which may trigger further tool calls via the tick system.

---

### 6. SuggestBackgroundPRTool

**Tool Name:** `SuggestBackgroundPR` (inferred from module naming)

**Feature Gate:** `process.env.USER_TYPE === 'ant'`

**Location:** `restored-src/src/tools/SuggestBackgroundPRTool/SuggestBackgroundPRTool.js` (not in restored source)

**Purpose:** Suggests or creates background Pull Requests as part of Anthropic's internal development workflow. Enables the assistant to propose code changes via PRs rather than direct file modifications.

**Integration Points:**
- `tools.ts:20-23` — Conditionally loaded when `USER_TYPE === 'ant'`
- `tools.ts:216` — Added to the tool pool when available

**Why Gated:** This tool is specific to Anthropic's internal GitHub workflow. It requires:
- Internal GitHub organization access
- Custom PR creation/management infrastructure
- Anthropic-specific code review processes

External users don't have access to this infrastructure, so the tool would be non-functional.

**Differs From Standard Tools:** Standard tools modify files directly in the working directory. SuggestBackgroundPRTool operates at the GitHub API level, creating PRs as a side effect. It's a higher-level workflow tool that abstracts over the file-edit → commit → push → PR-create pipeline.

---

### 7. VerifyPlanExecutionTool

**Tool Name:** `VerifyPlanExecution` (inferred; constant: `VERIFY_PLAN_EXECUTION_TOOL_NAME`)

**Feature Gate:** `process.env.CLAUDE_CODE_VERIFY_PLAN === 'true'`

**Location:** `restored-src/src/tools/VerifyPlanExecutionTool/VerifyPlanExecutionTool.js` (not in restored source)

**Purpose:** Verifies that the model's plan execution matches the intended plan. Acts as a guardrail during complex multi-step operations, ensuring the agent stays on track and doesn't deviate from the approved plan.

**Key Exports (from constants.js):**
- `VERIFY_PLAN_EXECUTION_TOOL_NAME` — Tool name string

**Integration Points:**
- `tools.ts:91-94` — Conditionally loaded when `CLAUDE_CODE_VERIFY_PLAN === 'true'`
- `classifierDecision.ts:37-41,91` — Tool name loaded for permission classification; added to safe YOLO allowlist
- `hooks/execAgentHook.ts:47` — Comment references: "programmatic construction, since refactored into VerifyPlanExecutionTool"
- `schemas/hooks.ts:136` — Schema comment: "has since been refactored into VerifyPlanExecutionTool"
- `attachments.ts:291,3900,3922` — `VERIFY_PLAN_REMINDER_CONFIG` with `TURNS_BETWEEN_REMINDERS`; used to periodically remind the model about plan verification
- `messages.ts:4241-4244` — Dead code elimination comment for external builds
- `ExitPlanModePermissionRequest.tsx:366` — DCE comment for external builds

**Why Gated:** Plan verification is an experimental safety feature that adds overhead to the agent loop. It's gated behind an env var so it can be:
- Enabled for testing/validation of the concept
- Disabled in production to reduce API costs and latency
- Toggled per-session for debugging agent behavior

**Differs From Standard Tools:** This is a meta-tool — it doesn't perform work on the codebase but instead validates the agent's own behavior against a plan. It's part of the safety/guardrail infrastructure rather than the capability surface. The `VERIFY_PLAN_REMINDER_CONFIG` suggests it periodically prompts the model to self-verify at configured turn intervals.

---

## How These Tools Differ From Standard Tools

| Aspect | Standard Tools | KAIROS/Ant-Only Tools |
|--------|---------------|----------------------|
| **Availability** | Always bundled | Conditionally bundled via DCE |
| **Build footprint** | Present in all builds | Zero footprint when gated off |
| **Target user** | External + internal | Internal Anthropic only |
| **Mode dependency** | Work in any mode | Often require proactive/assistant mode |
| **Infrastructure** | Self-contained | May require external services (webhooks, push APIs, GitHub org) |
| **Permission model** | Standard permission flow | Some have special permission UI (MonitorTool) |
| **Execution model** | Model-initiated | Some set up async event streams (SubscribePRTool) |

## Registration Pattern

All seven tools follow the same conditional registration pattern in `tools.ts`:

```typescript
// Build-time gate with lazy require
const ToolName = feature('FLAG') || process.env.VAR === 'value'
  ? require('./tools/ToolName/ToolName.js').ToolName
  : null

// Conditional inclusion in tool pool
...(ToolName ? [ToolName] : [])
```

This pattern ensures:
1. **Zero runtime cost** when disabled — `null` is a no-op in the spread
2. **Zero bundle cost** when disabled — Bun DCE eliminates the entire `require()` branch
3. **Lazy loading** — tool code is only loaded when the tool is actually available
4. **Type safety** — TypeScript infers `ToolName | null` from the ternary

## Related Modules

- [BriefTool](./BriefTool.md) — Primary user communication tool, also KAIROS-gated
- [AgentTool](./AgentTool.md) — Sub-agent execution, integrates with proactive mode
- [ScheduleCronTool](../01-core-modules/tool-system.md) — Cron scheduling, gated by `AGENT_TRIGGERS`
- [tool-system](../01-core-modules/tool-system.md) — Core tool abstraction and registry
