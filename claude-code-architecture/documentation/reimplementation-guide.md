---
title: "Claude Code CLI — Re-implementation Guide"
scope: "Step-by-step playbook for building this system from scratch"
last-analyzed: "2026-04-02"
---

# Re-implementation Guide

## 1. Recommended Implementation Order

Build the system in this order, testing each layer before moving to the next:

### Phase 1: Foundation (Weeks 1-2)
1. **Configuration System** — Settings loading, merging, JSONC parsing
2. **API Client** — Multi-provider client factory (start with direct Anthropic API)
3. **Message Types** — Define the message model (user, assistant, content blocks)
4. **Basic Query Loop** — Single-turn: user message → API call → text response
5. **Simple CLI** — Argument parsing, basic stdin/stdout interaction

### Phase 2: Tool System (Weeks 3-4)
6. **Tool Interface** — Define the `Tool` type with `buildTool()` factory
7. **Permission System (basic)** — Allow/deny rule evaluation, mode checking
8. **Core Tools** — Implement in this order:
   a. `Read` (simplest — read file, return content)
   b. `Glob` (file pattern matching)
   c. `Grep` (content search)
   d. `Write` (create/overwrite file)
   e. `Edit` (find-and-replace in file)
   f. `Bash` (shell command execution)
9. **Query Loop with Tools** — Tool use → permission check → execute → continue

### Phase 3: Context Management (Weeks 5-6)
10. **Token Counting** — Estimate token usage from messages
11. **Auto-Compaction** — Summarize old messages when approaching limit
12. **System Prompt** — Render system prompt with tool descriptions
13. **Message Normalization** — Strip local fields, apply thinking rules
14. **Session Persistence** — Save/load sessions to disk

### Phase 4: Interactive UI (Weeks 7-8)
15. **Terminal UI** — Prompt input, streaming text display, progress indicators
16. **Permission Prompts** — Interactive allow/deny dialogs
17. **Markdown Rendering** — Parse and render markdown in terminal
18. **Diff Display** — Show file edits as colored diffs
19. **Slash Commands** — `/help`, `/clear`, `/compact`, `/model`, etc.

### Phase 5: Advanced Features (Weeks 9-12)
20. **Multi-Agent** — Agent tool, sub-agent spawning, message isolation
21. **MCP Integration** — MCP client, server lifecycle, tool bridging
22. **Plugin System** — Plugin discovery, loading, validation
23. **Skill System** — Skill loading, execution via Skill tool
24. **Hook System** — PreToolUse/PostToolUse/Stop hooks
25. **Advanced Compaction** — Micro-compact, reactive compact, snip compact

### Phase 6: Polish (Weeks 13+)
26. **OAuth Authentication** — PKCE flow, token storage, refresh
27. **Cloud Providers** — Bedrock, Vertex, Foundry support
28. **Sandbox** — Process isolation for shell commands
29. **Voice Input** — Speech-to-text integration
30. **Remote Bridge** — WebSocket connection to claude.ai
31. **Telemetry** — OpenTelemetry integration
32. **Settings Migrations** — Handle settings format changes across versions

## 2. Minimum Viable Skeleton

The smallest subset that produces a working system:

```
┌─────────────────────────────────────────────────┐
│ CLI Entry Point                                  │
│   ├── Argument parsing (model, prompt)           │
│   ├── Config loading (API key from env)          │
│   └── API client construction                    │
│                                                  │
│ Query Loop                                       │
│   ├── Send messages to API (streaming)           │
│   ├── Render streamed text to stdout             │
│   ├── On tool_use: execute tool                  │
│   ├── Collect tool_result                        │
│   └── Continue loop or stop                      │
│                                                  │
│ Tools (minimum set)                              │
│   ├── Read — read file contents                  │
│   ├── Write — write file contents                │
│   ├── Edit — find/replace in file                │
│   ├── Bash — execute shell command               │
│   ├── Glob — find files by pattern               │
│   └── Grep — search file contents               │
│                                                  │
│ Permission System (basic)                        │
│   ├── Auto-allow read-only tools                 │
│   └── Ask user for write tools (stdin prompt)    │
└─────────────────────────────────────────────────┘
```

This skeleton can operate in a simple readline loop:
1. Print prompt
2. Read user input
3. Call API with message history + tool definitions
4. Stream response, execute tools, continue
5. Return to step 1

## 3. Interface Contracts to Lock First

Define these interfaces before writing implementation logic:

### 1. Message Type
Lock the shape of `UserMessage`, `AssistantMessage`, `SystemMessage` and their content block types. Every module depends on this.

### 2. Tool Interface
Lock the `Tool` interface: `call()`, `checkPermissions()`, `validateInput()`, `inputSchema`, `isReadOnly()`, `isConcurrencySafe()`. All tools implement this.

### 3. ToolUseContext
Lock the context object passed to every tool call. This is the "world" that tools see.

### 4. PermissionResult
Lock the three-way result: `{behavior: 'allow'}`, `{behavior: 'deny', message}`, `{behavior: 'ask', message}`.

### 5. Query Parameters and Yield Types
Lock what `query()` accepts and what it yields. This is the contract between the REPL/SDK and the core engine.

### 6. Settings Schema
Lock the settings file format (permissions, env, hooks, mcpServers). Multiple modules read this.

### 7. API Request/Response Format
Lock how messages are serialized for the API and how streaming events are parsed. This is the contract with the external API.

## 4. Hardest Parts (Ranked by Difficulty)

### Difficulty: Very Hard

1. **Custom Terminal UI Framework** — The Ink fork with Yoga layout, ANSI rendering, mouse support, selection, and a custom React reconciler is by far the most complex subsystem. Consider using an existing TUI library instead of building this from scratch.

2. **Streaming Response Handler** — Correctly reassembling partial JSON for tool inputs, handling thinking blocks, managing abort signals, and recovering from connection drops while yielding events to an async generator is extremely tricky.

3. **Context Compaction** — Multiple compaction strategies (auto, micro, reactive, snip) with correct token tracking across compaction boundaries, preservation of recent messages, and maintenance of message invariants.

### Difficulty: Hard

4. **Permission System** — Layered rule evaluation with pattern matching, hook integration, mode-specific behavior, and denial tracking. The precedence rules are subtle.

5. **Multi-Agent Orchestration** — Sub-agent spawning with isolated contexts, shared infrastructure state, progress forwarding, and clean abort/cleanup.

6. **Shell Command Sandbox** — Platform-specific sandboxing (Seatbelt on macOS, seccomp on Linux) that allows legitimate development operations while blocking dangerous ones.

7. **MCP Server Lifecycle** — Supporting three transport types (stdio, SSE, HTTP), server crash detection/restart, OAuth for MCP servers, and tool schema normalization.

### Difficulty: Medium

8. **OAuth PKCE Flow** — Standard OAuth 2.0 with PKCE, but with keychain integration, token refresh, and multi-instance race condition handling.

9. **Session Persistence** — JSONL serialization, resume with message replay, and maintaining valid message state.

10. **Plugin/Skill System** — Discovery, validation, loading, and integration of extensible components.

11. **Settings Management** — Multi-source merging with precedence, MDM integration, migrations, and JSONC parsing.

### Difficulty: Lower

12. **Individual Tool Implementations** — Most tools are straightforward once the interface is locked. The Bash tool is the most complex due to sandboxing.

13. **Slash Commands** — Mostly self-contained UI commands with simple implementations.

14. **API Client Factory** — Straightforward SDK usage with provider switching, though the retry/fallback logic adds complexity.

## 5. Traps & Pitfalls

### The Thinking Block Trap
Thinking blocks have three rules that MUST be obeyed:
1. Must be part of a query with `max_thinking_length > 0`
2. Must not be the last block in a message
3. Must be preserved for the entire assistant trajectory

Breaking any rule causes an API error with an unhelpful message. The fix requires understanding the trajectory concept: tool_use message → tool_result message → next assistant message are all one trajectory.

### The Tool Result Pairing Trap
If the model requests 3 tool uses and you only return 2 tool results, the API rejects the request. You MUST generate results for ALL tool uses, even if some were aborted or failed. Use synthetic error results for interrupted tools.

### The Prompt Cache Invalidation Trap
The system prompt is divided into sections with `cache_control` markers. If you change the order of sections, add a section in the middle, or modify text in a cached section, you bust the prompt cache. This silently increases API costs by 5-10x. Keep the prompt prefix maximally stable.

### The Concurrent Tool State Trap
Tools marked `isConcurrencySafe` can run in parallel, but the query loop must correctly assemble ALL tool results before sending the next API request. If a concurrent tool modifies the toolUseContext (via contextModifier), that modification must be applied after all parallel tools complete.

### The JSONC Settings Trap
Settings files use JSONC (JSON with comments). If you use standard `JSON.parse()`, it will fail on any settings file with comments. Users WILL have comments in their settings files.

### The macOS Keychain Trap
Keychain access on macOS requires the calling process to be signed or have keychain access entitlements. In SSH sessions, keychain access may fail. Always have a fallback to environment variable auth.

### The MCP stdio Trap
MCP stdio transport communicates via the subprocess's stdin/stdout. If the MCP server writes debug output to stdout (instead of stderr), it corrupts the JSON-RPC protocol. Always validate incoming messages and handle parse errors gracefully.

### The Token Count Estimation Trap
You cannot know the exact token count without calling the tokenizer API (which costs time/money). The system uses estimation heuristics (chars/4 for English). These estimates can be off by 20-30% for non-English text, code, or JSON. Build compaction thresholds with safety margin.

### The Feature Flag Build Trap
`feature('FLAG')` must be a top-level call for Bun's dead code elimination. If you wrap it in a function or use it with a variable, the dead code remains in the bundle. The `require()` pattern for conditional imports is also specific to how Bun handles this.

### The Abort Signal Propagation Trap
AbortController signals must propagate correctly through the tool execution chain. If a tool spawns a subprocess and the user presses Escape, the subprocess must be killed, the partial result discarded, and a synthetic tool result generated. Missing any step leaves orphaned processes or invalid message state.

## 6. Verification Checklist

For each module, verify against the original system's behavior:

### API Client
- [ ] Can authenticate with all 5 provider types
- [ ] Streaming correctly parses all SSE event types
- [ ] Retry logic backs off correctly on 429/529 errors
- [ ] OAuth token refresh happens before expiry
- [ ] Custom headers are forwarded correctly

### Query Loop
- [ ] Single-turn text response works
- [ ] Multi-turn tool use → result → continue works
- [ ] max_tokens recovery continues up to 3 times
- [ ] prompt_too_long triggers reactive compaction
- [ ] Stop hooks fire on end_turn
- [ ] User interruption generates synthetic tool results

### Tool System
- [ ] All tool input schemas validate correctly
- [ ] Permission check follows precedence rules
- [ ] Concurrent tools execute in parallel
- [ ] Tool results exceeding size limit are persisted to disk
- [ ] Error results are returned to model (not thrown)

### Permission System
- [ ] Policy deny cannot be overridden
- [ ] All permission modes behave correctly
- [ ] Pattern matching works (e.g., `Bash(git *)` matches `git status`)
- [ ] Hook integration works (exit code 0 = approve, 2 = deny)
- [ ] "Always allow" saves rule to correct source

### Context Management
- [ ] Auto-compact triggers at correct threshold
- [ ] Compacted messages maintain valid structure
- [ ] Recent messages are preserved
- [ ] Token tracking is accurate after compaction
- [ ] Multiple compaction cycles work correctly

### Terminal UI
- [ ] Streaming text renders character-by-character
- [ ] Markdown renders with formatting
- [ ] Code blocks have syntax highlighting
- [ ] File diffs show as colored diff
- [ ] Permission prompts accept keyboard input
- [ ] Escape interrupts current operation

### Session Management
- [ ] Session saves to disk after each turn
- [ ] Session resumes with complete history
- [ ] Resumed session can continue conversation
- [ ] Multiple sessions tracked independently

### Multi-Agent
- [ ] Sub-agent gets isolated message history
- [ ] Sub-agent can use tools
- [ ] Sub-agent result returns to parent
- [ ] Nested agents (agent spawns agent) work
- [ ] Agent abort cleans up child processes
