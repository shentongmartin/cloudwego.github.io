---
Description: ""
date: "2026-05-25"
lastmod: ""
tags: []
title: AgenticToolsNode&Tool 使用说明[Beta]
weight: 12
---

> 💡
> 本功能在 [v0.9](https://github.com/cloudwego/eino/releases/tag/v0.9.0-alpha.2) 版本开始提供。本文以当前 main 分支源码为准，代码位置主要包括 `compose/agentic_tools_node.go`、`compose/tool_node.go`、`components/tool/interface.go` 与 `schema/tool.go`。

## 基本介绍

`AgenticToolsNode` 是 Eino 在 agentic message 路径上的工具执行节点。它接收 `*schema.AgenticMessage`，从其中的 `FunctionToolCall` content block 提取工具调用，复用 `ToolsNode` 的执行能力完成本地工具执行，再把结果转换为 `FunctionToolResult` content block 返回。

它不负责“决定调用哪个工具”。工具选择由上游 `AgenticModel` 完成；`AgenticToolsNode` 只根据输入消息中的 function tool call 执行对应工具。

> 💡
> `AgenticToolsNode` 是 `ToolsNode` 的 agentic 包装层，因此复用 `ToolsNodeConfig`、`ToolsNodeOption`、工具别名、异常处理、执行顺序、参数处理器和 tool middleware 等能力。

## AgenticToolsNode 定义

代码位置：`compose/agentic_tools_node.go`

```go
func NewAgenticToolsNode(ctx context.Context, conf *ToolsNodeConfig) (*AgenticToolsNode, error)

type AgenticToolsNode struct {
    inner *ToolsNode
}

func (a *AgenticToolsNode) Invoke(ctx context.Context, input *schema.AgenticMessage, opts ...ToolsNodeOption) ([]*schema.AgenticMessage, error)

func (a *AgenticToolsNode) Stream(ctx context.Context, input *schema.AgenticMessage, opts ...ToolsNodeOption) (*schema.StreamReader[[]*schema.AgenticMessage], error)
```

输入输出规则：

<table>
<tr><td>方法</td><td>输入</td><td>输出</td></tr>
<tr><td><pre>Invoke</pre></td><td>assistant message，包含一个或多个 <pre>FunctionToolCall</pre> block</td><td>多个 user message，每个包含一个 <pre>FunctionToolResult</pre> 或 <pre>ToolSearchFunctionToolResult</pre> block</td></tr>
<tr><td><pre>Stream</pre></td><td>assistant message，包含一个或多个 <pre>FunctionToolCall</pre> block</td><td><pre>StreamReader</pre>，每个 chunk 是 <pre>[]*schema.AgenticMessage</pre></td></tr>
</table>

## 执行流程

<a href="/img/eino/RHFww5teMhOfEybBazKcMpWynIb.png" target="_blank"><img src="/img/eino/RHFww5teMhOfEybBazKcMpWynIb.png" width="100%" /></a>

核心转换逻辑：

- 只处理 `ContentBlockTypeFunctionToolCall` 且 `FunctionToolCall != nil` 的 block。
- `FunctionToolCall.CallID` 会转换为 `schema.ToolCall.ID`，用于关联工具结果。
- 标准工具输出会转换为 `FunctionToolResult.Content` 中的 text block。
- enhanced 工具输出会转换为 `FunctionToolResult.Content` 中的 text/image/audio/video/file block。
- tool search 结果会转换为 `ToolSearchFunctionToolResult` block。
- 输出 message 的 role 为 `schema.AgenticRoleTypeUser`。

## ToolsNodeConfig

代码位置：`compose/tool_node.go`

```go
type ToolsNodeConfig struct {
    Tools                []tool.BaseTool
    ToolAliases          map[string]ToolAliasConfig
    UnknownToolsHandler  func(ctx context.Context, name, input string) (string, error)
    ExecuteSequentially  bool
    ToolArgumentsHandler func(ctx context.Context, name, arguments string) (string, error)
    ToolCallMiddlewares  []ToolMiddleware
}
```

<table>
<tr><td>字段</td><td>说明</td></tr>
<tr><td><pre>Tools</pre></td><td>可执行工具列表。每个工具必须实现 <pre>tool.BaseTool</pre>，并至少实现一种执行接口</td></tr>
<tr><td><pre>ToolAliases</pre></td><td>工具名别名和参数别名配置，用于兼容模型生成的非标准名称</td></tr>
<tr><td><pre>UnknownToolsHandler</pre></td><td>处理模型幻觉出的未知工具；未设置时未知工具会报错</td></tr>
<tr><td><pre>ExecuteSequentially</pre></td><td><pre>true</pre> 表示按 tool call 顺序串行执行；默认 <pre>false</pre> 并行执行</td></tr>
<tr><td><pre>ToolArgumentsHandler</pre></td><td>参数别名处理后、工具执行前调用，可做参数清洗或补全</td></tr>
<tr><td><pre>ToolCallMiddlewares</pre></td><td>工具执行 middleware，支持标准与 enhanced、非流式与流式工具</td></tr>
</table>

### ToolsNodeOption

```go
type ToolsNodeOption func(o *toolsNodeOptions)

func WithToolOption(opts ...tool.Option) ToolsNodeOption
func WithToolList(tool ...tool.BaseTool) ToolsNodeOption
func WithToolAliases(toolAliases map[string]ToolAliasConfig) ToolsNodeOption
```

`WithToolList` 和 `WithToolAliases` 是调用级覆盖配置：

- `WithToolList`：为本次调用替换工具列表。
- `WithToolAliases` 搭配 `WithToolList`：为动态工具列表配置 alias。
- `WithToolAliases` 单独使用：替换全局 alias 配置，但保留原工具列表。

## Tool alias

代码位置：`compose/tool_node.go`

```go
type ToolAliasConfig struct {
    NameAliases      []string
    ArgumentsAliases map[string][]string
}
```

语义说明：

<table>
<tr><td>配置</td><td>说明</td></tr>
<tr><td><pre>NameAliases</pre></td><td>工具名别名。模型返回别名时，会解析为 canonical tool name</td></tr>
<tr><td><pre>ArgumentsAliases</pre></td><td>参数别名。key 为 canonical 参数名，value 为别名列表</td></tr>
</table>

示例：

```go
toolsNode, err := compose.NewAgenticToolsNode(ctx, &compose.ToolsNodeConfig{
    Tools: []tool.BaseTool{searchTool},
    ToolAliases: map[string]compose.ToolAliasConfig{
        "search": {
            NameAliases: []string{"web_search", "lookup"},
            ArgumentsAliases: map[string][]string{
                "query": {"q", "keyword"},
                "limit": {"count", "max_results"},
            },
        },
    },
})
```

参数 alias 处理规则：

- 只处理顶层 JSON object 的 key。
- 空字符串、非 object、JSON 解析失败时原样返回。
- canonical key 已存在时，不用 alias 覆盖 canonical value。
- alias remap 发生在 `ToolArgumentsHandler` 之前。
- alias 配置会校验空别名、空 canonical key、canonical key 包含 `.`、alias 与 schema property 冲突，以及同一 alias 映射到多个 canonical 的情况。

## Tool 接口

Eino 当前没有导出名为 `Tool` 的统一接口。工具能力由 `components/tool` 包中的以下接口组成。

代码位置：`components/tool/interface.go`

```go
type BaseTool interface {
    Info(ctx context.Context) (*schema.ToolInfo, error)
}

type InvokableTool interface {
    BaseTool
    InvokableRun(ctx context.Context, argumentsInJSON string, opts ...Option) (string, error)
}

type StreamableTool interface {
    BaseTool
    StreamableRun(ctx context.Context, argumentsInJSON string, opts ...Option) (*schema.StreamReader[string], error)
}

type EnhancedInvokableTool interface {
    BaseTool
    InvokableRun(ctx context.Context, toolArgument *schema.ToolArgument, opts ...Option) (*schema.ToolResult, error)
}

type EnhancedStreamableTool interface {
    BaseTool
    StreamableRun(ctx context.Context, toolArgument *schema.ToolArgument, opts ...Option) (*schema.StreamReader[*schema.ToolResult], error)
}
```

接口分层：

<table>
<tr><td>接口</td><td>用途</td></tr>
<tr><td><pre>BaseTool</pre></td><td>提供工具元信息，可传给模型用于识别工具</td></tr>
<tr><td><pre>InvokableTool</pre></td><td>非流式执行，入参为 JSON 字符串，返回字符串</td></tr>
<tr><td><pre>StreamableTool</pre></td><td>流式执行，入参为 JSON 字符串，返回字符串流</td></tr>
<tr><td><pre>EnhancedInvokableTool</pre></td><td>非流式 enhanced 工具，入参为 <pre>*schema.ToolArgument</pre>，返回多模态 <pre>*schema.ToolResult</pre></td></tr>
<tr><td><pre>EnhancedStreamableTool</pre></td><td>流式 enhanced 工具，返回 <pre>*schema.ToolResult</pre> 流</td></tr>
</table>

> 💡
> 如果一个工具同时实现标准接口和 enhanced 接口，`ToolsNode` 会优先使用 enhanced 接口，以保留结构化多模态结果。

## ToolInfo

代码位置：`schema/tool.go`

```go
type ToolInfo struct {
    Name  string
    Desc  string
    Extra map[string]any
    *ParamsOneOf
}
```

字段说明：

<table>
<tr><td>字段</td><td>说明</td></tr>
<tr><td><pre>Name</pre></td><td>工具唯一名称，应简洁且在工具集合内唯一</td></tr>
<tr><td><pre>Desc</pre></td><td>告诉模型何时、为何、如何使用该工具；可以包含少量示例</td></tr>
<tr><td><pre>Extra</pre></td><td>额外元信息，由组件实现或业务自定义</td></tr>
<tr><td><pre>ParamsOneOf</pre></td><td>参数定义，可通过 <pre>schema.NewParamsOneOfByParams</pre> 或 <pre>schema.NewParamsOneOfByJSONSchema</pre> 构造；为 <pre>nil</pre> 表示无参数</td></tr>
</table>

## ToolArgument 与 ToolResult

Enhanced 工具使用结构化入参与多模态输出。

```go
type ToolArgument struct {
    Text string
}

type ToolResult struct {
    Parts []ToolOutputPart
}
```

`ToolOutputPart` 可表示 text、image、audio、video、file 和 tool search result。`AgenticToolsNode` 会将 enhanced output 转换为 `FunctionToolResult.Content` 中的多模态 block。

## FunctionToolCall 与 FunctionToolResult

`AgenticToolsNode` 的输入来自 `schema.AgenticMessage` 中的 `FunctionToolCall` block：

```go
type FunctionToolCall struct {
    CallID    string
    Name      string
    Arguments string
}
```

工具执行结果会被转换为 `FunctionToolResult`：

```go
type FunctionToolResult struct {
    CallID  string
    Name    string
    Content []*FunctionToolResultContentBlock
}
```

`FunctionToolResultContentBlock` 支持 text、image、audio、video、file 五类内容。当前实现不再使用 `Result string` 字段。

## 单独使用示例

```go
toolsNode, err := compose.NewAgenticToolsNode(ctx, &compose.ToolsNodeConfig{
    Tools: []tool.BaseTool{weatherTool},
})
if err != nil {
    return err
}

input := &schema.AgenticMessage{
    Role: schema.AgenticRoleTypeAssistant,
    ContentBlocks: []*schema.ContentBlock{
        schema.NewContentBlock(&schema.FunctionToolCall{
            CallID:    "call_1",
            Name:      "get_weather",
            Arguments: `{"city":"Shenzhen","date":"tomorrow"}`,
        }),
    },
}

messages, err := toolsNode.Invoke(ctx, input)
if err != nil {
    return err
}
```

输出示意：

```go
[]*schema.AgenticMessage{
    {
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
    },
}
```

## 在编排中使用

### Chain

代码位置：`compose/chain.go`

```go
func (c *Chain[I, O]) AppendAgenticToolsNode(node *AgenticToolsNode, opts ...GraphAddNodeOpt) *Chain[I, O]
```

```go
chain := compose.NewChain[*schema.AgenticMessage, []*schema.AgenticMessage]()
chain.AppendAgenticToolsNode(toolsNode)
```

### Graph

代码位置：`compose/graph.go`

```go
func (g *graph) AddAgenticToolsNode(key string, node *AgenticToolsNode, opts ...GraphAddNodeOpt) error
```

```go
graph := compose.NewGraph[*schema.AgenticMessage, []*schema.AgenticMessage]()
err := graph.AddAgenticToolsNode("tools", toolsNode)
if err != nil {
    return err
}
```

### Workflow、Parallel、ChainBranch

```go
func (wf *Workflow[I, O]) AddAgenticToolsNode(key string, tools *AgenticToolsNode, opts ...GraphAddNodeOpt) *WorkflowNode

func (p *Parallel) AddAgenticToolsNode(outputKey string, node *AgenticToolsNode, opts ...GraphAddNodeOpt) *Parallel

func (cb *ChainBranch) AddAgenticToolsNode(key string, node *AgenticToolsNode, opts ...GraphAddNodeOpt) *ChainBranch
```

## Callback

代码位置：`utils/callbacks/template.go`

```go
type AgenticToolsNodeCallbackHandlers struct {
    OnStart               func(ctx context.Context, info *callbacks.RunInfo, input *schema.AgenticMessage) context.Context
    OnEnd                 func(ctx context.Context, info *callbacks.RunInfo, input []*schema.AgenticMessage) context.Context
    OnEndWithStreamOutput func(ctx context.Context, info *callbacks.RunInfo, output *schema.StreamReader[[]*schema.AgenticMessage]) context.Context
    OnError               func(ctx context.Context, info *callbacks.RunInfo, err error) context.Context
}
```

注册示例：

```go
handler := callbackHelper.NewHandlerHelper().
    AgenticToolsNode(&callbackHelper.AgenticToolsNodeCallbackHandlers{
        OnStart: func(ctx context.Context, info *callbacks.RunInfo, input *schema.AgenticMessage) context.Context {
            fmt.Printf("tool call blocks: %d\n", len(input.ContentBlocks))
            return ctx
        },
        OnEnd: func(ctx context.Context, info *callbacks.RunInfo, output []*schema.AgenticMessage) context.Context {
            fmt.Printf("tool results: %d\n", len(output))
            return ctx
        },
    }).
    Handler()

result, err := runnable.Invoke(ctx, input, compose.WithCallbacks(handler))
```

## 获取 ToolCallID

代码位置：`compose/tool_node.go`

```go
func GetToolCallID(ctx context.Context) string
```

在工具函数体或 tool callback handler 中，可以通过 `compose.GetToolCallID(ctx)` 获取当前 tool call id。

```go
func (t *MyTool) InvokableRun(ctx context.Context, args string, opts ...tool.Option) (string, error) {
    callID := compose.GetToolCallID(ctx)
    // callID 为空表示当前 ctx 不在 tool call 执行上下文中，或上下文值类型不匹配。
    return callID, nil
}
```

## 使用建议

- 文档和代码中应称为 `tool.BaseTool`、`InvokableTool`、`StreamableTool`、`EnhancedInvokableTool`、`EnhancedStreamableTool`，不要写成源码中不存在的导出 `Tool` 接口。
- 如果工具可能返回图片、音频、视频或文件，优先实现 enhanced 接口，避免把多模态结果压平成字符串。
- 多个 tool call 默认并行执行；需要保持顺序或工具间有副作用依赖时，设置 `ExecuteSequentially: true`。
- 使用 `ToolArgumentsHandler` 做参数清洗时，要注意它在参数 alias remap 之后执行。
- `UnknownToolsHandler` 适合兜底模型幻觉出的工具名，但不应掩盖真实配置错误。

## 已有实现

1. Google Search Tool：基于 Google 搜索的工具实现 [Tool - Googlesearch](/zh/docs/eino/ecosystem_integration/tool/tool_googlesearch)
2. DuckDuckGo Search Tool：基于 DuckDuckGo 搜索的工具实现 [Tool - DuckDuckGoSearch](/zh/docs/eino/ecosystem_integration/tool/tool_duckduckgo_search)
3. MCP：把 MCP server 作为 tool [Eino Tool - MCP](/zh/docs/eino/ecosystem_integration/tool/tool_mcp)

## 工具实现方式

- 基于 HTTP API 的 tool 实现：[如何使用 openapi 创建 tool/function call ?](https://bytedance.larkoffice.com/wiki/FjXzwf3exijtKyk2hh7cAmnZn1g)
- 基于 gRPC 的 tool 实现：[如何使用 proto3 创建 tool/function call ? ](https://bytedance.larkoffice.com/wiki/EPkawUVbdiGwxCkWCJTcAMQonbh)
- 基于 thrift 的 tool 实现：[如何使用 thrift idl 创建 tool/function call ? ](https://bytedance.larkoffice.com/wiki/PcHfwo6x0iOrXxkIjJecez8xnNg)
- 基于本地函数的工具实现：[如何创建一个 tool ?](/zh/docs/eino/core_modules/components/tools_node_guide/how_to_create_a_tool)
