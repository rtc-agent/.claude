# EntityRepository

实体 CRUD 层：事务性 upsert、字段级 diff、UIUpdateBus 集成。

**所属 package**: `persistence`

## 关键代码文件

- [entity-repository.ts](~/Workspaces/rtc-agent/web-components/packages/persistence/src/entity-repository.ts) — `EntityRepository` 类 + `initEntityRepository()` / `getEntityRepository()` 单例管理

## 职责

| 职责 | 方法示例 | 说明 |
| ------ | ---------- | ------ |
| 实体 Upsert | `upsertSession()` / `upsertMessage()` / `upsertRtc()` / `upsertTurn()` | 事务性 read-modify-write |
| 批量 Upsert | `applyUpdates()` / `applyUpdates()` | Gap fill 后批量写入 |
| 查询 | `getClientSession()` / `listMessagesBySession()` / `listRtcBySession()` | 按 client_id / server_id 查询 |
| 软删除 | `softDeleteSession()` | 设置 deleted_at，不物理删除 |
| UI 事件发出 | `emitUIUpdates()` | microdiff 计算字段级差异 |
| 设备过滤 | `getNextRtcToProcess()` | 仅返回当前 deviceId 的 RTC |

## Upsert 流程

```mermaid
flowchart TD
    A["upsertSession(partial, syncStatus, options)"] --> B["db.transaction('rw', sessions)"]
    B --> C{"session.client_id exists in DB?"}
    C -->|Yes| D["UPDATE: merge fields, preserve server_id"]
    C -->|No| E["CREATE: fill defaults, set timestamps"]
    D --> F{"options.silent?"}
    E --> F
    F -->|No| G["emitUIUpdates()"]
    F -->|Yes| H["Skip event emission"]
    G --> I["Return UpsertResult {before, after}"]
    H --> I
```

## UpsertResult 与字段级 Diff

```mermaid
sequenceDiagram
    participant ER as EntityRepository
    participant DB as IndexedDB
    participant Diff as microdiff
    participant Bus as UIUpdateBus

    ER->>DB: get(client_id) → existing
    ER->>ER: before = {...existing}
    ER->>ER: after = {...existing, ...changes}
    ER->>DB: put(after)
    ER->>Diff: diff(before, after)
    Diff-->>ER: [{path, oldValue, value}, ...]

    loop For each change
        ER->>Bus: publish({entity, action, entityId, field, oldValue, newValue})
    end
```

`emitUIUpdates()` 使用 `microdiff` 计算字段级差异，只发出变化的字段事件。例如 `title` 变更只发出 `{field: 'title', oldValue: 'old', newValue: 'new'}`，而不是整个 session 对象。

## 字段路径格式

```typescript
// microdiff path: ['content', 0, 'text']
// → field: 'content.0.text'
```

数组索引作为路径段，UI 层可精确定位到嵌套字段的变更。

## Streaming Status 状态保护

```mermaid
stateDiagram-v2
    [*] --> pending : Initial
    pending --> streaming : Live channel update
    streaming --> completed : Topic channel finalization
    streaming --> failed : Error

    completed --> completed : Idempotent (same-state OK)
    streaming --> streaming : Newer streaming content (same-state OK)

    note right of completed
        Terminal state — must NOT regress
        to pending or streaming
    end note
```

`isStreamingStatusRegression()` 防止 live channel 的 streaming 更新覆盖 topic channel 已到达的 completed 状态。

```typescript
// Terminal states must not regress to non-terminal states
if (existingStatus === 'completed' || existingStatus === 'failed') {
    return incomingStatus === 'streaming' || incomingStatus === 'pending';
}
```

**应用范围**：该检查同时应用于单条 upsert（`upsertMessage()`）和批量 merge（`applyUpdates()` → `mergeMessages()`）。批量操作中每条 message 独立检查，确保不会因为批量写入导致已 terminal 的消息回退。

## 事务保证

所有 upsert 操作使用 Dexie 事务：

```typescript
return db.transaction('rw', db.sessions, async () => {
    // Read-modify-write inside transaction
    let existing = await db.sessions.where('client_id').equals(clientId).first();
    // ...
    await db.sessions.put(updated);
});
```

Dexie 支持事务嵌套：如果已在事务中，复用父事务。

## 批量操作事务 (applyUpdates)

`applyUpdates()` 将多个 Update 事件合并为单个 Dexie 事务，预期减少 50-400 倍 DB 操作：

```mermaid
flowchart TD
    A["applyUpdates(updates[])"] --> B{"updates.length === 1?"}
    B -->|Yes| C["applyUpdateOriginal()<br/>(per-item path)"]
    B -->|No| D["Group items by entity"]
    D --> E["Load required session mappings"]
    E --> F["Prepare entity data<br/>(field mapping + defaults)"]
    F --> G["Batch resolve parent_message_id"]
    G --> H["Resolve delete client_ids"]
    H --> I["Determine tables to lock<br/>(only tables with ops)"]
    I --> J["db.transaction('rw', tablesToLock)"]

    J --> K["bulkGet existing records"]
    K --> L["Collect before snapshots"]
    L --> M["Merge in memory:<br/>{...existing, ...newData}"]
    M --> N["bulkPut / bulkDelete"]
    N --> O["Turn count writeback<br/>(per-session sub-transaction)"]
    O --> O1["db.transaction('rw', sessions, turns)"]
    O1 --> O2["countActiveTurns(sessionId)"]
    O2 --> O3["get(session) → check delta"]
    O3 --> O4["update(sessionId, counts)<br/>if changed"]
    O4 --> P["Transaction commit"]

    P --> Q["Emit batch UI updates<br/>(outside transaction)"]
```

关键设计：

- **动态表锁定** — 仅锁定有实际操作的表，减少事务冲突
- **事务内 bulkGet** — 确保读到一致的现有记录
- **Turn 计数写回（per-session 事务）** — 每个 session 独立事务：事务内重新 countActiveTurns → 读取当前 session → update。防止并发 writebackTurnCountsInTx 调用交错导致丢失更新（Fix: 隔离 turn count 更新）。Dexie 事务嵌套支持：如果已在父事务中，复用父事务。
- **UI 更新在事务外** — 避免在事务内执行 UI 回调导致阻塞

## UpsertOptions

| Option | Type | Description |
| -------- | ------ | ------------- |
| `silent` | `boolean` | 不发出 UIUpdateBus 事件 |
| `preserveSyncStatus` | `boolean` | 不修改 sync_status（用于聚合更新如 turn 计数） |

### preserveSyncStatus 用途

写入时聚合（write-time aggregation）场景：当 turn 数据 upsert 后，需要将 `pending_turn_count` / `running_turn_count` 回写到 session 行。此时不应覆盖 session 的 `sync_status`（可能已被其他更新设为最新值）。

```mermaid
sequenceDiagram
    participant ER as EntityRepository
    participant DB as IndexedDB

    Note over ER: Turn upserted
    ER->>ER: countActiveTurns(sessionClientId)
    ER->>ER: upsertSession({pending_turn_count, running_turn_count},<br/>'synced', {silent: true, preserveSyncStatus: true})
    Note over ER: sync_status 保持原值不被覆盖
    ER->>DB: put(updated session)
```

## Structured Clone 安全

`emitUIUpdates()` 使用 `safeClone()` 确保数据可通过 Comlink postMessage 传输：

```typescript
function safeClone<T>(obj: T): T {
    return JSON.parse(JSON.stringify(obj));
}
```

JSON 序列化剥离不可 clone 的对象（函数、DOM 元素、循环引用）。

## 单例管理

```typescript
let entityRepositoryInstance: EntityRepository | null = null;

export function initEntityRepository(deviceId: string): EntityRepository {
    entityRepositoryInstance = new EntityRepository(deviceId);
    return entityRepositoryInstance;
}

export function getEntityRepository(): EntityRepository {
    if (!entityRepositoryInstance) {
        throw new Error('EntityRepository not initialized');
    }
    return entityRepositoryInstance;
}
```

`initEntityRepository()` 在 `PersistenceLayer` 构造时调用。

## Turn 计数写回 (Write-Time Aggregation)

当 Turn 实体被 upsert 后，需要将 `pending_turn_count` / `running_turn_count` 回写到对应的 Session 行。这避免了在 Session 表中手动维护计数器，保证计数与 Turn 表一致。

```mermaid
flowchart TD
    A["applyUpdates() batch"] --> B["bulkPut turns"]
    B --> C["writebackTurnCountsInTx(turns)"]
    C --> D["Collect affected session_client_ids"]
    D --> E["Per-session sub-transaction"]
    E --> F["countActiveTurns(sessionId)"]
    F --> G["Re-read session from DB"]
    G --> H{"counts changed?"}
    H -->|Yes| I["session.update(pending_turn_count, running_turn_count)"]
    H -->|No| J["Skip (reduce unnecessary writes)"]
```

### countActiveTurns

[entity-repository.ts:404-413](~/Workspaces/rtc-agent/web-components/packages/persistence/src/entity-repository.ts#L404-L413)

按 `session_client_id` 统计 pending 和 running 状态的 turn 数量。由于 Dexie 不支持同一 query chain 的多个 `where()` 调用，使用 `Promise.all()` 并行查询两种状态：

```typescript
const [pending, running] = await Promise.all([
    base.filter(t => t.status === 'pending').count(),
    db.turns.where('session_client_id').equals(sessionId)
            .filter(t => t.status === 'running').count(),
]);
```

### writebackTurnCountsInTx 隔离保证

[entity-repository.ts:1434-1476](~/Workspaces/rtc-agent/web-components/packages/persistence/src/entity-repository.ts#L1434-L1476)

每个 session 的 read-update 包裹在独立事务中，防止并发 `writebackTurnCountsInTx` 调用在同一 session 上交錯导致丢失更新：

```mermaid
sequenceDiagram
    participant T1 as Sub-tx Session A
    participant T2 as Sub-tx Session B
    participant DB as IndexedDB

    Note over T1,T2: Parallel execution for different sessions
    T1->>DB: countActiveTurns(A)
    T2->>DB: countActiveTurns(B)
    DB-->>T1: {pending: 2, running: 1}
    DB-->>T2: {pending: 0, running: 3}
    T1->>DB: session.get(A)
    T2->>DB: session.get(B)
    T1->>DB: session.update(A, {counts})
    T2->>DB: session.update(B, {counts})
```

### preserveSyncStatus 选项

回写 turn 计数时使用 `preserveSyncStatus: true`，防止覆盖 session 的 `sync_status`（可能已被其他更新设为最新值）：

```typescript
await this.upsertSession({
    pending_turn_count: pending,
    running_turn_count: running,
}, 'synced', { silent: true, preserveSyncStatus: true });
```

## 查询方法与默认行为

### listSessions 分页

```typescript
async listSessions(cursor?: string, limit: number = 10000): Promise<LocalSession[]>
```

**默认行为变更**（Fix #120）：默认 `limit` 从 50 改为 10000（实际无限制）。之前的 50 条限制导致用户超过 50 个 session 时，部分 session 在 UI 中"消失"。

**性能警告**：当活跃 session 超过 1000 时，日志输出警告，提示考虑在 UI 层实现虚拟滚动。

**分页 API**：保留 `cursor` 参数用于未来分布式数据库场景的游标分页。cursor 是上一页最后一条记录的 `client_id`。

```mermaid
flowchart TD
    A["listSessions(cursor?, limit?)"] --> B["orderBy('updated_at').reverse()"]
    B --> C["Filter: !deleted_at"]
    C --> D{"cursor provided?"}
    D -->|Yes| E["Find cursor index"]
    E --> F{"Found?"}
    F -->|Yes| G["slice(startIdx + 1, startIdx + 1 + limit)"]
    F -->|No| H["Warn + slice from start"]
    D -->|No| I["slice(0, limit)"]
```

## 关键注释摘录

> **事务性 Upsert** — [entity-repository.ts:201-203](~/Workspaces/rtc-agent/web-components/packages/persistence/src/entity-repository.ts#L201-L203)
> _Wrap read-modify-write in transaction for atomicity. Dexie supports transaction nesting: if already inside a transaction, reuses parent._

<!-- separator -->

> **Streaming Status 保护** — [entity-repository.ts:89-102](~/Workspaces/rtc-agent/web-components/packages/persistence/src/entity-repository.ts#L89-L102)
> _Once a message reaches a terminal state (completed / failed), it must not revert to an earlier state (streaming / pending). This prevents live-channel streaming updates from overwriting the final completed content received via the topic channel._

<!-- separator -->

> **listSessions 默认限制** — [entity-repository.ts:271-312](~/Workspaces/rtc-agent/web-components/packages/persistence/src/entity-repository.ts#L271-L312)
> _Default limit changed from 50 to 10000 (effectively unlimited). The previous 50-session limit caused sessions to "disappear" from UI for users with more than 50 sessions. Performance warning logged when active sessions exceed 1000._

## 跨维度关联

- [[Database]] — 操作的底层 IndexedDB 表
- [[UIUpdateBus]] — 每字段差异事件发出到 UIUpdateBus
- [[PersistenceLayer]] — EntityRepository 的持有者
- [[SyncStatus]] — upsert 管理 sync_status 状态
- [[Publication]] — applyUpdates 处理 Centrifuge publication
- [[ProtocolTypes]] — `Session` / `Message` / `Turn` / `Rtc` 等域模型类型定义
- [[MessageRepository]] — MessageController 从 EntityRepository 读取数据后在 MessageRepository 中管理 UI 状态
- [[ConcurrencyPatterns]] — 事务保护 upsert + per-session 子事务隔离
- [[StreamingEvents]] — `isStreamingStatusRegression()` 保护 streaming_status 单调性
