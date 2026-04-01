# Ultraplan 工具

## 目的

`/ultraplan` 特性的工具 — 轮询远程 CCR（Claude Code Remote）会话以获取计划批准，扫描事件流以查找 ExitPlanMode 工具结果，并在用户输入中检测"ultraplan"/"ultrareview"关键词触发器。

## 位置

- `restored-src/src/utils/ultraplan/ccrSession.ts` — 用于计划批准的 CCR 会话轮询
- `restored-src/src/utils/ultraplan/keyword.ts` — ultraplan/ultrareview 的关键词触发检测

## 主要导出

### CCR 会话轮询（`ccrSession.ts`）

#### 类型
- `PollFailReason`: `'terminated' | 'timeout_pending' | 'timeout_no_plan' | 'extract_marker_missing' | 'network_or_unknown' | 'stopped'`
- `UltraplanPollError`: 带 `reason`（PollFailReason）和 `rejectCount` 属性的自定义错误
- `ScanResult`: 判别联合 — `{ kind: 'approved', plan } | { kind: 'teleport', plan } | { kind: 'rejected', id } | { kind: 'pending' } | { kind: 'terminated', subtype } | { kind: 'unchanged' }`
- `UltraplanPhase`: `'running' | 'needs_input' | 'plan_ready'` — 状态机阶段
- `PollResult`: `{ plan, rejectCount, executionTarget: 'local' | 'remote' }`

#### 常量
- `ULTRAPLAN_TELEPORT_SENTINEL`: `'__ULTRAPLAN_TELEPORT_LOCAL__'` — 浏览器 PlanModal 反馈中的标记，当用户点击"teleport back to terminal"时

#### 类
- `ExitPlanModeScanner`: CCR 事件流的纯有状态分类器。摄取 `SDKMessage[]` 批次并返回当前 ExitPlanMode 裁定。无 I/O、无计时器 — 用合成或记录的事件进行单元测试。
  - `ingest(newEvents)`: 处理新事件并返回 ScanResult
  - `hasPendingPlan`: 当存在带工具结果但尚无 ExitPlanMode tool_use 时为 true
  - `rejectCount`: 唯一被拒绝的计划 ID 数量

#### 函数
- `pollForApprovedExitPlanMode(sessionId, timeoutMs, onPhaseChange?, shouldStop?)`: 轮询远程会话以获取批准的 ExitPlanMode 结果。返回批准的计划文本和执行目标。失败时抛出 `UltraplanPollError`。
- `extractTeleportPlan(content)`: 在 ULTRAPLAN_TELEPORT_SENTINEL 标记后提取计划文本。当标记不存在时返回 null。
- `extractApprovedPlan(content)`: 从 "## Approved Plan:" 或 "## Approved Plan (edited by user):" 标记提取计划文本。

#### 轮询行为
- 每 3 秒轮询一次，瞬态网络错误最多容忍 5 次连续失败
- 阶段转换：`running` → `needs_input`（空闲且无待处理计划）→ `plan_ready`（ExitPlanMode 已发出，等待结果）
- 处理"approved"（远程执行）和"teleport"（本地执行，存档远程）两种结果
- 正常拒绝被跟踪和跳过，以便用户可以在浏览器中迭代
- CCR 在工具轮之间短暂翻转到 'idle' — 仅在无新事件到达时信任

### 关键词检测（`keyword.ts`）

#### 类型
- `TriggerPosition`: `{ word, start, end }` — 关键词触发器在文本中的位置

#### 函数
- `findUltraplanTriggerPositions(text)`: 找到作为实际启动指令的"ultraplan"关键词位置
- `findUltrareviewTriggerPositions(text)`: 找到"ultrareview"关键词位置
- `hasUltraplanKeyword(text)`: 检查文本是否包含可触发的"ultraplan"
- `hasUltrareviewKeyword(text)`: 检查文本是否包含可触发的"ultrareview"
- `replaceUltraplanKeyword(text)`: 将第一个可触发的"ultraplan"替换为"plan"（保留后缀的大小写）

#### 过滤规则（什么不触发）
- 在成对分隔符内：反引号、双引号、尖括号、大括号、方括号、圆括号
- 单引号仅在不是撇号时作为分隔符（前面有非单词字符的起始引号）
- 路径/标识符上下文：前面或后面有 `/`、`\` 或 `-`，或后面有 `.` + 单词字符（文件扩展名）
- 后面有 `?`：关于特性的问题不应调用它
- 斜杠命令输入：以 `/` 开头的文本是斜杠命令调用

## 设计注意事项

- `ExitPlanModeScanner` 是一个纯分类器 — 无 I/O、无计时器 — 使得用合成或记录的事件进行测试非常简单
- 扫描中的优先级：approved > terminated > rejected > pending > unchanged。批次可能同时包含批准的 tool_result 和后续的 `{type:'result'}`（用户批准了，然后远程崩溃）— 批准的计划是真实的，不应丢弃
- teleaport  sentinel 允许用户批准计划但请求本地执行而不是远程 — 计划文本嵌入在拒绝反馈中
- 关键词检测镜像 `findThinkingTriggerPositions`（thinking.ts），以便 PromptInput 统一处理两种触发类型
- `replaceUltraplanKeyword` 保留用户"plan"后缀的大小写 — "Ultraplan" → "Plan"，"ULTRAPLAN" → "PLAN"
