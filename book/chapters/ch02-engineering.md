# 第2章：工程基础 —— 构建系统与项目结构

> "优秀的工程不是复杂的堆砌，而是简单的坚持。当每一行代码都恰如其分，整个系统自然清晰可读。"

---

## 目录

2.1 [Bun 运行时与构建管线](#21-bun-运行时与构建管线)
2.2 [死代码消除：feature() 函数的多态构建](#22-死代码消除feature-函数的多态构建)
2.3 [模块化设计：src/ 目录的职责划分](#23-模块化设计src-目录的职责划分)
2.4 [类型系统哲学：TypeScript 类型定义的组织](#24-类型系统哲学typescript-类型定义的组织)
2.5 [常量管理：constants/ 的分层策略](#25-常量管理constants-的分层策略)
2.6 [小结与思考](#26-小结与思考)

---

## 2.1 Bun 运行时与构建管线

Claude Code 选择了 Bun 作为运行时和构建工具，这个决策背后有深刻的技术考量。Bun 不仅是一个 JavaScript 运行时，更是一个完整的工具链，将运行时、打包器、测试框架整合在一起。

### 2.1.1 bun:bundle：静态分析与死代码消除

Bun 最强大的特性之一是其内置的 `bun:bundle` 模块，它提供了编译时的特性检测功能。在 Claude Code 的代码库中，这个功能被广泛应用于特性门控（feature flags）：

```typescript
// src/entrypoints/cli.tsx:1
import { feature } from 'bun:bundle';

// src/constants/prompts.ts:48
import { feature } from 'bun:bundle'

// 死代码消除：条件导入
const getCachedMCConfigForFRC = feature('CACHED_MICROCOMPACT')
  ? (
      require('../services/compact/cachedMCConfig.js') as typeof import('../services/compact/cachedMCConfig.js')
    ).getCachedMCConfig
  : null
```

`feature()` 函数在编译时被求值，这意味着：
- 如果 `CACHED_MICROCOMPACT` 特性在编译时被禁用，整个模块路径都不会被打包进最终产物
- 这种机制不仅减少了包体积，还确保了实验性功能不会泄露到生产版本中

### 2.1.2 快速启动：零模块加载的快速路径

入口文件 `src/entrypoints/cli.tsx` 展示了性能优先的设计哲学。它实现了 10+ 种快速路径，每种路径都使用动态导入，只加载最少量的模块：

```typescript
// src/entrypoints/cli.tsx:33-42
async function main(): Promise<void> {
  const args = process.argv.slice(2);

  // Fast-path for --version/-v: zero module loading needed
  if (args.length === 1 && (args[0] === '--version' || args[0] === '-v' || args[0] === '-V')) {
    console.log(`${MACRO.VERSION} (Claude Code)`);
    return;
  }
  // ...
}
```

**cli.tsx 的快速路径清单**（按代码顺序）：

| 快速路径 | 触发条件 | feature() 门控 | 加载模块数 |
|----------|---------|----------------|-----------|
| `--version` | `args[0]` 匹配 | 无 | 0（零模块） |
| `--dump-system-prompt` | 内部评测工具 | `DUMP_SYSTEM_PROMPT` | 3（config, model, prompts） |
| `--claude-in-chrome-mcp` | Chrome 扩展 MCP | 无 | 1（mcpServer） |
| `--chrome-native-host` | Chrome 原生消息 | 无 | 1（chromeNativeHost） |
| `--computer-use-mcp` | 计算机使用 MCP | `CHICAGO_MCP` | 1（mcpServer） |
| `--daemon-worker` | 守护进程工作器 | `DAEMON` | 1（workerRegistry） |
| `remote-control/rc/bridge` | 远程控制模式 | `BRIDGE_MODE` | 5+（auth, bridge, policy） |
| `daemon` 子命令 | 守护进程管理 | `DAEMON` | 3（config, sinks, daemon） |
| `ps/logs/attach/kill/--bg` | 会话管理 | `BG_SESSIONS` | 2（config, bg） |
| `new/list/reply` | 模板任务 | `TEMPLATES` | 1（templateJobs） |
| `environment-runner` | BYOC 环境运行器 | `BYOC_ENVIRONMENT_RUNNER` | 1（environmentRunner） |
| `self-hosted-runner` | 自托管运行器 | `SELF_HOSTED_RUNNER` | 1（selfHostedRunner） |
| `--tmux + --worktree` | tmux 工作树 | 无 | 2（config, worktree） |
| 默认路径 | 无特殊标志 | 无 | 2+（earlyInput, main） |

**快速路径的架构模式**：

1. **feature() 内联检查**：`feature('DAEMON')` 等调用在编译时求值，外部构建中整个 if 分支被消除
2. **最小化 enableConfigs()**：只有需要配置的路径才调用 `enableConfigs()`，daemon-worker 等精简路径完全跳过
3. **profileCheckpoint 穿插**：每个快速路径入口都标记检查点，用于启动性能分析
4. **错误穿透**：tmux+worktree 路径失败时可以穿透到默认路径，提供优雅降级

### 2.1.3 动态导入：按需加载

对于非常规路径，系统使用动态导入来延迟模块加载：

```typescript
// src/entrypoints/cli.tsx:44-48
// For all other paths, load the startup profiler
const {
  profileCheckpoint
} = await import('../utils/startupProfiler.js');
profileCheckpoint('cli_entry');
```

这种模式让启动时间最小化，只有在真正需要某个功能时才加载相关代码。

### 2.1.4 多入口点架构

Claude Code 支持多种运行模式，每种模式都有独立的入口点：

```
src/entrypoints/
├── cli.tsx           # CLI 主入口
├── init.ts           # 初始化逻辑
├── mcp.ts            # MCP 服务器入口
├── sandboxTypes.ts   # 沙箱类型定义
├── agentSdkTypes.ts  # Agent SDK 类型
└── sdk/              # SDK 入口点
```

这种多入口点设计使得同一个代码库可以支持：
- CLI 运行模式
- MCP 服务器模式
- SDK 嵌入模式
- 沙箱隔离模式

> **设计思想 1：快速路径与延迟加载**
>
> - 将最常见的操作（如 --version）放在快速路径中，零模块加载
> - 使用动态导入延迟非关键路径的模块加载
> - 多入口点架构让同一代码库支持多种运行模式

---

## 2.2 死代码消除：feature() 函数的多态构建

`feature()` 函数是 Claude Code 构建系统的核心，它实现了编译时的特性门控和死代码消除。

### 2.2.1 特性门控模式

在代码库中，`feature()` 被用于控制各种实验性功能的启用：

```typescript
// src/constants/prompts.ts:64-98（简化）
const getCachedMCConfigForFRC = feature('CACHED_MICROCOMPACT')
  ? require('../services/compact/cachedMCConfig.js').getCachedMCConfig
  : null

const proactiveModule =
  feature('PROACTIVE') || feature('KAIROS')
    ? require('../proactive/index.js')
    : null

const BRIEF_PROACTIVE_SECTION: string | null =
  feature('KAIROS') || feature('KAIROS_BRIEF')
    ? require('../tools/BriefTool/prompt.js').BRIEF_PROACTIVE_SECTION
    : null
```

### 2.2.2 条件导入的类型安全

为了保持类型安全，条件导入使用了类型断言：

```typescript
require('../services/compact/cachedMCConfig.js') as typeof import('../services/compact/cachedMCConfig.js')
```

这种模式确保：
- 即使是动态导入，TypeScript 也能正确推断类型
- IDE 可以提供完整的类型提示和自动补全
- 重构时可以安全地移动和重命名模块

### 2.2.3 实验性功能的隔离

从 `src/utils/betas.ts` 可以看到复杂的特性门控逻辑：

```typescript
// src/utils/betas.ts:161-195
export function modelSupportsAutoMode(model: string): boolean {
  if (feature('TRANSCRIPT_CLASSIFIER')) {
    const m = getCanonicalName(model)
    // External: firstParty-only at launch
    if (process.env.USER_TYPE !== 'ant' && getAPIProvider() !== 'firstParty') {
      return false
    }
    // GrowthBook override: tengu_auto_mode_config.allowModels
    const config = getFeatureValue_CACHED_MAY_BE_STALE<{
      allowModels?: string[]
    }>('tengu_auto_mode_config', {})
    // ...
  }
  return false
}
```

这种多层门控确保：
- 实验性功能只在特定构建中启用
- 外部版本不会包含内部功能
- 功能可以独立于代码发布进行控制

### 2.2.4 构建时优化的效果

通过 `feature()` 的死代码消除，Claude Code 实现了：

1. **包体积优化**：实验性功能不会增加发布包的大小
2. **安全隔离**：内部功能不会意外泄露到外部版本
3. **A/B 测试支持**：同一代码库可以构建出不同功能的版本
4. **渐进式发布**：新功能可以逐步向用户开放

> **设计思想 2：编译时特性门控**
>
> - 使用 `feature()` 函数在编译时决定代码是否包含在构建中
> - 实验性功能完全隔离，不会影响稳定版本
> - 类型安全的条件导入保持开发体验
> - 支持多态构建：同一代码库生成不同功能的版本

---

## 2.3 模块化设计：src/ 目录的职责划分

Claude Code 的 `src/` 目录包含了约 55 个顶级目录/文件，展现了一个高度模块化的架构。

### 2.3.1 目录结构全景

```
src/
├── assistant/          # Assistant 模式（Kairos）
├── bootstrap/          # 启动时状态管理
├── bridge/             # 远程桥接功能（33个文件）
├── buddy/              # 协作功能
├── cli/                # CLI 特定逻辑
├── commands/           # 103+ 命令实现
├── components/         # 146+ React 组件
├── constants/          # 常量定义（21个文件）
├── context/            # React Context
├── coordinator/        # 协调器模式
├── entrypoints/        # 程序入口点
├── hooks/              # 87+ React Hooks
├── ink/                # 50+ 终端 UI 组件
├── keybindings/        # 键盘绑定
├── main.tsx            # 主入口文件（80万+字符）
├── memdir/             # 记忆目录
├── migrations/         # 数据迁移
├── moreright/          # MoreRight 集成
├── native-ts/          # 原生 TypeScript 绑定
├── outputStyles/       # 输出样式
├── plugins/            # 插件系统
├── query.ts            # 查询引擎核心（约1700行）
├── QueryEngine.ts      # 查询引擎实现（约1400行）
├── remote/             # 远程执行
├── schemas/            # JSON Schema
├── screens/            # 屏幕组件
├── server/             # 服务器功能
├── services/           # 38+ 服务模块
├── setup.ts            # 启动流程
├── skills/             # 技能系统
├── state/              # 状态管理
├── tasks/              # 后台任务系统
├── tools/              # 38+ 工具实现
├── types/              # 类型定义（7个.ts文件 + generated/）
├── utils/              # 331+ 工具函数
├── vim/                # Vim 集成
└── voice/              # 语音模式
```

### 2.3.2 核心模块详解

#### tools/：工具实现的动物园

每个工具都是一个独立的目录，包含完整的实现：

```
src/tools/
├── AgentTool/              # 子 Agent 管理（17个文件）
├── BashTool/               # Shell 命令执行（20个文件）
├── FileReadTool/           # 文件读取
├── FileEditTool/           # 文件编辑
├── FileWriteTool/          # 文件写入
├── GlobTool/               # 文件搜索
├── GrepTool/               # 内容搜索
├── SkillTool/              # 技能执行
├── PowerShellTool/         # PowerShell 支持（16个文件）
├── LSPTool/                # 语言服务器协议
├── MCPTool/                # MCP 服务器调用
├── AskUserQuestionTool/    # 用户交互
├── REPLTool/               # REPL 执行
├── ScheduleCronTool/       # 定时任务
├── SendMessageTool/        # 消息发送
├── EnterWorktreeTool/      # Worktree 管理
└── ...
```

**工具目录结构模式**：
```
tools/ExampleTool/
├── prompt.ts           # 工具描述和系统提示词
├── types.ts            # 工具特定类型
├── constants.ts        # 工具常量
├── ExampleTool.tsx     # 主实现
└── ...
```

#### commands/：命令实现的集市

与 tools 不同，commands 是用户可调用的命令：

```
src/commands/（103+ 个命令）
├── commit/             # Git commit 命令
├── review/             # 代码审查命令
├── plan/               # 计划模式命令
├── compact/            # 上下文压缩
├── clear/              # 清除会话
├── help/               # 帮助系统
├── share/              # 会话分享
├── issue/              # 问题报告
├── plugins/            # 插件管理
├── rename/             # 重命名
└── ...
```

#### components/：React 组件的花园

虽然是在终端运行，Claude Code 使用完整的 React 架构：

```
src/components/（146+ 组件）
├── Messages.tsx        # 消息列表容器
├── Message.tsx         # 单条消息渲染
├── PromptInput/        # 输入框组件
├── Spinner.tsx         # 加载动画
├── StructuredDiff.tsx  # 代码差异显示
├── TaskListV2.tsx      # 任务列表
├── ModelPicker.tsx     # 模型选择器
├── ErrorBoundary.tsx   # 错误边界
└── ...
```

#### services/：后台服务的机房

服务层封装了复杂的业务逻辑：

```
src/services/（38+ 服务）
├── compact/            # 上下文压缩服务
├── mcp/                # MCP 协议实现
├── api/                # Claude API 调用
├── analytics/          # 遥测和分析
├── skillSearch/        # 技能搜索
├── tools/              # 工具执行编排
├── lsp/                # LSP 客户端
└── ...
```

#### utils/：工具函数的海洋

`utils/` 目录包含 331+ 个工具函数文件，总计约 88,466 行代码。这是一个精心组织的工具库：

```
src/utils/（部分示例）
├── api.ts              # API 调用
├── auth.ts             # 认证
├── bash/               # Bash 相关工具
├── cwd.ts              # 当前工作目录
├── env.ts              # 环境变量
├── git.ts              # Git 操作
├── model/              # 模型管理
├── permissions/        # 权限系统
├── settings/           # 设置管理
├── plugins/            # 插件工具
└── ...
```

### 2.3.3 模块化的设计原则

1. **单一职责**：每个目录/模块都有明确的职责边界
2. **低耦合**：模块间通过类型接口通信，减少直接依赖
3. **高内聚**：相关功能聚合在同一模块内
4. **可测试性**：模块化设计便于单元测试

### 2.3.4 init.ts：启动流程的桥梁文件

`src/entrypoints/init.ts`（341行）是连接入口分派与业务运行的关键桥梁。它使用 `memoize` 确保只执行一次，在信任对话框之前完成所有系统级初始化：

```typescript
// src/entrypoints/init.ts:57
export const init = memoize(async (): Promise<void> => {
  const initStartTime = Date.now()
  // ... 15+ 项初始化任务
})
```

**init.ts 的 15+ 项初始化任务按依赖顺序排列**：

| 阶段 | 任务 | 依赖说明 |
|------|------|----------|
| 配置 | `enableConfigs()` | 所有后续任务的前提 |
| 环境 | `applySafeConfigEnvironmentVariables()` | 信任对话框之前的安全变量 |
| 环境 | `applyExtraCACertsFromConfig()` | TLS 连接之前，Bun 缓存 TLS 证书 |
| 生命周期 | `setupGracefulShutdown()` | 最早注册，确保后续步骤能被清理 |
| 分析 | `initialize1PEventLogging()` | 异步，不阻塞主流程 |
| 认证 | `populateOAuthAccountInfoIfNeeded()` | 异步，预填充 OAuth 缓存 |
| IDE | `initJetBrainsDetection()` | 异步，填充 JetBrains 缓存 |
| Git | `detectCurrentRepository()` | 异步，填充 Git 仓库缓存 |
| 远程 | `initializeRemoteManagedSettingsLoadingPromise()` | 条件执行，带超时防死锁 |
| 策略 | `initializePolicyLimitsLoadingPromise()` | 条件执行 |
| 记录 | `recordFirstStartTime()` | 标记首次启动时间 |
| 安全 | `configureGlobalMTLS()` | mTLS 双向认证配置 |
| 网络 | `configureGlobalAgents()` | 全局 HTTP 代理（依赖 mTLS） |
| 网络 | `preconnectAnthropicApi()` | TCP+TLS 预连接（~100-200ms 优化） |
| 代理 | `initUpstreamProxy()` | CCR 远程模式专用，条件执行 |
| 平台 | `setShellIfWindows()` | Windows git-bash 设置 |
| 清理 | `registerCleanup(shutdownLspServerManager)` | LSP 管理器清理 |
| 清理 | `registerCleanup(cleanupSessionTeams)` | Team 资源清理 |
| 存储 | `ensureScratchpadDir()` | 临时目录初始化 |

**设计亮点**：
- **渐进式加载**：分析、认证、IDE 检测等用 `void` 异步触发（fire-and-forget），不阻塞启动
- **profileCheckpoint 穿插**：每步之后插入性能检查点，便于定位启动瓶颈
- **错误隔离**：外层 try/catch 只捕获 `ConfigParseError`，其他错误上抛给调用者
- **信任前/后分离**：安全环境变量在信任对话框之前应用，完整变量在之后

> **设计思想 3：模块化与职责分离**
>
> - 按功能领域划分模块，而非按技术层次
> - 每个模块都有明确的边界和接口
> - 工具、命令、组件三者分离，各司其职
> - 服务层封装复杂逻辑，保持上层简洁

---

## 2.4 类型系统哲学：TypeScript 类型定义的组织

Claude Code 的类型系统展现了大型 TypeScript 项目的最佳实践。

### 2.4.1 types/ 目录结构

`src/types/` 目录极其精简，只包含 7 个手写的 `.ts` 文件和一个 `generated/` 子目录：

```
src/types/
├── command.ts          # 命令类型（217行）— PromptCommand, LocalCommand, LocalJSXCommand 联合类型
├── hooks.ts            # Hooks 配置与生命周期类型
├── ids.ts              # UUID 与各类 ID 生成类型
├── logs.ts             # 日志与持久化类型（331行）
├── permissions.ts      # 权限系统类型（PermissionMode, 权限规则等）
├── plugin.ts           # 插件类型（364行）— PluginManifest, PluginError 等
├── textInputTypes.ts   # 文本输入类型（VimMode, PromptInputMode 等）
└── generated/          # 自动生成的类型（ protobuf / gRPC 等）
    ├── events_mono/    # 单体事件类型
    └── google/         # Google API 相关类型
```

这种极简设计反映了一个重要的架构决策：**类型定义不应集中在 types/ 目录，而应就近放置**。大量类型直接定义在各自的模块中，例如 `src/Tool.ts` 中的 `Tool` 类型、`src/state/AppStateStore.ts` 中的 `AppState` 类型。`types/` 目录更多承载跨模块共享、具有全局意义的类型定义。

### 2.4.2 Command 类型：联合类型的力量

`src/types/command.ts` 展示了复杂的类型设计：

```typescript
// src/types/command.ts:16-57
export type PromptCommand = {
  type: 'prompt'
  progressMessage: string
  contentLength: number
  argNames?: string[]
  allowedTools?: string[]
  model?: string
  source: SettingSource | 'builtin' | 'mcp' | 'plugin' | 'bundled'
  // ...
}

export type LocalCommand = {
  type: 'local'
  supportsNonInteractive: boolean
  load: () => Promise<LocalCommandModule>
}

export type LocalJSXCommand = {
  type: 'local-jsx'
  load: () => Promise<LocalJSXCommandModule>
}

export type Command = CommandBase &
  (PromptCommand | LocalCommand | LocalJSXCommand)
```

这种设计使用**判别联合类型**（Discriminated Unions）：
- `type` 字段作为判别符，区分不同类型的命令
- 每种命令类型有自己特有的字段
- 类型安全且易于扩展

### 2.4.3 Plugin 类型：详细错误处理

`src/types/plugin.ts` 展示了复杂的错误类型设计：

```typescript
// src/types/plugin.ts:101-284
export type PluginError =
  | {
      type: 'path-not-found'
      source: string
      plugin?: string
      path: string
      component: PluginComponent
    }
  | {
      type: 'git-auth-failed'
      source: string
      plugin?: string
      gitUrl: string
      authType: 'ssh' | 'https'
    }
  | {
      type: 'plugin-not-found'
      source: string
      pluginId: string
      marketplace: string
    }
  // ... 20+ 种错误类型
```

这种精细化的错误类型设计使得：
1. 错误处理类型安全
2. UI 可以针对每种错误类型显示特定的指导
3. 错误信息结构化，便于日志分析

### 2.4.4 类型导出与重导出

类型定义的导出遵循清晰的规则：

```typescript
// 导出类型定义
export type { PluginAuthor, PluginManifest }

// 导出辅助函数
export function getPluginErrorMessage(error: PluginError): string {
  // ...
}
```

### 2.4.5 类型系统的设计原则

1. **集中定义**：相关类型聚合在同一文件中
2. **判别联合**：使用判别字段区分类型变体
3. **精确错误**：详细的错误类型支持更好的错误处理
4. **可重用**：通用类型定义为独立模块

> **设计思想 4：类型即文档**
>
> - 使用 TypeScript 的类型系统作为API文档
> - 判别联合类型确保类型安全和代码可读性
> - 精细的错误类型支持更好的用户体验
> - 类型定义与实现分离，便于维护

---

## 2.5 常量管理：constants/ 的分层策略

`src/constants/` 目录包含了系统的各种常量定义，展现了良好的常量组织策略。

### 2.5.1 constants/ 目录结构

`src/constants/` 包含 21 个文件，按功能域组织：

```
src/constants/
├── apiLimits.ts             # API 调用频率和 token 限制
├── betas.ts                 # Beta 特性标识与模型能力矩阵
├── common.ts                # 通用常量（默认值、共享字符串）
├── cyberRiskInstruction.ts  # 安全风险指令
├── errorIds.ts              # 错误 ID 枚举
├── figures.ts               # 终端 UI 图标（Unicode 符号）
├── files.ts                 # 文件路径与扩展名常量
├── github-app.ts            # GitHub App 配置（client ID 等）
├── keys.ts                  # settings.json 配置键名
├── messages.ts              # 用户可见消息模板
├── oauth.ts                 # OAuth 端点与配置
├── outputStyles.ts          # 输出样式定义（217行）
├── plugin.ts                # 插件系统常量
├── product.ts               # 产品信息（名称、版本号）
├── prompts.ts               # 系统提示词构建（915行，最大文件）
├── spinnerVerbs.ts          # 加载动画动词列表
├── system.ts                # 系统级常量（环境变量名等）
├── systemPromptSections.ts  # 系统提示词分段定义
├── toolLimits.ts            # 工具执行限制参数
├── tools.ts                 # 工具配置（名称、别名映射）
├── turnCompletionVerbs.ts   # 轮次完成状态动词
└── xml.ts                   # XML 标签常量
```

### 2.5.2 prompts.ts：系统提示词的核心

`prompts.ts` 是最大的常量文件（915行），包含了完整的系统提示词构建逻辑：

```typescript
// src/constants/prompts.ts:48
import { feature } from 'bun:bundle'

// 死代码消除：条件导入
const getCachedMCConfigForFRC = feature('CACHED_MICROCOMPACT')
  ? require('../services/compact/cachedMCConfig.js').getCachedMCConfig
  : null

// 系统提示词分段
export function getSystemPrompt(
  tools: Tools,
  model: string,
  additionalWorkingDirectories?: string[],
  mcpClients?: MCPServerConnection[],
): Promise<string[]> {
  // 构建完整的系统提示词
}
```

关键设计：
1. **分段构建**：系统提示词由多个独立段落组成
2. **条件包含**：根据特性标志动态包含内容
3. **缓存边界**：`SYSTEM_PROMPT_DYNAMIC_BOUNDARY` 标记缓存边界

```typescript
// src/constants/prompts.ts:114-115
export const SYSTEM_PROMPT_DYNAMIC_BOUNDARY =
  '__SYSTEM_PROMPT_DYNAMIC_BOUNDARY__'
```

### 2.5.3 outputStyles.ts：输出样式配置

`outputStyles.ts` 定义了不同的输出风格：

```typescript
// src/constants/outputStyles.ts:11-42
export type OutputStyleConfig = {
  name: string
  description: string
  prompt: string
  source: SettingSource | 'built-in' | 'plugin'
  keepCodingInstructions?: boolean
  forceForPlugin?: boolean
}

export const OUTPUT_STYLE_CONFIG: OutputStyles = {
  [DEFAULT_OUTPUT_STYLE_NAME]: null,
  Explanatory: {
    name: 'Explanatory',
    source: 'built-in',
    description: 'Claude explains its implementation choices',
    keepCodingInstructions: true,
    prompt: `...`,
  },
  Learning: {
    name: 'Learning',
    source: 'built-in',
    description: 'Claude pauses and asks you to write code',
    keepCodingInstructions: true,
    prompt: `...`,
  },
}
```

### 2.5.4 常量管理的分层策略

1. **按功能分组**：相关常量放在同一文件
2. **避免魔法值**：所有常量都有命名和类型
3. **支持扩展**：输出样式等支持插件扩展
4. **构建时优化**：配合 `feature()` 实现死代码消除

### 2.5.5 提示词构建的边界标记

```typescript
// src/constants/prompts.ts:572-573
// === BOUNDARY MARKER - DO NOT MOVE OR REMOVE ===
...(shouldUseGlobalCacheScope() ? [SYSTEM_PROMPT_DYNAMIC_BOUNDARY] : [])
// --- Dynamic content (registry-managed) ---
```

这个边界标记实现了：
- 缓存作用域的分离
- 静态内容与动态内容的明确划分
- 提示词缓存优化

> **设计思想 5：常量即配置**
>
> - 将所有可配置的值提取为常量
> - 按功能域分组常量，便于查找
> - 支持运行时扩展（如输出样式）
> - 使用类型确保常量的正确使用

---

## 2.6 小结与思考

### 2.6.1 核心设计原则总结

1. **快速路径优先**：将常见操作放入快速路径，零模块加载
2. **编译时优化**：使用 `feature()` 实现死代码消除
3. **模块化架构**：按功能域划分模块，职责清晰
4. **类型安全**：充分利用 TypeScript 的类型系统
5. **常量化配置**：避免魔法值，提高可维护性

### 2.6.2 构建系统的亮点

1. **多态构建**：同一代码库可以构建出不同功能的版本
2. **增量加载**：动态导入减少启动时间
3. **类型安全**：即使是动态导入也保持类型安全
4. **实验隔离**：实验性功能完全隔离，不会影响稳定版

### 2.6.3 项目结构的启示

1. **工具/命令分离**：AI 调用的工具与用户调用的命令分开
2. **组件化 UI**：即使终端应用也使用 React 组件
3. **服务层抽象**：复杂逻辑封装在服务层
4. **工具函数库**：331+ 工具函数支撑整个系统

### 2.6.4 类型系统的价值

1. **判别联合**：使用判别字段区分类型变体
2. **精细错误**：详细的错误类型支持更好的 UX
3. **类型导出**：类型定义与实现分离
4. **自动补全**：完整的类型支持 IDE 功能

### 2.6.5 给开发者的建议

1. **使用构建时特性门控**：将实验性功能用 `feature()` 包裹
2. **快速路径优化**：识别常见操作并优化其路径
3. **模块化设计**：按功能域而非技术层次划分模块
4. **类型即文档**：用 TypeScript 类型作为 API 文档
5. **常量化配置**：提取所有配置为常量

### 2.6.6 工程哲学

Claude Code 的工程基础体现了一个核心哲学：**简单胜于复杂**。

- 通过快速路径减少不必要的复杂性
- 通过死代码消除保持代码库的精简
- 通过模块化保持代码的可维护性
- 通过类型系统减少运行时错误
- 通过常量化减少配置错误

---

> "最好的工程是看不见的工程。当一切正常运行时，用户不会注意到构建系统、类型检查、模块加载——他们只会感受到快速、稳定、可靠的产品。"

本章介绍了 Claude Code 的工程基础：构建系统、项目结构、类型系统和常量管理。在接下来的章节中，我们将深入探讨 Query Engine、Tool 抽象、命令系统等核心组件的实现。

**下一章预告**：第3章将深入探讨启动与初始化流程，了解 Claude Code 是如何从一行命令启动到完整的交互界面的。

---

**源码引用索引**：

- `src/entrypoints/cli.tsx:1-303` - CLI 入口与快速路径（12+种快速路径分派）
- `src/entrypoints/init.ts:1-341` - 系统级初始化（15+项初始化任务）
- `src/constants/prompts.ts:48-915` - 系统提示词构建
- `src/types/command.ts:16-217` - 命令类型定义
- `src/types/plugin.ts:101-364` - 插件类型与错误处理
- `src/types/logs.ts:1-331` - 日志与持久化类型
- `src/constants/outputStyles.ts:11-217` - 输出样式配置
- `src/utils/betas.ts:1-435` - Beta 特性与模型支持
- `src/Tool.ts:1-100` - 工具接口定义
