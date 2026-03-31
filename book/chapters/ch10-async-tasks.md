# 第10章 后台任务与调度 —— 异步世界的编排

在交互式编程环境中，异步任务的编排是一个至关重要的设计问题。用户请求的操作可能需要长时间运行，某些操作需要在特定时间触发，还有些操作需要在隔离的环境中进行。Claude Code通过一套完整的后台任务与调度系统来应对这些挑战。

本章将深入探讨以下工具的设计与实现：TaskOutputTool、TaskStopTool、ScheduleCronTool系列、Worktree隔离工具（EnterWorktreeTool/ExitWorktreeTool）以及BriefTool。

当前源码里的异步任务系统已经明显超出这些工具本身。`src/tasks/` 目录表明后台任务至少包括：

- 本地 shell 任务
- 本地 agent 任务
- 远程 agent 任务
- 进程内 teammate 任务
- Dream / workflow / monitor 类任务

因此，本章的主题更接近“工具触发的后台执行运行时”，而不只是几个 task 相关工具。

## 10.1 TaskOutputTool：任务输出的统一通道

TaskOutputTool为后台任务提供了一个统一的输出通道。虽然从目录结构来看，该工具的实现相对简洁（主要定义了`TASK_OUTPUT_TOOL_NAME`常量），但它在整个任务系统中扮演着重要的角色。

### 命名规范与工具识别

```typescript
// src/tools/TaskOutputTool/constants.ts
export const TASK_OUTPUT_TOOL_NAME = 'TaskOutput'
```

这个常量定义遵循了Claude Code工具系统的命名规范：每个工具都有一个大写的`NAME`常量，用于系统内的工具注册、调用和识别。

在工具系统中，名称不仅是标识符，更是工具契约的组成部分。当一个工具被调用时，系统通过这个名称来定位对应的处理逻辑，并在日志、错误消息、UI展示中使用。

### 输出通道的设计意义

TaskOutputTool之所以重要，是因为它解决了异步任务执行中的一个根本问题：如何将长时间运行任务的输出实时传递给用户。

在同步调用模型中，函数返回值就是输出。但在后台任务模型中，任务可能在另一个进程中运行，或者运行时间远超单次API调用的超时限制。TaskOutputTool提供了一种机制，让任务能够持续向用户报告进度和结果。

这种设计类似于Unix中的管道概念，只不过这里的"管道"是结构化的工具调用，而非原始的stdout/stderr流。

## 10.2 TaskStopTool：任务生命周期的控制

TaskStopTool实现了任务终止的功能，是任务生命周期管理的重要组成部分。

### 输入验证的三层防护

```typescript
// src/tools/TaskStopTool/TaskStopTool.ts
async validateInput({ task_id, shell_id }, { getAppState }) {
  const id = task_id ?? shell_id
  if (!id) {
    return {
      result: false,
      message: 'Missing required parameter: task_id',
      errorCode: 1,
    }
  }

  const appState = getAppState()
  const task = appState.tasks?.[id] as TaskStateBase | undefined

  if (!task) {
    return {
      result: false,
      message: `No task found with ID: ${id}`,
      errorCode: 1,
    }
  }

  if (task.status !== 'running') {
    return {
      result: false,
      message: `Task ${id} is not running (status: ${task.status})`,
      errorCode: 3,
    }
  }

  return { result: true }
}
```

这段代码展示了一种优秀的输入验证模式：

1. **参数存在性检查**：确保必需的参数存在
2. **实体存在性检查**：确保任务在系统中存在
3. **状态合理性检查**：确保操作在当前状态下是合理的

每一层验证都会返回结构化的错误信息，包括明确的错误消息和错误代码。这种设计使得错误处理可以分层次进行，调用者可以根据错误代码采取不同的恢复策略。

### 向后兼容的别名机制

```typescript
export const TaskStopTool = buildTool({
  name: TASK_STOP_TOOL_NAME,
  aliases: ['KillShell'],  // 保留旧名称作为别名
  // ...
})
```

工具系统支持别名机制，这里`KillShell`是旧的工具名称，被保留为别名以确保向后兼容性。这对于维护长期运行的系统非常重要——当工具重构或改名时，旧脚本和会话记录仍然可以工作。

### 终止操作的业务逻辑

```typescript
async call({ task_id, shell_id }, { getAppState, setAppState }) {
  const id = task_id ?? shell_id
  if (!id) {
    throw new Error('Missing required parameter: task_id')
  }

  const result = await stopTask(id, {
    getAppState,
    setAppState,
  })

  return {
    data: {
      message: `Successfully stopped task: ${result.taskId} (${result.command})`,
      task_id: result.taskId,
      task_type: result.taskType,
      command: result.command,
    },
  }
}
```

实际的终止操作被委托给`stopTask`函数，这种关注点分离的设计使得工具层主要负责参数验证和结果格式化，而实际操作逻辑可以在其他模块中复用。

返回的数据结构包含了丰富的上下文信息：任务ID、任务类型和执行的命令。这些信息对于用户了解被终止的任务详情非常重要。

### 10.2.1 当前实现的一个关键细节：stopTask 已经是跨入口的共享能力

源码中的 `src/tasks/stopTask.ts` 明确写道，这段逻辑同时服务于：

- `TaskStopTool`，即 LLM 触发的工具调用
- SDK 的 `stop_task` control request

Claude Code 已经避免把“停止任务”写死在工具层，而是把它抽成了后台任务运行时的一部分。工具、SDK、UI 只是不同入口。

这类重构非常值得关注，因为它体现了一个成熟系统的演进方向：

**先有面向模型的工具接口，后沉淀为对所有入口复用的领域服务。**

## 10.2.2 后台任务的类型分层

`src/tasks/types.ts` 里的 `TaskState` 联合类型显示，Claude Code 的后台任务并不是单一模型：

```typescript
export type TaskState =
  | LocalShellTaskState
  | LocalAgentTaskState
  | RemoteAgentTaskState
  | InProcessTeammateTaskState
  | LocalWorkflowTaskState
  | MonitorMcpTaskState
  | DreamTaskState
```

这段定义揭示了三个事实：

1. **任务不是 shell 专属**
   shell 只是其中一种后台实体。

2. **本地与远程被统一进同一状态面**
   远程 agent 任务与本地 shell 任务都能进入同一个背景任务指示器。

3. **扩展能力已经渗透进调度层**
   workflow、monitor、dream 说明后台任务系统正在承载越来越多产品能力。

## 10.2.3 TodoWriteTool 与 Task v2 的并存

当前 `tools.ts` 中，`TodoWriteTool` 与 `TaskCreate/Update/List/Get` 并存，并由 `isTodoV2Enabled()` 分流：

- `TodoWriteTool` 继续维护旧的会话级待办列表
- `TaskCreateTool` 等新工具提供结构化任务对象

`TodoWriteTool` 本身也已经不是单纯的列表回写工具。它会：

1. 按 `agentId` 或 `sessionId` 分桶存储待办
2. 在主线程收尾时检测是否缺少 verification 步骤
3. 通过 tool result 对模型进行“流程性 nudging”

它在迁移期承担了双重角色：

- 数据写入接口
- 行为约束器

从工程角度看，这是一个很现实的折中：在新旧任务系统切换期间，用旧工具承载一部分流程治理，而把新能力逐步迁到 Task v2。

## 10.3 ScheduleCronTool：时间驱动的任务调度

ScheduleCronTool系列实现了基于cron表达式的任务调度功能。这是一个完整的调度系统，包含创建（CronCreateTool）、删除（CronDeleteTool）和列出（CronListTool）三个工具。

### CronCreateTool：任务的创建

#### Schema设计的艺术

```typescript
const inputSchema = lazySchema(() =>
  z.strictObject({
    cron: z
      .string()
      .describe(
        'Standard 5-field cron expression in local time: "M H DoM Mon DoW"',
      ),
    prompt: z.string().describe('The prompt to enqueue at each fire time.'),
    recurring: semanticBoolean(z.boolean().optional()).describe(
      `true (default) = fire on every cron match until deleted or auto-expired after ${DEFAULT_MAX_AGE_DAYS} days. false = fire once at the next match, then auto-delete.`,
    ),
    durable: semanticBoolean(z.boolean().optional()).describe(
      'true = persist to .claude/scheduled_tasks.json and survive restarts. false (default) = in-memory only, dies when this Claude session ends.',
    ),
  }),
)
```

这里的Schema设计展示了几个值得注意的实践：

1. **lazySchema**：使用延迟求值的Schema定义，避免模块加载时的性能开销
2. **semanticBoolean**：将可选布尔值转换为语义明确的描述，帮助用户理解参数含义
3. **详细的describe**：每个参数都有详细的描述字符串，这些描述会被用于生成向用户展示的prompt

#### 验证逻辑的层次化

```typescript
async validateInput(input): Promise<ValidationResult> {
  if (!parseCronExpression(input.cron)) {
    return {
      result: false,
      message: `Invalid cron expression '${input.cron}'. Expected 5 fields: M H DoM Mon DoW.`,
      errorCode: 1,
    }
  }

  if (nextCronRunMs(input.cron, Date.now()) === null) {
    return {
      result: false,
      message: `Cron expression '${input.cron}' does not match any calendar date in the next year.`,
      errorCode: 2,
    }
  }

  const tasks = await listAllCronTasks()
  if (tasks.length >= MAX_JOBS) {
    return {
      result: false,
      message: `Too many scheduled jobs (max ${MAX_JOBS}). Cancel one first.`,
      errorCode: 3,
    }
  }

  if (input.durable && getTeammateContext()) {
    return {
      result: false,
      message: 'durable crons are not supported for teammates (teammates do not persist across sessions)',
      errorCode: 4,
    }
  }

  return { result: true }
}
```

这个验证函数体现了防御性编程的多个方面：

1. **语法验证**：检查cron表达式是否符合语法规范
2. **语义验证**：检查表达式是否真的会在未来某个时间点触发
3. **资源限制**：防止任务数量无限增长
4. **业务规则验证**：确保durable cron只在适当的上下文中使用

每个错误都有唯一的errorCode，这为系统监控和问题诊断提供了便利。

#### 持久化与临时存储的选择

```typescript
async call({ cron, prompt, recurring = true, durable = false }) {
  const effectiveDurable = durable && isDurableCronEnabled()
  const id = await addCronTask(
    cron,
    prompt,
    recurring,
    effectiveDurable,
    getTeammateContext()?.agentId,
  )
  setScheduledTasksEnabled(true)

  return {
    data: {
      id,
      humanSchedule: cronToHuman(cron),
      recurring,
      durable: effectiveDurable,
    },
  }
}
```

这里的关键设计是`effectiveDurable`的计算：即使请求durable=true，如果系统级别的durable功能被禁用（通过`isDurableCronEnabled()`检查），实际的任务也会是临时的。这种"软开关"设计允许系统管理员在需要时快速关闭持久化功能，而不需要修改每个调用点。

### CronDeleteTool：任务的删除

```typescript
async validateInput(input): Promise<ValidationResult> {
  const tasks = await listAllCronTasks()
  const task = tasks.find(t => t.id === input.id)
  if (!task) {
    return {
      result: false,
      message: `No scheduled job with id '${input.id}'`,
      errorCode: 1,
    }
  }

  const ctx = getTeammateContext()
  if (ctx && task.agentId !== ctx.agentId) {
    return {
      result: false,
      message: `Cannot delete cron job '${input.id}': owned by another agent`,
      errorCode: 2,
    }
  }

  return { result: true }
}
```

CronDeleteTool的验证逻辑展示了多租户环境中的权限控制：每个teammate只能删除自己创建的cron任务。这种隔离机制防止了误操作和其他代理的干扰。

### CronListTool：任务的查询

```typescript
async call() {
  const allTasks = await listAllCronTasks()
  const ctx = getTeammateContext()
  const tasks = ctx
    ? allTasks.filter(t => t.agentId === ctx.agentId)
    : allTasks

  const jobs = tasks.map(t => ({
    id: t.id,
    cron: t.cron,
    humanSchedule: cronToHuman(t.cron),
    prompt: t.prompt,
    ...(t.recurring ? { recurring: true } : {}),
    ...(t.durable === false ? { durable: false } : {}),
  }))

  return { data: { jobs } }
}
```

CronListTool同样遵循了最小权限原则：teammate只能看到自己的任务，而team lead可以看到所有任务。这种设计在多代理协作中非常重要。

### 功能门控的实现

```typescript
export function isKairosCronEnabled(): boolean {
  return feature('AGENT_TRIGGERS')
    ? !isEnvTruthy(process.env.CLAUDE_CODE_DISABLE_CRON) &&
        getFeatureValue_CACHED_WITH_REFRESH(
          'tengu_kairos_cron',
          true,
          KAIROS_CRON_REFRESH_MS,
        )
    : false
}
```

这是功能门控（feature gating）的经典实现。三层检查确保了功能的灵活控制：

1. **编译时门控**：`feature('AGENT_TRIGGERS')`确保代码在不需要时可以被完全移除（DCE）
2. **环境变量覆盖**：`CLAUDE_CODE_DISABLE_CRON`允许本地开发时快速关闭功能
3. **远程配置**：GrowthBook提供实时的功能开关

这种分层设计使得功能可以在编译时、运行时和远程配置三个层面上进行控制，为产品的渐进式发布和紧急回滚提供了保障。

## 10.4 Worktree隔离：EnterWorktreeTool与ExitWorktreeTool

Git worktree功能允许在同一个仓库中同时检出多个分支到不同目录，这对于并行开发和测试非常有用。Claude Code通过EnterWorktreeTool和ExitWorktreeTool将这一能力集成到了AI编程助手的工作流中。

### EnterWorktreeTool：创建隔离环境

```typescript
async call(input) {
  if (getCurrentWorktreeSession()) {
    throw new Error('Already in a worktree session')
  }

  const mainRepoRoot = findCanonicalGitRoot(getCwd())
  if (mainRepoRoot && mainRepoRoot !== getCwd()) {
    process.chdir(mainRepoRoot)
    setCwd(mainRepoRoot)
  }

  const slug = input.name ?? getPlanSlug()
  const worktreeSession = await createWorktreeForSession(getSessionId(), slug)

  process.chdir(worktreeSession.worktreePath)
  setCwd(worktreeSession.worktreePath)
  setOriginalCwd(getCwd())
  saveWorktreeState(worktreeSession)
  clearSystemPromptSections()
  clearMemoryFileCaches()
  getPlansDirectory.cache.clear?.()

  return {
    data: {
      worktreePath: worktreeSession.worktreePath,
      worktreeBranch: worktreeSession.worktreeBranch,
      message: `Created worktree at ${worktreeSession.worktreePath}...`,
    },
  }
}
```

这段代码展示了一个复杂的会话状态转换过程：

1. **前置条件检查**：确保不在已有的worktree会话中
2. **环境准备**：切换到主仓库根目录
3. **worktree创建**：委托给专门的工具函数
4. **状态更新**：更新所有相关的会话状态
5. **缓存清理**：清除依赖旧环境的缓存

缓存清理步骤尤其重要：

```typescript
clearSystemPromptSections()
clearMemoryFileCaches()
getPlansDirectory.cache.clear?.()
```

这三种缓存清理分别对应：
- 系统提示词中的环境信息
- CLAUDE.md文件的解析结果
- 计划目录的路径缓存

当工作目录改变时，这些依赖路径的缓存必须失效，否则AI的行为可能与新环境不一致。

### ExitWorktreeTool：会话的清理与恢复

ExitWorktreeTool的设计比EnterWorktreeTool更为复杂，因为它需要处理一个关键问题：用户可能有未保存的工作。

#### 工作变更的统计

```typescript
async function countWorktreeChanges(
  worktreePath: string,
  originalHeadCommit: string | undefined,
): Promise<ChangeSummary | null> {
  const status = await execFileNoThrow('git', [
    '-C',
    worktreePath,
    'status',
    '--porcelain',
  ])
  if (status.code !== 0) {
    return null
  }
  const changedFiles = count(status.stdout.split('\n'), l => l.trim() !== '')

  if (!originalHeadCommit) {
    return null
  }

  const revList = await execFileNoThrow('git', [
    '-C',
    worktreePath,
    'rev-list',
    '--count',
    `${originalHeadCommit}..HEAD`,
  ])
  if (revList.code !== 0) {
    return null
  }
  const commits = parseInt(revList.stdout.trim(), 10) || 0

  return { changedFiles, commits }
}
```

这个函数统计worktree中的变更：未提交的文件数量和新增的提交数量。返回`null`表示无法确定状态（如git命令失败），调用者必须对此进行处理（fail-closed原则）。

#### 状态恢复的对称性

```typescript
function restoreSessionToOriginalCwd(
  originalCwd: string,
  projectRootIsWorktree: boolean,
): void {
  setCwd(originalCwd)
  setOriginalCwd(originalCwd)

  if (projectRootIsWorktree) {
    setProjectRoot(originalCwd)
    updateHooksConfigSnapshot()
  }

  saveWorktreeState(null)
  clearSystemPromptSections()
  clearMemoryFileCaches()
  getPlansDirectory.cache.clear?.()
}
```

这个函数是EnterWorktreeTool.call()中状态变更的逆操作。这种对称性设计确保了会话状态可以完全恢复，避免了状态泄漏或污染。

注意条件判断`if (projectRootIsWorktree)`：只有当项目根目录确实被设置为worktree路径时才需要恢复。这是因为EnterWorktreeTool有两种使用场景：
- 命令行参数`--worktree`启动时，会设置projectRoot
- 会话中通过EnterWorktreeTool进入worktree时，不设置projectRoot

#### 安全的删除流程

```typescript
if (input.action === 'remove' && !input.discard_changes) {
  const summary = await countWorktreeChanges(
    session.worktreePath,
    session.originalHeadCommit,
  )

  if (summary === null) {
    return {
      result: false,
      message: `Could not verify worktree state at ${session.worktreePath}. Refusing to remove without explicit confirmation.`,
      errorCode: 3,
    }
  }

  const { changedFiles, commits } = summary
  if (changedFiles > 0 || commits > 0) {
    const parts: string[] = []
    if (changedFiles > 0) {
      parts.push(
        `${changedFiles} uncommitted ${changedFiles === 1 ? 'file' : 'files'}`,
      )
    }
    if (commits > 0) {
      parts.push(
        `${commits} ${commits === 1 ? 'commit' : 'commits'} on ${session.worktreeBranch ?? 'the worktree branch'}`,
      )
    }
    return {
      result: false,
      message: `Worktree has ${parts.join(' and ')}. Removing will discard this work permanently. Confirm with the user, then re-invoke with discard_changes: true — or use action: "keep" to preserve the worktree.`,
      errorCode: 2,
    }
  }
}
```

这段代码体现了"安全第一"的设计哲学：

1. **状态验证**：在删除前验证worktree状态
2. **变更检测**：准确检测未提交的文件和新增的提交
3. **明确确认**：有变更时要求用户明确设置`discard_changes: true`
4. **友好提示**：提供清晰的替代方案（使用action: "keep"）

这种多层防护确保了用户不会意外丢失工作成果。

## 10.5 BriefTool：主动消息的传递

BriefTool实现了一个特殊的通信通道：AI可以向用户主动发送消息，而不仅仅是对用户输入做出响应。这对于后台任务和长时间运行的操作尤其重要。

### 功能激活的多层控制

```typescript
export function isBriefEnabled(): boolean {
  return feature('KAIROS') || feature('KAIROS_BRIEF')
    ? (getKairosActive() || getUserMsgOptIn()) && isBriefEntitled()
    : false
}

export function isBriefEntitled(): boolean {
  return feature('KAIROS') || feature('KAIROS_BRIEF')
    ? getKairosActive() ||
        isEnvTruthy(process.env.CLAUDE_CODE_BRIEF) ||
        getFeatureValue_CACHED_WITH_REFRESH(
          'tengu_kairos_brief',
          false,
          KAIROS_BRIEF_REFRESH_MS,
        )
    : false
}
```

这里有两层检查：

1. **isBriefEntitled()**：检查用户是否被授权使用Brief功能
2. **isBriefEnabled()**：检查用户是否激活了Brief功能

这种区分允许系统先决定谁可以使用某个功能，再由用户选择是否启用它。类似于应用商店中的"应用内购买"：用户拥有购买权限，但需要主动点击购买。

### 状态标签的语义设计

```typescript
const inputSchema = lazySchema(() =>
  z.strictObject({
    message: z.string().describe('The message for the user. Supports markdown formatting.'),
    attachments: z.array(z.string()).optional().describe(
      'Optional file paths (absolute or relative to cwd) to attach.',
    ),
    status: z.enum(['normal', 'proactive']).describe(
      `Use 'proactive' when you're surfacing something the user hasn't asked for and needs to see now — task completion while they're away, a blocker you hit, an unsolicited status update. Use 'normal' when replying to something they just said.`,
    ),
  }),
)
```

`status`字段的设计体现了对通信意图的精确建模：

- **normal**：响应式通信，用户问，AI答
- **proactive**：主动通信，AI在用户没有明确请求时发送消息

这种区分对于消息路由和用户体验非常重要。主动消息可能需要不同的展示方式、通知策略或交互模式。

### 附件系统的设计

```typescript
async call({ message, attachments, status }, context) {
  const sentAt = new Date().toISOString()
  logEvent('tengu_brief_send', {
    proactive: status === 'proactive',
    attachment_count: attachments?.length ?? 0,
  })

  if (!attachments || attachments.length === 0) {
    return { data: { message, sentAt } }
  }

  const appState = context.getAppState()
  const resolved = await resolveAttachments(attachments, {
    replBridgeEnabled: appState.replBridgeEnabled,
    signal: context.abortController.signal,
  })

  return {
    data: { message, attachments: resolved, sentAt },
  }
}
```

附件解析是一个异步操作，可能涉及：
- 文件存在性检查
- 文件类型检测
- 文件大小计算
- 图片预览生成

通过`abortController.signal`支持操作取消，这是长时间运行操作的最佳实践。

### Schema演进的兼容性

```typescript
const outputSchema = lazySchema(() =>
  z.object({
    message: z.string().describe('The message'),
    attachments: z
      .array(
        z.object({
          path: z.string(),
          size: z.number(),
          isImage: z.boolean(),
          file_uuid: z.string().optional(),
        }),
      )
      .optional()
      .describe('Resolved attachment metadata'),
    sentAt: z
      .string()
      .optional()
      .describe(
        'ISO timestamp captured at tool execution on the emitting process. Optional — resumed sessions replay pre-sentAt outputs verbatim.',
      ),
  }),
)
```

注意注释中的关键说明：`sentAt`是可选的，因为"会话恢复时会重放没有sentAt字段的旧输出"。这是API设计向后兼容性的一个重要原则：新增字段应该是可选的，除非可以保证所有消费者都已升级。

## 10.6 小结

本章探讨的后台任务与调度系统展示了Claude Code在处理异步操作时的设计哲学：

### 1. 防御性编程的实践
- 多层验证：语法、语义、业务规则
- 错误代码系统：为每种错误分配唯一代码
- Fail-closed原则：无法确定状态时拒绝操作

### 2. 状态管理的对称性
- EnterWorktreeTool和ExitWorktreeTool的状态操作是互逆的
- 缓存的建立和清理成对出现
- 会话状态的完整保存和恢复

### 3. 功能门控的分层设计
- 编译时门控（DCE）
- 环境变量覆盖
- 远程配置开关

### 4. 用户体验的细致考虑
- 详细的错误消息和恢复建议
- 工作成果的保护机制
- 清晰的操作语义（normal vs proactive）

这些设计原则和实现技巧不仅适用于AI编程助手，也是构建可靠异步系统的一般性指南。在下一章中，我们将探讨组件体系的设计，了解这些工具是如何在终端UI中呈现给用户的。
