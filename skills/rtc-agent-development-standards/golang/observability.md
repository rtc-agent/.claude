# Go 可观测性规范

> _不可观测的系统是黑盒。黑盒中的 bug 无法定位，性能瓶颈无法发现，容量规划无从谈起。_

---

## 1. 结构化日志

### 必须遵循

- **使用 `pkg/logger`**：所有日志通过项目统一的 logger 输出，不直接使用 `fmt.Println` 或 `log.Printf`
- **key-value 格式**：所有上下文信息以结构化字段传递，不拼接字符串

```go
// ✅ 结构化
logger.Info("session created",
    "session.id", session.ID,
    "user.id", session.OwnerRefID,
    "session.type", session.Type,
)

// ❌ 字符串拼接
logger.Info(fmt.Sprintf("session created: %s, user: %s", session.ID, session.OwnerRefID))
```

### 必须包含的上下文字段

| 场景 | 必须字段 |
|------|---------|
| RPC handler | `user.id`, `method` |
| 后台任务 | `session.id`, `turn.id`, `work.kind` |
| repo 操作 | 操作名 + 主键（如 `session.id`） |
| 外部调用 | 目标服务名 + 耗时 |

### 日志级别

| 级别 | 使用场景 |
|------|---------|
| **Debug** | 仅在开发环境启用；详细的中间状态、请求/响应体 |
| **Info** | 关键业务事件：会话创建、消息发送、turn 完成 |
| **Warn** | 可恢复的异常：重试成功、降级触发、慢查询 |
| **Error** | 不可恢复的错误：需人工介入，必须含 stack trace |

### 禁止

- 不打印敏感信息：token、password、API key、完整请求体中的用户数据
- 不在循环中打日志（会淹没日志系统）
- 不用 `fmt.Sprintf` 拼接日志消息

---

## 2. 分布式追踪

### 新增 handler / RPC 方法

必须创建 span，记录关键属性和错误。

```go
func (h *Handler) HandleSendMessage(ctx context.Context, req *SendRequest) (*SendResponse, error) {
    ctx, span := tracer.Start(ctx, "rpc.SendMessage",
        trace.WithAttributes(
            attribute.String("user.id", req.UserID),
            attribute.String("session.id", req.SessionID),
        ),
    )
    defer span.End()

    resp, err := h.service.SendMessage(ctx, req)
    if err != nil {
        span.RecordError(err)
        span.SetStatus(codes.Error, err.Error())
        return nil, err
    }
    return resp, nil
}
```

### 新增 repo 方法

数据库操作记录耗时指标，在 repo 层不单独创建 span（避免 span 爆炸）。慢查询通过指标告警。

### 新增后台 goroutine

必须从 context 获取 tracer 和 logger，并在 goroutine 入口创建 span。

```go
go func(ctx context.Context) {
    ctx, span := tracer.Start(ctx, "worker.processSession")
    defer span.End()

    // 工作...
}(ctx)
```

### traceparent 传播

跨 Redis PUB/SUB 边界时，必须使用 `pkg/centrifuge-plus/tracer.go` 中的工具函数传播 W3C traceparent。

```go
// 发送端
payload := appendTraceparent(ctx, data)

// 接收端
ctx = extractTraceparent(ctx, payload)
```

**新增涉及 Redis 消息传递的功能时，必须传播 traceparent，不可省略。**

---

## 3. 指标采集

### 命名规范

```
rtc_agent_<module>_<metric>_<unit>
```

示例：

```
rtc_agent_rpc_request_duration_seconds
rtc_agent_rpc_request_total
rtc_agent_turn_duration_seconds
rtc_agent_queue_depth
rtc_agent_llm_call_tokens_total
```

### 标签规范

- 标签数不超过 5 个
- 标签值必须是有限集合（不能用 user ID、session ID 等高基数值）
- 常用标签：`method`、`status`、`error_type`、`work_kind`

### 新增功能必须打点的指标

| 场景 | 指标 |
|------|------|
| 新增 RPC 方法 | 请求数、耗时分布、错误数 |
| 新增 repo 操作 | 耗时分布、慢查询计数 |
| 新增后台任务 | 执行次数、耗时、成功/失败比 |
| 新增 Redis 操作 | 耗时、连接池使用率 |
| 新增 LLM 调用 | token 消耗、耗时、错误数 |

### 使用 Metrics 接口

新增指标通过 `pkg/turn-agent/metrics.go` 的 `Metrics` 接口扩展，不直接依赖 Prometheus 客户端库。

```go
// ✅ 通过接口
metrics.RecordTurn(ctx, turnID, status, duration)

// ❌ 直接依赖 Prometheus
promCounter.WithLabelValues("success").Inc()
```

---

## 4. 健康检查

### 新增依赖时必须添加检查

当系统新增外部依赖（数据库、缓存、消息队列、第三方服务）时，必须在 `/readyz` 端点添加对应的健康检查。

```go
func (s *Server) readyzHandler(w http.ResponseWriter, r *http.Request) {
    checks := map[string]error{
        "database":    s.db.DB().Ping(r.Context()),
        "redis":       s.redis.Ping(r.Context()).Err(),
        "centrifuge":  s.centrifugeNode.Health(),
        // 新增依赖时在此添加
    }
    // ...
}
```

### 检查标准

- 超时设置：单个检查不超过 3 秒
- 整体超时：所有检查不超过 5 秒
- 分级响应：列出每个依赖的状态，方便定位问题

---

## 5. 新模块可观测性检查清单

每个新模块上线前，必须确认以下项目：

- [ ] 所有公开方法有日志输出（入口 + 出口 + 错误）
- [ ] 关键路径有 span（handler、外部调用）
- [ ] 关键指标已注册（请求数、耗时、错误数）
- [ ] 错误日志包含 stack trace
- [ ] 不打印敏感信息
- [ ] 跨 Redis/PUB-SUB 边界传播 traceparent
- [ ] 健康检查已更新（如有新依赖）
- [ ] 日志级别使用正确（debug 仅开发环境）

---

## 总结

可观测性不是事后补丁，而是设计的一部分。新增任何功能时，把日志、追踪、指标作为功能的组成部分一起交付，而非"以后再加"。

> _"如果一个功能没有日志，那它就没有完成。"_
