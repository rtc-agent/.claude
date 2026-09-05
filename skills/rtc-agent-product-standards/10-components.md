# 前端组件架构

## 概述

基于 Lit 的 Web Components 组件库，对外只暴露一个 `<rtc-agent>` 组件，内部包含 16 个子组件，使用 9 个 Controller 管理状态。

## 公开组件 `<rtc-agent>`

```mermaid
flowchart TD
    A[rtc-agent] --> B[属性]
    A --> C[事件]
    A --> D[CSS 变量]

    B --> B1[theme: light/dark/system]
    B --> B2[app-label: 标题文字]
    B --> B3[bubble-icon: 气泡图标]
    B --> B4[scenarios-url: 场景文档]
    B --> B5[agentConfig: 声明式配置]

    C --> C1[rtc-agent-ready]

    D --> D1[--rtc-window-default-width]
    D --> D2[--rtc-window-default-height]
    D --> D3[--rtc-bubble-size]
```

| 属性 | 说明 |
|------|------|
| theme | 主题切换（light/dark/system） |
| app-label | 标题栏文字 + 气泡 tooltip |
| bubble-icon | 最小化气泡内的 SVG/HTML |
| scenarios-url | 场景文档 URL |
| agentConfig | 声明式函数注册（推荐） |

## 窗口系统

```mermaid
flowchart LR
    A[窗口模式] --> B[normal<br/>浮动窗口]
    A --> C[maximized<br/>全屏]
    A --> D[minimized<br/>气泡]

    B --> B1[420x640<br/>可拖拽/缩放]
    C --> C1[100% 填充]
    D --> D1[40x40 圆形<br/>点击恢复]
```

- 拖拽：title-bar 作为手柄
- 缩放：8 方向 resize
- 键盘：方向键移动，Shift 加速
- 视口约束：始终保持在可视区域内

## 状态管理

```mermaid
flowchart TD
    A[9 个 Controller] --> B[WindowState<br/>窗口位置/尺寸]
    A --> C[Auth<br/>登录状态]
    A --> D[Session<br/>会话列表]
    A --> E[Message<br/>消息列表]
    A --> F[Mode<br/>工作模式]
    A --> G[ToolCall<br/>工具确认]
    A --> H[Persistence<br/>数据层]
    A --> I[Skill<br/>函数注册]
    A --> J[WindowInteraction<br/>拖拽交互]
```

- Controller 之间不直接引用
- 根组件 `<rtc-agent>` 作为中枢编排跨 Controller 通信
- 使用 `@lit/context` 向子组件分发状态

## 样式系统

| 层级 | 内容 |
|------|------|
| Design Tokens | 间距、字体、圆角、阴影、过渡、z-index |
| 颜色主题 | light / dark（VS Code 风格） |
| CSS 变量 | 组件级自定义（窗口尺寸、气泡大小） |

## 关键交互

**输入区域**：
- textarea + 底部工具栏
- Enter 提交，Shift+Enter 换行
- 工具栏：附件、工具、模式、发送/停止

**消息列表**：
- 自动滚动到底部
- 用户滚动离开时显示"新消息"按钮
- 支持 Markdown 渲染、代码高亮

**工具确认弹窗**：
- 显示工具名和参数
- Yes / No 按钮
- 点击背景等同拒绝
