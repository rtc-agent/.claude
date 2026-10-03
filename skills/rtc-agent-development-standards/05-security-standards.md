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

### 页面加载时 Token 刷新

当 access token 存储在 `localStorage` 且过期时间较短（如 15 分钟）时，用户重新打开 tab 或刷新页面后 token 可能已过期。在发起任何 API 调用之前，必须先尝试刷新 token，避免用户因 token 过期被意外登出。

```typescript
// ✅ 页面加载时检查并刷新过期 token
class AuthProvider {
  async initialize(): Promise<void> {
    const token = localStorage.getItem('rtc_access_token')
    if (!token) return  // 未登录，无需刷新

    if (this._isExpired(token)) {
      // token 已过期，尝试静默刷新
      try {
        await this._refreshToken()
      } catch {
        // 刷新失败（refresh token 也过期），清除登录状态
        this._clearAuth()
        return
      }
    }
    // token 有效（或刷新成功），通知宿主组件可以发起连接
    this.onLogin?.()
  }
}

// ❌ 不检查 token 有效性，直接使用
class AuthProvider {
  async initialize(): Promise<void> {
    const token = localStorage.getItem('rtc_access_token')
    if (token) this.onLogin?.()
    // token 可能已过期，API 调用会 401
  }
}
```

**约束**：

- 页面加载后、发起 API 调用或 WebSocket 连接前，必须检查 access token 是否过期
- 过期时尝试静默刷新，刷新失败则清除登录状态（而非让用户看到 401 错误）
- 刷新逻辑使用共享 Promise 守卫（参见[并发安全模式 - 共享 Promise 守卫](./js-ts/error-handling.md#并发安全模式)），防止并发刷新竞态
- 刷新成功后通过回调（如 `onLogin`）通知宿主组件，而非让宿主轮询

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

## 3. 客户端 S3 临时凭证安全

前端 `S3Client`（`@rtc-agent/client`）通过后端 API 获取 AWS 临时凭证（`accessKeyId`、`secretAccessKey`、`sessionToken`、`expiresAt`），直接操作 S3 对象存储。这引入了一个独立于 OAuth token 的安全面。

### 临时凭证保护

```typescript
// ❌ 临时凭证出现在日志中
log.debug('S3 credentials:', { accessKeyId: creds.access_key_id, sessionToken: creds.session_token })

// ✅ 只记录凭证状态，不记录凭证值
log.debug('S3 credentials refreshed', { expiresAt: creds.expires_at })
```

### 凭证生命周期约束

| 约束 | 说明 |
| --- | --- |
| **只通过后端获取** | 前端通过 `POST /api/credentials/temporary` 获取临时凭证，不直接持有 AWS 长期密钥 |
| **提前刷新** | 在过期前 5 分钟刷新（`CREDENTIAL_REFRESH_MARGIN_MS`），防止操作中途过期 |
| **并发刷新去重** | 用 `refreshPromise` 共享守卫防止并发刷新竞态（完成后清除，失败后也清除以允许重试） |
| **dispose 清除** | 组件销毁时必须将 `credentials`、`expiresAt`、`awsClient` 置 null，防止凭证残留 |

### 凭证刷新去重模式

```typescript
// ✅ 共享 Promise 守卫 + 失败后清除
private refreshPromise: Promise<void> | null = null

private async getClient(): Promise<AWSS3Client> {
  // 1. 凭证有效（提前 5 分钟刷新）→ 直接返回
  if (this.credentials && Date.now() < this.expiresAt - REFRESH_MARGIN) {
    return this.awsClient!
  }

  // 2. 正在刷新 → 等待同一个 Promise
  if (this.refreshPromise) {
    await this.refreshPromise
    return this.awsClient!
  }

  // 3. 发起刷新
  this.refreshPromise = this.refreshCredentials()
  try {
    await this.refreshPromise
  } catch (err) {
    this.refreshPromise = null  // 失败后清除，允许下次重试
    throw err
  }
  this.refreshPromise = null    // 成功后清除，下次获取最新值
  return this.awsClient!
}

// ❌ 不用去重，并发调用可能刷新多次，导致凭证不一致
async getClient() {
  if (expired) await this.refreshCredentials()  // 多个调用者各自刷新
}
```

**约束**：

- 临时凭证（`accessKeyId`、`secretAccessKey`、`sessionToken`）不出现在日志、错误消息、URL 中
- 凭证刷新必须用共享 Promise 守卫（参见[并发安全模式 - 共享 Promise 守卫](./js-ts/error-handling.md#并发安全模式)），完成后和失败后都必须清除 `refreshPromise`
- `dispose()` 必须清除所有凭证引用和 AWS Client 实例
- 前端不请求、不缓存 AWS 长期凭证，只通过后端获取短期临时凭证

---

## 4. 输入校验

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

### 请求体大小限制

所有接受请求体的 HTTP handler 必须在解析前设置 `http.MaxBytesReader`，防止恶意超大请求体耗尽内存（OOM）。

```go
// ✅ 在 Decode/Read 之前限制请求体
func (h *Handler) createUser(w http.ResponseWriter, r *http.Request) {
    r.Body = http.MaxBytesReader(w, r.Body, 1<<20)  // 1MB 上限
    var req CreateUserRequest
    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        http.Error(w, "invalid request", http.StatusBadRequest)
        return
    }
    // ...
}

// ✅ 文件上传：使用实际 Content-Length
r.Body = http.MaxBytesReader(w, r.Body, contentLength)

// ❌ 不限制，恶意请求可导致 OOM
func (h *Handler) createUser(w http.ResponseWriter, r *http.Request) {
    var req CreateUserRequest
    json.NewDecoder(r.Body).Decode(&req)  // 攻击者可发送 1GB JSON
}
```

**约束**：

- 所有 POST/PUT handler 在 `json.NewDecoder().Decode()` 或 `io.ReadAll()` 之前必须调用 `http.MaxBytesReader`
- 静态 API 请求通常限制 1MB（`1<<20`）；文件上传使用实际 `Content-Length`
- 限制值通过常量定义，不硬编码魔法数字（如 `maxXMLRequestBodySize`）
- 超出限制时 `json.Decoder` 返回 `http.MaxBytesError`，handler 返回 413 或 400

### 输入集合大小限制

当请求包含集合类型参数（数组、列表、批量操作项）时，必须限制集合元素数量。恶意客户端可发送包含数万元素的数组，每个元素触发一次数据库查询或外部调用，导致资源耗尽（DoS）。

```go
// ✅ 限制集合大小，防止过度 DB 查询
const MaxFilesPerMessage = 100  // 防止恶意客户端发送大量文件 ID 触发 N 次 DB 查询

func ValidateFilesExist(ctx context.Context, fileRepo repo.FileRepo,
    files []protocol.FileAttachment, userID uuid.UUID) error {
    if len(files) > MaxFilesPerMessage {
        return fmt.Errorf("too many file attachments: %d exceeds maximum %d",
            len(files), MaxFilesPerMessage)
    }
    // ... 后续校验
}

// ✅ 限制批量操作数量（S3 协议兼容）
if len(req.Objects) > 1000 {
    writeS3Error(w, s3ErrTooManyObjects)  // S3 协议限制 1000 keys/request
    return
}

// ✅ 限制字符串字段长度，防止 Redis/DB 存储膨胀
if len(req.Answer) > 10000 {
    http.Error(w, "answer too large", http.StatusBadRequest)
    return
}

// ❌ 不限制集合大小
func (h *Handler) sendMessage(w http.ResponseWriter, r *http.Request) {
    var req SendMessageRequest
    json.NewDecoder(r.Body).Decode(&req)
    // req.Files 可能有 10000 个元素，每个触发一次 DB 查询
    for _, f := range req.Files {
        repo.KeyExists(ctx, f.Fileid)  // 10000 次 DB 查询 → DoS
    }
}
```

**判断标准**：集合中的每个元素是否会触发后续操作（DB 查询、外部 API 调用、文件 I/O）？如果是，必须限制集合大小。限制值通过常量定义，附注释说明理由。

**约束**：

- 新增接收集合参数的 handler 时，评估"每个元素触发的操作成本"，设定合理上限
- 常量命名格式 `Max<Domain><Noun>`（如 `MaxFilesPerMessage`、`MaxBatchDeleteKeys`）
- 限制应在校验边界（handler / primitives 函数入口）尽早执行，在遍历集合之前

### 文件引用 ID 格式校验

当请求中包含文件/对象引用 ID（而非完整路径）时，必须对每个 ID 进行**格式校验**，然后由服务端**构造完整 key**（包含用户归属），最后**批量验证存在性**。三步缺一不可——跳过格式校验可能导致注入攻击，跳过 key 构造可能导致跨用户访问。

```go
// ✅ 三步防御：格式校验 + 服务端构造 key + 存在性检查
var fileIDPattern = regexp.MustCompile(`^[a-f0-9]{32}\.[a-zA-Z0-9]{1,10}$`)

func ValidateFilesExist(ctx context.Context, fileRepo repo.FileRepo,
    files []protocol.FileAttachment, userID uuid.UUID) error {
    // 1. 集合大小限制（防止批量 DoS）
    if len(files) > MaxFilesPerMessage {
        return fmt.Errorf("too many file attachments: %d exceeds maximum %d",
            len(files), MaxFilesPerMessage)
    }

    keys := make([]string, 0, len(files))
    for _, f := range files {
        // 2. 格式校验：严格正则，拒绝非法字符
        if !fileIDPattern.MatchString(f.Fileid) {
            return fmt.Errorf("invalid file ID format: %q", f.Fileid)
        }
        // 3. 服务端构造完整 key（包含用户归属，防止跨用户引用）
        fullKey := rtcoss3.BuildFileKey(userID.String(), f.Fileid)
        keys = append(keys, fullKey)
    }

    // 4. 批量存在性检查（一次 DB 查询，非 N 次）
    exists, err := fileRepo.KeysExist(ctx, keys)
    // ...
}

// ❌ 只用客户端提供的 ID，不校验格式、不构造 key
for _, f := range files {
    exists, _ := repo.KeyExists(ctx, f.Fileid)  // 可能注入路径遍历或跨用户访问
}
```

**约束**：

- 引用 ID 的正则必须足够严格（如 `^[a-f0-9]{32}\.[a-zA-Z0-9]{1,10}$`），只允许合法字符和长度
- 完整 key 必须在服务端构造（`user-{userID}/{fileID}`），不信任客户端提供的完整路径
- 存在性检查必须包含用户归属（key 中包含 userID），防止跨用户文件引用
- 批量查询优于逐条查询（`KeysExist` 而非 N 次 `KeyExists`），参见[输入集合大小限制](#输入集合大小限制)

**适用场景**：文件附件引用、对象存储 key 引用、任何"客户端提供 ID，服务端查找资源"的场景。

### 边界校验，内部信任

校验只在系统边界执行一次——API handler、WebSocket 入口、外部回调。边界内的函数信任已校验的数据，不重复检查。

注意：presigned URL 生成端点也是校验边界——客户端提交的 key 必须在生成 URL 前验证格式和用户归属，而非等到实际访问时才检查。

### 跨系统 TOCTOU 防护

当操作涉及两个独立系统（如数据库 + 对象存储）时，单独检查其中一个系统存在 TOCTOU（Time of Check / Time of Use）竞态——在检查和实际使用之间，另一个系统的状态可能已变化。必须在操作前同时验证两个系统的状态。

```go
// ✅ 双重验证：存储存在性 + 数据库归属
func (h *Handler) handleCopyObject(w http.ResponseWriter, r *http.Request, ...) {
    // 1. 检查对象存储中源文件存在（HeadObject）
    srcMeta, err := h.backend.HeadObject(ctx, bucket, srcKey)
    if err != nil {
        writeS3Error(w, s3ErrNoSuchKey)
        return
    }

    // 2. 验证数据库中该文件属于当前用户（归属校验）
    srcFile, err := h.uc.GetFileRecord(ctx, userID, srcKey)
    if srcFile == nil {
        writeS3Error(w, s3ErrAccessDenied)  // 文件存在但不属于该用户
        return
    }

    // 3. 两步验证通过后执行复制
    h.backend.CopyObject(ctx, srcKey, dstKey)
}

// ❌ 只检查存储，不验证归属
srcMeta, _ := h.backend.HeadObject(ctx, bucket, srcKey)
h.backend.CopyObject(ctx, srcKey, dstKey)  // 任何用户可复制他人文件
```

**约束**：

- 涉及"外部资源 + 数据库记录"的写操作（复制、移动、删除），必须同时验证两个系统的状态
- 验证顺序：先检查外部资源（成本较低），再检查数据库归属（安全边界）
- 即使外部资源存在，没有数据库记录也必须拒绝——防止未注册资源被操作

```go
// ✅ 边界校验：handler 层校验一次
func (h *Handler) uploadObject(w http.ResponseWriter, r *http.Request) {
    key := r.PathValue("key")
    if s3Err := rtcoss3.ValidateKey(key, userID); s3Err != nil {
        writeS3Error(w, s3Err)  // 在边界拒绝
        return
    }
    // 内部函数信任 key 已合法，不再校验
    h.uc.PutObject(r.Context(), key, r.Body)
}

// ✅ 内部函数不做冗余校验
func (uc *Usecase) PutObject(ctx context.Context, key string, body io.Reader) error {
    // key 已由 handler 校验，直接使用
    return uc.repo.Save(ctx, key, body)
}

// ❌ 内部重复校验
func (uc *Usecase) PutObject(ctx context.Context, key string, body io.Reader) error {
    if !isValidKey(key) {  // 多余：handler 已经校验过
        return fmt.Errorf("invalid key")
    }
    return uc.repo.Save(ctx, key, body)
}
```

### 文件上传 Content-Type 白名单

接受文件上传的接口必须通过白名单校验 Content-Type。客户端提供的 Content-Type 可能携带参数（如 `text/plain; charset=utf-8`），校验时必须先剥离参数再匹配。

```go
// ✅ 白名单 + 参数剥离
var AllowedContentTypePrefixes = []string{"image/", "text/"}

func IsAllowedContentType(contentType string) bool {
    // 剥离参数（如 "; charset=utf-8"）
    if idx := strings.Index(contentType, ";"); idx != -1 {
        contentType = strings.TrimSpace(contentType[:idx])
    }
    contentType = strings.ToLower(strings.TrimSpace(contentType))

    for _, prefix := range AllowedContentTypePrefixes {
        if strings.HasPrefix(contentType, prefix) {
            return true
        }
    }
    return false
}

// ❌ 不校验 Content-Type，允许任意文件类型上传
func (h *Handler) upload(w http.ResponseWriter, r *http.Request) {
    h.saveFile(r.Body)  // 不检查 Content-Type
}
```

**规则**：

- 白名单优于黑名单——只允许已知安全的类型
- 剥离参数后再匹配，避免 `text/plain; charset=utf-8` 被误拒
- 空 Content-Type 默认为 `application/octet-stream`，不在白名单中
- 前端上传前也应校验类型（UX 层面），但后端校验是安全边界

### 文件类型校验：Magic Bytes 内容检测

客户端提供的 `Content-Type` 头是**不可信的**——攻击者可以伪造任意 MIME 类型。当文件内容需要传递给外部 API（如 LLM）或存储时，必须通过读取文件头部的 magic bytes 检测真实的 MIME 类型，而非信任 `Content-Type` 头。

```go
import "github.com/gabriel-vasile/mimetype"

// ✅ 通过文件内容检测真实 MIME 类型
func detectMIME(data []byte) string {
    mime := mimetype.Detect(data)
    return mime.String()  // 基于 magic bytes，不信任客户端
}

// ❌ 信任客户端 Content-Type
mimeType := r.Header.Get("Content-Type")  // 可以是任意值
```

**防护场景**：

- **MIME 欺骗攻击**：攻击者将可执行文件伪装为 `image/png` 上传，Content-Type 头显示图片但实际内容是恶意代码
- **LLM API 安全**：将用户文件发送给 LLM API 前，必须确认文件的真实类型（如确认是图片而非脚本）
- **存储分类**：基于真实 MIME 类型选择存储路径和处理管道（图片走压缩管道，文本走截断管道）

**约束**：

- 接受文件内容的模块必须通过 magic bytes 检测真实 MIME 类型，不使用客户端 `Content-Type`
- Content-Type 白名单（上一节）与 magic bytes 检测互补：白名单用于快速拒绝明显不匹配的类型，magic bytes 用于确认真实类型
- 检测库使用 `github.com/gabriel-vasile/mimetype`（已引入），支持 170+ 种格式的 magic bytes 匹配
- 参考实现：`internal/agent/file_loader.go` 的 `LoadImageFromOSS` / `LoadTextFromOSS`

**规则**：

- 校验函数放在 `pkg/` 或 `internal/infra/` 中，供 handler 层调用
- 校验失败返回协议特定的错误类型（参见 [错误处理 - 协议兼容层错误类型](./golang/error-handling.md#协议兼容层的错误类型系统)）
- 内部函数通过注释说明信任前提（如 `// key 已由 handler 校验`）

---

## 5. CSRF 防护

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

## 6. 敏感信息

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

## 7. 依赖安全

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

## 8. 新增代码安全检查清单

提交前，必须逐项确认：

- [ ] 无 `unsafeHTML`（或已用 DOMPurify 清洗）
- [ ] Token 不在 URL 中传递
- [ ] Token 不在日志中打印
- [ ] 页面加载时检查 access token 过期并尝试静默刷新（参见[页面加载时 Token 刷新](#页面加载时-token-刷新)）
- [ ] 后端 RPC 接口校验所有输入参数
- [ ] 接受请求体的 POST/PUT handler 在解析前调用 `http.MaxBytesReader` 限制请求体大小（参见[请求体大小限制](#请求体大小限制)）
- [ ] 接收集合/数组参数的 handler 限制元素数量上限，防止逐元素操作导致资源耗尽（参见[输入集合大小限制](#输入集合大小限制)）
- [ ] 校验在系统边界执行一次，内部函数不重复校验
- [ ] 文件上传接口校验 Content-Type 白名单（剥离参数后匹配）
- [ ] 接受文件内容的模块通过 magic bytes 检测真实 MIME 类型，不信任客户端 Content-Type（参见[文件类型校验](#文件类型校验magic-bytes-内容检测)）
- [ ] 文件操作路径无遍历风险
- [ ] 文件引用 ID 经过严格正则格式校验，完整 key 由服务端构造（含用户归属），不信任客户端提供的完整路径（参见[文件引用 ID 格式校验](#文件引用-id-格式校验)）
- [ ] 跨系统操作（DB + 存储）有 TOCTOU 双重验证
- [ ] 无硬编码密钥/token/密码
- [ ] S3 临时凭证（accessKeyId、sessionToken）不出现在日志、URL、错误消息中
- [ ] S3 客户端 `dispose()` 清除所有凭证引用
- [ ] 错误响应不暴露内部细节
- [ ] CSP 头正确配置
- [ ] 依赖无已知漏洞（`pnpm audit` / `govulncheck`）
- [ ] 锁定文件已提交

---

## 总结

安全是一条红线。每一条约束背后都是一次真实的安全事件教训。

> _"安全不是做完功能后加的一层漆，而是设计时就融入的骨架。"_
