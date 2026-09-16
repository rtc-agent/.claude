# Eino TurnLoop 官方文档与示例

> Eino ADK 中 Agent 取消与 TurnLoop 多轮循环特性的完整指南。自 [v0.9.0-alpha.9](https://github.com/cloudwego/eino/releases/tag/v0.9.0-alpha.9) 版本引入。

## 文档索引

| 文档 | 说明 |
|------|------|
| [生命周期教程](./tutorial-lifecycle.md) | TurnLoop 核心概念：抢占（Preempt）、中止（Abort）、多轮生命周期管理 |
| [API 快速入门](./api-quickstart.md) | Agent 取消 API、TurnLoop 配置、取消模式、安全点机制详解 |
| [服务端示例](./server-example.go) | 完整服务端实现代码（基于 Hertz + SSE），展示 TurnLoop 在 Web 场景的集成 |

## 核心能力

### 1. Agent 取消（Cancel）

用户向 Agent 发送请求后，因等待过长或需求变更，可随时取消当前执行：

```go
cancelOpt, cancelFunc := adk.WithCancel()
iter := runner.Run(ctx, messages, cancelOpt)

// 发起取消
handle, _ := cancelFunc(adk.WithAgentCancelMode(adk.CancelImmediate))
err := handle.Wait()
```

**三种取消模式**：
- `CancelImmediate` — 立即中断，不等待安全点
- `CancelAfterChatModel` — 等当前 ChatModel 调用完成后取消
- `CancelAfterToolCalls` — 等当前 ToolCalls 全部完成后取消

### 2. TurnLoop 多轮循环

TurnLoop 是**持久运行的多轮执行循环**，在轮次之间保持 idle 等待，随时可通过 `Push()` 接收新输入并立即响应。

```go
loop := adk.NewTurnLoop(cfg)
loop.Push(&ChatItem{Query: "hello"})
loop.Run(ctx)
// → GenInput 构建输入 → Agent 执行 → OnAgentEvents 流式输出 → idle 等待
```

**关键特性**：
- ✅ 跨轮次持久运行、实时响应新输入
- ✅ 抢占正在进行的回答：`loop.Push(item, adk.WithPreempt(adk.AfterToolCalls))`
- ✅ 中止 Agent：`loop.Stop(adk.WithImmediate())`
- ✅ 灵活的 per-turn 输入构建：`GenInput` 回调

### 3. 抢占机制（Preempt）

当用户在 Agent 回答过程中发送新消息时，可触发抢占：

```go
loop.Push(item, adk.WithPreempt[*ChatItem, M](adk.AfterToolCalls))
```

TurnLoop 收到指令后：
1. 等待当前 tool call 完成（`AfterToolCalls` 不打断正在执行的工具）
2. 取消当前轮次
3. 从队列取出新 item，启动新一轮

**抢占安全点**：
| 模式 | 行为 |
|------|------|
| `AfterToolCalls` | 等待当前正在执行的工具调用完成后取消 |
| `AfterChatModel` | 等待当前大模型调用完成后取消 |
| `AnySafePoint` | 在任一安全点立即取消 |

### 4. 中止机制（Stop）

直接停止整个 TurnLoop：

```go
loop.Stop(adk.WithImmediate())  // 立即取消，不等待当前轮次
loop.Wait()                     // 等待完全退出
```

**三种停止模式**：
| 模式 | 行为 |
|------|------|
| `loop.Stop()` | 轮次边界退出：等待当前轮次完成后退出 |
| `loop.Stop(WithImmediate())` | 立即退出：取消当前轮次的 context |
| `loop.Stop(WithGraceful())` | 安全点退出：在下一个安全点退出 |

## 使用场景

### 典型模式：一个 Session 对应一个 TurnLoop

每个聊天 session 对应一个 TurnLoop 实例。用户发送第一条消息时创建并启动；后续消息通过 `Push()` 送入同一个循环。循环的生命周期与用户会话绑定。

```go
// 用户发送第一条消息时，创建并启动 TurnLoop
loop := adk.NewTurnLoop(cfg)
loop.Push(&ChatItem{Query: "hello"})
loop.Run(ctx)

// 用户发送第二条消息（此时 loop 处于 idle）
loop.Push(&ChatItem{Query: "explain Eino's architecture"})
```

## 与 Runner 的对比

| 能力 | Runner（单轮） | TurnLoop（多轮） |
|------|---------------|-----------------|
| 流式输出 | ✅ | ✅ |
| 审批 / 中断 | ✅ | ✅ |
| 跨轮次持久运行 | ❌ 每次 Run() 独立 | ✅ Push() 随时送入 |
| 抢占正在进行的回答 | ❌ | ✅ WithPreempt |
| 中止 Agent | ❌ | ✅ loop.Stop |
| 灵活的 per-turn 输入构建 | ❌ 业务层手动拼装 | ✅ GenInput 回调 |

## 官方资源

- [Eino ADK 文档](https://github.com/cloudwego/eino)
- [TurnLoop 示例代码](https://github.com/cloudwego/eino-examples/tree/main/quickstart/chatwitheino)
