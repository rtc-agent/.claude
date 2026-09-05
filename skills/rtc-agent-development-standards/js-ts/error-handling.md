# 前端错误处理规范

> _错误不是异常，是程序的一部分。好的错误处理让用户感到被尊重，而非被抛弃。_

---

## 1. 错误边界

### render 不应抛错

`render()` 必须是安全的。如果数据可能导致渲染错误，在渲染前处理。

```typescript
// ❌ render 中可能抛错
render() {
  return html`<div>${this.messages[0].content}</div>`  // messages 为空时崩溃
}

// ✅ 安全处理
render() {
  if (!this.messages || this.messages.length === 0) {
    return html`<div class="empty">暂无消息</div>`
  }
  return html`<div>${this.messages[0].content}</div>`
}
```

### 顶层错误捕获

根组件捕获未处理的错误，展示 fallback UI，防止白屏。

```typescript
// ✅ 全局错误捕获
window.addEventListener('error', (event) => {
  console.error('Uncaught error:', event.error)
  // 展示 fallback UI
  document.body.innerHTML = `
    <div class="error-fallback">
      <h2>应用出错</h2>
      <p>请刷新页面重试</p>
      <button onclick="location.reload()">刷新</button>
    </div>
  `
})

// Promise 未捕获错误
window.addEventListener('unhandledrejection', (event) => {
  console.error('Unhandled promise rejection:', event.reason)
})
```

---

## 2. 错误分类

### 可恢复错误

用户操作或网络问题导致的错误，可以通过重试、修正输入等方式恢复。

```typescript
// ✅ 可恢复：网络超时
try {
  await api.sendMessage(content)
} catch (err) {
  if (err instanceof TimeoutError) {
    showToast('发送超时，请重试')
    // 提供重试按钮
  }
}

// ✅ 可恢复：输入校验
if (!isValidEmail(email)) {
  showError('请输入有效的邮箱地址')
  // 用户修正后重试
}
```

### 不可恢复错误

程序 bug、数据损坏等，用户无法自行修复。

```typescript
// ✅ 不可恢复：数据损坏
if (!session || !session.id) {
  console.error('Session data corrupted:', session)
  showFatalError('数据异常，请联系支持')
  // 上报错误
  reportError('corrupted_session', { session })
  return
}
```

---

## 3. 用户可见的错误

### 人类可读

```typescript
// ❌ 技术细节
showError('ECONNREFUSED 127.0.0.1:6379')

// ✅ 人类可读
showError('无法连接到服务器，请检查网络后重试')
```

### 提供可操作建议

```typescript
// ❌ 只告诉用户出错了
showError('发送失败')

// ✅ 告诉用户该怎么做
showError('发送失败，请检查网络连接后重试')
```

### 不暴露技术细节

```typescript
// ❌ 暴露内部错误
showError(`Database error: ${err.message}`)

// ✅ 只展示用户需要知道的
showError('操作失败，请稍后重试')
// 内部错误记录到日志
console.error('Database error:', err)
```

---

## 4. 异步错误

### Promise 必须处理

```typescript
// ❌ 未处理的 Promise
api.sendMessage(content)  // 错误被吞

// ✅ try/catch
try {
  await api.sendMessage(content)
} catch (err) {
  handleError(err)
}

// ✅ .catch()
api.sendMessage(content).catch(handleError)
```

### 不吞错误

```typescript
// ❌ 吞掉错误
try {
  await api.sendMessage(content)
} catch (err) {
  // 什么都不做
}

// ✅ 至少记录日志
try {
  await api.sendMessage(content)
} catch (err) {
  console.error('Send message failed:', err)
  showToast('发送失败')
}
```

---

## 5. 错误上报

### 关键错误上报

```typescript
function reportError(type: string, context: Record<string, any>) {
  // 上报到监控平台
  fetch('/api/errors', {
    method: 'POST',
    body: JSON.stringify({
      type,
      context,
      timestamp: Date.now(),
      userAgent: navigator.userAgent,
    }),
  }).catch(() => {
    // 上报失败也不应影响主流程
  })
}

// 使用
try {
  await criticalOperation()
} catch (err) {
  reportError('critical_operation_failed', {
    userId: getCurrentUserId(),
    operation: 'sendMessage',
    error: err.message,
  })
  showToast('操作失败，请重试')
}
```

### 不上传敏感信息

```typescript
// ❌ 包含敏感信息
reportError('login_failed', {
  password: '***',  // 绝不上传
  token: '***',
})

// ✅ 只包含标识信息
reportError('login_failed', {
  userId: 'u123',
  step: 'password_verification',
})
```

---

## 6. Toast 通知

### 错误分类配色

```typescript
type ToastType = 'error' | 'warning' | 'info' | 'success'

function showToast(message: string, type: ToastType = 'info') {
  const toast = document.createElement('rtc-toast')
  toast.message = message
  toast.type = type  // error=红, warning=黄, info=蓝, success=绿
  toast.duration = type === 'error' ? 5000 : 3000  // 错误显示更久
  document.body.appendChild(toast)
}
```

### 使用场景

| 类型 | 场景 | 持续时间 |
|------|------|---------|
| `error` | 操作失败、网络错误 | 5s |
| `warning` | 输入警告、即将过期 | 3s |
| `info` | 操作提示、状态更新 | 3s |
| `success` | 操作成功 | 2s |

---

## 7. 新增代码错误处理检查清单

- [ ] `render()` 方法安全，不会因数据为空而崩溃
- [ ] 所有异步操作有 `try/catch` 或 `.catch()`
- [ ] 错误消息人类可读，不包含技术细节
- [ ] 可恢复错误提供重试或修正建议
- [ ] 关键错误上报到监控平台
- [ ] 上报内容不包含敏感信息
- [ ] 用户可见的错误用 Toast 展示
- [ ] 不可恢复错误展示 fallback UI

---

## 总结

错误处理是产品温度的体现。好的错误处理让用户感到被理解、被引导，而非被抛弃、被困惑。

> _"每一个错误消息都是与用户的一次对话。让它清晰、有用、有温度。"_
