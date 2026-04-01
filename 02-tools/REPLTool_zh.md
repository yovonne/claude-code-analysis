# REPLTool

## 用途

REPLTool 不是独立工具，而是一种模式配置，用于控制交互式 CLI 会话期间对模型可见的工具。当 REPL 模式启用时，原始工具（Read、Write、Edit、Glob、Grep、Bash、NotebookEdit、Agent）从直接模型使用中隐藏，迫使其通过 REPL 接口进行通信。这些工具在 REPL VM 上下文中仍然内部可访问。

## 位置

- `restored-src/src/tools/REPLTool/constants.ts` — REPL 模式检测和工具列表（47 行）
- `restored-src/src/tools/REPLTool/primitiveTools.ts` — 原始工具引用（40 行）

## 主要导出

| 导出 | 描述 |
|--------|-------------|
| `REPL_TOOL_NAME` | 常量：`'REPL'` |
| `isReplModeEnabled()` | 确定 REPL 模式是否处于活动状态 |
| `REPL_ONLY_TOOLS` | 从直接模型使用中隐藏的工具名称集合 |
| `getReplPrimitiveTools()` | 实际工具实例的延迟 getter |

## REPL 模式检测

当满足所有以下条件时，REPL 模式启用：

```typescript
function isReplModeEnabled(): boolean {
  // 1. CLAUDE_CODE_REPL 未明确设置为 false/0
  if (isEnvDefinedFalsy(process.env.CLAUDE_CODE_REPL)) return false
  
  // 2. 遗留 CLAUDE_REPL_MODE=1 强制启用
  if (isEnvTruthy(process.env.CLAUDE_REPL_MODE)) return true
  
  // 3. 默认：对 'ant' 用户在 CLI 模式启用，对 SDK 用户关闭
  return (
    process.env.USER_TYPE === 'ant' &&
    process.env.CLAUDE_CODE_ENTRYPOINT === 'cli'
  )
}
```

### 环境变量

| 变量 | 效果 |
|----------|--------|
| `CLAUDE_CODE_REPL=0` | 禁用 REPL 模式（选择退出） |
| `CLAUDE_REPL_MODE=1` | 强制启用 REPL 模式（遗留） |
| `USER_TYPE=ant` | 为 CLI 用户启用 REPL 模式（默认） |
| `CLAUDE_CODE_ENTRYPOINT=cli` | 默认 REPL 模式所需 |

### SDK 用户

SDK 入口点（sdk-ts、sdk-py、sdk-cli）**不**默认为 REPL 模式。SDK 消费者编写直接工具调用（Bash、Read 等）的脚本，REPL 模式会隐藏这些工具。`USER_TYPE` build-time define 确保 ant-native 二进制文件不会强制每个 SDK 子进程进入 REPL 模式。

## 隐藏的工具（REPL_ONLY_TOOLS）

当 REPL 模式启用时，这些工具从模型的直接工具列表中隐藏：

| 工具 | 用途 |
|------|---------|
| `FileRead` | 读取文件内容 |
| `FileWrite` | 写入文件内容 |
| `FileEdit` | 编辑文件内容 |
| `Glob` | 文件模式匹配 |
| `Grep` | 内容搜索 |
| `Bash` | Shell 命令执行 |
| `NotebookEdit` | Jupyter notebook 编辑 |
| `Agent` | 生成子代理 |

这些工具在 REPL VM 上下文中仍然可访问——模型通过 REPL 接口而不是直接调用它们来与它们通信。

## 原始工具引用

`getReplPrimitiveTools()` 返回 REPL 专用工具的实际工具实例：

```typescript
export function getReplPrimitiveTools(): readonly Tool[] {
  return (_primitiveTools ??= [
    FileReadTool,
    FileWriteTool,
    FileEditTool,
    GlobTool,
    GrepTool,
    BashTool,
    NotebookEditTool,
    AgentTool,
  ])
}
```

### 延迟初始化

getter 使用延迟初始化（`??=`）来避免循环依赖问题。导入链 `collapseReadSearch.ts → primitiveTools.ts → FileReadTool.tsx → ...` 通过工具注册表循环回来，因此顶级 const 会遇到"在初始化之前无法访问"。延迟到调用时间可以避免 TDZ（Temporal Dead Zone）。

### 用途

原始工具被导出，以便显示端代码（`collapseReadSearch`、渲染器）可以分类和渲染这些工具的虚拟消息，即使它们不在过滤的执行工具列表中。

## REPL 模式行为

当 REPL 模式活动时：

1. **工具列表过滤**：模型只能看到 REPL 工具，看不到原始工具
2. **REPL VM 上下文**：原始工具在 REPL 的执行上下文中仍然可用
3. **显示渲染**：UI 组件仍然可以使用 `getReplPrimitiveTools()` 对这些工具进行分类和渲染工具消息
4. **用户交互**：用户直接输入命令，这些命令被路由到适当的原始工具

## 依赖

| 模块 | 用途 |
|--------|---------|
| `utils/envUtils.ts` | 环境变量检查（isEnvDefinedFalsy、isEnvTruthy） |
| `../AgentTool/` | Agent 工具引用 |
| `../BashTool/` | Bash 工具引用 |
| `../FileEditTool/` | FileEdit 工具引用 |
| `../FileReadTool/` | FileRead 工具引用 |
| `../FileWriteTool/` | FileWrite 工具引用 |
| `../GlobTool/` | Glob 工具引用 |
| `../GrepTool/` | Grep 工具引用 |
| `../NotebookEditTool/` | NotebookEdit 工具引用 |
