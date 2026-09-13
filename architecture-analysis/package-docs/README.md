# Package 架构专题

本目录从各 package 的 `docs/` 中提炼架构说明，按问题重新组织为中文文档。它不是英文文档的逐段翻译，也不包含安装、命令用法、配置项清单或 API 教程。

这些专题适合按 package 单独查问题，与上层八章有内容重合。已经沿主线读过前八章，可以把这里当作索引，不必再顺序读一遍。

## 来源范围

当前架构来源覆盖：

- `packages/agent/docs/`：当前 Harness 规范、assistant/tool durability、session values、telemetry、plugin/facet 设计和实施 work packages，以及尚未进入源码的 pico 下一代设计文档。
- `packages/coding-agent/docs/`：文档站按 Start here、Customization、Reference、Programmatic Usage、Platform Setup、Development 六组导航；本目录从其中的 SDK、session format、compaction、extensions、TUI、RPC、安全、模型/provider 和设置文档抽取架构事实。
- `packages/chord/README.md` 与 `packages/chord/src/delta/README.md`：facet、service、replicated state、Delta 和 bundle/reload 边界。

## 专题

1. [AgentHarness：可恢复运行时与待实现能力](./01-agent-harness-status-and-design.md)
2. [Coding Agent 运行时、会话树与压缩](./02-coding-agent-runtime-session-compaction.md)
3. [扩展系统、资源加载与信任边界](./03-extension-resource-lifecycle.md)
4. [TUI 组件、渲染与焦点模型](./04-tui-rendering-architecture.md)
5. [RPC 模式与事件协议](./05-rpc-and-event-boundary.md)
6. [会话搜索与遥测边界](./06-search-and-telemetry.md)
7. [模型目录、Provider 组合与认证边界](./07-model-provider-and-auth-runtime.md)
8. [设置、Package 与资源解析链](./08-settings-package-resource-resolution.md)
9. [Chord：Facet、Service 与 Replicated State](./09-chord-facets-services-and-delta.md)

## 扫描清单

下表覆盖稳定产品文档，并把 agent-core 新增的研究/实施文档按主题归组。一个原文可以同时包含架构、计划和教程；已实现行为与设计计划按文末的状态标记区分。

| 原文 | 处理结果 |
|---|---|
| `agent/docs/harness.md`、`assistant-durability.md`、`tool-durability.md`、`values.md` | 提取 durable operation、外部效果检查点、typed value/list 和当前完成度，见专题 1、6 |
| `agent/docs/work-packages/`、`runtime-simplification.md`、`post-wp05-roadmap.md` | 用于校验实施决策；已落地部分并入专题 1，未实现计划不写成当前事实 |
| `agent/docs/pico-v3.md`、`pico2.md`、`agent/docs/pico/` | 下一代 pico reference design、使用面草案、handoff 与 implementation plan；当前 `packages/agent/src/` 尚无 pico runtime，不写成现有能力 |
| `agent/docs/plugins.md`、`mobile-handoff/` | 提取 facet/service、状态复制和跨进程边界；模式示例不写成内建能力，见专题 1、9 |
| `agent/src/search/index.ts` | 原 `agent/docs/search.md` 与具体实现已移除；当前仅保留服务接口草案，见专题 6 |
| `agent/docs/telemetry-schema.md` | 提取 span 层次和内容边界，不复制生成字段表，见专题 6 |
| `coding-agent/docs/sdk.md` | 提取 `AgentSessionRuntime`、`AgentSession`、资源与 mode 的关系，见专题 2、3、5 |
| `coding-agent/docs/session-format.md` | 提取 JSONL 树、entry 类型和 context projection，见专题 2 |
| `coding-agent/docs/sessions.md` | 提取 session 替换、branch/fork/clone 语义，见专题 2、3 |
| `coding-agent/docs/compaction.md` | 提取 compaction 与 branch summary 算法边界，见专题 2 |
| `coding-agent/docs/extensions.md` | 提取扩展加载、hook 顺序、生命周期和 UI 边界，见专题 3 |
| `coding-agent/docs/packages.md` | 提取 package 身份、scope、过滤和资源解析，见专题 8 |
| `coding-agent/docs/security.md` | 提取 project trust 与 OS 沙箱边界，见专题 3、8 |
| `coding-agent/docs/tui.md` | 提取组件协议、差量渲染、焦点和 overlay，见专题 4 |
| `coding-agent/docs/rpc.md` | 提取 JSONL 进程协议、事件顺序和 extension UI 子协议，见专题 5 |
| `coding-agent/docs/json.md` | 提取 JSON event stream 与 RPC 的边界，见专题 5 |
| `coding-agent/docs/models.md` | 提取模型/provider 配置的组合位置；字段与示例不收录，见专题 7 |
| `coding-agent/docs/custom-provider.md` | 提取 provider 注册、认证和 stream contract；实现教程不收录，见专题 7 |
| `coding-agent/docs/providers.md` | 提取 provider catalog 与 credential 解析边界；登录步骤和云配置不收录，见专题 7 |
| `coding-agent/docs/settings.md` | 提取 global/project merge、trust gate、资源设置和按模型解析 compaction 预算，见专题 2、8 |
| `coding-agent/docs/usage.md` | 提取 mode、消息队列、context file 和 trust 的架构信息，已并入专题 2、3、5、8；CLI 表不收录 |
| `coding-agent/docs/index.md` | 导航页，无独立架构内容 |
| `coding-agent/docs/quickstart.md` | 安装、认证和首次使用教程，不收录 |
| `coding-agent/docs/keybindings.md` | 按键参考和定制教程，不收录 |
| `coding-agent/docs/themes.md` | 主题制作和 token 参考；主题作为资源的边界已在专题 3、8 说明 |
| `coding-agent/docs/skills.md` | skill 编写和安装教程；skill 与 extension 的边界已在专题 3 说明 |
| `coding-agent/docs/prompt-templates.md` | template 写法与调用教程；资源类型边界已在专题 3 说明 |
| `coding-agent/docs/environment-variables.md` | 环境变量参考；shell tool 的 session metadata 注入不构成独立架构专题 |
| `coding-agent/docs/containerization.md` | 外部沙箱使用教程；“Pi 本身不是沙箱”已在专题 3 说明 |
| `coding-agent/docs/llama-cpp.md` | 本地模型服务安装和命令教程，不收录 |
| `coding-agent/docs/terminal-setup.md` | 终端兼容配置，不收录 |
| `coding-agent/docs/tmux.md` | tmux 配置，不收录 |
| `coding-agent/docs/termux.md` | Android 安装与排错，不收录 |
| `coding-agent/docs/windows.md` | Windows shell 设置，不收录 |
| `coding-agent/docs/shell-aliases.md` | shell 配置片段，不收录 |
| `coding-agent/docs/development.md` | 开发环境、测试和 rebrand 指南；包地图信息已由总览源码核对，不单独翻译 |
| `chord/README.md`、`chord/src/delta/README.md` | 提取 facet lifecycle、service、replicated state、Delta 和 bundle/reload，见专题 9 |

`docs.json` 是文档站导航与 redirect 的权威清单；静态测试会检查导航路径、Markdown 链接和孤儿页。它不提供运行时架构事实。`docs/images/` 是图片资源，也不属于架构来源。

## 状态标记

- **正式路径**：正式 CLI/SDK 当前确实调用。
- **实验路径**：源码、测试和实验入口已接通，但不属于默认稳定 CLI/SDK。
- **已实现基础模块**：源码与测试存在，但不代表已接入正式产品。
- **目标规格**：设计文档描述的目标，当前源码可能尚未实现。
