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
│   ├── agent/               # LLM Agent 核心（消息转换、文件加载、工具注册、消息规范化）
│   ├── handler/                 # 协议适配层
│   │   ├── http/                # HTTP API（含 oss3*.go S3 兼容层）
│   │   └── rpc/                 # RPC 接口
│   ├── usecase/                 # 业务逻辑层
│   │   └── primitives/          # 跨层校验原语（格式校验 + 标识符构造 + 存在性检查）
│   ├── repo/                    # 数据访问层
│   ├── model/                   # 数据模型
│   ├── svc/                     # Service Context（DI 容器）
│   └── infra/                   # 基础设施
│       ├── middleware/          # 中间件
│       ├── httputil/            # HTTP 工具
│       ├── contextx/            # Context key 集中定义（跨层共享）
│       ├── cache/               # 缓存
│       ├── config/              # 配置加载
│       └── auth/                # 认证
├── pkg/                         # 可复用包
│   ├── centrifuge-plus/         # Centrifuge 扩展
│   ├── circuitbreaker/          # 断路器
│   ├── logger/                  # 日志（含 SafeGo 并发安全启动）
│   ├── memory/                  # 内存管理工具
│   ├── protocol/                # 协议定义
│   ├── proxy/                   # 代理服务
│   ├── rtc-oss3/                # S3 兼容存储后端（SigV4、OSS 操作）
│   ├── rtc-queue/               # RTC 队列
│   ├── turn-agent/              # TURN agent（Eino turn loop）
│   ├── webfetch/                # Web 内容抓取
│   └── websearch/               # Web 搜索
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
| `internal/agent/` | LLM Agent 核心：消息转换、文件加载、工具注册、消息规范化、目标工作流 | HTTP/RPC 细节、SQL |
| `internal/handler/` | 协议适配（HTTP/RPC），参数校验，响应格式化 | 业务逻辑 |
| `internal/usecase/` | 业务逻辑编排，事务管理 | HTTP/RPC 细节、SQL |
| `internal/usecase/primitives/` | 跨层校验原语：格式校验 + 标识符构造 + 存在性检查，供 handler/usecase 调用 | 业务编排、HTTP/RPC 细节 |
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

### 大型包的子包拆分

当某个 `internal/` 包增长到 50+ 文件时（如 `internal/agent/` 有 80+ 文件），可以将辅助工具提取为子包。子包只包含可独立理解的工具代码，不包含业务逻辑。

```
internal/agent/
├── agent.go                    # 主入口
├── data_context.go             # 数据加载
├── data_context_convert.go     # 消息转换
├── file_loader.go              # 文件加载（图片预处理、文本截断）
├── message_normalizer.go       # 消息规范化
├── tools.go                    # 工具注册
├── tools_*.go                  # 各工具实现（30+ 文件）
├── command/                    # 子包：命令解析工具
│   ├── command.go
│   ├── registry.go
│   └── template.go
├── prompts/                    # 子目录：提示词模板（.md / .md.tmpl 文件）
├── stringutil/                 # 子包：字符串工具
│   └── stringutil.go
├── templateutil/               # 子包：模板工具
│   └── templateutil.go
└── util/                       # 子包：通用工具（map 操作等）
    └── maps.go
```

**约束**：

- 子包不导入父包（避免循环依赖）
- 子包只包含无状态的工具函数或纯数据类型，不包含业务逻辑
- 子包命名用小写单词，与父包名无前缀关系（`command/` 而非 `agentcommand/`）
- 业务逻辑仍留在父包，子包只提供"可独立测试和复用"的基础能力

### 大型 handler 模块的操作拆分

当一个协议兼容层模块（如 OSS3 S3 handler）增长到 1000+ 行时，按操作拆分为多个文件。每个文件负责一个 S3/HTTP 操作，共享中间件和辅助函数通过同包的 `utils`/`middleware` 文件提供。

```
internal/handler/http/
├── oss3.go                    # 路由注册、共享 handler 结构
├── oss3_middleware.go         # 认证、日志、指标等中间件
├── oss3_get.go                # GetObject
├── oss3_put.go                # PutObject
├── oss3_delete.go             # DeleteObject
├── oss3_copy.go               # CopyObject
├── oss3_list.go               # ListObjects / ListObjectsV2
├── oss3_multipart.go          # 多部分上传协调
├── oss3_mp_upload.go          # UploadPart
├── oss3_mp_complete.go        # CompleteMultipartUpload
├── oss3_mp_abort_list.go      # Abort/List multipart uploads
├── oss3_cors.go               # CORS 配置
├── oss3_sigv4.go              # SigV4 签名验证
└── oss3_test_helpers_test.go  # 共享测试 fixtures
```

**约束**：

- 每个操作文件保持在 500 行以内，遵循通用大文件约束
- 共享类型（如 `OSS3Handler` struct、中间件）放在主文件（如 `oss3.go`）或专用文件中
- 测试文件按职责拆分（参见 [测试规范 - 测试文件命名分类](../02-testing-standards.md#测试文件命名分类)）
- 新增操作时，优先创建独立文件而非追加到已有文件

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

## 3a. 跨层基础设施约定

### Context key 集中管理（`contextx/`）

所有跨层传递的 context key 集中在 `internal/infra/contextx/` 定义，提供类型安全的 `Get*` / `With*` 访问器对。业务代码不直接使用 `context.WithValue` + 裸 key。

```go
// internal/infra/contextx/keys.go
type contextKey struct{ name string }  // 未导出 struct，防止外部碰撞

var (
    userIDKey   = contextKey{"user_id"}
    deviceIDKey = contextKey{"device_id"}
)

// 类型安全的读取
func GetUserID(ctx context.Context) (uuid.UUID, bool) {
    id, ok := ctx.Value(userIDKey).(uuid.UUID)
    return id, ok
}

// 类型安全的注入
func WithClientInfo(ctx context.Context, userID uuid.UUID, deviceID string) context.Context {
    ctx = context.WithValue(ctx, userIDKey, userID)
    ctx = context.WithValue(ctx, deviceIDKey, deviceID)
    return ctx
}
```

**约束**：

- 新增 context key 时，在 `contextx/` 中注册，不分散定义
- 使用未导出 struct 类型作为 key（防止其他包创建冲突 key）
- 按子系统分组命名（如 OSS3 相关 key 加 `oss3` 前缀）
- 复合数据存为 struct 类型（如 `OSS3ParsedPath{Bucket, Key}`），不在 context 中散落多个相关值
- 当子系统 context key 达到 5 个以上时，提取到独立文件 `keys_<subsystem>.go`（如 `keys_oss3.go`），保持 `keys.go` 只包含跨子系统通用的 key

### Redis key 集中注册（`cache/keys.go`）

所有 Redis key 的 prefix 常量和构造函数集中在 `internal/infra/cache/keys.go` 注册。业务代码不硬编码 key 字符串。

```go
// cache/keys.go — 集中注册
const PrefixOSS3Quota = "oss3:quota:"

func OSS3Quota(userID string) string { return PrefixOSS3Quota + userID }
// 注释记录完整 key 格式、值类型、TTL
// Full key: oss3:quota:{user_id}
// Value: total bytes used (int64); no TTL (persistent, reconciled by cleanup).
```

**约束**：

- 新增 key 类型必须在 `keys.go` 注册（便于全局搜索和冲突检测）
- 每个 prefix 必须有对应的构造函数
- 注释中记录 key 格式、值类型、TTL

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
// internal/infra/config/config.go — 类型定义
type Config struct {
    Server    ServerConfig    `mapstructure:"server"`
    Database  DatabaseConfig  `mapstructure:"database"`
    // ...
}

type ServerConfig struct {
    Port    int    `mapstructure:"port"`
    Host    string `mapstructure:"host"`
}
```

### 三文件组织

配置模块拆分为三个文件，各司其职：

| 文件 | 职责 | 内容 |
|------|------|------|
| `config.go` | 类型定义 + 加载逻辑 | `Config` struct、`Load()` 函数 |
| `config_defaults.go` | 默认值注册 | `setDefaults()` + 各模块 `set*Defaults()` |
| `config_validation.go` | 校验逻辑 | `Config.Validate()` + 各子 struct 的 `Validate()` |

```go
// config_defaults.go — 按模块注册默认值
func setDefaults(v *viper.Viper) {
    setServerDefaults(v)
    setAuthDefaults(v)
    setCORSDefaults(v)
    // ...
}

func setServerDefaults(v *viper.Viper) {
    v.SetDefault("server.port", 8080)
    v.SetDefault("server.host", "0.0.0.0")
}

// config_validation.go — 每个子 struct 独立校验
func (c *Config) Validate() error {
    if c.Database.DSN == "" {
        return fmt.Errorf("database.dsn is required")
    }
    if err := c.Storage.Validate(); err != nil {
        return err
    }
    // ...
}

func (c *StorageConfig) Validate() error { /* ... */ }
func (c *AsynqConfig) Validate() error   { /* ... */ }
```

**约束**：

- 新增配置项时，同时在 `config_defaults.go` 注册默认值、在 `config_validation.go` 添加校验（如需要）
- 默认值使用 `viper.SetDefault()`，不使用 struct tag 的 `default`
- 校验函数返回第一个发现的错误（fail-fast），不收集所有错误
- 使用局部 `viper.New()` 实例，不污染全局状态——保证并行测试安全

### 环境变量覆盖

环境变量优先级高于配置文件。使用 `__` 双下划线映射嵌套字段：

```go
// 环境变量映射：DATABASE__DSN -> database.dsn
v.SetEnvKeyReplacer(strings.NewReplacer(".", "__"))
```

敏感字段支持 `${VAR_NAME}` 展开，通过 `expandEnvVars()` 在加载后处理。

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
