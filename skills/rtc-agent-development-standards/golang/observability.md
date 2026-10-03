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

### 日志事件名命名约定

日志的第一个参数（事件名）使用 `<模块或函数>.<具体事件>` 的 `snake_case` 点分隔格式，便于日志系统按模块过滤和告警。

```go
// ✅ 模块.事件 格式
logger.Info(ctx, "file_attachment.process_start", ...)
logger.Info(ctx, "file_attachment.load_image_success", ...)
logger.Warn(ctx, "file_attachment.load_text_failed", ...)
logger.Warn(ctx, "convertUserMessage.oss_not_configured", ...)
logger.Info(ctx, "gen_input.checkpoint_lost_fallback", ...)
logger.Warn(ctx, "gen_resume.history_modifier_load_failed", ...)

// ❌ 无结构、随意命名
logger.Info(ctx, "start processing file", ...)
logger.Info(ctx, "FileLoaded", ...)
logger.Warn(ctx, "oss issue", ...)
```

**规则**：

- 模块部分用函数名或子系统名（`file_attachment`、`convertUserMessage`、`gen_input`）
- 事件部分用 `snake_case`（`process_start`、`load_image_success`）
- 成功和失败用后缀区分（`_success`、`_failed`、`_start`、`_done`）
- 可选依赖跳过统一用 `<函数名>.<依赖>_not_configured`（参见下方"可选依赖跳过时打 Warn"）

### 禁止

- 不打印敏感信息：token、password、API key、完整请求体中的用户数据
- 不在循环中打日志（会淹没日志系统）
- 不用 `fmt.Sprintf` 拼接日志消息

### 可选依赖跳过时打 Warn

当功能依赖可选基础设施（如 OSS、外部缓存、第三方 API）而未配置时，必须在功能被跳过的路径打 `Warn` 日志，帮助运维快速定位"功能为什么不生效"。

```go
// ✅ 可选依赖未配置时打 Warn
if umc.Files != nil && len(*umc.Files) > 0 {
    if h.ossBackend == nil {
        h.logger.Warn(ctx, "convertUserMessage.oss_not_configured", map[string]any{
            "file_count": len(*umc.Files),
            "message":    "file attachments are skipped because OSS is not configured",
        })
    } else {
        h.processFileAttachments(ctx, result, *umc.Files)
    }
}

// ❌ 静默跳过，用户和运维都无法诊断
if umc.Files != nil && h.ossBackend != nil {
    h.processFileAttachments(ctx, result, *umc.Files)
}
// 当 ossBackend == nil 时，文件附件被静默丢弃，无任何日志
```

**规则**：

- 用 `Warn` 级别（不是 `Error`），因为这是预期内的降级，不是异常
- 日志事件名格式：`<函数名>.<依赖>_not_configured`（如 `convertUserMessage.oss_not_configured`）
- 包含上下文信息（如 `file_count`），帮助判断影响范围
- 仅用于"有输入但因依赖缺失而被跳过"的场景；没有输入时不需要打日志

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

### 新增 HTTP handler（协议兼容层）

对于协议兼容层（如 OSS3 S3 handler），每个操作函数使用 tracer 创建 span，记录请求级属性。tracer 实例通过包级变量获取。

```go
var tracer = otel.GetTracerProvider().Tracer("oss3")

func (h *OSS3Handler) handlePutObject(w http.ResponseWriter, r *http.Request, bucket, key string) {
    ctx, span := tracer.Start(r.Context(), "oss3.PutObject",
        trace.WithAttributes(
            attribute.String("oss3.bucket", bucket),
            attribute.String("oss3.key", key),
            attribute.Int64("oss3.content_length", r.ContentLength),
            attribute.String("user.id", contextx.GetOSS3UserID(r.Context())),
        ),
    )
    defer span.End()

    // ... 业务逻辑

    if err != nil {
        span.RecordError(err)
        span.SetStatus(codes.Error, err.Error())
        return
    }
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

### 跨队列边界传播 OpenTelemetry Trace Context

当工作项通过队列系统（asynq、rtc-queue 等非 PUB/SUB 机制）传递时，W3C traceparent 自动传播不适用。必须将 TraceID 和 SpanID 显式嵌入 payload struct，在消费端通过 `turnagent.WithTraceContext()` 恢复 trace context。

```go
// ✅ 生产端：提取 trace context，嵌入 payload
type WorkPayload struct {
    SessionID string `json:"session_id"`
    UserID    string `json:"user_id"`
    TraceID   string `json:"trace_id,omitempty"`   // OpenTelemetry trace ID
    SpanID    string `json:"span_id,omitempty"`     // OpenTelemetry span ID
}

func enqueueWork(ctx context.Context, sessionID string) {
    spanCtx := trace.SpanContextFromContext(ctx)
    payload := WorkPayload{
        SessionID: sessionID,
        TraceID:   spanCtx.TraceID().String(),
        SpanID:    spanCtx.SpanID().String(),
    }
    queue.Enqueue(payload)
}

// ✅ 消费端：从 payload 恢复 trace context
func processWork(ctx context.Context, payload WorkPayload) {
    if payload.TraceID != "" && payload.SpanID != "" {
        ctx = turnagent.WithTraceContext(ctx, payload.TraceID, payload.SpanID)
    }
    ctx, span := tracer.Start(ctx, "worker.processWork")
    defer span.End()
    // 后续操作自动关联到原始请求的 trace
}

// ❌ 不传播 trace context
func processWork(ctx context.Context, payload WorkPayload) {
    // ctx 是 worker 的独立 context，与原始请求的 trace 断开
    // 无法在 Jaeger/Tempo 中追踪从请求到队列处理的完整链路
}
```

**约束**：

- 涉及队列传递的 payload struct 必须包含 `TraceID` 和 `SpanID` 字段（`omitempty`，兼容旧 payload）
- 消费端必须在创建新 span 之前恢复 trace context
- 恢复失败（空字段）时静默跳过（旧 payload 可能没有 trace 信息），不报错

**适用场景**：asynq 任务队列、rtc-queue 工作项分发、任何非 PUB/SUB 的异步队列边界。与 traceparent 传播互补：PUB/SUB 用 traceparent（自动传播），队列用 struct 字段（显式传播）。

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

**例外：独立子系统使用子系统前缀**。当一个子系统规模足够大（10+ 文件、独立协议兼容层），且其指标域与主系统正交时，使用 `rtc_<subsystem>_<metric>_<unit>` 命名。判断标准：子系统的指标是否需要在独立的 Prometheus 告警规则和 Grafana Dashboard 中管理。

```
# ✅ 独立子系统：OSS3（S3 兼容层）
rtc_oss3_upload_bytes_total
rtc_oss3_request_duration_seconds
rtc_oss3_consistency_violation_total
rtc_oss3_quota_commit_retry_total

# ✅ 主系统模块
rtc_agent_rpc_request_duration_seconds
rtc_agent_turn_duration_seconds
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

**例外：独立子系统可直接使用 Prometheus**。当子系统满足以下全部条件时，可直接使用 `promauto` 注册指标：

1. 子系统规模大（10+ 文件），有独立的协议兼容层或外部接口
2. 指标域与主系统正交（不共享 label 维度）
3. 指标需要在独立的 Grafana Dashboard 和告警规则中管理

当前例外：`internal/handler/http/oss3_metrics.go`（OSS3 S3 兼容层，使用 `rtc_oss3_*` 命名前缀）。

```go
// ✅ 主系统模块：通过 Metrics 接口
metrics.RecordTurn(ctx, turnID, status, duration)

// ✅ 独立子系统例外：直接使用 promauto
var oss3RequestsTotal = promauto.NewCounterVec(
    prometheus.CounterOpts{
        Name: "rtc_oss3_requests_total",
        Help: "Total OSS3 S3 requests",
    },
    []string{"operation", "status"},
)
```

**判断标准**：新增子系统如果只需要 3-5 个通用指标（请求数、耗时、错误数），通过 `Metrics` 接口扩展。如果需要 10+ 个领域特定指标（一致性违规、孤儿资源、配额漂移等），可以使用 Prometheus 直接注册。

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
- [ ] 跨 Redis PUB/SUB 边界传播 traceparent；跨 asynq/rtc-queue 边界在 payload struct 中传播 TraceID/SpanID（参见下方[跨队列边界传播 OpenTelemetry Trace Context](#跨队列边界传播-opentelemetry-trace-context)）
- [ ] 健康检查已更新（如有新依赖）
- [ ] 日志级别使用正确（debug 仅开发环境）

---

## 总结

可观测性不是事后补丁，而是设计的一部分。新增任何功能时，把日志、追踪、指标作为功能的组成部分一起交付，而非"以后再加"。

> _"如果一个功能没有日志，那它就没有完成。"_
