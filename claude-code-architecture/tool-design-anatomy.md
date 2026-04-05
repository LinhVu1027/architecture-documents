# Tool Design Anatomy — Building Tools an LLM Can Actually Use

> How to design, validate, execute, and orchestrate tools for an AI coding agent — the patterns that make Claude Code's 40+ tools work reliably. Every diagram is a Mermaid diagram you can render in any Markdown viewer.

---

## Table of Contents

1. [Why Tool Design Is Different for LLMs](#1-why-tool-design-is-different-for-llms)
2. [Anatomy of a Tool Definition](#2-anatomy-of-a-tool-definition)
3. [The Tool Type System](#3-the-tool-type-system)
4. [Input Validation with Zod](#4-input-validation-with-zod)
5. [The ToolUseContext — Shared Brain](#5-the-tooluse-context--shared-brain)
6. [Tool Execution Pipeline](#6-tool-execution-pipeline)
7. [Concurrency Classification](#7-concurrency-classification)
8. [Streaming Tool Execution](#8-streaming-tool-execution)
9. [Tool Result Design Patterns](#9-tool-result-design-patterns)
10. [The Permission Gate](#10-the-permission-gate)
11. [Tool Registry & Merging](#11-tool-registry--merging)
12. [Anti-Patterns in Tool Design](#12-anti-patterns-in-tool-design)

---

## 1. Why Tool Design Is Different for LLMs

When a human uses an API, they read docs and adapt. When an LLM uses a tool, it relies on the **tool description, parameter schema, and error messages** to make decisions. Bad tool design = wrong tool choices, malformed inputs, and wasted tokens.

```mermaid
graph TB
    subgraph Human["Human API Consumer"]
        H1["Reads comprehensive documentation"]
        H2["Adapts to edge cases through experience"]
        H3["Understands implicit conventions"]
        H4["Asks colleagues when confused"]
    end

    subgraph LLM["LLM Tool Consumer"]
        L1["Only sees: name, description, JSON schema"]
        L2["No learning between sessions"]
        L3["Literal interpretation of descriptions"]
        L4["Self-corrects from error messages"]
    end

    subgraph Implications["Design Implications"]
        I1["Descriptions must be COMPLETE<br/>and UNAMBIGUOUS"]
        I2["Error messages must be ACTIONABLE<br/>('file not found' → suggest correct path)"]
        I3["Schema must PREVENT invalid states<br/>(use enums, constraints, defaults)"]
        I4["Names must DIFFERENTIATE clearly<br/>(Read vs Grep vs Glob)"]
    end

    Human --> Implications
    LLM --> Implications

    style Human fill:#3498db,color:#fff
    style LLM fill:#e74c3c,color:#fff
    style Implications fill:#27ae60,color:#fff
```

---

## 2. Anatomy of a Tool Definition

Every tool in Claude Code follows a consistent structure.

```mermaid
graph TD
    subgraph ToolDef["Tool Definition Structure"]
        direction TB
        Name["name: string<br/><i>'Read', 'Edit', 'Bash'</i>"]
        Description["description: string<br/><i>LLM-facing usage guide</i>"]
        InputSchema["inputSchema: ZodObject<br/><i>Validated parameter schema</i>"]
        Execute["call(input, context): Result<br/><i>The actual implementation</i>"]
        Flags["Behavioral Flags:<br/>- isReadOnly(): boolean<br/>- userFacingName(): string<br/>- isEnabled(context): boolean<br/>- getPermissions(input): PermResult<br/>- renderToolResultMessage(input, result)"]
        Limits["Resource Limits:<br/>- maxResultSizeChars: number<br/>- maxTokens: number"]
    end

    subgraph FileStructure["File Organization per Tool"]
        direction TB
        Dir["src/tools/BashTool/"]
        Main["BashTool.ts — Tool definition + call()"]
        Prompt["prompt.ts — Name + description constants"]
        Perms["bashPermissions.ts — Permission logic"]
        Utils["utils.ts — Helper functions"]
        Constants["constants.ts — Shared constants"]
    end

    ToolDef --> FileStructure

    style ToolDef fill:#2980b9,color:#fff
    style FileStructure fill:#27ae60,color:#fff
```

---

## 3. The Tool Type System

The `Tool.ts` type definitions form the contract between the LLM, the orchestrator, and tool implementations.

```mermaid
classDiagram
    class Tool {
        +name: string
        +description: string
        +inputSchema: ZodObject
        +call(input, context): ToolResultBlockParam
        +isReadOnly(input): boolean
        +userFacingName(input): string
        +isEnabled(context): boolean
        +getPermissions(input, context): PermissionResult
        +renderToolResultMessage(input, output): ReactNode
        +renderToolUseMessage(input): ReactNode
        +maxResultSizeChars?: number
        +maxTokens?: number
        +progressData?(input): ToolProgressData
    }

    class ToolUseContext {
        +options: ToolOptions
        +abortController: AbortController
        +setToolJSX: SetToolJSXFn
        +readFileState: FileStateCache
    }

    class ToolOptions {
        +tools: Tools
        +maxThinkingTokens: number
        +slowAndCapableModel: string
        +thinkingConfig: ThinkingConfig
        +permissionContext: ToolPermissionContext
        +agentId: AgentId
    }

    class ToolPermissionContext {
        +mode: PermissionMode
        +alwaysAllowRules: ToolPermissionRulesBySource
        +alwaysDenyRules: ToolPermissionRulesBySource
        +alwaysAskRules: ToolPermissionRulesBySource
        +shouldAvoidPermissionPrompts: boolean
    }

    Tool --> ToolUseContext : receives
    ToolUseContext --> ToolOptions : contains
    ToolOptions --> ToolPermissionContext : contains

    note for Tool "Each of 40+ tools implements this interface"
    note for ToolUseContext "Shared mutable state across all tools in a turn"
```

---

## 4. Input Validation with Zod

Every tool uses Zod v4 schemas for input validation. This is critical — the LLM can generate any JSON.

```mermaid
flowchart TD
    subgraph LLMOutput["LLM Generates Tool Call"]
        Raw["Raw JSON input from model:<br/>{ 'file_path': '/src/foo.ts',<br/>  'offset': 'ten' }"]
    end

    subgraph Validation["Zod Validation"]
        Parse["schema.safeParse(input)"]
        Success{"Valid?"}
        Valid["Parsed & typed input<br/>{ file_path: string, offset: number }"]
        Invalid["Validation error:<br/>'Expected number at offset,<br/>received string'"]
    end

    subgraph Response["Response to LLM"]
        UseResult["Execute tool with validated input"]
        ErrorResult["Return error as tool_result<br/>→ LLM self-corrects on next turn"]
    end

    Raw --> Parse --> Success
    Success -->|"Yes"| Valid --> UseResult
    Success -->|"No"| Invalid --> ErrorResult

    style LLMOutput fill:#2980b9,color:#fff
    style Validation fill:#f39c12,color:#333
    style Response fill:#27ae60,color:#fff
```

### Why Zod Over JSON Schema?

| Feature | JSON Schema | Zod |
|---|---|---|
| Type inference | Manual types | Automatic TypeScript types |
| Custom validators | Verbose | `.refine()` / `.transform()` |
| Error messages | Generic | Descriptive, customizable |
| Runtime + compile time | Runtime only | Both |
| API compatibility | Direct → API | Auto-converted via `.jsonSchema` |

---

## 5. The ToolUseContext — Shared Brain

The `ToolUseContext` is the shared mutable state that all tools can read and modify during a query loop iteration.

```mermaid
graph TD
    subgraph Context["ToolUseContext"]
        direction TB
        Options["options: {<br/>  tools: Tool[],<br/>  model: string,<br/>  permissionContext,<br/>  agentId,<br/>  thinkingConfig<br/>}"]
        Abort["abortController:<br/>Cancellation signal"]
        JSX["setToolJSX:<br/>Set React UI overlay"]
        FileState["readFileState:<br/>Cache of file content<br/>(avoids redundant reads)"]
    end

    subgraph Mutation["Context Modification Pattern"]
        direction TB
        M1["Tool reads context"]
        M2["Tool executes"]
        M3["Tool returns contextModifier<br/>(function that transforms context)"]
        M4["Orchestrator applies modifier<br/>AFTER tool completes"]
    end

    subgraph Examples["Context Modification Examples"]
        direction TB
        E1["FileEdit: updates readFileState<br/>with new file content"]
        E2["AgentTool: adds new MCP tools<br/>to options.tools"]
        E3["EnterWorktree: changes CWD<br/>for subsequent tools"]
    end

    Context --> Mutation --> Examples

    style Context fill:#2980b9,color:#fff
    style Mutation fill:#8e44ad,color:#fff
    style Examples fill:#27ae60,color:#fff
```

### Why Context Modifiers Are Deferred

Tools run concurrently when read-only. If two tools both modified the context simultaneously, you'd get race conditions. Instead:
1. Tools return **modifier functions** alongside their results
2. The orchestrator applies modifiers **sequentially** after the concurrent batch completes
3. This ensures consistency without locks

---

## 6. Tool Execution Pipeline

Every tool call goes through a multi-stage pipeline before execution.

```mermaid
flowchart TD
    subgraph Receive["1. Receive Tool Call"]
        TC["ToolUseBlock from API response:<br/>{ id, name, input }"]
    end

    subgraph Lookup["2. Tool Lookup"]
        Find["findToolByName(tools, name)"]
        Found{"Tool<br/>found?"}
        NotFound["Return error:<br/>'Tool not found'"]
    end

    subgraph Validate["3. Input Validation"]
        Parse["tool.inputSchema.safeParse(input)"]
        Valid{"Valid?"}
        Invalid["Return validation error"]
    end

    subgraph Permission["4. Permission Check"]
        GetPerms["tool.getPermissions(input, context)"]
        PermResult{"Allowed?"}
        Denied["Return denial message"]
        Hooks["Run PreToolUse hooks"]
        HookResult{"Hook<br/>approved?"}
        HookDenied["Return hook rejection"]
    end

    subgraph Execute["5. Execute"]
        Call["tool.call(validatedInput, context)"]
        Timeout["With timeout + abort signal"]
        Result["Tool result (string or content blocks)"]
    end

    subgraph PostProcess["6. Post-Process"]
        PostHooks["Run PostToolUse hooks"]
        Budget["Apply tool result budget<br/>(truncate/persist if oversized)"]
        Render["Render result for terminal UI"]
        Yield["Yield UserMessage with tool_result"]
    end

    TC --> Find --> Found
    Found -->|"Yes"| Parse --> Valid
    Found -->|"No"| NotFound
    Valid -->|"Yes"| GetPerms --> PermResult
    Valid -->|"No"| Invalid
    PermResult -->|"Yes"| Hooks --> HookResult
    PermResult -->|"No"| Denied
    HookResult -->|"Yes"| Call --> Result --> PostHooks --> Budget --> Render --> Yield
    HookResult -->|"No"| HookDenied

    style Receive fill:#2c3e50,color:#fff
    style Permission fill:#e74c3c,color:#fff
    style Execute fill:#27ae60,color:#fff
    style PostProcess fill:#2980b9,color:#fff
```

---

## 7. Concurrency Classification

The orchestrator partitions tool calls into concurrent-safe and serial batches.

```mermaid
flowchart TD
    subgraph Classification["Tool Concurrency Classification"]
        direction TB
        ReadOnly["isReadOnly(input) = true<br/>Examples: Read, Glob, Grep,<br/>WebFetch, WebSearch"]
        WriteTools["isReadOnly(input) = false<br/>Examples: Edit, Write, Bash,<br/>Agent (may write)"]
        Dynamic["Dynamic classification:<br/>Bash with read-only command<br/>→ CAN be concurrent"]
    end

    subgraph Partitioning["partitionToolCalls() Algorithm"]
        direction TB
        P1["Scan tool calls left-to-right"]
        P2{"Current tool<br/>read-only?"}
        P3["Add to concurrent batch"]
        P4["Close concurrent batch<br/>Create serial batch (size 1)"]
        P5["Continue scanning"]
    end

    subgraph Execution["Execution Pattern"]
        Batch1["Batch 1 (concurrent): Read A, Glob B, Grep C<br/>→ All 3 run in parallel (up to 10)"]
        Batch2["Batch 2 (serial): Edit A<br/>→ Runs alone"]
        Batch3["Batch 3 (concurrent): Read D, Read E<br/>→ Both run in parallel"]
    end

    Classification --> Partitioning
    P1 --> P2
    P2 -->|"Yes"| P3 --> P5
    P2 -->|"No"| P4 --> P5
    Partitioning --> Execution

    style ReadOnly fill:#27ae60,color:#fff
    style WriteTools fill:#e74c3c,color:#fff
    style Dynamic fill:#f39c12,color:#333
```

### Max Concurrency

```typescript
function getMaxToolUseConcurrency(): number {
  return parseInt(process.env.CLAUDE_CODE_MAX_TOOL_USE_CONCURRENCY || '', 10) || 10
}
```

Default: 10 concurrent read-only tools. Configurable via environment variable.

---

## 8. Streaming Tool Execution

The `StreamingToolExecutor` starts executing tools **before the API response finishes streaming**.

```mermaid
sequenceDiagram
    participant API as Claude API (Streaming)
    participant STE as StreamingToolExecutor
    participant T1 as Tool 1 (Read)
    participant T2 as Tool 2 (Grep)
    participant T3 as Tool 3 (Edit)

    API->>STE: content_block_start: tool_use (Read)
    API->>STE: content_block_delta: { file_path: "..." }
    API->>STE: content_block_stop
    STE->>T1: Start execution immediately
    Note over T1: Running in parallel with stream

    API->>STE: content_block_start: tool_use (Grep)
    API->>STE: content_block_stop
    STE->>T2: Start execution (concurrent with T1)

    T1-->>STE: Result ready (buffered)
    T2-->>STE: Result ready (buffered)

    API->>STE: content_block_start: tool_use (Edit)
    API->>STE: content_block_stop
    Note over STE: Edit is NOT read-only<br/>Wait for Read+Grep to complete first
    STE->>T3: Start execution (serial)

    API->>STE: message_stop
    T3-->>STE: Result ready
    STE-->>API: All results yielded in order
```

### Key Design: Order-Preserving Results

Even though tools execute out of order (concurrent reads finish before serial writes start), results are **buffered and yielded in the order tools were received**. This ensures the conversation history is deterministic.

### Error Propagation

When a Bash tool errors, the `StreamingToolExecutor` fires a child abort controller:
- Sibling tools in the same concurrent batch are aborted
- The parent query loop's abort controller is **NOT** fired (the model gets to see the error and self-correct)

---

## 9. Tool Result Design Patterns

How a tool formats its result determines whether the model can use the information effectively.

```mermaid
graph TD
    subgraph Good["Effective Result Patterns"]
        G1["STRUCTURED OUTPUT<br/>'Found 3 files:\\n1. src/foo.ts (42 lines)\\n2. src/bar.ts (18 lines)'<br/>→ Model can parse and reference"]
        G2["ACTIONABLE ERRORS<br/>'File not found: /src/foo.ts\\nDid you mean: /src/Foo.ts?'<br/>→ Model can self-correct"]
        G3["BOUNDED OUTPUT<br/>Line numbers in results (cat -n format):<br/>'42→  function foo() {'<br/>→ Model can reference exact lines"]
        G4["CONTEXT PRESERVATION<br/>File content with surrounding lines<br/>-B/-A flags on Grep results<br/>→ Model understands context"]
    end

    subgraph Bad["Ineffective Result Patterns"]
        B1["❌ RAW DUMPS<br/>Entire 10K line file<br/>→ Floods context, model loses focus"]
        B2["❌ TERSE ERRORS<br/>'Error'<br/>→ Model can't diagnose or fix"]
        B3["❌ UNSTABLE FORMATS<br/>Different format each time<br/>→ Model can't build reliable parsing"]
        B4["❌ BINARY DATA<br/>Raw bytes or encoded content<br/>→ Model can't process"]
    end

    style Good fill:#27ae60,color:#fff
    style Bad fill:#e74c3c,color:#fff
```

---

## 10. The Permission Gate

Every tool call passes through the permission system before execution.

```mermaid
flowchart TD
    subgraph Sources["Permission Sources"]
        Mode["Permission Mode<br/>(default / acceptEdits /<br/>bypassPermissions / plan)"]
        Rules["Permission Rules<br/>(alwaysAllow / alwaysDeny /<br/>alwaysAsk)"]
        Hooks["PreToolUse Hooks<br/>(custom shell scripts)"]
        Classifier["YOLO Classifier<br/>(ML safety classifier)"]
    end

    subgraph Resolution["Permission Resolution"]
        Check["checkPermission(tool, input, mode, rules)"]
        Result{"Decision"}
        Allow["ALLOW: execute immediately"]
        Deny["DENY: return rejection message"]
        Ask["ASK: show permission dialog to user"]
    end

    subgraph UserDialog["User Permission Dialog"]
        Dialog["'Allow Bash: rm -rf node_modules?'<br/>[Allow] [Deny] [Always Allow]"]
        UserAllow["User allows → execute"]
        UserDeny["User denies → return rejection"]
        UserAlways["User always-allows → add rule"]
    end

    Sources --> Resolution
    Result -->|"allow"| Allow
    Result -->|"deny"| Deny
    Result -->|"ask"| Ask --> Dialog
    Dialog --> UserAllow
    Dialog --> UserDeny
    Dialog --> UserAlways

    style Sources fill:#2c3e50,color:#fff
    style Resolution fill:#f39c12,color:#333
    style UserDialog fill:#2980b9,color:#fff
```

---

## 11. Tool Registry & Merging

Tools come from multiple sources and are merged into a single registry.

```mermaid
flowchart LR
    subgraph Sources["Tool Sources"]
        direction TB
        BuiltIn["Built-in Tools (40+)<br/>Read, Write, Edit, Bash,<br/>Glob, Grep, Agent, etc."]
        MCP["MCP Server Tools<br/>Dynamically loaded from<br/>connected MCP servers"]
        Plugin["Plugin Tools<br/>From installed plugins<br/>(plugin.json manifest)"]
    end

    subgraph Merge["useMergedTools()"]
        direction TB
        Collect["Collect all tool arrays"]
        Dedupe["Deduplicate by name<br/>(built-in wins on conflict)"]
        Filter["Filter by isEnabled(context)"]
        Sort["Sort for consistent ordering"]
    end

    subgraph Registry["Final Tool Registry"]
        direction TB
        Tools["tools: Tool[]<br/>Single flat array used by:<br/>- System prompt (tool descriptions)<br/>- Query loop (tool execution)<br/>- Permission system (tool rules)"]
    end

    Sources --> Merge --> Registry

    style Sources fill:#2980b9,color:#fff
    style Merge fill:#8e44ad,color:#fff
    style Registry fill:#27ae60,color:#fff
```

### Tool Search (Deferred Loading)

For sessions with 100+ MCP tools, loading all tool schemas into the system prompt is expensive. Tool Search allows tools to be **deferred** — only their names are visible until the model explicitly requests the full schema via the `ToolSearch` tool.

---

## 12. Anti-Patterns in Tool Design

Lessons learned from building 40+ tools for LLM consumption.

```mermaid
graph TD
    subgraph AntiPatterns["Anti-Patterns to Avoid"]
        AP1["❌ AMBIGUOUS NAMES<br/>'Search' — search what? Files? Content? Web?<br/>→ Use 'Glob' (file patterns), 'Grep' (content),<br/>'WebSearch' (internet)"]
        AP2["❌ OVERLOADED TOOLS<br/>One tool that reads, writes, AND deletes<br/>→ Model can't reason about safety per-action<br/>→ Split into Read, Write, Edit"]
        AP3["❌ UNBOUNDED OUTPUT<br/>Return entire git log with 10K commits<br/>→ Flood context, lose focus<br/>→ Always cap output, suggest pagination"]
        AP4["❌ SILENT FAILURES<br/>Return empty string on error<br/>→ Model thinks tool worked<br/>→ Always return explicit error messages"]
        AP5["❌ COMPLEX NESTED INPUT<br/>Deeply nested JSON with optional subobjects<br/>→ Model generates malformed input<br/>→ Flat schemas with clear descriptions"]
        AP6["❌ STATE DEPENDENCIES<br/>Tool B only works after Tool A<br/>→ Model can't infer ordering<br/>→ Make tools independently usable<br/>or document dependencies in description"]
    end

    style AntiPatterns fill:#e74c3c,color:#fff
```

### The Golden Rule

> **If you need to explain the tool's behavior in the system prompt (outside the tool description), the tool design is leaking.**

The system prompt should only contain **cross-tool coordination** guidance (e.g., "prefer Read over cat"). Individual tool behavior should be fully described in the tool's own description field.
