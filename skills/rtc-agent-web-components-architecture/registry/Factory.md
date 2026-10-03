# Factory

createRtcAgent 工厂函数：声明式配置 -> Web Component 实例。

**所属 package**: `component`

## 关键代码文件

- [factory.ts](~/Workspaces/rtc-agent/web-components/packages/component/src/factory.ts) — `createRtcAgent()` 函数（355 行）
- [types/factory.ts](~/Workspaces/rtc-agent/web-components/packages/component/src/types/factory.ts) — `RtcAgentConfig` / `AuthConfig` / `EventCallbacks` / `RtcAgentWithLifecycle` 类型

## 配置到属性的映射

```mermaid
flowchart TD
    Config["RtcAgentConfig"] --> Basic["Basic Properties"]
    Config --> Server["Server Config"]
    Config --> DB["Database Config"]
    Config --> Worker["Worker Config"]
    Config --> Window["Window Config"]
    Config --> ActBar["Activity Bar Config"]
    Config --> Agent["Agent Config"]
    Config --> Auth["Auth Config"]
    Config --> Events["Event Callbacks"]

    Basic --> P1["appLabel, theme, lang, bubbleIcon, logo"]
    Server --> P2["serverURL, redirectURI"]
    DB --> P3["databaseName"]
    Worker --> P4["workerUrl"]
    Window --> P5["windowConfig"]
    ActBar --> P6["activityBarConfig"]
    Agent --> P7["agentConfig (name, description, persona, functions, groups)"]
    Auth --> P8["_pendingAuthConfig / _pendingDynamicAuth / _pendingAuthProvider"]
    Events --> P9["addEventListener + EventBus bridging"]
```

## 三种认证模式检测

```mermaid
flowchart TD
    A["config.auth"] --> B{"'accessToken' in auth?"}
    B -->|Yes| C["StaticTokenAuth"]
    B -->|No| D{"'getToken' in auth<br/>AND NOT 'isLoggedIn'?"}
    D -->|Yes| E["DynamicTokenAuth"]
    D -->|No| F{"'isLoggedIn' in auth?"}
    F -->|Yes| G["AuthProvider"]
    F -->|No| H["No auth config"]

    C --> I["element._pendingAuthConfig"]
    E --> J["element._pendingDynamicAuth"]
    G --> K["element._pendingAuthProvider"]
```

## destroy() 清理流程

```mermaid
sequenceDiagram
    participant Host
    participant Element as rtc-agent
    participant DCB as disconnectedCallback

    Host->>Element: destroy()
    Element->>Element: remove() [triggers disconnectedCallback]
    DCB->>DCB: Clear controllers, listeners, timers
    Element->>Element: _pendingAuthConfig = undefined
    Element->>Element: _pendingDynamicAuth = undefined
    Element->>Element: _pendingAuthProvider = undefined
    Element->>Element: _eventUnsubscribes.forEach(unsub)
    Element->>Element: _eventBusUnsubscribes.forEach(unsub)
    Element->>Element: Set arrays to undefined
    Note over Element: All references released for GC
```

## 跨维度关联

- [[RtcAgentConfig]] — 配置类型详解
- [[ComponentEvents]] — 事件回调映射
- [[EventBus]] — EventBus 桥接
- [[Ready]] — 就绪信号
- [[AuthController]] — 认证配置的最终消费者
- [[ControllerPattern]] — 工厂创建的组件内部使用控制器模式
- [[ContextSystem]] — 工厂创建的组件内部使用 Context 系统
