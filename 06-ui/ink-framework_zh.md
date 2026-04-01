# Ink 框架

## 目的

Ink 框架是一个基于 React 的自定义终端 UI 渲染引擎，为 Claude Code 的全屏替代屏幕界面提供支持。它提供用于终端输出的 React 协调器、基于 Yoga 的 flexbox 布局系统、键盘/鼠标输入的事件分发器，以及用于无闪烁渲染的双缓冲屏幕差异引擎。

## 位置

`restored-src/src/ink/` — 整个 `ink/` 目录加上 `restored-src/src/ink.ts` 处的重导出桶。

## 关键导出（来自 `ink.ts`）

### 渲染 API

| 导出 | 描述 |
|--------|-------------|
| `render(node, options?)` | 异步渲染函数，在根组件外包装 `ThemeProvider` 并返回 `Instance` |
| `createRoot(options?)` | 创建可复用的 Ink 根（类似于 `react-dom` 的 `createRoot`）用于顺序屏幕 |
| `Root` | 带 `render()`、`unmount()`、`waitUntilExit()` 的托管根类型 |
| `Instance` | 带 `rerender()`、`unmount()`、`waitUntilExit()`、`cleanup()` 的渲染实例 |
| `RenderOptions` | 配置: `stdout`、`stdin`、`stderr`、`exitOnCtrlC`、`patchConsole`、`onFrame` |

### 核心组件

| 导出 | 描述 |
|--------|-------------|
| `Box` | Flexbox 布局容器（映射到 `ink-box`） |
| `Text` | 样式文本组件（映射到 `ink-text`） |
| `BaseBox` / `BaseText` | 非主题化的基础变体 |
| `Button` | 带焦点/悬停/激活状态渲染属性的交互式按钮 |
| `Link` | 带 OSC 8 支持的超链接组件和回退 |
| `Spacer` | 灵活空间（`flexGrow={1}`） |
| `Newline` | 插入换行符 |
| `Ansi` | 将 ANSI 转义序列解析为样式化 React 元素 |
| `RawAnsi` | 绕过 ANSI 解析往返用于预渲染输出 |
| `NoSelect` | 标记内容在全屏文本选择中不可选 |
| `ScrollBox` | 带命令式滚动 API 的溢出滚动容器 |
| `AlternateScreen` | 进入终端替代屏幕缓冲区（DEC 1049） |

### Hooks

| 导出 | 描述 |
|--------|-------------|
| `useInput(handler, options?)` | 捕获键盘输入；启用原始模式 |
| `useApp()` | 返回 `exit()` 以卸载应用 |
| `useStdin()` | 访问 stdin 流和原始模式控制 |
| `useAnimationFrame(intervalMs?)` | 共享时钟动画 hook；当屏幕外时暂停 |
| `useInterval(callback, intervalMs)` | 由共享时钟支持的间隔 hook |
| `useAnimationTimer(intervalMs)` | 返回时钟时间用于纯时间计算 |
| `useSelection()` | 全屏文本选择操作（复制、清除、Shift、捕获） |
| `useHasSelection()` | 选择存在状态的响应式布尔值 |
| `useTerminalFocus()` | 终端窗口是否有焦点（DECSET 1004） |
| `useTerminalTitle(title)` | 声明式设置终端标签/窗口标题（OSC 0） |
| `useTerminalViewport()` | 检测元素是否在可见视口内 |
| `useTabStatus(kind)` | 设置标签状态指示器（OSC 21337） |
| `measureElement(node)` | 返回 Box 元素的 `{width, height}` |

### 事件和焦点

| 导出 | 描述 |
|--------|-------------|
| `FocusManager` | 带标签循环和焦点栈的类 DOM 焦点管理器 |
| `EventEmitter` | 用于输入、焦点和终端事件的事件系统 |
| `InputEvent` | 带解析键信息的键盘输入事件 |
| `ClickEvent` | 鼠标点击事件（仅替代屏幕） |
| `TerminalFocusEvent` | 终端窗口焦点/失焦事件 |
| `Key` | 带修饰符标志（ctrl、shift、meta、super）的键对象 |

### 主题系统

| 导出 | 描述 |
|--------|-------------|
| `ThemeProvider` | 提供带自动/系统主题检测的主题上下文 |
| `useTheme()` | 返回 `[currentTheme, setThemeSetting]` |
| `useThemeSetting()` | 返回原始存储设置（可能是 `'auto'`） |
| `usePreviewTheme()` | 返回 `{setPreviewTheme, savePreview, cancelPreview}` |
| `color(c, theme, type?)` | 柯里化主题感知颜色函数 |

## 依赖

### 内部依赖

- `src/utils/` — 调试日志、配置、系统主题检测、环境变量工具
- `src/native-ts/yoga-layout/` — 基于 WASM 的 Yoga 布局引擎
- `src/bootstrap/state.js` — 后台间隔抑制的滚动活动标记

### 外部依赖

- `react` / `react-reconciler` — 带自定义主机配置的 React 19
- `wrap-ansi` — ANSI 感知文本包装（带 Bun 原生回退）
- `strip-ansi` — ANSI 转义序列移除
- `@alcalzone/ansi-tokenize` — 用于样式差异的 ANSI 标记化
- `usehooks-ts` — 用于稳定处理程序引用的 `useEventCallback`

## 实现细节

### 架构概述

Ink 框架遵循分层架构：

```
React 组件 (Box, Text, Button 等)
    ↓
React 协调器 (reconciler.ts 中的自定义主机配置)
    ↓
DOM 树 (DOMElement, TextNode — 终端的虚拟 DOM)
    ↓
Yoga 布局引擎 (WASM flexbox 布局)
    ↓
渲染器 (renderer.ts — DOM → 屏幕缓冲区)
    ↓
输出 (output.ts — 屏幕缓冲区操作)
    ↓
屏幕缓冲区 (screen.ts — 池化单元格数组)
    ↓
LogUpdate (log-update.ts — 差异和补丁生成)
    ↓
终端 (terminal.ts — ANSI 序列发送)
```

### React 协调器 (`reconciler.ts`)

实现自定义 React 协调器主机配置，将 React 元素映射到终端 DOM 节点：

- **节点类型**: `ink-root`、`ink-box`、`ink-text`、`ink-virtual-text`、`ink-link`、`ink-progress`、`ink-raw-ansi`
- **样式应用**: React `style` 属性通过 `applyStyles()` 转换为 Yoga 布局属性
- **文本处理**: 文本节点必须在 `<Text>` 组件内；裸露文本会抛出错误
- **脏跟踪**: 节点在属性/样式变化时标记为脏；祖先向上传播脏状态
- **Yoga 清理**: 释放节点前清除其 WASM 引用，然后调用 `freeRecursive()` 以防止悬空指针
- **事件处理程序**: 存储在 `_eventHandlers` 中（与属性分开），以便处理程序标识变化不会破坏 blit 优化
- **调试重绘**: 设置 `CLAUDE_CODE_DEBUG_REPAINTS` 时，捕获 React 组件所有者链以进行闪烁归因

### DOM 模型 (`dom.ts`)

协调器操作的虚拟 DOM 节点：

- **`DOMElement`**: 带有 `childNodes`、`attributes`、`style`、`yogaNode`、滚动状态、焦点管理器和事件处理程序的容器节点
- **`TextNode`**: 带 `nodeValue` 的叶子文本节点
- **Yoga 集成**: 大多数元素获得 Yoga 布局节点；`ink-virtual-text`、`ink-link` 和 `ink-progress` 除外
- **测量函数**: `ink-text` 节点获得执行文本包装的测量函数；`ink-raw-ansi` 节点使用常量时间测量（宽度 × 高度属性）
- **滚动状态**: `scrollTop`、`pendingScrollDelta`、`scrollAnchor`、`stickyScroll`、`scrollClampMin/Max` 用于虚拟滚动
- **`markDirty()`**: 向上遍历祖先链，标记元素为脏并为文本重新测量标记 yoga 为脏
- **`scheduleRenderFrom()`**: 向上遍历到根并调用 `onRender()`，用于绕过 React 的 DOM 级突变

### 布局引擎 (`layout/`)

- **`engine.ts`**: WASM Yoga 的薄包装（`createYogaLayoutNode()`）
- **`node.ts`**: Yoga 类型定义（LayoutAlign、LayoutDisplay、LayoutFlexDirection 等）
- **`yoga.ts`**: WASM Yoga 绑定和布局节点创建
- **`geometry.ts`**: Point、Rectangle、Size 类型和工具函数（clamp、unionRect）

### 渲染管道 (`renderer.ts`、`output.ts`)

**双缓冲渲染:**
1. **前帧**: 当前显示的屏幕缓冲区
2. **后帧**: 为下一帧构建中
3. **渲染器**: 遍历 DOM 树 → Yoga 布局 → 填充输出操作 → 生成屏幕缓冲区
4. **LogUpdate**: 将新屏幕与前一帧进行差异比较 → 生成 Patch 数组 → 优化 → 写入终端

**屏幕缓冲区 (`screen.ts`):**
- **CharPool**: 驻留的字符字符串，带 ASCII 快速路径（Int32Array 直接查找）
- **StylePool**: 驻留的样式对象，带缓存的 SGR 转换
- **HyperlinkPool**: 驻留的超链接字符串
- **单元格模型**: 用于字符、样式、超链接、宽度和 noSelect 标志的压缩整数数组
- **代际重置**: 池每 5 分钟重置以防止无限制增长

**输出操作:**
- `write`: 在位置写入文本，带软换行标志
- `blit`: 从前一帧复制区域（O(不变) 快速路径）
- `clear`: 清除区域
- `clip`/`unclip`: 剪切输出到子区域
- `noSelect`: 标记单元格为不可选
- `shift`: 滚动优化的行移位

### 帧系统 (`frame.ts`)

- **`Frame`**: 包含 `screen`、`viewport`、`cursor`、`scrollHint`、`scrollDrainPending`
- **`FrameEvent`**: 每帧时序数据 — `durationMs`、阶段分解（渲染器、差异、优化、写入、yoga、提交）、闪烁原因
- **`Patch`**: 终端更新操作 — stdout 内容、清除、光标、样式、超链接
- **`shouldClearScreen()`**: 确定何时进行完整重置（调整大小、溢出）

### 焦点管理器 (`focus.ts`)

类 DOM 焦点系统：
- **`activeElement`**: 当前聚焦的元素
- **焦点栈**: 最多 32 个条目；焦点变化时将前一个元素推入
- **标签循环**: `focusNext()` / `focusPrevious()` 遍历可标签元素（tabIndex >= 0）
- **节点移除**: 当聚焦元素被移除时自动从栈恢复焦点
- **点击聚焦**: 只聚焦具有数字 `tabIndex` 的元素
- **`getFocusManager(node)`**: 向上遍历到根（类似于 `node.ownerDocument`）

### 事件系统 (`events/`)

- **`EventEmitter`**: 带优先级监听器排序的中央事件总线
- **`Dispatcher`**: 将协调器优先级系统与事件处理桥接
- **`InputEvent`**: 带解析 `Key`（修饰符、特殊键）的键盘输入
- **`ClickEvent`**: 带位置信息的鼠标点击（仅替代屏幕）
- **`FocusEvent`**: 带 relatedTarget 的焦点/失焦
- **`TerminalFocusEvent`**: 终端窗口焦点（DECSET 1004）
- **`keyboard-event.ts`**: 带捕获/冒泡阶段的键盘事件

### ANSI 解析器 (`termio/`)

语义 ANSI 转义序列解析器：
- **`Parser`**: 流式解析器，增量输入并生成结构化操作
- **操作类型**: 文本段、光标操作、擦除、滚动、标题、链接、模式变化
- **样式跟踪**: 在解析调用间维护 TextStyle 状态（fg、bg、bold、italic、underline 等）
- **序列支持**: SGR（颜色/样式）、CSI（光标/擦除）、OSC（标题/超链接/标签状态）、ESC
- **`termio.ts`**: 解析器、类型和工具函数的桶重导出

### 终端通知 (`useTerminalNotification.ts`)

跨终端通知支持：
- **iTerm2**: 通过 OSC 的 Growl 风格通知
- **Kitty**: 带 ID 的多部分通知
- **Ghostty**: 原生通知协议
- **Bell**: 用于 tmux 兼容性的原始 BEL
- **Progress**: OSC 9;4 进度报告（ConEmu、Ghostty 1.2+、iTerm2 3.6.6+）

### 文本包装 (`wrap-text.ts`、`wrapAnsi.ts`)

- **`wrap`**: 软换行的单词包装
- **`wrap-trim`**: 去除首尾空白的单词包装
- **`truncate-end`**: 末尾省略号
- **`truncate-middle`**: 中间省略号
- **`truncate-start`**: 开头省略号
- 使用 `wrap-ansi` npm 包，带 Bun 原生回退（`Bun.wrapAnsi`）

### 选择系统 (`selection.ts`)

全屏文本选择（918 行）：
- **`SelectionState`**: 锚点/焦点点、拖动标志、anchorSpan 用于单词/行模式、滚出的累加器、用于夹紧跟踪的虚拟行
- **单词检测**: 匹配 iTerm2 默认值的 Unicode 感知字符类
- **行选择**: 整行选择
- **拖动到滚动**: 累积上方/下方滚出的行及软换行位
- **键盘扩展**: Shift+箭头在锚点固定的情况下移动焦点
- **键盘滚动**: PgUp/PgDn/Home/End 使用虚拟行跟踪同时移动锚点和焦点
- **文本提取**: 尊重 noSelect 单元格、软换行连接、尾随空白修剪
- **选择覆盖**: 将纯背景颜色应用于所选单元格（替换旧的 SGR-7 反转）

### Ink 实例 (`ink.tsx`)

主要的 `Ink` 类编排整个渲染生命周期：
- **节流渲染**: `scheduleRender()` 合并快速更新
- **终端调整大小**: 处理 SIGWINCH，重新计算布局
- **替代屏幕管理**: 进入/退出替代屏幕缓冲区，管理光标
- **控制台修补**: 拦截 `console.log`/`console.error` 以防止与 Ink 输出混合
- **信号处理**: SIGINT（Ctrl+C）、SIGTERM、SIGHUP 清理
- **外部编辑器暂停**: 外部编辑器打开时暂停 Ink 渲染
- **帧时序**: 可选的 `onFrame` 回调，带阶段分解
- **双缓冲**: 带 blit 优化的前/后帧交换

## 数据流

```
用户输入 (stdin)
    ↓
Ink 实例 (解析按键，更新选择)
    ↓
EventEmitter → useInput / useKeybinding 处理程序
    ↓
React 状态更新
    ↓
协调器 (差异 React 树 → DOM 突变)
    ↓
resetAfterCommit → onComputeLayout (Yoga)
    ↓
onRender → scheduleRender (节流)
    ↓
渲染器 (DOM → Yoga → 输出操作 → 屏幕)
    ↓
LogUpdate (差异屏幕 → Patches → 优化 → 写入)
    ↓
终端 (ANSI 序列 → stdout)
```

## 关键设计决策

1. **双缓冲渲染**: 带 blit 优化的前/后帧交换，用于 O(不变) 稳态帧
2. **共享池**: CharPool、StylePool、HyperlinkPool 在帧间驻留值以最小化分配
3. **微任务延迟**: `queueMicrotask()` 将快速滚动突变合并为单一渲染
4. **Yoga WASM**: 布局由原生 WASM 模块计算以提高性能
5. **React Compiler**: 组件使用 `react/compiler-runtime` 实现自动记忆化
6. **代际池重置**: 池定期重置以防止无限制内存增长
7. **虚拟滚动**: ScrollBox 使用夹紧边界防止突发滚动期间的空白屏幕
8. **选择作为覆盖层**: 选择高亮直接应用于屏幕缓冲区单元格，而非作为单独的渲染通道
