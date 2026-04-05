---
title: "Claude Code CLI — Detailed Architecture"
scope: "System architecture, deployment, and cross-cutting concerns"
last-analyzed: "2026-04-02"
---

# Detailed Architecture

## 1. System Context Diagram

```mermaid
C4Context
    title System Context — Claude Code CLI

    Person(dev, "Developer", "Terminal user or IDE user")

    System(cc, "Claude Code CLI", "AI coding assistant")

    System_Ext(api, "Anthropic Messages API", "LLM inference")
    System_Ext(bedrock, "AWS Bedrock", "Cloud inference")
    System_Ext(vertex, "Google Vertex AI", "Cloud inference")
    System_Ext(foundry, "Azure Foundry", "Cloud inference")
    System_Ext(mcp, "MCP Servers", "External tools")
    System_Ext(fs, "Local Filesystem", "Code, configs")
    System_Ext(git, "Git / GitHub", "Version control")
    System_Ext(shell, "System Shell", "Command execution")
    System_Ext(claudeai, "claude.ai", "OAuth, bridge, sharing")
    System_Ext(otel, "Telemetry Backend", "Analytics, traces")

    Rel(dev, cc, "Interactive prompts, CLI args")
    Rel(cc, api, "HTTPS streaming")
    Rel(cc, bedrock, "HTTPS via AWS SDK")
    Rel(cc, vertex, "HTTPS via GCP SDK")
    Rel(cc, foundry, "HTTPS via Azure SDK")
    Rel(cc, mcp, "JSON-RPC stdio/SSE/HTTP")
    Rel(cc, fs, "Read/Write/Glob/Grep")
    Rel(cc, git, "git CLI, gh CLI")
    Rel(cc, shell, "Subprocess spawning")
    Rel(cc, claudeai, "OAuth, WebSocket bridge")
    Rel(cc, otel, "gRPC telemetry export")
```

## 2. Container Diagram

```mermaid
C4Container
    title Container Diagram — Claude Code CLI

    Person(user, "Developer")

    System_Boundary(cli, "Claude Code CLI Process") {
        Container(entrypoint, "Entrypoint Layer", "TypeScript/Bun", "Fast-path routing, init pipeline")
        Container(repl, "REPL Screen", "React/Ink", "Interactive terminal UI")
        Container(queryEngine, "Query Engine", "TypeScript", "Core conversation loop, streaming, tool dispatch")
        Container(toolSystem, "Tool System", "TypeScript", "30+ built-in tools + MCP bridge")
        Container(permSystem, "Permission System", "TypeScript", "Rule evaluation, hook execution, user prompts")
        Container(stateStore, "State Store", "TypeScript", "AppState + Bootstrap state singletons")
        Container(compactor, "Compaction Engine", "TypeScript", "Auto/micro/reactive/snip compaction")
        Container(pluginSystem, "Plugin System", "TypeScript", "Plugin/skill/agent loading and execution")
        Container(apiClient, "API Client", "TypeScript", "Multi-provider client factory with retry")
        Container(mcpManager, "MCP Manager", "TypeScript", "Server lifecycle, tool bridging")
        Container(bridgeClient, "Bridge Client", "TypeScript", "WebSocket to claude.ai")
    }

    System_Ext(anthropicAPI, "Anthropic API")
    System_Ext(mcpServers, "MCP Servers")
    System_Ext(filesystem, "Filesystem")
    System_Ext(shellProc, "Shell")

    Rel(user, repl, "Keyboard input")
    Rel(repl, queryEngine, "User messages")
    Rel(queryEngine, apiClient, "API requests")
    Rel(queryEngine, toolSystem, "Tool dispatch")
    Rel(queryEngine, compactor, "Context management")
    Rel(toolSystem, permSystem, "Permission checks")
    Rel(toolSystem, filesystem, "File operations")
    Rel(toolSystem, shellProc, "Command execution")
    Rel(toolSystem, mcpManager, "MCP tool calls")
    Rel(apiClient, anthropicAPI, "HTTPS streaming")
    Rel(mcpManager, mcpServers, "JSON-RPC")
    Rel(pluginSystem, toolSystem, "Register tools")
    Rel(stateStore, repl, "State updates")
```

## 3. Component Diagram — Query Engine

```mermaid
C4Component
    title Component Diagram — Query Engine

    Container_Boundary(qe, "Query Engine") {
        Component(queryLoop, "queryLoop()", "AsyncGenerator", "Main conversation loop: stream → tools → continue/stop")
        Component(queryConfig, "QueryConfig", "Module", "Snapshot of env/feature state at query start")
        Component(tokenBudget, "TokenBudget", "Module", "Track cumulative output tokens, enforce budget")
        Component(streamExecutor, "StreamingToolExecutor", "Class", "Parallel tool execution with concurrency control")
        Component(toolOrch, "toolOrchestration", "Module", "runTools(): orchestrate parallel/sequential tool calls")
        Component(stopHooks, "StopHooks", "Module", "Execute Stop event hooks before terminating")
        Component(attachments, "Attachments", "Module", "Memory/CLAUDE.md injection, attachment dedup")
    }

    Component_Ext(apiClient, "API Client", "Streaming messages")
    Component_Ext(toolSystem, "Tool System", "Tool execution")
    Component_Ext(compactor, "Compaction Engine", "Context management")

    Rel(queryLoop, queryConfig, "Read config")
    Rel(queryLoop, apiClient, "API call per iteration")
    Rel(queryLoop, streamExecutor, "Dispatch tool uses")
    Rel(queryLoop, tokenBudget, "Check budget")
    Rel(queryLoop, compactor, "Trigger compaction")
    Rel(queryLoop, stopHooks, "On stop_reason=end_turn")
    Rel(queryLoop, attachments, "Inject memories")
    Rel(streamExecutor, toolOrch, "Run tools")
    Rel(toolOrch, toolSystem, "Execute individual tools")
```

## 4. Deployment Topology

```mermaid
flowchart TB
    subgraph LocalMachine["Developer's Machine"]
        BIN["bin/claude-haha (shell wrapper)"]
        BUN["Bun Runtime"]
        CLI["Claude Code CLI Process"]
        SANDBOX["macOS Seatbelt / Linux seccomp"]
        SHELL["Shell subprocesses"]
        MCP_LOCAL["Local MCP Servers (stdio)"]
    end

    subgraph CloudProviders["Cloud API Providers"]
        ANTHROPIC["Anthropic API\n(api.anthropic.com)"]
        BEDROCK["AWS Bedrock\n(bedrock-runtime.*.amazonaws.com)"]
        VERTEX["Google Vertex AI\n(*.aiplatform.googleapis.com)"]
        FOUNDRY["Azure Foundry\n(*.services.ai.azure.com)"]
    end

    subgraph RemoteInfra["Remote Infrastructure (optional)"]
        CCR["Claude Code Remote\n(Cloud Container)"]
        BRIDGE["claude.ai Bridge\n(WebSocket)"]
        OTEL["Telemetry Backend\n(gRPC)"]
    end

    subgraph ExternalMCP["Remote MCP Servers"]
        MCP_SSE["SSE MCP Servers"]
        MCP_HTTP["HTTP MCP Servers"]
    end

    BIN --> BUN
    BUN --> CLI
    CLI --> SANDBOX
    SANDBOX --> SHELL
    CLI --> MCP_LOCAL
    CLI --> ANTHROPIC
    CLI --> BEDROCK
    CLI --> VERTEX
    CLI --> FOUNDRY
    CLI --> BRIDGE
    CLI --> OTEL
    CLI --> MCP_SSE
    CLI --> MCP_HTTP
    CCR --> ANTHROPIC
```

## 5. Technology Stack Table

| Layer | Technology | Purpose | Alternatives to Consider |
|-------|-----------|---------|-------------------------|
| **Runtime** | Bun 1.x | JavaScript/TypeScript execution, bundling | Node.js, Deno |
| **Language** | TypeScript (ESNext target) | Type-safe application code | — |
| **UI Framework** | React 19 + Custom Ink | Declarative terminal UI | Blessed, terminal-kit, raw ANSI |
| **Layout Engine** | Yoga (via yoga-layout) | Flexbox-based terminal layout | Manual column calculation |
| **CLI Parsing** | Commander.js v14 | Argument/option parsing | yargs, oclif, clipanion |
| **API Client** | @anthropic-ai/sdk | Anthropic Messages API | Raw HTTP, axios |
| **Cloud SDKs** | @anthropic-ai/bedrock-sdk, vertex-sdk, foundry-sdk | Multi-cloud inference | Direct REST calls |
| **MCP** | @modelcontextprotocol/sdk | MCP client for tool servers | Custom JSON-RPC client |
| **Schema Validation** | Zod v4 | Runtime type validation | Joi, Yup, AJV |
| **Markdown** | marked | Markdown parsing for rendering | remark, markdown-it |
| **Syntax Highlighting** | Custom (HighlightedCode) | Code block colorization | Shiki, Prism |
| **HTTP** | axios, fetch | HTTP requests (web fetch, OAuth) | got, undici |
| **YAML** | yaml | YAML parsing for configs | js-yaml |
| **JSON** | jsonc-parser | JSON with comments parsing | json5 |
| **Telemetry** | @opentelemetry/* | Distributed tracing, metrics, logs | Custom logging |
| **Feature Flags** | GrowthBook | A/B testing, feature gating | LaunchDarkly, Unleash |
| **Secure Storage** | OS Keychain (via keytar) | OAuth token storage | File-based encryption |
| **Sandbox** | macOS Seatbelt, Linux seccomp | Process isolation | Docker, Firejail |
| **Diff** | Custom diff engine + color-diff | File change visualization | diff, jsdiff |
| **Process Mgmt** | Bun subprocess API | Shell command execution | child_process, execa |

## 6. Cross-Cutting Concerns

### Authentication & Authorization Model

**Authentication** supports four providers:
1. **Claude.ai OAuth** — For consumer users. OAuth 2.0 PKCE flow with claude.ai. Tokens stored in OS keychain. Scopes: `user:profile`, `user:inference`, `user:sessions:claude_code`, `user:mcp_servers`, `user:file_upload`.
2. **API Key** — Direct Anthropic API key via `ANTHROPIC_API_KEY` env var or keychain.
3. **AWS Bedrock** — IAM credentials via AWS SDK credential chain. Supports session tokens and bearer tokens.
4. **Google Vertex AI** — Google Cloud IAM via `google-auth-library`. Supports service accounts and ADC.
5. **Azure Foundry** — Azure AD token via `@azure/identity` DefaultAzureCredential, or API key.

**Authorization** is the permission system:
- **Permission Modes:** `default` (ask for destructive ops), `acceptEdits` (auto-allow file edits), `plan` (read-only), `bypassPermissions` (allow all), `auto` (ML classifier), `dontAsk` (deny what isn't explicitly allowed).
- **Rule Sources** (highest to lowest precedence): policy settings, CLI args, user settings, project settings, local settings, session rules.
- **Rule Types:** `allow` (auto-approve), `deny` (auto-reject), `ask` (prompt user). Rules can be tool-specific with pattern matching (e.g., `Bash(git *)` allows git commands).

### Observability

- **Telemetry Events:** `logEvent()` sends structured events for session start, tool use, API calls, errors, feature usage.
- **OpenTelemetry:** Full OTLP integration with traces (API calls, tool execution), metrics (token counts, latency), and logs.
- **Debug Logging:** `logForDebugging()` writes to stderr when `CLAUDE_CODE_DEBUG` is set.
- **Diagnostics:** `/doctor` command checks system health (API connectivity, tool availability, config validity).
- **Session Recording:** Conversation messages and tool results are persisted to `~/.claude/sessions/` for session resume.

### Configuration Management

Settings are loaded from multiple sources with defined precedence:
1. **CLI arguments** — Highest precedence for session options
2. **Environment variables** — `ANTHROPIC_*`, `CLAUDE_CODE_*`, etc.
3. **User settings** — `~/.claude/settings.json`
4. **Project settings** — `.claude/settings.json` in project root
5. **Local settings** — `.claude/settings.local.json` (gitignored)
6. **MDM settings** — macOS managed preferences (enterprise deployment)
7. **Remote managed settings** — Server-pushed configuration
8. **Policy limits** — Server-enforced guardrails

Configuration files use JSONC (JSON with comments) format.

### Secret Management

- OAuth tokens: Stored in OS keychain (macOS Keychain, Windows Credential Manager, Linux libsecret)
- API keys: Environment variables or keychain
- No secrets in config files (`.env` is gitignored)
- Custom `ANTHROPIC_API_KEY_HELPER` command support for enterprise secret managers
- MCP server credentials: Per-server OAuth or API key configuration

### Feature Flags

- **Build-time flags:** `feature('FLAG')` from `bun:bundle` — code is eliminated at bundle time for external builds
- **Runtime flags:** GrowthBook SDK — evaluated at startup, cached for session duration
- **Policy flags:** Server-pushed limits on tool availability, model access, etc.

## 7. Scalability & Performance Notes

### Startup Performance
- Shell wrapper does zero module loading for `--version`
- MDM settings and keychain reads fire during module evaluation (parallelized)
- OpenTelemetry gRPC exporter (~700KB) is lazy-loaded
- TCP pre-connection to API endpoint starts before first request
- OAuth token refresh is async and non-blocking

### Context Window Management
- **Auto-compact:** Triggers when context approaches 80% of model's context window
- **Micro-compact:** Time-based clearing of old tool results within a turn
- **Reactive compact:** Triggers on prompt-too-long API errors
- **Snip compact:** Targeted removal of low-value messages
- **Tool result budget:** Large tool outputs are persisted to disk and replaced with summaries

### Connection Handling
- API client uses configurable timeout (default 600s / 10 minutes)
- Retry with exponential backoff for transient errors (overloaded, network errors)
- Fallback model support (e.g., Sonnet → Haiku on quota exhaustion)
- Proxy support via `HTTPS_PROXY` / `HTTP_PROXY` environment variables
- mTLS support for enterprise deployments

### Rate Limiting
- Client-side rate limit tracking from API response headers
- Backoff with user notification on rate limits
- Usage tracking per model for cost monitoring
- Task budget system for capping total token spend per agentic turn

### Parallel Tool Execution
- Tools marked `isConcurrencySafe` can execute in parallel
- `StreamingToolExecutor` manages concurrent tool dispatch
- File operations are serialized when they target the same path
- Agent spawning creates independent task trees
