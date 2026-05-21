---
Description: ""
date: "2026-05-17"
lastmod: ""
tags: []
title: Agent Collaboration
weight: 4
---

# Multi-Agent Collaboration

Eino ADK provides two main Agent collaboration approaches:

## AgentAsTool (Recommended)

Wraps a sub-Agent as a Tool, allowing the parent Agent to autonomously decide when to call it via ToolCall. The sub-Agent executes independently, and results are returned to the parent Agent's context.

This is the most flexible and composable collaboration pattern:

- The parent Agent retains control and can continue reasoning based on sub-Agent results
- The sub-Agent receives an independent task description and does not inherit the parent Agent's full conversation history
- Multiple sub-Agents can be called in parallel

```go
import (
    "github.com/cloudwego/eino/adk"
    "github.com/cloudwego/eino/compose"
    "github.com/cloudwego/eino/components/tool"
)

// Create sub-Agent
subAgent, _ := adk.NewChatModelAgent(ctx, &adk.ChatModelAgentConfig{
    Name:        "researcher",
    Description: "Search and summarize relevant information",
    Instruction: "You are a research assistant...",
    Model:       chatModel,
    ToolsConfig: adk.ToolsConfig{
        ToolsNodeConfig: compose.ToolsNodeConfig{
            Tools: []tool.BaseTool{searchTool},
        },
    },
})

// Wrap as Tool
agentTool := adk.NewAgentTool(ctx, subAgent)

// Parent Agent registers sub-Agent Tool
parentAgent, _ := adk.NewChatModelAgent(ctx, &adk.ChatModelAgentConfig{
    Name:        "coordinator",
    Description: "Main Agent that coordinates tasks",
    Instruction: "You are a task coordinator...",
    Model:       chatModel,
    ToolsConfig: adk.ToolsConfig{
        ToolsNodeConfig: compose.ToolsNodeConfig{
            Tools: []tool.BaseTool{agentTool},
        },
    },
})
```

### AgentTool Options

<table>
<tr><td>Option</td><td>Description</td></tr>
<tr><td><pre>WithFullChatHistoryAsInput()</pre></td><td>Pass the parent Agent's full conversation history as sub-Agent input (by default only the model-generated request parameters are passed)</td></tr>
<tr><td><pre>WithAgentInputSchema(schema)</pre></td><td>Customize the sub-Agent's input schema</td></tr>
</table>

### Event Stream Pass-through

When `ToolsConfig.EmitInternalEvents = true`, the sub-Agent's events are passed through in real-time to the parent Agent's event stream, allowing end users to see the sub-Agent's intermediate process.

> 💡
> Pass-through events do not affect the parent Agent's state or checkpoint, and are only for user display. The only exception is the Interrupted action, which propagates across boundaries via CompositeInterrupt to support interrupt recovery.

### Pre-built Example: DeepAgents

[DeepAgents](/docs/eino/core_modules/eino_adk/agent_implementation/deepagents) is a best practice of the AgentAsTool pattern: the main Agent delegates subtasks to sub-Agents via **TaskTool**, combined with **WriteTodos** for task planning and progress tracking.

## Workflow Agents

Deterministic orchestration for multi-step tasks with fixed processes:

<table>
<tr><td>Type</td><td>Description</td><td>Constructor</td></tr>
<tr><td><strong>Sequential</strong></td><td>Executes sub-Agents sequentially in array order</td><td><pre>adk.NewSequentialAgent</pre></td></tr>
<tr><td><strong>Parallel</strong></td><td>Executes all sub-Agents concurrently, finishes when all complete</td><td><pre>adk.NewParallelAgent</pre></td></tr>
<tr><td><strong>Loop</strong></td><td>Loops execution of sub-Agent sequence until BreakLoop or MaxIterations exceeded</td><td><pre>adk.NewLoopAgent</pre></td></tr>
</table>

Context is passed between Workflow Agents via Transfer: the upstream Agent's output is automatically appended to the downstream Agent's input Messages.

# Context Passing

## SessionValues

Global KV store across Agents, concurrently safe for read/write by any Agent within a single run:

```go
// Read/write API
adk.AddSessionValue(ctx, "key", value)
val, ok := adk.GetSessionValue(ctx, "key")
adk.AddSessionValues(ctx, map[string]any{"k1": v1, "k2": v2})
all := adk.GetSessionValues(ctx)
```

> 💡
> SessionValues are implemented based on Context, and Runner reinitializes the Context at runtime. To inject data before running, use the `WithSessionValues` Option:

```go
iter := runner.Run(ctx, messages,
    adk.WithSessionValues(map[string]any{
        "user_id": "123",
    }),
)
```
