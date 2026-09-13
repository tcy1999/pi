# TUI 与交互

会话已能执行请求并发布事件后，interactive mode 将这些事件转换为终端组件。本章说明呈现层怎样处理状态、渲染和输入焦点；资源加载与扩展注册见[资源与扩展](./04-resources-and-extensions.md)。

## 1. TUI 分层

`pi-tui` 是业务无关组件库：

- `Component.render(width): string[]` 是最小渲染协议。
- `Container` 组合组件。
- `TuiBase` 管理终端、输入、焦点和 overlay（浮层），并将渲染间隔限制为最短约 16 ms。
- `TuiMainScreen` 做主屏增量行更新；`TuiAltScreen` 管理 fullscreen viewport、选择和搜索。
- editor、markdown、select list、image 等是复用组件。

`coding-agent/modes/interactive` 才把 AgentSessionEvent 转成消息组件、工具组件、footer 和命令 UI。`tui-renderer.ts` 集中创建常规与全屏渲染器，`chat-viewport.ts` 组合可滚动的对话记录和固定输入区域；实验性 client TUI 也复用这两个布局与创建函数。这个边界使 `pi-tui` 可独立用于其他终端应用。

工作、压缩、分支摘要和自动重试共用 `StatusIndicator` 渲染契约。支持嵌入状态的 editor 会把当前 indicator 放进顶部边框；不支持时才使用独立 status container。切换 renderer 或 editor 时会重新安置同一个活动 indicator，避免状态仅因 UI 重建而消失。

树导航与 compaction 共用产品级互斥边界。用户在选择摘要策略的对话框停留期间，另一个 compaction 或导航可能启动，因此 UI 在关闭对话框后、替换 status/escape handler 前会再次检查 `session.isCompacting`；`AgentSession.navigateTree()` 也做同样的最终保护。

## 2. 差量渲染

每次清屏重画会闪烁、破坏终端滚动历史，并增加慢速连接的传输量。TUI 保存上一帧规范化后的行，找到变化区域，只输出必要的光标移动指令和内容。内容缩短、图片、ANSI 样式、超链接和终端光标都需单独处理。

`CURSOR_MARKER` 是组件在字符串中发出的零宽标记；renderer 移除它并定位真实终端光标，以支持 IME 候选窗口。overlay 则把带 ANSI 的行按可见列宽切片后合成，不能用普通字符串索引。

## 3. Overlay 与焦点

overlay 有视觉层级、捕获/非捕获模式、隐藏状态和 pre-focus 链。焦点恢复状态机处理“overlay 暂时让焦点给嵌入控件，控件关闭后是否恢复 overlay”等情况。例如模型选择器打开嵌套输入框后，关闭输入框应把输入焦点交回仍可见的选择器；选择器也关闭时，再恢复此前的编辑器焦点。

### 3.1 具体实现

1. 读 [`tui.ts`](../packages/tui/src/tui.ts) 的 `requestRender()` 与 render pass，确认失效请求怎样合并成一次终端更新。
2. 继续读同一文件的 focus 与 overlay 方法，确认视觉层级和输入归属怎样联动。
3. 读 [`tui-renderer.ts`](../packages/coding-agent/src/modes/interactive/tui-renderer.ts) 与 [`chat-viewport.ts`](../packages/coding-agent/src/modes/interactive/chat-viewport.ts)，确认产品如何配置 renderer 与 fullscreen 布局。
4. 最后看 [interactive-mode.ts](../packages/coding-agent/src/modes/interactive/interactive-mode.ts) 的 `handleEvent()`，确认 Agent 事件怎样变成组件状态。

## 4. Footer 是什么

footer 是交互式终端底部的状态栏，默认显示目录、Git 分支、token/费用、context 占用、模型和扩展状态。自定义 footer 的渲染函数不能直接读取 `InteractiveMode` 的私有状态，所以 Pi 传入只读的 `FooterDataProvider`，提供 Git 分支、扩展 status 和可用 provider 数量。Git 监听和资源清理由产品层统一管理；footer 只读取并渲染。需要了解具体实现时，先读 [`footer.ts`](../packages/coding-agent/src/modes/interactive/components/footer.ts) 的渲染输入，再读 [`footer-data-provider.ts`](../packages/coding-agent/src/core/footer-data-provider.ts) 的只读数据边界。
