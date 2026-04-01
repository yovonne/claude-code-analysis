# 命令系统

## 目的

命令系统是 Claude Code CLI 处理斜杠命令（如 `/help`、`/config`、`/compact`）的核心机制。它为三种不同的命令类型提供了统一的注册、分发和执行架构：**本地**命令（返回文本）、**本地-jsx** 命令（渲染交互式 Ink UI）和**提示**命令（展开为模型可见的内容块/技能）。命令从多个来源加载——内置定义、插件目录、技能目录、捆绑技能、MCP 服务器和工作流脚本——并根据认证状态、功能标志、执行模式（交互式 vs 非交互式）以及远程/桥接上下文动态过滤。

## 位置

- `restored-src/src/commands.ts` — 命令注册、聚合、可用性过滤和远程模式门控
- `restored-src/src/commands/` — 单独的命令实现（~70+ 命令作为目录或文件）
- `restored-src/src/types/command.ts` — 命令类型定义和接口
- `restored-src/src/utils/slashCommandParsing.ts` — 斜杠命令字符串解析
- `restored-src/src/utils/processUserInput/processSlashCommand.tsx` — 斜杠命令执行管道
- `restored-src/src/utils/processUserInput/processUserInput.ts` — 顶层用户输入路由（斜杠 vs 文本 vs bash）

## 关键导出

### 从 commands.ts 导出

| 导出 | 描述 |
|--------|-------------|
| `getCommands(cwd)` | 异步函数，返回当前用户所有可用命令。按 `cwd` 缓存。应用可用性、`isEnabled()` 和去重过滤器。 |
| `filterCommandsForRemoteMode(commands)` | 过滤命令列表，仅保留 `REMOTE_SAFE_COMMANDS` 中的命令。在 `--remote` 模式中使用，以防止本地命令出现。 |
| `findCommand(commandName, commands)` | 按名称或别名查找命令。如果未找到则返回 `undefined`。 |
| `getCommand(commandName, commands)` | 类似 `findCommand`，但如果未找到则抛出 `ReferenceError`，并列出可用命令。 |
| `hasCommand(commandName, commands)` | 如果命令存在（按名称或别名）则返回 `true`。 |
| `builtInCommandNames()` | 所有内置命令名称和别名的缓存 `Set<string>`。 |
| `getSkillToolCommands(cwd)` | 返回 SkillTool 可见的提示类型命令（模型可调用的技能）。过滤掉内置命令和没有描述的命令。 |
| `getSlashCommandToolSkills(cwd)` | 仅返回技能命令（从 `skills/`、`plugin`、`bundled` 加载，或带有 `disableModelInvocation`）。 |
| `getMcpSkillCommands(mcpCommands)` | 当 `MCP_SKILLS` 功能启用时，返回 MCP 提供的技能命令。 |
| `isBridgeSafeCommand(cmd)` | 确定命令在远程控制桥接上接收时是否可以安全执行。 |
| `formatDescriptionWithSource(cmd)` | 格式化命令描述并添加来源注释（如 `(plugin)`、`(workflow)`、`(bundled)`）。 |
| `clearCommandsCache()` / `clearCommandMemoizationCaches()` | 使缓存的命令加载失效（用于插件/技能更改后）。 |
| `meetsAvailabilityRequirement(cmd)` | 检查命令的 `availability` 约束是否匹配当前认证状态（`claude-ai`、`console`）。 |
| `INTERNAL_ONLY_COMMANDS` | 仅对 Anthropic 内部用户可见的命令数组（`USER_TYPE === 'ant'`）。 |
| `REMOTE_SAFE_COMMANDS` | 对远程模式安全的 `Set<Command>`（仅影响本地 TUI 状态）。 |
| `BRIDGE_SAFE_COMMANDS` | 可通过远程控制桥接安全执行的 `local` 类型命令的 `Set<Command>`。 |

### 从 types/command.ts 导出

| 导出 | 描述 |
|--------|-------------|
| `Command` | 联合类型：`CommandBase & (PromptCommand | LocalCommand | LocalJSXCommand)` |
| `CommandBase` | 共享属性：`name`、`description`、`aliases`、`isEnabled`、`isHidden`、`availability`、`loadedFrom` 等。 |
| `PromptCommand` | 展开为发送给模型的 `ContentBlockParam[]` 的命令。有 `getPromptForCommand()`。 |
| `LocalCommand` | 返回 `LocalCommandResult`（`text`、`compact` 或 `skip`）的命令。 |
| `LocalJSXCommand` | 渲染 Ink JSX UI 的命令。有 `load()` 返回 `LocalJSXCommandModule`。 |
| `LocalCommandResult` | 可辨识联合：`{ type: 'text', value }`、`{ type: 'compact', compactionResult }`、`{ type: 'skip' }` |
| `LocalJSXCommandContext` | 传递给 JSX 命令的上下文：`canUseTool`、`setMessages`、`options`、`resume` 等。 |
| `LocalJSXCommandOnDone` | JSX 命令完成时的回调。控制显示模式、查询行为和后续输入。 |
| `CommandAvailability` | `'claude-ai' | 'console'` — 认证/提供商门控。 |
| `getCommandName(cmd)` | 解析用户可见名称，回退到 `cmd.name`。 |
| `isCommandEnabled(cmd)` | 解析 `isEnabled()`，默认为 `true`。 |

## 命令接口

`Command` 类型是基于 `type` 的可辨识联合：

```typescript
type Command = CommandBase & (
  | PromptCommand      // type: 'prompt' — 展开为模型的内容块
  | LocalCommand       // type: 'local' — 返回 text/compact/skip 结果
  | LocalJSXCommand    // type: 'local-jsx' — 渲染 Ink UI 组件
)
```

**CommandBase**（共享属性）：

```typescript
type CommandBase = {
  name: string                          // 命令标识符（如 'compact'）
  description: string                   // 在帮助/类型提示中显示
  aliases?: string[]                    // 替代名称（如 config → ['settings']）
  isEnabled?: () => boolean             // 动态启用检查（默认为 true）
  isHidden?: boolean                    // 从类型提示/帮助中隐藏（默认为 false）
  availability?: CommandAvailability[]  // 认证门控：'claude-ai' | 'console'
  argumentHint?: string                 // 参数的灰色提示文本（如 '[model]'）
  whenToUse?: string                    // 详细的使用场景（技能）
  version?: string
  loadedFrom?: 'commands_DEPRECATED' | 'skills' | 'plugin' | 'managed' | 'bundled' | 'mcp'
  kind?: 'workflow'                     // 在自动完成中标记
  immediate?: boolean                   // 无需等待停止点即可执行
  isSensitive?: boolean                 // 参数从历史记录中编辑
  disableModelInvocation?: boolean      // 阻止模型触发的调用
  userInvocable?: boolean               // 允许用户输入 /command（默认为 true）
  userFacingName?: () => string         // 覆盖显示名称
  hasUserSpecifiedDescription?: boolean
}
```

**PromptCommand**（技能/模型可调用命令）：

```typescript
type PromptCommand = {
  type: 'prompt'
  progressMessage: string               // 执行期间显示的"loading..."文本
  contentLength: number                 // 字符数用于 token 估算
  source: SettingSource | 'builtin' | 'mcp' | 'plugin' | 'bundled'
  argNames?: string[]
  allowedTools?: string[]               // 执行期间授予的额外工具
  model?: string                        // 覆盖此命令的模型
  context?: 'inline' | 'fork'           // inline = 当前对话，fork = 子 agent
  agent?: string                        // 分叉时的 agent 类型
  effort?: EffortValue
  paths?: string[]                      // Glob 模式 — 仅在匹配文件触及后可见
  skillRoot?: string                    // 技能钩子的基础目录（CLAUDE_PLUGIN_ROOT）
  hooks?: HooksSettings                 // 钩子注册
  pluginInfo?: { pluginManifest, repository }
  disableNonInteractive?: boolean
  getPromptForCommand(args, context): Promise<ContentBlockParam[]>
}
```

**LocalCommand**（纯文本/compact 结果）：

```typescript
type LocalCommand = {
  type: 'local'
  supportsNonInteractive: boolean
  load: () => Promise<{ call: (args, context) => Promise<LocalCommandResult> }>
}
```

**LocalJSXCommand**（交互式 Ink UI）：

```typescript
type LocalJSXCommand = {
  type: 'local-jsx'
  load: () => Promise<{ call: (onDone, context, args) => Promise<React.ReactNode> }>
}
```

## 命令注册

命令从**六个来源**注册，在 `commands.ts` 中组装：

### 1. 内置命令（静态导入）

~70 个命令在模块加载时直接导入，收集到缓存的 `COMMANDS()` 数组中：

```typescript
const COMMANDS = memoize((): Command[] => [
  addDir, advisor, agents, branch, btw, chrome, clear, color, compact,
  config, copy, desktop, context, cost, diff, doctor, effort, exit,
  fast, files, help, ide, init, mcp, memory, model, /* ... */
])
```

### 2. 功能门控命令（条件 require）

通过 `require()` 在功能标志激活时加载的命令：

```typescript
const proactive = feature('PROACTIVE') || feature('KAIROS')
  ? require('./commands/proactive.js').default : null
const bridge = feature('BRIDGE_MODE')
  ? require('./commands/bridge/index.js').default : null
const voiceCommand = feature('VOICE_MODE')
  ? require('./commands/voice/index.js').default : null
// ... spread into COMMANDS(): ...(bridge ? [bridge] : [])
```

### 3. 仅内部命令

仅当 `USER_TYPE === 'ant'` 且 `!IS_DEMO` 时可见的命令：

```typescript
export const INTERNAL_ONLY_COMMANDS = [
  backfillSessions, breakCache, bughunter, commit, commitPushPr,
  ctx_viz, goodClaude, issue, /* ... */
].filter(Boolean)
// Added to COMMANDS(): ...(process.env.USER_TYPE === 'ant' && !process.env.IS_DEMO ? INTERNAL_ONLY_COMMANDS : [])
```

### 4. 技能目录命令（异步、磁盘 I/O）

从项目中的 `skills/` 目录加载的命令：

```typescript
const { skillDirCommands } = await getSkillDirCommands(cwd)
```

### 5. 插件命令（异步、磁盘 I/O）

从已安装插件加载的命令：

```typescript
const pluginCommands = await getPluginCommands()
const pluginSkills = await getPluginSkills()
const builtinPluginSkills = getBuiltinPluginSkillCommands()
```

### 6. 工作流命令（功能门控）

从工作流脚本生成命令：

```typescript
const workflowCommands = getWorkflowCommands
  ? await getWorkflowCommands(cwd) : []
```

### 7. 捆绑技能（同步）

与 Claude Code 捆绑的技能，在启动时注册：

```typescript
const bundledSkills = getBundledSkills()
```

### 组装顺序

`loadAllCommands()` 按此顺序合并所有来源：

```typescript
return [
  ...bundledSkills,
  ...builtinPluginSkills,
  ...skillDirCommands,
  ...workflowCommands,
  ...pluginCommands,
  ...pluginSkills,
  ...COMMANDS(),  // 内置命令最后
]
```

## 命令分发机制

命令分发通过三层流动：

### 第 1 层：`processUserInput()` → `processUserInputBase()`

所有用户输入的入口点（`processUserInput.ts`）。它确定输入类型：

```
以 '/' 开头的输入      →  processSlashCommand()
模式为 'bash' 的输入   →  processBashCommand()
所有其他输入          →  processTextPrompt()
```

在分发之前，它处理：
- 桥接安全命令覆盖（移动/网络客户端）
- Ultraplan 关键字检测（重写为 `/ultraplan`）
- 图像处理和附件提取
- 钩子执行（`UserPromptSubmit` 钩子）

### 第 2 层：`processSlashCommand()`

位于 `processSlashCommand.tsx`。主要分发管道：

```
1. parseSlashCommand(inputString)          → 提取 commandName + args
2. hasCommand(commandName, commands)       → 验证命令存在
3. getMessagesForSlashCommand(...)         → 基于命令类型执行
4. 记录遥测 (tengu_input_command)
5. 返回 { messages, shouldQuery, ... }
```

### 第 3 层：`getMessagesForSlashCommand()` — 基于类型的路由

切换 `command.type`：

| 类型 | 执行路径 |
|------|---------------|
| `local-jsx` | 通过 `command.load()` 加载模块，调用 `mod.call(onDone, context, args)`。返回一个 Promise，在 `onDone()` 调用时或 JSX 渲染时解析。 |
| `local` | 通过 `command.load()` 加载模块，调用 `mod.call(args, context)`。返回 `LocalCommandResult`（`text`、`compact` 或 `skip`）。 |
| `prompt` | 如果 `command.context === 'fork'`，运行 `executeForkedSlashCommand()`（子 agent）。否则调用 `getMessagesForPromptSlashCommand()`，调用 `command.getPromptForCommand(args, context)` 获取内容块。 |

## 斜杠命令解析

解析由 `slashCommandParsing.ts` 中的 `parseSlashCommand()` 处理：

```typescript
export function parseSlashCommand(input: string): ParsedSlashCommand | null {
  // 1. 修剪并验证以 '/' 开头
  // 2. 按空格分割
  // 3. 检查 MCP 命令（第二个词是 '(MCP)'）
  // 4. 返回 { commandName, args, isMcp }
}
```

**示例：**
- `/search foo bar` → `{ commandName: 'search', args: 'foo bar', isMcp: false }`
- `/mcp:tool (MCP) arg1 arg2` → `{ commandName: 'mcp:tool (MCP)', args: 'arg1 arg2', isMcp: true }`
- `/` → `null`（无效）
- `not a command` → `null`（没有前导 `/`）

命令名称验证使用 `looksLikeCommand()`：

```typescript
export function looksLikeCommand(commandName: string): boolean {
  return !/[^a-zA-Z0-9:\-_]/.test(commandName)
}
```

这将命令名称与文件路径区分开来 — `/var/log` 被检测为文件路径，而不是命令。

## 命令类别

命令可以按功能分组：

| 类别 | 命令 |
|----------|----------|
| **会话管理** | `/clear`、`/compact`、`/resume`、`/rename`、`/session`、`/exit` |
| **配置** | `/config`（别名：`/settings`）、`/model`、`/theme`、`/color`、`/vim`、`/keybindings`、`/permissions`、`/privacy-settings`、`/output-style`、`/effort`、`/sandbox-toggle`、`/hooks` |
| **信息与状态** | `/help`、`/status`、`/cost`、`/usage`、`/stats`、`/statusline`、`/version`、`/release-notes`、`/diff`、`/context`、`/files`、`/insights` |
| **开发工作流** | `/review`、`/ultrareview`、`/plan`、`/fast`、`/passes`、`/branch`、`/tag`、`/tasks`、`/commit`、`/commit-push-pr`、`/security-review`、`/autofix-pr`、`/pr_comments` |
| **插件与扩展** | `/plugin`、`/mcp`、`/skills`、`/reload-plugins`、`/install-github-app`、`/install-slack-app` |
| **认证** | `/login`、`/logout`、`/oauth-refresh` |
| **通信** | `/copy`、`/share`、`/feedback`、`/btw`、`/export` |
| **调试与诊断** | `/doctor`、`/heapdump`、`/debug-tool-call`、`/ctx_viz`、`/ant-trace`、`/perf-issue`、`/env`、`/remote-env` |
| **IDE 集成** | `/ide`、`/desktop`、`/chrome` |
| **AI 功能** | `/memory`、`/thinkback`、`/thinkback-play`、`/advisor`、`/summary`、`/rewind` |
| **远程/桥接** | `/bridge`、`/bridge-kick`、`/mobile`、`/remote-setup`、`/remoteControlServer` |
| **功能门控** | `/proactive`、`/brief`、`/assistant`、`/voice`、`/fork`、`/peers`、`/buddy`、`/workflows`、`/torch`、`/ultraplan`、`/subscribe-pr` |
| **仅内部（Ant）** | `/backfill-sessions`、`/break-cache`、`/bughunter`、`/good-claude`、`/issue`、`/init-verifiers`、`/force-snip`、`/mock-limits`、`/teleport`、`/agents-platform` |

## 命令执行流

```
用户输入: /compact summarize key decisions
                    │
                    ▼
        ┌───────────────────────┐
        │   processUserInput()  │  ← 顶层入口
        │   (processUserInput.ts)│
        └───────────┬───────────┘
                    │ 输入以 '/' 开头
                    ▼
        ┌───────────────────────┐
        │ processUserInputBase()│  ← 桥接检查、ultraplan 重写、
        │                       │     图像处理、附件
        └───────────┬───────────┘
                    │
                    ▼
        ┌───────────────────────┐
        │ processSlashCommand() │  ← 主要分发
        │ (processSlashCommand.)│
        │        tsx            │
        └───────────┬───────────┘
                    │
          ┌─────────┼─────────┐
          ▼         ▼         ▼
    ┌─────────┐ ┌──────┐ ┌──────────┐
    │parseSlash│ │hasCmd│ │getMessages│
    │Command() │ │check │ │ForSlash  │
    └─────────┘ └──────┘ │Command() │
          │         │    └────┬─────┘
          │         │         │
          ▼         ▼    ┌────┴─────┐
    {name, args}   true  │ type switch│
                         │           │
                         ▼           ▼
                    ┌────────┐ ┌──────────┐ ┌──────────┐
                    │local   │ │local-jsx │ │ prompt   │
                    │        │ │          │ │          │
                    │mod.call│ │mod.call  │ │getPrompt │
                    │(args)  │ │(onDone)  │ │ForCommand│
                    └───┬────┘ └────┬─────┘ └────┬─────┘
                        │           │            │
                        ▼           ▼            ▼
                   LocalCommand  Ink JSX    ContentBlock[]
                   Result        UI         → model
```

### 结果处理

命令执行后，结果被转换为消息：

- **`local` 文本结果**：包装在 `<local-command-stdout>` 标签中作为系统消息
- **`local` compact 结果**：触发 `buildPostCompactMessages()` 用摘要重建对话
- **`local` skip 结果**：返回空消息，不查询
- **`local-jsx` 结果**：JSX 通过 `setToolJSX()` 渲染。`onDone()` 回调控制消息生成
- **`prompt` 内联结果**：内容块成为带有 `isMeta: true` 的用户消息
- **`prompt` 分叉结果**：通过 `runAgent()` 运行子 agent，收集消息，包装在 `<local-command-stdout>` 中

## 命令选项

命令参数作为原始字符串传递给命令的 `call()` 或 `getPromptForCommand()` 方法。参数解析由每个命令实现负责。

**参数提示**在类型提示 UI 中显示：

```typescript
// compact/index.ts
argumentHint: '<optional custom summarization instructions>'

// model/index.ts
argumentHint: '[model]'
```

**敏感命令**从历史记录中编辑其参数：

```typescript
isSensitive: true  // 参数显示为 '***'
```

**即时命令**无需等待模型停止点即可执行：

```typescript
immediate: true  // 绕过命令队列
```

## 命令帮助

`/help` 命令是一个 `local-jsx` 命令，懒加载其实现：

```typescript
// commands/help/index.ts
const help = {
  type: 'local-jsx',
  name: 'help',
  description: 'Show help and available commands',
  load: () => import('./help.js'),
} satisfies Command
```

命令描述通过 `formatDescriptionWithSource()` 显示来源注释：

- 内置命令：纯描述
- 插件命令：`(PluginName) description`
- 工作流命令：`description (workflow)`
- 捆绑技能：`description (bundled)`
- 用户技能：`description (source-name)`

## 远程模式过滤

存在两种独立的过滤机制：

### 1. 远程模式（`--remote`）

`filterCommandsForRemoteMode()` 过滤到 `REMOTE_SAFE_COMMANDS`：

```typescript
export const REMOTE_SAFE_COMMANDS: Set<Command> = new Set([
  session,   // 显示 QR 码 / URL 用于远程会话
  exit,      // 退出 TUI
  clear,     // 清屏
  help,      // 显示帮助
  theme,     // 更改终端主题
  color,     // 更改 agent 颜色
  vim,       // 切换 vim 模式
  cost,      // 显示会话成本
  usage,     // 显示使用信息
  copy,      // 复制最后一条消息
  btw,       // 快速笔记
  feedback,  // 发送反馈
  plan,      // 计划模式切换
  keybindings,
  statusline,
  stickers,
  mobile,    // 移动 QR 码
])
```

这些命令仅影响本地 TUI 状态，不依赖本地文件系统、git、shell、IDE、MCP 或其他本地执行上下文。

### 2. 桥接模式（从移动/网络远程控制）

`isBridgeSafeCommand()` 确定是否应执行通过桥接接收的命令：

```typescript
export function isBridgeSafeCommand(cmd: Command): boolean {
  if (cmd.type === 'local-jsx') return false   // Ink UI 阻止
  if (cmd.type === 'prompt') return true       // 技能安全（展开为文本）
  return BRIDGE_SAFE_COMMANDS.has(cmd)         // 显式本地允许列表
}

export const BRIDGE_SAFE_COMMANDS: Set<Command> = new Set([
  compact, clear, cost, summary, releaseNotes, files,
])
```

在 `processUserInputBase()` 中，桥接来源的输入在斜杠命令门之前检查：

```typescript
if (bridgeOrigin && inputString.startsWith('/')) {
  const parsed = parseSlashCommand(inputString)
  const cmd = findCommand(parsed.commandName, commands)
  if (cmd) {
    if (isBridgeSafeCommand(cmd)) {
      effectiveSkipSlash = false  // 允许执行
    } else {
      return { messages: [...], shouldQuery: false,
        resultText: `/${getCommandName(cmd)} isn't available over Remote Control.` }
    }
  }
}
```

## 关键命令概述

| 命令 | 类型 | 描述 |
|---------|------|-------------|
| `/help` | local-jsx | 显示帮助和可用命令 |
| `/config` | local-jsx | 打开配置面板（别名：`/settings`） |
| `/compact` | local | 清除对话，保留摘要在上下文中 |
| `/clear` | local | 清除对话历史（别名：`/reset`、`/new`） |
| `/model` | local-jsx | 设置 AI 模型（动态描述显示当前模型） |
| `/session` | local-jsx | 显示远程会话 URL 和 QR 码（仅在远程模式） |
| `/exit` | local-jsx | 退出 REPL（别名：`/quit`，即时） |
| `/copy` | local-jsx | 复制最后一条消息到剪贴板 |
| `/theme` | local-jsx | 更改终端主题 |
| `/vim` | local-jsx | 切换 vim 模式 |
| `/cost` | local-jsx | 显示会话成本 |
| `/usage` | local-jsx | 显示使用信息 |
| `/plan` | local-jsx | 切换计划模式 |
| `/skills` | local-jsx | 管理技能 |
| `/mcp` | local-jsx | 管理 MCP 服务器 |
| `/login` | local-jsx | 向 Anthropic 认证 |
| `/logout` | local-jsx | 登出 |
| `/review` | local | 审查代码更改 |
| `/status` | local-jsx | 显示会话状态 |
| `/doctor` | local-jsx | 诊断配置问题 |
| `/memory` | local-jsx | 管理长期内存 |
| `/plugin` | local-jsx | 管理插件 |
| `/share` | local-jsx | 分享会话 |
| `/feedback` | local-jsx | 发送反馈 |
| `/btw` | local-jsx | 快速笔记附加到上下文 |
| `/diff` | local-jsx | 显示当前差异 |
| `/files` | local-jsx | 列出跟踪的文件 |
| `/resume` | local-jsx | 恢复之前的会话 |
| `/rename` | local-jsx | 重命名当前会话 |
| `/keybindings` | local-jsx | 管理键盘绑定 |
| `/permissions` | local-jsx | 管理工具权限 |
| `/privacy-settings` | local-jsx | 隐私配置 |
| `/hooks` | local-jsx | 管理钩子 |
| `/output-style` | local-jsx | 配置输出格式 |
| `/effort` | local-jsx | 设置努力级别 |
| `/sandbox-toggle` | local-jsx | 切换沙箱模式 |
| `/chrome` | local-jsx | Chrome 浏览器集成 |
| `/mobile` | local-jsx | 移动 QR 码生成 |
| `/stickers` | local-jsx | 贴纸管理 |
| `/statusline` | local | 状态行切换 |
| `/release-notes` | local | 显示更新日志 |
| `/extra-usage` | local | 额外使用信息 |
| `/rate-limit-options` | local | 速率限制配置 |
| `/stats` | local-jsx | 显示统计信息 |
| `/fast` | local-jsx | 快速模式切换 |
| `/passes` | local-jsx | 通过配置 |
| `/branch` | local-jsx | 分支管理 |
| `/tag` | local-jsx | 标签管理 |
| `/tasks` | local-jsx | 任务管理 |
| `/agents` | local-jsx | Agent 管理 |
| `/add-dir` | local-jsx | 添加目录到上下文 |
| `/context` | local-jsx | 显示上下文信息 |
| `/desktop` | local-jsx | 桌面集成 |
| `/ide` | local-jsx | IDE 集成 |
| `/terminal-setup` | local-jsx | 终端配置 |
| `/upgrade` | local-jsx | 升级 Claude Code |
| `/export` | local-jsx | 导出会话数据 |
| `/rewind` | local-jsx | 回退对话 |
| `/thinkback` | local-jsx | 回溯功能 |
| `/thinkback-play` | local-jsx | 播放回溯录音 |
| `/advisor` | local | 顾问功能 |
| `/summary` | local | 总结对话 |
| `/teleport` | local | 传送功能 |
| `/security-review` | local | 安全审查 |
| `/commit` | local | Git 提交 |
| `/commit-push-pr` | local | 提交、推送并创建 PR |
| `/autofix-pr` | local | 自动修复 PR 问题 |
| `/pr_comments` | local-jsx | PR 评论管理 |
| `/ultrareview` | local | 超级审查模式 |
| `/heapdump` | local | 内存堆转储 |
| `/mock-limits` | local | 模拟限制（内部） |
| `/version` | local | 显示版本（内部） |
| `/env` | local-jsx | 环境变量 |
| `/remote-env` | local-jsx | 远程环境 |
| `/oauth-refresh` | local | 刷新 OAuth 令牌 |
| `/debug-tool-call` | local | 调试工具调用 |
| `/ant-trace` | local | Ant 跟踪（内部） |
| `/perf-issue` | local | 性能问题报告 |
| `/init` | local | 初始化项目 |
| `/init-verifiers` | local | 初始化验证器（内部） |
| `/backfill-sessions` | local | 补填会话（内部） |
| `/break-cache` | local | 破坏缓存（内部） |
| `/bughunter` | local | Bug 狩猎（内部） |
| `/good-claude` | local | Good Claude（内部） |
| `/issue` | local | 问题创建（内部） |
| `/force-snip` | local | 强制历史裁剪（内部） |
| `/bridge-kick` | local | 踢出桥接连接 |
| `/ctx_viz` | local | 上下文可视化（内部） |

## 关键算法

### 命令查找

```typescript
export function findCommand(commandName: string, commands: Command[]): Command | undefined {
  return commands.find(
    _ =>
      _.name === commandName ||
      getCommandName(_) === commandName ||
      _.aliases?.includes(commandName),
  )
}
```

命令通过 `name`、`userFacingName()` 或任何 `alias` 匹配。

### 可用性过滤

```typescript
export function meetsAvailabilityRequirement(cmd: Command): boolean {
  if (!cmd.availability) return true
  for (const a of cmd.availability) {
    switch (a) {
      case 'claude-ai':
        if (isClaudeAISubscriber()) return true
        break
      case 'console':
        if (!isClaudeAISubscriber() && !isUsing3PServices() && isFirstPartyAnthropicBaseUrl())
          return true
        break
    }
  }
  return false
}
```

没有 `availability` 的命令是通用的。有 `availability` 的命令必须匹配至少一种认证类型。

### 动态技能去重

当调用 `getCommands()` 时，动态技能针对基本命令去重：

```typescript
const baseCommandNames = new Set(baseCommands.map(c => c.name))
const uniqueDynamicSkills = dynamicSkills.filter(
  s => !baseCommandNames.has(s.name) && meetsAvailabilityRequirement(s) && isCommandEnabled(s),
)
// 插入在插件技能之后、内置命令之前
const insertIndex = baseCommands.findIndex(c => builtInNames.has(c.name))
```

### 缓存策略

- `COMMANDS()` — 缓存的（无键），每次调用返回相同数组
- `builtInCommandNames()` — 缓存的（无键），返回相同 Set
- `loadAllCommands(cwd)` — 按 `cwd` 缓存，缓存昂贵的磁盘 I/O
- `getSkillToolCommands(cwd)` — 按 `cwd` 缓存
- `getSlashCommandToolSkills(cwd)` — 按 `cwd` 缓存
- `meetsAvailabilityRequirement()` — **不**缓存（认证状态可能在会话期间更改）
- `isCommandEnabled()` — **不**缓存（功能标志可能更改）

### 懒加载模式

所有命令使用懒加载以最小化启动时间：

```typescript
// 命令注册（轻量级）
const compact = {
  type: 'local',
  name: 'compact',
  description: 'Clear conversation history but keep a summary...',
  load: () => import('./compact.js'),  // 动态导入
} satisfies Command

// 实现（在首次使用时加载）
// compact/compact.js 导出：
export async function call(args: string, context: LocalJSXCommandContext) {
  // 重的实现在这里
}
```

## 依赖

### 内部依赖

| 模块 | 用途 |
|--------|---------|
| `src/types/command.ts` | 命令类型定义 |
| `src/types/message.ts` | 命令结果的消息类型 |
| `src/bootstrap/state.ts` | 应用状态访问（会话 ID、远程模式等） |
| `src/constants/messages.ts` | 错误消息（`NO_CONTENT_MESSAGE`） |
| `src/constants/xml.ts` | XML 标签常量（`COMMAND_MESSAGE_TAG`、`COMMAND_NAME_TAG`） |
| `src/Tool.ts` | `ToolUseContext`、`SetToolJSXFn` 类型 |
| `src/utils/messages.ts` | 消息创建工具 |
| `src/utils/debug.ts` | 调试日志 |
| `src/utils/log.ts` | 错误日志 |
| `src/utils/errors.ts` | 错误类型（`AbortError`、`MalformedCommandError`） |
| `src/utils/envUtils.ts` | 环境变量工具 |
| `src/utils/permissions/permissions.ts` | 工具权限检查 |
| `src/utils/attachments.ts` | 附件提取 |
| `src/utils/telemetry/events.ts` | 遥测事件日志 |
| `src/services/analytics/index.js` | 分析事件跟踪 |
| `src/services/compact/compact.js` | 压缩逻辑 |
| `src/tools/AgentTool/runAgent.js` | 用于分叉命令的子 agent 执行 |
| `src/skills/loadSkillsDir.js` | 技能目录命令加载 |
| `src/skills/bundledSkills.js` | 捆绑技能加载 |
| `src/plugins/builtinPlugins.js` | 内置插件命令 |
| `src/utils/plugins/loadPluginCommands.js` | 插件命令加载 |
| `src/utils/slashCommandParsing.ts` | 斜杠命令字符串解析 |
| `src/utils/processUserInput/*.ts` | 输入处理管道 |

### 外部依赖

| 包 | 用途 |
|---------|---------|
| `@anthropic-ai/sdk` | `ContentBlockParam`、`TextBlockParam` 类型 |
| `lodash-es/memoize.js` | 命令缓存的记忆化 |
| `bun:bundle` | 功能标志检测（`feature()`） |
| `crypto` | UUID 生成 |

## 数据流

### 命令注册流

```
模块加载
    │
    ├── 静态导入 ~70 个内置命令
    ├── 条件 require() 用于功能门控命令
    │
    ▼
COMMANDS() [缓存数组]
    │
    ├── getSkills(cwd) → skillDirCommands, pluginSkills, bundledSkills, builtinPluginSkills
    ├── getPluginCommands() → pluginCommands
    ├── getWorkflowCommands(cwd) → workflowCommands（功能门控）
    │
    ▼
loadAllCommands(cwd) [按 cwd 缓存]
    │
    ├── meetsAvailabilityRequirement() 过滤（每次调用，不缓存）
    ├── isCommandEnabled() 过滤（每次调用，不缓存）
    ├── 动态技能去重
    │
    ▼
getCommands(cwd) → 过滤的 Command[]
```

### 命令执行流

```
用户输入: /command args
    │
    ▼
processUserInput()
    │
    ├── 桥接安全检查（如果有 bridgeOrigin）
    ├── Ultraplan 关键字检查
    ├── 附件提取
    │
    ▼
processUserInputBase()
    │
    ├── 输入以 '/' 开头？
    │
    ▼
processSlashCommand()
    │
    ├── parseSlashCommand() → { commandName, args, isMcp }
    ├── hasCommand() 检查
    │   └── 如果未找到：检查看起来像命令还是文件路径
    │       └── 已知命令 → "Unknown skill: X" 错误
    │       └── 不是命令 → 当作常规提示
    │
    ▼
getMessagesForSlashCommand()
    │
    ├── userInvocable 检查（如果为 false 则阻止）
    ├── 记录 prompt 命令的 skill 使用
    │
    ├── type === 'local-jsx'
    │   ├── command.load() → mod.call(onDone, context, args)
    │   ├── onDone() 回调解析 Promise
    │   └── setToolJSX() 渲染 Ink UI
    │
    ├── type === 'local'
    │   ├── command.load() → mod.call(args, context)
    │   └── 返回 LocalCommandResult：
    │       ├── 'text' → 带结果的系统消息
    │       ├── 'compact' → buildPostCompactMessages()
    │       └── 'skip' → 空消息
    │
    └── type === 'prompt'
        ├── context === 'fork'?
        │   └── executeForkedSlashCommand() → runAgent() 子 agent
        │       ├── 等待 MCP 稳定（最多 10 秒）
        │       ├── 运行 agent，收集消息
        │       ├── 提取结果文本
        │       └── 包装在 <local-command-stdout> 中
        │
        └── inline（默认）
            └── getMessagesForPromptSlashCommand()
                ├── 注册技能钩子
                ├── 添加调用的技能用于压缩保留
                ├── command.getPromptForCommand() → ContentBlockParam[]
                ├── 从技能内容提取附件
                └── 用元数据构建消息
    │
    ▼
返回 ProcessUserInputBaseResult
    │
    ├── messages: Message[]
    ├── shouldQuery: boolean
    ├── allowedTools?: string[]
    ├── model?: string
    ├── effort?: EffortValue
    ├── resultText?: string
    ├── nextInput?: string
    └── submitNextInput?: boolean
```

## 集成点

### 主入口点

命令在应用初始化期间加载并传递给 REPL 组件。在 `--remote` 模式中，命令通过 `filterCommandsForRemoteMode()` 预过滤。

### 状态管理

命令通过 `ToolUseContext` 访问应用状态，提供 `getAppState()`、`setAppState()` 和会话作用域数据。

### 工具系统

- **SkillTool**：使用 `getSkillToolCommands()` 显示模型可调用的技能
- **SlashCommandTool**：使用 `getSlashCommandToolSkills()` 获取斜杠命令技能
- 命令可以通过 `allowedTools` 授予额外的工具权限

### 压缩系统

- `/compact` 和 `/clear` 调用 `clearSystemPromptSections()` 重置缓存的系统提示部分
- 调用的技能通过 `addInvokedSkill()` 跟踪用于压缩保留
- `/compact` 返回 `type: 'compact'` 的 `LocalCommandResult`，触发完整对话重建

### 远程控制桥接

- 来自移动/网络的命令通过 `isBridgeSafeCommand()` 过滤
- `BRIDGE_SAFE_COMMANDS` 允许列表控制哪些 `local` 命令可以执行
- `prompt` 命令始终安全（展开为文本）
- `local-jsx` 命令始终阻止（渲染终端 UI）

### 分析与遥测

每次命令调用记录：
- `tengu_input_command` — 命令名称、来源、插件元数据
- `tengu_input_slash_invalid` — 未知命令尝试
- `tengu_input_slash_missing` — 格式错误的斜杠命令
- `tengu_input_prompt` — 常规提示检测
- 通过 `buildPluginCommandTelemetryFields()` 的插件特定遥测

### 钩子系统

Prompt 命令可以通过 `hooks` 属性注册钩子：

```typescript
if (command.hooks && hooksAllowedForThisSkill) {
  registerSkillHooks(context.setAppState, sessionId, command.hooks, command.name, command.skillRoot)
}
```

当 `hooks` 设置仅限于插件时，来自非管理员信任来源的钩子被阻止。

## 相关模块

- [主入口点](./main-entrypoint.md)
- [工具系统](./tool-system.md)
- [状态管理](./state-management.md)
