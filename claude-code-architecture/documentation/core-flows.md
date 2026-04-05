---
title: "Claude Code CLI — Core Business Flows"
scope: "Critical user-facing and system-facing workflows"
last-analyzed: "2026-04-02"
---

# Core Flows

## 1. Startup & Initialization

### Purpose
Bootstrap the application from shell invocation to interactive REPL.

### Actors
- Developer (user), Bun runtime, Filesystem, API endpoint, OAuth provider

### Preconditions
- Bun runtime installed
- Valid authentication configured (API key, OAuth, or cloud IAM)

### Postconditions
- Interactive REPL displayed with prompt ready
- All tools registered, MCP servers connected, plugins loaded

### Step-by-Step Walkthrough

1. Shell wrapper (`bin/claude-haha`) invokes Bun with environment file
2. `preload.ts` executes: sets `globalThis.MACRO` with version metadata
3. `cli.tsx` entry point checks for fast-path flags (`--version`, `--daemon-worker`, MCP modes)
4. If no fast-path, imports `main.tsx`
5. `init()` runs (memoized): enables configs, sets up graceful shutdown, configures network (mTLS, proxy, pre-connect)
6. MDM settings and keychain reads happen in parallel during module evaluation
7. System context gathered (OS info, git status, working directory)
8. GrowthBook feature flags initialized
9. Policy limits and remote managed settings loaded
10. Built-in plugins, skills, and MCP servers initialized
11. Commander.js parses CLI arguments
12. Commands registered (80+ built-in + MCP + plugin commands)
13. If no subcommand matched, REPL launches
14. React Ink component tree mounted
15. Trust dialog shown if first run
16. Session start hooks executed
17. Prompt input ready for user

```mermaid
flowchart TD
    A[Shell Wrapper] --> B{Fast Path?}
    B -->|--version| C[Print version, exit]
    B -->|--daemon-worker| D[Start daemon worker]
    B -->|MCP mode| E[Start MCP server]
    B -->|No| F[Load main.tsx]
    F --> G["init() - Config, Auth, Network"]
    G --> H[Load GrowthBook, Policies]
    H --> I[Init Plugins, Skills, MCP]
    I --> J[Parse CLI args]
    J --> K{Subcommand?}
    K -->|Yes| L[Execute command]
    K -->|No| M[Launch REPL]
    M --> N[Mount React Ink tree]
    N --> O[Trust dialog if needed]
    O --> P[Session start hooks]
    P --> Q[Ready for input]
```

### Edge Cases & Failure Modes
- **Config parse error:** Shows dialog with error details, continues with defaults
- **OAuth token expired:** Refresh attempted; if refresh fails, prompts for re-login
- **MCP server fails to start:** Warning shown, tools from that server unavailable
- **Network unavailable:** API pre-connect fails silently; error surfaces on first query
- **MDM settings unreadable:** Skipped silently, fallback to file-based settings

---

## 2. User Query → Model Response (The Query Loop)

### Purpose
Process a user message through the LLM and execute any requested tools.

### Actors
- User, Query Engine, Anthropic API, Tool System, Permission System, Compaction Engine

### Preconditions
- Session active with at least one user message
- Valid authentication

### Postconditions
- Model response rendered to terminal
- Any tool results applied (files edited, commands run, etc.)
- Token usage tracked

### Step-by-Step Walkthrough

1. User types message and presses Enter/Shift+Enter
2. Message added to conversation history
3. Memory prefetch starts (async background search for relevant CLAUDE.md content)
4. Attachment messages injected (CLAUDE.md, memories)
5. System prompt rendered (static sections + tool descriptions + context)
6. Messages normalized for API (strip local fields, apply thinking rules)
7. API request sent with streaming enabled
8. Stream events processed:
   a. Text deltas → rendered to terminal in real-time
   b. Thinking deltas → rendered in thinking panel
   c. Tool use blocks → collected with streaming JSON assembly
9. On `stop_reason: tool_use`:
   a. For each tool use block, concurrency check determines parallel vs. sequential
   b. Permission check for each tool (rules → hooks → user prompt)
   c. Tool executed, result collected
   d. Tool results assembled into user message
   e. Loop continues (go to step 6)
10. On `stop_reason: end_turn`:
    a. Stop hooks executed
    b. Auto-compact check: if token usage high, trigger compaction
    c. Tool use summary generated (async)
    d. Response complete, prompt ready for next input
11. On `stop_reason: max_tokens`:
    a. Recovery loop: up to 3 attempts with increased token budget
    b. If recovery exhausted, response shown as-is

```mermaid
flowchart TD
    A[User submits message] --> B[Inject attachments/memories]
    B --> C[Render system prompt]
    C --> D[Normalize messages for API]
    D --> E[Stream API request]
    E --> F{Stream events}
    F -->|Text delta| G[Render to terminal]
    F -->|Tool use| H[Collect tool uses]
    F -->|Thinking| I[Render thinking panel]
    F -->|Message stop| J{Stop reason?}
    J -->|tool_use| K[Check permissions]
    K --> L{Permitted?}
    L -->|Yes| M[Execute tools]
    L -->|No| N[Return denial to model]
    M --> O[Collect results]
    N --> O
    O --> D
    J -->|end_turn| P[Execute stop hooks]
    P --> Q[Check auto-compact]
    Q --> R[Ready for next input]
    J -->|max_tokens| S{Recovery attempts < 3?}
    S -->|Yes| T[Increase budget, continue]
    T --> D
    S -->|No| R
```

### Sequence Diagram

```mermaid
sequenceDiagram
    participant U as User
    participant R as REPL
    participant Q as Query Loop
    participant A as API
    participant T as Tool System
    participant P as Permissions
    participant C as Compaction

    U->>R: Submit message
    R->>Q: query(messages)
    Q->>Q: Inject attachments
    Q->>A: POST /v1/messages (stream)

    loop Stream Processing
        A-->>Q: content_block_delta (text)
        Q-->>R: Yield StreamEvent
        R-->>U: Render text
    end

    A-->>Q: message_stop (tool_use)
    Q->>T: Execute tool(name, input)
    T->>P: checkPermissions()
    P-->>T: Allow
    T-->>Q: ToolResult
    Q->>A: POST /v1/messages (tool_result)

    loop Stream Processing
        A-->>Q: content_block_delta (text)
        Q-->>R: Yield StreamEvent
    end

    A-->>Q: message_stop (end_turn)
    Q->>C: Check token usage
    C-->>Q: OK (no compaction needed)
    Q-->>R: Terminal
    R-->>U: Show prompt
```

### Edge Cases & Failure Modes
- **API timeout:** Retry with backoff; after max retries, show error to user
- **Prompt too long:** Trigger reactive compaction, retry with compacted context
- **Tool execution error:** Error returned to model as tool_result with `is_error: true`
- **User interruption (Escape):** Abort current API call, generate synthetic tool results for pending uses
- **Rate limit:** Wait with countdown timer, retry automatically
- **Network drop:** Retry with backoff; after exhaustion, surface error
- **Fallback model:** If primary model fails with quota error, try fallback model

---

## 3. Tool Execution Flow

### Purpose
Execute a tool requested by the model, with permission checks and error handling.

### Step-by-Step

1. Query loop receives tool_use block from API response
2. Tool looked up by name in tool registry (including aliases)
3. `validateInput()` called — checks input validity
4. `checkPermissions()` called — tool-specific permission logic
5. General permission system evaluates:
   a. Check deny rules (highest precedence)
   b. Check allow rules
   c. Check ask rules
   d. Execute PreToolUse hooks
   e. If undecided, prompt user (or auto-deny in non-interactive mode)
6. If permitted, `call()` executes the tool
7. Progress events emitted during execution
8. Result mapped to `ToolResultBlockParam` for API
9. PostToolUse hooks executed
10. If result exceeds `maxResultSizeChars`, persisted to disk with summary

```mermaid
flowchart TD
    A[Tool use block received] --> B[Look up tool by name]
    B --> C{Tool found?}
    C -->|No| D[Return error: unknown tool]
    C -->|Yes| E[validateInput]
    E --> F{Valid?}
    F -->|No| G[Return validation error]
    F -->|Yes| H[checkPermissions - tool-specific]
    H --> I[Evaluate permission rules]
    I --> J{Deny rule?}
    J -->|Yes| K[Return denial message]
    J -->|No| L{Allow rule?}
    L -->|Yes| M[Execute tool]
    L -->|No| N[Run PreToolUse hooks]
    N --> O{Hook decision?}
    O -->|Approve| M
    O -->|Deny| K
    O -->|Defer| P[Prompt user]
    P --> Q{User decision?}
    Q -->|Allow| M
    Q -->|Allow always| R[Save rule + Execute]
    Q -->|Deny| K
    M --> S[Collect result]
    S --> T[Run PostToolUse hooks]
    T --> U{Result too large?}
    U -->|Yes| V[Persist to disk, return summary]
    U -->|No| W[Return full result]
```

---

## 4. Context Compaction Flow

### Purpose
Reduce conversation context size to stay within model limits.

### Actors
- Query Loop, Compaction Engine, API (Haiku model for summarization)

### Step-by-Step

1. After each API response, token usage is checked
2. If usage exceeds threshold (configurable, ~80% of context window):
   a. **Auto-compact:** Summarize older messages using a fast model (Haiku)
   b. Preserve recent messages (last N turns + all pending tool results)
   c. Replace older messages with compact summary
   d. Insert tombstone markers for removed messages
3. If API returns `prompt_too_long` error:
   a. **Reactive compact:** Emergency compaction triggered
   b. More aggressive summarization
   c. Retry the failed request
4. Within a long tool execution turn:
   a. **Micro-compact:** Time-based clearing of old tool results
   b. Replace stale tool results with brief summaries

```mermaid
stateDiagram-v2
    [*] --> Normal
    Normal --> AutoCompact: Token usage > threshold
    Normal --> ReactiveCompact: prompt_too_long error
    Normal --> MicroCompact: Long turn, stale results

    AutoCompact --> Normal: Summarize + replace messages
    ReactiveCompact --> Normal: Emergency summarize + retry
    MicroCompact --> Normal: Clear old tool results

    Normal --> [*]: Session ends
```

---

## 5. Authentication Flow

### Purpose
Establish identity and API access.

### OAuth Flow (claude.ai)

```mermaid
sequenceDiagram
    participant User
    participant CLI as Claude Code
    participant Browser
    participant OAuth as claude.ai OAuth
    participant Keychain as OS Keychain

    User->>CLI: claude (first run or --force-login)
    CLI->>CLI: Check keychain for existing token
    alt Token exists and valid
        CLI->>CLI: Use existing token
    else Token expired
        CLI->>OAuth: Refresh token
        OAuth-->>CLI: New access token
        CLI->>Keychain: Store new token
    else No token
        CLI->>CLI: Generate PKCE code verifier + challenge
        CLI->>Browser: Open OAuth authorization URL
        Browser->>OAuth: User authorizes
        OAuth->>CLI: Redirect with authorization code
        CLI->>OAuth: Exchange code for tokens
        OAuth-->>CLI: Access token + refresh token
        CLI->>Keychain: Store tokens
    end
    CLI->>CLI: Configure API client with token
```

---

## 6. Multi-Agent (Sub-Agent) Flow

### Purpose
Spawn child agents for parallel or delegated work.

### Step-by-Step

1. Model calls the `Agent` tool with description, prompt, and optional configuration
2. Agent tool creates a new `LocalAgentTask`
3. Child task gets:
   - Cloned `ToolUseContext` with isolated message array
   - Own `AbortController`
   - Restricted tool set (based on agent type configuration)
   - Own system prompt (parent's system prompt + agent-specific instructions)
4. Child task runs its own query loop (same `query()` function)
5. Child can use tools, including spawning further sub-agents (depth-limited)
6. Progress events from child are forwarded to parent's UI
7. On completion, child's final message is returned to parent as tool result
8. Parent continues its own query loop with the agent's result

```mermaid
sequenceDiagram
    participant Parent as Parent Query Loop
    participant AgentTool as Agent Tool
    participant Child as Child Query Loop
    participant API as API

    Parent->>AgentTool: Agent(description, prompt)
    AgentTool->>Child: Create LocalAgentTask
    AgentTool->>Child: Start query loop

    loop Child execution
        Child->>API: API call
        API-->>Child: Response (may include tool_use)
        Child->>Child: Execute tools
    end

    Child-->>AgentTool: Final result
    AgentTool-->>Parent: ToolResult with agent output
    Parent->>API: Continue with agent result
```

---

## 7. Plugin Loading Flow

### Purpose
Discover, validate, and activate plugins.

### Step-by-Step

1. At startup, discover plugins from:
   a. Built-in plugins (bundled in source)
   b. User plugins in `~/.claude/plugins/`
   c. Project plugins in `.claude/plugins/`
   d. Development plugins via `--plugin-dir` flag
   e. DXT-packaged plugins
2. For each plugin, read `plugin.json` manifest
3. Validate manifest against schema
4. Load components:
   a. Commands (slash commands)
   b. Skills (prompt templates)
   c. Agents (agent definitions)
   d. Hooks (lifecycle interceptors)
   e. MCP servers (external tool providers)
5. Register commands and tools with the global registry
6. MCP servers started and connected
7. Plugin errors reported but don't block startup

```mermaid
flowchart TD
    A[Discover plugin sources] --> B[Built-in plugins]
    A --> C[User plugins ~/.claude/plugins/]
    A --> D[Project plugins .claude/plugins/]
    A --> E[Dev plugins --plugin-dir]
    A --> F[DXT packages]

    B & C & D & E & F --> G[Read plugin.json]
    G --> H[Validate manifest]
    H --> I{Valid?}
    I -->|No| J[Log error, skip]
    I -->|Yes| K[Load commands]
    K --> L[Load skills]
    L --> M[Load agents]
    M --> N[Load hooks]
    N --> O[Start MCP servers]
    O --> P[Register in global registry]
```

---

## 8. Session Resume Flow

### Purpose
Resume a previous conversation from persisted state.

### Step-by-Step

1. User invokes `claude --resume` or `/resume` command
2. Session list loaded from `~/.claude/sessions/`
3. User selects session (or most recent is auto-selected)
4. Session messages loaded from JSONL file
5. Messages replayed into conversation state
6. Previous permission rules restored
7. REPL rendered with historical messages
8. User can continue the conversation

---

## 9. Remote Bridge Flow

### Purpose
Allow claude.ai web interface to observe and control local Claude Code session.

### Step-by-Step

1. Bridge enabled via config or `/remote-control` command
2. CLI creates a session on claude.ai API
3. WebSocket connection established
4. Events from local session forwarded to bridge (messages, tool uses, progress)
5. Inbound prompts from web interface injected into local message queue
6. Session URL displayed for sharing
7. On disconnect, automatic reconnection with backoff
