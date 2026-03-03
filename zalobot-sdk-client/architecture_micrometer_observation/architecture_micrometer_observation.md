# Architecture: Micrometer Observation API in the Spring Ecosystem

## Overview

Micrometer's Observation API is the **"SLF4J for observability"** — a vendor-neutral facade that lets you instrument code once and get metrics, distributed traces, and logs from that single instrumentation point. Think of it like a power strip: you plug your code into one observation, and multiple handlers (metrics, tracing, logging) draw from that single connection. Spring Framework, Spring Kafka, and Spring Boot all follow an identical 5-artifact pattern to integrate with it, making it straightforward to apply the same strategy to any library — including Zalobot SDK.

---

## Interface Hierarchy

```
io.micrometer.observation
├── Observation                          ← The unified instrumentation point
│   ├── .start() / .stop() / .error()   ← Lifecycle
│   ├── .openScope()                     ← ThreadLocal binding
│   ├── Context                          ← Mutable data holder
│   └── Scope                            ← ThreadLocal scope (AutoCloseable)
│
├── ObservationRegistry                  ← Central hub (holds handlers, conventions, filters)
│   └── ObservationConfig               ← Registration point for all plugins
│
├── ObservationHandler<T extends Context>         ← Visitor pattern for lifecycle events
│   ├── AllMatchingCompositeObservationHandler    ← Calls ALL matching handlers
│   └── FirstMatchingCompositeObservationHandler  ← Calls FIRST matching handler
│
├── ObservationConvention<T extends Context>      ← Naming + tagging strategy
│   └── GlobalObservationConvention<T>            ← Registered on registry (not per-observation)
│
├── ObservationFilter                    ← Mutate context before handlers on stop()
├── ObservationPredicate                 ← Enable/disable observations conditionally
│
├── docs/
│   └── ObservationDocumentation         ← Self-documenting enum interface
│
└── transport/
    ├── SenderContext<C>                  ← Outgoing (inject trace headers)
    ├── ReceiverContext<C>               ← Incoming (extract trace headers)
    ├── RequestReplySenderContext<Req,Res>   ← Request-reply outgoing
    ├── RequestReplyReceiverContext<Req,Res> ← Request-reply incoming
    └── Propagator                       ← Header injection/extraction
        ├── Setter<C>
        └── Getter<C>
```

### Method Table: Core Interfaces

| Interface | Key Methods | Purpose |
|-----------|-------------|---------|
| `Observation` | `start()`, `stop()`, `error(Throwable)`, `openScope()`, `event(Event)`, `lowCardinalityKeyValue(String,String)` | Single instrumentation point |
| `ObservationRegistry` | `observationConfig()`, `getCurrentObservation()` | Central handler/convention registry |
| `ObservationHandler<T>` | `onStart(T)`, `onStop(T)`, `onError(T)`, `onScopeOpened(T)`, `onScopeClosed(T)`, `supportsContext(Context)` | React to observation lifecycle |
| `ObservationConvention<T>` | `getName()`, `getContextualName(T)`, `getLowCardinalityKeyValues(T)`, `getHighCardinalityKeyValues(T)`, `supportsContext(Context)` | Define observation naming + tags |
| `ObservationDocumentation` | `getDefaultConvention()`, `getLowCardinalityKeyNames()`, `getHighCardinalityKeyNames()`, `observation(custom, default, contextSupplier, registry)` | Self-documenting observation definitions |

---

## ASCII Class Diagram

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        ObservationRegistry                              │
│  ─────────────────────────────────────────────────────────────────────  │
│  - observationConfig: ObservationConfig                                 │
│  - currentObservationScope: ThreadLocal<Scope>                          │
│  ─────────────────────────────────────────────────────────────────────  │
│  + observationConfig(): ObservationConfig                               │
│  + getCurrentObservation(): Observation                                 │
│  + static create(): ObservationRegistry                                 │
│  + static NOOP: ObservationRegistry                                     │
└─────────────┬───────────────────────────────────────────────────────────┘
              │ creates
              ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                          SimpleObservation                              │
│  ─────────────────────────────────────────────────────────────────────  │
│  - context: Context          - registry: ObservationRegistry            │
│  - convention: Convention    - defaultConvention: Convention             │
│  ─────────────────────────────────────────────────────────────────────  │
│  + start() → calls handler.onStart(context)                             │
│  + stop()  → calls filter.map(context) then handler.onStop(context)     │
│  + error() → calls handler.onError(context)                             │
│  + openScope() → calls handler.onScopeOpened(context)                   │
└─────────────────────────────────────────────────────────────────────────┘

┌──────────────────────────┐   ┌────────────────────────────────────────┐
│ DefaultMeterObservation  │   │ DefaultTracingObservationHandler       │
│ Handler (metrics)        │   │ (tracing)                              │
│ ──────────────────────── │   │ ────────────────────────────────────── │
│ onStart: Timer.Sample    │   │ onStart: create child Span             │
│ onStop:  record timer    │   │ onStop:  tag + end Span                │
│ onEvent: increment ctr   │   │ onScopeOpened: put Span in ThreadLocal │
└──────────────────────────┘   └────────────────────────────────────────┘

┌────────────────────────────────┐   ┌──────────────────────────────────┐
│ PropagatingSenderTracing       │   │ PropagatingReceiverTracing       │
│ ObservationHandler             │   │ ObservationHandler               │
│ ────────────────────────────── │   │ ──────────────────────────────── │
│ onStart: inject headers into   │   │ onStart: extract headers from    │
│   carrier via Propagator.Setter│   │   carrier via Propagator.Getter  │
│ Supports: SenderContext        │   │ Supports: ReceiverContext        │
└────────────────────────────────┘   └──────────────────────────────────┘
```

---

## State Diagram: Observation Lifecycle

```
                    ┌──────────────┐
                    │   CREATED    │  Observation.createNotStarted(...)
                    └──────┬───────┘
                           │ observation.start()
                           │ → handler.onStart(context)
                           ▼
                    ┌──────────────┐
              ┌─────│   STARTED    │──────────────────────┐
              │     └──────┬───────┘                      │
              │            │ observation.openScope()      │
              │            │ → handler.onScopeOpened()    │
              │            ▼                              │
              │     ┌──────────────┐                      │
              │     │  IN SCOPE    │ ThreadLocal active    │
              │     └──────┬───────┘                      │
              │            │ scope.close()                │
              │            │ → handler.onScopeClosed()    │
              │            ▼                              │
              │     ┌──────────────┐                      │
              │     │ SCOPE CLOSED │                      │
              │     └──────┬───────┘                      │
              │            │                              │
              ▼            ▼                              │
  observation.error(ex)    │                              │
  → handler.onError()      │                              │
              │            │                              │
              └────────────┤                              │
                           │ observation.stop()           │
                           │ → filter.map(context)        │
                           │ → handler.onStop(context)    │
                           ▼                              │
                    ┌──────────────┐                      │
                    │   STOPPED    │◄─────────────────────┘
                    └──────────────┘   (stop without scope)
```

---

## Flows Diagram

### Flow 1: HTTP Client Request (Spring Framework — RestClient)

```
User code: restClient.get().uri("/api/users").retrieve()
    │
    ▼
1. DefaultRestClient creates:
   context = new ClientRequestObservationContext(request)
    │
    ▼
2. Observation created from enum:
   ClientHttpObservationDocumentation.HTTP_CLIENT_EXCHANGES
     .observation(customConvention, DEFAULT_CONVENTION, () -> context, registry)
    │
    ▼
3. observation.start()
   ├── DefaultMeterObservationHandler.onStart() → Timer.Sample.start()
   └── PropagatingSenderTracingHandler.onStart()
       → Create child span
       → Inject trace headers into HTTP request via Propagator.Setter
    │
    ▼
4. try (Scope scope = observation.openScope()) { ... }
   → Span placed in ThreadLocal (visible to downstream code)
    │
    ▼
5. Execute HTTP request → get response
   context.setResponse(response)  // Store for tag extraction
    │
    ▼
6. observation.stop()
   ├── Convention extracts tags: method=GET, uri=/api/users, status=200, outcome=SUCCESS
   ├── DefaultMeterObservationHandler.onStop() → Record Timer with tags
   └── TracingHandler.onStop() → Tag span, end span
```

### Flow 2: Kafka Listener Processing (Spring Kafka)

```
Kafka Consumer receives record
    │
    ▼
1. KafkaMessageListenerContainer creates:
   context = new KafkaRecordReceiverContext(record, listenerId, clientId, groupId, clusterId)
   ↑ extends ReceiverContext — Propagator.Getter extracts trace headers from Kafka headers
    │
    ▼
2. Observation created from enum:
   KafkaListenerObservation.LISTENER_OBSERVATION
     .observation(customConvention, DEFAULT_CONVENTION, () -> context, registry)
    │
    ▼
3. observation.start()
   ├── DefaultMeterObservationHandler.onStart() → Timer.Sample
   └── PropagatingReceiverTracingHandler.onStart()
       → Extract trace context from Kafka message headers
       → Create child span linked to producer span
    │
    ▼
4. Scope scope = observation.openScope()
   → Span in ThreadLocal (visible to @KafkaListener method)
    │
    ▼
5. invokeOnMessage(record)  // User's listener processes the message
    │
    ▼
6. finally { observation.stop(); scope.close(); }
   ├── Convention extracts tags: listener.id, topic, partition, offset, consumer.group
   ├── Handler records Timer + ends Span
   └── Trace links producer → consumer spans
```

### Flow 3: Spring Boot Auto-Configuration Wiring

```
Spring Boot Application Starts
    │
    ▼
1. CompositeMeterRegistryAutoConfiguration
   → Creates CompositeMeterRegistry (or NOOP if no backends)
    │
    ▼
2. MetricsAutoConfiguration
   → MeterRegistryPostProcessor: applies customizers, filters, binders
   → Creates DefaultMeterObservationHandler
   → Groups as metricsObservationHandlerGroup
    │
    ▼
3. BraveAutoConfiguration (or OpenTelemetryTracingAutoConfiguration)
   → Creates Brave Tracer → BraveTracer adapter → Propagator
    │
    ▼
4. MicrometerTracingAutoConfiguration
   → Creates DefaultTracingObservationHandler
   → Creates PropagatingSender/ReceiverTracingObservationHandler
   → Groups as tracingObservationHandlerGroup
   → TracingAndMeterObservationHandlerGroup wraps metrics with tracing awareness
    │
    ▼
5. ObservationAutoConfiguration
   → Creates ObservationRegistry.create()
   → ObservationRegistryPostProcessor configures:
     a. ObservationPredicates (enable/disable by name)
     b. GlobalObservationConventions (naming overrides)
     c. ObservationHandlerGroups (metrics + tracing handlers)
     d. ObservationFilters (context mutation)
     e. ObservationRegistryCustomizers (user callbacks)
    │
    ▼
6. WebMvcObservationAutoConfiguration
   → Registers ServerHttpObservationFilter bean
   → Wires ObservationRegistry into filter
    │
    ▼
7. RestClientObservationAutoConfiguration
   → Registers ObservationRestClientCustomizer
   → Wires ObservationRegistry into RestClient.Builder
```

---

## Design Patterns

| # | Pattern | Where Used | Why Chosen |
|---|---------|------------|------------|
| 1 | **Visitor** | `ObservationHandler` visits `Context` lifecycle | One observation, multiple handlers — each handler interprets events independently (metrics creates timers, tracing creates spans) |
| 2 | **Strategy** | `ObservationConvention` defines naming/tagging | Different domains (HTTP, Kafka, JMS) need different tag schemas; strategy allows hot-swapping conventions |
| 3 | **Template Method** | `ObservationHandler` lifecycle hooks | Framework controls call order (start→scope→stop); handlers fill in domain-specific behavior |
| 4 | **Composite** | `AllMatchingCompositeObservationHandler` | Multiple handlers can process the same observation simultaneously |
| 5 | **Registry** | `ObservationRegistry`, `ContextRegistry` | Central discovery point; decouples handler producers from handler consumers |
| 6 | **Context Object** | `Observation.Context` subclasses | Carries all observation data through lifecycle; extensible via `put()`/`get()` for handler-specific data |
| 7 | **Null Object** | `ObservationRegistry.NOOP`, `Observation.NOOP` | When observation is disabled, zero-overhead no-op instead of null checks everywhere |
| 8 | **Self-Documenting Enum** | `ObservationDocumentation` implementations | Enum constants double as documentation AND factory — know what tags exist at compile time |

---

## Architectural Decisions

| Decision | Chosen | Alternatives Rejected | Rationale |
|----------|--------|----------------------|-----------|
| Single Observation API (not separate metrics + tracing) | Unified `Observation` | Separate `Timer.record()` + `Tracer.newSpan()` | One instrumentation point → less code, guaranteed correlation, no drift between metrics and traces |
| Convention over configuration for naming | `ObservationConvention` with defaults | Hardcoded names, annotation-driven | Allows library to ship sensible defaults while users can override globally or per-observation |
| Context as mutable bag | `Context` with `put()`/`get()` | Immutable context, typed fields only | Handlers need to store handler-specific data (Timer.Sample, Span) without context knowing about all handler types |
| Enum-based documentation | `ObservationDocumentation` enum | XML/YAML documentation, Javadoc only | Compile-time enumeration of observations; generates docs automatically; serves as factory method |
| Transport contexts (Sender/Receiver) | Separate `SenderContext`/`ReceiverContext` extending `Context` | Single context with direction field | Type safety for propagation handlers; `Propagator.Setter` vs `Getter` is fundamentally different |
| NOOP registry instead of nullable | `ObservationRegistry.NOOP` | `@Nullable ObservationRegistry` | Eliminates null checks; zero-cost when observation is disabled; no NullPointerException risk |
| Convention selection: custom → global → default | 3-level fallback chain | Single convention only | Libraries provide defaults; users can override per-observation (custom) or application-wide (global) |

---

## The 5-Artifact Pattern (Used by ALL Spring Projects)

Every Spring project that integrates with Micrometer creates exactly these 5 artifacts:

```
┌──────────────────────────────────────────────────────────────────────┐
│ ARTIFACT 1: ObservationContext subclass                              │
│   Purpose: Hold domain-specific data for tag extraction              │
│   Example: KafkaRecordSenderContext, ServerRequestObservationContext │
│   Extends: Context, SenderContext, ReceiverContext, etc.             │
├──────────────────────────────────────────────────────────────────────┤
│ ARTIFACT 2: ObservationConvention interface                          │
│   Purpose: Marker interface with supportsContext() + default name    │
│   Example: KafkaTemplateObservationConvention                        │
│   Extends: ObservationConvention<MyContext>                          │
├──────────────────────────────────────────────────────────────────────┤
│ ARTIFACT 3: DefaultObservationConvention implementation              │
│   Purpose: Ship sensible defaults for naming + tags                  │
│   Example: DefaultKafkaTemplateObservationConvention                 │
│   Implements: MyConvention interface from Artifact 2                 │
├──────────────────────────────────────────────────────────────────────┤
│ ARTIFACT 4: ObservationDocumentation enum                            │
│   Purpose: Self-documenting observation definitions + factory        │
│   Example: KafkaTemplateObservation.TEMPLATE_OBSERVATION             │
│   Implements: ObservationDocumentation                               │
│   Contains: KeyName enums (low + high cardinality)                   │
│   Contains: DefaultConvention as inner class (singleton INSTANCE)    │
├──────────────────────────────────────────────────────────────────────┤
│ ARTIFACT 5: Integration point (where observation is created)         │
│   Purpose: Wrap existing operations with observation lifecycle       │
│   Example: KafkaTemplate.observeSend(), ServerHttpObservationFilter  │
│   Pattern: create → start → openScope → doWork → stop               │
└──────────────────────────────────────────────────────────────────────┘
```

---

## Mapping Table: Simplified ↔ Real Source

| Concept | Spring Kafka Source | Spring Framework Source | Spring Boot Source |
|---------|--------------------|-----------------------|-------------------|
| ObservationContext (sender) | `KafkaRecordSenderContext.java` | `ClientRequestObservationContext.java` (spring-web) | — |
| ObservationContext (receiver) | `KafkaRecordReceiverContext.java` | `ServerRequestObservationContext.java` (spring-web) | — |
| Convention interface | `KafkaTemplateObservationConvention.java` | `ServerRequestObservationConvention.java` | — |
| Default convention | `KafkaTemplateObservation.DefaultKafkaTemplateObservationConvention` (inner class) | `DefaultServerRequestObservationConvention.java` | — |
| Documentation enum | `KafkaTemplateObservation.java` | `ServerHttpObservationDocumentation.java` | — |
| Integration point | `KafkaTemplate.observeSend():819` | `ServerHttpObservationFilter.doFilterInternal()` | — |
| Auto-configuration | — | — | `ObservationAutoConfiguration.java` |
| Handler registration | — | — | `ObservationRegistryPostProcessor.java` |
| Convention as bean | — | — | `WebMvcObservationAutoConfiguration.java` |

---

## What We Simplified Away

| Feature | Why Omitted |
|---------|-------------|
| Reactive/Async observation support | Zalobot SDK is synchronous; async adds complexity without immediate benefit |
| Context Propagation library integration | Only needed for reactive chains (Project Reactor); not applicable to blocking long-polling |
| `ObservationFilter` and `ObservationPredicate` | Advanced features; can be added later without changing the core integration |
| `ObservedAspect` (@Observed annotation) | AOP-based; Zalobot SDK should use programmatic observation like Spring Framework does |
| Multiple tracing backends (Brave vs OpenTelemetry) | Spring Boot's auto-configuration handles this transparently; SDK doesn't need to care |
| `GlobalObservationConvention` | Primarily used by Spring Boot auto-config; SDK users can override via the custom convention parameter |
