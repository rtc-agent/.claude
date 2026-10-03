# Monorepo 规范

> _Monorepo 的力量在于共享，代价在于纪律。没有纪律的 monorepo 是一团缠在一起的线。_

---

## 1. 包组织

### 目录结构

```
web-components/
├── packages/
│   ├── component/          # 主组件库（Web Components，面向集成者）
│   ├── persistence/        # IndexedDB 持久化层（Dexie、EntityRepository、UIUpdateBus）
│   ├── client/             # 客户端 SDK（logger、S3 对象存储、日志工具）
│   ├── protocol/           # 协议类型定义（TypeScript 侧的协议常量与类型）
│   └── worker/             # SharedWorker（多 Tab 共享 WebSocket + IndexedDB）
├── package.json            # 根配置
├── pnpm-workspace.yaml     # workspace 声明
├── tsconfig.base.json      # 共享 TS 配置
└── pnpm-lock.yaml
```

### 包职责

| 包 | 职责 | 关键导出 |
| --- | --- | --- |
| **component** | Web Components 组件库，面向集成者 | `@customElement('rtc-*')` 组件、Lit Context 定义、Reactive Controllers、`FileStorage`（高层文件操作 API） |
| **persistence** | IndexedDB 持久化层，离线优先数据管理 | `EntityRepository`、`UIUpdateBus`、`VirtualFS`、`FileCacheRepository`、`FileOpCoordinator` |
| **client** | 客户端 SDK，基础设施层 | `createLogger`（模块日志器）、`S3Client`（S3 对象存储客户端）、日志工具 |
| **protocol** | 协议类型定义，TypeScript 侧的协议常量 | 协议常量、请求/响应类型、事件类型 |
| **worker** | SharedWorker，多 Tab 共享 WebSocket + IndexedDB | Worker 连接管理、跨 Tab 数据同步 |

### `@rtc-agent/client` 的 `S3Client`

`S3Client` 封装了 S3 对象存储操作，是前端与后端 S3 兼容存储交互的唯一入口。

**核心职责**：

- 通过后端 API（`POST /api/credentials/temporary`）获取 AWS 临时凭证，不直接持有长期密钥
- 凭证提前 5 分钟刷新（`CREDENTIAL_REFRESH_MARGIN_MS`），用 `refreshPromise` 共享守卫防止并发刷新竞态
- 自动选择上传策略：小文件（<= 5MB）用 `PutObjectCommand`；大文件用 `@aws-sdk/lib-storage` 的 `Upload` 分片上传
- 支持 `AbortSignal` 取消、上传进度回调、下载进度流式追踪
- `dispose()` 清除所有凭证、AWS Client 实例和 refresh Promise

**约束**：

- 临时凭证（`accessKeyId`、`secretAccessKey`、sessionToken`）不出现在日志中（参见[安全规范 - 客户端 S3 临时凭证安全](../05-security-standards.md#3-客户端-s3-临时凭证安全)）
- 对象键格式统一为 `user-{userId}/{md5Hash}.{ext}`，通过 `buildKey()` 构造，不手动拼接
- 使用 `createLogger('S3Client')` 记录操作日志，scope 为 `S3Client`

### 包名规范

统一使用 `@rtc-agent/` 前缀。

```json
// packages/component/package.json
{
  "name": "@rtc-agent/component"
}

// packages/persistence/package.json
{
  "name": "@rtc-agent/persistence"
}
```

---

## 2. 共享配置

### TypeScript 配置继承

```json
// tsconfig.base.json（根目录）
{
  "compilerOptions": {
    "target": "ES2020",
    "module": "ESNext",
    "moduleResolution": "bundler",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "declaration": true,
    "declarationMap": true,
    "sourceMap": true
  }
}
```

```json
// packages/component/tsconfig.json
{
  "extends": "../../tsconfig.base.json",
  "compilerOptions": {
    "outDir": "./dist",
    "rootDir": "./src"
  },
  "include": ["src"]
}
```

### ESLint / Prettier

根目录定义共享配置，各包继承。

```javascript
// eslint.config.js（根目录）
export default [
  {
    // 共享规则
    rules: {
      'no-unused-vars': 'warn',
      '@typescript-eslint/no-explicit-any': 'error',
    },
  },
]
```

```javascript
// packages/component/eslint.config.js
import baseConfig from '../../eslint.config.js'

export default [
  ...baseConfig,
  {
    // 包特定规则
  },
]
```

---

## 3. 包间依赖

### workspace 协议

包间依赖必须使用 `workspace:*` 协议。

```json
// packages/component/package.json
{
  "dependencies": {
    "@rtc-agent/persistence": "workspace:*"
  }
}
```

**禁止**：

- 不使用具体版本号（如 `"^1.0.0"`）引用内部包
- 不使用 `file:` 或 `link:` 协议

### 通过包入口导入

不跨包直接 import 源码文件。

```typescript
// ✅ 通过包入口导入
import { PersistenceLayer } from '@rtc-agent/persistence'

// ❌ 直接 import 源码
import { PersistenceLayer } from '@rtc-agent/persistence/src/persistence-layer'
```

### 包的入口定义

```json
// packages/persistence/package.json
{
  "main": "./dist/index.js",
  "module": "./dist/index.js",
  "types": "./dist/index.d.ts",
  "exports": {
    ".": {
      "types": "./dist/index.d.ts",
      "import": "./dist/index.js"
    }
  }
}
```

---

## 4. 构建顺序

### pnpm 自动处理

pnpm 根据 `workspace:*` 依赖自动确定构建拓扑。

```bash
# 按依赖顺序构建所有包
pnpm -r build

# 只构建某个包及其依赖
pnpm --filter @rtc-agent/component... build
```

### 版本协调

- 所有包使用相同的 TypeScript 版本
- 共享依赖（如 `lit`）在根 `package.json` 统一管理
- 不锁定内部包的版本（workspace 协议自动处理）

---

## 5. 库包构建：tsup 依赖打包策略

当库包（如 `persistence`、`client`）被嵌入到第三方宿主应用时，其依赖的底层库（如 `dexie`、`microdiff`）可能与宿主应用已有的版本冲突。tsup 默认 externalize `package.json` 中的 `dependencies`，但可通过 `noExternal` 强制打包指定依赖，消除版本冲突风险。

```typescript
// ✅ packages/persistence/tsup.config.ts
import { defineConfig } from 'tsup'

export default defineConfig({
  entry: ['src/index.ts'],
  format: ['esm'],
  dts: true,
  sourcemap: true,
  clean: true,
  // 强制打包：防止宿主应用版本冲突
  noExternal: ['dexie', 'microdiff'],
  // 保持 external：workspace 兄弟包（各有自己的 dist）和过重的第三方（如 @babel/*）
  external: [
    '@rtc-agent/client',
    '@rtc-agent/protocol',
    '@babel/core',
    '@babel/preset-typescript',
  ],
  shims: false,
  target: 'es2022',
})

// ❌ 不打包底层依赖，宿主应用若安装了不同版本的 dexie 会导致运行时冲突
export default defineConfig({
  entry: ['src/index.ts'],
  format: ['esm'],
  dts: true,
  // dexie 未打包，宿主应用的 dexie 版本可能与本包不一致
})
```

**判断标准**：

- **必须打包**：底层基础设施库（IndexedDB wrapper、diff 工具），宿主不太可能使用相同版本，版本不一致会导致运行时错误
- **保持 external**：workspace 兄弟包（各有独立 dist）、体积大的通用库（如 `@babel/*`，浏览器端不太可能冲突）
- **按需评估**：新增底层依赖时，评估宿主冲突风险，风险高的加入 `noExternal`

**约束**：

- `noExternal` 列表需在 commit message 中说明打包理由（参考 commit `aee1dc7`）
- `external` 中的 workspace 包必须已在宿主的依赖树中，否则运行时报 module not found
- `tsup.config.ts` 文件应注释 `noExternal` / `external` 分组的原因，便于后续维护

---

## 6. 新增包流程

### 目录结构模板

```
packages/new-package/
├── src/
│   └── index.ts            # 入口文件
├── package.json
├── tsconfig.json
├── eslint.config.js
└── README.md
```

### package.json 必须字段

```json
{
  "name": "@rtc-agent/new-package",
  "version": "0.0.1",
  "type": "module",
  "main": "./dist/index.js",
  "module": "./dist/index.js",
  "types": "./dist/index.d.ts",
  "exports": {
    ".": {
      "types": "./dist/index.d.ts",
      "import": "./dist/index.js"
    }
  },
  "scripts": {
    "build": "tsc",
    "dev": "tsc --watch",
    "typecheck": "tsc --noEmit"
  },
  "files": ["dist"]
}
```

### README 要求

每个包必须有 README，说明：

- 包的用途
- 导出的 API
- 使用示例

---

## 7. 脚本约定

### 统一脚本命名

| 脚本 | 用途 |
|------|------|
| `build` | 编译为可发布产物 |
| `dev` | 开发模式（watch） |
| `test` | 运行测试 |
| `lint` | ESLint 检查 |
| `typecheck` | TypeScript 类型检查 |
| `format` | Prettier 格式化 |

### 根 package.json 聚合命令

```json
{
  "scripts": {
    "build": "pnpm -r build",
    "dev": "pnpm -r --parallel dev",
    "test": "pnpm -r test",
    "lint": "pnpm -r lint",
    "typecheck": "pnpm -r typecheck",
    "format": "prettier --write \"packages/*/src/**/*.{ts,js}\""
  }
}
```

---

## 8. 新增包检查清单

- [ ] 包名使用 `@rtc-agent/` 前缀
- [ ] `package.json` 包含所有必须字段
- [ ] `tsconfig.json` 继承 `tsconfig.base.json`
- [ ] 入口文件 `src/index.ts` 存在
- [ ] `exports` 字段正确定义
- [ ] 有 README 说明用途和 API
- [ ] 包间依赖使用 `workspace:*`
- [ ] 脚本命名符合约定

---

## 总结

Monorepo 的纪律体现在细节：包名、依赖协议、配置继承、脚本命名。每一个细节的统一，都是未来维护成本的降低。

> _"共享不是免费的。它的代价是纪律，回报是效率。"_
