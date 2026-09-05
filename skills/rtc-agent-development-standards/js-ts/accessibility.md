# 可访问性规范

> _可访问性不是可选项。一个不可访问的产品，对一部分用户来说就是不存在的产品。_

---

## 1. ARIA 属性

### 交互式组件必须设置 role

```typescript
// ✅ 按钮
render() {
  return html`
    <button role="button" aria-label="发送消息">
      ${sendIcon}
    </button>
  `
}

// ✅ 自定义下拉菜单
render() {
  return html`
    <div role="listbox" aria-label="选择工作模式">
      <div role="option" aria-selected="true">手动模式</div>
      <div role="option" aria-selected="false">自动模式</div>
    </div>
  `
}

// ✅ 模态框
render() {
  return html`
    <div role="dialog" aria-modal="true" aria-labelledby="dialog-title">
      <h2 id="dialog-title">确认操作</h2>
      <p>确定要删除这条消息吗？</p>
    </div>
  `
}
```

### 状态变化用 aria-live

```typescript
// ✅ 动态内容通知屏幕阅读器
render() {
  return html`
    <div aria-live="polite" aria-atomic="true">
      ${this._statusMessage}
    </div>
  `
}

// aria-live 值：
// - polite: 等用户空闲时播报
// - assertive: 立即打断播报（用于错误、警告）
```

### 禁止

- 禁止移除 `outline` 而不提供替代焦点指示器
- 禁止用 `tabindex="-1"` 隐藏可交互元素（应该用 `disabled`）

---

## 2. 键盘导航

### 所有交互元素必须可 Tab 聚焦

```typescript
// ✅ 原生 button 天然可聚焦
render() {
  return html`<button @click=${this._handleClick}>操作</button>`
}

// ✅ 自定义交互元素必须设置 tabindex
render() {
  return html`
    <div
      role="button"
      tabindex="0"
      @click=${this._handleClick}
      @keydown=${this._handleKeydown}
    >
      操作
    </div>
  `
}

private _handleKeydown(e: KeyboardEvent) {
  if (e.key === 'Enter' || e.key === ' ') {
    e.preventDefault()
    this._handleClick()
  }
}
```

### 键盘操作映射

| 按键 | 行为 |
|------|------|
| `Tab` | 移动到下一个可聚焦元素 |
| `Shift+Tab` | 移动到上一个可聚焦元素 |
| `Enter` / `Space` | 触发按钮操作 |
| `Escape` | 关闭浮层、模态框、下拉菜单 |
| `ArrowUp` / `ArrowDown` | 在列表/菜单中上下导航 |
| `ArrowLeft` / `ArrowRight` | 在 tab 栏中切换 |
| `Home` / `End` | 跳到列表首/尾 |

### 浮层 Escape 关闭

```typescript
// ✅ 所有浮层、弹窗、下拉菜单必须响应 Escape
connectedCallback() {
  super.connectedCallback()
  this._keydownHandler = (e: KeyboardEvent) => {
    if (e.key === 'Escape') {
      this._close()
    }
  }
  document.addEventListener('keydown', this._keydownHandler)
}

disconnectedCallback() {
  super.disconnectedCallback()
  document.removeEventListener('keydown', this._keydownHandler)
}
```

---

## 3. 焦点管理

### 模态框焦点陷阱

模态框打开时，焦点必须限制在模态框内；关闭时，焦点必须返回触发元素。

```typescript
// ✅ 焦点陷阱
class ConfirmDialog extends LitElement {
  private _previousFocus?: HTMLElement

  show(triggerElement: HTMLElement) {
    this._previousFocus = triggerElement
    this.setAttribute('open', '')

    // 焦点移入模态框
    requestAnimationFrame(() => {
      const firstFocusable = this.shadowRoot!.querySelector(
        'button, [tabindex]:not([tabindex="-1"])'
      ) as HTMLElement
      firstFocusable?.focus()
    })
  }

  hide() {
    this.removeAttribute('open')
    // 焦点返回触发元素
    this._previousFocus?.focus()
  }
}
```

### delegatesFocus 用于 Shadow DOM

交互式组件使用 `delegatesFocus`，让 Shadow DOM 内的焦点行为像原生元素。

```typescript
static shadowRootOptions: ShadowRootInit = {
  mode: 'open',
  delegatesFocus: true,  // 点击组件任何位置都聚焦到第一个可聚焦元素
}
```

### 可见的焦点样式

焦点指示器必须可见。不全局移除 `outline`。

```css
/* ✅ 自定义焦点样式，但保持可见 */
:focus-visible {
  outline: 2px solid var(--rtc-color-primary);
  outline-offset: 2px;
}

/* ❌ 移除焦点指示器 */
:focus {
  outline: none;  /* 键盘用户无法看到焦点 */
}
```

---

## 4. 颜色与对比度

### 对比度要求

文字与背景对比度 ≥ 4.5:1（WCAG AA 标准）。

```css
/* ✅ 高对比度 */
:root {
  --rtc-color-text: #1a1a1a;        /* 与 #fff 对比度 16.75:1 */
  --rtc-color-text-secondary: #555;  /* 与 #fff 对比度 7.45:1 */
}

/* ❌ 低对比度 */
:root {
  --rtc-color-text-secondary: #ccc;  /* 与 #fff 对比度 1.6:1 — 不可读 */
}
```

### 不依赖颜色单一维度

信息的传达不能仅靠颜色。

```typescript
// ❌ 仅靠颜色区分状态
render() {
  return html`
    <span style="color: red">错误</span>
    <span style="color: green">成功</span>
  `
}

// ✅ 颜色 + 图标 + 文字
render() {
  return html`
    <span class="error">${errorIcon} 错误：请填写必填项</span>
    <span class="success">${checkIcon} 成功：消息已发送</span>
  `
}
```

---

## 5. 屏幕阅读器

### 图标按钮必须有文字替代

```typescript
// ❌ 屏幕阅读器不知道按钮做什么
render() {
  return html`<button>${sendIcon}</button>`
}

// ✅ 提供 aria-label
render() {
  return html`<button aria-label="发送消息">${sendIcon}</button>`
}
```

### 装饰性图形隐藏

```typescript
// ✅ 装饰性 Logo 对屏幕阅读器不可见
render() {
  return html`
    <div class="logo" aria-hidden="true">
      ${logoSvg}
    </div>
  `
}
```

### 动态内容更新

```typescript
// ✅ 消息列表更新通知屏幕阅读器
render() {
  return html`
    <div aria-live="polite">
      ${this._messages.map(msg => html`
        <div>${msg.content}</div>
      `)}
      ${this._isLoading ? html`<span>正在加载...</span>` : nothing}
    </div>
  `
}
```

---

## 6. 表单与输入

### 每个输入关联 label

```typescript
// ✅ 显式关联
render() {
  return html`
    <label for="msg-input">消息内容</label>
    <textarea id="msg-input" aria-required="true"></textarea>
  `
}

// ✅ Shadow DOM 内用 aria-labelledby
render() {
  return html`
    <span id="input-label">消息内容</span>
    <textarea aria-labelledby="input-label" aria-required="true"></textarea>
  `
}
```

### 错误信息关联

```typescript
// ✅ 错误信息用 aria-describedby 关联到输入
render() {
  return html`
    <input
      id="email"
      aria-invalid=${this._hasError}
      aria-describedby=${this._hasError ? 'email-error' : nothing}
    />
    ${this._hasError
      ? html`<span id="email-error" role="alert">请输入有效的邮箱地址</span>`
      : nothing
    }
  `
}
```

---

## 7. 新增组件 a11y 检查清单

新组件上线前，必须逐项确认：

- [ ] 所有交互元素可通过 Tab 键聚焦
- [ ] 所有按钮/链接有 `aria-label` 或可见文字
- [ ] 图标按钮有 `aria-label`
- [ ] 装饰性元素有 `aria-hidden="true"`
- [ ] 模态框有 `role="dialog"` + `aria-modal="true"`
- [ ] 模态框打开时焦点移入，关闭时焦点返回
- [ ] Escape 键可关闭浮层/弹窗
- [ ] 焦点样式可见（未全局移除 `outline`）
- [ ] 文字对比度 ≥ 4.5:1
- [ ] 信息不依赖颜色单一维度
- [ ] 表单输入关联 `<label>`
- [ ] 错误信息用 `aria-describedby` 关联
- [ ] 动态内容用 `aria-live` 通知
- [ ] 键盘操作符合预期（Enter/Space 触发，方向键导航）

---

## 总结

可访问性是产品质量的底线。它不是"以后再做"的功能，而是从第一行代码就要考虑的约束。

> _"好的可访问性让所有人受益——不只是残障用户，还有使用键盘的开发者、使用小屏幕的移动用户、以及在强光下看不清屏幕的每一个人。"_
