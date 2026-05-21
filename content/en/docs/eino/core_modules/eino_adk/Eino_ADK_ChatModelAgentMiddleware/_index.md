---
Description: ""
date: "2026-05-21"
lastmod: ""
tags: []
title: ChatModelAgentMiddleware
weight: 8
---

`ChatModelAgentMiddleware` is the core interface for customizing the behavior of `ChatModelAgent` (and `DeepAgent` built on top of it). Introduced in v0.8.0, it continues to evolve in subsequent versions.

## Type Conventions

This document uses the default `M = *schema.Message` aliases. The generic raw types are prefixed with `Typed`:

```go
type ChatModelAgentMiddleware     = TypedChatModelAgentMiddleware[*schema.Message]
type BaseChatModelAgentMiddleware = TypedBaseChatModelAgentMiddleware[*schema.Message]
type ChatModelAgentState          = TypedChatModelAgentState[*schema.Message]
type ModelContext                  = TypedModelContext[*schema.Message]
```

When using `*schema.AgenticMessage`, use the `Typed` generic versions directly.

---

## Interface Definition

```go
type ChatModelAgentMiddleware interface {
    // ── Lifecycle Hooks ──

    // BeforeAgent: called once before the agent runs, can modify instruction and tools configuration
    BeforeAgent(ctx context.Context, runCtx *ChatModelAgentContext) (context.Context, *ChatModelAgentContext, error)

    // AfterAgent: called after the agent terminates successfully (final answer or return-directly tool result)
    // Not called on error termination (max iterations, context cancellation, model error)
    AfterAgent(ctx context.Context, state *ChatModelAgentState) (context.Context, error)

    // BeforeModelRewriteState: called before each model call
    // The returned state is persisted; can modify Messages, ToolInfos, DeferredToolInfos
    BeforeModelRewriteState(ctx context.Context, state *ChatModelAgentState, mc *ModelContext) (context.Context, *ChatModelAgentState, error)

    // AfterModelRewriteState: called after each model call
    // The input state contains the model response as the last message
    AfterModelRewriteState(ctx context.Context, state *ChatModelAgentState, mc *ModelContext) (context.Context, *ChatModelAgentState, error)

    // ── Wrappers ──

    WrapInvokableToolCall(ctx context.Context, endpoint InvokableToolCallEndpoint, tCtx *ToolContext) (InvokableToolCallEndpoint, error)
    WrapStreamableToolCall(ctx context.Context, endpoint StreamableToolCallEndpoint, tCtx *ToolContext) (StreamableToolCallEndpoint, error)
    WrapEnhancedInvokableToolCall(ctx context.Context, endpoint EnhancedInvokableToolCallEndpoint, tCtx *ToolContext) (EnhancedInvokableToolCallEndpoint, error)
    WrapEnhancedStreamableToolCall(ctx context.Context, endpoint EnhancedStreamableToolCallEndpoint, tCtx *ToolContext) (EnhancedStreamableToolCallEndpoint, error)

    // WrapModel: wraps the ChatModel. The parameter type is model.BaseModel[M] (not ToolCallingChatModel)
    // The framework handles WithTools binding separately, not going through the user wrapper
    WrapModel(ctx context.Context, m model.BaseModel[M], mc *ModelContext) (model.BaseModel[M], error)
}
```

> 💡
> Embed `*BaseChatModelAgentMiddleware` to get default no-op implementations for all methods — only override the methods you care about.

### AgentMiddleware is Deprecated

> 💡
> The `AgentMiddleware` struct and the `ChatModelAgentConfig.Middlewares` field have been marked as Deprecated and will be removed in a future version. All new code should use `ChatModelAgentMiddleware` (interface-based Handlers).

`AgentMiddleware` is a struct with inherent limitations — users cannot extend methods, and callbacks only return error without propagating context. `ChatModelAgentMiddleware` is an interface:

- Hook methods return `(context.Context, ..., error)`, supporting context propagation
- Wrapper methods propagate modified context through the endpoint chain
- Custom handlers can carry arbitrary internal state

Migration mapping:

<table>
<tr><td>AgentMiddleware Field</td><td>ChatModelAgentMiddleware Replacement</td></tr>
<tr><td><pre>AdditionalInstruction</pre></td><td>Modify <pre>runCtx.Instruction</pre> in <pre>BeforeAgent</pre></td></tr>
<tr><td><pre>AdditionalTools</pre></td><td>Modify <pre>runCtx.Tools</pre> in <pre>BeforeAgent</pre></td></tr>
<tr><td><pre>BeforeChatModel</pre></td><td><pre>BeforeModelRewriteState</pre></td></tr>
<tr><td><pre>AfterChatModel</pre></td><td><pre>AfterModelRewriteState</pre></td></tr>
<tr><td><pre>WrapToolCall</pre></td><td><pre>WrapInvokableToolCall</pre> / <pre>WrapStreamableToolCall</pre> etc.</td></tr>
</table>

In the current version, both can coexist (Handlers execute after Middlewares), but you should migrate as soon as possible.

---

## Context Types

### ChatModelAgentContext

Input to `BeforeAgent`, called once before each Run:

```go
type ChatModelAgentContext struct {
    // Current instruction (includes agent config + framework appended + preceding handler modifications)
    Instruction string

    // Original tool list (includes framework implicit tools like transfer/exit)
    Tools []tool.BaseTool

    // Set of tool names configured to "return directly"
    ReturnDirectly map[string]bool

    // ToolInfo for the model's native tool search capability
    // After being set by a handler, the framework passes it to the model via model.WithToolSearchTool
    ToolSearchTool *schema.ToolInfo
}
```

### ChatModelAgentState

**Persistent state** passed before and after each model call (persists across iterations):

```go
type ChatModelAgentState struct {
    // All messages in the current session
    Messages []*schema.Message

    // Tool definitions passed to the model (via model.WithTools), can be modified in BeforeModelRewriteState
    ToolInfos []*schema.ToolInfo

    // Deferred tool definitions (via model.WithDeferredTools), for the model's native search capability
    // nil when not used
    DeferredToolInfos []*schema.ToolInfo
}
```

> 💡
> The recommended place to modify `ToolInfos` / `DeferredToolInfos` is `BeforeModelRewriteState` — this is the source of truth for tool configuration. Do not modify the tool list in `WrapModel`.

### ModelContext

Context for `WrapModel` and `Before/AfterModelRewriteState`:

```go
type ModelContext struct {
    // Deprecated: use ChatModelAgentState.ToolInfos instead
    Tools []*schema.ToolInfo

    // Model retry configuration
    ModelRetryConfig *ModelRetryConfig

    // Model failover configuration
    ModelFailoverConfig *ModelFailoverConfig[*schema.Message]
}
```

### ToolContext

Metadata for tool wrapping:

```go
type ToolContext struct {
    Name   string // Tool name
    CallID string // Unique identifier for this call
}
```

---

## Tool Call Endpoint Types

Tool wrapping uses function types instead of interfaces. The framework calls the corresponding Wrap method based on which interface the tool implements:

```go
// Standard tools
type InvokableToolCallEndpoint  func(ctx context.Context, argumentsInJSON string, opts ...tool.Option) (string, error)
type StreamableToolCallEndpoint func(ctx context.Context, argumentsInJSON string, opts ...tool.Option) (*schema.StreamReader[string], error)

// Enhanced tools (using ToolArgument/ToolResult)
type EnhancedInvokableToolCallEndpoint  func(ctx context.Context, toolArgument *schema.ToolArgument, opts ...tool.Option) (*schema.ToolResult, error)
type EnhancedStreamableToolCallEndpoint func(ctx context.Context, toolArgument *schema.ToolArgument, opts ...tool.Option) (*schema.StreamReader[*schema.ToolResult], error)
```

> 💡
> Each Wrap method is **only called when the tool implements the corresponding interface**. For example, if a tool only implements `InvokableTool`, only `WrapInvokableToolCall` will be called, not `WrapStreamableToolCall`.

---

## Execution Order

### Model Call Lifecycle (outer to inner)

1. ~~AgentMiddleware.BeforeChatModel~~ (**Deprecated**, will be removed)
2. **ChatModelAgentMiddleware.BeforeModelRewriteState**
3. `failoverModelWrapper` (internal — model failover, if configured)
4. `retryModelWrapper` (internal — failure retry)
5. `eventSenderModelWrapper` preprocessing (internal — prepares event sending)
6. **ChatModelAgentMiddleware.WrapModel** preprocessing (first registered → first executed)
7. `callbackInjectionModelWrapper` (internal)
8. **Model.Generate / Stream**
9. `callbackInjectionModelWrapper` postprocessing
10. **ChatModelAgentMiddleware.WrapModel** postprocessing (first registered → last executed)
11. `eventSenderModelWrapper` postprocessing
12. `retryModelWrapper` postprocessing
13. `failoverModelWrapper` postprocessing
14. **ChatModelAgentMiddleware.AfterModelRewriteState**
15. ~~AgentMiddleware.AfterChatModel~~ (**Deprecated**, will be removed)

### Tool Call Lifecycle (outer to inner)

1. `eventSenderToolHandler` (internal — sends tool result events)
2. `ToolsConfig.ToolCallMiddlewares`
3. ~~AgentMiddleware.WrapToolCall~~ (**Deprecated**, will be removed)
4. `cancelMonitoredToolHandler` (internal — cancel monitoring, only for Streamable/EnhancedStreamable tools)
5. **ChatModelAgentMiddleware.WrapXxxToolCall** (first registered → outermost)
6. `callbackInjectedToolCall` (internal — injects callback)
7. **Tool.InvokableRun / StreamableRun**

## Run-Local Storage API

Store and retrieve key-value pairs during the current agent `Run()`. Values are compatible with interrupt/resume — they are serialized and persisted with checkpoints.

```go
func SetRunLocalValue(ctx context.Context, key string, value any) error
func GetRunLocalValue(ctx context.Context, key string) (any, bool, error)
func DeleteRunLocalValue(ctx context.Context, key string) error
```

> 💡
> Custom types must be registered in `init()` via `schema.RegisterName[T]()` to ensure correct gob serialization. These functions can only be called within `ChatModelAgentMiddleware` callbacks.

### Example: Sharing State Across Callbacks

```go
func init() {
    schema.RegisterName[*ToolStats]("mypackage.ToolStats")
}

type ToolStats struct {
    Count int
    Name  string
}

type MyMiddleware struct {
    *adk.BaseChatModelAgentMiddleware
}

// Record stats after tool call
func (m *MyMiddleware) WrapInvokableToolCall(ctx context.Context, endpoint adk.InvokableToolCallEndpoint, tCtx *adk.ToolContext) (adk.InvokableToolCallEndpoint, error) {
    return func(ctx context.Context, args string, opts ...tool.Option) (string, error) {
        result, err := endpoint(ctx, args, opts...)

        _ = adk.SetRunLocalValue(ctx, "last_tool", &ToolStats{Count: 1, Name: tCtx.Name})
        return result, err
    }, nil
}

// Read stats after model call
func (m *MyMiddleware) AfterModelRewriteState(ctx context.Context, state *adk.ChatModelAgentState, mc *adk.ModelContext) (context.Context, *adk.ChatModelAgentState, error) {
    if val, found, _ := adk.GetRunLocalValue(ctx, "last_tool"); found {
        if stats, ok := val.(*ToolStats); ok {
            log.Printf("Last tool: %s (count=%d)", stats.Name, stats.Count)
        }
    }
    return ctx, state, nil
}
```

---

## SendEvent API

Send custom `AgentEvent` to the event stream during agent execution, which callers can receive when iterating over the event stream:

```go
func SendEvent(ctx context.Context, event *AgentEvent) error
```

Can only be called within `ChatModelAgentMiddleware` callbacks.

---

## State Type

> 💡
> `State` is kept exported only for checkpoint backward compatibility. **Do not use it directly** — use `ChatModelAgentState` in `ChatModelAgentMiddleware` callbacks, and use `SetRunLocalValue/GetRunLocalValue` instead of the original `State.Extra`. The `compose.ProcessState[*State]` usage will stop working in v1.0.0.

---

## Migration Guide

### Migrating from compose.ProcessState[*State]

**Before:**

```go
compose.ProcessState(ctx, func(_ context.Context, st *adk.State) error {
    st.Extra["myKey"] = myValue
    return nil
})
```

**After:**

```go
// Write
if err := adk.SetRunLocalValue(ctx, "myKey", myValue); err != nil {
    return ctx, state, err
}

// Read
if val, found, err := adk.GetRunLocalValue(ctx, "myKey"); err == nil && found {
    // use val
}
```

### Adapting to AfterAgent (new in v0.9)

`AfterAgent` is called after the agent **terminates successfully** (final answer or return-directly tool result), and can be used for post-processing:

```go
func (m *MyMiddleware) AfterAgent(ctx context.Context, state *adk.ChatModelAgentState) (context.Context, error) {
    log.Printf("Agent completed, %d messages total", len(state.Messages))
    // Audit, statistics, cleanup, etc.
    return ctx, nil
}
```

> 💡
> `AfterAgent` is called in registration order (same as `BeforeAgent`). If any handler returns an error, subsequent handlers are not called (fail-fast), and the error is sent to the event stream.

### Adapting to ToolInfos / DeferredToolInfos (new in v0.9)

`ChatModelAgentState` now has `ToolInfos` and `DeferredToolInfos` fields, replacing `ModelContext.Tools` as the source of truth for tool configuration:

```go
func (m *MyMiddleware) BeforeModelRewriteState(ctx context.Context, state *adk.ChatModelAgentState, mc *adk.ModelContext) (context.Context, *adk.ChatModelAgentState, error) {
    // Dynamically filter tools
    filtered := make([]*schema.ToolInfo, 0, len(state.ToolInfos))
    for _, t := range state.ToolInfos {
        if shouldInclude(t.Name) {
            filtered = append(filtered, t)
        }
    }
    state.ToolInfos = filtered
    return ctx, state, nil
}
```
