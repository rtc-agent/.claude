# ScrollSaver

Telegram 风格的滚动位置保持：在 DOM 变更（prepend/append/delete）前后保存和恢复滚动位置。

**所属 package**: `component`

## 关键代码文件

- [utils/scroll-saver.ts](~/Workspaces/rtc-agent/web-components/packages/component/src/utils/scroll-saver.ts) — ScrollSaver 实现
- [utils/message-virtual-scroll.ts](~/Workspaces/rtc-agent/web-components/packages/component/src/utils/message-virtual-scroll.ts) — 主要消费者

## 核心概念

```mermaid
sequenceDiagram
    participant VS as VirtualScroll
    participant SS as ScrollSaver
    participant DOM as Container DOM

    VS->>SS: save()
    Note over SS: 记录可见元素及其位置
    VS->>DOM: prepend/append/delete elements
    Note over DOM: DOM 结构变化<br/>scrollTop 可能偏移
    VS->>SS: restore()
    Note over SS: 找到锚点元素<br/>计算位置 delta<br/>调整 scrollTop
```

## 算法

### 1. save() — 记录状态

- 遍历容器中匹配 `query` 选择器的所有元素
- 记录在容器视口内可见的元素及其 `getBoundingClientRect()`
- 记录当前 `scrollTop` 和 `scrollHeight`

### 2. restore() — 恢复位置

三级 fallback 链：

```mermaid
flowchart TD
    A[restore] --> B{有已保存元素?}
    B -->|No| C["scrollTop = reverse ? scrollHeight : 0"]
    B -->|Yes| D{锚点元素仍连接?}
    D -->|Yes| E["计算 getBoundingClientRect delta<br/>调整 scrollTop"]
    D -->|No| F[重新查询可见元素]
    F --> G{找到新锚点?}
    G -->|Yes| E
    G -->|No| H["fallback: scrollTop += scrollHeight delta"]
```

### 锚点选择

- `reverse = true`（prepend 模式，从顶部加载）：锚点为第一个可见元素
- `reverse = false`（append 模式，从底部加载）：锚点为最后一个可见元素

### 位置参考边

根据元素是否溢出容器视口，选择 `top` 或 `bottom` 作为参考边：

```mermaid
flowchart LR
    A{reverse?} -->|true| B{溢出顶部?}
    A -->|false| C{溢出底部?}
    B -->|是且未溢出底部| D[参考边 = bottom]
    B -->|否| E[参考边 = top]
    C -->|是且未溢出顶部| F[参考边 = top]
    C -->|否| G[参考边 = bottom]
```

## 与 VirtualScroll 的集成

```mermaid
flowchart TB
    subgraph "MessageVirtualScroll"
        direction TB
        A[skeletonize] -->|before| B["scrollSaver.save()"]
        A -->|DOM change| C[更新骨架屏]
        A -->|after| D["scrollSaver.restore()"]
        
        E[restore] -->|before| F["scrollSaver.save()"]
        E -->|DOM change| G[恢复可见元素]
        E -->|after| H["scrollSaver.restore()"]
    end
```

## 关键注释摘录

> **来源** — [scroll-saver.ts:1-4](~/Workspaces/rtc-agent/web-components/packages/component/src/utils/scroll-saver.ts#L1-L4)
>
> ```text
> ScrollSaver - Preserves scroll position across DOM changes.
> Directly ported from Telegram Web's ScrollSaver.
> ```

> **Fallback 链** — [scroll-saver.ts:40-46](~/Workspaces/rtc-agent/web-components/packages/component/src/utils/scroll-saver.ts#L40-L46)
>
> ```text
> Fallback chain (matching Telegram Web's algorithm):
> 1. Try the saved anchor element
> 2. If anchor disconnected → re-query visible elements, pick new anchor
> 3. If still no anchor → fall back to scrollHeight delta
> ```

## 跨维度关联

- [[MessageVirtualScroll]] — ScrollSaver 的主要消费者
- [[VisibilityState]] — 与可见性状态机协作，确保只在 VISIBLE 状态下执行 restore
