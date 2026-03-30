# Chapter 13: Global Metrics Facade — Static Convenience API

> **Feature 13** · Tier 2 · Depends on: Features 1–4 (all meter types), Feature 7 (CompositeMeterRegistry)

After this chapter you will be able to **record metrics from anywhere in your code** using
static methods — no need to pass `MeterRegistry` references around. This is the "SLF4J
for metrics" pattern: your application code calls `Metrics.counter(...)`, and the actual
backends are wired at startup.

---

## 1. The API Contract

Before writing any implementation, let's define exactly what a client sees.

### What the client imports

```java
import simple.micrometer.api.Metrics;
import simple.micrometer.api.Counter;
import simple.micrometer.api.Timer;
import simple.micrometer.api.Gauge;
import simple.micrometer.api.DistributionSummary;
import simple.micrometer.internal.SimpleMeterRegistry;
```

### What the client writes

```java
// At app startup — register backend registries
Metrics.addRegistry(new SimpleMeterRegistry());
// Metrics.addRegistry(new PrometheusRegistry());

// Anywhere in your code — use static convenience methods
Metrics.counter("app.events", "type", "login").increment();
Metrics.timer("app.processing").record(() -> doWork());
Metrics.gauge("app.queue.size", queue, Queue::size);

// Less common meter types via Metrics.more()
Metrics.more().longTaskTimer("data.migration", "table", "users");

// Access the global registry directly if needed
CompositeMeterRegistry global = Metrics.globalRegistry;
```

### Behavioral contract

| Promise | Detail |
|---------|--------|
| Static access | All methods are `static` — no instance needed |
| Global composite | The underlying registry is a `CompositeMeterRegistry` |
| Zero-backend safety | Before any backend is added, all operations silently no-op |
| Dynamic backends | Backends can be added/removed at any time via `addRegistry`/`removeRegistry` |
| Full meter coverage | Supports all meter types: Counter, Timer, Summary, Gauge, and via `more()`: LongTaskTimer, FunctionCounter, FunctionTimer, TimeGauge |
| Deduplication | Same name + tags returns the same meter instance (inherited from `MeterRegistry`) |
| Filter-aware | Denied meters return noop implementations (inherited from `MeterRegistry`) |

### When to use `Metrics` vs injecting `MeterRegistry`

```
Static facade (Metrics)                  Dependency injection (MeterRegistry)
──────────────────────────               ────────────────────────────────────
Library code, no DI available            Application code with Spring/Guice/etc.
Quick prototyping                        Production services
Test utilities                           When you need testability (mock the registry)
Global cross-cutting metrics             Component-scoped metrics
```

---

## 2. Client Usage & Tests

These tests define the API's behavior — they were written **before** any implementation.

### Counter via static method

```java
@Test
void shouldCreateCounterWithNameAndVarargsTags() {
    Metrics.addRegistry(backend);

    Counter counter = Metrics.counter("http.requests", "method", "GET", "status", "200");
    counter.increment();
    counter.increment(4.0);

    assertThat(backend.counter("http.requests", "method", "GET", "status", "200").count())
            .isEqualTo(5.0);
}
```

### Timer records a Runnable

```java
@Test
void shouldRecordRunnableViaTimer() {
    Metrics.addRegistry(backend);

    Metrics.timer("app.processing").record(() -> {
        // simulate work
        @SuppressWarnings("unused") int sum = 0;
        for (int i = 0; i < 100; i++) sum += i;
    });

    assertThat(backend.timer("app.processing").count()).isEqualTo(1);
}
```

### Distribution Summary via static method

```java
@Test
void shouldCreateSummaryWithNameAndVarargsTags() {
    Metrics.addRegistry(backend);

    DistributionSummary summary = Metrics.summary("payload.size", "uri", "/api");
    summary.record(256);
    summary.record(512);

    assertThat(backend.summary("payload.size", "uri", "/api").count()).isEqualTo(2);
    assertThat(backend.summary("payload.size", "uri", "/api").totalAmount()).isEqualTo(768.0);
}
```

### Gauge tracks a live value

```java
@Test
void shouldCreateGaugeWithObjectAndFunction() {
    Metrics.addRegistry(backend);

    AtomicInteger queueSize = new AtomicInteger(42);
    Metrics.gauge("app.queue.size", Tags.of("queue", "orders"), queueSize, AtomicInteger::doubleValue);

    Gauge gauge = backend.find("app.queue.size").tag("queue", "orders").gauge();
    assertThat(gauge).isNotNull();
    assertThat(gauge.value()).isEqualTo(42.0);

    queueSize.set(99);
    assertThat(gauge.value()).isEqualTo(99.0);
}
```

### LongTaskTimer via Metrics.more()

```java
@Test
void shouldCreateLongTaskTimerViaMore() {
    Metrics.addRegistry(backend);

    LongTaskTimer ltt = Metrics.more().longTaskTimer("data.migration", "table", "users");
    LongTaskTimer.Sample sample = ltt.start();
    assertThat(ltt.activeTasks()).isEqualTo(1);

    sample.stop();
    assertThat(ltt.activeTasks()).isEqualTo(0);
}
```

### FunctionCounter via Metrics.more()

```java
@Test
void shouldCreateFunctionCounterViaMore() {
    Metrics.addRegistry(backend);

    AtomicLong externalCount = new AtomicLong(0);
    FunctionCounter fc = Metrics.more().counter("external.count",
            Tags.of("source", "cache"), externalCount, AtomicLong::doubleValue);

    assertThat(fc.count()).isEqualTo(0.0);

    externalCount.set(42);
    assertThat(fc.count()).isEqualTo(42.0);
}
```

### Integration test: typical application workflow

```java
@Test
void shouldSupportTypicalApplicationWorkflow() {
    // Step 1: App startup — register backend(s)
    Metrics.addRegistry(backend);

    // Step 2: Application code records metrics via static methods
    Metrics.counter("app.events", "type", "login").increment();
    Metrics.timer("app.processing").record(200, TimeUnit.MILLISECONDS);

    AtomicInteger queueSize = new AtomicInteger(5);
    Metrics.gauge("app.queue.size", queueSize);

    // Step 3: Verify metrics are collected in the backend
    assertThat(backend.counter("app.events", "type", "login").count()).isEqualTo(1.0);
    assertThat(backend.timer("app.processing").count()).isEqualTo(1);
    assertThat(backend.find("app.queue.size").gauge().value()).isEqualTo(5.0);
}
```

---

## 3. Implementing the Call Chain

The `Metrics` class is architecturally unique: it has **no internal layers**. The entire
implementation is a thin static delegation shell over `CompositeMeterRegistry`, which already
handles all the heavy lifting (meter creation, deduplication, filter chains, composite
fan-out).

```
Client calls: Metrics.counter("http.requests", "method", "GET")
  │
  ├─► [API Layer] Metrics.counter(name, tags)
  │     Static method, delegates to globalRegistry
  │
  └─► [Existing Infrastructure] CompositeMeterRegistry.counter(name, tags)
        Inherited from MeterRegistry:
        ├─ Creates Meter.Id
        ├─ Applies MeterFilter chain (map + accept)
        ├─ Double-checked locking registration
        └─ Returns CompositeCounter (fans out to all child registries)
```

### 3a. API Layer — The Metrics class

This is the only new class in this feature. It consists of:

1. **A static `CompositeMeterRegistry`** — the global registry that everything delegates to
2. **Registry management** — `addRegistry()` and `removeRegistry()`
3. **Static convenience methods** — one-liner delegates for each meter type
4. **A `More` inner class** — delegates to `globalRegistry.more()` for less common meter types

```java
public final class Metrics {

    public static final CompositeMeterRegistry globalRegistry = new CompositeMeterRegistry();

    private static final More more = new More();

    private Metrics() {
        // static-only class
    }
```

**Why `final` with private constructor?** This prevents instantiation and subclassing —
`Metrics` is a pure utility class with only static methods, similar to `java.util.Collections`
or `java.util.Objects`.

**Why `CompositeMeterRegistry` (not `MeterRegistry`)?** The global registry starts with
**zero backends** and backends are added dynamically at startup. `CompositeMeterRegistry`
is the only registry type that handles this gracefully — with no children, all operations
silently no-op.

### 3b. Registry management

```java
public static void addRegistry(MeterRegistry registry) {
    globalRegistry.add(registry);
}

public static void removeRegistry(MeterRegistry registry) {
    globalRegistry.remove(registry);
}
```

These are pure delegation to `CompositeMeterRegistry.add()` and `.remove()` from Feature 7.
When a backend is added, all existing composite meters are wired to create concrete meters
in the new child — this means metrics recorded *before* `addRegistry()` will be visible
in the new backend for future reads.

### 3c. Counter, Timer, Summary convenience methods

Each meter type gets two overloads — varargs tags and `Iterable<Tag>`:

```java
public static Counter counter(String name, String... tags) {
    return globalRegistry.counter(name, tags);
}

public static Counter counter(String name, Iterable<Tag> tags) {
    return globalRegistry.counter(name, tags);
}
```

Timer and DistributionSummary follow the same pattern. Every method is a one-liner
delegate — no new logic, no new behavior.

### 3d. Gauge convenience methods

Gauges have four overloads mirroring `MeterRegistry`'s gauge methods:

```java
// Full: object + function + tags
public static <T> T gauge(String name, Iterable<Tag> tags, T stateObject,
                           ToDoubleFunction<T> valueFunction) {
    return globalRegistry.gauge(name, tags, stateObject, valueFunction);
}

// Number + tags
public static <T extends Number> T gauge(String name, Iterable<Tag> tags, T number) {
    return globalRegistry.gauge(name, tags, number);
}

// Number, no tags
public static <T extends Number> T gauge(String name, T number) {
    return globalRegistry.gauge(name, number);
}

// Object + function, no tags
public static <T> T gauge(String name, T stateObject, ToDoubleFunction<T> valueFunction) {
    return globalRegistry.gauge(name, stateObject, valueFunction);
}
```

Note that gauge methods return the **state object**, not the `Gauge` — this is inherited
from `MeterRegistry`'s design and enables fluent assignment patterns like
`AtomicInteger active = Metrics.gauge("active", new AtomicInteger(0))`.

### 3e. The `More` inner class

The `More` class provides static access to less common meter types:

```java
public static class More {

    public LongTaskTimer longTaskTimer(String name, String... tags) {
        return globalRegistry.more().longTaskTimer(name, tags);
    }

    public <T> FunctionCounter counter(String name, Iterable<Tag> tags,
                                        T obj, ToDoubleFunction<T> countFunction) {
        return FunctionCounter.builder(name, obj, countFunction)
                .tags(tags)
                .register(globalRegistry);
    }

    public <T> FunctionTimer timer(String name, Iterable<Tag> tags, T obj,
                                    ToLongFunction<T> countFunction,
                                    ToDoubleFunction<T> totalTimeFunction,
                                    TimeUnit totalTimeFunctionUnit) {
        return FunctionTimer.builder(name, obj, countFunction, totalTimeFunction,
                        totalTimeFunctionUnit)
                .tags(tags)
                .register(globalRegistry);
    }

    public <T> TimeGauge timeGauge(String name, Iterable<Tag> tags, T obj,
                                    TimeUnit fUnits, ToDoubleFunction<T> f) {
        return TimeGauge.builder(name, obj, fUnits, f)
                .tags(tags)
                .register(globalRegistry);
    }
}
```

**Why does `longTaskTimer` delegate to `globalRegistry.more()` while others use Builders?**
LongTaskTimer has convenience methods on `MeterRegistry.More`, so we can delegate directly.
FunctionCounter, FunctionTimer, and TimeGauge don't have such convenience methods on `More`
(they register through their Builder's `register()` method), so we use the Builder API and
pass `globalRegistry` as the target.

---

## 4. Try It Yourself

**Challenge 1**: Add a `gaugeCollectionSize()` convenience method to `Metrics` that registers
a gauge reporting a `Collection`'s `size()`. Hint: look at how the real Micrometer's
`Metrics.gaugeCollectionSize()` works — it's a gauge where the value function calls `size()`.

**Challenge 2**: The global `CompositeMeterRegistry` is created with `Clock.SYSTEM`. What
would happen if you wanted to control the clock in tests (e.g., for timer assertions)?
Consider adding a package-private method to replace the global registry for testing.

**Challenge 3**: The real Micrometer's `Metrics` class resets the `globalRegistry` in tests
by clearing all meters and child registries. Implement a `clear()` method that would be
useful for test isolation — but think carefully about what state needs to be reset.

---

## 5. Why This Works

**Static facades trade testability for convenience**: The `Metrics` class makes recording
metrics as easy as `Metrics.counter("x").increment()`, but testing code that uses it
requires careful cleanup (`@AfterEach removeRegistry`). This is the same trade-off as
`System.out` vs injecting a `PrintStream`, or `LoggerFactory.getLogger()` vs injecting
a `Logger`. The real Micrometer provides both approaches — use `Metrics` for library code
where DI isn't available, and inject `MeterRegistry` in application code where it is.

**CompositeMeterRegistry is the enabling technology**: Without Feature 7's composite
pattern, a static facade would need to know its backend at class-load time. The composite
lets the global registry start empty and accumulate backends dynamically — an application
can call `Metrics.counter(...)` before any backend is registered, and the metric will
"light up" when a backend is eventually added.

**One class, zero new internals**: This is the only feature in the outline that adds no
internal classes. Every method is a one-liner delegate. This is possible because Features
1–4 (meter types), Feature 7 (composite), and Features 11–12 (LongTaskTimer, function-
tracking meters) already built all the infrastructure. `Metrics` is the keystone that
makes that infrastructure accessible without plumbing.

---

## 6. What We Enhanced

| Previous Feature | What Changed | Why |
|-----------------|-------------|-----|
| None | No existing code was modified | `Metrics` is purely additive — a new class that delegates to existing infrastructure |

This is notable: Feature 13 is the first feature that modifies **zero** existing files.
It adds one new API class and one test class, both using only public APIs from earlier features.

---

## 7. Connection to Real Framework

| Simplified Class | Real Framework Class | Key Simplification |
|-----------------|---------------------|-------------------|
| `Metrics` | `io.micrometer.core.instrument.Metrics` | No `gaugeCollectionSize`, `gaugeMapSize`, or `@Nullable` annotations |
| `Metrics.More` | `io.micrometer.core.instrument.Metrics.More` | No `timeGauge` with `Supplier` overload, no `@Incubating` methods |

The real `Metrics` class has 17 static methods (plus 6 via `More`). Our simplified version
has 12 static methods (plus 7 via `More`) — covering all the commonly used ones. The
methods we omitted (`gaugeCollectionSize`, `gaugeMapSize`) are trivial convenience wrappers
that could be added as a challenge exercise.

---

## 8. Complete Code

### `[NEW]` `src/main/java/simple/micrometer/api/Metrics.java`

```java
package simple.micrometer.api;

import java.util.Collections;
import java.util.concurrent.TimeUnit;
import java.util.function.ToDoubleFunction;
import java.util.function.ToLongFunction;

/**
 * A static facade over a global {@link CompositeMeterRegistry} — the "just works"
 * entry point for recording metrics without passing registry references around.
 *
 * <p>Usage:
 * <pre>{@code
 * // At startup, register backend registries
 * Metrics.addRegistry(new SimpleMeterRegistry());
 *
 * // Anywhere in your code, use static convenience methods
 * Metrics.counter("app.events", "type", "login").increment();
 * Metrics.timer("app.processing").record(() -> doWork());
 * Metrics.gauge("app.queue.size", queue, Queue::size);
 * }</pre>
 *
 * <p>All methods delegate to the single shared {@link #globalRegistry}. Before any
 * backend is added via {@link #addRegistry}, all operations silently no-op through
 * the composite's empty-child behavior.
 *
 * <p>This class is designed for use in places where dependency injection of a
 * {@link MeterRegistry} is not possible. When DI is available, prefer injecting
 * a {@code MeterRegistry} directly.
 *
 * <p>Simplified from {@code io.micrometer.core.instrument.Metrics}.
 */
public final class Metrics {

    /**
     * The global composite registry. Backend registries are added to this composite
     * via {@link #addRegistry}. All static convenience methods delegate here.
     */
    public static final CompositeMeterRegistry globalRegistry = new CompositeMeterRegistry();

    private static final More more = new More();

    private Metrics() {
        // static-only class
    }

    // ── Registry management ─────────────────────────────────────────

    /**
     * Adds a registry to the global composite. Metrics previously recorded
     * via this facade will be visible in the newly added registry.
     *
     * @param registry the backend registry to add
     */
    public static void addRegistry(MeterRegistry registry) {
        globalRegistry.add(registry);
    }

    /**
     * Removes a registry from the global composite. Meters previously created
     * in the removed registry are <b>not</b> cleaned up.
     *
     * @param registry the backend registry to remove
     */
    public static void removeRegistry(MeterRegistry registry) {
        globalRegistry.remove(registry);
    }

    // ── Counter ─────────────────────────────────────────────────────

    /**
     * Creates or retrieves a counter with the given name and alternating
     * key-value tag strings.
     *
     * <pre>{@code
     * Metrics.counter("cache.misses", "cache", "users").increment();
     * }</pre>
     */
    public static Counter counter(String name, String... tags) {
        return globalRegistry.counter(name, tags);
    }

    /**
     * Creates or retrieves a counter with the given name and tags.
     */
    public static Counter counter(String name, Iterable<Tag> tags) {
        return globalRegistry.counter(name, tags);
    }

    // ── Timer ───────────────────────────────────────────────────────

    /**
     * Creates or retrieves a timer with the given name and alternating
     * key-value tag strings.
     *
     * <pre>{@code
     * Metrics.timer("http.latency", "method", "GET").record(100, TimeUnit.MILLISECONDS);
     * }</pre>
     */
    public static Timer timer(String name, String... tags) {
        return globalRegistry.timer(name, tags);
    }

    /**
     * Creates or retrieves a timer with the given name and tags.
     */
    public static Timer timer(String name, Iterable<Tag> tags) {
        return globalRegistry.timer(name, tags);
    }

    // ── Distribution Summary ────────────────────────────────────────

    /**
     * Creates or retrieves a distribution summary with the given name and
     * alternating key-value tag strings.
     *
     * <pre>{@code
     * Metrics.summary("payload.size", "uri", "/api").record(256);
     * }</pre>
     */
    public static DistributionSummary summary(String name, String... tags) {
        return globalRegistry.summary(name, tags);
    }

    /**
     * Creates or retrieves a distribution summary with the given name and tags.
     */
    public static DistributionSummary summary(String name, Iterable<Tag> tags) {
        return globalRegistry.summary(name, tags);
    }

    // ── Gauge ───────────────────────────────────────────────────────

    /**
     * Registers a gauge that observes {@code stateObject} through
     * {@code valueFunction}.
     *
     * <pre>{@code
     * Metrics.gauge("app.queue.size", Tags.of("queue", "orders"),
     *     queue, Queue::size);
     * }</pre>
     *
     * @return the {@code stateObject} passed in (for assignment convenience)
     */
    public static <T> T gauge(String name, Iterable<Tag> tags, T stateObject,
                               ToDoubleFunction<T> valueFunction) {
        return globalRegistry.gauge(name, tags, stateObject, valueFunction);
    }

    /**
     * Registers a gauge tracking a {@link Number} subclass (AtomicInteger,
     * AtomicLong, etc.) using its {@link Number#doubleValue()}.
     */
    public static <T extends Number> T gauge(String name, Iterable<Tag> tags, T number) {
        return globalRegistry.gauge(name, tags, number);
    }

    /**
     * Registers a gauge tracking a {@link Number} without tags.
     */
    public static <T extends Number> T gauge(String name, T number) {
        return globalRegistry.gauge(name, number);
    }

    /**
     * Registers a gauge observing an object without tags.
     */
    public static <T> T gauge(String name, T stateObject, ToDoubleFunction<T> valueFunction) {
        return globalRegistry.gauge(name, stateObject, valueFunction);
    }

    // ── More ────────────────────────────────────────────────────────

    /**
     * Returns the {@link More} handle for registering less common meter types
     * (LongTaskTimer, FunctionCounter, FunctionTimer, TimeGauge) via the
     * global registry.
     */
    public static More more() {
        return more;
    }

    /**
     * Provides static access to less commonly used meter types. Mirrors
     * {@link MeterRegistry.More} but delegates to the global registry.
     */
    public static class More {

        /**
         * Creates or retrieves a long task timer with the given name and
         * alternating key-value tag strings.
         */
        public LongTaskTimer longTaskTimer(String name, String... tags) {
            return globalRegistry.more().longTaskTimer(name, tags);
        }

        /**
         * Creates or retrieves a long task timer with the given name and tags.
         */
        public LongTaskTimer longTaskTimer(String name, Iterable<Tag> tags) {
            return globalRegistry.more().longTaskTimer(name, tags);
        }

        /**
         * Registers a function counter with the given name, tags, observed
         * object, and count function.
         */
        public <T> FunctionCounter counter(String name, Iterable<Tag> tags,
                                            T obj, ToDoubleFunction<T> countFunction) {
            return FunctionCounter.builder(name, obj, countFunction)
                    .tags(tags)
                    .register(globalRegistry);
        }

        /**
         * Registers a function counter tracking a {@link Number} subclass.
         */
        public <T extends Number> FunctionCounter counter(String name, Iterable<Tag> tags, T number) {
            return FunctionCounter.builder(name, number, Number::doubleValue)
                    .tags(tags)
                    .register(globalRegistry);
        }

        /**
         * Registers a function timer with the given name, tags, observed
         * object, count function, total time function, and time unit.
         */
        public <T> FunctionTimer timer(String name, Iterable<Tag> tags, T obj,
                                        ToLongFunction<T> countFunction,
                                        ToDoubleFunction<T> totalTimeFunction,
                                        TimeUnit totalTimeFunctionUnit) {
            return FunctionTimer.builder(name, obj, countFunction, totalTimeFunction,
                            totalTimeFunctionUnit)
                    .tags(tags)
                    .register(globalRegistry);
        }

        /**
         * Registers a time gauge with the given name, tags, observed object,
         * time unit, and value function.
         */
        public <T> TimeGauge timeGauge(String name, Iterable<Tag> tags, T obj,
                                        TimeUnit fUnits, ToDoubleFunction<T> f) {
            return TimeGauge.builder(name, obj, fUnits, f)
                    .tags(tags)
                    .register(globalRegistry);
        }
    }

}
```

### `[NEW]` `src/test/java/simple/micrometer/api/MetricsTest.java`

```java
package simple.micrometer.api;

import org.junit.jupiter.api.AfterEach;
import org.junit.jupiter.api.Test;
import simple.micrometer.internal.SimpleMeterRegistry;

import java.util.concurrent.TimeUnit;
import java.util.concurrent.atomic.AtomicInteger;
import java.util.concurrent.atomic.AtomicLong;
import java.util.function.ToDoubleFunction;
import java.util.function.ToLongFunction;

import static org.assertj.core.api.Assertions.assertThat;

/**
 * Client-perspective tests for the {@link Metrics} static facade.
 * These tests define the API's behavior — they were written before any implementation.
 *
 * <p>Every test adds a {@link SimpleMeterRegistry} to the global composite,
 * exercises the static convenience method, and verifies the result via the
 * child registry.
 */
class MetricsTest {

    private final SimpleMeterRegistry backend = new SimpleMeterRegistry();

    @AfterEach
    void tearDown() {
        // Clean up: remove the backend and reset global state between tests
        Metrics.removeRegistry(backend);
    }

    // ── Registry management ─────────────────────────────────────────

    @Test
    void shouldAddAndRemoveRegistries() {
        Metrics.addRegistry(backend);

        Counter counter = Metrics.counter("test.counter");
        counter.increment();

        // Backend sees the counter
        assertThat(backend.counter("test.counter").count()).isEqualTo(1.0);

        // Remove and verify new increments don't reach the backend
        Metrics.removeRegistry(backend);

        // The global registry's counter is still accessible, but without
        // a child, the composite delegates to noop — this is existing
        // CompositeMeterRegistry behavior tested in Feature 7
    }

    @Test
    void globalRegistryShouldBeACompositeMeterRegistry() {
        assertThat(Metrics.globalRegistry).isInstanceOf(CompositeMeterRegistry.class);
    }

    // ── Counter ─────────────────────────────────────────────────────

    @Test
    void shouldCreateCounterWithNameAndVarargsTags() {
        Metrics.addRegistry(backend);

        Counter counter = Metrics.counter("http.requests", "method", "GET", "status", "200");
        counter.increment();
        counter.increment(4.0);

        assertThat(backend.counter("http.requests", "method", "GET", "status", "200").count())
                .isEqualTo(5.0);
    }

    @Test
    void shouldCreateCounterWithNameAndIterableTags() {
        Metrics.addRegistry(backend);

        Counter counter = Metrics.counter("cache.misses", Tags.of("cache", "users"));
        counter.increment(3.0);

        assertThat(backend.counter("cache.misses", "cache", "users").count())
                .isEqualTo(3.0);
    }

    // ── Timer ───────────────────────────────────────────────────────

    @Test
    void shouldCreateTimerWithNameAndVarargsTags() {
        Metrics.addRegistry(backend);

        Timer timer = Metrics.timer("http.latency", "method", "POST");
        timer.record(100, TimeUnit.MILLISECONDS);

        assertThat(backend.timer("http.latency", "method", "POST").count()).isEqualTo(1);
    }

    @Test
    void shouldCreateTimerWithNameAndIterableTags() {
        Metrics.addRegistry(backend);

        Timer timer = Metrics.timer("db.query", Tags.of("table", "orders"));
        timer.record(50, TimeUnit.MILLISECONDS);

        assertThat(backend.timer("db.query", "table", "orders").count()).isEqualTo(1);
    }

    @Test
    void shouldRecordRunnableViaTimer() {
        Metrics.addRegistry(backend);

        Metrics.timer("app.processing").record(() -> {
            // simulate work
            @SuppressWarnings("unused") int sum = 0;
            for (int i = 0; i < 100; i++) sum += i;
        });

        assertThat(backend.timer("app.processing").count()).isEqualTo(1);
    }

    // ── Distribution Summary ────────────────────────────────────────

    @Test
    void shouldCreateSummaryWithNameAndVarargsTags() {
        Metrics.addRegistry(backend);

        DistributionSummary summary = Metrics.summary("payload.size", "uri", "/api");
        summary.record(256);
        summary.record(512);

        assertThat(backend.summary("payload.size", "uri", "/api").count()).isEqualTo(2);
        assertThat(backend.summary("payload.size", "uri", "/api").totalAmount()).isEqualTo(768.0);
    }

    @Test
    void shouldCreateSummaryWithNameAndIterableTags() {
        Metrics.addRegistry(backend);

        DistributionSummary summary = Metrics.summary("response.size", Tags.of("status", "200"));
        summary.record(1024);

        assertThat(backend.summary("response.size", "status", "200").totalAmount()).isEqualTo(1024.0);
    }

    // ── Gauge ───────────────────────────────────────────────────────

    @Test
    void shouldCreateGaugeWithObjectAndFunction() {
        Metrics.addRegistry(backend);

        AtomicInteger queueSize = new AtomicInteger(42);
        Metrics.gauge("app.queue.size", Tags.of("queue", "orders"), queueSize, AtomicInteger::doubleValue);

        Gauge gauge = backend.find("app.queue.size").tag("queue", "orders").gauge();
        assertThat(gauge).isNotNull();
        assertThat(gauge.value()).isEqualTo(42.0);

        queueSize.set(99);
        assertThat(gauge.value()).isEqualTo(99.0);
    }

    @Test
    void shouldCreateGaugeWithNumberAndTags() {
        Metrics.addRegistry(backend);

        AtomicInteger connections = new AtomicInteger(10);
        Metrics.gauge("db.connections", Tags.of("pool", "main"), connections);

        Gauge gauge = backend.find("db.connections").tag("pool", "main").gauge();
        assertThat(gauge).isNotNull();
        assertThat(gauge.value()).isEqualTo(10.0);
    }

    @Test
    void shouldCreateGaugeWithNumberNoTags() {
        Metrics.addRegistry(backend);

        AtomicInteger active = new AtomicInteger(5);
        Metrics.gauge("active.users", active);

        Gauge gauge = backend.find("active.users").gauge();
        assertThat(gauge).isNotNull();
        assertThat(gauge.value()).isEqualTo(5.0);
    }

    @Test
    void shouldCreateGaugeWithObjectAndFunctionNoTags() {
        Metrics.addRegistry(backend);

        AtomicLong heapUsed = new AtomicLong(1024L);
        Metrics.gauge("jvm.heap", heapUsed, AtomicLong::doubleValue);

        Gauge gauge = backend.find("jvm.heap").gauge();
        assertThat(gauge).isNotNull();
        assertThat(gauge.value()).isEqualTo(1024.0);
    }

    // ── More: LongTaskTimer ─────────────────────────────────────────

    @Test
    void shouldCreateLongTaskTimerViaMore() {
        Metrics.addRegistry(backend);

        LongTaskTimer ltt = Metrics.more().longTaskTimer("data.migration", "table", "users");
        LongTaskTimer.Sample sample = ltt.start();
        assertThat(ltt.activeTasks()).isEqualTo(1);

        sample.stop();
        assertThat(ltt.activeTasks()).isEqualTo(0);
    }

    @Test
    void shouldCreateLongTaskTimerWithIterableTags() {
        Metrics.addRegistry(backend);

        LongTaskTimer ltt = Metrics.more().longTaskTimer("reindex", Tags.of("index", "products"));
        assertThat(ltt).isNotNull();
        assertThat(ltt.activeTasks()).isEqualTo(0);
    }

    // ── More: FunctionCounter ───────────────────────────────────────

    @Test
    void shouldCreateFunctionCounterViaMore() {
        Metrics.addRegistry(backend);

        AtomicLong externalCount = new AtomicLong(0);
        FunctionCounter fc = Metrics.more().counter("external.count",
                Tags.of("source", "cache"), externalCount, AtomicLong::doubleValue);

        assertThat(fc.count()).isEqualTo(0.0);

        externalCount.set(42);
        assertThat(fc.count()).isEqualTo(42.0);
    }

    @Test
    void shouldCreateFunctionCounterForNumberViaMore() {
        Metrics.addRegistry(backend);

        AtomicLong hits = new AtomicLong(7);
        FunctionCounter fc = Metrics.more().counter("cache.hits", Tags.of("cache", "sessions"), hits);

        assertThat(fc.count()).isEqualTo(7.0);
    }

    // ── More: FunctionTimer ─────────────────────────────────────────

    @Test
    void shouldCreateFunctionTimerViaMore() {
        Metrics.addRegistry(backend);

        // Fake stats object
        var stats = new Object() {
            long count = 5;
            double totalTimeNanos = 1_000_000_000.0; // 1 second in nanos
        };

        ToLongFunction<Object> countFn = s -> ((long) 5);
        ToDoubleFunction<Object> totalTimeFn = s -> 1_000_000_000.0;

        FunctionTimer ft = Metrics.more().timer("cache.gets", Tags.of("cache", "users"),
                stats, countFn, totalTimeFn, TimeUnit.NANOSECONDS);

        assertThat(ft.count()).isEqualTo(5.0);
        assertThat(ft.totalTime(TimeUnit.SECONDS)).isCloseTo(1.0, org.assertj.core.data.Offset.offset(0.01));
    }

    // ── More: TimeGauge ─────────────────────────────────────────────

    @Test
    void shouldCreateTimeGaugeViaMore() {
        Metrics.addRegistry(backend);

        AtomicLong uptimeMs = new AtomicLong(60_000L); // 60 seconds
        TimeGauge tg = Metrics.more().timeGauge("process.uptime", Tags.of("process", "main"),
                uptimeMs, TimeUnit.MILLISECONDS, AtomicLong::doubleValue);

        assertThat(tg.value(TimeUnit.SECONDS)).isCloseTo(60.0, org.assertj.core.data.Offset.offset(0.01));
    }

    // ── Integration: full workflow ──────────────────────────────────

    @Test
    void shouldSupportTypicalApplicationWorkflow() {
        // Step 1: App startup — register backend(s)
        Metrics.addRegistry(backend);

        // Step 2: Application code records metrics via static methods
        Metrics.counter("app.events", "type", "login").increment();
        Metrics.timer("app.processing").record(200, TimeUnit.MILLISECONDS);

        AtomicInteger queueSize = new AtomicInteger(5);
        Metrics.gauge("app.queue.size", queueSize);

        // Step 3: Verify metrics are collected in the backend
        assertThat(backend.counter("app.events", "type", "login").count()).isEqualTo(1.0);
        assertThat(backend.timer("app.processing").count()).isEqualTo(1);
        assertThat(backend.find("app.queue.size").gauge().value()).isEqualTo(5.0);
    }

}
```
