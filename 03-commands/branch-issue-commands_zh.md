# 分支和 Issue 命令

## 目的

记录用于对话分支（`/branch`）和 issue 管理（`/issue`）的 CLI 命令。`/branch` 命令创建当前对话的分支，而 `/issue` 是一个内部存根。

---

## `/branch`

### 目的

在当前点创建当前对话的分支（fork）。这会将整个对话历史复制到具有唯一会话 ID 的新会话中，允许用户在保留原始对话的同时探索替代方向。可选地接受分支的自定义名称。

### 位置

- `restored-src/src/commands/branch/index.ts` — 命令注册
- `restored-src/src/commands/branch/branch.ts` — 实现

### 命令注册

```typescript
{
  type: 'local-jsx',
  name: 'branch',
  aliases: feature('FORK_SUBAGENT') ? [] : ['fork'],
  description: 'Create a branch of the current conversation at this point',
  argumentHint: '[name]',
  load: () => import('./branch.js'),
}
```

当 `FORK_SUBAGENT` 功能标志未激活时，命令有别名 `fork`（因为 `/fork` 作为自己的命令存在）。

### 关键导出

#### 函数

| 导出 | 描述 |
|--------|-------------|
| `call(onDone, context, args)` | 入口点。使用可选的自定义标题创建分支，然后恢复到新会话。 |
| `createFork(customTitle?)` | 核心分支逻辑。读取当前 transcript，复制带有新会话 ID 的消息，并写入分支文件。 |
| `getUniqueForkName(baseName)` | 通过检查现有会话标题生成无碰撞的分支名称。 |
| `deriveFirstPrompt(firstUserMessage)` | 从对话中的第一个用户消息中提取单行标题基础。 |

#### 类型

| 导出 | 描述 |
|--------|-------------|
| `TranscriptEntry` | 扩展 `TranscriptMessage`，带有可选的 `forkedFrom` 字段，包含原始 `sessionId` 和 `messageUuid`。 |

### 实现细节

#### 核心分支逻辑（`createFork`）

分支创建过程：

1. **生成新会话 ID** — `randomUUID()` 为分支创建唯一 ID。
2. **定位 transcript 文件** — 获取当前 transcript 路径并计算分支会话路径。
3. **读取当前 transcript** — 解析 JSONL transcript 文件。如果为空或缺失则抛出。
4. **过滤主对话条目** — 排除侧链消息和非消息条目（元数据）。
5. **保留内容替换记录** — 复制 `content-replacement` 条目，这些条目跟踪哪些 `tool_result` 块被每消息预算的预览替换。没有这些，恢复分支将发送完整内容而不是预览，导致提示缓存未命中和永久超支。
6. **构建分支条目** — 对于每条消息：
   - 分配新的 `sessionId`。
   - 设置 `parentUuid` 链以进行消息排序。
   - 添加 `forkedFrom` 元数据，包含原始 `sessionId` 和 `messageUuid`。
   - 设置 `isSidechain: false`。
7. **写入分支文件** — 将所有条目序列化为 JSONL 并以模式 `0o600` 写入分支会话路径。

#### 唯一名称生成（`getUniqueForkName`）

生成无碰撞的分支名称：

1. 首先尝试 `{baseName} (Branch)`。
2. 如果存在，搜索匹配 `{baseName} (Branch` 模式的所有会话。
3. 使用正则表达式提取现有分支编号：`^{baseName} \(Branch(?: (\d+))?\)$`。
4. 从 2 开始找到下一个可用编号。
5. 返回 `{baseName} (Branch {nextNumber})`。

#### 标题派生（`deriveFirstPrompt`）

从第一个用户消息中提取标题：

1. 获取消息内容（字符串或内容数组中的第一个文本块）。
2. 将所有空白折叠为单个空格。
3. 截断为 100 个字符。
4. 如果没有可用内容则回退到 `'Branched conversation'`。

#### 恢复流程

分支创建后：

1. 构建包含分支元数据（日期、消息、第一个提示、消息计数、自定义标题、内容替换）的 `LogOption` 对象。
2. 通过 `saveCustomTitle()` 保存自定义标题，以便 `/status` 和 `/resume` 显示正确的名称。
3. 记录带消息计数和标题标志的 `tengu_conversation_forked` 分析事件。
4. 调用 `context.resume(sessionId, forkLog, 'fork')` 切换到分支会话。
5. 显示带有说明的成功消息，以恢复原始会话。

### 依赖

#### 内部依赖

| 模块 | 目的 |
|--------|---------|
| `src/bootstrap/state.js` | `getOriginalCwd()`、`getSessionId()` |
| `src/services/analytics/index.js` | `logEvent()` 用于遥测 |
| `src/utils/json.js` | `parseJSONL()` — JSONL 解析 |
| `src/utils/sessionStorage.js` | `getProjectDir()`、`getTranscriptPath()`、`getTranscriptPathForSession()`、`isTranscriptMessage()`、`saveCustomTitle()`、`searchSessionsByCustomTitle()` |
| `src/utils/slowOperations.js` | `jsonStringify()` — 安全的 JSON 序列化 |
| `src/utils/stringUtils.js` | `escapeRegExp()` — 用于分支名称模式的正则表达式转义 |
| `src/types/logs.js` | `TranscriptMessage`、`SerializedMessage`、`ContentReplacementEntry`、`LogOption`、`Entry` 类型 |

### 数据流

```
/branch [optional-name]
  │
  ├── deriveFirstPrompt(firstUserMessage)     ← Extract title base
  │
  ├── createFork(customTitle)
  │     ├── randomUUID()                      ← New session ID
  │     ├── readFile(currentTranscriptPath)   ← Read JSONL transcript
  │     ├── parseJSONL(entries)               ← Parse all entries
  │     ├── filter(mainConversationEntries)   ← Exclude sidechains
  │     ├── filter(contentReplacementRecords) ← Preserve preview replacements
  │     ├── Build forked entries:
  │     │     ├── sessionId = new UUID
  │     │     ├── forkedFrom = { originalSessionId, messageUuid }
  │     │     └── isSidechain = false
  │     ├── Append content-replacement entry  ← With fork's sessionId
  │     └── writeFile(forkSessionPath)        ← JSONL with mode 0o600
  │
  ├── getUniqueForkName(baseName)             ← Collision-free name
  ├── saveCustomTitle(sessionId, title, path) ← Persist title
  ├── logEvent('tengu_conversation_forked')   ← Analytics
  │
  └── context.resume(sessionId, forkLog, 'fork')
        │
        └── onDone('Branched conversation "{title}". You are now in the branch.
                    To resume the original: claude -r {originalSessionId}')
```

### 错误处理

| 错误条件 | 响应 |
|----------------|----------|
| 无 transcript 文件 | `Error('No conversation to branch')` → `onDone('Failed to branch conversation: ...')` |
| 空 transcript 文件 | `Error('No conversation to branch')` → same |
| 无主对话消息 | `Error('No messages to branch')` → same |
| 恢复上下文不可用 | 回退到消息：`Resume with: /resume {sessionId}` |
| 未知错误 | `onDone('Failed to branch conversation: Unknown error occurred')` |

### 配置

| 功能标志 | 效果 |
|-------------|---------|
| `FORK_SUBAGENT` | 激活时，移除 `fork` 别名（因为 `/fork` 作为自己的命令存在） |

---

## `/issue`

### 目的

Issue 创建/管理命令。当前已实现为禁用的存根。

### 位置

- `restored-src/src/commands/issue/index.js`

### 命令注册

```javascript
{
  isEnabled: () => false,
  isHidden: true,
  name: 'stub',
}
```

### 实现细节

此命令是一个**禁用存根** — `isEnabled` 始终返回 `false` 且 `isHidden` 为 `true`。它没有功能实现，用户无法调用。

该命令在命令系统的 `INTERNAL_ONLY_COMMANDS` 中列出（`commands.ts`），意味着即使启用，它也只对 Anthropic 内部用户（`USER_TYPE === 'ant'`）可见。Issue 创建/跟踪的预期功能在此版本代码库中不存在。

---

## 相关模块

- [Command System](../01-core-modules/command-system.md)
- [Session Storage Utils](../05-utils/session-storage.md)
- [Git Utils](../05-utils/git-utils.md)
