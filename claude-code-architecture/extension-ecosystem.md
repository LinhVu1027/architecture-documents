# Extension Ecosystem

> How plugins, hooks, skills, MCP servers, agents, and commands form a unified extensibility architecture. Every diagram is a Mermaid diagram you can render in any Markdown viewer.

---

## Table of Contents

1. [Ecosystem Overview](#1-ecosystem-overview)
2. [Plugin Architecture](#2-plugin-architecture)
3. [Hook System](#3-hook-system)
4. [Skill System](#4-skill-system)
5. [Custom Agent Definitions](#5-custom-agent-definitions)
6. [MCP Integration](#6-mcp-integration)
7. [Slash Commands](#7-slash-commands)
8. [Tool Registry & Merging](#8-tool-registry--merging)
9. [Cross-Cutting Interactions](#9-cross-cutting-interactions)
10. [Extension Point Comparison](#10-extension-point-comparison)

---

## 1. Ecosystem Overview

Claude Code has **six extension points** that work together as a unified ecosystem. Each point serves a different purpose but shares the same registration and permission infrastructure.

```mermaid
graph TB
    subgraph Core["Claude Code Core"]
        QueryLoop["Agentic Loop\n(query.ts)"]
        ToolRegistry["Tool Registry\n(tools.ts)"]
        CmdRegistry["Command Registry\n(commands.ts)"]
        HookEngine["Hook Engine\n(utils/hooks/)"]
        PermEngine["Permission Engine\n(utils/permissions/)"]
    end

    subgraph Extensions["Extension Points"]
        Plugins["Plugins\n(plugin.json manifest)"]
        Hooks["Hooks\n(settings.json)"]
        Skills["Skills\n(.md files with YAML)"]
        Agents["Agent Definitions\n(.claude/agents/*.md)"]
        MCP["MCP Servers\n(.mcp.json)"]
        Commands["Slash Commands\n(.ts/.md files)"]
    end

    subgraph Integration["Integration Layer"]
        MergedTools["useMergedTools()\nBuilt-in + MCP + Plugin tools"]
        MergedCmds["useMergedCommands()\nBuilt-in + Plugin commands"]
        MergedHooks["Merged hook configs\nUser + Plugin hooks"]
        MergedAgents["Merged agent definitions\nLocal + Plugin agents"]
    end

    Plugins -->|"declares"| MCP
    Plugins -->|"declares"| Hooks
    Plugins -->|"declares"| Skills
    Plugins -->|"declares"| Commands
    Plugins -->|"declares"| Agents

    MergedTools --> ToolRegistry
    MergedCmds --> CmdRegistry
    MergedHooks --> HookEngine

    ToolRegistry --> QueryLoop
    CmdRegistry --> QueryLoop
    HookEngine --> QueryLoop
    PermEngine --> QueryLoop

    style Core fill:#ff6b6b,color:#fff
    style Extensions fill:#45b7d1,color:#fff
    style Integration fill:#27ae60,color:#fff
```

### Design Insight: Why Six Extension Points Instead of One?

Each extension point has a fundamentally different execution model:

| Extension | Execution Model | Invoked By | Runs When |
|---|---|---|---|
| **Tools** | AI calls via `tool_use` block | Claude model | During agentic loop |
| **Hooks** | System triggers on events | Hook engine | Before/after tool use, stop, etc. |
| **Skills** | Prompt expansion | User slash command or pattern match | Before query starts |
| **Agents** | Spawn new query loop | AgentTool | When model needs delegation |
| **MCP** | JSON-RPC to external process | MCPTool proxy | When model calls MCP tool |
| **Commands** | Direct execution | User slash command | Immediate, outside loop |

A single "plugin" abstraction would force these different models into one interface, losing the clarity of when and how each runs.

---

## 2. Plugin Architecture

```mermaid
flowchart TD
    subgraph Manifest["plugin.json Manifest"]
        Meta["name, version, description"]
        Components["components:"]
        CompTools["  tools: [Tool definitions]"]
        CompCmds["  commands: [Command files]"]
        CompSkills["  skills: [Skill .md files]"]
        CompHooks["  hooks: [Hook configs]"]
        CompAgents["  agents: [Agent .md files]"]
        CompMCP["  mcpServers: [MCP configs]"]
        Components --> CompTools
        Components --> CompCmds
        Components --> CompSkills
        Components --> CompHooks
        Components --> CompAgents
        Components --> CompMCP
    end

    subgraph Loading["Plugin Loading Pipeline"]
        Discover["Discovery:\n1. .claude/plugins/\n2. /plugin install"]
        Parse["Parse plugin.json\n(schema validation)"]
        Resolve["Resolve paths:\n${CLAUDE_PLUGIN_ROOT}\n→ actual plugin dir"]
        Register["Register components\nwith respective registries"]
    end

    subgraph Runtime["Runtime Integration"]
        ToolMerge["Tools → useMergedTools()"]
        CmdMerge["Commands → useMergedCommands()"]
        HookMerge["Hooks → merged hook config"]
        AgentMerge["Agents → agent definitions"]
        MCPStart["MCP servers → started as children"]
    end

    Manifest --> Loading --> Runtime

    style Manifest fill:#45b7d1,color:#fff
    style Loading fill:#bb8fce,color:#fff
    style Runtime fill:#27ae60,color:#fff
```

### Plugin Lifecycle

```mermaid
stateDiagram-v2
    [*] --> Discovered: Plugin dir found

    Discovered --> Parsed: plugin.json valid
    Discovered --> Failed: Invalid manifest

    Parsed --> Loading: Components enumerated
    Loading --> Active: All components registered

    Active --> Active: User interacts with components
    Active --> Disabled: User disables / uninstalls

    Disabled --> Active: User re-enables
    Disabled --> [*]: Plugin removed

    Failed --> [*]: Error logged

    note right of Active
        Plugin components are live:
        - Tools available to model
        - Commands in slash palette
        - Hooks fire on events
        - Agents spawnable
    end note
```

### Design Insight: ${CLAUDE_PLUGIN_ROOT} Scoping

Plugins reference their own files using `${CLAUDE_PLUGIN_ROOT}`:

```json
{
  "commands": [
    {"source": "${CLAUDE_PLUGIN_ROOT}/commands/deploy.md"}
  ]
}
```

This pattern provides:
1. **Portability** — Plugin works regardless of where it's installed
2. **Security** — File references are scoped to the plugin directory (can't reference `/etc/passwd`)
3. **Isolation** — Multiple plugins can have identically-named files without collision

---

## 3. Hook System

```mermaid
flowchart TD
    subgraph HookEvents["Hook Events"]
        direction TB
        SessionStart["SessionStart\n→ App launches"]
        UserPrompt["UserPromptSubmit\n→ User presses Enter"]
        PreTool["PreToolUse\n→ Before tool execution"]
        PostTool["PostToolUse\n→ After tool execution"]
        PreCompact["PreCompact\n→ Before context compaction"]
        Stop["Stop\n→ Model finishes turn"]
        SubStop["SubagentStop\n→ Subagent finishes"]
        Notify["Notification\n→ System event"]
        SessionEnd["SessionEnd\n→ App closes"]
    end

    subgraph HookTypes["Hook Implementation Types"]
        Shell["Shell Script Hook\ntype: 'command'\nRuns bash script with\ntool context as args"]
        Prompt["Prompt-Based Hook\ntype: 'prompt'\nAI classifier evaluates\ntool call safety"]
    end

    subgraph HookConfig["Configuration (settings.json)"]
        Config["hooks: {\n  'PreToolUse': [\n    {\n      type: 'command',\n      command: 'validate.sh',\n      matcher: 'Bash'\n    }\n  ]\n}"]
    end

    subgraph HookResponse["Hook Responses"]
        Allow["ALLOW\n→ Proceed with tool"]
        Block["BLOCK + reason\n→ Return error to model"]
        Modify["MODIFY output\n→ Change tool result\n(PostToolUse only)"]
        Ignore["No response\n→ Default behavior"]
    end

    HookEvents --> HookTypes
    HookConfig --> HookTypes
    HookTypes --> HookResponse

    style PreTool fill:#ff6b6b,color:#fff
    style PostTool fill:#ff6b6b,color:#fff
    style Block fill:#e74c3c,color:#fff
    style Allow fill:#27ae60,color:#fff
```

### Hook Execution Flow (PreToolUse)

```mermaid
sequenceDiagram
    participant Tool as Tool Orchestrator
    participant HE as Hook Engine
    participant Shell as Shell Hook
    participant AI as Prompt Hook (AI)
    participant Perm as Permission Engine

    Tool->>HE: firePreToolUseHooks(toolName, input)

    HE->>HE: Find matching hooks<br/>(by tool name matcher)

    par Shell hooks (fast)
        HE->>Shell: Execute command<br/>with JSON input on stdin
        Shell-->>HE: stdout: "ALLOW" | "BLOCK: reason"
    and Prompt hooks (AI-based)
        HE->>AI: Classify tool call<br/>with context
        AI-->>HE: decision + explanation
    end

    alt Any hook BLOCKs
        HE-->>Tool: BLOCKED: "reason"
        Tool-->>Tool: Return error message to model
    else All hooks ALLOW or no hooks
        HE-->>Perm: Continue to permission check
    end
```

### Design Insight: Why Both Shell and Prompt Hooks?

```mermaid
graph LR
    subgraph ShellHooks["Shell Hooks"]
        SH1["✅ Fast (~5ms)"]
        SH2["✅ Deterministic"]
        SH3["✅ No API cost"]
        SH4["❌ Regex-based matching"]
        SH5["❌ Misses variations"]
    end

    subgraph PromptHooks["Prompt-Based Hooks"]
        PH1["✅ Semantic understanding"]
        PH2["✅ Catches edge cases"]
        PH3["✅ Context-aware"]
        PH4["❌ ~200ms latency"]
        PH5["❌ Small API cost"]
    end

    subgraph Combined["Best of Both"]
        C1["Shell: Fast rejection\nof obviously bad commands"]
        C2["Prompt: Catch subtle issues\nthat regex can't detect"]
    end

    ShellHooks --> Combined
    PromptHooks --> Combined

    style Combined fill:#27ae60,color:#fff
```

---

## 4. Skill System

```mermaid
flowchart TD
    subgraph SkillDef["Skill Definition (.md file)"]
        Frontmatter["YAML Frontmatter:\nname: deploy\ndescription: Deploy to AWS\ntrigger: 'deploy', 'ship it'\nfile_refs:\n  - '${CLAUDE_PLUGIN_ROOT}/templates/*'"]
        Body["Markdown Body:\nFull prompt expansion\nwith instructions,\nexamples, and context"]
    end

    subgraph Trigger["Trigger Mechanisms"]
        SlashCmd["/deploy\n(explicit invocation)"]
        PatternMatch["'deploy to AWS'\n(description matching)"]
        SkillTool["SkillTool\n(model invokes directly)"]
    end

    subgraph Execution["Skill Execution"]
        Expand["1. Load .md file"]
        ResolveRefs["2. Resolve file_refs\n(inject referenced files)"]
        InjectContext["3. Inject as system context\n(AttachmentMessage)"]
        QueryLoop["4. Model receives expanded\nprompt in next iteration"]
    end

    SkillDef --> Trigger
    Trigger --> Execution

    style SkillDef fill:#45b7d1,color:#fff
    style Trigger fill:#bb8fce,color:#fff
    style Execution fill:#27ae60,color:#fff
```

### Design Insight: Skills are Prompt Engineering, Not Code

Skills are intentionally **markdown files**, not code:

1. **No execution risk** — A skill can't run arbitrary code; it only expands to text that the model receives
2. **Easy to write** — Anyone who can write a prompt can create a skill
3. **Progressive disclosure** — Skills can include `file_refs` that inject code/templates as context, giving the model exactly what it needs
4. **Composable** — A skill can reference other skills, creating a hierarchy of expertise

This is fundamentally different from tools (which execute code) and hooks (which run scripts). Skills are pure knowledge injection.

---

## 5. Custom Agent Definitions

```mermaid
flowchart TD
    subgraph AgentDef["Agent Definition (.md file in .claude/agents/)"]
        AFrontmatter["YAML Frontmatter:\nname: code-reviewer\ndescription: Reviews code for bugs\ntools:\n  - Read\n  - Grep\n  - Glob\nmodel: sonnet\npermissionMode: plan"]
        ABody["Markdown Body:\nSystem prompt for the agent\nwith specialized instructions"]
    end

    subgraph Discovery["Agent Discovery"]
        Local[".claude/agents/*.md"]
        Plugin["plugin agents/\n(via plugin.json)"]
        Merged["Merged into\nagentDefinitions"]
    end

    subgraph Spawning["Agent Spawning"]
        AgentTool["AgentTool called with:\nsubagent_type: 'code-reviewer'\nprompt: 'Review src/foo.ts'"]
        ForkCtx["Fork ToolUseContext:\n- Filter tools to allowed set\n- Apply permission mode\n- Set system prompt from .md body"]
        RunQuery["Start new query() loop\nwith agent's configuration"]
    end

    AgentDef --> Discovery
    Discovery --> Spawning

    style AgentDef fill:#45b7d1,color:#fff
    style Spawning fill:#27ae60,color:#fff
```

### Built-in vs Custom Agent Types

```mermaid
graph TB
    subgraph BuiltIn["Built-in Agent Types"]
        Explore["Explore\n- Read-only tools\n- Fast codebase search"]
        Plan["Plan\n- Architecture design\n- No edit/write tools"]
        General["general-purpose\n- All tools\n- Full capabilities"]
    end

    subgraph Custom["Custom Agent Types (user/plugin defined)"]
        CodeReview["code-reviewer\n- Read + Grep only\n- Review-focused prompt"]
        TestRunner["test-runner\n- Bash + Read\n- Test execution prompt"]
        DocWriter["doc-writer\n- Read + Write + Glob\n- Documentation prompt"]
    end

    subgraph Config["Per-Agent Configuration"]
        Tools_["tools: [whitelist]"]
        Model_["model: sonnet|opus|haiku"]
        Mode_["permissionMode: plan|default"]
        Color_["color: #hex (UI)"]
    end

    BuiltIn --> Config
    Custom --> Config

    style BuiltIn fill:#45b7d1,color:#fff
    style Custom fill:#bb8fce,color:#fff
```

### Design Insight: Why Markdown-Based Agent Definitions?

The `.claude/agents/*.md` pattern means:
1. **Version controlled** — Agent definitions live in the repo alongside code
2. **Shareable** — Team members get the same agent configurations via git
3. **Declarative** — No code to maintain; just YAML config + prompt text
4. **Safe** — The frontmatter constrains what tools/mode the agent gets; it can't grant itself more power than declared

---

## 6. MCP Integration

```mermaid
flowchart TD
    subgraph Config["MCP Configuration Sources"]
        MCPJson[".mcp.json\n(project-level)"]
        Settings["settings.json\nmcpServers section"]
        Plugin["plugin.json\nmcpServers component"]
    end

    subgraph Connection["Connection Management"]
        Parse["Parse server configs"]
        Start["Start server processes\n(stdio/SSE/HTTP)"]
        ListTools["Call ListTools\nto discover capabilities"]
        ListResources["Call ListResources\nfor data access"]
    end

    subgraph Registration["Tool Registration"]
        Proxy["Create MCPTool proxy\nfor each server tool"]
        Name["Name convention:\nmcp__{server}__{tool}"]
        Merge["Merge into\nuseMergedTools()"]
    end

    subgraph Usage["Runtime Usage"]
        Model["Model calls:\nmcp__chrome__navigate(url)"]
        MCPTool["MCPTool proxy:\n1. Permission check\n2. JSON-RPC call\n3. Parse result"]
        Server["MCP Server executes\nand returns result"]
    end

    Config --> Connection --> Registration --> Usage

    style Config fill:#45b7d1,color:#fff
    style Registration fill:#bb8fce,color:#fff
    style Usage fill:#27ae60,color:#fff
```

### Deferred Tool Loading (ToolSearch)

```mermaid
sequenceDiagram
    participant Model as Claude Model
    participant TS as ToolSearchTool
    participant Registry as Tool Registry
    participant MCP as MCP Servers

    Note over Model: Model sees 200+ tools<br/>but only names + descriptions<br/>(no full schemas)

    Model->>TS: ToolSearch("select:mcp__chrome__navigate")

    TS->>Registry: Look up deferred tool
    Registry->>MCP: Fetch full schema<br/>(if not cached)
    MCP-->>Registry: JSON Schema definition
    Registry-->>TS: Full tool definition

    TS-->>Model: Tool schema now available<br/>Model can call the tool

    Model->>Registry: mcp__chrome__navigate({url: "..."})
```

### Design Insight: Why Deferred Tool Loading?

With 200+ tools (40+ built-in + MCP + plugins), sending all tool schemas in every API request would:
- Consume ~20K tokens of context window
- Slow down API calls (larger request body)
- Confuse the model with irrelevant tools

Deferred loading means the model sees tool **names and descriptions** (compact) but must explicitly load the full schema before calling a tool. This keeps the prompt lean while preserving access to the full ecosystem.

---

## 7. Slash Commands

```mermaid
flowchart TD
    subgraph CmdTypes["Command Types"]
        BuiltIn["Built-in Commands\n(TypeScript in src/commands/)\n/commit, /compact, /resume,\n/review, /mcp, /plugin"]
        Plugin_["Plugin Commands\n(.md or .ts files)\nDeclared in plugin.json"]
        Skill_["Skill-backed Commands\n(from skill definitions)\nAuto-registered from skills"]
    end

    subgraph Interface["Command Interface"]
        Name["name: string"]
        Desc["description: string"]
        Args["arguments?: ArgDef[]"]
        Enabled["isEnabled(context): boolean"]
        Handler["call(args, context): Result"]
    end

    subgraph Dispatch["Command Dispatch"]
        Input["/commit -m 'fix bug'"]
        Parse["Parse command name + args"]
        Lookup["useMergedCommands()\nfind by name"]
        Execute["Execute handler\nor expand skill"]
    end

    subgraph Display["Result Display"]
        Inline["CommandResultDisplay:\n- message: rendered in chat\n- toast: notification\n- silent: no output"]
    end

    CmdTypes --> Interface
    Interface --> Dispatch --> Display

    style CmdTypes fill:#45b7d1,color:#fff
    style Dispatch fill:#bb8fce,color:#fff
```

---

## 8. Tool Registry & Merging

```mermaid
flowchart TD
    subgraph Sources["Tool Sources"]
        BuiltIn["Built-in Tools (40+)\nsrc/tools/*\nBash, Read, Write, Edit,\nGlob, Grep, Agent, etc."]
        MCPTools["MCP Tools\nDiscovered via ListTools\nfrom connected servers"]
        PluginTools["Plugin Tools\nDefined in plugin.json\ncomponents.tools"]
    end

    subgraph Merging["useMergedTools() Hook"]
        Collect["1. Collect all sources"]
        Dedupe["2. Deduplicate by name\n(built-in > MCP > plugin)"]
        Filter["3. Filter by isEnabled()\n(some tools context-dependent)"]
        Defer["4. Mark low-priority tools\nas 'deferred' (name only)"]
        Final["5. Final Tools map"]
    end

    subgraph QueryUsage["Usage in Query Loop"]
        Schema["toolToAPISchema(tools)\n→ BetaToolUnion[]"]
        Prompt["tool.prompt(context)\n→ Dynamic descriptions"]
        Execute["findToolByName(name)\n→ Execute on tool_use"]
    end

    Sources --> Merging --> QueryUsage

    style Sources fill:#45b7d1,color:#fff
    style Merging fill:#bb8fce,color:#fff
    style QueryUsage fill:#27ae60,color:#fff
```

### Design Insight: Why Built-in > MCP > Plugin Priority?

If multiple sources define a tool with the same name:
1. **Built-in wins** — Core tools are security-audited and performance-optimized
2. **MCP second** — External servers provide specialized capabilities
3. **Plugin last** — User-installed plugins are the most flexible but least controlled

This prevents a malicious MCP server from overriding the `Read` tool with a data-exfiltrating version.

---

## 9. Cross-Cutting Interactions

```mermaid
flowchart TB
    subgraph Scenario1["Scenario: Plugin with Full Stack"]
        P1["Plugin 'deploy-aws' declares:"]
        P1Tools["Tool: runDeploy\n(executes deployment)"]
        P1Hooks["Hook: PreToolUse on Bash\n(block 'aws delete-*')"]
        P1Skills["Skill: /deploy\n(deployment guide prompt)"]
        P1Agent["Agent: deploy-reviewer\n(reviews IaC changes)"]
        P1MCP["MCP: aws-cdk-server\n(CDK operations)"]

        P1 --> P1Tools
        P1 --> P1Hooks
        P1 --> P1Skills
        P1 --> P1Agent
        P1 --> P1MCP
    end

    subgraph Interaction["How They Interact"]
        User["/deploy staging"]
        Skill["Skill expands: deployment\ninstructions + context"]
        Model["Model receives skill content\nand decides to use tools"]
        Tool["Model calls mcp__aws__synth()\n+ runDeploy()"]
        Hook["Hook validates: no\n'aws delete-*' commands"]
        Agent["Model spawns deploy-reviewer\nto check changes"]
        Result["Deployment complete\nwith safety checks"]

        User --> Skill --> Model --> Tool --> Hook
        Model --> Agent
        Hook --> Result
        Agent --> Result
    end

    style Scenario1 fill:#45b7d1,color:#fff
    style Interaction fill:#27ae60,color:#fff
```

### The Extension Composition Pattern

```mermaid
graph LR
    Skills -->|"inject knowledge"| Model["Claude Model"]
    Model -->|"calls"| Tools
    Tools -->|"validated by"| Hooks
    Hooks -->|"may spawn"| Agents
    Agents -->|"use"| MCP["MCP Servers"]
    MCP -->|"results flow to"| Model

    style Model fill:#ff6b6b,color:#fff
```

Extensions compose **around the model**, not around each other:
- Skills give the model knowledge
- The model uses tools based on that knowledge
- Hooks validate the model's tool usage
- Agents let the model delegate complex sub-tasks
- MCP provides access to external systems

---

## 10. Extension Point Comparison

```mermaid
mindmap
    root((Extension\nEcosystem))
        Plugins
            Container for all other extensions
            plugin.json manifest
            Installable packages
        Tools
            AI-invoked capabilities
            JSON Schema input
            AsyncGenerator execution
            Permission-gated
        Hooks
            Event-driven validation
            Shell or AI-powered
            Block/Allow/Modify
            Fire on tool use, stop, session
        Skills
            Prompt-based knowledge
            Markdown with YAML
            Triggered by pattern or slash
            No code execution
        Agents
            Specialized query loops
            Tool subset + custom prompt
            Background or foreground
            Isolated message history
        MCP Servers
            External processes
            JSON-RPC protocol
            OAuth authentication
            Dynamic tool discovery
        Commands
            User-facing slash actions
            Immediate execution
            Outside agentic loop
            TypeScript or Markdown

```

| Feature | Tools | Hooks | Skills | Agents | MCP | Commands |
|---|---|---|---|---|---|---|
| **Invoked by** | AI model | System events | User or AI | AI model | AI (via proxy) | User only |
| **Runs code** | Yes | Yes | No (prompt only) | Yes (query loop) | Yes (external) | Yes |
| **Has permissions** | Yes | N/A (IS the validator) | No | Inherited | Yes (via proxy) | No |
| **Can be blocked** | By hooks/perms | No | No | By parent perms | By hooks/perms | No |
| **Defined in** | TypeScript | JSON config | Markdown | Markdown | JSON config | TS or MD |
| **Scope** | Per-tool call | Per-event | Per-conversation | Per-task | Per-session | Per-invocation |
