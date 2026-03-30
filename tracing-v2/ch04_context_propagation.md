# Chapter 4: Context Propagation — Inject and Extract Trace Context

> **What you'll build**: The `Propagator` interface with its `Setter` and `Getter` inner interfaces
> — the mechanism that makes tracing *distributed* by injecting trace context into outgoing requests
> and extracting it from incoming requests.

---

## 1. The API Contract

After this chapter, a client can write:

```java
Propagator propagator = ...;

// SENDER: inject trace context into outgoing HTTP headers
Map<String, String> headers = new HashMap<>();
propagator.inject(span.context(), headers, Map::put);

// RECEIVER: extract trace context from incoming HTTP headers
Span.Builder builder = propagator.extract(headers, Map::get);
Span serverSpan = builder.kind(Span.Kind.SERVER).name("handle-request").start();
```

**Without propagation, tracing is local**: Chapters 1–3 let you create spans, tag them, and
attach baggage — but only within a single JVM. The `Propagator` is the bridge that connects
traces across service boundaries by serializing `TraceContext` into a carrier (HTTP headers,
gRPC metadata, message headers) on the sender side and deserializing it on the receiver side.

### The Three Types

```
Propagator                              ← inject() and extract() trace context
├── Setter<C>                           ← writes fields onto carrier (e.g., Map::put)
└── Getter<C>                           ← reads fields from carrier (e.g., Map::get)
```

**Why generics?** The carrier type `C` is generic because propagation isn't tied to HTTP.
The same `Propagator` works with a `Map<String, String>` (HTTP headers), a `Metadata` object
(gRPC), a `Headers` object (Kafka), or any custom carrier — you just supply the matching
`Setter` and `Getter`.

### The Asymmetry: inject takes TraceContext, extract returns Span.Builder

This is a deliberate design choice:

```
SENDER SIDE:
  I have a TraceContext → inject it into the carrier
  inject(context, carrier, setter)  → void

RECEIVER SIDE:
  I have a carrier → extract and create a new child span
  extract(carrier, getter)  → Span.Builder
```

The sender already has a `TraceContext` (from its current span), so `inject()` simply serializes
it. The receiver needs to create a *new* span parented to the extracted context — returning a
`Span.Builder` lets the caller configure the span (name, kind, tags) in one fluent chain:

```java
propagator.extract(headers, Map::get)
    .kind(Span.Kind.SERVER)
    .name("handle-request")
    .start();
```

If `extract()` returned a raw `TraceContext` instead, the caller would need to wire it into
a builder manually via `spanBuilder().setParent(ctx)` — an extra step that's easy to forget.

---

## 2. Client Usage & Tests

### Test: NOOP cascade for propagation

Just like every other API type, Propagator has a full NOOP cascade:

```java
@Test
void noopPropagator_shouldReturnEmptyFields() {
    Propagator propagator = Propagator.NOOP;
    assertThat(propagator.fields()).isEmpty();
}

@Test
void noopPropagator_injectShouldNotWriteAnything() {
    Propagator propagator = Propagator.NOOP;
    Map<String, String> headers = new HashMap<>();

    TraceContext context = buildContext("abc123", "def456", null, true);
    propagator.inject(context, headers, Map::put);

    assertThat(headers).isEmpty();
}

@Test
void noopPropagator_extractShouldReturnNoopBuilder() {
    Propagator propagator = Propagator.NOOP;
    Map<String, String> headers = Map.of("X-B3-TraceId", "abc123");

    Span.Builder builder = propagator.extract(headers, Map::get);
    assertThat(builder).isSameAs(Span.Builder.NOOP);

    Span span = builder.name("test").start();
    assertThat(span.isNoop()).isTrue();
}
```

**The NOOP chain**: `Propagator.NOOP.extract()` returns `Span.Builder.NOOP`, which produces
`Span.NOOP` with `TraceContext.NOOP`. The entire chain is safe to call without null checks.

### Test: NOOP Setter and Getter

The inner interfaces also have NOOP constants:

```java
@Test
void noopSetter_shouldNotThrow() {
    @SuppressWarnings("unchecked")
    Propagator.Setter<Map<String, String>> setter = Propagator.Setter.NOOP;
    Map<String, String> carrier = new HashMap<>();

    setter.set(carrier, "key", "value");
    assertThat(carrier).isEmpty();
}

@Test
void noopGetter_shouldReturnNull() {
    @SuppressWarnings("unchecked")
    Propagator.Getter<Map<String, String>> getter = Propagator.Getter.NOOP;
    Map<String, String> carrier = Map.of("key", "value");

    assertThat(getter.get(carrier, "key")).isNull();
}
```

### Test: Inject trace context into headers

A concrete test using a B3-style propagator (defined in the test class):

```java
@Test
void shouldInjectTraceContext_intoHeaders() {
    Propagator propagator = new B3Propagator();
    Map<String, String> headers = new HashMap<>();
    TraceContext context = buildContext("abc123", "def456", "parent789", true);

    propagator.inject(context, headers, Map::put);

    assertThat(headers).containsEntry("X-B3-TraceId", "abc123");
    assertThat(headers).containsEntry("X-B3-SpanId", "def456");
    assertThat(headers).containsEntry("X-B3-ParentSpanId", "parent789");
    assertThat(headers).containsEntry("X-B3-Sampled", "1");
}
```

**How `Map::put` works as a Setter**: `Setter<C>` has a single method `set(C carrier, String key, String value)`.
`Map::put` has the signature `put(String key, String value)` — when used as a method reference on a
`Map<String, String>`, Java resolves it as `(map, key, value) -> map.put(key, value)`, which
matches the `Setter<Map<String, String>>` functional interface.

### Test: Inject handles null parentId and sampled

Inject should skip fields that don't apply:

```java
@Test
void shouldInjectTraceContext_withNullParentId() {
    Propagator propagator = new B3Propagator();
    Map<String, String> headers = new HashMap<>();
    TraceContext context = buildContext("abc123", "def456", null, true);

    propagator.inject(context, headers, Map::put);

    assertThat(headers).containsEntry("X-B3-TraceId", "abc123");
    assertThat(headers).containsEntry("X-B3-SpanId", "def456");
    assertThat(headers).doesNotContainKey("X-B3-ParentSpanId");
}

@Test
void shouldInjectTraceContext_withSampledNull() {
    Propagator propagator = new B3Propagator();
    Map<String, String> headers = new HashMap<>();
    TraceContext context = buildContext("abc123", "def456", null, null);

    propagator.inject(context, headers, Map::put);

    assertThat(headers).doesNotContainKey("X-B3-Sampled");
}
```

### Test: Extract trace context from headers

```java
@Test
void shouldExtractTraceContext_fromHeaders() {
    Propagator propagator = new B3Propagator();
    Map<String, String> headers = Map.of(
            "X-B3-TraceId", "abc123",
            "X-B3-SpanId", "def456",
            "X-B3-ParentSpanId", "parent789",
            "X-B3-Sampled", "1");

    Span.Builder builder = propagator.extract(headers, Map::get);
    Span span = builder.kind(Span.Kind.SERVER).name("handle-request").start();

    // The extracted context should have been set as parent
    assertThat(span.context().traceId()).isEqualTo("abc123");
    assertThat(span.context().parentId()).isEqualTo("def456");
    assertThat(span.isNoop()).isFalse();
}
```

**Why parentId == "def456"**: The extracted `spanId` from the header becomes the *parent* of
the new span. The sender's span ID is the receiver's parent span ID — this is how parent-child
relationships cross service boundaries.

### Test: Extract returns NOOP when no context present

```java
@Test
void shouldReturnNoopBuilder_whenNoTraceIdInHeaders() {
    Propagator propagator = new B3Propagator();
    Map<String, String> headers = Map.of();

    Span.Builder builder = propagator.extract(headers, Map::get);
    assertThat(builder).isSameAs(Span.Builder.NOOP);

    Span span = builder.name("no-context").start();
    assertThat(span.isNoop()).isTrue();
}
```

**Graceful degradation**: When no trace headers are present, `extract()` returns
`Span.Builder.NOOP` rather than throwing. The receiver can still start a span — it will just
be a no-op. This means instrumentation code doesn't need to check whether propagation headers
exist.

### Test: Full round-trip — inject then extract

```java
@Test
void shouldRoundTrip_injectThenExtract() {
    Propagator propagator = new B3Propagator();

    // Sender side: inject context
    TraceContext originalContext = buildContext("trace-abc", "span-def", "parent-ghi", true);
    Map<String, String> headers = new HashMap<>();
    propagator.inject(originalContext, headers, Map::put);

    // Receiver side: extract context
    Span.Builder builder = propagator.extract(headers, Map::get);
    Span serverSpan = builder.kind(Span.Kind.SERVER).name("handle").start();

    // The extracted span should be a child of the injected context
    assertThat(serverSpan.context().traceId()).isEqualTo("trace-abc");
    assertThat(serverSpan.context().parentId()).isEqualTo("span-def");
}
```

**This is the heart of distributed tracing**: Service A creates a span, injects its context
into HTTP headers, and sends the request. Service B extracts the context from the headers and
creates a new span that is a child of Service A's span — sharing the same trace ID.

### Test: Propagation with custom carrier type

The carrier doesn't have to be a Map — any type works with the right Setter:

```java
@Test
void shouldWorkWithCustomCarrierType() {
    Propagator propagator = new B3Propagator();

    String[] carrier = new String[4];
    TraceContext context = buildContext("trace-x", "span-y", null, true);

    Propagator.Setter<String[]> arraySetter = (c, key, value) -> {
        switch (key) {
            case "X-B3-TraceId" -> c[0] = value;
            case "X-B3-SpanId" -> c[1] = value;
            case "X-B3-ParentSpanId" -> c[2] = value;
            case "X-B3-Sampled" -> c[3] = value;
        }
    };

    propagator.inject(context, carrier, arraySetter);
    assertThat(carrier[0]).isEqualTo("trace-x");
    assertThat(carrier[1]).isEqualTo("span-y");
    assertThat(carrier[3]).isEqualTo("1");
}
```

### Test: Fields describe propagation header names

```java
@Test
void shouldReturnPropagationFields() {
    Propagator propagator = new B3Propagator();

    List<String> fields = propagator.fields();
    assertThat(fields).containsExactlyInAnyOrder(
            "X-B3-TraceId", "X-B3-SpanId", "X-B3-ParentSpanId", "X-B3-Sampled");
}
```

**Why `fields()` matters**: Frameworks like Spring use `fields()` to know which HTTP headers
to allowlist in CORS, which to forward in proxies, and which to include in logging MDC. Without
it, frameworks would need to hardcode knowledge of specific propagation formats.

---

## 3. Implementing the Call Chain — Top-Down

### Layer 1: Propagator Interface (the API surface)

The Propagator is a single interface with three methods and two inner interfaces. It is
*pure API* — no implementation logic, no internal dependencies:

```java
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
```

**Design decisions**:
- **No default methods on the main interface** — every propagator must implement all three
  methods. There's no sensible default for `inject()` or `extract()`.
- **NOOP returns `Span.Builder.NOOP`** from `extract()` — this cascades to `Span.NOOP` and
  `TraceContext.NOOP`, maintaining the project's NOOP chain pattern.
- **`inject()` returns void** — the method writes *onto* the carrier via the Setter. There's
  nothing to return.
- **`extract()` returns `Span.Builder`** — as discussed, this is the asymmetric design choice.

### Layer 2: Setter\<C\> Inner Interface

```java
    interface Setter<C> {

        @SuppressWarnings("rawtypes")
        Setter NOOP = (carrier, key, value) -> {
        };

        void set(C carrier, String key, String value);
    }
```

**Why `@SuppressWarnings("rawtypes")`**: The NOOP constant is declared as raw `Setter` (not
`Setter<SomeType>`) because it needs to be assignable to `Setter<Map>`, `Setter<String[]>`,
`Setter<HttpHeaders>`, etc. Using raw types here is the pragmatic choice from the real
framework — it avoids needing a generic helper method just for the NOOP.

### Layer 3: Getter\<C\> Inner Interface

```java
    interface Getter<C> {

        @SuppressWarnings("rawtypes")
        Getter NOOP = (carrier, key) -> null;

        String get(C carrier, String key);
    }
```

**Simplified from real**: The real `Getter` also has a `getAll()` default method for
multi-valued headers (e.g., multiple `baggage` headers). We omit this since our simplified
version doesn't need multi-value support.

---

## 4. Try It Yourself

1. **Write a W3C Trace Context propagator** — instead of B3 headers (`X-B3-TraceId` etc.),
   implement the W3C `traceparent` format: `00-{traceId}-{spanId}-{flags}`. The single header
   carries all four fields.

2. **Write a composite propagator** — a `CompositePropagator` that takes multiple propagators
   and tries each one in order during `extract()`, returning the first non-NOOP result. For
   `inject()`, have it inject using all propagators. This is useful for supporting both B3
   and W3C formats simultaneously.

3. **Use the Propagator with the TestTracer from Chapter 1** — create a span with the
   `TestTracer`, inject its context into headers, then extract on the "receiver" side using
   `TestTracer.spanBuilder().setParent(extractedContext)` to create a proper child span.

---

## 5. Why This Works

### Strategy Pattern via Generics
The `Setter<C>` and `Getter<C>` interfaces are a textbook Strategy pattern — the Propagator
delegates the carrier-specific read/write to pluggable strategies. This keeps the Propagator
completely decoupled from any transport API. `Map::put` for HTTP, `metadata::put` for gRPC,
`headers::add` for Kafka — same Propagator, different strategies.

### Carrier Nullability
The carrier parameter in `inject()` and `Setter.set()` may be null. This sounds odd but
enables a useful pattern: Setter implementations that capture the carrier via closure
(lambda-based setters) don't need the carrier parameter at all. The real framework
allows this for flexibility.

### fields() Enables Framework Integration
The `fields()` method seems minor but is critical for framework integration. Spring Cloud
Sleuth (now Micrometer Tracing) uses it to automatically:
- Add propagation headers to CORS allowed-headers
- Forward them in gateway/proxy configurations
- Include them in MDC (Mapped Diagnostic Context) for logging

Without `fields()`, frameworks would need to hardcode header names per propagation format.

---

## 6. What We Enhanced

| Component | State After Ch04 | Simplifications |
|-----------|-------------------|-----------------|
| `Propagator` | 3 methods (`fields`, `inject`, `extract`) + NOOP | No composite propagator support |
| `Propagator.Setter<C>` | 1 method (`set`) + NOOP | — |
| `Propagator.Getter<C>` | 1 method (`get`) + NOOP | No `getAll()` default method for multi-valued headers |

**What's NOT connected yet**: The Propagator interface is defined but not yet used by any
bridge implementation. When we build the Brave bridge (Feature 8), `BravePropagator` will
delegate to Brave's `Propagation` API. The SimpleTracer (Feature 6) can use propagation
for testing inject/extract flows.

---

## 7. Connection to Real Framework

| Simplified Class | Real Framework Class | Source File |
|------------------|----------------------|-------------|
| `io.simpletracing.propagation.Propagator` | `io.micrometer.tracing.propagation.Propagator` | `micrometer-tracing/src/main/java/io/micrometer/tracing/propagation/Propagator.java` |
| `io.simpletracing.propagation.Propagator.Setter` | `io.micrometer.tracing.propagation.Propagator.Setter` | (same file) |
| `io.simpletracing.propagation.Propagator.Getter` | `io.micrometer.tracing.propagation.Propagator.Getter` | (same file) |

**Real framework extras we skipped**:
- `Getter.getAll()` — a default method that returns `Iterable<String>` for multi-valued
  headers (e.g., the `baggage` header can appear multiple times). Delegates to `get()` by
  default, wrapping the result in a singleton list.
- The real `Propagator` is used by `BravePropagator` and `OtelPropagator` bridge
  implementations. Our simplified version will only be implemented by the Brave bridge.

---

## 8. Complete Code

### `simple-tracing-api/src/main/java/io/simpletracing/propagation/Propagator.java` [NEW]

```java
package io.simpletracing.propagation;

import io.simpletracing.Span;
import io.simpletracing.TraceContext;

import java.util.Collections;
import java.util.List;

/**
 * Injects and extracts trace context across process boundaries (e.g., HTTP headers).
 * This is what makes tracing <b>distributed</b> — without propagation, each service
 * creates an isolated trace.
 *
 * <p>A Propagator uses a generic carrier type {@code C} (typically {@code Map<String, String>}
 * for HTTP) with pluggable {@link Setter} and {@link Getter} strategies, so the same
 * Propagator works with any transport (HTTP, gRPC, messaging, etc.).
 *
 * <p>Usage:
 * <pre>{@code
 * Propagator propagator = ...;
 *
 * // SENDER: inject trace context into outgoing HTTP headers
 * Map<String, String> headers = new HashMap<>();
 * propagator.inject(span.context(), headers, Map::put);
 *
 * // RECEIVER: extract trace context from incoming HTTP headers
 * Span.Builder builder = propagator.extract(headers, Map::get);
 * Span serverSpan = builder.kind(Span.Kind.SERVER).name("handle-request").start();
 * }</pre>
 *
 * <p>Maps to: {@code io.micrometer.tracing.propagation.Propagator}
 *
 * @see Setter
 * @see Getter
 */
public interface Propagator {

    /**
     * A no-op propagator that does not inject or extract anything.
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
     * Returns the propagation field names (header keys) that this propagator will
     * read from or write to. Useful for frameworks that need to allowlist these
     * headers (e.g., CORS, logging MDC).
     *
     * @return list of field names (e.g., {@code ["X-B3-TraceId", "X-B3-SpanId"]})
     */
    List<String> fields();

    /**
     * Injects the trace context identity into a carrier (e.g., HTTP headers).
     * The {@link Setter} writes each field onto the carrier.
     *
     * @param context the trace context to propagate
     * @param carrier the outgoing carrier (e.g., HTTP headers map), may be null
     * @param setter  strategy for writing fields onto the carrier
     * @param <C>     carrier type
     */
    <C> void inject(TraceContext context, C carrier, Setter<C> setter);

    /**
     * Extracts a trace context from an incoming carrier (e.g., HTTP request headers).
     * Returns a {@link Span.Builder} with the parent context set, ready for the
     * receiver to configure (name, kind) and start.
     *
     * <p><b>Why Span.Builder, not TraceContext?</b> The receiver needs to create a new
     * child span parented to the extracted context. Returning a builder lets the caller
     * set the span name, kind, and other properties in one fluent chain.
     *
     * @param carrier the incoming carrier (e.g., HTTP headers map), may be null
     * @param getter  strategy for reading fields from the carrier
     * @param <C>     carrier type
     * @return a span builder with the extracted parent context set, or
     *         {@link Span.Builder#NOOP} if no context was found
     */
    <C> Span.Builder extract(C carrier, Getter<C> getter);

    /**
     * Writes propagation fields onto a carrier. Implementations are typically
     * one-liners like {@code Map::put} or {@code HttpHeaders::set}.
     *
     * <p>Maps to: {@code io.micrometer.tracing.propagation.Propagator.Setter}
     *
     * @param <C> carrier type
     */
    interface Setter<C> {

        /**
         * A no-op setter that ignores all writes.
         */
        @SuppressWarnings("rawtypes")
        Setter NOOP = (carrier, key, value) -> {
        };

        /**
         * Sets a propagation field on the carrier.
         *
         * @param carrier the outgoing carrier, may be null
         * @param key     the field name (e.g., "X-B3-TraceId")
         * @param value   the field value (e.g., the trace ID)
         */
        void set(C carrier, String key, String value);

    }

    /**
     * Reads propagation fields from a carrier. Implementations are typically
     * one-liners like {@code Map::get} or {@code HttpHeaders::getFirst}.
     *
     * <p>Maps to: {@code io.micrometer.tracing.propagation.Propagator.Getter}
     *
     * @param <C> carrier type
     */
    interface Getter<C> {

        /**
         * A no-op getter that always returns null.
         */
        @SuppressWarnings("rawtypes")
        Getter NOOP = (carrier, key) -> null;

        /**
         * Reads a propagation field from the carrier.
         *
         * @param carrier the incoming carrier
         * @param key     the field name to read
         * @return the field value, or null if not present
         */
        String get(C carrier, String key);

    }

}
```

### `simple-tracing-api/src/test/java/io/simpletracing/propagation/PropagatorApiTest.java` [NEW]

```java
package io.simpletracing.propagation;

import io.simpletracing.Span;
import io.simpletracing.TraceContext;

import org.junit.jupiter.api.Test;

import java.util.HashMap;
import java.util.List;
import java.util.Map;

import static org.assertj.core.api.Assertions.assertThat;

/**
 * Client-perspective tests for Feature 4: Context Propagation.
 *
 * <p>These tests verify that the Propagator API works from a client's point of view:
 * injecting trace context into outgoing carriers and extracting it from incoming carriers.
 * A minimal B3-style propagator is used to verify the inject/extract round-trip.
 */
class PropagatorApiTest {

    // ── NOOP Cascade Tests ────────────────────────────────────────────────

    @Test
    void noopPropagator_shouldReturnEmptyFields() {
        Propagator propagator = Propagator.NOOP;
        assertThat(propagator.fields()).isEmpty();
    }

    @Test
    void noopPropagator_injectShouldNotWriteAnything() {
        Propagator propagator = Propagator.NOOP;
        Map<String, String> headers = new HashMap<>();

        TraceContext context = buildContext("abc123", "def456", null, true);
        propagator.inject(context, headers, Map::put);

        assertThat(headers).isEmpty();
    }

    @Test
    void noopPropagator_extractShouldReturnNoopBuilder() {
        Propagator propagator = Propagator.NOOP;
        Map<String, String> headers = Map.of("X-B3-TraceId", "abc123");

        Span.Builder builder = propagator.extract(headers, Map::get);
        assertThat(builder).isSameAs(Span.Builder.NOOP);

        Span span = builder.name("test").start();
        assertThat(span.isNoop()).isTrue();
    }

    @Test
    void noopSetter_shouldNotThrow() {
        @SuppressWarnings("unchecked")
        Propagator.Setter<Map<String, String>> setter = Propagator.Setter.NOOP;
        Map<String, String> carrier = new HashMap<>();

        setter.set(carrier, "key", "value");
        assertThat(carrier).isEmpty();
    }

    @Test
    void noopGetter_shouldReturnNull() {
        @SuppressWarnings("unchecked")
        Propagator.Getter<Map<String, String>> getter = Propagator.Getter.NOOP;
        Map<String, String> carrier = Map.of("key", "value");

        assertThat(getter.get(carrier, "key")).isNull();
    }

    // ── Inject Tests ─────────────────────────────────────────────────────

    @Test
    void shouldInjectTraceContext_intoHeaders() {
        Propagator propagator = new B3Propagator();
        Map<String, String> headers = new HashMap<>();
        TraceContext context = buildContext("abc123", "def456", "parent789", true);

        propagator.inject(context, headers, Map::put);

        assertThat(headers).containsEntry("X-B3-TraceId", "abc123");
        assertThat(headers).containsEntry("X-B3-SpanId", "def456");
        assertThat(headers).containsEntry("X-B3-ParentSpanId", "parent789");
        assertThat(headers).containsEntry("X-B3-Sampled", "1");
    }

    @Test
    void shouldInjectTraceContext_withNullParentId() {
        Propagator propagator = new B3Propagator();
        Map<String, String> headers = new HashMap<>();
        TraceContext context = buildContext("abc123", "def456", null, true);

        propagator.inject(context, headers, Map::put);

        assertThat(headers).containsEntry("X-B3-TraceId", "abc123");
        assertThat(headers).containsEntry("X-B3-SpanId", "def456");
        assertThat(headers).doesNotContainKey("X-B3-ParentSpanId");
        assertThat(headers).containsEntry("X-B3-Sampled", "1");
    }

    @Test
    void shouldInjectTraceContext_withSampledFalse() {
        Propagator propagator = new B3Propagator();
        Map<String, String> headers = new HashMap<>();
        TraceContext context = buildContext("abc123", "def456", null, false);

        propagator.inject(context, headers, Map::put);

        assertThat(headers).containsEntry("X-B3-Sampled", "0");
    }

    @Test
    void shouldInjectTraceContext_withSampledNull() {
        Propagator propagator = new B3Propagator();
        Map<String, String> headers = new HashMap<>();
        TraceContext context = buildContext("abc123", "def456", null, null);

        propagator.inject(context, headers, Map::put);

        assertThat(headers).doesNotContainKey("X-B3-Sampled");
    }

    // ── Extract Tests ────────────────────────────────────────────────────

    @Test
    void shouldExtractTraceContext_fromHeaders() {
        Propagator propagator = new B3Propagator();
        Map<String, String> headers = Map.of(
                "X-B3-TraceId", "abc123",
                "X-B3-SpanId", "def456",
                "X-B3-ParentSpanId", "parent789",
                "X-B3-Sampled", "1");

        Span.Builder builder = propagator.extract(headers, Map::get);
        Span span = builder.kind(Span.Kind.SERVER).name("handle-request").start();

        // The extracted context should have been set as parent
        assertThat(span.context().traceId()).isEqualTo("abc123");
        assertThat(span.context().parentId()).isEqualTo("def456");
        assertThat(span.isNoop()).isFalse();
    }

    @Test
    void shouldExtractTraceContext_withNoParentId() {
        Propagator propagator = new B3Propagator();
        Map<String, String> headers = Map.of(
                "X-B3-TraceId", "abc123",
                "X-B3-SpanId", "def456",
                "X-B3-Sampled", "1");

        Span.Builder builder = propagator.extract(headers, Map::get);
        Span span = builder.name("handle-request").start();

        assertThat(span.context().traceId()).isEqualTo("abc123");
        assertThat(span.context().parentId()).isEqualTo("def456");
    }

    @Test
    void shouldReturnNoopBuilder_whenNoTraceIdInHeaders() {
        Propagator propagator = new B3Propagator();
        Map<String, String> headers = Map.of();

        Span.Builder builder = propagator.extract(headers, Map::get);
        assertThat(builder).isSameAs(Span.Builder.NOOP);

        Span span = builder.name("no-context").start();
        assertThat(span.isNoop()).isTrue();
    }

    // ── Round-Trip Tests ─────────────────────────────────────────────────

    @Test
    void shouldRoundTrip_injectThenExtract() {
        Propagator propagator = new B3Propagator();

        // Sender side: inject context
        TraceContext originalContext = buildContext("trace-abc", "span-def", "parent-ghi", true);
        Map<String, String> headers = new HashMap<>();
        propagator.inject(originalContext, headers, Map::put);

        // Receiver side: extract context
        Span.Builder builder = propagator.extract(headers, Map::get);
        Span serverSpan = builder.kind(Span.Kind.SERVER).name("handle").start();

        // The extracted span should be a child of the injected context
        assertThat(serverSpan.context().traceId()).isEqualTo("trace-abc");
        assertThat(serverSpan.context().parentId()).isEqualTo("span-def");
    }

    @Test
    void shouldRoundTrip_preservingSampledFalse() {
        Propagator propagator = new B3Propagator();

        TraceContext context = buildContext("trace-1", "span-2", null, false);
        Map<String, String> headers = new HashMap<>();
        propagator.inject(context, headers, Map::put);

        assertThat(headers.get("X-B3-Sampled")).isEqualTo("0");
    }

    // ── Fields Tests ─────────────────────────────────────────────────────

    @Test
    void shouldReturnPropagationFields() {
        Propagator propagator = new B3Propagator();

        List<String> fields = propagator.fields();
        assertThat(fields).containsExactlyInAnyOrder(
                "X-B3-TraceId", "X-B3-SpanId", "X-B3-ParentSpanId", "X-B3-Sampled");
    }

    // ── Custom Carrier Type Tests ────────────────────────────────────────

    @Test
    void shouldWorkWithCustomCarrierType() {
        Propagator propagator = new B3Propagator();

        // Use a String array as carrier (index 0=traceId, 1=spanId, 2=sampled)
        String[] carrier = new String[4];
        TraceContext context = buildContext("trace-x", "span-y", null, true);

        Propagator.Setter<String[]> arraySetter = (c, key, value) -> {
            switch (key) {
                case "X-B3-TraceId" -> c[0] = value;
                case "X-B3-SpanId" -> c[1] = value;
                case "X-B3-ParentSpanId" -> c[2] = value;
                case "X-B3-Sampled" -> c[3] = value;
            }
        };

        propagator.inject(context, carrier, arraySetter);
        assertThat(carrier[0]).isEqualTo("trace-x");
        assertThat(carrier[1]).isEqualTo("span-y");
        assertThat(carrier[3]).isEqualTo("1");
    }

    // ── Helpers ──────────────────────────────────────────────────────────

    /**
     * Builds a TraceContext with the given fields using a simple record implementation.
     */
    private static TraceContext buildContext(String traceId, String spanId, String parentId,
            Boolean sampled) {
        return new TraceContext() {
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
        };
    }

    // ── Minimal B3 Propagator for Testing ────────────────────────────────

    /**
     * A minimal B3-style propagator that demonstrates the inject/extract pattern.
     * This is NOT a production implementation — it's just enough to prove the
     * Propagator API contract works.
     */
    private static class B3Propagator implements Propagator {

        private static final String TRACE_ID = "X-B3-TraceId";

        private static final String SPAN_ID = "X-B3-SpanId";

        private static final String PARENT_SPAN_ID = "X-B3-ParentSpanId";

        private static final String SAMPLED = "X-B3-Sampled";

        @Override
        public List<String> fields() {
            return List.of(TRACE_ID, SPAN_ID, PARENT_SPAN_ID, SAMPLED);
        }

        @Override
        public <C> void inject(TraceContext context, C carrier, Setter<C> setter) {
            setter.set(carrier, TRACE_ID, context.traceId());
            setter.set(carrier, SPAN_ID, context.spanId());
            if (context.parentId() != null) {
                setter.set(carrier, PARENT_SPAN_ID, context.parentId());
            }
            if (context.sampled() != null) {
                setter.set(carrier, SAMPLED, context.sampled() ? "1" : "0");
            }
        }

        @Override
        public <C> Span.Builder extract(C carrier, Getter<C> getter) {
            String traceId = getter.get(carrier, TRACE_ID);
            if (traceId == null) {
                return Span.Builder.NOOP;
            }
            String spanId = getter.get(carrier, SPAN_ID);

            // Build the parent context from extracted fields
            TraceContext extractedContext = buildContext(
                    traceId,
                    spanId,
                    getter.get(carrier, PARENT_SPAN_ID),
                    parseSampled(getter.get(carrier, SAMPLED)));

            // Return a builder with the parent set — the caller will
            // add name, kind, etc. and call start()
            return new ExtractedSpanBuilder(extractedContext);
        }

        private static Boolean parseSampled(String value) {
            if (value == null) {
                return null;
            }
            return "1".equals(value) || "true".equalsIgnoreCase(value);
        }

        /**
         * Builds a TraceContext from extracted fields.
         */
        private static TraceContext buildContext(String traceId, String spanId, String parentId,
                Boolean sampled) {
            return new TraceContext() {
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
            };
        }

    }

    /**
     * A minimal Span.Builder implementation that records the extracted parent context
     * and produces a simple Span. Used only in tests to verify the extract() contract.
     */
    private static class ExtractedSpanBuilder implements Span.Builder {

        private final TraceContext parentContext;

        private String name;

        private Span.Kind kind;

        ExtractedSpanBuilder(TraceContext parentContext) {
            this.parentContext = parentContext;
        }

        @Override
        public Span.Builder setParent(TraceContext context) {
            return this;
        }

        @Override
        public Span.Builder setNoParent() {
            return this;
        }

        @Override
        public Span.Builder name(String name) {
            this.name = name;
            return this;
        }

        @Override
        public Span.Builder event(String value) {
            return this;
        }

        @Override
        public Span.Builder tag(String key, String value) {
            return this;
        }

        @Override
        public Span.Builder error(Throwable throwable) {
            return this;
        }

        @Override
        public Span.Builder kind(Span.Kind spanKind) {
            this.kind = spanKind;
            return this;
        }

        @Override
        public Span.Builder remoteServiceName(String remoteServiceName) {
            return this;
        }

        @Override
        public Span start() {
            // Create a child span: same traceId, parent's spanId becomes our parentId
            return new ExtractedSpan(parentContext.traceId(), parentContext.spanId(), name, kind);
        }

    }

    /**
     * A minimal Span implementation that represents a span extracted from propagation.
     */
    private static class ExtractedSpan implements Span {

        private final TraceContext context;

        ExtractedSpan(String traceId, String parentSpanId, String name, Span.Kind kind) {
            this.context = new TraceContext() {
                @Override
                public String traceId() {
                    return traceId;
                }

                @Override
                public String parentId() {
                    return parentSpanId;
                }

                @Override
                public String spanId() {
                    return java.util.UUID.randomUUID().toString().replace("-", "").substring(0, 16);
                }

                @Override
                public Boolean sampled() {
                    return true;
                }
            };
        }

        @Override
        public boolean isNoop() {
            return false;
        }

        @Override
        public TraceContext context() {
            return context;
        }

        @Override
        public Span start() {
            return this;
        }

        @Override
        public Span name(String name) {
            return this;
        }

        @Override
        public Span event(String value) {
            return this;
        }

        @Override
        public Span tag(String key, String value) {
            return this;
        }

        @Override
        public Span error(Throwable throwable) {
            return this;
        }

        @Override
        public void end() {
        }

        @Override
        public void abandon() {
        }

        @Override
        public Span remoteServiceName(String remoteServiceName) {
            return this;
        }

    }

}
```
