# State

状态管理与状态机。

## Topics

| Topic | File | Description |
| ------- | ------ | ------------- |
| ConnectionState | [ConnectionState.md](ConnectionState.md) | 连接状态机：disconnected / connecting / connected / reconnecting |
| Offset | [Offset.md](Offset.md) | OffsetManager：两级缓存（内存 + IndexedDB）的 offset/epoch 管理 |
| SyncStatus | [SyncStatus.md](SyncStatus.md) | 实体同步状态：pending / synced / failed 的流转 |
| UIUpdate | [UIUpdate.md](UIUpdate.md) | UIUpdateBus：per-entity 队列化分发、suspend/resume 批量控制 |
| MasterLock | [MasterLock.md](MasterLock.md) | Web Locks API 驱动的 Master Tab 选举 |
| ControllerPattern | [ControllerPattern.md](ControllerPattern.md) | Lit ReactiveController 模式：20 个控制器的状态管理与生命周期 |
| ContextSystem | [ContextSystem.md](ContextSystem.md) | @lit/context 响应式状态分发：Controller 提供，UI 组件消费 |
| EntityRepository | [EntityRepository.md](EntityRepository.md) | 实体 CRUD 层：事务性 upsert、字段级 diff、UIUpdateBus 集成 |
| MessageRepository | [MessageRepository.md](MessageRepository.md) | Per-session 消息仓库：发布-订阅、双向游标分页、并发去重、不可变更新 |
| DialogState | [DialogState.md](DialogState.md) | Promise-based dialog overlay：Tool Confirm / Ask User / Restore Confirm |
| PersistenceController | [PersistenceController.md](PersistenceController.md) | PersistenceLayer 生命周期管理：SharedWorker 桥接、连接重试、Worker 适配器 |
| NotificationController | [NotificationController.md](NotificationController.md) | 通知系统：UIUpdateBus 订阅、焦点检测、声音/Toast/动画多通道通知 |
| MessageVirtualScroll | [MessageVirtualScroll.md](MessageVirtualScroll.md) | Telegram 风格虚拟滚动：骨架屏占位、可见性状态机、热/冷分离、滚动保持 |
| SessionManagement | [SessionManagement.md](SessionManagement.md) | 会话树层级构建 + Tab 页签管理：标题同步、瞬态参数、键盘导航 |
| EditorSystem | [EditorSystem.md](EditorSystem.md) | 文件编辑器双 Controller 架构：视图模式、脏跟踪、光标节流持久化 |
| VisibilityState | [VisibilityState.md](VisibilityState.md) | 可见性状态机：VISIBLE/HIDDEN/TRANSITIONING 三态控制虚拟滚动操作许可 |
| ScrollSaver | [ScrollSaver.md](ScrollSaver.md) | Telegram 风格滚动位置保持：锚点元素 + 三级 fallback 链 |
| WindowSystem | [WindowSystem.md](WindowSystem.md) | 浮动窗口状态机：normal/minimized/maximized、viewport 自适应、bubble 定位 |
| SettingsSystem | [SettingsSystem.md](SettingsSystem.md) | 全局设置系统：分组状态管理、DOM 副作用、多 Tab 同步 |
| ConcurrencyPatterns | [ConcurrencyPatterns.md](ConcurrencyPatterns.md) | 跨模块竞态防护模式：Generation Counter、Checkpoint、AbortController、事务保护、引用计数 Suspend、Loop-Until-Stable |
| SyncPattern | [SyncPattern.md](SyncPattern.md) | PersistenceLayer 乐观同步模式：本地优先写入、优雅关闭、指数退避重试、大结果截断 |
