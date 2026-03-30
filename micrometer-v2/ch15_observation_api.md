# Chapter 15: Observation API — Unified Instrumentation

> **Feature 15** · Tier 3 · Depends on: Features 1, 3 (Counter, Timer)

After this chapter you will be able to **instrument code with a single Observation that
produces metrics, traces, and logs simultaneously** — based on which handlers are registered.
This is the "SLF4J for instrumentation" — you code against a unified API, and the signal
type is a deployment decision.

---

## 1. The API Contract

Before writing any implementation, let's define exactly what a client sees.

### What the client imports

```java
import simple.micrometer.api.Observation;
import simple.micrometer.api.ObservationRegistry;
import simple.micrometer.api.ObservationHandler;
```

### What the client writes

```java
// Step 1: Create a registry and register handlers
ObservationRegistry observationRegistry = ObservationRegistry.create();
observationRegistry.observationConfig()
    .observationHandler(new SimpleTimerHandler(meterRegistry));

// Step 2: Instrument a block of code (convenience API)
Observation.createNotStarted("http.request", observationRegistry)
    .lowCardinalityKeyValue("method", "GET")
    .highCardinalityKeyValue("uri", "/api/users/123")
    .observe(() -> handleRequest());

// Step 3: Or use manual lifecycle for more control
Observation observation = Observation.start("db.query", observationRegistry);
try {
    observation.lowCardinalityKeyValue("db", "postgres");
    executeQuery();
} catch (Exception e) {
    observation.error(e);
    throw e;
} finally {
    observation.stop();
}
```

### Implementing a handler

```java
public class TimerObservationHandler implements ObservationHandler<Observation.Context> {

    private final MeterRegistry meterRegistry;

    public TimerObservationHandler(MeterRegistry meterRegistry) {
        this.meterRegistry = meterRegistry;
    }

    @Override
    public void onStart(Observation.Context context) {
        context.put(Timer.Sample.class, Timer.start(meterRegistry));
    }

    @Override
    public void onStop(Observation.Context context) {
        Timer.Sample sample = context.get(Timer.Sample.class);
        Tags tags = Tags.empty();
        for (var entry : context.getLowCardinalityKeyValues().entrySet()) {
            tags = tags.and(entry.getKey(), entry.getValue());
        }
        sample.stop(Timer.builder(context.getName()).tags(tags).register(meterRegistry));
    }

    @Override
    public boolean supportsContext(Observation.Context context) {
        return true;
    }
}
```

### Behavioral contract

| Promise | Detail |
|---------|--------|
| Lifecycle callbacks | Handlers receive `onStart`, `onStop`, `onError`, `onScopeOpened`, `onScopeClosed` at the appropriate lifecycle points |
| Handler ordering | `onStart`/`onScopeOpened` fire in registration (forward) order; `onStop`/`onScopeClosed` fire in reverse (LIFO) order |
| Handler filtering | Each handler declares which context types it supports via `supportsContext()`; filtering happens once at observation creation, not per callback |
| NOOP optimization | If the registry is null, NOOP, or has zero handlers, `createNotStarted` returns `Observation.NOOP` — zero allocation, zero overhead |
| Scope management | `openScope()` makes an observation "current" on the calling thread; closing restores the previous scope (linked-list stack) |
| Parent auto-detection | When creating an observation while a scope is open, the parent is automatically set from the current scope |
| Key-value cardinality | Low-cardinality key values are safe for metric tags (bounded set); high-cardinality are for trace attributes only (unbounded) |

---

## 2. Client Usage & Tests

Tests are written from the client's perspective — exactly as a framework user would call
the API. We write these BEFORE implementing.

### Test 1: Full lifecycle via observe()

```java
@Test
void shouldCallHandlerLifecycleMethods_WhenObserving() {
    List<String> events = new ArrayList<>();
    registry.observationConfig().observationHandler(new ObservationHandler<>() {
        @Override public void onStart(Observation.Context context) { events.add("start:" + context.getName()); }
        @Override public void onStop(Observation.Context context) { events.add("stop:" + context.getName()); }
        @Override public void onScopeOpened(Observation.Context context) { events.add("scopeOpen"); }
        @Override public void onScopeClosed(Observation.Context context) { events.add("scopeClose"); }
        @Override public boolean supportsContext(Observation.Context context) { return true; }
    });

    Observation.createNotStarted("test.observation", registry)
            .lowCardinalityKeyValue("key", "value")
            .observe(() -> { /* work */ });

    assertThat(events).containsExactly(
            "start:test.observation", "scopeOpen", "scopeClose", "stop:test.observation");
}
```

**Why this order?** The `observe()` default method calls `start()`, then opens a scope via
try-with-resources, runs the code, closes the scope (auto-close), and finally calls `stop()`.
The order is: **start → scopeOpen → [work] → scopeClose → stop**.

### Test 2: Error handling

```java
@Test
void shouldCallOnError_WhenObservedCodeThrows() {
    List<String> events = new ArrayList<>();
    registry.observationConfig().observationHandler(new ObservationHandler<>() {
        @Override public void onStart(Observation.Context context) { events.add("start"); }
        @Override public void onStop(Observation.Context context) { events.add("stop"); }
        @Override public void onError(Observation.Context context) { events.add("error:" + context.getError().getMessage()); }
        @Override public void onScopeOpened(Observation.Context context) { events.add("scopeOpen"); }
        @Override public void onScopeClosed(Observation.Context context) { events.add("scopeClose"); }
        @Override public boolean supportsContext(Observation.Context context) { return true; }
    });

    assertThatThrownBy(() ->
            Observation.createNotStarted("failing", registry)
                    .observe((Runnable) () -> { throw new RuntimeException("boom"); })
    ).hasMessage("boom");

    assertThat(events).containsExactly("start", "scopeOpen", "scopeClose", "error:boom", "stop");
}
```

**Why scopeClose before error?** Java's try-with-resources calls `close()` before the catch
block. So the sequence is: `runnable` throws → `scope.close()` (try-with-resources) → catch
block calls `error()` → finally block calls `stop()`.

### Test 3: Handler ordering — forward start, reverse stop

```java
@Test
void shouldCallMultipleHandlersInCorrectOrder() {
    List<String> events = new ArrayList<>();

    registry.observationConfig()
            .observationHandler(handlerNamed("A", events))
            .observationHandler(handlerNamed("B", events));

    Observation.createNotStarted("test", registry).observe(() -> {});

    // Forward for start/scopeOpen, reverse for scopeClose/stop
    assertThat(events).containsExactly(
            "A:start", "B:start",
            "A:scopeOpen", "B:scopeOpen",
            "B:scopeClose", "A:scopeClose",
            "B:stop", "A:stop");
}
```

**Why reverse order for stop/close?** This is the decorator/stack pattern. If handler A
acquires a resource on start and handler B acquires another, B should release first (LIFO).
Think of it like nested try-with-resources blocks.

### Test 4: NOOP when no handlers

```java
@Test
void shouldReturnNoopWhenRegistryHasNoHandlers() {
    ObservationRegistry emptyRegistry = ObservationRegistry.create();
    Observation obs = Observation.createNotStarted("test", emptyRegistry);
    assertThat(obs).isSameAs(Observation.NOOP);
}

@Test
void noopObservationShouldStillExecuteCode() {
    boolean[] ran = { false };
    Observation.NOOP.observe(() -> ran[0] = true);
    assertThat(ran[0]).isTrue();
}
```

### Test 5: Timer integration via handler

```java
@Test
void shouldCreateTimerMetricsFromObservationViaHandler() {
    MeterRegistry meterRegistry = new SimpleMeterRegistry(Clock.SYSTEM);
    registry.observationConfig().observationHandler(new TimerObservationHandler(meterRegistry));

    Observation.createNotStarted("http.request", registry)
            .lowCardinalityKeyValue("method", "GET")
            .observe(() -> { /* work */ });

    Timer timer = meterRegistry.find("http.request").tag("method", "GET").timer();
    assertThat(timer).isNotNull();
    assertThat(timer.count()).isEqualTo(1);
}
```

---

## 3. Implementing the Call Chain

### Call chain: observe()

```
Client calls: Observation.createNotStarted("http.request", registry)
                .lowCardinalityKeyValue("method", "GET")
                .observe(() -> handleRequest())
  │
  ├─► [Factory] Observation.createNotStarted(name, registry)
  │     1. If registry is null/NOOP/no handlers → return Observation.NOOP
  │     2. Create Context, set name, auto-set parent from current scope
  │     3. Return new SimpleObservation(registry, context)
  │        └─ Filters handlers: only those where supportsContext(ctx) == true
  │
  ├─► [Default Method] Observation.observe(Runnable)
  │     Calls start() → try (openScope()) { run } catch { error() } finally { stop() }
  │
  ├─► [Impl] SimpleObservation.start()
  │     Iterates handlers in FORWARD order → handler.onStart(context)
  │
  ├─► [Impl] SimpleObservation.openScope()
  │     1. Creates SimpleScope(registry, this)
  │        └─ Captures previousScope from registry ThreadLocal
  │        └─ Sets itself as current scope on the ThreadLocal
  │     2. Iterates handlers in FORWARD order → handler.onScopeOpened(context)
  │
  ├─► [Runnable executes — handlers can read/write context during this time]
  │
  ├─► [Impl] SimpleScope.close()  (try-with-resources)
  │     1. Iterates handlers in REVERSE order → handler.onScopeClosed(context)
  │     2. Restores previousScope on the ThreadLocal
  │
  └─► [Impl] SimpleObservation.stop()  (finally)
        Iterates handlers in REVERSE order → handler.onStop(context)
```

### 3a. API Layer — ObservationHandler

The handler interface defines the lifecycle callbacks. All callbacks have default no-op
implementations, so handlers only override what they care about. The only required method
is `supportsContext()`.

```java
public interface ObservationHandler<T extends Observation.Context> {

    default void onStart(T context) {}
    default void onStop(T context) {}
    default void onError(T context) {}
    default void onScopeOpened(T context) {}
    default void onScopeClosed(T context) {}

    boolean supportsContext(Observation.Context context);
}
```

**Why a type parameter `<T extends Context>`?** Handlers can declare interest in specific
context subtypes. A handler declared as `ObservationHandler<HttpContext>` receives a typed
`HttpContext` in its callbacks, avoiding casts. The `supportsContext` method checks at
runtime whether the handler can handle the actual context instance.

### 3b. API Layer — Observation

The Observation interface combines three roles:

1. **Static factory** — `createNotStarted()` and `start()` decide real vs. NOOP based on
   registry state
2. **Fluent API** — `lowCardinalityKeyValue()`, `highCardinalityKeyValue()`, `error()`
3. **Lifecycle template** — default `observe()` methods implement the try-with-resources
   pattern so every user doesn't have to

The key factory logic:

```java
static <T extends Context> Observation createNotStarted(String name,
        Supplier<T> contextSupplier, ObservationRegistry registry) {
    if (registry == null || registry.isNoop()) {
        return NOOP;
    }
    T context = contextSupplier.get();
    context.setName(name);
    // Auto-set parent from the current observation scope
    Observation current = registry.getCurrentObservation();
    if (current != null && current != NOOP) {
        context.setParentObservation(current);
    }
    return new SimpleObservation(registry, context);
}
```

**Why check `registry.isNoop()` instead of just `registry == NOOP`?** A `SimpleObservationRegistry`
with zero handlers is also "NOOP" — its `isNoop()` returns `true` when the handler list is
empty. This avoids creating a real `SimpleObservation` only to have no handlers to call.

The `observe()` default methods use Java's precise rethrow feature:

```java
default void observe(Runnable runnable) {
    start();
    try (Scope scope = openScope()) {
        runnable.run();
    }
    catch (Throwable error) {
        error(error);
        throw error;    // Precise rethrow: compiler knows only unchecked types possible
    }
    finally {
        stop();
    }
}
```

**Why `catch (Throwable)` compiles without `throws`?** Since Java 7, the compiler performs
"precise rethrow" analysis. It examines what the try block can actually throw — `Runnable.run()`
doesn't declare checked exceptions, and our `Scope.close()` narrows `AutoCloseable.close()`
to not throw checked exceptions. So the compiler knows `error` can only be `RuntimeException`
or `Error`, both unchecked.

#### Context inner class

The Context is a mutable state carrier that travels through all handler callbacks:

```java
class Context {
    private String name;
    private Throwable error;
    private Observation parentObservation;
    private final Map<String, String> lowCardinalityKeyValues = new ConcurrentHashMap<>();
    private final Map<String, String> highCardinalityKeyValues = new ConcurrentHashMap<>();
    private final Map<Object, Object> map = new ConcurrentHashMap<>();

    // Handlers use put/get to pass state between callbacks:
    public <T> void put(Object key, T value) { map.put(key, value); }
    public <T> T get(Object key) { return (T) map.get(key); }
}
```

**Why two cardinality levels?** Low-cardinality key values (e.g., HTTP method, status code)
have a bounded number of distinct values and are safe to use as metric tags. High-cardinality
values (e.g., request URI, user ID) have unbounded distinct values and would cause a
"cardinality explosion" if used as metric tags — they're only safe for trace attributes.

**Why the arbitrary `Map<Object, Object>`?** Handlers need to pass state between callbacks.
A timer handler stores a `Timer.Sample` on `onStart` and retrieves it on `onStop`. Using
`Class` objects as keys provides type-safe lookup:
```java
context.put(Timer.Sample.class, sample);    // onStart
Timer.Sample s = context.get(Timer.Sample.class);  // onStop
```

#### Scope inner interface

```java
interface Scope extends AutoCloseable {
    Observation getCurrentObservation();
    @Override void close();  // Narrows: no checked exceptions
}
```

**Why override `close()` without throws?** `AutoCloseable.close()` declares `throws Exception`.
By overriding without `throws`, we narrow the exception specification. This has two effects:
(1) implementations can't throw checked exceptions from `close()`, and (2) the `observe()`
method's try-with-resources doesn't introduce checked exceptions, enabling precise rethrow.

### 3c. API Layer — ObservationRegistry

The registry holds the handler configuration and tracks the current scope per thread:

```java
public interface ObservationRegistry {
    static ObservationRegistry create();  // Returns SimpleObservationRegistry
    Observation getCurrentObservation();
    Observation.Scope getCurrentObservationScope();
    void setCurrentObservationScope(Observation.Scope scope);
    ObservationConfig observationConfig();
    default boolean isNoop() { return this == NOOP; }
}
```

The `ObservationConfig` inner class uses `CopyOnWriteArrayList` for thread-safe handler
registration:

```java
class ObservationConfig {
    private final List<ObservationHandler<?>> handlers = new CopyOnWriteArrayList<>();

    public ObservationConfig observationHandler(ObservationHandler<?> handler) {
        handlers.add(handler);
        return this;
    }
}
```

### 3d. Implementation Layer — SimpleObservationRegistry

A thin implementation that uses a static `ThreadLocal` to track the current scope:

```java
public class SimpleObservationRegistry implements ObservationRegistry {
    private static final ThreadLocal<Observation.Scope> localScope = new ThreadLocal<>();
    private final ObservationConfig config = new ObservationConfig();

    @Override
    public boolean isNoop() {
        return ObservationRegistry.super.isNoop()
                || config.getObservationHandlers().isEmpty();
    }
}
```

**Why is the ThreadLocal static?** In the real Micrometer, there is a single scope chain per
thread regardless of which registry created the observations. Making it static matches this
design — scopes from different registries share the same thread-local stack.

**Why `isNoop()` checks for empty handlers?** This is an important optimization. If no
handlers are registered, creating a `SimpleObservation` would allocate objects and iterate
empty lists on every callback. By returning `true` from `isNoop()`, the factory method
short-circuits to `Observation.NOOP`.

### 3e. Implementation Layer — SimpleObservation

The core implementation that manages handler callbacks:

```java
public class SimpleObservation implements Observation {
    private final ObservationRegistry registry;
    private final Context context;
    private final Deque<ObservationHandler<Context>> handlers;

    public SimpleObservation(ObservationRegistry registry, Context context) {
        this.registry = registry;
        this.context = context;
        this.handlers = new ArrayDeque<>();
        // Filter once at creation time
        for (ObservationHandler<?> handler : registry.observationConfig().getObservationHandlers()) {
            if (handler.supportsContext(context)) {
                handlers.add((ObservationHandler<Context>) handler);
            }
        }
    }
}
```

**Why filter handlers at creation time?** The real framework also does this. Since
`supportsContext()` is deterministic for a given context, checking once and keeping the
result avoids N checks on every lifecycle callback. The filtered handlers are stored in
an `ArrayDeque`.

**Why `ArrayDeque` instead of `ArrayList`?** We need efficient iteration in both forward
and reverse order. `ArrayDeque` provides `descendingIterator()` for O(n) reverse iteration
without creating a reversed copy. The forward iteration uses the standard for-each loop
(the `Deque` implements `Iterable`).

#### SimpleScope — linked-list stack via ThreadLocal

```java
static class SimpleScope implements Scope {
    private final ObservationRegistry registry;
    private final SimpleObservation observation;
    private final Scope previousScope;

    SimpleScope(ObservationRegistry registry, SimpleObservation observation) {
        this.registry = registry;
        this.observation = observation;
        this.previousScope = registry.getCurrentObservationScope();  // capture
        registry.setCurrentObservationScope(this);                   // install
    }

    @Override
    public void close() {
        observation.notifyOnScopeClosed();
        registry.setCurrentObservationScope(previousScope);  // restore
    }
}
```

Each scope captures the previous scope and restores it on close. This naturally forms a
stack without any explicit stack data structure:

```
Thread-local state:  null → [ScopeA] → [ScopeB (current)]
                             ↑ previousScope

On ScopeB.close():   null → [ScopeA (current)]
```

This is how parent-child observation tracking works: when creating a child observation,
`Observation.createNotStarted` calls `registry.getCurrentObservation()`, which walks the
scope chain to find the parent.

---

## 4. Try It Yourself

1. **Logging handler**: Write an `ObservationHandler` that logs observation start/stop
   with name, duration, and key values to `System.out`.

2. **Custom context**: Create an `HttpContext extends Observation.Context` with fields for
   `method`, `uri`, and `statusCode`. Write a handler typed as
   `ObservationHandler<HttpContext>` that only processes HTTP observations.

3. **Counter handler**: Write a handler that increments a Counter (instead of recording a
   Timer) on each `onStop`. Use the observation name + low-cardinality key values as the
   counter's identity.

4. **Error rate tracking**: Extend the counter handler to maintain two counters — one for
   total observations and one for observations with errors. Check `context.getError() != null`
   in `onStop` to decide which counter to increment.

---

## 5. Why This Works

### The Bridge Pattern
The Observation API is a textbook bridge pattern. The abstraction (`Observation`) is
completely decoupled from the implementation (metrics, traces, logs). Adding a new signal
type means adding a new handler — zero changes to application instrumentation code.

### NOOP as a First-Class Citizen
The NOOP pattern appears at three levels: `Observation.NOOP`, `ObservationRegistry.NOOP`,
and `Scope.NOOP`. This isn't just defensive coding — it's a performance design. In
production, you might have thousands of instrumentation points. When a registry has no
handlers (e.g., in unit tests), every `createNotStarted` returns a singleton NOOP with
zero allocation, zero callback overhead. The NOOP's `observe()` just runs the code directly.

### Scope as Implicit Context Propagation
Scopes solve a fundamental problem: how does a child operation know its parent? Rather than
explicitly passing a parent observation through every method signature, the scope mechanism
uses a `ThreadLocal` to make the current observation implicitly available. This is the same
pattern used by SLF4J's MDC, Spring Security's `SecurityContextHolder`, and distributed
tracing's `Span.current()`.

### Key-Value Cardinality as an API Concern
Separating low-cardinality from high-cardinality key values at the API level (not at the
handler level) is a deliberate design choice. It shifts the cardinality decision from the
handler author to the instrumentation author, who understands the data best. Handlers can
then safely use low-cardinality values as metric tags and high-cardinality values as trace
attributes without risk of cardinality explosion.

---

## 6. What We Enhanced

| File | Change | Why |
|------|--------|-----|
| *(no existing files modified)* | — | Feature 15 is entirely additive — it introduces the Observation API layer that sits *above* the existing metric types |

**Why no modifications?** The Observation API lives in the `micrometer-observation` module
in the real framework, separate from `micrometer-core`. It doesn't change existing meter
types — it adds a higher-level abstraction that *uses* them via handlers. The
`TimerObservationHandler` example in our tests demonstrates this bridge: it uses the
existing `Timer` and `Timer.Sample` APIs from Feature 3 without modifying them.

---

## 7. Connection to Real Framework

| Simplified Class | Real Framework Class | What We Simplified |
|------------------|---------------------|--------------------|
| `Observation` | `io.micrometer.observation.Observation` | Removed `ObservationConvention` support, `Event` interface, `CheckedRunnable`/`CheckedCallable` variants, `scoped()`/`wrap()` methods |
| `Observation.Context` | `io.micrometer.observation.Observation.Context` | Used `Map<String, String>` for key values instead of `KeyValue`/`KeyValues` classes |
| `Observation.Scope` | `io.micrometer.observation.Observation.Scope` | Removed `reset()` and `makeCurrent()` (used for cross-thread context propagation) |
| `ObservationRegistry` | `io.micrometer.observation.ObservationRegistry` | Combined `ObservationConfig` into the interface as an inner class |
| `ObservationHandler` | `io.micrometer.observation.ObservationHandler` | Removed `FirstMatchingCompositeObservationHandler`, `AllMatchingCompositeObservationHandler` |
| `SimpleObservation` | `io.micrometer.observation.SimpleObservation` | Removed convention resolution, filter application, `onScopeReset`/`onScopeMakeCurrent` |
| `SimpleObservationRegistry` | `io.micrometer.observation.SimpleObservationRegistry` | Same behavior, just simpler |

### What we skipped and why

| Skipped Concept | Why |
|----------------|-----|
| `ObservationConvention` | Provides naming/key-value defaults and overrides — an advanced customization layer that adds complexity without teaching new patterns |
| `ObservationFilter` | Mutates the context at stop time — similar to `MeterFilter.map()` which we covered in Feature 5 |
| `ObservationPredicate` | Gates observation creation — similar to `MeterFilter.accept()` from Feature 5 |
| `NoopButScopeHandlingObservation` | A disabled-but-scope-aware NOOP for context propagation — needed only when integrating with `micrometer-tracing` |
| `NullObservation` | Used by `ObservationThreadLocalAccessor` for cross-thread context propagation — beyond our scope |
| Composite handlers | `FirstMatchingCompositeObservationHandler` and `AllMatchingCompositeObservationHandler` are convenience wrappers for handler delegation strategies |
| `Event` interface | Used for signaling mid-observation events (e.g., "message.sent") — adds API surface without teaching new internals |

**Real source files** (commit reference: `micrometer-observation` module):
- `Observation.java` — 1552 lines (ours: ~250 lines)
- `ObservationRegistry.java` — 221 lines (ours: ~145 lines)
- `ObservationHandler.java` — 329 lines (ours: ~95 lines)
- `SimpleObservation.java` — 398 lines (ours: ~193 lines)
- `SimpleObservationRegistry.java` — 75 lines (ours: ~60 lines)

---

## 8. Complete Code

All files created for this feature. Copy all `[NEW]` files to get a compiling project
with all tests passing.

### `src/main/java/simple/micrometer/api/ObservationHandler.java` [NEW]

```java
package simple.micrometer.api;

/**
 * Receives lifecycle callbacks from an {@link Observation}. Handlers are the
 * extension point where observation signals are converted into concrete
 * telemetry — metrics, traces, logs, or any combination.
 *
 * <p>A handler registers interest in specific {@link Observation.Context} types
 * via {@link #supportsContext}. Only handlers that support the context type
 * are notified of lifecycle events. This filtering happens once at observation
 * creation time, not on every callback.
 *
 * <p>Example — a handler that creates Timer metrics from observations:
 * <pre>{@code
 * public class TimerHandler implements ObservationHandler<Observation.Context> {
 *     private final MeterRegistry registry;
 *
 *     public TimerHandler(MeterRegistry registry) {
 *         this.registry = registry;
 *     }
 *
 *     @Override
 *     public void onStart(Observation.Context context) {
 *         context.put(Timer.Sample.class, Timer.start(registry));
 *     }
 *
 *     @Override
 *     public void onStop(Observation.Context context) {
 *         Timer.Sample sample = context.get(Timer.Sample.class);
 *         sample.stop(Timer.builder(context.getName()).register(registry));
 *     }
 *
 *     @Override
 *     public boolean supportsContext(Observation.Context context) {
 *         return true;
 *     }
 * }
 * }</pre>
 *
 * <p>Simplified from {@code io.micrometer.observation.ObservationHandler}.
 *
 * @param <T> the context type this handler can work with
 */
public interface ObservationHandler<T extends Observation.Context> {

    /**
     * Called when an observation is started via {@link Observation#start()}.
     * Handlers are notified in forward (registration) order.
     */
    default void onStart(T context) {
    }

    /**
     * Called when an observation is stopped via {@link Observation#stop()}.
     * Handlers are notified in <b>reverse</b> (LIFO) order — the last handler
     * to receive {@code onStart} is the first to receive {@code onStop}.
     */
    default void onStop(T context) {
    }

    /**
     * Called when an error is recorded via {@link Observation#error(Throwable)}.
     * The error is available on {@link Observation.Context#getError()}.
     * Handlers are notified in forward order.
     */
    default void onError(T context) {
    }

    /**
     * Called when a scope is opened via {@link Observation#openScope()}.
     * Handlers are notified in forward order.
     */
    default void onScopeOpened(T context) {
    }

    /**
     * Called when a scope is closed via {@link Observation.Scope#close()}.
     * Handlers are notified in <b>reverse</b> order.
     */
    default void onScopeClosed(T context) {
    }

    /**
     * Returns {@code true} if this handler can handle observations with the
     * given context type. This is called once at observation creation time to
     * filter the handler list — only matching handlers receive lifecycle
     * callbacks for that observation.
     *
     * @param context the observation's context (check its class or name)
     * @return true if this handler should receive callbacks for this observation
     */
    boolean supportsContext(Observation.Context context);

}
```

### `src/main/java/simple/micrometer/api/Observation.java` [NEW]

```java
package simple.micrometer.api;

import simple.micrometer.internal.SimpleObservation;

import java.util.Collections;
import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;
import java.util.function.Supplier;

/**
 * A higher-level instrumentation API that decouples code instrumentation from
 * the specific signal type. A single Observation can produce metrics (via Timer),
 * traces (via Span), and logs simultaneously — based on which
 * {@link ObservationHandler}s are registered with the {@link ObservationRegistry}.
 *
 * <p>Usage — wrapping a block of code:
 * <pre>{@code
 * Observation.createNotStarted("http.request", observationRegistry)
 *     .lowCardinalityKeyValue("method", "GET")
 *     .highCardinalityKeyValue("uri", "/api/users/123")
 *     .observe(() -> handleRequest());
 * }</pre>
 *
 * <p>Usage — manual lifecycle:
 * <pre>{@code
 * Observation observation = Observation.start("db.query", observationRegistry);
 * try {
 *     observation.lowCardinalityKeyValue("db", "postgres");
 *     executeQuery();
 * } catch (Exception e) {
 *     observation.error(e);
 *     throw e;
 * } finally {
 *     observation.stop();
 * }
 * }</pre>
 *
 * <p>Simplified from {@code io.micrometer.observation.Observation}.
 */
public interface Observation {

    // ── Static Factory Methods ───────────────────────────────────────

    static Observation createNotStarted(String name, ObservationRegistry registry) {
        return createNotStarted(name, Context::new, registry);
    }

    static <T extends Context> Observation createNotStarted(String name,
            Supplier<T> contextSupplier, ObservationRegistry registry) {
        if (registry == null || registry.isNoop()) {
            return NOOP;
        }
        T context = contextSupplier.get();
        context.setName(name);
        Observation current = registry.getCurrentObservation();
        if (current != null && current != NOOP) {
            context.setParentObservation(current);
        }
        return new SimpleObservation(registry, context);
    }

    static Observation start(String name, ObservationRegistry registry) {
        return createNotStarted(name, registry).start();
    }

    static <T extends Context> Observation start(String name,
            Supplier<T> contextSupplier, ObservationRegistry registry) {
        return createNotStarted(name, contextSupplier, registry).start();
    }

    // ── Instance Methods ─────────────────────────────────────────────

    Observation lowCardinalityKeyValue(String key, String value);
    Observation highCardinalityKeyValue(String key, String value);
    Observation error(Throwable error);
    Observation start();
    Context getContext();
    void stop();
    Scope openScope();

    // ── Default observe() Methods ────────────────────────────────────

    default void observe(Runnable runnable) {
        start();
        try (Scope scope = openScope()) {
            runnable.run();
        }
        catch (Throwable error) {
            error(error);
            throw error;
        }
        finally {
            stop();
        }
    }

    default <T> T observe(Supplier<T> supplier) {
        start();
        try (Scope scope = openScope()) {
            return supplier.get();
        }
        catch (Throwable error) {
            error(error);
            throw error;
        }
        finally {
            stop();
        }
    }

    // ── NOOP Singleton ───────────────────────────────────────────────

    Observation NOOP = new Observation() {

        private final Context noopContext = new Context();

        @Override public Observation lowCardinalityKeyValue(String key, String value) { return this; }
        @Override public Observation highCardinalityKeyValue(String key, String value) { return this; }
        @Override public Observation error(Throwable error) { return this; }
        @Override public Observation start() { return this; }
        @Override public Context getContext() { return noopContext; }
        @Override public void stop() {}
        @Override public Scope openScope() { return Scope.NOOP; }
        @Override public void observe(Runnable runnable) { runnable.run(); }
        @Override public <T> T observe(Supplier<T> supplier) { return supplier.get(); }
    };

    // ── Inner Class: Context ─────────────────────────────────────────

    class Context {

        private String name;
        private String contextualName;
        private Throwable error;
        private Observation parentObservation;
        private final Map<String, String> lowCardinalityKeyValues = new ConcurrentHashMap<>();
        private final Map<String, String> highCardinalityKeyValues = new ConcurrentHashMap<>();
        private final Map<Object, Object> map = new ConcurrentHashMap<>();

        public String getName() { return name; }
        public void setName(String name) { this.name = name; }
        public String getContextualName() { return contextualName != null ? contextualName : name; }
        public void setContextualName(String contextualName) { this.contextualName = contextualName; }
        public Throwable getError() { return error; }
        public void setError(Throwable error) { this.error = error; }
        public Observation getParentObservation() { return parentObservation; }
        public void setParentObservation(Observation parentObservation) { this.parentObservation = parentObservation; }

        public void addLowCardinalityKeyValue(String key, String value) { lowCardinalityKeyValues.put(key, value); }
        public void addHighCardinalityKeyValue(String key, String value) { highCardinalityKeyValues.put(key, value); }
        public Map<String, String> getLowCardinalityKeyValues() { return Collections.unmodifiableMap(lowCardinalityKeyValues); }
        public Map<String, String> getHighCardinalityKeyValues() { return Collections.unmodifiableMap(highCardinalityKeyValues); }

        public <T> void put(Object key, T value) { map.put(key, value); }
        @SuppressWarnings("unchecked")
        public <T> T get(Object key) { return (T) map.get(key); }
        public boolean containsKey(Object key) { return map.containsKey(key); }
    }

    // ── Inner Interface: Scope ───────────────────────────────────────

    interface Scope extends AutoCloseable {
        Observation getCurrentObservation();
        @Override void close();

        Scope NOOP = new Scope() {
            @Override public Observation getCurrentObservation() { return Observation.NOOP; }
            @Override public void close() {}
        };
    }

}
```

### `src/main/java/simple/micrometer/api/ObservationRegistry.java` [NEW]

```java
package simple.micrometer.api;

import simple.micrometer.internal.SimpleObservationRegistry;

import java.util.Collection;
import java.util.Collections;
import java.util.List;
import java.util.concurrent.CopyOnWriteArrayList;

/**
 * Registry that manages observation configuration and tracks the current
 * observation scope on each thread. The registry holds the list of
 * {@link ObservationHandler}s that receive lifecycle callbacks.
 *
 * <p>Simplified from {@code io.micrometer.observation.ObservationRegistry}.
 */
public interface ObservationRegistry {

    static ObservationRegistry create() {
        return new SimpleObservationRegistry();
    }

    Observation getCurrentObservation();
    Observation.Scope getCurrentObservationScope();
    void setCurrentObservationScope(Observation.Scope scope);
    ObservationConfig observationConfig();

    default boolean isNoop() {
        return this == NOOP;
    }

    ObservationRegistry NOOP = new ObservationRegistry() {

        private final ObservationConfig noopConfig = new ObservationConfig() {
            @Override
            public ObservationConfig observationHandler(ObservationHandler<?> handler) {
                return this;
            }
        };

        @Override public Observation getCurrentObservation() { return null; }
        @Override public Observation.Scope getCurrentObservationScope() { return null; }
        @Override public void setCurrentObservationScope(Observation.Scope scope) {}
        @Override public ObservationConfig observationConfig() { return noopConfig; }
    };

    class ObservationConfig {

        private final List<ObservationHandler<?>> handlers = new CopyOnWriteArrayList<>();

        public ObservationConfig observationHandler(ObservationHandler<?> handler) {
            handlers.add(handler);
            return this;
        }

        public Collection<ObservationHandler<?>> getObservationHandlers() {
            return Collections.unmodifiableList(handlers);
        }
    }

}
```

### `src/main/java/simple/micrometer/internal/SimpleObservationRegistry.java` [NEW]

```java
package simple.micrometer.internal;

import simple.micrometer.api.Observation;
import simple.micrometer.api.ObservationRegistry;

/**
 * Default implementation of {@link ObservationRegistry}. Uses a static
 * {@link ThreadLocal} to track the current observation scope per thread.
 *
 * <p>Simplified from {@code io.micrometer.observation.SimpleObservationRegistry}.
 */
public class SimpleObservationRegistry implements ObservationRegistry {

    private static final ThreadLocal<Observation.Scope> localScope = new ThreadLocal<>();

    private final ObservationConfig config = new ObservationConfig();

    @Override
    public Observation getCurrentObservation() {
        Observation.Scope scope = localScope.get();
        return scope != null ? scope.getCurrentObservation() : null;
    }

    @Override
    public Observation.Scope getCurrentObservationScope() {
        return localScope.get();
    }

    @Override
    public void setCurrentObservationScope(Observation.Scope scope) {
        localScope.set(scope);
    }

    @Override
    public ObservationConfig observationConfig() {
        return config;
    }

    @Override
    public boolean isNoop() {
        return ObservationRegistry.super.isNoop()
                || config.getObservationHandlers().isEmpty();
    }

}
```

### `src/main/java/simple/micrometer/internal/SimpleObservation.java` [NEW]

```java
package simple.micrometer.internal;

import simple.micrometer.api.Observation;
import simple.micrometer.api.ObservationHandler;
import simple.micrometer.api.ObservationRegistry;

import java.util.ArrayDeque;
import java.util.Deque;
import java.util.Iterator;

/**
 * Default implementation of {@link Observation} that manages the lifecycle
 * callbacks to registered {@link ObservationHandler}s.
 *
 * <p>Simplified from {@code io.micrometer.observation.SimpleObservation}.
 */
public class SimpleObservation implements Observation {

    private final ObservationRegistry registry;
    private final Context context;
    private final Deque<ObservationHandler<Context>> handlers;

    @SuppressWarnings("unchecked")
    public SimpleObservation(ObservationRegistry registry, Context context) {
        this.registry = registry;
        this.context = context;
        this.handlers = new ArrayDeque<>();
        for (ObservationHandler<?> handler : registry.observationConfig().getObservationHandlers()) {
            if (handler.supportsContext(context)) {
                handlers.add((ObservationHandler<Context>) handler);
            }
        }
    }

    @Override
    public Observation lowCardinalityKeyValue(String key, String value) {
        context.addLowCardinalityKeyValue(key, value);
        return this;
    }

    @Override
    public Observation highCardinalityKeyValue(String key, String value) {
        context.addHighCardinalityKeyValue(key, value);
        return this;
    }

    @Override
    public Observation error(Throwable error) {
        context.setError(error);
        for (ObservationHandler<Context> handler : handlers) {
            handler.onError(context);
        }
        return this;
    }

    @Override
    public Observation start() {
        for (ObservationHandler<Context> handler : handlers) {
            handler.onStart(context);
        }
        return this;
    }

    @Override
    public Context getContext() {
        return context;
    }

    @Override
    public void stop() {
        Iterator<ObservationHandler<Context>> it = handlers.descendingIterator();
        while (it.hasNext()) {
            it.next().onStop(context);
        }
    }

    @Override
    public Scope openScope() {
        SimpleScope scope = new SimpleScope(registry, this);
        for (ObservationHandler<Context> handler : handlers) {
            handler.onScopeOpened(context);
        }
        return scope;
    }

    void notifyOnScopeClosed() {
        Iterator<ObservationHandler<Context>> it = handlers.descendingIterator();
        while (it.hasNext()) {
            it.next().onScopeClosed(context);
        }
    }

    static class SimpleScope implements Scope {

        private final ObservationRegistry registry;
        private final SimpleObservation observation;
        private final Scope previousScope;

        SimpleScope(ObservationRegistry registry, SimpleObservation observation) {
            this.registry = registry;
            this.observation = observation;
            this.previousScope = registry.getCurrentObservationScope();
            registry.setCurrentObservationScope(this);
        }

        @Override
        public Observation getCurrentObservation() {
            return observation;
        }

        @Override
        public void close() {
            observation.notifyOnScopeClosed();
            registry.setCurrentObservationScope(previousScope);
        }
    }

}
```

### `src/test/java/simple/micrometer/api/ObservationTest.java` [NEW]

```java
package simple.micrometer.api;

import simple.micrometer.internal.SimpleMeterRegistry;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.util.ArrayList;
import java.util.List;
import java.util.concurrent.TimeUnit;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

class ObservationTest {

    ObservationRegistry registry;

    @BeforeEach
    void setUp() {
        registry = ObservationRegistry.create();
    }

    @Test
    void shouldCallHandlerLifecycleMethods_WhenObserving() {
        List<String> events = new ArrayList<>();
        registry.observationConfig().observationHandler(new ObservationHandler<>() {
            @Override public void onStart(Observation.Context context) { events.add("start:" + context.getName()); }
            @Override public void onStop(Observation.Context context) { events.add("stop:" + context.getName()); }
            @Override public void onScopeOpened(Observation.Context context) { events.add("scopeOpen"); }
            @Override public void onScopeClosed(Observation.Context context) { events.add("scopeClose"); }
            @Override public boolean supportsContext(Observation.Context context) { return true; }
        });

        Observation.createNotStarted("test.observation", registry)
                .lowCardinalityKeyValue("key", "value")
                .observe(() -> { });

        assertThat(events).containsExactly(
                "start:test.observation", "scopeOpen", "scopeClose", "stop:test.observation");
    }

    @Test
    void shouldReturnResultFromObserveWithSupplier() {
        registry.observationConfig().observationHandler(new ObservationHandler<>() {
            @Override public boolean supportsContext(Observation.Context context) { return true; }
        });

        String result = Observation.createNotStarted("compute", registry)
                .observe(() -> "hello world");

        assertThat(result).isEqualTo("hello world");
    }

    @Test
    void shouldCallOnError_WhenObservedCodeThrows() {
        List<String> events = new ArrayList<>();
        registry.observationConfig().observationHandler(new ObservationHandler<>() {
            @Override public void onStart(Observation.Context context) { events.add("start"); }
            @Override public void onStop(Observation.Context context) { events.add("stop"); }
            @Override public void onError(Observation.Context context) { events.add("error:" + context.getError().getMessage()); }
            @Override public void onScopeOpened(Observation.Context context) { events.add("scopeOpen"); }
            @Override public void onScopeClosed(Observation.Context context) { events.add("scopeClose"); }
            @Override public boolean supportsContext(Observation.Context context) { return true; }
        });

        assertThatThrownBy(() ->
                Observation.createNotStarted("failing", registry)
                        .observe((Runnable) () -> { throw new RuntimeException("boom"); })
        ).hasMessage("boom");

        assertThat(events).containsExactly("start", "scopeOpen", "scopeClose", "error:boom", "stop");
    }

    @Test
    void shouldSupportManualStartAndStop() {
        List<String> events = new ArrayList<>();
        registry.observationConfig().observationHandler(new ObservationHandler<>() {
            @Override public void onStart(Observation.Context context) { events.add("start"); }
            @Override public void onStop(Observation.Context context) { events.add("stop"); }
            @Override public boolean supportsContext(Observation.Context context) { return true; }
        });

        Observation observation = Observation.start("manual", registry);
        assertThat(events).containsExactly("start");

        observation.lowCardinalityKeyValue("db", "postgres");
        observation.stop();

        assertThat(events).containsExactly("start", "stop");
        assertThat(observation.getContext().getLowCardinalityKeyValues())
                .containsEntry("db", "postgres");
    }

    @Test
    void shouldManuallyHandleErrorAndStop() {
        List<String> events = new ArrayList<>();
        registry.observationConfig().observationHandler(new ObservationHandler<>() {
            @Override public void onStart(Observation.Context context) { events.add("start"); }
            @Override public void onStop(Observation.Context context) { events.add("stop"); }
            @Override public void onError(Observation.Context context) { events.add("error"); }
            @Override public boolean supportsContext(Observation.Context context) { return true; }
        });

        Observation observation = Observation.start("db.query", registry);
        try {
            observation.lowCardinalityKeyValue("db", "postgres");
            throw new RuntimeException("connection timeout");
        } catch (Exception e) {
            observation.error(e);
            assertThat(observation.getContext().getError()).hasMessage("connection timeout");
        } finally {
            observation.stop();
        }

        assertThat(events).containsExactly("start", "error", "stop");
    }

    @Test
    void shouldTrackLowAndHighCardinalityKeyValues() {
        registry.observationConfig().observationHandler(new ObservationHandler<>() {
            @Override public boolean supportsContext(Observation.Context context) { return true; }
        });

        Observation observation = Observation.createNotStarted("http.request", registry)
                .lowCardinalityKeyValue("method", "GET")
                .lowCardinalityKeyValue("status", "200")
                .highCardinalityKeyValue("uri", "/api/users/123");

        Observation.Context ctx = observation.getContext();
        assertThat(ctx.getLowCardinalityKeyValues())
                .containsEntry("method", "GET")
                .containsEntry("status", "200");
        assertThat(ctx.getHighCardinalityKeyValues())
                .containsEntry("uri", "/api/users/123");
    }

    @Test
    void shouldOverwriteKeyValueWithSameKey() {
        registry.observationConfig().observationHandler(new ObservationHandler<>() {
            @Override public boolean supportsContext(Observation.Context context) { return true; }
        });

        Observation observation = Observation.createNotStarted("http.request", registry)
                .lowCardinalityKeyValue("status", "200")
                .lowCardinalityKeyValue("status", "500");

        assertThat(observation.getContext().getLowCardinalityKeyValues())
                .containsEntry("status", "500")
                .hasSize(1);
    }

    @Test
    void shouldCallMultipleHandlersInCorrectOrder() {
        List<String> events = new ArrayList<>();

        registry.observationConfig()
                .observationHandler(new ObservationHandler<>() {
                    @Override public void onStart(Observation.Context c) { events.add("A:start"); }
                    @Override public void onStop(Observation.Context c) { events.add("A:stop"); }
                    @Override public void onScopeOpened(Observation.Context c) { events.add("A:scopeOpen"); }
                    @Override public void onScopeClosed(Observation.Context c) { events.add("A:scopeClose"); }
                    @Override public boolean supportsContext(Observation.Context c) { return true; }
                })
                .observationHandler(new ObservationHandler<>() {
                    @Override public void onStart(Observation.Context c) { events.add("B:start"); }
                    @Override public void onStop(Observation.Context c) { events.add("B:stop"); }
                    @Override public void onScopeOpened(Observation.Context c) { events.add("B:scopeOpen"); }
                    @Override public void onScopeClosed(Observation.Context c) { events.add("B:scopeClose"); }
                    @Override public boolean supportsContext(Observation.Context c) { return true; }
                });

        Observation.createNotStarted("test", registry).observe(() -> {});

        assertThat(events).containsExactly(
                "A:start", "B:start",
                "A:scopeOpen", "B:scopeOpen",
                "B:scopeClose", "A:scopeClose",
                "B:stop", "A:stop");
    }

    @Test
    void shouldOnlyCallHandlersThatSupportTheContext() {
        List<String> events = new ArrayList<>();

        registry.observationConfig()
                .observationHandler(new ObservationHandler<>() {
                    @Override public void onStart(Observation.Context c) { events.add("generic:start"); }
                    @Override public boolean supportsContext(Observation.Context c) { return true; }
                })
                .observationHandler(new ObservationHandler<>() {
                    @Override public void onStart(Observation.Context c) { events.add("http:start"); }
                    @Override public boolean supportsContext(Observation.Context c) { return c.getName().startsWith("http"); }
                });

        Observation.createNotStarted("http.request", registry).observe(() -> {});
        assertThat(events).containsExactly("generic:start", "http:start");

        events.clear();
        Observation.createNotStarted("db.query", registry).observe(() -> {});
        assertThat(events).containsExactly("generic:start");
    }

    @Test
    void shouldReturnNoopWhenRegistryIsNull() {
        assertThat(Observation.createNotStarted("test", null)).isSameAs(Observation.NOOP);
    }

    @Test
    void shouldReturnNoopWhenRegistryHasNoHandlers() {
        ObservationRegistry emptyRegistry = ObservationRegistry.create();
        assertThat(Observation.createNotStarted("test", emptyRegistry)).isSameAs(Observation.NOOP);
    }

    @Test
    void shouldReturnNoopForNoopRegistry() {
        assertThat(Observation.createNotStarted("test", ObservationRegistry.NOOP)).isSameAs(Observation.NOOP);
    }

    @Test
    void noopObservationShouldStillExecuteCode() {
        boolean[] ran = {false};
        Observation.NOOP.observe(() -> ran[0] = true);
        assertThat(ran[0]).isTrue();
    }

    @Test
    void noopObservationShouldReturnSupplierResult() {
        assertThat(Observation.NOOP.<String>observe(() -> "from noop")).isEqualTo("from noop");
    }

    @Test
    void shouldMakeObservationCurrentWhileScopeIsOpen() {
        registry.observationConfig().observationHandler(new ObservationHandler<>() {
            @Override public boolean supportsContext(Observation.Context c) { return true; }
        });

        assertThat(registry.getCurrentObservation()).isNull();

        Observation obs = Observation.start("test", registry);
        try (Observation.Scope scope = obs.openScope()) {
            assertThat(registry.getCurrentObservation()).isSameAs(obs);
        } finally {
            obs.stop();
        }

        assertThat(registry.getCurrentObservation()).isNull();
    }

    @Test
    void shouldAutoSetParentFromCurrentScope() {
        registry.observationConfig().observationHandler(new ObservationHandler<>() {
            @Override public boolean supportsContext(Observation.Context c) { return true; }
        });

        Observation parent = Observation.start("parent", registry);
        try (Observation.Scope parentScope = parent.openScope()) {
            Observation child = Observation.createNotStarted("child", registry);
            assertThat(child.getContext().getParentObservation()).isSameAs(parent);
        } finally {
            parent.stop();
        }
    }

    @Test
    void shouldRestorePreviousScope_WhenScopeCloses() {
        registry.observationConfig().observationHandler(new ObservationHandler<>() {
            @Override public boolean supportsContext(Observation.Context c) { return true; }
        });

        assertThat(registry.getCurrentObservation()).isNull();

        Observation outer = Observation.start("outer", registry);
        try (Observation.Scope outerScope = outer.openScope()) {
            assertThat(registry.getCurrentObservation()).isSameAs(outer);

            Observation inner = Observation.start("inner", registry);
            try (Observation.Scope innerScope = inner.openScope()) {
                assertThat(registry.getCurrentObservation()).isSameAs(inner);
            } finally {
                inner.stop();
            }

            assertThat(registry.getCurrentObservation()).isSameAs(outer);
        } finally {
            outer.stop();
        }

        assertThat(registry.getCurrentObservation()).isNull();
    }

    @Test
    void shouldStoreAndRetrieveArbitraryStateInContext() {
        Observation.Context context = new Observation.Context();
        context.put("startTime", 12345L);
        context.put(String.class, "typed-key");

        assertThat(context.<Long>get("startTime")).isEqualTo(12345L);
        assertThat(context.<String>get(String.class)).isEqualTo("typed-key");
        assertThat(context.containsKey("startTime")).isTrue();
        assertThat(context.containsKey("missing")).isFalse();
    }

    @Test
    void shouldCreateTimerMetricsFromObservationViaHandler() {
        MeterRegistry meterRegistry = new SimpleMeterRegistry(Clock.SYSTEM);
        registry.observationConfig().observationHandler(new TimerObservationHandler(meterRegistry));

        Observation.createNotStarted("http.request", registry)
                .lowCardinalityKeyValue("method", "GET")
                .observe(() -> {
                    long start = System.nanoTime();
                    while (System.nanoTime() - start < 1_000_000) { }
                });

        Timer timer = meterRegistry.find("http.request").tag("method", "GET").timer();
        assertThat(timer).isNotNull();
        assertThat(timer.count()).isEqualTo(1);
        assertThat(timer.totalTime(TimeUnit.MILLISECONDS)).isGreaterThan(0);
    }

    @Test
    void shouldRecordMultipleObservationsToSameTimer() {
        MeterRegistry meterRegistry = new SimpleMeterRegistry(Clock.SYSTEM);
        registry.observationConfig().observationHandler(new TimerObservationHandler(meterRegistry));

        for (int i = 0; i < 3; i++) {
            Observation.createNotStarted("db.query", registry)
                    .lowCardinalityKeyValue("db", "postgres")
                    .observe(() -> {});
        }

        Timer timer = meterRegistry.find("db.query").tag("db", "postgres").timer();
        assertThat(timer).isNotNull();
        assertThat(timer.count()).isEqualTo(3);
    }

    // ── Helper ───────────────────────────────────────────────────────

    static class TimerObservationHandler implements ObservationHandler<Observation.Context> {
        private final MeterRegistry meterRegistry;

        TimerObservationHandler(MeterRegistry meterRegistry) {
            this.meterRegistry = meterRegistry;
        }

        @Override
        public void onStart(Observation.Context context) {
            context.put(Timer.Sample.class, Timer.start(meterRegistry));
        }

        @Override
        public void onStop(Observation.Context context) {
            Timer.Sample sample = context.get(Timer.Sample.class);
            if (sample != null) {
                Tags tags = Tags.empty();
                for (var entry : context.getLowCardinalityKeyValues().entrySet()) {
                    tags = tags.and(entry.getKey(), entry.getValue());
                }
                sample.stop(Timer.builder(context.getName()).tags(tags).register(meterRegistry));
            }
        }

        @Override
        public boolean supportsContext(Observation.Context context) {
            return true;
        }
    }

}
```
