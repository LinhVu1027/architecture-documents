# Simple Tracing — API-First Learning Outline

> Learn Micrometer Tracing by building a simplified version from the **outside in** — start with
> the APIs that clients actually call, then implement the internal machinery behind each one.

## Quick Start

```bash
# After implementing Feature 1, a client can already do:
```
```java
Tracer tracer = new SimpleTracer();
ScopedSpan span = tracer.startScopedSpan("my-operation");
try {
    span.tag("user.id", "12345");
    // ... do work ...
} catch (Exception e) {
    span.error(e);
    throw e;
} finally {
    span.end();
}
```

## Architecture Overview

Micrometer Tracing is a **distributed tracing facade** — it provides a vendor-neutral API
for creating, managing, and propagating trace spans across service boundaries. Applications
use it to instrument their code with tracing, while the actual tracing backend (Brave,
OpenTelemetry, or a simple in-memory implementation for tests) is plugged in at runtime
through a bridge module.

**Real-world analogy**: Using Micrometer Tracing is like filling out a standardized
shipping label (the `Span` API) — you write the sender, receiver, timestamps, and notes
on a universal form. Whether the package goes via FedEx (Brave) or UPS (OpenTelemetry)
depends on which carrier (bridge module) you hand it to. Your labeling code never changes —
only the carrier does.

## Multi-Module Structure

This project mirrors the real Micrometer Tracing repository's multi-module layout:

```
simple-tracing/                              (root project)
├── simple-tracing-api/                      ← Core API interfaces (Tracer, Span, etc.)
│   └── io.simpletracing
├── simple-tracing-bridge-brave/             ← Brave bridge implementation
│   └── io.simpletracing.brave.bridge
├── simple-tracing-test/                     ← In-memory test implementations + assertions
│   └── io.simpletracing.test.simple
└── docs-v2/                                 ← Tutorial chapters
```

Maps to real modules:
| Simple Module | Real Module | Purpose |
|---|---|---|
| `simple-tracing-api` | `micrometer-tracing` | Core facade interfaces |
| `simple-tracing-bridge-brave` | `micrometer-tracing-bridge-brave` | Brave adapter |
| `simple-tracing-test` | `micrometer-tracing-test` | In-memory fakes + AssertJ assertions |

**Out of scope modules** (not built in simplified version):
- `micrometer-tracing-bridge-otel` — Only one bridge needed to learn the pattern
- `micrometer-tracing-reporter-wavefront` — Deprecated reporter
- `micrometer-tracing-integration-test` — Cross-bridge integration tests
- `micrometer-tracing-bom` — BOM is a publishing concern, not a learning one
- `benchmarks` — JMH benchmarks are a performance concern

## API Surface Map

```
┌──────────────────────────────────────────────────────────────────┐
│                         CLIENT CODE                              │
│  uses: Tracer, Span, ScopedSpan, Baggage, Propagator            │
│  annotations: @NewSpan, @ContinueSpan, @SpanTag                 │
│  testing: SpanAssert, SpansAssert, TracerAssert                  │
└──────┬──────────┬───────────┬──────────┬──────────┬──────────────┘
       │          │           │          │          │
  ┌────▼───┐ ┌───▼────┐ ┌────▼───┐ ┌───▼────┐ ┌───▼──────────┐
  │  F1    │ │  F2    │ │  F3    │ │  F4    │ │   F5         │
  │ Tracer │ │ Span   │ │Context │ │Baggage │ │ Propagator   │
  │ Scope  │ │Builder │ │Propag. │ │Manager │ │ Inject/Extr  │
  └────┬───┘ └───┬────┘ └────┬───┘ └───┬────┘ └───┬──────────┘
       │          │           │          │          │
  ┌────▼──────────▼───────────▼──────────▼──────────▼──────────┐
  │              SHARED INTERNALS                               │
  │  TraceContext, CurrentTraceContext, SpanCustomizer           │
  │  FinishedSpan, SpanReporter, SpanFilter                     │
  └────────────────────────────────────────────────────────────┘
       │                    │                    │
  ┌────▼──────┐      ┌─────▼──────┐      ┌─────▼──────┐
  │   F6      │      │    F7      │      │    F8      │
  │ Span      │      │ Obs.       │      │ @NewSpan   │
  │ Export    │      │ Handler    │      │ @Continue  │
  │ Pipeline  │      │ Bridge     │      │ Annotation │
  └───────────┘      └────────────┘      └────────────┘
       │                    │
  ┌────▼──────────────────────────────────────────────┐
  │              BRIDGE LAYER                          │
  │  F9: SimpleTracer (test fake)                     │
  │  F10: BraveTracer (Brave adapter)                 │
  └───────────────────────────────────────────────────┘
       │
  ┌────▼──────────────────────────────────────────────┐
  │              TEST ASSERTIONS                       │
  │  F11: SpanAssert, SpansAssert, TracerAssert        │
  └───────────────────────────────────────────────────┘
```

## Call Chain Diagrams

### tracer.startScopedSpan("name") Call Chain
```
Client calls: tracer.startScopedSpan("encode")
  │
  ├─► [API Layer] Tracer.startScopedSpan(String name)
  │     Creates a new span as child of current span, puts it in scope
  │
  ├─► [Context Layer] CurrentTraceContext.newScope(TraceContext)
  │     Saves previous context, sets new context as "current" (ThreadLocal)
  │
  ├─► [Span Layer] ScopedSpan — wraps span + scope together
  │     Client calls tag(), event(), error() on it
  │
  └─► [End] ScopedSpan.end()
        Closes scope (restores previous context), ends span, reports it
```

### tracer.nextSpan().name("x").start() Call Chain
```
Client calls: tracer.nextSpan().name("encode").start()
  │
  ├─► [API Layer] Tracer.nextSpan()
  │     Checks CurrentTraceContext for parent, creates child or root span
  │
  ├─► [Builder Layer] Span.Builder.name().kind().tag().start()
  │     Configures span before starting (parent, kind, tags, name)
  │
  ├─► [Scope Layer] tracer.withSpan(span) → SpanInScope
  │     Puts span in scope for downstream code (loggers, etc.)
  │
  └─► [End] span.end() + spanInScope.close()
        Ends span independently of scope, reports via SpanReporter
```

### propagator.inject() / extract() Call Chain
```
SENDER SIDE:
Client calls: propagator.inject(traceContext, httpHeaders, setter)
  │
  ├─► [API Layer] Propagator.inject(TraceContext, C, Setter<C>)
  │     Writes traceId, spanId, sampled flag into carrier using Setter
  │
  └─► [Carrier Layer] Setter.set(carrier, "X-B3-TraceId", traceId)
        Sets header values on the carrier (e.g., HTTP headers)

RECEIVER SIDE:
Client calls: propagator.extract(httpHeaders, getter)
  │
  ├─► [API Layer] Propagator.extract(C, Getter<C>)
  │     Reads traceId, spanId, sampled flag from carrier using Getter
  │
  └─► [Builder Layer] Returns Span.Builder with parent context set
        Caller then calls .name("handle").kind(SERVER).start()
```

## Features

---

### Feature 1: Core Span Lifecycle — Create, Scope, and End Spans {Tier: 1}
✅ Complete

**API Contract** (what clients call):
```java
// Simple usage — ScopedSpan (span + scope combined)
Tracer tracer = ...;
ScopedSpan span = tracer.startScopedSpan("encode");
try {
    span.tag("input.length", "256");
    span.event("encoding-started");
    return encoder.encode();
} catch (RuntimeException | Error e) {
    span.error(e);
    throw e;
} finally {
    span.end();
}

// Advanced usage — Span + explicit scope
Span span = tracer.nextSpan().name("encode").start();
try (Tracer.SpanInScope ws = tracer.withSpan(span)) {
    return encoder.encode();
} catch (RuntimeException | Error e) {
    span.error(e);
    throw e;
} finally {
    span.end();
}
```

**What it does**: Lets clients create spans to measure units of work, with automatic parent-child
relationships via thread-local scoping.

**Depth layers** (what gets built to support this API):

| Layer | Simplified Class | Real Framework Class | Purpose |
|-------|------------------|----------------------|---------|
| API | `Tracer` | `io.micrometer.tracing.Tracer` | Central entry point — create spans, manage scope |
| API | `Span` | `io.micrometer.tracing.Span` | Represents a unit of work with tags, events, errors |
| API | `Span.Builder` | `io.micrometer.tracing.Span.Builder` | Configure span before starting (parent, kind, name) |
| API | `Span.Kind` | `io.micrometer.tracing.Span.Kind` | Enum: SERVER, CLIENT, PRODUCER, CONSUMER |
| API | `ScopedSpan` | `io.micrometer.tracing.ScopedSpan` | Span that is automatically scoped |
| API | `SpanCustomizer` | `io.micrometer.tracing.SpanCustomizer` | Lightweight customizer (name, tag, event only) |
| API | `Tracer.SpanInScope` | `io.micrometer.tracing.Tracer.SpanInScope` | Closeable scope for a span |
| Context | `TraceContext` | `io.micrometer.tracing.TraceContext` | Holds traceId, spanId, parentId, sampled |
| Context | `TraceContext.Builder` | `io.micrometer.tracing.TraceContext.Builder` | Builds trace contexts |
| Context | `CurrentTraceContext` | `io.micrometer.tracing.CurrentTraceContext` | Thread-local scope management |
| Context | `CurrentTraceContext.Scope` | `io.micrometer.tracing.CurrentTraceContext.Scope` | Closeable scope that restores previous context |

**Module**: `simple-tracing-api`

**Real source files**: `Tracer.java`, `Span.java`, `ScopedSpan.java`, `SpanCustomizer.java`, `TraceContext.java`, `CurrentTraceContext.java`

**Depends on**: None (first feature)
**Complexity**: Medium · 11 new interfaces/classes
**Java version**: 17

**Concrete deliverables**:
- [ ] `api/Tracer.java` — Central tracing interface with NOOP constant
- [ ] `api/Span.java` — Span interface with Kind enum and Builder
- [ ] `api/ScopedSpan.java` — Auto-scoped span interface
- [ ] `api/SpanCustomizer.java` — Lightweight span customization
- [ ] `api/TraceContext.java` — Immutable trace identity (traceId, spanId, parentId, sampled)
- [ ] `api/TraceContext.Builder` — Builder for TraceContext
- [ ] `api/CurrentTraceContext.java` — Thread-local context scope management
- [ ] `test/TracerApiTest.java` — Client-perspective test verifying span creation and scoping
- [ ] `docs-v2/ch01_core_span_lifecycle.md` — Tutorial chapter

---

### Feature 2: Span Export Pipeline — Report, Filter, and Gate Finished Spans {Tier: 1}
✅ Complete

**API Contract** (what clients call):
```java
// Client configures the export pipeline
SpanReporter reporter = span -> System.out.println("Exported: " + span.getName());
SpanFilter filter = span -> span.setName(span.getName().toUpperCase());
SpanExportingPredicate predicate = span -> !span.getName().startsWith("internal-");

// FinishedSpan is what gets reported after a span ends
// (clients don't create these directly — the tracer does)
```

**What it does**: Provides the pipeline through which completed spans flow — they can be
transformed (SpanFilter), gate-checked (SpanExportingPredicate), and reported to external
systems (SpanReporter).

**Depth layers**:

| Layer | Simplified Class | Real Framework Class | Purpose |
|-------|------------------|----------------------|---------|
| API | `FinishedSpan` | `io.micrometer.tracing.exporter.FinishedSpan` | Completed span data model |
| API | `SpanReporter` | `io.micrometer.tracing.exporter.SpanReporter` | Reports spans to external systems |
| API | `SpanFilter` | `io.micrometer.tracing.exporter.SpanFilter` | Transforms spans before export |
| API | `SpanExportingPredicate` | `io.micrometer.tracing.exporter.SpanExportingPredicate` | Gates span export |

**Module**: `simple-tracing-api`

**Real source files**: `FinishedSpan.java`, `SpanReporter.java`, `SpanFilter.java`, `SpanExportingPredicate.java`, `SpanIgnoringSpanExportingPredicate.java`

**Depends on**: Feature 1 (uses `Span.Kind`, `TraceContext`, `Link`)
**Complexity**: Low · 4 new interfaces
**Java version**: 17

**Concrete deliverables**:
- [ ] `api/exporter/FinishedSpan.java` — Completed span data interface
- [ ] `api/exporter/SpanReporter.java` — Span reporting interface
- [ ] `api/exporter/SpanFilter.java` — Span transformation interface
- [ ] `api/exporter/SpanExportingPredicate.java` — Span export gating interface
- [ ] `test/ExportPipelineTest.java` — Tests verifying filter → predicate → reporter flow
- [ ] `docs-v2/ch02_span_export_pipeline.md` — Tutorial chapter

---

### Feature 3: Baggage — Propagate Key-Value Pairs Across Service Boundaries {Tier: 2}
✅ Complete

**API Contract** (what clients call):
```java
// Create baggage and put it in scope
Tracer tracer = ...;
try (BaggageInScope baggage = tracer.createBaggageInScope("user.id", "12345")) {
    // Baggage is now in scope — will be propagated to downstream services
    String userId = tracer.getBaggage("user.id").get();  // "12345"
}

// Read all baggage
Map<String, String> allBaggage = tracer.getAllBaggage();
```

**What it does**: Lets clients attach key-value metadata to the trace context that propagates
across service boundaries (unlike tags, which are span-local). Useful for tenant IDs,
correlation IDs, feature flags.

**Depth layers**:

| Layer | Simplified Class | Real Framework Class | Purpose |
|-------|------------------|----------------------|---------|
| API | `BaggageView` | `io.micrometer.tracing.BaggageView` | Read-only baggage view (name, get) |
| API | `Baggage` | `io.micrometer.tracing.Baggage` | Mutable baggage (set value, makeCurrent) |
| API | `BaggageInScope` | `io.micrometer.tracing.BaggageInScope` | Closeable scoped baggage |
| API | `BaggageManager` | `io.micrometer.tracing.BaggageManager` | Lifecycle management (create, get, getAll) |

**Module**: `simple-tracing-api`

**Real source files**: `BaggageView.java`, `Baggage.java`, `BaggageInScope.java`, `BaggageManager.java`

**Depends on**: Feature 1 (Tracer extends BaggageManager; baggage scoped to TraceContext)
**Complexity**: Medium · 4 new interfaces
**Java version**: 17

**Concrete deliverables**:
- [ ] `api/BaggageView.java` — Read-only baggage view
- [ ] `api/Baggage.java` — Mutable baggage interface
- [ ] `api/BaggageInScope.java` — Closeable scoped baggage
- [ ] `api/BaggageManager.java` — Baggage lifecycle manager (Tracer extends this)
- [ ] `test/BaggageApiTest.java` — Tests for baggage creation, scoping, and retrieval
- [ ] `docs-v2/ch03_baggage.md` — Tutorial chapter

---

### Feature 4: Context Propagation — Inject and Extract Trace Context {Tier: 1}
✅ Complete

**API Contract** (what clients call):
```java
Propagator propagator = ...;

// SENDER: inject trace context into outgoing HTTP headers
Map<String, String> headers = new HashMap<>();
propagator.inject(span.context(), headers, Map::put);

// RECEIVER: extract trace context from incoming HTTP headers
Span.Builder builder = propagator.extract(headers, Map::get);
Span serverSpan = builder.kind(Span.Kind.SERVER).name("handle-request").start();
```

**What it does**: Enables distributed tracing by injecting trace identity into outgoing
requests and extracting it from incoming requests, using pluggable carrier formats (e.g.,
B3 headers, W3C Trace Context).

**Depth layers**:

| Layer | Simplified Class | Real Framework Class | Purpose |
|-------|------------------|----------------------|---------|
| API | `Propagator` | `io.micrometer.tracing.propagation.Propagator` | Inject/extract trace context |
| API | `Propagator.Setter<C>` | `io.micrometer.tracing.propagation.Propagator.Setter` | Writes fields onto carrier |
| API | `Propagator.Getter<C>` | `io.micrometer.tracing.propagation.Propagator.Getter` | Reads fields from carrier |

**Module**: `simple-tracing-api`

**Real source files**: `Propagator.java`

**Depends on**: Feature 1 (uses TraceContext, Span.Builder)
**Complexity**: Low · 1 new interface with 2 inner interfaces
**Java version**: 17

**Concrete deliverables**:
- [ ] `api/propagation/Propagator.java` — Propagation interface with Setter and Getter
- [ ] `test/PropagatorApiTest.java` — Tests verifying inject/extract round-trip
- [ ] `docs-v2/ch04_context_propagation.md` — Tutorial chapter

---

### Feature 5: Link — Associate Related Spans Across Traces {Tier: 3}
✅ Complete

**API Contract** (what clients call):
```java
// Link a new span to an existing span in another trace
Span batchSpan = tracer.nextSpan().name("process-batch").start();
for (TraceContext messageContext : incomingMessages) {
    Span processingSpan = tracer.spanBuilder()
        .name("process-message")
        .addLink(new Link(messageContext))
        .start();
    // ... process message ...
    processingSpan.end();
}
```

**What it does**: Allows associating spans across different traces — commonly used in batch
processing where one span processes messages from multiple independent traces.

**Depth layers**:

| Layer | Simplified Class | Real Framework Class | Purpose |
|-------|------------------|----------------------|---------|
| API | `Link` | `io.micrometer.tracing.Link` | Holds a TraceContext + tags for the linked span |

**Module**: `simple-tracing-api`

**Real source files**: `Link.java`

**Depends on**: Feature 1 (uses TraceContext, Span.Builder.addLink)
**Complexity**: Low · 1 new class
**Java version**: 17

**Concrete deliverables**:
- [ ] `api/Link.java` — Span link with TraceContext and tags
- [ ] `test/LinkTest.java` — Tests for link creation and equality
- [ ] `docs-v2/ch05_link.md` — Tutorial chapter

---

### Feature 6: Simple Test Implementation — In-Memory Tracer for Testing {Tier: 1}
✅ Complete

**API Contract** (what clients call):
```java
// In a test class
SimpleTracer tracer = new SimpleTracer();

// ... exercise production code that uses Tracer ...

// Verify spans were created correctly
SimpleSpan span = tracer.onlySpan();  // asserts exactly one span
assertThat(span.getName()).isEqualTo("encode");
assertThat(span.getTags()).containsEntry("input.length", "256");
assertThat(span.getEvents()).hasSize(1);

// Or use the last span
SimpleSpan last = tracer.lastSpan();
```

**What it does**: Provides a fully functional in-memory implementation of all tracing
interfaces, enabling tests to verify tracing behavior without any real tracing backend.
This is the first **concrete implementation** of the API — making the interfaces usable.

**Depth layers**:

| Layer | Simplified Class | Real Framework Class | Purpose |
|-------|------------------|----------------------|---------|
| Impl | `SimpleTracer` | `io.micrometer.tracing.test.simple.SimpleTracer` | In-memory Tracer implementation |
| Impl | `SimpleSpan` | `io.micrometer.tracing.test.simple.SimpleSpan` | In-memory Span (also implements FinishedSpan) |
| Impl | `SimpleSpanBuilder` | `io.micrometer.tracing.test.simple.SimpleSpanBuilder` | In-memory Span.Builder |
| Impl | `SimpleScopedSpan` | `io.micrometer.tracing.test.simple.SimpleScopedSpan` | In-memory ScopedSpan |
| Impl | `SimpleTraceContext` | `io.micrometer.tracing.test.simple.SimpleTraceContext` | In-memory TraceContext with UUID generation |
| Impl | `SimpleTraceContextBuilder` | `io.micrometer.tracing.test.simple.SimpleTraceContextBuilder` | In-memory TraceContext.Builder |
| Impl | `SimpleCurrentTraceContext` | `io.micrometer.tracing.test.simple.SimpleCurrentTraceContext` | ThreadLocal-based CurrentTraceContext |
| Impl | `SimpleSpanCustomizer` | `io.micrometer.tracing.test.simple.SimpleSpanCustomizer` | In-memory SpanCustomizer |
| Impl | `SimpleBaggageInScope` | `io.micrometer.tracing.test.simple.SimpleBaggageInScope` | In-memory BaggageInScope |
| Impl | `SimpleBaggageManager` | `io.micrometer.tracing.test.simple.SimpleBaggageManager` | In-memory BaggageManager |

**Module**: `simple-tracing-test`

**Real source files**: `SimpleTracer.java`, `SimpleSpan.java`, `SimpleSpanBuilder.java`, `SimpleScopedSpan.java`, `SimpleTraceContext.java`, `SimpleTraceContextBuilder.java`, `SimpleCurrentTraceContext.java`, `SimpleSpanCustomizer.java`, `SimpleBaggageInScope.java`, `SimpleBaggageManager.java`

**Depends on**: Features 1-5 (implements all API interfaces)
**Complexity**: High · 10 new classes
**Java version**: 17

**Concrete deliverables**:
- [ ] `test-module/SimpleTracer.java` — In-memory tracer with span collection
- [ ] `test-module/SimpleSpan.java` — Records tags, events, errors in memory
- [ ] `test-module/SimpleSpanBuilder.java` — Builds SimpleSpan instances
- [ ] `test-module/SimpleScopedSpan.java` — ScopedSpan backed by SimpleSpan
- [ ] `test-module/SimpleTraceContext.java` — TraceContext with UUID-based IDs
- [ ] `test-module/SimpleTraceContextBuilder.java` — Builder for SimpleTraceContext
- [ ] `test-module/SimpleCurrentTraceContext.java` — ThreadLocal scope management
- [ ] `test-module/SimpleSpanCustomizer.java` — In-memory customizer
- [ ] `test-module/SimpleBaggageInScope.java` — In-memory scoped baggage
- [ ] `test-module/SimpleBaggageManager.java` — In-memory baggage manager
- [ ] `test-module/SimpleTracerTest.java` — Comprehensive tests for all Simple* classes
- [ ] `docs-v2/ch06_simple_test_implementation.md` — Tutorial chapter

---

### Feature 7: Test Assertions — AssertJ-Style Span Verification {Tier: 2}
✅ Complete

**API Contract** (what clients call):
```java
SimpleTracer tracer = new SimpleTracer();
// ... exercise code ...

// Fluent assertions on a single span
SpanAssert.assertThat(tracer.onlySpan())
    .hasNameEqualTo("encode")
    .hasTagWithKey("input.length")
    .hasEventWithNameEqualTo("encoding-started")
    .hasNoErrors()
    .isStarted()
    .isEnded();

// Fluent assertions on all spans
SpansAssert.assertThat(tracer.getSpans())
    .hasSize(3)
    .hasASpanWithName("encode")
    .hasASpanWithNameEqualTo("decode");

// Fluent assertions on the tracer itself
TracerAssert.assertThat(tracer)
    .onlySpan()
    .hasNameEqualTo("encode");
```

**What it does**: Provides AssertJ-style fluent assertions for verifying tracing behavior in
tests — making test code expressive and error messages informative.

**Depth layers**:

| Layer | Simplified Class | Real Framework Class | Purpose |
|-------|------------------|----------------------|---------|
| Assert | `SpanAssert` | `io.micrometer.tracing.test.simple.SpanAssert` | Assert on a single FinishedSpan |
| Assert | `SpansAssert` | `io.micrometer.tracing.test.simple.SpansAssert` | Assert on a collection of spans |
| Assert | `TracerAssert` | `io.micrometer.tracing.test.simple.TracerAssert` | Assert on the SimpleTracer |
| Assert | `TracingAssertions` | `io.micrometer.tracing.test.simple.TracingAssertions` | Static entry point for assertions |

**Module**: `simple-tracing-test`

**Real source files**: `SpanAssert.java`, `SpansAssert.java`, `TracerAssert.java`, `TracingAssertions.java`

**Depends on**: Feature 6 (asserts on SimpleTracer/SimpleSpan/FinishedSpan)
**Complexity**: Medium · 4 new classes
**Java version**: 17

**Concrete deliverables**:
- [ ] `test-module/SpanAssert.java` — Single span assertions
- [ ] `test-module/SpansAssert.java` — Collection assertions
- [ ] `test-module/TracerAssert.java` — Tracer-level assertions
- [ ] `test-module/TracingAssertions.java` — Static assertion entry point
- [ ] `test-module/AssertionTest.java` — Tests verifying assertion behavior
- [ ] `docs-v2/ch07_test_assertions.md` — Tutorial chapter

---

### Feature 8: Brave Bridge — Adapt Brave's API to the Tracing Facade {Tier: 1}
✅ Complete

**API Contract** (what clients call):
```java
// Client configures the Brave bridge
brave.Tracing braveTracing = brave.Tracing.newBuilder()
    .localServiceName("my-service")
    .build();

// Wrap Brave in the Micrometer facade
Tracer tracer = new BraveTracer(
    braveTracing.tracer(),
    new BraveCurrentTraceContext(braveTracing.currentTraceContext()),
    new BraveBaggageManager()
);

// Now use the standard Tracer API — all calls delegate to Brave
Span span = tracer.nextSpan().name("encode").start();
```

**What it does**: Adapts Brave's tracing API (brave.Tracer, brave.Span, etc.) to the
Micrometer Tracing facade interfaces, so applications can use Brave as their tracing
backend while coding against the vendor-neutral API.

**Depth layers**:

| Layer | Simplified Class | Real Framework Class | Purpose |
|-------|------------------|----------------------|---------|
| Bridge | `BraveTracer` | `io.micrometer.tracing.brave.bridge.BraveTracer` | Adapts brave.Tracer → Tracer |
| Bridge | `BraveSpan` | `io.micrometer.tracing.brave.bridge.BraveSpan` | Adapts brave.Span → Span |
| Bridge | `BraveSpanBuilder` | `io.micrometer.tracing.brave.bridge.BraveSpanBuilder` | Adapts → Span.Builder |
| Bridge | `BraveScopedSpan` | `io.micrometer.tracing.brave.bridge.BraveScopedSpan` | Adapts → ScopedSpan |
| Bridge | `BraveTraceContext` | `io.micrometer.tracing.brave.bridge.BraveTraceContext` | Adapts → TraceContext |
| Bridge | `BraveTraceContextBuilder` | — | Adapts → TraceContext.Builder |
| Bridge | `BraveCurrentTraceContext` | `io.micrometer.tracing.brave.bridge.BraveCurrentTraceContext` | Adapts → CurrentTraceContext |
| Bridge | `BraveSpanCustomizer` | `io.micrometer.tracing.brave.bridge.BraveSpanCustomizer` | Adapts → SpanCustomizer |
| Bridge | `BravePropagator` | `io.micrometer.tracing.brave.bridge.BravePropagator` | Adapts → Propagator |
| Bridge | `BraveBaggageManager` | `io.micrometer.tracing.brave.bridge.BraveBaggageManager` | Adapts → BaggageManager |
| Bridge | `BraveBaggageInScope` | `io.micrometer.tracing.brave.bridge.BraveBaggageInScope` | Adapts → BaggageInScope |
| Bridge | `BraveFinishedSpan` | `io.micrometer.tracing.brave.bridge.BraveFinishedSpan` | Adapts → FinishedSpan |
| Bridge | `CompositeSpanHandler` | `io.micrometer.tracing.brave.bridge.CompositeSpanHandler` | Bridges export pipeline to Brave's SpanHandler |

**Module**: `simple-tracing-bridge-brave`

**Real source files**: `BraveTracer.java`, `BraveSpan.java`, `BraveSpanBuilder.java`, `BraveScopedSpan.java`, `BraveTraceContext.java`, `BraveCurrentTraceContext.java`, `BraveSpanCustomizer.java`, `BravePropagator.java`, `BraveBaggageManager.java`, `BraveBaggageInScope.java`, `BraveFinishedSpan.java`, `CompositeSpanHandler.java`

**Depends on**: Features 1-5 (implements all API interfaces), Feature 2 (export pipeline integration)
**Complexity**: High · 13 new classes
**Java version**: 17

**Concrete deliverables**:
- [ ] `brave-bridge/BraveTracer.java` — Brave → Tracer adapter
- [ ] `brave-bridge/BraveSpan.java` — brave.Span → Span adapter
- [ ] `brave-bridge/BraveSpanBuilder.java` — Span.Builder adapter
- [ ] `brave-bridge/BraveScopedSpan.java` — ScopedSpan adapter
- [ ] `brave-bridge/BraveTraceContext.java` — TraceContext adapter
- [ ] `brave-bridge/BraveCurrentTraceContext.java` — CurrentTraceContext adapter
- [ ] `brave-bridge/BraveSpanCustomizer.java` — SpanCustomizer adapter
- [ ] `brave-bridge/BravePropagator.java` — Propagator adapter
- [ ] `brave-bridge/BraveBaggageManager.java` — BaggageManager adapter
- [ ] `brave-bridge/BraveBaggageInScope.java` — BaggageInScope adapter
- [ ] `brave-bridge/BraveFinishedSpan.java` — FinishedSpan adapter
- [ ] `brave-bridge/CompositeSpanHandler.java` — Export pipeline bridge
- [ ] `brave-bridge/BraveTracerTest.java` — Integration tests with real Brave
- [ ] `docs-v2/ch08_brave_bridge.md` — Tutorial chapter

---

### Feature 9: Observation Handler Bridge — Connect Tracing to Micrometer Observation {Tier: 2}
✅ Complete

**API Contract** (what clients call):
```java
// Wire tracing into the Observation API
ObservationRegistry registry = ObservationRegistry.create();
registry.observationConfig()
    .observationHandler(new DefaultTracingObservationHandler(tracer));

// Now every Observation automatically creates a span
Observation observation = Observation.start("my-operation", registry);
try (Observation.Scope scope = observation.openScope()) {
    // span is automatically in scope here
    observation.lowCardinalityKeyValue("result", "success");
} finally {
    observation.stop();  // span is ended here
}
```

**What it does**: Bridges Micrometer Observation (the instrumentation API) to Micrometer
Tracing — so that observations automatically create, tag, and end spans. This is how
Spring Boot auto-configures tracing for HTTP, messaging, and database observations.

**Depth layers**:

| Layer | Simplified Class | Real Framework Class | Purpose |
|-------|------------------|----------------------|---------|
| API | `TracingObservationHandler<T>` | `io.micrometer.tracing.handler.TracingObservationHandler` | Base handler with TracingContext |
| Impl | `DefaultTracingObservationHandler` | `io.micrometer.tracing.handler.DefaultTracingObservationHandler` | Local (non-propagating) handler |
| Impl | `PropagatingSenderTracingObservationHandler` | `io.micrometer.tracing.handler.PropagatingSenderTracingObservationHandler` | Outgoing request handler |
| Impl | `PropagatingReceiverTracingObservationHandler` | `io.micrometer.tracing.handler.PropagatingReceiverTracingObservationHandler` | Incoming request handler |

**Module**: `simple-tracing-api`

**Real source files**: `TracingObservationHandler.java`, `DefaultTracingObservationHandler.java`, `PropagatingSenderTracingObservationHandler.java`, `PropagatingReceiverTracingObservationHandler.java`

**Depends on**: Feature 1 (Tracer, Span), Feature 4 (Propagator for sender/receiver handlers)
**External dependency**: `micrometer-observation` (for ObservationHandler, Observation.Context, SenderContext, ReceiverContext)
**Complexity**: High · 4 new classes (1 interface + 3 implementations)
**Java version**: 17

**Concrete deliverables**:
- [ ] `api/handler/TracingObservationHandler.java` — Base handler with TracingContext inner class
- [ ] `api/handler/DefaultTracingObservationHandler.java` — Local span handler
- [ ] `api/handler/PropagatingSenderTracingObservationHandler.java` — Sender (inject) handler
- [ ] `api/handler/PropagatingReceiverTracingObservationHandler.java` — Receiver (extract) handler
- [ ] `test/ObservationHandlerTest.java` — Tests with SimpleTracer + ObservationRegistry
- [ ] `docs-v2/ch09_observation_handler_bridge.md` — Tutorial chapter

---

### Feature 10: Declarative Tracing — @NewSpan, @ContinueSpan, @SpanTag Annotations {Tier: 3}
✅ Complete

**API Contract** (what clients call):
```java
// Annotate methods for declarative tracing
public class UserService {

    @NewSpan("get-user")
    public User getUser(@SpanTag("user.id") String userId) {
        return userRepository.findById(userId);
    }

    @ContinueSpan(log = "validate-user")
    public void validateUser(@SpanTag("user.name") String name) {
        // enriches the current span instead of creating a new one
    }
}
```

**What it does**: Lets clients annotate methods with `@NewSpan` (creates a new child span)
and `@ContinueSpan` (enriches the current span), with `@SpanTag` on parameters to
automatically tag spans. An AspectJ aspect processes these annotations at runtime.

**Depth layers**:

| Layer | Simplified Class | Real Framework Class | Purpose |
|-------|------------------|----------------------|---------|
| API | `@NewSpan` | `io.micrometer.tracing.annotation.NewSpan` | Annotation to create new span |
| API | `@ContinueSpan` | `io.micrometer.tracing.annotation.ContinueSpan` | Annotation to continue current span |
| API | `@SpanTag` | `io.micrometer.tracing.annotation.SpanTag` | Annotation to tag span from parameters |
| Aspect | `SpanAspect` | `io.micrometer.tracing.annotation.SpanAspect` | AspectJ aspect processing annotations |
| Processing | `MethodInvocationProcessor` | `io.micrometer.tracing.annotation.MethodInvocationProcessor` | Processes annotated method invocations |
| Processing | `NewSpanParser` | `io.micrometer.tracing.annotation.NewSpanParser` | Parses @NewSpan annotation |
| Processing | `SpanTagAnnotationHandler` | `io.micrometer.tracing.annotation.SpanTagAnnotationHandler` | Processes @SpanTag annotations |

**Module**: `simple-tracing-api`

**Real source files**: `NewSpan.java`, `ContinueSpan.java`, `SpanTag.java`, `SpanAspect.java`, `MethodInvocationProcessor.java`, `ImperativeMethodInvocationProcessor.java`, `NewSpanParser.java`, `DefaultNewSpanParser.java`, `SpanTagAnnotationHandler.java`

**Depends on**: Feature 1 (Tracer, Span)
**External dependency**: `aspectjweaver`, `aopalliance`
**Complexity**: Medium · 7 new classes (3 annotations + 4 processing classes)
**Java version**: 17

**Concrete deliverables**:
- [ ] `api/annotation/NewSpan.java` — Create-new-span annotation
- [ ] `api/annotation/ContinueSpan.java` — Continue-current-span annotation
- [ ] `api/annotation/SpanTag.java` — Tag-from-parameter annotation
- [ ] `api/annotation/SpanAspect.java` — AspectJ aspect
- [ ] `api/annotation/MethodInvocationProcessor.java` — Processing interface
- [ ] `api/annotation/NewSpanParser.java` — @NewSpan parsing
- [ ] `api/annotation/SpanTagAnnotationHandler.java` — @SpanTag processing
- [ ] `test/AnnotationTracingTest.java` — Tests with Spring AOP or manual aspect invocation
- [ ] `docs-v2/ch10_declarative_tracing.md` — Tutorial chapter

---

## Implementation Notes

### Simplification Strategy

| Real Framework Concept | Simplified Version | Why Simplified |
|------------------------|--------------------|----------------|
| Java 8 compatibility + Java 11 tests | Java 17 baseline | Modern Java reduces boilerplate, lets us use records/sealed types |
| `@Nullable` via JSpecify | `@Nullable` via JSpecify (kept) | Important for API contract documentation |
| Reactor/reactive context propagation | Deferred | Reactive is an advanced use case |
| `contextpropagation` module integration | Deferred | Adds complexity without core learning value |
| OpenTelemetry bridge | Deferred | One bridge (Brave) is sufficient to learn the adapter pattern |
| Wavefront reporter | Deferred | Deprecated and specific to one vendor |
| `SpanDocumentation` / docs module | Deferred | Documentation DSL is not core tracing |
| `TracingAwareMeterObservationHandler` | Deferred | Metrics/exemplar integration is niche |
| `ThreadLocalSpan` | Deferred | Convenience class, not core |
| `SpanAndScope` holder | Deferred | Simple holder, can add later if needed |
| `Link` timestamp-based events | Simplified | Core link concept without full event timestamp support |
| Brave `SpanHandler` complexities | Simplified `CompositeSpanHandler` | Focus on the adapter pattern, not Brave internals |
| Multiple propagation formats (B3, W3C, AWS) | B3 only | One format is enough to learn propagation |
| Nebula/Spring publishing plugins | Standard Gradle java-library | No need for publishing infrastructure |

### In Scope
- Core tracing API (Tracer, Span, ScopedSpan, TraceContext, CurrentTraceContext)
- Span export pipeline (FinishedSpan, SpanReporter, SpanFilter, SpanExportingPredicate)
- Baggage API (BaggageView, Baggage, BaggageInScope, BaggageManager)
- Context propagation (Propagator with inject/extract)
- Span links (Link)
- Simple/test implementation (all Simple* classes)
- Test assertions (SpanAssert, SpansAssert, TracerAssert)
- Brave bridge (all Brave* adapter classes)
- Observation handler bridge (TracingObservationHandler + 3 implementations)
- Declarative tracing annotations (@NewSpan, @ContinueSpan, @SpanTag + aspect)

### Out of Scope
- OpenTelemetry bridge — one bridge is sufficient
- Wavefront reporter — deprecated
- Reactive/Reactor context propagation
- Context-propagation library integration (ThreadLocalAccessor)
- JMH benchmarks
- BOM publishing
- Integration test module (cross-bridge testing)
- Spring Boot auto-configuration
- SpanDocumentation / documentation conventions

### Java Version
- Base: Java 17
- No features require newer APIs

## Dependency Graph

```
Feature 1: Core Span Lifecycle (Tracer, Span, TraceContext, CurrentTraceContext)
    │
    ├── Feature 2: Span Export Pipeline (FinishedSpan, SpanReporter, SpanFilter)
    │
    ├── Feature 3: Baggage (BaggageView, Baggage, BaggageInScope, BaggageManager)
    │
    ├── Feature 4: Context Propagation (Propagator, Setter, Getter)
    │
    ├── Feature 5: Link (span-to-span associations)
    │
    ├── Feature 9: Observation Handler Bridge (requires micrometer-observation)
    │       │
    │       └── needs Feature 4 (Propagator for sender/receiver)
    │
    └── Feature 10: Declarative Tracing (@NewSpan, @ContinueSpan, @SpanTag)
            │
            └── needs aspectjweaver
    │
    Features 1-5 ──► Feature 6: Simple Test Implementation (implements all APIs)
                         │
                         └── Feature 7: Test Assertions (SpanAssert, etc.)
    │
    Features 1-5 ──► Feature 8: Brave Bridge (adapts Brave to facade)

```

## Status Markers
- ⬜ = not started
- ✅ = implemented, tests passing, tutorial written
