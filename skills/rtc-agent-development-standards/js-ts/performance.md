# 前端性能规范

> _性能是用户体验的隐形骨架。用户不会说"加载好快"，但会说"卡顿好烦"。_

---

## 1. Bundle 优化

### 包体积预算

| 类型 | 预算 | 说明 |
|------|------|------|
| 主 Bundle | < 150KB (gzip) | 首次加载必须 |
| 懒加载 Chunk | < 50KB (gzip) | 按需加载 |
| 总 Bundle | < 300KB (gzip) | 含所有 chunk |

### 动态 import 懒加载

非首屏必需的模块必须动态导入。

```typescript
// ✅ 懒加载
async connectedCallback() {
  super.connectedCallback()
  if (this.needsHeavyFeature) {
    const { HeavyComponent } = await import('./heavy-component.js')
    this._heavy = new HeavyComponent()
  }
}

// ✅ 路由级懒加载
const routes = {
  settings: () => import('./pages/settings.js'),
  analytics: () => import('./pages/analytics.js'),
}
```

### tree-shaking 友好

```typescript
// ✅ 具名导出，可 tree-shake
export function formatDate(d: Date): string { ... }
export function parseDate(s: string): Date { ... }

// ❌ 默认导出整个对象，无法 tree-shake
export default {
  formatDate: (d: Date) => ...,
  parseDate: (s: string) => ...,
}
```

---

## 2. 渲染性能

### Lit `hasChanged` 自定义比较

复杂类型默认引用比较，每次都是 `true`，导致不必要的 re-render。

```typescript
// ✅ 自定义比较逻辑
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

### 避免 render 中计算

```typescript
// ❌ 每次 render 都重新计算
render() {
  const sorted = [...this.items].sort((a, b) => a.name.localeCompare(b.name))
  return html`${sorted.map(item => html`<div>${item.name}</div>`)}`
}

// ✅ willUpdate 中计算，只在依赖变化时执行
willUpdate(changed: PropertyValues) {
  if (changed.has('items')) {
    this._sorted = [...this.items].sort((a, b) => a.name.localeCompare(b.name))
  }
}

render() {
  return html`${this._sorted.map(item => html`<div>${item.name}</div>`)}`
}
```

### 大列表虚拟化

超过 50 个列表项时使用 `@lit-labs/virtualizer`。

```typescript
import { virtualize } from '@lit-labs/virtualizer/virtualize.js'

render() {
  return html`
    <div style="height: 400px; overflow-y: auto;">
      ${virtualize(this.messages, msg => html`
        <div class="message">${msg.content}</div>
      `)}
    </div>
  `
}
```

---

## 3. 网络性能

### WebSocket 消息体积

```typescript
// ❌ 冗余字段
{ type: 'message', data: { id: '123', content: 'hello', timestamp: 1234567890, userId: 'u1', sessionId: 's1', ... } }

// ✅ 精简传输
{ t: 'msg', d: { i: '123', c: 'hello', ts: 1234567890 } }
```

### 请求合并

短时间内多次更新合并为一次请求。

```typescript
// ❌ 每个操作都发请求
saveField1() { api.update({ field1: this.value1 }) }
saveField2() { api.update({ field2: this.value2 }) }

// ✅ 批量更新
private _pendingUpdates: Record<string, any> = {}
private _flushTimer?: ReturnType<typeof setTimeout>

updateField(key: string, value: any) {
  this._pendingUpdates[key] = value
  clearTimeout(this._flushTimer)
  this._flushTimer = setTimeout(() => {
    api.update(this._pendingUpdates)
    this._pendingUpdates = {}
  }, 100)  // 100ms 内合并
}
```

---

## 4. 内存管理

### 事件监听器清理

```typescript
// ✅ connectedCallback 注册，disconnectedCallback 清理
private _resizeHandler = () => this._handleResize()

connectedCallback() {
  super.connectedCallback()
  window.addEventListener('resize', this._resizeHandler)
}

disconnectedCallback() {
  super.disconnectedCallback()
  window.removeEventListener('resize', this._resizeHandler)
}
```

### Floating UI cleanup

```typescript
// ✅ autoUpdate 返回的 cleanup 必须调用
private _cleanupFloating?: () => void

showDropdown() {
  this._cleanupFloating = autoUpdate(reference, floating, updatePosition)
}

hideDropdown() {
  this._cleanupFloating?.()  // 必须清理
  this._cleanupFloating = undefined
}

disconnectedCallback() {
  super.disconnectedCallback()
  this._cleanupFloating?.()
}
```

### GSAP 动画清理

```typescript
private _tl?: gsap.core.Timeline

firstUpdated() {
  this._tl = gsap.timeline().to(this._el, { opacity: 1 })
}

disconnectedCallback() {
  super.disconnectedCallback()
  this._tl?.kill()  // 防止泄漏
}
```

---

## 5. 性能指标

| 指标 | 目标 | 说明 |
|------|------|------|
| **FCP** | < 1s | 首次内容绘制 |
| **LCP** | < 2.5s | 最大内容绘制 |
| **TTI** | < 3.5s | 可交互时间 |
| **组件首次渲染** | < 16ms | 一帧内完成 |
| **列表滚动** | 60fps | 无卡顿 |

---

## 6. 新增代码性能检查清单

- [ ] 非首屏模块使用动态 `import()`
- [ ] 复杂类型定义 `hasChanged`
- [ ] 派生状态在 `willUpdate` 中计算
- [ ] 大列表使用虚拟化（>50 项）
- [ ] 事件监听器在 `disconnectedCallback` 清理
- [ ] Floating UI cleanup 调用
- [ ] GSAP 动画 `kill()` 清理
- [ ] Bundle 体积在预算内

---

## 总结

性能是持续的投资，不是一次性的优化。从第一行代码就遵循这些约束，让性能成为习惯而非负担。

> _"用户不会为你的代码复杂度买单，他们只会为流畅的体验停留。"_
