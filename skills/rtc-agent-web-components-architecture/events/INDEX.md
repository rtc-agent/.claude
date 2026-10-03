# Events

异步信号路径：事件总线与消息流。

## Topics

| Topic | File | Description |
| ------- | ------ | ------------- |
| Publication | [Publication.md](Publication.md) | Centrifuge Publication 事件处理：topic/live 双通道、gap fill、offset 连续性 |
| UIUpdateBus | [UIUpdateBus.md](UIUpdateBus.md) | UI 更新事件总线：entity 过滤、per-listener 队列化、suspend/resume |
| EventBus | [EventBus.md](EventBus.md) | FunctionRegistry 事件总线：function:start/success/error/progress |
| ComponentEvents | [ComponentEvents.md](ComponentEvents.md) | 组件 DOM CustomEvent：rtc-agent-ready、rtc-session-* 等 |
| BusHandler | [BusHandler.md](BusHandler.md) | UIUpdateBus 事件路由分发：entity 分类、Session 结构/轻量变更、DebouncedSessionLoader、Tab 对账（reconcileTabs） |
| CommandPipeline | [CommandPipeline.md](CommandPipeline.md) | Slash 命令管线：parseCommand 解析 -> handleCommand 分发 -> PersistenceLayer 执行 |
| StreamingEvents | [StreamingEvents.md](StreamingEvents.md) | 实时流式消息事件：stream:start/chunk/end，用于长文本生成的增量推送 |
