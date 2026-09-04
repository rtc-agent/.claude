---
name: rtc-agent-commitment
description: Use when committing, pushing, or creating PRs in the RTC Agent Workspace — covers the multi-repo layout, dynamic repo discovery, per-repo pre-commit hooks, Conventional Commits format, and cross-repo change coordination
---

# RTC Agent Workspace — Commitment Guide

## Overview

RTC Agent Workspace 是一个**多仓库工作区**：根目录 `/Users/leichujun/Workspaces/rtc-agent/` **不是 git 仓库**，其下多个子目录各自是独立的 git 仓库，分别推送到 GitHub `rtc-agent` 组织。提交代码必须进入对应子仓库操作，不能在根目录 `git commit`。

**强制验证**：`cd` 进入目标仓库后，必须执行 `pwd` 确认工作目录正确（输出应包含目标仓库路径），再继续后续操作。如果 `pwd` 输出不符预期，立即停止并排查，不要继续执行 `git` 命令。

## Dynamic Repo Discovery

**不要硬编码仓库列表**。新增或移除仓库时，通过扫描自动发现：

```bash
# 扫描所有包含 .git 子目录的目录（包括隐藏目录如 .claude）
cd /Users/leichujun/Workspaces/rtc-agent
for d in $(find . -maxdepth 2 -type d -name ".git" 2>/dev/null | sed 's|/.git$||' | grep -v '^.$'); do
  repo_path="${d#./}"
  echo "=== $repo_path ==="
  echo "  remote: $(git -C "$repo_path" remote get-url origin 2>/dev/null)"
  echo "  default branch: $(git -C "$repo_path" symbolic-ref --short refs/remotes/origin/HEAD 2>/dev/null | sed 's|origin/||')"
  echo "  language: $(ls "$repo_path"/go.mod "$repo_path"/package.json "$repo_path"/Cargo.toml 2>/dev/null | head -1)"
  echo ""
done
```

**推断元信息**：
- **默认分支**：从 `git symbolic-ref refs/remotes/origin/HEAD` 读取
- **语言/技术栈**：从根目录文件推断（`go.mod` → Go，`package.json` → JS/TS，`Cargo.toml` → Rust）
- **Pre-commit 检查**：从 `.claude/settings.json` 的 `hooks` 配置读取，或从 `.pre-commit-config.yaml` 读取

**当前已知的仓库**（仅供参考，以动态扫描为准）：
- `server/` — Go 后端，默认分支 `dev`，pre-commit 包含 `golangci-lint`
- `web-components/` — TypeScript pnpm monorepo，默认分支 `dev`，pre-commit 包含 `pnpm typecheck`
- `docs/` — OpenAPI 文档，默认分支 `main`
- `mermaid-live-editor/` — JavaScript，默认分支 `rtc-agent`
- `.claude/` — Claude Code 配置，默认分支 `main`

## When to Use

- 准备 `git commit` / `git push` / 创建 PR 时
- 修改涉及多个子目录，需要拆分提交时
- 不确定 commit message 怎么写时
- pre-commit hook 失败需要排查时

## Commit Message Format (Conventional Commits)

```
<type>(<scope>): <subject>

<body>

<footer>
```

### Type

| Type       | 用途                                   |
|------------|--------------------------------------|
| `feat`     | 新功能                                  |
| `fix`      | 修复 bug                               |
| `docs`     | 仅文档变更                                |
| `style`    | 格式化、分号等不影响逻辑的改动                      |
| `refactor` | 重构（不改功能、不修 bug）                      |
| `perf`     | 性能优化                                 |
| `test`     | 增加或修改测试                              |
| `build`    | 构建系统或外部依赖（`go.mod`、`pnpm-lock.yaml`） |
| `ci`       | CI 配置                                |
| `chore`    | 其它杂项                                 |

### Scope

Scope 取**受影响模块**名，小写、kebab-case。进入仓库后，取仓库内实际的包/模块名：

- Go 仓库：用 Go 包名（如 `centrifuge`, `eino-agent`, `api/handler`）
- JS/TS 仓库：用包名（如 `chat-panel`, `whiteboard`, `shared-ui`）
- OpenAPI 仓库：用 API 分组（如 `rooms`, `sessions`, `auth`）
- 配置仓库：`skills`, `agents`, `config`
- 通用规则：取变更所影响的最内层模块名

### Subject

- 英文、祈使句、首字母小写、末尾无句号
- ≤ 72 字符
- 描述**做了什么**，不是做了什么（"add retry logic" 而非 "added retry logic"）

### Body（可选）

- 空一行后写
- 说明 **why**，不是 what
- 每行 ≤ 72 字符，必要时折行

### Footer

**强制**：所有 AI 协作提交必须包含 `Co-Authored-By` trailer。

```
Co-Authored-By: Claude Code <noreply@anthropic.com>
```

Breaking changes 加：

```
BREAKING CHANGE: <描述>
```

### 完整示例

```
feat(server/centrifuge): add retry with exponential backoff for subscribe

The Centrifuge client sometimes drops connections during network
hiccups; reconnecting immediately causes thundering herd on the broker.
Add jittered exponential backoff (1s → 30s cap) to reduce load.

Co-Authored-By: Claude Code <noreply@anthropic.com>
```

```
fix(web-components/chat-panel): prevent duplicate message on WS reconnect

Reconnect fired `onMessage` before the dedup window cleared, rendering
the last message twice. Track last-seen message id in a ref and skip
if unchanged within 500ms.

Fixes #42

Co-Authored-By: Claude Code <noreply@anthropic.com>
```

## Commit Workflow

### 单仓库变更

```bash
cd server && \                  # ① 进入目标仓库
pwd && \                        # ② 验证工作目录（必须输出含 /server）
git status && \                 # ③ 确认改动范围
git add <paths> && \            # ④ 精准 stage，不要 git add .
git commit -m "..." && \        # ⑤ 遵循 Conventional Commits
git push origin dev             # ⑥ 推到当前分支的远端
```

**关键**：`cd` 后必须 `pwd` 验证。如果 `pwd` 输出不包含目标仓库路径（如 `/server`、`/web-components`），说明 `cd` 失败，**立即停止
**，不要继续执行后续命令。

### 跨仓库变更

当一个特性涉及多个仓库（典型如 server + web-components），**每个仓库单独提交**，通过 footer 关联：

```bash
# 仓库 A
cd server && pwd && git add . && \
git commit -m "feat(api): add room.events streaming endpoint

Required by web-components for real-time event rendering.

Relates-to: rtc-agent/web-components#<PR>
Co-Authored-By: Claude Code <noreply@anthropic.com>" && \
git push origin dev

# 仓库 B
cd ../web-components && pwd && git add . && \
git commit -m "feat(chat-panel): subscribe to room.events stream

Depends-on: rtc-agent/server#<PR>
Co-Authored-By: Claude Code <noreply@anthropic.com>" && \
git push origin dev
```

### PR 关联

- 跨仓库 PR 在 body 里写 `Relates-to: rtc-agent/<repo>#<number>` 或 `Depends-on: ...`
- 依赖方 PR 等被依赖方合并后再合并
- PR body 末尾同样要带 `Co-Authored-By`

## Pre-commit Hook 排查

### server: `golangci-lint` 失败

```bash
cd server
golangci-lint run --new-from-rev=HEAD ./...   # 看具体错误
go fmt ./...                                   # 自动修复格式
```

未装 golangci-lint：

```bash
go install github.com/golangci/golangci-lint/cmd/golangci-lint@latest
```

### web-components: `pnpm typecheck` 失败

```bash
cd web-components
pnpm typecheck    # 看 TS 错误
pnpm tsc --noEmit # 直接调用 tsc 看完整输出
```

### API key 误报

Hook 用正则扫 `sk-*` 等模式。若提交的是示例/文档里的占位符：

1. 改用明确的占位符：`sk-EXAMPLE-KEY-DO-NOT-USE`
2. 或在 pre-commit 里把该文件加到 skip 列表

**绝不要** `--no-verify` 跳过 hook。

### 大文件 / 可执行文件

- > 5MB 的文件 → 改用 Git LFS 或外部存储
- 可执行文件 → 不要提交二进制，改为源码 + 构建脚本

## Common Mistakes

| 错误                                                            | 正确做法                                                                                           |
|---------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| 在根目录 `/Users/leichujun/Workspaces/rtc-agent/` 执行 `git commit` | 先 `cd` 到具体子仓库，**`pwd` 验证后再操作**                                                                 |
| `cd <repo>` 后直接 `git commit`，没检查是否成功进入                   | `cd <repo> && pwd` — 确认输出包含目标仓库路径                                                            |
| `git add .` 一把梭                                               | 用 `git add <paths>` 精准 stage，避免带上 `.DS_Store`、临时文件                                             |
| Commit message 写 "update code" / "fix bug"                    | 用 Conventional Commits，描述**做了什么**                                                              |
| 漏掉 `Co-Authored-By`                                           | 所有 AI 协作提交必须带 trailer                                                                          |
| 跨仓库改动塞一个 commit                                               | 每个仓库独立 commit，footer 里互相引用                                                                     |
| pre-commit 失败就 `git commit --no-verify`                       | 修问题，不要跳过 hook                                                                                  |
| 在错误分支直接提交                                                    | 先通过动态扫描确认该仓库的默认分支，特性另开分支                                                                    |

## Quick Reference

```
┌─ cd <repo> && pwd                 进入仓库并验证（必须）
├─ git status                       看改动
├─ git add <paths>                  精准 stage
├─ git commit -m "<type>(<scope>): <subject>"   Conventional Commits
│    包含 Co-Authored-By trailer
├─ git push origin <branch>         推送
└─ 跨仓库？每个仓库独立走一遍，footer 互引
```
