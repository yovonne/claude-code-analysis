# 技能系统

## 目的

技能系统提供定义可重用、可调用工作流程的机制，Claude 模型可以执行这些工作流程。技能是基于提示的命令，为模型提供特定任务的详细指令。系统支持多种技能来源：捆绑（随 CLI 一起发布）、用户定义（文件系统）、项目级、管理（策略）、遗留命令和 MCP 发现的技能。

## 位置

- `restored-src/src/skills/bundledSkills.ts` — 捆绑技能定义类型和注册表
- `restored-src/src/skills/loadSkillsDir.ts` — 从目录加载技能、解析、去重、动态发现
- `restored-src/src/skills/mcpSkillBuilders.ts` — MCP 技能发现的写一次注册表（避免循环导入）
- `restored-src/src/skills/bundled/index.ts` — 捆绑技能初始化
- `restored-src/src/skills/bundled/` — 各捆绑技能实现

## 主要导出

### 类型（bundledSkills.ts）

- `BundledSkillDefinition`: 随 CLI 一起发布的技能定义。包括名称、描述、别名、允许的工具、模型、钩子、上下文、代理、文件和 `getPromptForCommand`。

### 函数（bundledSkills.ts）

- `registerBundledSkill(definition)`: 注册捆绑技能。处理带嵌入式引用文件的技能的可选文件提取。
- `getBundledSkills()`: 返回所有注册捆绑技能作为 `Command[]` 的副本。
- `clearBundledSkills()`: 清除注册表用于测试。
- `getBundledSkillExtractDir(skillName)`: 返回技能引用文件的确定性提取目录路径。

### 函数（loadSkillsDir.ts）

- `getSkillsPath(source, dir)`: 根据来源（userSettings、projectSettings、policySettings、plugin）返回 skills/commands 目录的文件系统路径。
- `estimateSkillFrontmatterTokens(skill)`: 从技能名称、描述和 whenToUse 估算令牌数。
- `parseSkillFrontmatterFields(frontmatter, markdownContent, resolvedName)`: 解析文件基础和 MCP 技能加载的所有共享 frontmatter 字段。
- `createSkillCommand(params)`: 从解析的技能数据创建 `Command` 对象，包括带参数替换和 shell 执行的 `getPromptForCommand`。
- `getSkillDirCommands(cwd)`: 记忆化函数，从管理、用户、项目、其他和遗留命令目录加载所有技能。处理去重和条件技能激活。
- `discoverSkillDirsForPaths(filePaths, cwd)`: 从文件路径向上遍历以发现嵌套的 `.claude/skills` 目录。
- `addSkillDirectories(dirs)`: 从发现的目录加载技能到动态技能映射。
- `activateConditionalSkillsForPaths(filePaths, cwd)`: 当匹配文件被触及时激活带有 `paths` frontmatter 的技能。
- `getDynamicSkills()`: 返回所有动态发现的技能。
- `getConditionalSkillCount()`: 返回待定条件技能计数。
- `clearSkillCaches()`, `clearDynamicSkills()`: 清除缓存和动态技能状态用于测试。

### 函数（mcpSkillBuilders.ts）

- `registerMCPSkillBuilders(builders)`: 注册 MCP 技能发现需要的两个函数（`createSkillCommand`、`parseSkillFrontmatterFields`）。
- `getMCPSkillBuilders()`: 检索注册的 builders。如果尚未注册则抛出错误。

### 捆绑技能（bundled/index.ts）

`initBundledSkills()` 在启动时注册所有捆绑技能：

| 技能 | 用途 |
|---|---|
| `update-config` | 配置 settings.json、权限、钩子、环境变量 |
| `keybindings-help` | 在 `~/.claude/keybindings.json` 中自定义键盘快捷键 |
| `verify` | 通过运行应用程序验证代码更改 [ANT-only] |
| `debug` | 通过调试日志分析调试会话问题 |
| `lorem-ipsum` | 生成填充文本用于长上下文测试 [ANT-only] |
| `skillify` | 将会话流程捕获为可重用技能 [ANT-only] |
| `remember` | 回顾和组织自动记忆条目 [ANT-only] |
| `simplify` | 代码审查以实现重用、质量和效率 |
| `batch` | 跨 5-30 个工作树代理的并行工作编排 |
| `stuck` | 诊断冻结/缓慢的 Claude Code 会话 [ANT-only] |
| `claude-in-chrome` | 通过 MCP 工具实现 Chrome 浏览器自动化 |
| `dream` | [KAIROS 功能门] |
| `hunter` | [REVIEW_ARTIFACT 功能门] |
| `loop` | [AGENT_TRIGGERS 功能门] |
| `scheduleRemoteAgents` | [AGENT_TRIGGERS_REMOTE 功能门] |
| `claudeApi` | [BUILDING_CLAUDE_APPS 功能门] |
| `runSkillGenerator` | [RUN_SKILL_GENERATOR 功能门] |

## 依赖

### 内部依赖

- `../types/command.js` — `Command` 和 `PromptCommand` 类型
- `../Tool.js` — `ToolUseContext` 类型
- `../utils/frontmatterParser.js` — YAML frontmatter 解析
- `../utils/argumentSubstitution.js` — `$arg` 占位符替换
- `../utils/markdownConfigLoader.js` — 遗留命令文件加载
- `../utils/settings/` — 设置加载、管理路径、仅插件策略
- `../utils/git/gitignore.js` — 动态发现的 Gitignore 检查
- `../utils/permissions/filesystem.js` — 捆绑技能根路径
- `../services/analytics/index.js` — 动态技能更改的事件日志
- `../services/tokenEstimation.js` — 令牌计数估算

### 外部依赖

- `ignore` — 用于条件技能的 Gitignore 风格模式匹配
- `lodash-es/memoize.js` — `getSkillDirCommands` 的记忆化
- `fs/promises` — 文件系统操作
- `path` — 路径操作

## 实现细节

### 技能加载架构

技能在启动时并行从多个来源加载：

```
getSkillDirCommands(cwd) [记忆化]
    ↓
并行加载:
├── 管理技能（策略）— ~/.claude/managed/.claude/skills/
├── 用户技能 — ~/.claude/skills/
├── 项目技能 — .claude/skills/（从项目目录到主目录）
├── 其他目录技能 — --add-dir 路径
└── 遗留命令 — ~/.claude/commands/ 和 .claude/commands/
    ↓
通过 realpath 去重（处理符号链接）
    ↓
分离条件技能（带 paths frontmatter）
    ↓
返回无条件技能；存储条件技能以供后续激活
```

### Frontmatter 解析

技能在其 `SKILL.md` 文件中使用 YAML frontmatter：

```yaml
---
name: my-skill
description: One-line description
when_to_use: Detailed description of when to auto-invoke
allowed-tools: [Read, Bash(gh:*)]
argument-hint: "<arg1> <arg2>"
arguments: [arg1, arg2]
context: inline | fork
model: sonnet | opus | haiku | inherit
user-invocable: true | false
disable-model-invocation: true | false
effort: low | medium | high | N
paths: [src/**/*.ts, tests/**/*.ts]
hooks: { PreToolUse: [...] }
agent: agent-name
version: 1.0.0
---
```

### 捆绑技能文件提取

技能可以通过 `files` 字段包含嵌入式引用文件：

1. 首次调用时，文件提取到捆绑技能根下的确定性目录
2. 提取被记忆化（promise 级别）以防止并发调用者的竞争条件
3. 文件使用 `O_EXCL | O_NOFOLLOW` 标志和 `0o600` 权限写入以确保安全
4. 路径遍历被验证 — 包含 `..` 或绝对路径的路径抛出错误
5. 技能提示前缀为 `Base directory for this skill: <dir>` 以便模型可以 Read/Grep 引用文件

### 动态技能发现

当文件被操作时，系统从文件的父目录向上遍历到 `cwd`，寻找 `.claude/skills/` 目录：

1. 每个发现的目录针对 `Set` 检查以避免冗余的 stat 调用
2. 被 Gitignore 的目录被跳过（通过 `git check-ignore`）
3. 技能加载时较深的路径优先于较浅的路径
4. `skillsLoaded` 信号通知监听器清除缓存

### 条件技能

带有 `paths` frontmatter 字段的技能被单独存储，当匹配文件被触及时激活：

1. 路径匹配使用 `ignore` 库（gitignore 风格模式）
2. 当文件操作发生时，所有条件技能针对文件路径检查
3. 匹配的技能从 `conditionalSkills` 移动到 `dynamicSkills`
4. 一旦激活，技能保持激活状态（在 `activatedConditionalSkillNames` 中跟踪）

### MCP 技能发现

为避免 `mcpSkills.ts` 和 `loadSkillsDir.ts` 之间的循环导入，MCP 需要的两个函数（`createSkillCommand`、`parseSkillFrontmatterFields`）在模块初始化时通过 `mcpSkillBuilders.ts` 叶子模块注册。该模块仅导入类型，因此两边都可以依赖它而不会形成循环。

### 参数替换

技能通过 `$arg_name` 占位符支持位置参数：

1. `argument-hint` 提供面向用户的指导
2. `arguments` 列出参数名称
3. `substituteArguments()` 在调用时替换技能提示中的占位符
4. `${CLAUDE_SKILL_DIR}` 替换为技能的目录路径
5. `${CLAUDE_SESSION_ID}` 替换为当前会话 ID

### 技能中的 Shell 执行

技能可以使用 `!`...`` 或 `!`...`!` 语法包含内联 shell 命令：

- Shell 命令通过 `executeShellCommandsInPrompt()` 执行
- MCP 技能**永远不允许**执行内联 shell 命令（安全边界）
- 非 MCP 技能为其 `allowedTools` 设置 `alwaysAllowRules` 以便 shell 命令权限

## 数据流

```
CLI 启动
    ↓
initBundledSkills() → 为每个技能调用 registerBundledSkill()
    ↓
getSkillDirCommands(cwd) [记忆化]
    ↓
┌─────────────────────────────────────────────────┐
│ 并行从所有来源加载                                 │
│ ├── 管理（策略）技能                              │
│ ├── 用户技能（~/.claude/skills/）                 │
│ ├── 项目技能（.claude/skills/）                   │
│ ├── 其他目录技能（--add-dir）                     │
│ └── 遗留命令（.claude/commands/）                 │
└─────────────────────────────────────────────────┘
    ↓
通过 realpath 去重
    ↓
├── 无条件技能 → 立即返回
└── 条件技能 → 存储以供路径激活
    ↓
发生文件操作
    ↓
├── discoverSkillDirsForPaths() → 查找新的 .claude/skills/ 目录
├── addSkillDirectories() → 加载新发现的技能
└── activateConditionalSkillsForPaths() → 激活匹配的技能
    ↓
动态技能与启动技能合并
    ↓
通过 Skill 工具供模型使用
```

## 集成点

- **Skill 工具** — 技能的主要调用机制
- **命令系统** — 技能是 `type: 'prompt'` 的 `Command` 对象
- **动态发现** — 当相关文件被触及时技能自动发现
- **MCP 系统** — MCP 服务器可以通过 builders 注册表发现和注册技能
- **插件系统** — 内置插件可以提供技能定义
- **Bare 模式** — `--bare` 标志跳过自动发现，仅加载显式 `--add-dir` 路径和捆绑技能
- **仅插件策略** — `isRestrictedToPluginOnly('skills')` 可以将技能锁定为仅插件

## 配置

| 环境变量 | 用途 |
|---|---|
| `CLAUDE_CODE_DISABLE_POLICY_SKILLS` | 禁用管理（策略）技能加载 |

| 设置 | 用途 |
|---|---|
| `projectSettings` enabled | 允许项目级技能发现 |
| `skillsLocked`（仅插件） | 将技能限制为仅插件 |

| Bare 模式 | 行为 |
|---|---|
| `--bare` 标志 | 跳过所有自动发现；仅加载 `--add-dir` 路径和捆绑技能 |

## 错误处理

- 缺少技能目录：记录为警告，返回空数组
- 无效 frontmatter：通过 `logForDebugging` 记录，跳过技能
- 文件提取失败：记录，技能继续但不添加 base-directory 前缀
- 重复技能：通过 realpath 检测，首次命中去重并记录
- 捆绑文件中的路径遍历：抛出错误，技能注册失败
- 循环导入避免：MCP builders 使用叶子模块模式

## 测试

- `clearBundledSkills()`、`clearDynamicSkills()`、`clearSkillCaches()` 提供测试隔离
- `rollWithSeed()` 模式不适用于此，但 `getBundledSkillExtractDir()` 是确定性的
- `parseSkillFrontmatterFields()` 是纯函数，易于测试
- `createSkillCommand()` 可以用模拟参数测试

## 相关模块

- [SkillTool](../02-tools/SkillTool.md) — 调用技能的工具
- [Plugins System](./plugins-system.md) — 插件可以提供技能
- [Hooks Utils](../05-utils/hooks-utils.md) — 技能可以定义钩子
- [Command System](../01-core-modules/command-system.md) — 作为命令的技能

## 备注

- 技能系统支持两种目录格式：现代 `skill-name/SKILL.md` 格式和 `/commands/` 目录中的遗留单个 `.md` 文件格式
- 技能默认为 `userInvocable: true` — 用户可以输入 `/skill-name` 来调用它们
- `userInvocable: false` 的技能对用户隐藏，但可以由模型自动调用
- `context: fork` 字段使技能在自己的上下文中作为子代理运行
- 条件技能（带 `paths` frontmatter）是一个强大的功能 — 它们仅在相关文件被触及时激活，保持上下文清洁
- 记忆化的 `getSkillDirCommands()` 使用 `lodash-es/memoize` — 缓存可以用 `clearSkillCaches()` 清除
- 几个捆绑技能是 ANT-only（verify、lorem-ipsum、skillify、remember、stuck），并由 `process.env.USER_TYPE !== 'ant'` 门控
- 功能门控技能（dream、hunter、loop 等）通过 `require()` 加载以进行延迟求值 — 它们不在模块顶层导入以避免加载未使用的代码
