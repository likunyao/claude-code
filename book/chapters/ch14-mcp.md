# 第14章 MCP协议 —— Model Context Protocol的设计与实现

> "协议的价值在于解耦。" MCP协议将AI能力与具体实现分离，使Claude Code能够动态扩展，而无需修改核心代码。

## 14.1 MCP架构概览：客户端、服务端、传输层

Model Context Protocol（MCP）是一个开放协议，用于连接AI助手与外部系统。在Claude Code中，MCP实现了一个可扩展的工具、资源和命令发现与调用机制。

### 三层架构

MCP的实现可以分为三个层次：

```
┌─────────────────────────────────────────────────────────────┐
│                     应用层 (Application Layer)               │
│  - 工具调用 (Tool Call)                                      │
│  - 资源访问 (Resource Access)                                │
│  - 提示模板 (Prompt Template)                                │
└─────────────────────────────────────────────────────────────┘
                              ▲
                              │ JSON-RPC 2.0
                              │
┌─────────────────────────────────────────────────────────────┐
│                      协议层 (Protocol Layer)                  │
│  - 消息序列化/反序列化                                        │
│  - 请求/响应路由                                              │
│  - 错误处理                                                  │
└─────────────────────────────────────────────────────────────┘
                              ▲
                              │
┌─────────────────────────────────────────────────────────────┐
│                      传输层 (Transport Layer)                 │
│  - stdio: 进程通信                                           │
│  - SSE: Server-Sent Events                                  │
│  - HTTP: Streamable HTTP                                    │
│  - WebSocket: 实时双向通信                                   │
└─────────────────────────────────────────────────────────────┘
```

### 类型系统

MCP的核心类型定义在`src/services/mcp/types.ts`中，包括：

```typescript
// 传输类型
type Transport = 'stdio' | 'sse' | 'http' | 'ws' | 'sdk'

// 服务器配置
type McpServerConfig =
  | McpStdioServerConfig    // 本地进程
  | McpSSEServerConfig      // SSE连接
  | McpHTTPServerConfig     // HTTP连接
  | McpWebSocketServerConfig // WebSocket
  | McpSdkServerConfig      // SDK内联

// 连接状态
type MCPServerConnection =
  | ConnectedMCPServer      // 已连接
  | FailedMCPServer         // 连接失败
  | NeedsAuthMCPServer      // 需要认证
  | PendingMCPServer        // 连接中
  | DisabledMCPServer       // 已禁用
```

这种类型设计体现了"状态机"思维：服务器连接有明确的状态转换，每个状态都有对应的处理逻辑。

## 14.2 服务端管理：mcp/的连接生命周期

MCP服务端管理的核心是`useManageMCPConnections` Hook（`src/services/mcp/useManageMCPConnections.ts`），它负责：
1. 初始化服务器连接
2. 管理连接状态变化
3. 处理自动重连
4. 响应配置变化

### 连接生命周期

```typescript
// 连接状态机
type ConnectionState =
  | 'pending'     // 初始状态，等待连接
  | 'connected'   // 连接成功
  | 'failed'      // 连接失败
  | 'needs-auth'  // 需要认证
  | 'disabled'    // 用户禁用
```

连接过程采用了"分阶段加载"策略：

```typescript
// Phase 1: 快速加载本地配置
const { servers: claudeCodeConfigs } = await getClaudeCodeMcpConfigs()

// Phase 2: 延迟加载远程配置
const claudeaiConfigs = await fetchClaudeAIMcpConfigsIfEligible()
```

这种设计确保了启动速度：本地配置立即加载，远程配置异步获取。

### 批量状态更新

为了优化性能，MCP状态更新采用了批量合并策略：

```typescript
const MCP_BATCH_FLUSH_MS = 16  // 16ms窗口

const flushPendingUpdates = () => {
  // 将所有pending更新合并为一次setState
  setAppState(prevState => {
    let mcp = prevState.mcp
    for (const update of pendingUpdates) {
      // 增量更新mcp状态
    }
    return { ...prevState, mcp }
  })
}
```

这种"时间窗口批量"模式避免了频繁的状态更新导致的不必要渲染。

### 自动重连机制

对于远程传输（SSE、HTTP、WebSocket），实现了指数退避的自动重连：

```typescript
const MAX_RECONNECT_ATTEMPTS = 5
const INITIAL_BACKOFF_MS = 1000
const MAX_BACKOFF_MS = 30000

const reconnectWithBackoff = async () => {
  for (let attempt = 1; attempt <= MAX_RECONNECT_ATTEMPTS; attempt++) {
    try {
      const result = await reconnectMcpServerImpl(name, config)
      if (result.client.type === 'connected') return
    } catch (error) {
      const backoffMs = Math.min(
        INITIAL_BACKOFF_MS * Math.pow(2, attempt - 1),
        MAX_BACKOFF_MS
      )
      await sleep(backoffMs)
    }
  }
}
```

这种设计平衡了"连接可靠性"与"资源消耗"。

## 14.3 MCPTool：动态工具的注册与执行

MCPTool（`src/tools/MCPTool/MCPTool.ts`）是所有MCP工具的模板，它采用"继承覆盖"模式：

```typescript
export const MCPTool = buildTool({
  isMcp: true,
  name: 'mcp',           // 被覆盖
  description() {},      // 被覆盖
  async call() {
    return { data: '' }  // 被覆盖
  }
})
```

真正的工具在`client.ts`的`fetchToolsForClient`中动态创建：

```typescript
const tools = await client.request(
  { method: 'tools/list' },
  ListToolsResultSchema
)

return tools.map(tool => ({
  ...MCPTool,
  name: buildMcpToolName(client.name, tool.name),
  description: () => tool.description,
  async call(args, context) {
    const result = await client.callTool({
      name: tool.name,
      arguments: args,
      _meta: meta
    })
    return { data: await processMCPResult(result) }
  }
}))
```

### 工具命名规范

MCP工具采用命名空间前缀避免冲突：

```typescript
// src/services/mcp/mcpStringUtils.ts
export function buildMcpToolName(serverName: string, toolName: string): string {
  return `mcp__${normalizeNameForMCP(serverName)}__${toolName}`
}
```

例如，Slack的`search_messages`工具命名为`mcp__slack__search_messages`。

### 工具结果处理

MCP工具返回的结果需要经过多层处理：

```typescript
// 1. 类型检查
const transformed = await transformMCPResult(result, tool, name)

// 2. 大小截断
if (await mcpContentNeedsTruncation(content)) {
  return await truncateMcpContentIfNeeded(content)
}

// 3. 持久化（对于大结果）
const persistResult = await persistToolResult(contentStr, persistId)
```

这种"管道处理"模式确保了结果的安全性和可管理性。

## 14.4 资源系统：ListMcpResourcesTool、ReadMcpResourceTool

MCP资源是服务器暴露的"数据端点"，类似于REST API的端点。

### 资源列表

`ListMcpResourcesTool`列出所有可用资源：

```typescript
export const ListMcpResourcesTool = buildTool({
  name: 'ListMcpResourcesTool',
  async call(input, { options: { mcpClients } }) {
    const results = await Promise.all(
      mcpClients.map(async client => {
        if (client.type !== 'connected') return []
        return await fetchResourcesForClient(client)
      })
    )
    return { data: results.flat() }
  }
})
```

资源列表使用LRU缓存：

```typescript
export const fetchResourcesForClient = memoizeWithLRU(
  async (client: MCPServerConnection) => {
    const result = await client.client.request(
      { method: 'resources/list' },
      ListResourcesResultSchema
    )
    return result.resources.map(r => ({
      ...r,
      server: client.name  // 添加服务器标识
    }))
  },
  (client) => client.name,
  20  // 最大缓存20个服务器
)
```

### 资源读取

`ReadMcpResourceTool`读取特定资源：

```typescript
const result = await client.request({
  method: 'resources/read',
  params: { uri }
}, ReadResourceResultSchema)

// 二进制内容特殊处理
if ('blob' in content) {
  const persisted = await persistBinaryContent(
    Buffer.from(content.blob, 'base64'),
    content.mimeType,
    persistId
  )
  return { blobSavedTo: persisted.filepath }
}
```

这种设计将"二进制内容"与"文本内容"统一处理，避免将base64直接注入上下文。

### 资源变化通知

MCP支持服务器主动通知资源变化：

```typescript
if (client.capabilities?.resources?.listChanged) {
  client.client.setNotificationHandler(
    ResourceListChangedNotificationSchema,
    async () => {
      fetchResourcesForClient.cache.delete(client.name)
      const newResources = await fetchResourcesForClient(client)
      updateServer({ ...client, resources: newResources })
    }
  )
}
```

这实现了"最终一致性"模型：服务器推送变化通知，客户端被动更新。

## 14.5 认证与安全：McpAuthTool的权限控制

MCP支持OAuth 2.0认证，实现位于`src/services/mcp/auth.ts`。

### 认证流程

1. **发现元数据**：从服务器获取OAuth配置
2. **动态客户端注册**：或使用预配置的客户端ID
3. **授权码流程**：通过本地回调服务器接收授权码
4. **令牌交换**：用授权码换取访问令牌

```typescript
export async function performMCPOAuthFlow(
  serverName: string,
  serverConfig: McpSSEServerConfig | McpHTTPServerConfig,
  onAuthorizationUrl: (url: string) => void
) {
  // 启动本地回调服务器
  const server = createServer((req, res) => {
    if (req.url?.startsWith('/callback')) {
      const code = new URL(req.url, 'http://localhost')
        .searchParams.get('code')
      resolver(code)  // 解析授权码Promise
    }
  })

  server.listen(port, '127.0.0.1')

  // 触发授权
  await sdkAuth(provider, { serverUrl })

  // 完成授权
  await sdkAuth(provider, { authorizationCode })
}
```

### 认证提供者

`ClaudeAuthProvider`实现了MCP SDK的`OAuthClientProvider`接口：

```typescript
export class ClaudeAuthProvider implements OAuthClientProvider {
  async tokens(): Promise<OAuthTokens | undefined> {
    const tokenData = storage.read()?.mcpOAuth[serverKey]

    // 主动刷新
    if (expiresIn <= 300 && tokenData.refreshToken) {
      return await this.refreshAuthorization(tokenData.refreshToken)
    }

    return {
      access_token: tokenData.accessToken,
      refresh_token: tokenData.refreshToken,
      expires_in: expiresIn
    }
  }
}
```

### 认证伪工具

当服务器需要认证时，`McpAuthTool`作为"伪工具"被插入工具列表：

```typescript
export function createMcpAuthTool(serverName: string, config): Tool {
  return {
    name: buildMcpToolName(serverName, 'authenticate'),
    description: `The "${serverName}" MCP server requires authentication...`,
    async call(_input, context) {
      const authUrl = await performMCPOAuthFlow(...)
      return {
        data: {
          status: 'auth_url',
          authUrl,
          message: `Ask the user to open this URL...`
        }
      }
    }
  }
}
```

这种"工具即状态"的设计，将认证过程转化为模型可理解的交互。

### 跨应用访问（XAA）

XAA（Cross-App Access）是Claude Code的扩展认证机制，允许多个MCP服务器共享一个IdP登录：

```typescript
async function performMCPXaaAuth(...) {
  // 1. 获取IdP的id_token（可缓存）
  const idToken = await acquireIdpIdToken(...)

  // 2. 执行RFC 8693令牌交换
  const tokens = await performCrossAppAccess({
    idpIdToken: idToken,
    idpTokenEndpoint: oidc.token_endpoint
  })

  // 3. 保存为普通OAuth令牌
  saveTokens(tokens)
}
```

XAA减少了重复登录：一个IdP会话，多个MCP服务器受益。

## 14.6 小结与思考

MCP协议的实现体现了几个重要设计原则：

1. **协议优先于实现**：通过标准化接口，实现可插拔的扩展
2. **渐进式增强**：从基本功能到高级特性（OAuth、XAA、Channel）
3. **状态管理**：清晰的状态机（pending→connected→failed）
4. **性能优化**：批量更新、LRU缓存、并发控制

### 设计模式总结

| 模式 | 应用场景 | 文件位置 |
|------|----------|----------|
| 模板方法 | MCPTool基类定义 | MCPTool.ts |
| 工厂方法 | createMcpAuthTool | McpAuthTool.ts |
| 策略模式 | 多种传输层 | InProcessTransport, SdkControlTransport |
| 观察者模式 | 资源变化通知 | useManageMCPConnections.ts |
| 适配器模式 | MCP工具→内部工具 | client.ts |

### 扩展性思考

MCP的设计使Claude Code能够：
- 无需修改代码即可支持新的MCP服务器
- 动态发现和调用工具
- 统一处理认证和权限
- 实现跨平台的能力扩展

这正是"开放协议"的价值：通过标准化接口，实现生态系统的繁荣。

---

"协议的价值在于连接，而非控制。" MCP协议的成功，在于它连接了AI与无限的外部世界，同时保持了核心系统的简洁与稳定。
