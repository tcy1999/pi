# Session Search 与遥测边界

来源：packages/agent/src/search/index.ts、packages/agent/docs/telemetry.md 和生成的 telemetry-schema.md。原 search.md 与旧 scanning/SQLite FTS 实现已从当前源码删除。

## 1. Search 当前只是 draft interface

当前 agent-core 暴露 SessionSearchService：

- searchSessions(query) 返回 session 级结果，可带最高命中 entry。
- 可选 searchEntries(query) 返回 entry 级结果。
- sync()/notify()/remove()/close() 描述索引生命周期。

结果是 Promise 数组，不再是旧文档中的 AsyncIterable；当前仓库也没有 scanning search 或 SQLite FTS backend。score、snippet 和 timestamp 仍是可选/后端字段，但不能把过去的索引一致性策略写成当前实现。

这个接口表达的稳定边界很窄：sessionId/entryId 是权威对象身份，索引是可重建派生状态。实际 tokenizer、ranking、增量 feed、只读加载和远程索引策略仍需未来实现决定。

## 2. 遥测按稳定语义分层

telemetry schema 将一次逻辑 provider 请求，以及 Harness 的 run、compaction、navigation、checkpoint、turn、retry step、tool、hook、sleep、event handler 和 session write 分成不同 span。schema 和 typed starter 已实现，但不能据此声称真实 run 已产生全部 span：当前主要落地点是 tool hook，Drive、event handler、session write 与跨进程 parentage 大多仍是实施工作。

父子关系表达结构：operation span 包含 turn/checkpoint，turn 或结构操作包含 retryable step，step 再关联 provider/tool 效果。这种拆分可以区分：

- provider 网络时间与 retry backoff。
- 工具执行与事件 handler。
- 普通 run、compaction 与 navigation。
- 首次执行与 recovery。
- operation terminal outcome 与单次 attempt outcome。

## 3. Context 与跨进程限制

每个异步 Harness/Lane/Session/Storage 调用显式接收 Context，span 从调用 Context 取得 parent，取消从 context.abortSignal 传播。共享对象不保存调用方 Context，也不依赖 AsyncLocalStorage。

本地 Context 和 typed telemetry 基础已实现，但完整 runtime instrumentation 尚未实现；跨进程 trace carrier 的注入、提取和远端 parent 重建也仍是设计项。因此 client/server request cancellation 已能工作，不代表一条 trace 已自动跨进程连续。

## 4. 高基数字段与内容边界

session ID、lane name、operation ID、turn ID、tool call ID 和 provider response ID 属于高基数属性；status、error type、operation kind 属于低基数属性。schema 标记这些差异，使 exporter 能选择采样或索引策略。

遥测记录结构、耗时、usage、结果分类和稳定 error code，不默认复制 prompt、工具参数或模型输出。对话内容属于 session、event 和 Transcript 数据，不应为了排障自动进入 telemetry。

telemetry-schema.md 由源码生成，中文架构文档只解释 span 分层和边界，不复制频繁变化的字段全集。

## 5. 正式产品与 Harness telemetry 不是一回事

正式 AgentSession 最可靠的观测仍是 AgentSessionEvent、assistant usage、tool progress 和产品 JSONL。coding-agent 的 install telemetry 是安装/版本报告，也不是 Harness trace。

实验性 session worker 使用 AgentHarness，但这只保证调用链携带 Context；在相应 span 调用点尚未接线前，不会仅因为使用 Harness 就产生完整 trace。是否导出、哪些操作已 instrument，以及怎样跨进程关联仍取决于实现和宿主 TelemetryContext。

## 6. 具体实现

1. 读 [search/index.ts](../../packages/agent/src/search/index.ts)，确认当前仅有的 draft search contract。
2. 读 [telemetry/src/index.ts](../../packages/telemetry/src/index.ts) 与 [memory.ts](../../packages/telemetry/src/memory.ts)，确认通用 span 接口与内存实现。
3. 读 [harness/telemetry.ts](../../packages/agent/src/harness/telemetry.ts)，确认 Harness 语义怎样映射到 span。
4. 读 [hooks.ts](../../packages/agent/src/harness/hooks.ts)，确认当前 tool hook span 的实际接入；再读 [telemetry.md](../../packages/agent/docs/telemetry.md) 的 status，核对仍未完成的传播与 instrumentation。
5. 字段全集最后对照生成的 [telemetry-schema.md](../../packages/agent/docs/telemetry-schema.md)。
