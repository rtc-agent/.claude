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

## 7. 各仓库特殊约定

### server (Go)

**测试框架**：

- 标准库 `testing` 包 + `testify`（`assert` / `require`）
- Mock 生成：`mockery` 或 `gomock`

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

**运行测试**：

```bash
go test ./...                          # 全部
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
