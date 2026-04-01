# 快捷键系统

## 目的

快捷键系统为 Claude Code 的终端 UI 提供了可配置的、上下文感知的键盘快捷键管理层。它支持多键和弦序列、特定平台显示、用户通过 JSON 配置覆盖、保留快捷键验证，以及用于在组件中声明式绑定的 React hooks。

## 位置

`restored-src/src/keybindings/`

## 关键导出

### 核心模块

| 模块 | 描述 |
|--------|-------------|
| `parser.ts` | 将击键/和弦字符串解析为结构化 `ParsedKeystroke` 对象 |
| `schema.ts` | `keybindings.json` 配置的 Zod 验证 schema |
| `resolver.ts` | 使用和弦状态管理将键输入解析为操作 |
| `match.ts` | Ink 的 Key 与解析绑定之间的修饰符匹配逻辑 |
| `defaultBindings.ts` | 所有上下文的默认快捷键定义 |
| `reservedShortcuts.ts` | 保留/不可重新绑定的快捷键定义 |
| `template.ts` | 快捷键配置模板生成 |
| `validate.ts` | 用户快捷键验证逻辑 |
| `shortcutFormat.ts` | 快捷键显示格式化 |

### React 集成

| 模块 | 描述 |
|--------|-------------|
| `KeybindingContext.tsx` | 提供解析、处理程序和状态的 React 上下文 |
| `KeybindingProviderSetup.tsx` | 带绑定加载和状态管理的提供者设置 |
| `useKeybinding.ts` | 用于注册单个操作处理程序的 hook |
| `useShortcutDisplay.ts` | 用于显示操作当前快捷键的 hook |

### 用户绑定

| 模块 | 描述 |
|--------|-------------|
| `loadUserBindings.ts` | 加载用户 `keybindings.json` 并与默认值合并 |

## 依赖

### 内部依赖

- `../ink.js` — Ink 框架（Key 类型、useInput）
- `../ink/events/input-event.js` — InputEvent 类型
- `../utils/platform.js` — 平台检测（macos、windows、linux）
- `../utils/bundledMode.js` — Bun vs Node 检测
- `../utils/semver.js` — VT 模式支持的语义版本比较
- `../utils/lazySchema.js` — 惰性 Zod schema 求值 |
- `../utils/envUtils.js` — 环境变量工具
- `../hooks/` — 共享 hooks

### 外部依赖

- `react` — React 19
- `zod/v4` — Schema 验证
- `bun:bundle` — Bun 功能标志系统

## 实现细节

### 解析 (`parser.ts`)

将字符串表示转换为结构化数据：

- **`parseKeystroke(input)`**: 将 `"ctrl+shift+k"` 解析为 `{key, ctrl, alt, shift, meta, super}`
- **`parseChord(input)`**: 将 `"ctrl+k ctrl+s"` 解析为 `ParsedKeystroke` 数组
- **修饰符别名**: `ctrl`/`control`、`alt`/`opt`/`option`/`meta`、`cmd`/`command`/`super`/`win`
- **键别名**: `esc`→`escape`、`return`→`enter`、`space`→` `、箭头（`↑`、`↓`、`←`、`→`）
- **`keystrokeToString(ks)`**: 规范字符串表示
- **`chordToString(chord)`**: 用空格连接击键
- **平台显示**: `keystrokeToDisplayString()` 在 macOS 上使用 `"opt"` 表示 alt，其他地方使用 `"alt"`
- **`parseBindings(blocks)`**: 将 JSON 配置块解析为平面 `ParsedBinding[]` 数组

### Schema (`schema.ts`)

`keybindings.json` 验证的 Zod schema：

- **上下文**: 20+ 个有效上下文（`Global`、`Chat`、`Autocomplete`、`Confirmation`、`Help`、`Transcript`、`HistorySearch`、`Task`、`ThemePicker`、`Settings`、`Tabs`、`Attachments`、`Footer`、`MessageSelector`、`DiffDialog`、`ModelPicker`、`Select`、`Plugin`）
- **操作**: 80+ 个按域组织的有效操作标识符（`app:*`、`chat:*`、`confirm:*`、`tabs:*`、`history:*`、`autocomplete:*`、`select:*`、`plugin:*` 等）
- **绑定值**: 操作字符串、`command:*` 命令绑定，或 `null` 取消绑定
- **元数据**: 可选的 `$schema` 和 `$docs` 字段用于编辑器验证

### 解析 (`resolver.ts`)

使用和弦支持将键输入解析为操作：

- **`resolveKey(input, key, activeContexts, bindings)`**: 单键解析（用户覆盖的最后一个绑定获胜）
- **`resolveKeyWithChordState(input, key, activeContexts, bindings, pendingChord)`**: 和弦感知解析
- **结果类型**: `match`（找到操作）、`none`（无绑定）、`unbound`（显式为 null）、`chord_started`（和弦的第一个击键）、`chord_cancelled`（无效延续）
- **`getBindingDisplayText(action, context, bindings)`**: 获取操作的显示文本（反向搜索用户覆盖优先级）
- **`buildParsedKeystroke(input, key)`**: 从 Ink 的输入/键构建 `ParsedKeystroke`

### 匹配 (`match.ts`)

Ink 与解析绑定之间的修饰符匹配：

- **`getKeyName(input, key)`**: 从 Ink 的 Key 对象提取规范化键名（将布尔标志如 `key.escape`、`key.return` 映射为字符串名称）
- **`modifiersMatch(inkMods, target)`**: 检查修饰符相等性
- **Alt/Meta 别名**: Ink 为 Alt/Option 设置 `key.meta` — 当 `key.meta` 为 true 时，配置中的 `alt` 和 `meta` 都匹配
- **Super (Cmd/Win)**: 与 alt/meta 不同；仅通过 kitty 键盘协议到达
- **`matchesBinding(input, key, binding)`**: 完整绑定匹配检查
- **滚轮支持**: 滚轮事件的 `wheelup`/`wheeldown` 键名

### 默认绑定 (`defaultBindings.ts`)

特定平台的默认快捷键：

- **全局**: `ctrl+c`（中断）、`ctrl+d`（退出）、`ctrl+l`（重绘）、`ctrl+t`（切换待办事项）、`ctrl+o`（切换成绩单）、`ctrl+r`（历史搜索）
- **聊天**: `escape`（取消）、`enter`（提交）、`up`/`down`（历史）、`ctrl+x ctrl+k`（终止代理）、模式循环键
- **平台差异**: 图像粘贴在 Windows 上使用 `alt+v`，其他地方使用 `ctrl+v`；模式循环根据 VT 模式支持使用 `shift+tab` 或 `meta+m`
- **VT 模式检测**: 检查 Node.js >= 22.17.0/<23.0.0 或 >= 24.2.0，或 Bun >= 1.2.23 用于 Windows VT 模式支持
- **功能门控绑定**: 快速搜索（`ctrl+shift+f`）、终端面板（`meta+j`）、简要切换（`ctrl+shift+b`）

### 保留快捷键 (`reservedShortcuts.ts`)

定义不能或不应重新绑定的快捷键：

- **不可重新绑定**: `ctrl+c`、`ctrl+d`（硬编码中断/退出）、`ctrl+m`（在终端中与 Enter 相同）
- **终端保留**: `ctrl+z`（SIGTSTP）、`ctrl+\`（SIGQUIT）
- **macOS 保留**: `cmd+c/v/x/q/w/tab/space`（系统级拦截）
- **严重级别**: `error`（永远不会工作）vs `warning`（可能被拦截）
- **`getReservedShortcuts()`**: 返回平台适当的保留列表
- **`getNonRebindableShortcuts()`**: 返回不能重新绑定的快捷键
- **`isReservedShortcut(chord)`**: 检查和弦是否保留
- **`getReservedWarning(chord)`**: 获取保留快捷键的警告消息

### React 上下文 (`KeybindingContext.tsx`)

提供快捷键解析和处理程序管理：

- **`KeybindingContextValue`**: `{resolve, setPendingChord, getDisplayText, bindings, pendingChord, activeContexts, registerActiveContext, unregisterActiveContext, registerHandler, invokeAction}`
- **处理程序注册表**: `Map<string, Set<HandlerRegistration>>` 将操作名称映射到处理程序集
- **待处理和弦 ref**: 用于即时访问的 `RefObject`（避免热路径中的 React 状态延迟）
- **活动上下文**: `Set<KeybindingContextName>` 跟踪当前活动的上下文
- **`KeybindingProvider`**: 用上下文值包装子组件，带记忆化的 resolve/getDisplayText 函数

### useKeybinding Hook (`useKeybinding.ts`)

声明式快捷键注册：

- **单一绑定**: `useKeybinding(action, handler, options?)`
- **多个绑定**: `useKeybindings(bindingsMap, options?)`
- **选项**: `context`（默认 `'Global'`）、`isActive`（启用/禁用）
- **和弦支持**: 自动管理待处理和弦状态
- **stopImmediatePropagation**: 一旦绑定被处理，防止其他处理程序触发
- **处理程序返回**: 返回 `false` 防止 `stopImmediatePropagation`（允许其他处理程序）
- **上下文构建**: 检查注册的活动上下文 + 绑定上下文 + 全局（去重，首个出现优先）

### useShortcutDisplay Hook (`useShortcutDisplay.ts`)

获取操作的显示文本：

- **`useShortcutDisplay(action, context?)`**: 返回格式化的快捷键字符串（例如 `"ctrl+t"`）
- **平台感知**: 使用平台适当的修饰符名称
- **响应式**: 绑定变化时更新

### 用户绑定加载 (`loadUserBindings.ts`)

加载和合并用户配置：

- **首先加载默认绑定**，然后用户绑定覆盖
- **验证**: 用户绑定根据 schema 验证
- **错误处理**: 无效绑定报告具体错误消息
- **合并策略**: 用户绑定追加到默认值之后（最后胜出解析）

## 数据流

```
keybindings.json (用户配置)
    ↓
loadUserBindings.ts (加载 + 验证 + 与默认值合并)
    ↓
ParsedBinding[] (所有绑定的平面数组)
    ↓
KeybindingProviderSetup.tsx (创建带绑定 + 状态的上下文)
    ↓
KeybindingContext.tsx (提供 resolve、处理程序、待处理和弦)
    ↓
useKeybinding.ts (组件注册处理程序)
    ↓
useInput (Ink hook 捕获键盘输入)
    ↓
resolveKeyWithChordState (解析输入 → 操作)
    ↓
invokeAction (调用注册的处理程序)
    ↓
stopImmediatePropagation (防止进一步处理)
```

## 上下文优先级解析

按优先级顺序检查上下文：
1. 注册的活动上下文（组件声明的，例如输入聚焦时的 "Chat"）
2. 绑定的声明上下文
3. `Global`（始终最后检查）

更具体的上下文优先。在相同上下文内，后绑定优先（用户覆盖默认值）。

## 关键设计决策

1. **最后胜出解析**: 用户绑定追加到默认值之后，因此用户配置始终获胜
2. **ref 中的和弦状态**: 待处理和弦存储在 `RefObject` 中，以便在热路径中访问而无需 React 状态延迟
3. **平台感知显示**: 修饰符名称适应平台（`opt` vs `alt`、`cmd` vs `super`）
4. **保留快捷键验证**: 防止用户绑定永远不会工作的快捷键
5. **处理程序返回值**: 从处理程序返回 `false` 防止 `stopImmediatePropagation`，允许其他处理程序
6. **上下文注册**: 组件在挂载/卸载时注册/取消注册上下文，以实现准确的优先级解析
7. **Zod schema 验证**: 用户配置使用描述性错误验证
8. **命令绑定**: `command:*` 语法允许将键绑定到斜杠命令
