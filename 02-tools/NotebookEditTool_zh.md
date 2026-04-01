# NotebookEditTool

## 用途

NotebookEditTool 提供 Jupyter notebook 文件（.ipynb）的单元格级编辑。它支持三种编辑模式——替换、插入和删除——允许模型修改 notebook 单元格，同时保留 notebook 结构、元数据和执行状态。该工具强制执行先读后编辑安全性和文件修改跟踪，以防止静默数据丢失。

## 位置

- `restored-src/src/tools/NotebookEditTool/NotebookEditTool.ts` — 主要工具定义（491 行）
- `restored-src/src/tools/NotebookEditTool/prompt.ts` — 工具提示和描述
- `restored-src/src/tools/NotebookEditTool/constants.ts` — 工具名称常量
- `restored-src/src/tools/NotebookEditTool/UI.tsx` — 工具使用/结果/错误消息的 UI 渲染

## 主要导出

| 导出 | 描述 |
|--------|-------------|
| `NotebookEditTool` | 通过 `buildTool()` 构建的完整工具定义 |
| `inputSchema` | 用于 notebook 编辑输入的 Zod schema |
| `outputSchema` | 编辑结果输出的 Zod schema |
| `Output` | Zod 推断的输出类型 |

## 输入/输出 Schema

### 输入 Schema

```typescript
{
  notebook_path: string,    // 必需：.ipynb 文件的绝对路径
  cell_id?: string,         // 可选：要编辑的单元格 ID（替换/删除时必需）
  new_source: string,       // 必需：单元格的新源代码内容
  cell_type?: 'code' | 'markdown',  // 可选：单元格类型（插入时必需）
  edit_mode?: 'replace' | 'insert' | 'delete',  // 默认：'replace'
}
```

### 输出 Schema

```typescript
{
  new_source: string,           // 写入单元格的新源代码
  cell_id?: string,             // 编辑的单元格的 ID
  cell_type: 'code' | 'markdown',  // 单元格类型
  language: string,             // 编程语言（来自 notebook 元数据）
  edit_mode: string,            // 使用的编辑模式
  error?: string,               // 操作失败时的错误消息
  notebook_path: string,        // notebook 文件的路径（用于归属）
  original_file: string,        // 修改前的原始内容
  updated_file: string,         // 修改后的更新内容
}
```

## 编辑模式

### 替换（默认）

替换现有单元格的内容：

```json
{
  "notebook_path": "/path/to/notebook.ipynb",
  "cell_id": "cell-abc123",
  "new_source": "print('hello')",
  "edit_mode": "replace"
}
```

- 需要 `cell_id`
- `cell_type` 是可选的（默认为当前单元格类型）
- 对于代码单元格：重置 `execution_count` 并清除 `outputs`
- 如果 `cell_id` 引用超出末尾，则自动转换为插入

### 插入

在指定单元格后插入新单元格：

```json
{
  "notebook_path": "/path/to/notebook.ipynb",
  "cell_id": "cell-abc123",
  "new_source": "## New Section",
  "cell_type": "markdown",
  "edit_mode": "insert"
}
```

- 需要 `cell_type`
- `cell_id` 是可选的（默认为在开头插入）
- 新单元格插入在指定单元格**之后**
- 为 nbformat 4.5+ 生成随机单元格 ID

### 删除

从 notebook 中移除单元格：

```json
{
  "notebook_path": "/path/to/notebook.ipynb",
  "cell_id": "cell-abc123",
  "edit_mode": "delete"
}
```

- 需要 `cell_id`
- schema 仍然需要 `new_source` 但被忽略

## 验证管道

### 文件验证

| 检查 | 错误代码 | 消息 |
|-------|-----------|---------|
| 文件必须是 .ipynb | 2 | "File must be a Jupyter notebook (.ipynb file)" |
| 文件未找到 | 1 | "Notebook file does not exist" |
| 无效 JSON | 6 | "Notebook is not valid JSON" |
| 文件尚未读取 | 9 | "File has not been read yet. Read it first before writing to it." |
| 自读取以来文件已修改 | 10 | "File has been modified since read..." |
| UNC 路径 | — | 跳过文件系统验证（安全） |

### 编辑模式验证

| 检查 | 错误代码 | 消息 |
|-------|-----------|---------|
| 无效编辑模式 | 4 | "Edit mode must be replace, insert, or delete" |
| 插入时没有 cell_type | 5 | "Cell type is required when using edit_mode=insert" |

### 单元格 ID 验证

| 检查 | 错误代码 | 消息 |
|-------|-----------|---------|
| 单元格 ID 未找到（替换/删除） | 7 | "Cell with index N does not exist" 或 "Cell with ID not found" |
| 单元格 ID 未找到（插入） | — | 默认为在开头插入 |

### 单元格 ID 解析

该工具支持两种单元格 ID 格式：
1. **实际单元格 ID**：notebook 的内部 `cell.id` 字段（例如 `"cell-abc123"`）
2. **数字索引**：`cell-N` 格式通过 `parseCellId()` 解析（例如 `"cell-0"`）

该工具首先尝试按实际 ID 查找，然后回退到数字索引解析。

## 执行管道

```
1. 接收输入
   - 模型返回带有 { notebook_path, cell_id?, new_source, cell_type?, edit_mode? } 的 NotebookEdit tool_use

2. 路径解析
   - 解析为绝对路径（相对路径相对于 cwd 解析）
   - 跳过 UNC 路径以确保安全（NTLM 凭证泄漏预防）

3. 验证
   - 检查文件扩展名（.ipynb）
   - 检查编辑模式有效性
   - 检查插入所需的 cell_type
   - 验证文件之前已被读取（先读后编辑安全性）
   - 验证自读取以来文件未被外部修改
   - 解析并验证 notebook JSON
   - 验证 cell_id 存在（用于替换/删除）

4. 权限检查
   - checkWritePermissionForTool() — 文件系统写权限

5. 编辑执行
   - 读取文件内容和编码（readFileSyncWithMetadata）
   - 解析 notebook JSON（非记忆化 jsonParse 以避免缓存变更）
   - 按 ID 或索引定位目标单元格
   - 应用编辑：
     - replace：更新单元格源，重置代码单元格的 execution_count/outputs
     - insert：使用随机 ID 创建新单元格（nbformat 4.5+），拼接到 cells 数组
     - delete：从 cells 数组中拼接删除单元格
   - 将更新的 notebook 写入磁盘（writeTextContent，保留编码/行尾）

6. 状态更新
   - 使用写后 mtime 更新 readFileState（防止陈旧的 Read 去重）
   - 如果启用则跟踪文件历史

7. 返回
   - 返回 { new_source, cell_type, language, edit_mode, cell_id, ... }
```

## 先读后编辑安全性

该工具强制要求在编辑之前必须已读取 notebook：

```typescript
const readTimestamp = toolUseContext.readFileState.get(fullPath)
if (!readTimestamp) {
  return { result: false, message: 'File has not been read yet...', errorCode: 9 }
}
```

这防止模型编辑从未见过的 notebook，或在外部更改后根据陈旧视图进行编辑——这会导致静默数据丢失。

## 文件历史跟踪

当文件历史启用时（`fileHistoryEnabled()`）：

```typescript
await fileHistoryTrackEdit(updateFileHistoryState, fullPath, parentMessage.uuid)
```

这跟踪编辑以实现撤销功能，将其链接到父消息 UUID。

## 写后状态管理

写入后，工具使用新的文件内容和修改时间戳更新 `readFileState`：

```typescript
readFileState.set(fullPath, {
  content: updatedContent,
  timestamp: getFileModificationTime(fullPath),
  offset: undefined,
  limit: undefined,
})
```

这是关键的：没有此更新，同一毫秒内的 Read→NotebookEdit→Read 序列将返回针对上下文内内容的陈旧 `file_unchanged` 存根。

## Notebook 格式支持

- 支持 Jupyter notebook 格式 nbformat 4+
- 为 nbformat 4.5+ 生成单元格 ID（使用 `Math.random().toString(36)`）
- 语言从 `notebook.metadata.language_info.name` 检测（默认为 `'python'`）
- 使用 1 空格缩进写入输出（`IPYNB_INDENT = 1`）
- 保留原始文件编码和行尾

## 错误处理

编辑期间的错误返回结构化错误输出而不是抛出：

```typescript
{
  new_source: "...",
  cell_type: "code",
  language: "python",
  edit_mode: "replace",
  error: "Error message",
  cell_id: "...",
  notebook_path: "/path/to/notebook.ipynb",
  original_file: "",
  updated_file: "",
}
```

这允许模型在不中断对话流程的情况下接收错误反馈。

## 依赖

| 模块 | 用途 |
|--------|---------|
| `utils/fileRead.ts` | readFileSyncWithMetadata — 带编码/行尾的单次读取 |
| `utils/file.ts` | writeTextContent、getFileModificationTime |
| `utils/notebook.ts` | parseCellId — 数字单元格 ID 解析 |
| `utils/json.ts` | safeParseJSON — JSON 解析 |
| `utils/permissions/filesystem.ts` | checkWritePermissionForTool |
| `utils/cwd.ts` | getCwd — 当前工作目录 |
| `utils/errors.ts` | isENOENT — 文件未找到检测 |
| `utils/fileHistory.ts` | 用于撤销的文件历史跟踪 |
| `utils/slowOperations.ts` | jsonParse（非记忆化）、jsonStringify |
| `services/analytics/` | 事件日志 |
