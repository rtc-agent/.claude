# 通用编码原则

> _适用于所有仓库、所有语言的基础准则。语言特定的细节在各自的规范文档中展开。_

---

## 1. 命名哲学

命名是代码可读性的基石。一个名字应当是微型文档——在读者看到定义之前，就已经传达了意图。

### 原则

- **名副其实**：名字应当回答"它是什么"或"它做什么"，而不是"它怎么实现"
- **读音可辨**：能用自然语言读出来的名字，才是好名字
- **避免缩写**：除非是领域内广泛认可的（如 `ctx`、`req`、`resp`），否则写全称
- **作用域决定长度**：作用域越大，名字越要清晰完整；局部变量可以简短

### 对比示例

```
❌  bad                                    ✅  good
───────────────────────────────────────────────────────────
d              → elapsed / duration
n              → count / itemCount
tmp            → result / buffer
data           → user / config / response   （具体化）
info           → accountDetails             （具体化）
handleError    → logAndNotify               （描述行为）
process        → validateOrder              （描述具体动作）
flag           → hasExpired / isEnabled     （布尔值带语义）
```

### 命名约定

| 范畴 | Go | TypeScript/JavaScript |
|------|-----|----------------------|
| 包/模块 | 短小写单词 `user`, `event` | kebab-case `user-service` |
| 类型/接口 | PascalCase `UserService` | PascalCase `UserService` |
| 函数/方法 | PascalCase 导出, camelCase 非导出 | camelCase |
| 变量/常量 | camelCase; 常量可用 PascalCase | camelCase; 常量 UPPER_SNAKE_CASE |
| 布尔值 | `is/has/can` 前缀 | `is/has/can` 前缀 |
| 枚举成员 | PascalCase | UPPER_SNAKE_CASE |

### 一致性铁律

同一个概念在整个项目中只使用一个名字。如果 HTTP 响应叫 `response`，就不要在另一处叫 `result` 或 `reply`。跨仓库保持一致。

---

## 2. 注释的纪律

> _好的代码是self-documenting的。注释不是拐杖，而是补充。_

### 写注释的场景

- **为什么（Why）**：业务背景、技术选型的理由、绕过某个限制的说明
- **警告（Warning）**：副作用、线程安全、性能陷阱
- **公开 API**：函数签名无法完整表达语义时
- **复杂算法**：非直觉的数学或业务逻辑

### 不写注释的场景

- 代码本身已经足够清晰（写注释反而是噪音）
- "这段代码做了什么"——如果代码说不清，重构代码，而非加注释
- 被注释掉的代码——删除它，git 记得一切
- 作者署名、日期——git blame 做这件事

### 格式约定

```go
// GetUser 根据 ID 获取用户信息。
//
// 如果用户不存在，返回 ErrUserNotFound，调用方应当处理该错误。
// 注意：此方法不会返回已软删除的用户。
func (s *UserService) GetUser(ctx context.Context, id string) (*User, error) {
```

```typescript
/**
 * 解析 WebSocket 帧并分发到对应的事件处理器。
 *
 * @remarks
 * 该方法会修改 frame 的内部状态（标记为已消费），
 * 因此调用后不应再次传递同一个 frame 实例。
 */
function dispatchFrame(frame: Frame): void {
```

---

## 3. 设计原则

### DRY — 不要重复自己

- 同一段逻辑出现两次，提取为函数/方法
- 但不要过早抽象——等第三次出现再提取（Rule of Three）
- 抽象要放在正确的位置：离使用处最近、离概念源最近

### KISS — 保持简单

- 能不用正则就不用正则
- 能不用反射就不用反射
- 能不用并发就不用并发
- 复杂度是 bug 的温床。每一层间接都需要读者在脑中多维护一层映射

### SOLID 在本项目的体现

| 原则 | 实践 |
|------|------|
| **单一职责** | 一个文件、一个函数、一个 struct 只做一件事 |
| **开闭原则** | 通过接口扩展行为，而非修改已有代码 |
| **里氏替换** | 子类型可以替代父类型而不破坏调用方 |
| **接口隔离** | 小接口优于大接口；依赖需要的，不依赖全部的 |
| **依赖反转** | 依赖抽象（接口），而非具体实现 |

### YAGNI — 不过度设计

不为"将来可能需要"而预留抽象。当需求到来时，在已有的简洁代码上扩展，比预先堆砌的脚手架更安全。

---

## 4. 代码组织

### 文件职责

一个文件应当围绕一个概念展开。当读者打开一个文件，应当在 5 秒内理解"这个文件是关于什么的"。

- **过大文件的信号**：需要频繁滚动才能理解上下文；同一文件中存在不相关的改动
- **拆分原则**：按职责拆分，而非按代码量。一个 100 行的文件如果职责清晰，不需要拆分

### 文件组织顺序

```
1. package 声明 / import
2. 类型定义（struct, interface, type）
3. 常量 / 变量
4. 构造函数 / 工厂函数
5. 公开方法（按逻辑分组）
6. 私有方法
7. init() / 注册函数（如适用）
```

### 导入分组

```go
import (
    // 标准库
    "context"
    "fmt"

    // 第三方库
    "github.com/anthropics/anthropic-sdk-go"

    // 项目内部
    "rtc-agent/server/internal/user"
)
```

---

## 5. 错误处理哲学

### 核心原则

- **不吞错误**：每个错误都必须被处理——返回、记录、或显式忽略（`_ = ...` 并附注释说明原因）
- **在边界处理，在中间传播**：底层函数返回错误，顶层函数处理错误。中间层不要"处理了一半"
- **错误消息要有上下文**：包含"什么操作失败了"和"为什么"，而非只有 "failed"
- **用错误类型，而非错误字符串**：调用方需要基于错误类型做分支决策时，使用自定义错误类型

### 反模式

```go
// ❌ 吞掉错误
data, _ := os.ReadFile(path)

// ❌ 错误消息无上下文
if err != nil {
    return fmt.Errorf("failed")
}

// ❌ 中间层重复包装
if err != nil {
    return fmt.Errorf("failed to get user: %w", err)  // 如果上层还会再包一层
}

// ✅ 正确做法
data, err := os.ReadFile(path)
if err != nil {
    return fmt.Errorf("read config %s: %w", path, err)
}
```

---

## 6. 可读性模式

### 提前返回（Early Return）

```go
// ❌ 嵌套地狱
func process(user *User) error {
    if user != nil {
        if user.IsActive() {
            if user.HasPermission("write") {
                // 真正的逻辑只有3行
                return nil
            }
            return ErrNoPermission
        }
        return ErrInactive
    }
    return ErrNilUser
}

// ✅ 提前返回，主线逻辑一目了然
func process(user *User) error {
    if user == nil {
        return ErrNilUser
    }
    if !user.IsActive() {
        return ErrInactive
    }
    if !user.HasPermission("write") {
        return ErrNoPermission
    }

    // 真正的逻辑
    return nil
}
```

### 减少嵌套

- 用 guard clause 提前退出
- 用早期 continue/break 简化循环
- 循环体超过 20 行时，考虑提取为函数

### 有意义的空行

空行是代码的标点符号——分隔逻辑段落，让读者有喘息的空间。

- 函数开头不空行
- 逻辑段落之间空一行
- 不要连续空两行以上

---

## 7. 函数设计

### 单一职责

一个函数做一件事，做好它。如果函数名需要用 "and" 连接（如 `validateAndSave`），它可能做了太多事。

### 参数控制

- 参数不超过 3 个。超过时，用 struct 封装
- 避免布尔参数——它意味着一个函数在做两件事
- 避免 nil 参数——用 optional 模式或零值替代

```go
// ❌ 布尔参数
func GetUser(id string, withProfile bool) (*User, error)

// ✅ 选项模式
func GetUser(ctx context.Context, id string, opts ...UserOption) (*User, error)
```

### 纯函数优先

无副作用的函数更容易测试、推理、复用。将副作用（I/O、状态变更）推到边界，保持核心逻辑纯粹。

---

## 8. 一致性约定

### 铁律

> 同一个概念，整个项目只用一个名字、一种模式。

- 不混用 `ID` / `Id` / `id` ——选定一个，贯穿始终
- 不混用 `Create` / `Add` / `New` / `Make` ——同一语义只用一个动词
- 不混用 camelCase / snake_case ——同一种语言内保持一致
- 不混用回调风格 / Promise 风格 / async-await ——同一项目选定一种

### 文件命名

同一类型的文件在项目中保持相同的命名模式：

```
✅  user_service.go  / order_service.go  / event_service.go
❌  user_service.go  / OrderHandler.go   / events.go
```

---

## 总结

这些原则不是教条，而是减少认知负担的工具。当你在具体场景中不确定如何抉择时，回到这个根本问题：

> _**哪种写法会让六个月后的自己（或团队中的新成员）更快理解这段代码？**_
