# 测试规范

> _没有测试的代码是未经证实的假设。测试不是写完代码后的附加任务，而是设计过程的一部分。_

---

## 1. 测试金字塔

测试分层是资源分配的指南——底层便宜且快速，顶层昂贵且缓慢。

```
        ╱  E2E  ╲          少量：核心用户流程
       ╱─────────╲
      ╱ 集成测试  ╲        适量：模块间协作、外部依赖
     ╱─────────────╲
    ╱   单元测试    ╲      大量：纯逻辑、边界条件
   ╱─────────────────╲
```

| 层级 | 关注点 | 速度 | 数量 |
|------|--------|------|------|
| **单元测试** | 单个函数/方法的正确性 | 毫秒级 | 最多 |
| **集成测试** | 模块间交互、数据库、外部服务 | 秒级 | 适量 |
| **E2E 测试** | 完整用户流程 | 分钟级 | 最少 |

### 什么该测

- **业务逻辑**：核心算法、状态转换、数据处理
- **边界条件**：空值、零值、极值、特殊字符
- **错误路径**：错误返回、超时、降级行为
- **并发安全**：竞态条件、死锁场景

### 什么不该测

- **框架内部行为**：不要测 React 是否渲染、ORM 是否生成 SQL
- **Getter/Setter**：没有逻辑的纯数据访问不需要测试
- **第三方库**：测试你如何使用它，而非它本身

---

## 2. 测试命名

测试名是活文档。当测试失败时，名字应当告诉你：哪个功能、什么场景、期望什么结果。

### Go 命名模板

```go
func TestGetUser_NotFound_ReturnsError(t *testing.T) { ... }
func TestCreateOrder_InsufficientStock_ReturnsErrStockInsufficient(t *testing.T) { ... }
func TestParseConfig_InvalidJSON_ReturnsParseError(t *testing.T) { ... }
```

格式：`Test<被测函数>_<场景>_<期望结果>`

### TypeScript 命名模板

```typescript
describe('UserService', () => {
  describe('getUser', () => {
    it('returns user when id exists', () => { ... })
    it('throws NotFoundError when id does not exist', () => { ... })
    it('excludes soft-deleted users from results', () => { ... })
  })
})
```

格式：`it('<动词> <期望行为> when <条件>')`

### 反模式

```
❌ TestGetUser(t *testing.T)            — 什么场景？期望什么？
❌ it('works correctly')                 — 什么是"正确"？
❌ Test1(t *testing.T)                  — 毫无意义
❌ it('should return true')             — 什么应该返回 true？
```

---

## 3. 测试组织

### 文件位置

测试文件与被测文件同级或紧邻。

```
server/
├── internal/
│   ├── user/
│   │   ├── service.go
│   │   ├── service_test.go          ← 同级
│   │   ├── repository.go
│   │   └── repository_test.go
│   └── order/
│       ├── service.go
│       └── service_test.go

web-components/
├── packages/
│   ├── chat-panel/
│   │   ├── src/
│   │   │   ├── ChatPanel.ts
│   │   │   └── ChatPanel.test.ts    ← 同级
│   │   └── package.json
```

### 测试目录

仅当测试需要辅助文件（fixtures、helpers、mocks）时，使用 `testdata/` 或 `__tests__/` 目录：

```go
server/internal/user/
├── service.go
├── service_test.go
└── testdata/
    ├── valid_user.json
    └── invalid_user.json
```

```typescript
web-components/packages/chat-panel/
├── src/
│   ├── ChatPanel.ts
│   ├── ChatPanel.test.ts
│   └── __fixtures__/
│       └── messages.ts
```

---

## 4. 测试编写原则

### 单一职责

每个测试只验证一件事。一个测试失败时，应当能立即定位问题，而非在多个断言中寻找线索。

```go
// ❌ 一个测试验证多件事
func TestCreateUser(t *testing.T) {
    user, err := CreateUser("alice")
    require.NoError(t, err)
    assert.Equal(t, "alice", user.Name)
    assert.NotEmpty(t, user.ID)
    assert.True(t, user.CreatedAt.Before(time.Now()))
    // 哪个断言失败了？
}

// ✅ 每个测试验证一件事
func TestCreateUser_ValidInput_ReturnsUser(t *testing.T) {
    user, err := CreateUser("alice")
    require.NoError(t, err)
    assert.Equal(t, "alice", user.Name)
}

func TestCreateUser_ValidInput_AssignsID(t *testing.T) {
    user, err := CreateUser("alice")
    require.NoError(t, err)
    assert.NotEmpty(t, user.ID)
}

func TestCreateUser_ValidInput_SetsCreatedAt(t *testing.T) {
    user, err := CreateUser("alice")
    require.NoError(t, err)
    assert.True(t, user.CreatedAt.Before(time.Now()))
}
```

### 测试独立性

测试之间不共享状态。每个测试独立设置环境、执行操作、验证结果。

```go
// ❌ 依赖执行顺序
var globalDB *sql.DB

func TestA(t *testing.T) {
    globalDB = setupDB()
    // ...
}

func TestB(t *testing.T) {
    // 依赖 TestA 设置的 globalDB
    user, _ := globalDB.GetUser("alice")
    // ...
}

// ✅ 每个测试独立设置
func TestA(t *testing.T) {
    db := setupDB(t)
    t.Cleanup(func() { db.Close() })
    // ...
}

func TestB(t *testing.T) {
    db := setupDB(t)
    t.Cleanup(func() { db.Close() })
    // ...
}
```

### 表驱动测试（Go）

当同一逻辑需要测试多个场景时，表驱动测试减少重复、提高可读性。

```go
func TestParseSize(t *testing.T) {
    tests := []struct {
        name    string
        input   string
        want    int64
        wantErr bool
    }{
        {name: "bytes", input: "100B", want: 100},
        {name: "kilobytes", input: "10KB", want: 10240},
        {name: "megabytes", input: "5MB", want: 5242880},
        {name: "invalid format", input: "abc", wantErr: true},
        {name: "empty string", input: "", wantErr: true},
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            got, err := ParseSize(tt.input)
            if tt.wantErr {
                assert.Error(t, err)
            } else {
                require.NoError(t, err)
                assert.Equal(t, tt.want, got)
            }
        })
    }
}
```

### 参数化测试（TypeScript）

```typescript
describe('parseSize', () => {
  const cases = [
    { input: '100B', expected: 100, description: 'bytes' },
    { input: '10KB', expected: 10240, description: 'kilobytes' },
    { input: '5MB', expected: 5242880, description: 'megabytes' },
  ]

  cases.forEach(({ input, expected, description }) => {
    it(`parses ${description}: ${input}`, () => {
      expect(parseSize(input)).toBe(expected)
    })
  })

  it('throws on invalid format', () => {
    expect(() => parseSize('abc')).toThrow()
  })
})
```

---

## 5. Mock 与 Stub 的使用边界

### 何时 Mock

- **外部服务**：HTTP 调用、数据库、消息队列
- **副作用**：文件系统、时间、随机数
- **难以触达的状态**：错误路径、超时、边界条件

### 何时不用 Mock

- **纯逻辑**：没有外部依赖的函数直接测试
- **数据结构**：序列化/反序列化用真实数据
- **简单转换**：不值得为 `add(1, 2)` 写 mock

### 过度 Mock 的反模式

```go
// ❌ Mock 一切，测试变成实现细节的奴隶
func TestGetUser(t *testing.T) {
    mockRepo := NewMockUserRepo()
    mockRepo.On("Find", "123").Return(&User{ID: "123", Name: "alice"}, nil)
    mockCache := NewMockCache()
    mockCache.On("Get", "user:123").Return(nil, ErrNotFound)
    mockCache.On("Set", "user:123", mock.Anything).Return(nil)

    service := NewUserService(mockRepo, mockCache)
    user, err := service.GetUser("123")

    // 测试的是 mock 的调用，而非真实行为
    mockRepo.AssertCalled(t, "Find", "123")
    mockCache.AssertCalled(t, "Set", "user:123", mock.Anything)
}

// ✅ Mock 边界，测试行为
func TestGetUser_CacheMiss_FetchesFromRepo(t *testing.T) {
    repo := &stubRepo{users: map[string]*User{"123": {ID: "123", Name: "alice"}}}
    cache := &stubCache{store: make(map[string]any)}
    service := NewUserService(repo, cache)

    user, err := service.GetUser("123")

    require.NoError(t, err)
    assert.Equal(t, "alice", user.Name)
}
```

### 接口嵌入 Stub 模式

当需要 stub 一个大型接口（5+ 方法）但只测试其中 1-2 个方法的行为时，通过嵌入接口本身来避免实现所有方法。未覆盖的方法在调用时会 panic（零值方法），这恰好能暴露测试中对未预期调用的遗漏。

```go
// ✅ 嵌入接口：只覆盖需要的方法
type noopBackend struct {
    rtcoss3.Backend  // 嵌入接口，未实现的方法调用时会 panic
}

// 只覆盖测试需要的方法
func (noopBackend) PutObject(ctx context.Context, key string, r io.Reader) error {
    return nil
}

func (noopBackend) DeleteObject(ctx context.Context, key string) error {
    return nil
}

// 测试中使用
uc := NewOSS3Usecase(deps)
uc.backend = noopBackend{}  // 其他方法调用会 panic，暴露意外依赖

// ❌ 实现所有方法：大量无意义的空实现
type fullBackend struct{}
func (fullBackend) PutObject(...) error { return nil }
func (fullBackend) GetObject(...) error { return nil }
func (fullBackend) DeleteObject(...) error { return nil }
// ... 还有 15 个无关方法
```

**适用场景**：

- 接口方法多（5+）但测试只关注少数方法
- 集成测试中需要 stub 外部依赖（如后端存储服务、消息队列）
- 配合 `t.Helper()` 提取为 `newTestXxx()` 函数复用

**复用模式**：当多个测试需要相同的基础 stub 时，用 `t.Helper()` 封装构造函数，确保失败时指向调用方行号。

```go
// ✅ 封装为 t.Helper() 复用的构造函数
func newTestUsecase(t *testing.T, backend rtcoss3.Backend) *OSS3Usecase {
    t.Helper()
    deps := &OSS3Deps{
        Backend: backend,
        Logger:  zap.NewNop(),
        Metrics: noopMetrics(),
    }
    return NewOSS3Usecase(deps)
}

func TestPutObject_QuotaExceeded_ReturnsError(t *testing.T) {
    uc := newTestUsecase(t, noopBackend{})
    // ... 测试逻辑
}

func TestDeleteObject_Success(t *testing.T) {
    uc := newTestUsecase(t, noopBackend{})
    // ... 测试逻辑
}
```

---

## 6. 覆盖率目标

覆盖率是指标，不是目标。追求 100% 覆盖率会导致测试质量下降——为覆盖率而写的测试往往验证实现细节，而非行为。

### 目标

| 仓库 | 目标 | 说明 |
|------|------|------|
| **server** | 核心逻辑 ≥ 80% | `internal/` 包的业务逻辑、错误处理路径 |
| **web-components** | 核心组件 ≥ 70% | 状态管理、事件处理、边界条件 |
| **docs** | N/A | OpenAPI 文档不适用传统测试覆盖率 |
| **mermaid-live-editor** | 核心功能 ≥ 60% | 解析、渲染、状态同步 |

### 关键路径 100%

以下路径必须有测试覆盖：

- **认证与授权**：登录、权限检查、token 验证
- **数据处理**：支付、订单、用户数据变更
- **并发安全**：共享状态、锁、channel
- **错误恢复**：panic recovery、超时处理、降级逻辑

### 不追求数字游戏

- 不为覆盖率而测试 getter/setter
- 不为覆盖率而测试框架行为
- 覆盖率工具会漏掉逻辑分支——人工判断比数字更重要

---

## 7. 性能测试

### 阈值必须包含 CI 裕量

性能测试（benchmark）的阈值不能基于本地机器的最佳表现设定。CI 环境、开发机、不同 OS 的负载差异可达 50-100%。阈值必须在"典型耗时"基础上留出足够裕量，否则测试会在 CI 中频繁 flaky。

```typescript
// ✅ 阈值留裕量 + 注释说明理由
it('benchmark: batch update 100 items', async () => {
  const elapsed = await measureBatchUpdate(100)
  // 典型耗时 ~200ms，阈值 600ms 留 3x 裕量覆盖 CI/dev 负载差异
  expect(elapsed).toBeLessThan(600)
})

// ❌ 阈值紧贴最佳表现，CI 必 flaky
expect(elapsed).toBeLessThan(250)  // 本地跑 200ms，CI 跑 300ms → 失败
```

### 性能测试的隔离原则

- 性能测试与功能测试分离（独立文件或独立 `describe` 块）
- 不在 CI 的默认测试套件中运行性能测试（用 `--tag` 或单独脚本）
- 性能测试失败时只 warn 不 block（避免环境波动阻塞发布）

```typescript
// ✅ 使用 describe 隔离 + tag 控制运行
describe('performance benchmarks', () => {
  it.skipIf(!process.env.RUN_BENCHMARKS)('batch update perf', async () => {
    // ...
  })
})

// 运行：RUN_BENCHMARKS=1 pnpm test
// 默认：跳过性能测试
```

### 相对优于绝对

优先比较"两个实现的相对性能"而非"绝对时间阈值"。相对比较不受环境波动影响。

```typescript
// ✅ 比较相对性能：batch vs single
it('benchmark: batch is faster than single for 100 items', async () => {
  const batchTime = await measureBatch(100)
  const singleTime = await measureSingle(100)
  expect(batchTime).toBeLessThan(singleTime * 0.5)  // batch 至少快 2 倍
})

// ⚠️ 绝对阈值仍有用：防止性能退化超过可接受范围
expect(batchTime).toBeLessThan(600)  // 安全网
```

---

## 8. 集成测试跳过约定

集成测试依赖外部服务（Redis、Docker Compose、真实 S3 等）时，必须在缺少依赖的环境下优雅跳过，而非失败。

### `t.Skip()` 附带原因

```go
// ✅ 跳过时说明原因和恢复方式
func TestCreateBucket_Integration(t *testing.T) {
    if os.Getenv("REDIS_URL") == "" {
        t.Skip("Integration test requires Redis - use Docker Compose for full testing")
    }
    // ... 测试逻辑
}

// ✅ 支持 testing.Short() 模式
func TestFullFlow_Integration(t *testing.T) {
    if testing.Short() {
        t.Skip("skipping integration test in short mode")
    }
    // ...
}

// ✅ 条件性跳过：凭据未配置时跳过
func TestPresignedURL_SigV4(t *testing.T) {
    token := os.Getenv("RTC_OSS3_REFRESH_TOKEN")
    if token == "" {
        t.Skip("RTC_OSS3_REFRESH_TOKEN not set")
    }
    // ...
}

// ❌ 不跳过，直接失败
func TestCreateBucket_Integration(t *testing.T) {
    conn := connectToRedis()  // panic: connection refused
}
```

### 约束

- `t.Skip()` 必须附带消息说明**需要什么依赖**以及**如何满足**（如"use Docker Compose"）
- 集成测试文件名使用 `_integration_test.go` 后缀，便于与单元测试区分
- 集成测试应在文件顶部或首个测试函数中集中检查依赖，不在每个测试函数中重复检查
- `go test ./...`（CI 默认）应能正常运行——所有需要外部服务的测试跳过，不报错

### 测试文件命名分类

当模块规模较大（10+ 文件）时，测试文件按职责进一步拆分为专用后缀，便于定位不同类型的测试。

| 后缀 | 用途 | 运行条件 |
| --- | --- | --- |
| `_test.go` | 单元测试，纯逻辑验证 | `go test ./...` 始终运行 |
| `_edge_test.go` | 边界条件、错误路径、协议违规输入 | `go test ./...` 始终运行 |
| `_bugfix_test.go` | 回归测试：针对特定 bug 的修复验证（注释标注 bug 编号） | `go test ./...` 始终运行 |
| `_integration_test.go` | 依赖外部服务（Redis、Docker、S3） | 缺依赖时自动跳过 |
| `_test_helpers_test.go` | 测试辅助函数和共享 fixtures | 被其他测试文件引用 |

```go
// ✅ 大模块测试文件按职责拆分
internal/handler/http/
├── oss3.go
├── oss3_test.go               # 核心逻辑单元测试
├── oss3_edge_test.go          # 边界条件：空 key、超长 path、无效签名
├── oss3_bugfix_test.go        # 回归测试：代码审查发现的特定 bug 修复
├── oss3_integration_test.go   # 完整 S3 协议流程（需 MinIO）
└── oss3_test_helpers_test.go  # 共享 mock、fixtures、assert helpers

// ❌ 所有测试混在一个文件
internal/handler/http/
├── oss3.go
└── oss3_test.go  # 3000+ 行，混合单元/集成/边界测试
```

**约束**：

- `_edge_test.go` 中的测试不依赖外部服务（与集成测试的区别），只测试输入边界和错误路径
- `_bugfix_test.go` 中的每个测试函数必须用注释标注对应的 bug 编号或描述（如 `// Bug #10: Path traversal prevention`），便于追溯修复上下文
- 拆分阈值：模块测试文件超过 500 行时考虑按职责拆分
- `_test_helpers_test.go` 中的函数使用 `t.Helper()` 确保失败时指向调用方行号

### 运行约定

```bash
go test ./...                        # 跳过集成测试（无外部依赖）
go test -short ./...                 # 显式跳过集成测试
docker compose up -d && go test ./... # 本地完整测试（含集成测试）
```

---

## 9. 各仓库特殊约定

### server (Go)

**测试框架**：

- 标准库 `testing` 包 + `testify`（`assert` / `require`）
- 测试替身：优先使用手写 stub 和接口嵌入（参见 [Section 5 - 接口嵌入 Stub 模式](#接口嵌入-stub-模式)），不依赖 `mockery`/`gomock` 等代码生成工具

**并发测试**：

```go
func TestConcurrentAccess(t *testing.T) {
    var wg sync.WaitGroup
    for i := 0; i < 100; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            // 操作共享资源
        }()
    }
    wg.Wait()
}
```

**并行测试（`t.Parallel()`）**：

独立测试（不依赖共享数据库状态、全局变量）应使用 `t.Parallel()` 并行执行，加速测试套件并帮助发现隐藏的竞态条件。

```go
func TestGetUser_ValidInput_ReturnsUser(t *testing.T) {
    t.Parallel()  // 独立测试，可并行
    // ...
}

func TestGetUser_NotFound_ReturnsError(t *testing.T) {
    t.Parallel()  // 独立测试，可并行
    // ...
}
```

**不适用 `t.Parallel()` 的场景**：

- 测试依赖共享数据库（同一行记录的读写）
- 测试修改全局状态（环境变量、全局配置）
- 测试依赖外部资源（文件锁、端口占用）

**判断标准**：测试是否能在不与其他测试交互的情况下独立运行？如果能，加 `t.Parallel()`。

**运行测试**：

```bash
go test ./...                          # 全部（集成测试依赖外部服务时自动跳过）
go test -short ./...                   # 显式跳过集成测试
go test -race ./...                    # 竞态检测
go test -coverprofile=coverage.out ./... && go tool cover -html=coverage.out
```

### web-components (TypeScript)

**测试框架**：

- `vitest`（推荐）或 `jest`
- DOM 测试：`@testing-library/dom` 或 `happy-dom`

**组件测试**：

```typescript
import { describe, it, expect } from 'vitest'
import { ChatPanel } from './ChatPanel'

describe('ChatPanel', () => {
  it('renders message list', () => {
    const panel = new ChatPanel()
    panel.messages = [{ id: '1', text: 'hello' }]
    panel.connectedCallback()

    const list = panel.shadowRoot!.querySelector('.message-list')
    expect(list?.children.length).toBe(1)
  })

  it('emits message-sent event', async () => {
    const panel = new ChatPanel()
    const handler = vi.fn()
    panel.addEventListener('message-sent', handler)

    panel.sendMessage('hello')

    expect(handler).toHaveBeenCalledOnce()
  })
})
```

**运行测试**：

```bash
pnpm test                        # 全部
pnpm test --coverage             # 覆盖率
pnpm test --watch                # 监听模式
```

---

## 总结

测试是代码的契约。好的测试应当：

- **快速**：毫秒级反馈，鼓励频繁运行
- **独立**：不依赖执行顺序，不共享状态
- **可读**：名字即文档，失败时立即定位问题
- **可靠**：不 flaky，不依赖外部环境
- **有意义**：测试行为，而非实现细节

> _"测试不是证明代码正确，而是证明代码还没被证明错误。"_
