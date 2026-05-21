---
Description: ""
date: "2026-05-19"
lastmod: ""
tags: []
title: Quickstart
weight: 1
---

# Installation

Eino ADK is available since v0.5.0, with v0.9.0 as the current recommended version:

```go
go get github.com/cloudwego/eino@latest
```

# Core Concepts

**Eino ADK** is a Go-based Agent development framework. The core primitive is **ChatModelAgent** — an intelligent agent that uses a ChatModel as its decision-maker, Tools as its action space, and autonomously drives problem-solving through a ReAct Loop.

> 💡
> If you only read one document, read: [Eino ADK: ChatModelAgent Introduction](/docs/eino/overview/eino_adk_quickstart)

## Component Map

<table>
<tr><td>Component</td><td>Responsibility</td><td>Documentation</td></tr>
<tr><td><strong>ChatModelAgent</strong></td><td>ReAct Loop: Reasoning → Action → Feedback, autonomous decision-making</td><td><a href="/docs/eino/overview/eino_adk_quickstart">ChatModelAgent Introduction</a></td></tr>
<tr><td><strong>Middleware</strong></td><td>Inject behavior at lifecycle points of the ReAct Loop (compression, search, retry, etc.)</td><td><a href="/docs/eino/core_modules/eino_adk/eino_adk_chatmodelagentmiddleware">ChatModelAgentMiddleware</a></td></tr>
<tr><td><strong>Runner</strong></td><td>Single Agent run entry: Query / Run → event stream</td><td><a href="/docs/eino/core_modules/eino_adk/agent_extension">Agent Runner and Extension</a></td></tr>
<tr><td><strong>TurnLoop</strong></td><td>Multi-turn runtime: Push / Preempt / Stop + declarative checkpoint/resume</td><td><a href="/docs/eino/core_modules/eino_adk/agent_cancel_and_turnloop_quickstart">Agent Cancel and TurnLoop</a></td></tr>
<tr><td><strong>DeepAgents</strong></td><td>Pre-built Agent: task planning (PlanTask) + subtask delegation (TaskTool)</td><td><a href="/docs/eino/core_modules/eino_adk/agent_implementation/deepagents">DeepAgents</a></td></tr>
</table>

## Other Agent Types

Besides ChatModelAgent, ADK also provides deterministic orchestration primitives:

- **Workflow Agents**: Sequential / Loop / Parallel Agent, for structured orchestration of predefined processes.
- **Custom Agent**: Implement the `Agent` interface to integrate with the framework.

> 💡
> Graph (deterministic orchestration) and Agent (autonomous decision-making) are two different forms of AI applications. When the core problem is "autonomous decision-making + runtime enhancement", ChatModelAgent is recommended. See "Why not continue using flow/react" in the ChatModelAgent Introduction.

# Examples

[eino-examples/adk](https://github.com/cloudwego/eino-examples/tree/main/adk) provides complete ADK example code:

- **ChatModelAgent Intro**: [chatmodel](https://github.com/cloudwego/eino-examples/tree/main/adk/intro/chatmodel) — Book recommendation Agent with interrupt and resume
- **DeepAgents**: [deep](https://github.com/cloudwego/eino-examples/tree/main/adk/multiagent/deep) — Task planning + subtask delegation
- **Workflow**: [sequential](https://github.com/cloudwego/eino-examples/tree/main/adk/intro/workflow/sequential) / [loop](https://github.com/cloudwego/eino-examples/tree/main/adk/intro/workflow/loop) / [parallel](https://github.com/cloudwego/eino-examples/tree/main/adk/intro/workflow/parallel)
- **Multi-Agent**: [supervisor](https://github.com/cloudwego/eino-examples/tree/main/adk/multiagent/supervisor) / [plan-execute](https://github.com/cloudwego/eino-examples/tree/main/adk/multiagent/plan-execute-replan)

# What's Next
