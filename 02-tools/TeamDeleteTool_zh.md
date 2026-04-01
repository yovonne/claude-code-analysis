# TeamDeleteTool

## 目的

TeamDeleteTool 在 swarm 工作完成后清理团队和任务目录。它删除团队配置、任务列表，并清除当前会话中的团队上下文。如果仍有活跃队友，该工具会阻止删除，确保在清理前进行优雅关闭。

## 位置

- `restored-src/src/tools/TeamDeleteTool/TeamDeleteTool.ts` — 主工具定义（140 行）
- `restored-src/src/tools/TeamDeleteTool/prompt.ts` — 工具提示（17 行）
- `restored-src/src/tools/TeamDeleteTool/constants.ts` — 工具名称常量
- `restored-src/src/tools/TeamDeleteTool/UI.tsx` — 工具使用/结果消息的 UI 渲染

## 关键导出

| 导出 | 描述 |
|--------|-------------|
| `TeamDeleteTool` | 通过 `buildTool()` 构建的完整工具定义 |
| `Input` | 空输入模式 — 团队名称从会话上下文自动检测 |
| `Output` | 输出类型：`{ success, message, team_name? }` |

## 输入/输出模式

### 输入模式

```typescript
{}  // 无输入参数 — 团队名称从会话自动检测
```

### 输出模式

```typescript
{
  success: boolean,       // 清理是否成功
  message: string,        // 人类可读的结果消息
  team_name?: string,     // 被清理的团队名称
}
```

## 清理流程

### 执行管道

```
1. 接收输入
   - Model 返回 TeamDelete tool_use，附带空输入

2. 团队检测
   - 从 AppState.teamContext.teamName 读取团队名称

3. 活跃成员检查
   - 读取团队配置文件
   - 过滤掉团队 lead — 只计算非 lead 成员
   - 过滤掉空闲/死亡成员（isActive === false）
   - 如果仍有活跃成员：返回失败并列出成员名称

4. 目录清理
   - 删除团队目录（~/.claude/teams/{team-name}/）
   - 删除任务目录（~/.claude/tasks/{team-name}/）
   - 从会话结束清理中取消注册团队（已清理）

5. 状态重置
   - 清除队友颜色分配（clearTeammateColors）
   - 清除 leader 团队名称（clearLeaderTeamName）
   - 从 AppState 清除团队上下文
   - 清除收件箱消息

6. 分析
   - 记录 'tengu_team_deleted' 事件

7. 返回
   - 返回 { success, message, team_name }
```

## 活跃成员保护

如果团队仍有活跃（非空闲）非 lead 成员，工具将**失败**：

```
{
  success: false,
  message: "Cannot cleanup team with N active member(s): name1, name2. Use requestShutdown to gracefully terminate teammates first.",
  team_name: "team-name"
}
```

这确保队友在其资源被清理前被优雅终止。`isActive === false`（空闲或崩溃）的成员不计入此检查。

## 清理操作

| 操作 | 描述 |
|-----------|-------------|
| `cleanupTeamDirectories(teamName)` | 从磁盘删除团队和任务目录 |
| `unregisterTeamForSessionCleanup(teamName)` | 防止在会话结束时重复清理 |
| `clearTeammateColors()` | 重置颜色分配以重新开始 |
| `clearLeaderTeamName()` | 重置 getTaskListId() 以回退到 session ID |

## AppState 更改

成功清理后，以下 AppState 字段被清除：

```typescript
{
  teamContext: undefined,     // 无活跃团队
  inbox: { messages: [] },    // 清除排队的消息
}
```

## 使用场景

- 所有队友已完成工作并关闭
- Swarm 项目完成
- 想要清理持久的团队和任务目录
- 启动新团队（必须先删除当前团队）

## 约束

- TeamDelete 如果活跃成员仍存在将失败
- 队友必须先通过 SendMessage shutdown_request 优雅终止
- 无输入参数 — 团队名称从当前会话上下文确定
- UI 抑制清理结果显示（批处理关闭消息覆盖它）

## 依赖

| 模块 | 目的 |
|--------|---------|
| `utils/swarm/teamHelpers.ts` | 团队文件读取、目录清理、清理注册 |
| `utils/swarm/teammateLayoutManager.ts` | 队友颜色管理 |
| `utils/tasks.ts` | Leader 团队名称清除 |
| `utils/swarm/constants.ts` | TEAM_LEAD_NAME 常量 |
| `services/analytics/` | 事件记录 |
