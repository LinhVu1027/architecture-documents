# Chapter 4: ContextSnapshotFactory

> Line references based on commit `be901c1` of the Context Propagation repository.

## Build Challenge

| Current State | Limitation | Objective |
|---------------|------------|-----------|
| We have `ContextSnapshot`, `ContextRegistry`, and `ThreadLocalAccessor` — but creating a snapshot requires manually constructing a `DefaultContextSnapshot` and calling `put()` for each ThreadLocal value | No single API call to capture all registered ThreadLocals at once; callers must know about `DefaultContextSnapshot` internals | Build a `ContextSnapshotFactory` that captures all registered ThreadLocal values in one `captureAll()` call, with configurable key filtering and clear-missing support |

---

## 4.1 The Integration Point

This feature introduces the **primary API** that downstream libraries (simple-micrometer, simple-tracing) and `ContextExecutorService` (ch05) will use. The factory sits between the `ContextRegistry` (which knows what to capture) and `DefaultContextSnapshot` (which stores the captured values).

**New file:** `src/main/java/dev/linhvu/context/ContextSnapshotFactory.java`

```java
public interface ContextSnapshotFactory {

    ContextSnapshot captureAll();

    static Builder builder() {
        return new DefaultContextSnapshotFactory.Builder();
    }

    interface Builder {

        ContextSnapshotFactory build();

        Builder clearMissing(boolean shouldClear);

        Builder contextRegistry(ContextRegistry contextRegistry);

        Builder captureKeyPredicate(Predicate<Object> captureKeyPredicate);

    }

}
```

Two key decisions here:

1. **Interface + static `builder()` method** rather than a concrete class with a public constructor. The `builder()` method returns `new DefaultContextSnapshotFactory.Builder()`, keeping the implementation class package-private — callers only depend on the interface.
2. **`captureAll()` takes no arguments** in our simplified version. The real framework's `captureAll(Object... contexts)` accepts external context objects (for Reactor's `Context`), but we've omitted `ContextAccessor` since we only handle ThreadLocal-based propagation.

This connects **ContextRegistry** (the source of accessors) to **DefaultContextSnapshot** (the storage). To make it work, we need to build:
- `DefaultContextSnapshotFactory` — the implementation that iterates accessors and reads values
- `Builder` — configures the factory with registry, key predicate, and clearMissing flag

## 4.2 DefaultContextSnapshotFactory

**New file:** `src/main/java/dev/linhvu/context/DefaultContextSnapshotFactory.java`

```java
final class DefaultContextSnapshotFactory implements ContextSnapshotFactory {

    private static final DefaultContextSnapshot EMPTY_SNAPSHOT = new DefaultContextSnapshot(
            new ContextRegistry(), false);

    private final ContextRegistry contextRegistry;

    private final boolean clearMissing;

    private final Predicate<Object> captureKeyPredicate;

    DefaultContextSnapshotFactory(ContextRegistry contextRegistry, boolean clearMissing,
            Predicate<Object> captureKeyPredicate) {
        this.contextRegistry = contextRegistry;
        this.clearMissing = clearMissing;
        this.captureKeyPredicate = captureKeyPredicate;
    }

    @Override
    public ContextSnapshot captureAll() {
        DefaultContextSnapshot snapshot = null;

        for (ThreadLocalAccessor<?> accessor : this.contextRegistry.getThreadLocalAccessors()) {
            if (this.captureKeyPredicate.test(accessor.key())) {
                Object value = accessor.getValue();
                if (value != null) {
                    if (snapshot == null) {
                        snapshot = new DefaultContextSnapshot(this.contextRegistry, this.clearMissing);
                    }
                    snapshot.put(accessor.key(), value);
                }
            }
        }

        if (snapshot != null) {
            return snapshot;
        }
        return this.clearMissing ? new DefaultContextSnapshot(this.contextRegistry, true) : EMPTY_SNAPSHOT;
    }
}
```

The `captureAll()` method does the actual work:
1. Iterate all `ThreadLocalAccessor`s from the registry
2. Filter by the `captureKeyPredicate`
3. Read each value via `accessor.getValue()`
4. Store non-null values in a `DefaultContextSnapshot`

Notice the **lazy allocation**: the snapshot is only created when the first non-null value is found. If all ThreadLocals are empty, a shared `EMPTY_SNAPSHOT` is returned (when `clearMissing=false`).

## 4.3 The Builder

The `Builder` is a static inner class of `DefaultContextSnapshotFactory`:

```java
static final class Builder implements ContextSnapshotFactory.Builder {

    private boolean clearMissing = false;

    private ContextRegistry contextRegistry = ContextRegistry.getInstance();

    private Predicate<Object> captureKeyPredicate = key -> true;

    Builder() {
    }

    @Override
    public ContextSnapshotFactory build() {
        return new DefaultContextSnapshotFactory(this.contextRegistry, this.clearMissing,
                this.captureKeyPredicate);
    }

    @Override
    public ContextSnapshotFactory.Builder clearMissing(boolean shouldClear) {
        this.clearMissing = shouldClear;
        return this;
    }

    @Override
    public ContextSnapshotFactory.Builder contextRegistry(ContextRegistry contextRegistry) {
        this.contextRegistry = contextRegistry;
        return this;
    }

    @Override
    public ContextSnapshotFactory.Builder captureKeyPredicate(Predicate<Object> captureKeyPredicate) {
        this.captureKeyPredicate = captureKeyPredicate;
        return this;
    }

}
```

The defaults are designed for the common case:
- **`contextRegistry`** defaults to the global singleton — most applications use a single shared registry
- **`captureKeyPredicate`** defaults to `key -> true` — capture everything
- **`clearMissing`** defaults to `false` — don't touch ThreadLocals that aren't in the snapshot

## 4.4 Try It Yourself

<details>
<summary>Challenge: Implement captureAll() with lazy snapshot allocation</summary>

Given this skeleton, fill in the `captureAll()` method. Requirements:
- Iterate all accessors from the registry
- Only capture keys that pass `captureKeyPredicate`
- Skip null values
- Lazily allocate the snapshot (don't create it until you find a non-null value)
- When nothing is captured: return an empty snapshot (clearMissing-aware)

```java
@Override
public ContextSnapshot captureAll() {
    DefaultContextSnapshot snapshot = null;

    for (ThreadLocalAccessor<?> accessor : this.contextRegistry.getThreadLocalAccessors()) {
        if (this.captureKeyPredicate.test(accessor.key())) {
            Object value = accessor.getValue();
            if (value != null) {
                if (snapshot == null) {
                    snapshot = new DefaultContextSnapshot(this.contextRegistry, this.clearMissing);
                }
                snapshot.put(accessor.key(), value);
            }
        }
    }

    if (snapshot != null) {
        return snapshot;
    }
    return this.clearMissing ? new DefaultContextSnapshot(this.contextRegistry, true) : EMPTY_SNAPSHOT;
}
```

</details>

<details>
<summary>Challenge: Why can't EMPTY_SNAPSHOT be shared when clearMissing=true?</summary>

When `clearMissing=false`, an empty snapshot does nothing during `setThreadLocals()` — no keys match, so no ThreadLocals are touched. All empty snapshots are functionally identical, making a static singleton safe.

When `clearMissing=true`, an empty snapshot must **clear all registered ThreadLocals** during `setThreadLocals()`. To do that, it needs access to the correct `ContextRegistry` to know which accessors exist. A static singleton uses an empty registry, so it would clear nothing — the wrong behavior. That's why `clearMissing=true` always creates a fresh `DefaultContextSnapshot` with the actual registry.

</details>

## 4.5 Tests

### Unit Tests

**New file:** `src/test/java/dev/linhvu/context/ContextSnapshotFactoryTest.java`

```java
class ContextSnapshotFactoryTest {

    private final ThreadLocal<String> threadLocal1 = new ThreadLocal<>();
    private final ThreadLocal<String> threadLocal2 = new ThreadLocal<>();
    private final ContextRegistry registry = new ContextRegistry();

    @AfterEach
    void cleanup() {
        threadLocal1.remove();
        threadLocal2.remove();
    }

    @Test
    void shouldCaptureThreadLocalValue_WhenAccessorHasValue() {
        registry.registerThreadLocalAccessor(new TestThreadLocalAccessor("key1", threadLocal1));
        threadLocal1.set("hello");

        ContextSnapshotFactory factory = ContextSnapshotFactory.builder()
            .contextRegistry(registry)
            .build();

        ContextSnapshot snapshot = factory.captureAll();

        threadLocal1.remove();
        try (ContextSnapshot.Scope scope = snapshot.setThreadLocals()) {
            assertThat(threadLocal1.get()).isEqualTo("hello");
        }
    }

    @Test
    void shouldSkipNullValues_WhenAccessorReturnsNull() { ... }

    @Test
    void shouldFilterByKeyPredicate_WhenPredicateConfigured() { ... }

    @Test
    void shouldClearMissingThreadLocals_WhenClearMissingEnabled() { ... }

    @Test
    void shouldClearAllThreadLocals_WhenEmptySnapshotWithClearMissing() { ... }

    @Test
    void shouldUseDefaults_WhenBuilderNotConfigured() { ... }
}
```

### Integration Tests

**New file:** `src/test/java/dev/linhvu/context/ContextSnapshotFactoryIntegrationTest.java`

```java
class ContextSnapshotFactoryIntegrationTest {

    @Test
    void shouldPropagateAcrossThreads_WhenFactoryCaptures() throws Exception {
        registry.registerThreadLocalAccessor(new TestThreadLocalAccessor("user", userThreadLocal));
        userThreadLocal.set("alice");

        ContextSnapshotFactory factory = ContextSnapshotFactory.builder()
            .contextRegistry(registry)
            .build();

        ContextSnapshot snapshot = factory.captureAll();

        AtomicReference<String> valueOnWorker = new AtomicReference<>();
        ExecutorService executor = Executors.newSingleThreadExecutor();
        try {
            executor.submit(snapshot.wrap(() -> {
                valueOnWorker.set(userThreadLocal.get());
            }));
            // ...
        } finally {
            executor.shutdown();
        }

        assertThat(valueOnWorker.get()).isEqualTo("alice");
    }

    @Test
    void shouldClearMissingOnWorkerThread_WhenClearMissingEnabled() { ... }

    @Test
    void shouldFilterCapture_WhenKeyPredicateConfigured() { ... }
}
```

**Run:** `./gradlew test` — expected: all 51 tests pass (9 factory unit + 4 factory integration + 38 prior tests)

---

## 4.6 Why This Works

> ★ **Insight** -------------------------------------------
> - **Why a Factory instead of static methods on ContextSnapshot?** The real framework originally had static factory methods on `ContextSnapshot` (e.g., `ContextSnapshot.captureAll()`), but they were deprecated in favor of `ContextSnapshotFactory`. The factory approach is superior because configuration (which registry? which keys? clearMissing?) is set once at construction time and reused for every capture. With static methods, you'd either need to pass these parameters on every call or rely on global state.
> - **When to skip the factory:** If you only need a one-off snapshot for testing, directly constructing `DefaultContextSnapshot` (as ch03's tests do) is fine. The factory shines when you're creating snapshots repeatedly — like `ContextExecutorService` does on every task submission.
> -----------------------------------------------------------

> ★ **Insight** -------------------------------------------
> - **Lazy allocation is a real performance optimization.** The `captureAll()` method doesn't allocate a `HashMap` until it finds a non-null value. In production, many threads are spawned in contexts where no observations or spans are active. Without lazy allocation, every `captureAll()` call would allocate and immediately discard an empty `HashMap` — death by a thousand small GCs.
> - **The shared `EMPTY_SNAPSHOT` singleton** takes this further: when `clearMissing=false` and nothing is captured, no allocation happens at all — the same empty snapshot object is returned every time.
> -----------------------------------------------------------

> ★ **Insight** -------------------------------------------
> - **`clearMissing` exists for Micrometer's `ObservationThreadLocalAccessor`.** When `clearMissing=true` and a ThreadLocal's key is absent from the snapshot, its `setValue()` (no-arg) method is called. For the `ObservationThreadLocalAccessor`, this sets a `NullObservation` — a safe "no observation is active" sentinel rather than `null`. Without this, code that switches threads might see a stale observation from a previous task on the worker thread. This is why `clearMissing` is a factory-level setting, not a snapshot-level one — it's an application-wide policy decision.
> -----------------------------------------------------------

## 4.7 What We Enhanced

| Aspect | Before (ch03) | Current (ch04) | Real Framework |
|--------|---------------|----------------|----------------|
| Snapshot creation | Manual: construct `DefaultContextSnapshot`, call `put()` for each key/value | Automated: `factory.captureAll()` reads all registered accessors in one call | `DefaultContextSnapshotFactory.captureAll()` also captures from external `ContextAccessor`-based contexts (`DefaultContextSnapshotFactory.java:54`) |
| Configuration | `clearMissing` and registry passed to `DefaultContextSnapshot` constructor directly | Builder API: `clearMissing`, `captureKeyPredicate`, `contextRegistry` configured once, reused per capture | Same builder pattern plus `captureFrom(Object...)` and `setThreadLocalsFrom()` for reactive context (`ContextSnapshotFactory.java:39-65`) |
| Key filtering | Only at restore time via `setThreadLocals(Predicate)` | At capture time via `captureKeyPredicate` AND at restore time | Same dual-filter approach (`DefaultContextSnapshotFactory.java:40`, `DefaultContextSnapshot.java:setThreadLocals`) |

## 4.8 Connection to Real Framework

| Simplified Code | Real Framework Code | File:Line | Key Difference |
|----------------|--------------------|-----------| ---------------|
| `ContextSnapshotFactory` | `ContextSnapshotFactory` | `ContextSnapshotFactory.java:28` | Real adds `captureFrom(Object...)` and `setThreadLocalsFrom()` for `ContextAccessor`-based contexts |
| `captureAll()` (no args) | `captureAll(Object... contexts)` | `ContextSnapshotFactory.java:39` | Real accepts external context objects (Reactor `Context`) to also capture from |
| `DefaultContextSnapshotFactory.captureAll()` | `DefaultContextSnapshotFactory.captureAll()` | `DefaultContextSnapshotFactory.java:54` | Real calls both `captureFromThreadLocals()` and `captureFromContext()` |
| `captureAll()` loop | `captureFromThreadLocals()` | `DefaultContextSnapshotFactory.java:69` | Identical logic — iterate accessors, check predicate, read value, put in snapshot |
| `Builder` | `Builder` | `DefaultContextSnapshotFactory.java:168` | Identical structure and defaults |

## 4.9 Complete Code

### Production Code

#### File: `src/main/java/dev/linhvu/context/ContextSnapshotFactory.java` [NEW]

```java
package dev.linhvu.context;

import java.util.function.Predicate;

/**
 * Factory for creating {@link ContextSnapshot} instances by capturing current
 * {@link ThreadLocal} values from all registered {@link ThreadLocalAccessor}s.
 *
 * <p>This is the recommended entry point for context propagation. Rather than
 * manually constructing a snapshot and putting values into it, use the factory
 * to capture all registered ThreadLocal values in one call:
 *
 * <pre>{@code
 * ContextSnapshotFactory factory = ContextSnapshotFactory.builder().build();
 * ContextSnapshot snapshot = factory.captureAll();
 *
 * // Later, on another thread:
 * try (ContextSnapshot.Scope scope = snapshot.setThreadLocals()) {
 *     // ThreadLocals are restored here
 * }
 * }</pre>
 *
 * <p>The factory is configured via a {@link Builder} that controls:
 * <ul>
 *   <li>{@code contextRegistry} — which registry to read accessors from</li>
 *   <li>{@code captureKeyPredicate} — filter which keys to capture</li>
 *   <li>{@code clearMissing} — whether to clear ThreadLocals not in the snapshot
 *       when restoring</li>
 * </ul>
 *
 * @see ContextSnapshot
 * @see ContextRegistry
 * @see ThreadLocalAccessor
 */
public interface ContextSnapshotFactory {

    /**
     * Capture values from all registered {@link ThreadLocalAccessor}s in the
     * configured {@link ContextRegistry}.
     *
     * <p>Iterates each accessor, calls {@link ThreadLocalAccessor#getValue()},
     * and stores non-null values in the returned snapshot. The snapshot's
     * {@code clearMissing} behavior is determined by the factory configuration.
     *
     * @return a snapshot containing all captured ThreadLocal values
     */
    ContextSnapshot captureAll();

    /**
     * Create a {@link Builder} for configuring a {@link ContextSnapshotFactory}.
     *
     * <p>Defaults:
     * <ul>
     *   <li>{@code contextRegistry} — {@link ContextRegistry#getInstance()}</li>
     *   <li>{@code captureKeyPredicate} — captures all keys</li>
     *   <li>{@code clearMissing} — {@code false}</li>
     * </ul>
     *
     * @return a new builder instance
     */
    static Builder builder() {
        return new DefaultContextSnapshotFactory.Builder();
    }

    /**
     * Builder for {@link ContextSnapshotFactory} instances.
     */
    interface Builder {

        /**
         * Build the configured {@link ContextSnapshotFactory}.
         * @return a new factory instance
         */
        ContextSnapshotFactory build();

        /**
         * Whether to clear ThreadLocal values at scope open when they are not
         * present in the snapshot.
         *
         * <p>When {@code true}, ThreadLocals registered in the registry but
         * absent from the snapshot are actively cleared via
         * {@link ThreadLocalAccessor#setValue()} during
         * {@link ContextSnapshot#setThreadLocals()}, and restored on scope
         * close. This is important for Micrometer's
         * {@code ObservationThreadLocalAccessor} which uses the no-arg
         * {@code setValue()} to create a {@code NullObservation}.
         *
         * @param shouldClear {@code true} to clear missing ThreadLocals
         * @return this builder
         */
        Builder clearMissing(boolean shouldClear);

        /**
         * Set the {@link ContextRegistry} to use for reading accessors.
         * @param contextRegistry the registry
         * @return this builder
         */
        Builder contextRegistry(ContextRegistry contextRegistry);

        /**
         * Set a predicate to filter which keys are captured. Only accessors
         * whose {@link ThreadLocalAccessor#key()} passes this predicate will
         * have their values captured.
         *
         * @param captureKeyPredicate the predicate to filter keys
         * @return this builder
         */
        Builder captureKeyPredicate(Predicate<Object> captureKeyPredicate);

    }

}
```

#### File: `src/main/java/dev/linhvu/context/DefaultContextSnapshotFactory.java` [NEW]

```java
package dev.linhvu.context;

import java.util.function.Predicate;

/**
 * Default implementation of {@link ContextSnapshotFactory}.
 *
 * <p>Captures ThreadLocal values by iterating all {@link ThreadLocalAccessor}s
 * in the configured {@link ContextRegistry}, reading each value, and storing
 * non-null values in a {@link DefaultContextSnapshot}.
 *
 * <p>The capture is lazy — if no accessor has a non-null value, an empty
 * snapshot is returned. The empty snapshot still respects {@code clearMissing}:
 * when enabled, restoring an empty snapshot will clear all registered
 * ThreadLocals (and restore them on scope close).
 */
final class DefaultContextSnapshotFactory implements ContextSnapshotFactory {

    private static final DefaultContextSnapshot EMPTY_SNAPSHOT = new DefaultContextSnapshot(
            new ContextRegistry(), false);

    private final ContextRegistry contextRegistry;

    private final boolean clearMissing;

    private final Predicate<Object> captureKeyPredicate;

    DefaultContextSnapshotFactory(ContextRegistry contextRegistry, boolean clearMissing,
            Predicate<Object> captureKeyPredicate) {
        this.contextRegistry = contextRegistry;
        this.clearMissing = clearMissing;
        this.captureKeyPredicate = captureKeyPredicate;
    }

    @Override
    public ContextSnapshot captureAll() {
        DefaultContextSnapshot snapshot = null;

        for (ThreadLocalAccessor<?> accessor : this.contextRegistry.getThreadLocalAccessors()) {
            if (this.captureKeyPredicate.test(accessor.key())) {
                Object value = accessor.getValue();
                if (value != null) {
                    if (snapshot == null) {
                        snapshot = new DefaultContextSnapshot(this.contextRegistry, this.clearMissing);
                    }
                    snapshot.put(accessor.key(), value);
                }
            }
        }

        if (snapshot != null) {
            return snapshot;
        }
        return this.clearMissing ? new DefaultContextSnapshot(this.contextRegistry, true) : EMPTY_SNAPSHOT;
    }

    /**
     * Builder for {@link DefaultContextSnapshotFactory}.
     */
    static final class Builder implements ContextSnapshotFactory.Builder {

        private boolean clearMissing = false;

        private ContextRegistry contextRegistry = ContextRegistry.getInstance();

        private Predicate<Object> captureKeyPredicate = key -> true;

        Builder() {
        }

        @Override
        public ContextSnapshotFactory build() {
            return new DefaultContextSnapshotFactory(this.contextRegistry, this.clearMissing,
                    this.captureKeyPredicate);
        }

        @Override
        public ContextSnapshotFactory.Builder clearMissing(boolean shouldClear) {
            this.clearMissing = shouldClear;
            return this;
        }

        @Override
        public ContextSnapshotFactory.Builder contextRegistry(ContextRegistry contextRegistry) {
            this.contextRegistry = contextRegistry;
            return this;
        }

        @Override
        public ContextSnapshotFactory.Builder captureKeyPredicate(Predicate<Object> captureKeyPredicate) {
            this.captureKeyPredicate = captureKeyPredicate;
            return this;
        }

    }

}
```

### Test Code

#### File: `src/test/java/dev/linhvu/context/ContextSnapshotFactoryTest.java` [NEW]

```java
package dev.linhvu.context;

import org.junit.jupiter.api.AfterEach;
import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;

class ContextSnapshotFactoryTest {

    private final ThreadLocal<String> threadLocal1 = new ThreadLocal<>();

    private final ThreadLocal<String> threadLocal2 = new ThreadLocal<>();

    private final ContextRegistry registry = new ContextRegistry();

    @AfterEach
    void cleanup() {
        threadLocal1.remove();
        threadLocal2.remove();
    }

    // --- captureAll: basic capture ---

    @Test
    void shouldCaptureThreadLocalValue_WhenAccessorHasValue() {
        // Arrange
        registry.registerThreadLocalAccessor(new TestThreadLocalAccessor("key1", threadLocal1));
        threadLocal1.set("hello");

        ContextSnapshotFactory factory = ContextSnapshotFactory.builder()
            .contextRegistry(registry)
            .build();

        // Act
        ContextSnapshot snapshot = factory.captureAll();

        // Assert — restore on a clean thread simulates propagation
        threadLocal1.remove();
        try (ContextSnapshot.Scope scope = snapshot.setThreadLocals()) {
            assertThat(threadLocal1.get()).isEqualTo("hello");
        }
    }

    @Test
    void shouldCaptureMultipleValues_WhenMultipleAccessorsRegistered() {
        // Arrange
        registry.registerThreadLocalAccessor(new TestThreadLocalAccessor("key1", threadLocal1));
        registry.registerThreadLocalAccessor(new TestThreadLocalAccessor("key2", threadLocal2));
        threadLocal1.set("value1");
        threadLocal2.set("value2");

        ContextSnapshotFactory factory = ContextSnapshotFactory.builder()
            .contextRegistry(registry)
            .build();

        // Act
        ContextSnapshot snapshot = factory.captureAll();

        // Assert
        threadLocal1.remove();
        threadLocal2.remove();
        try (ContextSnapshot.Scope scope = snapshot.setThreadLocals()) {
            assertThat(threadLocal1.get()).isEqualTo("value1");
            assertThat(threadLocal2.get()).isEqualTo("value2");
        }
    }

    // --- captureAll: null values are skipped ---

    @Test
    void shouldSkipNullValues_WhenAccessorReturnsNull() {
        // Arrange
        registry.registerThreadLocalAccessor(new TestThreadLocalAccessor("key1", threadLocal1));
        registry.registerThreadLocalAccessor(new TestThreadLocalAccessor("key2", threadLocal2));
        threadLocal1.set("value1");
        // threadLocal2 is null — should be skipped

        ContextSnapshotFactory factory = ContextSnapshotFactory.builder()
            .contextRegistry(registry)
            .build();

        // Act
        ContextSnapshot snapshot = factory.captureAll();

        // Assert — only key1 is in the snapshot
        threadLocal1.remove();
        threadLocal2.set("should-stay");
        try (ContextSnapshot.Scope scope = snapshot.setThreadLocals()) {
            assertThat(threadLocal1.get()).isEqualTo("value1");
            // key2 was not captured, so it's left untouched (clearMissing=false)
            assertThat(threadLocal2.get()).isEqualTo("should-stay");
        }
    }

    // --- captureAll: empty snapshot ---

    @Test
    void shouldReturnEmptySnapshot_WhenNoAccessorsHaveValues() {
        // Arrange
        registry.registerThreadLocalAccessor(new TestThreadLocalAccessor("key1", threadLocal1));
        // threadLocal1 is null

        ContextSnapshotFactory factory = ContextSnapshotFactory.builder()
            .contextRegistry(registry)
            .build();

        // Act
        ContextSnapshot snapshot = factory.captureAll();

        // Assert — snapshot does nothing
        threadLocal1.set("untouched");
        try (ContextSnapshot.Scope scope = snapshot.setThreadLocals()) {
            assertThat(threadLocal1.get()).isEqualTo("untouched");
        }
        assertThat(threadLocal1.get()).isEqualTo("untouched");
    }

    // --- captureKeyPredicate ---

    @Test
    void shouldFilterByKeyPredicate_WhenPredicateConfigured() {
        // Arrange
        registry.registerThreadLocalAccessor(new TestThreadLocalAccessor("include", threadLocal1));
        registry.registerThreadLocalAccessor(new TestThreadLocalAccessor("exclude", threadLocal2));
        threadLocal1.set("yes");
        threadLocal2.set("no");

        ContextSnapshotFactory factory = ContextSnapshotFactory.builder()
            .contextRegistry(registry)
            .captureKeyPredicate(key -> "include".equals(key))
            .build();

        // Act
        ContextSnapshot snapshot = factory.captureAll();

        // Assert — only "include" was captured
        threadLocal1.remove();
        threadLocal2.remove();
        try (ContextSnapshot.Scope scope = snapshot.setThreadLocals()) {
            assertThat(threadLocal1.get()).isEqualTo("yes");
            assertThat(threadLocal2.get()).isNull(); // was not captured
        }
    }

    // --- clearMissing ---

    @Test
    void shouldClearMissingThreadLocals_WhenClearMissingEnabled() {
        // Arrange — key1 has a value, key2 does not
        registry.registerThreadLocalAccessor(new TestThreadLocalAccessor("key1", threadLocal1));
        registry.registerThreadLocalAccessor(new TestThreadLocalAccessor("key2", threadLocal2));
        threadLocal1.set("captured");
        // threadLocal2 is null — will NOT be in snapshot

        ContextSnapshotFactory factory = ContextSnapshotFactory.builder()
            .contextRegistry(registry)
            .clearMissing(true)
            .build();

        // Act
        ContextSnapshot snapshot = factory.captureAll();

        // Simulate target thread: key2 has a pre-existing value
        threadLocal1.remove();
        threadLocal2.set("pre-existing");

        try (ContextSnapshot.Scope scope = snapshot.setThreadLocals()) {
            assertThat(threadLocal1.get()).isEqualTo("captured");
            // key2 was NOT in snapshot → clearMissing clears it
            assertThat(threadLocal2.get()).isNull();
        }

        // After scope close, key2 is restored
        assertThat(threadLocal2.get()).isEqualTo("pre-existing");
    }

    @Test
    void shouldNotClearMissingThreadLocals_WhenClearMissingDisabled() {
        // Arrange
        registry.registerThreadLocalAccessor(new TestThreadLocalAccessor("key1", threadLocal1));
        registry.registerThreadLocalAccessor(new TestThreadLocalAccessor("key2", threadLocal2));
        threadLocal1.set("captured");

        ContextSnapshotFactory factory = ContextSnapshotFactory.builder()
            .contextRegistry(registry)
            .clearMissing(false)
            .build();

        // Act
        ContextSnapshot snapshot = factory.captureAll();

        threadLocal1.remove();
        threadLocal2.set("pre-existing");

        try (ContextSnapshot.Scope scope = snapshot.setThreadLocals()) {
            assertThat(threadLocal1.get()).isEqualTo("captured");
            // key2 is NOT cleared — left untouched
            assertThat(threadLocal2.get()).isEqualTo("pre-existing");
        }
    }

    // --- clearMissing with empty snapshot ---

    @Test
    void shouldClearAllThreadLocals_WhenEmptySnapshotWithClearMissing() {
        // Arrange — no values set, so snapshot will be empty
        registry.registerThreadLocalAccessor(new TestThreadLocalAccessor("key1", threadLocal1));

        ContextSnapshotFactory factory = ContextSnapshotFactory.builder()
            .contextRegistry(registry)
            .clearMissing(true)
            .build();

        // Act
        ContextSnapshot snapshot = factory.captureAll();

        // Simulate target thread with pre-existing value
        threadLocal1.set("pre-existing");

        try (ContextSnapshot.Scope scope = snapshot.setThreadLocals()) {
            // Empty snapshot + clearMissing → everything cleared
            assertThat(threadLocal1.get()).isNull();
        }

        // Restored after scope close
        assertThat(threadLocal1.get()).isEqualTo("pre-existing");
    }

    // --- Builder defaults ---

    @Test
    void shouldUseDefaults_WhenBuilderNotConfigured() {
        // Arrange — use global registry
        ContextRegistry globalRegistry = ContextRegistry.getInstance();
        globalRegistry.registerThreadLocalAccessor(new TestThreadLocalAccessor("global-key", threadLocal1));
        threadLocal1.set("global-value");

        try {
            ContextSnapshotFactory factory = ContextSnapshotFactory.builder().build();

            // Act
            ContextSnapshot snapshot = factory.captureAll();

            // Assert
            threadLocal1.remove();
            try (ContextSnapshot.Scope scope = snapshot.setThreadLocals()) {
                assertThat(threadLocal1.get()).isEqualTo("global-value");
            }
        }
        finally {
            // Cleanup global registry
            globalRegistry.removeThreadLocalAccessor("global-key");
            threadLocal1.remove();
        }
    }

}
```

#### File: `src/test/java/dev/linhvu/context/ContextSnapshotFactoryIntegrationTest.java` [NEW]

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
 * Integration tests verifying that ContextSnapshotFactory works end-to-end:
 * factory captures ThreadLocal values on the source thread, and the snapshot
 * propagates them to a worker thread.
 */
class ContextSnapshotFactoryIntegrationTest {

    private final ThreadLocal<String> userThreadLocal = new ThreadLocal<>();

    private final ThreadLocal<String> traceThreadLocal = new ThreadLocal<>();

    private final ContextRegistry registry = new ContextRegistry();

    @AfterEach
    void cleanup() {
        userThreadLocal.remove();
        traceThreadLocal.remove();
    }

    @Test
    void shouldPropagateAcrossThreads_WhenFactoryCaptures() throws Exception {
        // Arrange
        registry.registerThreadLocalAccessor(new TestThreadLocalAccessor("user", userThreadLocal));
        userThreadLocal.set("alice");

        ContextSnapshotFactory factory = ContextSnapshotFactory.builder()
            .contextRegistry(registry)
            .build();

        // Act — capture on source thread
        ContextSnapshot snapshot = factory.captureAll();

        AtomicReference<String> valueOnWorker = new AtomicReference<>();
        CountDownLatch latch = new CountDownLatch(1);

        ExecutorService executor = Executors.newSingleThreadExecutor();
        try {
            executor.submit(snapshot.wrap(() -> {
                valueOnWorker.set(userThreadLocal.get());
                latch.countDown();
            }));
            latch.await();
        }
        finally {
            executor.shutdown();
        }

        // Assert
        assertThat(valueOnWorker.get()).isEqualTo("alice");
        assertThat(userThreadLocal.get()).isEqualTo("alice");
    }

    @Test
    void shouldPropagateMultipleValues_WhenMultipleAccessorsRegistered() throws Exception {
        // Arrange
        registry.registerThreadLocalAccessor(new TestThreadLocalAccessor("user", userThreadLocal));
        registry.registerThreadLocalAccessor(new TestThreadLocalAccessor("trace", traceThreadLocal));
        userThreadLocal.set("bob");
        traceThreadLocal.set("trace-456");

        ContextSnapshotFactory factory = ContextSnapshotFactory.builder()
            .contextRegistry(registry)
            .build();

        ContextSnapshot snapshot = factory.captureAll();

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
        assertThat(traceOnWorker.get()).isEqualTo("trace-456");
    }

    @Test
    void shouldClearMissingOnWorkerThread_WhenClearMissingEnabled() throws Exception {
        // Arrange — capture with NO values set, but clearMissing=true
        registry.registerThreadLocalAccessor(new TestThreadLocalAccessor("user", userThreadLocal));

        ContextSnapshotFactory factory = ContextSnapshotFactory.builder()
            .contextRegistry(registry)
            .clearMissing(true)
            .build();

        ContextSnapshot snapshot = factory.captureAll();

        AtomicReference<String> valueDuringScope = new AtomicReference<>();
        AtomicReference<String> valueAfterScope = new AtomicReference<>();
        CountDownLatch latch = new CountDownLatch(1);

        ExecutorService executor = Executors.newSingleThreadExecutor();
        try {
            // Set a value on the worker thread first
            executor.submit(() -> userThreadLocal.set("worker-value")).get();

            // Submit task with the empty snapshot
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

        // Assert — during scope, ThreadLocal was cleared
        assertThat(valueDuringScope.get()).isNull();
        // After scope close, worker's original value was restored
        assertThat(valueAfterScope.get()).isEqualTo("worker-value");
    }

    @Test
    void shouldFilterCapture_WhenKeyPredicateConfigured() throws Exception {
        // Arrange
        registry.registerThreadLocalAccessor(new TestThreadLocalAccessor("user", userThreadLocal));
        registry.registerThreadLocalAccessor(new TestThreadLocalAccessor("trace", traceThreadLocal));
        userThreadLocal.set("alice");
        traceThreadLocal.set("trace-789");

        // Only capture "user", exclude "trace"
        ContextSnapshotFactory factory = ContextSnapshotFactory.builder()
            .contextRegistry(registry)
            .captureKeyPredicate(key -> "user".equals(key))
            .build();

        ContextSnapshot snapshot = factory.captureAll();

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

        // Assert — user was propagated, trace was not
        assertThat(userOnWorker.get()).isEqualTo("alice");
        assertThat(traceOnWorker.get()).isNull();
    }

}
```

---

## Summary

| Concept | What It Means |
|---------|---------------|
| **ContextSnapshotFactory** | The recommended entry point for capturing ThreadLocal values — configured once, used repeatedly |
| **captureAll()** | Iterates all registered accessors, reads their current values, packs non-null values into a snapshot |
| **captureKeyPredicate** | Filter applied at capture time to select which keys are included in the snapshot |
| **clearMissing** | When `true`, ThreadLocals absent from the snapshot are actively cleared during scope open and restored on close |
| **Lazy allocation** | Snapshot is only allocated when a non-null value is found — empty captures cost nothing |
| **Builder pattern** | Separates factory construction (one-time) from snapshot creation (repeated) |

**Next: Chapter 5 — Executor Service Wrappers** — Wrap `ExecutorService` and `ScheduledExecutorService` so that every submitted task automatically captures a fresh snapshot and propagates ThreadLocal values to the worker thread.
