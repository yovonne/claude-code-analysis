# 自动梦服务

## 目的

后台记忆整合。当足够的时间过去且积累了足够的会话时，自动作为分叉子代理触发 /dream 提示。将近期会话学习整合到持久的、有组织的记忆文件中。

## 位置

`restored-src/src/services/autoDream/`

### 关键文件

| 文件 | 描述 |
|------|-------------|
| `autoDream.ts` | 主要编排器 — 门检查、分叉代理执行、任务管理 |
| `config.ts` | 叶子配置模块 — 来自设置的启用状态 + GrowthBook |
| `consolidationLock.ts` | 锁文件管理 — 基于 mtime 的调度、PID 所有权 |
| `consolidationPrompt.ts` | 整合代理的提示模板构建器 |

## 关键导出

### 函数 (autoDream.ts)
- `initAutoDream()`：初始化自动梦运行器（在启动或每次测试时调用一次）
- `executeAutoDream(context, appendSystemMessage)`：来自 stopHooks 的入口点 — 在 init 之前是空操作

### 函数 (config.ts)
- `isAutoDreamEnabled()`：检查自动梦是否启用（用户设置覆盖 GrowthBook 默认值）

### 函数 (consolidationLock.ts)
- `readLastConsolidatedAt()`：获取锁文件的 mtime (= lastConsolidatedAt)，如果不存在则为 0
- `tryAcquireConsolidationLock()`：获取锁 — 写入 PID，如果被阻止则返回先前的 mtime 或 null
- `rollbackConsolidationLock(priorMtime)`：在分叉失败后回滚 mtime
- `recordConsolidation()`：从手动 /dream 标记锁（乐观的，尽力而为）
- `listSessionsTouchedSince(sinceMs)`：获取 mtime 在 sinceMs 之后的会话 ID

### 函数 (consolidationPrompt.ts)
- `buildConsolidationPrompt(memoryRoot, transcriptDir, extra)`：构建完整的整合提示

### 类型
- `AutoDreamConfig`：`{ minHours: number, minSessions: number }`

### 常量
- `DEFAULTS`：`{ minHours: 24, minSessions: 5 }`
- `SESSION_SCAN_INTERVAL_MS`：10 分钟（扫描节流）

## 依赖

### 内部依赖
- `forkedAgent` — runForkedAgent 用于子代理执行
- `DreamTask` — 任务注册、进度跟踪、中止处理
- `extractMemories` — createAutoMemCanUseTool（共享工具权限）
- `memdir/paths` — getAutoMemPath, isAutoMemoryEnabled
- `bootstrap/state` — getKairosActive, getIsRemoteMode, getSessionId, getOriginalCwd
- `analytics/growthbook` — tengu_onyx_plover 功能标志
- `analytics` — 遥测事件

### 外部依赖
- `fs/promises` — 锁文件 I/O
- `path` — 路径操作

## 实现细节

### 门顺序（最便宜优先）

1. **功能门**：非 KAIROS 模式、非远程模式、启用自动记忆、启用自动梦
2. **时间门**：自 lastConsolidatedAt 以来的小时数 >= minHours（一次统计调用）
3. **扫描节流**：距离上次会话扫描至少 10 分钟
4. **会话门**：mtime > lastConsolidatedAt 的脚本数量 >= minSessions
5. **锁**：没有其他进程正在整合（基于 PID 的锁）

### 锁文件设计

- 锁文件：内存目录内的 `.consolidate-lock`
- 内容：持有者的 PID
- mtime：lastConsolidatedAt（用于时间门）
- 陈旧阈值：1 小时（HOLDER_STALE_MS）
- 回收：死 PID 或不可解析的内容 → 回收
- 竞争处理：两个回收者都写入 → 最后写入 PID 获胜，失败者在重新读取时退出
- 回滚：在分叉失败时，将 mtime 回滚到获取前的值

### 分叉代理执行

1. 使用会话计数和中止控制器注册 DreamTask
2. 使用内存根、脚本目录和工具约束构建整合提示
3. 使用以下条件运行分叉代理：
   - 只读工具约束（Bash 限于 ls/find/grep/cat/stat/wc/head/tail）
   - 仅内存目录写入权限
   - 跳过脚本（无录制）
   - 用于 UI 更新的进度观察器
4. 成功：完成任务，显示内联完成摘要
5. 失败：任务失败，回滚锁，记录事件

### 进度观察器

- 从助手消息中提取文本块和 tool_use 计数
- 从 Edit/Write 工具调用中收集文件路径
- 通过 addDreamTurn 更新 DreamTask 状态

### 配置

- 来自 GrowthBook 标志 `tengu_onyx_plover` 的 `minHours` 和 `minSessions`
- 用户设置 `autoDreamEnabled` 在明确设置时覆盖 GrowthBook
- 防御性逐字段验证（陈旧的 GB 缓存可能返回错误的类型）

## 数据流

```
后采样钩子
    ↓
executeAutoDream()
    ↓
门检查：功能 → 时间 → 节流 → 会话 → 锁
    ↓
注册 DreamTask
    ↓
buildConsolidationPrompt()
    ↓
runForkedAgent() — 只读工具，仅内存目录写入
    ↓
进度观察器 → addDreamTurn() → UI 更新
    ↓
成功：completeDreamTask() + 内联摘要
失败：failDreamTask() + rollbackConsolidationLock()
```

### 整合提示阶段

1. **定向**：ls 内存目录，读取 MEMORY.md，浏览现有文件
2. **收集**：扫描每日日志，检查漂移的记忆，grep 脚本
3. **整合**：写入/更新记忆文件，合并信号，修复矛盾
4. **修剪和索引**：更新 MEMORY.md（少于 200 行，约 25KB），删除陈旧条目

## 集成点

- **后采样钩子**：`executeAutoDream()` 从 backgroundHousekeeping 调用
- **手动 /dream**：通过 `recordConsolidation()` 使用相同的锁机制
- **DreamTask UI**：进度在后台任务对话框中可见
- **提取记忆**：共享 `createAutoMemCanUseTool` 用于工具权限

## 配置

- GrowthBook 标志：`tengu_onyx_plover`（启用、minHours、minSessions）
- 用户设置：`settings.json` 中的 `autoDreamEnabled`
- 默认值：24 小时，5 个会话
- 扫描节流：会话扫描间隔 10 分钟
- 锁陈旧阈值：1 小时

## 错误处理

- **门失败**：静默返回 — 正常跳过路径不记录
- **锁获取失败**：静默返回 — 另一个进程正在整合
- **分叉失败**：回滚锁，任务失败，记录错误事件
- **用户中止**：DreamTask.kill 处理中止，锁回滚，不会双重回滚
- **强制模式**：绕过启用/时间/会话门（测试覆盖，不对外暴露）

## 测试

- 在 beforeEach 中调用 `initAutoDream()` 以获取新的闭包作用域状态
- 状态是闭包作用域的（不是模块级别）用于测试隔离
- `isForced()` 是构建时测试覆盖（生产中始终返回 false）

## 相关模块

- [提取记忆](./extract-memories-service.md) — 记忆提取代理
- [分析](./analytics-service.md) — 遥测事件

## 备注

- 从 dream.ts 中提取，以便自动梦独立于 KAIROS 功能标志发货
- 锁文件 mtime 是 lastConsolidatedAt — 聪明的双重用途设计
- 会话扫描使用 mtime（触碰的会话），而不是 birthtime（ext4 上为 0）
- 分叉代理共享父级的提示缓存以提高效率
- 手动 /dream 使用乐观锁标记（无后技能完成钩子）
