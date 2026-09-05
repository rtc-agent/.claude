# API 文档规范

> _接口是模块之间的契约。契约必须清晰、精确、无歧义——因为它是双方唯一共同信赖的真相。_

---

## 1. 文件组织

### 目录结构

`docs/` 仓库包含两类内容：**协议定义**（`protocol/`）和**项目使用文档**（GitHub Pages 站点）。本规范仅约束 `protocol/` 下的接口定义。

```
docs/
├── protocol/                      # 接口与消息类型定义（本规范范围）
│   ├── openapi.yaml               # 入口索引（如适用）
│   ├── common/
│   │   ├── types.yaml             # 通用类型定义
│   │   ├── error.yaml             # 统一错误结构
│   │   └── pagination.yaml        # 分页相关类型
│   ├── rooms/
│   │   ├── messages.yaml          # 房间消息相关 Request/Response
│   │   ├── members.yaml           # 房间成员管理
│   │   └── events.yaml            # 房间事件定义
│   ├── sessions/
│   │   ├── session.yaml           # 会话管理
│   │   └── transcript.yaml        # 会话记录
│   └── auth/
│       └── auth.yaml              # 认证相关
├── guide/                         # 项目使用文档（GitHub Pages 站点）
│   └── ...                        # 由三方静态站点工具生成
└── package.json                   # 站点构建配置
```

### 组织原则

- **按领域拆分**：一个文件对应一个业务领域，不堆积所有定义
- **通用类型集中**：跨领域复用的类型放在 `common/`
- **入口文件索引**：根目录文件引用各子文件，形成完整视图
- **文件名 kebab-case**：小写单词，连字符分隔

---

## 2. Request / Response 命名

### 命名约定

| 方向 | 命名格式 | 示例 |
|------|---------|------|
| 客户端 → 服务端 | `<Action>Request` | `CreateRoomRequest` |
| 服务端 → 客户端 | `<Action>Response` | `CreateRoomResponse` |
| 事件推送（服务端 → 客户端） | `<Event>Event` | `MessageReceivedEvent` |
| 流式消息 | `<Resource>Stream` | `RoomEventStream` |

### 配对规则

- 每个 Request 必须有对应的 Response，即使 Response 只有 `success: bool`
- Request 和 Response 放在同一文件中，紧邻定义
- 如果 Response 会被多个 Request 共享，提取到公共区域并注释引用来源

### 方向标识

```yaml
# 明确标注通信方向
CreateRoomRequest:    # client → server
  description: 创建新房间

CreateRoomResponse:   # server → client
  description: 创建房间的结果
```

---

## 3. 字段定义规范

### 字段命名

选定一种命名风格后贯穿始终。推荐：

- **JSON 传输**：`camelCase`（`userId`, `roomId`）
- **协议定义文件内部**：与目标语言保持一致

### 类型选择

| 场景 | 推荐类型 | 说明 |
|------|---------|------|
| 唯一标识 | `string` | UUID 或业务 ID，不用数字 |
| 时间戳 | `string` (ISO 8601) | `"2026-09-05T10:30:00Z"` |
| 时间间隔 | `integer` + 单位注释 | 毫秒或秒，必须注释说明 |
| 金额 | `string` | 避免浮点精度问题 |
| 布尔标志 | `boolean` | 命名带 `is/has/can` 前缀 |
| 列表 | `array` | 空列表用 `[]`，不用 `null` |

### 必填与可选

```yaml
CreateRoomRequest:
  required:
    - name           # 必填：房间名称
    - type           # 必填：房间类型
  properties:
    name:
      type: string
      minLength: 1
      maxLength: 100
    type:
      type: string
      enum: [meeting, chat, review]
    description:
      type: string            # 可选：房间描述
      maxLength: 500
    maxMembers:
      type: integer           # 可选：最大成员数
      default: 50
      minimum: 2
      maximum: 1000
```

- **默认值必须显式声明**：不依赖隐式默认值
- **可选字段在注释中说明缺省行为**：`// 可选，默认 50`

---

## 4. 通用类型

### 复用原则

- **三次复用规则**：一个类型被三个以上地方使用时，提取到 `common/`
- **不要过早提取**：前两次出现时保留在各自领域，确认稳定后再提取
- **提取后注释来源**：说明哪些领域在使用这个类型

### 统一错误结构

```yaml
Error:
  description: 统一错误响应结构
  properties:
    code:
      type: string
      description: 机器可读的错误码，如 ROOM_NOT_FOUND
      example: ROOM_NOT_FOUND
    message:
      type: string
      description: 人类可读的错误描述
      example: 房间不存在或已被关闭
    details:
      type: object
      description: 可选的错误详情，用于提供额外上下文
      additionalProperties: true
```

### 统一分页结构

```yaml
PaginatedRequest:
  description: 分页请求的通用字段
  properties:
    pageSize:
      type: integer
      default: 20
      minimum: 1
      maximum: 100
    cursor:
      type: string
      description: 分页游标，首次请求不传，后续从响应中获取

PaginatedResponse:
  description: 分页响应的通用结构
  properties:
    items:
      type: array
      items: {}                 # 由具体接口指定
    nextCursor:
      type: string
      nullable: true
      description: 下一页游标，为 null 表示没有更多数据
    total:
      type: integer
      nullable: true
      description: 总数，部分场景可能不返回
```

---

## 5. 字段注释

每个字段必须有描述。没有注释的字段是契约中的漏洞。

### 注释内容标准

```yaml
User:
  properties:
    id:
      type: string
      description: |
        用户唯一标识，UUID v4 格式。
        由服务端在创建用户时生成，客户端不可修改。
      example: "550e8400-e29b-41d4-a716-446655440000"
    status:
      type: string
      description: |
        用户状态。
        - `active`: 正常活跃
        - `inactive`: 超过 30 天未登录
        - `suspended`: 被管理员暂停
      enum: [active, inactive, suspended]
    lastSeenAt:
      type: string
      format: date-time
      description: |
        最后活跃时间，ISO 8601 格式，UTC 时区。
        用户首次创建时为 null。
      nullable: true
```

### 注释应包含

- **是什么**：字段的语义定义
- **约束**：取值范围、格式要求、长度限制
- **由谁生成**：客户端填充还是服务端生成
- **边界行为**：null 的含义、默认值、特殊值
- **示例值**：帮助读者建立直觉

---

## 6. 枚举设计

### 命名约定

```yaml
RoomType:
  type: string
  description: 房间类型
  enum:
    - MEETING    # 会议房间
    - CHAT       # 即时聊天
    - REVIEW     # 代码审查

MemberRole:
  type: string
  description: 成员角色
  enum:
    - OWNER      # 房间创建者
    - ADMIN      # 管理员
    - MEMBER     # 普通成员
    - GUEST      # 访客
```

### 设计规则

- **值全大写 + 下划线分隔**（`UPPER_SNAKE_CASE`）
- **预留 UNKNOWN 值**：为未来扩展留余地

```yaml
RoomStatus:
  type: string
  enum:
    - UNKNOWN    # 未知状态，用于向前兼容
    - ACTIVE
    - ARCHIVED
    - DELETED
```

- **枚举值一旦发布不删除**：只能标记为废弃，不能直接移除
- **枚举值语义稳定**：不改变已有值的含义

---

## 7. 版本管理

### 兼容性规则

接口定义一旦发布，遵循**向后兼容**原则：

| 操作 | 是否允许 | 说明 |
|------|---------|------|
| 新增可选字段 | ✅ | 安全，旧客户端忽略即可 |
| 新增枚举值 | ✅ | 安全，旧客户端按 UNKNOWN 处理 |
| 废弃字段 | ✅ | 标记 deprecated，至少保留两个版本 |
| 删除字段 | ❌ | 破坏旧客户端 |
| 修改字段类型 | ❌ | 破坏解析 |
| 修改字段语义 | ❌ | 隐式破坏 |
| 将可选改为必填 | ❌ | 旧客户端不发送该字段 |

### 废弃字段处理

```yaml
User:
  properties:
    userName:
      type: string
      deprecated: true
      description: |
        已废弃。请使用 `displayName` 替代。
        将在 v3.0 移除。
    displayName:
      type: string
      description: 用户显示名称
```

### 破坏性变更

当必须做破坏性变更时：

1. 新增版本路径（如 `/v2/...`）或新增消息类型
2. 旧版本至少保留一个完整迭代周期
3. 在 CHANGELOG 和 ADR 中记录迁移指南

---

## 8. 反模式

### ❌ 过度嵌套

```yaml
# ❌ 三层嵌套，难以解析
Response:
  data:
    room:
      settings:
        notifications:
          enabled: true

# ✅ 扁平化
Response:
  data:
    roomId: "abc123"
    notificationsEnabled: true
```

### ❌ 巨型消息体

```yaml
# ❌ 一个 Request 承载过多职责
CreateAndConfigureRoomRequest:
  name: ...
  description: ...
  type: ...
  members: ...         # 同时邀请成员
  settings: ...        # 同时配置房间
  notifications: ...   # 同时设置通知

# ✅ 单一职责，分步完成
CreateRoomRequest:
  name: ...
  type: ...

InviteMembersRequest:
  roomId: ...
  members: ...

UpdateRoomSettingsRequest:
  roomId: ...
  settings: ...
```

### ❌ 缺少注释

```yaml
# ❌ 完全不知道这些字段什么意思
User:
  properties:
    a:
      type: string
    b:
      type: integer
    c:
      type: boolean

# ✅ 每个字段都有清晰描述
User:
  properties:
    id:
      type: string
      description: 用户唯一标识
    age:
      type: integer
      description: 用户年龄，仅对 18+ 用户返回
    isActive:
      type: boolean
      description: 用户当前是否在线
```

### ❌ 一个字段多重语义

```yaml
# ❌ value 可能是字符串、数字、对象——取决于 type
Message:
  properties:
    type:
      type: string
    value:
      type: object        # 语义不明

# ✅ 明确类型，或使用 oneOf
TextMessage:
  properties:
    content:
      type: string

ImageMessage:
  properties:
    imageUrl:
      type: string
    width:
      type: integer
    height:
      type: integer
```

---

## 总结

接口文档是系统各部分之间的契约。好的契约：

- **精确**：每个字段的类型、含义、约束都明确无误
- **完整**：覆盖所有端点、所有字段、所有错误场景
- **一致**：命名、风格、结构在整个项目中统一
- **演进**：支持向后兼容的扩展，不轻易破坏已有约定

> _"模糊的接口是 bug 的温床。精确的契约是稳定的基石。"_
