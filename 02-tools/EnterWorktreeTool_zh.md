# EnterWorktreeTool

## 目的

EnterWorktreeTool 创建一个隔离的 git worktree（或通过钩子的 VCS 不可知 worktree）并将当前会话切换到其中。这为并行开发提供了一个干净的隔离工作目录，而不影响主分支或工作树。该工具管理 worktree 创建的完整生命周期，包括分支设置、CWD 切换、缓存失效和会话状态持久化。

## 位置

- `restored-src/src/tools/EnterWorktreeTool/EnterWorktreeTool.ts` — 主工具定义和创建逻辑（128 行）
- `restored-src/src/tools/EnterWorktreeTool/prompt.ts` — 工具提示（31 行）
- `restored-src/src/tools/EnterWorktreeTool/UI.tsx` — 终端 UI 渲染（20 行）
- `restored-src/src/tools/EnterWorktreeTool/constants.ts` — 工具名称常量（2 行）

## 关键导出

| 导出 | 描述 |
|--------|-------------|
| `EnterWorktreeTool` | 通过 `buildTool()` 构建的完整工具定义 |
| `Output` | 输出类型：`{ worktreePath, worktreeBranch?, message }` |
| `ENTER_WORKTREE_TOOL_NAME` | 常量字符串 `'EnterWorktree'` |

## 输入/输出 Schema

### 输入 Schema

```typescript
{
  name?: string  // 可选 worktree 名称
}
```

`name` 参数根据 worktree 命名规则进行验证：
- 每个 `/` 分隔的段只能包含字母、数字、点、下划线和破折号
- 总共最多 64 个字符
- 如果未提供，通过 `getPlanSlug()` 生成随机名称

### 输出 Schema

```typescript
{
  worktreePath: string,   // 新 worktree 目录的绝对路径
  worktreeBranch?: string, // 为此 worktree 创建的分支名称
  message: string,        // 人类可读的确认
}
```

## Worktree 创建

### 创建流程

```
1. 模型调用 EnterWorktree
   - 可选：{ name: "feature-auth" }

2. 会话验证
   - 检查：不在 worktree 会话中
   - getCurrentWorktreeSession() 必须返回 null
   - 如果已在 worktree 会话中则抛出

3. 仓库根解析
   - findCanonicalGitRoot(getCwd()) — 找到主 git 仓库根
   - 如果不同于当前 CWD，chdir 到仓库根
   - 这确保从嵌套目录工作时 worktree 创建有效

4. WORKTREE 名称解析
   - slug = input.name ?? getPlanSlug()
   - getPlanSlug() 如果没有提供则生成随机名称

5. WORKTREE 创建
   - createWorktreeForSession(sessionId, slug)
   - 在 .claude/worktrees/ 内创建 git worktree
   - 基于 HEAD 创建新分支
   - 返回 { worktreePath, worktreeBranch, originalCwd, ... }

6. 会话状态切换
   - process.chdir(worktreePath) — 更改进程工作目录
   - setCwd(worktreePath) — 更新跟踪的 CWD
   - setOriginalCwd(worktreePath) — 将原始 CWD 设置为 worktree 路径
   - saveWorktreeState(worktreeSession) — 持久化 worktree 会话状态

7. 缓存失效
   - clearSystemPromptSections() — 强制 env_info_simple 重新计算
   - clearMemoryFileCaches() — 清除依赖 CWD 的记忆化缓存
   - getPlansDirectory.cache.clear() — 清除计划目录缓存

8. 分析
   - logEvent('tengu_worktree_created', { mid_session: true })

9. 返回结果
   - worktreePath、worktreeBranch、确认消息
```

## 分支管理

### 分支创建

worktree 基于当前 HEAD 创建新分支：

```typescript
const worktreeSession = await createWorktreeForSession(getSessionId(), slug)
```

`createWorktreeForSession` 函数（在 `utils/worktree.ts` 中）：
1. 从 HEAD 创建新分支，具有唯一名称
2. 在 `.claude/worktrees/<slug>/` 设置 git worktree
3. 可选地为终端隔离创建 tmux 会话
4. 记录原始 head 提交以在退出时进行更改跟踪

### 分支命名

分支名称源自 worktree slug：
- 用户提供的名称：`feature-auth` → 分支 `feature-auth`
- 自动生成的名称：`plan-abc123` → 分支 `plan-abc123`

分支从 HEAD 新建，因此它从没有唯一提交开始。

## 上下文切换

### 目录状态更改

工具执行到 worktree 的完整上下文切换：

| 状态 | 之前 | 之后 |
|-------|--------|-------|
| `process.cwd()` | 原始目录 | Worktree 路径 |
| `getCwd()` | 原始目录 | Worktree 路径 |
| `getOriginalCwd()` | 原始目录 | Worktree 路径（故意的） |
| `getCurrentWorktreeSession()` | null | Worktree 会话对象 |

**注意：** `setOriginalCwd` 被故意设置为 worktree 路径。这是因为 `getProjectRoot` 使用 `getOriginalCwd` 作为回退，在 worktree 会话期间项目根应解析到 worktree 位置。见 `state.ts` 注释了解原因。

### 缓存失效

清除三个缓存层以确保会话反映 worktree 上下文：

| 缓存 | 用途 |
|-------|---------|
| `clearSystemPromptSections()` | 强制 `env_info_simple` 使用 worktree 上下文重新计算 |
| `clearMemoryFileCaches()` | 清除依赖 CWD 的记忆化文件查找 |
| `getPlansDirectory.cache.clear()` | 清除计划目录解析缓存 |

### 会话状态持久化

通过 `saveWorktreeState(worktreeSession)` 保存 worktree 会话状态。这持久化：
- `worktreePath` — worktree 目录
- `worktreeBranch` — 分支名称
- `originalCwd` — 进入之前的会话位置
- `originalHeadCommit` — 创建时的 HEAD 提交（用于更改跟踪）
- `tmuxSessionName` — 关联的 tmux 会话（如果已创建）

此状态被 `ExitWorktreeTool` 用于正确恢复会话。

## VCS 不可知支持

### 基于钩子的 Worktree

在 git 仓库外部，工具委托给 `settings.json` 中配置的 `WorktreeCreate`/`WorktreeRemove` 钩子：

```json
{
  "hooks": {
    "WorktreeCreate": "custom-script.sh",
    "WorktreeRemove": "cleanup-script.sh"
  }
}
```

这允许非 git 版本控制系统或自定义工作流的类 worktree 隔离。`createWorktreeForSession` 函数检测是否在 git 仓库中并相应路由。

## UI 渲染

### 工具使用消息

```tsx
'Creating worktree…'
```

在 worktree 创建时显示简单的加载消息。

### 工具结果消息

```tsx
<Box flexDirection="column">
  <Text>
    Switched to worktree on branch <Text bold>{output.worktreeBranch}</Text>
  </Text>
  <Text dimColor>{output.worktreePath}</Text>
</Box>
```

以粗体显示分支名称，以暗淡文本显示 worktree 路径。

## 工具属性

| 属性 | 值 | 理由 |
|----------|-------|-----------|
| `shouldDefer` | `true` | 执行前需要用户批准 |
| `isDestructive` | 未定义 | Worktree 创建是非破坏性的 |
| `userFacingName` | `'Creating worktree'` | 在进度 UI 中显示 |
| `toAutoClassifierInput` | `input.name ?? ''` | 使用 worktree 名称进行自动分类 |

## 提示指导

### 使用时机

仅当用户明确提到"worktree"时：
- "start a worktree"
- "work in a worktree"
- "create a worktree"
- "use a worktree"

### 不使用时机

- 用户要求创建分支——使用 git 命令
- 用户要求切换分支——使用 git checkout/switch
- 用户要求修复 bug 或处理功能——使用正常 git 工作流
- 除非明确提到"worktree"，否则不要使用

### 要求

- 必须在 git 仓库中，或已配置 WorktreeCreate/WorktreeRemove 钩子
- 不能已经在 worktree 中

### 行为摘要

- 在 git repo 中：在 `.claude/worktrees/` 中从 HEAD 新建分支创建 worktree
- 在 git repo 外：委托给配置的钩子
- 将会话 CWD 切换到新 worktree
- 使用 ExitWorktree 离开（保留或删除）
- 会话退出时，提示用户保留或删除 worktree

## 集成点

### `bootstrap/state.js`

- `getSessionId()` — 返回当前会话标识符
- `setOriginalCwd(path)` — 设置会话的原始工作目录

### `utils/worktree.js`

- `createWorktreeForSession(sessionId, slug)` — 创建 worktree 和分支
- `getCurrentWorktreeSession()` — 返回当前 worktree 会话或 null
- `validateWorktreeSlug(slug)` — 验证 worktree 名称格式

### `utils/git.js`

- `findCanonicalGitRoot(cwd)` — 解析到规范 git 仓库的根（处理 worktree）

### `utils/Shell.js`

- `setCwd(path)` — 更新跟踪的当前工作目录

### `utils/sessionStorage.js`

- `saveWorktreeState(session)` — 持久化 worktree 会话状态以供稍后恢复

### `utils/claudemd.js`

- `clearMemoryFileCaches()` — 清除记忆化 CLAUDE.md 文件缓存

### `utils/plans.js`

- `getPlanSlug()` — 生成默认 worktree 名称的随机计划 slug
- `getPlansDirectory()` — 计划目录解析器（进入时清除缓存）

### `constants/systemPromptSections.js`

- `clearSystemPromptSections()` — 清除缓存的系统提示部分

### `services/analytics/index.js`

- `logEvent('tengu_worktree_created', ...)` — 记录 worktree 创建

## 数据流

```
Model calls EnterWorktree { name? }
    |
    v
┌─────────────────────────────────────────┐
│ VALIDATION                              │
│ - Check: not already in worktree session│
│ - Validate name format if provided      │
└─────────────────────────────────────────┘
    |
    v
┌─────────────────────────────────────────┐
│ REPO RESOLUTION                         │
│ - Find canonical git root               │
│ - chdir to root if needed               │
└─────────────────────────────────────────┘
    |
    v
┌─────────────────────────────────────────┐
│ WORKTREE CREATION                       │
│                                         │
│ createWorktreeForSession(sessionId, slug)│
│   - Create branch from HEAD             │
│   - Create git worktree directory       │
│   - Record original head commit         │
│   - Optionally create tmux session      │
│                                         │
│ Returns: {                              │
│   worktreePath, worktreeBranch,         │
│   originalCwd, originalHeadCommit,      │
│   tmuxSessionName                       │
│ }                                       │
└─────────────────────────────────────────┘
    |
    v
┌─────────────────────────────────────────┐
│ CONTEXT SWITCH                          │
│                                         │
│ - process.chdir(worktreePath)           │
│ - setCwd(worktreePath)                  │
│ - setOriginalCwd(worktreePath)          │
│ - saveWorktreeState(session)            │
│ - Clear system prompt caches            │
│ - Clear memory file caches              │
│ - Clear plans directory cache           │
└─────────────────────────────────────────┘
    |
    v
Session is now in worktree context
    |
    v
Result: { worktreePath, worktreeBranch, message }
```
