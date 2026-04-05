---
title: "Claude Code CLI — Testing Strategy"
scope: "Test pyramid, critical scenarios, and mocking approach"
last-analyzed: "2026-04-02"
---

# Testing Strategy

## 1. Test Pyramid Breakdown

### Unit Tests
- **Scope:** Individual functions, utility modules, schema validation, tool input processing
- **What they cover:**
  - Permission rule evaluation logic
  - Message normalization and formatting
  - Token counting and estimation
  - Shell command parsing and safety checking
  - Settings merging and migration
  - Tool input validation
  - Zod schema validation
  - String/text utilities (ANSI handling, width calculation, text wrapping)

### Integration Tests
- **Scope:** Module interactions, tool execution with mocked API, query loop behavior
- **What they cover:**
  - Query loop with mock API responses (tool use → result → continuation)
  - Compaction trigger and recovery
  - Permission system with layered rules
  - Hook execution pipeline
  - MCP tool bridging
  - Plugin loading and validation
  - Settings file reading and merging

### End-to-End Tests
- **Scope:** Full CLI invocation, session flows
- **What they cover:**
  - CLI argument parsing and routing
  - Session start → query → response → exit
  - Session resume from persisted state
  - Command execution (/help, /config, /model, etc.)
  - Error recovery (API failures, permission denials)

## 2. Critical Test Scenarios

These tests MUST pass for correctness:

### Message Integrity
1. Every `tool_use` block has a corresponding `tool_result` after normalization
2. Thinking blocks are never the last block in an assistant message
3. Thinking blocks are preserved across the assistant trajectory
4. Messages alternate between user and assistant roles in API payloads
5. Compacted messages maintain valid message structure

### Permission Safety
1. Policy deny rules cannot be overridden by lower-precedence allow rules
2. `plan` mode blocks all write operations
3. `bypassPermissions` mode allows all operations
4. Hook deny decisions are respected (exit code 2)
5. Denial tracking prevents infinite deny loops

### Tool Execution
1. Bash tool respects sandbox restrictions
2. File operations are validated against working directory scope
3. Concurrent tool execution doesn't corrupt shared state
4. Tool result size limits are enforced
5. Abort signals properly cancel running tools

### Context Management
1. Auto-compact triggers at correct token threshold
2. Compacted messages are valid API input
3. Tool results from pending turns are preserved during compaction
4. Token budget tracking is accurate across compaction boundaries

### API Client
1. Retry logic respects backoff intervals
2. OAuth token refresh doesn't deadlock with concurrent requests
3. Provider switching (Anthropic → Bedrock → Vertex) uses correct auth
4. Streaming reconnection handles partial messages correctly

## 3. Mocking & Faking Strategy

### API Layer Mocks
- **Mock Anthropic Client:** Returns predefined streaming responses
- **Behavior:** Simulates `message_start`, `content_block_start/delta/stop`, `message_delta`, `message_stop` events
- **Used in:** Query loop tests, compaction tests, tool execution tests

### Tool Mocks
- **Mock Tools:** Simplified tools that return predefined results
- **Behavior:** Skip actual file I/O, shell execution, network calls
- **Used in:** Tool orchestration tests, permission tests

### Filesystem Mocks
- **In-memory filesystem:** Simulates file read/write/glob/grep without disk access
- **Used in:** File tool tests, settings tests, plugin loading tests

### Permission Mocks
- **Mock canUseTool:** Returns predefined allow/deny decisions
- **Used in:** Query loop tests, tool execution tests

### MCP Server Mocks
- **Mock MCP Server:** In-process JSON-RPC handler
- **Behavior:** Responds to tools/list, tools/call without subprocess
- **Used in:** MCP integration tests

### Time Mocks
- **Fake timers:** Control time progression for timeout and interval tests
- **Used in:** Compaction timing tests, rate limit tests, sleep tool tests

## 4. Test Data & Fixtures

### Message Fixtures
- Predefined conversation sequences with various message types
- Tool use / tool result pairs for each built-in tool
- Multi-turn conversations with compaction points
- Messages with thinking blocks, images, documents

### Settings Fixtures
- Valid settings files with various permission configurations
- Invalid settings files for error handling tests
- Migration test fixtures (old format → new format)

### Plugin Fixtures
- Valid plugin manifests with all component types
- Invalid manifests for validation error tests
- DXT package fixtures

### API Response Fixtures
- Streaming response sequences for common scenarios
- Error response fixtures (rate limit, prompt too long, auth failure)
- Multi-tool-use response fixtures

## 5. Performance / Load Test Notes

### Startup Performance
- Tracked via profiling checkpoints (`profileCheckpoint()`)
- Target: REPL ready in < 2 seconds on warm start
- Measured: Time from shell invocation to first prompt display

### Streaming Throughput
- Characters-per-second rendering in terminal
- Token processing rate from API stream

### Context Window Pressure
- Tests with large conversation histories (approaching context limits)
- Compaction trigger timing and message count reduction
- Memory usage during large sessions

### Concurrent Tool Execution
- Multiple parallel tool calls (8+ concurrent)
- Sub-agent spawning under load
- File system contention with concurrent reads/writes

### Session Size
- Session persistence performance with 1000+ messages
- Resume speed for large sessions
- JSONL parsing performance
