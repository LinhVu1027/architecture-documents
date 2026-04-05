# Cost & Token Economics

> How Claude Code tracks, optimizes, and reports the real-dollar cost of AI-powered coding — from per-token pricing tiers to session-level cost dashboards. Every diagram is a Mermaid diagram you can render in any Markdown viewer.

---

## Table of Contents

1. [Why Cost Matters for AI Coding Agents](#1-why-cost-matters-for-ai-coding-agents)
2. [The Pricing Model Landscape](#2-the-pricing-model-landscape)
3. [Cost Tracking Architecture](#3-cost-tracking-architecture)
4. [Token Flow: Where Tokens Are Spent](#4-token-flow-where-tokens-are-spent)
5. [Prompt Caching: The 90% Discount](#5-prompt-caching-the-90-discount)
6. [The Max Output Tokens Optimization](#6-the-max-output-tokens-optimization)
7. [Model Selection Economics](#7-model-selection-economics)
8. [Subagent Cost Isolation](#8-subagent-cost-isolation)
9. [Cost Reporting & Dashboards](#9-cost-reporting--dashboards)
10. [Cost Optimization Strategies Summary](#10-cost-optimization-strategies-summary)

---

## 1. Why Cost Matters for AI Coding Agents

An AI coding agent that runs for hours can easily spend $10-50+ per session. Understanding and optimizing cost is essential for making the tool accessible.

```mermaid
graph TB
    subgraph CostDrivers["Primary Cost Drivers"]
        Input["Input Tokens<br/>System prompt + messages + tool results<br/>(grows with conversation length)"]
        Output["Output Tokens<br/>Model responses + tool calls<br/>(3-5x more expensive per token)"]
        Cache["Cache Operations<br/>Write: 1.25x input price<br/>Read: 0.1x input price"]
        WebSearch["Web Search<br/>$0.01 per request"]
    end

    subgraph Scale["Scale Factors"]
        Turns["# of Turns<br/>Each turn re-sends full history"]
        Tools["# of Tool Calls<br/>Each adds input + output tokens"]
        FileSize["File Sizes<br/>Large files = large tool results"]
        Agents["Subagents<br/>Each runs independent query loops"]
    end

    CostDrivers --> Scale

    style CostDrivers fill:#e74c3c,color:#fff
    style Scale fill:#f39c12,color:#333
```

---

## 2. The Pricing Model Landscape

Claude Code supports multiple model tiers with dramatically different cost structures.

```mermaid
graph LR
    subgraph Tiers["Model Pricing Tiers (per million tokens)"]
        direction TB
        Haiku35["Haiku 3.5<br/>$0.80 in / $4.00 out<br/>Cache write: $1.00<br/>Cache read: $0.08"]
        Haiku45["Haiku 4.5<br/>$1.00 in / $5.00 out<br/>Cache write: $1.25<br/>Cache read: $0.10"]
        Sonnet["Sonnet 4/4.5/4.6<br/>$3.00 in / $15.00 out<br/>Cache write: $3.75<br/>Cache read: $0.30"]
        Opus4["Opus 4/4.1<br/>$15.00 in / $75.00 out<br/>Cache write: $18.75<br/>Cache read: $1.50"]
        Opus46["Opus 4.5/4.6<br/>$5.00 in / $25.00 out<br/>Cache write: $6.25<br/>Cache read: $0.50"]
        Opus46Fast["Opus 4.6 (Fast Mode)<br/>$30.00 in / $150.00 out<br/>Cache write: $37.50<br/>Cache read: $3.00"]
    end

    style Haiku35 fill:#27ae60,color:#fff
    style Haiku45 fill:#2ecc71,color:#fff
    style Sonnet fill:#3498db,color:#fff
    style Opus4 fill:#9b59b6,color:#fff
    style Opus46 fill:#8e44ad,color:#fff
    style Opus46Fast fill:#e74c3c,color:#fff
```

### The Output Token Multiplier

Output tokens are **3-6x more expensive** than input tokens across all tiers. This has profound implications:
- **Verbose model responses** cost disproportionately more
- **Prompt engineering for conciseness** has direct cost impact (the "≤25 words between tools" instruction)
- **Tool calls** (which are output tokens) are expensive

---

## 3. Cost Tracking Architecture

Cost tracking flows through a centralized state system.

```mermaid
flowchart TD
    subgraph APIResponse["API Response"]
        Usage["usage: {<br/>  input_tokens: 5000,<br/>  output_tokens: 800,<br/>  cache_creation_input_tokens: 12000,<br/>  cache_read_input_tokens: 8000,<br/>  server_tool_use: { web_search_requests: 2 }<br/>}"]
    end

    subgraph Calculation["Cost Calculation"]
        GetModel["getCanonicalName(model)"]
        GetCosts["getModelCosts(model, usage)"]
        Compute["tokensToUSDCost(modelCosts, usage)<br/>= (input/1M × inputPrice)<br/>+ (output/1M × outputPrice)<br/>+ (cacheWrite/1M × writePrice)<br/>+ (cacheRead/1M × readPrice)<br/>+ (webSearches × $0.01)"]
    end

    subgraph State["Global Cost State (bootstrap/state.ts)"]
        Counters["totalCostUSD: number<br/>totalInputTokens: number<br/>totalOutputTokens: number<br/>totalCacheCreationTokens: number<br/>totalCacheReadTokens: number<br/>totalWebSearchRequests: number<br/>totalAPIDuration: number<br/>totalToolDuration: number<br/>modelUsage: Map<string, ModelUsage>"]
    end

    subgraph Persistence["Session Cost Persistence"]
        Save["saveCurrentSessionCosts()<br/>→ project config file"]
        Restore["getStoredSessionCosts()<br/>← project config file<br/>(survives process restart)"]
    end

    APIResponse --> Calculation --> State --> Persistence

    style APIResponse fill:#2980b9,color:#fff
    style Calculation fill:#8e44ad,color:#fff
    style State fill:#27ae60,color:#fff
    style Persistence fill:#f39c12,color:#333
```

### Per-Model Usage Tracking

Costs are tracked **per model** because a single session may use multiple models:
- Main loop: Opus 4.6
- Subagents: Sonnet 4.6 or Haiku (via `model` override)
- Compact operations: Same model or cheaper model
- Fast mode toggle: Changes Opus 4.6 pricing tier mid-session

---

## 4. Token Flow: Where Tokens Are Spent

Understanding where tokens go reveals the biggest optimization opportunities.

```mermaid
pie title "Typical Token Distribution (20-turn session)"
    "System Prompt (repeated)" : 15
    "Conversation History" : 30
    "Tool Results" : 25
    "Model Output (text)" : 10
    "Model Output (tool calls)" : 8
    "Attachments (CLAUDE.md, etc.)" : 7
    "Thinking Tokens" : 5
```

```mermaid
flowchart LR
    subgraph Input["INPUT TOKENS (cheaper)"]
        direction TB
        SystemPrompt["System Prompt<br/>~8-12K tokens<br/>(sent EVERY turn)"]
        History["Conversation History<br/>Grows linearly with turns<br/>(until compaction)"]
        ToolResults["Tool Results<br/>File contents, search results<br/>(largest variable)"]
        Attachments["Attachments<br/>CLAUDE.md, memory, plans<br/>(relatively stable)"]
    end

    subgraph Output["OUTPUT TOKENS (expensive)"]
        direction TB
        Text["Model Text<br/>Explanations, summaries"]
        ToolCalls["Tool Call JSON<br/>{ name, input } blocks"]
        Thinking["Thinking Tokens<br/>Internal reasoning<br/>(not billed at full rate)"]
    end

    subgraph Optimization["Optimization Targets"]
        direction TB
        O1["Prompt Caching<br/>→ System prompt at 0.1x"]
        O2["Tool Result Budget<br/>→ Cap large results"]
        O3["Compaction<br/>→ Summarize old history"]
        O4["Concise Output<br/>→ '≤25 words between tools'"]
        O5["Max Output Cap<br/>→ 8K default, escalate on need"]
    end

    Input --> Optimization
    Output --> Optimization

    style Input fill:#3498db,color:#fff
    style Output fill:#e74c3c,color:#fff
    style Optimization fill:#27ae60,color:#fff
```

---

## 5. Prompt Caching: The 90% Discount

Prompt caching is the most impactful cost optimization. It turns the system prompt from the biggest cost to nearly free.

```mermaid
sequenceDiagram
    participant Turn1 as Turn 1
    participant Turn2 as Turn 2
    participant Turn10 as Turn 10
    participant Cache as Prompt Cache

    Note over Turn1: System prompt: 10K tokens
    Turn1->>Cache: Cache WRITE (1.25x price)<br/>= 10K × $5/M × 1.25 = $0.0625

    Note over Turn2: System prompt: same 10K tokens
    Turn2->>Cache: Cache READ (0.1x price)<br/>= 10K × $5/M × 0.1 = $0.005

    Note over Turn10: System prompt: still same
    Turn10->>Cache: Cache READ<br/>= $0.005

    Note over Cache: Savings after 10 turns:<br/>Without cache: 10 × $0.05 = $0.50<br/>With cache: $0.0625 + 9 × $0.005 = $0.1075<br/>Savings: 78.5%
```

### Global vs Session Cache Scope

```mermaid
graph LR
    subgraph GlobalScope["scope: 'global'"]
        G1["Shared across ALL users<br/>on same model"]
        G2["Static system prompt prefix"]
        G3["Hit rate: ~95%+ for popular models"]
        G4["Most cost-effective"]
    end

    subgraph SessionScope["scope: 'session' (default)"]
        S1["Shared within one session"]
        S2["Dynamic prompt + message prefix"]
        S3["Hit rate: ~100% within session"]
        S4["Effective for conversation prefix"]
    end

    style GlobalScope fill:#27ae60,color:#fff
    style SessionScope fill:#3498db,color:#fff
```

---

## 6. The Max Output Tokens Optimization

A subtle but significant cost optimization: capping default output tokens.

```mermaid
flowchart TD
    subgraph Problem["The Problem"]
        P1["Default max_tokens: 32K or 64K"]
        P2["Actual p99 output: ~5K tokens"]
        P3["Over-reservation: 8-16x more<br/>than actually needed"]
        P4["Server reserves output token slots<br/>→ more expensive inference"]
    end

    subgraph Solution["The Capped Default Solution"]
        S1["CAPPED_DEFAULT_MAX_TOKENS = 8,000"]
        S2["<1% of requests actually hit 8K limit"]
        S3["Those requests get ONE clean retry<br/>at ESCALATED_MAX_TOKENS = 64,000"]
    end

    subgraph Flow["Execution Flow"]
        Start["API Request:<br/>max_tokens = 8,000"]
        Response{"Response ended<br/>due to max_tokens?"}
        Normal["Normal completion<br/>(99%+ of cases)"]
        Escalate["Set max_tokens = 64,000<br/>Continue from where model stopped<br/>(max_output_tokens recovery)"]

        Start --> Response
        Response -->|"No"| Normal
        Response -->|"Yes"| Escalate
    end

    Problem --> Solution --> Flow

    style Problem fill:#e74c3c,color:#fff
    style Solution fill:#27ae60,color:#fff
```

### The Recovery Limit

Max output token recovery is limited to **3 attempts** per query loop iteration. This prevents runaway costs if the model is generating extremely long output.

---

## 7. Model Selection Economics

Different tasks have dramatically different cost profiles.

```mermaid
graph TD
    subgraph Tasks["Task Types & Optimal Models"]
        direction TB
        Simple["Simple Tasks<br/>(file reads, searches)<br/>→ Haiku: $1-5/M"]
        Medium["Standard Coding<br/>(bug fixes, features)<br/>→ Sonnet: $3-15/M"]
        Complex["Complex Architecture<br/>(multi-file refactors)<br/>→ Opus: $5-25/M"]
        FastMode["Time-Critical<br/>(user waiting)<br/>→ Opus Fast: $30-150/M"]
    end

    subgraph Subagent["Subagent Model Selection"]
        direction TB
        SA1["Explore agents → Haiku<br/>(fast, cheap, read-only)"]
        SA2["Code review agents → Sonnet<br/>(quality matters)"]
        SA3["Implementation agents → Opus<br/>(complex reasoning)"]
        SA4["Compact operations → Same model<br/>(needs to understand context)"]
    end

    subgraph Impact["Cost Impact"]
        I1["A 30-turn session on Opus 4.6:<br/>~$2-8 depending on tool usage"]
        I2["Same session with Haiku subagents:<br/>~$1-4 (subagent cost drops 80%)"]
        I3["Same session with prompt caching:<br/>~$0.50-3 (input costs drop ~70%)"]
    end

    Tasks --> Subagent --> Impact

    style Simple fill:#27ae60,color:#fff
    style Medium fill:#3498db,color:#fff
    style Complex fill:#9b59b6,color:#fff
    style FastMode fill:#e74c3c,color:#fff
```

---

## 8. Subagent Cost Isolation

Subagents track their own costs, which roll up to the parent session.

```mermaid
flowchart TD
    subgraph Parent["Parent Session"]
        ParentCost["Session Total: $3.50"]
        ParentLoop["Main Loop: $2.00"]
    end

    subgraph Agents["Subagent Cost Tracking"]
        Agent1["Agent 1 (Explore)<br/>Model: Haiku<br/>Cost: $0.15<br/>Tokens: 50K in / 5K out"]
        Agent2["Agent 2 (Code Review)<br/>Model: Sonnet<br/>Cost: $0.85<br/>Tokens: 120K in / 20K out"]
        Agent3["Agent 3 (Compact)<br/>Model: Opus 4.6<br/>Cost: $0.50<br/>Tokens: 80K in / 5K out"]
    end

    subgraph Tracking["Cost Roll-up"]
        Accumulate["accumulateUsage()<br/>→ Per-model usage counters"]
        ForkedMetrics["tengu_fork_agent_query event<br/>→ Analytics with cost breakdown"]
        Total["Session total = sum of all"]
    end

    ParentLoop --> ParentCost
    Agent1 --> Tracking
    Agent2 --> Tracking
    Agent3 --> Tracking
    Tracking --> ParentCost

    style Parent fill:#2980b9,color:#fff
    style Agents fill:#8e44ad,color:#fff
    style Tracking fill:#27ae60,color:#fff
```

### Cache Sharing Between Parent and Subagent

The `CacheSafeParams` system ensures subagents can share the parent's prompt cache:

```typescript
type CacheSafeParams = {
  systemPrompt: SystemPrompt      // Must match parent
  userContext: { ... }             // Must match parent
  systemContext: { ... }           // Must match parent
  toolUseContext: ToolUseContext    // Must match parent
  forkContextMessages: Message[]   // Parent's messages as prefix
}
```

If any of these diverge, the subagent creates a new cache entry instead of sharing. This is why the `DANGEROUS_` prefix exists on volatile system prompt sections — they can inadvertently break cache sharing.

---

## 9. Cost Reporting & Dashboards

Claude Code provides real-time cost visibility to users.

```mermaid
flowchart LR
    subgraph RealTime["Real-Time Display"]
        StatusBar["Status bar:<br/>'$0.45 | 15K tokens | 3m 20s'"]
        PerTurn["Per-turn cost delta"]
        Warnings["Cost threshold warnings<br/>(CostThresholdDialog)"]
    end

    subgraph Session["Session Summary (on exit)"]
        Total["Total cost: $2.34"]
        Breakdown["Input: 250K tokens ($1.25)<br/>Output: 30K tokens ($0.75)<br/>Cache write: 12K ($0.08)<br/>Cache read: 180K ($0.09)<br/>Web search: 5 ($0.05)"]
        Duration["API time: 45s<br/>Tool time: 1m 30s<br/>Total time: 5m 15s"]
        Lines["Lines changed: +150 / -42"]
    end

    subgraph Persistence["Persistent Tracking"]
        ProjectConfig["Project config:<br/>Cumulative session costs"]
        Analytics["Analytics events:<br/>Per-model usage breakdowns"]
    end

    RealTime --> Session --> Persistence

    style RealTime fill:#2980b9,color:#fff
    style Session fill:#27ae60,color:#fff
    style Persistence fill:#f39c12,color:#333
```

---

## 10. Cost Optimization Strategies Summary

```mermaid
graph TD
    subgraph Strategies["All Cost Optimization Strategies"]
        S1["1. PROMPT CACHING<br/>Static prefix → 0.1x input cost<br/>Impact: ★★★★★"]
        S2["2. GLOBAL CACHE SCOPE<br/>Cross-org prefix sharing<br/>Impact: ★★★★☆"]
        S3["3. CAPPED MAX OUTPUT<br/>8K default, 64K on escalation<br/>Impact: ★★★☆☆"]
        S4["4. AUTO-COMPACTION<br/>Summarize old history<br/>Impact: ★★★★★"]
        S5["5. TOOL RESULT BUDGET<br/>Persist oversized results to disk<br/>Impact: ★★★★☆"]
        S6["6. CONCISE OUTPUT INSTRUCTIONS<br/>'≤25 words between tools'<br/>Impact: ★★★☆☆"]
        S7["7. SUBAGENT MODEL SELECTION<br/>Haiku for exploration, Sonnet for code<br/>Impact: ★★★☆☆"]
        S8["8. SECTION MEMOIZATION<br/>Compute prompt sections once<br/>Impact: ★★☆☆☆"]
        S9["9. HISTORY SNIP<br/>Remove old tool exchanges<br/>Impact: ★★★☆☆"]
        S10["10. CONTEXT COLLAPSE<br/>Fold read/search chains<br/>Impact: ★★☆☆☆"]
    end

    style S1 fill:#27ae60,color:#fff
    style S2 fill:#27ae60,color:#fff
    style S3 fill:#2ecc71,color:#fff
    style S4 fill:#27ae60,color:#fff
    style S5 fill:#27ae60,color:#fff
    style S6 fill:#2ecc71,color:#fff
    style S7 fill:#2ecc71,color:#fff
    style S8 fill:#3498db,color:#fff
    style S9 fill:#2ecc71,color:#fff
    style S10 fill:#3498db,color:#fff
```

### The Compound Effect

These strategies compound:
1. **Prompt caching** reduces the base cost per turn
2. **Compaction** keeps conversation history from growing unbounded
3. **Tool result budget** prevents single tool calls from dominating cost
4. **Max output cap** reduces wasted output reservation
5. **Subagent model selection** puts cheap models on cheap tasks

Together, a session that would cost $20 without optimization might cost $3-5 with all strategies active.
