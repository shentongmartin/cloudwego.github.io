---
Description: ""
date: "2026-05-17"
lastmod: ""
tags: []
title: DeepAgents
weight: 3
---

> 💡
> This feature requires eino >= v0.5.14.

## Overview

DeepAgents is an out-of-the-box solution built on ChatModelAgent. Without manually assembling prompts, tools, or context management, you can get an Agent with planning, file system, Shell execution, and sub-Agent delegation capabilities, while retaining all of ChatModelAgent's extension capabilities (custom tools, middleware, handlers).

**Built-in Capabilities**:

- **Planning** — `write_todos` tool for task decomposition and progress tracking
- **File System** — `ls`, `read_file`, `write_file`, `edit_file`, `glob`, `grep`
- **Shell** — `execute` (supports streaming)
- **Sub-Agent** — `task` tool delegates tasks to context-isolated sub-agents
- **Smart Defaults** — Built-in Prompts that teach the model to efficiently use tools
- **Context Management** — Large outputs are automatically saved to files

### Import

```go
import "github.com/cloudwego/eino/adk/prebuilt/deep"

agent, err := deep.New(ctx, &deep.Config{
    ChatModel: myModel,
})
```

---

## Full Config Definition

```go
type Config = TypedConfig[*schema.Message]

type TypedConfig[M adk.MessageType] struct {
    Name        string              // Agent identifier name
    Description string              // Purpose description
    ChatModel   model.BaseModel[M]  // Required; must support model.WithTools
    Instruction string              // System prompt; uses built-in default Prompt when empty

    // Sub-Agents (bound to TaskTool)
    SubAgents []adk.TypedAgent[M]

    // Custom tools
    ToolsConfig  adk.ToolsConfig
    MaxIteration int // Maximum reasoning iteration count

    // File system (choose one or combine)
    Backend        filesystem.Backend        // Registers ls/read_file/write_file/edit_file/glob/grep
    Shell          filesystem.Shell          // Registers execute (mutually exclusive with StreamingShell)
    StreamingShell filesystem.StreamingShell  // Registers execute (streaming, mutually exclusive with Shell)

    // Built-in feature toggles
    WithoutWriteTodos      bool // true disables the write_todos tool
    WithoutGeneralSubAgent bool // true disables the default general-purpose sub-Agent

    // TaskTool description generator (customize the task tool's description)
    TaskToolDescriptionGenerator func(ctx context.Context, agents []adk.TypedAgent[M]) (string, error)

    // Extensions
    Middlewares []adk.AgentMiddleware                   // struct-based middleware
    Handlers    []adk.TypedChatModelAgentMiddleware[M]  // interface-based handlers

    // Model fault tolerance
    ModelRetryConfig    *adk.TypedModelRetryConfig[M]
    ModelFailoverConfig *adk.ModelFailoverConfig[M]

    // Output storage (written to session via AddSessionValue)
    OutputKey string
}
```

### Constructors

```go
// Standard version (M = *schema.Message)
func New(ctx context.Context, cfg *Config) (adk.ResumableAgent, error)

// Generic version (supports *schema.AgenticMessage)
func NewTyped[M adk.MessageType](ctx context.Context, cfg *TypedConfig[M]) (adk.TypedResumableAgent[M], error)
```

> 💡
> Returns ResumableAgent (includes Resume method), which can be used with Runner's checkpoint/resume mechanism.

---

## Architecture

<a href="/img/eino/Ifu5bvB6conps5xBH5fcFdiCnCW.png" target="_blank"><img src="/img/eino/Ifu5bvB6conps5xBH5fcFdiCnCW.png" width="100%" /></a>

- **Main Agent**: System entry point, completes tasks by calling tools in ReAct mode
- **ChatModel** (`model.BaseModel[M]`): Responsible for reasoning and tool selection
- **Tools**:
  - `write_todos`: Built-in planning tool, decomposes tasks into structured TODO lists
  - `task`: Sub-Agent invocation entry (routing parameters: `subagent_type`, `description`)
  - Built-in tools (file system/Shell) + user-defined tools (`ToolsConfig`)
- **SubAgents**: Context-isolated, execute subtasks independently
  - `general-purpose`: Default sub-Agent with the same tools (except task) and configuration as the main Agent
  - Custom sub-Agents (`Config.SubAgents`)

---

## Built-in File System

<table>
<tr><td>Config Field</td><td>Registered Tools</td><td>Description</td></tr>
<tr><td><pre>Backend</pre></td><td>ls, read_file, write_file, edit_file, glob, grep</td><td>File system operations</td></tr>
<tr><td><pre>Shell</pre></td><td>execute</td><td>Non-streaming command execution, mutually exclusive with StreamingShell</td></tr>
<tr><td><pre>StreamingShell</pre></td><td>execute (streaming)</td><td>Streaming command execution, mutually exclusive with Shell</td></tr>
</table>

Internally implemented using FileSystem Middleware.

---

## Task Planning: write_todos

<a href="/img/eino/HOJtbxNKWoibi2xzXrAcx0BUndb.png" target="_blank"><img src="/img/eino/HOJtbxNKWoibi2xzXrAcx0BUndb.png" width="100%" /></a>

The `write_todos` tool writes a structured TODO list to the session (key: `deep_agent_session_key_todos`) for subsequent reasoning reference.

**TODO Structure**:

```go
type TODO struct {
    Content    string `json:"content"`
    ActiveForm string `json:"activeForm"`
    Status     string `json:"status"` // "pending" | "in_progress" | "completed"
}
```

**Workflow**:

1. Model receives user input
2. Calls `write_todos` to decompose tasks and write to context
3. Executes TODO items one by one (calls task or direct tools)
4. Calls `write_todos` again to update progress

> 💡
> For simple tasks, calling write_todos every time may be counterproductive. The built-in Prompt already includes positive and negative examples to guide when to use it. You can further tune this through custom Instructions. Configure WithoutWriteTodos=true to disable it completely.

---

## Sub-Agent Delegation: task Tool

**TaskTool** is the unified invocation entry for all sub-Agents:

- Parameters: `subagent_type` (target sub-Agent name), `description` (task description)
- Internally wraps each sub-Agent as a tool via `adk.NewTypedAgentTool`
- Default Description includes all available sub-Agent names and descriptions; customizable via `TaskToolDescriptionGenerator`

**Context Isolation**:

- Sub-Agents only receive the task description assigned by the main Agent, not sharing conversation history
- The main Agent only receives the sub-Agent's final result; intermediate steps are not passed back
- Avoids large numbers of tool calls and intermediate reasoning "polluting" the main Agent's context

**general-purpose Sub-Agent**:

- Created by default, has the same tools (except task), Instruction, and ModelFailoverConfig as the main Agent
- Used to execute general tasks without a specialized sub-Agent in an isolated context
- Configure `WithoutGeneralSubAgent=true` to disable

---

## Comparison with Other Solutions

<table>
<tr><td>Dimension</td><td>DeepAgents vs ReAct</td><td>DeepAgents vs Plan-and-Execute</td></tr>
<tr><td>Advantages</td><td>Built-in planning + sub-Agent context isolation, better performance in multi-step tasks</td><td>Plan/RePlan invoked as tools on demand, reducing unnecessary planning overhead</td></tr>
<tr><td>Disadvantages</td><td>Planning + sub-Agent calls increase model requests, latency, and token costs</td><td>Planning and delegation completed in a single call, requiring higher model capability</td></tr>
</table>

---

## Usage Example

### Excel Agent Scenario

<a href="/img/eino/PhKjbQyKZoqaM9xyxptcceM9nsg.png" target="_blank"><img src="/img/eino/PhKjbQyKZoqaM9xyxptcceM9nsg.png" width="100%" /></a>

- Main Agent configured with ReadFile tool to assist task formulation
- Code (Python for Excel operations) and WebSearch sub-Agents added

### Code

Full example: [https://github.com/cloudwego/eino-examples/tree/main/adk/multiagent/deep](https://github.com/cloudwego/eino-examples/tree/main/adk/multiagent/deep)

```go
agent, err := deep.New(ctx, &deep.Config{
    Name:      "ExcelAgent",
    ChatModel: myModel,
    Backend:   localBackend,
    SubAgents: []adk.Agent{codeAgent, webSearchAgent},
    ToolsConfig: adk.ToolsConfig{
        InvokableTools: []tool.InvokableTool{readFileTool},
    },
})
```
