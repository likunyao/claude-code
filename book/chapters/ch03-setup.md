# 第3章 启动与初始化 —— setup.ts的引导艺术

> 一个程序的启动就像一场交响乐的序曲，所有乐器必须和谐地同时发声，而不能是一团混乱的噪音。Claude Code 的启动流程展示了如何精心编排数百个初始化步骤，创造出优雅而高效的开始体验。

## 3.1 从main.tsx到setup.ts：启动流程分析

当我们执行 `claude` 命令时，首先进入的是 `main.tsx` 中的 `main()` 函数。这个函数不仅是程序的入口点，更是整个启动流程的指挥中心。

### 3.1.1 早期启动优化

在模块加载阶段，Claude Code 采用了一系列巧妙的优化策略来加速启动：

```typescript
// 这些副作用必须在所有其他导入之前运行：
// 1. profileCheckpoint 标记入口点，在繁重的模块评估开始之前
// 2. startMdmRawRead 启动 MDM 子进程（plutil/reg 查询），使其与剩余的
//    约 135ms 的导入并行运行
// 3. startKeychainPrefetch 启动两个 macOS 钥匙串读取（OAuth + 传统 API 密钥）
//    并行运行 —— 否则 isRemoteManagedSettingsEligible() 通过
//    applySafeConfigEnvironmentVariables() 中的同步 spawn 按顺序读取它们
//    （每次 macOS 启动约 65ms）
```

这种"预取"策略是 Claude Code 启动性能的核心。通过在早期阶段启动那些原本会阻塞的 I/O 操作，程序可以在加载其他模块的同时等待这些操作完成，从而实现真正的并行化。

### 3.1.2 main函数的职责

`main()` 函数在 `src/main.tsx` 的第 585 行定义，它承担着多重职责：

1. **安全设置**：设置 Windows PATH 劫持防护
2. **信号处理**：注册 SIGINT 和 exit 事件处理器
3. **URL 处理**：处理 `cc://` 和 `cc+unix://` 直接连接 URL
4. **深度链接**：处理操作系统协议处理器调用的 URI
5. **子命令路由**：SSH、assistant 等特殊模式的参数预处理
6. **Commander.js 初始化**：构建 CLI 参数解析器

函数签名如下：

```typescript
export async function main() {
  profileCheckpoint('main_function_start');

  // 安全防护：防止 Windows 从当前目录执行命令
  process.env.NoDefaultCurrentDirectoryInExePath = '1';

  // 初始化警告处理器
  initializeWarningHandler();
  // ...
}
```

### 3.1.3 preAction钩子：初始化的真正入口

Commander.js 的 `preAction` 钩子才是真正执行初始化的地方。这个钩子只在执行命令时运行，而不是在显示帮助时运行，避免了不必要的初始化开销。

```typescript
program.hook('preAction', async thisCommand => {
  profileCheckpoint('preAction_start');

  // 等待模块评估时启动的异步子进程完成
  await Promise.all([
    ensureMdmSettingsLoaded(),
    ensureKeychainPrefetchCompleted()
  ]);

  // 执行核心初始化
  await init();

  // 设置进程标题
  if (!isEnvTruthy(process.env.CLAUDE_CODE_DISABLE_TERMINAL_TITLE)) {
    process.title = 'claude';
  }

  // 初始化日志接收器
  initSinks();

  // 运行迁移
  runMigrations();

  // 加载远程管理设置（非阻塞）
  void loadRemoteManagedSettings();
  void loadPolicyLimits();
});
```

这种设计确保了只有在真正需要执行命令时才会进行完整的初始化，而在显示帮助信息时则保持轻量级。

### 3.1.4 从main到setup的调用链

当默认命令（即交互式 REPL）被执行时，会调用 `setup()` 函数。调用链如下：

```
main()
  └─ program.action(async (prompt, options) => { ... })
       └─ setup(cwd, permissionMode, ...)
            ├─ 环境探测与验证
            ├─ 配置加载
            ├─ 迁移执行
            └─ 入口分流
```

## 3.2 配置加载的层次：全局→项目→会话

Claude Code 采用三层配置系统，每一层都有特定的用途和优先级。

### 3.2.1 配置层次概述

1. **全局配置（Global Config）**：存储在 `~/.claude/config.json`，包含用户级别的设置
2. **项目配置（Project Config）**：存储在 `<project>/.claude/config.json`，包含项目特定的设置
3. **会话配置（Session Config）**：运行时配置，通过 CLI 参数或环境变量覆盖

### 3.2.2 配置加载机制

配置加载通过 `src/utils/config.ts` 中的函数实现：

```typescript
export function getGlobalConfig(): GlobalConfig {
  const configPath = getGlobalClaudeFile('config.json');
  return readConfigSafely(configPath, DEFAULT_GLOBAL_CONFIG);
}

export function getCurrentProjectConfig(): ProjectConfig {
  const configPath = join(getCwd(), '.claude', 'config.json');
  return readConfigSafely(configPath, DEFAULT_PROJECT_CONFIG);
}
```

配置读取采用"安全读取"策略，包含以下保护措施：

1. **BOM 剥离**：处理 UTF-8 BOM 标记
2. **错误恢复**：配置解析失败时回退到默认值
3. **文件锁**：防止并发写入冲突
4. **缓存机制**：避免重复文件读取

### 3.2.3 配置合并策略

不同来源的配置按照特定优先级合并：

```
CLI 参数 > 环境变量 > 项目配置 > 全局配置 > 默认值
```

这种设计使得用户可以在不同层次灵活地覆盖配置，同时保持合理的默认行为。

### 3.2.5 新近演化：本地设置之外，还有“远程托管设置”和“策略限制”

Claude Code 的初始化并不止于读取 `~/.claude/config.json` 和项目配置。`main.tsx` 在 `preAction` 里除了本地初始化，还会异步触发两条企业能力链路：

```typescript
void loadRemoteManagedSettings()
void loadPolicyLimits()
```

这两套机制都不是“启动失败即退出”的硬依赖，而是典型的 **fail-open 基础设施**：

1. **Remote Managed Settings**
   面向 Team / Enterprise 场景，把组织级设置从服务端拉到本地。
   它带有 checksum 校验、磁盘缓存、后台轮询和安全检查，目标是让企业管理员能够远程下发设置，而不是要求每台机器手动维护本地文件。

2. **Policy Limits**
   面向组织级能力开关，例如是否允许 remote sessions、某些 CLI 功能是否禁用。
   它和远程设置一样采用缓存、ETag/校验和、后台刷新，但语义上更偏“功能限制”，而不是“配置注入”。

3. **初始化顺序上的约束**
   两者都设计了 `initialize...LoadingPromise()` / `waitFor...ToLoad()` 这样的同步点。
   初始化因此呈现出“早触发、按需等待”的并发编排形态。

今天的 Claude Code 初始化可以概括为四层叠加：

```
默认值
  ← 本地全局 / 项目 / 会话设置
  ← 远程托管设置（组织下发）
  ← 策略限制（组织裁剪）
```

这里体现出的工程姿态是：**企业能力被设计成可缺席、可缓存、可后台刷新，而不是把主启动流程绑死在网络请求上**。

### 3.2.4 设置系统（Settings.json）

与配置系统并列的是设置系统，位于 `~/.claude/settings.json`。这是用户可编辑的设置文件，包含更细粒度的控制选项。

设置系统的类型定义位于 `src/utils/settings/types.ts`，支持：

- **环境变量注入**：通过 `env` 字段设置进程环境变量
- **模型配置**：默认模型、努力程度等
- **权限设置**：工具权限模式
- **插件配置**：插件目录和管理设置

```typescript
export type Settings = {
  env?: Record<string, string>;
  model?: ModelSetting;
  effort?: 'low' | 'medium' | 'high' | 'max';
  permissionMode?: PermissionMode;
  // ... 更多设置
};
```

## 3.3 环境探测：终端能力检测与适配

Claude Code 需要在多种终端环境中运行，因此必须具备强大的环境探测和适配能力。

### 3.3.1 Node.js版本检查

setup 函数的第一项任务就是验证运行环境：

```typescript
// 检查 Node.js 版本是否低于 18
const nodeVersion = process.version.match(/^v(\d+)\./)?.[1];
if (!nodeVersion || parseInt(nodeVersion) < 18) {
  console.error(
    chalk.bold.red(
      'Error: Claude Code requires Node.js version 18 or higher.'
    )
  );
  process.exit(1);
}
```

这种早期验证可以防止在 unsupported 环境中运行时出现难以诊断的错误。

### 3.3.2 终端备份与恢复

对于 macOS 用户，Claude Code 会修改终端设置以支持某些高级功能。为确保用户的原始设置不会丢失，系统会自动备份和恢复：

```typescript
// Terminal.app 备份检查
const restoredTerminalBackup = await checkAndRestoreTerminalBackup();
if (restoredTerminalBackup.status === 'restored') {
  console.log(
    chalk.yellow(
      'Detected an interrupted Terminal.app setup. ' +
      'Your original settings have been restored.'
    )
  );
}
```

类似地，对于 iTerm2：

```typescript
// iTerm2 备份检查（仅在 swarms 启用时）
if (isAgentSwarmsEnabled()) {
  const restoredIterm2Backup = await checkAndRestoreITerm2Backup();
  // ... 恢复逻辑
}
```

这种设计体现了"防御性编程"的思想：即使进程意外中断，用户的原始设置也能得到保护。

### 3.3.3 工作树（Worktree）环境探测

当使用 `--worktree` 参数时，系统需要探测当前是否在 Git 仓库中：

```typescript
if (worktreeEnabled) {
  const hasHook = hasWorktreeCreateHook();
  const inGit = await getIsGit();

  if (!hasHook && !inGit) {
    process.stderr.write(
      chalk.red(
        `Error: Can only use --worktree in a git repository, ` +
        `but ${chalk.bold(cwd)} is not a git repository.`
      )
    );
    process.exit(1);
  }

  // 解析到主仓库根目录
  const mainRepoRoot = findCanonicalGitRoot(getCwd());
  // ...
}
```

这种探测不仅检查 Git 是否存在，还考虑了自定义钩子的可能性，为非 Git VCS 系统留出了扩展空间。

### 3.3.4 沙箱环境检测

对于 `--dangerously-skip-permissions` 模式，系统会验证是否在安全的沙箱环境中：

```typescript
if (permissionMode === 'bypassPermissions' || allowDangerouslySkipPermissions) {
  // 检查是否以 root/sudo 运行
  if (process.platform !== 'win32' &&
      process.getuid?.() === 0 &&
      process.env.IS_SANDBOX !== '1') {
    console.error(
      '--dangerously-skip-permissions cannot be used with root/sudo privileges'
    );
    process.exit(1);
  }

  // 检查 Docker/沙箱环境
  const [isDocker, hasInternet] = await Promise.all([
    envDynamic.getIsDocker(),
    env.hasInternetAccess()
  ]);

  const isSandboxed = isDocker || isBubblewrap || isSandbox;
  if (!isSandboxed || hasInternet) {
    console.error(
      '--dangerously-skip-permissions can only be used in Docker/sandbox containers with no internet access'
    );
    process.exit(1);
  }
}
```

这种多层验证确保了危险模式只在真正安全的环境中启用。

## 3.4 迁移系统：migrations/的版本演进策略

软件的长期维护需要平滑的升级路径。Claude Code 的迁移系统提供了一种优雅的方式来处理配置和设置的演进。

### 3.4.1 迁移文件的组织

所有迁移文件位于 `src/migrations/` 目录下，每个文件负责一个特定的迁移任务：

```
src/migrations/
├── migrateAutoUpdatesToSettings.ts
├── migrateBypassPermissionsAcceptedToSettings.ts
├── migrateEnableAllProjectMcpServersToSettings.ts
├── migrateFennecToOpus.ts
├── migrateLegacyOpusToCurrent.ts
├── migrateOpusToOpus1m.ts
├── migrateReplBridgeEnabledToRemoteControlAtStartup.ts
├── migrateSonnet1mToSonnet45.ts
├── migrateSonnet45ToSonnet46.ts
├── resetAutoModeOptInForDefaultOffer.ts
└── resetProToOpusDefault.ts
```

### 3.4.2 迁移模式分析

#### 模式1：配置迁移到设置

将全局配置迁移到用户设置文件中：

```typescript
// migrateBypassPermissionsAcceptedToSettings.ts
export function migrateBypassPermissionsAcceptedToSettings(): void {
  const globalConfig = getGlobalConfig();

  if (!globalConfig.bypassPermissionsModeAccepted) {
    return; // 无需迁移
  }

  try {
    if (!hasSkipDangerousModePermissionPrompt()) {
      updateSettingsForSource('userSettings', {
        skipDangerousModePermissionPrompt: true
      });
    }

    // 从全局配置中移除旧字段
    saveGlobalConfig(current => {
      const { bypassPermissionsModeAccepted: _, ...rest } = current;
      return rest;
    });
  } catch (error) {
    logError(new Error(`Failed to migrate: ${error}`));
  }
}
```

#### 模式2：模型别名迁移

处理模型名称变更：

```typescript
// migrateFennecToOpus.ts
export function migrateFennecToOpus(): void {
  if (process.env.USER_TYPE !== 'ant') {
    return; // 仅对内部用户生效
  }

  const settings = getSettingsForSource('userSettings');
  const model = settings?.model;

  if (typeof model === 'string') {
    if (model.startsWith('fennec-latest[1m]')) {
      updateSettingsForSource('userSettings', { model: 'opus[1m]' });
    } else if (model.startsWith('fennec-latest')) {
      updateSettingsForSource('userSettings', { model: 'opus' });
    } else if (model.startsWith('fennec-fast-latest')) {
      updateSettingsForSource('userSettings', {
        model: 'opus[1m]',
        fastMode: true
      });
    }
  }
}
```

#### 模式3：一次性设置迁移

```typescript
// migrateAutoUpdatesToSettings.ts
export function migrateAutoUpdatesToSettings(): void {
  const globalConfig = getGlobalConfig();

  // 仅迁移用户显式禁用的情况（非原生安装保护）
  if (globalConfig.autoUpdates !== false ||
      globalConfig.autoUpdatesProtectedForNative === true) {
    return;
  }

  try {
    const userSettings = getSettingsForSource('userSettings') || {};

    // 设置环境变量以保留用户意图
    updateSettingsForSource('userSettings', {
      ...userSettings,
      env: {
        ...userSettings.env,
        DISABLE_AUTOUPDATER: '1'
      }
    });

    // 立即生效
    process.env.DISABLE_AUTOUPDATER = '1';

    // 迁移完成后从全局配置中移除
    saveGlobalConfig(current => {
      const { autoUpdates: _, autoUpdatesProtectedForNative: __, ...rest } = current;
      return rest;
    });
  } catch (error) {
    logError(new Error(`Failed to migrate auto-updates: ${error}`));
  }
}
```

### 3.4.3 迁移执行策略

迁移在 `main.tsx` 的 preAction 钩子中执行：

```typescript
program.hook('preAction', async thisCommand => {
  // ... 其他初始化

  runMigrations();  // 执行所有迁移

  // ... 继续执行
});
```

`runMigrations()` 函数会按需执行所有迁移函数。每个迁移函数都是**幂等的**（idempotent），即多次执行不会产生副作用，这确保了迁移的安全性和可重复性。

### 3.4.4 迁移设计原则

从这些迁移代码中，我们可以总结出几个关键设计原则：

1. **幂等性**：迁移可以安全地多次执行
2. **防御性编程**：使用 try-catch 包裹迁移逻辑
3. **渐进式迁移**：每次迁移只处理一个特定的变更
4. **审计日志**：记录迁移事件以便追踪
5. **回滚友好**：保留旧值直到确认迁移成功

## 3.5 入口分流：CLI、REPL、Bridge、Assistant模式的路由

Claude Code 支持多种运行模式，每种模式都有不同的启动路径和使用场景。

### 3.5.1 模式概览

| 模式 | 入口标志 | 用途 | 特点 |
|------|----------|------|------|
| CLI 默认 | 无参数或直接输入提示词 | 交互式对话 | 完整 TUI，支持所有功能 |
| CLI Print | `-p` 或 `--print` | 脚本集成 | 无头模式，输出到 stdout |
| MCP 服务器 | `mcp serve` | MCP 协议服务 | 标准工具协议 |
| Bridge 模式 | `--sdk-url` 等 | 远程会话管理 | 多会话协调 |
| Assistant 模式 | `--assistant` | Agent SDK 守护进程 | 精简 UI，专注代理 |
| SSH 远程 | `ssh <host>` | 远程开发 | SSH 隧道连接 |

### 3.5.2 CLI 默认模式（REPL）

这是最常用的模式，启动交互式 REPL：

```typescript
program.action(async (prompt, options) => {
  // ... 配置处理

  // 调用 setup 进行完整初始化
  await setup(
    cwd,
    permissionMode,
    allowDangerouslySkipPermissions,
    worktreeEnabled,
    // ... 其他参数
  );

  // 渲染 REPL
  await renderAndRun(
    <Root {...props} />,
    getRenderContext(options)
  );
});
```

setup 函数为 REPL 模式执行的初始化包括：

1. 启动 UDS 消息服务器（用于 Agent 间通信）
2. 捕获 Teammate 模式快照
3. 恢复终端备份
4. 处理 worktree 创建
5. 预取命令和插件
6. 注册各种钩子和后台任务

### 3.5.3 Print 模式（无头模式）

当使用 `-p` 或 `--print` 参数时，程序进入无头模式：

```typescript
if (printMode) {
  // 跳过信任对话框
  // 直接执行查询
  const result = await executeQuery(prompt);
  console.log(formatOutput(result));
  process.exit(0);
}
```

Print 模式的特点：

- **无交互**：不显示 TUI，所有输出到 stdout
- **快速退出**：执行完毕后立即退出
- **管道友好**：支持 JSON/stream-json 格式
- **简化初始化**：跳过 REPL 特定的预取

### 3.5.4 MCP 服务器模式

MCP 模式将 Claude Code 暴露为标准 MCP 服务器：

```typescript
mcp.command('serve')
  .description('Start the Claude Code MCP server')
  .option('-d, --debug', 'Enable debug mode')
  .action(async ({ debug, verbose }) => {
    await startMCPServer(cwd, debug, verbose);
  });
```

MCP 服务器实现位于 `src/entrypoints/mcp.ts`：

```typescript
export async function startMCPServer(
  cwd: string,
  debug: boolean,
  verbose: boolean
): Promise<void> {
  const server = new Server(
    {
      name: 'claude/tengu',
      version: MACRO.VERSION,
    },
    {
      capabilities: {
        tools: {},
      },
    }
  );

  server.setRequestHandler(
    ListToolsRequestSchema,
    async () => {
      const tools = getTools(toolPermissionContext);
      return {
        tools: await Promise.all(
          tools.map(async tool => ({
            ...tool,
            description: await tool.prompt(...),
            inputSchema: zodToJsonSchema(tool.inputSchema)
          }))
        ),
      };
    }
  );

  server.setRequestHandler(
    CallToolRequestSchema,
    async (request) => {
      // 工具调用逻辑
    }
  );
}
```

### 3.5.5 Bridge 模式

Bridge 模式用于远程会话管理，实现多会话协调：

```typescript
export async function runBridgeLoop(
  config: BridgeConfig,
  environmentId: string,
  environmentSecret: string,
  api: BridgeApiClient,
  spawner: SessionSpawner,
  logger: BridgeLogger,
  signal: AbortSignal,
  backoffConfig: BackoffConfig = DEFAULT_BACKOFF,
  initialSessionId?: string
): Promise<void> {
  // 持续轮询远程服务器
  while (!signal.aborted) {
    try {
      const work = await api.fetchWork(environmentId, {
        sessionId: currentSessionId,
        timeout: SESSION_TIMEOUT_MS,
      });

      if (work) {
        await spawner.spawn(work);
      }
    } catch (error) {
      // 退避重试
      await backoff(connBackoffMs);
    }
  }
}
```

Bridge 模式的关键特性：

- **会话生成**：动态创建和管理子会话
- **退避重试**：指数退避的网络错误处理
- **睡眠检测**：识别系统休眠并重置连接
- **令牌刷新**：自动刷新认证令牌

### 3.5.6 Assistant 模式

Assistant 模式专为 Agent SDK 守护进程设计：

```typescript
if (feature('KAIROS') && assistantModule?.isAssistantMode()) {
  if (!checkHasTrustDialogAccepted()) {
    console.warn('Assistant mode disabled: directory is not trusted.');
  } else {
    kairosEnabled = await kairosGate.isKairosEnabled();
    if (kairosEnabled) {
      opts.brief = true;
      setKairosActive(true);
      // 预先创建进程内团队
      assistantTeamContext = await assistantModule.initializeAssistantTeam();
    }
  }
}
```

Assistant 模式的特点：

- **精简 UI**：启用 brief 模式，减少输出冗余
- **进程内团队**：预先创建 Agent 团队
- **信任检查**：要求目录已被明确信任
- **门控启用**：通过 GrowthBook 功能门控制

## 3.6 小结与思考

Claude Code 的启动流程展示了几个重要的软件工程原则：

### 3.6.1 性能优先的设计

1. **并行预取**：在模块加载阶段启动 I/O 密集型操作
2. **延迟加载**：只在需要时加载重量级模块
3. **缓存策略**：避免重复的文件系统操作
4. **死码消除**：通过功能门移除未使用的代码路径

### 3.6.2 防御性编程

1. **版本检查**：早期验证运行环境
2. **备份恢复**：保护用户原始设置
3. **幂等迁移**：安全的配置升级
4. **沙箱验证**：危险模式的多层检查

### 3.6.3 可扩展性设计

1. **模式分离**：不同运行模式有独立的启动路径
2. **钩子系统**：通过钩子扩展初始化行为
3. **配置层次**：灵活的配置覆盖机制
4. **功能门控**：渐进式功能发布

### 3.6.4 用户体验考虑

1. **快速反馈**：最小化首次渲染时间
2. **优雅降级**：非关键功能失败不影响核心功能
3. **清晰错误**：有帮助的错误消息
4. **状态恢复**：从中断中自动恢复

启动流程是用户对软件的第一印象，也是系统稳定性的基石。Claude Code 通过精心的设计和实现，创造了一个既快速又可靠的启动体验。这不仅展示了扎实的技术功底，更体现了对用户体验的深度关注。

正如《C++编程思想》中所强调的："好的软件设计应该让复杂的系统看起来简单。"Claude Code 的启动流程正是这一理念的典范 —— 它将数百个复杂的初始化步骤隐藏在简洁的接口之后，为用户提供流畅无缝的开始体验。
