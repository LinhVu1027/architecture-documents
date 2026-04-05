# Error Handling & Resilience Patterns

> How Claude Code survives API failures, context overflows, tool crashes, network outages, and user cancellations without losing work. Every diagram is a Mermaid diagram you can render in any Markdown viewer.

---

## Table of Contents

1. [Resilience Architecture Overview](#1-resilience-architecture-overview)
2. [API Retry & Fallback](#2-api-retry--fallback)
3. [Context Overflow Recovery](#3-context-overflow-recovery)
4. [Max Output Token Recovery](#4-max-output-token-recovery)
5. [Tool Execution Error Handling](#5-tool-execution-error-handling)
6. [MCP Server Failure Handling](#6-mcp-server-failure-handling)
7. [Rate Limiting & Backoff](#7-rate-limiting--backoff)
8. [Cancellation & Abort Propagation](#8-cancellation--abort-propagation)
9. [Session Crash Recovery](#9-session-crash-recovery)
10. [Graceful Degradation](#10-graceful-degradation)
11. [Error Boundary Architecture](#11-error-boundary-architecture)

---

## 1. Resilience Architecture Overview

Claude Code's error handling follows the principle: **every error is an opportunity for recovery, not a reason to crash**.

```mermaid
flowchart TD
    subgraph ErrorTypes["Error Categories"]
        Transient["Transient Errors\n(retry will fix)\n- Rate limits\n- Network timeouts\n- API 500s"]
        Recoverable["Recoverable Errors\n(fallback available)\n- Context overflow\n- Max output tokens\n- Model unavailable"]
        ToolErrors["Tool Errors\n(model can self-correct)\n- Invalid input\n- File not found\n- Permission denied"]
        Fatal["Fatal Errors\n(must stop)\n- Auth failure\n- Unrecoverable crash\n- User abort"]
    end

    subgraph Strategies["Recovery Strategies"]
        Retry["Retry with backoff"]
        Fallback["Fallback model/path"]
        Compact["Context compaction"]
        SelfCorrect["Model self-correction"]
        Resume["Session resume"]
        Degrade["Graceful degradation"]
    end

    Transient --> Retry
    Recoverable --> Fallback
    Recoverable --> Compact
    ToolErrors --> SelfCorrect
    Fatal --> Resume
    Fatal --> Degrade

    style Transient fill:#f39c12,color:#fff
    style Recoverable fill:#e67e22,color:#fff
    style ToolErrors fill:#f1c40f,color:#333
    style Fatal fill:#e74c3c,color:#fff
    style Strategies fill:#27ae60,color:#fff
```

---

## 2. API Retry & Fallback

```mermaid
flowchart TD
    subgraph Request["API Request"]
        Call["sdk.beta.messages.stream(params)"]
    end

    subgraph Retry["withRetry() Wrapper (withRetry.ts)"]
        Attempt["Attempt API call"]
        Check{Error type?}

        Attempt --> Check

        Check -->|"429 Rate Limit"| RateLimit["Wait with exponential backoff\n(respects Retry-After header)"]
        Check -->|"500/502/503"| ServerError["Retry with backoff\n(up to N attempts)"]
        Check -->|"Network timeout"| Timeout["Retry with increased timeout"]
        Check -->|"529 Overloaded"| Overloaded["Longer backoff\n+ notify user"]

        RateLimit --> Attempt
        ServerError --> Attempt
        Timeout --> Attempt
        Overloaded --> Attempt
    end

    subgraph Fallback["Model Fallback"]
        Check -->|"Model unavailable\nor persistent failure"| FallbackModel

        FallbackModel["FallbackTriggeredError\n→ Switch to fallback model\n(e.g., opus → sonnet)"]
        FallbackModel --> RetryWithNew["Retry entire request\nwith fallback model"]
    end

    subgraph Success["Success Path"]
        Check -->|"200 OK"| Stream["Process response stream"]
    end

    subgraph GiveUp["Give Up"]
        Check -->|"401/403 Auth"| AuthError["Show auth error\nPrompt re-authentication"]
        Check -->|"Max retries"| MaxRetry["Show error to user\nPreserve conversation state"]
    end

    Request --> Retry

    style Retry fill:#f39c12,color:#fff
    style Fallback fill:#e67e22,color:#fff
    style Success fill:#27ae60,color:#fff
    style GiveUp fill:#e74c3c,color:#fff
```

### Design Insight: FallbackTriggeredError Pattern

The `FallbackTriggeredError` is a custom error type that signals the query loop to switch models:

```mermaid
sequenceDiagram
    participant Query as query.ts
    participant Retry as withRetry.ts
    participant API as Claude API

    Query->>Retry: Call with primary model (opus)
    Retry->>API: POST /messages (opus)
    API-->>Retry: 429 or persistent error

    Retry->>Retry: Exhausted retries for primary

    alt Fallback model configured
        Retry-->>Query: throw FallbackTriggeredError
        Query->>Query: Catch error, switch to fallback
        Query->>Retry: Retry with fallback model (sonnet)
        Retry->>API: POST /messages (sonnet)
        API-->>Retry: 200 OK
        Retry-->>Query: Stream response
    else No fallback
        Retry-->>Query: throw original error
    end
```

**Why a custom error type?** The retry layer doesn't know about model fallback — that's the query loop's concern. By throwing a specific error type, the separation of concerns is maintained: `withRetry` handles transient failures, `query()` handles strategic model switching.

---

## 3. Context Overflow Recovery

```mermaid
flowchart TD
    subgraph Normal["Normal Operation"]
        Messages["Messages accumulate\nin conversation history"]
        Check{"Estimated tokens >\ncontext threshold?"}
        Messages --> Check
        Check -->|No| Continue["Continue normally"]
    end

    subgraph Proactive["Proactive Recovery: Auto-Compact"]
        Check -->|Yes| AutoCompact["Auto-compact:\n1. Fork conversation\n2. Send to Sonnet for summary\n3. Replace old messages\n4. Re-attach CLAUDE.md files\n5. Continue with fresh context"]
    end

    subgraph Reactive["Reactive Recovery: prompt_too_long"]
        APIError["API returns prompt_too_long\n(estimation was too optimistic)"]
        ReactiveCompact["Reactive compact:\n1. Aggressive summarization\n2. Remove more context\n3. Retry immediately"]
        ContextCollapse["Context collapse:\n1. Group related messages\n2. Replace with summaries\n3. Preserve key decisions"]

        APIError --> ReactiveCompact
        APIError --> ContextCollapse
    end

    subgraph CircuitBreaker["Circuit Breaker"]
        MaxFail{"Consecutive\ncompact failures\n> 3?"}
        MaxFail -->|Yes| Stop["Stop compaction attempts\nNotify user of context issue"]
        MaxFail -->|No| RetryCompact["Retry with more\naggressive settings"]
    end

    AutoCompact --> Continue
    ReactiveCompact --> Continue
    ReactiveCompact --> CircuitBreaker
    ContextCollapse --> Continue

    style Normal fill:#27ae60,color:#fff
    style Proactive fill:#f39c12,color:#fff
    style Reactive fill:#e74c3c,color:#fff
    style CircuitBreaker fill:#c0392b,color:#fff
```

### Compaction Error Recovery Chain

```mermaid
stateDiagram-v2
    [*] --> Normal

    Normal --> AutoCompact: Token threshold exceeded
    AutoCompact --> Normal: Compaction successful

    AutoCompact --> ReactiveCompact: Auto-compact insufficient
    ReactiveCompact --> Normal: Compaction successful

    ReactiveCompact --> ContextCollapse: Still too large
    ContextCollapse --> Normal: Collapse successful

    ReactiveCompact --> CircuitBreaker: 3+ consecutive failures
    ContextCollapse --> CircuitBreaker: Collapse failed

    CircuitBreaker --> [*]: Notify user, stop retrying

    note right of AutoCompact
        Uses Sonnet for speed
        Preserves key context
        ~70% token reduction
    end note

    note right of ReactiveCompact
        More aggressive
        May lose detail
        ~80% token reduction
    end note
```

### Design Insight: Why a Circuit Breaker?

Without a circuit breaker, a pathological conversation could enter an infinite loop:
1. Compact → still too long → compact → still too long → ...

The `MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES = 3` constant prevents this runaway. After 3 failures, the system stops trying and surfaces the issue to the user, who can manually clear context or start a new session.

---

## 4. Max Output Token Recovery

```mermaid
flowchart TD
    subgraph Scenario["Scenario"]
        LongResponse["Model generates a\nvery long response\n(code, explanation, etc.)"]
        HitLimit["Hits max_output_tokens\n(response truncated)"]
    end

    subgraph Recovery["Auto-Continue Recovery"]
        LongResponse --> HitLimit

        HitLimit --> Counter{"Recovery count\n< 3?"}

        Counter -->|Yes| AutoContinue["Append system message:\n'please continue'\nto conversation"]
        AutoContinue --> NextIter["Next loop iteration:\nModel continues from\nwhere it left off"]
        NextIter --> HitLimit2{"Hits limit\nagain?"}
        HitLimit2 -->|Yes| Counter
        HitLimit2 -->|No| Done["Response complete"]

        Counter -->|No, 3+ recoveries| Yield["Yield truncated response\n+ warning to user"]
    end

    subgraph Withheld["Withholding Pattern"]
        Note["isWithheldMaxOutputTokens():\nDon't yield the truncated message\nuntil we know if recovery\nwill succeed.\n\nPrevents SDK callers from\nseeing intermediate errors."]
    end

    style Recovery fill:#f39c12,color:#fff
    style Withheld fill:#bb8fce,color:#fff
```

### Design Insight: The Withholding Pattern

```mermaid
sequenceDiagram
    participant REPL as REPL (consumer)
    participant Query as query() generator
    participant API as Claude API

    API-->>Query: Response with stop_reason=max_output_tokens

    Note over Query: DON'T yield this message yet!<br/>isWithheldMaxOutputTokens() = true

    Query->>Query: Check recovery counter < 3

    alt Can recover
        Query->>API: Send "please continue" message
        API-->>Query: Continuation response
        Query-->>REPL: Yield COMBINED response
        Note over REPL: User sees seamless<br/>complete response
    else Cannot recover (3+ attempts)
        Query-->>REPL: Yield truncated response
        Note over REPL: User sees response<br/>with truncation warning
    end
```

**Why withhold?** If Claude Code yields the truncated message immediately, SDK consumers (like the VS Code extension or desktop app) might treat it as a final response and close the session. By withholding until recovery is attempted, the generator protocol remains clean: yielded messages are final.

---

## 5. Tool Execution Error Handling

```mermaid
flowchart TD
    subgraph ToolCall["Tool Execution"]
        Execute["tool.call(input, context)"]
    end

    subgraph ErrorTypes["Error Types"]
        Execute --> ECheck{Error type?}

        ECheck -->|"Validation error"| ValErr["Input validation failed\n(JSON Schema mismatch)"]
        ECheck -->|"Permission denied"| PermErr["User denied permission\nor alwaysDeny matched"]
        ECheck -->|"Runtime error"| RunErr["Tool code threw exception\n(file not found, network, etc.)"]
        ECheck -->|"Timeout"| TimeErr["Tool exceeded time limit"]
        ECheck -->|"Hook blocked"| HookErr["PreToolUse hook blocked\nwith reason"]
    end

    subgraph Recovery["Recovery: Error as Message"]
        ValErr --> ErrMsg1["tool_result: {\n  is_error: true,\n  content: 'Validation error:\nexpected string for path'\n}"]
        PermErr --> ErrMsg2["tool_result: {\n  is_error: true,\n  content: 'Permission denied'\n}"]
        RunErr --> ErrMsg3["tool_result: {\n  is_error: true,\n  content: 'Error: ENOENT\nfile not found'\n}"]
        TimeErr --> ErrMsg4["tool_result: {\n  is_error: true,\n  content: 'Timeout exceeded'\n}"]
        HookErr --> ErrMsg5["tool_result: {\n  is_error: true,\n  content: 'Blocked by hook:\n[reason]'\n}"]
    end

    subgraph ModelResponse["Model Self-Correction"]
        ErrMsg1 --> Correct["Model receives error\nand adjusts approach:\n- Fix input format\n- Try different tool\n- Ask user for help"]
        ErrMsg2 --> Correct
        ErrMsg3 --> Correct
        ErrMsg4 --> Correct
        ErrMsg5 --> Correct
    end

    style ErrorTypes fill:#e74c3c,color:#fff
    style Recovery fill:#f39c12,color:#fff
    style ModelResponse fill:#27ae60,color:#fff
```

### Design Insight: Errors as Messages, Not Exceptions

Every tool error becomes a `tool_result` with `is_error: true` rather than throwing an exception. This is a critical design choice:

| Approach | What Happens |
|---|---|
| **Throw exception** | Query loop catches it, conversation breaks, user must restart |
| **Return error message** | Model sees the error, understands what went wrong, tries a different approach |

Example: Model tries `Read("/nonexistent/file.ts")` → gets error "file not found" → tries `Glob("**/file.ts")` to find the right path → succeeds. The model's ability to self-correct depends on receiving errors as conversational feedback.

### Missing Tool Result Recovery

```mermaid
flowchart TD
    subgraph Problem["The Problem"]
        Interrupt["User presses Ctrl+C\nduring tool execution"]
        Missing["Some tool_use blocks\nhave no matching tool_result"]
        APIReject["API rejects messages:\nevery tool_use needs\na tool_result"]
    end

    subgraph Recovery["yieldMissingToolResultBlocks()"]
        Scan["Scan assistant messages\nfor tool_use blocks"]
        Generate["Generate synthetic\ntool_result for each:\n{\n  is_error: true,\n  content: 'interrupted'\n}"]
        Append["Append to messages\nbefore next API call"]
    end

    Problem --> Recovery

    style Problem fill:#e74c3c,color:#fff
    style Recovery fill:#27ae60,color:#fff
```

---

## 6. MCP Server Failure Handling

```mermaid
flowchart TD
    subgraph Connection["MCP Connection Lifecycle"]
        Start["Start MCP server\n(stdio/SSE/HTTP)"]
        Connect["Establish connection"]
        ListTools["ListTools discovery"]
        Active["Active: proxying tool calls"]
    end

    subgraph Failures["Failure Scenarios"]
        Start --> StartFail{"Start failed?"}
        StartFail -->|Yes| Log["Log error\nContinue without server"]

        Connect --> ConnFail{"Connection lost?"}
        ConnFail -->|Yes| Reconnect["Attempt reconnection\nwith backoff"]

        Active --> ToolFail{"Tool call failed?"}
        ToolFail -->|Yes| ToolErr["Return error to model:\n'MCP server unavailable'"]
    end

    subgraph Degrade["Graceful Degradation"]
        Log --> Degrade1["MCP tools not available\nBuilt-in tools still work\nSession continues"]
        Reconnect --> Degrade2["Reconnection succeeds:\nTools re-discovered\nSession continues"]
        Reconnect --> Degrade3["Reconnection fails:\nMCP tools removed\nSession continues"]
        ToolErr --> Degrade4["Model informed of failure\nCan use alternative tools"]
    end

    style Failures fill:#e74c3c,color:#fff
    style Degrade fill:#27ae60,color:#fff
```

### Design Insight: MCP Failures Never Crash the Session

MCP servers are external processes that can fail independently. Claude Code treats them as **optional enhancements**:
- If a server fails to start → session continues without its tools
- If a connection drops → attempt reconnection, degrade gracefully
- If a tool call fails → model gets an error message and adapts

This is critical because a user might have 5 MCP servers configured, and one flaky server shouldn't prevent them from using Claude Code.

---

## 7. Rate Limiting & Backoff

```mermaid
sequenceDiagram
    participant Client as Claude Code
    participant API as Claude API

    Client->>API: Request 1
    API-->>Client: 200 OK

    Client->>API: Request 2
    API-->>Client: 200 OK

    Client->>API: Request 3
    API-->>Client: 429 Rate Limited
    Note over API: Retry-After: 30

    Client->>Client: Wait 30 seconds<br/>(from Retry-After header)
    Note over Client: Show user:<br/>"Rate limited, waiting 30s..."

    Client->>API: Request 3 (retry)
    API-->>Client: 200 OK

    Client->>API: Request 4
    API-->>Client: 529 Overloaded

    Client->>Client: Exponential backoff:<br/>1s → 2s → 4s → 8s
    Note over Client: Show user:<br/>"API overloaded, backing off..."

    Client->>API: Request 4 (retry)
    API-->>Client: 200 OK
```

### Design Insight: Why Respect Retry-After?

Some retry libraries use fixed exponential backoff. Claude Code respects the `Retry-After` header because:
1. The server knows its own capacity better than the client
2. Ignoring it risks being blocked entirely
3. The header often indicates when capacity will be available (more accurate than exponential guessing)

---

## 8. Cancellation & Abort Propagation

```mermaid
flowchart TD
    subgraph User["User Action"]
        CtrlC["User presses Ctrl+C\nor Escape"]
    end

    subgraph Propagation["Abort Signal Propagation"]
        UserAbort["AbortController.abort()"]
        APICancel["API stream cancelled\n(fetch abort signal)"]
        ToolCancel["Running tools receive\nabort signal"]
        SubagentCancel["Child agents' abort\ncontrollers triggered"]
    end

    subgraph Cleanup["Cleanup Actions"]
        MissingResults["Generate synthetic\ntool_results for\ninterrupted tools"]
        SaveState["Preserve conversation\nstate for /resume"]
        UIReset["Reset UI to\ninput mode"]
    end

    User --> UserAbort
    UserAbort --> APICancel
    UserAbort --> ToolCancel
    UserAbort --> SubagentCancel

    APICancel --> MissingResults
    ToolCancel --> MissingResults
    SubagentCancel --> MissingResults

    MissingResults --> SaveState --> UIReset

    style User fill:#e74c3c,color:#fff
    style Propagation fill:#f39c12,color:#fff
    style Cleanup fill:#27ae60,color:#fff
```

### Linked Abort Controllers for Agents

```mermaid
flowchart TD
    ParentAbort["Parent AbortController"]
    ChildAbort1["Child Agent 1\nAbortController"]
    ChildAbort2["Child Agent 2\nAbortController"]
    GrandChild["Grandchild Agent\nAbortController"]

    ParentAbort -->|"signal linked"| ChildAbort1
    ParentAbort -->|"signal linked"| ChildAbort2
    ChildAbort1 -->|"signal linked"| GrandChild

    ParentAbort -->|"abort()"| Cancel["ALL descendants\ncancelled immediately"]

    style Cancel fill:#e74c3c,color:#fff
```

---

## 9. Session Crash Recovery

```mermaid
flowchart TD
    subgraph Crash["Crash Scenarios"]
        OOM["Out of memory"]
        Kill["Process killed (SIGKILL)"]
        Panic["Unhandled exception"]
        Power["Power loss / system crash"]
    end

    subgraph Persistence["What's Persisted"]
        Messages["Conversation messages\n(written to session storage)"]
        Costs["Cost tracking data\n(saveCurrentSessionCosts)"]
        Settings["Permission decisions\n(saved to settings.json)"]
    end

    subgraph Resume["/resume Command"]
        ListSessions["List recent sessions\n(from storage)"]
        SelectSession["User selects session\nto restore"]
        RestoreMessages["Restore message history"]
        RestoreContext["Rebuild context:\n- System prompt\n- Tool registry\n- MCP connections"]
        Continue["Continue conversation\nwhere it left off"]
    end

    Crash --> Persistence
    Persistence --> Resume

    ListSessions --> SelectSession --> RestoreMessages --> RestoreContext --> Continue

    style Crash fill:#e74c3c,color:#fff
    style Persistence fill:#f39c12,color:#fff
    style Resume fill:#27ae60,color:#fff
```

### Design Insight: Why Session Persistence Over Checkpointing?

| Approach | Pros | Cons |
|---|---|---|
| **Checkpointing** (save full state) | Perfect restoration | Huge storage (AppState is complex), stale tool connections |
| **Message persistence** (chosen) | Lightweight, JSON-serializable | Must rebuild context on resume |

Claude Code chose message persistence because:
1. Messages are already JSON (discriminated unions, plain objects)
2. Context (system prompt, tools, MCP) must be fresh anyway (MCP servers may have restarted)
3. The model can re-establish understanding from conversation history

---

## 10. Graceful Degradation

```mermaid
flowchart TD
    subgraph Features["Features That Degrade"]
        Voice["Voice Input\n(feature-gated)"]
        IDE["IDE Integration\n(WebSocket bridge)"]
        MCP["MCP Servers\n(external processes)"]
        Analytics["Analytics/Telemetry\n(GrowthBook)"]
        Plugins["Plugins\n(optional packages)"]
        Sandbox["OS Sandbox\n(platform-specific)"]
    end

    subgraph Degradation["Degradation Behavior"]
        Voice --> VD["No voice:\nText input still works\nNo error shown"]
        IDE --> ID["No IDE:\nStandalone terminal mode\nFull functionality"]
        MCP --> MD["No MCP:\nBuilt-in tools only\nReduced capabilities"]
        Analytics --> AD["No analytics:\nApp works normally\nNo tracking"]
        Plugins --> PD["No plugins:\nCore features work\nReduced commands"]
        Sandbox --> SD["No sandbox:\nPermission system\nstill protects"]
    end

    style Features fill:#45b7d1,color:#fff
    style Degradation fill:#27ae60,color:#fff
```

### Design Insight: Feature Flags Enable Graceful Degradation

The `feature()` pattern from `bun:bundle` isn't just for build optimization — it's also a degradation mechanism:

```typescript
// If VOICE_MODE is disabled, this entire code path doesn't exist
if (feature('VOICE_MODE')) {
  const voice = require('./voice.js')
  // ... voice setup
}
// App works fine without it
```

Features behind flags can be disabled without affecting the rest of the system. This means:
- **Internal builds** can experiment with unstable features
- **OSS builds** ship only proven, stable features
- **Enterprise deployments** can disable features that violate policy

---

## 11. Error Boundary Architecture

```mermaid
flowchart TD
    subgraph Layers["Error Handling Layers"]
        L1["Layer 1: Tool-Level\ntry/catch in tool.call()\n→ Return error as tool_result"]
        L2["Layer 2: Orchestration-Level\ntoolOrchestration.ts\n→ Catch tool crashes, yield error"]
        L3["Layer 3: Query Loop\nquery.ts try/catch\n→ Recovery paths"]
        L4["Layer 4: REPL\nREPL.tsx error handling\n→ Show error, keep UI alive"]
        L5["Layer 5: React Error Boundary\nComponent-level catch\n→ Fallback UI"]
        L6["Layer 6: Process-Level\nuncaughtException handler\n→ Save state, exit cleanly"]
    end

    L1 --> L2 --> L3 --> L4 --> L5 --> L6

    subgraph Philosophy["Error Handling Philosophy"]
        P1["Innermost layer handles\nif it can recover"]
        P2["Each layer adds context\nbefore propagating"]
        P3["Outermost layer\npreserves user work"]
    end

    style L1 fill:#27ae60,color:#fff
    style L2 fill:#2ecc71,color:#fff
    style L3 fill:#f39c12,color:#fff
    style L4 fill:#e67e22,color:#fff
    style L5 fill:#e74c3c,color:#fff
    style L6 fill:#c0392b,color:#fff
```

### The Error Recovery Decision Tree

```mermaid
flowchart TD
    Error["Error occurs"] --> CanSelfCorrect{"Can model\nself-correct?"}

    CanSelfCorrect -->|"Yes (tool error)"| ReturnToModel["Return error as\ntool_result message"]
    CanSelfCorrect -->|"No"| CanRetry{"Transient\nerror?"}

    CanRetry -->|"Yes (429, 500)"| RetryWithBackoff["Retry with\nexponential backoff"]
    CanRetry -->|"No"| CanFallback{"Fallback\navailable?"}

    CanFallback -->|"Yes (alt model)"| UseFallback["Switch to\nfallback model"]
    CanFallback -->|"No"| CanCompact{"Context\noverflow?"}

    CanCompact -->|"Yes"| Compact["Compact conversation\nand retry"]
    CanCompact -->|"No"| CanDegrade{"Feature can\nbe disabled?"}

    CanDegrade -->|"Yes"| Degrade["Disable feature\ncontinue without it"]
    CanDegrade -->|"No"| ShowError["Show error to user\npreserve state for /resume"]

    style ReturnToModel fill:#27ae60,color:#fff
    style RetryWithBackoff fill:#2ecc71,color:#fff
    style UseFallback fill:#f39c12,color:#fff
    style Compact fill:#e67e22,color:#fff
    style Degrade fill:#e74c3c,color:#fff
    style ShowError fill:#c0392b,color:#fff
```

### The Resilience Philosophy

Claude Code's error handling follows three principles:

1. **Errors are data, not crashes** — Every tool error becomes a message the model can learn from. The AI gets better at its task by seeing what doesn't work.

2. **Progressive escalation** — Try the cheapest recovery first (retry), then more expensive ones (model fallback, compaction), and only stop when all options are exhausted.

3. **Never lose user work** — Even a fatal crash preserves the conversation via session storage. The `/resume` command can restore any session, including ones that ended in errors.
