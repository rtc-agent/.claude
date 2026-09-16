# Claude Code 命令系统调研报告

## 一、命令系统整体架构

### 1.1 三种命令类型

| 类型 | 执行方式 | 典型例子 |
|------|----------|----------|
| `local` | 本地执行，返回文本结果 | `/compact`, `/clear`, `/cost` |
| `local-jsx` | 渲染 React Ink UI | `/help`, `/config`, `/mcp` |
| `prompt` | 生成 prompt 注入对话上下文交给模型 | `/loop`, `/remember`, 用户自定义 skill |

### 1.2 命令来源（6种）

| 来源 | 说明 |
|------|------|
| 内建命令 | `src/commands/` 下的 TypeScript 模块，通过 `COMMANDS()` 聚合 |
| Skills 目录 | `~/.claude/skills/` 和 `.claude/skills/` 下的 SKILL.md |
| Bundled Skills | 通过 `registerBundledSkill()` 编程式注册（如 `/loop`） |
| Plugin | 插件的 markdown 文件和 skills/ 目录 |
| Workflow | 工作流脚本 |
| MCP | MCP 服务器提供的 prompt |

### 1.3 命令注册流程

```
COMMANDS() + loadAllCommands(cwd) → getCommands(cwd)
  → meetsAvailabilityRequirement()  -- auth/provider 门控
  → isCommandEnabled()              -- feature flag 门控
  → + getDynamicSkills()            -- 动态发现
  → 最终命令列表
```

合并顺序：bundled skills 排在最前，内建命令排在最后。`findCommand` 线性搜索，前面的优先命中。

### 1.4 命令查找规则

```typescript
findCommand(commandName, commands) {
  return commands.find(_ =>
    _.name === commandName ||           // 1. name 直接匹配
    getCommandName(_) === commandName || // 2. userFacingName 匹配
    _.aliases?.includes(commandName)     // 3. aliases 匹配
  )
}
```

### 1.5 processSlashCommand 完整流程

```
用户输入 /command args
  → parseSlashCommand()       -- 解析 commandName, args, isMcp
  → hasCommand()              -- 检查命令是否存在
  → getMessagesForSlashCommand()
      → switch(command.type)
          → 'local':      load() → call() → LocalCommandResult
          → 'local-jsx':  load() → call(onDone) → ReactNode
          → 'prompt':
              → context === 'fork'? → executeForkedSlashCommand()
              → else → getMessagesForPromptSlashCommand()
                  → command.getPromptForCommand(args, context)
                      → substituteArguments()    -- 参数替换
                      → ${CLAUDE_SKILL_DIR} 替换
                      → ${CLAUDE_SESSION_ID} 替换
                      → executeShellCommandsInPrompt()
                  → registerSkillHooks()
                  → addInvokedSkill()
                  → 构建消息序列
```

### 1.6 参数替换系统

| 占位符 | 说明 |
|--------|------|
| `$ARGUMENTS` | 完整参数字符串 |
| `$ARGUMENTS[N]` | 第 N 个参数（shell-quote 解析） |
| `$N` | 索引简写 |
| `$name` | 命名参数（通过 frontmatter `arguments` 定义） |

如果没有占位符且 `appendIfNoPlaceholder` 为 true，自动追加 `\n\nARGUMENTS: {args}`。

### 1.7 模型调用 vs 用户调用

| 维度 | 用户调用 /skill | 模型调用 SkillTool |
|------|----------------|-------------------|
| 入口 | processSlashCommand | SkillTool.call |
| 支持类型 | local, local-jsx, prompt | 仅 prompt |
| 权限检查 | userInvocable 字段 | deny/allow rules |
| 特殊能力 | - | contextModifier 动态修改工具权限 |

---

## 二、/compact 命令

### 2.1 多层次压缩体系

| 层次 | 名称 | 说明 | 是否调用 LLM |
|------|------|------|-------------|
| 1 | Microcompact | 清除旧工具结果，替换为标记 | 否 |
| 2 | Session Memory Compaction | 用后台提取的结构化笔记替代历史 | 否（提取时调用） |
| 3 | Legacy Full Compact | 调用 LLM 生成对话摘要 | 是 |
| 4 | Reactive Compact | API 返回 413 后的被动压缩 | 是 |
| 5 | Partial Compact | 用户选择特定消息范围 | 是 |

### 2.2 手动 /compact 执行流程

```
用户输入 /compact [自定义指令]
  → 尝试 Session Memory Compaction（如果有 memory 且无自定义指令）
  → 如果 reactive-only 模式，走 reactive 路径
  → 否则：先 microcompact，再 legacy full compact
  → 执行 postCompactCleanup
```

### 2.3 Legacy Full Compact 算法

1. **预处理**：执行 PreCompact hooks → 剥离图片 → 剥离附件
2. **调用 LLM**：优先 forked agent（prompt cache sharing），回退 streaming
3. **后处理**：剥离 `<analysis>` 只保留 `<summary>` → 包装为 user message → 注入最近文件（最多 5 个，每个最多 5000 tokens）、plan、skills 等附件
4. **构建输出**：`[boundaryMarker, summaryMessages, fileAttachments, hookResults]`

### 2.4 Compact Prompt（原文）

```
CRITICAL: Respond with TEXT ONLY. Do NOT call any tools.
- Do NOT use Read, Bash, Grep, Glob, Edit, Write, or ANY other tool.
- Tool calls will be REJECTED and will waste your only turn.
```

摘要要求 9 个 section：
1. Primary Request and Intent
2. Key Technical Concepts
3. Files and Code Sections（包含完整代码片段）
4. Errors and fixes
5. Problem Solving
6. All user messages（非 tool_result 的用户消息）
7. Pending Tasks
8. Current Work（包含文件名和代码片段）
9. Optional Next Step（包含对话的直接引用）

输出格式：
```xml
<analysis>
[思考过程]
</analysis>

<summary>
[结构化摘要]
</summary>
```

### 2.5 Microcompact 三种模式

| 模式 | 触发 | 机制 |
|------|------|------|
| Time-based | 距上一条 assistant 消息 > 60 分钟 | 直接修改消息内容 |
| Cached | 每次请求前 | 通过 API cache_edits 机制，不修改本地消息 |
| API Microcompact | 服务端 context management | trigger 阈值 180K tokens，保留目标 40K |

可清理的工具：Read, Bash, Grep, Glob, WebSearch, WebFetch, Edit, Write

### 2.6 Session Memory Compaction

- 后台持续提取结构化 notes（通过 post-sampling hook）
- 压缩时直接用 session memory 内容作为摘要，无需额外 LLM 调用
- 保留最近消息原文：min 10K tokens + 5 条 text-block 消息，max 40K tokens
- Session memory 模板 sections：Current State, Task specification, Files and Functions, Workflow, Errors & Corrections, Learnings, Key results, Worklog

### 2.7 自动触发条件

```
autoCompactThreshold = effectiveContextWindow - 13,000
effectiveContextWindow = contextWindow - min(maxOutputTokens, 20,000)

// 200K context window 模型: threshold = 200,000 - 20,000 - 13,000 = 167,000 tokens
```

熔断：连续失败 3 次停止尝试。

### 2.8 Post-Compact 清理

- 重置 microcompact 状态
- 重置 context collapse 状态
- 清除用户上下文缓存
- 清除 CLAUDE.md 文件缓存
- 清除系统 prompt 段落
- 清除分类器审批
- **故意不清除 invoked skill 内容**（需跨 compact 保留）

### 2.9 消息处理管线

```
contentReplacement (工具结果预算裁剪)
  → snipCompact (HISTORY_SNIP)
    → microcompact (清除旧工具结果)
      → contextCollapse (折叠已完成的上下文)
        → autocompact (自动压缩)
          → API call
```

### 2.10 关键文件

| 文件 | 作用 |
|------|------|
| `src/commands/compact/index.ts` | 命令注册入口 |
| `src/commands/compact/compact.ts` | 手动命令主入口 |
| `src/services/compact/compact.ts` | 核心压缩逻辑 |
| `src/services/compact/prompt.ts` | 压缩 prompt 模板 |
| `src/services/compact/microCompact.ts` | Microcompact 主逻辑 |
| `src/services/compact/sessionMemoryCompact.ts` | SM 压缩 |
| `src/services/compact/autoCompact.ts` | 自动触发 |
| `src/services/compact/postCompactCleanup.ts` | 后清理 |

---

## 三、/loop 命令

### 3.1 架构

`/loop` 是一个 bundled skill，本身不维护定时器。它把用户输入解析成结构化 prompt，交给模型，让模型通过 `CronCreate` 工具写入 cron 调度系统。

```
Skill 层: loop.ts → 解析输入 → 构建 prompt
工具层: CronCreate/CronDelete/CronList → 模型调用的工具
调度层: cronScheduler.ts → 1s tick + chokidar 文件监听 + 锁管理
持久化层: scheduled_tasks.json → 任务存储
cron 解析层: cron.ts → 5 字段 cron 解析
React 接入层: useScheduledTasks.ts → REPL 挂载调度器
```

### 3.2 USAGE_MESSAGE（原文）

```
Usage: /loop [interval] <prompt>

Run a prompt or slash command on a recurring interval.

Intervals: Ns, Nm, Nh, Nd (e.g. 5m, 30m, 2h, 1d). Minimum granularity is 1 minute.
If no interval is specified, defaults to 10m.

Examples:
  /loop 5m /babysit-prs
  /loop 30m check the deploy
  /loop 1h /standup 1
  /loop check the deploy          (defaults to 10m)
  /loop check the deploy every 20m
```

### 3.3 buildPrompt() 完整原文

```markdown
# /loop — schedule a recurring prompt

Parse the input below into `[interval] <prompt…>` and schedule it with CronCreate.

## Parsing (in priority order)

1. **Leading token**: if the first whitespace-delimited token matches `^\d+[smhd]$` (e.g. `5m`, `2h`), that's the interval; the rest is the prompt.
2. **Trailing "every" clause**: otherwise, if the input ends with `every <N><unit>` or `every <N> <unit-word>` (e.g. `every 20m`, `every 5 minutes`, `every 2 hours`), extract that as the interval and strip it from the prompt. Only match when what follows "every" is a time expression — `check every PR` has no interval.
3. **Default**: otherwise, interval is `10m` and the entire input is the prompt.

If the resulting prompt is empty, show usage `/loop [interval] <prompt>` and stop — do not call CronCreate.

Examples:
- `5m /babysit-prs` → interval `5m`, prompt `/babysit-prs` (rule 1)
- `check the deploy every 20m` → interval `20m`, prompt `check the deploy` (rule 2)
- `run tests every 5 minutes` → interval `5m`, prompt `run tests` (rule 2)
- `check the deploy` → interval `10m`, prompt `check the deploy` (rule 3)
- `check every PR` → interval `10m`, prompt `check every PR` (rule 3 — "every" not followed by time)
- `5m` → empty prompt → show usage

## Interval → cron

Supported suffixes: `s` (seconds, rounded up to nearest minute, min 1), `m` (minutes), `h` (hours), `d` (days). Convert:

| Interval pattern      | Cron expression     | Notes                                    |
|-----------------------|---------------------|------------------------------------------|
| `Nm` where N ≤ 59   | `*/N * * * *`     | every N minutes                          |
| `Nm` where N ≥ 60   | `0 */H * * *`     | round to hours (H = N/60, must divide 24)|
| `Nh` where N ≤ 23   | `0 */N * * *`     | every N hours                            |
| `Nd`                | `0 0 */N * *`     | every N days at midnight local           |
| `Ns`                | treat as `ceil(N/60)m` | cron minimum granularity is 1 minute  |

**If the interval doesn't cleanly divide its unit** (e.g. `7m` → `*/7 * * * *` gives uneven gaps at :56→:00; `90m` → 1.5h which cron can't express), pick the nearest clean interval and tell the user what you rounded to before scheduling.

## Action

1. Call CronCreate with:
   - `cron`: the expression from the table above
   - `prompt`: the parsed prompt from above, verbatim (slash commands are passed through unchanged)
   - `recurring`: `true`
2. Briefly confirm: what's scheduled, the cron expression, the human-readable cadence, that recurring tasks auto-expire after 7 days, and that they can cancel sooner with CronDelete (include the job ID).
3. **Then immediately execute the parsed prompt now** — don't wait for the first cron fire. If it's a slash command, invoke it via the Skill tool; otherwise act on it directly.

## Input

{用户原始输入}
```

### 3.4 CronCreate 工具描述（原文）

```
Schedule a prompt to be enqueued at a future time. Use for both recurring schedules and one-shot reminders.

Uses standard 5-field cron in the user's local timezone: minute hour day-of-month month day-of-week. "0 9 * * *" means 9am local — no timezone conversion needed.

## One-shot tasks (recurring: false)

For "remind me at X" or "at <time>, do Y" requests — fire once then auto-delete.
Pin minute/hour/day-of-month/month to specific values:
  "remind me at 2:30pm today to check the deploy" → cron: "30 14 <today_dom> <today_month> *", recurring: false

## Recurring jobs (recurring: true, the default)

For "every N minutes" / "every hour" / "weekdays at 9am" requests:
  "*/5 * * * *" (every 5 min), "0 * * * *" (hourly), "0 9 * * 1-5" (weekdays at 9am local)

## Avoid the :00 and :30 minute marks when the task allows it

Every user who asks for "9am" gets `0 9`, and every user who asks for "hourly" gets `0 *` — which means requests from across the planet land on the API at the same instant. When the user's request is approximate, pick a minute that is NOT 0 or 30:
  "every morning around 9" → "57 8 * * *" or "3 9 * * *" (not "0 9 * * *")
  "hourly" → "7 * * * *" (not "0 * * * *")

## Durability

By default (durable: false) the job lives only in this Claude session — nothing is written to disk, and the job is gone when Claude exits. Pass durable: true to write to .claude/scheduled_tasks.json so the job survives restarts. Only use durable: true when the user explicitly asks for the task to persist.

## Runtime behavior

Jobs only fire while the REPL is idle (not mid-query). Durable jobs persist to .claude/scheduled_tasks.json and survive session restarts. The scheduler adds a small deterministic jitter: recurring tasks fire up to 10% of their period late (max 15 min); one-shot tasks landing on :00 or :30 fire up to 90s early.

Recurring tasks auto-expire after 7 days — they fire one final time, then are deleted.
```

### 3.5 调度机制

- **定时器**：`setInterval(check, 1000)`，每秒 tick
- **文件监听**：chokidar 监听 `scheduled_tasks.json`，变更时重新加载
- **锁机制**：`O_EXCL` 原子创建 `.claude/scheduled_tasks.lock`，避免多会话双触发
- **任务过期**：recurring 任务默认 7 天后老化（最后一轮触发后删除）

### 3.6 任务生命周期

| 阶段 | 行为 |
|------|------|
| 创建 | 用户输入 → skill 生成 prompt → 模型调用 CronCreate → 写入 scheduled_tasks.json |
| 执行 | 调度器 tick → 触发 onFire → 入队命令队列（priority: 'later'）→ 同时追加 system 消息 |
| 恢复 | recurring 从当前时间重算下次触发（非上次计划时间），避免雪崩追跑 |
| 取消 | CronDelete(id) |
| 过期 | 7 天后自动老化删除 |

### 3.7 存储格式（scheduled_tasks.json）

```json
{
  "tasks": [
    {
      "id": "abcd1234",
      "cron": "*/5 * * * *",
      "prompt": "check the deploy",
      "createdAt": 1725598800000,
      "lastFiredAt": 1725599100000,
      "recurring": true,
      "permanent": false
    }
  ]
}
```

### 3.8 错过的任务通知（原文）

```
The following one-shot scheduled task(s) were missed while Claude was not running.
They have already been removed from .claude/scheduled_tasks.json.

Do NOT execute these prompts yet.
First use the AskUserQuestion tool to ask whether to run each one now.
Only execute if the user confirms.

[Every 5 minutes, created 2024-09-06 10:00:00]
```
check the deploy
```
```

### 3.9 关键文件

| 文件 | 作用 |
|------|------|
| `src/skills/bundled/loop.ts` | Skill 定义 + prompt 构建 |
| `src/tools/ScheduleCronTool/CronCreateTool.ts` | 创建任务工具 |
| `src/tools/ScheduleCronTool/CronDeleteTool.ts` | 删除任务工具 |
| `src/tools/ScheduleCronTool/CronListTool.ts` | 列出任务工具 |
| `src/tools/ScheduleCronTool/prompt.ts` | 工具 prompt + 常量 |
| `src/utils/cronScheduler.ts` | 调度器核心 |
| `src/utils/cronTasks.ts` | 任务 CRUD + jitter |
| `src/utils/cronTasksLock.ts` | 调度锁 |
| `src/utils/cron.ts` | cron 解析 |
| `src/hooks/useScheduledTasks.ts` | React 接入 |

---

## 四、/goal 相关机制

### 4.1 核心发现

此代码库中 **不存在独立的 `/goal` 命令**。但支撑"设定目标并自主持续执行"的底层机制全部存在。

### 4.2 Stop Hook 机制

**触发时机**：模型完成一轮回复（无更多 tool_use）后，在 `query.ts:1267` 触发 `handleStopHooks()`。

**Hook 阻止停止的两种方式**：

| 方式 | 触发条件 | 效果 |
|------|----------|------|
| `blockingError` | hook 返回非零退出码 | 作为 user message 注入对话，设置 `stopHookActive: true`，模型被迫再跑一轮 |
| `preventContinuation` | JSON 输出 `continue: false` | 直接终止查询 |

**Stop Hook 输入格式**：
```json
{
  "session_id": "...",
  "transcript_path": "...",
  "cwd": "...",
  "hook_event_name": "Stop",
  "stop_hook_active": true,
  "last_assistant_message": "..."
}
```

**Hook JSON 输出格式**：
```json
{
  "continue": false,
  "stopReason": "...",
  "decision": "block",
  "reason": "blocked because..."
}
```

**Stop Hook 反馈消息**：
```
Stop hook feedback:
{blockingError}
```

### 4.3 自主模式（Autonomous/Proactive Mode）

**Tick 循环**：每完成一轮对话后，如果 proactive 模式激活且队列为空，自动注入 `<tick>` 消息：

```typescript
const tickContent = `<tick>${new Date().toLocaleTimeString()}</tick>`
enqueue({ mode: 'prompt', value: tickContent, priority: 'later', isMeta: true })
```

**Proactive Section 完整原文**：

```markdown
# Autonomous work

You are running autonomously. You will receive `<tick>` prompts that keep you alive between turns — just treat them as "you're awake, what now?" The time in each `<tick>` is the user's current local time.

Multiple ticks may be batched into a single message. This is normal — just process the latest one. Never echo or repeat tick content in your response.

## Pacing

Use the Sleep tool to control how long you wait between actions. Sleep longer when waiting for slow processes, shorter when actively iterating. Each wake-up costs an API call, but the prompt cache expires after 5 minutes of inactivity — balance accordingly.

**If you have nothing useful to do on a tick, you MUST call Sleep.** Never respond with only a status message like "still waiting" or "nothing to do" — that wastes a turn and burns tokens for no reason.

## First wake-up

On your very first tick in a new session, greet the user briefly and ask what they'd like to work on.

## What to do on subsequent wake-ups

Look for useful work. Ask yourself: what don't I know yet? What could go wrong? What would I want to verify before calling this done?

Do not spam the user. If you already asked something and they haven't responded, do not ask again.

## Staying responsive

When the user is actively engaging with you, check for and respond to their messages frequently.

## Bias toward action

Act on your best judgment rather than asking for confirmation.
- Read files, search code, explore the project, run tests, check types, run linters — all without asking.
- Make code changes. Commit when you reach a good stopping point.
- If you're unsure between two reasonable approaches, pick one and go.

## Be concise

Keep your text output brief and high-level. Focus on:
- Decisions that need the user's input
- High-level status updates at natural milestones
- Errors or blockers that change the plan

## Terminal focus

- **Unfocused**: The user is away. Lean heavily into autonomous action.
- **Focused**: The user is watching. Be more collaborative.
```

### 4.4 Auto Mode 指令（原文）

**完整版**：
```markdown
## Auto Mode Active

Auto mode is active. The user chose continuous, autonomous execution. You should:

1. **Execute immediately** — Start implementing right away.
2. **Minimize interruptions** — Prefer making reasonable assumptions over asking questions.
3. **Prefer action over planning** — Do not enter plan mode unless explicitly asked.
4. **Expect course corrections** — The user may provide suggestions at any point.
5. **Do not take overly destructive actions** — Anything that deletes data or modifies shared systems still needs explicit confirmation.
6. **Avoid data exfiltration** — Post messages to chat platforms only if directed.
```

**简化版**（后续注入）：
```
Auto mode still active (see full instructions earlier in conversation). Execute autonomously, minimize interruptions, prefer action over planning.
```

### 4.5 Compact 后自主模式续接（原文）

```
You are running in autonomous/proactive mode. This is NOT a first wake-up — you were already working autonomously before compaction. Continue your work loop: pick up where you left off based on the summary above. Do not greet the user or ask what to work on.
```

### 4.6 Sleep Tool Prompt（原文）

```
Wait for a specified duration. The user can interrupt the sleep at any time.

Use this when the user tells you to sleep or rest, when you have nothing to do, or when you're waiting for something.

You may receive <tick> prompts — these are periodic check-ins. Look for useful work to do before sleeping.

You can call this concurrently with other tools — it won't interfere with them.

Prefer this over `Bash(sleep ...)` — it doesn't hold a shell process.

Each wake-up costs an API call, but the prompt cache expires after 5 minutes of inactivity — balance accordingly.
```

### 4.7 评判模型（Auto Mode Classifier）

使用 `sideQuery`（独立 API 调用）做两阶段安全判断：
- **Stage 1 (fast)**：max_tokens=64，快速 yes/no
- **Stage 2 (thinking)**：chain-of-thought 深度分析
- 输出 `<allow>` 或 `<block>` XML 判定

注意：这是安全权限评判，不是目标完成度评判。

### 4.8 如何构建 /goal

基于现有机制，/goal 的实现路径：

1. 用户设置目标 → 存入某处（内存或文件）
2. 注册 Stop hook → 每轮结束时执行
3. Hook 读取目标 → 调用 sideQuery 评判是否完成
4. 未完成 → 返回 blockingError，模型继续
5. 已完成 → 返回空/成功，模型停止
6. 用 proactive/tick 保持自主循环，用 maxTurns 防止失控

### 4.9 关键文件

| 文件 | 作用 |
|------|------|
| `src/query/stopHooks.ts` | Stop hook 核心调度 |
| `src/utils/hooks.ts` | Hook 执行引擎 |
| `src/constants/prompts.ts` | Proactive section prompt |
| `src/cli/print.ts` | Tick 生成 |
| `src/tools/SleepTool/prompt.ts` | Sleep 工具 prompt |
| `src/utils/messages.ts` | Auto mode 指令消息 |
| `src/utils/permissions/yoloClassifier.ts` | 评判分类器 |
| `src/entrypoints/sdk/coreSchemas.ts` | Hook schema 定义 |
| `src/services/compact/prompt.ts` | Compact 后自主续接 prompt |

---

## 五、总结

### 命令系统核心设计原则

1. **Skill 即 Prompt** — `/loop` 等 prompt 类型命令的本质是一段精心构造的 markdown 指令，让模型调用工具完成工作
2. **分层解耦** — 命令注册、解析、分发、执行各层独立
3. **多来源聚合** — 6 种来源的命令统一注册到同一个查找表
4. **懒加载** — 命令定义只是元数据，实现模块在 `load()` 时才导入
5. **缓存 + 热重载** — memoize 缓存命令列表，提供分层缓存清除

### 三个命令的实现模式

| 命令 | 模式 | 核心机制 |
|------|------|----------|
| `/compact` | 本地执行 | 多层压缩（microcompact → SM → legacy → reactive），LLM 摘要 + 上下文恢复 |
| `/loop` | Prompt → 工具调用 | Skill prompt 指导模型调用 CronCreate → cron 调度系统驱动 |
| `/goal` | 不存在（需构建） | 底层机制齐备：Stop Hook + tick 循环 + sideQuery 评判 |
