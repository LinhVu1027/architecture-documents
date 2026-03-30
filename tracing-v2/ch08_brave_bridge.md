# Chapter 8: Brave Bridge — Adapt Brave's API to the Tracing Facade

> **What you'll build**: Thirteen adapter classes that bridge Brave's tracing API to the
> Micrometer Tracing facade — enabling applications to use Brave as their tracing backend
> while coding against the vendor-neutral `io.simpletracing` interfaces.

---

## 1. The API Contract

After this chapter, a client can write:

```java
// Configure the Brave tracing backend
brave.Tracing braveTracing = brave.Tracing.newBuilder()
    .localServiceName("my-service")
    .addSpanHandler(new CompositeSpanHandler(
        List.of(),                          // predicates
        List.of(span -> log(span)),         // reporters
        List.of()                           // filters
    ))
    .build();

// Wrap Brave in the Micrometer facade
Tracer tracer = new BraveTracer(
    braveTracing.tracer(),
    new BraveCurrentTraceContext(braveTracing.currentTraceContext()),
    new BraveBaggageManager()
);

// Now use the standard Tracer API — all calls delegate to Brave
Span span = tracer.nextSpan().name("encode").start();
try (Tracer.SpanInScope ws = tracer.withSpan(span)) {
    span.tag("input.length", "256");
    span.event("encoding-started");
    return encoder.encode();
} catch (RuntimeException | Error e) {
    span.error(e);
    throw e;
} finally {
    span.end();  // Brave reports to Zipkin, logs, etc.
}
```

**Why this matters**: Up to now, we've had the API interfaces (Features 1-5) and an in-memory
test implementation (Feature 6). But the whole point of a **facade** is that applications
write code against it once, then swap backends without changing a line. This chapter delivers
the first real backend bridge — proving the facade works.

### The Bridge Architecture

```
                CLIENT CODE
                    │
         uses Tracer, Span, etc.
                    │
    ┌───────────────▼───────────────┐
    │   io.simpletracing (API)       │
    │   Tracer, Span, ScopedSpan,   │
    │   TraceContext, Propagator     │
    └───────────────┬───────────────┘
                    │ implements
    ┌───────────────▼───────────────┐
    │   Brave Bridge (this chapter)  │
    │   BraveTracer → brave.Tracer  │
    │   BraveSpan → brave.Span     │
    │   BraveCurrentTraceContext    │
    │     → brave.CurrentTraceCtx  │
    │   etc.                        │
    └───────────────┬───────────────┘
                    │ delegates to
    ┌───────────────▼───────────────┐
    │   Brave Library (io.zipkin)   │
    │   Sends spans to Zipkin,      │
    │   propagates B3 headers, etc. │
    └───────────────────────────────┘
```

Every bridge class follows the **Adapter pattern**:
1. Wraps exactly one Brave type as a field
2. Implements the corresponding `io.simpletracing` interface
3. Converts types at each method boundary using static `toBrave()`/`fromBrave()` helpers
4. Delegates the actual work to the Brave field

---

## 2. Client Usage & Tests

### Test Setup — Creating a Brave-Backed Tracer

```java
// Collects finished spans from Brave for verification
List<MutableSpan> reportedSpans = new CopyOnWriteArrayList<>();

StrictCurrentTraceContext braveCtc = StrictCurrentTraceContext.create();
Tracing braveTracing = Tracing.newBuilder()
    .localServiceName("test-service")
    .currentTraceContext(braveCtc)
    .addSpanHandler(new SpanHandler() {
        @Override
        public boolean end(brave.propagation.TraceContext context,
                           MutableSpan span, Cause cause) {
            if (cause == Cause.FINISHED) {
                reportedSpans.add(span);
            }
            return true;
        }
    })
    .build();

Tracer tracer = new BraveTracer(
    braveTracing.tracer(),
    new BraveCurrentTraceContext(braveTracing.currentTraceContext()),
    new BraveBaggageManager());
```

### Core Tests — Using the Facade Over Brave

```java
@Test
void shouldCreateAndEndSpan() {
    Span span = tracer.nextSpan().name("test-op").start();
    try {
        span.tag("key", "value");
        span.event("something-happened");
    } finally {
        span.end();
    }
    assertThat(reportedSpans).hasSize(1);
    assertThat(reportedSpans.get(0).name()).isEqualTo("test-op");
    assertThat(reportedSpans.get(0).tags()).containsEntry("key", "value");
}

@Test
void shouldCreateChildSpanFromScope() {
    Span parent = tracer.nextSpan().name("parent").start();
    try (Tracer.SpanInScope ws = tracer.withSpan(parent)) {
        Span child = tracer.nextSpan().name("child").start();
        assertThat(child.context().traceId())
                .isEqualTo(parent.context().traceId());
        child.end();
    } finally {
        parent.end();
    }
    assertThat(reportedSpans).hasSize(2);
}

@Test
void shouldInjectAndExtractTraceContext() {
    Propagator propagator = new BravePropagator(braveTracing);

    Span sender = tracer.nextSpan().name("send").start();
    Map<String, String> headers = new HashMap<>();
    propagator.inject(sender.context(), headers, Map::put);

    Span.Builder extracted = propagator.extract(headers, Map::get);
    Span receiver = extracted.name("receive").kind(Span.Kind.SERVER).start();

    assertThat(receiver.context().traceId())
            .isEqualTo(sender.context().traceId());
    receiver.end();
    sender.end();
}
```

---

## 3. Implementing the Call Chain — Building Each Layer Top-Down

### Layer 1: The Foundational Type — BraveTraceContext

Every bridge class needs to convert between Micrometer's string-based `TraceContext` and
Brave's long-based `brave.propagation.TraceContext`. This adapter is used everywhere.

```java
public class BraveTraceContext implements TraceContext {
    final brave.propagation.TraceContext traceContext;

    public BraveTraceContext(brave.propagation.TraceContext traceContext) {
        this.traceContext = traceContext;
    }

    // Convert Micrometer → Brave (for outbound calls)
    public static brave.propagation.TraceContext toBrave(TraceContext traceContext) {
        if (traceContext == null) return null;
        return ((BraveTraceContext) traceContext).traceContext;
    }

    // Convert Brave → Micrometer (for return values)
    public static TraceContext fromBrave(brave.propagation.TraceContext traceContext) {
        if (traceContext == null) return null;
        return new BraveTraceContext(traceContext);
    }

    @Override public String traceId()  { return traceContext.traceIdString(); }
    @Override public String parentId() { return traceContext.parentIdString(); }
    @Override public String spanId()   { return traceContext.spanIdString(); }
    @Override public Boolean sampled()  { return traceContext.sampled(); }
}
```

**Key insight**: Brave stores IDs as `long` internally (for performance), but exposes them
as hex strings via `traceIdString()`, `spanIdString()`, etc. Our facade uses strings
throughout, so the conversion is natural.

### Layer 2: The Scope Bridge — BraveCurrentTraceContext

```java
public class BraveCurrentTraceContext implements CurrentTraceContext {
    final brave.propagation.CurrentTraceContext delegate;

    @Override
    public TraceContext context() {
        return BraveTraceContext.fromBrave(delegate.get());
    }

    @Override
    public Scope newScope(TraceContext context) {
        var braveScope = delegate.newScope(BraveTraceContext.toBrave(context));
        return new BraveScope(braveScope);  // Wrap the scope too
    }
}
```

Notice the pattern: unwrap at the boundary → delegate → wrap the result.

### Layer 3: The Span Bridge — BraveSpan

```java
class BraveSpan implements Span {
    final brave.Span delegate;

    @Override public Span event(String value) {
        delegate.annotate(value);  // Brave calls these "annotations"
        return this;
    }

    @Override public Span error(Throwable throwable) {
        String message = throwable.getMessage();
        if (message == null) message = throwable.getClass().getSimpleName();
        delegate.tag("error", message);   // Tag for search
        delegate.error(throwable);         // Error for display
        return this;
    }

    @Override public void end() {
        delegate.finish();  // Brave calls it "finish"
    }
}
```

**Naming differences**: Brave uses different method names for some operations:
- Micrometer `event()` → Brave `annotate()`
- Micrometer `end()` → Brave `finish()`
- Micrometer `error()` → Brave `tag("error", msg)` + `error(throwable)`

### Layer 4: The Builder Bridge — BraveSpanBuilder

Unlike Brave (which configures spans on the live object), Micrometer's `Span.Builder`
accumulates configuration and applies it all at `start()`:

```java
class BraveSpanBuilder implements Span.Builder {
    private final brave.Tracer tracer;
    private TraceContextOrSamplingFlags parentContext;
    private String name;
    private Map<String, String> tags = new HashMap<>();
    // ... other fields accumulated

    @Override
    public Span start() {
        brave.Span span = (parentContext != null)
            ? tracer.nextSpan(parentContext)
            : tracer.nextSpan();

        if (name != null) span.name(name);
        if (kind != null) span.kind(kind);
        tags.forEach(span::tag);
        events.forEach(span::annotate);

        span.start();
        return BraveSpan.fromBrave(span);
    }
}
```

### Layer 5: The Central Orchestrator — BraveTracer

```java
public class BraveTracer implements Tracer {
    private final brave.Tracer tracer;
    private final CurrentTraceContext currentTraceContext;
    private final BaggageManager braveBaggageManager;

    @Override
    public Span nextSpan() {
        return BraveSpan.fromBrave(tracer.nextSpan());
    }

    @Override
    public SpanInScope withSpan(Span span) {
        var braveScope = tracer.withSpanInScope(BraveSpan.toBrave(span));
        return new BraveSpanInScope(braveScope);
    }

    @Override
    public ScopedSpan startScopedSpan(String name) {
        return new BraveScopedSpan(tracer.startScopedSpan(name));
    }

    // Baggage operations all delegate to braveBaggageManager
    @Override
    public Map<String, String> getAllBaggage() {
        return braveBaggageManager.getAllBaggage();
    }
    // ... etc.
}
```

### Layer 6: The Export Pipeline Bridge — CompositeSpanHandler

This is the integration point where Brave's span reporting mechanism meets our export
pipeline (from Feature 2):

```java
public class CompositeSpanHandler extends brave.handler.SpanHandler {
    private final List<SpanExportingPredicate> predicates;
    private final List<SpanReporter> reporters;
    private final List<SpanFilter> spanFilters;

    @Override
    public boolean end(TraceContext context, MutableSpan span, Cause cause) {
        if (cause != Cause.FINISHED) return true;

        // Gate: check predicates
        FinishedSpan fs = BraveFinishedSpan.fromBrave(span);
        for (var pred : predicates) {
            if (!pred.isExportable(fs)) return true;  // Drop span
        }

        // Transform: apply filters
        for (var filter : spanFilters) {
            fs = filter.map(fs);
        }

        // Report: send to all reporters
        for (var reporter : reporters) {
            reporter.report(fs);
        }
        return true;
    }
}
```

**How this connects**: When a Brave span finishes, Brave calls `SpanHandler.end()`.
Our `CompositeSpanHandler` intercepts this call, wraps the Brave `MutableSpan` in a
`BraveFinishedSpan` (which implements `FinishedSpan`), and feeds it through the
filter → predicate → reporter pipeline that clients configured.

### Layer 7: The Propagation Bridge — BravePropagator

```java
public class BravePropagator implements Propagator {
    private final Tracing tracing;

    @Override
    public <C> void inject(TraceContext context, C carrier, Setter<C> setter) {
        tracing.propagation()
            .injector(setter::set)
            .inject(BraveTraceContext.toBrave(context), carrier);
    }

    @Override
    public <C> Span.Builder extract(C carrier, Getter<C> getter) {
        TraceContextOrSamplingFlags extracted = tracing.propagation()
            .extractor(getter::get)
            .extract(carrier);
        return BraveSpanBuilder.toBuilder(tracing.tracer(), extracted);
    }
}
```

**Elegant mapping**: Micrometer's `Setter<C>` has `set(carrier, key, value)` — exactly
matching Brave's `Propagation.Setter` contract. The `setter::set` method reference does
the bridging without any wrapper class.

---

## 4. Try It Yourself

1. **Add a `BraveFinishedSpan.setTags()` that clears existing tags**: The current
   simplified version doesn't fully clear tags in `setTags()`. Implement it by iterating
   the existing tags map, setting each to null, then adding the new ones.

2. **Add link support to BraveSpanBuilder**: Links are stored as specially-named tags
   in Brave (e.g., `link.0.traceId`, `link.0.spanId`). Implement `addLink(Link)` that
   encodes the link's trace context into these tag keys.

3. **Add `remoteIpAndPort` to BraveSpan**: The real framework supports
   `Span.remoteIpAndPort(String ip, int port)`. Add this method to the `Span` interface
   and implement it in `BraveSpan` by delegating to `delegate.remoteIpAndPort(ip, port)`.

---

## 5. Why This Works

### The Adapter Pattern
The bridge works because every Brave type has a 1:1 correspondence with a Micrometer type.
Each bridge class is a **thin wrapper** — it holds a Brave object and translates method
calls at the boundary. The overhead is minimal (one object allocation per conversion) and
the semantics are preserved exactly.

### Type Safety at the Boundary
The static `toBrave()`/`fromBrave()` methods on each bridge class create a consistent
conversion protocol. Any bridge class that needs to convert types (and they all do) uses
these methods rather than casting directly. This makes the conversion explicit and
discoverable.

### The Facade Payoff
With this bridge in place, application code that uses `Tracer` gets Brave's full
capabilities (B3 propagation, Zipkin reporting, Brave instrumentation) without importing
a single Brave class. Swapping to a different backend (OpenTelemetry, for instance) would
mean writing a new bridge — the application code doesn't change.

---

## 6. What We Enhanced

| Feature | What was added/changed |
|---------|----------------------|
| Feature 1 (Tracer, Span) | First real implementation of all interfaces — `BraveTracer`, `BraveSpan`, `BraveScopedSpan` |
| Feature 1 (TraceContext) | `BraveTraceContext` bridges Brave's long-based IDs to string-based Micrometer IDs |
| Feature 1 (CurrentTraceContext) | `BraveCurrentTraceContext` delegates scope management to Brave's ThreadLocal |
| Feature 1 (SpanCustomizer) | `BraveSpanCustomizer` wraps Brave's customizer |
| Feature 2 (Export Pipeline) | `CompositeSpanHandler` plugs Micrometer's filter/predicate/reporter pipeline into Brave's `SpanHandler` |
| Feature 2 (FinishedSpan) | `BraveFinishedSpan` wraps `MutableSpan` with microsecond → Instant conversion |
| Feature 3 (Baggage) | `BraveBaggageManager` and `BraveBaggageInScope` wrap Brave's `BaggageField` |
| Feature 4 (Propagation) | `BravePropagator` delegates inject/extract to Brave's propagation system |

---

## 7. Connection to Real Framework

| Simplified Class | Real Class | Key Differences |
|-----------------|------------|-----------------|
| `BraveTracer` | `io.micrometer.tracing.brave.bridge.BraveTracer` | Real version has `getBaggageFields()`, supports observation handlers |
| `BraveSpan` | `io.micrometer.tracing.brave.bridge.BraveSpan` | Real version has `event(value, time, timeUnit)`, `end(time, timeUnit)`, `remoteIpAndPort()` |
| `BraveSpanBuilder` | `io.micrometer.tracing.brave.bridge.BraveSpanBuilder` | Real version has `startTimestamp()`, link encoding via `LinkUtils`, `remoteIpAndPort()` |
| `BraveScopedSpan` | `io.micrometer.tracing.brave.bridge.BraveScopedSpan` | Same structure |
| `BraveTraceContext` | `io.micrometer.tracing.brave.bridge.BraveTraceContext` | Same structure |
| `BraveTraceContextBuilder` | — | Real framework uses a different builder approach |
| `BraveCurrentTraceContext` | `io.micrometer.tracing.brave.bridge.BraveCurrentTraceContext` | Real version has `wrap(Callable)`, `wrap(Runnable)`, `wrap(Executor)`, `wrap(ExecutorService)` |
| `BraveSpanCustomizer` | `io.micrometer.tracing.brave.bridge.BraveSpanCustomizer` | Same structure |
| `BraveFinishedSpan` | `io.micrometer.tracing.brave.bridge.BraveFinishedSpan` | Real version has `getLinks()` via `LinkUtils`, `getLocalIp/Port`, `getRemoteIp/Port` |
| `BravePropagator` | `io.micrometer.tracing.brave.bridge.BravePropagator` | Real version has baggage field synchronization after extract |
| `BraveBaggageManager` | `io.micrometer.tracing.brave.bridge.BraveBaggageManager` | Real version has tag fields, remote fields, field caching |
| `BraveBaggageInScope` | `io.micrometer.tracing.brave.bridge.BraveBaggageInScope` | Real version has tag-span-if-on-tag-list, trace context change detection |
| `CompositeSpanHandler` | `io.micrometer.tracing.brave.bridge.CompositeSpanHandler` | Same structure — predicate → filter → reporter pipeline |

---

## 8. Complete Code

### `[NEW]` `simple-tracing-bridge-brave/src/main/java/io/simpletracing/brave/bridge/BraveTraceContext.java`

```java
package io.simpletracing.brave.bridge;

import io.simpletracing.TraceContext;

public class BraveTraceContext implements TraceContext {

    final brave.propagation.TraceContext traceContext;

    public BraveTraceContext(brave.propagation.TraceContext traceContext) {
        this.traceContext = traceContext;
    }

    public static brave.propagation.TraceContext toBrave(TraceContext traceContext) {
        if (traceContext == null) {
            return null;
        }
        return ((BraveTraceContext) traceContext).traceContext;
    }

    public static TraceContext fromBrave(brave.propagation.TraceContext traceContext) {
        if (traceContext == null) {
            return null;
        }
        return new BraveTraceContext(traceContext);
    }

    @Override
    public String traceId() {
        return this.traceContext.traceIdString();
    }

    @Override
    public String parentId() {
        return this.traceContext.parentIdString();
    }

    @Override
    public String spanId() {
        return this.traceContext.spanIdString();
    }

    @Override
    public Boolean sampled() {
        return this.traceContext.sampled();
    }

    @Override
    public String toString() {
        return "BraveTraceContext{" + this.traceContext + "}";
    }

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (o == null || getClass() != o.getClass()) return false;
        BraveTraceContext that = (BraveTraceContext) o;
        return this.traceContext.equals(that.traceContext);
    }

    @Override
    public int hashCode() {
        return this.traceContext.hashCode();
    }
}
```

### `[NEW]` `simple-tracing-bridge-brave/src/main/java/io/simpletracing/brave/bridge/BraveTraceContextBuilder.java`

```java
package io.simpletracing.brave.bridge;

import io.simpletracing.TraceContext;

class BraveTraceContextBuilder implements TraceContext.Builder {

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
        brave.propagation.TraceContext.Builder builder = brave.propagation.TraceContext.newBuilder();
        if (this.traceId != null) {
            if (this.traceId.length() > 16) {
                builder.traceIdHigh(hexToLong(this.traceId.substring(0, this.traceId.length() - 16)));
                builder.traceId(hexToLong(this.traceId.substring(this.traceId.length() - 16)));
            } else {
                builder.traceId(hexToLong(this.traceId));
            }
        }
        if (this.parentId != null) {
            builder.parentId(hexToLong(this.parentId));
        }
        if (this.spanId != null) {
            builder.spanId(hexToLong(this.spanId));
        }
        if (this.sampled != null) {
            builder.sampled(this.sampled);
        }
        return new BraveTraceContext(builder.build());
    }

    private static long hexToLong(String hex) {
        return Long.parseUnsignedLong(hex, 16);
    }
}
```

### `[NEW]` `simple-tracing-bridge-brave/src/main/java/io/simpletracing/brave/bridge/BraveCurrentTraceContext.java`

```java
package io.simpletracing.brave.bridge;

import io.simpletracing.CurrentTraceContext;
import io.simpletracing.TraceContext;

public class BraveCurrentTraceContext implements CurrentTraceContext {

    final brave.propagation.CurrentTraceContext delegate;

    public BraveCurrentTraceContext(brave.propagation.CurrentTraceContext delegate) {
        this.delegate = delegate;
    }

    public static brave.propagation.CurrentTraceContext toBrave(CurrentTraceContext context) {
        if (context == null) {
            return null;
        }
        return ((BraveCurrentTraceContext) context).delegate;
    }

    public static CurrentTraceContext fromBrave(brave.propagation.CurrentTraceContext context) {
        if (context == null) {
            return null;
        }
        return new BraveCurrentTraceContext(context);
    }

    @Override
    public TraceContext context() {
        brave.propagation.TraceContext braveContext = this.delegate.get();
        return BraveTraceContext.fromBrave(braveContext);
    }

    @Override
    public Scope newScope(TraceContext context) {
        brave.propagation.CurrentTraceContext.Scope braveScope =
                this.delegate.newScope(BraveTraceContext.toBrave(context));
        return new BraveScope(braveScope);
    }

    @Override
    public Scope maybeScope(TraceContext context) {
        brave.propagation.CurrentTraceContext.Scope braveScope =
                this.delegate.maybeScope(BraveTraceContext.toBrave(context));
        return new BraveScope(braveScope);
    }

    private static class BraveScope implements Scope {
        private final brave.propagation.CurrentTraceContext.Scope delegate;

        BraveScope(brave.propagation.CurrentTraceContext.Scope delegate) {
            this.delegate = delegate;
        }

        @Override
        public void close() {
            this.delegate.close();
        }
    }
}
```

### `[NEW]` `simple-tracing-bridge-brave/src/main/java/io/simpletracing/brave/bridge/BraveSpan.java`

```java
package io.simpletracing.brave.bridge;

import io.simpletracing.Span;
import io.simpletracing.TraceContext;

class BraveSpan implements Span {

    final brave.Span delegate;

    BraveSpan(brave.Span delegate) {
        this.delegate = delegate;
    }

    static brave.Span toBrave(Span span) {
        if (span == null) return null;
        return ((BraveSpan) span).delegate;
    }

    static Span fromBrave(brave.Span span) {
        if (span == null) return null;
        return new BraveSpan(span);
    }

    @Override public boolean isNoop() { return this.delegate.isNoop(); }
    @Override public TraceContext context() { return new BraveTraceContext(this.delegate.context()); }
    @Override public Span start() { this.delegate.start(); return this; }
    @Override public Span name(String name) { this.delegate.name(name); return this; }
    @Override public Span event(String value) { this.delegate.annotate(value); return this; }
    @Override public Span tag(String key, String value) { this.delegate.tag(key, value); return this; }

    @Override
    public Span error(Throwable throwable) {
        String message = throwable.getMessage();
        if (message == null) message = throwable.getClass().getSimpleName();
        this.delegate.tag("error", message);
        this.delegate.error(throwable);
        return this;
    }

    @Override public void end() { this.delegate.finish(); }
    @Override public void abandon() { this.delegate.abandon(); }
    @Override public Span remoteServiceName(String remoteServiceName) {
        this.delegate.remoteServiceName(remoteServiceName); return this;
    }

    @Override public String toString() { return "BraveSpan{" + this.delegate + "}"; }
    @Override public boolean equals(Object o) {
        if (this == o) return true;
        if (o == null || getClass() != o.getClass()) return false;
        return this.delegate.equals(((BraveSpan) o).delegate);
    }
    @Override public int hashCode() { return this.delegate.hashCode(); }
}
```

### `[NEW]` `simple-tracing-bridge-brave/src/main/java/io/simpletracing/brave/bridge/BraveSpanBuilder.java`

```java
package io.simpletracing.brave.bridge;

import java.util.ArrayList;
import java.util.HashMap;
import java.util.List;
import java.util.Map;

import brave.propagation.TraceContextOrSamplingFlags;

import io.simpletracing.Link;
import io.simpletracing.Span;
import io.simpletracing.TraceContext;

class BraveSpanBuilder implements Span.Builder {

    private final brave.Tracer tracer;
    private TraceContextOrSamplingFlags parentContext;
    private String name;
    private final List<String> events = new ArrayList<>();
    private final Map<String, String> tags = new HashMap<>();
    private Throwable error;
    private brave.Span.Kind kind;
    private String remoteServiceName;

    BraveSpanBuilder(brave.Tracer tracer) { this.tracer = tracer; }

    BraveSpanBuilder(brave.Tracer tracer, TraceContextOrSamplingFlags parentContext) {
        this.tracer = tracer;
        this.parentContext = parentContext;
    }

    static Span.Builder toBuilder(brave.Tracer tracer, TraceContextOrSamplingFlags context) {
        return new BraveSpanBuilder(tracer, context);
    }

    @Override public Span.Builder setParent(TraceContext context) {
        this.parentContext = TraceContextOrSamplingFlags.create(BraveTraceContext.toBrave(context));
        return this;
    }
    @Override public Span.Builder setNoParent() { this.parentContext = null; return this; }
    @Override public Span.Builder name(String name) { this.name = name; return this; }
    @Override public Span.Builder event(String value) { this.events.add(value); return this; }
    @Override public Span.Builder tag(String key, String value) { this.tags.put(key, value); return this; }
    @Override public Span.Builder error(Throwable throwable) { this.error = throwable; return this; }
    @Override public Span.Builder kind(Span.Kind spanKind) {
        this.kind = brave.Span.Kind.valueOf(spanKind.toString()); return this;
    }
    @Override public Span.Builder remoteServiceName(String remoteServiceName) {
        this.remoteServiceName = remoteServiceName; return this;
    }
    @Override public Span.Builder addLink(Link link) { return this; }

    @Override
    public Span start() {
        brave.Span span = (parentContext != null) ? tracer.nextSpan(parentContext) : tracer.nextSpan();
        if (name != null) span.name(name);
        if (kind != null) span.kind(kind);
        if (remoteServiceName != null) span.remoteServiceName(remoteServiceName);
        tags.forEach(span::tag);
        events.forEach(span::annotate);
        if (error != null) span.error(error);
        span.start();
        return BraveSpan.fromBrave(span);
    }
}
```

### `[NEW]` `simple-tracing-bridge-brave/src/main/java/io/simpletracing/brave/bridge/BraveScopedSpan.java`

```java
package io.simpletracing.brave.bridge;

import io.simpletracing.ScopedSpan;
import io.simpletracing.TraceContext;

class BraveScopedSpan implements ScopedSpan {

    final brave.ScopedSpan delegate;

    BraveScopedSpan(brave.ScopedSpan delegate) { this.delegate = delegate; }

    @Override public boolean isNoop() { return this.delegate.isNoop(); }
    @Override public TraceContext context() { return new BraveTraceContext(this.delegate.context()); }
    @Override public ScopedSpan name(String name) { this.delegate.name(name); return this; }
    @Override public ScopedSpan tag(String key, String value) { this.delegate.tag(key, value); return this; }
    @Override public ScopedSpan event(String value) { this.delegate.annotate(value); return this; }
    @Override public ScopedSpan error(Throwable throwable) { this.delegate.error(throwable); return this; }
    @Override public void end() { this.delegate.finish(); }

    @Override public boolean equals(Object o) {
        if (this == o) return true;
        if (o == null || getClass() != o.getClass()) return false;
        return this.delegate.equals(((BraveScopedSpan) o).delegate);
    }
    @Override public int hashCode() { return this.delegate.hashCode(); }
}
```

### `[NEW]` `simple-tracing-bridge-brave/src/main/java/io/simpletracing/brave/bridge/BraveSpanCustomizer.java`

```java
package io.simpletracing.brave.bridge;

import io.simpletracing.SpanCustomizer;

class BraveSpanCustomizer implements SpanCustomizer {

    private final brave.SpanCustomizer delegate;

    BraveSpanCustomizer(brave.SpanCustomizer delegate) { this.delegate = delegate; }

    static brave.SpanCustomizer toBrave(SpanCustomizer sc) {
        return sc == null ? null : ((BraveSpanCustomizer) sc).delegate;
    }
    static SpanCustomizer fromBrave(brave.SpanCustomizer sc) {
        return sc == null ? null : new BraveSpanCustomizer(sc);
    }

    @Override public SpanCustomizer name(String name) {
        return new BraveSpanCustomizer(this.delegate.name(name));
    }
    @Override public SpanCustomizer tag(String key, String value) {
        return new BraveSpanCustomizer(this.delegate.tag(key, value));
    }
    @Override public SpanCustomizer event(String value) {
        return new BraveSpanCustomizer(this.delegate.annotate(value));
    }
}
```

### `[NEW]` `simple-tracing-bridge-brave/src/main/java/io/simpletracing/brave/bridge/BraveFinishedSpan.java`

```java
package io.simpletracing.brave.bridge;

import java.time.Instant;
import java.time.temporal.ChronoUnit;
import java.util.Collection;
import java.util.Collections;
import java.util.List;
import java.util.Map;

import brave.handler.MutableSpan;

import io.simpletracing.Link;
import io.simpletracing.Span;
import io.simpletracing.exporter.FinishedSpan;

class BraveFinishedSpan implements FinishedSpan {

    private final MutableSpan mutableSpan;

    BraveFinishedSpan(MutableSpan mutableSpan) { this.mutableSpan = mutableSpan; }

    static FinishedSpan fromBrave(MutableSpan mutableSpan) { return new BraveFinishedSpan(mutableSpan); }
    static MutableSpan toBrave(FinishedSpan fs) { return ((BraveFinishedSpan) fs).mutableSpan; }

    @Override public String getName() { return mutableSpan.name(); }
    @Override public FinishedSpan setName(String name) { mutableSpan.name(name); return this; }
    @Override public Instant getStartTimestamp() { return microsToInstant(mutableSpan.startTimestamp()); }
    @Override public Instant getEndTimestamp() { return microsToInstant(mutableSpan.finishTimestamp()); }
    @Override public Map<String, String> getTags() { return mutableSpan.tags(); }

    @Override
    public FinishedSpan setTags(Map<String, String> tags) {
        Map<String, String> existing = mutableSpan.tags();
        for (String key : List.copyOf(existing.keySet())) { mutableSpan.tag(key, null); }
        tags.forEach(mutableSpan::tag);
        return this;
    }

    @Override public Collection<Map.Entry<Long, String>> getEvents() { return mutableSpan.annotations(); }

    @Override
    public FinishedSpan setEvents(Collection<Map.Entry<Long, String>> events) {
        events.forEach(e -> mutableSpan.annotate(e.getKey(), e.getValue()));
        return this;
    }

    @Override public String getSpanId() { return mutableSpan.id(); }
    @Override public String getParentId() { return mutableSpan.parentId(); }
    @Override public String getTraceId() { return mutableSpan.traceId(); }
    @Override public Throwable getError() { return mutableSpan.error(); }
    @Override public FinishedSpan setError(Throwable error) { mutableSpan.error(error); return this; }

    @Override
    public Span.Kind getKind() {
        brave.Span.Kind braveKind = mutableSpan.kind();
        return braveKind == null ? null : Span.Kind.valueOf(braveKind.name());
    }

    @Override public List<Link> getLinks() { return Collections.emptyList(); }
    @Override public String getRemoteServiceName() { return mutableSpan.remoteServiceName(); }
    @Override public FinishedSpan setRemoteServiceName(String name) { mutableSpan.remoteServiceName(name); return this; }

    private static Instant microsToInstant(long micros) {
        return Instant.EPOCH.plus(micros, ChronoUnit.MICROS);
    }
}
```

### `[NEW]` `simple-tracing-bridge-brave/src/main/java/io/simpletracing/brave/bridge/BravePropagator.java`

```java
package io.simpletracing.brave.bridge;

import java.util.List;

import brave.Tracing;
import brave.propagation.TraceContextOrSamplingFlags;

import io.simpletracing.Span;
import io.simpletracing.TraceContext;
import io.simpletracing.propagation.Propagator;

public class BravePropagator implements Propagator {

    private final Tracing tracing;

    public BravePropagator(Tracing tracing) { this.tracing = tracing; }

    @Override public List<String> fields() { return tracing.propagation().keys(); }

    @Override
    public <C> void inject(TraceContext context, C carrier, Setter<C> setter) {
        tracing.propagation().injector(setter::set)
                .inject(BraveTraceContext.toBrave(context), carrier);
    }

    @Override
    public <C> Span.Builder extract(C carrier, Getter<C> getter) {
        TraceContextOrSamplingFlags extracted = tracing.propagation()
                .extractor(getter::get).extract(carrier);
        return BraveSpanBuilder.toBuilder(tracing.tracer(), extracted);
    }
}
```

### `[NEW]` `simple-tracing-bridge-brave/src/main/java/io/simpletracing/brave/bridge/BraveBaggageInScope.java`

```java
package io.simpletracing.brave.bridge;

import brave.baggage.BaggageField;

import io.simpletracing.Baggage;
import io.simpletracing.BaggageInScope;
import io.simpletracing.TraceContext;

class BraveBaggageInScope implements Baggage, BaggageInScope {

    private final BaggageField delegate;
    private final String previousBaggage;
    private brave.propagation.TraceContext traceContext;

    BraveBaggageInScope(BaggageField delegate, brave.propagation.TraceContext traceContext) {
        this.delegate = delegate;
        this.traceContext = traceContext;
        this.previousBaggage = (traceContext != null) ? delegate.getValue(traceContext) : delegate.getValue();
    }

    @Override public String name() { return delegate.name(); }
    @Override public String get() {
        return (traceContext != null) ? delegate.getValue(traceContext) : delegate.getValue();
    }
    @Override public String get(TraceContext traceContext) {
        return delegate.getValue(BraveTraceContext.toBrave(traceContext));
    }
    @Override public Baggage set(String value) {
        if (traceContext != null) delegate.updateValue(traceContext, value);
        else delegate.updateValue(value);
        return this;
    }
    @Override public Baggage set(TraceContext traceContext, String value) {
        this.traceContext = BraveTraceContext.toBrave(traceContext);
        delegate.updateValue(this.traceContext, value);
        return this;
    }
    @Override public BaggageInScope makeCurrent() { return this; }
    @Override public BaggageInScope makeCurrent(String value) { set(value); return makeCurrent(); }
    @Override public BaggageInScope makeCurrent(TraceContext tc, String value) { set(tc, value); return makeCurrent(); }
    @Override public void close() {
        if (traceContext != null) delegate.updateValue(traceContext, previousBaggage);
    }
}
```

### `[NEW]` `simple-tracing-bridge-brave/src/main/java/io/simpletracing/brave/bridge/BraveBaggageManager.java`

```java
package io.simpletracing.brave.bridge;

import java.util.Map;

import brave.baggage.BaggageField;

import io.simpletracing.Baggage;
import io.simpletracing.BaggageInScope;
import io.simpletracing.BaggageManager;
import io.simpletracing.TraceContext;

public class BraveBaggageManager implements BaggageManager {

    private BraveTracer tracer;

    public BraveBaggageManager() {}

    void setTracer(BraveTracer tracer) { this.tracer = tracer; }

    @Override public Map<String, String> getAllBaggage() { return BaggageField.getAllValues(); }

    @Override
    public Map<String, String> getAllBaggage(TraceContext traceContext) {
        if (traceContext == null) return getAllBaggage();
        return BaggageField.getAllValues(BraveTraceContext.toBrave(traceContext));
    }

    @Override public Baggage getBaggage(String name) { return createBaggage(name); }

    @Override
    public Baggage getBaggage(TraceContext traceContext, String name) {
        BaggageField field = BaggageField.getByName(BraveTraceContext.toBrave(traceContext), name);
        if (field == null) return Baggage.NOOP;
        return new BraveBaggageInScope(field, BraveTraceContext.toBrave(traceContext));
    }

    @Override
    public Baggage createBaggage(String name) {
        return new BraveBaggageInScope(BaggageField.create(name), currentBraveContext());
    }

    @Override
    public Baggage createBaggage(String name, String value) {
        Baggage baggage = createBaggage(name);
        baggage.set(value);
        return baggage;
    }

    @Override public BaggageInScope createBaggageInScope(String name, String value) {
        return createBaggage(name).makeCurrent(value);
    }

    @Override
    public BaggageInScope createBaggageInScope(TraceContext tc, String name, String value) {
        BaggageField field = BaggageField.create(name);
        BraveBaggageInScope baggage = new BraveBaggageInScope(field, BraveTraceContext.toBrave(tc));
        return baggage.makeCurrent(tc, value);
    }

    private brave.propagation.TraceContext currentBraveContext() {
        if (tracer != null) {
            brave.Span current = BraveSpan.toBrave(tracer.currentSpan());
            if (current != null) return current.context();
        }
        return null;
    }
}
```

### `[NEW]` `simple-tracing-bridge-brave/src/main/java/io/simpletracing/brave/bridge/CompositeSpanHandler.java`

```java
package io.simpletracing.brave.bridge;

import java.util.Collections;
import java.util.List;

import brave.handler.MutableSpan;
import brave.handler.SpanHandler;
import brave.propagation.TraceContext;

import io.simpletracing.exporter.FinishedSpan;
import io.simpletracing.exporter.SpanExportingPredicate;
import io.simpletracing.exporter.SpanFilter;
import io.simpletracing.exporter.SpanReporter;

public class CompositeSpanHandler extends SpanHandler {

    private final List<SpanExportingPredicate> predicates;
    private final List<SpanReporter> reporters;
    private final List<SpanFilter> spanFilters;

    public CompositeSpanHandler(List<SpanExportingPredicate> predicates,
                                List<SpanReporter> reporters,
                                List<SpanFilter> spanFilters) {
        this.predicates = predicates != null ? predicates : Collections.emptyList();
        this.reporters = reporters != null ? reporters : Collections.emptyList();
        this.spanFilters = spanFilters != null ? spanFilters : Collections.emptyList();
    }

    public CompositeSpanHandler(List<SpanReporter> reporters) {
        this(Collections.emptyList(), reporters, Collections.emptyList());
    }

    @Override
    public boolean end(TraceContext context, MutableSpan span, Cause cause) {
        if (cause != Cause.FINISHED) return true;
        if (!shouldProcess(span)) return true;
        if (!super.end(context, span, cause)) return false;

        FinishedSpan finishedSpan = BraveFinishedSpan.fromBrave(span);
        for (SpanFilter filter : spanFilters) { finishedSpan = filter.map(finishedSpan); }
        for (SpanReporter reporter : reporters) { reporter.report(finishedSpan); }
        return true;
    }

    private boolean shouldProcess(MutableSpan span) {
        FinishedSpan fs = BraveFinishedSpan.fromBrave(span);
        for (SpanExportingPredicate p : predicates) { if (!p.isExportable(fs)) return false; }
        return true;
    }
}
```

### `[NEW]` `simple-tracing-bridge-brave/src/main/java/io/simpletracing/brave/bridge/BraveTracer.java`

```java
package io.simpletracing.brave.bridge;

import java.util.Map;

import brave.propagation.TraceContextOrSamplingFlags;

import io.simpletracing.Baggage;
import io.simpletracing.BaggageInScope;
import io.simpletracing.BaggageManager;
import io.simpletracing.CurrentTraceContext;
import io.simpletracing.ScopedSpan;
import io.simpletracing.Span;
import io.simpletracing.SpanCustomizer;
import io.simpletracing.TraceContext;
import io.simpletracing.Tracer;

public class BraveTracer implements Tracer {

    private final brave.Tracer tracer;
    private final CurrentTraceContext currentTraceContext;
    private final BaggageManager braveBaggageManager;

    public BraveTracer(brave.Tracer tracer,
                       CurrentTraceContext currentTraceContext,
                       BaggageManager braveBaggageManager) {
        this.tracer = tracer;
        this.currentTraceContext = currentTraceContext;
        this.braveBaggageManager = braveBaggageManager;
        if (braveBaggageManager instanceof BraveBaggageManager bbm) { bbm.setTracer(this); }
    }

    public BraveTracer(brave.Tracer tracer, CurrentTraceContext currentTraceContext) {
        this(tracer, currentTraceContext, BaggageManager.NOOP);
    }

    @Override public Span nextSpan() { return BraveSpan.fromBrave(tracer.nextSpan()); }

    @Override
    public Span nextSpan(Span parent) {
        if (parent == null) return nextSpan();
        var parentFlags = TraceContextOrSamplingFlags.create(BraveTraceContext.toBrave(parent.context()));
        return BraveSpan.fromBrave(tracer.nextSpan(parentFlags));
    }

    @Override public ScopedSpan startScopedSpan(String name) {
        return new BraveScopedSpan(tracer.startScopedSpan(name));
    }
    @Override public Span.Builder spanBuilder() { return new BraveSpanBuilder(tracer); }

    @Override
    public SpanInScope withSpan(Span span) {
        brave.Span braveSpan = (span != null) ? BraveSpan.toBrave(span) : null;
        return new BraveSpanInScope(tracer.withSpanInScope(braveSpan));
    }

    @Override public CurrentTraceContext currentTraceContext() { return currentTraceContext; }
    @Override public SpanCustomizer currentSpanCustomizer() {
        return BraveSpanCustomizer.fromBrave(tracer.currentSpanCustomizer());
    }
    @Override public Span currentSpan() {
        brave.Span c = tracer.currentSpan();
        return c == null ? null : BraveSpan.fromBrave(c);
    }
    @Override public TraceContext.Builder traceContextBuilder() { return new BraveTraceContextBuilder(); }

    @Override public Map<String, String> getAllBaggage() { return braveBaggageManager.getAllBaggage(); }
    @Override public Map<String, String> getAllBaggage(TraceContext tc) { return braveBaggageManager.getAllBaggage(tc); }
    @Override public Baggage getBaggage(String name) { return braveBaggageManager.getBaggage(name); }
    @Override public Baggage getBaggage(TraceContext tc, String name) { return braveBaggageManager.getBaggage(tc, name); }
    @Override public Baggage createBaggage(String name) { return braveBaggageManager.createBaggage(name); }
    @Override public Baggage createBaggage(String name, String value) { return braveBaggageManager.createBaggage(name, value); }
    @Override public BaggageInScope createBaggageInScope(String name, String value) { return braveBaggageManager.createBaggageInScope(name, value); }
    @Override public BaggageInScope createBaggageInScope(TraceContext tc, String name, String value) { return braveBaggageManager.createBaggageInScope(tc, name, value); }

    private static class BraveSpanInScope implements SpanInScope {
        private final brave.Tracer.SpanInScope delegate;
        BraveSpanInScope(brave.Tracer.SpanInScope delegate) { this.delegate = delegate; }
        @Override public void close() { delegate.close(); }
    }
}
```

### `[NEW]` `simple-tracing-bridge-brave/src/test/java/io/simpletracing/brave/bridge/BraveTracerTest.java`

*(See the full 26-test file in `src/test/` — covers span lifecycle, scoped spans,
parent-child relationships, span builder, current trace context, span customizer,
propagation inject/extract, export pipeline with filters and predicates, and
finished span mutation.)*
