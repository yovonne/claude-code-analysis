# 杂项命令

## 概述

杂项命令涵盖不属于其他命令类别的实用、促销、配置和特定功能的功能。包括边问题、会话标记、安装诊断、编辑器模式切换、语音模式、终端设置以及各种促销或功能门控命令。

---

## /btw

**目的：** 提问快速边问题而不中断主对话流程。

**来源：** `restored-src/src/commands/btw/`

| 文件 | 目的 |
|------|---------|
| `index.ts` | 命令注册为 local-jsx 类型 |
| `btw.tsx` | 带流式响应和滚动支持的边问题 UI |

**注册：**
```typescript
{
  type: 'local-jsx',
  name: 'btw',
  description: 'Ask a quick side question without interrupting the main conversation',
  immediate: true,
  argumentHint: '<question>',
  load: () => import('./btw.js'),
}
```

**行为：**
- 渲染可滚动对话框，通过 `runSideQuestion()` 发送问题作为边问题
- 使用缓存的系统提示参数（`getLastCacheSafeParams`）以在分叉请求上实现提示缓存命中
- 在答案流入时显示微调器，然后将响应渲染为 Markdown
- 在全局配置中跟踪使用计数（`btwUseCount`）
- 用 Escape、Enter、Space 或 Ctrl+C/D 关闭
- 可用上/下箭头或 Ctrl+P/Ctrl+N 滚动

---

## /summary

**目的：** 存根命令（禁用）。

**来源：** `restored-src/src/commands/summary/`

| 文件 | 目的 |
|------|---------|
| `index.js` | 存根注册 — 始终禁用并隐藏 |

**注册：**
```javascript
{ isEnabled: () => false, isHidden: true, name: 'stub' }
```

此命令是一个占位符，没有实现。

---

## /think-back

**目的：** 生成并播放个性化的"2025 Claude Code 年度回顾"ASCII 动画。

**来源：** `restored-src/src/commands/thinkback/`

| 文件 | 目的 |
|------|---------|
| `index.ts` | 带功能门检查的命令注册 |
| `thinkback.tsx` | 完整安装、菜单和动画流程 UI |

**注册：**
```typescript
{
  type: 'local-jsx',
  name: 'think-back',
  description: 'Your 2025 Claude Code Year in Review',
  isEnabled: () => checkStatsigFeatureGate_CACHED_MAY_BE_STALE('tengu_thinkback'),
  load: () => import('./thinkback.js'),
}
```

**功能门：** 由 `tengu_thinkback` Statsig 功能门控制。

**安装流程（`ThinkbackInstaller`）：**
1. 检查市场是否已安装（`loadKnownMarketplacesConfig`）
2. 如果市场缺失，使用适当的市场 repo（外部用户为 `anthropics/claude-plugins-official`，内部为 `anthropics/claude-code-marketplace`）通过 `addMarketplaceSource()` 安装
3. 如果市场存在但插件缺失，通过 `refreshMarketplace()` 刷新市场
4. 通过 `installSelectedPlugins([pluginId])` 安装插件
5. 如果插件已安装但被禁用，通过 `enablePluginOp()` 启用
6. 每步后清除所有缓存

**菜单流程（`ThinkbackMenu`）：**
- 显示标题为"Think Back on 2025 with Claude Code"的对话框
- 如果还没有动画，显示单个"Let's go!"选项以生成
- 如果动画已存在，提供四个操作：

| 操作 | 值 | 描述 |
|--------|-------|-------------|
| Play animation | `play` | 观看年度回顾 |
| Edit content | `edit` | 修改动画 |
| Fix errors | `fix` | 修复验证或渲染问题 |
| Regenerate | `regenerate` | 从头开始创建新动画 |

**生成操作：**
- `edit` → 发送提示使用 `mode=edit` 的 Skill 工具
- `fix` → 发送提示使用 `mode=fix` 的 Skill 工具（运行验证器，修复错误）
- `regenerate` → 发送提示使用 `mode=regenerate` 的 Skill 工具（删除现有，从头开始）

**动画播放（`playAnimation`）：**
1. 在已安装插件的技能目录中定位 `year_in_review.js` 和 `player.js`
2. 继续前验证两个文件都存在
3. 通过 `inkInstance.enterAlternateScreen()` 接管终端
4. 使用继承的 stdio 运行 `node player.js`
5. 通过 `inkInstance.exitAlternateScreen()` 恢复终端
6. 在系统浏览器中打开 `year_in_review.html` 以下载视频

---

## /thinkback-play

**目的：** 直接播放 thinkback 动画的隐藏命令。在生成完成后由 thinkback 技能调用。

**来源：** `restored-src/src/commands/thinkback-play/`

| 文件 | 目的 |
|------|---------|
| `index.ts` | 命令注册 — 隐藏，功能门控 |
| `thinkback-play.ts` | 定位插件并调用 `playAnimation()` |

**注册：**
```typescript
{
  type: 'local',
  name: 'thinkback-play',
  description: 'Play the thinkback animation',
  isEnabled: () => checkStatsigFeatureGate_CACHED_MAY_BE_STALE('tengu_thinkback'),
  isHidden: true,
  supportsNonInteractive: false,
  load: () => import('./thinkback-play.js'),
}
```

---

## /stickers

**目的：** 在浏览器中打开 Claude Code 贴纸订购页面。

**来源：** `restored-src/src/commands/stickers/`

| 文件 | 目的 |
|------|---------|
| `index.ts` | 命令注册为 local 类型 |
| `stickers.ts` | 在浏览器中打开 StickerMule URL |

**注册：**
```typescript
{
  type: 'local',
  name: 'stickers',
  description: 'Order Claude Code stickers',
  supportsNonInteractive: false,
  load: () => import('./stickers.js'),
}
```

**行为：**
- 通过 `openBrowser()` 打开 `https://www.stickermule.com/claudecode`
- 如果浏览器启动失败则显示成功消息或回退 URL

---

## /feedback

**目的：** 提交关于 Claude Code 的反馈或报告错误。

**来源：** `restored-src/src/commands/feedback/`

| 文件 | 目的 |
|------|---------|
| `index.ts` | 带可用性检查的命令注册 |
| `feedback.tsx` | 渲染 Feedback 组件 |

**注册：**
```typescript
{
  aliases: ['bug'],
  type: 'local-jsx',
  name: 'feedback',
  description: 'Submit feedback about Claude Code',
  argumentHint: '[report]',
  isEnabled: () => !(
    isEnvTruthy(process.env.CLAUDE_CODE_USE_BEDROCK) ||
    isEnvTruthy(process.env.CLAUDE_CODE_USE_VERTEX) ||
    isEnvTruthy(process.env.CLAUDE_CODE_USE_FOUNDRY) ||
    isEnvTruthy(process.env.DISABLE_FEEDBACK_COMMAND) ||
    isEnvTruthy(process.env.DISABLE_BUG_COMMAND) ||
    isEssentialTrafficOnly() ||
    process.env.USER_TYPE === 'ant' ||
    !isPolicyAllowed('allow_product_feedback')
  ),
  load: () => import('./feedback.js'),
}
```

**可用性条件**（任何为真时禁用）：
- 使用 Bedrock、Vertex 或 Foundry 提供者
- 设置了 `DISABLE_FEEDBACK_COMMAND` 或 `DISABLE_BUG_COMMAND` 环境变量
- 仅限基本流量模式激活
- 内部 Anthropic 用户（`USER_TYPE === 'ant'`）
- 策略不允许产品反馈

**行为：**
- 别名 `/bug` 提供相同功能
- 接受可选的 `[report]` 参数作为初始描述
- 使用当前消息和中止信号渲染 `Feedback` 组件

---

## /release-notes

**目的：** 查看最新的 Claude Code 发行说明和变更日志。

**来源：** `restored-src/src/commands/release-notes/`

| 文件 | 目的 |
|------|---------|
| `index.ts` | 命令注册为 local 类型 |
| `release-notes.ts` | 获取和格式化变更日志数据 |

**注册：**
```typescript
{
  description: 'View release notes',
  name: 'release-notes',
  type: 'local',
  supportsNonInteractive: true,
  load: () => import('./release-notes.js'),
}
```

**行为：**
1. 通过 `fetchAndStoreChangelog()` 尝试获取带 500ms 超时的最新变更日志
2. 如果获取成功，格式化和显示新说明
3. 如果获取失败或超时，回退到缓存的说明通过 `getStoredChangelog()`
4. 如果没有缓存说明可用，显示变更日志 URL

---

## /passes

**目的：** 与朋友分享免费一周的 Claude Code 并赚取额外使用次数。

**来源：** `restored-src/src/commands/passes/`

| 文件 | 目的 |
|------|---------|
| `index.ts` | 带动态描述和可见性的命令注册 |
| `passes.tsx` | 带有首次访问跟踪的 Passes UI |

**注册：**
```typescript
{
  type: 'local-jsx',
  name: 'passes',
  get description() {
    const reward = getCachedReferrerReward()
    if (reward) {
      return 'Share a free week of Claude Code with friends and earn extra usage'
    }
    return 'Share a free week of Claude Code with friends'
  },
  get isHidden() {
    const { eligible, hasCache } = checkCachedPassesEligibility()
    return !eligible || !hasCache
  },
  load: () => import('./passes.js'),
}
```

**可见性：** 当用户没有资格参与 passes 程序或资格缓存不可用时隐藏。

---

## /upgrade

**目的：** 升级到 Claude Max 订阅，获得更高速率限制和更多 Opus 访问。

**来源：** `restored-src/src/commands/upgrade/`

| 文件 | 目的 |
|------|---------|
| `index.ts` | 带可用性和启用检查的命令注册 |
| `upgrade.tsx` | 带 Max 计划检查和登录重定向的升级流程 |

**注册：**
```typescript
{
  type: 'local-jsx',
  name: 'upgrade',
  description: 'Upgrade to Max for higher rate limits and more Opus',
  availability: ['claude-ai'],
  isEnabled: () =>
    !isEnvTruthy(process.env.DISABLE_UPGRADE_COMMAND) &&
    getSubscriptionType() !== 'enterprise',
  load: () => import('./upgrade.js'),
}
```

**可用性：** 仅对 `claude-ai` 认证类型可用。企业用户和设置 `DISABLE_UPGRADE_COMMAND` 环境变量时禁用。

---

## /doctor

**目的：** 诊断和验证 Claude Code 安装和设置。

**来源：** `restored-src/src/commands/doctor/`

| 文件 | 目的 |
|------|---------|
| `index.ts` | 带启用检查的命令注册 |
| `doctor.tsx` | 渲染 Doctor 诊断屏幕 |

**注册：**
```typescript
{
  name: 'doctor',
  description: 'Diagnose and verify your Claude Code installation and settings',
  isEnabled: () => !isEnvTruthy(process.env.DISABLE_DOCTOR_COMMAND),
  type: 'local-jsx',
  load: () => import('./doctor.js'),
}
```

**行为：**
- 渲染执行诊断检查的 `Doctor` 屏幕组件
- 可通过 `DISABLE_DOCTOR_COMMAND` 环境变量禁用

---

## /keybindings

**目的：** 在编辑器中打开或创建用户的按键绑定配置文件。

**来源：** `restored-src/src/commands/keybindings/`

| 文件 | 目的 |
|------|---------|
| `index.ts` | 带启用自定义检查的命令注册 |
| `keybindings.ts` | 如需要则创建模板文件并在编辑器中打开 |

**注册：**
```typescript
{
  name: 'keybindings',
  description: 'Open or create your keybindings configuration file',
  isEnabled: () => isKeybindingCustomizationEnabled(),
  supportsNonInteractive: false,
  type: 'local',
  load: () => import('./keybindings.js'),
}
```

**可用性：** 仅当按键绑定自定义启用时（预览功能）。

---

## /vim

**目的：** 在 Vim 和 Normal（readline）编辑模式之间切换。

**来源：** `restored-src/src/commands/vim/`

| 文件 | 目的 |
|------|---------|
| `index.ts` | 命令注册为 local 类型 |
| `vim.ts` | 在全局配置中切换编辑器模式 |

**注册：**
```typescript
{
  name: 'vim',
  description: 'Toggle between Vim and Normal editing modes',
  supportsNonInteractive: false,
  type: 'local',
  load: () => import('./vim.js'),
}
```

---

## /voice

**目的：** 切换语音模式以进行按键通话语音输入。

**来源：** `restored-src/src/commands/voice/`

| 文件 | 目的 |
|------|---------|
| `index.ts` | 带功能门和可见性检查的命令注册 |
| `voice.ts` | 带预检检查的完整语音模式切换 |

**注册：**
```typescript
{
  type: 'local',
  name: 'voice',
  description: 'Toggle voice mode',
  availability: ['claude-ai'],
  isEnabled: () => isVoiceGrowthBookEnabled(),
  get isHidden() {
    return !isVoiceModeEnabled()
  },
  supportsNonInteractive: false,
  load: () => import('./voice.js'),
}
```

**可用性：** 仅对 `claude-ai` 认证类型和通过 GrowthBook 启用语音功能时可用。

---

## /tag

**目的：** 在当前会话上切换可搜索标签。

**来源：** `restored-src/src/commands/tag/`

| 文件 | 目的 |
|------|---------|
| `index.ts` | 命令注册（仅内部） |
| `tag.tsx` | 带删除确认对话框的标签切换 UI |

**注册：**
```typescript
{
  type: 'local-jsx',
  name: 'tag',
  description: 'Toggle a searchable tag on the current session',
  isEnabled: () => process.env.USER_TYPE === 'ant',
  argumentHint: '<tag-name>',
  load: () => import('./tag.js'),
}
```

**可用性：** 仅内部 Anthropic 用户（`USER_TYPE === 'ant'`）。

---

## /rewind

**目的：** 将代码和/或对话恢复到之前的点。

**来源：** `restored-src/src/commands/rewind/`

| 文件 | 目的 |
|------|---------|
| `index.ts` | 带 `checkpoint` 别名的命令注册 |
| `rewind.ts` | 打开消息选择器 UI |

**注册：**
```typescript
{
  description: 'Restore the code and/or conversation to a previous point',
  name: 'rewind',
  aliases: ['checkpoint'],
  argumentHint: '',
  type: 'local',
  supportsNonInteractive: false,
  load: () => import('./rewind.js'),
}
```

**行为：**
- 调用 `context.openMessageSelector()` 显示消息选择 UI
- 返回跳过结果以避免追加任何输出消息

---

## /onboarding

**目的：** 存根命令（禁用）。

**来源：** `restored-src/src/commands/onboarding/`

| 文件 | 目的 |
|------|---------|
| `index.js` | 存根注册 — 始终禁用并隐藏 |

此命令是一个占位符，没有实现。

---

## /terminal-setup

**目的：** 配置终端按键绑定以方便多行提示输入。

**来源：** `restored-src/src/commands/terminalSetup/`

| 文件 | 目的 |
|------|---------|
| `index.ts` | 带终端检测的命令注册 |
| `terminalSetup.tsx` | 特定于终端的按键绑定安装逻辑 |

**注册：**
```typescript
{
  type: 'local-jsx',
  name: 'terminal-setup',
  description:
    env.terminal === 'Apple_Terminal'
      ? 'Enable Option+Enter key binding for newlines and visual bell'
      : 'Install Shift+Enter key binding for newlines',
  isHidden: env.terminal !== null && env.terminal in NATIVE_CSIU_TERMINALS,
  load: () => import('./terminalSetup.js'),
}
```

**自动隐藏的终端**（原生 CSI u / Kitty 协议支持）：

| 终端 | 显示名称 |
|----------|-------------|
| `ghostty` | Ghostty |
| `kitty` | Kitty |
| `iTerm.app` | iTerm2 |
| `WezTerm` | WezTerm |
| `WarpTerminal` | Warp |

**支持的终端设置：**

| 终端 | 配置 |
|----------|--------------|
| `Apple_Terminal` | 通过 PlistBuddy 启用 Option 作为 Meta 键和可视铃声 |
| `vscode` | 在 VSCode keybindings.json 中安装 Shift+Enter 按键绑定 |
| `cursor` | 在 Cursor keybindings.json 中安装 Shift+Enter 按键绑定 |
| `windsurf` | 在 Windsurf keybindings.json 中安装 Shift+Enter 按键绑定 |
| `alacritty` | 追加 Shift+Return 绑定到 alacritty.toml |
| `zed` | 在 Zed keymap.json 中添加 Shift+Enter 绑定 |

---

## /break-cache

**目的：** 存根命令（禁用）。

**来源：** `restored-src/src/commands/break-cache/`

| 文件 | 目的 |
|------|---------|
| `index.js` | 存根注册 — 始终禁用并隐藏 |

此命令是一个占位符，没有实现。

---

## /ctx-viz

**目的：** 存根命令（禁用）。

**来源：** `restored-src/src/commands/ctx_viz/`

| 文件 | 目的 |
|------|---------|
| `index.js` | 存根注册 — 始终禁用并隐藏 |

此命令是一个占位符，没有实现。

---

## 摘要

| 命令 | 类型 | 可见性 | 关键功能 |
|---------|------|-----------|-------------|
| `/btw` | local-jsx | 始终可见 | 带提示缓存重用的边问题 |
| `/summary` | stub | 隐藏 | 占位符 |
| `/think-back` | local-jsx | 功能门控 | 2025 年度回顾动画 |
| `/thinkback-play` | local | 隐藏 | 动画播放助手 |
| `/stickers` | local | 始终可见 | 打开贴纸订购页面 |
| `/feedback` | local-jsx | 条件性 | 错误报告和反馈（别名：`/bug`） |
| `/release-notes` | local | 始终可见 | 带 500ms 新鲜获取的变更日志 |
| `/passes` | local-jsx | 条件性 | 推荐 passes 程序 |
| `/upgrade` | local-jsx | 条件性 | 升级到 Max 订阅 |
| `/doctor` | local-jsx | 条件性 | 安装诊断 |
| `/keybindings` | local | 条件性 | 打开按键绑定配置文件 |
| `/vim` | local | 始终可见 | 切换 Vim/Normal 编辑模式 |
| `/voice` | local | 功能门控 | 切换按键通话语音模式 |
| `/tag` | local-jsx | 仅内部 | 会话标记 |
| `/rewind` | local | 始终可见 | 恢复到之前的点（别名：`/checkpoint`） |
| `/onboarding` | stub | 隐藏 | 占位符 |
| `/terminal-setup` | local-jsx | 条件性 | 终端按键绑定配置 |
| `/break-cache` | stub | 隐藏 | 占位符 |
| `/ctx-viz` | stub | 隐藏 | 占位符 |
