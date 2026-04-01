# Memory Utilities

## Purpose

The memory utilities module defines the types and constants for Claude Code's memory system, which categorizes stored knowledge into different scopes (User, Project, Local, Managed, AutoMem, and optionally TeamMem). It also provides a lightweight synchronous check for whether a directory is within a Git repository.

## Location

- `restored-src/src/utils/memory/types.ts` — Memory type enum values
- `restored-src/src/utils/memory/versions.ts` — Git repository detection helper

## Key Exports

### Types (`types.ts`)

#### Constants
- `MEMORY_TYPE_VALUES`: Readonly array of valid memory type strings: `'User'`, `'Project'`, `'Local'`, `'Managed'`, `'AutoMem'`, and conditionally `'TeamMem'` (only when the `TEAMMEM` feature flag is enabled)

#### Types
- `MemoryType`: Union type derived from `MEMORY_TYPE_VALUES` — the canonical type for all memory scope identifiers

### Git Detection (`versions.ts`)

#### Functions
- `projectIsInGitRepo(cwd: string): boolean`: Synchronously checks whether a given working directory is inside a Git repository by calling `findGitRoot()`. Uses filesystem walking (no subprocess). The comment notes that `dirIsInGitRepo()` should be preferred for async checks.

## Dependencies

- `bun:bundle` — Feature flag system for conditional `TeamMem` type
- `../git.js` — Git root detection (`findGitRoot`)

## Design Notes

- The memory type enum is feature-flagged: `TeamMem` is only available when the `TEAMMEM` feature is enabled, allowing gradual rollout of team-scoped memory.
- `projectIsInGitRepo` is intentionally synchronous because it is called from contexts where async I/O is not feasible. It uses `findGitRoot` which walks the filesystem without spawning subprocesses.
