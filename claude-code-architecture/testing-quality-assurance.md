# Testing & Quality Assurance Strategy

> How to test an AI coding agent that uses LLMs, executes shell commands, modifies files, and interacts with external APIs — the unique QA challenges and patterns. Every diagram is a Mermaid diagram you can render in any Markdown viewer.

---

## Table of Contents

1. [Why Testing AI Agents Is Uniquely Hard](#1-why-testing-ai-agents-is-uniquely-hard)
2. [The Testing Pyramid for AI Agents](#2-the-testing-pyramid-for-ai-agents)
3. [Unit Testing Patterns](#3-unit-testing-patterns)
4. [Integration Testing Patterns](#4-integration-testing-patterns)
5. [The VCR Pattern: Recording API Interactions](#5-the-vcr-pattern-recording-api-interactions)
6. [Permission & Security Testing](#6-permission--security-testing)
7. [Prompt Testing & Evaluation](#7-prompt-testing--evaluation)
8. [Feature Flag Testing](#8-feature-flag-testing)
9. [Model Launch Validation](#9-model-launch-validation)
10. [Quality Gates & CI Pipeline](#10-quality-gates--ci-pipeline)

---

## 1. Why Testing AI Agents Is Uniquely Hard

Traditional software has deterministic outputs. AI agents have **non-deterministic** outputs from LLMs, **side effects** on the filesystem, and **external dependencies** on APIs.

```mermaid
graph TB
    subgraph Traditional["Traditional Software Testing"]
        T1["Deterministic: same input → same output"]
        T2["Isolated: mock external deps"]
        T3["Fast: milliseconds per test"]
        T4["Complete: full code coverage possible"]
    end

    subgraph AIAgent["AI Agent Testing"]
        A1["Non-deterministic: LLM outputs vary"]
        A2["Side effects: file writes, shell commands"]
        A3["Slow: API calls take seconds"]
        A4["Incomplete: can't cover all model behaviors"]
        A5["Expensive: each test costs real money"]
        A6["Evolving: model updates change behavior"]
    end

    subgraph Strategy["Claude Code's Strategy"]
        S1["Test the HARNESS, not the model"]
        S2["Mock the API boundary"]
        S3["Test tool behavior deterministically"]
        S4["Use VCR recordings for integration tests"]
        S5["Eval suites for prompt changes"]
        S6["Feature flags for gradual rollout"]
    end

    Traditional -.->|"challenges"| AIAgent
    AIAgent -.->|"addressed by"| Strategy

    style Traditional fill:#27ae60,color:#fff
    style AIAgent fill:#e74c3c,color:#fff
    style Strategy fill:#2980b9,color:#fff
```

### The Core Testing Philosophy

> **Test the harness, not the model.** The model's behavior is tested through evals. The harness (tool execution, permission checks, context management, state transitions) is tested through unit and integration tests.

---

## 2. The Testing Pyramid for AI Agents

```mermaid
graph TD
    subgraph Pyramid["Testing Pyramid"]
        direction TB
        Evals["EVALS (top)<br/>End-to-end prompt evaluation<br/>- Does the model choose the right tool?<br/>- Does it produce correct code?<br/>- Does it follow safety instructions?<br/><b>Slow, expensive, non-deterministic</b>"]
        Integration["INTEGRATION TESTS (middle)<br/>Multi-component interaction<br/>- Query loop with mocked API<br/>- Tool execution with real filesystem<br/>- Permission flow end-to-end<br/><b>Medium speed, deterministic</b>"]
        Unit["UNIT TESTS (base)<br/>Individual function testing<br/>- Token counting logic<br/>- Permission resolution<br/>- Path validation<br/>- Message normalization<br/><b>Fast, deterministic, comprehensive</b>"]
    end

    style Evals fill:#e74c3c,color:#fff
    style Integration fill:#f39c12,color:#333
    style Unit fill:#27ae60,color:#fff
```

---

## 3. Unit Testing Patterns

Claude Code's codebase uses several distinct unit testing patterns.

```mermaid
flowchart TD
    subgraph Patterns["Unit Test Patterns"]
        direction TB

        P1["PURE FUNCTION TESTS<br/>Token counting, cost calculation,<br/>path validation, format utilities<br/>→ Standard input/output assertions"]

        P2["SCHEMA VALIDATION TESTS<br/>Tool input schemas (Zod)<br/>→ Valid input passes, invalid rejects<br/>→ Error messages are actionable"]

        P3["STATE TRANSITION TESTS<br/>Permission resolution, abort propagation<br/>→ Given state X, action Y → state Z"]

        P4["MOCK BOUNDARY TESTS<br/>API client, filesystem, shell<br/>→ Mock at the I/O boundary<br/>→ Test business logic in isolation"]
    end

    subgraph Examples["Concrete Examples"]
        direction TB
        E1["roughTokenCountEstimation('hello world')<br/>→ expect(result).toBe(3)"]
        E2["bashTool.inputSchema.safeParse({ command: '' })<br/>→ expect(result.success).toBe(false)"]
        E3["checkPermission(tool, input, 'default', rules)<br/>→ expect(result).toBe('ask')"]
        E4["calculateUSDCost('opus-4-6', usage)<br/>→ expect(cost).toBeCloseTo(0.05)"]
    end

    Patterns --> Examples

    style Patterns fill:#2980b9,color:#fff
    style Examples fill:#27ae60,color:#fff
```

### What's Testable Without the Model

| Component | Testability | How |
|---|---|---|
| Token counting | Fully deterministic | Pure function tests |
| Cost calculation | Fully deterministic | Pure function tests |
| Permission resolution | Fully deterministic | State transition tests |
| Path validation | Fully deterministic | Pure function tests |
| Message normalization | Fully deterministic | Transform tests |
| Schema validation | Fully deterministic | Zod parse tests |
| Tool orchestration | Deterministic with mocks | Mock tool implementations |
| Context management | Deterministic with mocks | Mock token counts |
| Session persistence | Deterministic with temp dirs | Filesystem integration |

---

## 4. Integration Testing Patterns

Integration tests verify multiple components working together.

```mermaid
flowchart LR
    subgraph Setup["Test Setup"]
        direction TB
        MockAPI["Mock Anthropic API<br/>(return canned responses)"]
        TempDir["Temp directory<br/>(isolated filesystem)"]
        MockMCP["Mock MCP servers<br/>(controllable tools)"]
    end

    subgraph Test["Test Execution"]
        direction TB
        Scenario["Create scenario:<br/>1. Set up files<br/>2. Configure permissions<br/>3. Inject user message"]
        RunQuery["Run query loop<br/>(with mocked API)"]
        Verify["Verify:<br/>- Correct tools called<br/>- Files modified correctly<br/>- Messages in right order<br/>- State transitions correct"]
    end

    subgraph Cleanup["Cleanup"]
        direction TB
        RemoveTemp["Remove temp directory"]
        ResetState["Reset global state"]
        VerifyCost["Verify cost tracking"]
    end

    Setup --> Test --> Cleanup

    style Setup fill:#2980b9,color:#fff
    style Test fill:#27ae60,color:#fff
    style Cleanup fill:#f39c12,color:#333
```

### The Query Loop Integration Test Pattern

```mermaid
sequenceDiagram
    participant Test as Test Runner
    participant Query as query()
    participant MockAPI as Mock API
    participant Tool as Real Tool

    Test->>Query: Start with messages + mock context
    Query->>MockAPI: Send request
    MockAPI-->>Query: Return canned response<br/>(text + tool_use blocks)
    Query->>Tool: Execute tool with real input
    Tool-->>Query: Real tool result
    Query->>Test: Yield messages + events
    Test->>Test: Assert message sequence
    Test->>Test: Assert tool was called correctly
    Test->>Test: Assert state transitions
```

---

## 5. The VCR Pattern: Recording API Interactions

Claude Code uses a VCR (Video Cassette Recorder) pattern to record and replay API interactions.

```mermaid
flowchart TD
    subgraph Record["RECORD Mode"]
        R1["Run test against REAL API"]
        R2["Intercept all API requests/responses"]
        R3["Save to cassette file (JSON)"]
        R4["Cassette contains:<br/>- Request params<br/>- Response body<br/>- Token counts<br/>- Timing"]
    end

    subgraph Replay["REPLAY Mode (CI)"]
        P1["Load cassette file"]
        P2["Match incoming request to recording"]
        P3["Return recorded response"]
        P4["No real API call needed"]
        P5["Deterministic, fast, free"]
    end

    subgraph TokenVCR["Token Count VCR"]
        T1["withTokenCountVCR()"]
        T2["Record: actual count_tokens API call"]
        T3["Replay: return cached count"]
        T4["Avoids expensive token counting<br/>in CI tests"]
    end

    Record --> Replay
    Record --> TokenVCR

    style Record fill:#e74c3c,color:#fff
    style Replay fill:#27ae60,color:#fff
    style TokenVCR fill:#2980b9,color:#fff
```

### When to Re-record

- Model version changes (responses may differ)
- Tool schema changes (request format changes)
- Prompt changes (different model behavior expected)

---

## 6. Permission & Security Testing

Permission testing is critical — a bug means the AI can execute dangerous commands.

```mermaid
flowchart TD
    subgraph TestCases["Permission Test Categories"]
        direction TB
        TC1["MODE TESTS<br/>- default: asks for dangerous ops<br/>- acceptEdits: allows file writes<br/>- bypassPermissions: allows everything<br/>- plan: blocks all writes"]
        TC2["RULE TESTS<br/>- alwaysAllow: specific tool+pattern<br/>- alwaysDeny: blocks specific patterns<br/>- alwaysAsk: overrides allow rules"]
        TC3["HOOK TESTS<br/>- PreToolUse hooks can block<br/>- PostToolUse hooks can log<br/>- Hook errors don't crash agent"]
        TC4["ESCALATION TESTS<br/>- Subagent can't exceed parent perms<br/>- Worktree isolation enforced<br/>- File path restrictions respected"]
    end

    subgraph Dangerous["Dangerous Command Detection"]
        DC1["rm -rf /"]
        DC2["DROP TABLE users"]
        DC3["git push --force"]
        DC4["curl | bash"]
        DC5["chmod 777 /etc/passwd"]
    end

    subgraph Verification["Test Verification"]
        V1["Assert: permission dialog shown"]
        V2["Assert: command NOT executed"]
        V3["Assert: error message returned to model"]
        V4["Assert: audit log records attempt"]
    end

    TestCases --> Dangerous --> Verification

    style TestCases fill:#2980b9,color:#fff
    style Dangerous fill:#e74c3c,color:#fff
    style Verification fill:#27ae60,color:#fff
```

### The YOLO Classifier Testing

The YOLO (You Only Live Once) classifier automatically approves safe commands in `bypassPermissions` mode but still blocks truly dangerous ones. Testing this requires:

```mermaid
graph LR
    subgraph SafeCommands["Should Auto-Approve"]
        S1["ls -la"]
        S2["git status"]
        S3["npm test"]
        S4["cat README.md"]
    end

    subgraph DangerousCommands["Should Still Block"]
        D1["rm -rf /"]
        D2["curl url | sh"]
        D3["sudo anything"]
        D4["> /etc/hosts"]
    end

    SafeCommands -->|"classify"| Approve["AUTO-APPROVE"]
    DangerousCommands -->|"classify"| Block["STILL BLOCK"]

    style SafeCommands fill:#27ae60,color:#fff
    style DangerousCommands fill:#e74c3c,color:#fff
```

---

## 7. Prompt Testing & Evaluation

Prompt changes are the highest-risk changes. They affect all users and can't be easily rolled back.

```mermaid
flowchart TD
    subgraph EvalPipeline["Prompt Evaluation Pipeline"]
        Change["Prompt change PR"]
        EvalSuite["Run eval suite:<br/>- Tool selection accuracy<br/>- Safety compliance<br/>- Output quality<br/>- Cost impact"]
        ABTest["A/B test (GrowthBook):<br/>- Rolled out to % of users<br/>- Monitor metrics"]
        FullRollout["Full rollout<br/>(if metrics improve)"]
    end

    subgraph Metrics["Key Metrics"]
        ToolAccuracy["Tool Selection Accuracy<br/>Does model use Read instead of cat?"]
        SafetyRate["Safety Compliance Rate<br/>Does model ask before rm -rf?"]
        FalseClaimsRate["False Claims Rate<br/>Does model claim tests pass when they fail?"]
        OutputTokens["Output Token Efficiency<br/>How concise are responses?"]
        UserSatisfaction["User Satisfaction<br/>Thumbs up/down on responses"]
    end

    subgraph Guards["Quality Guards"]
        G1["@[MODEL LAUNCH] markers<br/>→ Grep-searchable prompt update points"]
        G2["USER_TYPE gates<br/>→ Ant-only changes tested internally first"]
        G3["Feature flags<br/>→ Gradual rollout with kill switch"]
    end

    EvalPipeline --> Metrics
    Metrics --> Guards

    style EvalPipeline fill:#2980b9,color:#fff
    style Metrics fill:#f39c12,color:#333
    style Guards fill:#27ae60,color:#fff
```

---

## 8. Feature Flag Testing

Feature flags enable testing new behavior in production without full rollout.

```mermaid
flowchart LR
    subgraph Flags["Feature Flags (bun:bundle)"]
        direction TB
        F1["REACTIVE_COMPACT"]
        F2["CONTEXT_COLLAPSE"]
        F3["HISTORY_SNIP"]
        F4["CACHED_MICROCOMPACT"]
        F5["TOKEN_BUDGET"]
        F6["EXPERIMENTAL_SKILL_SEARCH"]
        F7["COORDINATOR_MODE"]
        F8["PROACTIVE"]
    end

    subgraph BuildVariants["Build Variants"]
        Internal["Internal Build<br/>(all flags available)"]
        External["External/OSS Build<br/>(most flags = false → DCE)"]
    end

    subgraph Testing["Testing Strategy"]
        UnitBoth["Unit tests: both flag states"]
        IntegrationDefault["Integration: default flags"]
        EvalVariants["Evals: flag variants measured"]
    end

    Flags --> BuildVariants --> Testing

    style Flags fill:#8e44ad,color:#fff
    style BuildVariants fill:#2980b9,color:#fff
    style Testing fill:#27ae60,color:#fff
```

### Dead Code Elimination Testing

When `feature('HISTORY_SNIP')` is `false` at build time, the entire snip module is eliminated. Tests must verify:
1. The feature works correctly when enabled
2. The system works correctly when disabled (no runtime errors from missing modules)
3. The DCE actually removes the code (bundle size verification)

---

## 9. Model Launch Validation

New model launches require systematic validation.

```mermaid
flowchart TD
    subgraph Checklist["@[MODEL LAUNCH] Checklist"]
        direction TB
        C1["1. Update FRONTIER_MODEL_NAME"]
        C2["2. Update MODEL_COSTS pricing tier"]
        C3["3. Update model family IDs"]
        C4["4. Update knowledge cutoff date"]
        C5["5. Update 1M context support check"]
        C6["6. Update marketing name mapping"]
        C7["7. Update max output token defaults"]
        C8["8. Run full eval suite on new model"]
        C9["9. Verify prompt caching works"]
        C10["10. Check prompt-specific model behaviors<br/>(e.g., over-commenting, verbosity)"]
    end

    subgraph Validation["Validation Steps"]
        V1["grep -r '@\\[MODEL LAUNCH\\]' src/<br/>→ Find all update points"]
        V2["Run eval suite with new model"]
        V3["Compare metrics with previous model"]
        V4["A/B test with internal users"]
        V5["Monitor cost impact"]
    end

    Checklist --> Validation

    style Checklist fill:#2980b9,color:#fff
    style Validation fill:#27ae60,color:#fff
```

---

## 10. Quality Gates & CI Pipeline

```mermaid
flowchart TD
    subgraph PR["Pull Request"]
        Code["Code changes"]
    end

    subgraph Gate1["Gate 1: Static Analysis"]
        TypeCheck["TypeScript type checking"]
        Lint["ESLint + Biome"]
        CustomRules["Custom lint rules<br/>(e.g., prompt-spacing)"]
    end

    subgraph Gate2["Gate 2: Unit Tests"]
        Fast["Fast tests (< 30s total)"]
        Coverage["Coverage thresholds"]
    end

    subgraph Gate3["Gate 3: Integration Tests"]
        VCRReplay["VCR replay tests"]
        PermissionTests["Permission regression tests"]
        ToolTests["Tool behavior tests"]
    end

    subgraph Gate4["Gate 4: Prompt Evals"]
        ToolSelection["Tool selection accuracy"]
        SafetyCompliance["Safety compliance"]
        OutputQuality["Output quality metrics"]
    end

    subgraph Gate5["Gate 5: Build Verification"]
        BunBuild["bun:bundle build succeeds"]
        DCECheck["Dead code elimination verified"]
        BundleSize["Bundle size within limits"]
    end

    PR --> Gate1 --> Gate2 --> Gate3 --> Gate4 --> Gate5

    style Gate1 fill:#3498db,color:#fff
    style Gate2 fill:#27ae60,color:#fff
    style Gate3 fill:#f39c12,color:#333
    style Gate4 fill:#8e44ad,color:#fff
    style Gate5 fill:#e74c3c,color:#fff
```

### The Custom Lint Rule: `prompt-spacing`

There's a custom ESLint rule `custom-rules/prompt-spacing` that validates system prompt formatting. This is an example of treating prompts as first-class code with their own quality gates.
