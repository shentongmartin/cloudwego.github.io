---
Description: ""
date: "2026-05-17"
lastmod: ""
tags: []
title: PlanTask
weight: 6
---

> 💡
> This middleware was introduced in v0.8.0. Package path: `github.com/cloudwego/eino/adk/middlewares/plantask`

## Overview

`plantask` is a task management middleware that injects four tools into the Agent through the `BeforeAgent` hook, giving it structured task planning capabilities:

<table>
<tr><td>Tool</td><td>Function</td></tr>
<tr><td><pre>TaskCreate</pre></td><td>Create a task</td></tr>
<tr><td><pre>TaskGet</pre></td><td>Get details of a single task</td></tr>
<tr><td><pre>TaskUpdate</pre></td><td>Update task status/fields, set dependencies, delete task</td></tr>
<tr><td><pre>TaskList</pre></td><td>List summaries of all tasks</td></tr>
</table>

Core purpose: break complex requests into trackable sub-tasks, manage dependencies between tasks, and let users see execution progress.

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                              Agent                                      │
│                                                                         │
│  ┌───────────────────────────────────────────────────────────────────┐  │
│  │  BeforeAgent: Inject task tools (with sync.Mutex for concurrency) │  │
│  │    - TaskCreate                                                    │  │
│  │    - TaskGet                                                       │  │
│  │    - TaskUpdate                                                    │  │
│  │    - TaskList                                                      │  │
│  └───────────────────────────────────────────────────────────────────┘  │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                             Backend                                     │
│                                                                         │
│  Storage structure:                                                     │
│    baseDir/                                                             │
│    ├── .highwatermark    # Maximum assigned ID (plain numeric text)     │
│    ├── 1.json            # Task #1                                      │
│    ├── 2.json            # Task #2                                      │
│    └── ...                                                              │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## API

### Constructors

```go
// Generic version, supports *schema.Message and *schema.AgenticMessage
func NewTyped[M adk.MessageType](ctx context.Context, config *Config) (adk.TypedChatModelAgentMiddleware[M], error)

// Non-generic version, equivalent to NewTyped[*schema.Message]
func New(ctx context.Context, config *Config) (adk.ChatModelAgentMiddleware, error)
```

### Config

```go
type Config struct {
    Backend Backend  // Storage backend, required
    BaseDir string   // Task file storage directory, required
}
```

> 💡
> The Backend should be isolated at the session level — different sessions correspond to different Backend instances (i.e., different task lists).

### Backend Interface

`Backend` is defined within the `plantask` package and is a minimal subset of `filesystem.Backend`, retaining only the four methods needed for task storage:

```go
type Backend interface {
    LsInfo(ctx context.Context, req *LsInfoRequest) ([]FileInfo, error)
    Read(ctx context.Context, req *ReadRequest) (*filesystem.FileContent, error)
    Write(ctx context.Context, req *WriteRequest) error
    Delete(ctx context.Context, req *DeleteRequest) error
}
```

Type alias relationships:

```go
type FileInfo = filesystem.FileInfo        // Path, IsDir, Size, ModifiedAt
type LsInfoRequest = filesystem.LsInfoRequest  // Path string
type ReadRequest = filesystem.ReadRequest       // FilePath, Offset, Limit
type WriteRequest = filesystem.WriteRequest     // FilePath, Content string

// DeleteRequest is custom to the plantask package (filesystem package has no such type)
type DeleteRequest struct {
    FilePath string
}
```

> 💡
> Note that `Read` returns `*filesystem.FileContent` (containing a `Content string` field), not a bare string. Import path is `github.com/cloudwego/eino/adk/filesystem`.

---

## Task Structure

```go
type task struct {
    ID          string         `json:"id"`
    Subject     string         `json:"subject"`
    Description string         `json:"description"`
    Status      string         `json:"status"`
    Blocks      []string       `json:"blocks"`
    BlockedBy   []string       `json:"blockedBy"`
    ActiveForm  string         `json:"activeForm,omitempty"`
    Owner       string         `json:"owner,omitempty"`
    Metadata    map[string]any `json:"metadata,omitempty"`
}
```

### Status

<table>
<tr><td>Status Value</td><td>Description</td></tr>
<tr><td><pre>pending</pre></td><td>Pending (default on creation)</td></tr>
<tr><td><pre>in_progress</pre></td><td>In progress</td></tr>
<tr><td><pre>completed</pre></td><td>Completed</td></tr>
<tr><td><pre>deleted</pre></td><td>Deleted (physically deletes the JSON file and removes from other tasks' dependency lists)</td></tr>
</table>

Status transitions: `pending` → `in_progress` → `completed`; any status can be directly set to `deleted`.

---

## Tool Parameters

### TaskCreate

Tool name constant: `TaskCreateToolName = "TaskCreate"`

<table>
<tr><td>Parameter</td><td>Type</td><td>Required</td><td>Description</td></tr>
<tr><td><pre>subject</pre></td><td>string</td><td>Yes</td><td>Task title (imperative form)</td></tr>
<tr><td><pre>description</pre></td><td>string</td><td>Yes</td><td>Detailed task description, including context and acceptance criteria</td></tr>
<tr><td><pre>activeForm</pre></td><td>string</td><td>No</td><td>Active form text (e.g., "Running tests"), displayed to user when in_progress</td></tr>
<tr><td><pre>metadata</pre></td><td>object</td><td>No</td><td>Custom key-value pairs</td></tr>
</table>

After creation, the task ID auto-increments (based on the `.highwatermark` file), with initial status `pending`.

### TaskGet

Tool name constant: `TaskGetToolName = "TaskGet"`

<table>
<tr><td>Parameter</td><td>Type</td><td>Required</td><td>Description</td></tr>
<tr><td><pre>taskId</pre></td><td>string</td><td>Yes</td><td>Task ID (numeric string)</td></tr>
</table>

Returns complete task information: subject, description, status, blocks, blockedBy, owner.

### TaskUpdate

Tool name constant: `TaskUpdateToolName = "TaskUpdate"`

<table>
<tr><td>Parameter</td><td>Type</td><td>Required</td><td>Description</td></tr>
<tr><td><pre>taskId</pre></td><td>string</td><td>Yes</td><td>Task ID</td></tr>
<tr><td><pre>subject</pre></td><td>string</td><td>No</td><td>New title</td></tr>
<tr><td><pre>description</pre></td><td>string</td><td>No</td><td>New description</td></tr>
<tr><td><pre>activeForm</pre></td><td>string</td><td>No</td><td>New active form text</td></tr>
<tr><td><pre>status</pre></td><td>string</td><td>No</td><td>New status, enum: <pre>pending</pre> / <pre>in_progress</pre> / <pre>completed</pre> / <pre>deleted</pre></td></tr>
<tr><td><pre>addBlocks</pre></td><td>[]string</td><td>No</td><td>Add task IDs that are blocked by the current task (written bidirectionally)</td></tr>
<tr><td><pre>addBlockedBy</pre></td><td>[]string</td><td>No</td><td>Add task IDs that block the current task (written bidirectionally)</td></tr>
<tr><td><pre>owner</pre></td><td>string</td><td>No</td><td>Responsible agent name</td></tr>
<tr><td><pre>metadata</pre></td><td>object</td><td>No</td><td>Merged into existing metadata; setting a key to null deletes that key</td></tr>
</table>

Key behaviors:

- `status: "deleted"` physically deletes the task file and removes the ID from all other tasks' blocks/blockedBy
- Adding dependencies performs **cycle detection**; an error is returned if a cycle would be created
- When **all tasks are completed**, all task files are automatically deleted (cleanup mechanism)

### TaskList

Tool name constant: `TaskListToolName = "TaskList"`

No parameters. Returns a summary list of all tasks (sorted by ID), each formatted as:

```
#ID [status] subject [owner: xxx] [blocked by #x, #y]
```

---

## Usage Example

```go
ctx := context.Background()

// Backend should be isolated at the session level
middleware, err := plantask.New(ctx, &plantask.Config{
    Backend: myBackend,
    BaseDir: "/tasks",
})
if err != nil {
    return err
}

agent, err := adk.NewChatModelAgent(ctx, &adk.ChatModelAgentConfig{
    Model:    myModel,
    Handlers: []adk.ChatModelAgentMiddleware{middleware},
})
```

### Typical Flow

```
1. Receive complex task
       │
       ▼
2. TaskCreate to create multiple sub-tasks
   - #1: Analyze requirements
   - #2: Implement code
   - #3: Write tests
       │
       ▼
3. TaskUpdate to set dependencies
   - #2 addBlockedBy: ["1"]
   - #3 addBlockedBy: ["2"]
       │
       ▼
4. TaskList to view available tasks
       │
       ▼
5. TaskUpdate #1 → in_progress
       │
       ▼
6. After completion, TaskUpdate #1 → completed
       │
       ▼
7. Loop 4-6 until all completed
       │
       ▼
8. All completed → automatic cleanup of all files
```

---

## Dependency Management

- **blocks**: "After I complete, these tasks can start"
- **blockedBy**: "After these tasks complete, I can start"

Dependency writing is **bidirectional**: executing `addBlocks: ["2"]` on Task A will also write A's ID into Task #2's `blockedBy`.

```
Task #1 (blocks: ["2"])  ────►  Task #2 (blockedBy: ["1"])

#2 can only start after #1 completes
```

Cycle detection is implemented via DFS reachability:

```
#1 blocks #2
#2 blocks #1  ← Error: would create a cyclic dependency
```

---

## Implementation Details

<table>
<tr><td>Mechanism</td><td>Description</td></tr>
<tr><td>ID allocation</td><td><pre>.highwatermark</pre> file stores the current maximum ID, incremented by 1 on creation</td></tr>
<tr><td>Concurrency safety</td><td>All four tools share a single <pre>sync.Mutex</pre>, serializing execution within the same middleware instance</td></tr>
<tr><td>File format</td><td>One <pre>{id}.json</pre> file per task, JSON serialized using <pre>sonic</pre></td></tr>
<tr><td>Automatic cleanup</td><td>After TaskUpdate marks a task as completed, it checks — if all tasks are completed, batch delete all files</td></tr>
<tr><td>ID validation</td><td>Pure numeric regex <pre>^\d+$</pre></td></tr>
<tr><td>Delete cascading</td><td>When deleting a task, iterates all task files to remove references to that ID</td></tr>
</table>

---

## Multi-language Support

Tool descriptions support Chinese and English, switchable via global setting:

```go
// Use Chinese descriptions
adk.SetLanguage(adk.LanguageChinese)

// Use English descriptions (default)
adk.SetLanguage(adk.LanguageEnglish)
```

This setting affects all ADK built-in prompts and tool descriptions.
