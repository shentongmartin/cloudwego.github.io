---
Description: ""
date: "2026-05-17"
lastmod: ""
tags: []
title: Agent Runner and Extension
weight: 6
---

# Runner

Runner is the execution entry for Agents, responsible for managing Agent lifecycle, context initialization, Checkpoint persistence, and interrupt recovery. **Any Agent should be run through Runner.**

## Basic Usage

```go
import "github.com/cloudwego/eino/adk"

// Create Runner
runner := adk.NewRunner(ctx, adk.RunnerConfig{
    Agent:           agent,
    EnableStreaming: true,
    CheckPointStore: store, // Optional, required for interrupt recovery
})

// Method 1: Query — directly send a user question
iter := runner.Query(ctx, "Help me search today's news")

// Method 2: Run — pass in complete Messages
iter := runner.Run(ctx, []*schema.Message{
    schema.UserMessage("Hello"),
}, adk.WithSessionValues(map[string]any{"user": "alice"}))

// Consume event stream
for {
    event, ok := iter.Next()
    if !ok {
        break
    }
    // handle event
}
```

## Generic Support

```go
type TypedRunner[M MessageType] struct { ... }
type Runner = TypedRunner[*schema.Message]

func NewTypedRunner[M MessageType](conf TypedRunnerConfig[M]) *TypedRunner[M]
```

The `*schema.AgenticMessage` path uses `NewTypedRunner` for construction.

## Interrupt & Resume

An Agent can proactively interrupt during execution. Runner automatically saves state (requires `CheckPointStore` configuration), and can resume from the breakpoint later.

### Interrupt

An Agent triggers an interrupt by producing an event containing `Interrupted`:

```go
gen.Send(&adk.AgentEvent{
    Action: &adk.AgentAction{
        Interrupted: &adk.InterruptInfo{Data: myData},
    },
})
```

### State Persistence

After Runner captures the interrupt, it stores the running state (input, conversation history, InterruptInfo) into `CheckPointStore` keyed by CheckPointID:

```go
type CheckPointStore interface {
    Set(ctx context.Context, key string, value []byte) error
    Get(ctx context.Context, key string) ([]byte, bool, error)
}
```

Pass the CheckPointID via Option when calling:

```go
iter := runner.Run(ctx, messages, adk.WithCheckPointID("cp-123"))
```

> 💡
> ADK uses gob to serialize running state. Custom types need to be registered in advance via gob.RegisterName. Framework built-in types are automatically registered.

### Resume

```go
// Simple resume: implicitly resumes all interrupt points
iter, err := runner.Resume(ctx, "cp-123")

// Precise resume: specify target and data
iter, err := runner.ResumeWithParams(ctx, "cp-123", &adk.ResumeParams{
    Targets: map[string]any{
        "agent-address": resumeData,
    },
})
```

Resume requires the interrupted Agent to implement the `ResumableAgent` interface:

```go
type TypedResumableAgent[M MessageType] interface {
    TypedAgent[M]
    Resume(ctx context.Context, info *ResumeInfo, opts ...AgentRunOption) *AsyncIterator[*TypedAgentEvent[M]]
}
```

# Multi-Turn Runtime: TurnLoop

For scenarios requiring multi-turn interaction (chat applications, continuous conversations), ADK provides the `TurnLoop` runtime:

- **Push-based event loop**: Push new messages to trigger Agent execution
- **Preempt**: When a user sends a new message while the Agent is running, it can cancel the current run
- **Stop**: Stop the event loop
- **Declarative Checkpoint/Resume**: TurnLoop automatically manages input bookkeeping; the application layer only needs to declare the recovery strategy

See: [Agent Cancel and TurnLoop Quickstart](/docs/eino/core_modules/eino_adk/agent_cancel_and_turnloop_quickstart)

# Agent Cancel

Runtime cancellation capability added in v0.9, supporting:

- **CancelMode bitmask combination**: `CancelModelStream | CancelToolCalls`
- **CancelHandle.Wait()**: Wait for cancellation to complete
- **Integration with TurnLoop**: Automatically triggers Cancel on Preempt

See: [Agent Cancel and TurnLoop Quickstart](/docs/eino/core_modules/eino_adk/agent_cancel_and_turnloop_quickstart)
