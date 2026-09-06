# Agent 内核与工具循环

这里的 agent loop 指模型与工具之间的控制循环：请求模型；如果模型返回工具调用，就执行工具并把结果加入上下文，再请求模型；没有工具调用时结束。一次 run 是从一个新 prompt 开始到本轮处理完全停止；一个 turn 是其中的一次模型响应以及该响应产生的工具结果。

## 1. `Agent` 的责任

`packages/agent/src/agent.ts` 持有最小运行态：system prompt、模型、thinking level、工具、已完成消息、流式消息、待执行工具和错误。数组赋值会复制顶层数组，避免调用方无意中绕过状态更新边界。

`Agent` 负责：单次活动运行、abort、steering/follow-up 队列、把 loop 事件归约到状态、顺序 await 监听器。它不负责磁盘、UI、扩展发现或具体供应商。

## 2. Agent loop 的精确控制流

`Agent.prompt()` 负责保证同一时刻只有一个 active run，并创建 `AbortController`。真正决定“下一步做什么”的是 `packages/agent/src/agent-loop.ts` 中的 `runLoop()`。

`runLoop()` 有两层循环：

- 内层循环处理模型响应、工具调用和 steering。steering 是用户对当前工作的即时补充，会在当前模型响应及其工具调用结束后、下一次模型请求前插入。
- 外层循环处理 follow-up。follow-up 是等待当前工作自然结束后再开始的追加请求。

一次典型 run 是：

```text
user message
  -> agent_start / turn_start
  -> model response 1
  -> zero or more tool calls
  -> tool results enter context
  -> turn_end
  -> turn_start
  -> model response 2
  -> no tool calls
  -> turn_end / agent_end
```

下一轮不是工具自己发起的。`runLoop()` 看到工具结果后令 `hasMoreToolCalls` 保持为真，于是内层循环再次调用 `streamAssistantResponse()`。

### 2.1 每次模型请求前可以改变什么

从第二个 turn 开始，loop 先调用可选的 `prepareNextTurn()`。coding-agent 用这个位置执行可能的上下文压缩，并允许替换下一轮的 context、model 或 thinking level。准备期间新到达的 steering 也会在模型请求前再次读取。

随后 `streamAssistantResponse()`：

1. 用 `transformContext()` 处理 Agent 自己保存的消息。
2. 用 `convertToLlm()` 去掉只供产品或界面使用的消息字段，得到供应商可接收的消息。
3. 组合 system prompt、消息和工具定义。
4. 在请求时解析可能过期的 credential。
5. 消费供应商返回的流式事件，并转换成 `message_start/update/end`。

这里故意把 `AgentMessage[]` 到供应商 `Message[]` 的转换推迟到请求边界。产品可以在内存和磁盘中保留 custom、bash、compaction 等消息，而 provider 不必理解这些产品类型。

### 2.2 工具并发与确定的上下文顺序

若整个 loop 配置为 sequential，或本批任意工具声明 `executionMode: "sequential"`，本批工具按顺序执行；否则并行执行。

并行时有两个不同顺序：

- `tool_execution_update/end` 按真实进度发生，因此不同工具的事件可能交错。
- `ToolResultMessage` 等所有工具完成后，按 assistant 原始 tool-call 顺序加入 context。

这样 UI 能实时显示先完成的工具，同时下一次模型请求仍获得确定的消息顺序。参数会先经过 `prepareArguments` 和 TypeBox schema 校验，再经过 `beforeToolCall` hook；执行结束后再经过 `afterToolCall` hook。hook 是 Pi 在固定生命周期位置调用的扩展函数。

### 2.3 退出条件

内层循环在没有更多工具调用且没有 steering 时退出。以下情况会更早结束整个 run：

- assistant message 的 stop reason 是 `error` 或 `aborted`。
- `shouldStopAfterTurn()` 返回 true。
- 一批实际执行的工具结果全部带 `terminate: true`，因此不再为了这些结果继续请求模型。

内层结束后，外层读取 follow-up；有消息则重新进入内层，否则发出 `agent_end`。coding-agent 还会在底层 `agent_end` 后处理自动 retry、overflow compaction 和扩展新加入的队列，因此产品层的 `agent_settled` 才表示不会自动继续。

### 2.4 事件为什么按顺序等待

事件层次对应控制流：`agent_start/end` 包围 run，`turn_start/end` 包围一次模型响应和其工具结果，`message_*` 描述消息，`tool_execution_*` 描述工具。`Agent` 顺序等待每个订阅者处理事件；这让 `AgentSession` 能在运行进入下一阶段前执行扩展并持久化完成消息。

代价是慢事件处理函数会直接增加请求链路延迟。这里选择的是确定的状态顺序，而不是把所有观察者都变成不受控制的后台任务。

### 2.5 具体实现

理解 Agent 内核主要看这两个文件，顺序如下：

1. 读 [`agent-loop.ts`](../packages/agent/src/agent-loop.ts) 的 `runAgentLoop()`、`runLoop()`、`streamAssistantResponse()` 和 `executeToolCalls()`，建立控制流全貌。
2. 读 [`agent.ts`](../packages/agent/src/agent.ts) 的 `prompt()`、`runPromptMessages()`、`runWithLifecycle()` 与 `processEvents()`，确认状态和生命周期怎样包围 loop。

## 3. `AgentSession` 与 `AgentHarness`

`AgentSession` 是正式 coding-agent CLI/SDK 使用的产品运行时。`AgentHarness` 是 agent-core 中尚未完成的 durable runtime 外壳；其设计希望组合以下通用能力：

- `ExecutionEnv`：文件系统、shell、路径和临时文件端口。
- `Session`：追加消息、operation 和 usage。
- compaction/branch summarization。
- prompt templates、skills 和 system prompt 构建。
- typed telemetry。
- 工具集合和事件 reducer。

其中 durable session、compaction、工具、类型和部分配置访问已经实现并可独立测试；完整 run 调度尚未实现。当前 `prompt`、`skill`、`compact`、`navigateTree`、`resume`、队列、lane 和 watch 等公开操作都会返回 `HarnessNotImplemented`，已有 operation record 的恢复创建也未实现。

仓库里只有一个 `AgentHarness` 类。当前正式 CLI/SDK 由 `AgentSession` 直接持有 `Agent`，使用自己的 `SessionManager`、扩展系统和 compaction/branch summarization。不能把未完成的 Harness 画进正式请求路径：

```text
正式 CLI/SDK：AgentSession → Agent / agent-loop
                         → SessionManager JSONL

Harness 工程：AgentHarness 外壳（run 调度未完成）
             → 已实现的 durable Session API
             → JSONL / 内存 / SQLite backend
             → 已实现的 compaction / tools 等基础模块
```

英文 `packages/agent/docs/harness.md` 是目标实现规格，不是当前完成度清单。正式产品是否使用某个运行时，应从 CLI/SDK 入口的实际调用链判断；当前正式入口只创建 `AgentSession`。

### 3.1 具体实现

1. 先看 `agent-session.ts` 的构造函数，确认正式产品怎样直接组合 `Agent`。
2. 读 [`agent-harness.ts`](../packages/agent/src/harness/agent-harness.ts)，列出 Harness 已接入与未接入的操作。
3. 最后读目标规格 [`harness.md`](../packages/agent/docs/harness.md)，理解未来方向而不是推断当前调用链。

## 4. System prompt 与能力必须同步

system prompt 是每次模型请求最前面的行为说明。Pi 不把它视为一段固定常量：`AgentSession` 根据当前启用工具收集工具摘要和使用约束，再由 `buildSystemPrompt()` 组合基础角色、可用工具、通用 guidelines、Pi 文档位置、项目 context files、skills 和 cwd。

这里的关键不是“prompt 写得长”，而是模型描述与真实能力一致：

- 只有当前启用且提供 prompt snippet 的工具才进入 `Available tools`。
- `read`、`edit`、`bash` 等工具可以贡献自己的 guideline；同一 guideline 会去重。
- skills 只有在 `read` 可用时加入，因为模型需要通过读文件取得 skill 的完整内容。
- 项目 `AGENTS.md` 等 context file 用带来源路径的标签加入，保留指令来自哪里。
- extension 改变 active tools 或 reload 资源后，`AgentSession` 重建基础 system prompt；`before_agent_start` 还能只为本次 run 覆盖它。

工具 schema 与 system prompt 承担不同责任：schema 约束参数是否可解析，prompt snippet 和 guideline 告诉模型何时、怎样使用工具。只有 schema 而没有行为说明，模型仍可能选错工具；只有文字说明而没有 schema，则无法在执行前可靠验证参数。

### 4.1 具体实现

1. 读 [`system-prompt.ts`](../packages/coding-agent/src/core/system-prompt.ts) 的 `buildSystemPrompt()`，确认各输入怎样组成最终文本。
2. 接着看 `agent-session.ts` 的 `_rebuildSystemPrompt()` 与 `_refreshToolRegistry()`，确认 active tools、tool snippets 和 guidelines 怎样保持同步。
3. 到[扩展 API](./06-extension-resource-tui.md#81-具体实现)中的 `ToolDefinition`，看 schema、prompt metadata 和 renderer 怎样组成一个工具定义。
4. 读 [`skills.ts`](../packages/coding-agent/src/core/skills.ts)，确认 skill 怎样格式化进 prompt；资源的来源与加载过程见[资源系统](./06-extension-resource-tui.md#41-具体实现)。

## 5. 内建工具的设计

工具不是对 Node API 的薄包装，而是带 Agent 语义的安全适配器：

- `read` 支持文本分页、字节/行截断、图片 MIME 检测和可注入图片处理。
- `bash` 捕获 stdout/stderr，按 100ms 节流进度，尾部截断并把完整输出写入临时文件。
- `edit` 对原文件做唯一、非重叠的精确替换，保留 BOM/换行风格，并返回人类 diff 与统一 patch。
- 文件写工具通过 mutation queue 串行化同一路径写入，降低并发工具互相覆盖。

问题示例：如果一个 bash 命令输出 20MB，直接塞进上下文会耗尽 token；只返回前几行又常丢失最终错误。解决方案是保留尾部、提示截断范围并提供完整输出路径。

### 5.1 具体实现

1. 读 [`tools/index.ts`](../packages/coding-agent/src/core/tools/index.ts)，确认工具工厂怎样注入 cwd 和执行能力。
2. 按关心的问题选择 [`read.ts`](../packages/coding-agent/src/core/tools/read.ts)、[`bash.ts`](../packages/coding-agent/src/core/tools/bash.ts) 或 [`edit.ts`](../packages/coding-agent/src/core/tools/edit.ts)，理解参数验证、截断和结果格式。
3. 再对照 2.5 中的 `executeToolCalls()`，确认具体工具怎样进入共同调用生命周期。
4. 读 [`file-mutation-queue.ts`](../packages/coding-agent/src/core/tools/file-mutation-queue.ts)，确认同一路径写入为何必须串行化。

## 6. 中止与错误边界

AbortSignal 从 run 传到 provider 和工具。模型中止会生成 `aborted` assistant message，而不是让历史出现没有终态的流式消息。工具失败转换为 `toolResult` error，使模型有机会解释或恢复；基础设施级错误才终止 run。

重试分布在不同层，不能统一称为“Agent retry”：`pi-ai` 提供单次 assistant 调用的通用 retry helper，具体 provider 也可能处理传输级重试；未完成的 `AgentHarness` 外壳目前只保存 `RetryPolicy` 配置，尚未执行重试；正式 coding-agent 的 `AgentSession` 会根据完整 assistant error、产品设置和 UI 生命周期安排下一次 turn。默认策略避免在额度耗尽等非瞬时错误上长期隐藏真实状态。

## 7. Reducer 与不变量

状态变化集中通过事件 reducer 表达，重要不变量包括：

- 同一时刻只有一个 active run。
- `pendingToolCalls` 由 start/end 成对维护。
- completed message 只在 `message_end` 追加。
- streaming message 在 `message_start/update` 更新，在 end 清除。
- idle 必须晚于所有 awaited listener。

这种设计使 UI 可以只订阅事件而无需猜测内部阶段，也使 session 持久化能与 Agent 状态保持相同顺序。
