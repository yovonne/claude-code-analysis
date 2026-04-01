# ScheduleCronTool (CronCreate, CronDelete, CronList)

## Purpose

The ScheduleCronTool family provides cron-based scheduling of prompts within Claude Code. It supports both recurring schedules and one-shot reminders, with optional disk persistence for cross-session durability. The system uses standard 5-field cron expressions in the user's local timezone.

## Location

- `restored-src/src/tools/ScheduleCronTool/CronCreateTool.ts` — Create cron jobs (158 lines)
- `restored-src/src/tools/ScheduleCronTool/CronDeleteTool.ts` — Cancel cron jobs (96 lines)
- `restored-src/src/tools/ScheduleCronTool/CronListTool.ts` — List active cron jobs (98 lines)
- `restored-src/src/tools/ScheduleCronTool/prompt.ts` — Prompts, descriptions, and feature gates (136 lines)
- `restored-src/src/tools/ScheduleCronTool/UI.tsx` — UI rendering for all three tools

## Key Exports

### CronCreateTool

| Export | Description |
|--------|-------------|
| `CronCreateTool` | Tool for creating cron jobs |
| `CreateOutput` | Output type: `{ id, humanSchedule, recurring, durable? }` |
| `CRON_CREATE_TOOL_NAME` | Constant: `'CronCreate'` |
| `DEFAULT_MAX_AGE_DAYS` | Auto-expiry for recurring tasks |

### CronDeleteTool

| Export | Description |
|--------|-------------|
| `CronDeleteTool` | Tool for canceling cron jobs |
| `DeleteOutput` | Output type: `{ id }` |
| `CRON_DELETE_TOOL_NAME` | Constant: `'CronDelete'` |

### CronListTool

| Export | Description |
|--------|-------------|
| `CronListTool` | Tool for listing cron jobs |
| `ListOutput` | Output type: `{ jobs: [{ id, cron, humanSchedule, prompt, recurring?, durable? }] }` |
| `CRON_LIST_TOOL_NAME` | Constant: `'CronList'` |

## Feature Gating

All three tools share the same enablement gate:

```typescript
isEnabled() {
  return isKairosCronEnabled()
}
```

`isKairosCronEnabled()` combines:
- Build-time `feature('AGENT_TRIGGERS')` flag
- Runtime `tengu_kairos_cron` GrowthBook gate (5-minute refresh)
- `CLAUDE_CODE_DISABLE_CRON` environment variable override (kills entire scheduler)

Default is `true` — /loop is GA. GrowthBook is disabled for Bedrock/Vertex/Foundry and when telemetry is disabled.

### Durable Cron

`isDurableCronEnabled()` controls disk persistence separately:
- Defaults to `true`
- Narrower kill switch than `isKairosCronEnabled`
- When disabled, forces `durable: false` at call site

## CronCreateTool

### Input Schema

```typescript
{
  cron: string,           // 5-field cron: "M H DoM Mon DoW" (e.g., "*/5 * * * *")
  prompt: string,         // Prompt to enqueue at each fire time
  recurring?: boolean,    // true (default) = recurring, false = one-shot
  durable?: boolean,      // true = persist to disk, false (default) = session-only
}
```

### Output Schema

```typescript
{
  id: string,             // Job ID for use with CronDelete
  humanSchedule: string,  // Human-readable schedule description
  recurring: boolean,     // Whether the job is recurring
  durable?: boolean,      // Whether the job persists to disk
}
```

### Validation

| Check | Error Code | Message |
|-------|-----------|---------|
| Invalid cron expression | 1 | "Invalid cron expression. Expected 5 fields: M H DoM Mon DoW." |
| No matching date in next year | 2 | "Cron expression does not match any calendar date in the next year." |
| Too many jobs (max 50) | 3 | "Too many scheduled jobs (max 50). Cancel one first." |
| Durable cron for teammate | 4 | "durable crons are not supported for teammates" |

### Execution

1. Parse and validate cron expression
2. Check job count limit (MAX_JOBS = 50)
3. Check teammate durability restriction
4. Call `addCronTask(cron, prompt, recurring, durable, agentId)`
5. Enable scheduled tasks (`setScheduledTasksEnabled(true)`)
6. Return job ID and human-readable schedule

### Durability

| Mode | Storage | Survives Restart |
|------|---------|-----------------|
| `durable: false` (default) | In-memory session store | No |
| `durable: true` | `.claude/scheduled_tasks.json` | Yes |

Durable crons are not supported for teammates because teammates do not persist across sessions — a durable teammate cron would orphan on restart.

## CronDeleteTool

### Input Schema

```typescript
{
  id: string,    // Job ID from CronCreate
}
```

### Output Schema

```typescript
{
  id: string,    // The deleted job ID
}
```

### Validation

| Check | Error Code | Message |
|-------|-----------|---------|
| Job ID not found | 1 | "No scheduled job with id '...'" |
| Owned by another agent | 2 | "Cannot delete cron job '...': owned by another agent" |

### Teammate Restriction

Teammates may only delete their own cron jobs. The team lead (no teammate context) can delete any job.

### Execution

1. Validate job ID exists
2. Verify ownership (teammates can only delete own jobs)
3. Call `removeCronTasks([id])`
4. Return deleted job ID

## CronListTool

### Input Schema

```typescript
{}  // No input parameters
```

### Output Schema

```typescript
{
  jobs: [{
    id: string,
    cron: string,
    humanSchedule: string,
    prompt: string,
    recurring?: boolean,
    durable?: boolean,
  }]
}
```

### Properties

- **Read-only**: Yes — `isReadOnly()` returns true
- **Concurrency safe**: Yes
- **Team filtering**: Teammates see only their own jobs; team lead sees all

### Output Formatting

Jobs are formatted as:
```
{id} — {humanSchedule}{(recurring) | (one-shot)}{[session-only]}: {truncated prompt}
```

If no jobs exist: `"No scheduled jobs."`

## Scheduler Behavior

### Jitter

The scheduler adds deterministic jitter to prevent API request clustering:

| Task Type | Jitter |
|-----------|--------|
| Recurring | Up to 10% of period late (max 15 min) |
| One-shot on :00 or :30 | Up to 90 seconds early |

### Auto-Expiry

Recurring tasks auto-expire after `DEFAULT_MAX_AGE_DAYS` days. They fire one final time, then are deleted. This bounds session lifetime.

### Fire Conditions

Jobs only fire while the REPL is idle (not mid-query). This prevents interrupting active conversations.

## Cron Expression Format

Standard 5-field cron in local timezone:

```
M H DoM Mon DoW
```

| Field | Range | Example |
|-------|-------|---------|
| Minute | 0-59 | `*/5` (every 5 min), `30` |
| Hour | 0-23 | `14` (2pm), `*` |
| Day of Month | 1-31 | `28`, `*` |
| Month | 1-12 | `2` (Feb), `*` |
| Day of Week | 0-7 | `1-5` (weekdays), `*` |

### Examples

| Expression | Meaning |
|------------|---------|
| `*/5 * * * *` | Every 5 minutes |
| `30 14 28 2 *` | Feb 28 at 2:30pm (once per year) |
| `0 9 * * 1-5` | Weekdays at 9am |
| `0 * * * *` | Hourly |
| `57 8 * * *` | Daily at 8:57am (avoids :00) |

### Off-Minute Guidance

To avoid API request clustering, pick minutes that are NOT 0 or 30 when the user's request is approximate:

- "every morning around 9" → `"57 8 * * *"` or `"3 9 * * *"` (not `"0 9 * * *"`)
- "hourly" → `"7 * * * *"` (not `"0 * * * *"`)

Only use minute 0 or 30 when the user names that exact time.

## One-Shot vs Recurring

### One-Shot (recurring: false)

For "remind me at X" or "at <time>, do Y" requests:
- Fire once at the next matching time
- Auto-delete after firing
- Pin minute/hour/day-of-month/month to specific values

### Recurring (recurring: true, default)

For "every N minutes" / "every hour" / "weekdays at 9am" requests:
- Fire on every cron match
- Auto-expire after DEFAULT_MAX_AGE_DAYS days
- Use CronDelete to cancel sooner

## Dependencies

| Module | Purpose |
|--------|---------|
| `utils/cron.ts` | Cron expression parsing and human-readable conversion |
| `utils/cronTasks.ts` | Cron task CRUD, file management, next run calculation |
| `utils/teammateContext.ts` | Teammate context for ownership filtering |
| `utils/envUtils.ts` | Environment variable checking |
| `services/analytics/growthbook.ts` | Feature flag evaluation |
| `bootstrap/state.js` | Scheduled tasks enablement |
| `utils/lazySchema.ts` | Lazy Zod schema definition |
| `utils/semanticBoolean.ts` | Semantic boolean parsing |
