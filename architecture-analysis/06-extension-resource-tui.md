# 扩展系统、资源加载与 TUI

extension（扩展）是在 Pi 进程内执行的受信任 TypeScript/JavaScript 模块。它可以注册工具、命令、事件处理函数、模型 provider 和界面能力。hook 是 Pi 在输入、模型请求、工具调用或会话切换等固定时点调用的扩展函数。

## 1. 资源系统

`DefaultResourceLoader` 汇总五类资源：extensions、skills、prompt templates、themes 和 context files。resource 在这里指可被 Pi 发现和加载的扩展代码、Markdown 指令或主题文件。来源包括全局目录、项目 `.pi`/`.agents`、CLI 显式路径、npm/git/local packages 和内建扩展。

资源加载不是简单 glob：它还处理设置覆盖、package manifest、过滤规则、冲突诊断、来源元数据和 reload。显式 `ResourceLoader` 接口允许 SDK 完全替换默认发现逻辑。

## 2. 扩展 API

扩展是受信任的 TypeScript/JavaScript 模块，可注册：

- LLM 工具、斜杠命令和快捷键。
- Agent/session/tool/input 生命周期 hook。
- 自定义消息渲染、footer（底部状态栏）、status、widget（嵌入编辑器附近的小组件）和 overlay（覆盖在当前界面之上的浮层）。
- 模型 provider、auth 和 CLI flags。

扩展 runtime 先收集注册信息，再由 runner 在明确生命周期中调用。wrapper 将产品对象裁剪为 extension context，避免扩展依赖全部内部实现。session 替换时旧 context 会失效，因此 UI 必须同步解绑。

## 3. Skills 与 Prompt Templates

Skill 是带元数据的 Markdown 能力包，按需注入模型上下文；prompt template 是用户触发的文本展开。两者故意不是可执行扩展：技能影响模型行为，扩展直接运行宿主代码。区分两者能让安装、发现和信任提示更准确。

## 4. 项目信任与安全边界

项目目录可包含设置、扩展和 package，自动加载会执行不可信代码。Interactive 模式在首次遇到相关项目资源时请求信任；非交互模式按 global `defaultProjectTrust` 或 CLI override 决定。未信任前只加载 context files、全局扩展和 CLI 扩展。

但项目信任不是沙箱。Pi 默认拥有启动用户的文件、进程、网络和凭据权限。真正隔离需要 Docker、OpenShell 或 Gondolin 等外部沙箱。架构上这是明确选择：核心通过 `ExecutionEnv` 支持替换执行环境，但产品不伪装成进程内权限系统。

### 4.1 具体实现

1. 读 [`resource-loader.ts`](../packages/coding-agent/src/core/resource-loader.ts) 的 `ResourceLoader` 接口与 `DefaultResourceLoader.reload()`，确认最终资源集合怎样形成。
2. 读 [`package-manager.ts`](../packages/coding-agent/src/core/package-manager.ts)，向前追踪 npm、git、本地路径与 scope 的解析。
3. 读 [`extensions/loader.ts`](../packages/coding-agent/src/core/extensions/loader.ts)，确认扩展源码怎样执行并生成 runtime 注册信息。
4. 读 [`project-trust.ts`](../packages/coding-agent/src/core/project-trust.ts) 与 [`cli/project-trust.ts`](../packages/coding-agent/src/cli/project-trust.ts)，确认受保护资源在加载前怎样被 gate。

## 5. TUI 分层

`pi-tui` 是业务无关组件库：

- `Component.render(width): string[]` 是最小渲染协议。
- `Container` 组合组件。
- `TuiBase` 管理终端、输入、焦点、overlay 和 16ms 渲染节流。
- regular screen 做增量行更新；fullscreen/viewport 维护可滚动布局。
- editor、markdown、select list、image 等是复用组件。

`coding-agent/modes/interactive` 才把 AgentSessionEvent 转成消息组件、工具组件、footer 和命令 UI。这个边界使 TUI 可独立用于其他终端应用。

## 6. 差量渲染

终端不是 DOM。每次清屏重画会闪烁、破坏滚动历史并放大慢速连接成本。TUI 保存上一帧规范化行，找到变化区域，只写必要 cursor movement 和内容。内容缩短、图片行、ANSI style、超链接和硬件 cursor 都需专门处理。

`CURSOR_MARKER` 是组件在字符串中发出的零宽标记；renderer 移除它并定位真实终端光标，以支持 IME 候选窗口。overlay 则把带 ANSI 的行按可见列宽切片后合成，不能用普通字符串索引。

## 7. Overlay 与焦点

overlay 有视觉层级、捕获/非捕获模式、隐藏状态和 pre-focus 链。焦点恢复状态机处理“overlay 暂时让焦点给嵌入控件，控件关闭后是否恢复 overlay”等情况。实现复杂，但这是 settings、模型选择器、自定义扩展 UI 可以嵌套工作的基础。

### 7.1 具体实现

1. 读 [`tui.ts`](../packages/tui/src/tui.ts) 的 `requestRender()` 与 render pass，确认失效请求怎样合并成一次终端更新。
2. 继续读同一文件的 focus 与 overlay 方法，确认视觉层级和输入归属怎样联动。
3. 最后看 `interactive-mode.ts` 的 `handleEvent()`，确认 Agent 事件怎样变成组件状态。

## 8. Extension 体现的产品设计取舍

Pi 没有把所有产品策略写死在 `Agent` 或 `InteractiveMode` 中。核心保持一个较小的模型—工具循环，coding-agent 在明确事件位置允许扩展观察或替换行为。这一选择体现在四点：

- 扩展可以注册新能力，例如工具、命令和 provider，而不修改 agent loop。
- 扩展可以在固定 hook 改变输入、system prompt、provider payload、工具调用结果或 compaction；改变发生在明确边界，不是任意 monkey patch。
- 传给扩展的 context 只暴露受支持的产品操作，不直接交出全部内部对象。session 替换后旧 context 会失效，防止旧 UI 和订阅继续操作新会话。
- 资源发现、代码执行和 session 绑定分开。这样 reload 可以重新计算资源来源与冲突，再建立新的运行时绑定。

具体例子是 compaction。agent loop 只知道下一轮前可以 `prepareNextTurn()`；coding-agent 决定何时压缩；`session_before_compact` 又允许扩展取消默认压缩或提供自己的摘要。核心循环、产品策略和用户定制因此可以分别演进。

代价也很明确：事件顺序、错误策略、reload 和 session 切换必须稳定，否则扩展会观察到不一致状态；扩展在宿主进程执行，能力边界不是安全沙箱；大量 UI 能力仍由 `InteractiveMode` 连接，使这个文件承担较高的组合复杂度。

### 8.1 具体实现

1. 读 [`types.ts`](../packages/coding-agent/src/core/extensions/types.ts)，先确认扩展公开 API、事件和 context 的边界。
2. 接着读 [`runner.ts`](../packages/coding-agent/src/core/extensions/runner.ts)，确认可拦截 hook 的合并顺序和错误策略；模块加载过程已经在 4.1 的 `loader.ts` 说明。
3. 最后看 `agent-session.ts` 的 `bindExtensions()`、`_installAgentToolHooks()` 和 compaction 事件调用点，确认扩展怎样进入正式产品生命周期。

## 9. Footer 是什么

footer 是交互式终端底部的状态栏，默认显示目录、Git 分支、token/费用、context 占用、模型和扩展状态。自定义 footer 的渲染函数不能直接读取 `InteractiveMode` 的私有状态，所以 Pi 传入只读的 `FooterDataProvider`，提供 Git 分支、扩展 status 和可用 provider 数量。Git 监听和资源清理由产品层统一管理；footer 只读取并渲染。它不是主要架构主线；需要了解具体实现时，先读 [`footer.ts`](../packages/coding-agent/src/modes/interactive/components/footer.ts) 的渲染输入，再读 [`footer-data-provider.ts`](../packages/coding-agent/src/core/footer-data-provider.ts) 的只读数据边界。
