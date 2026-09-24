# RTC Agent 文档审计报告

**日期**: 2026-09-24
**审计范围**: `docs/src/content/docs/` 下全部 90 篇文档（中文 45 篇 + 英文 45 篇）
**审计方法**: 逐篇阅读文档，对比 server / web-components 代码实际实现

---

## 执行摘要

| 指标 | 数值 |
|------|------|
| 审计文档总数 | 90 篇（zh 45 + en 45） |
| 代码对比模块 | server (Go), web-components (TypeScript) |
| 严重问题 | 0 |
| 中等问题 | 1 |
| 轻微问题 | 0 |
| 建议 | 0 |

**总体评价**: 文档质量很高，与代码实现高度一致。协议文档（HTTP API、WebSocket RPC、实时事件）、组件 API 文档、配置项文档、部署文档均与代码完全匹配。唯一发现的问题是英文版 session.md 在翻译时遗漏了"重新打开会话"（Reopen）功能的描述。

---

## 问题清单

### 中等 (1)

#### 1. 英文版 session.md 遗漏 Closed -> Idle 重新打开功能

- **文件**: `docs/src/content/docs/en/features/session.md`
- **问题描述**: 英文版 session.md 在翻译时遗漏了会话"重新打开"功能的描述，导致文档与实际实现不一致。具体遗漏 4 处：
  1. 状态图中 `Closed --> [*]` 应为 `Closed --> Idle: Reopen session`
  2. Closed 状态可执行操作遗漏 "Reopen"
  3. 操作表缺少 "Open" 行
  4. 实时更新事件表缺少 "Opened" 事件
  5. 高级操作子图缺少 "Open" 节点
- **代码证据**:
  - `server/pkg/protocol/models.gen.go:165` -- `MethodSessionOpen RpcMethod = "v1.session.open"`
  - `server/internal/handler/rpc/handler.go:75` -- `protocol.MethodSessionOpen: dispatch(h.OpenSession)`
  - `server/pkg/protocol/models.gen.go:726` -- `OpenSessionResponse` 结构体定义
- **修正动作**: 补齐 5 处遗漏内容，使英文版与中文版保持一致

---

## 中英文同步状态

| 文档路径 | 中文行数 | 英文行数 | 同步状态 |
|----------|:--------:|:--------:|:--------:|
| introduction.md | 147 | 147 | 一致 |
| getting-started.md | 202 | 202 | 一致 |
| resume.md | -- | -- | 仅中文 |
| protocol/index.md | 176 | 176 | 一致 |
| protocol/http-api.md | 358 | 358 | 一致 |
| protocol/rpc.md | 407 | 407 | 一致 |
| protocol/events.md | 306 | 306 | 一致 |
| concepts/rtc.md | 215 | 215 | 一致 |
| concepts/virtual-fs.md | 207 | 207 | 一致 |
| concepts/script-engine.md | 254 | 254 | 一致 |
| concepts/work-modes.md | 163 | 163 | 一致 |
| features/session.md | 400 | 400 | **已修正** (原 397) |
| features/messaging.md | 388 | 388 | 一致 |
| features/skill-system.md | 310 | 310 | 一致 |
| features/commands.md | 284 | 284 | 一致 |
| features/memory.md | 296 | 296 | 一致 |
| features/context-management.md | 356 | 356 | 一致 |
| features/realtime.md | 272 | 272 | 一致 |
| features/llm-tools.md | 421 | 421 | 一致 |
| features/notifications.md | 337 | 337 | 一致 |
| features/settings.md | 398 | 398 | 一致 |
| integration/auth.md | 436 | 436 | 一致 |
| integration/component-api.md | 683 | 683 | 一致 |
| integration/function-registration.md | 258 | 258 | 一致 |
| integration/scenario-authoring.md | 233 | 233 | 一致 |
| integration/i18n.md | 350 | 350 | 一致 |
| integration/integration-tutorial.md | 236 | 236 | 一致 |
| integration/faq.md | 275 | 275 | 一致 |
| deployment/source-build.md | 104 | 104 | 一致 |
| deployment/cdn.md | 164 | 164 | 一致 |
| deployment/distributed-deploy.md | 211 | 211 | 一致 |
| architecture/index.md | 141 | 141 | 一致 |
| architecture/frontend.md | 356 | 356 | 一致 |
| architecture/backend.md | 415 | 415 | 一致 |
| operations/* (8 篇) | -- | -- | 一致 |
| showcase/* (3 篇) | -- | -- | 一致 |
| legal/* (2 篇) | -- | -- | 一致 |

---

## 已验证的代码一致性

### 协议层 (Protocol)

| 验证项 | 文档描述 | 代码实际 | 状态 |
|--------|---------|---------|:----:|
| HTTP 端点 | 4 个 OAuth2 + 3 个运维 + 2 个业务 | `server/internal/handler/http/` 路由注册 | 一致 |
| OAuth2 Token 响应 | `access_token + refresh_token + expires_in + user_id` | `OAuth2TokenExchangeResponse` 结构体 | 一致 |
| OAuth2 Refresh 响应 | `access_token + expires_in` | `OAuth2TokenRefreshResponse` 结构体 | 一致 |
| Healthz 响应 | `{"status": "ok"}` | `health.go:39` | 一致 |
| Readyz 响应 | `{"status": "ready", "checks": {...}}` | `health.go:94` | 一致 |
| RPC 方法数量 | 17 个（9 Action + 8 Query） | `handler.go:70-95` 注册 17 个路由 | 一致 |
| RPC 方法名 | `v1.session.*`, `v1.message.*`, `v1.turn.*`, `v1.rtc.*` | `models.gen.go:154-170` | 一致 |
| ContentType 枚举 | 9 种 | `models.gen.go:13-21` | 一致 |
| MessageRole 枚举 | 4 种 | `models.gen.go:88-91` | 一致 |
| 事件频道 | Topic + Live 双频道 | Centrifuge 配置 | 一致 |

### 组件层 (Web Component)

| 验证项 | 文档描述 | 代码实际 | 状态 |
|--------|---------|---------|:----:|
| 组件属性 | theme, lang, app-label, database-name, bubble-icon, scenarios-url, server-url, redirect-uri | `rtc-agent.ts:204-422` | 一致 |
| JS 属性 | agentConfig, registry, windowConfig, activityBarConfig | `rtc-agent.ts:263-491` | 一致 |
| 事件 | rtc-agent-ready | `rtc-agent.ts` connectedCallback | 一致 |
| reconnect() 方法 | 存在 | `rtc-agent.ts:1030` | 一致 |
| connectionFailed/connectionError | 存在 | `rtc-agent.ts:1043-1051` | 一致 |
| 工作模式 | manual, edit, plan, auto, bypass | `types/index.ts:200` | 一致 |
| CDN 版本 | @rtc-agent/component@0.2.3 | npm registry: 0.2.3 | 一致 |

### 服务端 (Server)

| 验证项 | 文档描述 | 代码实际 | 状态 |
|--------|---------|---------|:----:|
| Go 版本要求 | Go 1.27+ | `go.mod: go 1.27.0` | 一致 |
| 配置项 | database.dsn, redis.addr, llm.provider/api_key/model 等 | `config.go` 结构体 | 一致 |
| LLM 定价配置 | input/output/cached_read/cached_write/reasoning per million | `ModelPricingConfig` | 一致 |
| 重试配置 | retry_max_attempts (default 0), retry_base_delay (default 1s) | `config.go:406-407` | 一致 |
| 缓存告警阈值 | worker.cache_hit_rate_warn_threshold | `config.go:238` | 一致 |
| LLM 内置工具 | 16 个工具（subAgent, askUser, todoWrite, goal, loop 等） | `internal/agent/tools_*.go` | 一致 |
| Docker 端口 | 28080(Nginx), 25432(PG), 26379(Redis) 等 | `docker-compose.yml` | 一致 |

---

## 已修正内容汇总

### `docs/src/content/docs/en/features/session.md`

| 行号 | 修正前 | 修正后 |
|------|--------|--------|
| 22 | `Closed --> [*]` | `Closed --> Idle: Reopen session` |
| 37 | `Closed, no further writes allowed \| View history, Fork` | `Closed (archived), can view history or reopen \| View history, Fork, Reopen` |
| 54-57 | Advanced 子图只有 Fork + Close | 新增 `A3["🔓 Open<br/>Restore closed session"]` |
| 71 | 操作表到 Close 结束 | 新增 `Open` 行：Restore a Closed session back to Idle |
| 200-204 | 实时更新事件表只有 Created/Updated/Closed | 新增 `Opened` 事件行 |
