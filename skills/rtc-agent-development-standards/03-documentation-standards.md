# 文档规范

> _代码告诉机器做什么，文档告诉人为什么这样做。好的文档是项目最好的投资。_

---

## 1. README 规范

README 是项目的门面。一个陌生的开发者打开项目，README 决定了他是否会继续阅读，还是转身离开。

### 必备章节

每个仓库的 README.md 应包含以下章节（按需裁剪，不强制全部）：

```markdown
# 项目名称

一句话描述项目是什么。

## 功能特性

- 特性 1
- 特性 2
- 特性 3

## 快速开始

### 前置条件

列出需要的环境、工具、版本。

### 安装

\`\`\`bash
# 安装命令
\`\`\`

### 使用

\`\`\`bash
# 运行示例
\`\`\`

## 项目结构

简要说明关键目录的职责。

## 开发指南

### 本地开发

\`\`\`bash
# 启动开发环境
\`\`\`

### 测试

\`\`\`bash
# 运行测试
\`\`\`

### 构建

\`\`\`bash
# 构建命令
\`\`\`

## 贡献指南

如何提 issue、如何提 PR、代码规范链接。

## License

[MIT](LICENSE) 或其他许可证。
```

### 好的 README 的特征

- **5 分钟内可运行**：一个新人按照 README 操作，5 分钟内能把项目跑起来
- **有实际示例**：不是抽象的 API 描述，而是可以直接复制粘贴运行的代码
- **有截图或 GIF**：UI 项目应当有视觉展示
- **有徽章**：构建状态、覆盖率、版本号——一眼可见项目健康度
- **保持简洁**：README 不是百科全书，详细内容链接到专门文档

### 各仓库 README 特殊要求

| 仓库 | 额外章节 |
|------|---------|
| **server** | API 概览、架构图、环境变量说明 |
| **web-components** | 组件列表与预览、Demo 链接 |
| **docs** | API 文档导航、协议版本说明 |
| **mermaid-live-editor** | 在线 Demo 链接、截图 |
| **.claude** | Skill 列表与使用说明 |

---

## 2. 内联注释约定

与[通用原则](./01-universal-principles.md#2-注释的纪律)一致，补充跨仓库统一的标签格式。

### TODO / FIXME / DEPRECATED

使用统一格式，便于搜索和追踪：

```go
// TODO(PineappleBond): 添加连接池支持 — Issue #42
// FIXME: 高并发下存在竞态条件，需要加锁
// DEPRECATED: 使用 NewHandler 替代，将在 v2.0 移除
```

```typescript
// TODO(PineappleBond): 添加断线重连机制 — Issue #42
// FIXME: Safari 下 WebSocket 偶尔断开
// DEPRECATED: 使用 `v2` 版本 API 替代
```

### 格式规则

- **标签后附括号注明负责人**：`TODO(name)` 或 `TODO(团队名)`
- **关联 issue 编号**：`TODO: xxx — Issue #N`
- **DEPRECATED 必须给出替代方案**：告诉读者应该用什么
- **定期清理**：每个迭代检查 TODO 列表，过期的删除，仍有效的更新

---

## 3. 变更日志 (CHANGELOG)

### 格式：Keep a Changelog

遵循 [keepachangelog.com](https://keepachangelog.com/) 规范：

```markdown
# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/),
and this project adheres to [Semantic Versioning](https://semver.org/).

## [Unreleased]

### Added
- 新增房间事件流式推送接口

### Fixed
- 修复 WebSocket 重连时消息重复的问题

## [1.2.0] - 2026-09-01

### Added
- 支持房间级别的消息广播
- 新增用户在线状态查询 API

### Changed
- 优化 Centrifuge 连接池管理，降低 broker 负载

### Fixed
- 修复高并发下偶发的 panic (#128)

### Removed
- 移除废弃的 v1 认证接口
```

### 条目分类

| 分类 | 含义 |
|------|------|
| **Added** | 新功能 |
| **Changed** | 对现有功能的变更 |
| **Deprecated** | 即将移除的功能 |
| **Removed** | 已移除的功能 |
| **Fixed** | Bug 修复 |
| **Security** | 安全相关修复 |

### 自动化

- 基于 Conventional Commits 自动生成 CHANGELOG 条目
- Go 仓库推荐 `git-chglog`
- JS/TS 仓库推荐 `changesets` 或 `conventional-changelog`
- 每次发布前，人工审核自动生成的 CHANGELOG，确保语义准确

---

## 4. 架构决策记录 (ADR)

### 何时写 ADR

当一个决策满足以下任一条件时，应当写 ADR：

- 影响多个模块或多个仓库
- 有明显的替代方案，且选择理由不充分讨论无法还原
- 涉及技术选型（框架、库、协议）
- 不可逆或难以回退的决策

### ADR 模板

存放位置：`.claude/adr/` 或各仓库的 `docs/adr/`

```markdown
# ADR-001: 使用 Centrifuge 作为实时通信层

## 状态

已采纳 (2026-08-15)

## 上下文

RTC Agent 需要支持多用户实时协作，包括消息推送、状态同步和事件广播。
需要选择一个可靠的实时通信方案。

## 备选方案

1. **原生 WebSocket**：灵活但需要自行实现连接管理、重连、鉴权
2. **Socket.IO**：功能丰富但引入了自己的协议，与标准不兼容
3. **Centrifuge**：专为实时应用设计，自带连接管理、鉴权、历史消息

## 决策

选择 Centrifuge。理由：
- 成熟的连接管理和自动重连机制
- 服务端 SDK 覆盖 Go 和 JavaScript
- 支持频道级权限控制，适合多租户场景
- 自带消息历史，减少自建存储成本

## 后果

- **正面**：减少 2-3 周的自建 WebSocket 管理代码；开箱即用的鉴权机制
- **负面**：引入外部依赖；需要部署 Centrifugo 服务器
- **风险**：Centrifuge 社区活跃度需持续关注
```

### ADR 生命周期

| 状态 | 含义 |
|------|------|
| **提议中** | 正在讨论，尚未决定 |
| **已采纳** | 已决定并开始执行 |
| **已废弃** | 被新的 ADR 替代，注明替代 ADR 编号 |
| **已取代** | 同"已废弃" |

---

## 5. 文档即代码

文档不是写完就扔的一次性产物。它和代码一样，需要版本管理、review 和持续维护。

### 原则

- **与代码同步更新**：改代码时同步更新文档，而非"之后再补"
- **文档纳入 review**：PR 中的文档变更与代码变更同等审查
- **最小化文档**：只写必要的文档，避免冗余。能自动生成的不手写
- **单一事实来源**：同一信息只在一个地方维护，其他地方引用

### 自动化生成

| 文档类型 | 生成方式 |
|---------|---------|
| API 文档 | 从代码注释自动生成（Go: `godoc`，TS: `typedoc`） |
| OpenAPI 文档 | 手写维护（docs 仓库），或从代码注解生成 |
| CHANGELOG | 从 Conventional Commits 自动生成 |
| 架构图 | 从代码结构生成，或手写 Mermaid 图 |

---

## 6. 反模式

### ❌ 过时的文档

> _过时的文档比没有文档更糟——它给你虚假的信心。_

- 文档描述了已不存在的功能 → 删除或更新
- 代码示例无法运行 → 修复或删除
- **定期审计**：每个迭代花 30 分钟检查文档是否与代码一致

### ❌ 文档冗余

- 同一信息在 README、Wiki、注释中出现三次 → 选定一个事实来源
- README 里放了完整的 API 文档 → 链接到专门文档

### ❌ 文档与代码脱节

- 代码重构了，文档没更新 → 纳入 code review 检查项
- 新功能的 PR 没有附带文档 → PR 不完整，不应合并

---

## 总结

好的文档像好的代码一样——简洁、准确、易于维护。

| 文档类型 | 目标 |
|---------|------|
| README | 5 分钟内跑起来 |
| 注释 | 解释"为什么"，而非"是什么" |
| CHANGELOG | 每个版本的变化一目了然 |
| ADR | 决策理由可追溯、可讨论 |

> _"如果你不愿意维护文档，说明这个设计还不够清晰。"_
