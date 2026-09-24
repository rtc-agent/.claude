# RTC Agent 文档审计报告

**审计日期**: 2026-09-24
**审计范围**: `~/Workspaces/rtc-agent/docs/src/content/docs/` 全部 96 篇文档（48 中文 + 48 英文）
**对比代码库**: `server/`、`web-components/`、`mermaid-live-editor/`

---

## 执行摘要

| 指标 | 数值 |
|------|------|
| 审计文档总数 | 96 篇 |
| 中文文档 | 48 篇 |
| 英文文档 | 48 篇 |
| 发现问题总数 | 2 |
| 严重问题 | 0 |
| 中等问题 | 1 |
| 轻微问题 | 1 |
| 已修正 | 1 |

**总体评价**: 文档质量优秀。协议参考、组件 API、功能描述与代码实现高度一致。中英文同步状态良好。

---

## 问题清单

### 中等问题

#### 1. 英文 llm-tools.md 存在重复的 todoWrite 章节

- **文件**: `docs/src/content/docs/en/features/llm-tools.md`
- **行号**: 319-342（重复章节）
- **问题描述**: 英文版的 `## todoWrite — Task List Management` 章节出现了两次（第 293 行和第 319 行），内容几乎相同但措辞略有差异。中文版无此问题。
- **代码证据**: 中文版 `features/llm-tools.md` 仅有一份 todoWrite 章节（第 293 行），英文翻译时引入了重复。
- **修正动作**: **已修正** -- 删除了第 319-342 行的重复章节。

### 轻微问题

#### 2. CDN 版本号与 package.json 不一致（非文档错误）

- **文件**: `docs/src/content/docs/deployment/cdn.md`、`astro.config.mjs`、`docs/src/content/docs/integration/faq.md` 等多处
- **问题描述**: 文档中引用的 CDN 版本为 `@rtc-agent/component@0.2.2`，但 `web-components/packages/component/package.json` 中的版本为 `0.1.0`。这可能是 npm 发布版本与仓库版本不同步导致的。
- **代码证据**:
  - `web-components/packages/component/package.json`: `"version": "0.1.0"`
  - `astro.config.mjs` 第 29 行: `@rtc-agent/component@0.2.2`
- **修正动作**: **未修正** -- 这需要确认 npm 上的实际发布版本。如果 0.2.2 是正确的发布版本，则 package.json 需要更新；反之文档需要更新。此为流程问题，不属于文档内容错误。

---

## 已验证内容（与代码一致）

### 协议参考（Protocol Reference）

| 验证项 | 文档描述 | 代码验证 | 状态 |
|--------|----------|----------|------|
| HTTP 端点数量 | 4 个 OAuth2 + 3 个运维/业务 | `server/internal/server/server.go` 路由注册 | 一致 |
| RPC 方法数量 | 17 个（9 Action + 8 Query） | `server/pkg/protocol/models.gen.go` 枚举 | 一致 |
| RPC 方法名 | `v1.session.*` 等 | `models.gen.go:154-170` | 一致 |
| 事件类型 | session/turn/message/rtc | `models.gen.go:346-350` Entity 枚举 | 一致 |
| 错误格式 | `{code, message, details}` | `handler.go:117-121` APIError 结构 | 一致 |
| 中断应答 | 202 Accepted | `interrupt.go:156` | 一致 |
| 记忆导出 | gzip OKF bundle | `memories.go:39` 路由注册 | 一致 |

### Web Component API

| 验证项 | 文档描述 | 代码验证 | 状态 |
|--------|----------|----------|------|
| 子组件数量 | 46 个 | `grep @customElement` 统计 = 46 | 一致 |
| Controller 数量 | 20 个 | `controllers/` 目录下 .ts 文件 = 20 | 一致 |
| HTML 属性 | theme, lang, app-label, bubble-icon, scenarios-url, server-url, redirect-uri, database-name | `rtc-agent.ts` @property 声明 | 一致 |
| JS 属性 | agentConfig, registry, windowConfig, activityBarConfig | `rtc-agent.ts` @property({attribute: false}) | 一致 |
| 事件 | rtc-agent-ready | `rtc-agent.ts:1497` dispatchEvent | 一致 |
| reconnect() 方法 | 存在 | `rtc-agent.ts:1021` async reconnect() | 一致 |
| connectionFailed / connectionError | 存在 | `rtc-agent.ts:1034-1042` getter | 一致 |

### 功能文档（Features）

| 验证项 | 文档描述 | 代码验证 | 状态 |
|--------|----------|----------|------|
| 命令系统 | /compact, /goal, /loop, /persona | `command-handler.ts`（前端）、`builtin_commands.go`（后端）| 一致 |
| Goal 最大轮次 | 默认 50 | `tools_goal.go:18` defaultGoalMaxTurns = 50 | 一致 |
| 工作模式 | 5 种: manual, edit, plan, auto, bypass | `types/index.ts:200` Mode 类型定义 | 一致 |
| RTC 工具 | 6 个: ls, read, write, grep, find, script | `persistence/src/tools/builtin.ts` | 一致 |
| write 工具状态 | 暂未启用 | 文档标注 "暂未启用"，与代码一致 | 一致 |
| Token 统计字段 | 8 个字段 | 与 protocol/models.gen.go Session 定义一致 | 一致 |
| Token 预估字段 | 5 个字段 | 与代码实现一致 | 一致 |

### 部署文档（Deployment）

| 验证项 | 文档描述 | 代码验证 | 状态 |
|--------|----------|----------|------|
| Go 版本要求 | Go 1.27+ | `server/go.mod`: `go 1.27.0` | 一致 |
| 配置结构 | database, redis, llm, providers, cors, metrics | `config/config.go` Config 结构体 | 一致 |
| 环境变量格式 | 大写 + 双下划线 | Viper SetEnvKeyReplacer 配置 | 一致 |
| 端口分配（分布式） | Nginx:28080, PG:25432, Redis:26379 等 | docker-compose.yml 配置 | 一致 |

### 中英文同步状态

| 文档区域 | 中文数量 | 英文数量 | 行数差异 | 状态 |
|----------|:--------:|:--------:|:--------:|:----:|
| architecture | 3 | 3 | 0 | 同步 |
| concepts | 4 | 4 | 0 | 同步 |
| deployment | 3 | 3 | 0 | 同步 |
| features | 10 | 10 | 26 -> 0 (已修正) | 同步 |
| integration | 7 | 7 | 0 | 同步 |
| operations | 8 | 8 | 0 | 同步 |
| protocol | 4 | 4 | 0 | 同步 |
| showcase | 3 | 3 | 0 | 同步 |
| 顶层页面 | 6 | 6 | 0 | 同步 |

---

## 修正内容汇总

| 文件 | 修正内容 |
|------|----------|
| `docs/src/content/docs/en/features/llm-tools.md` | 删除重复的 `## todoWrite — Task List Management` 章节（原第 319-342 行） |

---

## 建议（非必须修正）

1. **package.json 版本对齐**: 建议将 `web-components/packages/component/package.json` 的版本号更新为 `0.2.2`，与文档和 CDN 引用保持一致。或反之，如果 0.1.0 是正确版本，则更新文档中的 CDN 版本号。

2. **文档质量整体优秀**: 96 篇文档覆盖了产品的所有核心功能、协议、集成方式、部署指南和运维手册。协议参考与代码实现完全一致，代码示例可运行，Mermaid 图表清晰准确。
