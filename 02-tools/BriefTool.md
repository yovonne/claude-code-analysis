# BriefTool

## Purpose

The BriefTool (user-facing name: `SendUserMessage`) is Claude's primary output channel for sending messages to the user. It is the mechanism through which Claude communicates answers, status updates, checkpoints, and proactive notifications. Text outside this tool is visible in the detail view but most users won't open it — the answer lives here. The tool supports markdown-formatted messages, file attachments (images, screenshots, diffs, logs), and distinguishes between normal replies and proactive notifications for downstream routing.

## Location

- `restored-src/src/tools/BriefTool/BriefTool.ts`
- `restored-src/src/tools/BriefTool/prompt.ts`
- `restored-src/src/tools/BriefTool/attachments.ts`
- `restored-src/src/tools/BriefTool/upload.ts`
- `restored-src/src/tools/BriefTool/UI.tsx`

## Key Exports

### Functions

- `BriefTool`: The main tool definition built via `buildTool()` — handles message delivery, attachment resolution, and status routing
- `isBriefEntitled()`: Entitlement check — whether the user is allowed to use Brief (combines feature flags, GrowthBook gate, and env bypass)
- `isBriefEnabled()`: Unified activation gate — whether Brief is actually active in the current session (requires entitlement + opt-in)
- `validateAttachmentPaths()`: Validates that attachment file paths exist, are regular files, and are accessible
- `resolveAttachments()`: Resolves attachment paths to metadata (size, isImage) and optionally uploads to cloud for web viewers
- `uploadBriefAttachment()`: Uploads a single attachment to the REPL bridge's private API for web preview
- `renderToolUseMessage()`, `renderToolResultMessage()`: UI rendering functions for terminal display

### Types

- `Output`: `{ message: string, attachments?: ResolvedAttachment[], sentAt?: string }`
- `ResolvedAttachment`: `{ path: string, size: number, isImage: boolean, file_uuid?: string }`
- `BriefUploadContext`: `{ replBridgeEnabled: boolean, signal?: AbortSignal }`

### Constants

- `BRIEF_TOOL_NAME`: `'SendUserMessage'`
- `LEGACY_BRIEF_TOOL_NAME`: `'Brief'` — legacy wire name for backward compatibility
- `DESCRIPTION`: `'Send a message to the user'`
- `BRIEF_TOOL_PROMPT`: Instructions for the LLM on how to use the tool
- `BRIEF_PROACTIVE_SECTION`: System prompt section about talking to the user
- `KAIROS_BRIEF_REFRESH_MS`: `300,000` (5 minutes) — GrowthBook feature refresh interval

## Dependencies

### Internal Dependencies

- `prompt.ts` — Tool name, description, prompt, and proactive section text
- `attachments.ts` — Attachment validation, resolution, and upload orchestration
- `upload.ts` — File upload to REPL bridge private API (axios-based multipart upload)
- `UI.tsx` — React components for rendering messages and attachment lists in terminal
- `bridge/bridgeConfig.js` — OAuth token and base URL for bridge uploads
- `imageResizer.js` — Image resizing and downsampling for pasted images
- `imageStore.js` — Image caching and storage for pasted content
- `formatBriefTimestamp.js` — Timestamp formatting for brief-only view
- `Markdown.js` — Markdown rendering for message content

### External Dependencies

- `zod/v4` — Schema validation with strict object
- `react` — UI component rendering with React Compiler
- `bun:bundle` — Feature flag gating (`KAIROS`, `KAIROS_BRIEF`, `BRIDGE_MODE`)
- `axios` — HTTP client for bridge uploads
- `crypto` — UUID generation for multipart form boundaries
- `fs/promises` — File stat and read operations

## Implementation Details

### Input Schema

```typescript
z.strictObject({
  message: z.string()
    .describe('The message for the user. Supports markdown formatting.'),
  attachments: z.array(z.string())
    .optional()
    .describe('Optional file paths (absolute or relative to cwd) to attach.'),
  status: z.enum(['normal', 'proactive'])
    .describe("Use 'proactive' when surfacing something the user hasn't asked for."),
})
```

| Field | Type | Description |
|-------|------|-------------|
| `message` | `string` | Message text (markdown supported) |
| `attachments` | `string[]` (optional) | File paths for images, screenshots, diffs, logs |
| `status` | `'normal' \| 'proactive'` | Routing intent — reply vs. unsolicited notification |

### Output Schema

```typescript
z.object({
  message: z.string().describe('The message'),
  attachments: z.array(z.object({
    path: z.string(),
    size: z.number(),
    isImage: z.boolean(),
    file_uuid: z.string().optional(),
  })).optional().describe('Resolved attachment metadata'),
  sentAt: z.string().optional()
    .describe('ISO timestamp at tool execution time'),
})
```

**Note:** `attachments` and `sentAt` must remain optional — resumed sessions replay pre-attachment/pre-sentAt outputs verbatim, and required fields would crash the UI renderer on resume.

### Feature Gating

BriefTool has a two-tier gating system:

#### Tier 1: Entitlement (`isBriefEntitled()`)

Determines whether the user is **allowed** to use Brief:

```
feature('KAIROS') || feature('KAIROS_BRIEF')
    ? getKairosActive() || isEnvTruthy(CLAUDE_CODE_BRIEF) || getFeatureValue('tengu_kairos_brief')
    : false
```

| Condition | Effect |
|-----------|--------|
| `KAIROS` or `KAIROS_BRIEF` feature off | Always false (dead-code eliminated in external builds) |
| `getKairosActive()` | Assistant mode is active |
| `CLAUDE_CODE_BRIEF` env var | Force-grants entitlement for dev/testing |
| `tengu_kairos_brief` GrowthBook flag | Remote feature flag (refreshed every 5 min) |

#### Tier 2: Activation (`isBriefEnabled()`)

Determines whether Brief is **actually active** in the current session:

```
(feature('KAIROS') || feature('KAIROS_BRIEF'))
    ? (getKairosActive() || getUserMsgOptIn()) && isBriefEntitled()
    : false
```

Activation requires explicit opt-in via one of:
- `--brief` CLI flag
- `defaultView: 'chat'` in settings
- `/brief` slash command
- `/config` defaultView picker
- `SendUserMessage` in `--tools` / SDK `tools` option
- `CLAUDE_CODE_BRIEF` env var (dev/testing bypass)

Assistant mode (`kairosActive`) bypasses the opt-in requirement since its system prompt hard-codes "you MUST use SendUserMessage".

**Kill-switch behavior:** The GrowthBook gate is re-checked on every call. Flipping `tengu_kairos_brief` off mid-session disables the tool on the next 5-minute refresh even for opted-in sessions.

### Tool Properties

| Property | Value | Rationale |
|----------|-------|-----------|
| `isReadOnly()` | `true` | Does not modify the filesystem |
| `isConcurrencySafe()` | `true` | Safe to call concurrently |
| `userFacingName()` | `''` (empty) | Drops the tool chrome from UI — message appears as plain text |
| `shouldDefer` | Not set | Does not defer execution |
| `maxResultSizeChars` | `100,000` | Maximum result size |

### The `call()` Method

```typescript
async call({ message, attachments, status }, context) {
  const sentAt = new Date().toISOString()
  logEvent('tengu_brief_send', {
    proactive: status === 'proactive',
    attachment_count: attachments?.length ?? 0,
  })
  if (!attachments || attachments.length === 0) {
    return { data: { message, sentAt } }
  }
  const appState = context.getAppState()
  const resolved = await resolveAttachments(attachments, {
    replBridgeEnabled: appState.replBridgeEnabled,
    signal: context.abortController.signal,
  })
  return { data: { message, attachments: resolved, sentAt } }
}
```

**Flow:**
1. Capture `sentAt` timestamp
2. Log analytics event with proactive flag and attachment count
3. If no attachments, return message + timestamp immediately
4. If attachments exist, resolve them (stat files, optionally upload)
5. Return message + resolved attachments + timestamp

### Attachment System

#### Validation (`validateAttachmentPaths()`)

Validates each attachment path:
1. Expands relative paths against cwd
2. Stats the file to verify it exists and is a regular file
3. Handles errors:
   - `ENOENT`: File does not exist (includes cwd in error message)
   - `EACCES`/`EPERM`: Permission denied
   - Other errors: Re-thrown

#### Resolution (`resolveAttachments()`)

Two-phase resolution:
1. **Stat phase** (serial, local, fast): Stats each file to get size and detect image type via extension
2. **Upload phase** (parallel, network, slow): Uploads to bridge if enabled

```typescript
// Phase 1: Stat all files
const stated: ResolvedAttachment[] = []
for (const rawPath of rawPaths) {
  const fullPath = expandPath(rawPath)
  const stats = await stat(fullPath)
  stated.push({
    path: fullPath,
    size: stats.size,
    isImage: IMAGE_EXTENSION_REGEX.test(fullPath),
  })
}

// Phase 2: Upload if bridge is enabled
if (feature('BRIDGE_MODE')) {
  const shouldUpload = uploadCtx.replBridgeEnabled || isEnvTruthy(process.env.CLAUDE_CODE_BRIEF_UPLOAD)
  const { uploadBriefAttachment } = await import('./upload.js')
  const uuids = await Promise.all(stated.map(a => uploadBriefAttachment(a.path, a.size, { ... })))
  return stated.map((a, i) => uuids[i] === undefined ? a : { ...a, file_uuid: uuids[i] })
}
return stated
```

The upload module is dynamically imported inside the `BRIDGE_MODE` feature guard so that `upload.ts` (with its axios, crypto, zod, auth utils, MIME map dependencies) is fully tree-shaken from non-bridge builds.

#### Upload (`uploadBriefAttachment()`)

Uploads a single file to the REPL bridge's private API:

**Endpoint:** `POST {baseUrl}/api/oauth/file_upload`

**Process:**
1. Check `BRIDGE_MODE` feature flag
2. Verify bridge is enabled
3. Check file size against `MAX_UPLOAD_BYTES` (30 MB)
4. Get OAuth access token
5. Read file content
6. Build multipart form with correct MIME type
7. POST to upload endpoint with Bearer auth
8. Parse response for `file_uuid`

**MIME type detection:**
```typescript
const MIME_BY_EXT: Record<string, string> = {
  '.png': 'image/png',
  '.jpg': 'image/jpeg',
  '.jpeg': 'image/jpeg',
  '.gif': 'image/gif',
  '.webp': 'image/webp',
}
// Default: 'application/octet-stream'
```

**Graceful degradation:** Every failure returns `undefined` rather than throwing. The attachment still carries `{path, size, isImage}` so local renderers work unaffected.

**Best-effort failures:**
- No OAuth token → skip
- File too large → skip (logs size and limit)
- Read error → skip
- Network error → skip
- Non-201 response → skip
- Invalid response shape → skip

### Result Formatting

`mapToolResultToToolResultBlockParam()` formats the result for the LLM:

```
Message delivered to user. (N attachment(s) included)
```

The attachment count suffix is only added when attachments are present.

## UI Rendering

### renderToolUseMessage

Returns an empty string — the tool has no visible "use" chrome because `userFacingName()` is empty.

### renderToolResultMessage

Renders the delivered message in three different view modes:

#### Transcript Mode (Ctrl+O)

Model text is NOT filtered — keeps the black circle marker so `SendUserMessage` is visually distinct from surrounding text blocks:

```
⏺ <markdown content>
    [attachment list]
```

#### Brief-Only Mode (Chat View)

"Claude" label with 2-column indent, matching the "You" label applied to user input:

```
Claude [timestamp]
  <markdown content>
  [attachment list]
```

#### Default View

No gutter mark — reads as plain text. The `userFacingName() === ''` causes `AssistantToolUseMessage` to render null (no tool chrome), and `dropTextInBriefTurns` hides redundant assistant text that would otherwise precede this content:

```
<markdown content>
[attachment list]
```

### AttachmentList Component

Renders a list of attached files:

```
· [image] /path/to/screenshot.png (1.2 MB)
· [file]  /path/to/log.txt (45 KB)
```

Each attachment shows:
- Type indicator: `[image]` or `[file]`
- Display path (relative when possible)
- Formatted file size

## Communication Guidelines (from BRIEF_PROACTIVE_SECTION)

The system prompt section `BRIEF_PROACTIVE_SECTION` provides guidelines for how Claude should communicate:

1. **Every reply goes through BriefTool** — even "hi" and "thanks"
2. **Ack before working** — "On it — checking the test output" prevents the user from staring at a spinner
3. **Pattern for longer work**: ack → work → result
4. **Checkpoints** — send updates when something useful happened (a decision, a surprise, a phase boundary)
5. **Keep messages tight** — the decision, the file:line, the PR number
6. **Second person always** — "your config", never third person
7. **No filler** — skip "running tests..." — a checkpoint earns its place by carrying information

### Status Field Routing

| Status | When to Use |
|--------|-------------|
| `normal` | Replying to what the user just asked |
| `proactive` | Surfacing something unsolicited — task completion while away, a blocker, an unsolicited status update |

Downstream routing uses this field to determine how to present the message to the user.

## Configuration

### Environment Variables

| Variable | Purpose |
|----------|---------|
| `CLAUDE_CODE_BRIEF` | Force-grants Brief entitlement (dev/testing bypass) |
| `CLAUDE_CODE_BRIEF_UPLOAD` | Enables attachment upload when running as subprocess (e.g., cowork desktop bridge) |

### Feature Flags

| Flag | Purpose |
|------|---------|
| `KAIROS` | Enables assistant mode (includes Brief) |
| `KAIROS_BRIEF` | Enables Brief independently of full assistant mode |
| `BRIDGE_MODE` | Enables REPL bridge for cloud attachment upload |

## Error Handling

### Attachment Validation Errors

| Error Code | Condition |
|------------|-----------|
| 1 | Attachment is not a regular file (e.g., directory) |
| 1 | Attachment does not exist (ENOENT) |
| 1 | Attachment is not accessible (permission denied) |

### Upload Failures

Upload failures are silent — they return `undefined` for `file_uuid` and the attachment retains its local path. This ensures:
- Local terminal rendering works unaffected
- Same-machine desktop rendering works
- Web viewers see inert attachment cards without preview

### Resume Safety

Both `attachments` and `sentAt` are optional in the output schema because:
- Resumed sessions replay pre-attachment outputs verbatim
- A required field would crash the UI renderer on resume
- Pre-sentAt outputs don't have timestamps

## Data Flow

```
Model calls SendUserMessage with { message, attachments?, status? }
    │
    ▼
call()
    ├── Capture sentAt timestamp
    ├── Log analytics event
    │
    ├── [no attachments] ──→ return { message, sentAt }
    │
    └── [has attachments] ──→ resolveAttachments()
        │
        ├── validateAttachmentPaths() — stat each file
        │
        ├── Build ResolvedAttachment[] with path, size, isImage
        │
        └── [BRIDGE_MODE enabled] ──→ uploadBriefAttachment() (parallel)
            │
            ├── Check size limit (30 MB)
            ├── Get OAuth token
            ├── Read file
            ├── Build multipart form
            ├── POST to /api/oauth/file_upload
            └── Extract file_uuid from response
        │
        ▼
    return { message, attachments: resolved[], sentAt }
    │
    ▼
mapToolResultToToolResultBlockParam()
    └── "Message delivered to user. (N attachment(s) included)"
    │
    ▼
renderToolResultMessage()
    ├── Transcript mode: ⏺ + markdown + attachments
    ├── Brief-only mode: "Claude" label + timestamp + markdown + attachments
    └── Default mode: plain markdown + attachments (no chrome)
```

## Integration Points

- **REPL Bridge**: Attachment upload to web viewers via `/api/oauth/file_upload`
- **Image Paste**: Pasted images in AskUserQuestionTool flow through the same image store
- **Plan Mode**: Used for all user communication during plan mode interviews
- **Assistant Mode**: Primary output channel — system prompt mandates its use for all user-visible text
- **SDK**: Available as a tool in the SDK `tools` option for programmatic message delivery
- **Analytics**: `tengu_brief_send` events track proactive vs. normal messages and attachment usage

## Related Modules

- [AskUserQuestionTool](./AskUserQuestionTool.md) — Interactive questions that may precede BriefTool messages
- [ExitPlanModeTool](./ExitPlanModeTool.md) — Plan mode exit that transitions to BriefTool communication
- [permissions/](../components/permissions/) — Permission system that BriefTool interacts with in plan mode
