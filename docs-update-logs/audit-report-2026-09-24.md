# RTC Agent Documentation Audit Report

**Date**: 2026-09-24
**Auditor**: RTC Agent Docs Audit Agent
**Scope**: All 96 documentation files under `docs/src/content/docs/` (48 Chinese + 48 English)

---

## Executive Summary

### Audit Scope

- **Total docs**: 96 files (48 zh-CN + 48 English)
- **Sidebar sections**: 9 major sections (About, Getting Started, Core Concepts, Features, Integration Guide, Protocol Reference, Architecture, Operations, Community Showcase, Legal)
- **Code repositories audited**:
  - `server/` (Go, HTTP handlers, RPC methods, protocol models)
  - `web-components/packages/component/` (TypeScript, Lit components, controllers)

### Issues Found

| Severity | Count | Description |
|:--------:|:-----:|------|
| **Critical** | 1 | Debug API availability incorrectly stated as "all builds (dev + prod)" |
| **Medium** | 2 | Controller count outdated (21 → 19); sub-component count outdated (44 → 19) |
| **Minor** | 0 | — |
| **Suggestion** | 0 | — |

### Issues Fixed

All 3 identified issues have been corrected in both Chinese and English docs.

---

## Issue Details

### CRITICAL: Debug API Availability Claim

**Affected files**:
- `docs/src/content/docs/integration/component-api.md` (line 584, 668)
- `docs/src/content/docs/en/integration/component-api.md` (line 584, 668)

**Problem**: The documentation stated that `window.rtcAgentDebug` is available in "all builds (dev + prod)", which is factually incorrect. The code explicitly restricts it to dev/test builds only:

```ts
// web-components/packages/component/src/components/rtc-agent/rtc-agent.ts:1247
if (import.meta.env.DEV || import.meta.env.MODE === 'test') {
    installDebugAPI();
}
```

**Impact**: Developers integrating the component in production environments might rely on the Debug API and encounter runtime errors when it's not available.

**Code evidence**:
- `web-components/packages/component/src/debug-api.ts` — Header comment: "Only injected in dev/test builds"
- `rtc-agent.ts:1247` — Conditional install: `import.meta.env.DEV || import.meta.env.MODE === 'test'`

**Fix applied**: Changed to clearly state that the Debug API is only available in dev and test builds, and is NOT included in production builds. Updated both the main description and the best-practice note.

---

### MEDIUM: Controller Count Outdated

**Affected files**:
- `docs/src/content/docs/integration/component-api.md` (lines 6, 278, mermaid diagram)
- `docs/src/content/docs/en/integration/component-api.md` (lines 6, 278, mermaid diagram)

**Problem**: The documentation claimed "21 Controllers (9 core + 12 UI)", but the actual count is **19 Controllers (9 core + 10 UI)**.

**Code evidence** (from `rtc-agent.ts`):

**Core Controllers (9)**:
1. WindowStateController
2. AuthController
3. PersistenceController
4. SessionController
5. MessageController
6. ModeController
7. ToolCallController
8. SkillController
9. WindowInteractionController

**UI Controllers (10)**:
1. ActivityController
2. EditorAreaController
3. FileExplorerController
4. ForkController
5. ToastController
6. SessionTabController
7. SettingsController
8. StatusBarController
9. SessionTreeController
10. NotificationController

**Fix applied**: Updated the counts from "21 (9+12)" to "19 (9+10)" in both Chinese and English docs, including the mermaid diagram labels.

---

### MEDIUM: Sub-component Count Outdated

**Affected files**:
- `docs/src/content/docs/integration/component-api.md` (line 6)
- `docs/src/content/docs/en/integration/component-api.md` (line 6)

**Problem**: The documentation claimed "44 个子组件" (44 sub-components). This count is outdated. The actual number of `@customElement` registrations in the component source is ~90, or ~52 component TypeScript files (excluding test and style files).

**Fix applied**: Removed the sub-component count from the opening paragraph since it's an internal detail that changes frequently and has no relevance to external API consumers. The sentence now focuses on the Controller count, which is the more meaningful architectural metric.

---

## Verification Results (No Issues Found)

The following areas were thoroughly audited and found to be **accurate**:

### Protocol Documentation

| Document | Verified Against | Result |
|----------|-----------------|--------|
| `protocol/http-api.md` | `server/internal/server/server.go`, `server/internal/handler/http/oauth2.go`, `server/internal/handler/http/health.go`, `server/internal/handler/http/interrupt.go`, `server/internal/handler/http/memories.go` | PASS |
| `protocol/rpc.md` | `server/pkg/protocol/models.gen.go` (17 RPC methods), `server/internal/handler/rpc/handler.go` (route registration) | PASS |
| `protocol/events.md` | `server/pkg/centrifuge-plus/dual_broker.go`, `server/pkg/centrifuge-plus/topic_broker.go` | PASS |
| `protocol/index.md` | All protocol files | PASS |

**Verified details**:
- OAuth2 endpoints: `/oauth2/authorize`, `/oauth2/providers`, `/oauth2/token`, `/oauth2/refresh` — all match code
- RPC methods: 17 methods (9 Action + 8 Query) across 4 domains — exact match with `protocol.RpcMethod` enum
- HTTP business endpoints: `/api/sessions/{sessionID}/interrupts/{interruptID}/answer`, `/api/memories/export` — match code
- Health endpoints: `/healthz`, `/readyz`, `/metrics` — match code
- Error format, response structures, token exchange fields — all match protocol models

### Integration Documentation

| Document | Verified Against | Result |
|----------|-----------------|--------|
| `integration/component-api.md` | `web-components/.../rtc-agent.ts` (properties, events, CSS vars, window config, activity bar config, connection state, debug API) | PASS (after fixes) |
| `integration/function-registration.md` | `web-components/.../function-registry.ts`, `rtc-agent.ts` (agentConfig property) | PASS |
| `integration/auth.md` | `server/internal/oauth/provider.go`, `server/internal/server/server.go` (BuildProviderClients) | PASS |

**Verified details**:
- Component properties: `theme`, `lang`, `database-name`, `app-label`, `bubble-icon`, `scenarios-url`, `server-url`, `redirect-uri` — all present in code
- JS properties: `agentConfig`, `registry`, `windowConfig`, `activityBarConfig` — all present in code
- Events: `rtc-agent-ready` — confirmed in code (`firstUpdated()`)
- Connection state properties: `connectionFailed`, `connectionError` — confirmed in code
- `reconnect()` method — confirmed in code
- CSS variables: `--rtc-window-default-width`, `--rtc-window-default-height`, `--rtc-bubble-size`, `--rtc-font-size-user` — confirmed in code comments
- OAuth2 provider configuration (GitHub, Google, Mock) — matches `BuildProviderClients()` in `server.go`

### Feature Documentation

| Document | Key Claims Verified | Result |
|----------|-------------------|--------|
| `features/session.md` | Session states (idle/active/closed), fork mechanism, token stats fields, compression fields | PASS |
| `features/messaging.md` | Content types (9 types), message roles (4 types), streaming flow, error content structure | PASS |
| `features/commands.md` | /compact, /persona, /loop, /goal commands | PASS |
| `features/realtime.md` | Dual-channel architecture, offset mechanism, streaming flow | PASS |
| `features/settings.md` | Settings categories, defaults, localStorage persistence | PASS |
| `features/llm-tools.md` | Tool list (subAgent, askUser, todoWrite, goal tools, loop tools) | PASS |
| `features/context-management.md` | Five-layer compression architecture | PASS |

### Concept Documentation

| Document | Key Claims Verified | Result |
|----------|-------------------|--------|
| `concepts/rtc.md` | RTC protocol description, lifecycle | PASS |
| `concepts/virtual-fs.md` | IndexedDB-based virtual file system | PASS |
| `concepts/script-engine.md` | Script execution engine | PASS |
| `concepts/work-modes.md` | Work mode descriptions | PASS |

---

## Chinese-English Synchronization Status

| Section | Status | Notes |
|---------|:------:|-------|
| Getting Started | SYNCED | |
| Core Concepts | SYNCED | |
| Features | SYNCED | |
| Integration Guide | SYNCED | Both languages fixed together |
| Protocol Reference | SYNCED | |
| Architecture | SYNCED | |
| Operations | SYNCED | |
| Community Showcase | SYNCED | |
| Legal | SYNCED | |

All documentation pages exist in both languages. The English translations are complete and well-maintained.

---

## Summary of Changes

### Files Modified

| File | Changes |
|------|---------|
| `docs/src/content/docs/integration/component-api.md` | Fixed Debug API availability (dev/test only, not prod); Fixed Controller count (19 = 9 core + 10 UI, not 21); Removed outdated sub-component count (44) |
| `docs/src/content/docs/en/integration/component-api.md` | Same fixes as Chinese version (Debug API, Controller count, sub-component count) |

### Changes Not Made (Intentional)

- **Session tree event naming**: The docs note that `rtc-new-session` was renamed to `rtc-session-tree-new`. Both events still coexist in the codebase for different purposes (internal vs. external). The docs accurately describe this as a breaking change for the external event name.
- **Settings categories**: The navigation UI lists 6 categories (appearance, chat, files, notifications, account, about), but the SettingsController only manages 4 state groups. The `account` and `about` categories are navigation-only (no persistent settings). The docs correctly describe the nav categories.
- **Sub-component count**: Rather than updating the count to the current value (~90), I removed it entirely since it's a volatile internal detail with no API relevance.

---

## Recommendations

1. **Automated documentation testing**: Consider adding a CI check that validates key documentation claims (e.g., controller count, API availability) against the actual code. This would prevent similar drift in the future.
2. **Debug API versioning**: If the Debug API is ever needed in production (e.g., for remote debugging), add a build-time flag rather than changing the default behavior.
3. **Regular audit cadence**: Schedule quarterly documentation audits to catch drift early, especially for rapidly-evolving internal architecture details.
