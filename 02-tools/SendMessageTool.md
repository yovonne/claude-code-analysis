# SendMessageTool

## Purpose

SendMessageTool is the inter-agent communication system for Claude Code's swarm protocol. It enables agents to send messages to teammates, broadcast to entire teams, handle shutdown requests/responses, and manage plan approval workflows. It supports both in-team messaging and cross-session communication via UDS sockets and Remote Control bridges.

## Location

- `restored-src/src/tools/SendMessageTool/SendMessageTool.ts` — Main tool definition (918 lines)
- `restored-src/src/tools/SendMessageTool/prompt.ts` — Tool prompt and description (50 lines)
- `restored-src/src/tools/SendMessageTool/constants.ts` — Tool name constant
- `restored-src/src/tools/SendMessageTool/UI.tsx` — UI rendering for tool use/result messages

## Key Exports

| Export | Description |
|--------|-------------|
| `SendMessageTool` | The complete tool definition built via `buildTool()` |
| `Input` | Zod-inferred input type: `{ to, summary?, message }` |
| `SendMessageToolOutput` | Union type: `MessageOutput | BroadcastOutput | RequestOutput | ResponseOutput` |
| `MessageRouting` | Routing metadata: `{ sender, senderColor?, target, targetColor?, summary?, content? }` |

## Input/Output Schemas

### Input Schema

```typescript
{
  to: string,              // Recipient: teammate name, "*" for broadcast, "uds:<path>", or "bridge:<session-id>"
  summary?: string,        // 5-10 word UI preview (required when message is a string)
  message: string | {      // Plain text or structured message
    type: 'shutdown_request', reason?: string
  } | {
    type: 'shutdown_response', request_id: string, approve: boolean, reason?: string
  } | {
    type: 'plan_approval_response', request_id: string, approve: boolean, feedback?: string
  }
}
```

### Output Types

**MessageOutput** (direct message):
```typescript
{ success: boolean, message: string, routing?: MessageRouting }
```

**BroadcastOutput** (broadcast to all):
```typescript
{ success: boolean, message: string, recipients: string[], routing?: MessageRouting }
```

**RequestOutput** (shutdown request):
```typescript
{ success: boolean, message: string, request_id: string, target: string }
```

**ResponseOutput** (shutdown/plan response):
```typescript
{ success: boolean, message: string, request_id?: string }
```

## Message Types

### Plain Text Messages

```json
{"to": "researcher", "summary": "assign task 1", "message": "start on task #1"}
```

- Sent to a specific teammate by name
- Requires a `summary` for UI preview
- Delivered to recipient's mailbox automatically

### Broadcast Messages

```json
{"to": "*", "summary": "team update", "message": "everyone check your tasks"}
```

- Sent to all teammates except the sender
- Expensive (linear in team size) — use only when everyone genuinely needs it
- Cannot send structured messages via broadcast

### Shutdown Request/Response

```json
// Request (team lead to teammate)
{"to": "researcher", "message": {"type": "shutdown_request", "reason": "work complete"}}

// Response (teammate to team lead)
{"to": "team-lead", "message": {"type": "shutdown_response", "request_id": "...", "approve": true}}
```

- Team lead sends shutdown_request to gracefully terminate a teammate
- Teammate responds with shutdown_response (approve or reject)
- Approving shutdown terminates the teammate's process
- Rejecting requires a reason

### Plan Approval Response

```json
{"to": "researcher", "message": {"type": "plan_approval_response", "request_id": "...", "approve": false, "feedback": "add error handling"}}
```

- Only team lead can approve/reject plans
- Approval inherits the leader's permission mode
- Rejection includes feedback for revision

## Message Routing

### In-Team Routing

Messages within a team are delivered via mailbox:

1. Sender calls SendMessageTool
2. Message written to recipient's mailbox (`writeToMailbox`)
3. Recipient receives message automatically (no polling needed)
4. Messages queued if recipient is mid-turn, delivered when turn ends

### Cross-Session Routing (UDS_INBOX feature)

| Target Scheme | Description |
|---------------|-------------|
| `"uds:/path/to.sock"` | Local Claude session's socket (same machine) |
| `"bridge:session_..."` | Remote Control peer session (cross-machine) |

Cross-session messages:
- Arrive wrapped as `<cross-session-message from="...">`
- To reply, copy the `from` attribute as your `to`
- Only plain text allowed (no structured messages)
- Require explicit user consent (safety check, not bypassable)

### In-Process Subagent Routing

When sending to a registered in-process subagent:
1. Look up agent by name in `agentNameRegistry`
2. If task is running: queue message for next tool round
3. If task is stopped: auto-resume agent with the message as prompt
4. If no active task: try resume from disk transcript

## Permission Checks

Bridge (cross-machine) messages require explicit user consent:

```typescript
{
  behavior: 'ask',
  message: "Send a message to Remote Control session ...?",
  decisionReason: { type: 'safetyCheck', reason: 'Cross-machine bridge message requires explicit user consent' }
}
```

This is a safety check, not a mode check — it cannot be bypassed by auto-mode or bypassPermissions.

## Validation Rules

| Rule | Error |
|------|-------|
| Empty `to` field | "to must not be empty" |
| Empty bridge/uds target | "address target must not be empty" |
| `to` contains `@` | "to must be a bare teammate name" |
| Structured message to bridge | "structured messages cannot be sent cross-session" |
| Bridge not connected | "Remote Control is not connected" |
| String message without summary | "summary is required when message is a string" |
| Broadcast with structured message | "structured messages cannot be broadcast" |
| shutdown_response to non-lead | "shutdown_response must be sent to team-lead" |
| Reject shutdown without reason | "reason is required when rejecting a shutdown request" |

## Shutdown Flow

```
Team Lead                          Teammate
    |                                 |
    |-- shutdown_request ------------>|
    |                                 |
    |                                 | (decides to approve/reject)
    |                                 |
    |<-- shutdown_response (approve) -|
    |                                 |
    | (terminates teammate)           | (exits)
    |                                 |
```

If the teammate approves:
- In-process: abort controller is signaled
- External: gracefulShutdown() is called immediately

## Plan Approval Flow

```
Teammate                           Team Lead
    |                                 |
    |-- plan_approval_request ------->|
    |                                 |
    |                                 | (reviews plan)
    |                                 |
    |<-- plan_approval_response ------|
    |   (approve or reject + feedback)|
    |                                 |
```

Only the team lead can approve or reject plans. The approval response includes the permission mode the teammate should inherit.

## Dependencies

| Module | Purpose |
|--------|---------|
| `utils/teammateMailbox.ts` | Mailbox write operations |
| `utils/teammate.ts` | Agent identity helpers |
| `utils/swarm/teamHelpers.ts` | Team file reading |
| `utils/swarm/constants.ts` | TEAM_LEAD_NAME constant |
| `utils/agentId.ts` | Request ID generation |
| `utils/permissions/` | Permission checking |
| `utils/peerAddress.ts` | Address parsing (uds/bridge schemes) |
| `utils/gracefulShutdown.ts` | Process shutdown |
| `bridge/peerSessions.js` | Cross-session messaging |
| `utils/udsClient.js` | UDS socket communication |
| `tasks/LocalAgentTask/` | In-process agent task management |
| `AgentTool/resumeAgent.js` | Agent resume functionality |
| `services/analytics/` | Event logging |
