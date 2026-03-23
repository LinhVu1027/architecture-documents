# Chapter 1: ThreadLocalAccessor

> Line references based on commit `be901c1` of the Context Propagation repository.

## Build Challenge

| Current State | Limitation | Objective |
|---------------|------------|-----------|
| No simplified framework exists yet | No abstraction for reading, setting, and clearing ThreadLocal values across thread boundaries | Build a `ThreadLocalAccessor<V>` interface that defines the contract for bridging a single ThreadLocal into the propagation system |

---

## 1.1 The Foundation: ThreadLocalAccessor Interface

This is a foundation chapter — there's no existing system to plug into. The `ThreadLocalAccessor` interface IS the core data structure that all future features depend on:

- **ContextRegistry** (ch02) will hold a list of `ThreadLocalAccessor` instances
- **ContextSnapshot** (ch03) will call `getValue()` to capture and `setValue()`/`restore()` to apply
- **ContextSnapshotFactory** (ch04) will iterate accessors to build snapshots
- **Executor wrappers** (ch05) will use snapshots that depend on this interface

**New file:** `src/main/java/dev/linhvu/context/ThreadLocalAccessor.java`

```java
package dev.linhvu.context;

public interface ThreadLocalAccessor<V> {

    Object key();

    V getValue();

    void setValue(V value);

    default void setValue() {
    }

    default void restore(V previousValue) {
        setValue(previousValue);
    }

    default void restore() {
        setValue();
    }

}
```

Six methods, four behaviors:

| Method | When Called | Purpose |
|--------|-----------|---------|
| `key()` | Registration & lookup | Unique identifier for this accessor in the registry |
| `getValue()` | Snapshot capture | Read the current ThreadLocal value (null = absent) |
| `setValue(V)` | Scope open | Set a captured value onto the target thread |
| `setValue()` | Scope open (clearMissing) | Clear the ThreadLocal when no value exists in the snapshot |
| `restore(V)` | Scope close | Restore the previous value when the scope ends |
| `restore()` | Scope close | Clear the ThreadLocal when no previous value existed |

Two key decisions here:

1. **`setValue` vs `restore` are separate operations.** For simple ThreadLocals they do the same thing (the defaults delegate). But scope-aware ThreadLocals like Micrometer's `ObservationThreadLocalAccessor` need different behavior on scope close — closing an Observation scope is NOT the same as setting a new Observation.

2. **`key()` returns `Object`, not `String`.** This allows enum constants or class literals as keys, though in practice most implementations use String constants.

## 1.2 A Concrete Implementation: TestThreadLocalAccessor

To verify the contract, we need a concrete implementation. This is the simplest possible accessor — it wraps a `ThreadLocal<String>` with a configurable key.

**New file:** `src/test/java/dev/linhvu/context/TestThreadLocalAccessor.java`

```java
package dev.linhvu.context;

public class TestThreadLocalAccessor implements ThreadLocalAccessor<String> {

    private final Object key;
    private final ThreadLocal<String> threadLocal;

    public TestThreadLocalAccessor(Object key, ThreadLocal<String> threadLocal) {
        this.key = key;
        this.threadLocal = threadLocal;
    }

    @Override
    public Object key() {
        return this.key;
    }

    @Override
    public String getValue() {
        return this.threadLocal.get();
    }

    @Override
    public void setValue(String value) {
        this.threadLocal.set(value);
    }

    @Override
    public void setValue() {
        this.threadLocal.remove();
    }

}
```

Notice that `restore(V)` and `restore()` are NOT overridden — they use the defaults, which delegate to `setValue(V)` and `setValue()`. This is correct for a simple ThreadLocal where restoring is the same as setting.

## 1.3 Try It Yourself

<details>
<summary>Challenge: Implement a scope-aware accessor where restore() does something different from setValue()</summary>

Imagine a ThreadLocal that maintains a stack of values (like nested Observation scopes). Setting pushes onto the stack, but restoring pops from it. How would you implement `restore(V)` differently from `setValue(V)`?

```java
public class StackThreadLocalAccessor implements ThreadLocalAccessor<String> {

    private final ThreadLocal<Deque<String>> stack = ThreadLocal.withInitial(ArrayDeque::new);

    @Override
    public Object key() {
        return "stack";
    }

    @Override
    public String getValue() {
        return stack.get().peek();
    }

    @Override
    public void setValue(String value) {
        stack.get().push(value);  // Push onto stack
    }

    @Override
    public void setValue() {
        stack.get().clear();
    }

    @Override
    public void restore(String previousValue) {
        // Pop instead of push — unwinding the scope
        stack.get().poll();
    }

    @Override
    public void restore() {
        stack.get().poll();
    }

}
```

This demonstrates WHY `restore` exists as a separate operation — it enables structured unwinding rather than blind reassignment.

</details>

<details>
<summary>Challenge: Implement a ThreadLocalAccessor for a ThreadLocal&lt;Integer&gt; counter</summary>

Build an accessor for a request counter ThreadLocal. Think about what `setValue()` (no-arg) should do — should it set the counter to 0, or remove it entirely?

```java
public class CounterThreadLocalAccessor implements ThreadLocalAccessor<Integer> {

    private static final ThreadLocal<Integer> counter = new ThreadLocal<>();

    @Override
    public Object key() {
        return "request.counter";
    }

    @Override
    public Integer getValue() {
        return counter.get();
    }

    @Override
    public void setValue(Integer value) {
        counter.set(value);
    }

    @Override
    public void setValue() {
        counter.remove();  // null = no counter, not zero
    }

}
```

</details>

## 1.4 Tests

**New file:** `src/test/java/dev/linhvu/context/ThreadLocalAccessorTest.java`

```java
package dev.linhvu.context;

import org.junit.jupiter.api.AfterEach;
import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;

class ThreadLocalAccessorTest {

    private final ThreadLocal<String> threadLocal = new ThreadLocal<>();

    private final TestThreadLocalAccessor accessor = new TestThreadLocalAccessor("test.key", threadLocal);

    @AfterEach
    void cleanup() {
        threadLocal.remove();
    }

    @Test
    void shouldReturnKey_WhenKeyRequested() {
        assertThat(accessor.key()).isEqualTo("test.key");
    }

    @Test
    void shouldReturnNull_WhenNoValueSet() {
        assertThat(accessor.getValue()).isNull();
    }

    @Test
    void shouldReturnValue_WhenValueSet() {
        accessor.setValue("hello");
        assertThat(accessor.getValue()).isEqualTo("hello");
    }

    @Test
    void shouldClearValue_WhenSetValueCalledWithNoArgs() {
        accessor.setValue("hello");
        accessor.setValue();
        assertThat(accessor.getValue()).isNull();
    }

    @Test
    void shouldRestorePreviousValue_WhenRestoreCalledWithValue() {
        accessor.setValue("current");
        accessor.restore("previous");
        assertThat(accessor.getValue()).isEqualTo("previous");
    }

    @Test
    void shouldClearValue_WhenRestoreCalledWithNoArgs() {
        accessor.setValue("current");
        accessor.restore();
        assertThat(accessor.getValue()).isNull();
    }

    @Test
    void shouldSupportScopeLifecycle_WhenUsedInTryWithResourcesPattern() {
        // Source thread has "original"
        accessor.setValue("original");
        String capturedValue = accessor.getValue();

        // Target thread has "other-thread-value"
        accessor.setValue("other-thread-value");
        String previousValue = accessor.getValue();

        // Scope open: apply captured value
        accessor.setValue(capturedValue);
        assertThat(accessor.getValue()).isEqualTo("original");

        // Scope close: restore previous
        accessor.restore(previousValue);
        assertThat(accessor.getValue()).isEqualTo("other-thread-value");
    }

    @Test
    void shouldSupportScopeLifecycle_WhenNoPreviousValueExisted() {
        accessor.setValue("original");
        String capturedValue = accessor.getValue();

        // Target thread has no value
        accessor.setValue();
        assertThat(accessor.getValue()).isNull();

        // Scope open
        accessor.setValue(capturedValue);
        assertThat(accessor.getValue()).isEqualTo("original");

        // Scope close — no previous value
        accessor.restore();
        assertThat(accessor.getValue()).isNull();
    }

    @Test
    void shouldDefaultRestoreToSetValue_WhenRestoreNotOverridden() {
        ThreadLocalAccessor<String> defaultAccessor = new ThreadLocalAccessor<>() {
            private String value;

            @Override
            public Object key() { return "default"; }

            @Override
            public String getValue() { return value; }

            @Override
            public void setValue(String value) { this.value = value; }

            @Override
            public void setValue() { this.value = null; }
        };

        defaultAccessor.setValue("hello");
        defaultAccessor.restore("world");
        assertThat(defaultAccessor.getValue()).isEqualTo("world");

        defaultAccessor.restore();
        assertThat(defaultAccessor.getValue()).isNull();
    }

}
```

**Run:** `./gradlew test` — expected: all 9 tests pass.

---

## 1.5 Why This Works

The four-method design (`setValue(V)` / `setValue()` / `restore(V)` / `restore()`) looks redundant at first glance — why not just `set` and `clear`? The answer lies in **scope semantics**.

> ★ **Insight** -------------------------------------------
> - **Why separate `restore` from `setValue`?** Consider Micrometer's `ObservationThreadLocalAccessor`. When an Observation scope opens, `setValue(observation)` makes it the current observation. When the scope closes, `restore(previousObservation)` doesn't just set the ThreadLocal — it also closes the observation's scope object. If restore delegated blindly to setValue, nested observations would leak scope objects.
> - **The default implementation is intentional.** For 90% of ThreadLocals (MDC values, security context, request attributes), restore IS the same as setValue. The defaults make the simple case trivial while the override path enables the complex case.
> - **Real-world example:** In the Context Propagation repo, `ScopedValueThreadLocalAccessor` overrides `restore()` to call `currentScope.close()` — demonstrating that scope teardown and value assignment are fundamentally different operations.
> -----------------------------------------------------------

> ★ **Insight** -------------------------------------------
> - **Why `Object key()` instead of `String key()`?** The Context Propagation library uses `Object` to allow any type as a key — enum constants, class literals, or string constants. In practice, most keys are strings (`"micrometer.observation"`, `"tracing.span"`), but the flexibility exists for type-safe keys without string comparison.
> - **Trade-off:** `Object` keys require careful `equals()`/`hashCode()` contracts. String keys are safer for ServiceLoader-discovered accessors where the producer and consumer may be in different classloaders.
> -----------------------------------------------------------

## 1.7 Connection to Real Framework

| Simplified Code | Real Framework Code | File:Line | Key Difference |
|----------------|--------------------|-----------| ---------------|
| `ThreadLocalAccessor<V>` | `ThreadLocalAccessor<V>` | `ThreadLocalAccessor.java:31` | Real has `@Nullable` on `getValue()` via JSpecify |
| `setValue()` (no-arg) | `setValue()` → `reset()` | `ThreadLocalAccessor.java:72` | Real has deprecated `reset()` bridge for backward compat |
| `restore(V)` / `restore()` | same | `ThreadLocalAccessor.java:92,104` | Identical contract |
| `TestThreadLocalAccessor` | `TestThreadLocalAccessor` | `TestThreadLocalAccessor.java:25` | Real version in test sources, same pattern |

**Omitted from simplified version:**
- `@Nullable` / JSpecify nullability annotations (out of scope per outline)
- Deprecated `reset()` method and its backward-compatibility bridge (no legacy code to support)
- `Slf4jThreadLocalAccessor` (SLF4J-specific integration, not core)

## 1.8 Complete Code

### Production Code

#### File: `src/main/java/dev/linhvu/context/ThreadLocalAccessor.java` [NEW]

```java
package dev.linhvu.context;

/**
 * Contract to assist with setting and clearing a {@link ThreadLocal}.
 *
 * <p>A ThreadLocalAccessor bridges a single ThreadLocal value into the
 * context propagation system. Libraries register an accessor for each
 * ThreadLocal they own (e.g., current Observation, current Span), and the
 * propagation infrastructure uses these accessors to capture, restore,
 * and clear values when work moves across thread boundaries.
 *
 * <p>The key design distinction is between {@code setValue}/{@code restore}:
 * <ul>
 *   <li>{@code setValue(V)} / {@code setValue()} are called at <b>scope open</b>
 *       — when a snapshot is applied to a new thread</li>
 *   <li>{@code restore(V)} / {@code restore()} are called at <b>scope close</b>
 *       — when a scope's try-with-resources block ends</li>
 * </ul>
 *
 * <p>For simple ThreadLocals, restore just delegates to setValue. But for
 * scope-aware ThreadLocals (like Observation or Span scopes), the restore
 * path can perform structured cleanup (e.g., closing a scope object) rather
 * than blindly reassigning the value.
 *
 * @param <V> the type of the ThreadLocal value
 * @see ContextRegistry
 */
public interface ThreadLocalAccessor<V> {

    /**
     * The key that uniquely identifies this accessor in the registry.
     * Typically a {@code String} constant, but can be any object.
     */
    Object key();

    /**
     * Read the current value from the ThreadLocal.
     * Returns {@code null} if no value is set.
     */
    V getValue();

    /**
     * Set the given value into the ThreadLocal.
     * Called at scope open when the snapshot contains a value for this key.
     *
     * @param value the value to set (never null)
     */
    void setValue(V value);

    /**
     * Clear the ThreadLocal value.
     * Called at scope open when the snapshot does NOT contain a value for this key
     * (only when {@code clearMissing} is enabled in the snapshot factory).
     */
    default void setValue() {
        // Subclasses should override to clear their ThreadLocal
    }

    /**
     * Restore a previous value into the ThreadLocal.
     * Called at scope close when a previous value existed before the scope was opened.
     *
     * <p>The default delegates to {@link #setValue(Object)}, which is correct for
     * simple ThreadLocals. Override this for scope-aware ThreadLocals where closing
     * a scope is different from setting a value (e.g., closing a Span scope).
     *
     * @param previousValue the value that was present before the scope was opened
     */
    default void restore(V previousValue) {
        setValue(previousValue);
    }

    /**
     * Restore the ThreadLocal to its cleared state.
     * Called at scope close when no previous value existed before the scope was opened.
     *
     * <p>The default delegates to {@link #setValue()}.
     */
    default void restore() {
        setValue();
    }

}
```

### Test Code

#### File: `src/test/java/dev/linhvu/context/TestThreadLocalAccessor.java` [NEW]

```java
package dev.linhvu.context;

/**
 * A simple ThreadLocalAccessor for testing.
 * Wraps a {@code ThreadLocal<String>} with a configurable key.
 */
public class TestThreadLocalAccessor implements ThreadLocalAccessor<String> {

    private final Object key;

    private final ThreadLocal<String> threadLocal;

    public TestThreadLocalAccessor(Object key, ThreadLocal<String> threadLocal) {
        this.key = key;
        this.threadLocal = threadLocal;
    }

    @Override
    public Object key() {
        return this.key;
    }

    @Override
    public String getValue() {
        return this.threadLocal.get();
    }

    @Override
    public void setValue(String value) {
        this.threadLocal.set(value);
    }

    @Override
    public void setValue() {
        this.threadLocal.remove();
    }

}
```

#### File: `src/test/java/dev/linhvu/context/ThreadLocalAccessorTest.java` [NEW]

```java
package dev.linhvu.context;

import org.junit.jupiter.api.AfterEach;
import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;

class ThreadLocalAccessorTest {

    private final ThreadLocal<String> threadLocal = new ThreadLocal<>();

    private final TestThreadLocalAccessor accessor = new TestThreadLocalAccessor("test.key", threadLocal);

    @AfterEach
    void cleanup() {
        threadLocal.remove();
    }

    @Test
    void shouldReturnKey_WhenKeyRequested() {
        assertThat(accessor.key()).isEqualTo("test.key");
    }

    @Test
    void shouldReturnNull_WhenNoValueSet() {
        assertThat(accessor.getValue()).isNull();
    }

    @Test
    void shouldReturnValue_WhenValueSet() {
        accessor.setValue("hello");
        assertThat(accessor.getValue()).isEqualTo("hello");
    }

    @Test
    void shouldClearValue_WhenSetValueCalledWithNoArgs() {
        accessor.setValue("hello");
        accessor.setValue();
        assertThat(accessor.getValue()).isNull();
    }

    @Test
    void shouldRestorePreviousValue_WhenRestoreCalledWithValue() {
        accessor.setValue("current");
        accessor.restore("previous");
        assertThat(accessor.getValue()).isEqualTo("previous");
    }

    @Test
    void shouldClearValue_WhenRestoreCalledWithNoArgs() {
        accessor.setValue("current");
        accessor.restore();
        assertThat(accessor.getValue()).isNull();
    }

    @Test
    void shouldSupportScopeLifecycle_WhenUsedInTryWithResourcesPattern() {
        // Source thread has "original"
        accessor.setValue("original");
        String capturedValue = accessor.getValue();

        // Target thread has "other-thread-value"
        accessor.setValue("other-thread-value");
        String previousValue = accessor.getValue();

        // Scope open: apply captured value
        accessor.setValue(capturedValue);
        assertThat(accessor.getValue()).isEqualTo("original");

        // Scope close: restore previous
        accessor.restore(previousValue);
        assertThat(accessor.getValue()).isEqualTo("other-thread-value");
    }

    @Test
    void shouldSupportScopeLifecycle_WhenNoPreviousValueExisted() {
        accessor.setValue("original");
        String capturedValue = accessor.getValue();

        // Target thread has no value
        accessor.setValue();
        assertThat(accessor.getValue()).isNull();

        // Scope open
        accessor.setValue(capturedValue);
        assertThat(accessor.getValue()).isEqualTo("original");

        // Scope close — no previous value
        accessor.restore();
        assertThat(accessor.getValue()).isNull();
    }

    @Test
    void shouldDefaultRestoreToSetValue_WhenRestoreNotOverridden() {
        ThreadLocalAccessor<String> defaultAccessor = new ThreadLocalAccessor<>() {
            private String value;

            @Override
            public Object key() { return "default"; }

            @Override
            public String getValue() { return value; }

            @Override
            public void setValue(String value) { this.value = value; }

            @Override
            public void setValue() { this.value = null; }
        };

        defaultAccessor.setValue("hello");
        defaultAccessor.restore("world");
        assertThat(defaultAccessor.getValue()).isEqualTo("world");

        defaultAccessor.restore();
        assertThat(defaultAccessor.getValue()).isNull();
    }

}
```

---

## Summary

| Concept | What It Means |
|---------|---------------|
| **ThreadLocalAccessor** | Interface that bridges a single ThreadLocal into the propagation system — defines read, set, clear, and restore |
| **setValue vs restore** | Setting applies a value at scope open; restoring unwinds at scope close — same for simple cases, different for scope-aware ThreadLocals |
| **Object key** | Unique identifier for an accessor in the registry — allows strings, enums, or class literals |
| **Default methods** | `restore(V)` → `setValue(V)` and `restore()` → `setValue()` make the simple case trivial while enabling override for complex cases |

**Next: Chapter 2 — ContextRegistry** — A central registry that holds all ThreadLocalAccessor instances, with a global singleton and ServiceLoader auto-discovery. This is where libraries register their accessors so the propagation system knows what to capture.
