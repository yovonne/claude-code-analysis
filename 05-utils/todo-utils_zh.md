# Todo 工具

## 目的

Todo 工具模块定义 Claude Code 使用的 todo 列表项的 Zod 模式和 TypeScript 类型。它提供用于跟踪带有内容、状态和活动形式字段的任务的验证数据结构。

## 位置

- `restored-src/src/utils/todo/types.ts` — Todo 项和列表模式定义

## 主要导出

### 模式

#### `TodoStatusSchema`
三个有效状态的 Zod 枚举模式：
- `'pending'` — 任务尚未开始
- `'in_progress'` — 任务正在处理中
- `'completed'` — 任务已完成

#### `TodoItemSchema`
单个 todo 项的 Zod 对象模式：
- `content`: 非空字符串（todo 文本）
- `status`: 三个状态枚举值之一
- `activeForm`: 非空字符串（显示用的主动动词形式，例如 "Implementing feature X"）

#### `TodoListSchema`
`TodoItemSchema` 对象数组的 Zod 数组模式。

### 类型

#### `TodoItem`
从 `TodoItemSchema` 推断的 TypeScript 类型：
```typescript
{
  content: string;    // 非空
  status: 'pending' | 'in_progress' | 'completed';
  activeForm: string; // 非空
}
```

#### `TodoList`
从 `TodoListSchema` 推断的类型：`TodoItem[]`

## 依赖

- `zod/v4` — 模式验证
- `../lazySchema.js` — 延迟模式评估以避免循环依赖

## 设计注意事项

- **延迟模式评估**：两个模式都使用 `lazySchema()` 包装器来延迟评估，防止模块初始化期间的循环依赖问题。
- **定义时验证**：`content` 和 `activeForm` 字段在模式级别（`min(1)`）强制非空字符串，确保无效数据被早期拒绝。
- **单一职责**：此模块故意最小化 — 它仅定义类型和模式。所有 todo 业务逻辑（创建、更新、渲染）都在其他地方。
