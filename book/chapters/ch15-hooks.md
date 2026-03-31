# 第15章 Hook 系统 —— 事件驱动的扩展机制

Hook 系统是 Claude Code 最强大的扩展机制之一。它允许用户和插件在特定事件发生时执行自定义代码，从而实现从简单的日志记录到复杂的权限控制等各种功能。本章将深入探讨 Claude Code 的 Hook 系统，了解其设计理念、实现细节和扩展方式。

从源码结构看，Hook 系统已经不是单一文件里的配置解释器，而是由三部分协作完成：

- `src/utils/hooks.ts`：运行时执行引擎与调度逻辑
- `src/utils/hooks/hooksConfigManager.ts`：事件面、配置与来源管理
- `src/types/hooks.ts`：JSON 输出协议、callback hook 类型与校验

Hook 已经从“运行 shell 命令的附加能力”演化成了一个独立的事件驱动扩展子系统。

## 15.1 Hook 分类与组织

Claude Code 的 Hook 系统支持多种事件类型，覆盖了应用的各个方面。这些事件类型在 `src/utils/hooks/hooksConfigManager.ts` 中定义：

### 15.1.1 生命周期事件

生命周期事件在会话的关键阶段触发：

- **SessionStart**：新会话开始时触发，可用于初始化环境变量、加载项目配置
- **SessionEnd**：会话结束时触发，可用于清理资源
- **Setup**：仓库设置时触发（init 或 maintenance），用于项目初始化和维护

```typescript
SessionStart: {
  summary: 'When a new session is started',
  description: 'Input to command is JSON with session start source.\nExit code 0 - stdout shown to Claude\nBlocking errors are ignored\nOther exit codes - show stderr to user only',
  matcherMetadata: {
    fieldToMatch: 'source',
    values: ['startup', 'resume', 'clear', 'compact'],
  },
}
```

### 15.1.2 工具调用事件

工具调用事件在工具执行的不同阶段触发：

- **PreToolUse**：工具执行前触发，可用于修改输入或阻止执行
- **PostToolUse**：工具执行后触发，可用于处理输出或记录日志
- **PostToolUseFailure**：工具执行失败时触发，可用于错误处理

```typescript
PreToolUse: {
  summary: 'Before tool execution',
  description: 'Input to command is JSON of tool call arguments.\nExit code 0 - stdout/stderr not shown\nExit code 2 - show stderr to model and block tool call\nOther exit codes - show stderr to user only but continue with tool call',
  matcherMetadata: {
    fieldToMatch: 'tool_name',
    values: toolNames,
  },
}
```

### 15.1.3 权限事件

权限事件与权限系统交互：

- **PermissionRequest**：权限对话框显示时触发，可用于自动决定权限
- **PermissionDenied**：自动模式拒绝工具调用时触发，可用于重试或通知

```typescript
PermissionRequest: {
  summary: 'When a permission dialog is displayed',
  description: 'Input to command is JSON with tool_name, tool_input, and tool_use_id.\nOutput JSON with hookSpecificOutput containing decision to allow or deny.\nExit code 0 - use hook decision if provided\nOther exit codes - show stderr to user only',
  matcherMetadata: {
    fieldToMatch: 'tool_name',
    values: toolNames,
  },
}
```

### 15.1.4 对话事件

对话事件与用户的交互过程相关：

- **UserPromptSubmit**：用户提交提示时触发，可用于修改或阻止提交
- **Stop**：Claude 即将结束响应时触发，可用于验证条件
- **StopFailure**：API 错误导致响应结束时触发
- **Notification**：发送通知时触发，可用于处理特定通知类型

### 15.1.5 子代理事件

子代理事件与 Agent 工具相关：

- **SubagentStart**：子代理启动时触发
- **SubagentStop**：子代理即将结束时触发
- **TeammateIdle**：队友即将闲置时触发
- **TaskCreated**：任务创建时触发
- **TaskCompleted**：任务完成时触发

### 15.1.6 配置事件

配置事件与配置文件相关：

- **ConfigChange**：配置文件在会话期间更改时触发，可用于阻止更改
- **InstructionsLoaded**：指令文件加载时触发，仅用于观察

### 15.1.7 环境事件

环境事件与环境变化相关：

- **CwdChanged**：工作目录更改后触发
- **FileChanged**：监视的文件更改时触发

### 15.1.8 压缩事件

压缩事件与对话压缩相关：

- **PreCompact**：对话压缩前触发，可用于阻止压缩或添加自定义指令
- **PostCompact**：对话压缩后触发，可用于处理压缩结果

### 15.1.9 MCP 事件

MCP（Model Context Protocol）事件与 MCP 服务器交互：

- **Elicitation**：MCP 服务器请求用户输入时触发
- **ElicitationResult**：用户响应 MCP 请求后触发

### 15.1.10 Worktree 事件

Worktree 事件与 VCS 无关的工作树相关：

- **WorktreeCreate**：创建工作树时触发
- **WorktreeRemove**：删除工作树时触发

## 15.2 生命周期

Hook 的生命周期从配置到执行包含多个阶段：

### 15.2.1 Hook 类型

Claude Code 支持三种类型的 Hook：

1. **命令 Hook（Command Hook）**：执行外部命令
2. **提示 Hook（Prompt Hook）**：使用 LLM 评估条件
3. **代理 Hook（Agent Hook）**：使用 LLM 代理验证条件

```typescript
export type Hook =
  | { type: 'command'; command: string; timeout?: number }
  | { type: 'prompt'; prompt: string; model?: string; timeout?: number }
  | { type: 'agent'; prompt: string; model?: string; timeout?: number }
```

当前实现中还存在一类更底层的 **callback hook**。它不经由外部命令或单独的提示词执行，而是由内部代码直接提供回调函数，并通过 `src/types/hooks.ts` 中的 `HookCallback` 类型接入。这样做的意义在于：

1. 某些内部能力不需要 shell 进程开销
2. 某些 hook 需要直接访问 AppState、attribution 等运行时上下文
3. Hook 体系既能服务用户扩展，也能服务系统内部基础设施

因此，Hook 系统的真实形态更接近“统一事件协议 + 多种执行后端”，而不是单纯的 shell hook 机制。

### 15.2.2 异步 JSON 输出协议

hook 返回值已经是一个结构化协议，而不仅仅是退出码语义。`src/types/hooks.ts` 定义了两类核心输出：

- **同步输出**：直接返回 `continue`、`decision`、`hookSpecificOutput` 等字段
- **异步输出**：返回 `{ async: true }`，由后续流程继续处理

这带来两个重要变化：

1. Hook 不再只是“阻断/放行”的布尔开关
2. Hook 可以把权限决策、补充上下文、watchPaths、MCP elicitation 响应等结构化信息回送给主系统

Hook 已经从“旁路脚本”升级成了 Claude Code 的一个正式控制面。

### 15.2.3 Hook 注册

Hook 可以通过多种方式注册：

1. **配置文件**：在 `settings.json` 或 `CLAUDE.md` 中定义
2. **插件**：插件可以注册自己的 Hook
3. **内置 Hook**：内部使用的 Hook（如分析 Hook）

```typescript
export type HookCallback = {
  type: 'callback'
  callback: (
    input: HookInput,
    toolUseID: string | null,
    abort: AbortSignal | undefined,
    hookIndex?: number,
    context?: HookCallbackContext,
  ) => Promise<HookJSONOutput>
  timeout?: number
  internal?: boolean
}
```

### 15.2.4 Hook 加载

Hook 在会话启动时加载：

```typescript
export function getAllHooks(appState: AppState): IndividualHookConfig[] {
  const hooks: IndividualHookConfig[] = []

  // 从用户设置加载
  const userHooks = appState.settings.hooks || {}
  for (const [event, eventHooks] of Object.entries(userHooks)) {
    for (const hook of eventHooks) {
      hooks.push({
        event: event as HookEvent,
        config: hook,
        source: 'userSettings',
      })
    }
  }

  // 从项目设置加载
  // 从本地设置加载
  // 从策略设置加载
  // 从技能加载

  return hooks
}
```

### 15.2.5 Hook 分组

Hook 按事件和匹配器分组，以便高效执行：

```typescript
export function groupHooksByEventAndMatcher(
  appState: AppState,
  toolNames: string[],
): Record<HookEvent, Record<string, IndividualHookConfig[]>> {
  const grouped: Record<HookEvent, Record<string, IndividualHookConfig[]>> = {
    PreToolUse: {},
    PostToolUse: {},
    // ... 其他事件
  }

  getAllHooks(appState).forEach(hook => {
    const eventGroup = grouped[hook.event]
    if (eventGroup) {
      const matcherKey = hook.matcher || ''
      if (!eventGroup[matcherKey]) {
        eventGroup[matcherKey] = []
      }
      eventGroup[matcherKey].push(hook)
    }
  })

  return grouped
}
```

## 15.3 执行引擎

Hook 执行引擎负责在适当的时候触发 Hook 并处理其结果。

### 15.3.1 同步与异步 Hook

Hook 可以同步或异步执行：

```typescript
export const syncHookResponseSchema = lazySchema(() =>
  z.object({
    continue: z.boolean().optional(),
    suppressOutput: z.boolean().optional(),
    stopReason: z.string().optional(),
    decision: z.enum(['approve', 'block']).optional(),
    reason: z.string().optional(),
    systemMessage: z.string().optional(),
    hookSpecificOutput: z.object({...}).optional(),
  })
)

export const asyncHookResponseSchema = z.object({
  async: z.literal(true),
  asyncTimeout: z.number().optional(),
})
```

异步 Hook 在后台执行，不阻塞主流程。它们注册到 `AsyncHookRegistry` 中：

```typescript
export type PendingAsyncHook = {
  processId: string
  hookId: string
  hookName: string
  hookEvent: HookEvent | 'StatusLine' | 'FileSuggestion'
  toolName?: string
  pluginId?: string
  startTime: number
  timeout: number
  command: string
  responseAttachmentSent: boolean
  shellCommand?: ShellCommand
  stopProgressInterval: () => void
}
```

### 15.3.2 命令 Hook 执行

命令 Hook 执行外部命令并处理其输出：

```typescript
export async function execCommandHook(
  hook: CommandHook,
  hookName: string,
  hookEvent: HookEvent,
  jsonInput: string,
  signal: AbortSignal,
  toolUseContext: ToolUseContext,
  toolUseID?: string,
): Promise<HookResult>
```

执行过程包括：

1. 构建命令环境（设置 `CLAUDE_HOOK_EVENT` 等环境变量）
2. 执行命令
3. 解析输出
4. 处理退出码

退出码的含义：
- **0**：成功
- **1**：非阻塞错误（显示给用户）
- **2**：阻塞错误（阻止操作，显示给模型）
- **其他**：非阻塞错误

### 15.3.3 提示 Hook 执行

提示 Hook 使用 LLM 评估条件：

```typescript
export async function execPromptHook(
  hook: PromptHook,
  hookName: string,
  hookEvent: HookEvent,
  jsonInput: string,
  signal: AbortSignal,
  toolUseContext: ToolUseContext,
  messages?: Message[],
  toolUseID?: string,
): Promise<HookResult>
```

提示 Hook 的特点：

1. 使用快速模型（如 Haiku）
2. 使用 JSON Schema 强制输出格式
3. 设置超时防止长时间等待
4. 返回结构化的结果

```typescript
const response = await queryModelWithoutStreaming({
  messages: messagesToQuery,
  systemPrompt: asSystemPrompt([
    `You are evaluating a hook in Claude Code.

Your response must be a JSON object matching one of the following schemas:
1. If the condition is met, return: {"ok": true}
2. If the condition is not met, return: {"ok": false, "reason": "Reason for why it is not met"}`,
  ]),
  outputFormat: {
    type: 'json_schema',
    schema: {
      type: 'object',
      properties: {
        ok: { type: 'boolean' },
        reason: { type: 'string' },
      },
      required: ['ok'],
      additionalProperties: false,
    },
  },
  // ...
})
```

### 15.3.4 代理 Hook 执行

代理 Hook 使用多轮 LLM 交互验证条件：

```typescript
export async function execAgentHook(
  hook: AgentHook,
  hookName: string,
  hookEvent: HookEvent,
  jsonInput: string,
  signal: AbortSignal,
  toolUseContext: ToolUseContext,
  toolUseID: string | undefined,
  _messages: Message[],
  agentName?: string,
): Promise<HookResult>
```

代理 Hook 的特点：

1. 可以使用工具（如文件读取）
2. 支持多轮交互（最多 50 轮）
3. 使用结构化输出工具返回结果
4. 设置会话规则允许读取会话记录

```typescript
const structuredOutputTool = createStructuredOutputTool()

const systemPrompt = asSystemPrompt([
  `You are verifying a stop condition in Claude Code. Your task is to verify that the agent completed the given plan. The conversation transcript is available at: ${transcriptPath}\nYou can read this file to analyze the conversation history if needed.

Use the available tools to inspect the codebase and verify the condition.
Use as few steps as possible - be efficient and direct.

When done, return your result using the ${SYNTHETIC_OUTPUT_TOOL_NAME} tool with:
- ok: true if the condition is met
- ok: false with reason if the condition is not met`,
])
```

### 15.3.5 Hook 聚合

多个 Hook 的结果会被聚合成一个单一的结果：

```typescript
export type AggregatedHookResult = {
  message?: Message
  blockingErrors?: HookBlockingError[]
  preventContinuation?: boolean
  stopReason?: string
  hookPermissionDecisionReason?: string
  permissionBehavior?: PermissionResult['behavior']
  additionalContexts?: string[]
  initialUserMessage?: string
  updatedInput?: Record<string, unknown>
  updatedMCPToolOutput?: unknown
  permissionRequestResult?: PermissionRequestResult
  retry?: boolean
}
```

聚合策略：
- 阻塞错误会阻止操作
- 非阻塞错误会显示给用户但不阻止操作
- 多个 Hook 可以提供额外的上下文
- 权限决策会按优先级聚合

## 15.4 权限拦截

Hook 系统与权限系统紧密集成，允许 Hook 影响权限决策。

### 15.4.1 PermissionRequest Hook

`PermissionRequest` Hook 可以在权限对话框显示时自动决定权限：

```typescript
PermissionRequest: {
  summary: 'When a permission dialog is displayed',
  description: 'Output JSON with hookSpecificOutput containing decision to allow or deny.',
}
```

Hook 可以返回以下决策：

```typescript
decision: z.union([
  z.object({
    behavior: z.literal('allow'),
    updatedInput: z.record(z.string(), z.unknown()).optional(),
    updatedPermissions: z.array(permissionUpdateSchema()).optional(),
  }),
  z.object({
    behavior: z.literal('deny'),
    message: z.string().optional(),
    interrupt: z.boolean().optional(),
  }),
])
```

### 15.4.2 PreToolUse Hook 拦截

`PreToolUse` Hook 可以通过返回退出码 2 来阻止工具调用：

```typescript
PreToolUse: {
  description: 'Exit code 2 - show stderr to model and block tool call\nOther exit codes - show stderr to user only but continue with tool call',
}
```

### 15.4.3 权限结果聚合

多个 Hook 的权限决策会按以下优先级聚合：

1. **Deny**（任何 deny 都会阻止）
2. **Allow**（所有 allow 都会允许）
3. **Ask**（默认行为）

```typescript
export type PermissionRequestResult =
  | {
      behavior: 'allow'
      updatedInput?: Record<string, unknown>
      updatedPermissions?: PermissionUpdate[]
    }
  | {
      behavior: 'deny'
      message?: string
      interrupt?: boolean
    }
```

## 15.5 自定义 Hook 开发

用户可以开发自己的 Hook 来扩展 Claude Code 的功能。

### 15.5.1 命令 Hook

最简单的 Hook 类型是命令 Hook，执行外部脚本：

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "command": "echo 'Running Bash tool'"
      }
    ]
  }
}
```

环境变量：
- `CLAUDE_HOOK_EVENT`：Hook 事件名称
- `CLAUDE_HOOK_INPUT`：JSON 格式的输入数据
- `CLAUDE_TOOL_NAME`：工具名称（仅 PreToolUse/PostToolUse）

### 15.5.2 提示 Hook

提示 Hook 使用 LLM 评估条件：

```json
{
  "hooks": {
    "Stop": [
      {
        "prompt": "Check if the response includes a summary of changes made"
      }
    ]
  }
}
```

提示模板支持 `$ARGUMENTS` 占位符，会被实际的 JSON 输入替换。

### 15.5.3 代理 Hook

代理 Hook 可以使用工具验证复杂条件：

```json
{
  "hooks": {
    "Stop": [
      {
        "type": "agent",
        "prompt": "Verify that all tests pass after the changes",
        "model": "claude-3-5-sonnet-20241022"
      }
    ]
  }
}
```

### 15.5.4 Hook 输出

Hook 可以通过 stdout 输出 JSON 来控制行为：

```json
{
  "continue": false,
  "reason": "Tests did not pass",
  "suppressOutput": true
}
```

### 15.5.5 匹配器（Matcher）

某些 Hook 支持匹配器，只在特定条件下触发：

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Read|Write",
        "command": "check-file-permissions.sh"
      }
    ]
  }
}
```

支持的匹配器：
- **工具名称**：PreToolUse、PostToolUse、PostToolUseFailure、PermissionDenied、PermissionRequest
- **通知类型**：Notification
- **会话源**：SessionStart
- **触发类型**：Setup、PreCompact、PostCompact
- **结束原因**：SessionEnd
- **代理类型**：SubagentStart、SubagentStop、TeammateIdle
- **配置源**：ConfigChange
- **加载原因**：InstructionsLoaded

## 15.6 小结

Claude Code 的 Hook 系统是一个强大而灵活的扩展机制，它体现了几个重要的设计原则：

### 15.6.1 关注点分离

Hook 系统将扩展逻辑从核心代码中分离出来：
- 核心代码负责执行主要逻辑
- Hook 负责扩展和自定义
- 配置文件管理 Hook 的注册和配置

### 15.6.2 类型安全

Hook 系统充分利用 TypeScript 的类型系统：
- 每个 Hook 事件都有明确的输入类型
- Hook 输出使用 Zod schema 验证
- 编译时检查确保 Hook 的正确使用

### 15.6.3 渐进式增强

Hook 系统支持从简单到复杂的各种用法：
1. **简单命令**：执行外部脚本
2. **条件评估**：使用 LLM 评估简单条件
3. **复杂验证**：使用 Agent Hook 进行多步骤验证

### 15.6.4 观察与控制

Hook 系统同时支持观察和控制：
- **观察**：PostToolUse、Notification 等事件只用于观察
- **控制**：PreToolUse、PermissionRequest 等事件可以控制行为

### 15.6.5 异步支持

异步 Hook 允许长时间运行的操作不阻塞主流程：
- 后台进程执行命令
- 定期检查输出
- 完成后发送响应

通过 Hook 系统，用户和插件开发者可以：
- 自动化常见任务
- 实施项目特定的规则
- 集成外部工具和服务
- 扩展 Claude Code 的功能而不修改核心代码

这使得 Claude Code 不仅是一个 AI 编程助手，更是一个可扩展的平台，能够适应各种不同的工作流程和需求。

---

> **延伸思考**
>
> 1. Hook 系统如何平衡灵活性和性能？
> 2. 为什么需要三种不同类型的 Hook（命令、提示、代理）？
> 3. Hook 的异步执行模型如何确保可靠性？
> 4. 如何设计 Hook 来避免无限递归（例如 UserPromptSubmit Hook 触发新的提交）？
> 5. Hook 系统如何与插件系统协同工作？
