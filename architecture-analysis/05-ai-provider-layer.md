# 模型、供应商与缓存

provider（模型供应商适配）是描述一组模型、认证方式、模型目录和请求实现的对象。它把 Anthropic、OpenAI、Google 等不同接口转换成 Pi 统一的消息和事件。

## 1. 统一对象模型

`pi-ai` 用以下核心类型隔离供应商差异：

- `Provider`：ID、名称、base URL、auth 方法、模型目录和流实现。
- `Model<Api>`：模型能力、输入类型、context window、max tokens、价格、reasoning 和 compat。
- `Context`：system prompt、统一 messages 和 tools。
- `AssistantMessageEvent`：start、text/thinking/tool-call delta、done/error。
- `ProviderStreams`：`stream`、简化的 `streamSimple`，以及可选 deferred fetch/cancel。

调用方通常使用 `Models.streamSimple()`；它负责鉴权、默认参数和 thinking level 映射。需要供应商高级能力时仍可用 typed API-specific options，避免最低公分母接口。

## 2. Provider 注册而非全局分支

以 OpenAI 为例，`openaiProvider()` 组合模型表、环境变量鉴权和 `openAIResponsesApi()`。其他 provider 使用同一工厂。`builtinModels()` 只是注册所有 provider 的便利函数；关注 bundle size 的应用可只导入单个 provider。

这比在一个巨大 `switch(provider)` 中处理所有请求更易扩展，也支持 extensions 注册自定义供应商。

## 3. 流适配

供应商适配器执行四步：

1. 将统一 context/messages/tools 转成供应商 payload。
2. 发出请求并消费原生 SSE/WebSocket/SDK 流。
3. 增量更新统一 `AssistantMessage`，发出规范化事件。
4. 计算 usage/cost，给出明确 stop reason。

`AssistantMessageEventStream` 同时是 async iterable 和 final-result promise。事件消费者获得低延迟更新，业务代码又能 `await stream.result()` 获取最终消息。

### 3.1 具体实现

1. 读 [`types.ts`](../packages/ai/src/types.ts) 的 `Model`、`Context` 与 `AssistantMessageEvent`，确认统一层承诺的数据形状。
2. 读 [`models.ts`](../packages/ai/src/models.ts) 的 `Models.streamSimple()`，确认模型选择、认证和公共 options 怎样进入请求。
3. 读 [`anthropic.ts`](../packages/ai/src/providers/anthropic.ts)，确认 provider 怎样组合模型目录、认证和 API implementation。
4. 读 [`anthropic-messages.ts`](../packages/ai/src/api/anthropic-messages.ts)，确认统一消息怎样变成网络 payload，原生流又怎样变回统一事件。

## 4. 延迟加载

API SDK 较重且供应商很多。`*.lazy.ts` 通过 `lazyApi()` 在首次请求时加载具体实现，但立即同步返回外层 event stream；加载失败也转换为规范化 error event。这保持 stream API 同步可用，又让 tree shaking 和 CLI 启动成本可控。

## 5. 兼容策略集中化

“OpenAI-compatible”并不代表行为一致。模型的 `compat` 描述 developer role、strict tools、prompt cache、session affinity、deferred tools 等能力。适配器按 capability 构建 payload，而不是按品牌猜测。

例如 OpenAI Responses adapter 会：

- 对最小输出 token、cache retention 和 session headers 做约束。
- 将 deferred tools 映射为 additional tools 或 tool search。
- 按 provider/base URL 选择 affinity header 格式。
- 清除流解析期间的 `partialJson` 等临时字段，禁止持久化。

## 6. 鉴权与模型目录

`ModelRuntime` 位于 coding-agent 产品层，组合 `pi-ai` provider auth、凭据存储和自定义 `models.json`。优先级为运行时覆盖、持久凭据、环境变量、fallback resolver。

模型目录既有生成的静态快照，也支持远端刷新和本地 store。刷新按 provider generation 隔离：新一代刷新不等待旧的卡住请求，旧结果也不能覆盖新结果。离线模式使用已缓存目录。生成文件 `models.generated.ts` 来自脚本，不应手改。

## 7. 成本与 thinking

统一 usage 区分 input、output、cache read/write 和可选 reasoning。成本由模型价格表计算，支持输入量阶梯和 Anthropic 长时缓存等特殊定价。

thinking level 使用跨供应商的 `off/minimal/low/medium/high/xhigh/max`，模型可通过 map 禁用或映射等级；`clampThinkingLevel()` 在请求能力不存在时选择最近可用等级。

## 8. 先区分三种 cache

cache（缓存）只表示“保存某些数据供后续复用”。Pi 中至少有三种不同机制：

| 名称 | 保存在哪里 | 复用什么 | 主要目的 |
|---|---|---|---|
| provider prompt cache | 模型供应商侧 | 多次请求相同的 prompt 前缀 | 降低输入成本和延迟 |
| model catalog cache | 本地 `models-store.json` | 远程获取的模型目录 | 离线启动和减少目录刷新 |
| process cache | 当前 Pi 进程内 | Git 分支、渲染结果或连接等派生数据 | 避免重复计算或重连 |

它们没有统一失效策略，也不能互相替代。讨论 Agent 工程中的 cache 时，通常首先指 provider prompt cache。

## 9. Prompt cache 怎样工作

Pi 每个 turn 仍会构造并发送完整逻辑 context；它不会把旧消息从请求中删掉并假设供应商记得。API adapter 根据供应商能力添加 cache marker、`prompt_cache_key`、retention 或 session affinity 字段。供应商识别与前一次相同的前缀后，usage 才会报告 `cacheRead`；新写入的可缓存前缀报告为 `cacheWrite`。

`cacheRetention` 表示缓存保留偏好：`none` 禁用，`short` 是默认的供应商常规保留，`long` 只在模型兼容信息声明支持时映射到较长 TTL（生存时间）。不同 provider 的网络字段不同，所以策略在统一 option 中表达，具体编码留在各 API adapter。

稳定的 session ID 有两个相关但不同的用途：它可以作为 prompt cache key，也可能作为供应商的请求路由或连接亲和标识。不能看到 `sessionId` 就断言一定发生了 cache hit；真实命中只以供应商返回的 usage 为准。

compaction 会把大量旧消息替换为一条新摘要，改变 prompt 前缀，因此紧接压缩后的请求可能减少 cache hit。这是“缩短 context”与“保持相同前缀供缓存复用”之间的真实取舍。压缩自己的总结请求明确使用 `cacheRetention: "none"`，避免为一次性总结写缓存。

`coding-agent/src/core/cache-stats.ts` 比较相邻 assistant usage，估算 cache miss 带来的额外费用；footer 只是展示累计 `cacheRead/cacheWrite` 和最近命中率，不负责缓存本身。

### 9.1 具体实现

缓存仍沿用 3.1 的公共类型和 provider adapter，这次关注 cache 相关字段：

1. 在 `types.ts` 中定位 `cacheRetention` 与 usage 字段，确认统一层表达什么、不表达什么。
2. 在 `anthropic-messages.ts` 中定位 cache marker，再与 [`openai-responses.ts`](../packages/ai/src/api/openai-responses.ts) 对照 key 与 retention 的不同映射。
3. 读 [`cache-stats.ts`](../packages/coding-agent/src/core/cache-stats.ts)，确认产品怎样依据供应商 usage 估算命中率和额外费用。
4. 在[压缩算法](./04-session-and-persistence.md#36-具体实现)的 `completeSummarization()` 中，可以看到一次性摘要请求为何禁用 prompt cache。

## 10. 评价

该层成功把供应商复杂度封装在边界内，同时保留非通用能力。主要维护成本是兼容矩阵和快速变化的模型元数据，因此仓库用生成脚本、目录校验和大量 provider regression tests 将其变成数据与测试问题，而不是散落在产品层的条件分支。
