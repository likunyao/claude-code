# 第16章 Skill与Command —— 行为复用的艺术

在Claude Code中，Skill和Command系统构成了用户交互与AI能力之间的桥梁。这个系统不仅让用户能够通过斜杠命令快速触发预定义行为，还允许AI主动识别并调用这些技能来完成复杂任务。本章将深入解析这个精巧的设计，揭示它如何实现行为的优雅复用。

## 16.1 技能注册与执行

### 16.1.1 技能的本质：可复用的提示词模板

在Claude Code的架构中，一个"技能"(Skill)本质上是一个结构化的提示词模板。当用户输入`/commit`或AI识别到需要创建git提交时，系统不是简单地执行一个函数，而是展开一段精心设计的提示词，让AI按照既定流程完成任务。

让我们从核心数据结构开始：

```typescript
// src/types/command.ts
export type Command = CommandBase & (
  | PromptCommand
  | HookCommand
  | ForkCommand
)

export type PromptCommand = {
  type: 'prompt'
  name: string
  description: string
  whenToUse?: string
  allowedTools: string[]
  getPromptForCommand: (
    args: string,
    context: ToolUseContext,
  ) => Promise<ContentBlockParam[]>
  // ...更多元数据
}
```

这个设计的精妙之处在于：技能不是命令式的代码，而是声明式的意图描述。`getPromptForCommand`函数返回的是要注入到对话中的内容块，让AI理解并执行。

### 16.1.2 内置技能的注册机制

Claude Code中有一批内置技能，如`/commit`、`/review-pr`等。它们的注册方式展示了系统的灵活性：

```typescript
// src/commands/commit.ts
const command = {
  type: 'prompt',
  name: 'commit',
  description: 'Create a git commit',
  allowedTools: ALLOWED_TOOLS,
  async getPromptForCommand(_args, context) {
    const promptContent = getPromptContent()
    const finalContent = await executeShellCommandsInPrompt(
      promptContent,
      { ...context, /* 权限修改 */ },
      '/commit',
    )
    return [{ type: 'text', text: finalContent }]
  },
} satisfies Command

export default command
```

注意`executeShellCommandsInPrompt`的调用。这个函数处理提示词中的shell命令注入语法(`\`...\``)，在展开提示词前执行这些命令并将结果注入。这使得技能可以动态获取当前状态（如git status、git diff）并传递给AI。

### 16.1.3 技能的发现与加载

技能的来源多样：内置、用户目录、项目目录、插件、MCP服务器。`loadSkillsDir.ts`实现了统一的发现和去重机制：

```typescript
// src/skills/loadSkillsDir.ts
export const getSkillDirCommands = memoize(
  async (cwd: string): Promise<Command[]> => {
    const managedSkillsDir = join(getManagedFilePath(), '.claude', 'skills')
    const userSkillsDir = join(getClaudeConfigHomeDir(), 'skills')
    const projectSkillsDirs = getProjectDirsUpToHome('skills', cwd)

    // 并行加载所有来源的技能
    const [managedSkills, userSkills, projectSkills, ...] = await Promise.all([
      loadSkillsFromSkillsDir(managedSkillsDir, 'policySettings'),
      loadSkillsFromSkillsDir(userSkillsDir, 'userSettings'),
      // ...
    ])

    // 去重：通过realpath解析符号链接，避免重复加载同一文件
    const fileIds = await Promise.all(
      allSkillsWithPaths.map(({ filePath }) => getFileIdentity(filePath))
    )

    const seenFileIds = new Map<string, string>()
    const deduplicatedSkills: Command[] = []
    for (const entry of allSkillsWithPaths) {
      const fileId = fileIds[i]
      if (fileId && !seenFileIds.has(fileId)) {
        seenFileIds.set(fileId, skill.source)
        deduplicatedSkills.push(skill)
      }
    }
    return deduplicatedSkills
  }
)
```

去重逻辑使用`realpath`来解析符号链接，确保同一物理文件只加载一次，无论通过多少路径访问。

### 16.1.4 条件技能：路径感知的自动激活

一个高级特性是"条件技能"：某些技能只有在操作特定路径下的文件时才会激活。这是通过frontmatter中的`paths`字段实现的：

```typescript
// src/skills/loadSkillsDir.ts
export function activateConditionalSkillsForPaths(
  filePaths: string[],
  cwd: string,
): string[] {
  const activated: string[] = []

  for (const [name, skill] of conditionalSkills) {
    if (!skill.paths || skill.paths.length === 0) continue

    // 使用ignore库实现gitignore风格的路径匹配
    const skillIgnore = ignore().add(skill.paths)
    for (const filePath of filePaths) {
      const relativePath = relative(cwd, filePath)
      if (skillIgnore.ignores(relativePath)) {
        dynamicSkills.set(name, skill)
        conditionalSkills.delete(name)
        activated.push(name)
        break
      }
    }
  }
  return activated
}
```

当用户编辑`package.json`时，npm相关的技能会自动激活，而不需要用户手动调用。

## 16.2 命令注册表模式

### 16.2.1 统一的命令注册表

所有命令（内置、技能、插件、MCP）最终汇聚到一个统一的注册表中。`commands.ts`是这个汇聚点：

```typescript
// src/commands.ts
export const getCommands = memoize(async (cwd: string): Promise<Command[]> => {
  const builtinCommands = [
    addDir, autofixPr, backfillSessions, btw, clear, color,
    commit, copy, desktop, commitPushPr, config, context, cost,
    // ...100+内置命令
  ]

  // 获取动态加载的技能
  const skillCommands = await getSkillDirCommands(cwd)

  // 获取bundled技能
  const bundledSkills = getBundledSkills()

  // 获取插件命令
  const pluginCommands = await getPluginCommands()

  // 合并并去重
  return uniqBy(
    [...builtinCommands, ...skillCommands, ...bundledSkills, ...pluginCommands],
    'name'
  )
})
```

这种设计让新命令的来源（插件、MCP、用户目录）对调用方透明。无论命令来自哪里，查找和调用的方式都是一致的。

### 16.2.3 Command 已经不是“内置命令 + skills”这么简单

从当前 `src/commands.ts` 看，命令系统已经是一个多来源聚合层，至少包括：

1. **内置命令**
   `/config`、`/review`、`/plugin`、`/permissions`、`/tasks` 等。

2. **技能目录命令**
   来自用户、项目、托管目录中的 Markdown / SKILL 文件。

3. **bundled skills**
   跟随产品发布的内置技能。

4. **插件命令与插件技能**
   通过插件系统动态挂载。

5. **feature-flag 命令**
   如 bridge、workflow、voice、ultraplan 等，只在特定构建或实验阶段出现。

Command 层承担的是“统一行为入口”，而不是“静态斜杠命令表”。

## 16.2.4 插件命令的真实载体是 Markdown + frontmatter

`src/utils/plugins/loadPluginCommands.ts` 表明，插件命令并不要求开发者写 TypeScript；它们可以直接来自 Markdown 文件，并通过 frontmatter 声明元数据。

加载过程大致是：

1. 递归扫描插件目录中的 Markdown
2. 识别 `SKILL.md` 这种技能目录入口
3. 解析 frontmatter
4. 从文件名和目录结构生成命名空间命令名
5. 把 Markdown 内容转换成 `Command`

这使得 Claude Code 的命令/技能系统具备了非常强的“内容驱动”特性：

- 普通开发者可以不用修改核心代码就扩展行为
- 文档、提示词、工具权限和参数说明可以共存于同一文件
- 插件加载器负责把这些内容翻译成统一的命令对象

### 16.2.5 命名空间的真正意义：隔离来源，而不是只为了好看

插件命令的命名规则不是简单的字符串拼接，而是把插件名、目录层级和 skill 目录一起编码进命令名，例如：

```text
pluginName:backend:deploy
pluginName:security:audit
```

这样做的价值有三点：

1. **避免冲突**
   不同插件都可以有 `review`、`deploy` 这样的常见名字。

2. **保留来源信息**
   用户和模型都能知道命令来自哪个插件。

3. **支持目录即组织结构**
   文件夹层级自然变成命令命名空间，不需要额外注册表。

### 16.2.2 命令查找与命名空间

命令支持命名空间，使用`:`作为分隔符：

```typescript
// src/skills/loadSkillsDir.ts
function buildNamespace(targetDir: string, baseDir: string): string {
  const relativePath = targetDir.slice(baseDir.length + 1)
  return relativePath ? relativePath.split(pathSep).join(':') : ''
}

function getSkillCommandName(filePath: string, baseDir: string): string {
  const skillDirectory = dirname(filePath)
  const parentOfSkillDir = dirname(skillDirectory)
  const commandBaseName = basename(skillDirectory)
  const namespace = buildNamespace(parentOfSkillDir, baseDir)
  return namespace ? `${namespace}:${commandBaseName}` : commandBaseName
}
```

`/project:backend:api:test`可以表示项目backend模块api子模块的测试技能。这种层次结构让大型项目可以有组织的技能库。

## 16.3 核心命令实现

### 16.3.1 commit命令：Git工作流的最佳实践封装

`/commit`命令是Skill系统设计理念的完美体现。它不直接执行git commit，而是构建一个结构化的提示词，引导AI按照最佳实践创建提交：

```typescript
// src/commands/commit.ts
function getPromptContent(): string {
  return `## Context

- Current git status: !\`git status\`
- Current git diff: !\`git diff HEAD\`
- Recent commits: !\`git log --oneline -10\`

## Git Safety Protocol

- NEVER update the git config
- NEVER skip hooks unless explicitly requested
- CRITICAL: ALWAYS create NEW commits
- Do not commit files that likely contain secrets

## Your task

1. Analyze all staged changes and draft a commit message
2. Stage relevant files and create the commit using HEREDOC syntax
`
}
```

注意Heredoc语法的使用：

```bash
git commit -m "$(cat <<'EOF'
Commit message here.
EOF
)"
```

这种写法避免了shell转义问题，支持多行提交信息。Skill系统通过模板引导AI使用正确的命令模式，而不是自己执行git操作。

### 16.3.2 review命令：代码审查的结构化流程

`/review`命令展示了如何构建多步骤的AI工作流：

```typescript
// src/commands/review.ts
const reviewSteps = [
  'Read the relevant files',
  'Analyze the changes',
  'Check for security vulnerabilities',
  'Verify error handling',
  'Review for potential bugs',
  'Suggest improvements'
]

const command = {
  type: 'prompt',
  name: 'review',
  whenToUse: 'When the user asks for a code review',
  async getPromptForCommand(args, context) {
    // 构建包含审查检查点的提示词
    const checklist = reviewSteps.map(step => `- [ ] ${step}`).join('\n')
    return [{
      type: 'text',
      text: `Please review the code changes:\n\n${checklist}\n\n...`
    }]
  }
}
```

这种结构化提示确保AI不会遗漏审查的关键步骤。

## 16.4 斜杠命令DSL

### 16.4.1 Shell命令注入语法

Skill系统支持特殊的shell命令注入语法，使用反引号：

```markdown
## Current Status
- Git branch: !`git branch --show-current`
- Recent commits: !`git log --oneline -5`
```

`!\`...\``语法会在技能展开时执行命令并将结果注入提示词。`executeShellCommandsInPrompt`实现这个功能：

```typescript
// src/utils/promptShellExecution.ts
export async function executeShellCommandsInPrompt(
  content: string,
  context: ToolUseContext,
  skillName: string,
): Promise<string> {
  const regex = /!(`(?:[^`]|``[^`])*?`)/g
  let match
  let result = content

  while ((match = regex.exec(content)) !== null) {
    const [fullMatch, command] = match
    const execResult = await execFile(command, { /* options */ })
    result = result.replace(fullMatch, execResult.stdout.trim())
  }
  return result
}
```

### 16.4.2 参数替换与变量展开

Skill支持参数替换和内置变量：

```markdown
# Skill: ${CLAUDE_SKILL_DIR}
Session ID: ${CLAUDE_SESSION_ID}
Arguments: $ARGUMENTS
```

`substituteArguments`函数处理这些替换：

```typescript
// src/utils/argumentSubstitution.ts
export function substituteArguments(
  content: string,
  args: string,
  allowPositional: boolean,
  argNames: string[],
): string {
  let result = content

  // 替换 $ARGUMENTS
  result = result.replace(/\$ARGUMENTS/g, args)

  // 替换命名参数 $1, $2 等
  if (allowPositional && argNames.length > 0) {
    const argValues = args.split(/\s+/)
    for (let i = 0; i < argNames.length; i++) {
      const placeholder = new RegExp(`\\$${i + 1}`, 'g')
      result = result.replace(placeholder, argValues[i] || '')
    }
  }

  return result
}
```

### 16.4.3 Frontmatter配置

每个Skill可以通过YAML frontmatter配置其行为：

```yaml
---
name: deploy
description: Deploy to production
when_to_use: When deploying to production
allowed-tools:
  - Bash
  - Read
  - Edit
model: opus
effort: high
context: fork
paths:
  - infra/**
  - deploy/**, ContentBlockParam[]>
```

`parseSkillFrontmatterFields`解析这些配置：

```typescript
// src/skills/loadSkillsDir.ts
export function parseSkillFrontmatterFields(
  frontmatter: FrontmatterData,
  markdownContent: string,
  resolvedName: string,
): { /* ... */ } {
  return {
    displayName: frontmatter.name,
    description: validatedDescription ?? extractDescriptionFromMarkdown(markdownContent),
    allowedTools: parseSlashCommandToolsFromFrontmatter(frontmatter['allowed-tools']),
    model: frontmatter.model ? parseUserSpecifiedModel(frontmatter.model) : undefined,
    effort: parseEffortValue(frontmatter.effort),
    executionContext: frontmatter.context === 'fork' ? 'fork' : undefined,
    paths: parseSkillPaths(frontmatter),
    // ...
  }
}
```

## 16.5 命令与工具的协作

### 16.5.1 Skill工具：命令的工具化封装

在Claude Code的AI工具调用体系中，Skill本身被封装为一个Tool：

```typescript
// src/tools/SkillTool/SkillTool.ts
export const SkillTool: Tool<InputSchema, Output, Progress> = buildTool({
  name: SKILL_TOOL_NAME, // "Skill"
  description: async ({ skill }) => `Execute skill: ${skill}`,

  async validateInput({ skill }, context): Promise<ValidationResult> {
    const commands = await getAllCommands(context)
    const foundCommand = findCommand(skill, commands)
    if (!foundCommand) {
      return {
        result: false,
        message: `Unknown skill: ${skill}`,
        errorCode: 2,
      }
    }
    return { result: true }
  },

  async call({ skill, args }, context, canUseTool, parentMessage) {
    const command = findCommand(skill, await getAllCommands(context))
    const processedCommand = await processPromptSlashCommand(
      skill, args || '', commands, context,
    )

    return {
      data: { success: true, commandName: skill },
      newMessages: processedCommand.messages,
      contextModifier(ctx) {
        // 修改上下文，如设置allowedTools
        return modifiedContext
      }
    }
  }
})
```

这个设计至关重要：它让技能调用成为AI工具调用体系的一部分，而不是特殊的外部命令。AI可以通过统一的方式调用技能、Bash工具、Read工具等。

### 16.5.2 权限系统集成

Skill调用集成在权限系统中：

```typescript
// src/tools/SkillTool/SkillTool.ts
async checkPermissions({ skill, args }, context): Promise<PermissionDecision> {
  const commands = await getAllCommands(context)
  const commandObj = findCommand(skill, commands)

  // 检查deny规则
  const denyRules = getRuleByContentsForTool(
    permissionContext,
    SkillTool as Tool,
    'deny',
  )
  for (const [ruleContent, rule] of denyRules) {
    if (ruleMatches(ruleContent, skill)) {
      return { behavior: 'deny', message: 'Blocked by permission rules' }
    }
  }

  // 检查allow规则
  const allowRules = getRuleByContentsForTool(/* ... */, 'allow')
  for (const [ruleContent, rule] of allowRules) {
    if (ruleMatches(ruleContent, skill)) {
      return { behavior: 'allow', updatedInput: { skill, args } }
    }
  }

  // 自动允许只使用安全属性的技能
  if (skillHasOnlySafeProperties(commandObj)) {
    return { behavior: 'allow' }
  }

  // 默认询问用户
  return {
    behavior: 'ask',
    message: `Execute skill: ${skill}`,
    suggestions: [/* 建议添加allow规则 */],
  }
}
```

`skillHasOnlySafeProperties`确保只有不包含危险操作的技能才能自动放行：

```typescript
const SAFE_SKILL_PROPERTIES = new Set([
  'type', 'name', 'description', 'model', 'effort', 'source',
  'allowedTools', 'disableModelInvocation', /* ... */
])

function skillHasOnlySafeProperties(command: Command): boolean {
  for (const key of Object.keys(command)) {
    if (!SAFE_SKILL_PROPERTIES.has(key)) {
      const value = command[key]
      if (value !== undefined && value !== null) {
        return false // 有非安全属性且有值
      }
    }
  }
  return true
}
```

这个白名单机制确保新增的属性默认需要权限检查，遵循"默认安全"原则。

### 16.5.3 Fork执行上下文

高级技能可以在独立的子Agent中执行（`context: fork`），实现隔离的token预算和工具权限：

```typescript
// src/tools/SkillTool/SkillTool.ts
async function executeForkedSkill(
  command: Command & { type: 'prompt' },
  commandName: string,
  args: string | undefined,
  context: ToolUseContext,
  canUseTool: CanUseToolFn,
  parentMessage: AssistantMessage,
): Promise<ToolResult<Output>> {
  const agentId = createAgentId()
  const { modifiedGetAppState, baseAgent, promptMessages } =
    await prepareForkedCommandContext(command, args || '', context)

  const agentMessages: Message[] = []
  for await (const message of runAgent({
    agentDefinition: baseAgent,
    promptMessages,
    toolUseContext: { ...context, getAppState: modifiedGetAppState },
    canUseTool,
    querySource: 'agent:custom',
    model: command.model,
    override: { agentId },
  })) {
    agentMessages.push(message)
  }

  const resultText = extractResultText(agentMessages, 'Skill execution completed')
  return {
    data: {
      success: true,
      commandName,
      status: 'forked',
      agentId,
      result: resultText,
    },
  }
}
```

Fork上下文让技能像插件一样独立运行，有自己独立的token预算、工具权限和执行环境，不会影响主对话的状态。

## 16.6 小结

Claude Code的Skill与Command系统展示了几个精妙的设计原则：

1. **声明式优于命令式**：技能是提示词模板而非直接代码，这让AI有理解上下文的灵活性
2. **来源透明化**：内置、用户、项目、插件、MCP等不同来源的技能通过统一接口访问
3. **权限的渐进式披露**：通过白名单自动允许安全技能，其他技能需要用户确认
4. **上下文感知激活**：条件技能根据操作路径自动激活，减少用户认知负担
5. **工具统一化**：Skill调用被封装为Tool，融入AI的工具调用体系

这个系统让Claude Code既能开箱即用（内置技能），又能无限扩展（用户技能、插件、MCP），同时保持一致的用户体验和安全边界。斜杠命令成为人与AI协作的自然语言延伸，而非分隔的两种交互模式。

---
