# Chapter 2: Span Export Pipeline — Report, Filter, and Gate Finished Spans

> **What you'll build**: The four export pipeline interfaces — `FinishedSpan`, `SpanFilter`,
> `SpanExportingPredicate`, and `SpanReporter` — that define how completed spans flow through
> transformation, gating, and reporting stages before reaching external systems.

---

## 1. The API Contract

After this chapter, a client can write:

```java
// Configure the export pipeline
SpanReporter reporter = span -> System.out.println("Exported: " + span.getName());
SpanFilter filter = span -> span.setName(span.getName().toUpperCase());
SpanExportingPredicate predicate = span -> !span.getName().startsWith("INTERNAL-");

// Run a finished span through the pipeline
FinishedSpan span = ...;  // created by the tracer when span.end() is called
FinishedSpan filtered = filter.map(span);
if (predicate.isExportable(filtered)) {
    reporter.report(filtered);
}
```

**Three stages, one flow**: Every finished span passes through filters (transform), then
predicates (gate), then reporters (export). The interfaces are decoupled — each is a single
functional interface that can be composed independently.

### Pipeline Architecture

```
span.end()
  │
  ▼
FinishedSpan         ← snapshot of the completed span's data
  │
  ├──► SpanFilter.map(span)         ← transform: rename, add tags, redact
  │        (repeat for each filter)
  │
  ├──► SpanExportingPredicate.isExportable(span)    ← gate: drop or keep
  │        (ALL predicates must return true)
  │
  └──► SpanReporter.report(span)    ← export: send to Zipkin, log, collect
```

---

## 2. Client Usage & Tests

### Test: Reporter receives finished spans

The most basic pipeline — a reporter collects everything:

```java
@Test
void reporter_shouldReceiveFinishedSpan() {
    List<FinishedSpan> reported = new ArrayList<>();
    SpanReporter reporter = reported::add;

    FinishedSpan span = TestFinishedSpan.create("my-operation");
    reporter.report(span);

    assertThat(reported).hasSize(1);
    assertThat(reported.get(0).getName()).isEqualTo("my-operation");
}
```

**Why this works**: `SpanReporter` is a `@FunctionalInterface` with a single `report()` method.
The method reference `reported::add` satisfies the contract — any lambda that accepts a
`FinishedSpan` works as a reporter.

### Test: Filter transforms a span before export

Filters mutate the `FinishedSpan` in place. This is why `FinishedSpan` has setters:

```java
@Test
void filter_shouldTransformSpanName() {
    SpanFilter uppercaseFilter = span -> span.setName(span.getName().toUpperCase());

    FinishedSpan span = TestFinishedSpan.create("my-operation");
    FinishedSpan result = uppercaseFilter.map(span);

    assertThat(result.getName()).isEqualTo("MY-OPERATION");
}
```

### Test: Multiple filters chain in order

Filters compose sequentially — each receives the output of the previous:

```java
@Test
void multipleFilters_shouldChainInOrder() {
    SpanFilter addPrefix = span -> span.setName("prefix-" + span.getName());
    SpanFilter addSuffix = span -> span.setName(span.getName() + "-suffix");

    FinishedSpan span = TestFinishedSpan.create("operation");

    FinishedSpan result = span;
    for (SpanFilter filter : List.of(addPrefix, addSuffix)) {
        result = filter.map(result);
    }

    assertThat(result.getName()).isEqualTo("prefix-operation-suffix");
}
```

### Test: Predicate gates export

Predicates decide "should this span be exported at all?":

```java
@Test
void predicate_shouldBlockExport() {
    SpanExportingPredicate blockInternal = span -> !span.getName().startsWith("internal-");

    FinishedSpan internal = TestFinishedSpan.create("internal-healthcheck");
    FinishedSpan external = TestFinishedSpan.create("http-request");

    assertThat(blockInternal.isExportable(internal)).isFalse();
    assertThat(blockInternal.isExportable(external)).isTrue();
}
```

### Test: Full pipeline — filter then gate then report

The complete flow showing all three stages working together:

```java
@Test
void fullPipeline_shouldFilterThenGateThenReport() {
    SpanFilter uppercaseFilter = span -> span.setName(span.getName().toUpperCase());
    SpanExportingPredicate blockInternal = span -> !span.getName().startsWith("INTERNAL-");
    List<FinishedSpan> reported = new ArrayList<>();
    SpanReporter reporter = reported::add;

    List<FinishedSpan> spans = List.of(
            TestFinishedSpan.create("http-request"),
            TestFinishedSpan.create("internal-healthcheck"),
            TestFinishedSpan.create("db-query")
    );

    for (FinishedSpan span : spans) {
        FinishedSpan filtered = uppercaseFilter.map(span);
        if (blockInternal.isExportable(filtered)) {
            reporter.report(filtered);
        }
    }

    // internal-healthcheck was blocked after uppercase → "INTERNAL-HEALTHCHECK"
    assertThat(reported).hasSize(2);
    assertThat(reported.get(0).getName()).isEqualTo("HTTP-REQUEST");
    assertThat(reported.get(1).getName()).isEqualTo("DB-QUERY");
}
```

**Why filter order matters**: The filter uppercases names *before* the predicate runs. So
`"internal-healthcheck"` becomes `"INTERNAL-HEALTHCHECK"`, and the predicate checks for
`"INTERNAL-"` prefix. If the predicate ran first (on the lowercase name with `"internal-"`),
it would still block — but the important point is that filters see the span first.

---

## 3. Implementing the Call Chain — Top-Down

### Layer 1: FinishedSpan (the data model)

The core data model for a completed span. Unlike `Span` (which is used during a span's
active lifecycle), `FinishedSpan` is **mutable by design** — filters need to transform it:

```java
public interface FinishedSpan {

    String getName();
    FinishedSpan setName(String name);

    Instant getStartTimestamp();
    Instant getEndTimestamp();

    default Duration getDuration() {
        return Duration.between(getStartTimestamp(), getEndTimestamp());
    }

    Map<String, String> getTags();
    FinishedSpan setTags(Map<String, String> tags);

    Collection<Map.Entry<Long, String>> getEvents();
    FinishedSpan setEvents(Collection<Map.Entry<Long, String>> events);

    String getSpanId();
    String getParentId();
    String getTraceId();

    Throwable getError();
    FinishedSpan setError(Throwable error);

    Span.Kind getKind();

    String getRemoteServiceName();
    FinishedSpan setRemoteServiceName(String remoteServiceName);
}
```

**Design decisions**:
- **Mutable setters return `this`** — enables fluent chaining in filters:
  `span.setName("new").setTags(Map.of("k","v"))`.
- **Identity fields (traceId, spanId, parentId) are read-only** — changing these would break
  trace correlation. Filters can change names and tags, but not identity.
- **`getDuration()` is a default method** — computed from timestamps, so implementations
  don't need to store it separately.
- **Events use `Map.Entry<Long, String>`** — the Long is the epoch timestamp in milliseconds,
  matching the real framework's approach.

### Layer 2: SpanFilter (the "map" stage)

A single-method functional interface that transforms finished spans:

```java
@FunctionalInterface
public interface SpanFilter {

    FinishedSpan map(FinishedSpan span);
}
```

**Why `map` not `filter`**: Despite the name `SpanFilter`, this is a *transformation*
(map operation), not a predicate. The method name `map` signals that it transforms and
returns the span. The actual filtering (drop/keep) is handled by `SpanExportingPredicate`.

### Layer 3: SpanExportingPredicate (the "gate" stage)

Decides whether a finished span should be exported:

```java
@FunctionalInterface
public interface SpanExportingPredicate {

    boolean isExportable(FinishedSpan span);
}
```

**Composition**: Multiple predicates are AND-composed — a span is exported only if ALL
predicates return `true`. This is different from filters (which chain sequentially) because
predicates are independent checks, not sequential transformations.

### Layer 4: SpanReporter (the "sink" stage)

The terminal stage that sends spans to external systems:

```java
public interface SpanReporter extends AutoCloseable {

    SpanReporter NOOP = span -> {};

    void report(FinishedSpan finishedSpan);

    @Override
    default void close() {}
}
```

**Design decisions**:
- **Extends `AutoCloseable`** — reporters often hold resources (HTTP connections to Zipkin,
  file handles, buffers). The default `close()` is a no-op so simple lambdas work.
- **`NOOP` constant** — follows the same NOOP pattern from Chapter 1. Use this when no
  reporting is needed (e.g., in unit tests where you just want to verify span creation).

---

## 4. Try It Yourself

1. **Write a `SpanFilter` that redacts sensitive tags** — given a set of tag keys to redact,
   replace their values with `"[REDACTED]"`. Test it with tags like `"auth.token"` and
   `"user.email"`.

2. **Write a `SpanExportingPredicate` that drops short spans** — spans with duration under
   a threshold (e.g., 10ms) should not be exported. Use the `getDuration()` default method.

3. **Compose a `SpanReporter` that fans out to multiple reporters** — write a
   `CompositeSpanReporter` that takes a `List<SpanReporter>` and calls `report()` on each.

---

## 5. Why This Works

### Mutable FinishedSpan, Immutable TraceContext
`TraceContext` (Chapter 1) is immutable — it represents identity that must not change.
`FinishedSpan` is mutable — it represents data that the pipeline should be able to transform.
This distinction is deliberate: filters can rename a span or add tags, but they cannot
change which trace the span belongs to.

### Functional Interface Composition
All three pipeline stages (`SpanFilter`, `SpanExportingPredicate`, `SpanReporter`) are
functional interfaces. This means they can be expressed as lambdas, method references, or
composed using Java's built-in functional composition. The pipeline orchestration is not
baked into the interfaces — it lives in the bridge implementations (later features), giving
the framework maximum flexibility.

### The Pipeline Is Lazy
Notice that the interfaces don't define the pipeline itself — they only define the stages.
The actual "apply filters → check predicates → report" loop is written by the bridge
implementation when a span ends. This means different bridges (Brave, OpenTelemetry, test
fake) can orchestrate the pipeline differently — or even skip stages if they handle
filtering internally.

---

## 6. What We Enhanced

| Component | State After Ch02 | Simplifications |
|-----------|-------------------|-----------------|
| `FinishedSpan` | 15 methods (name, tags, events, IDs, error, kind, timestamps) | No `typedTags`, `localIp`, `remoteIp`, `remotePort`, `localServiceName`, `links` |
| `SpanFilter` | 1 method (`map`) | — |
| `SpanExportingPredicate` | 1 method (`isExportable`) | No `SpanIgnoringSpanExportingPredicate` concrete impl |
| `SpanReporter` | 1 method + `close()` + NOOP | — |

**What's NOT connected yet**: These interfaces are standalone — `Span.end()` does not yet
trigger the export pipeline. That connection happens when we build bridge implementations
(Feature 9: SimpleTracer, Feature 10: BraveTracer). For now, the pipeline stages are defined
and tested in isolation.

---

## 7. Connection to Real Framework

| Simplified Class | Real Framework Class | Source File |
|------------------|----------------------|-------------|
| `io.simpletracing.exporter.FinishedSpan` | `io.micrometer.tracing.exporter.FinishedSpan` | `micrometer-tracing/src/main/java/io/micrometer/tracing/exporter/FinishedSpan.java` |
| `io.simpletracing.exporter.SpanReporter` | `io.micrometer.tracing.exporter.SpanReporter` | `micrometer-tracing/src/main/java/io/micrometer/tracing/exporter/SpanReporter.java` |
| `io.simpletracing.exporter.SpanFilter` | `io.micrometer.tracing.exporter.SpanFilter` | `micrometer-tracing/src/main/java/io/micrometer/tracing/exporter/SpanFilter.java` |
| `io.simpletracing.exporter.SpanExportingPredicate` | `io.micrometer.tracing.exporter.SpanExportingPredicate` | `micrometer-tracing/src/main/java/io/micrometer/tracing/exporter/SpanExportingPredicate.java` |

**Real framework extras we skipped**:
- `SpanIgnoringSpanExportingPredicate` — a concrete implementation that rejects spans by
  regex pattern matching on name. Uses `ConcurrentHashMap` for pattern caching.
- `FinishedSpan.getTypedTags()` / `setTypedTags()` — typed tag values (`Object` instead of
  `String`), with list-to-CSV conversion logic.
- `FinishedSpan.getLinks()` / `addLink()` — links to other spans (from `Link` class).
- `FinishedSpan.getLocalIp()` / `getRemoteIp()` / `getRemotePort()` — network identity.

---

## 8. Complete Code

### `simple-tracing-api/src/main/java/io/simpletracing/exporter/FinishedSpan.java` [NEW]

```java
package io.simpletracing.exporter;

import java.time.Duration;
import java.time.Instant;
import java.util.Collection;
import java.util.Map;

import io.simpletracing.Span;

/**
 * Represents a span that has been finished and is ready for export. This is the data
 * model that flows through the export pipeline: filters can mutate it, predicates can
 * gate it, and reporters can send it to external systems.
 *
 * <p>Unlike {@link io.simpletracing.Span} (which is used during a span's active lifecycle),
 * a {@code FinishedSpan} is mutable by design — {@link SpanFilter}s transform it before
 * export. Identity fields (traceId, spanId, parentId) are read-only since changing them
 * would break trace correlation.
 *
 * <p>Maps to: {@code io.micrometer.tracing.exporter.FinishedSpan}
 *
 * @see SpanFilter
 * @see SpanExportingPredicate
 * @see SpanReporter
 */
public interface FinishedSpan {

    /**
     * Returns the name of this span.
     * @return span name
     */
    String getName();

    /**
     * Sets the name of this span. Used by {@link SpanFilter}s to rename spans.
     * @param name new span name
     * @return this for chaining
     */
    FinishedSpan setName(String name);

    /**
     * Returns the start timestamp of this span.
     * @return start timestamp
     */
    Instant getStartTimestamp();

    /**
     * Returns the end timestamp of this span.
     * @return end timestamp
     */
    Instant getEndTimestamp();

    /**
     * Returns the duration of this span (end - start).
     * @return span duration
     */
    default Duration getDuration() {
        return Duration.between(getStartTimestamp(), getEndTimestamp());
    }

    /**
     * Returns the tags (key-value pairs) set on this span.
     * @return tags map
     */
    Map<String, String> getTags();

    /**
     * Replaces all tags on this span. Used by {@link SpanFilter}s to modify tags.
     * @param tags new tags
     * @return this for chaining
     */
    FinishedSpan setTags(Map<String, String> tags);

    /**
     * Returns the events recorded on this span. Each entry has a timestamp (epoch
     * milliseconds) as key and an event description as value.
     * @return events collection
     */
    Collection<Map.Entry<Long, String>> getEvents();

    /**
     * Replaces all events on this span.
     * @param events new events
     * @return this for chaining
     */
    FinishedSpan setEvents(Collection<Map.Entry<Long, String>> events);

    /**
     * Returns the span ID — unique within the trace.
     * @return span id
     */
    String getSpanId();

    /**
     * Returns the parent span ID, or {@code null} if this is a root span.
     * @return parent span id, or null
     */
    String getParentId();

    /**
     * Returns the trace ID — shared by all spans in the same trace.
     * @return trace id
     */
    String getTraceId();

    /**
     * Returns the error recorded on this span, or {@code null} if none.
     * @return throwable, or null
     */
    Throwable getError();

    /**
     * Sets the error on this span.
     * @param error the error
     * @return this for chaining
     */
    FinishedSpan setError(Throwable error);

    /**
     * Returns the {@link Span.Kind} of this span, or {@code null} if not set.
     * @return span kind, or null
     */
    Span.Kind getKind();

    /**
     * Returns the remote service name, or {@code null} if not set.
     * @return remote service name, or null
     */
    String getRemoteServiceName();

    /**
     * Sets the remote service name.
     * @param remoteServiceName remote service name
     * @return this for chaining
     */
    FinishedSpan setRemoteServiceName(String remoteServiceName);

}
```

### `simple-tracing-api/src/main/java/io/simpletracing/exporter/SpanFilter.java` [NEW]

```java
package io.simpletracing.exporter;

/**
 * Transforms a {@link FinishedSpan} before it is exported. Filters can rename spans,
 * add/remove tags, or modify any mutable field on the finished span.
 *
 * <p>Multiple filters are applied in order — each receives the output of the previous one.
 * This is the "map" stage of the export pipeline.
 *
 * <p>Maps to: {@code io.micrometer.tracing.exporter.SpanFilter}
 *
 * @see FinishedSpan
 * @see SpanExportingPredicate
 * @see SpanReporter
 */
@FunctionalInterface
public interface SpanFilter {

    /**
     * Transforms the given finished span. The returned span (typically the same object,
     * mutated in place) continues through the pipeline.
     * @param span a finished span to transform
     * @return the transformed span
     */
    FinishedSpan map(FinishedSpan span);

}
```

### `simple-tracing-api/src/main/java/io/simpletracing/exporter/SpanExportingPredicate.java` [NEW]

```java
package io.simpletracing.exporter;

/**
 * Decides whether a {@link FinishedSpan} should be exported. This is the "gate" stage
 * of the export pipeline — spans that fail the predicate are silently dropped.
 *
 * <p>Multiple predicates can be composed: a span is exported only if ALL predicates
 * return {@code true}.
 *
 * <p>Maps to: {@code io.micrometer.tracing.exporter.SpanExportingPredicate}
 *
 * @see FinishedSpan
 * @see SpanFilter
 * @see SpanReporter
 */
@FunctionalInterface
public interface SpanExportingPredicate {

    /**
     * Determines whether the given finished span should be exported.
     * @param span the finished span to evaluate
     * @return {@code true} if the span should be exported, {@code false} to drop it
     */
    boolean isExportable(FinishedSpan span);

}
```

### `simple-tracing-api/src/main/java/io/simpletracing/exporter/SpanReporter.java` [NEW]

```java
package io.simpletracing.exporter;

/**
 * Reports finished spans to an external system (e.g., Zipkin, a logging backend, or
 * an in-memory collector for tests). This is the terminal stage of the export pipeline.
 *
 * <p>Extends {@link AutoCloseable} so that reporters holding resources (HTTP connections,
 * buffers, file handles) can be cleaned up. The default {@link #close()} is a no-op.
 *
 * <p>Maps to: {@code io.micrometer.tracing.exporter.SpanReporter}
 *
 * @see FinishedSpan
 * @see SpanFilter
 * @see SpanExportingPredicate
 */
public interface SpanReporter extends AutoCloseable {

    /**
     * A no-op reporter that silently discards all spans.
     */
    SpanReporter NOOP = span -> {
    };

    /**
     * Reports the given finished span. Called after all filters and predicates have
     * been applied.
     * @param finishedSpan a finished span ready for export
     */
    void report(FinishedSpan finishedSpan);

    /**
     * Closes this reporter, releasing any held resources. Default is a no-op.
     */
    @Override
    default void close() {
    }

}
```

### `simple-tracing-api/src/test/java/io/simpletracing/exporter/ExportPipelineTest.java` [NEW]

```java
package io.simpletracing.exporter;

import java.time.Instant;
import java.util.ArrayList;
import java.util.Collection;
import java.util.HashMap;
import java.util.LinkedHashMap;
import java.util.List;
import java.util.Map;

import io.simpletracing.Span;
import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;

/**
 * Client-perspective tests for Feature 2: Span Export Pipeline.
 *
 * <p>These tests verify the three stages of the export pipeline from a client's
 * point of view: filters transform spans, predicates gate export, and reporters
 * receive the final result. A minimal {@link TestFinishedSpan} is used as the
 * data model.
 */
class ExportPipelineTest {

    // ── SpanReporter Tests ────────────────────────────────────────────────

    @Test
    void reporter_shouldReceiveFinishedSpan() {
        List<FinishedSpan> reported = new ArrayList<>();
        SpanReporter reporter = reported::add;

        FinishedSpan span = TestFinishedSpan.create("my-operation");
        reporter.report(span);

        assertThat(reported).hasSize(1);
        assertThat(reported.get(0).getName()).isEqualTo("my-operation");
    }

    @Test
    void noopReporter_shouldNotThrow() {
        FinishedSpan span = TestFinishedSpan.create("noop-test");
        SpanReporter.NOOP.report(span); // should not throw
    }

    // ── SpanFilter Tests ──────────────────────────────────────────────────

    @Test
    void filter_shouldTransformSpanName() {
        SpanFilter uppercaseFilter = span -> span.setName(span.getName().toUpperCase());

        FinishedSpan span = TestFinishedSpan.create("my-operation");
        FinishedSpan result = uppercaseFilter.map(span);

        assertThat(result.getName()).isEqualTo("MY-OPERATION");
    }

    @Test
    void filter_shouldAddTag() {
        SpanFilter tagAdder = span -> {
            Map<String, String> tags = new HashMap<>(span.getTags());
            tags.put("filtered", "true");
            return span.setTags(tags);
        };

        FinishedSpan span = TestFinishedSpan.create("tagged-op");
        span.setTags(Map.of("original", "yes"));
        FinishedSpan result = tagAdder.map(span);

        assertThat(result.getTags()).containsEntry("original", "yes");
        assertThat(result.getTags()).containsEntry("filtered", "true");
    }

    @Test
    void multipleFilters_shouldChainInOrder() {
        SpanFilter addPrefix = span -> span.setName("prefix-" + span.getName());
        SpanFilter addSuffix = span -> span.setName(span.getName() + "-suffix");

        FinishedSpan span = TestFinishedSpan.create("operation");

        // Apply filters in order: prefix first, then suffix
        FinishedSpan result = span;
        for (SpanFilter filter : List.of(addPrefix, addSuffix)) {
            result = filter.map(result);
        }

        assertThat(result.getName()).isEqualTo("prefix-operation-suffix");
    }

    // ── SpanExportingPredicate Tests ──────────────────────────────────────

    @Test
    void predicate_shouldAllowExport() {
        SpanExportingPredicate allowAll = span -> true;

        FinishedSpan span = TestFinishedSpan.create("allowed-op");
        assertThat(allowAll.isExportable(span)).isTrue();
    }

    @Test
    void predicate_shouldBlockExport() {
        SpanExportingPredicate blockInternal = span -> !span.getName().startsWith("internal-");

        FinishedSpan internal = TestFinishedSpan.create("internal-healthcheck");
        FinishedSpan external = TestFinishedSpan.create("http-request");

        assertThat(blockInternal.isExportable(internal)).isFalse();
        assertThat(blockInternal.isExportable(external)).isTrue();
    }

    @Test
    void multiplePredicates_shouldRequireAll() {
        SpanExportingPredicate notInternal = span -> !span.getName().startsWith("internal-");
        SpanExportingPredicate hasKind = span -> span.getKind() != null;

        FinishedSpan serverSpan = TestFinishedSpan.builder("http-request")
                .kind(Span.Kind.SERVER)
                .build();
        FinishedSpan noKindSpan = TestFinishedSpan.create("http-request");
        FinishedSpan internalSpan = TestFinishedSpan.builder("internal-check")
                .kind(Span.Kind.CLIENT)
                .build();

        List<SpanExportingPredicate> predicates = List.of(notInternal, hasKind);

        // Server span passes both predicates
        assertThat(predicates.stream().allMatch(p -> p.isExportable(serverSpan))).isTrue();
        // No-kind span fails the second predicate
        assertThat(predicates.stream().allMatch(p -> p.isExportable(noKindSpan))).isFalse();
        // Internal span fails the first predicate
        assertThat(predicates.stream().allMatch(p -> p.isExportable(internalSpan))).isFalse();
    }

    // ── Full Pipeline Tests: Filter → Predicate → Reporter ───────────────

    @Test
    void fullPipeline_shouldFilterThenGateThenReport() {
        // Setup: filter uppercases names, predicate blocks "INTERNAL-*", reporter collects
        SpanFilter uppercaseFilter = span -> span.setName(span.getName().toUpperCase());
        SpanExportingPredicate blockInternal = span -> !span.getName().startsWith("INTERNAL-");
        List<FinishedSpan> reported = new ArrayList<>();
        SpanReporter reporter = reported::add;

        // Three spans: one normal, one internal, one with error
        List<FinishedSpan> spans = List.of(
                TestFinishedSpan.create("http-request"),
                TestFinishedSpan.create("internal-healthcheck"),
                TestFinishedSpan.create("db-query")
        );

        // Run the pipeline: filter → predicate → report
        for (FinishedSpan span : spans) {
            FinishedSpan filtered = uppercaseFilter.map(span);
            if (blockInternal.isExportable(filtered)) {
                reporter.report(filtered);
            }
        }

        // Only http-request and db-query should be reported (internal was blocked)
        assertThat(reported).hasSize(2);
        assertThat(reported.get(0).getName()).isEqualTo("HTTP-REQUEST");
        assertThat(reported.get(1).getName()).isEqualTo("DB-QUERY");
    }

    @Test
    void fullPipeline_withMultipleFiltersAndPredicates() {
        // Filters: add environment tag, then uppercase the name
        SpanFilter addEnvTag = span -> {
            Map<String, String> tags = new HashMap<>(span.getTags());
            tags.put("env", "production");
            return span.setTags(tags);
        };
        SpanFilter uppercaseName = span -> span.setName(span.getName().toUpperCase());

        // Predicates: must have a kind, must not be internal
        SpanExportingPredicate hasKind = span -> span.getKind() != null;
        SpanExportingPredicate notInternal = span -> !span.getName().startsWith("INTERNAL-");

        List<FinishedSpan> reported = new ArrayList<>();
        SpanReporter reporter = reported::add;

        List<SpanFilter> filters = List.of(addEnvTag, uppercaseName);
        List<SpanExportingPredicate> predicates = List.of(hasKind, notInternal);

        // Input spans
        List<FinishedSpan> spans = List.of(
                TestFinishedSpan.builder("http-request").kind(Span.Kind.SERVER).build(),
                TestFinishedSpan.builder("internal-check").kind(Span.Kind.CLIENT).build(),
                TestFinishedSpan.create("no-kind-span")
        );

        // Pipeline: apply all filters, then check all predicates, then report
        for (FinishedSpan span : spans) {
            FinishedSpan filtered = span;
            for (SpanFilter f : filters) {
                filtered = f.map(filtered);
            }
            FinishedSpan result = filtered;
            if (predicates.stream().allMatch(p -> p.isExportable(result))) {
                reporter.report(result);
            }
        }

        // Only "http-request" passes both predicates (has kind + not internal)
        assertThat(reported).hasSize(1);
        assertThat(reported.get(0).getName()).isEqualTo("HTTP-REQUEST");
        assertThat(reported.get(0).getTags()).containsEntry("env", "production");
    }

    // ── FinishedSpan Data Model Tests ─────────────────────────────────────

    @Test
    void finishedSpan_shouldExposeAllFields() {
        Throwable error = new RuntimeException("boom");
        FinishedSpan span = TestFinishedSpan.builder("db-query")
                .traceId("abc123")
                .spanId("def456")
                .parentId("parent789")
                .kind(Span.Kind.CLIENT)
                .remoteServiceName("postgres")
                .error(error)
                .tag("db.type", "sql")
                .event(1000L, "query-started")
                .build();

        assertThat(span.getName()).isEqualTo("db-query");
        assertThat(span.getTraceId()).isEqualTo("abc123");
        assertThat(span.getSpanId()).isEqualTo("def456");
        assertThat(span.getParentId()).isEqualTo("parent789");
        assertThat(span.getKind()).isEqualTo(Span.Kind.CLIENT);
        assertThat(span.getRemoteServiceName()).isEqualTo("postgres");
        assertThat(span.getError()).isSameAs(error);
        assertThat(span.getTags()).containsEntry("db.type", "sql");
        assertThat(span.getEvents()).hasSize(1);
    }

    @Test
    void finishedSpan_shouldBeFluentlyMutable() {
        FinishedSpan span = TestFinishedSpan.create("original");

        span.setName("renamed")
                .setTags(Map.of("new-tag", "value"))
                .setError(new RuntimeException("oops"))
                .setRemoteServiceName("new-service");

        assertThat(span.getName()).isEqualTo("renamed");
        assertThat(span.getTags()).containsEntry("new-tag", "value");
        assertThat(span.getError()).hasMessage("oops");
        assertThat(span.getRemoteServiceName()).isEqualTo("new-service");
    }

    @Test
    void finishedSpan_durationShouldBeComputed() {
        Instant start = Instant.parse("2024-01-01T00:00:00Z");
        Instant end = Instant.parse("2024-01-01T00:00:01.500Z");
        FinishedSpan span = TestFinishedSpan.builder("timed-op")
                .startTimestamp(start)
                .endTimestamp(end)
                .build();

        assertThat(span.getDuration().toMillis()).isEqualTo(1500);
    }

    // ── TestFinishedSpan: Minimal in-memory implementation ───────────────

    /**
     * Mutable in-memory implementation of {@link FinishedSpan} for testing.
     * In a real application, the tracer bridge creates these when a span ends.
     */
    static class TestFinishedSpan implements FinishedSpan {

        private String name;
        private Instant startTimestamp;
        private Instant endTimestamp;
        private Map<String, String> tags;
        private Collection<Map.Entry<Long, String>> events;
        private final String spanId;
        private final String parentId;
        private final String traceId;
        private Throwable error;
        private final Span.Kind kind;
        private String remoteServiceName;

        private TestFinishedSpan(Builder builder) {
            this.name = builder.name;
            this.startTimestamp = builder.startTimestamp;
            this.endTimestamp = builder.endTimestamp;
            this.tags = new LinkedHashMap<>(builder.tags);
            this.events = new ArrayList<>(builder.events);
            this.spanId = builder.spanId;
            this.parentId = builder.parentId;
            this.traceId = builder.traceId;
            this.error = builder.error;
            this.kind = builder.kind;
            this.remoteServiceName = builder.remoteServiceName;
        }

        static TestFinishedSpan create(String name) {
            return builder(name).build();
        }

        static Builder builder(String name) {
            return new Builder(name);
        }

        @Override public String getName() { return name; }
        @Override public FinishedSpan setName(String name) { this.name = name; return this; }
        @Override public Instant getStartTimestamp() { return startTimestamp; }
        @Override public Instant getEndTimestamp() { return endTimestamp; }
        @Override public Map<String, String> getTags() { return tags; }
        @Override public FinishedSpan setTags(Map<String, String> tags) {
            this.tags = new LinkedHashMap<>(tags); return this;
        }
        @Override public Collection<Map.Entry<Long, String>> getEvents() { return events; }
        @Override public FinishedSpan setEvents(Collection<Map.Entry<Long, String>> events) {
            this.events = new ArrayList<>(events); return this;
        }
        @Override public String getSpanId() { return spanId; }
        @Override public String getParentId() { return parentId; }
        @Override public String getTraceId() { return traceId; }
        @Override public Throwable getError() { return error; }
        @Override public FinishedSpan setError(Throwable error) {
            this.error = error; return this;
        }
        @Override public Span.Kind getKind() { return kind; }
        @Override public String getRemoteServiceName() { return remoteServiceName; }
        @Override public FinishedSpan setRemoteServiceName(String remoteServiceName) {
            this.remoteServiceName = remoteServiceName; return this;
        }

        static class Builder {

            private final String name;
            private Instant startTimestamp = Instant.now();
            private Instant endTimestamp = Instant.now();
            private final Map<String, String> tags = new LinkedHashMap<>();
            private final List<Map.Entry<Long, String>> events = new ArrayList<>();
            private String spanId = "span-001";
            private String parentId;
            private String traceId = "trace-001";
            private Throwable error;
            private Span.Kind kind;
            private String remoteServiceName;

            Builder(String name) { this.name = name; }
            Builder startTimestamp(Instant v) { this.startTimestamp = v; return this; }
            Builder endTimestamp(Instant v) { this.endTimestamp = v; return this; }
            Builder tag(String k, String v) { this.tags.put(k, v); return this; }
            Builder event(long ts, String v) { this.events.add(Map.entry(ts, v)); return this; }
            Builder spanId(String v) { this.spanId = v; return this; }
            Builder parentId(String v) { this.parentId = v; return this; }
            Builder traceId(String v) { this.traceId = v; return this; }
            Builder error(Throwable v) { this.error = v; return this; }
            Builder kind(Span.Kind v) { this.kind = v; return this; }
            Builder remoteServiceName(String v) { this.remoteServiceName = v; return this; }
            TestFinishedSpan build() { return new TestFinishedSpan(this); }
        }

    }

}
```
