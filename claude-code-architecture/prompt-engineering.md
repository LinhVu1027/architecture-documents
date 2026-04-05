# Prompt Engineering & System Prompt Architecture

> How Claude Code constructs, layers, caches, and evolves its system prompt to make the AI model maximally effective at coding tasks. This is the **most critical subsystem** — the prompt IS the product. Every diagram is a Mermaid diagram you can render in any Markdown viewer.

---

## Table of Contents

1. [Why Prompt Engineering Matters Most](#1-why-prompt-engineering-matters-most)
2. [System Prompt Assembly Pipeline](#2-system-prompt-assembly-pipeline)
3. [The Layered Prompt Architecture](#3-the-layered-prompt-architecture)
4. [Prompt Priority & Override Hierarchy](#4-prompt-priority--override-hierarchy)
5. [Static vs Dynamic Boundary — The Cache Split](#5-static-vs-dynamic-boundary--the-cache-split)
6. [System Prompt Section Registry](#6-system-prompt-section-registry)
7. [Context Injection Points](#7-context-injection-points)
8. [CLAUDE.md Injection & Memory Loading](#8-claudemd-injection--memory-loading)
9. [Tool Description Design Principles](#9-tool-description-design-principles)
10. [Prompt Caching Economics](#10-prompt-caching-economics)
11. [Model-Specific Prompt Adaptations](#11-model-specific-prompt-adaptations)
12. [The Behavioral Shaping Strategy](#12-the-behavioral-shaping-strategy)

---

## 1. Why Prompt Engineering Matters Most

The system prompt is the single highest-leverage component in an AI coding agent. It determines:
- **What the model will do** — task interpretation, tool selection, coding style
- **What the model won't do** — safety boundaries, destructive action avoidance
- **How well the model performs** — the difference between a hallucinating chatbot and a 10x developer

```mermaid
graph TB
    subgraph Impact["Prompt Engineering Impact"]
        direction TB
        Quality["Code Quality<br/>Correct, secure, minimal"]
        Safety["Safety<br/>No rm -rf, no secret leaks"]
        Efficiency["Efficiency<br/>Right tool, first try"]
        Style["User Experience<br/>Concise, actionable output"]
        Cost["Cost<br/>Prompt caching saves 90%+ on input"]
    end

    subgraph Mechanism["How Prompts Achieve This"]
        Identity["Identity Framing<br/>'You are Claude Code...'"]
        Rules["Behavioral Rules<br/>'Prefer Edit over sed...'"]
        Context["Dynamic Context<br/>CWD, git status, model info"]
        Memory["Persistent Memory<br/>CLAUDE.md + memory files"]
        Sections["Section Registry<br/>Cached vs volatile sections"]
    end

    Identity --> Quality
    Identity --> Style
    Rules --> Safety
    Rules --> Efficiency
    Context --> Quality
    Context --> Efficiency
    Memory --> Quality
    Memory --> Style
    Sections --> Cost

    style Impact fill:#2c3e50,color:#fff
    style Mechanism fill:#27ae60,color:#fff
```

### The Core Insight

Claude Code doesn't just have a system prompt — it has a **prompt assembly pipeline** with 15+ composable sections, a cache boundary marker, feature-flagged sections, and a section registry with memoization. The prompt is treated like production code, not a static string.

---

## 2. System Prompt Assembly Pipeline

The system prompt goes through a multi-stage assembly before reaching the API.

```mermaid
flowchart TD
    subgraph Stage1["Stage 1: getSystemPrompt()"]
        direction TB
        S1A["Compute static sections<br/>(intro, system, tasks, actions,<br/>tools, tone, efficiency)"]
        S1B["Insert DYNAMIC_BOUNDARY marker"]
        S1C["Resolve dynamic sections<br/>(session guidance, memory,<br/>env info, language, MCP, etc.)"]
        S1A --> S1B --> S1C
    end

    subgraph Stage2["Stage 2: buildEffectiveSystemPrompt()"]
        direction TB
        S2A{"Override<br/>prompt set?"}
        S2B{"Agent<br/>definition?"}
        S2C{"Custom<br/>--system-prompt?"}
        S2D["Use default prompt"]
        S2E["Append appendSystemPrompt"]
        S2A -->|"Yes"| S2F["Use override only"]
        S2A -->|"No"| S2B
        S2B -->|"Yes"| S2G["Agent prompt<br/>REPLACES default"]
        S2B -->|"No"| S2C
        S2C -->|"Yes"| S2H["Custom prompt<br/>replaces default"]
        S2C -->|"No"| S2D
        S2G --> S2E
        S2H --> S2E
        S2D --> S2E
    end

    subgraph Stage3["Stage 3: API Request Construction"]
        direction TB
        S3A["splitSysPromptPrefix()<br/>Split at DYNAMIC_BOUNDARY"]
        S3B["Static prefix → cache_control:<br/>scope: 'global'"]
        S3C["Dynamic suffix → cache_control:<br/>scope: 'session'"]
        S3D["prependUserContext()"]
        S3E["appendSystemContext()"]
        S3A --> S3B
        S3A --> S3C
        S3B --> S3D
        S3C --> S3E
    end

    Stage1 --> Stage2 --> Stage3

    style Stage1 fill:#2980b9,color:#fff
    style Stage2 fill:#8e44ad,color:#fff
    style Stage3 fill:#c0392b,color:#fff
```

**Key files:**
- `src/constants/prompts.ts` — The master prompt builder (`getSystemPrompt()`)
- `src/utils/systemPrompt.ts` — Priority resolution (`buildEffectiveSystemPrompt()`)
- `src/constants/systemPromptSections.ts` — Section registry with memoization
- `src/utils/api.ts` — Cache boundary splitting (`splitSysPromptPrefix()`)
- `src/context.ts` — User/system context injection

---

## 3. The Layered Prompt Architecture

The system prompt is composed of distinct semantic layers, each serving a specific purpose.

```mermaid
graph TD
    subgraph StaticLayers["STATIC LAYERS (Cache-Friendly)"]
        direction TB
        L1["Layer 1: Identity & Framing<br/><i>'You are Claude Code, Anthropic's<br/>official CLI for Claude'</i>"]
        L2["Layer 2: System Rules<br/><i>Tool display, permissions,<br/>system-reminders, hooks</i>"]
        L3["Layer 3: Task Behavior<br/><i>Read before edit, no over-engineering,<br/>security awareness, code style</i>"]
        L4["Layer 4: Action Safety<br/><i>Reversibility checks, blast radius,<br/>destructive operation warnings</i>"]
        L5["Layer 5: Tool Usage<br/><i>Prefer Read over cat,<br/>Edit over sed, parallel calls</i>"]
        L6["Layer 6: Tone & Style<br/><i>No emojis, file:line refs,<br/>concise output</i>"]
        L7["Layer 7: Output Efficiency<br/><i>Go straight to the point,<br/>lead with action not reasoning</i>"]
        L1 --> L2 --> L3 --> L4 --> L5 --> L6 --> L7
    end

    Boundary["═══ DYNAMIC_BOUNDARY ═══<br/>(Cache breakpoint)"]

    subgraph DynamicLayers["DYNAMIC LAYERS (Per-Session)"]
        direction TB
        D1["Layer 8: Session Guidance<br/><i>Agent tool config, explore agents,<br/>skill discovery, verification</i>"]
        D2["Layer 9: Memory<br/><i>CLAUDE.md content, auto memory</i>"]
        D3["Layer 10: Environment<br/><i>CWD, git status, platform,<br/>model name, knowledge cutoff</i>"]
        D4["Layer 11: Language<br/><i>'Always respond in Japanese'</i>"]
        D5["Layer 12: MCP Instructions<br/><i>Server-provided usage guides</i>"]
        D6["Layer 13: Feature-Gated<br/><i>Token budget, brief mode,<br/>scratchpad, FRC</i>"]
        D1 --> D2 --> D3 --> D4 --> D5 --> D6
    end

    StaticLayers --> Boundary --> DynamicLayers

    style StaticLayers fill:#27ae60,color:#fff
    style Boundary fill:#f39c12,color:#333
    style DynamicLayers fill:#e74c3c,color:#fff
```

### Why This Layering?

| Layer | Purpose | Cache Impact |
|---|---|---|
| Identity + Rules | Same for ALL users, ALL sessions | Global cache hit |
| Task + Actions | Same for ALL users (slight ant/external variance) | Global cache hit |
| Tools + Tone | Same for all users (feature-flag variants) | Global cache hit |
| **BOUNDARY** | **Separates cacheable from volatile** | **Cache split point** |
| Session Guidance | Varies by enabled tools, session type | Session cache |
| Memory/Env | Varies by project, user, model | Per-request |

---

## 4. Prompt Priority & Override Hierarchy

Multiple prompt sources can compete. The resolution follows a strict priority.

```mermaid
flowchart TD
    Start["Prompt Sources"] --> P0{"overrideSystemPrompt<br/>set?"}
    P0 -->|"Yes"| Override["USE OVERRIDE ONLY<br/>(e.g., loop mode)"]
    P0 -->|"No"| P1{"Coordinator<br/>mode active?"}
    P1 -->|"Yes"| Coord["USE COORDINATOR PROMPT<br/>+ appendSystemPrompt"]
    P1 -->|"No"| P2{"Agent definition<br/>on main thread?"}
    P2 -->|"Yes + Proactive"| AgentAppend["APPEND agent prompt<br/>to default prompt"]
    P2 -->|"Yes + Normal"| AgentReplace["REPLACE default<br/>with agent prompt"]
    P2 -->|"No"| P3{"--system-prompt<br/>CLI flag?"}
    P3 -->|"Yes"| Custom["USE CUSTOM PROMPT<br/>+ appendSystemPrompt"]
    P3 -->|"No"| Default["USE DEFAULT PROMPT<br/>+ appendSystemPrompt"]

    style Override fill:#e74c3c,color:#fff
    style Coord fill:#8e44ad,color:#fff
    style AgentAppend fill:#f39c12,color:#333
    style AgentReplace fill:#2ecc71,color:#fff
    style Custom fill:#3498db,color:#fff
    style Default fill:#1abc9c,color:#fff
```

### Key Design Decision: Agent Append vs Replace

- **Normal mode**: Agent prompt **replaces** the default prompt entirely. The agent IS the identity.
- **Proactive mode**: Agent prompt is **appended** to the default autonomous agent prompt. This mirrors how teammates work — they add domain instructions on top of a shared base.

---

## 5. Static vs Dynamic Boundary — The Cache Split

This is the most cost-impactful design decision in the entire system.

```mermaid
sequenceDiagram
    participant Client as Claude Code
    participant API as Anthropic API
    participant Cache as Prompt Cache

    Note over Client: Turn 1
    Client->>API: [Static prefix | Dynamic suffix | Messages]
    API->>Cache: Hash static prefix → cache MISS
    Cache-->>API: Store prefix (scope: global)
    API-->>Client: Response (full input cost)

    Note over Client: Turn 2 (same session)
    Client->>API: [Static prefix | Dynamic suffix' | Messages']
    API->>Cache: Hash static prefix → cache HIT
    Cache-->>API: Return cached prefix
    Note over API: Only dynamic + messages<br/>counted as new input
    API-->>Client: Response (reduced input cost)

    Note over Client: Turn 1 (DIFFERENT user, same model)
    Client->>API: [Static prefix | Different dynamic | Messages]
    API->>Cache: Hash static prefix → cache HIT (global!)
    Cache-->>API: Return cached prefix
    Note over API: Global scope means<br/>cross-org cache sharing
    API-->>Client: Response (reduced input cost)
```

### The SYSTEM_PROMPT_DYNAMIC_BOUNDARY Marker

```
__SYSTEM_PROMPT_DYNAMIC_BOUNDARY__
```

This string literal in the prompt array is detected by `splitSysPromptPrefix()` in `src/utils/api.ts`:

- **Everything before** → `cache_control: { type: 'ephemeral', scope: 'global' }`
- **Everything after** → `cache_control: { type: 'ephemeral' }` (session-scoped)

### Why This Saves Massive Costs

The static prefix is ~8,000-12,000 tokens. With prompt caching:
- **Cache write**: 1.25x the normal input price (one-time)
- **Cache read**: 0.1x the normal input price (every subsequent turn)
- **Savings**: ~90% reduction on input tokens for the static portion

For a session with 20 turns, this means the static prefix is paid in full once, then at 10% for the remaining 19 turns.

---

## 6. System Prompt Section Registry

Sections are not raw strings — they're managed through a registry with memoization.

```mermaid
flowchart LR
    subgraph Registry["Section Registry"]
        direction TB
        Cached["systemPromptSection(name, compute)<br/>→ Memoized: computed ONCE,<br/>cached until /clear or /compact"]
        Volatile["DANGEROUS_uncachedSection(name, compute, reason)<br/>→ Volatile: recomputed EVERY turn<br/>BREAKS prompt cache when value changes"]
    end

    subgraph Cache["Section Cache (AppState)"]
        direction TB
        Map["Map<string, string | null>"]
        Get["cache.get('session_guidance')"]
        Set["cache.set('env_info', value)"]
    end

    subgraph Lifecycle["Cache Lifecycle"]
        direction TB
        Clear1["/clear command"]
        Clear2["/compact command"]
        Clear3["clearSystemPromptSections()"]
        Clear1 --> Clear3
        Clear2 --> Clear3
        Clear3 --> Reset["Reset cache + beta header latches"]
    end

    Registry --> Cache --> Lifecycle

    style Cached fill:#27ae60,color:#fff
    style Volatile fill:#e74c3c,color:#fff
```

### Registered Sections

| Section Name | Type | What It Contains |
|---|---|---|
| `session_guidance` | Cached | Agent tool config, explore agents, skills, verification |
| `memory` | Cached | CLAUDE.md content loaded via `loadMemoryPrompt()` |
| `ant_model_override` | Cached | Internal model-specific prompt suffix |
| `env_info_simple` | Cached | CWD, git, platform, model name, knowledge cutoff |
| `language` | Cached | Language preference (e.g., "Always respond in Japanese") |
| `output_style` | Cached | Custom output style prompt |
| `mcp_instructions` | **VOLATILE** | MCP server instructions (servers connect/disconnect) |
| `scratchpad` | Cached | Scratchpad directory instructions |
| `frc` | Cached | Function Result Clearing instructions |
| `summarize_tool_results` | Cached | Tool result summarization guidance |
| `token_budget` | Cached | Token budget continuation instructions |
| `numeric_length_anchors` | Cached | "≤25 words between tools, ≤100 words final" (ant-only) |

### Why DANGEROUS_uncached Exists

MCP servers can connect or disconnect between turns. If MCP instructions were cached, a newly connected server's instructions wouldn't appear until `/clear`. The `DANGEROUS_` prefix is a code-review signal: "this breaks the prompt cache — make sure the reason is worth it."

---

## 7. Context Injection Points

Beyond the system prompt, context is injected at three additional points.

```mermaid
flowchart TD
    subgraph SystemPrompt["System Prompt (position: 0)"]
        SP["getSystemPrompt() output<br/>↓<br/>buildEffectiveSystemPrompt()"]
    end

    subgraph UserContext["User Context (prepended to messages)"]
        UC["getUserContext()<br/>- gitStatus: branch + status<br/>- date: current date<br/>- currentSessionCosts: cost tracking<br/>- contextPercentage: window usage"]
    end

    subgraph SystemContext["System Context (appended to system prompt)"]
        SC["getSystemContext()<br/>- activeMemory: CLAUDE.md content<br/>- configState: settings reminders"]
    end

    subgraph Attachments["Attachment Messages (injected into conversation)"]
        AT["createAttachmentMessage()<br/>- claudemd_global: ~/.claude/CLAUDE.md<br/>- claudemd_project: .claude/CLAUDE.md<br/>- claudemd_local: .claude/CLAUDE.local.md<br/>- memory: auto-memory files<br/>- skill_discovery: relevant skills<br/>- mcp_instructions_delta: new MCP servers<br/>- deferred_tools_delta: new tools"]
    end

    SP --> UC --> SC --> Attachments

    style SystemPrompt fill:#2980b9,color:#fff
    style UserContext fill:#8e44ad,color:#fff
    style SystemContext fill:#c0392b,color:#fff
    style Attachments fill:#f39c12,color:#333
```

### The `<system-reminder>` Pattern

Context that changes between turns is injected as `<system-reminder>` tags within user messages or tool results. The system prompt tells the model:

> "Tool results and user messages may include `<system-reminder>` tags. Tags contain information from the system. They bear no direct relation to the specific tool results or user messages in which they appear."

This allows the system to piggyback context updates on existing messages without consuming a separate turn.

---

## 8. CLAUDE.md Injection & Memory Loading

CLAUDE.md files are the user's primary way to customize agent behavior per-project.

```mermaid
flowchart TD
    subgraph Sources["CLAUDE.md Sources (Priority Order)"]
        direction TB
        Global["~/.claude/CLAUDE.md<br/>(user-global preferences)"]
        Project["<project>/.claude/CLAUDE.md<br/>(shared team config, git-tracked)"]
        Local["<project>/.claude/CLAUDE.local.md<br/>(personal overrides, git-ignored)"]
        Root["<project>/CLAUDE.md<br/>(project root, legacy location)"]
        Parent["Parent directories' CLAUDE.md<br/>(monorepo workspace config)"]
    end

    subgraph Loading["Loading Pipeline"]
        direction TB
        Discover["getMemoryFiles()<br/>Scan all locations"]
        Dedupe["Deduplicate & order by priority"]
        Attach["createAttachmentMessage()<br/>per file, typed by source"]
        Filter["filterDuplicateMemoryAttachments()<br/>Remove if content unchanged"]
    end

    subgraph Injection["Injection into Conversation"]
        direction TB
        FirstTurn["Turn 1: All CLAUDE.md files<br/>injected as attachment messages"]
        SubsequentTurns["Turn N: Only inject if<br/>content changed (delta detection)"]
    end

    Sources --> Loading --> Injection

    style Sources fill:#2c3e50,color:#fff
    style Loading fill:#27ae60,color:#fff
    style Injection fill:#2980b9,color:#fff
```

### Auto Memory System

Beyond CLAUDE.md, the auto memory system in `~/.claude/projects/<hash>/memory/` stores:
- **User memories** — role, preferences, expertise level
- **Feedback memories** — corrections and confirmed approaches
- **Project memories** — ongoing initiatives, deadlines, decisions
- **Reference memories** — pointers to external systems

The `loadMemoryPrompt()` function reads `MEMORY.md` (the index) and injects it as a system prompt section.

---

## 9. Tool Description Design Principles

How tool descriptions are written has a massive impact on whether the model uses the right tool.

```mermaid
graph TD
    subgraph Principles["Tool Description Design Principles"]
        P1["1. NEGATIVE INSTRUCTIONS<br/>'Do NOT use Bash to run grep'<br/>Prevents common misuse"]
        P2["2. ALTERNATIVE MAPPING<br/>'Use Read instead of cat/head/tail'<br/>Maps old habits to new tools"]
        P3["3. WHEN-TO-USE GUIDANCE<br/>'For broader codebase exploration,<br/>use Agent with Explore type'<br/>Contextual tool selection"]
        P4["4. PARAMETER DOCUMENTATION<br/>JSON Schema with descriptions<br/>on every property"]
        P5["5. CONCURRENCY HINTS<br/>'isConcurrencySafe' flag<br/>Tells orchestrator about parallelism"]
        P6["6. RESULT SIZE BOUNDS<br/>'maxResultSizeChars'<br/>Prevents context flooding"]
    end

    subgraph AntiPatterns["Anti-Patterns Avoided"]
        A1["❌ 'Use this tool to search files'<br/>(too vague — which tool?)"]
        A2["❌ Describing capabilities without context<br/>(model doesn't know when to choose)"]
        A3["❌ No error examples<br/>(model can't self-correct)"]
    end

    Principles --> AntiPatterns

    style Principles fill:#27ae60,color:#fff
    style AntiPatterns fill:#e74c3c,color:#fff
```

### Real Example: The Bash Tool Steering

The system prompt contains this critical instruction:

```
Do NOT use the Bash to run commands when a relevant dedicated tool is provided.
- To read files use Read instead of cat, head, tail, or sed
- To edit files use Edit instead of sed or awk
- To create files use Write instead of cat with heredoc or echo redirection
- To search for files use Glob instead of find or ls
- To search the content of files, use Grep instead of grep or rg
```

This is in the **system prompt**, not the tool description, because:
1. It's a **cross-tool** concern (involves all tools, not just Bash)
2. It needs **high priority** (system prompt is always attended to)
3. It prevents the model's **pre-training bias** toward shell commands

---

## 10. Prompt Caching Economics

The financial impact of prompt caching strategy is substantial.

```mermaid
graph LR
    subgraph NoCaching["Without Caching (Naive)"]
        direction TB
        NC1["Turn 1: 10K tokens system prompt<br/>= 10K input tokens"]
        NC2["Turn 2: 10K system + 5K history<br/>= 15K input tokens"]
        NC3["Turn 10: 10K system + 50K history<br/>= 60K input tokens"]
        NC4["Total: ~310K input tokens"]
        NC1 --> NC2 --> NC3 --> NC4
    end

    subgraph WithCaching["With Cache Strategy"]
        direction TB
        WC1["Turn 1: 8K static (cache WRITE at 1.25x)<br/>+ 2K dynamic = 12K effective"]
        WC2["Turn 2: 8K static (cache READ at 0.1x)<br/>+ 2K dynamic + 5K history = 7.8K effective"]
        WC3["Turn 10: 8K static (cache READ at 0.1x)<br/>+ 2K dynamic + 50K history = 52.8K effective"]
        WC4["Total: ~210K effective tokens<br/>(~32% savings)"]
        WC1 --> WC2 --> WC3 --> WC4
    end

    subgraph GlobalCache["With Global Cache Scope"]
        direction TB
        GC1["Static prefix shared across<br/>ALL users on same model"]
        GC2["Cache READ probability: ~95%+<br/>for popular models"]
        GC3["Effective static cost:<br/>~0.1x on nearly every turn"]
        GC1 --> GC2 --> GC3
    end

    NoCaching -.->|"vs"| WithCaching
    WithCaching -.->|"enhanced by"| GlobalCache

    style NoCaching fill:#e74c3c,color:#fff
    style WithCaching fill:#f39c12,color:#333
    style GlobalCache fill:#27ae60,color:#fff
```

### The 2^N Variant Problem

Every conditional in the static prefix doubles the number of possible cache keys:

```
if (ant) → 2 variants
if (embeddedSearchTools) → 4 variants
if (replMode) → 8 variants
```

This is why session-variant guidance was **moved after** `SYSTEM_PROMPT_DYNAMIC_BOUNDARY`:
- `isForkSubagentEnabled()` reads session type → moved to dynamic
- `isReplModeEnabled()` depends on tools → handled per-path
- `areExplorePlanAgentsEnabled()` is A/B tested → moved to dynamic

The goal: **minimize the number of distinct static prefixes** to maximize global cache hit rate.

---

## 11. Model-Specific Prompt Adaptations

The prompt adapts to different models and user types.

```mermaid
flowchart TD
    subgraph ModelAdaptations["Model-Specific Adaptations"]
        direction TB
        KnowledgeCutoff["Knowledge cutoff date<br/>(varies per model family)"]
        ModelName["Marketing name injection<br/>'You are powered by Claude Opus 4.6'"]
        LatestModels["Latest model family IDs<br/>(for AI app building guidance)"]
        MaxOutput["Max output token defaults<br/>(8K capped → 64K escalated)"]
    end

    subgraph UserType["User Type Variants (ant vs external)"]
        direction TB
        AntOnly["Ant-only additions:<br/>- Thoroughness counterweight<br/>- Comment writing guidance<br/>- False claims mitigation<br/>- Numeric length anchors<br/>- Bug reporting guidance<br/>- Assertiveness counterweight"]
        External["External additions:<br/>- 'Your responses should be<br/>  short and concise'<br/>- Standard help/feedback info"]
    end

    subgraph Undercover["Undercover Mode"]
        direction TB
        UndercoverDesc["Strips ALL model names/IDs<br/>from system prompt to prevent<br/>leaking internal model info<br/>in public commits/PRs"]
    end

    ModelAdaptations --> UserType --> Undercover

    style ModelAdaptations fill:#2980b9,color:#fff
    style UserType fill:#8e44ad,color:#fff
    style Undercover fill:#e74c3c,color:#fff
```

### The `@[MODEL LAUNCH]` Pattern

Throughout `prompts.ts`, you'll find comments like:

```typescript
// @[MODEL LAUNCH]: Update the latest frontier model.
// @[MODEL LAUNCH]: Update the model family IDs below.
// @[MODEL LAUNCH]: Remove this section when we launch numbat.
```

These are search markers for the model launch checklist — when a new model ships, engineers grep for `@[MODEL LAUNCH]` to find every prompt location that needs updating.

---

## 12. The Behavioral Shaping Strategy

The prompt doesn't just instruct — it **shapes behavior** through specific psychological patterns.

```mermaid
graph TD
    subgraph Strategies["Behavioral Shaping Strategies"]
        direction TB
        S1["NEGATIVE EXAMPLES<br/>'Do not reply with just method_name,<br/>instead find the method and modify'<br/><b>Why:</b> Prevents common failure modes"]
        S2["CONCRETE ALTERNATIVES<br/>'Instead of cat, use Read tool'<br/><b>Why:</b> Redirects pre-training biases"]
        S3["PRIORITIZED CONCERNS<br/>'This is CRITICAL to assisting the user'<br/><b>Why:</b> Attention weighting"]
        S4["IDENTITY FRAMING<br/>'You are Claude Code, Anthropic's<br/>official CLI'<br/><b>Why:</b> Sets behavioral baseline"]
        S5["SAFETY THROUGH EXAMPLES<br/>'rm -rf, DROP TABLE, force-push'<br/><b>Why:</b> Concrete threat recognition"]
        S6["META-INSTRUCTIONS<br/>'Do not use a colon before tool calls'<br/><b>Why:</b> Fixes rendering artifacts"]
        S7["ANTI-OVERENGINEERING<br/>'Three similar lines of code is better<br/>than a premature abstraction'<br/><b>Why:</b> Fights model tendency to generalize"]
    end

    subgraph Results["Observed Results"]
        R1["Model uses Read instead of cat ~98%"]
        R2["Destructive operations always<br/>ask for confirmation"]
        R3["Code changes are minimal and focused"]
        R4["Tool calls don't have trailing colons"]
    end

    Strategies --> Results

    style Strategies fill:#2c3e50,color:#fff
    style Results fill:#27ae60,color:#fff
```

### The "Measure Twice, Cut Once" Principle

The safety section ends with:

> "Follow both the spirit and letter of these instructions - measure twice, cut once."

This is a behavioral anchoring technique — it gives the model a memorable metaphor to recall when making decisions about risky actions, even when the specific instructions don't cover the exact scenario.

### The FRC (Function Result Clearing) Pattern

Large tool results are automatically cleared from the conversation history and replaced with `[Old tool result content cleared]`. The system prompt tells the model:

> "When working with tool results, write down any important information you might need later in your response, as the original tool result may be cleared later."

This trains the model to extract and summarize key information immediately, rather than relying on being able to re-read tool results later — a critical behavior for long conversations.
