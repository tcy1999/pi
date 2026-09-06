# RPC 模式与事件协议

来源：`rpc.md`、`json.md`、`sdk.md`。排除启动参数、命令全集、类型表和客户端示例。

## 1. RPC 是正式 AgentSession 的进程边界

RPC mode 不是 `pi-client`/`pi-server`。它在一个子进程内运行正式 `AgentSessionRuntime`，通过 stdin/stdout 暴露控制接口：

```text
宿主进程
  → stdin JSONL command
pi coding-agent 子进程
  → AgentSessionRuntime → AgentSession
  → stdout JSONL response / event
```

它适合单个正式 coding-agent 子进程的语言无关集成。另一套实验边界由 protocol/client/server + Chord 组成：外层协议负责 server/session/attachment 路由，Chord service 负责 typed 调用和 replicated state，session worker 内运行 `AgentHarness`。

## 2. 三类消息

- command：宿主发出的 prompt、模型切换、会话操作等请求。
- response：命令是否成功；可用 request ID 关联。
- event：Agent、message、tool、queue、compaction 和 retry 的异步状态流。

命令完成与 Agent 流式事件是两个维度。客户端不能假设收到 response 才会出现事件，也不能把 `agent_end` 当成产品完全 idle；自动重试、overflow compaction 或 follow-up 之后才会出现 `agent_settled`。

## 3. JSONL framing

每条消息是一行 JSON，只以 LF `\n` 分隔。实现故意不用 Node `readline`，因为后者还会按 Unicode line separator 切分，而这些字符可以合法出现在 JSON 字符串中。

这里的 framing 与远程 protocol 的“长度前缀 + CBOR”不同。RPC 选择 JSONL 是为了子进程管道易调试；远程 protocol 选择二进制 framing 是为了 transport-neutral 的严格消息边界。远程 payload 中的 service 语义由 Chord 解析，不由 `pi-protocol` 枚举应用命令。

## 4. 流式消息是 delta，不是重复 snapshot

`message_start` 建立消息，随后 `message_update` 携带 text、thinking 或 tool-call delta，`message_end.message` 是最终权威结果。客户端需要按 `contentIndex` 归并 partial state。

工具事件分为 start、update、end。并行工具的 update 可以交错，end 按实际完成顺序出现，而最终 toolResult message 仍按 assistant 中的 tool-call 源顺序进入对话。UI 必须用 `toolCallId` 关联，不能依赖事件相邻。

## 5. Extension UI 子协议

RPC 没有真实 TUI，但扩展仍可能请求 select、confirm 或 input。运行时把这类调用转换为 stdout request，并等待宿主通过 stdin 返回 matching response。notify/status 等操作是 fire-and-forget。

因此 RPC 中 `hasUI` 可以为真，但只表示宿主能够处理结构化 UI 请求；依赖真实终端组件、主题或自定义 component 的能力仍不可用。

## 6. 生命周期语义

一次低层 run 用 `agent_start/end` 包围；一次模型响应及其工具批次用 `turn_start/end` 包围；message 与 tool 有各自 start/update/end。`agent_settled` 位于 session 产品策略完成之后，是外部状态集成更可靠的“不会自动继续”信号。

## 7. 具体实现

1. 读 [`rpc-types.ts`](../../packages/coding-agent/src/modes/rpc/rpc-types.ts)，确认命令、响应和事件的边界。
2. 读 [`rpc-mode.ts`](../../packages/coding-agent/src/modes/rpc/rpc-mode.ts)，确认服务端怎样分发命令并转发 session 事件。
3. 读 [`rpc-client.ts`](../../packages/coding-agent/src/modes/rpc/rpc-client.ts)，确认调用方怎样关联响应并维护状态。
4. 读 [`jsonl.ts`](../../packages/coding-agent/src/modes/rpc/jsonl.ts)，确认对象怎样映射到逐行 JSON 传输。
