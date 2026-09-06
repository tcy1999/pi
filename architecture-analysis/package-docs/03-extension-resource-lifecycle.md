# 扩展系统、资源加载与信任边界

来源：`extensions.md`、`sdk.md`、`packages.md`、`security.md`。排除扩展编写教程、API 方法全集、package 安装与 manifest 示例。

## 1. 扩展系统不是简单回调列表

扩展经过四个阶段：发现源码、执行 factory 收集注册、将注册结果绑定到当前 session、由 `ExtensionRunner` 按生命周期调用。扩展可以注册工具、命令、provider、输入转换、模型请求 hook 和 UI 能力，因此它属于产品运行时的一部分，不是静态配置。

```text
ResourceLoader 发现扩展与资源
  → loader 执行 extension factory
  → ExtensionRuntime 保存注册信息
  → AgentSession 绑定 ExtensionRunner
  → prompt / turn / tool / session 事件依次经过 runner
```

factory 可以异步完成启动注册，但不应直接创建长生命周期资源，因为某些调用只做 list-models 等工作，不一定启动 session。session 级进程、socket、watcher 应在 `session_start` 后建立，并在 `session_shutdown` 释放。

## 2. 事件分为观察与拦截

部分事件只报告事实，例如 message update、tool execution update；另一些事件可以改变控制流或数据：

- `input` 可以消费或转换输入。
- `before_agent_start` 可以注入消息或改 system prompt。
- `context` 可以改变本次模型请求看到的消息。
- provider hooks 可以改变 header 或 payload。
- `tool_call` 可以阻止工具，`tool_result` 可以改变返回结果。
- session before 事件可以取消切换、fork、压缩或导航。

需要返回值的 hook 通常顺序执行，使后一个 handler 能看到前一个 handler 的变更。普通扩展错误被记录后继续；工具调用拦截失败则偏向 fail-safe，避免在检查失效时仍执行工具。

## 3. Session 替换是完整生命周期边界

new、resume、fork 和 clone 不只是换一个 session 文件。运行时需要：

```text
before 事件，可取消
  → 中止并等待当前工作收尾
  → session_shutdown
  → 释放旧 extension context
  → 按目标 cwd 重建 settings、资源和模型服务
  → 创建并绑定新 AgentSession
  → session_start
```

旧 session 的事件订阅、UI 组件和 extension context 不能继续用于新 session。扩展的内存状态若需要支持分支，应从活动 branch 的持久 entry 重建，而不是只依赖进程内变量。

## 4. 资源系统的边界

`DefaultResourceLoader` 汇总 extensions、skills、prompt templates、themes 和 context files，并加载 system prompt 与 append-system-prompt 来源。来源可能是全局目录、项目目录、CLI 路径或 package。package manager 负责来源解析和安装，resource loader 负责把允许的路径变成带来源信息的资源，并报告冲突与诊断。

Skills 与 prompt templates 是模型输入资源；extensions 是在宿主进程执行的代码；themes 是表现资源。它们都能被 package 分发，但信任级别不同，不能因为目录结构相似就视为同一种扩展机制。

## 5. 项目信任不是沙箱

项目资源可能在启动时改变配置或执行扩展代码，因此正式产品先解析 trust，再加载受保护的项目资源。全局/CLI 扩展可以参与 trust 决策；项目扩展在信任完成前不能参与，否则会形成“由待审代码批准自己”的循环。

trust 只控制加载哪些输入。Pi 的工具和扩展仍以当前用户权限运行，没有进程内权限隔离。真正的文件、进程、凭据和网络边界必须由容器、VM 或操作系统策略提供。

## 6. 不同 mode 的 UI 能力

扩展始终运行在产品 session 中，但 UI 能力取决于 mode：TUI 能提供完整组件；RPC 可把对话框转换成请求/响应子协议；JSON 与 print 没有交互 UI。扩展应根据 `mode` 和 `hasUI` 判断能力，而不能假设注册扩展就一定存在终端。

## 7. 具体实现

1. 读 [`types.ts`](../../packages/coding-agent/src/core/extensions/types.ts)，确认扩展公开 API 与事件类型。
2. 读 [`loader.ts`](../../packages/coding-agent/src/core/extensions/loader.ts)，确认扩展 factory 怎样执行并收集注册。
3. 读 [`runner.ts`](../../packages/coding-agent/src/core/extensions/runner.ts)，确认 hook 顺序、结果合并和错误策略。
4. 读 [`agent-session.ts`](../../packages/coding-agent/src/core/agent-session.ts) 的 `bindExtensions()`，确认扩展怎样绑定到正式 session。
