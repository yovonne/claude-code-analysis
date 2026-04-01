# MCP 和插件命令

## 目的

记录管理 MCP（Model Context Protocol）服务器、插件和插件生命周期的 CLI 命令。这些命令允许用户添加、配置、启用/禁用 MCP 服务器，浏览和管理插件，以及在不重启的情况下刷新插件状态。

---

## /mcp（MCP 服务器管理）

### 目的
管理 MCP 服务器 — 添加、启用、禁用、重新连接以及配置 OAuth/XAA 认证。还提供完整的 MCP 设置 UI。

### 位置
`restored-src/src/commands/mcp/index.ts`
`restored-src/src/commands/mcp/mcp.tsx`
`restored-src/src/commands/mcp/addCommand.ts`
`restored-src/src/commands/mcp/xaaIdpCommand.ts`

### 命令定义
| 属性 | 值 |
|----------|-------|
| 名称 | `mcp` |
| 类型 | `local-jsx` |
| 描述 | Manage MCP servers |
| Immediate | `true` |
| 参数提示 | `[enable\|disable [server-name]]` |

### 子命令（通过 Commander.js）

#### `mcp add <name> <commandOrUrl> [args...]`
将 MCP 服务器配置添加到 Claude Code。

| 选项 | 描述 |
|--------|-------------|
| `-s, --scope <scope>` | 配置范围：`local`、`user` 或 `project`（默认：`local`） |
| `-t, --transport <transport>` | 传输类型：`stdio`、`sse` 或 `http`（默认：`stdio`） |
| `-e, --env <env...>` | 设置环境变量（例如 `-e KEY=value`） |
| `-H, --header <header...>` | 设置 WebSocket 头（例如 `-H "Authorization: Bearer ..."`） |
| `--client-id <clientId>` | OAuth 客户端 ID（用于 HTTP/SSE 服务器） |
| `--client-secret` | 提示输入 OAuth 客户端密钥 |
| `--callback-port <port>` | OAuth 回调的固定端口 |
| `--xaa` | 为此服务器启用 XAA (SEP-990)（隐藏，除非启用 XAA） |

**传输处理：**
- **stdio**：`commandOrUrl` 被解释为带可选参数的可执行命令。如果参数看起来像 URL 但未明确设置 `--transport`，则发出警告。
- **sse/http**：`commandOrUrl` 被解释为 URL。支持头、OAuth 客户端 ID/密钥和回调端口配置。

#### `mcp xaa` — XAA (SEP-990) IdP 连接管理
管理用于 MCP 服务器认证的 XAA 身份提供者连接。

| 子命令 | 描述 |
|------------|-------------|
| `mcp xaa setup` | 配置 IdP 连接（一次性设置）。需要 `--issuer <url>` 和 `--client-id <id>`。可选：`--client-secret`、`--callback-port` |
| `mcp xaa login` | 缓存 IdP id_token 以进行静默认证。默认：运行 OIDC 浏览器登录。使用 `--id-token <jwt>`：直接写入预获取的 JWT。`--force` 忽略缓存的令牌 |
| `mcp xaa show` | 显示当前 IdP 配置（issuer、客户端 ID、回调端口、密钥状态、登录状态） |
| `mcp xaa clear` | 从设置和钥匙串中移除 IdP 配置和缓存的令牌 |

#### REPL 内子命令（通过 JSX args）
| 用法 | 行为 |
|-------|----------|
| `/mcp` | 打开 MCP 设置 UI（Ant 用户重定向到插件管理标签） |
| `/mcp no-redirect` | 绕过 Ant 重定向打开 MCP 设置 UI（用于测试） |
| `/mcp reconnect <server-name>` | 重新连接特定 MCP 服务器 |
| `/mcp enable [server-name]` | 启用所有 MCP 服务器或特定服务器 |
| `/mcp disable [server-name]` | 禁用所有 MCP 服务器或特定服务器 |

### 实现细节

#### MCP 切换逻辑
`MCPToggle` React 组件：
1. 从 app 状态读取 MCP 客户端（排除 `ide` 客户端）
2. 根据动作（启用 vs 禁用）和目标（所有 vs 特定服务器）过滤要切换的客户端
3. 为每个匹配的服务器调用 `toggleMcpServer(name)`
4. 通过 `onComplete` 回调报告结果

#### XAA 安全模型
- IdP 连接是用户级别的：配置一次，所有 XAA 启用的服务器重用
- 存储在 `settings.xaaIdp`（非秘密）+ 钥匙串槽（按 issuer 键控的秘密）
- 与每个服务器 AS（授权服务器）秘密的信任域分离
- 验证 issuer URL 必须是 HTTPS（或仅限回环的 HTTP）
- 验证回调端口是正整数
- issuer 或客户端 ID 更改时清除陈旧的钥匙串槽

---

## /plugin（插件管理）

### 目的
提供用于浏览、安装、配置和管理 Claude Code 插件及市场的完整交互式 UI。

### 位置
`restored-src/src/commands/plugin/index.tsx`
`restored-src/src/commands/plugin/plugin.tsx`

### 命令定义
| 属性 | 值 |
|----------|-------|
| 名称 | `plugin` |
| 别名 | `plugins`, `marketplace` |
| 类型 | `local-jsx` |
| 描述 | Manage Claude Code plugins |
| Immediate | `true` |

### 关键导出

#### 函数
- `call(onDone, _context, args?)`：入口点；使用可选参数渲染 `PluginSettings`

### 实现细节

#### 插件设置 UI
`PluginSettings` 组件通过 `args` 参数支持多个标签/模式：
- **Installed**：查看和管理当前安装的插件
- **Discover**：浏览可用插件
- **Manage**：完整插件管理（包括 Ant 用户的 MCP 重定向）
- **Marketplaces**：配置插件市场来源

#### 组件架构
插件命令由丰富的组件层次结构支持：
- `PluginSettings` — 带标签导航的主容器
- `ManagePlugins` — 带启用/禁用/移除的已安装插件列表
- `DiscoverPlugins` — 插件发现和安装
- `BrowseMarketplace` — 市场浏览
- `ManageMarketplaces` — 添加/移除市场来源
- `AddMarketplace` — 添加新的市场来源
- `PluginOptionsDialog` / `PluginOptionsFlow` — 插件配置向导
- `PluginTrustWarning` — 未信任插件的安全警告
- `PluginErrors` — 失败插件操作的错误显示
- `PluginDetailsHelpers` — 插件元数据渲染工具
- `UnifiedInstalledCell` — 已安装插件条目的统一显示
- `ValidatePlugin` — 插件验证逻辑
- `usePagination` — 大插件列表的分页钩子

---

## /reload-plugins（插件刷新）

### 目的
在不重启的情况下激活当前会话中的待处理插件更改。刷新插件、技能、agent、钩子、MCP 服务器和 LSP 服务器。

### 位置
`restored-src/src/commands/reload-plugins/index.ts`
`restored-src/src/commands/reload-plugins/reload-plugins.ts`

### 命令定义
| 属性 | 值 |
|----------|-------|
| 名称 | `reload-plugins` |
| 类型 | `local` |
| 描述 | Activate pending plugin changes in the current session |
| 非交互式 | `false`（仅交互式） |

### 关键导出

#### 函数
- `call(_args, context)`：刷新所有插件源并报告计数

### 实现细节

#### 核心逻辑
1. **CCR 模式**：如果 `DOWNLOAD_USER_SETTINGS` 功能启用且处于远程模式，则从远程后端重新下载用户设置。触发 `settingsChangeDetector.notifyChange('userSettings')` 以在会话中应用更改。
2. **插件刷新**：调用 `refreshActivePlugins(context.setAppState)`，重新加载：
   - 启用的插件
   - 命令技能
   - Agent
   - 钩子
   - 插件 MCP 服务器
   - 插件 LSP 服务器
3. **结果格式化**：使用 `n(count, noun)` 助手构建每个类别带计数的摘要字符串，正确使用复数形式
4. **错误报告**：如果任何插件加载失败，附加一条注释引导用户使用 `/doctor` 获取详情

#### 托管设置行为
托管设置**不会**在重新加载期间重新获取 — 它们已经每小时轮询一次（`POLLING_INTERVAL_MS`）。策略 enforcement 具有最终一致性设计，提取失败时使用陈旧缓存回退。

---

## MCP 和插件命令概述

```
┌─────────────┐  ┌─────────────┐  ┌─────────────┐
│    /mcp     │  │  /plugin    │  │/reload-     │
│   (MCP      │  │  (plugin    │  │ plugins     │
│  servers)   │  │  management)│  │ (refresh)   │
└─────────────┘  └─────────────┘  └─────────────┘
```

### 关键功能
- **MCP 服务器**：通过 stdio/SSE/HTTP 连接本地或远程服务器，支持 OAuth 和 XAA 认证
- **插件系统**：可扩展的插件架构，支持自定义技能、agent、钩子
- **动态刷新**：无需重启即可加载/卸载插件和技能
