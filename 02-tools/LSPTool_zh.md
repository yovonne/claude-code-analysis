# LSPTool

## 用途

LSPTool 通过与语言服务器协议（LSP）服务器接口提供代码智能功能。它使 Claude 能够执行操作，如转到定义、查找引用、悬停信息、文档/工作区符号、调用层次结构分析和实现查找。该工具与已配置的 LSP 服务器通信，处理文件打开/关闭、过滤 gitignored 结果，并将响应格式化为按文件分组的人类可读输出。仅当 LSP 服务器连接并对文件类型可用时，该工具才启用。

## 位置

- `restored-src/src/tools/LSPTool/LSPTool.ts` — 主要工具定义和 LSP 请求处理（861 行）
- `restored-src/src/tools/LSPTool/prompt.ts` — 工具描述和操作文档（22 行）
- `restored-src/src/tools/LSPTool/schemas.ts` — 所有 LSP 操作的 Zod schema（216 行）
- `restored-src/src/tools/LSPTool/formatters.ts` — 每种操作类型的结果格式化（593 行）
- `restored-src/src/tools/LSPTool/symbolContext.ts` — 用于 UI 上下文的文件位置符号提取（91 行）
- `restored-src/src/tools/LSPTool/UI.tsx` — 带折叠/展开视图的终端 UI 渲染（228 行）

## 主要导出

### 来自 LSPTool.ts

| 导出 | 描述 |
|--------|-------------|
| `LSPTool` | 通过 `buildTool()` 构建的完整工具定义 |
| `Input` | Zod 推断的输入类型：`{ operation, filePath, line, character }` |
| `Output` | Zod 推断的输出类型：`{ operation, result, filePath, resultCount?, fileCount? }` |

### 来自 schemas.ts

| 导出 | 描述 |
|--------|-------------|
| `lspToolInputSchema` | 所有 9 个 LSP 操作的区分联合 schema |
| `LSPToolInput` | 工具输入的 TypeScript 类型 |
| `isValidLSPOperation(operation)` | 有效操作名称的类型守卫 |

### 来自 formatters.ts

| 导出 | 描述 |
|--------|-------------|
| `formatGoToDefinitionResult(result, cwd)` | 格式化定义位置 |
| `formatFindReferencesResult(result, cwd)` | 格式化按文件分组的引用位置 |
| `formatHoverResult(result, cwd)` | 格式化悬停信息 |
| `formatDocumentSymbolResult(result, cwd)` | 格式化文档符号（层级） |
| `formatWorkspaceSymbolResult(result, cwd)` | 格式化按文件分组的工作区符号 |
| `formatPrepareCallHierarchyResult(result, cwd)` | 格式化调用层次结构项 |
| `formatIncomingCallsResult(result, cwd)` | 格式化传入调用引用 |
| `formatOutgoingCallsResult(result, cwd)` | 格式化传出调用引用 |

### 来自 symbolContext.ts

| 导出 | 描述 |
|--------|-------------|
| `getSymbolAtPosition(filePath, line, character)` | 提取特定文件位置处的符号/单词，用于 UI 上下文 |

## 输入/输出 Schema

### 输入 Schema

```typescript
{
  operation: 'goToDefinition' | 'findReferences' | 'hover' | 'documentSymbol' |
             'workspaceSymbol' | 'goToImplementation' | 'prepareCallHierarchy' |
             'incomingCalls' | 'outgoingCalls',
  filePath: string,    // 文件的绝对或相对路径
  line: number,        // 1-based 行号（如编辑器中所示）
  character: number,   // 1-based 字符偏移（如编辑器中所示）
}
```

### 输出 Schema

```typescript
{
  operation: string,       // 执行 LSP 操作
  result: string,          // 格式化结果文本
  filePath: string,        // 执行操作的文件的路径
  resultCount?: number,     // 结果数量（定义、引用、符号）
  fileCount?: number,      // 包含结果的唯一文件数量
}
```

## 支持的操作

| 操作 | LSP 方法 | 描述 |
|-----------|-----------|-------------|
| `goToDefinition` | `textDocument/definition` | 查找符号定义位置 |
| `findReferences` | `textDocument/references` | 查找符号的所有引用 |
| `hover` | `textDocument/hover` | 获取悬停信息（文档、类型信息） |
| `documentSymbol` | `textDocument/documentSymbol` | 获取文档中的符号 |
| `workspaceSymbol` | `workspace/symbol` | 在工作区中搜索符号 |
| `goToImplementation` | `textDocument/implementation` | 查找接口/抽象方法的实现 |
| `prepareCallHierarchy` | `textDocument/prepareCallHierarchy` | 获取位置处的调用层次结构项 |
| `incomingCalls` | `textDocument/prepareCallHierarchy` → `callHierarchy/incomingCalls` | 查找调用此函数的内容 |
| `outgoingCalls` | `textDocument/prepareCallHierarchy` → `callHierarchy/outgoingCalls` | 查找此函数调用的内容 |

## LSP 请求管道

### 执行流程

```
1. 接收输入
   - 模型返回带有 { operation, filePath, line, character } 的 LSP tool_use

2. 验证
   - validateInput()：
     a. 根据区分联合 schema 解析
     b. 检查文件存在且是常规文件
     c. 跳过 UNC 路径 stat（安全：防止 NTLM 凭证泄漏）

3. 权限检查
   - checkReadPermissionForTool()：文件系统读取权限检查

4. LSP 初始化等待
   - 如果初始化是 'pending'，则 await waitForInitialization()
   - 在返回"没有可用的服务器"之前确保服务器已准备好

5. 文件打开
   - 如果文件尚未在 LSP 服务器中打开：
     a. 检查文件大小（最大 10 MB）
     b. 读取文件内容
     c. 发送 textDocument/didOpen 到 LSP 服务器

6. LSP 请求
   - 将操作映射到 LSP 方法和参数
   - 将 1-based line/character 转换为 0-based LSP 协议
   - 发送请求到 LSP 服务器

7. 调用层次结构两步（incomingCalls/outgoingCalls）
   - 第一请求返回 CallHierarchyItem(s)
   - 第二请求使用项获取实际调用

8. GITIGNORED 文件过滤
   - 对于基于位置的结果：
     a. 从结果中提取 URI
     b. 批量检查路径与 `git check-ignore`（每批 50 个）
     c. 过滤掉 gitignored 位置

9. 结果格式化
   - 根据操作类型格式化
   - 计算结果和唯一文件的数量
   - 用相对路径按文件分组
```

### 位置转换

用户输入使用 1-based 索引（编辑器约定）。LSP 协议使用 0-based 索引：

```typescript
const position = {
  line: input.line - 1,         // 1-based → 0-based
  character: input.character -  // 1-based → 0-based
}
```

### 调用层次结构两步过程

对于 `incomingCalls` 和 `outgoingCalls`：

```
步骤 1: textDocument/prepareCallHierarchy
  → 返回 CallHierarchyItem[]（位置处的函数/方法）

步骤 2: callHierarchy/incomingCalls 或 callHierarchy/outgoingCalls
  → 使用步骤 1 的第一项
  → 返回 CallHierarchyIncomingCall[] 或 CallHierarchyOutgoingCall[]
```

### Gitignore 过滤

使用批量 `git check-ignore` 过滤结果以排除 gitignored 文件：

```
1. 从结果 URI 中提取唯一文件路径
2. 批量分为 50 个一组
3. 运行：git check-ignore <path1> <path2> ...
4. 收集忽略的路径（退出代码 0 = 至少一个被忽略）
5. 过滤结果以排除忽略的路径
```

每批 50 个平衡效率和命令行长度限制。每批超时为 5 秒。

## 结果格式化

### 位置格式化

URI 在可能时转换为相对路径：

```
file:///home/user/project/src/file.ts → src/file.ts:42:10
```

规则：
- 移除 `file://` 协议前缀
- 解码百分比编码的字符
- 处理 Windows 驱动器字母（`/C:/path` → `C:/path`）
- 如果更短且不以 `../../` 开头，则使用相对路径
- 将反斜杠规范化 为正斜杠

### 操作特定格式

| 操作 | 格式 |
|-----------|--------|
| `goToDefinition` | "Defined in file:line:col" 或 "Found N definitions:\n  file:line:col" |
| `findReferences` | "Found N references across M files:\n\nfile:\n  Line L:C\n  Line L:C" |
| `hover` | 原始悬停内容（markdown/纯文本） |
| `documentSymbol` | 层级树："name (Kind) detail - Line N"，带缩进 |
| `workspaceSymbol` | "Found N symbols in workspace:\n\nfile:\n  name (Kind) - Line N in container" |
| `prepareCallHierarchy` | "Call hierarchy item: name (Kind) - file:line [detail]" |
| `incomingCalls` | "Found N incoming calls:\n\nfile:\n  name (Kind) - Line L [calls at: L:C, L:C]" |
| `outgoingCalls` | "Found N outgoing calls:\n\nfile:\n  name (Kind) - Line L [called from: L:C, L:C]" |

### 空结果消息

每个操作在未找到结果时提供有用的上下文：

| 操作 | 空消息 |
|-----------|--------------|
| `goToDefinition` | "No definition found. This may occur if the cursor is not on a symbol, or if the definition is in an external library not indexed by the LSP server." |
| `findReferences` | "No references found. This may occur if the symbol has no usages, or if the LSP server has not fully indexed the workspace." |
| `hover` | "No hover information available. This may occur if the cursor is not on a symbol, or if the LSP server has not fully indexed the file." |
| `documentSymbol` | "No symbols found in document. This may occur if the file is empty, not supported by the LSP server, or if the server has not fully indexed the file." |
| `workspaceSymbol` | "No symbols found in workspace. This may occur if the workspace is empty, or if the LSP server has not finished indexing the project." |

## 符号上下文提取

`getSymbolAtPosition()` 提取光标位置的符号以显示在工具使用消息中：

- 仅读取文件的前 64KB（覆盖约 1000 行）
- 使用正则表达式模式：`/[\w$'!]+|[+\-*/%&|^~<>=]+/g`
- 处理标准标识符、Rust 生命周期（`'a`）、Rust 宏（`macro_name!`）和运算符
- 将符号截断为 30 个字符
- 如果提取失败则回退到位置显示（`line:character`）
- 使用同步 I/O（从同步 React 渲染调用）

## UI 渲染

`LSPResultSummary` 组件提供折叠/展开视图：

- **非详细**：显示计数摘要（例如，"Found 3 references across 2 files"），带 `Ctrl+O` 展开
- **详细**：在摘要下方显示完整结果内容

操作特定的标签被映射：

| 操作 | 单数 | 复数 |
|-----------|----------|--------|
| `goToDefinition` | definition | definitions |
| `findReferences` | reference | references |
| `hover` | hover info | hover info（特殊："available"） |
| `documentSymbol` | symbol | symbols |
| `workspaceSymbol` | symbol | symbols |
| `goToImplementation` | implementation | implementations |
| `prepareCallHierarchy` | call item | call items |
| `incomingCalls` | caller | callers |
| `outgoingCalls` | callee | callees |

## 错误处理

| 错误条件 | 响应 |
|----------------|----------|
| 文件不存在 | `{ result: false, message: 'File does not exist: <path>', errorCode: 1 }` |
| 路径不是文件 | `{ result: false, message: 'Path is not a file: <path>', errorCode: 2 }` |
| 无效输入 schema | `{ result: false, message: 'Invalid input: <error>', errorCode: 3 }` |
| 文件访问错误 | `{ result: false, message: 'Cannot access file: <path>. <error>', errorCode: 4 }` |
| 文件太大（>10MB） | `{ result: 'File too large for LSP analysis (XMB exceeds 10MB limit)' }` |
| 没有可用的 LSP 服务器 | `{ result: 'No LSP server available for file type: <ext>' }` |
| LSP 请求失败 | `{ result: 'Error performing <operation>: <error>' }` |
| LSP 服务器管理器未初始化 | `{ result: 'LSP server manager not initialized' }` |

## 配置

### 限制

| 限制 | 值 | 描述 |
|-------|-------|-------------|
| `MAX_LSP_FILE_SIZE_BYTES` | 10 MB | LSP 分析的最大文件大小 |
| `maxResultSizeChars` | 100,000 | 存储在上下文中的最大结果大小 |
| Git check-ignore 批量大小 | 50 | gitignore 检查的每个批次路径数 |
| Git check-ignore 超时 | 5 秒 | 每批超时 |

### 特性标志

| 条件 | 效果 |
|-----------|--------|
| `isLspConnected()` 返回 false | 工具被禁用 |

## 依赖

### 内部

| 模块 | 用途 |
|--------|---------|
| `services/lsp/manager.js` | LSP 服务器生命周期和请求处理 |
| `utils/cwd.js` | 当前工作目录 |
| `utils/path.js` | 路径扩展 |
| `utils/fsOperations.js` | 文件系统抽象 |
| `utils/permissions/filesystem.js` | 读取权限检查 |
| `utils/execFileNoThrow.js` | Git check-ignore 执行 |
| `utils/debug.js` | 调试日志 |
| `utils/format.js` | 截断实用工具 |
| `utils/messages.js` | 消息提取 |
| `utils/file.js` | 显示路径格式化 |

### 外部

| 包 | 用途 |
|---------|---------|
| `vscode-languageserver-types` | LSP 类型定义（Location、SymbolInformation 等） |
| `zod/v4` | 输入/输出 schema 验证 |
| `react` | UI 渲染 |
| `fs/promises` | 文件操作 |
| `path`、`url` | 路径和 URI 处理 |

## 数据流

```
Model Request
    │
    ▼
LSPTool Input { operation, filePath, line, character }
    │
    ▼
┌─────────────────────────────────────────────────────────────────┐
│ VALIDATION                                                      │
│ - Schema validation (discriminated union)                       │
│ - File exists and is regular file                               │
│ - Skip UNC paths (NTLM security)                                │
└─────────────────────────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────────────────────────┐
│ PERMISSION CHECK                                                │
│  - Filesystem read permission                                   │
└─────────────────────────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────────────────────────┐
│ LSP INITIALIZATION                                              │
│  - Wait for initialization if pending                           │
│  - Get LSP server manager                                       │
│  - Open file in LSP if not already open (max 10MB)            │
└─────────────────────────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────────────────────────┐
│ LSP REQUEST                                                       │
│                                                                 │
│  1. Map operation → LSP method + params                         │
│  2. Convert 1-based → 0-based position                          │
│  3. Send request to LSP server                                  │
│  4. For call hierarchy: two-step process                        │
│  5. Filter gitignored results                                   │
└─────────────────────────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────────────────────────┐
│ RESULT FORMATTING                                               │
│  - Format based on operation type                               │
│  - Count results and unique files                               │
│  - Group by file with relative paths                            │
└─────────────────────────────────────────────────────────────────┘
    │
    ▼
Output { operation, result, filePath, resultCount?, fileCount? }
    │
    ▼
Model Response (tool_result block)
```
