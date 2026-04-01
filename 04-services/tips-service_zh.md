# 提示服务

## 目的

管理在加载微调器期间显示的上下文提示。提示基于用户行为、会话状态和环境相关。系统选择最长时间未显示的提示以确保多样性。

## 位置

`restored-src/src/services/tips/`

### 关键文件

| 文件 | 描述 |
|------|-------------|
| `tipScheduler.ts` | 提示选择逻辑 — 查找要显示的提示，记录显示的提示 |
| `tipRegistry.ts` | 带有相关性检查和内容生成的提示定义 |
| `tipHistory.ts` | 在全局配置中持久化提示显示历史 |

## 关键导出

### 函数 (tipScheduler.ts)
- `getTipToShowOnSpinner(context)`：获取在微调器期间显示的下一个提示
- `selectTipWithLongestTimeSinceShown(availableTips)`：选择最近显示时间最长的提示
- `recordShownTip(tip)`：记录提示已显示（历史 + 分析）

### 函数 (tipRegistry.ts)
- `getRelevantTips(context)`：按相关性过滤提示，返回符合条件的提示

### 函数 (tipHistory.ts)
- `recordTipShown(tipId)`：在当前启动计数时记录提示显示
- `getSessionsSinceLastShown(tipId)`：获取自提示上次显示以来的会话数

### 类型
- `Tip`：`{ id, content, cooldownSessions, isRelevant(context) }`
- `TipContext`：传递给相关性检查的上下文对象（bashTools、readFileState、theme）

## 依赖

### 内部依赖
- `settings/settings` — 读取 spinnerTipsEnabled、spinnerTipsOverride
- `config` — 启动计数的全局配置、功能使用跟踪
- `analytics` — 遥测事件
- `growthbook` — 提示内容的 A/B 测试变体

### 外部依赖
- `chalk` — 提示内容的终端着色

## 实现细节

### 提示选择算法

1. 检查提示是否禁用（`spinnerTipsEnabled === false`）
2. 获取所有相关提示（通过 `isRelevant()` 和冷却过滤）
3. 在可用提示中，选择上次显示时间最长的提示
4. 如果只有一个可用提示，直接返回

### 提示结构

每个提示有：
- `id`：用于跟踪的唯一标识符
- `content`：返回提示文本的异步函数（可以是带主题颜色的上下文感知）
- `cooldownSessions`：显示之间的最小会话数
- `isRelevant(context)`：返回提示是否适用的异步函数

### 提示类别

注册表包括约 50 个提示，涵盖：
- **新用户入门**：new-user-warmup、plan-mode、permissions
- **功能发现**：memory command、theme command、status line、prompt queue
- **终端集成**：terminal-setup、shift-enter、vscode-command-install
- **多会话**：git-worktrees、color-when-multi-clauding
- **插件市场**：frontend-design-plugin、vercel-plugin
- **平台特定**：powershell-tool-env、colorterm-truecolor、paste-images-mac
- **对话管理**：continue、rename-conversation、double-esc
- **高级功能**：custom-commands、custom-agents、agent-flag
- **升级销售**：desktop-app、desktop-shortcut、web-app、mobile-app
- **增长实验**：effort-high-nudge、subagent-fanout-nudge、loop-command-nudge
- **推荐/积分**：guest-passes、overage-credit
- **仅内部**：important-claudemd、skillify（仅限 ANT）

### 自定义提示

用户可以通过 `settings.json` 中的 `spinnerTipsOverride` 覆盖默认提示：
- `tips`：自定义提示字符串数组
- `excludeDefault`：如果为 true 且存在自定义提示，则完全跳过内置提示

### 冷却系统

- 提示通过全局配置中的 `numStartups` 跟踪上次显示的会话
- 每个提示有一个 `cooldownSessions` 值（0-30 个会话）
- 在冷却期内的提示被过滤掉
- 选择偏向于最长未显示的提示

### 上下文感知内容

一些提示基于上下文生成动态内容：
- 通过 `color('suggestion', ctx.theme)` 的主题颜色
- 插件安装状态
- 通过 GrowthBook 标志的 A/B 测试变体

## 数据流

```
微调器渲染
    ↓
getTipToShowOnSpinner(context)
    ↓
检查 spinnerTipsEnabled 设置
    ↓
getRelevantTips(context)
    ↓
过滤：每个提示的 isRelevant()
    ↓
过滤：getSessionsSinceLastShown() >= cooldownSessions
    ↓
selectTipWithLongestTimeSinceShown()
    ↓
返回提示或 undefined
    ↓
recordShownTip(tip) — 更新历史 + 记录事件
```

## 集成点

- **微调器 UI**：在加载微调器期间显示提示
- **全局配置**：提示历史持久化为 `tipsHistory` 映射
- **设置**：`spinnerTipsEnabled` 和 `spinnerTipsOverride` 控制行为
- **分析**：`tengu_tip_shown` 事件在显示提示时记录

## 配置

- 设置：`spinnerTipsEnabled`（默认 true）、`spinnerTipsOverride`（自定义提示）
- 全局配置：`tipsHistory` 映射（tipId → 显示时的 numStartups）
- 冷却范围：每个提示 0-30 个会话

## 错误处理

- 相关性检查失败被捕获并记录，提示被视为不相关
- 配置读取失败返回 false 表示相关性
- 提示内容生成失败会显示为空字符串

## 测试

- `getSessionsSinceLastShown()` 对从未显示的提示返回 Infinity
- 历史去重：相同提示在相同启动计数时不重新记录

## 相关模块

- [分析](./analytics-service.md) — 提示显示遥测
- [设置](./settings-commands.md) — 微调器提示配置

## 备注

- 提示在微调器加载期间显示，以在不中断工作流的情况下教育用户
- "最长未显示时间"的选择确保所有提示之间的均匀覆盖
- 仅内部提示通过 `USER_TYPE === 'ant'` 环境变量过滤
- A/B 测试变体允许在不更改代码的情况下测试不同的提示文案
- 自定义提示可以通过 `excludeDefault: true` 完全替换内置提示
