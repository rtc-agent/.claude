# /goal 命令设计文档 (v2)

**版本**: 2.0
**日期**: 2026-09-07
**状态**: 评审后重写（统一创建/取消路径、修正字段命名、明确 RPC 边界）

> v2 相对 v1 的主要变更：
>
> 1. **统一 goal 创建路径**：删除 `set_goal` RPC，创建只走 Agent `create_goal` tool；前端 `/goal` 作为 UX 入口，通过 send message RPC 发送，后端解析前缀注入 goal 提示词
> 2. **统一 goal 取消路径**：删除 `clear_goal` RPC，取消只走 Agent `cancel_goal` tool
> 3. **修正字段命名**：`ended_at` → `completed_at`；§3.5 字段说明补齐 `exhausted` 状态
> 4. **Judge 失败也 re-queue**：不再挂起等用户下次发消息；max_turns 兜底
> 5. **轮次计算规则明确**：主 Agent turn 完成即 `completed_turns += 1`，不论 Judge 是否报错
> 6. **token_usage 语义**：goal 生命周期内 session 的累计 token（主 Agent + Judge 合计），不再单独追踪 Judge
> 7. **Feedback 消息 role**：改为 `system`
> 8. **架构图修正**：Session 模型不再包含 goal 字段
> 9. **前端历史面板**：后端提供 `list_goals` RPC，前端暂不做 UI

---

## 1. 概述

### 1.1 功能定位

`/goal` 机制让 AI 持续工作直到某个完成条件满足。核心交互：

- **创建 goal**：前端 `/goal <自然语言目标>` 作为 UX 入口 → send message RPC → 后端解析 `/goal` 前缀，注入 **goal 提示词** 到 turn 上下文 → Agent 按提示词**先调研**，再**润色**成 SMART 条件，向用户提议 → 用户确认后 → Agent 调用 `create_goal` tool
- **取消 goal**：用户自然语言（"停下"、"算了"等）→ Agent 识别意图 → 调用 `cancel_goal` tool
- **完成 goal**：后端 Judge 模型判定达成，自动标记 `completed`
- **耗尽 goal**：后端达到 `max_turns` 仍未达成，自动标记 `exhausted`

前端**只有一个 `/goal <目标>` 命令**（无子命令如 `/goal clear`、`/goal history`）。取消通过自然语言对话；历史查询后端提供 RPC 接口，前端暂不做 UI。

### 1.2 核心机制

```text
用户输入：/goal 帮我把所有测试通过
   ↓
前端：send message RPC（内容: "/goal 帮我把所有测试通过"）
   ↓
后端：解析 /goal 前缀 → 注入 goal 提示词到 turn 上下文
   ↓
turn 1: Agent 按 goal 提示词工作
   ├─ 调研阶段：跑测试、看代码，了解现状
   ├─ 润色阶段：把模糊目标变成 SMART 条件
   ├─ 提议阶段：向用户提议 "我打算把 goal 定为：所有 42 个测试通过且无新增失败，你同意吗？"
   └─ 等待用户确认
   ↓
用户：同意
   ↓
turn 2（或 turn 1 后半段）: Agent 调用 create_goal tool
   ├─ 后端创建 goal 行（status=active）
   ├─ 推送 goal.created 事件
   └─ Agent 开始实际工作（修代码、跑测试）→ turn 完成
   ↓
Goal Checkpoint: completed_turns += 1 → Judge 评判
   ├─ achieved → goal.status=completed, 推送 goal.ended
   ├─ not achieved → 持久化 feedback → rtc-queue 入队 goal_resume
   └─ judge error → 持久化 fallback feedback → rtc-queue 入队 goal_resume（max_turns 兜底）
   ↓
turn 3: AI 看到 feedback 继续工作 → turn 完成 → Goal Checkpoint → ...
   ↓
循环直到：达成 / 耗尽 / 用户说"停下"→ Agent 调 cancel_goal
```

### 1.3 设计原则

1. **Agent 当 goal 教练**：用户说模糊意图，Agent 负责润色成 SMART 条件。用户不需要懂如何写"好 goal"
2. **自然语言驱动**：用户不说"创建 goal"，而是说"帮我把 X 做了"、"停下来" —— Agent 识别意图并调用对应 tool
3. **后端解析 goal 入口**：前端 `/goal` 是 UX 语法糖，后端通过解析消息前缀识别 goal 意图并注入专用提示词，确保 Agent 知道该走"调研 → 润色 → 提议 → 创建"流程
4. **turn 边界检查**：Goal 评判发生在 turn 之间，不侵入 turn 内部的 ChatModelAgent / middleware
5. **复用调度基础设施**：通过 rtc-queue 驱动多轮，与用户主动发消息走同一条路径，cancel / 限流 / 监控 / Centrifuge 推送全部免费继承
6. **显式状态管理**：goal 状态持久化到独立 goals 表，feedback 消息持久化到 DB，不依赖内存 state
7. **轻量评判**：使用独立的轻量模型（Haiku）降低成本，与主模型解耦
8. **防护机制**：max_turns 上限（默认 50 turn），防止失控；创建时固化当时的 max_turns 配置；Judge 失败也计入轮次，避免无限挂起

---

## 2. 架构设计

### 2.1 整体架构

**核心思路**：goal 驱动的"持续工作"是 **turn 间** 的编排，不是 turn 内的 middleware 钩子。每轮 turn 走完整的 `rtc-queue → Process() → TurnLoop → ChatModelAgent` 路径；turn 完成后，系统在边界检查 goal 状态，未达成则把 feedback 作为新 work item 重新投回 rtc-queue，触发下一轮 turn。

```
┌─────────────────────────────────────────────────────────────────────┐
│                         Frontend (Web)                               │
│  /goal <目标> → send message RPC（内容以 "/goal " 开头）             │
│  状态栏订阅 goal.* 事件；历史面板（暂不实现）                         │
└─────────────────────────────────────────────────────────────────────┘
                                ↓
┌─────────────────────────────────────────────────────────────────────┐
│                      Backend (Server)                                │
│                                                                      │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │               Message Ingestion                                │   │
│  │  解析消息前缀：若 "/goal " → 注入 goal 提示词到 turn 上下文  │   │
│  └──────────────────────────────────────────────────────────────┘   │
│                                ↓                                     │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │               rtc-queue (work items)                          │   │
│  │   submit │ goal_resume │ rtc_result │ ...                     │   │
│  └──────────────────────────────────────────────────────────────┘   │
│           ↓                                 ↑                        │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │            Worker → agent.Process(WorkPayload)                │   │
│  │              ↓                                                │   │
│  │     TurnLoop → ChatModelAgent (+ summarizeMW)                 │   │
│  │              ↓                                                │   │
│  │     OnAgentEvents → 流式事件 → Centrifuge                     │   │
│  │              ↓                                                │   │
│  │     turn 完成 (completeTurn / cancelTurn / failTurn)          │   │
│  │              ↓                                                │   │
│  │     ★ Goal Checkpoint (turn 边界)                             │   │
│  │       ├─ goal 非 active → 直接返回                            │   │
│  │       ├─ completed_turns >= max_turns → 标记 exhausted        │   │
│  │       ├─ completed_turns += 1（本轮一定计入）                 │   │
│  │       ├─ 调用 Judge 评判（传入 toolNames）                    │   │
│  │       │    ├─ achieved → goal.status=completed, publish       │   │
│  │       │    ├─ not achieved → 持久化 feedback → re-queue       │   │
│  │       │    └─ judge error → 持久化 fallback feedback → re-queue│  │
│  └──────────────────────────────────────────────────────────────┘   │
│                                                                      │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │               goals 表（独立）                                 │   │
│  │  Goal { id, session_id, condition, status, completed_turns,  │   │
│  │         token_usage, max_turns, created_at, completed_at,    │   │
│  │         last_judge_reason }                                  │   │
│  └──────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────┘
```

**为什么不在 ChatModelAgent middleware 内做**：

- middleware（如 `AfterModelRewriteState`）作用于 **单次 turn 内** 的 ReAct 循环，turn 结束后 TurnLoop 与 agent 实例全部销毁
- goal 驱动的"继续"需要跨 turn 编排，必须回到 rtc-queue 这条 turn 调度的单一入口
- 复用 rtc-queue 路径意味着 cancel / 限流 / 监控 / Centrifuge 推送 / turn 事件全部免费继承

### 2.2 核心组件

| 组件 | 职责 | 位置 |
|------|------|------|
| **Goal Checkpoint** | turn 边界检查 goal 状态，调用 Judge，决定是否 re-queue | `server/internal/agent/goal_checkpoint.go` |
| **Judge** | 独立评判模型，判断 goal 是否达成 | `server/internal/agent/judge.go` |
| **Goal Storage** | 持久化 goal 状态（独立表） | `server/internal/model/goal.go` |
| **Goal Prompt Injector** | 解析 `/goal` 前缀，注入 goal 提示词 | `server/internal/agent/goal_prompt.go` |
| **WorkPayload 扩展** | 新增 `WorkKindGoalResume`，标识 goal 驱动的 turn | `server/pkg/turn-agent/types.go` |

---

## 3. 数据模型

### 3.1 设计要点

**Goal 独立于 Session**。Session 是"对话容器"，Goal 是"工作目标"——两个正交的维度。把 Goal 作为独立表：

- 一个 Session 可以顺序跑多个 Goal（保留历史）
- 同一个 Goal 模板可以被多个 Session 复用（未来扩展）
- Goal 的状态变更不会污染 Session 行（避免 JSONB 频繁更新）
- Goal 历史可独立查询、统计、归档

### 3.2 Goal 模型

```go
// server/internal/model/goal.go

// Goal 状态常量
const (
    GoalStatusActive    = "active"     // 正在执行
    GoalStatusCompleted = "completed"  // 达成
    GoalStatusCancelled = "cancelled"  // 用户主动取消
    GoalStatusExhausted = "exhausted"  // 达到 max turns 仍未达成
)

// TerminalStatuses 终态集合（用于查询"是否已结束"）
var TerminalStatuses = []string{GoalStatusCompleted, GoalStatusCancelled, GoalStatusExhausted}

type Goal struct {
    ID              uuid.UUID  `gorm:"type:uuid;primaryKey" json:"id"`
    SessionID       uuid.UUID  `gorm:"type:uuid;not null;index:idx_goal_session_status" json:"session_id"`
    Condition       string     `gorm:"type:text;not null" json:"condition"`
    Status          string     `gorm:"type:varchar(20);not null;index:idx_goal_session_status;default:'active'" json:"status"`
    CompletedTurns  int        `gorm:"not null;default:0" json:"completed_turns"`
    TokenUsage      int        `gorm:"not null;default:0" json:"token_usage"`        // goal 生命周期内 session 累计 token（主 Agent + Judge）
    MaxTurns        int        `gorm:"not null;default:50" json:"max_turns"`         // 创建时固化当时的配置
    CreatedAt       time.Time  `gorm:"not null" json:"created_at"`
    CompletedAt     *time.Time `json:"completed_at,omitempty"`                       // 进入终态的时间（任意终态都填）
    LastJudgeReason string     `gorm:"type:text" json:"last_judge_reason,omitempty"`
}

func (Goal) TableName() string { return "goals" }

// IsTerminal 是否进入终态
func (g Goal) IsTerminal() bool {
    return g.Status == GoalStatusCompleted ||
        g.Status == GoalStatusCancelled ||
        g.Status == GoalStatusExhausted
}
```

**索引**：`(session_id, status)` 复合索引 —— 高频查询是"找出某 session 的 active goal"。

**`max_turns` 字段**：创建 goal 时固化当时的 max_turns 配置值，避免未来调整全局配置导致历史 goal 的判定标准漂移。

**`token_usage` 字段**：goal 生命周期内 session 的**累计 token 总消耗**（主 Agent + Judge 合计）。用于前端展示"这个 goal 花了多少 token"，未来可用于 token 预算上限。

**`completed_at` 字段**：goal 进入任意终态（completed / cancelled / exhausted）的时间。统一用一个字段覆盖所有终态，语义为"goal 完结时间"。

### 3.3 Session 模型（不修改）

**Session 模型不变**。不添加 `Goal` 字段，也不添加 `CurrentGoalID` 指针。

> 为什么不加 `CurrentGoalID`？避免双向状态同步的复杂度（goal 状态变了还要同步 session 行）。查询 active goal 直接走 goals 表的索引，开销可忽略。

### 3.4 GoalRepo

新增 Goal 仓储接口，遵循现有 Repo 模式（参考 [session_repo.go](server/internal/repo/session_repo.go)）：

```go
// server/internal/repo/goal_repo.go

type GoalRepo interface {
    Create(ctx context.Context, goal *model.Goal) error
    GetByID(ctx context.Context, id uuid.UUID) (*model.Goal, error)
    // FindActive 返回 session 当前活跃的 goal（最多一个）
    FindActive(ctx context.Context, sessionID uuid.UUID) (*model.Goal, error)
    // Update 通用字段更新
    Update(ctx context.Context, id uuid.UUID, fields map[string]any) error
    // ListBySession 列出 session 的所有 goal（历史查询）
    ListBySession(ctx context.Context, sessionID uuid.UUID, limit int, cursor string) ([]*model.Goal, string, error)
}
```

### 3.5 字段说明

| 字段 | 类型 | 说明 |
|------|------|------|
| `condition` | text | 完成条件的自然语言描述 |
| `status` | varchar(20) | `active` / `completed` / `cancelled` / `exhausted` |
| `completed_turns` | int | 已完成的主 Agent turn 数（每次 turn 正常完成 +1，不论 Judge 状态） |
| `token_usage` | int | goal 生命周期内 session 累计 token（主 Agent + Judge） |
| `max_turns` | int | 创建时固化的 max_turns 配置 |
| `created_at` | timestamp | goal 创建时间 |
| `completed_at` | timestamp | goal 进入终态的时间（任意终态都填） |
| `last_judge_reason` | text | 最近一次 Judge 给出的 reason，供前端展示进度 |

### 3.6 数据库迁移

```go
// server/internal/model/migrate.go
func AutoMigrate(db *gorm.DB) error {
    return db.AutoMigrate(
        // ... 现有模型 ...
        &Goal{},  // 新增
    )
}
```

### 3.7 Goal 状态更新示例

```go
// Goal Checkpoint 内: 累加 completed_turns + 写 last_judge_reason
err := deps.GoalRepo.Update(ctx, goal.ID, map[string]any{
    "completed_turns":   goal.CompletedTurns + 1,
    "last_judge_reason": reason,
    "token_usage":       goal.TokenUsage + turnTotalTokenUsage,  // 本 turn 主 Agent + Judge 的 token 合计
})

// Goal 达成:
now := time.Now()
err := deps.GoalRepo.Update(ctx, goal.ID, map[string]any{
    "status":        "completed",
    "completed_at":  now,
})
```

---

## 4. Goal Checkpoint 设计

### 4.1 设计思路

Goal Checkpoint 不是 middleware，而是 **turn 完成后的编排逻辑**。它挂在 `agent.Process()` 的末尾，与现有的 `prompt-too-long → reactive compact → 重试` 逻辑并列。

**触发条件**：turn 以 `completeTurn` 状态结束（不是 cancel / interrupt / fail）。

**四种结果**：

1. **goal 非 active** → 直接返回，无事发生
2. **达到 max_turns** → 标记 `exhausted`，推送 `goal.ended` 事件，不 re-queue
3. **goal 达成** → 标记 `completed`，推送 `goal.ended` 事件，不 re-queue
4. **goal 未达成（含 Judge 失败）** → `completed_turns += 1` → 持久化 feedback → re-queue

### 4.2 接口定义

```go
// server/internal/agent/goal_checkpoint.go

type goalCheckpoint struct {
    deps            *usecase.Dependencies
    judge           *Judge
    publisher       updates.UpdatePublisher  // 用于推送 goal.* 事件
    queue           queue.Publisher          // rtc-queue, 用于 re-queue
    logger          turnagent.Logger
    defaultMaxTurns int                      // 创建新 goal 时的默认值（从配置读取）
}

func newGoalCheckpoint(
    deps *usecase.Dependencies,
    judge *Judge,
    publisher updates.UpdatePublisher,
    queue queue.Publisher,
    logger turnagent.Logger,
    defaultMaxTurns int,
) *goalCheckpoint {
    return &goalCheckpoint{
        deps:            deps,
        judge:           judge,
        publisher:       publisher,
        queue:           queue,
        logger:          logger,
        defaultMaxTurns: defaultMaxTurns,
    }
}
```

### 4.3 核心方法

```go
// Evaluate 在 turn 完成后调用，判断是否需要触发下一轮
// 参数: sessionID, turnID (刚完成的 turn), finalStatus (turn 状态), turnTokenUsage (本 turn 主 Agent token)
func (g *goalCheckpoint) Evaluate(
    ctx context.Context,
    sessionID uuid.UUID,
    finalStatus TurnStatus,
    turnTokenUsage int,
) error {
    // 1. 只在正常完成的 turn 上评判
    //    cancel / interrupt / fail 不算一轮，不消耗 turn 配额
    if finalStatus != TurnStatusCompleted {
        return nil
    }

    // 2. 从 goals 表查当前 session 的 active goal
    goal, err := g.deps.GoalRepo.FindActive(ctx, sessionID)
    if err != nil {
        return fmt.Errorf("find active goal: %w", err)
    }
    if goal == nil {
        return nil  // 无活跃 goal
    }

    // 3. 加载 Session（用于事件推送，拿 ownerRefID 路由到用户 topic channel）
    session, err := g.deps.SessionRepo.GetByID(ctx, sessionID)
    if err != nil {
        return fmt.Errorf("load session: %w", err)
    }

    // 4. 累加 token_usage（本 turn 主 Agent 的 token，Judge 的 token 在 Evaluate 内部再加）
    newTokenUsage := goal.TokenUsage + turnTokenUsage

    // 5. 主 Agent turn 完成，先计入轮次（不论后续 Judge 状态）
    newTurns := goal.CompletedTurns + 1

    // 6. 达到最大轮次 → 标记 exhausted，推送事件，不 re-queue
    if newTurns > goal.MaxTurns {
        now := time.Now()
        if err := g.deps.GoalRepo.Update(ctx, goal.ID, map[string]any{
            "completed_turns": newTurns,
            "token_usage":     newTokenUsage,
            "status":          model.GoalStatusExhausted,
            "completed_at":    now,
        }); err != nil {
            return fmt.Errorf("mark goal exhausted: %w", err)
        }
        goal.Status = model.GoalStatusExhausted
        goal.CompletedAt = &now
        goal.CompletedTurns = newTurns
        goal.TokenUsage = newTokenUsage
        g.logIfEnabled(ctx, "goal.exhausted", map[string]any{
            "goal_id":         goal.ID.String(),
            "session_id":      sessionID.String(),
            "completed_turns": newTurns,
            "max_turns":       goal.MaxTurns,
        })
        _ = g.publishGoalEvent(ctx, session, "goal.ended", goal)
        return nil
    }

    // 7. 加载最近对话历史供 Judge 评判
    messages, err := g.deps.MessageRepo.GetRecent(ctx, sessionID, 20)
    if err != nil {
        return fmt.Errorf("load recent messages: %w", err)
    }

    // 8. 收集 Agent 可用的 toolNames（让 Judge 知道 AI 实际能做什么）
    toolNames := g.collectAgentToolNames(ctx, sessionID)

    // 9. 调用 Judge 评判
    achieved, reason, judgeTokenUsage, err := g.judge.Evaluate(ctx, goal.Condition, messages, toolNames)
    newTokenUsage += judgeTokenUsage  // Judge token 也要累加

    if err != nil {
        // Judge 失败：也计入轮次 + 也 re-queue，max_turns 兜底
        g.logIfEnabled(ctx, "goal.judge_error", map[string]any{
            "goal_id":    goal.ID.String(),
            "session_id": sessionID.String(),
            "error":      err.Error(),
        })
        reason = "Judge failed to evaluate; assuming not achieved"

        if updateErr := g.deps.GoalRepo.Update(ctx, goal.ID, map[string]any{
            "completed_turns":   newTurns,
            "token_usage":       newTokenUsage,
            "last_judge_reason": reason,
        }); updateErr != nil {
            return fmt.Errorf("update goal after judge error: %w", updateErr)
        }

        feedbackContent := fmt.Sprintf(
            "[Goal Not Yet Achieved — Turn %d/%d]\nJudge evaluation failed. Please continue working on the goal: %s",
            newTurns, goal.MaxTurns, goal.Condition,
        )
        if err := g.persistFeedbackMessage(ctx, sessionID, feedbackContent); err != nil {
            return fmt.Errorf("persist feedback: %w", err)
        }
        return g.requeueGoalResume(ctx, sessionID, goal, newTurns, reason)
    }

    // 10. 更新 completed_turns / token_usage / last_judge_reason
    if err := g.deps.GoalRepo.Update(ctx, goal.ID, map[string]any{
        "completed_turns":   newTurns,
        "token_usage":       newTokenUsage,
        "last_judge_reason": reason,
    }); err != nil {
        return fmt.Errorf("update goal turns: %w", err)
    }
    goal.CompletedTurns = newTurns
    goal.TokenUsage = newTokenUsage
    goal.LastJudgeReason = reason

    if achieved {
        // 11a. 目标达成：标记 completed
        now := time.Now()
        if err := g.deps.GoalRepo.Update(ctx, goal.ID, map[string]any{
            "status":       model.GoalStatusCompleted,
            "completed_at": now,
        }); err != nil {
            return fmt.Errorf("update goal completed: %w", err)
        }
        goal.Status = model.GoalStatusCompleted
        goal.CompletedAt = &now
        _ = g.publishGoalEvent(ctx, session, "goal.ended", goal)
        return nil
    }

    // 11b. 目标未达成：推送 goal.updated + 持久化 feedback + re-queue
    _ = g.publishGoalEvent(ctx, session, "goal.updated", goal)

    feedbackContent := fmt.Sprintf(
        "[Goal Not Yet Achieved — Turn %d/%d]\nReason: %s\n\nPlease continue working on the goal: %s",
        newTurns, goal.MaxTurns, reason, goal.Condition,
    )
    if err := g.persistFeedbackMessage(ctx, sessionID, feedbackContent); err != nil {
        return fmt.Errorf("persist feedback: %w", err)
    }

    return g.requeueGoalResume(ctx, sessionID, goal, newTurns, reason)
}

// requeueGoalResume 向 rtc-queue 发布 goal_resume work item
func (g *goalCheckpoint) requeueGoalResume(ctx context.Context, sessionID uuid.UUID, goal *model.Goal, newTurns int, reason string) error {
    payload := turnagent.WorkPayload{
        Kind:      turnagent.WorkKindGoalResume,
        SessionID: sessionID.String(),
    }
    body, _ := json.Marshal(payload)
    if err := g.queue.Publish(ctx, queue.Message{
        Body:      body,
        SessionID: sessionID,
    }); err != nil {
        return fmt.Errorf("publish goal_resume: %w", err)
    }

    g.logIfEnabled(ctx, "goal.requeued", map[string]any{
        "goal_id":         goal.ID.String(),
        "session_id":      sessionID.String(),
        "completed_turns": newTurns,
        "reason":          reason,
    })
    return nil
}

// persistFeedbackMessage 把 feedback 作为一条系统消息写入 DB
// 使用 system role + ContentTypeGoalFeedback，让 summarizeMW 可以识别并特殊处理
func (g *goalCheckpoint) persistFeedbackMessage(ctx context.Context, sessionID uuid.UUID, content string) error {
    msg := &model.Message{
        SessionID:   sessionID,
        Role:        string(schema.System),  // 系统生成的反馈
        Content:     content,
        ContentType: ContentTypeGoalFeedback,
        // ... 其他必要字段
    }
    return g.deps.MessageRepo.Create(ctx, msg)
}

// publishGoalEvent 推送 goal 生命周期事件到 Centrifuge
// event 取值: "goal.created" / "goal.updated" / "goal.ended"
func (g *goalCheckpoint) publishGoalEvent(ctx context.Context, session *model.Session, event string, goal *model.Goal) error {
    return g.publisher.Publish(ctx, updates.UpdatePublishItem{
        Channel: channel.UserTopic(session.OwnerRefID),
        Entity:  updates.EntityGoal,  // 新增 entity 类型
        Event:   event,
        Payload: toProtocolGoal(goal),
    })
}

// collectAgentToolNames 收集当前 session 的 Agent 可用的 tool 名字列表
func (g *goalCheckpoint) collectAgentToolNames(ctx context.Context, sessionID uuid.UUID) []string {
    // 从 deps 拿 Agent 注册的 tool 列表，返回 []string
    // 具体实现按现有 Agent tool 注册机制
    return nil  // 占位
}
```

### 4.4 集成点

在 `agent_process.go` 的 `Process()` 末尾调用。与现有的 prompt-too-long 重试逻辑并列：

```go
// server/pkg/turn-agent/agent_process.go 的 Process() 函数

func (a *Agent) Process(ctx context.Context, work WorkPayload) (finalErr error) {
    // ... 现有逻辑 ...

    // turn 完成，决定终态
    switch {
    case errors.Is(ctx.Err(), context.Canceled):
        turnStatus = TurnStatusCancelled
    case promptTooLong:
        // 现有: reactive compact → 重试
        return a.handlePromptTooLong(ctx, ...)
    case finalErr != nil:
        turnStatus = TurnStatusFailed
    default:
        turnStatus = TurnStatusCompleted
    }

    // 写终态 (completeTurn / cancelTurn / failTurn 回调)
    a.callbacks.OnTurnEnd(ctx, turnID, turnStatus, ...)

    // ★ 新增: Goal Checkpoint
    // 仅在 turn 正常完成时触发（cancel/interrupt/fail 不消耗 goal 配额）
    if a.goalCheckpoint != nil && turnStatus == TurnStatusCompleted {
        sessionUUID, _ := uuid.Parse(sessionID)
        turnTokenUsage := a.collectTurnTokenUsage()  // 本 turn 主 Agent 累计 token
        if err := a.goalCheckpoint.Evaluate(ctx, sessionUUID, turnStatus, turnTokenUsage); err != nil {
            a.logger.Error("goal checkpoint error", "error", err)
            // 不阻塞 turn 完成，只记日志
        }
    }

    return finalErr
}
```

### 4.5 rtc-queue 扩展

需要在 `server/pkg/turn-agent/types.go` 添加新的 WorkKind：

```go
const (
    WorkKindSubmit      WorkKind = "submit"
    WorkKindResume      WorkKind = "resume"        // 现有: RTC 结果提交后恢复
    WorkKindGoalResume  WorkKind = "goal_resume"   // 新增: goal 未达成触发
)
```

Worker 在处理 `WorkKindGoalResume` 时与 `WorkKindSubmit` 走同一条 `Process()` 路径 —— GenInput 加载消息时会看到上一轮持久化的 feedback message（system role + `ContentTypeGoalFeedback`），AI 自然知道要继续工作。

### 4.6 与 summarizeMW 的配合

feedback 消息使用 system role + `ContentTypeGoalFeedback` 类型，summarizeMW 在压缩时可以：

- 识别这类消息，保留最新一条，丢弃旧的（避免 N 轮 feedback 累积污染上下文）
- 或者把多条 feedback 合并成一条"progress summary"

> 这是 Phase 6（上下文管理）的优化项，Phase 3 先按默认 summarizeMW 行为跑通。

### 4.7 Turn 取消与 Goal 的交互

| 场景 | turn 状态 | goal 状态 | 行为 |
|------|----------|----------|------|
| AI 正常工作完成 | completed | active | Goal Checkpoint 评判 |
| 用户点 Stop（cancel turn） | cancelled | active | Goal 不消耗轮次，等待下一次用户消息 |
| AI 报错 | failed | active | 同上，不消耗轮次 |
| Agent 调用 `cancel_goal` tool | 当前 turn 继续运行 | cancelled | tool 内推送 `goal.ended`；当前 turn 完成后 Checkpoint 看到 cancelled 状态即跳过 |
| Agent 调用 `create_goal` tool（已有 active goal） | 当前 turn 继续 | 新 goal 创建失败 | tool 返回错误提示，Agent 应告知用户"已有一个 goal 在跑，要先取消吗？" |

**注意**：Agent 调用 `cancel_goal` 后，当前 turn 仍会继续运行到自然结束（不会立即中断）。Agent 的系统提示应引导它在调完 `cancel_goal` 后**立即结束当前 turn**（不再做新工作），避免用户感知延迟。

### 4.8 Agent Tools for Goal

Agent 通过 tool 调用驱动 goal 的创建与取消。**前端没有直接的 goal RPC** —— 所有 goal 生命周期转换由 Agent tool 触发。

#### 4.8.1 `create_goal` tool

```go
// server/internal/agent/tools_goal.go

createGoalTool, err := utils.InferTool("create_goal",
    `Create a goal for sustained, multi-turn work.

You MUST follow this workflow:
1. Investigate: Run relevant tools (test runner, linter, code search) to understand current state
2. Refine: Transform the user's vague intent into a concrete, measurable completion condition (SMART)
3. Propose: Present the refined condition to the user and wait for confirmation
4. Create: ONLY after user confirms, call this tool with the final condition

Do NOT call this tool for one-shot tasks (e.g., "write a function", "explain this code").
Do NOT call this tool if there is already an active goal.`,
    func(ctx context.Context, input *struct {
        Condition string `json:"condition" jsonschema_description:"The refined, concrete, measurable completion condition. Must be specific enough for an independent Judge to verify."`
    }) (string, error) {
        sessionID := getSessionIDFromContext(ctx)

        // 1. 检查是否已有 active goal
        existing, _ := deps.GoalRepo.FindActive(ctx, sessionID)
        if existing != nil {
            return "There is already an active goal. Ask the user whether to cancel it first.", nil
        }

        // 2. 创建新 goal 行
        goal := &model.Goal{
            ID:         uuid.New(),
            SessionID:  sessionID,
            Condition:  input.Condition,
            Status:     model.GoalStatusActive,
            MaxTurns:   deps.Config.DefaultGoalMaxTurns,
            CreatedAt:  time.Now(),
        }
        if err := deps.GoalRepo.Create(ctx, goal); err != nil {
            return "", fmt.Errorf("create goal: %w", err)
        }

        // 3. 加载 session（用于事件推送）
        session, err := deps.SessionRepo.GetByID(ctx, sessionID)
        if err != nil {
            return "", fmt.Errorf("load session: %w", err)
        }

        // 4. 推送 goal.created 事件
        _ = publishGoalEvent(ctx, session, "goal.created", goal, deps)

        return fmt.Sprintf("Goal created: %q. I'll keep working until this is achieved (max %d turns).",
            goal.Condition, goal.MaxTurns), nil
    })
```

#### 4.8.2 `cancel_goal` tool

```go
cancelGoalTool, err := utils.InferTool("cancel_goal",
    `Cancel the current active goal.

Call this when the user indicates they want to stop working on the current goal. Examples:
- "停下来" / "停下" / "stop"
- "别做了" / "算了" / "不做了"
- "取消 goal" / "这个 goal 算了"
- "I'm done with this goal"

Do NOT call this just because a subtask failed — only when the user explicitly wants to abandon the goal entirely.

IMPORTANT: After calling this tool, end your turn immediately. Do not do more work in this turn.`,
    func(ctx context.Context, input *struct{}) (string, error) {
        sessionID := getSessionIDFromContext(ctx)

        goal, err := deps.GoalRepo.FindActive(ctx, sessionID)
        if err != nil {
            return "", fmt.Errorf("find active goal: %w", err)
        }
        if goal == nil {
            return "No active goal to cancel.", nil
        }

        now := time.Now()
        if err := deps.GoalRepo.Update(ctx, goal.ID, map[string]any{
            "status":       model.GoalStatusCancelled,
            "completed_at": now,
        }); err != nil {
            return "", fmt.Errorf("cancel goal: %w", err)
        }
        goal.Status = model.GoalStatusCancelled
        goal.CompletedAt = &now

        session, err := deps.SessionRepo.GetByID(ctx, sessionID)
        if err != nil {
            return "", fmt.Errorf("load session: %w", err)
        }
        _ = publishGoalEvent(ctx, session, "goal.ended", goal, deps)

        return fmt.Sprintf("Goal cancelled: %q", goal.Condition), nil
    })
```

#### 4.8.3 Tool 注册

在 [data.go](server/internal/agent/data.go) 的 `CreateTools` 中，把两个 tool 加入到 agent 的工具列表：

```go
tools := []tool.BaseTool{
    // ... 现有工具 (ls, read_file, write_file, etc.) ...
    createGoalTool,
    cancelGoalTool,
}
```

#### 4.8.4 Agent 系统提示词

在 [agent.go](server/internal/agent/agent.go) 的 `SystemPrompt` 中加入 goal 引导：

> When the user's message starts with `/goal` (injected by the system as a goal-creation intent), follow the goal creation workflow: (1) investigate the request using available tools, (2) refine the vague intent into a SMART condition, (3) propose to the user and wait for confirmation, (4) call `create_goal` after confirmation.
>
> When the user signals they want to stop (e.g., "stop", "cancel", "停下", "算了"), call `cancel_goal` and end your turn immediately. Don't use goals for one-shot questions or single-step tasks.

#### 4.8.5 完整交互示例

```text
User: /goal 帮我把所有单元测试都通过
      ↓ (前端 send message RPC，内容: "/goal 帮我把所有单元测试都通过")
      ↓ (后端解析 /goal 前缀，注入 goal 提示词)
Agent: [goal 提示词生效] 我先调研下测试现状。
      (调用 test runner) 目前 42 个测试中有 7 个失败。
      我建议把 goal 定为："所有 42 个单元测试通过，且不引入新的失败"。你同意吗？
User: 同意
Agent: (调用 create_goal tool, condition="所有 42 个单元测试通过，无新增失败")
      [goal.created 事件推送, 前端状态栏出现 🎯]
      好的，开始工作。首先看 TestX 为什么失败……
      (调用 edit_file 修代码 → 调用 test runner → 没全过 → turn 结束)
      [Goal Checkpoint: completed_turns=1, Judge 判未达成 → feedback 入队 → 下一轮]
      (turn 3: 看 TestY → 修 → 跑 → ...)
      ...
User: 行了，停下吧，剩下的我明天再弄
Agent: (调用 cancel_goal tool，立即结束 turn)
      [goal.ended 事件推送, 前端状态栏清除]
      好的，已取消 goal。当前进度：42 个测试里还有 2 个失败（TestZ, TestW）。
```

---

## 5. Goal 提示词注入

### 5.1 设计思路

前端 `/goal <目标>` 通过 send message RPC 发送，内容以 `/goal ` 开头。后端在消息进入 turn 处理前**解析前缀**，识别 goal 创建意图，向 turn 上下文注入一段 **goal 提示词**，引导 Agent 走"调研 → 润色 → 提议 → 创建"流程。

> 为什么不直接在前端润色？因为 Agent 能调用工具（跑测试、看代码）获得客观信息，润色出的 SMART 条件才真实可判定。

### 5.2 注入实现

```go
// server/internal/agent/goal_prompt.go

const goalPrefix = "/goal "

// GoalPromptTemplate 注入到 turn 上下文的 system 提示词
const GoalPromptTemplate = `[SYSTEM: Goal Creation Intent]
The user wants to set a goal. Their original input: %q

You MUST follow this workflow:
1. Investigate: Use available tools to understand the current state. For code goals: run tests, check linters, inspect relevant files. For other goals: gather objective evidence.
2. Refine: Transform the user's vague intent into a concrete, measurable completion condition (SMART: Specific, Measurable, Achievable, Relevant, Time-bound).
3. Propose: Present the refined condition to the user in plain language and explicitly ask for confirmation.
4. Create: ONLY after the user confirms (or modifies and confirms), call the create_goal tool with the final condition.

Do NOT skip the investigation step. Do NOT call create_goal before getting user confirmation.
[END SYSTEM]`

// InjectGoalPrompt 检测消息前缀，返回是否 goal 意图 + 注入后的 system messages
func InjectGoalPrompt(userContent string) (isGoal bool, injectedSystem string) {
    if !strings.HasPrefix(userContent, goalPrefix) {
        return false, ""
    }
    rawGoal := strings.TrimSpace(strings.TrimPrefix(userContent, goalPrefix))
    return true, fmt.Sprintf(GoalPromptTemplate, rawGoal)
}
```

### 5.3 集成点

在 turn 入口（消息 ingest 阶段，GenInput 之前或之中）：

```go
// 在构建 turn messages 时
if isGoal, injected := InjectGoalPrompt(userContent); isGoal {
    messages = append(messages, schema.SystemMessage(injected))
    // 可选：把原始 "/goal xxx" 也作为 user message，让 Agent 看到用户原话
    messages = append(messages, schema.UserMessage(rawGoal))
} else {
    messages = append(messages, schema.UserMessage(userContent))
}
```

> 注意：前端发送时是否保留 `/goal ` 前缀由前端决定。后端 ingest 层兜底解析前缀；前端也可以选择剥离前缀，通过 metadata（如 `intent: 'goal_create'`）传递意图。两种机制可以共存。

---

## 6. Judge 设计

### 6.1 接口定义

```go
// server/internal/agent/judge.go

type Judge struct {
    model  model.ChatModel  // 独立配置的轻量模型（Haiku）
    logger turnagent.Logger
}

func newJudge(model model.ChatModel, logger turnagent.Logger) *Judge {
    return &Judge{
        model:  model,
        logger: logger,
    }
}
```

### 6.2 评判方法

```go
// Evaluate 判断 goal 是否达成
// 参数: condition, messages (最近对话), toolNames (Agent 可用工具列表)
// 返回: achieved, reason, tokenUsage, error
func (j *Judge) Evaluate(ctx context.Context, condition string, messages []*schema.Message, toolNames []string) (bool, string, int, error) {
    // 1. 构造评判 prompt
    prompt := j.buildJudgePrompt(condition, messages, toolNames)

    // 2. 调用 Judge 模型
    resp, err := j.model.Generate(ctx, []*schema.Message{
        schema.SystemMessage("You are a goal evaluator. Determine if the given condition has been achieved based on the conversation history."),
        schema.UserMessage(prompt),
    })
    if err != nil {
        return false, "", 0, fmt.Errorf("judge model error: %w", err)
    }

    // 3. 解析响应
    achieved, reason := j.parseResponse(resp.Content)

    // 4. 计算 token 使用量
    tokenUsage := 0
    if resp.ResponseMeta != nil && resp.ResponseMeta.Usage != nil {
        tokenUsage = resp.ResponseMeta.Usage.TotalTokens
    }

    return achieved, reason, tokenUsage, nil
}

// buildJudgePrompt 构造评判 prompt
// toolNames 让 Judge 知道 AI 实际可用的工具，避免评判"为什么不调 X 工具"而 AI 根本没有 X
func (j *Judge) buildJudgePrompt(condition string, messages []*schema.Message, toolNames []string) string {
    // 按 token 数截断（避免固定条数截断错过关键上下文）
    // 粗略估计: 1 token ≈ 4 字符, 保留最近 8k token ≈ 32k 字符
    const maxChars = 32000
    var conversation strings.Builder
    totalChars := 0
    // 从后往前追加，保证最新内容被保留
    for i := len(messages) - 1; i >= 0; i-- {
        msg := messages[i]
        line := fmt.Sprintf("[%s]: %s\n", string(msg.Role), msg.Content)
        if totalChars+len(line) > maxChars {
            break
        }
        conversation.WriteString(line)
        totalChars += len(line)
    }

    toolsList := "(none)"
    if len(toolNames) > 0 {
        toolsList = strings.Join(toolNames, ", ")
    }

    return fmt.Sprintf(`You are a goal evaluator. Determine whether the goal condition has been achieved based on the conversation history.

Goal Condition:
%s

Available Tools (AI can only use these):
%s

Conversation History (newest first):
%s

Answer with a single JSON object (no markdown fences, no prose):
{"achieved": true/false, "reason": "brief explanation citing specific evidence from the conversation"}
`, condition, toolsList, conversation.String())
}

// parseResponse 解析 Judge 模型的响应
// 鲁棒处理：剥离 markdown fence、regex fallback、空响应处理
func (j *Judge) parseResponse(content string) (bool, string) {
    content = strings.TrimSpace(content)
    if content == "" {
        return false, "Judge returned empty response"
    }

    // 1. 剥离 ```json ... ``` 或 ``` ... ``` 包裹
    if strings.HasPrefix(content, "```") {
        if end := strings.Index(content, "```", 3); end > 0 {
            inner := content[3:end]
            if strings.HasPrefix(inner, "json") {
                inner = inner[4:]
            }
            content = strings.TrimSpace(inner)
        }
    }

    // 2. 尝试直接 JSON 解析
    var result struct {
        Achieved bool   `json:"achieved"`
        Reason   string `json:"reason"`
    }
    if err := json.Unmarshal([]byte(content), &result); err == nil {
        return result.Achieved, result.Reason
    }

    // 3. Regex fallback: 找 {"achieved": ..., "reason": ...} 片段
    re := regexp.MustCompile(`"achieved"\s*:\s*(true|false)`)
    if m := re.FindStringSubmatch(content); len(m) == 2 {
        achieved := m[1] == "true"
        reasonRe := regexp.MustCompile(`"reason"\s*:\s*"([^"]*)"`)
        reason := "parsed via regex fallback"
        if rm := reasonRe.FindStringSubmatch(content); len(rm) == 2 {
            reason = rm[1]
        }
        return achieved, reason
    }

    // 4. 完全无法解析：默认未达成（不轻言放弃）
    return false, fmt.Sprintf("unparseable judge response: %s", truncate(content, 200))
}
```

### 6.3 Judge 模型选择与配置

| 模型 | 用途 | 原因 |
| --- | --- | --- |
| Claude Haiku | 默认 Judge 模型 | 成本低，速度快，足够准确 |
| Claude Sonnet | 可选升级 | 复杂 goal 时使用 |

**独立配置**：系统目前只有单一 `ChatModel`（主模型），需要在 `Dependencies` 增加独立的 `JudgeModel` 字段：

```go
// server/internal/usecase/dependencies.go
type Dependencies struct {
    // ... 现有字段 ...
    ChatModel  model.ChatModel  // 主模型 (Sonnet/Opus)
    JudgeModel model.ChatModel  // Judge 专用 (Haiku)
}
```

在 [llm.go](server/internal/server/llm.go) 增加 Judge 模型构造函数：

```go
// server/internal/server/llm.go
func newJudgeChatModel(ctx context.Context, cfg LLMConfig) (model.ChatModel, error) {
    // 复用 provider 配置，但强制使用 Haiku
    // 具体实现按 provider 类型分支
}
```

在 wire 注入时构造第二个实例：

```go
// server/cmd/wire_gen.go
judgeChatModel, err := newJudgeChatModel(ctx, cfg.LLM)
// ...
deps := &usecase.Dependencies{
    ChatModel:  mainChatModel,
    JudgeModel: judgeChatModel,
    // ...
}
```

> 重要：不要把主 ChatModel 传给 Judge。否则每次评判都按主模型计费，成本失控。

---

## 7. RPC 接口设计

### 7.1 RPC 边界说明

**创建和取消 goal 不通过独立 RPC**，而是通过 Agent tool 在服务端内部完成。前端与 goal 的交互：

- **创建 goal**：前端 `/goal <目标>` → 复用 send message RPC（后端解析前缀，注入 goal 提示词）→ Agent 接管 → 调 `create_goal` tool
- **取消 goal**：用户自然语言 → Agent 识别意图 → 调 `cancel_goal` tool
- **查询 goal**：前端调 `get_goal` / `list_goals` RPC（见下文）

**删除的 RPC**（v1 版本定义，v2 移除）：

- ~~`v1.session.set_goal`~~ — 创建只走 Agent tool
- ~~`v1.session.clear_goal`~~ — 取消只走 Agent tool

**保留的 RPC**：

- `v1.session.get_goal` — 查询当前 active goal
- `v1.session.list_goals` — 查询 goal 历史

仍然定义成独立 RPC（查询类）是为了：

- 未来前端做历史面板时直接调用
- 复用鉴权、归属校验等逻辑
- 保持 HTTP/WebSocket 边界清晰，便于调试

### 7.2 查询 Goal

**方法**: `v1.session.get_goal`

**请求**：

```json
{
    "session_id": "uuid"
}
```

**响应**（无 active goal 时 `goal` 为 null）：

```json
{
    "goal": {
        "id": "goal-uuid",
        "session_id": "session-uuid",
        "condition": "所有测试通过",
        "status": "active",
        "completed_turns": 5,
        "token_usage": 12345,
        "max_turns": 50,
        "last_judge_reason": "还差 2 个单元测试未通过: TestX, TestY",
        "created_at": "2026-09-07T10:00:00Z"
    }
}
```

### 7.3 列出 Goal 历史

**方法**: `v1.session.list_goals`

**请求**：

```json
{
    "session_id": "uuid",
    "limit": 20,
    "cursor": "optional-pagination-cursor"
}
```

**响应**：

```json
{
    "goals": [
        {
            "id": "goal-uuid-1",
            "session_id": "session-uuid",
            "condition": "所有测试通过",
            "status": "completed",
            "completed_turns": 3,
            "token_usage": 8000,
            "max_turns": 50,
            "created_at": "2026-09-07T10:00:00Z",
            "completed_at": "2026-09-07T10:05:00Z",
            "last_judge_reason": "全部 42 个测试通过"
        },
        {
            "id": "goal-uuid-2",
            "condition": "性能优化到 100ms 内",
            "status": "exhausted",
            "completed_turns": 50,
            "token_usage": 120000,
            "max_turns": 50,
            "created_at": "2026-09-07T11:00:00Z",
            "completed_at": "2026-09-07T11:45:00Z",
            "last_judge_reason": "P95 仍为 180ms，未能达到 100ms 目标"
        }
    ],
    "next_cursor": "optional-next-page-cursor"
}
```

**Handler 要点**：

- 复用 `loadOwnedSession` 做归属校验
- 调用 `GoalRepo.ListBySession(ctx, sessionID, limit, cursor)`，按 `created_at DESC` 排序
- 返回分页结果，支持 cursor 翻页

---

## 8. 前端交互

前端**只有 `/goal <自然语言目标>` 一个命令**（无子命令）。取消、查询、历史等功能通过 Agent 对话或 UI 控件完成。

### 8.1 `/goal` slash 命令

`/goal <目标>` 是一个**前端 UX 入口**，实际走 send message RPC。

**前端职责**：

- 识别输入框中以 `/goal ` 开头的消息
- 通过 send message RPC 发送（内容可保留 `/goal ` 前缀，后端负责解析；或前端剥离前缀并通过 metadata 标记 `intent: 'goal_create'`）
- 订阅 `goal.*` 事件，更新状态栏和发通知
- 提供 goal 状态栏（显示 active goal 进度）

### 8.2 Goal 事件订阅

前端订阅用户 topic channel 后，按 event name 过滤 goal 事件：

```typescript
// web-components/src/subscriptions/goal-events.ts

topicChannel.on('goal.created', (goal: Goal) => {
    goalStore.setActive(goal);              // 状态栏显示新 goal
    // 不需要 toast —— Agent 在对话中已经告知用户 goal 已创建
});

topicChannel.on('goal.updated', (goal: Goal) => {
    goalStore.setActive(goal);              // 进度更新（turns / token_usage / last_reason）
});

topicChannel.on('goal.ended', (goal: Goal) => {
    goalStore.clearActive();                // 清除状态栏
    switch (goal.status) {
        case 'completed':
            showToast('🎉 Goal achieved!');
            break;
        case 'exhausted':
            showToast('⚠️ Goal reached max turns without achieving. Try a more specific condition?');
            break;
        case 'cancelled':
            // 静默 —— Agent 在对话中已经告知用户 goal 已取消
            break;
    }
});
```

> 注：goal 事件和 session 事件都在用户的 topic channel 上，前端按 event name 路由到不同 handler。
>
> `goal.created` 和 `cancelled` 不发 toast —— Agent 在对话流中已经通过自然语言告知用户。只有 `completed` 和 `exhausted` 是后端自动触发的，需要 toast 提示用户。

### 8.3 状态栏

```typescript
// web-components/src/components/StatusBar.tsx

function GoalIndicator({ goal }: { goal: Goal | null }) {
    if (!goal || goal.status !== 'active') return null;

    return (
        <div className="goal-indicator">
            <span className="icon">🎯</span>
            <span className="condition">{goal.condition}</span>
            <span className="turns">Turn {goal.completed_turns}/{goal.max_turns}</span>
            <span className="tokens">🔥 {goal.token_usage.toLocaleString()} tokens</span>
            {goal.last_judge_reason && (
                <span className="reason" title={goal.last_judge_reason}>💬</span>
            )}
        </div>
    );
}
```

### 8.4 Goal 历史面板

> **v2 决策**：后端提供 `list_goals` RPC，前端暂不做 UI。未来实现时参考以下设计。

```typescript
// web-components/src/components/GoalHistoryPanel.tsx (暂不实现)

async function openGoalHistoryPanel(sessionId: string) {
    const { goals } = await rpcClient.call('v1.session.list_goals', {
        session_id: sessionId,
        limit: 20,
    });

    renderPanel(goals.map(g => ({
        condition: g.condition,
        statusBadge: statusBadge(g.status),  // ✅ completed / ❌ exhausted / ⏹ cancelled / 🔄 active
        turns: `${g.completed_turns}/${g.max_turns}`,
        tokens: g.token_usage,
        duration: g.completed_at ? formatDuration(g.created_at, g.completed_at) : 'in progress',
        reason: g.last_judge_reason,
    })));
}
```

---

## 9. 关键规则

### 9.1 生命周期

Goal 有四种状态，分三类：

- **进行中**：`active`
- **终态（成功）**：`completed` —— Judge 判定达成
- **终态（中止）**
  - `cancelled` —— Agent 调用 `cancel_goal` tool（识别到用户意图如"停下来"、"取消目标"）
  - `exhausted` —— 达到 `max_turns` 仍未达成

状态流转：

```text
     Agent 调用 create_goal tool
              ↓
         [ active ] ── Agent 调用 cancel_goal ─→ [ cancelled ]
              ↓
        Goal Checkpoint (每轮 turn 完成后)
              ↓
       ┌──────┴──────┐
       ↓              ↓
  [ completed ]  [ exhausted ]
```

**关键语义**：

- `active`：每次 turn 正常完成时触发 Goal Checkpoint 评判
- `completed`：推送 `goal.ended`，前端显示 🎉
- `cancelled`：Agent tool 即时生效；当前 turn 完成后 Checkpoint 看到 cancelled 状态即跳过
- `exhausted`：推送 `goal.ended`，前端显示 ⚠️ 引导用户优化条件
- **终态不可逆**：goal 一旦进入 completed/cancelled/exhausted，不能改回 active。需要新一轮就让 Agent 调用 `create_goal` 创建新 goal

### 9.2 事件模型

Goal 独立于 Session 后，有自己的事件生命周期，**不再通过 `session.updated` 承载**：

- **`goal.created`**：新 goal 创建时推送，payload 是完整 goal 对象
- **`goal.updated`**：每轮 turn 完成后（completed_turns / token_usage / last_judge_reason 变化）推送，payload 是完整 goal 对象
- **`goal.ended`**：goal 进入终态时推送（completed / cancelled / exhausted）。payload 是完整 goal 对象

事件发到用户的 topic channel（与 session 事件同 channel），前端按 event name 路由到 goal store。

### 9.3 约束条件

1. **单会话单 active goal**：一个会话同时只能有一个 `active` goal；Agent 调用 `create_goal` 时若已有 active goal，tool 返回错误提示让 Agent 询问用户是否先取消
2. **max_turns 创建时固化**：goal 行的 `max_turns` 字段保存创建时的配置值，避免未来改配置影响历史 goal
3. **仅 completed turn 计入轮次**：cancel / interrupt / fail 不消耗 goal 配额
4. **主 Agent turn 完成即计数**：`completed_turns += 1` 在 turn 完成时立即执行，**不论 Judge 是否成功评判**；Judge 失败也计入轮次，max_turns 兜底避免无限挂起
5. **Judge 失败容错**：Judge 模型错误时，**也 re-queue**，等下一轮再判（与"未达成"路径一致），记录日志
6. **Token 追踪**：`token_usage` 记录 goal 生命周期内 session 的累计 token（主 Agent + Judge 合计），每轮 turn 完成后累加
7. **Feedback 持久化**：feedback 消息写入 DB（system role + `ContentTypeGoalFeedback`），不依赖内存 state
8. **历史不可变**：终态 goal 行保留，不删除不修改，供 `list_goals` 查询
9. **`completed_at` 字段**：goal 进入任意终态的时间戳（completed / cancelled / exhausted 都填）

### 9.4 边界情况

- **Goal 已存在时 Agent 调用 create_goal** → tool 返回 `"There is already an active goal. Ask the user whether to cancel it first."`，Agent 据此询问用户是否用 `cancel_goal` 先取消
- **Judge 模型 API 错误** → 计入轮次 + re-queue，下一轮再判（不轻言放弃）；max_turns 兜底
- **达到最大轮次** → goal 进入 `exhausted` 终态，推送 `goal.ended`，前端显示 ⚠️ 引导
- **Session 关闭时 goal 仍活跃** → Goal 状态保留在 DB；**不会自动继续**，需要用户重新打开 session 并发消息才会触发下一轮评判
- **用户 cancel 正在跑的 turn** → turn 状态为 cancelled，Goal Checkpoint 跳过评判，goal 保持 active 等待下次
- **Agent 调用 cancel_goal 时 turn 还在跑** → goal 立即进入 cancelled；当前 turn 继续到自然结束，Checkpoint 看到 cancelled 后跳过
- **Judge 反复判定 "未达成" 但 AI 实际已无事可做** → 由 max_turns 兜底进入 exhausted；产品上建议 Agent 在润色阶段写 SMART 条件
- **历史 goal 查询** → 通过 `list_goals` RPC，按 created_at DESC 分页

---

## 10. 实现计划

### 10.1 Phase 1: 数据模型

- [ ] 创建 `server/internal/model/goal.go`，定义 `Goal` 独立表
- [ ] 创建 `server/internal/repo/goal_repo.go`，实现 `GoalRepo` 接口（含 `FindActive`、`ListBySession`、`Create`、`Update`）
- [ ] 更新 `protocol` 定义（OpenAPI spec）：新增 `get_goal` / `list_goals` 方法（v2 移除 `set_goal` / `clear_goal`）
- [ ] 重新生成 protocol 代码

### 10.2 Phase 2: Judge 实现

- [ ] 创建 `judge.go`
- [ ] 实现 `Evaluate` 方法（含 toolNames 参数）
- [ ] 配置 Judge 模型（Haiku）
- [ ] Dependencies 增加 JudgeModel 字段，wire 注入

### 10.3 Phase 3: Goal 提示词注入 + Checkpoint + rtc-queue 集成

- [ ] 创建 `server/internal/agent/goal_prompt.go`，实现 `/goal` 前缀解析与提示词注入
- [ ] 在 turn 入口（GenInput 阶段）集成 `InjectGoalPrompt`
- [ ] 在 `server/pkg/turn-agent/types.go` 新增 `WorkKindGoalResume`
- [ ] 创建 `server/internal/agent/goal_checkpoint.go`，实现 `Evaluate` 方法
- [ ] 在 `agent_process.go` 的 `Process()` 末尾调用 Goal Checkpoint
- [ ] feedback 消息使用 system role + `ContentTypeGoalFeedback` 持久化到 DB
- [ ] re-queue 逻辑：调用 `queue.Publish` 发布 `goal_resume` work item
- [ ] 验证：rtc-queue 是否已保证同 session 串行；否则需加 session 路由

### 10.4 Phase 4: Agent Tool + RPC 接口

- [ ] 创建 `tools_goal.go`，实现 `create_goal` / `cancel_goal` tool
- [ ] 在 `data.go` 注册两个 tool
- [ ] 更新 Agent 系统提示词，加入 goal 引导
- [ ] 创建 `action.goal.go`，实现 `GetGoal` / `ListGoals`（v2 移除 `SetGoal` / `ClearGoal`）
- [ ] 注册 RPC 路由
- [ ] 推送 `goal.created` / `goal.updated` / `goal.ended` 事件到 user topic channel
- [ ] 新增 `updates.EntityGoal` entity 类型

### 10.5 Phase 5: 前端集成

- [ ] `/goal <目标>` 命令解析 —— 识别前缀，通过 send message RPC 发送
- [ ] 订阅 `goal.created` / `goal.updated` / `goal.ended` 事件
- [ ] 状态栏显示（condition + completed_turns/max_turns + token_usage + last_judge_reason）
- [ ] 完成通知（🎉 completed / ⚠️ exhausted / 静默 cancelled）
- [ ] Goal 历史面板（**Phase 5 暂不实现**，后端 `list_goals` RPC 已就位，UI 留到后续版本）

---

## 11. 测试计划

### 11.1 单元测试

- **Judge.Evaluate — 目标达成**：返回 `achieved=true` + reason
- **Judge.Evaluate — 目标未达成**：返回 `achieved=false` + reason
- **Judge.Evaluate — toolNames 传入**：Judge prompt 中包含工具列表
- **Judge.parseResponse — JSON 响应**：正确解析
- **Judge.parseResponse — markdown fence 包裹**：剥离后正确解析
- **Judge.parseResponse — 完全无法解析**：返回 `achieved=false` + unparseable 提示
- **GoalPrompt.InjectGoalPrompt — 普通消息**：返回 `isGoal=false`
- **GoalPrompt.InjectGoalPrompt — /goal 前缀**：返回 `isGoal=true` + 注入提示词
- **GoalCheckpoint.Evaluate — goal 非 active**：直接返回，不调用 Judge
- **GoalCheckpoint.Evaluate — turn 非 completed（cancel/fail）**：跳过评判，不消耗轮次
- **GoalCheckpoint.Evaluate — 达到最大轮次**：goal 进入 `exhausted` 终态，推送 `goal.ended`，不 re-queue
- **GoalCheckpoint.Evaluate — Judge 报错**：计入轮次 + re-queue，max_turns 兜底
- **GoalCheckpoint.Evaluate — 未达成**：feedback 消息写入 DB（system role）+ goal_resume work item 发布到 rtc-queue + 推送 `goal.updated`
- **GoalCheckpoint.Evaluate — 达成**：goal 进入 `completed` 终态，推送 `goal.ended`
- **GoalCheckpoint.Evaluate — token 累加**：主 Agent token + Judge token 都计入 goal.token_usage
- **create_goal tool — 已存在 active goal**：返回错误提示，不创建新行
- **cancel_goal tool — 正常取消**：goal 状态变 cancelled，推送 goal.ended
- **cancel_goal tool — 无 active goal**：返回 "No active goal to cancel."

### 11.2 集成测试

- `/goal 所有测试通过` + Agent 调研润色 + 用户确认 + AI 完成工作 → Goal 达成，状态变 `completed`，推送 `goal.ended`
- `/goal 所有测试通过` + AI 未完成 → Goal 未达成，rtc-queue 收到 goal_resume，新 turn 看到 feedback 继续工作
- 达到 max_turns 上限 → goal 进入 `exhausted`，推送 `goal.ended`，前端显示 ⚠️
- 用户在 AI 工作时 cancel turn（Stop 按钮）→ goal 保持 active，等待下次
- 用户自然语言 "停下" → Agent 调 cancel_goal → goal 状态变 `cancelled`，推送 `goal.ended`
- `v1.session.list_goals` → 返回该 session 所有 goal（含 completed/cancelled/exhausted 历史），按 created_at DESC 排序
- `v1.session.get_goal` → 返回当前 active goal；无 active goal 时返回 null

---

## 12. 风险与缓解

- **Judge 误判（达成判未达成）**：AI 继续工作但无法达成 → 由 max_turns 兜底进入 `exhausted`；产品上引导 Agent 润色 SMART 条件
- **Judge 误判（未达成判达成）**：AI 提前停止，用户失望 → 允许用户取消后重设 goal；未来可考虑"Judge 两次确认"机制
- **无限循环（AI 反复做同样的事）**：每轮 feedback 相同 → summarizeMW 压缩 + 用户自然语言取消 + max_turns 兜底
- **Token 消耗过高**：跨 turn 累积 → `Goal.TokenUsage` 追踪（主 Agent + Judge 合计）；前端状态栏显示；未来可加 token 预算上限
- **rtc-queue 同 session 并发**：goal_resume 与用户新消息被不同 worker 同时认领 → 验证 queue 是否 session-scoped，否则需加分布式锁
- **Feedback 累积污染上下文**：N 轮 feedback 堆在消息历史 → system role + `ContentTypeGoalFeedback` 类型标识，summarizeMW 特殊处理（Phase 6 优化项）
- **Session 关闭后的"自动继续"误解**：文档需明确"保留状态，下次发消息时继续"，不是后台自动运行
- **Goal 表膨胀**：长期运行的 session 累积大量历史 goal → 建议加 TTL（如 90 天后归档），或在 list_goals 默认只查近期
- **Goal 提示词被 Agent 忽略**：Agent 可能没按"调研 → 润色 → 提议 → 创建"流程走 → 系统提示词强化 + 后续可考虑在 goal_prompt 内强制 tool 调用顺序

---

## 13. 未来扩展

1. **多 active goal 支持**：允许同时设定多个 active goal（目前限制为单 active）
2. **目标优先级**：多个 active goal 时按优先级评判
3. **Goal 暂停/恢复**：`/goal pause` / `/goal resume`，新增 `paused` 状态（独立命令）
4. **自定义 Judge**：允许用户指定评判模型
5. **Goal 模板**：预定义常用 goal 模板
6. **跨 Session Goal**：Goal 解耦 Session 后，可支持一个 Goal 跨多个 Session 执行
7. **Goal 归档**：长期历史 goal 自动归档（如 90 天后），减轻 goals 表压力
8. **前端 Goal 历史面板**：基于已实现的 `list_goals` RPC 做 UI
9. **下一轮 token 预估**：在状态栏显示"下一轮预估 X tokens，剩余 Y tokens 触发自动压缩"，提前告警

---

## 附录 A: 与 /loop 命令的对比

- **循环控制**：`/goal` 目标驱动；`/loop` 轮次驱动 / 时间驱动
- **停止条件**：`/goal` Judge 判定；`/loop` 轮次完成 / 时间到达
- **状态管理**：`/goal` 独立 goals 表；`/loop` 数据库任务表
- **底层机制**：两者都通过 rtc-queue 驱动 turn 循环，差异仅在"是否 re-queue"的判定逻辑
- **适用场景**：`/goal` 模糊目标（如"所有测试通过"）；`/loop` 明确任务（如"循环 5 轮"）

---

## 附录 B: v1 → v2 变更清单

| 主题 | v1 | v2 |
|------|----|----|
| 创建路径 | Agent tool + RPC `set_goal` | 仅 Agent tool |
| 取消路径 | Agent tool + RPC `clear_goal` | 仅 Agent tool |
| 前端 `/goal` | §1.1 说不存在 / §7 说存在 | 存在，作为 UX 入口 |
| `/goal` 子命令 | 模糊 | 明确无子命令 |
| 时间字段名 | `ended_at` | `completed_at` |
| status 字段说明 | §3.5 漏 `exhausted` | 4 种状态全列 |
| Judge 失败 | 不 re-queue，挂起等用户发消息 | 也 re-queue，max_turns 兜底 |
| 轮次计数 | Judge 判未达成才 +1 | 主 Agent turn 完成即 +1 |
| token_usage | Judge + 主 Agent（代码只追踪 Judge） | 整个 goal 生命周期 session 累计 token |
| Feedback role | user | system |
| Judge 参数 | `toolNames` 漏传 | 必须传 |
| Session 获取 | Evaluate 参数不足 | Evaluate 内加载 Session |
| 架构图 Session.goal 字段 | 画了（与 §3 冲突） | 删除 |
| 前端历史面板 | 做 | 暂不做（后端接口保留） |

---

## 附录 C: 相关文档

- [13-commands.md](../13-commands.md) - 命令系统总览
- [11-context-management.md](../11-context-management.md) - 上下文管理
- [Eino Middleware Reference](../../eino-agent/reference/middleware.md)

---

**文档结束**
