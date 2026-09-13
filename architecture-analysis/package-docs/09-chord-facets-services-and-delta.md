# Chord：Facet、Service 与 Replicated State

来源：`packages/chord/README.md`、`src/delta/README.md` 与 coding-agent 实验性 service/facet 接入。排除 API 教程和 Delta tuple 全表。

## 1. Chord 的边界

Chord 是独立的应用组合运行时，不依赖其他 Pi package。它解决“一个插件的能力如何拆到 worker、server、TUI 等不同环境，并通过稳定 service 连接”，不实现 Agent、会话存储、CBOR framing 或网络连接。

~~~mermaid
flowchart LR
  PLUGIN["Plugin"] --> WF["worker facet"]
  PLUGIN --> PF["presentation facet"]
  WF --> HOST1["FacetHost / worker"]
  PF --> HOST2["FacetHost / TUI"]
  HOST1 <-->|"application adapter"| HOST2
  HOST1 --> SERVICE["typed services"]
  SERVICE --> STATE["replicated state"]
  STATE --> DELTA["Delta operations"]
~~~

plugin 是分发单位；facet 是其中在特定环境加载并执行初始化的单元。例如，同一插件的 worker facet 提供服务，TUI facet 消费该服务并渲染结果。两个 facet 独立打包和加载，通过服务传递数据。

## 2. FacetHost 如何组合生命周期

facet 在同步 setup 阶段通过 env.provide()/provideMany() 声明 provider，通过 env.use()/observe() 声明 consumer。host 收集完整图后统一验证：缺失 provider、重复 singleton 或依赖不一致会在 activation 前失败。

激活顺序是 provider 先于 consumer，dispose 顺序反向。reload 先验证并激活 candidate，同时保留旧 provider 路由；切换成功后才释放 retired generation。这样 singleton consumer 保留同一个 facade，不经历普通 reload 的 unavailable 窗口。

service 有两种模式：

- singleton：每个 token 一个当前 provider，replacement 在稳定 facade 后切换。
- keyed：provider 动态创建命名 instance；地址包含 key 和 generation，旧 incarnation 不能冒充新实例。

service 可标记为 local，允许任意 JavaScript contract；可远程暴露的 service 必须以 strict JSON 作为参数、结果、catalogue 和状态边界。

## 3. 远程 Service 语义

Chord 定义 transport-neutral 的 service call、catalogue、subscribe/unsubscribe、snapshot/update 和 service error。应用提供 RemoteServiceTransport，将这些逻辑值装入自己的 envelope；Pi 用 pi-protocol 添加 server/session/attachment 路由和 CBOR framing。

consumer 只为 facet 实际需要的 service 建立 binding。provider 暂时断开或替换时，stable handle 保留但处于 unavailable；重新 hydration 后恢复。Chord 不自动 reconnect 网络，也不决定认证、权限或 route。

## 4. 状态副本与增量同步

服务提供方通过 `replicatedState(initial)` 创建可追踪状态，修改其代理对象后调用 `publish(context)`。消费方读取完整、不可变的状态副本；传输层用 Delta 批次编码变化，减少重复发送。

第一次 flush 一定是完整 base，后续 batch 只描述 set/delete、string append/front-truncate 和 array splice 等变化。每个订阅、每个 state member 都有独立的 path encoder/decoder 字典；complete replacement、provider replacement 或重新 hydration 会重置字典。Delta 假设单一权威 writer 和有序传输，sequence、重试与持久化由外层负责。

tracked JSON 必须是无环、无共享可变引用的树。插入 tracker 后只能通过 state proxy 修改；否则变化无法被观察。consumer 的 applyImmutable() 只复制发生变化路径上的容器，使相邻 replica revision 可以共享未变子树。

## 5. Bundle 与 reload 边界

Chord bundler 用 esbuild 将包中各 facet 入口构建为独立的 CommonJS 文件，以内容哈希标识，并写入 `chord-facets.json` 清单。加载器校验 SHA-256，在 `node:vm` 中编译，再通过宿主提供的外部模块解析器加载 peer dependency。

Chord 不安装 package、不运行 lifecycle script。构建先写完整临时目录再替换，加载失败或 reload 失败时保留旧 generation；切换成功后才释放旧 facet 与 bundle 引用，使旧代码可被垃圾回收。

coding-agent 用这一机制把一个实验性插件拆成 src/session.ts 与 src/tui.ts。server/session worker 选择并加载 session facet，presentation 获取匹配的 TUI artifact；/reload 重新构建并原子切换，不把所有插件代码塞入同一进程。

## 6. Context

Chord Context 提供取消和 invocation-scoped values，作用类似 Go context。并发调用各带自己的 Context；共享 service、Harness 或 Session 不保存调用方 Context，也不通过 AsyncLocalStorage 猜测。权限或 telemetry 可以由应用用 typed context key 传递，但 Chord 本身不定义这些策略。

## 7. 具体实现

1. 读 [api.ts](../../packages/chord/src/api.ts) 与 [facets/host.ts](../../packages/chord/src/facets/host.ts)，确认 facet graph、service provider 和 reload。
2. 读 [services/provider.ts](../../packages/chord/src/services/provider.ts)、[services/consumer.ts](../../packages/chord/src/services/consumer.ts) 与 [services/wire.ts](../../packages/chord/src/services/wire.ts)，确认远程 binding 与控制语义。
3. 读 [delta/index.ts](../../packages/chord/src/delta/index.ts) 和 [Delta guide](../../packages/chord/src/delta/README.md)，确认 tracker/replica 边界。
4. 读 [node/bundle.ts](../../packages/chord/src/node/bundle.ts) 与 [node/bundle-loader.ts](../../packages/chord/src/node/bundle-loader.ts)，确认构建、完整性校验和 generation 生命周期。
5. coding-agent 的实际服务集合见 [services/README.md](../../packages/coding-agent/src/experimental/services/README.md)。
