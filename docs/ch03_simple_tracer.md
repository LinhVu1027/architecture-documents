# Chapter 3: Simple Tracer (In-Memory Implementation)

> Line references based on commit `dceab1b9` of the Micrometer Tracing repository.

## Build Challenge

| Current State | Limitation | Objective |
|---------------|------------|-----------|
| We have `Span`, `TraceContext`, `Tracer`, `CurrentTraceContext` — all as interfaces with NOOP implementations | No concrete implementation exists — every method returns a no-op, no real IDs are generated, no scope is tracked, no spans are collected | Build a working in-memory tracer: `SimpleTracer`, `SimpleSpan`, `SimpleTraceContext`, `SimpleCurrentTraceContext`, `SimpleScopedSpan`, `SimpleSpanBuilder`, `SimpleSpanCustomizer`, `SimpleSpanInScope` — all with real behavior |

---

## 3.1 The Integration Point: SimpleTracer Brings the Interfaces to Life

This feature is the first **concrete implementation** of the tracing abstractions from Chapters 1-2. The integration point is `SimpleTracer` itself — it's the hub that connects:

- **Span creation** — `nextSpan()` creates `SimpleSpan` instances with real hex IDs
- **Scope management** — `withSpan()` uses `SimpleCurrentTraceContext` backed by `ThreadLocal`
- **Parent-child relationships** — child spans inherit the parent's traceId and record its spanId as their parentId
- **Span collection** — all created spans are stored in a `Deque<SimpleSpan>` for test inspection

Every future feature depends on `SimpleTracer` for testing. Feature 4 (Export Pipeline) will hook into `span.end()`. Feature 11 (Test Assertions) will query the collected spans. Features 8-9 (Observation Handlers) will use `SimpleTracer` as the backing tracer.

**Direction:** We start with `SimpleTraceContext` (ID generation), then `SimpleSpan` (data recording), then `SimpleCurrentTraceContext` (scope management), then `SimpleTracer` (the coordinator that ties them together), and finally the helper classes (`SimpleScopedSpan`, `SimpleSpanBuilder`, `SimpleSpanCustomizer`, `SimpleSpanInScope`).

Three key decisions:

1. **Instance-based state** — the real framework uses `static` fields for the `ThreadLocal<SimpleSpan>` and the `Map<TraceContext, SimpleSpan>`. We use instance fields so separate `SimpleTracer` instances don't interfere — cleaner for tests.
2. **TraceContext→Span map** — `newScope(TraceContext)` needs to look up which `SimpleSpan` owns that context. We maintain a `ConcurrentHashMap<TraceContext, SimpleSpan>` in the tracer, registered when each span is created.
3. **Proper scope restoration** — unlike the real test implementation (which returns `Scope.NOOP` when clearing scope), we always save and restore the previous span, so nested `withSpan(null)` calls work correctly.

## 3.2 SimpleTraceContext — The Mutable Tracking Number

**New file:** `src/main/java/dev/linhvu/tracing/simple/SimpleTraceContext.java`

```java
package dev.linhvu.tracing.simple;

import dev.linhvu.tracing.TraceContext;

import java.util.concurrent.ThreadLocalRandom;

public class SimpleTraceContext implements TraceContext {

    private volatile String traceId = "";

    private volatile String parentId = "";

    private volatile String spanId = "";

    private volatile Boolean sampled = true;

    @Override
    public String traceId() {
        return traceId;
    }

    @Override
    public String parentId() {
        return parentId;
    }

    @Override
    public String spanId() {
        return spanId;
    }

    @Override
    public Boolean sampled() {
        return sampled;
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

    String generateId() {
        return String.format("%016x", ThreadLocalRandom.current().nextLong());
    }

    @Override
    public String toString() {
        return "SimpleTraceContext{traceId='" + traceId + "', parentId='" + parentId
                + "', spanId='" + spanId + "', sampled=" + sampled + "}";
    }

}
```

The `TraceContext` interface from Chapter 1 is read-only — it only exposes getters. `SimpleTraceContext` adds setters so the tracer can configure IDs during span creation. The `generateId()` method produces 16-character lowercase hex strings from random longs.

**Simplification:** The real framework uses `EncodingUtils.fromLong(random.nextLong())` with hand-optimized hex encoding and `ThreadLocal` char buffers. We use `String.format("%016x", ...)` — same output, simpler code.

## 3.3 SimpleTraceContextBuilder — Reconstructing Contexts

**New file:** `src/main/java/dev/linhvu/tracing/simple/SimpleTraceContextBuilder.java`

```java
package dev.linhvu.tracing.simple;

import dev.linhvu.tracing.TraceContext;

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
        SimpleTraceContext ctx = new SimpleTraceContext();
        if (traceId != null) {
            ctx.setTraceId(traceId);
        }
        if (spanId != null) {
            ctx.setSpanId(spanId);
        }
        if (parentId != null) {
            ctx.setParentId(parentId);
        }
        if (sampled != null) {
            ctx.setSampled(sampled);
        }
        return ctx;
    }

}
```

The builder is used when reconstructing a trace context from extracted data (e.g., incoming HTTP headers in Feature 5). It's also returned by `SimpleTracer.traceContextBuilder()`.

## 3.4 SimpleSpan — The In-Memory Recording Span

**New file:** `src/main/java/dev/linhvu/tracing/simple/SimpleSpan.java`

```java
package dev.linhvu.tracing.simple;

import dev.linhvu.tracing.Span;

import java.time.Instant;
import java.util.AbstractMap;
import java.util.Collection;
import java.util.Collections;
import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.ConcurrentLinkedQueue;
import java.util.concurrent.TimeUnit;

public class SimpleSpan implements Span {

    private final Map<String, String> tags = new ConcurrentHashMap<>();

    private final ConcurrentLinkedQueue<Map.Entry<Long, String>> events = new ConcurrentLinkedQueue<>();

    private final SimpleTraceContext context = new SimpleTraceContext();

    private volatile String name;

    private volatile long startMillis;

    private volatile long endMillis;

    private volatile Throwable error;

    private volatile Kind spanKind;

    private volatile String remoteServiceName;

    private volatile String remoteIp;

    private volatile int remotePort;

    private volatile boolean abandoned;

    @Override
    public boolean isNoop() {
        return false;
    }

    @Override
    public SimpleTraceContext context() {
        return context;
    }

    @Override
    public SimpleSpan start() {
        this.startMillis = System.currentTimeMillis();
        return this;
    }

    @Override
    public SimpleSpan name(String name) {
        this.name = name;
        return this;
    }

    @Override
    public SimpleSpan tag(String key, String value) {
        this.tags.put(key, value);
        return this;
    }

    @Override
    public SimpleSpan event(String value) {
        this.events.add(new AbstractMap.SimpleEntry<>(
                TimeUnit.MILLISECONDS.toMicros(System.currentTimeMillis()), value));
        return this;
    }

    @Override
    public SimpleSpan event(String value, long time, TimeUnit timeUnit) {
        this.events.add(new AbstractMap.SimpleEntry<>(timeUnit.toMicros(time), value));
        return this;
    }

    @Override
    public SimpleSpan error(Throwable throwable) {
        this.error = throwable;
        return this;
    }

    @Override
    public void end() {
        this.endMillis = System.currentTimeMillis();
    }

    @Override
    public void end(long time, TimeUnit timeUnit) {
        this.endMillis = timeUnit.toMillis(time);
    }

    @Override
    public void abandon() {
        this.abandoned = true;
    }

    @Override
    public SimpleSpan remoteServiceName(String remoteServiceName) {
        this.remoteServiceName = remoteServiceName;
        return this;
    }

    @Override
    public SimpleSpan remoteIpAndPort(String ip, int port) {
        this.remoteIp = ip;
        this.remotePort = port;
        return this;
    }

    // --- Accessors for test inspection ---

    public String getName() {
        return name;
    }

    public Map<String, String> getTags() {
        return Collections.unmodifiableMap(tags);
    }

    public Collection<Map.Entry<Long, String>> getEvents() {
        return Collections.unmodifiableCollection(events);
    }

    public Throwable getError() {
        return error;
    }

    public Kind getKind() {
        return spanKind;
    }

    public String getRemoteServiceName() {
        return remoteServiceName;
    }

    public String getRemoteIp() {
        return remoteIp;
    }

    public int getRemotePort() {
        return remotePort;
    }

    public Instant getStartTimestamp() {
        return Instant.ofEpochMilli(startMillis);
    }

    public Instant getEndTimestamp() {
        return Instant.ofEpochMilli(endMillis);
    }

    public boolean isAbandoned() {
        return abandoned;
    }

    void setSpanKind(Kind kind) {
        this.spanKind = kind;
    }

    void setStartMillis(long startMillis) {
        this.startMillis = startMillis;
    }

    @Override
    public String toString() {
        return "SimpleSpan{name='" + name + "', traceId='" + context.traceId()
                + "', spanId='" + context.spanId() + "', parentId='" + context.parentId() + "'}";
    }

}
```

Each `SimpleSpan` owns a `SimpleTraceContext` — the IDs are configured by the tracer after creation. The span records all data in-memory: tags in a `ConcurrentHashMap`, events in a `ConcurrentLinkedQueue`, timestamps via `System.currentTimeMillis()`. The accessor methods (getters) return unmodifiable views for safe test inspection.

**Simplification:** The real `SimpleSpan` also implements `FinishedSpan` (from the export pipeline, Feature 4). We don't implement that yet — it'll be added in Chapter 4. The real version also uses a `Clock` abstraction for testable time; we use `System.currentTimeMillis()` directly.

## 3.5 SimpleCurrentTraceContext — ThreadLocal Scope Stack

**New file:** `src/main/java/dev/linhvu/tracing/simple/SimpleCurrentTraceContext.java`

```java
package dev.linhvu.tracing.simple;

import dev.linhvu.tracing.CurrentTraceContext;
import dev.linhvu.tracing.Span;
import dev.linhvu.tracing.TraceContext;

import java.util.Objects;
import java.util.concurrent.Callable;
import java.util.concurrent.Executor;

public class SimpleCurrentTraceContext implements CurrentTraceContext {

    private final SimpleTracer tracer;

    SimpleCurrentTraceContext(SimpleTracer tracer) {
        this.tracer = tracer;
    }

    @Override
    public TraceContext context() {
        Span span = tracer.currentSpan();
        return span != null ? span.context() : null;
    }

    @Override
    public Scope newScope(TraceContext context) {
        SimpleSpan previous = tracer.getCurrentSpanInternal();

        if (context != null) {
            tracer.setCurrentSpan(context);
        }
        else {
            tracer.resetCurrentSpan();
        }

        // Always restore previous state on close — even when clearing to null
        return previous != null
                ? () -> tracer.setCurrentSpanDirect(previous)
                : () -> tracer.resetCurrentSpan();
    }

    @Override
    public Scope maybeScope(TraceContext context) {
        SimpleSpan current = tracer.getCurrentSpanInternal();

        if (context == null) {
            if (current == null) {
                return Scope.NOOP;
            }
            return newScope(null);
        }

        if (current != null && Objects.equals(current.context(), context)) {
            return Scope.NOOP;
        }
        return newScope(context);
    }

    @Override
    public <C> Callable<C> wrap(Callable<C> task) {
        return task;
    }

    @Override
    public Runnable wrap(Runnable task) {
        return task;
    }

    @Override
    public Executor wrap(Executor delegate) {
        return delegate;
    }

}
```

This is where scope management happens. `newScope(context)` saves the previous span, sets the new one, and returns a `Scope` lambda that restores the previous state on close. This forms an implicit **stack**: nesting `newScope` calls pushes onto the stack, and closing scopes pops back.

The `maybeScope` optimization avoids creating a new scope if the given context is already current — this prevents redundant scope churn in hot paths where the same context might be scoped multiple times.

**Simplification:** The real `SimpleCurrentTraceContext` also propagates baggage from the parent context when entering a new scope. We skip this since baggage (Feature 6) isn't implemented yet. The real version also returns `Scope.NOOP` when context is null (losing the previous state); we properly save and restore.

## 3.6 SimpleTracer — The Coordinator

**New file:** `src/main/java/dev/linhvu/tracing/simple/SimpleTracer.java`

```java
package dev.linhvu.tracing.simple;

import dev.linhvu.tracing.Baggage;
import dev.linhvu.tracing.BaggageInScope;
import dev.linhvu.tracing.ScopedSpan;
import dev.linhvu.tracing.Span;
import dev.linhvu.tracing.SpanCustomizer;
import dev.linhvu.tracing.TraceContext;
import dev.linhvu.tracing.Tracer;

import java.util.Collections;
import java.util.Deque;
import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.LinkedBlockingDeque;

public class SimpleTracer implements Tracer {

    private final Map<TraceContext, SimpleSpan> contextToSpan = new ConcurrentHashMap<>();

    private final ThreadLocal<SimpleSpan> currentSpan = new ThreadLocal<>();

    private final SimpleCurrentTraceContext currentTraceContext;

    private final Deque<SimpleSpan> spans = new LinkedBlockingDeque<>();

    public SimpleTracer() {
        this.currentTraceContext = new SimpleCurrentTraceContext(this);
    }

    @Override
    public SimpleSpan nextSpan() {
        SimpleSpan span = createSpan(currentSpan());
        return span;
    }

    @Override
    public SimpleSpan nextSpan(Span parent) {
        SimpleSpan span = createSpan(parent);
        return span;
    }

    private SimpleSpan createSpan(Span parent) {
        SimpleSpan span = new SimpleSpan();
        SimpleTraceContext ctx = span.context();

        if (parent != null) {
            // Child span: inherit traceId, record parent's spanId
            ctx.setTraceId(parent.context().traceId());
            ctx.setParentId(parent.context().spanId());
            ctx.setSpanId(ctx.generateId());
        }
        else {
            // Root span: traceId = spanId (new trace)
            String id = ctx.generateId();
            ctx.setTraceId(id);
            ctx.setSpanId(id);
        }

        contextToSpan.put(ctx, span);
        spans.add(span);
        return span;
    }

    @Override
    public SimpleSpanInScope withSpan(Span span) {
        return new SimpleSpanInScope(
                currentTraceContext.newScope(span != null ? span.context() : null));
    }

    @Override
    public ScopedSpan startScopedSpan(String name) {
        return new SimpleScopedSpan(this).name(name);
    }

    @Override
    public SimpleSpanBuilder spanBuilder() {
        return new SimpleSpanBuilder(this);
    }

    @Override
    public TraceContext.Builder traceContextBuilder() {
        return new SimpleTraceContextBuilder();
    }

    @Override
    public SimpleCurrentTraceContext currentTraceContext() {
        return currentTraceContext;
    }

    @Override
    public SpanCustomizer currentSpanCustomizer() {
        return new SimpleSpanCustomizer(this);
    }

    @Override
    public SimpleSpan currentSpan() {
        return currentSpan.get();
    }

    // --- BaggageManager (NOOP stubs until Feature 6) ---

    @Override
    public Map<String, String> getAllBaggage() {
        return Collections.emptyMap();
    }

    @Override
    public Baggage getBaggage(String name) {
        return Baggage.NOOP;
    }

    @Override
    public Baggage getBaggage(TraceContext traceContext, String name) {
        return Baggage.NOOP;
    }

    @Override
    public Baggage createBaggage(String name) {
        return Baggage.NOOP;
    }

    @Override
    public Baggage createBaggage(String name, String value) {
        return Baggage.NOOP;
    }

    // --- Convenience methods for test inspection ---

    public Deque<SimpleSpan> getSpans() {
        return spans;
    }

    public SimpleSpan onlySpan() {
        if (spans.size() != 1) {
            throw new AssertionError("Expected 1 span, got " + spans.size());
        }
        SimpleSpan span = spans.getFirst();
        if (span.getStartTimestamp().toEpochMilli() == 0) {
            throw new AssertionError("Span must be started");
        }
        if (span.getEndTimestamp().toEpochMilli() == 0) {
            throw new AssertionError("Span must be finished");
        }
        return span;
    }

    public SimpleSpan lastSpan() {
        if (spans.isEmpty()) {
            throw new AssertionError("No spans created");
        }
        return spans.getLast();
    }

    // --- Package-private scope management (used by SimpleCurrentTraceContext) ---

    SimpleSpan getCurrentSpanInternal() {
        return currentSpan.get();
    }

    void setCurrentSpanDirect(SimpleSpan span) {
        currentSpan.set(span);
    }

    void resetCurrentSpan() {
        currentSpan.remove();
    }

    void setCurrentSpan(TraceContext context) {
        SimpleSpan span = contextToSpan.get(context);
        if (span != null) {
            currentSpan.set(span);
        }
        else {
            currentSpan.remove();
        }
    }

    void registerSpan(SimpleSpan span) {
        contextToSpan.put(span.context(), span);
        spans.add(span);
    }

}
```

`SimpleTracer` is the coordinator. The `createSpan(parent)` method is the core logic:
- **Root span** (no parent): generates one ID used for both `traceId` and `spanId`
- **Child span** (has parent): copies the parent's `traceId`, records the parent's `spanId` as `parentId`, generates a new `spanId`

The `contextToSpan` map bridges `TraceContext` → `SimpleSpan`, enabling `SimpleCurrentTraceContext.newScope(context)` to find the right span to make "current."

Baggage methods all return `NOOP` — Feature 6 will replace these stubs with a real `SimpleBaggageManager`.

## 3.7 Helper Classes

### SimpleSpanInScope — Scope Wrapper

**New file:** `src/main/java/dev/linhvu/tracing/simple/SimpleSpanInScope.java`

```java
package dev.linhvu.tracing.simple;

import dev.linhvu.tracing.CurrentTraceContext;
import dev.linhvu.tracing.Tracer;

public class SimpleSpanInScope implements Tracer.SpanInScope {

    private final CurrentTraceContext.Scope scope;

    SimpleSpanInScope(CurrentTraceContext.Scope scope) {
        this.scope = scope;
    }

    @Override
    public void close() {
        scope.close();
    }

}
```

### SimpleSpanCustomizer — Delegates to Current Span

**New file:** `src/main/java/dev/linhvu/tracing/simple/SimpleSpanCustomizer.java`

```java
package dev.linhvu.tracing.simple;

import dev.linhvu.tracing.Span;
import dev.linhvu.tracing.SpanCustomizer;

public class SimpleSpanCustomizer implements SpanCustomizer {

    private final SimpleTracer tracer;

    SimpleSpanCustomizer(SimpleTracer tracer) {
        this.tracer = tracer;
    }

    private Span currentSpan() {
        Span span = tracer.currentSpan();
        if (span == null) {
            throw new IllegalStateException("No current span in scope");
        }
        return span;
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

}
```

### SimpleScopedSpan — Auto-Scoped Convenience

**New file:** `src/main/java/dev/linhvu/tracing/simple/SimpleScopedSpan.java`

```java
package dev.linhvu.tracing.simple;

import dev.linhvu.tracing.ScopedSpan;
import dev.linhvu.tracing.TraceContext;
import dev.linhvu.tracing.Tracer;

public class SimpleScopedSpan implements ScopedSpan {

    private final SimpleSpan span;

    private final Tracer.SpanInScope scope;

    SimpleScopedSpan(SimpleTracer tracer) {
        this.span = tracer.nextSpan();
        this.span.start();
        this.scope = tracer.withSpan(span);
    }

    @Override
    public boolean isNoop() {
        return false;
    }

    @Override
    public TraceContext context() {
        return span.context();
    }

    @Override
    public ScopedSpan name(String name) {
        span.name(name);
        return this;
    }

    @Override
    public ScopedSpan tag(String key, String value) {
        span.tag(key, value);
        return this;
    }

    @Override
    public ScopedSpan event(String value) {
        span.event(value);
        return this;
    }

    @Override
    public ScopedSpan error(Throwable throwable) {
        span.error(throwable);
        return this;
    }

    @Override
    public void end() {
        scope.close();
        span.end();
    }

}
```

The constructor does three things: creates a span, starts it, and scopes it. `end()` reverses the order: closes the scope (restoring the previous span), then ends the span. This matches the `ScopedSpan` contract: "automatically in scope from creation until `end()` is called."

### SimpleSpanBuilder — Deferred Configuration

**New file:** `src/main/java/dev/linhvu/tracing/simple/SimpleSpanBuilder.java`

```java
package dev.linhvu.tracing.simple;

import dev.linhvu.tracing.Link;
import dev.linhvu.tracing.Span;
import dev.linhvu.tracing.TraceContext;

import java.util.ArrayList;
import java.util.HashMap;
import java.util.List;
import java.util.Map;
import java.util.concurrent.TimeUnit;

public class SimpleSpanBuilder implements Span.Builder {

    private final SimpleTracer tracer;

    private final List<String> events = new ArrayList<>();

    private final Map<String, String> tags = new HashMap<>();

    private final List<Link> links = new ArrayList<>();

    private Throwable error;

    private String remoteServiceName;

    private Span.Kind spanKind;

    private String name;

    private String ip;

    private int port;

    private long startTimestamp;

    private TimeUnit startTimestampUnit;

    private TraceContext parentContext;

    SimpleSpanBuilder(SimpleTracer tracer) {
        this.tracer = tracer;
    }

    @Override
    public Span.Builder setParent(TraceContext context) {
        this.parentContext = context;
        return this;
    }

    @Override
    public Span.Builder setNoParent() {
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
        this.error = throwable;
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
    public Span.Builder remoteIpAndPort(String ip, int port) {
        this.ip = ip;
        this.port = port;
        return this;
    }

    @Override
    public Span.Builder startTimestamp(long startTimestamp, TimeUnit unit) {
        this.startTimestamp = startTimestamp;
        this.startTimestampUnit = unit;
        return this;
    }

    @Override
    public Span.Builder addLink(Link link) {
        this.links.add(link);
        return this;
    }

    @Override
    public SimpleSpan start() {
        SimpleSpan span = new SimpleSpan();

        // Apply accumulated configuration
        tags.forEach(span::tag);
        events.forEach(span::event);
        if (name != null) {
            span.name(name);
        }
        if (error != null) {
            span.error(error);
        }
        if (spanKind != null) {
            span.setSpanKind(spanKind);
        }
        if (remoteServiceName != null) {
            span.remoteServiceName(remoteServiceName);
        }
        if (ip != null) {
            span.remoteIpAndPort(ip, port);
        }

        // Set up trace context (parent-child IDs)
        SimpleTraceContext ctx = span.context();
        if (parentContext != null) {
            ctx.setTraceId(parentContext.traceId());
            ctx.setParentId(parentContext.spanId());
            ctx.setSpanId(ctx.generateId());
            ctx.setSampled(parentContext.sampled());
        }
        else {
            String id = ctx.generateId();
            ctx.setTraceId(id);
            ctx.setSpanId(id);
        }

        // Start the span
        span.start();
        if (startTimestampUnit != null) {
            span.setStartMillis(startTimestampUnit.toMillis(startTimestamp));
        }

        // Register with tracer
        tracer.registerSpan(span);
        return span;
    }

}
```

The builder accumulates configuration (tags, events, name, kind, parent) and applies it all at once in `start()`. The parent context handling mirrors `SimpleTracer.createSpan()` — if a parent is set, the new span inherits its traceId and records its spanId as parentId.

## 3.8 Try It Yourself

<details>
<summary>Challenge: What happens if you close scopes in the wrong order?</summary>

Consider this code:

```java
SimpleTracer tracer = new SimpleTracer();
Span outer = tracer.nextSpan().name("outer").start();
Span inner = tracer.nextSpan(outer).name("inner").start();

Tracer.SpanInScope ws1 = tracer.withSpan(outer);   // scope for outer
Tracer.SpanInScope ws2 = tracer.withSpan(inner);   // scope for inner

// Close in WRONG order — ws1 first, then ws2
ws1.close();
// Current span is now null (ws1 restores to "nothing before outer")
ws2.close();
// Current span is now outer (ws2 restores to "outer was before inner")
```

The scope restoration is stack-based. If you close out of order, you get incorrect state. This is why try-with-resources is the recommended pattern — it guarantees LIFO (last in, first out) closure.

</details>

<details>
<summary>Challenge: Why does the root span use traceId == spanId?</summary>

In `createSpan(null)`:
```java
String id = ctx.generateId();
ctx.setTraceId(id);
ctx.setSpanId(id);
```

The root span starts a new trace. It generates one ID and uses it for both `traceId` and `spanId`. This is a convention from the W3C Trace Context spec: the first span in a trace shares its span ID as the trace ID. All child spans will inherit this `traceId` while generating their own unique `spanId`.

This convention also provides a quick check: if `traceId == spanId` and `parentId` is empty, it's the root span.

</details>

<details>
<summary>Challenge: Implement a minimal SimpleTracer that only supports nextSpan() and withSpan()</summary>

Try building the simplest possible tracer with just:
- A `ThreadLocal<SimpleSpan>` for current span
- `nextSpan()` that creates a root or child span
- `withSpan()` that scopes a span

```java
public class MinimalTracer {
    private final ThreadLocal<SimpleSpan> current = new ThreadLocal<>();

    public SimpleSpan nextSpan() {
        SimpleSpan span = new SimpleSpan();
        SimpleTraceContext ctx = span.context();
        SimpleSpan parent = current.get();
        if (parent != null) {
            ctx.setTraceId(parent.context().traceId());
            ctx.setParentId(parent.context().spanId());
            ctx.setSpanId(ctx.generateId());
        } else {
            String id = ctx.generateId();
            ctx.setTraceId(id);
            ctx.setSpanId(id);
        }
        return span;
    }

    public AutoCloseable withSpan(SimpleSpan span) {
        SimpleSpan previous = current.get();
        current.set(span);
        return () -> {
            if (previous != null) current.set(previous);
            else current.remove();
        };
    }

    public SimpleSpan currentSpan() {
        return current.get();
    }
}
```

This is ~30 lines for the core concept. Everything else in `SimpleTracer` is convenience and completeness: `spanBuilder()`, `startScopedSpan()`, `currentSpanCustomizer()`, `onlySpan()`, etc.

</details>

## 3.9 Tests

### Unit Tests

**New file:** `src/test/java/dev/linhvu/tracing/simple/SimpleTraceContextTest.java`

```java
package dev.linhvu.tracing.simple;

import org.junit.jupiter.api.Test;

import java.util.HashSet;
import java.util.Set;

import static org.assertj.core.api.Assertions.assertThat;

class SimpleTraceContextTest {

    @Test
    void shouldGenerateHexId_WhenCallingGenerateId() {
        SimpleTraceContext ctx = new SimpleTraceContext();
        String id = ctx.generateId();
        assertThat(id).hasSize(16).matches("[0-9a-f]{16}");
    }

    @Test
    void shouldGenerateUniqueIds_WhenCalledMultipleTimes() {
        SimpleTraceContext ctx = new SimpleTraceContext();
        Set<String> ids = new HashSet<>();
        for (int i = 0; i < 100; i++) {
            ids.add(ctx.generateId());
        }
        // With 64-bit random IDs, collisions in 100 draws are astronomically unlikely
        assertThat(ids).hasSizeGreaterThan(95);
    }

    @Test
    void shouldStoreTraceId_WhenSet() {
        SimpleTraceContext ctx = new SimpleTraceContext();
        ctx.setTraceId("abc123def456");
        assertThat(ctx.traceId()).isEqualTo("abc123def456");
    }

    @Test
    void shouldStoreSpanId_WhenSet() {
        SimpleTraceContext ctx = new SimpleTraceContext();
        ctx.setSpanId("span-001");
        assertThat(ctx.spanId()).isEqualTo("span-001");
    }

    @Test
    void shouldStoreParentId_WhenSet() {
        SimpleTraceContext ctx = new SimpleTraceContext();
        ctx.setParentId("parent-001");
        assertThat(ctx.parentId()).isEqualTo("parent-001");
    }

    @Test
    void shouldStoreSampled_WhenSet() {
        SimpleTraceContext ctx = new SimpleTraceContext();
        ctx.setSampled(false);
        assertThat(ctx.sampled()).isFalse();

        ctx.setSampled(null);
        assertThat(ctx.sampled()).isNull();
    }

    @Test
    void shouldHaveEmptyDefaults_WhenCreated() {
        SimpleTraceContext ctx = new SimpleTraceContext();
        assertThat(ctx.traceId()).isEmpty();
        assertThat(ctx.spanId()).isEmpty();
        assertThat(ctx.parentId()).isEmpty();
        assertThat(ctx.sampled()).isTrue();
    }

    @Test
    void shouldBuildContext_WhenUsingBuilder() {
        SimpleTraceContextBuilder builder = new SimpleTraceContextBuilder();
        var ctx = builder.traceId("trace-1")
                .spanId("span-1")
                .parentId("parent-1")
                .sampled(true)
                .build();

        assertThat(ctx.traceId()).isEqualTo("trace-1");
        assertThat(ctx.spanId()).isEqualTo("span-1");
        assertThat(ctx.parentId()).isEqualTo("parent-1");
        assertThat(ctx.sampled()).isTrue();
    }

    @Test
    void shouldBuildPartialContext_WhenOnlySomeFieldsSet() {
        SimpleTraceContextBuilder builder = new SimpleTraceContextBuilder();
        var ctx = (SimpleTraceContext) builder.traceId("trace-1").build();

        assertThat(ctx.traceId()).isEqualTo("trace-1");
        assertThat(ctx.spanId()).isEmpty();
        assertThat(ctx.parentId()).isEmpty();
    }

}
```

**New file:** `src/test/java/dev/linhvu/tracing/simple/SimpleSpanTest.java`

```java
package dev.linhvu.tracing.simple;

import org.junit.jupiter.api.Test;

import java.time.Instant;
import java.util.concurrent.TimeUnit;

import static org.assertj.core.api.Assertions.assertThat;

class SimpleSpanTest {

    @Test
    void shouldRecordName_WhenNameSet() {
        SimpleSpan span = new SimpleSpan();
        span.name("test-operation");
        assertThat(span.getName()).isEqualTo("test-operation");
    }

    @Test
    void shouldRecordTags_WhenTagsAdded() {
        SimpleSpan span = new SimpleSpan();
        span.tag("http.method", "GET");
        span.tag("http.status", "200");

        assertThat(span.getTags())
                .containsEntry("http.method", "GET")
                .containsEntry("http.status", "200");
    }

    @Test
    void shouldRecordEvents_WhenEventsAdded() {
        SimpleSpan span = new SimpleSpan();
        span.event("request.started");
        span.event("response.received");

        assertThat(span.getEvents()).hasSize(2);
        assertThat(span.getEvents())
                .extracting(e -> e.getValue())
                .containsExactly("request.started", "response.received");
    }

    @Test
    void shouldRecordTimestampedEvent_WhenEventWithTimeAdded() {
        SimpleSpan span = new SimpleSpan();
        span.event("custom", 1000L, TimeUnit.MILLISECONDS);

        assertThat(span.getEvents()).hasSize(1);
        var event = span.getEvents().iterator().next();
        assertThat(event.getKey()).isEqualTo(1000000L); // millis -> micros
        assertThat(event.getValue()).isEqualTo("custom");
    }

    @Test
    void shouldRecordError_WhenErrorSet() {
        SimpleSpan span = new SimpleSpan();
        RuntimeException ex = new RuntimeException("oops");
        span.error(ex);

        assertThat(span.getError()).isSameAs(ex);
    }

    @Test
    void shouldRecordStartTimestamp_WhenStarted() {
        Instant before = Instant.ofEpochMilli(System.currentTimeMillis());
        SimpleSpan span = new SimpleSpan();
        span.start();
        Instant after = Instant.ofEpochMilli(System.currentTimeMillis());

        assertThat(span.getStartTimestamp())
                .isAfterOrEqualTo(before)
                .isBeforeOrEqualTo(after);
    }

    @Test
    void shouldRecordEndTimestamp_WhenEnded() {
        SimpleSpan span = new SimpleSpan();
        span.start();
        Instant before = Instant.ofEpochMilli(System.currentTimeMillis());
        span.end();
        Instant after = Instant.ofEpochMilli(System.currentTimeMillis());

        assertThat(span.getEndTimestamp())
                .isAfterOrEqualTo(before)
                .isBeforeOrEqualTo(after);
    }

    @Test
    void shouldRecordExplicitEndTimestamp_WhenEndedWithTime() {
        SimpleSpan span = new SimpleSpan();
        span.start();
        span.end(5000L, TimeUnit.MILLISECONDS);

        assertThat(span.getEndTimestamp()).isEqualTo(Instant.ofEpochMilli(5000));
    }

    @Test
    void shouldSetRemoteService_WhenConfigured() {
        SimpleSpan span = new SimpleSpan();
        span.remoteServiceName("user-service");
        span.remoteIpAndPort("10.0.0.1", 8080);

        assertThat(span.getRemoteServiceName()).isEqualTo("user-service");
        assertThat(span.getRemoteIp()).isEqualTo("10.0.0.1");
        assertThat(span.getRemotePort()).isEqualTo(8080);
    }

    @Test
    void shouldMarkAbandoned_WhenAbandoned() {
        SimpleSpan span = new SimpleSpan();
        assertThat(span.isAbandoned()).isFalse();

        span.abandon();
        assertThat(span.isAbandoned()).isTrue();
    }

    @Test
    void shouldNotBeNoop_WhenCreated() {
        SimpleSpan span = new SimpleSpan();
        assertThat(span.isNoop()).isFalse();
    }

    @Test
    void shouldReturnFluentSelf_WhenChaining() {
        SimpleSpan span = new SimpleSpan();
        SimpleSpan result = span.name("op").tag("k", "v").event("e").error(new RuntimeException());
        assertThat(result).isSameAs(span);
    }

}
```

**New file:** `src/test/java/dev/linhvu/tracing/simple/SimpleTracerTest.java`

```java
package dev.linhvu.tracing.simple;

import dev.linhvu.tracing.CurrentTraceContext;
import dev.linhvu.tracing.ScopedSpan;
import dev.linhvu.tracing.Span;
import dev.linhvu.tracing.SpanCustomizer;
import dev.linhvu.tracing.Tracer;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Nested;
import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

class SimpleTracerTest {

    SimpleTracer tracer;

    @BeforeEach
    void setUp() {
        tracer = new SimpleTracer();
    }

    @Nested
    class SpanCreation {

        @Test
        void shouldCreateRootSpan_WhenNoCurrentSpan() {
            SimpleSpan span = tracer.nextSpan();

            assertThat(span.context().traceId()).isNotEmpty();
            assertThat(span.context().spanId()).isNotEmpty();
            assertThat(span.context().parentId()).isEmpty();
            assertThat(span.context().traceId()).isEqualTo(span.context().spanId());
        }

        @Test
        void shouldCreateChildSpan_WhenParentInScope() {
            SimpleSpan root = tracer.nextSpan().name("root").start();
            try (Tracer.SpanInScope ws = tracer.withSpan(root)) {
                SimpleSpan child = tracer.nextSpan().name("child");

                assertThat(child.context().traceId()).isEqualTo(root.context().traceId());
                assertThat(child.context().parentId()).isEqualTo(root.context().spanId());
                assertThat(child.context().spanId()).isNotEqualTo(root.context().spanId());
            }
        }

        @Test
        void shouldCreateChildSpan_WhenExplicitParentProvided() {
            SimpleSpan parent = tracer.nextSpan().name("parent").start();
            SimpleSpan child = tracer.nextSpan(parent).name("child");

            assertThat(child.context().traceId()).isEqualTo(parent.context().traceId());
            assertThat(child.context().parentId()).isEqualTo(parent.context().spanId());
        }

        @Test
        void shouldCollectAllCreatedSpans() {
            tracer.nextSpan().name("span-1");
            tracer.nextSpan().name("span-2");
            tracer.nextSpan().name("span-3");

            assertThat(tracer.getSpans()).hasSize(3);
        }

        @Test
        void shouldGenerateValidHexIds() {
            SimpleSpan span = tracer.nextSpan();

            assertThat(span.context().traceId()).matches("[0-9a-f]{16}");
            assertThat(span.context().spanId()).matches("[0-9a-f]{16}");
        }
    }

    @Nested
    class ScopeManagement {

        @Test
        void shouldReturnNullCurrentSpan_WhenNoScope() {
            assertThat(tracer.currentSpan()).isNull();
        }

        @Test
        void shouldTrackCurrentSpan_WhenInScope() {
            SimpleSpan span = tracer.nextSpan().start();
            try (Tracer.SpanInScope ws = tracer.withSpan(span)) {
                assertThat(tracer.currentSpan()).isSameAs(span);
            }
        }

        @Test
        void shouldRestorePreviousSpan_WhenScopeClosed() {
            SimpleSpan outer = tracer.nextSpan().start();
            try (Tracer.SpanInScope ws1 = tracer.withSpan(outer)) {
                assertThat(tracer.currentSpan()).isSameAs(outer);

                SimpleSpan inner = tracer.nextSpan().start();
                try (Tracer.SpanInScope ws2 = tracer.withSpan(inner)) {
                    assertThat(tracer.currentSpan()).isSameAs(inner);
                }

                assertThat(tracer.currentSpan()).isSameAs(outer);
            }

            assertThat(tracer.currentSpan()).isNull();
        }

        @Test
        void shouldClearScope_WhenWithSpanCalledWithNull() {
            SimpleSpan span = tracer.nextSpan().start();
            try (Tracer.SpanInScope ws1 = tracer.withSpan(span)) {
                assertThat(tracer.currentSpan()).isSameAs(span);

                try (Tracer.SpanInScope ws2 = tracer.withSpan(null)) {
                    assertThat(tracer.currentSpan()).isNull();
                }

                assertThat(tracer.currentSpan()).isSameAs(span);
            }
        }

        @Test
        void shouldReturnContextFromCurrentTraceContext() {
            assertThat(tracer.currentTraceContext().context()).isNull();

            SimpleSpan span = tracer.nextSpan().start();
            try (Tracer.SpanInScope ws = tracer.withSpan(span)) {
                assertThat(tracer.currentTraceContext().context())
                        .isSameAs(span.context());
            }
        }

        @Test
        void shouldSupportMaybeScope_WhenContextAlreadyCurrent() {
            SimpleSpan span = tracer.nextSpan().start();
            try (Tracer.SpanInScope ws = tracer.withSpan(span)) {
                CurrentTraceContext.Scope scope =
                        tracer.currentTraceContext().maybeScope(span.context());
                assertThat(scope).isSameAs(CurrentTraceContext.Scope.NOOP);
            }
        }
    }

    @Nested
    class ScopedSpanTests {

        @Test
        void shouldStartAndScope_WhenScopedSpanCreated() {
            ScopedSpan scoped = tracer.startScopedSpan("my-operation");

            assertThat(tracer.currentSpan()).isNotNull();
            assertThat(tracer.getSpans().getLast().getStartTimestamp().toEpochMilli())
                    .isGreaterThan(0);

            scoped.end();

            assertThat(tracer.currentSpan()).isNull();
            assertThat(tracer.getSpans().getLast().getEndTimestamp().toEpochMilli())
                    .isGreaterThan(0);
        }

        @Test
        void shouldRecordNameTagsEvents_WhenScopedSpanUsed() {
            ScopedSpan scoped = tracer.startScopedSpan("test-op");
            scoped.tag("key", "value");
            scoped.event("something.happened");
            scoped.end();

            SimpleSpan span = tracer.lastSpan();
            assertThat(span.getName()).isEqualTo("test-op");
            assertThat(span.getTags()).containsEntry("key", "value");
            assertThat(span.getEvents()).extracting(e -> e.getValue())
                    .contains("something.happened");
        }

        @Test
        void shouldRecordError_WhenScopedSpanHasError() {
            ScopedSpan scoped = tracer.startScopedSpan("failing-op");
            RuntimeException ex = new RuntimeException("fail");
            scoped.error(ex);
            scoped.end();

            assertThat(tracer.lastSpan().getError()).isSameAs(ex);
        }
    }

    @Nested
    class SpanBuilderTests {

        @Test
        void shouldBuildSpanWithConfiguration_WhenBuilderUsed() {
            Span span = tracer.spanBuilder()
                    .name("built-span")
                    .tag("env", "test")
                    .kind(Span.Kind.CLIENT)
                    .remoteServiceName("backend")
                    .start();

            SimpleSpan simple = (SimpleSpan) span;
            assertThat(simple.getName()).isEqualTo("built-span");
            assertThat(simple.getTags()).containsEntry("env", "test");
            assertThat(simple.getKind()).isEqualTo(Span.Kind.CLIENT);
            assertThat(simple.getRemoteServiceName()).isEqualTo("backend");
            assertThat(simple.getStartTimestamp().toEpochMilli()).isGreaterThan(0);
        }

        @Test
        void shouldBuildChildSpan_WhenParentContextSet() {
            SimpleSpan parent = tracer.nextSpan().name("parent").start();

            Span child = tracer.spanBuilder()
                    .setParent(parent.context())
                    .name("child")
                    .start();

            assertThat(child.context().traceId()).isEqualTo(parent.context().traceId());
            assertThat(child.context().parentId()).isEqualTo(parent.context().spanId());
        }

        @Test
        void shouldBuildRootSpan_WhenNoParent() {
            Span span = tracer.spanBuilder().name("root").start();

            assertThat(span.context().parentId()).isEmpty();
            assertThat(span.context().traceId()).isEqualTo(span.context().spanId());
        }

        @Test
        void shouldRegisterWithTracer_WhenBuilderStartCalled() {
            tracer.spanBuilder().name("builder-span").start();

            assertThat(tracer.getSpans()).hasSize(1);
            assertThat(tracer.lastSpan().getName()).isEqualTo("builder-span");
        }
    }

    @Nested
    class SpanCustomizerTests {

        @Test
        void shouldDelegateToCurrentSpan_WhenInScope() {
            SimpleSpan span = tracer.nextSpan().start();
            try (Tracer.SpanInScope ws = tracer.withSpan(span)) {
                SpanCustomizer customizer = tracer.currentSpanCustomizer();
                customizer.name("customized");
                customizer.tag("custom-key", "custom-value");
                customizer.event("custom-event");
            }

            assertThat(span.getName()).isEqualTo("customized");
            assertThat(span.getTags()).containsEntry("custom-key", "custom-value");
        }

        @Test
        void shouldThrowException_WhenNoSpanInScope() {
            SpanCustomizer customizer = tracer.currentSpanCustomizer();
            assertThatThrownBy(() -> customizer.name("oops"))
                    .isInstanceOf(IllegalStateException.class)
                    .hasMessageContaining("No current span in scope");
        }
    }

    @Nested
    class InspectionMethods {

        @Test
        void shouldReturnOnlySpan_WhenExactlyOneCompletedSpan() {
            tracer.nextSpan().name("single").start().end();

            SimpleSpan span = tracer.onlySpan();
            assertThat(span.getName()).isEqualTo("single");
        }

        @Test
        void shouldThrow_WhenOnlySpanCalledWithMultipleSpans() {
            tracer.nextSpan().start().end();
            tracer.nextSpan().start().end();

            assertThatThrownBy(() -> tracer.onlySpan())
                    .isInstanceOf(AssertionError.class)
                    .hasMessageContaining("Expected 1 span, got 2");
        }

        @Test
        void shouldReturnLastSpan_WhenMultipleSpansExist() {
            tracer.nextSpan().name("first");
            tracer.nextSpan().name("last");

            assertThat(tracer.lastSpan().getName()).isEqualTo("last");
        }

        @Test
        void shouldThrow_WhenLastSpanCalledWithNoSpans() {
            assertThatThrownBy(() -> tracer.lastSpan())
                    .isInstanceOf(AssertionError.class)
                    .hasMessageContaining("No spans created");
        }
    }

}
```

### Integration Test

**New file:** `src/test/java/dev/linhvu/tracing/integration/SimpleTracerIntegrationTest.java`

```java
package dev.linhvu.tracing.integration;

import dev.linhvu.tracing.ScopedSpan;
import dev.linhvu.tracing.Span;
import dev.linhvu.tracing.Tracer;
import dev.linhvu.tracing.simple.SimpleSpan;
import dev.linhvu.tracing.simple.SimpleTracer;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;

class SimpleTracerIntegrationTest {

    SimpleTracer tracer;

    @BeforeEach
    void setUp() {
        tracer = new SimpleTracer();
    }

    @Test
    void shouldMaintainTraceAcrossNestedSpans() {
        // Simulate: HTTP request -> database call -> cache lookup
        Span httpSpan = tracer.nextSpan().name("http-request").start();
        try (Tracer.SpanInScope ws1 = tracer.withSpan(httpSpan)) {
            httpSpan.tag("http.method", "GET");

            Span dbSpan = tracer.nextSpan().name("db-query").start();
            try (Tracer.SpanInScope ws2 = tracer.withSpan(dbSpan)) {
                dbSpan.tag("db.type", "postgresql");

                Span cacheSpan = tracer.nextSpan().name("cache-lookup").start();
                try (Tracer.SpanInScope ws3 = tracer.withSpan(cacheSpan)) {
                    cacheSpan.tag("cache.hit", "true");
                } finally {
                    cacheSpan.end();
                }
            } finally {
                dbSpan.end();
            }
        } finally {
            httpSpan.end();
        }

        // All three spans share the same traceId
        assertThat(tracer.getSpans()).hasSize(3);
        String traceId = httpSpan.context().traceId();
        for (SimpleSpan span : tracer.getSpans()) {
            assertThat(span.context().traceId()).isEqualTo(traceId);
        }

        // Parent-child relationships are correct
        SimpleSpan http = tracer.getSpans().stream()
                .filter(s -> "http-request".equals(s.getName())).findFirst().orElseThrow();
        SimpleSpan db = tracer.getSpans().stream()
                .filter(s -> "db-query".equals(s.getName())).findFirst().orElseThrow();
        SimpleSpan cache = tracer.getSpans().stream()
                .filter(s -> "cache-lookup".equals(s.getName())).findFirst().orElseThrow();

        assertThat(http.context().parentId()).isEmpty();
        assertThat(db.context().parentId()).isEqualTo(http.context().spanId());
        assertThat(cache.context().parentId()).isEqualTo(db.context().spanId());
    }

    @Test
    void shouldSupportMixedScopePatterns() {
        // Pattern 1: Explicit span + scope
        Span explicitSpan = tracer.nextSpan().name("explicit").start();
        try (Tracer.SpanInScope ws = tracer.withSpan(explicitSpan)) {
            explicitSpan.tag("pattern", "explicit");

            // Pattern 2: ScopedSpan (convenience)
            ScopedSpan scopedSpan = tracer.startScopedSpan("scoped");
            scopedSpan.tag("pattern", "scoped");

            assertThat(tracer.currentSpan()).isNotSameAs(explicitSpan);

            scopedSpan.end();

            assertThat(tracer.currentSpan()).isSameAs(explicitSpan);
        } finally {
            explicitSpan.end();
        }

        assertThat(tracer.getSpans()).hasSize(2);
    }

    @Test
    void shouldBuildSpanWithParentFromDifferentTrace() {
        // Simulate receiving a trace context from another service
        var extractedContext = tracer.traceContextBuilder()
                .traceId("aabbccddee112233")
                .spanId("1122334455667788")
                .sampled(true)
                .build();

        Span childSpan = tracer.spanBuilder()
                .setParent(extractedContext)
                .name("continued-from-remote")
                .kind(Span.Kind.SERVER)
                .start();

        SimpleSpan simple = (SimpleSpan) childSpan;
        assertThat(simple.context().traceId()).isEqualTo("aabbccddee112233");
        assertThat(simple.context().parentId()).isEqualTo("1122334455667788");
        assertThat(simple.context().spanId()).isNotEqualTo("1122334455667788");
        assertThat(simple.getKind()).isEqualTo(Span.Kind.SERVER);
    }

    @Test
    void shouldHandleErrorScenario() {
        Span span = tracer.nextSpan().name("failing-operation").start();
        try (Tracer.SpanInScope ws = tracer.withSpan(span)) {
            try {
                throw new RuntimeException("connection refused");
            }
            catch (Exception e) {
                span.error(e);
                span.tag("error.type", e.getClass().getSimpleName());
            }
        } finally {
            span.end();
        }

        SimpleSpan finished = tracer.onlySpan();
        assertThat(finished.getError()).isInstanceOf(RuntimeException.class);
        assertThat(finished.getError().getMessage()).isEqualTo("connection refused");
        assertThat(finished.getTags()).containsEntry("error.type", "RuntimeException");
    }

    @Test
    void shouldSupportAbandonedSpan() {
        Span span = tracer.nextSpan().name("maybe-operation").start();
        span.abandon();

        SimpleSpan simple = (SimpleSpan) span;
        assertThat(simple.isAbandoned()).isTrue();
        assertThat(tracer.getSpans()).hasSize(1);
    }

    @Test
    void shouldUseTracerThroughInterfaceType() {
        Tracer tracerInterface = tracer;

        Span span = tracerInterface.nextSpan().name("interface-test").start();
        try (Tracer.SpanInScope ws = tracerInterface.withSpan(span)) {
            assertThat(tracerInterface.currentSpan()).isNotNull();
            assertThat(tracerInterface.currentTraceContext().context()).isNotNull();
        } finally {
            span.end();
        }

        assertThat(tracerInterface.currentSpan()).isNull();
    }

}
```

## 3.10 Why This Works

> ★ **Insight** -------------------------------------------
> - **Why ThreadLocal for scope?** Each thread in a Java application has its own call stack. When thread A handles one HTTP request and thread B handles another, each needs its own "current span." `ThreadLocal` gives each thread an independent slot — writes on thread A are invisible to thread B. This is the same mechanism that `RequestContextHolder` in Spring MVC uses for `HttpServletRequest`.
> - **Trade-off:** ThreadLocal doesn't cross thread boundaries automatically. If you submit a task to a thread pool, the new thread won't see the current span. That's why `CurrentTraceContext.wrap(Runnable)` exists — to capture and restore context across threads. (Our simplified version passes through for now; real implementations do the capture.)
> -----------------------------------------------------------

> ★ **Insight** -------------------------------------------
> - **Why a TraceContext→Span map?** The scope management API works at the `TraceContext` level (IDs), not the `Span` level (full object). When `newScope(context)` is called, we need to find the `SimpleSpan` that owns that context so we can make it the "current span." Without this map, we'd need to scan all created spans — O(n) vs O(1). The real framework uses the same pattern with a static `ConcurrentHashMap`.
> - **When this matters:** In the Observation → Tracing bridge (Features 8-9), the `TracingObservationHandler` stores a `TraceContext` in the observation's context and later uses it to re-scope the span. The map is the mechanism that makes this roundtrip work.
> -----------------------------------------------------------

> ★ **Insight** -------------------------------------------
> - **Why do root spans use traceId == spanId?** This is a convention from the W3C Trace Context specification. The first span in a trace generates one random ID and uses it for both `traceId` and `spanId`. All child spans inherit this `traceId` while generating their own `spanId`. This means you can identify a root span by checking `traceId.equals(spanId) && parentId.isEmpty()`. The real framework's `SimpleSpan` follows the same convention.
> -----------------------------------------------------------

## 3.11 What We Enhanced

| File | Change | Why |
|------|--------|-----|
| *(no existing files modified)* | — | This feature adds 9 new classes in a new `dev.linhvu.tracing.simple` package. It implements the interfaces from Chapters 1-2 without modifying them. The integration point is the `implements Tracer` / `implements Span` / etc. declarations — the new classes plug into the existing type hierarchy. |

## 3.12 Connection to Real Framework

| Simplified Code | Real Framework Code | File:Line | Key Difference |
|----------------|--------------------|-----------| ---------------|
| `SimpleTracer` | `SimpleTracer` | `SimpleTracer.java:36` | Real uses static `ThreadLocal` and static `ConcurrentHashMap`; we use instance fields for test isolation |
| `SimpleSpan` | `SimpleSpan` | `SimpleSpan.java:40` | Real also implements `FinishedSpan` (export pipeline); we defer to Feature 4 |
| `SimpleTraceContext` | `SimpleTraceContext` | `SimpleTraceContext.java:33` | Real uses `EncodingUtils.fromLong(random.nextLong())`; we use `String.format("%016x", ...)` |
| `SimpleCurrentTraceContext` | `SimpleCurrentTraceContext` | `SimpleCurrentTraceContext.java:39` | Real propagates baggage from parent on scope entry; we skip (Feature 6). Real returns `Scope.NOOP` on null context; we properly save/restore. |
| `SimpleScopedSpan` | `SimpleScopedSpan` | `SimpleScopedSpan.java:29` | Real doesn't manage scope explicitly (`context()` returns detached context); we properly scope and un-scope |
| `SimpleSpanBuilder` | `SimpleSpanBuilder` | `SimpleSpanBuilder.java:38` | Real registers span in `start()` via `simpleTracer.getSpans().add(span)`; we use `registerSpan()` which also updates the context→span map |
| `SimpleSpanCustomizer` | `SimpleSpanCustomizer` | `SimpleSpanCustomizer.java:30` | Identical structure — delegates to `tracer.currentSpan()` |
| `SimpleSpanInScope` | `SimpleSpanInScope` | `SimpleSpanInScope.java:31` | Real has backward-compatible deprecated constructor; we only have the `Scope`-wrapping constructor |
| `SimpleTraceContextBuilder` | `SimpleTraceContextBuilder` | `SimpleTraceContextBuilder.java:29` | Identical structure |

## 3.13 Complete Code

### Production Code

#### File: `src/main/java/dev/linhvu/tracing/simple/SimpleTraceContext.java` [NEW]

```java
package dev.linhvu.tracing.simple;

import dev.linhvu.tracing.TraceContext;

import java.util.concurrent.ThreadLocalRandom;

/**
 * A mutable, in-memory implementation of {@link TraceContext}.
 * Generates random 16-character lowercase hex IDs for trace/span identification.
 *
 * <p>Unlike the interface (which is read-only), this implementation exposes setters
 * so the tracer can configure IDs when creating parent-child relationships.
 */
public class SimpleTraceContext implements TraceContext {

    private volatile String traceId = "";

    private volatile String parentId = "";

    private volatile String spanId = "";

    private volatile Boolean sampled = true;

    @Override
    public String traceId() {
        return traceId;
    }

    @Override
    public String parentId() {
        return parentId;
    }

    @Override
    public String spanId() {
        return spanId;
    }

    @Override
    public Boolean sampled() {
        return sampled;
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
     * Generates a random 16-character lowercase hex ID.
     * Simplified from the real framework's {@code EncodingUtils.fromLong(random.nextLong())}.
     */
    String generateId() {
        return String.format("%016x", ThreadLocalRandom.current().nextLong());
    }

    @Override
    public String toString() {
        return "SimpleTraceContext{traceId='" + traceId + "', parentId='" + parentId
                + "', spanId='" + spanId + "', sampled=" + sampled + "}";
    }

}
```

#### File: `src/main/java/dev/linhvu/tracing/simple/SimpleTraceContextBuilder.java` [NEW]

```java
package dev.linhvu.tracing.simple;

import dev.linhvu.tracing.TraceContext;

/**
 * Builds {@link SimpleTraceContext} instances from individual fields.
 * Used by {@link SimpleTracer#traceContextBuilder()} and in propagation
 * scenarios where a context is reconstructed from extracted headers.
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
        SimpleTraceContext ctx = new SimpleTraceContext();
        if (traceId != null) {
            ctx.setTraceId(traceId);
        }
        if (spanId != null) {
            ctx.setSpanId(spanId);
        }
        if (parentId != null) {
            ctx.setParentId(parentId);
        }
        if (sampled != null) {
            ctx.setSampled(sampled);
        }
        return ctx;
    }

}
```

#### File: `src/main/java/dev/linhvu/tracing/simple/SimpleSpan.java` [NEW]

```java
package dev.linhvu.tracing.simple;

import dev.linhvu.tracing.Span;

import java.time.Instant;
import java.util.AbstractMap;
import java.util.Collection;
import java.util.Collections;
import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.ConcurrentLinkedQueue;
import java.util.concurrent.TimeUnit;

/**
 * An in-memory implementation of {@link Span} that records all data for later inspection.
 * Stores name, tags, events, errors, timestamps, kind, and remote service metadata.
 *
 * <p>Each SimpleSpan owns a {@link SimpleTraceContext} that holds its trace/span/parent IDs.
 * The tracer configures these IDs when creating the span.
 */
public class SimpleSpan implements Span {

    private final Map<String, String> tags = new ConcurrentHashMap<>();

    private final ConcurrentLinkedQueue<Map.Entry<Long, String>> events = new ConcurrentLinkedQueue<>();

    private final SimpleTraceContext context = new SimpleTraceContext();

    private volatile String name;

    private volatile long startMillis;

    private volatile long endMillis;

    private volatile Throwable error;

    private volatile Kind spanKind;

    private volatile String remoteServiceName;

    private volatile String remoteIp;

    private volatile int remotePort;

    private volatile boolean abandoned;

    @Override
    public boolean isNoop() {
        return false;
    }

    @Override
    public SimpleTraceContext context() {
        return context;
    }

    @Override
    public SimpleSpan start() {
        this.startMillis = System.currentTimeMillis();
        return this;
    }

    @Override
    public SimpleSpan name(String name) {
        this.name = name;
        return this;
    }

    @Override
    public SimpleSpan tag(String key, String value) {
        this.tags.put(key, value);
        return this;
    }

    @Override
    public SimpleSpan event(String value) {
        this.events.add(new AbstractMap.SimpleEntry<>(
                TimeUnit.MILLISECONDS.toMicros(System.currentTimeMillis()), value));
        return this;
    }

    @Override
    public SimpleSpan event(String value, long time, TimeUnit timeUnit) {
        this.events.add(new AbstractMap.SimpleEntry<>(timeUnit.toMicros(time), value));
        return this;
    }

    @Override
    public SimpleSpan error(Throwable throwable) {
        this.error = throwable;
        return this;
    }

    @Override
    public void end() {
        this.endMillis = System.currentTimeMillis();
    }

    @Override
    public void end(long time, TimeUnit timeUnit) {
        this.endMillis = timeUnit.toMillis(time);
    }

    @Override
    public void abandon() {
        this.abandoned = true;
    }

    @Override
    public SimpleSpan remoteServiceName(String remoteServiceName) {
        this.remoteServiceName = remoteServiceName;
        return this;
    }

    @Override
    public SimpleSpan remoteIpAndPort(String ip, int port) {
        this.remoteIp = ip;
        this.remotePort = port;
        return this;
    }

    // --- Accessors for test inspection ---

    public String getName() {
        return name;
    }

    public Map<String, String> getTags() {
        return Collections.unmodifiableMap(tags);
    }

    public Collection<Map.Entry<Long, String>> getEvents() {
        return Collections.unmodifiableCollection(events);
    }

    public Throwable getError() {
        return error;
    }

    public Kind getKind() {
        return spanKind;
    }

    public String getRemoteServiceName() {
        return remoteServiceName;
    }

    public String getRemoteIp() {
        return remoteIp;
    }

    public int getRemotePort() {
        return remotePort;
    }

    public Instant getStartTimestamp() {
        return Instant.ofEpochMilli(startMillis);
    }

    public Instant getEndTimestamp() {
        return Instant.ofEpochMilli(endMillis);
    }

    public boolean isAbandoned() {
        return abandoned;
    }

    void setSpanKind(Kind kind) {
        this.spanKind = kind;
    }

    void setStartMillis(long startMillis) {
        this.startMillis = startMillis;
    }

    @Override
    public String toString() {
        return "SimpleSpan{name='" + name + "', traceId='" + context.traceId()
                + "', spanId='" + context.spanId() + "', parentId='" + context.parentId() + "'}";
    }

}
```

#### File: `src/main/java/dev/linhvu/tracing/simple/SimpleCurrentTraceContext.java` [NEW]

```java
package dev.linhvu.tracing.simple;

import dev.linhvu.tracing.CurrentTraceContext;
import dev.linhvu.tracing.Span;
import dev.linhvu.tracing.TraceContext;

import java.util.Objects;
import java.util.concurrent.Callable;
import java.util.concurrent.Executor;

/**
 * A ThreadLocal-based implementation of {@link CurrentTraceContext}.
 * Manages which {@link TraceContext} is "current" on the calling thread
 * by delegating to the tracer's ThreadLocal span storage.
 *
 * <p>{@link #newScope(TraceContext)} saves the previous span, sets the new one,
 * and returns a {@link Scope} that restores the previous on close — forming
 * a stack of scopes that supports nesting.
 */
public class SimpleCurrentTraceContext implements CurrentTraceContext {

    private final SimpleTracer tracer;

    SimpleCurrentTraceContext(SimpleTracer tracer) {
        this.tracer = tracer;
    }

    @Override
    public TraceContext context() {
        Span span = tracer.currentSpan();
        return span != null ? span.context() : null;
    }

    @Override
    public Scope newScope(TraceContext context) {
        SimpleSpan previous = tracer.getCurrentSpanInternal();

        if (context != null) {
            tracer.setCurrentSpan(context);
        }
        else {
            tracer.resetCurrentSpan();
        }

        // Always restore previous state on close — even when clearing to null
        return previous != null
                ? () -> tracer.setCurrentSpanDirect(previous)
                : () -> tracer.resetCurrentSpan();
    }

    @Override
    public Scope maybeScope(TraceContext context) {
        SimpleSpan current = tracer.getCurrentSpanInternal();

        if (context == null) {
            if (current == null) {
                return Scope.NOOP;
            }
            return newScope(null);
        }

        if (current != null && Objects.equals(current.context(), context)) {
            return Scope.NOOP;
        }
        return newScope(context);
    }

    @Override
    public <C> Callable<C> wrap(Callable<C> task) {
        return task;
    }

    @Override
    public Runnable wrap(Runnable task) {
        return task;
    }

    @Override
    public Executor wrap(Executor delegate) {
        return delegate;
    }

}
```

#### File: `src/main/java/dev/linhvu/tracing/simple/SimpleTracer.java` [NEW]

```java
package dev.linhvu.tracing.simple;

import dev.linhvu.tracing.Baggage;
import dev.linhvu.tracing.BaggageInScope;
import dev.linhvu.tracing.ScopedSpan;
import dev.linhvu.tracing.Span;
import dev.linhvu.tracing.SpanCustomizer;
import dev.linhvu.tracing.TraceContext;
import dev.linhvu.tracing.Tracer;

import java.util.Collections;
import java.util.Deque;
import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.LinkedBlockingDeque;

/**
 * An in-memory implementation of {@link Tracer} that creates real spans with
 * generated hex IDs, manages scope via ThreadLocal, and collects all created
 * spans in a deque for test inspection.
 *
 * <p>Key design decisions:
 * <ul>
 *   <li><strong>Instance-based state</strong> — unlike the real framework's static fields,
 *       all state is per-instance so tests can create isolated tracers without interference.</li>
 *   <li><strong>TraceContext→Span map</strong> — when {@code newScope(context)} is called,
 *       the tracer looks up which SimpleSpan owns that context to make it "current".</li>
 *   <li><strong>Baggage stubs</strong> — all BaggageManager methods return NOOP until
 *       Feature 6 provides a real implementation.</li>
 * </ul>
 */
public class SimpleTracer implements Tracer {

    private final Map<TraceContext, SimpleSpan> contextToSpan = new ConcurrentHashMap<>();

    private final ThreadLocal<SimpleSpan> currentSpan = new ThreadLocal<>();

    private final SimpleCurrentTraceContext currentTraceContext;

    private final Deque<SimpleSpan> spans = new LinkedBlockingDeque<>();

    public SimpleTracer() {
        this.currentTraceContext = new SimpleCurrentTraceContext(this);
    }

    @Override
    public SimpleSpan nextSpan() {
        SimpleSpan span = createSpan(currentSpan());
        return span;
    }

    @Override
    public SimpleSpan nextSpan(Span parent) {
        SimpleSpan span = createSpan(parent);
        return span;
    }

    private SimpleSpan createSpan(Span parent) {
        SimpleSpan span = new SimpleSpan();
        SimpleTraceContext ctx = span.context();

        if (parent != null) {
            // Child span: inherit traceId, record parent's spanId
            ctx.setTraceId(parent.context().traceId());
            ctx.setParentId(parent.context().spanId());
            ctx.setSpanId(ctx.generateId());
        }
        else {
            // Root span: traceId = spanId (new trace)
            String id = ctx.generateId();
            ctx.setTraceId(id);
            ctx.setSpanId(id);
        }

        contextToSpan.put(ctx, span);
        spans.add(span);
        return span;
    }

    @Override
    public SimpleSpanInScope withSpan(Span span) {
        return new SimpleSpanInScope(
                currentTraceContext.newScope(span != null ? span.context() : null));
    }

    @Override
    public ScopedSpan startScopedSpan(String name) {
        return new SimpleScopedSpan(this).name(name);
    }

    @Override
    public SimpleSpanBuilder spanBuilder() {
        return new SimpleSpanBuilder(this);
    }

    @Override
    public TraceContext.Builder traceContextBuilder() {
        return new SimpleTraceContextBuilder();
    }

    @Override
    public SimpleCurrentTraceContext currentTraceContext() {
        return currentTraceContext;
    }

    @Override
    public SpanCustomizer currentSpanCustomizer() {
        return new SimpleSpanCustomizer(this);
    }

    @Override
    public SimpleSpan currentSpan() {
        return currentSpan.get();
    }

    // --- BaggageManager (NOOP stubs until Feature 6) ---

    @Override
    public Map<String, String> getAllBaggage() {
        return Collections.emptyMap();
    }

    @Override
    public Baggage getBaggage(String name) {
        return Baggage.NOOP;
    }

    @Override
    public Baggage getBaggage(TraceContext traceContext, String name) {
        return Baggage.NOOP;
    }

    @Override
    public Baggage createBaggage(String name) {
        return Baggage.NOOP;
    }

    @Override
    public Baggage createBaggage(String name, String value) {
        return Baggage.NOOP;
    }

    // --- Convenience methods for test inspection ---

    /**
     * Returns all created spans (in creation order).
     */
    public Deque<SimpleSpan> getSpans() {
        return spans;
    }

    /**
     * Returns the single created span. Asserts exactly one span exists and
     * that it has been started and ended.
     */
    public SimpleSpan onlySpan() {
        if (spans.size() != 1) {
            throw new AssertionError("Expected 1 span, got " + spans.size());
        }
        SimpleSpan span = spans.getFirst();
        if (span.getStartTimestamp().toEpochMilli() == 0) {
            throw new AssertionError("Span must be started");
        }
        if (span.getEndTimestamp().toEpochMilli() == 0) {
            throw new AssertionError("Span must be finished");
        }
        return span;
    }

    /**
     * Returns the most recently created span.
     */
    public SimpleSpan lastSpan() {
        if (spans.isEmpty()) {
            throw new AssertionError("No spans created");
        }
        return spans.getLast();
    }

    // --- Package-private scope management (used by SimpleCurrentTraceContext) ---

    SimpleSpan getCurrentSpanInternal() {
        return currentSpan.get();
    }

    void setCurrentSpanDirect(SimpleSpan span) {
        currentSpan.set(span);
    }

    void resetCurrentSpan() {
        currentSpan.remove();
    }

    void setCurrentSpan(TraceContext context) {
        SimpleSpan span = contextToSpan.get(context);
        if (span != null) {
            currentSpan.set(span);
        }
        else {
            currentSpan.remove();
        }
    }

    void registerSpan(SimpleSpan span) {
        contextToSpan.put(span.context(), span);
        spans.add(span);
    }

}
```

#### File: `src/main/java/dev/linhvu/tracing/simple/SimpleSpanInScope.java` [NEW]

```java
package dev.linhvu.tracing.simple;

import dev.linhvu.tracing.CurrentTraceContext;
import dev.linhvu.tracing.Tracer;

/**
 * A simple wrapper around {@link CurrentTraceContext.Scope} that implements
 * {@link Tracer.SpanInScope}. Delegates close() to the underlying scope.
 */
public class SimpleSpanInScope implements Tracer.SpanInScope {

    private final CurrentTraceContext.Scope scope;

    SimpleSpanInScope(CurrentTraceContext.Scope scope) {
        this.scope = scope;
    }

    @Override
    public void close() {
        scope.close();
    }

}
```

#### File: `src/main/java/dev/linhvu/tracing/simple/SimpleSpanCustomizer.java` [NEW]

```java
package dev.linhvu.tracing.simple;

import dev.linhvu.tracing.Span;
import dev.linhvu.tracing.SpanCustomizer;

/**
 * Delegates all customization calls to the tracer's current span.
 * Throws {@link IllegalStateException} if no span is in scope.
 */
public class SimpleSpanCustomizer implements SpanCustomizer {

    private final SimpleTracer tracer;

    SimpleSpanCustomizer(SimpleTracer tracer) {
        this.tracer = tracer;
    }

    private Span currentSpan() {
        Span span = tracer.currentSpan();
        if (span == null) {
            throw new IllegalStateException("No current span in scope");
        }
        return span;
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

}
```

#### File: `src/main/java/dev/linhvu/tracing/simple/SimpleScopedSpan.java` [NEW]

```java
package dev.linhvu.tracing.simple;

import dev.linhvu.tracing.ScopedSpan;
import dev.linhvu.tracing.TraceContext;
import dev.linhvu.tracing.Tracer;

/**
 * A span that is automatically started and placed in scope on creation.
 * When {@link #end()} is called, the scope is closed (previous context restored)
 * and the span is finished — matching the {@link ScopedSpan} contract.
 *
 * <p>This is the convenience pattern: create → use → end, with no manual scope management.
 */
public class SimpleScopedSpan implements ScopedSpan {

    private final SimpleSpan span;

    private final Tracer.SpanInScope scope;

    SimpleScopedSpan(SimpleTracer tracer) {
        this.span = tracer.nextSpan();
        this.span.start();
        this.scope = tracer.withSpan(span);
    }

    @Override
    public boolean isNoop() {
        return false;
    }

    @Override
    public TraceContext context() {
        return span.context();
    }

    @Override
    public ScopedSpan name(String name) {
        span.name(name);
        return this;
    }

    @Override
    public ScopedSpan tag(String key, String value) {
        span.tag(key, value);
        return this;
    }

    @Override
    public ScopedSpan event(String value) {
        span.event(value);
        return this;
    }

    @Override
    public ScopedSpan error(Throwable throwable) {
        span.error(throwable);
        return this;
    }

    @Override
    public void end() {
        scope.close();
        span.end();
    }

}
```

#### File: `src/main/java/dev/linhvu/tracing/simple/SimpleSpanBuilder.java` [NEW]

```java
package dev.linhvu.tracing.simple;

import dev.linhvu.tracing.Link;
import dev.linhvu.tracing.Span;
import dev.linhvu.tracing.TraceContext;

import java.util.ArrayList;
import java.util.HashMap;
import java.util.List;
import java.util.Map;
import java.util.concurrent.TimeUnit;

/**
 * Accumulates span configuration (name, tags, kind, parent, etc.) and creates
 * a fully configured {@link SimpleSpan} on {@link #start()}.
 *
 * <p>If a parent context is set via {@link #setParent(TraceContext)}, the new span
 * inherits the parent's traceId and records the parent's spanId as its parentId.
 * Otherwise, a new root trace is created.
 */
public class SimpleSpanBuilder implements Span.Builder {

    private final SimpleTracer tracer;

    private final List<String> events = new ArrayList<>();

    private final Map<String, String> tags = new HashMap<>();

    private final List<Link> links = new ArrayList<>();

    private Throwable error;

    private String remoteServiceName;

    private Span.Kind spanKind;

    private String name;

    private String ip;

    private int port;

    private long startTimestamp;

    private TimeUnit startTimestampUnit;

    private TraceContext parentContext;

    SimpleSpanBuilder(SimpleTracer tracer) {
        this.tracer = tracer;
    }

    @Override
    public Span.Builder setParent(TraceContext context) {
        this.parentContext = context;
        return this;
    }

    @Override
    public Span.Builder setNoParent() {
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
        this.error = throwable;
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
    public Span.Builder remoteIpAndPort(String ip, int port) {
        this.ip = ip;
        this.port = port;
        return this;
    }

    @Override
    public Span.Builder startTimestamp(long startTimestamp, TimeUnit unit) {
        this.startTimestamp = startTimestamp;
        this.startTimestampUnit = unit;
        return this;
    }

    @Override
    public Span.Builder addLink(Link link) {
        this.links.add(link);
        return this;
    }

    @Override
    public SimpleSpan start() {
        SimpleSpan span = new SimpleSpan();

        // Apply accumulated configuration
        tags.forEach(span::tag);
        events.forEach(span::event);
        if (name != null) {
            span.name(name);
        }
        if (error != null) {
            span.error(error);
        }
        if (spanKind != null) {
            span.setSpanKind(spanKind);
        }
        if (remoteServiceName != null) {
            span.remoteServiceName(remoteServiceName);
        }
        if (ip != null) {
            span.remoteIpAndPort(ip, port);
        }

        // Set up trace context (parent-child IDs)
        SimpleTraceContext ctx = span.context();
        if (parentContext != null) {
            ctx.setTraceId(parentContext.traceId());
            ctx.setParentId(parentContext.spanId());
            ctx.setSpanId(ctx.generateId());
            ctx.setSampled(parentContext.sampled());
        }
        else {
            String id = ctx.generateId();
            ctx.setTraceId(id);
            ctx.setSpanId(id);
        }

        // Start the span
        span.start();
        if (startTimestampUnit != null) {
            span.setStartMillis(startTimestampUnit.toMillis(startTimestamp));
        }

        // Register with tracer
        tracer.registerSpan(span);
        return span;
    }

}
```

### Test Code

#### File: `src/test/java/dev/linhvu/tracing/simple/SimpleTraceContextTest.java` [NEW]

```java
package dev.linhvu.tracing.simple;

import org.junit.jupiter.api.Test;

import java.util.HashSet;
import java.util.Set;

import static org.assertj.core.api.Assertions.assertThat;

class SimpleTraceContextTest {

    @Test
    void shouldGenerateHexId_WhenCallingGenerateId() {
        SimpleTraceContext ctx = new SimpleTraceContext();
        String id = ctx.generateId();
        assertThat(id).hasSize(16).matches("[0-9a-f]{16}");
    }

    @Test
    void shouldGenerateUniqueIds_WhenCalledMultipleTimes() {
        SimpleTraceContext ctx = new SimpleTraceContext();
        Set<String> ids = new HashSet<>();
        for (int i = 0; i < 100; i++) {
            ids.add(ctx.generateId());
        }
        assertThat(ids).hasSizeGreaterThan(95);
    }

    @Test
    void shouldStoreTraceId_WhenSet() {
        SimpleTraceContext ctx = new SimpleTraceContext();
        ctx.setTraceId("abc123def456");
        assertThat(ctx.traceId()).isEqualTo("abc123def456");
    }

    @Test
    void shouldStoreSpanId_WhenSet() {
        SimpleTraceContext ctx = new SimpleTraceContext();
        ctx.setSpanId("span-001");
        assertThat(ctx.spanId()).isEqualTo("span-001");
    }

    @Test
    void shouldStoreParentId_WhenSet() {
        SimpleTraceContext ctx = new SimpleTraceContext();
        ctx.setParentId("parent-001");
        assertThat(ctx.parentId()).isEqualTo("parent-001");
    }

    @Test
    void shouldStoreSampled_WhenSet() {
        SimpleTraceContext ctx = new SimpleTraceContext();
        ctx.setSampled(false);
        assertThat(ctx.sampled()).isFalse();

        ctx.setSampled(null);
        assertThat(ctx.sampled()).isNull();
    }

    @Test
    void shouldHaveEmptyDefaults_WhenCreated() {
        SimpleTraceContext ctx = new SimpleTraceContext();
        assertThat(ctx.traceId()).isEmpty();
        assertThat(ctx.spanId()).isEmpty();
        assertThat(ctx.parentId()).isEmpty();
        assertThat(ctx.sampled()).isTrue();
    }

    @Test
    void shouldBuildContext_WhenUsingBuilder() {
        SimpleTraceContextBuilder builder = new SimpleTraceContextBuilder();
        var ctx = builder.traceId("trace-1")
                .spanId("span-1")
                .parentId("parent-1")
                .sampled(true)
                .build();

        assertThat(ctx.traceId()).isEqualTo("trace-1");
        assertThat(ctx.spanId()).isEqualTo("span-1");
        assertThat(ctx.parentId()).isEqualTo("parent-1");
        assertThat(ctx.sampled()).isTrue();
    }

    @Test
    void shouldBuildPartialContext_WhenOnlySomeFieldsSet() {
        SimpleTraceContextBuilder builder = new SimpleTraceContextBuilder();
        var ctx = (SimpleTraceContext) builder.traceId("trace-1").build();

        assertThat(ctx.traceId()).isEqualTo("trace-1");
        assertThat(ctx.spanId()).isEmpty();
        assertThat(ctx.parentId()).isEmpty();
    }

}
```

#### File: `src/test/java/dev/linhvu/tracing/simple/SimpleSpanTest.java` [NEW]

```java
package dev.linhvu.tracing.simple;

import org.junit.jupiter.api.Test;

import java.time.Instant;
import java.util.concurrent.TimeUnit;

import static org.assertj.core.api.Assertions.assertThat;

class SimpleSpanTest {

    @Test
    void shouldRecordName_WhenNameSet() {
        SimpleSpan span = new SimpleSpan();
        span.name("test-operation");
        assertThat(span.getName()).isEqualTo("test-operation");
    }

    @Test
    void shouldRecordTags_WhenTagsAdded() {
        SimpleSpan span = new SimpleSpan();
        span.tag("http.method", "GET");
        span.tag("http.status", "200");

        assertThat(span.getTags())
                .containsEntry("http.method", "GET")
                .containsEntry("http.status", "200");
    }

    @Test
    void shouldRecordEvents_WhenEventsAdded() {
        SimpleSpan span = new SimpleSpan();
        span.event("request.started");
        span.event("response.received");

        assertThat(span.getEvents()).hasSize(2);
        assertThat(span.getEvents())
                .extracting(e -> e.getValue())
                .containsExactly("request.started", "response.received");
    }

    @Test
    void shouldRecordTimestampedEvent_WhenEventWithTimeAdded() {
        SimpleSpan span = new SimpleSpan();
        span.event("custom", 1000L, TimeUnit.MILLISECONDS);

        assertThat(span.getEvents()).hasSize(1);
        var event = span.getEvents().iterator().next();
        assertThat(event.getKey()).isEqualTo(1000000L);
        assertThat(event.getValue()).isEqualTo("custom");
    }

    @Test
    void shouldRecordError_WhenErrorSet() {
        SimpleSpan span = new SimpleSpan();
        RuntimeException ex = new RuntimeException("oops");
        span.error(ex);

        assertThat(span.getError()).isSameAs(ex);
    }

    @Test
    void shouldRecordStartTimestamp_WhenStarted() {
        Instant before = Instant.ofEpochMilli(System.currentTimeMillis());
        SimpleSpan span = new SimpleSpan();
        span.start();
        Instant after = Instant.ofEpochMilli(System.currentTimeMillis());

        assertThat(span.getStartTimestamp())
                .isAfterOrEqualTo(before)
                .isBeforeOrEqualTo(after);
    }

    @Test
    void shouldRecordEndTimestamp_WhenEnded() {
        SimpleSpan span = new SimpleSpan();
        span.start();
        Instant before = Instant.ofEpochMilli(System.currentTimeMillis());
        span.end();
        Instant after = Instant.ofEpochMilli(System.currentTimeMillis());

        assertThat(span.getEndTimestamp())
                .isAfterOrEqualTo(before)
                .isBeforeOrEqualTo(after);
    }

    @Test
    void shouldRecordExplicitEndTimestamp_WhenEndedWithTime() {
        SimpleSpan span = new SimpleSpan();
        span.start();
        span.end(5000L, TimeUnit.MILLISECONDS);

        assertThat(span.getEndTimestamp()).isEqualTo(Instant.ofEpochMilli(5000));
    }

    @Test
    void shouldSetRemoteService_WhenConfigured() {
        SimpleSpan span = new SimpleSpan();
        span.remoteServiceName("user-service");
        span.remoteIpAndPort("10.0.0.1", 8080);

        assertThat(span.getRemoteServiceName()).isEqualTo("user-service");
        assertThat(span.getRemoteIp()).isEqualTo("10.0.0.1");
        assertThat(span.getRemotePort()).isEqualTo(8080);
    }

    @Test
    void shouldMarkAbandoned_WhenAbandoned() {
        SimpleSpan span = new SimpleSpan();
        assertThat(span.isAbandoned()).isFalse();

        span.abandon();
        assertThat(span.isAbandoned()).isTrue();
    }

    @Test
    void shouldNotBeNoop_WhenCreated() {
        SimpleSpan span = new SimpleSpan();
        assertThat(span.isNoop()).isFalse();
    }

    @Test
    void shouldReturnFluentSelf_WhenChaining() {
        SimpleSpan span = new SimpleSpan();
        SimpleSpan result = span.name("op").tag("k", "v").event("e").error(new RuntimeException());
        assertThat(result).isSameAs(span);
    }

}
```

#### File: `src/test/java/dev/linhvu/tracing/simple/SimpleTracerTest.java` [NEW]

```java
package dev.linhvu.tracing.simple;

import dev.linhvu.tracing.CurrentTraceContext;
import dev.linhvu.tracing.ScopedSpan;
import dev.linhvu.tracing.Span;
import dev.linhvu.tracing.SpanCustomizer;
import dev.linhvu.tracing.Tracer;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Nested;
import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

class SimpleTracerTest {

    SimpleTracer tracer;

    @BeforeEach
    void setUp() {
        tracer = new SimpleTracer();
    }

    @Nested
    class SpanCreation {

        @Test
        void shouldCreateRootSpan_WhenNoCurrentSpan() {
            SimpleSpan span = tracer.nextSpan();
            assertThat(span.context().traceId()).isNotEmpty();
            assertThat(span.context().spanId()).isNotEmpty();
            assertThat(span.context().parentId()).isEmpty();
            assertThat(span.context().traceId()).isEqualTo(span.context().spanId());
        }

        @Test
        void shouldCreateChildSpan_WhenParentInScope() {
            SimpleSpan root = tracer.nextSpan().name("root").start();
            try (Tracer.SpanInScope ws = tracer.withSpan(root)) {
                SimpleSpan child = tracer.nextSpan().name("child");
                assertThat(child.context().traceId()).isEqualTo(root.context().traceId());
                assertThat(child.context().parentId()).isEqualTo(root.context().spanId());
                assertThat(child.context().spanId()).isNotEqualTo(root.context().spanId());
            }
        }

        @Test
        void shouldCreateChildSpan_WhenExplicitParentProvided() {
            SimpleSpan parent = tracer.nextSpan().name("parent").start();
            SimpleSpan child = tracer.nextSpan(parent).name("child");
            assertThat(child.context().traceId()).isEqualTo(parent.context().traceId());
            assertThat(child.context().parentId()).isEqualTo(parent.context().spanId());
        }

        @Test
        void shouldCollectAllCreatedSpans() {
            tracer.nextSpan().name("span-1");
            tracer.nextSpan().name("span-2");
            tracer.nextSpan().name("span-3");
            assertThat(tracer.getSpans()).hasSize(3);
        }

        @Test
        void shouldGenerateValidHexIds() {
            SimpleSpan span = tracer.nextSpan();
            assertThat(span.context().traceId()).matches("[0-9a-f]{16}");
            assertThat(span.context().spanId()).matches("[0-9a-f]{16}");
        }
    }

    @Nested
    class ScopeManagement {

        @Test
        void shouldReturnNullCurrentSpan_WhenNoScope() {
            assertThat(tracer.currentSpan()).isNull();
        }

        @Test
        void shouldTrackCurrentSpan_WhenInScope() {
            SimpleSpan span = tracer.nextSpan().start();
            try (Tracer.SpanInScope ws = tracer.withSpan(span)) {
                assertThat(tracer.currentSpan()).isSameAs(span);
            }
        }

        @Test
        void shouldRestorePreviousSpan_WhenScopeClosed() {
            SimpleSpan outer = tracer.nextSpan().start();
            try (Tracer.SpanInScope ws1 = tracer.withSpan(outer)) {
                assertThat(tracer.currentSpan()).isSameAs(outer);
                SimpleSpan inner = tracer.nextSpan().start();
                try (Tracer.SpanInScope ws2 = tracer.withSpan(inner)) {
                    assertThat(tracer.currentSpan()).isSameAs(inner);
                }
                assertThat(tracer.currentSpan()).isSameAs(outer);
            }
            assertThat(tracer.currentSpan()).isNull();
        }

        @Test
        void shouldClearScope_WhenWithSpanCalledWithNull() {
            SimpleSpan span = tracer.nextSpan().start();
            try (Tracer.SpanInScope ws1 = tracer.withSpan(span)) {
                assertThat(tracer.currentSpan()).isSameAs(span);
                try (Tracer.SpanInScope ws2 = tracer.withSpan(null)) {
                    assertThat(tracer.currentSpan()).isNull();
                }
                assertThat(tracer.currentSpan()).isSameAs(span);
            }
        }

        @Test
        void shouldReturnContextFromCurrentTraceContext() {
            assertThat(tracer.currentTraceContext().context()).isNull();
            SimpleSpan span = tracer.nextSpan().start();
            try (Tracer.SpanInScope ws = tracer.withSpan(span)) {
                assertThat(tracer.currentTraceContext().context()).isSameAs(span.context());
            }
        }

        @Test
        void shouldSupportMaybeScope_WhenContextAlreadyCurrent() {
            SimpleSpan span = tracer.nextSpan().start();
            try (Tracer.SpanInScope ws = tracer.withSpan(span)) {
                CurrentTraceContext.Scope scope =
                        tracer.currentTraceContext().maybeScope(span.context());
                assertThat(scope).isSameAs(CurrentTraceContext.Scope.NOOP);
            }
        }
    }

    @Nested
    class ScopedSpanTests {

        @Test
        void shouldStartAndScope_WhenScopedSpanCreated() {
            ScopedSpan scoped = tracer.startScopedSpan("my-operation");
            assertThat(tracer.currentSpan()).isNotNull();
            assertThat(tracer.getSpans().getLast().getStartTimestamp().toEpochMilli())
                    .isGreaterThan(0);
            scoped.end();
            assertThat(tracer.currentSpan()).isNull();
            assertThat(tracer.getSpans().getLast().getEndTimestamp().toEpochMilli())
                    .isGreaterThan(0);
        }

        @Test
        void shouldRecordNameTagsEvents_WhenScopedSpanUsed() {
            ScopedSpan scoped = tracer.startScopedSpan("test-op");
            scoped.tag("key", "value");
            scoped.event("something.happened");
            scoped.end();
            SimpleSpan span = tracer.lastSpan();
            assertThat(span.getName()).isEqualTo("test-op");
            assertThat(span.getTags()).containsEntry("key", "value");
            assertThat(span.getEvents()).extracting(e -> e.getValue())
                    .contains("something.happened");
        }

        @Test
        void shouldRecordError_WhenScopedSpanHasError() {
            ScopedSpan scoped = tracer.startScopedSpan("failing-op");
            RuntimeException ex = new RuntimeException("fail");
            scoped.error(ex);
            scoped.end();
            assertThat(tracer.lastSpan().getError()).isSameAs(ex);
        }
    }

    @Nested
    class SpanBuilderTests {

        @Test
        void shouldBuildSpanWithConfiguration_WhenBuilderUsed() {
            Span span = tracer.spanBuilder()
                    .name("built-span").tag("env", "test")
                    .kind(Span.Kind.CLIENT).remoteServiceName("backend")
                    .start();
            SimpleSpan simple = (SimpleSpan) span;
            assertThat(simple.getName()).isEqualTo("built-span");
            assertThat(simple.getTags()).containsEntry("env", "test");
            assertThat(simple.getKind()).isEqualTo(Span.Kind.CLIENT);
            assertThat(simple.getRemoteServiceName()).isEqualTo("backend");
            assertThat(simple.getStartTimestamp().toEpochMilli()).isGreaterThan(0);
        }

        @Test
        void shouldBuildChildSpan_WhenParentContextSet() {
            SimpleSpan parent = tracer.nextSpan().name("parent").start();
            Span child = tracer.spanBuilder()
                    .setParent(parent.context()).name("child").start();
            assertThat(child.context().traceId()).isEqualTo(parent.context().traceId());
            assertThat(child.context().parentId()).isEqualTo(parent.context().spanId());
        }

        @Test
        void shouldBuildRootSpan_WhenNoParent() {
            Span span = tracer.spanBuilder().name("root").start();
            assertThat(span.context().parentId()).isEmpty();
            assertThat(span.context().traceId()).isEqualTo(span.context().spanId());
        }

        @Test
        void shouldRegisterWithTracer_WhenBuilderStartCalled() {
            tracer.spanBuilder().name("builder-span").start();
            assertThat(tracer.getSpans()).hasSize(1);
            assertThat(tracer.lastSpan().getName()).isEqualTo("builder-span");
        }
    }

    @Nested
    class SpanCustomizerTests {

        @Test
        void shouldDelegateToCurrentSpan_WhenInScope() {
            SimpleSpan span = tracer.nextSpan().start();
            try (Tracer.SpanInScope ws = tracer.withSpan(span)) {
                SpanCustomizer customizer = tracer.currentSpanCustomizer();
                customizer.name("customized");
                customizer.tag("custom-key", "custom-value");
                customizer.event("custom-event");
            }
            assertThat(span.getName()).isEqualTo("customized");
            assertThat(span.getTags()).containsEntry("custom-key", "custom-value");
        }

        @Test
        void shouldThrowException_WhenNoSpanInScope() {
            SpanCustomizer customizer = tracer.currentSpanCustomizer();
            assertThatThrownBy(() -> customizer.name("oops"))
                    .isInstanceOf(IllegalStateException.class)
                    .hasMessageContaining("No current span in scope");
        }
    }

    @Nested
    class InspectionMethods {

        @Test
        void shouldReturnOnlySpan_WhenExactlyOneCompletedSpan() {
            tracer.nextSpan().name("single").start().end();
            SimpleSpan span = tracer.onlySpan();
            assertThat(span.getName()).isEqualTo("single");
        }

        @Test
        void shouldThrow_WhenOnlySpanCalledWithMultipleSpans() {
            tracer.nextSpan().start().end();
            tracer.nextSpan().start().end();
            assertThatThrownBy(() -> tracer.onlySpan())
                    .isInstanceOf(AssertionError.class)
                    .hasMessageContaining("Expected 1 span, got 2");
        }

        @Test
        void shouldReturnLastSpan_WhenMultipleSpansExist() {
            tracer.nextSpan().name("first");
            tracer.nextSpan().name("last");
            assertThat(tracer.lastSpan().getName()).isEqualTo("last");
        }

        @Test
        void shouldThrow_WhenLastSpanCalledWithNoSpans() {
            assertThatThrownBy(() -> tracer.lastSpan())
                    .isInstanceOf(AssertionError.class)
                    .hasMessageContaining("No spans created");
        }
    }

}
```

#### File: `src/test/java/dev/linhvu/tracing/integration/SimpleTracerIntegrationTest.java` [NEW]

```java
package dev.linhvu.tracing.integration;

import dev.linhvu.tracing.ScopedSpan;
import dev.linhvu.tracing.Span;
import dev.linhvu.tracing.Tracer;
import dev.linhvu.tracing.simple.SimpleSpan;
import dev.linhvu.tracing.simple.SimpleTracer;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;

class SimpleTracerIntegrationTest {

    SimpleTracer tracer;

    @BeforeEach
    void setUp() {
        tracer = new SimpleTracer();
    }

    @Test
    void shouldMaintainTraceAcrossNestedSpans() {
        Span httpSpan = tracer.nextSpan().name("http-request").start();
        try (Tracer.SpanInScope ws1 = tracer.withSpan(httpSpan)) {
            httpSpan.tag("http.method", "GET");
            Span dbSpan = tracer.nextSpan().name("db-query").start();
            try (Tracer.SpanInScope ws2 = tracer.withSpan(dbSpan)) {
                dbSpan.tag("db.type", "postgresql");
                Span cacheSpan = tracer.nextSpan().name("cache-lookup").start();
                try (Tracer.SpanInScope ws3 = tracer.withSpan(cacheSpan)) {
                    cacheSpan.tag("cache.hit", "true");
                } finally {
                    cacheSpan.end();
                }
            } finally {
                dbSpan.end();
            }
        } finally {
            httpSpan.end();
        }

        assertThat(tracer.getSpans()).hasSize(3);
        String traceId = httpSpan.context().traceId();
        for (SimpleSpan span : tracer.getSpans()) {
            assertThat(span.context().traceId()).isEqualTo(traceId);
        }

        SimpleSpan http = tracer.getSpans().stream()
                .filter(s -> "http-request".equals(s.getName())).findFirst().orElseThrow();
        SimpleSpan db = tracer.getSpans().stream()
                .filter(s -> "db-query".equals(s.getName())).findFirst().orElseThrow();
        SimpleSpan cache = tracer.getSpans().stream()
                .filter(s -> "cache-lookup".equals(s.getName())).findFirst().orElseThrow();

        assertThat(http.context().parentId()).isEmpty();
        assertThat(db.context().parentId()).isEqualTo(http.context().spanId());
        assertThat(cache.context().parentId()).isEqualTo(db.context().spanId());
    }

    @Test
    void shouldSupportMixedScopePatterns() {
        Span explicitSpan = tracer.nextSpan().name("explicit").start();
        try (Tracer.SpanInScope ws = tracer.withSpan(explicitSpan)) {
            explicitSpan.tag("pattern", "explicit");
            ScopedSpan scopedSpan = tracer.startScopedSpan("scoped");
            scopedSpan.tag("pattern", "scoped");
            assertThat(tracer.currentSpan()).isNotSameAs(explicitSpan);
            scopedSpan.end();
            assertThat(tracer.currentSpan()).isSameAs(explicitSpan);
        } finally {
            explicitSpan.end();
        }
        assertThat(tracer.getSpans()).hasSize(2);
    }

    @Test
    void shouldBuildSpanWithParentFromDifferentTrace() {
        var extractedContext = tracer.traceContextBuilder()
                .traceId("aabbccddee112233").spanId("1122334455667788")
                .sampled(true).build();
        Span childSpan = tracer.spanBuilder()
                .setParent(extractedContext).name("continued-from-remote")
                .kind(Span.Kind.SERVER).start();
        SimpleSpan simple = (SimpleSpan) childSpan;
        assertThat(simple.context().traceId()).isEqualTo("aabbccddee112233");
        assertThat(simple.context().parentId()).isEqualTo("1122334455667788");
        assertThat(simple.context().spanId()).isNotEqualTo("1122334455667788");
        assertThat(simple.getKind()).isEqualTo(Span.Kind.SERVER);
    }

    @Test
    void shouldHandleErrorScenario() {
        Span span = tracer.nextSpan().name("failing-operation").start();
        try (Tracer.SpanInScope ws = tracer.withSpan(span)) {
            try {
                throw new RuntimeException("connection refused");
            } catch (Exception e) {
                span.error(e);
                span.tag("error.type", e.getClass().getSimpleName());
            }
        } finally {
            span.end();
        }
        SimpleSpan finished = tracer.onlySpan();
        assertThat(finished.getError()).isInstanceOf(RuntimeException.class);
        assertThat(finished.getError().getMessage()).isEqualTo("connection refused");
        assertThat(finished.getTags()).containsEntry("error.type", "RuntimeException");
    }

    @Test
    void shouldSupportAbandonedSpan() {
        Span span = tracer.nextSpan().name("maybe-operation").start();
        span.abandon();
        SimpleSpan simple = (SimpleSpan) span;
        assertThat(simple.isAbandoned()).isTrue();
        assertThat(tracer.getSpans()).hasSize(1);
    }

    @Test
    void shouldUseTracerThroughInterfaceType() {
        Tracer tracerInterface = tracer;
        Span span = tracerInterface.nextSpan().name("interface-test").start();
        try (Tracer.SpanInScope ws = tracerInterface.withSpan(span)) {
            assertThat(tracerInterface.currentSpan()).isNotNull();
            assertThat(tracerInterface.currentTraceContext().context()).isNotNull();
        } finally {
            span.end();
        }
        assertThat(tracerInterface.currentSpan()).isNull();
    }

}
```

---

## Summary

| Concept | What It Means |
|---------|---------------|
| **SimpleTracer** | In-memory `Tracer` implementation — creates spans with real IDs, manages scope via ThreadLocal, collects all spans for inspection |
| **SimpleSpan** | Concrete `Span` that records name, tags, events, errors, timestamps, kind, and remote service metadata |
| **SimpleTraceContext** | Mutable `TraceContext` with random hex ID generation and setters for parent-child configuration |
| **SimpleCurrentTraceContext** | ThreadLocal-based scope manager — `newScope()` forms an implicit stack with save/restore |
| **SimpleScopedSpan** | Creates, starts, and scopes a span in the constructor; `end()` closes scope and finishes the span |
| **SimpleSpanBuilder** | Accumulates configuration and creates a fully configured span on `start()` |
| **Instance-based state** | All state is per-tracer-instance (not static), so tests can create isolated tracers |
| **TraceContext→Span map** | Bridges `newScope(context)` to the right `SimpleSpan` — O(1) lookup via `ConcurrentHashMap` |
| **Root span convention** | Root spans use `traceId == spanId` with empty `parentId` — matches W3C Trace Context |

**Next: Chapter 4 — Span Export Pipeline** — post-processing for completed spans: filter, transform, and report finished spans to external systems via `FinishedSpan`, `SpanReporter`, `SpanFilter`, and `SpanExportingPredicate`
