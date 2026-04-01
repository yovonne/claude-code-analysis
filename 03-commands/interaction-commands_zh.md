# 交互命令

## 目的

记录促进会话期间用户交互的 CLI 命令：计划、上下文管理、文件处理、目录管理和内容复制。这些命令帮助用户控制对话流程并管理 Claude Code 可访问的内容。

---

## /plan（计划模式）

### 目的
启用计划模式以进行结构化任务规划，或在已进入计划模式时显示/编辑当前会话计划。

### 位置
`restored-src/src/commands/plan/index.ts`
`restored-src/src/commands/plan/plan.tsx`

### 命令定义
| 属性 | 值 |
|----------|-------|
| 名称 | `plan` |
| 类型 | `local-jsx` |
| 描述 | Enable plan mode or view the current session plan |
| 参数提示 | `[open\|<description>]` |

### 关键导出

#### 函数
- `call(onDone, context, args)`：入口点；启用计划模式或显示/编辑当前计划

#### 组件
- `PlanDisplay`：渲染当前计划内容、文件路径和编辑器提示的 React 组件

### 实现细节

#### 核心逻辑 — 两种模式

**模式 1：不在计划模式 — 启用计划模式**
1. 调用 `handlePlanModeTransition(currentMode, 'plan')` 处理模式转换
2. 通过 `applyPermissionUpdate(prepareContextForPlanMode(...), { type: 'setMode', mode: 'plan', destination: 'session' })` 使用计划模式权限更新 app 状态
3. 如果提供了描述参数（不是"open"），则在启用后触发查询
4. 返回"Enabled plan mode"

**模式 2：已在计划模式 — 查看或编辑计划**
1. 通过 `getPlan()` 检索计划内容，通过 `getPlanFilePath()` 获取文件路径
2. 如果没有计划内容：返回"Already in plan mode. No plan written yet."
3. 如果参数是"open"：在外部编辑器中打开计划文件
4. 否则：渲染 `PlanDisplay` 组件，显示：
   - "Current Plan"标题（粗体）
   - 计划文件路径（暗淡）
   - 计划内容
   - 编辑器提示（如果检测到外部编辑器）：`"/plan open" to edit this plan in <editor-name>`

---

## /compact（上下文压缩）

### 目的
在保持摘要的同时清除对话历史。通过汇总对话减少 token 使用。可选接受自定义汇总说明。

### 位置
`restored-src/src/commands/compact/index.ts`
`restored-src/src/commands/compact/compact.ts`

### 命令定义
| 属性 | 值 |
|----------|-------|
| 名称 | `compact` |
| 类型 | `local` |
| 描述 | Clear conversation history but keep a summary in context |
| 非交互式 | `true` |
| 参数提示 | `<optional custom summarization instructions>` |
| 启用 | 除非 `DISABLE_COMPACT` 环境变量为真 |

### 关键导出

#### 函数
- `call(args, context)`：主入口点；执行对话压缩
- `compactViaReactive(messages, context, customInstructions, reactive)`：响应式模式压缩路径
- `buildDisplayText(context, userDisplayMessage?)`：构建压缩后显示的暗淡摘要文本
- `getCacheSharingParams(context, messages)`：为缓存感知的压缩构建系统提示和上下文

### 实现细节

#### 压缩策略优先级
命令按顺序尝试三种压缩策略：

1. **会话记忆压缩**（仅无自定义说明时）
   - 最快路径 — 使用会话记忆压缩，无需完整模型调用
   - 不支持自定义说明
   - 成功时：清除用户上下文缓存，运行压缩后清理，通知缓存中断检测，抑制压缩警告

2. **响应式压缩**（通过 `REACTIVE_COMPACT` 门控）
   - 当 `reactiveCompact.isReactiveOnlyMode()` 返回 true 时使用
   - 与缓存安全参数构建并行运行压缩前钩子
   - 将钩子说明与自定义说明合并
   - 通过 `context.onCompactProgress` 回调显示进度
   - 处理失败原因：`too_few_groups`、`aborted`、`exhausted`、`error`、`media_unstrippable`

3. **传统压缩**（回退）
   - 首先运行 microcompact 以减少汇总前的 token
   - 使用缓存共享参数调用 `compactConversation`
   - 重置 `lastSummarizedMessageId`（遗留压缩替换所有消息）

#### 预压缩钩子
使用响应式压缩时，预压缩钩子被执行：
- 钩子与缓存参数构建并发运行（独立操作）
- 钩子结果可以通过 `mergeHookInstructions` 添加自定义说明
- 压缩后钩子在 `reactiveCompactOnPromptTooLong` 内部运行
- 前后钩子显示消息合并为最终输出

---

## /context（上下文使用可视化）

### 目的
将当前上下文使用可视化为彩色网格（交互式）或 markdown 表格（非交互式）。显示按类别划分的 token 消耗明细。

### 位置
`restored-src/src/commands/context/index.ts`
`restored-src/src/commands/context/context.tsx`（交互式）
`restored-src/src/commands/context/context-noninteractive.ts`（非交互式）

### 命令定义

**交互式版本：**
| 属性 | 值 |
|----------|-------|
| 名称 | `context` |
| 类型 | `local-jsx` |
| 描述 | Visualize current context usage as a colored grid |
| 启用 | 仅当不在非交互式会话中时 |

**非交互式版本：**
| 属性 | 值 |
|----------|-------|
| 名称 | `context` |
| 类型 | `local` |
| 描述 | Show current context usage |
| 非交互式 | `true` |
| 隐藏 | 当不在非交互式会话中时 |

### 关键导出

#### 函数
- `call(onDone, context)`（交互式）：分析上下文，渲染 `ContextVisualization` 组件
- `call(_args, context)`（非交互式）：将上下文数据作为 markdown 表格返回
- `collectContextData(context)`：两个版本和 SDK 使用的共享数据收集
- `formatContextAsMarkdownTable(data)`（非交互式）：将上下文数据格式化为 markdown 表格

---

## /files（上下文中的文件）

### 目的
列出当前加载到对话上下文中的所有文件。Ant 专用功能。

### 位置
`restored-src/src/commands/files/index.ts`
`restored-src/src/commands/files/files.ts`

### 命令定义
| 属性 | 值 |
|----------|-------|
| 名称 | `files` |
| 类型 | `local` |
| 描述 | List all files currently in context |
| 非交互式 | `true` |
| 启用 | 仅当 `USER_TYPE === 'ant'` 时 |

### 关键导出

#### 函数
- `call(_args, context)`：以相对路径形式返回上下文中文件的列表

---

## /add-dir（添加工作目录）

### 目的
向会话添加新的工作目录，扩展 Claude Code 可访问的文件系统范围。

### 位置
`restored-src/src/commands/add-dir/index.ts`
`restored-src/src/commands/add-dir/add-dir.tsx`
`restored-src/src/commands/add-dir/validation.ts`

### 命令定义
| 属性 | 值 |
|----------|-------|
| 名称 | `add-dir` |
| 类型 | `local-jsx` |
| 描述 | Add a new working directory |
| 参数提示 | `<path>` |

### 关键导出

#### 函数
- `call(onDone, context, args?)`：入口点；验证路径并显示添加目录 UI
- `validateDirectoryForWorkspace(directoryPath, permissionContext)`：验证目录路径
- `addDirHelpMessage(result)`：为验证失败生成人类可读的错误消息

#### 类型
- `AddDirectoryResult`：验证结果的区分联合：
  - `{ resultType: 'success', absolutePath }`
  - `{ resultType: 'emptyPath' }`
  - `{ resultType: 'pathNotFound' | 'notADirectory', directoryPath, absolutePath }`
  - `{ resultType: 'alreadyInWorkingDirectory', directoryPath, workingDir }`

---

## /copy（复制上次响应）

### 目的
将 Claude 的上次响应复制到剪贴板。支持选择特定代码块或完整响应。可以使用 `/copy N` 针对第 N 个最新响应。

### 位置
`restored-src/src/commands/copy/index.ts`
`restored-src/src/commands/copy/copy.tsx`

### 命令定义
| 属性 | 值 |
|----------|-------|
| 名称 | `copy` |
| 类型 | `local-jsx` |
| 描述 | Copy Claude's last response to clipboard (or /copy N for the Nth-latest) |

### 关键导出

#### 函数
- `call(onDone, context, args)`：入口点；收集最近的 assistant 文本并显示复制选择器
- `collectRecentAssistantTexts(messages)`：从新到旧遍历消息，提取 assistant 消息文本
- `extractCodeBlocks(markdown)`：解析 markdown 以提取带围栏的代码块
- `fileExtension(lang)`：将语言标识符映射到文件扩展名
- `copyOrWriteToFile(text, filename)`：复制到剪贴板并写入临时文件
- `writeToFile(text, filename)`：在 `COPY_DIR` 中写入文本到临时文件

#### 组件
- `CopyPicker`：用于选择复制内容的交互式对话框（完整响应或单个代码块）

### 实现细节

#### 助手文本收集
`collectRecentAssistantTexts` 从新到遍历消息：
- 跳过非 assistant 消息和 API 错误消息
- 从数组类型消息内容中提取文本内容
- 最多 `MAX_LOOKBACK`（20）条消息
- 返回数组，其中索引 0 = 最新，1 = 倒数第二个，等等

#### 代码块提取
使用 `marked` 词法分析器解析 markdown 并提取带围栏的代码块：
- 解析前去除提示 XML 标签
- 返回 `{ code, lang }` 对象数组

#### 复制选择器 UI
当有代码块且未设置 `copyFullResponse` 时：
1. 显示带选项的 `Select` 对话框：
   - "Full response"（字符数、行数）
   - 每个代码块（截断标签、语言、行数）
   - "Always copy full response"（偏好设置）
2. 键盘快捷键：
   - `Enter` — 复制选中的内容
   - `w` — 将选中的内容写入文件（跳过剪贴板）
   - `Esc` — 取消
3. 点击"Always copy full response"：将偏好保存到全局配置

#### 剪贴板 + 文件回退
`copyOrWriteToFile`：
1. 尝试通过 `setClipboard` 剪贴板（OSC 52 — 尽力而为）
2. 始终写入临时文件作为可靠回退
3. 返回带字符数、行数和文件路径的消息

---

## 复制操作数

`fileExtension(lang)`：
- 清理语言标识符（仅字母数字，防止路径遍历）
- 对有效语言返回 `.<lang>`（除了 `plaintext`）
- 默认返回 `.txt`
