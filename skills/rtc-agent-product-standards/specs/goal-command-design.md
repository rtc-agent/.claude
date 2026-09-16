# /goal 命令设计文档

**版本**: 1.3  
**日期**: 2026-09-07  
**状态**: 评审后修订（Agent 通过 tool 创建/取消 goal，前端无 slash 命令，4 状态生命周期，独立 goal.* 事件）

---

## 1. 概述

### 1.1 功能定位

`/goal` 机制让 AI 持续工作直到某个完成条件满足。核心交互：

- **创建 goal**：Agent 调用 `create_goal` tool。Agent 先把用户的模糊意图**润色**成具体、可测的条件，与用户确认后才创建 —— 因为 Agent 比用户更懂如何把"把代码写好"变成"所有 42 个单元测试通过 + 无回归错误"
- **取消 goal**：Agent 调用 `cancel_goal` tool。当用户说"停下来"、"别做了"、"这个 goal 算了"时，Agent 识别意图并取消
- **完成 goal**：后端 Judge 模型判定达成，自动标记 `completed`
- **耗尽 goal**：后端达到 `max_turns` 仍未达成，自动标记 `exhausted`

前端**没有** `/goal` slash 命令 —— 所有生命周期转换都由 Agent（创建/取消）或后端系统（完成/耗尽）驱动。前端只负责显示状态栏、接收 `goal.*` 事件、展示历史。

### 1.2 核心机制

```text
用户自然语言表达意图（"帮我把所有测试通过"）
   ↓
Agent 识别 goal 意图 → 润色条件 → 提议给用户确认
   ↓
用户确认 → Agent 调用 create_goal tool
   ↓
turn 1: AI 执行工作 → turn 完成
   ↓
Goal Checkpoint: Judge 评判 → 未达成? → 持久化 feedback → rtc-queue 入队 goal_resume
                                                          ↓
turn 2: AI 看到 feedback 继续工作 → turn 完成
   ↓
Goal Checkpoint: 再次评判 → ... 循环直到达成 / 耗尽 / 用户说"停下"→ Agent 调 cancel_goal
```

### 1.3 设计原则

1. **Agent 当 goal 教练**：用户说模糊意图，Agent 负责润色成 SMART 条件。用户不需要懂如何写"好 goal"
2. **自然语言驱动**：用户不说"创建 goal"，而是说"帮我把 X 做了"、"停下来" —— Agent 识别意图并调用对应 tool
3. **turn 边界检查**：Goal 评判发生在 turn 之间，不侵入 turn 内部的 ChatModelAgent / middleware
4. **复用调度基础设施**：通过 rtc-queue 驱动多轮，与用户主动发消息走同一条路径，cancel / 限流 / 监控 / Centrifuge 推送全部免费继承
5. **显式状态管理**：goal 状态持久化到独立 goals 表，feedback 消息持久化到 DB，不依赖内存 state
6. **轻量评判**：使用独立的轻量模型（Haiku）降低成本，与主模型解耦
7. **防护机制**：max_turns 上限（默认 50 turn），防止失控；创建时固化当时的 max_turns 配置

---

## 2. 架构设计

### 2.1 整体架构

**核心思路**：goal 驱动的"持续工作"是 **turn 间** 的编排，不是 turn 内的 middleware 钩子。每轮 turn 走完整的 `rtc-queue → Process() → TurnLoop → ChatModelAgent` 路径；turn 完成后，系统在边界检查 goal 状态，未达成则把 feedback 作为新 work item 重新投回 rtc-queue，触发下一轮 turn。

```
┌─────────────────────────────────────────────────────────────────────┐
│                         Frontend (Web)                               │
│  /goal <condition> → RPC v1.session.set_goal → 状态栏显示 goal      │
└─────────────────────────────────────────────────────────────────────┘
                                ↓
┌─────────────────────────────────────────────────────────────────────┐
│                      Backend (Server)                                │
│                                                                      │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │                    Session Model                              │   │
│  │  goal: { condition, status, completed_turns, token_usage }   │   │
│  └──────────────────────────────────────────────────────────────┘   │
│                                                                      │
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
│  │       ├─ 达到 max turns → 标记, 返回                          │   │
│  │       ├─ 调用 Judge 评判                                      │   │
│  │       │    ├─ achieved → goal.status=completed, publish       │   │
│  │       │    └─ not achieved →                                  │   │
│  │       │         1. 持久化 feedback message 到 DB              │   │
│  │       │         2. 向 rtc-queue 发布 goal_resume work item    │   │
│  │       │         3. worker 下一轮会看到 feedback, 继续工作     │   │
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
| **Goal Storage** | 持久化 goal 状态 | Session 模型的 `goal` 字段 |
| **RPC Handler** | 处理 `/goal` 命令 | `server/internal/handler/rpc/action.goal.go` |
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
    TokenUsage      int        `gorm:"not null;default:0" json:"token_usage"`
    MaxTurns        int        `gorm:"not null;default:50" json:"max_turns"`  // 创建时记录, 避免全局配置变更影响历史 goal
    CreatedAt       time.Time  `gorm:"not null" json:"created_at"`
    EndedAt         *time.Time `json:"ended_at,omitempty"`        // 进入终态的时间
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
    ListBySession(ctx context.Context, sessionID uuid.UUID, limit int) ([]*model.Goal, error)
}
```

### 3.5 字段说明

- **`condition`** (text)：完成条件的自然语言描述
- **`status`** (varchar 20)：`active` / `completed` / `cancelled`
- **`completed_turns`**：已完成并完成评判的 turn 数（每次 Process 正常返回 +1）
- **`token_usage`**：累计 token 消耗（Judge + 主 Agent）
- **`created_at`**：goal 创建时间
- **`completed_at`**：goal 完成时间（仅 status=completed 时有值）
- **`last_judge_reason`**：最近一次 Judge 给出的 reason，供前端展示进度

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
    "completed_turns":  goal.CompletedTurns + 1,
    "last_judge_reason": reason,
    "token_usage":      goal.TokenUsage + judgeTokenUsage,
})

// Goal 达成:
now := time.Now()
err := deps.GoalRepo.Update(ctx, goal.ID, map[string]any{
    "status":       "completed",
    "completed_at": now,
})
```

---

## 4. Goal Checkpoint 设计

### 4.1 设计思路

Goal Checkpoint 不是 middleware，而是 **turn 完成后的编排逻辑**。它挂在 `agent.Process()` 的末尾，与现有的 `prompt-too-long → reactive compact → 重试` 逻辑并列。

**触发条件**：turn 以 `completeTurn` 状态结束（不是 cancel / interrupt / fail）。

**三种结果**：

1. **goal 非 active** → 直接返回，无事发生
2. **goal 达成** → 标记 completed，推送 `session.updated` 事件
3. **goal 未达成** → 持久化 feedback 消息 → 向 rtc-queue 发布 `goal_resume` work item → worker 下一轮会看到 feedback 继续工作

### 4.2 接口定义

```go
// server/internal/agent/goal_checkpoint.go

type goalCheckpoint struct {
    deps          *usecase.Dependencies
    judge         *Judge
    publisher     updates.UpdatePublisher  // 用于推送 goal.* 事件
    queue         queue.Publisher          // rtc-queue, 用于 re-queue
    logger        turnagent.Logger
    defaultMaxTurns int                    // 创建新 goal 时的默认值（从配置读取）
}

func newGoalCheckpoint(
    deps *usecase.Dependencies,
    judge *Judge,
    publisher updates.UpdatePublisher,
    queue queue.Publisher,
    logger turnagent.Logger,
) *goalCheckpoint {
    return &goalCheckpoint{
        deps:      deps,
        judge:     judge,
        publisher: publisher,
        queue:     queue,
        logger:    logger,
        maxTurns:  50,
    }
}
```

### 4.3 核心方法

```go
// Evaluate 在 turn 完成后调用，判断是否需要触发下一轮
// 参数: sessionID, turnID (刚完成的 turn), finalStatus (turn 状态)
func (g *goalCheckpoint) Evaluate(
    ctx context.Context,
    sessionID uuid.UUID,
    finalStatus TurnStatus,
) error {
    // 1. 只在正常完成的 turn 上评判
    //    cancel / interrupt / fail 不算一轮，不消耗 turn 配额
    if finalStatus != TurnStatusCompleted {
        return nil
    }

    // 2. 从 goals 表查当前 session 的 active goal（独立于 Session 模型）
    goal, err := g.deps.GoalRepo.FindActive(ctx, sessionID)
    if err != nil {
        return fmt.Errorf("find active goal: %w", err)
    }
    if goal == nil {
        return nil  // 无活跃 goal
    }

    // 3. 达到最大轮次 → 标记 exhausted，推送事件，不 re-queue
    if goal.CompletedTurns >= goal.MaxTurns {
        now := time.Now()
        if err := g.deps.GoalRepo.Update(ctx, goal.ID, map[string]any{
            "status":    model.GoalStatusExhausted,
            "ended_at":  now,
        }); err != nil {
            return fmt.Errorf("mark goal exhausted: %w", err)
        }
        goal.Status = model.GoalStatusExhausted
        goal.EndedAt = &now
        g.logIfEnabled(ctx, "goal.exhausted", map[string]any{
            "goal_id":         goal.ID.String(),
            "session_id":      sessionID.String(),
            "completed_turns": goal.CompletedTurns,
            "max_turns":       goal.MaxTurns,
        })
        _ = g.publishGoalEvent(ctx, session, "goal.ended", goal)
        return nil
    }

    // 4. 加载最近对话历史供 Judge 评判
    messages, err := g.deps.MessageRepo.GetRecent(ctx, sessionID, 20)
    if err != nil {
        return fmt.Errorf("load recent messages: %w", err)
    }

    // 5. 调用 Judge 评判
    achieved, reason, tokenUsage, err := g.judge.Evaluate(ctx, goal.Condition, messages)
    if err != nil {
        // Judge 失败：默认保留 goal 到下一轮再判，不放行
        g.logIfEnabled(ctx, "goal.judge_error", map[string]any{
            "goal_id":    goal.ID.String(),
            "session_id": sessionID.String(),
            "error":      err.Error(),
        })
        return nil  // 不 re-queue，等用户下次发消息时再触发
    }

    if achieved {
        // 6a. 目标达成：标记 completed
        now := time.Now()
        if err := g.deps.GoalRepo.Update(ctx, goal.ID, map[string]any{
            "status":            model.GoalStatusCompleted,
            "ended_at":          now,
            "token_usage":       goal.TokenUsage + tokenUsage,
            "last_judge_reason": reason,
        }); err != nil {
            return fmt.Errorf("update goal completed: %w", err)
        }
        goal.Status = model.GoalStatusCompleted
        goal.EndedAt = &now
        goal.TokenUsage += tokenUsage
        goal.LastJudgeReason = reason
        _ = g.publishGoalEvent(ctx, session, "goal.ended", goal)
        return nil
    }

    // 6b. 目标未达成：累加 completed_turns、持久化 feedback、re-queue
    newTurns := goal.CompletedTurns + 1
    newTokenUsage := goal.TokenUsage + tokenUsage
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

    // 推送 goal.updated（每轮都推，前端可据此更新进度）
    _ = g.publishGoalEvent(ctx, session, "goal.updated", goal)

    // 持久化 feedback 消息到 DB（让下一轮 turn 的 GenInput 能加载到）
    feedbackContent := fmt.Sprintf(
        "[Goal Not Yet Achieved — Turn %d/%d]\nReason: %s\n\nPlease continue working on the goal: %s",
        newTurns, goal.MaxTurns, reason, goal.Condition,
    )
    if err := g.persistFeedbackMessage(ctx, sessionID, feedbackContent); err != nil {
        return fmt.Errorf("persist feedback: %w", err)
    }

    // 向 rtc-queue 发布 goal_resume work item
    payload := turnagent.WorkPayload{
        Kind:      turnagent.WorkKindGoalResume,
        SessionID: sessionID.String(),
    }
    body, _ := json.Marshal(payload)
    if err := g.queue.Publish(ctx, queue.Message{
        Body:        body,
        SessionID:   sessionID,
        // 队列路由：保证同 session 串行（依赖现有 rtc-queue 的 session-scoped 调度）
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

// persistFeedbackMessage 把 feedback 作为一条特殊消息写入 DB
// 使用单独的 ContentType，让前端和 summarizeMW 可以识别并特殊处理
func (g *goalCheckpoint) persistFeedbackMessage(ctx context.Context, sessionID uuid.UUID, content string) error {
    msg := &model.Message{
        SessionID:   sessionID,
        Role:        string(schema.User),
        Content:     content,
        ContentType: ContentTypeGoalFeedback,  // 新增消息类型
        // ... 其他必要字段
    }
    return g.deps.MessageRepo.Create(ctx, msg)
}

// publishGoalEvent 推送 goal 生命周期事件到 Centrifuge
// event 取值: "goal.created" / "goal.updated" / "goal.ended"
// 事件发到用户的 topic channel（与 session.updated 同 channel，前端按 event name 区分）
func (g *goalCheckpoint) publishGoalEvent(ctx context.Context, session *model.Session, event string, goal *model.Goal) error {
    return g.publisher.Publish(ctx, updates.UpdatePublishItem{
        Channel: channel.UserTopic(session.OwnerRefID),
        Entity:  updates.EntityGoal,  // 新增 entity 类型
        Event:   event,
        Payload: toProtocolGoal(goal),
    })
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
        if err := a.goalCheckpoint.Evaluate(ctx, sessionUUID, turnStatus); err != nil {
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

Worker 在处理 `WorkKindGoalResume` 时与 `WorkKindSubmit` 走同一条 `Process()` 路径 —— GenInput 加载消息时会看到上一轮持久化的 feedback message，AI 自然知道要继续工作。

### 4.6 与 summarizeMW 的配合

feedback 消息使用专门的 `ContentTypeGoalFeedback` 类型，summarizeMW 在压缩时可以：

- 识别这类消息，保留最新一条，丢弃旧的（避免 N 轮 feedback 累积污染上下文）
- 或者把多条 feedback 合并成一条"progress summary"

> 这是 Phase 6（上下文管理）的优化项，Phase 3 先按默认 summarizeMW 行为跑通。

### 4.7 Turn 取消与 Goal 的交互

- **用户发消息让 AI 工作** → turn `completed` / goal `active` → Goal Checkpoint 评判
- **用户点 Stop（cancel）** → turn `cancelled` / goal 保持 `active` → Goal 不消耗轮次，等待下一次用户消息
- **AI 报错** → turn `failed` / goal 保持 `active` → 同上，不消耗轮次
- **Agent 调用 `cancel_goal` tool** → goal 状态改为 `cancelled`，tool 内推送 `goal.ended` 事件
- **Agent 调用 `create_goal` tool（已有 active goal）** → tool 返回错误提示，Agent 应告知用户"已有一个 goal 在跑，要先取消吗？"

### 4.8 Agent Tools for Goal

Agent 通过 tool 调用驱动 goal 的创建与取消。**前端没有 `/goal` slash 命令**——所有 goal 生命周期转换由 Agent 或后端系统触发。

#### 4.8.1 `create_goal` tool

```go
// server/internal/agent/tools_goal.go

createGoalTool, err := utils.InferTool("create_goal",
    `Create a goal for sustained, multi-turn work. 

When the user describes an objective that requires ongoing effort (e.g., "help me pass all tests", "keep working until the bug is fixed", "optimize performance"), you should:
1. Refine the user's vague intent into a concrete, measurable completion condition (SMART-er)
2. Propose the refined condition to the user for confirmation
3. ONLY after the user confirms, call this tool with the final condition

Do NOT call this tool for one-shot tasks (e.g., "write a function", "explain this code"). Those don't need goal tracking.`,
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

        // 3. 推送 goal.created 事件
        _ = publishGoalEvent(ctx, sessionID, "goal.created", goal)

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

Do NOT call this just because a subtask failed — only when the user explicitly wants to abandon the goal entirely.`,
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
            "status":   model.GoalStatusCancelled,
            "ended_at": now,
        }); err != nil {
            return "", fmt.Errorf("cancel goal: %w", err)
        }
        goal.Status = model.GoalStatusCancelled
        goal.EndedAt = &now

        _ = publishGoalEvent(ctx, sessionID, "goal.ended", goal)

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

#### 4.8.4 Agent 何时识别 goal 意图

系统 prompt 里应加一段引导（放在 [agent.go](server/internal/agent/agent.go) 的 `SystemPrompt` 中）：

> When the user describes an objective that requires sustained, multi-turn effort (e.g., "pass all tests", "fix this bug completely", "refactor this module"), propose a concrete, measurable completion condition and call `create_goal` after confirmation. When the user signals they want to stop (e.g., "stop", "cancel", "never mind"), call `cancel_goal`. Don't use goals for one-shot questions or single-step tasks.

这条引导让 Agent 主动识别 goal 场景并润色条件，而不是等用户显式说"创建一个 goal"。

#### 4.8.5 完整交互示例

```text
User: 帮我把所有单元测试都通过
Agent: 我来看看现在的测试状况……（调用 test runner 工具）……目前 42 个测试中有 7 个失败。
      我建议把 goal 定为："所有 42 个单元测试通过，且不引入新的失败"。你同意吗？
User: 同意
Agent: (调用 create_goal tool, condition="所有 42 个单元测试通过，无新增失败")
      [goal.created 事件推送, 前端状态栏出现 🎯]
      好的，开始工作。首先看 TestX 为什么失败……
      (调用 edit_file 修代码 → 调用 test runner → 没全过 → turn 结束)
      [Goal Checkpoint: Judge 判未达成 → feedback 入队 → 下一轮]
      (turn 2: 看 TestY → 修 → 跑 → ...)
      ...
User: 行了，停下吧，剩下的我明天再弄
Agent: (调用 cancel_goal tool)
      [goal.ended 事件推送, 前端状态栏清除, toast: ⏹ 已取消]
      好的，已取消 goal。当前进度：42 个测试里还有 2 个失败（TestZ, TestW）。
```

---

## 5. Judge 设计

### 5.1 接口定义

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

### 5.2 评判方法

```go
// Evaluate 判断 goal 是否达成
// 返回: achieved, reason, tokenUsage, error
func (j *Judge) Evaluate(ctx context.Context, condition string, messages []*schema.Message) (bool, string, int, error) {
    // 1. 构造评判 prompt
    prompt := j.buildJudgePrompt(condition, messages)
    
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
// 注意：传入 toolNames 让 Judge 知道 AI 实际可用的工具，避免评判"为什么不调 X 工具"而 AI 根本没有 X
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
            // 去掉可能的 "json" 语言标识
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
        // 尝试抓 reason
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

### 5.3 Judge 模型选择与配置

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

## 6. RPC 接口设计

> **调用方说明**：本节定义的 `set_goal` / `clear_goal` 等 RPC 方法**不由前端直接调用**。前端的 `/goal` slash 命令只是一个 UX 入口 —— 它把用户的 goal 意图以结构化文本注入对话（详见 [7.0 节](#70-goal-slash-命令)），由 Agent 看到后接管，最终通过 Agent tool（见 [4.8 节](#48-agent-tools-for-goal)）在服务端内部调用 `SetGoal` / `ClearGoal` 的同等逻辑。
>
> 仍然定义成独立 RPC 方法（而不是 tool 内部函数）是为了：
>
> - 未来支持管理端 / 运维端直接操作 goal（如强制清理卡死的 active goal）
> - 复用鉴权、事件推送、归属校验等逻辑
> - 保持 HTTP/WebSocket 边界清晰，便于调试

### 6.1 设置 Goal

**方法**: `v1.session.set_goal`

**请求**：

```json
{
    "session_id": "uuid",
    "condition": "所有测试通过"
}
```

**响应**（新增 `id` 字段，因为 Goal 现在是独立实体）：

```json
{
    "goal": {
        "id": "goal-uuid",
        "session_id": "session-uuid",
        "condition": "所有测试通过",
        "status": "active",
        "completed_turns": 0,
        "token_usage": 0,
        "last_judge_reason": "",
        "created_at": "2026-09-07T10:00:00Z"
    }
}
```

**Handler 实现**（遵循现有 action handler 样板，参考 [action.stopturn.go](server/internal/handler/rpc/action.stopturn.go)）:

```go
// server/internal/handler/rpc/action.goal.go

func (h *Handler) SetGoal(ctx context.Context, req *protocol.SetGoalRequest) (*protocol.SetGoalResponse, error) {
    // 1. 鉴权（现有样板）
    userID, ok := contextx.GetUserID(ctx)
    if !ok {
        return nil, h.unauthorizedError(ctx)
    }

    sessionUUID, apiErr := parseUUID(req.SessionId, "session_id")
    if apiErr != nil {
        return nil, apiErr
    }

    // 2. 加载 session + 归属校验
    session, err := h.deps.SessionRepo.GetByID(ctx, sessionUUID)
    if err != nil {
        if repo.IsNotFound(err) {
            return nil, &APIError{Code: "session.not_found", Message: "session not found"}
        }
        return nil, h.internalError(ctx, "session.error", "load session", err)
    }
    if apiErr := primitives.CheckSessionOwnership(session, userID); apiErr != nil {
        return nil, apiErr
    }

    // 3. 检查是否已有活跃 goal（查 goals 表）
    existing, err := h.deps.GoalRepo.FindActive(ctx, sessionUUID)
    if err != nil {
        return nil, h.internalError(ctx, "goal.find_active_error", "find active goal", err)
    }
    if existing != nil {
        return nil, &APIError{Code: "goal.already_active", Message: "a goal is already active"}
    }

    // 4. 创建新 goal 行
    goal := &model.Goal{
        ID:             uuid.New(),
        SessionID:      sessionUUID,
        Condition:      req.Condition,
        Status:         model.GoalStatusActive,
        CompletedTurns: 0,
        TokenUsage:     0,
        MaxTurns:       h.cfg.DefaultMaxTurns,  // 创建时固化当时的配置
        CreatedAt:      time.Now(),
    }
    if err := h.deps.GoalRepo.Create(ctx, goal); err != nil {
        return nil, h.internalError(ctx, "goal.create_error", "failed to create goal", err)
    }

    // 5. 推送 goal.created 事件（前端订阅后刷新状态栏 / 显示 toast）
    _ = h.publishGoalEvent(ctx, session, "goal.created", goal)

    return &protocol.SetGoalResponse{
        Goal: toProtocolGoal(goal),
    }, nil
}
```

> 注：`/goal set` 是创建一条**新的** Goal 行。如果 session 已有历史 goal（completed/cancelled/exhausted），不影响 —— `FindActive` 只匹配 status='active'，历史 goal 自然保留在 goals 表中。

### 6.2 查询 Goal

**方法**: `v1.session.get_goal`

**响应**：

```json
{
    "goal": {
        "condition": "所有测试通过",
        "status": "active",
        "completed_turns": 5,
        "token_usage": 12345,
        "last_judge_reason": "还差 2 个单元测试未通过: TestX, TestY",
        "created_at": "2026-09-07T10:00:00Z"
    }
}
```

### 6.3 清除 Goal

**方法**: `v1.session.clear_goal`

**Handler 要点**：

- 调用 `GoalRepo.FindActive` 拿当前 active goal；如果无则返回 `goal.not_found`
- 更新 goal 状态为 `cancelled`，设置 `ended_at`
- 推送 `goal.ended` 事件（前端据此清除状态栏）
- **不删除行** —— 历史 goal 保留在 goals 表供 `list_goals` 查询

### 6.4 列出 Goal 历史

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
            "ended_at": "2026-09-07T10:05:00Z",
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
            "ended_at": "2026-09-07T11:45:00Z",
            "last_judge_reason": "P95 仍为 180ms，未能达到 100ms 目标"
        }
    ],
    "next_cursor": "optional-next-page-cursor"
}
```

**Handler 要点**：

- 复用 `loadOwnedSession` 做归属校验
- 调用 `GoalRepo.ListBySession(ctx, sessionID, limit)`，按 `created_at DESC` 排序
- 返回分页结果，支持 cursor 翻页

---

## 7. 前端交互

前端**只有 `/goal` 一个命令**，用于创建 goal 的入口。没有 `/goal clear`、`/goal list`、`/goal status` 等子命令 —— 取消、查询、历史等功能全部通过 Agent 对话或 UI 控件完成。

### 7.0 `/goal` slash 命令

`/goal <condition>` 是一个**纯前端 UX 入口**，不直接调用后端 RPC。它的作用是把用户的 goal 意图以结构化文本注入到当前对话，让 Agent 看到后接管后续流程：

```text
用户在输入框键入：/goal 重构这个模块

前端实际发送的消息内容：
"[Goal intent] 用户想设置一个目标：重构这个模块
请润色为可判定的 SMART 条件，向用户确认后调用 create_goal tool。"
```

后续流程完全由 Agent 驱动：

1. Agent 把模糊条件润色成 SMART 形式，向用户提议
2. 用户确认（或修改）后，Agent 调用 `create_goal` tool
3. 后端创建 goal 行，推送 `goal.created` 事件
4. 前端订阅事件后显示状态栏

> 为什么不直接调 RPC？因为只有 Agent 才能把用户模糊意图（"重构这个模块"）润色成 Judge 可判定的 SMART 条件（"所有 42 个测试通过且无 lint error"）。前端的 `/goal` 命令只是一个意图识别的钩子，实际创建仍走 Agent 驱动流程。

**前端职责**：

- 解析 `/goal` 命令，构造上述结构化消息发给后端（或等价地在消息上附加 `intent: 'goal_create'` 标记）
- 订阅 `goal.*` 事件，更新状态栏和发通知
- 提供入口查看 goal 历史（如状态栏上的"History"按钮、session 设置页）

### 7.1 Goal 事件订阅

前端订阅用户 topic channel 后，按 event name 过滤 goal 事件：

```typescript
// web-components/src/subscriptions/goal-events.ts

topicChannel.on('goal.created', (goal: Goal) => {
    goalStore.setActive(goal);              // 状态栏显示新 goal
    // 不需要 toast —— Agent 在对话中已经告知用户 goal 已创建
});

topicChannel.on('goal.updated', (goal: Goal) => {
    goalStore.setActive(goal);              // 进度更新（turns / last_reason）
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

### 7.2 状态栏

```typescript
// web-components/src/components/StatusBar.tsx

function GoalIndicator({ goal }: { goal: Goal | null }) {
    if (!goal || goal.status !== 'active') return null;

    return (
        <div className="goal-indicator">
            <span className="icon">🎯</span>
            <span className="condition">{goal.condition}</span>
            <span className="turns">Turn {goal.completed_turns}/{goal.max_turns}</span>
            {goal.last_judge_reason && (
                <span className="reason" title={goal.last_judge_reason}>💬</span>
            )}
        </div>
    );
}
```

### 7.3 Goal 历史面板

通过状态栏上的"History"按钮或 session 设置页入口打开：

```typescript
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
        duration: g.ended_at ? formatDuration(g.created_at, g.ended_at) : 'in progress',
        reason: g.last_judge_reason,
    })));
}
```

---

## 8. 关键规则

### 8.1 生命周期

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
        Goal Checkpoint
              ↓
       ┌──────┴──────┐
       ↓              ↓
  [ completed ]  [ exhausted ]
```

**关键语义**：

- `active`：每次 turn 正常完成时触发 Goal Checkpoint 评判
- `completed`：推送 `goal.ended`，前端显示 🎉
- `cancelled`：Agent tool 即时生效；如果此时有 turn 正在执行，该 turn 完成后 Goal Checkpoint 看到 cancelled 状态即跳过
- `exhausted`：推送 `goal.ended`，前端显示 ⚠️ 引导用户优化条件
- **终态不可逆**：goal 一旦进入 completed/cancelled/exhausted，不能改回 active。需要新一轮就让 Agent 调用 `create_goal` 创建新 goal

### 8.2 事件模型

Goal 独立于 Session 后，有自己的事件生命周期，**不再通过 `session.updated` 承载**：

- **`goal.created`**：新 goal 创建时推送，payload 是完整 goal 对象
- **`goal.updated`**：goal 字段变更时推送（每轮 turn 完成后 completed_turns / token_usage / last_judge_reason 变化）。payload 是完整 goal 对象
- **`goal.ended`**：goal 进入终态时推送（completed / cancelled / exhausted）。payload 是完整 goal 对象

事件发到用户的 topic channel（与 session 事件同 channel），前端按 event name 路由到 goal store。

### 8.3 约束条件

1. **单会话单 active goal**：一个会话同时只能有一个 `active` goal；Agent 调用 `create_goal` 时若已有 active goal，tool 返回错误提示让 Agent 询问用户是否先取消
2. **max_turns 创建时固化**：goal 行的 `max_turns` 字段保存创建时的配置值，避免未来改配置影响历史 goal
3. **仅 completed turn 计入轮次**：cancel / interrupt / fail 不消耗 goal 配额
4. **Judge 失败容错**：Judge 模型错误时，**保留 goal 等待下一轮再判**（不放行也不 cancel），记录日志
5. **Token 追踪**：记录 Judge + 主 Agent 跨 turn 累计的 token 消耗
6. **Feedback 持久化**：feedback 消息写入 DB 作为独立消息行（`ContentTypeGoalFeedback`），不依赖内存 state
7. **历史不可变**：终态 goal 行保留，不删除不修改，供 `list_goals` 查询

### 8.4 边界情况

- **Goal 已存在时 Agent 调用 create_goal** → tool 返回 `"There is already an active goal. Ask the user whether to cancel it first."`，Agent 据此询问用户是否用 `cancel_goal` 先取消
- **Judge 模型 API 错误** → 保留 goal，等待下一轮触发时再判（不轻言放弃）
- **达到最大轮次** → goal 进入 `exhausted` 终态，推送 `goal.ended`，前端显示 ⚠️ 引导
- **Session 关闭时 goal 仍活跃** → Goal 状态保留在 DB；**不会自动继续**，需要用户重新打开 session 并发消息才会触发下一轮评判
- **用户 cancel 正在跑的 turn** → turn 状态为 cancelled，Goal Checkpoint 跳过评判，goal 保持 active 等待下次
- **Judge 反复判定 "未达成" 但 AI 实际已无事可做** → 由 max_turns 兜底进入 exhausted；产品上建议前端引导用户写 SMART 条件
- **历史 goal 查询** → 通过 `list_goals` RPC，按 created_at DESC 分页

---

## 9. 实现计划

### 9.1 Phase 1: 数据模型

- [ ] 创建 `server/internal/model/goal.go`，定义 `Goal` 独立表
- [ ] 创建 `server/internal/repo/goal_repo.go`，实现 `GoalRepo` 接口（含 `FindActive`、`ListBySession`、`Create`、`Update`）
- [ ] 更新 `protocol` 定义（OpenAPI spec）：新增 `set_goal` / `get_goal` / `clear_goal` / `list_goals` 方法
- [ ] 重新生成 protocol 代码

### 9.2 Phase 2: Judge 实现

- [ ] 创建 `judge.go`
- [ ] 实现 `Evaluate` 方法
- [ ] 配置 Judge 模型（Haiku）

### 9.3 Phase 3: Goal Checkpoint + rtc-queue 集成

- [ ] 在 `server/pkg/turn-agent/types.go` 新增 `WorkKindGoalResume`
- [ ] 创建 `server/internal/agent/goal_checkpoint.go`，实现 `Evaluate` 方法
- [ ] 在 `agent_process.go` 的 `Process()` 末尾调用 Goal Checkpoint
- [ ] feedback 消息使用 `ContentTypeGoalFeedback` 持久化到 DB
- [ ] re-queue 逻辑：调用 `queue.Publish` 发布 `goal_resume` work item
- [ ] 验证：rtc-queue 是否已保证同 session 串行；否则需加 session 路由

### 9.4 Phase 4: RPC 接口

- [ ] 创建 `action.goal.go`
- [ ] 实现 `SetGoal` / `GetGoal` / `ClearGoal` / `ListGoals`，使用 `primitives.CheckSessionOwnership`
- [ ] 新增 `GoalRepo` 接口 + GORM 实现（含 `FindActive`、`ListBySession`）
- [ ] 注册 RPC 路由
- [ ] 推送 `goal.created` / `goal.updated` / `goal.ended` 事件到 user topic channel
- [ ] 新增 `updates.EntityGoal` entity 类型

### 9.5 Phase 5: 前端集成

- [ ] `/goal <condition>` 命令解析 —— 把用户意图以结构化文本注入对话（不直接调 RPC）
- [ ] 订阅 `goal.created` / `goal.updated` / `goal.ended` 事件
- [ ] 状态栏显示（condition + completed_turns/max_turns + last_judge_reason）
- [ ] 完成通知（🎉 completed / ⚠️ exhausted / 静默 cancelled）
- [ ] Goal 历史面板入口（状态栏 "History" 按钮或 session 设置页，调 `ListGoals` RPC 分页加载）

---

## 10. 测试计划

### 10.1 单元测试

- **Judge.Evaluate — 目标达成**：返回 `achieved=true` + reason
- **Judge.Evaluate — 目标未达成**：返回 `achieved=false` + reason
- **Judge.parseResponse — JSON 响应**：正确解析
- **Judge.parseResponse — markdown fence 包裹**：剥离后正确解析
- **Judge.parseResponse — 完全无法解析**：返回 `achieved=false` + unparseable 提示
- **GoalCheckpoint.Evaluate — goal 非 active**：直接返回，不调用 Judge
- **GoalCheckpoint.Evaluate — turn 非 completed（cancel/fail）**：跳过评判
- **GoalCheckpoint.Evaluate — 达到最大轮次**：goal 进入 `exhausted` 终态，推送 `goal.ended`，不 re-queue
- **GoalCheckpoint.Evaluate — Judge 报错**：保留 goal，不 re-queue，记日志
- **GoalCheckpoint.Evaluate — 未达成**：feedback 消息写入 DB + goal_resume work item 发布到 rtc-queue + 推送 `goal.updated`

### 10.2 集成测试

- `/goal 所有测试通过` + AI 完成工作 → Goal 达成，状态变 `completed`，推送 `goal.ended`
- `/goal 所有测试通过` + AI 未完成 → Goal 未达成，rtc-queue 收到 goal_resume，新 turn 看到 feedback 继续工作
- 达到 max_turns 上限 → goal 进入 `exhausted`，推送 `goal.ended`，前端显示 ⚠️
- 用户在 AI 工作时 cancel turn → goal 保持 active，等待下次
- `/goal clear` → goal 状态变 `cancelled`，推送 `goal.ended`，in-flight turn 完成后不再评判
- `/goal history` → 返回该 session 所有 goal（含 completed/cancelled/exhausted 历史），按 created_at DESC 排序

---

## 11. 风险与缓解

- **Judge 误判（达成判未达成）**：AI 继续工作但无法达成 → 由 max_turns 兜底进入 `exhausted`；产品上引导用户写 SMART 条件
- **Judge 误判（未达成判达成）**：AI 提前停止，用户失望 → 允许用户重设 goal；未来可考虑"Judge 两次确认"机制
- **无限循环（AI 反复做同样的事）**：每轮 feedback 相同 → summarizeMW 压缩 + 用户主动 `/goal clear`
- **Token 消耗过高**：跨 turn 累积 → `Goal.TokenUsage` 追踪；前端显示；未来可加 token 预算上限
- **rtc-queue 同 session 并发**：goal_resume 与用户新消息被不同 worker 同时认领 → 验证 queue 是否 session-scoped，否则需加分布式锁
- **Feedback 累积污染上下文**：N 轮 feedback 堆在消息历史 → `ContentTypeGoalFeedback` 类型标识，summarizeMW 特殊处理（Phase 6 优化项）
- **Session 关闭后的"自动继续"误解**：文档需明确"保留状态，下次发消息时继续"，不是后台自动运行
- **Goal 表膨胀**：长期运行的 session 累积大量历史 goal → 建议加 TTL（如 90 天后归档），或在 list_goals 默认只查近期

---

## 12. 未来扩展

1. **多 active goal 支持**：允许同时设定多个 active goal（目前限制为单 active）
2. **目标优先级**：多个 active goal 时按优先级评判
3. **Goal 暂停/恢复**：`/goal pause` / `/goal resume`，新增 `paused` 状态
4. **自定义 Judge**：允许用户指定评判模型
5. **Goal 模板**：预定义常用 goal 模板
6. **跨 Session Goal**：Goal 解耦 Session 后，可支持一个 Goal 跨多个 Session 执行
7. **Goal 归档**：长期历史 goal 自动归档（如 90 天后），减轻 goals 表压力

---

## 附录 A: 与 /loop 命令的对比

- **循环控制**：`/goal` 目标驱动；`/loop` 轮次驱动 / 时间驱动
- **停止条件**：`/goal` Judge 判定；`/loop` 轮次完成 / 时间到达
- **状态管理**：`/goal` 独立 goals 表；`/loop` 数据库任务表
- **底层机制**：两者都通过 rtc-queue 驱动 turn 循环，差异仅在"是否 re-queue"的判定逻辑
- **适用场景**：`/goal` 模糊目标（如"所有测试通过"）；`/loop` 明确任务（如"循环 5 轮"）

---

## 附录 B: 相关文档

- [13-commands.md](../13-commands.md) - 命令系统总览
- [11-context-management.md](../11-context-management.md) - 上下文管理
- [Eino Middleware Reference](../../eino-agent/reference/middleware.md)

---

**文档结束**
