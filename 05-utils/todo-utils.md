# Todo Utilities

## Purpose

The todo utilities module defines the Zod schemas and TypeScript types for todo list items used throughout Claude Code. It provides validated data structures for tracking tasks with content, status, and active form fields.

## Location

- `restored-src/src/utils/todo/types.ts` — Todo item and list schema definitions

## Key Exports

### Schemas

#### `TodoStatusSchema`
Zod enum schema with three valid statuses:
- `'pending'` — Task not yet started
- `'in_progress'` — Task currently being worked on
- `'completed'` — Task finished

#### `TodoItemSchema`
Zod object schema for individual todo items:
- `content`: Non-empty string (the todo text)
- `status`: One of the three status enum values
- `activeForm`: Non-empty string (active verb form for display, e.g., "Implementing feature X")

#### `TodoListSchema`
Zod array schema — an array of `TodoItemSchema` objects.

### Types

#### `TodoItem`
TypeScript type inferred from `TodoItemSchema`:
```typescript
{
  content: string;    // Non-empty
  status: 'pending' | 'in_progress' | 'completed';
  activeForm: string; // Non-empty
}
```

#### `TodoList`
TypeScript type inferred from `TodoListSchema`: `TodoItem[]`

## Dependencies

- `zod/v4` — Schema validation
- `../lazySchema.js` — Lazy schema evaluation to avoid circular dependencies

## Design Notes

- **Lazy schema evaluation**: Both schemas use `lazySchema()` wrapper to defer evaluation, preventing circular dependency issues during module initialization.
- **Validation at definition**: The `content` and `activeForm` fields enforce non-empty strings at the schema level (`min(1)`), ensuring invalid data is rejected early.
- **Single responsibility**: This module is intentionally minimal — it only defines types and schemas. All todo business logic (creation, updating, rendering) lives elsewhere.
