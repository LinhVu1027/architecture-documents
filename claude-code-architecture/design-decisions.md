# Design Decisions & Architectural Rationale

> Every major architectural choice in Claude Code was made for a reason. This document explains the **WHY** behind each decision, comparing alternatives and revealing the trade-offs. Every diagram is a Mermaid diagram you can render in any Markdown viewer.

---

## Table of Contents

1. [Why Bun Over Node.js](#1-why-bun-over-nodejs)
2. [Why React + Ink for a CLI](#2-why-react--ink-for-a-cli)
3. [Why AsyncGenerator for the Agentic Loop](#3-why-asyncgenerator-for-the-agentic-loop)
4. [Why a Custom Store Over Zustand/Redux](#4-why-a-custom-store-over-zustandredux)
5. [Why Commander.js for CLI Parsing](#5-why-commanderjs-for-cli-parsing)
6. [Why MCP (Model Context Protocol)](#6-why-mcp-model-context-protocol)
7. [Why Streaming Over Batch API Calls](#7-why-streaming-over-batch-api-calls)
8. [Why Read-Parallel, Write-Serial Tool Execution](#8-why-read-parallel-write-serial-tool-execution)
9. [Why Multiple Compaction Strategies](#9-why-multiple-compaction-strategies)
10. [Why Feature Flags for Dead Code Elimination](#10-why-feature-flags-for-dead-code-elimination)
11. [Why Fork Context for Subagents](#11-why-fork-context-for-subagents)
12. [Why Discriminated Unions Over Class Hierarchies](#12-why-discriminated-unions-over-class-hierarchies)
13. [Decision Matrix Summary](#13-decision-matrix-summary)

---

## 1. Why Bun Over Node.js

```mermaid
graph TB
    subgraph Decision["Runtime Decision"]
        Question["Which JavaScript runtime\nfor a CLI tool?"]
    end

    subgraph Options["Options Evaluated"]
        Node["Node.js 18+"]
        Deno["Deno"]
        Bun["Bun"]
    end

    subgraph Winner["✅ Bun Chosen"]
        W1["Native TypeScript execution\n(no tsc build step)"]
        W2["bun:bundle feature flags\n(dead code elimination)"]
        W3["Fast startup (~40ms vs ~120ms)"]
        W4["Built-in .env loading\n(--env-file flag)"]
        W5["Preload scripts\n(bunfig.toml → preload.ts)"]
        W6["ESM-first with\nCommonJS compat"]
    end

    Question --> Options
    Bun --> Winner

    style Winner fill:#27ae60,color:#fff
    style Bun fill:#27ae60,color:#fff
    style Node fill:#95a5a6,color:#fff
    style Deno fill:#95a5a6,color:#fff
```

### The Key Differentiator: `bun:bundle` Feature Flags

This is the single most impactful reason. Claude Code uses `feature()` from `bun:bundle` extensively:

```mermaid
flowchart LR
    subgraph Source["Source Code"]
        Code["if (feature('VOICE_MODE')) {\n  require('./voice.js')\n}"]
    end

    subgraph Build["Build Time"]
        BunBuild["Bun bundler evaluates\nfeature() at build time"]
    end

    subgraph Internal["Internal Build\n(VOICE_MODE=true)"]
        IncludeVoice["Voice module included\nin bundle"]
    end

    subgraph External["External/OSS Build\n(VOICE_MODE=false)"]
        ExcludeVoice["Entire if-block removed\nDead code eliminated"]
    end

    Source --> BunBuild
    BunBuild -->|"flag=true"| Internal
    BunBuild -->|"flag=false"| External

    style Internal fill:#27ae60,color:#fff
    style External fill:#e74c3c,color:#fff
```

The codebase has **17+ feature flags** in `query.ts` alone: `REACTIVE_COMPACT`, `CONTEXT_COLLAPSE`, `HISTORY_SNIP`, `CACHED_MICROCOMPACT`, `TOKEN_BUDGET`, `BG_SESSIONS`, `EXPERIMENTAL_SKILL_SEARCH`, `CHICAGO_MCP`, `TEMPLATES`, and more. No other runtime offers this build-time elimination natively.

### Comparison

| Criterion | Node.js | Deno | **Bun** |
|---|---|---|---|
| TypeScript execution | Requires `tsx`/`ts-node` | Native | **Native** |
| Build-time feature flags | Needs webpack/rollup plugin | No equivalent | **`bun:bundle` feature()** |
| Startup time | ~120ms | ~80ms | **~40ms** |
| Preload scripts | `--require` (CJS only) | No equivalent | **`bunfig.toml` preload** |
| .env file loading | `dotenv` package | `--env-file` (v1.38+) | **`--env-file` native** |
| Package ecosystem compat | 100% | ~95% | **~99%** |

### The Preload Pattern

```mermaid
sequenceDiagram
    participant OS as Operating System
    participant Bun as Bun Runtime
    participant Preload as preload.ts
    participant CLI as cli.tsx

    OS->>Bun: exec bun --env-file=.env cli.tsx
    Bun->>Bun: Read bunfig.toml
    Note over Bun: preload = ["./preload.ts"]

    Bun->>Preload: Execute BEFORE entry point
    Preload->>Preload: Set globalThis.MACRO = {<br/>  VERSION, BUILD_TIME,<br/>  PACKAGE_URL, etc.<br/>}

    Bun->>CLI: Execute entry point
    Note over CLI: MACRO globals available<br/>before any import runs
```

**Insight**: The `preload.ts` pattern sets build-time constants as globals before any application code runs. This avoids circular dependency issues — every module can access `MACRO.VERSION` without importing anything.

---

## 2. Why React + Ink for a CLI

```mermaid
graph TB
    subgraph Challenge["The Challenge"]
        C1["Build a complex TUI with:\n- 140+ components\n- Virtual scrolling\n- Diff rendering\n- Permission dialogs\n- Markdown rendering\n- Real-time streaming\n- Multiple input modes"]
    end

    subgraph Options["Options Evaluated"]
        Blessed["blessed/blessed-contrib\n(traditional TUI)"]
        Raw["Raw ANSI codes\n+ custom framework"]
        ReactInk["React + Ink"]
    end

    subgraph ReactWins["✅ React + Ink Chosen"]
        R1["Declarative UI:\nDescribe WHAT, not HOW"]
        R2["Component composition:\n140+ reusable components"]
        R3["80+ custom hooks:\nuseCanUseTool, useTerminalSize, etc."]
        R4["React Compiler (v19):\nAutomatic memoization"]
        R5["Ecosystem:\nSame patterns web devs know"]
        R6["Testing:\nReact testing patterns work"]
    end

    Challenge --> Options
    ReactInk --> ReactWins

    style ReactWins fill:#27ae60,color:#fff
    style ReactInk fill:#27ae60,color:#fff
```

### The Rendering Stack

```mermaid
flowchart TD
    subgraph React["React Layer"]
        JSX["JSX Components\n(<Box>, <Text>, etc.)"]
        Reconciler["React Reconciler\n(fiber tree diffing)"]
        Compiler["React Compiler\n(auto-memoization)"]
    end

    subgraph Ink["Ink Layer"]
        Yoga["Yoga Layout Engine\n(flexbox → terminal)"]
        InkRenderer["Ink Renderer\n(layout → ANSI)"]
    end

    subgraph Custom["Custom Extensions (src/ink/)"]
        VScroll["Virtual Scrolling\n(only render visible)"]
        Search["Search Highlighting\n(regex → styled regions)"]
        Focus["Terminal Focus/Blur\n(visibility tracking)"]
        TabStatus["Tab Status\n(activity indicators)"]
        Hyperlinks["Hyperlink Support\n(OSC 8 sequences)"]
    end

    subgraph Output["Terminal Output"]
        ANSI["ANSI escape sequences"]
        Stdout["process.stdout.write()"]
    end

    JSX --> Reconciler
    Compiler --> Reconciler
    Reconciler --> Yoga
    Yoga --> InkRenderer
    InkRenderer --> VScroll
    VScroll --> Search
    Search --> ANSI
    ANSI --> Stdout

    Custom --> InkRenderer

    style React fill:#61dafb,color:#333
    style Ink fill:#45b7d1,color:#fff
    style Custom fill:#bb8fce,color:#fff
```

### Why Not blessed?

| Factor | blessed | React + Ink |
|---|---|---|
| **Maintainability** | Imperative widget manipulation | Declarative component model |
| **State management** | Manual DOM-like updates | React state + hooks |
| **Code reuse** | Widget inheritance (fragile) | Component composition (flexible) |
| **Developer pool** | Niche TUI expertise | Any React developer |
| **Performance** | Full redraws on changes | Reconciliation diffs |
| **Maintenance status** | Unmaintained since 2017 | Active ecosystem |

**The decisive insight**: Claude Code's UI complexity (permission dialogs, diff views, markdown rendering, virtual scrolling, vim mode, autocomplete) would be unmanageable with imperative TUI code. React's component model scales to this complexity; blessed doesn't.

---

## 3. Why AsyncGenerator for the Agentic Loop

```mermaid
flowchart TD
    subgraph Pattern["The Agentic Loop Pattern"]
        Loop["while (true):\n  1. Call Claude API (streaming)\n  2. Yield tokens as they arrive\n  3. If tool_use → execute → yield results\n  4. If end_turn → return"]
    end

    subgraph Requirements["Requirements"]
        R1["Stream partial results\n(tokens arrive one at a time)"]
        R2["Backpressure\n(don't overwhelm the UI)"]
        R3["Cancellation\n(user presses Ctrl+C)"]
        R4["Composability\n(nested loops for subagents)"]
        R5["Error propagation\n(tool errors, API errors)"]
    end

    subgraph Alternatives["Alternatives Considered"]
        CB["Callbacks\n❌ No backpressure\n❌ Callback hell"]
        Promises["async/await\n❌ Can't yield partial results\n❌ All-or-nothing"]
        Observables["RxJS Observables\n⚠️ Heavy dependency\n⚠️ Complex operators"]
        AsyncGen["AsyncGenerator\n✅ Native backpressure\n✅ Yield partial results\n✅ try/finally for cleanup\n✅ for-await-of consumption"]
    end

    Pattern --> Requirements
    Requirements --> Alternatives
    AsyncGen -->|"Winner"| Selected["Selected"]

    style AsyncGen fill:#27ae60,color:#fff
    style Selected fill:#27ae60,color:#fff
```

### How the Generator Pattern Works

```mermaid
sequenceDiagram
    participant REPL as REPL (consumer)
    participant Gen as query() generator
    participant API as Claude API
    participant Tool as Tool Executor

    REPL->>Gen: for await (const event of query(...))

    Gen->>API: stream tokens
    loop For each token
        API-->>Gen: token delta
        Gen-->>REPL: yield StreamEvent
        Note over REPL: REPL processes at its own pace<br/>(backpressure automatic)
    end

    API-->>Gen: tool_use block complete
    Gen->>Tool: execute tool
    Tool-->>Gen: tool result

    Gen-->>REPL: yield ToolResultMessage
    Note over Gen: Loop continues automatically

    Gen->>API: stream next response
    API-->>Gen: end_turn
    Gen-->>REPL: return Terminal

    Note over REPL: for-await-of loop ends naturally
```

**Key insight**: The `AsyncGenerator` return type `AsyncGenerator<StreamEvent | Message, Terminal>` encodes the protocol:
- **Yielded values** = intermediate events (tokens, tool results, progress)
- **Return value** = terminal state (the final result)
- **throw()** = cancellation signal (from AbortController)

This is a perfect fit for an agentic loop that produces a stream of events and eventually terminates.

---

## 4. Why a Custom Store Over Zustand/Redux

```mermaid
graph LR
    subgraph Custom["Custom Store (35 lines)"]
        direction TB
        GetState["getState() → T"]
        SetState["setState(updater) → void"]
        Subscribe["subscribe(listener) → cleanup"]
    end

    subgraph React19["React 19 Integration"]
        useSyncExternalStore["useSyncExternalStore(\n  subscribe,\n  getState\n)"]
    end

    subgraph Zustand["Zustand (alternative)"]
        Z1["Similar API but\n+15KB bundle size"]
        Z2["Middleware system\n(unused complexity)"]
        Z3["DevTools integration\n(no browser in CLI)"]
    end

    subgraph Redux["Redux (alternative)"]
        R1["+40KB bundle size"]
        R2["Action creators\n(boilerplate)"]
        R3["Reducers\n(unnecessary indirection)"]
    end

    Custom -->|"Perfect match"| React19

    style Custom fill:#27ae60,color:#fff
    style Zustand fill:#95a5a6,color:#fff
    style Redux fill:#95a5a6,color:#fff
```

**The reasoning in 3 points**:

1. **35 lines vs 15KB+ dependency** — The store needs exactly `getState`, `setState`, `subscribe`. That's it.
2. **React 19's `useSyncExternalStore`** — This hook expects exactly those three methods. A custom store implements the contract perfectly.
3. **DeepImmutable type enforcement** — `AppState` is typed as `DeepImmutable<...>`. The custom store enforces this naturally; Zustand/Redux would need custom middleware.

---

## 5. Why Commander.js for CLI Parsing

```mermaid
graph TB
    subgraph Requirements["CLI Requirements"]
        R1["50+ CLI flags"]
        R2["TypeScript type safety"]
        R3["Subcommands (mcp add, etc.)"]
        R4["Help text generation"]
        R5["Environment variable fallbacks"]
    end

    subgraph Options["Options"]
        Commander["Commander.js\n(@commander-js/extra-typings)"]
        Yargs["yargs"]
        Clipanion["clipanion"]
        CAC["cac"]
    end

    subgraph WhyCommander["✅ Commander.js Chosen"]
        W1["@commander-js/extra-typings\n→ Full type inference\nfor .option() chains"]
        W2["Mature & battle-tested\n(1B+ downloads)"]
        W3["Lightweight\n(~35KB)"]
        W4["Chainable API\nmatches declarative style"]
    end

    Requirements --> Options
    Commander --> WhyCommander

    style WhyCommander fill:#27ae60,color:#fff
```

**Key differentiator**: The `@commander-js/extra-typings` package provides full TypeScript inference for option chains. When you write `.option('--print', 'Print mode')`, TypeScript knows `opts.print` is `boolean | undefined`. No manual type definitions needed for 50+ flags.

---

## 6. Why MCP (Model Context Protocol)

```mermaid
flowchart TB
    subgraph Problem["The Problem"]
        P1["AI needs to interact with:\n- Browsers (Chrome)\n- IDEs (VS Code, JetBrains)\n- Databases\n- APIs\n- Cloud services\n- Custom tools"]
        P2["Each integration needs:\n- Transport protocol\n- Schema definition\n- Auth handling\n- Error handling"]
    end

    subgraph Without["Without MCP: N×M Problem"]
        AI1["Claude Code"] --- Int1["Chrome adapter"]
        AI1 --- Int2["VS Code adapter"]
        AI1 --- Int3["Slack adapter"]
        AI1 --- Int4["Postgres adapter"]
        AI2["Cursor"] --- Int5["Chrome adapter"]
        AI2 --- Int6["VS Code adapter"]
        AI2 --- Int7["Slack adapter"]
        AI2 --- Int8["Postgres adapter"]
    end

    subgraph With["With MCP: N+M Solution"]
        AI3["Claude Code"] --- MCP["MCP Protocol"]
        AI4["Cursor"] --- MCP
        MCP --- S1["Chrome MCP Server"]
        MCP --- S2["IDE MCP Server"]
        MCP --- S3["Slack MCP Server"]
        MCP --- S4["Postgres MCP Server"]
    end

    Problem --> Without
    Problem --> With

    style With fill:#27ae60,color:#fff
    style Without fill:#e74c3c,color:#fff
```

### Claude Code's Dual MCP Role

```mermaid
flowchart LR
    subgraph Client["As MCP Client"]
        CC1["Claude Code connects to\nexternal MCP servers"]
        CC1 --> MCPProxy["MCPTool proxy:\nExposes remote tools\nas local tools"]
    end

    subgraph Server["As MCP Server"]
        CC2["Claude Code serves as\nMCP server for:"]
        CC2 --> Chrome["Chrome extension\n(--claude-in-chrome-mcp)"]
        CC2 --> IDE["IDE extensions\n(WebSocket bridge)"]
    end

    subgraph Why["Why This Matters"]
        W1["Extensibility without code changes"]
        W2["Community can build MCP servers"]
        W3["Same tool interface for\nbuilt-in and external tools"]
        W4["OAuth + transport abstraction\nhandled by MCP SDK"]
    end

    Client --> Why
    Server --> Why

    style Why fill:#27ae60,color:#fff
```

---

## 7. Why Streaming Over Batch API Calls

```mermaid
flowchart LR
    subgraph Batch["Batch Approach"]
        B1["Send request"] --> B2["Wait 5-30 seconds\n(user sees nothing)"] --> B3["Get complete response"]
    end

    subgraph Streaming["Streaming Approach (Chosen)"]
        S1["Send request"] --> S2["See first token\nin ~200ms"] --> S3["Tokens flow\ncontinuously"] --> S4["Response complete"]
    end

    subgraph Benefits["Why Streaming Wins"]
        BN1["Perceived latency:\n200ms vs 5-30s"]
        BN2["StreamingToolExecutor:\nStart tools before response ends"]
        BN3["User can cancel early\nif response is wrong"]
        BN4["Progress indication:\nUser sees the AI 'thinking'"]
    end

    Streaming --> Benefits

    style Batch fill:#e74c3c,color:#fff
    style Streaming fill:#27ae60,color:#fff
    style Benefits fill:#27ae60,color:#fff
```

### StreamingToolExecutor: The Overlap Optimization

```mermaid
gantt
    title Response Processing Timeline
    dateFormat X
    axisFormat %L ms

    section Batch Approach
    Wait for full response    :a1, 0, 5000
    Parse tool calls          :a2, 5000, 5050
    Execute Tool 1            :a3, 5050, 5550
    Execute Tool 2            :a4, 5550, 6050

    section Streaming Approach
    Stream tokens             :b1, 0, 5000
    Parse tool 1 (partial)    :b2, 2000, 2050
    Execute Tool 1 (early)    :b3, 2050, 2550
    Parse tool 2 (partial)    :b4, 3500, 3550
    Execute Tool 2 (early)    :b5, 3550, 4050
    Remaining tokens          :b6, 4050, 5000
```

**Streaming saves ~2-3 seconds** per tool-heavy response by overlapping API streaming with tool I/O.

---

## 8. Why Read-Parallel, Write-Serial Tool Execution

```mermaid
flowchart TD
    subgraph Model["Model sends 5 tool calls"]
        T1["Glob(**/*.ts)"]
        T2["Read(src/foo.ts)"]
        T3["Read(src/bar.ts)"]
        T4["Edit(src/foo.ts)"]
        T5["Bash(npm test)"]
    end

    subgraph Partition["partitionToolCalls()"]
        T1 --> ReadBatch
        T2 --> ReadBatch
        T3 --> ReadBatch
        T4 --> WriteBatch
        T5 --> WriteBatch

        ReadBatch["Read-only batch:\nisReadOnly() === true"]
        WriteBatch["Write batch:\nisReadOnly() === false"]
    end

    subgraph Execution["Execution Strategy"]
        ReadBatch --> Par["PARALLEL\n(Promise.allSettled, max 10)\n\nGlob + Read + Read\nrun simultaneously"]
        WriteBatch --> Ser["SERIAL\n(sequential for-loop)\n\nEdit → then Bash"]
        Par -->|"All complete"| Ser
    end

    subgraph Why["Why This Specific Strategy"]
        Safe["Read-only tools are\nside-effect free:\nparallel is safe"]
        Unsafe["Write tools may conflict:\n- Edit shifts line numbers\n- Bash depends on FS state\n- Write may conflict with Edit"]
        Perf["10-read parallel saves\n~5-10 seconds for\ncodebase exploration"]
    end

    style Par fill:#27ae60,color:#fff
    style Ser fill:#e74c3c,color:#fff
```

**The configurable limit** (`CLAUDE_CODE_MAX_TOOL_USE_CONCURRENCY=10`) prevents file descriptor exhaustion while maximizing throughput. In a typical "explore the codebase" turn, the model might issue 10 Glob/Read/Grep calls simultaneously — finishing in the time of the slowest one rather than the sum of all.

---

## 9. Why Multiple Compaction Strategies

```mermaid
flowchart TD
    subgraph Problem["The Problem"]
        P["Conversations can run\nfor hours with 1000s\nof tool calls.\n\nContext window is finite\n(200K-1M tokens)."]
    end

    subgraph Strategy["Layered Defense Strategy"]
        direction TB
        L1["Layer 1: Tool Result Budget\nCOST: Minimal (truncation)\nSAVINGS: ~30% of tool output\nWHEN: Every iteration"]

        L2["Layer 2: Snip Compaction\nCOST: None (just removes)\nSAVINGS: ~20% of old messages\nWHEN: Before API call"]

        L3["Layer 3: Microcompact\nCOST: Low (cached summaries)\nSAVINGS: ~15% of verbose results\nWHEN: Before API call"]

        L4["Layer 4: Context Collapse\nCOST: Medium (grouping)\nSAVINGS: ~25% via grouping\nWHEN: Before auto-compact"]

        L5["Layer 5: Auto-compact\nCOST: High (separate API call)\nSAVINGS: ~70% full reset\nWHEN: Approaching limit"]

        L6["Layer 6: Reactive Compact\nCOST: High (emergency)\nSAVINGS: ~80% aggressive\nWHEN: prompt_too_long error"]

        L1 --> L2 --> L3 --> L4 --> L5 --> L6
    end

    Problem --> Strategy

    style L1 fill:#27ae60,color:#fff
    style L2 fill:#2ecc71,color:#fff
    style L3 fill:#f1c40f,color:#333
    style L4 fill:#e67e22,color:#fff
    style L5 fill:#e74c3c,color:#fff
    style L6 fill:#c0392b,color:#fff
```

### Why Not Just Auto-Compact?

```mermaid
graph LR
    subgraph NaiveApproach["Naive: Only Auto-Compact"]
        N1["Run until context full"] --> N2["Expensive API call\nto summarize"] --> N3["Lose all detail\nfrom conversation"]
    end

    subgraph LayeredApproach["Layered: Progressive Compaction"]
        L1["Cheap truncation\nkeeps most detail"] --> L2["Remove old messages\n(still cheap)"] --> L3["Summarize verbose parts\n(cached)"] --> L4["Full compact only\nwhen truly needed"]
    end

    style NaiveApproach fill:#e74c3c,color:#fff
    style LayeredApproach fill:#27ae60,color:#fff
```

**The key insight**: Each layer preserves more conversational context than the next. By applying cheap strategies first, the expensive auto-compact triggers less frequently. A conversation that would need 10 auto-compacts with naive approach might need only 2-3 with layered compaction — saving both money (API calls) and context quality (more detail preserved).

---

## 10. Why Feature Flags for Dead Code Elimination

```mermaid
flowchart TD
    subgraph Codebase["Single Codebase, Multiple Builds"]
        Source["Shared TypeScript source\nwith feature() gates"]
    end

    subgraph Internal["Internal Build"]
        Source --> IB["All features enabled:\n- VOICE_MODE\n- DAEMON\n- COORDINATOR_MODE\n- KAIROS\n- PROACTIVE\n- AGENT_TRIGGERS\n- CHICAGO_MCP"]
        IB --> IBSize["Larger bundle\nFull capabilities"]
    end

    subgraph OSS["Open Source Build"]
        Source --> OB["Features disabled:\n- Dead code eliminated\n- No unused imports\n- Smaller bundle"]
        OB --> OBSize["Smaller bundle\nCore capabilities only"]
    end

    style Internal fill:#45b7d1,color:#fff
    style OSS fill:#27ae60,color:#fff
```

### Why Not Runtime Feature Flags?

| Approach | Bundle Size | Runtime Cost | Security |
|---|---|---|---|
| **Runtime flags** (process.env) | All code ships | Branch on every check | Internal code visible in bundle |
| **Build-time flags** (bun:bundle) | Dead code removed | Zero (code doesn't exist) | Internal code absent from bundle |

With 17+ feature flags in `query.ts` alone, runtime checking would add measurable overhead to the hot loop. Build-time elimination means the open-source build literally doesn't contain the code for internal features.

---

## 11. Why Fork Context for Subagents

```mermaid
flowchart TD
    subgraph Parent["Parent Agent Context"]
        PM["messages: Message[]"]
        PT["tools: Tools (full set)"]
        PS["appState: AppState"]
        PA["abortController"]
    end

    subgraph Fork["Fork Operation"]
        Parent --> ForkOp["Create forked ToolUseContext:\n1. New messages[] (empty or seeded)\n2. Filtered tools (per agent type)\n3. Shared appState reference\n4. New AbortController (child of parent)"]
    end

    subgraph Child["Child Agent Context"]
        CM["messages: [] (own history)"]
        CT["tools: filtered subset"]
        CS["appState: same reference"]
        CA["abortController: linked to parent"]
    end

    ForkOp --> Child

    subgraph Why["Why Fork?"]
        I1["Isolation: Agent's tool calls\ndon't pollute parent history"]
        I2["Scoping: Explore agent gets\nonly read-only tools"]
        I3["Cancellation: Aborting parent\ncancels all children"]
        I4["State sharing: Both see\nsame files, same settings"]
    end

    Child --> Why

    style Fork fill:#f39c12,color:#fff
    style Why fill:#27ae60,color:#fff
```

### The Trade-off: Shared State vs Full Isolation

```mermaid
graph LR
    subgraph Shared["What's SHARED"]
        S1["AppState (settings, tools)"]
        S2["File system access"]
        S3["MCP connections"]
        S4["Permission rules"]
    end

    subgraph Isolated["What's ISOLATED"]
        I1["Message history"]
        I2["Tool call context"]
        I3["Abort controller"]
        I4["Token budget"]
    end

    subgraph Rationale["Rationale"]
        R1["Shared = expensive to duplicate,\nrequires consistency"]
        R2["Isolated = must be independent\nfor correct parallel execution"]
    end

    Shared --> R1
    Isolated --> R2

    style Shared fill:#45b7d1,color:#fff
    style Isolated fill:#bb8fce,color:#fff
```

---

## 12. Why Discriminated Unions Over Class Hierarchies

```mermaid
graph TB
    subgraph ClassApproach["Class Hierarchy (Rejected)"]
        BaseMsg["abstract class Message"]
        UserMsg["class UserMessage\nextends Message"]
        AsstMsg["class AssistantMessage\nextends Message"]
        BaseMsg --> UserMsg
        BaseMsg --> AsstMsg
        Problem1["❌ Not serializable to JSON"]
        Problem2["❌ instanceof breaks across modules"]
        Problem3["❌ Mutable by default"]
    end

    subgraph UnionApproach["Discriminated Union (Chosen)"]
        Union["type Message =\n| UserMessage\n| AssistantMessage\n| AttachmentMessage\n| ..."]
        Benefit1["✅ JSON serializable (plain objects)"]
        Benefit2["✅ Type narrowing via .type field"]
        Benefit3["✅ DeepImmutable wrapping natural"]
        Benefit4["✅ Exhaustive switch checking"]
    end

    style ClassApproach fill:#e74c3c,color:#fff
    style UnionApproach fill:#27ae60,color:#fff
```

**Why this matters for Claude Code specifically**: Messages must be:
1. **Serialized** to JSON for session persistence (`/resume` command)
2. **Sent** to the compaction service for summarization
3. **Shared** between parent and child agent processes
4. **Wrapped** in `DeepImmutable<>` for state management

Plain objects with discriminated unions satisfy all four requirements naturally. Class instances would require custom serialization, deep cloning, and `Object.freeze` hacks.

---

## 13. Decision Matrix Summary

```mermaid
mindmap
    root((Claude Code\nDesign Decisions))
        Runtime
            Bun over Node.js
                Native TypeScript
                Feature flags
                Fast startup
        UI Framework
            React + Ink
                Component model
                Hooks ecosystem
                React Compiler
        Core Loop
            AsyncGenerator
                Backpressure
                Streaming yields
                Composable
        State Management
            Custom 35-line store
                Zero dependencies
                useSyncExternalStore
                DeepImmutable
        API Strategy
            Streaming SSE
                Low perceived latency
                StreamingToolExecutor
                Early cancellation
        Tool Execution
            Read-parallel Write-serial
                Correctness guarantee
                Max throughput
                Configurable limit
        Context Management
            Layered compaction
                Progressive cost
                Maximum detail retention
                Emergency fallback
        Extensibility
            MCP Protocol
                N+M integration
                Community ecosystem
                Dual client/server
        Code Organization
            Discriminated unions
                Serializable
                Type-safe
                Immutable-friendly
            Feature flags
                Build-time elimination
                Multi-variant builds
                Zero runtime cost
```

### The Overarching Philosophy

Every decision in Claude Code follows three principles:

1. **Minimize to the minimum** — Don't use a library when 35 lines suffice (store). Don't ship code that won't run (feature flags). Don't re-render what hasn't changed (virtual scrolling).

2. **Correctness over convenience** — Write tools run serially even though parallel would be faster (but wrong). Messages are immutable even though mutation would be simpler. Tool results go through hooks even though direct execution would be quicker.

3. **Stream everything** — Tokens stream from the API. Tool results stream to the UI. Agents yield results as they work. The generator pattern permeates the entire system because the alternative (waiting for complete results) wastes the user's time.
