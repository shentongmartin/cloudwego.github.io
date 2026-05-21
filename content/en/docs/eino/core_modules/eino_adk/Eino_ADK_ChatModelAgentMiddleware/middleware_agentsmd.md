---
Description: ""
date: "2026-05-17"
lastmod: ""
tags: []
title: AgentsMD
weight: 9
---

## Overview

`agentsmd` is an Eino ADK middleware that **automatically injects the content of Agents.md files into the message sequence on every model call**. The injected message is persisted by the framework into the agent's internal state, but **idempotency checks** (`Extra["__agentsmd_content__"]` marker) ensure it is never injected more than once. Since the injected content is fixed at its first appearance, **it will not change with subsequent summarization/compression**.

**Core value**: Define system-level behavior instructions and context for an Agent via Agents.md files (similar to Claude Code's CLAUDE.md), without manually managing system prompt composition.

**Package path**: `github.com/cloudwego/eino/adk/middlewares/agentsmd`

## Quick Start

```go
ctx := context.Background()

// 1. Create agentsmd middleware
mw, err := agentsmd.New(ctx, &agentsmd.Config{
    Backend:       myBackend, // Implements agentsmd.Backend interface
    AgentsMDFiles: []string{"/project/agents.md"},
})
if err != nil {
    panic(err)
}

// 2. Configure with Agent
agent, _ := adk.NewChatModelAgent(ctx, &adk.ChatModelAgentConfig{
    Model:    chatModel,
    Handlers: []adk.ChatModelAgentMiddleware{mw},
})
```

---

## Configuration Details

### Config Struct

```go
type Config struct {
    Backend             Backend
    AgentsMDFiles       []string
    AllAgentsMDMaxBytes int
    OnLoadWarning       func(filePath string, err error)
}
```

### Parameters

<table>
<tr><td>Parameter</td><td>Type</td><td>Required</td><td>Default</td><td>Description</td></tr>
<tr><td><pre>Backend</pre></td><td><pre>Backend</pre></td><td>Yes</td><td>—</td><td>File reading backend, responsible for actual file I/O</td></tr>
<tr><td><pre>AgentsMDFiles</pre></td><td><pre>[]string</pre></td><td>Yes</td><td>—</td><td>List of Agents.md file paths to load (at least one), loaded and injected in order</td></tr>
<tr><td><pre>AllAgentsMDMaxBytes</pre></td><td><pre>int</pre></td><td>No</td><td><pre>0</pre> (unlimited)</td><td>Total byte limit for all files; subsequent files are skipped once exceeded, but each file is always loaded in full</td></tr>
<tr><td><pre>OnLoadWarning</pre></td><td><pre>func(string, error)</pre></td><td>No</td><td><pre>log.Printf</pre></td><td>Callback for non-fatal errors (file missing, cyclic @import, depth limit exceeded, etc.)</td></tr>
</table>

### Validation Rules

`New` / `NewTyped` validates Config on creation:

- `Config` must not be nil
- `Backend` must not be nil
- `AgentsMDFiles` must contain at least one path
- `AllAgentsMDMaxBytes` must not be negative

---

## Constructors

### New — Standard Constructor

```go
func New(ctx context.Context, cfg *Config) (adk.ChatModelAgentMiddleware, error)
```

Returns `ChatModelAgentMiddleware` (i.e., `TypedChatModelAgentMiddleware[*schema.Message]`), suitable for standard `ChatModelAgent`.

### NewTyped — Generic Constructor

```go
func NewTyped[M adk.MessageType](_ context.Context, cfg *Config) (adk.TypedChatModelAgentMiddleware[M], error)
```

Generic version, supporting both `*schema.Message` and `*schema.AgenticMessage` message types. `New` internally calls `NewTyped[*schema.Message]`.

## Backend Interface

### Interface Definition

```go
type Backend interface {
    Read(ctx context.Context, req *ReadRequest) (*FileContent, error)
}
```

### Type Definitions

`ReadRequest` and `FileContent` are aliases for the same-named types in the `github.com/cloudwego/eino/adk/filesystem` package:

```go
type ReadRequest = filesystem.ReadRequest
type FileContent = filesystem.FileContent
```

> 💡
> **Backend Implementation Requirements**
>
> - When a file does not exist, implementations **must** return an error wrapping `os.ErrNotExist` (so that `errors.Is(err, os.ErrNotExist)` returns `true`); the loader uses this to distinguish "file missing" from "real I/O error"
> - Other errors (permission denied, I/O errors) will **abort the entire loading process** and are not treated as warnings
> - The `Read` method should be concurrency-safe

---

## @import Syntax

Agents.md files support the `@path` syntax for recursive inclusion of other files.

### Syntax Format

```markdown
# Project Instructions

You are a coding assistant.

Please follow these rules:
@rules/code-style.md
@rules/api-conventions.md
```

### Matching Rules

The loader uses the regex `@([a-zA-Z0-9_.~/][a-zA-Z0-9_.~/\-]*)` to scan file content, with the following filtering logic:

- **Paths containing /**: directly treated as @import (e.g., `@rules/style.md`)
- **Paths without /**: treated as @import only when the extension is in the allow list; otherwise ignored

**Allowed extensions**: `.md`, `.txt`, `.mdx`, `.yaml`, `.yml`, `.json`, `.toml`

This design avoids misinterpreting `@someone`, `@example.com`, etc. as import targets.

### Resolution Behavior

<table>
<tr><td>Rule</td><td>Description</td></tr>
<tr><td>Path resolution</td><td>Relative paths are resolved from the current file's directory; absolute paths are used as-is</td></tr>
<tr><td>Maximum recursion depth</td><td><strong>5 levels</strong> (exceeded paths are skipped and trigger <pre>OnLoadWarning</pre>)</td></tr>
<tr><td>Cycle detection</td><td>Paths already present in the current ancestor chain are skipped (triggers <pre>OnLoadWarning</pre>)</td></tr>
<tr><td>Global deduplication</td><td>The same file path is read and injected only once across the entire load</td></tr>
<tr><td>Original text preserved</td><td>@imported files are appended as separate paragraphs; the <pre>@path</pre> text in the original is <strong>not removed</strong></td></tr>
<tr><td>Byte budget</td><td>Once cumulative bytes exceed <pre>AllAgentsMDMaxBytes</pre>, subsequent imports are skipped</td></tr>
</table>

### Directory Structure Example

```
project/
├── Agents.md               # Main entry file
├── rules/
│   ├── code-style.md       # @rules/code-style.md
│   ├── api-conventions.md  # @rules/api-conventions.md
│   └── testing.md
└── context/
    └── architecture.md
```

---

## How It Works

### Implementation Hook

The middleware implements the `BeforeModelRewriteState` method of the `TypedChatModelAgentMiddleware` interface (**not** WrapModel). This hook triggers before each model call, when the state is being rewritten.

### Injection Flow

### Message Sequence After Injection

```
[System]     System prompt
[User]       ← Agents.md content (with Extra marker)
[User]       User historical message 1
[Assistant]  Assistant reply 1
[User]       Current user message
```

### Key Mechanisms

**1. Persistent injection + idempotency guarantee**

The framework persists the state returned by `BeforeModelRewriteState` into the agent's internal state (`st.Messages = state.Messages`). The injected message is marked with `Extra["__agentsmd_content__"]`; each time the hook is entered, it first scans for this marker — if found, it returns the original state directly, avoiding duplicate injection. Therefore, in effect: the content is injected and persisted on the first model call, and subsequent iterations do not re-insert it.

**2. Run-level caching**

Within the same `Run()`, content loaded for the first time is cached in RunLocal storage via `adk.SetRunLocalValue`. Subsequent model calls (e.g., during multi-turn tool calls) directly reuse the cache via `adk.GetRunLocalValue`. Each new `Run()` reloads from scratch, so file modifications take effect on the next Run.

**4. Insertion position**

Content is inserted as a `User` role message **before the first User message**. If there are no User messages in the sequence, it is appended to the end.

**5. Content formatting**

Loaded file content is formatted:

- Wrapped in `<system-reminder>` tags
- Includes i18n header (prompting the model to follow instructions) and footer (noting the context may not be relevant)
- Each file is displayed independently with a `File content: {path} (instructions):` prefix
- Language (Chinese/English) is controlled globally via `adk.SetLanguage`

---

## Notes

### Middleware Ordering

> 💡
> **It is recommended to place the agentsmd middleware after summarization/compression middlewares.** This ensures Agents.md content is not compressed by summarization, and the model receives full instructions on every call.

```go
Handlers: []adk.ChatModelAgentMiddleware{
    summarizationMiddleware, // Summarize first
    agentsMDMiddleware,      // Then inject Agents.md
}
```

### Error Handling

<table>
<tr><td>Scenario</td><td>Behavior</td></tr>
<tr><td>File not found (<pre>os.ErrNotExist</pre>)</td><td>Skip the file, trigger <pre>OnLoadWarning</pre></td></tr>
<tr><td>Cyclic @import</td><td>Skip the cyclic file, trigger <pre>OnLoadWarning</pre></td></tr>
<tr><td>@import depth exceeds 5 levels</td><td>Skip, trigger <pre>OnLoadWarning</pre></td></tr>
<tr><td>Cumulative size exceeds <pre>AllAgentsMDMaxBytes</pre></td><td>Skip subsequent files, trigger <pre>OnLoadWarning</pre> (the first file is always loaded in full)</td></tr>
<tr><td>Permission denied / I/O error</td><td><strong>Abort loading, return error</strong></td></tr>
<tr><td>All file contents empty</td><td>Do not inject; pass through original messages</td></tr>
</table>

### Performance Considerations

- Set `AllAgentsMDMaxBytes` reasonably to avoid injecting too much content that occupies the context window
- Agents.md content is loaded only once per `Run()` (run-level caching), but **every new `Run()` reloads**, so file edits take effect on the next Run
- Avoid importing too many files; the recursion depth limit is 5 levels

### Agents.md Writing Guidelines

- Keep content concise; only include instructions that truly affect model behavior
- Use @import to split by concerns (code standards, API conventions, architecture notes, etc.)
- Avoid including large code examples or data to prevent wasting the context window
- File content is wrapped in `<system-reminder>` tags when passed to the model

---

## FAQ

**Q: Will Agents.md content be saved into the conversation history?**

A: Yes. The state returned by `BeforeModelRewriteState` is persisted by the framework. However, due to the idempotency check (`Extra["__agentsmd_content__"]` marker), content is only injected once on the first model call; subsequent iterations skip it directly. It is recommended to place agentsmd after summarization to avoid the injected content being compressed by summarization.

**Q: What happens if an Agents.md file does not exist?**

A: That file is skipped, triggering the `OnLoadWarning` callback (defaults to `log.Printf`), without affecting other files' loading.

**Q: What is the base directory for @import paths?**

A: The directory of the current file. For example, `@rules/style.md` in `/project/Agents.md` resolves to `/project/rules/style.md`.

**Q: If multiple files @import the same file, will it be loaded multiple times?**

A: No. The loader maintains a global deduplication map (`seen`); the same path is read and injected only once.

**Q: Will the @path reference in the original text be replaced?**

A: No. @imported files are appended as separate paragraphs after the original text; the original content remains unchanged.

**Q: What is the difference between New and NewTyped?**

A: `New` returns `ChatModelAgentMiddleware` (i.e., `TypedChatModelAgentMiddleware[*schema.Message]`), suitable for standard Agents. `NewTyped` is the generic version that additionally supports the `*schema.AgenticMessage` type, for Agentic Model scenarios.
