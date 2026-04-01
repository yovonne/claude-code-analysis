# 计算机使用

## 目的

实现 Claude Code 的计算机使用功能（代号"Chicago"）——一个基于 MCP 的系统，允许 Claude 通过截图、鼠标和键盘输入控制用户的 macOS 桌面。涵盖完整技术栈：原生模块加载、CLI 执行器实现、会话上下文绑定、权限对话框、基于文件的锁定、Escape 热键中止、CFRunLoop 泵送、应用名称清理、工具渲染和回合结束清理。

## 位置

核心基础设施：
- `restored-src/src/utils/computerUse/common.ts` — 共享常量、终端包标识符检测、能力
- `restored-src/src/utils/computerUse/executor.ts` — CLI `ComputerExecutor` 实现（鼠标、键盘、截图、应用管理）
- `restored-src/src/utils/computerUse/wrapper.tsx` — `.call()` 覆盖适配器，位于 ToolUseContext 和 MCP 包之间
- `restored-src/src/utils/computerUse/hostAdapter.ts` — 进程生命周期单例主机适配器
- `restored-src/src/utils/computerUse/setup.ts` — 动态 MCP 配置和允许的工具名称设置
- `restored-src/src/utils/computerUse/mcpServer.ts` — 进程内 MCP 服务器用于 ListTools

原生模块加载器：
- `restored-src/src/utils/computerUse/inputLoader.ts` — `@ant/computer-use-input`（Rust/enigo — 鼠标、键盘）的延迟加载器
- `restored-src/src/utils/computerUse/swiftLoader.ts` — `@ant/computer-use-swift`（SCContentFilter 截图、NSWorkspace 应用、TCC）的延迟加载器

生命周期和安全性：
- `restored-src/src/utils/computerUse/computerUseLock.ts` — 用于跨会话互斥的基于文件的锁定（O_EXCL）
- `restored-src/src/utils/computerUse/cleanup.ts` — 回合结束清理：自动取消隐藏应用、注销 Escape、释放锁定
- `restored-src/src/utils/computerUse/drainRunLoop.ts` — libuv 下 @MainActor Swift 方法的共享 CFRunLoop 泵
- `restored-src/src/utils/computerUse/escHotkey.ts` — 通过 CGEventTap 全局 Escape → 中止

UI 和门控：
- `restored-src/src/utils/computerUse/toolRendering.tsx` — 终端中 `mcp__computer-use__*` 工具的渲染覆盖
- `restored-src/src/utils/computerUse/appNames.ts` — 应用名称过滤和提示注入加固
- `restored-src/src/utils/computerUse/gates.ts` — 特性门控、订阅检查、坐标模式

## 主要导出

### 公共（`common.ts`）

#### 常量
- `COMPUTER_USE_MCP_SERVER_NAME`: `'computer-use'`
- `CLI_HOST_BUNDLE_ID`: 哨兵包 ID（`'com.anthropic.claude-code.cli-no-window'`）— 从不匹配真实前台应用
- `CLI_CU_CAPABILITIES`: `{ screenshotFiltering: 'native', platform: 'darwin' }`

#### 函数
- `getTerminalBundleId()`: 从 `__CFBundleIdentifier` 环境变量或后备表检测终端模拟器的包 ID
- `isComputerUseMCPServer(name)`: 检查 MCP 服务器名称是否匹配 computer-use 服务器

### 执行器（`executor.ts`）

CLI `ComputerExecutor` 实现包装两个原生模块。与 Electron（Cowork）参考的主要区别：
- 无 `withClickThrough` — 终端没有窗口
- 终端作为替代主机 — 通过 `getTerminalBundleId()` 检测，免于隐藏并在激活 z-order 遍历中跳过，且从截图中排除
- 通过 `pbcopy`/`pbpaste` 实现剪贴板而非 Electron 剪贴板

#### 工厂
- `createCliExecutor(opts)`: 创建 `ComputerExecutor` 实例。在非 darwin 平台上抛出错误。

#### 执行器方法
- `prepareForAction(allowlistBundleIds, displayId?)`: 隐藏不允许的应用，激活目标应用。返回隐藏的应用 ID
- `previewHideSet(allowlistBundleIds, displayId?)`: 预览哪些应用会被隐藏
- `getDisplaySize(displayId?)`、`listDisplays()`、`findWindowDisplays(bundleIds)`: 显示几何查询
- `resolvePrepareCapture(opts)`: 解析显示，可选隐藏，捕获截图
- `screenshot(opts)`: 捕获排除终端的截图，预调整为 API 目标尺寸
- `zoom(regionLogical, allowedBundleIds, displayId?)`: 捕获屏幕区域
- `key(keySequence, repeat?)`: xdotool 风格的按键序列（例如 "ctrl+shift+a"）
- `holdKey(keyNames, durationMs)`: 按住按键持续一段时间
- `type(text, opts)`: 输入文本，可选通过剪贴板（保存 → 写入 → 验证 → Cmd+V → 恢复）
- `readClipboard()`、`writeClipboard(text)`: 通过 pbpaste/pbcopy 实现剪贴板
- `moveMouse(x, y)`、`click(x, y, button, count, modifiers?)`、`mouseDown()`、`mouseUp()`、`getCursorPosition()`: 鼠标操作
- `drag(from?, to)`: 拖动（60fps 缓出立方动画）
- `scroll(x, y, dx, dy)`: 在位置滚动
- `getFrontmostApp()`、`appUnderPoint(x, y)`、`listInstalledApps()`、`getAppIcon(path)`、`listRunningApps()`、`openApp(bundleId)`: 应用管理

#### 模块导出
- `unhideComputerUseApps(bundleIds)`: 取消隐藏回合期间隐藏的应用（在回合结束时调用）

### 包装器（`wrapper.tsx`）

将 MCP 包的 `bindSessionContext` 绑定到 Claude Code 的 `ToolUseContext`。

#### 函数
- `buildSessionContext()`: 构建 `ComputerUseSessionContext` 对象，包含 AppState 的读写回调、锁定管理和权限对话框
- `getOrBind()`: 获取或创建缓存绑定（构建一次，在进程生命周期内重用）
- `getComputerUseMCPToolOverrides(toolName)`: 获取 MCP 工具的完整覆盖对象（渲染 + `.call()`）
- `runPermissionDialog(req)`: 通过 `setToolJSX` + Promise 渲染批准对话框并等待用户响应

### 主机适配器（`hostAdapter.ts`）

#### 函数
- `getComputerUseHostAdapter()`: 进程生命周期单例。首次 CU 工具调用时构建一次。包括日志器、执行器、OS 权限检查、子门控和禁用检查。

### 设置（`setup.ts`）

#### 函数
- `setupComputerUseMCP()`: 构建动态 MCP 配置 + 允许的工具名称。创建 stdio 服务器配置，按名称拦截（实际不生成）。

### MCP 服务器（`mcpServer.ts`）

#### 函数
- `createComputerUseMcpServerForCli()`: 构造进程内服务器，在 `request_access` 描述中包含已安装应用名称
- `runComputerUseMcpServer()`: `--computer-use-mcp` 的子进程入口点。Stdio 传输，stdin 关闭时退出。

### 原生模块加载器

- `requireComputerUseInput()`: 延迟加载 `@ant/computer-use-input`（Rust/enigo）。内部缓存。
- `requireComputerUseSwift()`: 延迟加载 `@ant/computer-use-swift`。内部缓存。仅 macOS。

### 锁定（`computerUseLock.ts`）

#### 类型
- `AcquireResult`: `{ kind: 'acquired', fresh: boolean } | { kind: 'blocked', by: string }`
- `CheckResult`: `{ kind: 'free' } | { kind: 'held_by_self' } | { kind: 'blocked', by: }`

#### 函数
- `checkComputerUseLock()`: 检查锁定状态而不获取。执行陈旧 PID 恢复。
- `tryAcquireComputerUseLock()`: 通过 O_EXCL（open 'wx'）的原子测试和设置。返回 fresh/reentrant/blocked。
- `releaseComputerUseLock()`: 如果当前会话拥有则释放锁定。如果实际取消链接则返回 true。
- `isLockHeldLocally()`: 零系统调用检查 — 此进程是否相信自己持有锁定？

### 清理（`cleanup.ts`）

#### 函数
- `cleanupComputerUseAfterTurn(ctx)`: 回合结束清理：自动取消隐藏隐藏的应用（5秒超时）、注销 Escape 热键、释放文件锁定、发送退出通知。

### 排空 RunLoop（`drainRunLoop.ts`）

#### 函数
- `drainRunLoop(fn)`: 在共享 CFRunLoop 泵运行时等待 `fn()`。30秒超时。可嵌套。

### Escape 热键（`escHotkey.ts`）

#### 函数
- `registerEscHotkey(onEscape)`: 通过 CGEventTap 注册全局 Escape → 中止。如果 CGEventTap 创建失败（例如缺少辅助功能权限）则返回 false。
- `unregisterEscHotkey()`: 注销 Escape 热键并释放泵保留。
- `notifyExpectedEscape()`: 为模型合成的 Escape 打孔，以免触发中止。

### 应用名称（`appNames.ts`）

#### 函数
- `filterAppsForDescription(installed, homeDir)`: 将原始 Spotlight 结果过滤为面向用户的应用，然后清理。始终保留的应用（浏览器、生产力工具、开发工具）绕过路径/名称过滤。

#### 过滤
- 路径允许列表：`/Applications/`、`/System/Applications/`、`~/Applications/`
- 名称屏蔽列表模式：Helper、Agent、Service、Uninstaller、Updater、点前缀
- 字符允许列表：Unicode 字母、标记、数字、空格和有限标点
- 最大长度：40 个字符；最大数量：50（带有"… 和 N 更多"溢出）
- 始终保留的包 ID：约 30 个常见应用（Safari、Chrome、VSCode、Slack 等）

### 工具渲染（`toolRendering.tsx`）

#### 函数
- `getComputerUseMCPRenderingOverrides(toolName)`: `mcp__computer-use__*` 工具的渲染覆盖 — 自定义 `userFacingName`、`renderToolUseMessage`、`renderToolResultMessage`。

### 门控（`gates.ts`）

#### 函数
- `getChicagoEnabled()`: 检查计算机使用是否启用（订阅检查 + GrowthBook 配置）。具有 monorepo 访问权限的 Ants 被排除，除非 `ALLOW_ANT_COMPUTER_USE_MCP=1`。
- `getChicagoSubGates()`: 获取子特性门控（pixelValidation、clipboardPasteMultiline、mouseAnimation、hideBeforeAction、autoTargetDisplay、clipboardGuard）。
- `getChicagoCoordinateMode()`: 获取坐标模式（'pixels' 或 'normalized'）。在首次读取时冻结。

## 设计注意事项

- **CFRunLoop 泵送**：Swift 的 `@MainActor` 方法和 enigo 的 key() 分派到 `DispatchQueue.main`，在 libuv（Node/bun）下永不排出。`drainRunLoop` 泵在等待任何依赖主队列的调用时运行调用 `_drainMainRunLoop` 的 1ms `setInterval`。
- **终端作为替代主机**：检测终端模拟器的包 ID 并将其作为替代主机传递给 Swift — 从隐藏中免除、在激活 z-order 遍历中跳过、并从截图中排除。
- **基于文件的锁定**：使用 O_EXCL（open 'wx'）进行原子测试和设置。陈旧 PID 恢复取消链接死会话的锁定。注册为关闭清理处理程序。
- **Escape 中止**：CGEventTap 在注册时消耗系统范围的 Escape（PI 防御 — 提示注入无法用 Escape 关闭对话框）。通过 `notifyExpectedEscape()` 排除模型合成的 Escape。
- **截图调整大小**：预调整为 `targetImageSize` 输出，以便 API 转码器的早期返回触发 — 无服务器端调整大小，坐标缩放保持一致。
- **剪贴板输入**：保存 → 写入 → 读回验证 → Cmd+V → 100ms 睡眠 → 恢复。读回验证防止剪贴板写入静默失败时粘贴垃圾。
- **应用名称加固**：Unicode 感知的字符允许列表（`\p{L}\p{M}\p{N}`）、仅单空格（不是 `\s` 以防止换行注入）、长度限制、计数限制带溢出指示器。
