---
Description: ""
date: "2026-05-17"
lastmod: ""
tags: []
title: Get Started with Eino ADK in 5 Minutes
weight: 9
---

This article is for developers already familiar with Eino, focusing on the most important autonomous decision-making primitive in ADK: **ChatModelAgent** and its runtime enhancement mechanism **ChatModelAgentMiddleware**.

## Understanding ChatModelAgent

When we talk about "Agent," we almost always mean: an entity powered by a large model as its core, equipped with tools, capable of autonomous decision-making and solving complex real-world problems. `ChatModelAgent` is Eino ADK's direct implementation of this concept.

**ChatModelAgent = A ReAct Agent that uses ChatModel as the decision-maker, Tools as the action space, and tool feedback plus history as context for the next decision.**

Four key components:

1. **ChatModel**: The large model, responsible for reasoning and decision-making.
2. **Tools**: The tool collection, defining the range of actions the Agent can take.
3. **Feedback**: Tool execution results feed back into the model context, becoming the basis for the next decision.
4. **History**: Complete preservation of the reasoning trajectory, tool calls, and tool results throughout the problem-solving process.

Therefore, `ChatModelAgent` is not a single model call, but a sustained problem-solving process.

## ChatModelAgent's Execution Structure: ReAct Loop

`ChatModelAgent`'s core capability is **autonomous decision-making** — within a single `Run`, the model can repeatedly reason, act, and receive feedback until the problem is solved. The execution structure supporting this capability is the ReAct Loop.

Autonomous decision-making requires four elements to coexist:

1. **Decision-maker (ChatModel)**: Each round, based on current context, determines what to do next.
2. **Action space (Tools)**: Defines the concrete actions the Agent can take.
3. **Feedback signal (Tool Feedback)**: Action results are injected into the context, becoming the basis for subsequent decisions — this enables the Agent to correct course based on actual execution results rather than guessing everything at once.
4. **Accumulated context (History)**: Complete preservation of reasoning trajectory, tool calls, and tool results. Each round, the model doesn't see an isolated single query but the complete problem-solving process from start to current state.

All four are indispensable: without a decision-maker there's no reasoning, without action space there's no execution, without feedback there's no correction, without accumulated context there's no informed judgment based on history.

<a href="/img/eino/HAz4wb8f6h4XSOb7yUVc2CkUnAg.png" target="_blank"><img src="/img/eino/HAz4wb8f6h4XSOb7yUVc2CkUnAg.png" width="100%" /></a>

Key characteristic: **Accumulated context-driven progressive decision-making**. Each loop iteration doesn't start from scratch but continues on top of the complete trajectory of all prior reasoning and actions. Every model decision is made based on a continuously growing problem-solving context, enabling the Agent to handle complex tasks requiring multi-step reasoning, trial-and-error, and correction.

## What Makes Your ChatModelAgent Different

The ReAct Loop structure is fixed. So what makes **your** ChatModelAgent different from others, tailored to your specific problem?

Four dimensions:

1. **ChatModel** — Which model makes decisions.
2. **Instruction** — System instructions: role definition, behavioral constraints, few-shot examples.
3. **Tools** — Tool collection: determines what the Agent can do.
4. **Middleware (ChatModelAgentMiddleware)** — Inject behavior at specific lifecycle points of the ReAct Loop: intercept, modify, and enhance inputs and outputs within the loop.

The first three define what the Agent "is" — decision capability, role constraints, action scope.

Middleware defines how the Agent "runs" — it doesn't change the Loop's structure (reason → act → feedback always remains), but controls the specific runtime behavior of the loop. For example: compressing context before model calls, dynamically injecting tools before running, performing permission checks during tool calls, retrying or switching to backup models on failure. These are all runtime enhancements at specific loop points.

## Middleware: Injecting Behavior into the ReAct Loop

When building a ChatModelAgent, you'll encounter these typical problems:

- **Agent needs to read/write files, execute commands?** → Need to inject a set of general-purpose tools before running.
- **Agent needs to reuse predefined instructions and knowledge?** → Need to package reusable capabilities as Skills, loaded on demand.
- **Context growing too long, exceeding model window?** → Need to automatically compress history before each model call.
- **Too many tools, stuffing all into prompt dilutes attention?** → Need to search and load tools on demand.
- **Model occasionally fails or returns garbage?** → Need automatic retry or backup model switching.

The common thread: they don't need to change the ReAct Loop's structure, only intercept and enhance at specific points in the loop. This is what Middleware does.

Corresponding built-in Middleware:

<table>
<tr><td>Scenario</td><td>Middleware</td><td>What It Does</td></tr>
<tr><td>Need filesystem capabilities</td><td><strong>FileSystem</strong></td><td>Injects ls/read/write/edit/grep/execute tools before running</td></tr>
<tr><td>Reuse predefined capabilities</td><td><strong>Skill</strong></td><td>Packages instructions, knowledge, tools as skill units loadable on demand</td></tr>
<tr><td>Context exceeds window</td><td><strong>Reduction / Summarization</strong></td><td>Compresses messages and tool results before model calls</td></tr>
<tr><td>Too many tools</td><td><strong>ToolSearch</strong></td><td>Searches and loads Tools on demand rather than exposing all at once</td></tr>
<tr><td>Unstable model calls</td><td><strong>ModelRetry / ModelFailover</strong></td><td>Per-model-call retry / failover switching</td></tr>
</table>

Each Middleware implementation injects at a specific hook point in the ReAct Loop. The diagram below shows where `ChatModelAgentMiddleware` hooks are positioned in the loop:

<a href="/img/eino/RlIuwflSQh1gzlb7eMkcarFenbe.png" target="_blank"><img src="/img/eino/RlIuwflSQh1gzlb7eMkcarFenbe.png" width="100%" /></a>

Hook point summary:

<table>
<tr><td>Hook Point</td><td>Timing</td><td>Typical Use</td></tr>
<tr><td><pre>BeforeAgent</pre></td><td>Before Agent runs (once only)</td><td>Enhance Instruction, inject Tools</td></tr>
<tr><td><pre>BeforeModelRewriteState</pre></td><td>Before each model call</td><td>Modify Messages / ToolInfos</td></tr>
<tr><td><pre>AfterModelRewriteState</pre></td><td>After each model call</td><td>Modify model response or patch state</td></tr>
<tr><td><pre>WrapModel</pre></td><td>Per-model-call level</td><td>Retry, failover, rewrite model returns</td></tr>
<tr><td><pre>WrapToolCall</pre></td><td>Per-tool-call level</td><td>Permissions, security, output rewriting</td></tr>
<tr><td><pre>AfterAgent</pre></td><td>After Agent completes successfully</td><td>Post-processing, state cleanup</td></tr>
</table>

See the appendix at the end for a complete Middleware quick reference.

## Quick Start: Create and Run a ChatModelAgent

`Runner` is the entry point for executing an Agent. It transforms a user request into a single Agent run, handling per-run configuration, event stream output, streaming toggles, and runtime capabilities like checkpoint/resume. The minimal usage is: put a `ChatModelAgent` into `RunnerConfig`, then call `Query` or `Run`.

The following example shows how to create a minimal ChatModelAgent and execute it via Runner:

```go
package main

import (
    "context"
    "fmt"
    "log"

    "github.com/cloudwego/eino-ext/components/model/ark"
    "github.com/cloudwego/eino/adk"
    "github.com/cloudwego/eino/compose"
    "github.com/cloudwego/eino/components/tool"
)

func main() {
    ctx := context.Background()

    // 1. Create ChatModel
    chatModel, err := ark.NewChatModel(ctx, &ark.ChatModelConfig{
        Model:  "doubao-seed-1-8-251228",
        APIKey: "your_api_key", // Replace with your API Key
    })
    if err != nil {
        log.Fatal(err)
    }

    // 2. Create ChatModelAgent
    agent, err := adk.NewChatModelAgent(ctx, &adk.ChatModelAgentConfig{
        Name:        "my-assistant",
        Description: "An assistant that can answer questions using tools.",
        Instruction: "You are a helpful assistant. Please answer user questions using available tools.",
        Model:       chatModel,
        ToolsConfig: adk.ToolsConfig{
            ToolsNodeConfig: compose.ToolsNodeConfig{
                Tools: []tool.BaseTool{
                    // Register your tools, e.g., webSearchTool
                },
            },
        },
        // Handlers: []adk.ChatModelAgentMiddleware{...}, // Register Middleware
    })
    if err != nil {
        log.Fatal(err)
    }

    // 3. Execute Agent via Runner
    runner := adk.NewRunner(ctx, adk.RunnerConfig{
        Agent:           agent,
        EnableStreaming: true,
    })

    // 4. Send user request and consume event stream
    iter := runner.Query(ctx, "Help me search for today's news")
    for {
        event, ok := iter.Next()
        if !ok {
            break
        }
        fmt.Println(event)
    }
}
```

Core flow: `NewChatModelAgent` → `NewRunner` → `Runner.Query/Run` → consume `AsyncIterator` event stream.

For more basic examples, see: [Eino: Quick Start](/docs/eino/quick_start).

## Further Reading: DeepAgents

DeepAgents is a pre-built ChatModelAgent whose core value lies in two preset Middleware:

- **WriteTodos (PlanTask)**: Enables the main Agent to explicitly plan a task list before execution and continuously track progress during execution. Complex problems no longer rely on the model "thinking through everything at once" but instead decompose first, then progress step by step.
- **TaskTool**: Enables the main Agent to delegate subtasks to sub-Agents for independent execution, with results summarized back to the main loop. This allows a single Agent's capability boundary to be extended through composition.

Additionally, DeepAgents comes with preset system prompts and optional FileSystem Middleware, ready out-of-the-box for scenarios requiring task planning and multi-Agent collaboration.

```
DeepAgents = ChatModelAgent
           + WriteTodos (task planning and tracking)
           + TaskTool (subtask delegation)
           + Optional FileSystem
           + Preset system prompts
```

Further reading:

- Eino ADK Deep Agents complete guide: [Eino ADK: DeepAgents](/docs/eino/core_modules/eino_adk/agent_implementation/deepagents)
- DeepAgents examples: [eino-examples/adk/multiagent/deep at main · cloudwego/eino-examples](https://github.com/cloudwego/eino-examples/tree/main/adk/multiagent/deep)

## Further Reading: Why Not Continue Using flow/react?

Back to first principles: Graph and Agent are two fundamentally different AI application paradigms.

- **Graph**'s core is **determinism**: Developers predefine the topology, and node transitions are determined at compile time. Input is structured, output is predictable.
- **Agent**'s core is **autonomy**: The LLM dynamically decides the next action at runtime, execution paths are unpredictable, and output is a full-process event stream.

`flow/react` is essentially using Graph's approach to "simulate" an Agent — unrolling the ReAct reasoning loop into static nodes and edges. This works, but is fundamentally a mismatch: using deterministic orchestration to carry dynamic decision-making. As Agent complexity grows, this mismatch creates systemic problems:

1. **Deliverable mismatch**: Graph targets "final results," while Agent's deliverable is the full process (reasoning trajectory, intermediate tool calls, state changes). Using Graph for Agent means intermediate process can only be extracted through side channels like Callbacks — feasible, but a patch.
2. **Execution model mismatch**: Graph is a synchronous execution model, while Agents are naturally asynchronous long-running processes. Event stream output, checkpoint/resume, interrupt recovery, and other runtime capabilities need framework-level unified management at the Agent dimension, not scattered across Graph node callbacks.
3. **Extension point mismatch**: Agent runtime enhancements (context compression, dynamic tool loading, model retry, security control) are fundamentally interception and injection into the decision loop. In Graph, these capabilities have no unified mount point and are scattered across various nodes or edges; in ChatModelAgent, they have clear lifecycle hooks (Middleware).

Therefore, flow/react isn't deprecated but returns to its best-fit position: **deterministic process orchestration**. When the core problem is "autonomous decision-making + runtime enhancement," the correct abstraction is `ChatModelAgent + ChatModelAgentMiddleware`.

Further reading:

- Agent or Graph? AI Application Route Analysis: [Agent or Graph? AI Application Route Analysis](/docs/eino/overview/graph_or_agent)

<a href="/img/eino/Xs38beDNAobevkx0epfcjkCnnFb.png" target="_blank"><img src="/img/eino/Xs38beDNAobevkx0epfcjkCnnFb.png" width="100%" /></a>

## Appendix: Middleware Quick Reference

### Instance Overview

<table>
<tr><td>Middleware</td><td>Description</td></tr>
<tr><td><strong>Reduction</strong></td><td>Truncates overly long tool output / writes to filesystem, preventing token limit exceeded</td></tr>
<tr><td><strong>Summarization</strong></td><td>Historical message summary compression</td></tr>
<tr><td><strong>Skill</strong></td><td>Reusable instructions/knowledge exposed as Tools, Agent loads on demand</td></tr>
<tr><td><strong>FileSystem</strong></td><td>ls/read/write/edit/glob/grep/execute file operation tool set</td></tr>
<tr><td><strong>ToolSearch</strong></td><td><pre>tool_search</pre> meta-tool, searches and loads Tools on demand (reduces resident tool list footprint)</td></tr>
<tr><td><strong>PatchToolCall</strong></td><td>Patches dangling tool calls in message history (missing tool results)</td></tr>
<tr><td><strong>SafeTool</strong></td><td>WrapToolCall-level interception of tool execution errors, converting to readable text returned to the model, allowing Agent to self-correct rather than abort</td></tr>
<tr><td><strong>ModelRetry</strong></td><td>Retries on model call failure per configured strategy [built-in config]</td></tr>
<tr><td><strong>ModelFailover</strong></td><td>Switches to backup model on model call failure [built-in config]</td></tr>
<tr><td><strong>AgentsMD</strong></td><td>Injects Agents.md knowledge file into model context, improving context quality</td></tr>
<tr><td><strong>PlanTask</strong></td><td>Persistent task management tool set (create/get/update/list), supports dependency tracking</td></tr>
<tr><td><strong>WriteTodos</strong></td><td>Lightweight TODO list tool, Agent can create and track structured to-do items [DeepAgent built-in]</td></tr>
<tr><td><strong>TaskTool</strong></td><td>Sub-Agent delegation tool, main Agent uses it to dispatch subtasks to sub-Agents for independent execution [DeepAgent built-in]</td></tr>
<tr><td><strong>Permission</strong></td><td>Tool call permission control [WIP]</td></tr>
</table>

> Note: ModelRetry / ModelFailover are built-in fields of `ChatModelAgentConfig` (`ModelRetryConfig` / `ModelFailoverConfig`) in code, conceptually corresponding to the `WrapModel` hook. SafeTool is an example pattern (see ChatWithEino ch05), implemented as user-defined Middleware. WriteTodos / TaskTool are DeepAgent built-ins, not exported separately. Permission is a planned capability.

### Categories

<table>
<tr><td>Category</td><td>Problem Solved</td><td>Includes</td></tr>
<tr><td><strong>Extend general Tools</strong></td><td>Give Agent more capabilities</td><td>FileSystem, Skill, ToolSearch, PlanTask, WriteTodos, TaskTool</td></tr>
<tr><td><strong>Handle errors during ReAct process</strong></td><td>Improve reliability</td><td>ModelRetry, ModelFailover, SafeTool, PatchToolCall</td></tr>
<tr><td><strong>Keep context window within limits</strong></td><td>Prevent token overflow</td><td>Reduction, Summarization, ToolSearch</td></tr>
<tr><td><strong>Security and permissions</strong></td><td>Constrain Agent behavior</td><td>Permission</td></tr>
<tr><td><strong>Improve context content quality</strong></td><td>Help model see better context</td><td>Skill, AgentsMD</td></tr>
</table>

ToolSearch spans two categories: it is both "extend Tools" (providing on-demand tool discovery) and "keep context window within limits" (avoiding loading too many tool descriptions at once).

Further reading:

- ChatModelAgent Middleware detailed guide: [Eino ADK: ChatModelAgentMiddleware](/docs/eino/core_modules/eino_adk/eino_adk_chatmodelagentmiddleware)
