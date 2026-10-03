# Connection

RTCAgentClient WebSocket 连接生命周期（Centrifuge 驱动）。

**所属 package**: `client`

## 关键代码文件

- [client.ts](~/Workspaces/rtc-agent/web-components/packages/client/src/client.ts) — `RTCAgentClient` 类（1186 行），Centrifuge WebSocket 封装
- [client.ts:40-49](~/Workspaces/rtc-agent/web-components/packages/client/src/client.ts#L40-L49) — `GapFillTimeoutError` 类（gap fill 超时专用错误）
- [types.ts](~/Workspaces/rtc-agent/web-components/packages/client/src/types.ts) — `ConnectionState` 类型定义、`IRTCAgentClient` 接口

## 连接状态机

```mermaid
stateDiagram-v2
    [*] --> disconnected

    disconnected --> connecting : connect()
    connecting --> connected : centrifuge 'connected' event
    connecting --> disconnected : centrifuge 'disconnected' event

    connected --> disconnected : disconnect()
    connected --> reconnecting : network error / token expired

    reconnecting --> connecting : Centrifuge auto-reconnect
    reconnecting --> disconnected : shouldReconnect = false (relogin)

    note right of connecting
        wasConnected = false → 'connecting'
        wasConnected = true  → 'reconnecting'
    end note
```

## 生命周期流程

```mermaid
sequenceDiagram
    participant Caller
    participant RTCAgentClient
    participant Centrifuge
    participant Server

    Caller->>RTCAgentClient: connect()
    RTCAgentClient->>RTCAgentClient: setConnectionState('connecting')
    RTCAgentClient->>Centrifuge: new Centrifuge(endpoint, {getToken})
    RTCAgentClient->>Centrifuge: connect()
    Centrifuge->>Server: WebSocket handshake + JWT
    Server-->>Centrifuge: connected
    Centrifuge->>Centrifuge: emit 'connected'
    RTCAgentClient->>RTCAgentClient: wasConnected = true
    RTCAgentClient->>RTCAgentClient: setConnectionState('connected')
    RTCAgentClient->>RTCAgentClient: subscribeChannels()
    Note over RTCAgentClient: Subscribe topic:u={userId} + live:u={userId}
```

## Token 过期处理

```mermaid
flowchart TD
    A[Centrifuge getToken callback] --> B{Token expired?}
    B -->|No| C[Return token]
    B -->|Yes| D[Call onTokenExpired]
    D --> E{Action?}
    E -->|'refresh'| F[Call getToken again, return new token]
    E -->|'relogin'| G[shouldReconnect = false]
    G --> H[centrifuge.disconnect]
    H --> I[setConnectionState 'disconnected']
```

## PersistenceController 连接重试

`PersistenceController`（主线程侧）在 `connect()` 时支持自动重试和 generation counter 防竞态：

```mermaid
sequenceDiagram
    participant RC as rtc-agent (root)
    participant PC as PersistenceController
    participant WB as WorkerBridge
    participant SW as SharedWorker

    RC->>PC: connect()
    Note over PC: gen = _connectGeneration
    PC->>PC: _connecting = _connectWorker(config)

    loop Up to MAX_CONNECT_RETRIES (2)
        PC->>WB: new WorkerBridge()
        PC->>WB: initWorker() (fetch + blob URL)
        Note over PC: Checkpoint 1: bridge !== disconnected?
        PC->>WB: init(config)
        Note over PC: Checkpoint 2: bridge !== disconnected?
        PC->>WB: installVirtualFSProxy()
        PC->>WB: core.connect()
        PC->>PC: new MasterLock(userId)
    end

    PC-->>RC: PersistenceLayer adapter
```

### Generation Counter 防竞态

```mermaid
flowchart TD
    C["connect()"] --> G["Capture gen = _connectGeneration"]
    G --> P["_connecting = _connectWorker(config)"]
    P --> F[".finally() block"]
    F --> CK{"_connectGeneration === gen?"}
    CK -->|Yes| CLR["_connecting = undefined"]
    CK -->|No| SKIP["Skip (stale promise)"]

    D["disconnect()"] --> BUMP["_connectGeneration++"]
    BUMP --> CLR2["_connecting = undefined"]
```

**Why**: `disconnect()` 可能在 `_connecting` promise 仍在 pending 时被调用。递增 generation 后，旧 promise 的 `finally` 检测到 generation 不匹配，不会错误地清除新的 `_connecting`。

**Checkpoint 机制**: `_connectWorkerOnce()` 在两个 `await` 点（`initWorker()` 和 `init()`）后检查 `this._workerBridge !== bridge`，如果在 await 期间被 `disconnect()` 打断，优雅退出并清理。

## disconnect 清理

`disconnect()` 执行完整的状态清理，确保 reconnect 时从干净状态启动：

```mermaid
flowchart TD
    A["disconnect()"] --> B["shouldReconnect = false"]
    B --> C["wasConnected = false"]
    C --> C1["clearTimeout(_reconnectTimer)"]
    C1 --> D["_gapFillAbortControllers.forEach(c => c.abort())"]
    D --> E["_gapFillAbortControllers.clear()"]
    E --> F["gapFillTasks.clear()"]
    F --> G["isGapFillProcessing = false<br/>isUpdateProcessing = false"]
    G --> H["centrifuge.disconnect()"]
    H --> I["centrifuge = null"]
    I --> J["subscriptions.clear()"]
    J --> K["pendingUpdates.length = 0"]
    K --> L["lastOffsetCache.clear()"]
    L --> M["setConnectionState('disconnected')"]
```

关键设计：

| 清理项 | 防止的问题 |
| --- | --- |
| `_reconnectTimer` | 清除 message-size-limit 延迟重连定时器，防止 zombie reconnect |
| `_gapFillAbortControllers` | 中断 zombie gap fill 等待定时器 |
| `gapFillTasks` + `isGapFillProcessing` | 防止旧 gap fill 状态污染新连接 |
| `isUpdateProcessing` | 防止旧 update executor 标志位阻塞新连接的消息处理 |
| `lastOffsetCache` | offset 去重缓存与新连接的 offset 状态不一致 |

## Message Size Limit 恢复

服务端拒绝连接时（`message size limit exceeded`），Centrifuge 不会自动重试此类型的服务端拒绝。客户端检测到后延迟 3 秒手动重连：

```mermaid
flowchart TD
    A["centrifuge 'disconnected' event"] --> B{"reason ===<br/>'message size limit exceeded'?"}
    B -->|Yes| C["_reconnectTimer = setTimeout(3000ms)"]
    C --> D{"after 3s:<br/>shouldReconnect &&<br/>state === 'disconnected'?"}
    D -->|Yes| E["reconnect()"]
    D -->|No| F["Skip"]
    B -->|No| G["Normal disconnect handling"]
```

`RECONNECT_AFTER_SIZE_LIMIT_DELAY_MS = 3000` — 给服务端缓冲时间后重试。

## Invalid Token 处理

服务端可能在 JWT 未过期时拒绝 token（服务重启导致 JWT secret 轮换、服务端 token 撤销等）。检测到 `invalid token` 后立即断开 Centrifuge 实例，防止其用缓存旧 token 自动重连：

```mermaid
flowchart TD
    A["centrifuge 'disconnected'<br/>reason: 'invalid token'"] --> B["centrifuge = null<br/>(清除引用)"]
    B --> C["oldCentrifuge.disconnect()<br/>(阻止自动重连)"]
    C --> D["handleInvalidToken()"]
    D --> E["onTokenExpired()"]
    E -->|"'refresh'"| F["setTimeout(1000ms) → reconnect()"]
    E -->|"'relogin'"| G["shouldReconnect = false<br/>state = 'disconnected'"]
```

`RECONNECT_AFTER_TOKEN_REFRESH_DELAY_MS = 1000` — token 刷新后延迟 1 秒重连，避免快速重试循环。

## 关键注释摘录

> **Token 过期检测** — [client.ts:1020-1046](~/Workspaces/rtc-agent/web-components/packages/client/src/client.ts#L1020-L1046)
> _Check whether a JWT has expired. Parses the exp claim from the JWT payload (Unix timestamp in seconds) and compares it with the current time. Returns false on parse failure (to avoid blocking connections due to malformed tokens)._

<!-- separator -->

> **Invalid Token 处理** — [client.ts:169-179](~/Workspaces/rtc-agent/web-components/packages/client/src/client.ts#L169-L179)
> _Detect server-side token rejection (e.g. "invalid token"). Even if the JWT is not expired, we need to trigger the refresh mechanism. Critical fix: immediately disconnect the Centrifuge instance to prevent it from auto-reconnecting with the cached old token._

<!-- separator -->

> **disconnect 清理** — [client.ts:209-235](~/Workspaces/rtc-agent/web-components/packages/client/src/client.ts#L209-L235)
> disconnect() 清理所有状态：abort gap fill、clear subscriptions、clear pending updates、clear lastOffsetCache。确保 reconnect 时状态干净。

## Gap Fill 取消机制

`waitForGapFill()` 使用 `AbortController` 实现可中断的轮询等待，`disconnect()` 批量中断所有等待：

```mermaid
flowchart TD
    I["waitForGapFill(channel, targetOffset)"] --> J["Create AbortController per channel:offset"]
    J --> K["Store in _gapFillAbortControllers"]
    K --> L["Poll loop (100ms interval)"]
    L --> M{aborted?}
    M -->|Yes| N["Throw Error: 'Gap fill aborted'"]
    M -->|No| O{currentOffset >= target?}
    O -->|Yes| P["Return (caught up)"]
    O -->|No| L
    N --> Q["finally: cleanup AbortController"]
    P --> Q
```

关键设计：

- 每个 gap fill 等待创建独立的 `AbortController`，按 `channel:offset` 为 key 存储
- `disconnect()` 批量 abort 所有控制器，防止 zombie 定时器
- `waitForGapFill` 的 `finally` 块仅在 controller 仍是当前存储的实例时才删除（防止并发调用互相覆盖）
- 超时（30s）抛出 `GapFillTimeoutError` 而非静默返回
- `isUpdateProcessing` 和 `isGapFillProcessing` 双标志重置，确保 reconnect 后 `processUpdateQueue` 和 gap fill 调度器从干净状态启动

## 跨维度关联

- [[ConnectionState]] — 连接状态枚举定义
- [[Publication]] — 连接后订阅的 topic/live 通道
- [[StreamingEvents]] — live 通道的流式消息推送（streaming_status 状态机）
- [[ProtocolTypes]] — RPC 方法常量与请求/响应类型定义
- [[AuthController]] — 提供 token 给 RTCAgentClient
- [[SharedWorker]] — RTCAgentClient 在 Worker 内运行
- [[PersistenceLayer]] — RTCAgentClient 由 PersistenceLayer 创建并持有
- [[PersistenceController]] — 驱动连接/断开生命周期的组件层控制器
- [[RootComponentHelpers]] — `logout()` 和 `disconnectedCallback` 中的连接清理编排
- [[ConcurrencyPatterns]] — disconnect 批量 AbortController + Generation Counter + Checkpoint
