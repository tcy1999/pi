# Agent 工程实践：评测、安全与可靠性

本章把前面分散的机制放回 Agent 工程问题中判断。evaluation（行为评测）是用真实模型重复运行任务并衡量结果，而不是只验证某个函数是否按固定输入返回固定输出。安全边界是系统明确保证权限不会越过的位置。可靠性是模型、网络、工具或进程失败后，系统仍能保持状态可解释并有限恢复的能力。可观测性是通过事件、指标和 trace 判断系统正在做什么以及为何失败。

## 1. 先给覆盖结论

Pi 对运行时机制的实现深度高于它当前的系统评测覆盖：

| 工程问题 | 当前实现 | 当前限制 |
|---|---|---|
| Agent loop | 完整实现模型流、工具调用、steering/follow-up、abort 和事件归约 | 通用 loop 不负责产品持久化、压缩或权限策略 |
| 上下文管理 | 正式产品已有树形历史、上下文投影、compaction、branch summary 和 overflow 恢复 | 摘要由模型生成，仍可能丢失细节；压缩后 prompt cache 前缀会变化 |
| Evaluation | 用真实 `AgentSession`、真实模型、隔离目录、判分器、重复运行和成对比较 | 当前仓库只有基础 smoke 与 extension authoring 两组 eval，尚不是覆盖压缩、恢复、安全和工具质量的完整基准集 |
| 安全 | 有 project trust、扩展 tool hook、请求时凭据解析，以及可选外部 sandbox 示例 | 默认工具与扩展继承启动用户权限；Pi 没有内建沙箱，也不声称解决 prompt injection |
| 失败恢复 | 正式路径有有限 retry/overflow 恢复；Harness 有 durable operation、effect checkpoint、resume 和 replay policy | 两条路径都不承诺外部效果 exactly-once；Harness 仍可能生成 interrupted result |
| 并发一致性 | 单 active run、两类输入队列、并行工具的确定结果顺序、同路径写入队列 | 慢 hook 或监听器会阻塞主链路，这是确定顺序的代价 |
| 可观测性 | 正式路径有事件/usage/session；agent-core 已定义 typed schema、显式 Context 和部分 tool-hook span | 大多数 Harness runtime span 与跨进程 trace parent 重建尚未实现 |

因此，“理解 Pi 重要实现”不能只读包地图。默认产品仍以小型 Agent loop + `AgentSession` 组合产品策略；实验路径则以 durable `AgentHarness` + Chord services 组合跨进程运行。两者都要看清主动不解决的部分：默认权限隔离、外部效果 exactly-once 和成熟的全场景 eval 基准。

## 2. Evaluation：衡量端到端行为，而不是函数细节

普通测试适合验证 codec、reducer、截断或 session 后端契约等确定行为。Agent 的最终质量还取决于模型、system prompt、工具描述、资源加载和多轮决策，因此 Pi 另有私有的 `pi-evals` package，直接适配正式 `AgentSession`。

一次 eval run 的路径是：

```text
任务输入
  → 临时 workspace 与临时 agentDir
  → 真实 ModelRuntime + AgentSession
  → prompt / reload 等步骤
  → 最终回答 + 标准化消息/工具轨迹 + usage
  → judge（判分器）评分
  → session JSONL 与来源文件 artifact（运行附件）
```

这里有四个重要设计选择：

1. 使用正式 `AgentSession`，而不是另造一个简化 Agent，所以工具、扩展、资源 reload、usage 和持久化路径与产品一致。
2. 每次运行创建隔离的 workspace、agentDir 和内存 settings，避免本机扩展或项目配置污染结果；模型认证仍复用正常 `ModelRuntime`。
3. 同一输入可以运行 baseline（基线配置）和 candidate（候选配置）多次。报告比较通过率变化，同时单独报告 token、延迟和费用差值，避免把更贵的方案仅因得分更高就视为无条件更好。
4. eval 保存原生 session JSONL 和必要源码，便于在分数之外检查模型调用了什么工具、为何失败。artifact 是随评测保存的运行附件，可能包含 prompt、源码和工具输出，因此应按敏感运行数据处理，而不是普通公开报告。

当前 `extensions.eval.ts` 展示的是 Pi 很有代表性的评测：对比完整 system prompt 与删减版本，要求模型创建扩展、reload、调用新工具，再由判分器同时检查生成源码、扩展加载结果、工具轨迹和最终回答。它评测的是完整工作流，不只是回答文本。

但当前覆盖仍窄。仓库尚未形成针对长会话压缩质量、overflow 恢复、危险工具拒绝、不同模型工具成功率、跨平台 shell 行为和故障注入的系统 eval 集。这是当前工程能力的缺口，不应因为已有 `packages/evals` 就宣称 evaluation 已完整解决。

### 2.1 具体实现

1. 读 [`pi-harness.ts`](../packages/evals/src/pi-harness.ts)：确认 eval 怎样创建隔离的正式 `AgentSession`，怎样提取轨迹、usage 和 session artifact。
2. 读 [`extensions.eval.ts`](../packages/evals/src/extensions.eval.ts)：确认一个候选配置怎样经过多步任务、reload 和 judge 形成端到端评测。
3. 读 [`harness-table.ts`](../packages/evals/src/vitest-evals/harness-table.ts)：确认 baseline、candidate、repetition 和输入分组怎样建立可比较样本。
4. 读 [`reporter.ts`](../packages/evals/src/vitest-evals/reporter.ts) 与 [`summary.ts`](../packages/evals/src/vitest-evals/summary.ts)：确认通过率、token、延迟和费用怎样分别汇总。
5. 读 [`artifacts.ts`](../packages/evals/src/vitest-evals/artifacts.ts)：确认会话与生成源码怎样在临时目录删除前持久化。

## 3. 安全：Pi 明确选择本地信任模型

Pi 的基本安全模型不是“Agent 默认被限制”，而是“Pi 与启动它的本地用户处于同一权限边界”。内建 `read`、`write`、`edit` 和 `bash` 可以使用该用户拥有的文件与进程权限；extension 也是同进程受信任代码。

### 3.1 Project trust 只保护启动输入

project trust 判断项目本地 settings、extensions、skills、prompts、themes 和 system prompt 文件能否进入运行时。判断按规范化目录保存，最近的父目录决定可以继承。未信任项目不能让自己的 extension 参与 trust 决策，避免待审批代码批准自己。

这个机制解决的是“进入目录时，仓库能否静默改变 Pi 配置或执行扩展”，不解决“开始工作后，模型能否提出危险 shell 命令”。`AGENTS.md` 等 context files 仍可能进入模型输入；仓库文本或工具输出还可能包含 prompt injection（诱导模型偏离用户意图的恶意指令），Pi 没有声称能在进程内可靠消除它。

### 3.2 工具边界提供策略插入点，不等于默认策略

在工具执行前，`AgentSession` 把 `tool_call` 交给 extension runner。扩展可以修改输入或返回 block；检查 hook 抛错时执行会被阻止，而不是绕过检查。仓库的 `permission-gate` 示例在危险 bash 命令前询问用户，非交互模式默认拒绝。

但这个 gate 是示例扩展，不是所有安装的默认权限系统。类似地，sandbox 示例通过替换 bash 执行环境接入操作系统级限制，也不是 Pi 核心默认启用的隔离。真正需要限制文件、网络或凭据时，应把整个 Pi 或工具执行放入容器、VM、micro-VM 或操作系统策略中。

还有一个较小但重要的执行保护：模型响应若因输出 token 限制而截断，Pi 不执行其中任何 tool call。即使残缺参数碰巧能被 JSON 修复并通过 schema，也不能证明它仍代表模型原本意图。

### 3.3 凭据和运行数据同样属于安全面

模型目录展示与认证执行分开。动态 API key、OAuth refresh 和命令型凭据尽量在真正请求时解析，避免仅打开模型列表就触发外部命令或暴露短期值。另一方面，session、eval artifact 和完整工具输出可能包含源码、prompt、路径或秘密；它们需要和工作区数据采用相同保护级别。

### 3.4 具体实现

1. 读 [`trust-manager.ts`](../packages/coding-agent/src/core/trust-manager.ts)：确认 trust 决策怎样规范化、继承、加锁并持久化。
2. Project trust 怎样影响资源加载，接着看[项目信任的具体实现](./06-extension-resource-tui.md#41-具体实现)。
3. 工具拦截和截断保护分别见[扩展具体实现](./06-extension-resource-tui.md#81-具体实现)中的 `_installAgentToolHooks()`，以及[工具循环](./03-agent-core.md#25-具体实现)中的 `failToolCallsFromTruncatedMessage()`。
4. 对照 [`permission-gate.ts`](../packages/coding-agent/examples/extensions/permission-gate.ts) 与 [`sandbox/index.ts`](../packages/coding-agent/examples/extensions/sandbox/index.ts)：区分进程内策略确认和操作系统级隔离。
5. 读 [`resolve.ts`](../packages/ai/src/auth/resolve.ts)：确认存储凭据、环境凭据和 OAuth refresh 怎样在请求边界组合。

## 4. 失败恢复：先让状态可解释，再做有限重试

Pi 没有把所有失败统一成“再跑一次”。不同失败发生在不同边界：

- provider 或网络瞬时错误使用有限次数、指数退避且可中止的 retry；额度、账单和明确非瞬时错误快速失败。
- context overflow 不进入普通 retry，而是移除失败 assistant 消息、压缩上下文、重建状态并只重试一次。
- 工具业务错误变成 `toolResult` error 回到模型，让模型决定如何修正；基础设施异常才结束 run。
- 用户 abort 会传播到 provider、退避等待和工具，并归一化为完整 `aborted` assistant message，避免留下无法解释的半条流式消息。
- 已完成的 user、assistant 和 tool result 在 `message_end` 时逐条追加，而不是等整个 run 成功才统一保存。

这套策略的共同点是保持明确终态和有限恢复。但正式 `AgentSession` 仍无法保证外部工具效果只发生一次：进程可能在命令已经修改文件、tool result 尚未落盘时崩溃。

`AgentHarness` 已实现更强的恢复边界。operation admission 原子记录 intent；Drive 在 provider/tool 效果前写 effect-pending 状态，在效果后把 frame、输出、usage 与下一状态一起提交；retry wait、cancel request 和 terminal result 也可恢复。安全重放的工具可以继续，不可重放工具在结果不确定时生成 synthetic interrupted result。这个设计把不确定性显式化，但仍不是 exactly-once：外部效果或模型计费可能已发生，单机事务不能撤销。

### 4.1 具体实现

1. 读 [`retry.ts`](../packages/ai/src/utils/retry.ts)：确认错误分类、retry budget、指数退避和中止归一化。
2. 普通 retry 与 overflow recovery 怎样接入 `AgentSession`，看[上下文压缩的具体实现](./04-session-and-persistence.md#36-具体实现)，再定位 `_prepareRetry()`。
3. Agent loop 怎样处理 error、abort 和 tool result，见[核心 loop](./03-agent-core.md#25-具体实现)。
4. 消息何时落盘，可以在[一次请求的实现链](./02-runtime-and-request-flow.md#23-具体实现)中跟踪 `_handleAgentEvent()`。
5. Harness 的 operation/recovery 实现与剩余切片见 [`AgentSession` 与 `AgentHarness`](./03-agent-core.md#31-具体实现)。

## 5. 并发：允许工作并行，但固定状态提交顺序

Pi 同一 `Agent` 只允许一个 active run。用户在运行期间的新输入必须明确成为 steering 或 follow-up，避免两个 run 同时改写同一消息数组。单批工具可以并行执行，但 `ToolResultMessage` 按原始 tool-call 顺序进入 context；UI 进度仍按真实完成时间交错。文件工具再通过 mutation queue 串行化同一路径写入。

事件监听器和需要返回值的 extension hook 会被顺序等待。这降低并发度，却保证下一阶段开始前，产品状态、扩展决策和持久化处于确定顺序。Pi 的取舍不是最大吞吐，而是让模型下一次看到的上下文可以复现。

Harness 把并发边界提升到 lane：不同 lane 可共享 session 历史前缀并独立运行，但每条 lane 最多一个 operation；会话级 mutation line 再串行化跨 lane 的原子 commit。session worker 还用文件锁建立进程级所有权，server attachment 只授予 presentation-scoped 调用能力。这些层次分别解决“一个 lane 的状态机顺序”“一个 session 的写入顺序”和“一个进程拥有 durable 文件”，不能互相替代。

### 5.1 具体实现

并发相关代码分布在三个现有实现点：[核心 loop](./03-agent-core.md#25-具体实现)中的 `PendingMessageQueue` 和 `executeToolCallsParallel()`，[内建工具](./03-agent-core.md#51-具体实现)中的 `file-mutation-queue.ts`，以及[扩展实现](./06-extension-resource-tui.md#81-具体实现)中 `ExtensionRunner` 的顺序 handler 循环。

## 6. 可观测性、成本与性能

正式 coding-agent 当前最可靠的观测事实是 `AgentSessionEvent`、每条 assistant usage、工具进度、session JSONL 和 eval artifact。费用与 cache 统计直接从 provider usage 派生，不根据本地猜测宣称 cache hit。

`pi-telemetry` 还定义了 provider request、operation、turn、retry step、tool、hook、event handler 和 session write 等 span 契约。当前已落地的是显式 `Context` 传播、typed span helper，以及 `before_tool`/`after_tool` 的部分 hook span；大多数 Drive、event handler 和 session write span 仍未接线。正式 `AgentSession` 也没有自动产生这些 Harness span，跨进程 trace carrier 注入/提取尚未实现，不能把 schema 等同于完整分布式 trace。

另一个容易混淆的机制是 install telemetry：它是可配置的安装/版本报告，不等于上述 Agent 运行 trace。

性能优化分散在正确边界：provider prompt cache 复用稳定前缀；API 实现延迟加载减少启动成本；工具输出截断控制 context；TUI 差量渲染减少终端写入；并行工具降低独立调用的等待时间。这些优化都没有改变 agent loop 的基本语义。

### 6.1 具体实现

1. usage 与 cache 指标的数据来源见 [prompt cache 的具体实现](./05-ai-provider-layer.md#91-具体实现)。
2. 读 [`telemetry.ts`](../packages/agent/src/harness/telemetry.ts)、[`hooks.ts`](../packages/agent/src/harness/hooks.ts)、[`telemetry.md`](../packages/agent/docs/telemetry.md) 与 [`telemetry-schema.md`](../packages/agent/docs/telemetry-schema.md)，区分已定义 schema、已接入 hook 和剩余 instrumentation。
3. eval 怎样记录 token、费用、延迟和原生 session，见本章的[评测实现顺序](#21-具体实现)。
4. 读 coding-agent 的 [`telemetry.ts`](../packages/coding-agent/src/core/telemetry.ts)，确认 install telemetry 与 Agent trace 是两件事。

## 7. Pi 的整体设计哲学

把这些机制放在一起，Pi 的设计取向可以概括为源码中反复出现的四个选择：

1. **小而通用的控制循环**：`Agent`/agent loop 只处理模型、工具、队列和事件；正式产品策略留给 `AgentSession`。
2. **保存事实，派生模型视图**：session 保留完整历史树，context、compaction 和 branch summary 是面向下一次模型请求的投影。
3. **在明确边界开放定制**：extension 通过声明过的 hook、tool、provider 和 UI API 改变行为，而不是要求通用 loop 理解每种产品需求。
4. **不伪装安全与可靠性保证**：本地工具默认拥有用户权限，真正隔离交给 OS；正式产品只做有限 retry 和消息级持久化；Harness 提供 durable recovery，但明确不把它宣传成外部效果 exactly-once。

这四点也是阅读 Pi 时最值得追踪的主线：每遇到一个机制，先判断它属于通用控制、产品策略、模型适配、持久事实，还是扩展边界。这里使用的是仓库已有模块边界，不是额外创造一套架构层次。
