# RTC Agent — Claude Code 配置仓库

本仓库是 [RTC Agent Workspace](https://github.com/rtc-agent) 的 [Claude Code](https://claude.ai/code) 配置中心，存放跨子仓库共享的
skills、agents、commands、hooks、workflows 和 settings。

## 目录结构

```text
.claude/
├── README.md          # 本文件
├── LICENSE            # MIT
├── .gitignore         # 白名单模式，仅放行下方列出的路径
├── settings.json      # Claude Code 项目设置（可选）
├── skills/            # 共享 skills（按目录组织）
├── agents/            # 自定义 subagent 定义（可选）
├── commands/          # 自定义 slash commands（可选）
├── hooks/             # Claude Code hooks（可选）
└── workflows/         # 多 agent 编排脚本（可选）
```

## .gitignore 说明

本仓库的 `.gitignore` 采用**白名单模式**：默认拒绝所有文件，仅允许特定路径被追踪。新增文件时，必须先确认它出现在
`.gitignore` 的允许列表里，否则 `git add` 不会生效。

当前允许的内容：

- 根目录文件：`README.md`、`LICENSE`、`.gitignore`、`settings.json`
- 目录：`agents/`、`commands/`、`hooks/`、`skills/`、`workflows/`（及其子项）

如果要新增一类配置文件，请同步在 `.gitignore` 添加对应的 `!<path>` 规则。

## 已注册的 Skills

| SKILL                                                        | 介绍            |
|--------------------------------------------------------------|---------------|
| [rtc-agent-commitment](skills/rtc-agent-commitment/SKILL.md) | 多仓库工作区的提交规范指南 |

### `rtc-agent-commitment`

多仓库工作区的提交规范指南。覆盖：

- 工作区布局与各仓库的默认分支
- **强制验证**：`cd` 后必须 `pwd` 确认进入目标仓库
- Conventional Commits 格式（type / scope / subject / body / footer）
- AI 协作提交必须包含 `Co-Authored-By: Claude Code <noreply@anthropic.com>` trailer
- 跨仓库变更的拆分提交与 PR 互引方式
- Pre-commit hook（`golangci-lint`、`pnpm typecheck`、API key 扫描、大文件/可执行文件检查）的排查方法
- 常见错误对照表

触发场景：在 RTC Agent Workspace 内执行 `git commit`、`git push`、创建 PR，或跨仓库协调变更时调用。

## 使用方式

Claude Code 在工作区根目录启动时，会自动加载本仓库的 `CLAUDE.md` 与 `skills/` 下的 skill。无需手动导入。

调用特定 skill 示例：

```bash
/rtc-agent-commitment
```

或在对话中直接提及触发条件（例如"准备提交"），Claude 会按规则匹配并加载。
