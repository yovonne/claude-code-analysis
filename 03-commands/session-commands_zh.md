# 会话生命周期命令

## 目的

记录管理完整会话生命周期的 CLI 命令：查看会话信息、恢复过去的对话、重命名、导出、分享、填充和跨设备传送。

---

## /session（远程会话信息）

### 目的
在远程模式下显示远程会话 URL 并生成二维码以供浏览器访问。

### 位置
`restored-src/src/commands/session/index.ts`
`restored-src/src/commands/session/session.tsx`

### 命令定义
| 属性 | 值 |
|----------|-------|
| 名称 | `session` |
| 别名 | `remote` |
| 类型 | `local-jsx` |
| 描述 | Show remote session URL and QR code |
| 启用 | 仅当 `getIsRemoteMode()` 返回 true 时 |
| 隐藏 | 不在远程模式时 |

### 关键导出

#### 函数
- `call`: `LocalJSXCommandCall` — 渲染 `SessionInfo` React 组件

### 实现细节

#### 核心逻辑
命令渲染 `SessionInfo` React 组件：
1. 通过 `useAppState` 从 app 状态读取 `remoteSessionUrl`
2. 使用 `qrcode` 库从 URL 生成 ASCII 二维码（UTF8 格式，低错误纠正）
3. 显示逐行的二维码及其下的原始 URL
4. 不在远程模式时显示警告消息

#### UI 结构
```
Pane (context: Confirmation)
├── "Remote session" (bold header)
├── QR code lines (or "Generating QR code…" while loading)
├── "Open in browser: <url>" (URL in ide color)
└── "(press esc to close)" (dim)
```

#### 边缘情况
- 如果 `remoteSessionUrl` 为 null/undefined，显示警告："Not in remote mode. Start with `claude --remote` to use this command."
- 二维码生成失败被捕获并通过 `logForDebugging` 记录而不崩溃

---

## /resume（会话恢复）

### 目的
通过从会话日志中选择或提供会话 ID、搜索词或自定义标题来恢复之前的对话。

### 位置
`restored-src/src/commands/resume/index.ts`
`restored-src/src/commands/resume/resume.tsx`

### 命令定义
| 属性 | 值 |
|----------|-------|
| 名称 | `resume` |
| 别名 | `continue` |
| 类型 | `local-jsx` |
| 描述 | Resume a previous conversation |
| 参数提示 | `[conversation id or search term]` |

### 关键导出

#### 函数
- `call`: `LocalJSXCommandCall` — 入口点；显示选择器或按参数解析
- `filterResumableSessions(logs, currentSessionId)`：从日志列表中过滤掉侧链会话和当前会话

#### 类型
- `ResumeResult`：区分联合 — `{ resultType: 'sessionNotFound', arg }` | `{ resultType: 'multipleMatches', arg, count }`

### 实现细节

#### 核心逻辑 — 两种模式

**模式 1：无参数 — 交互式选择器**
1. 加载当前仓库的会话日志（包括 worktree 路径）
2. 通过 `filterResumableSessions` 过滤掉侧链会话和当前会话
3. 渲染 `LogSelector` 组件，带有：
   - "all projects" vs "same repo" 过滤切换
   - Agentic 会话搜索能力
   - 基于终端大小和模态上下文的最大高度
4. 选择时：
   - 从日志验证会话 UUID
   - 如果是精简日志则加载完整日志
   - 检查跨项目恢复：
     - **相同仓库 worktree**：直接恢复
     - **不同项目**：将恢复命令复制到剪贴板并显示说明
   - 调用 `context.resume(sessionId, log, entrypoint)` 执行恢复

**模式 2：提供参数 — 直接解析**
解析优先级顺序：
1. **有效 UUID**：直接会话 ID 匹配，提取以防被富化过滤掉的会话回退到 `getLastSessionLog`
2. **精确自定义标题匹配**：当启用自定义标题时，按标题搜索
3. **错误**：显示"session not found"或"multiple matches"消息

#### 跨项目恢复处理
当从不同目录选择会话时：
- 恢复命令（例如 `claude --continue <id> --cwd <path>`）被复制到剪贴板
- 用户看到格式化的输出和"(Command copied to clipboard)"通知
- 同一仓库的 worktree 被视为可直接恢复

---

## /rename（会话重命名）

### 目的
重命名当前对话会话，可以使用用户提供的名称或基于对话内容 AI 生成的名称。

### 位置
`restored-src/src/commands/rename/index.ts`
`restored-src/src/commands/rename/rename.ts`
`restored-src/src/commands/rename/generateSessionName.ts`

### 命令定义
| 属性 | 值 |
|----------|-------|
| 名称 | `rename` |
| 类型 | `local-jsx` |
| 描述 | Rename the current conversation |
| Immediate | `true` |
| 参数提示 | `[name]` |

### 关键导出

#### 函数
- `call(onDone, context, args)`：重命名命令的主入口点
- `generateSessionName(messages, signal)`：使用 Haiku 模型从对话内容生成驼峰式会话名称

### 实现细节

#### 核心逻辑
1. **队友守卫**：如果会话是 swarm 队友，重命名被阻止，消息为："This session is a swarm teammate. Teammate names are set by the team leader."
2. **名称解析**：
   - **无参数**：调用 `generateSessionName()`，使用对话文本查询 Haiku 模型生成驼峰式名称（2-4 个词，例如 "fix-login-bug"）
   - **有参数**：使用 trim 的参数作为新名称
3. **持久化**（三个存储）：
   - `saveCustomTitle(sessionId, newName, fullPath)` — 保存自定义标题到会话存储
   - `updateBridgeSessionTitle(bridgeSessionId, newName, ...)` — 同步标题到 claude.ai/code 桥接（尽力，非阻塞）
   - `saveAgentName(sessionId, newName, fullPath)` — 持久化作为 agent 名称以在提示栏显示
4. **App 状态更新**：更新 `standaloneAgentContext.name`
5. **确认**：显示"Session renamed to: <name>"

#### AI 名称生成
`generateSessionName` 函数：
- 从 compact 边界后的消息中提取对话文本
- 发送到 Haiku 模型，附带请求 kebab-case JSON 输出的系统提示
- 使用带 JSON schema 的结构化输出：`{ name: string }`
- 失败时返回 `null`（超时、速率限制、网络错误）

---

## /export（会话导出）

### 目的
将当前对话导出为文本文件或剪贴板，可选择直接文件写入或交互式对话框。

### 位置
`restored-src/src/commands/export/index.ts`
`restored-src/src/commands/export/export.tsx`

### 命令定义
| 属性 | 值 |
|----------|-------|
| 名称 | `export` |
| 类型 | `local-jsx` |
| 描述 | Export the current conversation to a file or clipboard |
| 参数提示 | `[filename]` |

### 关键导出

#### 函数
- `call(onDone, context, args)`：主入口点
- `extractFirstPrompt(messages)`：提取并截断第一个用户消息以生成文件名
- `sanitizeFilename(text)`：将文本转换为安全文件名
- `formatTimestamp(date)`：格式化为 `YYYY-MM-DD-HHMMSS`

### 实现细节

#### 核心逻辑 — 两种模式

**模式 1：提供文件名参数 — 直接导出**
1. Trim 参数；如果不以 `.txt` 结尾则附加
2. 通过 `join(getCwd(), filename)` 解析完整路径
3. 通过 `renderMessagesToPlainText` 将对话渲染为纯文本
4. 使用 UTF-8 编码和 flush 同步写入文件
5. 显示确认："Conversation exported to: <filepath>"

**模式 2：无参数 — 交互式对话框**
1. 将对话渲染为纯文本
2. 生成默认文件名：
   - 提取第一个用户提示 → sanitize → `<timestamp>-<sanitized>.txt`
   - 如果没有提示：`conversation-<timestamp>.txt`
3. 渲染带内容和默认文件名的 `ExportDialog` 组件
4. 对话框处理用户选择（文件写入或剪贴板复制）

#### 文件名清理
`sanitizeFilename` 函数按顺序应用这些转换：
1. 转换为小写
2. 移除特殊字符（保留 `a-z`、`0-9`、空白、连字符）
3. 将空格替换为连字符
4. 将多个连字符折叠为一个
5. 剥离前导/尾随连字符

---

## /share（会话分享）

### 目的
会话分享功能的占位符。

### 位置
`restored-src/src/commands/share/index.js`

### 命令定义
| 属性 | 值 |
|----------|-------|
| 名称 | `stub` |
| 启用 | `false` |
| 隐藏 | `true` |

此命令当前是一个禁用的存根，分享功能在此版本代码库中未实现。

---

## /backfill-sessions（会话填充）

### 目的
会话填充功能的占位符。

### 位置
`restored-src/src/commands/backfill-sessions/index.js`

### 命令定义
| 属性 | 值 |
|----------|-------|
| 名称 | `stub` |
| 启用 | `false` |
| 隐藏 | `true` |

此命令当前是一个禁用的存根，backfill-sessions 功能在此版本代码库中未实现。

---

## /teleport（跨设备会话迁移）

### 目的
跨设备会话迁移功能的占位符。

### 位置
`restored-src/src/commands/teleport/index.js`

### 命令定义
| 属性 | 值 |
|----------|-------|
| 名称 | `stub` |
| 启用 | `false` |
| 隐藏 | `true` |

此命令当前是一个禁用的存根，teleport 功能在此版本代码库中未实现。

---

## 会话生命周期概述

会话生命周期命令形成一个用于管理对话的内聚系统：

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│  /session   │     │   /resume   │     │   /rename   │
│  (view URL) │     │ (continue)  │     │  (set name) │
└─────────────┘     └─────────────┘     └─────────────┘
       │                    │                    │
       │              ┌─────┴─────┐              │
       │              │ Session   │              │
       │              │ Storage   │◄─────────────┘
       │              └─────┬─────┘
       │                    │
┌─────────────┐     ┌───────┴───────┐
│   /export   │     │  Reserved     │
│  (save to   │     │  /share       │
│   file)     │     │  /teleport    │
└─────────────┘     │  /backfill    │
                    └───────────────┘
```

### 生命周期阶段
1. **活跃会话**：`/session` 显示远程 URL，`/rename` 设置标识，`/export` 保存内容
2. **会话结束**：会话自动持久化到日志存储
3. **恢复**：`/resume` 从日志恢复，具有跨项目和 worktree 感知
4. **未来**：`/share`、`/teleport`、`/backfill-sessions` 为即将推出的功能保留

### 相关模块
- [clear-exit-commands.md](./clear-exit-commands.md) — 会话终止和清理
- [session-lifecycle.md](../09-data-flow/session-lifecycle.md) — 会话生命周期数据流
- [sessionStorage](../../restored-src/src/utils/sessionStorage.js) — 会话存储工具
