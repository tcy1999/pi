# TUI 组件、渲染与焦点模型

来源：`packages/coding-agent/docs/tui.md`，并核对 `packages/tui/src/`。排除自定义组件教程、内建组件 API 清单和示例。

## 1. 最小组件协议

TUI 不理解 Agent。组件只需要按给定宽度返回终端行，并在需要时处理输入与失效：

```text
render(width) → string[]
invalidate()
handleInput(data)   可选
```

coding-agent 的 interactive mode 把 AgentSessionEvent 转成这些组件；`pi-tui` 只负责布局、终端输出、焦点、滚动和 overlay。

## 2. 为什么需要差量渲染

终端没有 DOM。整屏清除重画会闪烁、破坏 scrollback，并放大远程终端的输出成本。TUI 保存上一帧行，比较新旧帧，只发必要的 cursor movement、行更新和清理序列。

更新不是普通字符串 diff：ANSI style、Unicode grapheme 宽度、宽字符、图片协议和内容缩短都会改变光标位置。overlay 合成也必须按可见列切片，不能按 JavaScript 字符索引。

渲染请求有最小约 16ms 间隔，用于合并短时间内的多次状态变化。组件修改状态后请求 render，而不是直接向 terminal 写业务内容。

## 3. Regular 与 fullscreen

regular 模式在主屏幕工作，尽量保留终端 scrollback，并对可见内容做差量更新。fullscreen 使用 alternate screen 和应用自己管理的 viewport，负责滚动、选择、搜索及固定布局。

它们共享组件和焦点基础，但输出策略不同，所以 fullscreen 不是给 regular 模式简单套一层高度限制。

当前具体类型是 `TuiMainScreen` 与 `TuiAltScreen`。coding-agent 不再把创建参数散落在各 presentation：`createInteractiveTui()` 是共享 composition root，统一终端、剪贴板、URL、搜索样式、jump-to-end、滚动条和右键粘贴；`createInteractiveTuiReference()` 为组件提供稳定代理，使底层 renderer 替换时不必重建所有引用。

fullscreen chat 的布局由 `createChatViewport()` 组合：可滚动 transcript 占剩余高度，pending/status/widgets/editor/footer 组成固定 dock。稳定 `InteractiveMode` 与实验性 client TUI 都复用这套 renderer/viewport，因此“共享外观”不等于“共享 AgentSession 状态”。

## 4. 焦点与硬件光标

TUI 只把输入发送给当前 focused component。可聚焦组件在渲染文本中放置零宽 `CURSOR_MARKER`，renderer 移除标记并把硬件光标移动到对应单元格。这让输入法候选窗口能够跟随真实编辑位置。

光标位置依赖最终布局，因此组件不能直接以字符串长度计算 terminal column。

## 5. Overlay 为什么复杂

Overlay 同时具有视觉顺序和焦点顺序。每个 overlay 记录显示状态、focus order、是否捕获输入，以及显示前的 focus。隐藏或移除顶层 overlay 后，焦点应恢复到下一个可见 overlay 或原组件。

嵌套交互还会出现 overlay 暂时把焦点交给内部组件，内部组件结束后需要恢复 overlay 的情况。因此实现维护显式 focus restore state，而不是简单保存一个 previous component。

## 6. Invalidation 与主题

如果组件缓存了已经着色的文本，主题改变后仅 requestRender 不够，因为缓存仍含旧 ANSI 颜色。`invalidate()` 用于清理这类派生缓存，然后下一次 render 从新主题重建。直接在每次 render 中调用 theme callback 的无状态组件则不需要额外缓存失效。

## 7. 具体实现

1. 读 [`tui.ts`](../../packages/tui/src/tui.ts) 的 `TuiBase.requestRender()`、`TuiMainScreen` 与 `TuiAltScreen` render pass，确认差量输出怎样调度。
2. 继续读同一文件的 focus、overlay、viewport、selection 与 search，确认输入和视觉层级的状态转换。
3. 读 [`tui-renderer.ts`](../../packages/coding-agent/src/modes/interactive/tui-renderer.ts) 与 [`chat-viewport.ts`](../../packages/coding-agent/src/modes/interactive/chat-viewport.ts)，确认 coding-agent 的共享 composition root 和 fullscreen dock。
4. 读 [`interactive-mode.ts`](../../packages/coding-agent/src/modes/interactive/interactive-mode.ts) 的 `handleEvent()`，确认正式路径怎样把 Agent 事件变成组件状态。
