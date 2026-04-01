# 清除和退出命令

## 目的

记录终止或重置会话状态的 CLI 命令：清除对话历史同时保持会话存活，以及完全退出 REPL。

---

## `/clear`（对话重置）

### 目的
清除当前对话历史并重置会话状态，同时保持 REPL 运行。提供全新上下文而不终止会话。

### 位置
`restored-src/src/commands/clear/index.ts`
`restored-src/src/commands/clear/clear.ts`
`restored-src/src/commands/clear/conversation.ts`
`restored-src/src/commands/clear/caches.ts`

### 命令定义
| 属性 | 值 |
|----------|-------|
| 名称 | `clear` |
| 别名 | `reset`, `new` |
| 类型 | `local` |
| 描述 | Clear conversation history and free up context |
| 支持非交互式 | `false`（创建新会话） |

### 关键导出

#### 来自 `index.ts`
- `clear` — 带延迟加载配置的命令元数据

#### 来自 `clear.ts`
- `call`: `LocalCommandCall` — 委托给 `clearConversation` 的入口点

#### 来自 `conversation.ts`
- `clearConversation(options)`: 主清除逻辑 — 重置消息、缓存、会话 ID 并触发生命周期钩子

#### 来自 `caches.ts`
- `clearSessionCaches(preservedAgentIds)`: 全面的会话缓存清除，不影响消息或会话 ID

### 实现细节

#### 清除序列
`clearConversation` 函数按顺序执行以下步骤：

**1. 会话结束钩子**
- 使用类型 `'clear'` 执行 `SessionEnd` 钩子
- 以 `CLAUDE_CODE_SESSIONEND_HOOKS_TIMEOUT_MS`（默认 1.5s）为界
- 使用 `AbortSignal.timeout` 强制执行

**2. 缓存清除提示**
- 记录带最后请求 ID 的 `tengu_cache_eviction_hint` 分析事件
- 向推理发出信号，表示此对话的缓存可以被驱逐

**3. 任务保留**
- 识别要保留的任务：后台任务和主会话任务
- 终止 `isBackgrounded === false` 的任务
- 保留的任务保留其每个 agent 状态（调用的技能、待处理回调、转储状态）

**4. 消息重置**
- `setMessages(() => [])` — 清除所有消息

**5. 上下文解除阻塞**
- 清除 PROACTIVE/KAIROS 功能的上下文阻塞标志
- 允许清除后恢复主动 tick

**6. 对话 ID 轮换**
- 通过 `randomUUID()` 生成新 UUID
- 强制 UI 重新渲染徽标

**7. 缓存清除**
- 调用 `clearSessionCaches(preservedAgentIds)` — 见下面的缓存清除部分
- 将工作目录重置为原始 cwd
- 清除文件状态缓存、发现的技能和加载的内存路径

**8. App 状态清理**
- 分区任务：杀死前台任务，保留后台任务
- 杀死运行中的 shell 任务（进程终止 + 清理）
- 通过 abort 控制器中止运行中的任务
- 将归属状态重置为空
- 清除 `standaloneAgentContext`（来自 `/rename`、`/color` 的名称/颜色）
- 清除文件历史（快照、跟踪文件、序列）
- 重置 MCP 状态（客户端、工具、命令、资源）同时保留 `pluginReconnectKey`

**9. 元数据和会话 ID 重置**
- 清除计划 slug 缓存
- 清除缓存的会话元数据（标题、标签、agent 名称/颜色）
- 通过 `setCurrentAsParent: true` 重新生成会话 ID 以进行分析血统追踪
- 更新 `CLAUDE_CODE_SESSION_ID` 环境变量（对于 `USER_TYPE === 'ant'`）
- 重置会话文件指针

**10. 任务输出符号链接修复**
- 重新指向保留的运行中本地 agent 任务的符号链接
- 确保 TaskOutput 从新会话目录读取，而不是冻结的清除前快照
- 主会话任务使用共享的每 agent 路径，因此不需要特殊处理

**11. 状态重新持久化**
- 如果启用则保存协调器模式状态
- 如果适用则保存 worktree 会话状态

**12. 会话开始钩子**
- 使用类型 `'clear'` 执行 `SessionStart` 钩子
- 使用钩子结果更新消息

#### 缓存清除（`clearSessionCaches`）
此函数是全面的缓存重置，清除：

| 类别 | 清除的缓存 |
|----------|---------------|
| 上下文 | 用户上下文、系统上下文、git 状态、会话开始日期 |
| 文件 | 文件建议缓存 |
| 命令 | 命令注册表、动态技能 |
| 提示 | 提示缓存中断检测、系统提示注入、转储状态、内存文件缓存 |
| 图像 | 存储的图像路径 |
| 会话 | 所有会话入口、会话环境变量、Tungsten 使用跟踪 |
| Swarm | 待处理权限回调 |
| 归属 | 文件内容缓存、待处理 bash 状态（功能标志） |
| 仓库 | 仓库检测缓存、git 目录解析缓存 |
| Bash | 命令前缀缓存（Haiku 提取） |
| 技能 | 调用的技能缓存、动态技能、发送的技能名称 |
| LSP | 诊断跟踪状态 |
| MagicDocs | 跟踪的 magic docs |
| WebFetch | URL 缓存（高达 50MB） |
| ToolSearch | 描述缓存（~500KB 用于 50 个 MCP 工具） |
| AgentTool | Agent 定义缓存 |
| SkillTool | 提示缓存 |

`preservedAgentIds` 参数允许选择性保留后台任务的每 agent 状态。当非空时：
- 按 agentId 键控的状态有选择地清除
- 按 requestId 键控的状态（待处理回调、转储状态、缓存中断跟踪）保持完整

#### 边缘情况
- 非交互式模式：`supportsNonInteractive` 为 `false`，所以 `/clear` 创建新会话
- 运行中的前台任务被杀死（shell 进程终止，abort 控制器触发）
- 后台任务在清除中存活并继续运行
- 压缩后清理有意运行，即使消息被清除（重置技能列表、内存文件）
- `pluginReconnectKey` 被保留，因此 `/clear` 不会导致无操作的 MCP 重连

### 数据流
```
User runs /clear
  → executeSessionEndHooks('clear')
  → logEvent('tengu_cache_eviction_hint')
  → Identify preserved tasks (backgrounded + main-session)
  → setMessages([])
  → Clear context-blocked flag
  → setConversationId(randomUUID())
  → clearSessionCaches(preservedAgentIds)
  → setCwd(originalCwd)
  → Clear file state, skills, memory paths
  → Kill foreground tasks, preserve background tasks
  → Reset attribution, standaloneAgentContext, fileHistory, MCP
  → clearAllPlanSlugs()
  → clearSessionMetadata()
  → regenerateSessionId(setCurrentAsParent: true)
  → resetSessionFilePointer()
  → Re-point task output symlinks
  → Save mode/worktree state
  → processSessionStartHooks('clear')
  → Update messages with hook results
```

### 集成点
- **生命周期钩子**：清除前后都触发 SessionEnd 和 SessionStart 钩子
- **任务系统**：后台任务存活；前台任务被终止
- **分析**：缓存驱逐提示和会话血统追踪
- **会话存储**：元数据、文件指针和 worktree 状态被重置
- **MCP 系统**：客户端被重置但重连密钥被保留

---

## `/exit`（REPL 终止）

### 目的
退出 REPL 会话，执行应用程序的优雅关闭。

### 位置
`restored-src/src/commands/exit/index.ts`
`restored-src/src/commands/exit/exit.tsx`

### 命令定义
| 属性 | 值 |
|----------|-------|
| 名称 | `exit` |
| 别名 | `quit` |
| 类型 | `local-jsx` |
| 描述 | Exit the REPL |
| Immediate | `true` |

### 关键导出

#### 函数
- `call(onDone)`: 退出命令的主入口点

#### 常量
- `GOODBYE_MESSAGES`: 随机告别消息数组 — `['Goodbye!', 'See ya!', 'Bye!', 'Catch you later!']`

### 实现细节

#### 核心逻辑 — 三种退出路径

**路径 1：后台会话（tmux 分离）**
在 `claude --bg` tmux 会话中运行时：
1. 调用 `onDone()` 不带消息
2. 通过 `spawnSync` 执行 `tmux detach-client`（stdio 忽略）
3. 返回 `null` — REPL 继续运行；`claude attach` 可以重新连接
4. 此路径覆盖 `/exit`、`/quit`、`Ctrl+C`、`Ctrl+D` — 都通过 `handleExit` 汇聚

**路径 2：Worktree 会话（确认流程）**
当在 worktree 会话中时（`getCurrentWorktreeSession() !== null`）：
1. 渲染带有 `showWorktree={true}` 的 `ExitFlow` 组件
2. `ExitFlow` 提供退出 worktree 的确认 UI
3. 取消：调用 `onDone()` 不带消息
4. 确认：继续优雅关闭

**路径 3：正常退出（优雅关闭）**
对于标准会话：
1. 从 `GOODBYE_MESSAGES` 选择随机告别消息
2. 调用 `onDone(goodbyeMessage)` 显示它
3. 等待 `gracefulShutdown(0, 'prompt_input_exit')` 进行干净终止
4. 返回 `null`

#### 优雅关闭
`gracefulShutdown` 函数（从 `../../utils/gracefulShutdown.js` 导入）处理：
- 会话持久化和最终状态保存
- 资源清理
- 进程终止

#### 边缘情况
- 后台会话分离而不是杀死，允许重新连接
- Worktree 会话需要确认以防止意外数据丢失
- 所有退出触发器（`/exit`、`/quit`、`Ctrl+C`、`Ctrl+D`）通过 REPL 的 `handleExit` 路由
- `immediate: true` 标志意味着命令执行而不等待待处理操作

### 数据流
```
User triggers exit (/exit, /quit, Ctrl+C, Ctrl+D)
  │
  ├─ BG session? → onDone() → tmux detach-client → REPL keeps running
  │
  ├─ Worktree? → Show ExitFlow → User confirms → gracefulShutdown
  │                        → User cancels → onDone() (no-op)
  │
  └─ Normal → Random goodbye → onDone(message) → gracefulShutdown(0, 'prompt_input_exit')
```

### 集成点
- **后台会话**：与 tmux 分离/附加生命周期集成
- **Worktree 模式**：使用 ExitFlow 组件进行安全的 worktree 退出确认
- **优雅关闭**：集中关闭工具处理所有清理
- **REPL 处理程序**：所有退出信号通过 REPL 的 `handleExit` 汇聚

---

## 清除 vs 退出比较

| 方面 | `/clear` | `/exit` |
|--------|----------|---------|
| **会话** | 继续使用新会话 ID | 完全终止 |
| **消息** | 完全清除 | 持久化到日志存储 |
| **任务** | 保留后台，杀死前台 | 通过关闭终止所有 |
| **钩子** | SessionEnd + SessionStart 触发 | 关闭期间 SessionEnd 触发 |
| **REPL** | 保持运行 | 退出 |
| **别名** | `reset`, `new` | `quit` |
| **类型** | `local` | `local-jsx` |
| **Immediate** | No | Yes |

---

## 相关模块
- [session-commands.md](./session-commands.md) — 会话生命周期管理命令
- [session-lifecycle.md](../09-data-flow/session-lifecycle.md) — 会话生命周期数据流
- [gracefulShutdown](../../restored-src/src/utils/gracefulShutdown.js) — 应用程序关闭工具
- [sessionStorage](../../restored-src/src/utils/sessionStorage.js) — 会话存储工具
