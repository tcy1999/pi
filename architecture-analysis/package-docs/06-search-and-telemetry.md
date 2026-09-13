# 会话搜索与遥测边界

来源：`packages/agent/src/search/index.ts`、`packages/agent/docs/telemetry.md` 和生成的 `telemetry-schema.md`。搜索部分以当前接口为准，不沿用已删除的扫描搜索和 SQLite 全文搜索实现。

## 1. 会话搜索的接口草案

当前 agent-core 暴露 `SessionSearchService` 接口，尚无搜索后端实现：

- `searchSessions(query)` 返回会话级结果，可附带一条最佳命中记录。
- 可选的 `searchEntries(query)` 返回历史记录级结果。
- `sync()` / `notify()` / `remove()` / `close()` 描述索引生命周期。

查询返回解析为结果数组的 Promise。`score` 和 `snippet` 可选；entry 级结果的 `timestamp` 必填，session 级结果若提供 `top` 命中，也必须包含该命中的时间戳。当前没有扫描搜索或 SQLite 全文搜索后端，接口本身不保证索引一致性。

结果通过 `sessionId` / `entryId` 引用会话和历史节点。分词、排序、增量索引、只读加载和远程索引策略仍待具体后端实现。

## 2. 遥测按稳定语义分层

遥测 schema（数据契约）将一次逻辑 provider 请求，以及 Harness 的 run、compaction、navigation、checkpoint、turn、retry step、tool、hook、sleep、event handler 和 session write 分成不同 span（一次操作的追踪记录）。schema 和 typed starter 已实现，但不能据此声称真实 run 已产生全部 span：当前主要落地点是 tool hook，Drive、event handler、session write 与跨进程 parentage 大多仍是实施工作。

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

session ID、lane name、operation ID、turn ID、tool call ID 和 provider response ID 属于高基数属性，即可能出现大量不同取值；status、error type、operation kind 的取值种类较少，属于低基数属性。schema 标记这些差异，使 exporter 能选择采样或索引策略。

遥测记录结构、耗时、usage、结果分类和稳定 error code，不默认复制 prompt、工具参数或模型输出。对话内容属于 session、event 和 Transcript 数据，不应为了排障自动进入 telemetry。

telemetry-schema.md 由源码生成，中文架构文档只解释 span 分层和边界，不复制频繁变化的字段全集。

## 5. 正式产品与 Harness telemetry 不是一回事

正式 AgentSession 最可靠的观测仍是 AgentSessionEvent、assistant usage、tool progress 和产品 JSONL。coding-agent 的 install telemetry 是安装/版本报告，也不是 Harness trace。

实验性 session worker 使用 AgentHarness，但这只保证调用链携带 Context；在相应 span 调用点尚未接入前，不会仅因为使用 Harness 就产生完整 trace。是否导出、哪些操作已记录 span，以及怎样跨进程关联仍取决于实现和宿主 TelemetryContext。

## 6. 具体实现

1. 读 [search/index.ts](../../packages/agent/src/search/index.ts)，确认当前搜索接口草案。
2. 读 [telemetry/src/index.ts](../../packages/telemetry/src/index.ts) 与 [memory.ts](../../packages/telemetry/src/memory.ts)，确认通用 span 接口与内存实现。
3. 读 [harness/telemetry.ts](../../packages/agent/src/harness/telemetry.ts)，确认 Harness 语义怎样映射到 span。
4. 读 [hooks.ts](../../packages/agent/src/harness/hooks.ts)，确认当前 tool hook span 的实际接入；再读 [telemetry.md](../../packages/agent/docs/telemetry.md) 的 status，核对仍未完成的传播与 instrumentation。
5. 字段全集最后对照生成的 [telemetry-schema.md](../../packages/agent/docs/telemetry-schema.md)。
