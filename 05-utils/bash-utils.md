# Bash Utilities

**Directory:** `restored-src/src/utils/bash/`

## Purpose

Provides bash command parsing, analysis, and security validation. Includes both a pure-TypeScript tree-sitter-compatible parser and a shell-quote-based legacy parser, plus command prefix extraction and shell completion.

## parser.ts

### AST-Based Security Parser

Main entry point for security-critical command parsing:

- `parseForSecurity(cmd)` - Parses command and extracts simple commands
  - Returns `ParseForSecurityResult`:
    - `{ kind: 'simple', commands: SimpleCommand[] }` - Safe, extractable
    - `{ kind: 'too-complex', reason, nodeType? }` - Cannot statically analyze
    - `{ kind: 'parse-unavailable' }` - Tree-sitter not loaded

- `parseForSecurityFromAst(cmd, root)` - Same as above with pre-parsed AST

### Pre-Checks (Tree-Sitter/Bash Differentials)

Rejects commands containing:
- Control characters (0x00-0x1F, 0x7F)
- Unicode whitespace (NBSP, zero-width spaces, etc.)
- Backslash-escaped whitespace (line continuation differentials)
- Zsh tilde bracket syntax (~[dynamic directory])
- Zsh equals expansion (=cmd)
- Brace expansion with quote obfuscation

### Command Extraction

- `SimpleCommand` type: `{ argv, envVars, redirects, text }`
- `Redirect` type: `{ op, target, fd? }`
- Recurses through structural nodes (program, list, pipeline, redirected_statement)
- Handles: for/while/if statements, subshells, test commands, declaration commands, unset commands
- Variable scope tracking: tracks assigned variables for $VAR resolution
- Command substitution: recursively extracts $(...) inner commands

### Safety Mechanisms

- Explicit allowlist of node types (fail-closed)
- DANGEROUS_TYPES set: command_substitution, process_substitution, expansion, etc.
- Variable scope isolation for || / | / & / subshells
- Brace expansion detection
- Backslash-whitespace detection
- Placeholder system for command substitution output

## bashParser.ts

### Pure-TypeScript Parser

Tree-sitter-compatible bash parser written in TypeScript:

- `parseSource(source, timeoutMs?)` - Parses bash source to AST
  - 50ms timeout (configurable via Infinity for tests)
  - 50,000 node budget cap
  - Returns TsNode or null on abort

- `ensureParserInitialized()` - No-op (pure-TS, no async init)
- `getParserModule()` - Returns parser module

### Lexer

Tokenizes bash input with UTF-8 byte offset tracking:
- Token types: WORD, NUMBER, OP, NEWLINE, COMMENT, DQUOTE, SQUOTE, ANSI_C, DOLLAR variants, BACKTICK, LT_PAREN, GT_PAREN, EOF
- Context-sensitive: 'cmd' vs 'arg' mode for [ [[ { handling
- Heredoc tracking: pending delimiters scanned at next newline
- UTF-8 support: byte table for non-ASCII inputs

### Parser

Recursive descent parser matching tree-sitter-bash grammar:
- Program, statements, and/or chains, pipelines
- Commands: simple, compound, control structures
- Variable assignments with subscripts
- Heredocs with quoted/unquoted delimiters
- Test commands ([ ] and [[ ]])
- Arithmetic expressions
- Function definitions

## ast.ts

### AST Walking Utilities

- `parseCommand(command)` - Parses command and extracts ParsedCommandData
  - Returns `{ rootNode, envVars, commandNode, originalCommand }`
  - Finds command node within AST
  - Extracts leading environment variables

- `parseCommandRaw(command)` - Raw parse without env extraction
  - Returns Node, null, or PARSE_ABORTED symbol
  - PARSE_ABORTED indicates module loaded but parse failed (fail-closed)

- `extractCommandArguments(commandNode)` - Extracts argv from command node
  - Handles declaration commands specially
  - Strips quotes from arguments

## commands.ts

### Command Splitting and Analysis

- `splitCommandWithOperators(command)` - Splits command on shell operators
  - Uses shell-quote with placeholder injection for quotes/newlines
  - Handles heredoc extraction before parsing
  - Joins line continuations (odd backslash count)
  - Restores heredocs after parsing

- `splitCommand_DEPRECATED(command)` - Legacy regex-based splitter
  - Strips redirections from command parts
  - Filters control operators

- `extractOutputRedirections(cmd)` - Extracts file redirections
  - Returns `{ commandWithoutRedirections, redirections, hasDangerousRedirection }`
  - Handles FD redirects (2>&1), force clobber (>!, >|), combined (>&, &>)
  - Detects dangerous expansion in targets ($VAR, backticks, globs)

- `isHelpCommand(command)` - Detects simple --help commands
  - Bypasses prefix extraction for safe help queries

- `isCommandList(command)` - Checks if command is a safe command list
  - Allows strings, globs, separators, FD redirects, output redirects

- `isUnsafeCompoundCommand_DEPRECATED(command)` - Legacy compound check

- `getCommandPrefix` / `getCommandSubcommandPrefix` - AI-based prefix extraction
  - Uses LLM with policy spec to extract command prefix
  - Help commands bypass extraction (return full command)

- `clearCommandPrefixCaches()` - Clears memoized prefix caches

## prefix.ts

### Static Prefix Extraction

- `getCommandPrefixStatic(command)` - Extracts prefix without AI
  - Uses tree-sitter parser
  - Handles wrapper commands (nice, sudo, timeout)
  - Recursively processes nested commands
  - Includes environment variable prefix

- `getCompoundCommandPrefixesStatic(command, excludeSubcommand?)` - Multi-command prefixes
  - Splits on && / || / ;
  - Collapses via word-aligned longest common prefix

## registry.ts

### Command Spec Registry

- `getCommandSpec(command)` - Returns command specification (memoized)
  - Checks built-in specs first
  - Falls back to @withfig/autocomplete specs
  - Security: rejects paths, .., and flag-like commands

- `loadFigSpec(command)` - Dynamically loads fig autocomplete spec

### Types

- `CommandSpec` - Command definition with subcommands, args, options
- `Argument` - Argument metadata (dangerous, variadic, optional, etc.)
- `Option` - Option definition with name, description, args

## shellCompletion.ts

### Shell Tab Completion

- `getShellCompletions(input, cursorOffset, abortSignal)` - Returns completion suggestions
  - Supports bash and zsh
  - Detects completion type: command, variable, file
  - Uses compgen (bash) or native zsh commands
  - Max 15 completions, 1000ms timeout

### Completion Types

- Command: `compgen -c` / zsh ${(k)commands}
- Variable: `compgen -v` / zsh ${(k)parameters}
- File: `compgen -f` / zsh glob with directory/file indicators

## shellQuote.ts

### Shell Quoting Utilities

- `quote(args)` - Quotes array of arguments for safe shell execution
- `tryParseShellCommand(command)` - Parses command using shell-quote library
  - Returns `{ success, tokens }` or `{ success: false, error }`

## heredoc.ts

### Heredoc Handling

- `extractHeredocs(command)` - Extracts heredocs before parsing
- `restoreHeredocs(parts, heredocs)` - Restores heredocs after parsing

## shellPrefix.ts

### Prefix Extraction Helpers

- `createCommandPrefixExtractor(config)` - Creates AI-based prefix extractor
- `createSubcommandPrefixExtractor(baseExtractor, splitFn)` - Wraps for compound commands

## ShellSnapshot.ts

### Shell Environment Snapshot

- `createAndSaveSnapshot(shellPath)` - Captures current shell environment
  - Saves to temp file for sourcing in spawned commands
  - Enables environment consistency across commands

## specs/

### Command Specifications

Built-in command specs for prefix extraction:
- `alias.ts` - alias command spec
- `timeout.ts` - timeout wrapper spec
- `time.ts` - time command spec
- `sleep.ts` - sleep command spec
- `srun.ts` - srun (Slurm) spec
- `pyright.ts` - pyright spec
- `nohup.ts` - nohup spec
- `index.ts` - Exports all specs

## ParsedCommand.ts

### Parsed Command Data Type

- `ParsedCommandData` - Result of command parsing
- `Node` - Alias for TsNode
- Constants: MAX_COMMAND_LENGTH (10000), DECLARATION_COMMANDS

## Dependencies

- `shell-quote` - Shell parsing library
- `@withfig/autocomplete` - Command completion specs
- `debug.js` - Debug logging
- `localInstaller.js` - Shell type detection
- `Shell.js` - Shell execution
- `memoize.js` - LRU memoization
