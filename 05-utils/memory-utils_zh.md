# 记忆工具

## 目的

记忆工具模块定义 Claude Code 记忆系统的类型和常量，将存储的知识分类到不同的作用域（User、Project、Local、Managed、AutoMem 和可选的 TeamMem）。它还提供了轻量级同步检查，用于判断目录是否在 Git 仓库内。

## 位置

- `restored-src/src/utils/memory/types.ts` — 记忆类型枚举值
- `restored-src/src/utils/memory/versions.ts` — Git 仓库检测辅助函数

## 主要导出

### 类型（`types.ts`）

#### 常量
- `MEMORY_TYPE_VALUES`: 有效记忆类型字符串的只读数组：`'User'`、`'Project'`、`'Local'`、`'Managed'`、`'AutoMem'`，以及条件性地 `'TeamMem'`（仅当 `TEAMMEM` 特性标志启用时）

#### 类型
- `MemoryType`: 从 `MEMORY_TYPE_VALUES` 派生的联合类型 — 所有记忆作用域标识符的规范类型

### Git 检测（`versions.ts`）

#### 函数
- `projectIsInGitRepo(cwd: string): boolean`: 同步检查给定工作目录是否在 Git 仓库内，通过调用 `findGitRoot()`。使用文件系统遍历（无子进程）。注释指出 `dirIsInGitRepo()` 应优先用于异步检查。

## 依赖

- `bun:bundle` — 用于条件性 `TeamMem` 类型的特性标志系统
- `../git.js` — Git 根检测（`findGitRoot`）

## 设计注意事项

- 记忆类型枚举是特性标志化的：`TeamMem` 仅在 `TEAMMEM` 特性启用时可用，允许团队作用域记忆的逐步推出。
- `projectIsInGitRepo` 故意设计为同步的，因为它在无法进行异步 I/O 的上下文中调用。它使用 `findGitRoot`，在遍历文件系统时不生成子进程。
