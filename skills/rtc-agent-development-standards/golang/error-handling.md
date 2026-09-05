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

---

## 4. 错误消息规范

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

## 5. panic 的边界

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

## 6. 反模式

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
