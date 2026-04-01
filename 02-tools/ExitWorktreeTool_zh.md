# ExitWorktreeTool

## 目的

ExitWorktreeTool 退出由 EnterWorktreeTool 创建的 worktree 会话，并将会话恢复到其原始工作目录。它提供两个操作 — 保留或删除 worktree — 以及安全检查以防止意外丢失未提交的工作。该工具处理完整会话状态恢复，包括 CWD、项目根目录、钩子配置和 CWD 依赖缓存。

## 位置

- `restored-src/src/tools/ExitWorktreeTool/ExitWorktreeTool.ts` — 主工具定义和清理逻辑（330 行）
- `restored-src/src/tools/ExitWorktreeTool/prompt.ts` — 工具提示（33 行）
- `restored-src/src/tools/ExitWorktreeTool/UI.tsx` — 终端 UI 渲染（25 行）
- `restored-src/src/tools/ExitWorktreeTool/constants.ts` — 工具名称常量（2 行）

## 关键导出

| 导出 | 描述 |
|--------|-------------|
| `ExitWorktreeTool` | 通过 `buildTool()` 构建的完整工具定义 |
| `Output` | 输出类型：`{ action, originalCwd, worktreePath, worktreeBranch?, tmuxSessionName?, discardedFiles?, discardedCommits?, message }` |
| `ENTER_WORKTREE_TOOL_NAME` | 常量字符串 `'ExitWorktree'` |

## 输入/输出 Schema

### 输入 Schema

```typescript
{
  action: 'keep' | 'remove',          // 必需：保留或删除 worktree
  discard_changes?: boolean,          // 删除带未提交工作时必需为 true
}
```

`action` 参数决定 worktree 的命运：
- `"keep"` — 将 worktree 目录和分支保留在磁盘上
- `"remove"` — 删除 worktree 目录及其分支

`discard_changes` 参数是安全门控：
- 当 `action: "remove"` 且 worktree 有未提交的文件或不在原始分支上的提交时，必须为 `true`
- 如果省略且存在更改，工具拒绝并列出更改

### 输出 Schema

```typescript
{
  action: 'keep' | 'remove',          // 所采取的操作
  originalCwd: string,                // 会话返回的目录
  worktreePath: string,               // 退出的 worktree 的路径
  worktreeBranch?: string,            // worktree 的分支名称
  tmuxSessionName?: string,           // 关联的 tmux 会话（如果有）
  discardedFiles?: number,            // 丢弃的未提交文件计数
  discardedCommits?: number,          // 丢弃的提交计数
  message: string,                    // 人类可读的确认
}
```

## Worktree 清理

### 清理流程

```
1. 模型调用 ExitWorktree
   - { action: 'keep' | 'remove', discard_changes?: boolean }

2. 验证（validateInput）
   - 检查：getCurrentWorktreeSession() 不为 null
   - 如果为 null：拒绝作为空操作（没有活动的 EnterWorktree 会话）
   - 如果 action === 'remove' && !discard_changes：
     a. 计算 worktree 更改（文件 + 提交）
     b. 如果存在更改：拒绝并显示详情
     c. 如果计数失败（null）：拒绝（失败关闭）

3. 更改计数（在执行时）
   - 重新计数更改（状态可能在验证后已更改）
   - 使用 git status --porcelain 获取文件计数
   - 使用 git rev-list --count 获取提交计数
   - 如果 git 命令失败则回退到 0/0

4. KEEP 操作
   - keepWorktree() — 在磁盘上保留 worktree
   - restoreSessionToOriginalCwd() — 恢复会话状态
   - 记录分析：tengu_worktree_kept
   - 如果适用则返回 tmux 重附加指令

5. REMOVE 操作
   - killTmuxSession() — 终止关联的 tmux 会话
   - cleanupWorktree() — 删除 worktree 目录和分支
   - restoreSessionToOriginalCwd() — 恢复会话状态
   - 记录分析：tengu_worktree_removed
   - 返回丢弃摘要
```

## 更改检测

### countWorktreeChanges 函数

计算相对于原始状态的 worktree 中未提交的文件和提交：

```typescript
async function countWorktreeChanges(
  worktreePath: string,
  originalHeadCommit: string | undefined,
): Promise<ChangeSummary | null>
```

**文件计数：**
```bash
git -C <worktreePath> status --porcelain
```
计算 porcelain 输出中的非空行数。

**提交计数：**
```bash
git -C <worktreePath> rev-list --count <originalHeadCommit>..HEAD
```
计算 worktree 分支独有的提交。

### 失败关闭设计

当状态无法可靠确定时，函数返回 `null`（而非 `0/0`）：

| 条件 | 返回 | 理由 |
|-----------|---------|-----------|
| `git status` 退出非零 | `null` | 锁文件、损坏的索引、错误的引用 |
| `git rev-list` 退出非零 | `null` | 错误引用或损坏的仓库 |
| `originalHeadCommit` 未定义但 git status 成功 | `null` | 基于钩子的 worktree — 没有可比较的基线提交 |

返回 `null` 而非 `0/0` 防止意外数据丢失。当更改计数未知时，安静的 `0/0` 会让 `cleanupWorktree` 销毁真实工作。

## 会话状态恢复

### restoreSessionToOriginalCwd 函数

恢复所有会话状态以反映原始目录：

```typescript
function restoreSessionToOriginalCwd(
  originalCwd: string,
  projectRootIsWorktree: boolean,
): void
```

**状态更改：**

| 状态 | 操作 |
|-------|--------|
| `setCwd(originalCwd)` | 恢复工作目录 |
| `setOriginalCwd(originalCwd)` | 重置原始 CWD（EnterWorktree 将其设置为 worktree 路径） |
| `setProjectRoot(originalCwd)` | 仅当 `projectRootIsWorktree` 为 true |
| `updateHooksConfigSnapshot()` | 从原始目录重新读取钩子（仅当 projectRoot 是 worktree 时） |
| `saveWorktreeState(null)` | 清除 worktree 会话状态 |
| `clearSystemPromptSections()` | 强制系统提示重新计算 |
| `clearMemoryFileCaches()` | 清除 CWD 依赖的记忆化缓存 |
| `getPlansDirectory.cache.clear()` | 清除计划目录缓存 |

### 项目根检测

工具检测项目根是否设置为 worktree：

```typescript
const projectRootIsWorktree = getProjectRoot() === getOriginalCwd()
```

当会话使用 `--worktree` 启动时为 true（setup.ts 将两者都设置为 worktree 路径）。对于会话中调用 EnterWorktreeTool，project root 不变，所以这是 false，跳过 project root 恢复。

这保留了"稳定项目标识"契约 — project root 不应基于用户在进入 worktree 之前恰好 cd 到哪里而改变。

## 分支管理

### Keep 操作

当 `action: 'keep'` 时：
- worktree 目录保留在磁盘上
- 分支保留在 git 仓库中
- tmux 会话（如果有）继续运行
- 返回 tmux 会话名称以便用户重新附加：`tmux attach -t <name>`

### Remove 操作

当 `action: 'remove'` 时：
- tmux 会话（如果有）通过 `killTmuxSession()` 终止
- worktree 目录通过 `cleanupWorktree()` 删除
- 分支从 git 仓库中移除
- 所有未提交的更改永久丢失

### 更改拒绝

当删除带未提交工作且 `discard_changes` 不为 `true` 时：

```
Worktree has 3 uncommitted files and 2 commits on the worktree branch.
Removing will discard this work permanently. Confirm with the user, then
re-invoke with discard_changes: true — or use action: "keep" to preserve
the worktree.
```

工具列出确切将被丢失的内容，实现知情的用户决策。

## 上下文切换

### 目录恢复

工具反转 EnterWorktreeTool 所做的所有上下文更改：

| 状态 | 退出前 | 退出后 |
|-------|-------------|------------|
| `process.cwd()` | Worktree 路径 | 原始 CWD |
| `getCwd()` | Worktree 路径 | 原始 CWD |
| `getOriginalCwd()` | Worktree 路径 | 原始 CWD |
| `getProjectRoot()` | Worktree 路径（如果是 --worktree 启动） | 原始 CWD（如果是 --worktree 启动） |
| `getCurrentWorktreeSession()` | 会话对象 | null |

### 缓存失效

清除入口时清除的相同缓存以确保会话反映原始目录：

| 缓存 | 用途 |
|-------|---------|
| `clearSystemPromptSections()` | 强制 `env_info_simple` 使用原始上下文重新计算 |
| `clearMemoryFileCaches()` | 清除原始 CWD 的记忆化文件查找 |
| `getPlansDirectory.cache.clear()` | 清除计划目录解析缓存 |

## UI 渲染

### 工具使用消息

```tsx
'Exiting worktree…'
```

在退出 worktree 时显示简单的加载消息。

### 工具结果消息

```tsx
const actionLabel = output.action === 'keep' ? 'Kept worktree' : 'Removed worktree';

<Box flexDirection="column">
  <Text>
    {actionLabel}
    {output.worktreeBranch ? (
      <>
        {' '}
        (branch <Text bold>{output.worktreeBranch}</Text>)
      </>
    ) : null}
  </Text>
  <Text dimColor>Returned to {output.originalCwd}</Text>
</Box>
```

显示所采取的操作、分支名称（如果有）加粗，以及暗淡文本显示返回的目录。

## 工具属性

| 属性 | 值 | 理由 |
|----------|-------|-----------|
| `shouldDefer` | `true` | 执行前需要用户批准 |
| `isDestructive` | `input.action === 'remove'` | 仅在删除时具有破坏性 |
| `userFacingName` | `'Exiting worktree'` | 在进度 UI 中显示 |
| `toAutoClassifierInput` | `input.action` | 使用操作进行自动分类 |

## 验证和错误处理

### 输入验证

| 条件 | 结果 | 错误代码 |
|-----------|--------|------------|
| 没有活动的 worktree 会话 | 拒绝作为空操作 | 1 |
| 无法验证 worktree 状态（git 失败） | 拒绝（失败关闭） | 3 |
| 未指定 discard_changes 的未提交更改 | 拒绝并列出更改 | 2 |

### 空操作保证

当在 EnterWorktree 会话之外调用时，工具是**空操作**：
- 报告没有活动的 worktree 会话
- 不采取文件系统操作
- 状态不变

这是安全设计 — 工具不会触及：
- 使用 `git worktree add` 手动创建的 worktree
- 之前会话中的 worktree
- 如果从未调用 EnterWorktree 则是任何目录

### 竞态条件防御

`call()` 方法即使 `validateInput()` 已检查也重新检查 `getCurrentWorktreeSession()`：

```typescript
const session = getCurrentWorktreeSession()
if (!session) {
  throw new Error('Not in a worktree session')
}
```

这防止验证和执行之间的竞态，因为 `currentWorktreeSession` 是模块级可变状态。

## 提示指导

### 范围

此工具**仅**操作本会话中 EnterWorktree 创建的 worktree。它不会触及：
- 使用 `git worktree add` 手动创建的 worktree
- 之前会话中的 worktree
- 如果从未调用 EnterWorktree 则是当前目录

### 使用时机

- 用户明确要求"退出 worktree"、"离开 worktree"、"返回"
- 不要主动调用 — 仅当用户要求时

### 参数

- `action`（必需）：`"keep"` 或 `"remove"`
  - `"keep"` — 为以后使用保留 worktree，或如果有要保存的更改
  - `"remove"` — 工作完成或放弃时的干净删除
- `discard_changes`（可选）：仅对 `action: "remove"` 有意义
  - 如果 worktree 有未提交的文件或提交，没有此设置为 `true` 时工具**拒绝**
  - 如果工具返回列出更改的错误，在重新调用之前与用户确认

### 行为摘要

- 将会话 CWD 恢复到 EnterWorktree 前的位置
- 清除 CWD 依赖缓存
- tmux 会话：在 `remove` 时终止，在 `keep` 时保持运行
- 退出后，可以为新的 worktree 再次调用 EnterWorktree

## 集成点

### `bootstrap/state.js`

- `getOriginalCwd()` — 返回原始工作目录
- `getProjectRoot()` — 返回项目根目录
- `setOriginalCwd(path)` — 恢复原始 CWD
- `setProjectRoot(path)` — 有条件地恢复项目根

### `utils/worktree.js`

- `getCurrentWorktreeSession()` — 返回当前 worktree 会话或 null
- `keepWorktree()` — 在磁盘上保留 worktree，清除会话状态
- `cleanupWorktree()` — 删除 worktree 目录和分支
- `killTmuxSession(name)` — 终止关联的 tmux 会话

### `utils/Shell.js`

- `setCwd(path)` — 更新跟踪的当前工作目录

### `utils/sessionStorage.js`

- `saveWorktreeState(null)` — 清除持久化的 worktree 会话状态

### `utils/execFileNoThrow.js`

- `execFileNoThrow('git', args)` — 运行 git 命令，在非零退出时不抛出（用于更改计数）

### `utils/claudemd.js`

- `clearMemoryFileCaches()` — 清除记忆化的 CLAUDE.md 文件缓存

### `utils/plans.js`

- `getPlansDirectory()` — 计划目录解析器（退出时清除缓存）

### `utils/hooks/hooksConfigSnapshot.js`

- `updateHooksConfigSnapshot()` — 从恢复的目录重新读取钩子配置

### `constants/systemPromptSections.js`

- `clearSystemPromptSections()` — 清除缓存的系统提示部分

### `utils/array.js`

- `count(array, predicate)` — 计算匹配谓词的元素（用于文件计数）

### `services/analytics/index.js`

- `logEvent('tengu_worktree_kept', ...)` — 记录 worktree keep 操作
- `logEvent('tengu_worktree_removed', ...)` — 记录 worktree remove 操作

## 数据流

```
Model calls ExitWorktree { action, discard_changes? }
    │
    ▼
┌─────────────────────────────────────────┐
│ 验证                              │
│                                         │
│ - 检查：活动 worktree 会话存在 │
│ - 如果删除：计数更改            │
│   - git status --porcelain（文件）      │
│   - git rev-list --count（提交）      │
│ - 如果存在更改但没有              │
│   discard_changes：拒绝               │
│ - 如果计数失败：拒绝（失败关闭）  │
└─────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────┐
│ 执行时重新计数              │
│ （状态可能在验证后更改） │
│                                         │
│ - 与验证相同的 git 命令       │
│ - 失败时回退到 0/0          │
│ - 仅用于分析 + 消息传递   │
└─────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────┐
│ 操作：KEEP 或 REMOVE                  │
│                                         │
│ KEEP：                                   │
│   - keepWorktree()                      │
│   - tmux 会话保持运行           │
│                                         │
│ REMOVE：                                 │
│   - 如果适用则 killTmuxSession()     │
│   - cleanupWorktree()                   │
│   - 分支和目录删除        │
└─────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────┐
│ 会话恢复                         │
│                                         │
│ - setCwd(originalCwd)                   │
│ - setOriginalCwd(originalCwd)           │
│ - 如果需要则 setProjectRoot(originalCwd) │
│ - 如果需要则 updateHooksConfigSnapshot() │
│ - saveWorktreeState(null)               │
│ - 清除所有 CWD 依赖缓存        │
└─────────────────────────────────────────┘
    │
    ▼
Session is back in original directory
    │
    ▼
Result: { action, originalCwd, worktreePath, discardedFiles?, discardedCommits?, message }
```
