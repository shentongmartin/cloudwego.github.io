---
Description: ""
date: "2026-05-25"
lastmod: ""
tags: []
title: AgenticChatTemplate Guide [Beta]
weight: 11
---

> 💡
> This feature is available starting from [v0.9](https://github.com/cloudwego/eino/releases/tag/v0.9.0-alpha.2). This document is based on the current main branch source code, with code primarily located in `components/prompt`, `schema/agentic_message.go`, and `compose`.

## Introduction

`AgenticChatTemplate` is a Prompt component abstraction designed for `*schema.AgenticMessage`. It fills variables from a `map[string]any` into agentic message templates and outputs `[]*schema.AgenticMessage` for use by `AgenticModel` or subsequent orchestration nodes.

Its design is essentially the same as the existing `ChatTemplate`, but the message type is switched from `*schema.Message` to `*schema.AgenticMessage`, and template rendering only applies to user input content blocks.

## Component Definition

Code location: `components/prompt/interface.go`

```go
type AgenticChatTemplate interface {
    Format(ctx context.Context, vs map[string]any, opts ...Option) ([]*schema.AgenticMessage, error)
}
```

`Format` parameter description:

<table>
<tr><td>Parameter</td><td>Description</td></tr>
<tr><td><pre>ctx</pre></td><td>Request context, also carries runtime information such as the callback manager</td></tr>
<tr><td><pre>vs</pre></td><td>Template variable map, where the key is the placeholder name and value is the actual value</td></tr>
<tr><td><pre>opts</pre></td><td>Prompt component Option, supports implementation-side custom extensions</td></tr>
</table>

Return values:

<table>
<tr><td>Return Value</td><td>Description</td></tr>
<tr><td><pre>[]*schema.AgenticMessage</pre></td><td>The rendered agentic message list</td></tr>
<tr><td><pre>error</pre></td><td>Returns an error when variables are missing, variable types don't match, or template rendering fails</td></tr>
</table>

## Construction Methods

### FromAgenticMessages

Code location: `components/prompt/agentic_chat_template.go`

```go
func FromAgenticMessages(formatType schema.FormatType, templates ...schema.AgenticMessagesTemplate) *DefaultAgenticChatTemplate
```

`FromAgenticMessages` does not return an `error`. It accepts a template format and a set of `schema.AgenticMessagesTemplate`, and returns the default implementation `*DefaultAgenticChatTemplate`.

```go
template := prompt.FromAgenticMessages(schema.FString,
    schema.SystemAgenticMessage("You are a {role}."),
    schema.AgenticMessagesPlaceholder("history", true),
    schema.UserAgenticMessage("Please help me {task}."),
)
```

### Supported FormatTypes

<table>
<tr><td>Format</td><td>Constant</td><td>Placeholder Example</td><td>Applicable Scenarios</td></tr>
<tr><td>FString</td><td><pre>schema.FString</pre></td><td><pre>{role}</pre></td><td>Simple variable substitution</td></tr>
<tr><td>GoTemplate</td><td><pre>schema.GoTemplate</pre></td><td><pre>{{.role}}</pre></td><td>Scenarios requiring Go <pre>text/template</pre> capabilities</td></tr>
<tr><td>Jinja2</td><td><pre>schema.Jinja2</pre></td><td><pre>{{ role }}</pre></td><td>Scenarios requiring Jinja2 syntax</td></tr>
</table>

## AgenticMessagesTemplate

Code location: `schema/agentic_message.go`

```go
type AgenticMessagesTemplate interface {
    Format(ctx context.Context, vs map[string]any, formatType FormatType) ([]*AgenticMessage, error)
}
```

Common implementations include:

<table>
<tr><td>Construction Method</td><td>Description</td></tr>
<tr><td><pre>&schema.AgenticMessage{...}</pre></td><td><pre>AgenticMessage</pre> itself implements <pre>AgenticMessagesTemplate</pre></td></tr>
<tr><td><pre>schema.SystemAgenticMessage(text)</pre></td><td>Constructs a <pre>system</pre> role message</td></tr>
<tr><td><pre>schema.UserAgenticMessage(text)</pre></td><td>Constructs a <pre>user</pre> role message</td></tr>
<tr><td><pre>schema.AgenticMessagesPlaceholder(key, optional)</pre></td><td>Inserts a set of historical agentic messages from the variable map</td></tr>
</table>

### Template Rendering Scope

`AgenticMessage.Format` only formats user input blocks:

- `ContentBlockTypeUserInputText`
- `ContentBlockTypeUserInputImage`
- `ContentBlockTypeUserInputAudio`
- `ContentBlockTypeUserInputVideo`
- `ContentBlockTypeUserInputFile`

Model output, reasoning, tool call, tool result, MCP, and server tool related blocks are not rendered as prompt template content.

> 💡
> It is recommended to use `schema.SystemAgenticMessage`, `schema.UserAgenticMessage`, and `schema.NewContentBlock` to construct templates, avoiding inconsistencies between `ContentBlock.Type` and content fields when setting them manually.

## Placeholder

Code location: `schema/agentic_message.go`

```go
func AgenticMessagesPlaceholder(key string, optional bool) AgenticMessagesTemplate
```

Behavior rules:

<table>
<tr><td>Scenario</td><td>Behavior</td></tr>
<tr><td><pre>vs[key]</pre> exists and type is <pre>[]*schema.AgenticMessage</pre></td><td>Inserts the message list as-is</td></tr>
<tr><td><pre>vs[key]</pre> does not exist and <pre>optional=true</pre></td><td>Returns an empty slice without error</td></tr>
<tr><td><pre>vs[key]</pre> does not exist and <pre>optional=false</pre></td><td>Returns an error</td></tr>
<tr><td><pre>vs[key]</pre> type is not <pre>[]*schema.AgenticMessage</pre></td><td>Returns an error</td></tr>
</table>

Example:

```go
history := []*schema.AgenticMessage{
    schema.UserAgenticMessage("What is oil painting?"),
    {
        Role: schema.AgenticRoleTypeAssistant,
        ContentBlocks: []*schema.ContentBlock{
            schema.NewContentBlock(&schema.AssistantGenText{Text: "Oil painting is ..."}),
        },
    },
}

messages, err := template.Format(ctx, map[string]any{
    "role":    "professional assistant",
    "task":    "write a short poem",
    "history": history,
})
if err != nil {
    return err
}
```

## Standalone Usage

```go
template := prompt.FromAgenticMessages(schema.FString,
    schema.SystemAgenticMessage("You are a {role}."),
    schema.AgenticMessagesPlaceholder("history", true),
    schema.UserAgenticMessage("Please help me {task}."),
)

messages, err := template.Format(ctx, map[string]any{
    "role": "concise assistant",
    "task": "summarize the following requirement",
    "history": []*schema.AgenticMessage{
        schema.UserAgenticMessage("Previous question"),
    },
})
if err != nil {
    return err
}
```

## Usage in Orchestration

### Chain

Code location: `compose/chain.go`

```go
func (c *Chain[I, O]) AppendAgenticChatTemplate(node prompt.AgenticChatTemplate, opts ...GraphAddNodeOpt) *Chain[I, O]
```

```go
chain := compose.NewChain[map[string]any, *schema.AgenticMessage]()
chain.AppendAgenticChatTemplate(template)
chain.AppendAgenticModel(model)
```

### Graph

Code location: `compose/graph.go`

```go
func (g *graph) AddAgenticChatTemplateNode(key string, node prompt.AgenticChatTemplate, opts ...GraphAddNodeOpt) error
```

```go
graph := compose.NewGraph[map[string]any, *schema.AgenticMessage]()
err := graph.AddAgenticChatTemplateNode("prompt", template)
if err != nil {
    return err
}
```

### Workflow

Code location: `compose/workflow.go`

```go
func (wf *Workflow[I, O]) AddAgenticChatTemplateNode(key string, chatTemplate prompt.AgenticChatTemplate, opts ...GraphAddNodeOpt) *WorkflowNode
```

### Parallel and ChainBranch

Agentic prompt can also be placed in `Parallel` or `ChainBranch`:

```go
func (p *Parallel) AddAgenticChatTemplate(outputKey string, node prompt.AgenticChatTemplate, opts ...GraphAddNodeOpt) *Parallel

func (cb *ChainBranch) AddAgenticChatTemplate(key string, node prompt.AgenticChatTemplate, opts ...GraphAddNodeOpt) *ChainBranch
```

## Getting Variables from Predecessor Nodes

`AgenticChatTemplate.Format` requires `map[string]any`. If the predecessor node's output is not a map, you can use `compose.WithOutputKey` when adding the node to wrap the output as a single-field map.

```go
graph.AddLambdaNode("query",
    compose.InvokableLambda(func(ctx context.Context, input string) (string, error) {
        return input, nil
    }),
    compose.WithOutputKey("task"),
)

graph.AddAgenticChatTemplateNode("prompt", template)
```

The wrapped map looks like:

```go
map[string]any{
    "task": previousNodeOutput,
}
```

## Callback

### Component-level Callback Payload

Code location: `components/prompt/agentic_callback_extra.go`

```go
type AgenticCallbackInput struct {
    Variables map[string]any
    Templates []schema.AgenticMessagesTemplate
    Extra     map[string]any
}

type AgenticCallbackOutput struct {
    Result    []*schema.AgenticMessage
    Templates []schema.AgenticMessagesTemplate
    Extra     map[string]any
}

func ConvAgenticCallbackInput(src callbacks.CallbackInput) *AgenticCallbackInput
func ConvAgenticCallbackOutput(src callbacks.CallbackOutput) *AgenticCallbackOutput
```

`DefaultAgenticChatTemplate.Format` triggers callbacks at the start and end, passing the above agentic payload.

### utils/callbacks Helper

Code location: `utils/callbacks/template.go`

The current public signature of `AgenticPromptCallbackHandler` is as follows:

```go
type AgenticPromptCallbackHandler struct {
    OnStart func(ctx context.Context, runInfo *callbacks.RunInfo, input *prompt.CallbackInput) context.Context
    OnEnd   func(ctx context.Context, runInfo *callbacks.RunInfo, output *prompt.CallbackOutput) context.Context
    OnError func(ctx context.Context, runInfo *callbacks.RunInfo, err error) context.Context
}
```

Register using the helper:

```go
handler := callbackHelper.NewHandlerHelper().
    AgenticPrompt(&callbackHelper.AgenticPromptCallbackHandler{
        OnStart: func(ctx context.Context, info *callbacks.RunInfo, input *prompt.CallbackInput) context.Context {
            if input != nil {
                fmt.Printf("variables: %v\n", input.Variables)
            }
            return ctx
        },
        OnError: func(ctx context.Context, info *callbacks.RunInfo, err error) context.Context {
            fmt.Printf("prompt error: %v\n", err)
            return ctx
        },
    }).
    Handler()

result, err := runnable.Invoke(ctx, variables, compose.WithCallbacks(handler))
```

> 💡
> If you need to directly handle `*prompt.AgenticCallbackInput` or `*prompt.AgenticCallbackOutput`, you should use the component-level callback payload along with `prompt.ConvAgenticCallbackInput/Output`. The current public signature of `utils/callbacks.AgenticPromptCallbackHandler` still reuses `prompt.CallbackInput/Output`.

## Custom Implementation

Implementing a custom `AgenticChatTemplate` only requires satisfying the interface:

```go
type MyAgenticPrompt struct {
    templates  []schema.AgenticMessagesTemplate
    formatType schema.FormatType
}

func (p *MyAgenticPrompt) Format(ctx context.Context, vs map[string]any, opts ...prompt.Option) ([]*schema.AgenticMessage, error) {
    result := make([]*schema.AgenticMessage, 0, len(p.templates))
    for _, tpl := range p.templates {
        msgs, err := tpl.Format(ctx, vs, p.formatType)
        if err != nil {
            return nil, err
        }
        result = append(result, msgs...)
    }
    return result, nil
}
```

If you need to support custom options, you can implement them using `prompt.WrapImplSpecificOptFn` and `prompt.GetImplSpecificOptions`.

## Best Practices

- `FromAgenticMessages` does not return an `error`, so examples should not be written as `template, err := ...`.
- The variable value for `AgenticMessagesPlaceholder` must be `[]*schema.AgenticMessage`.
- Template variable names should be stable and consistent; missing variables will return an error at runtime.
- Use `AgenticMessagesPlaceholder("history", true)` for conversation history, which naturally returns an empty list when there is no history.
- In Graph/Chain, ensure that the node following the Agentic prompt accepts `[]*schema.AgenticMessage` type, such as `AgenticModel`.
