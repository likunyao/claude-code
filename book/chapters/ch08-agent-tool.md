# 第8章：Agent Tool —— 嵌套智能体的递归之美

> "递归是计算的本质，而自我参照是智能的萌芽。" —— Douglas Hofstadter

在所有 Claude Code 的工具中，AgentTool 可能是最具哲学深度的设计。它赋予了 AI **创建另一个 AI** 的能力——一个可以嵌套、可以递归、可以组成团队的能力。这不是简单的函数调用，而是一种**智能体的自相似结构**：每个 Agent 都拥有完整的工具集、独立的上下文和自主的决策能力。

## 8.1 AgentTool：子智能体的创建与生命周期

### 8.1.1 输入模型

AgentTool 的输入模式分为两层：基础模式和完整模式。基础模式（`baseInputSchema`）包含核心字段：

```typescript
const baseInputSchema = lazySchema(() => z.object({
  description: z.string().describe('A short (3-5 word) description of the task'),
  prompt: z.string().describe('The task for the agent to perform'),
  subagent_type: z.string().optional().describe('The type of specialized agent to use'),
  model: z.enum(['sonnet', 'opus', 'haiku']).optional().describe('Optional model override'),
  run_in_background: z.boolean().optional().describe('Set to true to run this agent in the background')
}));
```

完整模式（`fullInputSchema`）在基础模式上合并了多智能体参数和隔离选项：

```typescript
const fullInputSchema = lazySchema(() => {
  const multiAgentInputSchema = z.object({
    name: z.string().optional()
      .describe('Name for the spawned agent. Makes it addressable via SendMessage({to: name})'),
    team_name: z.string().optional()
      .describe('Team name for spawning. Uses current team context if omitted.'),
    mode: permissionModeSchema().optional()
      .describe('Permission mode for spawned teammate (e.g., "plan")')
  });
  return baseInputSchema().merge(multiAgentInputSchema).extend({
    isolation: z.enum(['worktree', 'remote']).optional()
      .describe('Isolation mode. "worktree" creates a temporary git worktree. ' +
                '"remote" launches the agent in a remote CCR environment.'),
    cwd: z.string().optional()
      .describe('Absolute path to run the agent in. Mutually exclusive with isolation: "worktree".')
  });
});
```

每个字段都承载着设计决策：

- **description（3-5词）**：强制要求简短描述。这不是给 AI 看的，而是给用户和系统看的——它是任务追踪和 UI 展示的标签
- **prompt**：子智能体的完整任务指令，相当于一次全新的对话启动
- **subagent_type**：选择专业化的 Agent 类型（下一节详述）；省略时在 Fork 模式启用时触发 Fork Agent
- **model**：允许为子任务选择不同能力的模型——轻量任务用 Haiku，复杂推理用 Opus
- **run_in_background**：支持异步执行，子智能体可以在后台独立工作
- **name**：为子智能体命名，使其在运行期间可通过 `SendMessage({to: name})` 寻址——这是 Swarm/Team 模式的入口之一
- **team_name**：指定团队名称，配合 `name` 触发队友生成（spawnTeammate），在 tmux 中创建新的独立进程
- **mode**：控制队友的权限模式（如 `"plan"` 要求计划审批）
- **isolation**：隔离模式。`"worktree"` 创建临时 git worktree 使 Agent 在独立副本上工作；`"remote"` 在远程 CCR 环境中启动（仅限内部构建）
- **cwd**：覆盖 Agent 的工作目录，与 `isolation: "worktree"` 互斥

输入模式会根据功能开关动态裁剪。例如，当 Fork 子智能体实验启用时，`run_in_background` 字段会从 schema 中移除（因为所有子智能体都强制异步），而 `cwd` 字段则受 KAIROS 功能开关控制：

```typescript
export const inputSchema = lazySchema(() => {
  const schema = feature('KAIROS') ? fullInputSchema() : fullInputSchema().omit({ cwd: true });
  return isBackgroundTasksDisabled || isForkSubagentEnabled()
    ? schema.omit({ run_in_background: true })
    : schema;
});
```

### 8.1.2 执行流程

AgentTool 的 `call()` 方法实现了子智能体的完整执行路径，包含多条分支路由：

```typescript
async call({
  prompt, subagent_type, description, model, run_in_background,
  name, team_name, mode, isolation, cwd
}: AgentToolInput, toolUseContext, canUseTool, assistantMessage, onProgress) {
  // 阶段1：权限与参数验证
  //   - 检查 Agent Teams 是否可用（team_name 存在时）
  //   - 防止嵌套队友生成（teammates 不能生成 teammates）
  //   - 进程内队友不能生成后台 Agent

  // 阶段2：路由分发 -- 四条路径之一
  if (teamName && name) {
    // 路径A：Swarm/Team 队友生成
    //   -> spawnTeammate() 在 tmux 中创建新进程
    //   -> 返回 teammate_spawned 状态
  }

  if (effectiveIsolation === 'remote') {
    // 路径B：远程 Agent
    //   -> teleportToRemote() 在 CCR 环境中启动
    //   -> 返回 remote_launched 状态
  }

  if (isForkPath) {
    // 路径C：Fork Agent（省略 subagent_type 时）
    //   -> 共享父级对话上下文和系统提示
    //   -> buildForkedMessages() 构造消息
  } else {
    // 路径D：标准子智能体（指定 subagent_type）
    //   -> 查找匹配的 AgentDefinition
    //   -> 验证 MCP 服务器依赖
  }

  // 阶段3：隔离设置
  //   - worktree 隔离：createAgentWorktree() 创建独立 git 工作树
  //   - cwd 覆盖：runWithCwdOverride() 设置工作目录

  // 阶段4：异步/同步决策
  //   shouldRunAsync = (run_in_background || background || isCoordinator
  //                     || forceAsync || assistantForceAsync || proactive)
  //   - 同步：在父级 turn 中阻塞执行
  //   - 异步：注册为后台任务，通过通知回调结果

  // 阶段5：调用 runAgent() 执行查询循环
  //   - 构造系统提示、上下文、工具池
  //   - 流式处理消息
  //   - 收集结果并返回 ToolResult
}
```

四条路由路径的设计体现了**策略模式**：相同的 AgentTool 接口根据参数组合选择不同的执行策略，而调用者无需关心内部路由细节。

### 8.1.3 生命周期管理

子智能体的生命周期遵循明确的状态机：

```
创建 -> 初始化上下文 -> 执行查询循环 -> 收集结果 -> 销毁
  ^                                              |
  +---------- 可中断（AbortController）-----------+
```

关键设计：
- **独立性**：每个子智能体拥有独立的消息历史和上下文窗口
- **可中断性**：通过 AbortController 支持从父级取消；异步 Agent 使用独立的 AbortController（不受用户 ESC 影响）
- **资源清理**：完成后自动清理上下文、MCP 连接、session hooks、文件缓存和 todo 列表
- **结果传递**：子智能体的输出通过标准 ToolResult 机制返回给父级

## 8.2 Agent 定义：loadAgentsDir 的自定义 Agent 系统

Claude Code 不仅内置了多种标准 Agent 类型，还提供了多层的用户自定义 Agent 机制——通过 `.claude/agents/` 目录的 Markdown 文件、JSON 格式的设置项、以及插件系统。

### 8.2.1 Agent 定义格式

每个自定义 Agent 由一个 Markdown 文件定义，frontmatter 中使用 YAML 格式声明元数据，正文作为系统提示：

```yaml
---
name: my-custom-agent
description: 专门用于代码审查的 Agent
model: sonnet
tools:
  - Read
  - Grep
  - Glob
  - Bash
disallowedTools:
  - Write
  - Edit
effort: high
maxTurns: 50
permissionMode: dontAsk
memory: project
mcpServers:
  - slack
hooks:
  SubagentStart:
    - command: echo "agent starting"
background: false
isolation: worktree
skills:
  - commit
initialPrompt: /review-guidelines
color: blue
---

你是一个专门的代码审查 Agent。你的任务是...
```

系统在解析 Markdown frontmatter 后，会通过 Zod Schema（`AgentJsonSchema`）进行严格的类型验证。`parseAgentFromJson()` 函数使用 `AgentJsonSchema` 对 JSON 数据进行校验：

```typescript
const AgentJsonSchema = lazySchema(() =>
  z.object({
    description: z.string().min(1, 'Description cannot be empty'),
    tools: z.array(z.string()).optional(),
    disallowedTools: z.array(z.string()).optional(),
    prompt: z.string().min(1, 'Prompt cannot be empty'),
    model: z.string().trim().min(1).transform(m =>
      (m.toLowerCase() === 'inherit' ? 'inherit' : m)
    ).optional(),
    effort: z.union([z.enum(EFFORT_LEVELS), z.number().int()]).optional(),
    permissionMode: z.enum(PERMISSION_MODES).optional(),
    mcpServers: z.array(AgentMcpServerSpecSchema()).optional(),
    hooks: HooksSchema().optional(),
    maxTurns: z.number().int().positive().optional(),
    skills: z.array(z.string()).optional(),
    initialPrompt: z.string().optional(),
    memory: z.enum(['user', 'project', 'local']).optional(),
    background: z.boolean().optional(),
    isolation: z.enum(['worktree', 'remote']).optional(),
  })
)
```

`BaseAgentDefinition` 类型定义了所有 Agent 共享的字段：

```typescript
type BaseAgentDefinition = {
  agentType: string           // 唯一标识符
  whenToUse: string           // 描述何时使用此 Agent
  tools?: string[]            // 允许的工具白名单
  disallowedTools?: string[]  // 禁用的工具黑名单
  skills?: string[]           // 预加载的 Skill 名称
  mcpServers?: AgentMcpServerSpec[]  // Agent 专属 MCP 服务器
  hooks?: HooksSettings       // Agent 生命周期钩子
  color?: AgentColorName      // UI 显示颜色
  model?: string              // 模型选择
  effort?: EffortValue        // 推理努力级别
  permissionMode?: PermissionMode  // 权限模式
  maxTurns?: number           // 最大交互轮次
  memory?: AgentMemoryScope   // 持久记忆范围
  isolation?: 'worktree' | 'remote'  // 隔离模式
  background?: boolean        // 是否始终后台运行
  initialPrompt?: string      // 首轮用户消息前缀
  omitClaudeMd?: boolean      // 是否省略 CLAUDE.md 上下文
  requiredMcpServers?: string[]  // 必需的 MCP 服务器
  criticalSystemReminder_EXPERIMENTAL?: string  // 每轮重新注入的提醒
}
```

Agent 定义有三个来源变体，通过 `source` 字段区分：

- **BuiltInAgentDefinition**（`source: 'built-in'`）：内置 Agent，使用动态 `getSystemPrompt()` 函数
- **CustomAgentDefinition**（`source: 'userSettings' | 'projectSettings' | ...`）：用户/项目自定义 Agent，通过闭包存储系统提示
- **PluginAgentDefinition**（`source: 'plugin'`）：插件 Agent，附带 `plugin` 元数据

### 8.2.2 加载机制

`loadAgentsDir.ts` 中的 `getAgentDefinitionsWithOverrides()` 负责从多个来源加载和合并 Agent 定义：

```typescript
async function getAgentDefinitionsWithOverrides(cwd): Promise<AgentDefinitionsResult> {
  // 1. 扫描 .claude/agents/ 目录（通过 loadMarkdownFilesForSubdir）
  // 2. 解析每个 .md 文件的 frontmatter + 正文
  // 3. 同时加载插件 Agent（loadPluginAgents）
  // 4. 初始化 Agent 记忆快照（initializeAgentMemorySnapshots）
  // 5. 合并：内置 + 插件 + 自定义
  // 6. 按优先级去重：内置 > 插件 > 用户设置 > 项目设置 > flag设置 > 管理策略
  return { activeAgents, allAgents, failedFiles }
}
```

去重逻辑通过 `getActiveAgentsFromList()` 实现——按优先级分组后，相同 `agentType` 的 Agent 只保留最高优先级的版本：

```typescript
function getActiveAgentsFromList(allAgents: AgentDefinition[]): AgentDefinition[] {
  const agentGroups = [
    builtInAgents,    // 优先级最高
    pluginAgents,
    userAgents,
    projectAgents,
    flagAgents,
    managedAgents,    // 优先级最低
  ];
  // 后出现的同名 Agent 覆盖先出现的（Map.set 语义）
  for (const agents of agentGroups) {
    for (const agent of agents) {
      agentMap.set(agent.agentType, agent);
    }
  }
}
```

这种设计体现了**开放-封闭原则**：系统对扩展开放（用户可以添加新 Agent、安装插件 Agent），对修改封闭（核心 Agent 系统不需要改动）。同时，MCP 服务器需求过滤（`filterAgentsByMcpRequirements`）确保只有依赖满足的 Agent 才会被提供给模型。

## 8.3 子上下文隔离：createSubagentContext 的沙箱设计

### 8.3.1 上下文隔离的必要性

子智能体不能完全共享父级的上下文，原因包括：

1. **上下文窗口限制**：每个 Agent 有独立的 token 预算
2. **权限隔离**：子 Agent 可能拥有不同的权限级别
3. **状态独立性**：子 Agent 的状态变更不应直接污染父级
4. **可取消性**：取消子 Agent 不应影响父级的状态
5. **缓存一致性**：Fork Agent 需要与父级共享缓存安全参数以获得缓存命中

### 8.3.2 createSubagentContext 的真实实现

`createSubagentContext`（定义于 `src/utils/forkedAgent.ts`）接收两个参数：父级上下文和一个可选的 `overrides` 对象，用于精确控制隔离与共享的边界：

```typescript
export function createSubagentContext(
  parentContext: ToolUseContext,
  overrides?: SubagentContextOverrides,
): ToolUseContext {
  // AbortController：显式覆盖 > 共享父级 > 新建子控制器
  const abortController =
    overrides?.abortController
    ?? (overrides?.shareAbortController
      ? parentContext.abortController
      : createChildAbortController(parentContext.abortController));

  // getAppState：默认包裹一层 shouldAvoidPermissionPrompts
  const getAppState = overrides?.getAppState
    ?? (overrides?.shareAbortController
      ? parentContext.getAppState  // 共享 abortController 时可显示 UI
      : () => ({
          ...parentContext.getAppState(),
          toolPermissionContext: {
            ...state.toolPermissionContext,
            shouldAvoidPermissionPrompts: true,  // 默认禁止权限弹窗
          },
        }));

  return {
    // 可变状态：默认克隆，保持隔离
    readFileState: cloneFileStateCache(
      overrides?.readFileState ?? parentContext.readFileState
    ),
    nestedMemoryAttachmentTriggers: new Set<string>(),
    loadedNestedMemoryPaths: new Set<string>(),
    dynamicSkillDirTriggers: new Set<string>(),
    discoveredSkillNames: new Set<string>(),
    toolDecisions: undefined,
    contentReplacementState:
      overrides?.contentReplacementState
      ?? cloneContentReplacementState(parentContext.contentReplacementState),

    // AbortController
    abortController,
    // 状态访问
    getAppState,
    setAppState: overrides?.shareSetAppState ? parentContext.setAppState : () => {},
    setAppStateForTasks: parentContext.setAppStateForTasks ?? parentContext.setAppState,
    localDenialTracking: overrides?.shareSetAppState
      ? parentContext.localDenialTracking
      : createDenialTrackingState(),

    // 变更回调：默认空操作
    setInProgressToolUseIDs: () => {},
    setResponseLength: overrides?.shareSetResponseLength
      ? parentContext.setResponseLength : () => {},
    pushApiMetricsEntry: overrides?.shareSetResponseLength
      ? parentContext.pushApiMetricsEntry : undefined,
    updateFileHistoryState: () => {},
    updateAttributionState: parentContext.updateAttributionState,

    // UI 回调：子 Agent 不可控制父级 UI
    addNotification: undefined,
    setToolJSX: undefined,
    setStreamMode: undefined,
    setSDKStatus: undefined,
    openMessageSelector: undefined,

    // 可覆盖字段
    options: overrides?.options ?? parentContext.options,
    messages: overrides?.messages ?? parentContext.messages,
    agentId: overrides?.agentId ?? createAgentId(),
    agentType: overrides?.agentType,
    queryTracking: {
      chainId: randomUUID(),
      depth: (parentContext.queryTracking?.depth ?? -1) + 1,
    },
    fileReadingLimits: parentContext.fileReadingLimits,
    userModified: parentContext.userModified,
    criticalSystemReminder_EXPERIMENTAL: overrides?.criticalSystemReminder_EXPERIMENTAL,
    requireCanUseTool: overrides?.requireCanUseTool,
  };
}
```

### 8.3.3 overrides 参数的精细控制

`SubagentContextOverrides` 类型提供了丰富的控制选项，使调用者可以精确调节隔离级别：

```typescript
type SubagentContextOverrides = {
  options?: ToolUseContext['options'];     // 自定义工具集、模型等
  agentId?: AgentId;                       // 子 Agent 独立 ID
  agentType?: string;                      // Agent 类型标识
  messages?: Message[];                    // 初始消息
  readFileState?: ToolUseContext['readFileState'];  // 文件状态缓存
  abortController?: AbortController;       // 取消控制器

  // 共享选项（默认全部隔离，需要显式 opt-in）
  shareSetAppState?: boolean;              // 共享状态更新回调
  shareSetResponseLength?: boolean;        // 共享响应长度回调
  shareAbortController?: boolean;          // 共享取消控制器

  // 特殊用途
  criticalSystemReminder_EXPERIMENTAL?: string;
  requireCanUseTool?: boolean;
  contentReplacementState?: ContentReplacementState;
};
```

`runAgent.ts` 中的实际调用展示了不同场景下的 override 策略：

```typescript
const agentToolUseContext = createSubagentContext(toolUseContext, {
  options: agentOptions,           // 使用 Agent 专属的工具集和模型
  agentId,                         // 独立的 Agent ID
  agentType: agentDefinition.agentType,
  messages: initialMessages,       // 独立的消息列表
  readFileState: agentReadFileState,  // 独立的文件缓存
  abortController: agentAbortController,
  getAppState: agentGetAppState,   // 自定义的权限模式
  shareSetAppState: !isAsync,      // 同步 Agent 共享状态，异步隔离
  shareSetResponseLength: true,    // 始终共享响应长度（贡献指标）
  criticalSystemReminder_EXPERIMENTAL: agentDefinition.criticalSystemReminder_EXPERIMENTAL,
  contentReplacementState,
});
```

### 8.3.4 setAppStateForTasks 的特殊处理

上面代码中的关键设计是：子 Agent 的 `setAppState` 被设为空操作（`() => {}`），但 `setAppStateForTasks` 始终保留。这带来三个结果：

- 子 Agent **不能**直接修改全局 UI 状态
- 子 Agent **可以**注册/清理后台任务等基础设施——即使 `setAppState` 是空操作，`setAppStateForTasks` 仍然可达根 store
- 这确保了子 Agent 不会破坏父级的 UI 一致性，同时允许必要的跨级通信（如后台 bash 任务注册）
- 异步子 Agent 有独立的 `localDenialTracking`（拒绝计数器），避免父级的权限决策被污染

## 8.4 Fork Agent：共享上下文的高效子进程

### 8.4.1 Fork 的触发条件

当 `isForkSubagentEnabled()` 返回 `true`（功能开关开启且不在协调器模式/非交互模式下），省略 `subagent_type` 参数时会触发 Fork Agent 路径：

```typescript
// call() 中的路由逻辑
const effectiveType = subagent_type
  ?? (isForkSubagentEnabled() ? undefined : GENERAL_PURPOSE_AGENT.agentType);
const isForkPath = effectiveType === undefined;
```

Fork Agent 不是注册在内置 Agent 列表中的普通类型，而是一个合成的 Agent 定义：

```typescript
export const FORK_AGENT = {
  agentType: 'fork',
  tools: ['*'],              // 通配符：继承父级全部工具
  maxTurns: 200,
  model: 'inherit',          // 继承父级模型（确保缓存兼容）
  permissionMode: 'bubble',  // 权限弹窗上浮到父级终端
  source: 'built-in',
  baseDir: 'built-in',
  getSystemPrompt: () => '', // 未使用--直接传递父级已渲染的系统提示
} satisfies BuiltInAgentDefinition
```

### 8.4.2 提示缓存共享机制

Fork Agent 的核心优势在于**提示缓存共享**。其关键设计：

1. **系统提示字节精确传递**：Fork 子进程不重新生成系统提示，而是直接使用父级已渲染的 `renderedSystemPrompt` 字节。这避免了 GrowthBook 冷到热状态切换导致的缓存失效：

```typescript
if (isForkPath) {
  if (toolUseContext.renderedSystemPrompt) {
    forkParentSystemPrompt = toolUseContext.renderedSystemPrompt;
  }
  // 警告：不应重新调用 getSystemPrompt()，
  // 因为 GrowthBook 状态可能在父级 turn 期间改变
}
```

2. **消息构造最大化缓存命中**：`buildForkedMessages()` 产生的消息结构让所有 Fork 子进程共享字节完全相同的前缀。它保留父级 assistant 消息中的所有 tool_use 块，为每个生成相同的占位 tool_result，只在最后的文本块中包含不同的指令：

```typescript
function buildForkedMessages(directive, assistantMessage): MessageType[] {
  // 1. 克隆父级的完整 assistant 消息（所有 tool_use、thinking、text blocks）
  const fullAssistantMessage = { ...assistantMessage, uuid: randomUUID() };

  // 2. 为每个 tool_use block 构建相同的占位 tool_result
  const toolResultBlocks = toolUseBlocks.map(block => ({
    type: 'tool_result',
    tool_use_id: block.id,
    content: [{ type: 'text', text: 'Fork started -- processing in background' }]
  }));

  // 3. 最终结构：[assistant(所有tool_use), user(占位results... + 指令)]
  // 只有最后的指令文本不同，最大化缓存命中
  return [fullAssistantMessage, toolResultMessage];
}
```

3. **useExactTools 模式**：Fork 路径传递 `useExactTools: true`，使 `runAgent` 直接使用父级的工具池、思考配置和非交互会话标志，而不是重新解析 Agent 定义的工具列表：

```typescript
override: isForkPath ? { systemPrompt: forkParentSystemPrompt } : ...,
availableTools: isForkPath ? toolUseContext.options.tools : workerTools,
forkContextMessages: isForkPath ? toolUseContext.messages : undefined,
useExactTools: isForkPath ? true : undefined,
```

### 8.4.3 Fork 子进程的行为约束

Fork 子进程通过 `<fork-boilerplate>` XML 标签注入严格的行为约束：

```
STOP. READ THIS FIRST.
You are a forked worker process. You are NOT the main agent.

RULES (non-negotiable):
1. Do NOT spawn sub-agents; execute directly
2. Do NOT converse, ask questions, or suggest next steps
3. Do NOT editorialize or add meta-commentary
4. USE your tools directly
5. Keep your report under 500 words
6. Your response MUST begin with "Scope:"
...
```

系统通过两层保护防止递归 Fork：

```typescript
// 主检查：querySource（抗自动压缩，设置在 context.options 上）
if (toolUseContext.options.querySource === `agent:builtin:${FORK_AGENT.agentType}`)
  throw new Error('Fork is not available inside a forked worker.');

// 备用检查：消息历史扫描（当 querySource 未传递时，扫描 fork-boilerplate 标签）
if (isInForkChild(toolUseContext.messages))
  throw new Error('Fork is not available inside a forked worker.');
```

### 8.4.4 Worktree 隔离与 Fork 的组合

Fork 路径可以与 `isolation: "worktree"` 组合使用，在共享对话上下文的同时隔离文件系统操作：

```typescript
if (isForkPath && worktreeInfo) {
  promptMessages.push(createUserMessage({
    content: buildWorktreeNotice(getCwd(), worktreeInfo.worktreePath)
  }));
}
```

`buildWorktreeNotice` 注入的提示告知子进程：
- 它继承了父级在另一个目录的对话上下文
- 需要将上下文中的路径翻译为自己的工作树路径
- 在编辑前需要重新读取可能被父级修改的文件
- 它的修改独立于父级的文件

## 8.5 Swarm/Team 模式：多智能体协作

### 8.5.1 队友生成的触发

当同时提供 `name` 和 `team_name`（或存在当前团队上下文）参数时，AgentTool 进入 Swarm/Team 模式：

```typescript
if (teamName && name) {
  const result = await spawnTeammate({
    name,
    prompt,
    description,
    team_name: teamName,
    use_splitpane: true,
    plan_mode_required: spawnMode === 'plan',
    model: model ?? agentDef?.model,
    agent_type: subagent_type,
    invokingRequestId: assistantMessage?.requestId
  }, toolUseContext);

  return { data: { status: 'teammate_spawned', ...result.data } };
}
```

### 8.5.2 Team 模式的约束

Team 模式有多层防护确保拓扑正确性：

1. **团队名解析**：`resolveTeamName()` 从参数或当前上下文获取团队名
2. **嵌套防护**：队友不能再生成队友--`isTeammate() && teamName && name` 会抛出错误
3. **后台限制**：进程内队友（`isInProcessTeammate()`）不能生成后台 Agent
4. **功能开关**：`isAgentSwarmsEnabled()` 检查 Agent Teams 功能是否可用

返回的 `TeammateSpawnedOutput` 包含 tmux 会话信息（`tmux_session_name`、`tmux_pane_id`），以及队友的颜色和计划模式要求。这种设计使队友成为独立的进程，拥有自己的终端窗口和生命周期。

## 8.6 Remote Agent：云端隔离执行

### 8.6.1 远程 Agent 的触发

当 `isolation: 'remote'` 参数被设置时，Agent 通过 `teleportToRemote()` 在远程 CCR（Claude Code Runtime）环境中启动：

```typescript
if (effectiveIsolation === 'remote') {
  const eligibility = await checkRemoteAgentEligibility();
  if (!eligibility.eligible) {
    throw new Error(`Cannot launch remote agent:\n${reasons}`);
  }
  const session = await teleportToRemote({
    initialMessage: prompt,
    description,
    signal: toolUseContext.abortController.signal
  });
  const { taskId } = registerRemoteAgentTask({ session, command: prompt, ... });
  return { data: { status: 'remote_launched', taskId, sessionUrl, outputFile } };
}
```

远程 Agent 始终在后台运行，返回 `remote_launched` 状态和会话 URL。这种模式适合需要在独立计算环境中执行的任务--如需要特定硬件资源或需要与远程系统交互的场景。

## 8.7 Agent MCP 服务器：每个 Agent 的专属工具

### 8.7.1 Agent 级别的 MCP 配置

自定义 Agent 可以在其定义中声明专属的 MCP 服务器，这些服务器在 Agent 启动时连接，结束时清理：

```typescript
async function initializeAgentMcpServers(agentDefinition, parentClients) {
  if (!agentDefinition.mcpServers?.length) {
    return { clients: parentClients, tools: [], cleanup: async () => {} };
  }
  for (const spec of agentDefinition.mcpServers) {
    if (typeof spec === 'string') {
      // 按名称引用已有的 MCP 服务器
      config = getMcpConfigByName(spec);
    } else {
      // 内联定义：{ serverName: config }
      config = { ...serverConfig, scope: 'dynamic' };
    }
    const client = await connectToServer(name, config);
    agentTools.push(...await fetchToolsForClient(client));
  }
  return { clients: [...parentClients, ...agentClients], tools: agentTools, cleanup };
}
```

MCP 服务器规范支持两种形式：
- **字符串引用**：`"slack"` -- 引用已配置的 MCP 服务器
- **内联定义**：`{ slack: { command: "...", args: [...] } }` -- Agent 专属的临时服务器

### 8.7.2 安全隔离

在 `strictPluginOnlyCustomization` 模式下，非管理员信任来源的 Agent 的 frontmatter MCP 配置会被跳过，只有插件、内置和管理策略来源的 Agent 可以加载其 MCP 服务器。

## 8.8 Agent Hooks、Memory 和 Skills

### 8.8.1 Agent Hooks

自定义 Agent 可以定义生命周期钩子，在 Agent 启动时注册，结束时自动清理：

```typescript
// runAgent.ts 中的 hooks 注册
if (agentDefinition.hooks && hooksAllowedForThisAgent) {
  registerFrontmatterHooks(
    rootSetAppState, agentId, agentDefinition.hooks,
    `agent '${agentDefinition.agentType}'`,
    true  // isAgent=true：将 Stop 钩子转为 SubagentStop
  );
}
// finally 块中清理
if (agentDefinition.hooks) {
  clearSessionHooks(rootSetAppState, agentId);
}
```

Agent 还支持 `SubagentStart` 钩子，在子智能体启动时执行并可以注入额外上下文。

### 8.8.2 Agent Memory

Agent 可以启用持久记忆，通过 `memory` 字段指定作用范围：

- **`user`**：`~/.claude/agent-memory/<agentType>/` -- 跨项目共享
- **`project`**：`.claude/agent-memory/<agentType>/` -- 项目级别，可提交到版本控制
- **`local`**：`.claude/agent-memory-local/<agentType>/` -- 本地级别，不提交到版本控制

记忆以 `MEMORY.md` 文件形式存储，在 Agent 启动时通过 `loadAgentMemoryPrompt()` 注入系统提示：

```typescript
getSystemPrompt: () => {
  if (isAutoMemoryEnabled() && memory) {
    return systemPrompt + '\n\n' + loadAgentMemoryPrompt(agentType, memory);
  }
  return systemPrompt;
}
```

Agent 还支持**记忆快照**机制--项目级快照可以初始化用户的本地记忆，确保团队知识同步。

### 8.8.3 Agent Skills

Agent 定义中的 `skills` 字段允许预加载 Skill（斜杠命令），在 Agent 启动时将 Skill 内容作为初始消息注入：

```typescript
const skillsToPreload = agentDefinition.skills ?? [];
// 解析 Skill 名称（支持精确匹配、插件前缀、后缀匹配三种策略）
// 加载 Skill 内容并作为 createUserMessage 注入 initialMessages
```

## 8.9 递归查询：子 Agent 中的嵌套 Agent 调用

### 8.9.1 递归的边界

子 Agent 本身也可以使用 AgentTool 创建更深层级的 Agent，形成递归调用链：

```
主 Agent
  +-- 子 Agent A（研究任务）
        +-- 子子 Agent B（搜索特定文件）
              +-- 子子子 Agent C（分析文件内容）
```

系统通过以下机制防止无限递归：

1. **Token 预算**：每层 Agent 有独立的上下文窗口
2. **超时控制**：每层 Agent 有独立的执行超时
3. **Fork 防护**：Fork 子进程不能再次 Fork
4. **Team 防护**：队友不能生成嵌套队友

### 8.9.2 递归的价值

递归 Agent 调用的价值在于**任务的自动分解**：

- 一个复杂的研究任务可以被分解为多个子任务
- 每个子任务可以被进一步分解为更具体的操作
- 每一层都可以选择最适合的 Agent 类型

这种结构类似于人类的组织管理：一个大项目被分解为多个子项目，每个子项目有专门的团队负责。

## 8.10 内置 Agent 类型：六种专业角色

Claude Code 内置了六种专业 Agent 类型（另有合成的 Fork Agent），每种都针对特定的工作模式进行了优化。内置 Agent 的激活受功能开关和入口点控制：

```typescript
function getBuiltInAgents(): AgentDefinition[] {
  const agents: AgentDefinition[] = [
    GENERAL_PURPOSE_AGENT,        // 始终可用
    STATUSLINE_SETUP_AGENT,       // 始终可用
  ];
  if (areExplorePlanAgentsEnabled()) {
    agents.push(EXPLORE_AGENT, PLAN_AGENT);  // 功能开关控制
  }
  if (isNonSdkEntrypoint) {
    agents.push(CLAUDE_CODE_GUIDE_AGENT);    // 非 SDK 入口
  }
  if (featureFlag && growthbookGate) {
    agents.push(VERIFICATION_AGENT);          // 功能开关 + GrowthBook 门控
  }
  return agents;
}
```

### 8.10.1 general-purpose Agent

**角色**：全能型执行者

- `tools: ['*']` -- 拥有完整的工具集（通配符）
- 无 `disallowedTools` -- 不限制任何工具
- 可以读取、编写、执行
- 适合实际的代码实现和修改
- 默认模型由 `getDefaultSubagentModel()` 决定

### 8.10.2 Explore Agent

**角色**：快速搜索和信息收集专家

- 无 `tools` 白名单（依赖 `disallowedTools` 黑名单）
- `disallowedTools: ['Agent', 'ExitPlanMode', 'Edit', 'Write', 'NotebookEdit']` -- 通过黑名单移除所有修改工具和递归 Agent 能力
- `model: 'haiku'`（外部用户）/ `'inherit'`（内部用户）
- `omitClaudeMd: true` -- 省略 CLAUDE.md 以节省 token
- 目标是快速、全面地收集信息
- 是一次性 Agent（`ONE_SHOT_BUILTIN_AGENT_TYPES`），完成后不需要 SendMessage 继续对话

### 8.10.3 Plan Agent

**角色**：架构师和规划者

- 与 Explore Agent 共享相同的工具列表（`tools: EXPLORE_AGENT.tools`）
- 同样使用 `disallowedTools` 黑名单：`['Agent', 'ExitPlanMode', 'Edit', 'Write', 'NotebookEdit']`
- `model: 'inherit'` -- 继承父级模型
- 专注于理解架构和设计方案，输出实现计划
- 同样是一次性 Agent

### 8.10.4 statusline-setup Agent

**角色**：状态栏配置专家

- `tools: ['Read', 'Edit']` -- 精确白名单，只需要读写工具
- `model: 'sonnet'` -- 固定使用 Sonnet
- `color: 'orange'` -- 专属 UI 颜色
- 专门负责配置用户的 Claude Code 状态栏设置

### 8.10.5 claude-code-guide Agent

**角色**：产品知识专家

- `tools: ['Glob', 'Grep', 'Read', 'WebFetch', 'WebSearch']`（或 Bash 替代搜索工具）
- `model: 'haiku'` -- 快速响应
- `permissionMode: 'dontAsk'` -- 不需要权限确认
- 帮助用户了解 Claude Code、Agent SDK 和 Claude API
- 动态构建系统提示，注入当前项目的自定义技能、Agent、MCP 服务器和设置

### 8.10.6 verification Agent

**角色**：质量验证专家

- `disallowedTools: ['Agent', 'ExitPlanMode', 'Edit', 'Write', 'NotebookEdit']` -- 禁止修改项目文件
- `background: true` -- 始终后台运行
- `model: 'inherit'` -- 继承父级模型
- `color: 'red'` -- 专属 UI 颜色
- 专门验证实现工作，输出结构化的 PASS/FAIL/PARTIAL 报告
- `criticalSystemReminder_EXPERIMENTAL` -- 每轮重新注入验证约束提醒

### 8.10.7 工具限制机制：disallowedTools 黑名单

一个关键的设计细节：Explore、Plan 和 Verification Agent 并不是"只提供只读工具"，而是通过 `disallowedTools` 黑名单从完整工具池中**移除**特定工具。实际运行时，工具过滤通过 `filterToolsForAgent()` 和 `resolveAgentTools()` 两层过滤实现：

```typescript
function filterToolsForAgent({ tools, isBuiltIn, isAsync }) {
  return tools.filter(tool => {
    if (tool.name.startsWith('mcp__')) return true;  // MCP 工具始终允许
    if (ALL_AGENT_DISALLOWED_TOOLS.has(tool.name)) return false;   // 全局禁止列表
    if (!isBuiltIn && CUSTOM_AGENT_DISALLOWED_TOOLS.has(tool.name)) return false;  // 自定义 Agent 禁止列表
    if (isAsync && !ASYNC_AGENT_ALLOWED_TOOLS.has(tool.name)) return false;  // 异步 Agent 工具限制
    return true;
  });
}

// 然后应用 Agent 定义的 disallowedTools
const resolvedTools = resolveAgentTools(agentDefinition, filteredTools).resolvedTools;
```

这种黑名单机制比白名单更灵活：Agent 可以声明自己不想要的工具，而无需列举所有需要的工具。使用 `tools: ['*']` 表示需要所有工具，然后通过 `disallowedTools` 精确排除。

### 8.10.8 model 参数的经济性设计

AgentTool 允许为每个子 Agent 选择模型：

- **Haiku**：快速、低成本，适合简单的搜索和读取（Explore、Guide Agent 默认）
- **Sonnet**：平衡能力和成本，适合大多数任务（Statusline Agent 默认）
- **Opus**：最强推理能力，适合复杂分析和规划
- **inherit**：继承父级模型，确保缓存兼容（Plan、Verification、Fork Agent）

这种设计使得 AI 可以根据任务的复杂度自动选择最经济的模型--就像人类管理者会根据任务的复杂程度分配给不同资历的团队成员。

## 8.11 小结与思考

AgentTool 是 Claude Code 架构中最具野心的设计之一。它实现了：

1. **递归智能**：AI 可以创建 AI，形成任意深度的任务分解树
2. **角色专业化**：六种以上内置 Agent 类型针对不同工作模式优化
3. **上下文隔离**：子 Agent 拥有独立的上下文和生命周期，通过 overrides 精确控制共享边界
4. **安全约束**：通过 disallowedTools 黑名单和权限继承确保安全
5. **缓存优化**：Fork Agent 通过共享系统提示和工具定义实现提示缓存命中
6. **多智能体协作**：Team 模式支持在独立进程中生成队友
7. **云边协同**：Remote Agent 支持在远程计算环境中执行
8. **可扩展性**：自定义 Agent 支持 MCP 服务器、Hooks、Memory 和 Skills

从更深的层面看，AgentTool 实现了一种**分形智能结构**：每个 Agent 都是一个完整的 Claude Code 实例的缩影，拥有工具、上下文和决策能力。而整个系统通过这些自相似的结构实现了复杂任务的自动分解和协作执行。

这种设计对构建大规模 AI 系统具有重要启示：未来的 AI 应用可能不是单个巨大的模型，而是由多个专业化 Agent 组成的协作网络--每个 Agent 都有自己的专长、工具和上下文，通过消息传递和任务分配协调工作。

**思考题**：
1. 如果递归深度没有限制，系统可能出现什么问题？除了技术限制外，还有什么语义层面的问题？
2. 子 Agent 的上下文隔离与共享之间如何平衡？哪些状态应该共享，哪些必须隔离？`shareSetAppState` 和 `shareAbortController` 的默认值为什么不同？
3. Fork Agent 的提示缓存共享机制要求哪些参数必须与父级完全一致？如果其中一个参数不同会发生什么？
4. Explore Agent 使用 `disallowedTools` 黑名单而非 `tools` 白名单的设计有什么优势？在什么场景下白名单更合适？
