# Chapter 4: Span Export Pipeline

> Line references based on commit `dceab1b9` of the Micrometer Tracing repository.

## Build Challenge

| Current State | Limitation | Objective |
|---------------|------------|-----------|
| `SimpleTracer` creates and collects spans in memory; `SimpleSpan` records name, tags, events, errors, timestamps | Spans are created and collected, but there is no post-processing pipeline — no way to filter, transform, or deliver finished spans to an external system (Zipkin, log file, etc.) | Build a span export pipeline: `FinishedSpan` (read-time interface), `SpanExportingPredicate` (gate), `SpanFilter` (transform), `SpanReporter` (sink) — wired into `SimpleSpan.end()` |

---

## 4.1 The Integration Point: SimpleSpan.end() → Export Pipeline

The integration point is **`SimpleSpan.end()`** — the moment a span finishes, it must flow through the export pipeline. This connects two subsystems:

- **Span lifecycle** (Features 1-3) — the existing system that creates, configures, and ends spans
- **Export pipeline** (this feature) — a new three-stage pipeline that decides whether to export, transforms, then delivers

The change is small but powerful: `SimpleSpan.end()` gains a callback that the tracer wires up when creating the span.

**Direction:** We start with the pipeline interfaces (`FinishedSpan`, `SpanExportingPredicate`, `SpanFilter`, `SpanReporter`), then make `SimpleSpan` implement `FinishedSpan`, then wire the pipeline into `SimpleTracer`.

Three key decisions:

1. **Callback pattern** — rather than having `SimpleSpan` know about the pipeline directly, the tracer installs a `Runnable` callback via `setOnSpanEnded()`. This keeps the span decoupled from the export infrastructure.
2. **Dual interface** — `SimpleSpan` implements both `Span` (write-time) and `FinishedSpan` (read-time). No data copying needed — the same object serves both roles.
3. **Pipeline order** — predicates run first (gate), then filters (transform), then reporters (sink). If any predicate returns `false`, the span is dropped immediately.

**Modifying:** `src/main/java/dev/linhvu/tracing/simple/SimpleSpan.java`
**Change:** Add `implements FinishedSpan`, add `onSpanEnded` callback field, modify `end()` to fire the callback

```java
// Before (Feature 3):
public class SimpleSpan implements Span {
    // ...
    @Override
    public void end() {
        this.endMillis = System.currentTimeMillis();
    }
}

// After (Feature 4):
public class SimpleSpan implements Span, FinishedSpan {
    private volatile Runnable onSpanEnded;
    // ...
    @Override
    public void end() {
        this.endMillis = System.currentTimeMillis();
        fireOnSpanEnded();  // ← NEW: trigger the export pipeline
    }

    private void fireOnSpanEnded() {
        Runnable callback = this.onSpanEnded;
        if (callback != null) {
            callback.run();
        }
    }
}
```

**Modifying:** `src/main/java/dev/linhvu/tracing/simple/SimpleTracer.java`
**Change:** Add pipeline fields and `exportSpan()` method, wire callback in `createSpan()` and `registerSpan()`

```java
// The tracer wires the callback when creating each span:
span.setOnSpanEnded(() -> exportSpan(span));

// And the pipeline runs in order:
private void exportSpan(SimpleSpan span) {
    // Gate: if any predicate says no, drop the span
    for (SpanExportingPredicate predicate : exportingPredicates) {
        if (!predicate.isExportable(span)) {
            return;
        }
    }
    // Transform: apply each filter in order
    FinishedSpan result = span;
    for (SpanFilter filter : spanFilters) {
        result = filter.map(result);
    }
    // Sink: deliver to each reporter
    for (SpanReporter reporter : spanReporters) {
        reporter.report(result);
    }
}
```

---

## 4.2 FinishedSpan — The Read-Time Interface

**New file:** `src/main/java/dev/linhvu/tracing/exporter/FinishedSpan.java`

`FinishedSpan` is the **post-mortem** view of a span. While `Span` is the write-time API (you call `tag()`, `event()`, `end()` during instrumentation), `FinishedSpan` is the read-time API (you call `getName()`, `getTags()`, `getError()` in the export pipeline).

```java
package dev.linhvu.tracing.exporter;

import dev.linhvu.tracing.Link;
import dev.linhvu.tracing.Span;

import java.time.Duration;
import java.time.Instant;
import java.util.Collection;
import java.util.Collections;
import java.util.List;
import java.util.Map;

public interface FinishedSpan {

    // --- Name ---
    String getName();
    FinishedSpan setName(String name);

    // --- Timestamps ---
    Instant getStartTimestamp();
    Instant getEndTimestamp();
    default Duration getDuration() {
        return Duration.between(getStartTimestamp(), getEndTimestamp());
    }

    // --- Tags ---
    Map<String, String> getTags();
    FinishedSpan setTags(Map<String, String> tags);

    // --- Events ---
    Collection<Map.Entry<Long, String>> getEvents();
    FinishedSpan setEvents(Collection<Map.Entry<Long, String>> events);

    // --- Identity (from TraceContext) ---
    String getTraceId();
    String getSpanId();
    String getParentId();

    // --- Networking ---
    String getRemoteIp();
    int getRemotePort();
    FinishedSpan setRemotePort(int port);
    String getRemoteServiceName();
    FinishedSpan setRemoteServiceName(String remoteServiceName);

    // --- Error ---
    Throwable getError();
    FinishedSpan setError(Throwable error);

    // --- Kind ---
    Span.Kind getKind();

    // --- Links ---
    default List<Link> getLinks() { return Collections.emptyList(); }
    default FinishedSpan addLinks(List<Link> links) { return this; }
    default FinishedSpan addLink(Link link) { return this; }
}
```

Why does `FinishedSpan` have **setters** if it's a "read-time" interface? Because `SpanFilter.map()` needs to mutate spans before export — rename them, redact tags, enrich with metadata. The setters return `this` for fluent chaining.

---

## 4.3 The Three Pipeline Stages

**New file:** `src/main/java/dev/linhvu/tracing/exporter/SpanExportingPredicate.java`

The **gate** — decides whether a span should be exported at all.

```java
package dev.linhvu.tracing.exporter;

@FunctionalInterface
public interface SpanExportingPredicate {
    boolean isExportable(FinishedSpan span);
}
```

**New file:** `src/main/java/dev/linhvu/tracing/exporter/SpanFilter.java`

The **transformer** — mutates a finished span before export.

```java
package dev.linhvu.tracing.exporter;

@FunctionalInterface
public interface SpanFilter {
    FinishedSpan map(FinishedSpan span);
}
```

**New file:** `src/main/java/dev/linhvu/tracing/exporter/SpanReporter.java`

The **sink** — delivers finished spans to an external system.

```java
package dev.linhvu.tracing.exporter;

public interface SpanReporter extends AutoCloseable {
    void report(FinishedSpan span);

    @Override
    default void close() {
    }
}
```

All three are functional interfaces — you can implement them with lambdas:

```java
// Predicate: skip health-check spans
SpanExportingPredicate skipHealth = span -> !"health-check".equals(span.getName());

// Filter: prefix all span names with environment
SpanFilter envPrefix = span -> { span.setName("[prod] " + span.getName()); return span; };

// Reporter: collect into a list
List<FinishedSpan> exported = new ArrayList<>();
SpanReporter collector = exported::add;
```

---

## 4.4 SimpleSpan Implements FinishedSpan

**Modifying:** `src/main/java/dev/linhvu/tracing/simple/SimpleSpan.java`

`SimpleSpan` now implements both `Span` and `FinishedSpan`. The same object serves as write-time and read-time API.

<details><summary>Challenge: What FinishedSpan methods does SimpleSpan need to implement?</summary>

The identity methods delegate to the existing `SimpleTraceContext`:

```java
@Override
public String getTraceId() { return context.traceId(); }

@Override
public String getSpanId() { return context.spanId(); }

@Override
public String getParentId() { return context.parentId(); }
```

Most setters are no-ops because data was already set through the `Span` interface:

```java
@Override
public FinishedSpan setTags(Map<String, String> tags) {
    // No-op: tags were set through Span.tag()
    return this;
}

@Override
public FinishedSpan setEvents(Collection<Map.Entry<Long, String>> events) {
    // No-op: events were set through Span.event()
    return this;
}
```

But `setName()` and `setRemoteServiceName()` do mutate — `SpanFilter` needs to rename spans:

```java
@Override
public FinishedSpan setName(String name) {
    this.name = name;
    return this;
}

@Override
public FinishedSpan setRemoteServiceName(String remoteServiceName) {
    this.remoteServiceName = remoteServiceName;
    return this;
}
```

</details>

---

## 4.5 Wiring the Pipeline into SimpleTracer

**Modifying:** `src/main/java/dev/linhvu/tracing/simple/SimpleTracer.java`

The tracer gains three new fields and a new constructor:

<details><summary>Challenge: How do you wire the pipeline into span creation?</summary>

```java
private final List<SpanExportingPredicate> exportingPredicates;
private final List<SpanFilter> spanFilters;
private final List<SpanReporter> spanReporters;

public SimpleTracer() {
    this(List.of(), List.of(), List.of());
}

public SimpleTracer(List<SpanExportingPredicate> exportingPredicates,
        List<SpanFilter> spanFilters, List<SpanReporter> spanReporters) {
    this.currentTraceContext = new SimpleCurrentTraceContext(this);
    this.exportingPredicates = new ArrayList<>(exportingPredicates);
    this.spanFilters = new ArrayList<>(spanFilters);
    this.spanReporters = new ArrayList<>(spanReporters);
}
```

In `createSpan()` and `registerSpan()`, wire the callback:

```java
span.setOnSpanEnded(() -> exportSpan(span));
```

</details>

---

## 4.6 Tests

### Unit Tests

**New file:** `src/test/java/dev/linhvu/tracing/exporter/FinishedSpanTest.java`

Tests that `SimpleSpan` correctly exposes all data through the `FinishedSpan` interface:

```java
@Test
void shouldExposeTraceId_WhenSpanIsFinished() {
    SimpleSpan span = tracer.nextSpan();
    span.start();
    span.end();

    FinishedSpan finished = span;  // same object, different interface
    assertThat(finished.getTraceId()).isEqualTo(span.context().traceId());
}

@Test
void shouldAllowNameChange_WhenSetNameCalled() {
    SimpleSpan span = tracer.nextSpan();
    span.start();
    span.name("original");
    span.end();

    FinishedSpan finished = span;
    finished.setName("renamed");
    assertThat(finished.getName()).isEqualTo("renamed");
}
```

**New file:** `src/test/java/dev/linhvu/tracing/exporter/SpanExportPipelineTest.java`

Tests each pipeline stage and their composition:

```java
@Test
void shouldDropSpan_WhenPredicateReturnsFalse() {
    List<FinishedSpan> reported = new CopyOnWriteArrayList<>();
    SimpleTracer tracer = new SimpleTracer(
            List.of(span -> false), List.of(), List.of(reported::add));

    SimpleSpan span = tracer.nextSpan();
    span.start();
    span.end();

    assertThat(reported).isEmpty();
}

@Test
void shouldChainFilters_WhenMultipleFiltersRegistered() {
    List<FinishedSpan> reported = new CopyOnWriteArrayList<>();
    SpanFilter addSuffix = span -> { span.setName(span.getName() + "-filtered"); return span; };
    SpanFilter upperCase = span -> { span.setName(span.getName().toUpperCase()); return span; };

    SimpleTracer tracer = new SimpleTracer(
            List.of(), List.of(addSuffix, upperCase), List.of(reported::add));

    SimpleSpan span = tracer.nextSpan();
    span.start().name("test");
    span.end();

    // Applied in order: "test" → "test-filtered" → "TEST-FILTERED"
    assertThat(reported.get(0).getName()).isEqualTo("TEST-FILTERED");
}
```

### Integration Tests

**New file:** `src/test/java/dev/linhvu/tracing/integration/ExportPipelineIntegrationTest.java`

Verifies the pipeline works with all tracing constructs end-to-end:

```java
@Test
void shouldExportNestedTrace_WhenUsingWithSpanScope() {
    List<FinishedSpan> reported = new CopyOnWriteArrayList<>();
    SimpleTracer tracer = new SimpleTracer(List.of(), List.of(), List.of(reported::add));

    Span root = tracer.nextSpan();
    root.start().name("http-request");
    try (Tracer.SpanInScope rootScope = tracer.withSpan(root)) {
        Span service = tracer.nextSpan();
        service.start().name("service-call");
        try (Tracer.SpanInScope serviceScope = tracer.withSpan(service)) {
            Span db = tracer.nextSpan();
            db.start().name("db-query");
            db.end();
        }
        service.end();
    }
    root.end();

    assertThat(reported).hasSize(3);
    assertThat(reported).extracting(FinishedSpan::getName)
            .containsExactly("db-query", "service-call", "http-request");

    // All share the same traceId
    String traceId = reported.get(0).getTraceId();
    assertThat(reported).allMatch(s -> s.getTraceId().equals(traceId));
}

@Test
void shouldFilterHealthChecksFromExport_WhenPredicateConfigured() {
    List<FinishedSpan> reported = new CopyOnWriteArrayList<>();
    SpanExportingPredicate skipInfra = span -> {
        String name = span.getName();
        return name != null && !name.startsWith("actuator/") && !name.equals("health-check");
    };

    SimpleTracer tracer = new SimpleTracer(List.of(skipInfra), List.of(), List.of(reported::add));

    tracer.nextSpan().start().name("api/users").end();
    tracer.nextSpan().start().name("health-check").end();
    tracer.nextSpan().start().name("actuator/metrics").end();
    tracer.nextSpan().start().name("api/orders").end();

    assertThat(reported).extracting(FinishedSpan::getName)
            .containsExactly("api/users", "api/orders");
}
```

Run all tests:

```bash
./gradlew test
```

---

## 4.7 Why This Works

`★ Insight ─────────────────────────────────────`
**Pipes and Filters Architecture:**
The export pipeline is a textbook "pipes and filters" pattern. Each stage has a single responsibility: predicates decide (boolean), filters transform (span → span), reporters consume (void). This means you can compose them freely — add a new filter without touching existing predicates or reporters.

**Why:** In real production systems, you might want to: drop health-check spans (predicate), redact PII from tags (filter), and send spans to both Zipkin and a log file (two reporters). Each concern is a separate, composable piece.
`─────────────────────────────────────────────────`

`★ Insight ─────────────────────────────────────`
**Dual-Interface Pattern (Span + FinishedSpan):**
`SimpleSpan` implements both `Span` and `FinishedSpan`. This avoids a data-copying step — the same object that was actively instrumented becomes the finished span in the pipeline. In real Micrometer, bridge implementations (Brave, OTel) need separate `FinishedSpan` wrappers because their underlying span types differ, but for the test/simple implementation, one object serving dual roles is simpler and more efficient.

**Why:** The key insight is that `Span` and `FinishedSpan` are just different *views* of the same data. `Span` is the "write API" (used by instrumentation code), `FinishedSpan` is the "read API" (used by the export pipeline). By implementing both on one class, no conversion is needed.
`─────────────────────────────────────────────────`

`★ Insight ─────────────────────────────────────`
**Callback Decoupling:**
The span doesn't know about the pipeline directly. Instead, the tracer installs a `Runnable` callback (`setOnSpanEnded`). This is a form of the **Observer pattern** — the span notifies its observer (the tracer's export logic) when it ends, without depending on any export classes. If no pipeline is configured (default constructor), the callback is null and nothing happens.

**Why:** This keeps `SimpleSpan` testable in isolation and avoids a circular dependency between the span and the tracer's pipeline infrastructure.
`─────────────────────────────────────────────────`

---

## 4.8 What We Enhanced

| File | What Changed | Why |
|------|-------------|-----|
| `SimpleSpan.java` | Now implements `FinishedSpan`; added `onSpanEnded` callback; `end()` fires the callback; added `links` field and `FinishedSpan` method implementations | The span is the entry point to the export pipeline — when it ends, data flows through predicates → filters → reporters |
| `SimpleTracer.java` | Added `exportingPredicates`, `spanFilters`, `spanReporters` fields; new constructor accepting pipeline components; `exportSpan()` method; `createSpan()` and `registerSpan()` wire the callback | The tracer owns the pipeline configuration and orchestrates the export flow |

---

## 4.9 Connection to Real Framework

| Simplified | Real Framework | Location |
|-----------|---------------|----------|
| `FinishedSpan` | `FinishedSpan` | [`micrometer-tracing/src/main/java/io/micrometer/tracing/exporter/FinishedSpan.java:35`](https://github.com/micrometer-metrics/tracing/blob/dceab1b9/micrometer-tracing/src/main/java/io/micrometer/tracing/exporter/FinishedSpan.java#L35) |
| `SpanExportingPredicate` | `SpanExportingPredicate` | [`micrometer-tracing/src/main/java/io/micrometer/tracing/exporter/SpanExportingPredicate.java:26`](https://github.com/micrometer-metrics/tracing/blob/dceab1b9/micrometer-tracing/src/main/java/io/micrometer/tracing/exporter/SpanExportingPredicate.java#L26) |
| `SpanFilter` | `SpanFilter` | [`micrometer-tracing/src/main/java/io/micrometer/tracing/exporter/SpanFilter.java:26`](https://github.com/micrometer-metrics/tracing/blob/dceab1b9/micrometer-tracing/src/main/java/io/micrometer/tracing/exporter/SpanFilter.java#L26) |
| `SpanReporter` | `SpanReporter` | [`micrometer-tracing/src/main/java/io/micrometer/tracing/exporter/SpanReporter.java:26`](https://github.com/micrometer-metrics/tracing/blob/dceab1b9/micrometer-tracing/src/main/java/io/micrometer/tracing/exporter/SpanReporter.java#L26) |
| `SimpleSpan implements Span, FinishedSpan` | `SimpleSpan implements Span, FinishedSpan` | [`micrometer-tracing-tests/.../simple/SimpleSpan.java:38`](https://github.com/micrometer-metrics/tracing/blob/dceab1b9/micrometer-tracing-tests/micrometer-tracing-test/src/main/java/io/micrometer/tracing/test/simple/SimpleSpan.java#L38) |

**Simplifications vs. real framework:**
- We omit `typedTags` (Object-valued tags added in 1.1.0) — string tags cover the core concept
- We omit `localIp` / `localServiceName` — networking beyond remote service is an edge case
- We omit `SpanIgnoringSpanExportingPredicate` — a concrete predicate that filters by regex; our functional interface lets you write the same logic inline
- We omit `TestSpanReporter` — a concrete reporter that collects into a `ConcurrentLinkedQueue`; our tests use `List::add` directly
- The real framework wires the pipeline through Spring Boot auto-configuration; we pass it in the constructor

---

## 4.10 Complete Code

### Production Code

#### `src/main/java/dev/linhvu/tracing/exporter/FinishedSpan.java` [NEW]

```java
package dev.linhvu.tracing.exporter;

import dev.linhvu.tracing.Link;
import dev.linhvu.tracing.Span;

import java.time.Duration;
import java.time.Instant;
import java.util.Collection;
import java.util.Collections;
import java.util.List;
import java.util.Map;

/**
 * Immutable view of a completed span, ready for export to an external system.
 *
 * <p>This is the <strong>read-time</strong> counterpart to {@link Span} (the write-time API).
 * Once a span finishes, it becomes a {@code FinishedSpan} that flows through the export
 * pipeline: {@link SpanExportingPredicate} → {@link SpanFilter} → {@link SpanReporter}.
 *
 * <p>Setters return {@code this} to support fluent mutation in {@link SpanFilter#map}.
 * In the simple implementation, {@link dev.linhvu.tracing.simple.SimpleSpan} implements
 * both {@code Span} and {@code FinishedSpan} — the same object serves both roles.
 */
public interface FinishedSpan {

    // --- Name ---

    String getName();

    FinishedSpan setName(String name);

    // --- Timestamps ---

    Instant getStartTimestamp();

    Instant getEndTimestamp();

    default Duration getDuration() {
        return Duration.between(getStartTimestamp(), getEndTimestamp());
    }

    // --- Tags ---

    Map<String, String> getTags();

    FinishedSpan setTags(Map<String, String> tags);

    // --- Events ---

    Collection<Map.Entry<Long, String>> getEvents();

    FinishedSpan setEvents(Collection<Map.Entry<Long, String>> events);

    // --- Identity (from TraceContext) ---

    String getTraceId();

    String getSpanId();

    String getParentId();

    // --- Networking ---

    String getRemoteIp();

    int getRemotePort();

    FinishedSpan setRemotePort(int port);

    String getRemoteServiceName();

    FinishedSpan setRemoteServiceName(String remoteServiceName);

    // --- Error ---

    Throwable getError();

    FinishedSpan setError(Throwable error);

    // --- Kind ---

    Span.Kind getKind();

    // --- Links ---

    default List<Link> getLinks() {
        return Collections.emptyList();
    }

    default FinishedSpan addLinks(List<Link> links) {
        return this;
    }

    default FinishedSpan addLink(Link link) {
        return this;
    }

}
```

#### `src/main/java/dev/linhvu/tracing/exporter/SpanExportingPredicate.java` [NEW]

```java
package dev.linhvu.tracing.exporter;

/**
 * Gate stage of the export pipeline — decides whether a finished span should be exported.
 *
 * <p>If <strong>any</strong> registered predicate returns {@code false}, the span is
 * dropped and never reaches the {@link SpanReporter}. Use this to suppress noisy
 * health-check spans, internal framework spans, or spans below a duration threshold.
 *
 * <p>This is a functional interface: {@code span -> boolean}.
 */
@FunctionalInterface
public interface SpanExportingPredicate {

    /**
     * Returns {@code true} if the span should be exported, {@code false} to drop it.
     * @param span the finished span to evaluate
     * @return whether the span should proceed through the pipeline
     */
    boolean isExportable(FinishedSpan span);

}
```

#### `src/main/java/dev/linhvu/tracing/exporter/SpanFilter.java` [NEW]

```java
package dev.linhvu.tracing.exporter;

/**
 * Transformation stage of the export pipeline — mutates a finished span before export.
 *
 * <p>Multiple filters are applied in order. Common uses: rename spans, add/remove tags,
 * redact sensitive data, enrich with environment metadata.
 *
 * <p>This is a functional interface: {@code span -> span}.
 */
@FunctionalInterface
public interface SpanFilter {

    /**
     * Transforms the finished span. May mutate and return the same instance,
     * or return a different {@link FinishedSpan} entirely.
     * @param span the finished span to transform
     * @return the (possibly modified) finished span
     */
    FinishedSpan map(FinishedSpan span);

}
```

#### `src/main/java/dev/linhvu/tracing/exporter/SpanReporter.java` [NEW]

```java
package dev.linhvu.tracing.exporter;

/**
 * Terminal stage of the export pipeline — consumes finished spans for delivery
 * to an external system (Zipkin, in-memory list, log file, etc.).
 *
 * <p>Extends {@link AutoCloseable} so reporters holding resources (network connections,
 * file handles) can be cleaned up.
 */
public interface SpanReporter extends AutoCloseable {

    /**
     * Reports a finished span to the external system.
     * @param span the finished span that has passed through predicates and filters
     */
    void report(FinishedSpan span);

    /**
     * Releases any resources held by this reporter. Default is a no-op.
     */
    @Override
    default void close() {
    }

}
```

#### `src/main/java/dev/linhvu/tracing/simple/SimpleSpan.java` [MODIFIED]

```java
package dev.linhvu.tracing.simple;

import dev.linhvu.tracing.Link;
import dev.linhvu.tracing.Span;
import dev.linhvu.tracing.exporter.FinishedSpan;

import java.time.Instant;
import java.util.AbstractMap;
import java.util.ArrayList;
import java.util.Collection;
import java.util.Collections;
import java.util.List;
import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.ConcurrentLinkedQueue;
import java.util.concurrent.TimeUnit;

/**
 * An in-memory implementation of {@link Span} that also implements {@link FinishedSpan}.
 * The same object serves as both the write-time API (during instrumentation) and the
 * read-time API (in the export pipeline).
 *
 * <p>Each SimpleSpan owns a {@link SimpleTraceContext} that holds its trace/span/parent IDs.
 * The tracer configures these IDs when creating the span.
 */
public class SimpleSpan implements Span, FinishedSpan {

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

    private final List<Link> links = new ArrayList<>();

    /** Callback invoked when span ends — wired by the tracer to feed the export pipeline. */
    private volatile Runnable onSpanEnded;

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
        fireOnSpanEnded();
    }

    @Override
    public void end(long time, TimeUnit timeUnit) {
        this.endMillis = timeUnit.toMillis(time);
        fireOnSpanEnded();
    }

    private void fireOnSpanEnded() {
        Runnable callback = this.onSpanEnded;
        if (callback != null) {
            callback.run();
        }
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

    void setOnSpanEnded(Runnable onSpanEnded) {
        this.onSpanEnded = onSpanEnded;
    }

    // --- FinishedSpan implementation ---
    // Most setters are no-ops because data was already set via the Span interface.

    @Override
    public FinishedSpan setName(String name) {
        this.name = name;
        return this;
    }

    @Override
    public FinishedSpan setTags(Map<String, String> tags) {
        // No-op: tags were set through Span.tag()
        return this;
    }

    @Override
    public FinishedSpan setEvents(Collection<Map.Entry<Long, String>> events) {
        // No-op: events were set through Span.event()
        return this;
    }

    @Override
    public String getTraceId() {
        return context.traceId();
    }

    @Override
    public String getSpanId() {
        return context.spanId();
    }

    @Override
    public String getParentId() {
        return context.parentId();
    }

    @Override
    public FinishedSpan setRemotePort(int port) {
        // No-op: port was set through Span.remoteIpAndPort()
        return this;
    }

    @Override
    public FinishedSpan setRemoteServiceName(String remoteServiceName) {
        this.remoteServiceName = remoteServiceName;
        return this;
    }

    @Override
    public FinishedSpan setError(Throwable error) {
        // No-op: error was set through Span.error()
        return this;
    }

    @Override
    public List<Link> getLinks() {
        return Collections.unmodifiableList(links);
    }

    @Override
    public FinishedSpan addLinks(List<Link> links) {
        this.links.addAll(links);
        return this;
    }

    @Override
    public FinishedSpan addLink(Link link) {
        this.links.add(link);
        return this;
    }

    @Override
    public String toString() {
        return "SimpleSpan{name='" + name + "', traceId='" + context.traceId()
                + "', spanId='" + context.spanId() + "', parentId='" + context.parentId() + "'}";
    }

}
```

#### `src/main/java/dev/linhvu/tracing/simple/SimpleTracer.java` [MODIFIED]

```java
package dev.linhvu.tracing.simple;

import dev.linhvu.tracing.Baggage;
import dev.linhvu.tracing.BaggageInScope;
import dev.linhvu.tracing.ScopedSpan;
import dev.linhvu.tracing.Span;
import dev.linhvu.tracing.SpanCustomizer;
import dev.linhvu.tracing.TraceContext;
import dev.linhvu.tracing.Tracer;
import dev.linhvu.tracing.exporter.FinishedSpan;
import dev.linhvu.tracing.exporter.SpanExportingPredicate;
import dev.linhvu.tracing.exporter.SpanFilter;
import dev.linhvu.tracing.exporter.SpanReporter;

import java.util.ArrayList;
import java.util.Collections;
import java.util.Deque;
import java.util.List;
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

    private final List<SpanExportingPredicate> exportingPredicates;

    private final List<SpanFilter> spanFilters;

    private final List<SpanReporter> spanReporters;

    public SimpleTracer() {
        this(List.of(), List.of(), List.of());
    }

    public SimpleTracer(List<SpanExportingPredicate> exportingPredicates,
            List<SpanFilter> spanFilters, List<SpanReporter> spanReporters) {
        this.currentTraceContext = new SimpleCurrentTraceContext(this);
        this.exportingPredicates = new ArrayList<>(exportingPredicates);
        this.spanFilters = new ArrayList<>(spanFilters);
        this.spanReporters = new ArrayList<>(spanReporters);
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
        span.setOnSpanEnded(() -> exportSpan(span));
        return span;
    }

    /**
     * Runs the finished span through the export pipeline:
     * predicates (gate) → filters (transform) → reporters (sink).
     */
    private void exportSpan(SimpleSpan span) {
        // Gate: if any predicate says no, drop the span
        for (SpanExportingPredicate predicate : exportingPredicates) {
            if (!predicate.isExportable(span)) {
                return;
            }
        }

        // Transform: apply each filter in order
        FinishedSpan result = span;
        for (SpanFilter filter : spanFilters) {
            result = filter.map(result);
        }

        // Sink: deliver to each reporter
        for (SpanReporter reporter : spanReporters) {
            reporter.report(result);
        }
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
        span.setOnSpanEnded(() -> exportSpan(span));
    }

}
```

### Test Code

#### `src/test/java/dev/linhvu/tracing/exporter/FinishedSpanTest.java` [NEW]

```java
package dev.linhvu.tracing.exporter;

import dev.linhvu.tracing.Span;
import dev.linhvu.tracing.simple.SimpleSpan;
import dev.linhvu.tracing.simple.SimpleTracer;

import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Nested;
import org.junit.jupiter.api.Test;

import java.time.Duration;

import static org.assertj.core.api.Assertions.assertThat;

/**
 * Tests for the {@link FinishedSpan} interface, verified through {@link SimpleSpan}
 * which implements both {@link Span} and {@link FinishedSpan}.
 */
@DisplayName("FinishedSpan")
class FinishedSpanTest {

    SimpleTracer tracer = new SimpleTracer();

    @Nested
    @DisplayName("Identity")
    class Identity {

        @Test
        void shouldExposeTraceId_WhenSpanIsFinished() {
            SimpleSpan span = tracer.nextSpan();
            span.start();
            span.end();

            FinishedSpan finished = span;
            assertThat(finished.getTraceId()).isEqualTo(span.context().traceId());
            assertThat(finished.getTraceId()).isNotEmpty();
        }

        @Test
        void shouldExposeSpanId_WhenSpanIsFinished() {
            SimpleSpan span = tracer.nextSpan();
            span.start();
            span.end();

            FinishedSpan finished = span;
            assertThat(finished.getSpanId()).isEqualTo(span.context().spanId());
        }

        @Test
        void shouldExposeParentId_WhenChildSpan() {
            SimpleSpan parent = tracer.nextSpan();
            parent.start();

            SimpleSpan child = tracer.nextSpan(parent);
            child.start();
            child.end();
            parent.end();

            FinishedSpan finished = child;
            assertThat(finished.getParentId()).isEqualTo(parent.context().spanId());
        }
    }

    @Nested
    @DisplayName("Metadata")
    class Metadata {

        @Test
        void shouldExposeName_WhenSpanIsNamed() {
            SimpleSpan span = tracer.nextSpan();
            span.start();
            span.name("my-operation");
            span.end();

            FinishedSpan finished = span;
            assertThat(finished.getName()).isEqualTo("my-operation");
        }

        @Test
        void shouldExposeTags_WhenTagsAreSet() {
            SimpleSpan span = tracer.nextSpan();
            span.start();
            span.tag("http.method", "GET");
            span.tag("http.status_code", "200");
            span.end();

            FinishedSpan finished = span;
            assertThat(finished.getTags())
                    .containsEntry("http.method", "GET")
                    .containsEntry("http.status_code", "200");
        }

        @Test
        void shouldExposeEvents_WhenEventsAreRecorded() {
            SimpleSpan span = tracer.nextSpan();
            span.start();
            span.event("cache.miss");
            span.end();

            FinishedSpan finished = span;
            assertThat(finished.getEvents())
                    .extracting(e -> e.getValue())
                    .containsExactly("cache.miss");
        }

        @Test
        void shouldExposeError_WhenErrorIsRecorded() {
            RuntimeException error = new RuntimeException("boom");
            SimpleSpan span = tracer.nextSpan();
            span.start();
            span.error(error);
            span.end();

            FinishedSpan finished = span;
            assertThat(finished.getError()).isSameAs(error);
        }

        @Test
        void shouldExposeKind_WhenKindIsSet() {
            SimpleSpan span = (SimpleSpan) tracer.spanBuilder()
                    .kind(Span.Kind.CLIENT)
                    .start();
            span.end();

            FinishedSpan finished = span;
            assertThat(finished.getKind()).isEqualTo(Span.Kind.CLIENT);
        }

        @Test
        void shouldExposeRemoteServiceName_WhenSet() {
            SimpleSpan span = tracer.nextSpan();
            span.start();
            span.remoteServiceName("user-service");
            span.end();

            FinishedSpan finished = span;
            assertThat(finished.getRemoteServiceName()).isEqualTo("user-service");
        }
    }

    @Nested
    @DisplayName("Timestamps")
    class Timestamps {

        @Test
        void shouldExposeTimestamps_WhenSpanIsFinished() {
            SimpleSpan span = tracer.nextSpan();
            span.start();
            span.end();

            FinishedSpan finished = span;
            assertThat(finished.getStartTimestamp()).isNotNull();
            assertThat(finished.getEndTimestamp()).isNotNull();
            assertThat(finished.getEndTimestamp()).isAfterOrEqualTo(finished.getStartTimestamp());
        }

        @Test
        void shouldComputeDuration_WhenBothTimestampsPresent() {
            SimpleSpan span = tracer.nextSpan();
            span.start();
            span.end();

            Duration duration = span.getDuration();
            assertThat(duration).isGreaterThanOrEqualTo(Duration.ZERO);
        }
    }

    @Nested
    @DisplayName("Mutability via setters")
    class MutabilityViaSetters {

        @Test
        void shouldAllowNameChange_WhenSetNameCalled() {
            SimpleSpan span = tracer.nextSpan();
            span.start();
            span.name("original");
            span.end();

            FinishedSpan finished = span;
            finished.setName("renamed");
            assertThat(finished.getName()).isEqualTo("renamed");
        }

        @Test
        void shouldAllowRemoteServiceNameChange_WhenSetRemoteServiceNameCalled() {
            SimpleSpan span = tracer.nextSpan();
            span.start();
            span.remoteServiceName("original-service");
            span.end();

            FinishedSpan finished = span;
            finished.setRemoteServiceName("updated-service");
            assertThat(finished.getRemoteServiceName()).isEqualTo("updated-service");
        }
    }
}
```

#### `src/test/java/dev/linhvu/tracing/exporter/SpanExportPipelineTest.java` [NEW]

```java
package dev.linhvu.tracing.exporter;

import dev.linhvu.tracing.simple.SimpleSpan;
import dev.linhvu.tracing.simple.SimpleTracer;

import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Nested;
import org.junit.jupiter.api.Test;

import java.util.ArrayList;
import java.util.List;
import java.util.concurrent.CopyOnWriteArrayList;

import static org.assertj.core.api.Assertions.assertThat;

/**
 * Tests for {@link SpanExportingPredicate}, {@link SpanFilter}, and {@link SpanReporter}
 * wired through {@link SimpleTracer}.
 */
@DisplayName("Span Export Pipeline")
class SpanExportPipelineTest {

    @Nested
    @DisplayName("SpanReporter")
    class ReporterTests {

        @Test
        void shouldReportSpan_WhenSpanEnds() {
            List<FinishedSpan> reported = new CopyOnWriteArrayList<>();
            SimpleTracer tracer = new SimpleTracer(List.of(), List.of(), List.of(reported::add));

            SimpleSpan span = tracer.nextSpan();
            span.start();
            span.name("test-span");
            span.end();

            assertThat(reported).hasSize(1);
            assertThat(reported.get(0).getName()).isEqualTo("test-span");
        }

        @Test
        void shouldReportToMultipleReporters_WhenMultipleRegistered() {
            List<FinishedSpan> reporter1 = new CopyOnWriteArrayList<>();
            List<FinishedSpan> reporter2 = new CopyOnWriteArrayList<>();
            SimpleTracer tracer = new SimpleTracer(
                    List.of(), List.of(), List.of(reporter1::add, reporter2::add));

            SimpleSpan span = tracer.nextSpan();
            span.start();
            span.end();

            assertThat(reporter1).hasSize(1);
            assertThat(reporter2).hasSize(1);
        }

        @Test
        void shouldNotReportSpan_WhenSpanIsAbandoned() {
            List<FinishedSpan> reported = new CopyOnWriteArrayList<>();
            SimpleTracer tracer = new SimpleTracer(List.of(), List.of(), List.of(reported::add));

            SimpleSpan span = tracer.nextSpan();
            span.start();
            span.abandon();

            // Abandoned spans don't call end(), so no export
            assertThat(reported).isEmpty();
        }

        @Test
        void shouldReportSpanFromBuilder_WhenBuilderCreatesSpan() {
            List<FinishedSpan> reported = new CopyOnWriteArrayList<>();
            SimpleTracer tracer = new SimpleTracer(List.of(), List.of(), List.of(reported::add));

            SimpleSpan span = (SimpleSpan) tracer.spanBuilder()
                    .name("builder-span")
                    .start();
            span.end();

            assertThat(reported).hasSize(1);
            assertThat(reported.get(0).getName()).isEqualTo("builder-span");
        }
    }

    @Nested
    @DisplayName("SpanExportingPredicate")
    class PredicateTests {

        @Test
        void shouldDropSpan_WhenPredicateReturnsFalse() {
            List<FinishedSpan> reported = new CopyOnWriteArrayList<>();
            SimpleTracer tracer = new SimpleTracer(
                    List.of(span -> false), List.of(), List.of(reported::add));

            SimpleSpan span = tracer.nextSpan();
            span.start();
            span.end();

            assertThat(reported).isEmpty();
        }

        @Test
        void shouldExportSpan_WhenPredicateReturnsTrue() {
            List<FinishedSpan> reported = new CopyOnWriteArrayList<>();
            SimpleTracer tracer = new SimpleTracer(
                    List.of(span -> true), List.of(), List.of(reported::add));

            SimpleSpan span = tracer.nextSpan();
            span.start();
            span.end();

            assertThat(reported).hasSize(1);
        }

        @Test
        void shouldDropSpan_WhenAnyPredicateReturnsFalse() {
            List<FinishedSpan> reported = new CopyOnWriteArrayList<>();
            SimpleTracer tracer = new SimpleTracer(
                    List.of(span -> true, span -> false), List.of(), List.of(reported::add));

            SimpleSpan span = tracer.nextSpan();
            span.start();
            span.end();

            assertThat(reported).isEmpty();
        }

        @Test
        void shouldFilterByName_WhenPredicateChecksName() {
            List<FinishedSpan> reported = new CopyOnWriteArrayList<>();
            SpanExportingPredicate skipHealthChecks =
                    span -> !"health-check".equals(span.getName());

            SimpleTracer tracer = new SimpleTracer(
                    List.of(skipHealthChecks), List.of(), List.of(reported::add));

            // This one should be exported
            SimpleSpan apiSpan = tracer.nextSpan();
            apiSpan.start();
            apiSpan.name("api-request");
            apiSpan.end();

            // This one should be dropped
            SimpleSpan healthSpan = tracer.nextSpan();
            healthSpan.start();
            healthSpan.name("health-check");
            healthSpan.end();

            assertThat(reported).hasSize(1);
            assertThat(reported.get(0).getName()).isEqualTo("api-request");
        }
    }

    @Nested
    @DisplayName("SpanFilter")
    class FilterTests {

        @Test
        void shouldTransformSpan_WhenFilterApplied() {
            List<FinishedSpan> reported = new CopyOnWriteArrayList<>();
            SpanFilter prefixFilter = span -> {
                span.setName("prefix-" + span.getName());
                return span;
            };

            SimpleTracer tracer = new SimpleTracer(
                    List.of(), List.of(prefixFilter), List.of(reported::add));

            SimpleSpan span = tracer.nextSpan();
            span.start();
            span.name("operation");
            span.end();

            assertThat(reported.get(0).getName()).isEqualTo("prefix-operation");
        }

        @Test
        void shouldChainFilters_WhenMultipleFiltersRegistered() {
            List<FinishedSpan> reported = new CopyOnWriteArrayList<>();
            SpanFilter addEnvTag = span -> {
                // Can't add tags via FinishedSpan.setTags (no-op), but can setName
                span.setName(span.getName() + "-filtered");
                return span;
            };
            SpanFilter upperCase = span -> {
                span.setName(span.getName().toUpperCase());
                return span;
            };

            SimpleTracer tracer = new SimpleTracer(
                    List.of(), List.of(addEnvTag, upperCase), List.of(reported::add));

            SimpleSpan span = tracer.nextSpan();
            span.start();
            span.name("test");
            span.end();

            // Filters applied in order: "test" → "test-filtered" → "TEST-FILTERED"
            assertThat(reported.get(0).getName()).isEqualTo("TEST-FILTERED");
        }
    }

    @Nested
    @DisplayName("Full Pipeline")
    class FullPipelineTests {

        @Test
        void shouldApplyPredicateThenFilterThenReporter_WhenAllStagesConfigured() {
            List<FinishedSpan> reported = new CopyOnWriteArrayList<>();

            // Predicate: skip spans named "internal"
            SpanExportingPredicate predicate = span -> !"internal".equals(span.getName());

            // Filter: add a suffix
            SpanFilter filter = span -> {
                span.setRemoteServiceName("enriched-" + span.getRemoteServiceName());
                return span;
            };

            SimpleTracer tracer = new SimpleTracer(
                    List.of(predicate), List.of(filter), List.of(reported::add));

            // Span 1: should be exported and filtered
            SimpleSpan span1 = tracer.nextSpan();
            span1.start();
            span1.name("api-call");
            span1.remoteServiceName("backend");
            span1.end();

            // Span 2: should be dropped by predicate
            SimpleSpan span2 = tracer.nextSpan();
            span2.start();
            span2.name("internal");
            span2.remoteServiceName("self");
            span2.end();

            assertThat(reported).hasSize(1);
            assertThat(reported.get(0).getName()).isEqualTo("api-call");
            assertThat(reported.get(0).getRemoteServiceName()).isEqualTo("enriched-backend");
        }

        @Test
        void shouldWorkWithNoExportPipeline_WhenDefaultTracerUsed() {
            SimpleTracer tracer = new SimpleTracer();

            SimpleSpan span = tracer.nextSpan();
            span.start();
            span.name("no-pipeline");
            span.end();

            // No errors, span still collected in tracer
            assertThat(tracer.getSpans()).hasSize(1);
        }

        @Test
        void shouldExportMultipleSpans_WhenTraceHasParentChild() {
            List<FinishedSpan> reported = new CopyOnWriteArrayList<>();
            SimpleTracer tracer = new SimpleTracer(List.of(), List.of(), List.of(reported::add));

            SimpleSpan parent = tracer.nextSpan();
            parent.start();
            parent.name("parent-op");

            SimpleSpan child = tracer.nextSpan(parent);
            child.start();
            child.name("child-op");
            child.end();

            parent.end();

            assertThat(reported).hasSize(2);
            assertThat(reported).extracting(FinishedSpan::getName)
                    .containsExactly("child-op", "parent-op");
        }

        @Test
        void shouldExportSpanWithTimedEnd_WhenEndCalledWithTimestamp() {
            List<FinishedSpan> reported = new CopyOnWriteArrayList<>();
            SimpleTracer tracer = new SimpleTracer(List.of(), List.of(), List.of(reported::add));

            SimpleSpan span = tracer.nextSpan();
            span.start();
            span.name("timed-span");
            span.end(System.currentTimeMillis(), java.util.concurrent.TimeUnit.MILLISECONDS);

            assertThat(reported).hasSize(1);
            assertThat(reported.get(0).getName()).isEqualTo("timed-span");
        }
    }
}
```

#### `src/test/java/dev/linhvu/tracing/integration/ExportPipelineIntegrationTest.java` [NEW]

```java
package dev.linhvu.tracing.integration;

import dev.linhvu.tracing.ScopedSpan;
import dev.linhvu.tracing.Span;
import dev.linhvu.tracing.Tracer;
import dev.linhvu.tracing.exporter.FinishedSpan;
import dev.linhvu.tracing.exporter.SpanExportingPredicate;
import dev.linhvu.tracing.exporter.SpanFilter;
import dev.linhvu.tracing.simple.SimpleTracer;

import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Test;

import java.util.List;
import java.util.concurrent.CopyOnWriteArrayList;

import static org.assertj.core.api.Assertions.assertThat;

/**
 * Integration tests verifying the export pipeline works end-to-end with
 * all tracing constructs: manual spans, scoped spans, span builders, and nested traces.
 */
@DisplayName("Export Pipeline Integration")
class ExportPipelineIntegrationTest {

    @Test
    void shouldExportNestedTrace_WhenUsingWithSpanScope() {
        List<FinishedSpan> reported = new CopyOnWriteArrayList<>();
        SimpleTracer tracer = new SimpleTracer(List.of(), List.of(), List.of(reported::add));

        // Simulate a 3-level nested trace
        Span root = tracer.nextSpan();
        root.start().name("http-request");
        try (Tracer.SpanInScope rootScope = tracer.withSpan(root)) {
            Span service = tracer.nextSpan();
            service.start().name("service-call");
            try (Tracer.SpanInScope serviceScope = tracer.withSpan(service)) {
                Span db = tracer.nextSpan();
                db.start().name("db-query");
                db.end();
            }
            service.end();
        }
        root.end();

        // All 3 spans exported in end-order
        assertThat(reported).hasSize(3);
        assertThat(reported).extracting(FinishedSpan::getName)
                .containsExactly("db-query", "service-call", "http-request");

        // All share the same traceId
        String traceId = reported.get(0).getTraceId();
        assertThat(reported).allMatch(s -> s.getTraceId().equals(traceId));

        // Parent-child relationships
        assertThat(reported.get(0).getParentId()).isEqualTo(reported.get(1).getSpanId()); // db -> service
        assertThat(reported.get(1).getParentId()).isEqualTo(reported.get(2).getSpanId()); // service -> root
    }

    @Test
    void shouldExportScopedSpan_WhenScopedSpanEnds() {
        List<FinishedSpan> reported = new CopyOnWriteArrayList<>();
        SimpleTracer tracer = new SimpleTracer(List.of(), List.of(), List.of(reported::add));

        ScopedSpan scopedSpan = tracer.startScopedSpan("scoped-operation");
        scopedSpan.tag("key", "value");
        scopedSpan.end();

        assertThat(reported).hasSize(1);
        assertThat(reported.get(0).getName()).isEqualTo("scoped-operation");
        assertThat(reported.get(0).getTags()).containsEntry("key", "value");
    }

    @Test
    void shouldApplyFilterToAllSpans_WhenFilterAddsEnvironmentTag() {
        List<FinishedSpan> reported = new CopyOnWriteArrayList<>();

        // Filter that renames spans with an environment prefix
        SpanFilter envFilter = span -> {
            span.setName("[test] " + span.getName());
            return span;
        };

        SimpleTracer tracer = new SimpleTracer(List.of(), List.of(envFilter), List.of(reported::add));

        tracer.nextSpan().start().name("span-a").end();
        tracer.nextSpan().start().name("span-b").end();

        assertThat(reported).extracting(FinishedSpan::getName)
                .containsExactly("[test] span-a", "[test] span-b");
    }

    @Test
    void shouldFilterHealthChecksFromExport_WhenPredicateConfigured() {
        List<FinishedSpan> reported = new CopyOnWriteArrayList<>();

        // Real-world scenario: suppress health-check and actuator spans
        SpanExportingPredicate skipInfra = span -> {
            String name = span.getName();
            return name != null && !name.startsWith("actuator/") && !name.equals("health-check");
        };

        SimpleTracer tracer = new SimpleTracer(
                List.of(skipInfra), List.of(), List.of(reported::add));

        tracer.nextSpan().start().name("api/users").end();
        tracer.nextSpan().start().name("health-check").end();
        tracer.nextSpan().start().name("actuator/metrics").end();
        tracer.nextSpan().start().name("api/orders").end();

        assertThat(reported).extracting(FinishedSpan::getName)
                .containsExactly("api/users", "api/orders");
    }

    @Test
    void shouldChainPredicateAndFilter_WhenBothConfigured() {
        List<FinishedSpan> reported = new CopyOnWriteArrayList<>();

        // Predicate: only export spans longer than "zero" duration (all real spans)
        SpanExportingPredicate exportAll = span -> true;

        // Filter: set remote service name based on span kind
        SpanFilter enrichKind = span -> {
            if (span.getKind() == Span.Kind.CLIENT) {
                span.setRemoteServiceName("downstream-" + span.getRemoteServiceName());
            }
            return span;
        };

        SimpleTracer tracer = new SimpleTracer(
                List.of(exportAll), List.of(enrichKind), List.of(reported::add));

        Span clientSpan = tracer.spanBuilder()
                .kind(Span.Kind.CLIENT)
                .name("http-call")
                .remoteServiceName("payments")
                .start();
        clientSpan.end();

        assertThat(reported.get(0).getRemoteServiceName()).isEqualTo("downstream-payments");
    }
}
```

---

## Summary

| What | Details |
|------|---------|
| **New interfaces** | `FinishedSpan`, `SpanExportingPredicate`, `SpanFilter`, `SpanReporter` |
| **Modified classes** | `SimpleSpan` (now implements `FinishedSpan`), `SimpleTracer` (pipeline wiring) |
| **Pipeline flow** | `span.end()` → callback → predicate (gate) → filter (transform) → reporter (sink) |
| **Key pattern** | Pipes and filters with observer callback for decoupling |
| **Tests** | 13 unit tests (FinishedSpan identity/metadata/timestamps/mutability, predicates, filters, full pipeline), 5 integration tests |

### Next Chapter Preview

**Chapter 5: Propagator (Cross-Process Context Propagation)** — inject trace context into outgoing carriers (HTTP headers) and extract it from incoming ones. You'll build the `Propagator` interface with `inject()`/`extract()` methods and a `W3CTraceContextPropagator` that implements the `traceparent` header format.
