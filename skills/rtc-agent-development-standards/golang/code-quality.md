# Go 代码质量规范

> _代码质量不是一次性的审查结果，而是持续的纪律。每一条约束都来自过去踩过的坑。_

---

## 1. 重复代码

### 写新代码前先搜索

在编写新逻辑之前，先搜索项目中是否已有类似实现。

```bash
# 搜索关键字函数或模式
grep -rn "CreateSession" internal/
grep -rn "func.*Update.*Status" internal/
```

### 三次重复才提取

- 第一次出现：直接使用
- 第二次出现：保持原样，观察模式
- 第三次出现：提取到公共位置

### 提取位置

- **业务逻辑**：提取到 `internal/usecase/primitives/`
- **数据转换**：提取到对应的 `model/` 方法
- **基础设施操作**：提取到 `internal/infra/` 或 `pkg/`

---

## 2. 大文件约束

### 新文件不超过 500 行

超过 500 行的文件难以理解和维护。新文件必须控制在 500 行以内。

### 拆分策略

当文件超过 500 行时，按以下方式拆分：

| 拆分方式 | 适用场景 | 示例 |
|---------|---------|------|
| **按功能** | 文件包含多个独立功能 | `data.go` → `load_messages.go` + `create_tools.go` + `create_agent.go` |
| **按类型** | 文件包含多种类型的定义 | `types.go` → `session.go` + `message.go` + `turn.go` |
| **按阶段** | 文件包含不同处理阶段 | `handler.go` → `validation.go` + `execution.go` + `response.go` |

### 拆分信号

- 需要频繁滚动才能理解上下文
- 同一文件中存在不相关的改动
- 函数之间没有明显的逻辑分组

---

## 3. 废弃代码

### 0 引用代码必须删除

当代码没有任何引用时，必须立即删除。**例外**：如果有注释明确说明保留原因，可以暂时保留。

```go
// ✅ 删除无用代码
// 原来有 func OldHelper() { ... }，已删除

// ✅ 保留但说明原因
// Deprecated: 保留用于 v1 API 兼容，将在 v3.0 移除
func LegacyMethod() { ... }

// ❌ 无注释保留
// func OldHelper() { ... }  // 被注释掉但没有说明
```

### `// Deprecated:` 标记规范

标记废弃后，必须在下一个迭代清理。

```go
// Deprecated: 使用 NewMethod 替代。将在 2026-Q4 移除。
func OldMethod() { ... }
```

### PR review 时发现

- 发现 0 引用代码：直接要求删除，不留"以后再说"
- 发现被注释掉的代码：要求删除或恢复
- 发现 `// Deprecated:` 超过一个迭代：要求清理

---

## 4. 模块边界

### 依赖方向

依赖必须单向流动，禁止循环依赖。

```
handler → usecase → repo
   ↓         ↓        ↓
  svc ←─────────────┘
```

- `handler` 可以依赖 `usecase`、`repo`、`model`
- `usecase` 可以依赖 `repo`、`model`
- `repo` 只能依赖 `model`
- `svc` 持有所有依赖的实例，但不被业务层依赖

### 禁止

- `repo` 导入 `usecase` 或 `handler`
- `model` 导入 `repo` 或 `usecase`
- 新增循环依赖

### 新增包的约束

新增包时必须明确：

- 这个包属于哪一层？
- 它依赖哪些包？
- 哪些包会依赖它？

---

## 5. Redis 操作约束

### 两个及以上操作必须使用 Lua 脚本

当一次业务逻辑涉及多个 Redis 操作时，必须封装为 Lua 脚本，保证原子性。

```go
// ❌ 多个操作，非原子
redis.Set(ctx, "key1", "value1")
redis.Set(ctx, "key2", "value2")
redis.Publish(ctx, "channel", "data")

// ✅ Lua 脚本，原子执行
var multiSetScript = redis.NewScript(`
    redis.call("SET", KEYS[1], ARGV[1])
    redis.call("SET", KEYS[2], ARGV[2])
    redis.call("PUBLISH", KEYS[3], ARGV[3])
    return 1
`)
multiSetScript.Run(ctx, redis, []string{"key1", "key2", "channel"}, "value1", "value2", "data")
```

### 锁操作必须用 Lua

新增分布式锁时，必须用 Lua 脚本实现原子 check-and-set，避免 TOCTOU 竞态。

```go
// ✅ Lua 原子锁
var acquireLockScript = redis.NewScript(`
    if redis.call("SET", KEYS[1], ARGV[1], "NX", "EX", ARGV[2]) then
        return 1
    end
    return 0
`)
```

### 队列消费者必须幂等

新增队列消费者时，必须保证幂等性——同一条消息处理多次与处理一次结果相同。

- 使用唯一 ID 去重
- 操作前检查状态
- 失败可重试，不产生副作用

---

## 6. 事务上下文约束

### 所有 repo 方法必须使用 `DBFromContext`

所有 repo 方法必须使用 `repo.DBFromContext(ctx, r.db)` 获取数据库句柄，不直接使用 `r.db`。

```go
// ✅ 正确
func (r *UserRepo) Create(ctx context.Context, user *model.User) error {
    return repo.DBFromContext(ctx, r.db).WithContext(ctx).Create(user).Error
}

// ❌ 错误：直接使用 r.db，无法参与事务
func (r *UserRepo) Create(ctx context.Context, user *model.User) error {
    return r.db.WithContext(ctx).Create(user).Error
}
```

### 原子写入必须通过 `RunAndPublish`

需要原子写入的场景必须通过 `UpdatePublisher.RunAndPublish` 编排，不手动 `db.Begin()`。

```go
// ✅ 通过 RunAndPublish
updates, err := h.deps.UpdatePublisher.RunAndPublish(ctx, func(txCtx context.Context) ([]UpdatePublishItem, error) {
    primitives.CreateSession(txCtx, deps, session)
    primitives.CreateMessage(txCtx, deps, message)
    return primitives.BuildUpdates(...)
})

// ❌ 手动管理事务
tx := db.Begin()
defer func() {
    if err != nil {
        tx.Rollback()
    }
}()
// ...
tx.Commit()
```

### 只读操作在事务外执行

只读操作（如权限检查、数据查询）在 `RunAndPublish` 闭包外执行，使用普通 `ctx`，保持事务短小。

```go
// ✅ 只读在事务外
session, err := h.deps.SessionRepo.GetByID(ctx, sessionID)  // 普通 ctx
if err != nil {
    return err
}

// 只有写入在事务内
updates, err := h.deps.UpdatePublisher.RunAndPublish(ctx, func(txCtx context.Context) {
    primitives.UpdateStatus(txCtx, deps, sessionID, "active")
    // ...
})
```

### primitives 函数透传 `txCtx`

`usecase/primitives/` 中的函数接收 `txCtx`，透传给 repo，不在 primitives 内开启新事务。

```go
// ✅ 透传 txCtx
func CreateMessage(txCtx context.Context, deps *Dependencies, msg *model.Message) error {
    return deps.MessageRepo.Create(txCtx, msg)  // txCtx 透传
}

// ❌ 在 primitives 内开新事务
func CreateMessage(ctx context.Context, deps *Dependencies, msg *model.Message) error {
    tx := deps.DB.Begin()  // 不允许
    // ...
}
```

### 新增 repo 方法必须遵循同一模式

新增 repo 方法时，必须使用 `DBFromContext`，不引入新的事务传递方式。

---

## 7. 分布式约束

### 跨服务调用必须有超时和重试

```go
// ✅ 超时 + 重试
client := &http.Client{Timeout: 5 * time.Second}
err := retry.Do(func() error {
    return client.Call(ctx, req)
}, retry.WithMaxRetries(3), retry.WithBackoff(100*time.Millisecond))

// ❌ 无超时
client := &http.Client{}  // 默认无超时
resp, err := client.Get(url)
```

### 分布式状态必须有超时兜底

所有分布式锁、队列任务、临时状态必须设置 TTL，防止资源泄漏。

```go
// ✅ 设置 TTL
redis.Set(ctx, lockKey, workerID, 30*time.Second)

// ❌ 无 TTL
redis.Set(ctx, lockKey, workerID)  // 永不过期
```

---

## 8. 测试约束

### 必须测试的场景

| 新增内容 | 必须测试 |
|---------|---------|
| `usecase` 方法 | 单元测试：正常路径 + 错误路径 |
| `repo` 方法 | 集成测试：CRUD + 事务行为 |
| RPC 方法 | happy path + error path + 权限校验 |
| 后台任务 | 正常执行 + 取消 + panic 恢复 |

### 测试命名

```go
func TestCreateSession_ValidInput_CreatesSuccessfully(t *testing.T)
func TestCreateSession_DuplicateID_ReturnsError(t *testing.T)
func TestSendMessage_NotOwner_ReturnsPermissionDenied(t *testing.T)
```

---

## 9. PR 自检清单

提交 PR 前，必须逐项确认：

- [ ] 代码符合 gofmt 格式
- [ ] 无 0 引用代码（或注释说明保留原因）
- [ ] 新文件不超过 500 行
- [ ] 依赖方向正确，无循环依赖
- [ ] Redis 多操作使用 Lua 脚本
- [ ] repo 方法使用 `DBFromContext`
- [ ] 原子写入通过 `RunAndPublish`
- [ ] 新增功能有日志、追踪、指标
- [ ] 新增功能有测试
- [ ] 不打印敏感信息
- [ ] 错误处理正确（不吞错误，有上下文）

---

## 总结

代码质量是团队共识的体现。遵循这些约束，让代码库保持健康、可维护、可演进。

> _"今天的纪律是明天的自由。"_
