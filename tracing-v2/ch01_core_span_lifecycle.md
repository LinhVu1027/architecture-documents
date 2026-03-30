# Chapter 1: Core Span Lifecycle — Create, Scope, and End Spans

> **What you'll build**: The six core tracing interfaces — `SpanCustomizer`, `TraceContext`,
> `Span`, `ScopedSpan`, `CurrentTraceContext`, and `Tracer` — with full NOOP cascade, enabling
> clients to create spans, scope them, tag them, and end them.

---

## 1. The API Contract

After this chapter, a client can write:

```java
// Simple pattern — ScopedSpan
Tracer tracer = ...;
ScopedSpan span = tracer.startScopedSpan("encode");
try {
    span.tag("input.length", "256");
    span.event("encoding-started");
    return encoder.encode();
} catch (RuntimeException | Error e) {
    span.error(e);
    throw e;
} finally {
    span.end();
}

// Advanced pattern — Span + explicit scope
Span span = tracer.nextSpan().name("encode").start();
try (Tracer.SpanInScope ws = tracer.withSpan(span)) {
    return encoder.encode();
} catch (RuntimeException | Error e) {
    span.error(e);
    throw e;
} finally {
    span.end();
}
```

**Two patterns, one concept**: Both create a span to measure a unit of work. `ScopedSpan`
bundles span + scope (simpler, for single-threaded code). `Span` + `SpanInScope` keeps them
separate (for async code where scope and span have different lifetimes).

### Interface Hierarchy

```
SpanCustomizer              ← name, tag, event (for business code)
    ▲ extends
    │
  Span                      ← + lifecycle: start, end, abandon, context, isNoop
                               + inner: Kind enum, Builder interface

ScopedSpan                  ← standalone (same 3 methods but returns ScopedSpan)
                               + isNoop, context, end (closes scope too)

TraceContext                ← identity: traceId, spanId, parentId, sampled
                               + inner: Builder interface

CurrentTraceContext         ← thread-local scope: context, newScope, maybeScope
                               + inner: Scope interface

Tracer                      ← orchestrator: nextSpan, withSpan, startScopedSpan
                               + inner: SpanInScope interface
```

---

## 2. Client Usage & Tests

### Test: ScopedSpan with tags and events

This is the most common client pattern — create a scoped span, tag it, and end it:

```java
@Test
void shouldCreateScopedSpan_withTagsAndEvents() {
    TestTracer tracer = new TestTracer();

    ScopedSpan span = tracer.startScopedSpan("encode");
    try {
        span.tag("input.length", "256");
        span.event("encoding-started");
    } finally {
        span.end();
    }

    assertThat(tracer.finishedSpans()).hasSize(1);
    TestTracer.TestSpanData finished = tracer.finishedSpans().get(0);
    assertThat(finished.name).isEqualTo("encode");
    assertThat(finished.tags).containsEntry("input.length", "256");
    assertThat(finished.events).containsExactly("encoding-started");
}
```

### Test: Automatic parent-child via scoping

When a span is in scope, new spans automatically become its children:

```java
@Test
void scopedSpan_shouldRestorePreviousContext_onEnd() {
    TestTracer tracer = new TestTracer();

    assertThat(tracer.currentSpan()).isNull();

    ScopedSpan outer = tracer.startScopedSpan("outer");
    TraceContext outerCtx = outer.context();
    assertThat(tracer.currentTraceContext().context()).isSameAs(outerCtx);

    ScopedSpan inner = tracer.startScopedSpan("inner");
    assertThat(inner.context().parentId()).isEqualTo(outerCtx.spanId());

    inner.end();  // restores outer as current
    assertThat(tracer.currentTraceContext().context()).isSameAs(outerCtx);

    outer.end();  // clears scope
    assertThat(tracer.currentTraceContext().context()).isNull();
}
```

**Why this works**: `startScopedSpan` checks `CurrentTraceContext.context()` for the current
parent. If one exists, the new span inherits the same trace ID and sets its parent ID. When
`end()` is called, it closes the scope, restoring the previous context.

### Test: Span + SpanInScope (advanced pattern)

Scope and span lifecycle are independent — closing scope doesn't end the span:

```java
@Test
void spanScope_shouldBeIndependentOfSpanLifecycle() {
    TestTracer tracer = new TestTracer();

    Span span = tracer.nextSpan().name("async-op").start();
    Tracer.SpanInScope ws = tracer.withSpan(span);

    ws.close();  // scope closed, but span is NOT ended
    assertThat(tracer.currentSpan()).isNull();
    assertThat(tracer.finishedSpans()).isEmpty();

    span.end();  // NOW the span is recorded
    assertThat(tracer.finishedSpans()).hasSize(1);
}
```

### Test: NOOP cascade

The NOOP pattern eliminates null checks — every NOOP returns other NOOPs:

```java
@Test
void noopTracer_shouldReturnNoopSpans() {
    Tracer tracer = Tracer.NOOP;

    Span span = tracer.nextSpan();
    assertThat(span.isNoop()).isTrue();
    assertThat(span.context()).isSameAs(TraceContext.NOOP);
}
```

---

## 3. Implementing the Call Chain — Top-Down

### Layer 1: SpanCustomizer (the simplest interface)

The minimum viable tracing interface — 3 methods. Inject this into business code that
should annotate the current span but should NOT control its lifecycle:

```java
public interface SpanCustomizer {

    SpanCustomizer NOOP = new SpanCustomizer() {
        @Override public SpanCustomizer name(String name) { return this; }
        @Override public SpanCustomizer tag(String key, String value) { return this; }
        @Override public SpanCustomizer event(String value) { return this; }
    };

    SpanCustomizer name(String name);
    SpanCustomizer tag(String key, String value);
    SpanCustomizer event(String value);
}
```

**Design decision**: All setters return `this` for fluent chaining. The NOOP returns itself,
so chaining `SpanCustomizer.NOOP.name("x").tag("k", "v")` works safely.

### Layer 2: TraceContext (immutable identity)

The identity carrier — holds `traceId`, `spanId`, `parentId`, and `sampled`. This is the
data that travels across service boundaries:

```java
public interface TraceContext {

    TraceContext NOOP = new TraceContext() {
        @Override public String traceId() { return ""; }
        @Override public String parentId() { return null; }
        @Override public String spanId() { return ""; }
        @Override public Boolean sampled() { return null; }
    };

    String traceId();
    String parentId();     // null for root spans
    String spanId();
    Boolean sampled();     // true/false/null (deferred)

    interface Builder {
        Builder NOOP = /* returns this, builds TraceContext.NOOP */;

        Builder traceId(String traceId);
        Builder parentId(String parentId);
        Builder spanId(String spanId);
        Builder sampled(Boolean sampled);
        TraceContext build();
    }
}
```

**Key insight**: `sampled()` returns `Boolean` (nullable), not `boolean`. The tri-state
represents: `true` = definitely record, `false` = definitely don't, `null` = let downstream
decide. This deferred sampling is important for distributed tracing.

### Layer 3: Span (lifecycle + identity)

Extends `SpanCustomizer` with lifecycle control. The `Kind` enum and `Builder` are inner types:

```java
public interface Span extends SpanCustomizer {

    Span NOOP = /* isNoop()=true, context()=TraceContext.NOOP, all setters return this */;

    boolean isNoop();
    TraceContext context();
    Span start();
    Span name(String name);
    Span event(String value);
    Span tag(String key, String value);
    Span error(Throwable throwable);
    void end();
    void abandon();
    Span remoteServiceName(String remoteServiceName);

    enum Kind { SERVER, CLIENT, PRODUCER, CONSUMER }

    interface Builder {
        Builder NOOP = /* all setters return this, start() returns Span.NOOP */;

        Builder setParent(TraceContext context);
        Builder setNoParent();
        Builder name(String name);
        Builder event(String value);
        Builder tag(String key, String value);
        Builder error(Throwable throwable);
        Builder kind(Span.Kind spanKind);
        Builder remoteServiceName(String remoteServiceName);
        Span start();
    }
}
```

**`end()` vs `abandon()`**: `end()` records the span (it will be exported/reported).
`abandon()` ends the span without recording — useful when you decide mid-flight to discard it.

### Layer 4: ScopedSpan (span + scope combined)

Simpler than `Span` — no `start()` (already started), no `remoteServiceName` (in-process
only), and `end()` closes both the span and the scope:

```java
public interface ScopedSpan {

    ScopedSpan NOOP = /* isNoop()=true, context()=TraceContext.NOOP */;

    boolean isNoop();
    TraceContext context();
    ScopedSpan name(String name);
    ScopedSpan tag(String key, String value);
    ScopedSpan event(String value);
    ScopedSpan error(Throwable throwable);
    void end();  // ends span AND closes scope
}
```

**Why ScopedSpan doesn't extend SpanCustomizer**: The return types differ. `SpanCustomizer.name()`
returns `SpanCustomizer`, but `ScopedSpan.name()` returns `ScopedSpan`. Java doesn't support
covariant return types on interface inheritance for fluent APIs, so `ScopedSpan` is standalone.

### Layer 5: CurrentTraceContext (thread-local scope)

Manages which `TraceContext` is "current" on this thread. This is the mechanism behind
automatic parent-child relationships:

```java
public interface CurrentTraceContext {

    CurrentTraceContext NOOP = /* context()=TraceContext.NOOP, scopes are no-ops */;

    TraceContext context();
    Scope newScope(TraceContext context);
    Scope maybeScope(TraceContext context);

    interface Scope extends Closeable {
        Scope NOOP = () -> {};
        @Override void close();
    }
}
```

**`newScope` vs `maybeScope`**: `maybeScope` is an optimization — if the given context is
already current, it returns `Scope.NOOP` instead of creating a real scope. This avoids
unnecessary work in recursive or re-entrant code.

### Layer 6: Tracer (the orchestrator)

Ties everything together. Two creation patterns, scope management, and accessor methods:

```java
public interface Tracer {

    Tracer NOOP = /* all methods return their respective NOOPs */;

    Span nextSpan();
    Span nextSpan(Span parent);
    SpanInScope withSpan(Span span);
    ScopedSpan startScopedSpan(String name);
    Span.Builder spanBuilder();
    TraceContext.Builder traceContextBuilder();
    CurrentTraceContext currentTraceContext();
    SpanCustomizer currentSpanCustomizer();
    Span currentSpan();

    interface SpanInScope extends Closeable {
        @Override void close();
    }
}
```

**How `startScopedSpan` works internally**: It calls `nextSpan()` (which checks the current
context for a parent), sets the name, starts the span, then calls `withSpan()` to place it
in scope. When `ScopedSpan.end()` is called, it closes the scope (restoring the previous
context) and ends the span (recording it).

---

## 4. Try It Yourself

1. **Add a `tag(String, long)` default method** to `SpanCustomizer` that delegates to
   `tag(key, String.valueOf(value))`. The real framework has this.

2. **Add `event(String, long, TimeUnit)`** to `Span` for recording events with explicit
   timestamps. The NOOP should ignore the extra parameters.

3. **Write a test** that creates 3 levels of nested `ScopedSpan` and verifies the full
   parent-child chain: grandchild → child → root.

---

## 5. Why This Works

### The NOOP Cascade
Every interface has a `NOOP` constant, and every NOOP returns other NOOPs. This means
`Tracer.NOOP.nextSpan().name("x").tag("k","v").start()` never throws — the entire chain
is safe. This eliminates defensive null-checking throughout the codebase. It's the
Null Object pattern applied systematically across an interface graph.

### Scope as Closeable
Both `Tracer.SpanInScope` and `CurrentTraceContext.Scope` extend `Closeable`, enabling
try-with-resources:
```java
try (Tracer.SpanInScope ws = tracer.withSpan(span)) {
    // span is "current" here
}
// previous span restored automatically
```
This makes scope cleanup deterministic — no leaked thread-local state.

### Separation of SpanCustomizer and Span
Business code gets `SpanCustomizer` (can tag, can't stop). Instrumentation code gets
`Span` (full lifecycle). This enforces least-privilege at the type level.

---

## 6. What We Enhanced

| Component | State After Ch01 | Simplifications |
|-----------|-------------------|-----------------|
| `SpanCustomizer` | 3 abstract methods | No `tag(String, long/double/boolean)` defaults |
| `TraceContext` | 4 fields + Builder | — |
| `Span` | Core lifecycle + Kind + Builder | No timed `event`/`end`, no `remoteIpAndPort` |
| `ScopedSpan` | 7 methods | — |
| `CurrentTraceContext` | context + newScope + maybeScope | No `wrap()` methods for async propagation |
| `Tracer` | 9 methods + SpanInScope | Does not extend `BaggageManager` |

---

## 7. Connection to Real Framework

| Simplified Class | Real Framework Class | Source File |
|------------------|----------------------|-------------|
| `io.simpletracing.SpanCustomizer` | `io.micrometer.tracing.SpanCustomizer` | `micrometer-tracing/src/main/java/io/micrometer/tracing/SpanCustomizer.java` |
| `io.simpletracing.TraceContext` | `io.micrometer.tracing.TraceContext` | `micrometer-tracing/src/main/java/io/micrometer/tracing/TraceContext.java` |
| `io.simpletracing.Span` | `io.micrometer.tracing.Span` | `micrometer-tracing/src/main/java/io/micrometer/tracing/Span.java` |
| `io.simpletracing.ScopedSpan` | `io.micrometer.tracing.ScopedSpan` | `micrometer-tracing/src/main/java/io/micrometer/tracing/ScopedSpan.java` |
| `io.simpletracing.CurrentTraceContext` | `io.micrometer.tracing.CurrentTraceContext` | `micrometer-tracing/src/main/java/io/micrometer/tracing/CurrentTraceContext.java` |
| `io.simpletracing.Tracer` | `io.micrometer.tracing.Tracer` | `micrometer-tracing/src/main/java/io/micrometer/tracing/Tracer.java` |

---

## 8. Complete Code

### `simple-tracing-api/src/main/java/io/simpletracing/SpanCustomizer.java` [NEW]

```java
package io.simpletracing;

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

    SpanCustomizer event(String value);

}
```

### `simple-tracing-api/src/main/java/io/simpletracing/TraceContext.java` [NEW]

```java
package io.simpletracing;

public interface TraceContext {

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

    String traceId();

    String parentId();

    String spanId();

    Boolean sampled();

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

### `simple-tracing-api/src/main/java/io/simpletracing/Span.java` [NEW]

```java
package io.simpletracing;

public interface Span extends SpanCustomizer {

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
    };

    boolean isNoop();

    TraceContext context();

    Span start();

    @Override
    Span name(String name);

    @Override
    Span event(String value);

    @Override
    Span tag(String key, String value);

    Span error(Throwable throwable);

    void end();

    void abandon();

    Span remoteServiceName(String remoteServiceName);

    enum Kind {
        SERVER,
        CLIENT,
        PRODUCER,
        CONSUMER
    }

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
            public Builder kind(Span.Kind spanKind) {
                return this;
            }

            @Override
            public Builder remoteServiceName(String remoteServiceName) {
                return this;
            }

            @Override
            public Span start() {
                return Span.NOOP;
            }
        };

        Builder setParent(TraceContext context);

        Builder setNoParent();

        Builder name(String name);

        Builder event(String value);

        Builder tag(String key, String value);

        Builder error(Throwable throwable);

        Builder kind(Span.Kind spanKind);

        Builder remoteServiceName(String remoteServiceName);

        Span start();

    }

}
```

### `simple-tracing-api/src/main/java/io/simpletracing/ScopedSpan.java` [NEW]

```java
package io.simpletracing;

public interface ScopedSpan {

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

    boolean isNoop();

    TraceContext context();

    ScopedSpan name(String name);

    ScopedSpan tag(String key, String value);

    ScopedSpan event(String value);

    ScopedSpan error(Throwable throwable);

    void end();

}
```

### `simple-tracing-api/src/main/java/io/simpletracing/CurrentTraceContext.java` [NEW]

```java
package io.simpletracing;

import java.io.Closeable;

public interface CurrentTraceContext {

    CurrentTraceContext NOOP = new CurrentTraceContext() {
        @Override
        public TraceContext context() {
            return TraceContext.NOOP;
        }

        @Override
        public Scope newScope(TraceContext context) {
            return Scope.NOOP;
        }

        @Override
        public Scope maybeScope(TraceContext context) {
            return Scope.NOOP;
        }
    };

    TraceContext context();

    Scope newScope(TraceContext context);

    Scope maybeScope(TraceContext context);

    interface Scope extends Closeable {

        Scope NOOP = () -> {
        };

        @Override
        void close();

    }

}
```

### `simple-tracing-api/src/main/java/io/simpletracing/Tracer.java` [NEW]

```java
package io.simpletracing;

import java.io.Closeable;

public interface Tracer {

    Tracer NOOP = new Tracer() {
        @Override
        public Span nextSpan() {
            return Span.NOOP;
        }

        @Override
        public Span nextSpan(Span parent) {
            return Span.NOOP;
        }

        @Override
        public SpanInScope withSpan(Span span) {
            return () -> {
            };
        }

        @Override
        public ScopedSpan startScopedSpan(String name) {
            return ScopedSpan.NOOP;
        }

        @Override
        public Span.Builder spanBuilder() {
            return Span.Builder.NOOP;
        }

        @Override
        public TraceContext.Builder traceContextBuilder() {
            return TraceContext.Builder.NOOP;
        }

        @Override
        public CurrentTraceContext currentTraceContext() {
            return CurrentTraceContext.NOOP;
        }

        @Override
        public SpanCustomizer currentSpanCustomizer() {
            return SpanCustomizer.NOOP;
        }

        @Override
        public Span currentSpan() {
            return Span.NOOP;
        }
    };

    Span nextSpan();

    Span nextSpan(Span parent);

    SpanInScope withSpan(Span span);

    ScopedSpan startScopedSpan(String name);

    Span.Builder spanBuilder();

    TraceContext.Builder traceContextBuilder();

    CurrentTraceContext currentTraceContext();

    SpanCustomizer currentSpanCustomizer();

    Span currentSpan();

    interface SpanInScope extends Closeable {

        @Override
        void close();

    }

}
```

### `simple-tracing-api/src/test/java/io/simpletracing/TracerApiTest.java` [NEW]

```java
package io.simpletracing;

import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;

class TracerApiTest {

    // ── NOOP Cascade Tests ────────────────────────────────────

    @Test
    void noopTracer_shouldReturnNoopSpans() {
        Tracer tracer = Tracer.NOOP;
        Span span = tracer.nextSpan();
        assertThat(span.isNoop()).isTrue();
        assertThat(span.context()).isSameAs(TraceContext.NOOP);
    }

    @Test
    void noopTracer_shouldReturnNoopScopedSpan() {
        ScopedSpan scoped = Tracer.NOOP.startScopedSpan("test");
        assertThat(scoped.isNoop()).isTrue();
        scoped.end();
    }

    @Test
    void noopTracer_shouldReturnNoopBuilder() {
        Span span = Tracer.NOOP.spanBuilder()
                .name("test").kind(Span.Kind.SERVER).tag("k", "v").start();
        assertThat(span.isNoop()).isTrue();
    }

    @Test
    void noopTracer_shouldReturnNoopTraceContextBuilder() {
        TraceContext ctx = Tracer.NOOP.traceContextBuilder()
                .traceId("abc").spanId("def").sampled(true).build();
        assertThat(ctx).isSameAs(TraceContext.NOOP);
    }

    @Test
    void noopTracer_shouldReturnNoopCurrentTraceContext() {
        assertThat(Tracer.NOOP.currentTraceContext()).isSameAs(CurrentTraceContext.NOOP);
        assertThat(Tracer.NOOP.currentSpanCustomizer()).isSameAs(SpanCustomizer.NOOP);
        assertThat(Tracer.NOOP.currentSpan()).isSameAs(Span.NOOP);
    }

    // ── ScopedSpan (simple pattern) ──────────────────────────

    @Test
    void shouldCreateScopedSpan_withTagsAndEvents() {
        TestTracer tracer = new TestTracer();
        ScopedSpan span = tracer.startScopedSpan("encode");
        try {
            span.tag("input.length", "256");
            span.event("encoding-started");
        } finally {
            span.end();
        }
        assertThat(tracer.finishedSpans()).hasSize(1);
        var finished = tracer.finishedSpans().get(0);
        assertThat(finished.name).isEqualTo("encode");
        assertThat(finished.tags).containsEntry("input.length", "256");
        assertThat(finished.events).containsExactly("encoding-started");
    }

    @Test
    void scopedSpan_shouldRestorePreviousContext_onEnd() {
        TestTracer tracer = new TestTracer();
        assertThat(tracer.currentSpan()).isNull();

        ScopedSpan outer = tracer.startScopedSpan("outer");
        ScopedSpan inner = tracer.startScopedSpan("inner");
        assertThat(inner.context().parentId()).isEqualTo(outer.context().spanId());

        inner.end();
        assertThat(tracer.currentTraceContext().context()).isSameAs(outer.context());

        outer.end();
        assertThat(tracer.currentTraceContext().context()).isNull();
    }

    // ── Span + SpanInScope (advanced pattern) ────────────────

    @Test
    void shouldCreateSpanWithBuilder_andScope() {
        TestTracer tracer = new TestTracer();
        Span span = tracer.nextSpan().name("process").start();
        try (Tracer.SpanInScope ws = tracer.withSpan(span)) {
            span.tag("batch.size", "100");
            assertThat(tracer.currentSpan()).isSameAs(span);
        } finally {
            span.end();
        }
        assertThat(tracer.finishedSpans()).hasSize(1);
    }

    @Test
    void childSpan_shouldInheritTraceId_fromParent() {
        TestTracer tracer = new TestTracer();
        Span parent = tracer.nextSpan().name("parent").start();
        try (Tracer.SpanInScope ws = tracer.withSpan(parent)) {
            Span child = tracer.nextSpan().name("child").start();
            assertThat(child.context().traceId()).isEqualTo(parent.context().traceId());
            assertThat(child.context().parentId()).isEqualTo(parent.context().spanId());
            child.end();
        } finally {
            parent.end();
        }
    }

    // ── Span.Builder ─────────────────────────────────────────

    @Test
    void spanBuilder_shouldConfigureSpanBeforeStart() {
        TestTracer tracer = new TestTracer();
        Span span = tracer.spanBuilder()
                .name("http-request").kind(Span.Kind.CLIENT)
                .tag("http.method", "GET").remoteServiceName("backend")
                .start();
        span.end();
        var finished = tracer.finishedSpans().get(0);
        assertThat(finished.name).isEqualTo("http-request");
        assertThat(finished.kind).isEqualTo(Span.Kind.CLIENT);
        assertThat(finished.remoteServiceName).isEqualTo("backend");
    }

    // ... (20 tests total — see source file for complete listing)
}
```

### `simple-tracing-api/src/test/java/io/simpletracing/TestTracer.java` [NEW]

*Minimal in-memory implementation for testing. See source file for complete listing.*
*This will be replaced by the full `SimpleTracer` in Feature 6.*
