# 提取记忆服务

## 目的

从当前会话脚本中提取持久记忆，并将其写入自动记忆目录。通过停止钩子在每个完整查询循环结束时运行（当模型生成没有工具调用的最终响应时）。使用分叉代理模式共享父级的提示缓存。

## 位置

`restored-src/src/services/extractMemories/`

### 关键文件

| 文件 | 描述 |
|------|-------------|
| `extractMemories.ts` | 主服务 — 提取编排、工具权限、闭包状态 |
| `prompts.ts` | 提取代理的提示模板（仅自动和组合变体） |

## 关键导出

### 函数 (extractMemories.ts)
- `initExtractMemories()`：使用新的闭包状态初始化提取系统（在启动时调用一次）
- `executeExtractMemories(context, appendSystemMessage)`：在查询循环结束时运行提取
- `drainPendingExtraction(timeoutMs)`：用软超时等待所有进行中的提取
- `createAutoMemCanUseTool(memoryDir)`：为分叉代理权限创建 canUseTool 函数

### 函数 (prompts.ts)
- `buildExtractAutoOnlyPrompt(newMessageCount, existingMemories, skipIndex)`：仅自动记忆的提示
- `buildExtractCombinedPrompt(newMessageCount, existingMemories, skipIndex)`：自动 + 团队记忆的提示

### 类型
- `AppendSystemMessageFn`：用于附加系统消息的函数类型

## 依赖

### 内部依赖
- `forkedAgent` — runForkedAgent 用于子代理执行
- `memdir/memoryScan` — scanMemoryFiles, formatMemoryManifest
- `memdir/paths` — getAutoMemPath, isAutoMemoryEnabled, isAutoMemPath
- `memdir/teamMemPaths` — 团队记忆路径（条件性，功能门控）
- `memdir/memoryTypes` — 记忆类型定义和示例
- `messages` — createUserMessage, createMemorySavedMessage
- `analytics/growthbook` — 功能标志（tengu_passport_quail, tengu_moth_copse, tengu_bramble_lintel）
- `analytics` — 遥测事件

### 外部依赖
- `bun:bundle` — feature() 构建标志
- `path` — 路径操作

## 实现细节

### 提取流程

1. 检查资格：仅主代理、功能门、启用自动记忆、非远程模式
2. 计算自上次提取游标以来的新模型可见消息数
3. 检查互斥：如果主代理已写入记忆文件则跳过
4. 扫描现有记忆文件以获取清单（预注入，节省代理回合）
5. 构建提示（基于团队记忆状态的仅自动或组合变体）
6. 使用受限工具运行分叉代理（最多 5 回合）
7. 推进游标，提取写入的文件路径，记录遥测
8. 如果写了文件则显示内联"保存了 N 个记忆"消息

### 闭包作用域状态

所有可变状态都存在于 `initExtractMemories()` 闭包内：
- `inFlightExtractions`：待处理提取承诺的集合
- `lastMemoryMessageUuid`：增量处理的游标 UUID
- `hasLoggedGateFailure`：一次性标志以避免重复日志泛滥
- `inProgress`：防止重叠的提取运行
- `turnsSinceLastExtraction`：节流计数器（可通过 tengu_bramble_lintel 配置）
- `pendingContext`：在进行中提取后为后续运行藏匿的上下文

### 互斥

当主代理自己写入记忆时，分叉提取被跳过，游标前进超过写入。这使得主代理和后台代理每回合互斥，避免冗余工作。

### 工具权限 (createAutoMemCanUseTool)

- **允许**：FileRead, Grep, Glob（无限制，只读）
- **允许**：Bash（仅只读命令 — ls/find/grep/cat/stat/wc/head/tail）
- **允许**：FileEdit/FileWrite（仅记忆目录内的路径）
- **允许**：REPL（传递到内部工具检查）
- **拒绝**：其他一切（MCP、代理、可写 Bash、rm）

### 合并和后续运行

当提取已经在进行中时：
- 传入调用藏匿它们的上下文（最新覆盖之前的）
- 当前提取完成后，后续运行处理藏匿的上下文
- 后续运行跳过节流检查（已是已提交的工作）

### 排水机制

- `drainPendingExtraction()` 用可配置超时（默认 60 秒）等待所有进行中的提取
- 由 print.ts 在响应刷新后但在 gracefulShutdownSync 之前调用
- 确保在 5 秒关闭安全罩之前完成分叉代理

### 提示变体

- **仅自动**：四类型分类法，无范围指导（单一目录）
- **组合**：四类型分类法，带每类型范围指导（私有 vs 团队目录）
- **skipIndex**：为 true 时，从指令中省略第 2 步（MEMORY.md 索引更新）

## 数据流

```
查询循环完成（无工具调用）
    ↓
handleStopHooks → executeExtractMemories()
    ↓
检查门：主代理？功能？启用？不是远程？
    ↓
计算自游标以来的新消息
    ↓
检查 hasMemoryWritesSince() — 如果主代理写了则跳过
    ↓
节流检查：turnsSinceLastExtraction >= 阈值
    ↓
scanMemoryFiles() → formatMemoryManifest()
    ↓
buildExtractPrompt() — 仅自动或组合
    ↓
runForkedAgent() — 最多 5 回合，受限工具
    ↓
推进游标，提取写入的路径
    ↓
createMemorySavedMessage() → appendSystemMessage()
    ↓
遥测：tengu_extract_memories_extraction
```

## 集成点

- **停止钩子**：从 handleStopHooks 火焰和忘记调用
- **打印模式**：`drainPendingExtraction()` 在 gracefulShutdownSync 之前调用
- **自动梦**：共享 `createAutoMemCanUseTool` 用于工具权限
- **团队记忆**：通过 feature('TEAMMEM') 标志条件支持

## 配置

- 功能标志：
  - `tengu_passport_quail`：提取的主门
  - `tengu_moth_copse`：跳过 MEMORY.md 索引步骤
  - `tengu_bramble_lintel`：回合节流（默认 1）
- 构建标志：`TEAMMEM` 用于团队记忆支持
- 最大回合：5（硬上限以防止验证兔子洞）
- 排水超时：60 秒（默认）

## 错误处理

- **尽力而为**：提取错误记录但不通知用户
- **游标保存**：错误时，游标保持不动以便重新考虑消息
- **合并调用**：藏匿的上下文被最新覆盖（只有最近的才重要）
- **门失败**：每会话记录一次（hasLoggedGateFailure 标志）

## 测试

- `initExtractMemories()` 在 beforeEach 中调用以获取新的闭包状态
- 所有可变状态都是闭包作用域的用于测试隔离
- 测试可以验证工具拒绝、消息计数和游标前进

## 相关模块

- [自动梦](./auto-dream-service.md) — 后台记忆整合
- [团队记忆同步](./team-memory-sync-service.md) — 团队记忆文件同步

## 备注

- 主代理的系统提示始终有完整的保存指令 — 仅当主代理没写时提取才运行
- 通过给予 fork 相同的工具列表来保留提示缓存共享（工具是缓存键的一部分）
- 索引文件更新（MEMORY.md）从"保存的记忆"计数中过滤 — 用户可见的记忆是主题文件
- 当 TEAMMEM 启用时，团队记忆计数在遥测中单独跟踪
