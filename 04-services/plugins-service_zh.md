# 插件服务

## 目的

管理插件生命周期操作（安装、卸载、启用、禁用、更新）和后台市场安装。提供 CLI 命令包装器和可由交互式 UI 使用的纯库函数。

## 位置

`restored-src/src/services/plugins/`

### 关键文件

| 文件 | 描述 |
|------|-------------|
| `pluginOperations.ts` | 核心插件操作（安装、卸载、启用、禁用、更新）— 纯库函数 |
| `PluginInstallationManager.ts` | 用于启动的后台插件和市场安装管理器 |
| `pluginCliCommands.ts` | 处理控制台输出和进程退出的 CLI 命令包装器 |

## 关键导出

### 函数 (pluginOperations.ts)
- `installPluginOp(plugin, scope)`：从市场安装插件到指定作用域（user/project/local）
- `uninstallPluginOp(plugin, scope, deleteDataDir)`：从指定作用域卸载插件
- `enablePluginOp(plugin, scope)`：在指定作用域启用插件
- `disablePluginOp(plugin, scope)`：在指定作用域禁用插件
- `disableAllPluginsOp()`：禁用所有已启用的插件
- `updatePluginOp(plugin, scope)`：将插件更新到最新版本
- `setPluginEnabledOp(plugin, enabled, scope)`：具有完整作用域解析的核心启用/禁用
- `getPluginInstallationFromV2(pluginId)`：从 V2 数据获取插件的最相关安装
- `isPluginEnabledAtProjectScope(pluginId)`：检查插件是否在项目级设置中启用
- `assertInstallableScope(scope)`：运行时作用域验证
- `isInstallableScope(scope)`：可安装作用域的类型守卫
- `getProjectPathForScope(scope)`：获取项目/本地作用域的项目路径

### 函数 (PluginInstallationManager.ts)
- `performBackgroundPluginInstallations(setAppState)`：后台启动检查和安装

### 函数 (pluginCliCommands.ts)
- `installPlugin(plugin, scope)`：带控制台输出和遥测的 CLI 安装命令
- `uninstallPlugin(plugin, scope, keepData)`：CLI 卸载命令
- `enablePlugin(plugin, scope)`：CLI 启用命令
- `disablePlugin(plugin, scope)`：CLI 禁用命令
- `disableAllPlugins()`：CLI 禁用全部命令
- `updatePluginCli(plugin, scope)`：CLI 更新命令

### 类型
- `PluginScope`：`'user' | 'project' | 'local' | 'managed'`
- `InstallableScope`：`'user' | 'project' | 'local'`（不包括 'managed'）
- `PluginOperationResult`：结果对象，包含 success、message、pluginId、scope、reverseDependents
- `PluginUpdateResult`：结果对象，包含 success、message、oldVersion、newVersion、alreadyUpToDate

### 常量
- `VALID_INSTALLABLE_SCOPES`：`['user', 'project', 'local']`
- `VALID_UPDATE_SCOPES`：`['user', 'project', 'local', 'managed']`

## 依赖

### 内部依赖
- `marketplaceManager` — 插件市场加载和查找
- `pluginLoader` — 插件缓存和版本化缓存管理
- `installedPluginsManager` — 磁盘上的 V2 安装跟踪
- `pluginPolicy` — 组织策略执行
- `pluginIdentifier` — 插件名称/市场解析
- `settings` — 跨作用域读取/写入 enabledPlugins
- `cacheUtils` — 缓存清除和版本孤立
- `dependencyResolver` — 反向依赖检测

### 外部依赖
- `path` — 文件路径操作

## 实现细节

### 核心逻辑

插件服务遵循**设置优先**架构：

1. **安装**：搜索市场 → 写入设置（声明意图）→ 缓存插件 + 记录版本
2. **卸载**：从设置中移除 → 从 installed_plugins_v2.json 移除 → 清理缓存和数据
3. **启用/禁用**：从设置中解析插件 ID 和作用域 → 写入 enabledPlugins → 清除缓存
4. **更新**：非原地 — 下载到 temp → 计算版本 → 复制到新版本化缓存 → 更新磁盘 JSON

### 作用域解析

作用域遵循优先级：`local > project > user > managed`。自动检测作用域时，最具体提及插件的作用域获胜。'managed' 作用域是特殊的 — 插件只能从 managed-settings.json 安装，不能直接安装。

### 插件标识符格式

插件标识为 `name@marketplace`。裸名称通过搜索所有可编辑作用域中匹配键来解析。

### 反向依赖

卸载/禁用时，系统警告（但不阻止）其他已启用插件依赖目标。阻止会创建带有除名插件的墓碑。

### 版本化缓存

插件存储在版本化缓存目录中。更新创建新版本化路径；当不再被引用时，旧版本被孤立。

## 数据流

```
CLI/UI → pluginCliCommands.ts → pluginOperations.ts
                                    ↓
                           marketplaceManager (find plugin)
                                    ↓
                           settings (write enabledPlugins)
                                    ↓
                           pluginLoader (cache plugin)
                                    ↓
                           installedPluginsManager (record on disk)
```

### 后台安装流程

```
启动 → PluginInstallationManager
              ↓
         diffMarketplaces (declared vs materialized)
              ↓
         reconcileMarketplaces (install missing/changed)
              ↓
         refreshActivePlugins (if new installs)
         OR set needsRefresh (if updates only)
```

## 集成点

- **CLI**：`pluginCliCommands.ts` 用控制台输出、遥测和 process.exit 包装操作
- **UI**：`pluginOperations.ts` 函数直接由 ManagePlugins.tsx 使用
- **启动**：`PluginInstallationManager.ts` 运行后台市场协调
- **策略**：`isPluginBlockedByPolicy` 控制启用/安装操作
- **设置**：每个作用域级别的 settings.json 中的 enabledPlugins

## 配置

- 作用域：`user` (~/.claude/)、`project` (.claude/)、`local` (.claude/ 本地覆盖)、`managed`（远程托管）
- 插件数据目录在卸载时清理（可通过 `deleteDataDir` 标志配置）

## 错误处理

- 操作返回 `PluginOperationResult` 对象 — 核心操作中没有 process.exit() 或控制台写入
- CLI 包装器捕获错误，记录遥测（`tengu_plugin_command_failed`），并以代码 1 退出
- 策略阻止的插件返回描述性错误消息
- 除名插件通过回退到 installed_plugins.json 处理

## 测试

- 不存在 `_reset*ForTesting` 助手 — 测试直接使用纯操作函数
- 模块级状态最小；大多数状态存在于磁盘文件中

## 相关模块

- [远程托管设置](./remote-managed-settings.md) — 托管作用域插件安装
- [策略限制](./policy-limits-service.md) — 组织策略执行
- [分析](./analytics-service.md) — 插件遥测事件

## 备注

- 核心操作故意避免副作用（无控制台输出，无 process.exit）以便 UI 重用
- V2 installed_plugins.json 格式独立于市场状态跟踪安装
- 启动时市场协调处理已声明但未实现的市場
- 最后作用域卸载时清理插件选项和密码
