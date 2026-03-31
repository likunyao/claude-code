# 第20章 Bridge系统 —— 进程间的桥梁

现代AI应用面临一个核心挑战：如何在不同环境、不同进程之间建立可靠的通信机制。Claude Code通过Bridge系统构建了一个灵活的跨进程通信架构，支持Remote Control（远程控制）、桌面应用集成和多会话管理等高级功能。

## 20.1 桥接通信架构

Bridge系统的核心职责是在两个进程之间建立双向通信通道：

1. **CLI进程**：运行Claude Code REPL，执行实际工作
2. **控制进程**：Web界面、桌面应用或其他控制端

### 架构组件

```
┌─────────────────┐     HTTP Poll      ┌──────────────┐
│  CLI Process    │ ◄─────────────────► │  CCR Server  │
│  (replBridge)   │                     │  (Backend)   │
└────────┬────────┘                     └──────────────┘
         │
         │ WebSocket/SSE
         │
    ┌────▼─────────────────────────────────────┐
    │         Transport Layer                  │
    │  ┌────────────────┐  ┌────────────────┐ │
    │  │ HybridTransport│  │  SSETransport  │ │
    │  │    (v1)        │  │     (v2)       │ │
    │  └────────────────┘  └────────────────┘ │
    └──────────────────────────────────────────┘
```

### 环境注册与认证

通信的第一步是环境注册：

```typescript
// src/bridge/bridgeApi.ts
async registerBridgeEnvironment(
  config: BridgeConfig
): Promise<{ environment_id: string; environment_secret: string }> {
  const response = await axios.post(
    `${baseUrl}/v1/environments/bridge`,
    {
      machine_name: config.machineName,
      directory: config.dir,
      branch: config.branch,
      git_repo_url: config.gitRepoUrl,
      max_sessions: config.maxSessions,
      metadata: { worker_type: config.workerType }
    },
    { headers: getHeaders(token) }
  )
  return response.data
}
```

注册成功后，服务器返回：
- `environment_id`：环境标识符
- `environment_secret`：用于后续认证的密钥

### 轮询与任务分发

Bridge使用长轮询（long polling）机制获取工作：

```typescript
async pollForWork(
  environmentId: string,
  environmentSecret: string,
  signal?: AbortSignal,
  reclaimOlderThanMs?: number
): Promise<WorkResponse | null> {
  const response = await axios.get(
    `${baseUrl}/v1/environments/${environmentId}/work/poll`,
    {
      headers: getHeaders(environmentSecret),
      params: reclaimOlderThanMs !== undefined
        ? { reclaim_older_than_ms: reclaimOlderThanMs }
        : undefined,
      timeout: 10_000,
      signal
    }
  )
  return response.data
}
```

轮询参数说明：
- `reclaimOlderThanMs`：重新领取指定时间前未完成的任务
- 10秒超时：平衡响应速度和服务器负载
- AbortSignal支持：允许优雅取消轮询

## 20.2 多通道传输层

Bridge系统支持两种传输协议，通过统一的`ReplBridgeTransport`接口抽象：

### v1: HybridTransport

v1传输层混合使用WebSocket和HTTP：

```typescript
// src/bridge/replBridgeTransport.ts
export function createV1ReplTransport(
  hybrid: HybridTransport
): ReplBridgeTransport {
  return {
    write: msg => hybrid.write(msg),
    writeBatch: msgs => hybrid.writeBatch(msgs),
    close: () => hybrid.close(),
    isConnectedStatus: () => hybrid.isConnectedStatus(),
    getStateLabel: () => hybrid.getStateLabel(),
    setOnData: cb => hybrid.setOnData(cb),
    setOnClose: cb => hybrid.setOnClose(cb),
    setOnConnect: cb => hybrid.setOnConnect(cb),
    connect: () => void hybrid.connect(),
    getLastSequenceNum: () => 0, // v1不支持序列号
    droppedBatchCount: hybrid.droppedBatchCount,
    // v2专用功能在v1中为no-op
    reportState: () => {},
    reportMetadata: () => {},
    reportDelivery: () => {},
    flush: () => Promise.resolve()
  }
}
```

**特点**：
- WebSocket读取：服务器主动推送消息
- HTTP写入：通过POST请求发送消息
- Session-Ingress端点：使用传统的会话入口服务

### v2: SSETransport + CCRClient

v2传输层使用Server-Sent Events读取，CCRClient写入：

```typescript
export async function createV2ReplTransport(opts: {
  sessionUrl: string
  ingressToken: string
  sessionId: string
  initialSequenceNum?: number
  epoch?: number
  heartbeatIntervalMs?: number
  outboundOnly?: boolean
  getAuthToken?: () => string | undefined
}): Promise<ReplBridgeTransport> {
  // SSE读取流
  const sseUrl = new URL(sessionUrl)
  sseUrl.pathname = sseUrl.pathname.replace(/\/$/, '') + '/worker/events/stream'

  const sse = new SSETransport(
    sseUrl,
    {},
    sessionId,
    undefined,
    initialSequenceNum,
    getAuthHeaders
  )

  // CCR写入客户端
  const epoch = opts.epoch ?? (await registerWorker(sessionUrl, ingressToken))
  const ccr = new CCRClient(sse, new URL(sessionUrl), {
    getAuthHeaders,
    heartbeatIntervalMs: opts.heartbeatIntervalMs,
    onEpochMismatch: () => {
      // Epoch不匹配时关闭连接
      ccr.close()
      sse.close()
      throw new Error('epoch superseded')
    }
  })

  return {
    write(msg) { return ccr.writeEvent(msg) },
    async writeBatch(msgs) {
      for (const m of msgs) {
        if (closed) break
        await ccr.writeEvent(m)
      }
    },
    close() {
      closed = true
      ccr.close()
      sse.close()
    },
    getLastSequenceNum() { return sse.getLastSequenceNum() },
    reportState(state) { ccr.reportState(state) },
    reportMetadata(metadata) { ccr.reportMetadata(metadata) },
    reportDelivery(eventId, status) { ccr.reportDelivery(eventId, status) },
    flush() { return ccr.flush() },
    // ...其他方法
  }
}
```

**v2优势**：
- SSE序列号：支持从断点续传，避免重放历史消息
- Epoch机制：检测并处理worker重新注册
- 心跳保活：定期发送心跳保持连接活跃
- 状态上报：支持向服务器报告worker状态

### 20.2.1 v2 不是“换了协议”，而是换了会话语义

`src/bridge/replBridgeTransport.ts` 表明，v2 的设计重点并不只是把 WebSocket 改成 SSE，而是把 Bridge 连接从“尽力而为的消息通道”升级成“可恢复、可确认、可报告状态的会话通道”。

具体体现为：

1. **sequence number**
   新 transport 会继承旧 transport 的 `initialSequenceNum`，从而让服务端从正确位置继续推流，而不是从头重放。

2. **epoch**
   worker 重新注册后，旧 epoch 会被判定 superseded，transport 主动关闭并交还 poll loop 恢复。

3. **delivery acknowledgment**
   v2 不只接收事件，还会上报 `received / processing / processed` 等投递状态。

4. **worker state / metadata**
   transport 可以主动告诉后端当前是否 `requires_action`，以及附加外部 metadata。

这背后的架构变化是：Bridge 已经不再只是“把 REPL 输出搬到远端”，而是开始承担 **远程会话编排协议** 的角色。

### 传输层选择

系统通过特性门控选择传输层：

```typescript
// src/bridge/bridgeEnabled.ts
export function isEnvLessBridgeEnabled(): boolean {
  return feature('BRIDGE_MODE')
    ? getFeatureValue_CACHED_MAY_BE_STALE('tengu_bridge_repl_v2', false)
    : false
}
```

## 20.3 消息协议

Bridge系统定义了明确的消息协议，处理三种主要消息类型：

### 消息类型判定

```typescript
// src/bridge/bridgeMessaging.ts
export function isSDKMessage(value: unknown): value is SDKMessage {
  return (
    value !== null &&
    typeof value === 'object' &&
    'type' in value &&
    typeof value.type === 'string'
  )
}

export function isSDKControlResponse(
  value: unknown
): value is SDKControlResponse {
  return (
    value !== null &&
    typeof value === 'object' &&
    'type' in value &&
    value.type === 'control_response' &&
    'response' in value
  )
}

export function isSDKControlRequest(
  value: unknown
): value is SDKControlRequest {
  return (
    value !== null &&
    typeof value === 'object' &&
    'type' in value &&
    value.type === 'control_request' &&
    'request_id' in value &&
    'request' in value
  )
}
```

### 入站消息路由

```typescript
export function handleIngressMessage(
  data: string,

### 20.3.1 Bridge 与 CLI transport 的边界

Bridge 目录本身并不包办所有传输实现，真正的读写能力分散在 `src/cli/transports/` 与 `src/bridge/` 两侧：

- `SSETransport` 负责读侧事件流
- `CCRClient` 负责 v2 写侧与心跳
- `HybridTransport` 兼容旧路径
- `replBridgeTransport.ts` 负责把这些实现适配成统一接口

这是一种很稳健的分层：

1. `cli/transports/` 关注“怎么传”
2. `bridge/` 关注“何时切换、如何恢复、状态如何汇报”

因此，Bridge 的真正价值不是发明新协议，而是把多种传输协议编排成面向 REPL 的会话抽象。
  recentPostedUUIDs: BoundedUUIDSet,
  recentInboundUUIDs: BoundedUUIDSet,
  onInboundMessage?: (msg: SDKMessage) => void,
  onPermissionResponse?: (response: SDKControlResponse) => void,
  onControlRequest?: (request: SDKControlRequest) => void
): void {
  const parsed: unknown = jsonParse(data)

  // 控制响应（权限决策）
  if (isSDKControlResponse(parsed)) {
    onPermissionResponse?.(parsed)
    return
  }

  // 控制请求（服务器发起）
  if (isSDKControlRequest(parsed)) {
    onControlRequest?.(parsed)
    return
  }

  if (!isSDKMessage(parsed)) return

  // 去重：忽略自己发送的消息回显
  const uuid = parsed.uuid
  if (uuid && recentPostedUUIDs.has(uuid)) {
    return
  }

  // 去重：忽略已处理的入站消息
  if (uuid && recentInboundUUIDs.has(uuid)) {
    return
  }

  // 处理用户消息
  if (parsed.type === 'user') {
    if (uuid) recentInboundUUIDs.add(uuid)
    onInboundMessage?.(parsed)
  }
}
```

### 控制请求处理

服务器可以发送多种控制请求，客户端必须响应：

```typescript
export function handleServerControlRequest(
  request: SDKControlRequest,
  handlers: ServerControlRequestHandlers
): void {
  const { transport, sessionId, outboundOnly, onInterrupt, onSetModel, ... } = handlers

  let response: SDKControlResponse

  switch (request.request.subtype) {
    case 'initialize':
      // 响应初始化，声明能力
      response = {
        type: 'control_response',
        response: {
          subtype: 'success',
          request_id: request.request_id,
          response: {
            commands: [],
            output_style: 'normal',
            models: [],
            account: {},
            pid: process.pid
          }
        }
      }
      break

    case 'set_model':
      onSetModel?.(request.request.model)
      response = { type: 'control_response', response: { subtype: 'success', request_id: request.request_id } }
      break

    case 'interrupt':
      onInterrupt?.()
      response = { type: 'control_response', response: { subtype: 'success', request_id: request.request_id } }
      break

    // ... 其他case
  }

  const event = { ...response, session_id: sessionId }
  void transport.write(event)
}
```

### 去重机制

Bridge使用有界UUID集合实现去重：

```typescript
export class BoundedUUIDSet {
  private readonly capacity: number
  private readonly ring: (string | undefined)[]
  private readonly set = new Set<string>()
  private writeIdx = 0

  constructor(capacity: number) {
    this.capacity = capacity
    this.ring = new Array(capacity)
  }

  add(uuid: string): void {
    if (this.set.has(uuid)) return
    // 驱逐最旧的条目
    const evicted = this.ring[this.writeIdx]
    if (evicted !== undefined) {
      this.set.delete(evicted)
    }
    this.ring[this.writeIdx] = uuid
    this.set.add(uuid)
    this.writeIdx = (this.writeIdx + 1) % this.capacity
  }

  has(uuid: string): boolean {
    return this.set.has(uuid)
  }
}
```

这种环形缓冲区设计确保：
- 内存使用恒定O(capacity)
- 自动驱逐最旧条目
- 按时间顺序添加消息

## 20.4 远程控制

Remote Control是Bridge系统的核心应用场景，允许用户通过Web界面控制本地CLI会话。

### 会话生命周期

```typescript
// src/bridge/types.ts
export type SessionHandle = {
  sessionId: string
  done: Promise<SessionDoneStatus>
  kill(): void
  forceKill(): void
  activities: SessionActivity[]       // 活动环形缓冲区
  currentActivity: SessionActivity | null
  accessToken: string
  lastStderr: string[]
  writeStdin(data: string): void
  updateAccessToken(token: string): void
}
```

### 工作秘密解码

Bridge使用WorkSecret机制传递会话认证信息：

```typescript
// src/bridge/workSecret.ts
export async function decodeWorkSecret(
  secretB64: string
): Promise<WorkSecret> {
  const buffer = Buffer.from(secretB64, 'base64')
  const json = buffer.toString('utf-8')
  return JSON.parse(json)
}

// WorkSecret结构
export type WorkSecret = {
  version: number
  session_ingress_token: string
  api_base_url: string
  sources: Array<{
    type: string
    git_info?: { type: string; repo: string; ref?: string }
  }>
  auth: Array<{ type: string; token: string }>
  claude_code_args?: Record<string, string>
  mcp_config?: unknown
  environment_variables?: Record<string, string>
  use_code_sessions?: boolean
}
```

### 心跳机制

v2传输层实现心跳保活：

```typescript
class CCRClient {
  private heartbeatTimer?: ReturnType<typeof setInterval>

  async initialize(epoch: number): Promise<void> {
    this.workerEpoch = epoch
    this.startHeartbeat()
  }

  private startHeartbeat(): void {
    const interval = this.options.heartbeatIntervalMs ?? 20_000
    const jitter = interval * (this.options.heartbeatJitterFraction ?? 0)

    this.heartbeatTimer = setInterval(() => {
      void this.sendHeartbeat(jitter)
    }, interval)
  }

  private async sendHeartbeat(jitterMs: number): Promise<void> {
    const delay = Math.random() * jitterMs
    await sleep(delay)

    try {
      await this.request('POST', `/worker/${this.sessionId}/heartbeat`, {
        epoch: this.workerEpoch
      })
    } catch (err) {
      // 心跳失败不中断连接，由服务器检测超时
    }
  }
}
```

### 会话恢复

Bridge支持会话恢复机制：

```typescript
// src/bridge/bridgeApi.ts
async reconnectSession(
  environmentId: string,
  sessionId: string
): Promise<void> {
  const response = await withOAuthRetry(
    (token: string) => axios.post(
      `${baseUrl}/v1/environments/${environmentId}/bridge/reconnect`,
      { session_id: sessionId },
      { headers: getHeaders(token) }
    ),
    'ReconnectSession'
  )
  handleErrorStatus(response.status, response.data, 'ReconnectSession')
}
```

恢复过程：
1. 检测环境丢失
2. 尝试重新注册环境
3. 调用`reconnectSession`重新排队会话
4. 使用新的work secret建立传输层

## 20.5 桌面集成

Bridge系统为桌面应用提供深度集成支持。

### Worker类型标识

系统通过`worker_type`区分不同来源：

```typescript
export type BridgeWorkerType = 'claude_code' | 'claude_code_assistant'

const bridgeConfig: BridgeConfig = {
  // ...
  workerType: 'claude_code', // 或 'claude_code_assistant'
  metadata: { worker_type: 'claude_code' }
}
```

Web界面可以基于此过滤显示：
- `claude_code`：CLI启动的会话
- `claude_code_assistant`：Assistant模式的会话
- `cowork`：桌面应用启动的会话

### 多会话管理

Bridge支持管理多个并发会话：

```typescript
export type BridgeLogger = {
  addSession(sessionId: string, url: string): void
  removeSession(sessionId: string): void
  setSessionTitle(sessionId: string, title: string): void
  updateSessionActivity(sessionId: string, activity: SessionActivity): void
  updateSessionCount(active: number, max: number, mode: SpawnMode): void
  refreshDisplay(): void
}
```

多会话显示示例：
```
Connected (2/4 sessions)
  • session-abc123: Fix authentication bug
    Editing src/auth.ts
  • session-def456: Add tests
    Running test suite
```

### Spawn模式

Bridge支持三种会话生成模式：

```typescript
export type SpawnMode = 'single-session' | 'worktree' | 'same-dir'

const bridgeConfig: BridgeConfig = {
  spawnMode: 'worktree',
  // ...
}
```

模式说明：
- `single-session`：单个会话，会话结束后销毁环境
- `worktree`：持久服务器，每个会话使用独立的git worktree
- `same-dir`：持久服务器，所有会话共享同一目录（可能互相干扰）

## 20.6 错误处理与重连

Bridge系统实现了完善的错误处理和恢复机制。

### 错误分类

```typescript
export class BridgeFatalError extends Error {
  readonly status: number
  readonly errorType: string | undefined

  constructor(message: string, status: number, errorType?: string) {
    super(message)
    this.name = 'BridgeFatalError'
    this.status = status
    this.errorType = errorType
  }
}
```

### 错误状态处理

```typescript
function handleErrorStatus(
  status: number,
  data: unknown,
  context: string
): void {
  const detail = extractErrorDetail(data)
  const errorType = extractErrorTypeFromData(data)

  switch (status) {
    case 401:
      throw new BridgeFatalError(
        `${context}: Authentication failed. ${BRIDGE_LOGIN_INSTRUCTION}`,
        401,
        errorType
      )
    case 403:
      throw new BridgeFatalError(
        isExpiredErrorType(errorType)
          ? 'Remote Control session has expired.'
          : `${context}: Access denied.`,
        403,
        errorType
      )
    case 404:
      throw new BridgeFatalError(
        'Remote Control may not be available for your organization.',
        404,
        errorType
      )
    case 410:
      throw new BridgeFatalError(
        'Remote Control session has expired.',
        410,
        errorType ?? 'environment_expired'
      )
    case 429:
      throw new Error(`${context}: Rate limited. Polling too frequently.`)
    default:
      throw new Error(`${context}: Failed with status ${status}`)
  }
}
```

### 轮询错误恢复

```typescript
const POLL_ERROR_INITIAL_DELAY_MS = 2_000
const POLL_ERROR_MAX_DELAY_MS = 60_000
const POLL_ERROR_GIVE_UP_MS = 15 * 60 * 1000

async function pollWithRecovery(): Promise<WorkResponse | null> {
  let delay = POLL_ERROR_INITIAL_DELAY_MS
  const startTime = Date.now()

  while (Date.now() - startTime < POLL_ERROR_GIVE_UP_MS) {
    try {
      const work = await api.pollForWork(environmentId, secret)
      return work
    } catch (err) {
      if (Date.now() - startTime >= POLL_ERROR_GIVE_UP_MS) {
        throw err
      }
      await sleep(delay)
      delay = Math.min(delay * 2, POLL_ERROR_MAX_DELAY_MS)
    }
  }
  throw new Error('Poll error recovery timeout')
}
```

### 状态转换

Bridge通过状态机管理连接生命周期：

```typescript
export type BridgeState = 'ready' | 'connected' | 'reconnecting' | 'failed'

onStateChange?.('connected')
onStateChange?.('reconnecting', 'Connection lost, retrying...')
onStateChange?.('failed', errorMessage)
```

UI根据状态显示不同反馈：
- `ready`：显示等待连接
- `connected`：显示活动状态
- `reconnecting`：显示重连倒计时
- `failed`：显示错误信息

## 20.7 小结

Bridge系统是Claude Code跨进程通信的核心，展示了如何构建可靠的分布式通信架构：

**架构设计**：
- 环境注册与认证机制确保安全通信
- 长轮询实现高效的任务分发
- 双向通信支持命令下发和事件上报

**传输层抽象**：
- 统一的`ReplBridgeTransport`接口屏蔽协议差异
- v1 HybridTransport兼容传统WebSocket
- v2 SSETransport + CCRClient提供更现代的解决方案

**消息协议**：
- 明确的消息类型判定和路由
- 控制请求/响应机制支持服务器主动控制
- UUID去重避免消息重复处理

**错误处理**：
- 分类错误处理和友好的用户提示
- 指数退避的重试机制
- 状态机驱动的UI反馈

**扩展性**：
- Worker类型标识支持多端集成
- 多会话管理支持复杂协作场景
- Spawn模式适应不同的使用场景

Bridge系统的设计体现了分布式系统的几个关键原则：

1. **优雅降级**：连接失败时自动重连，不会完全阻塞
2. **透明性**：详细的状态日志和错误信息便于调试
3. **可扩展性**：清晰的抽象边界支持协议升级
4. **容错性**：多层次的去重和恢复机制

这个系统让Claude Code能够在本地CLI、Web界面和桌面应用之间无缝切换，为用户提供一致的体验。同时，它也为未来的多设备协作、远程协助等场景奠定了基础。
