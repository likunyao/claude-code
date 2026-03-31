# 第9章 Team与Task —— 群体智能的协调机制

> "一群蚂蚁能够建造复杂的蚁巢，并非因为每只蚂蚁都知晓整个蓝图，而是通过简单的局部交互规则涌现出群体智慧。" —— 《复杂系统导论》

在Claude Code中，多智能体协作系统的设计体现了类似的群体智能思想。Team（团队）提供了Agent集合的组织形式，Task（任务）则定义了协作的目标与依赖关系。两者结合，构建了一个能够并行处理复杂软件工程任务的协调机制。

在当前源码里，**“Task”已经不只是团队协作清单，而是独立的后台运行时抽象。** `src/tasks/` 目录下存在多种具体任务实现，包括 `LocalShellTask`、`LocalAgentTask`、`RemoteAgentTask`、`InProcessTeammateTask`、`DreamTask` 等。Task 既指“任务列表里的工作项”，也指“AppState 中可被调度、展示、停止的运行实体”。

本章将深入剖析：
1. TeamCreateTool与TeamDeleteTool——团队生命周期的管理
2. SendMessageTool——Agent间异步消息传递协议
3. Task系统的四大核心工具（Create、Get、Update、List）
4. 任务依赖的DAG模型与阻塞机制
5. 空闲Agent的工作窃取发现机制
6. 协调者模式（Coordinator Mode）的中心调度策略

---

## 9.1 TeamCreateTool/TeamDeleteTool：团队的创建与销毁

团队是Claude Code中多个Agent协作的容器。每个团队都有一个唯一的team-lead（团队领导者），负责协调其他teammate（队友）的工作。

### 9.1.1 TeamCreateTool的核心职责

TeamCreateTool的主要职责是初始化一个团队环境，包括：

**1. 唯一名称生成**
```typescript
function generateUniqueTeamName(providedName: string): string {
  if (!readTeamFile(providedName)) {
    return providedName
  }
  return generateWordSlug()  // 生成随机slug避免冲突
}
```

**2. Team文件结构**
```typescript
type TeamFile = {
  name: string
  description?: string
  createdAt: number
  leadAgentId: string              // team-lead@{teamName}
  leadSessionId?: string           // 领导者的会话UUID
  hiddenPaneIds?: string[]         // UI隐藏的窗格
  teamAllowedPaths?: TeamAllowedPath[]  // 全员可编辑路径
  members: Array<Member>           // 队伍成员列表
}
```

**3. 领导者Agent ID的确定性生成**
```typescript
const leadAgentId = formatAgentId(TEAM_LEAD_NAME, finalTeamName)
// 结果格式: "team-lead@my-project"
```

### 9.1.2 任务列表与团队的一对一绑定

一个关键设计决策是：Team = Project = TaskList。每个新创建的团队都会获得一个独立的任务列表目录：

```typescript
// 重置任务列表，确保编号从1开始
const taskListId = sanitizeName(finalTeamName)
await resetTaskList(taskListId)
await ensureTasksDir(taskListId)

// 注册团队名称，使getTaskListId()返回正确的目录
setLeaderTeamName(sanitizeName(finalTeamName))
```

这种设计确保了：
- 任务编号在团队内唯一且连续
- 领导者和所有teammate看到的是同一份任务列表
- 团队销毁时任务目录可以一起清理

### 9.1.3 Team 不是 Task 的唯一宿主

当前源码里有两套相互关联但不相同的“任务”概念：

1. **任务列表（Todo / Task List）**
   面向规划、依赖、负责人、状态流转。
   这部分由 `TaskCreateTool`、`TaskUpdateTool`、`TaskListTool` 等操作。

2. **后台运行任务（Background Tasks）**
   面向 shell、agent、workflow、remote session 等实际执行体。
   这部分由 `src/tasks/` 下的具体类型实现，并挂在 `AppState.tasks` 上。

Team 主要影响的是第一类任务的协作边界，但第二类任务可以在没有 team 的情况下存在。这个区分非常重要，否则会误以为所有 task 都是“团队任务”。

### 9.1.4 TeamDeleteTool的清理机制

TeamDeleteTool负责团队生命周期结束时的清理工作，但有一个重要的安全检查：

```typescript
// 检查是否有活跃成员
const activeMembers = nonLeadMembers.filter(m => m.isActive !== false)
if (activeMembers.length > 0) {
  return {
    success: false,
    message: `Cannot cleanup team with ${activeMembers.length} active member(s)...`
  }
}
```

只有当所有队友都处于非活跃状态时，才允许删除团队。这防止了在队友仍在工作时意外销毁共享资源。

清理过程包括：
1. 调用`cleanupTeamDirectories(teamName)`删除团队目录
2. 调用`clearLeaderTeamName()`清除领导者团队名称绑定
3. 清理任务目录和Git worktree
4. 从AppState中清除teamContext和inbox

### 9.1.5 会话级清理追踪

为防止异常退出导致的资源泄漏，Claude Code实现了会话级清理追踪：

```typescript
// 在bootstrap/state.ts中追踪
function getSessionCreatedTeams(): Set<string> {
  return globalThis.__SESSION_CREATED_TEAMS__ ??= new Set()
}

// 创建团队时注册
registerTeamForSessionCleanup(finalTeamName)

// 会话结束时自动清理
export async function cleanupSessionTeams(): Promise<void> {
  const teams = Array.from(getSessionCreatedTeams())
  await Promise.allSettled(teams.map(name => cleanupTeamDirectories(name)))
}
```

---

## 9.2 SendMessageTool：Agent间的消息传递协议

SendMessageTool是Agent之间异步通信的核心机制。它支持三种消息类型：
1. **普通消息**——纯文本发送给指定队友
2. **广播消息**——发送给团队所有成员
3. **结构化消息**——带类型的协议消息（如shutdown_request）

### 9.2.1 输入模式设计

SendMessageTool的输入模式使用Zod的discriminated union实现类型安全的结构化消息：

```typescript
const StructuredMessage = z.discriminatedUnion('type', [
  z.object({
    type: z.literal('shutdown_request'),
    reason: z.string().optional(),
  }),
  z.object({
    type: z.literal('shutdown_response'),
    request_id: z.string(),
    approve: semanticBoolean(),
    reason: z.string().optional(),
  }),
  z.object({
    type: z.literal('plan_approval_response'),
    request_id: z.string(),
    approve: semanticBoolean(),
    feedback: z.string().optional(),
  }),
])
```

这种设计使得编译期就能确保消息结构的正确性，同时保持了运行时的验证。

### 9.2.2 邮箱（Mailbox）机制

消息传递的核心是邮箱机制，每个Agent都有一个独立的邮箱：

```typescript
async function handleMessage(
  recipientName: string,
  content: string,
  summary: string | undefined,
  context: ToolUseContext,
): Promise<{ data: MessageOutput }> {
  const teamName = getTeamName(appState.teamContext)
  const senderName = getAgentName() || TEAM_LEAD_NAME
  const senderColor = getTeammateColor()

  await writeToMailbox(
    recipientName,
    {
      from: senderName,
      text: content,
      summary,
      timestamp: new Date().toISOString(),
      color: senderColor,
    },
    teamName,
  )
  // ...
}
```

邮箱的实现基于文件系统，每个Agent的邮箱是`~/.claude/teams/{teamName}/mailboxes/{agentName}.json`。这种设计的优势：
- **持久化**：消息在进程重启后仍然存在
- **跨进程**：支持多进程Agent之间的通信
- **简单可靠**：无需额外的消息队列服务

### 9.2.3 消息与任务的真正耦合点：不是 RPC，而是松耦合协作

从整体架构看，`SendMessageTool` 的作用更像 **协作面**，而不是严格的 RPC 面：

- 它适合发出计划确认、关闭请求、阶段性同步
- 它不承诺像函数调用那样同步返回结果
- 真正的执行进度与生命周期，更多落在 task 状态、spinner、background task UI 和状态通知体系里

Claude Code 的多智能体架构并没有走“代理之间相互调用函数”的传统 actor 路线，而是更接近：

**任务状态机 + 异步邮箱 + UI 可视化反馈** 的混合模型。

## 9.3 Task v2：从 Todo 列表走向结构化任务系统

`TaskCreateTool / TaskGetTool / TaskUpdateTool / TaskListTool` 这一组新工具，由 `isTodoV2Enabled()` 控制启用，代表 Claude Code 已经从旧的 `TodoWriteTool` 逐渐迁移到结构化任务列表。

`TaskCreateTool` 的输入已经不是“整表覆盖”，而是结构化字段：

```typescript
z.strictObject({
  subject: z.string(),
  description: z.string(),
  activeForm: z.string().optional(),
  metadata: z.record(z.string(), z.unknown()).optional(),
})
```

这种设计相较旧 Todo 方案有三个明显升级：

1. **增量操作**
   不再要求模型每次回写整张列表，而是创建、读取、更新单个任务。

2. **更强的可扩展性**
   `metadata` 为工作流、插件、外部系统接入留下空间。

3. **Hook 集成**
   任务创建后会触发 `executeTaskCreatedHooks(...)`，说明任务系统已经被纳入事件驱动扩展面，而不是单纯 UI 功能。

从设计思想上看，Task v2 把“计划”从模型自述，提升成了系统级对象。

### 9.2.4 广播消息的实现

广播消息需要遍历团队文件的所有成员，排除发送者自己：

```typescript
async function handleBroadcast(
  content: string,
  summary: string | undefined,
  context: ToolUseContext,
): Promise<{ data: BroadcastOutput }> {
  const teamFile = await readTeamFileAsync(teamName)
  const senderName = getAgentName() || TEAM_LEAD_NAME

  const recipients: string[] = []
  for (const member of teamFile.members) {
    if (member.name.toLowerCase() === senderName.toLowerCase()) {
      continue
    }
    recipients.push(member.name)
  }

  for (const recipientName of recipients) {
    await writeToMailbox(recipientName, { /* ... */ }, teamName)
  }

  return {
    data: {
      success: true,
      message: `Message broadcast to ${recipients.length} teammate(s)`,
      recipients,
    },
  }
}
```

### 9.2.5 结构化消息：关闭请求与响应

关闭流程是结构化消息的典型应用场景：

```typescript
// 领导者发送关闭请求
async function handleShutdownRequest(
  targetName: string,
  reason: string | undefined,
): Promise<{ data: RequestOutput }> {
  const requestId = generateRequestId('shutdown', targetName)
  const shutdownMessage = createShutdownRequestMessage({
    requestId,
    from: senderName,
    reason,
  })

  await writeToMailbox(targetName, {
    from: senderName,
    text: jsonStringify(shutdownMessage),
    timestamp: new Date().toISOString(),
    color: getTeammateColor(),
  }, teamName)

  return {
    data: {
      success: true,
      message: `Shutdown request sent to ${targetName}. Request ID: ${requestId}`,
      request_id: requestId,
      target: targetName,
    },
  }
}
```

队友收到请求后，可以选择批准或拒绝：

```typescript
// 队友批准关闭
async function handleShutdownApproval(
  requestId: string,
): Promise<{ data: ResponseOutput }> {
  const approvedMessage = createShutdownApprovedMessage({
    requestId,
    from: agentName,
    paneId: ownPaneId,
    backendType: ownBackendType,
  })

  await writeToMailbox(TEAM_LEAD_NAME, {
    from: agentName,
    text: jsonStringify(approvedMessage),
    timestamp: new Date().toISOString(),
    color: getTeammateColor(),
  }, teamName)

  // 触发中止控制器
  if (agentId) {
    const task = findTeammateTaskByAgentId(agentId, appState.tasks)
    if (task?.abortController) {
      task.abortController.abort()
    }
  }

  // 对于非进程内队友，调用gracefulShutdown
  if (ownBackendType !== 'in-process') {
    setImmediate(async () => {
      await gracefulShutdown(0, 'other')
    })
  }

  return {
    data: {
      success: true,
      message: `Shutdown approved. Agent ${agentName} is now exiting.`,
      request_id: requestId,
    },
  }
}
```

### 9.2.6 跨会话消息传递

SendMessageTool还支持跨会话的消息传递，通过`bridge:`前缀实现：

```typescript
if (addr.scheme === 'bridge') {
  const result = await postInterClaudeMessage(addr.target, input.message)
  return {
    data: {
      success: result.ok,
      message: result.ok
        ? `"${preview}" → ${input.to}`
        : `Failed to send to ${input.to}: ${result.error}`,
    },
  }
}
```

这允许两个独立的Claude Code会话之间进行通信，为远程协作和分布式任务处理提供了基础。

---

Task 系统的具体实现继续展开如下：

Task系统是团队协作的核心，提供了一套完整的任务管理工具。

### 9.3.1 Task数据结构

```typescript
type Task = {
  id: string              // 任务ID（数字字符串）
  subject: string         // 简短标题
  description: string     // 详细描述
  activeForm?: string     // 进行时形式的描述（用于spinner）
  owner?: string          // 所有者Agent ID
  status: TaskStatus      // pending | in_progress | completed
  blocks: string[]        // 此任务阻塞的任务ID列表
  blockedBy: string[]     // 阻塞此任务的任务ID列表
  metadata?: Record<string, unknown>  // 任意元数据
}
```

### 9.3.2 TaskCreateTool：任务创建

TaskCreateTool负责创建新任务，其核心功能是：

```typescript
export async function createTask(
  taskListId: string,
  taskData: Omit<Task, 'id'>,
): Promise<string> {
  const lockPath = await ensureTaskListLockFile(taskListId)

  let release: (() => Promise<void>) | undefined
  try {
    // 获取独占锁
    release = await lockfile.lock(lockPath, LOCK_OPTIONS)

    // 读取当前最高ID
    const highestId = await findHighestTaskId(taskListId)
    const id = String(highestId + 1)

    // 写入任务文件
    const task: Task = { id, ...taskData }
    const path = getTaskPath(taskListId, id)
    await writeFile(path, jsonStringify(task, null, 2))

    notifyTasksUpdated()
    return id
  } finally {
    if (release) {
      await release()
    }
  }
}
```

**高水位标记（High Water Mark）**机制确保即使任务被删除，ID也不会重复使用：

```typescript
async function findHighestTaskId(taskListId: string): Promise<number> {
  const [fromFiles, fromMark] = await Promise.all([
    findHighestTaskIdFromFiles(taskListId),
    readHighWaterMark(taskListId),
  ])
  return Math.max(fromFiles, fromMark)
}
```

### 9.3.3 TaskGetTool：任务检索

TaskGetTool根据ID获取单个任务的详细信息：

```typescript
export async function getTask(
  taskListId: string,
  taskId: string,
): Promise<Task | null> {
  const path = getTaskPath(taskListId, taskId)
  try {
    const content = await readFile(path, 'utf-8')
    const data = jsonParse(content) as { status?: string }

    // 状态迁移（兼容旧版本）
    if (process.env.USER_TYPE === 'ant') {
      if (data.status === 'open') data.status = 'pending'
      else if (data.status === 'resolved') data.status = 'completed'
      // ...
    }

    const parsed = TaskSchema().safeParse(data)
    return parsed.success ? parsed.data : null
  } catch (e) {
    const code = getErrnoCode(e)
    return code === 'ENOENT' ? null : null
  }
}
```

### 9.3.4 TaskListTool：任务列表

TaskListTool返回所有任务的摘要信息，用于快速浏览团队工作状态：

```typescript
export async function listTasks(taskListId: string): Promise<Task[]> {
  const dir = getTasksDir(taskListId)
  let files: string[]
  try {
    files = await readdir(dir)
  } catch {
    return []
  }

  const taskIds = files
    .filter(f => f.endsWith('.json'))
    .map(f => f.replace('.json', ''))

  const results = await Promise.all(
    taskIds.map(id => getTask(taskListId, id))
  )
  return results.filter((t): t is Task => t !== null)
}
```

**关键优化**：在返回结果时过滤掉已解析任务的阻塞关系：

```typescript
const resolvedTaskIds = new Set(
  allTasks.filter(t => t.status === 'completed').map(t => t.id),
)

const tasks = allTasks.map(task => ({
  id: task.id,
  subject: task.subject,
  status: task.status,
  owner: task.owner,
  blockedBy: task.blockedBy.filter(id => !resolvedTaskIds.has(id)),
}))
```

这确保了UI只显示真正未完成的阻塞关系。

### 9.3.5 TaskUpdateTool：任务更新

TaskUpdateTool是最复杂的任务工具，支持多种更新操作：

```typescript
const inputSchema = lazySchema(() => {
  const TaskUpdateStatusSchema = TaskStatusSchema().or(z.literal('deleted'))

  return z.strictObject({
    taskId: z.string(),
    subject: z.string().optional(),
    description: z.string().optional(),
    activeForm: z.string().optional(),
    status: TaskUpdateStatusSchema.optional(),
    addBlocks: z.array(z.string()).optional(),
    addBlockedBy: z.array(z.string()).optional(),
    owner: z.string().optional(),
    metadata: z.record(z.string(), z.unknown()).optional(),
  })
})
```

**自动分配所有者**：当队友将任务标记为in_progress时，自动设置owner：

```typescript
if (
  isAgentSwarmsEnabled() &&
  status === 'in_progress' &&
  owner === undefined &&
  !existingTask.owner
) {
  const agentName = getAgentName()
  if (agentName) {
    updates.owner = agentName
    updatedFields.push('owner')
  }
}
```

**任务完成钩子**：当任务标记为completed时执行预配置的钩子：

```typescript
if (status === 'completed') {
  const generator = executeTaskCompletedHooks(
    taskId,
    existingTask.subject,
    existingTask.description,
    getAgentName(),
    getTeamName(),
    undefined,
    context?.abortController?.signal,
    undefined,
    context,
  )

  for await (const result of generator) {
    if (result.blockingError) {
      blockingErrors.push(getTaskCompletedHookMessage(result.blockingError))
    }
  }

  if (blockingErrors.length > 0) {
    return { success: false, error: blockingErrors.join('\n') }
  }
}
```

**邮箱通知**：当任务所有权变更时，通知新所有者：

```typescript
if (updates.owner && isAgentSwarmsEnabled()) {
  const assignmentMessage = JSON.stringify({
    type: 'task_assignment',
    taskId,
    subject: existingTask.subject,
    description: existingTask.description,
    assignedBy: senderName,
    timestamp: new Date().toISOString(),
  })

  await writeToMailbox(
    updates.owner,
    {
      from: senderName,
      text: assignmentMessage,
      timestamp: new Date().toISOString(),
      color: senderColor,
    },
    taskListId,
  )
}
```

---

## 9.4 任务依赖与阻塞：blocks/blockedBy的DAG模型

Task系统通过`blocks`和`blockedBy`字段构建任务依赖图（DAG - Directed Acyclic Graph）。

### 9.4.1 依赖关系的双向维护

每个依赖关系在两个方向上维护：

```typescript
export async function blockTask(
  taskListId: string,
  fromTaskId: string,  // 阻塞者
  toTaskId: string,    // 被阻塞者
): Promise<boolean> {
  const [fromTask, toTask] = await Promise.all([
    getTask(taskListId, fromTaskId),
    getTask(taskListId, toTaskId),
  ])
  if (!fromTask || !toTask) {
    return false
  }

  // 更新源任务：A阻塞B
  if (!fromTask.blocks.includes(toTaskId)) {
    await updateTask(taskListId, fromTaskId, {
      blocks: [...fromTask.blocks, toTaskId],
    })
  }

  // 更新目标任务：B被A阻塞
  if (!toTask.blockedBy.includes(fromTaskId)) {
    await updateTask(taskListId, toTaskId, {
      blockedBy: [...toTask.blockedBy, fromTaskId],
    })
  }

  return true
}
```

这种双向存储设计使得：
- 查询某个任务阻塞了谁变得高效（直接读取`blocks`）
- 查询某个任务被谁阻塞也变得高效（直接读取`blockedBy`）

### 9.4.2 依赖解析与阻塞检查

当Agent尝试领取任务时，系统会检查是否有未完成的依赖：

```typescript
export async function claimTask(
  taskListId: string,
  taskId: string,
  claimantAgentId: string,
  options: ClaimTaskOptions = {},
): Promise<ClaimTaskResult> {
  // ...

  // 检查未解决的阻塞者
  const allTasks = await listTasks(taskListId)
  const unresolvedTaskIds = new Set(
    allTasks.filter(t => t.status !== 'completed').map(t => t.id),
  )
  const blockedByTasks = task.blockedBy.filter(id =>
    unresolvedTaskIds.has(id),
  )

  if (blockedByTasks.length > 0) {
    return {
      success: false,
      reason: 'blocked',
      task,
      blockedByTasks,
    }
  }

  // 领取任务
  const updated = await updateTaskUnsafe(taskListId, taskId, {
    owner: claimantAgentId,
  })
  return { success: true, task: updated! }
}
```

### 9.4.3 任务删除的级联清理

当任务被删除时，需要清理其他任务中对它的引用：

```typescript
export async function deleteTask(
  taskListId: string,
  taskId: string,
): Promise<boolean> {
  const path = getTaskPath(taskListId, taskId)

  try {
    // 更新高水位标记，防止ID重用
    const numericId = parseInt(taskId, 10)
    if (!isNaN(numericId)) {
      const currentMark = await readHighWaterMark(taskListId)
      if (numericId > currentMark) {
        await writeHighWaterMark(taskListId, numericId)
      }
    }

    // 删除任务文件
    await unlink(path)

    // 移除其他任务中对已删除任务的引用
    const allTasks = await listTasks(taskListId)
    for (const task of allTasks) {
      const newBlocks = task.blocks.filter(id => id !== taskId)
      const newBlockedBy = task.blockedBy.filter(id => id !== taskId)

      if (
        newBlocks.length !== task.blocks.length ||
        newBlockedBy.length !== task.blockedBy.length
      ) {
        await updateTask(taskListId, task.id, {
          blocks: newBlocks,
          blockedBy: newBlockedBy,
        })
      }
    }

    notifyTasksUpdated()
    return true
  } catch {
    return false
  }
}
```

---

## 9.5 工作窃取：空闲Agent的任务发现机制

当Agent完成当前任务后，需要发现并领取下一个可用任务。这个过程称为"工作窃取"（Work Stealing）。

### 9.5.1 任务领取的原子性保证

在多Agent环境下，任务领取必须是原子的，否则可能出现竞态条件：

```typescript
export async function claimTask(
  taskListId: string,
  taskId: string,
  claimantAgentId: string,
  options: ClaimTaskOptions = {},
): Promise<ClaimTaskResult> {
  const taskPath = getTaskPath(taskListId, taskId)

  // 检查任务是否存在
  const taskBeforeLock = await getTask(taskListId, taskId)
  if (!taskBeforeLock) {
    return { success: false, reason: 'task_not_found' }
  }

  if (options.checkAgentBusy) {
    return claimTaskWithBusyCheck(taskListId, taskId, claimantAgentId)
  }

  let release: (() => Promise<void>) | undefined
  try {
    // 获取任务文件的独占锁
    release = await lockfile.lock(taskPath, LOCK_OPTIONS)

    // 读取当前任务状态
    const task = await getTask(taskListId, taskId)
    if (!task) {
      return { success: false, reason: 'task_not_found' }
    }

    // 检查是否已被其他Agent领取
    if (task.owner && task.owner !== claimantAgentId) {
      return { success: false, reason: 'already_claimed', task }
    }

    // 检查是否已完成
    if (task.status === 'completed') {
      return { success: false, reason: 'already_resolved', task }
    }

    // 检查未解决的阻塞者
    const allTasks = await listTasks(taskListId)
    const unresolvedTaskIds = new Set(
      allTasks.filter(t => t.status !== 'completed').map(t => t.id),
    )
    const blockedByTasks = task.blockedBy.filter(id =>
      unresolvedTaskIds.has(id),
    )

    if (blockedByTasks.length > 0) {
      return { success: false, reason: 'blocked', task, blockedByTasks }
    }

    // 领取任务
    const updated = await updateTaskUnsafe(taskListId, taskId, {
      owner: claimantAgentId,
    })
    return { success: true, task: updated! }
  } finally {
    if (release) {
      await release()
    }
  }
}
```

### 9.5.2 忙碌检查：防止Agent monopolize任务

为了防止单个Agent同时领取过多任务，系统提供了忙碌检查：

```typescript
async function claimTaskWithBusyCheck(
  taskListId: string,
  taskId: string,
  claimantAgentId: string,
): Promise<ClaimTaskResult> {
  const lockPath = await ensureTaskListLockFile(taskListId)

  let release: (() => Promise<void>) | undefined
  try {
    // 获取任务列表级别的锁
    release = await lockfile.lock(lockPath, LOCK_OPTIONS)

    // 原子地读取所有任务
    const allTasks = await listTasks(taskListId)

    // 找到要领取的任务
    const task = allTasks.find(t => t.id === taskId)
    if (!task) {
      return { success: false, reason: 'task_not_found' }
    }

    // 检查Agent是否已有未完成的任务
    const agentOpenTasks = allTasks.filter(
      t =>
        t.status !== 'completed' &&
        t.owner === claimantAgentId &&
        t.id !== taskId,
    )

    if (agentOpenTasks.length > 0) {
      return {
        success: false,
        reason: 'agent_busy',
        task,
        busyWithTasks: agentOpenTasks.map(t => t.id),
      }
    }

    // 领取任务
    const updated = await updateTask(taskListId, taskId, {
      owner: claimantAgentId,
    })
    return { success: true, task: updated! }
  } finally {
    if (release) {
      await release()
    }
  }
}
```

### 9.5.3 任务发现策略

Agent如何发现下一个可领取的任务？系统提供`listTasks`工具，Agent可以：

1. 获取所有任务列表
2. 过滤出状态为pending且blockedBy为空的任务
3. 按ID顺序选择（或其他策略）
4. 尝试claim

```typescript
// Agent端的工作窃取逻辑示例
async function findAndClaimNextTask(): Promise<Task | null> {
  const tasks = await listTasks(getTaskListId())

  // 按ID排序，确保优先处理旧任务
  const sortedTasks = tasks
    .filter(t =>
      t.status === 'pending' &&
      t.blockedBy.length === 0 &&
      !t.owner  // 未被领取
    )
    .sort((a, b) => parseInt(a.id) - parseInt(b.id))

  for (const task of sortedTasks) {
    const result = await claimTask(
      getTaskListId(),
      task.id,
      getMyAgentId(),
      { checkAgentBusy: true }
    )

    if (result.success) {
      return result.task!
    }
  }

  return null  // 没有可用任务
}
```

---

## 9.6 协调者模式：coordinator/的中心调度

协调者模式（Coordinator Mode）是Claude Code的一种特殊运行模式，专门用于多Agent任务的编排。

### 9.6.1 协调者与普通模式的区别

```typescript
export function isCoordinatorMode(): boolean {
  if (feature('COORDINATOR_MODE')) {
    return isEnvTruthy(process.env.CLAUDE_CODE_COORDINATOR_MODE)
  }
  return false
}
```

在协调者模式下，Claude Code的角色从"直接执行者"转变为"任务分发者和结果综合者"。

### 9.6.2 协调者系统提示词

协调者模式的系统提示词详细说明了其职责和工作流：

```typescript
export function getCoordinatorSystemPrompt(): string {
  return `You are Claude Code, an AI assistant that orchestrates
    software engineering tasks across multiple workers.

    ## 1. Your Role

    You are a **coordinator**. Your job is to:
    - Help the user achieve their goal
    - Direct workers to research, implement and verify code changes
    - Synthesize results and communicate with the user
    - Answer questions directly when possible

    ## 2. Your Tools

    - **AgentTool** - Spawn a new worker
    - **SendMessageTool** - Continue an existing worker
    - **TaskStopTool** - Stop a running worker

    ## 3. Workers

    Workers execute tasks autonomously — especially research,
    implementation, or verification. Workers have access to
    standard tools, MCP tools, and project skills.

    ## 4. Task Workflow

    | Phase | Who | Purpose |
    |-------|-----|---------|
    | Research | Workers (parallel) | Investigate codebase |
    | Synthesis | You (coordinator) | Craft specs |
    | Implementation | Workers | Make changes |
    | Verification | Workers | Prove it works |

    ## 5. Concurrency

    **Parallelism is your superpower.** Launch independent workers
    concurrently whenever possible. Don't serialize work that can
    run simultaneously.

    ## 6. Writing Worker Prompts

    **Workers can't see your conversation.** Every prompt must be
    self-contained. After research completes, synthesize findings
    into a specific prompt.

    ### Always synthesize — your most important job

    When workers report research findings, **you must understand
    them before directing follow-up work**. Read the findings.
    Identify the approach. Then write a prompt that proves you
    understood by including specific file paths, line numbers,
    and exactly what to change.

    ### Good examples:
    - "Fix the null pointer in src/auth/validate.ts:42..."
    - "Create a new branch from main called 'fix/session-expiry'..."

    ### Bad examples:
    - "Based on your findings, fix the bug"  // 懒惰委托
    - "Fix the bug we discussed"             // 无上下文
    `
}
```

### 9.6.3 协调者模式的工具上下文

协调者模式会为worker工具注入额外的上下文：

```typescript
export function getCoordinatorUserContext(
  mcpClients: ReadonlyArray<{ name: string }>,
  scratchpadDir?: string,
): { [k: string]: string } {
  if (!isCoordinatorMode()) {
    return {}
  }

  const workerTools = Array.from(ASYNC_AGENT_ALLOWED_TOOLS)
    .filter(name => !INTERNAL_WORKER_TOOLS.has(name))
    .sort()
    .join(', ')

  let content = `Workers spawned via AgentTool have access to:
    ${workerTools}`

  if (mcpClients.length > 0) {
    const serverNames = mcpClients.map(c => c.name).join(', ')
    content += `\n\nWorkers also have access to MCP tools: ${serverNames}`
  }

  if (scratchpadDir && isScratchpadGateEnabled()) {
    content += `\n\nScratchpad directory: ${scratchpadDir}\nWorkers
      can read and write here without permission prompts.`
  }

  return { workerToolsContext: content }
}
```

### 9.6.4 会话模式匹配

当恢复会话时，系统会检查并匹配会话的模式：

```typescript
export function matchSessionMode(
  sessionMode: 'coordinator' | 'normal' | undefined,
): string | undefined {
  if (!sessionMode) {
    return undefined
  }

  const currentIsCoordinator = isCoordinatorMode()
  const sessionIsCoordinator = sessionMode === 'coordinator'

  if (currentIsCoordinator === sessionIsCoordinator) {
    return undefined
  }

  // 切换环境变量
  if (sessionIsCoordinator) {
    process.env.CLAUDE_CODE_COORDINATOR_MODE = '1'
  } else {
    delete process.env.CLAUDE_CODE_COORDINATOR_MODE
  }

  logEvent('tengu_coordinator_mode_switched', {
    to: sessionMode,
  })

  return sessionIsCoordinator
    ? 'Entered coordinator mode to match resumed session.'
    : 'Exited coordinator mode to match resumed session.'
}
```

---

## 9.7 小结与思考

### 9.7.1 设计模式总结

| 模式 | 实现 | 目的 |
|------|------|------|
| **Actor模型** | Agent + Mailbox | 异步消息传递，解耦通信双方 |
| **工作窃取** | claimTask + listTasks | 负载均衡，空闲Agent自发现任务 |
| **DAG依赖** | blocks/blockedBy | 任务依赖管理，确保执行顺序 |
| **协调者模式** | Coordinator Mode | 中心调度，简化并行任务编排 |
| **资源追踪** | registerTeamForSessionCleanup | 防止资源泄漏，确保清理 |

### 9.7.2 关键设计决策

1. **文件系统作为共享状态**：任务和邮箱都基于文件系统，天然支持跨进程通信和持久化。

2. **双向依赖维护**：blocks和blockedBy同时存储，O(1)查询，O(n)更新。

3. **文件锁保证原子性**：proper-lockfile库实现分布式锁，防止竞态条件。

4. **高水位标记防ID重用**：删除任务后，ID不再分配给新任务，避免混淆。

5. **团队=任务列表**：一个团队对应一个任务列表目录，简化权限管理和清理。

### 9.7.3 扩展思考

**问题1：如何处理循环依赖？**

当前实现没有显式检查循环依赖，因为任务的blockedBy是在创建时由Agent显式设置的。如果需要，可以在`blockTask`中添加环检测：

```typescript
function hasCycle(taskId: string, blocks: string[], visited: Set<string>): boolean {
  if (visited.has(taskId)) return true
  visited.add(taskId)

  for (const blockedId of blocks) {
    // 递归检查...
  }
  return false
}
```

**问题2：任务优先级如何实现？**

当前实现按ID顺序处理（FIFO）。如需优先级，可以在Task结构中添加`priority`字段，并在工作窃取时排序：

```typescript
const sortedTasks = tasks
  .filter(t => /* ... */)
  .sort((a, b) => (b.priority ?? 0) - (a.priority ?? 0))
```

**问题3：大规模场景下的性能优化？**

对于数百个任务，可以考虑：
- 使用SQLite替代文件系统存储
- 添加任务索引（按状态、所有者、优先级）
- 实现批量操作API

### 9.7.4 实践建议

1. **任务粒度适中**：太大难以并行，太小增加协调开销
2. **明确设置依赖**：充分利用blocks/blockedBy避免竞态
3. **合理使用广播**：只在必要时广播，避免消息爆炸
4. **及时释放任务**：完成后立即标记，释放阻塞关系
5. **善用协调者模式**：复杂任务先分解再分发，避免混乱

---

> "群体智能的精髓不在于个体多么聪明，而在于协作多么默契。" —— 感悟

通过本章的学习，我们看到了Claude Code如何通过简单的原语（Team、Task、SendMessage）构建出强大的多Agent协作系统。这种设计既保持了单个Agent的简洁性，又通过组合实现了复杂的群体行为。这正是"涌现"（Emergence）在工程领域的优雅体现。

下一章，我们将探讨**权限系统**——如何在强大的能力和安全的边界之间找到平衡。

---

*本章完*
