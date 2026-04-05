# Security Model & Trust Boundaries

> How Claude Code prevents an AI model from doing unauthorized damage — a layered defense-in-depth approach. Every diagram is a Mermaid diagram you can render in any Markdown viewer.

---

## Table of Contents

1. [Threat Model Overview](#1-threat-model-overview)
2. [Trust Boundary Map](#2-trust-boundary-map)
3. [Permission Engine Deep Dive](#3-permission-engine-deep-dive)
4. [Sandbox Architecture](#4-sandbox-architecture)
5. [Hook-Based Security Gates](#5-hook-based-security-gates)
6. [File Access Control](#6-file-access-control)
7. [Authentication & Secret Management](#7-authentication--secret-management)
8. [MCP Server Trust Model](#8-mcp-server-trust-model)
9. [Plugin Security](#9-plugin-security)
10. [Multi-Agent Security](#10-multi-agent-security)
11. [Defense-in-Depth Summary](#11-defense-in-depth-summary)

---

## 1. Threat Model Overview

Claude Code operates in a unique threat landscape: the AI model is simultaneously the **user's assistant** and a **potential source of risk**. It can generate and execute arbitrary code, modify files, and interact with external systems.

```mermaid
flowchart TB
    subgraph Threats["Primary Threat Vectors"]
        T1["Model Hallucination\n→ Generates dangerous commands\n(rm -rf, DROP TABLE)"]
        T2["Prompt Injection\n→ Malicious content in tool results\nredirects model behavior"]
        T3["Scope Creep\n→ Model performs actions\nbeyond user's intent"]
        T4["Data Exfiltration\n→ Model sends sensitive data\nto external services"]
        T5["Supply Chain\n→ MCP servers or plugins\ncontain malicious code"]
        T6["Privilege Escalation\n→ Subagent bypasses\nparent's permission scope"]
    end

    subgraph Defenses["Defense Layers"]
        D1["Permission System\n(rules + user approval)"]
        D2["Sandbox\n(macOS seatbelt / Linux)"]
        D3["Hook Validation\n(PreToolUse / PostToolUse)"]
        D4["Input Validation\n(JSON Schema per tool)"]
        D5["File Path Restrictions\n(working directory scope)"]
        D6["MCP OAuth + Scoping"]
    end

    T1 --> D1
    T1 --> D2
    T2 --> D3
    T3 --> D1
    T4 --> D2
    T4 --> D5
    T5 --> D6
    T6 --> D1

    style Threats fill:#e74c3c,color:#fff
    style Defenses fill:#27ae60,color:#fff
```

### Design Insight: Why Defense-in-Depth?

No single layer is sufficient:
- **Permissions alone** can't prevent a model from encoding `rm -rf /` as `echo cm0gLXJmIC8= | base64 -d | bash`
- **Sandboxing alone** would block legitimate operations (installing packages, running builds)
- **Hooks alone** depend on correct configuration

Claude Code layers all three so that if one layer fails, others catch the attack.

---

## 2. Trust Boundary Map

```mermaid
flowchart TB
    subgraph Trusted["TRUSTED ZONE (User's Machine)"]
        User["User\n(highest trust)"]
        Settings["Settings & Permissions\n(.claude/settings.json)"]
        Hooks["User-defined Hooks\n(shell scripts)"]
    end

    subgraph SemiTrusted["SEMI-TRUSTED ZONE (AI + Tools)"]
        Model["Claude Model\n(can hallucinate,\ncan be prompt-injected)"]
        BuiltInTools["Built-in Tools\n(validated, sandboxed)"]
        PluginTools["Plugin Tools\n(user-installed)"]
    end

    subgraph Untrusted["UNTRUSTED ZONE (External)"]
        MCPServers["MCP Servers\n(external processes)"]
        ToolResults["Tool Results\n(may contain injections)"]
        WebContent["Web Content\n(fetched pages)"]
        FileContent["File Content\n(may be adversarial)"]
    end

    User -->|"Configures rules"| Settings
    User -->|"Defines hooks"| Hooks
    Settings -->|"Governs"| Model
    Hooks -->|"Validates"| BuiltInTools

    Model -->|"Requests tool use"| BuiltInTools
    Model -->|"Requests tool use"| PluginTools
    BuiltInTools -->|"May invoke"| MCPServers

    MCPServers -->|"Returns"| ToolResults
    BuiltInTools -->|"Returns"| ToolResults
    ToolResults -->|"Fed back to"| Model

    style Trusted fill:#27ae60,color:#fff
    style SemiTrusted fill:#f39c12,color:#fff
    style Untrusted fill:#e74c3c,color:#fff
```

### Design Insight: The Model is Semi-Trusted

This is the fundamental tension in Claude Code's security model:

| Entity | Trust Level | Reason |
|---|---|---|
| **User** | Fully trusted | They own the machine; their intent is the ground truth |
| **Claude model** | Semi-trusted | Highly capable but can hallucinate, be manipulated by prompt injection, or misinterpret intent |
| **Built-in tools** | Semi-trusted | Code is audited but executes model's potentially dangerous requests |
| **MCP servers** | Untrusted | External processes that could be compromised |
| **File/web content** | Untrusted | Could contain prompt injection payloads |

The permission system exists because the model is semi-trusted — it needs to be able to do real work, but every destructive action should have user confirmation.

---

## 3. Permission Engine Deep Dive

```mermaid
flowchart TD
    subgraph RuleSources["Rule Sources (Priority Order)"]
        CLI["1. CLI Flags\n--allowedTools\n--permission-mode"]
        Project["2. Project Settings\n.claude/settings.json"]
        User["3. User Settings\n~/.claude/settings.json"]
        MDM["4. MDM / Enterprise\n(managed device policy)"]
        Remote["5. Remote Managed\n(server-pushed config)"]
    end

    subgraph RuleTypes["Three Rule Lists"]
        Deny["alwaysDeny\n❌ Never allow\nHighest priority"]
        Allow["alwaysAllow\n✅ Auto-approve\nSkips user prompt"]
        Ask["alwaysAsk\n❓ Always prompt\nEven if mode allows"]
    end

    subgraph Matching["Rule Matching Engine"]
        ToolMatch["Match tool name\n(exact or glob)"]
        InputMatch["Match input pattern\n(glob for paths,\nregex for commands)"]
        Examples["Examples:\nBash(npm test) → allow\nBash(rm *) → deny\nEdit(*.config) → ask"]
    end

    RuleSources --> RuleTypes
    RuleTypes --> Matching

    subgraph Evaluation["Evaluation Flow"]
        E1{"alwaysDeny\nmatch?"}
        E2{"alwaysAllow\nmatch?"}
        E3{"alwaysAsk\nmatch?"}
        E4{"Permission\nmode?"}
        E5["Prompt user"]
        E6["Auto-classify\n(AI classifier)"]

        E1 -->|Yes| Block["🚫 BLOCKED"]
        E1 -->|No| E2
        E2 -->|Yes| Approve["✅ APPROVED"]
        E2 -->|No| E3
        E3 -->|Yes| E5
        E3 -->|No| E4
        E4 -->|"plan"| PlanCheck{"Read-only\ntool?"}
        PlanCheck -->|Yes| Approve
        PlanCheck -->|No| Block
        E4 -->|"default"| WriteCheck{"Write\ntool?"}
        WriteCheck -->|Yes| E5
        WriteCheck -->|No| Approve
        E4 -->|"auto"| E6
        E6 -->|Safe| Approve
        E6 -->|Unsafe| E5
        E4 -->|"bypass"| Approve
        E5 -->|"Allow"| Approve
        E5 -->|"Deny"| Block
        E5 -->|"Allow always"| SaveRule["Save to alwaysAllow\n+ Approve"]
    end

    Matching --> Evaluation

    style Block fill:#e74c3c,color:#fff
    style Approve fill:#27ae60,color:#fff
    style Deny fill:#e74c3c,color:#fff
    style Allow fill:#27ae60,color:#fff
    style Ask fill:#f39c12,color:#fff
```

### Permission Modes Compared

```mermaid
quadrantChart
    title Permission Mode Safety vs Productivity
    x-axis "Low Productivity" --> "High Productivity"
    y-axis "Low Safety" --> "High Safety"
    quadrant-1 "Ideal (safe + productive)"
    quadrant-2 "Safe but slow"
    quadrant-3 "Dangerous"
    quadrant-4 "Fast but risky"
    "plan": [0.2, 0.95]
    "default": [0.5, 0.85]
    "auto": [0.75, 0.7]
    "acceptEdits": [0.8, 0.55]
    "dontAsk": [0.9, 0.3]
    "bypass": [0.95, 0.1]
```

### Design Insight: Why alwaysDeny > alwaysAllow > alwaysAsk?

The priority ordering prevents a common security mistake:

1. **Enterprise MDM** sets `alwaysDeny: ["Bash(curl *)"]` to prevent data exfiltration
2. **Developer** sets `alwaysAllow: ["Bash(curl localhost:*)"]` for local API testing
3. **Result**: The deny rule wins — enterprise policy cannot be overridden by user convenience

This is the **deny-by-default** principle: explicit denials are absolute, explicit allows are conditional, and everything else requires judgment.

---

## 4. Sandbox Architecture

Claude Code includes a sandboxing layer for Bash command execution that provides OS-level containment.

```mermaid
flowchart TD
    subgraph BashTool["BashTool Receives Command"]
        Cmd["User/model requests:\nbash('npm install lodash')"]
    end

    subgraph SandboxCheck["shouldUseSandbox() Check"]
        DisabledByUser{"dangerouslyDisableSandbox\n= true?"}
        ExcludedCmd{"Command in\nexcludedCommands?"}
        SandboxAvail{"SandboxManager\navailable?"}
    end

    subgraph SandboxExec["Sandboxed Execution"]
        MacOS["macOS: seatbelt\n(sandbox-exec profile)"]
        Linux["Linux: seccomp\nor namespace isolation"]
        Restrictions["Restrictions applied:\n- Network access control\n- File write scoping\n- Process spawn limits\n- System call filtering"]
    end

    subgraph DirectExec["Direct Execution"]
        Execa["execa(command, {shell: true})\nNo OS-level containment\n(permission system only)"]
    end

    Cmd --> DisabledByUser
    DisabledByUser -->|Yes| DirectExec
    DisabledByUser -->|No| ExcludedCmd
    ExcludedCmd -->|Yes| DirectExec
    ExcludedCmd -->|No| SandboxAvail
    SandboxAvail -->|Yes| SandboxExec
    SandboxAvail -->|No| DirectExec

    style SandboxExec fill:#27ae60,color:#fff
    style DirectExec fill:#f39c12,color:#fff
```

### Binary Hijack Protection

```mermaid
flowchart LR
    subgraph Attack["Attack: Binary Hijack"]
        A1["Model sets:\nPATH=/tmp/evil:$PATH"]
        A2["Then runs:\ngit status"]
        A3["Executes /tmp/evil/git\ninstead of /usr/bin/git"]
    end

    subgraph Defense["Defense: BINARY_HIJACK_VARS"]
        D1["bashPermissions.ts\ndetects env var manipulation"]
        D2["Strips PATH, LD_PRELOAD,\nLD_LIBRARY_PATH, DYLD_*\nfrom command prefix"]
        D3["Only allows safe wrappers:\nenv, timeout, nice, etc."]
    end

    Attack --> Defense

    style Attack fill:#e74c3c,color:#fff
    style Defense fill:#27ae60,color:#fff
```

### Design Insight: Sandbox is a Safety Net, Not the Primary Control

The sandbox is deliberately positioned as the **last line of defense**, not the first:

1. **Primary**: Permission system (user approval) — catches most dangerous operations
2. **Secondary**: Hook validation — catches pattern-specific risks
3. **Tertiary**: Sandbox — OS-level containment for when everything else fails

The `dangerouslyDisableSandbox` flag name itself is a security UX pattern — the word "dangerously" forces conscious acknowledgment of risk.

---

## 5. Hook-Based Security Gates

```mermaid
sequenceDiagram
    participant Model as Claude Model
    participant Pre as PreToolUse Hook
    participant Perm as Permission Engine
    participant Tool as Tool Execution
    participant Post as PostToolUse Hook
    participant Result as Tool Result

    Model->>Pre: tool_use block received

    Note over Pre: Hook types:<br/>1. Shell script hooks<br/>2. Prompt-based hooks<br/>(AI-powered validation)

    alt Shell Hook
        Pre->>Pre: Execute shell command<br/>with tool name + input as args
        Pre-->>Model: stdout = decision<br/>(ALLOW / BLOCK / message)
    else Prompt-based Hook
        Pre->>Pre: Send tool context to<br/>classifier model
        Pre-->>Model: AI decides if safe
    end

    alt Hook blocks
        Pre-->>Model: "Tool blocked by hook:<br/>[reason]"
    else Hook allows
        Pre->>Perm: Continue to permission check
        Perm->>Tool: Execute if permitted
        Tool->>Post: PostToolUse hook
        Post->>Post: Inspect/modify output
        Post->>Result: Final tool result
    end
```

### Hook Event Lifecycle

```mermaid
flowchart LR
    subgraph Events["Hook Events"]
        SessionStart["SessionStart\n(app launches)"]
        UserPrompt["UserPromptSubmit\n(before processing)"]
        PreTool["PreToolUse\n(before each tool)"]
        PostTool["PostToolUse\n(after each tool)"]
        PreCompact["PreCompact\n(before compaction)"]
        Stop["Stop\n(model finishes turn)"]
        SubagentStop["SubagentStop\n(subagent finishes)"]
        Notification["Notification\n(system event)"]
        SessionEnd["SessionEnd\n(app closes)"]
    end

    SessionStart --> UserPrompt --> PreTool --> PostTool --> Stop --> SessionEnd
    PreTool --> PreCompact
    PostTool --> SubagentStop
    Stop --> Notification

    style PreTool fill:#ff6b6b,color:#fff
    style PostTool fill:#ff6b6b,color:#fff
    style Stop fill:#f39c12,color:#fff
```

### Design Insight: Why Prompt-Based Hooks?

Traditional hooks run shell scripts — fast but brittle (regex matching misses edge cases). Prompt-based hooks use an AI classifier:

| Approach | Precision | Recall | Latency | Maintenance |
|---|---|---|---|---|
| **Regex/shell** | High (exact match) | Low (misses variations) | ~5ms | High (constant updates) |
| **AI classifier** | High | High | ~200ms | Low (model generalizes) |

Example: A shell hook blocking `rm -rf` misses `find / -delete`. An AI classifier understands the intent is "delete everything" regardless of the command variation.

---

## 6. File Access Control

```mermaid
flowchart TD
    subgraph Scoping["File Access Scoping"]
        WorkDir["Working Directory\n(git root or cwd)"]
        AddlDirs["Additional Working Directories\n(from settings)"]
        TempDir["Temp directories\n(/tmp, os.tmpdir())"]
    end

    subgraph ReadAccess["Read Access Rules"]
        R1["FileReadTool:\n- Can read any file\n(model needs to explore)"]
        R2["But tool result is scoped:\n- Large files truncated\n- Binary files rejected\n- Image files rendered"]
    end

    subgraph WriteAccess["Write Access Rules"]
        W1["FileWriteTool / FileEditTool:\n- Must read file first\n(prevents blind overwrites)"]
        W2["Permission required for:\n- Files outside working dir\n- Config files (.env, etc.)"]
        W3["Validation checks:\n- Path traversal prevention\n- Symlink resolution"]
    end

    subgraph BashAccess["Bash Access Rules"]
        B1["Commands scoped by:\n- alwaysAllow patterns\n- alwaysDeny patterns\n- Sandbox restrictions"]
        B2["Working directory set to\ngit root (not /)"]
    end

    Scoping --> ReadAccess
    Scoping --> WriteAccess
    Scoping --> BashAccess

    style ReadAccess fill:#27ae60,color:#fff
    style WriteAccess fill:#f39c12,color:#fff
    style BashAccess fill:#ff6b6b,color:#fff
```

### Design Insight: Read-Before-Write Requirement

The `FileWriteTool` and `FileEditTool` enforce a **read-before-write** invariant:

```
Tool call: Write(file_path: "src/config.ts", content: "...")
→ ERROR: "You must use your Read tool at least once before editing"
```

This prevents two attack vectors:
1. **Blind overwrite** — Model hallucinates file contents and overwrites real data
2. **Race condition** — Model edits based on stale cached content

The read operation updates the `readFileState` cache, ensuring edits are based on current file contents.

---

## 7. Authentication & Secret Management

```mermaid
flowchart TD
    subgraph AuthMethods["Authentication Methods"]
        APIKey["API Key\n(ANTHROPIC_API_KEY)"]
        OAuth["OAuth Flow\n(browser-based)"]
        AWSCreds["AWS Credentials\n(for Bedrock)"]
        GCPCreds["GCP Credentials\n(for Vertex AI)"]
    end

    subgraph Storage["Credential Storage"]
        Keychain["OS Keychain\n(macOS Keychain,\nLinux secret-service)"]
        EnvFile[".env file\n(gitignored)"]
        AWSConfig["~/.aws/config\n(standard AWS)"]
    end

    subgraph Protection["Protection Mechanisms"]
        Prefetch["Keychain prefetch\n(parallel at startup,\nnever logged)"]
        NoLog["Credentials excluded from:\n- Analytics events\n- Diagnostic logs\n- Tool results\n- Session persistence"]
        GitIgnore[".env in .gitignore\n(template only)"]
        NoCommit["Commit hook warns\non .env, credentials.json"]
    end

    AuthMethods --> Storage
    Storage --> Protection

    style Protection fill:#27ae60,color:#fff
```

### Design Insight: Why Keychain Over Environment Variables?

| Storage | Security | UX | Persistence |
|---|---|---|---|
| **Environment variable** | Visible to child processes, leaked in error dumps | Manual export | Per-session |
| **OS Keychain** | Encrypted at rest, requires user auth to access | Automatic after first auth | Permanent |
| **.env file** | Plaintext on disk, risk of git commit | Easy to manage | Permanent |

Claude Code prefers the OS Keychain (via `startKeychainPrefetch()` at boot) but falls back to environment variables for CI/CD compatibility. The prefetch runs in parallel with module loading so there's no startup penalty.

---

## 8. MCP Server Trust Model

```mermaid
flowchart TD
    subgraph MCPConfig["MCP Server Configuration"]
        ConfigFile[".mcp.json or\nsettings.json"]
        MCPTypes["Transport Types:\n- stdio (local process)\n- SSE (HTTP streaming)\n- HTTP (REST)"]
    end

    subgraph Trust["Trust Boundaries"]
        StdioTrust["stdio servers:\n- Run as local process\n- Same user permissions\n- Can access filesystem"]
        HTTPTrust["HTTP/SSE servers:\n- Network accessible\n- OAuth scoping\n- Limited to declared tools"]
    end

    subgraph Protections["Protection Layers"]
        OAuthScope["OAuth Scoping\n(server declares capabilities)"]
        ToolProxy["MCPTool Proxy\n(all calls go through\npermission engine)"]
        ResultValidation["Result Validation\n(schema checking)"]
        ChannelPerms["Channel Permissions\n(per-server restrictions)"]
    end

    subgraph Risk["Residual Risks"]
        StdioRisk["stdio: Full local access\n(trusted by design)"]
        InjectionRisk["Prompt injection via\ntool result content"]
        ExfilRisk["Server could log\ntool call inputs"]
    end

    MCPConfig --> Trust
    Trust --> Protections
    Protections --> Risk

    style Protections fill:#27ae60,color:#fff
    style Risk fill:#e74c3c,color:#fff
```

### Design Insight: MCP Tools Go Through the Same Permission Engine

When an MCP server exposes a tool called `database_query`, that tool is registered through the `MCPTool` proxy and subject to the **exact same permission rules** as built-in tools:

```
alwaysDeny: ["mcp__myserver__database_query(DROP *)"]
alwaysAllow: ["mcp__myserver__database_query(SELECT *)"]
```

This means enterprise policies can restrict MCP tool usage without modifying the MCP server itself. The naming convention `mcp__{server}__{tool}` makes pattern-based rules possible.

---

## 9. Plugin Security

```mermaid
flowchart TD
    subgraph PluginLoad["Plugin Loading"]
        Discovery["Plugin Discovery:\n1. .claude/plugins/\n2. Installed via /plugin install\n3. Plugin manifest (plugin.json)"]
        Validation["Manifest Validation:\n- Schema check\n- Component enumeration\n- Path resolution"]
    end

    subgraph PluginScope["Plugin Scoping"]
        Tools["Plugin Tools:\n→ Registered in tool registry\n→ Subject to permission engine"]
        Hooks["Plugin Hooks:\n→ Merged with user hooks\n→ Same event system"]
        Commands["Plugin Commands:\n→ Merged with built-in\n→ Same slash dispatch"]
        Agents["Plugin Agents:\n→ Custom agent types\n→ Same spawn mechanism"]
        MCP["Plugin MCP Servers:\n→ Started as child processes\n→ Same trust model"]
    end

    subgraph Isolation["Isolation Mechanisms"]
        PathScope["CLAUDE_PLUGIN_ROOT\n→ Scopes file refs to\nplugin directory"]
        NoPersist["No direct state access\n→ Plugins use .local.md\nfor config"]
        SamePerms["Same permission system\n→ No privilege escalation"]
    end

    PluginLoad --> PluginScope --> Isolation

    style Isolation fill:#27ae60,color:#fff
```

### Design Insight: Why No Plugin Sandbox?

Plugins run with the same permissions as the user — there's no VM or container isolation. This is a deliberate trade-off:

| Sandboxed Plugins | Unsandboxed Plugins (Chosen) |
|---|---|
| Secure against malicious plugins | Relies on trust (user installs explicitly) |
| Can't access filesystem freely | Full filesystem access for real tools |
| Complex API for host communication | Direct function calls, simple integration |
| Slow (IPC overhead) | Fast (in-process) |

The reasoning: if a user installs a plugin, they're granting it trust. The permission system still governs what the AI *does with* plugin tools, even if the plugin code itself is unrestricted.

---

## 10. Multi-Agent Security

```mermaid
flowchart TD
    subgraph Parent["Parent Agent"]
        ParentPerms["Permission context:\nalwaysAllow: [Bash(npm *)]\nalwaysDeny: [Bash(rm *)]"]
        ParentMode["Mode: default"]
    end

    subgraph Fork["Context Fork"]
        Inherit["Child INHERITS:\n- Permission rules\n- alwaysDeny (cannot remove)\n- Permission mode (or stricter)"]
        Restrict["Child may RESTRICT:\n- Fewer tools available\n- Stricter permission mode\n- Limited working directory"]
        NoEscalate["Child CANNOT:\n- Add alwaysAllow rules\n- Weaken permission mode\n- Access parent's denied tools"]
    end

    subgraph Modes["Agent Permission Modes"]
        PlanAgent["Plan Agent:\nmode = 'plan'\n→ Read-only tools only"]
        ExploreAgent["Explore Agent:\nmode = inherited\n→ No Edit/Write/Agent tools"]
        CustomAgent["Custom Agent:\nmode = configured\n→ Tool whitelist applied"]
    end

    subgraph Bubble["Permission Bubbling"]
        WorkerReq["Worker needs\npermission approval"]
        LeaderBridge["Request bubbles to\nleader/parent terminal"]
        UserApproval["User sees prompt\nin main REPL"]
    end

    Parent --> Fork
    Fork --> Modes
    Modes --> Bubble

    style NoEscalate fill:#27ae60,color:#fff
    style Bubble fill:#f39c12,color:#fff
```

### Design Insight: Permission Bubbling in Swarms

When a worker agent in a swarm needs user permission, it can't prompt the user directly (it has no terminal). Instead:

1. Worker creates a `SandboxPermissionRequest` via the mailbox system
2. `useSwarmPermissionPoller` on the leader agent detects the pending request
3. Leader surfaces the permission dialog in the main REPL
4. User's decision flows back through the mailbox to the worker

This preserves the single-point-of-approval principle — the user always sees permission requests in one place, even with 10 agents running in parallel.

---

## 11. Defense-in-Depth Summary

```mermaid
flowchart TD
    subgraph Layer1["Layer 1: Permission Rules"]
        L1["alwaysDeny / alwaysAllow / alwaysAsk\nConfigured per-project, per-user, enterprise\nCannot be overridden by AI"]
    end

    subgraph Layer2["Layer 2: Permission Mode"]
        L2["plan / default / auto / bypass\nDetermines baseline behavior\nfor unconfigured operations"]
    end

    subgraph Layer3["Layer 3: User Approval"]
        L3["Interactive permission dialog\nUser sees exact tool + input\nCan allow once or always"]
    end

    subgraph Layer4["Layer 4: Pre-Tool Hooks"]
        L4["Shell scripts or AI classifiers\nValidate tool calls before execution\nCan block with reason"]
    end

    subgraph Layer5["Layer 5: Input Validation"]
        L5["JSON Schema validation\nPer-tool input constraints\nReject malformed requests"]
    end

    subgraph Layer6["Layer 6: Sandbox"]
        L6["OS-level containment\nmacOS seatbelt / Linux seccomp\nNetwork + filesystem restrictions"]
    end

    subgraph Layer7["Layer 7: Post-Tool Hooks"]
        L7["Inspect and modify tool output\nDetect sensitive data in results\nAudit trail for compliance"]
    end

    Layer1 --> Layer2 --> Layer3 --> Layer4 --> Layer5 --> Layer6 --> Layer7

    subgraph Cross["Cross-Cutting Concerns"]
        CC1["Read-before-write enforcement"]
        CC2["Binary hijack detection"]
        CC3["Secret detection in commits"]
        CC4["Permission bubbling in swarms"]
        CC5["Prompt injection flagging"]
    end

    style Layer1 fill:#c0392b,color:#fff
    style Layer2 fill:#e74c3c,color:#fff
    style Layer3 fill:#e67e22,color:#fff
    style Layer4 fill:#f39c12,color:#fff
    style Layer5 fill:#f1c40f,color:#333
    style Layer6 fill:#27ae60,color:#fff
    style Layer7 fill:#2ecc71,color:#fff
```

### The Security Philosophy

Claude Code's security model follows three principles:

1. **Assume the model will try dangerous things** — Not maliciously, but through hallucination or misunderstanding. Every tool call is potentially dangerous until proven safe.

2. **Make the safe path the easy path** — `default` mode asks for confirmation on writes. `alwaysAllow` patterns let users pre-approve safe operations. The UX encourages security without making it painful.

3. **Enterprise overrides individual** — MDM policies and `alwaysDeny` rules cannot be weakened by per-project settings, user preferences, or AI behavior. This makes Claude Code deployable in regulated environments.
