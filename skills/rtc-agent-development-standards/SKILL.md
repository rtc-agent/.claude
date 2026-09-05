---
name: rtc-agent-development-standards
description: Use when writing, reviewing, or refactoring code across any RTC Agent repository (server, web-components, docs, .claude). Covers code style, architecture, git workflow, testing, and documentation standards for Go and TypeScript/TypeScript Web Component stacks.
---

# RTC Agent 开发规范

> _"像诗一样编写代码"_ — 代码不仅是给机器执行的指令，更是写给人阅读的作品。

## 核心哲学

**简洁胜于冗长** — 每一行代码都应该有存在的理由。能用一行说清的，不写两行。能用一个词命名的，不用一个句子。

**可读性即正义** — 代码被阅读的次数远多于被编写的次数。选择让读者理解的写法，而非让作者炫技的写法。

**一致性即秩序** — 同一个项目里，代码看起来应该像同一个人写的。风格统一，命名规律，模式可预测。

**意图自解释** — 好的代码不需要注释来解释"为什么"。命名即文档，结构即逻辑，模式即意图。

**渐进式优雅** — 不追求一步到位的完美，但追求每次提交都比上次更好。童子军原则：离开时比来时更干净。

**开源即责任** — 五个仓库全部开源。每一行代码都是面向公众的作品，接受世界的审视。以匠人之心打磨，以开放之姿示人。

---

## 仓库速查

| 仓库 | 路径 | 技术栈 | 主要适用规范 |
|------|------|--------|-------------|
| **server** | `./server` | Go, Anthropic SDK, Eino, Centrifuge | [Go 规范](./golang/README.md) |
| **web-components** | `./web-components` | TypeScript, pnpm monorepo | [JS/TS 规范](./js-ts/README.md) |
| **docs** | `./docs` | OpenAPI 3.1 | [API 文档规范](#api-文档规范) |
| **mermaid-live-editor** | `./mermaid-live-editor` | JavaScript | [JS/TS 规范](./js-ts/README.md) |
| **.claude** | `./.claude` | Claude Code 配置 | 通用规范 |

---

## 通用规范

适用于所有仓库的基础准则。

| 文档 | 说明 |
|------|------|
| [通用原则](./01-universal-principles.md) | 命名、注释、DRY/KISS/SOLID、代码组织 |
| [Git 工作流](../rtc-agent-commitment/SKILL.md) | 提交信息格式、分支策略、PR 流程、版本发布（独立 skill） |
| [测试规范](./02-testing-standards.md) | 测试金字塔、覆盖率目标、测试命名与组织 |
| [文档规范](./03-documentation-standards.md) | README、内联注释、变更日志、架构决策记录 |
| [API 文档规范](./04-api-documentation.md) | OpenAPI 3.1 编写规范、示例设计、文档版本管理 |
| [安全规范](./05-security-standards.md) | XSS、Token 安全、输入校验、CSRF、敏感信息 |

---

## Go 语言规范

专用于 `server` 仓库的 Go 代码标准。

| 文档 | 说明 |
|------|------|
| [Go 规范索引](./golang/README.md) | Go 规范总览与快速入门 |
| [代码风格](./golang/code-style.md) | gofmt、命名约定、包组织、导入分组 |
| [错误处理](./golang/error-handling.md) | 错误传播、自定义错误类型、panic 使用边界 |
| [并发规范](./golang/concurrency.md) | goroutine 管理、channel 模式、context 使用 |
| [项目结构](./golang/project-structure.md) | 目录布局、模块划分、依赖管理 |
| [可观测性](./golang/observability.md) | 日志、追踪、指标采集、健康检查 |
| [代码质量](./golang/code-quality.md) | 重复代码、废弃代码、事务上下文、Redis Lua、PR 自检 |
| [后端性能](./golang/performance.md) | 数据库优化、并发、缓存、Centrifuge、pprof |

---

## JS/TS 规范

专用于 `web-components` 仓库的 TypeScript / Web Component 标准。

| 文档 | 说明 |
|------|------|
| [JS/TS 规范索引](./js-ts/README.md) | JS/TS 规范总览与快速入门 |
| [TypeScript 规范](./js-ts/typescript.md) | 类型系统、接口设计、泛型使用、类型安全 |
| [Web Components](./js-ts/web-components.md) | Lit 组件设计、生命周期、Shadow DOM、属性与事件 |
| [样式规范](./js-ts/styling.md) | 容器边界、z-index 层级、Floating UI、CSS 变量、GSAP 动画 |
| [IndexedDB 规范](./js-ts/indexeddb.md) | Dexie.js、持久化策略、同步模型、VirtualFS |
| [SVG 规范](./js-ts/svg.md) | 图标来源、清洗流程、样式约束、尺寸档位 |
| [可访问性](./js-ts/accessibility.md) | ARIA、键盘导航、焦点管理、对比度、屏幕阅读器 |
| [前端性能](./js-ts/performance.md) | Bundle 优化、渲染性能、网络、内存管理 |
| [前端错误处理](./js-ts/error-handling.md) | 错误边界、错误分类、用户提示、错误上报 |
| [Monorepo 规范](./js-ts/monorepo.md) | pnpm workspace、包组织、共享配置、包间依赖 |

---

## 何时查阅本规范

- ✍️ **编写新代码时** — 确保风格一致、模式正确
- 🔍 **代码审查时** — 以规范为准绳，而非个人偏好
- 🏗️ **架构设计时** — 遵循已确立的原则和模式
- 📚 **编写文档时** — 保持文档风格统一
- 🐛 **调试修复时** — 修复代码同样需要符合规范
- 🌱 **重构优化时** — 以"诗一般优雅"为目标

---

## 规范的本质

规范不是束缚，而是共识。它让团队中的每个人——包括未来的自己——都能快速理解代码的意图，减少认知负担，把精力集中在真正需要创造力的地方。

> _"好的代码是一首诗：每一行都有意义，每一个词都恰到好处。"_
