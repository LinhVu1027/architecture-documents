# Chapter 7: Test Assertions — AssertJ-Style Span Verification

> **What you'll build**: Four fluent assertion classes — `SpanAssert`, `SpansAssert`,
> `TracerAssert`, and `TracingAssertions` — that let tests verify tracing behavior with
> expressive, domain-specific assertions and informative error messages.

---

## 1. The API Contract

After this chapter, a client can write:

```java
SimpleTracer tracer = new SimpleTracer();
// ... exercise production code that uses Tracer ...

// Fluent assertions on a single span
SpanAssert.assertThat(tracer.onlySpan())
    .hasNameEqualTo("encode")
    .hasTagWithKey("input.length")
    .hasTag("input.length", "256")
    .hasEventWithNameEqualTo("encoding-started")
    .hasNoErrors()
    .isStarted()
    .isEnded();

// Fluent assertions on all spans
SpansAssert.assertThat(tracer.getSpans())
    .hasSize(3)
    .hasASpanWithName("encode")
    .hasASpanWithNameEqualTo("decode")
    .haveSameTraceId();

// Fluent assertions on the tracer itself
TracerAssert.assertThat(tracer)
    .onlySpan()
    .hasNameEqualTo("encode");

// Or use the unified entry point
import static io.simpletracing.test.simple.TracingAssertions.*;
assertThat(tracer).onlySpan().hasNameEqualTo("encode");
```

**Why this matters**: Chapter 6 gave us `SimpleTracer` — an in-memory tracer that collects
spans. But to verify those spans in tests, clients had to write verbose assertions like
`assertThat(span.getName()).isEqualTo("encode")`. This chapter provides **domain-specific
assertion methods** that are more readable, more expressive, and produce better error messages
when they fail.

### The Assertion Architecture

```
TracingAssertions (static entry point — dispatches by type)
├── assertThat(FinishedSpan)   → SpanAssert
├── assertThat(Collection)     → SpansAssert
└── assertThat(SimpleTracer)   → TracerAssert
                                   ├── onlySpan()  → SpanAssert
                                   ├── lastSpan()   → SpanAssert
                                   └── reportedSpans() → AssertJ CollectionAssert

SpanAssert extends AbstractAssert<SELF, FinishedSpan>
├── Name: hasNameEqualTo, doesNotHaveNameEqualTo
├── Tags: hasTag, hasTagWithKey, hasNoTags, doesNotHaveTag
├── Events: hasEventWithNameEqualTo, doesNotHaveEventWithNameEqualTo
├── Lifecycle: isStarted, isNotStarted, isEnded, isNotEnded
├── Errors: hasNoErrors, assertThatThrowable → SpanAssertReturningAssert
├── Kind: hasKindEqualTo, doesNotHaveKindEqualTo
├── Identity: hasSpanIdEqualTo, hasParentIdEqualTo, hasTraceIdEqualTo
├── Remote: hasRemoteServiceNameEqualTo, doesNotHaveRemoteServiceNameEqualTo
└── Links: hasLink, doesNotHaveLink

SpansAssert extends CollectionAssert<FinishedSpan>
├── (inherits: hasSize, isEmpty, contains, etc.)
├── Trace: haveSameTraceId
├── Name: hasASpanWithName, hasASpanWithNameIgnoreCase
├── Count: hasNumberOfSpansEqualTo, hasNumberOfSpansWithNameEqualTo
├── Tags: hasASpanWithATag, hasASpanWithATagKey
├── Remote: hasASpanWithRemoteServiceName
├── Navigation: assertThatASpanWithNameEqualTo → SpansAssertReturningAssert
└── ForAll: forAllSpansWithNameEqualTo
```

**4 classes, 0 new interfaces** — this feature adds no new API interfaces. It builds
entirely on top of the `FinishedSpan` interface (Feature 2) and the `SimpleTracer`/`SimpleSpan`
implementations (Feature 6).

### The Returning Assert Pattern

The most interesting design pattern: **inner classes that enable navigation between assertion
scopes**. For example, `SpansAssert.assertThatASpanWithNameEqualTo("encode")` returns a
`SpansAssertReturningAssert` — which extends `SpanAssert` (giving you all single-span
assertions) but adds a `backToSpans()` method to return to the collection context:

```java
SpansAssert.assertThat(spans)
    .assertThatASpanWithNameEqualTo("encode")  // → SpansAssertReturningAssert
        .hasTag("input.length", "256")          // ← SpanAssert method
        .isEnded()                              // ← SpanAssert method
    .backToSpans()                              // → back to SpansAssert
    .hasASpanWithName("decode");                // ← SpansAssert method
```

This is possible because `SpanAssert<SELF>` uses a self-referencing type parameter. When
`SpansAssertReturningAssert extends SpanAssert<SpansAssertReturningAssert>`, all inherited
assertion methods return `SpansAssertReturningAssert` instead of raw `SpanAssert`, keeping
the `backToSpans()` method visible in the fluent chain.

### Module Placement

All classes go in the `simple-tracing-test` module (`io.simpletracing.test.simple` package),
alongside the `SimpleTracer` and `SimpleSpan` they assert on. This mirrors the real framework
where the assertion utilities live in the same test support module.

### Modification to Existing Code

This feature adds one method to the `FinishedSpan` interface from Feature 2:

```java
// In io.simpletracing.exporter.FinishedSpan
default List<Link> getLinks() {
    return Collections.emptyList();
}
```

This is a **default method** — no existing code breaks. The real `FinishedSpan` has this
method; we deferred it in Feature 2 as a simplification. Now `SpanAssert.hasLink()` needs
it, so we add it. `SimpleSpan.getLinks()` already returns `List<Link>` and now overrides
this default.

---

## 2. Client Usage & Tests

### Test: Fluent assertion chain on a single span

The most common usage — verify everything about a span in a single fluent chain:

```java
@Test
void shouldSupportFluentChaining() {
    SimpleSpan span = createFinishedSpan("encode");
    span.tag("input.length", "256");
    span.event("encoding-started");

    SpanAssert.assertThat(span)
            .hasNameEqualTo("encode")
            .hasTagWithKey("input.length")
            .hasTag("input.length", "256")
            .hasEventWithNameEqualTo("encoding-started")
            .hasNoErrors()
            .isStarted()
            .isEnded();
}
```

### Test: Throwable assertions with back-navigation

Asserting on a span's error and then returning to span-level assertions:

```java
@Test
void shouldSupportThrowableAssertions() {
    SimpleSpan span = createFinishedSpan("encode");
    span.error(new IllegalStateException("encoding failed"));

    SpanAssert.assertThat(span)
            .assertThatThrowable()
                .isInstanceOf(IllegalStateException.class)
                .hasMessage("encoding failed")
            .backToSpan()
            .hasNameEqualTo("encode");
}
```

### Test: Collection assertions with span drilling

Navigate from collection-level to span-level assertions and back:

```java
@Test
void shouldSupportAssertThatASpanWithNameNavigation() {
    createFinishedSpanViaTracer("encode");
    createFinishedSpanViaTracer("decode");

    SpansAssert.assertThat(castSpans(tracer.getSpans()))
            .assertThatASpanWithNameEqualTo("encode")
                .isStarted()
                .isEnded()
            .backToSpans()
            .hasASpanWithName("decode");
}
```

### Test: Tracer-level assertions

Start from the tracer and navigate to span assertions:

```java
@Test
void shouldNavigateToOnlySpan() {
    createFinishedSpanViaTracer("encode");

    TracerAssert.assertThat(tracer)
            .onlySpan()
            .hasNameEqualTo("encode");
}
```

### Test: Verify that assertions fail correctly

Testing the test utilities — the assertion should fail with `AssertionError`:

```java
@Test
void shouldFailWhenNameDoesNotMatch() {
    SimpleSpan span = createFinishedSpan("encode");
    assertThatThrownBy(() -> SpanAssert.assertThat(span).hasNameEqualTo("decode"))
            .isInstanceOf(AssertionError.class);
}
```

### Test: Same trace ID across parent-child spans

```java
@Test
void shouldPassWhenAllSpansHaveSameTraceId() {
    Span parent = tracer.nextSpan().name("parent").start();
    try (Tracer.SpanInScope ws = tracer.withSpan(parent)) {
        Span child = tracer.nextSpan().name("child").start();
        child.end();
    } finally {
        parent.end();
    }

    SpansAssert.assertThat(castSpans(tracer.getSpans()))
            .haveSameTraceId();
}
```

---

## 3. Implementing the Call Chain — Top-Down

### Layer 1: SpanAssert — The Core Single-Span Assertion

`SpanAssert` extends AssertJ's `AbstractAssert<SELF, FinishedSpan>`. The `SELF` type parameter
is the key to the returning assert pattern:

```java
@SuppressWarnings({ "unchecked", "rawtypes" })
public class SpanAssert<SELF extends SpanAssert<SELF>>
        extends AbstractAssert<SELF, FinishedSpan> {

    protected SpanAssert(FinishedSpan actual) {
        super(actual, SpanAssert.class);
    }
```

Each assertion method follows the same pattern:
1. Call `isNotNull()` — inherited from `AbstractAssert`, verifies the span isn't null
2. Check the condition on `this.actual` (the `FinishedSpan`)
3. Call `failWithMessage()` with a descriptive message if the condition fails
4. Return `(SELF) this` for fluent chaining

Example — tag assertion that also delegates to `hasTagWithKey`:

```java
public SELF hasTag(String key, String value) {
    isNotNull();
    hasTagWithKey(key);  // reuse — first verify key exists
    String tagValue = this.actual.getTags().get(key);
    Assertions.assertThat(tagValue).isNotNull();
    if (!tagValue.equals(value)) {
        failWithMessage(
            "Span should have a tag with key <%s> and value <%s>. "
            + "The key is correct but the value is <%s>",
            key, value, tagValue);
    }
    return (SELF) this;
}
```

The `SpanAssertReturningAssert` inner class extends `AbstractThrowableAssert` and holds a
reference back to the parent `SpanAssert`:

```java
public static class SpanAssertReturningAssert
        extends AbstractThrowableAssert<SpanAssertReturningAssert, Throwable> {

    private final SpanAssert spanAssert;

    public SpanAssertReturningAssert(Throwable throwable, SpanAssert spanAssert) {
        super(throwable, SpanAssertReturningAssert.class);
        this.spanAssert = spanAssert;
    }

    public SpanAssert backToSpan() {
        return this.spanAssert;
    }
}
```

### Layer 2: SpansAssert — Collection-Level Assertions

`SpansAssert` extends AssertJ's `CollectionAssert<FinishedSpan>`. This gives clients all
standard collection assertions (`hasSize`, `isEmpty`, `contains`) for free:

```java
public class SpansAssert extends CollectionAssert<FinishedSpan> {

    protected SpansAssert(Collection<? extends FinishedSpan> actual) {
        super(actual);
    }
```

The `hasASpanWithName(String, Consumer<SpanAssert>)` method is particularly interesting — it
finds spans by name, applies a custom assertion, and catches `AssertionError` to try the next:

```java
public SpansAssert hasASpanWithName(String name, Consumer<SpanAssert> spanConsumer) {
    isNotEmpty();
    FinishedSpan match = this.actual.stream()
            .filter(f -> name.equals(f.getName()))
            .filter(f -> {
                try {
                    spanConsumer.accept(SpanAssert.assertThat(f));
                    return true;
                } catch (AssertionError e) {
                    return false;
                }
            })
            .findFirst().orElse(null);
    if (match == null) {
        failWithMessage("Not a single span with name <%s> was found or has passed "
            + "the assertion. Found following spans %s", name, spansAsString());
    }
    return this;
}
```

The `SpansAssertReturningAssert` extends `SpanAssert<SpansAssertReturningAssert>` — this is
where the self-referencing type parameter pays off. It inherits all 30+ span assertion methods
with the correct return type, and adds `backToSpans()`:

```java
public static class SpansAssertReturningAssert
        extends SpanAssert<SpansAssertReturningAssert> {

    private final SpansAssert spansAssert;

    public SpansAssertReturningAssert(SpansAssert spansAssert, FinishedSpan span) {
        super(span);
        this.spansAssert = spansAssert;
    }

    public SpansAssert backToSpans() {
        return this.spansAssert;
    }
}
```

### Layer 3: TracerAssert — Tracer-Level Bridge

`TracerAssert` is the simplest class — it asserts on `SimpleTracer` and navigates to
`SpanAssert` or standard AssertJ collection assertions:

```java
public class TracerAssert extends AbstractAssert<TracerAssert, SimpleTracer> {

    public SpanAssert onlySpan() {
        isNotNull();
        return SpanAssert.assertThat(this.actual.onlySpan());
    }

    public SpanAssert lastSpan() {
        isNotNull();
        return SpanAssert.assertThat(this.actual.lastSpan());
    }

    public AbstractCollectionAssert<?, Collection<? extends SimpleSpan>,
            SimpleSpan, ObjectAssert<SimpleSpan>> reportedSpans() {
        return Assertions.assertThat(this.actual.getSpans());
    }
}
```

### Layer 4: TracingAssertions — Static Entry Point

`TracingAssertions` provides a single import for all assertion types:

```java
public class TracingAssertions {
    public static SpanAssert assertThat(FinishedSpan actual) { ... }
    public static SpansAssert assertThat(Collection<FinishedSpan> actual) { ... }
    public static TracerAssert assertThat(SimpleTracer actual) { ... }
    // + then() aliases for BDD style
}
```

---

## 4. Try It Yourself

1. **Add `hasErrorOfType(Class<?>)`**: Add an assertion to `SpanAssert` that checks if the
   span's error is of a specific exception class. Hint: use `isInstanceOf` logic on
   `this.actual.getError()`.

2. **Add `hasASpanWithEvent(String)`**: Add an assertion to `SpansAssert` that verifies at
   least one span in the collection has an event with the given name. Hint: use
   `eventNames()` logic from `SpanAssert`.

3. **Add `doesNotHaveASpanWithName(String)`**: The inverse of `hasASpanWithName` — verify
   no span has the given name.

---

## 5. Why This Works

### AbstractAssert Gives You Framework for Free
By extending `AbstractAssert`, every method inherits `as()` (description), `overridingErrorMessage()`,
`isNotNull()`, and the `WritableAssertionInfo` system. Clients can write
`SpanAssert.assertThat(span).as("auth span").hasNameEqualTo("auth")` and get the description
in the error message — we didn't write any of that code.

### The SELF Type Parameter Enables the Returning Assert Pattern
Without `SELF`, `hasNameEqualTo()` would return `SpanAssert` — losing the `backToSpans()` method
when called from `SpansAssertReturningAssert`. With `SELF extends SpanAssert<SELF>`, each
subclass's methods return the subclass type, keeping navigation methods visible throughout the
fluent chain. This is AssertJ's recommended pattern for extensible assertions.

### CollectionAssert Gives You Standard Assertions for Free
`SpansAssert extends CollectionAssert<FinishedSpan>` means clients get `hasSize()`, `isEmpty()`,
`contains()`, `filteredOn()`, and dozens of other methods without us writing a single line. We
only add what's domain-specific: `hasASpanWithName()`, `haveSameTraceId()`, etc.

---

## 6. What We Enhanced

| Component | State After Ch07 | Simplifications |
|-----------|-------------------|-----------------|
| `SpanAssert` | Full single-span assertion with name, tags, events, lifecycle, errors, kind, identity, remote service, links | No `KeyName` overloads (no micrometer-commons dependency); no IP/port assertions |
| `SpansAssert` | Full collection assertion with name search, trace correlation, counting, tags, remote service, navigation | No `KeyName` overloads |
| `TracerAssert` | Tracer-level bridge to SpanAssert and CollectionAssert | Identical to real framework |
| `TracingAssertions` | Static entry point for all assertion types | Identical to real framework |
| `FinishedSpan` (modified) | Added `getLinks()` default method | Was deferred in Feature 2; now needed for link assertions |

**What this feature enables**: Tests can now use expressive, domain-specific assertions instead
of raw `assertThat(span.getName()).isEqualTo("encode")`. Error messages are contextualized
(e.g., "Span should have a tag with key <foo> but it's not there. List of all keys <[bar, baz]>").
Feature 8 (Brave Bridge) will use these assertions in its integration tests.

---

## 7. Connection to Real Framework

| Simplified Class | Real Framework Class | Source File |
|------------------|----------------------|-------------|
| `io.simpletracing.test.simple.SpanAssert` | `io.micrometer.tracing.test.simple.SpanAssert` | `micrometer-tracing-test/src/main/java/.../SpanAssert.java` |
| `io.simpletracing.test.simple.SpansAssert` | `io.micrometer.tracing.test.simple.SpansAssert` | `micrometer-tracing-test/src/main/java/.../SpansAssert.java` |
| `io.simpletracing.test.simple.TracerAssert` | `io.micrometer.tracing.test.simple.TracerAssert` | `micrometer-tracing-test/src/main/java/.../TracerAssert.java` |
| `io.simpletracing.test.simple.TracingAssertions` | `io.micrometer.tracing.test.simple.TracingAssertions` | `micrometer-tracing-test/src/main/java/.../TracingAssertions.java` |

**Real framework extras we skipped**:
- `KeyName` overloads (`hasTag(KeyName, String)`, `hasASpanWithATag(KeyName, String)`, etc.) — these use `io.micrometer.common.docs.KeyName` for type-safe observation key references; we use plain strings
- IP/port assertions (`hasIpEqualTo`, `hasPortEqualTo`, `hasIpThatIsNotBlank`, etc.) — our simplified `FinishedSpan` doesn't track remote IP or port
- `@Nullable` annotations from `org.jspecify.annotations` — we skip the nullability annotation dependency
- `StringUtils.isBlank`/`isNotBlank` from `io.micrometer.common.util` — used for IP blank checks

---

## 8. Complete Code

### `simple-tracing-api/src/main/java/io/simpletracing/exporter/FinishedSpan.java` [MODIFIED]

```java
package io.simpletracing.exporter;

import java.time.Duration;
import java.time.Instant;
import java.util.Collection;
import java.util.Collections;
import java.util.List;
import java.util.Map;

import io.simpletracing.Link;
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
     * Returns the links associated with this span. Links connect spans across different
     * traces, commonly used in batch processing scenarios.
     * @return list of links (never null, may be empty)
     */
    default List<Link> getLinks() {
        return Collections.emptyList();
    }

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

### `simple-tracing-test/src/main/java/io/simpletracing/test/simple/SpanAssert.java` [NEW]

```java
package io.simpletracing.test.simple;

import java.util.List;
import java.util.Map;
import java.util.Objects;
import java.util.function.Consumer;
import java.util.stream.Collectors;

import io.simpletracing.Link;
import io.simpletracing.Span;
import io.simpletracing.exporter.FinishedSpan;
import org.assertj.core.api.AbstractAssert;
import org.assertj.core.api.AbstractThrowableAssert;
import org.assertj.core.api.Assertions;

/**
 * AssertJ-style fluent assertions for a single {@link FinishedSpan}. Provides
 * domain-specific assertion methods that produce clear, informative error messages
 * when a span's state doesn't match expectations.
 *
 * <p>Usage:
 * <pre>{@code
 * SpanAssert.assertThat(tracer.onlySpan())
 *     .hasNameEqualTo("encode")
 *     .hasTagWithKey("input.length")
 *     .hasEventWithNameEqualTo("encoding-started")
 *     .hasNoErrors()
 *     .isStarted()
 *     .isEnded();
 * }</pre>
 *
 * <p>The {@code SELF} type parameter enables the "returning assert" pattern — inner
 * classes like {@link SpanAssertReturningAssert} extend this class and add navigation
 * methods (e.g., {@code backToSpan()}) while inheriting all assertion methods with
 * correct return types.
 *
 * <p>Maps to: {@code io.micrometer.tracing.test.simple.SpanAssert}
 *
 * @param <SELF> self type for fluent chaining
 */
@SuppressWarnings({ "unchecked", "rawtypes" })
public class SpanAssert<SELF extends SpanAssert<SELF>> extends AbstractAssert<SELF, FinishedSpan> {

    protected SpanAssert(FinishedSpan actual) {
        super(actual, SpanAssert.class);
    }

    /**
     * Creates the assert object for {@link FinishedSpan}.
     * @param actual span to assert against
     * @return span assertions
     */
    public static SpanAssert assertThat(FinishedSpan actual) {
        return new SpanAssert(actual);
    }

    /**
     * Creates the assert object for {@link FinishedSpan}. BDD-style alias for
     * {@link #assertThat(FinishedSpan)}.
     * @param actual span to assert against
     * @return span assertions
     */
    public static SpanAssert then(FinishedSpan actual) {
        return new SpanAssert(actual);
    }

    // ── Name assertions ─────────────────────────────────────────────────

    public SELF hasNameEqualTo(String spanName) {
        isNotNull();
        if (!this.actual.getName().equals(spanName)) {
            failWithMessage("Span should have a name <%s> but has <%s>", spanName, this.actual.getName());
        }
        return (SELF) this;
    }

    public SELF doesNotHaveNameEqualTo(String spanName) {
        isNotNull();
        if (this.actual.getName().equals(spanName)) {
            failWithMessage("Span should not have a name <%s>", spanName);
        }
        return (SELF) this;
    }

    // ── Tag assertions ──────────────────────────────────────────────────

    public SELF hasNoTags() {
        isNotNull();
        Map<String, String> tags = this.actual.getTags();
        if (!tags.isEmpty()) {
            failWithMessage("Span should have no tags but has <%s>", tags);
        }
        return (SELF) this;
    }

    public SELF hasTagWithKey(String key) {
        isNotNull();
        if (!this.actual.getTags().containsKey(key)) {
            failWithMessage("Span should have a tag with key <%s> but it's not there. List of all keys <%s>", key,
                    this.actual.getTags().keySet());
        }
        return (SELF) this;
    }

    public SELF hasTag(String key, String value) {
        isNotNull();
        hasTagWithKey(key);
        String tagValue = this.actual.getTags().get(key);
        Assertions.assertThat(tagValue).isNotNull();
        if (!tagValue.equals(value)) {
            failWithMessage(
                    "Span should have a tag with key <%s> and value <%s>. The key is correct but the value is <%s>",
                    key, value, tagValue);
        }
        return (SELF) this;
    }

    public SELF doesNotHaveTagWithKey(String key) {
        isNotNull();
        if (this.actual.getTags().containsKey(key)) {
            failWithMessage("Span should not have a tag with key <%s>", key);
        }
        return (SELF) this;
    }

    public SELF doesNotHaveTag(String key, String value) {
        isNotNull();
        doesNotHaveTagWithKey(key);
        String tagValue = this.actual.getTags().get(key);
        if (value.equals(tagValue)) {
            failWithMessage("Span should not have a tag with key <%s> and value <%s>", key, value);
        }
        return (SELF) this;
    }

    // ── Event assertions ────────────────────────────────────────────────

    public SELF hasEventWithNameEqualTo(String eventName) {
        isNotNull();
        List<String> eventNames = eventNames();
        if (!eventNames.contains(eventName)) {
            failWithMessage("Span should have an event with name <%s> but has <%s>", eventName, eventNames);
        }
        return (SELF) this;
    }

    public SELF doesNotHaveEventWithNameEqualTo(String eventName) {
        isNotNull();
        List<String> eventNames = eventNames();
        if (eventNames.contains(eventName)) {
            failWithMessage("Span should not have an event with name <%s>", eventName);
        }
        return (SELF) this;
    }

    private List<String> eventNames() {
        return this.actual.getEvents().stream().map(Map.Entry::getValue).collect(Collectors.toList());
    }

    // ── Lifecycle assertions ────────────────────────────────────────────

    public SELF isStarted() {
        isNotNull();
        if (this.actual.getStartTimestamp().getEpochSecond() == 0) {
            failWithMessage("Span should be started");
        }
        return (SELF) this;
    }

    public SELF isNotStarted() {
        isNotNull();
        if (this.actual.getStartTimestamp().getEpochSecond() != 0) {
            failWithMessage("Span should not be started");
        }
        return (SELF) this;
    }

    public SELF isEnded() {
        isNotNull();
        if (this.actual.getEndTimestamp().toEpochMilli() == 0) {
            failWithMessage("Span should be ended");
        }
        return (SELF) this;
    }

    public SELF isNotEnded() {
        isNotNull();
        if (this.actual.getEndTimestamp().toEpochMilli() != 0) {
            failWithMessage("Span should not be ended");
        }
        return (SELF) this;
    }

    // ── Error assertions ────────────────────────────────────────────────

    /**
     * Verifies that this span has no error recorded.
     * @return {@code this} assertion object
     */
    public SELF hasNoErrors() {
        isNotNull();
        if (this.actual.getError() != null) {
            failWithMessage("Span should have no errors but has <%s>", this.actual.getError());
        }
        return (SELF) this;
    }

    /**
     * Navigates to a throwable assertion on {@link FinishedSpan#getError()}, with
     * the ability to return back to this span assert via {@link SpanAssertReturningAssert#backToSpan()}.
     * @return throwable assertion with back-navigation
     */
    public SpanAssertReturningAssert assertThatThrowable() {
        return new SpanAssertReturningAssert(actual.getError(), this);
    }

    /**
     * BDD-style alias for {@link #assertThatThrowable()}.
     * @return throwable assertion with back-navigation
     */
    public SpanAssertReturningAssert thenThrowable() {
        return assertThatThrowable();
    }

    // ── Kind assertions ─────────────────────────────────────────────────

    public SELF hasKindEqualTo(Span.Kind kind) {
        isNotNull();
        if (!kind.equals(this.actual.getKind())) {
            failWithMessage("Span should have span kind equal to <%s> but has <%s>", kind, this.actual.getKind());
        }
        return (SELF) this;
    }

    public SELF doesNotHaveKindEqualTo(Span.Kind kind) {
        isNotNull();
        if (kind.equals(this.actual.getKind())) {
            failWithMessage("Span should not have span kind equal to <%s>", kind);
        }
        return (SELF) this;
    }

    // ── Remote service name assertions ──────────────────────────────────

    public SELF hasRemoteServiceNameEqualTo(String remoteServiceName) {
        isNotNull();
        if (!remoteServiceName.equals(this.actual.getRemoteServiceName())) {
            failWithMessage("Span should have remote service name equal to <%s> but has <%s>", remoteServiceName,
                    this.actual.getRemoteServiceName());
        }
        return (SELF) this;
    }

    public SELF doesNotHaveRemoteServiceNameEqualTo(String remoteServiceName) {
        isNotNull();
        if (remoteServiceName.equals(this.actual.getRemoteServiceName())) {
            failWithMessage("Span should not have remote service name equal to <%s>", remoteServiceName);
        }
        return (SELF) this;
    }

    // ── Identity assertions ─────────────────────────────────────────────

    public SELF hasSpanIdEqualTo(String spanId) {
        isNotNull();
        if (!this.actual.getSpanId().equals(spanId)) {
            failWithMessage("Span should have span id equal to <%s> but has <%s>", spanId, this.actual.getSpanId());
        }
        return (SELF) this;
    }

    public SELF doesNotHaveSpanIdEqualTo(String spanId) {
        isNotNull();
        if (this.actual.getSpanId().equals(spanId)) {
            failWithMessage("Span should not have span id equal to <%s>", spanId);
        }
        return (SELF) this;
    }

    public SELF hasParentIdEqualTo(String parentSpanId) {
        isNotNull();
        if (parentSpanId == null) {
            if (this.actual.getParentId() == null) {
                return (SELF) this;
            }
            failWithMessage("Span should have parent span id equal to <null> but has <%s>",
                    this.actual.getParentId());
        }
        if (!Objects.equals(parentSpanId, this.actual.getParentId())) {
            failWithMessage("Span should have parent span id equal to <%s> but has <%s>", parentSpanId,
                    this.actual.getParentId());
        }
        return (SELF) this;
    }

    public SELF doesNotHaveParentIdEqualTo(String parentSpanId) {
        isNotNull();
        if (parentSpanId == null) {
            if (this.actual.getParentId() != null) {
                return (SELF) this;
            }
            failWithMessage("Span should not have parent span id equal to <null>");
        }
        if (Objects.equals(parentSpanId, this.actual.getParentId())) {
            failWithMessage("Span should not have parent span id equal to <%s>", parentSpanId);
        }
        return (SELF) this;
    }

    public SELF hasTraceIdEqualTo(String traceId) {
        isNotNull();
        if (!this.actual.getTraceId().equals(traceId)) {
            failWithMessage("Span should have trace id equal to <%s> but has <%s>", traceId,
                    this.actual.getTraceId());
        }
        return (SELF) this;
    }

    public SELF doesNotHaveTraceIdEqualTo(String traceId) {
        isNotNull();
        if (this.actual.getTraceId().equals(traceId)) {
            failWithMessage("Span should not have trace id equal to <%s>", traceId);
        }
        return (SELF) this;
    }

    // ── Link assertions ─────────────────────────────────────────────────

    public SELF hasLink(Link link) {
        isNotNull();
        if (!this.actual.getLinks().contains(link)) {
            failWithMessage("Span should have a link <%s> but has <%s>", link, this.actual.getLinks());
        }
        return (SELF) this;
    }

    public SELF doesNotHaveLink(Link link) {
        isNotNull();
        if (this.actual.getLinks().contains(link)) {
            failWithMessage("Span should not have a link but at least one was found <%s>", this.actual.getLinks());
        }
        return (SELF) this;
    }

    public SELF hasLink(Consumer<Link> consumer) {
        isNotNull();
        boolean atLeastOnePassed = false;
        for (Link entry : this.actual.getLinks()) {
            try {
                consumer.accept(entry);
                atLeastOnePassed = true;
                break;
            }
            catch (AssertionError error) {
                // try next link
            }
        }
        if (!atLeastOnePassed) {
            failWithMessage("Not a single link has passed the assertion");
        }
        return (SELF) this;
    }

    public SELF doesNotHaveLink(Consumer<Link> consumer) {
        isNotNull();
        Link passingEntry = null;
        for (Link entry : this.actual.getLinks()) {
            try {
                consumer.accept(entry);
                passingEntry = entry;
                break;
            }
            catch (AssertionError error) {
                // try next link
            }
        }
        if (passingEntry != null) {
            failWithMessage("At least one link has passed the assertion. First link passing assertion <%s>",
                    passingEntry);
        }
        return (SELF) this;
    }

    // ── Inner: SpanAssertReturningAssert ─────────────────────────────────

    /**
     * Extends {@link AbstractThrowableAssert} with a {@link #backToSpan()} method,
     * allowing fluent navigation from throwable assertions back to span assertions:
     *
     * <pre>{@code
     * SpanAssert.assertThat(span)
     *     .assertThatThrowable()
     *         .isInstanceOf(RuntimeException.class)
     *     .backToSpan()
     *     .hasNameEqualTo("encode");
     * }</pre>
     */
    public static class SpanAssertReturningAssert
            extends AbstractThrowableAssert<SpanAssertReturningAssert, Throwable> {

        private final SpanAssert spanAssert;

        public SpanAssertReturningAssert(Throwable throwable, SpanAssert spanAssert) {
            super(throwable, SpanAssertReturningAssert.class);
            this.spanAssert = spanAssert;
        }

        /**
         * Goes back to the previous {@link SpanAssert}.
         * @return previous span assert
         */
        public SpanAssert backToSpan() {
            return this.spanAssert;
        }

    }

}
```

### `simple-tracing-test/src/main/java/io/simpletracing/test/simple/SpansAssert.java` [NEW]

```java
package io.simpletracing.test.simple;

import java.util.Collection;
import java.util.function.Consumer;
import java.util.stream.Collectors;

import io.simpletracing.exporter.FinishedSpan;
import org.assertj.core.api.CollectionAssert;

/**
 * AssertJ-style fluent assertions for a collection of {@link FinishedSpan}s. Extends
 * {@link CollectionAssert} so clients get all standard collection assertions
 * ({@code hasSize()}, {@code isEmpty()}, {@code contains()}) for free, plus
 * span-specific methods layered on top.
 *
 * <p>Usage:
 * <pre>{@code
 * SpansAssert.assertThat(tracer.getSpans())
 *     .hasSize(3)
 *     .hasASpanWithName("encode")
 *     .haveSameTraceId();
 * }</pre>
 *
 * <p>The {@link SpansAssertReturningAssert} inner class enables navigation from a
 * specific span back to the collection context:
 * <pre>{@code
 * SpansAssert.assertThat(spans)
 *     .assertThatASpanWithNameEqualTo("encode")
 *         .hasTag("input.length", "256")
 *     .backToSpans()
 *     .hasASpanWithName("decode");
 * }</pre>
 *
 * <p>Maps to: {@code io.micrometer.tracing.test.simple.SpansAssert}
 */
public class SpansAssert extends CollectionAssert<FinishedSpan> {

    protected SpansAssert(Collection<? extends FinishedSpan> actual) {
        super(actual);
    }

    public static SpansAssert assertThat(Collection<? extends FinishedSpan> actual) {
        return new SpansAssert(actual);
    }

    public static SpansAssert then(Collection<? extends FinishedSpan> actual) {
        return new SpansAssert(actual);
    }

    // ── Trace ID assertions ─────────────────────────────────────────────

    public SpansAssert haveSameTraceId() {
        isNotEmpty();
        long distinctTraceIds = this.actual.stream()
                .map(FinishedSpan::getTraceId)
                .distinct()
                .count();
        if (distinctTraceIds != 1) {
            failWithMessage("Spans should have same trace ids but found %s trace ids. Found following spans \n%s",
                    distinctTraceIds, spansAsString());
        }
        return this;
    }

    // ── Name-based queries ──────────────────────────────────────────────

    public SpansAssert hasASpanWithName(String name) {
        isNotEmpty();
        extractSpanWithName(name);
        return this;
    }

    @SuppressWarnings("rawtypes")
    public SpansAssert hasASpanWithName(String name, Consumer<SpanAssert> spanConsumer) {
        isNotEmpty();
        FinishedSpan match = this.actual.stream()
                .filter(f -> name.equals(f.getName()))
                .filter(f -> {
                    try {
                        spanConsumer.accept(SpanAssert.assertThat(f));
                        return true;
                    }
                    catch (AssertionError e) {
                        return false;
                    }
                })
                .findFirst()
                .orElse(null);
        if (match == null) {
            failWithMessage(
                    "Not a single span with name <%s> was found or has passed the assertion. Found following spans %s",
                    name, spansAsString());
        }
        return this;
    }

    public SpansAssert hasASpanWithNameIgnoreCase(String name) {
        isNotEmpty();
        this.actual.stream()
                .filter(f -> name.equalsIgnoreCase(f.getName()))
                .findFirst()
                .orElseThrow(() -> {
                    failWithMessage(
                            "There should be at least one span with name (ignoring case) <%s> but found none. Found following spans \n%s",
                            name, spansAsString());
                    return new AssertionError();
                });
        return this;
    }

    @SuppressWarnings("rawtypes")
    public SpansAssert hasASpanWithNameIgnoreCase(String name, Consumer<SpanAssert> spanConsumer) {
        isNotEmpty();
        FinishedSpan match = this.actual.stream()
                .filter(f -> name.equalsIgnoreCase(f.getName()))
                .filter(f -> {
                    try {
                        spanConsumer.accept(SpanAssert.assertThat(f));
                        return true;
                    }
                    catch (AssertionError e) {
                        return false;
                    }
                })
                .findFirst()
                .orElse(null);
        if (match == null) {
            failWithMessage(
                    "Not a single span with name <%s> was found (ignoring case) or has passed the assertion. Found following spans %s",
                    name, spansAsString());
        }
        return this;
    }

    // ── Counting assertions ─────────────────────────────────────────────

    public SpansAssert hasNumberOfSpansEqualTo(int expectedNumberOfSpans) {
        isNotEmpty();
        if (this.actual.size() != expectedNumberOfSpans) {
            failWithMessage("There should be <%s> spans but there were <%s>. Found following spans \n%s",
                    expectedNumberOfSpans, this.actual.size(), spansAsString());
        }
        return this;
    }

    public SpansAssert hasNumberOfSpansWithNameEqualTo(String spanName, int expectedNumberOfSpans) {
        isNotEmpty();
        long count = this.actual.stream().filter(f -> spanName.equals(f.getName())).count();
        if (count != expectedNumberOfSpans) {
            failWithMessage("There should be <%s> spans with name <%s> but there were <%s>. Found following spans \n%s",
                    expectedNumberOfSpans, spanName, count, spansAsString());
        }
        return this;
    }

    public SpansAssert hasNumberOfSpansWithNameEqualToIgnoreCase(String spanName, int expectedNumberOfSpans) {
        isNotEmpty();
        long count = this.actual.stream().filter(f -> spanName.equalsIgnoreCase(f.getName())).count();
        if (count != expectedNumberOfSpans) {
            failWithMessage(
                    "There should be <%s> spans with name (ignoring case) <%s> but there were <%s>. Found following spans \n%s",
                    expectedNumberOfSpans, spanName, count, spansAsString());
        }
        return this;
    }

    // ── Remote service name assertions ──────────────────────────────────

    public SpansAssert hasASpanWithRemoteServiceName(String remoteServiceName) {
        isNotEmpty();
        this.actual.stream()
                .filter(f -> remoteServiceName.equals(f.getRemoteServiceName()))
                .findFirst()
                .orElseThrow(() -> {
                    failWithMessage(
                            "There should be at least one span with remote service name <%s> but found none. Found following spans \n%s",
                            remoteServiceName, spansAsString());
                    return new AssertionError();
                });
        return this;
    }

    // ── Tag assertions ──────────────────────────────────────────────────

    public SpansAssert hasASpanWithATag(String key, String value) {
        isNotEmpty();
        this.actual.stream()
                .filter(f -> value.equals(f.getTags().get(key)))
                .findFirst()
                .orElseThrow(() -> {
                    failWithMessage(
                            "There should be at least one span with tag key <%s> and value <%s> but found none. Found following spans \n%s",
                            key, value, spansAsString());
                    return new AssertionError();
                });
        return this;
    }

    public SpansAssert hasASpanWithATagKey(String key) {
        isNotEmpty();
        this.actual.stream()
                .filter(f -> f.getTags().containsKey(key))
                .findFirst()
                .orElseThrow(() -> {
                    failWithMessage(
                            "There should be at least one span with tag key <%s> but found none. Found following spans \n%s",
                            key, spansAsString());
                    return new AssertionError();
                });
        return this;
    }

    // ── Navigation to single-span assertions ────────────────────────────

    public SpansAssertReturningAssert assertThatASpanWithNameEqualTo(String name) {
        isNotEmpty();
        FinishedSpan span = extractSpanWithName(name);
        return new SpansAssertReturningAssert(this, span);
    }

    public SpansAssertReturningAssert thenASpanWithNameEqualTo(String name) {
        return assertThatASpanWithNameEqualTo(name);
    }

    public SpansAssertReturningAssert assertThatASpanWithNameEqualToIgnoreCase(String name) {
        isNotEmpty();
        FinishedSpan span = extractSpanWithNameIgnoreCase(name);
        return new SpansAssertReturningAssert(this, span);
    }

    public SpansAssertReturningAssert thenASpanWithNameEqualToIgnoreCase(String name) {
        return assertThatASpanWithNameEqualToIgnoreCase(name);
    }

    // ── For-all assertions ──────────────────────────────────────────────

    @SuppressWarnings("rawtypes")
    public SpansAssert forAllSpansWithNameEqualTo(String name, Consumer<SpanAssert> spanConsumer) {
        isNotEmpty();
        hasASpanWithName(name);
        this.actual.stream()
                .filter(f -> name.equals(f.getName()))
                .forEach(f -> spanConsumer.accept(SpanAssert.then(f)));
        return this;
    }

    @SuppressWarnings("rawtypes")
    public SpansAssert forAllSpansWithNameEqualToIgnoreCase(String name, Consumer<SpanAssert> spanConsumer) {
        isNotEmpty();
        hasASpanWithNameIgnoreCase(name);
        this.actual.stream()
                .filter(f -> name.equalsIgnoreCase(f.getName()))
                .forEach(f -> spanConsumer.accept(SpanAssert.then(f)));
        return this;
    }

    // ── Private helpers ─────────────────────────────────────────────────

    private FinishedSpan extractSpanWithName(String name) {
        return this.actual.stream().filter(f -> name.equals(f.getName())).findFirst().orElseThrow(() -> {
            failWithMessage(
                    "There should be at least one span with name <%s> but found none. Found following spans \n%s", name,
                    spansAsString());
            return new AssertionError();
        });
    }

    private FinishedSpan extractSpanWithNameIgnoreCase(String name) {
        return this.actual.stream().filter(f -> name.equalsIgnoreCase(f.getName())).findFirst().orElseThrow(() -> {
            failWithMessage(
                    "There should be at least one span with name (ignore case) <%s> but found none. Found following spans \n%s",
                    name, spansAsString());
            return new AssertionError();
        });
    }

    private String spansAsString() {
        return this.actual.stream().map(Object::toString).collect(Collectors.joining("\n"));
    }

    // ── Inner: SpansAssertReturningAssert ────────────────────────────────

    /**
     * Extends {@link SpanAssert} with a {@link #backToSpans()} method, enabling
     * navigation from a single span's assertions back to the collection context.
     */
    public static class SpansAssertReturningAssert extends SpanAssert<SpansAssertReturningAssert> {

        private final SpansAssert spansAssert;

        public SpansAssertReturningAssert(SpansAssert spansAssert, FinishedSpan span) {
            super(span);
            this.spansAssert = spansAssert;
        }

        public SpansAssert backToSpans() {
            return this.spansAssert;
        }

    }

}
```

### `simple-tracing-test/src/main/java/io/simpletracing/test/simple/TracerAssert.java` [NEW]

```java
package io.simpletracing.test.simple;

import java.util.Collection;

import org.assertj.core.api.AbstractAssert;
import org.assertj.core.api.AbstractCollectionAssert;
import org.assertj.core.api.Assertions;
import org.assertj.core.api.ObjectAssert;

/**
 * AssertJ-style fluent assertions for {@link SimpleTracer}. Acts as an entry point
 * that navigates to {@link SpanAssert} (for single spans) or standard AssertJ
 * collection assertions (for all reported spans).
 *
 * <p>Usage:
 * <pre>{@code
 * TracerAssert.assertThat(tracer)
 *     .onlySpan()
 *     .hasNameEqualTo("encode")
 *     .hasTagWithKey("input.length");
 * }</pre>
 *
 * <p>Maps to: {@code io.micrometer.tracing.test.simple.TracerAssert}
 */
@SuppressWarnings("rawtypes")
public class TracerAssert extends AbstractAssert<TracerAssert, SimpleTracer> {

    protected TracerAssert(SimpleTracer actual) {
        super(actual, TracerAssert.class);
    }

    public static TracerAssert assertThat(SimpleTracer actual) {
        return new TracerAssert(actual);
    }

    public static TracerAssert then(SimpleTracer actual) {
        return new TracerAssert(actual);
    }

    /**
     * Verifies that exactly one span was created and returns a {@link SpanAssert} for it.
     * Delegates to {@link SimpleTracer#onlySpan()}, which asserts the span was started
     * and ended.
     * @return span assertions for the only span
     */
    public SpanAssert onlySpan() {
        isNotNull();
        return SpanAssert.assertThat(this.actual.onlySpan());
    }

    /**
     * Returns a {@link SpanAssert} for the most recently created span.
     * @return span assertions for the last span
     */
    public SpanAssert lastSpan() {
        isNotNull();
        return SpanAssert.assertThat(this.actual.lastSpan());
    }

    /**
     * Returns standard AssertJ collection assertions for all spans created by this tracer.
     * @return collection assertions on all reported spans
     */
    public AbstractCollectionAssert<?, Collection<? extends SimpleSpan>, SimpleSpan, ObjectAssert<SimpleSpan>> reportedSpans() {
        return Assertions.assertThat(this.actual.getSpans());
    }

}
```

### `simple-tracing-test/src/main/java/io/simpletracing/test/simple/TracingAssertions.java` [NEW]

```java
package io.simpletracing.test.simple;

import java.util.Collection;

import io.simpletracing.exporter.FinishedSpan;

/**
 * Static entry point for all tracing assertions. Provides overloaded {@code assertThat()}
 * and {@code then()} factory methods that dispatch to the appropriate assertion class
 * based on the argument type.
 *
 * <p>This class exists so clients can use a single static import for all tracing assertions:
 * <pre>{@code
 * import static io.simpletracing.test.simple.TracingAssertions.*;
 *
 * // Assert on a single span
 * assertThat(span).hasNameEqualTo("encode");
 *
 * // Assert on a collection of spans
 * assertThat(spans).hasASpanWithName("encode");
 *
 * // Assert on the tracer
 * assertThat(tracer).onlySpan().hasNameEqualTo("encode");
 * }</pre>
 *
 * <p>Maps to: {@code io.micrometer.tracing.test.simple.TracingAssertions}
 */
public class TracingAssertions {

    @SuppressWarnings("rawtypes")
    public static SpanAssert assertThat(FinishedSpan actual) {
        return new SpanAssert(actual);
    }

    @SuppressWarnings("rawtypes")
    public static SpanAssert then(FinishedSpan actual) {
        return new SpanAssert(actual);
    }

    public static SpansAssert assertThat(Collection<FinishedSpan> actual) {
        return new SpansAssert(actual);
    }

    public static SpansAssert then(Collection<FinishedSpan> actual) {
        return new SpansAssert(actual);
    }

    public static TracerAssert assertThat(SimpleTracer actual) {
        return new TracerAssert(actual);
    }

    public static TracerAssert then(SimpleTracer actual) {
        return new TracerAssert(actual);
    }

}
```

### `simple-tracing-test/src/test/java/io/simpletracing/test/simple/AssertionTest.java` [NEW]

```java
package io.simpletracing.test.simple;

import java.util.Collection;

import io.simpletracing.Link;
import io.simpletracing.Span;
import io.simpletracing.Tracer;
import io.simpletracing.exporter.FinishedSpan;
import org.junit.jupiter.api.AfterEach;
import org.junit.jupiter.api.Nested;
import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThatThrownBy;

/**
 * Tests for the assertion classes: {@link SpanAssert}, {@link SpansAssert},
 * {@link TracerAssert}, and {@link TracingAssertions}.
 *
 * <p>These tests verify that each assertion method passes when it should and fails
 * (with a meaningful message) when it shouldn't. This is "testing the test utilities"
 * — ensuring the assertion framework itself is correct before clients rely on it.
 */
class AssertionTest {

    private final SimpleTracer tracer = new SimpleTracer();

    @AfterEach
    void cleanup() {
        SimpleTracer.resetCurrentSpan();
    }

    // ── SpanAssert ──────────────────────────────────────────────────────

    @Nested
    class SpanAssertTests {

        @Test
        void shouldPassWhenNameMatches() {
            SimpleSpan span = createFinishedSpan("encode");
            SpanAssert.assertThat(span).hasNameEqualTo("encode");
        }

        @Test
        void shouldFailWhenNameDoesNotMatch() {
            SimpleSpan span = createFinishedSpan("encode");
            assertThatThrownBy(() -> SpanAssert.assertThat(span).hasNameEqualTo("decode"))
                    .isInstanceOf(AssertionError.class);
        }

        @Test
        void shouldPassWhenDoesNotHaveName() {
            SimpleSpan span = createFinishedSpan("encode");
            SpanAssert.assertThat(span).doesNotHaveNameEqualTo("decode");
        }

        @Test
        void shouldPassWhenTagWithKeyExists() {
            SimpleSpan span = createFinishedSpan("encode");
            span.tag("input.length", "256");
            SpanAssert.assertThat(span).hasTagWithKey("input.length");
        }

        @Test
        void shouldFailWhenTagWithKeyMissing() {
            SimpleSpan span = createFinishedSpan("encode");
            assertThatThrownBy(() -> SpanAssert.assertThat(span).hasTagWithKey("missing"))
                    .isInstanceOf(AssertionError.class);
        }

        @Test
        void shouldPassWhenTagWithKeyAndValueMatch() {
            SimpleSpan span = createFinishedSpan("encode");
            span.tag("input.length", "256");
            SpanAssert.assertThat(span).hasTag("input.length", "256");
        }

        @Test
        void shouldFailWhenTagValueDoesNotMatch() {
            SimpleSpan span = createFinishedSpan("encode");
            span.tag("input.length", "256");
            assertThatThrownBy(() -> SpanAssert.assertThat(span).hasTag("input.length", "512"))
                    .isInstanceOf(AssertionError.class);
        }

        @Test
        void shouldPassWhenNoTags() {
            SimpleSpan span = createFinishedSpan("encode");
            SpanAssert.assertThat(span).hasNoTags();
        }

        @Test
        void shouldFailWhenExpectingNoTagsButHasTags() {
            SimpleSpan span = createFinishedSpan("encode");
            span.tag("key", "value");
            assertThatThrownBy(() -> SpanAssert.assertThat(span).hasNoTags())
                    .isInstanceOf(AssertionError.class);
        }

        @Test
        void shouldPassWhenDoesNotHaveTagWithKey() {
            SimpleSpan span = createFinishedSpan("encode");
            SpanAssert.assertThat(span).doesNotHaveTagWithKey("missing");
        }

        @Test
        void shouldPassWhenEventExists() {
            SimpleSpan span = createFinishedSpan("encode");
            span.event("encoding-started");
            SpanAssert.assertThat(span).hasEventWithNameEqualTo("encoding-started");
        }

        @Test
        void shouldFailWhenEventMissing() {
            SimpleSpan span = createFinishedSpan("encode");
            assertThatThrownBy(() -> SpanAssert.assertThat(span).hasEventWithNameEqualTo("missing"))
                    .isInstanceOf(AssertionError.class);
        }

        @Test
        void shouldPassWhenDoesNotHaveEvent() {
            SimpleSpan span = createFinishedSpan("encode");
            SpanAssert.assertThat(span).doesNotHaveEventWithNameEqualTo("missing");
        }

        @Test
        void shouldPassWhenStartedAndEnded() {
            SimpleSpan span = createFinishedSpan("encode");
            SpanAssert.assertThat(span).isStarted().isEnded();
        }

        @Test
        void shouldPassWhenNotStarted() {
            SimpleSpan span = new SimpleSpan();
            span.name("not-started");
            SpanAssert.assertThat(span).isNotStarted();
        }

        @Test
        void shouldPassWhenNotEnded() {
            SimpleSpan span = new SimpleSpan();
            span.name("started-only");
            span.start();
            SpanAssert.assertThat(span).isStarted().isNotEnded();
        }

        @Test
        void shouldPassWhenHasNoErrors() {
            SimpleSpan span = createFinishedSpan("encode");
            SpanAssert.assertThat(span).hasNoErrors();
        }

        @Test
        void shouldFailWhenHasNoErrorsButErrorPresent() {
            SimpleSpan span = createFinishedSpan("encode");
            span.error(new RuntimeException("boom"));
            assertThatThrownBy(() -> SpanAssert.assertThat(span).hasNoErrors())
                    .isInstanceOf(AssertionError.class);
        }

        @Test
        void shouldPassWhenKindMatches() {
            SimpleSpan span = createFinishedSpan("encode");
            span.setSpanKind(Span.Kind.CLIENT);
            SpanAssert.assertThat(span).hasKindEqualTo(Span.Kind.CLIENT);
        }

        @Test
        void shouldPassWhenDoesNotHaveKind() {
            SimpleSpan span = createFinishedSpan("encode");
            span.setSpanKind(Span.Kind.CLIENT);
            SpanAssert.assertThat(span).doesNotHaveKindEqualTo(Span.Kind.SERVER);
        }

        @Test
        void shouldPassWhenRemoteServiceNameMatches() {
            SimpleSpan span = createFinishedSpan("encode");
            span.remoteServiceName("auth-service");
            SpanAssert.assertThat(span).hasRemoteServiceNameEqualTo("auth-service");
        }

        @Test
        void shouldPassWhenDoesNotHaveRemoteServiceName() {
            SimpleSpan span = createFinishedSpan("encode");
            span.remoteServiceName("auth-service");
            SpanAssert.assertThat(span)
                    .doesNotHaveRemoteServiceNameEqualTo("payment-service");
        }

        @Test
        void shouldPassWhenSpanIdMatches() {
            SimpleSpan span = createFinishedSpan("encode");
            String spanId = span.getSpanId();
            SpanAssert.assertThat(span).hasSpanIdEqualTo(spanId);
        }

        @Test
        void shouldPassWhenTraceIdMatches() {
            SimpleSpan span = createFinishedSpan("encode");
            String traceId = span.getTraceId();
            SpanAssert.assertThat(span).hasTraceIdEqualTo(traceId);
        }

        @Test
        void shouldPassWhenParentIdMatches() {
            Span parent = tracer.nextSpan().name("parent").start();
            try (Tracer.SpanInScope ws = tracer.withSpan(parent)) {
                Span child = tracer.nextSpan().name("child").start();
                child.end();
                SpanAssert.assertThat((FinishedSpan) child)
                        .hasParentIdEqualTo(parent.context().spanId());
            } finally {
                parent.end();
            }
        }

        @Test
        void shouldPassWhenParentIdIsEmpty() {
            // Root spans in SimpleTraceContext have parentId="" (empty string, not null)
            SimpleSpan span = createFinishedSpan("root");
            SpanAssert.assertThat(span).hasParentIdEqualTo("");
        }

        @Test
        void shouldSupportFluentChaining() {
            SimpleSpan span = createFinishedSpan("encode");
            span.tag("input.length", "256");
            span.event("encoding-started");

            SpanAssert.assertThat(span)
                    .hasNameEqualTo("encode")
                    .hasTagWithKey("input.length")
                    .hasTag("input.length", "256")
                    .hasEventWithNameEqualTo("encoding-started")
                    .hasNoErrors()
                    .isStarted()
                    .isEnded();
        }

        @Test
        void shouldSupportThrowableAssertions() {
            SimpleSpan span = createFinishedSpan("encode");
            span.error(new IllegalStateException("encoding failed"));

            SpanAssert.assertThat(span)
                    .assertThatThrowable()
                        .isInstanceOf(IllegalStateException.class)
                        .hasMessage("encoding failed")
                    .backToSpan()
                    .hasNameEqualTo("encode");
        }

        @Test
        void shouldPassWhenLinkExists() {
            SimpleSpan linked = createFinishedSpan("linked");
            Link link = new Link(linked.context());

            SimpleSpan span = new SimpleSpan();
            span.name("batch");
            span.getLinks().add(link);
            span.start();
            span.end();

            SpanAssert.assertThat(span).hasLink(link);
        }

        @Test
        void shouldPassWhenDoesNotHaveLink() {
            SimpleSpan linked = createFinishedSpan("linked");
            Link link = new Link(linked.context());

            SimpleSpan span = createFinishedSpan("batch");
            SpanAssert.assertThat(span).doesNotHaveLink(link);
        }

        @Test
        void shouldSupportThenAlias() {
            SimpleSpan span = createFinishedSpan("encode");
            SpanAssert.then(span).hasNameEqualTo("encode");
        }
    }

    // ── SpansAssert ─────────────────────────────────────────────────────

    @Nested
    class SpansAssertTests {

        @Test
        void shouldPassWhenHasSize() {
            createFinishedSpanViaTracer("encode");
            createFinishedSpanViaTracer("decode");
            createFinishedSpanViaTracer("send");

            SpansAssert.assertThat(castSpans(tracer.getSpans()))
                    .hasNumberOfSpansEqualTo(3);
        }

        @Test
        void shouldFailWhenSizeDoesNotMatch() {
            createFinishedSpanViaTracer("encode");

            assertThatThrownBy(() -> SpansAssert.assertThat(castSpans(tracer.getSpans()))
                    .hasNumberOfSpansEqualTo(3))
                    .isInstanceOf(AssertionError.class);
        }

        @Test
        void shouldPassWhenHasSpanWithName() {
            createFinishedSpanViaTracer("encode");
            createFinishedSpanViaTracer("decode");

            SpansAssert.assertThat(castSpans(tracer.getSpans()))
                    .hasASpanWithName("encode")
                    .hasASpanWithName("decode");
        }

        @Test
        void shouldFailWhenNoSpanWithName() {
            createFinishedSpanViaTracer("encode");

            assertThatThrownBy(() -> SpansAssert.assertThat(castSpans(tracer.getSpans()))
                    .hasASpanWithName("missing"))
                    .isInstanceOf(AssertionError.class);
        }

        @Test
        void shouldPassWhenAllSpansHaveSameTraceId() {
            Span parent = tracer.nextSpan().name("parent").start();
            try (Tracer.SpanInScope ws = tracer.withSpan(parent)) {
                Span child = tracer.nextSpan().name("child").start();
                child.end();
            } finally {
                parent.end();
            }

            SpansAssert.assertThat(castSpans(tracer.getSpans()))
                    .haveSameTraceId();
        }

        @Test
        void shouldPassWhenHasSpanWithNameAndAssertion() {
            createFinishedSpanViaTracer("encode");

            SpansAssert.assertThat(castSpans(tracer.getSpans()))
                    .hasASpanWithName("encode", spanAssert -> spanAssert.isStarted().isEnded());
        }

        @Test
        void shouldPassWhenHasSpanWithNameIgnoreCase() {
            createFinishedSpanViaTracer("encode");

            SpansAssert.assertThat(castSpans(tracer.getSpans()))
                    .hasASpanWithNameIgnoreCase("ENCODE");
        }

        @Test
        void shouldPassForNumberOfSpansWithName() {
            createFinishedSpanViaTracer("encode");
            createFinishedSpanViaTracer("encode");
            createFinishedSpanViaTracer("decode");

            SpansAssert.assertThat(castSpans(tracer.getSpans()))
                    .hasNumberOfSpansWithNameEqualTo("encode", 2);
        }

        @Test
        void shouldPassWhenHasSpanWithRemoteServiceName() {
            Span span = tracer.nextSpan().name("call").start();
            span.remoteServiceName("auth-service");
            span.end();

            SpansAssert.assertThat(castSpans(tracer.getSpans()))
                    .hasASpanWithRemoteServiceName("auth-service");
        }

        @Test
        void shouldPassWhenHasSpanWithTag() {
            Span span = tracer.nextSpan().name("encode").start();
            span.tag("input.length", "256");
            span.end();

            SpansAssert.assertThat(castSpans(tracer.getSpans()))
                    .hasASpanWithATag("input.length", "256");
        }

        @Test
        void shouldPassWhenHasSpanWithTagKey() {
            Span span = tracer.nextSpan().name("encode").start();
            span.tag("input.length", "256");
            span.end();

            SpansAssert.assertThat(castSpans(tracer.getSpans()))
                    .hasASpanWithATagKey("input.length");
        }

        @Test
        void shouldSupportAssertThatASpanWithNameNavigation() {
            createFinishedSpanViaTracer("encode");
            createFinishedSpanViaTracer("decode");

            SpansAssert.assertThat(castSpans(tracer.getSpans()))
                    .assertThatASpanWithNameEqualTo("encode")
                        .isStarted()
                        .isEnded()
                    .backToSpans()
                    .hasASpanWithName("decode");
        }

        @Test
        void shouldSupportForAllSpansWithName() {
            Span s1 = tracer.nextSpan().name("encode").start();
            s1.tag("type", "json");
            s1.end();
            Span s2 = tracer.nextSpan().name("encode").start();
            s2.tag("type", "xml");
            s2.end();

            SpansAssert.assertThat(castSpans(tracer.getSpans()))
                    .forAllSpansWithNameEqualTo("encode",
                            spanAssert -> spanAssert.hasTagWithKey("type"));
        }

        @Test
        void shouldSupportThenAlias() {
            createFinishedSpanViaTracer("encode");

            SpansAssert.then(castSpans(tracer.getSpans()))
                    .hasASpanWithName("encode");
        }
    }

    // ── TracerAssert ────────────────────────────────────────────────────

    @Nested
    class TracerAssertTests {

        @Test
        void shouldNavigateToOnlySpan() {
            createFinishedSpanViaTracer("encode");

            TracerAssert.assertThat(tracer)
                    .onlySpan()
                    .hasNameEqualTo("encode");
        }

        @Test
        void shouldNavigateToLastSpan() {
            createFinishedSpanViaTracer("encode");
            createFinishedSpanViaTracer("decode");

            TracerAssert.assertThat(tracer)
                    .lastSpan()
                    .hasNameEqualTo("decode");
        }

        @Test
        void shouldNavigateToReportedSpans() {
            createFinishedSpanViaTracer("encode");
            createFinishedSpanViaTracer("decode");

            TracerAssert.assertThat(tracer)
                    .reportedSpans()
                    .hasSize(2);
        }

        @Test
        void shouldSupportThenAlias() {
            createFinishedSpanViaTracer("encode");

            TracerAssert.then(tracer)
                    .onlySpan()
                    .hasNameEqualTo("encode");
        }
    }

    // ── TracingAssertions (static entry point) ──────────────────────────

    @Nested
    class TracingAssertionsTests {

        @Test
        void shouldCreateSpanAssert() {
            SimpleSpan span = createFinishedSpan("encode");
            TracingAssertions.assertThat((FinishedSpan) span).hasNameEqualTo("encode");
        }

        @Test
        void shouldCreateSpansAssert() {
            createFinishedSpanViaTracer("encode");
            createFinishedSpanViaTracer("decode");

            TracingAssertions.assertThat(castSpans(tracer.getSpans()))
                    .hasASpanWithName("encode");
        }

        @Test
        void shouldCreateTracerAssert() {
            createFinishedSpanViaTracer("encode");

            TracingAssertions.assertThat(tracer)
                    .onlySpan()
                    .hasNameEqualTo("encode");
        }

        @Test
        void shouldSupportThenAliases() {
            SimpleSpan span = createFinishedSpan("encode");
            TracingAssertions.then((FinishedSpan) span).hasNameEqualTo("encode");

            createFinishedSpanViaTracer("decode");
            TracingAssertions.then(tracer).lastSpan().hasNameEqualTo("decode");
        }
    }

    // ── Helpers ─────────────────────────────────────────────────────────

    private SimpleSpan createFinishedSpan(String name) {
        SimpleSpan span = new SimpleSpan();
        span.name(name);
        span.start();
        span.end();
        return span;
    }

    private void createFinishedSpanViaTracer(String name) {
        Span span = tracer.nextSpan().name(name).start();
        span.end();
    }

    @SuppressWarnings("unchecked")
    private static Collection<FinishedSpan> castSpans(Collection<? extends SimpleSpan> spans) {
        return (Collection<FinishedSpan>) (Collection<?>) spans;
    }

}
```
