# 模型、认证与供应商运行时

provider（模型供应商适配）是描述一组模型、认证方式、模型目录和请求实现的对象。它把 Anthropic、OpenAI、Google 等不同接口转换成 Pi 统一的消息和事件。

## 1. 统一对象模型

`pi-ai` 用以下核心类型隔离供应商差异：

- `Provider`：ID、名称、base URL、auth 方法、模型目录和流实现。
- `Model<Api>`：模型能力、输入类型、context window、max tokens、价格、reasoning 和 compat。
- `Context`：system prompt、统一 messages 和 tools。
- `AssistantMessageEvent`：start、text/thinking/tool-call delta、done/error。
- `ProviderStreams`：`stream`、简化的 `streamSimple`，以及可选 deferred fetch/cancel。
- `AssistantMessageFrame`：可独立持久化和归约的流式帧，供 durable Harness 在重启后恢复部分 assistant 输出。
- `ImageProvider`/`ImageModel`：独立于对话模型的图片生成目录和请求接口。

调用方通常使用 `Models.streamSimple()`；它负责鉴权、默认参数和 thinking level（推理强度）映射。需要供应商专有能力时，可以使用对应 API 的类型化选项。

## 2. 注册供应商实现，并按模型所属供应商调用

不同供应商使用不同的认证方式和请求接口。Pi 将这些实现与模型目录组合成 `Provider` 对象，通过 `models.setProvider(provider)` 按 `provider.id` 保存到当前 `Models` 实例中。发起请求时，`Models.streamSimple()` 根据 `model.provider` 查找对应对象，解析凭据并合并请求参数，再调用该对象的 `streamSimple()`。

以 OpenAI 为例，`openaiProvider()` 调用 `createProvider()`，组合内建模型表、支持读取 `OPENAI_API_KEY` 的认证方法和 `openAIResponsesApi()` 请求实现。注册后，`model.provider` 为 `"openai"` 的请求就会交给这个对象，最终由 OpenAI Responses 适配器发送请求并转换响应事件。

接入新供应商时，可以实现并注册新的 `Provider`，公共请求入口继续按相同流程查找和调用，无需增加针对该供应商的 `if` 或 `switch` 分支。coding-agent 的扩展也支持注册自定义供应商，具体配置由 `ModelRuntime` 组合后加入模型集合。

`builtinModels()` 创建一个 `Models` 实例，并注册 `builtinProviders()` 返回的全部内建供应商。只需要某个供应商的应用，也可以单独导入其工厂函数，再将返回的对象注册到自己创建的集合中，以减少打包时引入的模块。

实现入口：[models.ts](../packages/ai/src/models.ts) 的 `setProvider()`、`streamSimple()` 和 `createProvider()`；OpenAI 的组合见 [openai.ts](../packages/ai/src/providers/openai.ts)，内建供应商的批量注册见 [all.ts](../packages/ai/src/providers/all.ts)。

### 2.1 ModelRuntime 创建时做了什么

`ModelRuntime.create()` 为 coding-agent 初始化模型和认证相关的运行时对象：读取模型配置、准备凭据与模型目录存储、注册供应商，并默认执行一次刷新，更新模型列表和认证状态。后续列出模型、登录和发起请求都通过这个运行时完成。

```text
ModelRuntime.create(options)
  → 凭据存储 + 临时 key 覆盖层
  → models.json 配置快照 + 模型目录存储
  → 内建 provider + 远程目录能力
  → createModels() + provider 组合
  → 默认执行 refresh：先恢复缓存，可选联网，再计算可用性
  → 返回可列模型、管理认证和发起请求的 ModelRuntime
```

具体按以下顺序执行：

1. **准备凭据。** 使用调用方注入的 `CredentialStore`，否则创建默认 `AuthStorage`；外面统一包一层 `RuntimeCredentials`，让临时 API key 可以覆盖持久凭据。创建存储会读取本地认证状态，但不等于发起登录。
2. **读取模型配置。** 默认读取 agent 目录的 `models.json`；`modelsPath: null` 表示不加载该文件。`ModelConfig.load()` 去掉 BOM 和 JSON 注释、校验 schema，再创建不可变的 provider 配置快照。文件不存在按空配置处理；格式或读取错误保存在配置对象中，可通过 runtime 的 `getError()` 查询。
3. **准备目录存储。** 优先使用注入的 `ModelsStore`；否则默认使用与 `models.json` 同目录的 `models-store.json`，路径可覆盖。关闭模型配置文件且未注入 store 时，改用内存目录存储。这个存储保存模型元数据，不保存 API key。
4. **构造 provider。** 调用 `builtinProviders()` 得到内建 provider；除 Radius 外，通过 `withRemoteCatalog()` 为它们增加恢复缓存和刷新远程目录的功能。这一步不会发送网络请求。
5. **创建集合并组合配置。** 构造函数调用 pi-ai 的 `createModels({ credentials, modelsStore })`，保存内建 provider，再通过 `rebuildProviders()` / `recomposeProvider()` 应用配置。随后 `configureRadiusProviders()` 根据 `oauth: "radius"` 与网关地址建立相应 provider，再重建集合。扩展 provider 的注册表此时为空；加载扩展、调用 `registerProvider()` 属于后续资源装配，不是 `create()` 自己发现并执行扩展。
6. **初始化目录和认证快照。** 除非 `refreshOnCreate: false`，创建流程会等待 `runtime.refresh()`。它重新加载配置并组合 provider，交给 `Models.refresh()` 先恢复本地目录，再在允许时联网，最后更新 `all`、`available`、已配置 provider、已存储凭据 provider 和认证来源等快照。

**创建选项与实际行为**

| 条件 | 创建时的行为 |
|---|---|
| 默认选项 | 恢复本地目录、检查认证配置并计算可用模型；不联网刷新目录 |
| `allowModelNetwork: true` 且未设置 `PI_OFFLINE` | 允许初始化阶段联网刷新目录；具体是否请求仍由 provider 的认证与新鲜度策略决定 |
| 设置了 `PI_OFFLINE` | 创建时不启用目录网络刷新；这是目录刷新策略，不是进程级网络沙箱 |
| `refreshOnCreate: false` | 跳过初始刷新；同步模型列表仍有已组合的模型，但可用模型和认证快照尚未初始化 |
| 提供 `modelRefreshTimeoutMs` 且启用创建时联网刷新 | 对初始刷新阶段设置取消 timer，并与调用方 signal 合并；不是整个构造过程的统一超时 |

完整模型列表与可用模型列表不同。例如，内建目录中包含 OpenAI 的模型，但用户既没有保存 API key，也没有设置相应环境变量，这些模型仍会出现在完整列表中，却不会进入可用列表。检查模型是否可用时，主要看认证配置是否完整，以及模型是否符合 provider 的过滤条件。Pi 不会逐个调用模型来确认权限、余额或服务是否正常。

`ModelRuntime.create()` 不负责选择本次对话的模型或创建 AgentSession。实际调用模型时，还会重新读取和解析凭据，见本章第 6 节。

部分初始化步骤失败时，仍可能返回 `ModelRuntime`。例如，应用 provider 配置失败时，会记录错误，并在有内建 provider 的情况下保留它；刷新模型目录失败时，会按 provider 收集错误。`create()` 会等待 `refresh()` 完成，但不会因为返回结果中有刷新错误就抛出异常。因此，创建成功并不代表所有模型目录都刷新成功。可以通过 `getError()` 查看配置、provider 组合和可用性检查中的错误；主动调用 `refresh()` 时，还应检查返回的 `errors` 和 `aborted`。

实现入口：[model-runtime.ts](../packages/coding-agent/src/core/model-runtime.ts) 的 `create()`、构造函数、`recomposeProvider()`、`refresh()` 与 `runAvailabilityRefresh()`；配置读取见 [model-config.ts](../packages/coding-agent/src/core/model-config.ts)，目录存储见 [models-store.ts](../packages/coding-agent/src/core/models-store.ts)。

## 3. 流适配

供应商适配器执行四步：

1. 将统一 context/messages/tools 转成供应商 payload。
2. 发出请求并消费原生 SSE/WebSocket/SDK 流。
3. 增量更新统一 `AssistantMessage`，发出规范化事件。
4. 计算 usage/cost，给出明确 stop reason。

`AssistantMessageEventStream` 同时是 async iterable 和 final-result promise。事件消费者获得低延迟更新，业务代码又能 `await stream.result()` 获取最终消息。

统一 stream event 适合同进程观察，但不适合直接充当 durable log：事件中的 partial message 会重复大量前缀，重启也需要严格验证顺序。`AssistantMessageFrameEncoder` 将其转为 start、内容增量、usage、done/error 等紧凑帧；`reduceAssistantMessageFrames()` 可从已提交帧重建 pending 或 settled assistant message。Harness 为每次 provider attempt 使用独立 frame list，完成后再把最终 message 放入 entry 树。

**网络块、协议事件、消息块是三种粒度**

一次网络读取可能只有半个 JSON，也可能包含多个事件。不能把每个 `reader.read()` 的结果直接当作一个完整消息。

可选的 `streamProxy()` 提供了一个具体例子：它用流式 `TextDecoder` 处理跨网络块的 UTF-8 字符，累积 buffer，按换行取出完整的 `data: ` 行，保留最后的残行。EOF 时再刷新 decoder，并解析可能没有换行结尾的最后一条事件。这是该代理约定的逐行解析器，不是完整通用 SSE 规范解析器。

协议事件随后更新同一条 assistant 的多个 content block：

```text
start
  text_start(index=0) → text_delta("正在") → text_delta("检查") → text_end
  toolcall_start(index=1) → toolcall_delta(参数片段) → toolcall_end
done(reason=toolUse)
```

`contentIndex` 区分正文、thinking 和工具调用；`toolCallId` 关联后续工具结果。`Agent` 把这些事件转换成 `message_start/update/end`，UI 无须直接处理供应商 wire format。

**不完整参数可显示，不代表可执行**

工具参数可能依次收到 `{"path":`、`"src/`、`index.ts"}`。`parseStreamingJson()` 先尝试完整 JSON 与有限修复，再尝试 partial-json，失败时返回空对象。这允许界面展示正在形成的参数。

正式 loop 会等 assistant 响应完成后才执行工具，并经过 `prepareArguments`、schema 校验和 `beforeToolCall`。如果 stop reason 是 `length`，整条响应中的工具调用都生成失败结果，不执行任何一个，即使某些参数碰巧能被补全并通过 schema。原因是语法合法不能证明被截断的参数表达了完整意图。

**断流必须变成失败，不能当作成功**

`AssistantMessageEventStream` 同时提供异步迭代和 `result()` promise。`done/error` 决定最终消息；error 也是一个带 stop reason 的 `AssistantMessage`，并不意味着 `result()` 一定 reject。消费者必须检查消息终态。

`streamProxy()` 明确检查是否收到终态事件。即使 TCP/HTTP 正常 EOF，只要还没有 `done/error`，就生成连接提前关闭的 error，避免用户只看到半句话而调用方永远等不到最终结果。用户取消则归为 `aborted`。

`lazyStream()` 把异步认证、模块加载或转发抛错转换为 error 事件，让同步返回 stream 的接口保持一致。自定义 provider 仍须遵守终态契约：单独调用 `end()` 而既不传 result、也不发终态事件，虽然可以结束迭代，却不会自动完成 `result()`。底层通用 stream 不会替失约 provider 推断答案或设置超时。

**顺序消费不等于完整背压**

背压指消费者处理不过来时，让生产者减速或暂停。当前 `EventStream.push()` 是同步操作：有等待者就交付，没有就进入 FIFO 队列；没有队列容量上限，也没有等待消费者腾出空间的协议。

`Agent` 顺序 await 其订阅者，所以慢扩展会拖慢事件消费，但不保证上游 provider 同步停读网络。大量 delta 可能在中间积累。该流也不是广播总线：多个迭代器共享一个队列，不能假设每个迭代器都会收到全部事件。

产品侧 `AgentSession.subscribe()` 的回调签名为 `void`，内部同步调用，不 await 回调返回的异步任务。它与 `Agent.subscribe()` 的等待语义不同。若异步消费者需要可靠完成边界，应自行管理队列和等待，而不是依赖一个未被等待的 promise。

实现入口：[proxy.ts](../packages/agent/src/proxy.ts)、[event-stream.ts](../packages/ai/src/utils/event-stream.ts)、[json-parse.ts](../packages/ai/src/utils/json-parse.ts)、[lazy.ts](../packages/ai/src/api/lazy.ts)、[agent-loop.ts](../packages/agent/src/agent-loop.ts)、[agent.ts](../packages/agent/src/agent.ts)。

### 3.1 具体实现

1. 读 [`types.ts`](../packages/ai/src/types.ts) 的 `Model`、`Context` 与 `AssistantMessageEvent`，确认统一层承诺的数据形状。
2. 读 [`models.ts`](../packages/ai/src/models.ts) 的 `Models.streamSimple()`，确认模型选择、认证和公共 options 怎样进入请求。
3. 读 [`anthropic.ts`](../packages/ai/src/providers/anthropic.ts)，确认 provider 怎样组合模型目录、认证和 API implementation。
4. 读 [`anthropic-messages.ts`](../packages/ai/src/api/anthropic-messages.ts)，确认统一消息怎样变成网络 payload，原生流又怎样变回统一事件。
5. 读 [`assistant-message-frame.ts`](../packages/ai/src/utils/assistant-message-frame.ts)，确认临时流事件如何编码为可持久化的帧，再还原成消息。

## 4. 延迟加载

API SDK 较重且供应商很多。`*.lazy.ts` 通过 `lazyApi()` 在首次请求时加载具体实现，但立即同步返回外层 event stream；加载失败也转换为规范化 error event。这保持 stream API 同步可用，又让 tree shaking 和 CLI 启动成本可控。

## 5. 兼容策略集中化

“OpenAI-compatible”并不代表行为一致。模型的 `compat` 描述 developer role、strict tools、prompt cache、session affinity、deferred tools 等能力。适配器按 capability 构建 payload，而不是按品牌猜测。

例如 OpenAI Responses adapter 会：

- 对最小输出 token、cache retention 和 session headers 做约束。
- 将 deferred tools 映射为 additional tools 或 tool search。
- 按 provider/base URL 选择 affinity header 格式。
- 清除流解析期间的 `partialJson` 等临时字段，禁止持久化。

兼容策略不一定只存在于 API adapter。OpenCode/OpenCode Go 在 provider 组合层包装所有受支持的 API stream，把 `sessionId` 映射为 `x-opencode-session`；OpenRouter 的 Chat Completions 与 Anthropic Messages 则按 compat 映射为 `x-session-id`。这类路由头仍可被调用方显式 header 覆盖。

deferred tool loading 也保留供应商差异。Fireworks Messages 已声明原生 tool reference 能力，但只有 loader 名称 `ToolSearch` 或 `tool_search` 会让提示前缀真正延迟加载；公共层允许其他名称，适配元数据负责表达这个限制。

## 6. 鉴权与模型目录

模型认证回答“以什么凭据调用哪个 provider”，与模型目录发现和本地工具权限分开。下面沿登录、请求和退出说明它的完整生命周期。

### 6.1 三个独立边界

正式模型认证链处理的是“以什么凭据调用哪个 provider”。project trust 处理“项目配置和扩展能否进入运行时”；工具权限由本地操作系统权限和可选扩展策略决定。供应商 OAuth 登录不会建立一个限制文件读写的沙箱。

```text
登录交互 → ModelRuntime.login → Models.login → provider.auth.<method>.login
                                             ↓ 返回 credential
                                      CredentialStore.modify
                                             ↓
                              同步本地 provider / 可用模型快照

发起模型请求 → ModelRuntime.prepareRequest → getAuth → provider.streamSimple
```

`Models.login()` 等待 provider 登录完成，再通过 store 保存凭据。`ModelRuntime` 按 provider 串行安排登录、退出和临时 key 变更，并在提交后同步模型与认证快照。

以 OpenRouter 登录为例，Pi 生成随机 verifier 和它的 SHA-256 challenge，把 challenge 放进浏览器授权 URL，取得授权码后携带 verifier 换取凭据。这就是 PKCE：兑换方必须证明自己持有发起登录时的秘密。当前实现启动一次性的本地回调服务，也允许远程终端手工粘贴授权结果；登录和兑换分别有超时，最终关闭回调服务、取消另一条等待路径。浏览器授权完成、凭据兑换成功、凭据保存成功是三个不同阶段。

这里有一种值得单独报告的失败：凭据已经写入，但本地快照更新失败。代码用 `CredentialSynchronizationError` 表达这个状态，不能将它笼统解释成“登录没有成功、什么都没保存”。

### 6.2 凭据的选择顺序与失败处理

`resolveProviderAuth()` 的请求解析顺序是：

1. 请求显式提供 `apiKey` 且 provider 支持 API key 时，使用该覆盖值。
2. 否则读取 `CredentialStore`。coding-agent 外面还有 `RuntimeCredentials`，非空临时 key 优先于底层持久凭据，且不写入磁盘。
3. 存储中有 OAuth 就走该 provider 的 OAuth 处理；有 API key 就走 API key 处理。
4. 只有没有存储凭据时，才进入 provider 的 ambient 解析，例如环境变量、云平台 profile 或凭据文件。

例子：用户已经保存账户 A 的 OAuth，本机环境又有账户 B 的 API key。A 刷新失败时，解析器抛错，不会静默改用 B。这避免请求在用户不知情时切换身份、权限或计费账户。存储类型与 provider 不匹配时也不会简单回退到环境凭据。

`models.json`、扩展 header 和自定义认证由 provider 组合层处理，不能把所有配置压成一条通用的“环境变量永远优先”规则。`ModelRuntime.prepareRequest()` 在认证后合并请求 headers，再执行可选的 `transformHeaders`；header 名称合并不区分大小写。

### 6.3 OAuth 刷新如何避免并发覆盖

普通请求解析中，access token 剩余有效期不足默认五分钟就触发刷新。刷新在 `credentials.modify()` 内完成，并在锁内再次读取和检查当前凭据：

```text
A、B 都读到即将过期的 token
  → A 取得存储锁，再检查，刷新并写入新凭据，释放锁
  → B 取得锁，读到 A 的新凭据，跳过本次刷新
```

这个第二次检查是必要的：refresh token 可能随刷新轮换，两个请求都使用旧值会造成冲突。锁内若发现凭据已被删除，就不把旧的 OAuth 状态写回去。

普通请求的刷新 signal 合并调用方取消和 15 秒刷新超时。显式要求最短有效期的调用，还会检查刷新结果是否满足要求。目录刷新另走 `Models.resolveRefreshCredential()`，按实际过期时间检查并使用目录任务的 signal；不能把普通请求的五分钟窗口和 15 秒超时推广到所有 OAuth 调用。

OAuth 也不一定产生短期 access/refresh token 对。例如当前 OpenRouter 实现用 PKCE 授权码换取 API key，保存为 `type: "oauth"`，但 `refresh` 为空，`expires` 为 `Number.MAX_SAFE_INTEGER`，其刷新方法直接返回原凭据。这说明字段容器统一，供应商语义仍然不同。

### 6.4 文件保护、读缓存与退出登录

默认 `AuthStorage` 保存 agent 目录下的 `auth.json`，内容为 JSON，没有在此实现中加密或接入系统钥匙串。文件创建使用 `0600`，新建父目录使用 `0700`；已有文件的权限或管理员 ACL 不会被强制重设。

`FileAuthStorageBackend` 用 `proper-lockfile` 协调读改写。异步抢锁有可取消的退避和截止时间，锁失效会阻止继续写入。它最终使用 `writeFileSync` 直接写文件，因此这里的互斥不能解释成“临时文件替换加 fsync”的崩溃安全事务。

读缓存使用文件 revision 判断是否需要重新加载，并合并共享读状态上的并发 reload。同步 `reload()` 失败会保留最后有效快照；异步读取的错误处理还取决于调用方是否传 signal，不能统一说成永远返回旧值或永远抛错。

`Models.logout()` 删除本地存储凭据；coding-agent 的存储覆盖层同时清除该 provider 的临时 key。此流程没有调用供应商远程撤销接口。删除之后，若环境仍有可用认证，该 provider 仍可能可调用；退出本地保存账户不等于禁用所有认证来源。

实现入口：[model-runtime.ts](../packages/coding-agent/src/core/model-runtime.ts)、[models.ts](../packages/ai/src/models.ts)、[resolve.ts](../packages/ai/src/auth/resolve.ts)、[auth-storage.ts](../packages/coding-agent/src/core/auth-storage.ts)、[runtime-credentials.ts](../packages/coding-agent/src/core/runtime-credentials.ts)、[OpenRouter OAuth](../packages/ai/src/auth/oauth/openrouter.ts)。

### 6.5 网络代理与认证边界

`pi-ai` 的 [node-http-proxy.ts](../packages/ai/src/utils/node-http-proxy.ts) 解析目标协议对应的代理变量、`ALL_PROXY` 和 `NO_PROXY`，接受 HTTP/HTTPS 代理 URL，拒绝 SOCKS/PAC。这属于请求出站的正向代理选择，不是登录系统。

agent-core 的 [streamProxy()](../packages/agent/src/proxy.ts) 则是另一种可选应用接入方式：客户端 POST 到自定义服务的 `/api/stream`，用 `Authorization: Bearer <authToken>` 证明自己有权调用该服务，由服务端管理模型供应商认证。这里的 token 与供应商 API key 是两个边界。这个文件提供客户端实现，不包含 token 签发、服务端校验、用户角色或 Nginx 反向代理部署，也不能据它推断默认 CLI 必经该服务。

这些机制应与[跨进程协议](./08-remote-protocol-and-engineering.md)分开阅读：模型请求代理、出站网络代理、远程 session 路由各自有不同的数据与权限边界。

### 6.6 模型目录与凭据分开

模型目录既有生成的静态快照，也支持远端刷新和本地 store。刷新按 provider generation 隔离：新一代刷新不等待旧的卡住请求，旧结果也不能覆盖新结果。离线模式使用已缓存目录。生成文件 `models.generated.ts` 来自脚本，不应手改。

## 7. 成本与 thinking

统一 usage 区分 input、output、cache read/write 和可选 reasoning。成本由模型价格表计算，支持输入量阶梯和 Anthropic 长时缓存等特殊定价。

thinking level 使用跨供应商的 `off/minimal/low/medium/high/xhigh/max`，模型可通过 map 禁用或映射等级；`clampThinkingLevel()` 在请求能力不存在时选择最近可用等级。

## 8. 先区分不同的 cache

cache（缓存）只表示“保存某些数据供后续复用”。Pi 中有几类不同机制：

| 名称 | 保存在哪里 | 复用什么 | 主要目的 |
|---|---|---|---|
| provider prompt cache | 模型供应商侧 | 多次请求相同的 prompt 前缀 | 降低输入成本和延迟 |
| model catalog cache | 本地 `models-store.json` | 远程获取的模型目录 | 离线启动和减少目录刷新 |
| auth 文件读缓存 | 本地进程内 | 凭据快照 | 按文件 revision 减少重复读盘 |
| 配置命令缓存 | 本地进程内 | `!command` 的输出或失败值 | 减少重复执行凭据命令 |
| process cache | 当前 Pi 进程内 | Git 分支、渲染结果或连接等派生数据 | 避免重复计算或重连 |

它们没有统一失效策略，也不能互相替代。下文分别说明本地缓存的失效规则和供应商 prompt cache。

### 8.1 本地缓存的失效与刷新

默认远程目录包装 `withRemoteCatalog()` 使用四小时刷新窗口，有缓存 body 才发送 ETag 条件请求。304 更新检查时间；临时错误保留缓存内容和 validator。是否采用远程模型还会比较远程 `lastModified` 与内建模型生成时间，避免旧目录覆盖更新的内建数据。该四小时规则属于这个包装，不是所有动态 provider 的统一 TTL。

`Models` 为 provider 刷新分配 generation，新刷新使旧 generation 失效，并在发布前后检查。例子：请求 A 卡住，用户重载后请求 B 先完成；A 晚到不能再更新当前内存目录。持久化还依赖 ModelsStore 遵守 signal 与写入契约。

凭据命令缓存尤其容易误解：`resolveConfigValue()` 会缓存同一命令字符串的结果，包括失败值；`resolveConfigValueOrThrow()` 走 uncached 路径，`resolveHeadersOrThrow()` 也通过它解析。二者都有用途，因此“在请求边界解析”不等于“所有命令每次请求都执行”。命令执行设置了十秒超时，且运行的是受信任配置提供的 shell 代码。

实现入口：[remote-catalog-provider.ts](../packages/coding-agent/src/core/remote-catalog-provider.ts)、[models.ts](../packages/ai/src/models.ts) 的刷新与发布方法、[resolve-config-value.ts](../packages/coding-agent/src/core/resolve-config-value.ts)。

## 9. Prompt cache 怎样工作

Pi 每个 turn 仍会构造并发送完整逻辑 context；它不会把旧消息从请求中删掉并假设供应商记得。API adapter 根据供应商能力添加 cache marker、`prompt_cache_key`、retention 或 session affinity 字段。供应商识别与前一次相同的前缀后，usage 才会报告 `cacheRead`；新写入的可缓存前缀报告为 `cacheWrite`。

`cacheRetention` 表示缓存保留偏好：`none` 禁用，`short` 是默认的供应商常规保留，`long` 只在模型兼容信息声明支持时映射到较长 TTL（生存时间）。不同 provider 的网络字段不同，所以策略在统一 option 中表达，具体编码留在各 API adapter。

稳定的 session ID 有两个相关但不同的用途：它可以作为 prompt cache key，也可能作为供应商的请求路由或连接亲和标识。不能看到 `sessionId` 就断言一定发生了 cache hit；真实命中只以供应商返回的 usage 为准。

上下文压缩会把大量旧消息替换为摘要，改变 prompt 前缀，因此紧接压缩后的请求可能减少缓存命中。评估压缩成本时，需要同时考虑输入量减少和缓存命中变化。压缩使用的总结请求明确设置 `cacheRetention: "none"`，避免为一次性总结写缓存。

`coding-agent/src/core/cache-stats.ts` 比较相邻 assistant usage，估算 cache miss 带来的额外费用；footer 只是展示累计 `cacheRead/cacheWrite` 和最近命中率，不负责缓存本身。

### 9.1 具体实现

缓存仍沿用 3.1 的公共类型和 provider adapter，这次关注 cache 相关字段：

1. 在 [types.ts](../packages/ai/src/types.ts) 中定位 `cacheRetention` 与 usage 字段，确认统一层表达什么、不表达什么。
2. 在 [anthropic-messages.ts](../packages/ai/src/api/anthropic-messages.ts) 中定位 cache marker，再与 [`openai-responses.ts`](../packages/ai/src/api/openai-responses.ts) 对照 key 与 retention 的不同映射；OpenCode 的跨 API session header 见 [`opencode-headers.ts`](../packages/ai/src/providers/opencode-headers.ts)。
3. 读 [`cache-stats.ts`](../packages/coding-agent/src/core/cache-stats.ts)，确认产品怎样依据供应商 usage 估算命中率和额外费用。
4. 在[压缩算法](./06-session-and-persistence.md#36-具体实现)的 `completeSummarization()` 中，可以看到一次性摘要请求为何禁用 prompt cache。
