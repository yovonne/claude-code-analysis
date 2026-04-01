# 设计系统

## 目的

设计系统提供主题感知的 UI 原语，构成了 Claude Code 终端界面的基础。它将原始 Ink 组件与应用程序的主题系统连接起来，通过主题键解析而非硬编码颜色实现所有 UI 组件的一致样式。

## 位置

`restored-src/src/components/design-system/`

## 关键导出

### 核心组件

| 组件 | 描述 |
|-----------|-------------|
| `ThemedBox` | 主题感知 Box；解析主题颜色键（borderColor、backgroundColor）为原始颜色 |
| `ThemedText` | 主题感知 Text；解析 color/backgroundColor 主题键，支持悬停颜色上下文 |
| `ThemeProvider` | 提供主题上下文，带自动/系统主题检测、预览和持久化 |
| `Dialog` | 模态对话框，带标题、副标题、边框窗格、输入指南和取消快捷键 |
| `FuzzyPicker<T>` | 通用模糊搜索选择器，带过滤、预览、键盘导航和操作提示 |
| `Tabs` / `Tab` | 带键盘导航的标签页界面，支持受控/非受控模式和内容高度管理 |
| `Pane` | 带可选颜色主题的带边框容器 |
| `ListItem` | 带焦点状态的可选择列表项 |
| `Byline` | 键盘快捷方式的紧凑提示/操作行 |
| `KeyboardShortcutHint` | 显示键盘快捷方式及操作标签 |
| `ProgressBar` | Unicode 块字符进度条，支持子字符精度 |
| `Ratchet` | 带增量步骤的进度指示器 |
| `StatusIcon` | 状态指示器图标 |
| `LoadingState` | 加载状态显示 |
| `Divider` | 水平分割线 |

### 工具函数

| 导出 | 描述 |
|--------|-------------|
| `color(c, theme, type?)` | 柯里化主题感知颜色函数；解析主题键为原始颜色 |
| `TextHoverColorContext` | 用于覆盖子树悬停文本颜色的 React 上下文 |

## 依赖

### 内部依赖

- `../../ink.js` — Ink 框架（Box、Text、useTheme、useInput、useTerminalFocus 等）
- `../../ink/styles.js` — Ink 样式类型
- `../../ink/dom.js` — DOMElement 类型
- `../../ink/events/` — 事件类型（ClickEvent、FocusEvent、KeyboardEvent）
- `../../utils/theme.js` — 主题定义和 `getTheme()`
- `../../utils/systemTheme.js` — 系统主题检测
- `../../utils/config.js` — 配置持久化（`getGlobalConfig`、`saveGlobalConfig`）
- `../../ink/hooks/use-stdin.js` — 用于终端查询器访问的 useStdin
- `../../hooks/` — 共享 hooks（useExitOnCtrlCDWithKeybindings、useSearchInput、useTerminalSize）
- `../../keybindings/useKeybinding.js` — useKeybinding、useKeybindings
- `../../context/modalContext.js` — 模态上下文（useIsInsideModal、useModalScrollRef）

### 外部依赖

- `react` — React 19
- `bun:bundle` — Bun 功能标志系统
- `zod/v4` — Schema 验证（用于 FuzzyPicker）

## 实现细节

### ThemeProvider

管理应用程序的主题状态，具有以下功能：

- **主题设置**: `'dark' | 'light' | 'auto'`
- **自动主题**: 当 `AUTO_THEME` 功能启用时，通过 OSC 11 轮询监视终端主题变化
- **预览模式**: 临时主题预览（由 ThemePicker 使用），支持保存/取消
- **持久化**: 主题变化时保存到全局配置
- **系统主题种子**: 从 `$COLORFGBG` 环境变量种子初始化，由 OSC 11 监视器校正
- **上下文值**: `{themeSetting, setThemeSetting, setPreviewTheme, savePreview, cancelPreview, currentTheme}`

### ThemedBox

使用主题解析包装基础 `Box` 组件：

- **颜色属性**: `borderColor`、`borderTopColor`、`borderBottomColor`、`borderLeftColor`、`borderRightColor`、`backgroundColor`
- **解析**: 检查值是否为原始颜色（以 `rgb(`、`#`、`ansi256(`、`ansi:` 开头）—— 如果是，则直接传递；否则在主题中查找
- **事件支持**: 完整的事件处理器支持（onClick、onFocus、onBlur、onKeyDown、onMouseEnter、onMouseLeave）
- **布局属性**: 所有 Box 布局属性（flexWrap、flexDirection、flexGrow、flexShrink、overflow 等）

### ThemedText

使用主题解析包装基础 `Text` 组件：

- **颜色属性**: `color` 和 `backgroundColor` 接受主题键
- **悬停颜色上下文**: `TextHoverColorContext` 允许父组件设置悬停颜色，该颜色传播到子树中未着色的 ThemedText
- **优先级**: 显式 `color` > 悬停颜色 > dimColor（使用主题的 inactive 颜色）
- **样式属性**: `bold`、`italic`、`underline`、`strikethrough`、`inverse`、`dimColor`、`wrap`

### Dialog

可复用的对话框组件，具有：

- **结构**: 标题（粗体、主题化）→ 副标题（暗淡）→ 内容（gap=1）→ 输入指南
- **边框**: 通过 `Pane` 组件的可选着色边框
- **取消处理**: 内置 `confirm:no` 快捷键（Esc/n）和 app:exit/interrupt（Ctrl+C/D）
- **可配置取消**: `isCancelActive` 属性用于嵌入文本字段聚焦时禁用取消快捷键
- **自定义输入指南**: `inputGuide` 渲染属性接收 `exitState` 用于显示 Ctrl+C/D 待处理状态
- **颜色**: 主题化边框颜色（默认: `'permission'`）

### FuzzyPicker

通用模糊搜索选择器组件：

- **泛型类型**: `<T>` 适用于任何项类型
- **功能**: 搜索输入、过滤列表、键盘导航（上下/Enter）、预览面板、操作提示（Tab/Shift+Tab）
- **布局**: 标题 → SearchBox → 列表 → 预览 → 提示
- **方向**: `'down'`（默认）或 `'up'`（atuin 风格，items[0] 在底部）
- **可见性**: 根据终端高度自适应可见数量（最小 2，默认 8，chrome 占 10 行）
- **操作**: 主要（Enter）、次要（Tab）、第三（Shift+Tab），具有可配置的处理程序和提示
- **焦点跟踪**: `onFocus` 回调用于异步预览加载
- **空状态**: 可自定义的空消息
- **匹配标签**: 显示匹配计数的状态行

### Tabs

带键盘导航的标签页界面：

- **结构**: 标签头部行 → 可选横幅 → 标签内容
- **导航**: 左右箭头或 Tab/Shift+Tab 切换标签
- **模式**: 受控（`selectedTab` + `onTabChange`）或非受控（`defaultTab`）
- **内容高度**: 固定 `contentHeight` 防止标签间布局偏移（overflow hidden）
- **头部焦点**: 头部可以聚焦/失焦；内容可以选择退出导航键并返回头部
- **来自内容的导航**: `navFromContent` 允许从聚焦内容使用 Tab/箭头键切换标签
- **上下文**: `TabsContext` 提供 `selectedTab`、`width`、`headerFocused`、`focusHeader`、`blurHeader`、`registerOptIn`
- **Hooks**: `useTabsWidth()` 用于内容宽度，`useTabHeaderFocus()` 用于头部焦点门控

### ProgressBar

Unicode 块字符进度条：

- **字符**: 使用 9 个块字符（` `、`▏`、`▎`、`▍`、`▌`、`▋`、`▊`、`▉`、`█`）实现子字符精度
- **属性**: `ratio`（0-1）、`width`（字符宽度）、`fillColor`、`emptyColor`
- **渲染**: 填充部分使用主题颜色，空部分使用背景主题颜色
- **精度**: 计算完整块 + 部分块 + 空块

## 数据流

```
ThemeProvider (提供 currentTheme)
    ↓
ThemedBox / ThemedText (解析主题键 → 原始颜色)
    ↓
Ink Box / Text (使用解析的颜色渲染)
    ↓
终端输出 (ANSI 转义序列)
```

## 主题解析

颜色值遵循此解析链：

1. **原始颜色检测**: 如果值以 `rgb(`、`#`、`ansi256(` 或 `ansi:` 开头——原样使用
2. **主题键查找**: 否则，在 `getTheme(themeName)[key]` 中查找值
3. **回退**: 未定义的颜色被传递（Ink 处理默认值）

## 关键设计决策

1. **主题键优于原始颜色**: 组件接受主题键（`'primary'`、`'permission'` 等）以实现一致的主题化
2. **预览模式**: 主题变化可以在提交前预览，实现流畅的主题选择器 UX
3. **自动主题检测**: 通过 OSC 11 监视终端主题变化，实现无缝的深色/浅色切换
4. **悬停颜色上下文**: 跨组件悬停着色，无需属性穿透
5. **渐进宽度门控**: FuzzyPicker 和 Tabs 适应可用的终端宽度
6. **React Compiler**: 所有组件使用 `react/compiler-runtime` 实现自动记忆化
7. **受控/非受控**: Tabs 和其他组件支持两种模式以提高灵活性
