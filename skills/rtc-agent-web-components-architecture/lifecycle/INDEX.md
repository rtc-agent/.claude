# Lifecycle

组件从启动到销毁的生命周期事件。

## Topics

| Topic | File | Description |
| ----- | ---- | ----------- |
| Connection | [Connection.md](Connection.md) | RTCAgentClient WebSocket 连接生命周期（Centrifuge 驱动） |
| Database | [Database.md](Database.md) | IndexedDB (Dexie) 初始化、版本迁移、关闭流程 |
| SharedWorker | [SharedWorker.md](SharedWorker.md) | SharedWorker 启动、多 Tab 连接、Comlink 桥接 |
| WorkerBridge | [WorkerBridge.md](WorkerBridge.md) | 主线程侧 Comlink 桥接：跨域加载、virtualFS 代理、Token 协调 |
| Ready | [Ready.md](Ready.md) | 组件 Ready 信号机制（Promise + CustomEvent 双通道） |
| I18nTheme | [I18nTheme.md](I18nTheme.md) | i18n 语言切换与 Theme 主题初始化的生命周期 |
| PersistenceLayer | [PersistenceLayer.md](PersistenceLayer.md) | 持久化层核心编排器：集成 Client + IndexedDB + OffsetManager |
| RtcProcessor | [RtcProcessor.md](RtcProcessor.md) | RTC 串行处理管线：权限确认、工具执行、结果提交、重试退避 |
| ScriptEngine | [ScriptEngine.md](ScriptEngine.md) | LLM 脚本沙箱：Babel AST 安全变换、受限 API 注入、超时控制 |
| ComponentHierarchy | [ComponentHierarchy.md](ComponentHierarchy.md) | UI 组件树：48 个 Lit 组件的层次结构与 Context 消费关系 |
| SkillSystem | [SkillSystem.md](SkillSystem.md) | Skill 文档生成与 Scenario 加载：FunctionDef/Scenario -> VirtualFS 文档 |
| FloatingPanelController | [FloatingPanelController.md](FloatingPanelController.md) | Overlay 面板定位：@floating-ui/dom 封装，供 mode/command/scenario 面板复用 |
| RootComponentHelpers | [RootComponentHelpers.md](RootComponentHelpers.md) | 根组件逻辑拆分：session-loader / vfs-operations / connection-setup 等 helper 模式 |
| DesignSystem | [DesignSystem.md](DesignSystem.md) | Design Token 体系与主题系统：结构 token + 颜色主题分离，Z-index 层叠规范 |
| ChatLayout | [ChatLayout.md](ChatLayout.md) | 聊天页面编排：两栏布局、Unsaved Tab 管理、Fork 流程、Resize 持久化、全局事件路由 |
| LogoSystem | [LogoSystem.md](LogoSystem.md) | 自定义品牌 Logo 系统：LogoContext + rtc-logo 组件 + 主题感知渲染 |
| OverlaySystem | [OverlaySystem.md](OverlaySystem.md) | 弹出层系统：OverlayManager + 浮动面板 + 动态对话框 + Toast |
