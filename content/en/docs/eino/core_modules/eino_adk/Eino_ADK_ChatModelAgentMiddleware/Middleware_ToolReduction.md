---
Description: ""
date: "2026-05-17"
lastmod: ""
tags: []
title: Reduction
weight: 5
---

`adk/middlewares/reduction`

> 💡
> This middleware was introduced in v0.8.0.

## Overview

The `reduction` middleware manages the token count occupied by tool outputs in Agent conversations, operating in two phases:

1. **Truncation**: Triggered immediately when a tool call returns. When a single output exceeds `MaxLengthForTrunc`, the full content is stored in the Backend and the message is replaced with a truncated summary.
2. **Clear**: Triggered before model calls (`BeforeModelRewriteState`). When total tokens exceed `MaxTokensForClear`, it iterates through historical messages and offloads old tool arguments and results to the Backend.

---

## Architecture

```
Tool call returns result
                          │
                          ▼
┌─────────────────────────────────────────────────────────────┐
│     WrapInvokableToolCall / WrapStreamableToolCall          │
│     WrapEnhancedInvokableToolCall / WrapEnhancedStreamable  │
│                                                             │
│  Truncation (can be skipped via SkipTruncation)             │
│    Result length > MaxLengthForTrunc?                       │
│      Yes → Truncate content, save full content to Backend   │
│      No  → Return as-is                                     │
└─────────────────────────────────────────────────────────────┘
                          │
                          ▼
                    Result added to Messages
                          │
                          ▼
┌─────────────────────────────────────────────────────────────┐
│                  BeforeModelRewriteState                    │
│                                                             │
│  Clear (can be skipped via SkipClear)                       │
│    Total tokens > MaxTokensForClear?                        │
│      Yes → ClearMessageRewriter preprocessing              │
│         → Old tool results stored to Backend, replaced     │
│           with file paths                                   │
│         → ClearAtLeastTokens minimum release check         │
│         → ClearPostProcess callback                        │
│      No  → Do nothing                                       │
└─────────────────────────────────────────────────────────────┘
                          │
                          ▼
                     Call Model
```

---

## Generic System

This middleware follows the ADK standard generic pattern, supporting both `*schema.Message` and `*schema.AgenticMessage`:

```go
// Generic config, M is constrained to adk.MessageType
type TypedConfig[M adk.MessageType] struct { ... }

// Backward-compatible alias
type Config = TypedConfig[*schema.Message]
```

Constructors are also available in both generic and non-generic forms:

```go
func NewTyped[M adk.MessageType](ctx context.Context, config *TypedConfig[M]) (adk.TypedChatModelAgentMiddleware[M], error)
func New(ctx context.Context, config *Config) (adk.ChatModelAgentMiddleware, error)
```

---

## Configuration

### TypedConfig[M] Main Configuration

<table>
<tr><td>Field</td><td>Type</td><td>Description</td></tr>
<tr><td>Backend</td><td><pre>Backend</pre></td><td>Storage backend. <strong>Required</strong> when <pre>SkipTruncation</pre> is false; can be nil when only doing Clear without offload.</td></tr>
<tr><td>SkipTruncation</td><td><pre>bool</pre></td><td>Skip the truncation phase.</td></tr>
<tr><td>SkipClear</td><td><pre>bool</pre></td><td>Skip the clear phase.</td></tr>
<tr><td>ReadFileToolName</td><td><pre>string</pre></td><td>Tool name for reading offloaded content. Default <pre>"read_file"</pre>.</td></tr>
<tr><td>RootDir</td><td><pre>string</pre></td><td>Root directory for saving content. Default <pre>"/tmp"</pre>. Truncated content is saved to <pre>{RootDir}/trunc/{tool_call_id}</pre>, cleared content to <pre>{RootDir}/clear/{tool_call_id}</pre>.</td></tr>
<tr><td>GenTruncOffloadFilePath</td><td><pre>func(ctx, *ToolDetail) (string, error)</pre></td><td>Custom truncation file path generator. When set, RootDir does not apply to truncation. Useful for scenarios where tool_call_id is not unique.</td></tr>
<tr><td>GenClearOffloadFilePath</td><td><pre>func(ctx, *ToolDetail) (string, error)</pre></td><td>Custom clear file path generator. When set, RootDir does not apply to clear.</td></tr>
<tr><td>MaxLengthForTrunc</td><td><pre>int</pre></td><td>Maximum character length to trigger truncation. Default <pre>50000</pre>.</td></tr>
<tr><td>TruncExcludeTools</td><td><pre>[]string</pre></td><td>List of tool names to exclude from truncation.</td></tr>
<tr><td>TokenCounter</td><td><pre>func(ctx, []M, []*schema.ToolInfo) (int64, error)</pre></td><td>Token counting function. Defaults to character_count/4 estimation. <strong>Recommend replacing with tiktoken-go/tokenizer</strong>.</td></tr>
<tr><td>MaxTokensForClear</td><td><pre>int64</pre></td><td>Token threshold to trigger clear. Default <pre>160000</pre>.</td></tr>
<tr><td>ClearRetentionSuffixLimit</td><td><pre>int</pre></td><td>Keep the most recent N assistant message rounds without clearing. Default <pre>1</pre>.</td></tr>
<tr><td>ClearAtLeastTokens</td><td><pre>int64</pre></td><td>Minimum token amount that must be released by clearing. If not met, clearing is not executed (avoids needlessly breaking prompt cache). Default <pre>0</pre>.</td></tr>
<tr><td>ClearExcludeTools</td><td><pre>[]string</pre></td><td>List of tool names to exclude from clearing.</td></tr>
<tr><td>ClearMessageRewriter</td><td><pre>func(ctx, M, []M) ([]M, error)</pre></td><td>Message rewrite callback before clearing. Parameters are toolCallMsg and the corresponding toolResponseMsgs. Can be used to rewrite write_file/edit_file calls into system-reminders. Returning nil removes that message group.</td></tr>
<tr><td>ClearPostProcess</td><td><pre>func(ctx, *adk.TypedChatModelAgentState[M]) context.Context</pre></td><td>Callback after clearing completes, can save state or send notifications. Returns a potentially updated context.</td></tr>
<tr><td>ToolConfig</td><td><pre>map[string]*ToolReductionConfig</pre></td><td>Per-tool configuration, takes precedence over global settings.</td></tr>
</table>

### ToolReductionConfig Tool-level Configuration

```go
type ToolReductionConfig struct {
    Backend        Backend
    SkipTruncation bool
    TruncHandler   func(ctx context.Context, detail *ToolDetail) (*TruncResult, error)
    SkipClear      bool
    ClearHandler   func(ctx context.Context, detail *ToolDetail) (*ClearResult, error)
}
```

- `TruncHandler` / `ClearHandler`: when nil and not skipped, the global default handler is used.
- `Backend`: independent storage backend for this tool, overrides the global Backend.

### ToolDetail Tool Details

```go
type ToolDetail struct {
    ToolContext       *adk.ToolContext
    ToolArgument      *schema.ToolArgument
    ToolResult        *schema.ToolResult                    // non-streaming
    StreamToolResult  *schema.StreamReader[*schema.ToolResult] // streaming
}
```

### TruncResult Truncation Result

```go
type TruncResult struct {
    NeedTrunc        bool
    ToolResult       *schema.ToolResult                    // Required when NeedTrunc && non-streaming
    StreamToolResult *schema.StreamReader[*schema.ToolResult] // Required when NeedTrunc && streaming
    NeedOffload      bool
    OffloadFilePath  string  // Required when NeedOffload
    OffloadContent   string  // Required when NeedOffload
}
```

### ClearResult Clear Result

```go
type ClearResult struct {
    NeedClear       bool
    ToolArgument    *schema.ToolArgument  // Required when NeedClear
    ToolResult      *schema.ToolResult    // Required when NeedClear
    NeedOffload     bool
    OffloadFilePath string  // Required when NeedOffload
    OffloadContent  string  // Required when NeedOffload
}
```

### Backend Interface

```go
// Defined in reduction/internal, exported via type alias
type Backend interface {
    Write(context.Context, *filesystem.WriteRequest) error
}
```

`filesystem.WriteRequest` contains two fields: `FilePath string` and `Content string`.

---

## Creating the Middleware

### Basic Usage

```go
import "github.com/cloudwego/eino/adk/middlewares/reduction"

middleware, err := reduction.New(ctx, &reduction.Config{
    Backend: myBackend,
})

agent, err := adk.NewChatModelAgent(ctx, &adk.ChatModelAgentConfig{
    Model:       chatModel,
    Middlewares: []adk.ChatModelAgentMiddleware{middleware},
})
```

### Generic Usage (AgenticMessage)

```go
middleware, err := reduction.NewTyped[*schema.AgenticMessage](ctx, &reduction.TypedConfig[*schema.AgenticMessage]{
    Backend: myBackend,
    TokenCounter: myAgenticTokenCounter,
})

agent, err := adk.NewTypedChatModelAgent(ctx, &adk.TypedChatModelAgentConfig[*schema.AgenticMessage]{
    Model:       chatModel,
    Middlewares: []adk.TypedChatModelAgentMiddleware[*schema.AgenticMessage]{middleware},
})
```

### Custom Configuration

```go
middleware, err := reduction.New(ctx, &reduction.Config{
    Backend:           myBackend,
    RootDir:           "/data/agent",
    MaxLengthForTrunc: 30000,
    MaxTokensForClear: 100000,
    ClearRetentionSuffixLimit: 2,
    ClearAtLeastTokens: 10000,
    TruncExcludeTools: []string{"search_tool"},
    ClearExcludeTools: []string{"read_file"},
    ClearMessageRewriter: func(ctx context.Context, toolCallMsg *schema.Message, toolResponseMsgs []*schema.Message) ([]*schema.Message, error) {
        // Rewrite write_file calls into system-reminder
        return []*schema.Message{schema.UserMessage("<system-reminder>file written</system-reminder>")}, nil
    },
    ClearPostProcess: func(ctx context.Context, state *adk.ChatModelAgentState) context.Context {
        log.Printf("Clear completed, messages: %d", len(state.Messages))
        return ctx
    },
    ToolConfig: map[string]*reduction.ToolReductionConfig{
        "grep": {Backend: grepBackend},
        "read_file": {SkipClear: true},
    },
})
```

### Truncation Only

```go
middleware, err := reduction.New(ctx, &reduction.Config{
    Backend:   myBackend,
    SkipClear: true,
})
```

### Clear Only

```go
middleware, err := reduction.New(ctx, &reduction.Config{
    SkipTruncation: true,
    MaxTokensForClear: 100000,
    // When Backend is nil, clearing still replaces content with placeholders but does not perform offload
})
```

---

## How It Works

### Truncation

Handled in `WrapInvokableToolCall` / `WrapStreamableToolCall` / `WrapEnhancedInvokableToolCall` / `WrapEnhancedStreamableToolCall`:

1. Tool returns result
2. Check `TruncExcludeTools`; skip if matched
3. Look up ToolConfig → global defaultConfig to obtain TruncHandler
4. TruncHandler determines: reads the full output, checks if the total length of all text parts exceeds `MaxLengthForTrunc`
5. If exceeded: retains the first and last `MaxLengthForTrunc/(textParts*2)` characters as a preview, stores the full content in the Backend
6. Returns a truncation notice informing the agent of the file path for the full content

> 💡
> For streaming tools, the default TruncHandler waits for the complete stream to be read before deciding whether to truncate. If you need strict incremental streaming behavior, provide a custom TruncHandler for that tool.

### Clear

Handled in `BeforeModelRewriteState`:

1. Use `TokenCounter` to calculate total tokens
2. Skip if not exceeding `MaxTokensForClear`
3. Determine clear range: from the first unprocessed assistant message to `len(messages) - ClearRetentionSuffixLimit` rounds
4. If `ClearMessageRewriter` is configured, execute rewrite preprocessing on messages within the range first
5. Iterate through tool call messages in range, skipping `ClearExcludeTools`
6. Call ClearHandler for each tool call, replacing arguments and results
7. If `ClearAtLeastTokens` is set: operate on a copy first, compare token difference before and after clearing; abandon this clearing attempt if threshold not met
8. Once threshold is met, execute actual offload writes and update state.Messages
9. Call `ClearPostProcess`

---

## Multi-language Support

Truncation and clear prompt text supports automatic Chinese/English switching:

```go
adk.SetLanguage(adk.LanguageChinese)  // Chinese
adk.SetLanguage(adk.LanguageEnglish)  // English (default)
```

---

## Notes

- When `SkipTruncation` is false, `Backend` **must** be set
- The default TokenCounter uses character_count/4 estimation; recommend replacing with `github.com/tiktoken-go/tokenizer`
- Already processed messages are marked via the Extra field `_reduction_mw_processed` and will not be processed again
- Configuration in `ToolConfig` takes precedence over global settings; if a ToolConfig only sets `SkipTruncation: false` without providing a `TruncHandler`, it falls back to the default handler
- `GenTruncOffloadFilePath` / `GenClearOffloadFilePath` are useful for scenarios where tool_call_id is not unique (e.g., retries), preventing file overwrites
- `ClearMessageRewriter` executes after the clear range is determined but before per-tool clearing, suitable for compressing write/edit-type calls into brief prompts
- `ClearAtLeastTokens` set to 0 means clearing executes whenever the threshold is exceeded; values greater than 0 can avoid minimal clearing that would break prompt cache
- Legacy API (`NewClearToolResult`, `NewToolResultMiddleware`) is deprecated; recommend migrating to `New` / `NewTyped`
