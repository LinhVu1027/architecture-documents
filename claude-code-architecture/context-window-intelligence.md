# Context Window Intelligence & Compaction Strategies

> How Claude Code manages the finite context window as an intelligent resource — compacting, snipping, collapsing, budgeting, and persisting to keep the model effective across arbitrarily long sessions. Every diagram is a Mermaid diagram you can render in any Markdown viewer.

---

## Table of Contents

1. [Why Context Management Is the Hardest Problem](#1-why-context-management-is-the-hardest-problem)
2. [The Context Management Pipeline](#2-the-context-management-pipeline)
3. [Token Counting & Estimation](#3-token-counting--estimation)
4. [Auto-Compact: The Primary Safety Net](#4-auto-compact-the-primary-safety-net)
5. [Reactive Compact: Emergency Recovery](#5-reactive-compact-emergency-recovery)
6. [History Snip: Surgical Message Removal](#6-history-snip-surgical-message-removal)
7. [Context Collapse: Read/Search Folding](#7-context-collapse-readsearch-folding)
8. [Tool Result Budget & Persistence](#8-tool-result-budget--persistence)
9. [Microcompact & Caching](#9-microcompact--caching)
10. [Session Memory: Surviving Compaction](#10-session-memory-surviving-compaction)
11. [Token Budget: Spending Control](#11-token-budget-spending-control)
12. [The Complete Decision Flow](#12-the-complete-decision-flow)

---

## 1. Why Context Management Is the Hardest Problem

An AI coding agent that runs for hours, reads hundreds of files, and executes dozens of tools will inevitably exceed any context window. The challenge: **how do you compress without losing the information the model needs right now?**

```mermaid
graph TB
    subgraph Problem["The Fundamental Tension"]
        MoreContext["More Context = Better Decisions<br/>Model sees full history, all files,<br/>all tool results"]
        LessContext["Less Context = Faster + Cheaper<br/>Smaller requests, less cost,<br/>focused attention"]
        Limit["Hard Limit: 200K or 1M tokens<br/>Cannot be exceeded"]
    end

    subgraph Solution["Claude Code's Solution"]
        Layered["Layered Compaction<br/>5+ strategies at different thresholds"]
        Intelligent["Intelligent Summarization<br/>AI-powered conversation summaries"]
        Persistence["Disk Persistence<br/>Large results saved to files"]
        Budgeting["Active Budgeting<br/>Token budget tracking per turn"]
    end

    MoreContext -.->|"tension"| LessContext
    LessContext -.->|"bounded by"| Limit
    Problem -->|"resolved by"| Solution

    style Problem fill:#e74c3c,color:#fff
    style Solution fill:#27ae60,color:#fff
```

---

## 2. The Context Management Pipeline

Every API call goes through a multi-stage context preparation pipeline.

```mermaid
flowchart TD
    subgraph Input["Raw Messages"]
        Messages["messages[] array<br/>(user, assistant, tool results,<br/>attachments, system)"]
    end

    subgraph Stage1["Stage 1: Tool Result Budget"]
        TRB["applyToolResultBudget()<br/>- Cap individual tool results<br/>- Persist oversized results to disk<br/>- Replace with file reference"]
    end

    subgraph Stage2["Stage 2: History Snip"]
        HS["snipCompactIfNeeded()<br/>- Remove old completed tool exchanges<br/>- Keep recent + important messages<br/>- Surgical, no AI summary needed"]
    end

    subgraph Stage3["Stage 3: Context Collapse"]
        CC["contextCollapse()<br/>- Fold consecutive read/search ops<br/>- Compress tool result chains<br/>- Reduce visual noise"]
    end

    subgraph Stage4["Stage 4: Microcompact"]
        MC["microcompact()<br/>- Use cached summaries of prior compacts<br/>- Avoid re-summarizing already-summarized content"]
    end

    subgraph Stage5["Stage 5: Auto-Compact Check"]
        AC{"Token count ><br/>threshold?"}
        Compact["compactConversation()<br/>- AI summarizes full history<br/>- Creates compact boundary message<br/>- Preserves key context"]
        Skip["Continue to API call"]
    end

    subgraph Stage6["Stage 6: Normalize for API"]
        Norm["normalizeMessagesForAPI()<br/>- Convert internal types → API types<br/>- Strip metadata, UUIDs<br/>- Merge consecutive same-role messages"]
    end

    Input --> Stage1 --> Stage2 --> Stage3 --> Stage4 --> Stage5
    AC -->|"Yes"| Compact --> Stage6
    AC -->|"No"| Skip --> Stage6

    style Stage1 fill:#3498db,color:#fff
    style Stage2 fill:#9b59b6,color:#fff
    style Stage3 fill:#e67e22,color:#fff
    style Stage4 fill:#1abc9c,color:#fff
    style Stage5 fill:#e74c3c,color:#fff
    style Stage6 fill:#2c3e50,color:#fff
```

---

## 3. Token Counting & Estimation

Accurate token counting is critical — over-estimate and you compact too early, under-estimate and you hit API errors.

```mermaid
flowchart LR
    subgraph Strategies["Token Counting Strategies"]
        direction TB
        APIUsage["1. API Usage Data<br/>Most accurate — from last response<br/>input_tokens + cache_tokens + output_tokens"]
        CountAPI["2. Count Tokens API<br/>Exact count via API call<br/>(Bedrock/Vertex/Anthropic)"]
        Estimation["3. Rough Estimation<br/>Fast heuristic: bytes / 4<br/>Used when no API data available"]
    end

    subgraph Function["tokenCountWithEstimation()"]
        direction TB
        Check["Check: has recent API usage?"]
        Yes["Use getTokenCountFromUsage()"]
        No["Use roughTokenCountEstimationForMessages()"]
        Check -->|"Yes"| Yes
        Check -->|"No"| No
    end

    subgraph Gotchas["Token Counting Gotchas"]
        direction TB
        G1["Split assistant messages share<br/>the same message.id but different<br/>usage — deduplicate!"]
        G2["Cache tokens count toward<br/>context window size but not<br/>toward billing the same way"]
        G3["Synthetic messages (internal)<br/>are excluded from counts"]
    end

    Strategies --> Function --> Gotchas

    style Strategies fill:#2980b9,color:#fff
    style Function fill:#27ae60,color:#fff
    style Gotchas fill:#f39c12,color:#333
```

### The roughTokenCountEstimation Heuristic

```typescript
// ~4 bytes per token for English text
const BYTES_PER_TOKEN = 4
function roughTokenCountEstimation(text: string): number {
  return Math.ceil(Buffer.byteLength(text, 'utf-8') / BYTES_PER_TOKEN)
}
```

This is intentionally simple — it's used when speed matters more than precision (e.g., during tool result budgeting). The API usage data from the previous turn provides the accurate count for threshold decisions.

---

## 4. Auto-Compact: The Primary Safety Net

Auto-compact is the workhorse of context management. It triggers when context approaches the window limit.

```mermaid
flowchart TD
    subgraph Thresholds["Threshold Calculation"]
        ContextWindow["Context Window Size<br/>(200K or 1M tokens)"]
        ReserveOutput["Reserve for Output<br/>-20K tokens"]
        EffectiveWindow["Effective Window<br/>= contextWindow - 20K"]
        AutoCompactBuffer["Auto-Compact Buffer<br/>-13K tokens"]
        Threshold["Auto-Compact Threshold<br/>= effective - 13K"]

        ContextWindow --> ReserveOutput --> EffectiveWindow --> AutoCompactBuffer --> Threshold
    end

    subgraph Example["Example: 200K Context"]
        E1["200,000 - 20,000 = 180,000 effective"]
        E2["180,000 - 13,000 = 167,000 threshold"]
        E3["Compact triggers at ~167K tokens"]
    end

    subgraph WarningLevels["Warning Levels"]
        Warning["Warning: effective - 20K<br/>(~160K for 200K window)"]
        Error["Error: effective - 20K<br/>(same, but different UI state)"]
        AutoCompact["Auto-Compact: effective - 13K<br/>(~167K for 200K window)"]
    end

    Thresholds --> Example
    Example --> WarningLevels

    style Thresholds fill:#2980b9,color:#fff
    style Example fill:#27ae60,color:#fff
    style WarningLevels fill:#f39c12,color:#333
```

### The Compaction Process

```mermaid
sequenceDiagram
    participant Loop as Query Loop
    participant AutoC as Auto-Compact
    participant Fork as Forked Agent
    participant API as Claude API

    Loop->>AutoC: Token count > threshold
    AutoC->>AutoC: isAutoCompactEnabled()?

    alt Enabled
        AutoC->>AutoC: executePreCompactHooks()
        AutoC->>Fork: runForkedAgent(messages, "Summarize conversation")
        Fork->>API: Send full history with compact prompt
        API-->>Fork: Summary (~2-5K tokens)
        Fork-->>AutoC: Compact summary
        AutoC->>AutoC: buildPostCompactMessages()
        Note over AutoC: Creates SystemCompactBoundaryMessage<br/>with summary text
        AutoC->>AutoC: Re-inject CLAUDE.md attachments
        AutoC->>AutoC: Re-inject plan file if active
        AutoC->>AutoC: executePostCompactHooks()
        AutoC->>AutoC: trySessionMemoryCompaction()
        AutoC-->>Loop: Compacted messages[] (much smaller)
    else Disabled or circuit-breaker tripped
        AutoC-->>Loop: Original messages (unchanged)
    end
```

### The Circuit Breaker

Auto-compact has a circuit breaker to prevent infinite retry loops:

```typescript
const MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES = 3
```

If compaction fails 3 times in a row (e.g., the context is so large even the compact call hits `prompt_too_long`), it stops trying. This prevents wasting ~250K API calls/day globally (measured from production data).

---

## 5. Reactive Compact: Emergency Recovery

When auto-compact isn't enough and the API returns `prompt_too_long`, reactive compact kicks in.

```mermaid
flowchart TD
    subgraph Trigger["Trigger: prompt_too_long Error"]
        APIError["API returns 400:<br/>'prompt is too long'"]
    end

    subgraph Recovery["Reactive Compact Recovery"]
        Check{"Already attempted<br/>reactive compact<br/>this iteration?"}
        Yes["Give up — yield error<br/>to user"]
        No["Run compactConversation()<br/>with force flag"]
        Result{"Compact<br/>succeeded?"}
        Retry["Retry API call with<br/>compacted messages"]
        Fail["Yield original error"]
    end

    APIError --> Check
    Check -->|"Yes"| Yes
    Check -->|"No"| No
    No --> Result
    Result -->|"Yes"| Retry
    Result -->|"No"| Fail

    style Trigger fill:#e74c3c,color:#fff
    style Recovery fill:#f39c12,color:#333
```

**Key insight**: Reactive compact is a **one-shot** recovery. If it fails, the error propagates to the user. This prevents infinite compact→retry→error loops.

---

## 6. History Snip: Surgical Message Removal

History Snip (feature flag: `HISTORY_SNIP`) is a lightweight alternative to full compaction.

```mermaid
flowchart LR
    subgraph Before["Before Snip"]
        direction TB
        M1["Turn 1: Read file A (500 tokens)"]
        M2["Turn 2: Edit file A (200 tokens)"]
        M3["Turn 3: Read file B (800 tokens)"]
        M4["Turn 4: Read file C (600 tokens)"]
        M5["Turn 5: Edit file B (300 tokens)"]
        M6["Turn 6: Read file D (current)"]
    end

    subgraph After["After Snip"]
        direction TB
        S1["Turn 1: [snipped]"]
        S2["Turn 2: Edit file A (kept)"]
        S3["Turn 3: [snipped]"]
        S4["Turn 4: Read file C (kept — recent)"]
        S5["Turn 5: Edit file B (kept)"]
        S6["Turn 6: Read file D (kept — current)"]
    end

    Before -->|"snipCompactIfNeeded()"| After

    style Before fill:#e74c3c,color:#fff
    style After fill:#27ae60,color:#fff
```

### What Gets Snipped

- Old read/search tool results whose content is no longer relevant
- Tool results that have been superseded by edits
- Messages beyond a sliding window of recent history

### What's Preserved

- Recent messages (within a configurable window)
- Write/edit tool calls (they represent state changes)
- The compact boundary summary (if one exists)
- Messages the model is currently referencing

---

## 7. Context Collapse: Read/Search Folding

Context Collapse (feature flag: `CONTEXT_COLLAPSE`) folds consecutive read/search operations into compact groups for the UI, reducing visual and cognitive noise.

```mermaid
flowchart TD
    subgraph Raw["Raw Message Stream"]
        R1["Assistant: Let me search for the file"]
        R2["Tool: Glob('*.ts') → 15 results"]
        R3["Tool: Grep('function') → 30 lines"]
        R4["Tool: Read('src/foo.ts') → 200 lines"]
        R5["Tool: Read('src/bar.ts') → 150 lines"]
        R6["Assistant: I found the issue..."]
    end

    subgraph Collapsed["Collapsed View"]
        C1["Assistant: Let me search for the file"]
        C2["📁 Read 2 files, searched 2 patterns<br/>(expandable group)"]
        C3["Assistant: I found the issue..."]
    end

    Raw -->|"collapseReadSearch()"| Collapsed

    style Raw fill:#95a5a6,color:#fff
    style Collapsed fill:#27ae60,color:#fff
```

### Collapsibility Detection

```typescript
type SearchOrReadResult = {
  isCollapsible: boolean  // Can this be folded?
  isSearch: boolean       // Glob/Grep operations
  isRead: boolean         // File read operations
  isList: boolean         // Directory listing
  isREPL: boolean         // REPL tool usage
  isMemoryWrite: boolean  // Memory file writes
  isAbsorbedSilently: boolean  // Snip/ToolSearch — no count increment
}
```

Tools are categorized by their read/write nature. Read-only tools are collapsible; write tools break the collapse group because they represent observable state changes.

---

## 8. Tool Result Budget & Persistence

Large tool results are the primary source of context bloat. The tool result budget system prevents any single tool from consuming too much context.

```mermaid
flowchart TD
    subgraph ToolResult["Tool Produces Result"]
        Result["Tool result:<br/>e.g., 100K chars from reading<br/>a large log file"]
    end

    subgraph Budget["Budget Check"]
        Check{"Result size ><br/>persistence threshold?"}
        Small["Result fits in context<br/>→ Include as-is"]
        Large["Result exceeds threshold<br/>→ Persist to disk"]
    end

    subgraph Persist["Disk Persistence"]
        Write["Write to:<br/>~/.claude/projects/<hash>/sessions/<id>/tool-results/<uuid>.txt"]
        Replace["Replace result with:<br/>'<persisted-output>\\nTool output too large...\\nSaved to: /path/to/file\\nUse Read tool to access.\\n</persisted-output>'"]
    end

    subgraph Thresholds["Threshold Configuration"]
        Default["Default: 50K chars<br/>(DEFAULT_MAX_RESULT_SIZE_CHARS)"]
        PerTool["Per-tool override:<br/>maxResultSizeChars in tool definition"]
        GrowthBook["GrowthBook override:<br/>tengu_satin_quoll flag<br/>(per-tool thresholds)"]
        Infinite["Special: Infinity<br/>(tool self-manages its output size)"]
    end

    ToolResult --> Budget
    Check -->|"No"| Small
    Check -->|"Yes"| Large --> Persist
    Thresholds --> Budget

    style ToolResult fill:#2980b9,color:#fff
    style Budget fill:#f39c12,color:#333
    style Persist fill:#e74c3c,color:#fff
    style Thresholds fill:#27ae60,color:#fff
```

### The FRC (Function Result Clearing) Pattern

Beyond budgeting new results, old tool results are proactively cleared:

```
[Old tool result content cleared]
```

The system prompt warns the model:
> "When working with tool results, write down any important information you might need later in your response, as the original tool result may be cleared later."

This trains the model to **extract and summarize** key findings immediately, a behavior essential for long sessions.

---

## 9. Microcompact & Caching

Microcompact (feature flag: `CACHED_MICROCOMPACT`) avoids redundant summarization by caching compact summaries.

```mermaid
flowchart LR
    subgraph Problem["Problem"]
        P1["Conversation compacted at turn 10<br/>Summary: 'User asked to fix auth bug...'"]
        P2["At turn 20, need to compact again"]
        P3["Without cache: re-summarize turns 1-20<br/>INCLUDING the already-summarized turns 1-10"]
    end

    subgraph Solution["Microcompact Solution"]
        S1["Cache the summary from turn 10"]
        S2["At turn 20: reuse cached summary<br/>for turns 1-10, only summarize 11-20"]
        S3["Compose: cached summary + new summary"]
    end

    Problem -->|"waste"| Solution

    style Problem fill:#e74c3c,color:#fff
    style Solution fill:#27ae60,color:#fff
```

### How It Works

1. After compaction, the summary is stored as a `MicrocompactBoundaryMessage`
2. On next compaction, messages after the boundary are the only ones needing summarization
3. The cached summary is prepended to the new summary
4. Result: compaction gets cheaper over time as more history is pre-summarized

---

## 10. Session Memory: Surviving Compaction

When compaction discards message history, critical information can be lost. Session memory persists key facts across compact boundaries.

```mermaid
flowchart TD
    subgraph Before["Before Compaction"]
        Msgs["Full message history<br/>(150K tokens)"]
        Facts["Key facts embedded in messages:<br/>- User prefers TypeScript<br/>- Working on auth module<br/>- 3 files already edited<br/>- Test command is 'bun test'"]
    end

    subgraph Compact["Compaction"]
        Summary["AI Summary: 'User is fixing auth<br/>bug in login.ts. Has edited 3 files.<br/>Tests passing.'"]
    end

    subgraph SessionMem["Session Memory (post-compact)"]
        SM1["trySessionMemoryCompaction()"]
        SM2["Extract persistent facts"]
        SM3["Store in session memory file"]
        SM4["Re-inject as attachment on next turn"]
    end

    subgraph After["After Compaction"]
        CompactedMsgs["Compact boundary + summary<br/>(~5K tokens)"]
        Reinjected["Re-injected context:<br/>- CLAUDE.md files<br/>- Active plan file<br/>- Session memory<br/>- File state cache"]
    end

    Before --> Compact --> SessionMem
    SessionMem --> After

    style Before fill:#e74c3c,color:#fff
    style Compact fill:#f39c12,color:#333
    style SessionMem fill:#2980b9,color:#fff
    style After fill:#27ae60,color:#fff
```

---

## 11. Token Budget: Spending Control

The token budget system (feature flag: `TOKEN_BUDGET`) allows users to specify how many tokens they want the model to spend on a task.

```mermaid
flowchart TD
    subgraph UserInput["User Specifies Budget"]
        Input["'+500k' or 'spend 2M tokens'"]
        Parse["parseTokenBudget()<br/>→ 500,000 tokens"]
    end

    subgraph Tracking["Budget Tracking"]
        Tracker["BudgetTracker:<br/>- continuationCount<br/>- lastDeltaTokens<br/>- lastGlobalTurnTokens"]
        Check["checkTokenBudget()"]
    end

    subgraph Decision["Decision Logic"]
        UnderBudget{"turnTokens <<br/>90% of budget?"}
        Diminishing{"3+ continuations AND<br/>delta < 500 tokens?"}
        Continue["ACTION: Continue<br/>Inject nudge message:<br/>'You've used X% of your<br/>Y token budget. Keep going.'"]
        Stop["ACTION: Stop<br/>Budget reached or<br/>diminishing returns"]
    end

    UserInput --> Tracking --> Decision
    UnderBudget -->|"Yes"| Continue
    UnderBudget -->|"No"| Stop
    Continue --> Diminishing
    Diminishing -->|"Yes"| Stop
    Diminishing -->|"No"| Continue

    style UserInput fill:#2980b9,color:#fff
    style Tracking fill:#8e44ad,color:#fff
    style Decision fill:#27ae60,color:#fff
```

### The Diminishing Returns Guard

The system detects when the model is making less than 500 tokens of progress per continuation after 3+ attempts. This prevents the model from spinning its wheels when it's effectively done but hasn't explicitly said so.

---

## 12. The Complete Decision Flow

Here's how all the compaction strategies work together in a single query loop iteration.

```mermaid
flowchart TD
    Start["Query Loop Iteration Start"]

    Start --> TRB["1. Apply Tool Result Budget<br/>Persist oversized results to disk"]
    TRB --> Snip{"2. HISTORY_SNIP<br/>enabled?"}
    Snip -->|"Yes"| DoSnip["snipCompactIfNeeded()<br/>Remove old tool exchanges"]
    Snip -->|"No"| SkipSnip["Skip"]
    DoSnip --> Collapse
    SkipSnip --> Collapse

    Collapse{"3. CONTEXT_COLLAPSE<br/>enabled?"} -->|"Yes"| DoCollapse["contextCollapse()<br/>Fold read/search groups"]
    Collapse -->|"No"| SkipCollapse["Skip"]
    DoCollapse --> Normalize
    SkipCollapse --> Normalize

    Normalize["4. normalizeMessagesForAPI()<br/>Convert to API format"]

    Normalize --> APICall["5. Send to Claude API"]
    APICall --> APIResult{"API Response"}

    APIResult -->|"Success"| ProcessResponse["Process response,<br/>execute tools"]
    APIResult -->|"prompt_too_long"| Reactive{"6. REACTIVE_COMPACT<br/>already attempted?"}

    Reactive -->|"No"| DoReactive["compactConversation(force=true)<br/>Emergency compaction"]
    DoReactive --> RetryAPI["Retry API call"]
    Reactive -->|"Yes"| GiveUp["Yield error to user"]

    ProcessResponse --> CheckThreshold{"7. Token count ><br/>auto-compact threshold?"}
    CheckThreshold -->|"Yes"| CircuitBreaker{"Consecutive failures<br/>< 3?"}
    CircuitBreaker -->|"Yes"| AutoCompact["compactConversation()<br/>Summarize + compact"]
    CircuitBreaker -->|"No"| SkipAutoCompact["Skip (circuit breaker)"]
    CheckThreshold -->|"No"| Continue["Continue to next iteration"]
    AutoCompact --> Continue
    SkipAutoCompact --> Continue

    style Start fill:#2c3e50,color:#fff
    style AutoCompact fill:#e74c3c,color:#fff
    style DoReactive fill:#c0392b,color:#fff
    style DoSnip fill:#9b59b6,color:#fff
    style DoCollapse fill:#e67e22,color:#fff
    style Continue fill:#27ae60,color:#fff
```

### Context Window Sizes

| Model | Default Window | 1M Enabled | Effective (after reserves) |
|---|---|---|---|
| Claude Sonnet 4.6 | 200K | 1M (opt-in) | 167K / 967K |
| Claude Opus 4.6 | 200K | 1M (with suffix) | 167K / 967K |
| Claude Haiku 4.5 | 200K | No | 167K |
| Custom/Bedrock | Per model capability | Via env override | Capability - 33K |

### Environment Overrides

| Variable | Effect |
|---|---|
| `CLAUDE_CODE_AUTO_COMPACT_WINDOW` | Cap the effective context window size |
| `CLAUDE_AUTOCOMPACT_PCT_OVERRIDE` | Set compact threshold as % of effective window |
| `CLAUDE_CODE_MAX_CONTEXT_TOKENS` | Override context window detection entirely |
| `CLAUDE_CODE_DISABLE_1M_CONTEXT` | Force 200K even on 1M-capable models |
