# SkillTool

## 用途

SkillTool 是 Claude 在对话中调用斜杠命令技能（也称为命令或提示）的机制。技能是可重用的、特定领域的指令集，可扩展 Claude 的能力——如 `/commit`、`/review-pr`、`/pdf` 等。该工具将技能名称解析为其定义，将其内容加载到对话上下文中，要么内联执行（将技能消息展开到当前回合），要么在分叉子代理中执行（隔离执行，有自己的令牌预算）。它还支持从云存储实验性加载远程技能。

## 位置

- `restored-src/src/tools/SkillTool/SkillTool.ts`
- `restored-src/src/tools/SkillTool/prompt.ts`
- `restored-src/src/tools/SkillTool/constants.ts`
- `restored-src/src/tools/SkillTool/UI.tsx`

## 主要导出

### 函数

- `SkillTool`：通过 `buildTool()` 构建的主要工具定义——处理技能验证、权限检查、加载和执行
- `getPrompt()`：生成工具系统提示的记忆化函数，包含技能列表
- `formatCommandsWithinBudget()`：在字符预算内格式化技能描述以包含在上下文窗口中
- `getCharBudget()`：根据上下文窗口大小计算技能列表的字符预算
- `getSkillToolInfo()`、`getSkillInfo()`：返回技能计数元数据
- `getLimitedSkillToolCommands()`：返回包含在 SkillTool 提示中的命令
- `clearPromptCache()`：清除记忆化的提示缓存

### 类型

- `Progress`（也称 `SkillToolProgress`）：分叉技能执行的进度类型，包含子代理消息和提示内容
- `Output`：内联和分叉技能结果 schema 的联合
- `InputSchema`：`{ skill: string, args?: string }`

### 常量

- `SKILL_TOOL_NAME`：`'Skill'`
- `SKILL_BUDGET_CONTEXT_PERCENT`：`0.01` — 分配给技能列表的上下文窗口的 1%
- `CHARS_PER_TOKEN`：`4` — 字符转令牌估算因子
- `DEFAULT_CHAR_BUDGET`：`8,000` — 后备预算（200K tokens × 4 的 1%）
- `MAX_LISTING_DESC_CHARS`：`250` — 每个条目的描述长度硬上限

## 依赖

### 内部依赖

- `prompt.ts` — 带有动态技能列表的工具提示生成
- `constants.ts` — 工具名称常量
- `UI.tsx` — 在终端中渲染技能工具使用/结果/进度的 React 组件
- `runAgent.ts` — 核心代理执行引擎（用于分叉技能执行）
- `processSlashCommand.js` — 斜杠命令处理和消息展开
- `commands.js` — 命令发现、加载和管理
- `remoteSkillModules` — 远程技能搜索/加载的条件模块（仅 ant，实验性）
- `agentContext.js` — 代理身份和上下文跟踪
- `messages.js` — 消息创建和规范化工具
- `pluginTelemetry.js` — 插件/技能使用分析

### 外部依赖

- `zod/v4` — Schema 验证
- `react` — UI 组件渲染
- `bun:bundle` — 功能标志门控（`EXPERIMENTAL_SKILL_SEARCH`）
- `lodash-es/uniqBy` — 命令去重
- `path` — 远程技能缓存路径的目录解析

## 实现细节

### 输入 Schema

```typescript
{
  skill: string,   // 技能名称（例如 "commit"、"review-pr"、"pdf"）
  args?: string    // 传递给技能的可选参数
}
```

技能名称通过去除前导斜杠进行规范化（兼容用户输入的 `/skill` 模式）。

### 输出 Schema

输出是根据执行模式的两种形状的联合：

**内联执行（默认）：**
```typescript
{
  success: boolean,
  commandName: string,
  allowedTools?: string[],   // 技能限制的工具
  model?: string,            // 如果指定则模型覆盖
  status?: 'inline'
}
```

**分叉执行：**
```typescript
{
  success: boolean,
  commandName: string,
  status: 'forked',
  agentId: string,           // 执行技能的分叉代理
  result: string             // 分叉执行的结果文本
}
```

### 技能发现和加载

#### 命令来源

技能从多个来源发现，合并和去重：

| 来源 | 描述 |
|--------|-------------|
| 内置 | Claude Code 附带的捆绑技能 |
| 用户设置 | 来自 `.claude/skills/` 或用户配置的技能 |
| 项目设置 | 来自项目级 `.claude/` 目录的技能 |
| 插件/市场 | 从插件市场安装的技能 |
| MCP 技能 | 从 MCP 服务器加载的技能（`loadedFrom === 'mcp'`） |

`getAllCommands()` 函数合并本地命令和 MCP 技能：
```typescript
async function getAllCommands(context: ToolUseContext): Promise<Command[]> {
  const mcpSkills = context
    .getAppState()
    .mcp.commands.filter(cmd => cmd.type === 'prompt' && cmd.loadedFrom === 'mcp')
  if (mcpSkills.length === 0) return getCommands(getProjectRoot())
  const localCommands = await getCommands(getProjectRoot())
  return uniqBy([...localCommands, ...mcpSkills], 'name')
}
```

#### 提示中的技能列表

`getPrompt()` 函数生成 LLM 的指令以及可用技能的格式化列表。技能在字符预算内格式化：

1. 计算预算：`contextWindowTokens × 4 × 0.01`（或通过 `SLASH_COMMAND_TOOL_CHAR_BUDGET` 的环境覆盖）
2. 首先尝试完整描述
3. 如果超出预算，完全保留捆绑技能描述，截断其他
4. 如果严重超出预算，仅显示非捆绑技能名称
5. 每个条目上限：`MAX_LISTING_DESC_CHARS`（250 个字符）

```
Execute a skill within the main conversation

When users ask you to perform tasks, check if any of the available skills match.
...

How to invoke:
- Use this tool with the skill name and optional arguments
- Examples:
  - `skill: "pdf"` - invoke the pdf skill
  - `skill: "commit", args: "-m 'Fix bug'"` - invoke with arguments
  - `skill: "review-pr", args: "123"` - invoke with arguments
  - `skill: "ms-office-suite:pdf"` - invoke using fully qualified name

Important:
- Available skills are listed in system-reminder messages in the conversation
- When a skill matches the user's request, this is a BLOCKING REQUIREMENT
- NEVER mention a skill without actually calling this tool
- Do not invoke a skill that is already running
- If you see a <command-name> tag in the current conversation turn, the skill has ALREADY been loaded
```

### 输入验证

`validateInput()` 函数执行多项检查：

1. **格式检查**：技能名称在修剪后必须非空
2. **斜杠前缀检测**：如果技能名称以 `/` 开头则记录分析
3. **远程规范技能处理**（仅 ant）：根据发现的远程技能验证 `_canonical_<slug>` 名称
4. **命令存在性**：在合并的命令注册表中查找命令
5. **模型调用检查**：拒绝具有 `disableModelInvocation: true` 的技能
6. **提示类型检查**：仅基于提示的技能（`type === 'prompt'`）有效

| 错误代码 | 条件 |
|------------|-----------|
| 1 | 空或仅空白字符的技能名称 |
| 2 | 未知技能名称 |
| 4 | 技能具有 `disableModelInvocation` |
| 5 | 技能不是基于提示的技能 |
| 6 | 远程技能在会话中未发现 |

### 权限检查

`checkPermissions()` 函数确定技能执行是否需要用户权限：

1. **拒绝规则**：首先检查 `SkillTool` 拒绝规则（始终遵守）
2. **远程规范技能**（仅 ant）：自动授予（不涉及用户创作内容）
3. **允许规则**：检查 `SkillTool` 允许规则以进行精确匹配或前缀匹配（`skill:*`）
4. **安全属性检查**：自动允许仅使用安全 `PromptCommand` 属性的技能
5. **默认**：请求用户权限并提供创建允许规则的建议

规则匹配支持：
- **精确匹配**：`commit` 匹配技能 `commit`
- **前缀匹配**：`review:*` 匹配 `review-pr`、`review-doc` 等

安全属性允许列表（`SAFE_SKILL_PROPERTIES`）：
```typescript
const SAFE_SKILL_PROPERTIES = new Set([
  'type', 'progressMessage', 'contentLength', 'argNames', 'model', 'effort',
  'source', 'pluginInfo', 'disableNonInteractive', 'skillRoot', 'context',
  'agent', 'getPromptForCommand', 'frontmatterKeys',
  'name', 'description', 'hasUserSpecifiedDescription', 'isEnabled', 'isHidden',
  'aliases', 'isMcp', 'argumentHint', 'whenToUse', 'paths', 'version',
  'disableModelInvocation', 'userInvocable', 'loadedFrom', 'immediate',
  'userFacingName',
])
```

如果技能具有任何**不**在此集合中且有意义的属性，则需要权限。

### 执行路径

`call()` 方法有三条执行路径：

#### 1. 远程技能执行（仅 ant，实验性）

```
EXPERIMENTAL_SKILL_SEARCH + USER_TYPE=ant + _canonical_<slug>
    │
    ▼
stripCanonicalPrefix(slug)
    │
    ▼
loadRemoteSkill(slug, url) — 从云存储（GCS/S3/HTTP）获取 SKILL.md
    │
    ├── 缓存检查（本地磁盘缓存）
    ├── 解析 frontmatter
    ├── 注入基础目录头
    ├── 替换 ${CLAUDE_SKILL_DIR} 和 ${CLAUDE_SESSION_ID}
    ├── 使用 addInvokedSkill() 注册以在压缩中存活
    │
    ▼
返回带有技能内容作为 meta 用户消息的内联结果
```

远程技能是声明性 markdown——不需要斜杠命令展开（`!command` 替换、`$ARGUMENTS` 插值）。

#### 2. 分叉子代理执行

```
command.context === 'fork'
    │
    ▼
executeForkedSkill()
    │
    ├── prepareForkedCommandContext(command, args, context)
    │   ├── 使用参数处理技能提示
    │   ├── 构建隔离的代理定义
    │   └── 返回修改后的 getAppState、提示消息、技能内容
    │
    ├── 使用分叉上下文运行 runAgent()
    │   ├── 独立的令牌预算
    │   ├── 通过 onProgress 回调进行进度报告
    │   └── 收集代理消息
    │
    ├── extractResultText(agentMessages)
    │
    └── clearInvokedSkillsForAgent(agentId) — 清理
```

分叉技能在隔离的子代理中运行：
- 独立的令牌预算
- 进度消息作为 `skill_progress` 事件转发到父代理
- 结果文本提取并作为 `status: 'forked'` 输出返回
- 技能内容注册以在压缩中存活

#### 3. 内联执行（默认）

```
标准技能调用
    │
    ▼
processPromptSlashCommand(skill, args, commands, context)
    │
    ├── 使用参数展开技能提示
    │   ├── !command 替换
    │   ├── $ARGUMENTS 插值
    │   └── Frontmatter 键注入
    │
    ├── 处理允许的工具限制
    ├── 应用模型覆盖（如果指定）
    ├── 应用努力覆盖（如果指定）
    │
    ▼
返回 newMessages + contextModifier
    │
    ├── newMessages：技能提示消息，标记有 toolUseID
    ├── contextModifier:
    │   ├── allowedTools：自动允许技能的限制工具集
    │   ├── mainLoopModel：解析技能模型覆盖
    │   └── effortValue：应用技能努力级别
```

内联执行将技能内容直接展开到当前对话回合。技能的系统提示和任何参数插值都被处理，结果消息作为标记有工具使用 ID 的用户消息注入。

### 上下文修改

技能可以通过 `contextModifier` 函数修改执行上下文：

| 修改 | 触发器 | 效果 |
|-------------|---------|--------|
| `allowedTools` | 技能指定工具限制 | 在权限上下文中自动允许这些工具 |
| `mainLoopModel` | 技能有 `model` 字段 | 覆盖后续回合的模型 |
| `effortValue` | 技能有 `effort` 字段 | 设置会话的努力级别 |

模型解析使用 `resolveSkillModelOverride()` 来保留后缀如 `[1m]`（例如 `opus[1m]` → 保留有效的上下文窗口）。

### 进度报告

对于分叉技能，进度通过 `onProgress` 回调报告：

```typescript
onProgress({
  toolUseID: `skill_${parentMessage.message.id}`,
  data: {
    message: m,           // 子代理消息
    type: 'skill_progress',
    prompt: skillContent, // 技能的提示内容
    agentId,              // 子代理 ID
  },
})
```

UI 使用 `SubAgentProvider` 和 `MessageComponent` 在压缩模式下渲染进度消息，最多显示 3 条最新消息，并带有 "+N more" 指示器。

### 遥测

每次技能调用都会记录 `tengu_skill_tool_invocation` 事件：

| 字段 | 描述 |
|-------|-------------|
| `command_name` | 清理后的名称（内置/捆绑/官方或"自定义"） |
| `_PROTO_skill_name` | 完整技能名称（PII 标记，未编辑） |
| `execution_context` | `'inline'`、`'fork'` 或 `'remote'` |
| `invocation_trigger` | `'claude-proactive'` 或 `'nested-skill'` |
| `query_depth` | 技能调用嵌套深度 |
| `was_discovered` | 技能是否通过搜索发现（实验性） |
| `skill_name`、`skill_source`、`skill_loaded_from`、`skill_kind` | 内部仅 ant 字段 |
| `plugin_name`、`plugin_repository`、`_PROTO_marketplace_name` | 插件归属 |

附加事件：
- `tengu_skill_tool_slash_prefix`：当模型发送带前导 `/` 的技能名称时
- `tengu_skill_descriptions_truncated`：当技能列表超出预算时

### UI 渲染

#### renderToolUseMessage

在执行期间显示技能名称。来自弃用 `/commands` 文件夹的旧技能带 `/` 前缀显示。

#### renderToolUseProgressMessage

对于分叉技能，显示实时子代理进度：
- 尚无进度消息时显示 "Initializing…"
- 非 verbose 模式下最多显示 3 条最新消息
- Verbose 模式下显示所有消息
- 隐藏消息的 "+N more tool uses" 指示器
- 使用 `SubAgentProvider` 实现可展开的子代理 UI

#### renderToolResultMessage

- **分叉**：显示 "Done" 署名
- **内联**：显示 "Successfully loaded skill"，如果适用则包含工具计数和模型覆盖

#### renderToolUseRejectedMessage / renderToolUseErrorMessage

显示进度历史，然后是后备拒绝/错误消息。

## 配置

### 环境变量

| 变量 | 用途 |
|----------|---------|
| `SLASH_COMMAND_TOOL_CHAR_BUDGET` | 覆盖技能列表字符预算 |
| `USER_TYPE=ant` | 启用仅 ant 功能（远程技能、规范技能、详细遥测） |

### 功能标志

| 标志 | 用途 |
|------|---------|
| `EXPERIMENTAL_SKILL_SEARCH` | 启用远程技能搜索、加载和规范技能 |

## 错误处理

### 验证错误

| 错误代码 | 条件 |
|------------|-----------|
| 1 | 空或无效的技能名称格式 |
| 2 | 未知技能——在命令注册表中未找到 |
| 4 | 技能具有 `disableModelInvocation` 标志 |
| 5 | 技能不是基于提示的技能（例如， plain CLI 命令） |
| 6 | 远程技能在当前会话中未发现 |

### 运行时错误

- **命令处理失败**：如果 `processedCommand.shouldQuery` 为 false 则抛出 `Error('Command processing failed')`
- **远程技能加载失败**：抛出包含 slug 和底层错误消息的描述性错误
- **远程技能未找到**：如果技能之前未通过 `DiscoverSkills` 发现则抛出

## 数据流

```
Model calls Skill tool with { skill, args }
    │
    ▼
validateInput()
    ├── Strip leading slash
    ├── Check remote canonical (ant-only)
    ├── Look up command in registry
    ├── Check disableModelInvocation
    └── Verify prompt-based skill
    │
    ▼
checkPermissions()
    ├── Check deny rules → deny
    ├── Check remote canonical (ant-only) → allow
    ├── Check allow rules → allow
    ├── Check safe properties → allow
    └── Default → ask with suggestions
    │
    ▼
call() — Execution path selection
    │
    ├── [remote canonical] ──→ executeRemoteSkill()
    │   ├── loadRemoteSkill(slug, url)
    │   ├── Parse frontmatter, inject headers
    │   ├── Substitute ${CLAUDE_SKILL_DIR}, ${CLAUDE_SESSION_ID}
    │   ├── addInvokedSkill() for compaction
    │   └── Return inline result with meta user message
    │
    ├── [fork context] ──→ executeForkedSkill()
    │   ├── prepareForkedCommandContext()
    │   ├── runAgent() in isolated context
    │   ├── Progress reporting via onProgress
    │   ├── extractResultText()
    │   └── Return forked result
    │
    └── [inline] ──→ processPromptSlashCommand()
        ├── Expand skill prompt with args
        ├── Build newMessages
        ├── Build contextModifier (tools, model, effort)
        └── Return inline result with messages + context modifier
    │
    ▼
mapToolResultToToolResultBlockParam()
    ├── Forked: "Skill completed (forked execution). Result: ..."
    └── Inline: "Launching skill: <name>"
```

## 集成点

- **命令注册表**：从 `.claude/`、插件、MCP 服务器和捆绑来源发现技能
- **代理系统**：分叉技能使用 `runAgent()` 进行隔离的子代理执行
- **权限系统**：支持具有前缀匹配和安全属性允许列表的允许/拒绝规则
- **插件系统**：跟踪市场技能的插件归属
- **压缩系统**：`addInvokedSkill()` 和 `clearInvokedSkillsForAgent()` 确保技能内容在上下文压缩中存活
- **模型覆盖系统**：技能可以覆盖活动模型和努力级别
- **工具限制**：技能可以限制其执行期间可用的工具
- **SDK 事件队列**：分叉技能执行发出进度事件

## 相关模块

- [AgentTool](./AgentTool.md) — 分叉技能使用的子代理执行引擎
- [ExitPlanModeTool](./ExitPlanModeTool.md) — 计划模式退出（内置技能）
- [commands.js](../commands.js) — 命令发现和管理系统
- [processSlashCommand.js](../utils/processUserInput/processSlashCommand.js) — 斜杠命令处理
- [pluginTelemetry.js](../utils/telemetry/pluginTelemetry.js) — 插件使用分析
