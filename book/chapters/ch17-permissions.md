# 第17章 权限系统 —— 安全计算的守门人

> "自由不是想做什么就做什么，而是有能力选择不做什么。" —— 沙德的哲学

Claude Code 作为一个能够执行真实代码、修改文件系统、调用外部服务的 AI 系统，其权限系统构成了整个应用安全架构的基石。这不是一个简单的"允许/拒绝"开关，而是一个精心设计、多层次、可配置的权限决策引擎。

本章将深入剖析 Claude Code 的权限系统设计，从权限模式的概念到具体的决策流程，从规则匹配到沙箱机制，揭示这个安全守门人的工作原理。

## 17.1 PermissionMode：权限模式的分类

权限模式（PermissionMode）是 Claude Code 权限系统的顶层设计概念，它定义了系统在不同场景下的默认行为策略。源码中的权限模式定义位于 `/src/types/permissions.ts` 和 `/src/utils/permissions/PermissionMode.ts`。

### 17.1.1 权限模式的全貌

```typescript
// 用户可见的外部权限模式
export const EXTERNAL_PERMISSION_MODES = [
  'acceptEdits',      // 接受编辑模式
  'bypassPermissions',// 绕过权限模式
  'default',          // 默认模式
  'dontAsk',          // 禁止询问模式
  'plan',             // 计划模式
] as const

// 内部完整的权限模式（包含 ant-only 特性）
export type InternalPermissionMode = ExternalPermissionMode | 'auto' | 'bubble'
export type PermissionMode = InternalPermissionMode
```

这个设计体现了两个重要的工程决策：

1. **内外分离**：用户通过 UI 和设置文件只能配置 `ExternalPermissionMode`，而内部系统可能包含额外的模式（如 ant-only 的 `auto` 模式）
2. **特性开关控制**：通过 `feature('TRANSCRIPT_CLASSIFIER')` 动态决定是否启用 `auto` 模式

### 17.1.2 各模式的语义与配置

每个权限模式都有完整的配置定义：

```typescript
type PermissionModeConfig = {
  title: string        // 完整标题
  shortTitle: string   // 短标题（UI显示）
  symbol: string       // 符号标识
  color: ModeColorKey  // 颜色主题
  external: ExternalPermissionMode  // 对应的外部模式
}

const PERMISSION_MODE_CONFIG: Partial<Record<PermissionMode, PermissionModeConfig>> = {
  default: {
    title: 'Default',
    shortTitle: 'Default',
    symbol: '',
    color: 'text',
    external: 'default',
  },
  plan: {
    title: 'Plan Mode',
    shortTitle: 'Plan',
    symbol: PAUSE_ICON,
    color: 'planMode',
    external: 'plan',
  },
  acceptEdits: {
    title: 'Accept edits',
    shortTitle: 'Accept',
    symbol: '⏵⏵',
    color: 'autoAccept',
    external: 'acceptEdits',
  },
  bypassPermissions: {
    title: 'Bypass Permissions',
    shortTitle: 'Bypass',
    symbol: '⏵⏵',
    color: 'error',
    external: 'bypassPermissions',
  },
  // ... auto 模式配置（ant-only）
}
```

这个配置驱动了 UI 的显示、状态栏的颜色、模式切换的逻辑等多个方面。

### 17.1.3 模式切换的图灵完备性

`getNextPermissionMode` 函数定义了模式切换的图灵机逻辑：

```typescript
export function getNextPermissionMode(
  toolPermissionContext: ToolPermissionContext,
  _teamContext?: { leadAgentId: string },
): PermissionMode {
  switch (toolPermissionContext.mode) {
    case 'default':
      // Ant 用户跳过 acceptEdits 和 plan，直接到 bypass 或 auto
      if (process.env.USER_TYPE === 'ant') {
        if (toolPermissionContext.isBypassPermissionsModeAvailable) {
          return 'bypassPermissions'
        }
        if (canCycleToAuto(toolPermissionContext)) {
          return 'auto'
        }
        return 'default'
      }
      return 'acceptEdits'

    case 'acceptEdits':
      return 'plan'

    case 'plan':
      if (toolPermissionContext.isBypassPermissionsModeAvailable) {
        return 'bypassPermissions'
      }
      if (canCycleToAuto(toolPermissionContext)) {
        return 'auto'
      }
      return 'default'

    // ... 其他状态转换
  }
}
```

这个状态机设计确保了：
1. 用户只能通过 Shift+Tab 在允许的模式间循环
2. Ant 用户和外部用户有不同的可用模式路径
3. 自动模式（auto）有严格的准入检查

### 17.1.4 自动模式的真实难点：不是切换 UI，而是“危险规则剥离”

当前 `permissionSetup.ts` 里有一段非常关键的逻辑：系统会识别并剥离那些会让 Auto Mode 失去意义的危险规则。

例如对 Bash / PowerShell，下面这类规则都可能被视为危险：

- 整个工具级放行
- `python:*`、`node:*` 这类脚本解释器前缀
- `invoke-expression:*`、`start-process:*` 这类 PowerShell 执行逃逸入口

Auto Mode 的核心并不是“自动点同意”，而是：

1. 先审查已有权限规则
2. 剥离会绕过分类器的危险放行项
3. 再让模型在受控空间里自动执行

这和很多工具的“自动模式”有本质差异。Claude Code 在这里追求的不是最大化自动化，而是 **维持自动化前提下的安全边界可解释性**。

## 17.2 ToolPermissionContext：细粒度的权限规则

`ToolPermissionContext` 是权限检查的核心上下文对象，它包含了所有进行权限决策所需的信息。这个类型的设计体现了"上下文即配置"的理念。

### 17.2.1 上下文结构解析

```typescript
export type ToolPermissionContext = {
  readonly mode: PermissionMode  // 当前权限模式
  readonly additionalWorkingDirectories: ReadonlyMap<
    string,
    AdditionalWorkingDirectory
  >
  readonly alwaysAllowRules: ToolPermissionRulesBySource  // 自动允许规则
  readonly alwaysDenyRules: ToolPermissionRulesBySource   // 自动拒绝规则
  readonly alwaysAskRules: ToolPermissionRulesBySource    // 自动询问规则
  readonly isBypassPermissionsModeAvailable: boolean      // bypass模式是否可用
  readonly strippedDangerousRules?: ToolPermissionRulesBySource  // 已剥离的危险规则
  readonly shouldAvoidPermissionPrompts?: boolean          // 是否避免权限提示
  readonly awaitAutomatedChecksBeforeDialog?: boolean      // 是否等待自动化检查
  readonly prePlanMode?: PermissionMode                    // 进入plan前的模式
}
```

### 17.2.2 规则来源的多样性

规则可以来自多个来源（`PermissionRuleSource`）：

```typescript
export type PermissionRuleSource =
  | 'userSettings'      // 用户设置
  | 'projectSettings'   // 项目设置
  | 'localSettings'     // 本地设置
  | 'flagSettings'      // 特性标志设置
  | 'policySettings'    // 策略设置（企业策略）
  | 'cliArg'            // 命令行参数
  | 'command'           // 命令指定
  | 'session'           // 会话临时规则
```

这种多来源设计支持了从企业级策略到临时会话规则的各种使用场景。

### 17.2.3 规则值的结构

每条权限规则都有明确的值结构：

```typescript
export type PermissionRuleValue = {
  toolName: string      // 工具名称
  ruleContent?: string  // 规则内容（可选）
}

export type PermissionRule = {
  source: PermissionRuleSource       // 规则来源
  ruleBehavior: PermissionBehavior   // 规则行为
  ruleValue: PermissionRuleValue     // 规则值
}

export type PermissionBehavior = 'allow' | 'deny' | 'ask'
```

例如：
- `{toolName: "Bash"}` → 对整个 Bash 工具的规则
- `{toolName: "Bash", ruleContent: "npm:*"}` → 对 `npm` 命令的前缀规则
- `{toolName: "Bash", ruleContent: "git commit"}` → 对特定命令的精确规则

## 17.3 alwaysAllow/alwaysDeny/alwaysAsk的三级策略

Claude Code 的权限系统采用了三级策略设计，每种行为（allow/deny/ask）都有独立的规则集合。

### 17.3.1 规则检索的工厂模式

系统通过一组工厂函数获取特定行为的规则：

```typescript
export function getAllowRules(
  context: ToolPermissionContext,
): PermissionRule[] {
  return PERMISSION_RULE_SOURCES.flatMap(source =>
    (context.alwaysAllowRules[source] || []).map(ruleString => ({
      source,
      ruleBehavior: 'allow',
      ruleValue: permissionRuleValueFromString(ruleString),
    })),
  )
}

export function getDenyRules(context: ToolPermissionContext): PermissionRule[] { /* ... */ }
export function getAskRules(context: ToolPermissionContext): PermissionRule[] { /* ... */ }
```

这种设计使得规则可以高效地从多个来源聚合和转换。

### 17.3.2 工具级与内容级规则匹配

权限检查分为两个层次：

**工具级匹配**：检查整个工具是否被规则覆盖

```typescript
function toolMatchesRule(
  tool: Pick<Tool, 'name' | 'mcpInfo'>,
  rule: PermissionRule,
): boolean {
  // 规则必须有内容才能匹配工具级
  if (rule.ruleValue.ruleContent !== undefined) {
    return false
  }

  const nameForRuleMatch = getToolNameForPermissionCheck(tool)

  // 直接工具名匹配
  if (rule.ruleValue.toolName === nameForRuleMatch) {
    return true
  }

  // MCP 服务器级权限匹配
  // 例如规则 "mcp__server1" 匹配工具 "mcp__server1__tool1"
  const ruleInfo = mcpInfoFromString(rule.ruleValue.toolName)
  const toolInfo = mcpInfoFromString(nameForRuleMatch)

  return (
    ruleInfo !== null &&
    toolInfo !== null &&
    (ruleInfo.toolName === undefined || ruleInfo.toolName === '*') &&
    ruleInfo.serverName === toolInfo.serverName
  )
}
```

**内容级匹配**：由各个工具自己实现 `checkPermissions` 方法

```typescript

### 17.3.3 权限系统已经不只是本地规则，还叠加了组织级限制

如果只讲 `alwaysAllow / alwaysDeny / alwaysAsk`，这套权限模型仍然是不完整的。`main.tsx` 在启动时还会加载两套组织级约束：

1. **Remote Managed Settings**
   由服务端下发设置，经过 checksum 校验、本地缓存和安全检查后参与本地配置解析。

2. **Policy Limits**
   由组织策略决定哪些能力被禁用或裁剪，例如某些远程能力、企业受控功能等。

它们和权限规则的关系可以概括为：

- **权限规则** 决定“这个工具调用是否需要允许”
- **远程设置** 决定“会话默认行为与配置如何被组织统一管理”
- **策略限制** 决定“某类能力是否根本不该暴露”

这三层合起来，Claude Code 的安全模型才算完整。

权限系统的边界已经从单机 REPL 扩展到企业治理面。
// Bash 工具的实现示例（简化）
async checkPermissions(input: ParsedInput, context: ToolUseContext): Promise<PermissionResult> {
  // 1. 检查命令前缀规则
  const prefixRules = getRuleByContentsForTool(context, this, 'allow')
  for (const [prefix, rule] of prefixRules) {
    if (input.command.startsWith(prefix)) {
      return { behavior: 'allow', decisionReason: { type: 'rule', rule } }
    }
  }

  // 2. 检查安全模式
  if (this.isDangerousCommand(input.command)) {
    return { behavior: 'ask', decisionReason: { type: 'safetyCheck', reason: '...' } }
  }

  // 3. 默认行为
  return { behavior: 'passthrough', message: '...' }
}
```

### 17.3.4 决策管道的完整流程

`hasPermissionsToUseTool` 函数实现了完整的权限决策管道：

```typescript
export const hasPermissionsToUseTool: CanUseToolFn = async (
  tool, input, context, assistantMessage, toolUseID,
): Promise<PermissionDecision> => {
  // 第一步：基于规则的检查（绕过模式也会执行这些检查）
  const result = await hasPermissionsToUseToolInner(tool, input, context)

  // 成功时重置连续拒绝计数
  if (result.behavior === 'allow') {
    // ... 重置拒绝状态
    return result
  }

  // 第二步：应用模式转换
  if (result.behavior === 'ask') {
    const appState = context.getAppState()

    // dontAsk 模式：将 ask 转换为 deny
    if (appState.toolPermissionContext.mode === 'dontAsk') {
      return {
        behavior: 'deny',
        decisionReason: { type: 'mode', mode: 'dontAsk' },
        message: DONT_ASK_REJECT_MESSAGE(tool.name),
      }
    }

    // auto 模式：使用 AI 分类器代替用户提示
    if (appState.toolPermissionContext.mode === 'auto') {
      // ... 运行分类器逻辑
    }

    // 无头代理：自动拒绝（除非 hook 允许）
    if (appState.toolPermissionContext.shouldAvoidPermissionPrompts) {
      // ... 运行 hook 逻辑
    }
  }

  return result
}
```

内部函数 `hasPermissionsToUseToolInner` 执行基础的规则检查：

```typescript
async function hasPermissionsToUseToolInner(
  tool: Tool,
  input: { [key: string]: unknown },
  context: ToolUseContext,
): Promise<PermissionDecision> {
  const appState = context.getAppState()

  // 1a. 检查工具是否被整体拒绝
  const denyRule = getDenyRuleForTool(appState.toolPermissionContext, tool)
  if (denyRule) {
    return {
      behavior: 'deny',
      decisionReason: { type: 'rule', rule: denyRule },
      message: `Permission to use ${tool.name} has been denied.`,
    }
  }

  // 1b. 检查工具是否总是询问
  const askRule = getAskRuleForTool(appState.toolPermissionContext, tool)
  if (askRule) {
    // 沙箱自动允许的例外处理
    const canSandboxAutoAllow = /* ... */
    if (!canSandboxAutoAllow) {
      return {
        behavior: 'ask',
        decisionReason: { type: 'rule', rule: askRule },
        message: createPermissionRequestMessage(tool.name),
      }
    }
  }

  // 1c. 调用工具自身的权限检查
  let toolPermissionResult = await tool.checkPermissions(parsedInput, context)

  // 1d. 工具拒绝了权限
  if (toolPermissionResult?.behavior === 'deny') {
    return toolPermissionResult
  }

  // 1e. 工具需要用户交互（即使绕过模式）
  if (tool.requiresUserInteraction?.() && toolPermissionResult?.behavior === 'ask') {
    return toolPermissionResult
  }

  // 1f. 内容级 ask 规则优先于绕过模式
  if (toolPermissionResult?.behavior === 'ask' &&
      toolPermissionResult.decisionReason?.type === 'rule' &&
      toolPermissionResult.decisionReason.rule.ruleBehavior === 'ask') {
    return toolPermissionResult
  }

  // 1g. 安全检查（.git/, .claude/ 等）免疫绕过
  if (toolPermissionResult?.behavior === 'ask' &&
      toolPermissionResult.decisionReason?.type === 'safetyCheck') {
    return toolPermissionResult
  }

  // 2a. 检查模式是否允许运行
  const shouldBypassPermissions =
    appState.toolPermissionContext.mode === 'bypassPermissions' ||
    (appState.toolPermissionContext.mode === 'plan' &&
     appState.toolPermissionContext.isBypassPermissionsModeAvailable)

  if (shouldBypassPermissions) {
    return {
      behavior: 'allow',
      decisionReason: { type: 'mode', mode: appState.toolPermissionContext.mode },
    }
  }

  // 2b. 检查工具是否总是被允许
  const alwaysAllowedRule = toolAlwaysAllowedRule(appState.toolPermissionContext, tool)
  if (alwaysAllowedRule) {
    return {
      behavior: 'allow',
      decisionReason: { type: 'rule', rule: alwaysAllowedRule },
    }
  }

  // 3. 将 passthrough 转换为 ask
  return toolPermissionResult.behavior === 'passthrough'
    ? { ...toolPermissionResult, behavior: 'ask' }
    : toolPermissionResult
}
```

这个管道设计确保了：
1. 拒绝规则始终优先
2. 安全检查免疫绕过模式
3. 内容级 ask 规则不能被绕过
4. 模式转换发生在规则检查之后

## 17.4 沙箱机制：Bash命令的安全边界

沙箱机制是 Bash 工具的安全边界，它限制了命令执行的环境和范围。

### 17.4.1 沙箱决策逻辑

`shouldUseSandbox` 函数决定了是否使用沙箱：

```typescript
export function shouldUseSandbox(input: Partial<SandboxInput>): boolean {
  if (!SandboxManager.isSandboxingEnabled()) {
    return false
  }

  // 用户明确禁用沙箱且允许非沙箱命令
  if (input.dangerouslyDisableSandbox &&
      SandboxManager.areUnsandboxedCommandsAllowed()) {
    return false
  }

  if (!input.command) {
    return false
  }

  // 命令包含用户配置的排除命令时不使用沙箱
  if (containsExcludedCommand(input.command)) {
    return false
  }

  return true
}
```

### 17.4.2 排除命令的匹配

`containsExcludedCommand` 函数实现了复杂的命令匹配逻辑：

```typescript
function containsExcludedCommand(command: string): boolean {
  // 检查动态配置的禁用命令（ant-only）
  if (process.env.USER_TYPE === 'ant') {
    const disabledCommands = getFeatureValue_CACHED_MAY_BE_STALE(
      'tengu_sandbox_disabled_commands',
      { commands: [], substrings: [] }
    )

    // 子串检查
    for (const substring of disabledCommands.substrings) {
      if (command.includes(substring)) {
        return true
      }
    }

    // 命令起始检查
    const commandParts = splitCommand_DEPRECATED(command)
    for (const part of commandParts) {
      const baseCommand = part.trim().split(' ')[0]
      if (baseCommand && disabledCommands.commands.includes(baseCommand)) {
        return true
      }
    }
  }

  // 检查用户配置的排除命令
  const userExcludedCommands = settings.sandbox?.excludedCommands ?? []
  if (userExcludedCommands.length === 0) {
    return false
  }

  // 分解复合命令并检查每个子命令
  let subcommands: string[]
  try {
    subcommands = splitCommand_DEPRECATED(command)
  } catch {
    subcommands = [command]
  }

  for (const subcommand of subcommands) {
    // 生成多个候选命令（剥离环境变量和包装器）
    const candidates = generateCandidates(subcommand)

    for (const pattern of userExcludedCommands) {
      const rule = bashPermissionRule(pattern)
      for (const cand of candidates) {
        if (ruleMatches(rule, cand)) {
          return true
        }
      }
    }
  }

  return false
}
```

### 17.4.3 沙箱与自动允许的交互

当 `autoAllowBashIfSandboxed` 启用时，沙箱内的命令可以自动允许：

```typescript
// 在 ask 规则检查中
const canSandboxAutoAllow =
  tool.name === BASH_TOOL_NAME &&
  SandboxManager.isSandboxingEnabled() &&
  SandboxManager.isAutoAllowBashIfSandboxedEnabled() &&
  shouldUseSandbox(input)

if (!canSandboxAutoAllow) {
  return {
    behavior: 'ask',
    decisionReason: { type: 'rule', rule: askRule },
    message: createPermissionRequestMessage(tool.name),
  }
}
```

这个设计体现了"安全即自由"的理念：在安全的沙箱环境中，可以给予更多的自由。

## 17.5 denialTracking：拒绝状态的追踪与学习

拒绝追踪（DenialTracking）是自动模式中的一个重要机制，它跟踪连续拒绝和总拒绝次数，以决定何时回退到人工提示。

### 17.5.1 追踪状态的结构

```typescript
export type DenialTrackingState = {
  consecutiveDenials: number  // 连续拒绝次数
  totalDenials: number        // 总拒绝次数
}

export const DENIAL_LIMITS = {
  maxConsecutive: 3,  // 最多连续拒绝3次
  maxTotal: 20,       // 最多总拒绝20次
} as const
```

### 17.5.2 状态更新逻辑

```typescript
export function recordDenial(state: DenialTrackingState): DenialTrackingState {
  return {
    ...state,
    consecutiveDenials: state.consecutiveDenials + 1,
    totalDenials: state.totalDenials + 1,
  }
}

export function recordSuccess(state: DenialTrackingState): DenialTrackingState {
  if (state.consecutiveDenials === 0) return state
  return {
    ...state,
    consecutiveDenials: 0,  // 成功时重置连续计数
  }
}

export function shouldFallbackToPrompting(state: DenialTrackingState): boolean {
  return (
    state.consecutiveDenials >= DENIAL_LIMITS.maxConsecutive ||
    state.totalDenials >= DENIAL_LIMITS.maxTotal
  )
}
```

### 17.5.3 在权限决策中的应用

```typescript
// 在 hasPermissionsToUseTool 中
if (result.behavior === 'allow') {
  // 成功时重置连续拒绝
  const currentDenialState = context.localDenialTracking ?? appState.denialTracking
  if (appState.toolPermissionContext.mode === 'auto' &&
      currentDenialState &&
      currentDenialState.consecutiveDenials > 0) {
    const newDenialState = recordSuccess(currentDenialState)
    persistDenialState(context, newDenialState)
  }
  return result
}

// 分类器拒绝时
if (classifierResult.shouldBlock) {
  const newDenialState = recordDenial(denialState)
  persistDenialState(context, newDenialState)

  // 检查是否超过限制
  const denialLimitResult = handleDenialLimitExceeded(
    newDenialState,
    appState,
    classifierResult.reason,
    assistantMessage,
    tool,
    result,
    context,
  )
  if (denialLimitResult) {
    return denialLimitResult  // 回退到人工提示
  }

  return {
    behavior: 'deny',
    decisionReason: { type: 'classifier', classifier: 'auto-mode', reason: classifierResult.reason },
    message: buildYoloRejectionMessage(classifierResult.reason),
  }
}
```

### 17.5.4 拒绝限制处理

```typescript
function handleDenialLimitExceeded(
  denialState: DenialTrackingState,
  appState: /* ... */,
  classifierReason: string,
  /* ... */
): PermissionDecision | null {
  if (!shouldFallbackToPrompting(denialState)) {
    return null
  }

  const hitTotalLimit = denialState.totalDenials >= DENIAL_LIMITS.maxTotal
  const isHeadless = appState.toolPermissionContext.shouldAvoidPermissionPrompts

  const warning = hitTotalLimit
    ? `${denialState.totalDenials} actions were blocked this session. Please review the transcript before continuing.`
    : `${denialState.consecutiveDenials} consecutive actions were blocked. Please review the transcript before continuing.`

  // 无头模式下抛出异常终止代理
  if (isHeadless) {
    throw new AbortError('Agent aborted: too many classifier denials in headless mode')
  }

  // 达到总限制时重置计数
  if (hitTotalLimit) {
    persistDenialState(context, {
      ...denialState,
      totalDenials: 0,
      consecutiveDenials: 0,
    })
  }

  // 回退到人工提示
  return {
    ...result,
    decisionReason: {
      type: 'classifier',
      classifier: 'auto-mode',
      reason: `${warning}\n\nLatest blocked action: ${classifierReason}`,
    },
  }
}
```

这个设计确保了：
1. 分类器错误不会永久阻止用户
2. 连续失败时让用户介入是合理的
3. 总拒绝次数限制防止过度依赖分类器

## 17.6 小结与思考

Claude Code 的权限系统是一个设计精良的安全架构，它体现了以下重要原则：

### 17.6.1 分层防御（Defense in Depth）

1. **规则层**：基于工具和内容的规则匹配
2. **模式层**：基于当前权限模式的行为转换
3. **沙箱层**：命令执行环境的隔离
4. **分类器层**：AI驱动的智能决策（auto 模式）

每一层都能独立阻止危险操作，多层的组合构成了纵深防御体系。

### 17.6.2 默认安全（Secure by Default）

- 默认模式需要用户确认大多数操作
- 安全检查（.git/, .claude/ 等）免疫绕过模式
- 危险模式（bypass）有明确的警告和限制

### 17.6.3 可配置性与可用性的平衡

- 多种权限模式满足不同使用场景
- 细粒度的规则控制
- 企业级策略支持（policySettings）
- 无头代理的特殊处理

### 17.6.4 AI 辅助安全的探索

Auto 模式代表了权限系统的未来方向：通过 AI 理解上下文来做出更智能的权限决策，同时保留人工介入的机制作为安全网。

这个系统展示了如何在给予 AI 强大能力的同时保持人类控制权 —— 这不仅是技术问题，更是人机协作哲学的体现。权限系统的每一个设计决策都在回答同一个问题：如何让 AI 成为有用的助手，而不是失控的代理人？

正如源码中的注释所强调的：权限提示是安全控制，而不是用户便利功能。这种对安全边界的清晰认识，是构建可靠 AI 系统的基础。
