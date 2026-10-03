# Go 并发规范

> _并发是 Go 的超能力，也是 bug 的放大器。用纪律约束并发，让它在可控的边界内释放力量。_

---

## 1. goroutine 生命周期

### 铁律：每个 goroutine 必须有退出机制

没有退出机制的 goroutine 是泄漏。泄漏的 goroutine 持有引用，阻止 GC，最终耗尽内存。

```go
// ❌ 无法退出的 goroutine
go func() {
    for {
        event := <-eventChan
        process(event)  // 永远不会停
    }
}()

// ✅ 通过 channel 或 context 退出
func (s *Server) startEventListener(ctx context.Context) {
    go func() {
        for {
            select {
            case <-ctx.Done():
                return
            case event := <-eventChan:
                process(event)
            }
        }
    }()
}
```

### 用 errgroup 管理一组 goroutine

`errgroup` 是一组 goroutine 的最佳管理方式——统一等待、统一取消、统一错误收集。

```go
func (s *Service) LoadAllData(ctx context.Context) error {
    g, ctx := errgroup.WithContext(ctx)

    g.Go(func() error {
        defer func() {
            if r := recover(); r != nil {
                log.Printf("goroutine panic in loadUsers: %v", r)
            }
        }()
        return s.loadUsers(ctx)
    })

    g.Go(func() error {
        defer func() {
            if r := recover(); r != nil {
                log.Printf("goroutine panic in loadRooms: %v", r)
            }
        }()
        return s.loadRooms(ctx)
    })

    g.Go(func() error {
        defer func() {
            if r := recover(); r != nil {
                log.Printf("goroutine panic in loadMessages: %v", r)
            }
        }()
        return s.loadMessages(ctx)
    })

    return g.Wait()  // 任一错误，取消其余
}
```

**提示**：当多个 goroutine 需要相同的 panic 拦截逻辑时，提取为辅助函数（如 `safeGo(g, fn)`），避免在每个 `g.Go` 中重复 `defer recover()`。

**例外：用 `sync.WaitGroup` 替代 `errgroup`**。当并行操作中单个失败不应取消其余操作时（如并行加载多个文件，一个失败只需记录错误，不影响其他文件），使用 `WaitGroup` + 预分配结果数组。`errgroup` 的语义是"任一错误，取消其余"，不适用此场景。

```go
// ✅ WaitGroup：单个失败不影响其余（panic 拦截见 Section 7 完整示例）
results := make([]loadResult, len(files))
var wg sync.WaitGroup
for i, file := range files {
    wg.Add(1)
    go func(idx int, f File) {
        defer wg.Done()
        defer func() {
            if r := recover(); r != nil {
                results[idx] = loadResult{err: fmt.Errorf("panic: %v", r)}
            }
        }()
        data, err := loadFile(ctx, f)
        results[idx] = loadResult{data: data, err: err}  // 错误被记录，不传播
    }(i, file)
}
wg.Wait()
// 遍历 results 处理错误
```

### 何时选择 errgroup vs WaitGroup

| 场景 | 选择 | 原因 |
|------|------|------|
| 所有操作必须全部成功 | `errgroup` | 任一失败，取消其余 |
| 单个失败不应影响其余 | `WaitGroup` | 独立收集结果和错误 |
| 需要统一错误返回 | `errgroup` | `g.Wait()` 返回首个错误 |
| 需要按索引收集结果 | `WaitGroup` | 预分配数组按索引写入 |

### 记录 goroutine 的用途

当 goroutine 的生命周期超过当前函数时，用注释说明它的职责和退出方式。

```go
// 后台清理过期会话，每 5 分钟执行一次。
// 通过 ctx 取消退出，在 Server.Shutdown() 中调用 cancel()。
go s.runSessionCleanup(ctx)
```

---

## 2. goroutine panic 拦截

### 核心规则

> **一个 goroutine 的 panic 会杀死整个进程，而不只是那个 goroutine。**

`recover()` 只对当前 goroutine 有效。`go func()` 启动的新 goroutine 不在你的 recover 边界内。

### 必须拦截

| 场景 | 原因 |
|------|------|
| `go func()` 启动的任何 goroutine | 不在调用方的 recover 范围内 |
| `sync.WaitGroup` 中的 goroutine | 与 `errgroup` 一样，WaitGroup 不自动 recover |
| 长生命周期的 goroutine（事件监听、后台 worker） | 一个 panic 不应杀死整个服务 |
| goroutine 池 / worker pool | 一个坏任务不应杀死整个池 |
| `errgroup` 中的 goroutine | `errgroup` 默认不 recover，需自行包装 |

### 不需要拦截

| 场景 | 原因 |
|------|------|
| HTTP handler | 框架已 recover |
| 已被 recover 边界覆盖的 goroutine | 不重复防御 |
| 启动阶段的 panic（配置加载失败） | 应直接崩溃，让运维发现问题 |

### 判断规则

```
如果你写了 `go func()`，
那个 func 里就应该有 defer recover()。
除非你能证明它一定在别人的 recover 边界内。
```

### 实现模式

```go
// ✅ 使用 logger.SafeGo（项目统一实现，带名称标识和 panic 日志）
logger.SafeGo("queue-worker", func() {
    processEvent(event)
})

// logger.SafeGo 内部实现：
// func SafeGo(name string, fn func()) {
//     go func() {
//         defer func() {
//             if r := recover(); r != nil {
//                 Error(context.Background(), "goroutine panic",
//                     zap.String("name", name),
//                     zap.Any("recover", r),
//                     zap.String("stack", string(debug.Stack())))
//             }
//         }()
//         fn()
//     }()
// }

// ✅ errgroup 中的 recover（errgroup 不自动 recover，需手动包装）
g.Go(func() error {
    defer func() {
        if r := recover(); r != nil {
            log.Printf("goroutine panic: %v", r)
        }
    }()
    return doWork()
})
```

**规则**：`internal/` 中的新代码使用 `logger.SafeGo("name", fn)` 启动后台 goroutine。`name` 参数用于日志标识，格式为 `"模块.操作"`（如 `"oss3.cleanup"`、`"loopWorkflow.scheduleNext"`）。不要手写 `go func()` + `defer recover()`。

**例外：`pkg/` 包**。`pkg/` 下的独立可复用包（如 `pkg/turn-agent/`、`pkg/rtc-queue/`、`pkg/rtc-oss3/`）不依赖 `internal/`，无法使用 `logger.SafeGo`。这些包可以手写 `go func()` + `defer recover()`，但必须遵循以下约束：

- 每个 goroutine 必须有 `defer recover()` 拦截 panic
- 恢复日志使用包自身的日志系统，事件名遵循 `snake_case` 格式
- 恢复日志包含 `stack` 字段（`string(debug.Stack())`），便于定位问题

```go
// ✅ pkg/ 包中的 goroutine：手动 recover + 结构化日志
go func() {
    defer func() {
        if r := recover(); r != nil {
            mgr.log(context.Background(), LogLevelError, "session_manager.lock_renewal_panic", map[string]any{
                "session_id": mgr.sessionID,
                "panic":      fmt.Sprintf("%v", r),
                "stack":      string(debug.Stack()),
            })
        }
    }()
    mgr.runLockRenewal(ctx)
}()
```

### 嵌套 recover：回调安全

当 panic 恢复逻辑需要调用外部回调（如 `FailTurn`、`CompleteWork`）时，回调本身也可能 panic（如数据库不可达）。如果不在回调外层再套一层 `recover()`，二次 panic 会穿透恢复逻辑，杀死整个进程。

```go
// ✅ 嵌套 recover：回调 panic 不影响外层恢复
defer func() {
    r := recover()
    if r == nil {
        return
    }
    pErr := fmt.Errorf("panic in callback: %v", r)
    log.Error("turn.panicked", "error", pErr, "stack", string(debug.Stack()))

    if turnID != "" {
        func() {
            defer func() { _ = recover() }()  // 嵌套 recover：吞掉回调 panic
            _ = cfg.FailTurn(ctx, turnID, pErr)
        }()
    }
}()

// ❌ 无嵌套 recover：FailTurn panic 会穿透外层恢复
defer func() {
    r := recover()
    if r == nil {
        return
    }
    pErr := fmt.Errorf("panic: %v", r)
    cfg.FailTurn(ctx, turnID, pErr)  // 如果 FailTurn 也 panic，进程崩溃
}()
```

**约束**：

- 恢复逻辑中调用的任何外部回调（回调可能在不可靠环境中执行），必须用 `func() { defer recover(); ... }()` 包装
- 嵌套 recover 中不需要日志——外层 recover 已经记录了 panic 详情
- 嵌套 recover 的 IIFE（立即调用函数表达式）中忽略回调返回值（`_ =`），因为此时关注的是"尽力恢复"而非结果

---

## 3. channel 模式

### 有缓冲 vs 无缓冲

| 类型 | 行为 | 适用场景 |
|------|------|---------|
| **无缓冲** | 发送方阻塞直到接收方就绪 | 同步信号、严格的生产-消费配对 |
| **有缓冲** | 发送方在缓冲区满前不阻塞 | 生产者不应被慢消费者阻塞 |

```go
// ✅ 无缓冲：同步信号
done := make(chan struct{})
go func() {
    doWork()
    close(done)  // 通知完成
}()
<-done  // 等待完成

// ✅ 有缓冲：异步事件
events := make(chan Event, 100)
go func() {
    for event := range events {
        processEvent(event)
    }
}()
// 生产者不会被慢消费者阻塞（直到缓冲区满）
events <- Event{...}
```

### 关闭 channel 的规则

- **只有发送方关闭 channel**，接收方不应关闭
- **不要关闭已关闭的 channel**（会 panic）
- **关闭后仍可读取**，读到零值
- **用 `ok` 判断 channel 是否关闭**

```go
// ✅ 接收方安全读取
for {
    msg, ok := <-ch
    if !ok {
        return  // channel 已关闭
    }
    process(msg)
}

// ✅ 用 range 遍历（自动处理关闭）
for msg := range ch {
    process(msg)
}
```

### fan-in / fan-out

```go
// fan-in：多个 channel 合并为一个
func merge(channels ...<-chan int) <-chan int {
    var wg sync.WaitGroup
    out := make(chan int)

    for _, ch := range channels {
        wg.Add(1)
        go func(c <-chan int) {
            defer wg.Done()
            for v := range c {
                out <- v
            }
        }(ch)
    }

    go func() {
        wg.Wait()
        close(out)
    }()

    return out
}

// fan-out：一个 channel 分发给多个 worker
func fanOut(input <-chan int, workers int) []<-chan int {
    outputs := make([]<-chan int, workers)
    for i := 0; i < workers; i++ {
        outputs[i] = worker(input)
    }
    return outputs
}
```

---

## 4. context 使用

### 作为第一个参数

```go
// ✅ context 是第一个参数
func GetUser(ctx context.Context, id string) (*User, error)

// ❌ context 不是第一个参数
func GetUser(id string, ctx context.Context) (*User, error)
```

### 不存 struct 字段

context 的生命周期是请求级的，不应跨请求复用。

```go
// ❌ context 存为字段
type Service struct {
    ctx context.Context  // 哪个请求的 context？
    repo UserRepository
}

// ✅ context 通过参数传递
type Service struct {
    repo UserRepository
}

func (s *Service) GetUser(ctx context.Context, id string) (*User, error) {
    return s.repo.Find(ctx, id)
}
```

### 传递请求级数据

context 可以传递请求 ID、用户信息等，但不要滥用。

```go
// ✅ 传递请求 ID
ctx = context.WithValue(ctx, requestIDKey, reqID)

// 获取
reqID := ctx.Value(requestIDKey).(string)
```

**避免在 context 中传递大量数据**。如果需要传递多个值，定义一个 struct。

### 集中管理 context key

所有跨层传递的 context key 必须在 `internal/infra/contextx/` 集中定义，使用未导出的 struct 类型防止外部包创建冲突的 key。新增功能不得在各自包内定义新的 context key。

```go
// ✅ 在 contextx 集中定义
// internal/infra/contextx/keys.go
func WithClientInfo(ctx context.Context, userID uuid.UUID, deviceID string) context.Context
func GetUserID(ctx context.Context) (uuid.UUID, bool)

// 使用
ctx = contextx.WithClientInfo(ctx, userID, deviceID)
userID, _ := contextx.GetUserID(ctx)

// ❌ 在业务包内定义 key
// internal/user/keys.go
type ctxKey string
const userKey ctxKey = "user"
```

**例外**：`pkg/` 下的独立可复用包（如 `pkg/turn-agent/`）可以在包内定义自己的 context key，因为它们不依赖 `internal/`。`pkg/turn-agent/` 定义了 `WithUserID` / `UserIDFromContext`、`WithSessionID` / `SessionIDFromContext` 等，供 agent loop 内部使用。新增 `pkg/` 包如需 context key，遵循同样的集中模式（一个包一个文件定义所有 key）。

```go
// ✅ pkg/ 包内的 context key 集中定义（pkg/turn-agent/types.go）
type ctxUserIDKey struct{}
type ctxSessionIDKey struct{}
type ctxTurnIDKey struct{}

// WithUserID / UserIDFromContext — 供下游代码（如文件加载）构造用户级 key
func WithUserID(ctx context.Context, userID string) context.Context
func UserIDFromContext(ctx context.Context) string

// ❌ pkg/ 包内散落定义
// pkg/turn-agent/agent.go: type userIDKey string
// pkg/turn-agent/process.go: type sessionKey string
```

### 后台 goroutine 必须传递身份

启动后台 goroutine 时，必须确保 UserID 等身份信息通过 context 传递。后台 worker 无法访问 HTTP 请求的 context，需要显式注入。

```go
// ✅ 显式传递 UserID 到后台 worker
func (h *Handler) processAsync(ctx context.Context, userID uuid.UUID) {
    bgCtx := context.WithoutCancel(ctx)
    bgCtx = contextx.WithClientInfo(bgCtx, userID, "")  // 注入身份
    go func() {
        loadAttachments(bgCtx, userID)  // worker 可从 bgCtx 获取 userID
    }()
}

// ❌ 后台 worker 无法获取 UserID
go func() {
    loadAttachments(ctx, userID)  // ctx 已 cancel，且 userID 未注入
}()
```

### 跨队列边界传递身份

context 值无法跨越队列边界（如 Redis PUB/SUB、消息队列）——序列化会丢失 context 中的值。当工作项通过队列传递时，必须将身份信息直接嵌入 payload struct，在消费端重新注入到 context。

```go
// ✅ 身份嵌入 payload struct
type WorkPayload struct {
    SessionID string `json:"session_id"`
    UserID    string `json:"user_id"`     // 身份字段，用于消费端注入 context
    // ... 其他业务字段
}

// 生产端：从 context 提取身份，嵌入 payload
func (h *Handler) enqueueWork(ctx context.Context, sessionID string) {
    userID, _ := contextx.GetUserID(ctx)
    payload := WorkPayload{
        SessionID: sessionID,
        UserID:    userID.String(),  // 嵌入身份
    }
    h.queue.Publish(payload)
}

// 消费端：从 payload 提取身份，注入 context
func (w *Worker) processWork(payload WorkPayload) {
    ctx := context.Background()
    uid, _ := uuid.Parse(payload.UserID)
    ctx = contextx.WithClientInfo(ctx, uid, "")  // 重新注入身份
    // 现在 ctx 中有 UserID，后续函数可通过 contextx.GetUserID(ctx) 获取
    loadAttachments(ctx, uid)
}

// ❌ 假设 context 能跨队列传递
func (w *Worker) processWork(ctx context.Context, payload WorkPayload) {
    // ctx 是 worker 的 context，不包含原始请求的 UserID！
    loadAttachments(ctx, ???)  // UserID 丢失
}
```

**约束**：

- 队列 payload struct 必须包含身份字段（`UserID`、可选 `DeviceID`）
- 消费端必须从 payload 提取身份并注入 context，不假设 context 携带身份信息
- 多源 fallback：当 payload 有多个可能的身份来源时（如 newItems、interrupted、unhandled），按优先级依次尝试
- payload struct 同时传播 OpenTelemetry trace context（`TraceID`、`SpanID`），详见[可观测性规范 - 跨队列边界传播 Trace Context](./observability.md#跨队列边界传播-opentelemetry-trace-context)

### 多源 Fallback 身份提取

当消费端从多个工作项列表中提取身份时，按优先级依次尝试，直到找到第一个非空值。不同列表的优先级反映了数据的新鲜度：新工作项 > 被中断的工作项 > 未处理的工作项。

```go
// ✅ 多源 fallback：按优先级依次尝试提取 UserID
var userID string
for _, item := range newItems {
    if item.UserID != "" {
        userID = item.UserID
        break
    }
}
if userID == "" {
    for _, item := range interrupted {
        if item.UserID != "" {
            userID = item.UserID
            break
        }
    }
}
if userID == "" {
    for _, item := range unhandled {
        if item.UserID != "" {
            userID = item.UserID
            break
        }
    }
}
if userID != "" {
    ctx = WithUserID(ctx, userID)
}

// ❌ 只从一个来源提取，可能丢失身份
if len(newItems) > 0 {
    ctx = WithUserID(ctx, newItems[0].UserID)
}
// 如果 newItems 全部没有 UserID（例如旧版本 payload），身份丢失
```

**适用场景**：checkpoint resume（从 interrupted/unhandled/newItems 三个列表恢复身份）、批量任务处理（主任务无身份时从子任务提取）。

**判断标准**：身份来源可能有多个，且不同来源的可靠性不同时，使用 fallback 链。单一来源直接用 `payload.UserID` 即可。

---

## 5. sync 原语

### sync.Mutex

```go
type Counter struct {
    mu    sync.Mutex
    count int64
}

// ✅ 锁粒度尽可能小
func (c *Counter) Increment() {
    c.mu.Lock()
    defer c.mu.Unlock()
    c.count++
}

// ✅ 读操作可以用 RWMutex
type Cache struct {
    mu    sync.RWMutex
    data  map[string]any
}

func (c *Cache) Get(key string) (any, bool) {
    c.mu.RLock()
    defer c.mu.RUnlock()
    val, ok := c.data[key]
    return val, ok
}

func (c *Cache) Set(key string, val any) {
    c.mu.Lock()
    defer c.mu.Unlock()
    c.data[key] = val
}
```

**规则**：

- 用 `defer` 解锁，避免遗漏
- 锁粒度尽可能小，不要锁整个函数
- 不要在持锁时做 I/O 或网络调用

### sync.WaitGroup

```go
var wg sync.WaitGroup

for i := 0; i < 10; i++ {
    wg.Add(1)  // ✅ 在启动 goroutine 前 Add
    go func(id int) {
        defer wg.Done()
        process(id)
    }(i)
}

wg.Wait()
```

**规则**：

- `Add` 在 `go` 之前调用，避免竞态
- `Done` 用 `defer` 调用，确保 panic 时也能执行
- 不要把 WaitGroup 传值，传指针

### sync.Once

```go
var (
    instance *Singleton
    once     sync.Once
)

func GetInstance() *Singleton {
    once.Do(func() {
        instance = &Singleton{}
    })
    return instance
}
```

适用场景：单例初始化、一次性配置加载。

---

## 6. 竞态安全

### 共享状态最小化

```go
// ❌ 多个 goroutine 共享可变状态
var users []*User

go func() {
    users = append(users, newUser)  // 竞态
}()

// ✅ 用 channel 传递，而非共享
usersCh := make(chan *User, 100)

go func() {
    for user := range usersCh {
        users = append(users, user)  // 单 goroutine 消费，安全
    }
}()
```

### -race 检测

```bash
go test -race ./...   # 所有测试必须通过竞态检测
```

CI 中必须开启 `-race`，本地开发也建议开启。

### 何时用 channel 替代 mutex

| 场景 | 选择 |
|------|------|
| 传递数据所有权 | channel |
| 协调多个 goroutine | channel 或 WaitGroup |
| 保护内部状态（如 cache） | mutex |
| 简单的计数器 | mutex（性能更好） |

---

## 7. 并行操作

### 用索引数组收集结果，避免竞态

多个 goroutine 并行执行独立任务时，用预分配的 `results` 数组按索引写入，避免对共享 slice 的 `append` 竞态。

```go
// ✅ 按索引写入 + panic 拦截（参见 Section 2）
results := make([]loadResult, len(files))
var wg sync.WaitGroup
for i, file := range files {
    wg.Add(1)
    go func(idx int, f File) {
        defer wg.Done()
        defer func() {
            if r := recover(); r != nil {
                log.Printf("goroutine panic loading file %s: %v", f.Name, r)
                results[idx] = loadResult{err: fmt.Errorf("panic: %v", r)}
            }
        }()
        data, err := loadFile(ctx, f)
        results[idx] = loadResult{data: data, err: err}  // 按索引写入，安全
    }(i, file)
}
wg.Wait()

// ❌ append 到共享 slice，竞态
var results []loadResult
for _, file := range files {
    go func(f File) {
        data, _ := loadFile(ctx, f)
        results = append(results, loadResult{data: data})  // 竞态！
    }(file)
}
```

**注意**：WaitGroup 中的 goroutine 同样需要 panic 拦截（参见 [Section 2](#2-goroutine-panic-拦截)）。`logger.SafeGo` 适用于不需要收集返回值的后台任务；对于需要按索引收集结果的 WaitGroup 模式，直接在 goroutine 内 `defer recover()` 并将错误写入 results 数组。

### 注释说明并发安全假设

当使用并行操作时，必须在函数注释中说明为什么该模式是安全的——哪些操作是无状态的、哪些共享变量受保护。

```go
// Concurrency model: Files are loaded in parallel using goroutines.
// This is safe because:
//   - loadFile is stateless and concurrent-safe
//   - results array is accessed by index, no race condition
//   - Context cancellation propagates to all goroutines via shared ctx
func processFiles(ctx context.Context, files []File) { ... }
```

---

## 8. Detached Context（后台 goroutine）

### 用 `context.WithoutCancel` 保留追踪信息

当 goroutine 的生命周期超过请求（如 fire-and-forget 任务调度），使用 `context.WithoutCancel(ctx)` 派生 context。它保留 trace 值和 logger，但剥离取消信号，确保 goroutine 不因请求结束而被中断。

```go
// ✅ 保留 trace，剥离 cancel
detachedCtx := context.WithoutCancel(ctx)
logger.SafeGo("scheduleNext", func() {
    // 加 timeout 防止 goroutine 泄漏
    schedCtx, cancel := context.WithTimeout(detachedCtx, 10*time.Second)
    defer cancel()
    doWork(schedCtx)
})

// ❌ 直接使用请求 ctx，请求结束后 goroutine 被取消
go func() {
    doWork(ctx)  // ctx 可能已被 cancel
}()
```

### 超时必须在 goroutine 内部创建

`context.WithTimeout` 和 `defer cancel()` 必须在 goroutine 内部创建，不能在外部创建后传入。外部的 `defer cancel()` 在函数返回时执行——此时 goroutine 可能仍在运行，timeout 被提前取消。

```go
// ❌ 超时在外部创建：defer cancel() 在 handler 返回时执行，goroutine 被提前取消
func (h *Handler) StopTurn(ctx context.Context, req *StopTurnRequest) (*StopTurnResponse, error) {
    detachedCtx, detachedCancel := context.WithTimeout(context.WithoutCancel(ctx), 30*time.Second)
    defer detachedCancel()  // ← handler 返回时执行，goroutine 可能还没完成！
    logger.SafeGo("stop-active-turns", func() {
        h.stopActiveTurns(detachedCtx, sessionID)  // ctx 可能已被 cancel
    })
    return &StopTurnResponse{}, nil  // handler 返回 → defer cancel → goroutine 被取消
}

// ✅ 超时在 goroutine 内部创建：cancel 在 goroutine 完成后执行
func (h *Handler) StopTurn(ctx context.Context, req *StopTurnRequest) (*StopTurnResponse, error) {
    detachedCtx := context.WithoutCancel(ctx)
    logger.SafeGo("stop-active-turns", func() {
        timeoutCtx, cancel := context.WithTimeout(detachedCtx, 30*time.Second)
        defer cancel()  // ← goroutine 完成后才执行
        h.stopActiveTurns(timeoutCtx, sessionID)
    })
    return &StopTurnResponse{}, nil
}
```

**规则**：当 goroutine 的生命周期超过函数返回时（fire-and-forget），超时必须在 goroutine 内部创建。唯一的例外是超时仅用于等待 goroutine 启动（如同步检查），而非约束 goroutine 的完整执行。

### `context.Background()` vs `context.WithoutCancel(ctx)`

| 场景 | 选择 | 原因 |
| --- | --- | --- |
| 后台任务需要 trace 和 logger | `context.WithoutCancel(ctx)` | 保留值，可追踪 |
| 补偿操作（cleanup/reconcile） | `context.Background()` | 原 ctx 即将返回错误响应，trace 无意义 |
| 异步清理（删除孤儿资源） | `context.Background()` | 与原请求完全解耦 |
| defer 中的资源释放（释放锁/配额） | `context.Background()` | 请求 ctx 可能已 cancel，释放操作必须执行 |

```go
// ✅ defer 中使用 Background：确保资源释放不受请求取消影响
var quotaRequestID string
if contentLength > 0 {
    qrid, err := h.uc.CheckAndReserveQuota(r.Context(), userID, contentLength)
    if err != nil { return err }
    quotaRequestID = qrid
}
uploadSucceeded := false
defer func() {
    if !uploadSucceeded && quotaRequestID != "" {
        // 使用 Background：即使请求 ctx 已 cancel，配额也必须释放
        h.uc.ReleaseQuota(context.Background(), userID, quotaRequestID)
    }
}()

// ❌ defer 中使用请求 ctx：ctx 已 cancel 时释放操作会失败
defer func() {
    if !uploadSucceeded {
        h.uc.ReleaseQuota(r.Context(), userID, quotaRequestID)  // r.Context() 可能已 cancel
    }
}()
```

**约束**：defer 中的清理操作（释放锁、归还配额、删除临时文件）必须使用 `context.Background()` 配合合理的超时。这些操作是"必须执行"的，不能被请求的取消信号阻止。

```go
// ✅ 补偿场景：使用 context.Background()，与原请求解耦
func (uc *Usecase) asyncCleanup(bucket, key string) {
    cleanupCtx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
    defer cancel()
    logger.SafeGo("async-backend-cleanup", func() {
        if err := uc.backend.DeleteObject(cleanupCtx, bucket, key); err != nil {
            logger.Warn(cleanupCtx, "async cleanup failed (reconciler will retry)", "error", err)
        }
    })
}

// ✅ 任务调度场景：使用 WithoutCancel 保留 trace
detachedCtx := context.WithoutCancel(ctx)
logger.SafeGo("scheduleNext", func() {
    schedCtx, cancel := context.WithTimeout(detachedCtx, 10*time.Second)
    defer cancel()
    doWork(schedCtx)  // trace 可追踪到发起请求
})
```

### 必须配合超时

Detached context 没有取消信号，必须显式设置超时，否则 goroutine 可能永久挂起。

```go
// ✅ WithTimeout 兜底
ctx, cancel := context.WithTimeout(detachedCtx, 10*time.Second)
defer cancel()

// ❌ 无超时，可能泄漏
doWork(detachedCtx)
```

**例外：LLM 流式响应**。当 context 作为 LLM 流式调用的 `RunCtx` 时，不能设置固定超时——流式响应可能持续任意长时间（复杂任务的 LLM 输出可达数分钟）。此时使用 `context.WithCancel(context.Background())`，通过上层 turnCtx 的取消信号（StopTurn / 锁丢失 / worker 关闭）和 `resumeCancelMu` 机制控制生命周期，而非固定超时。必须在注释中说明为什么不用超时。

```go
// ✅ 例外：LLM 流式响应，固定超时会杀死活跃流
//
// IMPORTANT: No timeout is set here. The resume context becomes the RunCtx
// for the entire subsequent agent execution (including LLM streaming),
// which can last arbitrarily long for complex tasks. A fixed timeout would
// kill active streams mid-output — the user-visible symptom is "Response Timeout"
// even though chunks are still flowing.
//
// Cancellation is still possible: the caller's turnCtx (from Process) is
// the parent of the TurnLoop's execution, and StopTurn / lock-loss / worker
// shutdown all cancel that context.
resumeCtx, cancel := context.WithCancel(context.Background())
mgr.resumeCancel = cancel
```

### 后台 goroutine 必须显式传递身份

`context.WithoutCancel` 保留了 context 中的值（包括 UserID），但如果你在 detached context 之上再创建新 context（如 `context.Background()`），身份信息会丢失。后台 worker 需要访问用户数据时，必须确保 UserID 通过 context 传递。

```go
// ✅ 模式 1：WithoutCancel 保留原有 context 中的 UserID
func (h *Handler) processAsync(ctx context.Context) {
    detachedCtx := context.WithoutCancel(ctx)  // UserID 已在 ctx 的 values 中
    logger.SafeGo("processAsync", func() {
        workCtx, cancel := context.WithTimeout(detachedCtx, 30*time.Second)
        defer cancel()
        doWork(workCtx)  // 可从 workCtx 获取 UserID
    })
}

// ✅ 模式 2：显式注入身份到新 context
func (h *Handler) processAsync(ctx context.Context, userID uuid.UUID) {
    bgCtx := context.WithoutCancel(ctx)
    bgCtx = contextx.WithClientInfo(bgCtx, userID, "")  // 显式注入身份
    logger.SafeGo("processAsync", func() {
        workCtx, cancel := context.WithTimeout(bgCtx, 30*time.Second)
        defer cancel()
        loadAttachments(workCtx)  // worker 可从 workCtx 获取 userID
    })
}

// ❌ 身份丢失：使用 Background() 丢失了 UserID
func (h *Handler) processAsync(ctx context.Context, userID uuid.UUID) {
    bgCtx := context.Background()  // UserID 丢失！
    go func() {
        loadAttachments(bgCtx, userID)  // bgCtx 中没有 UserID
    }()
}
```

---

## 9. defer 阻塞操作保护

### 问题场景

HTTP handler 的 `defer` 中执行资源清理（如 `resp.Body.Close()`、`obj.Close()`）时，清理操作可能因网络问题阻塞。如果客户端已断开，`r.Context()` 已取消，而 `Close()` 等待底层 TCP 连接超时，defer 会一直阻塞——HTTP handler 无法返回响应，直到清理完成。

### 模式：goroutine + channel + timeout

将可能阻塞的 defer 操作放入 goroutine，通过 channel + select 实现超时保护：

```go
// ✅ defer 阻塞保护：goroutine + timeout
defer func() {
    closeCtx, closeCancel := context.WithTimeout(context.Background(), 5*time.Second)
    defer closeCancel()

    closeDone := make(chan struct{})
    logger.SafeGo("object-close", func() {
        // 使用 context.Background() — r.Context() 可能已取消，logger 会丢弃消息
        bgCtx := context.Background()
        if err := obj.Close(); err != nil {
            logger.Warn(bgCtx, "failed to close object",
                zap.String("key", key),
                zap.Error(err))
        }
        close(closeDone)
    })

    select {
    case <-closeDone:
        // 清理完成
    case <-closeCtx.Done():
        // 清理超时 — 记录警告但不阻塞响应
        // 泄漏的 goroutine 受底层 Transport 超时约束（如 ResponseHeaderTimeout: 60s）
        logger.Warn(context.Background(), "object close timed out",
            zap.String("key", key))
    }
}()

// ❌ 直接 defer Close()：可能阻塞 handler 返回
defer obj.Close()  // 如果 Close() 等待 TCP 超时，handler 被阻塞 60s+
```

### 设计决策

| 决策 | 理由 |
|------|------|
| 使用 `context.Background()` 而非 `r.Context()` | 请求 context 可能已取消，logger 依赖它会被跳过 |
| 使用 `logger.SafeGo` 而非裸 `go func()` | 拦截 goroutine panic，带名称标识便于日志追踪 |
| 通过 Transport 超时约束泄漏上界 | 接受最坏情况下的短暂 goroutine 泄漏，换取 handler 不阻塞 |
| timeout 通常 5-10s | 足够覆盖正常 Close()（毫秒级），远小于 Transport 超时 |

### 何时使用

| 场景 | 是否使用 | 原因 |
|------|----------|------|
| HTTP handler 的 `resp.Body.Close()` | 是 | 网络对象关闭可能等待 TCP |
| 数据库连接 `conn.Close()` | 通常不需要 | 本地连接关闭快速 |
| 文件 `f.Close()` | 不需要 | 本地文件操作不会阻塞 |
| 外部 HTTP 客户端 response body | 是 | 与后端连接可能已断开 |

### 与已有模式的关系

- **与 defer 清理的区别**：defer 清理（Section 8）解决"用什么 context"——`context.Background()` 确保清理不受请求取消影响；本节解决"清理操作本身可能阻塞"——通过 goroutine + timeout 保护 handler 不被阻塞
- **与 SafeGo 的关系**：使用 `logger.SafeGo` 启动 goroutine，自动拦截 panic

---

## 10. 反模式

### ❌ goroutine 泄漏

```go
// ❌ 无法退出
go func() {
    for {
        // 永远运行
    }
}()

// ✅ 通过 context 或 channel 退出
go func(ctx context.Context) {
    for {
        select {
        case <-ctx.Done():
            return
        default:
            // 工作
        }
    }
}(ctx)
```

### ❌ nil channel

```go
// ❌ 向 nil channel 发送会永久阻塞
var ch chan int
ch <- 1  // 死锁

// ✅ 初始化 channel
ch := make(chan int)
ch <- 1  // 正常
```

### ❌ 关闭已关闭的 channel

```go
// ❌ 重复关闭会 panic
close(ch)
close(ch)  // panic: close of closed channel

// ✅ 用 sync.Once 保护，或只在发送方关闭
var closeOnce sync.Once
closeOnce.Do(func() {
    close(ch)
})
```

### ❌ 用 time.Sleep 同步

```go
// ❌ 不可靠
go doWork()
time.Sleep(100 * time.Millisecond)  // 怎么知道 100ms 够了？
processResult()

// ✅ 用 channel 或 WaitGroup 同步
done := make(chan struct{})
go func() {
    doWork()
    close(done)
}()
<-done
processResult()
```

---

## 11. 启动恢复 vs 运行时扫描

当系统管理有状态的工作项（如 turn、任务队列）时，需要处理"工作项卡住"的场景（worker 崩溃、网络分区、服务器重启）。恢复机制分两层：启动时恢复和运行时扫描。两者的设计目标不同，不能合并。

### 启动恢复（Startup Recovery）

服务器重启后，立即扫描所有卡住的工作项并恢复。**不使用时间阈值**——因为服务器刚重启，所有卡住的工作项都需要立即处理。

```go
// ✅ 启动恢复：无时间阈值，恢复所有卡住项
func (s *Server) recoverStaleTurns(ctx context.Context) {
    staleTurns, _ := s.svcCtx.TurnRepo.FindStaleTurns(ctx, staleTurnStatuses)
    for _, turn := range staleTurns {
        s.markAndPublishStaleTurn(ctx, turn)  // 立即恢复
    }
    s.cleanupGhostWorksAndLocks(ctx, sessionIDs)  // 清理幽灵工作项和过期锁
}

// ❌ 启动时也用时间阈值
func (s *Server) recoverStaleTurns(ctx context.Context) {
    staleTurns, _ := s.svcCtx.TurnRepo.FindStaleTurns(ctx, staleTurnStatuses)
    for _, turn := range staleTurns {
        if time.Since(turn.StartedAt) < 10*time.Minute {
            continue  // 刚重启就设阈值？卡住的工作项要等多久才能恢复？
        }
    }
}
```

### 运行时扫描（Runtime Scanner）

周期性检查卡住的工作项。**必须使用时间阈值**——避免误判正在正常执行的工作项。配合分布式锁防止多节点并发扫描。

```go
// ✅ 运行时扫描：时间阈值 + 分布式锁 + worker 存活检查
func (s *Server) staleTurnScanner(ctx context.Context) {
    const (
        scannerInterval = 5 * time.Minute
        scannerLockTTL  = 4 * time.Minute  // < interval，防止锁重叠
    )
    ticker := time.NewTicker(scannerInterval)
    for {
        select {
        case <-ctx.Done():
            return
        case <-ticker.C:
            // 分布式锁：只有一个节点扫描
            acquired, _ := s.redis.SetNX(ctx, lockKey, instanceID, scannerLockTTL).Result()
            if !acquired { continue }
            s.periodicRecoverStaleTurns(ctx)
        }
    }
}

func (s *Server) periodicRecoverStaleTurns(ctx context.Context) {
    // 时间阈值：区分"正在执行"和"卡住"
    // running > 10min → 可能卡住，检查 worker 存活
    // pending > 2min  → 可能卡住，直接标记失败
    if age < runningThreshold { continue }
    if s.isWorkerAliveForSession(ctx, sessionID) { continue }  // worker 还活着，别动
}
```

### 两者的关键差异

| 维度 | 启动恢复 | 运行时扫描 |
| --- | --- | --- |
| 时间阈值 | 无（立即恢复所有） | 有（避免误判活跃项） |
| Worker 存活检查 | 不做（无 worker 连接） | 必须做（防止干扰活跃 worker） |
| 恢复方式 | submit（新建 turn） | submit（避免 checkpoint 查找失败） |
| 分布式锁 | 不需要（单实例启动） | 必须（多节点并发防护） |
| 幽灵工作项清理 | 做（requeue ghost works） | 不做（运行时不应有 ghost） |
| Session 锁释放 | 做（检查 worker 存活后释放） | 做（检查 worker 存活后释放） |

### Worker 存活检查防止 Split-Brain

释放 session 锁之前，**必须检查是否有新的 worker 已获取锁**。滚动部署期间，旧服务器崩溃后新服务器可能已启动，新 worker 可能已获取锁。无条件释放会删除新 worker 的锁，导致 split-brain。

```go
// ✅ 检查 worker 存活再释放锁
func (s *Server) cleanupGhostWorksAndLocks(ctx context.Context, sessionIDs map[string]bool) {
    for _, sessionID := range sessionIDs {
        if s.isWorkerAliveForSession(ctx, sessionID) {
            continue  // 新 worker 已获取锁，不要释放
        }
        s.queue.ForceReleaseSession(ctx, sessionID)  // 确认无活跃 worker 才释放
    }
}

// ❌ 不检查直接释放：滚动部署时删除新 worker 的锁
func (s *Server) cleanupGhostWorksAndLocks(ctx context.Context, sessionIDs map[string]bool) {
    for _, sessionID := range sessionIDs {
        s.queue.ForceReleaseSession(ctx, sessionID)  // 可能删除新 worker 的锁！
    }
}
```

**约束**：

- 启动恢复和运行时扫描必须分开实现，不合并为单一函数
- 运行时扫描必须使用分布式锁（`SetNX`），锁 TTL 小于扫描间隔
- 释放 session 锁前必须检查 worker 存活（`TTL > 0`），防止 split-brain
- 恢复方式统一使用 submit（新建 turn），不使用 resume（避免 checkpoint 查找失败）
- 启动恢复后清理幽灵工作项（ghost works），重新入队到对应 session
- 恢复后同步 session 状态（如果无其他 running turn，标记 session 为 idle）
