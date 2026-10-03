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

## 4. 并发安全模式

前端虽然没有多线程，但异步操作的交错执行同样会产生竞态。以下是项目中已确立的防护模式。

### Generation Counter 丢弃过期结果

当异步操作的结果可能因为新的操作发起而变得过期时，用 generation counter 检测并丢弃过期结果。

```typescript
// ✅ Generation counter 防止渲染过期数据
private _parseGeneration = 0

private async _parseMarkdown() {
  const generation = ++this._parseGeneration  // 递增，标记本次操作
  const rawHtml = await marked.parse(this.content)  // 异步操作

  // 如果在 await 期间发起了新的解析，generation 已经不匹配
  if (generation !== this._parseGeneration) return  // 丢弃过期结果

  this._renderedHtml = rawHtml
}

// ❌ 不检查，可能渲染旧内容
private async _parseMarkdown() {
  this._renderedHtml = await marked.parse(this.content)  // 可能被后来的操作覆盖
}
```

**适用场景**：Markdown 解析、文件加载、搜索建议、任何"最新一次操作的结果才是正确结果"的场景。

### Generation Counter 防止过期 Promise 破坏共享状态

当异步操作的 `finally` 回调会修改共享状态（如清除 `_connecting` promise）时，仅用 Promise 去重不够——如果中间操作（如 logout）清除了 `_connecting` 并启动了新操作，旧 Promise 的 `finally` 会错误地清除新 Promise 的引用。此时需要 generation counter 让过期 Promise 自我抑制。

```typescript
// ✅ Generation counter 防止 stale finally 破坏新状态
private _connecting?: Promise<void>
private _connectGeneration = 0

private _connectWithRetry(): Promise<void> {
  if (this._connecting) return this._connecting
  this._connecting = this._doConnect()
  return this._connecting
}

private async _doConnect(): Promise<void> {
  // 捕获当前 generation
  const gen = this._connectGeneration
  try {
    await establishConnection()
  } finally {
    // 只有 generation 匹配时才清除 _connecting
    // 如果 logout 已 bump generation，说明 _connecting 已属于新操作，不清除
    if (gen === this._connectGeneration) {
      this._connecting = undefined
    }
  }
}

// logout 时 bump generation 而非直接清除 _connecting
logout() {
  this._connectGeneration++  // 使进行中的 Promise 的 finally 自我抑制
  this._connecting = undefined
  this._rtcProcessor?.destroy()
}

// ❌ 不用 generation，logout 清除 _connecting 后旧 finally 会再次清除
private async _doConnect(): Promise<void> {
  try {
    await establishConnection()
  } finally {
    this._connecting = undefined  // 如果 logout 后发起了新连接，这里会错误清除新引用
  }
}
```

**判断标准**：当 Promise 的 `finally` 回调会修改共享引用（如 `_connecting`、`_refreshing`），且存在"中间操作可能替换该引用"的场景时，使用 generation counter 保护。与基本 generation counter 的区别：基本版丢弃过期结果（数据层），此版保护共享状态不被过期回调破坏（协调层）。

### Promise 去重防止重复执行

连接、初始化等不可并发执行的操作，用 `_promise` 缓存防止重复调用。

```typescript
// ✅ Promise 去重
private _connecting?: Promise<void>

private _connectWithRetry(): Promise<void> {
  if (this._connecting) return this._connecting  // 复用进行中的 Promise
  this._connecting = this._doConnect()
  return this._connecting
}

// ❌ 不防护，多次调用创建多个连接
async connect() {
  await this._doConnect()  // 每次调用都执行
}
```

**适用场景**：WebSocket 连接、Worker 初始化、OAuth token refresh、任何"同时只能执行一次"的操作。

**反模式：不要用 Promise 去重缓存会变化的状态值。** Promise 去重适用于幂等操作（连接、初始化），但**不适用于**每次调用应返回最新值的操作。典型的反面案例是 token refresh——如果用 `_tokenPromise` 去重，一旦 refresh 成功返回了新 token，后续调用会永远返回缓存的旧 Promise（指向已过期的 token），导致认证永久失败。

```typescript
// ❌ 错误：token 被永久缓存
private _tokenPromise?: Promise<string>
async getToken(): Promise<string> {
  if (!this._tokenPromise) {
    this._tokenPromise = this._doRefresh()  // 首次 refresh 成功
  }
  return this._tokenPromise  // 永远返回同一个 token
}

// ✅ 正确：每次 refresh 后清除缓存，下次调用获取最新值
async refreshToken(): Promise<string> {
  const token = await this._doRefresh()
  this._token = token  // 更新缓存的值
  return token
}
```

### 并发防护：共享 Promise 守卫

对于"同时只能执行一次但结果每次不同"的操作（如 token refresh），用 `_refreshing` promise 缓存进行中的操作，完成后清除。并发调用等待同一个 Promise，但不会永久缓存旧值。

```typescript
// ✅ 共享 Promise 守卫：防止并发 refresh 竞态
private _refreshing?: Promise<string>

async refreshToken(): Promise<string> {
  if (this._refreshing) return this._refreshing  // 等待进行中的操作
  this._refreshing = this._doRefresh()
  try {
    const token = await this._refreshing
    this._token = token
    return token
  } finally {
    this._refreshing = undefined  // 必须清除，下次调用获取最新值
  }
}

// ❌ 不防护，并发 refresh 竞态导致 localStorage 损坏
async refreshToken(): Promise<string> {
  const token = await this._doRefresh()  // 两个并发调用同时执行
  this._token = token                     // 结果不确定
  return token
}
```

**适用场景**：token refresh、配置热更新、任何"结果会变但执行必须互斥"的操作。与 Promise 去重的区别：Promise 去重永久缓存（适合连接），共享 Promise 守卫用完即清（适合状态刷新）。

### 异步安全守卫：引用有效性检查

异步操作（`await`）期间，组件可能已被销毁或替换。`await` 之后更新状态前，必须检查引用是否仍然有效。

```typescript
// ✅ 异步安全守卫：await 后检查引用
async setAuthProvider(provider: AuthProvider) {
  this._authProvider = provider
  if (!provider.isLoggedIn()) {
    const success = await provider.refreshToken()  // 异步操作
    // await 期间 provider 可能已被替换（logout/destroy）
    if (this._authProvider !== provider) return     // 守卫：已不是我的 provider
    if (success) this._fireLogin()
  }
}

// ❌ 不检查，可能在已销毁的 provider 上更新状态
async setAuthProvider(provider: AuthProvider) {
  this._authProvider = provider
  if (!provider.isLoggedIn()) {
    const success = await provider.refreshToken()
    // await 期间 this._authProvider 可能已变成另一个 provider
    if (success) this._fireLogin()  // 错误地在新 provider 上触发 login
  }
}
```

**适用场景**：组件生命周期中的异步回调、Provider 模式中的异步初始化、任何 `await` 后需要更新组件状态的场景。与 generation counter 的区别：generation counter 检测"更新的同操作"覆盖了旧操作，引用守卫检测"对象本身已不再有效"。

### 文件上传重试：内存缓存 File 对象

文件上传失败后，用户应能重试而无需重新选择文件。将 `File` 对象缓存在内存 `Map` 中，暴露公共重试方法。重试按钮以 SVG 图标覆盖错误状态，提供清晰的视觉反馈。

```typescript
// ✅ 缓存 File 对象，支持重试
@customElement('rtc-input-area')
export class RtcInputArea extends LitElement {
  /** File objects for retry (fileid -> File) */
  private _fileObjects: Map<string, File> = new Map();
  @state() private _uploadStates: Map<string, string> = new Map();

  // 上传时缓存 File 对象
  private async _handleFiles(files: File[]) {
    for (const file of files) {
      const fileid = generateFileId(file);
      this._fileObjects.set(fileid, file);  // 缓存用于重试
      this._uploadStates.set(fileid, 'loading');
      try {
        const result = await fileStorage.upload({ file, fileid, /* ... */ });
        if (result.syncStatus === 'synced') {
          this._uploadStates.set(fileid, 'loaded');
        } else {
          this._uploadStates.set(fileid, 'error');  // 离线/未同步
        }
      } catch (err) {
        this._uploadStates.set(fileid, 'error');
      }
    }
  }

  // 公共重试方法，供父组件调用
  retryUpload(fileid: string) {
    const file = this._fileObjects.get(fileid);
    if (!file) return;
    this._uploadStates.set(fileid, 'loading');
    this._doUpload(file, fileid);
  }
}

// 父组件连接重试事件
private _handleFileRetry(e: CustomEvent, sessionId: string): void {
  const inputArea = this._getInputAreaBySession(sessionId);
  if (!inputArea) return;
  (inputArea as unknown as { retryUpload: (fileid: string) => void })
    .retryUpload(e.detail.file.fileid);
}

// ❌ 不缓存 File 对象，重试需要用户重新选择
private async _handleFiles(files: File[]) {
  for (const file of files) {
    try {
      await fileStorage.upload({ file });
    } catch {
      showError('上传失败');  // 用户无法重试，只能重新选择
    }
  }
}
```

**约束**：

- File 对象仅在内存中缓存（`Map<string, File>`），不持久化到 IndexedDB（File 对象不可序列化）
- 重试方法必须是公共 API（`retryUpload(fileid)`），供父组件通过事件调用
- 上传结果需检查 `syncStatus`：`synced` 标记 `loaded`，否则标记 `error`
- 重试按钮使用 SVG 图标覆盖错误状态（hover 显示），提供清晰的视觉反馈
- 禁用"网络恢复时自动重试"——给用户手动重试的控制权，避免意外的后台流量（注意：此规则仅适用于用户可见的上传操作。后台同步任务的重试策略参见 [IndexedDB - 同步失败处理](./indexeddb.md#同步失败处理) 和 [TrackedBackgroundTask](./indexeddb.md#trackedbackgroundtask--生命周期感知的后台任务)，后者使用指数退避自动重试）

**适用场景**：

- 文件上传（图片、文档、附件）
- 任何需要"失败后重试"的用户操作，其中原始输入数据需要保留

---

### Per-Key 锁防止资源冲突

同一资源（如文件）的并发写入需要互斥。用 `Set<string>` 或 `Map` 追踪进行中的操作。

```typescript
// ✅ Per-file 锁
const _savingFiles = new Set<string>()

async function handleEditorSave(filePath: string) {
  if (_savingFiles.has(filePath)) return  // 正在保存，跳过
  _savingFiles.add(filePath)
  try {
    await virtualFS.write(filePath, content)
  } finally {
    _savingFiles.delete(filePath)  // 必须清理
  }
}

// ❌ 无锁，并发写入导致数据损坏
async function handleEditorSave(filePath: string) {
  await virtualFS.write(filePath, content)  // 可能并发执行
}
```

### Observer 回调错误隔离

当使用观察者模式（listener/callback 注册与通知）时，单个 listener 抛出的异常不应中断其他 listener 的通知链。每个回调调用必须包裹在 `try/catch` 中。

```typescript
// ✅ 错误隔离：单个 listener 异常不影响其他 listener（persistence/src/file-op-coordinator.ts）
private _notifyAll(md5: string, ext: string, status: FileSyncStatus, errorMessage?: string): void {
  for (const listener of this._listeners) {
    try {
      listener(md5, ext, status, errorMessage);
    } catch (err) {
      log.warn('FileOpCoordinator: listener threw:', err);
      // 继续通知其他 listener
    }
  }
}

// ❌ 不隔离：第一个抛错的 listener 会中断整个通知链
private _notifyAll(md5: string, ext: string, status: FileSyncStatus): void {
  for (const listener of this._listeners) {
    listener(md5, ext, status);  // 如果这个 listener 抛错，后面的 listener 都不会被调用
  }
}
```

**约束**：

- 所有遍历调用多个回调/监听者的代码必须用 `try/catch` 隔离每个调用
- 回调异常使用 `warn` 级别日志记录，包含错误对象（保留 stack trace）
- 监听器注册方法（`onStatusChange()`）必须返回 `dispose` 函数，调用方在组件/模块销毁时调用，防止泄漏
- `clearAllListeners()` 仅在容器销毁时调用（如持久化层 `close()`）

**防止的问题**：一个有 bug 的 listener（如访问已销毁的 DOM 引用）导致所有后续 listener 不执行，UI 层无法感知文件状态变化。

**适用场景**：事件总线（UIUpdateBus）、状态变更通知（FileOpCoordinator）、插件系统、任何"一对多"的回调分发。

---

## 5. 动态导入重试模式

在开发环境中，Vite 的热更新可能导致 chunk hash 变化，使得已经加载的模块引用失效（stale chunk error）。对于通过 `import()` 动态加载的大型依赖（如 DOMPurify、highlight.js、marked），需要提供重试机制。

### 重试包装函数

```typescript
// ✅ 动态导入重试：处理开发环境的 stale chunk 错误
private async _loadDOMPurify(attempts = 2): Promise<typeof import('dompurify').default> {
    try {
        const {default: DOMPurify} = await import('dompurify');
        return DOMPurify;
    } catch (err) {
        if (attempts > 0) {
            console.warn('[rtc-agent] DOMPurify load failed, retrying...', err);
            return this._loadDOMPurify(attempts - 1);
        }
        throw err;
    }
}

// ❌ 不重试，开发环境容易失败
private async _loadDOMPurify(): Promise<typeof import('dompurify').default> {
    const {default: DOMPurify} = await import('dompurify');
    return DOMPurify;  // 开发环境 chunk 过期时永久失败
}
```

### 批量加载多个模块

当需要同时加载多个模块时，封装成单一方法：

```typescript
// ✅ 批量加载 + 重试
private async _loadModules(attempts = 2): Promise<{
    marked: typeof import('marked').marked;
    DOMPurify: typeof import('dompurify').default;
    hljs: typeof import('highlight.js').default;
}> {
    try {
        const [marked, DOMPurify, hljs] = await Promise.all([
            import('marked'),
            import('dompurify'),
            import('highlight.js'),
        ]);
        return {marked: marked.marked, DOMPurify: DOMPurify.default, hljs: hljs.default};
    } catch (err) {
        if (attempts > 0) {
            console.warn('[rtc-agent] modules load failed, retrying...', err);
            return this._loadModules(attempts - 1);
        }
        throw err;
    }
}
```

### 规则

- **默认重试 2 次**（`attempts = 2`），足以覆盖 stale chunk 场景
- **仅在 catch 中重试**，成功路径无额外开销
- **重试时记录 warn 日志**，便于开发调试
- **最终失败后 throw**，让调用方处理（如展示 fallback UI）

### 适用场景

- Markdown 渲染器（marked + DOMPurify + highlight.js）
- 语法高亮库
- 其他通过 `import()` 动态加载的大型依赖

---

## 6. AbortSignal 快速失败模式

当异步操作支持取消时，必须在操作的关键检查点验证信号状态，而非只在操作开始时检查一次。

```typescript
// ✅ 快速失败 + 清理
async function uploadFile(file: File, signal?: AbortSignal): Promise<void> {
  // 1. 操作前快速失败
  if (signal?.aborted) {
    throw new DOMException('Upload aborted', 'AbortError')
  }

  // 2. 注册 abort 监听器
  let cleanupAbort: (() => void) | undefined
  if (signal) {
    const onAbort = () => controller.abort()
    signal.addEventListener('abort', onAbort, { once: true })
    cleanupAbort = () => signal.removeEventListener('abort', onAbort)
  }

  try {
    // 3. 执行可能耗时的操作
    await doUpload(file)

    // 4. 操作后再次检查（可能在 await 期间被取消）
    if (signal?.aborted) {
      throw new DOMException('Upload aborted', 'AbortError')
    }
  } finally {
    // 5. 必须清理监听器
    cleanupAbort?.()
  }
}

// ❌ 只检查一次，await 期间取消无效
async function uploadFile(file: File, signal?: AbortSignal): Promise<void> {
  if (signal?.aborted) throw new DOMException('Aborted', 'AbortError')
  await doUpload(file)  // 如果此处被取消，操作仍然继续
}
```

**规则**：

- 每个 `await` 之后检查 `signal?.aborted`，防止在等待期间发起的取消被忽略

- abort 监听器必须在 `finally` 中清理，防止内存泄漏

- 抛出 `DOMException('...', 'AbortError')` 而非普通 `Error`，让调用方能用 `err.name === 'AbortError'` 识别

---

## 7. 异步错误

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

## 8. 错误上报

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

## 9. Toast 通知

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

|类型|场景|持续时间|
|---|---|---|
|`error`|操作失败、网络错误|5s|
|`warning`|输入警告、即将过期|3s|
|`info`|操作提示、状态更新|3s|
|`success`|操作成功|2s|

---

## 10. 日志规范

### 必须使用 `createLogger`

所有前端模块必须使用 `@rtc-agent/client` 提供的 `createLogger` 创建模块级日志器，不直接使用 `console.log`。

```typescript
// ✅ 模块级 scoped logger
import { createLogger } from '@rtc-agent/client'
const log = createLogger('WorkerBridge')

log.info('connected')             // [WorkerBridge] connected
log.debug('payload', { ... })     // [WorkerBridge] payload { ... }
log.warn('retrying', attempt)     // [WorkerBridge] retrying 2
log.error('failed', err)          // [WorkerBridge] failed Error: ...

// ❌ 直接使用 console
console.log('connected')          // 无模块标识，生产环境无法过滤
console.error('failed', err)      // 生产环境可能暴露给用户
```

### 日志级别

|级别|使用场景|
|---|---|
|`debug`|开发调试信息，生产环境默认不输出|
|`info`|关键业务事件：连接建立、文件上传完成|
|`warn`|可恢复异常：重试、降级|
|`error`|不可恢复错误：需人工介入|

### scope 命名

`createLogger` 的 `scope` 参数使用 PascalCase，与模块/类名一致。

```typescript
// ✅ 与模块名一致
const log = createLogger('FileStorage')      // utils/file-storage.ts
const log = createLogger('WorkerBridge')     // worker-bridge.ts
const log = createLogger('SessionTree')      // controllers/session-tree.controller.ts

// ❌ 随意命名
const log = createLogger('file')             // 不明确
const log = createLogger('my-module')        // 不统一
```

### 规则

- **生产环境**默认 `info` 级别，`debug` 不输出
- **开发环境**默认 `debug` 级别，输出所有日志
- 不打印敏感信息（token、password）
- 错误日志包含 error 对象（保留 stack trace）

---

## 11. 新增代码错误处理检查清单

- [ ] `render()` 方法安全，不会因数据为空而崩溃
- [ ] 所有异步操作有 `try/catch` 或 `.catch()`
- [ ] 错误消息人类可读，不包含技术细节
- [ ] 可恢复错误提供重试或修正建议
- [ ] 关键错误上报到监控平台
- [ ] 上报内容不包含敏感信息
- [ ] 用户可见的错误用 Toast 展示
- [ ] 不可恢复错误展示 fallback UI
- [ ] 支持取消的操作检查 AbortSignal（每个 await 后验证）
- [ ] Promise 去重仅用于幂等操作，不缓存会变化的状态值
- [ ] 状态刷新操作（如 token refresh）使用共享 Promise 守卫，完成后清除缓存
- [ ] `await` 后更新状态前检查对象引用仍有效（异步安全守卫）
- [ ] 同一实体的并发写操作通过 Promise 链序列化
- [ ] Promise 的 `finally` 修改共享引用时，使用 generation counter 防止过期回调破坏新状态
- [ ] 文件上传采用三阶段管道（本地缓存 → 即时显示 → 后台同步），离线时文件仍可用
- [ ] 文件上传失败后支持重试，重试从缓存的 File 对象恢复，不要求用户重新选择
- [ ] Observer/事件总线的回调分发用 `try/catch` 隔离每个 listener，单个 listener 异常不中断通知链

---

## 总结

错误处理是产品温度的体现。好的错误处理让用户感到被理解、被引导，而非被抛弃、被困惑。

> _"每一个错误消息都是与用户的一次对话。让它清晰、有用、有温度。"_
