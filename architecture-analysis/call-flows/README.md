# 核心调用链

本目录按运行流程追踪当前正式代码。它与 `package-docs` 的区别是：package 文档回答“组件属于谁、负责什么”，这里回答“一项操作从入口到结果依次经过什么，以及跨过了哪些 package 边界”。

调用链不严格按 package 拆分。每篇文档标出主负责 package，并在流程中保留跨包调用，否则一次请求会被人为切成互不相连的几段。

## 调用链索引

| 调用链 | 主负责 package | 跨包边界 | 主要问题 |
|---|---|---|---|
| [Runtime 启动与 Session 装配](./runtime-bootstrap.md) | `pi-coding-agent` | `pi-agent-core`、`pi-ai` | 为什么先创建 services，再创建 session |
| [模型 Provider 请求](./model-provider-request.md) | `pi-coding-agent`、`pi-ai` | 厂商 API implementation | 模型选择、认证、header 和流事件如何进入请求 |
| [Settings、资源与 Extension Reload](./resource-reload.md) | `pi-coding-agent` | package/extension loader | reload 更新什么，什么必须保持同一实例 |
| [Prompt、Agent Loop 与工具执行](./prompt-and-agent-loop.md) | `pi-coding-agent`、`pi-agent-core` | `pi-ai`、工具实现 | 一次 prompt 如何循环调用模型和工具 |
| [Session 替换](./session-replacement.md) | `pi-coding-agent` | mode 宿主绑定 | new、resume、fork、import 为什么替换整个 runtime |
| [TUI 输入、事件与渲染](./tui-event-rendering.md) | `pi-coding-agent`、`pi-tui` | `AgentSession` 事件边界 | 输入怎样进入 session，流事件怎样变成终端更新 |

## 统一阅读约定

每篇调用链固定区分五件事：

1. 入口与终点：从哪个公开操作开始，到哪个可观察结果结束。
2. package 边界：控制权在哪一步交给另一个 package。
3. 所有权与生命周期：对象由谁持有，何时复用或重建。
4. 分支与失败：取消、reload、abort、重试或部分失败发生在哪里。
5. 不变量：修改代码时必须继续成立的约束。

图中的调用只表示默认正式 CLI/SDK 的 `AgentSession` 路径。已接入实验性 client/server/session-worker 的 `AgentHarness` 路径不混入这组主链，单独见[跨进程协议、测试与工程治理](../08-remote-protocol-and-engineering.md)。

## 与其他文档的关系

- [启动与运行时装配](../02-runtime-bootstrap.md)给出依赖创建总图，[AgentSession 与 Agent 执行](../05-agent-session-and-execution.md)给出请求与事件总图；本目录展开其中的关键链路。
- [Package 架构专题](../package-docs/README.md)解释包内组件和设计边界，本目录不重复完整组件说明。
- 各源码文件中的注释说明局部契约和不变量；跨文件、跨包的顺序以本目录为导航。
