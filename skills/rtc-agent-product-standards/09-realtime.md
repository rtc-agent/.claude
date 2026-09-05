# 实时通信

## 概述

基于 Centrifuge WebSocket 的实时通信系统，采用双频道架构：Topic 频道保证可靠性，Live 频道提供低延迟。

## 双频道架构

```mermaid
flowchart TD
    A[实时通信] --> B[Topic 频道<br/>topic:u=userID]
    A --> C[Live 频道<br/>live:u=userID]

    B --> B1[持久化<br/>写入数据库]
    B --> B2[Offset 追踪<br/>保证连续性]
    B --> B3[离线恢复<br/>支持历史补全]

    C --> C1[非持久化<br/>Redis PUB/SUB]
    C --> C2[即发即弃<br/>可接受丢失]
    C --> C3[低延迟<br/>实时推送]
```

| 频道 | 用途 | 特性 |
|------|------|------|
| Topic | 状态变更事件 | 持久化、Offset 追踪、离线恢复 |
| Live | 流式消息中间 chunks | 非持久化、低延迟、可丢失 |

## 事件分布

| 事件类型 | Topic 频道 | Live 频道 |
|---------|-----------|-----------|
| session.created/updated | ✅ | ❌ |
| turn.created/updated | ✅ | ❌ |
| message.created | ✅ | ❌ |
| message.updated（流完成） | ✅ | ❌ |
| message.updated（流中间 chunk） | ❌ | ✅ |
| rtc.created/updated | ✅ | ❌ |

## Offset 机制

```mermaid
flowchart TD
    A[服务端发布事件] --> B[原子分配 Offset]
    B --> C[写入数据库]
    C --> D[推送到频道]

    E[客户端接收] --> F{检测 Offset 连续性}
    F -->|连续| G[正常处理]
    F -->|跳跃| H[拉取缺失历史]
    H --> I[补全后继续]
```

- 每个 Topic 事件分配严格递增的 Offset
- 客户端检测 Offset 跳跃，自动补全历史
- 保证消息不丢失、不重复

## 流式消息流程

```mermaid
sequenceDiagram
    participant LLM as AI 模型
    participant Server as 服务端
    participant Redis as Redis
    participant Live as Live 频道
    participant Topic as Topic 频道
    participant UI as 前端

    LLM->>Server: 流式 chunk
    Server->>Redis: 缓冲 chunk
    Server->>Live: 推送中间 chunk
    Live->>UI: 实时显示

    LLM->>Server: 流结束
    Server->>Redis: 读取所有 chunks
    Server->>Server: 拼接完整内容
    Server->>Topic: 推送完整消息
    Topic->>UI: 更新最终状态
```

1. 第一个 chunk：创建消息记录，推送到 Topic
2. 中间 chunks：追加到 Redis，推送到 Live（实时显示）
3. 最后一个 chunk：拼接完整内容，更新数据库，推送到 Topic

## 离线恢复

```mermaid
flowchart TD
    A[客户端重连] --> B{检测 Offset}
    B -->|连续| C[直接处理]
    B -->|跳跃| D[拉取历史]
    D --> E[补全缺失消息]
    E --> C

    F{Epoch 变更?}
    F -->|是| G[清空本地 Offset]
    G --> H[从最新位置开始]
```

| 场景 | 行为 |
|------|------|
| 短暂断网 | 自动推送离线消息 |
| 长时间断网 | 检测 Offset 跳跃，拉取历史 |
| 历史被清理 | Epoch 变更，从最新位置开始 |

## 重连行为

- Centrifuge SDK 自动重连
- Token 失效时尝试刷新，失败则需重新登录
- Offset 持久化到 IndexedDB，页面刷新后恢复
