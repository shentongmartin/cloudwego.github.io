---
Description: ""
date: "2026-05-25"
lastmod: ""
tags: []
title: AgenticChatTemplate 使用说明[Beta]
weight: 11
---

> 💡
> 本功能在 [v0.9](https://github.com/cloudwego/eino/releases/tag/v0.9.0-alpha.2) 版本开始提供。本文以当前 main 分支源码为准，代码位置主要包括 `components/prompt`、`schema/agentic_message.go` 与 `compose`。

## 基本介绍

`AgenticChatTemplate` 是面向 `*schema.AgenticMessage` 的 Prompt 组件抽象，用于把 `map[string]any` 中的变量填充到 agentic message 模板中，并输出 `[]*schema.AgenticMessage` 供 `AgenticModel` 或后续编排节点使用。

它与已有 `ChatTemplate` 的设计基本一致，但消息类型从 `*schema.Message` 切换为 `*schema.AgenticMessage`，并且模板渲染只作用于用户输入类内容块。

## 组件定义

代码位置：`components/prompt/interface.go`

```go
type AgenticChatTemplate interface {
    Format(ctx context.Context, vs map[string]any, opts ...Option) ([]*schema.AgenticMessage, error)
}
```

`Format` 参数说明：

<table>
<tr><td>参数</td><td>说明</td></tr>
<tr><td><pre>ctx</pre></td><td>请求上下文，也承载 callback manager 等运行时信息</td></tr>
<tr><td><pre>vs</pre></td><td>模板变量表，key 为占位符名称，value 为实际值</td></tr>
<tr><td><pre>opts</pre></td><td>Prompt 组件 Option，支持实现侧自定义扩展</td></tr>
</table>

返回值：

<table>
<tr><td>返回值</td><td>说明</td></tr>
<tr><td><pre>[]*schema.AgenticMessage</pre></td><td>渲染后的 agentic message 列表</td></tr>
<tr><td><pre>error</pre></td><td>缺少变量、变量类型不匹配或模板渲染失败时返回错误</td></tr>
</table>

## 构造方式

### FromAgenticMessages

代码位置：`components/prompt/agentic_chat_template.go`

```go
func FromAgenticMessages(formatType schema.FormatType, templates ...schema.AgenticMessagesTemplate) *DefaultAgenticChatTemplate
```

`FromAgenticMessages` 不返回 `error`。它接收模板格式和一组 `schema.AgenticMessagesTemplate`，返回默认实现 `*DefaultAgenticChatTemplate`。

```go
template := prompt.FromAgenticMessages(schema.FString,
    schema.SystemAgenticMessage("You are a {role}."),
    schema.AgenticMessagesPlaceholder("history", true),
    schema.UserAgenticMessage("Please help me {task}."),
)
```

### 支持的 FormatType

<table>
<tr><td>格式</td><td>常量</td><td>占位符示例</td><td>适用场景</td></tr>
<tr><td>FString</td><td><pre>schema.FString</pre></td><td><pre>{role}</pre></td><td>简单变量替换</td></tr>
<tr><td>GoTemplate</td><td><pre>schema.GoTemplate</pre></td><td><pre>{{.role}}</pre></td><td>需要 Go <pre>text/template</pre> 能力的场景</td></tr>
<tr><td>Jinja2</td><td><pre>schema.Jinja2</pre></td><td><pre>{{ role }}</pre></td><td>需要 Jinja2 语法的场景</td></tr>
</table>

## AgenticMessagesTemplate

代码位置：`schema/agentic_message.go`

```go
type AgenticMessagesTemplate interface {
    Format(ctx context.Context, vs map[string]any, formatType FormatType) ([]*AgenticMessage, error)
}
```

当前常用实现包括：

<table>
<tr><td>构造方式</td><td>说明</td></tr>
<tr><td><pre>&schema.AgenticMessage{...}</pre></td><td><pre>AgenticMessage</pre> 自身实现了 <pre>AgenticMessagesTemplate</pre></td></tr>
<tr><td><pre>schema.SystemAgenticMessage(text)</pre></td><td>构造 <pre>system</pre> role 消息</td></tr>
<tr><td><pre>schema.UserAgenticMessage(text)</pre></td><td>构造 <pre>user</pre> role 消息</td></tr>
<tr><td><pre>schema.AgenticMessagesPlaceholder(key, optional)</pre></td><td>从变量表中插入一组历史 agentic messages</td></tr>
</table>

### 模板渲染范围

`AgenticMessage.Format` 只格式化用户输入类 block：

- `ContentBlockTypeUserInputText`
- `ContentBlockTypeUserInputImage`
- `ContentBlockTypeUserInputAudio`
- `ContentBlockTypeUserInputVideo`
- `ContentBlockTypeUserInputFile`

模型输出、reasoning、tool call、tool result、MCP 和 server tool 相关 block 不作为 prompt 模板内容渲染。

> 💡
> 推荐用 `schema.SystemAgenticMessage`、`schema.UserAgenticMessage` 和 `schema.NewContentBlock` 构造模板，避免手动设置 `ContentBlock.Type` 时与内容字段不一致。

## Placeholder

代码位置：`schema/agentic_message.go`

```go
func AgenticMessagesPlaceholder(key string, optional bool) AgenticMessagesTemplate
```

行为规则：

<table>
<tr><td>场景</td><td>行为</td></tr>
<tr><td><pre>vs[key]</pre> 存在且类型为 <pre>[]*schema.AgenticMessage</pre></td><td>原样插入该消息列表</td></tr>
<tr><td><pre>vs[key]</pre> 不存在且 <pre>optional=true</pre></td><td>返回空切片，不报错</td></tr>
<tr><td><pre>vs[key]</pre> 不存在且 <pre>optional=false</pre></td><td>返回错误</td></tr>
<tr><td><pre>vs[key]</pre> 类型不是 <pre>[]*schema.AgenticMessage</pre></td><td>返回错误</td></tr>
</table>

示例：

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

## 单独使用

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

## 在编排中使用

### Chain

代码位置：`compose/chain.go`

```go
func (c *Chain[I, O]) AppendAgenticChatTemplate(node prompt.AgenticChatTemplate, opts ...GraphAddNodeOpt) *Chain[I, O]
```

```go
chain := compose.NewChain[map[string]any, *schema.AgenticMessage]()
chain.AppendAgenticChatTemplate(template)
chain.AppendAgenticModel(model)
```

### Graph

代码位置：`compose/graph.go`

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

代码位置：`compose/workflow.go`

```go
func (wf *Workflow[I, O]) AddAgenticChatTemplateNode(key string, chatTemplate prompt.AgenticChatTemplate, opts ...GraphAddNodeOpt) *WorkflowNode
```

### Parallel 与 ChainBranch

Agentic prompt 也可以放入 `Parallel` 或 `ChainBranch`：

```go
func (p *Parallel) AddAgenticChatTemplate(outputKey string, node prompt.AgenticChatTemplate, opts ...GraphAddNodeOpt) *Parallel

func (cb *ChainBranch) AddAgenticChatTemplate(key string, node prompt.AgenticChatTemplate, opts ...GraphAddNodeOpt) *ChainBranch
```

## 从前驱节点获取变量

`AgenticChatTemplate.Format` 需要 `map[string]any`。如果前驱节点输出不是 map，可以在加节点时使用 `compose.WithOutputKey` 把输出包装为单字段 map。

```go
graph.AddLambdaNode("query",
    compose.InvokableLambda(func(ctx context.Context, input string) (string, error) {
        return input, nil
    }),
    compose.WithOutputKey("task"),
)

graph.AddAgenticChatTemplateNode("prompt", template)
```

包装后的 map 形如：

```go
map[string]any{
    "task": previousNodeOutput,
}
```

## Callback

### 组件级 callback payload

代码位置：`components/prompt/agentic_callback_extra.go`

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

`DefaultAgenticChatTemplate.Format` 会在开始和结束时触发 callback，并传递上述 agentic payload。

### utils/callbacks helper

代码位置：`utils/callbacks/template.go`

当前 `AgenticPromptCallbackHandler` 的公开签名如下：

```go
type AgenticPromptCallbackHandler struct {
    OnStart func(ctx context.Context, runInfo *callbacks.RunInfo, input *prompt.CallbackInput) context.Context
    OnEnd   func(ctx context.Context, runInfo *callbacks.RunInfo, output *prompt.CallbackOutput) context.Context
    OnError func(ctx context.Context, runInfo *callbacks.RunInfo, err error) context.Context
}
```

使用 helper 注册：

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
> 如果需要直接处理 `*prompt.AgenticCallbackInput` 或 `*prompt.AgenticCallbackOutput`，应使用组件级 callback payload 和 `prompt.ConvAgenticCallbackInput/Output`。`utils/callbacks.AgenticPromptCallbackHandler` 当前公开签名仍复用 `prompt.CallbackInput/Output`。

## 自定义实现

实现自定义 `AgenticChatTemplate` 只需要满足接口：

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

如需支持自定义 option，可通过 `prompt.WrapImplSpecificOptFn` 和 `prompt.GetImplSpecificOptions` 实现。

## 使用建议

- `FromAgenticMessages` 不返回 `error`，示例中不应写成 `template, err := ...`。
- `AgenticMessagesPlaceholder` 的变量值必须是 `[]*schema.AgenticMessage`。
- 模板变量命名应稳定一致，缺失变量会在运行时返回错误。
- 对历史对话使用 `AgenticMessagesPlaceholder("history", true)`，可以在无历史时自然返回空列表。
- 在 Graph/Chain 中，确保 Agentic prompt 后接收的是 `[]*schema.AgenticMessage` 类型的节点，例如 `AgenticModel`。
