# 第22章 设计模式映射 —— Claude Code中的经典模式

> "Design patterns are not about design per se, they are about the communication of design." —— GoF

设计模式是软件工程中反复出现问题的成熟解决方案。Claude Code作为一款复杂的AI应用，在其架构中自然地运用了多种经典设计模式。本章将从源码角度剖析这些模式的应用，展示理论模式如何在实战中落地。

## 22.1 命令模式(Command Pattern)：Tool与Command

命令模式将请求封装为对象，从而允许用不同的请求对客户进行参数化、对请求排队或记录请求日志。在Claude Code中，Tool系统是命令模式的典型实现。

### 22.1.1 Tool接口设计

每个Tool都是一个独立的命令对象：

```typescript
export type Tool<
  Input extends AnyObject = AnyObject,
  Output = unknown,
  P extends ToolProgressData = ToolProgressData,
> = {
  name: string
  aliases?: string[]
  inputSchema: Input
  outputSchema?: z.ZodType<unknown>

  // 命令执行
  call(
    args: z.infer<Input>,
    context: ToolUseContext,
    canUseTool: CanUseToolFn,
    parentMessage: AssistantMessage,
    onProgress?: ToolCallProgress<P>,
  ): Promise<ToolResult<Output>>

  // 命令描述
  description(
    input: z.infer<Input>,
    options: {
      isNonInteractiveSession: boolean
      toolPermissionContext: ToolPermissionContext
      tools: Tools
    },
  ): Promise<string>

  // 安全检查
  checkPermissions(
    input: z.infer<Input>,
    context: ToolUseContext,
  ): Promise<PermissionResult>

  // 元方法
  isConcurrencySafe(input: z.infer<Input>): boolean
  isReadOnly(input: z.infer<Input>): boolean
  isDestructive?(input: z.infer<Input>): boolean
  // ...
}
```

Tool接口的命令模式特征：
- **封装请求**：每个Tool封装了特定操作的完整逻辑
- **参数化**：通过`inputSchema`定义可配置的参数
- **可撤销性**：`isDestructive`标识命令是否可逆
- **执行者分离**：调用者（LLM）与执行者（Tool实现）解耦

### 22.1.2 buildTool工厂方法

`buildTool`函数实现了命令对象的统一构建：

```typescript
const TOOL_DEFAULTS = {
  isEnabled: () => true,
  isConcurrencySafe: (_input?: unknown) => false,
  isReadOnly: (_input?: unknown) => false,
  isDestructive: (_input?: unknown) => false,
  checkPermissions: (
    input: { [key: string]: unknown },
    _ctx?: ToolUseContext,
  ): Promise<PermissionResult> =>
    Promise.resolve({ behavior: 'allow', updatedInput: input }),
  toAutoClassifierInput: (_input?: unknown) => '',
  userFacingName: (_input?: unknown) => '',
}

export function buildTool<D extends AnyToolDef>(def: D): BuiltTool<D> {
  return {
    ...TOOL_DEFAULTS,
    userFacingName: () => def.name,
    ...def,
  } as BuiltTool<D>
}
```

这种工厂模式的运用：
- **默认行为**：为所有命令提供安全默认值
- **选择性覆盖**：实现者只需覆盖需要的方法
- **类型推断**：TypeScript确保类型安全

### 22.1.3 命令队列与执行

工具调用通过队列管理，支持并发和串行执行：

```typescript
export interface ToolUseContext {
  messages: Message[]
  setInProgressToolUseIDs: (f: (prev: Set<string>) => Set<string>) => void
  // ...
}

// 并发安全的工具可以并行执行
const isConcurrencySafe = tool.isConcurrencySafe(input)
if (isConcurrencySafe) {
  // 加入并发队列
} else {
  // 等待前面的工具完成
}
```

命令队列的价值：
- **并发控制**：安全命令并行执行提高效率
- **依赖管理**：不安全命令串行执行保证正确性
- **进度追踪**：实时跟踪所有正在执行的命令

## 22.2 策略模式(Strategy Pattern)：权限与模式切换

策略模式定义了一系列算法，将每个算法封装起来，并使它们可以互换。Claude Code的权限系统是策略模式的经典应用。

### 22.2.1 PermissionMode枚举

权限模式定义了不同的安全策略：

权限模式定义在 `src/types/permissions.ts` 中，分为外部模式和内部模式两层：

```typescript
// src/types/permissions.ts:16-22
// 外部模式：用户可直接配置的权限策略
export const EXTERNAL_PERMISSION_MODES = [
  'acceptEdits',       // 接受编辑：自动接受文件修改
  'bypassPermissions', // 绕过权限：所有工具调用直接执行
  'default',           // 默认：询问未知操作
  'dontAsk',           // 不询问：基于已有规则自动决策
  'plan',              // 计划模式：只读浏览不执行修改
] as const

// 内部模式：包含外部模式 + 系统内部使用的额外模式
export type InternalPermissionMode =
  | ExternalPermissionMode
  | 'auto'   // 自动模式：通过分类器自动决策（Feature Flag门控）
  | 'bubble' // 冒泡模式：子Agent将权限请求冒泡到父级
```

每种模式代表一种不同的权限验证策略，运行时可以动态切换。

### 22.2.2 策略实现：权限检查管道

权限检查不是简单的 switch-case，而是一个多步骤管道，实现在 `src/utils/permissions/` 中：

```typescript
// 简化示意：权限检查的核心流程
// 实际实现在 src/utils/permissions/permissions.ts
async function hasPermissionsToUseToolInner(tool, input, context) {
  // 步骤1：检查 alwaysDeny 规则（最高优先级）
  //   按来源优先级检查 deny 规则

  // 步骤2：检查 alwaysAsk 规则
  //   强制要求用户确认

  // 步骤3：调用 tool.checkPermissions()
  //   工具自身定义的权限逻辑

  // 步骤4：检查 bypass 模式
  //   bypassPermissions 模式下直接放行

  // 步骤5：检查 alwaysAllow 规则
  //   按来源优先级检查 allow 规则

  // 步骤6：passthrough → 转为 ask 用户
  //   未匹配任何规则时询问用户
}
```

对于 `plan` 模式，有专门的 `checkPlanModePermissions` 逻辑：只读工具（如 FileRead、Grep、Glob）被放行，编辑类工具被拒绝。这种分离确保了计划模式不会意外执行破坏性操作。

策略模式的优点在此体现：
- **管道式设计**：权限检查是一个有序的多步骤管道，而非单一 switch
- **规则优先级**：deny 规则优先于 allow 规则，确保安全
- **工具自定义**：每个工具可以通过 `checkPermissions` 添加专属检查
- **运行时切换**：用户可以随时改变权限策略

### 22.2.3 prePlanMode 状态保存

Claude Code 支持 `plan` 模式的临时进入和自动恢复。`ToolPermissionContext` 中的 `prePlanMode` 字段保存了进入计划模式前的权限状态：

```typescript
// src/Tool.ts:137 - ToolPermissionContext 中的模式保存
export type ToolPermissionContext = DeepImmutable<{
  mode: PermissionMode
  // ...
  prePlanMode?: PermissionMode  // 保存进入计划模式前的状态
}>
```

进入计划模式时保存当前模式，退出时恢复——这种状态快照机制确保了临时性切换不影响用户的全局设置。

## 22.3 观察者模式(Observer Pattern)：React状态与事件

观察者模式定义对象间的一对多依赖关系，当一个对象状态改变时，所有依赖者都会收到通知。Claude Code使用React的状态管理实现了观察者模式。

### 22.3.1 AppState全局状态

`AppState`是整个应用的状态中心：

```typescript
export type AppState = {
  // 消息列表
  messages: Message[]

  // 权限状态
  toolPermissionContext: ToolPermissionContext

  // 进度状态
  inProgressToolUseIDs: Set<string>

  // UI状态
  responseLength: number
  streamMode: SpinnerMode

  // ...更多状态
}

// 状态更新函数
export function setAppState(f: (prev: AppState) => AppState): void {
  const newState = f(store.getState())
  store.setState(newState)
}
```

观察者模式的特征：
- **Subject**：`AppState`是被观察的主题
- **Observer**：React组件订阅状态变化
- **Notification**：状态更新自动触发组件重渲染

### 22.3.2 状态订阅与通知

React组件通过`useAppState`订阅状态：

```typescript
export function useAppState<T>(
  selector: (state: AppState) => T,
): T {
  const [, forceUpdate] = React.useReducer(x => x + 1, 0)

  React.useEffect(() => {
    // 订阅状态变化
    const unsubscribe = store.subscribe((newState) => {
      const selected = selector(newState)
      // 仅当相关状态变化时才重渲染
      if (selected !== selector(store.getState())) {
        forceUpdate()
      }
    })

    return unsubscribe
  }, [selector])

  return selector(store.getState())
}
```

这种实现的特点：
- **精确订阅**：组件只订阅需要的状态片段
- **按需更新**：只有相关状态变化时才重渲染
- **自动清理**：组件卸载时自动取消订阅

### 22.3.3 事件通知系统

除了状态变化，Claude Code还有专门的事件通知系统：

```typescript
export type Notification = {
  id: string
  type: 'info' | 'warning' | 'error' | 'success'
  title?: string
  message: string
  duration?: number
  actions?: NotificationAction[]
}

// 通知队列
const [notifications, setNotifications] = React.useState<Notification[]>([])

// 发送通知
function addNotification(notif: Notification): void {
  setNotifications(prev => [...prev, notif])
}
```

事件通知的用途：
- **用户反馈**：显示操作结果（成功/失败）
- **系统警告**：提醒用户潜在问题
- **进度更新**：长时间操作的进度反馈

## 22.4 中间件模式(Middleware)：Hook系统

中间件模式允许在请求/响应链中插入自定义处理逻辑。Claude Code的Hook系统是这一模式的典型应用。

### 22.4.1 Hook的真实架构

与许多框架不同，Claude Code 的 Hook 系统**不是基于 JavaScript 回调的面向对象接口**，而是基于**外部命令执行**的配置驱动系统。用户在 `settings.json` 中声明 Hook，系统在特定事件发生时执行外部 shell 命令。

Hook 的配置格式定义在 `src/types/hooks.ts` 中：

```typescript
// src/types/hooks.ts - Hook 配置格式（简化示意）
type HookConfig = {
  type: 'command'           // 命令类型：执行外部 shell 命令
  matcher?: string          // 可选：工具名匹配模式（如 "Bash|Edit"）
  hooks: string[]           // 要执行的命令列表
  timeout?: number          // 超时时间（秒）
}

// 支持的事件类型（通过 HOOK_EVENTS 数组定义，共 27 种）
type HookEvent =
  | 'PreToolUse'            // 工具执行前
  | 'PostToolUse'           // 工具执行后
  | 'UserPromptSubmit'      // 用户提交提示后
  | 'SessionStart'          // 会话开始时
  | 'SessionEnd'            // 会话结束时
  | 'PreCompact'            // 上下文压缩前
  | 'PostCompact'           // 上下文压缩后
  // ...更多事件
```

Hook 执行的结果通过 JSON Schema 定义（`syncHookResponseSchema`），支持 `approve`、`deny`、`block` 等决策：

```typescript
// src/types/hooks.ts - Hook 响应的 Zod Schema（简化示意）
const syncHookResponseSchema = z.object({
  decision: z.enum(['approve', 'deny', 'block']).optional(),
  reason: z.string().optional(),
  // ...更多字段
})
```

### 22.4.2 Hook执行机制

Hook 在运行时通过 `execCommandHook` 函数执行：

```typescript
// 简化示意：命令 Hook 的执行流程
// 实际实现在 src/utils/hooks/ 目录下
async function execCommandHook(config: HookConfig, event: HookEvent, payload: unknown) {
  // 1. 通过环境变量传递事件数据
  //    CLAUDE_HOOK_EVENT = event name
  //    CLAUDE_HOOK_INPUT = JSON.stringify(payload)

  // 2. 执行 shell 命令
  //    退出码 0 = approve
  //    退出码 2 = deny/block
  //    其他退出码 = 错误

  // 3. 解析 stdout JSON（如果有）
  //    使用 syncHookResponseSchema 验证
}
```

这种设计的特点：
- **语言无关**：Hook 可以用任何语言编写（Python、Bash、Node.js 等）
- **进程隔离**：Hook 在独立进程中执行，不会崩溃主进程
- **退出码语义**：标准的 Unix 退出码约定，简单直观

### 22.4.3 实际应用案例

用户在 `settings.json` 中配置 Hook：

```json
{
  "hooks": {
    "PreToolUse": [{
      "matcher": "Bash|Edit|Write",
      "hooks": ["python scripts/check_sensitive_files.py"],
      "timeout": 10
    }],
    "PostToolUse": [{
      "matcher": "Bash",
      "hooks": ["bash scripts/log_commands.sh"]
    }]
  }
}
```

Hook 展示了中间件模式的价值：
- **横切关注点**：安全、日志等跨工具逻辑集中管理
- **可组合性**：多个 Hook 可以按顺序执行
- **无侵入性**：工具实现不需要知道 Hook 的存在
- **用户可扩展**：不需要修改源码即可添加自定义行为

## 22.5 工厂模式(Factory)：工具的动态创建

工厂模式定义了创建对象的接口，但由子类决定实例化哪个类。Claude Code在多个层面使用了工厂模式。

### 22.5.1 MCP工具工厂

MCP（Model Context Protocol）工具在运行时动态创建：

```typescript
export async function createMCPTools(
  serverConnections: MCPServerConnection[],
): Promise<Tools> {
  const tools: Tool[] = []

  for (const connection of serverConnections) {
    for (const mcpTool of connection.tools) {
      const tool = await buildMCPTool({
        name: mcpTool.name,
        description: mcpTool.description,
        inputSchema: mcpTool.inputSchema,
        serverName: connection.serverName,
        client: connection.client,
      })

      tools.push(tool)
    }
  }

  return tools
}
```

MCP工具工厂的特点：
- **动态发现**：工具列表来自服务器配置
- **统一接口**：MCP工具与内置工具接口一致
- **延迟加载**：只在需要时创建工具实例

### 22.5.2 Agent子代理工厂

子代理根据配置动态创建：

```typescript
export async function createSubagent(
  agentDefinition: AgentDefinition,
  parentContext: ToolUseContext,
): Promise<AgentResult> {
  const agentContext = createSubagentContext(parentContext, {
    agentId: generateAgentId(),
    agentType: agentDefinition.type,
    // ...继承父上下文
  })

  // 根据类型选择Agent实现
  switch (agentDefinition.type) {
    case 'custom':
      return await runCustomAgent(agentDefinition, agentContext)
    case 'builtin':
      return await runBuiltinAgent(agentDefinition, agentContext)
    default:
      throw new Error(`Unknown agent type: ${agentDefinition.type}`)
  }
}
```

Agent工厂的设计考量：
- **类型隔离**：不同类型Agent有独立的实现
- **上下文继承**：子代理继承父代理的部分配置
- **错误隔离**：子代理错误不影响父代理

### 22.5.3 消息构建工厂

API请求消息通过工厂函数构建：

```typescript
export async function queryModel(
  messages: Message[],
  systemPrompt: SystemPrompt,
  thinkingConfig: ThinkingConfig,
  tools: Tools,
  signal: AbortSignal,
  options: Options,
): AsyncGenerator<StreamEvent | AssistantMessage | SystemAPIErrorMessage, void> {
  // 1. 构建工具Schema
  const toolSchemas = await Promise.all(
    filteredTools.map(tool =>
      toolToAPISchema(tool, {
        getToolPermissionContext: options.getToolPermissionContext,
        tools,
        agents: options.agents,
        model: options.model,
      }),
    ),
  )

  // 2. 构建系统提示
  systemPrompt = asSystemPrompt([
    getAttributionHeader(fingerprint),
    getCLISyspromptPrefix({ ... }),
    ...systemPrompt,
  ])

  // 3. 构建缓存控制
  const system = buildSystemPromptBlocks(systemPrompt, enablePromptCaching, {
    querySource: options.querySource,
  })

  // 4. 构建请求参数
  const params = {
    model: normalizeModelStringForAPI(options.model),
    messages: addCacheBreakpoints(messagesForAPI, ...),
    system,
    tools: allTools,
    betas: betasParams,
    metadata: getAPIMetadata(),
    max_tokens: maxOutputTokens,
    // ...
  }
}
```

消息工厂的职责：
- **参数组装**：从多个来源收集参数
- **格式转换**：内部格式→API格式
- **条件注入**：根据环境添加可选参数

## 22.6 代理模式(Proxy)：MCP工具的透明代理

代理模式为其他对象提供一种代理，以控制对这个对象的访问。MCP工具的实现是代理模式的典型案例。

### 22.6.1 MCP工具代理结构

MCP 工具通过 spread 模式动态创建，在 `src/services/mcp/client.ts` 的 `fetchToolsForClient` 函数中：

```typescript
// src/services/mcp/client.ts - MCP 工具创建（简化示意）
// 实际实现使用 memoizeWithLRU 缓存 fetchToolsForClient 的结果
async function fetchToolsForClient(client: MCPServerConnection) {
  const mcpTools = await client.listTools()

  return mcpTools.map(mcpTool => ({
    ...MCPTool,                                    // spread 基础 MCP 工具实现
    name: buildMcpToolName(client.name, mcpTool.name),  // mcp__serverName__toolName
    mcpInfo: {
      serverName: client.name,
      toolName: mcpTool.name,
    },
    // ... 其他工具特定配置
  }))
}
```

`MCPTool` 基础对象定义了代理行为：将 Claude Code 的工具调用协议转换为 MCP 协议的 `callTool` 请求，再将 MCP 响应转换回内部的 `ToolResult` 格式。

MCP 工具名通过 `buildMcpToolName` 构建（`src/services/mcp/mcpStringUtils.ts`），格式为 `mcp__{serverName}__{toolName}`，其中 serverName 和 toolName 都经过 `normalizeNameForMCP` 标准化处理。

MCP 代理的关键特征：
- **透明访问**：上层代码使用与内置工具相同的 Tool 接口
- **协议转换**：内部格式 ↔ MCP 协议格式
- **spread 创建**：基于基础 MCPTool 对象动态覆盖属性
      return true
    },

    isReadOnly() {
      // 从MCP工具元数据推断
      return !config.description.toLowerCase().includes('modify')
    },

    isDestructive() {
      return config.description.toLowerCase().includes('delete')
    },
  }
}
```

MCP代理的关键特征：
- **透明访问**：上层代码不知道是本地还是远程工具
- **协议转换**：内部格式↔MCP协议格式
- **错误处理**：将网络错误转换为工具错误

### 22.6.2 工具发现的缓存机制

MCP 工具发现（从服务器获取工具列表）使用 `memoizeWithLRU` 进行缓存，避免重复的网络调用：

```typescript
// src/services/mcp/client.ts - 使用 memoizeWithLRU 缓存
// 实际缓存大小由 MCP_FETCH_CACHE_SIZE 常量定义（默认 20）
const cachedFetchTools = memoizeWithLRU(
  fetchToolsForClient,
  (client) => client.name,
  MCP_FETCH_CACHE_SIZE  // 最大缓存 20 个服务器的工具列表
)
```

与虚构的独立 `LRUCache` + `getOrCreateMCPTool` 模式不同，实际实现使用通用的 `memoizeWithLRU` 函数包装，将整个 `fetchToolsForClient` 函数变为带缓存的版本。这是**函数级缓存**而非**对象级缓存**——缓存键是服务器名称，值是完整的工具列表。

### 22.6.3 Elicitation 机制

MCP 代理的一个独特能力是 Elicitation——MCP 服务器可以向用户请求输入（`src/services/mcp/elicitationHandler.ts`）。当工具调用返回错误码 `-32042` 时，系统会通过 `handleElicitation` 展示交互界面，让用户输入所需信息后重试。这种设计使 MCP 工具能够在执行过程中动态获取用户输入，突破了简单的请求-响应模式。

## 22.7 小结

Claude Code的代码库中蕴含着丰富的设计模式应用。本章总结的六种模式只是冰山一角，但它们展示了经典设计思想在现代AI应用中的价值：

1. **命令模式**：Tool系统将操作封装为可重用的命令对象
2. **策略模式**：PermissionMode实现可插拔的权限策略
3. **观察者模式**：React状态管理实现响应式UI更新
4. **中间件模式**：Hook系统提供灵活的扩展点
5. **工厂模式**：动态创建工具、代理和消息对象
6. **代理模式**：MCP工具透明桥接远程服务

这些模式并非孤立存在，而是相互协作构成完整的架构。例如：
- 工厂创建的命令对象可以通过中间件链执行
- 观察者模式通知策略模式的状态变化
- 代理对象本身可能由工厂创建

> "Patterns are not solutions, they are hints to solutions." —— 设计模式智慧

设计模式的价值不在于背诵定义，而在于识别何时以及如何应用它们。Claude Code的源码展示了这一点——模式的使用总是服务于实际的工程需求：解耦、扩展、复用、维护。

在AI应用的快速迭代中，设计模式提供了稳定的架构骨架。它们不是限制创新的桎梏，而是支撑变化的框架。正如GoF在《设计模式》开篇所说："找到变化的东西，把它封装起来"——这或许是对Claude Code架构设计的最好总结。

至此，本书的源码分析章节告一段落。从最初的启动流程到最终的API通信，从底层的工具系统到上层的用户界面，我们全面剖析了Claude Code的架构设计。希望这些分析能够帮助读者理解现代AI应用的构建之道，在自己的项目中做出更好的设计决策。
