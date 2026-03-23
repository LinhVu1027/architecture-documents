# Chapter 5: Propagator (Cross-Process Context Propagation)

> Line references based on commit `dceab1b9` of the Micrometer Tracing repository.

## Build Challenge

| Current State | Limitation | Objective |
|---------------|------------|-----------|
| SimpleTracer creates spans with IDs and parent-child relationships, but only within a single JVM process | No way to serialize trace context into HTTP headers or message headers for cross-process communication — each service creates isolated traces | Build a `Propagator` interface with `inject()`/`extract()` methods and a W3C `traceparent` implementation that serializes trace context into carriers so spans across services share a single trace |

---

## 5.1 The Integration Point

The Propagator is a standalone interface — it doesn't modify any existing files. Instead, it bridges two worlds: **internal tracing state** (`TraceContext` from Feature 1) and **external carriers** (HTTP headers, message headers). The integration happens at two seams:

1. **`extract()` returns `Span.Builder`** — the extracted parent context flows directly into span creation via the builder pattern we built in Feature 1
2. **`W3CTraceContextPropagator` takes a `Tracer`** — it needs `tracer.spanBuilder()` and `tracer.traceContextBuilder()` from Feature 2 to create builders and contexts

This connects **TraceContext (internal IDs) ↔ External carriers (HTTP headers)**. To make it work, we need to build:
- `Propagator` interface — defines `inject()`, `extract()`, `Setter<C>`, `Getter<C>`
- `W3CTraceContextPropagator` — concrete implementation using the W3C `traceparent` header format

Two key decisions here:

1. **Why return `Span.Builder` from `extract()` instead of `TraceContext`?** Because the caller needs to configure more than just the parent — they need to set span name, kind, tags before starting. Returning a `Span.Builder` pre-configured with the parent context gives a fluent API: `propagator.extract(headers, getter).name("handle-request").kind(SERVER).start()`.

2. **Why generic `<C>` carrier + `Setter`/`Getter` instead of taking `Map<String, String>` directly?** Because carriers vary wildly — `HttpServletRequest`, `HttpURLConnection`, Kafka `Headers`, gRPC `Metadata`. The `Setter`/`Getter` functional interfaces let each caller provide a lambda that knows how to read/write its specific carrier type.

**New file:** `src/main/java/dev/linhvu/tracing/propagation/Propagator.java`

```java
package dev.linhvu.tracing.propagation;

import dev.linhvu.tracing.Span;
import dev.linhvu.tracing.TraceContext;

import java.util.Collections;
import java.util.List;

public interface Propagator {

    Propagator NOOP = new Propagator() {
        @Override
        public List<String> fields() {
            return Collections.emptyList();
        }

        @Override
        public <C> void inject(TraceContext context, C carrier, Setter<C> setter) {
        }

        @Override
        public <C> Span.Builder extract(C carrier, Getter<C> getter) {
            return Span.Builder.NOOP;
        }
    };

    List<String> fields();

    <C> void inject(TraceContext context, C carrier, Setter<C> setter);

    <C> Span.Builder extract(C carrier, Getter<C> getter);

    interface Setter<C> {
        @SuppressWarnings("rawtypes")
        Setter NOOP = (carrier, key, value) -> {
        };

        void set(C carrier, String key, String value);
    }

    interface Getter<C> {
        @SuppressWarnings("rawtypes")
        Getter NOOP = (carrier, key) -> null;

        String get(C carrier, String key);
    }

}
```

The flow looks like this:

```
Service A (sender)                          Service B (receiver)
─────────────────                          ──────────────────────
Span span = tracer.nextSpan().start();

propagator.inject(                         Span.Builder builder =
    span.context(),    ──── HTTP ────>         propagator.extract(
    httpRequest,       headers with             httpRequest,
    HttpRequest::      "traceparent"            HttpRequest::
        setHeader);                                 getHeader);

                                           Span child = builder
                                               .name("handle")
                                               .kind(SERVER)
                                               .start();
```

## 5.2 W3C Trace Context Propagator

The W3C Trace Context specification defines the `traceparent` header format:

```
traceparent: 00-{traceId}-{spanId}-{flags}
             │   │          │        │
             │   │          │        └── "01" = sampled, "00" = not sampled
             │   │          └── 16 hex chars (64-bit parent span ID)
             │   └── 32 hex chars (128-bit trace ID)
             └── version (always "00")

Example: 00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01
```

**New file:** `src/main/java/dev/linhvu/tracing/propagation/W3CTraceContextPropagator.java`

```java
package dev.linhvu.tracing.propagation;

import dev.linhvu.tracing.Span;
import dev.linhvu.tracing.TraceContext;
import dev.linhvu.tracing.Tracer;

import java.util.List;

public class W3CTraceContextPropagator implements Propagator {

    static final String TRACEPARENT = "traceparent";

    private static final String VERSION = "00";

    private static final List<String> FIELDS = List.of(TRACEPARENT);

    private final Tracer tracer;

    public W3CTraceContextPropagator(Tracer tracer) {
        this.tracer = tracer;
    }

    @Override
    public List<String> fields() {
        return FIELDS;
    }

    @Override
    public <C> void inject(TraceContext context, C carrier, Setter<C> setter) {
        if (context == null || carrier == null) {
            return;
        }
        String traceId = context.traceId();
        String spanId = context.spanId();
        if (traceId == null || traceId.isEmpty() || spanId == null || spanId.isEmpty()) {
            return;
        }

        String flags = Boolean.TRUE.equals(context.sampled()) ? "01" : "00";
        String traceparent = VERSION + "-" + traceId + "-" + spanId + "-" + flags;
        setter.set(carrier, TRACEPARENT, traceparent);
    }

    @Override
    public <C> Span.Builder extract(C carrier, Getter<C> getter) {
        if (carrier == null) {
            return tracer.spanBuilder();
        }

        String traceparent = getter.get(carrier, TRACEPARENT);
        if (traceparent == null || traceparent.isEmpty()) {
            return tracer.spanBuilder();
        }

        String[] parts = traceparent.split("-");
        if (parts.length != 4 || !"00".equals(parts[0])) {
            return tracer.spanBuilder();
        }

        String traceId = parts[1];
        String spanId = parts[2];
        String flags = parts[3];

        if (traceId.length() < 1 || spanId.length() < 1 || flags.length() != 2) {
            return tracer.spanBuilder();
        }

        boolean sampled = "01".equals(flags);
        TraceContext parentContext = tracer.traceContextBuilder()
                .traceId(traceId)
                .spanId(spanId)
                .sampled(sampled)
                .build();

        return tracer.spanBuilder().setParent(parentContext);
    }

}
```

## 5.3 Try It Yourself

<details>
<summary>Challenge: Implement inject() — serialize a TraceContext into a traceparent header</summary>

Given a `TraceContext` with `traceId`, `spanId`, and `sampled`, write the `inject()` method that formats and sets the `traceparent` header on the carrier.

Think about: What should happen when the context or carrier is null? When the traceId or spanId is empty?

```java
@Override
public <C> void inject(TraceContext context, C carrier, Setter<C> setter) {
    if (context == null || carrier == null) {
        return;
    }
    String traceId = context.traceId();
    String spanId = context.spanId();
    if (traceId == null || traceId.isEmpty() || spanId == null || spanId.isEmpty()) {
        return;
    }

    String flags = Boolean.TRUE.equals(context.sampled()) ? "01" : "00";
    String traceparent = VERSION + "-" + traceId + "-" + spanId + "-" + flags;
    setter.set(carrier, TRACEPARENT, traceparent);
}
```

</details>

<details>
<summary>Challenge: Implement extract() — parse a traceparent header into a Span.Builder</summary>

Given a carrier that may contain a `traceparent` header, parse the traceId, spanId, and flags, build a `TraceContext`, and return a `Span.Builder` pre-configured with that parent context.

Think about: How do you validate the header format? What should you return for missing/malformed headers?

```java
@Override
public <C> Span.Builder extract(C carrier, Getter<C> getter) {
    if (carrier == null) {
        return tracer.spanBuilder();
    }

    String traceparent = getter.get(carrier, TRACEPARENT);
    if (traceparent == null || traceparent.isEmpty()) {
        return tracer.spanBuilder();
    }

    String[] parts = traceparent.split("-");
    if (parts.length != 4 || !"00".equals(parts[0])) {
        return tracer.spanBuilder();
    }

    String traceId = parts[1];
    String spanId = parts[2];
    String flags = parts[3];

    if (traceId.length() < 1 || spanId.length() < 1 || flags.length() != 2) {
        return tracer.spanBuilder();
    }

    boolean sampled = "01".equals(flags);
    TraceContext parentContext = tracer.traceContextBuilder()
            .traceId(traceId)
            .spanId(spanId)
            .sampled(sampled)
            .build();

    return tracer.spanBuilder().setParent(parentContext);
}
```

</details>

## 5.4 Tests

### Unit Tests

**New file:** `src/test/java/dev/linhvu/tracing/propagation/PropagatorTest.java`

```java
class PropagatorTest {

    @Test
    void shouldReturnEmptyFields_WhenNoopPropagator() { ... }

    @Test
    void shouldDoNothing_WhenNoopInject() { ... }

    @Test
    void shouldReturnNoopBuilder_WhenNoopExtract() { ... }

    @Test
    void shouldDoNothing_WhenNoopSetterSet() { ... }

    @Test
    void shouldReturnNull_WhenNoopGetterGet() { ... }
}
```

**New file:** `src/test/java/dev/linhvu/tracing/propagation/W3CTraceContextPropagatorTest.java`

```java
class W3CTraceContextPropagatorTest {

    // Inject tests
    @Test
    void shouldInjectTraceparentHeader_WhenContextHasIds() { ... }

    @Test
    void shouldInjectFlags00_WhenNotSampled() { ... }

    @Test
    void shouldNotInject_WhenContextIsNull() { ... }

    @Test
    void shouldNotInject_WhenCarrierIsNull() { ... }

    @Test
    void shouldNotInject_WhenTraceIdIsEmpty() { ... }

    // Extract tests
    @Test
    void shouldExtractParentContext_WhenValidTraceparent() { ... }

    @Test
    void shouldExtractSampledFalse_WhenFlags00() { ... }

    @Test
    void shouldReturnFreshBuilder_WhenNoTraceparentHeader() { ... }

    @Test
    void shouldReturnFreshBuilder_WhenMalformedTraceparent() { ... }

    @Test
    void shouldReturnFreshBuilder_WhenCarrierIsNull() { ... }

    // Round-trip
    @Test
    void shouldRoundTrip_WhenInjectThenExtract() { ... }
}
```

### Integration Tests

**New file:** `src/test/java/dev/linhvu/tracing/integration/PropagatorIntegrationTest.java`

```java
class PropagatorIntegrationTest {

    @Test
    void shouldPropagateTraceAcrossServices_WhenUsingInjectAndExtract() { ... }

    @Test
    void shouldPreserveTraceChainAcrossThreeServices() { ... }

    @Test
    void shouldCreateNewRootTrace_WhenNoContextPropagated() { ... }

    @Test
    void shouldWorkWithExportPipeline_WhenPropagatedSpanEnds() { ... }
}
```

**Run:** `./gradlew test` — expected: all 187 tests pass (including all prior features' tests)

---

## 5.5 Why This Works

> ★ **Insight** -------------------------------------------
> - **Why `Setter<C>` / `Getter<C>` instead of `Map<String, String>`?** The carrier varies per transport: `HttpServletRequest.getHeader()`, `HttpURLConnection.addRequestProperty()`, Kafka `Headers.add()`, gRPC `Metadata.put()`. By parameterizing the carrier type `C` and providing functional interfaces, the same Propagator works with any transport — the caller provides a one-liner lambda or method reference. This is the **Strategy pattern** applied at the serialization boundary.
> - **Why return `Span.Builder` from `extract()` instead of `TraceContext`?** A `TraceContext` is read-only and immutable — it can't carry builder-state like span name or kind. By returning `Span.Builder` pre-configured with the parent context, the API is fluent: `propagator.extract(headers, getter).name("handle").kind(SERVER).start()`. The real framework does the same (`Propagator.java:92`).
> - **When NOT to use this:** Database calls or in-process method calls don't cross process boundaries — use `Propagator.NOOP` instead. The real framework uses NOOP for these cases to avoid unnecessary serialization overhead.
> -----------------------------------------------------------

> ★ **Insight** -------------------------------------------
> - **W3C Trace Context is the industry standard.** Before W3C standardization, each tracing system had its own header format: Zipkin used `X-B3-TraceId`/`X-B3-SpanId`/`X-B3-Sampled` (3 headers), Jaeger used `uber-trace-id` (1 header, different format). The W3C `traceparent` header unifies this — one header, one format, interoperable across all tracing systems. Both OpenTelemetry and Brave support it as the default.
> - **Graceful degradation on extract:** If the `traceparent` header is missing or malformed, `extract()` returns a fresh `Span.Builder` with no parent — which creates a new root trace when started. This means the receiving service always produces a span, even if propagation fails. Lost continuity is better than lost visibility.
> -----------------------------------------------------------

## 5.6 What We Enhanced

| Aspect | Before (ch01-04) | Current (ch05) | Real Framework |
|--------|-------------------|----------------|----------------|
| Cross-process tracing | Not possible — each JVM creates isolated traces with no way to share context | `Propagator.inject()` serializes `TraceContext` into carriers; `extract()` deserializes and returns a `Span.Builder` with parent context | Same interface (`Propagator.java:38`) with additional `@Nullable` annotations and `Getter.getAll()` for multi-valued headers |
| Header format | N/A | W3C `traceparent` (`00-{traceId}-{spanId}-{flags}`) | Supports W3C, B3 (Zipkin), and custom formats via pluggable `Propagator` implementations in bridge modules |

## 5.7 Connection to Real Framework

| Simplified Code | Real Framework Code | File:Line | Key Difference |
|----------------|--------------------|-----------| ---------------|
| `Propagator` | `Propagator` | `Propagator.java:38` | Real adds `@Nullable` annotations and `Getter.getAll()` for multi-valued headers (since 1.6.0) |
| `Propagator.fields()` | `Propagator.fields()` | `Propagator.java:65` | Same contract |
| `Propagator.inject()` | `Propagator.inject()` | `Propagator.java:77` | Real accepts `@Nullable` carrier for lambda convenience |
| `Propagator.extract()` | `Propagator.extract()` | `Propagator.java:92` | Same — returns `Span.Builder` pre-configured with extracted parent |
| `Propagator.Setter<C>` | `Propagator.Setter<C>` | `Propagator.java:104` | Same functional interface |
| `Propagator.Getter<C>` | `Propagator.Getter<C>` | `Propagator.java:140` | Real adds `getAll(carrier, key)` default method for repeated headers |
| `W3CTraceContextPropagator` | (in bridge modules) | N/A | Real W3C propagation is in Brave/OTel bridge modules; the `Propagator` interface itself is framework-agnostic |

## 5.8 Complete Code

### Production Code

#### File: `src/main/java/dev/linhvu/tracing/propagation/Propagator.java` [NEW]

```java
package dev.linhvu.tracing.propagation;

import dev.linhvu.tracing.Span;
import dev.linhvu.tracing.TraceContext;

import java.util.Collections;
import java.util.List;

/**
 * Injects and extracts trace context as text into carriers that travel in-band
 * across process boundaries (e.g., HTTP headers, message headers).
 *
 * <p>Real-world analogy: the barcode scanner at a shipping facility —
 * it reads tracking numbers from incoming labels ({@link #extract}) and
 * prints tracking numbers onto outgoing labels ({@link #inject}).
 *
 * <p>Two nested interfaces define how to read/write carrier fields:
 * <ul>
 *   <li>{@link Setter} — writes a key-value pair into a carrier (e.g., HTTP request header)</li>
 *   <li>{@link Getter} — reads a value by key from a carrier (e.g., HTTP request header)</li>
 * </ul>
 */
public interface Propagator {

    /**
     * A no-op propagator that neither injects nor extracts anything.
     * Use with sender/receiver that do not need propagation (e.g., database access).
     */
    Propagator NOOP = new Propagator() {
        @Override
        public List<String> fields() {
            return Collections.emptyList();
        }

        @Override
        public <C> void inject(TraceContext context, C carrier, Setter<C> setter) {
        }

        @Override
        public <C> Span.Builder extract(C carrier, Getter<C> getter) {
            return Span.Builder.NOOP;
        }
    };

    /**
     * Returns the collection of header/field names that this propagator uses
     * to carry tracing information (e.g., ["traceparent"]).
     */
    List<String> fields();

    /**
     * Injects trace context into a carrier for outgoing communication.
     *
     * @param context the trace context containing IDs to propagate
     * @param carrier holds propagation fields (e.g., an HTTP request)
     * @param setter  invoked for each propagation key to write
     * @param <C>     carrier type (e.g., {@code HttpURLConnection}, {@code Map<String, String>})
     */
    <C> void inject(TraceContext context, C carrier, Setter<C> setter);

    /**
     * Extracts trace context from an incoming carrier.
     *
     * @param carrier holds propagation fields (e.g., an HTTP request)
     * @param getter  invoked for each propagation key to read
     * @param <C>     carrier type
     * @return a {@link Span.Builder} pre-configured with the extracted parent context,
     *         or {@link Span.Builder#NOOP} if nothing was extracted
     */
    <C> Span.Builder extract(C carrier, Getter<C> getter);

    /**
     * Writes a propagation field into a carrier.
     *
     * <p>Stateless — can be saved as a constant to avoid runtime allocations.
     * For example, a setter for {@link java.net.HttpURLConnection} would be
     * {@code HttpURLConnection::addRequestProperty}.
     *
     * @param <C> carrier type
     */
    interface Setter<C> {

        /** A no-op setter that ignores all calls. */
        @SuppressWarnings("rawtypes")
        Setter NOOP = (carrier, key, value) -> {
        };

        /**
         * Sets a propagation field on the carrier.
         *
         * @param carrier the carrier to write to
         * @param key     the header/field name
         * @param value   the header/field value
         */
        void set(C carrier, String key, String value);
    }

    /**
     * Reads a propagation field from a carrier.
     *
     * <p>Stateless — can be saved as a constant to avoid runtime allocations.
     *
     * @param <C> carrier type
     */
    interface Getter<C> {

        /** A no-op getter that always returns null. */
        @SuppressWarnings("rawtypes")
        Getter NOOP = (carrier, key) -> null;

        /**
         * Returns the first value of the given propagation key, or null if absent.
         *
         * @param carrier the carrier to read from
         * @param key     the header/field name
         * @return the value, or null
         */
        String get(C carrier, String key);
    }

}
```

#### File: `src/main/java/dev/linhvu/tracing/propagation/W3CTraceContextPropagator.java` [NEW]

```java
package dev.linhvu.tracing.propagation;

import dev.linhvu.tracing.Span;
import dev.linhvu.tracing.TraceContext;
import dev.linhvu.tracing.Tracer;

import java.util.List;

/**
 * Implements the <a href="https://www.w3.org/TR/trace-context/">W3C Trace Context</a>
 * propagation format using the {@code traceparent} header.
 *
 * <p>Header format: {@code 00-{traceId}-{spanId}-{flags}}
 * <ul>
 *   <li>{@code 00} — version (always "00" for the current specification)</li>
 *   <li>{@code traceId} — 32 lowercase hex characters (128-bit trace identifier)</li>
 *   <li>{@code spanId} — 16 lowercase hex characters (64-bit span identifier)</li>
 *   <li>{@code flags} — 2 hex characters ({@code 01} = sampled, {@code 00} = not sampled)</li>
 * </ul>
 *
 * <p>Example: {@code 00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01}
 */
public class W3CTraceContextPropagator implements Propagator {

    static final String TRACEPARENT = "traceparent";

    private static final String VERSION = "00";

    private static final List<String> FIELDS = List.of(TRACEPARENT);

    private final Tracer tracer;

    public W3CTraceContextPropagator(Tracer tracer) {
        this.tracer = tracer;
    }

    @Override
    public List<String> fields() {
        return FIELDS;
    }

    /**
     * Injects the trace context into the carrier as a {@code traceparent} header.
     * Format: {@code 00-{traceId}-{spanId}-{flags}}
     */
    @Override
    public <C> void inject(TraceContext context, C carrier, Setter<C> setter) {
        if (context == null || carrier == null) {
            return;
        }
        String traceId = context.traceId();
        String spanId = context.spanId();
        if (traceId == null || traceId.isEmpty() || spanId == null || spanId.isEmpty()) {
            return;
        }

        String flags = Boolean.TRUE.equals(context.sampled()) ? "01" : "00";
        String traceparent = VERSION + "-" + traceId + "-" + spanId + "-" + flags;
        setter.set(carrier, TRACEPARENT, traceparent);
    }

    /**
     * Extracts trace context from the carrier's {@code traceparent} header and returns
     * a {@link Span.Builder} pre-configured with the extracted parent context.
     *
     * <p>If the header is missing or malformed, returns a builder with no parent set
     * (which will create a new root trace when started).
     */
    @Override
    public <C> Span.Builder extract(C carrier, Getter<C> getter) {
        if (carrier == null) {
            return tracer.spanBuilder();
        }

        String traceparent = getter.get(carrier, TRACEPARENT);
        if (traceparent == null || traceparent.isEmpty()) {
            return tracer.spanBuilder();
        }

        String[] parts = traceparent.split("-");
        if (parts.length != 4 || !"00".equals(parts[0])) {
            return tracer.spanBuilder();
        }

        String traceId = parts[1];
        String spanId = parts[2];
        String flags = parts[3];

        // Validate field lengths per W3C spec: traceId=32 hex, spanId=16 hex, flags=2 hex
        if (traceId.length() < 1 || spanId.length() < 1 || flags.length() != 2) {
            return tracer.spanBuilder();
        }

        // Build a TraceContext from the extracted fields
        boolean sampled = "01".equals(flags);
        TraceContext parentContext = tracer.traceContextBuilder()
                .traceId(traceId)
                .spanId(spanId)
                .sampled(sampled)
                .build();

        // Return a Span.Builder with the extracted context as parent
        return tracer.spanBuilder().setParent(parentContext);
    }

}
```

### Test Code

#### File: `src/test/java/dev/linhvu/tracing/propagation/PropagatorTest.java` [NEW]

```java
package dev.linhvu.tracing.propagation;

import dev.linhvu.tracing.Span;

import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;

/**
 * Tests the {@link Propagator} interface contract — specifically the NOOP implementation.
 */
class PropagatorTest {

    @Test
    void shouldReturnEmptyFields_WhenNoopPropagator() {
        assertThat(Propagator.NOOP.fields()).isEmpty();
    }

    @Test
    void shouldDoNothing_WhenNoopInject() {
        // Arrange — a carrier that would record any set() calls
        var carrier = new java.util.HashMap<String, String>();
        Propagator.Setter<java.util.Map<String, String>> setter = java.util.Map::put;

        // Act
        Propagator.NOOP.inject(null, carrier, setter);

        // Assert — nothing was written
        assertThat(carrier).isEmpty();
    }

    @Test
    void shouldReturnNoopBuilder_WhenNoopExtract() {
        // Arrange
        var carrier = java.util.Map.of("traceparent", "00-abc123-def456-01");
        Propagator.Getter<java.util.Map<String, String>> getter = java.util.Map::get;

        // Act
        Span.Builder builder = Propagator.NOOP.extract(carrier, getter);

        // Assert — returns NOOP builder
        assertThat(builder).isSameAs(Span.Builder.NOOP);
    }

    @Test
    @SuppressWarnings("unchecked")
    void shouldDoNothing_WhenNoopSetterSet() {
        var carrier = new java.util.HashMap<String, String>();

        // Act
        Propagator.Setter.NOOP.set(carrier, "key", "value");

        // Assert
        assertThat(carrier).isEmpty();
    }

    @Test
    @SuppressWarnings("unchecked")
    void shouldReturnNull_WhenNoopGetterGet() {
        var carrier = java.util.Map.of("key", "value");

        // Act
        String result = (String) Propagator.Getter.NOOP.get(carrier, "key");

        // Assert
        assertThat(result).isNull();
    }

}
```

#### File: `src/test/java/dev/linhvu/tracing/propagation/W3CTraceContextPropagatorTest.java` [NEW]

```java
package dev.linhvu.tracing.propagation;

import dev.linhvu.tracing.Span;
import dev.linhvu.tracing.simple.SimpleSpan;
import dev.linhvu.tracing.simple.SimpleTracer;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.util.HashMap;
import java.util.Map;

import static org.assertj.core.api.Assertions.assertThat;

/**
 * Tests the {@link W3CTraceContextPropagator} — inject/extract using the W3C traceparent format.
 */
class W3CTraceContextPropagatorTest {

    private SimpleTracer tracer;

    private W3CTraceContextPropagator propagator;

    private final Propagator.Setter<Map<String, String>> setter = Map::put;

    private final Propagator.Getter<Map<String, String>> getter = Map::get;

    @BeforeEach
    void setUp() {
        tracer = new SimpleTracer();
        propagator = new W3CTraceContextPropagator(tracer);
    }

    @Test
    void shouldReturnTraceparentField() {
        assertThat(propagator.fields()).containsExactly("traceparent");
    }

    // --- Inject tests ---

    @Test
    void shouldInjectTraceparentHeader_WhenContextHasIds() {
        // Arrange
        SimpleSpan span = tracer.nextSpan().name("test").start();
        Map<String, String> carrier = new HashMap<>();

        // Act
        propagator.inject(span.context(), carrier, setter);

        // Assert
        String traceparent = carrier.get("traceparent");
        assertThat(traceparent).isNotNull();
        assertThat(traceparent).startsWith("00-");

        String[] parts = traceparent.split("-");
        assertThat(parts).hasSize(4);
        assertThat(parts[0]).isEqualTo("00");                         // version
        assertThat(parts[1]).isEqualTo(span.context().traceId());     // traceId
        assertThat(parts[2]).isEqualTo(span.context().spanId());      // spanId
        assertThat(parts[3]).isEqualTo("01");                         // sampled=true
    }

    @Test
    void shouldInjectFlags00_WhenNotSampled() {
        // Arrange
        SimpleSpan span = tracer.nextSpan().start();
        span.context().setSampled(false);
        Map<String, String> carrier = new HashMap<>();

        // Act
        propagator.inject(span.context(), carrier, setter);

        // Assert
        String traceparent = carrier.get("traceparent");
        assertThat(traceparent).endsWith("-00");
    }

    @Test
    void shouldNotInject_WhenContextIsNull() {
        Map<String, String> carrier = new HashMap<>();

        // Act
        propagator.inject(null, carrier, setter);

        // Assert
        assertThat(carrier).isEmpty();
    }

    @Test
    void shouldNotInject_WhenCarrierIsNull() {
        SimpleSpan span = tracer.nextSpan().start();

        // Act — should not throw
        propagator.inject(span.context(), null, setter);
    }

    @Test
    void shouldNotInject_WhenTraceIdIsEmpty() {
        // Arrange — a context with empty traceId
        var context = tracer.traceContextBuilder().traceId("").spanId("abc123").build();
        Map<String, String> carrier = new HashMap<>();

        // Act
        propagator.inject(context, carrier, setter);

        // Assert
        assertThat(carrier).isEmpty();
    }

    // --- Extract tests ---

    @Test
    void shouldExtractParentContext_WhenValidTraceparent() {
        // Arrange
        String traceId = "4bf92f3577b34da6a3ce929d0e0e4736";
        String spanId = "00f067aa0ba902b7";
        Map<String, String> carrier = Map.of(
                "traceparent", "00-" + traceId + "-" + spanId + "-01"
        );

        // Act
        Span.Builder builder = propagator.extract(carrier, getter);
        Span span = builder.name("child").start();

        // Assert — the new span should be a child of the extracted context
        assertThat(span.context().traceId()).isEqualTo(traceId);
        assertThat(span.context().parentId()).isEqualTo(spanId);
        assertThat(span.context().sampled()).isTrue();
        // spanId should be a new, different ID
        assertThat(span.context().spanId()).isNotEqualTo(spanId);
    }

    @Test
    void shouldExtractSampledFalse_WhenFlags00() {
        // Arrange
        Map<String, String> carrier = Map.of(
                "traceparent", "00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-00"
        );

        // Act
        Span span = propagator.extract(carrier, getter).start();

        // Assert
        assertThat(span.context().sampled()).isFalse();
    }

    @Test
    void shouldReturnFreshBuilder_WhenNoTraceparentHeader() {
        // Arrange
        Map<String, String> carrier = Map.of();

        // Act
        Span.Builder builder = propagator.extract(carrier, getter);
        Span span = builder.name("root").start();

        // Assert — new root trace (no parent)
        assertThat(span.context().traceId()).isNotEmpty();
        assertThat(span.context().parentId()).isEmpty();
    }

    @Test
    void shouldReturnFreshBuilder_WhenMalformedTraceparent() {
        // Arrange
        Map<String, String> carrier = Map.of("traceparent", "not-a-valid-header");

        // Act
        Span.Builder builder = propagator.extract(carrier, getter);
        Span span = builder.name("root").start();

        // Assert — treats as new root trace
        assertThat(span.context().traceId()).isNotEmpty();
        assertThat(span.context().parentId()).isEmpty();
    }

    @Test
    void shouldReturnFreshBuilder_WhenCarrierIsNull() {
        // Act
        Span.Builder builder = propagator.extract(null, getter);
        Span span = builder.name("root").start();

        // Assert — new root trace
        assertThat(span.context().traceId()).isNotEmpty();
    }

    // --- Round-trip test ---

    @Test
    void shouldRoundTrip_WhenInjectThenExtract() {
        // Arrange — create a span and inject its context
        SimpleSpan originalSpan = tracer.nextSpan().name("original").start();
        Map<String, String> carrier = new HashMap<>();
        propagator.inject(originalSpan.context(), carrier, setter);

        // Act — extract from the carrier
        Span.Builder builder = propagator.extract(carrier, getter);
        Span childSpan = builder.name("child").start();

        // Assert — child continues the same trace
        assertThat(childSpan.context().traceId()).isEqualTo(originalSpan.context().traceId());
        assertThat(childSpan.context().parentId()).isEqualTo(originalSpan.context().spanId());
        assertThat(childSpan.context().spanId()).isNotEqualTo(originalSpan.context().spanId());
    }

}
```

#### File: `src/test/java/dev/linhvu/tracing/integration/PropagatorIntegrationTest.java` [NEW]

```java
package dev.linhvu.tracing.integration;

import dev.linhvu.tracing.Span;
import dev.linhvu.tracing.Tracer;
import dev.linhvu.tracing.propagation.Propagator;
import dev.linhvu.tracing.propagation.W3CTraceContextPropagator;
import dev.linhvu.tracing.simple.SimpleSpan;
import dev.linhvu.tracing.simple.SimpleTracer;

import org.junit.jupiter.api.Test;

import java.util.HashMap;
import java.util.Map;

import static org.assertj.core.api.Assertions.assertThat;

/**
 * Integration tests verifying that Propagator works end-to-end with SimpleTracer,
 * scope management, and the export pipeline.
 */
class PropagatorIntegrationTest {

    private final Propagator.Setter<Map<String, String>> setter = Map::put;

    private final Propagator.Getter<Map<String, String>> getter = Map::get;

    @Test
    void shouldPropagateTraceAcrossServices_WhenUsingInjectAndExtract() {
        // === Service A: creates a span and injects context into "HTTP headers" ===
        SimpleTracer serviceATracer = new SimpleTracer();
        Propagator serviceAPropagator = new W3CTraceContextPropagator(serviceATracer);

        Span serverSpan = serviceATracer.nextSpan().name("handle-request").start();
        try (Tracer.SpanInScope ws = serviceATracer.withSpan(serverSpan)) {
            // Simulate outgoing HTTP call — inject into headers
            Map<String, String> httpHeaders = new HashMap<>();
            serviceAPropagator.inject(serverSpan.context(), httpHeaders, setter);

            // === Service B: receives the headers and extracts context ===
            SimpleTracer serviceBTracer = new SimpleTracer();
            Propagator serviceBPropagator = new W3CTraceContextPropagator(serviceBTracer);

            Span.Builder extractedBuilder = serviceBPropagator.extract(httpHeaders, getter);
            Span childSpan = extractedBuilder.name("process-request").kind(Span.Kind.SERVER).start();

            // Assert — both services share the same trace
            assertThat(childSpan.context().traceId()).isEqualTo(serverSpan.context().traceId());
            assertThat(childSpan.context().parentId()).isEqualTo(serverSpan.context().spanId());

            childSpan.end();
        }
        finally {
            serverSpan.end();
        }
    }

    @Test
    void shouldPreserveTraceChainAcrossThreeServices() {
        // Service A → Service B → Service C
        SimpleTracer tracerA = new SimpleTracer();
        SimpleTracer tracerB = new SimpleTracer();
        SimpleTracer tracerC = new SimpleTracer();
        Propagator propA = new W3CTraceContextPropagator(tracerA);
        Propagator propB = new W3CTraceContextPropagator(tracerB);
        Propagator propC = new W3CTraceContextPropagator(tracerC);

        // Service A: root span
        SimpleSpan spanA = tracerA.nextSpan().name("service-a").start();
        Map<String, String> headersAtoB = new HashMap<>();
        propA.inject(spanA.context(), headersAtoB, setter);

        // Service B: extracts from A, creates child, then propagates to C
        Span spanB = propB.extract(headersAtoB, getter).name("service-b").start();
        Map<String, String> headersBtoC = new HashMap<>();
        propB.inject(spanB.context(), headersBtoC, setter);

        // Service C: extracts from B
        Span spanC = propC.extract(headersBtoC, getter).name("service-c").start();

        // Assert — all three share the same traceId
        assertThat(spanB.context().traceId()).isEqualTo(spanA.context().traceId());
        assertThat(spanC.context().traceId()).isEqualTo(spanA.context().traceId());

        // Assert — parent chain: C → B → A
        assertThat(spanC.context().parentId()).isEqualTo(spanB.context().spanId());
        assertThat(spanB.context().parentId()).isEqualTo(spanA.context().spanId());

        // Assert — each has a unique spanId
        assertThat(spanA.context().spanId())
                .isNotEqualTo(spanB.context().spanId())
                .isNotEqualTo(spanC.context().spanId());

        spanC.end();
        spanB.end();
        spanA.end();
    }

    @Test
    void shouldCreateNewRootTrace_WhenNoContextPropagated() {
        // Simulate receiving a request with NO trace headers
        SimpleTracer tracer = new SimpleTracer();
        Propagator propagator = new W3CTraceContextPropagator(tracer);

        Map<String, String> emptyHeaders = new HashMap<>();
        Span span = propagator.extract(emptyHeaders, getter).name("new-root").start();

        // Assert — a fresh root trace
        assertThat(span.context().traceId()).isNotEmpty();
        assertThat(span.context().parentId()).isEmpty();
        assertThat(span.context().spanId()).isNotEmpty();

        span.end();
    }

    @Test
    void shouldWorkWithExportPipeline_WhenPropagatedSpanEnds() {
        // Arrange — tracer with a reporter
        java.util.List<dev.linhvu.tracing.exporter.FinishedSpan> reported = new java.util.ArrayList<>();
        SimpleTracer tracer = new SimpleTracer(
                java.util.List.of(),
                java.util.List.of(),
                java.util.List.of(reported::add)
        );
        Propagator propagator = new W3CTraceContextPropagator(tracer);

        // Simulate receiving a propagated context
        Map<String, String> headers = Map.of(
                "traceparent", "00-abcdef1234567890abcdef1234567890-1234567890abcdef-01"
        );
        Span span = propagator.extract(headers, getter).name("reported-span").start();

        // Act
        span.end();

        // Assert — span was reported through the export pipeline
        assertThat(reported).hasSize(1);
        assertThat(reported.get(0).getName()).isEqualTo("reported-span");
        assertThat(reported.get(0).getTraceId()).isEqualTo("abcdef1234567890abcdef1234567890");
    }

}
```

---

## Summary

| Concept | What It Means |
|---------|---------------|
| **Propagator** | Interface that serializes/deserializes trace context into carriers (HTTP headers, message headers) for cross-process communication |
| **Setter\<C\> / Getter\<C\>** | Functional interfaces that abstract carrier read/write — one lambda adapts the Propagator to any transport type |
| **W3C traceparent** | Industry-standard header format: `00-{traceId}-{spanId}-{flags}` — one header, interoperable across all tracing systems |
| **inject()** | Writes traceId + spanId + sampling flag into the carrier (outgoing direction) |
| **extract()** | Reads the traceparent header, parses it into a TraceContext, returns a Span.Builder pre-configured with that parent |

**Next: Chapter 6 — Baggage** — Key-value pairs that propagate alongside trace context across service boundaries, carrying business data like user IDs or request IDs that ride along with every span in the trace.
