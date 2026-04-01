# 调试命令

## 目的

记录用于调试、性能分析和内部诊断的 CLI 命令。大多数这些命令对普通用户隐藏，用于开发或支持目的。

---

## /debug-tool-call

### 目的
调试工具调用功能的占位符。

### 位置
`restored-src/src/commands/debug-tool-call/index.js`

### 命令定义
| 属性 | 值 |
|----------|-------|
| 名称 | `stub` |
| 启用 | `false` |
| 隐藏 | `true` |

### 实现细节
此命令当前是一个禁用的存根：
```js
export default { isEnabled: () => false, isHidden: true, name: 'stub' };
```
调试工具调用功能在此版本代码库中未实现。

---

## /heapdump（堆转储）

### 目的
将 JavaScript 堆转储到用户的桌面用于内存分析和泄漏检测。

### 位置
`restored-src/src/commands/heapdump/index.ts`
`restored-src/src/commands/heapdump/heapdump.ts`

### 命令定义
| 属性 | 值 |
|----------|-------|
| 名称 | `heapdump` |
| 类型 | `local` |
| 描述 | Dump the JS heap to ~/Desktop |
| 非交互式 | `true` |
| 隐藏 | `true` |

### 关键导出

#### 函数
- `call()`：触发堆转储并返回文件路径

### 实现细节

#### 核心逻辑
1. 调用 `performHeapDump()`，创建：
   - `.heapsnapshot` 文件（用于 Chrome DevTools 的 V8 堆快照）
   - 诊断文件
2. 成功时返回文件路径
3. 失败时返回错误消息

#### 边缘情况
- 堆转储失败：返回描述性错误消息
- 文件写入到 `~/Desktop` 以便访问

### 数据流
```
User runs /heapdump
  → performHeapDump()
  │
  ├─ Success → return { type: 'text', value: '<heap-path>\n<diag-path>' }
  └─ Failure → return { type: 'text', value: 'Failed to create heap dump: <error>' }
```

---

## /ant-trace

### 目的
Ant 追踪功能的占位符。

### 位置
`restored-src/src/commands/ant-trace/index.js`

### 命令定义
| 属性 | 值 |
|----------|-------|
| 名称 | `stub` |
| 启用 | `false` |
| 隐藏 | `true` |

### 实现细节
此命令当前是一个禁用的存根：
```js
export default { isEnabled: () => false, isHidden: true, name: 'stub' };
```
Ant 追踪功能在此版本代码库中未实现。

---

## /ctx_viz（上下文可视化调试）

### 目的
上下文可视化调试的占位符。

### 位置
`restored-src/src/commands/ctx_viz/index.js`

### 命令定义
| 属性 | 值 |
|----------|-------|
| 名称 | `stub` |
| 启用 | `false` |
| 隐藏 | `true` |

### 实现细节
此命令当前是一个禁用的存根：
```js
export default { isEnabled: () => false, isHidden: true, name: 'stub' };
```
注意：交互式 `/context` 命令（记录在 interaction-commands.md 中）为最终用户提供上下文可视化。此存根是一个单独调试工具。

---

## /perf-issue（性能问题调试）

### 目的
性能问题调试的占位符。

### 位置
`restored-src/src/commands/perf-issue/index.js`

### 命令定义
| 属性 | 值 |
|----------|-------|
| 名称 | `stub` |
| 启用 | `false` |
| 隐藏 | `true` |

### 实现细节
此命令当前是一个禁用的存根：
```js
export default { isEnabled: () => false, isHidden: true, name: 'stub' };
```
性能问题调试功能在此版本代码库中未实现。

---

## /bughunter（Bug 狩猎调试）

### 目的
Bug 狩猎功能的占位符。

### 位置
`restored-src/src/commands/bughunter/index.js`

### 命令定义
| 属性 | 值 |
|----------|-------|
| 名称 | `stub` |
| 启用 | `false` |
| 隐藏 | `true` |

### 实现细节
此命令当前是一个禁用的存根：
```js
export default { isEnabled: () => false, isHidden: true, name: 'stub' };
```
Bug 狩猎功能在此版本代码库中未实现。

---

## 调试命令概述

六个调试命令中，只有 `/heapdump` 实现了。其余五个是保留的存根：

| 命令 | 状态 | 目的 |
|---------|--------|---------|
| `/heapdump` | 已实现 | 用于内存分析的 JS 堆快照 |
| `/debug-tool-call` | 存根 | 保留用于工具调用调试 |
| `/ant-trace` | 存根 | 保留用于 Ant 追踪 |
| `/ctx_viz` | 存根 | 保留用于上下文可视化调试 |
| `/perf-issue` | 存根 | 保留用于性能问题调试 |
| `/bughunter` | 存根 | 保留用于 bug 狩猎 |

所有存根命令遵循相同模式：
```js
export default { isEnabled: () => false, isHidden: true, name: 'stub' };
```

这确保它们不会出现在帮助输出中、无法调用，并且在保留命令名称以供将来实现的同时不会干扰命令系统。
