---
Description: ""
date: "2026-05-17"
lastmod: ""
tags: []
title: Overview
weight: 2
---

# What is Eino ADK?

Eino ADK is a Go-based Agent development framework, providing:

- **ChatModelAgent**: A ReAct Agent with LLM as the decision-maker, supporting tool calls, autonomous reasoning, and runtime enhancement (Middleware)
- **Workflow Agents**: Deterministic orchestration primitives (Sequential / Loop / Parallel)
- **Runner / TurnLoop**: Agent execution entry, supporting event streams, checkpoint/resume, and multi-turn preemption
- **Multi-Agent Collaboration**: AgentAsTool (recommended), Workflow composition

Widely applicable, model-agnostic, and deployment-agnostic.

# ADK Architecture

## Agent Interface

All ADK functionality revolves around the `Agent` interface:

```go
type Agent interface {
    Name(ctx context.Context) string
    Description(ctx context.Context) string
    Run(ctx context.Context, input *AgentInput, options ...AgentRunOption) *AsyncIterator[*AgentEvent]
}
```

Semantics of `Run`:

1. Get task information from `AgentInput` and Context
2. Execute the task asynchronously, writing produced events to `AsyncIterator`
3. Return the Iterator immediately after starting the async task (Future pattern)

## ChatModelAgent

The core implementation of ADK. Uses a ChatModel as the decision-maker and autonomously drives problem-solving through a ReAct Loop.

**ChatModelAgent = ChatModel + Tools + ReAct Loop + Middleware**

For detailed introduction, see: [Eino ADK: ChatModelAgent Introduction](/docs/eino/overview/eino_adk_quickstart)

## Multi-Agent Collaboration

> 💡
> Recommended approach: **AgentAsTool** — Convert a sub-Agent to a Tool, and the parent Agent calls it via ToolCall and obtains the result. This is the most flexible and composable collaboration pattern.

<table>
<tr><td>Collaboration Approach</td><td>Mechanism</td><td>Applicable Scenarios</td></tr>
<tr><td><strong>AgentAsTool</strong> (Recommended)</td><td>Sub-Agent wrapped as Tool, parent Agent autonomously decides whether to call</td><td>Delegating subtasks, capability composition</td></tr>
<tr><td><strong>Workflow</strong></td><td>Sequential / Loop / Parallel deterministic orchestration</td><td>Multi-step tasks with fixed processes</td></tr>
</table>

See: [Agent Collaboration](/docs/eino/core_modules/eino_adk/agent_collaboration)

## Runner

Runner is the execution entry for Agents. The following features are only available when running through Runner:

- **Event stream output**: Query/Run → AsyncIterator[AgentEvent]
- **Checkpoint / Resume**: Persist running state, support interrupt recovery
- **TurnLoop**: Multi-turn runtime, Push/Preempt/Stop

```go
runner := adk.NewRunner(ctx, adk.RunnerConfig{
    Agent:           agent,
    EnableStreaming: true,
    CheckPointStore: store, // Optional
})

iter := runner.Query(ctx, "Your question")
```

See: [Agent Runner and Extension](/docs/eino/core_modules/eino_adk/agent_extension) | [Agent Cancel and TurnLoop](/docs/eino/core_modules/eino_adk/agent_cancel_and_turnloop_quickstart)
