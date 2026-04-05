# Multi-Agent Orchestration & Communication

> How Claude Code spawns, manages, communicates with, and orchestrates multiple AI agents working in parallel — from simple subagents to full team swarms. Every diagram is a Mermaid diagram you can render in any Markdown viewer.

---

## Table of Contents

1. [Why Multi-Agent Architecture](#1-why-multi-agent-architecture)
2. [The Agent Hierarchy](#2-the-agent-hierarchy)
3. [Subagent Spawning & Lifecycle](#3-subagent-spawning--lifecycle)
4. [Fork Agents: Background Workers](#4-fork-agents-background-workers)
5. [Built-in Agent Types](#5-built-in-agent-types)
6. [Inter-Agent Communication](#6-inter-agent-communication)
7. [Prompt Cache Sharing](#7-prompt-cache-sharing)
8. [The Swarm/Team System](#8-the-swarmteam-system)
9. [Agent Permission Inheritance](#9-agent-permission-inheritance)
10. [Agent Context Isolation](#10-agent-context-isolation)
11. [Coordinator Mode](#11-coordinator-mode)

---

## 1. Why Multi-Agent Architecture

A single agent has a finite context window and attention budget. Multi-agent architecture solves three problems:

```mermaid
graph TB
    subgraph Problems["Problems with Single Agent"]
        P1["CONTEXT POLLUTION<br/>Research results fill the window<br/>with data the model doesn't need<br/>for the next step"]
        P2["SERIAL BOTTLENECK<br/>Can't read 10 files in parallel<br/>while also writing code"]
        P3["TASK COMPLEXITY<br/>Complex tasks exceed a single<br/>model's reasoning capacity"]
    end

    subgraph Solutions["Multi-Agent Solutions"]
        S1["CONTEXT ISOLATION<br/>Subagent's research stays in its<br/>own context → parent gets summary"]
        S2["PARALLELISM<br/>Multiple agents work simultaneously<br/>on independent subtasks"]
        S3["SPECIALIZATION<br/>Explore agent for research,<br/>Review agent for verification,<br/>Build agent for implementation"]
    end

    P1 --> S1
    P2 --> S2
    P3 --> S3

    style Problems fill:#e74c3c,color:#fff
    style Solutions fill:#27ae60,color:#fff
```

---

## 2. The Agent Hierarchy

```mermaid
graph TD
    subgraph Hierarchy["Agent Hierarchy"]
        Main["MAIN SESSION<br/>(REPL.tsx → query loop)<br/>- Full UI access<br/>- All permissions<br/>- User interaction"]

        subgraph Subagents["Subagents (AgentTool)"]
            Explore["Explore Agent<br/>- Read-only tools<br/>- Fast model (Haiku)<br/>- Returns summary"]
            Plan["Plan Agent<br/>- Read-only tools<br/>- Architectural analysis<br/>- Returns plan"]
            Custom["Custom Agent<br/>- .claude/agents/*.md<br/>- Custom system prompt<br/>- Configurable tools"]
            Plugin["Plugin Agent<br/>- plugin.json defined<br/>- Scoped tools<br/>- Custom prompts"]
        end

        subgraph Forked["Fork Agents"]
            SessionMem["Session Memory Fork<br/>- Summarizes for memory<br/>- Shares parent cache"]
            CompactFork["Compact Fork<br/>- Summarizes conversation<br/>- Shares parent cache"]
            PromptSuggest["Prompt Suggestion Fork<br/>- Suggests next steps<br/>- Background execution"]
        end

        subgraph Team["Team / Swarm"]
            Leader["Leader<br/>- Coordinates work<br/>- Assigns tasks"]
            Teammate1["Teammate 1<br/>- Independent context<br/>- In-process or separate"]
            Teammate2["Teammate 2<br/>- Independent context<br/>- Reports progress"]
        end

        Main --> Subagents
        Main --> Forked
        Main --> Team
        Leader --> Teammate1
        Leader --> Teammate2
    end

    style Main fill:#2c3e50,color:#fff
    style Subagents fill:#2980b9,color:#fff
    style Forked fill:#8e44ad,color:#fff
    style Team fill:#27ae60,color:#fff
```

---

## 3. Subagent Spawning & Lifecycle

Subagents are spawned via the `AgentTool` and managed through the `runAgent()` function.

```mermaid
sequenceDiagram
    participant Parent as Parent Query Loop
    participant AgentTool as AgentTool.call()
    participant RunAgent as runAgent()
    participant SubQuery as Subagent query()
    participant API as Claude API

    Parent->>AgentTool: tool_use: { prompt, subagent_type }
    AgentTool->>RunAgent: Create subagent context

    Note over RunAgent: Setup Phase
    RunAgent->>RunAgent: 1. Create child abort controller
    RunAgent->>RunAgent: 2. Clone file state cache
    RunAgent->>RunAgent: 3. Build system prompt (agent-specific)
    RunAgent->>RunAgent: 4. Filter available tools
    RunAgent->>RunAgent: 5. Connect MCP servers (if any)
    RunAgent->>RunAgent: 6. Create agent transcript directory

    Note over RunAgent: Execution Phase
    RunAgent->>SubQuery: Start query loop
    loop Agentic Loop
        SubQuery->>API: Send messages
        API-->>SubQuery: Stream response
        SubQuery->>SubQuery: Execute tools
        SubQuery-->>Parent: Yield progress messages
    end

    Note over RunAgent: Completion
    SubQuery-->>RunAgent: Terminal state (end_turn)
    RunAgent->>RunAgent: Extract final text response
    RunAgent->>RunAgent: Record transcript
    RunAgent->>RunAgent: Log analytics (cost, turns, duration)
    RunAgent-->>AgentTool: Return summary text
    AgentTool-->>Parent: tool_result: "Agent summary..."
```

### Subagent Configuration

| Property | Source | Effect |
|---|---|---|
| `subagent_type` | LLM parameter | Selects built-in or custom agent |
| `model` | Agent definition or LLM parameter | Model for subagent (can differ from parent) |
| `prompt` | LLM parameter | Initial message to subagent |
| `tools` | Agent definition `allowedTools` | Restricts available tools |
| `isolation` | LLM parameter (`"worktree"`) | Creates git worktree for agent |
| `run_in_background` | LLM parameter | Returns immediately, notifies on completion |
| `maxTurns` | Agent definition or default | Limits agentic loop iterations |

---

## 4. Fork Agents: Background Workers

Fork agents are lightweight query loops that share the parent's prompt cache.

```mermaid
flowchart TD
    subgraph Parent["Parent Session"]
        ParentPrompt["System Prompt + Messages<br/>(cache key established)"]
        CacheSafe["saveCacheSafeParams({<br/>  systemPrompt,<br/>  userContext,<br/>  systemContext,<br/>  toolUseContext,<br/>  forkContextMessages<br/>})"]
    end

    subgraph Fork["Forked Agent"]
        SharedCache["Uses parent's CacheSafeParams<br/>→ EXACT same cache key prefix"]
        NewMessages["Appends fork-specific messages<br/>(prompt, instructions)"]
        IndependentLoop["Runs independent query loop<br/>with isolated mutable state"]
    end

    subgraph Benefits["Cache Sharing Benefits"]
        B1["System prompt: cache HIT (free)"]
        B2["Message prefix: cache HIT (free)"]
        B3["Only fork-specific content costs new tokens"]
        B4["~70-90% input token savings"]
    end

    Parent --> Fork --> Benefits

    style Parent fill:#2980b9,color:#fff
    style Fork fill:#8e44ad,color:#fff
    style Benefits fill:#27ae60,color:#fff
```

### Fork Agent Use Cases

```mermaid
graph LR
    subgraph UseCases["Fork Agent Use Cases"]
        direction TB
        UC1["SESSION MEMORY COMPACT<br/>After conversation compaction,<br/>extract key facts for persistence<br/>Label: 'session_memory'"]
        UC2["PROMPT SUGGESTION<br/>After user turn completes,<br/>suggest next steps<br/>Label: 'prompt_suggestion'"]
        UC3["POST-TURN SUMMARY<br/>Summarize what was accomplished<br/>for the status bar<br/>Label: 'post_turn_summary'"]
        UC4["COMPACT<br/>Summarize full conversation<br/>when context is too large<br/>Label: 'compact'"]
    end

    style UseCases fill:#2c3e50,color:#fff
```

---

## 5. Built-in Agent Types

Claude Code provides specialized built-in agents optimized for common tasks.

```mermaid
graph TD
    subgraph BuiltIn["Built-in Agent Types"]
        direction TB
        Explore["EXPLORE AGENT<br/>Purpose: Codebase research<br/>Model: Haiku (fast, cheap)<br/>Tools: Read, Glob, Grep (read-only)<br/>Trigger: 'subagent_type=Explore'<br/>Guidance: Quick/medium/thorough"]
        Plan2["PLAN AGENT<br/>Purpose: Architecture design<br/>Model: Default (capable)<br/>Tools: Read, Glob, Grep (read-only)<br/>Trigger: 'subagent_type=Plan'<br/>Output: Implementation plan"]
        Verify["VERIFICATION AGENT<br/>Purpose: Adversarial code review<br/>Model: Default (capable)<br/>Tools: All read tools + Bash (tests)<br/>Trigger: 'subagent_type=verification'<br/>Output: PASS/FAIL/PARTIAL verdict"]
    end

    subgraph Custom["Custom Agent Types"]
        direction TB
        UserDefined[".claude/agents/my-agent.md<br/>---<br/>name: my-agent<br/>description: Does X<br/>model: haiku<br/>allowedTools: [Read, Grep]<br/>---<br/>Custom system prompt here"]
        PluginDefined["plugin.json agents[]<br/>Defined by installed plugins<br/>e.g., feature-dev:code-reviewer"]
    end

    style BuiltIn fill:#2980b9,color:#fff
    style Custom fill:#27ae60,color:#fff
```

### Agent Selection Logic

```mermaid
flowchart TD
    Request["Agent tool_use:<br/>{ subagent_type: 'X' }"]
    IsBuiltIn{"Is 'X' a<br/>built-in type?"}
    BuiltIn["Load built-in agent definition"]
    IsCustom{"Is 'X' in<br/>.claude/agents/?"}
    Custom["Load custom agent .md file"]
    IsPlugin{"Is 'X' from<br/>a plugin?"}
    Plugin["Load plugin agent definition"]
    Default["Use general-purpose<br/>agent (fork)"]

    Request --> IsBuiltIn
    IsBuiltIn -->|"Yes"| BuiltIn
    IsBuiltIn -->|"No"| IsCustom
    IsCustom -->|"Yes"| Custom
    IsCustom -->|"No"| IsPlugin
    IsPlugin -->|"Yes"| Plugin
    IsPlugin -->|"No"| Default

    style BuiltIn fill:#2980b9,color:#fff
    style Custom fill:#27ae60,color:#fff
    style Plugin fill:#8e44ad,color:#fff
    style Default fill:#f39c12,color:#333
```

---

## 6. Inter-Agent Communication

Agents communicate through several mechanisms.

```mermaid
flowchart TD
    subgraph Mechanisms["Communication Mechanisms"]
        direction TB

        M1["TOOL RESULT<br/>(Subagent → Parent)<br/>Agent returns text summary<br/>as tool_result to parent"]

        M2["SEND MESSAGE<br/>(Agent ↔ Agent)<br/>SendMessageTool delivers<br/>messages between named agents"]

        M3["PROGRESS MESSAGES<br/>(Subagent → Parent UI)<br/>Streaming progress updates<br/>displayed in parent's terminal"]

        M4["TASK SYSTEM<br/>(Agent ↔ Agent)<br/>Shared task list for<br/>coordination and tracking"]

        M5["FILE SYSTEM<br/>(Agent ↔ Agent)<br/>Agents read/write shared files<br/>(plans, memory, code)"]
    end

    subgraph Patterns["Communication Patterns"]
        direction TB
        P1["REQUEST-RESPONSE<br/>Parent spawns subagent → waits for result"]
        P2["FIRE-AND-FORGET<br/>Parent spawns background agent → continues"]
        P3["PUB-SUB<br/>Agent publishes progress → parent UI subscribes"]
        P4["SHARED STATE<br/>Agents coordinate through task list + files"]
    end

    Mechanisms --> Patterns

    style Mechanisms fill:#2980b9,color:#fff
    style Patterns fill:#27ae60,color:#fff
```

### The SendMessage Tool

```mermaid
sequenceDiagram
    participant AgentA as Agent A (named: "researcher")
    participant SendMsg as SendMessageTool
    participant AgentB as Agent B (named: "implementer")

    AgentA->>SendMsg: { to: "implementer",<br/>message: "Found the bug in auth.ts:42" }
    SendMsg->>AgentB: Inject user message<br/>into agent B's context
    Note over AgentB: Agent B processes<br/>the message in its<br/>next iteration
    AgentB-->>SendMsg: Acknowledgment
    SendMsg-->>AgentA: "Message delivered"
```

---

## 7. Prompt Cache Sharing

The most impactful optimization: subagents share the parent's prompt cache.

```mermaid
flowchart TD
    subgraph CacheKey["Prompt Cache Key Components"]
        direction TB
        K1["System Prompt (identical)"]
        K2["Tools array (identical or subset)"]
        K3["Model (may differ)"]
        K4["Thinking config (derived from model)"]
        K5["Message prefix (parent's history)"]
    end

    subgraph Sharing["Cache Sharing Strategy"]
        direction TB
        ForkAgent["Fork Agents:<br/>Use CacheSafeParams directly<br/>→ 100% cache sharing"]
        SubAgent["Subagents with parent context:<br/>forkContextMessages = parent messages<br/>→ Prefix cache sharing"]
        Independent["Independent agents:<br/>Different system prompt<br/>→ No cache sharing"]
    end

    subgraph Warning["Cache Invalidation Risks"]
        direction TB
        W1["⚠️ Different maxOutputTokens<br/>→ Changes budget_tokens<br/>→ Different thinking config<br/>→ CACHE MISS"]
        W2["⚠️ DANGEROUS_uncached sections<br/>→ Dynamic system prompt content<br/>→ May break prefix match"]
        W3["⚠️ Different model<br/>→ Different cache namespace<br/>→ No sharing possible"]
    end

    CacheKey --> Sharing --> Warning

    style CacheKey fill:#2980b9,color:#fff
    style Sharing fill:#27ae60,color:#fff
    style Warning fill:#e74c3c,color:#fff
```

---

## 8. The Swarm/Team System

The swarm system enables multiple agents working as a coordinated team.

```mermaid
flowchart TD
    subgraph Team["Team Architecture"]
        Leader["LEADER<br/>- Coordinates work<br/>- Assigns tasks to teammates<br/>- Has full UI access<br/>- Manages permissions"]

        TM1["TEAMMATE 1<br/>- In-process runner<br/>- AsyncLocalStorage isolation<br/>- Progress tracking<br/>- Reports to leader"]

        TM2["TEAMMATE 2<br/>- In-process runner<br/>- Independent context<br/>- Can use SendMessage<br/>- Can create/update tasks"]
    end

    subgraph Coordination["Coordination Mechanisms"]
        TaskList["Shared Task List<br/>TaskCreate/Update/List/Get tools"]
        SendMsg2["SendMessage<br/>Direct agent-to-agent messaging"]
        PermSync["Permission Synchronization<br/>Mailbox-based async permission flow"]
        Progress["Progress Tracking<br/>Activity descriptions + status updates"]
    end

    subgraph Lifecycle["Teammate Lifecycle"]
        Create["TeamCreate tool<br/>→ Spawn teammate"]
        Run["In-process runner<br/>→ runWithTeammateContext()"]
        Monitor["Leader monitors progress<br/>→ getProgressUpdate()"]
        Complete["Teammate completes<br/>→ Idle notification to leader"]
        Cleanup["Cleanup<br/>→ evictTaskOutput(), kill shells"]
    end

    Leader --> TM1
    Leader --> TM2
    Team --> Coordination
    Coordination --> Lifecycle

    style Leader fill:#2c3e50,color:#fff
    style TM1 fill:#2980b9,color:#fff
    style TM2 fill:#2980b9,color:#fff
    style Coordination fill:#f39c12,color:#333
    style Lifecycle fill:#27ae60,color:#fff
```

### In-Process vs Separate Process

| Aspect | In-Process Teammate | Separate Process |
|---|---|---|
| Context isolation | AsyncLocalStorage | Process boundary |
| Communication | Direct function calls | UDS messaging |
| Resource sharing | Shared memory | File-based |
| Abort propagation | Direct AbortController | Signal-based |
| UI integration | Shared Ink renderer | Separate output |

---

## 9. Agent Permission Inheritance

Subagents inherit permissions from their parent with **restriction only** — they can never have MORE permissions.

```mermaid
flowchart TD
    subgraph Parent["Parent Permissions"]
        PM["Mode: acceptEdits"]
        PA["Allow: Bash(npm *), Read(*), Edit(*)"]
        PD["Deny: Bash(rm -rf *)"]
    end

    subgraph SubAgent["Subagent Permissions"]
        SM["Mode: acceptEdits (inherited)"]
        SA["Allow: Read(*), Glob(*)<br/>(subset — no Bash, no Edit)"]
        SD["Deny: Bash(rm -rf *) (inherited)<br/>+ additional restrictions"]
    end

    subgraph Rules["Inheritance Rules"]
        R1["1. Mode can only be EQUAL or MORE restrictive"]
        R2["2. Allow rules can only be SUBSET of parent"]
        R3["3. Deny rules are UNION of parent + agent"]
        R4["4. shouldAvoidPermissionPrompts = true<br/>(background agents can't show UI)"]
    end

    Parent --> SubAgent
    SubAgent --> Rules

    style Parent fill:#27ae60,color:#fff
    style SubAgent fill:#2980b9,color:#fff
    style Rules fill:#f39c12,color:#333
```

---

## 10. Agent Context Isolation

Each agent runs with isolated context to prevent interference.

```mermaid
flowchart LR
    subgraph Shared["Shared Across All Agents"]
        FS["Filesystem<br/>(same git repo)"]
        Git["Git state<br/>(unless worktree)"]
        MCPServers["MCP server connections"]
    end

    subgraph Isolated["Isolated Per Agent"]
        Messages2["messages[] array<br/>(independent conversation)"]
        FileCache["File state cache<br/>(cloned from parent)"]
        AbortCtrl["Abort controller<br/>(child of parent's)"]
        AgentID["Agent ID<br/>(unique identifier)"]
        Transcript["Transcript storage<br/>(separate directory)"]
        DenialState["Denial tracking state<br/>(fresh per agent)"]
        ContentReplace["Content replacement state<br/>(cloned from parent)"]
    end

    subgraph IsolationMechanism["Isolation Mechanism"]
        Clone["createSubagentContext()<br/>Deep clone mutable state<br/>Share immutable references"]
        AsyncLocal["runWithAgentContext()<br/>AsyncLocalStorage for<br/>agent-specific globals"]
    end

    Shared --> IsolationMechanism
    Isolated --> IsolationMechanism

    style Shared fill:#f39c12,color:#333
    style Isolated fill:#2980b9,color:#fff
    style IsolationMechanism fill:#27ae60,color:#fff
```

---

## 11. Coordinator Mode

Coordinator mode (feature flag: `COORDINATOR_MODE`) is a specialized multi-agent pattern where a coordinator agent manages the overall workflow.

```mermaid
flowchart TD
    subgraph Coordinator["Coordinator Agent"]
        CoordPrompt["Special coordinator system prompt<br/>(replaces default prompt)"]
        CoordRole["Role: Plan, delegate, verify<br/>Does NOT write code directly"]
    end

    subgraph Workers["Worker Agents"]
        W1["Worker 1: Implementation<br/>Writes code, runs tests"]
        W2["Worker 2: Review<br/>Reviews changes, suggests fixes"]
        W3["Worker 3: Testing<br/>Writes/runs test suites"]
    end

    subgraph Flow["Coordination Flow"]
        F1["1. User gives high-level task"]
        F2["2. Coordinator creates plan"]
        F3["3. Coordinator spawns workers"]
        F4["4. Workers execute in parallel"]
        F5["5. Workers report progress"]
        F6["6. Coordinator verifies results"]
        F7["7. Coordinator reports to user"]
    end

    Coordinator --> Workers
    Flow --> Coordinator

    style Coordinator fill:#2c3e50,color:#fff
    style Workers fill:#2980b9,color:#fff
    style Flow fill:#27ae60,color:#fff
```

### When to Use Multi-Agent vs Single Agent

```mermaid
graph TD
    subgraph Decision["Decision Matrix"]
        direction TB
        SingleAgent["SINGLE AGENT when:<br/>- Task is straightforward<br/>- Few files involved<br/>- Sequential steps<br/>- User wants to see progress"]
        SubAgents["SUBAGENTS when:<br/>- Research needed (Explore)<br/>- Architecture planning (Plan)<br/>- Independent verification<br/>- Context would get too large"]
        TeamSwarm["TEAM/SWARM when:<br/>- Multiple independent tasks<br/>- Different expertise needed<br/>- Parallel workstreams<br/>- Large codebase changes"]
        Coordinator2["COORDINATOR when:<br/>- Complex multi-step projects<br/>- Need orchestration logic<br/>- Quality gates between steps<br/>- Human oversight important"]
    end

    style SingleAgent fill:#27ae60,color:#fff
    style SubAgents fill:#3498db,color:#fff
    style TeamSwarm fill:#8e44ad,color:#fff
    style Coordinator2 fill:#e74c3c,color:#fff
```
