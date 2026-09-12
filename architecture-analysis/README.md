# Pi 项目架构分析

本文档集基于 `main` 分支提交 `196aff41c`（2026-09-12）进行静态分析。分析对象包括源码、包清单、项目文档、测试入口、CI 和发布脚本；关键结论均回溯到当前源码。

## 核心名词

- `AgentSession`：正式 coding-agent 使用的一次产品会话，连接模型、工具、扩展、设置和磁盘历史。
- `AgentHarness`：`pi-agent-core` 中面向可恢复、可替换存储会话的 durable runtime；正式 CLI 仍使用 `AgentSession`，实验性 client/server 路径已通过 session worker 使用 Harness。
- durable session（持久会话）：进程退出后仍可从后端恢复的会话数据和操作记录。
- Chord：独立的应用组合运行时，提供 facet、service、replicated state 和远程 service 边界；实验性 coding-agent 跨进程架构用它组合插件和服务。
- provider（模型供应商适配）：描述一组模型、认证方式和请求实现的对象，例如 Anthropic 或 OpenAI provider。
- extension（扩展）：在 Pi 进程内执行的受信任代码模块，可以注册工具、命令、事件处理函数、模型 provider 和界面能力。
- hook（扩展事件处理函数）：Pi 在输入、模型请求、工具调用或会话切换等固定时点调用的扩展函数。

## 阅读顺序

每章的“具体实现”都会说明先看哪个文件、重点看哪些方法。同一个核心文件再次出现时，只会指出新的相关方法。

1. [架构总览](./01-architecture-overview.md)：正式产品路径、包边界，以及未接入正式入口的组件。
2. [启动与运行时装配](./02-runtime-bootstrap.md)：确定会话与 cwd，准备 services，再创建会话并交给 mode。
3. [模型、认证与供应商运行时](./03-model-and-auth-runtime.md)：ModelRuntime 创建、凭据与登录、provider 组合、流式协议、代理和缓存。
4. [资源加载与扩展](./04-resources-and-extensions.md)：设置与信任如何影响资源发现，扩展如何补齐 provider、工具和提示定义。
5. [AgentSession 与 Agent 执行](./05-agent-session-and-execution.md)：输入预处理、请求与事件时序、模型—工具循环、工具并发、停止与预算边界。
6. [会话、上下文压缩与持久化](./06-session-and-persistence.md)：历史如何生成上下文，压缩与恢复如何工作，会话替换如何重建依赖。
7. [TUI 与交互](./07-tui-and-interaction.md)：会话事件如何成为终端组件，以及差量渲染、焦点、overlay 和 footer。
8. [跨进程协议与工程实践](./08-remote-protocol-and-engineering.md)：实验性远程路径、测试与发布、行为评测、安全、失败重试、取消、并发和可观测性。

主线按“启动总图 → 准备模型依赖 → 加载资源与扩展 → 执行请求 → 保存与恢复 → 呈现结果”展开。它是阅读依赖顺序：模型章会连同流协议、认证和缓存一起解释 provider 层，不表示模型请求在创建 Agent 之前已经发生。最后一章补充跨进程运行与整体工程取舍。

按需查阅的两组专题不计入主线编号：

- [Package 架构专题](./package-docs/README.md)：按 package 查组件与设计边界，与主线有内容重合。
- [核心调用链](./call-flows/README.md)：逐函数追踪跨 package 的调用顺序、对象生命周期、重建边界和失败路径。

## 一句话结论

正式发布的 `pi` 仍是 `pi-coding-agent` 组装出的编码 Agent：`AgentSession` 负责产品会话，`pi-agent-core` 的 `Agent` 和 agent loop 负责模型—工具循环，`pi-ai` 负责模型供应商接口，`pi-tui` 负责终端 UI。与此同时，仓库已经实现另一条实验路径：durable `AgentHarness` 在独立 session worker 中运行，Chord 负责 facet/service 组合与状态复制，`pi-protocol`/`pi-client`/`pi-server` 负责跨进程路由。两条路径共享底层包，但正式默认 CLI/SDK 尚未切换到 Harness。

## 分析边界

- 文档描述当前 HEAD 的可达代码，不根据 Git 历史推断迁移计划。
- “源码存在”“有测试”“被正式 CLI/SDK 调用”是三种不同状态，文中会明确区分。
- 供应商和模型目录变化频繁，因此重点分析机制，不逐一枚举模型。
- 未执行真实模型请求或交互式 TUI；这些会产生外部调用且不是静态架构分析所必需。
