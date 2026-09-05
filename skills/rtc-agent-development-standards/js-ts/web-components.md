# Web Components 规范（Lit）

> _组件是产品的积木。每一块积木都应当自包含、可组合、类型安全——让集成者像搭乐高一样使用它们。_

---

## 1. 组件结构

### 使用 TypeScript 装饰器

所有属性、状态、查询必须使用装饰器声明，不手动操作 `observedAttributes`。

```typescript
import { LitElement, html, css } from 'lit'
import { customElement, property, state, query } from 'lit/decorators.js'

@customElement('chat-panel')
export class ChatPanel extends LitElement {
  // ✅ 用装饰器
  @property({ type: String }) sessionId = ''
  @property({ type: Boolean, reflect: true }) disabled = false
  @state() private _messages: ChatMessage[] = []
  @query('.message-list') private _listEl!: HTMLElement
}
```

### 公开属性 vs 内部状态

| 装饰器 | 用途 | 示例 |
|--------|------|------|
| `@property` | 公开属性，外部可设置 | `sessionId`, `disabled` |
| `@state` | 内部状态，外部不可见 | `_messages`, `_loading` |

```typescript
// ✅ 分离清晰
@property({ type: String }) sessionId = ''      // 外部控制
@state() private _messages: ChatMessage[] = []   // 内部管理

// ❌ 用 @property 暴露内部状态
@property() private _loading = false  // 不应暴露
```

### 属性反射（reflect）谨慎使用

只有当外部需要通过 CSS 属性选择器或 JS 查询感知状态时，才启用 reflect。

```typescript
// ✅ 需要 CSS 选择器感知
@property({ type: Boolean, reflect: true }) disabled = false
// :host([disabled]) { opacity: 0.5; }

// ❌ 不需要反射
@property({ type: Boolean, reflect: true }) private _internal = false
```

### 始终提供默认值

所有属性必须有默认值，避免 `undefined` 传播。

```typescript
// ✅ 有默认值
@property({ type: String }) name = ''
@property({ type: Number }) count = 0
@property({ type: Boolean }) active = false

// ❌ 无默认值
@property({ type: String }) name!: string  // undefined 风险
```

---

## 2. 渲染规范

### `render()` 必须保持纯函数

`render()` 不产生副作用，不修改状态，不调用异步操作。

```typescript
// ✅ 纯函数
render() {
  return html`
    <div class="messages">
      ${this._messages.map(msg => html`<div>${msg.content}</div>`)}
    </div>
  `
}

// ❌ 在 render 中产生副作用
render() {
  this._loading = true  // 修改状态
  fetch('/api/data')     // 异步操作
  return html`...`
}
```

### 空内容用 `nothing`

```typescript
import { nothing } from 'lit'

// ✅ 用 nothing
render() {
  if (this._messages.length === 0) {
    return nothing
  }
  return html`...`
}

// ❌ 返回 null 或空字符串
render() {
  if (this._messages.length === 0) {
    return null  // 不推荐
  }
}
```

### 列表用 `repeat()` + key

```typescript
import { repeat } from 'lit/directives/repeat.js'

// ✅ repeat 保证高效的列表更新
render() {
  return html`
    <ul>
      ${repeat(this._messages, msg => msg.id, msg => html`
        <li>${msg.content}</li>
      `)}
    </ul>
  `
}
```

### 条件子树用 `cache()`

```typescript
import { cache } from 'lit/directives/cache.js'

// ✅ cache 保留条件分支的 DOM 状态
render() {
  return html`
    ${cache(
      this.view === 'list'
        ? html`<list-view .items=${this.items}></list-view>`
        : html`<detail-view .item=${this.selected}></detail-view>`
    )}
  `
}
```

### 派生状态在 `willUpdate()` 中计算

```typescript
// ✅ willUpdate 中计算派生状态
willUpdate(changed: PropertyValues) {
  if (changed.has('items')) {
    this._sortedItems = [...this.items].sort((a, b) => a.name.localeCompare(b.name))
  }
}

// ❌ 在 render 中计算（每次渲染都重新计算）
render() {
  const sorted = [...this.items].sort(...)  // 性能浪费
  return html`...`
}
```

---

## 3. 样式规范

### 必须使用 `static styles`

样式必须在组件内定义，不依赖外部样式表。

```typescript
import { css } from 'lit'

@customElement('chat-panel')
export class ChatPanel extends LitElement {
  static styles = css`
    :host {
      display: block;
      font-family: var(--chat-font-family, system-ui);
    }

    :host([hidden]) {
      display: none;
    }

    .message {
      padding: 8px 12px;
      border-radius: 8px;
    }
  `
}
```

### CSS 自定义属性用于主题

暴露 CSS 自定义属性，让外部控制主题。

```typescript
static styles = css`
  :host {
    --chat-bg: #ffffff;
    --chat-text: #333333;
    --chat-primary: #0066cc;

    background: var(--chat-bg);
    color: var(--chat-text);
  }

  .button {
    background: var(--chat-primary);
  }
`
```

### `::part()` 用于深层样式暴露

当外部需要精细控制内部样式时，使用 `part` 暴露。

```typescript
static styles = css`
  .header {
    /* 内部样式 */
  }
`

render() {
  return html`<div class="header" part="header">...</div>`
}

// 外部可以覆盖
// chat-panel::part(header) { background: red; }
```

---

## 4. 事件规范

### 必须使用 `composed: true`

事件默认不能穿越 Shadow DOM 边界。必须设置 `composed: true`。

```typescript
// ✅ composed: true
this.dispatchEvent(new CustomEvent('message-sent', {
  detail: { message },
  bubbles: true,
  composed: true,  // 必须
}))

// ❌ 默认 composed: false，外部无法监听
this.dispatchEvent(new CustomEvent('message-sent', {
  detail: { message },
}))
```

### 事件命名 kebab-case

```typescript
// ✅ kebab-case
this.dispatchEvent(new CustomEvent('message-sent'))
this.dispatchEvent(new CustomEvent('session-updated'))

// ❌ camelCase 或 PascalCase
this.dispatchEvent(new CustomEvent('messageSent'))
this.dispatchEvent(new CustomEvent('MessageSent'))
```

### `disconnectedCallback` 中清理监听器

```typescript
@customElement('chat-panel')
export class ChatPanel extends LitElement {
  private _resizeHandler = () => this._handleResize()

  connectedCallback() {
    super.connectedCallback()
    window.addEventListener('resize', this._resizeHandler)
  }

  disconnectedCallback() {
    super.disconnectedCallback()
    window.removeEventListener('resize', this._resizeHandler)  // 必须清理
  }
}
```

---

## 5. 生命周期

### `super()` 调用顺序

构造函数中 `super()` 必须是第一条语句。

```typescript
// ✅ 正确
constructor() {
  super()
  this._internal = 'init'
}

// ❌ 错误
constructor() {
  this._internal = 'init'  // super() 之前
  super()
}
```

### DOM 操作在 `firstUpdated` 中

```typescript
// ✅ firstUpdated 保证 DOM 已渲染
firstUpdated() {
  this._listEl.scrollTop = this._listEl.scrollHeight
}

// ❌ connectedCallback 中 DOM 可能未渲染
connectedCallback() {
  super.connectedCallback()
  this._listEl.scrollTop = 0  // _listEl 可能还是 undefined
}
```

### 异步操作使用 `updateComplete`

```typescript
// ✅ 等待渲染完成
async doSomething() {
  await this.updateComplete
  // DOM 已更新
  this._listEl.scrollTop = this._listEl.scrollHeight
}
```

---

## 6. 可访问性

### 交互式组件使用 `delegatesFocus`

```typescript
static styles = css`
  :host {
    display: block;
  }
`

// ✅ delegatesFocus 让焦点行为像原生元素
static shadowRootOptions: ShadowRootInit = {
  mode: 'open',
  delegatesFocus: true,
}
```

### ARIA 属性必须正确设置

```typescript
// ✅ 交互式组件必须有 ARIA
render() {
  return html`
    <button
      role="button"
      aria-label=${this.label}
      aria-disabled=${this.disabled}
      @click=${this._handleClick}
    >
      ${this.label}
    </button>
  `
}
```

### 表单组件使用 Form-Associated

```typescript
// ✅ Form-Associated Custom Element
@customElement('chat-input')
export class ChatInput extends LitElement {
  static formAssociated = true

  private _internals: ElementInternals

  constructor() {
    super()
    this._internals = this.attachInternals()
  }

  // 参与表单提交
  get form() { return this._internals.form }
  get name() { return this.getAttribute('name') }
  get value() { return this._value }
}
```

---

## 7. 性能约束

### 复杂类型自定义 `hasChanged`

```typescript
// ✅ 自定义比较逻辑，避免不必要的更新
@property({
  type: Array,
  hasChanged(newVal: string[], oldVal: string[]) {
    if (!oldVal) return true
    if (newVal.length !== oldVal.length) return true
    return newVal.some((v, i) => v !== oldVal[i])
  },
})
items: string[] = []
```

### 懒加载重依赖

```typescript
// ✅ 按需加载
async connectedCallback() {
  super.connectedCallback()
  if (this.needsHeavyFeature) {
    const { heavyModule } = await import('./heavy-module.js')
    this._heavy = new heavyModule()
  }
}
```

### 昂贵计算使用 memoization

```typescript
// ✅ 缓存计算结果
private _sortedCache?: { input: string[], output: string[] }

private _getSorted(items: string[]): string[] {
  if (this._sortedCache?.input === items) {
    return this._sortedCache.output
  }
  const sorted = [...items].sort()
  this._sortedCache = { input: items, output: sorted }
  return sorted
}
```

---

## 8. 新增组件检查清单

新组件上线前，必须确认：

- [ ] 使用 `@customElement` 注册
- [ ] 公开属性用 `@property`，内部状态用 `@state`
- [ ] 所有属性有默认值
- [ ] `render()` 是纯函数
- [ ] 样式用 `static styles` 封装
- [ ] 事件 `composed: true`
- [ ] 事件命名 kebab-case
- [ ] `disconnectedCallback` 清理监听器
- [ ] ARIA 属性正确设置
- [ ] 暴露 CSS 自定义属性用于主题
- [ ] 导出 TypeScript 类型

---

## 9. 组件间状态传播

### Lit Context 为共享状态首选

跨 Shadow DOM 的数据共享必须使用 `@lit/context`，不通过属性层层穿透。

```typescript
import { createContext } from '@lit/context'
import { ContextProvider } from '@lit/context'
import { consume } from '@lit/context/decorators.js'

// ✅ 定义 context（统一 {state, actions} 结构）
export interface SessionContextValue {
  state: {
    currentSessionId: string | null
    sessions: Session[]
  }
  actions: {
    switchSession: (id: string) => void
    createSession: () => void
  }
}

export const SessionContext = createContext<SessionContextValue>('rtc-session')

// ✅ 根组件创建 provider
@customElement('rtc-agent')
export class RTCAgent extends LitElement {
  private _sessionProvider = new ContextProvider(this, {
    context: SessionContext,
    initialValue: { state: {...}, actions: {...} },
  })

  updated() {
    this._sessionProvider.setValue(this._session.value)
  }
}

// ✅ 子组件消费 context
@customElement('rtc-input-area')
export class InputArea extends LitElement {
  @consume({ context: SessionContext, subscribe: true })
  @state()
  private _sessionCtx: SessionContextValue = { state: { currentSessionId: null, sessions: [] }, actions: { switchSession: () => {}, createSession: () => {} } }
}
```

### context 值结构

所有 context 值必须遵循 `{state, actions}` 结构：

- **state**：只读数据，消费方不可直接修改
- **actions**：修改 state 的方法集合

### 事件通信（子→根）

子组件向根组件通信使用 `CustomEvent`，必须设置 `bubbles: true, composed: true`。

```typescript
// ✅ 子组件 dispatch
this.dispatchEvent(new CustomEvent('rtc-input-submit', {
  bubbles: true,
  composed: true,
  detail: { content: 'hello' },
}))

// ✅ 根组件监听（在 connectedCallback 中注册，disconnectedCallback 中清理）
connectedCallback() {
  super.connectedCallback()
  this.addEventListener('rtc-input-submit', this._onInputSubmit)
}

disconnectedCallback() {
  super.disconnectedCallback()
  this.removeEventListener('rtc-input-submit', this._onInputSubmit)
}
```

### 事件命名规范

```text
rtc-<domain>-<action>
```

示例：`rtc-input-submit`、`rtc-new-session`、`rtc-window-minimize`、`rtc-fork-requested`

### Reactive Controller 模式

状态封装在 Reactive Controller 中，Controller 暴露 `{state, actions}` 给 context。

```typescript
// ✅ Controller 封装状态
class SessionController implements ReactiveController {
  private _state: SessionState = { currentSessionId: null, sessions: [] }

  get value() {
    return {
      state: this._state,
      actions: {
        switchSession: (id: string) => this._switchSession(id),
        createSession: () => this._createSession(),
      },
    }
  }

  private _switchSession(id: string) {
    this._state = { ...this._state, currentSessionId: id }
    this.host.requestUpdate()
  }
}
```

### 属性穿透边界

属性（`@property`）仅用于叶子组件，从直接父组件接收数据。中间层组件不传递属性，通过 Context 获取数据。

```typescript
// ✅ 叶子组件用属性
@customElement('rtc-user-message')
export class UserMessage extends LitElement {
  @property({ type: Object }) message: Message = { ... }
}

// 父组件传递
render() {
  return html`<rtc-user-message .message=${item.message}></rtc-user-message>`
}

// ❌ 中间层不要穿透属性
// rtc-content-wrapper 不应传递 message 给 rtc-content-area
```

### UIUpdateBus（持久化→UI）

持久化层的 IndexedDB 变更通过 `UIUpdateBus` 推送到 UI。根组件订阅 bus，分发到对应 Controller。

```typescript
// 根组件订阅
const bus = getUIUpdateBus()
this._busUnsub = bus.subscribe((event) => {
  if (event.entity === 'message') {
    this._messageController.reload(event.entityId)
  }
})

// 清理
disconnectedCallback() {
  this._busUnsub?.()
}
```

### 新增组件选择指南

| 场景 | 方式 |
|------|------|
| 多个组件需要共享状态 | Lit Context |
| 子组件通知根组件执行操作 | CustomEvent |
| 父组件传数据给直接子组件 | `@property` |
| 持久化层变更通知 UI | UIUpdateBus |
| Controller 之间协调 | 根组件通过回调串联 |

---

## 总结

好的组件像好的函数——输入明确、输出可预测、副作用可控。遵循这些规范，让组件库成为团队的基石，而非负担。

> _"组件是产品的原子。每个原子都应当稳定、可组合、可信赖。"_
