# Chapter 12: Cross-Thread Context Propagation

> Line references based on commit `dceab1b9` of the Micrometer Tracing repository.

## Build Challenge

| Current State | Limitation | Objective |
|---------------|------------|-----------|
| Spans and baggage are tracked per-thread via `ThreadLocal` in `SimpleTracer`; the context-propagation library (simple-context-propagation) provides `ThreadLocalAccessor`, `ContextSnapshot`, and `ContextExecutorService` | When work hops to a different thread (thread pool, `CompletableFuture`, reactive scheduler), the current span and baggage are lost — the worker thread has an empty `ThreadLocal` | Build `ThreadLocalAccessor` implementations that capture spans and baggage, restore them on worker threads, and coordinate with `ObservationThreadLocalAccessor` to avoid duplicating span scopes |

---

## 12.1 The Integration Point

The integration point is the `ContextRegistry` — where our new `ThreadLocalAccessor` implementations register themselves alongside the existing `ObservationThreadLocalAccessor` (OTLA). Once registered, `ContextSnapshot.captureAll()` and `ContextExecutorService` automatically propagate spans and baggage across thread boundaries.

```java
// Registration — the point where tracing connects to context-propagation
ContextRegistry registry = ContextRegistry.getInstance();

// OTLA must be registered first (from simple-micrometer)
registry.registerThreadLocalAccessor(new ObservationThreadLocalAccessor(observationRegistry));

// Our new accessors — registered AFTER OTLA
registry.registerThreadLocalAccessor(
        new ObservationAwareSpanThreadLocalAccessor(observationRegistry, tracer));
registry.registerThreadLocalAccessor(
        new ObservationAwareBaggageThreadLocalAccessor(observationRegistry, tracer));
```

Two key decisions here:

1. **Registration order matters.** OTLA must be registered before the span/baggage accessors because `ContextSnapshot.setThreadLocals()` processes accessors in registration order. OTLA needs to restore the observation (and its span via `TracingObservationHandler`) before the span accessor checks whether to act.

2. **Observation awareness.** Both accessors inspect `ObservationRegistry.getCurrentObservation()` and the `TracingContext` to determine if OTLA has already handled the current span. Without this coordination, both OTLA and the span accessor would try to scope the same span, creating a duplicate scope that breaks the cleanup chain.

This connects the **Tracing API** (spans, baggage) to the **Context Propagation API** (ThreadLocalAccessor, ContextSnapshot). To make it work, we need to build:
- `BaggageToPropagate` — value type wrapping baggage entries for snapshot storage
- `ObservationAwareSpanThreadLocalAccessor` — captures/restores spans, yields when OTLA manages the span
- `ObservationAwareBaggageThreadLocalAccessor` — captures/restores baggage entries

**Modifying:** `build.gradle`
**Change:** Add dependency on `simple-context-propagation`

```java
dependencies {
    implementation 'dev.linhvu:simple-micrometer:1.0-SNAPSHOT'
    implementation 'dev.linhvu:simple-context-propagation:1.0-SNAPSHOT'
    implementation 'org.assertj:assertj-core:3.27.3'
    // ...
}
```

## 12.2 BaggageToPropagate — The Baggage Snapshot Value Type

**New file:** `src/main/java/dev/linhvu/tracing/contextpropagation/BaggageToPropagate.java`

```java
package dev.linhvu.tracing.contextpropagation;

import java.util.HashMap;
import java.util.Map;
import java.util.Objects;

public class BaggageToPropagate {

    private final Map<String, String> baggage;

    public BaggageToPropagate(Map<String, String> baggage) {
        this.baggage = new HashMap<>(baggage);
    }

    public BaggageToPropagate(String... baggage) {
        if (baggage.length % 2 != 0) {
            throw new IllegalArgumentException(
                    "Baggage must be provided as key-value pairs, got odd count: "
                            + baggage.length);
        }
        this.baggage = new HashMap<>();
        for (int i = 0; i < baggage.length; i += 2) {
            this.baggage.put(baggage[i], baggage[i + 1]);
        }
    }

    public Map<String, String> getBaggage() {
        return baggage;
    }

    @Override
    public boolean equals(Object o) { /* ... */ }

    @Override
    public int hashCode() { /* ... */ }
}
```

The map is defensively copied on construction — mutations to the original map do not affect the snapshot. This is the generic type parameter for `ObservationAwareBaggageThreadLocalAccessor`.

## 12.3 ObservationAwareSpanThreadLocalAccessor — Span Propagation

**New file:** `src/main/java/dev/linhvu/tracing/contextpropagation/ObservationAwareSpanThreadLocalAccessor.java`

```java
package dev.linhvu.tracing.contextpropagation;

import dev.linhvu.context.ThreadLocalAccessor;
import dev.linhvu.micrometer.observation.Observation;
import dev.linhvu.micrometer.observation.ObservationRegistry;
import dev.linhvu.tracing.Span;
import dev.linhvu.tracing.Tracer;
import dev.linhvu.tracing.handler.TracingObservationHandler;

import java.io.Closeable;
import java.util.Map;
import java.util.Objects;
import java.util.concurrent.ConcurrentHashMap;

public class ObservationAwareSpanThreadLocalAccessor implements ThreadLocalAccessor<Span> {

    public static final String KEY = "micrometer.tracing";

    final Map<Thread, SpanAction> spanActions = new ConcurrentHashMap<>();

    private final ObservationRegistry registry;
    private final Tracer tracer;

    // ...

    @Override
    public Span getValue() {
        Observation currentObservation = registry.getCurrentObservation();
        Span currentSpan = tracer.currentSpan();

        if (currentObservation != null) {
            TracingObservationHandler.TracingContext tracingContext =
                    currentObservation.getContext().get(
                            TracingObservationHandler.TracingContext.class);
            if (tracingContext != null) {
                Span observationSpan = tracingContext.getSpan();
                if (currentSpan != null && !currentSpan.equals(observationSpan)) {
                    return currentSpan; // Manual child — propagate it
                }
                return null; // OTLA manages this span — yield
            }
        }
        return currentSpan; // No observation — we own the span
    }

    @Override
    public void setValue(Span value) {
        SpanAction previous = spanActions.get(Thread.currentThread());
        Tracer.SpanInScope scope = tracer.withSpan(value);
        SpanAction action = new SpanAction(previous, spanActions);
        spanActions.put(Thread.currentThread(), action);
        action.setScope(scope);
    }

    @Override
    public void restore(Span previousValue) {
        SpanAction spanAction = spanActions.get(Thread.currentThread());
        if (spanAction == null) return;
        spanAction.close(); // Pop the linked list
    }
}
```

The `getValue()` decision tree is the heart of the coordination logic:

```
getValue()
    │
    ├── Observation active?
    │   ├── YES → TracingContext has span?
    │   │         ├── YES → currentSpan == observationSpan?
    │   │         │         ├── YES → return null (OTLA handles it)
    │   │         │         └── NO  → return currentSpan (manual child)
    │   │         └── NO  → return currentSpan
    │   └── NO  → return currentSpan
```

The `SpanAction` inner class forms a **per-thread linked list** for nested scope management:

```java
static class SpanAction implements AutoCloseable {
    private final SpanAction previous;
    private final Map<Thread, SpanAction> todo;
    private Closeable scope;

    @Override
    public void close() {
        if (scope != null) { scope.close(); }
        if (previous != null) {
            todo.put(Thread.currentThread(), previous); // Pop to previous
        } else {
            todo.remove(Thread.currentThread()); // Stack empty — remove entry
        }
    }
}
```

## 12.4 ObservationAwareBaggageThreadLocalAccessor — Baggage Propagation

**New file:** `src/main/java/dev/linhvu/tracing/contextpropagation/ObservationAwareBaggageThreadLocalAccessor.java`

```java
package dev.linhvu.tracing.contextpropagation;

import dev.linhvu.context.ThreadLocalAccessor;
import dev.linhvu.tracing.BaggageInScope;
import dev.linhvu.tracing.Span;
import dev.linhvu.tracing.Tracer;

import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;
import java.util.function.Consumer;

public class ObservationAwareBaggageThreadLocalAccessor
        implements ThreadLocalAccessor<BaggageToPropagate> {

    public static final String KEY = "micrometer.tracing.baggage";

    final Map<Thread, BaggageAndScope> baggageInScope = new ConcurrentHashMap<>();

    // ...

    @Override
    public void setValue(BaggageToPropagate value) {
        BaggageAndScope previousScope = baggageInScope.get(Thread.currentThread());
        Span currentSpan = tracer.currentSpan();
        if (currentSpan == null) return;

        BaggageAndScope scope = openScopeForEachBaggageEntry(value, currentSpan);
        if (scope != null) {
            scope = scopeRestoringBaggageAndScope(scope, previousScope);
            baggageInScope.put(Thread.currentThread(), scope);
        }
    }

    @Override
    public void restore(BaggageToPropagate previousValue) {
        closeCurrentScope(); // Triggers the entire Consumer chain
    }
}
```

The baggage accessor uses **`Consumer.andThen()` composition** instead of an explicit linked list. Each baggage entry's close action is chained, and a final restore action pops the per-thread map:

```
accept(null) → close(entry1) → close(entry2) → restorePreviousScope
```

The key difference from the span accessor: `getCurrentSpan()` always returns a span reference (even when OTLA manages it), because baggage queries require a span context to look up against.

## 12.5 Try It Yourself

<details>
<summary>Challenge: Implement the getValue() coordination logic for the span accessor</summary>

Given a `Tracer`, an `ObservationRegistry`, and the `TracingObservationHandler.TracingContext` class, write the `getValue()` method that:
1. Returns `null` when no span is in scope
2. Returns the current span when no observation is active
3. Returns `null` when an observation's span matches the current span (OTLA handles it)
4. Returns the current span when it differs from the observation's span (manual child)

```java
@Override
public Span getValue() {
    Observation currentObservation = registry.getCurrentObservation();
    Span currentSpan = tracer.currentSpan();

    if (currentObservation != null) {
        TracingObservationHandler.TracingContext tracingContext =
                currentObservation.getContext().get(
                        TracingObservationHandler.TracingContext.class);
        if (tracingContext != null) {
            Span observationSpan = tracingContext.getSpan();
            if (currentSpan != null && !currentSpan.equals(observationSpan)) {
                return currentSpan;
            }
            return null;
        }
    }
    return currentSpan;
}
```

</details>

<details>
<summary>Challenge: Implement the SpanAction linked-list close() method</summary>

The `close()` method must:
1. Close the scope (if non-null)
2. Pop the linked list — restore the previous node into the map, or remove the entry if there's no previous

```java
@Override
public void close() {
    if (scope != null) {
        try { scope.close(); }
        catch (Exception ignored) { }
    }
    if (previous != null) {
        todo.put(Thread.currentThread(), previous);
    } else {
        todo.remove(Thread.currentThread());
    }
}
```

</details>

## 12.6 Tests

### Unit Tests

**New file:** `src/test/java/dev/linhvu/tracing/contextpropagation/BaggageToPropagateTest.java`

```java
class BaggageToPropagateTest {

    @Test
    void shouldCreateFromMap_WhenGivenValidEntries() { /* ... */ }

    @Test
    void shouldCreateFromVarargs_WhenGivenKeyValuePairs() { /* ... */ }

    @Test
    void shouldThrowException_WhenOddNumberOfVarargs() { /* ... */ }

    @Test
    void shouldDefensivelyCopyMap_WhenConstructed() { /* ... */ }

    @Test
    void shouldBeEqual_WhenSameBaggageEntries() { /* ... */ }
}
```

**New file:** `src/test/java/dev/linhvu/tracing/contextpropagation/ObservationAwareSpanThreadLocalAccessorTest.java`

```java
class ObservationAwareSpanThreadLocalAccessorTest {

    @Test
    void shouldReturnNull_WhenNoSpanInScope() { /* ... */ }

    @Test
    void shouldReturnSpan_WhenManuallyScoped() { /* ... */ }

    @Test
    void shouldReturnNull_WhenObservationManagesSpan() { /* ... */ }

    @Test
    void shouldReturnChildSpan_WhenManualChildCreatedUnderObservation() { /* ... */ }

    @Test
    void shouldSetAndRestoreSpan_WhenSetValueAndRestoreCalled() { /* ... */ }

    @Test
    void shouldSupportNestedSetValueAndRestore() { /* ... */ }

    @Test
    void shouldCleanUpSpanActionsMap_WhenFullyRestored() { /* ... */ }
}
```

**New file:** `src/test/java/dev/linhvu/tracing/contextpropagation/ObservationAwareBaggageThreadLocalAccessorTest.java`

```java
class ObservationAwareBaggageThreadLocalAccessorTest {

    @Test
    void shouldReturnNull_WhenNoBaggageExists() { /* ... */ }

    @Test
    void shouldCaptureBaggage_WhenBaggageInScope() { /* ... */ }

    @Test
    void shouldSetAndRestoreBaggage_WhenSetValueAndRestoreCalled() { /* ... */ }

    @Test
    void shouldSetMultipleBaggageEntries_WhenMultipleEntriesProvided() { /* ... */ }

    @Test
    void shouldNotSetBaggage_WhenNoSpanInScope() { /* ... */ }

    @Test
    void shouldCleanUpScopeMap_WhenFullyRestored() { /* ... */ }
}
```

### Integration Tests

**New file:** `src/test/java/dev/linhvu/tracing/integration/CrossThreadContextPropagationIntegrationTest.java`

```java
class CrossThreadContextPropagationIntegrationTest {

    @Test
    void shouldPropagateManualSpan_WhenUsingContextSnapshot() { /* ... */ }

    @Test
    void shouldPropagateObservationSpan_WhenUsingContextSnapshot() { /* ... */ }

    @Test
    void shouldPropagateBaggage_WhenUsingContextSnapshot() { /* ... */ }

    @Test
    void shouldPropagateSpanAndBaggage_WhenUsingContextExecutorService() { /* ... */ }

    @Test
    void shouldClearSpanOnWorkerThread_WhenScopeCloses() { /* ... */ }

    @Test
    void shouldPropagateManualChildSpan_WhenObservationAlsoActive() { /* ... */ }
}
```

**Run:** `./gradlew test` — expected: all 57 test suites pass (including all prior features' tests)

---

## 12.7 Why This Works

> ★ **Insight** -------------------------------------------
> - **Why not just wrap tasks with `tracer.withSpan()`?** A naive approach would capture the current span and wrap each task with `tracer.withSpan(capturedSpan)`. But this breaks when observations are involved — `ObservationThreadLocalAccessor` already restores the observation, which triggers `TracingObservationHandler.onScopeOpened()` to scope the span. A second `withSpan()` would create a duplicate scope, and the cleanup chain would unwind incorrectly. The observation-awareness check (`getValue()` returning `null` when OTLA manages the span) prevents this.
> - **When is observation awareness unnecessary?** If your application never uses the Observation API — only the Tracer API directly — then a simpler `ThreadLocalAccessor<Span>` that always returns `tracer.currentSpan()` would suffice. The observation-awareness complexity exists specifically for Spring Boot-style applications where instrumentation uses `Observation` and the `TracingObservationHandler` bridge creates spans automatically.
> -----------------------------------------------------------

> ★ **Insight** -------------------------------------------
> - **Linked list vs. Consumer composition — two patterns, one goal.** The span accessor uses an explicit `SpanAction` linked list (with a `previous` pointer), while the baggage accessor uses `Consumer.andThen()` to build a cleanup chain. Both achieve the same thing: nested set/restore cycles where each restore undoes exactly one level. The linked-list approach is easier to debug (you can inspect the stack depth), while the `Consumer` approach is more composable (naturally chains multiple baggage entries).
> - **Why `ConcurrentHashMap<Thread, ...>` instead of `ThreadLocal`?** A `ThreadLocal` would be simpler, but these accessors need to track scope state that is visible to the `restore()` call — which happens on the same thread as `setValue()`. The `ConcurrentHashMap` makes the scope chain inspectable (for testing) and enables explicit cleanup via `remove(Thread.currentThread())`. The real framework uses the same pattern.
> -----------------------------------------------------------

> ★ **Insight** -------------------------------------------
> - **Registration order is a load-bearing contract.** `ContextSnapshot.setThreadLocals()` processes accessors in registration order, and `DefaultScope.close()` processes them in reverse. OTLA must be first (so it restores the observation before the span accessor checks), and the span accessor must be second (so the baggage accessor can call `tracer.currentSpan()` during `setValue()`). Getting this order wrong doesn't throw an error — it silently produces duplicate or missing scopes.
> -----------------------------------------------------------

## 12.8 What We Enhanced

| Aspect | Before (ch06/ch08) | Current (ch12) | Real Framework |
|--------|-------------------|----------------|----------------|
| Cross-thread spans | `CurrentTraceContext.wrap()` was a passthrough — span lost on thread hop | `ObservationAwareSpanThreadLocalAccessor` captures and restores spans via `ContextSnapshot` | `ObservationAwareSpanThreadLocalAccessor.java:58` — same pattern with additional scope validation and logging |
| Cross-thread baggage | Baggage entries only accessible on the creating thread | `ObservationAwareBaggageThreadLocalAccessor` opens `BaggageInScope` on worker threads | `ObservationAwareBaggageThreadLocalAccessor.java:44` — same pattern with `Consumer.andThen` composition |
| Observation coordination | Span accessor and OTLA unaware of each other | `getValue()` checks `TracingContext` to yield when OTLA manages the span | `ObservationAwareSpanThreadLocalAccessor.java:98` — same `TracingContext` inspection pattern |
| Thread pool integration | No built-in support | `ContextExecutorService.wrap(executor, factory)` auto-propagates all registered context | `ContextExecutorService` from context-propagation library |

## 12.9 Connection to Real Framework

| Simplified Code | Real Framework Code | File:Line | Key Difference |
|----------------|--------------------|-----------| ---------------|
| `ObservationAwareSpanThreadLocalAccessor` | `ObservationAwareSpanThreadLocalAccessor` | `ObservationAwareSpanThreadLocalAccessor.java:58` | Real version has additional scope chain validation in `restore()` and assertion error logging on mismatched spans |
| `SpanAction` linked list | `SpanAction` | `ObservationAwareSpanThreadLocalAccessor.java:200` | Same structure — `previous` pointer + `ConcurrentHashMap<Thread, SpanAction>` |
| `getValue()` coordination | `getValue()` | `ObservationAwareSpanThreadLocalAccessor.java:98` | Real version also handles `@Nullable` annotations and has a static logger for diagnostic warnings |
| `ObservationAwareBaggageThreadLocalAccessor` | `ObservationAwareBaggageThreadLocalAccessor` | `ObservationAwareBaggageThreadLocalAccessor.java:44` | Real version has logging when baggage entry duplicates detected, deprecated `reset()` method |
| `BaggageAndScope` consumer chain | `BaggageAndScope` | `ObservationAwareBaggageThreadLocalAccessor.java:257` | Same `Consumer` composition pattern; real version also preserves entry reference for toString |
| `BaggageToPropagate` | `BaggageToPropagate` | `BaggageToPropagate.java:28` | Identical structure — `Map<String, String>` wrapper with defensive copy |

## 12.10 Complete Code

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
    implementation 'dev.linhvu:simple-context-propagation:1.0-SNAPSHOT'
    implementation 'org.assertj:assertj-core:3.27.3'

    testImplementation platform('org.junit:junit-bom:5.11.4')
    testImplementation 'org.junit.jupiter:junit-jupiter'
    testRuntimeOnly 'org.junit.platform:junit-platform-launcher'
}

test {
    useJUnitPlatform()
}
```

#### File: `src/main/java/dev/linhvu/tracing/contextpropagation/BaggageToPropagate.java` [NEW]

```java
package dev.linhvu.tracing.contextpropagation;

import java.util.HashMap;
import java.util.Map;
import java.util.Objects;

/**
 * Value type holding baggage key-value entries that should be propagated across
 * thread boundaries. Used as the generic type parameter for
 * {@link ObservationAwareBaggageThreadLocalAccessor}.
 *
 * <p>The map is defensively copied on construction so that mutations to the
 * original map do not affect the snapshot.
 */
public class BaggageToPropagate {

    private final Map<String, String> baggage;

    /**
     * Creates a baggage snapshot from a map of key-value entries.
     *
     * @param baggage the baggage entries to propagate
     */
    public BaggageToPropagate(Map<String, String> baggage) {
        this.baggage = new HashMap<>(baggage);
    }

    /**
     * Creates a baggage snapshot from alternating key-value pairs.
     *
     * @param baggage alternating key-value pairs (must have even length)
     * @throws IllegalArgumentException if the number of arguments is odd
     */
    public BaggageToPropagate(String... baggage) {
        if (baggage.length % 2 != 0) {
            throw new IllegalArgumentException(
                    "Baggage must be provided as key-value pairs, got odd count: "
                            + baggage.length);
        }
        this.baggage = new HashMap<>();
        for (int i = 0; i < baggage.length; i += 2) {
            this.baggage.put(baggage[i], baggage[i + 1]);
        }
    }

    /**
     * Returns the baggage entries.
     */
    public Map<String, String> getBaggage() {
        return baggage;
    }

    @Override
    public boolean equals(Object o) {
        if (this == o) {
            return true;
        }
        if (o == null || getClass() != o.getClass()) {
            return false;
        }
        BaggageToPropagate that = (BaggageToPropagate) o;
        return Objects.equals(baggage, that.baggage);
    }

    @Override
    public int hashCode() {
        return Objects.hash(baggage);
    }

    @Override
    public String toString() {
        return "BaggageToPropagate{" + baggage + '}';
    }

}
```

#### File: `src/main/java/dev/linhvu/tracing/contextpropagation/ObservationAwareSpanThreadLocalAccessor.java` [NEW]

```java
package dev.linhvu.tracing.contextpropagation;

import dev.linhvu.context.ThreadLocalAccessor;
import dev.linhvu.micrometer.observation.Observation;
import dev.linhvu.micrometer.observation.ObservationRegistry;
import dev.linhvu.tracing.Span;
import dev.linhvu.tracing.Tracer;
import dev.linhvu.tracing.handler.TracingObservationHandler;

import java.io.Closeable;
import java.util.Map;
import java.util.Objects;
import java.util.concurrent.ConcurrentHashMap;

/**
 * A {@link ThreadLocalAccessor} that propagates the current {@link Span} across
 * thread boundaries. Coordinates with {@code ObservationThreadLocalAccessor} (OTLA)
 * to avoid duplicating spans when OTLA has already scoped one via
 * {@link TracingObservationHandler}.
 *
 * <p><b>Decision logic in {@link #getValue()}:</b>
 * <ul>
 *   <li>If a current observation exists and its span is the same as
 *       {@code tracer.currentSpan()}, then OTLA is managing the span → return
 *       {@code null} (nothing for this accessor to propagate)</li>
 *   <li>If the current span is different from the observation's span, the user
 *       manually created a child span → return that span</li>
 *   <li>If no observation is active, return {@code tracer.currentSpan()}</li>
 * </ul>
 *
 * <p><b>Scope management:</b> Uses a {@link ConcurrentHashMap} keyed by
 * {@link Thread} to maintain a per-thread linked list of {@link SpanAction}
 * nodes. Each {@code setValue()} pushes a node; each {@code restore()} pops one.
 * This supports nested context hops.
 *
 * <p><b>Registration order:</b> This accessor <b>must</b> be registered after
 * {@code ObservationThreadLocalAccessor} in the {@link dev.linhvu.context.ContextRegistry}
 * so that OTLA processes first.
 *
 * @see TracingObservationHandler.TracingContext
 */
public class ObservationAwareSpanThreadLocalAccessor implements ThreadLocalAccessor<Span> {

    /**
     * The key under which the current span is stored in context snapshots.
     */
    public static final String KEY = "micrometer.tracing";

    final Map<Thread, SpanAction> spanActions = new ConcurrentHashMap<>();

    private final ObservationRegistry registry;

    private final Tracer tracer;

    public ObservationAwareSpanThreadLocalAccessor(Tracer tracer) {
        this(ObservationRegistry.create(), tracer);
    }

    public ObservationAwareSpanThreadLocalAccessor(ObservationRegistry observationRegistry,
            Tracer tracer) {
        this.registry = Objects.requireNonNull(observationRegistry);
        this.tracer = Objects.requireNonNull(tracer);
    }

    @Override
    public Object key() {
        return KEY;
    }

    /**
     * Captures the current span for propagation — but only if it was NOT
     * already created by {@code TracingObservationHandler} via OTLA.
     */
    @Override
    public Span getValue() {
        Observation currentObservation = registry.getCurrentObservation();
        Span currentSpan = tracer.currentSpan();

        if (currentObservation != null) {
            // An observation is active — check if OTLA already manages the span
            TracingObservationHandler.TracingContext tracingContext =
                    currentObservation.getContext().get(
                            TracingObservationHandler.TracingContext.class);
            if (tracingContext != null) {
                Span observationSpan = tracingContext.getSpan();
                if (currentSpan != null && !currentSpan.equals(observationSpan)) {
                    // User manually created a child span outside of observation
                    return currentSpan;
                }
                // OTLA manages this span — nothing for us to propagate
                return null;
            }
        }

        // No observation, or no TracingContext — we own the span
        return currentSpan;
    }

    /**
     * Restores a previously captured span onto the current thread by opening
     * a new scope via {@code tracer.withSpan()}.
     */
    @Override
    public void setValue(Span value) {
        SpanAction previous = spanActions.get(Thread.currentThread());
        Tracer.SpanInScope scope = tracer.withSpan(value);
        SpanAction action = new SpanAction(previous, spanActions);
        spanActions.put(Thread.currentThread(), action);
        action.setScope(scope);
    }

    /**
     * Clears the span on the current thread (pushes a null-span scope).
     */
    @Override
    public void setValue() {
        SpanAction previous = spanActions.get(Thread.currentThread());
        if (previous == null) {
            return;
        }
        Tracer.SpanInScope scope = tracer.withSpan(null);
        previous.setScope(scope);
    }

    /**
     * Tears down the scope set by {@link #setValue(Span)}, restoring the
     * previous span.
     */
    @Override
    public void restore(Span previousValue) {
        SpanAction spanAction = spanActions.get(Thread.currentThread());
        if (spanAction == null) {
            return;
        }
        spanAction.close();
    }

    /**
     * Tears down the scope when there was no previous span.
     */
    @Override
    public void restore() {
        SpanAction spanAction = spanActions.get(Thread.currentThread());
        if (spanAction == null) {
            return;
        }
        spanAction.close();
    }

    // =========================================================================
    // Inner class: SpanAction — per-thread linked list node
    // =========================================================================

    /**
     * A node in a per-thread linked list that tracks open span scopes.
     * Each {@code setValue()} pushes a new node; each {@code restore()} pops one.
     *
     * <p>The linked-list structure supports nested context hops: thread A hops
     * to B, which hops to C. When C finishes, B's scope is restored; when B
     * finishes, A's scope is restored.
     */
    static class SpanAction implements AutoCloseable {

        private final SpanAction previous;

        private final Map<Thread, SpanAction> todo;

        private Closeable scope;

        SpanAction(SpanAction previous, Map<Thread, SpanAction> todo) {
            this.previous = previous;
            this.todo = todo;
        }

        void setScope(Closeable scope) {
            this.scope = scope;
        }

        @Override
        public void close() {
            if (scope != null) {
                try {
                    scope.close();
                }
                catch (Exception ignored) {
                    // Scope.close() does not throw
                }
            }
            // Pop the linked list: restore previous node or remove entry
            if (previous != null) {
                todo.put(Thread.currentThread(), previous);
            }
            else {
                todo.remove(Thread.currentThread());
            }
        }

    }

}
```

#### File: `src/main/java/dev/linhvu/tracing/contextpropagation/ObservationAwareBaggageThreadLocalAccessor.java` [NEW]

```java
package dev.linhvu.tracing.contextpropagation;

import dev.linhvu.context.ThreadLocalAccessor;
import dev.linhvu.micrometer.observation.Observation;
import dev.linhvu.micrometer.observation.ObservationRegistry;
import dev.linhvu.tracing.BaggageInScope;
import dev.linhvu.tracing.Span;
import dev.linhvu.tracing.Tracer;
import dev.linhvu.tracing.handler.TracingObservationHandler;

import java.util.Map;
import java.util.Objects;
import java.util.concurrent.ConcurrentHashMap;
import java.util.function.Consumer;

/**
 * A {@link ThreadLocalAccessor} that propagates baggage entries across thread
 * boundaries. Like its span counterpart, it is observation-aware and coordinates
 * with {@code ObservationThreadLocalAccessor} to determine the current span for
 * baggage queries.
 *
 * <p><b>How it works:</b>
 * <ul>
 *   <li>{@link #getValue()} — reads all current baggage and wraps it in a
 *       {@link BaggageToPropagate} snapshot</li>
 *   <li>{@link #setValue(BaggageToPropagate)} — opens a {@link BaggageInScope}
 *       for each entry, composing the close actions via {@code Consumer.andThen()}</li>
 *   <li>{@link #restore} — closes the current scope chain, restoring previous state</li>
 * </ul>
 *
 * <p><b>Scope management:</b> Uses {@code Consumer.andThen()} composition to
 * chain close actions for multiple baggage entries, plus a restore action that
 * pops the per-thread map back to the previous state.
 */
public class ObservationAwareBaggageThreadLocalAccessor
        implements ThreadLocalAccessor<BaggageToPropagate> {

    /**
     * The key under which baggage is stored in context snapshots.
     */
    public static final String KEY = "micrometer.tracing.baggage";

    final Map<Thread, BaggageAndScope> baggageInScope = new ConcurrentHashMap<>();

    private final Tracer tracer;

    private final ObservationRegistry registry;

    public ObservationAwareBaggageThreadLocalAccessor(ObservationRegistry observationRegistry,
            Tracer tracer) {
        this.registry = Objects.requireNonNull(observationRegistry);
        this.tracer = Objects.requireNonNull(tracer);
    }

    @Override
    public Object key() {
        return KEY;
    }

    /**
     * Captures all current baggage entries for propagation.
     */
    @Override
    public BaggageToPropagate getValue() {
        Span currentSpan = getCurrentSpan(registry.getCurrentObservation());
        Map<String, String> allBaggage;
        if (currentSpan != null) {
            allBaggage = tracer.getAllBaggage(currentSpan.context());
        }
        else {
            allBaggage = tracer.getAllBaggage();
        }
        if (allBaggage == null || allBaggage.isEmpty()) {
            return null;
        }
        return new BaggageToPropagate(allBaggage);
    }

    /**
     * Determines the current span, coordinating with the observation system.
     * Unlike the span accessor (which returns null when OTLA manages the span),
     * the baggage accessor always needs a span reference to query baggage against.
     */
    private Span getCurrentSpan(Observation currentObservation) {
        Span currentSpan = tracer.currentSpan();

        if (currentObservation != null) {
            TracingObservationHandler.TracingContext tracingContext =
                    currentObservation.getContext().get(
                            TracingObservationHandler.TracingContext.class);
            if (tracingContext != null) {
                Span observationSpan = tracingContext.getSpan();
                if (currentSpan != null && !currentSpan.equals(observationSpan)) {
                    // User manually created a child span
                    return currentSpan;
                }
                // Return the observation's span (baggage needs a span reference)
                return observationSpan;
            }
        }

        return currentSpan;
    }

    /**
     * Restores baggage entries onto the current thread by opening a
     * {@link BaggageInScope} for each entry.
     */
    @Override
    public void setValue(BaggageToPropagate value) {
        BaggageAndScope previousScope = baggageInScope.get(Thread.currentThread());

        Span currentSpan = tracer.currentSpan();
        if (currentSpan == null) {
            return;
        }

        BaggageAndScope scope = openScopeForEachBaggageEntry(value, currentSpan);
        if (scope != null) {
            scope = scopeRestoringBaggageAndScope(scope, previousScope);
            baggageInScope.put(Thread.currentThread(), scope);
        }
    }

    /**
     * Clears baggage on the current thread when the snapshot has no baggage.
     */
    @Override
    public void setValue() {
        BaggageAndScope previousScope = baggageInScope.get(Thread.currentThread());
        if (previousScope == null) {
            return;
        }
        // Push a null-span scope as a barrier
        Tracer.SpanInScope spanScope = tracer.withSpan(null);
        BaggageAndScope scope = new BaggageAndScope(o -> {
            try {
                spanScope.close();
            }
            catch (Exception ignored) {
            }
        });
        scope = scopeRestoringBaggageAndScope(scope, previousScope);
        baggageInScope.put(Thread.currentThread(), scope);
    }

    @Override
    public void restore(BaggageToPropagate previousValue) {
        closeCurrentScope();
    }

    @Override
    public void restore() {
        closeCurrentScope();
    }

    /**
     * Closes the current baggage scope chain for this thread.
     */
    void closeCurrentScope() {
        BaggageAndScope scope = baggageInScope.get(Thread.currentThread());
        if (scope != null) {
            scope.accept(null);
        }
    }

    /**
     * Opens a {@link BaggageInScope} for each entry in the baggage snapshot.
     * The close actions are chained via {@code andThen()}.
     */
    private BaggageAndScope openScopeForEachBaggageEntry(BaggageToPropagate value,
            Span currentSpan) {
        BaggageAndScope scope = null;

        for (Map.Entry<String, String> entry : value.getBaggage().entrySet()) {
            BaggageInScope baggageInScope = tracer.createBaggageInScope(
                    currentSpan.context(), entry.getKey(), entry.getValue());
            BaggageAndScope entryScope = new BaggageAndScope(
                    o -> baggageInScope.close(), entry);

            if (scope == null) {
                scope = entryScope;
            }
            else {
                scope = scope.andThen(entryScope);
            }
        }

        return scope;
    }

    /**
     * Appends a restore action to the scope chain. When the chain is closed,
     * the previous scope is restored into the per-thread map (or the entry
     * is removed if there was no previous scope).
     */
    private BaggageAndScope scopeRestoringBaggageAndScope(BaggageAndScope currentScope,
            BaggageAndScope previousScope) {
        return currentScope.andThen(new BaggageAndScope(o -> {
            if (previousScope != null) {
                baggageInScope.put(Thread.currentThread(), previousScope);
            }
            else {
                baggageInScope.remove(Thread.currentThread());
            }
        }));
    }

    // =========================================================================
    // Inner class: BaggageAndScope — composable scope close action
    // =========================================================================

    /**
     * A composable consumer that wraps scope-closing logic for baggage entries.
     * Multiple entries' close actions are chained using {@link #andThen}.
     */
    static class BaggageAndScope implements Consumer<Object> {

        private final Consumer<Object> consumer;

        private final Map.Entry<String, String> entry;

        BaggageAndScope(Consumer<Object> consumer) {
            this(consumer, null);
        }

        BaggageAndScope(Consumer<Object> consumer, Map.Entry<String, String> entry) {
            this.consumer = consumer;
            this.entry = entry;
        }

        @Override
        public void accept(Object o) {
            consumer.accept(o);
        }

        /**
         * Chains another consumer, returning a new {@code BaggageAndScope}.
         */
        public BaggageAndScope andThen(Consumer<? super Object> after) {
            return new BaggageAndScope(o -> {
                this.accept(o);
                after.accept(o);
            }, this.entry);
        }

        @Override
        public String toString() {
            return "BaggageAndScope{entry=" + entry + '}';
        }

    }

}
```

### Test Code

#### File: `src/test/java/dev/linhvu/tracing/contextpropagation/BaggageToPropagateTest.java` [NEW]

```java
package dev.linhvu.tracing.contextpropagation;

import org.junit.jupiter.api.Test;

import java.util.Map;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

/**
 * Unit tests for {@link BaggageToPropagate}.
 */
class BaggageToPropagateTest {

    @Test
    void shouldCreateFromMap_WhenGivenValidEntries() {
        // Arrange
        Map<String, String> entries = Map.of("user-id", "u-123", "tenant", "t-456");

        // Act
        BaggageToPropagate baggage = new BaggageToPropagate(entries);

        // Assert
        assertThat(baggage.getBaggage()).containsEntry("user-id", "u-123")
                .containsEntry("tenant", "t-456");
    }

    @Test
    void shouldCreateFromVarargs_WhenGivenKeyValuePairs() {
        // Act
        BaggageToPropagate baggage = new BaggageToPropagate("user-id", "u-123",
                "tenant", "t-456");

        // Assert
        assertThat(baggage.getBaggage()).hasSize(2)
                .containsEntry("user-id", "u-123")
                .containsEntry("tenant", "t-456");
    }

    @Test
    void shouldThrowException_WhenOddNumberOfVarargs() {
        assertThatThrownBy(() -> new BaggageToPropagate("user-id", "u-123", "orphan"))
                .isInstanceOf(IllegalArgumentException.class)
                .hasMessageContaining("odd count");
    }

    @Test
    void shouldDefensivelyCopyMap_WhenConstructed() {
        // Arrange
        Map<String, String> original = new java.util.HashMap<>();
        original.put("key", "value");
        BaggageToPropagate baggage = new BaggageToPropagate(original);

        // Act — mutate original
        original.put("key", "modified");

        // Assert — snapshot is unaffected
        assertThat(baggage.getBaggage()).containsEntry("key", "value");
    }

    @Test
    void shouldBeEqual_WhenSameBaggageEntries() {
        BaggageToPropagate a = new BaggageToPropagate(Map.of("k", "v"));
        BaggageToPropagate b = new BaggageToPropagate(Map.of("k", "v"));

        assertThat(a).isEqualTo(b);
        assertThat(a.hashCode()).isEqualTo(b.hashCode());
    }

    @Test
    void shouldNotBeEqual_WhenDifferentBaggageEntries() {
        BaggageToPropagate a = new BaggageToPropagate(Map.of("k", "v1"));
        BaggageToPropagate b = new BaggageToPropagate(Map.of("k", "v2"));

        assertThat(a).isNotEqualTo(b);
    }

    @Test
    void shouldHaveReadableToString() {
        BaggageToPropagate baggage = new BaggageToPropagate("user", "alice");

        assertThat(baggage.toString()).contains("user").contains("alice");
    }

}
```

#### File: `src/test/java/dev/linhvu/tracing/contextpropagation/ObservationAwareSpanThreadLocalAccessorTest.java` [NEW]

```java
package dev.linhvu.tracing.contextpropagation;

import dev.linhvu.micrometer.observation.Observation;
import dev.linhvu.micrometer.observation.ObservationRegistry;
import dev.linhvu.tracing.Span;
import dev.linhvu.tracing.Tracer;
import dev.linhvu.tracing.handler.DefaultTracingObservationHandler;
import dev.linhvu.tracing.handler.TracingObservationHandler;
import dev.linhvu.tracing.simple.SimpleSpan;
import dev.linhvu.tracing.simple.SimpleTracer;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;

/**
 * Unit tests for {@link ObservationAwareSpanThreadLocalAccessor}.
 */
class ObservationAwareSpanThreadLocalAccessorTest {

    private SimpleTracer tracer;

    private ObservationRegistry registry;

    private ObservationAwareSpanThreadLocalAccessor accessor;

    @BeforeEach
    void setUp() {
        tracer = new SimpleTracer();
        registry = ObservationRegistry.create();
        registry.observationConfig()
                .observationHandler(new DefaultTracingObservationHandler(tracer));
        accessor = new ObservationAwareSpanThreadLocalAccessor(registry, tracer);
    }

    @Test
    void shouldReturnCorrectKey() {
        assertThat(accessor.key()).isEqualTo("micrometer.tracing");
    }

    @Test
    void shouldReturnNull_WhenNoSpanInScope() {
        assertThat(accessor.getValue()).isNull();
    }

    @Test
    void shouldReturnSpan_WhenManuallyScoped() {
        // Arrange — manually create and scope a span (no observation)
        SimpleSpan span = tracer.nextSpan().name("manual").start();
        try (Tracer.SpanInScope ws = tracer.withSpan(span)) {
            // Act
            Span captured = accessor.getValue();

            // Assert — with no observation, the span is returned
            assertThat(captured).isSameAs(span);
        }
        finally {
            span.end();
        }
    }

    @Test
    void shouldReturnNull_WhenObservationManagesSpan() {
        // Arrange — create an observation (which creates a span via handler)
        Observation observation = Observation.createNotStarted("test.op", registry).start();
        try (Observation.Scope scope = observation.openScope()) {
            // Act — OTLA manages the span, so this accessor should back off
            Span captured = accessor.getValue();

            // Assert
            assertThat(captured).isNull();
        }
        observation.stop();
    }

    @Test
    void shouldReturnChildSpan_WhenManualChildCreatedUnderObservation() {
        // Arrange — observation creates a span, then user creates a manual child
        Observation observation = Observation.createNotStarted("parent.op", registry)
                .start();
        try (Observation.Scope scope = observation.openScope()) {
            SimpleSpan manualChild = tracer.nextSpan().name("manual-child").start();
            try (Tracer.SpanInScope ws = tracer.withSpan(manualChild)) {
                // Act — manual child is different from observation's span
                Span captured = accessor.getValue();

                // Assert — returns the manual child
                assertThat(captured).isSameAs(manualChild);
            }
            finally {
                manualChild.end();
            }
        }
        observation.stop();
    }

    @Test
    void shouldSetAndRestoreSpan_WhenSetValueAndRestoreCalled() {
        // Arrange
        SimpleSpan span = tracer.nextSpan().name("propagated").start();

        // Act — set the span on this thread
        accessor.setValue(span);

        // Assert — span is now current
        assertThat(tracer.currentSpan()).isSameAs(span);

        // Act — restore (remove)
        accessor.restore(span);

        // Assert — span is removed from scope
        assertThat(tracer.currentSpan()).isNull();
        span.end();
    }

    @Test
    void shouldSupportNestedSetValueAndRestore() {
        // Arrange
        SimpleSpan outer = tracer.nextSpan().name("outer").start();
        SimpleSpan inner = tracer.nextSpan().name("inner").start();

        // Act — push outer, then push inner
        accessor.setValue(outer);
        assertThat(tracer.currentSpan()).isSameAs(outer);

        accessor.setValue(inner);
        assertThat(tracer.currentSpan()).isSameAs(inner);

        // Restore inner — should pop back to outer
        accessor.restore(inner);
        assertThat(tracer.currentSpan()).isSameAs(outer);

        // Restore outer — should clear
        accessor.restore(outer);
        assertThat(tracer.currentSpan()).isNull();

        outer.end();
        inner.end();
    }

    @Test
    void shouldHandleSetValueNoArg_WhenPreviousActionExists() {
        // Arrange — set a span first
        SimpleSpan span = tracer.nextSpan().name("span").start();
        accessor.setValue(span);
        assertThat(tracer.currentSpan()).isSameAs(span);

        // Act — clear (no-arg setValue)
        accessor.setValue();

        // Assert — span is cleared
        assertThat(tracer.currentSpan()).isNull();

        // Restore
        accessor.restore();
        assertThat(tracer.currentSpan()).isSameAs(span);

        // Final cleanup
        accessor.restore(span);
        span.end();
    }

    @Test
    void shouldCleanUpSpanActionsMap_WhenFullyRestored() {
        SimpleSpan span = tracer.nextSpan().name("temp").start();

        accessor.setValue(span);
        assertThat(accessor.spanActions).isNotEmpty();

        accessor.restore(span);
        assertThat(accessor.spanActions).isEmpty();

        span.end();
    }

}
```

#### File: `src/test/java/dev/linhvu/tracing/contextpropagation/ObservationAwareBaggageThreadLocalAccessorTest.java` [NEW]

```java
package dev.linhvu.tracing.contextpropagation;

import dev.linhvu.micrometer.observation.Observation;
import dev.linhvu.micrometer.observation.ObservationRegistry;
import dev.linhvu.tracing.BaggageInScope;
import dev.linhvu.tracing.Tracer;
import dev.linhvu.tracing.handler.DefaultTracingObservationHandler;
import dev.linhvu.tracing.simple.SimpleSpan;
import dev.linhvu.tracing.simple.SimpleTracer;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;

/**
 * Unit tests for {@link ObservationAwareBaggageThreadLocalAccessor}.
 */
class ObservationAwareBaggageThreadLocalAccessorTest {

    private SimpleTracer tracer;

    private ObservationRegistry registry;

    private ObservationAwareBaggageThreadLocalAccessor accessor;

    @BeforeEach
    void setUp() {
        tracer = new SimpleTracer();
        registry = ObservationRegistry.create();
        registry.observationConfig()
                .observationHandler(new DefaultTracingObservationHandler(tracer));
        accessor = new ObservationAwareBaggageThreadLocalAccessor(registry, tracer);
    }

    @Test
    void shouldReturnCorrectKey() {
        assertThat(accessor.key()).isEqualTo("micrometer.tracing.baggage");
    }

    @Test
    void shouldReturnNull_WhenNoBaggageExists() {
        assertThat(accessor.getValue()).isNull();
    }

    @Test
    void shouldCaptureBaggage_WhenBaggageInScope() {
        // Arrange
        SimpleSpan span = tracer.nextSpan().name("request").start();
        try (Tracer.SpanInScope ws = tracer.withSpan(span)) {
            try (BaggageInScope userId = tracer.createBaggageInScope("user-id", "u-123")) {
                // Act
                BaggageToPropagate captured = accessor.getValue();

                // Assert
                assertThat(captured).isNotNull();
                assertThat(captured.getBaggage()).containsEntry("user-id", "u-123");
            }
        }
        finally {
            span.end();
        }
    }

    @Test
    void shouldReturnNull_WhenBaggageIsEmpty() {
        SimpleSpan span = tracer.nextSpan().name("request").start();
        try (Tracer.SpanInScope ws = tracer.withSpan(span)) {
            // No baggage created
            assertThat(accessor.getValue()).isNull();
        }
        finally {
            span.end();
        }
    }

    @Test
    void shouldSetAndRestoreBaggage_WhenSetValueAndRestoreCalled() {
        // Arrange — create a span in scope (baggage requires a span)
        SimpleSpan span = tracer.nextSpan().name("request").start();
        try (Tracer.SpanInScope ws = tracer.withSpan(span)) {
            BaggageToPropagate baggage = new BaggageToPropagate("user-id", "u-123");

            // Act — set baggage
            accessor.setValue(baggage);

            // Assert — baggage is now available
            assertThat(tracer.getAllBaggage()).containsEntry("user-id", "u-123");

            // Act — restore
            accessor.restore(baggage);

            // Assert — scope map is cleaned up
            assertThat(accessor.baggageInScope).isEmpty();
        }
        finally {
            span.end();
        }
    }

    @Test
    void shouldSetMultipleBaggageEntries_WhenMultipleEntriesProvided() {
        SimpleSpan span = tracer.nextSpan().name("request").start();
        try (Tracer.SpanInScope ws = tracer.withSpan(span)) {
            BaggageToPropagate baggage = new BaggageToPropagate(
                    "user-id", "u-123", "tenant", "t-456");

            // Act
            accessor.setValue(baggage);

            // Assert
            assertThat(tracer.getAllBaggage())
                    .containsEntry("user-id", "u-123")
                    .containsEntry("tenant", "t-456");

            // Cleanup
            accessor.restore(baggage);
        }
        finally {
            span.end();
        }
    }

    @Test
    void shouldNotSetBaggage_WhenNoSpanInScope() {
        // No span in scope
        BaggageToPropagate baggage = new BaggageToPropagate("user-id", "u-123");

        // Act — should be a no-op
        accessor.setValue(baggage);

        // Assert — no scope was opened
        assertThat(accessor.baggageInScope).isEmpty();
    }

    @Test
    void shouldCleanUpScopeMap_WhenFullyRestored() {
        SimpleSpan span = tracer.nextSpan().name("request").start();
        try (Tracer.SpanInScope ws = tracer.withSpan(span)) {
            BaggageToPropagate baggage = new BaggageToPropagate("key", "value");

            accessor.setValue(baggage);
            assertThat(accessor.baggageInScope).isNotEmpty();

            accessor.restore(baggage);
            assertThat(accessor.baggageInScope).isEmpty();
        }
        finally {
            span.end();
        }
    }

}
```

#### File: `src/test/java/dev/linhvu/tracing/integration/CrossThreadContextPropagationIntegrationTest.java` [NEW]

```java
package dev.linhvu.tracing.integration;

import dev.linhvu.context.ContextExecutorService;
import dev.linhvu.context.ContextRegistry;
import dev.linhvu.context.ContextSnapshot;
import dev.linhvu.context.ContextSnapshotFactory;
import dev.linhvu.micrometer.observation.Observation;
import dev.linhvu.micrometer.observation.ObservationRegistry;
import dev.linhvu.micrometer.observation.contextpropagation.ObservationThreadLocalAccessor;
import dev.linhvu.tracing.BaggageInScope;
import dev.linhvu.tracing.Span;
import dev.linhvu.tracing.Tracer;
import dev.linhvu.tracing.contextpropagation.BaggageToPropagate;
import dev.linhvu.tracing.contextpropagation.ObservationAwareBaggageThreadLocalAccessor;
import dev.linhvu.tracing.contextpropagation.ObservationAwareSpanThreadLocalAccessor;
import dev.linhvu.tracing.handler.DefaultTracingObservationHandler;
import dev.linhvu.tracing.simple.SimpleSpan;
import dev.linhvu.tracing.simple.SimpleTracer;

import org.junit.jupiter.api.AfterEach;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.util.concurrent.CountDownLatch;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.TimeUnit;
import java.util.concurrent.atomic.AtomicReference;

import static org.assertj.core.api.Assertions.assertThat;

/**
 * Integration tests verifying that spans and baggage propagate correctly
 * across thread boundaries using {@link ContextSnapshot} and
 * {@link ContextExecutorService}.
 */
class CrossThreadContextPropagationIntegrationTest {

    private SimpleTracer tracer;

    private ObservationRegistry observationRegistry;

    private ContextRegistry contextRegistry;

    private ContextSnapshotFactory snapshotFactory;

    private ObservationAwareSpanThreadLocalAccessor spanAccessor;

    private ObservationAwareBaggageThreadLocalAccessor baggageAccessor;

    @BeforeEach
    void setUp() {
        tracer = new SimpleTracer();
        observationRegistry = ObservationRegistry.create();
        observationRegistry.observationConfig()
                .observationHandler(new DefaultTracingObservationHandler(tracer));

        // Create a local registry (not the global singleton) to avoid test interference
        contextRegistry = new ContextRegistry();

        // Register accessors in order: OTLA first, then span, then baggage
        ObservationThreadLocalAccessor otla =
                new ObservationThreadLocalAccessor(observationRegistry);
        spanAccessor = new ObservationAwareSpanThreadLocalAccessor(
                observationRegistry, tracer);
        baggageAccessor = new ObservationAwareBaggageThreadLocalAccessor(
                observationRegistry, tracer);

        contextRegistry.registerThreadLocalAccessor(otla);
        contextRegistry.registerThreadLocalAccessor(spanAccessor);
        contextRegistry.registerThreadLocalAccessor(baggageAccessor);

        snapshotFactory = ContextSnapshotFactory.builder()
                .contextRegistry(contextRegistry)
                .clearMissing(true)
                .build();
    }

    @AfterEach
    void tearDown() {
        contextRegistry.removeThreadLocalAccessor(ObservationThreadLocalAccessor.KEY);
        contextRegistry.removeThreadLocalAccessor(
                ObservationAwareSpanThreadLocalAccessor.KEY);
        contextRegistry.removeThreadLocalAccessor(
                ObservationAwareBaggageThreadLocalAccessor.KEY);
    }

    @Test
    void shouldPropagateManualSpan_WhenUsingContextSnapshot() throws Exception {
        // Arrange — manually create and scope a span
        SimpleSpan span = tracer.nextSpan().name("manual-span").start();
        AtomicReference<Span> capturedOnOtherThread = new AtomicReference<>();
        CountDownLatch latch = new CountDownLatch(1);

        try (Tracer.SpanInScope ws = tracer.withSpan(span)) {
            // Capture snapshot while span is in scope
            ContextSnapshot snapshot = snapshotFactory.captureAll();

            // Act — run on another thread with snapshot
            new Thread(() -> {
                try (ContextSnapshot.Scope scope = snapshot.setThreadLocals()) {
                    capturedOnOtherThread.set(tracer.currentSpan());
                }
                latch.countDown();
            }).start();

            latch.await(5, TimeUnit.SECONDS);
        }
        finally {
            span.end();
        }

        // Assert
        assertThat(capturedOnOtherThread.get()).isSameAs(span);
    }

    @Test
    void shouldPropagateObservationSpan_WhenUsingContextSnapshot() throws Exception {
        // Arrange — create an observation (span managed by handler)
        Observation observation = Observation.createNotStarted("obs.op", observationRegistry)
                .start();
        AtomicReference<Span> capturedOnOtherThread = new AtomicReference<>();
        CountDownLatch latch = new CountDownLatch(1);

        try (Observation.Scope scope = observation.openScope()) {
            Span currentSpan = tracer.currentSpan();
            assertThat(currentSpan).isNotNull();

            // Capture — OTLA captures the observation, span accessor yields
            ContextSnapshot snapshot = snapshotFactory.captureAll();

            // Act — restore on another thread
            new Thread(() -> {
                try (ContextSnapshot.Scope s = snapshot.setThreadLocals()) {
                    capturedOnOtherThread.set(tracer.currentSpan());
                }
                latch.countDown();
            }).start();

            latch.await(5, TimeUnit.SECONDS);
        }
        observation.stop();

        // Assert — the observation's span was propagated (via OTLA + handler)
        assertThat(capturedOnOtherThread.get()).isNotNull();
    }

    @Test
    void shouldPropagateBaggage_WhenUsingContextSnapshot() throws Exception {
        // Arrange
        SimpleSpan span = tracer.nextSpan().name("request").start();
        AtomicReference<String> capturedUserId = new AtomicReference<>();
        CountDownLatch latch = new CountDownLatch(1);

        try (Tracer.SpanInScope ws = tracer.withSpan(span)) {
            try (BaggageInScope userId = tracer.createBaggageInScope("user-id", "u-123")) {
                // Capture snapshot with span + baggage
                ContextSnapshot snapshot = snapshotFactory.captureAll();

                // Act — restore on another thread
                new Thread(() -> {
                    try (ContextSnapshot.Scope scope = snapshot.setThreadLocals()) {
                        capturedUserId.set(
                                tracer.getAllBaggage().get("user-id"));
                    }
                    latch.countDown();
                }).start();

                latch.await(5, TimeUnit.SECONDS);
            }
        }
        finally {
            span.end();
        }

        // Assert
        assertThat(capturedUserId.get()).isEqualTo("u-123");
    }

    @Test
    void shouldPropagateSpanAndBaggage_WhenUsingContextExecutorService()
            throws Exception {
        // Arrange
        ExecutorService rawExecutor = Executors.newSingleThreadExecutor();
        ExecutorService contextExecutor = ContextExecutorService.wrap(
                rawExecutor, snapshotFactory);

        SimpleSpan span = tracer.nextSpan().name("request").start();
        AtomicReference<Span> capturedSpan = new AtomicReference<>();
        AtomicReference<String> capturedBaggage = new AtomicReference<>();
        CountDownLatch latch = new CountDownLatch(1);

        try (Tracer.SpanInScope ws = tracer.withSpan(span)) {
            try (BaggageInScope userId = tracer.createBaggageInScope("user-id", "u-789")) {
                // Act — submit to context-propagating executor
                contextExecutor.submit(() -> {
                    capturedSpan.set(tracer.currentSpan());
                    capturedBaggage.set(
                            tracer.getAllBaggage().get("user-id"));
                    latch.countDown();
                });

                latch.await(5, TimeUnit.SECONDS);
            }
        }
        finally {
            span.end();
            rawExecutor.shutdown();
        }

        // Assert
        assertThat(capturedSpan.get()).isSameAs(span);
        assertThat(capturedBaggage.get()).isEqualTo("u-789");
    }

    @Test
    void shouldClearSpanOnWorkerThread_WhenScopeCloses() throws Exception {
        // Arrange
        SimpleSpan span = tracer.nextSpan().name("temp").start();
        AtomicReference<Span> afterScopeClose = new AtomicReference<>();
        CountDownLatch latch = new CountDownLatch(1);

        try (Tracer.SpanInScope ws = tracer.withSpan(span)) {
            ContextSnapshot snapshot = snapshotFactory.captureAll();

            new Thread(() -> {
                // Before restore: no span on worker thread
                assertThat(tracer.currentSpan()).isNull();

                try (ContextSnapshot.Scope scope = snapshot.setThreadLocals()) {
                    // During: span is present
                    assertThat(tracer.currentSpan()).isSameAs(span);
                }

                // After scope close: span is cleaned up
                afterScopeClose.set(tracer.currentSpan());
                latch.countDown();
            }).start();

            latch.await(5, TimeUnit.SECONDS);
        }
        finally {
            span.end();
        }

        // Assert
        assertThat(afterScopeClose.get()).isNull();
    }

    @Test
    void shouldPropagateManualChildSpan_WhenObservationAlsoActive() throws Exception {
        // Arrange — observation is active, but user created a manual child
        Observation observation = Observation.createNotStarted("parent", observationRegistry)
                .start();
        AtomicReference<Span> capturedChild = new AtomicReference<>();
        CountDownLatch latch = new CountDownLatch(1);

        try (Observation.Scope scope = observation.openScope()) {
            SimpleSpan manualChild = tracer.nextSpan().name("manual-child").start();
            try (Tracer.SpanInScope ws = tracer.withSpan(manualChild)) {
                // Both OTLA (observation) and span accessor (manual child) should capture
                ContextSnapshot snapshot = snapshotFactory.captureAll();

                new Thread(() -> {
                    try (ContextSnapshot.Scope s = snapshot.setThreadLocals()) {
                        capturedChild.set(tracer.currentSpan());
                    }
                    latch.countDown();
                }).start();

                latch.await(5, TimeUnit.SECONDS);
            }
            finally {
                manualChild.end();
            }
        }
        observation.stop();

        // Assert — the manual child should be the current span on the worker thread
        assertThat(capturedChild.get()).isNotNull();
    }

}
```

---

## Summary

| Concept | What It Means |
|---------|---------------|
| **`ThreadLocalAccessor<V>`** | Bridges a single `ThreadLocal` into the context-propagation system — implementations define how to capture, restore, and clear values |
| **`ObservationAwareSpanThreadLocalAccessor`** | Captures the current span for cross-thread propagation, yielding when `ObservationThreadLocalAccessor` already manages it |
| **`ObservationAwareBaggageThreadLocalAccessor`** | Captures baggage entries and opens `BaggageInScope` on worker threads, using `Consumer.andThen()` composition for cleanup |
| **`BaggageToPropagate`** | Value type wrapping `Map<String, String>` baggage entries for snapshot storage |
| **SpanAction linked list** | Per-thread singly-linked list (`ConcurrentHashMap<Thread, SpanAction>`) supporting nested set/restore cycles |
| **Observation awareness** | Coordination pattern: check `TracingContext.getSpan()` vs `tracer.currentSpan()` to determine if OTLA already handles the span |

This is the final feature in the Simple Tracing project. The full stack — from core span interfaces through observation handlers to cross-thread propagation — now mirrors the architecture of the real Micrometer Tracing library.
