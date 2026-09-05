# Monorepo 规范

> _Monorepo 的力量在于共享，代价在于纪律。没有纪律的 monorepo 是一团缠在一起的线。_

---

## 1. 包组织

### 目录结构

```
web-components/
├── packages/
│   ├── component/          # 主组件库
│   ├── persistence/        # IndexedDB 持久化层
│   └── shared/             # 共享工具（如有）
├── package.json            # 根配置
├── pnpm-workspace.yaml     # workspace 声明
├── tsconfig.base.json      # 共享 TS 配置
└── pnpm-lock.yaml
```

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

## 5. 新增包流程

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

## 6. 脚本约定

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

## 7. 新增包检查清单

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
