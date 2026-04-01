# Branch and Issue Commands

## Purpose

Documents the CLI commands for conversation branching (`/branch`) and issue management (`/issue`). The `/branch` command creates a fork of the current conversation, while `/issue` is an internal stub.

---

## `/branch`

### Purpose

Create a branch (fork) of the current conversation at the current point. This duplicates the entire conversation history into a new session with a unique session ID, allowing the user to explore alternative directions while preserving the original conversation. Optionally accepts a custom name for the branch.

### Location

- `restored-src/src/commands/branch/index.ts` — Command registration
- `restored-src/src/commands/branch/branch.ts` — Implementation

### Command Registration

```typescript
{
  type: 'local-jsx',
  name: 'branch',
  aliases: feature('FORK_SUBAGENT') ? [] : ['fork'],
  description: 'Create a branch of the current conversation at this point',
  argumentHint: '[name]',
  load: () => import('./branch.js'),
}
```

The command has an alias `fork` when the `FORK_SUBAGENT` feature flag is not active (i.e., `/fork` doesn't exist as its own command).

### Key Exports

#### Functions

| Export | Description |
|--------|-------------|
| `call(onDone, context, args)` | Entry point. Creates a fork with optional custom title, then resumes into the new session. |
| `createFork(customTitle?)` | Core fork logic. Reads the current transcript, copies messages with a new session ID, and writes the fork file. |
| `getUniqueForkName(baseName)` | Generates a collision-free fork name by checking existing session titles. |
| `deriveFirstPrompt(firstUserMessage)` | Extracts a single-line title base from the first user message in the conversation. |

#### Types

| Export | Description |
|--------|-------------|
| `TranscriptEntry` | Extends `TranscriptMessage` with an optional `forkedFrom` field containing the original `sessionId` and `messageUuid`. |

### Implementation Details

#### Core Fork Logic (`createFork`)

The fork creation process:

1. **Generate new session ID** — `randomUUID()` creates a unique ID for the fork.
2. **Locate transcript files** — Gets the current transcript path and computes the fork session path.
3. **Read current transcript** — Parses the JSONL transcript file. Throws if empty or missing.
4. **Filter main conversation entries** — Excludes sidechain messages and non-message entries (metadata).
5. **Preserve content-replacement records** — Copies `content-replacement` entries that track which `tool_result` blocks were replaced with previews by the per-message budget. Without these, resuming the fork would send full content instead of previews, causing prompt cache misses and permanent overage.
6. **Build forked entries** — For each message:
   - Assigns the new `sessionId`.
   - Sets `parentUuid` chain for message ordering.
   - Adds `forkedFrom` metadata with the original `sessionId` and `messageUuid`.
   - Sets `isSidechain: false`.
7. **Write fork file** — Serializes all entries as JSONL and writes to the fork session path with mode `0o600`.

#### Unique Name Generation (`getUniqueForkName`)

Generates collision-free fork names:

1. Tries `{baseName} (Branch)` first.
2. If that exists, searches for all sessions matching `{baseName} (Branch` pattern.
3. Extracts existing fork numbers using regex: `^{baseName} \(Branch(?: (\d+))?\)$`.
4. Finds the next available number starting from 2.
5. Returns `{baseName} (Branch {nextNumber})`.

#### Title Derivation (`deriveFirstPrompt`)

Extracts a title from the first user message:

1. Gets the message content (string or first text block from content array).
2. Collapses all whitespace to single spaces.
3. Trims and truncates to 100 characters.
4. Falls back to `'Branched conversation'` if no content available.

#### Resume Flow

After fork creation:

1. Builds a `LogOption` object with the fork's metadata (date, messages, first prompt, message count, custom title, content replacements).
2. Saves the custom title via `saveCustomTitle()` so `/status` and `/resume` show the correct name.
3. Logs the `tengu_conversation_forked` analytics event with message count and title flag.
4. Calls `context.resume(sessionId, forkLog, 'fork')` to switch into the forked session.
5. Displays a success message with instructions to resume the original session.

### Dependencies

#### Internal Dependencies

| Module | Purpose |
|--------|---------|
| `src/bootstrap/state.js` | `getOriginalCwd()`, `getSessionId()` |
| `src/services/analytics/index.js` | `logEvent()` for telemetry |
| `src/utils/json.js` | `parseJSONL()` — JSONL parsing |
| `src/utils/sessionStorage.js` | `getProjectDir()`, `getTranscriptPath()`, `getTranscriptPathForSession()`, `isTranscriptMessage()`, `saveCustomTitle()`, `searchSessionsByCustomTitle()` |
| `src/utils/slowOperations.js` | `jsonStringify()` — safe JSON serialization |
| `src/utils/stringUtils.js` | `escapeRegExp()` — regex escaping for fork name pattern |
| `src/types/logs.js` | `TranscriptMessage`, `SerializedMessage`, `ContentReplacementEntry`, `LogOption`, `Entry` types |

### Data Flow

```
/branch [optional-name]
  │
  ├── deriveFirstPrompt(firstUserMessage)     ← Extract title base
  │
  ├── createFork(customTitle)
  │     ├── randomUUID()                      ← New session ID
  │     ├── readFile(currentTranscriptPath)   ← Read JSONL transcript
  │     ├── parseJSONL(entries)               ← Parse all entries
  │     ├── filter(mainConversationEntries)   ← Exclude sidechains
  │     ├── filter(contentReplacementRecords) ← Preserve preview replacements
  │     ├── Build forked entries:
  │     │     ├── sessionId = new UUID
  │     │     ├── forkedFrom = { originalSessionId, messageUuid }
  │     │     └── isSidechain = false
  │     ├── Append content-replacement entry  ← With fork's sessionId
  │     └── writeFile(forkSessionPath)        ← JSONL with mode 0o600
  │
  ├── getUniqueForkName(baseName)             ← Collision-free name
  ├── saveCustomTitle(sessionId, title, path) ← Persist title
  ├── logEvent('tengu_conversation_forked')   ← Analytics
  │
  └── context.resume(sessionId, forkLog, 'fork')
        │
        └── onDone('Branched conversation "{title}". You are now in the branch.
                    To resume the original: claude -r {originalSessionId}')
```

### Error Handling

| Error Condition | Response |
|----------------|----------|
| No transcript file | `Error('No conversation to branch')` → `onDone('Failed to branch conversation: ...')` |
| Empty transcript file | `Error('No conversation to branch')` → same |
| No main conversation messages | `Error('No messages to branch')` → same |
| Resume context unavailable | Falls back to message: `Resume with: /resume {sessionId}` |
| Unknown error | `onDone('Failed to branch conversation: Unknown error occurred')` |

### Configuration

| Feature Flag | Effect |
|-------------|--------|
| `FORK_SUBAGENT` | When active, removes the `fork` alias (because `/fork` exists as its own command) |

---

## `/issue`

### Purpose

Issue creation/management command. Currently implemented as a disabled stub.

### Location

- `restored-src/src/commands/issue/index.js`

### Command Registration

```javascript
{
  isEnabled: () => false,
  isHidden: true,
  name: 'stub',
}
```

### Implementation Details

This command is a **disabled stub** — `isEnabled` always returns `false` and `isHidden` is `true`. It has no functional implementation and cannot be invoked by users.

The command is listed in `INTERNAL_ONLY_COMMANDS` in the command system (`commands.ts`), meaning it would only be visible to Anthropic internal users (`USER_TYPE === 'ant'`) even if enabled. The intended functionality for issue creation/tracking is not present in this version of the codebase.

---

## Related Modules

- [Command System](../01-core-modules/command-system.md)
- [Session Storage Utils](../05-utils/session-storage.md)
- [Git Utils](../05-utils/git-utils.md)
