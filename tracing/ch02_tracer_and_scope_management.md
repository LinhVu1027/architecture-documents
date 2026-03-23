# Chapter 2: Tracer & Scope Management

> Line references based on commit `dceab1b9` of the Micrometer Tracing repository.

## Build Challenge

| Current State | Limitation | Objective |
|---------------|------------|-----------|
| We have `Span`, `TraceContext`, `ScopedSpan`, `SpanCustomizer`, and `Link` — the data model for tracing | No way to **create** spans, **scope** them to a thread, or carry **baggage** data alongside the trace | Build the `Tracer` facade (span creation + scope), `CurrentTraceContext` (thread-local scope management), and baggage stub interfaces (`BaggageView`, `Baggage`, `BaggageInScope`, `BaggageManager`) — all with NOOP implementations |

---

## 2.1 The Integration Point: Tracer as the Central Facade

The `Tracer` interface is this feature's integration point — it's the **entry point** that connects the Span/TraceContext data model from Chapter 1 to everything that comes after:

- **Feature 3 (SimpleTracer)** will provide the first concrete `Tracer` implementation
- **Feature 6 (Baggage)** will fill in the `BaggageManager` stub methods with real behavior
- **Feature 8 (TracingObservationHandler)** will use `Tracer` to create spans from observations
- **Feature 10 (Annotation-Based Tracing)** will wrap `Tracer` for declarative `@NewSpan`/`@ContinueSpan`

The type hierarchy we're building:

```
BaggageView (interface)                ← read-only: name(), get()
  ^                                       ^
  | extends                               | extends
  |                                       |
Baggage (interface)                    BaggageInScope (interface + Closeable)
  ← mutable: set(), makeCurrent()          ← scoped: close() removes from scope

BaggageManager (interface)             ← create/retrieve baggage entries
  ^
  | extends
  |
Tracer (interface)                     ← THE CENTRAL FACADE
  |-- nextSpan(), withSpan(), startScopedSpan()
  |-- currentSpan(), currentSpanCustomizer()
  |-- spanBuilder(), traceContextBuilder()
  |-- currentTraceContext() → CurrentTraceContext
  |-- inner interface: SpanInScope (Closeable)

CurrentTraceContext (interface)        ← thread-local scope manager
  |-- context(), newScope(), maybeScope()
  |-- wrap(Runnable/Callable/Executor)
  |-- inner interface: Scope (Closeable)
```

**Direction:** We start with the baggage type hierarchy (stubs needed for `Tracer` to compile), then `CurrentTraceContext` (the scope engine), then `Tracer` itself (the facade that ties everything together).

Two key decisions:

1. **`Tracer extends BaggageManager`** — since baggage is always propagated alongside trace context, the single entry point (`Tracer`) should also manage baggage. This prevents users from needing to discover a separate `BaggageManager`.
2. **Scope is independent of span lifecycle** — `Tracer.withSpan()` and `CurrentTraceContext.newScope()` manage which span/context is "current" on the thread. Closing a scope does NOT end the span. This two-axis design prevents subtle bugs in async code where scope and span ownership may be on different threads.

## 2.2 Baggage Type Hierarchy (Stubs)

These are stub interfaces — they define the types that `BaggageManager` (and therefore `Tracer`) needs to compile. Full baggage implementation comes in Feature 6.

**New file:** `src/main/java/dev/linhvu/tracing/BaggageView.java`

```java
package dev.linhvu.tracing;

public interface BaggageView {

    BaggageView NOOP = new BaggageView() {
        @Override public String name() { return ""; }
        @Override public String get() { return null; }
        @Override public String get(TraceContext traceContext) { return null; }
    };

    String name();
    String get();
    String get(TraceContext traceContext);

}
```

Three read-only methods: the baggage entry's name (key), and two ways to get its value — from the current scope or from a specific trace context.

**New file:** `src/main/java/dev/linhvu/tracing/Baggage.java`

```java
package dev.linhvu.tracing;

public interface Baggage extends BaggageView {

    Baggage NOOP = new Baggage() {
        @Override public String name() { return ""; }
        @Override public String get() { return null; }
        @Override public String get(TraceContext traceContext) { return null; }
        @Override public Baggage set(String value) { return this; }
        @Override public Baggage set(TraceContext traceContext, String value) { return this; }
        @Override public BaggageInScope makeCurrent() { return BaggageInScope.NOOP; }
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

`Baggage` adds mutation (`set()`) and scope management (`makeCurrent()`). The default `makeCurrent(value)` methods chain `set()` → `makeCurrent()` — a convenience that reduces boilerplate.

**New file:** `src/main/java/dev/linhvu/tracing/BaggageInScope.java`

```java
package dev.linhvu.tracing;

import java.io.Closeable;

public interface BaggageInScope extends BaggageView, Closeable {

    BaggageInScope NOOP = new BaggageInScope() {
        @Override public String name() { return ""; }
        @Override public String get() { return null; }
        @Override public String get(TraceContext traceContext) { return null; }
        @Override public void close() { }
    };

    @Override
    void close(); // removes checked exception from Closeable

}
```

`BaggageInScope` extends both `BaggageView` (can read) and `Closeable` (can close to remove from scope). The `close()` override removes the checked `IOException` from `Closeable.close()`.

**New file:** `src/main/java/dev/linhvu/tracing/BaggageManager.java`

```java
package dev.linhvu.tracing;

import java.util.Collections;
import java.util.List;
import java.util.Map;

public interface BaggageManager {

    BaggageManager NOOP = new BaggageManager() {
        @Override public Map<String, String> getAllBaggage() { return Collections.emptyMap(); }
        @Override public Baggage getBaggage(String name) { return Baggage.NOOP; }
        @Override public Baggage getBaggage(TraceContext traceContext, String name) { return Baggage.NOOP; }
        @Override public Baggage createBaggage(String name) { return Baggage.NOOP; }
        @Override public Baggage createBaggage(String name, String value) { return Baggage.NOOP; }
    };

    Map<String, String> getAllBaggage();

    default Map<String, String> getAllBaggage(TraceContext traceContext) {
        return getAllBaggage();
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

    default List<String> getBaggageFields() {
        return Collections.emptyList();
    }

}
```

`BaggageManager` is the aggregation point: CRUD operations for baggage entries plus convenience `createBaggageInScope()` default methods that chain `createBaggage()` → `makeCurrent(value)`.

## 2.3 CurrentTraceContext — The Scope Engine

**New file:** `src/main/java/dev/linhvu/tracing/CurrentTraceContext.java`

```java
package dev.linhvu.tracing;

import java.io.Closeable;
import java.util.concurrent.Callable;
import java.util.concurrent.Executor;

public interface CurrentTraceContext {

    CurrentTraceContext NOOP = new CurrentTraceContext() {
        @Override public TraceContext context() { return TraceContext.NOOP; }
        @Override public Scope newScope(TraceContext context) { return Scope.NOOP; }
        @Override public Scope maybeScope(TraceContext context) { return Scope.NOOP; }
        @Override public <C> Callable<C> wrap(Callable<C> task) { return task; }
        @Override public Runnable wrap(Runnable task) { return task; }
        @Override public Executor wrap(Executor delegate) { return delegate; }
    };

    TraceContext context();
    Scope newScope(TraceContext context);
    Scope maybeScope(TraceContext context);
    <C> Callable<C> wrap(Callable<C> task);
    Runnable wrap(Runnable task);
    Executor wrap(Executor delegate);

    interface Scope extends Closeable {
        Scope NOOP = () -> {};
        @Override void close();
    }

}
```

This is the **scope engine** of the tracing system. It operates at the `TraceContext` level (not `Span` level) and answers: "What trace context is current on this thread?"

Key methods:
- **`context()`** — read what's current
- **`newScope(context)`** — push a new context; returns a `Scope` that restores the previous when closed
- **`maybeScope(context)`** — optimization: returns `Scope.NOOP` if the context is already current
- **`wrap(Runnable/Callable/Executor)`** — context propagation across threads

## 2.4 Tracer — The Central Facade

**New file:** `src/main/java/dev/linhvu/tracing/Tracer.java`

```java
package dev.linhvu.tracing;

import java.io.Closeable;
import java.util.Collections;
import java.util.Map;

public interface Tracer extends BaggageManager {

    Tracer NOOP = new Tracer() {
        @Override public Span nextSpan() { return Span.NOOP; }
        @Override public Span nextSpan(Span parent) { return Span.NOOP; }
        @Override public SpanInScope withSpan(Span span) { return () -> {}; }
        @Override public ScopedSpan startScopedSpan(String name) { return ScopedSpan.NOOP; }
        @Override public Span.Builder spanBuilder() { return Span.Builder.NOOP; }
        @Override public TraceContext.Builder traceContextBuilder() { return TraceContext.Builder.NOOP; }
        @Override public CurrentTraceContext currentTraceContext() { return CurrentTraceContext.NOOP; }
        @Override public SpanCustomizer currentSpanCustomizer() { return SpanCustomizer.NOOP; }
        @Override public Span currentSpan() { return Span.NOOP; }
        // BaggageManager
        @Override public Map<String, String> getAllBaggage() { return Collections.emptyMap(); }
        @Override public Baggage getBaggage(String name) { return Baggage.NOOP; }
        @Override public Baggage getBaggage(TraceContext traceContext, String name) { return Baggage.NOOP; }
        @Override public Baggage createBaggage(String name) { return Baggage.NOOP; }
        @Override public Baggage createBaggage(String name, String value) { return Baggage.NOOP; }
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
        @Override void close();
    }

}
```

The `Tracer` interface supports two usage patterns:

**Pattern 1: Explicit span + scope management**
```java
Span span = tracer.nextSpan().name("my-operation").start();
try (Tracer.SpanInScope ws = tracer.withSpan(span)) {
    // span is now "current" on this thread
    span.tag("result", "success");
} finally {
    span.end(); // must be ended separately!
}
```

**Pattern 2: Convenience scoped span**
```java
ScopedSpan span = tracer.startScopedSpan("my-operation");
try {
    span.tag("result", "success");
} catch (Exception e) {
    span.error(e);
} finally {
    span.end(); // ends span AND removes from scope
}
```

## 2.5 Try It Yourself

<details>
<summary>Challenge: Why are there two scope types — Tracer.SpanInScope and CurrentTraceContext.Scope?</summary>

They operate at different abstraction levels:

- **`CurrentTraceContext.Scope`** operates on `TraceContext` — the raw IDs. This is the lower-level mechanism.
- **`Tracer.SpanInScope`** operates on `Span` — the full span object. Internally, a `Tracer` implementation's `withSpan()` will extract the span's `TraceContext` and call `currentTraceContext().newScope(span.context())`.

The separation lets `CurrentTraceContext` focus purely on thread-local context management without knowing about spans, while `Tracer.SpanInScope` provides a higher-level API that works with the span concept users interact with.

</details>

<details>
<summary>Challenge: What would break if Tracer did NOT extend BaggageManager?</summary>

Nothing would break mechanically — `Tracer` and `BaggageManager` could be independent interfaces, and implementations could implement both. But it would mean users need to inject two separate objects:

```java
// Without: Tracer extends BaggageManager
@Inject Tracer tracer;
@Inject BaggageManager baggageManager; // extra dependency to manage

// With: Tracer extends BaggageManager
@Inject Tracer tracer;
tracer.createBaggageInScope("user-id", "123"); // one dependency does it all
```

Since baggage is always propagated alongside trace context, combining them into one entry point is a pragmatic design choice. The downside is a wider interface, but the NOOP pattern and default methods keep implementations manageable.

</details>

<details>
<summary>Challenge: Implement a recording BaggageManager that tracks which baggage was created</summary>

This tests your understanding of how `BaggageManager.createBaggageInScope()` chains through `createBaggage()` → `Baggage.makeCurrent(value)`:

```java
class RecordingBaggageManager implements BaggageManager {
    String lastCreatedName;

    @Override
    public Map<String, String> getAllBaggage() { return Collections.emptyMap(); }

    @Override
    public Baggage getBaggage(String name) { return Baggage.NOOP; }

    @Override
    public Baggage getBaggage(TraceContext traceContext, String name) { return Baggage.NOOP; }

    @Override
    public Baggage createBaggage(String name) {
        this.lastCreatedName = name;
        return Baggage.NOOP;
    }

    @Override
    public Baggage createBaggage(String name, String value) {
        return createBaggage(name);
    }
}

// Test it:
var mgr = new RecordingBaggageManager();
mgr.createBaggageInScope("user-id", "123"); // uses default method
assert mgr.lastCreatedName.equals("user-id");
```

</details>

## 2.6 Tests

### Unit Tests

**New file:** `src/test/java/dev/linhvu/tracing/BaggageViewTest.java`

```java
@Test
void shouldReturnEmptyName_WhenNoop() {
    assertThat(BaggageView.NOOP.name()).isEmpty();
}

@Test
void shouldReturnNullValue_WhenNoopGet() {
    assertThat(BaggageView.NOOP.get()).isNull();
}
```

**New file:** `src/test/java/dev/linhvu/tracing/BaggageTest.java`

```java
@Test
void shouldReturnSelf_WhenNoopSet() {
    Baggage result = Baggage.NOOP.set("value");
    assertThat(result).isSameAs(Baggage.NOOP);
}

@Test
void shouldDelegateThroughSetAndMakeCurrent_WhenMakeCurrentWithValue() {
    // The default makeCurrent(value) calls set(value).makeCurrent()
    BaggageInScope result = Baggage.NOOP.makeCurrent("value");
    assertThat(result).isSameAs(BaggageInScope.NOOP);
}
```

**New file:** `src/test/java/dev/linhvu/tracing/BaggageInScopeTest.java`

```java
@Test
void shouldDoNothing_WhenNoopClose() {
    assertThatCode(() -> BaggageInScope.NOOP.close()).doesNotThrowAnyException();
}

@Test
void shouldExtendBaggageViewAndCloseable() {
    assertThat(BaggageInScope.NOOP).isInstanceOf(BaggageView.class);
    assertThat(BaggageInScope.NOOP).isInstanceOf(Closeable.class);
}
```

**New file:** `src/test/java/dev/linhvu/tracing/BaggageManagerTest.java`

```java
@Test
void shouldReturnEmptyMap_WhenNoopGetAllBaggage() {
    assertThat(BaggageManager.NOOP.getAllBaggage()).isEmpty();
}

@Test
void shouldVerifyDefaultMethodDelegation_WhenCreateBaggageInScope() {
    // Verify: createBaggageInScope → createBaggage → Baggage → makeCurrent
    var recording = new RecordingBaggageManager();
    recording.createBaggageInScope("user-id", "123");
    assertThat(recording.lastCreatedName).isEqualTo("user-id");
}
```

**New file:** `src/test/java/dev/linhvu/tracing/CurrentTraceContextTest.java`

```java
@Test
void shouldReturnNoopTraceContext_WhenNoopContext() {
    assertThat(CurrentTraceContext.NOOP.context()).isSameAs(TraceContext.NOOP);
}

@Test
void shouldReturnSameRunnable_WhenNoopWrapRunnable() {
    AtomicBoolean ran = new AtomicBoolean(false);
    Runnable original = () -> ran.set(true);
    Runnable wrapped = CurrentTraceContext.NOOP.wrap(original);
    assertThat(wrapped).isSameAs(original);
}
```

**New file:** `src/test/java/dev/linhvu/tracing/TracerTest.java`

```java
@Test
void shouldReturnNoopSpan_WhenNoopNextSpan() {
    assertThat(Tracer.NOOP.nextSpan()).isSameAs(Span.NOOP);
}

@Test
void shouldSupportExplicitSpanAndScopePattern() {
    Span span = Tracer.NOOP.nextSpan().name("operation").start();
    try (Tracer.SpanInScope ws = Tracer.NOOP.withSpan(span)) {
        span.tag("key", "value");
    } finally {
        span.end();
    }
}

@Test
void shouldExtendBaggageManager() {
    assertThat(Tracer.NOOP).isInstanceOf(BaggageManager.class);
}
```

**Run:** `./gradlew test` — expected: all tests pass (ch01 + ch02)

---

## 2.7 Why This Works

> ★ **Insight** -------------------------------------------
> - **Why two layers of scope management?** `CurrentTraceContext` manages scope at the `TraceContext` level (raw IDs), while `Tracer.withSpan()` manages scope at the `Span` level (full object). This separation means `CurrentTraceContext` can be implemented once (e.g., via `ThreadLocal`) and reused across all tracer implementations. The `Tracer` layer adds span-specific behavior (like populating MDC or updating metrics) on top.
> - **Trade-off:** Two scope concepts to learn. But the layering prevents tracer implementations from reinventing thread-local management — they just delegate to `CurrentTraceContext`.
> - **Real-world parallel:** In Spring Boot, `SimpleCurrentTraceContext` (Feature 3) provides the `ThreadLocal`-based implementation, while both the Brave and OpenTelemetry bridge tracers delegate to their own `CurrentTraceContext` implementations with the same interface.
> -----------------------------------------------------------

> ★ **Insight** -------------------------------------------
> - **Why are baggage interfaces defined as stubs in this feature?** Because `Tracer extends BaggageManager`, and `BaggageManager` references `Baggage` and `BaggageInScope` return types. We need these types to exist for `Tracer` to compile, even though meaningful baggage behavior comes in Feature 6. This is a common pattern in large frameworks: define the type contracts early, implement them later.
> - **When to use this pattern:** When you have a central interface (`Tracer`) that aggregates capabilities from multiple feature areas. Define the types up front so the central interface compiles, and fill in behavior incrementally.
> -----------------------------------------------------------

> ★ **Insight** -------------------------------------------
> - **Why `wrap(Runnable/Callable/Executor)` on CurrentTraceContext?** In async code, trace context lives in `ThreadLocal` and doesn't automatically cross thread boundaries. The `wrap` methods capture the current context at wrap time and restore it when the task executes — even on a different thread. Without this, any span started in a thread pool would lose its parent context.
> - **When this matters:** Every time you use `@Async`, `CompletableFuture`, `ExecutorService`, or reactive streams. The `wrap` methods are the mechanism that Spring Boot's auto-configuration uses to make trace context propagation "just work" across thread boundaries.
> -----------------------------------------------------------

## 2.8 What We Enhanced

| File | Change | Why |
|------|--------|-----|
| *(no existing files modified)* | — | This feature adds 6 new interfaces without modifying any Chapter 1 files. The integration point is the `Tracer` interface itself — it references `Span`, `SpanCustomizer`, `ScopedSpan`, and `TraceContext` from Chapter 1 via return types and parameters, creating the dependency link. |

## 2.9 Connection to Real Framework

| Simplified Code | Real Framework Code | File:Line | Key Difference |
|----------------|--------------------|-----------| ---------------|
| `Tracer` | `Tracer` | `Tracer.java:69` | Real version uses `@Nullable` annotations from jspecify; we use plain nulls |
| `Tracer.SpanInScope` | `Tracer.SpanInScope` | `Tracer.java:242` | Identical structure |
| `CurrentTraceContext` | `CurrentTraceContext` | `CurrentTraceContext.java:36` | Real has `wrap(ExecutorService)` in addition to `wrap(Executor)`; we omit `ExecutorService` wrapping for simplicity |
| `CurrentTraceContext.Scope` | `CurrentTraceContext.Scope` | `CurrentTraceContext.java:137` | Identical structure |
| `BaggageManager` | `BaggageManager` | `BaggageManager.java:32` | Real has deprecated `createBaggage` methods; we keep them non-deprecated since they're needed for the stub |
| `BaggageView` | `BaggageView` | `BaggageView.java:28` | Identical structure |
| `Baggage` | `Baggage` | `Baggage.java:33` | Real has deprecated `set()` methods; we keep them non-deprecated |
| `BaggageInScope` | `BaggageInScope` | `BaggageInScope.java:35` | Identical structure |

## 2.10 Complete Code

### Production Code

#### File: `src/main/java/dev/linhvu/tracing/BaggageView.java` [NEW]

```java
package dev.linhvu.tracing;

/**
 * Read-only view of a baggage entry — a key-value pair that propagates alongside
 * trace context across service boundaries. Think of it as a "note taped to the
 * package" that rides along with every leg of the delivery.
 *
 * <p>This is the read-only side; {@link Baggage} extends this with mutation methods.
 * Full implementation comes in Feature 6 — this is the stub interface.
 */
public interface BaggageView {

    /**
     * A no-op baggage view that returns empty/null for all methods.
     */
    BaggageView NOOP = new BaggageView() {
        @Override
        public String name() {
            return "";
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

    /**
     * Returns the baggage entry name (the key).
     */
    String name();

    /**
     * Returns the baggage value from the current scope, or null if not set.
     */
    String get();

    /**
     * Returns the baggage value associated with the given trace context,
     * or null if not set.
     */
    String get(TraceContext traceContext);

}
```

#### File: `src/main/java/dev/linhvu/tracing/Baggage.java` [NEW]

```java
package dev.linhvu.tracing;

/**
 * A mutable baggage entry that can be set and placed into scope.
 * Extends {@link BaggageView} with write operations.
 *
 * <p>Baggage is custom data (like user IDs or request IDs) that propagates alongside
 * trace context across service boundaries. Unlike span tags which are local to a span,
 * baggage travels with the trace across process boundaries.
 *
 * <p>This is a stub interface for Feature 2 — full implementation in Feature 6.
 */
public interface Baggage extends BaggageView {

    /**
     * A no-op baggage that silently ignores all calls.
     */
    Baggage NOOP = new Baggage() {
        @Override
        public String name() {
            return "";
        }

        @Override
        public String get() {
            return null;
        }

        @Override
        public String get(TraceContext traceContext) {
            return null;
        }

        @Override
        public Baggage set(String value) {
            return this;
        }

        @Override
        public Baggage set(TraceContext traceContext, String value) {
            return this;
        }

        @Override
        public BaggageInScope makeCurrent() {
            return BaggageInScope.NOOP;
        }
    };

    /**
     * Sets the baggage value in the current scope.
     */
    Baggage set(String value);

    /**
     * Sets the baggage value for a specific trace context.
     */
    Baggage set(TraceContext traceContext, String value);

    /**
     * Makes this baggage entry "current" — places it in scope so it will be
     * propagated with outgoing requests. Returns a closeable scope handle.
     */
    BaggageInScope makeCurrent();

    /**
     * Convenience: sets the value and makes it current in one call.
     */
    default BaggageInScope makeCurrent(String value) {
        return set(value).makeCurrent();
    }

    /**
     * Convenience: sets the value for a specific trace context and makes it current.
     */
    default BaggageInScope makeCurrent(TraceContext traceContext, String value) {
        return set(traceContext, value).makeCurrent();
    }

}
```

#### File: `src/main/java/dev/linhvu/tracing/BaggageInScope.java` [NEW]

```java
package dev.linhvu.tracing;

import java.io.Closeable;

/**
 * A baggage entry that is "in scope" — it will be propagated with outgoing requests.
 * Extends {@link BaggageView} for reading and {@link Closeable} for scope management.
 *
 * <p>Closing removes the baggage from the current scope. Typical usage:
 * <pre>{@code
 * try (BaggageInScope bis = baggage.makeCurrent("user-123")) {
 *     // baggage is propagated with any outgoing requests here
 * }
 * // baggage is no longer in scope
 * }</pre>
 *
 * <p>This is a stub interface for Feature 2 — full implementation in Feature 6.
 */
public interface BaggageInScope extends BaggageView, Closeable {

    /**
     * A no-op baggage-in-scope that silently ignores all calls.
     */
    BaggageInScope NOOP = new BaggageInScope() {
        @Override
        public String name() {
            return "";
        }

        @Override
        public String get() {
            return null;
        }

        @Override
        public String get(TraceContext traceContext) {
            return null;
        }

        @Override
        public void close() {
        }
    };

    /**
     * Removes this baggage entry from scope. Does not throw checked exceptions
     * (unlike {@link Closeable#close()}).
     */
    @Override
    void close();

}
```

#### File: `src/main/java/dev/linhvu/tracing/BaggageManager.java` [NEW]

```java
package dev.linhvu.tracing;

import java.util.Collections;
import java.util.List;
import java.util.Map;

/**
 * Manages baggage entries — key-value pairs that propagate with trace context.
 * The {@link Tracer} interface extends this, so every tracer is also a baggage manager.
 *
 * <p>This is a stub interface for Feature 2 — full implementation in Feature 6.
 * The methods are defined here so that {@link Tracer} has the complete contract,
 * but meaningful behavior is deferred to the baggage feature.
 */
public interface BaggageManager {

    /**
     * A no-op baggage manager that returns empty/NOOP for all methods.
     */
    BaggageManager NOOP = new BaggageManager() {
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
    };

    /**
     * Returns all baggage key-value pairs from the current scope.
     */
    Map<String, String> getAllBaggage();

    /**
     * Returns all baggage key-value pairs for a specific trace context.
     * Defaults to delegating to {@link #getAllBaggage()}.
     */
    default Map<String, String> getAllBaggage(TraceContext traceContext) {
        return getAllBaggage();
    }

    /**
     * Retrieves a baggage entry by name. Returns a new entry with null value if not found.
     */
    Baggage getBaggage(String name);

    /**
     * Retrieves a baggage entry by name for a specific trace context.
     */
    Baggage getBaggage(TraceContext traceContext, String name);

    /**
     * Creates or returns an existing baggage entry with the given name.
     */
    Baggage createBaggage(String name);

    /**
     * Creates or returns an existing baggage entry with the given name and value.
     */
    Baggage createBaggage(String name, String value);

    /**
     * Creates baggage, sets its value, and places it in scope in one call.
     */
    default BaggageInScope createBaggageInScope(String name, String value) {
        return createBaggage(name).makeCurrent(value);
    }

    /**
     * Creates baggage, sets its value for a specific trace context, and places it in scope.
     */
    default BaggageInScope createBaggageInScope(TraceContext traceContext, String name, String value) {
        return createBaggage(name).makeCurrent(traceContext, value);
    }

    /**
     * Returns all registered baggage field names.
     */
    default List<String> getBaggageFields() {
        return Collections.emptyList();
    }

}
```

#### File: `src/main/java/dev/linhvu/tracing/CurrentTraceContext.java` [NEW]

```java
package dev.linhvu.tracing;

import java.io.Closeable;
import java.util.concurrent.Callable;
import java.util.concurrent.Executor;

/**
 * Manages which {@link TraceContext} is "current" on the calling thread.
 * This is the lower-level scope manager — {@link Tracer#withSpan(Span)} delegates to this.
 *
 * <p>Real-world analogy: if TraceContext is the tracking number on a package, then
 * CurrentTraceContext is the clipboard at the sorting station that shows which
 * package the worker is currently handling.
 *
 * <p>Key principle: scope is <strong>independent of span lifecycle</strong>.
 * Closing a scope does NOT end the span; ending a span does NOT close the scope.
 * Both must be managed explicitly.
 *
 * <p>Usage:
 * <pre>{@code
 * CurrentTraceContext.Scope scope = currentTraceContext.newScope(traceContext);
 * try {
 *     // traceContext is now "current" on this thread
 *     assert currentTraceContext.context() == traceContext;
 * } finally {
 *     scope.close(); // restores the previous context
 * }
 * }</pre>
 */
public interface CurrentTraceContext {

    /**
     * A no-op implementation that always returns NOOP context and passthrough wrappers.
     */
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
    };

    /**
     * Returns the current {@link TraceContext} in scope, or null if none.
     */
    TraceContext context();

    /**
     * Sets the given context as current and returns a {@link Scope} that must be closed
     * to restore the previous context. Passing null clears the current context.
     */
    Scope newScope(TraceContext context);

    /**
     * Like {@link #newScope(TraceContext)}, but returns a no-op scope if the given
     * context is already in scope. This is an optimization to avoid redundant scope
     * creation when the context hasn't changed.
     */
    Scope maybeScope(TraceContext context);

    /**
     * Wraps a {@link Callable} so it executes with the current trace context,
     * even if run on a different thread.
     */
    <C> Callable<C> wrap(Callable<C> task);

    /**
     * Wraps a {@link Runnable} so it executes with the current trace context,
     * even if run on a different thread.
     */
    Runnable wrap(Runnable task);

    /**
     * Wraps an {@link Executor} so that submitted tasks inherit the current trace context.
     */
    Executor wrap(Executor delegate);

    /**
     * A scope handle that must be closed to restore the previous trace context.
     * Implements {@link Closeable} for use in try-with-resources.
     */
    interface Scope extends Closeable {

        /**
         * A no-op scope whose close() does nothing.
         */
        Scope NOOP = () -> {
        };

        /**
         * Restores the previous trace context. Does not throw checked exceptions.
         */
        @Override
        void close();

    }

}
```

#### File: `src/main/java/dev/linhvu/tracing/Tracer.java` [NEW]

```java
package dev.linhvu.tracing;

import java.io.Closeable;
import java.util.Collections;
import java.util.Map;

/**
 * The central tracing facade — the primary entry point for creating and managing spans.
 * Extends {@link BaggageManager} so every tracer can also manage baggage.
 *
 * <p>Real-world analogy: the logistics coordinator who creates new delivery legs,
 * knows which leg is currently being handled, and can look up tracking numbers.
 *
 * <p>Two usage patterns are supported:
 *
 * <p><strong>Pattern 1: Explicit span + scope management</strong>
 * <pre>{@code
 * Span span = tracer.nextSpan().name("my-operation").start();
 * try (Tracer.SpanInScope ws = tracer.withSpan(span)) {
 *     // span is now the "current" span on this thread
 *     span.tag("result", "success");
 * } finally {
 *     span.end();
 * }
 * }</pre>
 *
 * <p><strong>Pattern 2: Convenience scoped span</strong>
 * <pre>{@code
 * ScopedSpan span = tracer.startScopedSpan("my-operation");
 * try {
 *     span.tag("result", "success");
 * } catch (Exception e) {
 *     span.error(e);
 * } finally {
 *     span.end(); // ends span AND removes from scope
 * }
 * }</pre>
 */
public interface Tracer extends BaggageManager {

    /**
     * A no-op tracer that returns NOOP objects for all methods.
     */
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

        // BaggageManager delegation
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
    };

    /**
     * Creates a new span as a child of the current span in scope, or a new root trace
     * if no span is in scope. The span is NOT started — call {@link Span#start()}.
     */
    Span nextSpan();

    /**
     * Creates a new span as a child of the given parent span.
     * Falls back to {@link #nextSpan()} if parent is null.
     */
    Span nextSpan(Span parent);

    /**
     * Places the given span into scope on the current thread. Returns a
     * {@link SpanInScope} that must be closed to restore the previous scope.
     *
     * <p><strong>Important:</strong> closing the scope does NOT end the span.
     * The span must be ended separately via {@link Span#end()}.
     */
    SpanInScope withSpan(Span span);

    /**
     * Creates a new span, starts it, and places it in scope — all in one call.
     * The returned {@link ScopedSpan} ends and removes from scope when
     * {@link ScopedSpan#end()} is called.
     */
    ScopedSpan startScopedSpan(String name);

    /**
     * Returns a builder for constructing a span with deferred configuration.
     */
    Span.Builder spanBuilder();

    /**
     * Returns a builder for constructing a {@link TraceContext}.
     */
    TraceContext.Builder traceContextBuilder();

    /**
     * Returns the {@link CurrentTraceContext} — the lower-level scope manager
     * that tracks which TraceContext is current on the calling thread.
     */
    CurrentTraceContext currentTraceContext();

    /**
     * Returns a {@link SpanCustomizer} for the current span in scope, or null
     * if no span is in scope. Use this when you only need to add tags/events
     * without access to lifecycle methods.
     */
    SpanCustomizer currentSpanCustomizer();

    /**
     * Returns the current span in scope, or null if none.
     */
    Span currentSpan();

    /**
     * A scope handle for a span placed into scope via {@link #withSpan(Span)}.
     * Must be closed to restore the previous scope.
     *
     * <p>Implements {@link Closeable} for try-with-resources usage:
     * <pre>{@code
     * try (Tracer.SpanInScope ws = tracer.withSpan(span)) {
     *     // span is current
     * }
     * // previous span restored
     * }</pre>
     */
    interface SpanInScope extends Closeable {

        /**
         * Restores the previous span scope. Does not throw checked exceptions.
         */
        @Override
        void close();

    }

}
```

### Test Code

#### File: `src/test/java/dev/linhvu/tracing/BaggageViewTest.java` [NEW]

```java
package dev.linhvu.tracing;

import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;

class BaggageViewTest {

    @Test
    void shouldReturnEmptyName_WhenNoop() {
        assertThat(BaggageView.NOOP.name()).isEmpty();
    }

    @Test
    void shouldReturnNullValue_WhenNoopGet() {
        assertThat(BaggageView.NOOP.get()).isNull();
    }

    @Test
    void shouldReturnNullValue_WhenNoopGetWithTraceContext() {
        assertThat(BaggageView.NOOP.get(TraceContext.NOOP)).isNull();
    }

}
```

#### File: `src/test/java/dev/linhvu/tracing/BaggageTest.java` [NEW]

```java
package dev.linhvu.tracing;

import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;

class BaggageTest {

    @Test
    void shouldReturnSelf_WhenNoopSet() {
        Baggage result = Baggage.NOOP.set("value");
        assertThat(result).isSameAs(Baggage.NOOP);
    }

    @Test
    void shouldReturnSelf_WhenNoopSetWithTraceContext() {
        Baggage result = Baggage.NOOP.set(TraceContext.NOOP, "value");
        assertThat(result).isSameAs(Baggage.NOOP);
    }

    @Test
    void shouldReturnNoopBaggageInScope_WhenNoopMakeCurrent() {
        BaggageInScope result = Baggage.NOOP.makeCurrent();
        assertThat(result).isSameAs(BaggageInScope.NOOP);
    }

    @Test
    void shouldDelegateThroughSetAndMakeCurrent_WhenMakeCurrentWithValue() {
        // The default makeCurrent(value) calls set(value).makeCurrent()
        // On NOOP, this chains through and returns BaggageInScope.NOOP
        BaggageInScope result = Baggage.NOOP.makeCurrent("value");
        assertThat(result).isSameAs(BaggageInScope.NOOP);
    }

    @Test
    void shouldDelegateThroughSetAndMakeCurrent_WhenMakeCurrentWithTraceContextAndValue() {
        BaggageInScope result = Baggage.NOOP.makeCurrent(TraceContext.NOOP, "value");
        assertThat(result).isSameAs(BaggageInScope.NOOP);
    }

    @Test
    void shouldExtendBaggageView() {
        // Baggage IS-A BaggageView
        assertThat(Baggage.NOOP).isInstanceOf(BaggageView.class);
    }

    @Test
    void shouldReturnEmptyNameAndNullValue_WhenNoopRead() {
        assertThat(Baggage.NOOP.name()).isEmpty();
        assertThat(Baggage.NOOP.get()).isNull();
        assertThat(Baggage.NOOP.get(TraceContext.NOOP)).isNull();
    }

}
```

#### File: `src/test/java/dev/linhvu/tracing/BaggageInScopeTest.java` [NEW]

```java
package dev.linhvu.tracing;

import java.io.Closeable;

import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatCode;

class BaggageInScopeTest {

    @Test
    void shouldDoNothing_WhenNoopClose() {
        assertThatCode(() -> BaggageInScope.NOOP.close()).doesNotThrowAnyException();
    }

    @Test
    void shouldExtendBaggageViewAndCloseable() {
        assertThat(BaggageInScope.NOOP).isInstanceOf(BaggageView.class);
        assertThat(BaggageInScope.NOOP).isInstanceOf(Closeable.class);
    }

    @Test
    void shouldReturnEmptyNameAndNullValue_WhenNoopRead() {
        assertThat(BaggageInScope.NOOP.name()).isEmpty();
        assertThat(BaggageInScope.NOOP.get()).isNull();
        assertThat(BaggageInScope.NOOP.get(TraceContext.NOOP)).isNull();
    }

}
```

#### File: `src/test/java/dev/linhvu/tracing/BaggageManagerTest.java` [NEW]

```java
package dev.linhvu.tracing;

import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;

class BaggageManagerTest {

    @Test
    void shouldReturnEmptyMap_WhenNoopGetAllBaggage() {
        assertThat(BaggageManager.NOOP.getAllBaggage()).isEmpty();
    }

    @Test
    void shouldDelegateToGetAllBaggage_WhenGetAllBaggageWithTraceContext() {
        // Default method delegates to getAllBaggage()
        assertThat(BaggageManager.NOOP.getAllBaggage(TraceContext.NOOP)).isEmpty();
    }

    @Test
    void shouldReturnNoopBaggage_WhenNoopGetBaggage() {
        assertThat(BaggageManager.NOOP.getBaggage("key")).isSameAs(Baggage.NOOP);
    }

    @Test
    void shouldReturnNoopBaggage_WhenNoopGetBaggageWithTraceContext() {
        assertThat(BaggageManager.NOOP.getBaggage(TraceContext.NOOP, "key")).isSameAs(Baggage.NOOP);
    }

    @Test
    void shouldReturnNoopBaggage_WhenNoopCreateBaggage() {
        assertThat(BaggageManager.NOOP.createBaggage("key")).isSameAs(Baggage.NOOP);
    }

    @Test
    void shouldReturnNoopBaggage_WhenNoopCreateBaggageWithValue() {
        assertThat(BaggageManager.NOOP.createBaggage("key", "value")).isSameAs(Baggage.NOOP);
    }

    @Test
    void shouldReturnNoopBaggageInScope_WhenCreateBaggageInScope() {
        // Default method: createBaggage(name).makeCurrent(value)
        BaggageInScope result = BaggageManager.NOOP.createBaggageInScope("key", "value");
        assertThat(result).isSameAs(BaggageInScope.NOOP);
    }

    @Test
    void shouldReturnNoopBaggageInScope_WhenCreateBaggageInScopeWithTraceContext() {
        BaggageInScope result = BaggageManager.NOOP.createBaggageInScope(TraceContext.NOOP, "key", "value");
        assertThat(result).isSameAs(BaggageInScope.NOOP);
    }

    @Test
    void shouldReturnEmptyList_WhenNoopGetBaggageFields() {
        assertThat(BaggageManager.NOOP.getBaggageFields()).isEmpty();
    }

    @Test
    void shouldVerifyDefaultMethodDelegation_WhenCreateBaggageInScope() {
        // Verify the default createBaggageInScope chains:
        // createBaggage(name) -> Baggage -> makeCurrent(value) -> BaggageInScope
        // This tests the wiring between BaggageManager and Baggage default methods
        var recording = new RecordingBaggageManager();
        BaggageInScope result = recording.createBaggageInScope("user-id", "123");

        assertThat(recording.lastCreatedName).isEqualTo("user-id");
        assertThat(recording.lastBaggage.lastSetValue).isEqualTo("123");
        assertThat(result).isNotNull();
    }

    // --- Recording implementations to verify default method delegation ---

    static class RecordingBaggage implements Baggage {
        String lastSetValue;

        @Override
        public String name() {
            return "recording";
        }

        @Override
        public String get() {
            return null;
        }

        @Override
        public String get(TraceContext traceContext) {
            return null;
        }

        @Override
        public Baggage set(String value) {
            this.lastSetValue = value;
            return this;
        }

        @Override
        public Baggage set(TraceContext traceContext, String value) {
            this.lastSetValue = value;
            return this;
        }

        @Override
        public BaggageInScope makeCurrent() {
            return BaggageInScope.NOOP;
        }
    }

    static class RecordingBaggageManager implements BaggageManager {
        String lastCreatedName;
        RecordingBaggage lastBaggage;

        @Override
        public java.util.Map<String, String> getAllBaggage() {
            return java.util.Collections.emptyMap();
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
            this.lastCreatedName = name;
            this.lastBaggage = new RecordingBaggage();
            return lastBaggage;
        }

        @Override
        public Baggage createBaggage(String name, String value) {
            return createBaggage(name);
        }
    }

}
```

#### File: `src/test/java/dev/linhvu/tracing/CurrentTraceContextTest.java` [NEW]

```java
package dev.linhvu.tracing;

import java.util.concurrent.Callable;
import java.util.concurrent.Executor;
import java.util.concurrent.atomic.AtomicBoolean;

import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatCode;

class CurrentTraceContextTest {

    @Test
    void shouldReturnNoopTraceContext_WhenNoopContext() {
        assertThat(CurrentTraceContext.NOOP.context()).isSameAs(TraceContext.NOOP);
    }

    @Test
    void shouldReturnNoopScope_WhenNoopNewScope() {
        CurrentTraceContext.Scope scope = CurrentTraceContext.NOOP.newScope(TraceContext.NOOP);
        assertThatCode(scope::close).doesNotThrowAnyException();
    }

    @Test
    void shouldReturnNoopScope_WhenNoopMaybeScope() {
        CurrentTraceContext.Scope scope = CurrentTraceContext.NOOP.maybeScope(TraceContext.NOOP);
        assertThatCode(scope::close).doesNotThrowAnyException();
    }

    @Test
    void shouldReturnSameCallable_WhenNoopWrapCallable() throws Exception {
        Callable<String> original = () -> "hello";
        Callable<String> wrapped = CurrentTraceContext.NOOP.wrap(original);

        assertThat(wrapped).isSameAs(original);
        assertThat(wrapped.call()).isEqualTo("hello");
    }

    @Test
    void shouldReturnSameRunnable_WhenNoopWrapRunnable() {
        AtomicBoolean ran = new AtomicBoolean(false);
        Runnable original = () -> ran.set(true);
        Runnable wrapped = CurrentTraceContext.NOOP.wrap(original);

        assertThat(wrapped).isSameAs(original);
        wrapped.run();
        assertThat(ran).isTrue();
    }

    @Test
    void shouldReturnSameExecutor_WhenNoopWrapExecutor() {
        Executor original = Runnable::run;
        Executor wrapped = CurrentTraceContext.NOOP.wrap(original);

        assertThat(wrapped).isSameAs(original);
    }

    @Test
    void shouldDoNothing_WhenNoopScopeClose() {
        assertThatCode(() -> CurrentTraceContext.Scope.NOOP.close()).doesNotThrowAnyException();
    }

    @Test
    void shouldBeCloseable_WhenScope() {
        // Scope extends Closeable — can be used in try-with-resources
        assertThat(CurrentTraceContext.Scope.NOOP).isInstanceOf(java.io.Closeable.class);
    }

}
```

#### File: `src/test/java/dev/linhvu/tracing/TracerTest.java` [NEW]

```java
package dev.linhvu.tracing;

import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatCode;

class TracerTest {

    // --- Span creation ---

    @Test
    void shouldReturnNoopSpan_WhenNoopNextSpan() {
        assertThat(Tracer.NOOP.nextSpan()).isSameAs(Span.NOOP);
    }

    @Test
    void shouldReturnNoopSpan_WhenNoopNextSpanWithParent() {
        assertThat(Tracer.NOOP.nextSpan(Span.NOOP)).isSameAs(Span.NOOP);
    }

    @Test
    void shouldReturnNoopScopedSpan_WhenNoopStartScopedSpan() {
        assertThat(Tracer.NOOP.startScopedSpan("test")).isSameAs(ScopedSpan.NOOP);
    }

    // --- Scope management ---

    @Test
    void shouldReturnCloseableSpanInScope_WhenNoopWithSpan() {
        Tracer.SpanInScope scope = Tracer.NOOP.withSpan(Span.NOOP);
        assertThat(scope).isNotNull();
        assertThatCode(scope::close).doesNotThrowAnyException();
    }

    @Test
    void shouldBeCloseable_WhenSpanInScope() {
        assertThat(Tracer.NOOP.withSpan(Span.NOOP)).isInstanceOf(java.io.Closeable.class);
    }

    // --- Current span ---

    @Test
    void shouldReturnNoopSpan_WhenNoopCurrentSpan() {
        assertThat(Tracer.NOOP.currentSpan()).isSameAs(Span.NOOP);
    }

    @Test
    void shouldReturnNoopSpanCustomizer_WhenNoopCurrentSpanCustomizer() {
        assertThat(Tracer.NOOP.currentSpanCustomizer()).isSameAs(SpanCustomizer.NOOP);
    }

    // --- Builders ---

    @Test
    void shouldReturnNoopSpanBuilder_WhenNoopSpanBuilder() {
        assertThat(Tracer.NOOP.spanBuilder()).isSameAs(Span.Builder.NOOP);
    }

    @Test
    void shouldReturnNoopTraceContextBuilder_WhenNoopTraceContextBuilder() {
        assertThat(Tracer.NOOP.traceContextBuilder()).isSameAs(TraceContext.Builder.NOOP);
    }

    // --- CurrentTraceContext ---

    @Test
    void shouldReturnNoopCurrentTraceContext_WhenNoopCurrentTraceContext() {
        assertThat(Tracer.NOOP.currentTraceContext()).isSameAs(CurrentTraceContext.NOOP);
    }

    // --- BaggageManager methods inherited ---

    @Test
    void shouldReturnEmptyMap_WhenNoopGetAllBaggage() {
        assertThat(Tracer.NOOP.getAllBaggage()).isEmpty();
    }

    @Test
    void shouldReturnNoopBaggage_WhenNoopGetBaggage() {
        assertThat(Tracer.NOOP.getBaggage("key")).isSameAs(Baggage.NOOP);
    }

    @Test
    void shouldReturnNoopBaggage_WhenNoopGetBaggageWithTraceContext() {
        assertThat(Tracer.NOOP.getBaggage(TraceContext.NOOP, "key")).isSameAs(Baggage.NOOP);
    }

    @Test
    void shouldReturnNoopBaggage_WhenNoopCreateBaggage() {
        assertThat(Tracer.NOOP.createBaggage("key")).isSameAs(Baggage.NOOP);
    }

    @Test
    void shouldReturnNoopBaggage_WhenNoopCreateBaggageWithValue() {
        assertThat(Tracer.NOOP.createBaggage("key", "value")).isSameAs(Baggage.NOOP);
    }

    @Test
    void shouldExtendBaggageManager() {
        // Tracer IS-A BaggageManager
        assertThat(Tracer.NOOP).isInstanceOf(BaggageManager.class);
    }

    // --- Two usage patterns documented in Javadoc ---

    @Test
    void shouldSupportExplicitSpanAndScopePattern() {
        // Pattern 1: create span, scope separately, end separately
        Span span = Tracer.NOOP.nextSpan().name("operation").start();
        try (Tracer.SpanInScope ws = Tracer.NOOP.withSpan(span)) {
            span.tag("key", "value");
        } finally {
            span.end();
        }
        // No exception = success
    }

    @Test
    void shouldSupportConvenienceScopedSpanPattern() {
        // Pattern 2: startScopedSpan combines create + start + scope
        ScopedSpan span = Tracer.NOOP.startScopedSpan("operation");
        try {
            span.tag("key", "value");
        } catch (Exception e) {
            span.error(e);
        } finally {
            span.end();
        }
        // No exception = success
    }

}
```

---

## Summary

| Concept | What It Means |
|---------|---------------|
| **Tracer** | Central facade — creates spans, manages scope, provides access to current context. Extends BaggageManager. |
| **CurrentTraceContext** | Lower-level scope manager — tracks which TraceContext is current on the calling thread via `newScope()`/`maybeScope()` |
| **CurrentTraceContext.Scope** | Closeable handle that restores the previous trace context when closed |
| **Tracer.SpanInScope** | Closeable handle for span-level scope — wraps CurrentTraceContext.Scope at a higher level |
| **BaggageView** | Read-only view of a baggage entry: name, get value |
| **Baggage** | Mutable baggage: extends BaggageView with set() and makeCurrent() |
| **BaggageInScope** | Scoped baggage: extends BaggageView + Closeable, close removes from scope |
| **BaggageManager** | CRUD for baggage entries — Tracer extends this, so every tracer manages baggage |
| **Scope independence** | Closing a scope does NOT end the span; ending a span does NOT close the scope — both managed separately |
| **wrap() methods** | Capture current trace context and restore it when a Runnable/Callable/Executor runs on a different thread |

**Next: Chapter 3 — Simple Tracer (In-Memory Implementation)** — the first concrete `Tracer` that creates real spans with IDs, manages scope via `ThreadLocal`, and collects finished spans for inspection
