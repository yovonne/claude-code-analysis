# TeamCreateTool

## 目的

TeamCreateTool 创建一个新的多 agent swarm 团队来协调并行工作。它同时建立团队配置文件和相应的任务列表目录，使团队 lead 能够生成队友、分配任务和管理协作工作。此工具仅在启用 agent swarms 时可用。

## 位置

- `restored-src/src/tools/TeamCreateTool/TeamCreateTool.ts` — 主工具定义（241 行）
- `restored-src/src/tools/TeamCreateTool/prompt.ts` — 带团队工作流说明的工具提示（114 行）
- `restored-src/src/tools/TeamCreateTool/constants.ts` — 工具名称常量
- `restored-src/src/tools/TeamCreateTool/UI.tsx` — 工具使用消息的 UI 渲染

## 关键导出

| 导出 | 描述 |
|--------|-------------|
| `TeamCreateTool` | 通过 `buildTool()` 构建的完整工具定义 |
| `Input` | Zod 推断的输入类型：`{ team_name, description?, agent_type? }` |
| `Output` | 输出类型：`{ team_name, team_file_path, lead_agent_id }` |

## 输入/输出模式

### 输入模式

```typescript
{
  team_name: string,       // 必需：新团队的名称
  description?: string,    // 可选：团队描述/目的
  agent_type?: string,     // 可选：团队 lead 的类型/角色
}
```

### 输出模式

```typescript
{
  team_name: string,         // 最终团队名称（如果名称已存在可能不同）
  team_file_path: string,    // 团队配置文件的路径
  lead_agent_id: string,     // 团队 lead 的确定性 agent ID
}
```

## 团队创建流程

### 执行管道

```
1. 接收输入
   - Model 返回 TeamCreate tool_use，包含 { team_name, description?, agent_type? }

2. 验证
   - validateInput()：确保 team_name 非空

3. 预检查
   - 验证尚未领导团队（每个 leader 一个团队）
   - 检查团队名称是否已存在；如果存在，通过 generateWordSlug() 生成唯一名称

4. 创建团队文件
   - 生成确定性 lead agent ID：formatAgentId(TEAM_LEAD_NAME, teamName)
   - 使用 lead member、session ID、model、cwd 构建 TeamFile 对象
   - 将团队文件写入 ~/.claude/teams/{team-name}/config.json
   - 注册团队以便在会话结束时清理

5. 任务列表设置
   - 重置任务列表（编号从 1 开始）
   - 在 ~/.claude/tasks/{team-name}/ 创建任务目录
   - 设置 leader 团队名称，以便 getTaskListId() 返回正确的目录

6. 应用状态更新
   - 使用团队名称、路径、lead ID 和队友映射更新 AppState.teamContext
   - 通过 assignTeammateColor() 分配队友颜色

7. 分析
   - 记录带团队元数据的 'tengu_team_created' 事件

8. 返回
   - 返回 { team_name, team_file_path, lead_agent_id }
```

### 唯一名称生成

如果请求的团队名称已存在，`generateUniqueTeamName()` 使用 `generateWordSlug()` 生成新的唯一名称，而不是失败。这可以防止命名冲突，同时允许用户继续。

## 团队文件结构

团队文件（`~/.claude/teams/{team-name}/config.json`）包含：

```typescript
{
  name: string,           // 团队名称
  description?: string,   // 团队描述
  createdAt: number,      // 时间戳
  leadAgentId: string,    // 确定性 lead agent ID
  leadSessionId: string,  // 用于团队发现的 session ID
  members: [{
    agentId: string,      // 唯一 agent 标识符
    name: string,         // 人类可读名称（例如 "team-lead"）
    agentType: string,    // agent 的角色/类型
    model: string,        // 使用的模型
    joinedAt: number,     // 时间戳
    tmuxPaneId: string,   // tmux 窗格（lead 为空）
    cwd: string,          // 当前工作目录
    subscriptions: [],    // 事件订阅
  }]
}
```

## 团队工作流

工具提示定义了 7 步团队工作流：

1. **创建团队** 使用 TeamCreate — 同时创建团队和任务列表
2. **创建任务** 使用 Task 工具 — 自动使用团队的任務列表
3. **生成队友** 使用 Agent 工具，附带 `team_name` 和 `name` 参数
4. **分配任务** 使用 TaskUpdate，附带 `owner` 参数
5. **队友工作** 分配到的任务，并通过 TaskUpdate 标记完成
6. **队友空闲** 在回合之间（正常行为，不是错误）
7. **关闭团队** 通过 SendMessage，附带 `message: {type: "shutdown_request"}`

## 队友 Agent 类型

生成队友时，agent 类型决定可用工具：

- **只读 agent**（Explore、Plan）— 不能编辑/写入文件；只能研究/搜索/规划
- **全功能 agent**（general-purpose）— 访问所有工具，包括文件编辑和 bash
- **自定义 agent**（来自 `.claude/agents/`）— 可能有限制工具的自定义

## 任务列表协调

团队共享 `~/.claude/tasks/{team-name}/` 中的任务列表：

1. 队友定期检查 TaskList，特别是在完成任务后
2. 通过 TaskUpdate（设置 `owner`）认领未分配、未阻塞的任务
3. 按 ID 顺序优先处理任务（先处理低的）以进行上下文设置
4. 通过 TaskCreate 在识别到额外工作时创建新任务
5. 通过 TaskUpdate 标记任务完成
6. 如果所有任务被阻塞，通知团队 lead

## 通信规则

- 队友的消息**自动送达** — 无需手动检查收件箱
- 空闲通知是自动的且正常的 — 不要将空闲视为错误
- 始终按**名称**（而非 UUID）引用队友以进行消息传递和任务分配
- 不要发送结构化 JSON 状态消息 — 使用纯文本和 TaskUpdate
- 不要使用终端工具查看团队活动 — 始终使用 SendMessage

## 约束

- 每个 leader 一次只能有一个团队
- 团队名称必须非空
- 团队注册为在会话结束时自动清理
- 每个新团队的任務列表编号重置为 1
- 团队 lead 不被视为"队友"（isTeammate() 返回 false）

## 依赖

| 模块 | 目的 |
|--------|---------|
| `utils/swarm/teamHelpers.ts` | 团队文件读写、路径解析、清理注册 |
| `utils/swarm/teammateLayoutManager.ts` | 队友颜色分配 |
| `utils/tasks.ts` | 任务列表重置、目录创建、leader 团队名称 |
| `utils/agentId.ts` | Agent ID 格式化 |
| `utils/model/model.ts` | 团队 lead 的模型解析 |
| `utils/cwd.ts` | 当前工作目录 |
| `utils/words.ts` | 唯一单词 slug 生成 |
| `services/analytics/` | 事件记录 |
