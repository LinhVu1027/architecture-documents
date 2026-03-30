# Chapter 3: Baggage — Propagate Key-Value Pairs Across Service Boundaries

> **What you'll build**: The four baggage interfaces — `BaggageView`, `Baggage`, `BaggageInScope`,
> and `BaggageManager` — that let clients attach key-value metadata to the trace context for
> propagation across service boundaries. You'll also extend `Tracer` to include baggage management.

---

## 1. The API Contract

After this chapter, a client can write:

```java
// Simple pattern — create baggage and put it in scope in one call
Tracer tracer = ...;
try (BaggageInScope scope = tracer.createBaggageInScope("user.id", "12345")) {
    String userId = tracer.getBaggage("user.id").get();  // "12345"
    Map<String, String> all = tracer.getAllBaggage();     // {"user.id": "12345"}
}

// Advanced pattern — create, then scope separately
Baggage baggage = tracer.createBaggage("feature.flag");
try (BaggageInScope scope = baggage.makeCurrent("dark-mode")) {
    String flag = tracer.getBaggage("feature.flag").get();  // "dark-mode"
}
```

**Baggage vs Tags**: Tags are span-local — they annotate a single span and stay there. Baggage
propagates with the trace context across service boundaries via HTTP headers. When you set
`user.id = "12345"` as baggage, every downstream service in the call chain can read it.
Use baggage for cross-cutting concerns (tenant IDs, correlation IDs, feature flags) that all
services need access to.

### Type Hierarchy

```
BaggageView (read-only)       ← name(), get(), get(TraceContext)
    ▲ extends         ▲ extends
    │                 │
  Baggage           BaggageInScope
  (mutable)          (scoped + closeable)
  set(), makeCurrent()    close()

BaggageManager (factory)      ← getBaggage(), createBaggage(), createBaggageInScope()
    ▲ extends
    │
  Tracer                      ← adds span operations (from Feature 1)
```

**Why BaggageInScope does NOT extend Baggage**: Once baggage is in scope, you should only
be able to read it and close the scope. Allowing mutations on a scoped baggage entry would
create confusing semantics — does `set()` on a scoped entry affect the scope? The separation
keeps the contract clean.

---

## 2. Client Usage & Tests

### Test: NOOP cascade for baggage

Just like spans, all baggage types have NOOP constants that form a safe cascade:

```java
@Test
void noopTracer_shouldReturnNoopBaggage() {
    Tracer tracer = Tracer.NOOP;

    Baggage baggage = tracer.getBaggage("user.id");
    assertThat(baggage).isSameAs(Baggage.NOOP);
    assertThat(baggage.name()).isEqualTo("no-op");
    assertThat(baggage.get()).isNull();
}

@Test
void noopTracer_shouldReturnNoopBaggageInScope() {
    Tracer tracer = Tracer.NOOP;

    try (BaggageInScope scope = tracer.createBaggageInScope("user.id", "12345")) {
        assertThat(scope).isSameAs(BaggageInScope.NOOP);
        assertThat(scope.name()).isEqualTo("no-op");
        assertThat(scope.get()).isNull();
    }
}
```

**Why this works**: `Tracer.NOOP` now includes `BaggageManager` methods that return `Baggage.NOOP`,
which in turn returns `BaggageInScope.NOOP` from `makeCurrent()`. The chain never breaks — any
code path through NOOPs stays within NOOPs.

### Test: Create baggage in scope and retrieve it

The simplest real usage — create baggage with a value and read it back:

```java
@Test
void shouldCreateBaggageInScope_andRetrieveValue() {
    TestTracer tracer = new TestTracer();

    try (BaggageInScope scope = tracer.createBaggageInScope("user.id", "12345")) {
        // Readable via the scope handle
        assertThat(scope.get()).isEqualTo("12345");
        assertThat(scope.name()).isEqualTo("user.id");

        // Also readable via the tracer
        Baggage baggage = tracer.getBaggage("user.id");
        assertThat(baggage.get()).isEqualTo("12345");
    }
}
```

**Why this works**: `createBaggageInScope()` is a default method on `BaggageManager` that calls
`createBaggage(name).makeCurrent(value)`. This puts the baggage into the thread-local store,
making it accessible via both the `BaggageInScope` handle and `tracer.getBaggage()`.

### Test: Multiple baggage entries in scope

Baggage entries are independent — you can have multiple in scope simultaneously:

```java
@Test
void shouldReturnAllBaggage_whenMultipleInScope() {
    TestTracer tracer = new TestTracer();

    try (BaggageInScope scope1 = tracer.createBaggageInScope("user.id", "12345")) {
        try (BaggageInScope scope2 = tracer.createBaggageInScope("tenant.id", "acme")) {
            Map<String, String> all = tracer.getAllBaggage();
            assertThat(all).containsEntry("user.id", "12345");
            assertThat(all).containsEntry("tenant.id", "acme");
            assertThat(all).hasSize(2);
        }
    }
}
```

### Test: Baggage cleanup on scope close

When a scope closes, the baggage entry is removed:

```java
@Test
void shouldCleanupBaggage_whenScopeClosed() {
    TestTracer tracer = new TestTracer();

    try (BaggageInScope scope = tracer.createBaggageInScope("user.id", "12345")) {
        assertThat(tracer.getBaggage("user.id").get()).isEqualTo("12345");
    }

    // After scope is closed, baggage should be gone
    assertThat(tracer.getAllBaggage()).isEmpty();
}
```

### Test: Nested baggage restores previous value

When two scopes use the same key, closing the inner scope restores the outer value:

```java
@Test
void nestedBaggage_shouldRestorePreviousValue_whenInnerScopeClosed() {
    TestTracer tracer = new TestTracer();

    try (BaggageInScope outer = tracer.createBaggageInScope("user.id", "outer-value")) {
        assertThat(tracer.getBaggage("user.id").get()).isEqualTo("outer-value");

        try (BaggageInScope inner = tracer.createBaggageInScope("user.id", "inner-value")) {
            assertThat(tracer.getBaggage("user.id").get()).isEqualTo("inner-value");
        }

        // After inner closes, outer value is restored
        assertThat(tracer.getBaggage("user.id").get()).isEqualTo("outer-value");
    }

    assertThat(tracer.getAllBaggage()).isEmpty();
}
```

**Why this works**: The `TestBaggageInScope` captures the previous value at creation time. On
`close()`, it restores the previous value (or removes the key if there was no previous value).
This is exactly how `CurrentTraceContext.Scope` works for span contexts — same pattern, different
data.

---

## 3. Implementing the Call Chain — Top-Down

### Layer 1: BaggageView (the read-only root)

The foundation — three read methods, nothing else:

```java
public interface BaggageView {

    BaggageView NOOP = new BaggageView() {
        @Override
        public String name() { return "no-op"; }

        @Override
        public String get() { return null; }

        @Override
        public String get(TraceContext traceContext) { return null; }
    };

    String name();

    String get();

    String get(TraceContext traceContext);
}
```

**Design note**: `name()` returns the baggage *key* (e.g., `"user.id"`). `get()` returns the
*value* for the current trace context. `get(TraceContext)` allows looking up the value for a
specific context — useful in bridge implementations where different contexts may have different
baggage.

### Layer 2: Baggage (adds mutation)

Extends `BaggageView` with write operations:

```java
public interface Baggage extends BaggageView {

    Baggage NOOP = new Baggage() {
        // ... all methods return this or BaggageInScope.NOOP
    };

    Baggage set(String value);

    Baggage set(TraceContext traceContext, String value);

    BaggageInScope makeCurrent();

    default BaggageInScope makeCurrent(String value) {
        return set(value).makeCurrent();
    }

    default BaggageInScope makeCurrent(TraceContext traceContext, String value) {
        return set(traceContext, value).makeCurrent();
    }
}
```

**Key design decision — default methods for `makeCurrent` overloads**: The two-arg and three-arg
`makeCurrent` methods are defaults that delegate to `set()` + `makeCurrent()`. This means
implementations only need to implement the zero-arg `makeCurrent()` plus the `set()` methods.
The default methods handle the "set then scope" composition automatically.

### Layer 3: BaggageInScope (adds scope lifecycle)

The scoped, closeable variant:

```java
public interface BaggageInScope extends BaggageView, Closeable {

    BaggageInScope NOOP = new BaggageInScope() {
        @Override public String name() { return "no-op"; }
        @Override public String get() { return null; }
        @Override public String get(TraceContext traceContext) { return null; }
        @Override public void close() { }
    };

    @Override
    void close();
}
```

**Why it extends `Closeable` and not `AutoCloseable`**: Same reason as `CurrentTraceContext.Scope` —
`Closeable`'s `close()` doesn't throw checked exceptions, making try-with-resources cleaner.
The `close()` override ensures no `IOException` is declared.

### Layer 4: BaggageManager (the factory)

Factory and lookup interface — `Tracer` extends this:

```java
public interface BaggageManager {

    BaggageManager NOOP = new BaggageManager() {
        // ... returns empty maps and Baggage.NOOP / BaggageInScope.NOOP
    };

    Map<String, String> getAllBaggage();

    default Map<String, String> getAllBaggage(TraceContext traceContext) {
        if (traceContext == null) { return getAllBaggage(); }
        return Collections.emptyMap();
    }

    Baggage getBaggage(String name);

    Baggage getBaggage(TraceContext traceContext, String name);

    Baggage createBaggage(String name);

    Baggage createBaggage(String name, String value);

    default BaggageInScope createBaggageInScope(String name, String value) {
        return createBaggage(name).makeCurrent(value);
    }

    default BaggageInScope createBaggageInScope(TraceContext traceContext, String name, String value) {
        return createBaggage(name).makeCurrent(traceContext, value);
    }
}
```

**Key design decision — `createBaggageInScope` as default methods**: These are convenience
methods that compose `createBaggage()` + `makeCurrent()`. Implementations only need to provide
`createBaggage()` — the default methods handle the rest. This mirrors how the real framework
works, reducing the implementation burden on bridge modules.

### Layer 5: Tracer extends BaggageManager (the connection)

The final piece — making baggage operations accessible through the tracer:

```java
public interface Tracer extends BaggageManager {
    // ... all existing span methods from Feature 1 ...
    // ... inherits all BaggageManager methods ...
}
```

**Why Tracer extends BaggageManager** (instead of having a separate `getBaggageManager()` method):
In practice, every instrumented method that needs baggage already has a `Tracer` reference (for
creating spans). Making `Tracer` extend `BaggageManager` means no additional dependency — you
write `tracer.getBaggage("user.id")` instead of `tracer.getBaggageManager().getBaggage("user.id")`.
This is interface composition at work.

---

## 4. Try It Yourself

1. **Missing baggage lookup**: What happens when you call `tracer.getBaggage("nonexistent")`?
   Write a test to verify. (Hint: check the NOOP cascade)

2. **Baggage with spans**: Create a span, put it in scope, then create baggage inside the span
   scope. Verify both are accessible simultaneously. What happens to the baggage when the span
   scope closes?

3. **Thread isolation**: Create baggage on Thread A. Verify it is NOT visible on Thread B.
   (This tests the thread-local storage model)

---

## 5. Why This Works

### Design insight: Scope-based lifecycle management

Both span scoping (`Tracer.SpanInScope`) and baggage scoping (`BaggageInScope`) follow the
same pattern:

1. **Enter scope** — save previous state, set new state as current
2. **Use** — code within the scope sees the new state
3. **Exit scope** — restore previous state

This is the **RAII pattern** (Resource Acquisition Is Initialization) applied to trace context.
In C++ you'd use destructors; in Java, try-with-resources + `Closeable` achieves the same
guarantee — the previous state is always restored, even if an exception occurs.

### Design insight: NOOP cascade protects production code

The NOOP chain (`Tracer.NOOP` → `Baggage.NOOP` → `BaggageInScope.NOOP`) means instrumented
code never needs null checks:

```java
// This is safe even with NOOP tracer — no NPEs, no special cases
try (BaggageInScope scope = tracer.createBaggageInScope("user.id", "12345")) {
    String userId = tracer.getBaggage("user.id").get();  // returns null for NOOP, never throws
}
```

### Design insight: Interface segregation in the type hierarchy

The three baggage interfaces (`BaggageView`, `Baggage`, `BaggageInScope`) follow the Interface
Segregation Principle — each consumer gets only the methods it needs:

| Consumer | Needs | Interface |
|----------|-------|-----------|
| Downstream code reading baggage | `name()`, `get()` | `BaggageView` |
| Code setting baggage values | `set()`, `makeCurrent()` | `Baggage` |
| Code managing baggage scope | `get()`, `close()` | `BaggageInScope` |

---

## 6. What We Enhanced

| Existing Component | Enhancement | Why This Feature Requires It |
|--------------------|-------------|------------------------------|
| `Tracer` interface | Now extends `BaggageManager` | Baggage operations must be accessible through the tracer |
| `Tracer.NOOP` | Added `BaggageManager` method implementations | NOOP cascade must cover baggage methods |
| `TestTracer` | Added thread-local baggage store + `BaggageManager` impl | Tests need a working baggage implementation |

---

## 7. Connection to Real Framework

| Simplified | Real Framework | File | Key Differences |
|------------|---------------|------|-----------------|
| `BaggageView` | `io.micrometer.tracing.BaggageView` | `micrometer-tracing/src/.../BaggageView.java` | Real uses `@Nullable` annotations |
| `Baggage` | `io.micrometer.tracing.Baggage` | `micrometer-tracing/src/.../Baggage.java` | Real marks `set()` methods as `@Deprecated` in favor of `makeCurrent()` |
| `BaggageInScope` | `io.micrometer.tracing.BaggageInScope` | `micrometer-tracing/src/.../BaggageInScope.java` | Identical structure |
| `BaggageManager` | `io.micrometer.tracing.BaggageManager` | `micrometer-tracing/src/.../BaggageManager.java` | Real marks `createBaggage()` as `@Deprecated`; adds `getBaggageFields()` |
| `Tracer extends BaggageManager` | `io.micrometer.tracing.Tracer extends BaggageManager` | `micrometer-tracing/src/.../Tracer.java` | Same relationship in real framework |

**What we simplified**:
- No `@Deprecated` annotations — the real framework marks `set()` and `createBaggage()` as
  deprecated in favor of `makeCurrent()` and `createBaggageInScope()`. We keep both but don't
  mark deprecation since this is a learning project.
- No `@Nullable` annotations — the real framework uses `org.jspecify.annotations.Nullable`.
  We document nullability in Javadoc instead.
- No `getBaggageFields()` method — returns the list of configured baggage field names. Omitted
  because it requires a configuration concept we haven't built yet.

---

## 8. Complete Code

### `simple-tracing-api/src/main/java/io/simpletracing/BaggageView.java` [NEW]

```java
package io.simpletracing;

public interface BaggageView {

    BaggageView NOOP = new BaggageView() {
        @Override
        public String name() {
            return "no-op";
        }

        @Override
        public String get() {
            return null;
        }

        @Override
        public String get(TraceContext traceContext) {
            return null;
        }
    };

    String name();

    String get();

    String get(TraceContext traceContext);

}
```

### `simple-tracing-api/src/main/java/io/simpletracing/Baggage.java` [NEW]

```java
package io.simpletracing;

public interface Baggage extends BaggageView {

    Baggage NOOP = new Baggage() {
        @Override
        public String name() { return "no-op"; }

        @Override
        public String get() { return null; }

        @Override
        public String get(TraceContext traceContext) { return null; }

        @Override
        public Baggage set(String value) { return this; }

        @Override
        public Baggage set(TraceContext traceContext, String value) { return this; }

        @Override
        public BaggageInScope makeCurrent() { return BaggageInScope.NOOP; }

        @Override
        public BaggageInScope makeCurrent(String value) { return BaggageInScope.NOOP; }

        @Override
        public BaggageInScope makeCurrent(TraceContext traceContext, String value) {
            return BaggageInScope.NOOP;
        }
    };

    Baggage set(String value);

    Baggage set(TraceContext traceContext, String value);

    BaggageInScope makeCurrent();

    default BaggageInScope makeCurrent(String value) {
        return set(value).makeCurrent();
    }

    default BaggageInScope makeCurrent(TraceContext traceContext, String value) {
        return set(traceContext, value).makeCurrent();
    }

}
```

### `simple-tracing-api/src/main/java/io/simpletracing/BaggageInScope.java` [NEW]

```java
package io.simpletracing;

import java.io.Closeable;

public interface BaggageInScope extends BaggageView, Closeable {

    BaggageInScope NOOP = new BaggageInScope() {
        @Override
        public String name() { return "no-op"; }

        @Override
        public String get() { return null; }

        @Override
        public String get(TraceContext traceContext) { return null; }

        @Override
        public void close() { }
    };

    @Override
    void close();

}
```

### `simple-tracing-api/src/main/java/io/simpletracing/BaggageManager.java` [NEW]

```java
package io.simpletracing;

import java.util.Collections;
import java.util.Map;

public interface BaggageManager {

    BaggageManager NOOP = new BaggageManager() {
        @Override
        public Map<String, String> getAllBaggage() { return Collections.emptyMap(); }

        @Override
        public Baggage getBaggage(String name) { return Baggage.NOOP; }

        @Override
        public Baggage getBaggage(TraceContext traceContext, String name) { return Baggage.NOOP; }

        @Override
        public Baggage createBaggage(String name) { return Baggage.NOOP; }

        @Override
        public Baggage createBaggage(String name, String value) { return Baggage.NOOP; }

        @Override
        public BaggageInScope createBaggageInScope(String name, String value) {
            return BaggageInScope.NOOP;
        }

        @Override
        public BaggageInScope createBaggageInScope(TraceContext traceContext, String name, String value) {
            return BaggageInScope.NOOP;
        }
    };

    Map<String, String> getAllBaggage();

    default Map<String, String> getAllBaggage(TraceContext traceContext) {
        if (traceContext == null) { return getAllBaggage(); }
        return Collections.emptyMap();
    }

    Baggage getBaggage(String name);

    Baggage getBaggage(TraceContext traceContext, String name);

    Baggage createBaggage(String name);

    Baggage createBaggage(String name, String value);

    default BaggageInScope createBaggageInScope(String name, String value) {
        return createBaggage(name).makeCurrent(value);
    }

    default BaggageInScope createBaggageInScope(TraceContext traceContext, String name, String value) {
        return createBaggage(name).makeCurrent(traceContext, value);
    }

}
```

### `simple-tracing-api/src/main/java/io/simpletracing/Tracer.java` [MODIFIED]

```java
package io.simpletracing;

import java.io.Closeable;
import java.util.Collections;
import java.util.Map;

public interface Tracer extends BaggageManager {

    Tracer NOOP = new Tracer() {
        @Override
        public Span nextSpan() { return Span.NOOP; }

        @Override
        public Span nextSpan(Span parent) { return Span.NOOP; }

        @Override
        public SpanInScope withSpan(Span span) { return () -> { }; }

        @Override
        public ScopedSpan startScopedSpan(String name) { return ScopedSpan.NOOP; }

        @Override
        public Span.Builder spanBuilder() { return Span.Builder.NOOP; }

        @Override
        public TraceContext.Builder traceContextBuilder() { return TraceContext.Builder.NOOP; }

        @Override
        public CurrentTraceContext currentTraceContext() { return CurrentTraceContext.NOOP; }

        @Override
        public SpanCustomizer currentSpanCustomizer() { return SpanCustomizer.NOOP; }

        @Override
        public Span currentSpan() { return Span.NOOP; }

        // ── BaggageManager methods ──

        @Override
        public Map<String, String> getAllBaggage() { return Collections.emptyMap(); }

        @Override
        public Baggage getBaggage(String name) { return Baggage.NOOP; }

        @Override
        public Baggage getBaggage(TraceContext traceContext, String name) { return Baggage.NOOP; }

        @Override
        public Baggage createBaggage(String name) { return Baggage.NOOP; }

        @Override
        public Baggage createBaggage(String name, String value) { return Baggage.NOOP; }
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

### `simple-tracing-api/src/test/java/io/simpletracing/BaggageApiTest.java` [NEW]

```java
package io.simpletracing;

import org.junit.jupiter.api.Test;
import java.util.Map;
import static org.assertj.core.api.Assertions.assertThat;

class BaggageApiTest {

    // ── NOOP Cascade Tests ────────────────────────────────────

    @Test
    void noopTracer_shouldReturnNoopBaggage() {
        Tracer tracer = Tracer.NOOP;
        Baggage baggage = tracer.getBaggage("user.id");
        assertThat(baggage).isSameAs(Baggage.NOOP);
        assertThat(baggage.name()).isEqualTo("no-op");
        assertThat(baggage.get()).isNull();
    }

    @Test
    void noopTracer_shouldReturnNoopBaggageInScope() {
        try (BaggageInScope scope = Tracer.NOOP.createBaggageInScope("user.id", "12345")) {
            assertThat(scope).isSameAs(BaggageInScope.NOOP);
            assertThat(scope.get()).isNull();
        }
    }

    @Test
    void noopBaggage_fluentMethodsShouldReturnSameInstance() {
        Baggage baggage = Baggage.NOOP;
        assertThat(baggage.set("value")).isSameAs(baggage);
        assertThat(baggage.makeCurrent()).isSameAs(BaggageInScope.NOOP);
        assertThat(baggage.makeCurrent("value")).isSameAs(BaggageInScope.NOOP);
    }

    // ── createBaggageInScope (simple pattern) ─────────────────

    @Test
    void shouldCreateBaggageInScope_andRetrieveValue() {
        TestTracer tracer = new TestTracer();
        try (BaggageInScope scope = tracer.createBaggageInScope("user.id", "12345")) {
            assertThat(scope.get()).isEqualTo("12345");
            assertThat(scope.name()).isEqualTo("user.id");
            assertThat(tracer.getBaggage("user.id").get()).isEqualTo("12345");
        }
    }

    @Test
    void shouldReturnAllBaggage_whenMultipleInScope() {
        TestTracer tracer = new TestTracer();
        try (BaggageInScope s1 = tracer.createBaggageInScope("user.id", "12345")) {
            try (BaggageInScope s2 = tracer.createBaggageInScope("tenant.id", "acme")) {
                Map<String, String> all = tracer.getAllBaggage();
                assertThat(all).containsEntry("user.id", "12345");
                assertThat(all).containsEntry("tenant.id", "acme");
            }
        }
    }

    @Test
    void shouldCleanupBaggage_whenScopeClosed() {
        TestTracer tracer = new TestTracer();
        try (BaggageInScope scope = tracer.createBaggageInScope("user.id", "12345")) {
            assertThat(tracer.getBaggage("user.id").get()).isEqualTo("12345");
        }
        assertThat(tracer.getAllBaggage()).isEmpty();
    }

    @Test
    void nestedBaggage_shouldRestorePreviousValue_whenInnerScopeClosed() {
        TestTracer tracer = new TestTracer();
        try (BaggageInScope outer = tracer.createBaggageInScope("user.id", "outer")) {
            try (BaggageInScope inner = tracer.createBaggageInScope("user.id", "inner")) {
                assertThat(tracer.getBaggage("user.id").get()).isEqualTo("inner");
            }
            assertThat(tracer.getBaggage("user.id").get()).isEqualTo("outer");
        }
        assertThat(tracer.getAllBaggage()).isEmpty();
    }

    // ── Baggage.makeCurrent (advanced pattern) ────────────────

    @Test
    void shouldCreateBaggage_thenMakeCurrent() {
        TestTracer tracer = new TestTracer();
        Baggage baggage = tracer.createBaggage("feature.flag");
        try (BaggageInScope scope = baggage.makeCurrent("dark-mode")) {
            assertThat(tracer.getBaggage("feature.flag").get()).isEqualTo("dark-mode");
        }
        assertThat(tracer.getAllBaggage()).isEmpty();
    }

    // ── Baggage with Spans ────────────────────────────────────

    @Test
    void baggage_shouldBeAccessible_withinSpanScope() {
        TestTracer tracer = new TestTracer();
        Span span = tracer.nextSpan().name("process").start();
        try (Tracer.SpanInScope ws = tracer.withSpan(span)) {
            try (BaggageInScope scope = tracer.createBaggageInScope("req.id", "req-123")) {
                assertThat(tracer.currentSpan()).isSameAs(span);
                assertThat(tracer.getBaggage("req.id").get()).isEqualTo("req-123");
            }
        } finally {
            span.end();
        }
    }

    // ... (15 tests total — see source file for complete listing)
}
```

### `simple-tracing-api/src/test/java/io/simpletracing/TestTracer.java` [MODIFIED]

*Added thread-local baggage store and `BaggageManager` implementation with inner classes
`TestBaggage` and `TestBaggageInScope`. See source file for complete listing.*
