# Schemas

## 目的

用于验证 hook 配置的 Zod v4 schema 定义。提取到单独文件以打破 `settings/types.ts` 和 `plugins/schemas.ts` 之间的循环依赖。

## 位置

`restored-src/src/schemas/hooks.ts`

## 关键导出

### Hook 命令 Schemas

四个区分的 hook 命令类型，由 `type` 字段联合：

#### `BashCommandHookSchema` (`type: 'command'`)
当 hook 事件触发时执行 shell 命令。

| 字段 | 类型 | 描述 |
|-------|------|-------------|
| `command` | `string` | 要执行的 shell 命令 |
| `shell` | `'bash' \| 'powershell'` | Shell 解释器（默认为 bash） |
| `timeout` | `number` | 超时秒数 |
| `statusMessage` | `string` | hook 运行时的自定义旋转器消息 |
| `once` | `boolean` | 运行一次后移除 |
| `async` | `boolean` | 在后台运行而不阻塞 |
| `asyncRewake` | `boolean` | 后台运行，在退出码 2 时唤醒模型 |

#### `PromptHookSchema` (`type: 'prompt'`)
当 hook 事件触发时评估 LLM 提示。

| 字段 | 类型 | 描述 |
|-------|------|-------------|
| `prompt` | `string` | LLM 提示（使用 `$ARGUMENTS` 获取 hook 输入 JSON） |
| `model` | `string` | 要使用的模型（默认为小快模型） |
| `timeout` | `number` | 超时秒数 |
| `statusMessage` | `string` | 自定义旋转器消息 |
| `once` | `boolean` | 运行一次后移除 |

#### `HttpHookSchema` (`type: 'http'`)
将 hook 输入 JSON POST 到 URL。

| 字段 | 类型 | 描述 |
|-------|------|-------------|
| `url` | `string` (URL) | 要 POST到的 URL |
| `headers` | `Record<string, string>` | 额外头部（支持 `$VAR` 插值） |
| `allowedEnvVars` | `string[]` | 头部插值的白名单环境变量 |
| `timeout` | `number` | 超时秒数 |
| `statusMessage` | `string` | 自定义旋转器消息 |
| `once` | `boolean` | 运行一次后移除 |

#### `AgentHookSchema` (`type: 'agent'`)
使用验证提示运行代理验证器。

| 字段 | 类型 | 描述 |
|-------|------|-------------|
| `prompt` | `string` | 验证提示（例如"验证单元测试已运行"） |
| `model` | `string` | 要使用的模型（默认为 Haiku） |
| `timeout` | `number` | 超时秒数（默认 60） |
| `statusMessage` | `string` | 自定义旋转器消息 |
| `once` | `boolean` | 运行一次后移除 |

### 共享字段

#### `IfConditionSchema`
所有 hook 类型上的可选 `if` 字段 — 权限规则语法，用于过滤 hook 运行时机（例如 `"Bash(git *)"`）。使用延迟 schema 以避免循环导入。

### 复合 Schemas

- `HookCommandSchema`: 所有四种 hook 命令类型的区分联合
- `HookMatcherSchema`: `{ matcher?: string, hooks: HookCommand[] }` — 匹配事件并运行关联的 hooks
- `HooksSchema`: `Partial<Record<HookEvent, HookMatcher[]>>` — 按事件键控的顶级 hooks 配置

### 推断类型

- `HookCommand`: 所有 hook 命令类型的联合
- `BashCommandHook`、`PromptHook`、`AgentHook`、`HttpHook`: 提取的变体
- `HookMatcher`: 匹配器配置类型
- `HooksSettings`: `Partial<Record<HookEvent, HookMatcher[]>>` — 运行时设置类型

## 依赖

### 内部依赖

- `entrypoints/agentSdkTypes.js` — `HOOK_EVENTS`、`HookEvent` 枚举
- `utils/lazySchema.js` — 延迟 schema 求值以打破循环依赖
- `utils/shell/shellProvider.js` — 用于 shell 枚举的 `SHELL_TYPES`

### 外部依赖

- `zod/v4` — Schema 验证

## 实现细节

### 延迟 Schema 模式

所有 schema 使用 `lazySchema(() => ...)` 延迟求值。这打破了循环依赖，因为：
1. `settings/types.ts` 从 `schemas/hooks.ts` 导入
2. `plugins/schemas.ts` 从 `schemas/hooks.ts` 导入
3. 没有延迟求值，这些在模块加载时形成循环

### AgentHook 转换警告

`AgentHookSchema.prompt` 字段故意不使用 `.transform()` 将字符串包装在函数中。之前的转换将字符串包装在 `(_msgs) => prompt` 中，用于程序化构造用例，但该用例已被重构。转换导致 `updateSettingsForSource` 在 JSON 往返期间静默丢弃用户来自 `settings.json` 的提示（gh-24920、CC-79）。

### 权限规则语法

`if` 条件使用权限规则语法（例如 `"Bash(git *)"`、`"Read(*.ts)"`）在生成前过滤 hooks。这避免为不匹配的命令运行 hooks。

## 相关模块

- [Hooks 系统](./hooks-system.md) — Hooks 根据这些 schemas 验证
- [类型](./types-system.md) — `types/hooks.ts` 中的 Hook 类型与这些 schemas 对齐
- [Vendor 模块](./vendor-modules.md) — 插件 schemas 引用 hook schemas
