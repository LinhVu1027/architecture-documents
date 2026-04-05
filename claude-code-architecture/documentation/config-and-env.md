---
title: "Claude Code CLI — Configuration & Environment"
scope: "All environment variables, config files, feature flags, and secrets"
last-analyzed: "2026-04-02"
---

# Configuration & Environment

## 1. Complete Environment Variable Catalog

### Authentication

| Name | Type | Default | Required | Description | Module |
|------|------|---------|----------|-------------|--------|
| `ANTHROPIC_API_KEY` | string | — | Yes (if not using OAuth/cloud) | Direct API authentication key | services/api/client |
| `ANTHROPIC_AUTH_TOKEN` | string | — | No | Bearer token for API access (alternative to API key) | services/api/client |
| `ANTHROPIC_API_KEY_HELPER` | string | — | No | Command that outputs API key (for secret managers) | utils/auth |

### API Configuration

| Name | Type | Default | Required | Description | Module |
|------|------|---------|----------|-------------|--------|
| `ANTHROPIC_BASE_URL` | string | `https://api.anthropic.com` | No | Override API endpoint | services/api/client |
| `API_TIMEOUT_MS` | number | `600000` (10 min) | No | API request timeout in milliseconds | services/api/client |
| `ANTHROPIC_CUSTOM_HEADERS` | string | — | No | Additional headers (newline-separated `Name: Value`) | services/api/client |

### Model Selection

| Name | Type | Default | Required | Description | Module |
|------|------|---------|----------|-------------|--------|
| `ANTHROPIC_DEFAULT_HAIKU_MODEL` | string | — | No | Override default Haiku model ID | utils/model |
| `ANTHROPIC_DEFAULT_OPUS_MODEL` | string | — | No | Override default Opus model ID | utils/model |
| `ANTHROPIC_DEFAULT_SONNET_MODEL` | string | — | No | Override default Sonnet model ID | utils/model |
| `ANTHROPIC_SMALL_FAST_MODEL` | string | — | No | Override small fast model (used for compaction, etc.) | utils/model |

### AWS Bedrock

| Name | Type | Default | Required | Description | Module |
|------|------|---------|----------|-------------|--------|
| `CLAUDE_CODE_USE_BEDROCK` | boolean | `false` | No | Enable Bedrock as API provider | services/api/client |
| `AWS_REGION` / `AWS_DEFAULT_REGION` | string | `us-east-1` | No | AWS region for Bedrock | services/api/client |
| `ANTHROPIC_SMALL_FAST_MODEL_AWS_REGION` | string | — | No | Override region for Haiku on Bedrock | services/api/client |
| `AWS_BEARER_TOKEN_BEDROCK` | string | — | No | Bearer token auth for Bedrock | services/api/client |
| `CLAUDE_CODE_SKIP_BEDROCK_AUTH` | boolean | `false` | No | Skip Bedrock auth (for proxies) | services/api/client |

### Google Vertex AI

| Name | Type | Default | Required | Description | Module |
|------|------|---------|----------|-------------|--------|
| `CLAUDE_CODE_USE_VERTEX` | boolean | `false` | No | Enable Vertex as API provider | services/api/client |
| `ANTHROPIC_VERTEX_PROJECT_ID` | string | — | Yes (if Vertex) | GCP project ID | services/api/client |
| `CLOUD_ML_REGION` | string | `us-east5` | No | Default GCP region | services/api/client |
| `VERTEX_REGION_CLAUDE_*` | string | — | No | Per-model region overrides | services/api/client |
| `CLAUDE_CODE_SKIP_VERTEX_AUTH` | boolean | `false` | No | Skip Vertex auth (for proxies) | services/api/client |

### Azure Foundry

| Name | Type | Default | Required | Description | Module |
|------|------|---------|----------|-------------|--------|
| `CLAUDE_CODE_USE_FOUNDRY` | boolean | `false` | No | Enable Foundry as API provider | services/api/client |
| `ANTHROPIC_FOUNDRY_RESOURCE` | string | — | Yes (if Foundry) | Azure resource name | services/api/client |
| `ANTHROPIC_FOUNDRY_BASE_URL` | string | — | No | Full base URL (alternative to resource) | services/api/client |
| `ANTHROPIC_FOUNDRY_API_KEY` | string | — | No | Foundry API key (alternative to Azure AD) | services/api/client |
| `CLAUDE_CODE_SKIP_FOUNDRY_AUTH` | boolean | `false` | No | Skip Foundry auth | services/api/client |

### Behavior & Features

| Name | Type | Default | Required | Description | Module |
|------|------|---------|----------|-------------|--------|
| `CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC` | boolean | `false` | No | Reduce non-essential network calls | various |
| `DISABLE_TELEMETRY` | boolean | `false` | No | Disable all telemetry | services/analytics |
| `CLAUDE_CODE_ADDITIONAL_PROTECTION` | boolean | `false` | No | Enable additional safety protections | services/api/client |
| `CLAUDE_CODE_DEBUG` | boolean | `false` | No | Enable debug logging to stderr | utils/debug |
| `CLAUDE_CODE_FORCE_RECOVERY_CLI` | boolean | `false` | No | Force fallback readline REPL | entrypoints/cli |

### Remote / Container

| Name | Type | Default | Required | Description | Module |
|------|------|---------|----------|-------------|--------|
| `CLAUDE_CODE_REMOTE` | boolean | `false` | No | Running in remote container (CCR) | entrypoints/cli |
| `CLAUDE_CODE_CONTAINER_ID` | string | — | No | Remote container ID | services/api/client |
| `CLAUDE_CODE_REMOTE_SESSION_ID` | string | — | No | Remote session ID | services/api/client |

### SDK / Programmatic

| Name | Type | Default | Required | Description | Module |
|------|------|---------|----------|-------------|--------|
| `CLAUDE_AGENT_SDK_CLIENT_APP` | string | — | No | SDK consumer app identifier | services/api/client |

### Build & Version

| Name | Type | Default | Required | Description | Module |
|------|------|---------|----------|-------------|--------|
| `CLAUDE_CODE_LOCAL_VERSION` | string | `999.0.0-local` | No | Override version string | preload |
| `CLAUDE_CODE_LOCAL_PACKAGE_URL` | string | `claude-code-local` | No | Override package URL | preload |
| `CLAUDE_CODE_LOCAL_BUILD_TIME` | string | Current time | No | Override build timestamp | preload |

### OAuth (Internal)

| Name | Type | Default | Required | Description | Module |
|------|------|---------|----------|-------------|--------|
| `USER_TYPE` | string | — | No | User type (e.g., `ant` for internal) | constants/oauth |
| `USE_STAGING_OAUTH` | boolean | `false` | No | Use staging OAuth endpoints | constants/oauth |

### Proxy

| Name | Type | Default | Required | Description | Module |
|------|------|---------|----------|-------------|--------|
| `HTTPS_PROXY` / `HTTP_PROXY` | string | — | No | HTTP/S proxy URL | utils/proxy |
| `NO_PROXY` | string | — | No | Proxy bypass list | utils/proxy |
| `NODE_EXTRA_CA_CERTS` | string | — | No | Additional CA certificates | entrypoints/init |

## 2. Configuration File Formats

### `~/.claude/settings.json` (User Settings)

Global user preferences. JSONC format.

```
{
  // Permission configuration
  "permissions": {
    "allow": ["Bash(git *)", "Read", "Glob", "Grep"],
    "deny": ["Bash(rm -rf *)"],
    "defaultMode": "default",
    "additionalDirectories": ["/shared/project"]
  },

  // Environment variable overrides
  "env": {
    "ANTHROPIC_API_KEY": "sk-..."
  },

  // Model preference
  "model": "claude-sonnet-4-6",

  // Hook definitions
  "hooks": {
    "PreToolUse": [...],
    "PostToolUse": [...],
    "Stop": [...],
    "SessionStart": [...]
  },

  // MCP server configurations
  "mcpServers": {
    "server-name": {
      "command": "npx",
      "args": ["-y", "@server/mcp"],
      "env": {"API_KEY": "..."}
    }
  },

  // UI preferences
  "theme": "dark",
  "verbose": false,
  "autoUpdates": true
}
```

### `.claude/settings.json` (Project Settings)

Per-project configuration. Same schema as user settings. Checked into version control.

### `.claude/settings.local.json` (Local Settings)

Per-project, per-machine configuration. Same schema. Gitignored.

### `CLAUDE.md` (Project Instructions)

Markdown file with optional YAML frontmatter. Loaded as part of system prompt.

```
---
description: "Project instructions for Claude Code"
---

# Build Instructions
- Run `npm install` to install dependencies
- Run `npm test` to run tests
- Run `npm run build` to build

# Code Conventions
- Use TypeScript strict mode
- Follow ESLint configuration
```

Searched in this order:
1. `CLAUDE.md` in project root
2. `.claude/CLAUDE.md` in project root
3. `CLAUDE.md` in each parent directory up to home
4. `~/.claude/CLAUDE.md` (global)

### `plugin.json` (Plugin Manifest)

```
{
  "name": "plugin-name",
  "version": "1.0.0",
  "description": "Plugin description",
  "author": "Author",
  "commands": ["commands/"],
  "skills": ["skills/"],
  "agents": ["agents/"],
  "hooks": {
    "PreToolUse": [...]
  },
  "mcpServers": {
    "server": {...}
  }
}
```

### `.mcp.json` (MCP Configuration)

Project-level MCP server configuration:
```
{
  "mcpServers": {
    "server-name": {
      "command": "node",
      "args": ["./mcp-server.js"],
      "env": {}
    }
  }
}
```

### `~/.claude/keybindings.json` (Keybindings)

Custom keyboard shortcut definitions.

## 3. Feature Flags

### Build-Time Flags (Bun `feature()`)

| Flag | Default (External) | Behavior When On | Behavior When Off |
|------|-------------------|------------------|-------------------|
| `VOICE_MODE` | Off | Voice input UI and STT integration available | Voice code completely eliminated |
| `DUMP_SYSTEM_PROMPT` | Off | `--dump-system-prompt` flag available | Flag rejected |
| `CHICAGO_MCP` | Off | Computer use MCP server available | Code eliminated |
| `DAEMON` | Off | Background daemon mode available | Code eliminated |
| `BG_SESSIONS` | Off | `ps`, `logs`, `attach`, `kill` commands | Code eliminated |
| `BRIDGE_MODE` | Off | Remote bridge to claude.ai | Code eliminated |
| `REACTIVE_COMPACT` | Off | Reactive compaction on prompt_too_long | Standard error handling |
| `CONTEXT_COLLAPSE` | Off | Context collapse optimization | Standard compaction |
| `HISTORY_SNIP` | Off | Snip compaction strategy | Standard compaction |
| `TRANSCRIPT_CLASSIFIER` | Off | `auto` permission mode with ML classifier | Mode unavailable |
| `EXPERIMENTAL_SKILL_SEARCH` | Off | Dynamic skill search in prompts | Static skill list |
| `TEMPLATES` | Off | Template job system | Code eliminated |
| `TOKEN_BUDGET` | Off | Per-turn token budget tracking | Unlimited turns |
| `ENABLE_AGENT_SWARMS` | Off | Team/swarm multi-agent feature | Code eliminated |

### Runtime Flags (GrowthBook)

Runtime flags are evaluated at session start and cached. They control A/B experiments and gradual rollouts. Specific flag names are dynamic and change with each release.

## 4. Secrets

| Secret | Storage | Rotation | Description |
|--------|---------|----------|-------------|
| Anthropic API Key | Environment variable or keychain | Manual | Direct API authentication |
| OAuth Access Token | OS Keychain | Auto (refresh token flow) | claude.ai user authentication |
| OAuth Refresh Token | OS Keychain | On re-login | Long-lived token for token refresh |
| AWS Credentials | AWS credential chain | Per AWS config | Bedrock IAM authentication |
| GCP Credentials | ADC or service account | Per GCP config | Vertex IAM authentication |
| Azure AD Token | DefaultAzureCredential | Auto | Foundry Azure AD authentication |
| MCP Server API Keys | Per-server env config | Manual | Third-party MCP server auth |

**Security notes:**
- Secrets are NEVER written to settings files (settings files are validated to not contain secret patterns)
- The keychain access can fail on headless/SSH systems — fallback to env vars
- `ANTHROPIC_API_KEY_HELPER` supports integration with enterprise secret managers (Vault, AWS Secrets Manager)
- Settings files support `env` blocks for environment variables, but these are NOT encrypted
