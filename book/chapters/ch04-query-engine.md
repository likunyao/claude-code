# 第4章：Query Engine —— AI 交互的心脏

> "复杂性是设计不佳的征兆。" —— 在构建与AI模型交互的核心引擎时，这一原则尤为重要。

Query Engine 是 Claude Code 中负责与 Anthropic API 交互的核心子系统。它不仅处理用户输入与模型响应的流转，还负责上下文窗口管理、流式响应解析、错误恢复、费用追踪等关键功能。本章将深入分析 `query.ts` 和 `QueryEngine.ts` 的设计思想，探讨函数式与类式设计的协作、消息流转模型、上下文压缩策略以及错误处理机制。

## 4.1 query.ts 与 QueryEngine.ts 的协作

在 Claude Code 的架构中，`query.ts` 和 `QueryEngine.ts` 承担了不同的职责。这种分离体现了**关注点分离**（Separation of Concerns）的设计原则。

### 4.1.1 职责划分

**QueryEngine.ts** 是会话级别的状态管理器，负责：

- 会话生命周期管理（构造、状态持久化、中断处理）
- 用户输入处理与斜杠命令分发
- 系统提示词构建与上下文注入
- SDK 消息格式转换与事件产生

```typescript
// QueryEngine 类的核心结构
export class QueryEngine {
  private config: QueryEngineConfig
  private mutableMessages: Message[]        // 会话消息历史
  private abortController: AbortController  // 中断控制
  private permissionDenials: SDKPermissionDenial[]
  private readFileState: FileStateCache     // 文件读取缓存

  async *submitMessage(prompt: string | ContentBlockParam[], options?) {
    // 1. 处理用户输入（包括斜杠命令）
    // 2. 构建系统提示词和上下文
    // 3. 调用 query() 执行查询循环
    // 4. 转换消息格式为 SDK 消息并 yield
  }
}
```

**query.ts** 提供的是查询循环的纯函数实现，负责：

- 单次查询的迭代控制（多轮对话的 while(true) 循环）
- 工具调用编排与结果聚合
- 流式响应处理与增量 yield
- 错误恢复与重试决策

```typescript
export async function* query(params: QueryParams): AsyncGenerator<...> {
  // 查询循环核心逻辑
  while (true) {
    // 1. 执行压缩
    // 2. 调用模型 API
    // 3. 执行工具
    // 4. 决定继续或终止
  }
}
```

### 4.1.2 函数式 vs 类式的设计选择

`query()` 采用**生成器函数**（AsyncGenerator）的形式，这是一种纯函数式的交互模式。每个 `yield` 都是一个纯数据转换的边界——输入是查询参数，输出是消息事件。这种设计的优势在于：

1. **可测试性**：函数无需构造复杂的类实例即可测试
2. **可组合性**：生成器可以通过 `yield*` 轻松组合
3. **状态透明**：所有跨迭代状态显式声明在 `State` 类型中

```typescript
// query.ts 中的状态管理
type State = {
  messages: Message[]
  toolUseContext: ToolUseContext
  autoCompactTracking: AutoCompactTrackingState | undefined
  maxOutputTokensRecoveryCount: number
  hasAttemptedReactiveCompact: boolean
  maxOutputTokensOverride: number | undefined
  pendingToolUseSummary: Promise<ToolUseSummaryMessage | null> | undefined
  stopHookActive: boolean | undefined
  turnCount: number
  transition: Continue | undefined
}

// 每次迭代开始时解构状态
let { toolUseContext } = state
const { messages, autoCompactTracking, ... } = state

// 继续时创建新的状态对象
state = {
  messages: [...messagesForQuery, ...assistantMessages, ...toolResults],
  toolUseContext: toolUseContextWithQueryTracking,
  autoCompactTracking: tracking,
  turnCount: nextTurnCount,
  transition: { reason: 'next_turn' },
  // ...
}
```

`QueryEngine` 采用**面向对象**的设计，适合管理会话级别的状态：

1. **封装性**：内部状态（如 `mutableMessages`）不暴露给外部
2. **生命周期**：构造、销毁与会话一一对应
3. **多接口**：提供 `interrupt()`、`getMessages()` 等辅助方法

```typescript
// QueryEngine 封装会话状态
export class QueryEngine {
  private mutableMessages: Message[]
  private abortController: AbortController
  private readFileState: FileStateCache

  // 外部只能通过 submitMessage 与会话交互
  async *submitMessage(prompt, options) { /* ... */ }

  // 提供受控的中断能力
  interrupt(): void {
    this.abortController.abort()
  }

  // 只读访问消息历史
  getMessages(): readonly Message[] {
    return this.mutableMessages
  }
}
```

### 4.1.3 分层调用关系

```
┌─────────────────────────────────────────────────────────┐
│                     用户/SDK 调用                        │
└─────────────────────────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────┐
│                   QueryEngine.submitMessage             │
│  • 用户输入处理                │
│  • 斜杠命令分发                                          │
│  • 系统提示词构建                                        │
└─────────────────────────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────┐
│                       query()                           │
│  • 查询循环控制            │
│  • 工具编排                │
│  • 错误恢复                                              │
└─────────────────────────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────┐
│                   claude.ts API 调用                    │
│  • queryModel() 流式调用                                 │
│  • withRetry() 重试包装                                  │
└─────────────────────────────────────────────────────────┘
```

这种分层设计使得：
- **QueryEngine** 可以被 SDK、REPL、测试框架等不同环境复用
- **query()** 可以被不同的查询入口（如 AgentTool）调用
- **API 层** 独立于业务逻辑，可以独立测试和替换

## 4.2 消息流转模型

### 4.2.1 消息类型体系

Claude Code 定义了一套严格的消息类型体系，用于表示对话中的各种事件：

```typescript
// 用户消息：用户输入或工具执行结果
type UserMessage = {
  type: 'user'
  message: { role: 'user', content: string | ContentBlock[] }
  uuid: string
  timestamp: string
  isMeta?: boolean          // 元消息（如系统注入）
  toolUseResult?: string    // 工具结果摘要
}

// 助手消息：模型响应
type AssistantMessage = {
  type: 'assistant'
  message: BetaMessage
  uuid: string
  timestamp: string
  requestId?: string        // API 请求 ID
  isApiErrorMessage?: boolean
  apiError?: string
}

// 附件消息：上下文信息注入
type AttachmentMessage = {
  type: 'attachment'
  attachment: {
    type: 'edited_text_file' | 'file_change' | 'queued_command' | ...
  }
  uuid: string
}

// 系统消息：控制信号
type SystemMessage = {
  type: 'system'
  subtype: 'compact_boundary' | 'api_error' | 'local_command' | ...
}

// 进度消息：流式渲染状态
type ProgressMessage = {
  type: 'progress'
  progress: {
    type: 'file_read' | 'command_execution' | ...
    status: 'running' | 'completed' | 'error'
  }
}

// 流事件：SSE 原始事件
type StreamEvent = {
  type: 'stream_event'
  event: BetaRawMessageStreamEvent
  ttftMs?: number  // Time to First Token
}
```

### 4.2.2 完整流转链路

```typescript
// 用户输入 → SDK 消息
for await (const message of query({ ... })) {
  switch (message.type) {
    case 'assistant':
      // 1. 推入可变消息数组
      this.mutableMessages.push(message)
      // 2. 转换为 SDK 格式 yield
      yield* normalizeMessage(message)
      break

    case 'user':
      // 工具结果或附件
      this.mutableMessages.push(message)
      yield* normalizeMessage(message)
      break

    case 'attachment':
      // 上下文附件（如编辑的文件）
      this.mutableMessages.push(message)
      // 记录到 transcript
      if (persistSession) {
        void recordTranscript([...messages, message])
      }
      break

    case 'system':
      if (message.subtype === 'compact_boundary') {
        // 压缩边界：释放历史消息内存
        const idx = this.mutableMessages.findIndex(m => m === message)
        if (idx > 0) {
          this.mutableMessages.splice(0, idx)
        }
      }
      break
  }
}
```

### 4.2.3 消息转换：内部 → API

在调用 API 之前，内部消息格式需要转换为 Anthropic API 格式：

```typescript
export function normalizeMessagesForAPI(
  messages: Message[],
  tools: Tools
): (UserMessage | AssistantMessage)[] {
  const result: (UserMessage | AssistantMessage)[] = []

  for (const message of messages) {
    switch (message.type) {
      case 'user':
      case 'assistant':
        // 直接添加（已是 UserMessage | AssistantMessage）
        result.push(message)
        break

      case 'attachment':
        // 转换为用户消息
        result.push(attachmentMessageToUserMessage(message, tools))
        break

      case 'progress':
        // 进度消息不发送到 API
        break

      case 'system':
        // 系统消息根据子类型处理
        if (message.subtype === 'compact_boundary') {
          // 压缩边界不发送
          continue
        }
        break
    }
  }

  return result
}
```

### 4.2.4 消息转换：内部 → SDK

SDK 调用者（如 Desktop、VS Code）期望不同的消息格式：

```typescript
// QueryEngine.submitMessage 中的转换
if (message.type === 'assistant') {
  // 推入本地消息数组
  this.mutableMessages.push(message)

  // 转换并 yield SDK 消息
  yield* normalizeMessage(message)
  // normalizeMessage 展开 stream_event，过滤进度消息
}

// normalizeMessage 的实现
export function* normalizeMessage(message: Message): Generator<SDKMessage> {
  switch (message.type) {
    case 'assistant':
      yield {
        type: 'assistant',
        message: message.message,
        session_id: getSessionId(),
        uuid: message.uuid,
        timestamp: message.timestamp,
        request_id: message.requestId,
      }
      break

    case 'progress':
      // 进度消息转换为 SDK 事件
      yield {
        type: 'progress',
        progress: message.progress,
        uuid: message.uuid,
      }
      break
  }
}
```

## 4.3 上下文窗口管理

上下文窗口管理是 Query Engine 的核心挑战之一。随着对话的进行，消息历史会不断增长，最终可能超出模型的上下文限制。Claude Code 采用了多层策略来应对这一问题。

### 4.3.1 Token 计数与估算

```typescript
export function tokenCountWithEstimation(messages: Message[]): number {
  let total = 0
  for (const message of messages) {
    // 优先使用 API 返回的精确计数
    if (message.type === 'assistant' || message.type === 'user') {
      const usage = message.message.usage
      if (usage) {
        // API 返回的是从开头到这条消息的累积计数
        total = Math.max(total, usage.input_tokens)
        continue
      }
    }
    // 降级到估算
    total += roughTokenCountEstimation(message)
  }
  return total
}

export function roughTokenCountEstimation(message: Message): number {
  if (message.type === 'user') {
    const content = message.message.content
    if (typeof content === 'string') {
      return Math.ceil(content.length / 4)  // 约 4 字符/token
    }
    // 数组内容需要遍历计算
    let total = 0
    for (const block of content) {
      if (block.type === 'text') total += Math.ceil(block.text.length / 4)
      else if (block.type === 'image') total += 850  // 图像固定 850 tokens
      else if (block.type === 'tool_result') total += roughTokenCountEstimation(block.content)
    }
    return total
  }
  // ... 其他消息类型
}
```

### 4.3.2 自动压缩（Auto-Compact）

当上下文接近限制时，系统会自动触发压缩：

```typescript
// query.ts 中的自动压缩检查
const { compactionResult, consecutiveFailures } = await deps.autocompact(
  messagesForQuery,
  toolUseContext,
  {
    systemPrompt,
    userContext,
    systemContext,
    toolUseContext,
    forkContextMessages: messagesForQuery,
  },
  querySource,
  tracking,
  snipTokensFreed,
)

if (compactionResult) {
  const {
    preCompactTokenCount,
    postCompactTokenCount,
    truePostCompactTokenCount,
    compactionUsage,
  } = compactionResult

  // 重置压缩跟踪状态
  tracking = {
    compacted: true,
    turnId: deps.uuid(),
    turnCounter: 0,
    consecutiveFailures: 0,
  }

  // 构建压缩后的消息
  const postCompactMessages = buildPostCompactMessages(compactionResult)

  // yield 压缩边界标记
  for (const message of postCompactMessages) {
    yield message
  }

  messagesForQuery = postCompactMessages
}
```

压缩的实现涉及以下步骤：

1. **调用 Forked Agent**：使用独立的子对话执行压缩，避免污染主对话的提示词缓存
2. **生成摘要**：将旧消息摘要为简洁的文本描述
3. **保留上下文**：通过附件机制保留最近的文件读取结果、Plan 文件等关键信息
4. **注入边界标记**：添加 `compact_boundary` 系统消息，标记压缩点

```typescript
// compact.ts 中的压缩摘要生成
const compactPrompt = getCompactPrompt(customInstructions)
const summaryRequest = createUserMessage({ content: compactPrompt })

// 使用 Forked Agent 执行压缩（复用主对话的提示词缓存）
const result = await runForkedAgent({
  promptMessages: [summaryRequest],
  cacheSafeParams,
  canUseTool: createCompactCanUseTool(),
  querySource: 'compact',
  forkLabel: 'compact',
  maxTurns: 1,
  skipCacheWrite: true,
})

// 解压缩后重新注入上下文
const [fileAttachments, asyncAgentAttachments] = await Promise.all([
  createPostCompactFileAttachments(
    preCompactReadFileState,
    context,
    POST_COMPACT_MAX_FILES_TO_RESTORE,
  ),
  createAsyncAgentAttachmentsIfNeeded(context),
])

// 添加边界标记
const boundaryMarker = createCompactBoundaryMessage(
  isAutoCompact ? 'auto' : 'manual',
  preCompactTokenCount ?? 0,
  messages.at(-1)?.uuid,
)
```

### 4.3.3 Micro-Compact：工具结果压缩

除了全量压缩外，Claude Code 还实现了**微压缩**（Micro-Compact），专门针对工具结果：

```typescript
// microcompact.ts
export async function microcompact(
  messages: Message[],
  toolUseContext: ToolUseContext,
  querySource: QuerySource
): Promise<MicrocompactResult> {
  // 应用工具结果预算限制
  messages = await applyToolResultBudget(
    messages,
    toolUseContext.contentReplacementState,
    persistReplacements
      ? records => void recordContentReplacement(records, toolUseContext.agentId)
      : undefined,
    new Set(
      toolUseContext.options.tools
        .filter(t => !Number.isFinite(t.maxResultSizeChars))
        .map(t => t.name),
    ),
  )

  // 针对大工具结果的缓存编辑压缩
  // feature-gated: CACHED_MICROCOMPACT
  const consumedCacheEdits = consumePendingCacheEdits()

  return { messages, compactionInfo: { pendingCacheEdits: consumedCacheEdits } }
}
```

微压缩的特点：
- **粒度更细**：只压缩工具结果，不影响对话正文
- **缓存友好**：使用 `cache_edits` API 精确删除已缓存的内容
- **透明性**：对模型不可见，不影响对话连续性

### 4.3.4 响应式压缩（Reactive Compact）

当 API 返回 `prompt_too_long` 错误时，系统会触发**响应式压缩**：

```typescript
// query.ts 中的响应式压缩处理
if (isWithheld413) {
  // 首先尝试排空已暂存的 Context Collapse
  if (feature('CONTEXT_COLLAPSE') && contextCollapse) {
    const drained = contextCollapse.recoverFromOverflow(
      messagesForQuery,
      querySource,
    )
    if (drained.committed > 0) {
      state = {
        messages: drained.messages,
        transition: { reason: 'collapse_drain_retry' },
        // ...
      }
      continue
    }
  }
}

// 如果 Collapse 不足以恢复，执行 Reactive Compact
if ((isWithheld413 || isWithheldMedia) && reactiveCompact) {
  const compacted = await reactiveCompact.tryReactiveCompact({
    hasAttempted: hasAttemptedReactiveCompact,
    querySource,
    aborted: toolUseContext.abortController.signal.aborted,
    messages: messagesForQuery,
    cacheSafeParams: { /* ... */ },
  })

  if (compacted) {
    // 成功压缩，重试
    const postCompactMessages = buildPostCompactMessages(compacted)
    state = {
      messages: postCompactMessages,
      transition: { reason: 'reactive_compact_retry' },
      // ...
    }
    continue
  }

  // 无法恢复，surface 错误
  yield lastMessage
  return { reason: isWithheldMedia ? 'image_error' : 'prompt_too_long' }
}
```

### 4.3.5 上下文压缩的优先级

Claude Code 采用了分层的上下文管理策略：

| 策略 | 触发条件 | 范围 | 保留内容 |
|------|----------|------|----------|
| Tool Budget | 每个 tool_result | 单个结果 | 截断到 maxResultSizeChars |
| Micro-Compact | 每次查询前 | 最近工具结果 | 使用缓存编辑删除 |
| Snip Compact | 配置启用 | 消息头尾 | 保留关键上下文 |
| Auto-Compact | Token 阈值 | 整个历史 | 摘要 + 附件 |
| Reactive Compact | 413 错误 | 整个历史 | 摘要 + 附件 |
| Context Collapse | 配置启用 | 历史消息 | 投影式压缩视图 |

### 4.3.6 Context Collapse：投影式压缩

**Context Collapse** 是一种实验性的上下文管理策略，它与传统的压缩机制有本质区别：

```typescript
// contextCollapse/index.ts
export async function applyCollapsesIfNeeded(
  messages: Message[],
  toolUseContext: ToolUseContext,
  querySource: QuerySource,
): Promise<{ messages: Message[] }> {
  if (!isContextCollapseEnabled()) {
    return { messages }
  }

  // 获取已暂存的 collapses
  const store = getCollapseStore()
  const { committed, staged } = await store.drain()

  // 应用投影式压缩视图
  const collapsedView = projectView(messages, {
    committed,
    staged,
  })

  return { messages: collapsedView }
}
```

Context Collapse 的特点：
- **投影式**：不修改原始消息数组，而是通过投影生成压缩视图
- **增量式**：每次添加新的 collapse 记录，逐步压缩历史
- **可恢复**：可以通过 `recoverFromOverflow` 排空已暂存的 collapses

```typescript
// recoverFromOverflow 实现
export function recoverFromOverflow(
  messages: Message[],
  querySource: QuerySource,
): { messages: Message[]; committed: number } {
  const store = getCollapseStore()
  const { committed, staged } = store.drainAll()

  if (committed === 0) {
    return { messages, committed: 0 }
  }

  // 应用所有 collapses
  const collapsedView = projectView(messages, {
    committed,
    staged: [],
  })

  logEvent('tengu_context_collapse_drain', {
    committed,
    staged: staged.length,
    querySource,
  })

  return { messages: collapsedView, committed }
}
```

### 4.3.7 提示词缓存优化

Claude Code 通过精细的提示词缓存管理来降低 API 成本：

```typescript
// addCacheBreakpoints 实现
export function addCacheBreakpoints(
  messages: (UserMessage | AssistantMessage)[],
  enablePromptCaching: boolean,
  querySource?: QuerySource,
  useCachedMC = false,
  newCacheEdits?: CachedMCEditsBlock | null,
  pinnedEdits?: CachedMCPinnedEdits[],
  skipCacheWrite = false,
): MessageParam[] {
  // 只有一个消息级 cache_control 标记
  // fire-and-forget 模式：移动到倒数第二条消息
  const markerIndex = skipCacheWrite ? messages.length - 2 : messages.length - 1

  const result = messages.map((msg, index) => {
    const addCache = index === markerIndex
    if (msg.type === 'user') {
      return userMessageToMessageParam(msg, addCache, enablePromptCaching, querySource)
    }
    return assistantMessageToMessageParam(msg, addCache, enablePromptCaching, querySource)
  })

  // 缓存编辑功能（feature-gated）
  if (useCachedMC) {
    // 去重已删除的缓存引用
    const seenDeleteRefs = new Set<string>()

    // 重新插入已固定的缓存编辑
    for (const pinned of pinnedEdits ?? []) {
      const msg = result[pinned.userMessageIndex]
      if (msg && msg.role === 'user') {
        if (!Array.isArray(msg.content)) {
          msg.content = [{ type: 'text', text: msg.content as string }]
        }
        const dedupedBlock = deduplicateEdits(pinned.block)
        if (dedupedBlock.edits.length > 0) {
          insertBlockAfterToolResults(msg.content, dedupedBlock)
        }
      }
    }

    // 插入新的缓存编辑
    if (newCacheEdits && result.length > 0) {
      const dedupedNewEdits = deduplicateEdits(newCacheEdits)
      if (dedupedNewEdits.edits.length > 0) {
        for (let i = result.length - 1; i >= 0; i--) {
          const msg = result[i]
          if (msg && msg.role === 'user') {
            if (!Array.isArray(msg.content)) {
              msg.content = [{ type: 'text', text: msg.content as string }]
            }
            insertBlockAfterToolResults(msg.content, dedupedNewEdits)
            pinCacheEdits(i, newCacheEdits)
            break
          }
        }
      }
    }
  }

  return result
}
```

提示词缓存的设计要点：
1. **单一标记**：每次请求只有一个 `cache_control` 标记，避免缓存碎片化
2. **缓存编辑**：使用 `cache_edits` 精确删除已缓存的内容，避免重复计费
3. **会话稳定**：使用 latched 策略，避免中途切换导致缓存失效
4. **1H TTL**：对符合条件的用户启用 1 小时缓存 TTL

## 4.4 流式响应处理

Claude Code 使用 Server-Sent Events (SSE) 流式接收模型响应，实现增量渲染和实时反馈。

### 4.4.1 SSE 解析循环

```typescript
// claude.ts 中的流式处理
for await (const part of stream) {
  resetStreamIdleTimer()  // 空闲超时看门狗
  const now = Date.now()

  // 检测流式停顿
  if (lastEventTime !== null) {
    const timeSinceLastEvent = now - lastEventTime
    if (timeSinceLastEvent > STALL_THRESHOLD_MS) {
      stallCount++
      totalStallTime += timeSinceLastEvent
      logEvent('tengu_streaming_stall', {
        stall_duration_ms: timeSinceLastEvent,
        stall_count: stallCount,
      })
    }
  }
  lastEventTime = now

  switch (part.type) {
    case 'message_start': {
      partialMessage = part.message
      ttftMs = Date.now() - start  // Time to First Token
      usage = updateUsage(usage, part.message?.usage)
      break
    }

    case 'content_block_start': {
      const { index, content_block } = part
      switch (content_block.type) {
        case 'tool_use':
          contentBlocks[index] = { ...content_block, input: '' }
          break
        case 'text':
          contentBlocks[index] = { ...content_block, text: '' }
          break
        case 'thinking':
          contentBlocks[index] = { ...content_block, thinking: '', signature: '' }
          break
        default:
          contentBlocks[index] = { ...content_block }
      }
      break
    }

    case 'content_block_delta': {
      const { index, delta } = part
      const contentBlock = contentBlocks[index]

      switch (delta.type) {
        case 'text_delta':
          contentBlock.text += delta.text
          break
        case 'input_json_delta':
          contentBlock.input += delta.partial_json
          break
        case 'thinking_delta':
          contentBlock.thinking += delta.thinking
          break
        case 'signature_delta':
          contentBlock.signature = delta.signature
          break
      }
      break
    }

    case 'content_block_stop': {
      // 完成一个内容块，yield AssistantMessage
      const m: AssistantMessage = {
        message: {
          ...partialMessage,
          content: normalizeContentFromAPI([contentBlock], tools, agentId),
        },
        requestId: streamRequestId,
        type: 'assistant',
        uuid: randomUUID(),
        timestamp: new Date().toISOString(),
      }
      newMessages.push(m)
      yield m
      break
    }

    case 'message_delta': {
      usage = updateUsage(usage, part.usage)
      stopReason = part.delta.stop_reason

      // 更新最后一条消息的 usage 和 stop_reason
      const lastMsg = newMessages.at(-1)
      if (lastMsg) {
        lastMsg.message.usage = usage
        lastMsg.message.stop_reason = stopReason
      }

      // 计算并累积费用
      const costUSDForPart = calculateUSDCost(resolvedModel, usage)
      costUSD += addToTotalSessionCost(costUSDForPart, usage, model)

      // 处理特殊停止原因
      if (stopReason === 'max_tokens') {
        yield createAssistantAPIErrorMessage({
          content: `${API_ERROR_MESSAGE_PREFIX}: Claude's response exceeded the ${maxOutputTokens} output token maximum.`,
          apiError: 'max_output_tokens',
        })
      }
      break
    }

    case 'message_stop':
      break
  }

  // yield 流事件给调用者
  yield {
    type: 'stream_event',
    event: part,
    ...(part.type === 'message_start' ? { ttftMs } : undefined),
  }
}
```

### 4.4.2 增量渲染与用户体验

流式响应不仅是为了减少首字节延迟（TTFB），更是为了提供实时反馈：

```typescript
// StreamingToolExecutor：边接收边执行工具
if (streamingToolExecutor) {
  for await (const result of streamingToolExecutor.getCompletedResults()) {
    if (result.message) {
      yield result.message
      toolResults.push(...normalizeMessagesForAPI([result.message], tools))
    }
  }
}
```

这种设计的优势：
- **并行性**：工具执行与模型响应并行进行
- **早期反馈**：用户可以立即看到部分结果
- **流畅性**：避免长时间的等待空白

### 4.4.3 空闲超时看门狗

为了防止流式连接静默挂起，Claude Code 实现了空闲超时机制：

```typescript
const STREAM_IDLE_TIMEOUT_MS = parseInt(process.env.CLAUDE_STREAM_IDLE_TIMEOUT_MS || '') || 90_000
const STREAM_IDLE_WARNING_MS = STREAM_IDLE_TIMEOUT_MS / 2

let streamIdleAborted = false
let streamIdleTimer: ReturnType<typeof setTimeout> | null = null

function resetStreamIdleTimer(): void {
  if (streamIdleTimer !== null) {
    clearTimeout(streamIdleTimer)
  }
  streamIdleTimer = setTimeout(() => {
    streamIdleAborted = true
    logForDebugging('Streaming idle timeout: no chunks received')
    releaseStreamResources()  // 释放原生资源
  }, STREAM_IDLE_TIMEOUT_MS)
}

// 在循环开始时重置
resetStreamIdleTimer()

// 每次 chunk 到达时重置
for await (const part of stream) {
  resetStreamIdleTimer()
  // ...
}
```

### 4.4.4 流式工具执行（Streaming Tool Execution）

**Streaming Tool Execution** 是 Claude Code 的一个创新特性，允许在接收模型响应的同时执行工具：

```typescript
// StreamingToolExecutor 的核心实现
export class StreamingToolExecutor {
  private tools: Tools
  private canUseTool: CanUseToolFn
  private toolUseContext: ToolUseContext
  private pendingTools: Map<string, PendingTool> = new Map()
  private completedResults: ToolResult[] = []

  addTool(toolUse: ToolUseBlock, assistantMessage: AssistantMessage): void {
    const tool = findToolByName(this.tools, toolUse.name)
    if (!tool) return

    this.pendingTools.set(toolUse.id, {
      toolUse,
      tool,
      assistantMessage,
      startedAt: Date.now(),
    })
  }

  async *getCompletedResults(): AsyncGenerator<ToolResult> {
    while (this.pendingTools.size > 0) {
      // 检查是否有工具完成执行
      for (const [id, pending] of this.pendingTools) {
        if (this.isToolComplete(pending)) {
          this.pendingTools.delete(id)
          const result = await this.executeTool(pending)
          this.completedResults.push(result)
          yield result
        }
      }

      // 短暂等待避免空转
      await sleep(10)
    }
  }

  discard(): void {
    // 丢弃所有待处理的工具
    this.pendingTools.clear()
  }

  getRemainingResults(): AsyncGenerator<ToolResult> {
    // 获取剩余工具的结果
    return this.executeAllRemaining()
  }
}
```

流式工具执行的优势：

1. **减少延迟**：工具执行与模型响应并行，总耗时 = max(模型响应, 工具执行)
2. **早期反馈**：用户可以立即看到工具执行结果，无需等待模型完全响应
3. **更好的用户体验**：避免长时间的空白等待

```typescript
// query.ts 中的流式工具执行集成
if (streamingToolExecutor && !toolUseContext.abortController.signal.aborted) {
  for (const toolBlock of msgToolUseBlocks) {
    streamingToolExecutor.addTool(toolBlock, message)
  }
}

// 获取已完成的工具结果
for await (const result of streamingToolExecutor.getCompletedResults()) {
  if (result.message) {
    yield result.message
    toolResults.push(...normalizeMessagesForAPI([result.message], tools))
  }
}
```

### 4.4.5 流式停顿检测与诊断

Claude Code 还实现了流式停顿检测功能，用于诊断网络或 API 问题：

```typescript
const STALL_THRESHOLD_MS = 30_000  // 30 秒

for await (const part of stream) {
  const now = Date.now()

  // 检测流式停顿（仅在第一个 chunk 之后）
  if (lastEventTime !== null) {
    const timeSinceLastEvent = now - lastEventTime
    if (timeSinceLastEvent > STALL_THRESHOLD_MS) {
      stallCount++
      totalStallTime += timeSinceLastEvent
      logEvent('tengu_streaming_stall', {
        stall_duration_ms: timeSinceLastEvent,
        stall_count: stallCount,
        total_stall_time_ms: totalStallTime,
        event_type: part.type,
        model: model,
        request_id: streamRequestId,
      })
    }
  }
  lastEventTime = now
}
```

### 4.4.6 原生资源释放

流式响应涉及原生资源（如 TLS socket、缓冲区），必须正确释放以避免内存泄漏：

```typescript
function releaseStreamResources(): void {
  cleanupStream(stream)
  stream = undefined
  if (streamResponse) {
    streamResponse.body?.cancel().catch(() => {})
    streamResponse = undefined
  }
}

// 在所有退出路径上释放
try {
  // 流式处理逻辑
} finally {
  releaseStreamResources()
}
```

为什么这很重要？
- **原生泄漏**：V8 堆外的 TLS socket 缓冲区不受 GC 管理，必须手动释放
- **会话存活**：即使生成器被 GC，原生资源可能仍然占用内存
- **累积效应**：长会话中多次流式调用会累积大量未释放的资源

## 4.5 错误恢复与重试机制

Claude Code 面临的错误类型多样：网络超时、API 过载、上下文溢出、模型限制等。系统通过 `categorizeRetryableAPIError` 和多层重试策略来保证鲁棒性。

### 4.5.1 错误分类

```typescript
export function categorizeRetryableAPIError(
  error: APIError,
): SDKAssistantMessageError {
  // 529 服务器过载
  if (
    error.status === 529 ||
    error.message?.includes('"type":"overloaded_error"')
  ) {
    return 'rate_limit'
  }

  // 429 速率限制
  if (error.status === 429) {
    return 'rate_limit'
  }

  // 401/403 认证错误
  if (error.status === 401 || error.status === 403) {
    return 'authentication_failed'
  }

  // 408+ 服务器错误
  if (error.status !== undefined && error.status >= 408) {
    return 'server_error'
  }

  return 'unknown'
}
```

### 4.5.2 withRetry：指数退避重试

```typescript
export async function withRetry<T>(
  getClient: () => Promise Anthropic>,
  fn: (client: Anthropic, attempt: number, context: RetryContext) => Promise<T>,
  options: RetryOptions,
): Promise<T> {
  let consecutive529Errors = 0

  while (true) {
    const client = await getClient()

    try {
      return await fn(client, attempt + 1, {
        model: options.model,
        maxTokensOverride: undefined,
        fastMode: options.fastMode ?? false,
      })
    } catch (error) {
      // 用户中断不重试
      if (error instanceof APIUserAbortError) {
        throw error
      }

      // 模型回退信号
      if (error instanceof FallbackTriggeredError) {
        throw error  // 上层处理模型切换
      }

      // 529 过载：连续 3 次后触发模型回退
      if (is529Error(error)) {
        consecutive529Errors++

        if (consecutive529Errors >= 3 && options.fallbackModel) {
          logEvent('tengu_model_fallback_triggered', {
            reason: 'consecutive_529',
            original_model: options.model,
            fallback_model: options.fallbackModel,
          })
          throw new FallbackTriggeredError(options.model, options.fallbackModel)
        }
      } else {
        consecutive529Errors = 0
      }

      // 检查是否可重试
      const errorType = classifyAPIError(error)
      if (errorType === 'unknown' || errorType === 'authentication_failed') {
        throw new CannotRetryError(error, retryContext)
      }

      // 等待后重试
      const delay = getRetryDelay(attempt)
      await sleep(delay, signal, { abortError: () => new APIUserAbortError() })
      attempt++
    }
  }
}

// 指数退避 + 抖动
function getRetryDelay(attempt: number): number {
  const baseDelay = 1000
  const maxDelay = 30000
  const exponentialDelay = baseDelay * Math.pow(2, attempt)
  const jitter = Math.random() * 1000
  return Math.min(exponentialDelay + jitter, maxDelay)
}
```

### 4.5.3 max_output_tokens 恢复

当模型达到输出令牌限制时，Claude Code 会自动恢复：

```typescript
const MAX_OUTPUT_TOKENS_RECOVERY_LIMIT = 3

// 检查是否是 max_output_tokens 错误
if (isWithheldMaxOutputTokens(lastMessage)) {
  // 首次命中：升级到 64k 限制
  const capEnabled = getFeatureValue_CACHED_MAY_BE_STALE(
    'tengu_otk_slot_v1',
    false,
  )
  if (
    capEnabled &&
    maxOutputTokensOverride === undefined &&
    !process.env.CLAUDE_CODE_MAX_OUTPUT_TOKENS
  ) {
    logEvent('tengu_max_tokens_escalate', {
      escalatedTo: ESCALATED_MAX_TOKENS,
    })
    state = {
      messages: messagesForQuery,
      maxOutputTokensOverride: ESCALATED_MAX_TOKENS,
      transition: { reason: 'max_output_tokens_escalate' },
    }
    continue
  }

  // 多轮恢复：注入恢复消息继续
  if (maxOutputTokensRecoveryCount < MAX_OUTPUT_TOKENS_RECOVERY_LIMIT) {
    const recoveryMessage = createUserMessage({
      content: `Output token limit hit. Resume directly — no apology, no recap of what you were doing. Pick up mid-thought if that is where the cut happened.`,
      isMeta: true,
    })

    state = {
      messages: [
        ...messagesForQuery,
        ...assistantMessages,
        recoveryMessage,
      ],
      maxOutputTokensRecoveryCount: maxOutputTokensRecoveryCount + 1,
      transition: {
        reason: 'max_output_tokens_recovery',
        attempt: maxOutputTokensRecoveryCount + 1,
      },
    }
    continue
  }

  // 恢复耗尽，surface 错误
  yield lastMessage
}
```

### 4.5.4 流式到非流式的回退

当流式请求失败时，系统会自动回退到非流式模式：

```typescript
try {
  // 尝试流式请求
  for await (const part of stream) {
    // ...
  }
} catch (streamingError) {
  // 禁用回退检查
  const disableFallback = isEnvTruthy('CLAUUDE_CODE_DISABLE_NONSTREAMING_FALLBACK') ||
    getFeatureValue_CACHED_MAY_BE_STALE('tengu_disable_streaming_to_non_streaming_fallback', false)

  if (disableFallback) {
    throw streamingError
  }

  logEvent('tengu_streaming_fallback_to_non_streaming', {
    model: options.model,
    error: streamingError instanceof Error ? streamingError.name : String(streamingError),
    fallback_cause: streamIdleAborted ? 'watchdog' : 'other',
  })

  // 回退到非流式请求
  const result = yield* executeNonStreamingRequest(
    { model: options.model, source: options.querySource },
    { model: options.model, fallbackModel: options.fallbackModel, /* ... */ },
    paramsFromContext,
    (attempt, _startTime, tokens) => {
      attemptNumber = attempt
      maxOutputTokens = tokens
    },
    params => captureAPIRequest(params, options.querySource),
    streamRequestId,
  )

  // 构造消息并 yield
  const m: AssistantMessage = {
    message: {
      ...result,
      content: normalizeContentFromAPI(result.content, tools, options.agentId),
    },
    requestId: streamRequestId,
    type: 'assistant',
    uuid: randomUUID(),
    timestamp: new Date().toISOString(),
  }
  yield m
}
```

### 4.5.5 模型回退（Model Fallback）

当连续多次遇到 529 过载错误时，系统会自动回退到备用模型：

```typescript
// withRetry 中的 529 处理
if (is529Error(error)) {
  consecutive529Errors++

  // 连续 3 次 529 后触发模型回退
  if (consecutive529Errors >= 3 && options.fallbackModel) {
    logEvent('tengu_model_fallback_triggered', {
      reason: 'consecutive_529',
      original_model: options.model,
      fallback_model: options.fallbackModel,
    })
    throw new FallbackTriggeredError(options.model, options.fallbackModel)
  }
} else {
  consecutive529Errors = 0
}
```

`FallbackTriggeredError` 是一个特殊的错误类型，它会被 `query.ts` 捕获并处理：

```typescript
// query.ts 中的模型回退处理
if (error instanceof FallbackTriggeredError && fallbackModel) {
  currentModel = fallbackModel
  attemptWithFallback = true

  // 清空助手消息
  yield* yieldMissingToolResultBlocks(
    assistantMessages,
    'Model fallback triggered',
  )
  assistantMessages.length = 0
  toolResults.length = 0
  toolUseBlocks.length = 0

  // 丢弃流式工具执行器
  if (streamingToolExecutor) {
    streamingToolExecutor.discard()
    streamingToolExecutor = new StreamingToolExecutor(
      toolUseContext.options.tools,
      canUseTool,
      toolUseContext,
    )
  }

  // 更新工具使用上下文
  toolUseContext.options.mainLoopModel = fallbackModel

  // 移除签名块（不同模型的签名不兼容）
  if (process.env.USER_TYPE === 'ant') {
    messagesForQuery = stripSignatureBlocks(messagesForQuery)
  }

  // 记录回退事件
  logEvent('tengu_model_fallback_triggered', {
    original_model: error.originalModel,
    fallback_model: error.fallbackModel,
  })

  // 通知用户
  yield createSystemMessage(
    `Switched to ${renderModelName(error.fallbackModel)} due to high demand for ${renderModelName(error.originalModel)}`,
    'warning',
  )

  continue
}
```

### 4.5.6 错误 withhold 机制

某些错误（如 prompt_too_long、max_output_tokens）需要在确认无法恢复后才向用户展示。Claude Code 使用 **withhold 机制** 来延迟这些错误：

```typescript
// 错误 withhold 检查
let withheld = false
if (feature('CONTEXT_COLLAPSE')) {
  if (contextCollapse?.isWithheldPromptTooLong(message, isPromptTooLongMessage, querySource)) {
    withheld = true
  }
}
if (reactiveCompact?.isWithheldPromptTooLong(message)) {
  withheld = true
}
if (mediaRecoveryEnabled && reactiveCompact?.isWithheldMediaSizeError(message)) {
  withheld = true
}
if (isWithheldMaxOutputTokens(message)) {
  withheld = true
}

// 只有没有被 withhold 的消息才 yield
if (!withheld) {
  yield yieldMessage
}
```

 withhold 机制的好处：
1. **避免误导**：在确认无法恢复之前，不让用户看到错误
2. **自动恢复**：系统可以静默尝试恢复（如压缩、重试）
3. **更好的体验**：成功恢复时用户完全感知不到错误

### 4.5.7 分类错误的完整谱系

`classifyAPIError` 函数将所有可能的 API 错误分类为不同的类型：

```typescript
export function classifyAPIError(error: unknown): string {
  // 用户中断
  if (error instanceof Error && error.message === 'Request was aborted.') {
    return 'aborted'
  }

  // 超时
  if (error instanceof APIConnectionTimeoutError ||
      (error instanceof APIConnectionError && error.message.toLowerCase().includes('timeout'))) {
    return 'api_timeout'
  }

  // 重复 529
  if (error instanceof Error && error.message.includes(REPEATED_529_ERROR_MESSAGE)) {
    return 'repeated_529'
  }

  // 容量关闭开关
  if (error instanceof Error && error.message.includes(CUSTOM_OFF_SWITCH_MESSAGE)) {
    return 'capacity_off_switch'
  }

  // 速率限制
  if (error instanceof APIError && error.status === 429) {
    return 'rate_limit'
  }

  // 服务器过载
  if (error instanceof APIError && (error.status === 529 || error.message?.includes('"type":"overloaded_error"'))) {
    return 'server_overload'
  }

  // Prompt 太长
  if (error instanceof Error && error.message.toLowerCase().includes(PROMPT_TOO_LONG_ERROR_MESSAGE.toLowerCase())) {
    return 'prompt_too_long'
  }

  // PDF 错误
  if (/maximum of \d+ PDF pages/.test(error.message)) {
    return 'pdf_too_large'
  }

  // 图像尺寸错误
  if (error instanceof APIError && error.status === 400 && error.message.includes('image exceeds')) {
    return 'image_too_large'
  }

  // 工具使用错误
  if (error instanceof APIError && error.status === 400 &&
      error.message.includes('`tool_use` ids were found without `tool_result`')) {
    return 'tool_use_mismatch'
  }

  // 认证错误
  if (error instanceof Error && error.message.toLowerCase().includes('x-api-key')) {
    return 'invalid_api_key'
  }

  // ... 更多分类

  return 'unknown'
}
```

## 4.6 费用追踪

Claude Code 提供实时的费用追踪功能，帮助用户了解 API 调用的成本。

### 4.6.1 成本计算

```typescript
// cost-tracker.ts
export function addToTotalSessionCost(
  cost: number,
  usage: Usage,
  model: string,
): number {
  // 累积模型级别的使用量
  const modelUsage = getUsageForModel(model) ?? {
    inputTokens: 0,
    outputTokens: 0,
    cacheReadInputTokens: 0,
    cacheCreationInputTokens: 0,
    webSearchRequests: 0,
    costUSD: 0,
  }

  modelUsage.inputTokens += usage.input_tokens
  modelUsage.outputTokens += usage.output_tokens
  modelUsage.cacheReadInputTokens += usage.cache_read_input_tokens ?? 0
  modelUsage.cacheCreationInputTokens += usage.cache_creation_input_tokens ?? 0
  modelUsage.webSearchRequests += usage.server_tool_use?.web_search_requests ?? 0
  modelUsage.costUSD += cost

  // 更新全局状态
  addToTotalCostState(cost, modelUsage, model)

  // 记录到 Statsig 指标
  getCostCounter()?.add(cost, { model })
  getTokenCounter()?.add(usage.input_tokens, { model, type: 'input' })
  getTokenCounter()?.add(usage.output_tokens, { model, type: 'output' })
  getTokenCounter()?.add(usage.cache_read_input_tokens ?? 0, { model, type: 'cacheRead' })
  getTokenCounter()?.add(usage.cache_creation_input_tokens ?? 0, { model, type: 'cacheCreation' })

  // 处理 Advisor 工具的额外费用
  let totalCost = cost
  for (const advisorUsage of getAdvisorUsage(usage)) {
    const advisorCost = calculateUSDCost(advisorUsage.model, advisorUsage)
    totalCost += addToTotalSessionCost(
      advisorCost,
      advisorUsage,
      advisorUsage.model,
    )
  }

  return totalCost
}
```

### 4.6.2 费用显示

```typescript
export function formatTotalCost(): string {
  const costDisplay = formatCost(getTotalCostUSD()) +
    (hasUnknownModelCost()
      ? ' (costs may be inaccurate due to usage of unknown models)'
      : '')

  const modelUsageDisplay = formatModelUsage()

  return chalk.dim(
    `Total cost:            ${costDisplay}\n` +
    `Total duration (API):  ${formatDuration(getTotalAPIDuration())}
Total duration (wall): ${formatDuration(getTotalDuration())}
Total code changes:    ${getTotalLinesAdded()} lines added, ${getTotalLinesRemoved()} lines removed
${modelUsageDisplay}`,
  )
}

function formatModelUsage(): string {
  const modelUsageMap = getModelUsage()
  if (Object.keys(modelUsageMap).length === 0) {
    return 'Usage:                 0 input, 0 output, 0 cache read, 0 cache write'
  }

  // 按短名称聚合
  const usageByShortName: { [shortName: string]: ModelUsage } = {}
  for (const [model, usage] of Object.entries(modelUsageMap)) {
    const shortName = getCanonicalName(model)
    if (!usageByShortName[shortName]) {
      usageByShortName[shortName] = { /* 初始化 */ }
    }
    const accumulated = usageByShortName[shortName]
    accumulated.inputTokens += usage.inputTokens
    accumulated.outputTokens += usage.outputTokens
    accumulated.cacheReadInputTokens += usage.cacheReadInputTokens
    accumulated.cacheCreationInputTokens += usage.cacheCreationInputTokens
    accumulated.webSearchRequests += usage.webSearchRequests
    accumulated.costUSD += usage.costUSD
  }

  let result = 'Usage by model:'
  for (const [shortName, usage] of Object.entries(usageByShortName)) {
    const usageString =
      `  ${formatNumber(usage.inputTokens)} input, ` +
      `${formatNumber(usage.outputTokens)} output, ` +
      `${formatNumber(usage.cacheReadInputTokens)} cache read, ` +
      `${formatNumber(usage.cacheCreationInputTokens)} cache write` +
      (usage.webSearchRequests > 0
        ? `, ${formatNumber(usage.webSearchRequests)} web search`
        : '') +
      ` (${formatCost(usage.costUSD)})`
    result += `\n` + `${shortName}:`.padStart(21) + usageString
  }
  return result
}
```

### 4.6.3 会话持久化

费用状态会在会话间持久化：

```typescript
export function saveCurrentSessionCosts(fpsMetrics?: FpsMetrics): void {
  saveCurrentProjectConfig(current => ({
    ...current,
    lastCost: getTotalCostUSD(),
    lastAPIDuration: getTotalAPIDuration(),
    lastAPIDurationWithoutRetries: getTotalAPIDurationWithoutRetries(),
    lastToolDuration: getTotalToolDuration(),
    lastDuration: getTotalDuration(),
    lastLinesAdded: getTotalLinesAdded(),
    lastLinesRemoved: getTotalLinesRemoved(),
    lastTotalInputTokens: getTotalInputTokens(),
    lastTotalOutputTokens: getTotalOutputTokens(),
    lastTotalCacheCreationInputTokens: getTotalCacheCreationInputTokens(),
    lastTotalCacheReadInputTokens: getTotalCacheReadInputTokens(),
    lastTotalWebSearchRequests: getTotalWebSearchRequests(),
    lastFpsAverage: fpsMetrics?.averageFps,
    lastFpsLow1Pct: fpsMetrics?.low1PctFps,
    lastModelUsage: Object.fromEntries(
      Object.entries(getModelUsage()).map(([model, usage]) => [
        model,
        {
          inputTokens: usage.inputTokens,
          outputTokens: usage.outputTokens,
          cacheReadInputTokens: usage.cacheReadInputTokens,
          cacheCreationInputTokens: usage.cacheCreationInputTokens,
          webSearchRequests: usage.webSearchRequests,
          costUSD: usage.costUSD,
        },
      ]),
    ),
    lastSessionId: getSessionId(),
  }))
}

export function restoreCostStateForSession(sessionId: string): boolean {
  const data = getStoredSessionCosts(sessionId)
  if (!data) return false
  setCostStateForRestore(data)
  return true
}
```

### 4.6.4 Token Budget：自动续传机制

当 Token 预算接近上限时，系统会自动注入续传消息：

```typescript
// tokenBudget.ts
const COMPLETION_THRESHOLD = 0.9  // 90% 阈值
const DIMINISHING_THRESHOLD = 500  // 500 token 收益阈值

export function checkTokenBudget(
  tracker: BudgetTracker,
  agentId: string | undefined,
  budget: number | null,
  globalTurnTokens: number,
): TokenBudgetDecision {
  if (agentId || budget === null || budget <= 0) {
    return { action: 'stop', completionEvent: null }
  }

  const turnTokens = globalTurnTokens
  const pct = Math.round((turnTokens / budget) * 100)
  const deltaSinceLastCheck = globalTurnTokens - tracker.lastGlobalTurnTokens

  // 检测收益递减
  const isDiminishing =
    tracker.continuationCount >= 3 &&
    deltaSinceLastCheck < DIMINISHING_THRESHOLD &&
    tracker.lastDeltaTokens < DIMINISHING_THRESHOLD

  // 未达到收益递减且未超过阈值时继续
  if (!isDiminishing && turnTokens < budget * COMPLETION_THRESHOLD) {
    tracker.continuationCount++
    tracker.lastDeltaTokens = deltaSinceLastCheck
    tracker.lastGlobalTurnTokens = globalTurnTokens
    return {
      action: 'continue',
      nudgeMessage: getBudgetContinuationMessage(pct, turnTokens, budget),
      continuationCount: tracker.continuationCount,
      pct,
      turnTokens,
      budget,
    }
  }

  // 达到阈值或收益递减时停止
  if (isDiminishing || tracker.continuationCount > 0) {
    return {
      action: 'stop',
      completionEvent: {
        continuationCount: tracker.continuationCount,
        pct,
        turnTokens,
        budget,
        diminishingReturns: isDiminishing,
        durationMs: Date.now() - tracker.startedAt,
      },
    }
  }

  return { action: 'stop', completionEvent: null }
}
```

### 4.6.5 使用量更新策略

API 返回的使用量是**累积值**，需要正确处理：

```typescript
// claude.ts 中的使用量更新
export function updateUsage(
  usage: Readonly<NonNullableUsage>,
  partUsage: BetaMessageDeltaUsage | undefined,
): NonNullableUsage {
  if (!partUsage) {
    return { ...usage }
  }
  return {
    input_tokens:
      partUsage.input_tokens !== null && partUsage.input_tokens > 0
        ? partUsage.input_tokens
        : usage.input_tokens,
    cache_creation_input_tokens:
      partUsage.cache_creation_input_tokens !== null &&
      partUsage.cache_creation_input_tokens > 0
        ? partUsage.cache_creation_input_tokens
        : usage.cache_creation_input_tokens,
    cache_read_input_tokens:
      partUsage.cache_read_input_tokens !== null &&
      partUsage.cache_read_input_tokens > 0
        ? partUsage.cache_read_input_tokens
        : usage.cache_read_input_tokens,
    output_tokens: partUsage.output_tokens ?? usage.output_tokens,
    // ...
  }
}
```

注意事项：
1. **非空检查**：只有非零值才更新，避免 message_delta 的零值覆盖 message_start 的值
2. **累积特性**：input_tokens 在 message_start 后不再变化，output_tokens 逐步增长
3. **delta 语义**：message_delta 只包含自上次事件以来的变化

## 4.7 小结与思考

Query Engine 是 Claude Code 与 AI 模型交互的核心枢纽。通过对 `query.ts` 和 `QueryEngine.ts` 的深入分析，我们可以总结以下设计思想：

### 4.7.1 架构设计

1. **分层职责**：QueryEngine 管理会话状态，query() 处理查询逻辑，API 层负责网络通信
2. **生成器模式**：使用 AsyncGenerator 实现流式控制，保持代码的可读性和可组合性
3. **状态显式化**：所有跨迭代状态集中在 `State` 类型中，易于追踪和测试

### 4.7.2 错误处理

1. **分类策略**：通过 `categorizeRetryableAPIError` 区分可恢复和不可恢复错误
2. **多层重试**：指数退避重试 + 模型回退 + 非流式回退
3. **优雅降级**：max_output_tokens 恢复、prompt_too_long 压缩等

### 4.7.3 上下文管理

1. **多层策略**：工具预算 → 微压缩 → 自动压缩 → 响应式压缩
2. **缓存友好**：使用 Forked Agent 复用提示词缓存
3. **透明性**：压缩对模型透明，通过附件机制保留关键上下文

### 4.7.4 可观测性

1. **实时费用**：每次 API 调用后立即计算并累积
2. **事件追踪**：通过 logEvent 记录关键决策点
3. **性能指标**：TTFT、停顿检测、API 时长等

### 4.7.5 思考题

1. **生成器 vs 回调**：为什么使用 AsyncGenerator 而不是回调或 Promise 链？流式控制的优势在哪里？

2. **状态管理**：`State` 类型集中了所有跨迭代状态，这种设计在什么情况下会变得过于复杂？如何权衡？

3. **错误分类**：`categorizeRetryableAPIError` 的分类策略是否完备？是否有遗漏或误判的情况？

4. **压缩策略**：多层压缩机制是否足够智能？能否引入更智能的预测性压缩？

5. **费用追踪**：如何在不影响性能的前提下实现更精细的费用追踪？例如按工具、按文件统计？

### 4.7.6 设计模式总结

Query Engine 的设计中体现了多个经典的设计模式：

| 设计模式 | 应用场景 | 优势 |
|----------|----------|------|
| **生成器模式** | query() 的流式处理 | 惰性求值、可组合性 |
| **策略模式** | 不同的压缩策略 | 算法可替换、易于扩展 |
| **责任链模式** | 错误分类与处理 | 逐级处理、灵活组合 |
| **观察者模式** | 进度事件流 | 解耦生产者与消费者 |
| **适配器模式** | 消息格式转换 | 统一接口、隔离变化 |
| **模板方法模式** | Forked Agent 调用 | 复用流程、定制步骤 |

### 4.7.7 依赖注入设计

Query Engine 使用了显式的依赖注入模式，使得测试和扩展变得简单：

```typescript
// deps.ts
export type QueryDeps = {
  // 模型调用
  callModel: typeof queryModelWithStreaming

  // 压缩
  microcompact: typeof microcompactMessages
  autocompact: typeof autoCompactIfNeeded

  // 平台
  uuid: () => string
}

// 生产环境依赖
export function productionDeps(): QueryDeps {
  return {
    callModel: queryModelWithStreaming,
    microcompact: microcompactMessages,
    autocompact: autoCompactIfNeeded,
    uuid: randomUUID,
  }
}

// 测试环境依赖
export function testDeps(overrides: Partial<QueryDeps>): QueryDeps {
  return {
    ...productionDeps(),
    ...overrides,
  }
}
```

这种设计的优势：
1. **可测试性**：测试时可以注入 mock 函数
2. **可替换性**：可以轻松切换不同的实现
3. **类型安全**：TypeScript 保证类型兼容性

### 4.7.8 配置管理（QueryConfig）

`QueryConfig` 捕获查询开始时的不可变配置：

```typescript
// config.ts
export type QueryConfig = {
  sessionId: SessionId

  // Runtime gates (env/statsig)
  gates: {
    streamingToolExecution: boolean
    emitToolUseSummaries: boolean
    isAnt: boolean
    fastModeEnabled: boolean
  }
}

export function buildQueryConfig(): QueryConfig {
  return {
    sessionId: getSessionId(),
    gates: {
      streamingToolExecution: checkStatsigFeatureGate_CACHED_MAY_BE_STALE(
        'tengu_streaming_tool_execution2',
      ),
      emitToolUseSummaries: isEnvTruthy(
        process.env.CLAUDE_CODE_EMIT_TOOL_USE_SUMMARIES,
      ),
      isAnt: process.env.USER_TYPE === 'ant',
      fastModeEnabled: !isEnvTruthy(process.env.CLAUDE_CODE_DISABLE_FAST_MODE),
    },
  }
}
```

配置设计的原则：
1. **不可变性**：配置在查询开始时确定，之后不再修改
2. **类型安全**：使用 TypeScript 类型确保配置的正确性
3. **环境感知**：根据环境变量和 Statsig 特性门动态配置

### 4.7.9 Transition 类型系统

查询循环的状态转换通过 `Transition` 类型系统显式化：

```typescript
// transitions.ts
export type Terminal =
  | { reason: 'completed' }
  | { reason: 'aborted_streaming' }
  | { reason: 'aborted_tools' }
  | { reason: 'model_error'; error: unknown }
  | { reason: 'stop_hook_prevented' }
  | { reason: 'stop_hook_blocking' }
  | { reason: 'max_turns'; turnCount: number }
  | { reason: 'prompt_too_long' }
  | { reason: 'image_error' }
  | { reason: 'hook_stopped' }

export type Continue =
  | { reason: 'next_turn' }
  | { reason: 'collapse_drain_retry'; committed: number }
  | { reason: 'reactive_compact_retry' }
  | { reason: 'max_output_tokens_escalate' }
  | { reason: 'max_output_tokens_recovery'; attempt: number }
  | { reason: 'token_budget_continuation' }
  | { reason: 'stop_hook_blocking' }

// State 中的 transition 字段记录了上一次转换的原因
type State = {
  // ...
  transition: Continue | undefined  // 上一次继续的原因
}
```

这种设计的优势：
1. **可追溯性**：每次状态转换都有明确的原因记录
2. **可测试性**：测试可以断言 `transition.reason` 的值
3. **可诊断性**：调试时可以清晰地看到状态转换路径

### 4.7.10 性能分析技巧

Query Engine 的性能直接影响用户体验。关键的性能指标包括：

| 指标 | 说明 | 目标值 |
|------|------|--------|
| **TTFT** | Time To First Token | < 2s |
| **流式停顿** | chunk 间隔 > 30s | < 1 次/会话 |
| **API 时长** | 单次 API 调用时长 | 取决于输出长度 |
| **工具延迟** | 工具执行延迟 | < 1s（本地） |
| **总延迟** | 用户输入到首字符输出 | TTFT + 0.5s |

性能优化技巧：

1. **并行处理**：
```typescript
// 压缩、工具执行、模型响应并行
const [compactionResult, toolResults] = await Promise.all([
  deps.autocompact(...),
  runTools(toolUseBlocks, ...),
])
```

2. **缓存预取**：
```typescript
// 内存预取
using pendingMemoryPrefetch = startRelevantMemoryPrefetch(
  state.messages,
  state.toolUseContext,
)

// Skill 发现预取
const pendingSkillPrefetch = skillPrefetch?.startSkillDiscoveryPrefetch(...)
```

3. **惰性求值**：
```typescript
// 只在需要时计算
const querySourceForEvent =
  recompactionInfo?.querySource ?? context.options.querySource ?? 'unknown'
```

4. **资源复用**：
```typescript
// Anthropic 客户端缓存
const cachedClient = await getAnthropicClient({ /* ... */ })

// 文件读取缓存
const cached = readFileState.get(filePath)
```

### 4.7.11 测试策略

Query Engine 的测试策略体现了测试金字塔的原则：

1. **单元测试**：测试单个函数，如 `tokenCountWithEstimation`、`categorizeRetryableAPIError`
2. **集成测试**：测试多个组件的交互，如 `query` 循环
3. **端到端测试**：测试完整的用户场景

```typescript
// 典型的单元测试
describe('tokenCountWithEstimation', () => {
  it('returns 0 for empty messages', () => {
    expect(tokenCountWithEstimation([])).toBe(0)
  })

  it('prioritizes API-reported usage', () => {
    const messages: Message[] = [
      { type: 'assistant', message: { usage: { input_tokens: 1000 } } },
      { type: 'user', message: { content: 'hello' } },
    ]
    expect(tokenCountWithEstimation(messages)).toBe(1000)
  })

  it('falls back to estimation when usage unavailable', () => {
    const messages: Message[] = [
      { type: 'user', message: { content: 'hello world' } },
    ]
    expect(tokenCountWithEstimation(messages)).toBe(3)  // 约 12 字符 / 4
  })
})

// 典型的集成测试
describe('query loop with tool calls', () => {
  it('continues after tool execution', async () => {
    const deps = testDeps({
      callModel: async function* () {
        yield { type: 'assistant', message: { content: [{ type: 'tool_use', name: 'Read', id: '1' }] } }
      },
    })
    const result = await collect(query({ deps, /* ... */ }))
    expect(result.length).toBeGreaterThan(1)
  })
})
```

### 4.7.12 调试与诊断工具

Query Engine 提供了丰富的调试和诊断工具：

1. **Checkpoint 系统**：在关键位置设置检查点，记录时间戳
```typescript
queryCheckpoint('query_fn_entry')
queryCheckpoint('query_snip_start')
queryCheckpoint('query_microcompact_start')
queryCheckpoint('query_autocompact_start')
queryCheckpoint('query_api_streaming_start')
queryCheckpoint('query_api_streaming_end')
```

2. **事件日志**：通过 `logEvent` 记录关键决策
```typescript
logEvent('tengu_auto_compact_succeeded', {
  originalMessageCount: messages.length,
  compactedMessageCount: compactionResult.summaryMessages.length,
  preCompactTokenCount,
  postCompactTokenCount,
  truePostCompactTokenCount,
  compactionInputTokens: compactionUsage?.input_tokens,
  compactionOutputTokens: compactionUsage?.output_tokens,
  queryChainId: queryTracking.chainId,
  queryDepth: queryTracking.depth,
})
```

3. **性能追踪**：通过 `headlessProfilerCheckpoint` 记录性能
```typescript
headlessProfilerCheckpoint('before_getSystemPrompt')
headlessProfilerCheckpoint('after_getSystemPrompt')
headlessProfilerCheckpoint('system_message_yielded')
headlessProfilerCheckpoint('api_request_sent')
headlessProfilerCheckpoint('first_chunk')
```

### 4.7.13 可扩展性设计

Query Engine 的设计考虑了未来的扩展性：

1. **Feature Gates**：通过 `feature()` 函数控制功能开关
```typescript
const reactiveCompact = feature('REACTIVE_COMPACT')
  ? require('./services/compact/reactiveCompact.js')
  : null
```

2. **Plugin 机制**：通过 Hook 系统允许扩展功能
```typescript
const hookResult = await executePreCompactHooks(
  { trigger: isAutoCompact ? 'auto' : 'manual' },
  context.abortController.signal,
)
```

3. **配置驱动**：通过环境变量和配置文件控制行为
```typescript
const STREAM_IDLE_TIMEOUT_MS = parseInt(process.env.CLAUDE_STREAM_IDLE_TIMEOUT_MS || '') || 90_000
```

### 4.7.14 实战案例分析

**案例 1：处理超大 PDF 文件**

当用户上传一个 100 页的 PDF 文件时，Query Engine 需要应对：

1. **预处理验证**：`imageValidation.ts` 检查 PDF 大小
2. **API 错误处理**：`errors.ts` 识别 PDF 超限错误
3. **恢复策略**：提示用户使用 `pdftotext` 转换为文本

```typescript
// errors.ts
export function getPdfTooLargeErrorMessage(): string {
  const limits = `max ${API_PDF_MAX_PAGES} pages, ${formatFileSize(PDF_TARGET_RAW_SIZE)}`
  return getIsNonInteractiveSession()
    ? `PDF too large (${limits}). Try reading the file a different way (e.g., extract text with pdftotext).`
    : `PDF too large (${limits}). Double press esc to go back and try again, or use pdftotext to convert to text first.`
}
```

**案例 2：流式中断恢复**

当用户在流式响应中途按下 ESC 键时：

1. **信号传递**：AbortController 传递到 API 层
2. **资源清理**：释放流式连接和原生资源
3. **状态恢复**：确保对话状态一致

```typescript
if (toolUseContext.abortController.signal.aborted) {
  if (streamingToolExecutor) {
    for await (const update of streamingToolExecutor.getRemainingResults()) {
      if (update.message) {
        yield update.message
      }
    }
  }

  yield createUserInterruptionMessage({
    toolUse: toolUseBlocks.length > 0,
  })

  return { reason: 'aborted_streaming' }
}
```

### 4.7.15 与传统 Web 应用的对比

| 方面 | 传统 Web 应用 | Claude Code Query Engine |
|------|---------------|-------------------------|
| **请求模式** | 一次性请求 | 多轮流式请求 |
| **状态管理** | 服务器会话 | 客户端会话 |
| **错误处理** | 全局错误处理器 | 分类错误恢复 |
| **上下文管理** | 无（或简单缓存） | 多层压缩策略 |
| **费用追踪** | 无 | 实时追踪 |

### 4.7.16 最佳实践建议

基于对 Query Engine 的分析，我们可以总结以下最佳实践：

1. **使用 AsyncGenerator**：对于流式处理，生成器比回调或 Promise 链更优雅
2. **显式状态管理**：将所有跨迭代状态集中在一个类型中
3. **分类错误处理**：根据错误类型采用不同的恢复策略
4. **提前验证**：在调用 API 之前验证输入，避免无效请求
5. **复用缓存**：通过 Forked Agent 等机制复用提示词缓存
6. **追踪性能**：使用 checkpoint 和 event log 记录关键指标
7. **测试隔离**：使用依赖注入将测试与实现解耦

### 4.7.17 阅读建议

理解 Query Engine 需要对以下概念有深入理解：

1. **TypeScript 高级类型**：泛型、类型守卫、联合类型
2. **异步编程**：AsyncGenerator、Promise、AbortSignal
3. **流式处理**：SSE、增量解析、背压管理
4. **设计模式**：生成器模式、策略模式、责任链模式

建议的阅读顺序：
1. 先读 `QueryEngine.ts` 理解会话级别的管理
2. 再读 `query.ts` 理解查询循环的核心逻辑
3. 然后读 `claude.ts` 理解 API 调用层
4. 最后读 `errors.ts` 和 `cost-tracker.ts` 理解错误处理和费用追踪

> "复杂性是设计不佳的征兆。" —— 虽然 Query Engine 的复杂性不可避免，但通过良好的设计和清晰的职责划分，我们可以让复杂的逻辑变得可理解、可测试、可维护。

1. **缓存复用**：Forked Agent 通过 `skipCacheWrite` 复用主对话的缓存
2. **并行执行**：压缩、工具执行、模型响应并行进行
3. **惰性求值**：生成器按需产生消息，避免不必要的计算
4. **资源复用**：Anthropic 客户端缓存、文件读取缓存、压缩缓存

### 4.7.18 未来演进方向

1. **预测性压缩**：基于对话模式预测何时需要压缩
2. **智能缓存管理**：自动识别和淘汰不常用的缓存内容
3. **多模型协同**：根据任务复杂度动态选择模型
4. **边缘计算**：在本地进行更多计算，减少 API 调用

> "最好的代码是没有代码。" —— 虽然 Query Engine 的复杂性不可避免，但通过良好的设计和清晰的职责划分，我们可以让复杂的逻辑变得可理解、可测试、可维护。
