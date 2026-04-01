# Bash 工具

**目录：** `restored-src/src/utils/bash/`

## 目的

提供 bash 命令解析、分析和安全验证。包括纯 TypeScript 的 tree-sitter 兼容解析器和基于 shell-quote 的遗留解析器，以及命令前缀提取和 shell 补全。

## parser.ts

### 基于 AST 的安全解析器

安全关键命令解析的主要入口点：

- `parseForSecurity(cmd)` - 解析命令并提取简单命令
  - 返回 `ParseForSecurityResult`:
    - `{ kind: 'simple', commands: SimpleCommand[] }` - 安全的、可提取的
    - `{ kind: 'too-complex', reason, nodeType? }` - 无法静态分析
    - `{ kind: 'parse-unavailable' }` - Tree-sitter 未加载

- `parseForSecurityFromAst(cmd, root)` - 与上述相同，带预解析的 AST

### 预检查（Tree-Sitter/Bash 差异）

拒绝包含以下内容的命令：
- 控制字符（0x00-0x1F、0x7F）
- Unicode 空白（NBSP、零宽空格等）
- 反斜杠转义空白（行继续差异）
- Zsh 波浪号括号语法（~[动态目录]）
- Zsh 等号扩展（=cmd）
- 带引号混淆的大括号扩展

### 命令提取

- `SimpleCommand` 类型: `{ argv, envVars, redirects, text }`
- `Redirect` 类型: `{ op, target, fd? }`
- 通过结构节点递归（program、list、pipeline、redirected_statement）
- 处理：for/while/if 语句、子 shell、test 命令、声明命令、unset 命令
- 变量作用域跟踪：跟踪已分配变量用于 $VAR 解析
- 命令替换：递归提取 $(...) 内部命令

### 安全机制

- 显式允许的节点类型列表（fail-closed）
- DANGEROUS_TYPES 集合：command_substitution、process_substitution、expansion 等
- || / | / & / 子 shell 的变量作用域隔离
- 大括号扩展检测
- 反斜杠空白检测
- 命令替换输出的占位符系统

## bashParser.ts

### 纯 TypeScript 解析器

用 TypeScript 编写的 tree-sitter 兼容 bash 解析器：

- `parseSource(source, timeoutMs?)` - 将 bash 源代码解析为 AST
  - 50ms 超时（可通过 Infinity 配置用于测试）
  - 50,000 节点预算上限
  - 中止时返回 TsNode 或 null

- `ensureParserInitialized()` - 空操作（纯 TS，无异步初始化）
- `getParserModule()` - 返回解析器模块

### 词法分析器

带 UTF-8 字节偏移跟踪的 bash 输入标记化：
- 标记类型：WORD、NUMBER、OP、NEWLINE、COMMENT、DQUOTE、SQUOTE、ANSI_C、DOLLAR 变体、BACKTICK、LT_PAREN、GT_PAREN、EOF
- 上下文敏感：'cmd' vs 'arg' 模式用于 [ [[ { 处理
- Heredoc 跟踪：在下一个换行符扫描待定定界符
- UTF-8 支持：用于非 ASCII 输入的字节表

### 解析器

匹配 tree-sitter-bash 语法的递归下降解析器：
- Program、statements、and/or chains、pipelines
- 命令：简单、复合、控制结构
- 带下标的变量赋值
- 带引号/非引号定界符的 heredoc
- Test 命令（[ ] 和 [[ ]]）
- 算术表达式
- 函数定义

## ast.ts

### AST 遍历工具

- `parseCommand(command)` - 解析命令并提取 ParsedCommandData
  - 返回 `{ rootNode, envVars, commandNode, originalCommand }`
  - 在 AST 中查找命令节点
  - 提取领先的环境变量

- `parseCommandRaw(command)` - 无环境提取的原始解析
  - 返回 Node、null 或 PARSE_ABORTED 符号
  - PARSE_ABORTED 表示模块已加载但解析失败（fail-closed）

- `extractCommandArguments(commandNode)` - 从命令节点提取 argv
  - 特殊处理声明命令
  - 从参数中剥离引号

## commands.ts

### 命令拆分和分析

- `splitCommandWithOperators(command)` - 在 shell 运算符上拆分命令
  - 使用占位符注入进行 shell-quote 处理引号/换行符
  - 在解析前提取 heredoc
  - 连接行继续（奇数反斜杠计数）
  - 解析后恢复 heredoc

- `splitCommand_DEPRECATED(command)` - 遗留的正则表达式拆分器
  - 从命令部分剥离重定向
  - 过滤控制运算符

- `extractOutputRedirections(cmd)` - 提取文件重定向
  - 返回 `{ commandWithoutRedirections, redirections, hasDangerousRedirection }`
  - 处理 FD 重定向（2>&1）、强制覆盖（>!、>|）、组合（>&、&>）
  - 检测目标中的危险扩展（$VAR、反引号、glob）

- `isHelpCommand(command)` - 检测简单的 --help 命令
  - 为安全帮助查询绕过前缀提取

- `isCommandList(command)` - 检查命令是否为安全命令列表
  - 允许字符串、glob、分隔符、FD 重定向、输出重定向

- `isUnsafeCompoundCommand_DEPRECATED(command)` - 遗留复合检查

- `getCommandPrefix` / `getCommandSubcommandPrefix` - 基于 AI 的前缀提取
  - 使用 LLM 和策略规范提取命令前缀
  - 帮助命令绕过提取（返回完整命令）

- `clearCommandPrefixCaches()` - 清除记忆化的前缀缓存

## prefix.ts

### 静态前缀提取

- `getCommandPrefixStatic(command)` - 无 AI 提取前缀
  - 使用 tree-sitter 解析器
  - 处理包装命令（nice、sudo、timeout）
  - 递归处理嵌套命令
  - 包含环境变量前缀

- `getCompoundCommandPrefixesStatic(command, excludeSubcommand?)` - 多命令前缀
  - 在 && / || / ; 上拆分
  - 通过词对齐最长公共前缀折叠

## registry.ts

### 命令规范注册表

- `getCommandSpec(command)` - 返回命令规范（记忆化）
  - 首先检查内置规范
  - 回退到 @withfig/autocomplete 规范
  - 安全：拒绝路径、.. 和类似标志的命令

- `loadFigSpec(command)` - 动态加载 fig 自动补全规范

### 类型

- `CommandSpec` - 带子命令、参数、选项的命令定义
- `Argument` - 参数元数据（危险、可变参数、可选等）
- `Option` - 带名称、描述、参数的选项定义

## shellCompletion.ts

### Shell Tab 补全

- `getShellCompletions(input, cursorOffset, abortSignal)` - 返回补全建议
  - 支持 bash 和 zsh
  - 检测补全类型：命令、变量、文件
  - 使用 compgen（bash）或原生 zsh 命令
  - 最多 15 个补全，1000ms 超时

### 补全类型

- 命令：`compgen -c` / zsh ${(k)commands}
- 变量：`compgen -v` / zsh ${(k)parameters}
- 文件：`compgen -f` / zsh 带有目录/文件指示符的 glob

## shellQuote.ts

### Shell 引号工具

- `quote(args)` - 为安全 shell 执行引用参数数组
- `tryParseShellCommand(command)` - 使用 shell-quote 库解析命令
  - 返回 `{ success, tokens }` 或 `{ success: false, error }`

## heredoc.ts

### Heredoc 处理

- `extractHeredocs(command)` - 在解析前提取 heredoc
- `restoreHeredocs(parts, heredocs)` - 在解析后恢复 heredoc

## shellPrefix.ts

### 前缀提取辅助函数

- `createCommandPrefixExtractor(config)` - 创建基于 AI 的前缀提取器
- `createSubcommandPrefixExtractor(baseExtractor, splitFn)` - 包装用于复合命令

## ShellSnapshot.ts

### Shell 环境快照

- `createAndSaveSnapshot(shellPath)` - 捕获当前 shell 环境
  - 保存到临时文件以在生成命令中获取
  - 实现跨命令的环境一致性

## specs/

### 命令规范

用于前缀提取的内置命令规范：
- `alias.ts` - alias 命令规范
- `timeout.ts` - timeout 包装器规范
- `time.ts` - time 命令规范
- `sleep.ts` - sleep 命令规范
- `srun.ts` - srun (Slurm) 规范
- `pyright.ts` - pyright 规范
- `nohup.ts` - nohup 规范
- `index.ts` - 导出所有规范

## ParsedCommand.ts

### 解析命令数据类型

- `ParsedCommandData` - 命令解析的结果
- `Node` - TsNode 的别名
- 常量：MAX_COMMAND_LENGTH (10000)、DECLARATION_COMMANDS

## 依赖

- `shell-quote` - Shell 解析库
- `@withfig/autocomplete` - 命令补全规范
- `debug.js` - 调试日志
- `localInstaller.js` - Shell 类型检测
- `Shell.js` - Shell 执行
- `memoize.js` - LRU 记忆化
