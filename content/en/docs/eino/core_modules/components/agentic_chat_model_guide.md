---
Description: ""
date: "2026-05-25"
lastmod: ""
tags: []
title: AgenticModel User Guide [Beta]
weight: 10
---

> 💡
> This feature is available starting from [v0.9](https://github.com/cloudwego/eino/releases/tag/v0.9.0-alpha.2). This document is based on the current main branch source code, with code primarily located in `components/model/interface.go`, `components/model/option.go`, and `schema/agentic_message.go`.

## Introduction

`AgenticModel` is Eino's model component abstraction for agentic provider APIs. It uses `*schema.AgenticMessage` as the message carrier and represents text, reasoning, multimodal content, function tool calls, server-side built-in tool calls, MCP tool calls, and approval results through ordered `ContentBlock`s.

Compared to the traditional `ChatModel`, `AgenticModel` is more suitable for directly leveraging the native agentic capabilities of provider APIs such as OpenAI Responses API, Claude API, and Gemini API: a single model request may contain multiple reasoning segments, multiple tool calls, server tool results, or MCP approval information. Eino no longer flattens these into a single text field but preserves them as structured blocks.

## Relationship with ChatModel

<table>
<tr><td>Dimension</td><td>AgenticModel</td><td>ChatModel</td></tr>
<tr><td>Base Interface</td><td><pre>model.BaseModel[*schema.AgenticMessage]</pre></td><td><pre>model.BaseModel[*schema.Message]</pre></td></tr>
<tr><td>Type Alias</td><td><pre>type AgenticModel = BaseModel[*schema.AgenticMessage]</pre></td><td><pre>type BaseChatModel = BaseModel[*schema.Message]</pre></td></tr>
<tr><td>Message Structure</td><td><pre>AgenticMessage.ContentBlocks</pre></td><td><pre>Message.Content</pre>, <pre>ToolCalls</pre>, <pre>ToolCallID</pre>, etc.</td></tr>
<tr><td>Tool Binding</td><td>Passed at call time via <pre>model.WithTools(...)</pre>, <pre>model.WithAgenticToolChoice(...)</pre></td><td><pre>ToolCallingChatModel.WithTools(...)</pre> or call-time Option</td></tr>
<tr><td>Expressiveness</td><td>reasoning, multimodal input/output, function tool, server tool, MCP tool, approval</td><td>Traditional chat completion messages and tool calls</td></tr>
<tr><td>Downstream Executor</td><td><pre>compose.AgenticToolsNode</pre></td><td><pre>compose.ToolsNode</pre></td></tr>
</table>

> 💡
> `AgenticModel` does not have a `WithTools` method. `WithTools` is an instance method of `ToolCallingChatModel`; in the agentic path, tool information is passed via `model.WithTools(tools)` as a request-level Option.

## Component Definition

### BaseModel and AgenticModel

Code location: `components/model/interface.go`

```go
// BaseModel is the generic base model interface parameterized by message type M.
type BaseModel[M any] interface {
    Generate(ctx context.Context, input []M, opts ...Option) (M, error)
    Stream(ctx context.Context, input []M, opts ...Option) (*schema.StreamReader[M], error)
}

// AgenticModel is a type alias for BaseModel specialized with
// *schema.AgenticMessage.
type AgenticModel = BaseModel[*schema.AgenticMessage]
```

### Generate

`Generate` blocks until the model returns a complete response.

```go
Generate(ctx context.Context, input []*schema.AgenticMessage, opts ...model.Option) (*schema.AgenticMessage, error)
```

Parameter description:

<table>
<tr><td>Parameter</td><td>Description</td></tr>
<tr><td><pre>ctx</pre></td><td>Request context, also carries runtime information such as callback manager</td></tr>
<tr><td><pre>input</pre></td><td>Chronologically ordered list of agentic messages</td></tr>
<tr><td><pre>opts</pre></td><td>Request-level model configuration, such as temperature, tools, tool choice, model name, etc.</td></tr>
</table>

### Stream

`Stream` returns an incremental message stream. The caller is responsible for reading and closing the `StreamReader`.

```go
Stream(ctx context.Context, input []*schema.AgenticMessage, opts ...model.Option) (*schema.StreamReader[*schema.AgenticMessage], error)
```

```go
reader, err := m.Stream(ctx, messages)
if err != nil {
    return err
}
defer reader.Close()

for {
    chunk, err := reader.Recv()
    if errors.Is(err, io.EOF) {
        break
    }
    if err != nil {
        return err
    }
    // handle chunk
}
```

> 💡
> `schema.StreamReader` can only be consumed once. If multiple consumers need to read the same stream, the reader should be copied before reading, rather than reused after consumption.

## Request-Level Options

Code location: `components/model/option.go`

<table>
<tr><td>Option</td><td>Applicable Scope</td><td>Description</td></tr>
<tr><td><pre>model.WithTools(tools)</pre></td><td>General, commonly used in agentic path</td><td>Sets the tool definitions that can be called by the model for this request; <pre>nil</pre> is normalized to an empty slice</td></tr>
<tr><td><pre>model.WithDeferredTools(tools)</pre></td><td>AgenticModel</td><td>Registers tools that can be lazily loaded by the model's built-in tool search; do not also place them in <pre>WithTools</pre></td></tr>
<tr><td><pre>model.WithToolSearchTool(tool)</pre></td><td>AgenticModel</td><td>Registers the tool search tool used by the model to retrieve deferred tools; do not place it in <pre>WithTools</pre></td></tr>
<tr><td><pre>model.WithAgenticToolChoice(choice)</pre></td><td>AgenticModel</td><td>Controls how the agentic model selects function/MCP/server tools</td></tr>
<tr><td><pre>model.WithToolChoice(choice, names...)</pre></td><td>ChatModel</td><td>Tool choice only applicable to ChatModel</td></tr>
</table>

`AgenticToolChoice` is defined in `schema/tool.go`:

```go
type AgenticToolChoice struct {
    Type    ToolChoice
    Allowed *AgenticAllowedToolChoice
    Forced  *AgenticForcedToolChoice
}

type AllowedTool struct {
    FunctionName string
    MCPTool      *AllowedMCPTool
    ServerTool   *AllowedServerTool
}
```

## AgenticMessage

Code location: `schema/agentic_message.go`

```go
type AgenticRoleType string

const (
    AgenticRoleTypeSystem    AgenticRoleType = "system"
    AgenticRoleTypeUser      AgenticRoleType = "user"
    AgenticRoleTypeAssistant AgenticRoleType = "assistant"
)

type AgenticMessage struct {
    Role          AgenticRoleType
    ContentBlocks []*ContentBlock
    ResponseMeta  *AgenticResponseMeta
    Extra         map[string]any
}
```

`AgenticMessage` only has three role types: `system`, `user`, and `assistant`. Tool calls and tool results are not independent roles but are expressed through `ContentBlock`.

### ResponseMeta

```go
type AgenticResponseMeta struct {
    TokenUsage      *TokenUsage
    OpenAIExtension *openai.ResponseMetaExtension
    GeminiExtension *gemini.ResponseMetaExtension
    ClaudeExtension *claude.ResponseMetaExtension
    Extension       any
}
```

`TokenUsage` provides universal token statistics; OpenAI, Gemini, and Claude provider extensions use dedicated fields; other model implementations can use `Extension`.

## ContentBlock

`ContentBlock` is the smallest content unit of `AgenticMessage`. A message can contain multiple ordered blocks to express multiple reasoning segments, text, multimodal output, and tool events within a single response.

```go
type ContentBlock struct {
    Type ContentBlockType

    Reasoning *Reasoning

    UserInputText  *UserInputText
    UserInputImage *UserInputImage
    UserInputAudio *UserInputAudio
    UserInputVideo *UserInputVideo
    UserInputFile  *UserInputFile

    AssistantGenText  *AssistantGenText
    AssistantGenImage *AssistantGenImage
    AssistantGenAudio *AssistantGenAudio
    AssistantGenVideo *AssistantGenVideo

    FunctionToolCall   *FunctionToolCall
    FunctionToolResult *FunctionToolResult

    ToolSearchFunctionToolResult *ToolSearchFunctionToolResult

    ServerToolCall   *ServerToolCall
    ServerToolResult *ServerToolResult

    MCPToolCall             *MCPToolCall
    MCPToolResult           *MCPToolResult
    MCPListToolsResult      *MCPListToolsResult
    MCPToolApprovalRequest  *MCPToolApprovalRequest
    MCPToolApprovalResponse *MCPToolApprovalResponse

    StreamingMeta *StreamingMeta
    Extra         map[string]any
}
```

<table>
<tr><td>Type</td><td>Constant</td><td>Corresponding Field</td><td>Description</td></tr>
<tr><td>reasoning</td><td><pre>ContentBlockTypeReasoning</pre></td><td><pre>Reasoning</pre></td><td>Model reasoning summary or raw reasoning text</td></tr>
<tr><td>User Input</td><td><pre>ContentBlockTypeUserInputText/Image/Audio/Video/File</pre></td><td><pre>UserInput*</pre></td><td>User-side text and multimodal input</td></tr>
<tr><td>Model Output</td><td><pre>ContentBlockTypeAssistantGenText/Image/Audio/Video</pre></td><td><pre>AssistantGen*</pre></td><td>Text or multimodal content generated by the model</td></tr>
<tr><td>Function Tool Call</td><td><pre>ContentBlockTypeFunctionToolCall</pre></td><td><pre>FunctionToolCall</pre></td><td>Local function tool call generated by the provider</td></tr>
<tr><td>Function Tool Result</td><td><pre>ContentBlockTypeFunctionToolResult</pre></td><td><pre>FunctionToolResult</pre></td><td>Result returned to the model after user-side execution of a function tool</td></tr>
<tr><td>Tool Search Result</td><td><pre>ContentBlockTypeToolSearchResult</pre></td><td><pre>ToolSearchFunctionToolResult</pre></td><td>Tool definitions discovered and loaded by client-side tool search</td></tr>
<tr><td>Server Tool</td><td><pre>ContentBlockTypeServerToolCall/Result</pre></td><td><pre>ServerToolCall/Result</pre></td><td>Provider server-side executed built-in tools, e.g., web search</td></tr>
<tr><td>MCP Tool</td><td><pre>ContentBlockTypeMCPToolCall/Result/ListToolsResult</pre></td><td><pre>MCP*</pre></td><td>Provider-side hosted MCP tool calls, results, and tool lists</td></tr>
<tr><td>MCP Approval</td><td><pre>ContentBlockTypeMCPToolApprovalRequest/Response</pre></td><td><pre>MCPToolApproval*</pre></td><td>Human approval requests and responses before MCP tool execution</td></tr>
</table>

> 💡
> Currently, the JSON tag of the `ToolSearchFunctionToolResult` field is `tool_search_function_tool_result`, but the string value of `ContentBlockTypeToolSearchResult` is `tool_search_result`. Documentation and business logic should follow the source code definitions.

## Common Content Structures

### Reasoning

```go
type Reasoning struct {
    Text      string
    Signature string
}
```

`Signature` is used for scenarios where certain providers require encrypted reasoning tokens to be passed back.

### User Input Content

```go
type UserInputText struct {
    Text string
}

type UserInputImage struct {
    URL        string
    Base64Data string
    MIMEType   string
    Detail     ImageURLDetail
}

type UserInputFile struct {
    URL        string
    Name       string
    Base64Data string
    MIMEType   string
}
```

`UserInputAudio` and `UserInputVideo` similarly support `URL`, `Base64Data`, and `MIMEType`.

### Model Generated Content

```go
type AssistantGenText struct {
    Text            string
    OpenAIExtension *openai.AssistantGenTextExtension
    ClaudeExtension *claude.AssistantGenTextExtension
    Extension       any
}
```

`AssistantGenImage`, `AssistantGenAudio`, and `AssistantGenVideo` support `URL`, `Base64Data`, and `MIMEType`.

### Function tool call

```go
type FunctionToolCall struct {
    CallID    string
    Name      string
    Arguments string
}
```

`Arguments` is a JSON string generated by the model and passed to `AgenticToolsNode` for execution.

### Function tool result

In the current implementation, `FunctionToolResult` no longer uses `Result string` but instead uses `Content []*FunctionToolResultContentBlock` to uniformly carry text and multimodal tool results.

```go
type FunctionToolResult struct {
    CallID  string
    Name    string
    Content []*FunctionToolResultContentBlock
}

type FunctionToolResultContentBlock struct {
    Type  FunctionToolResultContentBlockType
    Text  *UserInputText
    Image *UserInputImage
    Audio *UserInputAudio
    Video *UserInputVideo
    File  *UserInputFile
    Extra  map[string]any
}
```

### Server tool

```go
type ServerToolCall struct {
    Name      string
    CallID    string
    Arguments any
}

type ServerToolResult struct {
    Name    string
    CallID  string
    Content any
}
```

Server tools are executed by the model server side. The specific structure of `Arguments` and `Content` is determined by the component implementation and provider protocol.

### MCP tool

```go
type MCPToolCall struct {
    ServerLabel       string
    ApprovalRequestID string
    CallID            string
    Name              string
    Arguments         string
}

type MCPToolResult struct {
    ServerLabel string
    CallID      string
    Name        string
    Content     string
    Error       *MCPToolCallError
}
```

MCP approval-related structures:

```go
type MCPToolApprovalRequest struct {
    ID          string
    Name        string
    Arguments   string
    ServerLabel string
}

type MCPToolApprovalResponse struct {
    ApprovalRequestID string
    Approve           bool
    Reason            string
}
```

## Helper Functions

### Shortcut Messages

```go
msg := schema.SystemAgenticMessage("You are a helpful assistant.")
msg := schema.UserAgenticMessage("Analyze this file.")
```

Both `SystemAgenticMessage` and `UserAgenticMessage` create an `AgenticMessage` with a `UserInputText` block.

### NewContentBlock

```go
block := schema.NewContentBlock(&schema.AssistantGenText{
    Text: "done",
})

chunk := schema.NewContentBlockChunk(
    &schema.AssistantGenText{Text: "partial"},
    &schema.StreamingMeta{Index: 0},
)
```

`NewContentBlock` automatically sets the `ContentBlock.Type` and corresponding field based on the specific content type passed in; unsupported types return `nil`.

## Collaboration with AgenticToolsNode

When `AgenticModel` generates local function tool calls, it produces `FunctionToolCall` blocks in the assistant message. `compose.AgenticToolsNode` reads these blocks, executes the corresponding `tool.BaseTool`, and converts the results into `FunctionToolResult` blocks with role `user` to return to the model.

```go
input := &schema.AgenticMessage{
    Role: schema.AgenticRoleTypeAssistant,
    ContentBlocks: []*schema.ContentBlock{
        schema.NewContentBlock(&schema.FunctionToolCall{
            CallID:    "call_1",
            Name:      "get_weather",
            Arguments: `{"city":"Beijing"}`,
        }),
    },
}

results, err := toolsNode.Invoke(ctx, input)
```

Output structure illustration:

```go
&schema.AgenticMessage{
    Role: schema.AgenticRoleTypeUser,
    ContentBlocks: []*schema.ContentBlock{
        schema.NewContentBlock(&schema.FunctionToolResult{
            CallID: "call_1",
            Name:   "get_weather",
            Content: []*schema.FunctionToolResultContentBlock{
                {
                    Type: schema.FunctionToolResultContentBlockTypeText,
                    Text: &schema.UserInputText{Text: "sunny"},
                },
            },
        }),
    },
}
```

## Minimal Usage Example

```go
messages := []*schema.AgenticMessage{
    schema.SystemAgenticMessage("You are a concise assistant."),
    schema.UserAgenticMessage("Search recent Go release notes and summarize them."),
}

resp, err := m.Generate(ctx, messages,
    model.WithTools([]*schema.ToolInfo{searchToolInfo}),
    model.WithAgenticToolChoice(&schema.AgenticToolChoice{
        Type: schema.ToolChoiceAllowed,
    }),
)
if err != nil {
    return err
}

for _, block := range resp.ContentBlocks {
    if block.Type == schema.ContentBlockTypeAssistantGenText && block.AssistantGenText != nil {
        fmt.Println(block.AssistantGenText.Text)
    }
}
```

## Best Practices

- Prefer using `schema.NewContentBlock` to construct blocks to avoid inconsistency between `Type` and content fields.
- Tool results should be expressed using `FunctionToolResult.Content`; avoid flattening multimodal results into strings.
- Provider-specific response information should be placed in the corresponding extension fields or `Extension`; do not write them into the general fields of the main structure.
- In streaming output, `StreamingMeta.Index` can be used to indicate the block index in the final response that the current chunk corresponds to.
- Do not assume a `tool` role exists in documentation and business code; tool events in the agentic path are carried by `ContentBlock`.
