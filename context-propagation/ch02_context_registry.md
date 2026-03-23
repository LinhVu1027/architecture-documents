# Chapter 2: ContextRegistry

> Line references based on commit `be901c1` of the Context Propagation repository.

## Build Challenge

| Current State | Limitation | Objective |
|---------------|------------|-----------|
| A `ThreadLocalAccessor<V>` interface exists but there's no way to collect and manage multiple accessors | Each library's accessor is standalone — the propagation system can't discover or iterate all registered ThreadLocals | Build a `ContextRegistry` that centrally manages all `ThreadLocalAccessor` instances with a global singleton, ServiceLoader discovery, and thread-safe storage |

---

## 2.1 The Integration Point

ContextRegistry is the **bridge between library registration and context capture**. It connects two subsystems:

- **Upstream:** Libraries like Micrometer and Tracing that register their ThreadLocals at startup
- **Downstream:** ContextSnapshot (ch03) and ContextSnapshotFactory (ch04) that iterate all accessors to capture/restore values

```
Library Registration                    Context Capture
─────────────────                       ───────────────
Micrometer ──┐                      ┌── ContextSnapshotFactory.captureAll()
Tracing ─────┼── ContextRegistry ───┤
Custom ──────┘   (this chapter)     └── ContextSnapshot.setThreadLocals()
```

The registry is the **company directory** from the outline's analogy — it knows every desk item (ThreadLocal) across every employee (library). When someone moves buildings (switches threads), the registry tells the moving crew (snapshot) exactly what to pack.

Two key decisions:

1. **Why a central registry instead of passing accessors directly?** Decoupling. The library that registers an accessor (Micrometer) doesn't know the code that captures context (a servlet filter or executor wrapper). The registry makes them independent.

2. **Why `CopyOnWriteArrayList` instead of synchronized list?** Registration happens at startup (few writes). Capture/restore happens on every request (many reads). `CopyOnWriteArrayList` optimizes for exactly this pattern — lock-free reads, expensive writes.

To make this work, we need to build:
- Storage for `ThreadLocalAccessor` instances with same-key replacement
- Convenience methods to register plain `ThreadLocal` objects without implementing the full interface
- `ServiceLoader` discovery to auto-detect accessors on the classpath
- A global singleton for application-wide access

## 2.2 The ContextRegistry Class

**New file:** `src/main/java/dev/linhvu/context/ContextRegistry.java`

```java
package dev.linhvu.context;

import java.util.Collections;
import java.util.List;
import java.util.ServiceLoader;
import java.util.concurrent.CopyOnWriteArrayList;
import java.util.function.Consumer;
import java.util.function.Supplier;

public class ContextRegistry {

    private static final ContextRegistry instance = new ContextRegistry().loadThreadLocalAccessors();

    private final List<ThreadLocalAccessor<?>> threadLocalAccessors = new CopyOnWriteArrayList<>();

    private final List<ThreadLocalAccessor<?>> readOnlyThreadLocalAccessors = Collections
        .unmodifiableList(this.threadLocalAccessors);
```

Three fields, three roles:

| Field | Type | Purpose |
|-------|------|---------|
| `instance` | `static final ContextRegistry` | Global singleton, eagerly initialized with ServiceLoader |
| `threadLocalAccessors` | `CopyOnWriteArrayList` | The actual mutable storage |
| `readOnlyThreadLocalAccessors` | `Collections.unmodifiableList` | An unmodifiable *view* of the same list — reflects mutations but prevents external modification |

### Core Registration

```java
    public ContextRegistry registerThreadLocalAccessor(ThreadLocalAccessor<?> accessor) {
        for (ThreadLocalAccessor<?> existing : this.threadLocalAccessors) {
            if (existing.key().equals(accessor.key())) {
                this.threadLocalAccessors.remove(existing);
                break;
            }
        }
        this.threadLocalAccessors.add(accessor);
        return this;
    }
```

**Same-key replacement:** If you register an accessor with key `"micrometer.observation"` and one already exists with that key, the old one is removed first. This allows tests to replace accessors and supports ServiceLoader re-loading without duplicates.

### Convenience: Wrap a ThreadLocal

```java
    public <V> ContextRegistry registerThreadLocalAccessor(String key, ThreadLocal<V> threadLocal) {
        return registerThreadLocalAccessor(key, threadLocal::get, threadLocal::set, threadLocal::remove);
    }
```

Delegates to the callback variant using method references on the `ThreadLocal`.

### Convenience: Wrap Callbacks

```java
    public <V> ContextRegistry registerThreadLocalAccessor(String key, Supplier<V> getSupplier,
            Consumer<V> setConsumer, Runnable resetTask) {

        return registerThreadLocalAccessor(new ThreadLocalAccessor<V>() {

            @Override
            public Object key() {
                return key;
            }

            @Override
            public V getValue() {
                return getSupplier.get();
            }

            @Override
            public void setValue(V value) {
                setConsumer.accept(value);
            }

            @Override
            public void setValue() {
                resetTask.run();
            }
        });
    }
```

Creates an anonymous `ThreadLocalAccessor` from callbacks. This is useful when you don't want to create a full class just to register a ThreadLocal — pass lambdas instead.

### Removal

```java
    public boolean removeThreadLocalAccessor(String key) {
        return removeThreadLocalAccessor((Object) key);
    }

    public boolean removeThreadLocalAccessor(Object key) {
        for (ThreadLocalAccessor<?> existing : this.threadLocalAccessors) {
            if (existing.key().equals(key)) {
                return this.threadLocalAccessors.remove(existing);
            }
        }
        return false;
    }
```

The `String` overload delegates to the `Object` overload (with an explicit cast to avoid recursive dispatch). Returns `true` if an accessor was actually removed.

### ServiceLoader Discovery

```java
    public ContextRegistry loadThreadLocalAccessors() {
        ServiceLoader.load(ThreadLocalAccessor.class).forEach(this::registerThreadLocalAccessor);
        return this;
    }
```

Discovers `ThreadLocalAccessor` implementations listed in `META-INF/services/dev.linhvu.context.ThreadLocalAccessor`. Because it delegates to `registerThreadLocalAccessor()`, same-key replacement applies — if an SPI accessor has the same key as a manually registered one, the SPI version wins.

### Getter and Singleton

```java
    public List<ThreadLocalAccessor<?>> getThreadLocalAccessors() {
        return this.readOnlyThreadLocalAccessors;
    }

    public static ContextRegistry getInstance() {
        return instance;
    }
}
```

`getThreadLocalAccessors()` returns the read-only view. It reflects live changes — when a new accessor is registered, it immediately appears in previously obtained lists.

## 2.3 Try It Yourself

<details>
<summary>Challenge: Register two ThreadLocals and iterate them to capture values</summary>

Imagine you have two ThreadLocals — a user ID and a request ID. Register both and then iterate the registry to build a simple "snapshot" map.

```java
ThreadLocal<String> userId = new ThreadLocal<>();
ThreadLocal<String> requestId = new ThreadLocal<>();
userId.set("user-42");
requestId.set("req-abc");

ContextRegistry registry = new ContextRegistry();
registry.registerThreadLocalAccessor("userId", userId);
registry.registerThreadLocalAccessor("requestId", requestId);

// Build a snapshot map by iterating the registry
Map<Object, Object> snapshot = new HashMap<>();
for (ThreadLocalAccessor<?> accessor : registry.getThreadLocalAccessors()) {
    Object value = accessor.getValue();
    if (value != null) {
        snapshot.put(accessor.key(), value);
    }
}

// snapshot = {userId=user-42, requestId=req-abc}
```

This is exactly what `ContextSnapshotFactory.captureAll()` will do in Chapter 4 — the registry provides the iteration, the factory builds the snapshot.

</details>

<details>
<summary>Challenge: Use ServiceLoader to auto-discover an accessor</summary>

1. Create a class that implements `ThreadLocalAccessor<String>`:

```java
package dev.linhvu.context;

public class AutoDiscoveredAccessor implements ThreadLocalAccessor<String> {
    private static final ThreadLocal<String> LOCALE = ThreadLocal.withInitial(() -> "en");

    @Override public Object key() { return "locale"; }
    @Override public String getValue() { return LOCALE.get(); }
    @Override public void setValue(String value) { LOCALE.set(value); }
    @Override public void setValue() { LOCALE.remove(); }
}
```

2. Create `src/main/resources/META-INF/services/dev.linhvu.context.ThreadLocalAccessor` containing:
```
dev.linhvu.context.AutoDiscoveredAccessor
```

3. Now `ContextRegistry.getInstance()` will automatically include this accessor — no manual registration needed.

</details>

## 2.4 Tests

### Unit Tests

**New file:** `src/test/java/dev/linhvu/context/ContextRegistryTest.java`

```java
package dev.linhvu.context;

import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;

class ContextRegistryTest {

    private final ContextRegistry registry = new ContextRegistry();

    // --- Registration ---

    @Test
    void shouldRegisterAccessor_WhenThreadLocalAccessorProvided() {
        ThreadLocal<String> threadLocal = new ThreadLocal<>();
        TestThreadLocalAccessor accessor = new TestThreadLocalAccessor("foo", threadLocal);

        registry.registerThreadLocalAccessor(accessor);

        assertThat(registry.getThreadLocalAccessors()).containsExactly(accessor);
    }

    @Test
    void shouldReplaceAccessor_WhenSameKeyRegisteredTwice() {
        TestThreadLocalAccessor accessor1 = new TestThreadLocalAccessor("foo", new ThreadLocal<>());
        TestThreadLocalAccessor accessor2 = new TestThreadLocalAccessor("foo", new ThreadLocal<>());

        registry.registerThreadLocalAccessor(accessor1);
        assertThat(registry.getThreadLocalAccessors()).containsExactly(accessor1);

        registry.registerThreadLocalAccessor(accessor2);
        assertThat(registry.getThreadLocalAccessors()).containsExactly(accessor2);
    }

    @Test
    void shouldKeepBothAccessors_WhenDifferentKeysRegistered() {
        TestThreadLocalAccessor accessor1 = new TestThreadLocalAccessor("foo", new ThreadLocal<>());
        TestThreadLocalAccessor accessor2 = new TestThreadLocalAccessor("bar", new ThreadLocal<>());

        registry.registerThreadLocalAccessor(accessor1);
        registry.registerThreadLocalAccessor(accessor2);

        assertThat(registry.getThreadLocalAccessors()).containsExactly(accessor1, accessor2);
    }

    @Test
    void shouldReplaceAndAddNewAccessor_WhenMixedKeysRegistered() {
        TestThreadLocalAccessor accessor1 = new TestThreadLocalAccessor("foo", new ThreadLocal<>());
        TestThreadLocalAccessor accessor2 = new TestThreadLocalAccessor("foo", new ThreadLocal<>());
        TestThreadLocalAccessor accessor3 = new TestThreadLocalAccessor("bar", new ThreadLocal<>());

        registry.registerThreadLocalAccessor(accessor1);
        registry.registerThreadLocalAccessor(accessor2);
        registry.registerThreadLocalAccessor(accessor3);

        assertThat(registry.getThreadLocalAccessors()).containsExactly(accessor2, accessor3);
    }

    // --- Convenience registration ---

    @Test
    void shouldRegisterFromThreadLocal_WhenConvenienceMethodUsed() {
        ThreadLocal<String> threadLocal = new ThreadLocal<>();
        threadLocal.set("hello");

        registry.registerThreadLocalAccessor("tl.key", threadLocal);

        ThreadLocalAccessor<?> accessor = registry.getThreadLocalAccessors().get(0);
        assertThat(accessor.key()).isEqualTo("tl.key");
        assertThat(accessor.getValue()).isEqualTo("hello");

        threadLocal.remove();
    }

    @Test
    void shouldRegisterFromCallbacks_WhenSupplierConsumerRunnableProvided() {
        ThreadLocal<Integer> threadLocal = new ThreadLocal<>();
        threadLocal.set(42);

        registry.registerThreadLocalAccessor("cb.key",
                threadLocal::get, threadLocal::set, threadLocal::remove);

        ThreadLocalAccessor<?> accessor = registry.getThreadLocalAccessors().get(0);
        assertThat(accessor.key()).isEqualTo("cb.key");
        assertThat(accessor.getValue()).isEqualTo(42);

        threadLocal.remove();
    }

    // --- Removal ---

    @Test
    void shouldRemoveAccessor_WhenStringKeyProvided() {
        TestThreadLocalAccessor accessor1 = new TestThreadLocalAccessor("foo", new ThreadLocal<>());
        TestThreadLocalAccessor accessor2 = new TestThreadLocalAccessor("bar", new ThreadLocal<>());
        registry.registerThreadLocalAccessor(accessor1);
        registry.registerThreadLocalAccessor(accessor2);

        assertThat(registry.removeThreadLocalAccessor("foo")).isTrue();
        assertThat(registry.getThreadLocalAccessors()).containsExactly(accessor2);
    }

    @Test
    void shouldRemoveAccessor_WhenObjectKeyProvided() {
        Object customKey = new Object();
        TestThreadLocalAccessor accessor = new TestThreadLocalAccessor(customKey, new ThreadLocal<>());
        registry.registerThreadLocalAccessor(accessor);

        assertThat(registry.removeThreadLocalAccessor(customKey)).isTrue();
        assertThat(registry.getThreadLocalAccessors()).isEmpty();
    }

    @Test
    void shouldReturnFalse_WhenRemovingNonexistentKey() {
        assertThat(registry.removeThreadLocalAccessor("nonexistent")).isFalse();
    }

    // --- Read-only view ---

    @Test
    void shouldReturnReadOnlyList_WhenGetThreadLocalAccessorsCalled() {
        TestThreadLocalAccessor accessor = new TestThreadLocalAccessor("foo", new ThreadLocal<>());
        registry.registerThreadLocalAccessor(accessor);

        var list = registry.getThreadLocalAccessors();

        TestThreadLocalAccessor accessor2 = new TestThreadLocalAccessor("bar", new ThreadLocal<>());
        registry.registerThreadLocalAccessor(accessor2);
        assertThat(list).containsExactly(accessor, accessor2);
    }

    // --- Singleton ---

    @Test
    void shouldReturnSameInstance_WhenGetInstanceCalledMultipleTimes() {
        ContextRegistry instance1 = ContextRegistry.getInstance();
        ContextRegistry instance2 = ContextRegistry.getInstance();

        assertThat(instance1).isSameAs(instance2);
    }

    // --- ServiceLoader ---

    @Test
    void shouldLoadAccessors_WhenServiceLoaderDiscoveryTriggered() {
        ContextRegistry freshRegistry = new ContextRegistry();
        freshRegistry.loadThreadLocalAccessors();

        TestThreadLocalAccessor accessor = new TestThreadLocalAccessor("manual", new ThreadLocal<>());
        freshRegistry.registerThreadLocalAccessor(accessor);
        assertThat(freshRegistry.getThreadLocalAccessors()).containsExactly(accessor);
    }

    // --- toString ---

    @Test
    void shouldIncludeAccessorInfo_WhenToStringCalled() {
        TestThreadLocalAccessor accessor = new TestThreadLocalAccessor("foo", new ThreadLocal<>());
        registry.registerThreadLocalAccessor(accessor);

        String result = registry.toString();
        assertThat(result).startsWith("ContextRegistry{");
        assertThat(result).contains("threadLocalAccessors=");
    }
}
```

**Run:** `./gradlew test` — expected: all 22 tests pass (13 new + 9 from ch01).

---

## 2.5 Why This Works

The registry's power is in what it enables, not what it does. By itself, it's just a list with some convenience methods. But it creates a **seam** between producers and consumers of thread-local state.

> ★ **Insight** -------------------------------------------
> - **Why `CopyOnWriteArrayList` instead of `synchronized` or `ConcurrentHashMap`?** The access pattern is "write rarely, read constantly." `CopyOnWriteArrayList` is ideal: reads (iteration during context capture) are lock-free and don't even need a volatile read of internal state. Writes (registration at startup) copy the entire array, which is fine for a list that typically contains 2-5 accessors. A `ConcurrentHashMap<Object, ThreadLocalAccessor<?>>` would also work but adds unnecessary key-lookup overhead when the primary operation is "iterate all."
> - **When this pattern breaks:** If accessors were registered/unregistered frequently during request processing, `CopyOnWriteArrayList` would be a performance disaster — every mutation copies the array. But the real Context Propagation library documents that the registry "is intended to be initialized on startup." The data structure matches the contract.
> - **Real-world parallel:** Spring's `HandlerMapping` registry follows the same pattern — initialized once during `ApplicationContext.refresh()`, read on every request.
> -----------------------------------------------------------

> ★ **Insight** -------------------------------------------
> - **Why an unmodifiable view instead of defensive copy?** `Collections.unmodifiableList(this.threadLocalAccessors)` wraps the live `CopyOnWriteArrayList`. Changes to the underlying list are visible through the view. A defensive copy (`new ArrayList<>(this.threadLocalAccessors)`) would be a frozen snapshot — callers wouldn't see later registrations. The live view is correct because `ContextSnapshotFactory` calls `getThreadLocalAccessors()` at capture time and expects to see all accessors registered up to that point.
> - **The `readOnlyThreadLocalAccessors` field is created once in the constructor** and returned on every `getThreadLocalAccessors()` call. No new object allocation per call.
> -----------------------------------------------------------

> ★ **Insight** -------------------------------------------
> - **Why ServiceLoader?** It enables **zero-code registration**. A library like Micrometer ships a `META-INF/services/io.micrometer.context.ThreadLocalAccessor` file listing `ObservationThreadLocalAccessor`. When the class is on the classpath, the registry auto-discovers it at startup. Application code never calls `registerThreadLocalAccessor(new ObservationThreadLocalAccessor())` — it just works. This is the same mechanism Spring Boot uses for auto-configuration discovery (via `spring.factories` and now `AutoConfiguration.imports`).
> -----------------------------------------------------------

## 2.6 What We Enhanced

| Aspect | Before (ch01) | Current (ch02) | Real Framework |
|--------|--------------|----------------|----------------|
| Accessor management | Each `ThreadLocalAccessor` is standalone — no central tracking | `ContextRegistry` collects all accessors with same-key replacement | Same pattern with added `ContextAccessor` support (`ContextRegistry.java:45`) |
| Registration API | Must implement `ThreadLocalAccessor<V>` directly | Convenience methods: pass a `ThreadLocal`, or `Supplier`/`Consumer`/`Runnable` callbacks | Same convenience methods (`ContextRegistry.java:91,105`) |
| Discovery | Manual instantiation only | `ServiceLoader` auto-discovery via `META-INF/services/` | Same ServiceLoader pattern (`ContextRegistry.java:201`) |
| Thread safety | N/A (single accessor, no shared state) | `CopyOnWriteArrayList` — lock-free reads, copy-on-write mutations | Same (`ContextRegistry.java:47`) |

## 2.7 Connection to Real Framework

| Simplified Code | Real Framework Code | File:Line | Key Difference |
|----------------|--------------------|-----------| ---------------|
| `ContextRegistry` | `ContextRegistry` | `ContextRegistry.java:40` | Real also manages `ContextAccessor` instances (reactive context) |
| `threadLocalAccessors` field | same | `ContextRegistry.java:47` | Identical: `CopyOnWriteArrayList<ThreadLocalAccessor<?>>` |
| `registerThreadLocalAccessor(ThreadLocalAccessor)` | same | `ContextRegistry.java:138` | Identical same-key replacement logic |
| `registerThreadLocalAccessor(String, ThreadLocal)` | same | `ContextRegistry.java:91` | Identical delegation to callback variant |
| `registerThreadLocalAccessor(String, Supplier, Consumer, Runnable)` | same | `ContextRegistry.java:105` | Real uses `@Nullable` on the type parameter |
| `removeThreadLocalAccessor(Object)` | same | `ContextRegistry.java:164` | Identical iteration + remove pattern |
| `loadThreadLocalAccessors()` | same | `ContextRegistry.java:201` | Identical ServiceLoader pattern |
| `getInstance()` | same | `ContextRegistry.java:263` | Real chains `loadContextAccessors().loadThreadLocalAccessors()` |
| *(not implemented)* | `registerContextAccessor()` | `ContextRegistry.java:60` | Validates readable/writable type conflicts (reactive-only) |
| *(not implemented)* | `getContextAccessorForRead/Write()` | `ContextRegistry.java:211,225` | Looks up ContextAccessor by context type (reactive-only) |

**Omitted from simplified version:**
- All `ContextAccessor` management (reactive context bridging — out of scope per outline)
- Type-conflict validation for `ContextAccessor` registration (no `ContextAccessor` = no conflict checking needed)
- `@Nullable` / JSpecify nullability annotations

## 2.8 Complete Code

### Production Code

#### File: `src/main/java/dev/linhvu/context/ContextRegistry.java` [NEW]

```java
package dev.linhvu.context;

import java.util.Collections;
import java.util.List;
import java.util.ServiceLoader;
import java.util.concurrent.CopyOnWriteArrayList;
import java.util.function.Consumer;
import java.util.function.Supplier;

/**
 * Registry that provides access to instances of {@link ThreadLocalAccessor}.
 *
 * <p>A static instance is available via {@link #getInstance()}. It is intended to be
 * initialized on startup, and to be aware of all available accessors, as many as
 * possible. The means to control what context gets propagated is in the snapshot,
 * which filters context values by key.
 *
 * <p>Thread safety is achieved via {@link CopyOnWriteArrayList} — reads are lock-free
 * and fast (ideal for the read-heavy access pattern during context capture/restore),
 * while writes (registration) create a new copy of the underlying array.
 */
public class ContextRegistry {

    private static final ContextRegistry instance = new ContextRegistry().loadThreadLocalAccessors();

    private final List<ThreadLocalAccessor<?>> threadLocalAccessors = new CopyOnWriteArrayList<>();

    private final List<ThreadLocalAccessor<?>> readOnlyThreadLocalAccessors = Collections
        .unmodifiableList(this.threadLocalAccessors);

    /**
     * Register a {@link ThreadLocalAccessor} for the given {@link ThreadLocal}.
     * @param key the {@link ThreadLocalAccessor#key() key} to associate with the
     * ThreadLocal value
     * @param threadLocal the underlying {@code ThreadLocal}
     * @return the same registry instance
     * @param <V> the type of value stored in the ThreadLocal
     */
    public <V> ContextRegistry registerThreadLocalAccessor(String key, ThreadLocal<V> threadLocal) {
        return registerThreadLocalAccessor(key, threadLocal::get, threadLocal::set, threadLocal::remove);
    }

    /**
     * Register a {@link ThreadLocalAccessor} from callbacks.
     * @param key the {@link ThreadLocalAccessor#key() key} to associate with the
     * ThreadLocal value
     * @param getSupplier callback to use for getting the value
     * @param setConsumer callback to use for setting the value
     * @param resetTask callback to use for resetting the value
     * @return the same registry instance
     * @param <V> the type of value stored in the ThreadLocal
     */
    public <V> ContextRegistry registerThreadLocalAccessor(String key, Supplier<V> getSupplier,
            Consumer<V> setConsumer, Runnable resetTask) {

        return registerThreadLocalAccessor(new ThreadLocalAccessor<V>() {

            @Override
            public Object key() {
                return key;
            }

            @Override
            public V getValue() {
                return getSupplier.get();
            }

            @Override
            public void setValue(V value) {
                setConsumer.accept(value);
            }

            @Override
            public void setValue() {
                resetTask.run();
            }
        });
    }

    /**
     * Register a {@link ThreadLocalAccessor}. If there is an existing registration with
     * the same {@link ThreadLocalAccessor#key() key}, it is removed first.
     * @param accessor the accessor to register
     * @return the same registry instance
     */
    public ContextRegistry registerThreadLocalAccessor(ThreadLocalAccessor<?> accessor) {
        for (ThreadLocalAccessor<?> existing : this.threadLocalAccessors) {
            if (existing.key().equals(accessor.key())) {
                this.threadLocalAccessors.remove(existing);
                break;
            }
        }
        this.threadLocalAccessors.add(accessor);
        return this;
    }

    /**
     * Removes a {@link ThreadLocalAccessor}.
     * @param key under which the accessor got registered
     * @return {@code true} when accessor got successfully removed
     */
    public boolean removeThreadLocalAccessor(String key) {
        return removeThreadLocalAccessor((Object) key);
    }

    /**
     * Removes a {@link ThreadLocalAccessor}.
     * @param key under which the accessor got registered
     * @return {@code true} when accessor got successfully removed
     */
    public boolean removeThreadLocalAccessor(Object key) {
        for (ThreadLocalAccessor<?> existing : this.threadLocalAccessors) {
            if (existing.key().equals(key)) {
                return this.threadLocalAccessors.remove(existing);
            }
        }
        return false;
    }

    /**
     * Load {@link ThreadLocalAccessor} implementations through the {@link ServiceLoader}
     * mechanism.
     * <p>
     * Note that existing registrations with the same {@link ThreadLocalAccessor#key()
     * key}, if any, are removed first (handled by {@link #registerThreadLocalAccessor}).
     */
    public ContextRegistry loadThreadLocalAccessors() {
        ServiceLoader.load(ThreadLocalAccessor.class).forEach(this::registerThreadLocalAccessor);
        return this;
    }

    /**
     * Return a read-only list of registered {@link ThreadLocalAccessor}'s.
     */
    public List<ThreadLocalAccessor<?>> getThreadLocalAccessors() {
        return this.readOnlyThreadLocalAccessors;
    }

    @Override
    public String toString() {
        return "ContextRegistry{threadLocalAccessors=" + this.threadLocalAccessors + "}";
    }

    /**
     * Return a global {@link ContextRegistry} instance.
     * <p>
     * <strong>Note:</strong> The global instance should be initialized on startup to
     * ensure it has the ability to propagate throughout the application. The registry
     * itself is not intended as a mechanism to control what gets propagated. It is in
     * the snapshot where more fine-grained decisions can be made about which context
     * values to propagate.
     */
    public static ContextRegistry getInstance() {
        return instance;
    }

}
```

### Test Code

#### File: `src/test/java/dev/linhvu/context/ContextRegistryTest.java` [NEW]

```java
package dev.linhvu.context;

import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;

class ContextRegistryTest {

    private final ContextRegistry registry = new ContextRegistry();

    // --- Registration ---

    @Test
    void shouldRegisterAccessor_WhenThreadLocalAccessorProvided() {
        ThreadLocal<String> threadLocal = new ThreadLocal<>();
        TestThreadLocalAccessor accessor = new TestThreadLocalAccessor("foo", threadLocal);

        registry.registerThreadLocalAccessor(accessor);

        assertThat(registry.getThreadLocalAccessors()).containsExactly(accessor);
    }

    @Test
    void shouldReplaceAccessor_WhenSameKeyRegisteredTwice() {
        TestThreadLocalAccessor accessor1 = new TestThreadLocalAccessor("foo", new ThreadLocal<>());
        TestThreadLocalAccessor accessor2 = new TestThreadLocalAccessor("foo", new ThreadLocal<>());

        registry.registerThreadLocalAccessor(accessor1);
        assertThat(registry.getThreadLocalAccessors()).containsExactly(accessor1);

        registry.registerThreadLocalAccessor(accessor2);
        assertThat(registry.getThreadLocalAccessors()).containsExactly(accessor2);
    }

    @Test
    void shouldKeepBothAccessors_WhenDifferentKeysRegistered() {
        TestThreadLocalAccessor accessor1 = new TestThreadLocalAccessor("foo", new ThreadLocal<>());
        TestThreadLocalAccessor accessor2 = new TestThreadLocalAccessor("bar", new ThreadLocal<>());

        registry.registerThreadLocalAccessor(accessor1);
        registry.registerThreadLocalAccessor(accessor2);

        assertThat(registry.getThreadLocalAccessors()).containsExactly(accessor1, accessor2);
    }

    @Test
    void shouldReplaceAndAddNewAccessor_WhenMixedKeysRegistered() {
        TestThreadLocalAccessor accessor1 = new TestThreadLocalAccessor("foo", new ThreadLocal<>());
        TestThreadLocalAccessor accessor2 = new TestThreadLocalAccessor("foo", new ThreadLocal<>());
        TestThreadLocalAccessor accessor3 = new TestThreadLocalAccessor("bar", new ThreadLocal<>());

        registry.registerThreadLocalAccessor(accessor1);
        registry.registerThreadLocalAccessor(accessor2);
        registry.registerThreadLocalAccessor(accessor3);

        assertThat(registry.getThreadLocalAccessors()).containsExactly(accessor2, accessor3);
    }

    // --- Convenience registration: ThreadLocal ---

    @Test
    void shouldRegisterFromThreadLocal_WhenConvenienceMethodUsed() {
        ThreadLocal<String> threadLocal = new ThreadLocal<>();
        threadLocal.set("hello");

        registry.registerThreadLocalAccessor("tl.key", threadLocal);

        assertThat(registry.getThreadLocalAccessors()).hasSize(1);
        ThreadLocalAccessor<?> accessor = registry.getThreadLocalAccessors().get(0);
        assertThat(accessor.key()).isEqualTo("tl.key");
        assertThat(accessor.getValue()).isEqualTo("hello");

        threadLocal.remove();
    }

    // --- Convenience registration: Supplier/Consumer/Runnable ---

    @Test
    void shouldRegisterFromCallbacks_WhenSupplierConsumerRunnableProvided() {
        ThreadLocal<Integer> threadLocal = new ThreadLocal<>();
        threadLocal.set(42);

        registry.registerThreadLocalAccessor("cb.key",
                threadLocal::get, threadLocal::set, threadLocal::remove);

        ThreadLocalAccessor<?> accessor = registry.getThreadLocalAccessors().get(0);
        assertThat(accessor.key()).isEqualTo("cb.key");
        assertThat(accessor.getValue()).isEqualTo(42);

        // Test setValue via the accessor
        @SuppressWarnings("unchecked")
        ThreadLocalAccessor<Integer> typed = (ThreadLocalAccessor<Integer>) accessor;
        typed.setValue(99);
        assertThat(threadLocal.get()).isEqualTo(99);

        // Test clear via the accessor
        typed.setValue();
        assertThat(threadLocal.get()).isNull();

        threadLocal.remove();
    }

    // --- Removal ---

    @Test
    void shouldRemoveAccessor_WhenStringKeyProvided() {
        TestThreadLocalAccessor accessor1 = new TestThreadLocalAccessor("foo", new ThreadLocal<>());
        TestThreadLocalAccessor accessor2 = new TestThreadLocalAccessor("bar", new ThreadLocal<>());
        registry.registerThreadLocalAccessor(accessor1);
        registry.registerThreadLocalAccessor(accessor2);

        assertThat(registry.removeThreadLocalAccessor("foo")).isTrue();
        assertThat(registry.getThreadLocalAccessors()).containsExactly(accessor2);
    }

    @Test
    void shouldRemoveAccessor_WhenObjectKeyProvided() {
        Object customKey = new Object();
        TestThreadLocalAccessor accessor = new TestThreadLocalAccessor(customKey, new ThreadLocal<>());
        registry.registerThreadLocalAccessor(accessor);

        assertThat(registry.removeThreadLocalAccessor(customKey)).isTrue();
        assertThat(registry.getThreadLocalAccessors()).isEmpty();
    }

    @Test
    void shouldReturnFalse_WhenRemovingNonexistentKey() {
        assertThat(registry.removeThreadLocalAccessor("nonexistent")).isFalse();
    }

    // --- Read-only view ---

    @Test
    void shouldReturnReadOnlyList_WhenGetThreadLocalAccessorsCalled() {
        TestThreadLocalAccessor accessor = new TestThreadLocalAccessor("foo", new ThreadLocal<>());
        registry.registerThreadLocalAccessor(accessor);

        var list = registry.getThreadLocalAccessors();

        // Verify it reflects mutations to the underlying list
        TestThreadLocalAccessor accessor2 = new TestThreadLocalAccessor("bar", new ThreadLocal<>());
        registry.registerThreadLocalAccessor(accessor2);
        assertThat(list).containsExactly(accessor, accessor2);
    }

    // --- Singleton ---

    @Test
    void shouldReturnSameInstance_WhenGetInstanceCalledMultipleTimes() {
        ContextRegistry instance1 = ContextRegistry.getInstance();
        ContextRegistry instance2 = ContextRegistry.getInstance();

        assertThat(instance1).isSameAs(instance2);
    }

    // --- ServiceLoader ---

    @Test
    void shouldLoadAccessors_WhenServiceLoaderDiscoveryTriggered() {
        // Fresh registry — no SPI in our test classpath, so loadThreadLocalAccessors
        // should not throw and the registry should remain functional
        ContextRegistry freshRegistry = new ContextRegistry();
        freshRegistry.loadThreadLocalAccessors();

        // Should still be able to register manually after loading
        TestThreadLocalAccessor accessor = new TestThreadLocalAccessor("manual", new ThreadLocal<>());
        freshRegistry.registerThreadLocalAccessor(accessor);
        assertThat(freshRegistry.getThreadLocalAccessors()).containsExactly(accessor);
    }

    // --- toString ---

    @Test
    void shouldIncludeAccessorInfo_WhenToStringCalled() {
        TestThreadLocalAccessor accessor = new TestThreadLocalAccessor("foo", new ThreadLocal<>());
        registry.registerThreadLocalAccessor(accessor);

        String result = registry.toString();
        assertThat(result).startsWith("ContextRegistry{");
        assertThat(result).contains("threadLocalAccessors=");
    }

}
```

---

## Summary

| Concept | What It Means |
|---------|---------------|
| **ContextRegistry** | Central collection of all `ThreadLocalAccessor` instances — the "company directory" that tells the propagation system what ThreadLocals exist |
| **CopyOnWriteArrayList** | Thread-safe list optimized for read-heavy workloads — lock-free iteration, copy-on-write mutations |
| **Same-key replacement** | Registering an accessor with an existing key removes the old one first — prevents duplicates and supports test overrides |
| **ServiceLoader discovery** | `META-INF/services/` SPI mechanism — libraries ship their accessors, the registry discovers them at startup with zero application code |
| **Unmodifiable view** | `Collections.unmodifiableList()` wrapping the live list — prevents external mutation while reflecting internal changes |
| **Fluent API** | Registration methods return `this` — enables chaining like `registry.registerThreadLocalAccessor(a).registerThreadLocalAccessor(b)` |

**Next: Chapter 3 — ContextSnapshot and Scope** — The "moving box" that captures ThreadLocal values from the registry into a key-value map, and restores them on a target thread via a try-with-resources `Scope`. This is where the actual propagation happens.
