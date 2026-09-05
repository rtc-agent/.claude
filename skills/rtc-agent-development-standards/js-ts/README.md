# JS/TS 规范

> _适用于 `web-components` 仓库（TypeScript pnpm monorepo）。_

---

## 概述

本规范是 `web-components` 仓库的编写指南，覆盖 TypeScript 类型系统、Web Component 设计模式和样式规范三个维度。目标是让组件库的代码风格统一、类型安全、易于集成。

---

## 快速参考

| 领域 | 核心规则 |
|------|---------|
| **类型** | 严格模式；接口优于 type alias；避免 `any` |
| **命名** | 组件 PascalCase；变量/函数 camelCase；常量 UPPER_SNAKE_CASE |
| **格式** | Prettier 格式化；ESLint 检查；2 空格缩进 |
| **组件** | 单一职责；属性声明完整；事件类型化 |
| **样式** | CSS 变量；Shadow DOM 封装；响应式设计 |
| **测试** | 组件测试；属性/事件/状态覆盖 |

---

## 核心原则

### 类型即文档

TypeScript 的类型系统是第一道防线。好的类型定义让调用方不需要看实现就能正确使用。

```typescript
// ✅ 类型即文档
interface ChatMessage {
  id: string
  sessionId: string
  content: string
  role: 'user' | 'assistant' | 'system'
  timestamp: number
  metadata?: MessageMetadata
}

// ❌ 缺少类型
const message = {
  id: '123',
  content: 'hello',
  // role 是什么？timestamp 是什么？
}
```

### 组件内聚

一个组件应当是完整的、可独立使用的单元。它的状态、样式、行为封装在一起。

```typescript
// ✅ 内聚：状态、样式、行为封装
class ChatPanel extends HTMLElement {
  private messages: ChatMessage[] = []
  private shadow: ShadowRoot

  constructor() {
    super()
    this.shadow = this.attachShadow({ mode: 'open' })
    this.render()
  }

  // 公开 API 清晰
  addMessage(msg: ChatMessage): void { ... }
  clearMessages(): void { ... }
}

// ❌ 分散：状态和逻辑散落在外部
const messages = []
function addMessage(msg) { messages.push(msg); render() }
function render() { /* 操作 DOM */ }
```

### 纯函数优先

将业务逻辑写成纯函数，副作用推到边界。

```typescript
// ✅ 纯函数
function formatTimestamp(ts: number): string {
  return new Date(ts).toLocaleString()
}

function filterActiveSessions(sessions: Session[]): Session[] {
  return sessions.filter(s => s.status === 'active')
}

// ❌ 副作用混在逻辑中
function processSessions(sessions: Session[]) {
  sessions.forEach(s => {
    if (s.status === 'active') {
      document.querySelector('#list').append(createItem(s))
    }
  })
}
```

### 副作用边界化

DOM 操作、网络请求、事件监听等副作用集中在组件的生命周期方法中，不散落在业务逻辑里。

```typescript
class ChatPanel extends HTMLElement {
  connectedCallback() {
    // 副作用集中在这里
    this.setupEventListeners()
    this.loadInitialData()
  }

  disconnectedCallback() {
    this.cleanup()
  }

  // 业务逻辑是纯函数
  private formatMessages(messages: ChatMessage[]): string {
    return messages.map(m => m.content).join('\n')
  }
}
```

---

## 子文档导航

| 文档 | 内容 |
|------|------|
| [TypeScript 规范](./typescript.md) | 类型系统、接口设计、泛型、避免 `any` |
| [Web Components](./web-components.md) | 组件设计、生命周期、Shadow DOM、属性与事件 |
| [样式规范](./styling.md) | CSS 变量、Shadow DOM 样式、主题系统、GSAP 动画 |
| [IndexedDB 规范](./indexeddb.md) | Dexie.js、持久化策略、同步模型、VirtualFS |
| [SVG 规范](./svg.md) | 图标来源、清洗流程、样式约束、尺寸档位 |
| [可访问性](./accessibility.md) | ARIA、键盘导航、焦点管理、对比度、屏幕阅读器 |
| [前端性能](./performance.md) | Bundle 优化、渲染性能、网络、内存管理 |
| [前端错误处理](./error-handling.md) | 错误边界、错误分类、用户提示、错误上报 |
| [Monorepo 规范](./monorepo.md) | pnpm workspace、包组织、共享配置、包间依赖 |

---

## 工具链

| 工具 | 用途 | 命令 |
|------|------|------|
| ESLint | 代码质量检查 | `pnpm lint` |
| Prettier | 代码格式化 | `pnpm format` |
| TypeScript | 类型检查 | `pnpm typecheck` |
| vitest | 单元测试 | `pnpm test` |

所有工具应在 pre-commit hook 中自动执行。
