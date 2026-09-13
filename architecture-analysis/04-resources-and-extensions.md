# 资源加载与扩展

extension（扩展）是在 Pi 进程内执行的受信任 TypeScript/JavaScript 模块。它可以注册工具、命令、事件处理函数、模型 provider 和界面能力。hook 是 Pi 在输入、模型请求、工具调用或会话切换等固定时点调用的扩展函数。

此时基础 ModelRuntime 已存在。本章解释怎样按 cwd 和信任状态加载设置与资源，并补齐扩展 provider、工具和提示定义；后续 AgentSession 才把这些定义绑定到可执行会话。扩展 UI 的渲染机制见 [TUI 与交互](./07-tui-and-interaction.md)。

## 1. 资源系统

`DefaultResourceLoader` 汇总 extensions、skills、prompt templates、themes 和 context files 五类集合，另外加载 system prompt 与 append-system-prompt 来源。resource 在这里指可被 Pi 发现和加载的扩展代码、Markdown 指令或主题文件。来源包括全局目录、项目 `.pi`/`.agents`、CLI 显式路径、npm/git/local packages 和内建扩展。

资源加载不是简单 glob：它还处理设置覆盖、package manifest、过滤规则、冲突诊断、来源元数据和 reload。显式 `ResourceLoader` 接口允许 SDK 完全替换默认发现逻辑。

### 1.1 Extension 与 Package 的区别

extension 解决“怎样给运行中的 Pi 增加或改变行为”；Pi package 解决“怎样把一组资源一起安装、更新和分享”。这里的 package 指资源分发包，而不是仓库中 `packages/ai`、`packages/tui` 等源码包。

| 维度 | Extension | Pi package |
|---|---|---|
| 基本单位 | 在宿主进程执行的 TypeScript/JavaScript 模块 | 来自 npm、Git 或本地路径的一组资源 |
| 包含内容 | 工具、命令、事件处理函数、provider、UI 等注册逻辑 | extensions、skills、prompt templates、themes |
| 主要生命周期 | 加载模块、收集注册、绑定 session、响应事件、释放会话资源 | 解析来源、安装或更新、发现资源、按配置筛选 |
| 运行时接入 | 由 extension loader 加载，再由 session 和 runner 绑定、调用 | 提供资源路径，具体内容交给各类 loader |

例如，一个代码审查 package 可以同时带有注册 `/review` 命令的 extension、描述审查流程的 skill 和展开审查输入的 prompt template。安装后，三类资源分别进入自己的加载机制；package 本身没有额外的工具调用或 session 事件接口。单个 extension 也可以直接放在 `.pi/extensions/` 中使用，只有 skill 或 theme 的 package 同样成立。

### 1.2 Package 的设计：先解析来源，再加载内容

同一资源可以来自全局安装、项目安装或 CLI 显式路径。如果每类资源都自行处理 npm 下载、Git 来源、配置覆盖和冲突，加载规则就容易分叉。因此 Pi 将来源解析与内容加载分开：

```text
有效 settings 中的 package 来源 + 全局/项目约定目录
  → PackageManager：解析来源、安装位置、manifest/约定目录与过滤规则
  → 带来源、作用域和启用状态的资源路径
  → DefaultResourceLoader：合并 CLI/SDK 显式输入，加载资源并报告冲突
      → extension loader：执行 factory，收集工具、命令和事件注册
          → AgentSession 绑定 ExtensionRunner，按生命周期调用
      → skill / prompt / theme loader：读取对应资源内容
```

package manifest 是包中声明资源位置的清单；没有清单时使用约定目录。过滤规则在包允许的资源集合内选择启用项。作用域表示来源属于用户、项目或临时输入；同一包的身份去重与同名资源的冲突处理分开，来源信息保留到诊断阶段，才能解释“这项资源来自哪里、为什么另一项没有生效”。项目来源进入加载链前还受第 2 节的信任规则约束。

这个划分让分发方式与运行时能力可以分别演进：新增 package 来源不必重写 extension 事件系统，SDK 也可以直接提供资源路径、factory 或自定义 loader。代价是必须维护来源优先级、过滤和冲突规则的一致性；reload 时要重新解析并发布资源集合，再重绑扩展，不能只重新执行某个扩展文件。

设置合并、包身份与优先级的细节见[设置、Package 与资源解析链](./package-docs/08-settings-package-resource-resolution.md)；扩展注册与会话生命周期见[扩展系统、资源加载与信任边界](./package-docs/03-extension-resource-lifecycle.md)。

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

### 4.1 Skill 的渐进式加载

如果启动时把所有 skill 正文和参考资料都放进 system prompt，即使用户只想修一个小错误，也会占用大量无关上下文。Pi 采用渐进式披露（progressive disclosure）：先提供可用技能的摘要，任务需要时再读取具体指令和参考资料。以下描述正式 coding-agent 的 `AgentSession` 路径。

| 阶段 | 实际行为 | 进入模型上下文的内容 |
|---|---|---|
| 资源发现 | 启动或 reload 时读取 skill 文件，解析 frontmatter（文件头部元数据），校验并处理冲突；`Skill` 对象保存名称、描述、路径和来源等元数据 | 发现本身不注入正文 |
| 能力提示 | `buildSystemPrompt()` 调用 `formatSkillsForPrompt()`，生成 `<available_skills>` | 名称、描述、文件位置，以及按需读取的指引 |
| 正文加载 | 模型根据任务选择 skill 并调用文件读取工具；或用户显式输入 `/skill:name`，由宿主展开 | 工具结果中的文件内容，或展开后的用户消息正文 |
| 参考资料 | 模型按 skill 指令继续读取引用文档、使用脚本或素材 | 实际读取或执行后返回的内容 |

这里延迟的是正文进入模型上下文的时机。发现阶段的 `loadSkillFromFile()` 已经读取整个 Markdown 文件来解析元数据，但返回的 `Skill` 对象不保存正文。引用目录也不会因为 skill 被发现或选中就自动全部注入。

例如，安装了 `code-review` 和 `pdf-tools` 后，模型先看到两者的名称、描述和路径。用户要求审查代码时，模型可读取 `code-review/SKILL.md`，再按其中指令读取 `references/checklist.md`；这条路径不需要加载 `pdf-tools` 的正文。引用的相对路径以 skill 所在目录为基准。

### 4.2 正文进入上下文的两条路径

**模型自主读取**：system prompt 提示模型在任务匹配描述时读取对应文件。优先使用 `read`，没有 `read` 时使用 `bash`；两者都未启用时，不加入 skill 清单。匹配由模型判断，没有宿主自动匹配并注入正文的步骤，因此模型也可能漏读。读取结果作为普通工具结果进入后续模型请求。

**用户显式触发**：输入 `/skill:code-review 检查错误处理` 后，`AgentSession._expandSkillCommand()` 查找已发现的 skill，重新读取文件，去掉 frontmatter，将正文包在带名称、位置和引用目录说明的 `<skill>` 块中，再追加命令参数。这发生在输入预处理阶段，正文进入用户消息，无需模型先发起读文件调用；顺序见[输入预处理与队列](./05-agent-session-and-execution.md#14-输入预处理与队列)。

`disable-model-invocation: true` 会把该 skill 从 system prompt 清单中隐藏，但仍允许 `/skill:name` 显式展开。这个字段控制自动发现提示，不构成文件访问权限限制。

### 4.3 具体实现

1. 读 [`skills.ts`](../packages/coding-agent/src/core/skills.ts) 的 `Skill`、`loadSkillFromFile()` 和 `formatSkillsForPrompt()`，对照发现时保存的元数据与模型看到的摘要。
2. 读 [`system-prompt.ts`](../packages/coding-agent/src/core/system-prompt.ts) 的 `buildSystemPrompt()`，确认 `read` / `bash` 可用性怎样决定清单和读取指引。
3. 读 [`agent-session.ts`](../packages/coding-agent/src/core/agent-session.ts) 的 `_expandSkillCommand()`，确认显式命令怎样把正文放进输入。编写与安装规则见 [Skills 用户文档](../packages/coding-agent/docs/skills.md)。

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
