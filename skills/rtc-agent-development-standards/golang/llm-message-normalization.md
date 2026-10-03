# LLM 消息规范化

> _LLM API 对消息序列有严格的结构约束。数据库中的消息不必满足这些约束——规范化是发送到 LLM 之前的最后一道工序。_

---

## 1. 为什么需要规范化

数据库中存储的消息序列可能违反 LLM API 的约束：

- 用户连续发送多条消息（如文字 + 图片附件），导致连续 user 角色
- 工具调用/结果的配对可能被中断（如 agent 崩溃后恢复）
- 系统消息可能被插入到对话中间

LLM API（Claude、OpenAI）要求消息满足交替角色、工具配对等约束。规范化层在消息发送到 LLM 之前执行结构化修复，使内部存储的灵活性与外部 API 的严格性解耦。

---

## 2. 规范化管道

规范化按固定顺序执行四步，每步的输出作为下一步的输入：

```text
原始消息序列
  │
  ├─ Step 1: extractSystemMessages   → 系统消息提取到首部
  ├─ Step 2: repairToolPairing       → 修复工具调用/结果配对
  ├─ Step 3: mergeConsecutiveSameRole → 合并连续同角色消息
  └─ Step 4: validateMessageSequence  → 验证最终序列结构
  │
  ▼
规范化后的消息序列（满足 LLM API 约束）
```

**顺序约束**：repair 必须在 merge 之前。原因：repair 删除中间的 assistant/tool 消息可能创建新的连续同角色对（如 `user → assistant(tool_call) → user` 在 assistant 被删除后变成 `user → user`），merge 需要吸收这些新产生的连续对。

```go
// ✅ 固定管道顺序
messages = extractSystemMessages(messages)
messages = repairToolPairing(messages)       // 先修复
messages = mergeConsecutiveSameRole(messages) // 后合并
if err := validateMessageSequence(messages); err != nil {
    // 验证失败 = repair/merge 无法修复的结构性问题
}

// ❌ 顺序错误：merge 在 repair 之前
messages = mergeConsecutiveSameRole(messages) // 此时连续对可能不完整
messages = repairToolPairing(messages)         // repair 可能创建新的连续对 → 未合并！
```

**约束**：

- 规范化在 `loadMessages` 管道的最后一步执行，所有内容注入（附件、命令、场景）完成之后
- 新增消息处理步骤时，明确其在管道中的位置——repair 之后、validate 之前
- 验证失败仅用于记录 warn 日志（表示 repair 无法修复的 bug），不阻塞消息发送

---

## 3. 连续同角色消息合并

### 合并规则

相邻的 user 消息合并为一条：`Content` 和 `ReasoningContent` 用 `"\n"` 连接，`MultiContent`（图片等多模态内容）用 `append` 拼接。合并后的消息保留第一条的元数据（`CreatedAt`、`TokenUsage`、`Extra` 等）。

**assistant 消息不在此处合并**——assistant 消息的合并由 `MergeAssistantMiddleware` 在 `schema.Message` 层执行，使用 `AssistantGenMultiContent` 而非 `"\n"` 拼接，以获得更好的缓存命中率。

```go
// ✅ 合并连续 user 消息
// 输入: [{Role: "user", Content: "hello"}, {Role: "user", Content: "world"}]
// 输出: [{Role: "user", Content: "hello\nworld"}]

// ✅ 合并 MultiContent（图片等多模态内容）
// 输入: [{Role: "user", Content: "look", MultiContent: [img1]},
//         {Role: "user", Content: "at this", MultiContent: [img2]}]
// 输出: [{Role: "user", Content: "look\nat this", MultiContent: [img1, img2]}]

// ❌ 合并 assistant 消息（应由 MergeAssistantMiddleware 处理）
// assistant 合并不在此处，因为需要使用 MultiContent 而非 "\n" 拼接
```

### Copy-on-First-Merge 语义

合并时不能直接修改调用方的原始消息对象。第一次合并发生时，对被合并的消息做浅拷贝，后续合并直接修改拷贝后的对象。未参与合并的消息不拷贝（直接透传）。

```go
// ✅ Copy-on-first-merge：只在第一次合并时拷贝
// 第一次合并：浅拷贝 prev，修改拷贝
copied := *prev
result[len(result)-1] = &copied
copied.Content = joinContent(copied.Content, curr.Content)

// 后续合并：直接修改已拷贝的对象（不需要再拷贝）
prev.Content = joinContent(prev.Content, curr.Content)

// ❌ 直接修改调用方的原始消息
prev.Content = joinContent(prev.Content, curr.Content) // prev 可能是调用方的对象！

// ❌ 每次都拷贝（性能浪费）
for each merge {
    copied := *prev // 不必要的重复拷贝
}
```

### MultiContent 合并的安全拷贝

合并 `MultiContent` 时必须使用 `make + append` 创建新的底层数组，不能直接 `append(prev.MultiContent, curr.MultiContent...)`。原因：prev 即使已浅拷贝，其 `MultiContent` 的 slice header 仍指向原始底层数组，直接 append 可能修改原始数组或与其他 goroutine 共享的 slice。

```go
// ✅ 安全拷贝：make + append 创建新的底层数组
merged := make([]schema.MessageInputPart, 0, len(prev.MultiContent)+len(curr.MultiContent))
merged = append(merged, prev.MultiContent...)
merged = append(merged, curr.MultiContent...)
prev.MultiContent = merged

// ❌ 直接 append：可能修改共享的底层数组
prev.MultiContent = append(prev.MultiContent, curr.MultiContent...)
```

---

## 4. 工具调用/结果配对修复

每个 assistant 的 `tool_calls` 必须有对应数量的 tool result 消息，反之亦然。修复使用计数器方法：遇到 assistant 的 tool_calls 时增加 pending 计数，遇到 tool result 时减少。配对不匹配时删除孤立的消息。

```go
// ✅ 计数器配对
pending := 0
for _, msg := range messages {
    switch msg.Role {
    case turnagent.RoleAssistant:
        pending += len(msg.ToolCalls)
    case turnagent.RoleTool:
        if pending <= 0 {
            // 孤立的 tool result，删除
            continue
        }
        pending--
    }
}
// 结束后 pending > 0 表示有 assistant tool_calls 没有结果
```

**约束**：

- 删除方向：优先删除孤立的 tool result（没有对应 tool_call），其次删除没有结果的 tool_calls
- 修复后日志记录删除数量，便于追踪数据质量问题
- 修复是 best-effort——无法修复的结构性问题由 validate 步骤报告

---

## 5. Content 与 MultiContent 的双字段映射

Eino 的 `schema.Message` 同时支持 `Content`（纯文本）和 `UserInputMultiContent`（多模态内容数组）。当用户消息同时包含文本和附件（图片等）时，两个字段都需要填充。

**关键约束**：Eino 的 Claude 适配器使用 `else if` 逻辑——`UserInputMultiContent` 存在时忽略 `Content`。因此当同时有文本和多模态内容时，必须将 `Content` 作为 `Text` part 加入 `UserInputMultiContent`，否则文本会被静默丢弃。

```go
// ✅ 双字段填充：确保 Content 在 MultiContent 中也有对应 part
if len(m.MultiContent) > 0 {
    var parts []schema.MessageInputPart
    if m.Content != "" {
        parts = append(parts, schema.MessageInputPart{
            Type: schema.ChatMessagePartTypeText,
            Text: m.Content,
        })
    }
    parts = append(parts, m.MultiContent...)
    em.UserInputMultiContent = parts
    // Content 保留原值（向后兼容，Claude 适配器此时忽略它）
}

// ❌ 只设置 Content，不设置 MultiContent
em.Content = m.Content
// 如果 m.MultiContent 有内容，Claude 适配器会忽略 Content

// ❌ 只设置 MultiContent，不包含 Content
em.UserInputMultiContent = m.MultiContent
// m.Content 的文本丢失！
```

**约束**：

- 新增多模态内容类型时，确保 `toEinoMessage` 正确处理 Content + MultiContent 的合并
- 此行为依赖 Claude 适配器的 `else if` 逻辑——更换 LLM 适配器时需验证此行为
- `Content` 字段保留原值（不置空），确保不使用 MultiContent 的旧路径仍正常工作

---

## 6. LLM 输出防御性清洗

不同 LLM 模型（尤其是通过代理调用的模型）可能在输出中泄漏非内容标记，污染后续对话的上下文。规范化层必须在将 assistant 消息重新注入 LLM 上下文之前，清洗这些泄漏。

### Think Tag 泄漏

部分模型（如 qwen3.7-plus 通过代理）会将 `<think>...</think>` 思考标签泄漏到 `text` 内容字段中。如果不清除，这些标签会作为 assistant 历史消息被重新发送给 LLM，导致后续 turn 的上下文被污染。

```go
// ✅ 在 convertDBMessage 中清洗 assistant 消息的 think 标签泄漏
// data_context_convert.go
if msg.Role == string(schema.Assistant) {
    text = sanitizeThinkTagLeak(text)
}

// thinkTagPattern matches a complete <think>...</think> block (case-insensitive,
// dot-all) that may leak into assistant text content during streaming.
var thinkTagPattern = regexp.MustCompile(`(?is)<think>.*?</think>\s*`)

func sanitizeThinkTagLeak(content string) string {
    return thinkTagPattern.ReplaceAllString(content, "")
}

// ❌ 不清洗，泄漏的 think 标签进入后续 LLM 上下文
// assistant 消息的 text 内容原样传递到 LLM
// → LLM 看到自己之前的 "thinking" 内容，可能产生重复思考或混乱
```

### 防御性清洗的判断标准

| 条件 | 是否需要清洗 | 原因 |
| --- | --- | --- |
| 消息来自 assistant 角色 | 是 | 只有 assistant 输出可能泄漏思考标签 |
| 消息来自 user 角色 | 否 | 用户输入是原始数据，不需要清洗 |
| 消息来自 tool 角色 | 否 | 工具输出是结构化的，不含思考标签 |
| 内容字段是 `ReasoningContent` | 否 | 思考内容是合法的 reasoning 数据 |

**约束**：

- 清洗仅在 assistant 消息的 `Content`（text）字段执行，不处理 `ReasoningContent`
- 清洗函数使用正则匹配完整标签对（`<think>...</think>`），不处理未闭合的标签（避免误删用户输入中的 `<think` 文本）
- 新增 LLM 提供商或模型时，观察其是否有类似的输出泄漏，如有则添加对应的清洗规则
- 清洗函数放在 `data_context_convert.go` 中，与消息转换逻辑同文件

---

## 7. 新增内容类型的处理

当新增内容类型（如视频、音频、PDF）需要注入到用户消息时：

1. **加载阶段**（`file_loader.go`）：从 OSS 加载文件，检测 MIME 类型，预处理（如图片压缩、文本截断）
2. **注入阶段**（`data_context_convert.go`）：将加载结果注入到消息的 `MultiContent`（图片）或 `Content`（文本，XML 包裹）
3. **规范化阶段**（`message_normalizer.go`）：合并连续 user 消息时，正确合并 `MultiContent` 和 `Content`
4. **映射阶段**（`toEinoMessage`）：将 `Content` + `MultiContent` 正确映射到 Eino 的 `schema.Message`

```go
// 图片 → MultiContent
msg.MultiContent = append(msg.MultiContent, ImageToInputPart(imageData, mimeType))

// 文本文件 → Content（XML 包裹）
msg.Content += fmt.Sprintf("\n<file name=%q>%s</file>", filename, textContent)
```

**约束**：

- 新增内容类型必须明确走 MultiContent（二进制/多模态）还是 Content（文本）路径
- 文件加载必须有大小限制（参见 [性能规范 - 防御性 I/O 限制](./performance.md#防御性-io-限制)）
- 文件加载必须通过 magic bytes 检测真实 MIME 类型（参见 [安全规范 - 文件类型校验](../05-security-standards.md#文件类型校验magic-bytes-内容检测)）
- 文件加载在后台 worker 中执行时，UserID 必须通过 context 传递（参见 [并发规范 - 后台 goroutine 必须传递身份](./concurrency.md#后台-goroutine-必须显式传递身份)）

---

## 8. Checkpoint Resume 消息处理

当 TurnLoop 从 checkpoint 恢复执行时（如工具调用中断后等待用户审批），消息处理需要特殊逻辑。checkpoint 保存的是中断时的消息快照，但中断期间数据库可能新增了用户消息，且 eino 的 ToolNode 会在恢复时自动为"待处理的 tool call"生成 tool result——如果也从数据库加载这些 tool result，就会产生重复。

### HistoryModifier 重新加载消息

使用 `WithHistoryModifier` 在恢复时从数据库重新加载消息，替代 checkpoint 中可能过时的消息快照。这确保中断期间用户发送的新消息被包含在 LLM 上下文中。

```go
// ✅ 通过 HistoryModifier 在恢复时重新加载消息
runOpts := []adk.AgentRunOption{
    adk.WithHistoryModifier(func(ctx context.Context, msgs []*schema.Message) []*schema.Message {
        ctx = WithLoadSource(ctx, LoadSourceGenResume)  // 标记为 resume 加载

        // 提取 pending tool call IDs（见下节）
        pendingIDs := extractPendingToolCallIDs(msgs)
        if len(pendingIDs) > 0 {
            ctx = WithPendingToolCallIDs(ctx, pendingIDs)
        }

        // 从数据库重新加载消息（捕获中断期间的新消息）
        freshMsgs, err := mgr.cfg.LoadMessages(ctx, mgr.sessionID)
        if err != nil {
            // 加载失败时降级使用 checkpoint 消息（不阻塞恢复）
            mgr.log(ctx, LogLevelWarn, "gen_resume.history_modifier_load_failed", ...)
            return msgs
        }
        return toEinoMessages(freshMsgs)
    }),
}

// ❌ 使用 checkpoint 的消息快照，中断期间的新用户消息被丢弃
```

### Pending Tool Call ID 提取

checkpoint 消息中可能有 assistant 的 `tool_calls` 但缺少对应的 tool result（因为中断时 tool 还未执行）。恢复时 eino 的 ToolNode 会自动重新调用这些 tool 并生成 result。如果 `LoadMessages` 也从数据库加载了这些 tool result（可能由之前的恢复路径写入），就会出现重复。

解决方法：从 checkpoint 消息中提取"待处理的 tool call ID"（有 tool_call 但无匹配 tool result 的 ID），通过 context 传递到 `LoadMessages`，后者跳过这些 ID 对应的 tool result。

```go
// ✅ 提取 pending tool call IDs
func extractPendingToolCallIDs(msgs []*schema.Message) map[string]bool {
    // 收集所有 assistant tool_call IDs
    allCallIDs := make(map[string]bool)
    for _, msg := range msgs {
        if msg.Role == schema.Assistant {
            for _, tc := range msg.ToolCalls {
                allCallIDs[tc.ID] = true
            }
        }
    }
    // 移除已有匹配 tool result 的 ID
    for _, msg := range msgs {
        if msg.Role == schema.Tool && msg.ToolCallID != "" {
            delete(allCallIDs, msg.ToolCallID)
        }
    }
    return allCallIDs  // 剩余的 = 待 eino ToolNode 处理的
}

// ❌ 不提取 pending IDs，从 DB 加载所有 tool result → 重复
```

**约束**：

- `GenResume` 必须使用 `WithHistoryModifier` 重新加载消息，不使用 checkpoint 的静态快照
- `extractPendingToolCallIDs` 在 checkpoint 消息上运行（不是 DB 消息），识别 eino 会自动处理的 tool call
- `LoadMessages` 必须检查 `PendingToolCallIDs` context 值，跳过对应 ID 的 tool result
- HistoryModifier 加载失败时降级使用 checkpoint 消息（warn 日志），不阻塞恢复流程
- `LoadSource` context 值标记为 `LoadSourceGenResume`，供 `LoadMessages` 区分恢复场景（跳过命令检测和 prompt 持久化，避免重复）

---

## 总结

消息规范化是内部灵活存储和外部 API 严格约束之间的桥梁。新增消息处理逻辑时，遵循以下原则：

- **修复先于合并**：repair 可以创建新的连续对，merge 必须吸收它们
- **不修改原始对象**：copy-on-first-merge 保护调用方的数据
- **安全拷贝 slice**：`make + append` 避免共享底层数组
- **双字段同步**：Content 和 MultiContent 必须同时填充，防止适配器静默丢弃
- **防御性清洗**：assistant 输出在重新注入 LLM 上下文前，必须清洗模型特有的标记泄漏
- **Checkpoint Resume 消息刷新**：恢复时通过 HistoryModifier 重新加载消息，跳过 eino 会自动生成的 pending tool result，防止重复

> _"规范化不是改变数据，而是让数据适应接口的期望。"_
