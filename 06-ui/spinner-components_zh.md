# 旋转组件

## 目的

旋转组件系统为 Claude Code 的终端 UI 提供动画加载指示器。它包括多种动画样式（旋转字形、闪烁文本、微光效果、闪烁字符）、停滞检测及颜色转换、减少运动支持，以及队友特定的旋转器变体——所有这些都与共享动画时钟同步以提高性能。

## 位置

`restored-src/src/components/Spinner/`

## 关键导出

### 核心组件

| 组件 | 描述 |
|-----------|-------------|
| `SpinnerGlyph` | 带基于帧的动画、停滞检测和减少运动支持的主要旋转字符 |
| `GlimmerMessage` | 带闪光/微光动画效果的消息文本 |
| `ShimmerChar` | 带闪光颜色动画的单个字符 |
| `FlashingChar` | 在基色和闪光颜色之间进行基于不透明度闪烁动画的字符 |
| `SpinnerAnimationRow` | 组合旋转器、消息、计时器、token 和思考状态的完整动画状态行 |

### 队友组件

| 组件 | 描述 |
|-----------|-------------|
| `TeammateSpinnerLine` | 队友代理显示的旋转器行 |
| `TeammateSpinnerTree` | 用于群体可视化的队友旋转器树 |

### Hooks

| Hook | 描述 |
|------|-------------|
| `useShimmerAnimation(mode, message, isStalled)` | 计算闪光动画的微光索引；停滞时取消订阅时钟 |
| `useStalledAnimation(time, responseLength, hasActiveTools, reducedMotion)` | 检测 token 流停滞并计算到红色的平滑强度衰减 |

### 工具函数

| 函数 | 描述 |
|----------|-------------|
| `getDefaultCharacters()` | 返回平台适当的旋转器字符（Ghostty、macOS、Linux 变体） |
| `interpolateColor(color1, color2, t)` | 两个颜色之间的线性 RGB 插值 |
| `toRGBColor(color)` | 将 RGB 对象转换为用于 Text 组件的 `rgb(r,g,b)` 字符串 |
| `parseRGB(colorStr)` | 解析 `rgb(r,g,b)` 字符串为 RGB 对象（带缓存） |
| `hueToRgb(hue)` | HSL 色相到 RGB 转换（用于语音模式波形颜色） |

### 类型

| 类型 | 描述 |
|------|-------------|
| `SpinnerMode` | `'requesting' | 'tool-use' | 'tool-input' | 'responding' | 'thinking'` |

## 依赖

### 内部依赖

- `../../ink.js` — Ink 框架（Box、Text、useTheme、useAnimationFrame）
- `../../ink/stringWidth.js` — 字符串宽度计算
- `../../utils/theme.js` — 主题定义和 `getTheme()`
- `../../utils/format.js` — 持续时间和数字格式化
- `../../utils/ink.js` — `toInkColor()` 工具函数 |
- `../../tasks/InProcessTeammateTask/types.js` — 队友任务状态类型
- `../design-system/Byline.js` — 用于状态提示的 Byline 组件 |

### 外部依赖

- `react` — React 19
- `figures` — Unicode 符号（用于 token 计数的箭头）|

## 实现细节

### SpinnerGlyph

主要旋转字符组件：

- **帧动画**: 基于帧索引循环遍历 `SPINNER_FRAMES`（字符正向 + 反向）
- **平台字符**: Ghostty 使用 `['·', '✢', '✳', '✶', '✻', '*']`（避免偏移 `✽`），macOS 使用 `['·', '✢', '✳', '✶', '✻', '✽']`，Linux 使用 `['·', '✢', '*', '✶', '✻', '✽']`
- **减少运动**: 显示单个 `●` 点，在 2 秒周期内切换暗淡/可见
- **停滞检测**: 基于 `stalledIntensity`（0-1）将颜色从消息颜色插值到错误红色（`rgb(171,43,63)`）
- **回退**: 如果 RGB 解析失败，当停滞强度 > 0.5 时切换到 `'error'` 主题颜色

### GlimmerMessage

带移动闪光效果的消息文本：

- **微光索引**: 扫过消息文本的闪光高亮位置
- **速度**: `'requesting'` 模式为 50ms，其他模式为 200ms
- **闪烁不透明度**: 工具使用模式闪烁效果的 sin 波不透明度
- **停滞处理**: 停滞时微光索引设置为 -100（屏幕外）
- **颜色插值**: 消息颜色和闪光颜色之间的平滑 RGB 插值

### SpinnerAnimationRow

完整的动画状态行（265 行）：

- **动画隔离**: 拥有 `useAnimationFrame(50)` 订阅；父组件从 50ms 渲染循环中解放出来
- **渐进宽度门控**: 根据可用的终端宽度动态显示/隐藏思考、计时器和 token 计数
- **显示的组件**:
  - SpinnerGlyph（基于帧的旋转字符）
  - GlimmerMessage（动画消息文本）
  - Timer（经过时间，格式化持续时间）
  - Token 计数（带模式特定字形：工具使用/响应的向下箭头，请求的向上箭头）
  - 思考状态（带闪光颜色动画）
  - 队友状态（前景色的队友名称带颜色）
- **经过时间**: 尊重暂停状态的挂钟计算（`pauseStartTimeRef`、`totalPausedMsRef`）
- **Token 计数器动画**: 平滑增量动画（根据间隙大小每帧 3/8/50 token）
- **思考闪光**: 单独的 sin 波不透明度动画，3 秒延迟和 2 秒发光周期
- **状态组合**: 根据可用空间有条件地构建部分数组，用 Byline 或括号包装

### useShimmerAnimation

计算文本动画的闪光位置：

- **闪光速度**: `'requesting'` 模式为 50ms，其他模式为 200ms
- **时钟订阅**: 停滞时向 `useAnimationFrame` 传递 `null` — 取消订阅时钟以节省 CPU
- **周期计算**: `cyclePosition = floor(time / glimmerSpeed)`，`cycleLength = messageWidth + 20`
- **方向**: `'requesting'` 模式从左到右扫，其他模式从右到左
- **屏幕外**: 停滞时返回 -100（微光索引在可见范围外）

### useStalledAnimation

检测 token 流何时停滞并计算平滑强度：

- **停滞检测**: 3 秒没有新 token（当 `responseLength` 增加时重置）
- **活跃工具覆盖**: `hasActiveTools=true` 重置停滞计时器（工具仍在工作）
- **强度衰减**: 停滞检测后 2 秒内从 0 衰减到 1
- **平滑转换**: 50ms 间隔插值，0.1 平滑因子
- **减少运动**: 即时强度变化（无平滑）
- **挂载处理**: 尚无 token 时使用 `mountTime` 进行初始停滞计算

### 颜色工具函数

- **`interpolateColor(color1, color2, t)`**: 带四舍五入的线性 RGB 插值
- **`parseRGB(colorStr)`**: 基于正则表达式的 `rgb(r,g,b)` 解析器，带 Map 缓存
- **`toRGBColor(color)`**: 将 RGB 对象格式化为 `rgb(r,g,b)` 字符串
- **`hueToRgb(hue)`**: 完全 HSL 到 RGB 转换（s=0.7，l=0.6）用于语音模式波形颜色

## 动画架构

```
共享时钟 (ClockContext)
    ↓
useAnimationFrame(50) — 单一订阅者驱动时钟
    ↓
┌─────────────────────────────────────────────┐
│ SpinnerAnimationRow (50ms 渲染循环)         │
│  ├── SpinnerGlyph (基于帧的字符)            │
│  ├── GlimmerMessage (闪光扫动)              │
│  ├── Timer (经过时间)                       │
│  ├── Token 计数器 (平滑增量)                 │
│  ├── 思考闪光 (sin 波不透明度)               │
│  └── 队友状态                               │
└─────────────────────────────────────────────┘
    ↓
useStalledAnimation (停滞检测 + 强度)
    ↓
颜色插值 (消息颜色 → 错误红色)
```

## 性能优化

1. **动画隔离**: 只有 `SpinnerAnimationRow` 以 50ms 重新渲染；父组件每轮 ~25 次渲染而非 ~383 次
2. **时钟取消订阅**: `useShimmerAnimation` 停滞时向 `useAnimationFrame` 传递 `null` — 动画不可见时不 tick
3. **记忆化 stringWidth**: `stringWidth(message)` 在 50ms 循环中记忆化（Bun 原生调用昂贵）
4. **基于 ref 的状态**: 计时器 ref（`loadingStartTimeRef`、`totalPausedMsRef`）避免状态变化时的重新渲染
5. **平滑强度**: 50ms 步长插值避免突然的颜色跳跃
6. **减少运动**: 禁用闪光，使用简单点切换，即时强度变化
7. **死代码消除**: 队友组件使用动态 `require()` 以在不需要时进行 tree-shaking

## 旋转器模式

| 模式 | 描述 | 闪光速度 | 字形 |
|------|-------------|-----------|-------|
| `requesting` | 等待 API 响应 | 50ms（快） | ↑ 箭头 |
| `tool-use` | 执行工具 | 200ms（慢） | ↓ 箭头 |
| `tool-input` | 为工具提供输入 | 200ms | ↓ 箭头 |
| `responding` | 生成响应 | 200ms | ↓ 箭头 |
| `thinking` | 扩展思考模式 | 200ms | — |

## 关键设计决策

1. **共享动画时钟**: 所有旋转器订阅单一 ClockContext — 动画保持同步，一次唤醒
2. **基于帧的动画**: 旋转器字符使用帧索引（而非时间）以保持一致的动画速度
3. **通过 token 计数检测停滞**: 通过监视响应长度变化检测停滞，而非仅依赖时间
4. **渐进宽度门控**: 状态元素（思考、计时器、token）根据可用的终端宽度显示/隐藏
5. **颜色插值优于主题切换**: 停滞颜色转换使用平滑 RGB 插值而非突然的主题变化
6. **平台特定字符**: Ghostty（避免渲染偏移）、macOS 和 Linux 使用不同的旋转器字符
7. **基于 ref 的计时器状态**: 使用 ref 而非 state 用于计时器值，以避免不必要的重新渲染
8. **队友隔离**: 队友旋转器组件动态导入以进行死代码消除
