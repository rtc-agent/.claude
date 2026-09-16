# Claude Code 上下文管理配置参考

## 1. Auto Compact 配置

### 阈值计算
```typescript
// 预留输出 token 数
MAX_OUTPUT_TOKENS_FOR_SUMMARY = 20_000

// 缓冲区 token 数
AUTOCOMPACT_BUFFER_TOKENS = 13_000
WARNING_THRESHOLD_BUFFER_TOKENS = 20_000
ERROR_THRESHOLD_BUFFER_TOKENS = 20_000
MANUAL_COMPACT_BUFFER_TOKENS = 3_000

// 计算公式
effectiveContextWindow = contextWindow - MAX_OUTPUT_TOKENS_FOR_SUMMARY
autoCompactThreshold = effectiveContextWindow - AUTOCOMPACT_BUFFER_TOKENS

// 对于 200K 模型
// effectiveContextWindow = 200,000 - 20,000 = 180,000
// autoCompactThreshold = 180,000 - 13,000 = 167,000 tokens
```

### 熔断机制
```typescript
MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES = 3
// 连续失败 3 次后停止尝试自动压缩
```

### 环境变量覆盖
```bash
CLAUDE_CODE_AUTO_COMPACT_WINDOW      # 覆盖上下文窗口大小
CLAUDE_AUTOCOMPACT_PCT_OVERRIDE      # 按百分比设置阈值 (0-100)
DISABLE_COMPACT                      # 禁用所有压缩
DISABLE_AUTO_COMPACT                 # 仅禁用自动压缩（保留手动 /compact）
CLAUDE_CODE_BLOCKING_LIMIT_OVERRIDE  # 覆盖阻塞限制
```

---

## 2. Microcompact 配置

### 可压缩的工具列表
```typescript
COMPACTABLE_TOOLS = [
  'Read',           // FILE_READ_TOOL_NAME
  'Bash',           // SHELL_TOOL_NAMES
  'PowerShell',     // SHELL_TOOL_NAMES
  'Grep',           // GREP_TOOL_NAME
  'Glob',           // GLOB_TOOL_NAME
  'WebSearch',      // WEB_SEARCH_TOOL_NAME
  'WebFetch',       // WEB_FETCH_TOOL_NAME
  'Edit',           // FILE_EDIT_TOOL_NAME
  'Write',          // FILE_WRITE_TOOL_NAME
]
```

### Time-based Microcompact 配置
```typescript
{
  enabled: true,
  gapThresholdMinutes: 60,  // 距上次 assistant 消息超过 60 分钟触发
  keepRecent: 5,            // 保留最近 5 个 tool 结果
}

// 清理后的内容替换为
TIME_BASED_MC_CLEARED_MESSAGE = '[Old tool result content cleared]'
```

### API-level Context Management 配置
```typescript
DEFAULT_MAX_INPUT_TOKENS = 180_000      // 触发阈值
DEFAULT_TARGET_INPUT_TOKENS = 40_000    // 保留最近 40k tokens

// 策略类型
type: 'clear_tool_uses_20250919'   // 清理 tool 结果
type: 'clear_thinking_20251015'    // 清理 thinking 块
```

### Cached Microcompact 配置
```typescript
{
  triggerThreshold: number,  // 触发阈值（tool 结果数量）
  keepRecent: number,        // 保留最近的 tool 结果数量
}
```

---

## 3. Session Memory Compact 配置

### 消息保留策略
```typescript
DEFAULT_SM_COMPACT_CONFIG = {
  minTokens: 10_000,          // 至少保留 10K tokens
  minTextBlockMessages: 5,    // 至少保留 5 条有文本块的消息
  maxTokens: 40_000,          // 最多保留 40K tokens
}
```

### 保留逻辑
从后往前计算，直到满足以下条件之一：
1. `totalTokens >= minTokens` **且** `textBlockMessageCount >= minTextBlockMessages`
2. `totalTokens >= maxTokens`（达到上限就停）

### API 不变量保护
- **tool_use/tool_result 配对**：如果保留的消息中有 `tool_result`，必须包含对应的 `tool_use` 所在的 assistant 消息
- **thinking 块共享**：如果保留的 assistant 消息和前面的消息共享同一个 `message.id`（streaming 拆分），需要包含前面那些消息

---

## 4. Post-Compact 文件恢复配置

```typescript
POST_COMPACT_MAX_FILES_TO_RESTORE = 5        // 最多恢复 5 个文件
POST_COMPACT_TOKEN_BUDGET = 50_000            // 总 token 预算 50K
POST_COMPACT_MAX_TOKENS_PER_FILE = 5_000      // 每个文件最多 5K tokens
POST_COMPACT_MAX_TOKENS_PER_SKILL = 5_000     // 每个 skill 最多 5K tokens
POST_COMPACT_SKILLS_TOKEN_BUDGET = 25_000     // skills 总预算 25K
```

---

## 5. Token 计算方式

### 粗略估算
```typescript
roughTokenCountEstimation(content, bytesPerToken = 4)
// 即 content.length / 4

// 对于 JSON 文件：bytesPerToken = 2（更密集）
// image/document：固定 2000 tokens
```

### 精确计算（优先使用 API 返回的 usage）
```typescript
getTokenCountFromUsage(usage) = 
  input_tokens + 
  cache_creation_input_tokens + 
  cache_read_input_tokens + 
  output_tokens
```

### 消息 token 估算
```typescript
estimateMessageTokens(messages)
// 遍历所有消息，累加各类型 block 的 token 数
// 最后乘以 4/3 作为保守估计
```

---

## 6. 上下文窗口大小

```typescript
MODEL_CONTEXT_WINDOW_DEFAULT = 200_000  // 默认 200K

// 支持的上下文窗口：
// - 1M context：模型名含 [1m] 后缀
// - 200K context：默认
```

---

## 7. 输出 token 管理

```typescript
COMPACT_MAX_OUTPUT_TOKENS = 20_000      // 压缩输出上限
CAPPED_DEFAULT_MAX_TOKENS = 8_000       // 默认输出上限（带 slot-reservation 优化）
ESCALATED_MAX_TOKENS = 64_000           // 升级后的上限
MAX_OUTPUT_TOKENS_DEFAULT = 32_000      // 默认最大输出
MAX_OUTPUT_TOKENS_UPPER_LIMIT = 64_000  // 绝对上限
```
