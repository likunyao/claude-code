# 第24章：可扩展性设计 —— 面向未来的架构

> "一个软件系统的寿命与其扩展能力成正比。设计良好的系统不是为当前需求建造的，而是为未来的变化预留空间的。"

Claude Code作为一个AI辅助编程工具，其生命力很大程度上来自于其强大的可扩展性。从插件系统到Feature Flags，从自定义Agent到MCP生态集成，Claude Code展现了一个面向未来的架构应该具备的所有特征。

在 Claude Code 中，"可扩展性"已经不能只理解成“别人可以加一点新功能”。它实际上分成了四个层面：

1. **内容扩展**：Markdown 命令、技能、output styles
2. **协议扩展**：MCP server、resource、tool
3. **运行时扩展**：plugin、workflow、task、bridge session
4. **治理扩展**：managed settings、policy limits、permission rules

## 24.1 插件系统

插件系统是Claude Code可扩展性的核心。它允许第三方开发者扩展功能，而无需修改核心代码库。

### 24.1.1 插件目录结构

一个典型的Claude Code插件具有以下结构：

```
my-plugin/
├── .claude-plugin/
│   └── plugin.json          # 插件清单（可选）
├── commands/                # 自定义斜杠命令
│   ├── build.md
│   └── deploy.md
├── agents/                  # 自定义AI Agents
│   └── code-reviewer.md
├── skills/                  # 自定义Skills
│   └── security-audit.md
├── output-styles/           # 自定义输出样式
│   └── concise.md
├── hooks/                   # Hook配置
│   └── hooks.json
├── .mcp.json                # MCP服务器配置
└── settings.json            # 插件设置
```

这种结构化的目录组织使得插件的功能模块清晰明了。

### 24.1.2 插件不只是“加命令”，而是多入口能力包

插件系统至少会向这些子系统注入内容：

- `commands/`
- `skills/`
- `agents/`
- `hooks/`
- `.mcp.json`
- `output-styles/`
- 用户配置项与变量替换

因此插件更接近一个“微型发行包”，而不是传统 CLI 的单一 command extension。

### 24.1.3 插件清单（Plugin Manifest）

`plugin.json`文件描述了插件的元数据和组件：

```json
{
  "name": "code-assistant",
  "version": "1.2.0",
  "description": "AI驱动的代码辅助工具",
  "author": {
    "name": "Jane Doe",
    "email": "jane@example.com"
  },
  "commands": ["./commands/*.md"],
  "agents": ["./agents/*.md"],
  "skills": ["./skills/*.md"],
  "hooks": "./hooks/hooks.json",
  "mcpServers": "./.mcp.json",
  "userConfig": {
    "apiKey": {
      "type": "string",
      "description": "API密钥",
      "required": true
    }
  }
}
```

### 24.1.4 插件加载流程

插件的加载是一个复杂的过程，涉及多个步骤：

```typescript
// src/utils/plugins/pluginLoader.ts

export async function createPluginFromPath(
  pluginPath: string,
  source: string,
  enabled: boolean,
  fallbackName: string,
): Promise<{ plugin: LoadedPlugin; errors: PluginError[] }> {
  const errors: PluginError[] = []

  // 步骤1：加载或创建插件清单
  const manifest = await loadPluginManifest(
    join(pluginPath, '.claude-plugin', 'plugin.json'),
    fallbackName,
    source
  )

  // 步骤2：检测可选目录
  const [commandsDirExists, agentsDirExists, skillsDirExists] =
    await Promise.all([
      pathExists(join(pluginPath, 'commands')),
      pathExists(join(pluginPath, 'agents')),
      pathExists(join(pluginPath, 'skills')),
    ])

  // 步骤3：创建基础插件对象
  const plugin: LoadedPlugin = {
    name: manifest.name,
    manifest,
    path: pluginPath,
    source,
    enabled,
  }

  // 步骤4：注册组件路径
  if (commandsDirExists) {
    plugin.commandsPath = join(pluginPath, 'commands')
  }
  if (agentsDirExists) {
    plugin.agentsPath = join(pluginPath, 'agents')
  }
  if (skillsDirExists) {
    plugin.skillsPath = join(pluginPath, 'skills')
  }

  // 步骤5：加载Hooks配置
  if (manifest.hooks) {
    plugin.hooksConfig = await loadPluginHooks(...)
  }

  return { plugin, errors }
}
```

### 24.1.5 版本化缓存机制

为了支持插件的版本管理，系统实现了版本化缓存：

```typescript
// 版本化缓存路径格式
// ~/.claude/plugins/cache/{marketplace}/{plugin}/{version}/

export function getVersionedCachePath(
  pluginId: string,
  version: string,
): string {
  const { name: pluginName, marketplace } = parsePluginIdentifier(pluginId)
  const sanitizedMarketplace = marketplace.replace(/[^a-zA-Z0-9\-_]/g, '-')
  const sanitizedPlugin = pluginName.replace(/[^a-zA-Z0-9\-_]/g, '-')
  const sanitizedVersion = version.replace(/[^a-zA-Z0-9\-_.]/g, '-')

  return join(
    getPluginsDirectory(),
    'cache',
    sanitizedMarketplace,
    sanitizedPlugin,
    sanitizedVersion,
  )
}
```

这种设计允许同一插件的不同版本并存，便于版本回滚和A/B测试。

### 24.1.6 插件源类型

插件可以从多种来源加载：

```typescript
type PluginSource =
  | string                    // 本地路径
  | { source: 'npm', package: string, version?: string }
  | { source: 'github', repo: string, ref?: string, sha?: string }
  | { source: 'url', url: string, ref?: string, sha?: string }
  | { source: 'git-subdir', url: string, path: string, ref?: string }
```

每种来源都有对应的安装逻辑。例如，对于git-subdir类型，系统使用sparse-checkout只下载需要的子目录：

```typescript
// src/utils/plugins/pluginLoader.ts

export async function installFromGitSubdir(
  url: string,
  targetPath: string,
  subdirPath: string,
  ref?: string,
  sha?: string,
): Promise<string | undefined> {
  // 使用partial clone + sparse-checkout只下载子目录
  const cloneArgs = [
    'clone',
    '--depth', '1',
    '--filter=tree:0',     // 部分克隆
    '--no-checkout',
  ]
  if (ref) {
    cloneArgs.push('--branch', ref)
  }

  // 稀疏检出特定子目录
  await execFileNoThrow(gitExe(), [
    'sparse-checkout', 'set', '--cone', '--', subdirPath
  ])

  // 移动子目录到目标位置
  await rename(resolvedSubdir, targetPath)

  return resolvedSha
}
```

这种优化对于大型monorepo插件特别有价值，可以显著减少下载时间和存储空间。

### 24.1.7 插件命令的真正装载方式：扫描 Markdown，再转成 Command

`src/utils/plugins/loadPluginCommands.ts` 展示了一个非常有代表性的设计：插件命令和插件技能都可以直接来自 Markdown 文件。

加载器会：

1. 递归扫描插件目录
2. 特判 `SKILL.md` 目录式技能
3. 解析 frontmatter
4. 根据目录层次生成带插件名前缀的命名空间
5. 提取描述、allowed-tools、参数占位符、shell frontmatter 等信息
6. 最终组装成统一的 `Command`

这个方案的价值在于：

- 扩展开发门槛低
- 提示词与元数据共址
- 插件系统与技能系统共享一套内容管线

从架构上看，Claude Code 是把“扩展”做成了内容编译问题，而不是只做成代码加载问题。

Feature Flags、平台检测与能力裁剪，构成了插件体系之外更深的一层扩展能力。

如果只写插件，仍然不足以解释 Claude Code 的扩展性。更深的一层是：**系统能力是运行时裁剪出来的。**

`tools.ts`、`commands.ts`、`main.tsx` 里到处都有这类逻辑：

- `feature('AGENT_TRIGGERS')`
- `feature('WEB_BROWSER_TOOL')`
- `process.env.USER_TYPE === 'ant'`
- `isTodoV2Enabled()`
- `isWorktreeModeEnabled()`

Claude Code 不是把所有可选功能都硬编码进一个固定发行版，而是：

1. 编译期做一轮 dead code elimination
2. 运行期再根据用户类型、平台、订阅和实验开关做二次裁剪

于是“扩展性”不只是对外开放接口，也包括 **对内保持产品线分化的能力**。这对一个既有公开版本、又有内部版本、还持续做实验开关的系统尤其重要。

### 24.1.8 插件合并策略

当多个源提供相同插件时，系统有明确的优先级规则：

```typescript
// src/utils/plugins/pluginLoader.ts

export function mergePluginSources(sources: {
  session: LoadedPlugin[]      // --plugin-dir指定的会话插件
  marketplace: LoadedPlugin[]   // 已安装的市场插件
  builtin: LoadedPlugin[]      // 内置插件
  managedNames?: Set<string>   // 企业策略锁定的插件
}): { plugins: LoadedPlugin[]; errors: PluginError[] } {
  // 企业策略优先于一切
  const sessionPlugins = sources.session.filter(p => {
    if (managed?.has(p.name)) {
      errors.push({
        type: 'generic-error',
        source: p.source,
        error: `插件"${p.name}"被托管设置锁定`,
      })
      return false
    }
    return true
  })

  // 会话插件覆盖已安装插件
  const marketplacePlugins = sources.marketplace.filter(p => {
    return !sessionNames.has(p.name)
  })

  // 最终顺序：会话 > 市场 > 内置
  return {
    plugins: [...sessionPlugins, ...marketplacePlugins, ...sources.builtin],
    errors,
  }
}
```

### 24.1.9 企业策略支持

插件系统内置了企业策略支持，允许组织管理员控制可用插件：

```typescript
// 白名单模式：只允许列表中的来源
const strictAllowlist = getStrictKnownMarketplaces()

// 黑名单模式：禁止列表中的来源
const blocklist = getBlockedMarketplaces()

// 失败关闭：如果无法验证来源且存在策略，则阻止
if (!marketplaceConfig && hasEnterprisePolicy) {
  errors.push({
    type: 'marketplace-blocked-by-policy',
    source: pluginId,
    plugin: pluginName,
    marketplace: marketplaceName,
    blockedByBlocklist: strictAllowlist === null,
    allowedSources: (strictAllowlist ?? []).map(formatSourceForDisplay),
  })
  return null
}
```

## 24.2 Feature Flags编译时开关

Feature Flags（功能开关）是渐进式发布和A/B测试的基础设施。Claude Code使用多层Feature Flag系统。

### 24.2.1 Bun Bundle编译时特性

```typescript
// src/utils/envDynamic.ts

import { feature } from 'bun:bundle'

function isMuslEnvironment(): boolean {
  // 在原生Linux构建中，这些在编译时静态已知
  if (feature('IS_LIBC_MUSL')) return true
  if (feature('IS_LIBC_GLIBC')) return false

  // Node环境的回退：运行时检测
  return muslRuntimeCache ?? false
}
```

`feature()`函数在打包时被替换为常量，使得条件分支在运行时完全消除。

### 24.2.2 GrowthBook远程配置

```typescript
// src/services/analytics/growthbook.ts

let client: GrowthBook | null = null

export async function initGrowthBook(): Promise<void> {
  client = new GrowthBook({
    apiHost: 'https://cdn.growthbook.io',
    clientKey: getGrowthBookClientKey(),
    attributes: getUserAttributes(),
  })

  await client.loadFeatures()
}

export function getFeatureValue_CACHED_MAY_BE_STALE<T>(
  key: string,
  defaultValue: T,
): T {
  // 允许在init完成前访问，返回缓存值或默认值
  const envOverride = hasGrowthBookEnvOverride(key)
  if (envOverride) {
    return getEnvOverrides()?.[key] as T ?? defaultValue
  }

  return client?.evalFeature(key)?.value ?? defaultValue
}
```

`_CACHED_MAY_BE_STALE`后缀表明这个函数可能在Feature Flag初始化完成前被调用，返回的可能是缓存值或默认值。

### 24.2.3 环境变量覆盖

为了便于内部测试，系统支持通过环境变量覆盖Feature Flag：

```typescript
// src/services/analytics/growthbook.ts

function getEnvOverrides(): Record<string, unknown> | null {
  if (process.env.USER_TYPE !== 'ant') {
    return null  // 仅内部用户可用
  }

  const raw = process.env.CLAUDE_INTERNAL_FC_OVERRIDES
  if (!raw) return null

  try {
    return JSON.parse(raw)
  } catch {
    logError(new Error(`解析CLAUDE_INTERNAL_FC_OVERRIDES失败: ${raw}`))
    return null
  }
}

export function hasGrowthBookEnvOverride(feature: string): boolean {
  const overrides = getEnvOverrides()
  return overrides !== null && feature in overrides
}
```

这种机制允许在不需要更改远程配置的情况下快速测试不同的Feature Flag组合。

### 24.2.4 Feature Flag的分层决策

Claude Code使用分层决策来决定使用哪个Feature Flag值：

```
1. 环境变量覆盖 (CLAUDE_INTERNAL_FC_OVERRIDES)
   ↓
2. GrowthBook远程配置
   ↓
3. 磁盘缓存
   ↓
4. 编译时常量 (bun:bundle feature())
   ↓
5. 默认值
```

这种分层确保了在所有情况下都有合理的默认行为。

## 24.3 治理面扩展：managed settings、policy limits、permission rules

可扩展性并不只体现为“功能扩展”。当前源码新增的 `remoteManagedSettings` 和 `policyLimits` 表明，连系统治理面也在被模块化：

- 远程托管设置负责组织级配置注入
- 策略限制负责组织级功能裁剪
- 本地权限规则负责单次操作授权

这三者彼此独立又互相叠加，使 Claude Code 能同时服务：

- 个人开发者的本地自由配置
- 团队/企业的集中治理
- 实验功能和产品分层发布

因此，Claude Code 的可扩展性，不只是“能加东西”，而是“能在不同用户、不同组织、不同产品线之间，重组出不同的系统形态”。

## 24.4 自定义Agent扩展(.claude/agents/)

除了插件系统，Claude Code还支持通过`.claude/agents/`目录添加自定义Agent。

### 24.4.1 Agent定义格式

```markdown
<!-- .claude/agents/code-reviewer.md -->

You are a code review specialist with expertise in:
- Security vulnerabilities
- Performance optimization
- Code maintainability

When reviewing code:
1. Check for common security issues
2. Identify performance bottlenecks
3. Suggest improvements for readability

Always provide specific, actionable feedback.
```

### 24.4.2 Agent加载机制

```typescript
// src/tools/AgentTool/loadAgentsDir.ts

export async function loadAgentsFromDir(
  agentsDir: string,
): Promise<AgentDefinition[]> {
  const agents: AgentDefinition[] = []

  const entries = await readdir(agentsDir, { withFileTypes: true })
  for (const entry of entries) {
    if (!entry.isFile() || !entry.name.endsWith('.md')) {
      continue
    }

    const filePath = join(agentsDir, entry.name)
    const content = await readFile(filePath, 'utf-8')
    const name = entry.name.replace('.md', '')

    agents.push({
      name,
      description: extractDescription(content),
      prompt: content,
      source: `file:${filePath}`,
    })
  }

  return agents
}
```

### 24.4.3 与插件Agent的对比

| 特性 | .claude/agents/ | 插件Agents |
|------|-----------------|------------|
| 位置 | 项目本地 | 插件目录 |
| 分发 | Git仓库 | Marketplace |
| 作用域 | 单项目 | 全局 |
| 更新 | Git pull | 插件更新 |
| 版本控制 | Git | 插件版本 |

两者各有优势，选择取决于具体的使用场景。

## 24.5 配置热更新

配置的动态更新能力对于长期运行的会话至关重要。

### 24.5.1 设置层叠系统

Claude Code使用多层级设置系统：

```
1. 插件设置 (plugin.settings)
   ↓
2. 项目设置 (.claude/settings.json)
   ↓
3. 用户全局设置 (~/.claude/settings.json)
   ↓
4. 托管设置 (企业策略)
   ↓
5. 默认值
```

### 24.5.2 设置缓存与失效

```typescript
// src/utils/plugins/pluginLoader.ts

export function cachePluginSettings(plugins: LoadedPlugin[]): void {
  const settings = mergePluginSettings(plugins)
  setPluginSettingsBase(settings)

  // 只有实际有插件设置时才重置缓存
  if (settings && Object.keys(settings).length > 0) {
    resetSettingsCache()
    logForDebugging(`缓存插件设置: ${Object.keys(settings).join(', ')}`)
  }
}

export function clearPluginCache(reason?: string): void {
  loadAllPlugins.cache?.clear?.()
  loadAllPluginsCacheOnly.cache?.clear?.()

  // 清理插件设置缓存
  if (getPluginSettingsBase() !== undefined) {
    resetSettingsCache()
  }
  clearPluginSettingsBase()
}
```

### 24.5.3 配置变更通知

```typescript
// src/services/analytics/growthbook.ts

type GrowthBookRefreshListener = () => void | Promise<void>
const refreshed = createSignal()

export function onGrowthBookRefresh(
  listener: GrowthBookRefreshListener,
): () => void {
  return refreshed.subscribe(listener)
}

// 当Feature Flag更新时
function notifyRefreshed(): void {
  refreshed.send()
}
```

这种发布-订阅模式允许系统各部分对配置变更做出响应。

## 24.6 MCP生态集成

Model Context Protocol (MCP) 是Claude Code扩展能力的重要组成部分。

### 24.6.1 MCP服务器配置

```json
{
  "mcpServers": {
    "filesystem": {
      "type": "stdio",
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-filesystem", "/allowed/path"],
      "env": {
        "PATH": "${env:PATH}"
      }
    },
    "github": {
      "type": "stdio",
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": {
        "GITHUB_TOKEN": "${user_config.github_token}"
      }
    }
  }
}
```

### 24.6.2 插件MCP服务器加载

```typescript
// src/utils/plugins/mcpPluginIntegration.ts

export async function loadPluginMcpServers(
  plugin: LoadedPlugin,
  errors: PluginError[] = [],
): Promise<Record<string, McpServerConfig> | undefined> {
  let servers: Record<string, McpServerConfig> = {}

  // 检查.mcp.json文件（最低优先级）
  const defaultMcpServers = await loadMcpServersFromFile(
    plugin.path,
    '.mcp.json',
  )
  if (defaultMcpServers) {
    servers = { ...servers, ...defaultMcpServers }
  }

  // 处理manifest.mcpServers（较高优先级）
  if (plugin.manifest.mcpServers) {
    const mcpServersSpec = plugin.manifest.mcpServers

    if (typeof mcpServersSpec === 'string') {
      // 单个MCPB文件或JSON路径
      if (isMcpbSource(mcpServersSpec)) {
        const mcpbServers = await loadMcpServersFromMcpb(
          plugin,
          mcpServersSpec,
          errors,
        )
        if (mcpbServers) {
          servers = { ...servers, ...mcpbServers }
        }
      }
    } else if (Array.isArray(mcpServersSpec)) {
      // 并行加载多个规范
      const results = await Promise.all(
        mcpServersSpec.map(async spec => {
          if (typeof spec === 'string' && isMcpbSource(spec)) {
            return await loadMcpServersFromMcpb(plugin, spec, errors)
          }
          return await loadMcpServersFromFile(plugin.path, spec)
        })
      )
      for (const result of results) {
        if (result) {
          servers = { ...servers, ...result }
        }
      }
    }
  }

  return servers
}
```

### 24.6.3 环境变量解析

```typescript
// src/utils/plugins/mcpPluginIntegration.ts

export function resolvePluginMcpEnvironment(
  config: McpServerConfig,
  plugin: { path: string; source: string },
  userConfig?: UserConfigValues,
): McpServerConfig {
  const resolveValue = (value: string): string => {
    // 首先替换插件特定变量
    let resolved = substitutePluginVariables(value, plugin)

    // 然后替换用户配置变量
    if (userConfig) {
      resolved = substituteUserConfigVariables(resolved, userConfig)
    }

    // 最后展开通用环境变量
    const { expanded } = expandEnvVarsInString(resolved)
    return expanded
  }

  switch (config.type) {
    case 'stdio': {
      return {
        ...config,
        command: config.command ? resolveValue(config.command) : undefined,
        args: config.args?.map(arg => resolveValue(arg)),
        env: {
          CLAUDE_PLUGIN_ROOT: plugin.path,
          CLAUDE_PLUGIN_DATA: getPluginDataDir(plugin.source),
          ...(config.env || {}),
        }
      }
    }
    // ... 其他类型
  }
}
```

### 24.6.4 作用域隔离

为了避免不同插件的MCP服务器名称冲突，系统使用作用域前缀：

```typescript
// src/utils/plugins/mcpPluginIntegration.ts

export function addPluginScopeToServers(
  servers: Record<string, McpServerConfig>,
  pluginName: string,
  pluginSource: string,
): Record<string, ScopedMcpServerConfig> {
  const scopedServers: Record<string, ScopedMcpServerConfig> = {}

  for (const [name, config] of Object.entries(servers)) {
    const scopedName = `plugin:${pluginName}:${name}`
    scopedServers[scopedName] = {
      ...config,
      scope: 'dynamic',
      pluginSource,
    }
  }

  return scopedServers
}
```

这样，即使两个插件都提供名为"database"的MCP服务器，它们也不会冲突。

### 24.6.5 MCPB（MCP Bundle）格式

MCPB是一种打包格式，允许插件发布者提供预配置的MCP服务器：

```typescript
// src/utils/plugins/mcpbHandler.ts

export async function loadMcpbFile(
  mcpbPath: string,
  pluginPath: string,
  pluginId: string,
  statusCallback: (status: string) => void,
): Promise<McpbLoadResult | McpbNeedsConfigResult> {
  // 1. 下载MCPB文件（如果是URL）
  const localPath = mcpbPath.startsWith('http')
    ? await downloadMcpb(mcpbPath, statusCallback)
    : mcpbPath

  // 2. 解析DXT清单
  const manifest = await parseManifest(localPath)

  // 3. 检查用户配置需求
  if (manifest.userConfig && !isUserConfigured(pluginId, manifest.name)) {
    return { status: 'needs-config', manifest }
  }

  // 4. 提取并转换DXT清单为MCP配置
  const extractedPath = await extractMcpb(localPath)
  const mcpConfig = await convertManifestToMcpConfig(manifest)

  return {
    status: 'success',
    manifest,
    mcpConfig,
    extractedPath,
  }
}
```

## 24.7 全书总结与展望

本书通过分析Claude Code的源码，探讨了现代TypeScript应用的多个设计维度：

### 24.7.1 架构设计原则

1. **分层清晰**：从工具层到服务层，从业务逻辑到基础设施，每一层都有明确的职责
2. **接口优先**：通过定义清晰的接口实现模块间的松耦合
3. **可测试性**：通过依赖注入、模块化设计等提高代码的可测试性

### 24.7.2 并发与异步处理

1. **取消传播**：通过AbortController和WeakRef实现内存安全的取消层次结构
2. **并行执行**：在独立操作上使用Promise.all，在有依赖的操作上保持串行
3. **容错设计**：通过Promise.allSettled等模式确保部分失败不影响整体

### 24.7.3 可扩展性架构

1. **插件系统**：支持多源插件加载、版本化缓存、企业策略
2. **Feature Flags**：编译时开关与运行时配置相结合
3. **MCP集成**：通过标准协议扩展AI能力

### 24.7.4 面向未来的思考

Claude Code的设计展现了对未来变化的适应性：

- **协议化**：通过MCP等标准化协议，避免与特定实现绑定
- **渐进增强**：新功能通过Feature Flags逐步推出，允许快速迭代
- **社区生态**：插件系统和自定义Agent允许社区贡献扩展

### 24.7.5 对软件工程的启示

1. **设计是演进的**：没有完美的初始设计，重要的是建立能够演进的架构
2. **简单胜于复杂**：复杂的特性（如WeakRef取消传播）服务于简单的用户需求（取消操作）
3. **工具服务于人**：所有技术选择最终都应指向更好的用户体验

正如《C++编程思想》所强调的，优秀的软件设计不仅仅是关于"如何做"，更是关于"为何这样做"。Claude Code的源码为我们提供了一个活生生的案例，展示了如何在真实项目中平衡各种技术约束，构建出既强大又优雅的系统。

未来，随着AI技术的不断演进，我们可以期待更多创新的设计模式出现。但无论技术如何变化，一些基本原则将始终适用：

- **清晰性**：代码应该是自解释的
- **模块化**：系统应该由可独立理解和测试的模块组成
- **可扩展性**：新功能应该能够添加而不需要修改现有代码
- **容错性**：系统应该能够优雅地处理错误

愿这本书能够帮助读者在自己的项目中应用这些原则，构建出更好的软件系统。

---

*本章基于Claude Code源码分析，主要参考文件：*
- `src/plugins/` - 插件系统核心
- `src/utils/plugins/` - 插件加载器与工具
- `src/utils/plugins/mcpPluginIntegration.ts` - MCP集成
- `src/services/analytics/growthbook.ts` - Feature Flag系统
- `src/utils/envDynamic.ts` - 编译时特性检测
- `src/tools/AgentTool/loadAgentsDir.ts` - 自定义Agent加载
