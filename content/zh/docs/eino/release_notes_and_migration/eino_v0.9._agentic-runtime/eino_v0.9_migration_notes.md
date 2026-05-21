---
Description: ""
date: "2026-05-19"
lastmod: ""
tags: []
title: Eino V0.9 更新注意事项
weight: 1
---

本文列出现有用户从 V0.8.x 升级到 V0.9 `agentic-runtime` 时需要关注的 API 和语义变化。未列出的新增能力通常不影响既有 `*schema.Message` 路径。

## API 显式变更

### Agent Transfer / Workflow Agent / Supervisor 标记为 NOT RECOMMENDED

V0.9 将基于 Agent Transfer（全上下文共享）的多 Agent 协作模式整体标记为 **NOT RECOMMENDED**。受影响的公开 API 包括：

**Agent Transfer 相关**：

- `SetSubAgents`
- `AgentWithOptions` / `WithDisallowTransferToParent` / `WithHistoryRewriter`
- `ChatModelAgentConfig.Exit` / `ChatModelAgentConfig.OutputKey`
- `AgentWithDeterministicTransferTo`
- `OnSetSubAgents` / `OnSetAsSubAgent` / `OnDisallowTransferToParent`

**Workflow Agent**：

- `NewSequentialAgent` / `SequentialAgentConfig`
- `NewParallelAgent` / `ParallelAgentConfig`
- `NewLoopAgent` / `LoopAgentConfig`

**Supervisor**：

- `supervisor.New` / `supervisor.Config`

> 💡
> 这些 API 仍然可以使用，不会编译失败，但不建议在新项目中采用。经验表明，Agent 之间共享完整对话上下文的 transfer 模式在实际效果上并不优于工具调用模式。

推荐迁移方向：

- 使用 `ChatModelAgent` + `AgentTool`（将子 Agent 封装为工具，按需调用）。
- 使用 `DeepAgent`（结构化子任务委派）。
- 上述两种方式均可获得更好的可控性、可观测性和 prompt cache 效率。

### ChatModelAgentMiddleware 新增 AfterAgent

`ChatModelAgentMiddleware` 新增 `AfterAgent` 方法。手写实现该接口的类型需要补充该方法，否则会编译失败。

推荐做法：

- 如果 middleware 不需要特殊收尾逻辑，嵌入 `*adk.BaseChatModelAgentMiddleware`。
- 如果 middleware 需要在 Agent 成功结束后清理状态、记录事件或补充统计，实现 `AfterAgent(ctx, state)`。

影响范围：

- 仅影响显式实现 `ChatModelAgentMiddleware` 的用户代码。
- 通过 `BaseChatModelAgentMiddleware` 组合扩展的代码可保持兼容。

### AgentMiddleware 结构体废弃

`AgentMiddleware` 结构体及 `ChatModelAgentConfig.Middlewares` 字段已标记为 **Deprecated**，将在未来版本中移除。

> 💡
> AgentMiddleware 和 Middlewares 字段均已废弃。请迁移至 interface-based 的 Handlers（ChatModelAgentMiddleware）方式。

迁移方式：

- 将 `Middlewares []AgentMiddleware` 中的各项逻辑迁移到 `Handlers []ChatModelAgentMiddleware`。
- `AgentMiddleware.BeforeChatModel` → 实现 `ChatModelAgentMiddleware.BeforeModelRewriteState`。
- `AgentMiddleware.AfterChatModel` → 实现 `ChatModelAgentMiddleware.AfterModelRewriteState`。
- `AgentMiddleware.WrapToolCall` → 实现 `ChatModelAgentMiddleware.WrapToolCall`。
- `AgentMiddleware.AdditionalInstruction` → 在 `BeforeModelRewriteState` 中修改 `state.Instruction`。
- `AgentMiddleware.AdditionalTools` → 在 `BeforeModelRewriteState` 中修改 `state.ToolInfos`。
- 如果 middleware 不需要特殊逻辑，嵌入 `*adk.BaseChatModelAgentMiddleware` 以获得默认空实现。

影响范围：

- 所有在 `ChatModelAgentConfig.Middlewares` 中使用 `AgentMiddleware` 的代码需要迁移。
- 当前版本两种方式可共存（Handlers 在 Middlewares 之后执行），但建议尽早迁移以避免未来版本移除时的编译失败。

### summarization.SummarizeMessages 被移除

`summarization.SummarizeMessages` 和 `summarization.SummarizeOutput` 不再导出。

迁移方式：

- 构造 summarization middleware 时继续使用 `summarization.New` 或 `summarization.NewTyped`。
- 需要主动触发同步 summarization 时，使用 `TypedMiddleware.Summarize`。

该调整将 summarization 的配置、状态读取和执行逻辑收敛到 middleware 内部，避免独立函数与运行时状态语义分叉。

## 需要关注语义变化的能力

### Summarization Finalize 后处理语义变化

V0.8.x 中，summarization middleware 会先执行默认 summary 后处理，再调用用户配置的 `Finalize`。因此自定义 `Finalize` 收到的 `summary` 已经包含 `PreserveUserMessages` 替换、`TranscriptFilePath` 注入和 summary preamble。

V0.9 中，如果设置了 `Config.Finalize`，middleware 会直接把模型生成的 raw summary 传给 `Finalize`，不再自动执行默认后处理。受影响的配置包括：

- `PreserveUserMessages`
- `TranscriptFilePath`

迁移方式：

- 如果希望保留默认后处理，不要设置 `Finalize`，让 middleware 使用默认 finalization 路径。
- 如果必须自定义 `Finalize`，但仍希望保留默认后处理，先通过 `DefaultFinalizer` 构造默认 finalizer，再在自定义逻辑中显式组合。
- `DefaultFinalizer` 不会自动读取外层 `Config.PreserveUserMessages` 和 `Config.TranscriptFilePath`；需要通过 `DefaultFinalizerConfig` 显式传入。
- 使用 `NewFinalizer().PreserveSkills(...).Build()` 的代码需要特别检查：该 finalizer 只负责 preserve skills，不会自动补上 `PreserveUserMessages` 和 `TranscriptFilePath`。

### 工具列表修改路径调整

`ModelContext.Tools` 不再是推荐的工具列表修改入口。

升级建议：

- 在 `BeforeModelRewriteState` 中修改 `state.ToolInfos`。
- 如需模型原生 deferred tool search，修改 `state.DeferredToolInfos`。
- 不建议在 `WrapModel` 中修改工具列表；该修改只影响当前模型调用，后续 middleware、后续 turn 或 checkpoint/resume 不会继承这次修改。

### ToolSearch / AgentsMD Middleware 内部实现迁移

ToolSearch 和 AgentsMD middleware 的内部实现从 `WrapModel`（v0.8.x）迁移至 `BeforeModelRewriteState`（v0.9）。

> 💡
> 对仅使用 `toolsearch.New()` / `agentsmd.New()` 的用户，公开 API（Config 结构体、构造函数）未变化，无需修改代码。

语义变化：

- **v0.8.x**：middleware 通过 `WrapModel` 在模型调用时临时注入工具列表（via `model.Option`），变更不持久化，不进入 agent state。
- **v0.9**：middleware 在 `BeforeModelRewriteState` 中直接修改 `state.ToolInfos` / `state.DeferredToolInfos` 和 `state.Messages`（注入提醒消息），变更随 state 持久化。

影响：

- **Checkpoint/Resume**：ToolSearch 注入的提醒消息和动态工具搜索结果现在会随 checkpoint 持久化并在恢复时正确重建，v0.8.x 中这些信息会在恢复后丢失。
- **其他 Middleware 可见性**：后续 middleware 的 `BeforeModelRewriteState` / `AfterModelRewriteState` 现在能看到 ToolSearch 修改后的 `state.ToolInfos`，而 v0.8.x 中这些修改对其他 middleware 不可见。
- **Prompt Cache**：由于工具列表变更现在反映在 state 中（而非每次模型调用时临时注入），模型的 KV-cache 行为可能有差异。

需要注意：

- 如果有自定义 middleware 依赖 `WrapModel` 中的 `ModelContext.Tools` 来读取/修改工具列表，应迁移至 `BeforeModelRewriteState` 中读取 `state.ToolInfos`。

### Model Retry 决策语义增强

`ModelRetryConfig` 新增 `ShouldRetry`。当 `ShouldRetry` 非空时，`IsRetryAble` 会被忽略。

需要注意：

- 旧的 `IsRetryAble` 仍可用于错误维度的简单重试。
- 使用 `ShouldRetry` 后，应显式处理成功输出但业务不接受的场景。
- Interrupt 和 `ErrStreamCanceled` 不作为普通 retry error 处理。

### Cancel 错误语义

V0.9 引入主动取消语义后，应用需要区分主动取消、普通错误和业务 interrupt。

升级建议：

- 上层应区分 `CancelError`、普通 error 和业务 interrupt。
- 如果应用主动接入 `WithCancel`，不要把 `CancelError` 当作普通业务失败处理。

### AgenticMessage 迁移需要理解新的消息结构

`TypedChatModelAgent[*schema.AgenticMessage]` 是面向模型原生 Agentic 协议的新路径。迁移到该路径不只是把泛型参数从 `*schema.Message` 改成 `*schema.AgenticMessage`，还需要按 `AgenticMessage` 的 content block 结构处理消息内容。

需要注意：

- AgenticMessage 路径使用 `AgenticModel` 与 `AgenticToolsNode` 处理工具调用。
- 工具调用和工具结果通过 `AgenticMessage` content block 表达，尤其需要正确处理 tool call / tool result content block。
- Agent transfer 能力不适用于 AgenticMessage 路径。
- 既有应用如果不需要模型原生 Agentic 协议，建议继续使用默认 `*schema.Message` 路径；只有在明确要接入 `AgenticModel` 协议时再迁移。

### 模型适配器需要识别新增 option

V0.9 引入 `AgenticModel` 后，模型适配器需要更严格地处理 call-time options。`AgenticModel` 是 `BaseModel[*schema.AgenticMessage]` 的别名，不再提供类似 `ToolCallingChatModel.WithTools` 的增强接口；工具绑定统一通过 `model.WithTools` 作为 `model.Option` 传入。

需要注意：

- 所有支持 AgenticMessage 的模型适配器都应读取 `Options.Tools`，并将其映射到 provider 的 tool calling 协议。
- `AgenticModel` 不应要求用户先调用某个 `WithTools` 方法得到“带工具的模型实例”；ADK 会在每次模型调用时通过 `model.WithTools` 传递当前工具列表。
- 如果适配器只从自身 config 读取工具，而忽略 `model.WithTools`，在 ChatModelAgent / AgenticToolsNode 路径下会出现模型看不到工具或工具列表不随运行态变化的问题。

V0.9 还在 `model.Options` 中新增：

- `DeferredTools`
- `ToolSearchTool`
- `AgenticToolChoice`

现有模型适配器忽略这些 option 通常不会导致编译失败，但会导致 deferred tool search、模型原生 tool search 或 agentic tool choice 不生效。适配器维护者应按目标 provider 的协议补齐转换逻辑。

### ToolInfo 序列化形态变化

`ToolInfo` 增加显式 JSON/Gob 编解码，以保留 `ParamsOneOf`。

影响：

- `ToolInfo` 进入了 `ChatModelAgentState.ToolInfos` / `DeferredToolInfos`，因此可能随 Agent state 一起进入 checkpoint。
- 显式 JSON/Gob 编解码用于保证 `ParamsOneOf` 在 checkpoint、deep copy 和恢复过程中不会丢失。
- 如果外部系统直接依赖旧版 `ToolInfo` JSON 形态，需要重新确认序列化兼容性。
