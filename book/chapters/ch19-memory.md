# 第19章 记忆系统 —— AI的长期记忆机制

人类之所以能够学习、成长并积累智慧，根本原因在于我们拥有记忆。AI助手亦是如此——没有记忆的AI就像永远失忆的来访者，每次对话都是全新的开始。Claude Code通过精心设计的记忆系统，让AI能够跨越会话边界，真正"记住"用户、项目和历史协作。

Claude Code 的记忆能力至少分成三层：

- `src/memdir/`：持久化、可审阅的文件式长期记忆
- `src/services/extractMemories/`：对话结束后的自动提取与写回
- `src/services/SessionMemory/`：服务当前会话压缩与恢复的会话级记忆

因此，Claude Code 的“记忆”已经不是单一目录里的 Markdown 文件，而是长期记忆、会话记忆和自动提取流程的组合。

## 19.1 memdir文件式记忆

记忆系统采用基于文件系统的设计，称为memdir（memory directory）。这种设计具有几个显著优势：

**持久性**：记忆以Markdown文件形式存储在用户文件系统中，进程重启后依然保留。默认路径位于`~/.claude/projects/<sanitized-project-root>/memory/`。

**透明性**：用户可以直接查看和编辑记忆文件，了解AI记住了什么。文件使用Markdown格式，便于人类阅读和版本控制。

**可迁移性**：记忆文件可以随项目代码一起提交到仓库，团队成员可以共享项目相关的记忆。

**简单性**：文件系统是最通用的存储抽象，无需额外的数据库服务。

### 目录结构

```
~/.claude/projects/
└── <project-slug>/
    └── memory/
        ├── MEMORY.md           # 记忆索引文件
        ├── user_role.md        # 用户角色记忆
        ├── feedback_testing.md # 反馈记忆
        ├── project_context.md  # 项目上下文
        └── reference_systems.md# 外部系统引用
```

每个记忆文件都是独立的，包含frontmatter元数据（name、description、type）和正文内容。

### 环境隔离

记忆系统通过`getAutoMemPath()`实现按项目隔离：

```typescript
// src/memdir/paths.ts
export function getAutoMemPath(): string {
  const projectDir = getProjectDir(getOriginalCwd())
  const projectSlug = sanitizeProjectSlug(projectDir)
  return join(getMemoryBase(), 'projects', projectSlug, 'memory', '')
}
```

这种设计确保不同项目的记忆互不干扰，AI能够根据当前工作目录自动加载正确的记忆上下文。

## 19.2 记忆类型：四种记忆维度

Claude Code定义了四种记忆类型，每种类型都有明确的用途和保存策略：

### 19.2.1 user记忆

user记忆存储关于用户角色、目标、责任和知识的信息。这些记忆帮助AI调整协作风格——与资深工程师协作的方式应该与编程新手不同。

**何时保存**：
- 了解到用户的角色定位
- 发现用户的技术背景（如"十年Go经验，React新手"）
- 知道用户的偏好或工作习惯

**示例场景**：
```
用户：我是个数据科学家，正在调查日志系统
AI：[保存user记忆：用户是数据科学家，当前关注可观测性/日志]
```

### 19.2.2 feedback记忆

feedback记忆记录用户给出的工作方式指导——包括要避免的和要保持的。这是最重要的记忆类型，因为它确保AI不会反复犯同样的错误。

**保存时机**：
- 用户纠正你的做法（"不要那样"、"停止做X"）
- 用户确认非显而易见的成功做法（"是的，完全正确"、"完美，继续保持"）

**结构要求**：
```markdown
---
name: feedback_testing
description: 测试相关反馈
type: feedback
---

# 规则本身
集成测试必须使用真实数据库，不要使用mock

**Why:** 上个季度因为mock测试通过但生产迁移失败，我们吃过亏

**How to apply:** 编写任何涉及数据库的测试时，直接连接测试数据库
```

### 19.2.3 project记忆

project记忆存储项目中正在进行的工作、目标、计划、bug或事故信息——这些信息无法从代码或git历史中推导出来。

**保存时机**：
- 了解到谁在做什么、为什么做、何时完成
- 知道项目的约束条件（如"周四后冻结合并"）
- 了解项目动机（如"重构中间件是合规要求，非技术债清理"）

**注意事项**：project记忆时效性强，保存时必须将相对日期转换为绝对日期（如"Thursday" → "2026-03-05"），确保记忆在时间流逝后仍可理解。

### 19.2.4 reference记忆

reference记忆存储指向外部系统的指针——告诉AI在哪里可以找到项目目录之外的最新信息。

**保存时机**：
- 得知bug在某个Linear项目中追踪
- 得知反馈在特定Slack频道讨论
- 得知监控在某个Grafana面板展示

**示例**：
```
用户：INGEST Linear项目追踪所有pipeline bug
AI：[保存reference记忆：pipeline bug追踪位置为Linear项目"INGEST"]
```

### 不该保存的内容

记忆系统有明确的排除范围，避免存储可从项目状态推导的信息：

- 代码模式、约定、架构、文件路径或项目结构——可从当前项目读取
- Git历史、最近变更——`git log`/`git blame`是权威来源
- 调试方案或修复记录——修复已在代码中，commit message包含上下文
- CLAUDE.md中已记录的内容
- 临时任务详情——当前对话中的临时状态

## 19.3 记忆加载与注入

记忆通过系统提示词注入到AI的上下文中。每次会话开始时，`loadMemoryPrompt()`函数构建记忆相关的指令：

```typescript
// src/memdir/memdir.ts
export async function loadMemoryPrompt(): Promise<string | null> {
  if (!isAutoMemoryEnabled()) {
    return null
  }

  const autoDir = getAutoMemPath()
  await ensureMemoryDirExists(autoDir)

  return buildMemoryLines('auto memory', autoDir, extraGuidelines).join('\n')
}
```

### MEMORY.md索引文件

MEMORY.md是记忆系统的入口点，包含所有记忆文件的链接索引：

```markdown
- [User Role](user_role.md) — 高级软件工程师，专注后端开发
- [Testing Feedback](feedback_testing.md) — 集成测试必须使用真实数据库
- [Release Context](project_release.md) — 移动团队周四后冻结合并
```

### 截断保护

为防止MEMORY.md过度消耗上下文，系统实现了双重保护：

```typescript
export const MAX_ENTRYPOINT_LINES = 200
export const MAX_ENTRYPOINT_BYTES = 25_000

export function truncateEntrypointContent(raw: string): EntrypointTruncation {
  const lines = trimmed.split('\n')
  const wasLineTruncated = lines.length > MAX_ENTRYPOINT_LINES
  const wasByteTruncated = trimmed.length > MAX_ENTRYPOINT_BYTES

  // 先按行截断（自然边界），再按字节截断
  let truncated = wasLineTruncated
    ? lines.slice(0, MAX_ENTRYPOINT_LINES).join('\n')
    : trimmed

  if (truncated.length > MAX_ENTRYPOINT_BYTES) {
    const cutAt = truncated.lastIndexOf('\n', MAX_ENTRYPOINT_BYTES)
    truncated = truncated.slice(0, cutAt > 0 ? cutAt : MAX_ENTRYPOINT_BYTES)
  }

  return {
    content: truncated + warning,
    wasLineTruncated,
    wasByteTruncated
  }
}
```

### 记忆漂移警示

系统明确警告AI记忆可能过时：

```markdown
## Before recommending from memory

A memory that names a specific function, file, or flag is a claim that it existed *when the memory was written*. It may have been renamed, removed, or never merged.

- If the memory names a file path: check the file exists.
- If the memory names a function or flag: grep for it.
- If the user is about to act on your recommendation: verify first.

"The memory says X exists" is not the same as "X exists now."
```

## 19.4 自动记忆提取

记忆系统的核心创新是自动记忆提取（extractMemories）。在每个查询循环结束时，系统会分析对话内容并自动提取值得保存的记忆。

## 19.5 SessionMemory：长期记忆之外的会话级记忆层

长期记忆解决的是“跨会话保留什么”，但它并不覆盖当前会话中为压缩、恢复和持续推理所需的中间知识。当前源码通过 `src/services/SessionMemory/` 引入了一层会话级记忆：

- `sessionMemory.ts`：会话记忆的组织与生命周期
- `sessionMemoryUtils.ts`：辅助构造与格式化
- `prompts.ts`：把会话记忆注入压缩与恢复提示词

这层设计很重要，因为它说明 Claude Code 已经把“记忆”拆成两个不同问题：

1. **长期可持久化的知识**
   适合写入 `memdir/`，供未来会话使用。

2. **当前会话内的阶段性知识**
   适合在压缩、恢复、摘要之间流转，但未必应该永久保存。

这样的分层避免了一个常见陷阱：把所有中间状态都写进长期记忆，最终让记忆目录变成低质量日志堆积。

## 19.6 团队记忆同步：从个人记忆走向共享记忆

源码中还存在 `src/services/teamMemorySync/`，包括 watcher、secretScanner、guard 等模块。记忆系统已经开始处理“团队共享”场景，而不只是单用户本地记忆。

这里最突出的特征是它的安全姿态：

- 同步前会做 secret scanning
- 有 team memory secret guard
- 通过 watcher 监听而不是盲目全量同步

Claude Code 并没有把“共享记忆”当作简单的文件同步问题，而是把它视为一个需要安全审查的协作面。

### 工作流程

```typescript
// src/services/extractMemories/extractMemories.ts
export async function executeExtractMemories(
  context: REPLHookContext,
  appendSystemMessage?: AppendSystemMessage
): Promise<void> {
  // 检查特性门控
  if (!getFeatureValue_CACHED_MAY_BE_STALE('tengu_passport_quail', false)) {
    return
  }

  // 计算新消息数量
  const newMessageCount = countModelVisibleMessagesSince(
    messages,
    lastMemoryMessageUuid
  )

  // 预注入记忆文件清单，避免agent花费一轮执行ls
  const existingMemories = formatMemoryManifest(
    await scanMemoryFiles(memoryDir, signal)
  )

  // 构建提取提示词
  const userPrompt = buildExtractAutoOnlyPrompt(
    newMessageCount,
    existingMemories
  )

  // 运行forked agent执行提取
  const result = await runForkedAgent({
    promptMessages: [createUserMessage({ content: userPrompt })],
    cacheSafeParams,
    canUseTool,
    querySource: 'extract_memories',
    maxTurns: 5
  })
}
```

### Forked Agent模式

提取使用"完美分支"（perfect fork）模式——创建一个与主对话共享prompt缓存的新agent实例：

1. **共享缓存**：forked agent重用父对话的system prompt缓存，避免重复计算
2. **隔离执行**：forked agent的输出不影响主对话
3. **工具限制**：forked agent只能访问Read/Grep/Glob和memory目录内的Edit/Write

### 互斥机制

当主agent已经写入记忆文件时，forked提取会被跳过：

```typescript
function hasMemoryWritesSince(
  messages: Message[],
  sinceUuid: string | undefined
): boolean {
  // 检查是否有assistant消息向memory路径写入了文件
  for (const message of messages) {
    if (message.type !== 'assistant') continue
    for (const block of content) {
      if (getWrittenFilePath(block) && isAutoMemPath(filePath)) {
        return true
      }
    }
  }
  return false
}
```

这种设计确保两种写入方式互斥，避免重复工作。

### 节流与合并

提取过程实现了节流机制，避免频繁调用：

```typescript
let turnsSinceLastExtraction = 0

if (!isTrailingRun) {
  turnsSinceLastExtraction++
  const threshold = getFeatureValue_CACHED_MAY_BE_STALE(
    'tengu_bramble_lintel',
    null
  ) ?? 1
  if (turnsSinceLastExtraction < threshold) {
    return
  }
}
```

当提取正在进行时，新的请求会被合并，在当前提取完成后执行一次"尾随提取"：

```typescript
if (inProgress) {
  pendingContext = { context, appendSystemMessage }
  return
}

// 在finally块中处理pending
const trailing = pendingContext
pendingContext = undefined
if (trailing) {
  await runExtraction({
    context: trailing.context,
    appendSystemMessage: trailing.appendSystemMessage,
    isTrailingRun: true
  })
}
```

## 19.7 记忆检索与相关性选择

记忆系统不仅负责存储，还需要智能检索。`findRelevantMemories()`函数实现了基于查询的记忆检索：

```typescript
// src/memdir/findRelevantMemories.ts
export async function findRelevantMemories(
  query: string,
  memoryDir: string,
  signal: AbortSignal,
  recentTools: readonly string[] = [],
  alreadySurfaced: ReadonlySet<string> = new Set()
): Promise<RelevantMemory[]> {
  // 扫描记忆文件，读取frontmatter
  const memories = await scanMemoryFiles(memoryDir, signal)

  // 使用Sonnet选择最相关的记忆（最多5个）
  const selectedFilenames = await selectRelevantMemories(
    query,
    memories,
    signal,
    recentTools
  )

  return selected.map(m => ({
    path: m.filePath,
    mtimeMs: m.mtimeMs
  }))
}
```

### 选择策略

系统使用单独的Sonnet调用来选择相关记忆，而非简单的向量搜索：

```typescript
const SELECT_MEMORIES_SYSTEM_PROMPT = `You are selecting memories that will be useful to Claude Code as it processes a user's query.

Return a list of filenames for the memories that will clearly be useful (up to 5).
- If unsure, do not include it. Be selective.
- If no memories would be useful, return an empty list.
`
```

这种设计有几个优点：

1. **精确性**：LLM理解语义，优于关键词匹配
2. **选择性**：明确指示"不确定就排除"，减少噪音
3. **上下文感知**：考虑recentTools，避免重复推荐正在使用的工具文档

### 工具使用感知

当用户正在使用某个工具时，系统会排除该工具的基础文档：

```typescript
const toolsSection = recentTools.length > 0
  ? `\n\nRecently used tools: ${recentTools.join(', ')}`
  : ''
```

这避免了在用户已经熟练使用工具时，重复推送入门文档的干扰。

## 19.8 记忆索引管理

记忆系统通过MEMORY.md索引文件组织所有记忆。管理这个索引需要平衡完整性和简洁性。

### 两步保存法

系统明确要求两步保存流程：

**Step 1**：将记忆写入独立的主题文件

**Step 2**：在MEMORY.md中添加索引链接

这种分离设计确保：

- MEMORY.md保持简洁（每条索引约150字符）
- 详细内容在主题文件中展开
- 索引文件可以快速扫描主题列表

### 重复检测

系统在提取提示词中明确要求检查重复：

```markdown
## Existing memory files

- [User Role](user_role.md) — 高级软件工程师
- [Testing](feedback_testing.md) — 集成测试使用真实数据库

Check this list before writing — update an existing file rather than creating a duplicate.
```

这减少了记忆碎片化，保持记忆库的整洁。

### 去重与更新

当发现记忆过时或错误时，系统鼓励更新而非累积：

```markdown
- Update or remove memories that turn out to be wrong or outdated
- Do not write duplicate memories. First check if there is an existing memory you can update before writing a new one.
```

这种"保鲜"机制确保记忆库不会随时间膨胀成过时信息的堆场。

## 19.9 小结

Claude Code的记忆系统展示了如何为AI构建长期记忆能力：

1. **文件式存储**：利用文件系统的持久性、透明性和简单性
2. **类型化记忆**：通过四种明确的类型，规范记忆的保存和检索
3. **自动提取**：在对话结束时自动分析并提取值得保存的记忆
4. **智能检索**：使用LLM进行语义相关性的记忆选择
5. **索引管理**：通过MEMORY.md平衡完整性和简洁性

这个系统的核心思想是：记忆不是简单地存储所有对话，而是有选择地保存那些**在未来会话中有价值的上下文**。这种设计让AI能够像人类一样，通过记忆积累经验，提供越来越个性化的帮助。

记忆系统也体现了AI产品设计的几个重要原则：

- **透明性**：用户可以查看AI记住了什么
- **可编辑性**：用户可以修正或删除不准确的记忆
- **适度性**：不是所有信息都值得记忆，系统有明确的排除范围
- **时效性**：记忆可能过时，系统鼓励验证和更新

这些原则共同构成了一个既强大又可控的AI记忆系统。
