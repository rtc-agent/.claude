# Go 错误处理

> _错误是值，不是异常。显式处理每一个错误，让错误链成为调试的地图。_

---

## 1. 错误传播

### 用 `%w` 包装，附加上下文

错误在传播时应当积累上下文——每一层告诉读者"什么操作失败了"。

```go
// ✅ 每一层附加上下文
func (s *UserService) GetUser(ctx context.Context, id string) (*User, error) {
    user, err := s.repo.Find(ctx, id)
    if err != nil {
        return nil, fmt.Errorf("get user %s: %w", id, err)
    }
    return user, nil
}

func (h *Handler) handleGetUser(w http.ResponseWriter, r *http.Request) {
    user, err := h.service.GetUser(r.Context(), userID)
    if err != nil {
        h.logger.Error("handle get user", "id", userID, "error", err)
        http.Error(w, "internal error", http.StatusInternalServerError)
        return
    }
    // ...
}
```

### 避免重复包装

如果上层已经有足够的上下文，下层不必再包一层。

```go
// ❌ 重复包装，错误消息臃肿
// "handle get user: get user 123: find user 123: database timeout"
func (h *Handler) handleGetUser(w http.ResponseWriter, r *http.Request) {
    user, err := h.service.GetUser(r.Context(), userID)
    if err != nil {
        return fmt.Errorf("handle get user: %w", err)  // 多余的一层
    }
}

// ✅ 只在有意义的边界包装
// "get user 123: database timeout"
```

### 不要裸 `return err`

无上下文的裸返回让调用方无法定位问题。

```go
// ❌ 裸返回
func (s *Service) ProcessOrder(order *Order) error {
    if err := s.validate(order); err != nil {
        return err  // 什么操作？什么参数？
    }
    if err := s.save(order); err != nil {
        return err  // 同样不明
    }
    return nil
}

// ✅ 附加上下文
func (s *Service) ProcessOrder(order *Order) error {
    if err := s.validate(order); err != nil {
        return fmt.Errorf("validate order %s: %w", order.ID, err)
    }
    if err := s.save(order); err != nil {
        return fmt.Errorf("save order %s: %w", order.ID, err)
    }
    return nil
}
```

---

## 2. 自定义错误类型

### 何时创建

当调用方需要基于错误类型做**分支决策**时，使用自定义错误类型。

```go
// ✅ 调用方需要根据错误类型做不同处理
var ErrUserNotFound = errors.New("user not found")

func (h *Handler) handleGetUser(w http.ResponseWriter, r *http.Request) {
    user, err := h.service.GetUser(r.Context(), userID)
    if err != nil {
        if errors.Is(err, ErrUserNotFound) {
            http.Error(w, "not found", http.StatusNotFound)
            return
        }
        h.logger.Error("get user failed", "error", err)
        http.Error(w, "internal error", http.StatusInternalServerError)
        return
    }
    // ...
}
```

### 错误类型设计

```go
// 简单哨兵错误：不需要额外信息
var (
    ErrNotFound      = errors.New("not found")
    ErrUnauthorized  = errors.New("unauthorized")
    ErrForbidden     = errors.New("forbidden")
)

// 带数据的错误类型：需要错误详情
type ValidationError struct {
    Field   string
    Message string
}

func (e *ValidationError) Error() string {
    return fmt.Sprintf("validation failed: %s %s", e.Field, e.Message)
}

// 使用
func validateEmail(email string) error {
    if !strings.Contains(email, "@") {
        return &ValidationError{Field: "email", Message: "invalid format"}
    }
    return nil
}

// 调用方用 errors.As 提取
var valErr *ValidationError
if errors.As(err, &valErr) {
    fmt.Printf("field: %s, message: %s\n", valErr.Field, valErr.Message)
}
```

### Repo 层的 `(nil, nil)` 约定

Repo 方法的标准做法是"未找到"时返回哨兵错误（如 `ErrNotFound`）。但当调用方需要频繁区分"未找到"和"数据库错误"时（如 instant upload 路径先查再创建），返回 `(nil, nil)` 减少调用方的 `errors.Is` 开销。

```go
// ✅ (nil, nil) 表示未找到 — 必须用注释说明原因
// GetByUserAndKey returns (nil, nil) when the record is not found.
// This deviates from the typical sentinel error pattern because the upload
// path needs to distinguish "not found" from "database error" on every call.
func (r *fileRepo) GetByUserAndKey(ctx context.Context, userID, key string) (*model.File, error) {
    var file model.File
    err := DBFromContext(ctx, r.db).Where("user_id = ? AND key = ?", userID, key).First(&file).Error
    if err != nil {
        if errors.Is(err, gorm.ErrRecordNotFound) {
            return nil, nil  // not found is not an error here
        }
        return nil, fmt.Errorf("get file by user %s key %s: %w", userID, key, err)
    }
    return &file, nil
}

// ❌ 不一致：同一个 repo 有些方法返回 ErrNotFound，有些返回 (nil, nil)
func (r *repo) Get(ctx context.Context, id string) (*Entity, error) {
    // ...
    if errors.Is(err, gorm.ErrRecordNotFound) {
        return nil, ErrNotFound  // 标准做法
    }
}
func (r *repo) GetByKey(ctx context.Context, key string) (*Entity, error) {
    // ...
    if errors.Is(err, gorm.ErrRecordNotFound) {
        return nil, nil  // 同一个 repo 两种风格，混乱
    }
}
```

**约束**：

- `(nil, nil)` 仅在 repo 层使用，usecase/handler 层不采用此约定
- 必须在方法注释中说明为什么选择 `(nil, nil)` 而非哨兵错误
- 同一个 repo 内保持一致——要么全部用哨兵错误，要么全部用 `(nil, nil)`

---

## 3. 哨兵错误

### 命名约定

```go
// 格式：Err<Domain><Condition>
var (
    ErrUserNotFound       = errors.New("user not found")
    ErrUserAlreadyExists  = errors.New("user already exists")
    ErrRoomClosed         = errors.New("room is closed")
    ErrPermissionDenied   = errors.New("permission denied")
)
```

- **`Err` 前缀**：一眼识别为错误
- **PascalCase**：因为是导出变量
- **消息小写**：Go 错误消息惯例

### 领域特定错误前缀

当一个模块需要与通用错误（`ErrNotFound`）区分时，使用领域前缀。这在协议兼容层（如 S3）中尤为重要——不同协议的同名错误需要独立识别。

```go
// ✅ 通用错误 — 无前缀
var (
    ErrNotFound      = errors.New("not found")
    ErrAlreadyExists = errors.New("already exists")
)

// ✅ 领域前缀 — S3 兼容层
const (
    S3ErrAccessDenied        = s3ErrorCode("AccessDenied")
    S3ErrBucketAlreadyExists = s3ErrorCode("BucketAlreadyExists")
    S3ErrNoSuchKey           = s3ErrorCode("NoSuchKey")
    S3ErrNoSuchUpload      = s3ErrorCode("NoSuchUpload")
)

// ✅ 领域前缀 — OSS3 后端
var (
    ErrOSSEntityTooSmall = errors.New("EntityTooSmall")
    ErrOSSInvalidPart    = errors.New("InvalidPart")
)
```

**判断标准**：错误需要被调用方按**领域**区分（而非仅按错误类型）时，加前缀。通用业务逻辑使用 `Err*`，协议兼容层使用协议前缀（如 `S3Err*`）。

### 协议兼容层的错误类型系统

当项目需要实现外部协议兼容（如 S3 API）时，哨兵常量不够用——协议要求错误响应包含特定的错误码、人类可读消息和 HTTP 状态码。此时需要定义完整的错误类型系统。

```go
// ✅ 协议错误类型：携带 code + message + HTTP status
type S3Error struct {
    Code       string  // 协议规定的错误码，如 "AccessDenied"
    Message    string  // 协议规定的人类可读消息
    HTTPCode   int     // 对应的 HTTP 状态码
}

func (e *S3Error) Error() string {
    return fmt.Sprintf("%s: %s", e.Code, e.Message)
}

// 集中定义所有协议错误
var (
    ErrAccessDenied    = &S3Error{"AccessDenied", "Access Denied", http.StatusForbidden}
    ErrNoSuchKey       = &S3Error{"NoSuchKey", "The specified key does not exist", http.StatusNotFound}
    ErrInvalidKeyFormat = &S3Error{"InvalidKeyFormat", "The specified key does not match the required format", http.StatusBadRequest})
```

**何时使用协议错误类型**：

| 场景              | 错误方式                      | 原因                                                   |
| ----------------- | ----------------------------- | ------------------------------------------------------ |
| 通用业务逻辑      | Go 哨兵错误 `Err*`            | 内部使用，`errors.Is` 匹配                             |
| 领域区分          | 带前缀的常量 `S3Err*`         | 需要按领域分类                                         |
| 协议兼容层        | 错误类型系统 `*S3Error`       | 需要映射到协议规定的错误码、消息、HTTP 状态            |

**规则**：

- 协议错误集中在一个文件中定义（如 `errors.go`），不散落在各 handler
- 错误到 HTTP 状态的映射通过类型本身完成，不在 handler 中逐个 `switch`
- 调用方用 `errors.As` 提取协议错误：`var s3Err *S3Error; if errors.As(err, &s3Err) { ... }`

### 错误分组

一个包/模块的哨兵错误集中在一个文件中定义。

```go
// internal/user/errors.go
package user

import "errors"

var (
    ErrNotFound       = errors.New("user not found")
    ErrAlreadyExists  = errors.New("user already exists")
    ErrInactive       = errors.New("user is inactive")
    ErrSuspended      = errors.New("user is suspended")
)
```

### 聚合检查器

当一组相关的哨兵错误需要在调用方频繁做同一类判断时，提供聚合检查器函数，避免调用方遗漏检查某个变体。调用方只需调用一个函数，不必了解包内所有哨兵错误的细节。

```go
// internal/repo/errors.go
// IsNotFound checks whether the error is a "not found" variant.
func IsNotFound(err error) bool {
    return errors.Is(err, ErrNotFound) ||
        errors.Is(err, ErrSessionNotFound) ||
        errors.Is(err, ErrFileNotFound) ||
        errors.Is(err, ErrMultipartUploadNotFound)
        // ... 所有 *NotFound 变体
}

// ✅ 调用方：一行检查，不会遗漏
if repo.IsNotFound(err) {
    http.Error(w, "not found", http.StatusNotFound)
    return
}

// ❌ 调用方手动枚举：新增 ErrXxxNotFound 时容易遗漏
if errors.Is(err, repo.ErrNotFound) || errors.Is(err, repo.ErrFileNotFound) {
    // 漏了 ErrSessionNotFound！
}
```

**约束**：

- 聚合检查器只聚合语义相同的错误（如所有 `*NotFound` 变体），不混合不同语义的错误
- 新增哨兵错误时，如果属于已有聚合类别，必须同步更新聚合函数
- 聚合函数命名格式为 `Is<Category>`（如 `IsNotFound`、`IsRetryable`），返回 `bool`
- 聚合函数与哨兵错误定义放在同一个文件中（如 `errors.go`），保持同步

---

## 4. 可重试错误检测

当操作可能因瞬态故障失败时，需要区分"可重试"和"不可重试"错误，避免无意义地重试逻辑错误。

### 数据库可重试错误

GORM 的 `ErrDuplicatedKey` 可能来自业务冲突（不可重试）或死锁/序列化失败（可重试）。必须按错误码细分。

```go
// ✅ 精确检测可重试的数据库错误
func isRetryableSQLErr(err error) bool {
    if err == nil {
        return false
    }
    if !errors.Is(err, gorm.ErrDuplicatedKey) {
        return false
    }
    // 只有特定 PostgreSQL 错误码才是瞬态冲突
    var pgErr *pgconn.PgError
    if !errors.As(err, &pgErr) {
        return false
    }
    switch pgErr.Code {
    case "23505",  // unique_violation（并发插入冲突）
         "40001":  // serialization_failure
        return true
    }
    return false
}
```

### 错误分类映射

不同来源的错误需要映射到不同的 HTTP 响应码。集中处理，避免在每个 handler 中重复。

```go
// ✅ 集中映射
func mapOSSErrToHTTP(err error) (int, string) {
    switch {
    case errors.Is(err, repo.ErrBucketAlreadyExists):
        return http.StatusConflict, "BucketAlreadyExists"
    case errors.Is(err, repo.ErrKeyNotFound):
        return http.StatusNotFound, "NoSuchKey"
    case errors.Is(err, repo.ErrQuotaExceeded):
        return http.StatusInsufficientStorage, "InsufficientStorage"
    case errors.Is(err, gorm.ErrDuplicatedKey):
        return http.StatusConflict, "BucketAlreadyExists"
    default:
        return http.StatusInternalServerError, "InternalError"
    }
}
```

**约束**：

- 新增 repo 操作时，考虑其可能的瞬态错误，并在 isRetryable 函数中覆盖

- 错误到 HTTP 状态的映射集中在一个函数中，不散落在 handler 各处

- 不可重试的错误（权限、校验、业务规则）必须用 `backoff.Permanent` 阻止重试

### 重试配置（cenkalti/backoff）

项目使用 `cenkalti/backoff/v5` 库实现指数退避重试。所有重试必须显式配置参数，不使用默认值（默认间隔过长）。

```go
// ✅ 标准重试配置
import "github.com/cenkalti/backoff/v5"

bo := backoff.NewExponentialBackOff()
bo.InitialInterval = 50 * time.Millisecond   // 快速首次重试
bo.MaxInterval = 200 * time.Millisecond      // 防止间隔过长

_, err := backoff.Retry(ctx, func() (struct{}, error) {
    if err := doOperation(ctx); err != nil {
        if !isTransientError(err) {
            return struct{}{}, backoff.Permanent(err)  // 不可重试，立即停止
        }
        return struct{}{}, err  // 可重试，继续
    }
    return struct{}{}, nil
},
    backoff.WithBackOff(bo),
    backoff.WithMaxTries(3),
)
```

**参数选择指南**：

| 场景                               | InitialInterval | MaxInterval | MaxTries | 原因                               |
| ---------------------------------- | --------------- | ----------- | -------- | ---------------------------------- |
| Redis 瞬态错误（如 quota commit）  | 50ms            | 200ms       | 3        | 快速重试，Redis 故障通常秒级恢复   |
| 等待状态就绪（如 InterruptID）     | 20ms            | 200ms       | 10       | 高频轮询短时间，等待异步状态变更   |
| DB 记录创建（如 copy/delete 后）   | 100ms           | 2s          | 3        | 较长间隔，避免对 DB 造成压力       |

### 重试前提：幂等性保证

**重试有副作用的操作前，必须确保操作具有幂等性。** 无幂等保护的重试会导致重复扣费、重复创建记录等数据损坏。

```go
// ✅ 幂等重试：SETNX 标记防止重复扣费
// commitQuotaWithRetry retries CommitQuota on transient Redis errors.
// The idempotency marker inside CommitQuota (SETNX) prevents double-charging.
func commitQuotaWithRetry(ctx context.Context, uc *Usecase, userID, requestID string, amount int64) error {
    // CommitQuota 内部使用 SETNX quota_request_id 作为幂等标记
    // 相同的 requestID 只会扣费一次，重试安全
    bo := backoff.NewExponentialBackOff()
    bo.InitialInterval = 50 * time.Millisecond
    bo.MaxInterval = 200 * time.Millisecond
    _, err := backoff.Retry(ctx, func() (struct{}, error) {
        if err := uc.CommitQuota(ctx, userID, requestID, amount); err != nil {
            if !isTransientRedisError(err) {
                return struct{}{}, backoff.Permanent(err)
            }
            return struct{}{}, err
        }
        return struct{}{}, nil
    }, backoff.WithBackOff(bo), backoff.WithMaxTries(3))
    return err
}

// ✅ 幂等重试：条件写入（仅当记录不存在时创建）
_, retryErr := backoff.Retry(ctx, func() (struct{}, error) {
    return struct{}{}, uc.CreateFileRecord(ctx, file)  // DB UNIQUE 约束保证幂等
}, backoff.WithBackOff(bo), backoff.WithMaxTries(3))
```

```go
// ❌ 无幂等保护的重试 — 每次重试都扣费
_, err := backoff.Retry(ctx, func() (struct{}, error) {
    return struct{}{}, uc.ChargeBalance(ctx, userID, amount)  // 每次调用都扣钱！
}, backoff.WithBackOff(bo), backoff.WithMaxTries(3))
```

**常见幂等机制**：

- **SETNX 标记** -- Redis 操作。`SETNX quota_request_id "1"` 防止重复扣费
- **UNIQUE 约束** -- DB 插入。相同 key 只能插入一次
- **条件写入** -- 状态变更。`UPDATE ... WHERE status = 'pending'`
- **SetNX 开关** -- 事件触发。`SETNX orphan_triggered "1"` 防止重复触发

### 重试耗尽后的处理

所有重试失败后，根据操作的性质选择不同的处理策略。

```go
// ✅ 策略 1：记录孤儿，等待对账（主操作成功，辅助操作失败）
// 场景：文件已上传到 MinIO，但 DB 记录创建失败
if retryErr != nil {
    logger.Error(ctx, "failed to create file record after all retries, orphaned object",
        zap.String("user_id", userID),
        zap.String("key", key),
        zap.Error(retryErr))
    RecordOrphanedRecord("create_failed")  // Prometheus 指标，触发告警
    // 对账任务会定期清理孤儿记录
}

// ✅ 策略 2：静默降级（非关键操作）
// 场景：quota commit 失败，文件已上传成功
if err != nil {
    logger.Error(ctx, "quota commit failed after retries; reconciliation will sync",
        zap.String("quota_request_id", requestID),
        zap.Error(err))
    RecordOrphanedQuotaCommit()  // 指标告警，不阻塞主流程
    // Redis pending key 会自然过期；对账任务会同步 Redis 与 DB
}

// ✅ 策略 3：返回错误（关键操作）
// 场景：等待 InterruptID 超时，但不影响主流程
if err != nil || gotID == "" {
    logger.Warn(ctx, "InterruptID not available after retry",
        zap.String("turn", turnID))
    return "", nil  // 返回空值，调用方处理降级
}
```

**约束**：

- 重试耗尽后必须记录 error 或 warn 日志，包含足够的上下文（user_id、key、request_id）
- 非关键操作的重试耗尽不应阻塞主流程——记录指标后继续
- 产生孤儿资源时，必须有对账机制清理（异步任务或 TTL 过期）
- 用 Prometheus 指标（如 `RecordOrphanedRecord`）追踪重试耗尽事件，便于告警

---

## 5. 事务补偿回调

当 `RunAndPublish` 事务提交成功后，外部资源（如 OSS 文件）已无法通过数据库回滚。如果后续操作（如更新状态）失败，需要提供补偿回调让调用方清理外部资源。

```go
// ✅ 补偿回调清理外部资源
func (uc *Usecase) Create(ctx context.Context, req *CreateRequest) (*Result, error) {
    // 事务写入
    result, err := uc.deps.UpdatePublisher.RunAndPublish(ctx, func(txCtx context.Context) (*Result, error) {
        // ... 事务内写入
        return &Result{Key: key, VersionID: versionID}, nil
    })
    if err != nil {
        // 可重试错误：补偿回滚后端对象，返回重试
        if isRetryableSQLErr(err) {
            backendRollback := func(context.Context) error {
                return uc.backend.DeleteObject(ctx, result.Bucket, result.Key)
            }
            return nil, newRollbackErr(err, backendRollback)
        }
        return nil, mapErr(err)
    }
    // ...
}
```

**规则**：

- 补偿回调用 `context.WithoutCancel(ctx)` 或新 context 调用，不复用可能已取消的 txCtx

- 补偿回调中失败不阻塞主流程错误返回，仅记录 warn 日志

- `RunAndPublish` 返回 error 后，检查是否需要补偿——不要把补偿逻辑遗漏

### 尽力而为的清理方法

当清理操作（释放配额、删除临时资源）失败不影响主操作的成功时，将清理封装为"尽力而为"的方法：使用 `context.Background()` 防止取消传播，记录日志但不返回错误。调用方无需处理清理失败——对账机制（reconciliation）最终会同步状态。

```go
// ✅ 尽力而为的清理方法
// ReleaseQuotaForDeletedFile releases quota after a file is deleted.
// Uses Background context internally to avoid cancellation issues when
// the client disconnects. Logs errors but does not fail the caller —
// reconciliation will sync eventually.
func (uc *Usecase) ReleaseQuotaForDeletedFile(ctx context.Context, userID, key string, fileSize int64) {
    if fileSize <= 0 {
        return
    }
    bgCtx := context.Background()
    if err := uc.AdjustQuota(bgCtx, userID, -fileSize); err != nil {
        logger.Warn(bgCtx, "failed to adjust quota after delete",
            zap.String("user_id", userID),
            zap.String("key", key),
            zap.Int64("size", fileSize),
            zap.Error(err))
    }
}

// ❌ 返回错误给调用方，调用方无法有意义地处理
func (uc *Usecase) ReleaseQuotaForDeletedFile(ctx context.Context, userID string, fileSize int64) error {
    return uc.AdjustQuota(ctx, userID, -fileSize)
    // 调用方：if err := ...; err != nil { ??? }
    // 清理失败不应该导致删除操作返回 500
}
```

**约束**：

- 方法注释必须说明"为什么是尽力而为"（如"reconciliation will sync eventually"）
- 内部使用 `context.Background()`，不接受调用方的 context（可能已取消）
- 失败时记录 Warn 级别日志，包含足够的上下文供对账使用
- 返回值必须是 `void`（Go 中不返回 error），防止调用方误用
- 必须有对账机制兜底（reconciler、TTL 过期），不能仅依赖尽力而为的清理

**判断标准**：清理操作失败后，主操作是否应该返回错误？如果答案是"不应该"（如文件已删除，配额释放失败不应影响删除的成功响应），使用尽力而为模式。如果答案是"应该"（如事务提交后外部状态更新失败），使用[事务补偿回调](#5-事务补偿回调)。

---

## 6. 错误消息规范

### 格式

- **小写开头，无标点**：Go 错误消息惯例
- **包含上下文**：操作 + 参数 + 原因

```go
// ✅ 清晰、有上下文
fmt.Errorf("read config file %s: %w", path, err)
fmt.Errorf("connect to database %s: %w", dsn, err)

// ❌ 无上下文
fmt.Errorf("failed")
fmt.Errorf("error occurred")
fmt.Errorf("Read Config File failed.")  // 大写开头、有标点
```

### 不要包含敏感信息

```go
// ❌ 密码、token 出现在错误中
fmt.Errorf("login failed for user %s with password %s: %w", username, password, err)

// ✅ 只包含标识信息
fmt.Errorf("login failed for user %s: %w", username, err)
```

---

## 7. panic 的边界

### panic 仅用于不可恢复的编程错误

```go
// ✅ 不可恢复的编程错误，可以 panic
func NewConfig(path string) *Config {
    c, err := LoadConfig(path)
    if err != nil {
        panic(fmt.Errorf("load config %s: %w", path, err))  // 启动时配置错误不可恢复
    }
    return c
}

// ✅ 不应该发生的状态
func (s *Server) init() {
    if s.db == nil {
        panic("database not initialized")  // 编程错误
    }
}
```

### 在边界 recover

```go
// HTTP handler 的 recover
func RecoveryMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        defer func() {
            if r := recover(); r != nil {
                logger.Error("panic recovered",
                    "panic", r,
                    "stack", string(debug.Stack()),
                    "path", r.URL.Path,
                )
                http.Error(w, "internal error", http.StatusInternalServerError)
            }
        }()
        next.ServeHTTP(w, r)
    })
}

// main 的 recover
func main() {
    defer func() {
        if r := recover(); r != nil {
            logger.Fatal("panic", "panic", r, "stack", string(debug.Stack()))
            os.Exit(1)
        }
    }()
    // ...
}
```

---

## 8. 反模式

### ❌ 吞掉错误

```go
// ❌ 完全忽略错误
data, _ := os.ReadFile(path)

// ✅ 至少记录日志
data, err := os.ReadFile(path)
if err != nil {
    log.Printf("read file %s: %v", path, err)
    // 使用默认值或返回
}
```

如果确实要忽略，显式注释说明原因：

```go
// 忽略错误：关闭文件失败不影响业务逻辑
_ = f.Close()
```

### ❌ 字符串匹配错误

```go
// ❌ 用字符串比较错误
if err.Error() == "user not found" {
    // 脆弱，消息一变就坏
}

// ✅ 用 errors.Is 比较
if errors.Is(err, ErrNotFound) {
    // 稳定，基于类型
}
```

**例外：外部库错误的瞬态分类**。当外部库（如 Redis 客户端、HTTP 客户端）不暴露类型化错误，而你需要区分"可重试"和"不可重试"错误时，可以基于错误消息的关键字进行分类。必须封装为独立的检测函数并附注释说明为什么使用字符串匹配。

```go
// ✅ 外部库不暴露类型化错误，字符串匹配是唯一选择
// isTransientRedisError reports whether the error is a transient Redis failure
// that is safe to retry (connection reset, I/O timeout, NOSCRIPT).
func isTransientRedisError(err error) bool {
    if err == nil {
        return false
    }
    msg := err.Error()
    return strings.Contains(msg, "connection reset") ||
        strings.Contains(msg, "i/o timeout") ||
        strings.Contains(msg, "NOSCRIPT")
}

// ❌ 不封装，散落在业务逻辑中
if strings.Contains(err.Error(), "timeout") {
    // 重试...
}
```

**约束**：

- 字符串匹配仅用于对**外部库**错误的分类，不用于项目内部错误的判断
- 必须封装为命名函数（如 `isTransient*`），附带 godoc 说明为什么不能用 `errors.Is`
- 匹配关键字必须足够具体（如 `"connection reset"` 而非 `"error"`），避免误判

### ❌ 在中间层多次包装

```go
// ❌ 三层包装，错误消息臃肿
// "handle: service: repo: timeout"
func (h *Handler) handle() error {
    err := h.service.Process()
    if err != nil {
        return fmt.Errorf("handle: %w", err)
    }
}

func (s *Service) Process() error {
    err := s.repo.Save()
    if err != nil {
        return fmt.Errorf("service: %w", err)
    }
}

func (r *Repository) Save() error {
    return fmt.Errorf("repo: %w", dbErr)
}

// ✅ 只在有意义的边界包装一次
// "save to database: timeout"
```

### ❌ 返回 nil error 但结果无效

```go
// ❌ 返回了 nil error，但 user 是 nil
func (s *Service) GetUser(id string) (*User, error) {
    user, _ := s.repo.Find(id)  // 忽略了错误
    return user, nil
}

// ✅ 错误必须传播
func (s *Service) GetUser(id string) (*User, error) {
    user, err := s.repo.Find(id)
    if err != nil {
        return nil, fmt.Errorf("find user %s: %w", id, err)
    }
    return user, nil
}
```
