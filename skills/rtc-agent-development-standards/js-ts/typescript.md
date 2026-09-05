# TypeScript 规范

> _类型是代码的第一道文档。好的类型定义让调用方不需要看实现就能正确使用，让重构不再提心吊胆。_

---

## 1. 严格模式

### 必须开启

`tsconfig.json` 中 `strict: true` 必须开启，不允许关闭任何严格检查项。

```json
{
  "compilerOptions": {
    "strict": true,
    "noImplicitAny": true,
    "strictNullChecks": true,
    "strictFunctionTypes": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true
  }
}
```

### 不允许逃逸

- 不允许用 `// @ts-ignore` 压制类型错误
- 不允许用 `as any` 绕过类型检查
- 遇到第三方库无类型时，先找 `@types/xxx`，其次写 `.d.ts` 声明

---

## 2. 接口 vs type

### 对象结构用 `interface`

```typescript
// ✅ interface 描述对象结构
interface ChatMessage {
  id: string
  content: string
  role: 'user' | 'assistant'
  timestamp: number
}

interface ChatPanelProps {
  sessionId: string
  onMessageSent?: (msg: ChatMessage) => void
}
```

### 联合、交叉、工具类型用 `type`

```typescript
// ✅ type 描述类型组合
type MessageRole = 'user' | 'assistant' | 'system'

type ReadonlyMessage = Readonly<ChatMessage>

type ApiResponse<T> = { data: T } | { error: string }
```

### 判断标准

> 如果你在描述"一个东西长什么样"，用 `interface`。
> 如果你在描述"类型之间如何组合"，用 `type`。

---

## 3. 禁止 `any`

### 新增代码禁止使用 `any`

```typescript
// ❌ any
function process(data: any): any { ... }

// ✅ unknown + 类型守卫
function process(data: unknown): Result {
  if (isChatMessage(data)) {
    // data 在这里收窄为 ChatMessage
    return handleChatMessage(data)
  }
  throw new Error('unsupported data type')
}

function isChatMessage(value: unknown): value is ChatMessage {
  return (
    typeof value === 'object' &&
    value !== null &&
    'id' in value &&
    'content' in value
  )
}
```

### 已有代码中的 `any`

- 不主动清理（避免引入破坏性变更）
- 修改相关代码时，顺手收窄类型
- PR review 时，新增的 `any` 必须要求修改

### `unknown` 而非 `any`

当类型不确定时，用 `unknown` 替代 `any`。`unknown` 强制你在使用前做类型检查。

```typescript
// ✅ unknown 强制类型检查
function parseResponse(raw: unknown): ChatMessage {
  if (!isChatMessage(raw)) {
    throw new TypeError('invalid response')
  }
  return raw  // 这里已经是 ChatMessage
}
```

---

## 4. 类型导出

### 组件的公开类型必须导出

消费者不应重复定义已有类型。

```typescript
// ✅ 导出所有公开类型
export interface ChatMessage { ... }
export interface ChatPanelProps { ... }
export type MessageRole = 'user' | 'assistant' | 'system'

export class ChatPanel extends HTMLElement { ... }

// ❌ 只导出类，消费者需要自己定义 Props
export class ChatPanel extends HTMLElement { ... }
// 消费者不得不：
// interface MyChatMessage { id: string; content: string; ... }
```

### 统一从包入口导出

```typescript
// packages/chat-panel/src/index.ts
export { ChatPanel } from './ChatPanel'
export type { ChatPanelProps, ChatMessage } from './types'
```

---

## 5. 泛型使用

### 组件和工具函数支持泛型

```typescript
// ✅ 泛型列表组件
class ListPanel<T> extends HTMLElement {
  private items: T[] = []
  private renderItem: (item: T) => HTMLElement

  constructor(renderItem: (item: T) => HTMLElement) {
    super()
    this.renderItem = renderItem
  }

  setItems(items: T[]): void {
    this.items = items
    this.render()
  }
}

// ✅ 泛型工具函数
functiongroupBy<T, K extends string | number>(
  items: T[],
  keyFn: (item: T) => K,
): Record<K, T[]> {
  return items.reduce((acc, item) => {
    const key = keyFn(item)
    ;(acc[key] ??= []).push(item)
    return acc
  }, {} as Record<K, T[]>)
}
```

### 泛型命名

- 单个泛型：`T`
- 有语义的泛型：`TItem`、`TResult`、`TData`
- 多个泛型：`T`、`U` 或 `TItem`、`TKey`

---

## 6. 枚举替代

### 推荐 `as const` + 联合类型

`enum` 在编译后生成额外代码，不利于 tree-shaking。推荐用 `as const` 替代。

```typescript
// ❌ enum
enum MessageStatus {
  Pending = 'pending',
  Sent = 'sent',
  Failed = 'failed',
}

// ✅ as const + 联合类型
const MessageStatus = {
  Pending: 'pending',
  Sent: 'sent',
  Failed: 'failed',
} as const

type MessageStatus = typeof MessageStatus[keyof typeof MessageStatus]
// 'pending' | 'sent' | 'failed'
```

### 何时可以用 `enum`

- 需要反向映射（数字枚举）
- 需要计算成员
- 团队共识允许

---

## 7. 类型收窄

### 善用类型守卫

```typescript
// ✅ is 类型守卫
function isChatMessage(value: unknown): value is ChatMessage {
  return (
    typeof value === 'object' &&
    value !== null &&
    'id' in value &&
    'content' in value &&
    'role' in value
  )
}

// 使用
const data: unknown = await response.json()
if (isChatMessage(data)) {
  // data 自动收窄为 ChatMessage
  console.log(data.content)
}
```

### 内置收窄工具

```typescript
// in 操作符
if ('sessionId' in message) {
  // message 有 sessionId 属性
}

// typeof
if (typeof value === 'string') {
  // value 是 string
}

// instanceof
if (error instanceof TypeError) {
  // error 是 TypeError
}
```

---

## 8. 新增代码约束

### 新增公开 API 必须有完整类型

```typescript
// ❌ 缺少返回类型
function formatMessage(msg: ChatMessage) {
  return msg.content.trim()
}

// ✅ 完整类型
function formatMessage(msg: ChatMessage): string {
  return msg.content.trim()
}
```

### 新增组件必须导出 Props 类型

```typescript
// ✅ 导出 Props
export interface WhiteboardProps {
  sessionId: string
  readOnly?: boolean
  onDraw?: (event: DrawEvent) => void
}

export class Whiteboard extends HTMLElement {
  // ...
}
```

### 新增函数必须有参数和返回类型

```typescript
// ❌ 缺少类型
function parse(data) { ... }

// ✅ 完整类型
function parse(data: string): ParsedResult { ... }
```

---

## 总结

类型是契约，不是负担。写好类型定义，编译器会成为你最可靠的 reviewer。

> _"如果你不能用类型描述它，你就没有真正理解它。"_
