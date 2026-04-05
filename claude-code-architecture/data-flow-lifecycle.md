# Data Flow & Message Lifecycle

> Complete end-to-end trace of how data flows through Claude Code — from the user's keystroke to terminal pixel, and everything in between. Every diagram is a Mermaid diagram you can render in any Markdown viewer.

---

## Table of Contents

1. [End-to-End Overview](#1-end-to-end-overview)
2. [User Input Path](#2-user-input-path)
3. [Message Construction & Transformation](#3-message-construction--transformation)
4. [System Prompt Assembly](#4-system-prompt-assembly)
5. [API Request Construction](#5-api-request-construction)
6. [Streaming Response Handling](#6-streaming-response-handling)
7. [Tool Result Lifecycle](#7-tool-result-lifecycle)
8. [State Updates During the Loop](#8-state-updates-during-the-loop)
9. [Message Rendering Path](#9-message-rendering-path)
10. [Inter-Process Communication](#10-inter-process-communication)

---

## 1. End-to-End Overview

This is the complete journey of a single user interaction — from typing a message to seeing the AI's response rendered in the terminal.

```mermaid
flowchart TD
    subgraph Input["1. INPUT CAPTURE"]
        Keystroke["User types in terminal"] --> InkInput["Ink captures stdin"]
        InkInput --> PromptInput["PromptInput component\n(React controlled input)"]
        PromptInput --> Submit["User presses Enter"]
    end

    subgraph Transform["2. MESSAGE CONSTRUCTION"]
        Submit --> ParseRefs["Parse @file references\nand slash commands"]
        ParseRefs --> BuildMsg["createUserMessage()\nAttach metadata:\n- UUID, timestamp\n- tool results\n- image attachments"]
        BuildMsg --> AppendHistory["Append to messages[] array\nin AppState"]
    end

    subgraph Context["3. CONTEXT ASSEMBLY"]
        AppendHistory --> SysPrompt["Build system prompt\n(getSystemPrompt + buildEffectiveSystemPrompt)"]
        SysPrompt --> Attachments["Inject CLAUDE.md files\n+ memory attachments"]
        Attachments --> UserCtx["Prepend user context\n(getUserContext)"]
        UserCtx --> SysCtx["Append system context\n(getSystemContext)"]
        SysCtx --> Compact["Context management pipeline:\n1. Tool result budget\n2. Snip compaction\n3. Microcompact\n4. Context collapse\n5. Auto-compact"]
    end

    subgraph API["4. API STREAMING"]
        Compact --> Normalize["normalizeMessagesForAPI()\nConvert internal → API format"]
        Normalize --> BuildReq["Build BetaMessageStreamParams:\n- model, max_tokens, tools[]\n- system prompt, messages\n- betas, thinking config"]
        BuildReq --> Stream["SDK.beta.messages.stream()\n→ SSE event stream"]
    end

    subgraph Process["5. RESPONSE PROCESSING"]
        Stream --> TokenEvents["Stream events:\n- content_block_start\n- content_block_delta\n- message_delta\n- message_stop"]
        TokenEvents --> ParseBlocks["Parse into blocks:\n- TextBlock → display text\n- ThinkingBlock → internal reasoning\n- ToolUseBlock → tool invocations"]
        ParseBlocks --> BuildAssistant["Build AssistantMessage:\n- UUID, model, cost\n- content blocks\n- usage stats"]
    end

    subgraph Tools["6. TOOL EXECUTION"]
        BuildAssistant --> HasTools{Contains\ntool_use blocks?}
        HasTools -->|Yes| Partition["Partition:\nread-only vs write"]
        Partition --> Execute["Execute tools\n(parallel/serial)"]
        Execute --> ToolResult["Build tool_result\nUserMessages"]
        ToolResult --> AppendHistory
        HasTools -->|No| Render
    end

    subgraph Render["7. TERMINAL RENDERING"]
        BuildAssistant --> MsgList["Messages.tsx\n(message list component)"]
        MsgList --> Markdown["Markdown.tsx\n(parse + highlight)"]
        Markdown --> InkRender["Ink renderer\n(React → ANSI escape codes)"]
        InkRender --> Terminal["Terminal stdout\n(user sees response)"]
    end

    style Input fill:#4ecdc4,color:#fff
    style Transform fill:#45b7d1,color:#fff
    style Context fill:#bb8fce,color:#fff
    style API fill:#f39c12,color:#fff
    style Process fill:#ff6b6b,color:#fff
    style Tools fill:#f7dc6f,color:#333
    style Render fill:#27ae60,color:#fff
```

### Design Insight: Why AsyncGenerator for the Loop?

The `query()` function returns `AsyncGenerator<StreamEvent | Message, Terminal>`. This was a deliberate choice:

| Alternative | Why Rejected |
|---|---|
| **Callbacks** | No backpressure — if tokens stream faster than UI renders, events pile up |
| **Promises** | Can't yield intermediate results — streaming needs partial updates |
| **Observables (RxJS)** | Heavy dependency, complex operator chains for a fundamentally sequential flow |
| **AsyncGenerator** | **Chosen** — natural fit for "produce values over time, consumer controls pace" |

The generator pattern means the REPL can `for await (const event of query(...))` and process each token/tool-result at its own pace. The loop naturally pauses API streaming when the UI needs to catch up.

---

## 2. User Input Path

```mermaid
sequenceDiagram
    participant TTY as Terminal (stdin)
    participant Ink as Ink Runtime
    participant PI as PromptInput Component
    participant REPL as REPL Screen
    participant History as History Manager
    participant Query as query() Loop

    TTY->>Ink: Raw keypress event
    Ink->>PI: useInput() hook fires

    alt Normal character
        PI->>PI: Update controlled text state
        PI->>PI: Trigger autocomplete<br/>(files, commands, @mentions)
    else Enter key
        PI->>PI: Validate non-empty input
        PI->>History: addToHistory(text)
        PI->>PI: parseReferences(text)<br/>Resolve @file paths
        PI->>PI: expandPastedTextRefs(text)
        PI->>REPL: onSubmit(processedText)
    else Slash command (/)
        PI->>PI: Match against command registry
        PI->>REPL: Execute command handler
    else Up/Down arrow
        PI->>History: Navigate history
    else Escape (vim mode)
        PI->>PI: Toggle vim normal/insert mode
    end

    REPL->>REPL: createUserMessage(text)
    REPL->>REPL: setState: append message
    REPL->>Query: Start query() generator
    Query-->>REPL: Yield stream events
```

### Design Insight: Why React for a CLI?

Using React + Ink for the terminal UI is counterintuitive but powerful:

1. **Component composition** — The 140+ UI components (PromptInput, Messages, PermissionRequest, DiffView, etc.) compose just like web React
2. **State-driven rendering** — When `AppState` changes, React's reconciliation only re-renders affected terminal regions
3. **Hooks ecosystem** — 80+ custom hooks (useCanUseTool, useTerminalSize, useSearchHighlight, useVoice) provide clean state management
4. **React Compiler** — The `react/compiler-runtime` import enables automatic memoization, crucial for terminal performance where every re-render redraws characters

---

## 3. Message Construction & Transformation

```mermaid
classDiagram
    class Message {
        <<union type>>
        UserMessage
        AssistantMessage
        SystemMessage
        AttachmentMessage
        ProgressMessage
        ToolUseSummaryMessage
        TombstoneMessage
    }

    class UserMessage {
        type: "user"
        uuid: string
        message: MessageParam
        toolUseResult?: string
        sourceToolAssistantUUID?: string
        images?: ImageBlockParam[]
    }

    class AssistantMessage {
        type: "assistant"
        uuid: string
        message: MessageParam
        model: string
        costUSD: number
        durationMs: number
        usage: BetaUsage
        apiError?: StopReason
    }

    class AttachmentMessage {
        type: "attachment"
        uuid: string
        content: string
        source: string
        scope: "project" | "user" | "global"
    }

    class ToolUseSummaryMessage {
        type: "tool_use_summary"
        uuid: string
        summary: string
        toolUseBlockIds: string[]
    }

    Message <|-- UserMessage
    Message <|-- AssistantMessage
    Message <|-- AttachmentMessage
    Message <|-- ToolUseSummaryMessage
```

### Message Transformation Pipeline

```mermaid
flowchart LR
    subgraph Creation["Message Creation (utils/messages.ts)"]
        direction TB
        A["createUserMessage()"] --> B["Assigns UUID"]
        B --> C["Wraps content in\nMessageParam format"]
        C --> D["Attaches metadata:\ntimestamp, source, etc."]
    end

    subgraph Normalization["API Normalization (normalizeMessagesForAPI)"]
        direction TB
        E["Filter internal-only types\n(Progress, Tombstone)"] --> F["Convert AttachmentMessages\ninto system/user content"]
        F --> G["Ensure tool_result pairing\n(every tool_use gets a result)"]
        G --> H["Strip thinking blocks\nfrom non-adjacent messages"]
        H --> I["Output: MessageParam[]"]
    end

    subgraph Enrichment["Context Enrichment (utils/api.ts)"]
        direction TB
        J["prependUserContext()\nAdd git status, env info"] --> K["appendSystemContext()\nAdd system reminders"]
    end

    Creation --> Normalization --> Enrichment

    style Creation fill:#4ecdc4,color:#fff
    style Normalization fill:#45b7d1,color:#fff
    style Enrichment fill:#bb8fce,color:#fff
```

### Design Insight: Why a Union Type for Messages?

Claude Code uses a discriminated union (`type` field) rather than a class hierarchy for messages:

- **Serialization** — Messages must be serializable to JSON for session persistence and compaction. Plain objects serialize trivially; class instances don't.
- **Pattern matching** — `if (msg.type === 'assistant')` is idiomatic TypeScript with full type narrowing. No `instanceof` checks needed.
- **Immutability** — `DeepImmutable<AppState>` wraps the entire state. Immutable plain objects are natural; immutable class instances fight against OOP patterns.

---

## 4. System Prompt Assembly

The system prompt is the most carefully constructed piece of the entire request. It's built in layers:

```mermaid
flowchart TD
    subgraph Layer1["Layer 1: Base System Prompt (constants/prompts.ts)"]
        Base["getSystemPrompt()\n- Core identity & capabilities\n- Tool usage instructions\n- Tone & style guidelines\n- Output efficiency rules\n- Git commit protocol\n- PR creation protocol"]
    end

    subgraph Layer2["Layer 2: Effective Prompt (utils/systemPrompt.ts)"]
        Build["buildEffectiveSystemPrompt()"]
        Base --> Build
        Tools["Tool descriptions & schemas\n(dynamically generated per-tool)"] --> Build
        Agents["Agent definitions\n(from .claude/agents/ + plugins)"] --> Build
        Skills["Available skills list\n(from skills/ + plugins)"] --> Build
        ModelInfo["Model-specific instructions\n(thinking config, capabilities)"] --> Build
    end

    subgraph Layer3["Layer 3: Attachments (utils/attachments.ts)"]
        Build --> AttachMsgs["getAttachmentMessages()"]
        CLAUDE_MD["CLAUDE.md files\n(project, user, global)"] --> AttachMsgs
        Memory["Memory files\n(~/.claude/projects/*/memory/)"] --> AttachMsgs
        PluginMD["Plugin .local.md files"] --> AttachMsgs
    end

    subgraph Layer4["Layer 4: Runtime Context (context.ts)"]
        AttachMsgs --> UserCtx["getUserContext()\n- Current date\n- Git status snapshot\n- Working directory\n- Custom context"]
        UserCtx --> SysCtx["getSystemContext()\n- System reminders\n- Available deferred tools\n- Active tasks\n- Memory refresh"]
    end

    subgraph Layer5["Layer 5: Cache Optimization"]
        SysCtx --> CacheControl["Apply cache_control markers\n- Static prefix → cached\n- Dynamic suffix → uncached\n- splitSysPromptPrefix()"]
    end

    Layer5 --> Final["Final system prompt\nsent to Claude API"]

    style Layer1 fill:#4ecdc4,color:#fff
    style Layer2 fill:#45b7d1,color:#fff
    style Layer3 fill:#bb8fce,color:#fff
    style Layer4 fill:#f39c12,color:#fff
    style Layer5 fill:#ff6b6b,color:#fff
```

### Design Insight: Why Split the System Prompt for Caching?

The `splitSysPromptPrefix()` function divides the system prompt into a **static prefix** and **dynamic suffix**:

- **Static prefix** (tool descriptions, identity, rules) → marked with `cache_control: { type: 'ephemeral' }` → Anthropic's prompt caching reuses this across requests
- **Dynamic suffix** (current date, git status, task list) → changes each request

This saves ~10K tokens of re-processing per API call within a session, reducing both latency and cost.

---

## 5. API Request Construction

```mermaid
sequenceDiagram
    participant Query as query.ts
    participant Claude as claude.ts
    participant SDK as @anthropic-ai/sdk
    participant API as Claude API

    Query->>Claude: createApiStream(messages, systemPrompt, tools, config)

    Claude->>Claude: normalizeMessagesForAPI(messages)
    Note over Claude: Filter internal types,<br/>ensure tool_result pairing,<br/>strip stale thinking blocks

    Claude->>Claude: toolToAPISchema(tools)
    Note over Claude: Convert Tool[] to BetaToolUnion[]<br/>with JSON Schema input_schema

    Claude->>Claude: Build BetaMessageStreamParams
    Note over Claude: {<br/>  model: "claude-opus-4-6",<br/>  max_tokens: 16384,<br/>  system: [...cached + dynamic],<br/>  messages: MessageParam[],<br/>  tools: BetaToolUnion[],<br/>  betas: ["...feature-betas"],<br/>  stream: true,<br/>  thinking: { type, budget_tokens }<br/>}

    Claude->>Claude: getMergedBetas()
    Note over Claude: Merge model betas +<br/>provider-specific betas +<br/>feature flag betas

    Claude->>Claude: computeFingerprintFromMessages()
    Note over Claude: Hash for deduplication<br/>and telemetry

    Claude->>SDK: client.beta.messages.stream(params)
    SDK->>API: POST /v1/messages (SSE)
    API-->>SDK: Stream of BetaRawMessageStreamEvent
    SDK-->>Claude: Async iterator of events
    Claude-->>Query: Yield parsed events
```

### Design Insight: Why the Beta API Surface?

Claude Code uses `client.beta.messages.stream()` rather than the stable API because:

1. **Extended thinking** — The `thinking` parameter for chain-of-thought is a beta feature
2. **Computer use** — Beta tool types support computer_use_20241022
3. **Task budgets** — `output_config.task_budget` for agentic cost control
4. **Prompt caching** — `cache_control` on system prompt blocks
5. **Future-proofing** — New capabilities land in beta first; Claude Code ships weekly and needs them immediately

---

## 6. Streaming Response Handling

```mermaid
flowchart TD
    subgraph SSE["Server-Sent Events from API"]
        E1["message_start\n{id, model, usage}"]
        E2["content_block_start\n{index, type: text|thinking|tool_use}"]
        E3["content_block_delta\n{index, delta: text_delta|thinking_delta|input_json_delta}"]
        E4["content_block_stop\n{index}"]
        E5["message_delta\n{stop_reason, usage}"]
        E6["message_stop"]
        E1 --> E2 --> E3 -->|"repeats"| E3
        E3 --> E4 --> E2
        E4 --> E5 --> E6
    end

    subgraph Handler["Response Handler (claude.ts)"]
        Parse["Parse SSE events"]
        E6 --> Parse

        subgraph BlockAccum["Block Accumulator"]
            TextAcc["TextBlock accumulator\n(concatenate deltas)"]
            ThinkAcc["ThinkingBlock accumulator\n(internal reasoning)"]
            ToolAcc["ToolUseBlock accumulator\n(parse JSON incrementally)"]
        end

        Parse --> BlockAccum

        subgraph StreamExec["StreamingToolExecutor"]
            STEParse["Parse partial tool JSON\nas it streams in"]
            STEStart["Start tool execution\nBEFORE stream completes"]
            STEParse --> STEStart
        end

        ToolAcc -->|"Streaming exec enabled"| StreamExec
    end

    subgraph Yield["Yielded to REPL"]
        StreamEvent["StreamEvent\n{type, content_block, delta}"]
        AssistantMsg["AssistantMessage\n(complete response)"]
    end

    BlockAccum --> StreamEvent
    BlockAccum --> AssistantMsg

    style SSE fill:#45b7d1,color:#fff
    style Handler fill:#ff6b6b,color:#fff
    style StreamExec fill:#f39c12,color:#fff
    style Yield fill:#4ecdc4,color:#fff
```

### Design Insight: StreamingToolExecutor — Starting Before the Stream Ends

The `StreamingToolExecutor` is a performance optimization that starts executing tools **while the API response is still streaming**:

1. As `input_json_delta` events arrive, the executor parses the partial JSON
2. For tools like `Read` or `Glob`, the file path appears early in the JSON
3. The executor begins file I/O before the API finishes generating the rest of the response
4. By the time the full tool_use block arrives, the result may already be ready

This overlaps API latency with I/O latency, reducing perceived response time for multi-tool responses.

---

## 7. Tool Result Lifecycle

```mermaid
flowchart TD
    subgraph Extract["Extract Tool Calls"]
        AssistantMsg["AssistantMessage\nwith tool_use blocks"]
        Extract1["Extract ToolUseBlock[]\nfrom response content"]
        AssistantMsg --> Extract1
    end

    subgraph Partition["Partition by Safety"]
        Extract1 --> Check["Check each tool's\nisReadOnly()"]
        Check --> ReadOnly["Read-only batch:\nGlob, Grep, Read,\nWebFetch, WebSearch"]
        Check --> Write["Write batch:\nEdit, Write, Bash,\nAgent"]
    end

    subgraph Execute["Execute (toolOrchestration.ts)"]
        ReadOnly --> Parallel["Run concurrently\n(up to 10 via\nrunToolsConcurrently)"]
        Write --> Serial["Run serially\n(via runToolsSerially)"]
        Parallel -->|"All complete"| Serial
    end

    subgraph SingleExec["Per-Tool Execution (toolExecution.ts)"]
        direction TB
        S1["Pre-tool hooks\n(PreToolUse event)"]
        S2["Permission check\n(canUseTool())"]
        S3["Input validation\n(JSON Schema)"]
        S4["tool.call(input, context)\n→ AsyncGenerator"]
        S5["Collect progress yields"]
        S6["Collect final result"]
        S7["Post-tool hooks\n(PostToolUse event)"]
        S1 --> S2 --> S3 --> S4 --> S5 --> S6 --> S7
    end

    Execute --> SingleExec

    subgraph Result["Result as Message"]
        SingleExec --> BuildResult["createUserMessage({\n  content: [{\n    type: 'tool_result',\n    tool_use_id: '...',\n    content: result\n  }],\n  toolUseResult: summary\n})"]
        BuildResult --> Append["Append to messages[]\n→ Next loop iteration"]
    end

    style Extract fill:#45b7d1,color:#fff
    style Partition fill:#bb8fce,color:#fff
    style Execute fill:#f7dc6f,color:#333
    style SingleExec fill:#ff6b6b,color:#fff
    style Result fill:#4ecdc4,color:#fff
```

### Design Insight: Why Read-Parallel, Write-Serial?

This concurrency model is driven by **correctness constraints**:

| Operation | Safe to Parallelize? | Reason |
|---|---|---|
| Read file A + Read file B | Yes | No side effects, no ordering dependency |
| Read file A + Grep for pattern | Yes | Both are pure observations |
| Edit file A + Edit file B | **No** | Both may modify overlapping regions; Edit uses line-based patches that shift if another edit changes line numbers |
| Edit file A + Bash(npm test) | **No** | Bash may read the file being edited; ordering matters |
| Bash(cmd1) + Bash(cmd2) | **No** | Shell commands may have implicit ordering dependencies |

The 10-concurrent-read limit prevents file descriptor exhaustion while still being much faster than serial execution for large codebase searches.

---

## 8. State Updates During the Loop

```mermaid
stateDiagram-v2
    [*] --> Idle: App starts

    state Idle {
        [*] --> WaitingForInput
        WaitingForInput: messages = [...history]
        WaitingForInput: isGenerating = false
    }

    Idle --> QueryStarted: User submits prompt

    state QueryStarted {
        [*] --> AppendUserMsg
        AppendUserMsg: messages = [..., userMsg]
        AppendUserMsg: isGenerating = true

        AppendUserMsg --> Streaming
        Streaming: currentAssistantMsg = partial
        Streaming: tokenCount updating

        Streaming --> ToolExecution: tool_use received
        ToolExecution: currentToolExecution = {...}
        ToolExecution: permission requests may fire

        ToolExecution --> AppendResults
        AppendResults: messages = [..., assistantMsg, ...toolResults]

        AppendResults --> Streaming: Loop continues

        Streaming --> Complete: end_turn
        Complete: messages = [..., finalAssistantMsg]
        Complete: isGenerating = false
        Complete: cost updated
    }

    QueryStarted --> Idle: Query complete

    state CompactBranch {
        [*] --> Compacting
        Compacting: messages = [summaryMsg, ...attachments]
        Compacting: Separate API call to Sonnet
    }

    QueryStarted --> CompactBranch: Context limit hit
    CompactBranch --> QueryStarted: Resume with compacted state
```

### The External Store Pattern

```mermaid
flowchart LR
    subgraph Store["createStore<AppState> (store.ts)"]
        State["let state: AppState"]
        GetState["getState() → state"]
        SetState["setState(updater)\n1. Run updater(prev)\n2. Check Object.is()\n3. Update state ref\n4. Call onChange()\n5. Notify listeners"]
        Subscribe["subscribe(listener)\n→ cleanup function"]
    end

    subgraph React["React Integration"]
        useSyncExternalStore["useSyncExternalStore(\n  store.subscribe,\n  store.getState\n)"]
    end

    subgraph Consumers["State Consumers"]
        REPL["REPL.tsx"]
        Messages["Messages.tsx"]
        Spinner["Spinner.tsx"]
        PermDialog["Permission Dialogs"]
        StatusLine["Status Line"]
    end

    Store --> useSyncExternalStore
    useSyncExternalStore --> Consumers

    style Store fill:#ff6b6b,color:#fff
    style React fill:#45b7d1,color:#fff
```

### Design Insight: Why a Custom Store Instead of Zustand/Redux?

The store implementation in `store.ts` is only **35 lines**:

1. **Zero dependencies** — No bundle size cost for a simple get/set/subscribe pattern
2. **React 19 integration** — `useSyncExternalStore` is the React-blessed way to read external state; the custom store implements exactly its contract
3. **DeepImmutable** — AppState is typed as `DeepImmutable<...>`, enforcing immutability at the type level. No middleware needed.
4. **onChange callback** — Single point for side effects (analytics, persistence) without Redux middleware complexity

---

## 9. Message Rendering Path

```mermaid
flowchart TD
    subgraph Data["Message Data"]
        MsgArray["messages: Message[]"]
    end

    subgraph List["VirtualMessageList"]
        Filter["Filter displayable messages\n(skip internal types)"]
        Virtualize["Virtual scrolling\n(only render visible rows)"]
        MsgArray --> Filter --> Virtualize
    end

    subgraph MsgComponent["Per-Message Rendering"]
        TypeSwitch{"message.type?"}
        Virtualize --> TypeSwitch

        UserRender["UserMessage:\n- Plain text\n- @file references highlighted\n- Image thumbnails"]
        AssistantRender["AssistantMessage:\n- Markdown parsing (marked)\n- Syntax highlighting (highlight.js)\n- Thinking block collapse\n- Tool use rendering"]
        ToolRender["Tool Results:\n- Diff view (for edits)\n- File content (for reads)\n- Search results (for grep)\n- Error display"]
        AttachRender["Attachment:\n- CLAUDE.md content\n- Collapsible sections"]

        TypeSwitch -->|user| UserRender
        TypeSwitch -->|assistant| AssistantRender
        TypeSwitch -->|tool_result| ToolRender
        TypeSwitch -->|attachment| AttachRender
    end

    subgraph InkRender["Ink Rendering Pipeline"]
        Measure["Measure terminal dimensions\n(useTerminalSize)"]
        Layout["Yoga layout engine\n(flexbox for terminal)"]
        Paint["Generate ANSI escape codes:\n- Colors (chalk)\n- Box drawing\n- Cursor positioning"]
        Flush["Write to stdout"]
        Measure --> Layout --> Paint --> Flush
    end

    MsgComponent --> InkRender

    style Data fill:#45b7d1,color:#fff
    style List fill:#bb8fce,color:#fff
    style MsgComponent fill:#ff6b6b,color:#fff
    style InkRender fill:#4ecdc4,color:#fff
```

### Design Insight: Why Virtual Scrolling?

Long conversations can have thousands of messages with syntax-highlighted code blocks. Without virtual scrolling:
- Every re-render would process ALL messages (O(n) where n = total messages)
- Terminal flickering from redrawing unchanged content
- Memory pressure from holding rendered output of invisible messages

Virtual scrolling renders only the ~50 visible lines, making rendering O(1) regardless of conversation length. Combined with React's reconciliation, only changed characters are repainted.

---

## 10. Inter-Process Communication

Claude Code communicates across process boundaries through multiple channels:

```mermaid
flowchart TB
    subgraph MainProcess["Main Claude Code Process"]
        REPL["REPL + Query Loop"]
        UDS["Unix Domain Socket Server\n(inter-process messaging)"]
        Bridge["IDE Bridge\n(WebSocket server)"]
    end

    subgraph Subagents["Subagent Processes"]
        Agent1["Agent 1\n(own query loop)"]
        Agent2["Agent 2\n(own query loop)"]
    end

    subgraph IDEs["IDE Extensions"]
        VSCode["VS Code Extension"]
        JetBrains["JetBrains Plugin"]
    end

    subgraph ChromeExt["Chrome Extension"]
        ChromeMCP["Chrome MCP Server\n(--claude-in-chrome-mcp)"]
    end

    subgraph MailboxSystem["Mailbox System (utils/mailbox.ts)"]
        Mailbox["Mailbox instance\n- queue: Message[]\n- waiters: Set<Waiter>\n- send(msg) → notify waiters\n- waitFor(predicate) → Promise"]
    end

    REPL <-->|"SendMessage tool"| MailboxSystem
    Agent1 <-->|"SendMessage tool"| MailboxSystem
    Agent2 <-->|"SendMessage tool"| MailboxSystem

    REPL <-->|"UDS protocol"| UDS
    UDS <-->|"Agent spawn/result"| Subagents

    Bridge <-->|"WebSocket\nJSON-RPC"| IDEs

    REPL <-->|"MCP protocol\n(stdio/SSE)"| ChromeMCP

    style MainProcess fill:#4ecdc4,color:#fff
    style Subagents fill:#45b7d1,color:#fff
    style IDEs fill:#bb8fce,color:#fff
    style MailboxSystem fill:#f39c12,color:#fff
```

### The Mailbox Pattern

```mermaid
sequenceDiagram
    participant Leader as Leader Agent
    participant MB as Mailbox
    participant Worker1 as Worker Agent 1
    participant Worker2 as Worker Agent 2

    Leader->>Worker1: spawn(task: "search for X")
    Leader->>Worker2: spawn(task: "search for Y")

    Worker1->>MB: send({content: "found X in foo.ts"})
    MB->>MB: Check waiters → no match

    Leader->>MB: waitFor(msg => msg.from === "worker1")
    Note over MB: Leader blocks on Promise

    MB->>Leader: Resolve with worker1's message

    Worker2->>MB: send({content: "found Y in bar.ts"})
    Leader->>MB: waitFor(msg => msg.from === "worker2")
    MB->>Leader: Resolve immediately (already queued)
```

### Design Insight: Why Unix Domain Sockets for IPC?

Claude Code uses UDS rather than TCP for inter-process communication:

| Property | UDS | TCP |
|---|---|---|
| **Security** | File-system permissions — only the user can connect | Port scanning exposes the service to local network |
| **Performance** | Kernel bypass (no TCP stack overhead) | Full TCP handshake + stack processing |
| **Discovery** | Socket file path is deterministic | Need port allocation + discovery mechanism |
| **Cleanup** | OS cleans up on process exit | Ports can linger in TIME_WAIT |

The socket path includes the session ID, so multiple Claude Code instances don't collide.

---

## Summary: The Complete Data Journey

```mermaid
graph LR
    K["⌨️ Keystroke"] --> PI["PromptInput"]
    PI --> UM["UserMessage"]
    UM --> SP["+ System Prompt"]
    SP --> CC["Context\nCompaction"]
    CC --> API["Claude API\n(streaming)"]
    API --> AM["AssistantMessage"]
    AM --> TU{"Tool Use?"}
    TU -->|Yes| TE["Tool\nExecution"]
    TE --> TR["ToolResult\nMessage"]
    TR --> CC
    TU -->|No| MR["Markdown\nRenderer"]
    MR --> INK["Ink\n(ANSI)"]
    INK --> T["🖥️ Terminal"]

    style K fill:#4ecdc4,color:#fff
    style API fill:#ff6b6b,color:#fff
    style TE fill:#f7dc6f,color:#333
    style T fill:#27ae60,color:#fff
```

Every step in this pipeline was designed for a specific trade-off:
- **AsyncGenerator** → backpressure-aware streaming
- **Discriminated unions** → type-safe serializable messages
- **Split system prompt** → prompt caching cost savings
- **Read-parallel, write-serial** → correctness with maximum throughput
- **Virtual scrolling** → O(1) rendering regardless of history length
- **UDS** → secure, fast inter-process communication
