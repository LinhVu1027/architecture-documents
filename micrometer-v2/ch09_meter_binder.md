# Chapter 9: MeterBinder — Reusable Instrumentation Modules

> **Feature 9** · Tier 2 · Depends on: Features 1–2 (Counter, Gauge, MeterRegistry)

After this chapter you will be able to **package reusable instrumentation as composable
modules** that register their own meters when bound to a registry — the standard pattern
for JVM metrics, HTTP client metrics, cache metrics, and any other library-provided
instrumentation.

---

## 1. The API Contract

Before writing any implementation, let's define exactly what a client sees.

### What the client imports

```java
import simple.micrometer.api.MeterBinder;
import simple.micrometer.api.binder.JvmMemoryMetrics;
import simple.micrometer.api.MeterRegistry;
import simple.micrometer.api.Gauge;
```

### What the client writes

```java
MeterRegistry registry = new SimpleMeterRegistry();

// Bind a reusable instrumentation module
new JvmMemoryMetrics().bindTo(registry);

// Now registry has gauges for JVM memory metrics
Gauge heapUsed = registry.find("jvm.memory.used")
    .tag("area", "heap")
    .gauge();

// Custom binder via lambda
MeterBinder myBinder = reg -> {
    Gauge.builder("app.queue.size", queue, Queue::size)
        .register(reg);
};
myBinder.bindTo(registry);
```

### Behavioral contract

| Promise | Detail |
|---------|--------|
| Functional interface | `MeterBinder` has exactly one method — `bindTo(MeterRegistry)` — so it can be implemented as a lambda |
| Registry-agnostic | Binders work with any `MeterRegistry` subclass (SimpleMeterRegistry, CompositeMeterRegistry, etc.) |
| Composable | Multiple binders can be bound to the same registry — they just register different meters |
| Reusable | The same binder instance can be bound to multiple registries |
| No lifecycle coupling | `MeterBinder` has no `unbind()` or `close()` — meters persist for the registry's lifetime |
| Extra tags support | `JvmMemoryMetrics` accepts extra tags in its constructor, appended to all registered gauges |

---

## 2. Client Usage & Tests

These tests define the API's behavior — they were written **before** any implementation.

### Register a counter via lambda binder

```java
@Test
void shouldRegisterCounterViaLambdaBinder() {
    MeterBinder binder = reg -> {
        Counter.builder("app.events")
                .tag("type", "login")
                .register(reg);
    };

    binder.bindTo(registry);

    Counter counter = registry.find("app.events")
            .tag("type", "login")
            .counter();
    assertThat(counter).isNotNull();
}
```

### Register a gauge via lambda binder

```java
@Test
void shouldRegisterGaugeViaLambdaBinder() {
    List<String> queue = new ArrayList<>();
    queue.add("item1");
    queue.add("item2");

    MeterBinder binder = reg -> {
        Gauge.builder("app.queue.size", queue, List::size)
                .register(reg);
    };

    binder.bindTo(registry);

    Gauge gauge = registry.find("app.queue.size").gauge();
    assertThat(gauge).isNotNull();
    assertThat(gauge.value()).isEqualTo(2.0);
}
```

### Register multiple meters in a single binder

```java
@Test
void shouldRegisterMultipleMetersInSingleBinder() {
    AtomicInteger activeUsers = new AtomicInteger(42);

    MeterBinder binder = reg -> {
        Counter.builder("app.requests.total")
                .description("Total requests")
                .register(reg);
        Gauge.builder("app.users.active", activeUsers, AtomicInteger::doubleValue)
                .description("Active users")
                .register(reg);
    };

    binder.bindTo(registry);

    assertThat(registry.find("app.requests.total").counter()).isNotNull();
    assertThat(registry.find("app.users.active").gauge()).isNotNull();
    assertThat(registry.find("app.users.active").gauge().value()).isEqualTo(42.0);
}
```

### Bind to multiple registries independently

```java
@Test
void shouldBindToMultipleRegistries() {
    SimpleMeterRegistry registry2 = new SimpleMeterRegistry();

    MeterBinder binder = reg -> {
        Counter.builder("app.events").register(reg);
    };

    binder.bindTo(registry);
    binder.bindTo(registry2);

    Counter c1 = registry.find("app.events").counter();
    Counter c2 = registry2.find("app.events").counter();
    assertThat(c1).isNotNull();
    assertThat(c2).isNotNull();

    c1.increment(10);
    assertThat(c1.count()).isEqualTo(10.0);
    assertThat(c2.count()).isEqualTo(0.0); // independent
}
```

### JvmMemoryMetrics registers heap and nonheap gauges

```java
@Test
void shouldHaveHeapAndNonheapAreas() {
    new JvmMemoryMetrics().bindTo(registry);

    Gauge heapGauge = registry.find("jvm.memory.used")
            .tag("area", "heap").gauge();
    Gauge nonheapGauge = registry.find("jvm.memory.used")
            .tag("area", "nonheap").gauge();

    assertThat(heapGauge).isNotNull();
    assertThat(nonheapGauge).isNotNull();
}
```

### JvmMemoryMetrics includes extra tags when provided

```java
@Test
void shouldIncludeExtraTagsWhenProvided() {
    new JvmMemoryMetrics(Tags.of("app", "order-service")).bindTo(registry);

    Gauge gauge = registry.find("jvm.memory.used")
            .tag("area", "heap")
            .tag("app", "order-service")
            .gauge();
    assertThat(gauge).isNotNull();
}
```

---

## 3. Implementing the Call Chain

The MeterBinder feature has the shallowest call chain of any feature so far — it is
purely an API-layer pattern with no dispatch, no processing, and no infrastructure:

```
Client calls: new JvmMemoryMetrics().bindTo(registry)
  │
  └─► [API Layer] JvmMemoryMetrics.bindTo(registry)
        Iterates JMX MemoryPoolMXBeans and BufferPoolMXBeans
        For each pool:
          1. Determines tags (id=poolName, area=heap/nonheap)
          2. Calls Gauge.builder(name, pool, function).tags(tags).register(registry)
             └─► (reuses the existing Gauge registration chain from Feature 2)
```

There is no new internal machinery — `MeterBinder` is a design pattern, not an
implementation layer. The real work happens in the binder implementations, which
reuse the existing meter registration APIs from Features 1–4.

### Layer 1: MeterBinder — the contract

The interface is exactly one method:

```java
@FunctionalInterface
public interface MeterBinder {
    void bindTo(MeterRegistry registry);
}
```

That's it. The `@FunctionalInterface` annotation is important — it enables lambda syntax
like `MeterBinder binder = reg -> { ... }`, which makes ad-hoc binders trivial to create.

The contract says: "given a registry, register whatever meters you want on it." This
decouples *what to measure* from *where to send it*. A `JvmMemoryMetrics` binder doesn't
know whether the registry is a `SimpleMeterRegistry` (for testing), a Prometheus registry,
or a `CompositeMeterRegistry` publishing to three backends simultaneously.

### Layer 2: JvmMemoryMetrics — a real binder implementation

This is a concrete `MeterBinder` that demonstrates the pattern with real JVM instrumentation.
It uses the `java.lang.management` API to discover memory pools and buffer pools at bind time:

```java
@Override
public void bindTo(MeterRegistry registry) {
    // Memory pools: heap and non-heap
    for (MemoryPoolMXBean pool : ManagementFactory.getMemoryPoolMXBeans()) {
        String poolId = pool.getName();
        String area = pool.getType() == MemoryType.HEAP ? "heap" : "nonheap";
        Tags tags = Tags.of("id", poolId, "area", area).and(extraTags);

        Gauge.builder("jvm.memory.used", pool,
                        mem -> getUsageValue(mem, MemoryUsage::getUsed))
                .tags(tags)
                .description("The amount of used memory")
                .baseUnit("bytes")
                .register(registry);
        // ... committed and max gauges similarly
    }

    // Buffer pools: direct and mapped byte buffers
    for (BufferPoolMXBean bufferPool : ManagementFactory.getPlatformMXBeans(BufferPoolMXBean.class)) {
        // ... buffer count, memory used, total capacity gauges
    }
}
```

Key design choices:

- **Uses `Gauge`, not `Counter`**: Memory usage is a point-in-time value (it goes up AND
  down), not a monotonically increasing total. Each gauge evaluates a `ToDoubleFunction`
  against the MXBean on every `value()` call.

- **Passes the MXBean as the state object**: `Gauge.builder("jvm.memory.used", pool, fn)`
  holds a `WeakReference` to `pool`. Since MXBeans are managed by the JVM platform (strong
  references exist in the `ManagementFactory`), they won't be garbage collected.

- **Null-safe value extraction**: The `getUsageValue()` helper handles `null` from
  `pool.getUsage()` (possible during JVM shutdown), returning `NaN`.

- **Extra tags via constructor**: Operators can add infrastructure-level tags like
  `application=order-service` without modifying the binder's code.

---

## 4. Try It Yourself

1. **Write a `ProcessMetrics` binder** that registers gauges for `process.cpu.count`
   (using `Runtime.getRuntime().availableProcessors()`) and `process.uptime`
   (using `ManagementFactory.getRuntimeMXBean().getUptime()`).

2. **Write a `CacheMetrics` binder** that takes a `Map<String, Object>` in its constructor
   and registers a `Gauge` for `cache.size` and a `Counter` for `cache.evictions`.

3. **Bind the same binder to both a `SimpleMeterRegistry` and a `CompositeMeterRegistry`**
   and verify that both receive the meters independently.

---

## 5. Why This Works

### A functional interface is the simplest possible contract

`MeterBinder` has exactly one abstract method — `bindTo(MeterRegistry)`. This makes it a
functional interface that can be implemented as a lambda, a method reference, or a full
class. The pattern is intentionally minimal: there is no lifecycle management, no
configuration, no registration callbacks. Each binder is a self-contained unit that knows
what meters to register and how.

### Decoupling instrumentation from the registry implementation

A `JvmMemoryMetrics` binder doesn't import `SimpleMeterRegistry` or any specific backend.
It only uses the abstract `MeterRegistry` API (`Gauge.builder(...).register(registry)`).
This means the same binder works with any registry — Prometheus, Datadog, Graphite, or a
testing registry. The monitoring backend is a deployment decision, not a coding decision.

### Why no `unbind()` method?

The real Micrometer `MeterBinder` has no `unbind()` either. Once bound, meters persist for
the registry's lifetime. Binders that need cleanup (like `JvmGcMetrics`, which registers
JMX notification listeners) add `AutoCloseable` as a second interface — keeping the base
contract simple.

### JMX MXBeans as the data source

`JvmMemoryMetrics` uses `ManagementFactory.getMemoryPoolMXBeans()` to discover memory pools
dynamically. This means it adapts to whatever JVM it runs on — G1, ZGC, Shenandoah, or
older collectors. The pool names become tag values (`id`), so the metrics are automatically
dimensioned without hard-coding GC-specific knowledge.

---

## 6. What We Enhanced

| Enhancement | Description | Enabled by |
|-------------|-------------|------------|
| Reusable instrumentation | Package metrics as composable modules, bindable to any registry | `MeterBinder.bindTo()` |
| Lambda binders | Create ad-hoc binders with `MeterBinder binder = reg -> { ... }` | `@FunctionalInterface` |
| JVM memory visibility | Out-of-the-box gauges for heap/nonheap memory pools and buffer pools | `JvmMemoryMetrics` |
| Extra tags | Add infrastructure-level tags without modifying binder source code | `JvmMemoryMetrics(Tags)` constructor |
| Multi-registry binding | Bind the same binder to multiple registries independently | `MeterBinder` contract |

---

## 7. Connection to Real Framework

| Simplified Class | Real Framework Class | Key Simplifications |
|------------------|---------------------|---------------------|
| `MeterBinder` | `io.micrometer.core.instrument.binder.MeterBinder` | Identical — no simplification needed for a one-method interface |
| `JvmMemoryMetrics` | `io.micrometer.core.instrument.binder.jvm.JvmMemoryMetrics` | No `MeterConvention` abstraction, no `BaseUnits` constants class, simpler constructor overloads |

**Source reference**: Micrometer commit `2c8a4606c4771bd98a0bf8f4077ebd383ef2fe4b`

| Simplified | Real Source | Lines |
|------------|------------|-------|
| `MeterBinder.bindTo()` | `MeterBinder.java:29` | Single abstract method, identical contract |
| `JvmMemoryMetrics.bindTo()` | `JvmMemoryMetrics.java:67-113` | Iterates MXBeans, registers gauges |
| `getUsageValue()` | `JvmMemory.getUsageValue()` | Null-safe memory usage extraction |

**What the real framework adds:**
- **`MeterConvention` abstraction**: Allows customizable metric naming per-binder (e.g.,
  OpenTelemetry-compatible names like `jvm.memory.usage` instead of `jvm.memory.used`).
  We hard-code the Micrometer-standard names.
- **`BaseUnits` constants class**: Provides standardized unit strings like `BaseUnits.BYTES`,
  `BaseUnits.BUFFERS`. We use string literals.
- **`JvmGcMetrics`**: A more complex binder that uses JMX notification listeners to track
  GC pauses, concurrent phases, and memory promotion/allocation rates. It implements both
  `MeterBinder` and `AutoCloseable` because it needs to unregister its notification listeners.
- **`JvmThreadMetrics`**, **`JvmCompilationMetrics`**, **`ClassLoaderMetrics`**: Additional
  JVM binders for thread states, JIT compilation time, and class loading statistics.
- **Spring Boot auto-configuration**: In Spring Boot, binders are discovered as beans and
  automatically bound to all `MeterRegistry` beans via `MeterRegistryPostProcessor`.

---

## 8. Complete Code

All files created or modified in this feature. Copy all `[NEW]` files and replace all
`[MODIFIED]` files to get a compiling project with all tests passing.

### `src/main/java/simple/micrometer/api/MeterBinder.java` [NEW]

```java
package simple.micrometer.api;

/**
 * Binders register one or more metrics to provide information about the state
 * of some aspect of the application or its container.
 *
 * <p>This is the standard pattern for packaging reusable instrumentation as
 * composable modules. Library authors implement {@code MeterBinder} to register
 * their own meters when bound to a registry — for example, JVM metrics, HTTP
 * client metrics, cache metrics, connection pool metrics, etc.
 *
 * <p>Usage:
 * <pre>{@code
 * // Bind a reusable instrumentation module
 * new JvmMemoryMetrics().bindTo(registry);
 *
 * // Custom binder via lambda
 * MeterBinder myBinder = reg -> {
 *     Gauge.builder("app.queue.size", queue, Queue::size)
 *         .register(reg);
 * };
 * myBinder.bindTo(registry);
 * }</pre>
 *
 * <p>Because this interface has exactly one abstract method, it is a
 * {@link FunctionalInterface} and can be implemented as a lambda expression.
 *
 * <p>Simplified from {@code io.micrometer.core.instrument.binder.MeterBinder}.
 */
@FunctionalInterface
public interface MeterBinder {

    /**
     * Registers meters with the given registry. Implementations should create
     * and register all their meters inside this method.
     *
     * @param registry the registry to bind meters to
     */
    void bindTo(MeterRegistry registry);

}
```

### `src/main/java/simple/micrometer/api/binder/JvmMemoryMetrics.java` [NEW]

```java
package simple.micrometer.api.binder;

import simple.micrometer.api.Gauge;
import simple.micrometer.api.MeterBinder;
import simple.micrometer.api.MeterRegistry;
import simple.micrometer.api.Tags;

import java.lang.management.BufferPoolMXBean;
import java.lang.management.ManagementFactory;
import java.lang.management.MemoryPoolMXBean;
import java.lang.management.MemoryType;
import java.lang.management.MemoryUsage;

/**
 * Registers gauges that report utilization of JVM memory pools and buffer pools.
 *
 * <p>After binding, the registry will contain gauges like:
 * <ul>
 *   <li>{@code jvm.memory.used} — current used bytes per memory pool</li>
 *   <li>{@code jvm.memory.committed} — committed bytes per memory pool</li>
 *   <li>{@code jvm.memory.max} — maximum bytes per memory pool</li>
 *   <li>{@code jvm.buffer.count} — number of buffers per buffer pool</li>
 *   <li>{@code jvm.buffer.memory.used} — memory used per buffer pool</li>
 *   <li>{@code jvm.buffer.total.capacity} — total capacity per buffer pool</li>
 * </ul>
 *
 * <p>Each memory pool gauge is tagged with:
 * <ul>
 *   <li>{@code id} — the pool name (e.g., "G1 Eden Space", "Metaspace")</li>
 *   <li>{@code area} — "heap" or "nonheap"</li>
 * </ul>
 *
 * <p>Each buffer pool gauge is tagged with:
 * <ul>
 *   <li>{@code id} — the buffer pool name (e.g., "direct", "mapped")</li>
 * </ul>
 *
 * <p>Usage:
 * <pre>{@code
 * MeterRegistry registry = new SimpleMeterRegistry();
 * new JvmMemoryMetrics().bindTo(registry);
 *
 * // Now query the metrics
 * Gauge heapUsed = registry.find("jvm.memory.used")
 *     .tag("area", "heap")
 *     .gauge();
 * }</pre>
 *
 * <p>Simplified from {@code io.micrometer.core.instrument.binder.jvm.JvmMemoryMetrics}.
 * The real framework adds a {@code MeterConvention} abstraction for customizable metric
 * naming (e.g., OpenTelemetry-compatible names), a {@code BaseUnits} constants class,
 * and extra-tag support via constructor parameters.
 *
 * @see MemoryPoolMXBean
 * @see BufferPoolMXBean
 */
public class JvmMemoryMetrics implements MeterBinder {

    private final Tags extraTags;

    /**
     * Creates a JvmMemoryMetrics binder with no extra tags.
     */
    public JvmMemoryMetrics() {
        this(Tags.empty());
    }

    /**
     * Creates a JvmMemoryMetrics binder with extra tags that will be added
     * to every gauge registered by this binder.
     *
     * @param extraTags additional tags for all gauges
     */
    public JvmMemoryMetrics(Tags extraTags) {
        this.extraTags = extraTags;
    }

    @Override
    public void bindTo(MeterRegistry registry) {
        // Memory pools: heap and non-heap (e.g., G1 Eden, G1 Old Gen, Metaspace)
        for (MemoryPoolMXBean pool : ManagementFactory.getMemoryPoolMXBeans()) {
            String poolId = pool.getName();
            String area = pool.getType() == MemoryType.HEAP ? "heap" : "nonheap";

            Tags tags = Tags.of("id", poolId, "area", area).and(extraTags);

            Gauge.builder("jvm.memory.used", pool,
                            mem -> getUsageValue(mem, MemoryUsage::getUsed))
                    .tags(tags)
                    .description("The amount of used memory")
                    .baseUnit("bytes")
                    .register(registry);

            Gauge.builder("jvm.memory.committed", pool,
                            mem -> getUsageValue(mem, MemoryUsage::getCommitted))
                    .tags(tags)
                    .description("The amount of memory that is committed for the JVM to use")
                    .baseUnit("bytes")
                    .register(registry);

            Gauge.builder("jvm.memory.max", pool,
                            mem -> getUsageValue(mem, MemoryUsage::getMax))
                    .tags(tags)
                    .description("The maximum amount of memory that can be used")
                    .baseUnit("bytes")
                    .register(registry);
        }

        // Buffer pools: direct and mapped byte buffers
        for (BufferPoolMXBean bufferPool : ManagementFactory.getPlatformMXBeans(BufferPoolMXBean.class)) {
            Tags tags = Tags.of("id", bufferPool.getName()).and(extraTags);

            Gauge.builder("jvm.buffer.count", bufferPool, BufferPoolMXBean::getCount)
                    .tags(tags)
                    .description("An estimate of the number of buffers in the pool")
                    .baseUnit("buffers")
                    .register(registry);

            Gauge.builder("jvm.buffer.memory.used", bufferPool, BufferPoolMXBean::getMemoryUsed)
                    .tags(tags)
                    .description("An estimate of the memory that the JVM is using for this buffer pool")
                    .baseUnit("bytes")
                    .register(registry);

            Gauge.builder("jvm.buffer.total.capacity", bufferPool, BufferPoolMXBean::getTotalCapacity)
                    .tags(tags)
                    .description("An estimate of the total capacity of the buffers in this pool")
                    .baseUnit("bytes")
                    .register(registry);
        }
    }

    /**
     * Safely extracts a usage value from a memory pool. Returns {@link Double#NaN}
     * if the pool's usage is unavailable.
     */
    private static double getUsageValue(MemoryPoolMXBean pool,
                                        java.util.function.ToLongFunction<MemoryUsage> extractor) {
        MemoryUsage usage = pool.getUsage();
        if (usage == null) {
            return Double.NaN;
        }
        return extractor.applyAsLong(usage);
    }

}
```

### `src/test/java/simple/micrometer/api/MeterBinderTest.java` [NEW]

```java
package simple.micrometer.api;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Nested;
import org.junit.jupiter.api.Test;
import simple.micrometer.api.binder.JvmMemoryMetrics;
import simple.micrometer.internal.SimpleMeterRegistry;

import java.util.ArrayList;
import java.util.Collection;
import java.util.List;
import java.util.concurrent.atomic.AtomicInteger;

import static org.assertj.core.api.Assertions.assertThat;

/**
 * Client-perspective tests for the {@link MeterBinder} API.
 *
 * <p>These tests verify:
 * <ul>
 *   <li>Custom binders can register meters via lambda</li>
 *   <li>A binder can be bound to multiple registries</li>
 *   <li>{@link JvmMemoryMetrics} registers expected JVM gauges</li>
 * </ul>
 */
@DisplayName("MeterBinder API")
class MeterBinderTest {

    private SimpleMeterRegistry registry;

    @BeforeEach
    void setUp() {
        registry = new SimpleMeterRegistry();
    }

    // ── Custom binder (lambda) ─────────────────────────────────────────

    @Nested
    @DisplayName("Custom lambda binder")
    class CustomBinder {

        @Test
        @DisplayName("should register a counter via lambda binder")
        void shouldRegisterCounterViaLambdaBinder() {
            MeterBinder binder = reg -> {
                Counter.builder("app.events")
                        .tag("type", "login")
                        .register(reg);
            };

            binder.bindTo(registry);

            Counter counter = registry.find("app.events")
                    .tag("type", "login")
                    .counter();
            assertThat(counter).isNotNull();
        }

        @Test
        @DisplayName("should register a gauge via lambda binder")
        void shouldRegisterGaugeViaLambdaBinder() {
            List<String> queue = new ArrayList<>();
            queue.add("item1");
            queue.add("item2");

            MeterBinder binder = reg -> {
                Gauge.builder("app.queue.size", queue, List::size)
                        .register(reg);
            };

            binder.bindTo(registry);

            Gauge gauge = registry.find("app.queue.size").gauge();
            assertThat(gauge).isNotNull();
            assertThat(gauge.value()).isEqualTo(2.0);
        }

        @Test
        @DisplayName("should register multiple meters in a single binder")
        void shouldRegisterMultipleMetersInSingleBinder() {
            AtomicInteger activeUsers = new AtomicInteger(42);

            MeterBinder binder = reg -> {
                Counter.builder("app.requests.total")
                        .description("Total requests")
                        .register(reg);
                Gauge.builder("app.users.active", activeUsers, AtomicInteger::doubleValue)
                        .description("Active users")
                        .register(reg);
            };

            binder.bindTo(registry);

            assertThat(registry.find("app.requests.total").counter()).isNotNull();
            assertThat(registry.find("app.users.active").gauge()).isNotNull();
            assertThat(registry.find("app.users.active").gauge().value()).isEqualTo(42.0);
        }

        @Test
        @DisplayName("should bind to multiple registries independently")
        void shouldBindToMultipleRegistries() {
            SimpleMeterRegistry registry2 = new SimpleMeterRegistry();

            MeterBinder binder = reg -> {
                Counter.builder("app.events")
                        .register(reg);
            };

            binder.bindTo(registry);
            binder.bindTo(registry2);

            // Both registries have independent counters
            Counter c1 = registry.find("app.events").counter();
            Counter c2 = registry2.find("app.events").counter();
            assertThat(c1).isNotNull();
            assertThat(c2).isNotNull();

            c1.increment(10);
            assertThat(c1.count()).isEqualTo(10.0);
            assertThat(c2.count()).isEqualTo(0.0); // independent
        }

        @Test
        @DisplayName("should work with method reference as binder")
        void shouldWorkWithMethodReference() {
            MeterBinder binder = MeterBinderTest.this::registerAppMetrics;
            binder.bindTo(registry);

            assertThat(registry.find("app.startup.time").gauge()).isNotNull();
        }
    }

    private void registerAppMetrics(MeterRegistry reg) {
        Gauge.builder("app.startup.time", () -> 1234)
                .description("Application startup time in ms")
                .register(reg);
    }

    // ── JvmMemoryMetrics ──────────────────────────────────────────────

    @Nested
    @DisplayName("JvmMemoryMetrics binder")
    class JvmMemoryMetricsBinder {

        @Test
        @DisplayName("should register jvm.memory.used gauges")
        void shouldRegisterMemoryUsedGauges() {
            new JvmMemoryMetrics().bindTo(registry);

            Collection<Gauge> gauges = registry.find("jvm.memory.used").gauges();
            assertThat(gauges).isNotEmpty();
        }

        @Test
        @DisplayName("should register jvm.memory.committed gauges")
        void shouldRegisterMemoryCommittedGauges() {
            new JvmMemoryMetrics().bindTo(registry);

            Collection<Gauge> gauges = registry.find("jvm.memory.committed").gauges();
            assertThat(gauges).isNotEmpty();
        }

        @Test
        @DisplayName("should register jvm.memory.max gauges")
        void shouldRegisterMemoryMaxGauges() {
            new JvmMemoryMetrics().bindTo(registry);

            Collection<Gauge> gauges = registry.find("jvm.memory.max").gauges();
            assertThat(gauges).isNotEmpty();
        }

        @Test
        @DisplayName("should tag memory gauges with id and area")
        void shouldTagMemoryGaugesWithIdAndArea() {
            new JvmMemoryMetrics().bindTo(registry);

            // Every jvm.memory.used gauge should have both "id" and "area" tags
            Collection<Gauge> gauges = registry.find("jvm.memory.used").gauges();
            for (Gauge gauge : gauges) {
                List<Tag> tags = gauge.getId().getTags();
                assertThat(tags.stream().anyMatch(t -> t.getKey().equals("id")))
                        .as("Gauge should have 'id' tag")
                        .isTrue();
                assertThat(tags.stream().anyMatch(t -> t.getKey().equals("area")))
                        .as("Gauge should have 'area' tag")
                        .isTrue();
            }
        }

        @Test
        @DisplayName("should have heap and nonheap areas")
        void shouldHaveHeapAndNonheapAreas() {
            new JvmMemoryMetrics().bindTo(registry);

            Gauge heapGauge = registry.find("jvm.memory.used")
                    .tag("area", "heap")
                    .gauge();
            Gauge nonheapGauge = registry.find("jvm.memory.used")
                    .tag("area", "nonheap")
                    .gauge();

            // At least one of each should exist on any standard JVM
            assertThat(heapGauge).isNotNull();
            assertThat(nonheapGauge).isNotNull();
        }

        @Test
        @DisplayName("should return positive values for memory used")
        void shouldReturnPositiveValuesForMemoryUsed() {
            new JvmMemoryMetrics().bindTo(registry);

            Gauge heapUsed = registry.find("jvm.memory.used")
                    .tag("area", "heap")
                    .gauge();
            assertThat(heapUsed).isNotNull();
            assertThat(heapUsed.value()).isGreaterThan(0);
        }

        @Test
        @DisplayName("should register buffer pool gauges")
        void shouldRegisterBufferPoolGauges() {
            new JvmMemoryMetrics().bindTo(registry);

            // Buffer pool gauges should exist (direct and mapped on standard JVMs)
            Collection<Gauge> countGauges = registry.find("jvm.buffer.count").gauges();
            Collection<Gauge> usedGauges = registry.find("jvm.buffer.memory.used").gauges();
            Collection<Gauge> capacityGauges = registry.find("jvm.buffer.total.capacity").gauges();

            assertThat(countGauges).isNotEmpty();
            assertThat(usedGauges).isNotEmpty();
            assertThat(capacityGauges).isNotEmpty();
        }

        @Test
        @DisplayName("should tag buffer gauges with id")
        void shouldTagBufferGaugesWithId() {
            new JvmMemoryMetrics().bindTo(registry);

            Collection<Gauge> gauges = registry.find("jvm.buffer.count").gauges();
            for (Gauge gauge : gauges) {
                List<Tag> tags = gauge.getId().getTags();
                assertThat(tags.stream().anyMatch(t -> t.getKey().equals("id")))
                        .as("Buffer gauge should have 'id' tag")
                        .isTrue();
            }
        }

        @Test
        @DisplayName("should include extra tags when provided")
        void shouldIncludeExtraTagsWhenProvided() {
            new JvmMemoryMetrics(Tags.of("app", "order-service")).bindTo(registry);

            Gauge gauge = registry.find("jvm.memory.used")
                    .tag("area", "heap")
                    .tag("app", "order-service")
                    .gauge();
            assertThat(gauge).isNotNull();
        }

        @Test
        @DisplayName("should bind to multiple registries independently")
        void shouldBindToMultipleRegistries() {
            SimpleMeterRegistry registry2 = new SimpleMeterRegistry();
            JvmMemoryMetrics binder = new JvmMemoryMetrics();

            binder.bindTo(registry);
            binder.bindTo(registry2);

            // Both registries should have memory gauges
            assertThat(registry.find("jvm.memory.used").gauges()).isNotEmpty();
            assertThat(registry2.find("jvm.memory.used").gauges()).isNotEmpty();
        }
    }

}
```
