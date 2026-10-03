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

### 全局类型声明（HTMLElementTagNameMap）

每个自定义元素必须在全局 `HTMLElementTagNameMap` 中注册类型映射，使 `document.createElement('rtc-xxx')` 和模板中能获得正确的类型推导。

```typescript
// ✅ 在组件文件末尾声明
declare global {
  interface HTMLElementTagNameMap {
    'rtc-chat-panel': ChatPanel
  }
}

// ✅ 集中声明（适用于跨包使用）
// packages/component/src/elements.ts
import type { RtcChatPanel } from './components/chat-panel/chat-panel'
import type { RtcDrawer } from './components/drawer/rtc-drawer'
// ...

declare global {
  interface HTMLElementTagNameMap {
    'rtc-chat-panel': RtcChatPanel
    'rtc-drawer': RtcDrawer
    // 所有自定义元素集中注册
  }
}
```

**约束**：

- 标签名必须与 `@customElement('rtc-xxx')` 完全一致
- 类型必须是组件类本身，不能是 `any` 或 `HTMLElement`
- 集中声明文件（如 `elements.ts`）优先于分散声明，便于维护

**防止的问题**：

- `document.createElement('rtc-chat-panel')` 返回 `HTMLElement`，无法访问组件 API
- TypeScript 无法推导组件属性类型，集成者被迫手动断言

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

### Stale Async Guard — await 后验证身份

异步操作（token refresh、Worker 连接、API 调用）的 `await` 期间，组件状态可能已被外部改变（销毁、重置、替换 provider）。`await` 返回后，必须验证关键引用仍指向当前实例，否则用过期数据更新状态会导致幽灵写入或竞态。

```typescript
// ✅ await 后验证身份
void provider.refreshToken().then(result => {
  // Guard: 如果 _authProvider 在 await 期间被替换（logout/destroy），跳过
  if (this._authProvider !== provider) {
    log.debug('Auth provider changed during refresh, skipping')
    return
  }
  this._state = { isLoggedIn: true, accessToken: result.accessToken }
  this.host.requestUpdate()
})

// ✅ 通用模式：await 后检查组件是否仍活跃
async _loadData() {
  const sessionId = this.sessionId  // 捕获当前值
  const data = await api.fetch(sessionId)
  // await 期间 sessionId 可能已被改变或组件已 disconnected
  if (this.sessionId !== sessionId) return  // 身份不匹配，丢弃
  if (!this.isConnected) return              // 组件已卸载，丢弃
  this._data = data
}

// ❌ 不验证，用过期数据覆盖当前状态
async _loadData() {
  const data = await api.fetch(this.sessionId)
  this._data = data  // await 期间 sessionId 可能已变，数据对应旧的 session
}
```

**适用场景**：

- Auth token refresh（provider 可能在 await 期间被替换）
- Worker 连接验证（Worker 可能在 await 期间被销毁）
- 数据加载（组件可能在 await 期间被卸载或切换 session）
- 任何 "捕获状态 → await → 使用结果" 的模式

**判断标准**：`await` 前捕获的值，在 `await` 后是否仍有效？如果外部可能改变该值（通过属性变更、组件销毁、provider 替换），就需要 guard。

### `updated()` 中的延迟状态变更 — `queueMicrotask`

在 `updated()` 生命周期中修改 `@property` 或 `@state` 会触发 Lit 的 "change-in-update" 警告，因为 Lit 正在执行更新循环，同步修改状态会导致无限循环或性能问题。使用 `queueMicrotask()` 将状态变更推迟到当前更新循环结束后执行。

```typescript
// ✅ 在 updated() 中用 queueMicrotask 延迟状态变更
updated(changedProperties: PropertyValues) {
  super.updated(changedProperties)
  if (changedProperties.has('sessionId')) {
    // 同步修改会触发 Lit 警告，推迟到下一微任务
    queueMicrotask(() => {
      this._messages = []
      this._inputValue = ''
    })
  }
}

// ✅ 适用于响应属性变化需要同步重置内部状态的场景
updated(changedProperties: PropertyValues) {
  super.updated(changedProperties)
  if (changedProperties.has('activeTab')) {
    queueMicrotask(() => {
      this._scrollPosition = 0
      this._selectedItems = []
    })
  }
}

// ❌ 在 updated() 中同步修改状态
updated(changedProperties: PropertyValues) {
  super.updated(changedProperties)
  if (changedProperties.has('sessionId')) {
    this._messages = []  // Lit: "change-in-update" 警告
    this._inputValue = ''  // 可能触发无限循环
  }
}
```

**为什么用 `queueMicrotask` 而不是 `setTimeout`**：

- `queueMicrotask` 在当前宏任务结束后立即执行（微任务队列），保证状态变更在下一帧渲染前完成
- `setTimeout` 推迟到下一个宏任务（至少 4ms 后），可能导致 UI 闪烁或状态不一致

**适用场景**：

- 属性变化后需要重置多个内部状态
- 响应外部 prop 需要同步更新关联的内部状态
- 避免 Lit 的 "change-in-update" 警告

**判断标准**：在 `updated()` 中是否需要修改 `@property` 或 `@state`？如果需要，用 `queueMicrotask` 包裹。

### 自包含组件：通过 Context Provider 获取外部数据

当组件需要访问外部数据（如文件存储、认证状态、配置）时，通过 Context Provider 模式获取，而非通过属性层层传递（prop drilling）。组件内部自行从 context 读取所需数据，对外保持自包含。

```typescript
// ✅ 通过 Context Provider 获取外部数据（自包含）
// 组件内部自行从 FileStorageContext 获取文件存储能力
// 外部使用者无需关心文件存储细节
import { FileStorageContext, type FileStorageContextValue } from '../../contexts/file-storage.js'

@customElement('rtc-file-thumbnail')
export class RtcFileThumbnail extends LitElement {
  static context = FileStorageContext

  private async _loadThumbnail() {
    const fileStorage = this.context.fileStorage
    const blob = await fileStorage.getThumbnail(this.fileId)
    // ...
  }
}

// 父组件提供 context
@customElement('rtc-file-preview-area')
export class RtcFilePreviewArea extends LitElement {
  render() {
    return html`
      <file-storage-context-provider .value=${this._storageContext}>
        <rtc-file-thumbnail .fileId=${file.fileid}></rtc-file-thumbnail>
      </file-storage-context-provider>
    `
  }
}

// ❌ Prop drilling：每个中间组件都要传递 fileStorage
<parent .fileStorage=${storage}>
  <child .fileStorage=${storage}>
    <thumbnail .fileStorage=${storage}></thumbnail>  // 中间层只是转发
  </child>
</parent>
```

**适用场景**：

- 文件存储访问（缩略图加载、文件上传状态）
- 认证状态（token、用户信息）
- 全局配置（主题、语言、功能开关）

**判断标准**：当一个数据需要被 3 层以上组件使用时，用 Context Provider 替代 prop drilling。仅被 1-2 层使用的数据直接通过 `@property` 传递。

### 生命周期取消：AbortController 清理长时间操作

组件销毁（`destroy()` / `disconnectedCallback()`）时，进行中的长时间操作（catch-up 回放、数据加载、文件上传）必须被取消。使用 `AbortController` 在生命周期结束时中止操作，防止在已销毁的组件上更新状态或浪费资源。

```typescript
// ✅ AbortController 取消长时间操作
@customElement('rtc-worker-bridge')
export class WorkerBridge {
  private _catchUpAbortController?: AbortController;

  async _doCatchUp(): Promise<void> {
    // 取消前一个未完成的 catch-up
    this._catchUpAbortController?.abort();
    this._catchUpAbortController = new AbortController();
    const signal = this._catchUpAbortController.signal;

    let fromSeq = this._lastProcessedSeq;
    let hasMore = true;

    while (hasMore && !signal.aborted) {
      const result = await this._core.getCatchUpEvents(fromSeq);
      // 每个 await 后检查 signal
      if (signal.aborted) break;

      for (const entry of result.entries) {
        if (entry.seq <= this._lastProcessedSeq) continue;
        this._deliverEvent(entry);
      }
      fromSeq = result.entries[result.entries.length - 1]?.seq ?? fromSeq;
      hasMore = result.hasMore;
    }
  }

  async destroy(): Promise<void> {
    // 销毁时取消进行中的 catch-up
    this._catchUpAbortController?.abort();
    this._catchUpAbortController = undefined;
    // ... 其他清理
  }
}

// ❌ 不取消，destroy 后 catch-up 仍在运行
async destroy(): Promise<void> {
  // _doCatchUp 可能仍在执行，回调会操作已销毁的组件
  this._callbacks = null;  // catch-up 循环中访问 this._callbacks 会出错
}
```

**约束**：

- `destroy()` / `disconnectedCallback()` 中必须 abort 所有进行中的受控操作
- AbortController 在每次新操作开始时重新创建（先 abort 旧的，再创建新的）
- 循环内的每个 `await` 之后检查 `signal.aborted`
- 被 abort 的操作应静默退出（不抛错、不触发回调），因为组件已销毁

**适用场景**：

- Worker catch-up 回放（可能涉及多页数据加载）
- 批量文件操作（上传、下载）
- 长时间轮询或数据流

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

### 大文件约束

新文件不超过 500 行。超过 500 行的文件难以理解和维护。测试文件放宽到 800 行——表驱动测试和多个场景天然产生更多代码。

**例外：高内聚的工具/仓库文件**。当文件是单一职责的工具类或仓库类（如 `FileStorage`、`FileCacheRepository`），且逻辑紧密耦合不适合拆分时，可以超过 500 行。但必须在文件顶部用注释说明"为什么是单一大文件"。

```typescript
// ✅ 文件顶部注释说明为什么是大文件
/**
 * File Storage Utility
 *
 * High-level API for file operations with automatic MD5 calculation,
 * caching, and progress tracking. Wraps the WorkerBridge file operations.
 *
 * This file is intentionally large (900+ lines) because it provides a cohesive
 * API surface for file management. Splitting into multiple files would scatter
 * related functionality (upload, download, cache, batch operations) and make
 * the API harder to discover and maintain.
 */
export class FileStorage { ... }

// ❌ 无注释的大文件
// 900+ 行代码，没有说明为什么不能拆分
```

**拆分策略**：

| 拆分方式 | 适用场景 | 示例 |
|---------|---------|------|
| **按功能** | 文件包含多个独立功能 | `storage.ts` → `upload.ts` + `download.ts` + `cache.ts` |
| **按类型** | 文件包含多种类型的定义 | `types.ts` → `interfaces.ts` + `models.ts` |
| **提取 helpers** | 辅助函数占比过大 | `storage.ts` → `storage.ts` + `helpers.ts` |

**拆分信号**：

- 需要频繁滚动才能理解上下文
- 同一文件中存在不相关的改动
- 函数之间没有明显的逻辑分组
- 文件超过 800 行且无顶部注释说明原因

**判断标准**：文件中的函数是否都服务于同一个核心职责？如果是，保持单一大文件并添加顶部注释；如果不是，按功能拆分。

### 文件组织：`contexts/` 目录

所有 Lit Context 定义集中在 `src/contexts/` 目录下，每个 context 一个文件。组件通过 `import { XxxContext } from '../contexts/xxx.js'` 消费。不在组件文件内部内联定义 context。

```
packages/component/src/contexts/
├── auth.ts              # AuthContext
├── file-storage.ts      # FileStorageContext（服务注入）
├── session.ts           # SessionContext
├── mode.ts              # ModeContext
└── ...
```

**约束**：

- 新增 context 时在 `contexts/` 目录创建独立文件
- 文件名与 context 名对应（`file-storage.ts` -> `FileStorageContext`）
- 组件只 import context 定义，不自己创建 context

### Blob URL 生命周期

通过 `URL.createObjectURL()` 创建的 blob URL 必须在组件销毁时释放，否则会造成内存泄漏。

```typescript
// ✅ disconnectedCallback 中释放 blob URL
private _thumbnailUrl: string = ''

disconnectedCallback(): void {
  super.disconnectedCallback()
  if (this._thumbnailUrl.startsWith('blob:')) {
    URL.revokeObjectURL(this._thumbnailUrl)
    this._thumbnailUrl = ''
  }
}

// ❌ 不释放，blob URL 持续占用内存
```

**约束**：

- 创建 blob URL 的组件必须在 `disconnectedCallback` 中调用 `URL.revokeObjectURL()`
- 如果 blob URL 由服务层统一管理（如 FileStorage），组件调用服务的释放方法而非直接 revoke
- 与 native `loading="lazy"` 配合使用时，lazy 属性处理可视性，blob URL 生命周期仍由组件管理

### 服务层共享 Blob URL 缓存（引用计数）

当多个叶子组件（如 `rtc-file-thumbnail`、`rtc-file-preview-modal`）需要同一个文件的 blob URL 时，服务层（如 `FileStorage`）维护共享缓存 + 引用计数，避免重复创建 blob URL。组件通过 `getThumbnailUrl()` / `releaseThumbnailUrl()` 与缓存交互，不直接管理 blob URL 生命周期。

```typescript
// ✅ 服务层：共享缓存 + 引用计数（FileStorage 内部）
private _blobCache = new Map<string, { url: string; refCount: number }>()
private _inflight = new Map<string, Promise<string>>()

async getThumbnailUrl(key: string): Promise<string> {
  // 1. 缓存命中：递增引用计数，直接返回
  const cached = this._blobCache.get(key)
  if (cached) { cached.refCount++; return cached.url }

  // 2. 进行中的请求：复用 Promise（去重）
  const inflight = this._inflight.get(key)
  if (inflight) return inflight

  // 3. 创建新 blob URL
  const promise = this._createBlobUrl(key)
  this._inflight.set(key, promise)
  const url = await promise
  this._inflight.delete(key)
  this._blobCache.set(key, { url, refCount: 1 })
  return url
}

releaseThumbnailUrl(url: string): void {
  for (const [key, entry] of this._blobCache) {
    if (entry.url === url) {
      entry.refCount--
      if (entry.refCount <= 0) {
        URL.revokeObjectURL(url)
        this._blobCache.delete(key)
      }
      return
    }
  }
}

// ✅ 组件层：获取 + 释放，不直接 revoke
@customElement('rtc-file-thumbnail')
export class FileThumbnail extends LitElement {
  private _thumbnailUrl = ''

  async updated(changed: PropertyValues) {
    if (changed.has('md5')) {
      this._thumbnailUrl = await fileStorage.getThumbnailUrl(key)
    }
  }

  disconnectedCallback() {
    super.disconnectedCallback()
    // 通过服务层释放，服务层决定何时真正 revoke
    fileStorage.releaseThumbnailUrl(this._thumbnailUrl)
  }
}

// ❌ 每个组件独立创建 blob URL，同一文件 N 个缩略图 = N 个 blob URL
```

**约束**：

- 服务层必须实现引用计数（`refCount`），只有计数归零时才真正 `URL.revokeObjectURL()`
- 进行中的请求通过 `_inflight` Promise 去重，防止并发创建多个相同 blob URL
- 组件层只调用 `getThumbnailUrl()` / `releaseThumbnailUrl()`，不直接操作 blob URL
- 此模式适用于"多个消费者共享同一资源"的场景；单一消费者场景直接用基本 blob URL 生命周期即可

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

### 动态 import 必须有重试策略

动态 `import()` 在开发环境（HMR 重建导致 chunk hash 变化）和生产环境（网络抖动）都可能失败。所有动态 import 必须实现重试，且使用 `_modulesPromise` 缓存模式避免重复加载。

```typescript
// ✅ 带重试的动态 import + Promise 缓存
private _modulesPromise: Promise<Modules> | null = null

private async _loadModules(attempts = 2): Promise<Modules> {
  try {
    const [mod1, mod2] = await Promise.all([
      import('heavy-lib-1'),
      import('heavy-lib-2'),
    ])
    return { lib1: mod1.default, lib2: mod2.default }
  } catch (err) {
    if (attempts > 0) {
      console.warn('[component-name] Module load failed, retrying...', err)
      this._modulesPromise = null  // 清除缓存，强制重新加载
      return this._loadModules(attempts - 1)
    }
    console.error('[component-name] Module load failed after retries:', err)
    throw err
  }
}

// 使用时通过缓存的 Promise 获取
private async _getModules(): Promise<Modules> {
  if (!this._modulesPromise) {
    this._modulesPromise = this._loadModules()
  }
  return this._modulesPromise
}

// ❌ 无重试，HMR 后直接崩溃
private async _loadModules() {
  const { default: mod } = await import('heavy-lib')
  return mod
}
```

**规则**：

- 默认重试 2 次（`attempts = 2`），覆盖 HMR 导致的 stale chunk 场景
- 重试前清除 `_modulesPromise` 缓存，避免重复返回失败的 Promise
- 最终失败时抛出错误，由调用方决定降级策略

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

## 8. SharedWorker 生命周期

SharedWorker 独立于页面生命周期存在——关闭 tab 不会终止它，强制杀 tab 后它可能残留旧状态。因此连接 SharedWorker 需要额外的验证和防护。

### 开发环境隔离 Worker 名称

开发模式下 HMR 重建可能导致旧 Worker 持有过期代码。通过在 Worker 名称中附加时间戳，确保每次开发会话创建新的 Worker 实例。

```typescript
// ✅ 开发模式隔离
private static _getWorkerName(): string {
  const baseName = 'rtc-agent-worker'
  if (import.meta.env?.DEV) {
    return `${baseName}-dev-${Date.now()}`  // 每次会话唯一
  }
  return baseName  // 生产环境固定名称，多 tab 共享
}

// ❌ 固定名称，开发时可能连接 stale worker
new SharedWorker('/worker.js', { name: 'my-worker' })
```

### Vite Worker 文件名 Hash 格式

Vite 构建 Worker 时，输出文件名的 hash 部分包含下划线和连字符（如 `shared-worker-Bw4_Rekb.js`）。任何匹配 worker 文件名的脚本或配置必须使用 `[\w-]+` 而非 `[A-Za-z0-9]+`。

```javascript
// ✅ setup 脚本 — 匹配 Vite hash 的完整字符集
const hashedWorkerFile = workerFiles.find(
  f => f !== 'shared-worker.js' && f.match(/shared-worker-[\w-]+\.js/)
);

// ❌ 只匹配字母数字，Vite hash 含下划线时匹配失败
const hashedWorkerFile = workerFiles.find(
  f => f !== 'shared-worker.js' && f.match(/shared-worker-[A-Za-z0-9]+\.js/)
);
```

**约束**：

- 新增涉及 worker 文件名匹配的脚本/配置时，正则必须使用 `[\w-]+` 匹配 hash 部分
- 如果 worker 文件未找到，打印可用文件列表辅助诊断（`console.log('Available:', files)`)
- 构建后检查 `dist/` 中 worker 文件名格式是否符合预期

### 连接后必须验证存活

创建 SharedWorker 后，必须通过 `ping`/`pong` 验证 Worker 是否响应，超时则给出明确的恢复提示。

```typescript
// ✅ 带超时的存活验证
private async _verifyWorkerAlive(): Promise<void> {
  const result = await Promise.race([
    this._core.ping(),
    new Promise<never>((_, reject) =>
      setTimeout(() => reject(new Error(
        'Worker verification timed out. ' +
        'This may indicate a stale SharedWorker from a previous session. ' +
        'To resolve: visit chrome://inspect/#workers and terminate stale workers.'
      )), VERIFICATION_TIMEOUT_MS)
    ),
  ])
  if (result !== 'pong') throw new Error('Worker health check failed')
}

// ❌ 不验证，假设 Worker 一定可用
this._core = Comlink.wrap<WorkerCore>(port)
// 直接使用，可能连到 stale worker
```

### 初始化用 Promise 去重

`init()` 可能被多次调用（如 React StrictMode double-mount）。用 `_initPromise` 缓存防止重复初始化，并在完成后清理。

```typescript
// ✅ Promise 去重
private _initPromise?: Promise<void>

async init(): Promise<void> {
  if (this._initPromise) return this._initPromise
  this._initPromise = this._doInit()
  try {
    await this._initPromise
  } finally {
    this._initPromise = undefined
  }
}

// ❌ 不防护，多次 init 导致多份 Worker
async init() {
  await this._doInit()  // 每次调用都执行
}
```

### destroy 后重置状态

`destroy()` 必须清理所有连接状态并重置标志位，使组件可以重新 `init()`。

```typescript
// ✅ 完整清理
async destroy(force: boolean) {
  this._connectionListeners.clear()
  this._initPromise = undefined
  this._verified = false
  // ... 清理 Worker 连接
}
```

### 区分临时卸载与永久销毁

组件可能被临时卸载（React StrictMode double-mount、路由切换）或永久销毁（tab 关闭、`beforeunload`）。`destroy()` 的参数应区分这两种场景——临时卸载时保留可恢复的状态（如 sessionStorage 游标），避免不必要的重建开销。

```typescript
// ✅ 区分临时与永久
async destroy(clearStorage = false) {
  // 两种场景都清理的：连接、监听器、Promise 缓存
  this._connectionListeners.clear()
  this._initPromise = undefined

  // 仅永久销毁时清理的：持久化游标、缓存数据
  if (clearStorage) {
    sessionStorage.removeItem(this._cursorKey)
    this._hasRestoredFromStorage = false
  }
  // clearStorage = false 时保留游标，重新 init() 可跳过 catch-up
}

// 使用场景
disconnectedCallback() {
  // 临时卸载：保留游标，下次挂载可快速恢复
  this._bridge?.destroy(false)
}

// beforeunload 时
window.addEventListener('beforeunload', () => {
  // 永久销毁：清理所有状态
  this._bridge?.destroy(true)
})

// ❌ 不区分，每次卸载都清游标
async destroy() {
  sessionStorage.removeItem(this._cursorKey)  // 重新挂载后需要重新 catch-up
}
```

**判断标准**：清理操作是否可逆？可恢复的数据（游标、缓存）在临时卸载时保留；不可恢复的资源（连接、监听器）始终清理。

### MasterLock — 主 Tab 选举（Web Locks API）

当多个 tab 共享同一个 SharedWorker 时，需要一个 tab 作为"主 tab"负责管理 WebSocket 连接等全局资源。`MasterLock` 基于 Web Locks API 实现主 tab 选举，确保同一用户下只有一个 tab 持有 master 角色。

```typescript
// ✅ MasterLock：基于 Web Locks API 的主 tab 选举
import { MasterLock } from './master-lock.js'

const masterLock = new MasterLock(`user-${userId}`)
masterLock.onAcquire = () => {
    // 成为主 tab：建立 WebSocket 连接
    void connectToWorker()
}
masterLock.onRelease = () => {
    // 失去主 tab 身份：断开 WebSocket
    disconnectFromWorker()
}
await masterLock.acquire()  // 开始尝试获取锁（可能排队）

// Tab 关闭时浏览器自动释放锁 → 其他 tab 自动升级
// 无需手动处理 tab 崩溃场景

// ❌ 自行实现 tab 选举（如 BroadcastChannel 投票）
// 浏览器已提供 Web Locks API，不需要重新发明
```

**设计原则**：

- 每个 tab 持有自己的 `MasterLock` 实例；tab 自身决定是否为主 tab
- Worker 不参与选举——它们不知道也不关心谁是 master
- Tab 关闭 → 浏览器自动释放锁 → 其他 tab 按队列顺序获取 → 自动升级
- 锁名包含 `userId`，实现多用户隔离（不同用户的 tab 互不干扰）

**约束**：

- 需要主 tab 选举的场景必须使用 `MasterLock`，不自行实现选举机制
- `onRelease` 回调必须清理 master 专属资源（WebSocket 连接等），防止资源泄漏
- `acquire()` 是异步的——不要在 `connectedCallback` 中同步依赖 master 状态

---

## 9. 新增组件检查清单

新组件上线前，必须确认：

- [ ] 使用 `@customElement` 注册
- [ ] 公开属性用 `@property`，内部状态用 `@state`
- [ ] 所有属性有默认值
- [ ] `render()` 是纯函数
- [ ] 样式用 `static styles` 封装
- [ ] 事件 `composed: true`
- [ ] 事件命名 kebab-case
- [ ] `disconnectedCallback` 清理监听器
- [ ] `disconnectedCallback` 释放 blob URL（如有）
- [ ] `disconnectedCallback` 清理 teleported 元素（如有 DOM Teleport）
- [ ] 通过服务层共享 blob URL 时，使用 `getThumbnailUrl()`/`releaseThumbnailUrl()` 引用计数接口
- [ ] ARIA 属性正确设置
- [ ] 暴露 CSS 自定义属性用于主题
- [ ] 导出 TypeScript 类型
- [ ] 用户可见文本使用 `msg()` 标记（i18n）
- [ ] 日志使用 `createLogger` 而非 `console`
- [ ] 新增 Lit Context 定义放在 `contexts/` 目录
- [ ] 新文件不超过 500 行（测试文件 800 行；高内聚工具/仓库文件可例外，但需在顶部注释说明原因）
- [ ] 根组件超过 500 行时，业务逻辑提取到 `helpers/` 子目录
- [ ] 对话框 overlay 使用纯函数工厂模式（返回 Promise，自动清理 DOM）
- [ ] 文件上传组件实现三阶段管道（本地缓存 → 即时显示 → 后台同步）
- [ ] 文件上传组件保存 `File` 对象供 retry 使用，支持离线重试
- [ ] 文件上传状态通过不可变 `Map` 更新追踪（`idle → loading → loaded | error`）

---

## 10. 组件间状态传播

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

**例外：服务注入 context**。当 context 用于注入服务实例（而非状态管理）时，直接暴露服务引用。判断标准：context 的值是"一个有自己方法和生命周期的对象"还是"一组状态和修改状态的函数"。

```typescript
// ✅ 标准 {state, actions}：状态管理
export interface SessionContextValue {
  state: SessionState
  actions: { switchSession: (id: string) => void; createSession: () => void }
}

// ✅ 服务注入例外：注入服务实例
export interface FileStorageContextValue {
  fileStorage: FileStorage | null  // 服务实例，不是状态
}

// ✅ 简单例外：只有 1-2 个动作时，动作可平铺在顶层
export interface AuthContextValue {
  state: AuthState
  login: () => void   // 不需要 actions 包装
  logout: () => void
}
```

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

**文件组织**：Controller 文件放在 `controllers/` 目录下，命名格式为 `<name>.controller.ts`。

```
packages/component/src/controllers/
├── auth.controller.ts           # AuthController
├── session-tree.controller.ts   # SessionTreeController
├── mode.controller.ts           # ModeController
├── event-binding.controller.ts  # EventBindingController
└── function-debug.controller.ts # FunctionDebugController
```

**类命名**：`<Name>Controller`，实现 `ReactiveController` 接口。

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

**生命周期**：Controller 在宿主组件的 `connectedCallback` 中通过 `host.addController(this)` 注册，在 `hostDisconnected()` 中清理资源。

```typescript
// ✅ 标准生命周期
class MyController implements ReactiveController {
  constructor(private host: ReactiveControllerHost) {
    this.host.addController(this)  // 注册
  }

  hostConnected() { /* 初始化 */ }
  hostDisconnected() { /* 清理：取消订阅、清除缓存 */ }
}
```

**EventBindingController 模式**：当根组件有大量 DOM 事件绑定（30+）时，提取到独立的 `EventBindingController` 中，通过依赖注入接收所需的 controller 和回调。

```typescript
// ✅ 集中管理事件绑定
class EventBindingController implements ReactiveController {
  constructor(host: ReactiveControllerHost, private deps: EventBindingDeps) {
    this.host.addController(this)
  }

  bindEvents(element: HTMLElement) {
    element.addEventListener('click', this._onClick)
    // ...
  }

  unbindEvents(element: HTMLElement) {
    element.removeEventListener('click', this._onClick)
    // ...
  }

  hostDisconnected() {
    // 自动清理所有绑定
  }
}
```

**Controller 回调模式**：当 Controller 的异步操作需要在特定时机通知宿主组件时，使用可选回调属性。宿主在 `connectedCallback` 中设置回调，确保生命周期协调。

```typescript
// ✅ Controller 暴露可选回调
class AuthController implements ReactiveController {
  /**
   * 登录状态就绪时触发。覆盖所有登录路径：
   * - 初始 token 加载（localStorage 有效 token）
   * - Token refresh 成功（过期 token 在页面加载后刷新）
   * - 登录对话框完成（用户显式登录）
   *
   * 宿主用于触发 WebSocket 连接，修复 connectedCallback() 在
   * 异步 token refresh 完成前运行的竞态条件。
   */
  onLogin?: () => void

  private _fireLogin() {
    // ... 更新内部状态 ...
    this.onLogin?.()  // 通知宿主
  }
}

// ✅ 宿主在 connectedCallback 中设置回调
@customElement('rtc-agent')
export class RTCAgent extends LitElement {
  connectedCallback() {
    super.connectedCallback()
    // 设置回调：auth 就绪后触发连接
    this._auth.onLogin = () => {
      void this._connectWithRetry()
    }
    // 如果已登录（同步检查），立即连接
    if (this._auth.state.isLoggedIn) {
      void this._connectWithRetry()
    }
  }

  disconnectedCallback() {
    // 清理回调，防止闭包泄漏
    this._auth.onLogin = undefined
    super.disconnectedCallback()
  }
}

// ❌ 不用回调，依赖宿主轮询或事件
// 宿主无法在"异步 auth 就绪"和"同步 auth 就绪"两种路径统一处理
```

**适用场景**：Controller 内部有异步初始化（token refresh、Worker 连接验证），宿主需要在就绪后执行一次性操作（建立 WebSocket、加载数据）。回调覆盖所有就绪路径，宿主无需区分"同步就绪"和"异步就绪"。

**规则**：

- 回调属性为可选（`onLogin?: () => void`），Controller 不假设宿主一定设置
- 宿主在 `connectedCallback` 设置，`disconnectedCallback` 清理
- 回调命名用 `on<Event>` 格式（如 `onLogin`、`onReady`）
- 回调内不传递数据（仅通知），需要数据时用 CustomEvent 替代

### Imperative Overlay Factories

当根组件需要弹出一次性对话框（确认、导出、命令面板等）时，使用纯函数工厂创建 overlay 元素，返回 Promise 等待用户响应。工厂函数负责创建、挂载、清理；调用方只需 `await` 结果。

```typescript
// ✅ 纯函数工厂：创建 overlay → 返回 Promise → 自动清理
// helpers/dialog-helpers.ts
export function showToolConfirmDialog(
    rtc: LocalRtc,
    host: ShadowRoot,
): Promise<boolean> {
    return new Promise((resolve) => {
        const el = document.createElement('rtc-tool-confirm')
        el.toolCall = { id: rtc.client_id, toolName: rtc.tool_name, ... }

        const cleanup = () => {
            el.removeEventListener('rtc-tool-call-approved', onApproved)
            el.removeEventListener('rtc-tool-call-denied', onDenied)
            el.remove()  // 必须从 DOM 移除
        }
        const onApproved = () => { cleanup(); resolve(true) }
        const onDenied = () => { cleanup(); resolve(false) }

        el.addEventListener('rtc-tool-call-approved', onApproved)
        el.addEventListener('rtc-tool-call-denied', onDenied)
        host.appendChild(el)  // 挂载到宿主 shadowRoot
    })
}

// 调用方：一行 await
const approved = await showToolConfirmDialog(rtc, this.shadowRoot!)

// ❌ 在组件内联创建 overlay 逻辑
private async _confirmTool(rtc: LocalRtc): Promise<boolean> {
    return new Promise((resolve) => {
        // 50 行创建/监听/清理逻辑混杂在组件中...
    })
}
```

**规则**：

- 工厂函数是纯函数（无 `this` 依赖），接收所需参数（数据 + host shadowRoot）
- 必须在 resolve 前清理 DOM 元素和事件监听器
- 工厂函数集中在 `helpers/dialog-helpers.ts`，不分散在组件方法中
- 返回 Promise 让调用方以 `await` 获取结果，不使用回调

### DOM Teleport（Stacking Context 逃逸）

当子组件的弹出层（菜单、工具提示）需要逃逸祖先元素的 `overflow: hidden` 或层叠上下文，但又不适合使用 `<dialog>` 或 Imperative Overlay Factory 时，将元素动态创建并附加到宿主组件的 `shadowRoot` 顶层。floating-ui 使用 `position: absolute` 相对于 offset parent（即 shadow root）定位，坐标不受 teleport 影响。

```typescript
// ✅ Teleport：附加到 shadowRoot 顶层，逃逸祖先的 overflow/stacking context
private _moreMenuEl: RtcMessageMoreMenu | null = null

private async _teleportMenu() {
    if (!this.shadowRoot || this._moreMenuEl) return  // 防止重复创建

    const menu = document.createElement('rtc-message-more-menu')
    menu.syncStatus = this.message.syncStatus
    menu.addEventListener('rtc-message-more-menu-select', this._onMenuSelect)

    this.shadowRoot.appendChild(menu)   // 附加到 shadowRoot 顶层
    this._moreMenuEl = menu

    await this.updateComplete
    this._startPositioning()  // floating-ui autoUpdate
}

private _removeMenu() {
    this._stopPositioning()
    if (this._moreMenuEl) {
        this._moreMenuEl.removeEventListener('rtc-message-more-menu-select', this._onMenuSelect)
        this._moreMenuEl.remove()   // 从 DOM 移除
        this._moreMenuEl = null
    }
}

// disconnectedCallback 中必须清理
disconnectedCallback() {
    super.disconnectedCallback()
    this._removeMenu()
}

// ❌ 在 render() 中创建弹出层：受祖先 overflow:hidden 裁剪
render() {
    return html`
        <div class="wrapper" style="overflow: hidden">
            <button @click=${this._toggleMenu}>...</button>
            ${this._showMenu ? html`<rtc-message-more-menu></rtc-message-more-menu>` : nothing}
            <!-- 被 wrapper 的 overflow:hidden 裁剪 -->
        </div>
    `
}
```

**与 Imperative Overlay Factory 的区别**：

- **Teleport**：元素在逻辑上属于子组件（如消息的 more-menu），由子组件自行管理生命周期
- **Overlay Factory**：一次性对话框（如确认框），由工厂函数创建并自动清理，与组件无归属关系

**约束**：

- Teleported 元素必须在 `disconnectedCallback` 中清理（移除 DOM + 清理事件监听器 + 停止 floating-ui）
- 用私有字段（如 `_moreMenuEl`）追踪引用，防止重复创建
- floating-ui 使用 `strategy: 'absolute'`，坐标相对于 offset parent（shadow root），teleport 不影响定位
- 仅在需要逃逸祖先层叠上下文时使用；如果弹出层不受裁剪，优先使用常规渲染（在 `render()` 中声明）

### `helpers/` 目录提取

当根组件（如 `rtc-agent.ts`）超过 500 行时，将业务逻辑提取到 `helpers/` 子目录。每个 helper 文件负责一个关注点，根组件只做组合和调度。

```text
packages/component/src/components/rtc-agent/
├── rtc-agent.ts              # 根组件：组合 + 调度（保持精简）
└── helpers/
    ├── command-handler.ts    # 命令解析与分发
    ├── dialog-helpers.ts     # 对话框 overlay 工厂
    ├── connection-setup.ts   # WebSocket 连接逻辑
    ├── session-loader.ts     # 会话加载逻辑
    ├── vfs-operations.ts     # VirtualFS 操作
    └── bus-handler.ts        # UIUpdateBus 事件处理
```

**约束**：

- helpers 文件导出纯函数，不依赖组件实例（通过参数传入所需数据）
- 每个 helper 文件有明确的单一职责
- 根组件 import helpers 并组合，保持 `rtc-agent.ts` 作为编排层
- helper 文件名用 kebab-case，与组件文件命名风格一致

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

### 叶子组件自加载（Self-Contained Data Loading）

当叶子组件需要依赖服务（如 `FileStorage`）获取的异步数据（如缩略图、文件预览）时，叶子组件应直接消费对应的 Context 自行加载，而非由父组件加载后通过属性传递。这避免了父组件管理大量子组件的数据状态，也消除了"父组件未加载完导致子组件数据缺失"的时序问题。

```typescript
// ✅ 叶子组件自加载：消费 Context 自行获取数据
@customElement('rtc-file-thumbnail')
export class FileThumbnail extends LitElement {
  @consume({ context: FileStorageContext, subscribe: true })
  @state()
  private _fileStorageCtx: FileStorageContextValue = { fileStorage: null }

  @property({ type: String }) md5 = ''
  @property({ type: String }) ext = ''

  // 自行加载缩略图，不依赖父组件传递 URL
  async updated(changed: PropertyValues) {
    if (changed.has('md5') || changed.has('ext')) {
      const { fileStorage } = this._fileStorageCtx
      if (fileStorage && this.md5) {
        this._thumbnailUrl = await fileStorage.getThumbnailUrl({
          md5: this.md5, ext: this.ext,
        })
      }
    }
  }

  disconnectedCallback() {
    super.disconnectedCallback()
    // 自加载的资源必须自行释放
    this._fileStorageCtx.fileStorage?.releaseThumbnailUrl(this._thumbnailUrl)
  }
}

// 父组件只传递标识信息，不加载数据
render() {
  return html`<rtc-file-thumbnail .md5=${file.md5} .ext=${file.ext}></rtc-file-thumbnail>`
}

// ❌ 父组件加载后传递：父组件管理 N 个子组件的缩略图状态
@customElement('rtc-file-thumbnail')
export class FileThumbnail extends LitElement {
  @property({ type: String }) thumbnailUrl = ''  // 父组件负责加载并传递
}
```

**判断标准**：数据是否需要异步获取且依赖共享服务？如果是，叶子组件消费 Context 自加载；如果数据是同步的纯展示数据（如名称、大小），用属性传递即可。

**约束**：

- 自加载的组件必须在 `disconnectedCallback` 中释放资源（如 blob URL、缓存引用）
- 自加载只用于叶子组件，中间层组件不通过自加载回避属性传递
- 多个叶子组件需要相同资源时，服务层应提供缓存/共享机制（如 FileStorage 的 blob URL 共享 + refcount）

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

### Per-Entity Promise 链序列化

当同一实体（如 session、file）的多个异步操作可能并发执行时，用 Promise 链序列化操作，防止竞态条件（如旧操作覆盖新操作的结果）。

```typescript
// ✅ Per-entity Promise 链
private _sessionOps = new Map<string, Promise<void>>()

private _enqueueSessionOp(sessionId: string, op: () => Promise<void>): void {
  const prev = this._sessionOps.get(sessionId) ?? Promise.resolve()
  const next = prev.then(op, op)  // 无论前一个成功或失败，都执行下一个
  this._sessionOps.set(sessionId, next)

  // 清理已完成的链
  next.finally(() => {
    if (this._sessionOps.get(sessionId) === next) {
      this._sessionOps.delete(sessionId)
    }
  })
}

// 使用
this._enqueueSessionOp(sessionId, () => this._reloadMessages(sessionId))
this._enqueueSessionOp(sessionId, () => this._updateTurnCount(sessionId))
// 两个操作串行执行，不会竞态

// ❌ 不序列化，旧操作可能覆盖新操作
async _reloadMessages(sessionId: string) {
  const msgs = await api.getMessages(sessionId)  // 两个并发调用交错执行
  this._messages = msgs                           // 结果不确定
}
```

**适用场景**：同一实体的加载/刷新操作、IndexedDB 写入、Worker 通信中的 per-session 请求。判断标准：两个异步操作对同一实体的写入可能交错导致状态不一致。

### Per-Session State with Immutable Map Updates

当布局组件需要追踪多个 session 的独立状态（如文件附件、上传进度）时，使用 `@state()` 装饰 `Map<string, EntityState>`，并在每次更新时创建新 Map 触发 Lit 响应式更新。

```typescript
// ✅ 不可变 Map 更新：每次 set 创建新 Map 引用
@state()
private _fileStates = new Map<string, FileState>()

private _handleFilesChanged(e: CustomEvent, sessionId: string): void {
  const detail = e.detail
  // 创建新 Map，触发 Lit 的引用比较更新
  this._fileStates = new Map(this._fileStates).set(sessionId, {
    files: detail.files,
    uploadProgress: detail.uploadProgress,
    uploadStates: detail.uploadStates,
  })
}

// 渲染时按 session 获取对应状态
private _renderFilePreview(sessionId: string) {
  const fileState = this._fileStates.get(sessionId)
  if (!fileState || fileState.files.length === 0) return nothing
  return html`<rtc-file-preview-area .files=${fileState.files} ...></rtc-file-preview-area>`
}

// ❌ 直接 mutate Map，Lit 不会检测到变化
private _handleFilesChanged(e: CustomEvent, sessionId: string): void {
  this._fileStates.set(sessionId, newState)  // Map 引用未变，不触发更新
}
```

**规则**：

- 更新 `Map`/`Set` 类型的 `@state()` 时，必须创建新实例（`new Map(this._map).set(...)`），不能直接 mutate
- 渲染方法按 key（如 `sessionId`）获取对应状态，支持多 tab 独立渲染
- 适用于布局组件追踪多个 session 的独立 UI 状态（文件预览、上传进度等）

### Offline-First File Upload（三阶段上传）

文件上传采用"本地优先"策略：先缓存到 IndexedDB（即时获得 fileid 和缩略图），再后台同步到 S3。即使用户离线或 S3 上传失败，文件仍可附加到消息中。

```typescript
// ✅ 三阶段上传管道
private async _uploadFiles(files: File[]) {
    // Phase 1: 本地缓存（IndexedDB），获得 fileid
    // 即使离线也能完成此步
    const entries: Array<{file: File; attachment: FileAttachment}> = []
    for (const file of files) {
        const fileInfo = await fileStorage.cacheFileForUpload(file)
        const fileid = `${fileInfo.md5}.${fileInfo.ext}`
        entries.push({file, attachment: {fileid, mimetype: file.type, ...}})
        this._fileObjects.set(fileid, file)  // 保存 File 对象供 retry 使用
        this._uploadStates.set(fileid, 'loading')
    }

    // Phase 2: 立即显示（缩略图从 IndexedDB 加载，无需等待 S3）
    this._pendingFiles = [...this._pendingFiles, ...entries.map(e => e.attachment)]
    this._uploadStates = new Map(this._uploadStates)  // 不可变更新，触发 Lit re-render
    this._notifyFileStateChange()

    // Phase 3: 后台同步到 S3（失败不影响 UI）
    for (const {file, attachment} of entries) {
        this._syncToS3(file, attachment)  // 不 await，后台执行
    }
}

private async _syncToS3(file: File, attachment: FileAttachment): Promise<void> {
    const fileid = attachment.fileid
    try {
        const result = await fileStorage.upload({
            file,
            onProgress: (loaded, total) => {
                this._uploadProgress.set(fileid, Math.round((loaded / total) * 100))
                this._uploadProgress = new Map(this._uploadProgress)
                this._notifyFileStateChange()
            },
        })
        this._uploadStates.set(fileid, result.syncStatus === 'synced' ? 'loaded' : 'error')
    } catch (error) {
        this._uploadStates.set(fileid, 'error')
    }
    this._uploadStates = new Map(this._uploadStates)
    this._notifyFileStateChange()
}

// ✅ 重试：从缓存的 File 对象恢复
public retryUpload(fileid: string) {
    const file = this._fileObjects.get(fileid)
    if (!file) return
    this._uploadStates.set(fileid, 'loading')
    this._syncToS3(file, attachment)
}

// ❌ 同步上传：用户必须等待 S3 返回才能继续
private async _uploadFiles(files: File[]) {
    for (const file of files) {
        await fileStorage.upload(file)  // 阻塞，离线时直接失败
        this._pendingFiles.push(attachment)  // 用户看到空白等待
    }
}
```

**规则**：

- **Phase 1 本地缓存是安全网**：文件先写入 IndexedDB，获得基于 MD5 的 fileid，后续步骤失败不影响用户继续操作
- **Phase 2 即时显示**：缩略图从 `FileStorage.getThumbnailUrl()` 加载（读取 IndexedDB 缓存），不等待 S3 上传完成
- **Phase 3 后台异步**：`_syncToS3` 不 await，失败时标记状态为 `error`，UI 显示重试按钮
- **保存 File 对象供重试**：`_fileObjects: Map<fileid, File>` 在内存中保留原始 `File` 引用，retry 时直接使用，无需用户重新选择
- **上传状态机**：`idle → loading → loaded | error`，通过 `Map<fileid, UploadState>` 追踪，每次更新创建新 Map 引用
- **进度语义**：在线上传时 `onProgress` 反映 S3 进度（0% -> 100%）；离线时 `onProgress` 跳到 100%（写入 IndexedDB 完成），后台同步进度不通过 `onProgress` 报告，而通过 `syncStatus`（`pending` / `syncing` / `synced` / `failed`）追踪。UI 组件（进度条、重试按钮）必须同时检查 `onProgress` 和 `syncStatus`，否则离线上传的文件会显示 100% 但实际未同步到服务端

**适用场景**：文件附件上传、图片/视频上传、任何"离线可用 + 后台同步"的写入操作。判断标准：用户上传的文件是否应在离线时仍可用。

### FileStorage 并发控制（滑动窗口）

`FileStorage` 工具通过 `runWithConcurrency` 实现滑动窗口并发控制，避免同时发起过多 S3 请求导致浏览器连接耗尽。

```typescript
// ✅ 滑动窗口并发控制
async function runWithConcurrency<T>(
    tasks: (() => Promise<T>)[],
    concurrency: number,
    onProgress?: (completed: number, total: number) => void,
    signal?: AbortSignal
): Promise<(T | Error)[]> {
    const results: (T | Error)[] = new Array(tasks.length)
    let nextIndex = 0
    let completed = 0

    async function worker() {
        while (nextIndex < tasks.length) {
            if (signal?.aborted) return
            const idx = nextIndex++
            try {
                results[idx] = await tasks[idx]()
            } catch (e) {
                results[idx] = e instanceof Error ? e : new Error(String(e))
            }
            completed++
            onProgress?.(completed, tasks.length)
        }
    }

    // 启动 concurrency 个 worker，每个从 tasks 中拉取下一个
    await Promise.all(Array.from({length: Math.min(concurrency, tasks.length)}, () => worker()))
    return results
}

// 使用：批量上传，最多 3 个并发
const tasks = files.map(file => () => fileStorage.upload({file}))
const results = await runWithConcurrency(tasks, 3, (done, total) => {
    console.log(`${done}/${total} files uploaded`)
}, abortSignal)
```

**规则**：

- 默认并发数 3（浏览器同域 HTTP/1.1 连接数限制约 6 个，留一半给其他请求）
- 结果数组按索引写入（`results[idx] = ...`），不 append 到共享数组，避免竞态
- 单个任务失败不中止其他任务，失败结果存为 `Error` 对象
- `AbortSignal` 可中止尚未开始的任务，已进行中的任务不受影响

---

## 11. 国际化 (i18n)

### 使用 `@lit/localize`

所有用户可见的文本必须通过 `@lit/localize` 的 `msg()` 函数标记，不硬编码字符串。

```typescript
import { msg } from '@lit/localize'
import { localized } from '@lit/localize'

// ✅ 标记用户可见文本
@localized()
@customElement('rtc-chat-panel')
export class ChatPanel extends LitElement {
  render() {
    return html`<button>${msg('Send')}</button>`
  }
}

// ❌ 硬编码文本
render() {
  return html`<button>Send</button>`  // 无法翻译
}
```

### 装饰器顺序

`@localized()` 必须在 `@customElement()` 之前：

```typescript
// ✅ 正确顺序
@localized()
@customElement('rtc-my-component')
export class MyComponent extends LitElement { }

// ❌ 错误顺序
@customElement('rtc-my-component')
@localized()
export class MyComponent extends LitElement { }
```

### locale 动态加载

locale bundle 通过动态 `import()` 按需加载，遵循已有的动态 import 重试模式（参见第 7 节）。

```typescript
// core/i18n.ts
const localeModules: Record<string, () => Promise<LocaleModule>> = {
  'en-US': async () => {
    const mod = await import('../locales/en-US.js')
    return mod.default
  },
}
```

### locale Context

locale 状态通过 Lit Context 传播，遵循 `{state, actions}` 模式：

```typescript
export interface LocaleContextValue {
  locale: SupportedLocale
  setLocale: (locale: SupportedLocale) => Promise<void>
  locales: readonly SupportedLocale[]
}

export const localeContext = createContext<LocaleContextValue>(Symbol('locale'))
```

### locale 持久化

用户选择的 locale 存储在 localStorage，key 格式遵循 `rtc_` 前缀约定：

```typescript
const STORAGE_KEY = 'rtc-agent-locale'  // rtc_ 前缀
localStorage.setItem(STORAGE_KEY, locale)
```

### 新增组件检查清单补充

- [ ] 用户可见文本使用 `msg()` 标记
- [ ] 组件类添加 `@localized()` 装饰器（在 `@customElement()` 之前）

---

## 总结

好的组件像好的函数——输入明确、输出可预测、副作用可控。遵循这些规范，让组件库成为团队的基石，而非负担。

> _"组件是产品的原子。每个原子都应当稳定、可组合、可信赖。"_
