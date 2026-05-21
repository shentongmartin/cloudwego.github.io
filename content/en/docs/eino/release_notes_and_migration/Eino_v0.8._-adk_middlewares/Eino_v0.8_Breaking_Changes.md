---
Description: ""
date: "2026-03-10"
lastmod: ""
tags: []
title: Eino v0.8 Breaking Changes
weight: 1
---

## 1. API Breaking Changes

### 1.1 Filesystem Shell Interface Renamed

**Location**: `adk/filesystem/backend.go` **Description**: Shell-related interfaces have been renamed and no longer embed the `Backend` interface. **Before (v0.7.x)**:

```go
type ShellBackend interface {
    Backend
    Execute(ctx context.Context, input *ExecuteRequest) (result *ExecuteResponse, err error)
}

type StreamingShellBackend interface {
    Backend
    ExecuteStreaming(ctx context.Context, input *ExecuteRequest) (result *schema.StreamReader[*ExecuteResponse], err error)
}
```

**After (v0.8.0)**:

```go
type Shell interface {
    Execute(ctx context.Context, input *ExecuteRequest) (result *ExecuteResponse, err error)
}

type StreamingShell interface {
    ExecuteStreaming(ctx context.Context, input *ExecuteRequest) (result *schema.StreamReader[*ExecuteResponse], err error)
}
```

**Impact**:

- `ShellBackend` renamed to `Shell`
- `StreamingShellBackend` renamed to `StreamingShell`
- Interfaces no longer embed `Backend`; if your implementation relies on the combined interface, you need to implement them separately. **Migration Guide**:

```go
// Before
type MyBackend struct {}
func (b *MyBackend) Execute(...) {...}
// MyBackend implementing ShellBackend required implementing all Backend methods

// After
type MyShell struct {}
func (s *MyShell) Execute(...) {...}
// MyShell only needs to implement the Shell interface methods
// If Backend functionality is also needed, implement both interfaces separately
```

---

### 1.2 Filesystem Backend: Read Return Value Breaking Change

- **Location**: adk/filesystem/backend.go
- **Description**: The return value of `Backend.Read` has been incompatibly changed from returning `string` to returning a `*FileContent` struct.

**Before (v0.7.x)**:

```go
type Backend interface {
    ...
    Read(ctx context.Context, req *ReadRequest) (string, error)
    ...
 }
```

**After (v0.8.0)**:

```go
type Backend interface {
    ...
    Read(ctx context.Context, req *ReadRequest) (*FileContent, error)
    ...
 }
```

**Impact:**

- v0.7.x Read interface returned `string`. v0.8.0 Read interface returns struct `FileContent`, which is a breaking change.
- For Backend implementors: Need to replace the Read method implementation, changing from returning String to returning *FileContent.
- For Backend consumers: Need to upgrade the Backend implementation to one supporting v0.8. Also need to modify the Backend.Read call to use the new *FileContent return value.

## 2. Behavioral Breaking Changes

### 2.1 AgentEvent Sending Mechanism Change

**Location**: `adk/chatmodel.go` **Description**: The `AgentEvent` sending mechanism in `ChatModelAgent` has been changed from the eino callback mechanism to the Middleware mechanism. **Before (v0.7.x)**:

- `AgentEvent` was sent through eino's callback mechanism
- If users customized a ChatModel or Tool Decorator/Wrapper, and the original ChatModel/Tool had embedded Callback hooks, the `AgentEvent` would be sent **inside** the Decorator/Wrapper
- This applied to all ChatModels implemented in eino-ext, but might not apply to most user-implemented Tools and Tools provided by eino directly **After (v0.8.0)**:
- `AgentEvent` is sent through the Middleware mechanism
- `AgentEvent` is sent **outside** the user's custom Decorator/Wrapper **Impact**:
- Under normal circumstances, users won't notice this change
- If users previously implemented a ChatModel or Tool Decorator/Wrapper, the relative position of event sending changes
- Position change may cause the content of `AgentEvent` to change: previously events did not include changes made by Decorator/Wrapper, now events will include them **Rationale**:
- In normal business scenarios, we want the emitted events to include changes made by Decorator/Wrapper **Migration Guide**: If you previously wrapped ChatModel or Tool through Decorator/Wrapper, switch to implementing the `ChatModelAgentMiddleware` interface:

```go
// Before: Wrapping ChatModel through Decorator/Wrapper
type MyModelWrapper struct {
    inner model.BaseChatModel
}

func (w *MyModelWrapper) Generate(ctx context.Context, input []*schema.Message, opts ...model.Option) (*schema.Message, error) {
    // Custom logic
    return w.inner.Generate(ctx, input, opts...)
}

// After: Implement the WrapModel method of ChatModelAgentMiddleware
type MyMiddleware struct{}

func (m *MyMiddleware) WrapModel(ctx context.Context, chatModel model.BaseChatModel, mc *ModelContext) (model.BaseChatModel, error) {
    return &myWrappedModel{inner: chatModel}, nil
}

// For Tool Wrappers, switch to implementing WrapInvokableToolCall / WrapStreamableToolCall methods
```

### 2.2 filesystem.ReadRequest.Offset Semantic Change

**Location**: `adk/filesystem/backend.go` **Description**: The `Offset` field has been changed from 0-based to 1-based. **Before (v0.7.x)**:

```go
type ReadRequest struct {
    FilePath string
    // Offset is the 0-based line number to start reading from.
    Offset int
    Limit  int
}
```

**After (v0.8.0)**:

```go
type ReadRequest struct {

    FilePath string
    // Offset specifies the starting line number (1-based) for reading.
    // Line 1 is the first line of the file.
    // Values < 1 will be treated as 1.
    Offset int
    Limit  int
}
```

**Migration Guide**:

```go
// Before: Reading from line 0 (i.e., the first line)
req := &ReadRequest{Offset: 0, Limit: 100}

// After: Reading from line 1 (i.e., the first line)
req := &ReadRequest{Offset: 1, Limit: 100}

// If you previously used Offset: 10 to mean starting from line 11
// Now you need to use Offset: 11
```

---

### 2.3 filesystem.FileInfo.Path Semantic Change

**Location**: `adk/filesystem/backend.go` **Description**: The `FileInfo.Path` field no longer guarantees an absolute path. **Before (v0.7.x)**:

```go
type FileInfo struct {
    // Path is the absolute path of the file or directory.
    Path string
}
```

**After (v0.8.0)**:

```go
type FileInfo struct {
    // Path is the path of the file or directory, which can be a filename,
    // relative path, or absolute path.
    Path string
    // ...
}
```

**Impact**:

- Code that depends on `Path` being an absolute path may encounter issues
- Need to check and handle relative path cases

---

### 2.4 filesystem.WriteRequest Behavior Change

**Location**: `adk/filesystem/backend.go` **Description**: The write behavior of `WriteRequest` has been changed from "error if file exists" to "overwrite if file exists". **Before (v0.7.x)**:

```go
// WriteRequest comment:
// The file will be created if it does not exist, or error if file exists.
type WriteRequest struct {
    // FilePath is the absolute path of the file to write. Must start with '/'.
    // The file will be created if it does not exist, or error if file exists.
    FilePath string

    ...
}
```

**After (v0.8.0)**:

```go
// WriteRequest comment:
// Creates the file if it does not exist, overwrites if it exists.
type WriteRequest struct {
    // FilePath is the path of the file to write.
    FilePath string

    ....
}
```

**Impact**:

- Code that previously relied on "error if file exists" behavior will no longer error, but instead overwrite directly
- May lead to unexpected data loss **Migration Guide**:
- If you need to preserve the original behavior, check whether the file exists before writing
- Previously FilePath represented an absolute path; the new version does not stipulate that FilePath must be an absolute path. Scenarios that depended on absolute paths need to adapt accordingly

---

### 2.5 GrepRequest.Pattern Semantic Change

**Location**: `adk/filesystem/backend.go` **Description**: `GrepRequest.Pattern` has been changed from literal matching to regular expression matching. **Before (v0.7.x)**:

```go
// Pattern is the literal string to search for. This is not a regular expression.
// The search performs an exact substring match within the file's content.
```

**After (v0.8.0)**:

```go
// Pattern is the search pattern, supports full regular expression syntax.
// Uses ripgrep syntax (not grep).
```

**Impact**:

- Search patterns containing regex special characters will behave differently
- For example, searching for `interface{}` now requires escaping to `interface\{\}` **Migration Guide**:

```go
// Before: Literal search
req := &GrepRequest{Pattern: "interface{}"}

// After: Regex search, special characters need escaping
req := &GrepRequest{Pattern: "interface\\{\\}"}

// Or if searching for literals containing . * + ?, they also need escaping
// Before
req := &GrepRequest{Pattern: "config.json"}
// After
req := &GrepRequest{Pattern: "config\\.json"}
```

---

### 2.6 EditRequest.FilePath Semantic Change

**Location**: `adk/filesystem/backend.go` **Description**: The mandatory absolute path description has been removed from EditRequest.FilePath comments. **Before (v0.7.x)**:

```go
type EditRequest struct {
     // FilePath is the absolute path of the file to edit. Must start with '/'.
      FilePath string
    ....
    }
  }
```

**After (v0.8.0)**:

```go
type EditRequest struct {
   // FilePath is the path of the file to edit.
    FilePath string
}
```

**Impact**:

- In the old version, `FilePath` defaulted to representing an absolute path; the new version no longer guarantees `FilePath` is an absolute path. Logic that previously relied on `FilePath` being an absolute path needs to be adapted accordingly.

## Migration Recommendations

1. **Fix compilation errors first**: Type changes (such as Shell interface renaming) will cause compilation failures and need to be fixed first
2. **Pay attention to semantic changes**: `ReadRequest.Offset` changing from 0-based to 1-based, `Pattern` changing from literal to regex—these won't cause compilation errors but will change runtime behavior
3. **Check file operations**: The overwrite behavior change in `WriteRequest` may lead to data loss and requires additional checking
4. **Migrate Decorator/Wrapper**: If you have custom ChatModel/Tool Decorator/Wrappers, switch to implementing `ChatModelAgentMiddleware`
5. **Upgrade backend implementations as needed**: If using the local/ark agentkit backend provided by eino-ext, upgrade to the corresponding latest version: [adk/backend/local/v0.2.1](https://github.com/cloudwego/eino-ext/releases/tag/adk%2Fbackend%2Flocal%2Fv0.2.1) [adk/backend/agentkit/v0.2.1](https://github.com/cloudwego/eino-ext/releases/tag/adk%2Fbackend%2Fagentkit%2Fv0.2.1)
6. **Test verification**: After migration, conduct comprehensive testing, especially code involving file operations and search functionality
