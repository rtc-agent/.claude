# IndexedDB 持久化规范

> _本地存储不是服务端的镜像，而是产品体验的加速器。选择正确的持久化策略，比选择正确的数据结构更重要。_

---

## 1. Dexie.js 唯一标准

### 必须通过 Dexie.js 操作 IndexedDB

所有 IndexedDB 操作必须通过 Dexie.js 封装层。不直接使用原生 `indexedDB` API，不引入 `localforage`、`idb` 等其他 wrapper。

```typescript
// ✅ 使用 Dexie
import { Dexie } from 'dexie'

const db = new Dexie('mydb')
db.version(1).stores({ items: 'id, name' })
await db.items.put({ id: '1', name: 'test' })

// ❌ 原生 IndexedDB
const request = indexedDB.open('mydb')
request.onsuccess = (event) => { ... }

// ❌ 其他 wrapper
import localforage from 'localforage'
await localforage.setItem('key', value)
```

### 统一版本

当前使用 `dexie: ^4.4.5`。不升级或降级到不兼容版本。

---

## 2. 数据库版本与迁移

### 每次 schema 变更递增版本号

```typescript
// ✅ 新增版本
db.version(7).stores({
  items: 'id, name, createdAt',  // 新增索引
  newTable: 'id, itemId',        // 新增表
})

// ❌ 修改已有版本的 schema
db.version(6).stores({
  items: 'id, name, createdAt',  // 不应修改 v6
})
```

### 迁移逻辑必须处理数据转换

当主键、字段类型或语义发生变化时，迁移必须转换已有数据。

```typescript
db.version(2).stores({
  sessions: 'client_id, server_id, sync_status',
}).upgrade(tx => {
  // 将旧的 server_id 主键迁移到 client_id
  return tx.table('sessions').toCollection().modify(session => {
    session.client_id = generateClientId()
    session.server_id = session.id
    delete session.id
  })
})
```

### 不跳版本

版本号必须连续递增。即使中间版本从未发布，也要保留迁移链。

---

## 3. 主键策略

### client_id 为主键

所有需要同步的实体使用客户端生成的 ID 作为主键。

```typescript
interface LocalSession {
  client_id: string     // 主键，客户端生成（UUID）
  server_id?: string    // 服务端 ID，同步后回填
  sync_status: SyncStatus
  // ...
}
```

### server_id 作为索引

服务端 ID 作为索引字段，用于服务端推送数据时的查找和映射。

```typescript
db.version(1).stores({
  sessions: 'client_id, server_id, sync_status, owner_ref_id',
  //           ↑ 主键      ↑ 索引（用于服务端→本地映射）
})
```

### ID 生成方式统一

使用项目中已有的 ID 生成方式（UUID 或 nanoid），不自创生成逻辑。

---

## 4. 持久化策略分类

新增数据时，必须明确选择以下四种策略之一：

### 离线优先（Offline-First）

**适用场景**：用户输入、消息发送等不能丢失的操作。

**行为**：先写本地（`sync_status = 'pending'`），立即返回，后台同步到服务端。

```typescript
// ✅ 离线优先：先写本地
async sendMessage(content: string) {
  const message = { client_id: genId(), content, sync_status: 'pending' }
  await db.messages.put(message)        // 先写本地
  this.uiBus.emit('message.created', message)  // 立即通知 UI
  this._syncToServer(message)           // 后台同步
}
```

### 服务端优先（Server-First）

**适用场景**：查询类数据、配置数据、需要从服务端获取最新状态的。

**行为**：先请求服务端，成功后写入本地缓存。

```typescript
// ✅ 服务端优先
async getSessions() {
  const sessions = await api.getSessions()    // 先请求服务端
  await db.sessions.bulkPut(sessions)         // 写入本地缓存
  return sessions
}
```

### 仅缓存（Cache-Only）

**适用场景**：列表、搜索结果等可从服务端完全重建的数据。

**行为**：本地仅作为加速层，服务端为唯一数据源。可随时清除。

```typescript
// ✅ 仅缓存
async searchMessages(query: string) {
  const cached = await db.messages.where('content').startsWith(query).toArray()
  if (cached.length > 0) return cached        // 有缓存先用

  const results = await api.search(query)     // 无缓存请求服务端
  await db.messages.bulkPut(results)
  return results
}
```

### 不持久化

**适用场景**：临时 UI 状态（当前选中的 tab、展开/折叠状态等）。

**行为**：使用 `localStorage` 或 `sessionStorage`，不进入 IndexedDB。

```typescript
// ✅ 不持久化到 IndexedDB
localStorage.setItem('rtc_active_tab', 'chat')
```

### 判断标准

| 问 | 答案 | 策略 |
|-----|------|------|
| 数据丢失用户能否接受？ | 不能 | 离线优先 |
| 数据能否从服务端完整重建？ | 能 | 仅缓存 |
| 数据是否以服务端为准？ | 是 | 服务端优先 |
| 只是 UI 状态？ | 是 | 不持久化 |

---

## 5. 同步状态模型

### 四态生命周期

离线优先的数据需要 `sync_status`，状态机包含四个状态：

```typescript
// ✅ 四态生命周期
type SyncStatus = 'pending' | 'syncing' | 'synced' | 'failed'

interface LocalMessage {
  client_id: string
  content: string
  sync_status: SyncStatus
  error_message?: string   // failed 时记录原因
  synced_at?: number       // synced 时记录时间
}

// ✅ 服务端优先 / 仅缓存：不需要 sync_status
interface CachedSession {
  client_id: string
  server_id: string
  // 无 sync_status
}
```

状态转移：

```text
         ┌───────┐
    ┌───►│pending│◄──── recoverStaleSyncing()
    │    └───┬───┘      (启动时降级 syncing → pending)
    │        │ transitionSyncStatus('pending', 'syncing')
    │        ▼
    │    ┌───────┐
    │    │syncing│
    │    └───┬───┘
    │   ┌────┴────┐
    │   ▼         ▼
    │ ┌──────┐ ┌──────┐
    │ │synced│ │failed│─── retry ──► pending
    │ └──────┘ └──────┘
    │                    (用户手动重试)
    └────────────────────
```

### 原子状态转移

状态转移必须在 Dexie 事务中原子执行，通过 precondition 检查防止竞态：

```typescript
// ✅ 原子转移：precondition + modify
async transitionSyncStatus(
  key: string,
  from: SyncStatus | SyncStatus[],
  to: SyncStatus,
  metadata?: { errorMessage?: string }
): Promise<boolean> {
  const fromSet = new Set(Array.isArray(from) ? from : [from])
  let transitioned = false
  await db.fileCache.where('[md5+ext]').equals(key).modify(entry => {
    if (!fromSet.has(entry.syncStatus as SyncStatus)) {
      return  // precondition 不满足，跳过
    }
    entry.syncStatus = to
    if (metadata?.errorMessage) entry.errorMessage = metadata.errorMessage
    transitioned = true
  })
  return transitioned
}

// ❌ 非原子：先读后写，竞态窗口
async transitionSyncStatus(key: string, to: SyncStatus) {
  const entry = await db.fileCache.get(key)  // 读取
  if (entry.syncStatus === 'pending') {
    entry.syncStatus = to                     // 写入时可能已被其他操作修改
    await db.fileCache.put(entry)
  }
}
```

### 启动时恢复残留状态

浏览器崩溃、tab 强制关闭可能使条目卡在 `syncing` 状态。启动时必须扫描并降级：

```typescript
// ✅ 启动恢复：syncing → pending
async recoverStaleSyncing(): Promise<void> {
  await db.fileCache
    .where('syncStatus').equals('syncing')
    .modify(entry => {
      entry.syncStatus = 'pending'
      entry.errorMessage = 'Recovered from stale syncing state'
    })
}

// ❌ 不恢复，syncing 条目永远无法重试
```

### 驱逐保护

LRU 驱逐和过期清理只能驱逐 `synced` 状态的条目。`pending`、`syncing`、`failed` 条目包含未同步的数据，驱逐会导致数据丢失。

```typescript
// ✅ 驱逐前过滤状态
async evictLRU(maxBytes: number): Promise<void> {
  const candidates = await db.fileCache
    .where('syncStatus').equals('synced')  // 只驱逐已同步的
    .sortBy('lastAccessedAt')
  // ...
}

// ❌ 不检查状态，可能驱逐未同步的条目
const candidates = await db.fileCache.orderBy('lastAccessedAt').limit(10).toArray()
```

### 同步失败处理

- `failed` 状态的条目必须有重试机制
- 重试策略：指数退避，最大重试次数有限
- 超过重试次数后通知用户，提供手动重试按钮，不静默丢弃
- 重试 delay 必须可被取消（组件销毁时不阻塞关闭）
- 注意：后台同步任务的重试 delay 需要可取消（此处），而用户可见的文件上传操作禁用自动重试（参见[文件上传重试](./error-handling.md#文件上传重试内存缓存-file-对象)），两者适用场景不同

### 错误类型判别：取消 vs 失败

当异步操作同时支持 `AbortSignal`（取消）和状态机转换时，必须区分"取消"和"操作失败"——取消不应触发 `failed` 状态转换。`AbortError` 表示操作被有意中止（如组件销毁、生命周期关闭），状态应保持不变（通常是 `pending`，以便后续恢复）；只有真正的操作错误（网络失败、S3 错误等）才应转换到 `failed` 状态。

```typescript
// ✅ 错误类型判别：取消 ≠ 失败
async run(spec: { from: SyncStatus; to: SyncStatus; action: (...) }) {
  // 1. 转换到中间状态（如 'syncing'）
  const transitioned = await repo.transitionSyncStatus(key, spec.from, spec.to);
  if (!transitioned) return { success: false, aborted: true };

  try {
    await spec.action(freshEntry, signal);
    // 2. 成功：转换到 'synced'
    await repo.transitionSyncStatus(key, spec.to, 'synced');
    return { success: true };
  } catch (err) {
    // 3. 取消：状态保持不变（不转 'failed'），以便恢复
    if (err instanceof DOMException && err.name === 'AbortError') {
      return { success: false, aborted: true };
    }
    if (err instanceof LifecycleClosedError) {
      return { success: false, aborted: true };  // DB 已关闭，不操作
    }
    // 4. 操作失败：转换到 'failed'
    await repo.transitionSyncStatus(key, spec.to, 'failed', {
      errorMessage: String(err),
    });
    return { success: false, error: err };
  }
}

// ❌ 不区分错误类型，取消也会标记为 failed
try {
  await action(entry, signal);
} catch (err) {
  // AbortError 也被当作失败处理 → 状态变成 'failed'
  // 用户看到"上传失败"，但实际只是页面切换导致的取消
  await repo.transitionSyncStatus(key, 'syncing', 'failed');
}
```

**错误分类规则**：

| 错误类型                       | 语义               | 状态动作                               |
| ------------------------------ | ------------------ | -------------------------------------- |
| `AbortError` (DOMException)    | 调用方主动取消     | 状态不变（保持 `pending`/`syncing`）   |
| `LifecycleClosedError`         | 持久化层已关闭     | 状态不变，停止重试                     |
| 其他 Error（网络、S3 等）      | 操作失败           | 转换到 `failed`                        |

**防止的问题**：取消被误判为失败 → 状态变成 `failed` → 用户看到错误的"上传失败"提示 → 实际上操作只是被暂停，应可恢复。

**约束**：

- 使用 `AbortController` + 状态机的异步操作必须区分 `AbortError` 和操作错误
- `AbortError` 不触发 `failed` 状态转换，保持当前状态（通常是中间状态或 `pending`）
- 使用 `FileSyncTransaction` 模式（`persistence/src/file-sync-transaction.ts`）封装此逻辑，不在业务代码中重复实现

### 大文件分片上传进度持久化

大文件（>5MB）上传使用 S3 multipart upload 时，必须在 IndexedDB 中持久化每个分片的上传进度，以支持页面关闭或网络中断后从断点恢复（而非重新开始）。

```typescript
// ✅ 持久化分片进度，支持断点恢复
interface UploadProgressEntry {
  md5: string
  ext: string
  uploadId: string       // S3 multipart upload ID
  s3Key: string
  fileSize: number
  partSize: number       // 每个分片大小（S3 最小 5MB）
  parts: UploadPartRecord[]  // 每个分片的进度
  createdAt: number
  expiresAt: number      // upload ID 有效期（S3 默认 7 天）
}

interface UploadPartRecord {
  partNumber: number
  etag?: string          // 上传成功后 S3 返回的 ETag
  status: 'pending' | 'uploading' | 'completed'
}

// 恢复流程：
// 1. 启动时扫描未完成的 UploadProgressEntry
// 2. 对每个 entry，只重新上传 status !== 'completed' 的分片
// 3. 所有分片完成后调用 CompleteMultipartUpload
// 4. 清除进度记录，更新 fileCache 的 syncStatus 为 'synced'
```

**约束**：

- 分片进度记录与 `fileCache` 条目通过 `(md5, ext)` 关联
- `uploadId` 有过期时间（S3 默认 7 天），过期后需重新发起 multipart upload
- 配合 [Offline-First File Upload](./web-components.md#offline-first-file-upload三阶段上传) 的三阶段管道使用：Phase 1 本地缓存 → Phase 3 后台 S3 同步时启用分片进度追踪
- 所有分片完成后必须清除进度记录，防止 IndexedDB 无限增长

---

## 6. EntityRepository 模式

### 新增实体必须通过 EntityRepository

不直接操作 Dexie 表，统一通过 `EntityRepository` 的 upsert 方法。

```typescript
// ✅ 通过 EntityRepository
const repo = getEntityRepository()
const result = await repo.upsertSession({ client_id: id, title: 'New' })

// ❌ 直接操作 Dexie
const db = getDatabase()
await db.sessions.put({ client_id: id, title: 'New' })
```

### 字段级变更检测

`EntityRepository` 使用 `microdiff` 检测字段级变更，并通过 `UIUpdateBus` 通知 UI。新增实体自动享受此机制。

### silent 模式

批量写入或内部操作时使用 `silent: true` 抑制 UI 事件。

```typescript
await repo.upsertSession(data, { silent: true })
```

### Upsert 事务保护

所有 `upsert*` 方法必须将 read-modify-write 包裹在 `db.transaction('rw', ...)` 中，防止并发多 Tab 更新导致数据丢失。Dexie 支持事务嵌套——如果已在事务中，会复用父事务。

```typescript
// ✅ 事务保护：read-modify-write 原子执行
async upsertSession(session, syncStatus, options?) {
  return db.transaction('rw', db.sessions, async () => {
    // 1. 读取现有记录（在事务内）
    const existing = await db.sessions.where('client_id').equals(session.client_id).first();

    if (existing) {
      // 2. 合并更新
      const updated = {
        ...existing,
        ...session,
        // preserveSyncStatus: 聚合更新（如 turn 计数）时保留现有 sync_status
        sync_status: options?.preserveSyncStatus ? existing.sync_status : syncStatus,
        server_id: session.server_id || existing.server_id,
      };
      await db.sessions.put(updated);
    } else {
      // 3. 创建新记录，填充安全默认值
      await db.sessions.put({ ...session, client_id: session.client_id || '', ... });
    }
  });
}

// ❌ 无事务保护：两个 Tab 同时 upsert 可能丢失一方的修改
async upsertSession(session, syncStatus) {
  const existing = await db.sessions.where('client_id').equals(session.client_id).first();
  // ← 此处另一个 Tab 可能已修改了同一条记录
  await db.sessions.put({ ...existing, ...session });
}
```

### `preserveSyncStatus` 选项

当服务端推送数据更新本地实体（如聚合更新 turn 计数、session 元数据）时，不得覆盖本地 `sync_status`。服务端推送的 `syncStatus` 参数默认是 `'synced'`，但本地可能有 `pending` 或 `failed` 的待同步数据——直接覆盖会导致数据丢失。

```typescript
// ✅ 聚合更新：保留本地 sync_status
await repo.upsertSession(serverData, 'synced', { preserveSyncStatus: true });
// 本地 sync_status = 'pending' → 保持 'pending'，不回归为 'synced'

// ❌ 默认行为：覆盖 sync_status，可能丢失待同步数据
await repo.upsertSession(serverData, 'synced');
// 本地 sync_status = 'pending' → 被覆盖为 'synced'，待同步数据丢失！
```

**约束**：

- 所有 `upsert*` 方法必须包裹在 `db.transaction('rw', ...)` 中
- 跨表操作（如 session + turns）必须在同一个事务中，锁住所有涉及的表
- 服务端推送触发的更新必须使用 `preserveSyncStatus: true`
- 用户主动操作（创建、编辑）使用默认行为（设置新的 `sync_status`）

---

## 7. VirtualFS 规范

### 路径归一化

所有路径必须通过 `normalizePath()` 处理，自动去除尾部斜杠、统一分隔符。

### 禁止路径遍历

```typescript
// ❌ 路径遍历，被阻止
await virtualFS.read('../../../etc/passwd')

// ✅ 合法路径
await virtualFS.read('/functions/helper.js')
```

### 新增文件类型必须注册

新增文件类型时，必须在 `type` 索引中可查询。

---

## 8. localStorage / sessionStorage 使用边界

### 仅用于轻量 UI 状态

| 用途 | 存储 | key |
|------|------|-----|
| 设备标识 | localStorage | `rtc_device_id` |
| 设备名称 | localStorage | `rtc_device_name` |
| OAuth token | localStorage | `rtc_access_token` / `rtc_refresh_token` |
| 工作模式 | localStorage | `rtc_mode` |
| OAuth CSRF state | sessionStorage | `rtc_oauth_state` |

### 约束

- **key 统一 `rtc_` 前缀**：避免与宿主页面冲突
- **不存业务数据**：消息、会话、文件等必须走 IndexedDB
- **不存大量数据**：localStorage 有 5-10MB 限制
- **敏感数据谨慎**：token 存 localStorage 有 XSS 风险，评估后决定

---

## 9. 性能约束

### 批量写入

```typescript
// ✅ 批量写入
await db.messages.bulkPut(messages)

// ❌ 逐条写入
for (const msg of messages) {
  await db.messages.put(msg)  // 每次都是独立事务
}
```

### 大查询分页

```typescript
// ✅ 分页查询
const PAGE_SIZE = 50
const messages = await db.messages
  .orderBy('created_at')
  .reverse()
  .offset(page * PAGE_SIZE)
  .limit(PAGE_SIZE)
  .toArray()
```

### 不在事务中做耗时计算

```typescript
// ❌ 事务中做耗时操作
await db.transaction('rw', db.messages, async () => {
  const heavy = computeExpensiveResult()  // 阻塞事务
  await db.messages.put(heavy)
})

// ✅ 先计算，再写入
const heavy = computeExpensiveResult()
await db.transaction('rw', db.messages, async () => {
  await db.messages.put(heavy)
})
```

### 持久化 UI 更新队列

当 IndexedDB 变更需要通知 UI 层时，变更事件先写入 IndexedDB（fire-and-forget），再分发到内存中的订阅者。这样即使 tab 被关闭再重新打开，也能通过回放持久化队列中的事件来追平状态。

```typescript
// ✅ 事件先持久化，再分发
async emit(event: UIUpdateEvent) {
  // 1. 持久化事件（structuredClone + JSON 降级）
  const seq = await this._persistEvent(event)

  // 2. 分发到内存中的订阅者
  for (const subscriber of this._subscribers) {
    subscriber({ ...event, seq })
  }
}

// 新 tab 打开时回放未处理的事件
async catchUp(lastSeq: number) {
  const pending = await db.uiEvents
    .where('seq').above(lastSeq)
    .sortBy('seq')
  for (const event of pending) {
    this._dispatchToSubscriber(event)
  }
}
```

**规则**：

- 事件使用 `structuredClone` 序列化，非克隆对象降级为 `JSON.parse(JSON.stringify())`
- 等待 `add()` 完成以获取实际的自增 `seq`，不使用预估序号
- Worker 已持久化的事件使用 `skipPersist: true` 避免重复写入
- 回放时使用 `seqOverride` 保留原始序号

### 事件消费者注册先于 Catch-Up 回放

当持久化队列需要回放遗漏事件时，**实时事件回调必须在 catch-up 之前注册**。这样 catch-up 期间到达的实时事件会更新游标，回放查询自动跳过已投递的事件，同时防止事件丢失和重复。

```typescript
// ✅ 先注册回调，再 catch-up
async init(): Promise<void> {
  // 1. 注册实时事件回调（先！）
  await this._core.registerCallback(this._proxiedCallbacks)

  // 2. Catch-up 回放遗漏事件（后！）
  //    回放期间到达的实时事件会通过回调更新 _lastProcessedSeq，
  //    catch-up 查询条件 seq > _lastProcessedSeq 自动跳过已投递的事件
  await this._doCatchUp()
}

// 实时回调：更新游标
onUIUpdate: (payload: UIUpdatePayload) => {
  bus.publish(payload.event, { skipPersist: true, seqOverride: payload.seq })
  if (payload.seq > this._lastProcessedSeq) {
    this._updateLastProcessedSeq(payload.seq)
  }
}

// Catch-up 回放：幂等守卫跳过已投递事件
while (hasMore && !signal.aborted) {
  const result = await this._core.getCatchUpEvents(fromSeq)
  for (const entry of result.entries) {
    // 幂等守卫：跳过实时回调已投递的事件
    if (entry.seq <= this._lastProcessedSeq) continue
    deliverEvent(entry)
  }
  fromSeq = result.entries[result.entries.length - 1].seq
  hasMore = result.hasMore
}

// ❌ 先 catch-up 再注册回调
async init(): Promise<void> {
  await this._doCatchUp()            // 回放期间到达的实时事件无人接收 → 丢失
  await this._core.registerCallback(...)  // 注册太晚，遗漏了 catch-up 期间的事件
}
```

**适用场景**：Worker 持久化 UI 更新队列的 catch-up、任何"从持久化队列回放 + 实时事件并行投递"的系统。

**规则**：

- 回调注册必须在 catch-up 之前完成
- Catch-up 循环内必须有 seq 幂等守卫（`seq <= _lastProcessedSeq` 则跳过）
- 实时回调必须同步更新游标（`_lastProcessedSeq`），不能延迟
- 大量事件回放时使用时间切片（`performance.now()` + 帧预算）避免阻塞主线程

### Gap 检测：只检查起始间隙，不检查中间间隙

Catch-up 回放时的 gap 检测**只检查起始间隙**（返回的最低 seq > fromSeq + 1），**故意不检查中间间隙**。IndexedDB 的自增主键在事务失败（约束冲突、配额超限）时会跳过序号，这不代表数据丢失——事件从未被持久化。如果 gap 检测检查所有间隙，会将正常的序号跳过误判为数据丢失，触发不必要的 gap-fill。

```typescript
// ✅ 只检查起始间隙
const entries = await db.ui_updates.where('seq').above(fromSeq).limit(limit + 1).toArray();
let hasGap = false;
if (fromSeq > 0 && entries.length > 0) {
    const minSeq = entries[0].seq;
    if (minSeq > fromSeq + 1) {
        hasGap = true;  // TTL 清理删除了 fromSeq 之后最早的事件 → 真正的数据丢失
    }
}
// 中间间隙（如 entries[2].seq > entries[1].seq + 1）不检查：
// IndexedDB auto-increment 在事务失败时跳过序号，不代表数据丢失

// ❌ 检查所有间隙 → 误判
for (let i = 1; i < entries.length; i++) {
    if (entries[i].seq > entries[i-1].seq + 1) {
        hasGap = true;  // 误报！可能只是 auto-increment 跳过
    }
}
```

**约束**：修改 gap 检测逻辑时，不得添加中间间隙检查。如果需要检测中间间隙，必须先确认序号跳过不是由 IndexedDB auto-increment 行为引起的。

### 批量操作 Suspend/Resume — 防止 UI 抖动

当一批操作会在短时间内产生大量 UI 更新事件（如 gap-fill 回放、批量同步）时，逐条分发会导致 UI 组件频繁重渲染。`UIUpdateBus.suspend()` / `resume()` 提供引用计数的暂停/恢复机制：suspend 期间事件仅在内存中收集（不持久化、不分发），resume 时统一发出 `BulkUpdateEvent`，UI 组件据此一次性 reload 而非逐条处理。

```typescript
// ✅ 批量操作包裹 suspend/resume
bus.suspend()
try {
  for (const event of batchEvents) {
    await bus.publish(event)  // suspend 期间：仅收集，不持久化、不分发
  }
} finally {
  bus.resume()  // 触发 BulkUpdateEvent，UI 组件一次性 reload
}

// UI 组件订阅 BulkUpdate 而非逐条处理
bus.onBulkUpdate(({ entities, eventCount }) => {
  if (entities.has('message')) {
    this._reloadMessages()  // 一次性 reload，不处理每个 event
  }
})

// ❌ 批量操作不使用 suspend → 100 条事件 = 100 次 UI 重渲染
for (const event of batchEvents) {
  await bus.publish(event)  // 每条都触发所有 listener
}
```

**安全机制**：

- **引用计数**：`suspend()` 递增深度，`resume()` 递减，只有深度归零时才真正恢复。支持嵌套 suspend（如 gap-fill 内部触发同步）
- **安全超时**（30 秒）：防止 bug 导致永久 suspend。超时后 `_forceResume()` 强制恢复，重置深度，flush 已收集事件
- **suspend 期间不持久化**：避免 resume 分发后与 catch-up 回放重复投递。suspend 事件是纯本地批处理，不进入 IndexedDB 队列
- **`resume()` 不匹配 `suspend()` 时忽略**：防止多调用 resume 导致意外 flush

**约束**：

- 所有可能产生 10+ 条事件的批量操作必须包裹 `suspend()` / `resume()`
- `suspend()` 和 `resume()` 必须成对调用（推荐 `try/finally` 模式）
- 订阅方必须实现 `onBulkUpdate` 处理批量通知，不能只依赖逐条 `onUIUpdate`
- suspend 期间的持久化跳过是有意设计——修改此行为需同步评估 catch-up 重复投递风险

---

## 10. 持久化层生命周期管理

持久化层（`persistence` 包）有明确的生命周期：`open()` -> 活跃 -> `close()`。所有后台任务、资源、DB 操作必须感知此生命周期，否则会导致 `DatabaseClosedError`、资源泄漏或 tab 关闭后的幽灵写入。

### LifecycleGuard — 生命周期状态守卫

`close()` 之后、`closeDatabase()` 之前存在时间窗口，期间后台任务仍可能尝试 DB 操作。`LifecycleGuard` 提前拦截，将 Dexie 的不透明 `DatabaseClosedError` 替换为语义明确的 `LifecycleClosedError`。

```typescript
// ✅ 每次 DB 操作前检查
const guard = new LifecycleGuard('FileCacheRepository');

async updateSyncStatus(...) {
  guard.assertActive();  // 关闭状态直接抛出 LifecycleClosedError
  await db.fileCache.where(...).modify(...);
}

// ✅ run() 安全包装：关闭时返回 fallback 而非抛异常
const result = await guard.run(() => fetchExpensiveData(), fallbackValue);

// ❌ 不检查，依赖 Dexie 抛出的不透明错误
async updateSyncStatus(...) {
  await db.fileCache.where(...).modify(...);  // close() 后抛 DatabaseClosedError
}
```

**约束**：

- 每个持有 Dexie 引用的 Repository 必须接受 `LifecycleGuard` 作为构造参数
- 所有 DB 操作方法的第一行调用 `guard.assertActive()`
- `close()` 必须在 `closeDatabase()` 之前调用

### SyncTaskTracker — 后台任务追踪

`close()` 必须等待所有进行中的后台任务完成后才能关闭数据库。`SyncTaskTracker` 追踪所有进行中的任务，`drain(timeoutMs)` 带超时等待全部 resolve，返回 `{ completed: boolean; remainingTasks: string[] }`。

```typescript
// ✅ 后台任务通过 tracker 注册
const tracker = new SyncTaskTracker();

// fire-and-forget 任务也被追踪
const task = syncToServer(data);
tracker.track(task, 'session-sync');

// close() 等待所有任务完成（带超时保护）
async close() {
  guard.close();                       // 1. 拒绝新的 DB 操作
  const { completed, remainingTasks } = await tracker.drain(5000);  // 2. 等待，最多 5s
  if (!completed) {
    log.warn(`close: ${remainingTasks.length} tasks still running:`, remainingTasks);
  }
  await db.close();                    // 3. 安全关闭数据库
}

// drain 进入 closing 状态后拒绝新任务（防止关闭期间启动新操作）
// track() 在 closing 时记录警告并跳过

// ❌ 不追踪，close() 时后台任务仍在写入已关闭的数据库
// ❌ 无超时，drain 可能永远等待（如卡住的网络请求）
```

**约束**：

- `drain()` 必须带超时参数（默认 5000ms），防止单个卡住的任务阻塞整个关闭流程
- `drain()` 返回 `completed: false` 时不抛异常——由调用方决定是否记录警告或强制关闭
- `track()` 在 `drain()` 调用后拒绝新任务，记录 warn 日志（而非静默忽略）

### TrackedBackgroundTask — 生命周期感知的后台任务

fire-and-forget 的后台任务必须使用 `TrackedBackgroundTask`，它组合了 `SyncTaskTracker`（追踪）+ `LifecycleGuard`（生命周期守卫）+ `AbortController`（可取消延迟）。

```typescript
// ✅ 生命周期感知的后台任务
const task = new TrackedBackgroundTask(tracker, guard, closeSignal);

task.run('upload-sync', async (signal) => {
  await uploadToServer(file);
  await repo.transitionSyncStatus(md5, ext, 'uploading', 'synced');
}, {
  retries: 3,
  baseDelay: 1000,
  exponentialBackoff: true,
});

// 关闭时：
// 1. LifecycleGuard 阻止后续重试
// 2. AbortSignal 取消等待中的 delay（不用等完整退避时间）
// 3. SyncTaskTracker 等待当前 attempt 完成后才关闭 DB

// ❌ 裸 setTimeout + 无追踪
setTimeout(async () => {
  await retryUpload();  // close() 后仍在运行，写入已关闭的 DB
}, 5000);
```

**约束**：

- 所有 fire-and-forget 异步操作必须通过 `TrackedBackgroundTask.run()` 启动
- 重试 delay 必须可被 `AbortSignal` 取消（不能在 close 后还等完整退避时间）
- `closeSignal`（来自 `close()` 的 AbortController）必须连接到任务
- `closeSignal` 的 abort listener 必须在 `task.finally()` 中通过 `removeEventListener` 清理，防止长期存活的 closeSignal 累积 listener 导致内存泄漏
- `_cancellableDelay` 在 abort 时 **resolve**（非 reject），因此 delay 结束后必须检查 `signal.aborted` 再执行下一次重试，否则会多执行一次不期望的 action

### ResourceScope — 自动资源清理

当一个操作涉及多个 timer 和 event listener 时，使用 `ResourceScope` 统一管理生命周期。`dispose()` 清理所有受管资源，防止泄漏。

```typescript
// ✅ 受管资源
const scope = new ResourceScope();
try {
  const timer = scope.setTimeout(() => retry(), 1000);
  scope.addEventListener(signal, 'abort', () => cleanup());
  await operation();
} finally {
  scope.dispose();  // 清理所有 timer 和 listener
}

// ❌ 手动管理，容易遗漏
const timer = setTimeout(() => retry(), 1000);
signal.addEventListener('abort', handler);
// 忘记清理...
```

**约束**：

- `setTimeout()` 替代全局 `setTimeout()`（scope dispose 后调用会 **throw**，fail-fast 暴露 bug）
- `addEventListener()` 替代直接 `addEventListener()`（scope disposed 后调用只记录 **warn 并跳过**，不 throw -- 因为 listener 注册通常是异步回调中的延迟操作，throw 会中断正常流程）
- `{ once: true }` listener 触发后自动从追踪集合移除，防止集合无限增长
- 配合 `try/finally` 确保 `dispose()` 总是被调用

### FileOpCoordinator — 状态变更自动广播

当多个来源（background sync、resume upload、手动操作）可能修改同一实体的状态时，状态变更后需要通知所有监听者（如 WorkerCore 向主线程广播进度）。`FileOpCoordinator` 封装了"状态转移 + 自动通知"的原子组合，避免各来源各自实现广播逻辑导致遗漏。

```typescript
// ✅ 状态变更 + 自动广播
const coordinator = new FileOpCoordinator();

// 注册监听者（返回 dispose 函数）
const dispose = coordinator.onStatusChange((md5, ext, status, errorMessage?) => {
  workerCore.notifyFileStatusChange(md5, ext, status, errorMessage);
});

// 状态变更 + 自动通知（封装 transitionSyncStatus + notify）
await coordinator.transitionAndNotify(
  repo, md5, ext,
  ['pending', 'failed'], 'syncing',
);
// 内部：1. transitionSyncStatus() 原子转移
//       2. 成功后自动调用所有 onStatusChange 回调

// ❌ 各来源各自实现广播，容易遗漏
async function backgroundSync(file) {
  await repo.transitionSyncStatus(md5, ext, 'pending', 'syncing');
  // 忘记通知监听者 → WorkerCore 不知道状态变化
}
```

**约束**：

- 所有需要广播状态变更的场景必须通过 `coordinator.transitionAndNotify()` 而非直接调用 `repo.transitionSyncStatus()`
- `onStatusChange()` 返回的 dispose 函数必须在组件/模块销毁时调用，防止泄漏
- `clearAllListeners()` 仅在持久化层 `close()` 时调用

**防止的问题**：background sync 和 resumeInterruptedUploads 的状态变更缺少 per-file 事件广播，导致 UI 层无法感知文件状态变化。

### In-Flight 操作去重

当多个调用方可能同时请求同一操作（下载同一文件、刷新同一 token、同步同一 session）时，使用 `Map<key, Promise<T>>` 追踪进行中的操作，后续调用方共享同一 Promise 而非发起重复操作。

```typescript
// ✅ In-flight 去重：多个调用方共享同一 Promise
private _inflight = new Map<string, Promise<Blob>>();

async download(key: string): Promise<Blob> {
  const existing = this._inflight.get(key);
  if (existing) return existing;  // 复用进行中的操作

  const promise = this._doDownload(key).finally(() => {
    this._inflight.delete(key);  // 完成后清理，防止内存泄漏
  });
  this._inflight.set(key, promise);
  return promise;
}

// ❌ 不去重：两个组件同时请求同一文件，触发两次下载
async download(key: string): Promise<Blob> {
  return this._doDownload(key);  // 每次调用都发起新请求
}
```

**防止的问题**：

- 多个组件同时渲染同一文件的缩略图，触发多次 S3 下载（浪费带宽、增加延迟）
- Auth token 刷新期间多个 API 调用同时触发刷新请求
- 同一 session 的同步操作被多个事件并发触发

**约束**：

- `Map<key, Promise<T>>` 的 key 必须能唯一标识操作（如 `${md5}.${ext}` 表示文件下载）
- 必须在 `.finally()` 中清理条目——无论成功或失败，防止 map 无限增长
- 调用方无需感知去重逻辑——接口与普通异步方法相同
- 与 `LifecycleGuard` 配合：lifecycle 关闭时清理所有 in-flight 条目

**适用场景**：

- 文件下载/上传（同一文件不被重复传输）
- Token 刷新（同一时刻只有一个刷新请求）
- 数据同步（同一实体不被并发同步）
- 缩略图生成（同一文件只生成一次缩略图）

---

## 总结

IndexedDB 是产品体验的基石。选对策略，数据流畅；选错策略，处处是坑。

| 策略 | 何时用 | 风险 |
|------|--------|------|
| 离线优先 | 用户输入不能丢 | 同步冲突需处理 |
| 服务端优先 | 数据以服务端为准 | 弱网体验差 |
| 仅缓存 | 可重建的加速层 | 缓存失效需处理 |
| 不持久化 | 临时 UI 状态 | 刷新丢失 |

> _"持久化不是免费的。每一条写入都有代价——选择你真正需要的。"_
