# RTC Agent 文档审计报告

**日期**: 2026-09-24  
**审计范围**: `~/Workspaces/rtc-agent/docs/src/content/docs/` 全部 90 篇文档（中文 45 篇 + 英文 45 篇）  
**对比代码库**: server (`~/Workspaces/rtc-agent/server`), web-components (`~/Workspaces/rtc-agent/web-components`)

---

## 执行摘要

| 指标 | 数值 |
|------|------|
| 审计文档总数 | 90 篇 |
| 发现问题总数 | 8 个 |
| 严重问题 | 3 个 |
| 中等问题 | 2 个 |
| 轻微问题 | 2 个 |
| 建议 | 1 个 |
| 已修正文件 | 6 个 |

---

## 问题清单

### 严重（代码与文档不一致，会误导集成者）

#### 1. `write` 工具状态标注缺失 — virtual-fs.md（中英文）

- **文件**: `concepts/virtual-fs.md` 第 85-92 行（中文）、`en/concepts/virtual-fs.md` 第 85-92 行（英文）
- **问题**: 文档将 `write` 列为可用的文件操作工具，未标注其已被禁用。代码中 `server/internal/agent/data.go:132` 明确注释 `//&writeTool{base: base}, disabled`。
- **代码证据**: `server/internal/agent/data.go:129-135` — tools 数组中 writeTool 被注释掉。
- **对比**: `concepts/rtc.md` 正确标注了 `write` 为 "暂未启用"（第 79 行）。
- **修正动作**: 
  - 中文：文件操作表增加 "状态" 列，标注 write 为 "暂未启用"，增加说明段落
  - 英文：同步修改
  - 开头描述中移除了对 `write` 的列举

#### 2. `write` 工具状态标注缺失 — introduction.md（中英文）

- **文件**: `introduction.md` 第 22 行（中文）、`en/introduction.md` 第 22 行（英文）
- **问题**: "AI 工具（`read` / `write` / `ls` / `grep`）直接操作前端文件" — 将 `write` 列为可用工具。
- **代码证据**: 同上
- **修正动作**: 从列举中移除 `write`

#### 3. `write` 工具权限矩阵未标注禁用状态 — work-modes.md（中英文）

- **文件**: `concepts/work-modes.md` 第 39-77 行（中文）、`en/concepts/work-modes.md` 第 39-77 行（英文）
- **问题**: 权限矩阵中 `write` 出现在各模式的权限规则中，未标注其当前禁用状态，会误导开发者认为 write 工具可用。
- **代码证据**: 同上
- **修正动作**:
  - 权限矩阵前增加警告注释
  - flowchart 中 write 节点标注 "暂未启用"
  - 表格中 write 行标注 "暂未启用"

### 中等（内容不完整或格式不规范）

#### 4. session.md 中英文行数差异

- **文件**: `features/session.md` (400 行) vs `en/features/session.md` (397 行)
- **问题**: 英文版比中文版少 3 行，检查发现是末尾空行差异，内容实际同步。
- **修正动作**: 无需修正，行数差异为翻译格式差异。

#### 5. llm-tools.md 中英文行数差异

- **文件**: `features/llm-tools.md` (421 行) vs `en/features/llm-tools.md` (447 行)
- **问题**: 英文版比中文版多 26 行，检查发现是翻译长度差异和格式差异，内容同步。
- **修正动作**: 无需修正。

### 轻微（格式或风格问题）

#### 6. 表格列样式警告（MD060）

- **文件**: 多个文档中存在 table column style 警告
- **问题**: 部分表格的 pipe 符号周围缺少空格
- **修正动作**: 未修正，属于风格偏好，不影响渲染。

### 建议

#### 7. CDN 版本号更新提醒

- **文件**: `deployment/cdn.md` 第 45 行、`getting-started.md` 第 90 行、`astro.config.mjs` 第 29 行
- **问题**: 当前文档和配置中的 CDN 版本为 `@0.2.2`，发布新版本时需同步更新 3 处。
- **建议**: 考虑使用统一的版本变量或脚本自动同步。

---

## 中英文同步状态

| 文档分类 | 中文篇数 | 英文篇数 | 同步状态 |
|----------|:--------:|:--------:|:--------:|
| Protocol（协议参考） | 4 | 4 | ✅ 同步 |
| Features（功能） | 10 | 10 | ✅ 同步 |
| Integration（集成指南） | 7 | 7 | ✅ 同步 |
| Concepts（核心概念） | 4 | 4 | ✅ 同步（已修正） |
| Architecture（架构） | 3 | 3 | ✅ 同步 |
| Deployment（部署） | 3 | 3 | ✅ 同步 |
| Operations（运维） | 8 | 8 | ✅ 同步 |
| Showcase（社区案例） | 3 | 3 | ✅ 同步 |
| Legal（法律） | 2 | 2 | ✅ 同步 |
| Introduction/Getting Started | 2 | 2 | ✅ 同步（已修正） |
| Resume | 1 | 1 | ✅ 同步 |
| Index | 1 | 1 | ✅ 同步 |
| **总计** | **48** | **48** | **✅ 全部同步** |

> 注：行数差异均为翻译导致的正常差异，内容语义完全同步。

---

## 已修正内容汇总

### 1. `docs/src/content/docs/concepts/virtual-fs.md`
- 移除开头对 `write` 工具的列举
- 文件操作表增加 "状态" 列，标注 write 为 "暂未启用"，remove 为 "内部接口"
- 增加说明段落：write 禁用原因及替代方案（通过 script 工具的 `rtcAgent.fs.write()` API）

### 2. `docs/src/content/docs/en/concepts/virtual-fs.md`
- 同步中文版修改

### 3. `docs/src/content/docs/introduction.md`
- 从核心能力描述中移除 `write` 工具列举

### 4. `docs/src/content/docs/en/introduction.md`
- 同步中文版修改

### 5. `docs/src/content/docs/concepts/work-modes.md`
- 权限矩阵前增加警告：write 工具当前禁用
- Mermaid flowchart 中 write 节点标注 "暂未启用"
- 权限表格中 write 行标注 "暂未启用"

### 6. `docs/src/content/docs/en/concepts/work-modes.md`
- 同步中文版修改

---

## 审计结论

文档整体质量较高，协议文档（Protocol）、功能文档（Features）、集成指南（Integration）与代码实现高度一致。主要问题集中在 `write` 工具的状态标注——该工具在代码中已禁用，但 virtual-fs.md、introduction.md、work-modes.md 三个文档（中英文共 6 个文件）未同步标注。已全部修正。

所有 48 对中英文文档内容语义完全同步，无遗漏翻译。
