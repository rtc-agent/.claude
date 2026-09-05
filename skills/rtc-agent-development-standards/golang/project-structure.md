# Go 项目结构

> _项目结构是架构的骨架。清晰的目录布局让新人一眼看懂代码如何组织，让老手快速定位问题。_

---

## 1. 当前架构说明

server 仓库采用分层架构，按职责划分目录：

```
server/
├── main.go                      # 程序入口
├── cmd/                         # Cobra 子命令
├── internal/                    # 私有业务逻辑
│   ├── handler/                 # 协议适配层
│   │   ├── http/                # HTTP API
│   │   └── rpc/                 # RPC 接口
│   ├── usecase/                 # 业务逻辑层
│   ├── repo/                    # 数据访问层
│   ├── model/                   # 数据模型
│   ├── svc/                     # Service Context（DI 容器）
│   └── infra/                   # 基础设施
│       ├── middleware/          # 中间件
│       ├── httputil/            # HTTP 工具
│       ├── context/             # Context 工具
│       ├── cache/               # 缓存
│       ├── config/              # 配置加载
│       └── auth/                # 认证
├── pkg/                         # 可复用包
│   ├── centrifuge-plus/         # Centrifuge 扩展
│   ├── logger/                  # 日志
│   ├── turn-agent/              # TURN agent
│   ├── rtc-queue/               # RTC 队列
│   └── protocol/                # 协议定义
├── etc/                         # 配置文件
│   └── dev/                     # 开发环境配置
├── scripts/                     # 脚本
└── bin/                         # 构建产物
```

### 请求流转路径

```
HTTP/RPC 请求
    ↓
handler/          协议适配，参数解析
    ↓
usecase/          业务逻辑编排
    ↓
repo/             数据持久化
    ↓
model/            数据模型定义（贯穿各层）
```

### Service Context（`svc/`）

`svc/` 是依赖注入容器，持有所有服务的实例。在 `main.go` 中初始化一次，传递给各层使用。

```go
// internal/svc/context.go
type ServiceContext struct {
    Config     *config.Config
    Logger     *logger.Logger
    UserRepo   repo.UserRepository
    RoomRepo   repo.RoomRepository
    UserUC     *usecase.UserUseCase
    RoomUC     *usecase.RoomUseCase
}

func NewServiceContext(cfg *config.Config) *ServiceContext {
    // 初始化依赖
}
```

---

## 2. 目录职责定义

| 目录 | 职责 | 不应包含 |
|------|------|---------|
| `main.go` | 程序入口，解析命令行，调用 `cmd/` | 业务逻辑 |
| `cmd/` | Cobra 子命令定义 | 业务逻辑 |
| `internal/handler/` | 协议适配（HTTP/RPC），参数校验，响应格式化 | 业务逻辑 |
| `internal/usecase/` | 业务逻辑编排，事务管理 | HTTP/RPC 细节、SQL |
| `internal/repo/` | 数据访问（数据库、缓存、外部 API） | 业务逻辑 |
| `internal/model/` | 数据模型定义（struct、常量） | 业务逻辑 |
| `internal/svc/` | DI 容器，依赖组装 | 业务逻辑 |
| `internal/infra/` | 跨层基础设施（中间件、工具、配置） | 业务逻辑 |
| `pkg/` | 可跨项目复用的包 | 项目特有逻辑 |
| `etc/` | 配置文件 | 代码 |
| `scripts/` | 构建、部署、开发脚本 | 代码 |
| `bin/` | 构建产物 | 源码 |

---

## 3. `internal/` 组织规范

### 当前：按技术层划分

```
internal/
├── handler/       # 所有 HTTP/RPC handler
├── usecase/       # 所有业务用例
├── repo/          # 所有数据访问
├── model/         # 所有数据模型
└── svc/           # DI 容器
```

**优点**：

- 职责清晰，同层代码放在一起
- 新人容易理解分层架构
- 适合中小规模项目

**约定**：

- 同一领域的代码在不同层使用相同的包名或文件名
- 例如：`handler/user.go`、`usecase/user.go`、`repo/user.go` 都是用户相关

### 可选演进：按领域划分

当项目规模增大时，可以考虑按领域重组：

```
internal/
├── user/
│   ├── handler.go      # 用户 handler
│   ├── usecase.go      # 用户业务逻辑
│   ├── repo.go         # 用户数据访问
│   ├── model.go        # 用户模型
│   └── errors.go       # 用户相关错误
├── room/
│   ├── handler.go
│   ├── usecase.go
│   ├── repo.go
│   └── model.go
└── common/             # 跨领域共享
    ├── svc/
    └── infra/
```

**迁移策略**：

- 不强制立即迁移，按需渐进
- 新增领域优先考虑领域驱动组织
- 现有代码可在重构时逐步迁移

---

## 4. `pkg/` 使用边界

### 什么该放 `pkg/`

- **真正可跨项目复用**：如 `logger`、`protocol`、`centrifuge-plus`
- **独立的工具库**：不依赖项目业务逻辑
- **对外提供的 SDK**：如客户端库

### 什么不该放 `pkg/`

- 项目特有的业务逻辑 → 放 `internal/`
- 不确定是否复用 → 先放 `internal/`，需要时再提取
- 临时工具 → 放 `scripts/` 或删除

### 判断标准

> 如果另一个项目可以直接 import 并使用，放 `pkg/`。
> 如果依赖项目的 `internal/`，则不是 `pkg/` 的候选。

---

## 5. 依赖注入

### 构造函数注入

所有依赖通过构造函数参数传入，不使用全局变量。

```go
// ✅ 构造函数注入
type UserUseCase struct {
    repo   UserRepository
    logger *logger.Logger
}

func NewUserUseCase(repo UserRepository, logger *logger.Logger) *UserUseCase {
    return &UserUseCase{repo: repo, logger: logger}
}

// ❌ 全局变量
var userRepo UserRepository

func GetUser(id string) (*User, error) {
    return userRepo.Find(id)  // 隐式依赖
}
```

### 在 `svc/` 中组装

```go
// main.go
func main() {
    cfg := config.Load()
    svc := svc.NewServiceContext(cfg)

    // 启动服务
    server.Run(svc)
}

// internal/svc/context.go
func NewServiceContext(cfg *config.Config) *ServiceContext {
    db := initDB(cfg.Database)
    logger := initLogger(cfg.Log)

    userRepo := repo.NewUserRepo(db)
    roomRepo := repo.NewRoomRepo(db)

    return &ServiceContext{
        Config:   cfg,
        Logger:   logger,
        UserRepo: userRepo,
        RoomRepo: roomRepo,
        UserUC:   usecase.NewUserUseCase(userRepo, logger),
        RoomUC:   usecase.NewRoomUseCase(roomRepo, logger),
    }
}
```

### 接口依赖

上层依赖下层的接口，而非具体实现。

```go
// internal/usecase/user.go
type UserRepository interface {
    Find(ctx context.Context, id string) (*model.User, error)
    Save(ctx context.Context, user *model.User) error
}

type UserUseCase struct {
    repo UserRepository  // 依赖接口
}
```

---

## 6. 配置管理

### 配置文件组织

```
etc/
├── dev/
│   ├── config.yaml      # 开发环境主配置
│   └── secrets.yaml     # 开发环境敏感配置（不入 git）
├── staging/
│   └── config.yaml
└── production/
    └── config.yaml
```

### 配置结构体

```go
// internal/infra/config/config.go
type Config struct {
    Server   ServerConfig   `yaml:"server"`
    Database DatabaseConfig `yaml:"database"`
    Log      LogConfig      `yaml:"log"`
}

type ServerConfig struct {
    Port    int    `yaml:"port"`
    Host    string `yaml:"host"`
}

type DatabaseConfig struct {
    DSN          string `yaml:"dsn"`
    MaxOpenConns int    `yaml:"maxOpenConns"`
}
```

### 环境变量覆盖

环境变量优先级高于配置文件，适合敏感信息和部署时的动态配置。

```go
// 环境变量覆盖配置文件
if dsn := os.Getenv("DATABASE_DSN"); dsn != "" {
    cfg.Database.DSN = dsn
}
```

### 敏感信息

- **不入 git**：`secrets.yaml` 加入 `.gitignore`
- **环境变量**：生产环境通过环境变量注入
- **不硬编码**：密码、token、API key 绝不写死在代码中

```go
// ❌ 硬编码
const apiKey = "sk-1234567890abcdef"

// ✅ 从配置读取
apiKey := cfg.Anthropic.APIKey
```

---

## 总结

清晰的项目结构是团队协作的基础。遵循这些规范：

- **目录职责明确**：每个目录只放该放的东西
- **分层清晰**：handler → usecase → repo，依赖单向流动
- **依赖注入**：通过构造函数，不使用全局变量
- **配置外置**：敏感信息不入代码，不入 git
- **渐进演进**：从分层架构起步，按需向领域驱动演进

> _"好的项目结构让代码自己说话，无需解释就能理解。"_
