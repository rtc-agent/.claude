# 样式规范

> _样式不是装饰，是产品的语言。统一的样式体系让组件像同一个设计师的手笔。_

---

## 1. 容器边界约束

### 子组件不可逃逸

所有子组件必须约束在 `<rtc-agent>` 窗口内。根组件通过 `overflow: hidden` 实现严格裁剪。

```css
/* :host 的样式 — 不可改变 */
:host {
  position: fixed;
  overflow: hidden;    /* 主要约束机制 */
  z-index: 9999;       /* 整个窗口位于宿主页面之上 */
  container-type: inline-size;
}
```

### 新增浮层组件的约束

- 浮层的 `z-index` 必须使用 `--rtc-z-*` 变量，不硬编码
- 浮层定位使用 `position: fixed` 或 `position: absolute`，坐标由 Floating UI 计算
- 即使使用 `position: fixed`（视口相对），视觉上仍被 `:host` 的 `overflow: hidden` 裁剪

### 禁止

- 禁止将浮层 append 到 `document.body`（会逃逸 Shadow DOM 边界）
- 禁止修改根组件的 `overflow` 属性
- 禁止在子组件中设置超出窗口范围的 `position: absolute` 定位

---

## 2. z-index 层级规范

### 必须使用 CSS 变量

所有 z-index 必须引用已有的 CSS 变量体系，禁止硬编码数值。

```css
/* ✅ 使用变量 */
.overlay { z-index: var(--rtc-z-overlay); }
.modal   { z-index: var(--rtc-z-modal); }
.toast   { z-index: var(--rtc-z-toast); }

/* ❌ 硬编码 */
.overlay { z-index: 500; }
.modal   { z-index: 999; }
```

### 层级体系

```
:host (z-index: 9999)                    ← 整个窗口位于宿主页面之上
  └── .window-container
        ├── rtc-title-bar     (z: 10)    ← 标题栏
        └── .content-area     (z: 1)     ← 内容区域
              └── rtc-overlay-manager (z: 500)  ← 浮层管理
                    ├── dropdown/panel   (z: 500)  ← 下拉菜单、面板
                    ├── modal dialog     (z: 501)  ← 模态对话框
                    └── rtc-toast        (z: 1000) ← 通知提示
```

| 变量 | 值 | 用途 |
|------|-----|------|
| `--rtc-z-content` | 1 | 内容区域、普通组件 |
| `--rtc-z-title-bar` | 10 | 标题栏 |
| `--rtc-z-overlay` | 500 | 下拉菜单、面板、气泡 |
| `--rtc-z-modal` | 501 | 模态对话框 |
| `--rtc-z-toast` | 1000 | 通知提示 |

### 新增层级

如果需要新增层级，必须在上述体系中定位，并更新 `styles/tokens.ts`：

- 比 overlay 低但不影响内容 → 在 1-499 之间
- 比 modal 高但比 toast 低 → 在 502-999 之间
- 比 toast 还高 → 评估是否真的需要，避免层级膨胀

---

## 3. 浮层与弹窗

### 弹窗（Modal）

弹窗使用 `position: absolute` 覆盖整个内容区域。

```typescript
// ✅ 弹窗样式
static styles = css`
  :host {
    position: absolute;
    inset: 0;
    z-index: var(--rtc-z-modal);
  }

  .backdrop {
    position: absolute;
    inset: 0;
    background: var(--rtc-color-backdrop);
  }

  .dialog {
    position: absolute;
    top: 50%;
    left: 50%;
    transform: translate(-50%, -50%);
    max-width: var(--rtc-dialog-max-width, 360px);
  }
`
```

**必须**：

- `role="dialog"` + `aria-modal="true"`
- 背景遮罩点击关闭（如适用）
- Escape 键关闭
- 焦点陷阱（focus trap）

### 下拉菜单 / 面板

使用 `position: fixed` + Floating UI 定位。

```typescript
// ✅ 面板样式
static styles = css`
  :host {
    position: fixed;
    z-index: var(--rtc-z-overlay);
    max-height: var(--rtc-overlay-panel-max-height, 320px);
    overflow-y: auto;
  }
`
```

**必须**：

- 使用 Floating UI 计算位置
- 点击外部关闭
- Escape 键关闭

---

## 4. Floating UI 定位

### 唯一官方定位库

所有下拉菜单、气泡、弹出面板必须使用 `@floating-ui/dom`。不使用 CSS 手动定位，不使用 Popper.js。

### 统一 middleware 配置

```typescript
import { computePosition, flip, shift, offset, autoUpdate } from '@floating-ui/dom'

// ✅ 统一配置
const cleanup = autoUpdate(reference, floating, async () => {
  const { x, y } = await computePosition(reference, floating, {
    placement: 'bottom-end',
    middleware: [
      offset(6),              // 与参考元素的间距
      flip({ padding: 8 }),   // 空间不足时翻转方向
      shift({ padding: 8 }),  // 沿边界滑动
    ],
  })
  Object.assign(floating.style, { left: `${x}px`, top: `${y}px` })
})

// ✅ 清理（必须在 disconnectedCallback 或关闭时调用）
cleanup()
```

### placement 选择

| 场景 | placement |
|------|-----------|
| 下拉菜单（按钮下方） | `bottom-end` |
| 上拉面板（按钮上方） | `top-end` |
| 气泡提示 | 根据触发元素位置选择 |

### 必须做的事

- 使用 `autoUpdate` 持续更新位置（窗口 resize、滚动时自动修正）
- 关闭时调用 `cleanup()` 函数，释放监听器
- 使用 `composedPath()` 检测点击外部（穿透 Shadow DOM）

---

## 5. CSS 变量体系

### 三层结构

```
tokens.ts          → 结构 token（间距、字体、圆角、阴影、z-index）
themes/light.ts    → 亮色主题颜色 token
themes/dark.ts     → 暗色主题颜色 token
base.ts            → 基础重置样式
```

### 命名规范

所有变量以 `--rtc-` 为前缀。

```css
/* 间距 */
--rtc-spacing-xs: 4px;
--rtc-spacing-sm: 8px;
--rtc-spacing-md: 16px;
--rtc-spacing-lg: 24px;

/* 字体 */
--rtc-font-size-xs: 12px;
--rtc-font-size-sm: 13px;
--rtc-font-size-base: 14px;

/* 圆角 */
--rtc-border-radius-sm: 4px;
--rtc-border-radius: 6px;
--rtc-border-radius-lg: 8px;

/* 阴影 */
--rtc-shadow-sm: ...;
--rtc-shadow-md: ...;
--rtc-shadow-lg: ...;

/* 颜色（主题文件中定义） */
--rtc-color-primary: ...;
--rtc-color-text: ...;
--rtc-color-bg: ...;
--rtc-color-border: ...;
```

### 新增 token 规则

- **结构 token**：放入 `tokens.ts`，与主题无关
- **颜色 token**：放入 `themes/light.ts` 和 `themes/dark.ts`，必须同时提供两套
- **组件局部 token**：在组件 `:host` 上定义，如 `--rtc-dialog-max-width`

### 使用 token

组件样式必须通过 `var()` 引用 token，不硬编码原始值。

```css
/* ✅ 使用 token */
.button {
  padding: var(--rtc-spacing-sm) var(--rtc-spacing-md);
  font-size: var(--rtc-font-size-base);
  border-radius: var(--rtc-border-radius);
  color: var(--rtc-color-text);
  background: var(--rtc-color-bg);
}

/* ❌ 硬编码 */
.button {
  padding: 8px 16px;
  font-size: 14px;
  border-radius: 6px;
  color: #333;
  background: #fff;
}
```

---

## 6. 主题系统

### 主题切换

通过 `:host([theme])` 属性切换主题。

```css
/* 默认 / 亮色 */
:host, :host([theme='light']) {
  --rtc-color-bg: #ffffff;
  --rtc-color-text: #1a1a1a;
}

/* 暗色 */
:host([theme='dark']) {
  --rtc-color-bg: #1a1a1a;
  --rtc-color-text: #f0f0f0;
}
```

### 组件必须支持主题

- 所有颜色值必须使用 `--rtc-color-*` 变量
- 不硬编码颜色值
- 测试时确认 light/dark 两套主题都正常显示

### 系统偏好

支持 `prefers-color-scheme` 自动跟随系统：

```typescript
@property({ type: String }) theme: 'light' | 'dark' | 'system' = 'system'
```

---

## 7. 动画约束

### 使用 GSAP

所有新增动画必须使用 GSAP，不用 CSS animation 或 Web Animation API。

```typescript
import gsap from 'gsap'

// ✅ GSAP 动画
gsap.to(this._dialogEl, {
  opacity: 1,
  scale: 1,
  duration: parseFloat(getComputedStyle(this).getPropertyValue('--rtc-transition-duration')) || 0.15,
  ease: 'power2.out',
})
```

### 动画生命周期管理

```typescript
// ✅ 在 firstUpdated 中创建
private _tl?: gsap.core.Timeline

firstUpdated() {
  this._tl = gsap.timeline()
    .from(this._headerEl, { y: -20, opacity: 0 })
    .from(this._contentEl, { y: 20, opacity: 0 }, '-=0.1')
}

// ✅ 在 disconnectedCallback 中销毁
disconnectedCallback() {
  super.disconnectedCallback()
  this._tl?.kill()  // 防止泄漏
}
```

### 尊重 `prefers-reduced-motion`

```typescript
const prefersReducedMotion = window.matchMedia('(prefers-reduced-motion: reduce)').matches

if (prefersReducedMotion) {
  gsap.defaults({ duration: 0.01 })
}
```

### 动画时长一致性

使用 CSS 变量保持时长一致：

```css
:host {
  --rtc-transition-duration: 0.15s;
  --rtc-transition-duration-slow: 0.3s;
}
```

---

## 8. 新增组件样式检查清单

新组件上线前，必须确认：

- [ ] 样式使用 `static styles` 封装在 Shadow DOM 中
- [ ] 所有颜色使用 `--rtc-color-*` 变量
- [ ] 所有间距使用 `--rtc-spacing-*` 变量
- [ ] z-index 使用 `--rtc-z-*` 变量，不硬编码
- [ ] 浮层/弹窗不逃逸 `<rtc-agent>` 容器
- [ ] 下拉菜单使用 Floating UI 定位
- [ ] 动画使用 GSAP
- [ ] 支持 light/dark 两套主题
- [ ] 尊重 `prefers-reduced-motion`
- [ ] `disconnectedCallback` 清理动画实例和 Floating UI cleanup

---

## 总结

样式是产品品质的直接体现。遵循这套体系，让每个组件都像是同一双手打磨出来的。

> _"统一不是限制创造力，而是把创造力集中在真正需要它的地方。"_
