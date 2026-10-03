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

### 资源标识符必须集中构造

涉及安全校验的资源标识符（如 OSS key、文件路径、缓存 key）必须通过集中定义的构造函生成，不在业务代码中手动拼接。构造函与校验函数放在同一文件中，确保构造和校验逻辑始终同步。

```go
// ✅ 集中构造 + 校验（pkg/rtc-oss3/key_validation.go）
// BuildUserPrefix returns the user prefix for S3 keys.
func BuildUserPrefix(userID string) string {
    return fmt.Sprintf("user-%s/", userID)
}

// BuildFileKey constructs a full S3 key from user ID and file ID.
func BuildFileKey(userID, fileID string) string {
    return BuildUserPrefix(userID) + fileID
}

// ValidateKey checks that the key matches the expected format.
func ValidateKey(key, userID string) *S3Error { ... }

// 使用方：
fullKey := rtcoss3.BuildFileKey(userID, fileID)  // 构造
s3Err := rtcoss3.ValidateKey(key, userID)         // 校验

// ❌ 手动拼接，格式变更时遗漏
fullKey := fmt.Sprintf("user-%s/%s", userID, fileID)

// ❌ 校验和构造分散在不同包，格式变更容易遗漏一端
```

**判断标准**：当标识符的格式与安全校验耦合时（如 key 必须包含 userID 前缀才能通过权限检查），必须集中构造。纯内部使用的缓存 key 等可以不强制。

**规则**：

- 构造数和校验函数放在同一文件中（如 `key_validation.go`）
- 废弃的拼接方式用 `// Deprecated:` 标记，指向新的构造函数
- 新增资源类型时，同步提供 `Build*` + `Validate*` 函数对

---

## 2. 大文件约束

### 新文件不超过 500 行

超过 500 行的文件难以理解和维护。新文件必须控制在 500 行以内。测试文件放宽到 800 行——表驱动测试和多个场景天然产生更多代码，但仍需在超出时考虑拆分。

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

### 大功能的文件组织

当一个功能涉及 10+ 个文件时，使用统一的命名前缀将所有文件关联在一起，并按操作/职责进一步拆分。辅助文件（测试、指标、中间件）使用描述性后缀。

```text
# ✅ 大功能：统一前缀 + 按操作/职责拆分
internal/handler/http/
├── oss3.go                    # 主 handler + 路由分发 + 错误映射
├── oss3_put.go                # PUT 操作
├── oss3_get.go                # GET 操作
├── oss3_delete.go             # DELETE 操作
├── oss3_copy.go               # COPY 操作
├── oss3_list.go               # LIST 操作
├── oss3_multipart.go          # 分片上传通用逻辑
├── oss3_mp_upload.go          # 分片上传具体操作
├── oss3_mp_complete.go        # 分片完成
├── oss3_mp_abort_list.go      # 分片中止/列表
├── oss3_middleware.go          # 中间件（限流、业务限制）
├── oss3_access_log.go         # 访问日志中间件
├── oss3_metrics.go            # 指标中间件 + 指标注册
├── oss3_sigv4.go              # SigV4 认证中间件
├── oss3_cors.go               # CORS 中间件
├── oss3_xml.go                # XML 序列化辅助
├── oss3_test.go               # 主测试
├── oss3_edge_test.go          # 边界测试
├── oss3_integration_test.go   # 集成测试
├── oss3_test_helpers_test.go  # 测试辅助函数
└── oss3_utils_test.go         # 测试工具函数

# ❌ 无规律命名，难以找到相关文件
internal/handler/http/
├── storage.go
├── upload.go
├── download.go
├── s3handler.go
├── middleware_auth.go
├── middleware_log.go
└── s3_tests.go
```

**约束**：

- 前缀统一（如 `oss3_`），不混用不同前缀
- 主文件（`oss3.go`）负责路由分发和共享逻辑，不超过 300 行
- 每个操作文件（`oss3_put.go`）只包含该操作的 handler 和直接辅助函数
- 测试辅助函数集中在 `_test_helpers_test.go`，不分散在各测试文件

### 补偿流程保持线性

实现补偿流程（步骤 A → B → C，失败时回滚已完成步骤）的函数应保持线性，不拆分为多个小函数。补偿步骤之间存在紧密的状态依赖和错误处理耦合，拆分到不同函数会隐藏数据流和回滚逻辑，增加理解成本。必须在函数顶部用注释说明"为什么是线性的"。

```go
// ✅ 补偿流程保持线性，在函数顶部注释说明原因
//
// handlePutObject handles PUT /{bucket}/{key} — upload object.
//
// The function is intentionally linear (not deeply decomposed) because it
// implements a compensation flow: quota reservation -> backend upload ->
// quota commit -> DB record. Splitting it would hide the data flow.
func (h *OSS3Handler) handlePutObject(...) {
    // 1. 配额预留
    // 2. 上传到后端
    // 3. 配额提交（失败时 defer 释放预留）
    // 4. 写入数据库记录（失败时记录孤儿资源）
}

// ❌ 过度拆分补偿流程，隐藏状态依赖
func (h *OSS3Handler) handlePutObject(...) {
    h.reserveQuota(...)     // 状态：quotaRequestID
    h.uploadToBackend(...)  // 状态：uploadSucceeded
    h.commitQuota(...)      // 状态：需要知道前两步的结果
    h.createRecord(...)     // 状态：需要知道前三步的结果
}
```

**判断标准**：函数中是否有多个步骤共享补偿状态（如 `quotaRequestID`、`uploadSucceeded`），且后续步骤的 defer 依赖前面步骤的结果？如果是，保持线性。如果函数包含多个独立的功能（不同 HTTP 方法、不同业务操作），则应该拆分。

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

### 幂等操作使用 SETNX 标记

需要重试的外部操作（如配额提交、文件记录创建）必须使用幂等标记，防止重试导致双重扣费或重复写入。

```go
// ✅ 幂等提交：SETNX 标记防止重复执行
func (uc *Usecase) CommitQuota(ctx context.Context, userID, requestID string, amount int64) error {
    // SETNX：只有首次执行成功，后续重试直接返回成功
    marker := "quota:commit:" + requestID
    ok, err := uc.redis.SetNX(ctx, marker, "1", 24*time.Hour).Result()
    if err != nil {
        return fmt.Errorf("set quota marker: %w", err)
    }
    if !ok {
        return nil  // 已提交，幂等跳过
    }
    // 执行实际扣减
    return uc.adjustQuota(ctx, userID, amount)
}

// ❌ 无幂等标记，重试会重复扣减
func (uc *Usecase) CommitQuota(ctx context.Context, userID string, amount int64) error {
    return uc.adjustQuota(ctx, userID, amount)  // 每次调用都扣减
}
```

**适用场景**：配额提交、跨系统资源转移、任何"执行一次但可能需要重试"的操作。

### Lua 脚本返回值约定

Lua 脚本有多种执行结果时，使用整数编码返回值，Go 侧通过 `switch` 穷举所有可能的返回值。每个返回值必须有明确的语义，不允许忽略未知返回值。

```go
// ✅ 整数返回值 + switch 穷举（internal/usecase/oss3_quota.go）
result, err := script.Run(ctx, uc.redis, keys, args...).Int()
if err != nil {
    return fmt.Errorf("quota commit: %w", err)
}
switch result {
case 1:
    return nil  // 成功：pending 已提交到 quota 计数器
case 0:
    // pending key 过期（TTL），清理标记
    _ = uc.redis.Del(ctx, markerKey).Err()
    logger.Warn(ctx, "quota commit skipped: pending reservation expired")
    return fmt.Errorf("quota commit: pending expired: %w", rtcoss3.ErrQuotaExceeded)
case -1:
    // 调用方 bug（如 amount 超出 pending 预留量）
    _ = uc.redis.Del(ctx, markerKey).Err()
    logger.Error(ctx, "quota commit failed: amount exceeds pending (caller bug)")
    return fmt.Errorf("quota commit: amount %d exceeds pending", amount)
default:
    // 防御性编程：未知返回值不应出现，必须记录并报错
    logger.Error(ctx, "quota commit returned unexpected value",
        zap.Int("result", result))
    return fmt.Errorf("quota commit: unexpected return value %d", result)
}

// ❌ 忽略返回值语义或用 if/else 而非 switch
if result == 1 {
    return nil
}
return fmt.Errorf("failed")  // 丢失了 result=0 和 result=-1 的语义区分
```

**约束**：

- Lua 脚本的返回值语义必须在脚本和 Go 调用方之间保持一致，修改脚本时必须同步修改 Go 侧的 switch
- `default` 分支必须记录 Error 日志，包含未知返回值，便于发现脚本变更未同步的问题
- 返回值含义通过注释说明（在 switch 的每个 case 中标注语义）
- 语义不同的返回值使用不同的整数（如 1=成功、0=无操作/过期、-1=参数错误），不使用"0=失败、非0=成功"这种模糊约定

**判断标准**：当 Lua 脚本有三种及以上执行结果时使用此约定。如果脚本只有两种结果（成功/失败），直接用 `result == 1` 判断即可。

---

## 6. 跨层校验原语

### 放在 `usecase/primitives/` 的校验函数

当校验需要跨多个层（handler 输入 → 基础设施构造 → repo 查询）时，将校验逻辑封装为 `usecase/primitives/` 中的独立函数，而非散落在 handler 或 usecase 方法中。

```go
// ✅ 跨层校验原语（internal/usecase/primitives/validate_files.go）
//
// 三步编排：
// 1. 格式校验：正则验证 file ID 格式（{md5}.{ext}）
// 2. 构造标识符：通过集中构造函数 rtcoss3.BuildFileKey(userID, fileID) 构造完整 key
// 3. 批量存在性检查：调用 repo.KeysExist() 验证所有文件属于当前用户
func ValidateFilesExist(ctx context.Context, fileRepo repo.FileRepo,
    files []protocol.FileAttachment, userID uuid.UUID) error {

    for _, f := range files {
        if !fileIDPattern.MatchString(f.Fileid) {
            return fmt.Errorf("invalid file ID format: %q", f.Fileid)
        }
        fullKey := rtcoss3.BuildFileKey(userID.String(), f.Fileid)  // 集中构造
        keys = append(keys, fullKey)
    }
    // 批量查询存在性
    return fileRepo.ValidateKeysExist(ctx, keys)
}

// ❌ 散落在 handler 中：格式校验、key 构造、DB 查询混在一起
func (h *Handler) sendMessage(...) {
    for _, f := range files {
        if !isValidFormat(f.Fileid) { ... }           // 内联格式校验
        key := fmt.Sprintf("user-%s/%s", uid, f.Fileid)  // 手动拼接 key
        exists, _ := h.repo.KeyExists(ctx, key)           // 逐条查询
    }
}
```

**判断标准**：当校验同时涉及输入格式验证、资源标识符构造（使用集中构造函数）、以及持久化层存在性检查时，提取为 primitives 函数。单纯的形式校验（如参数非空）留在 handler 层；单纯的 DB 查询留在 repo 层。

**约束**：

- 校验函数使用 `rtcoss3.BuildFileKey()` 等集中构造函数，不在校验逻辑中手动拼接标识符（确保校验规则与构造规则始终同步）
- 校验函数接收接口参数（如 `repo.FileRepo`），不依赖具体实现，便于测试
- 校验函数集中在一个文件中（如 `validate_files.go`），不散落在多个文件
- 新增资源类型（如图片、视频引用）的校验遵循同一模式：格式 → 构造 → 存在性

---

## 7. 事务上下文约束

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

### 事务提交后的补偿

当事务内写入成功后，如果外部操作（如更新 OSS 状态）失败，需要提供补偿回调清理已提交的外部资源。可重试的数据库错误（如死锁）触发重试时，必须先补偿回滚外部资源。详见[错误处理规范 - 事务补偿回调](./error-handling.md#5-事务补偿回调)。

---

## 8. 分布式约束

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

## 9. 重试与退避

### 统一使用 `cenkalti/backoff/v5`

需要重试的操作必须使用项目已引入的 `cenkalti/backoff/v5`，不自写 `for` 循环 + `time.Sleep`。

```go
import "github.com/cenkalti/backoff/v5"

// ✅ 使用 backoff v5 函数式选项 API
bo := backoff.NewExponentialBackOff()
_, err := backoff.Retry(ctx, func() (Result, error) {
    result, err := doOperation(ctx)
    if err != nil && !isRetryable(err) {
        return Result{}, backoff.Permanent(err)  // 不可重试，立即停止
    }
    return result, err
},
    backoff.WithBackOff(bo),
    backoff.WithMaxTries(3),
)

// ❌ 手写重试循环
for i := 0; i < 3; i++ {
    if err := fn(); err != nil {
        time.Sleep(time.Second)  // 硬编码间隔，无退避
    }
}
```

### backoff v5 关键 API

| API | 用途 |
| --- | --- |
| `backoff.Retry(ctx, fn, ...opts)` | 带 context 的重试入口（v5 签名，ctx 为第一参数） |
| `backoff.Permanent(err)` | 包装不可重试的错误，立即终止重试 |
| `backoff.WithBackOff(bo)` | 指定退避策略（ExponentialBackOff 等） |
| `backoff.WithMaxTries(n)` | 最大重试次数 |
| `backoff.WithMaxElapsedTime(d)` | 最大总耗时 |

**注意**：v5 签名为 `Retry(ctx, fn, ...opts)`，ctx 作为第一个参数传入。不要使用已废弃的 `backoff.WithContext(b, ctx)` 包装方式。

### 退避参数默认约定

| 操作类型 | InitialInterval | MaxInterval | MaxTries | 理由 |
| --- | --- | --- | --- | --- |
| 后端 I/O（OSS put） | 50ms | 200ms | 3 | 快速重试，减少延迟影响 |
| 后端 I/O（OSS copy/delete） | 100ms | 2s | 3 | 操作较慢，允许更长间隔 |
| RPC resume / 队列恢复 | 20ms | 200ms | 10 | 快速探测 + 多次重试 |
| 数据库可重试错误 | 50ms | 5s | 10 | 死锁恢复需要较长间隔 |

新增重试操作时，默认使用 `InitialInterval(50ms)` + `MaxInterval(200ms)` + `MaxTries(3)`。仅在操作有明确理由时才偏离默认值，并附注释说明原因。

```go
// ✅ 后端 I/O 重试：快速退避，3 次足够
bo := backoff.NewExponentialBackOff()
bo.InitialInterval = 50 * time.Millisecond
bo.MaxInterval = 200 * time.Millisecond

_, err := backoff.Retry(ctx, func() (struct{}, error) {
    if err := operation(ctx); err != nil {
        if !isTransientError(err) {
            return struct{}{}, backoff.Permanent(err)
        }
        return struct{}{}, err
    }
    return struct{}{}, nil
},
    backoff.WithBackOff(bo),
    backoff.WithMaxTries(3),
)

// ✅ 数据库死锁重试：需要更长间隔和更多次数，附注释
// InitialInterval=50ms, MaxInterval=5s: deadlock recovery needs longer intervals
// MaxTries(10): deadlock under high concurrency may need more retries
bo := backoff.NewExponentialBackOff()
bo.InitialInterval = 50 * time.Millisecond
bo.MaxInterval = 5 * time.Second

// ❌ 无理由地偏离默认值
bo.InitialInterval = 10 * time.Second  // 首次重试等 10 秒？
backoff.WithMaxTries(100)              // 为什么 100 次？
```

### 可重试操作的条件

| 条件 | 说明 |
|------|------|
| 操作幂等 | 重试多次与一次结果相同（写入、状态转换） |
| 错误可恢复 | 网络超时、限流、临时不可用；非逻辑错误 |
| 有超时兜底 | 通过 `backoff.Retry(ctx, fn, ...)` 的 ctx 绑定超时，防止无限重试 |
| 有最大次数 | 配置 `backoff.WithMaxTries(n)`，防止无限退避 |
| 不可重试错误用 Permanent | 逻辑错误、权限不足等，用 `backoff.Permanent(err)` 立即停止 |

### 配置化退避参数

重试参数通过配置注入，不硬编码在业务逻辑中。

```go
// ✅ 配置化
type RetryConfig struct {
    MaxRetries    int           `mapstructure:"max_retries"`
    BaseDelay     time.Duration `mapstructure:"base_delay"`
    BackoffFactor float64       `mapstructure:"backoff_factor"`
}

// 使用
b := backoff.NewExponentialBackOff()
b.InitialInterval = cfg.BaseDelay
b.Multiplier = cfg.BackoffFactor
b.MaxElapsedTime = 30 * time.Second
```

### 退避期间响应 context 取消

退避等待必须响应 context 取消，否则进程关闭时会卡住。

```go
// ✅ v5 的 Retry 直接接受 ctx，自动响应取消
_, err := backoff.Retry(ctx, fn, backoff.WithBackOff(bo), backoff.WithMaxTries(3))

// ❌ 不响应 context 的 Sleep
for i := 0; i < 3; i++ {
    time.Sleep(time.Second)  // 无法被取消
}
```

### 重试耗尽后记录孤儿资源

当重试次数耗尽但外部资源已创建时（如后端文件已上传但数据库记录失败），必须记录孤儿资源指标，供后台对账任务清理。不要只打日志了事——日志会丢失，指标不会。

```go
// ✅ 重试耗尽后记录孤儿
_, err := backoff.Retry(ctx, func() (struct{}, error) {
    return struct{}{}, h.oss3UC.CreateFileRecord(ctx, file)
},
    backoff.WithBackOff(bo),
    backoff.WithMaxTries(3),
)
if err != nil {
    logger.Error(ctx, "failed to create file record after all retries; orphaned object",
        zap.String("bucket", bucket),
        zap.String("key", key),
        zap.Error(err),
    )
    RecordOrphanedRecord("upload_failed")  // 指标记录，供对账使用
}

// ❌ 只打日志，没有指标
if err != nil {
    logger.Error(ctx, "create file record failed", zap.Error(err))
    // 孤儿资源无人清理
}
```

**约束**：

- 新增涉及"外部资源创建 + 数据库记录"的操作时，必须在重试失败后调用 `RecordOrphaned*` 指标

- 孤儿指标必须包含足够的上下文（如 `bucket` + `key`）供对账任务定位资源

- 后台对账任务（reconciler）必须定期扫描孤儿指标并清理

---

## 10. 测试约束

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

## 11. LLM 消息防御性规范化

当构建发送给 LLM API 的消息序列时，数据库中的消息可能不符合 API 的结构约束（如连续同角色消息、孤立的 tool call/result 对）。必须通过防御性规范化层确保消息序列的合法性。

### 规范化管道

```go
// ✅ 四步规范化管道（顺序重要）
func normalizeMessagesForLLM(messages []*Message) ([]*Message, error) {
    // Step 1: 提取 system 消息到前导位置
    messages = extractSystemMessages(messages)
    
    // Step 2: 修复 tool call/result 配对（删除孤立项）
    // 必须在 merge 之前运行，因为删除可能创建新的连续同角色对
    messages = repairToolPairing(messages)
    
    // Step 3: 合并连续同角色 user 消息
    // 合并 Content、ReasoningContent、MultiContent
    messages = mergeConsecutiveSameRole(messages)
    
    // Step 4: 验证最终序列的结构正确性
    if err := validateMessageSequence(messages); err != nil {
        return nil, fmt.Errorf("normalize: %w", err)
    }
    
    return messages, nil
}
```

### Copy-on-First-Merge 模式

合并连续消息时，只在第一次合并时浅拷贝，避免修改调用方的原始对象。这是一个容易遗漏的防御性细节——如果直接修改 `result[len(result)-1]` 的字段，可能意外改变调用方持有的同一指针所指向的对象。

```go
// ✅ Copy-on-first-merge：避免污染原始数据
mergedIdx := -1  // 追踪 result 中最后一个已拷贝的索引；-1 表示未拷贝

for i := 0; i < len(messages); i++ {
    curr := messages[i]
    if len(result) > 0 {
        prev := result[len(result)-1]
        if curr.Role == prev.Role && curr.Role == RoleUser {
            // 第一次合并时拷贝：prev 仍是调用方的原始对象
            if mergedIdx != len(result)-1 {
                copied := *prev  // 浅拷贝
                result[len(result)-1] = &copied
                prev = &copied
                mergedIdx = len(result)-1
            }
            // 后续合并直接追加：prev 已是独立副本
            prev.Content = joinContent(prev.Content, curr.Content)
            prev.ReasoningContent = joinContent(prev.ReasoningContent, curr.ReasoningContent)
            // MultiContent 合并必须用 make + append，不能直接 append(prev.MultiContent, ...)
            // 因为浅拷贝只复制了 slice header，底层 array 仍与原始对象共享
            if len(curr.MultiContent) > 0 {
                merged := make([]schema.MessageInputPart, 0, len(prev.MultiContent)+len(curr.MultiContent))
                merged = append(merged, prev.MultiContent...)
                merged = append(merged, curr.MultiContent...)
                prev.MultiContent = merged
            }
            continue
        }
    }
    mergedIdx = -1  // 新组开始，重置追踪
    result = append(result, curr)
}

// ❌ 直接修改，可能污染调用方的原始对象
prev.Content = joinContent(prev.Content, curr.Content)  // prev 可能是调用方的指针
// ❌ 直接 append MultiContent，共享底层 array
prev.MultiContent = append(prev.MultiContent, curr.MultiContent...)  // 原始对象也被修改
```

**约束**：

- 合并操作中第一个被修改的对象必须浅拷贝，后续同组合并可直接修改
- Slice 类型字段（如 `MultiContent`、`Content`）合并时必须 `make` + `append`，不能直接 `append` 到浅拷贝的 slice header
- 用 `mergedIdx` 或等效机制追踪"是否已拷贝"，避免重复拷贝
- 合并所有携带内容的字段：`Content`（文本）、`ReasoningContent`（推理）、`MultiContent`（图片等多媒体）——遗漏任何字段会导致内容静默丢失

### 防御性清洗

对于 LLM 可能泄漏的内容（如 thinking tags 混入 text content），提供清洗函数：

```go
// ✅ 防御性清洗：防止模型泄漏 thinking tags
func sanitizeThinkTagLeak(text string) string {
    // 移除意外出现在 text content 中的 <think>...</think> 标签
    return regexp.MustCompile(`(?s)<think>.*?</think>`).ReplaceAllString(text, "")
}
```

### 规则

- **规范化作为最后一道防线**：在所有内容注入（attachments、commands、scenarios）完成后运行
- **repair 在 merge 之前**：删除孤立项可能创建新的连续同角色对
- **异常要记录日志**：规范化过程中的异常是观测窗口，用 `Info` 级别记录
- **验证只检查基本规则**：repair 已处理大部分结构问题，验证只捕获 repair 无法修复的情况

### 两层规范化架构

消息规范化分两层运行，各司其职：

**第一层（GenInput 阶段）：** `normalizeMessagesForLLM` 在消息从数据库加载后、注入 turn-agent 之前运行。处理结构性问题：提取 system 消息、修复 tool pairing、合并连续 user 消息（通过 "\n" 连接文本 + 追加 MultiContent）。

**第二层（ChatModel 调用前）：** `MergeAssistantMiddleware` 在 ReAct 循环中每次调用 LLM 之前运行。处理 assistant 消息的合并（使用 `AssistantGenMultiContent` 内容块而非 "\n" 连接，确保 cache hit），并执行 tool call 排序（按 name+arguments 确定性排序，防止并行 tool call 的顺序变化导致缓存失效）和 tool result 去重。

```text
数据库消息
    ↓
第一层：normalizeMessagesForLLM（结构性修复）
  - extract system → 前导
  - repair tool pairing
  - merge consecutive user（"\n" + MultiContent append）
  - validate sequence
    ↓
turn-agent.Message 序列
    ↓
第二层：MergeAssistantMiddleware（每次 ChatModel 调用前）
  - merge adjacent assistant（MultiContent 块合并）
  - sort parallel tool calls（确定性排序，优化缓存命中）
  - deduplicate tool results
    ↓
schema.Message 序列 → LLM API
```

**为什么分两层？**

- User 消息用 "\n" 连接（简单文本拼接），assistant 消息用 MultiContent 块合并（保留结构化内容，提高缓存命中率）
- 第一层在消息注入管道完成后运行一次；第二层在每轮 LLM 调用前都运行，处理 ReAct 循环中动态生成的 assistant 消息
- Tool call 排序放在第二层，因为排序后的序列直接影响 Anthropic API 的 prompt cache 命中率

**约束**：

- 新增消息规范化逻辑时，判断它属于哪一层：结构性修复（第一层）还是调用前优化（第二层）
- 第二层中间件必须注册为 ChatModel 的 middleware，确保每轮调用都执行
- Tool call 排序必须使用确定性 key（如 `name + json(arguments)`），不能用 map 遍历顺序

### 适用场景

- 所有发送给 LLM API 的消息序列
- 从数据库加载消息后、发送前的最终处理
- 任何可能产生非标准消息序列的管道

---

## 12. LLM 内容预处理管道

当将用户生成的内容（文件、图片、文本）发送给 LLM API 时，内容必须经过预处理管道以满足 API 的硬约束。用户不可控的输入（上传的文件）不能直接传递给外部 API——必须有归一化层将输入约束到 API 的合法范围内。

### 多级别降级策略

对于有大小限制的内容（如图片），采用逐级降级的策略，优先保留质量，逐步妥协：

```go
// ✅ 多级别降级管道（internal/agent/file_loader.go）
//
// 约束：Anthropic API 限制 base64 编码后图片 ≤ 5MB
// 策略：快速路径 → 尺寸缩放 → 多级质量降级 → 兜底压缩

func LoadImageFromOSS(ctx context.Context, backend Backend, bucket, key string) ([]byte, string, error) {
    // 1. 读取原始数据（限制最大读取量防 OOM）
    // 2. 通过 magic bytes 检测 MIME 类型（不信任客户端 Content-Type）
    // 3. EXIF 自动旋转（处理手机拍照方向问题）
    // 4. 快速路径：已满足约束则直接返回（零质量损失）
    if rawSize <= targetRawSize && fitsIn(maxWidth, maxHeight) {
        return data, mime, nil
    }
    // 5. 缩放到约束范围内（2000x2000 bounding box）
    resized := imaging.Resize(img, maxWidth, maxHeight, imaging.Lanczos)
    // 6. 多级质量降级：PNG → JPEG [80, 60, 40, 20]
    for _, quality := range []int{80, 60, 40, 20} {
        buf := encodeJPEG(resized, quality)
        if len(buf) <= targetSize {
            return buf, "image/jpeg", nil
        }
    }
    // 7. 兜底：400x400 JPEG quality=20（极端情况）
    // 8. 安全网：最终验证 base64 大小 ≤ API 限制
}
```

### 文本文件截断

文本文件也有大小限制，超出时截断并附加提示标记：

```go
// ✅ 截断 + 提示标记
const MaxTextFileSize = 256 * 1024  // 256KB
const TextFileTruncation = "... [文件已截断，超出 256KB 限制]"

func LoadTextFromOSS(ctx context.Context, backend Backend, bucket, key string) (string, error) {
    // 限制读取量 + 截断提示
    limited := io.LimitReader(reader, MaxTextFileSize+1)
    data, _ := io.ReadAll(limited)
    if len(data) > MaxTextFileSize {
        return string(data[:MaxTextFileSize]) + TextFileTruncation, nil
    }
    return string(data), nil
}
```

### 规则

- **约束常量集中定义**：API 限制（如 `APIImageMaxBase64Size`、`MaxTextFileSize`）在文件顶部集中定义，标注来源（如 "aligned with reference project apiLimits.ts"）
- **快速路径优先**：先检查是否已满足约束，避免不必要的处理
- **降级要有兜底**：多级降级后仍不满足时，使用最激进的压缩（400x400 JPEG quality=20），而非报错
- **安全网验证**：最终返回前再次验证约束，防止边界情况穿透
- **MIME 通过 magic bytes 检测**：不信任客户端提供的 Content-Type，使用 `mimetype.Detect()` 从文件头判断

### 适用场景

- 图片/音频/视频预处理以满足 LLM API 大小限制
- 文本文件截断以适应 context window 限制
- 任何"外部不可控输入需要适配外部 API 约束"的场景

---

## 13. 分页循环终止约束

### 循环必须有硬性上限

分页遍历外部 API（S3 ListObjects、数据库游标、HTTP 分页）的循环，必须有独立于数据返回条件的硬性终止约束。仅依赖"空响应则退出"不够——API 行为变化、分页 token 异常、或测试环境 mock 不完整都可能导致无限循环。

```go
// ✅ 双重终止：空响应 + 最大页数
const maxPages = 1000  // 安全网，防止 API 异常导致无限循环
for page := 0; page < maxPages; page++ {
    resp, err := api.ListObjects(ctx, params)
    if err != nil {
        return fmt.Errorf("list page %d: %w", page, err)
    }
    process(resp.Items)
    if !resp.IsTruncated {
        break  // 正常终止
    }
    params.ContinuationToken = resp.NextToken
}

// ❌ 仅依赖 API 的终止信号
for {
    resp, _ := api.ListObjects(ctx, params)
    process(resp.Items)
    if !resp.IsTruncated {
        break  // 如果 IsTruncated 永远为 true（API bug / mock 不完整），循环永不退出
    }
}
```

### 测试环境中的分页终止

集成测试中 mock 的分页响应必须设置终止条件。测试 helper 中的 mock 如果总是返回 `IsTruncated: true`，会导致分页循环无限执行。

```go
// ✅ 测试 mock 设置终止条件
callCount := 0
mockBackend.On("ListObjects", mock.Anything, mock.Anything).Return(func(...) ListResult {
    callCount++
    return ListResult{
        Items:        testItems,
        IsTruncated:  callCount < 3,  // 第 3 次返回 false，终止循环
        NextToken:    "next",
    }
}, nil)

// ❌ 测试 mock 永远返回 IsTruncated: true
mockBackend.On("ListObjects", ...).Return(ListResult{IsTruncated: true}, nil)
// 分页循环永远不会退出 → 测试挂起
```

**约束**：

- 所有分页遍历循环必须有最大迭代次数（常量定义，附注释说明理由）
- 达到上限时记录 Warn 日志，帮助发现异常
- 集成测试中的 mock 分页响应必须设置终止条件
- 此约束同样适用于 `for range` 遍历游标查询的数据库操作

---

## 14. HTTP Middleware ResponseWriter 包装

当中间件需要捕获响应状态码或字节数（如访问日志、指标采集）时，必须使用 `github.com/felixge/httpsnoop` 包装 `http.ResponseWriter`，不能手写 wrapper struct。

### 问题

`http.ResponseWriter` 可能实现可选接口（`http.Hijacker`、`http.Flusher`、`http.Pusher` 等）。手写的 wrapper struct 通常只实现 `Write`/`WriteHeader`/`Header`，丢失了这些可选接口，导致 WebSocket hijacking、SSE 推送、HTTP/2 server push 等功能静默失效。

```go
// ❌ 手写 wrapper：丢失 Hijacker/Flusher 接口
type responseWriterWrapper struct {
    http.ResponseWriter
    statusCode int
}
func (w *responseWriterWrapper) WriteHeader(code int) {
    w.statusCode = code
    w.ResponseWriter.WriteHeader(code)
}
// Hijack() 被吞掉了！WebSocket 中间件 downstream 会 panic

// ✅ 使用 httpsnoop：保留所有可选接口
import "github.com/felixge/httpsnoop"

metrics := httpsnoop.CaptureMetrics(http.HandlerFunc(next.ServeHTTP), w, r)
// metrics.Code — 响应状态码
// metrics.Written — 写入字节数
// metrics.Duration — 处理耗时
// 所有 ResponseWriter 接口完整保留
```

### 约束

- 需要捕获响应元数据（状态码、字节数）的中间件必须使用 `httpsnoop.CaptureMetrics`
- 不需要捕获响应元数据的中间件（如认证、CORS）直接传递原始 `ResponseWriter`
- 引入 `httpsnoop` 依赖时不需要额外审批——它是纯标准库兼容的轻量库
- **例外**：需要捕获**响应体内容**（如错误响应体用于日志记录）的中间件可以使用手写 wrapper，因为 `httpsnoop` 不支持 body capture。手写 wrapper 必须：
  - 嵌入 `http.ResponseWriter` 作为匿名字段（保留可选接口的透传）
  - 显式绕过需要可选接口的请求（如 WebSocket upgrade 请求必须跳过 wrapper，避免 Hijacker 失效）
  - 对 body 截断后再缓存，防止大响应体导致内存暴涨

```go
// ✅ 手写 wrapper 的合法场景：捕获错误响应体用于日志
type responseWriter struct {
    http.ResponseWriter     // 匿名字段，保留 Hijacker/Flusher 等可选接口
    statusCode  int
    body        *bytes.Buffer
    captureBody bool
}

// WebSocket/健康检查等路径直接传递原始 ResponseWriter，不经过 wrapper
if isWebSocketUpgrade(r) {
    next.ServeHTTP(w, r)  // 原始 w，Hijacker 可用
    return
}
```

---

## 15. 孤儿资源对账（Orphan Reconciler）

当操作涉及两个独立系统（如对象存储 + 数据库），主操作成功后辅助操作可能失败（如 DB 记录创建失败），产生"孤儿资源"。重试是即时防护，对账是最终安全网——两者互补，缺一不可。

### 对账器设计模式

对账器（Reconciler）是后台定时任务，扫描系统 A 中的资源，与系统 B 交叉比对，清理不一致的记录。

```go
// ✅ 孤儿对账器：扫描 + 比对 + 清理（internal/usecase/oss3_orphan_reconciler.go）
//
// 算法：
//  1. 分批列举系统 A（MinIO）中的资源
//  2. 对每批资源，检查系统 B（DB）中是否有对应记录
//  3. 存在于 A 但不存在于 B 的资源是孤儿候选
//  4. 跳过冷却期内的候选（可能仍在上传中）
//  5. 删除剩余孤儿（dry-run 模式只记录不删除）
func (uc *OSS3Usecase) ReconcileOrphans(ctx context.Context) (*OrphanReconcileResult, error) {
    // 1. 分布式锁：防止多节点并发对账
    acquired, err := uc.acquireOrphanLock(ctx, lockTTL)
    if !acquired { return nil, nil }  // 其他节点在跑，跳过
    defer uc.releaseOrphanLock(ctx, holderUUID)

    // 2. 分批列举 + 批量比对（内存友好）
    for page := 0; page < maxPages; page++ {
        objects, _ := uc.backend.ListObjects(ctx, bucket, prefix, batchSize)
        keys := extractKeys(objects)
        exists, _ := uc.repo.KeysExist(ctx, keys)  // 批量查询 DB

        for _, obj := range objects {
            if exists[obj.Key] { continue }
            // 3. 冷却期检查：防止误删正在上传的文件
            if time.Since(obj.LastModified) < cooldown { continue }
            // 4. 删除孤儿
            if !dryRun { uc.backend.DeleteObject(ctx, bucket, obj.Key) }
        }
        if !objects.IsTruncated { break }
    }
    return result, nil
}

// ❌ 不做对账，孤儿资源永久残留
// 文件上传后 DB 记录失败 → 文件永远占用存储空间，无人清理
```

### 对账器必备要素

| 要素 | 说明 | 原因 |
| --- | --- | --- |
| 分布式锁 | `SetNX` + TTL，防止多节点并发 | 并发对账可能重复删除 |
| 冷却期 | 跳过最近 N 分钟内创建的资源 | 避免误删正在上传的文件 |
| 分批处理 | 每次处理固定数量（如 500） | 防止内存溢出 |
| 分页上限 | 最大遍历页数（参见[分页循环终止约束](#13-分页循环终止约束)） | 防止无限循环 |
| 详细指标 | 扫描数、发现数、删除数、跳过数、释放字节数 | 可观测性，异常告警 |
| Dry-run 模式 | 配置开关，只记录不删除 | 上线前验证逻辑正确性 |

### 与即时补偿的关系

```text
主操作（上传文件 → 创建 DB 记录）
    │
    ├── 即时重试（backoff.Retry，3 次）
    │   └── 失败 → 记录孤儿指标（RecordOrphaned*）
    │
    └── 对账器（后台定时任务）
        └── 扫描 → 发现孤儿 → 清理
```

**约束**：

- 涉及"两个独立系统"的写操作必须有对账机制（仅靠重试不够）
- 对账器必须使用分布式锁（`SetNX`），锁 TTL 小于调度间隔
- 对账器必须有冷却期，防止误删正在创建的资源
- 对账结果必须记录详细指标（扫描数、删除数、释放空间），便于告警和容量管理
- 对账器必须支持 dry-run 模式，上线前验证逻辑正确性
- 即时补偿（重试 + 孤儿指标）和对账器互补：前者处理大多数情况，后者兜底极端场景

---

## 16. PR 自检清单

提交 PR 前，必须逐项确认：

- [ ] 代码符合 gofmt 格式
- [ ] 无 0 引用代码（或注释说明保留原因）
- [ ] 新文件不超过 500 行（测试文件 800 行）
- [ ] 依赖方向正确，无循环依赖
- [ ] Redis 多操作使用 Lua 脚本
- [ ] repo 方法使用 `DBFromContext`
- [ ] 原子写入通过 `RunAndPublish`
- [ ] 事务提交后的外部操作有补偿逻辑
- [ ] 补偿流程函数保持线性，不拆分步骤到多个小函数，并在函数顶部注释说明原因（参见[补偿流程保持线性](#补偿流程保持线性)）
- [ ] 重试耗尽后记录孤儿资源指标
- [ ] 幂等操作使用 SETNX 标记（如配额提交）
- [ ] defer 清理使用 `context.Background()`
- [ ] HTTP handler 中可能阻塞的 defer 清理（如网络对象 Close）使用 goroutine + timeout 保护（参见[并发规范 - defer 阻塞操作保护](./concurrency.md#9-defer-阻塞操作保护)）
- [ ] 新增功能有日志、追踪、指标
- [ ] 可选依赖未配置时，跳过路径打 Warn 日志（参见[可观测性规范](./observability.md#可选依赖跳过时打-warn)）
- [ ] 新增功能有测试
- [ ] 不打印敏感信息
- [ ] 错误处理正确（不吞错误，有上下文）
- [ ] 接受请求体的 POST/PUT handler 在解析前调用 `http.MaxBytesReader` 限制请求体大小（参见[安全规范 - 请求体大小限制](../05-security-standards.md#请求体大小限制)）
- [ ] 接收集合/数组参数的 handler 限制元素数量上限（参见[安全规范 - 输入集合大小限制](../05-security-standards.md#输入集合大小限制)）
- [ ] 文本截断操作使用 UTF-8 安全截断（`utf8.RuneStart` 回退扫描，参见[性能规范 - UTF-8 安全截断](./performance.md#utf-8-安全截断)）
- [ ] 并行操作按索引写入结果数组，不 append 到共享 slice（参见[并发规范 - 并行操作](./concurrency.md#7-并行操作)）
- [ ] WaitGroup / errgroup 中的 goroutine 有 panic 拦截（`defer recover()`，参见[并发规范 - goroutine panic 拦截](./concurrency.md#2-goroutine-panic-拦截)）
- [ ] 后台 goroutine 通过 context 传递 UserID 等身份信息（参见[并发规范 - 后台 goroutine](./concurrency.md#后台-goroutine-必须传递身份)）
- [ ] 跨队列（Redis/消息队列）传递的工作项在 payload struct 中嵌入身份字段（`UserID`）和 trace context 字段（`TraceID`/`SpanID`），消费端注入 context（参见[并发规范 - 跨队列边界传递身份](./concurrency.md#跨队列边界传递身份) 和[可观测性规范 - 跨队列传播 Trace Context](./observability.md#跨队列边界传播-opentelemetry-trace-context)）
- [ ] 函数入口处校验必要的 context 值（如 UserID、SessionID），缺失时打 Error 日志并 early return（参见[并发规范 - 后台 goroutine 必须显式传递身份](./concurrency.md#后台-goroutine-必须显式传递身份)）
- [ ] 日志事件名使用 `<模块>.<事件>` 的 `snake_case` 点分隔格式（参见[可观测性规范 - 日志事件名命名约定](./observability.md#日志事件名命名约定)）
- [ ] LLM 内容经过预处理管道适配 API 约束（图片多级别降级、文本截断，参见[LLM 内容预处理管道](#12-llm-内容预处理管道)）
- [ ] 安全相关的资源标识符通过集中构造函数生成，不手动拼接（参见[重复代码 - 资源标识符集中构造](#资源标识符必须集中构造)）
- [ ] 跨层校验逻辑（格式校验 + 标识符构造 + 存在性检查）提取为 `usecase/primitives/` 中的独立函数（参见[跨层校验原语](#6-跨层校验原语)）
- [ ] 分页遍历循环有最大迭代次数上限（参见[分页循环终止约束](#13-分页循环终止约束)）
- [ ] 需要捕获响应元数据（状态码/字节数）的中间件使用 `httpsnoop.CaptureMetrics`，不手写 ResponseWriter wrapper；需要捕获响应体内容的中间件可使用手写 wrapper（必须嵌入 `http.ResponseWriter`，参见[HTTP Middleware ResponseWriter 包装](#14-http-middleware-responsewriter-包装)）
- [ ] Lua 脚本有多种返回值时使用 `switch` 穷举，`default` 分支记录 Error 日志（参见[Lua 脚本返回值约定](#lua-脚本返回值约定)）
- [ ] 尽力而为的清理方法使用 `context.Background()` + 日志记录，不返回 error（参见[尽力而为的清理方法](./error-handling.md#尽力而为的清理方法)）
- [ ] 涉及"两个独立系统"的写操作有对账机制兜底（分布式锁 + 冷却期 + 分批扫描，参见[孤儿资源对账](#15-孤儿资源对账orphan-reconciler)）

---

## 总结

代码质量是团队共识的体现。遵循这些约束，让代码库保持健康、可维护、可演进。

> _"今天的纪律是明天的自由。"_
