---
Description: ""
date: "2026-05-17"
lastmod: ""
tags: []
title: Eino Dev Visual Debugging Plugin Guide
weight: 3
---

## Overview

> 💡
> Use this plugin to visually debug orchestration artifacts built with Eino (Graph, Chain):
>
> 1. Visual rendering of orchestration
> 2. Start from any operable node and debug with mock input

## Quick Start

### Download eino-examples

Repo: [https://github.com/cloudwego/eino-examples](https://github.com/cloudwego/eino-examples)

```bash
git clone https://github.com/cloudwego/eino-examples.git
# or
git clone git@github.com:cloudwego/eino-examples.git
```

### Install Dependencies

In the project directory, run the following in order:

```bash
go get github.com/cloudwego/eino-ext/devops@latest
go mod tidy
```

### Run the Demo

Open `eino-examples/devops/debug/main.go` and run `main.go`. The plugin launches a local HTTP service to connect to your process; allow network access if prompted.

<a href="/img/eino/eino_debug_enter_config_page.png" target="_blank"><img src="/img/eino/eino_debug_enter_config_page.png" width="100%" /></a>

### Configure Debug Address

<table><tbody><tr>
<td>
1) Click the debug feature entry on the left or center to open configuration
<a href="/img/eino/eino_debug_enter_page.png" target="_blank"><img src="/img/eino/eino_debug_enter_page.png" width="100%" /></a>
</td><td>
2) Click “Configure Address”
<a href="/img/eino/eino_debug_config_3_page.png" target="_blank"><img src="/img/eino/eino_debug_config_3_page.png" width="100%" /></a>
</td></tr></tbody></table>

<table><tbody><tr>
<td>
3) Enter 127.0.0.1:52538
<a href="/img/eino/eino_debug_config_2_page.png" target="_blank"><img src="/img/eino/eino_debug_config_2_page.png" width="100%" /></a>
</td><td>
4) Confirm to enter the debug view, then select the Graph to debug
<a href="/img/eino/eino_orchestration_index_2_page.png" target="_blank"><img src="/img/eino/eino_orchestration_index_2_page.png" width="100%" /></a>
</td></tr></tbody></table>

### Start Debugging

<table><tbody><tr>
<td>
1) Click “Test Run” to start from START
<a href="/img/eino/eino_debug_enter_test_run_2_page.png" target="_blank"><img src="/img/eino/eino_debug_enter_test_run_2_page.png" width="100%" /></a>
</td><td>
2) Enter "hello eino" and confirm
<a href="/img/eino/eino_debug_run_input_mock_data_page.png" target="_blank"><img src="/img/eino/eino_debug_run_input_mock_data_page.png" width="100%" /></a>

</td></tr></tbody></table>

<table><tbody><tr>
<td>3) Inspect per-node inputs/outputs</td><td>4) Switch Input/Output views</td>
</tr>
<tr>
<td><a href="/img/eino/eino_debug_test_run_detail_page.png" target="_blank"><img src="/img/eino/eino_debug_test_run_detail_page.png" width="100%" /></a></td>
<td><a href="/img/eino/eino_debug_index_page.png" target="_blank"><img src="/img/eino/eino_debug_index_page.png" width="100%" /></a></td>
</tr></tbody></table>

## Feature Overview

### Local or Remote Debugging

Configure `IP:Port` to connect to the target process, whether local or remote.

<a href="/img/eino/eino_debug_run_config_page.png" target="_blank"><img src="/img/eino/eino_debug_run_config_page.png" width="100%" /></a>

### Orchestration Visualization

Supports Graph and Chain topology visualization.

<a href="/img/eino/eino_debug_list_nodes_page.png" target="_blank"><img src="/img/eino/eino_debug_list_nodes_page.png" width="100%" /></a>

### Start from Any Node

<a href="/img/eino/eino_debug_test_run_of_one_node_page.png" target="_blank"><img src="/img/eino/eino_debug_test_run_of_one_node_page.png" width="100%" /></a>

### Inspect Node Results

Each node’s input, output, and execution time are shown in order.

<a href="/img/eino/eino_debug_run_detail_v2_page.png" target="_blank"><img src="/img/eino/eino_debug_run_detail_v2_page.png" width="100%" /></a>

## Debugging from Scratch

### Orchestrate with Eino

The plugin supports debugging Graph and Chain artifacts. Example registration:

```go
func RegisterSimpleGraph(ctx context.Context) {
    g := compose.NewGraph[string, string]()
    _ = g.AddLambdaNode("node_1", compose.InvokableLambda(func(ctx context.Context, input string) (output string, err error) {
       return input + " process by node_1,", nil
    }))
    _ = g.AddLambdaNode("node_2", compose.InvokableLambda(func(ctx context.Context, input string) (output string, err error) {
       return input + " process by node_2,", nil
    }))
    _ = g.AddLambdaNode("node_3", compose.InvokableLambda(func(ctx context.Context, input string) (output string, err error) {
       return input + " process by node_3,", nil
    }))

    _ = g.AddEdge(compose.START, "node_1")
    _ = g.AddEdge("node_1", "node_2")
    _ = g.AddEdge("node_2", "node_3")
    _ = g.AddEdge("node_3", compose.END)

    _, err := g.Compile(ctx)
    if err != nil {
       logs.Errorf("compile graph failed, err=%v", err)
       return
    }
}
```

### Install Dependencies

```bash
go get github.com/cloudwego/eino-ext/devops@latest
go mod tidy
```

### Initialize Debugging

Because debugging starts an HTTP service in your main process to interact with the local plugin, you must call `Init()` from `github.com/cloudwego/eino-ext/devops` to start the debug service.

> 💡
> Notes
>
> 1. Ensure the target orchestration has run `Compile()` at least once.
> 2. `devops.Init()` must run before calling `Compile()`.
> 3. Make sure the main process stays alive after `devops.Init()`.
> 4. Starting from v0.1.9, the debug service default listen address changed from `0.0.0.0` to `127.0.0.1` (local connections only). For remote debugging, explicitly specify the listen IP via `WithDevServerIP`, e.g.: `devops.Init(ctx, devops.WithDevServerIP("0.0.0.0"))`.

```go
// 1. Initialize debug service
err := devops.Init(ctx)
if err != nil {
    logs.Errorf("[eino dev] init failed, err=%v", err)
    return
}

// 2. Compile the target orchestration artifact to debug
RegisterSimpleGraph(ctx)
```

### Run Your Process

Run your process locally or remotely, and ensure the main process does not exit.

In `github.com/cloudwego/eino-examples/devops/debug/main.go`, `main()` looks like:

```go
func main() {
    ctx := context.Background()
    // Init eino devops server
    err := devops.Init(ctx)
    if err != nil {
       logs.Errorf("[eino dev] init failed, err=%v", err)
       return
    }

    // Register chain, graph and state_graph for demo use
    chain.RegisterSimpleChain(ctx)
    graph.RegisterSimpleGraph(ctx)
    graph.RegisterSimpleStateGraph(ctx)

    // Blocking process exits
    sigs := make(chan os.Signal, 1)
    signal.Notify(sigs, syscall.SIGINT, syscall.SIGTERM)
    <-sigs

    // Exit
    logs.Infof("[eino dev] shutting down\n")
}
```

### Configure Address

- **IP**: IP address of the server where the user process is running.
  - If the user process is running on local computer, enter `127.0.0.1`;
  - If the user process is running on a remote server, enter the remote server's IP address, supporting both IPv4 and IPv6.
- **Port**: Port the debug service listens on, default is `52538`, configurable via the `WithDevServerPort` option method.

> 💡
> Notes
>
> - Local debugging: The system may pop up a network access warning; allow access.
> - Remote debugging: Ensure the port is accessible. Additionally, starting from v0.1.9, the default listen address is `127.0.0.1` only; for remote debugging you must specify an accessible IP (e.g., `0.0.0.0`) via `WithDevServerIP` when calling `devops.Init()`.

Once IP and Port are configured, click confirm. The debug plugin will automatically connect to the target debug server. If successfully connected, the connection status indicator will turn green.

<a href="/img/eino/eino_debug_ip_port_show_page.png" target="_blank"><img src="/img/eino/eino_debug_ip_port_show_page.png" width="100%" /></a>

### Select an Artifact

Ensure your target orchestration has been compiled at least once. Multiple `Compile()` runs register multiple artifacts; you’ll see them in the selection list.

<a href="/img/eino/eino_debug_panel_3.png" target="_blank"><img src="/img/eino/eino_debug_panel_3.png" width="100%" /></a>

### Start Debugging

- From START: click “Test Run”, enter mock input (complex types are inferred), and confirm.

<a href="/img/eino/eino_debug_run_code_2.png" target="_blank"><img src="/img/eino/eino_debug_run_code_2.png" width="100%" /></a>

<a href="/img/eino/eino_debug_run_input_mock_data_2_page.png" target="_blank"><img src="/img/eino/eino_debug_run_input_mock_data_2_page.png" width="100%" /></a>

- From a specific node: click the run button on that node to start debugging from there.

<a href="/img/eino/eino_debug_button_run_code.png" target="_blank"><img src="/img/eino/eino_debug_button_run_code.png" width="100%" /></a>

<a href="/img/eino/eino_debug_run_of_mock_input_of_page.png" target="_blank"><img src="/img/eino/eino_debug_run_of_mock_input_of_page.png" width="100%" /></a>

### View Execution Results

Debugging from the START node: after clicking Test Run, view debug results in the plugin panel below.

<a href="/img/eino/eino_debug_test_run_result_page.png" target="_blank"><img src="/img/eino/eino_debug_test_run_result_page.png" width="100%" /></a>

Debugging from any operable node: view debug results in the plugin panel below.

<a href="/img/eino/eino_debug_results.png" target="_blank"><img src="/img/eino/eino_debug_results.png" width="100%" /></a>

## Advanced

### Specify Implementation Type for Interface Fields

Interface-typed fields render as `{}` by default. Type a space inside `{}` to select an implementation type. The plugin uses a special JSON structure:

```json
{
  "_value": {},           // JSON value of the concrete type
  "_eino_go_type": "*model.MyConcreteType" // Go type name
}
```

> 💡
> Common interface types like `string`, `schema.Message` are built-in. To register custom types, use `devops.AppendType` during `Init()`.

1) Suppose you have orchestration code where the graph input is `any`, and `node_1` takes `*NodeInfo`:

```go
type NodeInfo struct {
    Message string
}

func RegisterGraphOfInterfaceType(ctx context.Context) {
    // Define a graph that input parameter is any.
    g := compose.NewGraph[any, string]()

    _ = g.AddLambdaNode("node_1", compose.InvokableLambda(func(ctx context.Context, input *NodeInfo) (output string, err error) {
       if input == nil {
          return "", nil
       }
       return input.Message + " process by node_1,", nil
    }))

    _ = g.AddLambdaNode("node_2", compose.InvokableLambda(func(ctx context.Context, input string) (output string, err error) {
       return input + " process by node_2,", nil
    }))

    _ = g.AddLambdaNode("node_3", compose.InvokableLambda(func(ctx context.Context, input string) (output string, err error) {
       return input + " process by node_3,", nil
    }))

    _ = g.AddEdge(compose._START_, "node_1")

    _ = g.AddEdge("node_1", "node_2")

    _ = g.AddEdge("node_2", "node_3")

    _ = g.AddEdge("node_3", compose._END_)

    r, err := g.Compile(ctx)
    if err != nil {
       logs.Errorf("compile graph failed, err=%v", err)
       return
    }
}
```

2) Before debugging, register the custom `*NodeInfo` type with `AppendType` at `Init()`:

```go
err := devops.Init(ctx, devops.AppendType(&graph.NodeInfo{}))
```

3) During Test Run, interface fields show `{}` by default. Type a space inside `{}` to view all built-in and custom types, and select the concrete implementation type for that interface.

<a href="/img/eino/eino_debug_run_code.png" target="_blank"><img src="/img/eino/eino_debug_run_code.png" width="100%" /></a>

4) Fill in the debug node input in the `_value` field.

<a href="/img/eino/eino_debug_run_code_3.png" target="_blank"><img src="/img/eino/eino_debug_run_code_3.png" width="100%" /></a>

5) Click confirm to view the debug results.

<a href="/img/eino/eino_debug_panel_2.png" target="_blank"><img src="/img/eino/eino_debug_panel_2.png" width="100%" /></a>

#### Debugging `map[string]any`

Here we explain how to debug when the input type is `map[string]any`. If a node's input type is `map[string]any`, as shown below:

```go
func RegisterAnyInputGraph(ctx context.Context) {
        g := compose.NewGraph[map[string]any, string]()

        _ = g.AddLambdaNode("node_1", compose.InvokableLambda(func(ctx context.Context, input map[string]any) (output string, err error) {
                for k, v := range input {
                        switch v.(type) {
                        case string:
                                output += k + ":" + v.(string) + ","
                        case int:
                                output += k + ":" + fmt.Sprintf("%d", v.(int))
                        default:
                                return "", fmt.Errorf("unsupported type: %T", v)
                        }
                }

                return output, nil
        }))

        _ = g.AddLambdaNode("node_2", compose.InvokableLambda(func(ctx context.Context, input string) (output string, err error) {
                return input + " process by node_2,", nil
        }))

        _ = g.AddEdge(compose.START, "node_1")

        _ = g.AddEdge("node_1", "node_2")

        _ = g.AddEdge("node_2", compose.END)

        r, err := g.Compile(ctx)
        if err != nil {
                logs.Errorf("compile graph failed, err=%v", err)
                return
        }

        message, err := r.Invoke(ctx, map[string]any{"name": "bob", "score": 100})
        if err != nil {
                logs.Errorf("invoke graph failed, err=%v", err)
                return
        }

        logs.Infof("eino any input graph output is: %v", message)
}
```

During debugging, in the Test Run JSON input box, you need to enter content in the following format:

```json
{
  "name": {
    "_value": "alice",
    "_eino_go_type": "string"
  },
  "score": {
    "_value": "99",
    "_eino_go_type": "int"
  }
}
```
