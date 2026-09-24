# RTC Agent 文档审计报告

**日期**: 2026-09-24
**审计范围**: `docs/src/content/docs/` 下全部中文和英文文档（共 ~92 个 .md/.mdx 文件）
**代码参照**: `server/` (Go), `web-components/` (TypeScript)

---

## 执行摘要

| 指标 | 数值 |
|------|------|
| 审计文档数 | ~92（中文 ~46 + 英文 ~46） |
| 覆盖模块 | protocol, features, concepts, integration, deployment, architecture, operations, showcase |
| 发现问题总数 | 5 |
| 严重问题 | 0 |
| 中等问题 | 2 |
| 轻微问题 | 0 |
| 建议优化 | 3 |
| 已修正文件 | 4 |

**总体评价**: 文档质量较高，协议描述、API 参考、架构说明与代码实现高度一致。中英文文档内容同步。发现的 2 个中等问题已修正。

---

## 问题清单

### 中等问题（已修正）

#### 1. `virtual-fs.md` (中/英): `write` 工具状态标注错误

- **文件**: `docs/src/content/docs/concepts/virtual-fs.md` 第 89 行
- **英文文件**: `docs/src/content/docs/en/concepts/virtual-fs.md` 第 89 行
- **问题描述**: 文档标注 `write` 工具为 "🔜 暂未启用"，并附警告称 "write 工具当前处于禁用状态"。但实际代码中 `write` 工具已完整实现并注册为活跃工具。
- **代码证据**:
  - `server/internal/agent/data.go:132` — `&writeTool{base: base}` 注册在活跃工具列表中
  - `server/internal/agent/tools.go:101-116` — `writeTool` 完整实现（Info + InvokableRun）
  - `web-components/packages/persistence/src/tools/builtin.ts:130-151` — 前端 `WriteTool` 完整实现，支持 `overwrite` / `append` / `create-new` 三种模式
- **修正动作**:
  - 将 `write` 状态从 "🔜 暂未启用" 改为 "✅ 可用"
  - 补充 `mode` 参数值 `create-new`
  - 移除 "write 工具当前处于禁用状态" 警告
  - 中英文文档同步修正

#### 2. `work-modes.md` (中/英): 权限矩阵中 `write` 标注错误

- **文件**: `docs/src/content/docs/concepts/work-modes.md` 第 43, 50, 57, 74 行
- **英文文件**: `docs/src/content/docs/en/concepts/work-modes.md` 第 43, 50, 57, 74 行
- **问题描述**: 权限矩阵中多处标注 "write（暂未启用）" / "write (not yet enabled)"，并在矩阵上方添加了 "write 工具当前处于禁用状态" 的警告框。与实际代码不符。
- **代码证据**: 同上（write 工具已完整注册并活跃）
- **修正动作**:
  - 移除 "write 工具当前处于禁用状态" 警告框
  - Mermaid 图中 "write（暂未启用）" → "write"
  - 权限表格中 "write（暂未启用）" → "write"
  - 中英文文档同步修正

### 建议优化（未修正，记录备查）

#### 3. 部署文档中 LLM 模型示例可能造成误解

- **文件**: `getting-started.md`, `deployment/source-build.md`, `deployment/distributed-deploy.md`
- **问题描述**: 配置示例中 `llm.model` 写为 `"claude-sonnet-4-20250514"`，但 `etc/config.docker.yaml` 的实际默认值为 `"qwen3.7-plus"`。虽然文档有括号注释 "实际默认配置可能不同（如 qwen3.7-plus）"，但示例值与实际配置不一致可能让首次部署的用户困惑。
- **建议**: 将示例模型名改为与 `config.docker.yaml` 一致，或使用占位符 `"your-model-name"` 并附说明。

#### 4. 部署文档中 `providers.mock` 配置不一致

- **文件**: `deployment/distributed-deploy.md`
- **问题描述**: 文档配置示例显示 `providers.mock.enabled: true` 和 `url: "http://mock-oauth2:10060"`，但 `etc/config.docker.yaml` 中为 `enabled: false` 和 `url: "http://192.168.31.60:20060"`。文档是作为分布式部署的覆盖配置，逻辑上合理，但应更明确说明这是覆盖值而非默认值。
- **建议**: 在配置片段前加注释说明 "以下配置覆盖 config.docker.yaml 中的默认值"。

#### 5. `concepts/rtc.md` 中 `write` 工具参数缺少 `mode`

- **文件**: `docs/src/content/docs/concepts/rtc.md` 第 79 行
- **问题描述**: RTC 内置工具表中 `write` 的关键参数只列了 `path` 和 `content`，未提及 `mode` 参数。实际实现支持 `overwrite` / `append` / `create-new` 三种写入模式。
- **建议**: 在参数列补充 `mode`。

---

## 中英文同步状态

| 区域 | 中文 | 英文 | 同步状态 |
|------|:----:|:----:|:--------:|
| About (resume) | ✅ | ✅ | 同步 |
| Getting Started | ✅ | ✅ | 同步 |
| Deployment (3 篇) | ✅ | ✅ | 同步 |
| Concepts (4 篇) | ✅ | ✅ | 同步 |
| Features (10 篇) | ✅ | ✅ | 同步 |
| Integration (7 篇) | ✅ | ✅ | 同步 |
| Protocol (4 篇) | ✅ | ✅ | 同步 |
| Architecture (3 篇) | ✅ | ✅ | 同步 |
| Operations (8 篇) | ✅ | ✅ | 同步 |
| Showcase (3 篇) | ✅ | ✅ | 同步 |
| Legal (2 篇) | ✅ | ✅ | 同步 |

**结论**: 中英文文档完全同步，所有中文文档都有对应的英文翻译版本。本次修正的 2 个问题已同步修正中英文版本。

---

## 已修正内容汇总

| 文件 | 修改点 |
|------|--------|
| `docs/src/content/docs/concepts/virtual-fs.md` | `write` 工具状态改为 "可用"；补充 `mode` 参数值；移除禁用警告 |
| `docs/src/content/docs/en/concepts/virtual-fs.md` | 同上（英文版） |
| `docs/src/content/docs/concepts/work-modes.md` | 移除 `write` 禁用警告框；Mermaid 图和表格中移除 "暂未启用" 标注 |
| `docs/src/content/docs/en/concepts/work-modes.md` | 同上（英文版） |

---

## 审计方法

1. 解析 `astro.config.mjs` 获取完整文档目录树（sidebar 配置）
2. 逐一读取所有中文和英文文档文件
3. 根据文档内容动态调研代码模块：
   - HTTP API 端点 → `server/internal/handler/http/` (oauth2.go, health.go, interrupt.go, memories.go)
   - WebSocket RPC 方法 → `server/internal/handler/rpc/` (handler.go, action.*, query.*)
   - RPC 方法枚举 → `server/pkg/protocol/models.gen.go`
   - OAuth2 协议模型 → `server/pkg/protocol/models.gen.go` (OAuth2TokenExchangeResponse 等)
   - LLM 内置工具 → `server/internal/agent/tools_*.go`, `server/internal/agent/data.go`
   - 前端内置工具 → `web-components/packages/persistence/src/tools/builtin.ts`
   - Web Component 属性 → `web-components/packages/component/src/components/rtc-agent/rtc-agent.ts`
   - 工作模式 → `web-components/packages/component/src/contexts/mode.ts`
   - 服务端配置 → `server/internal/infra/config/config.go`, `server/etc/config.docker.yaml`
   - Docker 部署 → `server/docker-compose.yml`
   - npm 包版本 → `npm view @rtc-agent/component version`
4. 逐项对比文档与代码的一致性
5. 检查中英文文档同步状态

## 验证正确的关键文档内容

以下内容经代码验证确认准确：

- **HTTP API**: 4 个 OAuth2 端点、3 个运维端点、2 个业务端点 — 与 `server/internal/handler/http/` 完全一致
- **WebSocket RPC**: 17 个方法（9 Action + 8 Query），4 个业务域 — 与 `server/pkg/protocol/models.gen.go` 枚举完全一致
- **实时事件**: Topic/Live 双频道、事件分布表、Update 模型 — 与代码实现一致
- **OAuth2 响应格式**: access_token, refresh_token, expires_in, user_id — 与 `OAuth2TokenExchangeResponse` 结构一致
- **错误格式**: code, message, details — 与 `APIError` 结构一致
- **ContentType 枚举**: 9 种类型 — 与代码一致
- **MessageRole 枚举**: 4 种角色 — 与代码一致
- **Component 属性**: theme, lang, database-name, app-label, server-url, redirect-uri, scenarios-url, bubble-icon — 与 `rtc-agent.ts` 完全一致
- **JS 属性**: agentConfig, registry, windowConfig, activityBarConfig — 与代码一致
- **npm CDN 版本**: `@0.2.3` — 与 npm 发布版本一致
- **LLM 内置工具**: subAgent, listSubAgent, getSubAgentMessage, stopSubAgent, sendMessageToSubAgent, askUser, todoWrite, createGoal, completeGoal, cancelGoal, createLoop, cancelLoop, completeLoop, listLoops, pauseLoop, resumeLoop, saveSessionMemory, listSessionMemories, searchMemory, saveUserMemory, updateUserMemory, deleteUserMemory, listUserMemory — 全部在代码中验证存在
- **RTC 内置工具**: ls, read, write, edit, grep, find, script, askUser — 全部在代码中验证存在
- **Docker 端口映射**: 25432, 26379, 26686, 24317, 29090, 23100, 24040, 23001, 29093, 20060, 28080 — 与 docker-compose.yml 一致
- **Go 版本**: 1.27+ — 与 go.mod `go 1.27.0` 一致
- **配置默认值**: `llm.retry_max_attempts` 默认 0, `llm.retry_base_delay` 默认 1s — 与 config.go SetDefault 一致
