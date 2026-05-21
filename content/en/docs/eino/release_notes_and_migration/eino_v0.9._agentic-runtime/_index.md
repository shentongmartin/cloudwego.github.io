---
Description: ""
date: "2026-05-21"
lastmod: ""
tags: []
title: v0.9.* agentic-runtime
weight: 9
---

The theme of V0.9 is `agentic-runtime`. This release focuses on ADK's message protocol, Agent execution control, and multi-turn runtime capabilities. While preserving the default `*schema.Message` path, it introduces `AgenticMessage` along with generic abstractions, laying the foundation for richer model-native Agent protocols, server-side tool calling, execution interruption and resumption.

## 1. AgenticMessage and ADK Support

V0.9 introduces `schema.AgenticMessage` to express a more complete Agentic message structure compared to the traditional `schema.Message`.

- `AgenticMessage` adopts a content block model, supporting structured fragments such as text, reasoning content, tool calls, tool results, server-side tools, MCP tools, and multimodal content.
- `[]ContentBlock` preserves the block ordering from different model protocol responses more completely; the new block types are also better suited for structures like tool use, reasoning, and streaming metadata in protocols such as OpenAI Responses API, Claude, and Gemini.
- `components/model` introduces the `AgenticModel` component for integrating model implementations that use `AgenticMessage` as input/output.
- ADK provides typed agent, typed event, typed runner, and typed `ChatModelAgent` support for the `AgenticMessage` path, enabling AgenticModel to participate in the ADK Agent lifecycle.
- [Eino: Quick Start](/docs/eino/quick_start): The entire series has been rewritten based on AgenticMessage.

## 2. ChatModelAgent Capability Enhancements

V0.9 systematically enhances `ChatModelAgent`'s execution control, model call reliability, and middleware extension points.

### Cancel

- Introduces Agent Cancel capability for externally terminating a running Agent.
- Supports safe-point cancellation, recursive cancellation, cancel timeout escalation, and checkpoint persistence during cancellation.
- Interrupts that occur during cancellation are unified under cancel semantics; callers can distinguish active cancellation from normal business failures via `CancelError`.
- [Eino ADK: Agent Cancel and TurnLoop Quick Start](/docs/eino/core_modules/eino_adk/agent_cancel_and_turnloop_quickstart)

### Model Retry

- Retry has been expanded from simple error retry to `ShouldRetry(ctx, RetryContext) -> RetryDecision`.
- Retry decisions can read model output, reject outputs that don't meet conditions, modify the next input, append model options, and override backoff.

### Model Failover

- Introduces Model Failover capability for switching to backup models after a model call failure.
- Failover decisions can read the failed attempt's output, error, original input, and attempt number, then select which model to use next.
- Supports rewriting input for backup models; also supports prioritizing the last successfully called model to reduce the cost of starting from the fixed primary model each time.
- [ChatModel Failover Guide](/docs/eino/core_modules/eino_adk/agent_implementation/chat_model/chatmodel_failover_guide)

### Middleware Enhancements

- `ChatModelAgentMiddleware` adds `AfterAgent` for executing cleanup logic after an Agent completes successfully.
- Summarization, reduction, skill, filesystem, plan-task, patch-tool-calls and other middlewares have been genericized to support the `AgenticMessage` path.
- Summarization middleware adds `TypedMiddleware.Summarize`, transitioning synchronous summarization from a standalone function to a cohesive middleware capability.
- Filesystem middleware enhances multimodal reading capabilities and adds PDF pages validation.
- Introduces `agentsmd` middleware for loading and injecting `AGENTS.md` style project instructions.
- `ChatModelAgentState` adds `ToolInfos` and `DeferredToolInfos` as the primary path for middlewares to adjust the tool set visible to the model.
- `ToolInfos` represents tools directly visible to the current model call; `DeferredToolInfos` represents candidate tools that can be discovered by the model on demand through tool search mechanisms.
- Tool search middleware supports three tool loading approaches: using the model-side native tool search capability to load from deferred tools on demand; providing a fixed-schema `ToolSearchTool` per model protocol requirements, allowing the model to search deferred tools through this entry point; without relying on model-side protocol, using Eino's custom `tool_search` tool to retrieve tools and append matches to regular `ToolInfos`.
- Compose adds `AgenticToolsNode`; `ToolsNode` adds tool name and argument alias support.
- [Eino ADK: ChatModelAgentMiddleware](/docs/eino/core_modules/eino_adk/eino_adk_chatmodelagentmiddleware)

## 3. TurnLoop

V0.9 introduces `TurnLoop` to elevate a one-shot Agent run into a continuously running, externally-driven turn-level runtime.

- Designed for multi-turn execution: `TurnLoop` continuously receives external input, with each turn independently planning input, constructing the Agent, and consuming events—suitable for long-running interactive Agents.
- Supports input merging: `GenInput` decides at the turn boundary which inputs to consume in this turn and which to continue waiting for, enabling applications to implement batching, deduplication, merging of consecutive user inputs, and other strategies.
- Supports preemption: `Push` with a preempt option atomically writes new input and requests cancellation of the current turn, allowing high-priority input to interrupt a running Agent.
- Supports declarative checkpoint/resume: on recovery, applications don't need to manually restore the input queue; `TurnLoop` distinguishes between interrupted inputs, unprocessed inputs, and newly arrived inputs after recovery—applications only need to declare how these inputs re-enter subsequent turns.
- [Eino ADK: Agent Cancel and TurnLoop Quick Start](/docs/eino/core_modules/eino_adk/agent_cancel_and_turnloop_quickstart)

## Upgrade Guide

> 💡
> As of now (5.19), the latest version is v0.9.0-beta.1, with the stable release expected in about one week. Before the stable release, always use the latest beta version; after the stable release, upgrade to the latest stable version.

```bash
# Upgrade to the latest beta (use before stable release)
go get github.com/cloudwego/eino@v0.9.0-beta.1
go get github.com/cloudwego/eino-ext/components/model/agenticopenai@v0.2.0-beta.1
go get github.com/cloudwego/eino-ext/components/model/agenticark@v0.2.0-beta.1
go get github.com/cloudwego/eino-ext/components/model/agenticclaude@v0.1.0-beta.1
go get github.com/cloudwego/eino-ext/components/model/agenticgemini@v0.2.0-beta.1
go get github.com/cloudwego/eino-ext/components/model/agenticdeepseek@v0.1.0-beta.1
go get github.com/cloudwego/eino-ext/components/model/agenticqwen@v0.1.0-beta.1
go get github.com/cloudwego/eino-ext/components/model/agenticopenai@v0.2.0-beta.1
go get github.com/cloudwego/eino-ext/callbacks/cozeloop@v0.3.0-beta.1

# After stable release, replace with the latest stable version numbers
go get github.com/cloudwego/eino@v0.9.0
go get github.com/cloudwego/eino-ext/components/model/agenticopenai@v0.2.0
go get github.com/cloudwego/eino-ext/components/model/agenticark@v0.2.0
go get github.com/cloudwego/eino-ext/components/model/agenticclaude@v0.1.0
go get github.com/cloudwego/eino-ext/components/model/agenticgemini@v0.2.0
go get github.com/cloudwego/eino-ext/components/model/agenticdeepseek@v0.1.0
go get github.com/cloudwego/eino-ext/components/model/agenticqwen@v0.1.0
go get github.com/cloudwego/eino-ext/components/model/agenticopenai@v0.2.0
go get github.com/cloudwego/eino-ext/callbacks/cozeloop@v0.3.0
```

Check the latest version numbers: [github.com/cloudwego/eino/tags](https://github.com/cloudwego/eino/tags)
