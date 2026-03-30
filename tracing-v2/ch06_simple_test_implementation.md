# Chapter 6: Simple Test Implementation — In-Memory Tracer for Testing

> **What you'll build**: A fully functional in-memory implementation of all tracing interfaces —
> `SimpleTracer`, `SimpleSpan`, `SimpleScopedSpan`, `SimpleSpanBuilder`, `SimpleTraceContext`,
> `SimpleCurrentTraceContext`, `SimpleSpanCustomizer`, `SimpleBaggageInScope`, and
> `SimpleBaggageManager` — enabling tests to verify tracing behavior without any real tracing
> backend.

---

## 1. The API Contract

After this chapter, a client can write:

```java
// In a test class
SimpleTracer tracer = new SimpleTracer();

// ... exercise production code that uses Tracer ...
Span span = tracer.nextSpan().name("encode").start();
try (Tracer.SpanInScope ws = tracer.withSpan(span)) {
    span.tag("input.length", "256");
    span.event("encoding-started");
} finally {
    span.end();
}

// Verify spans were created correctly
SimpleSpan finished = tracer.onlySpan();  // asserts exactly one span
assertThat(finished.getName()).isEqualTo("encode");
assertThat(finished.getTags()).containsEntry("input.length", "256");
assertThat(finished.getEvents()).hasSize(1);

// Or use the last span when multiple spans exist
SimpleSpan last = tracer.lastSpan();
```

**Why this matters**: Chapters 1–5 defined abstract interfaces — elegant API contracts with
NOOP defaults. But interfaces alone can't run tests or prove behavior. This chapter provides
the **first concrete implementation**, making every interface from Features 1–5 usable with
real in-memory state.

### The Implementation Hierarchy

```
SimpleTracer (implements Tracer)
├── creates: SimpleSpan (implements Span + FinishedSpan)
├── creates: SimpleScopedSpan (implements ScopedSpan)
├── creates: SimpleSpanBuilder (implements Span.Builder)
├── creates: SimpleSpanCustomizer (implements SpanCustomizer)
├── manages: SimpleCurrentTraceContext (implements CurrentTraceContext)
├── delegates: SimpleBaggageManager (implements BaggageManager)
│   └── creates: SimpleBaggageInScope (implements Baggage + BaggageInScope)
└── uses: SimpleTraceContext (implements TraceContext)
    └── built by: SimpleTraceContextBuilder (implements TraceContext.Builder)
```

**10 classes implementing 10+ interfaces** — this is the largest single feature, and for good
reason. It's the first time the API becomes concrete. Every interface method that was previously
a `return this` or `return null` now has real behavior backed by real state.

### The Dual-Interface Pattern

The most interesting design decision: `SimpleSpan` implements **both** `Span` (the active
recording interface) **and** `FinishedSpan` (the export data model). During the span's
lifecycle, it records tags, events, and errors as a `Span`. After `end()`, the same object
serves as a `FinishedSpan` ready for the export pipeline (Feature 2). This works for the
test implementation because there's no separate backend — the in-memory object already holds
all the data.

### Module Placement

All classes go in the `simple-tracing-test` module (`io.simpletracing.test.simple` package).
This mirrors the real framework where `micrometer-tracing-test` provides `SimpleTracer` as a
testing utility separate from the core API.

---

## 2. Client Usage & Tests

### Test: Create and end a span with tags and events

The most fundamental test — verify that a span records its name, tags, and events:

```java
@Test
void shouldCreateAndEndSpan() {
    Span span = tracer.nextSpan().name("encode").start();
    try {
        span.tag("input.length", "256");
        span.event("encoding-started");
    } finally {
        span.end();
    }

    SimpleSpan finished = tracer.onlySpan();
    assertThat(finished.getName()).isEqualTo("encode");
    assertThat(finished.getTags()).containsEntry("input.length", "256");
    assertThat(finished.getEvents()).hasSize(1);
    assertThat(finished.getEvents().iterator().next().getValue())
        .isEqualTo("encoding-started");
    assertThat(finished.isStarted()).isTrue();
    assertThat(finished.isEnded()).isTrue();
}
```

### Test: Parent-child relationships from current scope

When a span is in scope, `nextSpan()` creates a child that inherits the parent's traceId:

```java
@Test
void shouldCreateChildSpanFromCurrentScope() {
    Span parent = tracer.nextSpan().name("parent").start();
    try (Tracer.SpanInScope ws = tracer.withSpan(parent)) {
        Span child = tracer.nextSpan().name("child").start();
        child.end();

        SimpleSpan childSpan = tracer.lastSpan();
        assertThat(childSpan.getTraceId()).isEqualTo(parent.context().traceId());
        assertThat(childSpan.getParentId()).isEqualTo(parent.context().spanId());
        assertThat(childSpan.getSpanId()).isNotEqualTo(parent.context().spanId());
    } finally {
        parent.end();
    }
}
```

### Test: ScopedSpan manages both span and scope

`ScopedSpan` bundles span + scope — when it ends, both are cleaned up:

```java
@Test
void shouldCreateScopedSpanThatIsInScope() {
    ScopedSpan scoped = tracer.startScopedSpan("encode");
    try {
        scoped.tag("input.length", "256");
        scoped.event("started");
        assertThat(tracer.currentSpan()).isNotNull();
    } finally {
        scoped.end();
    }
    assertThat(tracer.currentSpan()).isNull();

    SimpleSpan finished = tracer.onlySpan();
    assertThat(finished.getName()).isEqualTo("encode");
    assertThat(finished.getTags()).containsEntry("input.length", "256");
}
```

### Test: Baggage creation and scope

Baggage is scoped — visible while in scope, cleared on close:

```java
@Test
void shouldCreateAndReadBaggageInScope() {
    Span span = tracer.nextSpan().start();
    try (Tracer.SpanInScope ws = tracer.withSpan(span)) {
        try (BaggageInScope baggage = tracer.createBaggageInScope("user.id", "12345")) {
            assertThat(baggage.name()).isEqualTo("user.id");
            assertThat(baggage.get()).isEqualTo("12345");
        }
    } finally {
        span.end();
    }
}
```

### Test: Links via SpanBuilder

`SimpleSpanBuilder` stores links — connecting this to Feature 5 (Link):

```java
@Test
void shouldAddLinksViaSpanBuilder() {
    TraceContext linkedCtx = tracer.traceContextBuilder()
        .traceId("linked-trace").spanId("linked-span").sampled(true).build();

    Span span = tracer.spanBuilder()
        .name("batch-process")
        .addLink(new Link(linkedCtx))
        .addLink(new Link(linkedCtx, Map.of("batch.index", 1)))
        .start();
    span.end();

    SimpleSpan finished = tracer.onlySpan();
    assertThat(finished.getLinks()).hasSize(2);
    assertThat(finished.getLinks().get(0).getTraceContext().traceId())
        .isEqualTo("linked-trace");
}
```

---

## 3. Implementing the Call Chain — Top-Down

### Layer 1: SimpleTraceContext — Identity with ID Generation

The foundation — a mutable `TraceContext` that generates hex IDs:

```java
public class SimpleTraceContext implements TraceContext {
    private static final Random random = new Random();
    private volatile String traceId = "";
    private volatile String parentId = "";
    private volatile String spanId = "";
    private volatile Boolean sampled = false;
    private final Map<String, String> baggageFromParent = new ConcurrentHashMap<>();

    String generateId() {
        return String.format("%016x", random.nextLong());
    }
    // ... setters, getters, baggage propagation ...
}
```

**Why mutable?** The API's `TraceContext` interface is read-only — but the implementation
needs setters because the tracer builds up context incrementally. When creating a child span,
it first creates a `SimpleTraceContext`, then sets the traceId (inherited from parent),
parentId (parent's spanId), and spanId (freshly generated).

**Why hex IDs?** The `generateId()` method produces 16-character hex strings (64-bit),
matching real tracing systems like Brave/Zipkin. The format is `%016x` — left-padded with
zeros for consistency.

### Layer 2: SimpleSpan — The Dual-Interface Record

The central data holder — records span state during its lifecycle and exposes it for export:

```java
public class SimpleSpan implements Span, FinishedSpan {
    private final Map<String, String> tags = new ConcurrentHashMap<>();
    private volatile long startMillis;
    private volatile long endMillis;
    private volatile Throwable throwable;
    private volatile String name;
    private final Queue<Event> events = new ConcurrentLinkedQueue<>();
    private final SimpleTraceContext context = new SimpleTraceContext();
    private final List<Link> links = new ArrayList<>();

    public SimpleSpan() {
        SimpleTracer.bindSpanToTraceContext(context(), this);
    }
    // ... Span methods (start, end, tag, event, error) ...
    // ... FinishedSpan methods (getName, getTags, getEvents, etc.) ...
}
```

**Why `ConcurrentHashMap` and `ConcurrentLinkedQueue`?** Although most tests are
single-threaded, the real framework's `SimpleSpan` uses concurrent collections because spans
*can* be shared across threads (e.g., a span created in one thread but ended in another). We
follow the same pattern for thread-safety.

**Constructor registration**: The constructor calls `SimpleTracer.bindSpanToTraceContext()` —
this registers the span in a static map so that `setCurrentSpan(TraceContext)` can look up
the span from its context later.

### Layer 3: SimpleCurrentTraceContext — ThreadLocal Scope Management

The scope manager that makes `withSpan()` and `startScopedSpan()` work:

```java
public class SimpleCurrentTraceContext implements CurrentTraceContext {
    private final SimpleTracer simpleTracer;

    @Override
    public Scope newScope(TraceContext context) {
        if (context == null) {
            SimpleTracer.resetCurrentSpan();
            return new RevertToNullScope();
        }
        SimpleSpan previous = SimpleTracer.getCurrentSpan();
        SimpleTracer.setCurrentSpan(context);
        // Propagate baggage from parent context
        List<BaggageInScope> baggageScopes = new ArrayList<>();
        if (context instanceof SimpleTraceContext simpleCtx) {
            for (Map.Entry<String, String> entry : simpleCtx.baggageFromParent().entrySet()) {
                baggageScopes.add(simpleTracer.createBaggageInScope(
                    context, entry.getKey(), entry.getValue()));
            }
        }
        if (previous != null) {
            return new RevertToPreviousScope(previous, baggageScopes);
        }
        return new RevertToNullScope(baggageScopes);
    }
}
```

**The bracket pattern**: `newScope()` saves the previous span, sets the new one, and returns
a `Scope` whose `close()` restores the previous span. This is a classic functional pattern —
acquire/use/restore — implemented with inner classes that capture the restoration state.

**Baggage propagation**: When entering a new scope, baggage from the parent context is
automatically propagated by creating `BaggageInScope` entries. When the scope closes, these
are also closed.

### Layer 4: SimpleTracer — The Glue

The central tracer that ties everything together:

```java
public class SimpleTracer implements Tracer {
    private static final Map<TraceContext, SimpleSpan> traceContextToSpans =
        new ConcurrentHashMap<>();
    private static final ThreadLocal<SimpleSpan> scopedSpans = new ThreadLocal<>();
    private final Deque<SimpleSpan> spans = new LinkedBlockingDeque<>();

    @Override
    public Span nextSpan() {
        SimpleSpan span = simpleSpan();
        this.spans.add(span);
        return span;
    }

    private SimpleSpan simpleSpan() {
        SimpleSpan span = new SimpleSpan();
        SimpleSpan currentSpan = scopedSpans.get();
        if (currentSpan != null) {
            span.context().setTraceId(currentSpan.context().traceId());
            span.context().setParentId(currentSpan.context().spanId());
            span.context().setSpanId(span.context().generateId());
        } else {
            String id = span.context().generateId();
            span.context().setTraceId(id);
            span.context().setSpanId(id);
        }
        return span;
    }
}
```

**Static vs instance state**: The ThreadLocal (`scopedSpans`) and context-to-span map
(`traceContextToSpans`) are **static** because there's one "current span" per thread across
all tracer instances. But the span collection (`spans`) is **per-instance** — each test's
`SimpleTracer` has its own isolated collection.

**Root vs child span IDs**: Root spans get the same value for both traceId and spanId. Child
spans inherit the parent's traceId and generate a fresh spanId.

### Layer 5: Supporting Classes

**SimpleSpanBuilder** — Accumulates configuration (name, kind, tags, parent, links) and
produces a configured `SimpleSpan` on `start()`. When a parent is set explicitly via
`setParent()`, the child inherits the parent's traceId. When no parent is set, it checks
the current scope.

**SimpleScopedSpan** — Wraps a `SimpleSpan` that is automatically created, started, and
scoped on construction. `end()` closes both the scope and the span.

**SimpleSpanCustomizer** — Dynamically resolves the current span from the tracer on each
call, then delegates to it. Throws `IllegalStateException` if no span is in scope.

**SimpleBaggageInScope** — Implements both `Baggage` and `BaggageInScope`. Tracks a single
name/value pair with scope lifecycle (`inScope` flag). The `get(TraceContext)` method only
returns the value if the baggage is in scope AND the context matches.

**SimpleBaggageManager** — Factory and lookup for baggage entries, using a
`ConcurrentHashMap<TraceContext, ThreadLocal<Set<SimpleBaggageInScope>>>` structure. Each
trace context gets its own thread-local set of baggage.

---

## 4. Try It Yourself

1. **Add `remoteIpAndPort()` support to `SimpleSpan`**: The real `SimpleSpan` tracks remote
   IP and port. Add `ip` and `port` fields, implement `remoteIpAndPort(String ip, int port)`,
   and add the corresponding `FinishedSpan` getters (`getRemoteIp()`, `getRemotePort()`).

2. **Test concurrent span creation**: Write a test that creates spans from multiple threads
   simultaneously and verify that `tracer.getSpans()` contains all of them (tests the
   thread-safety of the `LinkedBlockingDeque`).

3. **Add span-in-scope nesting depth tracking**: Modify `SimpleCurrentTraceContext` to track
   the nesting depth (how many scopes are open). This helps catch scope leaks in tests.

---

## 5. Why This Works

### The Facade Makes Testing Possible
The entire test module exists because the API is defined as interfaces. Production code
depends on `Tracer` (the interface), and tests swap in `SimpleTracer` (the in-memory
implementation). This is the **dependency inversion principle** at framework scale — the
application doesn't know or care whether spans go to Zipkin or an in-memory list.

### Static ThreadLocal for Cross-Class Scope
The "current span" ThreadLocal is static on `SimpleTracer`, not on
`SimpleCurrentTraceContext`. This is deliberate — multiple objects need to read/write the
current span (the tracer, the context manager, the span customizer), and a static ThreadLocal
is the simplest way to share this state without passing references everywhere. The real
framework uses the same pattern.

### Dual-Interface Simplification
`SimpleSpan implements Span, FinishedSpan` is a simplification unique to the test module.
In a real bridge (Brave, OpenTelemetry), the active span and the finished span are different
objects because the backend's internal representation differs from the export format. But for
in-memory testing, one object serving both roles keeps things simple while still exercising
both API contracts.

---

## 6. What We Enhanced

| Component | State After Ch06 | Simplifications |
|-----------|-------------------|-----------------|
| `SimpleTracer` | Full `Tracer` + `BaggageManager` implementation with span collection | No sampled-based filtering; all spans recorded |
| `SimpleSpan` | `Span` + `FinishedSpan` dual implementation | No `remoteIpAndPort()`; no `localServiceName`; `Event` uses millis only |
| `SimpleSpanBuilder` | Full `Span.Builder` with parent, kind, tags, links | No `startTimestamp()` support |
| `SimpleScopedSpan` | `ScopedSpan` with automatic scope management | Identical to real framework |
| `SimpleTraceContext` | Mutable `TraceContext` with hex ID generation | Uses `Random` instead of `ThreadLocalRandom`; no 128-bit traceId |
| `SimpleTraceContextBuilder` | `TraceContext.Builder` → `SimpleTraceContext` | Identical to real framework |
| `SimpleCurrentTraceContext` | ThreadLocal scope with previous-span restoration | No `wrap(Callable/Runnable/Executor)` delegation |
| `SimpleSpanCustomizer` | Dynamic current-span delegation | Identical to real framework |
| `SimpleBaggageInScope` | `Baggage` + `BaggageInScope` dual implementation | Simplified context matching (reference equality) |
| `SimpleBaggageManager` | Per-context ThreadLocal baggage sets | No remote field propagation |

**What this feature enables**: With `SimpleTracer` in place, every API interface from Features
1–5 is now testable without mocks. Feature 7 (Test Assertions) builds on this to provide
fluent AssertJ-style assertions. Feature 8 (Brave Bridge) uses `SimpleTracer` in its test
suite to verify that Brave's tracing delegates correctly through the facade.

---

## 7. Connection to Real Framework

| Simplified Class | Real Framework Class | Source File |
|------------------|----------------------|-------------|
| `io.simpletracing.test.simple.SimpleTracer` | `io.micrometer.tracing.test.simple.SimpleTracer` | `micrometer-tracing-test/src/main/java/.../SimpleTracer.java` |
| `io.simpletracing.test.simple.SimpleSpan` | `io.micrometer.tracing.test.simple.SimpleSpan` | `micrometer-tracing-test/src/main/java/.../SimpleSpan.java` |
| `io.simpletracing.test.simple.SimpleSpanBuilder` | `io.micrometer.tracing.test.simple.SimpleSpanBuilder` | `micrometer-tracing-test/src/main/java/.../SimpleSpanBuilder.java` |
| `io.simpletracing.test.simple.SimpleScopedSpan` | `io.micrometer.tracing.test.simple.SimpleScopedSpan` | `micrometer-tracing-test/src/main/java/.../SimpleScopedSpan.java` |
| `io.simpletracing.test.simple.SimpleTraceContext` | `io.micrometer.tracing.test.simple.SimpleTraceContext` | `micrometer-tracing-test/src/main/java/.../SimpleTraceContext.java` |
| `io.simpletracing.test.simple.SimpleTraceContextBuilder` | `io.micrometer.tracing.test.simple.SimpleTraceContextBuilder` | `micrometer-tracing-test/src/main/java/.../SimpleTraceContextBuilder.java` |
| `io.simpletracing.test.simple.SimpleCurrentTraceContext` | `io.micrometer.tracing.test.simple.SimpleCurrentTraceContext` | `micrometer-tracing-test/src/main/java/.../SimpleCurrentTraceContext.java` |
| `io.simpletracing.test.simple.SimpleSpanCustomizer` | `io.micrometer.tracing.test.simple.SimpleSpanCustomizer` | `micrometer-tracing-test/src/main/java/.../SimpleSpanCustomizer.java` |
| `io.simpletracing.test.simple.SimpleBaggageInScope` | `io.micrometer.tracing.test.simple.SimpleBaggageInScope` | `micrometer-tracing-test/src/main/java/.../SimpleBaggageInScope.java` |
| `io.simpletracing.test.simple.SimpleBaggageManager` | `io.micrometer.tracing.test.simple.SimpleBaggageManager` | `micrometer-tracing-test/src/main/java/.../SimpleBaggageManager.java` |

**Real framework extras we skipped**:
- `SimpleSpan.remoteIpAndPort()`, `localServiceName` — extra fields not needed for core testing
- `SimpleSpan.Clock` — the real framework uses an injectable `Clock` for deterministic timestamps
- `SimpleCurrentTraceContext.wrap(Callable/Runnable/Executor/ExecutorService)` — context propagation across threads (returns unwrapped for simplicity)
- `EncodingUtils.fromLong()` — the real framework uses a dedicated hex encoding utility; we use `String.format()`
- `SimpleBaggageInScope` deprecated constructor overloads from the real framework
- `SimpleBaggageManager.remoteFields` — field list for propagation configuration

---

## 8. Complete Code

### `simple-tracing-test/src/main/java/io/simpletracing/test/simple/SimpleTraceContext.java` [NEW]

```java
package io.simpletracing.test.simple;

import java.util.Map;
import java.util.Random;
import java.util.concurrent.ConcurrentHashMap;

import io.simpletracing.TraceContext;

/**
 * In-memory implementation of {@link TraceContext}. Unlike the API interface (which is
 * read-only), this implementation has setters — needed because the tracer builds up
 * the context incrementally (generate IDs, set parent, inherit traceId).
 *
 * <p>ID generation uses {@link Random} to produce 16-character hex strings, matching
 * the format used by real tracing systems (64-bit hex-encoded span/trace IDs).
 *
 * <p>Maps to: {@code io.micrometer.tracing.test.simple.SimpleTraceContext}
 */
public class SimpleTraceContext implements TraceContext {

    private static final Random random = new Random();

    private volatile String traceId = "";

    private volatile String parentId = "";

    private volatile String spanId = "";

    private volatile Boolean sampled = false;

    private final Map<String, String> baggageFromParent = new ConcurrentHashMap<>();

    @Override
    public String traceId() {
        return this.traceId;
    }

    @Override
    public String parentId() {
        return this.parentId;
    }

    @Override
    public String spanId() {
        return this.spanId;
    }

    @Override
    public Boolean sampled() {
        return this.sampled;
    }

    public void setTraceId(String traceId) {
        this.traceId = traceId;
    }

    public void setParentId(String parentId) {
        this.parentId = parentId;
    }

    public void setSpanId(String spanId) {
        this.spanId = spanId;
    }

    public void setSampled(Boolean sampled) {
        this.sampled = sampled;
    }

    /**
     * Generates a 16-character hex ID from a random long. This matches the format
     * used by Brave and Zipkin for 64-bit trace/span identifiers.
     * @return a 16-character lowercase hex string
     */
    String generateId() {
        return String.format("%016x", random.nextLong());
    }

    /**
     * Adds baggage entries inherited from the parent span's context. This is used
     * during span creation to propagate baggage down the call chain.
     * @param baggage parent baggage entries to copy
     */
    public void addParentBaggage(Map<String, String> baggage) {
        this.baggageFromParent.putAll(baggage);
    }

    /**
     * Returns baggage entries inherited from the parent context.
     * @return parent baggage map (never null)
     */
    public Map<String, String> baggageFromParent() {
        return this.baggageFromParent;
    }

}
```

### `simple-tracing-test/src/main/java/io/simpletracing/test/simple/SimpleTraceContextBuilder.java` [NEW]

```java
package io.simpletracing.test.simple;

import io.simpletracing.TraceContext;

/**
 * In-memory implementation of {@link TraceContext.Builder}. Accumulates fields and
 * produces a {@link SimpleTraceContext} on {@link #build()}.
 *
 * <p>Only non-null fields are set on the built context — unset fields retain their
 * defaults (empty string for IDs, {@code false} for sampled).
 *
 * <p>Maps to: {@code io.micrometer.tracing.test.simple.SimpleTraceContextBuilder}
 */
public class SimpleTraceContextBuilder implements TraceContext.Builder {

    private String traceId;

    private String parentId;

    private String spanId;

    private Boolean sampled;

    @Override
    public TraceContext.Builder traceId(String traceId) {
        this.traceId = traceId;
        return this;
    }

    @Override
    public TraceContext.Builder parentId(String parentId) {
        this.parentId = parentId;
        return this;
    }

    @Override
    public TraceContext.Builder spanId(String spanId) {
        this.spanId = spanId;
        return this;
    }

    @Override
    public TraceContext.Builder sampled(Boolean sampled) {
        this.sampled = sampled;
        return this;
    }

    @Override
    public TraceContext build() {
        SimpleTraceContext context = new SimpleTraceContext();
        if (this.traceId != null) {
            context.setTraceId(this.traceId);
        }
        if (this.parentId != null) {
            context.setParentId(this.parentId);
        }
        if (this.spanId != null) {
            context.setSpanId(this.spanId);
        }
        if (this.sampled != null) {
            context.setSampled(this.sampled);
        }
        return context;
    }

}
```

### `simple-tracing-test/src/main/java/io/simpletracing/test/simple/SimpleSpan.java` [NEW]

```java
package io.simpletracing.test.simple;

import java.time.Instant;
import java.util.AbstractMap;
import java.util.ArrayList;
import java.util.Collection;
import java.util.List;
import java.util.Map;
import java.util.Queue;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.ConcurrentLinkedQueue;

import io.simpletracing.Link;
import io.simpletracing.Span;
import io.simpletracing.TraceContext;
import io.simpletracing.exporter.FinishedSpan;

/**
 * In-memory implementation of {@link Span} that also implements {@link FinishedSpan}.
 * Records all span data (tags, events, errors, timestamps) in memory so tests can
 * verify tracing behavior.
 *
 * <p>During the span's active lifecycle, it behaves as a {@link Span} — accepting
 * tags, events, errors, and lifecycle calls (start/end/abandon). After {@link #end()},
 * the same object serves as a {@link FinishedSpan} for the export pipeline.
 *
 * <p>Maps to: {@code io.micrometer.tracing.test.simple.SimpleSpan}
 */
public class SimpleSpan implements Span, FinishedSpan {

    private final Map<String, String> tags = new ConcurrentHashMap<>();

    private volatile boolean abandoned;

    private volatile long startMillis;

    private volatile long endMillis;

    private volatile Throwable throwable;

    private volatile String remoteServiceName;

    private volatile Span.Kind spanKind;

    private final Queue<Event> events = new ConcurrentLinkedQueue<>();

    private volatile String name;

    private volatile boolean noop;

    private final SimpleTraceContext context = new SimpleTraceContext();

    private final List<Link> links = new ArrayList<>();

    public SimpleSpan() {
        SimpleTracer.bindSpanToTraceContext(context(), this);
    }

    // ── Span methods ────────────────────────────────────────────────────

    @Override
    public boolean isNoop() {
        return this.noop;
    }

    public void setNoop(boolean noop) {
        this.noop = noop;
    }

    @Override
    public SimpleTraceContext context() {
        return this.context;
    }

    @Override
    public Span start() {
        this.startMillis = System.currentTimeMillis();
        return this;
    }

    @Override
    public Span name(String name) {
        this.name = name;
        return this;
    }

    @Override
    public Span event(String value) {
        this.events.add(new Event(System.currentTimeMillis(), value));
        return this;
    }

    @Override
    public Span tag(String key, String value) {
        this.tags.put(key, value);
        return this;
    }

    @Override
    public Span error(Throwable throwable) {
        this.throwable = throwable;
        return this;
    }

    @Override
    public void end() {
        this.endMillis = System.currentTimeMillis();
    }

    @Override
    public void abandon() {
        this.abandoned = true;
    }

    @Override
    public Span remoteServiceName(String remoteServiceName) {
        this.remoteServiceName = remoteServiceName;
        return this;
    }

    // ── FinishedSpan methods ────────────────────────────────────────────

    @Override
    public String getName() {
        return this.name;
    }

    @Override
    public FinishedSpan setName(String name) {
        this.name = name;
        return this;
    }

    @Override
    public Instant getStartTimestamp() {
        return Instant.ofEpochMilli(this.startMillis);
    }

    @Override
    public Instant getEndTimestamp() {
        return Instant.ofEpochMilli(this.endMillis);
    }

    @Override
    public Map<String, String> getTags() {
        return this.tags;
    }

    @Override
    public FinishedSpan setTags(Map<String, String> tags) {
        this.tags.clear();
        this.tags.putAll(tags);
        return this;
    }

    @Override
    public Collection<Map.Entry<Long, String>> getEvents() {
        return this.events.stream().map(Event::toEntry).toList();
    }

    @Override
    public FinishedSpan setEvents(Collection<Map.Entry<Long, String>> events) {
        this.events.clear();
        events.forEach(e -> this.events.add(new Event(e.getKey(), e.getValue())));
        return this;
    }

    @Override
    public String getSpanId() {
        return this.context.spanId();
    }

    @Override
    public String getParentId() {
        return this.context.parentId();
    }

    @Override
    public String getTraceId() {
        return this.context.traceId();
    }

    @Override
    public Throwable getError() {
        return this.throwable;
    }

    @Override
    public FinishedSpan setError(Throwable error) {
        this.throwable = error;
        return this;
    }

    @Override
    public Span.Kind getKind() {
        return this.spanKind;
    }

    @Override
    public String getRemoteServiceName() {
        return this.remoteServiceName;
    }

    @Override
    public FinishedSpan setRemoteServiceName(String remoteServiceName) {
        this.remoteServiceName = remoteServiceName;
        return this;
    }

    // ── Additional accessors for tests ──────────────────────────────────

    public boolean isAbandoned() {
        return this.abandoned;
    }

    public boolean isStarted() {
        return this.startMillis > 0;
    }

    public boolean isEnded() {
        return this.endMillis > 0;
    }

    public List<Link> getLinks() {
        return this.links;
    }

    void setSpanKind(Span.Kind kind) {
        this.spanKind = kind;
    }

    // ── Inner: Event ────────────────────────────────────────────────────

    /**
     * A timestamped event recorded on this span.
     */
    static class Event {

        private final long timestamp;

        private final String eventName;

        Event(long timestamp, String eventName) {
            this.timestamp = timestamp;
            this.eventName = eventName;
        }

        Map.Entry<Long, String> toEntry() {
            return new AbstractMap.SimpleEntry<>(this.timestamp, this.eventName);
        }

        public String getEventName() {
            return this.eventName;
        }

        public long getTimestamp() {
            return this.timestamp;
        }

    }

    @Override
    public String toString() {
        return "SimpleSpan{name='" + this.name + "', traceId='" + this.context.traceId()
                + "', spanId='" + this.context.spanId() + "'}";
    }

}
```

### `simple-tracing-test/src/main/java/io/simpletracing/test/simple/SimpleCurrentTraceContext.java` [NEW]

```java
package io.simpletracing.test.simple;

import java.util.ArrayList;
import java.util.List;
import java.util.Map;

import io.simpletracing.BaggageInScope;
import io.simpletracing.CurrentTraceContext;
import io.simpletracing.TraceContext;

/**
 * In-memory implementation of {@link CurrentTraceContext} using a ThreadLocal in
 * {@link SimpleTracer} to track the "current" span. Manages scope transitions by
 * saving and restoring previous spans.
 *
 * <p>When entering a new scope, baggage from the parent context is propagated by
 * creating {@link BaggageInScope} entries. When the scope is closed, these baggage
 * scopes are also closed.
 *
 * <p>Maps to: {@code io.micrometer.tracing.test.simple.SimpleCurrentTraceContext}
 */
public class SimpleCurrentTraceContext implements CurrentTraceContext {

    private final SimpleTracer simpleTracer;

    public SimpleCurrentTraceContext(SimpleTracer simpleTracer) {
        this.simpleTracer = simpleTracer;
    }

    @Override
    public TraceContext context() {
        SimpleSpan currentSpan = this.simpleTracer.currentSimpleSpan();
        return currentSpan != null ? currentSpan.context() : null;
    }

    @Override
    public Scope newScope(TraceContext context) {
        if (context == null) {
            SimpleTracer.resetCurrentSpan();
            return new RevertToNullScope();
        }
        SimpleSpan previous = SimpleTracer.getCurrentSpan();
        SimpleTracer.setCurrentSpan(context);
        // Propagate baggage from the parent context
        List<BaggageInScope> baggageScopes = new ArrayList<>();
        if (context instanceof SimpleTraceContext simpleCtx) {
            Map<String, String> parentBaggage = simpleCtx.baggageFromParent();
            for (Map.Entry<String, String> entry : parentBaggage.entrySet()) {
                BaggageInScope scope = this.simpleTracer.createBaggageInScope(
                        context, entry.getKey(), entry.getValue());
                baggageScopes.add(scope);
            }
        }
        if (previous != null) {
            return new RevertToPreviousScope(previous, baggageScopes);
        }
        return new RevertToNullScope(baggageScopes);
    }

    @Override
    public Scope maybeScope(TraceContext context) {
        if (context == null) {
            SimpleTracer.resetCurrentSpan();
            return Scope.NOOP;
        }
        SimpleSpan currentSpan = this.simpleTracer.currentSimpleSpan();
        if (currentSpan != null && currentSpan.context() == context) {
            return Scope.NOOP;
        }
        return newScope(context);
    }

    // ── Inner: RevertToPreviousScope ────────────────────────────────────

    private static class RevertToPreviousScope implements Scope {

        private final SimpleSpan previous;

        private final List<BaggageInScope> baggageScopes;

        RevertToPreviousScope(SimpleSpan previous, List<BaggageInScope> baggageScopes) {
            this.previous = previous;
            this.baggageScopes = baggageScopes;
        }

        @Override
        public void close() {
            SimpleTracer.setCurrentSpan(this.previous);
            for (BaggageInScope scope : this.baggageScopes) {
                scope.close();
            }
        }

    }

    // ── Inner: RevertToNullScope ────────────────────────────────────────

    private static class RevertToNullScope implements Scope {

        private final List<BaggageInScope> baggageScopes;

        RevertToNullScope() {
            this.baggageScopes = List.of();
        }

        RevertToNullScope(List<BaggageInScope> baggageScopes) {
            this.baggageScopes = baggageScopes;
        }

        @Override
        public void close() {
            SimpleTracer.resetCurrentSpan();
            for (BaggageInScope scope : this.baggageScopes) {
                scope.close();
            }
        }

    }

}
```

### `simple-tracing-test/src/main/java/io/simpletracing/test/simple/SimpleSpanBuilder.java` [NEW]

```java
package io.simpletracing.test.simple;

import java.util.ArrayList;
import java.util.HashMap;
import java.util.List;
import java.util.Map;

import io.simpletracing.Link;
import io.simpletracing.Span;
import io.simpletracing.TraceContext;

/**
 * In-memory implementation of {@link Span.Builder}. Accumulates configuration (name,
 * kind, tags, parent, links) and produces a {@link SimpleSpan} on {@link #start()}.
 *
 * <p>When {@link #start()} is called, a new {@link SimpleSpan} is created and all
 * accumulated configuration is transferred to it. If a parent context was set, the new
 * span inherits its traceId and uses the parent's spanId as its parentId.
 *
 * <p>Maps to: {@code io.micrometer.tracing.test.simple.SimpleSpanBuilder}
 */
public class SimpleSpanBuilder implements Span.Builder {

    private final List<String> events = new ArrayList<>();

    private final Map<String, String> tags = new HashMap<>();

    private final List<Link> links = new ArrayList<>();

    private Throwable throwable;

    private String remoteServiceName;

    private Span.Kind spanKind;

    private String name;

    private final SimpleTracer simpleTracer;

    private TraceContext parentContext;

    private boolean noParent;

    public SimpleSpanBuilder(SimpleTracer simpleTracer) {
        this.simpleTracer = simpleTracer;
    }

    @Override
    public Span.Builder setParent(TraceContext context) {
        this.parentContext = context;
        this.noParent = false;
        return this;
    }

    @Override
    public Span.Builder setNoParent() {
        this.noParent = true;
        this.parentContext = null;
        return this;
    }

    @Override
    public Span.Builder name(String name) {
        this.name = name;
        return this;
    }

    @Override
    public Span.Builder event(String value) {
        this.events.add(value);
        return this;
    }

    @Override
    public Span.Builder tag(String key, String value) {
        this.tags.put(key, value);
        return this;
    }

    @Override
    public Span.Builder error(Throwable throwable) {
        this.throwable = throwable;
        return this;
    }

    @Override
    public Span.Builder kind(Span.Kind spanKind) {
        this.spanKind = spanKind;
        return this;
    }

    @Override
    public Span.Builder remoteServiceName(String remoteServiceName) {
        this.remoteServiceName = remoteServiceName;
        return this;
    }

    @Override
    public Span.Builder addLink(Link link) {
        this.links.add(link);
        return this;
    }

    @Override
    public Span start() {
        SimpleSpan span = new SimpleSpan();
        // Apply accumulated configuration
        if (this.name != null) {
            span.name(this.name);
        }
        if (this.spanKind != null) {
            span.setSpanKind(this.spanKind);
        }
        if (this.remoteServiceName != null) {
            span.remoteServiceName(this.remoteServiceName);
        }
        if (this.throwable != null) {
            span.error(this.throwable);
        }
        this.tags.forEach(span::tag);
        this.events.forEach(span::event);
        this.links.forEach(link -> span.getLinks().add(link));
        // Set parent-child relationships
        if (!this.noParent) {
            TraceContext parent = this.parentContext;
            if (parent == null) {
                // Check current context
                parent = this.simpleTracer.currentTraceContext().context();
            }
            if (parent != null) {
                span.context().setTraceId(parent.traceId());
                span.context().setParentId(parent.spanId());
                span.context().setSpanId(span.context().generateId());
                span.context().setSampled(parent.sampled());
                this.simpleTracer.getSpans().add(span);
                return span.start();
            }
        }
        // Root span — generate new IDs
        String id = span.context().generateId();
        span.context().setTraceId(id);
        span.context().setSpanId(id);
        this.simpleTracer.getSpans().add(span);
        return span.start();
    }

}
```

### `simple-tracing-test/src/main/java/io/simpletracing/test/simple/SimpleScopedSpan.java` [NEW]

```java
package io.simpletracing.test.simple;

import io.simpletracing.ScopedSpan;
import io.simpletracing.Tracer;
import io.simpletracing.TraceContext;

/**
 * In-memory implementation of {@link ScopedSpan}. Wraps a {@link SimpleSpan} that is
 * automatically created, started, and placed in scope upon construction. When
 * {@link #end()} is called, both the span and its scope are closed.
 *
 * <p>This is the "convenience" API — a single object that manages both the span and
 * its scope. The {@link Tracer#startScopedSpan(String)} method returns one of these.
 *
 * <p>Maps to: {@code io.micrometer.tracing.test.simple.SimpleScopedSpan}
 */
public class SimpleScopedSpan implements ScopedSpan {

    private final SimpleSpan span;

    private final Tracer.SpanInScope scopeHandle;

    public SimpleScopedSpan(SimpleTracer simpleTracer) {
        this.span = (SimpleSpan) simpleTracer.nextSpan().start();
        this.scopeHandle = simpleTracer.withSpan(this.span);
    }

    @Override
    public boolean isNoop() {
        return this.span.isNoop();
    }

    @Override
    public TraceContext context() {
        return this.span.context();
    }

    @Override
    public ScopedSpan name(String name) {
        this.span.name(name);
        return this;
    }

    @Override
    public ScopedSpan tag(String key, String value) {
        this.span.tag(key, value);
        return this;
    }

    @Override
    public ScopedSpan event(String value) {
        this.span.event(value);
        return this;
    }

    @Override
    public ScopedSpan error(Throwable throwable) {
        this.span.error(throwable);
        return this;
    }

    @Override
    public void end() {
        this.scopeHandle.close();
        this.span.end();
    }

}
```

### `simple-tracing-test/src/main/java/io/simpletracing/test/simple/SimpleSpanCustomizer.java` [NEW]

```java
package io.simpletracing.test.simple;

import io.simpletracing.Span;
import io.simpletracing.SpanCustomizer;
import io.simpletracing.Tracer;

/**
 * In-memory implementation of {@link SpanCustomizer}. Dynamically resolves the current
 * span from the tracer on each method call, then delegates customization to it.
 *
 * <p>This is the "lightweight" customization interface — business code that receives a
 * {@code SpanCustomizer} can annotate the current span (name, tag, event) but cannot
 * start or stop it. The dynamic lookup means it always operates on whatever span is
 * currently in scope, even if scope changes between calls.
 *
 * <p>Maps to: {@code io.micrometer.tracing.test.simple.SimpleSpanCustomizer}
 */
public class SimpleSpanCustomizer implements SpanCustomizer {

    private final Tracer tracer;

    public SimpleSpanCustomizer(SimpleTracer tracer) {
        this.tracer = tracer;
    }

    @Override
    public SpanCustomizer name(String name) {
        currentSpan().name(name);
        return this;
    }

    @Override
    public SpanCustomizer tag(String key, String value) {
        currentSpan().tag(key, value);
        return this;
    }

    @Override
    public SpanCustomizer event(String value) {
        currentSpan().event(value);
        return this;
    }

    private Span currentSpan() {
        Span current = this.tracer.currentSpan();
        if (current == null) {
            throw new IllegalStateException("No span is currently in scope");
        }
        return current;
    }

}
```

### `simple-tracing-test/src/main/java/io/simpletracing/test/simple/SimpleBaggageInScope.java` [NEW]

```java
package io.simpletracing.test.simple;

import java.util.Objects;

import io.simpletracing.Baggage;
import io.simpletracing.BaggageInScope;
import io.simpletracing.CurrentTraceContext;
import io.simpletracing.TraceContext;

/**
 * In-memory implementation of both {@link Baggage} and {@link BaggageInScope}. Tracks
 * a single baggage entry (name + value) with scope lifecycle (in-scope / closed).
 *
 * <p>The scope semantics: when {@link #makeCurrent()} is called, this baggage becomes
 * "in scope" and its value is visible via {@link #get()}. When {@link #close()} is
 * called, it exits scope. The {@link #get(TraceContext)} method only returns the value
 * if the baggage is in scope AND the trace context matches.
 *
 * <p>Maps to: {@code io.micrometer.tracing.test.simple.SimpleBaggageInScope}
 */
public class SimpleBaggageInScope implements Baggage, BaggageInScope {

    private final String name;

    private volatile TraceContext traceContext;

    private volatile String value;

    private volatile boolean inScope;

    private volatile boolean closed;

    private CurrentTraceContext currentTraceContext = CurrentTraceContext.NOOP;

    public SimpleBaggageInScope(CurrentTraceContext currentTraceContext, String name,
            TraceContext traceContext) {
        this.currentTraceContext = currentTraceContext;
        this.name = name;
        this.traceContext = traceContext;
    }

    public SimpleBaggageInScope(CurrentTraceContext currentTraceContext, String name) {
        this.currentTraceContext = currentTraceContext;
        this.name = name;
        this.traceContext = currentTraceContext.context();
    }

    public SimpleBaggageInScope(String name) {
        this.name = name;
    }

    @Override
    public String name() {
        return this.name;
    }

    @Override
    public String get() {
        TraceContext ctx = this.currentTraceContext.context();
        if (ctx == null) {
            return null;
        }
        return get(ctx);
    }

    @Override
    public String get(TraceContext traceContext) {
        if (!this.inScope) {
            return null;
        }
        if (this.traceContext == traceContext) {
            return this.value;
        }
        return null;
    }

    @Override
    public Baggage set(String value) {
        this.value = value;
        this.traceContext = this.currentTraceContext.context();
        return this;
    }

    @Override
    public Baggage set(TraceContext traceContext, String value) {
        this.value = value;
        this.traceContext = traceContext;
        return this;
    }

    @Override
    public BaggageInScope makeCurrent() {
        this.inScope = true;
        return this;
    }

    @Override
    public BaggageInScope makeCurrent(String value) {
        this.value = value;
        return makeCurrent();
    }

    @Override
    public BaggageInScope makeCurrent(TraceContext traceContext, String value) {
        this.value = value;
        this.traceContext = traceContext;
        return makeCurrent();
    }

    @Override
    public void close() {
        this.inScope = false;
        this.closed = true;
    }

    public boolean isClosed() {
        return this.closed;
    }

    public boolean isInScope() {
        return this.inScope;
    }

    @Override
    public boolean equals(Object o) {
        if (this == o) {
            return true;
        }
        if (o == null || getClass() != o.getClass()) {
            return false;
        }
        SimpleBaggageInScope that = (SimpleBaggageInScope) o;
        return Objects.equals(this.name, that.name);
    }

    @Override
    public int hashCode() {
        return Objects.hash(this.name);
    }

}
```

### `simple-tracing-test/src/main/java/io/simpletracing/test/simple/SimpleBaggageManager.java` [NEW]

```java
package io.simpletracing.test.simple;

import java.util.Collections;
import java.util.LinkedHashMap;
import java.util.LinkedHashSet;
import java.util.List;
import java.util.Map;
import java.util.Set;
import java.util.concurrent.ConcurrentHashMap;

import io.simpletracing.Baggage;
import io.simpletracing.BaggageInScope;
import io.simpletracing.BaggageManager;
import io.simpletracing.TraceContext;

/**
 * In-memory implementation of {@link BaggageManager}. Manages baggage entries per
 * trace context using a nested {@code ConcurrentHashMap<TraceContext, ThreadLocal<Set>>}
 * structure.
 *
 * <p>Each trace context gets its own thread-local set of {@link SimpleBaggageInScope}
 * entries. This allows baggage to be both context-specific (different traces can have
 * different baggage) and thread-safe (each thread has its own view).
 *
 * <p>Maps to: {@code io.micrometer.tracing.test.simple.SimpleBaggageManager}
 */
public class SimpleBaggageManager implements BaggageManager {

    private final Map<TraceContext, ThreadLocal<Set<SimpleBaggageInScope>>> baggagesByContext =
            new ConcurrentHashMap<>();

    private final SimpleTracer simpleTracer;

    private final List<String> remoteFields;

    public SimpleBaggageManager(SimpleTracer simpleTracer) {
        this.simpleTracer = simpleTracer;
        this.remoteFields = List.of();
    }

    public SimpleBaggageManager(SimpleTracer simpleTracer, List<String> remoteFields) {
        this.simpleTracer = simpleTracer;
        this.remoteFields = remoteFields;
    }

    @Override
    public Map<String, String> getAllBaggage() {
        TraceContext ctx = this.simpleTracer.currentTraceContext().context();
        if (ctx == null) {
            return Collections.emptyMap();
        }
        return getAllBaggageForCtx(ctx);
    }

    @Override
    public Map<String, String> getAllBaggage(TraceContext traceContext) {
        if (traceContext == null) {
            return getAllBaggage();
        }
        return getAllBaggageForCtx(traceContext);
    }

    private Map<String, String> getAllBaggageForCtx(TraceContext traceContext) {
        Map<String, String> result = new LinkedHashMap<>();
        Set<SimpleBaggageInScope> baggages = getBaggagesForContext(traceContext);
        for (SimpleBaggageInScope baggage : baggages) {
            String value = baggage.get(traceContext);
            if (value != null) {
                result.put(baggage.name(), value);
            }
        }
        // Also include baggage from parent context
        if (traceContext instanceof SimpleTraceContext simpleCtx) {
            result.putAll(simpleCtx.baggageFromParent());
        }
        return Collections.unmodifiableMap(result);
    }

    @Override
    public Baggage getBaggage(String name) {
        TraceContext ctx = this.simpleTracer.currentTraceContext().context();
        if (ctx == null) {
            return Baggage.NOOP;
        }
        return getBaggage(ctx, name);
    }

    @Override
    public Baggage getBaggage(TraceContext traceContext, String name) {
        SimpleBaggageInScope found = baggageForName(traceContext, name);
        if (found != null) {
            return found;
        }
        return Baggage.NOOP;
    }

    private SimpleBaggageInScope baggageForName(TraceContext traceContext, String name) {
        Set<SimpleBaggageInScope> baggages = getBaggagesForContext(traceContext);
        for (SimpleBaggageInScope baggage : baggages) {
            if (baggage.name().equalsIgnoreCase(name)) {
                return baggage;
            }
        }
        return null;
    }

    @Override
    public Baggage createBaggage(String name) {
        return createSimpleBaggage(name);
    }

    @Override
    public Baggage createBaggage(String name, String value) {
        SimpleBaggageInScope baggage = createSimpleBaggage(name);
        baggage.set(value);
        return baggage;
    }

    @Override
    public BaggageInScope createBaggageInScope(String name, String value) {
        SimpleBaggageInScope baggage = createSimpleBaggage(name);
        return baggage.makeCurrent(value);
    }

    @Override
    public BaggageInScope createBaggageInScope(TraceContext traceContext, String name,
            String value) {
        SimpleBaggageInScope baggage = createSimpleBaggage(name);
        return baggage.makeCurrent(traceContext, value);
    }

    SimpleBaggageInScope createSimpleBaggage(String name) {
        TraceContext ctx = this.simpleTracer.currentTraceContext().context();
        if (ctx != null) {
            SimpleBaggageInScope existing = baggageForName(ctx, name);
            if (existing != null) {
                return existing;
            }
        }
        SimpleBaggageInScope baggage = new SimpleBaggageInScope(
                this.simpleTracer.currentTraceContext(), name);
        if (ctx != null) {
            Set<SimpleBaggageInScope> baggages = getBaggagesForContext(ctx);
            baggages.add(baggage);
        }
        return baggage;
    }

    private Set<SimpleBaggageInScope> getBaggagesForContext(TraceContext traceContext) {
        return this.baggagesByContext
                .computeIfAbsent(traceContext,
                        k -> ThreadLocal.withInitial(LinkedHashSet::new))
                .get();
    }

    public List<String> getBaggageFields() {
        return this.remoteFields;
    }

}
```

### `simple-tracing-test/src/main/java/io/simpletracing/test/simple/SimpleTracer.java` [NEW]

```java
package io.simpletracing.test.simple;

import java.util.Deque;
import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.LinkedBlockingDeque;

import io.simpletracing.Baggage;
import io.simpletracing.BaggageInScope;
import io.simpletracing.CurrentTraceContext;
import io.simpletracing.ScopedSpan;
import io.simpletracing.Span;
import io.simpletracing.SpanCustomizer;
import io.simpletracing.TraceContext;
import io.simpletracing.Tracer;

/**
 * Fully functional in-memory implementation of {@link Tracer}. Provides span creation,
 * scope management, baggage, and test-assertion helpers — everything needed to verify
 * tracing behavior without a real backend.
 *
 * <p>This is the central class of the test module. It collects all created spans in a
 * {@link Deque} and exposes them via {@link #onlySpan()}, {@link #lastSpan()}, and
 * {@link #getSpans()} for test assertions.
 *
 * <p><b>Thread-local state</b>: The "current span" is tracked via a static
 * {@link ThreadLocal}, shared across all {@code SimpleTracer} instances. This mirrors
 * the real framework's behavior where there is one "current span" per thread. A static
 * map also tracks the association between {@link TraceContext} and {@link SimpleSpan},
 * so that scope restoration can look up the span from its context.
 *
 * <p>Maps to: {@code io.micrometer.tracing.test.simple.SimpleTracer}
 */
public class SimpleTracer implements Tracer {

    // ── Static thread-local state ───────────────────────────────────────

    private static final Map<TraceContext, SimpleSpan> traceContextToSpans =
            new ConcurrentHashMap<>();

    private static final ThreadLocal<SimpleSpan> scopedSpans = new ThreadLocal<>();

    static void bindSpanToTraceContext(TraceContext context, SimpleSpan span) {
        traceContextToSpans.put(context, span);
    }

    static SimpleSpan getCurrentSpan() {
        return scopedSpans.get();
    }

    static void setCurrentSpan(SimpleSpan span) {
        scopedSpans.set(span);
    }

    static void setCurrentSpan(TraceContext context) {
        SimpleSpan span = traceContextToSpans.get(context);
        if (span != null) {
            scopedSpans.set(span);
        }
    }

    static void resetCurrentSpan() {
        scopedSpans.remove();
    }

    // ── Instance fields ─────────────────────────────────────────────────

    private final SimpleCurrentTraceContext currentTraceContext =
            new SimpleCurrentTraceContext(this);

    final SimpleBaggageManager simpleBaggageManager = new SimpleBaggageManager(this);

    private final Deque<SimpleSpan> spans = new LinkedBlockingDeque<>();

    // ── Tracer: span creation ───────────────────────────────────────────

    @Override
    public Span nextSpan() {
        SimpleSpan span = simpleSpan();
        this.spans.add(span);
        return span;
    }

    @Override
    public Span nextSpan(Span parent) {
        if (parent == null) {
            return nextSpan();
        }
        SimpleSpan span = simpleSpan();
        span.context().setTraceId(parent.context().traceId());
        span.context().setParentId(parent.context().spanId());
        span.context().setSpanId(span.context().generateId());
        span.context().setSampled(parent.context().sampled());
        // Copy baggage from parent
        if (parent.context() instanceof SimpleTraceContext parentCtx) {
            Map<String, String> parentBaggage =
                    this.simpleBaggageManager.getAllBaggage(parent.context());
            span.context().addParentBaggage(parentBaggage);
        }
        this.spans.add(span);
        return span;
    }

    @Override
    public SpanInScope withSpan(Span span) {
        CurrentTraceContext.Scope scope = this.currentTraceContext.newScope(
                span != null ? span.context() : null);
        return scope::close;
    }

    @Override
    public ScopedSpan startScopedSpan(String name) {
        return new SimpleScopedSpan(this).name(name);
    }

    @Override
    public Span.Builder spanBuilder() {
        return new SimpleSpanBuilder(this);
    }

    @Override
    public TraceContext.Builder traceContextBuilder() {
        return new SimpleTraceContextBuilder();
    }

    @Override
    public CurrentTraceContext currentTraceContext() {
        return this.currentTraceContext;
    }

    @Override
    public SpanCustomizer currentSpanCustomizer() {
        return new SimpleSpanCustomizer(this);
    }

    @Override
    public Span currentSpan() {
        return scopedSpans.get();
    }

    /**
     * Returns the current span as a {@link SimpleSpan}, or null. Used internally
     * by {@link SimpleCurrentTraceContext}.
     */
    SimpleSpan currentSimpleSpan() {
        return scopedSpans.get();
    }

    // ── BaggageManager delegation ───────────────────────────────────────

    @Override
    public Map<String, String> getAllBaggage() {
        return this.simpleBaggageManager.getAllBaggage();
    }

    @Override
    public Map<String, String> getAllBaggage(TraceContext traceContext) {
        return this.simpleBaggageManager.getAllBaggage(traceContext);
    }

    @Override
    public Baggage getBaggage(String name) {
        return this.simpleBaggageManager.getBaggage(name);
    }

    @Override
    public Baggage getBaggage(TraceContext traceContext, String name) {
        return this.simpleBaggageManager.getBaggage(traceContext, name);
    }

    @Override
    public Baggage createBaggage(String name) {
        return this.simpleBaggageManager.createBaggage(name);
    }

    @Override
    public Baggage createBaggage(String name, String value) {
        return this.simpleBaggageManager.createBaggage(name, value);
    }

    @Override
    public BaggageInScope createBaggageInScope(String name, String value) {
        return this.simpleBaggageManager.createBaggageInScope(name, value);
    }

    @Override
    public BaggageInScope createBaggageInScope(TraceContext traceContext, String name,
            String value) {
        return this.simpleBaggageManager.createBaggageInScope(traceContext, name, value);
    }

    // ── Test helpers ────────────────────────────────────────────────────

    /**
     * Asserts that exactly one span was created and returns it. The span must have
     * been started and ended. Use this in tests that expect a single span.
     * @return the only span
     * @throws AssertionError if zero or more than one span exists
     */
    public SimpleSpan onlySpan() {
        if (this.spans.size() != 1) {
            throw new AssertionError(
                    "Expected exactly 1 span, but found " + this.spans.size()
                            + ": " + this.spans);
        }
        SimpleSpan span = this.spans.getFirst();
        if (!span.isStarted()) {
            throw new AssertionError("Span was not started: " + span);
        }
        if (!span.isEnded()) {
            throw new AssertionError("Span was not ended: " + span);
        }
        return span;
    }

    /**
     * Returns the most recently created span. The span must have been started.
     * @return the last span
     * @throws AssertionError if no spans exist
     */
    public SimpleSpan lastSpan() {
        if (this.spans.isEmpty()) {
            throw new AssertionError("No spans were created");
        }
        SimpleSpan span = this.spans.getLast();
        if (!span.isStarted()) {
            throw new AssertionError("Last span was not started: " + span);
        }
        return span;
    }

    /**
     * Returns all spans created by this tracer, in creation order.
     * @return deque of all spans
     */
    public Deque<SimpleSpan> getSpans() {
        return this.spans;
    }

    // ── Private helpers ─────────────────────────────────────────────────

    private SimpleSpan simpleSpan() {
        SimpleSpan span = new SimpleSpan();
        SimpleSpan currentSpan = scopedSpans.get();
        if (currentSpan != null) {
            // Child span — inherit traceId, set parentId
            span.context().setTraceId(currentSpan.context().traceId());
            span.context().setParentId(currentSpan.context().spanId());
            span.context().setSpanId(span.context().generateId());
            span.context().setSampled(currentSpan.context().sampled());
            // Copy baggage from parent
            Map<String, String> parentBaggage =
                    this.simpleBaggageManager.getAllBaggage(currentSpan.context());
            span.context().addParentBaggage(parentBaggage);
        } else {
            // Root span — generate new IDs
            String id = span.context().generateId();
            span.context().setTraceId(id);
            span.context().setSpanId(id);
        }
        return span;
    }

}
```

### `simple-tracing-test/src/test/java/io/simpletracing/test/simple/SimpleTracerTest.java` [NEW]

```java
package io.simpletracing.test.simple;

import java.util.Map;

import io.simpletracing.BaggageInScope;
import io.simpletracing.Link;
import io.simpletracing.ScopedSpan;
import io.simpletracing.Span;
import io.simpletracing.SpanCustomizer;
import io.simpletracing.TraceContext;
import io.simpletracing.Tracer;
import org.junit.jupiter.api.AfterEach;
import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

/**
 * Comprehensive tests for the Simple test implementation. These tests verify that
 * SimpleTracer and its supporting classes correctly implement all API interfaces
 * from Features 1-5.
 */
class SimpleTracerTest {

    private final SimpleTracer tracer = new SimpleTracer();

    @AfterEach
    void cleanup() {
        SimpleTracer.resetCurrentSpan();
    }

    // ── 1. Core Span Lifecycle ──────────────────────────────────────────

    @Test
    void shouldCreateAndEndSpan() {
        Span span = tracer.nextSpan().name("encode").start();
        try {
            span.tag("input.length", "256");
            span.event("encoding-started");
        } finally {
            span.end();
        }
        SimpleSpan finished = tracer.onlySpan();
        assertThat(finished.getName()).isEqualTo("encode");
        assertThat(finished.getTags()).containsEntry("input.length", "256");
        assertThat(finished.getEvents()).hasSize(1);
    }

    @Test
    void shouldRecordErrorOnSpan() {
        RuntimeException error = new RuntimeException("boom");
        Span span = tracer.nextSpan().name("fail").start();
        span.error(error);
        span.end();
        assertThat(tracer.onlySpan().getError()).isSameAs(error);
    }

    @Test
    void shouldGenerateTraceAndSpanIds() {
        Span span = tracer.nextSpan().start();
        span.end();
        SimpleSpan finished = tracer.onlySpan();
        assertThat(finished.getTraceId()).isNotEmpty();
        assertThat(finished.getSpanId()).isNotEmpty();
        assertThat(finished.getTraceId()).isEqualTo(finished.getSpanId());
    }

    // ── 2. Parent-Child ─────────────────────────────────────────────────

    @Test
    void shouldCreateChildSpanFromCurrentScope() {
        Span parent = tracer.nextSpan().name("parent").start();
        try (Tracer.SpanInScope ws = tracer.withSpan(parent)) {
            Span child = tracer.nextSpan().name("child").start();
            child.end();
            SimpleSpan childSpan = tracer.lastSpan();
            assertThat(childSpan.getTraceId()).isEqualTo(parent.context().traceId());
            assertThat(childSpan.getParentId()).isEqualTo(parent.context().spanId());
        } finally {
            parent.end();
        }
    }

    @Test
    void shouldRestorePreviousSpanOnScopeClose() {
        Span outer = tracer.nextSpan().name("outer").start();
        try (Tracer.SpanInScope outerScope = tracer.withSpan(outer)) {
            Span inner = tracer.nextSpan().name("inner").start();
            try (Tracer.SpanInScope innerScope = tracer.withSpan(inner)) {
                assertThat(tracer.currentSpan()).isSameAs(inner);
            }
            assertThat(tracer.currentSpan()).isSameAs(outer);
            inner.end();
        }
        assertThat(tracer.currentSpan()).isNull();
        outer.end();
    }

    // ── 3. ScopedSpan ───────────────────────────────────────────────────

    @Test
    void shouldCreateScopedSpanThatIsInScope() {
        ScopedSpan scoped = tracer.startScopedSpan("encode");
        try {
            scoped.tag("input.length", "256");
            assertThat(tracer.currentSpan()).isNotNull();
        } finally {
            scoped.end();
        }
        assertThat(tracer.currentSpan()).isNull();
        assertThat(tracer.onlySpan().getName()).isEqualTo("encode");
    }

    // ── 4. SpanBuilder ──────────────────────────────────────────────────

    @Test
    void shouldBuildSpanWithAllConfiguration() {
        Span span = tracer.spanBuilder()
            .name("process").kind(Span.Kind.SERVER)
            .tag("http.method", "GET").remoteServiceName("gateway").start();
        span.end();
        SimpleSpan finished = tracer.onlySpan();
        assertThat(finished.getKind()).isEqualTo(Span.Kind.SERVER);
        assertThat(finished.getTags()).containsEntry("http.method", "GET");
        assertThat(finished.getRemoteServiceName()).isEqualTo("gateway");
    }

    // ── 5. Baggage ──────────────────────────────────────────────────────

    @Test
    void shouldCreateAndReadBaggageInScope() {
        Span span = tracer.nextSpan().start();
        try (Tracer.SpanInScope ws = tracer.withSpan(span)) {
            try (BaggageInScope baggage =
                    tracer.createBaggageInScope("user.id", "12345")) {
                assertThat(baggage.get()).isEqualTo("12345");
            }
        } finally {
            span.end();
        }
    }

    // ── 6. Links ────────────────────────────────────────────────────────

    @Test
    void shouldAddLinksViaSpanBuilder() {
        TraceContext linkedCtx = tracer.traceContextBuilder()
            .traceId("linked-trace").spanId("linked-span").sampled(true).build();
        Span span = tracer.spanBuilder().name("batch-process")
            .addLink(new Link(linkedCtx)).start();
        span.end();
        assertThat(tracer.onlySpan().getLinks()).hasSize(1);
    }

    // ── 7. Test helpers ─────────────────────────────────────────────────

    @Test
    void shouldThrowWhenOnlySpanCalledWithNoSpans() {
        assertThatThrownBy(() -> tracer.onlySpan())
            .isInstanceOf(AssertionError.class);
    }

    @Test
    void shouldReturnAllSpans() {
        tracer.nextSpan().name("a").start().end();
        tracer.nextSpan().name("b").start().end();
        assertThat(tracer.getSpans()).hasSize(2);
    }
}
```
