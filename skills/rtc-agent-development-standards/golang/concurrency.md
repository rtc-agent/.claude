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
        return s.loadUsers(ctx)
    })

    g.Go(func() error {
        return s.loadRooms(ctx)
    })

    g.Go(func() error {
        return s.loadMessages(ctx)
    })

    return g.Wait()  // 任一错误，取消其余
}
```

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
// ✅ 通用 recover 包装
func safeGo(fn func()) {
    go func() {
        defer func() {
            if r := recover(); r != nil {
                log.Printf("goroutine panic recovered: %v\n%s", r, debug.Stack())
            }
        }()
        fn()
    }()
}

// 使用
safeGo(func() {
    processEvent(event)
})

// ✅ errgroup 中的 recover
g.Go(func() error {
    defer func() {
        if r := recover(); r != nil {
            log.Printf("goroutine panic: %v", r)
        }
    }()
    return doWork()
})
```

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

## 7. 反模式

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
