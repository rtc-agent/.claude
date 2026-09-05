---
name: rtc-agent-e2e-standards
description: Use when writing, running, or debugging E2E tests for RTC Agent — covers requirement analysis, test structure, dev backdoors, environment setup, troubleshooting, and test reports
---

# RTC Agent E2E 测试规范

## 核心原则

**写 E2E 测试之前，必须先理解产品需求。**

这不是建议，是强制要求。没有例外。

## 强制流程

### 第 1 步：明确测试场景

用户提出测试需求。如果不清晰，先和用户确认要测什么。

### 第 2 步：调研产品需求（必须）

**调度 Explore 子代理调研**，不要手动猜测：

1. **扫描代码**：根据测试场景，让子代理搜索相关的代码模块、API、数据结构
2. **发现需求文档**：子代理根据代码分析，自动找到对应的产品需求文档（`rtc-agent-product-standards/*.md`）
3. **阅读需求**：子代理阅读需求文档，提取用户场景、业务规则、边界情况
4. **汇报发现**：子代理返回调研结果，包括代码结构和需求要点

**示例 prompt**：
```
调研 RTC Agent 的认证流程，准备编写 E2E 测试：
1. 搜索 server/ 和 web-components/ 中与认证相关的代码
2. 找到对应的产品需求文档并阅读
3. 提取：用户场景、业务规则、边界情况、错误处理
4. 汇报发现
```

**不要**：手动查表找文档。
**要**：让子代理动态扫描代码和文档。

### 第 3 步：和用户确认需求（必须）

列出场景、边界情况、错误处理，等待用户确认后进入下一步。

### 第 4 步：决定测试工程

| 工程 | 框架 | 适用 |
|------|------|------|
| `server/` | Go testing + httptest | 后端 API、业务逻辑、数据库 |
| `web-components/` | Playwright | 前端 UI、用户交互、端到端流程 |

决策依据：测后端 → server/；测前端交互 → web-components/；完整流程 → 两边都写。

### 第 5 步：编写测试

见下文「编写规范」。

### 第 6 步：运行测试

见下文「运行规范」。

### 第 7 步：测试结果报告

见下文「报告规范」。

---

## 编写规范

### 目录结构

```
server/
  tests/
    e2e/
      <module>_test.go          # 按模块分文件
    helpers/
      db.go                     # 数据库 helper
      api.go                    # HTTP client helper
      fixture.go                # 测试数据构造
      cleanup.go                # 数据清理

web-components/packages/component/
  tests/
    <module>.spec.ts            # 按模块分文件
    helpers/
      fixtures.ts               # 测试 fixture
      api.ts                    # API helper
      debug.ts                  # debugRtcAgent wrapper
```

### 命名规范

**Go 测试**
```go
func TestAuth_Login_Success(t *testing.T)
func TestAuth_Login_InvalidCredentials(t *testing.T)
func TestAuth_Token_Refresh(t *testing.T)
```
格式：`Test<模块>_<场景>_<预期结果>`

**Playwright 测试**
```typescript
test('auth - login success', async ({ page }) => { ... })
test('auth - invalid credentials', async ({ page }) => { ... })
test('auth - token refresh', async ({ page }) => { ... })
```
格式：`<模块> - <场景描述>`

### 测试结构

每个测试用例遵循 **Arrange-Act-Assert**：

```go
func TestAuth_Login_Success(t *testing.T) {
    // Arrange: 准备测试数据
    user := helpers.CreateUser(t, db, "test@example.com")

    // Act: 执行被测操作
    resp := helpers.PostJSON(t, server, "/oauth2/token", loginReq{...})

    // Assert: 验证结果
    assert.Equal(t, 200, resp.StatusCode)
    assert.NotEmpty(t, resp.Body.AccessToken)
}
```

### 测试覆盖要求

**每个需求场景必须编写一个测试 function，包含全流程 happy path。**

- 一个 function = 一个完整的正常路径测试
- 全流程：从开始到结束，所有步骤成功，不跳步
- 不 mock 中间环节（除非必要）

示例：
```go
// 一个 function = 完整的 OAuth2 登录流程
func TestAuth_OAuth2_Login_HappyPath(t *testing.T) {
    // 获取授权 URL → 模拟授权 → 换取 token → 验证 token
    // 全流程，不拆分
}
```

```typescript
// 一个 function = 完整的消息收发流程
test('message - send and receive - happy path', async ({ page }) => {
    // 登录 → 创建会话 → 发送消息 → 等待回复 → 验证
    // 全流程，不拆分
})
```

**边界情况和错误处理**：在 happy path function 之外，补充专门的测试 function。

### 测试隔离

- 每个测试独立，不依赖其他测试的状态
- 测试前 setup，测试后 teardown
- 使用 unique 前缀（如 `test-<uuid>`）避免冲突
- 测试结束后清理自己创建的数据

---

## 测试后门规范

E2E 测试需要后门来绕过认证、注入数据、检查状态。

### 后端：`backdoor` cobra 子命令

在 `server/cmd/` 下新建 `backdoor_*.go` 文件，所有测试后门命令都是 `backdoor` 的子命令。

**命令体系**：
```
backdoor
├── token <user-id>              # 生成测试 JWT
├── create-user <email>          # 创建测试用户，输出 user-id
├── seed                         # 插入测试数据集
├── reset-db                     # TRUNCATE 所有表
├── inspect-session <id>         # 查看会话详情
└── list-users                   # 列出所有用户
```

**示例**：`server/cmd/backdoor_token.go`
```go
package cmd

import (
    "fmt"
    "github.com/google/uuid"
    "github.com/spf13/cobra"
    "github.com/rtc-agent/server/internal/infra/auth"
)

var backdoorTokenCmd = &cobra.Command{
    Use:   "token [user-id]",
    Short: "Generate a test JWT token (dev only)",
    Args:  cobra.ExactArgs(1),
    RunE: func(cmd *cobra.Command, args []string) error {
        userID, err := uuid.Parse(args[0])
        if err != nil {
            return fmt.Errorf("invalid user-id: %w", err)
        }
        deviceID, _ := cmd.Flags().GetString("device-id")

        signer := auth.NewJWTSigner(
            "rtc-agent-dev-jwt-secret-change-me-in-production",
            3600,
        )
        token, expiresAt, err := signer.SignAccessToken(userID, deviceID)
        if err != nil {
            return err
        }

        fmt.Printf("Token: %s\nExpires: %s\n", token, expiresAt)
        return nil
    },
}

func init() {
    backdoorTokenCmd.Flags().String("device-id", "test-device", "Device ID")
    backdoorCmd.AddCommand(backdoorTokenCmd)
}
```

**父命令**：`server/cmd/backdoor.go`
```go
package cmd

import "github.com/spf13/cobra"

var backdoorCmd = &cobra.Command{
    Use:   "backdoor",
    Short: "Test backdoor commands (dev/test only, NEVER enable in production)",
}

func init() {
    rootCmd.AddCommand(backdoorCmd)
}
```

**注意**：`backdoor_*.go` 文件与 `backdoor.go` 在同一 `cmd` 包内，通过包级变量 `backdoorCmd` 跨文件引用。

**编译约束**：backdoor 命令必须通过 build tag 控制，生产构建不包含：
```go
//go:build dev || test
```

### 前端：`window.debugRtcAgent`

仅在 `import.meta.env.DEV` 或 `import.meta.env.TEST` 下注入。

```typescript
// src/debug.ts
export function setupDebugInterface() {
  if (!import.meta.env.DEV && !import.meta.env.TEST) return;

  window.debugRtcAgent = {
    // 查看状态
    getState: () => ({ /* auth, session, messages 状态 */ }),

    // 清理数据
    clearIndexedDB: async () => { indexedDB.deleteDatabase('rtc-agent'); },
    clearLocalStorage: () => { localStorage.clear(); },

    // 模拟登录（跳过 OAuth2）
    loginAs: (userId: string) => { /* 设置 token，reload */ },

    // 查看 VirtualFS
    listFiles: async () => { /* 列出虚拟文件系统内容 */ },

    // 触发事件
    triggerEvent: (name: string, data: any) => {
      window.dispatchEvent(new CustomEvent(name, { detail: data }));
    },

    // 日志历史
    logs: [] as string[],
  };
}

declare global {
  interface Window {
    debugRtcAgent?: { /* ... */ };
  }
}
```

### 后端 HTTP 层：dev bypass

已有机制，通过 `X-User-ID` / `X-Device-ID` header 绕过 JWT：
```bash
curl -H "X-User-ID: <uuid>" -H "X-Device-ID: <device>" \
     http://localhost:8888/api/...
```
仅在 `allowDevBypass: true` 时生效。

---

## 运行规范

### 环境启动

```bash
# 1. 启动依赖（PostgreSQL + Redis）
cd server && go run main.go dev dependence start
# 或：cd server/etc/dev && docker-compose up -d

# 2. 数据库迁移
cd server && go run main.go migrate

# 3. 启动 server
cd server && go run main.go serve
# 配置文件：server/etc/config.local.yaml
```

**如果需要测试 OAuth2 认证流程**，需要启动两个 server：

```bash
# Terminal 1: 启动 mock OAuth2 server
cd server && go run main.go mock-oauth2
# 配置文件：server/etc/mock-oauth2.yaml
# 默认端口：10060

# Terminal 2: 启动主 server
cd server && go run main.go serve
# 配置文件：server/etc/config.local.yaml
# 默认端口：8888
```

**配置文件说明**：
- `server/etc/config.yaml`：默认配置模板
- `server/etc/config.local.yaml`：本地覆盖配置（优先级更高，gitignore）
- `server/etc/mock-oauth2.yaml`：mock OAuth2 server 配置（client_id、client_secret、端口）

启动时 server 会合并两个配置，`config.local.yaml` 中的值覆盖 `config.yaml`。

```bash
# 4. 启动前端（Playwright 测试需要）
# Playwright 自动启动 web server（localhost:3000）
# 或手动：cd web-components && pnpm dev
```

### 运行测试

**server/ (Go)**
```bash
cd server
go test ./tests/e2e/... -v              # 所有 E2E
go test ./tests/e2e/... -run TestAuth   # 按模块
go test ./tests/e2e/... -count=1        # 跳过缓存
```

**web-components/ (Playwright)**
```bash
cd web-components
pnpm exec playwright test                          # 所有
pnpm exec playwright test tests/auth.spec.ts       # 单文件
pnpm exec playwright test -g "login success"       # 按名称
pnpm exec playwright test --headed                 # 有头模式
pnpm exec playwright test --ui                     # UI 模式
DEBUG=pw:api pnpm exec playwright test             # 调试日志
```

### 数据清理

```bash
# 清空所有数据（Docker volume 删除 + 重建）
cd server && go run main.go dev dependence flushall

# 或通过 backdoor
cd server && go run main.go backdoor reset-db

# PostgreSQL 手动
psql -h localhost -p 15432 -U rtc_agent -d rtc_agent \
  -c "TRUNCATE oauth2_users, sessions, messages CASCADE;"

# Redis 手动
redis-cli -h localhost -p 16379 FLUSHDB

# IndexedDB（Playwright 中）
await page.evaluate(() => indexedDB.deleteDatabase('rtc-agent'));
```

---

## 排查问题

### PostgreSQL

```bash
# 连接
psql -h localhost -p 15432 -U rtc_agent -d rtc_agent

# 常用查询（示例，实际字段以 server/internal/model/*.go 为准）
SELECT * FROM oauth2_users;
SELECT * FROM sessions WHERE user_id = '...';
SELECT * FROM refresh_tokens WHERE user_id = '...';
SELECT * FROM pg_indexes WHERE tablename = 'sessions';

# 查看所有表
\dt

# 查看表结构
\d oauth2_users
```

### Redis

```bash
# 连接
redis-cli -h localhost -p 16379

# 常用命令
KEYS *                         # 所有 key
GET session:<id>               # 查看 session
TTL session:<id>               # 查看过期时间
SCAN 0 MATCH oauth:state:*     # 查找 OAuth state
```

### IndexedDB（浏览器）

```javascript
// 控制台
debugRtcAgent.listFiles()      // 虚拟文件
debugRtcAgent.getState()       // 应用状态

// Chrome DevTools → Application → IndexedDB → rtc-agent
```

### console.log

```typescript
// Playwright 捕获
page.on('console', msg => console.log('BROWSER:', msg.text()));
page.on('pageerror', err => console.log('PAGE ERROR:', err.message));

// 调试日志
DEBUG=pw:api,pw:browser pnpm exec playwright test
```

### Server 日志

```bash
# 调高日志级别
# etc/config.yaml → log.level: "debug"

# 查看 Docker 日志
docker logs rtc-agent-postgres
docker logs rtc-agent-redis
```

---

## 报告规范

### 测试报告位置

```
server/
  tests/
    e2e/
      reports/                  # 首次运行时手动创建
        <date>-<module>.md      # 测试结果报告

web-components/packages/component/
  tests/
    reports/                    # 首次运行时手动创建
      <date>-<module>.md
```

### 报告模板

```markdown
# E2E 测试报告: <模块名称>

**日期**: YYYY-MM-DD
**环境**: local / CI
**工程**: server / web-components

## 概要

| 指标 | 值 |
|------|-----|
| 总用例 | N |
| 通过 | N |
| 失败 | N |
| 跳过 | N |
| 耗时 | Xs |

## 失败用例

### <测试名称>
- **预期**: ...
- **实际**: ...
- **排查**: PostgreSQL 中 sessions 表无记录，原因是 ...
- **修复建议**: ...

## 覆盖场景

- [x] 正常流程
- [x] 边界情况
- [ ] 并发场景（未实现）

## 备注

- 发现的 bug: #42
- 需要补充的场景: ...
```

### 何时写报告

- 有失败用例时必须写
- 全部通过可写简报
- CI 失败必须写报告并关联 issue

---

## 红旗信号 - 停下来

- "我知道这个功能怎么工作"
- "先写个基础版本，用户会纠正"
- "需求太明显了，不用确认"
- "用户很忙，不要打扰"

**这些意味着：停下。调研需求。和用户确认。**

## 快速检查清单

- [ ] 我读过产品需求了吗？
- [ ] 我和用户确认过理解吗？
- [ ] 我决定了在哪个工程写测试吗？
- [ ] 测试用了 Arrange-Act-Assert 结构吗？
- [ ] 测试是独立的，不依赖其他测试？
- [ ] 失败用例写测试报告了吗？
