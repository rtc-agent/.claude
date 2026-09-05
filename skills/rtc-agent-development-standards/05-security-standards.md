# 安全规范

> _安全不是功能，是底线。一次安全事件足以摧毁用户的全部信任。_

---

## 1. XSS 防护

### 前端：警惕 unsafeHTML

Lit 的模板语法自动转义，但 `unsafeHTML` 绕过了这层保护。

```typescript
// ✅ Lit 自动转义，安全
render() {
  return html`<div>${userInput}</div>`  // userInput 被自动转义
}

// ❌ unsafeHTML 不转义，危险
import { unsafeHTML } from 'lit/directives/unsafe-html.js'
render() {
  return html`<div>${unsafeHTML(userInput)}</div>`  // XSS 风险
}

// ✅ 必须使用 unsafeHTML 时，先清洗
import DOMPurify from 'dompurify'
render() {
  return html`<div>${unsafeHTML(DOMPurify.sanitize(userInput))}</div>`
}
```

### 后端：RPC 响应不拼接用户输入

Go 的 RPC 响应通过 JSON 序列化，天然防 XSS。但如果有 HTML 渲染场景（如错误页面），必须转义。

```go
// ✅ JSON 响应，安全
func (h *Handler) getUser(w http.ResponseWriter, r *http.Request) {
    json.NewEncoder(w).Encode(user)  // 自动转义
}

// ❌ HTML 响应拼接用户输入
fmt.Fprintf(w, "<div>%s</div>", userInput)  // XSS 风险

// ✅ HTML 响应使用模板（自动转义）
tmpl.Execute(w, data)  // html/template 自动转义
```

### CSP（Content Security Policy）

后端 HTTP 响应设置 CSP 头，限制脚本来源。

```go
w.Header().Set("Content-Security-Policy", "default-src 'self'; script-src 'self'")
```

---

## 2. OAuth Token 安全

### Access Token 存储

| 方式 | 优点 | 缺点 |
|------|------|------|
| `localStorage` | 简单，跨 tab 共享 | XSS 可窃取 |
| `httpOnly Cookie` | XSS 无法读取 | 需要 CSRF 防护 |

当前项目使用 `localStorage` 存储 access token。如果引入 XSS 风险，token 可能被窃取。

**约束**：

- 严格控制 `unsafeHTML` 使用，减少 XSS 面
- Token 设置短过期时间（当前 15 分钟）
- 敏感操作要求重新认证

### Refresh Token 安全

```typescript
// ❌ refresh token 存 localStorage
localStorage.setItem('rtc_refresh_token', refreshToken)

// ✅ refresh token 应通过 httpOnly cookie 传递
// 或由后端管理，前端只持有短期 access token
```

**约束**：

- Refresh token 不在 URL 中传递
- Refresh token 不在日志中打印
- Refresh token 使用一次后即失效（rotation）

### Token 不写入 URL

```typescript
// ❌ token 在 URL 中
window.location.href = `/callback?token=${accessToken}`

// ❌ token 在 hash 中（虽然不会被服务端日志记录，但仍可通过 referer 泄漏）
window.location.hash = `#token=${accessToken}`

// ✅ 通过 POST body 或 Authorization header 传递
fetch('/api/data', {
  headers: { Authorization: `Bearer ${token}` }
})
```

---

## 3. 输入校验

### 前端校验为 UX，后端校验为安全

```typescript
// 前端：改善用户体验
function validateEmail(email: string): boolean {
  return /^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(email)
}
```

```go
// 后端：安全边界，必须校验
func (h *Handler) createUser(w http.ResponseWriter, r *http.Request) {
    var req CreateUserRequest
    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        http.Error(w, "invalid request", http.StatusBadRequest)
        return
    }

    // 必须校验
    if req.Email == "" || !isValidEmail(req.Email) {
        http.Error(w, "invalid email", http.StatusBadRequest)
        return
    }
    if len(req.Name) > 100 {
        http.Error(w, "name too long", http.StatusBadRequest)
        return
    }
}
```

### 路径遍历防护

VirtualFS 已有路径遍历防护。新增文件操作 API 必须遵循同一模式。

```go
// ✅ 拒绝路径遍历
func validatePath(path string) error {
    if strings.Contains(path, "..") {
        return fmt.Errorf("path traversal not allowed")
    }
    clean := filepath.Clean(path)
    if !strings.HasPrefix(clean, "/") {
        return fmt.Errorf("path must be absolute")
    }
    return nil
}
```

```typescript
// ✅ VirtualFS 已有的防护
class VirtualFS {
  private normalizePath(path: string): string {
    if (path.includes('..')) {
      throw new Error('Path traversal not allowed')
    }
    // ...
  }
}
```

---

## 4. CSRF 防护

### OAuth State 验证

当前已通过 Redis 存储 + Lua `GetDel` 原子操作防止 state 重放。

**约束**：

- 所有 OAuth 流程必须验证 `state` 参数
- `state` 必须一次性使用（读取后立即删除）
- `state` 必须有过期时间（当前 TTL 10 分钟）

### 跨域请求校验

后端校验请求的 `Origin` 头。

```go
func CORSMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        origin := r.Header.Get("Origin")
        if !isAllowedOrigin(origin) {
            http.Error(w, "forbidden origin", http.StatusForbidden)
            return
        }
        next.ServeHTTP(w, r)
    })
}
```

---

## 5. 敏感信息

### 不硬编码

```go
// ❌ 硬编码密钥
const apiKey = "sk-1234567890abcdef"

// ✅ 从配置读取
apiKey := cfg.Anthropic.APIKey
```

```typescript
// ❌ 硬编码
const API_KEY = 'sk-1234567890abcdef'

// ✅ 从环境变量或后端获取
const apiKey = import.meta.env.VITE_API_KEY
```

### 日志不打印敏感字段

```go
// ❌ 日志包含敏感信息
logger.Info("user login", "password", password, "token", token)

// ✅ 只记录标识信息
logger.Info("user login", "user.id", userID)
```

```typescript
// ❌
console.log('auth', { token, refreshToken, password })

// ✅
console.log('auth', { userId, action: 'login' })
```

### 错误响应不暴露内部细节

```go
// ❌ 暴露内部错误
http.Error(w, fmt.Sprintf("database error: %v", err), 500)

// ✅ 返回通用错误
http.Error(w, "internal server error", 500)
// 内部错误只记录到日志
logger.Error("database error", "error", err)
```

---

## 6. 依赖安全

### 定期审计

```bash
# Go
govulncheck ./...

# JS/TS
pnpm audit
```

### 锁定文件必须提交

```bash
# Go
go.sum    # 必须提交

# JS/TS
pnpm-lock.yaml    # 必须提交
```

### 依赖更新策略

- 安全补丁：立即更新
- 次要版本：每个迭代评估
- 主要版本：评估 breaking changes 后升级

---

## 7. 新增代码安全检查清单

提交前，必须逐项确认：

- [ ] 无 `unsafeHTML`（或已用 DOMPurify 清洗）
- [ ] Token 不在 URL 中传递
- [ ] Token 不在日志中打印
- [ ] 后端 RPC 接口校验所有输入参数
- [ ] 文件操作路径无遍历风险
- [ ] 无硬编码密钥/token/密码
- [ ] 错误响应不暴露内部细节
- [ ] CSP 头正确配置
- [ ] 依赖无已知漏洞（`pnpm audit` / `govulncheck`）
- [ ] 锁定文件已提交

---

## 总结

安全是一条红线。每一条约束背后都是一次真实的安全事件教训。

> _"安全不是做完功能后加的一层漆，而是设计时就融入的骨架。"_
