# Buddy System（伙伴系统）

## 目的

Buddy System 为 Claude Code 终端 UI 添加了一个持久的动画伙伴（"buddy"）。这个伙伴是一个程序生成的 ASCII 艺术角色，具有物种、稀有度、属性、帽子和个性。它显示空闲动画、反应气泡，并通过 `/buddy` 命令响应。

## 位置

`restored-src/src/buddy/` — 6 个文件：
- `types.ts` — 类型定义、物种、稀有度、属性
- `companion.ts` — 伙伴生成（基于确定性 PRNG 的抽取）
- `sprites.ts` — 带动画帧和帽子的 ASCII 艺术精灵渲染
- `CompanionSprite.tsx` — 在 UI 中渲染伙伴的 React 组件
- `prompt.ts` — 伙伴感知的系统提示注入
- `useBuddyNotification.tsx` — 通知钩子和 `/buddy` 命令检测

## 主要导出

### 类型（types.ts）

- `Rarity`: `'common' | 'uncommon' | 'rare' | 'epic' | 'legendary'`
- `Species`: 18 种物种，包括 duck、goose、cat、dragon、octopus、owl 等
- `Eye`: 6 种眼睛样式（`·`、`✦`、`×`、`◉`、`@`、`°`）
- `Hat`: 8 种帽子类型（none、crown、tophat、propeller、halo、wizard、beanie、tinyduck）
- `StatName`: 5 种属性（DEBUGGING、PATIENCE、CHAOS、WISDOM、SNARK）
- `CompanionBones`: 从 `hash(userId)` 派生的确定性物理特征
- `CompanionSoul`: 模型生成的名称和个性
- `Companion`: 完整伙伴对象（bones + soul + hatchedAt）
- `StoredCompanion`: 持久化到配置的内容（只有 soul；bones 重新生成）

### 常量（types.ts）

- `RARITY_WEIGHTS`: common(60)、uncommon(25)、rare(10)、epic(4)、legendary(1)
- `RARITY_STARS`: 每个稀有度的星星显示字符串
- `RARITY_COLORS`: 每个稀有度的主题颜色映射
- `SPECIES`: 18 种物种名称数组

### 函数（companion.ts）

- `roll(userId)`: 使用从 hashed userId 种子的 Mulberry32 PRNG 进行确定性伙伴抽取。每个 userId 缓存。
- `rollWithSeed(seed)`: 使用任意种子字符串的非缓存抽取。
- `companionUserId()`: 从 OAuth 账户 UUID 或 config userID 派生用户 ID。
- `getCompanion()`: 将重新生成的 bones 与配置中存储的 soul 合并。如果没有任何伙伴孵化，返回 undefined。

### 函数（sprites.ts）

- `renderSprite(bones, frame)`: 渲染 5 行 ASCII 艺术精灵，带眼睛替换和帽子覆盖。
- `spriteFrameCount(species)`: 返回物种的动画帧数。
- `renderFace(bones)`: 为窄终端返回紧凑的单行面部表示。

### 函数（prompt.ts）

- `companionIntroText(name, species)`: 生成向模型解释伙伴的系统提示部分。
- `getCompanionIntroAttachment(messages)`: 如果伙伴在当前消息列表中尚未宣布，返回 `companion_intro` 附件。

### 函数（useBuddyNotification.tsx）

- `isBuddyTeaserWindow()`: 在 2026 年 4 月 1-7 日预览期返回 true。
- `isBuddyLive()`: 2026 年 4 月后返回 true（功能永久上线）。
- `useBuddyNotification()`: React 钩子，在没有伙伴时在启动时显示彩虹色 `/buddy` 预览通知。
- `findBuddyTriggerPositions(text)`: 在文本中查找所有 `/buddy` 命令位置以进行语法高亮。

### React 组件（CompanionSprite.tsx）

- `CompanionSprite`: 主要组件，渲染带空闲动画、气泡和抚摸效果的伙伴。
- `CompanionFloatingBubble`: 全屏模式的浮动气泡覆盖（避免 overflow 裁剪）。
- `companionReservedColumns()`: 计算伙伴 UI 占用的终端列数。

## 依赖

### 内部依赖

- `../utils/config.js` — `getGlobalConfig()` 用于伙伴存储和静音状态
- `../state/AppState.js` — `companionReaction`、`companionPetAt`、`footerSelection` 状态
- `../hooks/useTerminalSize.js` — 用于响应式渲染的终端尺寸
- `../ink.js` — Ink 框架组件（Box、Text）
- `../context/notifications.js` — 预览通知的通知系统
- `../utils/fullscreen.js` — 全屏模式检测
- `../utils/theme.js` — 主题颜色类型
- `../utils/thinking.js` — `getRainbowColor()` 用于预览文本
- `bun:bundle` — 功能标志系统（`feature('BUDDY')`）

### 外部依赖

- `react` — React 钩子（useEffect、useRef、useState）
- `react/compiler-runtime` — React Compiler 优化
- `figures` — 终端 Unicode 符号（hearts 等）

## 实现细节

### 伙伴生成

伙伴使用种子 PRNG 从用户 ID 确定性生成：

1. **哈希** — `hashString(userId + SALT)` 产生数字种子（尽可能使用 `Bun.hash`，否则回退到 FNV-1a）
2. **PRNG** — Mulberry32 种子 PRNG 生成随机序列
3. **稀有度抽取** — 加权选择：common 60%、uncommon 25%、rare 10%、epic 4%、legendary 1%
4. **物种/眼睛/帽子** — 从相应数组均匀随机选择
5. **属性** — 一个峰值属性，一个低谷属性，其余分散。稀有度设定下限（common=5、uncommon=15、rare=25、epic=35、legendary=50）
6. **闪光** — 1% 几率闪光变体
7. **缓存** — 每个 userId 缓存结果以避免热路径上的冗余计算（500ms tick、每次按键、每次 turn 观察器）

### 安全设计

物种名称通过 `String.fromCharCode()` 构建，而不是字面量，以避免触发构建时 `excluded-strings.txt` 中的 canary 检查。这保持构建管道的秘密检测为实际代号启用，同时允许物种名称在 bundle 中。

Bones **从不持久化** — 每次读取时从 `hash(userId)` 重新生成。只有 soul（名称、个性、hatchedAt）被存储。这意味着：
- 物种重命名不会破坏存储的伙伴
- 用户无法编辑配置来伪造传奇稀有度
- 同一用户始终获得相同的伙伴

### 精灵动画

每个物种有 3 个动画帧用于空闲烦躁行为：

- **空闲序列**: `[0, 0, 0, 0, 1, 0, 0, 0, -1, 0, 0, 2, 0, 0, 0]` — 大部分休息（帧 0），偶尔烦躁（帧 1-2），罕见眨眼（-1）
- **Tick 速率**: 每 tick 500ms
- **兴奋状态**: 反应或被抚摸时，快速循环所有帧
- **眨眼**: 帧 -1 将眼睛字符替换为 `-` 一个 tick

### UI 渲染

伙伴适应终端宽度：

- **宽终端（≥100 列）**: 带可选气泡的完整精灵
- **窄终端（<100 列）**: 紧凑的单行面部 + 名称显示
- **全屏模式**: 气泡渲染为浮动覆盖（`CompanionFloatingBubble`）以避免 `overflowY: hidden` 裁剪
- **非全屏**: 气泡在精灵旁边内联渲染，缩小输入区域

气泡行为：
- 显示 20 个 tick（约 10 秒）
- 在最后 6 个 tick（约 3 秒）淡出，以便用户知道它即将消失
- 文本在 30 个字符宽度处换行
- 抚摸效果：5 帧心形动画在精灵上方浮动（约 2.5 秒）

### Buddy 预览

在预览窗口（2026 年 4 月 1-7 日），没有伙伴的用户在启动时会在 `/buddy` 上看到彩虹色通知。2026 年 4 月后，功能永久上线。`external === 'ant'` 检查绕过内部构建的日期限制。

## 数据流

```
用户启动 Claude Code
    ↓
getCompanion() → 检查 config.companion
    ↓（如果存在）
roll(userId) → 从哈希重新生成 bones
    ↓
合并存储的 soul + 重新生成的 bones
    ↓
CompanionSprite 渲染:
    ├── 500ms tick 间隔 → 循环空闲动画
    ├── AppState.companionReaction → 显示气泡
    ├── AppState.companionPetAt → 显示心形动画
    └── footerSelection === 'companion' → 聚焦状态
    ↓
用户输入 /buddy → findBuddyTriggerPositions() 高亮它
```

## 集成点

- **AppState** — `companionReaction`（气泡文本）、`companionPetAt`（抚摸时间戳）、`footerSelection`（聚焦状态）
- **系统提示** — `companionIntroText()` 将伙伴感知注入模型的指令
- **消息附件** — `getCompanionIntroAttachment()` 添加 `companion_intro` 附件以宣布伙伴
- **配置存储** — `config.companion` 存储 soul；`config.companionMuted` 禁用显示
- **终端布局** — `companionReservedColumns()` 告诉输入区域保留多少宽度
- **通知系统** — `useBuddyNotification()` 显示预览通知

## 配置

| 配置字段 | 类型 | 用途 |
|---|---|---|
| `config.companion` | `StoredCompanion \| undefined` | 持久化的伙伴 soul（名称、个性、hatchedAt） |
| `config.companionMuted` | `boolean` | 为 true 时禁用伙伴显示 |
| `config.oauthAccount.accountUuid` | `string` | 用作伙伴生成的确定性种子 |

| 功能门 | 用途 |
|---|---|
| `BUDDY` | 所有 buddy 功能的总开关 |

## 错误处理

- 缺少伙伴：`getCompanion()` 返回 `undefined`；组件渲染 `null`
- 静音伙伴：`companionMuted` 检查导致所有渲染路径提前返回
- 功能门禁用：`feature('BUDDY')` 检查导致提前返回，零开销

## 测试

- `rollWithSeed()` 允许使用已知种子进行确定性测试
- `rollCache` 可以在测试之间清除以实现隔离
- PRNG（Mulberry32）是确定性的，因此相同种子始终产生相同的伙伴

## 相关模块

- [Config](../05-utils/config.md) — 全局配置中的伙伴存储
- [State Management](../01-core-modules/state-management.md) — 反应和宠物状态的 AppState
- [Ink Framework](../06-ui/ink-framework.md) — 终端 UI 渲染

## 备注

- 伙伴是一个终端 UI 功能，不是 AI 代理 — 它是一个具有程序生成特征的动画 ASCII 艺术角色
- 18 种物种包括：duck、goose、blob、cat、dragon、octopus、owl、penguin、turtle、snail、ghost、axolotl、capybara、cactus、robot、rabbit、mushroom、chonk
- `/buddy` 命令触发伙伴孵化流程（在命令系统的其他地方处理）
- 抚摸心形在约 2.5 秒内以 5 帧浮动，使用 `figures` 包的 Unicode 心形符号
- 伙伴的稀有度通过 `RARITY_COLORS` 映射到主题颜色来确定其在终端中的颜色
