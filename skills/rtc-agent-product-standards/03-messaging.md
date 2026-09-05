# 消息与对话

## 概述

消息是用户与 AI 交互的载体。用户发送消息，AI 流式返回回复，支持 Markdown 渲染、代码高亮、思考过程展示、工具调用卡片。

## 消息类型

```mermaid
flowchart TD
    A[消息类型] --> B[用户消息]
    A --> C[AI 消息]
    A --> D[系统消息]

    C --> C1[文本回复<br/>Markdown 渲染]
    C --> C2[思考过程<br/>可折叠]
    C --> C3[工具调用<br/>卡片展示]
```

| 类型 | 说明 |
|------|------|
| 用户消息 | 用户输入的内容 |
| AI 回复 | AI 生成的文本，支持 Markdown |
| 思考过程 | AI 的推理过程，默认折叠，可展开查看 |
| 工具调用 | AI 请求执行工具，显示输入参数和输出结果 |
| 系统消息 | 系统生成的通知消息 |

## 消息发送流程

```mermaid
flowchart TD
    A[用户输入消息] --> B[本地写入 IndexedDB]
    B --> C[立即显示在界面]
    B --> D[后台同步到服务端]
    D --> E{成功?}
    E -->|是| F[标记 synced]
    E -->|否| G[标记 failed<br/>显示重试按钮]
    F --> H[服务端创建消息记录]
    H --> I[异步创建 Turn]
    I --> J[AI 开始处理]
    J --> K[流式返回结果]
```

消息先存本地立即显示，后台异步同步，保证响应速度。

## 发送失败交互

```mermaid
flowchart TD
    A[消息发送失败] --> B[消息显示失败状态]
    B --> C[显示重试按钮]
    C --> D{用户操作}
    D -->|点击重试| E[使用相同 client_id 重新发送]
    D -->|不操作| F[保持失败状态]
    E --> G{成功?}
    G -->|是| H[标记 synced<br/>移除重试按钮]
    G -->|否| I[保持 failed 状态]
```

| 状态 | 用户看到的行为 |
|------|---------------|
| pending | 消息显示发送中状态 |
| synced | 消息正常显示 |
| failed | 消息显示失败标记 + 重试按钮 |

重试使用相同的 client_id，服务端幂等去重，不会产生重复消息。

## Turn 状态

```mermaid
stateDiagram-v2
    [*] --> Pending: 创建 Turn
    Pending --> Running: 开始执行
    Running --> Completed: 正常完成
    Running --> Failed: 执行失败
    Running --> Interrupted: 等待人工确认
    Running --> Cancelled: 用户取消
    Interrupted --> Running: 提交确认后恢复
    Completed --> [*]
    Failed --> [*]
    Cancelled --> [*]
```

| 状态 | 含义 |
|------|------|
| Pending | 等待执行 |
| Running | 正在执行 |
| Completed | 正常完成 |
| Failed | 执行失败 |
| Interrupted | 被中断（等待工具执行结果） |
| Cancelled | 用户取消 |

## 流式消息

```mermaid
sequenceDiagram
    actor User as 用户
    participant UI as 前端界面
    participant Live as Live 频道
    participant Server as 服务端
    participant LLM as AI 模型

    User->>UI: 发送消息
    UI->>Server: 同步消息
    Server->>LLM: 请求 AI 回复
    LLM-->>Server: 流式返回 chunk
    Server-->>Live: 推送 stream:chunk
    Live-->>UI: 实时显示增量
    LLM-->>Server: 流结束
    Server-->>UI: 推送 stream:end
    UI->>UI: 标记完成
```

流式消息实时显示 AI 生成过程，无需等待完整回复。

## 停止 Turn

用户可随时停止正在执行的 Turn：
- 点击停止按钮
- 服务端取消队列中的任务
- 正在执行的 Turn 标记为 Cancelled

## AI Thinking 展示

```mermaid
flowchart LR
    A[思考过程] --> B{展开/折叠}
    B -->|折叠| C[显示 思考过程 标签]
    B -->|展开| D[显示完整思考内容]
    D --> E[Markdown 渲染]
```

- 默认折叠，节省空间
- 点击展开查看 AI 推理过程
- 流式生成时有脉冲动画

## 工具调用卡片

```mermaid
flowchart TD
    A[工具调用] --> B[toolcall_input]
    A --> C[toolcall_output]

    B --> B1[显示工具名称]
    B --> B2[显示输入参数<br/>JSON 格式化]

    C --> C1[显示输出结果<br/>JSON 格式化]
    C --> C2[复制按钮]
```

- 输入参数和输出结果分区展示
- JSON 自动格式化
- 状态指示：运行中（橙色脉冲）/ 完成（绿色）
- 支持复制输入/输出内容

## 消息列表

| 特性 | 说明 |
|------|------|
| 排序 | 按发送时间升序 |
| 分页 | 游标分页，从服务端加载历史消息 |
| 自动滚动 | 新消息到达时自动滚动到底部 |
| 离开底部提示 | 用户滚动离开底部时显示"新消息"按钮 |

## 消息重试

| 场景 | 处理方式 |
|------|---------|
| 发送失败 | 标记 failed，显示重试按钮 |
| 幂等保证 | 使用 client_id 去重，重试不会产生重复消息 |
| RTC 失败 | 指数退避重试（1s, 2s, 4s...30s 封顶） |
