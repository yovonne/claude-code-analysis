# Worktree 模式

## 目的

单一函数，无条件为所有用户启用 worktree 模式。之前由 GrowthBook 特性标志门控，但该标志的缓存模式在缓存填充前的首次启动时导致静默失败。

## 位置

- `restored-src/src/utils/worktreeModeEnabled.ts`

## 主要导出

### 函数

- `isWorktreeModeEnabled(): boolean`: 始终返回 `true`。Worktree 模式无条件为所有用户启用。

## 设计注意事项

- 之前由 GrowthBook 标志 `tengu_worktree_mode` 门控，但 `CACHED_MAY_BE_STALE` 模式在缓存填充前的首次启动时返回默认值（`false`），静默吞噬 `--worktree`
- 参见 GitHub issue: https://github.com/anthropics/claude-code/issues/27044
- 这是一个 12 行的文件 — 整个实现是 `return true`
