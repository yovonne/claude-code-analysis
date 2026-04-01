# 插件系统

## 目的

插件系统为 Claude Code 提供模块化扩展架构。内置插件随 CLI 一起发布，用户可通过 `/plugin` UI 启用/禁用。它们可以提供多种组件，包括技能、钩子和 MCP 服务器。系统区分内置插件（随 CLI 一起发布）和市场插件（从外部来源下载）。

## 位置

- `restored-src/src/plugins/builtinPlugins.ts` — 内置插件注册表和管理
- `restored-src/src/plugins/bundled/index.ts` — 内置插件初始化脚手架

## 主要导出

### 函数（builtinPlugins.ts）

- `registerBuiltinPlugin(definition)`: 在内部映射中注册内置插件定义。在启动时从 `initBuiltinPlugins()` 调用。
- `isBuiltinPluginId(pluginId)`: 检查插件 ID 是否以 `@builtin` 后缀结尾。
- `getBuiltinPluginDefinition(name)`: 按名称检索特定插件定义用于 UI 显示，无需市场查找。
- `getBuiltinPlugins()`: 返回按用户设置和 `defaultEnabled` 回退分组的 `{ enabled: LoadedPlugin[], disabled: LoadedPlugin[] }`。`isAvailable() === false` 的插件完全省略。
- `getBuiltinPluginSkillCommands()`: 将启用的内置插件提取为 `Command[]` 对象。
- `clearBuiltinPlugins()`: 清除注册表用于测试隔离。

### 函数（bundled/index.ts）

- `initBuiltinPlugins()`: 在 CLI 启动时调用的初始化函数。当前为空 — 用于未来将可切换技能迁移到内置插件的脚手架。

### 常量

- `BUILTIN_MARKETPLACE_NAME`: `'builtin'` — 在插件 ID（`{name}@builtin`）和路径字段中使用的哨兵值。

### 类型（引用）

- `BuiltinPluginDefinition`: 带技能、钩子、MCP 服务器、可用性检查和默认启用状态的插件定义。
- `LoadedPlugin`: 带清单、启用状态、钩子配置和 MCP 服务器列表的运行时插件对象。

## 依赖

### 内部依赖

- `../skills/bundledSkills.ts` — 插件提供技能的类型 `BundledSkillDefinition`
- `../types/plugin.js` — `BuiltinPluginDefinition` 和 `LoadedPlugin` 类型
- `../utils/settings/settings.js` — 用户插件偏好的 `getSettings_DEPRECATED()`
- `../commands.js` — 技能命令转换的 `Command` 类型

## 实现细节

### 核心逻辑

内置插件系统作为写一次注册表运行，带运行时启用/禁用评估：

1. **注册阶段** — 在启动时，`initBuiltinPlugins()` 为每个内置插件调用 `registerBuiltinPlugin()`。当前没有注册插件（空脚手架）。

2. **解析阶段** — `getBuiltinPlugins()` 迭代注册表，评估 `isAvailable()` 检查，从设置中读取用户偏好，并将每个插件分类为启用或禁用。

3. **技能提取** — `getBuiltinPluginSkillCommands()` 将启用的插件技能定义转换为与命令系统兼容的 `Command` 对象。

### 插件 ID 约定

内置插件使用 `{name}@builtin` 格式来区分市场插件（`{name}@{marketplace}`）。`@builtin` 后缀由 `isBuiltinPluginId()` 检查，并在 `LoadedPlugin.path` 字段中用作哨兵（因为内置插件没有文件系统路径）。

### 启用状态解析

```
用户设置存在？
    ├── 是 → 使用用户设置（true/false）
    └── 否 → 使用 definition.defaultEnabled ?? true
```

默认为 `true` — 内置插件默认启用，除非用户明确禁用或插件指定 `defaultEnabled: false`。

### 可用性检查

插件可以定义返回 `false` 的 `isAvailable()` 回调来完全隐藏插件（不仅仅是禁用）。这在启用状态解析之前检查，因此不可用的插件不会出现在启用或禁用列表中。

### 技能到命令的转换

`skillDefinitionToCommand()` 将 `BundledSkillDefinition` 映射到 `Command` 对象：

- `source` 和 `loadedFrom` 设置为 `'bundled'`（不是 `'builtin'`）以保持技能在 Skill 工具列表和分析中可见
- `isBuiltin` 跟踪在 `LoadedPlugin.isBuiltin` 上
- `isEnabled` 默认为 `() => true`
- `isHidden` 是 `userInvocable` 的反义
- `contentLength` 为 0（不适用于捆绑技能）

## 数据流

```
CLI 启动
    ↓
initBuiltinPlugins()
    ↓ registerBuiltinPlugin() 为每个插件
BUILTIN_PLUGINS Map 填充
    ↓
getBuiltinPlugins()
    ↓
├── 检查 isAvailable() → 如果为 false 则跳过
├── 读取用户设置 → enabledPlugins[pluginId]
├── 解析启用状态（用户 > 默认 > true）
└── 分类为 enabled[] 或 disabled[]
    ↓
getBuiltinPluginSkillCommands()
    ↓
├── 仅过滤启用的插件
├── 从每个插件提取技能
└── 通过 skillDefinitionToCommand() 转换为 Command[]
    ↓
命令通过 Skill 工具供模型使用
```

## 集成点

- **命令系统** — 插件技能成为斜杠命令（`/skill-name`）
- **设置系统** — 用户偏好存储在 `settings.enabledPlugins[pluginId]`
- **插件 UI** — `/plugin` 命令在 "内置" 部分显示内置插件
- **技能工具** — 插件技能出现在模型的可用工具列表中
- **MCP 系统** — 插件可以提供 MCP 服务器配置
- **钩子系统** — 插件可以定义钩子配置

## 配置

| 设置路径 | 类型 | 用途 |
|---|---|---|
| `settings.enabledPlugins['{name}@builtin']` | `boolean \| undefined` | 用户对内置插件的启用/禁用偏好 |

| 插件定义字段 | 类型 | 默认值 | 用途 |
|---|---|---|---|
| `name` | `string` | — | 插件标识符 |
| `description` | `string` | — | 用户面向描述 |
| `version` | `string` | — | 插件版本 |
| `defaultEnabled` | `boolean` | `true` | 默认是否启用 |
| `isAvailable` | `() => boolean` | — | 运行时可用性检查 |
| `skills` | `BundledSkillDefinition[]` | — | 插件提供的技能 |
| `hooks` | `HooksSettings` | — | 钩子配置 |
| `mcpServers` | `MCPServer[]` | — | MCP 服务器配置 |

## 错误处理

- 缺少设置：`getSettings_DEPRECATED()` 可能返回 undefined；通过可选链优雅处理
- 无效插件定义：注册表使用 Map 进行 O(1) 查找；缺少的名称返回 `undefined`
- 测试隔离：`clearBuiltinPlugins()` 在测试运行之间重置注册表

## 测试

- `clearBuiltinPlugins()` 为注册表提供测试隔离
- `getBuiltinPluginDefinition()` 允许直接访问以对单个插件进行单元测试
- 启用/禁用拆分基于设置是确定性的，使用模拟设置进行测试 straightforward

## 相关模块

- [Skills System](./skills-system.md) — 插件作为组件之一提供技能
- [Plugins Service](../04-services/plugins-service.md) — 市场插件管理的服务层
- [Settings Utils](../05-utils/settings-utils.md) — 插件偏好存储
- [Command System](../01-core-modules/command-system.md) — 插件技能成为命令

## 备注

- 内置插件系统当前是脚手架 — `initBuiltinPlugins()` 为空，没有注册插件
- 架构设计用于未来迁移应该可由用户切换的捆绑技能
- 内置插件与捆绑技能不同，它们出现在 `/plugin` UI 中并且可以由用户开启/关闭
- `'bundled'`（source/loadedFrom）和 `'builtin'`（LoadedPlugin.isBuiltin）之间的区别是有意的：`'bundled'` 保持技能在工具列表中可见，而 `'builtin'` 跟踪用户可切换的方面
- 插件 ID 使用 `@builtin` 作为市场名称，区别于实际市场来源如 `claude-code-marketplace` 或 `claude-plugins-official`
