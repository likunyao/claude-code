# 第21章 API层 —— 与Claude的通信

> "The API layer is where the rubber meets the road — it's the boundary between your application and the model." —— API设计原则

Claude Code与Claude模型的交互通过一个精心设计的API层实现。这个层次不仅要处理认证、请求构建、响应解析等基本功能，还要应对网络波动、速率限制、流式传输等复杂场景。本章将深入剖析Claude Code的API层架构，揭示其设计背后的工程思考。

API 层的边界比“模型请求客户端”更宽。除了 `claude.ts / client.ts / withRetry.ts` 这些核心文件之外，`services/api/` 目录还承担了：

- `bootstrap.ts`：启动阶段的引导数据获取
- `filesApi.ts`：会话文件下载与文件规格解析
- `usage.ts` / `emptyUsage.ts`：用量与降级占位
- `promptCacheBreakDetection.ts`：提示缓存破坏检测
- `sessionIngress.ts`：会话入口与远程事件通路
- `overageCreditGrant.ts` / `ultrareviewQuota.ts`：产品配额与额度能力

Claude Code 的 API 层已经不仅是“模型 RPC”，而是把模型调用、产品配额、会话基础设施和启动引导统一纳入了同一服务边界。

## 21.1 API客户端分层设计

Claude Code的API层采用了清晰的分层架构，每一层都有明确的职责：

```
┌─────────────────────────────────────────────────────────────┐
│                     Query Engine Layer                      │
│  (query.ts - 消息管理、工具编排、流式响应聚合)                  │
└─────────────────────────────────────────────────────────────┘
                              │
┌─────────────────────────────────────────────────────────────┐
│                      API Client Layer                        │
│  (client.ts - 客户端创建、认证管理、多提供商支持)                │
└─────────────────────────────────────────────────────────────┘
                              │
┌─────────────────────────────────────────────────────────────┐
│                      Request Builder                         │
│  (claude.ts - 参数构建、消息转换、Beta配置)                     │
└─────────────────────────────────────────────────────────────┘
                              │
┌─────────────────────────────────────────────────────────────┐
│                      Retry Layer                             │
│  (withRetry.ts - 重试逻辑、错误分类、退避策略)                  │
└─────────────────────────────────────────────────────────────┘
                              │
┌─────────────────────────────────────────────────────────────┐
│                      Transport Layer                         │
│  (Anthropic SDK - HTTP通信、流式处理、连接管理)                 │
└─────────────────────────────────────────────────────────────┘
```

### 21.1.1 多提供商支持

Claude Code不仅支持Anthropic的第一方API，还集成了AWS Bedrock、Azure Foundry、Google Vertex AI等多个云提供商。这种设计使企业用户能够在自己的基础设施上运行Claude模型。

`getAnthropicClient`函数是这一架构的核心入口：

```typescript
export async function getAnthropicClient({
  apiKey,
  maxRetries,
  model,
  fetchOverride,
  source,
}: {
  apiKey?: string
  maxRetries: number
  model?: string
  fetchOverride?: ClientOptions['fetch']
  source?: string
}): Promise<Anthropic> {
  // 1. 构建通用配置
  const defaultHeaders: { [key: string]: string } = {
    'x-app': 'cli',
    'User-Agent': getUserAgent(),
    'X-Claude-Code-Session-Id': getSessionId(),
    ...customHeaders,
  }

  // 2. OAuth认证检查
  await checkAndRefreshOAuthTokenIfNeeded()

  // 3. 根据环境变量选择提供商
  if (isEnvTruthy(process.env.CLAUDE_CODE_USE_BEDROCK)) {
    return new AnthropicBedrock(bedrockArgs) as unknown as Anthropic
  }
  if (isEnvTruthy(process.env.CLAUDE_CODE_USE_FOUNDRY)) {
    return new AnthropicFoundry(foundryArgs) as unknown as Anthropic
  }
  if (isEnvTruthy(process.env.CLAUDE_CODE_USE_VERTEX)) {
    return new AnthropicVertex(vertexArgs) as unknown as Anthropic
  }

  // 4. 默认使用第一方API
  return new Anthropic(clientConfig)
}
```

这种设计的精妙之处在于：
- **统一接口**：无论使用哪个提供商，上层代码都通过`Anthropic`类型进行交互
- **环境驱动**：通过环境变量控制提供商选择，无需修改代码
- **认证隔离**：每个提供商的认证逻辑独立处理，OAuth、API Key、AWS凭证等互不干扰

### 21.1.2 请求上下文传递

API层的每个函数都需要大量的上下文信息。Claude Code使用`Options`对象来集中管理这些参数：

```typescript
export type Options = {
  getToolPermissionContext: () => Promise<ToolPermissionContext>
  model: string
  toolChoice?: BetaToolChoiceTool | BetaToolChoiceAuto | undefined
  isNonInteractiveSession: boolean
  extraToolSchemas?: BetaToolUnion[]
  maxOutputTokensOverride?: number
  fallbackModel?: string
  querySource: QuerySource
  agents: AgentDefinition[]
  allowedAgentTypes?: string[]
  hasAppendSystemPrompt: boolean
  fetchOverride?: ClientOptions['fetch']
  enablePromptCaching?: boolean
  skipCacheWrite?: boolean
  temperatureOverride?: number
  effortValue?: EffortValue
  mcpTools: Tools
  hasPendingMcpServers?: boolean
  queryTracking?: QueryChainTracking
  agentId?: AgentId
  outputFormat?: BetaJSONOutputFormat
  fastMode?: boolean
  advisorModel?: string
  addNotification?: (notif: Notification) => void
  taskBudget?: { total: number; remaining?: number }
}
```

这种设计模式的优势：
1. **参数集中管理**：避免函数签名过长
2. **类型安全**：TypeScript确保所有必需参数都被提供
3. **易于扩展**：添加新参数不需要修改函数签名
4. **上下文传递**：可以在调用链中传递和修改选项

## 21.2 OAuth认证流程

Claude Code支持两种认证方式：API Key和OAuth 2.0。OAuth认证为Claude.ai订阅用户提供了更流畅的体验，无需手动复制粘贴API Key。

### 21.2.1 PKCE流程

OAuth 2.0 PKCE（Proof Key for Code Exchange）流程是Claude Code认证机制的核心。相比传统的OAuth流程，PKCE不需要客户端密钥，更适合CLI应用。

```typescript
export class OAuthService {
  private codeVerifier: string
  private authCodeListener: AuthCodeListener | null = null
  private port: number | null = null

  constructor() {
    // 1. 生成code_verifier（随机字符串）
    this.codeVerifier = crypto.generateCodeVerifier()
  }

  async startOAuthFlow(
    authURLHandler: (url: string, automaticUrl?: string) => Promise<void>,
    options?: OAuthFlowOptions,
  ): Promise<OAuthTokens> {
    // 2. 启动本地HTTP服务器监听回调
    this.authCodeListener = new AuthCodeListener()
    this.port = await this.authCodeListener.start()

    // 3. 生成PKCE参数
    const codeChallenge = crypto.generateCodeChallenge(this.codeVerifier)
    const state = crypto.generateState()

    // 4. 构建授权URL
    const authUrl = new URL(getOauthConfig().CONSOLE_AUTHORIZE_URL)
    authUrl.searchParams.append('code', 'true')
    authUrl.searchParams.append('client_id', getOauthConfig().CLIENT_ID)
    authUrl.searchParams.append('response_type', 'code')
    authUrl.searchParams.append('redirect_uri', `http://localhost:${this.port}/callback`)
    authUrl.searchParams.append('scope', ALL_OAUTH_SCOPES.join(' '))
    authUrl.searchParams.append('code_challenge', codeChallenge)
    authUrl.searchParams.append('code_challenge_method', 'S256')
    authUrl.searchParams.append('state', state)

    // 5. 等待授权码
    const authorizationCode = await this.waitForAuthorizationCode(state, ...)

    // 6. 交换access token
    const tokenResponse = await client.exchangeCodeForTokens(
      authorizationCode,
      state,
      this.codeVerifier,
      this.port!,
    )

    return this.formatTokens(tokenResponse, ...)
  }
}
```

PKCE的安全性体现在：
- **code_verifier**：客户端生成的随机字符串，从未通过网络传输
- **code_challenge**：`SHA256(code_verifier)`的base64url编码，发送给授权服务器
- **验证过程**：授权服务器在交换token时验证`code_verifier`的正确性

即使攻击者截获了授权码，没有原始的`code_verifier`也无法获取token。

### 21.2.2 Token刷新机制

OAuth access token通常有较短的有效期（如1小时），refresh token则可以长期使用。Claude Code实现了自动刷新机制：

```typescript
export async function refreshOAuthToken(
  refreshToken: string,
  { scopes: requestedScopes }: { scopes?: string[] } = {},
): Promise<OAuthTokens> {
  const requestBody = {
    grant_type: 'refresh_token',
    refresh_token: refreshToken,
    client_id: getOauthConfig().CLIENT_ID,
    scope: requestedScopes?.length ? requestedScopes : CLAUDE_AI_OAUTH_SCOPES,
  }

  const response = await axios.post(getOauthConfig().TOKEN_URL, requestBody, {
    headers: { 'Content-Type': 'application/json' },
    timeout: 15000,
  })

  const data = response.data as OAuthTokenExchangeResponse
  const {
    access_token: accessToken,
    refresh_token: newRefreshToken = refreshToken,
    expires_in: expiresIn,
  } = data

  const expiresAt = Date.now() + expiresIn * 1000

  // 智能优化：如果已有profile信息，跳过额外的API调用
  const existing = getClaudeAIOAuthTokens()
  const haveProfileAlready =
    config.oauthAccount?.billingType !== undefined &&
    existing?.subscriptionType != null

  const profileInfo = haveProfileAlready ? null : await fetchProfileInfo(accessToken)

  return {
    accessToken,
    refreshToken: newRefreshToken,
    expiresAt,
    scopes: parseScopes(data.scope),
    subscriptionType: profileInfo?.subscriptionType ?? existing?.subscriptionType ?? null,
    // ...
  }
}
```

刷新机制的优化亮点：
1. **条件请求**：只在必要时获取用户profile，减少API调用
2. **Rolling Refresh**：新的refresh token立即替换旧的，支持会话延续
3. **缓存复用**：已有的subscription信息不会重复获取

### 21.2.3 认证状态管理

Claude Code在每次API请求前都会检查token的有效性：

```typescript
// 在getAnthropicClient中调用
await checkAndRefreshOAuthTokenIfNeeded()

// 检查逻辑
export function isOAuthTokenExpired(expiresAt: number | null): boolean {
  if (expiresAt === null) return false
  const bufferTime = 5 * 60 * 1000 // 5分钟缓冲
  const now = Date.now()
  const expiresWithBuffer = now + bufferTime
  return expiresWithBuffer >= expiresAt
}
```

这种"提前刷新"策略确保在实际请求发出前token始终有效，避免了因token过期导致的请求失败。

## 21.3 速率限制与优雅降级

与Claude API交互时，速率限制是必须面对的现实。Claude Code实现了多层次的应对策略，从智能重试到模型回退，确保用户体验的流畅性。

### 21.3.1 错误分类与重试策略

`withRetry`函数是速率限制处理的核心。它首先对错误进行分类，然后采取相应的策略：

```typescript
export async function* withRetry<T>(
  getClient: () => Promise<Anthropic>,
  operation: (client: Anthropic, attempt: number, context: RetryContext) => Promise<T>,
  options: RetryOptions,
): AsyncGenerator<SystemAPIErrorMessage, T> {
  const maxRetries = getMaxRetries(options)
  const retryContext: RetryContext = { /* ... */ }
  let client: Anthropic | null = null
  let consecutive529Errors = 0

  for (let attempt = 1; attempt <= maxRetries + 1; attempt++) {
    try {
      if (
        client === null ||
        (lastError instanceof APIError && lastError.status === 401) ||
        isOAuthTokenRevokedError(lastError) ||
        isBedrockAuthError(lastError) ||
        isVertexAuthError(lastError)
      ) {
        // 认证错误时强制刷新token
        if ((lastError instanceof APIError && lastError.status === 401) || ...) {
          const failedAccessToken = getClaudeAIOAuthTokens()?.accessToken
          if (failedAccessToken) {
            await handleOAuth401Error(failedAccessToken)
          }
        }
        client = await getClient()
      }

      return await operation(client, attempt, retryContext)
    } catch (error) {
      lastError = error

      // 529过载错误处理
      if (is529Error(error)) {
        consecutive529Errors++
        if (consecutive529Errors >= MAX_529_RETRIES) {
          if (options.fallbackModel) {
            throw new FallbackTriggeredError(options.model, options.fallbackModel)
          }
          throw new CannotRetryError(
            new Error(REPEATED_529_ERROR_MESSAGE),
            retryContext,
          )
        }
      }

      // 计算重试延迟
      const delayMs = getRetryDelay(attempt, getRetryAfter(error))

      // 向用户通知重试
      if (error instanceof APIError) {
        yield createSystemAPIErrorMessage(error, delayMs, attempt, maxRetries)
      }

      await sleep(delayMs, options.signal, { abortError })
    }
  }
}
```

### 21.3.2 指数退避与抖动

重试延迟采用指数退避算法，并添加随机抖动避免"惊群效应"：

```typescript
export function getRetryDelay(
  attempt: number,
  retryAfterHeader?: string | null,
  maxDelayMs = 32000,
): number {
  // 优先使用服务器返回的Retry-After头
  if (retryAfterHeader) {
    const seconds = parseInt(retryAfterHeader, 10)
    if (!isNaN(seconds)) {
      return seconds * 1000
    }
  }

  // 指数退避：500ms, 1000ms, 2000ms, 4000ms, ...
  const baseDelay = Math.min(
    BASE_DELAY_MS * Math.pow(2, attempt - 1),
    maxDelayMs,
  )

  // 添加±25%的随机抖动
  const jitter = Math.random() * 0.25 * baseDelay
  return baseDelay + jitter
}
```

这种策略的优势：
- **指数增长**：初期快速重试，后期逐渐放缓
- **服务器指令优先**：尊重API返回的`Retry-After`头
- **抖动避免同步**：多个客户端不会同时重试

### 21.3.3 Fast Mode降级

Fast Mode是Claude Code的一项性能优化特性，可以在不改变模型行为的情况下显著提升响应速度。当Fast Mode触发速率限制时，系统会自动降级到标准模式：

```typescript
// 在withRetry中处理Fast Mode降级
if (
  wasFastModeActive &&
  !isPersistentRetryEnabled() &&
  error instanceof APIError &&
  (error.status === 429 || is529Error(error))
) {
  const retryAfterMs = getRetryAfterMs(error)

  if (retryAfterMs !== null && retryAfterMs < SHORT_RETRY_THRESHOLD_MS) {
    // 短暂延迟后重试，保持Fast Mode
    await sleep(retryAfterMs, options.signal, { abortError })
    continue
  }

  // 长延迟或未知延迟：切换到标准模式
  const cooldownMs = Math.max(
    retryAfterMs ?? DEFAULT_FAST_MODE_FALLBACK_HOLD_MS,
    MIN_COOLDOWN_MS,
  )
  triggerFastModeCooldown(Date.now() + cooldownMs, 'rate_limit')
  retryContext.fastMode = false
  continue
}
```

Fast Mode降级策略的设计考量：
- **短暂等待**：如果Retry-After小于20秒，等待后重试（保持模型一致性）
- **长时间限**：切换到标准模式，避免长时间阻塞用户
- **Cooling Period**：降级后持续10分钟，防止频繁切换

### 21.3.4 模型回退机制

当主模型（如Opus）持续遇到529过载错误时，系统会自动回退到备用模型（如Sonnet）：

```typescript
if (
  is529Error(error) &&
  (process.env.FALLBACK_FOR_ALL_PRIMARY_MODELS ||
    (!isClaudeAISubscriber() && isNonCustomOpusModel(options.model)))
) {
  consecutive529Errors++
  if (consecutive529Errors >= MAX_529_RETRIES) {
    if (options.fallbackModel) {
      logEvent('tengu_api_opus_fallback_triggered', {
        original_model: options.model,
        fallback_model: options.fallbackModel,
        provider: getAPIProviderForStatsig(),
      })

      throw new FallbackTriggeredError(options.model, options.fallbackModel)
    }
  }
}
```

这种机制通过特殊的`FallbackTriggeredError`实现，上层代码捕获后执行模型切换：
- **透明降级**：用户感知不到切换过程
- **功能保持**：降级后功能完全一致
- **性能差异**：Sonnet可能比Opus慢，但总比报错好

## 21.4 流式处理

流式响应是提升用户体验的关键技术。Claude Code实现了完整的流式处理管道，从底层SSE解析到上层渲染优化。

### 21.4.1 流式与非流式的双重路径

Claude Code同时支持流式和非流式两种API调用方式，并在流式失败时自动回退到非流式：

```typescript
async function* queryModel(...): AsyncGenerator<...> {
  try {
    // 首选流式处理
    const generator = withRetry(...)
    stream = await generator

    for await (const part of stream) {
      switch (part.type) {
        case 'message_start':
          partialMessage = part.message
          ttftMs = Date.now() - start
          break

        case 'content_block_delta':
          // 累积增量内容
          const contentBlock = contentBlocks[part.index]
          if (delta.type === 'text_delta') {
            contentBlock.text += delta.text
          }
          break

        case 'content_block_stop':
          // 完成一个内容块，yield给上层
          const m: AssistantMessage = { /* ... */ }
          newMessages.push(m)
          yield m
          break
      }
    }
  } catch (streamingError) {
    // 流式失败时回退到非流式
    if (streamingError instanceof APIError || isStreamIncomplete()) {
      logEvent('tengu_streaming_fallback_to_non_streaming', { ... })

      const result = yield* executeNonStreamingRequest(...)
      const m: AssistantMessage = { ...result }
      newMessages.push(m)
      yield m
    }
  }
}
```

这种双重路径设计确保了：
- **最佳体验**：正常情况下使用流式响应，TTFB（Time To First Token）更低
- **容错能力**：流式失败时自动降级，不会中断用户操作
- **透明切换**：上层代码无需关心底层使用哪种方式

### 21.4.2 空闲超时监控

流式连接可能因为网络问题而静默中断，导致长时间无响应。Claude Code实现了空闲超时监控：

```typescript
const STREAM_IDLE_TIMEOUT_MS = parseInt(process.env.CLAUDE_STREAM_IDLE_TIMEOUT_MS || '', 10) || 90_000
const STREAM_IDLE_WARNING_MS = STREAM_IDLE_TIMEOUT_MS / 2

let streamIdleAborted = false
let streamIdleTimer: ReturnType<typeof setTimeout> | null = null

function resetStreamIdleTimer(): void {
  clearStreamIdleTimers()
  if (!streamWatchdogEnabled) return

  streamIdleTimer = setTimeout(() => {
    streamIdleAborted = true
    logForDebugging('Streaming idle timeout: no chunks received')
    releaseStreamResources()
  }, STREAM_IDLE_TIMEOUT_MS)
}

// 在流式循环中每次收到数据时重置
for await (const part of stream) {
  resetStreamIdleTimer()
  // 处理part...
}
```

空闲监控的保护机制：
- **检测静默中断**：90秒无数据自动终止
- **警告日志**：45秒无数据时记录警告
- **资源释放**：超时后正确释放HTTP连接

### 21.4.3 流式停滞检测

除了超时检测，Claude Code还监控流式响应中的停滞现象：

```typescript
const STALL_THRESHOLD_MS = 30_000 // 30秒
let totalStallTime = 0
let stallCount = 0

for await (const part of stream) {
  const now = Date.now()

  if (lastEventTime !== null) {
    const timeSinceLastEvent = now - lastEventTime
    if (timeSinceLastEvent > STALL_THRESHOLD_MS) {
      stallCount++
      totalStallTime += timeSinceLastEvent
      logEvent('tengu_streaming_stall', {
        stall_duration_ms: timeSinceLastEvent,
        stall_count: stallCount,
        total_stall_time_ms: totalStallTime,
      })
    }
  }
  lastEventTime = now

  // 处理part...
}
```

停滞检测的价值：
- **性能分析**：识别API响应异常
- **用户体验**：长时间停滞时提供反馈
- **问题诊断**：收集数据用于后端优化

### 21.4.4 资源泄漏防护

流式响应处理中的资源泄漏是一个常见问题。Claude Code通过显式释放机制确保不泄漏任何资源：

```typescript
function releaseStreamResources(): void {
  if (stream?.controller && !stream.controller.signal.aborted) {
    stream.controller.abort()
  }
  if (streamResponse?.body) {
    streamResponse.body.cancel().catch(() => {})
  }
  stream = undefined
  streamResponse = undefined
}

try {
  // 流式处理逻辑
} finally {
  // 确保资源总是被释放
  releaseStreamResources()
}
```

资源泄漏的防护要点：
1. **Abort Signal**：取消底层HTTP请求
2. **Body Cancel**：取消Response body的读取
3. **Finally块**：确保异常时也能执行清理
4. **引用清空**：帮助垃圾回收

## 21.5 模型管理

Claude Code支持多种Claude模型，并且需要处理不同模型的特性差异。模型管理模块负责抽象这些差异，为上层提供统一的接口。

### 21.5.1 模型字符串处理

模型标识在不同提供商有不同的格式。Claude Code使用规范化函数统一处理：

```typescript
export function normalizeModelStringForAPI(model: string): string {
  // Bedrock: anthropic.claude-3-opus-20240229-v1:0
  if (getAPIProvider() === 'bedrock') {
    return getBedrockModelString(model)
  }

  // Vertex: claude-3-opus@20240229
  if (getAPIProvider() === 'vertex') {
    return getVertexModelString(model)
  }

  // 第一方API: claude-opus-4-20250514
  return model
}
```

这种抽象使得上层代码可以使用统一的模型名称，而底层自动转换为各API所需的格式。

### 21.5.2 模型特性检测

不同模型支持不同的功能。Claude Code通过特性检测来实现条件功能：

```typescript
export function modelSupportsThinking(model: string): boolean {
  const normalized = normalizeModelStringForAPI(model)
  // 只有4.6+模型支持thinking
  return (
    normalized.includes('claude-sonnet-4-6') ||
    normalized.includes('claude-opus-4-6')
  )
}

export function modelSupportsStructuredOutputs(model: string): boolean {
  const normalized = normalizeModelStringForAPI(model)
  return modelSupportsThinking(model) || /* 其他支持的结构化输出模型 */
}

export function modelSupportsEffort(model: string): boolean {
  return modelSupportsThinking(model)
}
```

特性检测的好处：
- **向前兼容**：新模型添加时自动支持
- **条件功能**：根据模型能力动态调整
- **类型安全**：编译时检查模型特性使用

### 21.5.3 Max Tokens管理

不同模型有不同的默认和最大token限制。Claude Code实现了动态调整机制：

```typescript
export function getMaxOutputTokensForModel(model: string): number {
  const maxOutputTokens = getModelMaxOutputTokens(model)

  // Slot-reservation cap: 默认降至8k以节省配额
  const defaultTokens = isMaxTokensCapEnabled()
    ? Math.min(maxOutputTokens.default, CAPPED_DEFAULT_MAX_TOKENS)
    : maxOutputTokens.default

  // 环境变量覆盖
  const result = validateBoundedIntEnvVar(
    'CLAUDE_CODE_MAX_OUTPUT_TOKENS',
    process.env.CLAUDE_CODE_MAX_OUTPUT_TOKENS,
    defaultTokens,
    maxOutputTokens.upperLimit,
  )
  return result.effective
}
```

这种多层次的token管理确保了：
- **合理默认值**：不同模型有合适的默认值
- **用户控制**：通过环境变量可以覆盖
- **安全边界**：不会超过模型的硬性限制

### 21.5.4 模型回退的完整流程

当主模型不可用时，Claude Code实现了完整的回退流程：

```typescript
// 1. 在withRetry中检测到需要回退
if (consecutive529Errors >= MAX_529_RETRIES && options.fallbackModel) {
  throw new FallbackTriggeredError(options.model, options.fallbackModel)
}

// 2. 上层捕获并执行回退
try {
  return await queryModel(...)
} catch (error) {
  if (error instanceof FallbackTriggeredError) {
    logEvent('model_fallback_initiated', {
      from: error.originalModel,
      to: error.fallbackModel,
    })

    // 使用fallback model重试
    return await queryModel(..., {
      ...options,
      model: error.fallbackModel,
    })
  }
  throw error
}
```

回退流程的完整性体现在：
- **透明切换**：用户无感知
- **日志记录**：用于分析和调试
- **单次回退**：避免级联失败

## 21.6 小结

Claude Code的API层是一个精心设计的工程系统，它处理了从认证到重试、从流式到降级的各种复杂场景。本章的核心要点：

1. **分层架构**：清晰的职责分离使每个层次都可以独立测试和优化
2. **OAuth PKCE**：安全而流畅的认证体验，无需用户管理API Key
3. **智能重试**：指数退避、服务器指令优先、模型回退等多层次策略
4. **流式处理**：双重路径、空闲监控、停滞检测确保响应的可靠性
5. **模型管理**：统一的抽象接口处理不同提供商和模型的差异

这个API层的设计哲学是：**优雅降级，永不失败**。无论遇到何种错误，系统都应该尝试恢复而不是简单地报错退出。这种哲学贯穿于从重试逻辑到模型回退的每一个设计决策中。

> "A robust API layer doesn't just handle success — it handles failure gracefully." —— API设计原则

下一章将探讨Claude Code中应用的各种设计模式，从命令模式到策略模式，从观察者模式到中间件模式，我们将看到经典设计思想在现代AI应用中的精彩演绎。
