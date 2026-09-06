# Session Search 与遥测边界

来源：`packages/agent/docs/search.md` 和生成的 `telemetry-schema.md`。排除 Elasticsearch 示例代码与逐字段 schema 表。

## 1. Search 只定义最小公共身份

公共 `SessionSearch` 返回至少 `(sessionId, entryId)`。snippet、score、timestamp、offset 和 ranking 都属于具体 backend，因为扫描、SQLite FTS 和远程索引不可能共享完全相同的排序语义。

结果使用 `AsyncIterable`，使调用方可以边搜边显示、达到数量后停止，并通过 `AbortSignal` 取消上一轮 search-as-you-type。debounce 属于 UI，不进入搜索接口。

## 2. 两种已实现搜索方式

- scanning search：把 session/storage 投影为可搜索文本并顺序匹配，适用于已打开 session 或只读加载的 JSONL。
- SQLite FTS：首次非空搜索时建立 FTS 表并从 canonical entries rebuild，之后由 trigger 与事务写入保持同步。

扫描 JSONL 时不能为了读取搜索内容调用可能获取 writer lease 的 `SessionRepo.open()`；搜索应使用只读加载或已有 storage。

## 3. 索引是派生状态

entry/session 是权威数据，搜索索引可以重建。外部 Elasticsearch 等索引需要应用自己实现 feed 和 catch-up；agent-core 只规定查询形状，不把特定索引产品写进 session 核心。

SQLite FTS 有一个不同取舍：索引 trigger 与 canonical write 共处一个事务，因此 FTS 失败可能回滚主写入。这换来了提交后立即一致，而不是最终一致。

## 4. 遥测按稳定语义分层

telemetry schema 已将一次逻辑 provider 请求，以及目标 Harness 中的 operation、turn、retry step、tool、hook、sleep、event handler 和 session write 分成不同 span。父子关系表达结构：run 包含 turn/checkpoint，turn 或结构操作包含 retryable step，step 再关联 provider/tool 效果。由于完整 `AgentHarness` 调度尚未实现，不能据此声称所有 Harness span 已在真实 run 中产生；这里解释的是已定义的观测契约。

这种拆分避免把所有耗时都塞进一个“Agent run”span，也能区分：

- provider 网络慢，还是 retry backoff。
- 工具执行慢，还是事件 handler 慢。
- 正常 run，还是 compaction/navigation。
- 首次执行，还是 recovery。

## 5. 高基数字段与内容边界

session ID、operation ID、tool call ID 和 provider response ID 属于高基数属性；状态、error type、operation kind 属于低基数属性。schema 明确标记它们，使 exporter 能选择采样或索引策略。

遥测应记录结构、耗时、usage、结果分类和稳定 error code，而不是默认复制 prompt、工具参数或模型输出。对话内容属于 session 与事件数据，不应因为可观测性方便就自动进入 telemetry。

`telemetry-schema.md` 由源码生成，字段表是机器同步的参考；中文架构文档只解释 span 分层和边界，不复制可能频繁变化的字段全集。

## 6. 具体实现

1. 读 [`scanning.ts`](../../packages/agent/src/search/scanning.ts)，确认最小搜索接口怎样投影 session 内容。
2. 读 [`search-backend.ts`](../../packages/session-backends/sqlite-node/src/sqlite/search-backend.ts)，确认 FTS 索引、重建和事务一致性。
3. 读 [`telemetry/src/index.ts`](../../packages/telemetry/src/index.ts) 与 [`memory.ts`](../../packages/telemetry/src/memory.ts)，确认通用 span 接口与内存实现。
4. 读 [`harness/telemetry.ts`](../../packages/agent/src/harness/telemetry.ts)，确认 Harness 语义怎样映射到 span；字段全集最后对照 [`telemetry-schema.md`](../../packages/agent/docs/telemetry-schema.md)。
