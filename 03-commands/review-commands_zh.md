# 审查命令

## 目的

记录与代码审查工作流相关的 CLI 命令：`/review`（ultrareview）、`/diff`、`/autofix-pr`、`/pr_comments` 和 `/good-claude`。这些命令支持远程代码审查会话、diff 检查、PR 评论获取和自动 PR 修复。

---

## /review（Ultrareview）

### 目的

在云中启动远程代码审查会话。Ultrareview 创建远程 agent 会话，分析针对基本分支或 pull request 的代码更改，运行多个并行 agent（"bug hunters"）以发现问题。结果通过任务通知异步到达。

### 位置

- `restored-src/src/commands/review/ultrareviewEnabled.ts` — 功能门
- `restored-src/src/commands/review/ultrareviewCommand.tsx` — 命令入口点
- `restored-src/src/commands/review/reviewRemote.ts` — 远程会话启动逻辑
- `restored-src/src/commands/review/UltrareviewOverageDialog.tsx` — 计费确认对话框

### 功能门

```typescript
// ultrareviewEnabled.ts
export function isUltrareviewEnabled(): boolean {
  const cfg = getFeatureValue_CACHED_MAY_BE_STALE('tengu_review_bughunter_config', null)
  return cfg?.enabled === true
}
```

命令由 `tengu_review_bughunter_config` GrowthBook 功能标志visibility门控。当 `enabled` 为 false 时，命令从 `getCommands()` 中完全过滤掉。

### 关键导出

#### 来自 `ultrareviewCommand.tsx`

| 导出 | 描述 |
|--------|-------------|
| `call(onDone, context, args)` | 命令入口点。检查超支门，必要时显示计费对话框，启动远程审查。 |
| `contentBlocksToString(blocks)` | 将 `ContentBlockParam[]` 转换为显示用的纯字符串。 |
| `launchAndDone(args, context, onDone, billingNote, signal)` | 启动远程审查并使用结果调用 `onDone`。处理中止信号。 |

#### 来自 `reviewRemote.ts`

| 导出 | 描述 |
|--------|-------------|
| `checkOverageGate()` | 确定计费状态：`proceed`、`not-enabled`、`low-balance` 或 `needs-confirm`。 |
| `confirmOverage()` | 设置会话级别的超支确认标志。 |
| `launchRemoteReview(args, context, billingNote)` | 创建远程审查会话。返回 `ContentBlockParam[]` 或 `null`。 |
| `OverageGate` | 用于计费门结果的区分联合类型。 |

#### 来自 `UltrareviewOverageDialog.tsx`

| 导出 | 描述 |
|--------|-------------|
| `UltrareviewOverageDialog({ onProceed, onCancel })` | 提示用户确认 Extra Usage 计费或取消的对话框。 |

### 实现细节

#### 计费门逻辑（`checkOverageGate`）

门确定用户是否可以启动 ultrareview 以及在什么计费条款下：

1. **团队/企业订阅者** — 始终进行（ultrareview 包含在计划中）。不检查免费审查配额。
2. **剩余免费审查** — 继续，计费注意指示这是第几个免费审查。
3. **免费审查已用尽** — 检查 Extra Usage 设置：
   - **未启用** — 返回 `not-enabled`，引导用户启用 Extra Usage。
   - **余额低**（< $10 可用）— 返回 `low-balance` 及可用金额。
   - **会话中首次超支** — 返回 `needs-confirm` 以显示计费对话框。
   - **已确认** — 继续，计费注意指示 Extra Usage 计费。

#### 远程审查启动（`launchRemoteReview`）

支持两种模式：

**PR 模式**（当 args 是数字 PR 编号时）：
- 通过 `detectCurrentRepositoryWithHost()` 检测当前 GitHub 仓库。
- 使用 `refs/pull/{N}/head` 作为分支引用。
- 使用 `BUGHUNTER_PR_NUMBER` 和 `BUGHUNTER_REPOSITORY` 环境变量远程传送到会话。
- 编排器使用 `--pr {N}` 标志运行。

**分支模式**（当 args 不是 PR 编号时）：
- 打包本地工作树（`useBundle: true`）。
- 针对默认分支计算 merge-base SHA。
- 验证：merge-base 存在，diff 非空，bundle 不太大。
- 使用 `BUGHUNTER_BASE_BRANCH` 设置为 merge-base SHA 进行传送。

#### 会话注册

成功传送后，会话作为 `RemoteAgentTask` 注册，类型为 `ultrareview`。轮询循环通过任务通知将结果传回本地会话。提供会话 URL 以进行跟踪。

---

## /diff

### 目的

在交互式对话框中查看未提交的更改和每轮次 diff。显示当前工作树相对于上次提交状态以及每个对话轮次引入的 diff 的当前状态。

### 位置

- `restored-src/src/commands/diff/index.ts` — 命令注册
- `restored-src/src/commands/diff/diff.tsx` — JSX 实现

### 命令注册

```typescript
{
  type: 'local-jsx',
  name: 'diff',
  description: 'View uncommitted changes and per-turn diffs',
  load: () => import('./diff.js'),
}
```

### 关键导出

| 导出 | 描述 |
|--------|-------------|
| `call(onDone, context)` | 入口点。延迟加载 `DiffDialog` 并使用当前对话消息渲染。 |

### 实现细节

该命令是延迟加载 `DiffDialog` 组件的薄包装器。它传递完整的消息历史（`context.messages`），组件计算并显示对话轮次之间以及当前文件状态的 diff。

---

## /autofix-pr

### 目的

自动修复在 pull request 中识别的问题。当前已实现为禁用的存根。

### 位置

- `restored-src/src/commands/autofix-pr/index.js`

### 命令注册

```javascript
{
  isEnabled: () => false,
  isHidden: true,
  name: 'stub',
}
```

---

## /pr_comments

### 目的

获取并显示 GitHub pull request 的评论。当市场是私有时，回退到使用 `gh` CLI 检索 PR 评论的 AI 生成提示。

### 位置

- `restored-src/src/commands/pr_comments/index.ts`

### 命令注册

```typescript
createMovedToPluginCommand({
  name: 'pr-comments',
  description: 'Get comments from a GitHub pull request',
  progressMessage: 'fetching PR comments',
  pluginName: 'pr-comments',
  pluginCommand: 'pr-comments',
  async getPromptWhileMarketplaceIsPrivate(args) { /* ... */ }
})
```

### 实现细节

此命令已**迁移到插件**（`pr-comments`）。`createMovedToPluginCommand` 包装器处理转换：

1. **插件可用** — 委托给 `pr-comments` 插件。
2. **插件不可用（私有市场）** — 生成详细提示，指导 AI 模型使用 GitHub CLI（`gh`）获取 PR 评论。

#### 回退提示步骤

当插件不可用时，生成的提示指示模型：

1. 运行 `gh pr view --json number,headRepository` 获取 PR 编号和仓库信息。
2. 运行 `gh api /repos/{owner}/{repo}/issues/{number}/comments` 获取 PR 级评论。
3. 运行 `gh api /repos/{owner}/{repo}/pulls/{number}/comments` 获取审查评论，包括 `body`、`diff_hunk`、`path`、`line` 字段。
4. 解析并格式化所有评论及文件/行上下文、diff hunk 和线程回复。
5. 仅返回格式化的评论，不带附加文本。

---

## /good-claude

### 目的

内部 Anthropic 命令。当前已实现为禁用的存根。

### 位置

- `restored-src/src/commands/good-claude/index.js`

### 命令注册

```javascript
{
  isEnabled: () => false,
  isHidden: true,
  name: 'stub',
}
```

---

## 数据流

```
/review <PR#>
  │
  ├── checkOverageGate()
  │     ├── Team/Enterprise? → proceed
  │     ├── Free reviews remaining? → proceed with note
  │     ├── Extra Usage enabled?
  │     │     ├── Balance >= $10?
  │     │     │     ├── Session confirmed? → proceed
  │     │     │     └── Not confirmed? → needs-confirm → show dialog
  │     │     └── Balance < $10? → low-balance error
  │     └── Extra Usage not enabled? → not-enabled error
  │
  ├── launchRemoteReview()
  │     ├── checkRemoteAgentEligibility()
  │     ├── PR mode or Branch mode?
  │     │     ├── PR mode: detect repo → teleport with refs/pull/N/head
  │     │     └── Branch mode: compute merge-base → validate diff → teleport with bundle
  │     ├── registerRemoteAgentTask({ type: 'ultrareview', ... })
  │     └── Return launch result with session URL
  │
  └── onDone(result) → task-notification delivers findings

/diff
  │
  └── DiffDialog(messages=context.messages)
        └── Computes and displays per-turn and uncommitted diffs

/autofix-pr → disabled stub
/pr_comments → delegates to pr-comments plugin (or gh CLI fallback)
/good-claude → disabled stub
```

## 相关模块

- [Command System](../01-core-modules/command-system.md)
- [Teleport Utils](../05-utils/teleport.md)
- [Git Utils](../05-utils/git-utils.md)
- [Remote Agent Task](../01-core-modules/task-system.md)
