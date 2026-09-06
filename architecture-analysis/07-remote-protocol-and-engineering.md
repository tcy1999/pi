# 跨进程协议、测试与工程治理

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
| 行为评测 | pi-evals + vitest-evals | 用真实模型比较 prompt、tool 和 skill 效果 |
| 发布 smoke | 仓库外隔离安装 | 防止 workspace 解析掩盖打包问题 |

根 test.sh 清理相关环境变量并使用临时用户目录，避免本机凭据和配置使测试误调用真实 API。

## 7. 静态质量门与发布

npm run check 执行格式与静态检查、精确依赖、runtime dependency、TS import 与 entry graph 检查、shrinkwrap/install-lock 校验、类型检查和 browser smoke。它不等于测试通过。

项目将供应链作为安全边界：直接外部依赖使用精确版本；CI 以 --ignore-scripts 安装；coding-agent shrinkwrap 固定发布侧传递依赖；带 lifecycle script 的新依赖必须显式审核。Chord facet bundler 只构建已有 package，不安装依赖或执行 package lifecycle script。

release script 同步所有包版本、生成产物、运行检查、创建提交和 tag；tag CI 构建并发布 npm，再验证 tarball 可用后更新公开版本标记。发布 helper 是幂等的，失败时应修复并重跑 job，而不是为同一版本重新执行 release。

### 7.1 具体实现

1. [package.json](../package.json)：确认 check、test、eval 与 release 是不同入口。
2. [test.sh](../test.sh)：确认普通测试怎样隔离本机认证和配置。
3. session 后端怎样共享行为契约，见 [durable session 后端](./04-session-and-persistence.md#71-具体实现)。
4. [faux.ts](../packages/ai/src/providers/faux.ts)：确认无需真实模型的 provider 场景怎样构造。
5. [build-binaries.yml](../.github/workflows/build-binaries.yml)：确认构建、发布和发布后验证的边界。
