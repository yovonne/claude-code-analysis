# 语音模式

## 目的

语音模式通过 claude.ai 上的 `voice_stream` 端点实现与 Claude Code 的实时语音交互。它提供对话式语音界面作为基于文本的终端交互的替代方案，需要 Anthropic OAuth 认证。

## 位置

`restored-src/src/voice/voiceModeEnabled.ts`

## 主要导出

### 函数

- `isVoiceGrowthBookEnabled()`: 检查 GrowthBook 功能标志和 kill-switch。如果 `tengu_amber_quartz_disabled` 标志被翻转（紧急关闭），则返回 `true`（紧急关闭）。默认为 `false` 意味着缺失/过时的磁盘缓存读取为"未杀死"。
- `hasVoiceAuth()`: 检查用户是否有有效的 Anthropic OAuth 令牌。仅当 Anthropic 认证启用 AND 存在访问令牌时返回 `true`。
- `isVoiceModeEnabled()`: 结合认证 + GrowthBook kill-switch 的完整运行时检查。这是决定语音模式是否应该激活的最终函数。

## 依赖

### 内部依赖

- `../services/analytics/growthbook.js` — `getFeatureValue_CACHED_MAY_BE_STALE()` 用于 GrowthBook 标志检查
- `../utils/auth.js` — `getClaudeAIOAuthTokens()` 用于 OAuth 令牌检索和 `isAnthropicAuthEnabled()` 用于认证提供者检查
- `bun:bundle` — `feature()` 用于构建时功能标志

## 实现细节

### 三层门控

语音模式使用三层门系统：

```
isVoiceModeEnabled()
    ↓
├── hasVoiceAuth()
│   ├── isAnthropicAuthEnabled() → Anthropic OAuth 提供商活跃？
│   └── getClaudeAIOAuthTokens() → accessToken 存在？
│
└── isVoiceGrowthBookEnabled()
    ├── feature('VOICE_MODE') → 构建时功能标志？
    └── !getFeatureValue_CACHED_MAY_BE_STALE('tengu_amber_quartz_disabled', false)
        → Kill-switch 未激活？
```

所有三个条件都必须为 true 才能启用语音模式。

### 认证层

`hasVoiceAuth()` 执行两个检查：

1. **提供者检查** — `isAnthropicAuthEnabled()` 验证认证提供者是否为 Anthropic（不是 API 密钥、Bedrock、Vertex 或 Foundry）。语音模式使用 claude.ai 上的 `voice_stream` 端点，仅适用于 OAuth 认证。

2. **令牌检查** — `getClaudeAIOAuthTokens()` 检索缓存的 OAuth 令牌。该函数被记忆化 — 首次调用在 macOS 上生成 `security` 模块（约 20-50ms），后续调用是缓存命中。记忆化在令牌刷新时清除（约每小时一次）。

没有提供者检查，语音 UI 会渲染但 `connectVoiceStream` 在用户未登录时会静默失败。

### GrowthBook 层

`isVoiceGrowthBookEnabled()` 使用正三元模式以获得正确的 bundle 行为：

```typescript
return feature('VOICE_MODE')
    ? !getFeatureValue_CACHED_MAY_BE_STALE('tengu_amber_quartz_disabled', false)
    : false
```

关键设计决策：

- **Kill-switch 默认为 `false`**：缺失或过时的磁盘缓存读取为"未杀死"，因此新安装立即启用语音工作，无需等待 GrowthBook 初始化。
- **正三元模式**：`if (!feature(...)) return` 模式不会从外部构建中消除内联字符串字面量。三元式确保正确的 tree-shaking。
- **缓存可能过时**：GrowthBook 值缓存在磁盘上可能过时，但这对 kill-switch 是可接受的 — 最坏情况是短暂窗口中禁用功能仍然可见。

### 使用模式

| 上下文 | 要使用的函数 | 原因 |
|---|---|---|
| 命令注册 | `isVoiceModeEnabled()` | 需要完整检查 |
| 配置 UI 可见性 | `isVoiceModeEnabled()` | 需要完整检查 |
| `/voice` 命令 | `isVoiceModeEnabled()` | 需要完整检查 |
| React 渲染路径 | `useVoiceEnabled()` | 记忆化认证检查，避免冗余 keychain 读取 |
| VoiceModeNotice | `isVoiceModeEnabled()` | 命令时间路径，可接受新的 keychain 读取 |

## 数据流

```
用户启动 Claude Code
    ↓
启动期间调用 isVoiceModeEnabled()
    ↓
├── isAnthropicAuthEnabled()
│   └── 检查认证提供者配置
│       ↓
├── getClaudeAIOAuthTokens()
│   └── 从 keychain 读取 OAuth 令牌（记忆化）
│       ↓
├── feature('VOICE_MODE')
│   └── 检查构建时功能标志
│       ↓
└── getFeatureValue_CACHED_MAY_BE_STALE('tengu_amber_quartz_disabled')
    └── 检查 kill-switch 标志（从磁盘缓存）
    ↓
全部为 true → 语音模式启用
任一为 false → 语音模式禁用
```

## 集成点

- **`/voice` 命令** — 语音模式的入口点，使用 `isVoiceModeEnabled()` 进行验证
- **Config 工具** — 可以启用/禁用语音模式，检查 `isVoiceModeEnabled()`
- **VoiceModeNotice** — UI 通知组件，使用 `isVoiceModeEnabled()`
- **React 组件** — 使用 `useVoiceEnabled()` 钩子而不是直接调用 `isVoiceModeEnabled()` 以记忆化认证检查
- **connectVoiceStream** — 到 claude.ai 语音端点的实际 WebSocket 连接

## 配置

| 功能门 | 用途 |
|---|---|
| `VOICE_MODE` | 语音模式的构建时功能标志 |
| `tengu_amber_quartz_disabled` | GrowthBook kill-switch 标志（默认：false） |

| 认证要求 | 值 |
|---|---|
| 提供者 | 仅限 Anthropic OAuth（不是 API 密钥、Bedrock、Vertex 或 Foundry） |
| 令牌 | 来自 `getClaudeAIOAuthTokens()` 的有效 `accessToken` |
| 端点 | claude.ai 上的 `voice_stream` |

## 错误处理

- **缺少 GrowthBook 标志**：默认为 `false`（未杀死） — 新安装上语音模式工作
- **缺少 OAuth 令牌**：`hasVoiceAuth()` 返回 `false` — 语音模式隐藏而不是显示损坏的 UI
- **错误的认证提供者**：`isAnthropicAuthEnabled()` 返回 `false` — 防止静默 `connectVoiceStream` 失败
- **过时缓存**：`getFeatureValue_CACHED_MAY_BE_STALE` 可能返回过时值，但这对 kill-switch 是可接受的（最坏情况：短暂可见窗口）

## 性能考虑

- `getClaudeAIOAuthTokens()` 被记忆化 — 首次调用在 macOS 上成本约 20-50ms（生成 `security` 模块），后续调用是 O(1) 缓存命中
- 令牌刷新大约每小时发生一次，导致每个刷新周期一次冷生成
- 对于 React 渲染路径，`useVoiceEnabled()` 钩子记忆化认证检查以避免每次渲染时冗余的 keychain 读取
- 命令时间路径（`/voice`、Config 工具）直接使用 `isVoiceModeEnabled()` — 在此频率下新的 keychain 读取是可接受的

## 相关模块

- [Auth](../05-utils/auth.md) — OAuth 令牌管理
- [Config Tool](../02-tools/ConfigTool.md) — 语音模式配置
- [GrowthBook Service](../04-services/) — 功能标志管理

## 备注

- 语音模式专用于 Anthropic OAuth 认证 — 不能与 API 密钥、AWS Bedrock、Google Vertex 或 Azure Foundry 一起使用
- kill-switch 标志名称 `tengu_amber_quartz_disabled` 遵循功能切换的内部命名约定
- `voice_stream` 端点是 claude.ai 特有的，不通过其他 Anthropic API 表面提供
- 三层门控（认证 + 构建标志 + kill-switch）确保语音模式可以在多个级别禁用：按用户（无认证）、按构建（功能标志）或全局（kill-switch）
