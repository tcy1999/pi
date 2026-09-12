# 跨进程协议与工程实践

这里的 protocol 是跨进程路由 envelope；Chord 定义 envelope 内的 service 调用与 replicated state；client/server 负责连接和路由；coding-agent 的实验性 server/session worker 才提供真正执行模型和工具的 Agent。

## 1. 先明确当前状态

仓库现在有一条可运行但仍标记为实验性的远程 coding-agent 路径：

- pi-protocol、pi-client、pi-server 和 Chord 是独立发布包。
- 默认 pi CLI/SDK 仍创建旧的 AgentSession，不经过这条路径。
- 设置 PI_EXPERIMENTAL=1 后，实验性 pi server/pi client 会启动 server、client、coordinator 和独立 session worker。
- 每个 session worker 打开 durable JSONL Session，创建 AgentHarness/main lane 和一个 Chord FacetHost。
- client 使用正式 coding-agent 的 alternate-screen renderer 展示 Chord Transcript replica，并通过 AgentController service 驱动 worker。

因此不能再说“仓库没有 Harness server 接入”；准确边界是“实验路径已端到端接入，但默认产品入口和稳定 SDK 尚未切换”。

## 2. 四层分别做什么

~~~mermaid
flowchart LR
  UI["实验性 client / presentation"] --> CLIENT["pi-client\nrequest / cancel / attachment"]
  CLIENT <-->|"长度前缀 + CBOR"| SERVER["pi-server\n连接与 route fence"]
  SERVER --> HOST["coding-agent ServerHost"]
  HOST --> WORKER["每会话 worker\nAgentHarness + FacetHost"]
  WORKER --> MODEL["ModelRuntime / Provider"]
  WORKER --> ENV["NodeExecutionEnv"]

  PROTOCOL["pi-protocol\n外层 envelope"] -.-> CLIENT
  PROTOCOL -.-> SERVER
  CHORD["Chord\nservice / facet / replica / Delta"] -.-> CLIENT
  CHORD -.-> SERVER
  CHORD -.-> WORKER
~~~

- pi-protocol 定义 hello、request/response/cancel、service update、attachment change、路由 target、strict-JSON 边界、CBOR 和 framing。
- Chord 定义 $chord.service catalogue/subscribe/unsubscribe、typed service、singleton/keyed instance、replicated state 与 Delta；它不规定外层网络协议。
- pi-client 维护 request ID、pending promise、取消、连接状态和当前 SessionTarget，并把 route 适配成 Chord transport。它不解释具体应用 service。
- pi-server 校验 route，把 server-wide 调用交给 RoutedServerServiceHost，把 session 调用交给 RoutedSessionAttachment；它等待已接纳调用结束后再释放 attachment。
- coding-agent facets 定义实际服务，例如 SessionDirectory、SessionManagement、PresentationPlugins、SessionPlugins、Models、AgentController、Transcript、SlashCommands 和 PresentationUI。

## 3. 路由、attachment 与进程生命周期

第一帧必须是 protocol hello。client 预先知道稳定的 logical serverId，握手会验证物理 endpoint 没有接到错误 server。server-wide target 是 { serverId }；session target 还包含 { sessionId, attachmentId }。attachmentId 由 server 为当前 presentation 生成，可拒绝切换会话后迟到的旧 frame。

management service 的 attach/detach 不直接返回 route；server 通过 out-of-band attachment message 发布当前 target。连接断开只释放该 presentation 的 attachment，不删除 durable session。client 会 reject 本地 pending 请求并清除 route，但不会自动 reconnect 或重放；调用方必须重新连接、重新 attach，并只重复已知安全的操作。

coding-agent 进一步把生命周期拆成 coordinator、可替换 server 和每会话 worker：

~~~text
稳定 Unix endpoint
  → coordinator（只转发字节与私有控制消息）
  → 当前 server generation
  → SessionWorkerManager
  → 独立 worker（文件锁 + JsonlSessionRepo + AgentHarness）
~~~

coordinator 不解释 Pi payload；server generation 替换时会断开旧 public connections，并通知 worker。worker 根据当前 attachment demand、active operation 和 grace period 判断是否退休，使 presentation 断开不会立即杀死仍在执行的 durable run。

## 4. 协议格式与状态同步

每个 frame 是四字节大端长度加一个 definite-length CBOR item，默认上限 16 MiB。protocol schema 拒绝未知 envelope 字段，并递归拒绝非 strict JSON 的 payload；但 service payload 对它是 opaque value。Chord adapter 在边界继续解析 { serviceId, instance?, member, args }、catalogue、subscription snapshot/update 和 service error。

replicated state 的 provider 发布完整逻辑状态变化，Chord 在 publication 时生成 Delta operations。每个订阅独立维护 path encoder/decoder：首次 hydration 是完整 base，后续是有序增量；provider 替换、断线或重新 hydration 时重置字典。client 先安装 subscription snapshot，再调用 start() 释放 hydration 期间缓冲的 update，避免 snapshot 与增量之间丢事件。

Transcript 是普通 Chord service，不是 protocol 内建消息类型。服务端拥有权威 Harness lane snapshot，presentation 看到完整 immutable replica；外层协议只运送 subscription update。

## 5. Facet 与 service 为什么在这里

远程功能不能靠一份手写 RPC 命令表扩展，否则 worker、server 和 TUI 每加一个插件能力都要同步修改协议。Chord facet 在 setup 时声明提供和消费的 service；host 验证完整依赖图，先激活 provider、再激活 consumer，并逆序 dispose。非 local service 自动进入 catalogue，远端只绑定实际需要的 token。

插件可把 session 与 presentation 代码打成独立、带 SHA-256 的 facet bundle。reload 先加载和验证 candidate，在旧 provider 仍可用时切换实现，再释放 retired generation。singleton consumer 保留稳定 facade；keyed instance 使用新 generation，避免把旧 instance 混入新实现。

### 5.1 具体实现

1. 读 [protocol.ts](../packages/protocol/src/protocol.ts)、[codec.ts](../packages/protocol/src/codec.ts) 和 [framing.ts](../packages/protocol/src/framing.ts)，确认外层 wire 边界。
2. 读 [client.ts](../packages/client/src/client.ts)，确认 request、cancel、attachment 与 Chord service transport。
3. 读 [server.ts](../packages/server/src/server.ts)、[session-router.ts](../packages/server/src/session-router.ts) 和 [types.ts](../packages/server/src/types.ts)，确认 target fencing 与 attachment 释放。
4. 读 [session-worker.ts](../packages/coding-agent/src/experimental/session-worker.ts) 和 [services/worker.ts](../packages/coding-agent/src/experimental/services/worker.ts)，确认 Harness 与 Chord service 的真实接入。
5. Chord 的 facet、service、replicated state 和 bundle 见 [Chord 架构专题](./package-docs/09-chord-facets-services-and-delta.md)。

## 6. 测试架构

| 层 | 方法 | 目的 |
|---|---|---|
| 纯逻辑 | Vitest / node:test | reducer、codec、Delta、布局和截断等确定性行为 |
| 契约一致性 | Storage 与 SessionRepo conformance | 保证 JSONL、Memory、SQLite 的可观察语义一致 |
| Harness 故障矩阵 | faux provider、gating/instrumented storage | 验证 effect checkpoint、恢复、retry、工具落位和 terminal transaction |
| 远程生命周期 | in-memory/Unix transport、worker fixtures | 验证 route、cancel、attachment、server replacement 和 worker retirement |
| Provider | faux provider + fixtures | 无真实密钥验证 streaming、tools、retry 和 compaction |
| 终端 | virtual terminal / tmux | Unicode、cursor、overlay、fullscreen、selection 和搜索回归 |
| 产品回归 | coding-agent/test/suite/regressions | 用最小场景固定 issue 行为 |
| 文档一致性 | 静态导航测试 + model-backed docs eval | 检查站点导航/孤儿页，并逐页核对实现声明 |
| 行为评测 | pi-evals + vitest-evals | 用真实模型比较 prompt、extension、model 和 provider 定制效果 |
| 发布 smoke | 仓库外隔离安装 | 防止 workspace 解析掩盖打包问题 |

根 test.sh 清理相关环境变量并使用临时用户目录，避免本机凭据和配置使测试误调用真实 API。

## 7. 静态质量门与发布

npm run check 执行格式与静态检查、精确依赖、runtime dependency、TS import 与 entry graph 检查、shrinkwrap/install-lock 校验、类型检查和 browser smoke。它不等于测试通过。

项目将供应链作为安全边界：直接外部依赖使用精确版本；CI 以 --ignore-scripts 安装；coding-agent shrinkwrap 固定发布侧传递依赖；带 lifecycle script 的新依赖必须显式审核。Chord facet bundler 只构建已有 package，不安装依赖或执行 package lifecycle script。

release script 同步所有包版本、生成产物、运行检查、创建提交和 tag；tag CI 构建并发布 npm，再验证 tarball 可用后更新公开版本标记。发布 helper 是幂等的，失败时应修复并重跑 job，而不是为同一版本重新执行 release。

### 7.1 具体实现

1. [package.json](../package.json)：确认 check、test、eval 与 release 是不同入口。
2. [test.sh](../test.sh)：确认普通测试怎样隔离本机认证和配置。
3. session 后端怎样共享行为契约，见 [durable session 后端](./06-session-and-persistence.md#71-具体实现)。
4. [faux.ts](../packages/ai/src/providers/faux.ts)：确认无需真实模型的 provider 场景怎样构造。
5. [build-binaries.yml](../.github/workflows/build-binaries.yml)：确认构建、发布和发布后验证的边界。

## 8. 工程机制的覆盖与限制

本章把前面分散的机制放回 Agent 工程问题中判断。evaluation（行为评测）是用真实模型重复运行任务并衡量结果，而不是只验证某个函数是否按固定输入返回固定输出。安全边界是系统明确保证权限不会越过的位置。可靠性是模型、网络、工具或进程失败后，系统仍能保持状态可解释并有限恢复的能力。可观测性是通过事件、指标和 trace 判断系统正在做什么以及为何失败。

Pi 对运行时机制的实现深度高于它当前的系统评测覆盖：

| 工程问题 | 当前实现 | 当前限制 |
|---|---|---|
| Agent loop | 完整实现模型流、工具调用、steering/follow-up、abort 和事件归约 | 通用 loop 不负责产品持久化、压缩或权限策略 |
| 上下文管理 | 正式产品已有树形历史、上下文投影、compaction、branch summary 和 overflow 恢复 | 摘要由模型生成，仍可能丢失细节；压缩后 prompt cache 前缀会变化 |
| Evaluation | 用真实 `AgentSession`、真实模型、隔离目录、文档审计、判分器、重复运行和成对比较 | 已覆盖 smoke、extension/model/provider 定制和文档—实现核对；尚不是覆盖压缩、恢复、安全和工具质量的完整基准集 |
| 安全 | 有 project trust、扩展 tool hook、请求时凭据解析，以及可选外部 sandbox 示例 | 默认工具与扩展继承启动用户权限；Pi 没有内建沙箱，也不声称解决 prompt injection |
| 失败恢复 | 正式路径有有限 retry/overflow 恢复；Harness 有 durable operation、effect checkpoint、resume 和 replay policy | 两条路径都不承诺外部效果 exactly-once；Harness 仍可能生成 interrupted result |
| 循环与执行预算 | 停止 hook、terminate、abort、可选 bash timeout | 通用 loop 无默认最大轮数、重复调用检测或总费用熔断；bash 无默认超时 |
| 并发一致性 | 单 active run、两类输入队列、并行工具的确定结果顺序、同路径写入队列 | 并行批次无数值并发上限；慢 hook 或 Agent 监听器会阻塞主链路 |
| 可观测性 | 正式路径有事件/usage/session；agent-core 已定义 typed schema、显式 Context 和部分 tool-hook span | 大多数 Harness runtime span 与跨进程 trace parent 重建尚未实现 |

因此，“理解 Pi 重要实现”不能只读包地图。默认产品仍以小型 Agent loop + `AgentSession` 组合产品策略；实验路径则以 durable `AgentHarness` + Chord services 组合跨进程运行。两者都要看清主动不解决的部分：默认权限隔离、外部效果 exactly-once 和成熟的全场景 eval 基准。

## 9. Evaluation：衡量端到端行为，而不是函数细节

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
4. eval 保存原生 session JSONL 和必要源码，便于在分数之外检查模型调用了什么工具、为何失败。artifact 目录还保存 `runs.jsonl`、文本/JSON 比较报告与来源附件；这些内容可能包含 prompt、源码和工具输出，因此应按敏感运行数据处理，而不是普通公开报告。

当前评测分成两类。`smoke.eval.ts` 和 `docs.eval.ts` 是普通 eval：前者验证基础回答，后者遍历 `packages/coding-agent/docs/` 中的每个 Markdown 页面，让模型读取完整页面、搜索实现证据，并以受约束工具结果提交 match/mismatch。`extensions.eval.ts`、`models.eval.ts` 和 `providers.eval.ts` 是成对比较：它们对比完整 system prompt 与移除 Pi 文档块后的 baseline，分别验证扩展创建、在既有 provider 中添加模型、配置 OpenAI-compatible provider，以及实现自定义 streaming provider。判分同时检查生成配置/源码、reload 后运行时行为、网络探针、工具轨迹和最终回答，而不只比较文本。

比较报告按 eval set 汇总 baseline/candidate 的通过率百分点差，并把 token、延迟和估算费用作为独立的配对差值；`--repetitions` 或 `PI_EVAL_REPETITIONS` 控制比较套件的重复次数。当前覆盖仍窄：仓库尚未形成针对长会话压缩质量、overflow 恢复、危险工具拒绝、不同模型工具成功率、跨平台 shell 行为和故障注入的系统 eval 集。这是当前工程能力的缺口，不应因为已有 `packages/evals` 就宣称 evaluation 已完整解决。

### 9.1 具体实现

1. 读 [`pi-harness.ts`](../packages/evals/src/pi-harness.ts)：确认 eval 怎样创建隔离的正式 `AgentSession`，怎样提取轨迹、usage 和 session artifact。
2. 读 [`docs.eval.ts`](../packages/evals/src/docs.eval.ts)：确认文档页面怎样被逐页审计并用结构化工具结果判定。
3. 读 [`extensions.eval.ts`](../packages/evals/src/extensions.eval.ts)、[`models.eval.ts`](../packages/evals/src/models.eval.ts) 与 [`providers.eval.ts`](../packages/evals/src/providers.eval.ts)：确认三类定制怎样经过 reload、运行时探针和 judge 形成端到端评测。
4. 读 [`harness-table.ts`](../packages/evals/src/vitest-evals/harness-table.ts)：确认 baseline、candidate、repetition 和输入分组怎样建立可比较样本。
5. 读 [`reporter.ts`](../packages/evals/src/vitest-evals/reporter.ts) 与 [`summary.ts`](../packages/evals/src/vitest-evals/summary.ts)：确认通过率、token、延迟和费用怎样分别汇总。
6. 读 [`artifacts.ts`](../packages/evals/src/vitest-evals/artifacts.ts)：确认会话、报告与生成源码怎样在临时目录删除前持久化。

## 10. 安全：Pi 明确选择本地信任模型

Pi 的基本安全模型不是“Agent 默认被限制”，而是“Pi 与启动它的本地用户处于同一权限边界”。内建 `read`、`write`、`edit` 和 `bash` 可以使用该用户拥有的文件与进程权限；extension 也是同进程受信任代码。

### 10.1 Project trust 只保护启动输入

project trust 判断项目本地 settings、extensions、skills、prompts、themes 和 system prompt 文件能否进入运行时。判断按规范化目录保存，最近的父目录决定可以继承。未信任项目不能让自己的 extension 参与 trust 决策，避免待审批代码批准自己。

这个机制解决的是“进入目录时，仓库能否静默改变 Pi 配置或执行扩展”，不解决“开始工作后，模型能否提出危险 shell 命令”。`AGENTS.md` 等 context files 仍可能进入模型输入；仓库文本或工具输出还可能包含 prompt injection（诱导模型偏离用户意图的恶意指令），Pi 没有声称能在进程内可靠消除它。

### 10.2 工具边界提供策略插入点，不等于默认策略

在工具执行前，`AgentSession` 把 `tool_call` 交给 extension runner。扩展可以修改输入或返回 block；检查 hook 抛错时执行会被阻止，而不是绕过检查。仓库的 `permission-gate` 示例在危险 bash 命令前询问用户，非交互模式默认拒绝。

但这个 gate 是示例扩展，不是所有安装的默认权限系统。类似地，sandbox 示例通过替换 bash 执行环境接入操作系统级限制，也不是 Pi 核心默认启用的隔离。真正需要限制文件、网络或凭据时，应把整个 Pi 或工具执行放入容器、VM、micro-VM 或操作系统策略中。

还有一个较小但重要的执行保护：模型响应若因输出 token 限制而截断，Pi 不执行其中任何 tool call。即使残缺参数碰巧能被 JSON 修复并通过 schema，也不能证明它仍代表模型原本意图。

### 10.3 凭据和运行数据同样属于安全面

模型目录展示与认证执行分开。动态 API key、OAuth refresh 和命令型凭据尽量在真正请求时解析，避免仅打开模型列表就触发外部命令或暴露短期值。另一方面，session、eval artifact 和完整工具输出可能包含源码、prompt、路径或秘密；它们需要和工作区数据采用相同保护级别。

### 10.4 具体实现

1. 读 [`trust-manager.ts`](../packages/coding-agent/src/core/trust-manager.ts)：确认 trust 决策怎样规范化、继承、加锁并持久化。
2. Project trust 怎样影响资源加载，接着看[项目信任的具体实现](./04-resources-and-extensions.md#21-具体实现)。
3. 工具拦截和截断保护分别见[扩展具体实现](./04-resources-and-extensions.md#51-具体实现)中的 `_installAgentToolHooks()`，以及[工具循环](./05-agent-session-and-execution.md#35-具体实现)中的 `failToolCallsFromTruncatedMessage()`。
4. 对照 [`permission-gate.ts`](../packages/coding-agent/examples/extensions/permission-gate.ts) 与 [`sandbox/index.ts`](../packages/coding-agent/examples/extensions/sandbox/index.ts)：区分进程内策略确认和操作系统级隔离。
5. 读 [`resolve.ts`](../packages/ai/src/auth/resolve.ts)：确认存储凭据、环境凭据和 OAuth refresh 怎样在请求边界组合。

## 11. 失败恢复：先让状态可解释，再做有限重试

Pi 没有把所有失败统一成“再跑一次”。不同失败发生在不同边界：

- provider 或网络瞬时错误使用有限次数、指数退避且可中止的 retry；单次 agent-level 等待默认最多 60 秒，会话级错误分类会让额度、账单等明确非瞬时错误快速失败。
- context overflow 不进入普通 retry，而是移除失败 assistant 消息、压缩上下文、重建状态并只重试一次。
- 工具业务错误变成 `toolResult` error 回到模型，让模型决定如何修正；基础设施异常才结束 run。
- 用户 abort 会传播到 provider、退避等待和工具，并归一化为完整 `aborted` assistant message，避免留下无法解释的半条流式消息。
- 已完成的 user、assistant 和 tool result 在 `message_end` 时逐条追加，而不是等整个 run 成功才统一保存。

这套策略的共同点是保持明确终态和有限恢复。但正式 `AgentSession` 仍无法保证外部工具效果只发生一次：进程可能在命令已经修改文件、tool result 尚未落盘时崩溃。

`AgentHarness` 已实现更强的恢复边界。operation admission 原子记录 intent；Drive 在 provider/tool 效果前写 effect-pending 状态，在效果后把 frame、输出、usage 与下一状态一起提交；retry wait、cancel request 和 terminal result 也可恢复。安全重放的工具可以继续，不可重放工具在结果不确定时生成 synthetic interrupted result。这个设计把不确定性显式化，但仍不是 exactly-once：外部效果或模型计费可能已发生，单机事务不能撤销。

**请求级与会话级重试**

| 层次 | 重做什么 | 决策依据 | 等待边界 |
|---|---|---|---|
| `retryProviderRequest()` | 重新发起一次 SDK 请求 | provider error 的状态码与 headers | 支持 Retry-After、抖动和可取消等待 |
| `AgentSession._prepareRetry()` | 从现有上下文重新生成失败 assistant 响应 | `isRetryableAssistantError()`、产品 retry 设置 | 有限次数、指数退避、可取消等待 |
| overflow recovery | 压缩上下文后重新生成 | context overflow / 可恢复的 length stop | 一次恢复过程只做一次 compact-and-retry |
| 工具业务失败 | 不由通用工具执行器自动重做 | 转为 error tool result 交给模型 | 模型若再次调用，属于下一次决策 |

`retryProviderRequest()` 遵循 `x-should-retry`，否则把网络错误及 408、409、429、5xx 等视为可重试。它要求使用方关闭 SDK 自带重试，以便退避等待可响应取消；helper 的 `maxRetries` 默认是 0。没有服务端提示时采用带抖动的指数等待；服务端要求的等待超过默认 60 秒上限时直接抛错，而不是擅自提前重试。

会话级错误分类不同：先排除 quota、billing 等明确额度耗尽，再匹配瞬时网络或服务错误。不能因此声称“所有层都不会重试余额不足的 429”。产品普通重试默认开启，最多三次，基础等待两秒；`retryDelayMs()` 将单次等待限制为默认最多 60 秒，这个函数本身不加随机抖动。

两层同时启用时，实际请求次数会叠加。例如外层重试两次、每次内层重试一次，在失败条件持续满足时最多可能发起 `(2+1) × (1+1) = 6` 次请求。它们不是共享的全局请求或费用预算。

**重试保留历史，但移除失败上下文**

```text
user → assistant(error)
           ↓ message_end 保存到 session 历史
           ↓ auto_retry_start
           ↓ 从 Agent 当前上下文移除失败 assistant
           ↓ 可取消等待
       agent.continue() → 新 assistant
```

失败消息留在 session 中便于追踪，但不能把它当成完整回答继续拼接。重试也不是从已经显示的最后一个字符继续下载，而是重新生成本次 assistant 响应。之前完成的工具结果仍在上下文中，不由该流程整体重放。

`AgentSession` 在非 error 响应后重置 retry 计数，所以这个预算不限制整场会话总共能发生多少次网络失败。上下文溢出使用单独的 `_overflowRecoveryAttempted`，防止在同一恢复过程中不断“压缩—失败—再压缩”。

**重试和取消都不承诺外部效果只发生一次**

模型服务可能已处理并计费，但客户端没有收到结果。工具也可能已修改文件，随后在输出或存储阶段失败。再次调用可能重复副作用；现有通用 loop 没有为任意外部 API 提供幂等键或事务回滚。

实现入口：[provider-retry.ts](../packages/ai/src/utils/provider-retry.ts)、[retry.ts](../packages/ai/src/utils/retry.ts)、[agent-session.ts](../packages/coding-agent/src/core/agent-session.ts) 的 `_handlePostAgentRun()`、`_prepareRetry()`、`_checkCompaction()`，默认值见 [settings-manager.ts](../packages/coding-agent/src/core/settings-manager.ts) 的 `getRetrySettings()`。

### 11.1 具体实现

1. 读 [`retry.ts`](../packages/ai/src/utils/retry.ts)：确认错误分类、retry budget、`maxAgentDelayMs` 上限和中止归一化。
2. 普通 retry 与 overflow recovery 怎样接入 `AgentSession`，看[上下文压缩的具体实现](./06-session-and-persistence.md#36-具体实现)，再定位 `_prepareRetry()`。
3. Agent loop 怎样处理 error、abort 和 tool result，见[核心 loop](./05-agent-session-and-execution.md#35-具体实现)。
4. 消息何时落盘，可以在[一次请求的实现链](./05-agent-session-and-execution.md#13-具体实现)中跟踪 `_handleAgentEvent()`。
5. Harness 的 operation/recovery 实现与剩余切片见 [`AgentSession` 与 `AgentHarness`](./05-agent-session-and-execution.md#41-具体实现)。

### 11.2 取消、消息持久化与运行结束

`AgentSession.abort()` 取消 retry、compaction、branch summary 和 Agent，再等待 idle。模型/工具使用 run 的 AbortSignal；内建本地 shell 会在取消或超时时终止进程树，并在 `finally` 清理 timer 与监听器。单独的用户 bash 执行还有自己的 `abortBash()`，不能把它和普通 run 的 abort 当成同一个控制器。

取消是让未完成工作停止，不会撤销已经写入的文件、已发出的网络请求或已计费的模型计算。自定义工具若忽略 signal，等待 idle 仍可能很久。

正式消息处理的顺序是：Agent 更新状态 → AgentSession 等待 extension hook → 通知产品监听器 → `message_end` 时调用 SessionManager 追加消息。由此有两个实际边界：

- 收到产品侧 `message_end` 回调时，当前回调之后才执行默认 append；事件本身不是落盘事务的确认。
- `message_update` 不走这个 append 分支，故正式会话不能仅凭屏幕上显示过部分文字，就保证崩溃后能恢复这些文字。也不能把 append 调用等同于每条消息都 fsync。

`agent_end` 只表示底层这一轮 loop 结束。产品还可能退避重试、自动压缩或处理扩展新入队的消息；`AgentSession` 的 `agent_settled` 表示该轮产品后处理已走到结束边界。需要等待完整工作的调用方不应只监听第一次 `agent_end`。

`dispose()` 另外使旧 extension context 失效、取消关联工作、解除订阅、清理 session 级 provider 资源。这是 session replacement/reload 必须考虑的所有权问题；否则旧回调可能继续修改已经替换的会话。

实现入口：[agent-session.ts](../packages/coding-agent/src/core/agent-session.ts) 的 `_handleAgentEvent()`、`_runAgentPrompt()`、`abort()`、`dispose()`；持久化格式与实验性 frame 恢复见[会话与持久化](./06-session-and-persistence.md)。

## 12. 并发：允许工作并行，但固定状态提交顺序

Pi 同一 `Agent` 只允许一个 active run。用户在运行期间的新输入必须明确成为 steering 或 follow-up，避免两个 run 同时改写同一消息数组。单批工具可以并行执行，但 `ToolResultMessage` 按原始 tool-call 顺序进入 context；UI 进度仍按真实完成时间交错。文件工具再通过 mutation queue 串行化同一路径写入。

事件监听器和需要返回值的 extension hook 会被顺序等待。这降低并发度，却保证下一阶段开始前，产品状态、扩展决策和持久化处于确定顺序。Pi 的取舍不是最大吞吐，而是让模型下一次看到的上下文可以复现。

Harness 把并发边界提升到 lane：不同 lane 可共享 session 历史前缀并独立运行，但每条 lane 最多一个 operation；会话级 mutation line 再串行化跨 lane 的原子 commit。session worker 还用文件锁建立进程级所有权，server attachment 只授予 presentation-scoped 调用能力。这些层次分别解决“一个 lane 的状态机顺序”“一个 session 的写入顺序”和“一个进程拥有 durable 文件”，不能互相替代。

### 12.1 具体实现

并发相关代码分布在三个现有实现点：[核心 loop](./05-agent-session-and-execution.md#35-具体实现)中的 `PendingMessageQueue` 和 `executeToolCallsParallel()`，[内建工具](./05-agent-session-and-execution.md#61-具体实现)中的 `file-mutation-queue.ts`，以及[扩展实现](./04-resources-and-extensions.md#51-具体实现)中 `ExtensionRunner` 的顺序 handler 循环。

## 13. 可观测性、成本与性能

正式 coding-agent 当前最可靠的观测事实是 `AgentSessionEvent`、每条 assistant usage、工具进度、session JSONL 和 eval artifact。费用与 cache 统计直接从 provider usage 派生，不根据本地猜测宣称 cache hit。

`pi-telemetry` 还定义了 provider request、operation、turn、retry step、tool、hook、event handler 和 session write 等 span 契约。当前已落地的是显式 `Context` 传播、typed span helper，以及 `before_tool`/`after_tool` 的部分 hook span；大多数 Drive、event handler 和 session write span 仍未接线。正式 `AgentSession` 也没有自动产生这些 Harness span，跨进程 trace carrier 注入/提取尚未实现，不能把 schema 等同于完整分布式 trace。

另一个容易混淆的机制是 install telemetry：它是可配置的安装/版本报告，不等于上述 Agent 运行 trace。

性能优化分散在正确边界：provider prompt cache 复用稳定前缀；API 实现延迟加载减少启动成本；工具输出截断控制 context；TUI 差量渲染减少终端写入；并行工具降低独立调用的等待时间。这些优化都没有改变 agent loop 的基本语义。

排查问题时应区分 provider error、auto retry、compaction、tool execution 和最终 settled，使用 session ID、toolCallId、provider/model 关联事件。usage 与估算费用只覆盖获得并记录的数据，不能把断流时缺失的 usage 当成供应商没有计费。会话历史、导出和完整工具日志可能含敏感内容，凭据文件权限不会自动保护这些派生数据。

### 13.1 具体实现

1. usage 与 cache 指标的数据来源见 [prompt cache 的具体实现](./03-model-and-auth-runtime.md#91-具体实现)。
2. 读 [`telemetry.ts`](../packages/agent/src/harness/telemetry.ts)、[`hooks.ts`](../packages/agent/src/harness/hooks.ts)、[`telemetry.md`](../packages/agent/docs/telemetry.md) 与 [`telemetry-schema.md`](../packages/agent/docs/telemetry-schema.md)，区分已定义 schema、已接入 hook 和剩余 instrumentation。
3. eval 怎样记录 token、费用、延迟和原生 session，见本章的[评测实现顺序](#91-具体实现)。
4. 读 coding-agent 的 [`telemetry.ts`](../packages/coding-agent/src/core/telemetry.ts)，确认 install telemetry 与 Agent trace 是两件事。

## 14. Pi 的整体设计哲学

把这些机制放在一起，Pi 的设计取向可以概括为源码中反复出现的四个选择：

1. **小而通用的控制循环**：`Agent`/agent loop 只处理模型、工具、队列和事件；正式产品策略留给 `AgentSession`。
2. **保存事实，派生模型视图**：session 保留完整历史树，context、compaction 和 branch summary 是面向下一次模型请求的投影。
3. **在明确边界开放定制**：extension 通过声明过的 hook、tool、provider 和 UI API 改变行为，而不是要求通用 loop 理解每种产品需求。
4. **不伪装安全与可靠性保证**：本地工具默认拥有用户权限，真正隔离交给 OS；正式产品只做有限 retry 和消息级持久化；Harness 提供 durable recovery，但明确不把它宣传成外部效果 exactly-once。

这四点也是阅读 Pi 时最值得追踪的主线：每遇到一个机制，先判断它属于通用控制、产品策略、模型适配、持久事实，还是扩展边界。这里使用的是仓库已有模块边界，不是额外创造一套架构层次。
