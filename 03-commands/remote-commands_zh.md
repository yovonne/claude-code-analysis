# 远程命令

## 目的

记录管理远程会话连接、环境配置和远程控制（桥接）功能的 CLI 命令。这些命令使 Claude Code 能够在远程和分布式环境中运行。

---

## /web-setup（远程会话设置）

### 目的
通过连接用户的 GitHub 账户在网页（claude.ai/code）上设置 Claude Code。用于首次远程会话配置。

### 位置
`restored-src/src/commands/remote-setup/index.ts`
`restored-src/src/commands/remote-setup/remote-setup.tsx`
`restored-src/src/commands/remote-setup/api.ts`

### 命令定义
| 属性 | 值 |
|----------|-------|
| 名称 | `web-setup` |
| 类型 | `local-jsx` |
| 描述 | Setup Claude Code on the web (requires connecting your GitHub account) |
| 可用性 | `['claude-ai']` |
| 启用 | 当 `tengu_cobalt_lantern` 功能标志开启且 `allow_remote_sessions` 策略允许时 |
| 隐藏 | 当 `allow_remote_sessions` 策略不允许时 |

### 关键导出

#### 函数
- `call(onDone)`：入口点；渲染 `Web` 设置组件
- `checkLoginState()`：检查 GitHub 认证状态和令牌可用性
- `importGithubToken(token)`：POSTs GitHub 令牌到 CCR 后端进行验证和存储
- `createDefaultEnvironment()`：为首次用户尽力创建默认环境
- `isSignedIn()`：检查用户是否有有效的 Claude OAuth 凭证
- `getCodeWebUrl()`：返回 claude.ai/code URL

#### 类型
- `CheckResult`：登录状态的区分联合（`not_signed_in`、`has_gh_token`、`gh_not_installed`、`gh_not_authenticated`）
- `ImportTokenResult`：`{ github_username: string }`
- `ImportTokenError`：区分联合（`not_signed_in`、`invalid_token`、`server`、`network`）

#### 类
- `RedactedGithubToken`：包装原始 GitHub 令牌，使其字符串表示被编辑。`toString()`、`toJSON()` 和 `inspect.custom` 都返回 `[REDACTED:gh-token]`。仅在原始值被放入 HTTP body 的时刻调用 `.reveal()`。

### 实现细节

#### 设置流程（多步）
1. **检查**：通过 `checkLoginState()` 验证登录状态：
   - 通过 `isSignedIn()` 检查 Claude OAuth（调用 `prepareApiRequest()`）
   - 通过 `getGhAuthStatus()` 检查 GitHub CLI 认证
   - 如果已认证，通过 `gh auth token` 检索令牌（5 秒超时）
   - 包装令牌在 `RedactedGithubToken` 中以安全处理

2. **确认**：显示确认对话框，说明：
   - Claude on the web 需要 GitHub 连接以进行克隆/推送
   - 使用本地凭证向 GitHub 认证
   - 用户可以继续或取消

3. **上传**：导入 GitHub 令牌：
   - POST 到 `/v1/code/github/import-token` 和令牌
   - 服务器向 GitHub 的 `/user` 端点验证
   - 令牌被 Fernet 加密并存储在 `sync_user_tokens`
   - 尽力创建默认环境（非致命）
   - 在浏览器中打开 claude.ai/code

#### 令牌导入 API
`importGithubToken` 函数：
- 使用 `prepareApiRequest()` 进行 OAuth 认证
- 发送到 `{BASE_API_URL}/v1/code/github/import-token`
- 包含 `anthropic-beta: ccr-byoc-2025-07-29` 头
- 包含 `x-organization-uuid` 头
- 处理响应代码：200（成功）、400（无效令牌）、401（未登录）
- 捕获网络错误并报告，不记录令牌

#### 默认环境创建
`createDefaultEnvironment()` 创建受信任网络访问环境：
- 首先检查现有环境（避免重复）
- POST 到 `{BASE_API_URL}/v1/environment_providers/cloud/create`
- 环境配置：`anthropic_cloud` 类型、Python 3.11 + Node 20、默认主机访问
- 失败是非致命的 — 网页 onboarding 回退到 env-setup

#### 安全注意事项
- `RedactedGithubToken` 防止日志、错误或 JSON 序列化中的意外令牌泄漏
- 令牌仅在 HTTP body 构造的单个时刻揭示
- 错误消息从不包含令牌数据
- POST body 明确从 axios 错误日志中排除

---

## /remote-env（远程环境配置）

### 目的
为远程（teleport）会话配置默认远程环境。

### 位置
`restored-src/src/commands/remote-env/index.ts`
`restored-src/src/commands/remote-env/remote-env.tsx`

### 命令定义
| 属性 | 值 |
|----------|-------|
| 名称 | `remote-env` |
| 类型 | `local-jsx` |
| 描述 | Configure the default remote environment for teleport sessions |
| 启用 | 当用户是 claude.ai 订阅者且 `allow_remote_sessions` 策略允许时 |
| 隐藏 | 当不是订阅者或策略不允许时 |

### 关键导出

#### 函数
- `call(onDone)`：入口点；渲染 `RemoteEnvironmentDialog`

---

## /remote-control（桥接/远程控制）

### 目的
将当前终端连接到远程控制会话，实现 CLI 和 claude.ai 之间的双向通信。

### 位置
`restored-src/src/commands/bridge/index.ts`
`restored-src/src/commands/bridge/bridge.tsx`

### 命令定义
| 属性 | 值 |
|----------|-------|
| 名称 | `remote-control` |
| 别名 | `rc` |
| 类型 | `local-jsx` |
| 描述 | Connect this terminal for remote-control sessions |
| 参数提示 | `[name]` |
| Immediate | `true` |
| 启用 | 当 `BRIDGE_MODE` 功能开启且桥接启用时 |
| 隐藏 | 当桥接未启用时 |

### 关键导出

#### 函数
- `call(onDone, _context, args)`：入口点；渲染带可选会话名称的 `BridgeToggle`
- `checkBridgePrerequisites()`：连接前验证所有桥接先决条件

#### 组件
- `BridgeToggle`：管理桥接连接生命周期
- `BridgeDisconnectDialog`：当已连接时显示；提供断开、显示二维码或继续选项

### 实现细节

#### 桥接连接架构
启用时，桥接：
1. 在 AppState 中设置 `replBridgeEnabled`
2. 在 REPL.tsx 中触发 `useReplBridge` 以初始化连接
3. 向后端注册环境
4. 使用当前对话创建会话
5. 从后端轮询工作
6. 连接用于 CLI 和 claude.ai 之间双向消息的入口 WebSocket

#### 先决条件检查（`checkBridgePrerequisites`）
1. **组织策略**：等待策略限制加载，检查 `allow_remote_control`
2. **桥接禁用原因**：检查 `getBridgeDisabledReason()` 的任何阻止条件
3. **版本兼容性**：
   - 根据 `isEnvLessBridgeEnabled()` 和 `KAIROS` 确定 v1 vs v2 路径
   - 在助手模式（KAIROS）中，强制 v1 路径无论 env-less 标志如何
   - 检查适当路径的最低版本
4. **访问令牌**：通过 `getBridgeAccessToken()` 验证桥接访问令牌存在

#### 连接流程
1. 检查是否已连接（`replBridgeConnected` 或 `replBridgeEnabled` 没有 `replBridgeOutboundOnly`）
2. 如果已连接：显示带有会话 URL 和选项的 `BridgeDisconnectDialog`
3. 如果未连接：
   - 运行先决条件检查
   - 如果 `shouldShowRemoteCallout()`：显示 onboarding callout 而不是连接
   - 否则：设置 `replBridgeEnabled: true`、`replBridgeExplicit: true`、`replBridgeOutboundOnly: false`
   - 显示"Remote Control connecting…"消息

#### 断开对话框
当已连接时，显示：
- 会话 URL（或连接 URL 如果会话尚未活跃）
- 三个选项：
  1. **Disconnect**：终止桥接连接
  2. **Show/Hide QR code**：为会话 URL 生成二维码
  3. **Continue**：关闭对话框并保持连接
- 键盘导航：上/下选择，回车接受，Esc 继续

#### Env-Less 桥接（v2）vs 传统（v1）
- **v2（env-less**）：当 `isEnvLessBridgeEnabled()` 为真且会话不是永久时使用
- **v1（传统）**：在助手模式（KAIROS 设置 `perpetual=true`）或 env-less 标志关闭时使用

---

## 远程命令概述

这三个命令管理远程会话生态系统：

```
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│  /web-setup  │     │ /remote-env  │     │ /remote-control│
│ (connect      │     │ (configure   │     │ (bridge CLI   │
│  GitHub to   │     │  remote      │     │  to claude.ai)│
│  claude.ai)  │     │  environment)│     │               │
└──────┬───────┘     └──────┬───────┘     └──────┬───────┘
       │                    │                    │
       └────────────────────┼────────────────────┘
                            │
              ┌─────────────┴─────────────┐
              │    Remote Session         │
              │    Infrastructure         │
              │  (teleport, bridge,       │
              │   environments)           │
              └───────────────────────────┘
```

### 先决条件链
1. **策略**：`allow_remote_sessions` 和 `allow_remote_control` 必须允许
2. **认证**：用户必须登录 Claude（OAuth）
3. **功能标志**：`tengu_cobalt_lantern` 用于 web-setup，`BRIDGE_MODE` 用于 remote-control
4. **GitHub**：通过 web-setup 连接用于 claude.ai/code 功能
5. **环境**：通过 remote-env 配置用于 teleport 会话
6. **桥接**：通过 remote-control 激活用于双向 CLI-网页通信
