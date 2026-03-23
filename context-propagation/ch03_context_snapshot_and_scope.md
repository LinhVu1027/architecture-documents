# Chapter 3: ContextSnapshot and Scope

> Line references based on commit `be901c1` of the Context Propagation repository.

## Build Challenge

| Current State | Limitation | Objective |
|---------------|------------|-----------|
| A `ContextRegistry` holds all `ThreadLocalAccessor` instances, but there's no mechanism to capture their values and restore them on another thread | When work moves from thread A to thread B (via `ExecutorService`, `CompletableFuture`, etc.), all ThreadLocal values are lost — the target thread has no knowledge of the source thread's state | Build a `ContextSnapshot` that captures ThreadLocal values into a "moving box" and a `Scope` that unpacks them on the target thread with automatic restoration via try-with-resources |

---

## 3.1 The Integration Point

The integration point is `DefaultContextSnapshot.setThreadLocals()` — the exact place where the snapshot (a map of captured values) meets the registry (the list of accessors) to manipulate actual ThreadLocals on the current thread.

```
ContextRegistry                       ContextSnapshot (HashMap)
──────────────                        ─────────────────────────
ThreadLocalAccessor "user"  ─┐    ┌─  "user" → "alice"
ThreadLocalAccessor "trace" ─┼────┤   "trace" → "trace-123"
ThreadLocalAccessor "baggage"┘    └─  (not present → skip or clear)
                                 │
                     setThreadLocals()
                                 │
                                 ▼
                        ┌─────────────┐
                        │   Scope     │
                        │ (remembers  │
                        │  previous   │
                        │  values)    │
                        └──────┬──────┘
                               │
                          close() → restore in REVERSE order
```

Two key decisions:

1. **Why does the snapshot extend `HashMap` rather than hold a separate field?** The snapshot IS its data. By extending `HashMap<Object, Object>`, there's no indirection — `containsKey()`, `get()`, and `put()` operate directly on the snapshot. This is compact and avoids an extra allocation.

2. **Why forward-set, reverse-restore?** When libraries have interdependencies (e.g., a Span accessor depends on an Observation accessor), setting in forward order and restoring in reverse order ensures stack-like cleanup. The last thing set is the first thing unwound — just like `try-with-resources` blocks that close in reverse declaration order.

This connects the **ContextRegistry** (what ThreadLocals exist) to the **thread boundary crossing** (setting/restoring values on the target thread). To make it work, we need to build:
- A `ContextSnapshot` interface defining the public API (`setThreadLocals`, `wrap`, `wrapExecutor`)
- A nested `Scope` interface for the try-with-resources pattern
- A `DefaultContextSnapshot` implementation with the capture-restore algorithm
- A `DefaultScope` inner class that restores in reverse order

## 3.2 The ContextSnapshot Interface

**New file:** `src/main/java/dev/linhvu/context/ContextSnapshot.java`

```java
package dev.linhvu.context;

import java.util.concurrent.Callable;
import java.util.concurrent.Executor;
import java.util.function.Predicate;

public interface ContextSnapshot {

    Scope setThreadLocals();

    Scope setThreadLocals(Predicate<Object> keyPredicate);

    default Runnable wrap(Runnable runnable) {
        return () -> {
            try (Scope scope = setThreadLocals()) {
                runnable.run();
            }
        };
    }

    default <T> Callable<T> wrap(Callable<T> callable) {
        return () -> {
            try (Scope scope = setThreadLocals()) {
                return callable.call();
            }
        };
    }

    default Executor wrapExecutor(Executor executor) {
        return runnable -> executor.execute(wrap(runnable));
    }

    interface Scope extends AutoCloseable {

        @Override
        void close();

    }

}
```

The interface has two abstract methods (`setThreadLocals` overloads) and three default convenience methods. The `wrap` methods are syntactic sugar — they all delegate to `setThreadLocals()` inside a try-with-resources block. The `Scope` nested interface overrides `close()` to remove the checked exception, enabling clean try-with-resources usage.

## 3.3 The DefaultContextSnapshot Implementation

**New file:** `src/main/java/dev/linhvu/context/DefaultContextSnapshot.java`

```java
package dev.linhvu.context;

import java.util.HashMap;
import java.util.List;
import java.util.Map;
import java.util.function.Predicate;

final class DefaultContextSnapshot extends HashMap<Object, Object> implements ContextSnapshot {

    private final ContextRegistry contextRegistry;

    private final boolean clearMissing;

    DefaultContextSnapshot(ContextRegistry contextRegistry, boolean clearMissing) {
        this.contextRegistry = contextRegistry;
        this.clearMissing = clearMissing;
    }

    @Override
    public Scope setThreadLocals() {
        return setThreadLocals(key -> true);
    }

    @Override
    public Scope setThreadLocals(Predicate<Object> keyPredicate) {
        Map<Object, Object> previousValues = null;
        List<ThreadLocalAccessor<?>> accessors = this.contextRegistry.getThreadLocalAccessors();

        for (int i = 0; i < accessors.size(); i++) {
            ThreadLocalAccessor<?> accessor = accessors.get(i);
            Object key = accessor.key();

            if (keyPredicate.test(key)) {
                if (containsKey(key)) {
                    previousValues = setThreadLocal(key, get(key), accessor, previousValues);
                }
                else if (this.clearMissing) {
                    previousValues = clearThreadLocal(key, accessor, previousValues);
                }
            }
        }

        return DefaultScope.from(previousValues, this.contextRegistry);
    }

    @SuppressWarnings("unchecked")
    static <V> Map<Object, Object> setThreadLocal(Object key, V value,
            ThreadLocalAccessor<?> accessor, Map<Object, Object> previousValues) {

        if (previousValues == null) {
            previousValues = new HashMap<>();
        }
        previousValues.put(key, accessor.getValue());
        ((ThreadLocalAccessor<V>) accessor).setValue(value);
        return previousValues;
    }

    static Map<Object, Object> clearThreadLocal(Object key,
            ThreadLocalAccessor<?> accessor, Map<Object, Object> previousValues) {

        if (previousValues == null) {
            previousValues = new HashMap<>();
        }
        previousValues.put(key, accessor.getValue());
        accessor.setValue();
        return previousValues;
    }

    @Override
    public String toString() {
        return "DefaultContextSnapshot" + super.toString();
    }

    static class DefaultScope implements Scope {

        private final Map<Object, Object> previousValues;

        private final ContextRegistry contextRegistry;

        DefaultScope(Map<Object, Object> previousValues, ContextRegistry contextRegistry) {
            this.previousValues = previousValues;
            this.contextRegistry = contextRegistry;
        }

        @Override
        public void close() {
            List<ThreadLocalAccessor<?>> accessors = this.contextRegistry.getThreadLocalAccessors();
            for (int i = accessors.size() - 1; i >= 0; --i) {
                ThreadLocalAccessor<?> accessor = accessors.get(i);
                if (this.previousValues.containsKey(accessor.key())) {
                    Object previousValue = this.previousValues.get(accessor.key());
                    resetThreadLocalValue(accessor, previousValue);
                }
            }
        }

        @SuppressWarnings("unchecked")
        private <V> void resetThreadLocalValue(ThreadLocalAccessor<?> accessor, V previousValue) {
            if (previousValue != null) {
                ((ThreadLocalAccessor<V>) accessor).restore(previousValue);
            }
            else {
                accessor.restore();
            }
        }

        static Scope from(Map<Object, Object> previousValues, ContextRegistry registry) {
            if (previousValues != null) {
                return new DefaultScope(previousValues, registry);
            }
            return () -> {
            };
        }

    }

}
```

Let's trace through `setThreadLocals()` step by step:

1. **`previousValues` starts as `null`** — lazy allocation avoids creating a map when nothing is modified
2. **Forward iteration** through all registered accessors (index 0, 1, 2...)
3. For each accessor whose key matches the predicate:
   - If the snapshot **contains** a value for that key → save the current ThreadLocal value, set the new one
   - If the snapshot **does not contain** a value AND `clearMissing` is true → save and clear
   - Otherwise → skip (leave the target thread's ThreadLocal untouched)
4. **`DefaultScope.from()`** returns either a `DefaultScope` (if anything was modified) or a no-op lambda `() -> {}` (if nothing changed)
5. **On `close()`**, `DefaultScope` iterates the accessors in **reverse** order, calling `restore(previousValue)` or `restore()` for each one that was modified

## 3.4 Try It Yourself

<details>
<summary>Challenge 1: Implement setThreadLocals() with the lazy previousValues pattern</summary>

Given this skeleton, fill in the `setThreadLocals(Predicate)` method:

```java
@Override
public Scope setThreadLocals(Predicate<Object> keyPredicate) {
    Map<Object, Object> previousValues = null;
    List<ThreadLocalAccessor<?>> accessors = this.contextRegistry.getThreadLocalAccessors();

    for (int i = 0; i < accessors.size(); i++) {
        ThreadLocalAccessor<?> accessor = accessors.get(i);
        Object key = accessor.key();

        if (keyPredicate.test(key)) {
            if (containsKey(key)) {
                // TODO: save previous value and set the captured one
                previousValues = setThreadLocal(key, get(key), accessor, previousValues);
            }
            else if (this.clearMissing) {
                // TODO: save previous value and clear
                previousValues = clearThreadLocal(key, accessor, previousValues);
            }
        }
    }

    return DefaultScope.from(previousValues, this.contextRegistry);
}
```

The key insight: `previousValues` is lazily allocated by the helper methods. If no accessor's key matches the predicate (or the snapshot is empty), `previousValues` stays `null` and `DefaultScope.from()` returns a no-op scope.

</details>

<details>
<summary>Challenge 2: Implement DefaultScope.close() with reverse-order iteration</summary>

Fill in the `close()` method that restores ThreadLocals in reverse order:

```java
@Override
public void close() {
    List<ThreadLocalAccessor<?>> accessors = this.contextRegistry.getThreadLocalAccessors();
    // Iterate in REVERSE — last set is first restored
    for (int i = accessors.size() - 1; i >= 0; --i) {
        ThreadLocalAccessor<?> accessor = accessors.get(i);
        if (this.previousValues.containsKey(accessor.key())) {
            Object previousValue = this.previousValues.get(accessor.key());
            resetThreadLocalValue(accessor, previousValue);
        }
    }
}
```

The reverse iteration ensures stack-like unwinding. Note that we check `containsKey()` rather than just `get()` because the previous value CAN be `null` (meaning the ThreadLocal wasn't set before the scope opened).

</details>

## 3.5 Tests

### Unit Tests

**New file:** `src/test/java/dev/linhvu/context/ContextSnapshotTest.java`

```java
package dev.linhvu.context;

import java.util.ArrayList;
import java.util.List;
import java.util.concurrent.Callable;
import java.util.concurrent.Executor;
import java.util.concurrent.atomic.AtomicReference;

import org.junit.jupiter.api.AfterEach;
import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;

class ContextSnapshotTest {

    private final ThreadLocal<String> threadLocal1 = new ThreadLocal<>();

    private final ThreadLocal<String> threadLocal2 = new ThreadLocal<>();

    private final ContextRegistry registry = new ContextRegistry();

    @AfterEach
    void cleanup() {
        threadLocal1.remove();
        threadLocal2.remove();
    }

    @Test
    void shouldSetThreadLocalValue_WhenSnapshotContainsKey() {
        registry.registerThreadLocalAccessor(new TestThreadLocalAccessor("key1", threadLocal1));
        DefaultContextSnapshot snapshot = new DefaultContextSnapshot(registry, false);
        snapshot.put("key1", "captured-value");

        ContextSnapshot.Scope scope = snapshot.setThreadLocals();

        assertThat(threadLocal1.get()).isEqualTo("captured-value");
        scope.close();
    }

    @Test
    void shouldRestoreThreadLocals_WhenScopeIsClosed() {
        registry.registerThreadLocalAccessor(new TestThreadLocalAccessor("key1", threadLocal1));
        threadLocal1.set("previous-value");

        DefaultContextSnapshot snapshot = new DefaultContextSnapshot(registry, false);
        snapshot.put("key1", "captured-value");

        try (ContextSnapshot.Scope scope = snapshot.setThreadLocals()) {
            assertThat(threadLocal1.get()).isEqualTo("captured-value");
        }

        assertThat(threadLocal1.get()).isEqualTo("previous-value");
    }

    @Test
    void shouldRestoreToNull_WhenNoPreviousValueExisted() {
        registry.registerThreadLocalAccessor(new TestThreadLocalAccessor("key1", threadLocal1));

        DefaultContextSnapshot snapshot = new DefaultContextSnapshot(registry, false);
        snapshot.put("key1", "captured-value");

        try (ContextSnapshot.Scope scope = snapshot.setThreadLocals()) {
            assertThat(threadLocal1.get()).isEqualTo("captured-value");
        }

        assertThat(threadLocal1.get()).isNull();
    }

    @Test
    void shouldNotModifyThreadLocal_WhenSnapshotDoesNotContainKeyAndClearMissingFalse() {
        registry.registerThreadLocalAccessor(new TestThreadLocalAccessor("key1", threadLocal1));
        threadLocal1.set("existing-value");

        DefaultContextSnapshot snapshot = new DefaultContextSnapshot(registry, false);

        try (ContextSnapshot.Scope scope = snapshot.setThreadLocals()) {
            assertThat(threadLocal1.get()).isEqualTo("existing-value");
        }

        assertThat(threadLocal1.get()).isEqualTo("existing-value");
    }

    @Test
    void shouldClearThreadLocal_WhenClearMissingEnabled() {
        registry.registerThreadLocalAccessor(new TestThreadLocalAccessor("key1", threadLocal1));
        threadLocal1.set("existing-value");

        DefaultContextSnapshot snapshot = new DefaultContextSnapshot(registry, true);

        try (ContextSnapshot.Scope scope = snapshot.setThreadLocals()) {
            assertThat(threadLocal1.get()).isNull();
        }

        assertThat(threadLocal1.get()).isEqualTo("existing-value");
    }

    @Test
    void shouldRestoreInReverseOrder_WhenMultipleAccessorsRegistered() {
        List<String> restoreOrder = new ArrayList<>();
        // ... (registers two accessors that track restore order)
        // Assert: restoreOrder is ["second", "first"] — reverse of registration
    }

    @Test
    void shouldFilterByKeyPredicate_WhenPredicateProvided() {
        // Only "key1" is set; "key2" is untouched despite being in the snapshot
    }

    @Test
    void shouldReturnNoOpScope_WhenNoThreadLocalsModified() {
        // Empty snapshot → no-op scope (not a DefaultScope instance)
    }

    @Test
    void shouldWrapRunnable_WhenWrapCalled() {
        // Wrapped Runnable sees the propagated value; restored after run
    }

    @Test
    void shouldWrapCallable_WhenWrapCalled() {
        // Wrapped Callable returns the propagated value; restored after call
    }

    @Test
    void shouldWrapExecutor_WhenWrapExecutorCalled() {
        // Wrapped Executor propagates values to submitted tasks
    }

    @Test
    void shouldIncludeClassName_WhenToStringCalled() {
        // toString starts with "DefaultContextSnapshot" and includes entries
    }
}
```

The tests cover: basic set/restore, clearMissing behavior, reverse-order restoration, key predicate filtering, no-op scope optimization, wrap(Runnable), wrap(Callable), wrapExecutor, and toString. See the Complete Code section (3.8) for the full test implementations.

### Integration Tests

**New file:** `src/test/java/dev/linhvu/context/ContextSnapshotIntegrationTest.java`

```java
class ContextSnapshotIntegrationTest {

    @Test
    void shouldPropagateAcrossThreads_WhenSnapshotUsedWithExecutor() {
        // Register accessor, set value on source thread, create snapshot,
        // submit wrapped Runnable to ExecutorService, verify value propagated
    }

    @Test
    void shouldPropagateMultipleAccessors_WhenRegisteredInRegistry() {
        // Two accessors, two ThreadLocals — both propagated to worker thread
    }

    @Test
    void shouldRestoreWorkerThreadState_WhenScopeClosed() {
        // Worker thread has pre-existing value → scope restores it after close
    }

    @Test
    void shouldClearMissingOnWorkerThread_WhenClearMissingEnabled() {
        // Worker thread has value, snapshot is empty, clearMissing=true
        // → value cleared during scope, restored on close
    }
}
```

These integration tests use real `ExecutorService` instances to verify cross-thread propagation. See the Complete Code section (3.8) for the full implementations.

**Run:** `./gradlew test` — expected: all 38 tests pass (9 + 13 + 12 + 4)

---

## 3.6 Why This Works

> ★ **Insight** -------------------------------------------
> **Lazy allocation prevents unnecessary work.** The `previousValues` map starts as `null` and is only allocated when the first ThreadLocal is actually modified. If the snapshot is empty or no accessor keys match the predicate, `DefaultScope.from(null, registry)` returns a no-op lambda `() -> {}` — zero allocations for a no-op propagation. In high-throughput servers, this matters: a snapshot might be captured speculatively (e.g., by middleware) but never apply if no context was present.
> -----------------------------------------------------------

> ★ **Insight** -------------------------------------------
> **`setValue` vs `restore` enables structured scope cleanup.** A naive implementation might just call `setValue(previousValue)` on scope close. But the real framework distinguishes between "opening a scope" (`setValue`) and "closing a scope" (`restore`). This is critical for Micrometer: when `ObservationThreadLocalAccessor.restore()` is called, it can stop the observation's timer and close the associated scope — operations that are different from simply reassigning the ThreadLocal. Without this distinction, context propagation would break observation lifecycle management.
> -----------------------------------------------------------

> ★ **Insight** -------------------------------------------
> **`clearMissing` solves the "stale context" problem.** Consider: Thread A has `user=alice` and `trace=123`. Thread B already has a stale `trace=old-456` from a previous request. When a snapshot from Thread A propagates `user=alice` (but `trace` isn't captured because the source had no active trace), Thread B would see both `user=alice` AND the stale `trace=old-456`. With `clearMissing=true`, registered ThreadLocals that are absent from the snapshot are actively cleared — Thread B sees `user=alice` and `trace=null`. This is why `ContextSnapshotFactory` defaults to `clearMissing=false` but allows enabling it.
> -----------------------------------------------------------

## 3.7 What We Enhanced

| Aspect | Before (ch02) | Current (ch03) | Real Framework |
|--------|---------------|----------------|----------------|
| **Context capture** | Registry holds accessors but can't read their values into a portable container | `DefaultContextSnapshot` stores captured values in a `HashMap`, portable across threads | Same approach — `DefaultContextSnapshot extends HashMap<Object, Object>` (`DefaultContextSnapshot.java:33`) |
| **Thread restoration** | Accessors have `setValue`/`restore` methods but nothing orchestrates calling them across all accessors | `setThreadLocals()` iterates all accessors, sets values, returns a `Scope` | Same algorithm with additional support for `ContextAccessor` (reactive context) (`DefaultContextSnapshot.java:78`) |
| **Scope lifecycle** | Manual set/restore — error-prone, no guarantee of cleanup on exceptions | `Scope` implements `AutoCloseable` — try-with-resources guarantees cleanup | Same pattern (`ContextSnapshot.java:307`) |
| **Convenience wrappers** | None — caller must manually manage scope open/close | `wrap(Runnable)`, `wrap(Callable)`, `wrapExecutor(Executor)` handle scope lifecycle | Same defaults on the interface, plus `wrap(Consumer)` (`ContextSnapshot.java:88`) |

## 3.8 Connection to Real Framework

| Simplified Code | Real Framework Code | File:Line | Key Difference |
|----------------|--------------------|-----------| ---------------|
| `ContextSnapshot` interface | `ContextSnapshot` | `ContextSnapshot.java:44` | Real version adds `updateContext(C)` for reactive `ContextAccessor` and deprecated static factory methods |
| `setThreadLocals()` | `setThreadLocals()` | `ContextSnapshot.java:70` | Identical contract |
| `wrap(Runnable)` | `wrap(Runnable)` | `ContextSnapshot.java:88` | Identical implementation |
| `wrapExecutor(Executor)` | `wrapExecutor(Executor)` | `ContextSnapshot.java:134` | Identical implementation |
| `Scope` interface | `Scope` | `ContextSnapshot.java:307` | Identical — nested interface extending `AutoCloseable` |
| `DefaultContextSnapshot` | `DefaultContextSnapshot` | `DefaultContextSnapshot.java:33` | Real version handles `ContextAccessor` in `updateContext` methods; also has JSR-305 nullability annotations |
| `setThreadLocals(Predicate)` impl | `setThreadLocals(Predicate)` | `DefaultContextSnapshot.java:78` | Identical algorithm — forward iteration, lazy previousValues, clearMissing branching |
| `setThreadLocal` helper | `setThreadLocal` | `DefaultContextSnapshot.java:99` | Identical — save previous, set new, return map |
| `clearThreadLocal` helper | `clearThreadLocal` | `DefaultContextSnapshot.java:109` | Identical — save previous, call `setValue()` (no-arg) |
| `DefaultScope` | `DefaultScope` | `DefaultContextSnapshot.java:125` | Identical — reverse iteration on `close()`, `from()` factory with null check |
| `DefaultScope.close()` | `close()` | `DefaultContextSnapshot.java:137` | Identical reverse-order restoration loop |
| `DefaultScope.from()` | `from()` | `DefaultContextSnapshot.java:158` | Identical — returns no-op lambda when `previousValues` is null |

## 3.9 Complete Code

### Production Code

#### File: `src/main/java/dev/linhvu/context/ContextSnapshot.java` [NEW]

```java
package dev.linhvu.context;

import java.util.concurrent.Callable;
import java.util.concurrent.Executor;
import java.util.function.Predicate;

/**
 * Holds values captured from {@link ThreadLocal}s and exposes methods to
 * propagate them to other threads.
 *
 * <p>A snapshot is created (typically by {@code ContextSnapshotFactory}) by
 * reading the current values of all registered {@link ThreadLocalAccessor}s.
 * It can then be used to set those values on any thread via
 * {@link #setThreadLocals()}, which returns a {@link Scope} that restores the
 * previous values when closed.
 *
 * <p>Implementations must not store null values. If a ThreadLocal has no value
 * (i.e., {@code getValue()} returns null), it is simply absent from the
 * snapshot.
 *
 * @see ThreadLocalAccessor
 * @see ContextRegistry
 */
public interface ContextSnapshot {

    /**
     * Set ThreadLocal values from this snapshot and return a {@link Scope}
     * that restores the previous values on {@link Scope#close()}.
     *
     * <p>Usage:
     * <pre>{@code
     * try (Scope scope = snapshot.setThreadLocals()) {
     *     // ThreadLocals are set to snapshot values
     *     doWork();
     * }
     * // ThreadLocals are restored to their previous values
     * }</pre>
     */
    Scope setThreadLocals();

    /**
     * Set ThreadLocal values from this snapshot, but only for keys matching
     * the given predicate.
     *
     * @param keyPredicate filter to select which keys to set
     * @return a Scope that restores the previous values on close
     */
    Scope setThreadLocals(Predicate<Object> keyPredicate);

    /**
     * Wrap a {@link Runnable} so that this snapshot's ThreadLocal values are
     * set before execution and restored afterward.
     */
    default Runnable wrap(Runnable runnable) {
        return () -> {
            try (Scope scope = setThreadLocals()) {
                runnable.run();
            }
        };
    }

    /**
     * Wrap a {@link Callable} so that this snapshot's ThreadLocal values are
     * set before execution and restored afterward.
     */
    default <T> Callable<T> wrap(Callable<T> callable) {
        return () -> {
            try (Scope scope = setThreadLocals()) {
                return callable.call();
            }
        };
    }

    /**
     * Wrap an {@link Executor} so that every task it runs has this snapshot's
     * ThreadLocal values set.
     *
     * <p>For a more complete wrapper, see {@code ContextExecutorService} which
     * captures a fresh snapshot per task submission.
     */
    default Executor wrapExecutor(Executor executor) {
        return runnable -> executor.execute(wrap(runnable));
    }

    /**
     * A scope that, when closed, restores ThreadLocal values to the state
     * they were in before {@link ContextSnapshot#setThreadLocals()} was called.
     *
     * <p>Extends {@link AutoCloseable} so it can be used in a
     * try-with-resources block. The {@link #close()} method is redeclared
     * without a checked exception.
     */
    interface Scope extends AutoCloseable {

        @Override
        void close();

    }

}
```

#### File: `src/main/java/dev/linhvu/context/DefaultContextSnapshot.java` [NEW]

```java
package dev.linhvu.context;

import java.util.HashMap;
import java.util.List;
import java.util.Map;
import java.util.function.Predicate;

/**
 * Default implementation of {@link ContextSnapshot} backed by a {@link HashMap}.
 *
 * <p>This class extends {@code HashMap<Object, Object>} directly — the snapshot
 * IS the map. Captured ThreadLocal values are stored as entries where the key
 * is the {@link ThreadLocalAccessor#key()} and the value is the captured value.
 *
 * <p>The {@code clearMissing} flag controls behavior for registered ThreadLocals
 * that are NOT present in the snapshot:
 * <ul>
 *   <li>{@code false} (default) — leave them untouched on the target thread</li>
 *   <li>{@code true} — actively clear them via {@code accessor.setValue()}</li>
 * </ul>
 */
final class DefaultContextSnapshot extends HashMap<Object, Object> implements ContextSnapshot {

    private final ContextRegistry contextRegistry;

    private final boolean clearMissing;

    DefaultContextSnapshot(ContextRegistry contextRegistry, boolean clearMissing) {
        this.contextRegistry = contextRegistry;
        this.clearMissing = clearMissing;
    }

    @Override
    public Scope setThreadLocals() {
        return setThreadLocals(key -> true);
    }

    /**
     * Set ThreadLocal values from this snapshot for keys matching the predicate.
     *
     * <p>Algorithm:
     * <ol>
     *   <li>Iterate all registered accessors in <b>forward</b> order</li>
     *   <li>For each matching key: save the current (previous) value, then set the new value</li>
     *   <li>Return a {@link DefaultScope} that restores in <b>reverse</b> order on close</li>
     * </ol>
     *
     * <p>The lazy allocation of {@code previousValues} avoids allocating a map (and a
     * Scope object) when no ThreadLocals are actually modified — a no-op lambda is
     * returned instead.
     */
    @Override
    public Scope setThreadLocals(Predicate<Object> keyPredicate) {
        Map<Object, Object> previousValues = null;
        List<ThreadLocalAccessor<?>> accessors = this.contextRegistry.getThreadLocalAccessors();

        for (int i = 0; i < accessors.size(); i++) {
            ThreadLocalAccessor<?> accessor = accessors.get(i);
            Object key = accessor.key();

            if (keyPredicate.test(key)) {
                if (containsKey(key)) {
                    previousValues = setThreadLocal(key, get(key), accessor, previousValues);
                }
                else if (this.clearMissing) {
                    previousValues = clearThreadLocal(key, accessor, previousValues);
                }
            }
        }

        return DefaultScope.from(previousValues, this.contextRegistry);
    }

    /**
     * Save the current ThreadLocal value and set the new one.
     */
    @SuppressWarnings("unchecked")
    static <V> Map<Object, Object> setThreadLocal(Object key, V value,
            ThreadLocalAccessor<?> accessor, Map<Object, Object> previousValues) {

        if (previousValues == null) {
            previousValues = new HashMap<>();
        }
        previousValues.put(key, accessor.getValue());
        ((ThreadLocalAccessor<V>) accessor).setValue(value);
        return previousValues;
    }

    /**
     * Save the current ThreadLocal value and clear it.
     */
    static Map<Object, Object> clearThreadLocal(Object key,
            ThreadLocalAccessor<?> accessor, Map<Object, Object> previousValues) {

        if (previousValues == null) {
            previousValues = new HashMap<>();
        }
        previousValues.put(key, accessor.getValue());
        accessor.setValue();
        return previousValues;
    }

    @Override
    public String toString() {
        return "DefaultContextSnapshot" + super.toString();
    }

    /**
     * Default implementation of {@link Scope} that restores ThreadLocal values
     * in <b>reverse</b> order when closed.
     *
     * <p>The reverse iteration mirrors the forward iteration in
     * {@link DefaultContextSnapshot#setThreadLocals(Predicate)} — the last
     * ThreadLocal that was set is the first to be restored. This stack-like
     * unwinding is essential for correct behavior when ThreadLocalAccessors
     * have interdependencies (e.g., a Span accessor that depends on an
     * Observation accessor).
     */
    static class DefaultScope implements Scope {

        private final Map<Object, Object> previousValues;

        private final ContextRegistry contextRegistry;

        DefaultScope(Map<Object, Object> previousValues, ContextRegistry contextRegistry) {
            this.previousValues = previousValues;
            this.contextRegistry = contextRegistry;
        }

        @Override
        public void close() {
            List<ThreadLocalAccessor<?>> accessors = this.contextRegistry.getThreadLocalAccessors();
            for (int i = accessors.size() - 1; i >= 0; --i) {
                ThreadLocalAccessor<?> accessor = accessors.get(i);
                if (this.previousValues.containsKey(accessor.key())) {
                    Object previousValue = this.previousValues.get(accessor.key());
                    resetThreadLocalValue(accessor, previousValue);
                }
            }
        }

        @SuppressWarnings("unchecked")
        private <V> void resetThreadLocalValue(ThreadLocalAccessor<?> accessor, V previousValue) {
            if (previousValue != null) {
                ((ThreadLocalAccessor<V>) accessor).restore(previousValue);
            }
            else {
                accessor.restore();
            }
        }

        /**
         * Create a Scope, or return a no-op lambda if nothing was modified.
         */
        static Scope from(Map<Object, Object> previousValues, ContextRegistry registry) {
            if (previousValues != null) {
                return new DefaultScope(previousValues, registry);
            }
            return () -> {
            };
        }

    }

}
```

### Test Code

#### File: `src/test/java/dev/linhvu/context/ContextSnapshotTest.java` [NEW]

```java
package dev.linhvu.context;

import java.util.ArrayList;
import java.util.List;
import java.util.concurrent.Callable;
import java.util.concurrent.Executor;
import java.util.concurrent.atomic.AtomicReference;

import org.junit.jupiter.api.AfterEach;
import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;

class ContextSnapshotTest {

    private final ThreadLocal<String> threadLocal1 = new ThreadLocal<>();

    private final ThreadLocal<String> threadLocal2 = new ThreadLocal<>();

    private final ContextRegistry registry = new ContextRegistry();

    @AfterEach
    void cleanup() {
        threadLocal1.remove();
        threadLocal2.remove();
    }

    // --- setThreadLocals: basic capture and restore ---

    @Test
    void shouldSetThreadLocalValue_WhenSnapshotContainsKey() {
        // Arrange
        registry.registerThreadLocalAccessor(new TestThreadLocalAccessor("key1", threadLocal1));
        DefaultContextSnapshot snapshot = new DefaultContextSnapshot(registry, false);
        snapshot.put("key1", "captured-value");

        // Act
        ContextSnapshot.Scope scope = snapshot.setThreadLocals();

        // Assert
        assertThat(threadLocal1.get()).isEqualTo("captured-value");
        scope.close();
    }

    @Test
    void shouldRestoreThreadLocals_WhenScopeIsClosed() {
        // Arrange
        registry.registerThreadLocalAccessor(new TestThreadLocalAccessor("key1", threadLocal1));
        threadLocal1.set("previous-value");

        DefaultContextSnapshot snapshot = new DefaultContextSnapshot(registry, false);
        snapshot.put("key1", "captured-value");

        // Act
        try (ContextSnapshot.Scope scope = snapshot.setThreadLocals()) {
            assertThat(threadLocal1.get()).isEqualTo("captured-value");
        }

        // Assert — previous value restored
        assertThat(threadLocal1.get()).isEqualTo("previous-value");
    }

    @Test
    void shouldRestoreToNull_WhenNoPreviousValueExisted() {
        // Arrange
        registry.registerThreadLocalAccessor(new TestThreadLocalAccessor("key1", threadLocal1));
        assertThat(threadLocal1.get()).isNull();

        DefaultContextSnapshot snapshot = new DefaultContextSnapshot(registry, false);
        snapshot.put("key1", "captured-value");

        // Act
        try (ContextSnapshot.Scope scope = snapshot.setThreadLocals()) {
            assertThat(threadLocal1.get()).isEqualTo("captured-value");
        }

        // Assert — restored to null (no previous value)
        assertThat(threadLocal1.get()).isNull();
    }

    // --- setThreadLocals: clearMissing ---

    @Test
    void shouldNotModifyThreadLocal_WhenSnapshotDoesNotContainKeyAndClearMissingFalse() {
        // Arrange
        registry.registerThreadLocalAccessor(new TestThreadLocalAccessor("key1", threadLocal1));
        threadLocal1.set("existing-value");

        DefaultContextSnapshot snapshot = new DefaultContextSnapshot(registry, false);
        // snapshot does NOT contain "key1"

        // Act
        try (ContextSnapshot.Scope scope = snapshot.setThreadLocals()) {
            // Assert — value is untouched
            assertThat(threadLocal1.get()).isEqualTo("existing-value");
        }

        // Still untouched after scope close
        assertThat(threadLocal1.get()).isEqualTo("existing-value");
    }

    @Test
    void shouldClearThreadLocal_WhenClearMissingEnabled() {
        // Arrange
        registry.registerThreadLocalAccessor(new TestThreadLocalAccessor("key1", threadLocal1));
        threadLocal1.set("existing-value");

        DefaultContextSnapshot snapshot = new DefaultContextSnapshot(registry, true);
        // snapshot does NOT contain "key1" — clearMissing will clear it

        // Act
        try (ContextSnapshot.Scope scope = snapshot.setThreadLocals()) {
            // Assert — value was cleared
            assertThat(threadLocal1.get()).isNull();
        }

        // Assert — previous value restored after scope close
        assertThat(threadLocal1.get()).isEqualTo("existing-value");
    }

    // --- setThreadLocals: reverse-order restoration ---

    @Test
    void shouldRestoreInReverseOrder_WhenMultipleAccessorsRegistered() {
        // Arrange — track the order in which restore is called
        List<String> restoreOrder = new ArrayList<>();

        registry.registerThreadLocalAccessor(new ThreadLocalAccessor<String>() {
            @Override
            public Object key() {
                return "first";
            }

            @Override
            public String getValue() {
                return threadLocal1.get();
            }

            @Override
            public void setValue(String value) {
                threadLocal1.set(value);
            }

            @Override
            public void setValue() {
                threadLocal1.remove();
            }

            @Override
            public void restore(String previousValue) {
                restoreOrder.add("first");
                threadLocal1.set(previousValue);
            }

            @Override
            public void restore() {
                restoreOrder.add("first");
                threadLocal1.remove();
            }
        });

        registry.registerThreadLocalAccessor(new ThreadLocalAccessor<String>() {
            @Override
            public Object key() {
                return "second";
            }

            @Override
            public String getValue() {
                return threadLocal2.get();
            }

            @Override
            public void setValue(String value) {
                threadLocal2.set(value);
            }

            @Override
            public void setValue() {
                threadLocal2.remove();
            }

            @Override
            public void restore(String previousValue) {
                restoreOrder.add("second");
                threadLocal2.set(previousValue);
            }

            @Override
            public void restore() {
                restoreOrder.add("second");
                threadLocal2.remove();
            }
        });

        DefaultContextSnapshot snapshot = new DefaultContextSnapshot(registry, false);
        snapshot.put("first", "value-A");
        snapshot.put("second", "value-B");

        // Act
        try (ContextSnapshot.Scope scope = snapshot.setThreadLocals()) {
            // values are set in forward order
            assertThat(threadLocal1.get()).isEqualTo("value-A");
            assertThat(threadLocal2.get()).isEqualTo("value-B");
        }

        // Assert — restore happens in reverse order: "second" before "first"
        assertThat(restoreOrder).containsExactly("second", "first");
    }

    // --- setThreadLocals: key predicate filtering ---

    @Test
    void shouldFilterByKeyPredicate_WhenPredicateProvided() {
        // Arrange
        registry.registerThreadLocalAccessor(new TestThreadLocalAccessor("key1", threadLocal1));
        registry.registerThreadLocalAccessor(new TestThreadLocalAccessor("key2", threadLocal2));

        DefaultContextSnapshot snapshot = new DefaultContextSnapshot(registry, false);
        snapshot.put("key1", "value1");
        snapshot.put("key2", "value2");

        // Act — only set "key1"
        try (ContextSnapshot.Scope scope = snapshot.setThreadLocals(key -> "key1".equals(key))) {
            // Assert — key1 is set, key2 is untouched
            assertThat(threadLocal1.get()).isEqualTo("value1");
            assertThat(threadLocal2.get()).isNull();
        }

        // key1 restored, key2 still null
        assertThat(threadLocal1.get()).isNull();
        assertThat(threadLocal2.get()).isNull();
    }

    // --- No-op scope optimization ---

    @Test
    void shouldReturnNoOpScope_WhenNoThreadLocalsModified() {
        // Arrange — registry has an accessor, but snapshot has no matching key
        registry.registerThreadLocalAccessor(new TestThreadLocalAccessor("key1", threadLocal1));
        threadLocal1.set("untouched");

        DefaultContextSnapshot snapshot = new DefaultContextSnapshot(registry, false);
        // snapshot is empty — no keys match

        // Act
        ContextSnapshot.Scope scope = snapshot.setThreadLocals();

        // Assert — scope is the no-op lambda, not a DefaultScope
        assertThat(scope).isNotInstanceOf(DefaultContextSnapshot.DefaultScope.class);
        scope.close(); // should be safe to call

        // Value was never modified
        assertThat(threadLocal1.get()).isEqualTo("untouched");
    }

    // --- wrap(Runnable) ---

    @Test
    void shouldWrapRunnable_WhenWrapCalled() {
        // Arrange
        registry.registerThreadLocalAccessor(new TestThreadLocalAccessor("key1", threadLocal1));

        DefaultContextSnapshot snapshot = new DefaultContextSnapshot(registry, false);
        snapshot.put("key1", "propagated");

        AtomicReference<String> valueSeenInRunnable = new AtomicReference<>();

        // Act
        Runnable wrapped = snapshot.wrap(() -> valueSeenInRunnable.set(threadLocal1.get()));
        wrapped.run();

        // Assert
        assertThat(valueSeenInRunnable.get()).isEqualTo("propagated");
        // ThreadLocal is restored after wrap completes
        assertThat(threadLocal1.get()).isNull();
    }

    // --- wrap(Callable) ---

    @Test
    void shouldWrapCallable_WhenWrapCalled() throws Exception {
        // Arrange
        registry.registerThreadLocalAccessor(new TestThreadLocalAccessor("key1", threadLocal1));

        DefaultContextSnapshot snapshot = new DefaultContextSnapshot(registry, false);
        snapshot.put("key1", "propagated");

        // Act
        Callable<String> wrapped = snapshot.wrap(() -> threadLocal1.get());
        String result = wrapped.call();

        // Assert
        assertThat(result).isEqualTo("propagated");
        assertThat(threadLocal1.get()).isNull();
    }

    // --- wrapExecutor ---

    @Test
    void shouldWrapExecutor_WhenWrapExecutorCalled() {
        // Arrange
        registry.registerThreadLocalAccessor(new TestThreadLocalAccessor("key1", threadLocal1));

        DefaultContextSnapshot snapshot = new DefaultContextSnapshot(registry, false);
        snapshot.put("key1", "propagated");

        AtomicReference<String> valueSeenInTask = new AtomicReference<>();

        // Use a direct executor (runs on same thread) for simplicity
        Executor directExecutor = Runnable::run;

        // Act
        Executor wrapped = snapshot.wrapExecutor(directExecutor);
        wrapped.execute(() -> valueSeenInTask.set(threadLocal1.get()));

        // Assert
        assertThat(valueSeenInTask.get()).isEqualTo("propagated");
        assertThat(threadLocal1.get()).isNull();
    }

    // --- toString ---

    @Test
    void shouldIncludeClassName_WhenToStringCalled() {
        DefaultContextSnapshot snapshot = new DefaultContextSnapshot(registry, false);
        snapshot.put("key1", "value1");

        assertThat(snapshot.toString()).startsWith("DefaultContextSnapshot");
        assertThat(snapshot.toString()).contains("key1=value1");
    }

}
```

#### File: `src/test/java/dev/linhvu/context/ContextSnapshotIntegrationTest.java` [NEW]

```java
package dev.linhvu.context;

import java.util.concurrent.CountDownLatch;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.atomic.AtomicReference;

import org.junit.jupiter.api.AfterEach;
import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;

/**
 * Integration tests verifying that ContextSnapshot works end-to-end with
 * ContextRegistry and ThreadLocalAccessor to propagate context across threads.
 */
class ContextSnapshotIntegrationTest {

    private final ThreadLocal<String> userThreadLocal = new ThreadLocal<>();

    private final ThreadLocal<String> traceThreadLocal = new ThreadLocal<>();

    private final ContextRegistry registry = new ContextRegistry();

    @AfterEach
    void cleanup() {
        userThreadLocal.remove();
        traceThreadLocal.remove();
    }

    @Test
    void shouldPropagateAcrossThreads_WhenSnapshotUsedWithExecutor() throws Exception {
        // Arrange — register accessors and set values on the "source" thread
        registry.registerThreadLocalAccessor(new TestThreadLocalAccessor("user", userThreadLocal));
        userThreadLocal.set("alice");

        // Capture a snapshot on the source thread
        DefaultContextSnapshot snapshot = new DefaultContextSnapshot(registry, false);
        snapshot.put("user", userThreadLocal.get());

        AtomicReference<String> valueOnWorkerThread = new AtomicReference<>();
        CountDownLatch latch = new CountDownLatch(1);

        // Act — submit wrapped task to a different thread
        ExecutorService executor = Executors.newSingleThreadExecutor();
        try {
            executor.submit(snapshot.wrap(() -> {
                valueOnWorkerThread.set(userThreadLocal.get());
                latch.countDown();
            }));
            latch.await();
        }
        finally {
            executor.shutdown();
        }

        // Assert — the worker thread saw the propagated value
        assertThat(valueOnWorkerThread.get()).isEqualTo("alice");

        // Source thread is unaffected
        assertThat(userThreadLocal.get()).isEqualTo("alice");
    }

    @Test
    void shouldPropagateMultipleAccessors_WhenRegisteredInRegistry() throws Exception {
        // Arrange — two accessors, two ThreadLocals
        registry.registerThreadLocalAccessor(new TestThreadLocalAccessor("user", userThreadLocal));
        registry.registerThreadLocalAccessor(new TestThreadLocalAccessor("trace", traceThreadLocal));
        userThreadLocal.set("bob");
        traceThreadLocal.set("trace-123");

        DefaultContextSnapshot snapshot = new DefaultContextSnapshot(registry, false);
        snapshot.put("user", userThreadLocal.get());
        snapshot.put("trace", traceThreadLocal.get());

        AtomicReference<String> userOnWorker = new AtomicReference<>();
        AtomicReference<String> traceOnWorker = new AtomicReference<>();
        CountDownLatch latch = new CountDownLatch(1);

        // Act
        ExecutorService executor = Executors.newSingleThreadExecutor();
        try {
            executor.submit(snapshot.wrap(() -> {
                userOnWorker.set(userThreadLocal.get());
                traceOnWorker.set(traceThreadLocal.get());
                latch.countDown();
            }));
            latch.await();
        }
        finally {
            executor.shutdown();
        }

        // Assert
        assertThat(userOnWorker.get()).isEqualTo("bob");
        assertThat(traceOnWorker.get()).isEqualTo("trace-123");
    }

    @Test
    void shouldRestoreWorkerThreadState_WhenScopeClosed() throws Exception {
        // Arrange — worker thread has its own pre-existing value
        registry.registerThreadLocalAccessor(new TestThreadLocalAccessor("user", userThreadLocal));

        DefaultContextSnapshot snapshot = new DefaultContextSnapshot(registry, false);
        snapshot.put("user", "propagated-value");

        AtomicReference<String> valueDuringScope = new AtomicReference<>();
        AtomicReference<String> valueAfterScope = new AtomicReference<>();
        CountDownLatch latch = new CountDownLatch(1);

        ExecutorService executor = Executors.newSingleThreadExecutor();
        try {
            // First, set a value on the worker thread
            executor.submit(() -> userThreadLocal.set("worker-original")).get();

            // Now submit the wrapped task
            executor.submit(() -> {
                try (ContextSnapshot.Scope scope = snapshot.setThreadLocals()) {
                    valueDuringScope.set(userThreadLocal.get());
                }
                valueAfterScope.set(userThreadLocal.get());
                latch.countDown();
            });
            latch.await();
        }
        finally {
            executor.shutdown();
        }

        // Assert — during scope, the propagated value was active
        assertThat(valueDuringScope.get()).isEqualTo("propagated-value");
        // After scope close, the worker's original value was restored
        assertThat(valueAfterScope.get()).isEqualTo("worker-original");
    }

    @Test
    void shouldClearMissingOnWorkerThread_WhenClearMissingEnabled() throws Exception {
        // Arrange
        registry.registerThreadLocalAccessor(new TestThreadLocalAccessor("user", userThreadLocal));

        // Snapshot does NOT contain "user" — with clearMissing=true, it should be cleared
        DefaultContextSnapshot snapshot = new DefaultContextSnapshot(registry, true);

        AtomicReference<String> valueDuringScope = new AtomicReference<>();
        AtomicReference<String> valueAfterScope = new AtomicReference<>();
        CountDownLatch latch = new CountDownLatch(1);

        ExecutorService executor = Executors.newSingleThreadExecutor();
        try {
            // Set a value on the worker thread
            executor.submit(() -> userThreadLocal.set("worker-value")).get();

            // Submit task — clearMissing should clear "user" during scope
            executor.submit(() -> {
                try (ContextSnapshot.Scope scope = snapshot.setThreadLocals()) {
                    valueDuringScope.set(userThreadLocal.get());
                }
                valueAfterScope.set(userThreadLocal.get());
                latch.countDown();
            });
            latch.await();
        }
        finally {
            executor.shutdown();
        }

        // Assert — during scope, the ThreadLocal was cleared
        assertThat(valueDuringScope.get()).isNull();
        // After scope close, the worker's original value was restored
        assertThat(valueAfterScope.get()).isEqualTo("worker-value");
    }

}
```

---

## Summary

| Concept | What It Means |
|---------|---------------|
| **ContextSnapshot** | A portable container (HashMap) of captured ThreadLocal values — the "moving box" |
| **Scope** | A try-with-resources handle that restores ThreadLocal values on `close()` — the "unpacking receipt" |
| **Forward set, reverse restore** | ThreadLocals are set in registration order and restored in reverse — stack-like unwinding for correct cleanup |
| **clearMissing** | When true, registered ThreadLocals absent from the snapshot are actively cleared (prevents stale context leaking) |
| **Lazy previousValues** | The map of saved values is only allocated when a ThreadLocal is actually modified — zero-cost no-op propagation |
| **wrap(Runnable/Callable)** | Convenience methods that handle scope lifecycle automatically via try-with-resources |
| **Pattern: Capture-Restore** | Capture state on the source thread, restore it on the target thread, then clean up — the core mechanism of context propagation |

**Next: Chapter 4 — ContextSnapshotFactory** — The recommended entry point for creating snapshots. While this chapter showed the snapshot and scope mechanics, Chapter 4 adds the `captureAll()` method that reads ThreadLocal values from all registered accessors into a snapshot, plus a Builder with `clearMissing` and `captureKeyPredicate` configuration.
