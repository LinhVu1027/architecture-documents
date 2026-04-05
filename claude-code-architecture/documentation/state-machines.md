---
title: "Claude Code CLI — State Machines"
scope: "All stateful entities and their transitions"
last-analyzed: "2026-04-02"
---

# State Machines

## 1. Query Loop State Machine

The core conversation loop transitions between states based on API responses and tool execution.

```mermaid
stateDiagram-v2
    [*] --> Idle: Session starts

    Idle --> PreparingRequest: User submits message
    PreparingRequest --> Streaming: API request sent

    Streaming --> ProcessingToolUse: stop_reason = tool_use
    Streaming --> StopHooks: stop_reason = end_turn
    Streaming --> MaxTokenRecovery: stop_reason = max_tokens
    Streaming --> ErrorHandling: API error

    ProcessingToolUse --> PermissionCheck: For each tool use
    PermissionCheck --> ToolExecution: Permitted
    PermissionCheck --> ToolDenied: Denied
    ToolExecution --> ToolComplete: Tool finishes
    ToolDenied --> ToolComplete: Denial message created
    ToolComplete --> PreparingRequest: More tools or continue
    ToolComplete --> CompactionCheck: All tools done

    CompactionCheck --> PreparingRequest: Context OK
    CompactionCheck --> Compacting: Token threshold exceeded
    Compacting --> PreparingRequest: Messages compacted

    MaxTokenRecovery --> PreparingRequest: Recovery count < 3
    MaxTokenRecovery --> StopHooks: Recovery exhausted

    StopHooks --> Idle: Hooks complete
    StopHooks --> PreparingRequest: Hook requests continuation

    ErrorHandling --> RetryWithBackoff: Transient error
    ErrorHandling --> ReactiveCompact: prompt_too_long
    ErrorHandling --> FallbackModel: Quota exhausted + fallback available
    ErrorHandling --> Idle: Fatal error

    RetryWithBackoff --> Streaming: Retry attempt
    ReactiveCompact --> PreparingRequest: Context compacted
    FallbackModel --> PreparingRequest: Model switched
```

### Transition Table

| From State | Event | Guard | To State | Side Effects |
|-----------|-------|-------|----------|-------------|
| Idle | User submits message | — | PreparingRequest | Inject attachments, memories |
| PreparingRequest | — | — | Streaming | Normalize messages, send API request |
| Streaming | stop_reason=tool_use | — | ProcessingToolUse | Collect tool use blocks |
| Streaming | stop_reason=end_turn | — | StopHooks | — |
| Streaming | stop_reason=max_tokens | recovery < 3 | MaxTokenRecovery | Increment recovery counter |
| Streaming | API error (transient) | retries remaining | RetryWithBackoff | Log error, increment retry |
| Streaming | API error (prompt_too_long) | — | ReactiveCompact | — |
| ProcessingToolUse | Permission check | allowed | ToolExecution | — |
| ProcessingToolUse | Permission check | denied | ToolDenied | Create denial tool_result |
| ToolComplete | All tools done | tokens OK | PreparingRequest | Assemble tool results |
| ToolComplete | All tools done | tokens high | Compacting | — |
| Compacting | Summary complete | — | PreparingRequest | Replace messages with summary |
| StopHooks | Hooks complete | no continuation | Idle | Yield terminal event |
| MaxTokenRecovery | recovery < 3 | — | PreparingRequest | Increase max_tokens |
| MaxTokenRecovery | recovery >= 3 | — | StopHooks | — |
| ErrorHandling | Transient error | retries left | RetryWithBackoff | Exponential backoff |
| ErrorHandling | Quota error | fallback model set | FallbackModel | Switch model |
| ErrorHandling | Fatal error | — | Idle | Yield error message |

## 2. Permission Decision State Machine

```mermaid
stateDiagram-v2
    [*] --> EvaluateRules: Tool permission check

    EvaluateRules --> PolicyCheck: Start evaluation
    PolicyCheck --> Denied: Policy denies
    PolicyCheck --> CLIArgCheck: No policy rule

    CLIArgCheck --> Allowed: CLI allows
    CLIArgCheck --> Denied: CLI denies
    CLIArgCheck --> UserSettingsCheck: No CLI rule

    UserSettingsCheck --> Allowed: User allows
    UserSettingsCheck --> Denied: User denies
    UserSettingsCheck --> ProjectSettingsCheck: No user rule

    ProjectSettingsCheck --> Allowed: Project allows
    ProjectSettingsCheck --> Denied: Project denies
    ProjectSettingsCheck --> LocalSettingsCheck: No project rule

    LocalSettingsCheck --> Allowed: Local allows
    LocalSettingsCheck --> Denied: Local denies
    LocalSettingsCheck --> SessionCheck: No local rule

    SessionCheck --> Allowed: Session allows
    SessionCheck --> HookEvaluation: No session rule

    HookEvaluation --> Allowed: Hook approves
    HookEvaluation --> Denied: Hook denies
    HookEvaluation --> ModeCheck: Hook defers

    ModeCheck --> Allowed: bypassPermissions mode
    ModeCheck --> Denied: plan mode + write tool
    ModeCheck --> Allowed: acceptEdits mode + edit tool
    ModeCheck --> ClassifierCheck: auto mode
    ModeCheck --> UserPrompt: default mode

    ClassifierCheck --> Allowed: Classifier approves
    ClassifierCheck --> UserPrompt: Classifier uncertain

    UserPrompt --> Allowed: User allows
    UserPrompt --> Denied: User denies
    UserPrompt --> AlwaysAllow: User allows always

    AlwaysAllow --> Allowed: Save rule + allow

    Allowed --> [*]
    Denied --> [*]
```

## 3. Task Lifecycle State Machine

```mermaid
stateDiagram-v2
    [*] --> Created: Task instantiated

    Created --> Running: Start execution
    Running --> Completed: Successful finish
    Running --> Error: Unrecoverable error
    Running --> Cancelled: User cancellation or parent abort

    Completed --> [*]
    Error --> [*]
    Cancelled --> [*]
```

### Task Types and Their States

| Task Type | Created By | Special States | Notes |
|-----------|-----------|----------------|-------|
| LocalMainSessionTask | REPL | — | Root task, lives for session duration |
| LocalAgentTask | Agent tool | Inherits parent's permission context | Can spawn child agents |
| LocalShellTask | Bash tool | Has guard conditions | Sandboxed execution |
| RemoteAgentTask | Remote bridge | Has connection states | WebSocket-dependent |
| InProcessTeammateTask | Team feature | Shared AppState | Parallel to main task |
| DreamTask | Auto-dream / /dream | Background mode | Low-priority context processing |

## 4. MCP Server Connection State Machine

```mermaid
stateDiagram-v2
    [*] --> Disconnected: Server configured

    Disconnected --> Connecting: Start connection
    Connecting --> Connected: Handshake success
    Connecting --> Error: Connection failed

    Connected --> Disconnected: Server closed
    Connected --> Error: Runtime error

    Error --> Connecting: Retry (with backoff)
    Error --> Disconnected: Max retries exceeded

    Connected --> [*]: Session ends (cleanup)
    Disconnected --> [*]: Session ends
    Error --> [*]: Session ends
```

### Transition Table

| From | Event | To | Side Effects |
|------|-------|----|-------------|
| Disconnected | Config available | Connecting | Spawn process (stdio) or open connection (SSE/HTTP) |
| Connecting | JSON-RPC initialize success | Connected | Register tools and resources |
| Connecting | Timeout or error | Error | Log error |
| Connected | Process exit | Disconnected | Remove tools from registry |
| Connected | JSON-RPC error | Error | Log error, remove tools |
| Error | Auto-retry | Connecting | Backoff delay |
| Error | Max retries | Disconnected | Warn user |

## 5. REPL Input State Machine

```mermaid
stateDiagram-v2
    [*] --> WaitingForInput: REPL ready

    WaitingForInput --> Composing: Key press
    Composing --> Composing: More key presses
    Composing --> WaitingForInput: Escape (clear)
    Composing --> Processing: Enter/Shift+Enter (submit)

    Processing --> Streaming: API call started
    Streaming --> Streaming: Receiving tokens
    Streaming --> ToolRunning: Tool execution started
    ToolRunning --> PermissionPrompt: Tool needs permission
    PermissionPrompt --> ToolRunning: User responds
    ToolRunning --> Streaming: Tool complete, continue
    Streaming --> WaitingForInput: Response complete

    Streaming --> WaitingForInput: User interrupts (Escape)
    ToolRunning --> WaitingForInput: User interrupts (Escape)
```

## 6. Speculation State Machine

```mermaid
stateDiagram-v2
    [*] --> Idle: Default state

    Idle --> Active: Prompt suggestion available
    Active --> Active: Processing speculative turn
    Active --> Idle: User submits different message
    Active --> Applied: User accepts suggestion

    Applied --> Idle: Speculated messages merged

    Active --> Idle: Speculation fails or times out
```

## 7. Bridge Connection State Machine

```mermaid
stateDiagram-v2
    [*] --> Disabled: Bridge not configured

    Disabled --> Registering: Bridge enabled
    Registering --> Ready: Environment registered
    Registering --> Error: Registration failed

    Ready --> Connected: WebSocket connected
    Ready --> Ready: Polling for connection

    Connected --> Ready: WebSocket closed
    Connected --> Reconnecting: WebSocket error
    Reconnecting --> Connected: Reconnection success
    Reconnecting --> Ready: Max reconnects exceeded

    Connected --> Disabled: Bridge disabled
    Ready --> Disabled: Bridge disabled
    Error --> Disabled: Bridge disabled
```

## 8. OAuth Token State Machine

```mermaid
stateDiagram-v2
    [*] --> NoToken: First run

    NoToken --> Authorizing: Login initiated
    Authorizing --> ValidToken: Authorization success
    Authorizing --> NoToken: Authorization cancelled/failed

    ValidToken --> ValidToken: Token used for API call
    ValidToken --> Refreshing: Token near expiry
    ValidToken --> Expired: Token expired

    Refreshing --> ValidToken: Refresh success
    Refreshing --> Expired: Refresh failed

    Expired --> Authorizing: Re-login required
```
