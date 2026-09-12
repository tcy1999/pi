# 启动与运行时装配

## 1. 启动阶段

`packages/coding-agent/src/cli.ts` 只负责进程初始化：设置进程标识、预配置 HTTP dispatcher，然后调用 `main(argv)`。`main.ts` 负责把命令行参数转换成一个可运行的 Agent 会话。启动主线可以分成三步：

1. **确定会话和工作目录。** 解析参数，运行迁移并加载启动设置，选择已有会话或创建新会话，再确定最终工作目录（cwd）。
2. **为这个目录创建运行环境。** 检查项目信任，加载设置、模型目录、扩展和其他资源，确定模型、思考级别和工具配置，创建 `AgentSession`。
3. **进入所选运行模式（mode）。** 准备标准输入、文件输入、主题和启动诊断，再将运行环境交给对应模式。模式决定如何接收输入、输出结果；会话执行仍由 `AgentSession` 负责。

模式在解析参数时确定，在会话及其依赖准备好后启动。各模式的输入输出区别见[第 2 节](#2-mode-负责输入输出agentsession-负责会话处理)。

第二步“为目录创建运行环境”也用于后续会话切换。例如，从目录 A 切换到目录 B 的会话时，设置、扩展、工具路径和模型配置都可能改变，不能直接沿用 A 的依赖。因此，`createAgentSessionServices()` 先按目标 cwd 加载这些服务，再解析会话选项并创建 `AgentSession`。`main.ts` 将这套过程封装为可重复调用的工厂，`AgentSessionRuntime` 则管理当前会话及其切换，让新建、恢复、派生会话都能复用同一套创建流程。

上述三步描述的是进入 Agent 会话的主线。一次性命令在这条主线的不同位置结束：

| 处理位置 | 命令 | 为什么在这里处理 |
|---|---|---|
| 创建运行环境之前 | auth、package、config、version、export | 完成各自的操作即可退出，无需进入会话运行模式 |
| 创建运行环境之后、进入 mode 之前 | help、list-models | help 需要扩展注册的命令行选项；list-models 需要模型配置和扩展提供的模型供应商。这些资源加载后才能给出完整结果 |

offline 等进程级选项在启动早期生效，影响后续初始化，不属于单独的运行模式。


本章先给启动装配总图；模型与认证、资源与扩展、请求执行分别在后续章节展开。需要逐函数追踪初始化、替换、请求、资源重载、Provider 请求和 TUI 渲染时，转到[核心调用链](./call-flows/README.md)。

### 1.1 具体实现

按启动时对象出现的顺序阅读：

1. [`cli.ts`](../packages/coding-agent/src/cli.ts)：确认极薄进程入口怎样调用 `main()`。
2. [`main.ts`](../packages/coding-agent/src/main.ts)：跟踪 `appMode`、`sessionManager`、`settingsManager`、`modelRuntime`、`resourceLoader` 和 `runtime` 的创建与传递，重点看 `resourceLoader` 如何使用同一个 `settingsManager`，以及加载出的扩展如何影响后续模型和工具配置。
3. [`agent-session-services.ts`](../packages/coding-agent/src/core/agent-session-services.ts)：确认这些依赖为何要先于 session 创建。
4. [`agent-session-runtime.ts`](../packages/coding-agent/src/core/agent-session-runtime.ts)：它负责替换整组服务，具体切换顺序见[会话替换生命周期](./06-session-and-persistence.md#9-会话替换生命周期)。
5. [`sdk.ts`](../packages/coding-agent/src/core/sdk.ts)：对照 CLI 与 SDK 怎样共享会话构造逻辑，以及依赖准备流程的差异。

### 1.2 CLI 与 SDK：整体思路一致，依赖准备不同

两者都先确定工作目录、准备配置和模型/资源依赖，再解析会话选项并构造 `AgentSession`。CLI 先调用 `createAgentSessionServices()`，解析命令行对应的模型和工具选项，再通过 `createAgentSessionFromServices()` 将完整依赖传给 `createAgentSession()`。SDK 直接调用 `createAgentSession()` 时，则在函数内部补齐调用方未提供的依赖，不会自动经过 services 工厂。

因此，共享的是最终会话构造逻辑，不能据此认为初始化路径和默认行为完全相同。CLI 还负责会话选择、项目信任决策和命令行参数处理；SDK 允许调用方注入依赖，也可以显式使用 services/runtime API。具体差异见[启动调用链](./call-flows/runtime-bootstrap.md#21-sdk-默认入口与-cli-的差异)。

## 2. Mode 负责输入输出，AgentSession 负责会话处理

mode 是用户或外部程序操作会话的入口。例如，同样是“解释这个文件”，可以在终端编辑器里输入，也可以作为一次命令的参数传入，还可以由另一个程序通过 RPC 提交。这些入口最终都把普通提示交给 `AgentSession.prompt()`，由它完成输入预处理、调用 Agent、处理扩展和保存历史。

区别在于输入从哪里来，以及会话事件怎样交给调用方：

| 模式 | 输入 | 输出与后续行为 | 代码入口 |
|---|---|---|---|
| interactive | 用户在终端编辑器中输入 | 将事件显示为消息和工具执行状态，继续等待用户输入 | [interactive-mode.ts](../packages/coding-agent/src/modes/interactive/interactive-mode.ts) |
| print（文本） | 本次启动准备好的提示、标准输入或文件内容 | 输出文本结果，然后退出 | [print-mode.ts](../packages/coding-agent/src/modes/print-mode.ts) |
| JSON | 本次启动准备好的输入，与 print 共用执行入口 | 将执行事件逐行输出为 JSON，然后退出 | [print-mode.ts](../packages/coding-agent/src/modes/print-mode.ts)、[json-event.ts](../packages/coding-agent/src/modes/json-event.ts) |
| RPC | 外部程序通过标准输入发送 JSON 命令 | 通过标准输出返回响应和事件，继续接收控制命令 | [rpc-mode.ts](../packages/coding-agent/src/modes/rpc/rpc-mode.ts)、[rpc-types.ts](../packages/coding-agent/src/modes/rpc/rpc-types.ts) |

“共用运行时”指这些模式使用同一套会话实现，不是四个模式同时连接同一个会话。一次启动选择其中一种模式；模式负责输入输出，`AgentSession` 负责会话处理，它内部的 `Agent` 负责模型与工具的执行循环。这样，新增一种输入输出方式时，可以复用已有会话逻辑。

这里的 RPC 是对正式 `AgentSession` 的进程接口。实验性 client/server 则在会话 worker 中运行另一套 `AgentHarness`，通过 Chord 提供服务，不属于上表的运行模式。它的结构单独见[跨进程组件说明](./01-architecture-overview.md#4-跨进程组件是什么)。

## 3. 依赖怎样先于 Agent 装配

以 CLI 的 services 工厂为例，创建基础模型运行时和补齐扩展能力是两个阶段：

```text
确定目标 cwd、SessionManager 与项目信任
  → 创建或复用基础 ModelRuntime
  → 使用传入的 SettingsManager，缺省时按 cwd 创建
  → 创建 ResourceLoader 并 reload
  → 收集并注册扩展 provider / native provider
  → ModelRuntime.refresh({ allowNetwork: false })
  → 解析本次会话的模型、thinking level 与工具选项
  → createAgentSessionFromServices() → Agent + AgentSession
  → AgentSessionRuntime 持有当前会话，交给 mode
```

CLI 通常已在进入 services 工厂前准备目标 cwd 的设置实例，再让 resource loader 和 session 共用它。图中的“缺省时创建”是工厂的补齐行为，不表示所有入口都在 ModelRuntime 之后才读取设置。

基础 `ModelRuntime.create()` 准备凭据、模型配置、目录存储和内建 provider；它不发现项目扩展。资源加载完成后，services 才把扩展注册的 provider 加入同一 runtime，并刷新模型视图。因此这里既有“先有模型基础设施”，也有“再由资源补齐模型能力”；本次会话选模型要等两者完成。

这也是后续章节的阅读顺序：[模型与认证](./03-model-and-auth-runtime.md) → [资源与扩展](./04-resources-and-extensions.md) → [AgentSession 与执行循环](./05-agent-session-and-execution.md)。前两章解释依赖如何准备，第 5 章再解释准备好的对象如何执行一次请求。流式协议会在模型章就地展开，避免把同一适配层拆散。

具体实现从 [agent-session-services.ts](../packages/coding-agent/src/core/agent-session-services.ts) 的 `createAgentSessionServices()` 读起，再看 `createAgentSessionFromServices()` 怎样将依赖传给 [sdk.ts](../packages/coding-agent/src/core/sdk.ts)。逐函数时序见[启动调用链](./call-flows/runtime-bootstrap.md)。
