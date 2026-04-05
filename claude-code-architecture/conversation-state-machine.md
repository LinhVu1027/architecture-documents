# Conversation State Machine & Session Lifecycle

> How Claude Code manages the state transitions of a conversation — from idle to streaming, from tool execution to compaction, from session start to session recovery. Every diagram is a Mermaid diagram you can render in any Markdown viewer.

---

## Table of Contents

1. [Why State Management Matters](#1-why-state-management-matters)
2. [The REPL State Machine](#2-the-repl-state-machine)
3. [The Query Loop State Machine](#3-the-query-loop-state-machine)
4. [Session Lifecycle](#4-session-lifecycle)
5. [Message Queue & Command Lifecycle](#5-message-queue--command-lifecycle)
6. [Cancellation & Abort Propagation](#6-cancellation--abort-propagation)
7. [Session Persistence & Recovery](#7-session-persistence--recovery)
8. [Worktree Isolation](#8-worktree-isolation)
9. [Inter-Process Communication](#9-inter-process-communication)
10. [The AppState Architecture](#10-the-appstate-architecture)

---

## 1. Why State Management Matters

An AI coding agent has more state transitions than a typical CLI. It must handle: streaming responses, tool execution, permission dialogs, cancellation, compaction, subagent spawning, session switching, and crash recovery — all without losing the user's work.

```mermaid
stateDiagram-v2
    [*] --> Boot
    Boot --> Idle: Setup complete
    Idle --> Processing: User submits prompt
    Processing --> Streaming: API call starts
    Streaming --> ToolExecution: Tool call received
    ToolExecution --> PermissionDialog: Permission needed
    PermissionDialog --> ToolExecution: User approves
    PermissionDialog --> Streaming: User denies
    ToolExecution --> Streaming: Tool complete → more to stream
    Streaming --> Compacting: Context too large
    Compacting --> Streaming: Compact done → retry
    Streaming --> Idle: Response complete
    Processing --> Idle: Error / Cancel
    Streaming --> Idle: User cancels (Escape)
    ToolExecution --> Idle: User cancels (Escape×2)
    Idle --> [*]: User exits (Ctrl+D)
```

---

## 2. The REPL State Machine

The REPL (`src/screens/REPL.tsx`) is the main interactive screen. Its state drives the entire UI.

```mermaid
stateDiagram-v2
    state "REPL States" as REPL {
        [*] --> Idle

        state Idle {
            [*] --> WaitingForInput
            WaitingForInput --> ProcessingSlashCommand: /command entered
            ProcessingSlashCommand --> WaitingForInput: Command complete
            WaitingForInput --> QueuedCommands: Multiple commands queued
            QueuedCommands --> WaitingForInput: Queue drained
        }

        state ActiveQuery {
            [*] --> BuildingContext
            BuildingContext --> CallingAPI
            CallingAPI --> StreamingResponse
            StreamingResponse --> ExecutingTools
            ExecutingTools --> WaitingForPermission
            WaitingForPermission --> ExecutingTools: Approved
            ExecutingTools --> CallingAPI: More tools → next turn
            StreamingResponse --> PostProcessing
            PostProcessing --> RunningStopHooks
        }

        state Recovery {
            [*] --> ErrorDisplay
            ErrorDisplay --> RetryDecision
            RetryDecision --> ActiveQuery: Auto-retry
            RetryDecision --> Idle: Give up
        }

        Idle --> ActiveQuery: User submits prompt
        ActiveQuery --> Idle: Query complete
        ActiveQuery --> Recovery: API error
        Recovery --> Idle: Recovery complete
        ActiveQuery --> Idle: User cancels

        state SessionManagement {
            SwitchSession
            RestoreSession
            ClearSession
        }

        Idle --> SessionManagement: /resume, /clear, switch
        SessionManagement --> Idle: Complete
    }
```

### Key UI States

| State | Spinner | Prompt Input | Permission Dialog |
|---|---|---|---|
| Idle | Hidden | Visible, editable | Hidden |
| BuildingContext | "Thinking..." | Hidden | Hidden |
| StreamingResponse | "Responding..." | Hidden | Hidden |
| ExecutingTools | "Running tool_name..." | Hidden | Hidden |
| WaitingForPermission | Paused | Hidden | Visible |
| PostProcessing | "Processing..." | Hidden | Hidden |
| Error | Hidden | Visible | Hidden |

---

## 3. The Query Loop State Machine

The query loop (`src/query.ts`) is the core agentic loop. It's an `AsyncGenerator` that yields messages and events.

```mermaid
flowchart TD
    subgraph QueryLoop["query() AsyncGenerator"]
        Start["Entry: messages + systemPrompt"]

        subgraph Iteration["Loop Iteration"]
            PrepareContext["1. Prepare context<br/>- Apply tool result budget<br/>- History snip<br/>- Token budget check"]
            SendToAPI["2. API Call<br/>- normalizeMessagesForAPI()<br/>- Stream response"]
            ProcessStream["3. Process stream<br/>- Yield text/thinking blocks<br/>- Collect tool_use blocks"]
            ExecuteTools["4. Execute tools<br/>- StreamingToolExecutor<br/>- Yield tool results"]
            PostSampling["5. Post-sampling hooks<br/>- executePostSamplingHooks()"]
        end

        subgraph Transitions["Continue Transitions"]
            ToolUse["TOOL_USE: model called tools<br/>→ continue with tool results"]
            MaxOutput["MAX_OUTPUT_TOKENS: response truncated<br/>→ continue with recovery message"]
            ReactiveCompact["REACTIVE_COMPACT: prompt too long<br/>→ compact and retry"]
            TokenBudget["TOKEN_BUDGET: under budget<br/>→ inject nudge, continue"]
            Fallback["FALLBACK: primary model failed<br/>→ retry with fallback model"]
        end

        subgraph Terminal["Terminal States"]
            EndTurn["END_TURN: no tool use<br/>→ return to user"]
            MaxTurns["MAX_TURNS: hit turn limit<br/>→ return to user"]
            StopHook["STOP_HOOK: hook terminated<br/>→ return to user"]
            Abort["ABORT: user cancelled<br/>→ return with error"]
        end

        Start --> PrepareContext
        PrepareContext --> SendToAPI
        SendToAPI --> ProcessStream
        ProcessStream --> ExecuteTools
        ExecuteTools --> PostSampling

        PostSampling --> ToolUse
        PostSampling --> MaxOutput
        PostSampling --> TokenBudget
        PostSampling --> EndTurn
        PostSampling --> MaxTurns
        PostSampling --> StopHook

        SendToAPI -->|"prompt_too_long"| ReactiveCompact
        SendToAPI -->|"model error"| Fallback

        ToolUse --> PrepareContext
        MaxOutput --> PrepareContext
        ReactiveCompact --> PrepareContext
        TokenBudget --> PrepareContext
        Fallback --> PrepareContext
    end

    style QueryLoop fill:#2c3e50,color:#fff
    style Transitions fill:#f39c12,color:#333
    style Terminal fill:#27ae60,color:#fff
```

### The State Object

```typescript
type State = {
  messages: Message[]                    // Conversation history
  toolUseContext: ToolUseContext          // Shared tool state
  autoCompactTracking: AutoCompactTrackingState  // Compact state
  maxOutputTokensRecoveryCount: number   // Recovery attempts
  hasAttemptedReactiveCompact: boolean   // Emergency compact flag
  maxOutputTokensOverride: number | undefined
  pendingToolUseSummary: Promise<...>    // Background summary
  stopHookActive: boolean               // Stop hooks running
  turnCount: number                     // Current turn number
  transition: Continue | undefined      // Why we continued
}
```

---

## 4. Session Lifecycle

A session goes through distinct phases from creation to termination.

```mermaid
flowchart TD
    subgraph Creation["Session Creation"]
        NewSession["New session<br/>(fresh start)"]
        ResumeSession["Resume session<br/>(/resume command)"]
        SwitchSession["Switch session<br/>(session selector)"]
    end

    subgraph Setup["Setup Phase (setup.ts)"]
        S1["1. Node.js version check"]
        S2["2. Session ID creation/restoration"]
        S3["3. UDS messaging server start"]
        S4["4. Git root detection"]
        S5["5. Worktree creation (if enabled)"]
        S6["6. Session memory initialization"]
        S7["7. Hook configuration snapshot"]
        S8["8. Background housekeeping launch"]
        S1 --> S2 --> S3 --> S4 --> S5 --> S6 --> S7 --> S8
    end

    subgraph Active["Active Session"]
        Conversation["Conversation loop<br/>(REPL ↔ query loop)"]
        CostTracking["Cost tracking<br/>(per-turn accumulation)"]
        Transcript["Transcript recording<br/>(session storage)"]
        Activity["Activity tracking<br/>(idle detection)"]
    end

    subgraph Termination["Session Termination"]
        T1["User exits (Ctrl+D / Ctrl+C)"]
        T2["Save session costs"]
        T3["Log analytics event"]
        T4["Print cost summary"]
        T5["Cleanup worktree (if any)"]
        T6["Stop background workers"]
        T1 --> T2 --> T3 --> T4 --> T5 --> T6
    end

    Creation --> Setup --> Active --> Termination

    style Creation fill:#2980b9,color:#fff
    style Setup fill:#8e44ad,color:#fff
    style Active fill:#27ae60,color:#fff
    style Termination fill:#e74c3c,color:#fff
```

### Session Storage Structure

```
~/.claude/projects/<project-hash>/
├── sessions/
│   └── <session-id>/
│       ├── transcript.jsonl     # Full message history
│       ├── tool-results/        # Persisted oversized tool outputs
│       ├── agent-<id>/          # Subagent transcripts
│       └── metadata.json        # Session metadata
├── config.json                  # Project config (cumulative costs)
├── CLAUDE.md                    # Project memory
└── memory/                      # Auto-memory files
    ├── MEMORY.md               # Memory index
    ├── user_role.md            # User memories
    ├── feedback_testing.md     # Feedback memories
    └── project_auth.md         # Project memories
```

---

## 5. Message Queue & Command Lifecycle

Messages and commands are queued and processed in priority order.

```mermaid
flowchart TD
    subgraph Input["Input Sources"]
        UserType["User types in prompt"]
        SlashCmd["User types /command"]
        QueuedCmd["Queued commands<br/>(from startup args)"]
        HookMsg["Hook-injected messages"]
        AgentMsg["Subagent messages<br/>(SendMessage tool)"]
    end

    subgraph Queue["Message Queue Manager"]
        Enqueue["enqueue(message, priority)"]
        Sort["Sort by priority<br/>(higher priority first)"]
        Dequeue["getCommandsByMaxPriority()"]
        Consume["consume(uuid)"]
    end

    subgraph Processing["Processing"]
        IsSlash{"Is slash<br/>command?"}
        RunCmd["Execute command handler<br/>(local, no API call)"]
        RunQuery["Start query loop<br/>(API call + tools)"]
        Lifecycle["Command lifecycle tracking:<br/>started → completed"]
    end

    Input --> Queue
    Queue --> Processing
    IsSlash -->|"Yes"| RunCmd
    IsSlash -->|"No"| RunQuery
    RunCmd --> Lifecycle
    RunQuery --> Lifecycle

    style Input fill:#2980b9,color:#fff
    style Queue fill:#f39c12,color:#333
    style Processing fill:#27ae60,color:#fff
```

---

## 6. Cancellation & Abort Propagation

Cancellation must propagate through multiple layers without losing data.

```mermaid
flowchart TD
    subgraph UserAction["User Cancellation"]
        Esc1["Escape (first press)<br/>→ Cancel current API stream"]
        Esc2["Escape (second press)<br/>→ Abort tool execution"]
        CtrlC["Ctrl+C<br/>→ Force quit if stuck"]
    end

    subgraph Propagation["Abort Propagation"]
        Parent["Parent AbortController"]
        Stream["API Stream<br/>(AbortError terminates SSE)"]
        Tools["Tool Execution<br/>(sibling abort controller)"]
        Subagents["Subagent Query Loops<br/>(child abort controllers)"]
        BashProc["Bash Processes<br/>(SIGTERM → SIGKILL)"]
    end

    subgraph Recovery["Post-Cancel Recovery"]
        MissingResults["Generate missing tool_result blocks<br/>for any incomplete tool_use"]
        ErrorMsg["'The user interrupted the assistant'"]
        ReturnToIdle["Return to idle state<br/>(prompt input visible)"]
    end

    Esc1 --> Parent
    Esc2 --> Parent
    Parent --> Stream
    Parent --> Tools
    Tools --> Subagents
    Tools --> BashProc
    Stream --> Recovery
    Tools --> Recovery

    style UserAction fill:#e74c3c,color:#fff
    style Propagation fill:#f39c12,color:#333
    style Recovery fill:#27ae60,color:#fff
```

### The Missing Tool Result Problem

When the user cancels mid-tool-execution, the API has already received `tool_use` blocks but won't get `tool_result` blocks. The system generates synthetic error results:

```typescript
function* yieldMissingToolResultBlocks(assistantMessages, errorMessage) {
  for (const assistantMessage of assistantMessages) {
    for (const toolUse of assistantMessage.message.content) {
      yield createUserMessage({
        content: [{ type: 'tool_result', content: errorMessage, is_error: true, tool_use_id: toolUse.id }]
      })
    }
  }
}
```

This ensures the message history remains valid (every `tool_use` has a corresponding `tool_result`).

---

## 7. Session Persistence & Recovery

Sessions survive process crashes and can be resumed.

```mermaid
flowchart LR
    subgraph Write["Session Write Path"]
        direction TB
        W1["Every message → transcript.jsonl<br/>(append-only JSONL)"]
        W2["Every API call → usage tracking<br/>(in-memory, periodic flush)"]
        W3["Every tool result → tool-results/<br/>(if persisted to disk)"]
    end

    subgraph Recovery["Session Recovery"]
        direction TB
        R1["Parse transcript.jsonl"]
        R2["Rebuild messages[] array"]
        R3["Restore cost state"]
        R4["Restore file state cache"]
        R5["Resume from last state"]
    end

    subgraph Resume["Session Resume Flow"]
        direction TB
        ListSessions["listSessionsImpl()<br/>Show recent sessions"]
        SelectSession["User selects session"]
        RestoreSession["sessionRestore.ts<br/>Rebuild full state"]
        ContinueREPL["Continue REPL<br/>with restored messages"]
    end

    Write --> Recovery --> Resume

    style Write fill:#2980b9,color:#fff
    style Recovery fill:#8e44ad,color:#fff
    style Resume fill:#27ae60,color:#fff
```

---

## 8. Worktree Isolation

Subagents can run in isolated git worktrees to prevent interference with the main workspace.

```mermaid
flowchart TD
    subgraph Main["Main Session"]
        MainCwd["CWD: /project"]
        MainGit["Branch: main"]
        MainFiles["Working files: modified"]
    end

    subgraph Worktree["Worktree Session"]
        WTCreate["git worktree add<br/>/tmp/worktree-<uuid>"]
        WTCwd["CWD: /tmp/worktree-<uuid>"]
        WTBranch["Branch: worktree-<uuid>"]
        WTFiles["Working files: clean copy"]
    end

    subgraph Cleanup["Worktree Lifecycle"]
        NoChanges{"Agent made<br/>changes?"}
        Remove["git worktree remove<br/>(auto-cleanup)"]
        Keep["Keep worktree<br/>Return path + branch<br/>to parent"]
    end

    Main --> WTCreate --> Worktree
    Worktree --> NoChanges
    NoChanges -->|"No"| Remove
    NoChanges -->|"Yes"| Keep

    style Main fill:#2980b9,color:#fff
    style Worktree fill:#8e44ad,color:#fff
    style Cleanup fill:#27ae60,color:#fff
```

---

## 9. Inter-Process Communication

Claude Code uses Unix Domain Sockets (UDS) for inter-process communication.

```mermaid
flowchart LR
    subgraph MainProcess["Main Process"]
        UDSServer["UDS Messaging Server<br/>(started in setup.ts)"]
        MessageHandler["Message handlers:<br/>- inject_message<br/>- permission_request<br/>- permission_response<br/>- status_update"]
    end

    subgraph Workers["Background Workers"]
        Daemon["Daemon Worker<br/>(--daemon-worker)"]
        Housekeeping["Background Housekeeping<br/>(session cleanup, analytics)"]
    end

    subgraph IDE["IDE Integration"]
        VSCode["VS Code Extension<br/>(reads session state)"]
        JetBrains["JetBrains Plugin<br/>(reads session state)"]
    end

    subgraph Swarm["Swarm / Team"]
        Leader["Leader Process"]
        Workers2["Teammate Workers<br/>(in-process or separate)"]
        Mailbox["Permission Mailbox<br/>(async permission sync)"]
    end

    Workers --> UDSServer
    IDE --> UDSServer
    Leader --> Workers2
    Workers2 --> Mailbox --> Leader

    style MainProcess fill:#2980b9,color:#fff
    style Workers fill:#8e44ad,color:#fff
    style IDE fill:#27ae60,color:#fff
    style Swarm fill:#f39c12,color:#333
```

---

## 10. The AppState Architecture

All application state lives in a centralized `AppState` type.

```mermaid
graph TD
    subgraph AppState["AppState (src/state/AppState.ts)"]
        direction TB
        Messages["messages: Message[]<br/>The conversation history"]
        Tools_["tools: Tool[]<br/>Available tools"]
        Model["model: string<br/>Current model"]
        Permissions["permissionContext: ToolPermissionContext<br/>Permission rules + mode"]
        Session["sessionId: string<br/>Current session ID"]
        Cost["cost: CostState<br/>Accumulated costs"]
        UI["uiState: UIState<br/>Spinner, prompt mode, etc."]
        Agents["activeAgents: Map<AgentId, AgentState><br/>Running subagents"]
        Tasks["tasks: TaskState[]<br/>User-visible task list"]
    end

    subgraph Consumers["State Consumers"]
        REPL2["REPL.tsx<br/>(reads messages, tools, UI state)"]
        Query["query.ts<br/>(reads/writes messages, tools)"]
        ToolExec["Tool execution<br/>(reads permissions, writes results)"]
        CostTracker["Cost tracker<br/>(reads/writes cost state)"]
        StatusBar["Status bar<br/>(reads cost, model, session)"]
    end

    AppState --> Consumers

    style AppState fill:#2c3e50,color:#fff
    style Consumers fill:#27ae60,color:#fff
```

### Why a Custom Store?

The existing documentation covers this (design-decisions.md), but the key insight for the state machine: React state management via Ink's rendering means that **state updates trigger UI re-renders**. The custom store provides:
- **Selective updates** — only re-render components that use changed state
- **Batch updates** — multiple state changes in one render cycle
- **Snapshot isolation** — tools see consistent state during execution
