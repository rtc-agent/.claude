# Go 语言规范

> _适用于 `server` 仓库的所有 Go 代码。_

---

## 概述

本规范是 Go 代码的编写指南，覆盖代码风格、错误处理、并发模式和项目结构四个维度。目标是让 `server` 仓库的 Go 代码风格统一、意图清晰、易于维护。

---

## 快速参考

| 领域 | 核心规则 |
|------|---------|
| **格式** | `gofmt` 格式化，`golangci-lint` 检查，不接受风格争论 |
| **命名** | 导出用 PascalCase，非导出用 camelCase；包名短小写 |
| **错误** | 不吞错误；在边界处理，在中间传播；用 `%w` 包装 |
| **并发** | goroutine 必须有退出机制；用 context 传递取消信号 |
| **接口** | 接受接口，返回结构体；小接口优于大接口 |
| **测试** | 表驱动测试；竞态检测 `-race`；mock 边界而非一切 |

---

## 核心原则

### 接受接口，返回结构体

函数参数用接口，返回值用具体类型。这让调用方获得完整的类型信息，同时保持参数的灵活性。

```go
// ✅ 接受接口，返回结构体
func NewUserService(repo UserRepository) *UserService {
    return &UserService{repo: repo}
}

// ❌ 返回接口，调用方需要类型断言
func NewUserService(repo UserRepository) UserService {
    return &userServiceImpl{repo: repo}
}
```

### 错误是值，不是异常

Go 的错误是一等公民。显式处理每个错误，不要用 panic 替代错误返回。

```go
// ✅ 显式处理错误
user, err := repo.GetUser(ctx, id)
if err != nil {
    return fmt.Errorf("get user %s: %w", id, err)
}

// ❌ panic 替代错误
if err != nil {
    panic(err)
}
```

### 零值必须有意义

结构体的零值（`var s MyStruct`）应当是可用的、安全的。不需要额外的初始化就能正常工作。

```go
// ✅ 零值可用
type Counter struct {
    mu    sync.Mutex
    count int64     // 零值 0 是合理的初始值
}

// ❌ 零值不可用，必须通过构造函数初始化
type DB struct {
    conn *sql.DB    // 零值 nil，使用前必须初始化
}
```

### 清晰胜于聪明

选择最朴素的实现。如果代码需要注释解释"巧妙"之处，那它还不够好。

---

## 子文档导航

| 文档 | 内容 |
|------|------|
| [代码风格](./code-style.md) | `gofmt`、命名约定、包组织、导入分组、注释格式 |
| [错误处理](./error-handling.md) | 错误传播模式、自定义错误类型、`%w` 包装、panic 边界 |
| [并发规范](./concurrency.md) | goroutine 生命周期、panic 拦截、channel 模式、context 使用、sync 原语 |
| [项目结构](./project-structure.md) | 目录布局、模块划分、依赖注入、配置管理 |
| [可观测性](./observability.md) | 结构化日志、分布式追踪、指标采集、健康检查 |
| [代码质量](./code-quality.md) | 重复代码、废弃代码、事务上下文、Redis Lua、PR 自检清单 |
| [后端性能](./performance.md) | 数据库优化、并发、缓存、Centrifuge、pprof |

---

## 工具链

| 工具 | 用途 | 命令 |
|------|------|------|
| `gofmt` / `goimports` | 代码格式化 | `gofmt -w .` |
| `go vet` | 静态分析 | `go vet ./...` |
| `golangci-lint` | 综合 lint | `golangci-lint run ./...` |
| `go test -race` | 竞态检测 | `go test -race ./...` |
| `mockery` | Mock 生成 | `mockery --name=UserRepository` |
| `go tool cover` | 覆盖率 | `go test -coverprofile=coverage.out ./...` |

所有工具应在 pre-commit hook 中自动执行，不在本地检查通过不应提交。
