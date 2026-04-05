---
title: "Claude Code CLI — Documentation Index"
scope: "Complete documentation set for re-implementation"
last-analyzed: "2026-04-02"
---

# Claude Code CLI — Documentation Index

Complete reverse-engineering documentation for the Claude Code CLI system. These documents provide everything needed to re-implement the system from scratch in any programming language.

## Start Here

| Document | Description |
|----------|-------------|
| [blueprint.md](./blueprint.md) | **Read first.** Project identity, high-level architecture, tech decisions, module map, glossary, conventions, and critical re-implementation warnings. |
| [reimplementation-guide.md](./reimplementation-guide.md) | Step-by-step playbook: implementation order, minimum viable skeleton, interface contracts to lock first, difficulty rankings, traps & pitfalls, verification checklist. |

## Architecture & Design

| Document | Description |
|----------|-------------|
| [architecture.md](./architecture.md) | System context, container, and component diagrams (Mermaid C4). Deployment topology. Technology stack. Cross-cutting concerns (auth, observability, config, secrets, feature flags). |
| [state-machines.md](./state-machines.md) | All stateful entities: query loop, permission decisions, task lifecycle, MCP connections, REPL input, speculation, bridge, OAuth tokens. Full state diagrams and transition tables. |
| [dependency-map.md](./dependency-map.md) | Internal module dependency graph. External dependency catalog with behavior descriptions. Dependency risk notes for re-implementation. |

## Data & Interfaces

| Document | Description |
|----------|-------------|
| [data-models.md](./data-models.md) | Entity-relationship diagram. Entity catalog (Message, AppState, ToolUseContext, Settings, Task, MCP). Schema evolution/migrations. Runtime data structures. Serialization contracts (API, session, settings, MCP, SSE). |
| [api-contracts.md](./api-contracts.md) | Anthropic Messages API contract. Tool system interface. Query loop parameters. MCP protocol. Key interaction sequences (Mermaid). Hook event contracts. |
| [config-and-env.md](./config-and-env.md) | Complete environment variable catalog (50+ vars). Configuration file formats. Feature flags (build-time and runtime). Secrets management. |

## Business Logic

| Document | Description |
|----------|-------------|
| [core-flows.md](./core-flows.md) | Critical workflows with step-by-step walkthroughs, flowcharts, and sequence diagrams: startup, query loop, tool execution, compaction, authentication, multi-agent, plugin loading, session resume, remote bridge. |

## Quality & Security

| Document | Description |
|----------|-------------|
| [testing-strategy.md](./testing-strategy.md) | Test pyramid breakdown. Critical test scenarios. Mocking & faking strategy. Test data & fixtures. Performance test notes. |
| [security-model.md](./security-model.md) | Authentication flow (5 providers). Authorization model (modes, rules, precedence). Data protection. Input validation. Known attack surfaces and defenses. |

## Module Deep Dives

Detailed documentation for each major subsystem:

| Document | Scope |
|----------|-------|
| [module-deep-dives/query-engine.md](./module-deep-dives/query-engine.md) | Core conversation loop (`src/query.ts`): streaming, tool dispatch, compaction, recovery |
| [module-deep-dives/tool-system.md](./module-deep-dives/tool-system.md) | Tool interface, 30+ built-in tools, orchestration (`src/Tool.ts`, `src/tools/`) |
| [module-deep-dives/permission-system.md](./module-deep-dives/permission-system.md) | Rule evaluation, modes, hooks, denial tracking (`src/utils/permissions/`, `src/hooks/`) |
| [module-deep-dives/api-client.md](./module-deep-dives/api-client.md) | Multi-provider client, streaming, retry, fallback (`src/services/api/`) |
| [module-deep-dives/terminal-ui.md](./module-deep-dives/terminal-ui.md) | Custom Ink framework: React reconciler, Yoga layout, ANSI, events (`src/ink/`) |
| [module-deep-dives/multi-agent.md](./module-deep-dives/multi-agent.md) | Task system, agent spawning, context isolation (`src/tasks/`, `src/tools/AgentTool/`) |
| [module-deep-dives/mcp-integration.md](./module-deep-dives/mcp-integration.md) | MCP server lifecycle, tool bridging, transports (`src/services/mcp/`) |
| [module-deep-dives/compaction-engine.md](./module-deep-dives/compaction-engine.md) | Auto/reactive/micro/snip compaction strategies (`src/services/compact/`) |
| [module-deep-dives/plugin-skill-system.md](./module-deep-dives/plugin-skill-system.md) | Plugin/skill/agent loading and execution (`src/plugins/`, `src/skills/`) |
| [module-deep-dives/settings-config.md](./module-deep-dives/settings-config.md) | Settings hierarchy, merging, validation, migrations (`src/utils/settings/`) |
| [module-deep-dives/commands-system.md](./module-deep-dives/commands-system.md) | 88+ slash commands: registration, lazy loading, filtering, types (`src/commands/`) |
