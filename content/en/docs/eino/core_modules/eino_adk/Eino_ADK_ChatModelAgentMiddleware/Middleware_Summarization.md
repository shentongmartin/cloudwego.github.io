---
Description: ""
date: "2026-05-17"
lastmod: ""
tags: []
title: Summarization
weight: 4
---

> 💡
> This middleware was introduced in v0.8.0. Package path: `github.com/cloudwego/eino/adk/middlewares/summarization`

## Overview

The Summarization middleware automatically calls a summary model to compress conversation history when the conversation token count exceeds a threshold, keeping long conversations coherent within the model's context window. The middleware hooks into `BeforeModelRewriteState`, checking trigger conditions before each model call. When triggered, it executes: counting → summary generation (with retry/failover) → post-processing → state replacement.

## Generic System

All core types and functions in this package provide both a **Typed generic version** (`M adk.MessageType`) and a **non-generic alias** (fixed to `*schema.Message`).

<table>
<tr><td>Generic Version</td><td>Non-generic Alias (= Typed\[*schema.Message\])</td></tr>
<tr><td><pre>TypedConfig[M]</pre></td><td><pre>Config</pre></td></tr>
<tr><td><pre>NewTyped[M](ctx, *TypedConfig[M])</pre></td><td><pre>New(ctx, *Config)</pre></td></tr>
<tr><td><pre>TypedTokenCounterFunc[M]</pre></td><td><pre>TokenCounterFunc</pre></td></tr>
<tr><td><pre>TypedGenModelInputFunc[M]</pre></td><td><pre>GenModelInputFunc</pre></td></tr>
<tr><td><pre>TypedGetFailoverModelFunc[M]</pre></td><td><pre>GetFailoverModelFunc</pre></td></tr>
<tr><td><pre>TypedFinalizeFunc[M]</pre></td><td><pre>FinalizeFunc</pre></td></tr>
<tr><td><pre>TypedCallbackFunc[M]</pre></td><td><pre>CallbackFunc</pre></td></tr>
<tr><td><pre>TypedUserMessageFilterFunc[M]</pre></td><td><pre>UserMessageFilterFunc</pre></td></tr>
<tr><td><pre>TypedPreserveUserMessages[M]</pre></td><td><pre>PreserveUserMessages</pre></td></tr>
<tr><td><pre>TypedRetryConfig[M]</pre></td><td><pre>RetryConfig</pre></td></tr>
<tr><td><pre>TypedFailoverConfig[M]</pre></td><td><pre>FailoverConfig</pre></td></tr>
<tr><td><pre>TypedFailoverContext[M]</pre></td><td><pre>FailoverContext</pre></td></tr>
<tr><td><pre>TypedFinalizerBuilder[M]</pre></td><td><pre>FinalizerBuilder</pre></td></tr>
</table>

Unless otherwise noted, type signatures in this document use the generic form `M`. When using non-generic aliases, `M` = `*schema.Message`.

### Constructors

```go
// Generic version — supports *schema.Message and *schema.AgenticMessage
func NewTyped[M adk.MessageType](ctx context.Context, cfg *TypedConfig[M]) (adk.TypedChatModelAgentMiddleware[M], error)

// Non-generic version — equivalent to NewTyped[*schema.Message]
func New(ctx context.Context, cfg *Config) (adk.ChatModelAgentMiddleware, error)
```

## TypedConfig[M] Configuration

<table>
<tr><td>Field</td><td>Type</td><td>Required</td><td>Default</td><td>Description</td></tr>
<tr><td>Model</td><td><pre>model.BaseModel[M]</pre></td><td>Yes</td><td>—</td><td>Model used to generate summaries</td></tr>
<tr><td>ModelOptions</td><td><pre>[]model.Option</pre></td><td>No</td><td>—</td><td>Options passed to the summary model</td></tr>
<tr><td>TokenCounter</td><td><pre>TypedTokenCounterFunc[M]</pre></td><td>No</td><td>Estimates based on the most recent assistant message's total_tokens as baseline, incremental messages at ~4 chars/token</td><td>Custom token counting function</td></tr>
<tr><td>Trigger</td><td><pre>*TriggerCondition</pre></td><td>No</td><td>ContextTokens=160,000</td><td>Condition to trigger summarization</td></tr>
<tr><td>UserInstruction</td><td><pre>string</pre></td><td>No</td><td>Built-in prompt</td><td>Custom user-level summarization instruction, overrides the default instruction</td></tr>
<tr><td>TranscriptFilePath</td><td><pre>string</pre></td><td>No</td><td>—</td><td>Full conversation transcript file path, appended to the summary to remind the model where to find original context. <strong>Only effective when Finalize is not set</strong></td></tr>
<tr><td>GenModelInput</td><td><pre>TypedGenModelInputFunc[M]</pre></td><td>No</td><td>sysInstruction → contextMsgs → userInstruction</td><td>Full control over constructing the summary model input</td></tr>
<tr><td>Finalize</td><td><pre>TypedFinalizeFunc[M]</pre></td><td>No</td><td>Built-in post-processing</td><td>Custom summary post-processing. When set, the middleware no longer performs any default post-processing</td></tr>
<tr><td>Callback</td><td><pre>TypedCallbackFunc[M]</pre></td><td>No</td><td>—</td><td>Called after Finalize, with parameters <pre>before, after adk.TypedChatModelAgentState[M]</pre> (value types), read-only</td></tr>
<tr><td>EmitInternalEvents</td><td><pre>bool</pre></td><td>No</td><td>false</td><td>Whether to send internal events at key points</td></tr>
<tr><td>PreserveUserMessages</td><td><pre>*TypedPreserveUserMessages[M]</pre></td><td>No</td><td>Enabled: true</td><td>Preserve original user messages in the summary. <strong>Only effective when Finalize is not set</strong></td></tr>
<tr><td>Retry</td><td><pre>*TypedRetryConfig[M]</pre></td><td>No</td><td>nil (no retry)</td><td>Retry strategy for the primary model summary generation</td></tr>
<tr><td>Failover</td><td><pre>*TypedFailoverConfig[M]</pre></td><td>No</td><td>nil</td><td>Failover strategy after primary model failure</td></tr>
</table>

> 💡
> **Finalize override semantics**: Once a custom `Finalize` is set, the middleware will **skip all default post-processing** — `PreserveUserMessages` and `TranscriptFilePath` will no longer take effect. To reuse default post-processing logic in a custom Finalize, use the `DefaultFinalizer` function.

## Sub-configuration Structs

### TriggerCondition

Summarization is triggered when **any** condition is met.

```go
type TriggerCondition struct {
    ContextTokens   int // Trigger when token count exceeds this threshold
    ContextMessages int // Trigger when message count exceeds this threshold
}
```

### TypedPreserveUserMessages\[M\]

When enabled, replaces the `<all_user_messages>...</all_user_messages>` section in the summary with the most recent original user messages.

```go
type TypedPreserveUserMessages[M adk.MessageType] struct {
    Enabled   bool
    MaxTokens int                        // Max tokens for preserved user messages; defaults to TriggerCondition.ContextTokens / 3
    Filter    TypedUserMessageFilterFunc[M] // Filter function, return false to exclude a message
}
```

### TypedRetryConfig[M]

```go
type TypedRetryConfig[M adk.MessageType] struct {
    MaxRetries  *int                                                            // Default 3
    ShouldRetry func(ctx context.Context, resp M, err error) bool              // Default: retry when err != nil
    BackoffFunc func(ctx context.Context, attempt int, resp M, err error) time.Duration // Default: exponential backoff + jitter
}
```

### TypedFailoverConfig[M]

```go
type TypedFailoverConfig[M adk.MessageType] struct {
    MaxRetries     *int                                                            // Default 3
    ShouldFailover func(ctx context.Context, resp M, err error) bool              // Default: failover when err != nil
    BackoffFunc    func(ctx context.Context, attempt int, resp M, err error) time.Duration
    GetFailoverModel TypedGetFailoverModelFunc[M] // Returns (failoverModel model.BaseModel[M], failoverModelInputMsgs []M, failoverErr error)
}
```

### TypedFailoverContext[M]

Context passed to the `GetFailoverModel` callback.

```go
type TypedFailoverContext[M adk.MessageType] struct {
    Attempt           int  // Current failover attempt number, starting from 1
    SystemInstruction M    // System instruction (set internally by the middleware, not configurable)
    UserInstruction   M    // User instruction
    OriginalMessages  []M  // Original complete conversation
    LastModelResponse M    // Model response from the last attempt
    LastErr           error
}
```

### TypedTokenCounterInput[M]

```go
type TypedTokenCounterInput[M adk.MessageType] struct {
    Messages []M
    Tools    []*schema.ToolInfo
}
```

## Function Type Signature Reference

```go
type TypedTokenCounterFunc[M]      func(ctx context.Context, input *TypedTokenCounterInput[M]) (int, error)
type TypedGenModelInputFunc[M]     func(ctx context.Context, sysInstruction, userInstruction M, originalMsgs []M) ([]M, error)
type TypedGetFailoverModelFunc[M]  func(ctx context.Context, failoverCtx *TypedFailoverContext[M]) (model.BaseModel[M], []M, error)
type TypedFinalizeFunc[M]          func(ctx context.Context, originalMessages []M, summary M) ([]M, error)
type TypedCallbackFunc[M]          func(ctx context.Context, before, after adk.TypedChatModelAgentState[M]) error
type TypedUserMessageFilterFunc[M] func(ctx context.Context, msg M) (bool, error)
```

## DefaultFinalizer

`DefaultFinalizer` is a standalone factory function that returns a `TypedFinalizeFunc[M]` consistent with the middleware's default post-processing logic. Use it when you need to reuse default logic (preserving user messages, appending transcript path, etc.) in a custom `Finalize`.

```go
func DefaultFinalizer[M adk.MessageType](cfg *DefaultFinalizerConfig[M]) (TypedFinalizeFunc[M], error)
```

### DefaultFinalizerConfig[M]

```go
type DefaultFinalizerConfig[M adk.MessageType] struct {
    PreserveUserMessages *TypedPreserveUserMessages[M] // Default Enabled=true, MaxTokens=30000
    TranscriptFilePath   string
}
```

**Example**: Execute default post-processing first in a custom Finalize, then add a system message:

```go
defaultFinalize, err := summarization.DefaultFinalizer[*schema.Message](&summarization.DefaultFinalizerConfig[*schema.Message]{
    TranscriptFilePath: "/path/to/transcript.txt",
})
if err != nil {
    // handle error
}

cfg := &summarization.Config{
    Model: yourModel,
    Finalize: func(ctx context.Context, originalMessages []*schema.Message, summary *schema.Message) ([]*schema.Message, error) {
        msgs, err := defaultFinalize(ctx, originalMessages, summary)
        if err != nil {
            return nil, err
        }
        // Add system message before the summary
        return append([]*schema.Message{schema.SystemMessage("your system prompt")}, msgs...), nil
    },
}
```

## FinalizerBuilder

`TypedFinalizerBuilder[M]` provides a chainable API for building `TypedFinalizeFunc[M]`, supporting linking multiple handlers and an optional custom finalizer.

```go
func NewTypedFinalizer[M adk.MessageType]() *TypedFinalizerBuilder[M]
func NewFinalizer() *FinalizerBuilder // = NewTypedFinalizer[*schema.Message]

func (b *TypedFinalizerBuilder[M]) PreserveSkills(config *PreserveSkillsConfig) *TypedFinalizerBuilder[M]
func (b *TypedFinalizerBuilder[M]) Custom(fn TypedFinalizeFunc[M]) *TypedFinalizerBuilder[M]
func (b *TypedFinalizerBuilder[M]) Build() (TypedFinalizeFunc[M], error)
```

Execution order: Handlers transform the summary in registration order → Custom determines the final output message list. If Custom is not set, returns `[]M{summary}`.

### PreserveSkills

Preserves skill content loaded by the Skill middleware after summary compression, ensuring the agent retains skill knowledge after context window compression.

```go
type PreserveSkillsConfig struct {
    SkillToolName     string // Skill tool name, must match the Skill middleware. Default "skill"
    MaxSkills         *int   // Maximum number of skills to preserve. Default 5; 0 means disabled
    MaxTokensPerSkill *int   // Maximum tokens per skill, truncated if exceeded. Default 5000
    SkillsTokenBudget *int   // Total token budget for all skills. Default 25000
}
```

**Example**:

```go
finalizer, err := summarization.NewFinalizer().
    PreserveSkills(&summarization.PreserveSkillsConfig{}).
    Custom(func(ctx context.Context, origMsgs []*schema.Message, summary *schema.Message) ([]*schema.Message, error) {
        return []*schema.Message{schema.SystemMessage("system prompt"), summary}, nil
    }).
    Build()

cfg := &summarization.Config{
    Model:    yourModel,
    Finalize: finalizer,
}
```

## Summarize Method

`TypedMiddleware[M]` exposes a `Summarize` method that can manually trigger a summarization outside of the middleware's automatic trigger:

```go
func (m *TypedMiddleware[M]) Summarize(ctx context.Context, state *adk.TypedChatModelAgentState[M]) ([]M, error)
```

This method executes the full summarization flow (generation → post-processing → Callback → events) but **does not check trigger conditions**. Returns the replaced message list.

## How It Works

<a href="/img/eino/DwTrwyD1eh2DqNbsGE8cfdTNnYb.png" target="_blank"><img src="/img/eino/DwTrwyD1eh2DqNbsGE8cfdTNnYb.png" width="100%" /></a>

**Trigger condition check**: First checks `ContextMessages` (message count), then calculates token count via `TokenCounter` and compares with `ContextTokens`. Triggered if either is met.

**Default post-processing** (when Finalize is not set):

1. Replaces `<all_user_messages>...</all_user_messages>` in the summary with the most recent original user messages (controlled by `PreserveUserMessages`)
2. Appends `TranscriptFilePath` hint
3. Adds summary preamble and continuation instructions

## Internal Events

When `EmitInternalEvents = true`, the middleware sends events via `adk.TypedSendEvent`:

<table>
<tr><td>Event Type</td><td>Trigger Timing</td><td>Carried Data</td></tr>
<tr><td><pre>ActionTypeBeforeSummarize</pre></td><td>After trigger condition is met, before calling the model</td><td><pre>TypedBeforeSummarizeAction[M]{Messages}</pre>: original message list</td></tr>
<tr><td><pre>ActionTypeGenerateSummary</pre></td><td>After each model generation attempt (including retry/failover)</td><td><pre>TypedGenerateSummaryAction[M]{Attempt, Phase, ModelResponse, GetError()}</pre></td></tr>
<tr><td><pre>ActionTypeAfterSummarize</pre></td><td>After summary completion and Finalize</td><td><pre>TypedAfterSummarizeAction[M]{Messages}</pre>: final message list</td></tr>
</table>

Events are wrapped in `TypedCustomizedAction[M]` and placed in the `adk.AgentAction.CustomizedAction` field. `GenerateSummaryPhase` has two values: `GenerateSummaryPhasePrimary` (primary model/retry) and `GenerateSummaryPhaseFailover` (failover).

## Usage Examples

### Minimal Configuration

```go
mw, err := summarization.New(ctx, &summarization.Config{
    Model: yourChatModel,
})

agent, err := adk.NewChatModelAgent(ctx, &adk.ChatModelAgentConfig{
    Model:       yourChatModel,
    Middlewares: []adk.ChatModelAgentMiddleware{mw},
})
```

### Custom Trigger + Retry + Failover

```go
mw, err := summarization.New(ctx, &summarization.Config{
    Model: yourChatModel,
    Trigger: &summarization.TriggerCondition{
        ContextTokens:   100000,
        ContextMessages: 80,
    },
    TranscriptFilePath: "/path/to/transcript.txt",
    Retry: &summarization.RetryConfig{
        MaxRetries: ptrOf(2),
    },
    Failover: &summarization.FailoverConfig{
        MaxRetries: ptrOf(3),
        GetFailoverModel: func(ctx context.Context, fctx *summarization.FailoverContext) (model.BaseModel[*schema.Message], []*schema.Message, error) {
            return backupModel, nil, nil // Returning nil input will reuse the default input
        },
    },
})
```

### FinalizerBuilder + PreserveSkills + DefaultFinalizer

```go
defaultFinalize, _ := summarization.DefaultFinalizer[*schema.Message](
    &summarization.DefaultFinalizerConfig[*schema.Message]{
        TranscriptFilePath: "/path/to/transcript.txt",
    },
)

finalizer, err := summarization.NewFinalizer().
    PreserveSkills(&summarization.PreserveSkillsConfig{
        MaxSkills: ptrOf(3),
    }).
    Custom(func(ctx context.Context, origMsgs []*schema.Message, summary *schema.Message) ([]*schema.Message, error) {
        msgs, err := defaultFinalize(ctx, origMsgs, summary)
        if err != nil {
            return nil, err
        }
        return append([]*schema.Message{schema.SystemMessage("system prompt")}, msgs...), nil
    }).
    Build()

cfg := &summarization.Config{
    Model:    yourModel,
    Finalize: finalizer,
}
```

## Notes

1. **Set TranscriptFilePath**: Strongly recommended to provide a conversation transcript file path so the model can trace back details from the original records after summarization.
2. **Adjust trigger threshold**: `Trigger.ContextTokens` should be set to 80-90% of the model's context window. The default value of 160,000 is suitable for models with 200k windows.
3. **Custom TokenCounter**: For production environments, it's recommended to implement a counter that precisely matches the model's tokenizer. The default estimator uses the most recent assistant message's `ResponseMeta.Usage.TotalTokens` as a baseline and estimates incremental messages at ~4 chars/token.
4. **Finalize override**: Setting `Finalize` means `PreserveUserMessages` and `TranscriptFilePath` no longer take effect automatically. To reuse them, use `DefaultFinalizer` or `FinalizerBuilder`.
5. **GetFailoverModel constraint**: The callback must return a non-nil model and non-empty input message list.
