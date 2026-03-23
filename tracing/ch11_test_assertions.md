# Chapter 11: Test Assertions

> Line references based on commit `dceab1b9` of the Micrometer Tracing repository.

## Build Challenge

| Current State | Limitation | Objective |
|---------------|------------|-----------|
| `SimpleTracer` collects finished spans and provides `onlySpan()` / `lastSpan()` for inspection | Verifying span names, tags, events, kinds, and parent-child relationships requires verbose manual assertions with raw `getTags().get("key")` calls | Build fluent, AssertJ-style assertion helpers (`SpanAssert`, `SpansAssert`, `TracerAssert`) that make tracing tests readable and expressive |

---

## 11.1 The Integration Point

Test assertion classes don't plug into an existing runtime pipeline — they plug into the **test layer** by reading from `FinishedSpan` and `SimpleTracer`. The integration point is where test code meets the tracing data model:

```
Test Code
    │
    ├──> TracerAssert.assertThat(tracer)
    │        ├── onlySpan()  ──> SpanAssert  (wraps FinishedSpan)
    │        ├── lastSpan()  ──> SpanAssert
    │        └── reportedSpans() ──> SpansAssert (wraps List<FinishedSpan>)
    │
    ├──> SpansAssert.assertThat(spans)
    │        └── assertThatASpanWithNameEqualTo("name")
    │                └──> SpansAssertReturningAssert (SpanAssert + backToSpans())
    │
    └──> SpanAssert.assertThat(finishedSpan)
             ├── hasNameEqualTo(), hasTag(), hasKindEqualTo(), ...
             └── reads from FinishedSpan interface
```

The key design question: **how do you support drilling into a single span from a collection, asserting on it, and navigating back?** The answer is self-referential generics and an inner subclass.

Two key decisions here:

1. **`SpanAssert<SELF extends SpanAssert<SELF>>`** — the self-referential generic allows `SpansAssertReturningAssert` (which extends `SpanAssert`) to return its own type from every fluent method, keeping `backToSpans()` accessible at any point in the chain.
2. **Extending `AbstractAssert`** — leveraging AssertJ's base class gives us `isNotNull()`, `failWithMessage()`, and the `actual` field for free, so each assertion method is just a null-check + condition + error message.

To make this work, we need to build:
- `SpanAssert` — fluent assertions on a single `FinishedSpan`
- `SpansAssert` — collection-level assertions with drill-down navigation
- `TracerAssert` — entry point from `SimpleTracer` to the assertion hierarchy

**Modifying:** `build.gradle`
**Change:** Move AssertJ from `testImplementation` to `implementation` so assertion classes can live in production code (matching the real framework's `micrometer-tracing-test` module)

```groovy
dependencies {
    implementation 'dev.linhvu:simple-micrometer:1.0-SNAPSHOT'
    implementation 'org.assertj:assertj-core:3.27.3'

    testImplementation platform('org.junit:junit-bom:5.11.4')
    testImplementation 'org.junit.jupiter:junit-jupiter'
    testRuntimeOnly 'org.junit.platform:junit-platform-launcher'
}
```

## 11.2 SpanAssert — Single-Span Assertions

**New file:** `src/main/java/dev/linhvu/tracing/simple/SpanAssert.java`

```java
package dev.linhvu.tracing.simple;

import dev.linhvu.tracing.Span;
import dev.linhvu.tracing.exporter.FinishedSpan;
import org.assertj.core.api.AbstractAssert;

import java.util.List;
import java.util.Map;
import java.util.Objects;

public class SpanAssert<SELF extends SpanAssert<SELF>> extends AbstractAssert<SELF, FinishedSpan> {

    protected SpanAssert(FinishedSpan actual, Class<?> selfType) {
        super(actual, selfType);
    }

    public static SpanAssert<?> assertThat(FinishedSpan actual) {
        return new SpanAssert<>(actual, SpanAssert.class);
    }

    public static SpanAssert<?> then(FinishedSpan actual) {
        return assertThat(actual);
    }

    // --- Name ---

    @SuppressWarnings("unchecked")
    public SELF hasNameEqualTo(String name) {
        isNotNull();
        if (!Objects.equals(actual.getName(), name)) {
            failWithMessage("Expected span name <%s> but was <%s>", name, actual.getName());
        }
        return (SELF) this;
    }

    // --- Tags ---

    @SuppressWarnings("unchecked")
    public SELF hasTag(String key, String value) {
        isNotNull();
        hasTagWithKey(key);
        String tagValue = actual.getTags().get(key);
        if (!Objects.equals(tagValue, value)) {
            failWithMessage("Expected tag <%s>=<%s> but was <%s>=<%s>",
                    key, value, key, tagValue);
        }
        return (SELF) this;
    }

    // --- Events ---

    @SuppressWarnings("unchecked")
    public SELF hasEventWithNameEqualTo(String eventName) {
        isNotNull();
        List<String> names = actual.getEvents().stream()
                .map(Map.Entry::getValue).toList();
        if (!names.contains(eventName)) {
            failWithMessage("Expected event <%s> but events were <%s>", eventName, names);
        }
        return (SELF) this;
    }

    // ... (hasKindEqualTo, hasRemoteServiceNameEqualTo, hasError, hasNoError,
    //      isStarted, isEnded, hasTraceIdEqualTo, hasSpanIdEqualTo, hasParentIdEqualTo)
    // Each follows the same pattern: isNotNull() → check → failWithMessage → return (SELF) this
}
```

Every assertion method follows a three-step pattern:
1. `isNotNull()` — inherited from `AbstractAssert`, guards against null spans
2. Check the condition against `actual` (the `FinishedSpan`)
3. On failure: `failWithMessage()` with a descriptive error; on success: `return (SELF) this` for chaining

## 11.3 SpansAssert — Collection-Level Assertions

**New file:** `src/main/java/dev/linhvu/tracing/simple/SpansAssert.java`

```java
package dev.linhvu.tracing.simple;

import dev.linhvu.tracing.exporter.FinishedSpan;
import org.assertj.core.api.AbstractAssert;

import java.util.Collection;
import java.util.List;
import java.util.stream.Collectors;

public class SpansAssert extends AbstractAssert<SpansAssert, List<? extends FinishedSpan>> {

    protected SpansAssert(List<? extends FinishedSpan> actual) {
        super(actual, SpansAssert.class);
    }

    public static SpansAssert assertThat(List<? extends FinishedSpan> actual) {
        return new SpansAssert(actual);
    }

    public SpansAssert haveSameTraceId() {
        isNotNull();
        long distinctTraceIds = actual.stream()
                .map(FinishedSpan::getTraceId).distinct().count();
        if (distinctTraceIds != 1) {
            failWithMessage("Expected all spans to have same traceId but found <%d> distinct",
                    distinctTraceIds);
        }
        return this;
    }

    // Navigation: drill into a single span, then come back
    public SpansAssertReturningAssert assertThatASpanWithNameEqualTo(String name) {
        isNotNull();
        FinishedSpan span = extractSpanWithName(name);
        return new SpansAssertReturningAssert(span, this);
    }

    // Inner class: SpanAssert subclass that remembers the parent SpansAssert
    public static class SpansAssertReturningAssert
            extends SpanAssert<SpansAssertReturningAssert> {

        private final SpansAssert spansAssert;

        SpansAssertReturningAssert(FinishedSpan span, SpansAssert spansAssert) {
            super(span, SpansAssertReturningAssert.class);
            this.spansAssert = spansAssert;
        }

        public SpansAssert backToSpans() {
            return spansAssert;
        }
    }
}
```

The `SpansAssertReturningAssert` inner class is the key enabler for the two-level navigation pattern:

```java
SpansAssert.assertThat(spans)
    .hasNumberOfSpansEqualTo(3)               // → SpansAssert
    .assertThatASpanWithNameEqualTo("http")   // → SpansAssertReturningAssert (a SpanAssert)
        .hasTag("method", "GET")              // → SpansAssertReturningAssert (SELF generic!)
        .isEnded()                            // → SpansAssertReturningAssert
    .backToSpans()                            // → SpansAssert
    .hasASpanWithName("db.query");            // → SpansAssert
```

Without the `SELF` generic on `SpanAssert`, the `hasTag()` call would return `SpanAssert<?>`, losing the `backToSpans()` method.

## 11.4 TracerAssert — Entry Point

**New file:** `src/main/java/dev/linhvu/tracing/simple/TracerAssert.java`

```java
package dev.linhvu.tracing.simple;

import dev.linhvu.tracing.exporter.FinishedSpan;
import org.assertj.core.api.AbstractAssert;
import java.util.List;

public class TracerAssert extends AbstractAssert<TracerAssert, SimpleTracer> {

    protected TracerAssert(SimpleTracer actual) {
        super(actual, TracerAssert.class);
    }

    public static TracerAssert assertThat(SimpleTracer actual) {
        return new TracerAssert(actual);
    }

    public SpanAssert<?> onlySpan() {
        isNotNull();
        return SpanAssert.assertThat(actual.onlySpan());
    }

    public SpanAssert<?> lastSpan() {
        isNotNull();
        return SpanAssert.assertThat(actual.lastSpan());
    }

    public SpansAssert reportedSpans() {
        isNotNull();
        List<FinishedSpan> spans = List.copyOf(actual.getSpans());
        return SpansAssert.assertThat(spans);
    }
}
```

`TracerAssert` is the simplest class — it provides three navigation methods that delegate to the tracer's inspection API and wrap the result in the appropriate assertion type.

## 11.5 Try It Yourself

<details>
<summary>Challenge: Add a doesNotHaveTag(key, value) method to SpanAssert</summary>

Think about: How should this differ from `doesNotHaveTagWithKey(key)`? The tag key might exist with a different value.

```java
@SuppressWarnings("unchecked")
public SELF doesNotHaveTag(String key, String value) {
    isNotNull();
    String tagValue = actual.getTags().get(key);
    if (tagValue != null && tagValue.equals(value)) {
        failWithMessage("Expected tag <%s> NOT to have value <%s>", key, value);
    }
    return (SELF) this;
}
```

</details>

<details>
<summary>Challenge: Add forAllSpansWithNameEqualTo(name, Consumer&lt;SpanAssert&gt;) to SpansAssert</summary>

This should assert that ALL spans with the given name pass the consumer's assertion (not just one).

```java
public SpansAssert forAllSpansWithNameEqualTo(String name,
        java.util.function.Consumer<SpanAssert<?>> spanConsumer) {
    isNotNull();
    List<? extends FinishedSpan> matching = actual.stream()
            .filter(s -> name.equals(s.getName()))
            .toList();
    if (matching.isEmpty()) {
        failWithMessage("Expected at least one span with name <%s>", name);
    }
    for (FinishedSpan span : matching) {
        spanConsumer.accept(SpanAssert.assertThat(span));
    }
    return this;
}
```

</details>

## 11.6 Tests

### Unit Tests

**New file:** `src/test/java/dev/linhvu/tracing/simple/SpanAssertTest.java`

```java
class SpanAssertTest {

    @Test
    void shouldPassNameAssertion_WhenNameMatches() {
        SimpleSpan span = spanWithName("http.request");
        SpanAssert.assertThat(span).hasNameEqualTo("http.request");
    }

    @Test
    void shouldFailNameAssertion_WhenNameDoesNotMatch() {
        SimpleSpan span = spanWithName("http.request");
        assertThatThrownBy(() -> SpanAssert.assertThat(span).hasNameEqualTo("wrong"))
                .isInstanceOf(AssertionError.class)
                .hasMessageContaining("wrong");
    }

    @Test
    void shouldSupportFluentChaining() {
        SimpleSpan span = spanWithName("http.request");
        span.tag("http.method", "GET");
        span.event("retry");
        span.setSpanKind(Span.Kind.CLIENT);
        span.remoteServiceName("backend");
        span.start();
        span.end();

        SpanAssert.assertThat(span)
                .hasNameEqualTo("http.request")
                .hasTag("http.method", "GET")
                .hasEventWithNameEqualTo("retry")
                .hasKindEqualTo(Span.Kind.CLIENT)
                .hasRemoteServiceNameEqualTo("backend")
                .hasNoError()
                .isStarted()
                .isEnded();
    }
    // ... (25 total test methods covering all assertion types)
}
```

**New file:** `src/test/java/dev/linhvu/tracing/simple/SpansAssertTest.java`

```java
class SpansAssertTest {

    @Test
    void shouldNavigateToSpanAndBack() {
        SimpleSpan s1 = span("http.request");
        s1.tag("http.method", "GET");
        s1.setSpanKind(Span.Kind.CLIENT);
        s1.start();
        s1.end();

        SimpleSpan s2 = span("db.query");
        s2.start();
        s2.end();

        SpansAssert.assertThat(List.<FinishedSpan>of(s1, s2))
                .hasNumberOfSpansEqualTo(2)
                .assertThatASpanWithNameEqualTo("http.request")
                    .hasTag("http.method", "GET")
                    .hasKindEqualTo(Span.Kind.CLIENT)
                    .isEnded()
                .backToSpans()
                .hasASpanWithName("db.query");
    }
    // ... (12 total test methods)
}
```

**New file:** `src/test/java/dev/linhvu/tracing/simple/TracerAssertTest.java`

```java
class TracerAssertTest {

    @Test
    void shouldVerifyParentChildViaReportedSpans() {
        SimpleTracer tracer = new SimpleTracer();
        SimpleSpan parent = tracer.nextSpan().name("parent").start();
        SimpleSpan child = tracer.nextSpan(parent).name("child").start();
        child.end();
        parent.end();

        TracerAssert.assertThat(tracer)
                .reportedSpans()
                .hasNumberOfSpansEqualTo(2)
                .haveSameTraceId()
                .assertThatASpanWithNameEqualTo("child")
                    .hasParentIdEqualTo(parent.context().spanId())
                .backToSpans()
                .assertThatASpanWithNameEqualTo("parent")
                    .hasParentIdEqualTo("");
    }
    // ... (6 total test methods)
}
```

### Integration Tests

**New file:** `src/test/java/dev/linhvu/tracing/integration/TestAssertionsIntegrationTest.java`

```java
class TestAssertionsIntegrationTest {

    @Test
    void shouldAssertFullTracingWorkflow() {
        SimpleTracer tracer = new SimpleTracer();

        SimpleSpan httpSpan = (SimpleSpan) tracer.spanBuilder()
                .name("http.request")
                .kind(Span.Kind.SERVER)
                .remoteServiceName("gateway")
                .tag("http.method", "GET")
                .tag("http.url", "/api/users")
                .start();

        try (var scope = tracer.withSpan(httpSpan)) {
            SimpleSpan dbSpan = tracer.nextSpan().name("db.query").start();
            dbSpan.tag("db.type", "postgresql");
            dbSpan.event("connection.acquired");
            dbSpan.end();
        }
        httpSpan.event("response.sent");
        httpSpan.end();

        TracerAssert.assertThat(tracer)
                .reportedSpans()
                .hasNumberOfSpansEqualTo(2)
                .haveSameTraceId()
                .assertThatASpanWithNameEqualTo("http.request")
                    .hasTag("http.method", "GET")
                    .hasKindEqualTo(Span.Kind.SERVER)
                    .hasRemoteServiceNameEqualTo("gateway")
                    .hasEventWithNameEqualTo("response.sent")
                    .hasNoError()
                    .isStarted()
                    .isEnded()
                .backToSpans()
                .assertThatASpanWithNameEqualTo("db.query")
                    .hasTag("db.type", "postgresql")
                    .hasEventWithNameEqualTo("connection.acquired")
                    .hasParentIdEqualTo(httpSpan.context().spanId())
                .backToSpans();
    }

    @Test
    void shouldAssertSpansFromExportPipeline() {
        List<FinishedSpan> reported = new ArrayList<>();
        SpanFilter prefixer = span -> span.setName("traced." + span.getName());
        SpanReporter collector = reported::add;

        SimpleTracer tracer = new SimpleTracer(List.of(), List.of(prefixer), List.of(collector));
        tracer.nextSpan().name("request").start().end();

        SpansAssert.assertThat(reported)
                .hasNumberOfSpansEqualTo(1)
                .assertThatASpanWithNameEqualTo("traced.request")
                    .isStarted()
                    .isEnded()
                .backToSpans();
    }
    // ... (4 total test methods)
}
```

**Run:** `./gradlew test` — expected: all 384+ tests pass (including prior features' tests)

---

## 11.7 Why This Works

> ★ **Insight** -------------------------------------------
> - **Self-referential generics (`SELF extends SpanAssert<SELF>`)** are the same pattern AssertJ itself uses in `AbstractAssert`. Without this, every fluent method on a subclass would return the parent type, losing subclass-specific methods like `backToSpans()`. The `@SuppressWarnings("unchecked")` cast `(SELF) this` is safe because `SELF` is always the concrete class at runtime.
> - **When to skip this pattern:** If your assertion class has no subclasses, a concrete return type is simpler and easier to read. Only introduce the generic when you need navigable assertion hierarchies.
> -----------------------------------------------------------

> ★ **Insight** -------------------------------------------
> - **Assertions read from `FinishedSpan`, not `SimpleSpan`** — the assertion API operates on the read-time interface, meaning these assertions work with any `FinishedSpan` implementation. If you later implement a Brave or OpenTelemetry bridge that produces its own `FinishedSpan`, the same `SpanAssert` works unchanged.
> - **Real-world impact:** In the actual Micrometer Tracing project, `SpanAssert` is used in integration tests that run against both Brave and OpenTelemetry tracers — the same assertion code, different backends.
> -----------------------------------------------------------

> ★ **Insight** -------------------------------------------
> - **`failWithMessage()` over `assertThat()`** — using `failWithMessage` from `AbstractAssert` gives consistent, descriptive error messages that show both expected and actual values, plus the assertion's context. Using nested `Assertions.assertThat()` inside a custom assertion is an anti-pattern because the error messages lose the outer context.
> -----------------------------------------------------------

## 11.8 What We Enhanced

| Aspect | Before (ch03–ch04) | Current (ch11) | Real Framework |
|--------|---------------------|----------------|----------------|
| Span verification | Manual: `assertThat(span.getName()).isEqualTo(...)` on each field separately | Fluent: `SpanAssert.assertThat(span).hasNameEqualTo(...).hasTag(...).isEnded()` — single chain | Same pattern with additional `KeyName` overloads and IP/port/link assertions (`SpanAssert.java:44`) |
| Collection assertions | Manual: loop through `tracer.getSpans()` and check properties one by one | `SpansAssert` with `haveSameTraceId()`, `hasASpanWithName()`, drill-down via `assertThatASpanWithNameEqualTo()` | Extends `CollectionAssert` for inherited collection methods; includes case-insensitive variants (`SpansAssert.java:37`) |
| Tracer entry point | Direct: `tracer.onlySpan()` then manual assertions | `TracerAssert.assertThat(tracer).onlySpan().hasNameEqualTo(...)` — one fluent chain from tracer to span details | Same pattern; `reportedSpans()` returns standard AssertJ `AbstractCollectionAssert` rather than `SpansAssert` (`TracerAssert.java:115`) |
| Build dependency | AssertJ only in test scope | AssertJ as `implementation` dependency so assertion classes live in main source (test-utilities module) | Separate `micrometer-tracing-test` module with AssertJ as compile dependency |

## 11.9 Connection to Real Framework

| Simplified Code | Real Framework Code | File:Line | Key Difference |
|----------------|--------------------|-----------| ---------------|
| `SpanAssert<SELF>` | `SpanAssert<SELF>` | `SpanAssert.java:44` | Real adds `KeyName` overloads, IP/port/link assertions with `Consumer`, `SpanAssertReturningAssert` for throwable navigation |
| `hasNameEqualTo()` | `hasNameEqualTo()` | `SpanAssert.java:523` | Identical pattern; real version has Javadoc with pass/fail examples |
| `SpansAssert` | `SpansAssert` | `SpansAssert.java:37` | Real extends `CollectionAssert<FinishedSpan>` (inheriting `isEmpty`, `hasSize`, etc.); includes case-insensitive variants and `forAllSpansWithNameEqualTo()` |
| `SpansAssertReturningAssert` | `SpansAssertReturningAssert` | `SpansAssert.java:578` | Identical design — SpanAssert subclass with `backToSpans()` |
| `haveSameTraceId()` | `haveSameTraceId()` | `SpansAssert.java:78` | Identical implementation using `distinct().count()` |
| `TracerAssert` | `TracerAssert` | `TracerAssert.java:36` | Real `reportedSpans()` returns standard AssertJ `AbstractCollectionAssert` rather than custom `SpansAssert` |
| `onlySpan()` / `lastSpan()` | `onlySpan()` / `lastSpan()` | `TracerAssert.java:78,97` | Identical delegation pattern to `SimpleTracer` |

## 11.10 Complete Code

### Production Code

#### File: `build.gradle` [MODIFIED]

```groovy
plugins {
    id 'java'
}

group = 'dev.linhvu'
version = '1.0-SNAPSHOT'

java {
    sourceCompatibility = JavaVersion.VERSION_17
    targetCompatibility = JavaVersion.VERSION_17
}

repositories {
    mavenCentral()
}

dependencies {
    implementation 'dev.linhvu:simple-micrometer:1.0-SNAPSHOT'
    implementation 'org.assertj:assertj-core:3.27.3'

    testImplementation platform('org.junit:junit-bom:5.11.4')
    testImplementation 'org.junit.jupiter:junit-jupiter'
    testRuntimeOnly 'org.junit.platform:junit-platform-launcher'
}

test {
    useJUnitPlatform()
}
```

#### File: `src/main/java/dev/linhvu/tracing/simple/SpanAssert.java` [NEW]

```java
package dev.linhvu.tracing.simple;

import dev.linhvu.tracing.Span;
import dev.linhvu.tracing.exporter.FinishedSpan;
import org.assertj.core.api.AbstractAssert;

import java.util.List;
import java.util.Map;
import java.util.Objects;

/**
 * AssertJ-style fluent assertions on a single {@link FinishedSpan}.
 *
 * <p>Usage:
 * <pre>{@code
 * SpanAssert.assertThat(finishedSpan)
 *     .hasNameEqualTo("http.request")
 *     .hasTag("http.method", "GET")
 *     .hasKindEqualTo(Span.Kind.CLIENT)
 *     .isStarted()
 *     .isEnded()
 *     .hasNoError();
 * }</pre>
 *
 * <p>The self-referential generic {@code SELF} enables subclasses (like
 * {@link SpansAssert.SpansAssertReturningAssert}) to return their own type
 * from every fluent method, preserving navigation methods like {@code backToSpans()}.
 */
public class SpanAssert<SELF extends SpanAssert<SELF>> extends AbstractAssert<SELF, FinishedSpan> {

    protected SpanAssert(FinishedSpan actual, Class<?> selfType) {
        super(actual, selfType);
    }

    public static SpanAssert<?> assertThat(FinishedSpan actual) {
        return new SpanAssert<>(actual, SpanAssert.class);
    }

    public static SpanAssert<?> then(FinishedSpan actual) {
        return assertThat(actual);
    }

    // --- Name ---

    @SuppressWarnings("unchecked")
    public SELF hasNameEqualTo(String name) {
        isNotNull();
        if (!Objects.equals(actual.getName(), name)) {
            failWithMessage("Expected span name <%s> but was <%s>", name, actual.getName());
        }
        return (SELF) this;
    }

    @SuppressWarnings("unchecked")
    public SELF doesNotHaveNameEqualTo(String name) {
        isNotNull();
        if (Objects.equals(actual.getName(), name)) {
            failWithMessage("Expected span name NOT to be <%s>", name);
        }
        return (SELF) this;
    }

    // --- Tags ---

    @SuppressWarnings("unchecked")
    public SELF hasNoTags() {
        isNotNull();
        if (!actual.getTags().isEmpty()) {
            failWithMessage("Expected no tags but found <%s>", actual.getTags());
        }
        return (SELF) this;
    }

    @SuppressWarnings("unchecked")
    public SELF hasTagWithKey(String key) {
        isNotNull();
        if (!actual.getTags().containsKey(key)) {
            failWithMessage("Expected tag with key <%s> but tags were <%s>", key, actual.getTags());
        }
        return (SELF) this;
    }

    @SuppressWarnings("unchecked")
    public SELF hasTag(String key, String value) {
        isNotNull();
        hasTagWithKey(key);
        String tagValue = actual.getTags().get(key);
        if (!Objects.equals(tagValue, value)) {
            failWithMessage("Expected tag <%s>=<%s> but was <%s>=<%s>",
                    key, value, key, tagValue);
        }
        return (SELF) this;
    }

    @SuppressWarnings("unchecked")
    public SELF doesNotHaveTagWithKey(String key) {
        isNotNull();
        if (actual.getTags().containsKey(key)) {
            failWithMessage("Expected no tag with key <%s> but tags were <%s>",
                    key, actual.getTags());
        }
        return (SELF) this;
    }

    // --- Events ---

    @SuppressWarnings("unchecked")
    public SELF hasEventWithNameEqualTo(String eventName) {
        isNotNull();
        List<String> names = eventNames();
        if (!names.contains(eventName)) {
            failWithMessage("Expected event <%s> but events were <%s>", eventName, names);
        }
        return (SELF) this;
    }

    @SuppressWarnings("unchecked")
    public SELF doesNotHaveEventWithNameEqualTo(String eventName) {
        isNotNull();
        List<String> names = eventNames();
        if (names.contains(eventName)) {
            failWithMessage("Expected no event <%s> but events were <%s>", eventName, names);
        }
        return (SELF) this;
    }

    private List<String> eventNames() {
        return actual.getEvents().stream()
                .map(Map.Entry::getValue)
                .toList();
    }

    // --- Kind ---

    @SuppressWarnings("unchecked")
    public SELF hasKindEqualTo(Span.Kind kind) {
        isNotNull();
        if (!Objects.equals(kind, actual.getKind())) {
            failWithMessage("Expected kind <%s> but was <%s>", kind, actual.getKind());
        }
        return (SELF) this;
    }

    @SuppressWarnings("unchecked")
    public SELF doesNotHaveKindEqualTo(Span.Kind kind) {
        isNotNull();
        if (Objects.equals(kind, actual.getKind())) {
            failWithMessage("Expected kind NOT to be <%s>", kind);
        }
        return (SELF) this;
    }

    // --- Remote Service Name ---

    @SuppressWarnings("unchecked")
    public SELF hasRemoteServiceNameEqualTo(String remoteServiceName) {
        isNotNull();
        if (!Objects.equals(remoteServiceName, actual.getRemoteServiceName())) {
            failWithMessage("Expected remote service name <%s> but was <%s>",
                    remoteServiceName, actual.getRemoteServiceName());
        }
        return (SELF) this;
    }

    @SuppressWarnings("unchecked")
    public SELF doesNotHaveRemoteServiceNameEqualTo(String remoteServiceName) {
        isNotNull();
        if (Objects.equals(remoteServiceName, actual.getRemoteServiceName())) {
            failWithMessage("Expected remote service name NOT to be <%s>", remoteServiceName);
        }
        return (SELF) this;
    }

    // --- Error ---

    @SuppressWarnings("unchecked")
    public SELF hasError() {
        isNotNull();
        if (actual.getError() == null) {
            failWithMessage("Expected span to have an error but it had none");
        }
        return (SELF) this;
    }

    @SuppressWarnings("unchecked")
    public SELF hasNoError() {
        isNotNull();
        if (actual.getError() != null) {
            failWithMessage("Expected span to have no error but found <%s>", actual.getError());
        }
        return (SELF) this;
    }

    // --- Lifecycle ---

    @SuppressWarnings("unchecked")
    public SELF isStarted() {
        isNotNull();
        if (actual.getStartTimestamp().toEpochMilli() == 0) {
            failWithMessage("Expected span to be started");
        }
        return (SELF) this;
    }

    @SuppressWarnings("unchecked")
    public SELF isNotStarted() {
        isNotNull();
        if (actual.getStartTimestamp().toEpochMilli() != 0) {
            failWithMessage("Expected span NOT to be started");
        }
        return (SELF) this;
    }

    @SuppressWarnings("unchecked")
    public SELF isEnded() {
        isNotNull();
        if (actual.getEndTimestamp().toEpochMilli() == 0) {
            failWithMessage("Expected span to be ended");
        }
        return (SELF) this;
    }

    @SuppressWarnings("unchecked")
    public SELF isNotEnded() {
        isNotNull();
        if (actual.getEndTimestamp().toEpochMilli() != 0) {
            failWithMessage("Expected span NOT to be ended");
        }
        return (SELF) this;
    }

    // --- IDs ---

    @SuppressWarnings("unchecked")
    public SELF hasTraceIdEqualTo(String traceId) {
        isNotNull();
        if (!Objects.equals(actual.getTraceId(), traceId)) {
            failWithMessage("Expected traceId <%s> but was <%s>", traceId, actual.getTraceId());
        }
        return (SELF) this;
    }

    @SuppressWarnings("unchecked")
    public SELF hasSpanIdEqualTo(String spanId) {
        isNotNull();
        if (!Objects.equals(actual.getSpanId(), spanId)) {
            failWithMessage("Expected spanId <%s> but was <%s>", spanId, actual.getSpanId());
        }
        return (SELF) this;
    }

    @SuppressWarnings("unchecked")
    public SELF hasParentIdEqualTo(String parentId) {
        isNotNull();
        if (!Objects.equals(parentId, actual.getParentId())) {
            failWithMessage("Expected parentId <%s> but was <%s>", parentId, actual.getParentId());
        }
        return (SELF) this;
    }

}
```

#### File: `src/main/java/dev/linhvu/tracing/simple/SpansAssert.java` [NEW]

```java
package dev.linhvu.tracing.simple;

import dev.linhvu.tracing.exporter.FinishedSpan;
import org.assertj.core.api.AbstractAssert;

import java.util.Collection;
import java.util.List;
import java.util.stream.Collectors;

/**
 * AssertJ-style fluent assertions on a collection of {@link FinishedSpan}s.
 *
 * <p>Usage:
 * <pre>{@code
 * SpansAssert.assertThat(finishedSpans)
 *     .hasNumberOfSpansEqualTo(3)
 *     .haveSameTraceId()
 *     .hasASpanWithName("http.request")
 *     .assertThatASpanWithNameEqualTo("http.request")
 *         .hasTag("http.method", "GET")
 *         .isEnded()
 *     .backToSpans()
 *     .hasASpanWithRemoteServiceName("user-service");
 * }</pre>
 *
 * <p>The {@link SpansAssertReturningAssert} inner class enables drill-down
 * into a single span and navigation back to the collection level.
 */
public class SpansAssert extends AbstractAssert<SpansAssert, List<? extends FinishedSpan>> {

    protected SpansAssert(List<? extends FinishedSpan> actual) {
        super(actual, SpansAssert.class);
    }

    public static SpansAssert assertThat(List<? extends FinishedSpan> actual) {
        return new SpansAssert(actual);
    }

    public static SpansAssert assertThat(Collection<? extends FinishedSpan> actual) {
        return new SpansAssert(List.copyOf(actual));
    }

    public static SpansAssert then(List<? extends FinishedSpan> actual) {
        return assertThat(actual);
    }

    // --- Size ---

    public SpansAssert hasNumberOfSpansEqualTo(int expectedNumberOfSpans) {
        isNotNull();
        if (actual.size() != expectedNumberOfSpans) {
            failWithMessage("Expected <%d> spans but found <%d>:\n%s",
                    expectedNumberOfSpans, actual.size(), spansAsString());
        }
        return this;
    }

    public SpansAssert hasNumberOfSpansWithNameEqualTo(String spanName,
            int expectedNumberOfSpans) {
        isNotNull();
        long count = actual.stream().filter(s -> spanName.equals(s.getName())).count();
        if (count != expectedNumberOfSpans) {
            failWithMessage("Expected <%d> spans with name <%s> but found <%d>:\n%s",
                    expectedNumberOfSpans, spanName, count, spansAsString());
        }
        return this;
    }

    // --- Trace ID coherence ---

    public SpansAssert haveSameTraceId() {
        isNotNull();
        long distinctTraceIds = actual.stream()
                .map(FinishedSpan::getTraceId)
                .distinct()
                .count();
        if (distinctTraceIds != 1) {
            failWithMessage(
                    "Expected all spans to have same traceId but found <%d> distinct trace IDs",
                    distinctTraceIds);
        }
        return this;
    }

    // --- Name-based lookup ---

    public SpansAssert hasASpanWithName(String name) {
        isNotNull();
        extractSpanWithName(name);
        return this;
    }

    // --- Remote service name ---

    public SpansAssert hasASpanWithRemoteServiceName(String remoteServiceName) {
        isNotNull();
        boolean found = actual.stream()
                .anyMatch(s -> remoteServiceName.equals(s.getRemoteServiceName()));
        if (!found) {
            failWithMessage(
                    "Expected a span with remote service name <%s> but none found:\n%s",
                    remoteServiceName, spansAsString());
        }
        return this;
    }

    // --- Tag-based lookup ---

    public SpansAssert hasASpanWithATag(String key, String value) {
        isNotNull();
        boolean found = actual.stream()
                .anyMatch(s -> value.equals(s.getTags().get(key)));
        if (!found) {
            failWithMessage(
                    "Expected a span with tag <%s>=<%s> but none found:\n%s",
                    key, value, spansAsString());
        }
        return this;
    }

    public SpansAssert hasASpanWithATagKey(String key) {
        isNotNull();
        boolean found = actual.stream()
                .anyMatch(s -> s.getTags().containsKey(key));
        if (!found) {
            failWithMessage(
                    "Expected a span with tag key <%s> but none found:\n%s",
                    key, spansAsString());
        }
        return this;
    }

    // --- Navigation to single-span assertions ---

    public SpansAssertReturningAssert assertThatASpanWithNameEqualTo(String name) {
        isNotNull();
        FinishedSpan span = extractSpanWithName(name);
        return new SpansAssertReturningAssert(span, this);
    }

    public SpansAssertReturningAssert thenASpanWithNameEqualTo(String name) {
        return assertThatASpanWithNameEqualTo(name);
    }

    // --- Helpers ---

    private FinishedSpan extractSpanWithName(String name) {
        for (FinishedSpan span : actual) {
            if (name.equals(span.getName())) {
                return span;
            }
        }
        failWithMessage("Expected a span with name <%s> but none found:\n%s",
                name, spansAsString());
        return null; // unreachable — failWithMessage throws
    }

    private String spansAsString() {
        return actual.stream()
                .map(Object::toString)
                .collect(Collectors.joining("\n"));
    }

    // --- Inner class: navigation back from SpanAssert to SpansAssert ---

    /**
     * A {@link SpanAssert} subclass that remembers the parent {@link SpansAssert},
     * enabling navigation back via {@link #backToSpans()}.
     *
     * <p>Because {@code SpanAssert} uses self-referential generics
     * ({@code SELF extends SpanAssert<SELF>}), all chained methods on this class
     * return {@code SpansAssertReturningAssert} — so {@code backToSpans()} is
     * always accessible after any assertion.
     */
    public static class SpansAssertReturningAssert
            extends SpanAssert<SpansAssertReturningAssert> {

        private final SpansAssert spansAssert;

        SpansAssertReturningAssert(FinishedSpan span, SpansAssert spansAssert) {
            super(span, SpansAssertReturningAssert.class);
            this.spansAssert = spansAssert;
        }

        public SpansAssert backToSpans() {
            return spansAssert;
        }

    }

}
```

#### File: `src/main/java/dev/linhvu/tracing/simple/TracerAssert.java` [NEW]

```java
package dev.linhvu.tracing.simple;

import dev.linhvu.tracing.exporter.FinishedSpan;
import org.assertj.core.api.AbstractAssert;

import java.util.List;

/**
 * AssertJ-style fluent assertions on a {@link SimpleTracer}, providing
 * convenient entry points to span-level assertions.
 *
 * <p>Usage:
 * <pre>{@code
 * TracerAssert.assertThat(tracer)
 *     .onlySpan()
 *         .hasNameEqualTo("http.request")
 *         .hasTag("http.method", "GET");
 *
 * TracerAssert.assertThat(tracer)
 *     .reportedSpans()
 *         .hasNumberOfSpansEqualTo(3)
 *         .haveSameTraceId();
 * }</pre>
 */
public class TracerAssert extends AbstractAssert<TracerAssert, SimpleTracer> {

    protected TracerAssert(SimpleTracer actual) {
        super(actual, TracerAssert.class);
    }

    public static TracerAssert assertThat(SimpleTracer actual) {
        return new TracerAssert(actual);
    }

    public static TracerAssert then(SimpleTracer actual) {
        return assertThat(actual);
    }

    /**
     * Asserts that the tracer has exactly one span (that was started and ended)
     * and navigates to a {@link SpanAssert} on it.
     */
    public SpanAssert<?> onlySpan() {
        isNotNull();
        return SpanAssert.assertThat(actual.onlySpan());
    }

    /**
     * Navigates to a {@link SpanAssert} on the most recently created span.
     */
    public SpanAssert<?> lastSpan() {
        isNotNull();
        return SpanAssert.assertThat(actual.lastSpan());
    }

    /**
     * Navigates to a {@link SpansAssert} on all spans created by this tracer.
     */
    public SpansAssert reportedSpans() {
        isNotNull();
        List<FinishedSpan> spans = List.copyOf(actual.getSpans());
        return SpansAssert.assertThat(spans);
    }

}
```

### Test Code

#### File: `src/test/java/dev/linhvu/tracing/simple/SpanAssertTest.java` [NEW]

```java
package dev.linhvu.tracing.simple;

import dev.linhvu.tracing.Span;
import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThatThrownBy;

class SpanAssertTest {

    @Test
    void shouldPassNameAssertion_WhenNameMatches() {
        SimpleSpan span = spanWithName("http.request");

        SpanAssert.assertThat(span).hasNameEqualTo("http.request");
    }

    @Test
    void shouldFailNameAssertion_WhenNameDoesNotMatch() {
        SimpleSpan span = spanWithName("http.request");

        assertThatThrownBy(() -> SpanAssert.assertThat(span).hasNameEqualTo("wrong"))
                .isInstanceOf(AssertionError.class)
                .hasMessageContaining("wrong");
    }

    @Test
    void shouldPassDoesNotHaveName_WhenNameDiffers() {
        SimpleSpan span = spanWithName("http.request");

        SpanAssert.assertThat(span).doesNotHaveNameEqualTo("other");
    }

    @Test
    void shouldPassTagAssertion_WhenTagExists() {
        SimpleSpan span = spanWithName("test");
        span.tag("http.method", "GET");

        SpanAssert.assertThat(span)
                .hasTagWithKey("http.method")
                .hasTag("http.method", "GET");
    }

    @Test
    void shouldFailTagAssertion_WhenTagKeyMissing() {
        SimpleSpan span = spanWithName("test");

        assertThatThrownBy(() -> SpanAssert.assertThat(span).hasTagWithKey("missing"))
                .isInstanceOf(AssertionError.class);
    }

    @Test
    void shouldPassNoTags_WhenSpanHasNoTags() {
        SimpleSpan span = spanWithName("test");

        SpanAssert.assertThat(span).hasNoTags();
    }

    @Test
    void shouldPassDoesNotHaveTagWithKey_WhenKeyAbsent() {
        SimpleSpan span = spanWithName("test");
        span.tag("existing", "value");

        SpanAssert.assertThat(span).doesNotHaveTagWithKey("missing");
    }

    @Test
    void shouldPassEventAssertion_WhenEventExists() {
        SimpleSpan span = spanWithName("test");
        span.event("cache.miss");

        SpanAssert.assertThat(span).hasEventWithNameEqualTo("cache.miss");
    }

    @Test
    void shouldFailEventAssertion_WhenEventMissing() {
        SimpleSpan span = spanWithName("test");

        assertThatThrownBy(() ->
                SpanAssert.assertThat(span).hasEventWithNameEqualTo("missing"))
                .isInstanceOf(AssertionError.class);
    }

    @Test
    void shouldPassDoesNotHaveEvent_WhenEventAbsent() {
        SimpleSpan span = spanWithName("test");
        span.event("other.event");

        SpanAssert.assertThat(span).doesNotHaveEventWithNameEqualTo("missing");
    }

    @Test
    void shouldPassKindAssertion_WhenKindMatches() {
        SimpleSpan span = spanWithName("test");
        span.setSpanKind(Span.Kind.CLIENT);

        SpanAssert.assertThat(span).hasKindEqualTo(Span.Kind.CLIENT);
    }

    @Test
    void shouldFailKindAssertion_WhenKindDiffers() {
        SimpleSpan span = spanWithName("test");
        span.setSpanKind(Span.Kind.SERVER);

        assertThatThrownBy(() ->
                SpanAssert.assertThat(span).hasKindEqualTo(Span.Kind.CLIENT))
                .isInstanceOf(AssertionError.class);
    }

    @Test
    void shouldPassDoesNotHaveKind_WhenKindDiffers() {
        SimpleSpan span = spanWithName("test");
        span.setSpanKind(Span.Kind.SERVER);

        SpanAssert.assertThat(span).doesNotHaveKindEqualTo(Span.Kind.CLIENT);
    }

    @Test
    void shouldPassRemoteServiceNameAssertion_WhenMatches() {
        SimpleSpan span = spanWithName("test");
        span.remoteServiceName("user-service");

        SpanAssert.assertThat(span).hasRemoteServiceNameEqualTo("user-service");
    }

    @Test
    void shouldPassDoesNotHaveRemoteServiceName_WhenDiffers() {
        SimpleSpan span = spanWithName("test");
        span.remoteServiceName("user-service");

        SpanAssert.assertThat(span).doesNotHaveRemoteServiceNameEqualTo("other");
    }

    @Test
    void shouldPassHasError_WhenErrorRecorded() {
        SimpleSpan span = spanWithName("test");
        span.error(new RuntimeException("boom"));

        SpanAssert.assertThat(span).hasError();
    }

    @Test
    void shouldPassHasNoError_WhenNoErrorRecorded() {
        SimpleSpan span = spanWithName("test");

        SpanAssert.assertThat(span).hasNoError();
    }

    @Test
    void shouldFailHasError_WhenNoErrorRecorded() {
        SimpleSpan span = spanWithName("test");

        assertThatThrownBy(() -> SpanAssert.assertThat(span).hasError())
                .isInstanceOf(AssertionError.class);
    }

    @Test
    void shouldPassIsStarted_WhenSpanStarted() {
        SimpleSpan span = spanWithName("test");
        span.start();

        SpanAssert.assertThat(span).isStarted();
    }

    @Test
    void shouldPassIsNotStarted_WhenSpanNotStarted() {
        SimpleSpan span = spanWithName("test");

        SpanAssert.assertThat(span).isNotStarted();
    }

    @Test
    void shouldPassIsEnded_WhenSpanEnded() {
        SimpleSpan span = spanWithName("test");
        span.start();
        span.end();

        SpanAssert.assertThat(span).isEnded();
    }

    @Test
    void shouldPassIsNotEnded_WhenSpanNotEnded() {
        SimpleSpan span = spanWithName("test");
        span.start();

        SpanAssert.assertThat(span).isNotEnded();
    }

    @Test
    void shouldPassTraceIdAssertion_WhenMatches() {
        SimpleSpan span = spanWithName("test");
        span.context().setTraceId("abc123");

        SpanAssert.assertThat(span).hasTraceIdEqualTo("abc123");
    }

    @Test
    void shouldPassSpanIdAssertion_WhenMatches() {
        SimpleSpan span = spanWithName("test");
        span.context().setSpanId("def456");

        SpanAssert.assertThat(span).hasSpanIdEqualTo("def456");
    }

    @Test
    void shouldPassParentIdAssertion_WhenMatches() {
        SimpleSpan span = spanWithName("test");
        span.context().setParentId("parent789");

        SpanAssert.assertThat(span).hasParentIdEqualTo("parent789");
    }

    @Test
    void shouldPassParentIdAssertion_WhenEmpty() {
        SimpleSpan span = spanWithName("test");

        // SimpleTraceContext defaults parentId to "" (empty string)
        SpanAssert.assertThat(span).hasParentIdEqualTo("");
    }

    @Test
    void shouldSupportFluentChaining() {
        SimpleSpan span = spanWithName("http.request");
        span.tag("http.method", "GET");
        span.event("retry");
        span.setSpanKind(Span.Kind.CLIENT);
        span.remoteServiceName("backend");
        span.start();
        span.end();

        SpanAssert.assertThat(span)
                .hasNameEqualTo("http.request")
                .hasTag("http.method", "GET")
                .hasEventWithNameEqualTo("retry")
                .hasKindEqualTo(Span.Kind.CLIENT)
                .hasRemoteServiceNameEqualTo("backend")
                .hasNoError()
                .isStarted()
                .isEnded();
    }

    @Test
    void shouldSupportBddStyleThen() {
        SimpleSpan span = spanWithName("test");
        span.start();

        SpanAssert.then(span)
                .hasNameEqualTo("test")
                .isStarted();
    }

    // --- Helper ---

    private SimpleSpan spanWithName(String name) {
        SimpleSpan span = new SimpleSpan();
        span.name(name);
        return span;
    }

}
```

#### File: `src/test/java/dev/linhvu/tracing/simple/SpansAssertTest.java` [NEW]

```java
package dev.linhvu.tracing.simple;

import dev.linhvu.tracing.Span;
import dev.linhvu.tracing.exporter.FinishedSpan;
import org.junit.jupiter.api.Test;

import java.util.List;

import static org.assertj.core.api.Assertions.assertThatThrownBy;

class SpansAssertTest {

    @Test
    void shouldPassHasNumberOfSpans_WhenCountMatches() {
        List<FinishedSpan> spans = List.of(span("a"), span("b"), span("c"));

        SpansAssert.assertThat(spans).hasNumberOfSpansEqualTo(3);
    }

    @Test
    void shouldFailHasNumberOfSpans_WhenCountDiffers() {
        List<FinishedSpan> spans = List.of(span("a"), span("b"));

        assertThatThrownBy(() -> SpansAssert.assertThat(spans).hasNumberOfSpansEqualTo(5))
                .isInstanceOf(AssertionError.class)
                .hasMessageContaining("5");
    }

    @Test
    void shouldPassHaveSameTraceId_WhenAllSame() {
        SimpleSpan s1 = span("a");
        s1.context().setTraceId("trace1");
        SimpleSpan s2 = span("b");
        s2.context().setTraceId("trace1");

        SpansAssert.assertThat(List.<FinishedSpan>of(s1, s2)).haveSameTraceId();
    }

    @Test
    void shouldFailHaveSameTraceId_WhenDifferent() {
        SimpleSpan s1 = span("a");
        s1.context().setTraceId("trace1");
        SimpleSpan s2 = span("b");
        s2.context().setTraceId("trace2");

        assertThatThrownBy(() ->
                SpansAssert.assertThat(List.<FinishedSpan>of(s1, s2)).haveSameTraceId())
                .isInstanceOf(AssertionError.class);
    }

    @Test
    void shouldPassHasASpanWithName_WhenExists() {
        List<FinishedSpan> spans = List.of(span("http.request"), span("db.query"));

        SpansAssert.assertThat(spans).hasASpanWithName("db.query");
    }

    @Test
    void shouldFailHasASpanWithName_WhenMissing() {
        List<FinishedSpan> spans = List.of(span("http.request"));

        assertThatThrownBy(() -> SpansAssert.assertThat(spans).hasASpanWithName("missing"))
                .isInstanceOf(AssertionError.class)
                .hasMessageContaining("missing");
    }

    @Test
    void shouldPassHasNumberOfSpansWithName_WhenCountMatches() {
        List<FinishedSpan> spans = List.of(span("retry"), span("retry"), span("other"));

        SpansAssert.assertThat(spans).hasNumberOfSpansWithNameEqualTo("retry", 2);
    }

    @Test
    void shouldPassHasASpanWithRemoteServiceName_WhenExists() {
        SimpleSpan s = span("rpc");
        s.remoteServiceName("user-service");

        SpansAssert.assertThat(List.<FinishedSpan>of(s))
                .hasASpanWithRemoteServiceName("user-service");
    }

    @Test
    void shouldPassHasASpanWithATag_WhenExists() {
        SimpleSpan s = span("http");
        s.tag("http.status", "200");

        SpansAssert.assertThat(List.<FinishedSpan>of(s))
                .hasASpanWithATag("http.status", "200");
    }

    @Test
    void shouldPassHasASpanWithATagKey_WhenKeyExists() {
        SimpleSpan s = span("http");
        s.tag("http.method", "GET");

        SpansAssert.assertThat(List.<FinishedSpan>of(s))
                .hasASpanWithATagKey("http.method");
    }

    @Test
    void shouldNavigateToSpanAndBack() {
        SimpleSpan s1 = span("http.request");
        s1.tag("http.method", "GET");
        s1.setSpanKind(Span.Kind.CLIENT);
        s1.start();
        s1.end();

        SimpleSpan s2 = span("db.query");
        s2.start();
        s2.end();

        SpansAssert.assertThat(List.<FinishedSpan>of(s1, s2))
                .hasNumberOfSpansEqualTo(2)
                .assertThatASpanWithNameEqualTo("http.request")
                    .hasTag("http.method", "GET")
                    .hasKindEqualTo(Span.Kind.CLIENT)
                    .isEnded()
                .backToSpans()
                .hasASpanWithName("db.query");
    }

    @Test
    void shouldSupportThenASpanAlias() {
        SimpleSpan s = span("test");
        s.start();

        SpansAssert.assertThat(List.<FinishedSpan>of(s))
                .thenASpanWithNameEqualTo("test")
                    .isStarted()
                .backToSpans()
                .hasNumberOfSpansEqualTo(1);
    }

    @Test
    void shouldSupportBddStyleThen() {
        List<FinishedSpan> spans = List.of(span("a"), span("b"));

        SpansAssert.then(spans).hasNumberOfSpansEqualTo(2);
    }

    // --- Helper ---

    private SimpleSpan span(String name) {
        SimpleSpan s = new SimpleSpan();
        s.name(name);
        return s;
    }

}
```

#### File: `src/test/java/dev/linhvu/tracing/simple/TracerAssertTest.java` [NEW]

```java
package dev.linhvu.tracing.simple;

import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThatThrownBy;

class TracerAssertTest {

    @Test
    void shouldNavigateToOnlySpan_WhenSingleSpanExists() {
        SimpleTracer tracer = new SimpleTracer();
        tracer.nextSpan().name("http.request").start().tag("k", "v").end();

        TracerAssert.assertThat(tracer)
                .onlySpan()
                .hasNameEqualTo("http.request")
                .hasTag("k", "v")
                .isStarted()
                .isEnded();
    }

    @Test
    void shouldFailOnlySpan_WhenMultipleSpansExist() {
        SimpleTracer tracer = new SimpleTracer();
        tracer.nextSpan().name("a").start().end();
        tracer.nextSpan().name("b").start().end();

        assertThatThrownBy(() -> TracerAssert.assertThat(tracer).onlySpan())
                .isInstanceOf(AssertionError.class)
                .hasMessageContaining("1");
    }

    @Test
    void shouldNavigateToLastSpan() {
        SimpleTracer tracer = new SimpleTracer();
        tracer.nextSpan().name("first").start().end();
        tracer.nextSpan().name("second").start().end();

        TracerAssert.assertThat(tracer)
                .lastSpan()
                .hasNameEqualTo("second");
    }

    @Test
    void shouldNavigateToReportedSpans() {
        SimpleTracer tracer = new SimpleTracer();
        tracer.nextSpan().name("a").start().end();
        tracer.nextSpan().name("b").start().end();
        tracer.nextSpan().name("c").start().end();

        TracerAssert.assertThat(tracer)
                .reportedSpans()
                .hasNumberOfSpansEqualTo(3)
                .hasASpanWithName("b");
    }

    @Test
    void shouldSupportBddStyleThen() {
        SimpleTracer tracer = new SimpleTracer();
        tracer.nextSpan().name("test").start().end();

        TracerAssert.then(tracer)
                .onlySpan()
                .hasNameEqualTo("test");
    }

    @Test
    void shouldVerifyParentChildViaReportedSpans() {
        SimpleTracer tracer = new SimpleTracer();
        SimpleSpan parent = tracer.nextSpan().name("parent").start();
        SimpleSpan child = tracer.nextSpan(parent).name("child").start();
        child.end();
        parent.end();

        TracerAssert.assertThat(tracer)
                .reportedSpans()
                .hasNumberOfSpansEqualTo(2)
                .haveSameTraceId()
                .assertThatASpanWithNameEqualTo("child")
                    .hasParentIdEqualTo(parent.context().spanId())
                .backToSpans()
                .assertThatASpanWithNameEqualTo("parent")
                    .hasParentIdEqualTo("");
    }

}
```

#### File: `src/test/java/dev/linhvu/tracing/integration/TestAssertionsIntegrationTest.java` [NEW]

```java
package dev.linhvu.tracing.integration;

import dev.linhvu.tracing.Span;
import dev.linhvu.tracing.exporter.FinishedSpan;
import dev.linhvu.tracing.exporter.SpanFilter;
import dev.linhvu.tracing.exporter.SpanReporter;
import dev.linhvu.tracing.simple.SimpleSpan;
import dev.linhvu.tracing.simple.SimpleTracer;
import dev.linhvu.tracing.simple.SpanAssert;
import dev.linhvu.tracing.simple.SpansAssert;
import dev.linhvu.tracing.simple.TracerAssert;
import org.junit.jupiter.api.Test;

import java.util.ArrayList;
import java.util.List;

class TestAssertionsIntegrationTest {

    @Test
    void shouldAssertFullTracingWorkflow() {
        // Arrange: create a tracer and perform some traced work
        SimpleTracer tracer = new SimpleTracer();

        // Simulate an HTTP server span with a child DB query
        SimpleSpan httpSpan = (SimpleSpan) tracer.spanBuilder()
                .name("http.request")
                .kind(Span.Kind.SERVER)
                .remoteServiceName("gateway")
                .tag("http.method", "GET")
                .tag("http.url", "/api/users")
                .start();

        try (var scope = tracer.withSpan(httpSpan)) {
            SimpleSpan dbSpan = tracer.nextSpan().name("db.query").start();
            dbSpan.tag("db.type", "postgresql");
            dbSpan.event("connection.acquired");
            dbSpan.end();
        }

        httpSpan.event("response.sent");
        httpSpan.end();

        // Assert using TracerAssert: navigate through tracer → spans → single span
        TracerAssert.assertThat(tracer)
                .reportedSpans()
                .hasNumberOfSpansEqualTo(2)
                .haveSameTraceId()
                .hasASpanWithName("http.request")
                .hasASpanWithName("db.query")
                .assertThatASpanWithNameEqualTo("http.request")
                    .hasTag("http.method", "GET")
                    .hasKindEqualTo(Span.Kind.SERVER)
                    .hasRemoteServiceNameEqualTo("gateway")
                    .hasEventWithNameEqualTo("response.sent")
                    .hasNoError()
                    .isStarted()
                    .isEnded()
                .backToSpans()
                .assertThatASpanWithNameEqualTo("db.query")
                    .hasTag("db.type", "postgresql")
                    .hasEventWithNameEqualTo("connection.acquired")
                    .hasParentIdEqualTo(httpSpan.context().spanId())
                .backToSpans();
    }

    @Test
    void shouldAssertSpansFromExportPipeline() {
        // Arrange: tracer with a filter that prefixes span names
        List<FinishedSpan> reported = new ArrayList<>();
        SpanFilter prefixer = span -> span.setName("traced." + span.getName());
        SpanReporter collector = reported::add;

        SimpleTracer tracer = new SimpleTracer(List.of(), List.of(prefixer), List.of(collector));
        tracer.nextSpan().name("request").start().end();

        // Assert on the filtered/exported spans
        SpansAssert.assertThat(reported)
                .hasNumberOfSpansEqualTo(1)
                .assertThatASpanWithNameEqualTo("traced.request")
                    .isStarted()
                    .isEnded()
                .backToSpans();
    }

    @Test
    void shouldAssertErrorSpan() {
        SimpleTracer tracer = new SimpleTracer();
        SimpleSpan span = tracer.nextSpan().name("failing.operation").start();
        span.error(new RuntimeException("connection refused"));
        span.end();

        TracerAssert.assertThat(tracer)
                .onlySpan()
                .hasNameEqualTo("failing.operation")
                .hasError()
                .isEnded();
    }

    @Test
    void shouldUseSpanAssertDirectlyOnFinishedSpan() {
        SimpleSpan span = new SimpleSpan();
        span.name("standalone");
        span.tag("env", "test");
        span.start();
        span.end();

        SpanAssert.assertThat((FinishedSpan) span)
                .hasNameEqualTo("standalone")
                .hasTag("env", "test")
                .isStarted()
                .isEnded()
                .hasNoError();
    }

}
```

---

## Summary

| Concept | What It Means |
|---------|---------------|
| **`SpanAssert`** | Fluent assertions on a single `FinishedSpan` — check name, tags, events, kind, errors, lifecycle, and IDs |
| **`SpansAssert`** | Collection-level assertions — check span count, trace ID coherence, find spans by name/tag/service, drill into individual spans |
| **`TracerAssert`** | Entry-point assertions on `SimpleTracer` — navigate to `onlySpan()`, `lastSpan()`, or `reportedSpans()` |
| **Self-referential generics** | `SELF extends SpanAssert<SELF>` allows subclasses to return their own type from fluent methods, enabling bidirectional navigation |
| **`SpansAssertReturningAssert`** | Inner class that extends `SpanAssert` and adds `backToSpans()` — enables the two-level navigation pattern |

**This is the final chapter.** All 11 features of simple-tracing are now implemented — from the core data model (`Span`, `TraceContext`) through the in-memory tracer, export pipeline, propagation, baggage, observation handlers, annotation-based tracing, and now test assertions. You have a complete simplified distributed tracing facade that mirrors the architecture of Micrometer Tracing.
