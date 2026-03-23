# Chapter 1: Span & TraceContext

> Line references based on commit `dceab1b9` of the Micrometer Tracing repository.

## Build Challenge

| Current State | Limitation | Objective |
|---------------|------------|-----------|
| No simplified tracing framework exists yet | Cannot represent a unit of work, carry trace IDs, or define the vocabulary that all tracing features share | Build the core data model: `Span`, `SpanCustomizer`, `TraceContext`, `ScopedSpan`, `Link`, and `Span.Kind` — all as interfaces with embedded NOOP implementations |

---

## 1.1 The Foundation: Span as the Core Abstraction

This is a foundation chapter — there is no prior system to integrate with. The `Span` interface is the core abstraction that **every future feature** will connect to:

- **Feature 2 (Tracer)** will create and manage `Span` instances
- **Feature 3 (SimpleTracer)** will provide a concrete in-memory `Span` implementation
- **Feature 4 (Export Pipeline)** will consume finished `Span` data
- **Feature 5 (Propagator)** will inject/extract `TraceContext` across process boundaries
- **Feature 8 (TracingObservationHandler)** will automatically create `Span`s from observations

The type hierarchy we're building:

```
SpanCustomizer (interface)          ← simplest customization: name, tag, event
  ^
  |  extends
  |
Span (interface)                   ← full lifecycle: start, end, error, context
  |-- inner enum: Span.Kind       ← SERVER, CLIENT, PRODUCER, CONSUMER
  |-- inner interface: Span.Builder ← fluent construction before starting

TraceContext (interface)            ← the "tracking number": traceId, spanId, parentId
  |-- inner interface: TraceContext.Builder

ScopedSpan (interface)              ← auto-scoped convenience (no manual start/scope)

Link (class)                        ← cross-trace references
```

Two key decisions here:

1. **Everything is an interface with a NOOP constant** — this lets library code call tracing methods safely even when no tracer is configured. The NOOP pattern avoids null checks everywhere.
2. **SpanCustomizer is separated from Span** — code that only needs to add tags/events to the current span doesn't need access to lifecycle methods (start/end) or trace context. This follows the Interface Segregation Principle.

## 1.2 SpanCustomizer — The Simplest Tracing Surface

**New file:** `src/main/java/dev/linhvu/tracing/SpanCustomizer.java`

```java
package dev.linhvu.tracing;

public interface SpanCustomizer {

    SpanCustomizer NOOP = new SpanCustomizer() {
        @Override
        public SpanCustomizer name(String name) {
            return this;
        }

        @Override
        public SpanCustomizer tag(String key, String value) {
            return this;
        }

        @Override
        public SpanCustomizer event(String value) {
            return this;
        }
    };

    SpanCustomizer name(String name);

    SpanCustomizer tag(String key, String value);

    default SpanCustomizer tag(String key, long value) {
        return tag(key, String.valueOf(value));
    }

    default SpanCustomizer tag(String key, double value) {
        return tag(key, String.valueOf(value));
    }

    default SpanCustomizer tag(String key, boolean value) {
        return tag(key, String.valueOf(value));
    }

    SpanCustomizer event(String value);

}
```

Three abstract methods (`name`, `tag`, `event`), three default convenience overloads that convert primitive values to strings. The NOOP returns `this` from every method — fluent chaining works even on the no-op.

## 1.3 TraceContext — The Tracking Number

**New file:** `src/main/java/dev/linhvu/tracing/TraceContext.java`

```java
package dev.linhvu.tracing;

public interface TraceContext {

    TraceContext NOOP = new TraceContext() {
        @Override
        public String traceId() { return ""; }

        @Override
        public String parentId() { return null; }

        @Override
        public String spanId() { return ""; }

        @Override
        public Boolean sampled() { return null; }
    };

    String traceId();

    String parentId();

    String spanId();

    Boolean sampled();

    interface Builder {

        Builder NOOP = new Builder() {
            @Override public Builder traceId(String traceId) { return this; }
            @Override public Builder parentId(String parentId) { return this; }
            @Override public Builder spanId(String spanId) { return this; }
            @Override public Builder sampled(Boolean sampled) { return this; }
            @Override public TraceContext build() { return TraceContext.NOOP; }
        };

        Builder traceId(String traceId);
        Builder parentId(String parentId);
        Builder spanId(String spanId);
        Builder sampled(Boolean sampled);
        TraceContext build();
    }

}
```

Four fields identify a span within a distributed trace:
- **traceId** — shared across all spans in one trace (the "order number")
- **spanId** — unique within the trace (the "leg ID")
- **parentId** — the span that created this one (null for root spans)
- **sampled** — true/false/null (deferred) sampling decision

## 1.4 Span — The Full Lifecycle

**New file:** `src/main/java/dev/linhvu/tracing/Span.java`

```java
package dev.linhvu.tracing;

import java.util.concurrent.TimeUnit;

public interface Span extends SpanCustomizer {

    Span NOOP = new Span() {
        @Override public boolean isNoop() { return true; }
        @Override public TraceContext context() { return TraceContext.NOOP; }
        @Override public Span start() { return this; }
        @Override public Span name(String name) { return this; }
        @Override public Span event(String value) { return this; }
        @Override public Span event(String value, long time, TimeUnit timeUnit) { return this; }
        @Override public Span tag(String key, String value) { return this; }
        @Override public Span error(Throwable throwable) { return this; }
        @Override public void end() { }
        @Override public void end(long time, TimeUnit timeUnit) { }
        @Override public void abandon() { }
        @Override public Span remoteServiceName(String remoteServiceName) { return this; }
        @Override public Span remoteIpAndPort(String ip, int port) { return this; }
    };

    boolean isNoop();
    TraceContext context();
    Span start();

    @Override Span name(String name);
    @Override Span tag(String key, String value);
    @Override Span event(String value);

    Span event(String value, long time, TimeUnit timeUnit);
    Span error(Throwable throwable);
    void end();
    void end(long time, TimeUnit timeUnit);
    void abandon();
    Span remoteServiceName(String remoteServiceName);
    Span remoteIpAndPort(String ip, int port);

    @Override default Span tag(String key, long value) { return tag(key, String.valueOf(value)); }
    @Override default Span tag(String key, double value) { return tag(key, String.valueOf(value)); }
    @Override default Span tag(String key, boolean value) { return tag(key, String.valueOf(value)); }

    enum Kind {
        SERVER, CLIENT, PRODUCER, CONSUMER
    }

    interface Builder {
        Builder NOOP = new Builder() {
            @Override public Builder setParent(TraceContext context) { return this; }
            @Override public Builder setNoParent() { return this; }
            @Override public Builder name(String name) { return this; }
            @Override public Builder event(String value) { return this; }
            @Override public Builder tag(String key, String value) { return this; }
            @Override public Builder error(Throwable throwable) { return this; }
            @Override public Builder kind(Kind spanKind) { return this; }
            @Override public Builder remoteServiceName(String remoteServiceName) { return this; }
            @Override public Builder remoteIpAndPort(String ip, int port) { return this; }
            @Override public Builder startTimestamp(long startTimestamp, TimeUnit unit) { return this; }
            @Override public Span start() { return Span.NOOP; }
        };

        Builder setParent(TraceContext context);
        Builder setNoParent();
        Builder name(String name);
        Builder event(String value);
        Builder tag(String key, String value);
        Builder error(Throwable throwable);
        Builder kind(Kind spanKind);
        Builder remoteServiceName(String remoteServiceName);
        Builder remoteIpAndPort(String ip, int port);
        Builder startTimestamp(long startTimestamp, TimeUnit unit);
        Span start();

        default Builder tag(String key, long value) { return tag(key, String.valueOf(value)); }
        default Builder tag(String key, double value) { return tag(key, String.valueOf(value)); }
        default Builder tag(String key, boolean value) { return tag(key, String.valueOf(value)); }
        default Builder addLink(Link link) { return this; }
    }

}
```

Key things `Span` adds over `SpanCustomizer`:
- **Lifecycle:** `start()` → do work → `end()` (or `abandon()` if aborted)
- **Context access:** `context()` returns the `TraceContext` with all IDs
- **Error recording:** `error(Throwable)` attaches an exception to the span
- **Remote metadata:** `remoteServiceName()` and `remoteIpAndPort()` for RPC spans
- **Covariant returns:** `name()`, `tag()`, `event()` return `Span` (not `SpanCustomizer`)

## 1.5 ScopedSpan & Link

**New file:** `src/main/java/dev/linhvu/tracing/ScopedSpan.java`

```java
package dev.linhvu.tracing;

public interface ScopedSpan {

    ScopedSpan NOOP = new ScopedSpan() {
        @Override public boolean isNoop() { return true; }
        @Override public TraceContext context() { return TraceContext.NOOP; }
        @Override public ScopedSpan name(String name) { return this; }
        @Override public ScopedSpan tag(String key, String value) { return this; }
        @Override public ScopedSpan event(String value) { return this; }
        @Override public ScopedSpan error(Throwable throwable) { return this; }
        @Override public void end() { }
    };

    boolean isNoop();
    TraceContext context();
    ScopedSpan name(String name);
    ScopedSpan tag(String key, String value);
    ScopedSpan event(String value);
    ScopedSpan error(Throwable throwable);
    void end();

}
```

`ScopedSpan` is intentionally simpler than `Span` — no `start()` (it's started on creation), no `abandon()`, no timed `end()`, no remote service methods. It's designed for the common case: "I want a span for this block of code."

**New file:** `src/main/java/dev/linhvu/tracing/Link.java`

```java
package dev.linhvu.tracing;

import java.util.Collections;
import java.util.Map;
import java.util.Objects;

public class Link {

    public static final Link NOOP = new Link(TraceContext.NOOP, Collections.emptyMap());

    private final TraceContext traceContext;
    private final Map<String, Object> tags;

    public Link(TraceContext traceContext, Map<String, Object> tags) {
        this.traceContext = traceContext;
        this.tags = tags;
    }

    public Link(TraceContext traceContext) {
        this(traceContext, Collections.emptyMap());
    }

    public Link(Span span, Map<String, Object> tags) {
        this(span.context(), tags);
    }

    public Link(Span span) {
        this(span.context(), Collections.emptyMap());
    }

    public TraceContext getTraceContext() { return traceContext; }
    public Map<String, Object> getTags() { return tags; }

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (o == null || getClass() != o.getClass()) return false;
        Link link = (Link) o;
        return Objects.equals(traceContext, link.traceContext) && Objects.equals(tags, link.tags);
    }

    @Override
    public int hashCode() { return Objects.hash(traceContext, tags); }

    @Override
    public String toString() {
        return "Link{traceContext=" + traceContext + ", tags=" + tags + "}";
    }

}
```

`Link` is the only **concrete class** in this feature — it's a simple value object referencing another span's trace context with optional tags. Four constructor overloads accept either `TraceContext` or `Span`, with or without tags.

## 1.6 Try It Yourself

<details>
<summary>Challenge: Implement a minimal SpanCustomizer that records all tags into a Map</summary>

Think about: How would you verify that `tag(String, long)` delegates correctly to `tag(String, String)`?

```java
public class MapSpanCustomizer implements SpanCustomizer {
    private final Map<String, String> tags = new LinkedHashMap<>();
    private String name;

    @Override
    public SpanCustomizer name(String name) {
        this.name = name;
        return this;
    }

    @Override
    public SpanCustomizer tag(String key, String value) {
        tags.put(key, value);
        return this;
    }

    @Override
    public SpanCustomizer event(String value) {
        return this; // events not tracked in this simple impl
    }

    public Map<String, String> getTags() { return tags; }
    public String getName() { return name; }
}
```

</details>

<details>
<summary>Challenge: Why doesn't ScopedSpan extend SpanCustomizer?</summary>

`ScopedSpan` has `name()`, `tag()`, and `event()` methods — the same as `SpanCustomizer`. But it doesn't extend it because the return types are different: `ScopedSpan`'s methods return `ScopedSpan`, not `SpanCustomizer`. Java doesn't allow a class to implement the same interface method with different return types unless one is a subtype. Since `ScopedSpan` is not a `SpanCustomizer`, the two type hierarchies remain independent — keeping `ScopedSpan` as the simpler, self-contained API it's designed to be.

</details>

## 1.7 Tests

### Unit Tests

**New file:** `src/test/java/dev/linhvu/tracing/SpanCustomizerTest.java`

```java
@Test
void shouldReturnSelf_WhenNoopNameCalled() {
    SpanCustomizer noop = SpanCustomizer.NOOP;
    assertThat(noop.name("test")).isSameAs(noop);
}

@Test
void shouldDelegateToStringTag_WhenLongTagCalled() {
    var recorder = new RecordingSpanCustomizer();
    recorder.tag("count", 42L);
    assertThat(recorder.lastTagValue).isEqualTo("42");
}
```

**New file:** `src/test/java/dev/linhvu/tracing/TraceContextTest.java`

```java
@Test
void shouldReturnEmptyTraceId_WhenNoopUsed() {
    assertThat(TraceContext.NOOP.traceId()).isEmpty();
}

@Test
void shouldReturnNoopContext_WhenNoopBuilderBuilds() {
    TraceContext ctx = TraceContext.Builder.NOOP.build();
    assertThat(ctx).isSameAs(TraceContext.NOOP);
}
```

**New file:** `src/test/java/dev/linhvu/tracing/SpanTest.java`

```java
@Test
void shouldBeNoop_WhenNoopSpanUsed() {
    assertThat(Span.NOOP.isNoop()).isTrue();
}

@Test
void shouldHaveFourKindValues() {
    assertThat(Span.Kind.values()).containsExactly(
        Span.Kind.SERVER, Span.Kind.CLIENT,
        Span.Kind.PRODUCER, Span.Kind.CONSUMER);
}

@Test
void shouldReturnNoopSpan_WhenNoopBuilderStarts() {
    assertThat(Span.Builder.NOOP.start()).isSameAs(Span.NOOP);
}
```

**New file:** `src/test/java/dev/linhvu/tracing/ScopedSpanTest.java` — verifies NOOP behavior and that the API surface is intentionally smaller than Span.

**New file:** `src/test/java/dev/linhvu/tracing/LinkTest.java` — verifies all four constructors, equals/hashCode, toString, and the NOOP constant.

**Run:** `./gradlew test` — expected: all tests pass ✅

---

## 1.8 Why This Works

> ★ **Insight** -------------------------------------------
> - **Why NOOP implementations inside the interface?** The Null Object Pattern eliminates null checks throughout the codebase. Any code can safely call `Span.NOOP.tag("k", "v")` without checking if tracing is enabled. This is critical for library instrumentation — libraries don't know if the application has configured a tracer.
> - **When to skip it:** If your type has only one implementation and is always present, a NOOP adds unnecessary complexity. NOOPs shine when the component is optional (like tracing in a library).
> - **Real-world parallel:** Every Micrometer interface follows this pattern — `MeterRegistry.NOOP`, `Timer.NOOP`, etc. It's the foundation that lets Spring Boot auto-configuration work: if no tracer is on the classpath, everything silently no-ops.
> -----------------------------------------------------------

> ★ **Insight** -------------------------------------------
> - **Why separate SpanCustomizer from Span?** Interface Segregation Principle — code that adds tags to the current span (like a web filter adding `http.method`) shouldn't have access to `start()`/`end()`. `SpanCustomizer` is the narrow view; `Span` is the full control surface.
> - **Trade-off:** Two types instead of one adds a concept to learn. But it prevents misuse — you can't accidentally `end()` a span you didn't create if you only have a `SpanCustomizer` reference.
> - **Real-world parallel:** In production, `SpanCustomizer` is what gets injected into `@ContinueSpan`-annotated methods (Feature 10). The method can add tags but can't mess with the span lifecycle.
> -----------------------------------------------------------

> ★ **Insight** -------------------------------------------
> - **Why is `sampled()` a `Boolean` (nullable) instead of `boolean`?** Three states: `true` (definitely sampled), `false` (definitely not sampled), `null` (deferred — let downstream services decide). This three-state design allows "lazy sampling" where the decision propagates through the system until someone commits to it.
> - **When this matters:** In high-throughput systems, you might want the first service to defer sampling so a downstream service with more context can make a better decision (e.g., always sample error paths).
> -----------------------------------------------------------

## 1.9 Connection to Real Framework

| Simplified Code | Real Framework Code | File:Line | Key Difference |
|----------------|--------------------|-----------| ---------------|
| `SpanCustomizer` | `SpanCustomizer` | `SpanCustomizer.java:26` | Real version is identical — we kept the full interface |
| `TraceContext` | `TraceContext` | `TraceContext.java:25` | Real uses `@Nullable` from jspecify; we use plain nulls |
| `TraceContext.Builder` | `TraceContext.Builder` | `TraceContext.java:72` | Identical structure |
| `Span` | `Span` | `Span.java:30` | Real has `tagOfStrings/Longs/Doubles/Booleans` list convenience methods; we omit them |
| `Span.Kind` | `Span.Kind` | `Span.java:148` | Identical |
| `Span.Builder` | `Span.Builder` | `Span.java:173` | Real has `tagOfStrings/Longs/Doubles/Booleans`; we omit them |
| `ScopedSpan` | `ScopedSpan` | `ScopedSpan.java:24` | Identical structure |
| `Link` | `Link` | `Link.java:27` | Identical — both are concrete value objects |

## 1.10 Complete Code

### Production Code

#### File: `src/main/java/dev/linhvu/tracing/SpanCustomizer.java` [NEW]

```java
package dev.linhvu.tracing;

/**
 * The simplest set of operations for customizing the current span in scope.
 * This is the base type that {@link Span} extends — it provides name, tag, and event
 * without exposing lifecycle methods (start/end) or trace context.
 */
public interface SpanCustomizer {

    /**
     * A no-op implementation that silently ignores all calls.
     * Used when tracing is disabled or no tracer is configured.
     */
    SpanCustomizer NOOP = new SpanCustomizer() {
        @Override
        public SpanCustomizer name(String name) {
            return this;
        }

        @Override
        public SpanCustomizer tag(String key, String value) {
            return this;
        }

        @Override
        public SpanCustomizer event(String value) {
            return this;
        }
    };

    /**
     * Sets the name of the span.
     * @param name span name
     * @return this for chaining
     */
    SpanCustomizer name(String name);

    /**
     * Adds a tag (key-value pair) to the span.
     * @param key tag key
     * @param value tag value
     * @return this for chaining
     */
    SpanCustomizer tag(String key, String value);

    /**
     * Adds a tag with a long value, converting to string.
     */
    default SpanCustomizer tag(String key, long value) {
        return tag(key, String.valueOf(value));
    }

    /**
     * Adds a tag with a double value, converting to string.
     */
    default SpanCustomizer tag(String key, double value) {
        return tag(key, String.valueOf(value));
    }

    /**
     * Adds a tag with a boolean value, converting to string.
     */
    default SpanCustomizer tag(String key, boolean value) {
        return tag(key, String.valueOf(value));
    }

    /**
     * Records an event (annotation) on the span.
     * @param value event description
     * @return this for chaining
     */
    SpanCustomizer event(String value);

}
```

#### File: `src/main/java/dev/linhvu/tracing/TraceContext.java` [NEW]

```java
package dev.linhvu.tracing;

/**
 * Carries the trace and span identification data that ties spans into a distributed trace.
 * Think of it as the "tracking number" on a package — it contains the traceId (which order),
 * spanId (which leg), parentId (previous leg), and sampling decision.
 */
public interface TraceContext {

    /**
     * A no-op trace context with empty IDs and no sampling decision.
     */
    TraceContext NOOP = new TraceContext() {
        @Override
        public String traceId() {
            return "";
        }

        @Override
        public String parentId() {
            return null;
        }

        @Override
        public String spanId() {
            return "";
        }

        @Override
        public Boolean sampled() {
            return null;
        }
    };

    /**
     * Returns the trace ID — shared across all spans in a single trace.
     */
    String traceId();

    /**
     * Returns the parent span ID, or null if this is the root span.
     */
    String parentId();

    /**
     * Returns this span's unique ID within the trace.
     */
    String spanId();

    /**
     * Returns the sampling decision: true (sampled), false (not sampled),
     * or null (deferred — let downstream decide).
     */
    Boolean sampled();

    /**
     * Builder for constructing {@link TraceContext} instances.
     */
    interface Builder {

        Builder NOOP = new Builder() {
            @Override
            public Builder traceId(String traceId) {
                return this;
            }

            @Override
            public Builder parentId(String parentId) {
                return this;
            }

            @Override
            public Builder spanId(String spanId) {
                return this;
            }

            @Override
            public Builder sampled(Boolean sampled) {
                return this;
            }

            @Override
            public TraceContext build() {
                return TraceContext.NOOP;
            }
        };

        Builder traceId(String traceId);

        Builder parentId(String parentId);

        Builder spanId(String spanId);

        Builder sampled(Boolean sampled);

        TraceContext build();

    }

}
```

#### File: `src/main/java/dev/linhvu/tracing/Span.java` [NEW]

```java
package dev.linhvu.tracing;

import java.util.concurrent.TimeUnit;

/**
 * The central tracing abstraction — represents one unit of work within a distributed trace.
 * Extends {@link SpanCustomizer} with lifecycle methods (start/end), error recording,
 * trace context access, and remote service metadata.
 *
 * <p>Real-world analogy: one leg of a package delivery (pickup → warehouse, warehouse → truck).
 */
public interface Span extends SpanCustomizer {

    /**
     * A no-op span that silently ignores all calls.
     * Used when tracing is disabled or the span is not sampled.
     */
    Span NOOP = new Span() {
        @Override
        public boolean isNoop() {
            return true;
        }

        @Override
        public TraceContext context() {
            return TraceContext.NOOP;
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
        public Span event(String value, long time, TimeUnit timeUnit) {
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
        public void end(long time, TimeUnit timeUnit) {
        }

        @Override
        public void abandon() {
        }

        @Override
        public Span remoteServiceName(String remoteServiceName) {
            return this;
        }

        @Override
        public Span remoteIpAndPort(String ip, int port) {
            return this;
        }
    };

    /**
     * Returns true if this is a no-op span (not recording).
     */
    boolean isNoop();

    /**
     * Returns the trace context (traceId, spanId, parentId, sampled) for this span.
     */
    TraceContext context();

    /**
     * Starts the span — sets the start timestamp. Must be called before end().
     */
    Span start();

    // --- Override SpanCustomizer methods with covariant return types ---

    @Override
    Span name(String name);

    @Override
    Span tag(String key, String value);

    @Override
    Span event(String value);

    /**
     * Records a timestamped event on this span.
     * @param value event description
     * @param time timestamp value
     * @param timeUnit unit of the timestamp
     */
    Span event(String value, long time, TimeUnit timeUnit);

    /**
     * Records an error on this span.
     */
    Span error(Throwable throwable);

    /**
     * Ends (finishes) the span — sets the end timestamp and reports it.
     */
    void end();

    /**
     * Ends the span with an explicit timestamp.
     */
    void end(long time, TimeUnit timeUnit);

    /**
     * Abandons the span — it will not be reported. Use when the operation
     * is aborted and the span data is not meaningful.
     */
    void abandon();

    /**
     * Sets the name of the remote service this span communicates with.
     */
    Span remoteServiceName(String remoteServiceName);

    /**
     * Sets the IP and port of the remote service.
     */
    Span remoteIpAndPort(String ip, int port);

    // --- Default tag overloads with covariant return ---

    @Override
    default Span tag(String key, long value) {
        return tag(key, String.valueOf(value));
    }

    @Override
    default Span tag(String key, double value) {
        return tag(key, String.valueOf(value));
    }

    @Override
    default Span tag(String key, boolean value) {
        return tag(key, String.valueOf(value));
    }

    // =========================================================================
    // Inner types
    // =========================================================================

    /**
     * The type of span — indicates the role this service plays in the communication.
     */
    enum Kind {
        /** Server-side handling of an RPC or remote request. */
        SERVER,
        /** Client-side wrapper around an RPC or remote request. */
        CLIENT,
        /** Producer sending a message to a broker. */
        PRODUCER,
        /** Consumer receiving a message from a broker. */
        CONSUMER
    }

    /**
     * Builder for constructing and starting a {@link Span}.
     * Allows setting parent context, name, tags, kind, and remote info before starting.
     */
    interface Builder {

        Builder NOOP = new Builder() {
            @Override
            public Builder setParent(TraceContext context) {
                return this;
            }

            @Override
            public Builder setNoParent() {
                return this;
            }

            @Override
            public Builder name(String name) {
                return this;
            }

            @Override
            public Builder event(String value) {
                return this;
            }

            @Override
            public Builder tag(String key, String value) {
                return this;
            }

            @Override
            public Builder error(Throwable throwable) {
                return this;
            }

            @Override
            public Builder kind(Kind spanKind) {
                return this;
            }

            @Override
            public Builder remoteServiceName(String remoteServiceName) {
                return this;
            }

            @Override
            public Builder remoteIpAndPort(String ip, int port) {
                return this;
            }

            @Override
            public Builder startTimestamp(long startTimestamp, TimeUnit unit) {
                return this;
            }

            @Override
            public Span start() {
                return Span.NOOP;
            }
        };

        /**
         * Sets the parent trace context. The new span will be a child of this context.
         */
        Builder setParent(TraceContext context);

        /**
         * Makes this a root span with no parent.
         */
        Builder setNoParent();

        Builder name(String name);

        Builder event(String value);

        Builder tag(String key, String value);

        Builder error(Throwable throwable);

        /**
         * Sets the span kind (SERVER, CLIENT, PRODUCER, CONSUMER).
         */
        Builder kind(Kind spanKind);

        Builder remoteServiceName(String remoteServiceName);

        Builder remoteIpAndPort(String ip, int port);

        /**
         * Sets an explicit start timestamp instead of using the current time.
         */
        Builder startTimestamp(long startTimestamp, TimeUnit unit);

        /**
         * Starts and returns the span. After this call, the span is recording.
         */
        Span start();

        // --- Default convenience methods ---

        default Builder tag(String key, long value) {
            return tag(key, String.valueOf(value));
        }

        default Builder tag(String key, double value) {
            return tag(key, String.valueOf(value));
        }

        default Builder tag(String key, boolean value) {
            return tag(key, String.valueOf(value));
        }

        /**
         * Adds a link to another span. Default implementation is a no-op;
         * tracer implementations override to actually store links.
         */
        default Builder addLink(Link link) {
            return this;
        }

    }

}
```

#### File: `src/main/java/dev/linhvu/tracing/ScopedSpan.java` [NEW]

```java
package dev.linhvu.tracing;

/**
 * A span that is automatically "in scope" from creation until {@link #end()} is called.
 * Unlike {@link Span}, a ScopedSpan is already started and scoped upon creation —
 * you don't call start() or manage scope manually.
 *
 * <p>Use this when you want a simple try-with-resources pattern:
 * <pre>{@code
 * ScopedSpan span = tracer.startScopedSpan("my-operation");
 * try {
 *     // do work — this span is the "current" span
 *     span.tag("result", "success");
 * } catch (Exception e) {
 *     span.error(e);
 * } finally {
 *     span.end(); // removes from scope and finishes the span
 * }
 * }</pre>
 */
public interface ScopedSpan {

    /**
     * A no-op scoped span that silently ignores all calls.
     */
    ScopedSpan NOOP = new ScopedSpan() {
        @Override
        public boolean isNoop() {
            return true;
        }

        @Override
        public TraceContext context() {
            return TraceContext.NOOP;
        }

        @Override
        public ScopedSpan name(String name) {
            return this;
        }

        @Override
        public ScopedSpan tag(String key, String value) {
            return this;
        }

        @Override
        public ScopedSpan event(String value) {
            return this;
        }

        @Override
        public ScopedSpan error(Throwable throwable) {
            return this;
        }

        @Override
        public void end() {
        }
    };

    /**
     * Returns true if this is a no-op scoped span.
     */
    boolean isNoop();

    /**
     * Returns the trace context for this span.
     */
    TraceContext context();

    /**
     * Sets the name of this span.
     */
    ScopedSpan name(String name);

    /**
     * Adds a tag to this span.
     */
    ScopedSpan tag(String key, String value);

    /**
     * Records an event on this span.
     */
    ScopedSpan event(String value);

    /**
     * Records an error on this span.
     */
    ScopedSpan error(Throwable throwable);

    /**
     * Ends the span — removes it from scope and finishes recording.
     */
    void end();

}
```

#### File: `src/main/java/dev/linhvu/tracing/Link.java` [NEW]

```java
package dev.linhvu.tracing;

import java.util.Collections;
import java.util.Map;
import java.util.Objects;

/**
 * A link between spans — typically across different traces.
 * Links allow associating spans that are causally related but don't have a direct
 * parent-child relationship (e.g., a batch job linking to the requests that triggered it).
 */
public class Link {

    /**
     * A no-op link with an empty trace context and no tags.
     */
    public static final Link NOOP = new Link(TraceContext.NOOP, Collections.emptyMap());

    private final TraceContext traceContext;

    private final Map<String, Object> tags;

    public Link(TraceContext traceContext, Map<String, Object> tags) {
        this.traceContext = traceContext;
        this.tags = tags;
    }

    public Link(TraceContext traceContext) {
        this(traceContext, Collections.emptyMap());
    }

    public Link(Span span, Map<String, Object> tags) {
        this(span.context(), tags);
    }

    public Link(Span span) {
        this(span.context(), Collections.emptyMap());
    }

    public TraceContext getTraceContext() {
        return traceContext;
    }

    public Map<String, Object> getTags() {
        return tags;
    }

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (o == null || getClass() != o.getClass()) return false;
        Link link = (Link) o;
        return Objects.equals(traceContext, link.traceContext) && Objects.equals(tags, link.tags);
    }

    @Override
    public int hashCode() {
        return Objects.hash(traceContext, tags);
    }

    @Override
    public String toString() {
        return "Link{traceContext=" + traceContext + ", tags=" + tags + "}";
    }

}
```

### Test Code

#### File: `src/test/java/dev/linhvu/tracing/SpanCustomizerTest.java` [NEW]

```java
package dev.linhvu.tracing;

import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;

class SpanCustomizerTest {

    @Test
    void shouldReturnSelf_WhenNoopNameCalled() {
        SpanCustomizer noop = SpanCustomizer.NOOP;
        assertThat(noop.name("test")).isSameAs(noop);
    }

    @Test
    void shouldReturnSelf_WhenNoopTagCalled() {
        SpanCustomizer noop = SpanCustomizer.NOOP;
        assertThat(noop.tag("key", "value")).isSameAs(noop);
    }

    @Test
    void shouldReturnSelf_WhenNoopEventCalled() {
        SpanCustomizer noop = SpanCustomizer.NOOP;
        assertThat(noop.event("event")).isSameAs(noop);
    }

    @Test
    void shouldDelegateToStringTag_WhenLongTagCalled() {
        var recorder = new RecordingSpanCustomizer();
        recorder.tag("count", 42L);
        assertThat(recorder.lastTagKey).isEqualTo("count");
        assertThat(recorder.lastTagValue).isEqualTo("42");
    }

    @Test
    void shouldDelegateToStringTag_WhenDoubleTagCalled() {
        var recorder = new RecordingSpanCustomizer();
        recorder.tag("rate", 3.14);
        assertThat(recorder.lastTagKey).isEqualTo("rate");
        assertThat(recorder.lastTagValue).isEqualTo("3.14");
    }

    @Test
    void shouldDelegateToStringTag_WhenBooleanTagCalled() {
        var recorder = new RecordingSpanCustomizer();
        recorder.tag("enabled", true);
        assertThat(recorder.lastTagKey).isEqualTo("enabled");
        assertThat(recorder.lastTagValue).isEqualTo("true");
    }

    static class RecordingSpanCustomizer implements SpanCustomizer {
        String lastTagKey;
        String lastTagValue;

        @Override
        public SpanCustomizer name(String name) {
            return this;
        }

        @Override
        public SpanCustomizer tag(String key, String value) {
            this.lastTagKey = key;
            this.lastTagValue = value;
            return this;
        }

        @Override
        public SpanCustomizer event(String value) {
            return this;
        }
    }

}
```

#### File: `src/test/java/dev/linhvu/tracing/TraceContextTest.java` [NEW]

```java
package dev.linhvu.tracing;

import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;

class TraceContextTest {

    @Test
    void shouldReturnEmptyTraceId_WhenNoopUsed() {
        assertThat(TraceContext.NOOP.traceId()).isEmpty();
    }

    @Test
    void shouldReturnEmptySpanId_WhenNoopUsed() {
        assertThat(TraceContext.NOOP.spanId()).isEmpty();
    }

    @Test
    void shouldReturnNullParentId_WhenNoopUsed() {
        assertThat(TraceContext.NOOP.parentId()).isNull();
    }

    @Test
    void shouldReturnNullSampled_WhenNoopUsed() {
        assertThat(TraceContext.NOOP.sampled()).isNull();
    }

    @Test
    void shouldReturnSelf_WhenNoopBuilderMethodsCalled() {
        TraceContext.Builder builder = TraceContext.Builder.NOOP;
        assertThat(builder.traceId("abc")).isSameAs(builder);
        assertThat(builder.parentId("def")).isSameAs(builder);
        assertThat(builder.spanId("ghi")).isSameAs(builder);
        assertThat(builder.sampled(true)).isSameAs(builder);
    }

    @Test
    void shouldReturnNoopContext_WhenNoopBuilderBuilds() {
        TraceContext ctx = TraceContext.Builder.NOOP.build();
        assertThat(ctx).isSameAs(TraceContext.NOOP);
    }

}
```

#### File: `src/test/java/dev/linhvu/tracing/SpanTest.java` [NEW]

```java
package dev.linhvu.tracing;

import org.junit.jupiter.api.Test;

import java.util.concurrent.TimeUnit;

import static org.assertj.core.api.Assertions.assertThat;

class SpanTest {

    @Test
    void shouldBeNoop_WhenNoopSpanUsed() {
        assertThat(Span.NOOP.isNoop()).isTrue();
    }

    @Test
    void shouldReturnNoopContext_WhenNoopSpanUsed() {
        assertThat(Span.NOOP.context()).isSameAs(TraceContext.NOOP);
    }

    @Test
    void shouldReturnSelf_WhenNoopSpanLifecycleCalled() {
        Span noop = Span.NOOP;
        assertThat(noop.start()).isSameAs(noop);
        assertThat(noop.name("test")).isSameAs(noop);
        assertThat(noop.tag("k", "v")).isSameAs(noop);
        assertThat(noop.event("evt")).isSameAs(noop);
        assertThat(noop.event("evt", 1L, TimeUnit.MILLISECONDS)).isSameAs(noop);
        assertThat(noop.error(new RuntimeException())).isSameAs(noop);
        assertThat(noop.remoteServiceName("svc")).isSameAs(noop);
        assertThat(noop.remoteIpAndPort("127.0.0.1", 8080)).isSameAs(noop);
        noop.end();
        noop.end(1L, TimeUnit.MILLISECONDS);
        noop.abandon();
    }

    @Test
    void shouldHaveFourKindValues() {
        assertThat(Span.Kind.values()).containsExactly(
                Span.Kind.SERVER, Span.Kind.CLIENT,
                Span.Kind.PRODUCER, Span.Kind.CONSUMER
        );
    }

    @Test
    void shouldReturnSelf_WhenNoopBuilderMethodsCalled() {
        Span.Builder builder = Span.Builder.NOOP;
        assertThat(builder.setParent(TraceContext.NOOP)).isSameAs(builder);
        assertThat(builder.setNoParent()).isSameAs(builder);
        assertThat(builder.name("test")).isSameAs(builder);
        assertThat(builder.event("evt")).isSameAs(builder);
        assertThat(builder.tag("k", "v")).isSameAs(builder);
        assertThat(builder.error(new RuntimeException())).isSameAs(builder);
        assertThat(builder.kind(Span.Kind.SERVER)).isSameAs(builder);
        assertThat(builder.remoteServiceName("svc")).isSameAs(builder);
        assertThat(builder.remoteIpAndPort("127.0.0.1", 8080)).isSameAs(builder);
        assertThat(builder.startTimestamp(1L, TimeUnit.MILLISECONDS)).isSameAs(builder);
    }

    @Test
    void shouldReturnNoopSpan_WhenNoopBuilderStarts() {
        Span span = Span.Builder.NOOP.start();
        assertThat(span).isSameAs(Span.NOOP);
    }

    @Test
    void shouldReturnSelf_WhenNoopBuilderAddLinkCalled() {
        Span.Builder builder = Span.Builder.NOOP;
        assertThat(builder.addLink(Link.NOOP)).isSameAs(builder);
    }

    @Test
    void shouldDelegateToStringTag_WhenBuilderLongTagCalled() {
        var recorder = new RecordingBuilder();
        recorder.tag("count", 42L);
        assertThat(recorder.lastTagKey).isEqualTo("count");
        assertThat(recorder.lastTagValue).isEqualTo("42");
    }

    @Test
    void shouldDelegateToStringTag_WhenBuilderDoubleTagCalled() {
        var recorder = new RecordingBuilder();
        recorder.tag("rate", 3.14);
        assertThat(recorder.lastTagKey).isEqualTo("rate");
        assertThat(recorder.lastTagValue).isEqualTo("3.14");
    }

    @Test
    void shouldDelegateToStringTag_WhenBuilderBooleanTagCalled() {
        var recorder = new RecordingBuilder();
        recorder.tag("enabled", true);
        assertThat(recorder.lastTagKey).isEqualTo("enabled");
        assertThat(recorder.lastTagValue).isEqualTo("true");
    }

    static class RecordingBuilder implements Span.Builder {
        String lastTagKey;
        String lastTagValue;

        @Override public Span.Builder setParent(TraceContext context) { return this; }
        @Override public Span.Builder setNoParent() { return this; }
        @Override public Span.Builder name(String name) { return this; }
        @Override public Span.Builder event(String value) { return this; }
        @Override public Span.Builder tag(String key, String value) {
            this.lastTagKey = key;
            this.lastTagValue = value;
            return this;
        }
        @Override public Span.Builder error(Throwable throwable) { return this; }
        @Override public Span.Builder kind(Span.Kind spanKind) { return this; }
        @Override public Span.Builder remoteServiceName(String remoteServiceName) { return this; }
        @Override public Span.Builder remoteIpAndPort(String ip, int port) { return this; }
        @Override public Span.Builder startTimestamp(long startTimestamp, TimeUnit unit) { return this; }
        @Override public Span start() { return Span.NOOP; }
    }

}
```

#### File: `src/test/java/dev/linhvu/tracing/ScopedSpanTest.java` [NEW]

```java
package dev.linhvu.tracing;

import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;

class ScopedSpanTest {

    @Test
    void shouldBeNoop_WhenNoopScopedSpanUsed() {
        assertThat(ScopedSpan.NOOP.isNoop()).isTrue();
    }

    @Test
    void shouldReturnNoopContext_WhenNoopScopedSpanUsed() {
        assertThat(ScopedSpan.NOOP.context()).isSameAs(TraceContext.NOOP);
    }

    @Test
    void shouldReturnSelf_WhenNoopScopedSpanMethodsCalled() {
        ScopedSpan noop = ScopedSpan.NOOP;
        assertThat(noop.name("test")).isSameAs(noop);
        assertThat(noop.tag("k", "v")).isSameAs(noop);
        assertThat(noop.event("evt")).isSameAs(noop);
        assertThat(noop.error(new RuntimeException())).isSameAs(noop);
        noop.end();
    }

    @Test
    void shouldHaveFewerMethodsThanSpan_WhenComparedConceptually() {
        ScopedSpan scoped = ScopedSpan.NOOP;
        scoped.name("op").tag("k", "v").event("evt").error(new RuntimeException());
        scoped.end();
    }

}
```

#### File: `src/test/java/dev/linhvu/tracing/LinkTest.java` [NEW]

```java
package dev.linhvu.tracing;

import org.junit.jupiter.api.Test;

import java.util.Collections;
import java.util.Map;

import static org.assertj.core.api.Assertions.assertThat;

class LinkTest {

    @Test
    void shouldStoreTraceContextAndTags_WhenCreatedWithBoth() {
        TraceContext ctx = TraceContext.NOOP;
        Map<String, Object> tags = Map.of("reason", "batch-trigger");

        Link link = new Link(ctx, tags);

        assertThat(link.getTraceContext()).isSameAs(ctx);
        assertThat(link.getTags()).isEqualTo(tags);
    }

    @Test
    void shouldHaveEmptyTags_WhenCreatedWithContextOnly() {
        Link link = new Link(TraceContext.NOOP);

        assertThat(link.getTraceContext()).isSameAs(TraceContext.NOOP);
        assertThat(link.getTags()).isEmpty();
    }

    @Test
    void shouldExtractContextFromSpan_WhenCreatedWithSpan() {
        Link link = new Link(Span.NOOP, Map.of("key", "value"));

        assertThat(link.getTraceContext()).isSameAs(TraceContext.NOOP);
        assertThat(link.getTags()).containsEntry("key", "value");
    }

    @Test
    void shouldExtractContextFromSpan_WhenCreatedWithSpanOnly() {
        Link link = new Link(Span.NOOP);

        assertThat(link.getTraceContext()).isSameAs(TraceContext.NOOP);
        assertThat(link.getTags()).isEmpty();
    }

    @Test
    void shouldBeEqual_WhenSameContextAndTags() {
        Map<String, Object> tags = Map.of("k", "v");
        Link link1 = new Link(TraceContext.NOOP, tags);
        Link link2 = new Link(TraceContext.NOOP, tags);

        assertThat(link1).isEqualTo(link2);
        assertThat(link1.hashCode()).isEqualTo(link2.hashCode());
    }

    @Test
    void shouldNotBeEqual_WhenDifferentTags() {
        Link link1 = new Link(TraceContext.NOOP, Map.of("k", "v1"));
        Link link2 = new Link(TraceContext.NOOP, Map.of("k", "v2"));

        assertThat(link1).isNotEqualTo(link2);
    }

    @Test
    void shouldHaveReadableToString() {
        Link link = new Link(TraceContext.NOOP, Map.of("reason", "test"));

        assertThat(link.toString()).contains("Link{");
        assertThat(link.toString()).contains("traceContext=");
        assertThat(link.toString()).contains("tags=");
    }

    @Test
    void shouldHaveNoopLinkConstant() {
        assertThat(Link.NOOP.getTraceContext()).isSameAs(TraceContext.NOOP);
        assertThat(Link.NOOP.getTags()).isEqualTo(Collections.emptyMap());
    }

}
```

---

## Summary

| Concept | What It Means |
|---------|---------------|
| **SpanCustomizer** | Narrow interface for adding name/tag/event to the current span — no lifecycle control |
| **TraceContext** | The "tracking number" — traceId + spanId + parentId + sampled decision |
| **Span** | Full tracing abstraction: lifecycle (start/end), error recording, remote metadata, extends SpanCustomizer |
| **ScopedSpan** | Convenience span that is automatically in scope from creation until end() |
| **Link** | Value object connecting spans across traces without parent-child relationship |
| **Span.Kind** | Role in communication: SERVER, CLIENT, PRODUCER, CONSUMER |
| **NOOP pattern** | Every type has a built-in no-op instance — safe to call when tracing is disabled |
| **Covariant return** | Span overrides SpanCustomizer methods to return `Span` for better fluent chaining |

**Next: Chapter 2 — Tracer & Scope Management** — the central facade that creates spans, manages which span is "current" on the thread, and provides the entry point users interact with
