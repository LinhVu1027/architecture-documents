# Performance & Optimization Strategies

> How Claude Code achieves fast startup, responsive streaming, and efficient resource usage through layered optimization techniques. Every diagram is a Mermaid diagram you can render in any Markdown viewer.

---

## Table of Contents

1. [Performance Architecture Overview](#1-performance-architecture-overview)
2. [Startup Optimization](#2-startup-optimization)
3. [API Prompt Caching](#3-api-prompt-caching)
4. [Streaming & StreamingToolExecutor](#4-streaming--streamingtoolexecutor)
5. [Concurrent Tool Execution](#5-concurrent-tool-execution)
6. [Caching Strategies](#6-caching-strategies)
7. [Lazy Loading & Dead Code Elimination](#7-lazy-loading--dead-code-elimination)
8. [Terminal Rendering Performance](#8-terminal-rendering-performance)
9. [Token Estimation & Budget Management](#9-token-estimation--budget-management)
10. [Speculation & Prefetching](#10-speculation--prefetching)

---

## 1. Performance Architecture Overview

Claude Code optimizes at every layer of the stack — from build-time code elimination to runtime streaming overlap.

```mermaid
flowchart TD
    subgraph BuildTime["Build-Time Optimizations"]
        DCE["Dead Code Elimination\n(bun:bundle feature flags)"]
        Preload["Preload Scripts\n(MACRO globals set early)"]
    end

    subgraph StartupTime["Startup Optimizations"]
        ParPrefetch["Parallel Prefetching\n(credentials, settings, features)"]
        LazyImport["Lazy Module Loading\n(feature-gated dynamic imports)"]
        FastPath["Fast Paths\n(--version exits before full init)"]
    end

    subgraph RequestTime["Per-Request Optimizations"]
        PromptCache["Prompt Caching\n(static prefix cached at API)"]
        Streaming["Token Streaming\n(200ms first token)"]
        StreamExec["StreamingToolExecutor\n(start tools before stream ends)"]
    end

    subgraph LoopTime["Per-Loop-Iteration Optimizations"]
        ToolPar["Read-Parallel Tool Execution\n(up to 10 concurrent)"]
        MicroComp["Microcompact Cache\n(reuse prior summaries)"]
        TokenEst["Token Estimation\n(avoid re-counting)"]
    end

    subgraph RenderTime["Rendering Optimizations"]
        VScroll["Virtual Scrolling\n(render only visible rows)"]
        ReactComp["React Compiler\n(automatic memoization)"]
        DeferValue["useDeferredValue\n(non-blocking updates)"]
    end

    BuildTime --> StartupTime --> RequestTime --> LoopTime --> RenderTime

    style BuildTime fill:#c0392b,color:#fff
    style StartupTime fill:#e74c3c,color:#fff
    style RequestTime fill:#f39c12,color:#fff
    style LoopTime fill:#f1c40f,color:#333
    style RenderTime fill:#27ae60,color:#fff
```

---

## 2. Startup Optimization

```mermaid
gantt
    title Claude Code Boot Timeline
    dateFormat X
    axisFormat %L ms

    section Critical Path (Blocking)
    Parse CLI args           :crit, a1, 0, 5
    Check fast paths         :crit, a2, 5, 8
    Import main.tsx          :crit, a3, 8, 50
    Commander.js setup       :crit, a4, 50, 70
    setup() call             :crit, a5, 70, 120
    React render REPL        :crit, a6, 120, 150

    section Parallel (Non-Blocking)
    MDM settings read        :b1, 8, 35
    Keychain prefetch        :b2, 8, 60
    GrowthBook init          :b3, 50, 90
    MCP server connections   :b4, 120, 250
    Plugin loading           :b5, 120, 180
    Background housekeeping  :b6, 150, 200
```

### Fast Paths

```mermaid
flowchart TD
    Entry["cli.tsx entry point"] --> Check{"Fast path\ncheck"}

    Check -->|"--version"| Version["Print version\nExit (0ms init)"]
    Check -->|"--dump-system-prompt"| Dump["Print prompt\nExit (minimal init)"]
    Check -->|"--daemon-worker"| Daemon["Start worker\n(skip UI init)"]
    Check -->|"Normal"| FullInit["Full initialization\n(main.tsx import)"]

    style Version fill:#27ae60,color:#fff
    style Dump fill:#27ae60,color:#fff
    style Daemon fill:#f39c12,color:#fff
    style FullInit fill:#45b7d1,color:#fff
```

### Design Insight: Why Parallel Prefetching?

```mermaid
sequenceDiagram
    participant Main as main.tsx
    participant MDM as MDM Reader
    participant KC as Keychain
    participant GB as GrowthBook

    Note over Main: All three start BEFORE<br/>Commander.js setup

    par Parallel startup
        Main->>MDM: startMdmRawRead()
        Main->>KC: startKeychainPrefetch()
        Main->>GB: initializeGrowthBook()
    end

    Note over Main: Commander.js setup runs<br/>while prefetches are in-flight

    Main->>Main: Commander.js program definition
    Main->>Main: setup()

    Note over MDM,GB: By now, prefetches have<br/>finished (or close to it)

    Main->>MDM: await mdmResult
    Main->>KC: await keychainResult
```

The parallel prefetch pattern overlaps I/O-bound operations (keychain access, HTTP for GrowthBook, file reads for MDM) with CPU-bound operations (module parsing, Commander setup). Total startup time ≈ max(prefetch, init) instead of sum(prefetch + init).

---

## 3. API Prompt Caching

```mermaid
flowchart TD
    subgraph SystemPrompt["System Prompt Structure"]
        Static["STATIC PREFIX (~8K tokens)\n- Tool descriptions\n- Identity & capabilities\n- Usage instructions\n- Tone & style rules"]
        Dynamic["DYNAMIC SUFFIX (~2K tokens)\n- Current date\n- Git status\n- Active tasks\n- System reminders"]
    end

    subgraph CacheControl["cache_control Markers"]
        Static --> CacheMarker["cache_control:\n  type: 'ephemeral'\n(Tells API to cache this block)"]
        Dynamic --> NoCacheMarker["No cache_control\n(Recomputed each request)"]
    end

    subgraph APISide["API-Side Caching"]
        FirstReq["Request 1:\nProcess full 10K tokens\nCache static prefix"]
        SubseqReq["Requests 2-N:\nReuse cached prefix (~0ms)\nOnly process dynamic 2K"]
    end

    CacheControl --> APISide

    subgraph Savings["Cost & Latency Savings"]
        TokenSave["~80% fewer input tokens processed\nper request after first"]
        LatencySave["~200ms faster per request\n(skip processing cached tokens)"]
        CostSave["Cached tokens billed at\nreduced rate"]
    end

    APISide --> Savings

    style Static fill:#27ae60,color:#fff
    style Dynamic fill:#f39c12,color:#fff
    style Savings fill:#27ae60,color:#fff
```

### splitSysPromptPrefix() — The Cache Boundary

```mermaid
flowchart LR
    Full["Full System Prompt\n(10K tokens)"] --> Split["splitSysPromptPrefix()"]

    Split --> Prefix["Cached Prefix\n(8K tokens)\n\n✅ Tool schemas\n✅ Instructions\n✅ Rules"]
    Split --> Suffix["Uncached Suffix\n(2K tokens)\n\n🔄 Date\n🔄 Git status\n🔄 Tasks"]

    Prefix --> API1["cache_control:\n{type: 'ephemeral'}"]
    Suffix --> API2["No cache control"]

    style Prefix fill:#27ae60,color:#fff
    style Suffix fill:#f39c12,color:#fff
```

### Design Insight: Why "ephemeral" Cache Type?

Anthropic's prompt caching offers `ephemeral` caching that persists for ~5 minutes. This is perfect for Claude Code because:
- Within a session, requests happen every few seconds (well within 5 min window)
- Across sessions, the system prompt changes (different tools, different project context)
- No stale cache risk — cache naturally expires when the user starts a new session

---

## 4. Streaming & StreamingToolExecutor

```mermaid
flowchart TD
    subgraph Traditional["Traditional: Wait-Then-Execute"]
        T1["Stream all tokens\n(5 seconds)"]
        T2["Parse complete response"]
        T3["Extract tool_use blocks"]
        T4["Execute Tool 1 (500ms)"]
        T5["Execute Tool 2 (500ms)"]
        T1 --> T2 --> T3 --> T4 --> T5
        TTotal["Total: 6.5 seconds"]
    end

    subgraph Streaming["StreamingToolExecutor: Overlap"]
        S1["Stream tokens..."]
        S2["Partial tool JSON detected\nat 2s mark"]
        S3["Start Tool 1 early\n(file path known)"]
        S4["More tokens stream...\nTool 1 completes"]
        S5["Tool 2 JSON detected\nat 3.5s mark"]
        S6["Start Tool 2 early"]
        S7["Stream ends at 5s\nTools already done"]
        S1 --> S2 --> S3
        S2 --> S4
        S4 --> S5 --> S6
        S6 --> S7
        STotal["Total: 5.0 seconds\n(1.5s saved)"]
    end

    style Traditional fill:#e74c3c,color:#fff
    style Streaming fill:#27ae60,color:#fff
    style TTotal fill:#e74c3c,color:#fff
    style STotal fill:#27ae60,color:#fff
```

### How Partial JSON Parsing Works

```mermaid
sequenceDiagram
    participant API as Claude API Stream
    participant STE as StreamingToolExecutor
    participant Tool as Tool Registry

    API->>STE: input_json_delta: '{"file_'
    STE->>STE: Buffer: '{"file_'

    API->>STE: input_json_delta: 'path": "src/fo'
    STE->>STE: Buffer: '{"file_path": "src/fo'

    API->>STE: input_json_delta: 'o.ts"'
    STE->>STE: Buffer: '{"file_path": "src/foo.ts"'
    Note over STE: file_path complete!<br/>Can start Read tool early

    STE->>Tool: Start Read("src/foo.ts")
    Note over Tool: File I/O begins<br/>while API still streaming

    API->>STE: input_json_delta: ', "limit": 50}'
    STE->>STE: Full JSON complete
    Tool-->>STE: File content ready
    Note over STE: Tool result was ready<br/>before stream finished!
```

### Design Insight: Why This Matters

For a typical "explore the codebase" interaction where the model reads 5-8 files:
- **Without streaming exec**: 5s stream + 4s tool execution = **9 seconds**
- **With streaming exec**: 5s stream (overlapped with tool I/O) + 0.5s remaining = **5.5 seconds**
- **Savings**: ~40% reduction in perceived latency

The key insight is that most tool inputs (especially file paths for Read/Glob/Grep) appear at the **beginning** of the JSON object, so parsing can start before the full input is available.

---

## 5. Concurrent Tool Execution

```mermaid
flowchart TD
    subgraph Before["Without Concurrency"]
        S1["Glob(**/*.ts) → 200ms"]
        S2["Read(a.ts) → 50ms"]
        S3["Read(b.ts) → 50ms"]
        S4["Read(c.ts) → 50ms"]
        S5["Grep(pattern) → 150ms"]
        S1 --> S2 --> S3 --> S4 --> S5
        STotal["Total: 500ms (serial)"]
    end

    subgraph After["With Read-Parallel (Chosen)"]
        P1["Glob + Read(a) + Read(b)\n+ Read(c) + Grep\nAll concurrent"]
        PTotal["Total: 200ms\n(bottleneck = slowest tool)"]
    end

    style Before fill:#e74c3c,color:#fff
    style After fill:#27ae60,color:#fff
    style STotal fill:#e74c3c,color:#fff
    style PTotal fill:#27ae60,color:#fff
```

### Concurrency Limit: Why 10?

```mermaid
graph LR
    subgraph Tradeoff["Concurrency vs Resources"]
        Low["1-3 concurrent:\nUnderutilized I/O\nSlow exploration"]
        Medium["5-10 concurrent:\nOptimal throughput\nReasonable FD usage"]
        High["20-50 concurrent:\nFile descriptor exhaustion\nOS-level throttling\nDiminishing returns"]
    end

    Medium --> Selected["Selected: 10\n(configurable via\nCLAUDE_CODE_MAX_TOOL_USE_CONCURRENCY)"]

    style Medium fill:#27ae60,color:#fff
    style Selected fill:#27ae60,color:#fff
```

---

## 6. Caching Strategies

```mermaid
flowchart TD
    subgraph Caches["Cache Layers"]
        FileCache["File State Cache\n(createFileStateCacheWithSizeLimit)\nLRU cache of file contents\nPrevents re-reading unchanged files"]
        MicroCache["Microcompact Cache\n(CACHED_MICROCOMPACT feature)\nCached summaries of tool results\nReused across compaction cycles"]
        PromptCacheSvc["Prompt Cache (API-side)\n(cache_control: ephemeral)\nStatic system prompt prefix\nReused across API calls"]
        SkillCache["Skill Discovery Cache\n(prefetch results)\nAvailable skills list\nReduced re-discovery"]
        TokenCache["Token Count Cache\n(from last API response usage)\nAvoid re-counting tokens\nUse API-reported counts"]
    end

    subgraph Strategy["Cache Strategy Matrix"]
        direction TB
        S1["File Cache:\nLRU eviction\nSize-limited\nInvalidated on write"]
        S2["Micro Cache:\nContent-addressed\nPersists across compactions\nFeature-flag gated"]
        S3["Prompt Cache:\nTime-based (5min TTL)\nAPI-managed\nAutomatic"]
        S4["Token Cache:\nPer-response\nBest-effort estimate\nNo explicit invalidation"]
    end

    Caches --> Strategy

    style FileCache fill:#4ecdc4,color:#fff
    style MicroCache fill:#45b7d1,color:#fff
    style PromptCacheSvc fill:#bb8fce,color:#fff
    style SkillCache fill:#f39c12,color:#fff
    style TokenCache fill:#f1c40f,color:#333
```

### Design Insight: Why LRU for File State?

```mermaid
graph TB
    subgraph Problem["The Problem"]
        P["Model reads 100+ files\nin a long session.\nCaching ALL file contents\nwould use GBs of memory."]
    end

    subgraph Solution["LRU Cache with Size Limit"]
        LRU["READ_FILE_STATE_CACHE_SIZE\nlimits cache entries"]
        Recent["Recently-read files\nstay cached (hot)"]
        Old["Oldest-accessed files\nevicted (cold)"]
        Merge["mergeFileStateCaches()\ncombines parent + child caches\nwhen agents return"]
    end

    Problem --> Solution

    style Solution fill:#27ae60,color:#fff
```

The LRU strategy matches real usage: the model tends to read the same files repeatedly (editing a file requires reading it first), so recently-read files are the most valuable to cache.

---

## 7. Lazy Loading & Dead Code Elimination

```mermaid
flowchart TD
    subgraph FeatureFlags["Build-Time Feature Flags"]
        F1["feature('VOICE_MODE')"]
        F2["feature('REACTIVE_COMPACT')"]
        F3["feature('CONTEXT_COLLAPSE')"]
        F4["feature('HISTORY_SNIP')"]
        F5["feature('CACHED_MICROCOMPACT')"]
        F6["feature('BG_SESSIONS')"]
        F7["feature('COORDINATOR_MODE')"]
        F8["feature('CHICAGO_MCP')"]
    end

    subgraph Pattern["Conditional Require Pattern"]
        Code["const module = feature('X')\n  ? require('./module.js')\n  : null"]
    end

    subgraph Build["Build Output"]
        Enabled["Feature ON:\nModule code included\nrequire() executes"]
        Disabled["Feature OFF:\nEntire if-block removed\nModule never loaded\nBundle smaller"]
    end

    FeatureFlags --> Pattern
    Pattern --> Build

    style Disabled fill:#27ae60,color:#fff
    style Enabled fill:#45b7d1,color:#fff
```

### Dynamic Import Pattern

```mermaid
flowchart LR
    subgraph Eager["Eager Import (Avoided)"]
        E1["import { heavy } from './heavy.js'\n// Loaded at startup\n// Even if never used"]
    end

    subgraph Lazy["Feature-Gated Require (Used)"]
        L1["const heavy = feature('HEAVY')\n  ? require('./heavy.js')\n  : null\n\n// Only loaded if feature ON\n// Eliminated at build time if OFF"]
    end

    style Eager fill:#e74c3c,color:#fff
    style Lazy fill:#27ae60,color:#fff
```

### Design Insight: 17+ Feature Gates in query.ts Alone

The query loop (`query.ts`) is the hot path — it runs on every single user interaction. Having 17+ feature gates means the open-source build has significantly less code to parse and execute in this critical path. Each gate eliminates both the import and all usage sites, compounding the savings.

---

## 8. Terminal Rendering Performance

```mermaid
flowchart TD
    subgraph Challenge["The Challenge"]
        C1["Long conversations with:\n- 1000+ messages\n- Code blocks with syntax highlighting\n- Diffs with color coding\n- Markdown formatting\n\nRe-rendering everything on\nevery state change = unusable"]
    end

    subgraph Solutions["Optimization Stack"]
        VS["Virtual Scrolling\n(src/ink/ custom extension)\nOnly render visible ~50 lines\nO(1) regardless of history"]
        RC["React Compiler\n(react/compiler-runtime)\nAutomatic React.memo\nAutomatic useMemo/useCallback"]
        DV["useDeferredValue\nLow-priority state updates\ndon't block user input"]
        Yoga["Yoga Layout Engine\nEfficient flexbox calculation\nIncremental relayout"]
    end

    Challenge --> Solutions

    style Challenge fill:#e74c3c,color:#fff
    style Solutions fill:#27ae60,color:#fff
```

### Virtual Scrolling Deep Dive

```mermaid
flowchart LR
    subgraph Without["Without Virtual Scrolling"]
        W1["Message 1"] --> W2["Message 2"] --> W3["..."] --> W4["Message 999"] --> W5["Message 1000"]
        WNote["Render ALL 1000 messages\n= 5000+ terminal lines\n= ~200ms per frame"]
    end

    subgraph With["With Virtual Scrolling"]
        V1["Viewport: rows 450-500"]
        V2["Only render Message 45-50\n= ~50 terminal lines\n= ~5ms per frame"]
    end

    style Without fill:#e74c3c,color:#fff
    style With fill:#27ae60,color:#fff
```

### Design Insight: React Compiler for CLI

The `react/compiler-runtime` import in REPL.tsx enables React 19's automatic optimization:

| Without Compiler | With Compiler |
|---|---|
| Manual `React.memo()` wrappers | Automatic memoization |
| Manual `useMemo()` for expensive computations | Compiler detects stable values |
| Manual `useCallback()` for handler refs | Compiler hoists stable callbacks |
| Easy to miss optimizations | Comprehensive coverage |

In a CLI app with 140+ components, manually memoizing everything would be impractical. The compiler does it automatically, preventing unnecessary terminal redraws.

---

## 9. Token Estimation & Budget Management

```mermaid
flowchart TD
    subgraph Counting["Token Counting Strategies"]
        Exact["Exact Count:\nAPI returns usage.input_tokens\nin every response"]
        Estimate["Estimation:\ntokenCountWithEstimation()\nUse last-known usage\n+ estimate delta"]
        Budget["Budget Check:\ncheckTokenBudget()\nTrack cumulative output tokens"]
    end

    subgraph Usage["Where Token Counts Are Used"]
        AutoCompact["Auto-compact trigger:\nIf estimated > threshold\n→ compact conversation"]
        ToolBudget["Tool result budget:\nTruncate outputs >\nMAX_TOOL_RESULT_TOKENS"]
        CostTracker["Cost tracker:\nDisplay $ cost in status line"]
        BudgetLimit["Token budget:\nStop agent if exceeding\nconfigured budget"]
    end

    Counting --> Usage

    style Exact fill:#27ae60,color:#fff
    style Estimate fill:#f39c12,color:#fff
```

### Design Insight: Why Estimate Instead of Count?

Counting tokens precisely requires a tokenizer (which is ~1MB of data and ~10ms per call). Instead, Claude Code:

1. Uses the **exact count from the last API response** (`usage.input_tokens`)
2. Estimates the delta since then (new messages added since last API call)
3. Uses `finalContextTokensFromLastResponse()` as the authoritative baseline

This is accurate enough for compaction triggers (±5% error is fine when the threshold is 80% of context window) while avoiding tokenizer dependency.

---

## 10. Speculation & Prefetching

```mermaid
flowchart TD
    subgraph Prefetch["Prefetching Patterns"]
        MemPrefetch["Memory Prefetch\nstartRelevantMemoryPrefetch()\nLoad CLAUDE.md files before\nmodel needs them"]
        SkillPrefetch["Skill Discovery Prefetch\n(EXPERIMENTAL_SKILL_SEARCH)\nDiscover available skills\nbefore model asks"]
        KeychainPrefetch["Keychain Prefetch\nstartKeychainPrefetch()\nLoad credentials while\nmodules are still importing"]
        MDMPrefetch["MDM Prefetch\nstartMdmRawRead()\nRead enterprise policies\nat startup"]
    end

    subgraph Speculation["Speculation System"]
        SpecState["SpeculationState in AppState"]
        SpecStrategy["Speculative execution:\nPredict likely next tool call\nand start it early"]
    end

    subgraph Timing["When Prefetches Run"]
        Boot["At boot:\nKeychain + MDM + GrowthBook"]
        LoopStart["At loop iteration start:\nMemory + Skills"]
        ToolComplete["After tool completes:\nSpeculation for next tool"]
    end

    Prefetch --> Timing
    Speculation --> Timing

    style Prefetch fill:#45b7d1,color:#fff
    style Speculation fill:#bb8fce,color:#fff
```

### Design Insight: Why Prefetch Memory at Loop Start?

CLAUDE.md files and memory attachments are needed for every API call, but they may change between iterations (if a tool modifies them). By prefetching at the **start of each loop iteration** (not just at boot), Claude Code:

1. Gets the **freshest** CLAUDE.md content
2. Overlaps the file I/O with context management (compaction, etc.)
3. Has results ready by the time the API call is constructed

---

## Performance Impact Summary

```mermaid
graph TB
    subgraph Savings["Measured Impact Areas"]
        S1["Startup: ~40ms\n(Bun + parallel prefetch\nvs ~120ms on Node.js)"]
        S2["First token: ~200ms\n(streaming vs 5-30s batch)"]
        S3["Tool execution: ~60% faster\n(10x parallel reads +\nstreaming overlap)"]
        S4["Rendering: O(1)\n(virtual scrolling vs O(n))"]
        S5["API cost: ~80% less\ninput token processing\n(prompt caching)"]
        S6["Bundle size: ~30-50% smaller\n(dead code elimination\nvia feature flags)"]
    end

    style Savings fill:#27ae60,color:#fff
```

### The Overarching Performance Philosophy

Every optimization in Claude Code follows one principle: **overlap everything possible**.

- **Startup**: Overlap credential fetching with module loading
- **API calls**: Overlap static prompt caching with dynamic computation
- **Streaming**: Overlap token reception with tool execution
- **Tool execution**: Overlap multiple read operations via concurrency
- **Rendering**: Overlap state updates with deferred rendering

The result is a system that feels responsive even when doing complex multi-tool agentic work.
