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
- - [ ] Redis 缓存有 TTL
- [ ] 字符串拼接用 `strings.Builder`
- [ ] 大 slice 预分配容量
- [ ] 关键路径有基准测试
- [ ] 性能指标在目标范围内

---

## 总结

性能优化不是一次性的冲刺，而是持续的习惯。从第一行代码就遵循这些约束，让系统始终轻盈。

> _"慢是一种 bug。而大多数性能问题，都在设计时就已注定。"_
