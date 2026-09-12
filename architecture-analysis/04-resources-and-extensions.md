# 资源加载与扩展

extension（扩展）是在 Pi 进程内执行的受信任 TypeScript/JavaScript 模块。它可以注册工具、命令、事件处理函数、模型 provider 和界面能力。hook 是 Pi 在输入、模型请求、工具调用或会话切换等固定时点调用的扩展函数。

此时基础 ModelRuntime 已存在。本章解释怎样按 cwd 和信任状态加载设置与资源，并补齐扩展 provider、工具和提示定义；后续 AgentSession 才把这些定义绑定到可执行会话。扩展 UI 的渲染机制见 [TUI 与交互](./07-tui-and-interaction.md)。

## 1. 资源系统

`DefaultResourceLoader` 汇总 extensions、skills、prompt templates、themes 和 context files 五类集合，另外加载 system prompt 与 append-system-prompt 来源。resource 在这里指可被 Pi 发现和加载的扩展代码、Markdown 指令或主题文件。来源包括全局目录、项目 `.pi`/`.agents`、CLI 显式路径、npm/git/local packages 和内建扩展。

资源加载不是简单 glob：它还处理设置覆盖、package manifest、过滤规则、冲突诊断、来源元数据和 reload。显式 `ResourceLoader` 接口允许 SDK 完全替换默认发现逻辑。

## 2. 项目信任与安全边界

项目目录可包含设置、扩展和 package，自动加载会执行不可信代码。Interactive 模式在首次遇到相关项目资源时请求信任；非交互模式按 global `defaultProjectTrust` 或 CLI override 决定。未信任前只加载 context files、全局扩展和 CLI 扩展。

但项目信任不是沙箱。Pi 默认拥有启动用户的文件、进程、网络和凭据权限。真正隔离需要 Docker、OpenShell 或 Gondolin 等外部沙箱。durable Harness 可通过 `ExecutionEnv` 替换文件和进程执行环境；正式 `AgentSession` 路径的 coding tools 仍在宿主进程中运行。两条路径都不应被理解为进程内权限系统。

### 2.1 具体实现

1. 读 [`resource-loader.ts`](../packages/coding-agent/src/core/resource-loader.ts) 的 `ResourceLoader` 接口与 `DefaultResourceLoader.reload()`，确认最终资源集合怎样形成。
2. 读 [`package-manager.ts`](../packages/coding-agent/src/core/package-manager.ts)，向前追踪 npm、git、本地路径与 scope 的解析。
3. 读 [`extensions/loader.ts`](../packages/coding-agent/src/core/extensions/loader.ts)，确认扩展源码怎样执行并生成 runtime 注册信息。
4. 读 [`project-trust.ts`](../packages/coding-agent/src/core/project-trust.ts) 与 [`cli/project-trust.ts`](../packages/coding-agent/src/cli/project-trust.ts)，确认受保护资源在加载前怎样被 gate。

## 3. 扩展 API

扩展是受信任的 TypeScript/JavaScript 模块，可注册：

- LLM 工具、斜杠命令和快捷键。
- Agent/session/tool/input 生命周期 hook。
- 自定义消息渲染、footer（底部状态栏）、status、widget（嵌入编辑器附近的小组件）和 overlay（覆盖在当前界面之上的浮层）。
- 模型 provider、auth 和 CLI flags。

扩展 runtime 先收集注册信息，再由 runner 在明确生命周期中调用。wrapper 将产品对象裁剪为 extension context，避免扩展依赖全部内部实现。session 替换时旧 context 会失效，因此 UI 必须同步解绑。

## 4. Skills 与 Prompt Templates

Skill 是带元数据的 Markdown 能力包，按需注入模型上下文；prompt template 是用户触发的文本展开。两者故意不是可执行扩展：技能影响模型行为，扩展直接运行宿主代码。区分两者能让安装、发现和信任提示更准确。

## 5. Extension 体现的产品设计取舍

Pi 没有把所有产品策略写死在 `Agent` 或 `InteractiveMode` 中。核心保持一个较小的模型—工具循环，coding-agent 在明确事件位置允许扩展观察或替换行为。这一选择体现在四点：

- 扩展可以注册新能力，例如工具、命令和 provider，而不修改 agent loop。
- 扩展可以在固定 hook 改变输入、system prompt、provider payload、工具调用结果或 compaction；改变发生在明确边界，不是任意 monkey patch。
- 传给扩展的 context 只暴露受支持的产品操作，不直接交出全部内部对象。session 替换后旧 context 会失效，防止旧 UI 和订阅继续操作新会话。
- 资源发现、代码执行和 session 绑定分开。这样 reload 可以重新计算资源来源与冲突，再建立新的运行时绑定。

具体例子是 compaction。agent loop 只知道下一轮前可以 `prepareNextTurn()`；coding-agent 决定何时压缩；`session_before_compact` 又允许扩展取消默认压缩或提供自己的摘要。核心循环、产品策略和用户定制因此可以分别演进。

代价也很明确：事件顺序、错误策略、reload 和 session 切换必须稳定，否则扩展会观察到不一致状态；扩展在宿主进程执行，能力边界不是安全沙箱；大量 UI 能力仍由 `InteractiveMode` 连接，使这个文件承担较高的组合复杂度。

### 5.1 具体实现

1. 读 [`types.ts`](../packages/coding-agent/src/core/extensions/types.ts)，先确认扩展公开 API、事件和 context 的边界。
2. 接着读 [`runner.ts`](../packages/coding-agent/src/core/extensions/runner.ts)，确认可拦截 hook 的合并顺序和错误策略；模块加载过程已经在 2.1 的 [loader.ts](../packages/coding-agent/src/core/extensions/loader.ts) 说明。
3. 最后看 [agent-session.ts](../packages/coding-agent/src/core/agent-session.ts) 的 `bindExtensions()`、`_installAgentToolHooks()` 和 compaction 事件调用点，确认扩展怎样进入正式产品生命周期。
