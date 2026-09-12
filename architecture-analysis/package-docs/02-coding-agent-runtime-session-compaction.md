# Coding Agent 运行时、会话树与压缩

来源：`sdk.md`、`session-format.md`、`sessions.md`、`compaction.md`。排除安装、命令教程、设置项和 API 方法清单。

## 1. 三个运行时层次

正式产品路径可以拆成：

```text
AgentSessionRuntime
  管理当前 AgentSession，以及 cwd 变化时的整体替换
    ↓
AgentSession
  输入预处理、扩展、产品事件、重试、压缩和持久化编排
    ↓
pi-agent-core Agent / agent-loop
  模型流、工具循环、steer/follow-up 和基础状态
```

`AgentSession` 表示一个稳定会话。`AgentSessionRuntime` 处理 new、resume、fork、clone、import 等会替换 session 或 cwd 绑定服务的操作。二者分开是因为 cwd 会改变 settings、资源、工具路径和模型配置，不能只替换消息数组。

## 2. SessionManager 的 JSONL 树

文件第一行是 session header，后续每行追加一个 entry。entry 用 `id`/`parentId` 形成树，当前 leaf 表示活动位置。跳到历史节点继续对话只需改变 leaf 并追加新 child，不需要复制文件。

主要 entry 分为：

- message：user、assistant、tool result 等 AgentMessage。
- model/thinking change：记录路径上生效的模型状态。
- compaction：历史摘要和保留边界。
- branch summary：从离开分支带入目标分支的摘要。
- custom：扩展持久状态，不进入模型上下文。
- custom message：扩展注入且会进入模型上下文的消息。
- label/session info：书签与会话展示信息。

磁盘历史、活动分支和发给模型的上下文不是同一个东西。`buildSessionContext()` 沿当前 leaf 回溯活动路径，再应用模型变更、压缩和自定义 entry 的投影规则。

## 3. 持久化时机

`AgentSession` 订阅 Agent 事件。user message、assistant message 和 tool result 在各自完成时追加，因此长请求中间已经存在可恢复的已完成历史。流式 partial assistant message 不作为最终 entry 提前写入。

## 4. 压缩如何改变上下文

压缩解决 context window 限制，不删除历史。流程是：

1. 从新到旧估算 token，选择要保留的近期边界。
2. 收集边界之前需要总结的消息。
3. 调用模型生成结构化摘要。
4. 追加 `CompactionEntry`。
5. 后续上下文变为“摘要 + retained tail + 压缩后的新消息”。

当单个 turn 自身超过保留预算时，切点可能位于 turn 中部。实现会分别总结更早历史和当前 turn 的前缀，避免把 tool result 与对应 tool call 错误拆开。

自动压缩有三个触发语义：阈值、provider 返回 context overflow 后的恢复，以及手动请求。触发和 UI/重试编排由 `AgentSession` 负责，摘要算法位于 coding-agent 的 compaction 模块。

`reserveTokens` 与 `keepRecentTokens` 可以在普通 compaction 设置中定义，也可以由 `modelOverrides` 按精确 `provider/modelId` 覆盖；两个字段独立回退到普通值和内建默认值。手动、阈值、overflow 与 extension preparation 使用同一组按活动模型解析的值，模型切换从下一次检查开始生效。

## 5. Branch summary 与 compaction 的区别

compaction 缩短同一活动路径的模型上下文；branch summary 在树导航时，把离开分支的重要信息附加到目标分支。后者先找旧 leaf 与目标位置的共同祖先，只总结旧分支独有的区间。

两者使用相似的结构化摘要与文件操作累计信息，但产生原因和落点不同：compaction 是上下文容量管理，branch summary 是跨分支信息转移。

compaction 与树导航共享互斥状态。`navigateTree()` 在已有压缩或导航运行时直接拒绝，UI 在异步摘要选择对话框返回后也会再次检查，避免旧操作覆盖新操作的 leaf、status indicator 或 escape handler。

## 6. 与 agent-core durable session 的区别

本篇描述的是正式产品当前使用的 `SessionManager`。agent-core 的 `SessionRepo`/`Storage` 是另一套 durable session API，使用 entry、typed value/list、usage 与可恢复 operation。两者不能因为都使用 JSONL、树和压缩就视为同一实现；只有实验性 session worker 使用后者。

## 7. 具体实现

1. 读 [`agent-session.ts`](../../packages/coding-agent/src/core/agent-session.ts) 的 `prompt()` 与 `_handleAgentEvent()`，确认产品编排边界。
2. 读 [`session-manager.ts`](../../packages/coding-agent/src/core/session-manager.ts) 的 `_persist()` 与 `buildContextEntries()`，确认历史写入和上下文投影。
3. 读 [`compaction.ts`](../../packages/coding-agent/src/core/compaction/compaction.ts) 的 `prepareCompaction()` 与 `compact()`，确认切分和摘要算法。
4. 回到 [`agent-session.ts`](../../packages/coding-agent/src/core/agent-session.ts) 的 `_checkCompaction()`，确认阈值、overflow 和恢复怎样接入产品生命周期。
