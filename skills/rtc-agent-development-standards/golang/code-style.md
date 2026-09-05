# Go 代码风格

> _格式不是偏好，是纪律。统一风格让代码像同一个人心智的产物。_

---

## 1. 格式化

**唯一标准：`gofmt`。** 没有配置选项，没有风格争论。所有 Go 文件必须通过 `gofmt` 格式化。

```bash
# 检查格式
gofmt -l .

# 自动修复
gofmt -w .
```

### 行长度

Go 不强求行长度限制，但过长的行损害可读性。

- **建议**：单行不超过 120 字符
- **多参数函数签名**：一行一个参数，或合理分组

```go
// ✅ 参数多时换行
func NewUserService(
    repo UserRepository,
    cache Cache,
    notifier Notifier,
) *UserService {

// ✅ 短签名保持一行
func GetUser(ctx context.Context, id string) (*User, error) {
```

### 大括号

- 左大括号不换行（K&R 风格）
- `if`/`for`/`switch` 后必须有大括号，即使只有一行

```go
// ✅ 正确
if err != nil {
    return err
}

// ❌ 缺少大括号
if err != nil
    return err
```

---

## 2. 包命名

### 规则

- **短小写单词**：`user`、`event`、`room`，不用 `user_service` 或 `UserService`
- **不用下划线**：`chatpanel` 而非 `chat_panel`
- **不用复数**：`user` 而非 `users`（包内类型已经表达了复数语义）
- **避免与标准库冲突**：不要用 `time`、`http`、`context` 等

### 包名即命名空间

包名会成为类型的前缀，避免在类型名中重复包名。

```go
// ✅ 包名 + 类型名自然组合
package user
type Service struct { ... }      // user.Service — 清晰
type Repository interface { ... } // user.Repository — 清晰

// ❌ 类型名重复包名
package user
type UserService struct { ... }   // user.UserService — 冗余
type UserData struct { ... }      // user.UserData — 冗余
```

### 避免 `util` / `helper` / `common` 包

这些包名没有表达任何领域含义，最终变成垃圾场。

```go
// ❌ 无意义的包
package util
func FormatDate(t time.Time) string { ... }
func ValidateEmail(s string) bool { ... }

// ✅ 按领域组织
package formatting
func Date(t time.Time) string { ... }

package validation
func Email(s string) bool { ... }
```

---

## 3. 变量与函数命名

### 导出与非导出

| 场景 | 命名 | 示例 |
|------|------|------|
| 导出类型 | PascalCase | `UserService`, `RoomConfig` |
| 导出函数 | PascalCase | `GetUser`, `CreateRoom` |
| 非导出类型 | camelCase | `userCache`, `roomState` |
| 非导出函数 | camelCase | `parseConfig`, `validateInput` |
| 导出常量 | PascalCase | `MaxRetryCount`, `DefaultTimeout` |
| 非导出常量 | camelCase 或 PascalCase | 同包内可见性一致即可 |

### 局部变量

作用域越小，名字越短。

```go
// ✅ 循环变量可以短
for i, item := range items {
    process(item)
}

// ✅ 短作用域内简短命名
f, err := os.Open(path)
if err != nil {
    return err
}
defer f.Close()

// ✅ 长作用域或重要变量用完整名字
userSession := sessionManager.GetSession(userID)
```

### 方法接收器

- **短名字**：1-2 个字母，或类型首字母
- **同一类型的所有方法使用相同的接收器名**

```go
// ✅ 一致的接收器命名
func (s *UserService) GetUser(id string) (*User, error) { ... }
func (s *UserService) CreateUser(name string) (*User, error) { ... }
func (s *UserService) DeleteUser(id string) error { ... }

// ❌ 同一类型不同接收器名
func (u *UserService) GetUser(id string) (*User, error) { ... }
func (s *UserService) CreateUser(name string) (*User, error) { ... }
```

### 函数命名

函数名描述行为，而非实现。

```go
// ✅ 描述行为
func GetUser(ctx context.Context, id string) (*User, error)
func ValidateEmail(email string) error
func RetryWithBackoff(fn func() error, maxRetries int) error

// ❌ 描述实现或模糊
func HandleUser(ctx context.Context, id string) (*User, error)  // handle 什么？
func Check(email string) bool                                    // check 什么？
func Process(data []byte) error                                  // process 什么？
```

---

## 4. 常量

### 分组与 iota

相关常量用 `const ()` 分组，用 `iota` 生成递增序列。

```go
const (
    StatusUnknown  = iota // 0
    StatusActive          // 1
    StatusInactive        // 2
    StatusSuspended       // 3
)
```

### 带单位的常量

常量名应包含单位，避免歧义。

```go
// ✅ 单位明确
const (
    DefaultTimeoutSeconds = 30
    MaxRetries            = 3
    MaxPayloadBytes       = 1024 * 1024  // 1MB
)

// ❌ 单位不明
const (
    DefaultTimeout = 30    // 秒？毫秒？
    MaxSize        = 1024  // 字节？KB？
)
```

---

## 5. 导入分组

三个分组，空行分隔。用 `goimports` 自动管理。

```go
import (
    // 标准库
    "context"
    "fmt"
    "sync"
    "time"

    // 第三方库
    "github.com/anthropics/anthropic-sdk-go"
    "github.com/centrifugal/centrifuge-go"
    "go.uber.org/zap"

    // 项目内部
    "rtc-agent/server/internal/config"
    "rtc-agent/server/internal/user"
)
```

### 避免导入别名

除非解决命名冲突，否则不用别名。

```go
// ✅ 正常导入
import "github.com/pkg/errors"

// ✅ 命名冲突时使用别名，并注释原因
import (
    "database/sql"
    pg "github.com/lib/pq"  // 避免与 sql 包混淆
)

// ❌ 无意义的别名
import u "rtc-agent/server/internal/user"
```

---

## 6. 注释

### 公开 API 必须有 godoc 注释

```go
// GetUser 根据 ID 获取用户信息。
//
// 如果用户不存在，返回 ErrUserNotFound。
// 已软删除的用户不会被返回。
func (s *UserService) GetUser(ctx context.Context, id string) (*User, error) {
```

格式：`// FunctionName` + 描述（以被注释对象的名字开头）。

### 包注释

每个包应有 `doc.go` 或在包内文件中提供包级注释。

```go
// Package user 提供用户管理的核心逻辑，包括用户创建、查询、更新和删除。
//
// 本包不处理认证和授权，这些由 auth 包负责。
package user
```

### 内部注释

内部实现注释解释"为什么"，而非"是什么"。

```go
// 使用带缓冲的 channel 而非无缓冲，
// 因为事件生产者不应当阻塞在消费者处理慢时。
events := make(chan Event, 100)
```

---

## 7. 接口设计

### 小接口优于大接口

Go 的接口应在消费方定义，方法数控制在 1-3 个。

```go
// ✅ 小接口，灵活组合
type Reader interface {
    Read(ctx context.Context, id string) (*User, error)
}

type Writer interface {
    Write(ctx context.Context, user *User) error
}

// ✅ 按需组合
type ReadWriter interface {
    Reader
    Writer
}

// ❌ 大接口，难以实现和测试
type UserRepo interface {
    Create(ctx context.Context, user *User) error
    Get(ctx context.Context, id string) (*User, error)
    Update(ctx context.Context, user *User) error
    Delete(ctx context.Context, id string) error
    List(ctx context.Context, opts ListOptions) ([]*User, error)
    Count(ctx context.Context) (int, error)
    // ...更多方法
}
```

### 在消费方定义接口

接口应当由使用它的包定义，而非实现它的包。

```go
// ✅ 在消费方定义
package notification

type UserProvider interface {
    GetUser(ctx context.Context, id string) (*user.User, error)
}

type Service struct {
    users UserProvider  // 不依赖具体的 user.Service
}

// ❌ 直接使用具体实现
package notification

type Service struct {
    users *user.Service  // 紧耦合
}
```
