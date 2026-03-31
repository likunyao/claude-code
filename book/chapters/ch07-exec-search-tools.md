# 第7章：执行与搜索工具 —— 连接外部世界的桥梁

> "工具是手的延伸，而好的工具设计则是意图的延伸。"

Claude Code 的强大之处在于它不仅能理解代码，还能直接操作计算环境。这种能力通过两类关键工具实现：**执行工具**（BashTool）和**搜索工具**（WebSearchTool、WebFetchTool、ToolSearchTool）。本章将深入分析这些工具如何成为 AI 与外部世界交互的桥梁，同时补充介绍专门用于非交互场景的 **StructuredOutputTool**。

## 7.1 BashTool：沙箱化的命令执行

BashTool 是 Claude Code 中最强大也最危险的工具。它允许 AI 直接执行 shell 命令，拥有与人类开发者相同的系统能力——同时需要严格的安全约束。整个 BashTool 工具由近 20 个源文件组成，安全相关代码超过 10 万行，是 Claude Code 中安全投入最大的工具。

### 7.1.1 输入模型

BashTool 的完整输入 Schema（`fullInputSchema`）定义了它与 shell 交互的全部接口：

```typescript
// 文件: src/tools/BashTool/BashTool.tsx
const fullInputSchema = lazySchema(() =>
  z.strictObject({
    command: z.string().describe('The command to execute'),
    timeout: semanticNumber(z.number().optional())
      .describe(`Optional timeout in milliseconds (max ${getMaxTimeoutMs()})`),
    description: z.string().optional()
      .describe('Clear, concise description of what this command does...'),
    run_in_background: semanticBoolean(z.boolean().optional())
      .describe('Set to true to run this command in the background...'),
    dangerouslyDisableSandbox: semanticBoolean(z.boolean().optional())
      .describe('Set this to true to dangerously override sandbox mode...'),
    _simulatedSedEdit: z.object({
      filePath: z.string(),
      newContent: z.string()
    }).optional()
      .describe('Internal: pre-computed sed edit result from preview')
  })
)
```

**设计要点**：

- `_simulatedSedEdit` 是一个内部字段，永远不暴露给模型。它由权限对话框在用户批准 sed 编辑预览后注入，确保用户预览的内容与实际写入的内容完全一致。
- `run_in_background` 在后台任务被禁用时会从模型可见的 Schema 中移除（通过 `omit()`），这是一种优雅的功能降级模式。
- `semanticBoolean` 和 `semanticNumber` 是 Claude Code 自定义的 Zod 包装器，它们优化了参数在 LLM 上下文中的表达方式，确保模型能更准确地理解和生成这些参数。

暴露给模型的 Schema（`inputSchema`）会剔除 `_simulatedSedEdit`（防止模型绕过权限检查）：

```typescript
const inputSchema = lazySchema(() =>
  isBackgroundTasksDisabled
    ? fullInputSchema().omit({
        run_in_background: true,
        _simulatedSedEdit: true
      })
    : fullInputSchema().omit({ _simulatedSedEdit: true })
)
```

输出 Schema 包含 14 个字段，覆盖了从标准输出到后台任务 ID 的完整执行上下文：

```typescript
const outputSchema = lazySchema(() =>
  z.object({
    stdout: z.string(),
    stderr: z.string(),
    rawOutputPath: z.string().optional(),
    interrupted: z.boolean(),
    isImage: z.boolean().optional(),
    backgroundTaskId: z.string().optional(),
    backgroundedByUser: z.boolean().optional(),
    assistantAutoBackgrounded: z.boolean().optional(),
    dangerouslyDisableSandbox: z.boolean().optional(),
    returnCodeInterpretation: z.string().optional(),
    noOutputExpected: z.boolean().optional(),
    structuredContent: z.array(z.any()).optional(),
    persistedOutputPath: z.string().optional(),
    persistedOutputSize: z.number().optional(),
  })
)
```

### 7.1.2 多层安全防护体系

BashTool 的安全设计采用**纵深防御**（Defense in Depth）策略，分布在多个文件中，每一层都有明确的安全职责。

**第一层：语法安全验证（bashSecurity.ts，约 2500 行）**

`bashCommandIsSafe_DEPRECATED` 函数是安全验证的核心入口。它执行一组有序的验证器链（validators pipeline），每个验证器返回 `ask`（需要用户确认）、`passthrough`（放行）或 `allow`（允许）：

```typescript
const validators = [
  validateJqCommand,              // jq 系统命令注入检测
  validateObfuscatedFlags,        // 混淆标志检测
  validateShellMetacharacters,     // Shell 元字符检测
  validateDangerousVariables,      // 危险变量检测
  validateCommentQuoteDesync,      // 注释-引号反同步检测
  validateQuotedNewline,           // 引号内换行检测
  validateCarriageReturn,          // 回车符检测
  validateNewlines,                // 换行符检测
  validateIFSInjection,            // IFS 注入检测
  validateProcEnvironAccess,       // /proc/environ 访问检测
  validateDangerousPatterns,       // 危险模式检测
  validateRedirections,            // 重定向检测
  validateBackslashEscapedWhitespace,  // 反斜杠转义空白检测
  validateBackslashEscapedOperators,   // 反斜杠转义操作符检测
  validateUnicodeWhitespace,       // Unicode 空白字符检测
  validateMidWordHash,             // 词中 # 字符检测
  validateBraceExpansion,          // 大括号展开检测
  validateZshDangerousCommands,    // Zsh 危险命令检测
  validateMalformedTokenInjection, // 畸形标记注入检测
]
```

验证器链的关键设计是**不短路**：即使某个非误解析验证器返回 `ask`，后续的误解析验证器仍然会继续运行。这是因为非误解析验证器的 `ask` 结果可能被下游的权限系统丢弃，如果短路了，攻击者可以利用这一点绕过安全检查。

**第二层：Tree-sitter AST 解析（ast.ts）**

当 tree-sitter 可用时，`parseForSecurity` 函数提供比正则表达式更精确的命令解析。它将命令解析为 AST 节点，然后检查每个子命令的语义：

```typescript
// src/utils/bash/ast.ts
const parsed = await parseForSecurity(command)
if (parsed.kind !== 'simple') {
  // 解析失败或命令过于复杂 — 安全起见，触发 hook
  return () => true
}
// 在 argv 级别匹配（去掉 VAR=val 前缀）
const subcommands = parsed.commands.map(c => c.argv.join(' '))
return pattern => {
  const prefix = permissionRuleExtractPrefix(pattern)
  return subcommands.some(cmd => {
    if (prefix !== null) {
      return cmd === prefix || cmd.startsWith(`${prefix} `)
    }
    return matchWildcardPattern(pattern, cmd)
  })
}
```

**第三层：命令替换和危险模式检测（bashSecurity.ts）**

`COMMAND_SUBSTITUTION_PATTERNS` 定义了 12 种需要拦截的 shell 替换模式：

```typescript
const COMMAND_SUBSTITUTION_PATTERNS = [
  { pattern: /<\(/, message: 'process substitution <()' },
  { pattern: />\(/, message: 'process substitution >()' },
  { pattern: /=\(/, message: 'Zsh process substitution =()' },
  { pattern: /(?:^|[\s;&|])=[a-zA-Z_]/, message: 'Zsh equals expansion (=cmd)' },
  { pattern: /\$\(/, message: '$() command substitution' },
  { pattern: /\$\{/, message: '${} parameter substitution' },
  { pattern: /\$\[/, message: '$[] legacy arithmetic expansion' },
  { pattern: /~\[/, message: 'Zsh-style parameter expansion' },
  { pattern: /\(e:/, message: 'Zsh-style glob qualifiers' },
  { pattern: /\(\+/, message: 'Zsh glob qualifier with command execution' },
  { pattern: /\}\s*always\s*\{/, message: 'Zsh always block (try/always construct)' },
  { pattern: /<#/, message: 'PowerShell comment syntax' },
]
```

**第四层：Zsh 特殊攻击防护**

`ZSH_DANGEROUS_COMMANDS` 列出了 15 种 Zsh 专有的危险内置命令，它们可以通过 `zmodload` 加载模块来绕过常规安全检查：

```typescript
const ZSH_DANGEROUS_COMMANDS = new Set([
  'zmodload',    // 加载危险模块的网关
  'emulate',     // 等效于 eval 的命令执行
  'sysopen',     // 细粒度文件控制 (zsh/system)
  'sysread',     // 读取文件描述符
  'syswrite',    // 写入文件描述符
  'sysseek',     // 定位文件描述符
  'zpty',        // 在伪终端上执行命令
  'ztcp',        // 创建 TCP 连接（数据外泄）
  'zsocket',     // 创建 Unix/TCP 套接字
  'zf_rm',       // 内置 rm（绕过二进制检查）
  'zf_mv',       // 内置 mv
  // ... 更多内置命令
])
```

**第五层：只读约束验证（readOnlyValidation.ts，约 2000 行）**

`checkReadOnlyConstraints` 函数使用命令白名单（`COMMAND_ALLOWLIST`）来判定命令是否为纯读取操作。白名单为每个命令定义了安全的标志参数：

```typescript
const COMMAND_ALLOWLIST: Record<string, CommandConfig> = {
  xargs: {
    safeFlags: {
      '-I': '{}',
      '-n': 'number',
      '-P': 'number',
      // 注意：-i 和 -e 被故意排除！
      // GNU getopt 的可选参数语义会导致验证器和实际行为不一致
    },
  },
  file: {
    safeFlags: {
      '--brief': 'none', '-b': 'none',
      '--mime': 'none', '-i': 'none',
      // ... 只允许读取相关标志
    },
  },
  // 还包括 git read-only 命令、ripgrep、pyright 等
}
```

**第六层：沙盒隔离（shouldUseSandbox.ts）**

`shouldUseSandbox` 函数决定命令是否在沙盒中执行：

```typescript
export function shouldUseSandbox(input: Partial<SandboxInput>): boolean {
  if (!SandboxManager.isSandboxingEnabled()) {
    return false
  }
  // 显式覆盖且策略允许非沙盒命令
  if (input.dangerouslyDisableSandbox &&
      SandboxManager.areUnsandboxedCommandsAllowed()) {
    return false
  }
  if (!input.command) {
    return false
  }
  // 用户配置的排除命令
  if (containsExcludedCommand(input.command)) {
    return false
  }
  return true
}
```

排除命令支持三种模式：精确匹配（`npm run lint`）、前缀匹配（`npm run test:*`）和通配符匹配。系统会将复合命令拆分为子命令逐一检查。

### 7.1.3 核心执行流程

BashTool 的 `call` 方法是一个精心设计的异步生成器管道：

```typescript
async call(input, toolUseContext, _canUseTool, parentMessage, onProgress) {
  // 1. 处理模拟 sed 编辑 — 直接应用而非执行 sed
  if (input._simulatedSedEdit) {
    return applySedEdit(input._simulatedSedEdit, toolUseContext, parentMessage)
  }

  // 2. 启动异步命令生成器
  const commandGenerator = runShellCommand({ input, abortController, ... })

  // 3. 消费生成器，转发进度更新
  do {
    generatorResult = await commandGenerator.next()
    if (!generatorResult.done && onProgress) {
      onProgress({ toolUseID: `bash-progress-${counter++}`, data: { ... } })
    }
  } while (!generatorResult.done)

  // 4. 后处理：Git 操作追踪、Claude Code Hints 提取、图像检测
  result = generatorResult.value
  trackGitOperations(input.command, result.code, result.stdout)
  const extracted = extractClaudeCodeHints(strippedStdout, input.command)
  let isImage = isImageOutput(strippedStdout)

  // 5. 大输出持久化：超过阈值时写入磁盘
  const MAX_PERSISTED_SIZE = 64 * 1024 * 1024  // 64 MB
  if (result.outputFilePath && result.outputTaskId) {
    // 复制到工具结果目录，超过 64MB 则截断
  }

  return { data }
}
```

`runShellCommand` 是一个独立的异步生成器函数，它处理命令执行的完整生命周期：

```typescript
async function* runShellCommand({ input, abortController, ... }) {
  // 判断是否使用沙盒和自动后台化
  const shellCommand = await exec(command, signal, 'bash', {
    timeout: timeoutMs,
    shouldUseSandbox: shouldUseSandbox(input),
    shouldAutoBackground,
  })

  // 显式后台运行
  if (run_in_background === true && !isBackgroundTasksDisabled) {
    const shellId = await spawnBackgroundTask()
    return { stdout: '', stderr: '', code: 0, backgroundTaskId: shellId }
  }

  // 等待 2 秒阈值（避免闪烁）
  const initialResult = await Promise.race([
    resultPromise,
    new Promise(resolve => setTimeout(resolve, 2000))
  ])
  if (initialResult !== null) return initialResult

  // 进度循环：由共享轮询器驱动
  while (true) {
    const result = await Promise.race([resultPromise, progressSignal])
    if (result !== null) {
      // 命令完成 — 清理并返回
      shellCommand.cleanup()
      return result
    }
    // 产生进度更新
    yield { type: 'progress', fullOutput, output, elapsedTimeSeconds, ... }
  }
}
```

### 7.1.4 后台任务管理与自动后台化

BashTool 有三种将命令置于后台的机制：

1. **显式后台化**：模型设置 `run_in_background: true`
2. **超时自动后台化**：命令超过 `timeout` 时通过 `shellCommand.onTimeout` 回调触发
3. **Assistant 模式自动后台化**：主线程阻塞超过 15 秒（`ASSISTANT_BLOCKING_BUDGET_MS`）时自动后台化

```typescript
// 在 Assistant 模式下，保持主代理响应
if (getKairosActive() && isMainThread && !isBackgroundTasksDisabled) {
  setTimeout(() => {
    if (shellCommand.status === 'running' && backgroundShellId === undefined) {
      assistantAutoBackgrounded = true
      startBackgrounding('tengu_bash_command_assistant_auto_backgrounded')
    }
  }, ASSISTANT_BLOCKING_BUDGET_MS).unref()
}
```

`sleep` 命令被特别处理——它会被 `detectBlockedSleepPattern` 检测并阻止，引导模型使用 `Monitor` 工具或 `run_in_background` 代替。

### 7.1.5 命令语义分类

BashTool 对命令进行语义分类，用于 UI 折叠显示和并发安全性判断：

```typescript
const BASH_SEARCH_COMMANDS = new Set([
  'find', 'grep', 'rg', 'ag', 'ack', 'locate', 'which', 'whereis'
])
const BASH_READ_COMMANDS = new Set([
  'cat', 'head', 'tail', 'less', 'more',
  'wc', 'stat', 'file', 'strings',
  'jq', 'awk', 'cut', 'sort', 'uniq', 'tr'
])
const BASH_LIST_COMMANDS = new Set(['ls', 'tree', 'du'])
const BASH_SILENT_COMMANDS = new Set([
  'mv', 'cp', 'rm', 'mkdir', 'rmdir', 'chmod', 'touch', 'ln', 'cd'
])
```

`isSearchOrReadBashCommand` 函数对管道中的每个命令段进行判断——所有段都必须是搜索/读取命令，整个管道才被认为是可折叠的。语义中立命令（`echo`、`printf`、`true`、`false`）在任意位置都会被跳过。

### 7.1.6 安全的 Heredoc 处理

BashTool 对 heredoc 替换模式有专门的安全处理。`isSafeHeredoc` 函数只允许一种特定的安全模式：

```typescript
// 唯一被允许的模式：$(cat <<'DELIM'...DELIM)
// 其中分隔符必须被单引号引用或转义（使 body 成为字面文本）
```

函数使用**行级匹配**（而非正则表达式）来精确复制 bash 的 heredoc 关闭行为，避免正则的贪婪匹配跳过早期分隔符的风险。

### 7.1.7 设计哲学总结

BashTool 的设计体现了 Claude Code 工具设计的核心哲学：**能力最大化，风险最小化**。

- 20+ 个安全验证器形成纵深防御链
- Tree-sitter AST 解析提供比正则更精确的命令分析
- 异步生成器模式实现流式输出和可中断性
- 三级后台化机制支持各种长时间运行场景
- 命令语义分类驱动 UI 和并发安全决策

## 7.2 WebSearchTool：实时信息检索

WebSearchTool 赋予 Claude Code 获取实时互联网信息的能力——这是突破训练数据时间限制的关键机制。

### 7.2.1 输入输出 Schema

```typescript
// 文件: src/tools/WebSearchTool/WebSearchTool.ts
const inputSchema = lazySchema(() =>
  z.strictObject({
    query: z.string().min(2).describe('The search query to use'),
    allowed_domains: z.array(z.string()).optional()
      .describe('Only include search results from these domains'),
    blocked_domains: z.array(z.string()).optional()
      .describe('Never include search results from these domains'),
  })
)
```

输出 Schema 包含搜索结果和模型对结果的分析文本：

```typescript
const outputSchema = lazySchema(() =>
  z.object({
    query: z.string(),
    results: z.array(z.union([searchResultSchema(), z.string()])),
    durationSeconds: z.number(),
  })
)
```

`results` 是一个联合类型：`SearchResult`（包含 `tool_use_id` 和搜索命中列表）和 `string`（模型的文本分析）交替出现。

### 7.2.2 双模型搜索架构

WebSearchTool 采用**双模型架构**：它不是直接调用搜索 API，而是通过一个辅助模型（通常是 Haiku）来执行搜索，利用 Anthropic 的 `web_search_20250305` 服务端工具：

```typescript
async call(input, context, _canUseTool, _parentMessage, onProgress) {
  const toolSchema = makeToolSchema(input)

  const useHaiku = getFeatureValue_CACHED_MAY_BE_STALE('tengu_plum_vx3', false)
  const queryStream = queryModelWithStreaming({
    messages: [userMessage],
    systemPrompt: asSystemPrompt([
      'You are an assistant for performing a web search tool use'
    ]),
    tools: [],
    options: {
      model: useHaiku ? getSmallFastModel() : context.options.mainLoopModel,
      toolChoice: useHaiku ? { type: 'tool', name: 'web_search' } : undefined,
      extraToolSchemas: [toolSchema],
    },
  })
  // ... 流式处理事件 ...
}
```

`makeToolSchema` 构建搜索工具的配置：

```typescript
function makeToolSchema(input: Input): BetaWebSearchTool20250305 {
  return {
    type: 'web_search_20250305',
    name: 'web_search',
    allowed_domains: input.allowed_domains,
    blocked_domains: input.blocked_domains,
    max_uses: 8,  // 硬编码最多 8 次搜索
  }
}
```

### 7.2.3 流式进度追踪

搜索过程通过流式事件向用户展示实时进度：

```typescript
for await (const event of queryStream) {
  // 追踪 server_tool_use 开始，记录 tool_use_id
  if (event.type === 'stream_event' && event.event?.type === 'content_block_start') {
    // 提取搜索查询并报告进度
    onProgress({
      toolUseID: `search-progress-${progressCounter}`,
      data: { type: 'query_update', query }
    })
  }

  // 搜索结果到达时报告
  if (contentBlock?.type === 'web_search_tool_result') {
    onProgress({
      data: { type: 'search_results_received', resultCount, query }
    })
  }
}
```

### 7.2.4 结果后处理

`makeOutputFromSearchResponse` 将原始的 `BetaContentBlock` 流转换为结构化输出。它追踪四种块类型（`server_tool_use`、`web_search_tool_result`、`text` 和搜索命中），将文本和搜索结果交替排列。

`mapToolResultToToolResultBlockParam` 在结果末尾附加强制提醒：

```typescript
formattedOutput +=
  '\nREMINDER: You MUST include the sources above in your response ' +
  'to the user using markdown hyperlinks.'
```

### 7.2.5 可用性判断

WebSearchTool 的 `isEnabled` 根据模型提供商判断可用性：

```typescript
isEnabled() {
  const provider = getAPIProvider()
  const model = getMainLoopModel()
  if (provider === 'firstParty') return true
  if (provider === 'vertex') {
    return model.includes('claude-opus-4') ||
           model.includes('claude-sonnet-4') ||
           model.includes('claude-haiku-4')
  }
  if (provider === 'foundry') return true
  return false
}
```

### 7.2.6 输入验证

```typescript
async validateInput(input) {
  if (!query.length) return { result: false, message: 'Error: Missing query' }
  if (allowed_domains?.length && blocked_domains?.length) {
    return {
      result: false,
      message: 'Error: Cannot specify both allowed_domains and blocked_domains'
    }
  }
  return { result: true }
}
```

## 7.3 WebFetchTool：网页内容的获取与解析

WebFetchTool 是 WebSearchTool 的补充——当 AI 需要深入了解某个网页的具体内容时，它负责获取和解析网页。

### 7.3.1 输入输出 Schema

```typescript
// 文件: src/tools/WebFetchTool/WebFetchTool.ts
const inputSchema = lazySchema(() =>
  z.strictObject({
    url: z.string().url().describe('The URL to fetch content from'),
    prompt: z.string().describe('The prompt to run on the fetched content'),
  })
)

const outputSchema = lazySchema(() =>
  z.object({
    bytes: z.number(),
    code: z.number(),
    codeText: z.string(),
    result: z.string(),
    durationMs: z.number(),
    url: z.string(),
  })
)
```

关键设计：`prompt` 字段是必填的——WebFetchTool 不是简单地返回原始内容，而是要求调用者指定要从页面中提取什么信息。

### 7.3.2 预批准域名系统

`preapproved.ts` 定义了一组预批准域名，这些代码相关的域名不需要用户授权即可获取：

```typescript
export const PREAPPROVED_HOSTS = new Set([
  'platform.claude.com', 'code.claude.com',
  'modelcontextprotocol.io', 'github.com/anthropics', 'agentskills.io',
  'docs.python.org', 'go.dev', 'pkg.go.dev',
  'developer.mozilla.org',  // MDN
  'learn.microsoft.com', 'docs.oracle.com',
  'kotlinlang.org', 'ruby-doc.org',
  'docs.swift.org', 'www.php.net',
  'en.cppreference.com',
  'httpd.apache.org',
  // ... 更多编程语言文档站点
])
```

域名检查使用两级索引：主机名精确匹配（O(1) Set 查找）和路径前缀匹配（用于如 `github.com/anthropics` 这样的路径作用域条目）。路径匹配强制执行段边界——`/anthropics` 不会匹配 `/anthropics-evil/malware`。

### 7.3.3 获取流程

`getURLMarkdownContent` 实现了完整的获取管道：

```typescript
async function getURLMarkdownContent(url, abortController) {
  // 1. URL 验证（长度 ≤ 2000、无用户名/密码、可公开解析的主机名）
  if (!validateURL(url)) throw new Error('Invalid URL')

  // 2. 15 分钟 LRU 缓存检查（50MB 大小限制）
  const cachedEntry = URL_CACHE.get(url)
  if (cachedEntry) return cachedEntry

  // 3. HTTP → HTTPS 协议升级
  if (parsedUrl.protocol === 'http:') {
    parsedUrl.protocol = 'https:'
  }

  // 4. 域名黑名单预检（通过 api.anthropic.com）
  const checkResult = await checkDomainBlocklist(hostname)

  // 5. 安全重定向处理
  const response = await getWithPermittedRedirects(
    upgradedUrl, signal, isPermittedRedirect
  )

  // 6. HTML → Markdown 转换（使用 Turndown）
  if (contentType.includes('text/html')) {
    markdownContent = (await getTurndownService()).turndown(htmlContent)
  }

  // 7. 二进制内容持久化（PDF 等）
  if (isBinaryContentType(contentType)) {
    await persistBinaryContent(rawBuffer, contentType, persistId)
  }
}
```

### 7.3.4 安全重定向处理

重定向遵循严格的同源策略（`isPermittedRedirect`）：

```typescript
function isPermittedRedirect(originalUrl, redirectUrl): boolean {
  // 协议必须相同
  if (parsedRedirect.protocol !== parsedOriginal.protocol) return false
  // 端口必须相同
  if (parsedRedirect.port !== parsedOriginal.port) return false
  // 不允许用户名/密码
  if (parsedRedirect.username || parsedRedirect.password) return false
  // 主机名匹配（允许添加/移除 www. 前缀）
  return stripWww(original.hostname) === stripWww(redirect.hostname)
}
```

跨域重定向不会自动跟随——工具会返回重定向信息，让 AI 用新的 URL 再次调用 WebFetchTool。这防止了开放重定向漏洞被利用。

### 7.3.5 辅助模型处理

`applyPromptToMarkdown` 使用辅助模型（Haiku）处理获取的内容：

```typescript
async function applyPromptToMarkdown(
  prompt, markdownContent, signal, isNonInteractiveSession, isPreapprovedDomain
) {
  // 截断到 100,000 字符
  const truncatedContent = markdownContent.length > MAX_MARKDOWN_LENGTH
    ? markdownContent.slice(0, MAX_MARKDOWN_LENGTH) + '\n\n[Content truncated...]'
    : markdownContent

  const modelPrompt = makeSecondaryModelPrompt(
    truncatedContent, prompt, isPreapprovedDomain
  )
  const assistantMessage = await queryHaiku({
    systemPrompt: asSystemPrompt([]),
    userPrompt: modelPrompt,
    signal,
  })
}
```

对于预批准域名，辅助模型被允许引用更多内容；对于非预批准域名，辅助模型有严格的引用限制（125 字符最大引用长度）。

### 7.3.6 权限模型

WebFetchTool 的权限检查（`checkPermissions`）使用域名级别的规则匹配：

```typescript
async checkPermissions(input, context) {
  // 预批准域名直接放行
  if (isPreapprovedHost(parsedUrl.hostname, parsedUrl.pathname)) {
    return { behavior: 'allow', decisionReason: { type: 'other', reason: 'Preapproved host' } }
  }
  // 规则匹配：deny → ask → allow → 默认 ask
  const ruleContent = `domain:${hostname}`
  // 检查 deny 规则
  // 检查 ask 规则
  // 检查 allow 规则
  // 默认返回 ask（需要用户确认）
}
```

## 7.4 ToolSearchTool：工具自发现机制

ToolSearchTool 是一个元工具——它帮助 AI 发现和加载延迟加载的工具。在 Claude Code 中，许多工具（特别是 MCP 工具）不会在初始化时全部加载，而是按需获取。

### 7.4.1 延迟加载策略

`isDeferredTool` 函数决定哪些工具需要通过 ToolSearchTool 才能使用：

```typescript
// 文件: src/tools/ToolSearchTool/prompt.ts
export function isDeferredTool(tool: Tool): boolean {
  // 显式 opt-out：始终加载
  if (tool.alwaysLoad === true) return false
  // MCP 工具始终延迟加载
  if (tool.isMcp === true) return true
  // ToolSearch 自身不延迟
  if (tool.name === TOOL_SEARCH_TOOL_NAME) return false
  // Agent 工具在 Fork 实验中不延迟
  if (feature('FORK_SUBAGENT') && tool.name === AGENT_TOOL_NAME) {
    if (isForkSubagentEnabled()) return false
  }
  // 其他工具根据 shouldDefer 标志决定
  return tool.shouldDefer === true
}
```

### 7.4.2 输入 Schema 与查询形式

```typescript
// 文件: src/tools/ToolSearchTool/ToolSearchTool.ts
const inputSchema = lazySchema(() =>
  z.object({
    query: z.string().describe(
      'Query to find deferred tools. Use "select:<tool_name>" for ' +
      'direct selection, or keywords to search.'
    ),
    max_results: z.number().optional().default(5)
      .describe('Maximum number of results to return (default: 5)'),
  })
)

const outputSchema = lazySchema(() =>
  z.object({
    matches: z.array(z.string()),
    query: z.string(),
    total_deferred_tools: z.number(),
    pending_mcp_servers: z.array(z.string()).optional(),
  })
)
```

支持三种查询形式：

1. **精确选择**：`select:Read,Edit,Grep` — 通过名称获取指定工具
2. **关键词搜索**：`notebook jupyter` — 返回最佳匹配
3. **必选词搜索**：`+slack send` — `+` 前缀的词必须出现在工具名中

### 7.4.3 搜索算法

`searchToolsWithKeywords` 实现了一个基于评分的搜索：

```typescript
async function searchToolsWithKeywords(
  query, deferredTools, tools, maxResults
): Promise<string[]> {
  // 快速路径：精确名称匹配
  const exactMatch = deferredTools.find(t => t.name.toLowerCase() === queryLower)
  if (exactMatch) return [exactMatch.name]

  // MCP 前缀匹配
  if (queryLower.startsWith('mcp__') && queryLower.length > 5) {
    const prefixMatches = deferredTools
      .filter(t => t.name.toLowerCase().startsWith(queryLower))
    if (prefixMatches.length > 0) return prefixMatches
  }

  // 关键词评分搜索
  // 必选词（+前缀）必须全部匹配
  // 可选词按匹配度评分
  // MCP 工具名部分匹配权重 12，普通工具 10
  // searchHint 匹配权重 4
  // 描述匹配权重 2
}
```

`parseToolName` 函数处理两种工具命名约定：

```typescript
function parseToolName(name: string) {
  if (name.startsWith('mcp__')) {
    // MCP 工具：mcp__server__action → ['server', 'action']
    const parts = withoutPrefix.split('__').flatMap(p => p.split('_'))
    return { parts, isMcp: true }
  }
  // 普通工具：CamelCase → ['camel', 'case']
  const parts = name
    .replace(/([a-z])([A-Z])/g, '$1 $2')
    .toLowerCase().split(/\s+/)
  return { parts, isMcp: false }
}
```

### 7.4.4 工具引用机制

当搜索找到匹配时，`mapToolResultToToolResultBlockParam` 返回 `tool_reference` 块：

```typescript
mapToolResultToToolResultBlockParam(content, toolUseID) {
  if (content.matches.length === 0) {
    let text = 'No matching deferred tools found'
    if (content.pending_mcp_servers?.length > 0) {
      text += '. Some MCP servers are still connecting: ' +
              content.pending_mcp_servers.join(', ')
    }
    return { type: 'tool_result', tool_use_id: toolUseID, content: text }
  }
  return {
    type: 'tool_result',
    tool_use_id: toolUseID,
    content: content.matches.map(name => ({
      type: 'tool_reference',
      tool_name: name,
    })),
  }
}
```

`tool_reference` 块由 Anthropic API 的客户端扩展机制处理——被引用的工具 Schema 会自动注入到模型的上下文中。

### 7.4.5 描述缓存

工具描述查询是昂贵的（需要获取每个工具的完整提示），因此使用了 LRU 缓存：

```typescript
const getToolDescriptionMemoized = memoize(
  async (toolName, tools) => {
    const tool = findToolByName(tools, toolName)
    return tool.prompt({ ... })
  },
  (toolName) => toolName  // 按工具名缓存
)
```

当延迟加载的工具集合变化时（通过 `maybeInvalidateCache`），缓存会被自动清除。

## 7.5 StructuredOutputTool：非交互场景的结构化输出

StructuredOutputTool 是一个特殊工具，专为 SDK/CLI 非交互式会话设计，让 Claude 能以结构化 JSON 格式返回结果。

### 7.5.1 基础定义

```typescript
// 文件: src/tools/SyntheticOutputTool/SyntheticOutputTool.ts
export const SYNTHETIC_OUTPUT_TOOL_NAME = 'StructuredOutput'

export function isSyntheticOutputToolEnabled(opts: {
  isNonInteractiveSession: boolean
}): boolean {
  return opts.isNonInteractiveSession
}
```

它只在非交互会话中启用——典型的交互式聊天不需要结构化输出。

### 7.5.2 动态 Schema 验证

`createSyntheticOutputTool` 是工厂函数，接受用户提供的 JSON Schema，创建配置好的工具实例：

```typescript
export function createSyntheticOutputTool(
  jsonSchema: Record<string, unknown>,
): CreateResult {
  // 1. 模式缓存（WeakMap 按引用 identity）
  const cached = toolCache.get(jsonSchema)
  if (cached) return cached

  // 2. 使用 Ajv 验证 Schema 本身
  const ajv = new Ajv({ allErrors: true })
  const isValidSchema = ajv.validateSchema(jsonSchema)
  if (!isValidSchema) {
    return { error: ajv.errorsText(ajv.errors) }
  }

  // 3. 编译 Schema 为验证函数
  const validateSchema = ajv.compile(jsonSchema)

  // 4. 返回配置好的工具，call 方法中执行验证
  return {
    tool: {
      ...SyntheticOutputTool,
      inputJSONSchema: jsonSchema,
      async call(input) {
        const isValid = validateSchema(input)
        if (!isValid) {
          throw new TelemetrySafeError(
            `Output does not match required schema: ${errors}`
          )
        }
        return {
          data: 'Structured output provided successfully',
          structured_output: input,
        }
      },
    }
  }
}
```

### 7.5.3 性能优化：Identity 缓存

工作流脚本可能在一次运行中对同一个 Schema 对象调用 `agent({schema: BUGS_SCHEMA})` 30-80 次。使用 `WeakMap` 缓存避免了每次调用都重新创建 Ajv 实例和编译 Schema（约 1.4ms 的 JIT 代码生成）：

```typescript
const toolCache = new WeakMap<object, CreateResult>()
```

`WeakMap` 按对象引用（identity）缓存，不会阻止垃圾回收。80 次调用的工作流从约 110ms 降到约 4ms 的 Ajv 开销。

### 7.5.4 安全设计

StructuredOutputTool 的权限检查始终返回 `allow`：

```typescript
async checkPermissions(input): Promise<PermissionResult> {
  return { behavior: 'allow', updatedInput: input }
}
```

这是合理的——它只是一个数据返回工具，不执行任何外部操作。

## 7.6 小结与思考

本章分析的五类工具代表了 AI 与外部世界交互的五种模式：

| 工具 | 交互模式 | 核心价值 | 安全机制 |
|------|----------|----------|----------|
| BashTool | 命令执行 | 系统操作能力 | 20+ 验证器纵深防御 |
| WebSearchTool | 信息检索 | 实时信息获取 | 域名过滤、双模型架构 |
| WebFetchTool | 内容获取 | 深度内容理解 | 域名黑名单、安全重定向 |
| ToolSearchTool | 自省查询 | 工具集感知 | 只读、并发安全 |
| StructuredOutputTool | 结构化输出 | SDK 集成 | Schema 验证 |

这些工具共同构成了 Claude Code 的**外部交互层**，让 AI 能够突破自身模型的限制，与真实的计算环境和信息世界交互。

从设计模式的角度看：
- BashTool 的安全管道体现了**责任链模式**——20 个验证器有序执行，每个只关注一种威胁
- WebSearchTool 的双模型架构体现了**代理模式**——辅助模型作为搜索代理
- WebFetchTool 的缓存和预检体现了**防御性编程**——多层保护确保安全获取
- ToolSearchTool 的评分搜索体现了**策略模式**——不同查询形式使用不同的搜索策略
- StructuredOutputTool 的工厂函数体现了**工厂模式**——根据 Schema 动态创建工具实例

**思考题**：

1. BashTool 的 20 个验证器为什么不短路？如果在某个非误解析验证器返回 `ask` 后就停止，可能导致什么安全问题？
2. WebFetchTool 的安全重定向策略为什么只允许同域（strip www 后匹配）？如果你是攻击者，如何尝试利用开放重定向？
3. ToolSearchTool 的延迟加载机制在什么场景下最有价值？如果所有工具都预先加载，会有什么问题？
4. StructuredOutputTool 为什么使用 `WeakMap` 而非普通 `Map` 缓存？这对工作流脚本的内存管理有什么影响？
