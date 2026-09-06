# Pi 项目架构分析

本文档集基于 `main` 分支提交 `9767ba275`（2026-09-06）进行静态分析。分析对象包括源码、包清单、项目文档、测试入口、CI 和发布脚本；关键结论均回溯到当前源码。

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
2. [启动与请求时序](./02-runtime-and-request-flow.md)：一次本地请求怎样经过 UI、会话、模型和工具。
3. [Agent 内核与工具循环](./03-agent-core.md)：循环条件、工具并发、队列插入、事件顺序，以及 `AgentSession` 与 `AgentHarness` 的区别。
4. [会话、上下文压缩与持久化](./04-session-and-persistence.md)：上下文怎样从历史生成，压缩怎样触发、切分、总结和恢复。
5. [模型、供应商与缓存](./05-ai-provider-layer.md)：统一模型接口、流协议、鉴权、prompt cache 和模型目录缓存。
6. [扩展系统、资源加载与 TUI](./06-extension-resource-tui.md)：扩展为何是产品组合边界、生命周期、信任边界和终端渲染。
7. [跨进程协议与工程治理](./07-remote-protocol-and-engineering.md)：Chord service、protocol/client/server、实验性 session worker、测试架构、供应链和发布。
8. [Agent 工程实践](./08-agent-engineering-practices.md)：evaluation、安全、失败恢复、并发、可观测性和成本控制，以及当前实现的边界。
9. [Package 架构专题](./package-docs/README.md)：从各 package 的 docs 中抽取的中文架构说明，不包含安装和使用教程。
10. [核心调用链](./call-flows/README.md)：按运行流程追踪跨 package 的真实调用顺序、对象生命周期、重建边界和失败路径。

第 9 项适合按 package 查问题，与前八章内容有重合；沿主线学习时看到第 8 章即可。

第 10 项不按 package 切断流程。每篇文档会标出主负责 package 和跨包边界，适合从入口函数一路跟到最终副作用。

## 一句话结论

正式发布的 `pi` 仍是 `pi-coding-agent` 组装出的编码 Agent：`AgentSession` 负责产品会话，`pi-agent-core` 的 `Agent` 和 agent loop 负责模型—工具循环，`pi-ai` 负责模型供应商接口，`pi-tui` 负责终端 UI。与此同时，仓库已经实现另一条实验路径：durable `AgentHarness` 在独立 session worker 中运行，Chord 负责 facet/service 组合与状态复制，`pi-protocol`/`pi-client`/`pi-server` 负责跨进程路由。两条路径共享底层包，但正式默认 CLI/SDK 尚未切换到 Harness。

## 分析边界

- 文档描述当前 HEAD 的可达代码，不根据 Git 历史推断迁移计划。
- “源码存在”“有测试”“被正式 CLI/SDK 调用”是三种不同状态，文中会明确区分。
- 供应商和模型目录变化频繁，因此重点分析机制，不逐一枚举模型。
- 未执行真实模型请求或交互式 TUI；这些会产生外部调用且不是静态架构分析所必需。
