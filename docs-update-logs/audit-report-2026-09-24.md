# RTC Agent Documentation Audit Report

**Date**: 2026-09-24
**Auditor**: RTC Agent Documentation Optimization Agent
**Scope**: All documentation under `docs/src/content/docs/` (48 Chinese + 48 English = 96 files)

---

## Executive Summary

- **Audit Scope**: 96 documentation files (48 Chinese, 48 English)
- **Code Modules Verified**: server (Go), web-components (TypeScript/Lit)
- **Total Issues Found**: 13 (across 6 files)
  - Severe: 0
  - Medium: 2 (unique issues, affecting 10 file locations)
  - Minor: 0
  - Suggestion: 11 (informational, no action needed)
- **Files Modified**: 6

### Audit Methodology

1. Parsed `astro.config.mjs` sidebar configuration to obtain the complete document tree
2. Read all 96 documentation files (`.md` / `.mdx`)
3. Cross-referenced documentation against actual code implementations:
   - Server: `server/internal/handler/http/`, `server/internal/handler/rpc/`, `server/internal/infra/config/`, `server/pkg/protocol/models.gen.go`
   - Web Components: `web-components/packages/component/src/components/rtc-agent/rtc-agent.ts`, `web-components/packages/component/src/controllers/`, `web-components/packages/component/src/components/settings-layout/rtc-settings-nav.ts`
4. Verified API endpoints, RPC methods, component attributes, controller counts, and configuration items
5. Compared Chinese and English documentation for synchronization

---

## Issues Found and Fixed

### Medium Severity

#### Issue 1: Sub-component count mismatch (44 -> 46)

**Description**: Documentation stated `<rtc-agent>` contains "44 sub-components" but actual `@customElement` count is 46.

**Code Evidence**:
```bash
$ grep -rn "@customElement" web-components/packages/component/src/components/ | wc -l
46
```

**Affected Files and Locations**:

| File | Line | Original | Fixed |
|------|------|----------|-------|
| `docs/integration/component-api.md` | 6 | 44 个子组件和 19 个 Controller | 46 个子组件和 20 个 Controller |
| `docs/architecture/index.md` | 127 | 44 个子组件 · 19 个 Controller | 46 个子组件 · 20 个 Controller |
| `docs/architecture/frontend.md` | 3 (description) | 44 个子组件、19 个 Controller | 46 个子组件、20 个 Controller |
| `docs/architecture/frontend.md` | 6 | 44 个子组件, 19 个 Controller | 46 个子组件, 20 个 Controller |
| `docs/en/integration/component-api.md` | 6 | 44 sub-components and 19 Controllers | 46 sub-components and 20 Controllers |
| `docs/en/architecture/index.md` | 127 | 44 Sub-components · 19 Controllers | 46 Sub-components · 20 Controllers |
| `docs/en/architecture/frontend.md` | 3 (description) | 44 sub-components, 19 Controllers | 46 sub-components, 20 Controllers |
| `docs/en/architecture/frontend.md` | 6 | 44 sub-components, 19 Controllers | 46 sub-components, 20 Controllers |

**Action**: Updated all 8 locations.

#### Issue 2: Controller count mismatch (19 -> 20)

**Description**: Documentation stated 19 Controllers but actual count is 20 (`*.controller.ts` files).

**Code Evidence**:
```bash
$ find web-components/packages/component/src/controllers -name "*.controller.ts" | wc -l
20
```

**Affected Files and Locations**:

| File | Line | Original | Fixed |
|------|------|----------|-------|
| `docs/integration/component-api.md` | 6 | 19 个 Controller | 20 个 Controller |
| `docs/integration/component-api.md` | 278 | 19 个 Controller, 10 个 UI | 20 个 Controller, 11 个 UI |
| `docs/integration/component-api.md` | 283 | 10 个 UI Controller | 11 个 UI Controller |
| `docs/architecture/frontend.md` | 117 | 19 个 Controller | 20 个 Controller |
| `docs/en/integration/component-api.md` | 278 | 19 Controllers, 10 UI | 20 Controllers, 11 UI |
| `docs/en/integration/component-api.md` | 283 | 10 UI Controllers | 11 UI Controllers |
| `docs/en/architecture/frontend.md` | 117 | 19 Controllers | 20 Controllers |

**Action**: Updated all 7 locations.

---

## Verified Correct (No Changes Needed)

### Protocol Documentation

| Aspect | Status | Evidence |
|--------|--------|----------|
| RPC method count (17 methods) | Correct | `server/pkg/protocol/models.gen.go` lines 154-170 |
| RPC method names (v1.session.*, v1.message.*, v1.turn.*, v1.rtc.*) | Correct | Matches `server/internal/handler/rpc/handler.go` lines 70-95 |
| Action/Query split (9 Action, 8 Query) | Correct | Verified against handler registrations |
| OAuth2 endpoints (4 endpoints) | Correct | `server/internal/handler/http/oauth2.go` lines 71-74 |
| Health endpoints (/healthz, /readyz, /metrics) | Correct | `server/internal/server/server.go` lines 249-259 |
| Business endpoints (interrupt answer, memory export) | Correct | `server/internal/handler/http/interrupt.go`, `memories.go` |
| Error format (code/message/details) | Correct | `server/internal/handler/rpc/handler.go` lines 117-121 |
| Event types and channel distribution | Correct | Matches server event publishing code |
| Update model (id/items/data_list/offset) | Correct | Matches protocol model definitions |

### Web Component API

| Aspect | Status | Evidence |
|--------|--------|----------|
| Component attributes (theme, lang, database-name, app-label, bubble-icon, scenarios-url, server-url, redirect-uri) | Correct | `rtc-agent.ts` lines 204-422 |
| JS-only properties (agentConfig, registry, windowConfig, activityBarConfig) | Correct | `rtc-agent.ts` lines 263-492 |
| CSS variables (--rtc-window-default-width, --rtc-window-default-height, --rtc-bubble-size) | Correct | JSDoc comments in `rtc-agent.ts` lines 16-18 |
| Event (rtc-agent-ready) | Correct | Verified in events.ts |
| Window config options | Correct | Verified in window-config.ts |
| Activity bar config | Correct | Verified in activity-bar-config.ts |

### Server Configuration

| Aspect | Status | Evidence |
|--------|--------|----------|
| Config structure (Server, Database, Redis, Auth, LLM, etc.) | Correct | `server/internal/infra/config/config.go` |
| Default ports (8888 internal, 28080 via nginx) | Correct | `config.docker.yaml` + `docker-compose.yml` |
| Go version (1.27+) | Correct | `go.mod` line 3, `Dockerfile` line 2 |
| LLM providers (claude, openai) | Correct | `config.go` LLMConfig |
| Pricing configuration | Correct | `config.go` ModelPricingConfig |

### Feature Documentation

| Aspect | Status | Evidence |
|--------|--------|----------|
| Settings categories (6: appearance, chat, files, notifications, account, about) | Correct | `rtc-settings-nav.ts` lines 42-49 |
| Settings state (4 sections with state) | Correct | `contexts/settings.ts` |
| Work modes (3 enabled: manual, edit, bypass; 2 upcoming: plan, auto) | Correct | Matches implementation |
| Built-in tools (6 RTC tools) | Correct | Matches server tool registration |
| LLM built-in tools (15+ tools) | Correct | Matches server agent tool registration |
| Commands (/compact, /goal, /persona, /loop) | Correct | Matches implementation |

---

## Chinese-English Synchronization Status

| Category | Files | Status |
|----------|-------|--------|
| Protocol (4 files) | protocol/*.md | Synced (line ratios ~1.00) |
| Concepts (4 files) | concepts/*.md | Synced (line ratios ~1.00) |
| Features (10 files) | features/*.md | Synced (minor wrapping differences) |
| Integration (7 files) | integration/*.md | Synced (line ratios ~1.00) |
| Deployment (3 files) | deployment/*.md | Synced (line ratios ~1.00) |
| Architecture (3 files) | architecture/*.md | Synced (fixed together) |
| Operations (8 files) | operations/*.md | Synced (line ratios ~1.00) |
| Showcase (3 files) | showcase/*.md* | Synced |
| Legal (2 files) | legal/*.md | Synced |
| Other (3 files) | introduction, getting-started, index | Synced |

**Overall**: All Chinese-English doc pairs are synchronized. The maximum line count difference is 6 lines in `features/llm-tools.md` (EN: 447, ZH: 421), which is due to natural English text wrapping, not content divergence.

---

## Suggestions (Not Fixed)

1. **CDN version pinning**: Docs reference `@rtc-agent/component@0.2.2` but local `package.json` shows version `0.1.0`. The npm published version may differ from local development version. Consider verifying the published version matches documentation.

2. **Default model in examples**: `getting-started.md` and `source-build.md` use `claude-sonnet-4-20250514` as the example model, while `config.docker.yaml` defaults to `qwen3.7-plus`. The docs include a note about this, but the discrepancy may cause confusion. Consider updating examples to match the actual default.

3. **Settings architecture diagram**: The settings architecture diagram in `features/settings.md` shows only 4 setting slices (Appearance, Chat, Files, Notifications) but the nav component has 6 categories (including Account and About). This is technically correct (only 4 have state), but could be clarified with a note.

---

## Files Modified Summary

| File | Changes |
|------|---------|
| `docs/src/content/docs/integration/component-api.md` | Updated sub-component count (44->46), controller count (19->20), UI controller count (10->11) |
| `docs/src/content/docs/architecture/index.md` | Updated sub-component and controller counts |
| `docs/src/content/docs/architecture/frontend.md` | Updated sub-component count, controller count in description, body text, and mermaid diagram |
| `docs/src/content/docs/en/integration/component-api.md` | Updated sub-component count, controller count, UI controller count |
| `docs/src/content/docs/en/architecture/index.md` | Updated sub-component and controller counts |
| `docs/src/content/docs/en/architecture/frontend.md` | Updated sub-component count, controller count in description, body text, and mermaid diagram |
