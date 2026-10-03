# Go 后端性能规范

> _性能是系统的呼吸。慢下来，用户就离开了。_

---

## 1. 数据库

### 索引覆盖

所有查询条件字段必须有索引。

```go
// ✅ 查询有索引
db.Where("owner_ref_id = ? AND status = ?", ownerID, "active").Find(&sessions)
// sessions 表有索引：(owner_ref_id, status)

// ❌ 查询无索引（全表扫描）
db.Where("description LIKE ?", "%keyword%").Find(&sessions)
```

### 避免 N+1 查询

```go
// ❌ N+1：N 个会话，N+1 次查询
sessions := findSessions()
for _, s := range sessions {
    s.Messages = findMessages(s.ID)  // 每次循环一次查询
}

// ✅ 批量查询 + 内存关联
sessions := findSessions()
sessionIDs := extractIDs(sessions)
messages := findMessagesBySessionIDs(sessionIDs)  // 一次查询
messagesBySession := groupBy(messages, "session_id")
for _, s := range sessions {
    s.Messages = messagesBySession[s.ID]
}
```

### 批量操作

```go
// ❌ 逐条插入
for _, msg := range messages {
    db.Create(&msg)
}

// ✅ 批量插入
db.Create(&messages)  // 一次 SQL
```

### 连接池配置

```go
db, _ := gorm.Open(postgres.Open(dsn))
sqlDB := db.DB()

sqlDB.SetMaxIdleConns(10)       // 空闲连接数
sqlDB.SetMaxOpenConns(100)      // 最大打开连接数
sqlDB.SetConnMaxLifetime(time.Hour)  // 连接最大存活时间
```

---

## 2. 并发

### goroutine 泄漏检测

所有测试必须通过 `-race` 检测。

```bash
go test -race ./...
```

### goroutine 退出机制

每个 goroutine 必须有明确的退出路径。

```go
// ✅ 通过 context 退出
go func(ctx context.Context) {
    for {
        select {
        case <-ctx.Done():
            return
        case event := <-events:
            process(event)
        }
    }
}(ctx)

// ❌ 无法退出
go func() {
    for event := range events {
        process(event)
    }
}()
```

### channel 缓冲

```go
// ✅ 生产者不被慢消费者阻塞
events := make(chan Event, 100)

// ❌ 无缓冲，生产者阻塞
events := make(chan Event)
```

### context 取消传播

```go
// ✅ 子操作继承父 context 的取消
func (s *Service) ProcessOrder(ctx context.Context, order *Order) error {
    // 所有子操作共享同一个 ctx
    if err := s.validate(ctx, order); err != nil {
        return err
    }
    if err := s.save(ctx, order); err != nil {
        return err
    }
    return nil
}
```

---

## 3. 缓存

### Redis 缓存策略

```go
// ✅ 缓存读取，miss 时查询数据库
func (s *Service) GetUser(ctx context.Context, id string) (*User, error) {
    // 先查缓存
    cached, err := s.cache.Get(ctx, "user:"+id)
    if err == nil {
        var user User
        json.Unmarshal([]byte(cached), &user)
        return &user, nil
    }

    // 缓存 miss，查数据库
    user, err := s.repo.Find(ctx, id)
    if err != nil {
        return nil, err
    }

    // 写入缓存（带 TTL）
    data, _ := json.Marshal(user)
    s.cache.Set(ctx, "user:"+id, string(data), 5*time.Minute)

    return user, nil
}
```

### 缓存失效

- **写时失效**：更新数据时删除对应缓存
- **TTL 兜底**：所有缓存必须设置过期时间
- **缓存穿透**：空值也缓存（短 TTL），防止恶意请求

### Singleflight 防止缓存击穿

高并发场景下，缓存 miss 可能导致大量并发请求同时穿透到后端。使用 `golang.org/x/sync/singleflight` 合并并发请求，只让一个请求真正执行查询。

```go
import "golang.org/x/sync/singleflight"

type Service struct {
    credFlight singleflight.Group
}

// ✅ singleflight 合并并发请求
func (s *Service) GetCredential(ctx context.Context, key string) (*Credential, error) {
    // 先查缓存
    if cached, err := s.cache.Get(key); err == nil {
        return cached, nil
    }

    // DoChan：并发请求合并为一个，其他请求等待结果
    ch := s.credFlight.DoChan(key, func() (interface{}, error) {
        // 使用 Background context：查询独立于任何请求的生命周期
        return s.repo.GetByAccessKeyID(context.Background(), key)
    })

    select {
    case <-ctx.Done():
        return nil, ctx.Err()  // 这个请求放弃了，但查询继续为其他请求服务
    case r := <-ch:
        return r.Val.(*Credential), r.Err
    }
}
```

**要点**：

- 用 `DoChan`（非 `Do`）+ `select` 让每个调用方可独立响应自己的 context 取消
- flight 函数内使用 `context.Background()`，防止某个调用方的取消影响共享查询
- 适用于高并发的缓存 miss 场景（如凭证查询、配置加载）

### Lua 脚本原子性

涉及多个 Redis 操作必须用 Lua 脚本（参见代码质量规范）。

---

## 4. Centrifuge

### 消息压缩

Centrifuge 支持 WebSocket 消息压缩，应在配置中启用。

```go
node := centrifuge.New(centrifuge.Config{
    // 启用压缩
    // 具体配置参考 Centrifuge 文档
})
```

### 连接数管理

```go
// 监控连接数
node.On().Connect(func(client *centrifuge.Client) {
    metrics.IncrGauge("centrifuge_connections")
})

node.On().Disconnect(func(client *centrifuge.Client) {
    metrics.DecrGauge("centrifuge_connections")
})
```

### 频道历史清理

```go
// 配置频道历史消息保留策略
// 避免无限增长
channelConfig := centrifuge.ChannelConfig{
    HistorySize: 100,           // 最多保留 100 条
    HistoryTTL: 24 * time.Hour, // 保留 24 小时
}
```

---

## 5. 内存

### 对象池

频繁创建销毁的对象使用 `sync.Pool`。

```go
var bufPool = sync.Pool{
    New: func() any {
        return new(bytes.Buffer)
    },
}

func process(data []byte) {
    buf := bufPool.Get().(*bytes.Buffer)
    defer bufPool.Put(buf)

    buf.Reset()
    buf.Write(data)
    // 使用 buf...
}
```

### 大 slice 预分配

```go
// ❌ 动态扩容，多次分配
var items []Item
for _, d := range data {
    items = append(items, toItem(d))
}

// ✅ 预分配容量
items := make([]Item, 0, len(data))
for _, d := range data {
    items = append(items, toItem(d))
}
```

### strings.Builder

```go
// ❌ 字符串拼接，每次创建新字符串
var result string
for _, s := range parts {
    result += s  // O(n²)
}

// ✅ strings.Builder，一次分配
var builder strings.Builder
for _, s := range parts {
    builder.WriteString(s)
}
result := builder.String()
```

### 防御性 I/O 限制

从外部来源（文件、网络、用户上传）读取数据时，必须限制读取大小防止 OOM。对于可能超限的数据，实现多级降级管道。

```go
// ✅ 限制读取大小 + 多级降级
const (
    MaxImageReadSize   = 20 * 1024 * 1024  // 20MB 硬上限
    TargetImageSize    = 3.75 * 1024 * 1024 // 目标大小（base64 后 5MB）
    MaxImageWidth      = 2000
    MaxImageHeight     = 2000
)

func LoadImageFromOSS(ctx context.Context, backend Backend, key string) ([]byte, error) {
    // 1. 限制读取大小，防止 OOM
    reader, _, err := backend.GetObject(ctx, bucket, key)
    if err != nil { return nil, err }
    defer reader.Close()
    
    limited := io.LimitReader(reader, MaxImageReadSize+1)
    data, err := io.ReadAll(limited)
    if err != nil { return nil, err }
    if int64(len(data)) > MaxImageReadSize {
        return nil, fmt.Errorf("image exceeds maximum read size (%d MB)", MaxImageReadSize/(1024*1024))
    }
    
    // 2. 快速路径：已满足要求
    if len(data) <= TargetImageSize && fitsWithin(data, MaxImageWidth, MaxImageHeight) {
        return data, nil
    }
    
    // 3. 多级降级：逐步降低质量直到满足目标
    qualities := []int{80, 60, 40, 20}
    for _, q := range qualities {
        compressed := compressImage(data, q)
        if len(compressed) <= TargetImageSize {
            return compressed, nil
        }
    }
    
    // 4. 最终兜底：缩略图
    return createThumbnail(data, 400, 400, 20)
}

// ❌ 不限制读取大小
data, err := io.ReadAll(reader)  // 恶意上传 1GB 文件导致 OOM
```

**约束**：

- 所有从外部读取的操作必须使用 `io.LimitReader` 或等效限制
- 限制值通过常量定义，不硬编码在逻辑中
- 对于需要降级的场景，实现多级管道（质量递减、尺寸递减）
- 每级降级后检查是否满足目标，避免过度处理
- 最终兜底方案必须存在（如缩略图、截断）
- 截断文本时必须保证 UTF-8 字符边界完整（见下方）

**UTF-8 安全截断**：截断文本内容时，不能直接在字节边界切割——中文、emoji 等多字节字符会被截断成乱码。必须向后扫描找到合法的 rune 起始位置。

```go
// ✅ UTF-8 安全截断
func truncateUTF8(content string, maxBytes int, suffix string) string {
    if len(content) <= maxBytes {
        return content
    }
    cutPoint := maxBytes
    for cutPoint > 0 && !utf8.RuneStart(content[cutPoint]) {
        cutPoint--  // 回退到合法的 rune 起始位置
    }
    return content[:cutPoint] + suffix
}

// ❌ 直接字节截断，可能切断多字节字符
content = content[:maxBytes]  // 中文/emoji 可能变成乱码
```

**约束**：所有对文本内容的截断操作（日志截断、消息截断、文件预览截断）必须使用 `utf8.RuneStart` 回退扫描。

**适用场景**：文件上传处理、图片/视频预处理、大文本文件读取、任何处理不可信大小外部数据的场景。

### 并行 I/O 加载

当需要加载多个独立的外部资源（如多个文件附件）时，使用 `sync.WaitGroup` + 按索引结果数组并行加载，而非逐个串行。单个加载失败不阻塞其他加载。

```go
// ✅ 并行加载 + 按索引收集结果
type loadResult struct {
    data []byte
    mime string
    err  error
}

func loadFilesParallel(ctx context.Context, files []File) []loadResult {
    results := make([]loadResult, len(files))
    var wg sync.WaitGroup
    for i, f := range files {
        wg.Add(1)
        go func(idx int, file File) {
            defer wg.Done()
            defer func() {
                if r := recover(); r != nil {
                    results[idx] = loadResult{err: fmt.Errorf("panic: %v", r)}
                }
            }()
            data, mime, err := LoadFromOSS(ctx, file.Key)
            results[idx] = loadResult{data: data, mime: mime, err: err}
        }(i, f)
    }
    wg.Wait()
    return results
}

// ❌ 串行加载：N 个文件 = N 次网络延迟
for _, f := range files {
    data, err := LoadFromOSS(ctx, f.Key)  // 每个等待 100ms → 总共 100*N ms
}
```

**约束**：

- 使用 `WaitGroup`（非 `errgroup`）：单个失败不应取消其余加载（参见 [并发规范 - errgroup vs WaitGroup](./concurrency.md#何时选择-errgroup-vs-waitgroup)）
- 按索引写入 `results` 数组，不对共享 slice 做 `append`，避免竞态
- 每个 goroutine 必须有 `defer recover()` 拦截 panic（参见 [并发规范 - goroutine panic 拦截](./concurrency.md#2-goroutine-panic-拦截)）
- 后台 worker 中执行时，UserID 必须通过 context 传递（参见 [并发规范 - 后台 goroutine 必须传递身份](./concurrency.md#后台-goroutine-必须显式传递身份)）
- 在函数注释中说明并发安全假设（哪些操作是无状态的、哪些共享变量受保护）

**适用场景**：LLM 消息附件加载、批量文件下载、并行 API 调用。判断标准：多个独立的外部 I/O 操作，单个失败不应阻塞其余。

---

## 6. 性能指标

| 指标 | 目标 | 说明 |
|------|------|------|
| **HTTP API P99** | < 200ms | 99 分位响应时间 |
| **RPC 响应** | < 100ms | Centrifuge RPC |
| **数据库查询 P99** | < 50ms | 单条查询 |
| **队列消费延迟** | < 5s | 从入队到开始处理 |
| **goroutine 数** | < 10000 | 稳态值 |

---

## 7. pprof 与基准测试

### pprof 使用

```go
// 启动 pprof
import _ "net/http/pprof"

// 在 main 中
go func() {
    http.ListenAndServe("localhost:6060", nil)
}()
```

```bash
# CPU 分析（30 秒）
go tool pprof http://localhost:6060/debug/pprof/profile?seconds=30

# 内存分析
go tool pprof http://localhost:6060/debug/pprof/heap

# 查看 goroutine
go tool pprof http://localhost:6060/debug/pprof/goroutine
```

### 基准测试

关键路径必须有 bench 覆盖。

```go
func BenchmarkProcessOrder(b *testing.B) {
    order := &Order{ID: "test", Items: []Item{{...}}}
    svc := NewService(...)

    b.ResetTimer()
    for i := 0; i < b.N; i++ {
        svc.ProcessOrder(context.Background(), order)
    }
}

// 运行
go test -bench=BenchmarkProcessOrder -benchmem ./...
```

---

## 8. 新增代码性能检查清单

- [ ] 查询条件字段有索引
- [ ] 无 N+1 查询
- [ ] 批量操作替代逐条
- [ ] goroutine 有退出机制
- [ ] channel 有合理缓冲
- [ ] Redis 缓存有 TTL
- [ ] 字符串拼接用 `strings.Builder`
- [ ] 大 slice 预分配容量
- [ ] 关键路径有基准测试
- [ ] 性能指标在目标范围内

---

## 总结

性能优化不是一次性的冲刺，而是持续的习惯。从第一行代码就遵循这些约束，让系统始终轻盈。

> _"慢是一种 bug。而大多数性能问题，都在设计时就已注定。"_
