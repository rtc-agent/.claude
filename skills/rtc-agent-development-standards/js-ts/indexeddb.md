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

### 仅离线优先的数据需要 `sync_status`

```typescript
// ✅ 离线优先：需要 sync_status
interface LocalMessage {
  client_id: string
  content: string
  sync_status: 'pending' | 'synced' | 'failed'
}

// ✅ 服务端优先 / 仅缓存：不需要 sync_status
interface CachedSession {
  client_id: string
  server_id: string
  title: string
  // 无 sync_status
}
```

### 同步失败处理

- `failed` 状态的记录必须有重试机制
- 重试策略：指数退避，最大重试次数有限
- 超过重试次数后通知用户，不静默丢弃

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
