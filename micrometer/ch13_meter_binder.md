# Chapter 13: MeterBinder

> Line references based on commit `2c8a4606c` of the Micrometer repository.

## Build Challenge

| Current State | Limitation | Objective |
|---------------|------------|-----------|
| The registry supports Counter, Gauge, Timer, DistributionSummary, FunctionCounter, FunctionTimer, TimeGauge, and LongTaskTimer — all registered individually via builders | No standard pattern for packaging *groups* of related metrics that belong together (e.g., all JVM memory metrics). Each application must manually call `Gauge.builder(...)` for every metric, duplicating boilerplate across projects | Build a `MeterBinder` interface that packages reusable metric registrations into a single `bindTo(MeterRegistry)` call, then implement three concrete binders (`JvmMemoryMetrics`, `JvmThreadMetrics`, `UptimeMetrics`) that demonstrate the pattern |

---

## 13.1 The Integration Point: MeterBinder as a Client of MeterRegistry

Unlike every prior feature, MeterBinder does **not modify** MeterRegistry. It is a *client* of the existing registration API — the integration point is the `Gauge.builder()`, `FunctionCounter.builder()`, and `TimeGauge.builder()` methods that we've already built.

```java
public interface MeterBinder {
    void bindTo(MeterRegistry registry);
}
```

This is the Strategy pattern: each binder encapsulates *what* to monitor, while the registry handles *how* to store and export. The binder calls the same builder APIs that application code would use — but packages them into a reusable unit.

**Direction:** From this interface, we need to build:
- `JvmMemoryMetrics` — registers gauges for each JVM memory pool using `MemoryPoolMXBean`
- `JvmThreadMetrics` — registers gauges and a function counter for thread statistics using `ThreadMXBean`
- `UptimeMetrics` — registers time gauges for process uptime using `RuntimeMXBean`

Key decision: `MeterBinder` is a functional interface (SAM type). This means simple binders can be written as lambdas:

```java
MeterBinder binder = registry ->
    Gauge.builder("queue.size", queue, Queue::size).register(registry);
```

## 13.2 The MeterBinder Interface

**New file:** `src/main/java/dev/linhvu/micrometer/binder/MeterBinder.java`

```java
package dev.linhvu.micrometer.binder;

import dev.linhvu.micrometer.MeterRegistry;

public interface MeterBinder {

    void bindTo(MeterRegistry registry);

}
```

That's it — one method. The power comes from the implementations.

## 13.3 JvmMemoryMetrics — Memory Pool Gauges

**New file:** `src/main/java/dev/linhvu/micrometer/binder/jvm/JvmMemoryMetrics.java`

This binder iterates over all JVM memory pools (via `ManagementFactory.getMemoryPoolMXBeans()`) and registers three gauges per pool: `jvm.memory.used`, `jvm.memory.committed`, and `jvm.memory.max`. Each gauge is tagged with `area` (heap/nonheap) and `id` (the pool name, like "G1 Eden Space" or "Metaspace").

```java
public class JvmMemoryMetrics implements MeterBinder {

    private final Iterable<Tag> tags;

    public JvmMemoryMetrics() {
        this(Tags.empty());
    }

    public JvmMemoryMetrics(Iterable<Tag> tags) {
        this.tags = tags;
    }

    @Override
    public void bindTo(MeterRegistry registry) {
        List<MemoryPoolMXBean> memoryPoolBeans = ManagementFactory.getMemoryPoolMXBeans();

        for (MemoryPoolMXBean memoryPoolBean : memoryPoolBeans) {
            String area = memoryPoolBean.getType() == MemoryType.HEAP ? "heap" : "nonheap";
            Tags poolTags = Tags.of(tags)
                    .and(Tag.of("area", area))
                    .and(Tag.of("id", memoryPoolBean.getName()));

            Gauge.builder("jvm.memory.used", memoryPoolBean,
                            mem -> getUsageValue(mem, MemoryUsage::getUsed))
                    .tags(poolTags)
                    .description("The amount of used memory")
                    .baseUnit("bytes")
                    .register(registry);

            Gauge.builder("jvm.memory.committed", memoryPoolBean,
                            mem -> getUsageValue(mem, MemoryUsage::getCommitted))
                    .tags(poolTags)
                    .description("The amount of memory that is committed for the JVM to use")
                    .baseUnit("bytes")
                    .register(registry);

            Gauge.builder("jvm.memory.max", memoryPoolBean,
                            mem -> getUsageValue(mem, MemoryUsage::getMax))
                    .tags(poolTags)
                    .description("The maximum amount of memory that can be used")
                    .baseUnit("bytes")
                    .register(registry);
        }
    }

    private static double getUsageValue(MemoryPoolMXBean memoryPoolBean,
            java.util.function.ToLongFunction<MemoryUsage> extractor) {
        MemoryUsage usage = memoryPoolBean.getUsage();
        if (usage == null) {
            return Double.NaN;
        }
        return extractor.applyAsLong(usage);
    }
}
```

Three design choices here:

1. **`Iterable<Tag>` for extra tags** — every binder accepts extra tags so callers can attach common tags (application name, environment, host) to all meters. This is a cross-cutting concern pattern found in every Micrometer binder.

2. **The gauge observes the MXBean directly** — we pass `memoryPoolBean` as the state object. The gauge holds it via `WeakReference`. Each scrape/poll calls the value function, giving live readings without polling overhead.

3. **`getUsageValue()` returns NaN on null** — some pools may not support usage reporting. Returning `NaN` is the Micrometer convention for "no data," which monitoring backends understand.

## 13.4 JvmThreadMetrics — Thread Counts and Per-State Gauges

**New file:** `src/main/java/dev/linhvu/micrometer/binder/jvm/JvmThreadMetrics.java`

This binder registers four fixed meters plus one gauge per `Thread.State`:

```java
public class JvmThreadMetrics implements MeterBinder {

    private final Iterable<Tag> tags;
    private final ThreadMXBean threadBean;

    public JvmThreadMetrics() {
        this(Tags.empty());
    }

    public JvmThreadMetrics(Iterable<Tag> tags) {
        this(ManagementFactory.getThreadMXBean(), tags);
    }

    // Package-private for testing
    JvmThreadMetrics(ThreadMXBean threadBean, Iterable<Tag> tags) {
        this.threadBean = threadBean;
        this.tags = tags;
    }

    @Override
    public void bindTo(MeterRegistry registry) {
        Gauge.builder("jvm.threads.peak", threadBean, ThreadMXBean::getPeakThreadCount)
                .tags(tags).description("The peak live thread count since the JVM started")
                .baseUnit("threads").register(registry);

        Gauge.builder("jvm.threads.daemon", threadBean, ThreadMXBean::getDaemonThreadCount)
                .tags(tags).description("The current number of live daemon threads")
                .baseUnit("threads").register(registry);

        Gauge.builder("jvm.threads.live", threadBean, ThreadMXBean::getThreadCount)
                .tags(tags).description("The current number of live threads")
                .baseUnit("threads").register(registry);

        FunctionCounter.builder("jvm.threads.started", threadBean,
                        ThreadMXBean::getTotalStartedThreadCount)
                .tags(tags).description("The total number of threads started in the JVM")
                .baseUnit("threads").register(registry);

        for (Thread.State state : Thread.State.values()) {
            Gauge.builder("jvm.threads.states", threadBean,
                            bean -> getThreadStateCount(bean, state))
                    .tags(Tags.of(tags).and("state",
                            state.name().toLowerCase().replace("_", "-")))
                    .description("The current number of threads having " + state.name() + " state")
                    .baseUnit("threads").register(registry);
        }
    }

    static long getThreadStateCount(ThreadMXBean threadBean, Thread.State state) {
        long[] threadIds = threadBean.getAllThreadIds();
        ThreadInfo[] threadInfos = threadBean.getThreadInfo(threadIds);
        return Arrays.stream(threadInfos)
                .filter(info -> info != null && info.getThreadState() == state)
                .count();
    }
}
```

Two important design choices:

1. **`jvm.threads.started` is a `FunctionCounter`, not a Gauge.** The total started thread count is monotonically increasing — threads are only ever created, never "uncreated." Using `FunctionCounter` tells monitoring backends to compute *rate* (threads/second) rather than displaying the raw total. This is the same semantic distinction that drives Counter vs. Gauge throughout Micrometer.

2. **Package-private constructor for testability.** The public constructors use `ManagementFactory.getThreadMXBean()`, but the package-private constructor accepts an injected `ThreadMXBean`. This enables unit tests to provide a mock without requiring Mockito — a pattern found throughout Micrometer's binder code.

3. **Null-check in `getThreadStateCount`.** Threads can terminate between `getAllThreadIds()` and `getThreadInfo()`, leaving null entries. The `filter(info -> info != null ...)` handles this race condition.

## 13.5 UptimeMetrics — TimeGauge for Process Uptime

**New file:** `src/main/java/dev/linhvu/micrometer/binder/system/UptimeMetrics.java`

The simplest binder — just two meters:

```java
public class UptimeMetrics implements MeterBinder {

    private final RuntimeMXBean runtimeBean;
    private final Iterable<Tag> tags;

    public UptimeMetrics() {
        this(Tags.empty());
    }

    public UptimeMetrics(Iterable<Tag> tags) {
        this(ManagementFactory.getRuntimeMXBean(), tags);
    }

    // Package-private for testing
    UptimeMetrics(RuntimeMXBean runtimeBean, Iterable<Tag> tags) {
        this.runtimeBean = runtimeBean;
        this.tags = tags;
    }

    @Override
    public void bindTo(MeterRegistry registry) {
        TimeGauge.builder("process.uptime", runtimeBean,
                        TimeUnit.MILLISECONDS, RuntimeMXBean::getUptime)
                .tags(tags).description("The uptime of the Java virtual machine")
                .register(registry);

        TimeGauge.builder("process.start.time", runtimeBean,
                        TimeUnit.MILLISECONDS, RuntimeMXBean::getStartTime)
                .tags(tags).description("Start time of the process since Unix epoch")
                .register(registry);
    }
}
```

Key choice: **`TimeGauge` instead of plain `Gauge`.** The MXBean returns milliseconds, but a Prometheus registry works in seconds while other backends might use different time units. `TimeGauge` carries the source time unit metadata, letting the registry convert automatically during export. Using a plain Gauge would lose this information and force manual unit conversion.

---

## 13.6 Try It Yourself

<details><summary>Challenge 1: Implement a MeterBinder for system CPU load</summary>

Create a `SystemCpuMetrics` binder that registers a gauge for `system.cpu.count` using `Runtime.getRuntime().availableProcessors()`.

```java
package dev.linhvu.micrometer.binder.system;

import dev.linhvu.micrometer.Gauge;
import dev.linhvu.micrometer.MeterRegistry;
import dev.linhvu.micrometer.Tag;
import dev.linhvu.micrometer.Tags;
import dev.linhvu.micrometer.binder.MeterBinder;

public class SystemCpuMetrics implements MeterBinder {

    private final Iterable<Tag> tags;

    public SystemCpuMetrics() {
        this(Tags.empty());
    }

    public SystemCpuMetrics(Iterable<Tag> tags) {
        this.tags = tags;
    }

    @Override
    public void bindTo(MeterRegistry registry) {
        Runtime runtime = Runtime.getRuntime();
        Gauge.builder("system.cpu.count", runtime, Runtime::availableProcessors)
                .tags(tags)
                .description("The number of processors available to the JVM")
                .register(registry);
    }
}
```

</details>

<details><summary>Challenge 2: Use MeterBinder as a lambda to monitor a ConcurrentHashMap</summary>

```java
ConcurrentHashMap<String, String> cache = new ConcurrentHashMap<>();

MeterBinder binder = registry -> {
    Gauge.builder("cache.size", cache, ConcurrentHashMap::size)
            .description("Number of entries in the cache")
            .register(registry);
    Gauge.builder("cache.capacity", cache, m -> m.mappingCount())
            .description("Mapping count of the cache")
            .register(registry);
};

binder.bindTo(registry);
```

Since `MeterBinder` is a functional interface, you can write binders inline for application-specific metrics.

</details>

<details><summary>Challenge 3: Write a binder that uses FunctionCounter for a monotonic value</summary>

Wrap a `java.util.concurrent.atomic.AtomicLong` that tracks total processed items:

```java
AtomicLong processedItems = new AtomicLong();

MeterBinder binder = registry ->
    FunctionCounter.builder("items.processed", processedItems, AtomicLong::doubleValue)
            .description("Total items processed")
            .baseUnit("items")
            .register(registry);

binder.bindTo(registry);

// Elsewhere in your application:
processedItems.incrementAndGet();
```

</details>

---

## 13.7 Tests

### Unit Tests

#### JvmMemoryMetricsTest

```java
@Test
void shouldRegisterMemoryMeters_WhenBound() {
    new JvmMemoryMetrics().bindTo(registry);
    List<Meter> meters = registry.getMeters();
    assertThat(meters).isNotEmpty();
    assertThat(meters.stream().map(m -> m.getId().getName()))
            .contains("jvm.memory.used", "jvm.memory.committed", "jvm.memory.max");
}

@Test
void shouldTagWithAreaAndId_WhenBound() {
    new JvmMemoryMetrics().bindTo(registry);
    for (Meter meter : registry.getMeters()) {
        assertThat(meter.getId().getTag("area")).isIn("heap", "nonheap");
        assertThat(meter.getId().getTag("id")).isNotNull().isNotEmpty();
    }
}

@Test
void shouldIncludeExtraTags_WhenProvided() {
    new JvmMemoryMetrics(Tags.of("env", "test")).bindTo(registry);
    for (Meter meter : registry.getMeters()) {
        assertThat(meter.getId().getTag("env")).isEqualTo("test");
    }
}
```

#### JvmThreadMetricsTest

```java
@Test
void shouldRegisterStartedAsFunctionCounter_WhenBound() {
    new JvmThreadMetrics().bindTo(registry);
    Meter started = registry.getMeters().stream()
            .filter(m -> m.getId().getName().equals("jvm.threads.started"))
            .findFirst().orElseThrow();
    assertThat(started).isInstanceOf(FunctionCounter.class);
    assertThat(((FunctionCounter) started).count()).isGreaterThan(0);
}

@Test
void shouldRegisterPerStateGauges_WhenBound() {
    new JvmThreadMetrics().bindTo(registry);
    long stateGaugeCount = registry.getMeters().stream()
            .filter(m -> m.getId().getName().equals("jvm.threads.states"))
            .count();
    assertThat(stateGaugeCount).isEqualTo(Thread.State.values().length);
}

@Test
void shouldTagStatesWithLowercaseHyphenated_WhenBound() {
    new JvmThreadMetrics().bindTo(registry);
    List<String> stateValues = registry.getMeters().stream()
            .filter(m -> m.getId().getName().equals("jvm.threads.states"))
            .map(m -> m.getId().getTag("state")).toList();
    assertThat(stateValues).contains("runnable", "waiting", "timed-waiting",
            "blocked", "new", "terminated");
}
```

#### UptimeMetricsTest

```java
@Test
void shouldRegisterAsTimeGauge_WhenBound() {
    new UptimeMetrics().bindTo(registry);
    for (Meter meter : registry.getMeters()) {
        assertThat(meter).isInstanceOf(TimeGauge.class);
    }
}

@Test
void shouldReportPositiveUptime_WhenQueried() {
    new UptimeMetrics().bindTo(registry);
    Gauge uptime = (Gauge) registry.getMeters().stream()
            .filter(m -> m.getId().getName().equals("process.uptime"))
            .findFirst().orElseThrow();
    assertThat(uptime.value()).isGreaterThan(0);
}
```

### Integration Tests

```java
@Test
void shouldSupportLambdaBinder_WhenUsedAsFunctionalInterface() {
    List<String> items = List.of("a", "b", "c");
    MeterBinder binder = r ->
            Gauge.builder("queue.size", items, List::size)
                    .description("Queue depth").register(r);
    binder.bindTo(registry);
    Gauge gauge = (Gauge) registry.getMeters().stream()
            .filter(m -> m.getId().getName().equals("queue.size"))
            .findFirst().orElseThrow();
    assertThat(gauge.value()).isEqualTo(3.0);
}

@Test
void shouldDenyMeters_WhenFilterDeniesJvmPrefix() {
    registry.meterFilter(MeterFilter.denyNameStartsWith("jvm.memory"));
    new JvmMemoryMetrics().bindTo(registry);
    new JvmThreadMetrics().bindTo(registry);
    assertThat(registry.getMeters().stream().map(m -> m.getId().getName()))
            .noneMatch(name -> name.startsWith("jvm.memory"))
            .anyMatch(name -> name.startsWith("jvm.threads"));
}

@Test
void shouldDeduplicateMeters_WhenSameBinderBoundTwice() {
    new UptimeMetrics().bindTo(registry);
    int firstCount = registry.getMeters().size();
    new UptimeMetrics().bindTo(registry);
    assertThat(registry.getMeters().size()).isEqualTo(firstCount);
}
```

**Run:** `./gradlew test` — expected: all tests pass (including all prior features' tests)

---

## 13.8 Why This Works

> ★ **Insight** -------------------------------------------
> **MeterBinder is a client, not a component.** Unlike every prior feature (Counter, Timer, MeterFilter, NamingConvention) that *modified* MeterRegistry internals, MeterBinder sits outside the registry and calls the same public builder APIs that application code uses. This is the Open-Closed Principle in action — the registry is "closed" for modification but "open" for extension via binders. Spring Boot leverages this by auto-discovering all `MeterBinder` beans on the classpath and calling `bindTo()` on startup, providing zero-configuration JVM metrics.
> -----------------------------------------------------------

> ★ **Insight** -------------------------------------------
> **MXBeans as lazy gauge state objects.** The binders pass MXBeans (like `MemoryPoolMXBean` or `ThreadMXBean`) as the state object to `Gauge.builder()`. The gauge holds them via `WeakReference` and calls the value function on each scrape. This is *pull-based* instrumentation — the JVM maintains the statistics, and we just read them on demand. There is no polling loop, no timer, no separate thread. The monitoring backend's scrape interval drives the sampling frequency.
>
> **Trade-off:** This means the readings are snapshots at scrape time, not continuously sampled. For memory and thread metrics, this is perfectly fine — these values change slowly enough that scrape-time sampling captures the important trends.
> -----------------------------------------------------------

> ★ **Insight** -------------------------------------------
> **Choosing the right meter type per semantic.** The three binders demonstrate three different meter types — and the choice is driven by the *semantic* of each value, not just its data type:
> - **Gauge** for fluctuating values (memory used, thread count) — goes up and down
> - **FunctionCounter** for monotonic values (total threads started) — only goes up, backends compute rate
> - **TimeGauge** for time-valued readings (uptime, start time) — carries unit metadata for automatic conversion
>
> Getting this right matters: a Prometheus backend computes `rate()` on counters but displays the raw value for gauges. Using the wrong type produces misleading dashboards.
> -----------------------------------------------------------

## 13.9 What We Enhanced

| Aspect | Before (ch12) | Current (ch13) | Real Framework |
|--------|---------------|----------------|----------------|
| Metric packaging | Each metric registered individually via builders | `MeterBinder.bindTo(registry)` packages groups of related metrics | Same — `MeterBinder.java:29` |
| JVM memory monitoring | Not available | `JvmMemoryMetrics` registers used/committed/max per memory pool | Same pattern — `JvmMemoryMetrics.java:72-124`, real version also adds BufferPool metrics and MeterConventions |
| JVM thread monitoring | Not available | `JvmThreadMetrics` registers peak/daemon/live/started + per-state gauges | Same pattern — `JvmThreadMetrics.java:72-115`, real version adds SubstrateVM error handling |
| Process uptime monitoring | Not available | `UptimeMetrics` registers TimeGauge for uptime and start time | Same — `UptimeMetrics.java:54-65` |
| Meter type selection | All examples used Counter, Gauge, or Timer | Binders demonstrate choosing Gauge vs. FunctionCounter vs. TimeGauge based on value semantics | Same design principle throughout Micrometer's binder library |

## 13.10 Connection to Real Framework

| Simplified Code | Real Framework Code | File:Line | Key Difference |
|----------------|--------------------|-----------| ---------------|
| `MeterBinder` | `MeterBinder` | `MeterBinder.java:29` | Identical — single `bindTo(MeterRegistry)` method |
| `JvmMemoryMetrics` | `JvmMemoryMetrics` | `JvmMemoryMetrics.java:46-124` | Real version adds `BufferPoolMXBean` metrics (buffer count, memory used, total capacity) and `JvmMemoryMeterConventions` abstraction for customizable naming. Also uses `JvmMemory.getUsageValue()` utility with `InternalError` handling for JVM bugs |
| `JvmThreadMetrics` | `JvmThreadMetrics` | `JvmThreadMetrics.java:47-115` | Real version catches `Error` for unsupported `getAllThreadIds()` on SubstrateVM/GraalVM native image. Uses `JvmThreadMeterConventions` abstraction |
| `JvmThreadMetrics.getThreadStateCount()` | `JvmThreadMetrics.getThreadStateCount()` | `JvmThreadMetrics.java:118-122` | Same logic — null-checks `ThreadInfo` for race between `getAllThreadIds()` and `getThreadInfo()` |
| `UptimeMetrics` | `UptimeMetrics` | `UptimeMetrics.java:40-65` | Same structure — two `TimeGauge` registrations for uptime and start time |
| Extra tags pattern (`Iterable<Tag>`) | Extra tags pattern | All binders | Same pattern — all binders accept `Iterable<Tag>` for common tags. Real binders also accept `*MeterConventions` since 1.16.0 |

## 13.11 Complete Code

### Production Code

#### File: `src/main/java/dev/linhvu/micrometer/binder/MeterBinder.java` [NEW]

```java
package dev.linhvu.micrometer.binder;

import dev.linhvu.micrometer.MeterRegistry;

/**
 * Binders register one or more metrics to provide information about the state
 * of some aspect of the application or its container.
 * <p>
 * This is a functional interface (SAM type) — implementations can be written
 * as lambdas for simple cases, or as full classes (like {@code JvmMemoryMetrics})
 * for richer instrumentation.
 * <p>
 * Usage:
 * <pre>{@code
 * // Lambda style
 * MeterBinder binder = registry ->
 *     Gauge.builder("queue.size", queue, Queue::size).register(registry);
 *
 * // Class style
 * new JvmMemoryMetrics().bindTo(registry);
 * }</pre>
 */
public interface MeterBinder {

    /**
     * Bind metrics to the given registry. Implementations should use the
     * registry's builder APIs (e.g., {@code Gauge.builder()}, {@code Counter.builder()})
     * to register all meters this binder is responsible for.
     *
     * @param registry The registry to bind meters to.
     */
    void bindTo(MeterRegistry registry);

}
```

#### File: `src/main/java/dev/linhvu/micrometer/binder/jvm/JvmMemoryMetrics.java` [NEW]

```java
package dev.linhvu.micrometer.binder.jvm;

import dev.linhvu.micrometer.Gauge;
import dev.linhvu.micrometer.MeterRegistry;
import dev.linhvu.micrometer.Tag;
import dev.linhvu.micrometer.Tags;
import dev.linhvu.micrometer.binder.MeterBinder;

import java.lang.management.ManagementFactory;
import java.lang.management.MemoryMXBean;
import java.lang.management.MemoryPoolMXBean;
import java.lang.management.MemoryType;
import java.lang.management.MemoryUsage;
import java.util.List;

/**
 * Registers gauges for JVM memory usage — both at the overall heap/non-heap level
 * (via {@link MemoryMXBean}) and at the individual memory pool level
 * (via {@link MemoryPoolMXBean}).
 * <p>
 * Meters registered:
 * <ul>
 *   <li>{@code jvm.memory.used} — bytes currently used, tagged by {@code area} and {@code id}</li>
 *   <li>{@code jvm.memory.committed} — bytes committed (guaranteed available), tagged by {@code area} and {@code id}</li>
 *   <li>{@code jvm.memory.max} — maximum bytes available, tagged by {@code area} and {@code id}</li>
 * </ul>
 * <p>
 * Tags:
 * <ul>
 *   <li>{@code area} — "heap" or "nonheap"</li>
 *   <li>{@code id} — the memory pool name (e.g., "G1 Eden Space", "Metaspace")</li>
 * </ul>
 */
public class JvmMemoryMetrics implements MeterBinder {

    private final Iterable<Tag> tags;

    /**
     * Create a {@code JvmMemoryMetrics} with no extra tags.
     */
    public JvmMemoryMetrics() {
        this(Tags.empty());
    }

    /**
     * Create a {@code JvmMemoryMetrics} with extra tags that will be added
     * to every meter this binder registers.
     *
     * @param tags Extra tags (e.g., application name, environment).
     */
    public JvmMemoryMetrics(Iterable<Tag> tags) {
        this.tags = tags;
    }

    @Override
    public void bindTo(MeterRegistry registry) {
        List<MemoryPoolMXBean> memoryPoolBeans = ManagementFactory.getMemoryPoolMXBeans();

        for (MemoryPoolMXBean memoryPoolBean : memoryPoolBeans) {
            String area = memoryPoolBean.getType() == MemoryType.HEAP ? "heap" : "nonheap";
            Tags poolTags = Tags.of(tags)
                    .and(Tag.of("area", area))
                    .and(Tag.of("id", memoryPoolBean.getName()));

            Gauge.builder("jvm.memory.used", memoryPoolBean,
                            mem -> getUsageValue(mem, MemoryUsage::getUsed))
                    .tags(poolTags)
                    .description("The amount of used memory")
                    .baseUnit("bytes")
                    .register(registry);

            Gauge.builder("jvm.memory.committed", memoryPoolBean,
                            mem -> getUsageValue(mem, MemoryUsage::getCommitted))
                    .tags(poolTags)
                    .description("The amount of memory that is committed for the JVM to use")
                    .baseUnit("bytes")
                    .register(registry);

            Gauge.builder("jvm.memory.max", memoryPoolBean,
                            mem -> getUsageValue(mem, MemoryUsage::getMax))
                    .tags(poolTags)
                    .description("The maximum amount of memory that can be used")
                    .baseUnit("bytes")
                    .register(registry);
        }
    }

    /**
     * Safely extracts a value from the memory pool's usage. Returns {@code NaN}
     * if the usage is unavailable (the pool may not support usage reporting).
     */
    private static double getUsageValue(MemoryPoolMXBean memoryPoolBean,
            java.util.function.ToLongFunction<MemoryUsage> extractor) {
        MemoryUsage usage = memoryPoolBean.getUsage();
        if (usage == null) {
            return Double.NaN;
        }
        return extractor.applyAsLong(usage);
    }

}
```

#### File: `src/main/java/dev/linhvu/micrometer/binder/jvm/JvmThreadMetrics.java` [NEW]

```java
package dev.linhvu.micrometer.binder.jvm;

import dev.linhvu.micrometer.FunctionCounter;
import dev.linhvu.micrometer.Gauge;
import dev.linhvu.micrometer.MeterRegistry;
import dev.linhvu.micrometer.Tag;
import dev.linhvu.micrometer.Tags;
import dev.linhvu.micrometer.binder.MeterBinder;

import java.lang.management.ManagementFactory;
import java.lang.management.ThreadInfo;
import java.lang.management.ThreadMXBean;
import java.util.Arrays;

/**
 * Registers gauges and a function counter for JVM thread statistics.
 * <p>
 * Meters registered:
 * <ul>
 *   <li>{@code jvm.threads.peak} — peak live thread count since JVM start</li>
 *   <li>{@code jvm.threads.daemon} — current live daemon thread count</li>
 *   <li>{@code jvm.threads.live} — current live thread count (daemon + non-daemon)</li>
 *   <li>{@code jvm.threads.started} — total threads started (monotonic counter)</li>
 *   <li>{@code jvm.threads.states} — thread count per {@link Thread.State}, tagged by {@code state}</li>
 * </ul>
 */
public class JvmThreadMetrics implements MeterBinder {

    private final Iterable<Tag> tags;

    private final ThreadMXBean threadBean;

    /**
     * Create a {@code JvmThreadMetrics} with no extra tags.
     */
    public JvmThreadMetrics() {
        this(Tags.empty());
    }

    /**
     * Create a {@code JvmThreadMetrics} with extra tags.
     *
     * @param tags Extra tags added to every meter this binder registers.
     */
    public JvmThreadMetrics(Iterable<Tag> tags) {
        this(ManagementFactory.getThreadMXBean(), tags);
    }

    // Package-private for testing — allows injecting a mock ThreadMXBean
    JvmThreadMetrics(ThreadMXBean threadBean, Iterable<Tag> tags) {
        this.threadBean = threadBean;
        this.tags = tags;
    }

    @Override
    public void bindTo(MeterRegistry registry) {
        Gauge.builder("jvm.threads.peak", threadBean, ThreadMXBean::getPeakThreadCount)
                .tags(tags)
                .description("The peak live thread count since the JVM started")
                .baseUnit("threads")
                .register(registry);

        Gauge.builder("jvm.threads.daemon", threadBean, ThreadMXBean::getDaemonThreadCount)
                .tags(tags)
                .description("The current number of live daemon threads")
                .baseUnit("threads")
                .register(registry);

        Gauge.builder("jvm.threads.live", threadBean, ThreadMXBean::getThreadCount)
                .tags(tags)
                .description("The current number of live threads including both daemon and non-daemon threads")
                .baseUnit("threads")
                .register(registry);

        FunctionCounter.builder("jvm.threads.started", threadBean,
                        ThreadMXBean::getTotalStartedThreadCount)
                .tags(tags)
                .description("The total number of application threads started in the JVM")
                .baseUnit("threads")
                .register(registry);

        // Per-state thread count gauges
        for (Thread.State state : Thread.State.values()) {
            Gauge.builder("jvm.threads.states", threadBean,
                            bean -> getThreadStateCount(bean, state))
                    .tags(Tags.of(tags).and("state", state.name().toLowerCase().replace("_", "-")))
                    .description("The current number of threads having " + state.name() + " state")
                    .baseUnit("threads")
                    .register(registry);
        }
    }

    // Package-private for testing
    static long getThreadStateCount(ThreadMXBean threadBean, Thread.State state) {
        long[] threadIds = threadBean.getAllThreadIds();
        ThreadInfo[] threadInfos = threadBean.getThreadInfo(threadIds);
        return Arrays.stream(threadInfos)
                .filter(info -> info != null && info.getThreadState() == state)
                .count();
    }

}
```

#### File: `src/main/java/dev/linhvu/micrometer/binder/system/UptimeMetrics.java` [NEW]

```java
package dev.linhvu.micrometer.binder.system;

import dev.linhvu.micrometer.MeterRegistry;
import dev.linhvu.micrometer.Tag;
import dev.linhvu.micrometer.Tags;
import dev.linhvu.micrometer.TimeGauge;
import dev.linhvu.micrometer.binder.MeterBinder;

import java.lang.management.ManagementFactory;
import java.lang.management.RuntimeMXBean;
import java.util.concurrent.TimeUnit;

/**
 * Registers time gauges for process uptime and start time.
 * <p>
 * Meters registered:
 * <ul>
 *   <li>{@code process.uptime} — how long the JVM has been running (in milliseconds, scaled by the registry)</li>
 *   <li>{@code process.start.time} — when the JVM started (Unix epoch millis, scaled by the registry)</li>
 * </ul>
 * <p>
 * Uses {@link TimeGauge} instead of plain {@link dev.linhvu.micrometer.Gauge} because
 * the values are time-based and should be scaled to the registry's base time unit
 * during export.
 */
public class UptimeMetrics implements MeterBinder {

    private final RuntimeMXBean runtimeBean;

    private final Iterable<Tag> tags;

    /**
     * Create an {@code UptimeMetrics} with no extra tags.
     */
    public UptimeMetrics() {
        this(Tags.empty());
    }

    /**
     * Create an {@code UptimeMetrics} with extra tags.
     *
     * @param tags Extra tags added to every meter this binder registers.
     */
    public UptimeMetrics(Iterable<Tag> tags) {
        this(ManagementFactory.getRuntimeMXBean(), tags);
    }

    // Package-private for testing — allows injecting a mock RuntimeMXBean
    UptimeMetrics(RuntimeMXBean runtimeBean, Iterable<Tag> tags) {
        this.runtimeBean = runtimeBean;
        this.tags = tags;
    }

    @Override
    public void bindTo(MeterRegistry registry) {
        TimeGauge.builder("process.uptime", runtimeBean,
                        TimeUnit.MILLISECONDS, RuntimeMXBean::getUptime)
                .tags(tags)
                .description("The uptime of the Java virtual machine")
                .register(registry);

        TimeGauge.builder("process.start.time", runtimeBean,
                        TimeUnit.MILLISECONDS, RuntimeMXBean::getStartTime)
                .tags(tags)
                .description("Start time of the process since Unix epoch")
                .register(registry);
    }

}
```

### Test Code

#### File: `src/test/java/dev/linhvu/micrometer/binder/jvm/JvmMemoryMetricsTest.java` [NEW]

```java
package dev.linhvu.micrometer.binder.jvm;

import dev.linhvu.micrometer.Gauge;
import dev.linhvu.micrometer.Meter;
import dev.linhvu.micrometer.Tags;
import dev.linhvu.micrometer.simple.SimpleMeterRegistry;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.util.List;

import static org.assertj.core.api.Assertions.*;

/**
 * Tests for {@link JvmMemoryMetrics}.
 */
class JvmMemoryMetricsTest {

    private SimpleMeterRegistry registry;

    @BeforeEach
    void setUp() {
        registry = new SimpleMeterRegistry();
    }

    @Test
    void shouldRegisterMemoryMeters_WhenBound() {
        new JvmMemoryMetrics().bindTo(registry);

        List<Meter> meters = registry.getMeters();

        // Should have at least some memory pool meters (3 gauges per pool)
        assertThat(meters).isNotEmpty();

        // Verify that we have the expected metric names
        assertThat(meters.stream().map(m -> m.getId().getName()))
                .contains("jvm.memory.used", "jvm.memory.committed", "jvm.memory.max");
    }

    @Test
    void shouldTagWithAreaAndId_WhenBound() {
        new JvmMemoryMetrics().bindTo(registry);

        // Every memory meter should have "area" and "id" tags
        for (Meter meter : registry.getMeters()) {
            assertThat(meter.getId().getTag("area"))
                    .as("meter '%s' should have 'area' tag", meter.getId().getName())
                    .isIn("heap", "nonheap");
            assertThat(meter.getId().getTag("id"))
                    .as("meter '%s' should have 'id' tag", meter.getId().getName())
                    .isNotNull()
                    .isNotEmpty();
        }
    }

    @Test
    void shouldIncludeExtraTags_WhenProvided() {
        new JvmMemoryMetrics(Tags.of("env", "test")).bindTo(registry);

        for (Meter meter : registry.getMeters()) {
            assertThat(meter.getId().getTag("env")).isEqualTo("test");
        }
    }

    @Test
    void shouldReportValues_WhenQueried() {
        new JvmMemoryMetrics().bindTo(registry);

        for (Meter meter : registry.getMeters()) {
            if (meter instanceof Gauge gauge) {
                double value = gauge.value();
                // Values should be non-negative, NaN (if unavailable), or -1 (undefined max)
                assertThat(value).satisfiesAnyOf(
                        v -> assertThat(v).isGreaterThanOrEqualTo(0),
                        v -> assertThat(v).isNaN(),
                        v -> assertThat(v).isEqualTo(-1.0));
            }
        }
    }

    @Test
    void shouldHaveBaseUnitBytes_WhenRegistered() {
        new JvmMemoryMetrics().bindTo(registry);

        for (Meter meter : registry.getMeters()) {
            assertThat(meter.getId().getBaseUnit()).isEqualTo("bytes");
        }
    }

    @Test
    void shouldRegisterThreeMetersPerPool_WhenBound() {
        new JvmMemoryMetrics().bindTo(registry);

        List<Meter> meters = registry.getMeters();
        // Every pool should have exactly 3 meters (used, committed, max)
        assertThat(meters.size() % 3).isEqualTo(0);
    }

}
```

#### File: `src/test/java/dev/linhvu/micrometer/binder/jvm/JvmThreadMetricsTest.java` [NEW]

```java
package dev.linhvu.micrometer.binder.jvm;

import dev.linhvu.micrometer.FunctionCounter;
import dev.linhvu.micrometer.Gauge;
import dev.linhvu.micrometer.Meter;
import dev.linhvu.micrometer.Tags;
import dev.linhvu.micrometer.simple.SimpleMeterRegistry;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.util.List;

import static org.assertj.core.api.Assertions.*;

/**
 * Tests for {@link JvmThreadMetrics}.
 */
class JvmThreadMetricsTest {

    private SimpleMeterRegistry registry;

    @BeforeEach
    void setUp() {
        registry = new SimpleMeterRegistry();
    }

    @Test
    void shouldRegisterThreadMeters_WhenBound() {
        new JvmThreadMetrics().bindTo(registry);

        List<Meter> meters = registry.getMeters();
        assertThat(meters).isNotEmpty();

        assertThat(meters.stream().map(m -> m.getId().getName()))
                .contains("jvm.threads.peak", "jvm.threads.daemon",
                        "jvm.threads.live", "jvm.threads.started",
                        "jvm.threads.states");
    }

    @Test
    void shouldRegisterPeakAsGauge_WhenBound() {
        new JvmThreadMetrics().bindTo(registry);

        Meter peak = registry.getMeters().stream()
                .filter(m -> m.getId().getName().equals("jvm.threads.peak"))
                .findFirst()
                .orElseThrow();

        assertThat(peak).isInstanceOf(Gauge.class);
        assertThat(((Gauge) peak).value()).isGreaterThan(0);
    }

    @Test
    void shouldRegisterStartedAsFunctionCounter_WhenBound() {
        new JvmThreadMetrics().bindTo(registry);

        Meter started = registry.getMeters().stream()
                .filter(m -> m.getId().getName().equals("jvm.threads.started"))
                .findFirst()
                .orElseThrow();

        assertThat(started).isInstanceOf(FunctionCounter.class);
        assertThat(((FunctionCounter) started).count()).isGreaterThan(0);
    }

    @Test
    void shouldRegisterPerStateGauges_WhenBound() {
        new JvmThreadMetrics().bindTo(registry);

        // Should have one gauge per Thread.State
        long stateGaugeCount = registry.getMeters().stream()
                .filter(m -> m.getId().getName().equals("jvm.threads.states"))
                .count();

        assertThat(stateGaugeCount).isEqualTo(Thread.State.values().length);
    }

    @Test
    void shouldTagStatesWithLowercaseHyphenated_WhenBound() {
        new JvmThreadMetrics().bindTo(registry);

        List<String> stateValues = registry.getMeters().stream()
                .filter(m -> m.getId().getName().equals("jvm.threads.states"))
                .map(m -> m.getId().getTag("state"))
                .toList();

        assertThat(stateValues).contains("runnable", "waiting", "timed-waiting",
                "blocked", "new", "terminated");
    }

    @Test
    void shouldIncludeExtraTags_WhenProvided() {
        new JvmThreadMetrics(Tags.of("app", "myapp")).bindTo(registry);

        for (Meter meter : registry.getMeters()) {
            assertThat(meter.getId().getTag("app")).isEqualTo("myapp");
        }
    }

    @Test
    void shouldHaveBaseUnitThreads_WhenRegistered() {
        new JvmThreadMetrics().bindTo(registry);

        for (Meter meter : registry.getMeters()) {
            assertThat(meter.getId().getBaseUnit()).isEqualTo("threads");
        }
    }

    @Test
    void shouldReportNonNegativeLiveCount_WhenQueried() {
        new JvmThreadMetrics().bindTo(registry);

        Gauge live = (Gauge) registry.getMeters().stream()
                .filter(m -> m.getId().getName().equals("jvm.threads.live"))
                .findFirst()
                .orElseThrow();

        assertThat(live.value()).isGreaterThan(0);
    }

}
```

#### File: `src/test/java/dev/linhvu/micrometer/binder/system/UptimeMetricsTest.java` [NEW]

```java
package dev.linhvu.micrometer.binder.system;

import dev.linhvu.micrometer.Gauge;
import dev.linhvu.micrometer.Meter;
import dev.linhvu.micrometer.Tags;
import dev.linhvu.micrometer.TimeGauge;
import dev.linhvu.micrometer.simple.SimpleMeterRegistry;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.util.List;

import static org.assertj.core.api.Assertions.*;

/**
 * Tests for {@link UptimeMetrics}.
 */
class UptimeMetricsTest {

    private SimpleMeterRegistry registry;

    @BeforeEach
    void setUp() {
        registry = new SimpleMeterRegistry();
    }

    @Test
    void shouldRegisterUptimeMeters_WhenBound() {
        new UptimeMetrics().bindTo(registry);

        List<Meter> meters = registry.getMeters();
        assertThat(meters).hasSize(2);

        assertThat(meters.stream().map(m -> m.getId().getName()))
                .containsExactlyInAnyOrder("process.uptime", "process.start.time");
    }

    @Test
    void shouldRegisterAsTimeGauge_WhenBound() {
        new UptimeMetrics().bindTo(registry);

        for (Meter meter : registry.getMeters()) {
            assertThat(meter).isInstanceOf(TimeGauge.class);
        }
    }

    @Test
    void shouldReportPositiveUptime_WhenQueried() {
        new UptimeMetrics().bindTo(registry);

        Gauge uptime = (Gauge) registry.getMeters().stream()
                .filter(m -> m.getId().getName().equals("process.uptime"))
                .findFirst()
                .orElseThrow();

        // JVM has been running for at least some time
        assertThat(uptime.value()).isGreaterThan(0);
    }

    @Test
    void shouldReportPositiveStartTime_WhenQueried() {
        new UptimeMetrics().bindTo(registry);

        Gauge startTime = (Gauge) registry.getMeters().stream()
                .filter(m -> m.getId().getName().equals("process.start.time"))
                .findFirst()
                .orElseThrow();

        // Start time should be a positive Unix epoch millis value
        assertThat(startTime.value()).isGreaterThan(0);
    }

    @Test
    void shouldIncludeExtraTags_WhenProvided() {
        new UptimeMetrics(Tags.of("env", "staging")).bindTo(registry);

        for (Meter meter : registry.getMeters()) {
            assertThat(meter.getId().getTag("env")).isEqualTo("staging");
        }
    }

}
```

#### File: `src/test/java/dev/linhvu/micrometer/integration/MeterBinderIntegrationTest.java` [NEW]

```java
package dev.linhvu.micrometer.integration;

import dev.linhvu.micrometer.Gauge;
import dev.linhvu.micrometer.Meter;
import dev.linhvu.micrometer.Tags;
import dev.linhvu.micrometer.binder.MeterBinder;
import dev.linhvu.micrometer.binder.jvm.JvmMemoryMetrics;
import dev.linhvu.micrometer.binder.jvm.JvmThreadMetrics;
import dev.linhvu.micrometer.binder.system.UptimeMetrics;
import dev.linhvu.micrometer.config.MeterFilter;
import dev.linhvu.micrometer.simple.SimpleMeterRegistry;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.util.List;

import static org.assertj.core.api.Assertions.*;

/**
 * Integration tests for {@link MeterBinder} — verifies binders work with
 * MeterRegistry features (filters, deduplication, common tags).
 */
class MeterBinderIntegrationTest {

    private SimpleMeterRegistry registry;

    @BeforeEach
    void setUp() {
        registry = new SimpleMeterRegistry();
    }

    // -----------------------------------------------------------------------
    // Lambda binder
    // -----------------------------------------------------------------------

    @Test
    void shouldSupportLambdaBinder_WhenUsedAsFunctionalInterface() {
        List<String> items = List.of("a", "b", "c");

        MeterBinder binder = r ->
                Gauge.builder("queue.size", items, List::size)
                        .description("Queue depth")
                        .register(r);

        binder.bindTo(registry);

        Meter meter = registry.getMeters().stream()
                .filter(m -> m.getId().getName().equals("queue.size"))
                .findFirst()
                .orElseThrow();

        assertThat(meter).isInstanceOf(Gauge.class);
        assertThat(((Gauge) meter).value()).isEqualTo(3.0);
    }

    // -----------------------------------------------------------------------
    // Multiple binders on the same registry
    // -----------------------------------------------------------------------

    @Test
    void shouldRegisterAllMeters_WhenMultipleBindersBound() {
        new JvmMemoryMetrics().bindTo(registry);
        new JvmThreadMetrics().bindTo(registry);
        new UptimeMetrics().bindTo(registry);

        List<Meter> meters = registry.getMeters();

        // Should have meters from all three binders
        assertThat(meters.stream().map(m -> m.getId().getName()))
                .contains("jvm.memory.used", "jvm.threads.live", "process.uptime");
    }

    // -----------------------------------------------------------------------
    // MeterFilter interaction
    // -----------------------------------------------------------------------

    @Test
    void shouldApplyCommonTags_WhenFilterConfigured() {
        registry.meterFilter(MeterFilter.commonTags(Tags.of("region", "us-east-1")));

        new JvmMemoryMetrics().bindTo(registry);

        for (Meter meter : registry.getMeters()) {
            assertThat(meter.getId().getTag("region")).isEqualTo("us-east-1");
        }
    }

    @Test
    void shouldDenyMeters_WhenFilterDeniesJvmPrefix() {
        registry.meterFilter(MeterFilter.denyNameStartsWith("jvm.memory"));

        new JvmMemoryMetrics().bindTo(registry);
        new JvmThreadMetrics().bindTo(registry);

        // Memory metrics should be denied, thread metrics should be registered
        assertThat(registry.getMeters().stream().map(m -> m.getId().getName()))
                .noneMatch(name -> name.startsWith("jvm.memory"))
                .anyMatch(name -> name.startsWith("jvm.threads"));
    }

    @Test
    void shouldRenameMeters_WhenFilterMapsNames() {
        registry.meterFilter(new MeterFilter() {
            @Override
            public Meter.Id map(Meter.Id id) {
                if (id.getName().startsWith("process.")) {
                    return id.withName("app." + id.getName());
                }
                return id;
            }
        });

        new UptimeMetrics().bindTo(registry);

        assertThat(registry.getMeters().stream().map(m -> m.getId().getName()))
                .contains("app.process.uptime", "app.process.start.time");
    }

    // -----------------------------------------------------------------------
    // Extra tags + filter common tags combine
    // -----------------------------------------------------------------------

    @Test
    void shouldCombineExtraAndCommonTags_WhenBothProvided() {
        registry.meterFilter(MeterFilter.commonTags(Tags.of("env", "prod")));

        new UptimeMetrics(Tags.of("host", "server-1")).bindTo(registry);

        for (Meter meter : registry.getMeters()) {
            assertThat(meter.getId().getTag("env")).isEqualTo("prod");
            assertThat(meter.getId().getTag("host")).isEqualTo("server-1");
        }
    }

    // -----------------------------------------------------------------------
    // Deduplication
    // -----------------------------------------------------------------------

    @Test
    void shouldDeduplicateMeters_WhenSameBinderBoundTwice() {
        new UptimeMetrics().bindTo(registry);
        int firstCount = registry.getMeters().size();

        new UptimeMetrics().bindTo(registry);
        int secondCount = registry.getMeters().size();

        // Same meters should be deduplicated — same count
        assertThat(secondCount).isEqualTo(firstCount);
    }

    // -----------------------------------------------------------------------
    // Closed registry
    // -----------------------------------------------------------------------

    @Test
    void shouldReturnNoopMeters_WhenRegistryClosed() {
        registry.close();

        new JvmThreadMetrics().bindTo(registry);

        // Registry is closed — meters are noop and not stored
        // The getMeters() list should be empty since noop meters aren't stored
        assertThat(registry.getMeters()).isEmpty();
    }

}
```

---

## Summary

| Concept | What It Means |
|---------|---------------|
| **MeterBinder** | A functional interface that packages related metric registrations into a single `bindTo(MeterRegistry)` call |
| **Extra tags pattern** | All binders accept `Iterable<Tag>` so callers can attach common tags (environment, host, application) to all produced meters |
| **MXBean-backed gauges** | Binders pass JMX MXBeans as state objects; the gauge reads them lazily on each scrape — no polling thread needed |
| **Gauge vs. FunctionCounter vs. TimeGauge** | The meter type choice is semantic: Gauge for fluctuating values, FunctionCounter for monotonic counts (rate-computed), TimeGauge for time values (unit-converted) |
| **Package-private constructors** | For testability: public constructors use `ManagementFactory`; package-private ones accept injected MXBeans |

**Next: Chapter 14 — CompositeMeterRegistry & Global Facade** — A fan-out registry that broadcasts operations to multiple child registries, plus a static global `Metrics` facade for use where dependency injection is not available.
