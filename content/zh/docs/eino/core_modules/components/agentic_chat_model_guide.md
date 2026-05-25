---
Description: ""
date: "2026-05-25"
lastmod: ""
tags: []
title: AgenticModel 使用说明[Beta]
weight: 10
---

> 💡
> 本功能在 [v0.9](https://github.com/cloudwego/eino/releases/tag/v0.9.0-alpha.2) 版本开始提供。本文以当前 main 分支源码为准，代码位置主要包括 `components/model/interface.go`、`components/model/option.go` 与 `schema/agentic_message.go`。

## 基本介绍

`AgenticModel` 是 Eino 面向 agentic provider API 的模型组件抽象。它使用 `*schema.AgenticMessage` 作为消息载体，通过有序的 `ContentBlock` 表达文本、reasoning、多模态内容、函数工具调用、服务端内置工具调用、MCP 工具调用和审批结果。

与传统 `ChatModel` 相比，`AgenticModel` 更适合直接承接 OpenAI Responses API、Claude API、Gemini API 等 provider 原生的 agentic 能力：一次模型请求中可能包含多段 reasoning、多次工具调用、服务端工具结果或 MCP 审批信息。Eino 不再把这些内容压平成单一文本字段，而是保留为结构化块。

## 与 ChatModel 的关系

<table>
<tr><td>维度</td><td>AgenticModel</td><td>ChatModel</td></tr>
<tr><td>基础接口</td><td><pre>model.BaseModel[*schema.AgenticMessage]</pre></td><td><pre>model.BaseModel[*schema.Message]</pre></td></tr>
<tr><td>类型别名</td><td><pre>type AgenticModel = BaseModel[*schema.AgenticMessage]</pre></td><td><pre>type BaseChatModel = BaseModel[*schema.Message]</pre></td></tr>
<tr><td>消息结构</td><td><pre>AgenticMessage.ContentBlocks</pre></td><td><pre>Message.Content</pre>、<pre>ToolCalls</pre>、<pre>ToolCallID</pre> 等字段</td></tr>
<tr><td>工具绑定</td><td>调用时通过 <pre>model.WithTools(...)</pre>、<pre>model.WithAgenticToolChoice(...)</pre> 传入</td><td><pre>ToolCallingChatModel.WithTools(...)</pre> 或调用时 Option</td></tr>
<tr><td>表达能力</td><td>reasoning、多模态输入输出、function tool、server tool、MCP tool、approval</td><td>传统 chat completion 消息与 tool call</td></tr>
<tr><td>下游执行器</td><td><pre>compose.AgenticToolsNode</pre></td><td><pre>compose.ToolsNode</pre></td></tr>
</table>

> 💡
> `AgenticModel` 没有 `WithTools` 方法。`WithTools` 是 `ToolCallingChatModel` 的实例方法；agentic 路径的工具信息通过 `model.WithTools(tools)` 作为请求级 Option 传入。

## 组件定义

### BaseModel 与 AgenticModel

代码位置：`components/model/interface.go`

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

`Generate` 阻塞直到模型返回完整响应。

```go
Generate(ctx context.Context, input []*schema.AgenticMessage, opts ...model.Option) (*schema.AgenticMessage, error)
```

参数说明：

<table>
<tr><td>参数</td><td>说明</td></tr>
<tr><td><pre>ctx</pre></td><td>请求上下文，也承载 callback manager 等运行时信息</td></tr>
<tr><td><pre>input</pre></td><td>按时间顺序排列的 agentic message 列表</td></tr>
<tr><td><pre>opts</pre></td><td>请求级模型配置，例如温度、工具、tool choice、模型名等</td></tr>
</table>

### Stream

`Stream` 返回增量消息流。调用方负责读取并关闭 `StreamReader`。

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
> `schema.StreamReader` 只能消费一次。如需多个消费者读取同一个流，应在读取前复制 reader，而不是在消费后复用。

## 请求级 Option

代码位置：`components/model/option.go`

<table>
<tr><td>Option</td><td>适用范围</td><td>说明</td></tr>
<tr><td><pre>model.WithTools(tools)</pre></td><td>通用，agentic 路径常用</td><td>设置本次请求可被模型调用的工具定义；<pre>nil</pre> 会被规范化为空切片</td></tr>
<tr><td><pre>model.WithDeferredTools(tools)</pre></td><td>AgenticModel</td><td>注册可被模型内置 tool search 延迟加载的工具；不要同时放入 <pre>WithTools</pre></td></tr>
<tr><td><pre>model.WithToolSearchTool(tool)</pre></td><td>AgenticModel</td><td>注册模型用于检索 deferred tools 的 tool search 工具；不要放入 <pre>WithTools</pre></td></tr>
<tr><td><pre>model.WithAgenticToolChoice(choice)</pre></td><td>AgenticModel</td><td>控制 agentic 模型如何选择 function/MCP/server tool</td></tr>
<tr><td><pre>model.WithToolChoice(choice, names...)</pre></td><td>ChatModel</td><td>仅适用于 ChatModel 的 tool choice</td></tr>
</table>

`AgenticToolChoice` 定义在 `schema/tool.go`：

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

代码位置：`schema/agentic_message.go`

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

`AgenticMessage` 只有 `system`、`user`、`assistant` 三类 role。工具调用和工具结果不是独立 role，而是由 `ContentBlock` 表达。

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

`TokenUsage` 是通用 token 统计；OpenAI、Gemini、Claude 的 provider 扩展使用专门字段；其他模型实现可以放入 `Extension`。

## ContentBlock

`ContentBlock` 是 `AgenticMessage` 的最小内容单元。一个 message 可以包含多个有序 block，用于表达一次响应中的多段 reasoning、文本、多模态输出和工具事件。

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
<tr><td>类型</td><td>常量</td><td>对应字段</td><td>说明</td></tr>
<tr><td>reasoning</td><td><pre>ContentBlockTypeReasoning</pre></td><td><pre>Reasoning</pre></td><td>模型 reasoning 摘要或原始 reasoning 文本</td></tr>
<tr><td>用户输入</td><td><pre>ContentBlockTypeUserInputText/Image/Audio/Video/File</pre></td><td><pre>UserInput*</pre></td><td>用户侧文本与多模态输入</td></tr>
<tr><td>模型输出</td><td><pre>ContentBlockTypeAssistantGenText/Image/Audio/Video</pre></td><td><pre>AssistantGen*</pre></td><td>模型生成的文本或多模态内容</td></tr>
<tr><td>函数工具调用</td><td><pre>ContentBlockTypeFunctionToolCall</pre></td><td><pre>FunctionToolCall</pre></td><td>provider 生成的本地 function tool call</td></tr>
<tr><td>函数工具结果</td><td><pre>ContentBlockTypeFunctionToolResult</pre></td><td><pre>FunctionToolResult</pre></td><td>用户侧执行 function tool 后返回给模型的结果</td></tr>
<tr><td>tool search 结果</td><td><pre>ContentBlockTypeToolSearchResult</pre></td><td><pre>ToolSearchFunctionToolResult</pre></td><td>客户端 tool search 发现并加载的工具定义</td></tr>
<tr><td>服务端工具</td><td><pre>ContentBlockTypeServerToolCall/Result</pre></td><td><pre>ServerToolCall/Result</pre></td><td>provider 服务端执行的内置工具，例如 web search</td></tr>
<tr><td>MCP 工具</td><td><pre>ContentBlockTypeMCPToolCall/Result/ListToolsResult</pre></td><td><pre>MCP*</pre></td><td>provider 侧托管的 MCP 工具调用、结果与工具列表</td></tr>
<tr><td>MCP 审批</td><td><pre>ContentBlockTypeMCPToolApprovalRequest/Response</pre></td><td><pre>MCPToolApproval*</pre></td><td>MCP 工具执行前的人类审批请求和响应</td></tr>
</table>

> 💡
> 当前 `ToolSearchFunctionToolResult` 字段的 JSON tag 是 `tool_search_function_tool_result`，但 `ContentBlockTypeToolSearchResult` 的字符串值是 `tool_search_result`。文档和业务逻辑应以源码定义为准。

## 常用内容结构

### Reasoning

```go
type Reasoning struct {
    Text      string
    Signature string
}
```

`Signature` 用于某些 provider 的加密 reasoning token 回传场景。

### 用户输入内容

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

`UserInputAudio` 与 `UserInputVideo` 同样支持 `URL`、`Base64Data`、`MIMEType`。

### 模型生成内容

```go
type AssistantGenText struct {
    Text            string
    OpenAIExtension *openai.AssistantGenTextExtension
    ClaudeExtension *claude.AssistantGenTextExtension
    Extension       any
}
```

`AssistantGenImage`、`AssistantGenAudio`、`AssistantGenVideo` 支持 `URL`、`Base64Data`、`MIMEType`。

### Function tool call

```go
type FunctionToolCall struct {
    CallID    string
    Name      string
    Arguments string
}
```

`Arguments` 是 JSON 字符串，由模型生成并交给 `AgenticToolsNode` 执行。

### Function tool result

当前实现中，`FunctionToolResult` 不再使用 `Result string`，而是使用 `Content []*FunctionToolResultContentBlock`，以统一承载文本和多模态工具结果。

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

Server tool 由模型服务端执行，`Arguments` 与 `Content` 的具体结构由组件实现和 provider 协议决定。

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

MCP 审批相关结构：

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

## 构造辅助函数

### 快捷消息

```go
msg := schema.SystemAgenticMessage("You are a helpful assistant.")
msg := schema.UserAgenticMessage("Analyze this file.")
```

`SystemAgenticMessage` 与 `UserAgenticMessage` 都会创建一个带 `UserInputText` block 的 `AgenticMessage`。

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

`NewContentBlock` 会根据传入的具体内容类型自动设置 `ContentBlock.Type` 与对应字段；不支持的类型会返回 `nil`。

## 与 AgenticToolsNode 的协作

`AgenticModel` 生成本地函数工具调用时，会在 assistant message 中产生 `FunctionToolCall` block。`compose.AgenticToolsNode` 会读取这些 block，执行对应 `tool.BaseTool`，再把结果转换成 role 为 `user` 的 `FunctionToolResult` block 返回给模型。

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

输出结构示意：

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

## 最小调用示例

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

## 使用建议

- 优先使用 `schema.NewContentBlock` 构造 block，避免 `Type` 与内容字段不一致。
- 工具结果应使用 `FunctionToolResult.Content` 表达，避免把多模态结果压平成字符串。
- Provider 专属响应信息应放入对应扩展字段或 `Extension`，不要写入主结构的通用字段。
- 流式输出中可以通过 `StreamingMeta.Index` 表示当前 chunk 对应最终响应中的 block 索引。
- 文档和业务代码中不要假设存在 `tool` role；agentic 路径的工具事件由 `ContentBlock` 承载。
