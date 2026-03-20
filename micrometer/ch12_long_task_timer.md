# Chapter 12: LongTaskTimer

> Line references based on commit `2c8a4606c` of the Micrometer repository.

## Build Challenge

| Current State | Limitation | Objective |
|---------------|------------|-----------|
| The registry supports Counter, Gauge, Timer, DistributionSummary, FunctionCounter, FunctionTimer, and TimeGauge — Timer records duration *after* a task completes | Cannot monitor long-running operations *while they are still executing* — a regular Timer only tells you about the past, never the present. If a batch import is stuck for 2 hours, Timer reports nothing until it finishes (or never, if it hangs) | Build a `LongTaskTimer` that tracks currently active tasks using a `ConcurrentSkipListSet`, reporting live `activeTasks()`, `duration()`, and `max()` from an in-flight task set |

---

## 12.1 The Integration Point: MeterRegistry.More gains `longTaskTimer()`

**Modifying:** `src/main/java/dev/linhvu/micrometer/MeterRegistry.java`
**Change:** Add `newLongTaskTimer()` abstract factory method and `longTaskTimer()` registration in the `More` inner class.

```java
// --- New abstract factory method (alongside existing newTimer, newDistributionSummary, etc.) ---

protected abstract LongTaskTimer newLongTaskTimer(Meter.Id id,
        DistributionStatisticConfig distributionStatisticConfig);

// --- In the More inner class, add: ---

LongTaskTimer longTaskTimer(Meter.Id id, DistributionStatisticConfig distributionStatisticConfig) {
    return registerMeterIfNecessary(LongTaskTimer.class, id,
            mappedId -> newLongTaskTimer(mappedId,
                    configureDistributionStatistics(mappedId, distributionStatisticConfig)),
            NoopLongTaskTimer::new);
}
```

Two key decisions here:

1. **Registration goes through `More`, not the top-level registry.** LongTaskTimer is a specialized meter — most applications use Counter/Timer/Gauge. Placing it in `More` keeps the main API surface clean while still giving the Builder a clean registration path: `Builder.register()` calls `registry.more().longTaskTimer(id, config)`.

2. **The distribution config pipeline is reused.** Even though our simplified LongTaskTimer doesn't use histogram buckets, the `configureDistributionStatistics()` pipeline is invoked — this means MeterFilters can still influence the timer's config, and the architecture is consistent with Timer and DistributionSummary.

This connects **LongTaskTimer.Builder** to **Registry Lifecycle Management** (deduplication, filtering, factory dispatch). To make it work, we need to build:
- `LongTaskTimer` interface — with `start()`, `duration()`, `activeTasks()`, `max()`, `Sample`, and a `Builder`
- `DefaultLongTaskTimer` — the `ConcurrentSkipListSet`-based implementation
- `NoopLongTaskTimer` — the no-op fallback for denied/closed registrations

## 12.2 The LongTaskTimer Interface

**Modifying:** `src/main/java/dev/linhvu/micrometer/LongTaskTimer.java`
**Change:** Replace the stub interface with the full LongTaskTimer API.

The interface defines three categories of methods:

**Core query methods** — what makes LongTaskTimer unique:
```java
Sample start();                    // Begin timing a task
double duration(TimeUnit unit);    // Cumulative duration of ALL active tasks
int activeTasks();                 // How many tasks are running right now
double max(TimeUnit unit);         // Duration of the longest-running task
TimeUnit baseTimeUnit();           // The time unit used for export
HistogramSnapshot takeSnapshot();  // Point-in-time snapshot for publishing
```

**The `measure()` default** — emits three measurements:
```java
@Override
default Iterable<Measurement> measure() {
    return Arrays.asList(
            new Measurement(() -> (double) activeTasks(), Statistic.ACTIVE_TASKS),
            new Measurement(() -> duration(baseTimeUnit()), Statistic.DURATION),
            new Measurement(() -> max(baseTimeUnit()), Statistic.MAX));
}
```

Compare this with Timer's measurements (COUNT, TOTAL_TIME, MAX). LongTaskTimer replaces COUNT with ACTIVE_TASKS and TOTAL_TIME with DURATION — reflecting that it measures the *present*, not accumulated history.

**Convenience methods** — `record(Runnable)`, `record(Supplier)`, `recordCallable(Callable)`, `record(Consumer<Sample>)`. All follow the same pattern: `start()` in try, execute, `stop()` in finally.

**The `Sample` abstract class:**
```java
abstract class Sample {
    public abstract long stop();
    public abstract double duration(TimeUnit unit);
}
```

`Sample` is the handle to an individual active task. Unlike Timer.Sample (which just records a start time), LongTaskTimer.Sample has a live `duration()` method — you can query how long the task has been running while it's still active.

**The `Builder`:**
```java
public LongTaskTimer register(MeterRegistry registry) {
    return registry.more()
            .longTaskTimer(
                    new Meter.Id(name, tags, Meter.Type.LONG_TASK_TIMER, description, null),
                    distributionConfigBuilder.build());
}
```

The default expected range is **2 minutes to 2 hours** — reflecting that LongTaskTimer is for genuinely long operations, not sub-second request handling (that's what Timer is for).

## 12.3 DefaultLongTaskTimer — The ConcurrentSkipListSet Implementation

**New file:** `src/main/java/dev/linhvu/micrometer/internal/DefaultLongTaskTimer.java`

The heart of the implementation is a `ConcurrentSkipListSet<SampleImpl>`:

```java
private final NavigableSet<SampleImpl> activeTasks = new ConcurrentSkipListSet<>();
```

**Why ConcurrentSkipListSet over other collections?**

| Operation | ConcurrentSkipListSet | ConcurrentLinkedQueue | ArrayList + synchronized |
|-----------|----------------------|----------------------|-------------------------|
| `start()` (insert) | O(log N) | O(1) | O(1) amortized |
| `stop()` (remove from middle) | O(log N) | O(N) | O(N) |
| `activeTasks()` (size) | O(N)* | O(N)* | O(1) |
| `duration()` (iterate) | O(N) | O(N) | O(N) |

\* `ConcurrentSkipListSet.size()` is O(N) but this is fine — it runs on the publishing thread, not the application thread.

Tasks complete out of order. A batch job that started first might finish last. This means removal happens from arbitrary positions in the collection, not just the head. Skip list's O(log N) removal beats a queue's O(N) scan.

**Sample ordering and tiebreaking:**

```java
@Override
public Sample start() {
    long startTime = clock.monotonicTime();
    SampleImpl sample = new SampleImpl(startTime);
    if (!activeTasks.add(sample)) {
        // Two tasks started at the exact same nanosecond — use counter as tiebreaker
        sample = new SampleImplCounted(startTime, nextNonZeroCounter());
        activeTasks.add(sample);
    }
    return sample;
}
```

Samples compare by start time, with a counter-based tiebreaker when two tasks start at the same nanosecond. `SampleImpl` has `counter() = 0`; `SampleImplCounted` gets a unique non-zero counter. This ensures the `ConcurrentSkipListSet` (which is a `Set` — no duplicates) can hold multiple simultaneous tasks.

**`max()` leverages sort order:**

```java
@Override
public double max(TimeUnit unit) {
    try {
        return activeTasks.first().duration(unit);
    } catch (NoSuchElementException e) {
        return 0.0;
    }
}
```

Since samples are sorted by start time, `first()` is the *oldest* — and the oldest task has the longest duration. This is O(1) instead of scanning all tasks.

## 12.4 NoopLongTaskTimer

**New file:** `src/main/java/dev/linhvu/micrometer/noop/NoopLongTaskTimer.java`

```java
public class NoopLongTaskTimer extends NoopMeter implements LongTaskTimer {
    // start() returns a NoopSample, all queries return 0
}
```

Returned when a MeterFilter denies registration or the registry is closed. The `record()` convenience methods still *execute* the supplied function (application logic must not be affected by metering being disabled) — they just don't record anything.

## 12.5 SimpleMeterRegistry Wiring

**Modifying:** `src/main/java/dev/linhvu/micrometer/simple/SimpleMeterRegistry.java`
**Change:** Add `newLongTaskTimer()` factory method.

```java
@Override
protected LongTaskTimer newLongTaskTimer(Meter.Id id,
        DistributionStatisticConfig distributionStatisticConfig) {
    return new DefaultLongTaskTimer(id, clock, getBaseTimeUnit());
}
```

## 12.6 Try It Yourself

<details>
<summary>Challenge 1: Implement the Sample.stop() method</summary>

Given a `ConcurrentSkipListSet<SampleImpl> activeTasks` and a `Clock clock`, implement `stop()` so that:
1. The sample is removed from the active set
2. The duration in nanoseconds is captured *before* marking as stopped
3. The `stopped` flag prevents further duration queries from returning live values

```java
@Override
public long stop() {
    activeTasks.remove(this);
    long duration = (long) duration(TimeUnit.NANOSECONDS);
    stopped = true;
    return duration;
}
```

Note the ordering: remove first, then capture duration, then set stopped. If we set `stopped = true` first, `duration()` would return -1 and we'd lose the final measurement.

</details>

<details>
<summary>Challenge 2: Why does max() use first() instead of last()?</summary>

Think about the sort order. Samples are sorted by start time (ascending). The *first* element started earliest — it's been running the longest. So `first().duration()` gives the maximum duration.

If we used `last()`, we'd get the *newest* task, which has the *shortest* duration — the opposite of what we want.

```java
@Override
public double max(TimeUnit unit) {
    try {
        return activeTasks.first().duration(unit);
    } catch (NoSuchElementException e) {
        return 0.0;
    }
}
```

</details>

## 12.7 Tests

### Unit Tests

**New file:** `src/test/java/dev/linhvu/micrometer/LongTaskTimerTest.java`

Tests the interface layer: builder, convenience methods, default `measure()`, and `mean()`.

```java
@Test
void shouldProduceThreeMeasurements_WhenMeasureCalled() {
    // Verifies ACTIVE_TASKS, DURATION, MAX statistics
}

@Test
void shouldStopSample_WhenRunnableThrows() {
    // Verifies try/finally ensures cleanup even on exception
}
```

**New file:** `src/test/java/dev/linhvu/micrometer/internal/DefaultLongTaskTimerTest.java`

Tests the implementation: start/stop lifecycle, duration computation, max, mean, out-of-order stop, same-nanosecond tiebreaking, snapshots.

```java
@Test
void shouldTrackTasksIndependently_WhenStoppedOutOfOrder() {
    LongTaskTimer.Sample first = timer.start();
    clock.addSeconds(5);
    LongTaskTimer.Sample second = timer.start();
    clock.addSeconds(5);

    first.stop(); // Out of order — first started, first stopped

    assertThat(timer.activeTasks()).isEqualTo(1);
    assertThat(timer.duration(TimeUnit.SECONDS)).isEqualTo(5.0);
}

@Test
void shouldHandleSameStartTime_WhenTwoTasksStartSimultaneously() {
    LongTaskTimer.Sample first = timer.start();
    LongTaskTimer.Sample second = timer.start(); // Same nanosecond

    assertThat(timer.activeTasks()).isEqualTo(2); // Both tracked
}
```

**New file:** `src/test/java/dev/linhvu/micrometer/noop/NoopLongTaskTimerTest.java`

Verifies the no-op implementation returns zeros and doesn't throw.

### Integration Tests

**New file:** `src/test/java/dev/linhvu/micrometer/integration/LongTaskTimerIntegrationTest.java`

Tests registry registration, deduplication, MeterFilter integration, visitor pattern dispatch, and end-to-end usage.

```java
@Test
void shouldTrackLongRunningTask_WhenUsedEndToEnd() {
    LongTaskTimer ltt = LongTaskTimer.builder("batch.import")
            .tag("source", "csv").register(registry);

    LongTaskTimer.Sample sample = ltt.start();
    clock.addSeconds(30);

    assertThat(ltt.activeTasks()).isEqualTo(1);
    assertThat(ltt.duration(TimeUnit.SECONDS)).isEqualTo(30.0);

    LongTaskTimer.Sample sample2 = ltt.start();
    clock.addSeconds(10);

    assertThat(ltt.activeTasks()).isEqualTo(2);
    assertThat(ltt.duration(TimeUnit.SECONDS)).isEqualTo(50.0); // 40 + 10

    sample.stop();
    assertThat(ltt.duration(TimeUnit.SECONDS)).isEqualTo(10.0);
}
```

**Run:** `./gradlew test` — expected: all tests pass (including all prior features' tests)

---

## 12.8 Why This Works

> ★ **Insight** -------------------------------------------
> **Live set vs. accumulators:** Timer uses `LongAdder`/`DoubleAdder` for lock-free accumulation of *completed* events. LongTaskTimer uses a `ConcurrentSkipListSet` of *active* tasks. This is a fundamental data structure choice driven by the question being asked: "what happened?" (accumulator) vs. "what is happening?" (live set). You cannot answer "how many tasks are running right now?" with an adder — you need the actual set of in-flight samples.
>
> **Trade-off:** The live set approach means every `start()` allocates an object and every `stop()` removes it. For high-throughput short operations (thousands per second), this is wasteful — use Timer instead. LongTaskTimer is designed for low-throughput, high-duration operations where the allocation cost is negligible compared to the operation duration.
> -----------------------------------------------------------

> ★ **Insight** -------------------------------------------
> **ConcurrentSkipListSet ordering enables O(1) max:** By sorting samples by start time, the oldest (longest-running) task is always `first()`. This gives us `max()` in O(1) without scanning. This is a real design choice from Micrometer — the comment in `DefaultLongTaskTimer.java:30-44` explains the rationale explicitly. The alternative (`ConcurrentLinkedQueue`) would give O(1) insertion but O(N) removal and O(N) max computation.
> -----------------------------------------------------------

> ★ **Insight** -------------------------------------------
> **The Sample as a live handle:** Timer.Sample is a one-shot recording mechanism — `sample.stop()` records the duration and the sample is discarded. LongTaskTimer.Sample is a *live handle* — while the task is active, you can call `sample.duration(TimeUnit.SECONDS)` to check how long it's been running. This is what enables the `record(Consumer<Sample>)` convenience method, where the running code can introspect its own timing.
> -----------------------------------------------------------

## 12.9 What We Enhanced

| Aspect | Before (ch11) | Current (ch12) | Real Framework |
|--------|---------------|----------------|----------------|
| In-flight monitoring | Not possible — Timer only records completed events | `LongTaskTimer` tracks active tasks with `activeTasks()`, `duration()`, `max()` | Same — `LongTaskTimer.java:1-50` |
| Active task data structure | N/A | `ConcurrentSkipListSet` for O(log N) insert/remove | Same — `DefaultLongTaskTimer.java:44` with detailed rationale comment |
| `More` inner class | Provides `FunctionCounter`, `FunctionTimer`, `TimeGauge` | Also provides `longTaskTimer()` registration | Same pattern — `MeterRegistry.java` routes through `More` |
| `measure()` statistics | Timer emits COUNT, TOTAL_TIME, MAX | LongTaskTimer emits ACTIVE_TASKS, DURATION, MAX | Same — `LongTaskTimer.java:206-210` |
| Snapshot support | N/A for LongTaskTimer | Basic `takeSnapshot()` with count/total/max | Real version includes percentile interpolation and histogram bucket computation — `DefaultLongTaskTimer.java:130-210` |

## 12.10 Connection to Real Framework

| Simplified Code | Real Framework Code | File:Line | Key Difference |
|----------------|--------------------|-----------| ---------------|
| `LongTaskTimer` | `LongTaskTimer` | `LongTaskTimer.java:38` | Real version implements `HistogramSupport`, has `@Timed` builder, `MeterProvider`, and primitive supplier overloads (`BooleanSupplier`, `IntSupplier`, etc.) |
| `LongTaskTimer.Sample` | `LongTaskTimer.Sample` | `LongTaskTimer.java:186-198` | Same abstract class with `stop()` and `duration()` |
| `LongTaskTimer.Builder` | `LongTaskTimer.Builder` | `LongTaskTimer.java:206-340` | Real builder has `publishPercentileHistogram()`, `publishPercentiles()`, `percentilePrecision()`, `MeterProvider` support |
| `DefaultLongTaskTimer` | `DefaultLongTaskTimer` | `DefaultLongTaskTimer.java:34` | Real version has full `takeSnapshot()` with percentile interpolation across active tasks, histogram bucket computation, and `forEachActive()` for subclass registries |
| `DefaultLongTaskTimer.SampleImpl` | `DefaultLongTaskTimer.SampleImpl` | `DefaultLongTaskTimer.java:157` | Same `Comparable` with start-time ordering and counter tiebreaker |
| `DefaultLongTaskTimer.SampleImplCounted` | `DefaultLongTaskTimer.SampleImplCounted` | `DefaultLongTaskTimer.java:197` | Same pattern — extends `SampleImpl` with non-zero counter |
| `NoopLongTaskTimer` | `NoopLongTaskTimer` | `NoopLongTaskTimer.java:22` | Same structure — returns `NoopSample` and zeros |

## 12.11 Complete Code

### Production Code

#### File: `src/main/java/dev/linhvu/micrometer/LongTaskTimer.java` [MODIFIED]

```java
package dev.linhvu.micrometer;

import dev.linhvu.micrometer.distribution.DistributionStatisticConfig;
import dev.linhvu.micrometer.distribution.HistogramSnapshot;

import java.time.Duration;
import java.util.Arrays;
import java.util.concurrent.Callable;
import java.util.concurrent.TimeUnit;
import java.util.function.Consumer;
import java.util.function.Supplier;

/**
 * Tracks the duration of long-running tasks that are still in progress,
 * reporting active task count and cumulative duration.
 * <p>
 * Unlike a regular {@link Timer} which records completed events, a LongTaskTimer
 * lets you ask "how many tasks are running right now, and how long have they been
 * going?" This is critical for monitoring batch jobs, migrations, or any operation
 * where you need visibility <i>during</i> execution, not just after.
 * <p>
 * Usage:
 * <pre>{@code
 * LongTaskTimer ltt = LongTaskTimer.builder("batch.import")
 *     .tag("source", "csv")
 *     .register(registry);
 *
 * // Option 1: wrap a block
 * ltt.record(() -> importData());
 *
 * // Option 2: manual start/stop
 * LongTaskTimer.Sample sample = ltt.start();
 * try {
 *     importData();
 * } finally {
 *     sample.stop();
 * }
 *
 * // Query active tasks from another thread
 * ltt.activeTasks();           // 1
 * ltt.duration(TimeUnit.SECONDS); // how long it's been running
 * }</pre>
 */
public interface LongTaskTimer extends Meter {

    /**
     * Create a builder for a long task timer.
     */
    static Builder builder(String name) {
        return new Builder(name);
    }

    /**
     * Start keeping time for a task.
     *
     * @return A sample that can be used to stop the task and query its duration.
     */
    Sample start();

    /**
     * @param unit The time unit to scale the duration to.
     * @return The cumulative duration of all current tasks.
     */
    double duration(TimeUnit unit);

    /**
     * @return The current number of tasks being executed.
     */
    int activeTasks();

    /**
     * @param unit The base unit of time to scale the mean to.
     * @return The distribution average for all active tasks.
     */
    default double mean(TimeUnit unit) {
        int activeTasks = activeTasks();
        return activeTasks == 0 ? 0 : duration(unit) / activeTasks;
    }

    /**
     * The amount of time the longest running task has been running.
     *
     * @param unit The time unit to scale the max to.
     * @return The maximum active task duration.
     */
    double max(TimeUnit unit);

    /**
     * @return The base time unit of this long task timer.
     */
    TimeUnit baseTimeUnit();

    /**
     * Takes a snapshot of the current histogram data. For LongTaskTimer, this
     * reports active task count, cumulative duration, and max duration.
     */
    HistogramSnapshot takeSnapshot();

    @Override
    default Iterable<Measurement> measure() {
        return Arrays.asList(
                new Measurement(() -> (double) activeTasks(), Statistic.ACTIVE_TASKS),
                new Measurement(() -> duration(baseTimeUnit()), Statistic.DURATION),
                new Measurement(() -> max(baseTimeUnit()), Statistic.MAX));
    }

    // -----------------------------------------------------------------------
    // Convenience recording methods
    // -----------------------------------------------------------------------

    /**
     * Executes the callable {@code f} and records the time taken.
     */
    default <T> T recordCallable(Callable<T> f) throws Exception {
        Sample sample = start();
        try {
            return f.call();
        } finally {
            sample.stop();
        }
    }

    /**
     * Executes the supplier {@code f} and records the time taken.
     */
    default <T> T record(Supplier<T> f) {
        Sample sample = start();
        try {
            return f.get();
        } finally {
            sample.stop();
        }
    }

    /**
     * Executes the runnable {@code f} and records the time taken, passing the
     * sample to the consumer for in-flight duration queries.
     */
    default void record(Consumer<Sample> f) {
        Sample sample = start();
        try {
            f.accept(sample);
        } finally {
            sample.stop();
        }
    }

    /**
     * Executes the runnable {@code f} and records the time taken.
     */
    default void record(Runnable f) {
        Sample sample = start();
        try {
            f.run();
        } finally {
            sample.stop();
        }
    }

    // -----------------------------------------------------------------------
    // Sample — represents one active task
    // -----------------------------------------------------------------------

    /**
     * Represents a single active task being timed by a {@link LongTaskTimer}.
     * Created by {@link #start()} and completed by {@link #stop()}.
     */
    abstract class Sample {

        /**
         * Records the duration of the operation and removes the task from the
         * active set.
         *
         * @return The duration, in nanoseconds, of this sample.
         */
        public abstract long stop();

        /**
         * @param unit time unit to which the return value will be scaled.
         * @return duration of this sample so far (or -1 if already stopped).
         */
        public abstract double duration(TimeUnit unit);

    }

    // -----------------------------------------------------------------------
    // Builder
    // -----------------------------------------------------------------------

    /**
     * Fluent builder for long task timers.
     * <p>
     * Default expected range is 2 minutes to 2 hours — reflecting truly
     * long-running operations.
     */
    class Builder {

        private static final Duration DEFAULT_MINIMUM_EXPECTED_DURATION = Duration.ofMinutes(2);

        private static final Duration DEFAULT_MAXIMUM_EXPECTED_DURATION = Duration.ofHours(2);

        static final DistributionStatisticConfig DEFAULT_DISTRIBUTION_CONFIG = DistributionStatisticConfig
                .builder()
                .minimumExpectedValue((double) DEFAULT_MINIMUM_EXPECTED_DURATION.toNanos())
                .maximumExpectedValue((double) DEFAULT_MAXIMUM_EXPECTED_DURATION.toNanos())
                .build();

        private final String name;

        private Tags tags = Tags.empty();

        private final DistributionStatisticConfig.Builder distributionConfigBuilder =
                DistributionStatisticConfig.builder();

        private String description;

        private Builder(String name) {
            this.name = name;
            minimumExpectedValue(DEFAULT_MINIMUM_EXPECTED_DURATION);
            maximumExpectedValue(DEFAULT_MAXIMUM_EXPECTED_DURATION);
        }

        public Builder tags(String... tags) {
            return tags(Tags.of(tags));
        }

        public Builder tags(Iterable<Tag> tags) {
            this.tags = this.tags.and(tags);
            return this;
        }

        public Builder tag(String key, String value) {
            this.tags = tags.and(key, value);
            return this;
        }

        public Builder description(String description) {
            this.description = description;
            return this;
        }

        /**
         * Publish SLO boundaries in the set of histogram buckets.
         */
        public Builder serviceLevelObjectives(Duration... slos) {
            if (slos != null) {
                this.distributionConfigBuilder
                        .serviceLevelObjectives(Arrays.stream(slos).mapToDouble(Duration::toNanos).toArray());
            }
            return this;
        }

        /**
         * Sets the minimum value this timer is expected to observe.
         */
        public Builder minimumExpectedValue(Duration min) {
            if (min != null) {
                this.distributionConfigBuilder.minimumExpectedValue((double) min.toNanos());
            }
            return this;
        }

        /**
         * Sets the maximum value this timer is expected to observe.
         */
        public Builder maximumExpectedValue(Duration max) {
            if (max != null) {
                this.distributionConfigBuilder.maximumExpectedValue((double) max.toNanos());
            }
            return this;
        }

        /**
         * Add the long task timer to a single registry, or return an existing one
         * in that registry.
         */
        public LongTaskTimer register(MeterRegistry registry) {
            return registry.more()
                    .longTaskTimer(
                            new Meter.Id(name, tags, Meter.Type.LONG_TASK_TIMER, description, null),
                            distributionConfigBuilder.build());
        }

    }

}
```

#### File: `src/main/java/dev/linhvu/micrometer/internal/DefaultLongTaskTimer.java` [NEW]

```java
package dev.linhvu.micrometer.internal;

import dev.linhvu.micrometer.AbstractMeter;
import dev.linhvu.micrometer.Clock;
import dev.linhvu.micrometer.LongTaskTimer;
import dev.linhvu.micrometer.Meter;
import dev.linhvu.micrometer.TimeUtils;
import dev.linhvu.micrometer.distribution.HistogramSnapshot;

import java.util.NavigableSet;
import java.util.NoSuchElementException;
import java.util.concurrent.ConcurrentSkipListSet;
import java.util.concurrent.TimeUnit;
import java.util.concurrent.atomic.AtomicInteger;

/**
 * Default implementation of {@link LongTaskTimer} that tracks currently active tasks
 * using a {@link ConcurrentSkipListSet}.
 * <p>
 * Why ConcurrentSkipListSet?
 * <ul>
 *   <li>Starting/stopping tasks is O(log N) — happens on the application thread</li>
 *   <li>Querying duration/activeTasks is O(N) — happens on the publishing thread</li>
 *   <li>Tasks complete out of order (removal from the middle), so a queue's O(N) removal
 *       would be worse than a skip list's O(log N)</li>
 * </ul>
 * <p>
 * Samples are naturally ordered by start time. When two samples start at the same
 * nanosecond (possible under high concurrency), a counter-based tiebreaker ensures
 * uniqueness in the set.
 */
public class DefaultLongTaskTimer extends AbstractMeter implements LongTaskTimer {

    private final NavigableSet<SampleImpl> activeTasks = new ConcurrentSkipListSet<>();

    private final AtomicInteger counter = new AtomicInteger();

    private final Clock clock;

    private final TimeUnit baseTimeUnit;

    /**
     * Create a new DefaultLongTaskTimer.
     *
     * @param id           The meter identity.
     * @param clock        Clock for time measurement.
     * @param baseTimeUnit The base time unit for this timer.
     */
    public DefaultLongTaskTimer(Meter.Id id, Clock clock, TimeUnit baseTimeUnit) {
        super(id);
        this.clock = clock;
        this.baseTimeUnit = baseTimeUnit;
    }

    @Override
    public Sample start() {
        long startTime = clock.monotonicTime();
        SampleImpl sample = new SampleImpl(startTime);
        if (!activeTasks.add(sample)) {
            // Two tasks started at the exact same nanosecond — use counter as tiebreaker
            sample = new SampleImplCounted(startTime, nextNonZeroCounter());
            activeTasks.add(sample);
        }
        return sample;
    }

    private int nextNonZeroCounter() {
        int nextCount;
        while ((nextCount = counter.incrementAndGet()) == 0) {
            // skip zero since SampleImpl uses 0 as the default counter
        }
        return nextCount;
    }

    @Override
    public double duration(TimeUnit unit) {
        long now = clock.monotonicTime();
        long sum = 0L;
        for (SampleImpl task : activeTasks) {
            sum += now - task.startTime();
        }
        return TimeUtils.convert(sum, TimeUnit.NANOSECONDS, unit);
    }

    @Override
    public double max(TimeUnit unit) {
        try {
            // first() returns the oldest (longest-running) task
            return activeTasks.first().duration(unit);
        } catch (NoSuchElementException e) {
            return 0.0;
        }
    }

    @Override
    public int activeTasks() {
        return activeTasks.size();
    }

    @Override
    public TimeUnit baseTimeUnit() {
        return baseTimeUnit;
    }

    @Override
    public HistogramSnapshot takeSnapshot() {
        double duration = duration(TimeUnit.NANOSECONDS);
        double max = max(TimeUnit.NANOSECONDS);
        return HistogramSnapshot.empty(activeTasks.size(), duration, max);
    }

    // -----------------------------------------------------------------------
    // Sample implementations
    // -----------------------------------------------------------------------

    /**
     * A sample backed by the enclosing timer's active task set.
     * Ordered by start time, with a counter tiebreaker for same-nanosecond starts.
     */
    class SampleImpl extends Sample implements Comparable<SampleImpl> {

        private final long startTime;

        private volatile boolean stopped;

        SampleImpl(long startTime) {
            this.startTime = startTime;
        }

        int counter() {
            return 0;
        }

        @Override
        public long stop() {
            activeTasks.remove(this);
            long duration = (long) duration(TimeUnit.NANOSECONDS);
            stopped = true;
            return duration;
        }

        @Override
        public double duration(TimeUnit unit) {
            return stopped ? -1 : TimeUtils.convert(
                    clock.monotonicTime() - startTime, TimeUnit.NANOSECONDS, unit);
        }

        long startTime() {
            return startTime;
        }

        @Override
        public int compareTo(SampleImpl that) {
            if (this == that) {
                return 0;
            }
            int startCompare = Long.compare(this.startTime, that.startTime);
            if (startCompare == 0) {
                return Integer.compare(this.counter(), that.counter());
            }
            return startCompare;
        }

        @Override
        public String toString() {
            double durationNanos = duration(TimeUnit.NANOSECONDS);
            return "SampleImpl{duration(seconds)=" +
                    TimeUtils.convert(durationNanos, TimeUnit.NANOSECONDS, TimeUnit.SECONDS) +
                    ", startTimeNanos=" + startTime + '}';
        }

    }

    /**
     * Extended sample with a non-zero counter for breaking ties when two tasks
     * start at the exact same nanosecond.
     */
    class SampleImplCounted extends SampleImpl {

        private final int counterValue;

        SampleImplCounted(long startTime, int counterValue) {
            super(startTime);
            this.counterValue = counterValue;
        }

        @Override
        int counter() {
            return counterValue;
        }

    }

}
```

#### File: `src/main/java/dev/linhvu/micrometer/noop/NoopLongTaskTimer.java` [NEW]

```java
package dev.linhvu.micrometer.noop;

import dev.linhvu.micrometer.LongTaskTimer;
import dev.linhvu.micrometer.Meter;
import dev.linhvu.micrometer.distribution.HistogramSnapshot;

import java.util.concurrent.TimeUnit;

/**
 * A long task timer that does nothing. {@link #start()} returns a no-op sample,
 * and all query methods return zero.
 * <p>
 * Used when the registry is closed or a MeterFilter denies registration.
 */
public class NoopLongTaskTimer extends NoopMeter implements LongTaskTimer {

    public NoopLongTaskTimer(Meter.Id id) {
        super(id);
    }

    @Override
    public Sample start() {
        return new NoopSample();
    }

    @Override
    public double duration(TimeUnit unit) {
        return 0;
    }

    @Override
    public int activeTasks() {
        return 0;
    }

    @Override
    public double max(TimeUnit unit) {
        return 0;
    }

    @Override
    public HistogramSnapshot takeSnapshot() {
        return HistogramSnapshot.empty(0, 0, 0);
    }

    @Override
    public TimeUnit baseTimeUnit() {
        return TimeUnit.SECONDS;
    }

    static class NoopSample extends Sample {

        @Override
        public long stop() {
            return 0;
        }

        @Override
        public double duration(TimeUnit unit) {
            return 0;
        }

    }

}
```

#### File: `src/main/java/dev/linhvu/micrometer/MeterRegistry.java` [MODIFIED]

```java
package dev.linhvu.micrometer;

import dev.linhvu.micrometer.config.MeterFilter;
import dev.linhvu.micrometer.config.MeterFilterReply;
import dev.linhvu.micrometer.config.NamingConvention;
import dev.linhvu.micrometer.distribution.DistributionStatisticConfig;
import dev.linhvu.micrometer.noop.NoopCounter;
import dev.linhvu.micrometer.noop.NoopDistributionSummary;
import dev.linhvu.micrometer.noop.NoopFunctionCounter;
import dev.linhvu.micrometer.noop.NoopFunctionTimer;
import dev.linhvu.micrometer.noop.NoopGauge;
import dev.linhvu.micrometer.noop.NoopLongTaskTimer;
import dev.linhvu.micrometer.noop.NoopTimeGauge;
import dev.linhvu.micrometer.noop.NoopTimer;

import java.util.ArrayList;
import java.util.Arrays;
import java.util.Collection;
import java.util.Collections;
import java.util.List;
import java.util.Map;
import java.util.Objects;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.TimeUnit;
import java.util.function.Function;
import java.util.function.ToDoubleFunction;
import java.util.function.ToLongFunction;

// ... (unchanged registry code) ...

    // New abstract factory method:
    protected abstract LongTaskTimer newLongTaskTimer(Meter.Id id,
            DistributionStatisticConfig distributionStatisticConfig);

    // In the More inner class, new method:
    LongTaskTimer longTaskTimer(Meter.Id id, DistributionStatisticConfig distributionStatisticConfig) {
        return registerMeterIfNecessary(LongTaskTimer.class, id,
                mappedId -> newLongTaskTimer(mappedId,
                        configureDistributionStatistics(mappedId, distributionStatisticConfig)),
                NoopLongTaskTimer::new);
    }

// ... (rest unchanged) ...
```

*See the full file in `src/main/java/dev/linhvu/micrometer/MeterRegistry.java` — only the additions above were changed.*

#### File: `src/main/java/dev/linhvu/micrometer/simple/SimpleMeterRegistry.java` [MODIFIED]

```java
// New import:
import dev.linhvu.micrometer.internal.DefaultLongTaskTimer;

// New factory method:
@Override
protected LongTaskTimer newLongTaskTimer(Meter.Id id,
        DistributionStatisticConfig distributionStatisticConfig) {
    return new DefaultLongTaskTimer(id, clock, getBaseTimeUnit());
}
```

*See the full file in `src/main/java/dev/linhvu/micrometer/simple/SimpleMeterRegistry.java` — only the additions above were changed.*

### Test Code

#### File: `src/test/java/dev/linhvu/micrometer/LongTaskTimerTest.java` [NEW]

```java
package dev.linhvu.micrometer;

import dev.linhvu.micrometer.distribution.HistogramSnapshot;
import org.junit.jupiter.api.Test;

import java.util.List;
import java.util.concurrent.TimeUnit;

import static org.assertj.core.api.Assertions.*;

/**
 * Tests for the {@link LongTaskTimer} interface: builder, convenience methods,
 * default measure(), and mean().
 */
class LongTaskTimerTest {

    @Test
    void shouldCreateBuilder_WhenGivenName() {
        LongTaskTimer.Builder builder = LongTaskTimer.builder("test.timer");
        assertThat(builder).isNotNull();
    }

    @Test
    void shouldBuildWithTags_WhenTagsProvided() {
        MockClock clock = new MockClock();
        var registry = new dev.linhvu.micrometer.simple.SimpleMeterRegistry(clock);

        LongTaskTimer ltt = LongTaskTimer.builder("test.timer")
                .tag("env", "prod")
                .tags("region", "us-east")
                .description("A test timer")
                .register(registry);

        assertThat(ltt).isNotNull();
        assertThat(ltt.getId().getName()).isEqualTo("test.timer");
        assertThat(ltt.getId().getTag("env")).isEqualTo("prod");
        assertThat(ltt.getId().getTag("region")).isEqualTo("us-east");
    }

    @Test
    void shouldReturnZeroMean_WhenNoActiveTasks() {
        MockClock clock = new MockClock();
        var registry = new dev.linhvu.micrometer.simple.SimpleMeterRegistry(clock);

        LongTaskTimer ltt = LongTaskTimer.builder("test.timer").register(registry);

        assertThat(ltt.mean(TimeUnit.SECONDS)).isEqualTo(0.0);
    }

    @Test
    void shouldProduceThreeMeasurements_WhenMeasureCalled() {
        MockClock clock = new MockClock();
        var registry = new dev.linhvu.micrometer.simple.SimpleMeterRegistry(clock);
        LongTaskTimer ltt = LongTaskTimer.builder("test.timer").register(registry);

        List<Measurement> measurements = new java.util.ArrayList<>();
        ltt.measure().forEach(measurements::add);

        assertThat(measurements).hasSize(3);
        assertThat(measurements.get(0).getStatistic()).isEqualTo(Statistic.ACTIVE_TASKS);
        assertThat(measurements.get(1).getStatistic()).isEqualTo(Statistic.DURATION);
        assertThat(measurements.get(2).getStatistic()).isEqualTo(Statistic.MAX);
    }

    @Test
    void shouldWrapRunnable_WhenRecordRunnableCalled() {
        MockClock clock = new MockClock();
        var registry = new dev.linhvu.micrometer.simple.SimpleMeterRegistry(clock);
        LongTaskTimer ltt = LongTaskTimer.builder("test.timer").register(registry);

        boolean[] ran = {false};
        ltt.record(() -> ran[0] = true);

        assertThat(ran[0]).isTrue();
        assertThat(ltt.activeTasks()).isEqualTo(0);
    }

    @Test
    void shouldWrapSupplier_WhenRecordSupplierCalled() {
        MockClock clock = new MockClock();
        var registry = new dev.linhvu.micrometer.simple.SimpleMeterRegistry(clock);
        LongTaskTimer ltt = LongTaskTimer.builder("test.timer").register(registry);

        String result = ltt.record(() -> "hello");

        assertThat(result).isEqualTo("hello");
        assertThat(ltt.activeTasks()).isEqualTo(0);
    }

    @Test
    void shouldWrapCallable_WhenRecordCallableCalled() throws Exception {
        MockClock clock = new MockClock();
        var registry = new dev.linhvu.micrometer.simple.SimpleMeterRegistry(clock);
        LongTaskTimer ltt = LongTaskTimer.builder("test.timer").register(registry);

        int result = ltt.recordCallable(() -> 42);

        assertThat(result).isEqualTo(42);
        assertThat(ltt.activeTasks()).isEqualTo(0);
    }

    @Test
    void shouldPassSampleToConsumer_WhenRecordConsumerCalled() {
        MockClock clock = new MockClock();
        var registry = new dev.linhvu.micrometer.simple.SimpleMeterRegistry(clock);
        LongTaskTimer ltt = LongTaskTimer.builder("test.timer").register(registry);

        LongTaskTimer.Sample[] captured = new LongTaskTimer.Sample[1];
        ltt.record(sample -> {
            captured[0] = sample;
            // During execution, the task should be active
            assertThat(ltt.activeTasks()).isEqualTo(1);
        });

        assertThat(captured[0]).isNotNull();
        assertThat(ltt.activeTasks()).isEqualTo(0);
    }

    @Test
    void shouldStopSample_WhenRunnableThrows() {
        MockClock clock = new MockClock();
        var registry = new dev.linhvu.micrometer.simple.SimpleMeterRegistry(clock);
        LongTaskTimer ltt = LongTaskTimer.builder("test.timer").register(registry);

        assertThatThrownBy(() -> ltt.record((Runnable) () -> {
            throw new RuntimeException("boom");
        })).isInstanceOf(RuntimeException.class);

        // Sample should still be stopped even if the runnable threw
        assertThat(ltt.activeTasks()).isEqualTo(0);
    }
}
```

#### File: `src/test/java/dev/linhvu/micrometer/internal/DefaultLongTaskTimerTest.java` [NEW]

```java
package dev.linhvu.micrometer.internal;

import dev.linhvu.micrometer.LongTaskTimer;
import dev.linhvu.micrometer.Meter;
import dev.linhvu.micrometer.MockClock;
import dev.linhvu.micrometer.Tags;
import dev.linhvu.micrometer.distribution.HistogramSnapshot;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.util.concurrent.TimeUnit;

import static org.assertj.core.api.Assertions.*;

/**
 * Tests for {@link DefaultLongTaskTimer} — the ConcurrentSkipListSet-based implementation.
 */
class DefaultLongTaskTimerTest {

    private MockClock clock;
    private DefaultLongTaskTimer timer;

    @BeforeEach
    void setUp() {
        clock = new MockClock();
        timer = new DefaultLongTaskTimer(
                new Meter.Id("test.ltt", Tags.empty(), Meter.Type.LONG_TASK_TIMER, null, null),
                clock, TimeUnit.SECONDS);
    }

    // -----------------------------------------------------------------------
    // Basic start/stop lifecycle
    // -----------------------------------------------------------------------

    @Test
    void shouldHaveZeroActiveTasks_WhenNewlyCreated() {
        assertThat(timer.activeTasks()).isEqualTo(0);
        assertThat(timer.duration(TimeUnit.SECONDS)).isEqualTo(0.0);
        assertThat(timer.max(TimeUnit.SECONDS)).isEqualTo(0.0);
    }

    @Test
    void shouldTrackOneActiveTask_WhenStarted() {
        LongTaskTimer.Sample sample = timer.start();

        assertThat(timer.activeTasks()).isEqualTo(1);
        assertThat(sample).isNotNull();
    }

    @Test
    void shouldRemoveActiveTask_WhenStopped() {
        LongTaskTimer.Sample sample = timer.start();
        clock.addSeconds(5);

        long duration = sample.stop();

        assertThat(timer.activeTasks()).isEqualTo(0);
        assertThat(duration).isEqualTo(TimeUnit.SECONDS.toNanos(5));
    }

    @Test
    void shouldReturnNegativeOne_WhenSampleDurationQueriedAfterStop() {
        LongTaskTimer.Sample sample = timer.start();
        sample.stop();

        assertThat(sample.duration(TimeUnit.SECONDS)).isEqualTo(-1.0);
    }

    // -----------------------------------------------------------------------
    // Duration and max computation
    // -----------------------------------------------------------------------

    @Test
    void shouldReportDuration_WhenTaskIsActive() {
        timer.start();
        clock.addSeconds(10);

        assertThat(timer.duration(TimeUnit.SECONDS)).isEqualTo(10.0);
    }

    @Test
    void shouldReportCumulativeDuration_WhenMultipleTasksActive() {
        timer.start();
        clock.addSeconds(5);
        timer.start();
        clock.addSeconds(3);

        // First task: 8 seconds, second task: 3 seconds → total 11 seconds
        assertThat(timer.duration(TimeUnit.SECONDS)).isEqualTo(11.0);
        assertThat(timer.activeTasks()).isEqualTo(2);
    }

    @Test
    void shouldReportMax_WhenMultipleTasksActive() {
        timer.start();
        clock.addSeconds(10);
        timer.start();
        clock.addSeconds(5);

        // The oldest task has been running for 15 seconds
        assertThat(timer.max(TimeUnit.SECONDS)).isEqualTo(15.0);
    }

    @Test
    void shouldReportZeroMax_WhenNoTasksActive() {
        assertThat(timer.max(TimeUnit.SECONDS)).isEqualTo(0.0);
    }

    @Test
    void shouldReportMean_WhenMultipleTasksActive() {
        timer.start();
        clock.addSeconds(10);
        timer.start();
        clock.addSeconds(2);

        // Task 1: 12s, Task 2: 2s → mean = 7s
        assertThat(timer.mean(TimeUnit.SECONDS)).isEqualTo(7.0);
    }

    // -----------------------------------------------------------------------
    // Multiple tasks — start/stop ordering
    // -----------------------------------------------------------------------

    @Test
    void shouldTrackTasksIndependently_WhenStoppedOutOfOrder() {
        LongTaskTimer.Sample first = timer.start();
        clock.addSeconds(5);
        LongTaskTimer.Sample second = timer.start();
        clock.addSeconds(5);

        // Stop the first task (out of order)
        first.stop();

        assertThat(timer.activeTasks()).isEqualTo(1);
        // Only the second task remains (running for 5 seconds)
        assertThat(timer.duration(TimeUnit.SECONDS)).isEqualTo(5.0);
    }

    @Test
    void shouldHandleStoppingAlreadyStoppedSample_Gracefully() {
        LongTaskTimer.Sample sample = timer.start();
        sample.stop();

        // Second stop should not throw — it just returns -1 nanos (already removed)
        long secondStop = sample.stop();
        // The sample is already removed, so duration returns -1 (stopped)
        assertThat(timer.activeTasks()).isEqualTo(0);
    }

    // -----------------------------------------------------------------------
    // Sample duration
    // -----------------------------------------------------------------------

    @Test
    void shouldReportSampleDuration_WhenQueried() {
        LongTaskTimer.Sample sample = timer.start();
        clock.addSeconds(7);

        assertThat(sample.duration(TimeUnit.SECONDS)).isEqualTo(7.0);
        assertThat(sample.duration(TimeUnit.MILLISECONDS)).isEqualTo(7000.0);
    }

    // -----------------------------------------------------------------------
    // Time unit conversion
    // -----------------------------------------------------------------------

    @Test
    void shouldConvertDuration_WhenDifferentUnitRequested() {
        timer.start();
        clock.add(2500, TimeUnit.MILLISECONDS);

        assertThat(timer.duration(TimeUnit.SECONDS)).isEqualTo(2.5);
        assertThat(timer.duration(TimeUnit.MILLISECONDS)).isEqualTo(2500.0);
    }

    @Test
    void shouldReportBaseTimeUnit() {
        assertThat(timer.baseTimeUnit()).isEqualTo(TimeUnit.SECONDS);
    }

    // -----------------------------------------------------------------------
    // Snapshot
    // -----------------------------------------------------------------------

    @Test
    void shouldTakeSnapshot_WhenTasksActive() {
        timer.start();
        clock.addSeconds(10);
        timer.start();
        clock.addSeconds(5);

        HistogramSnapshot snapshot = timer.takeSnapshot();

        assertThat(snapshot.count()).isEqualTo(2);
        // Total in nanoseconds: task1=15s, task2=5s → 20s in nanos
        assertThat(snapshot.total()).isEqualTo(TimeUnit.SECONDS.toNanos(20));
        // Max in nanoseconds: 15s
        assertThat(snapshot.max()).isEqualTo(TimeUnit.SECONDS.toNanos(15));
    }

    @Test
    void shouldReturnEmptySnapshot_WhenNoTasksActive() {
        HistogramSnapshot snapshot = timer.takeSnapshot();

        assertThat(snapshot.count()).isEqualTo(0);
        assertThat(snapshot.total()).isEqualTo(0.0);
        assertThat(snapshot.max()).isEqualTo(0.0);
    }

    // -----------------------------------------------------------------------
    // Same-nanosecond tiebreaking
    // -----------------------------------------------------------------------

    @Test
    void shouldHandleSameStartTime_WhenTwoTasksStartSimultaneously() {
        // Both start at the same clock time (no clock.add between starts)
        LongTaskTimer.Sample first = timer.start();
        LongTaskTimer.Sample second = timer.start();

        assertThat(timer.activeTasks()).isEqualTo(2);

        first.stop();
        assertThat(timer.activeTasks()).isEqualTo(1);

        second.stop();
        assertThat(timer.activeTasks()).isEqualTo(0);
    }
}
```

#### File: `src/test/java/dev/linhvu/micrometer/noop/NoopLongTaskTimerTest.java` [NEW]

```java
package dev.linhvu.micrometer.noop;

import dev.linhvu.micrometer.LongTaskTimer;
import dev.linhvu.micrometer.Meter;
import dev.linhvu.micrometer.Tags;
import dev.linhvu.micrometer.distribution.HistogramSnapshot;
import org.junit.jupiter.api.Test;

import java.util.concurrent.TimeUnit;

import static org.assertj.core.api.Assertions.*;

/**
 * Tests for {@link NoopLongTaskTimer} — verifies it does nothing but is safe to call.
 */
class NoopLongTaskTimerTest {

    private final NoopLongTaskTimer noop = new NoopLongTaskTimer(
            new Meter.Id("noop.ltt", Tags.empty(), Meter.Type.LONG_TASK_TIMER, null, null));

    @Test
    void shouldReturnZero_ForAllQueryMethods() {
        assertThat(noop.activeTasks()).isEqualTo(0);
        assertThat(noop.duration(TimeUnit.SECONDS)).isEqualTo(0.0);
        assertThat(noop.max(TimeUnit.SECONDS)).isEqualTo(0.0);
    }

    @Test
    void shouldReturnNoopSample_WhenStartCalled() {
        LongTaskTimer.Sample sample = noop.start();
        assertThat(sample).isNotNull();
        assertThat(sample.stop()).isEqualTo(0);
        assertThat(sample.duration(TimeUnit.SECONDS)).isEqualTo(0.0);
    }

    @Test
    void shouldReturnEmptySnapshot_WhenTakeSnapshotCalled() {
        HistogramSnapshot snapshot = noop.takeSnapshot();
        assertThat(snapshot.count()).isEqualTo(0);
        assertThat(snapshot.total()).isEqualTo(0.0);
        assertThat(snapshot.max()).isEqualTo(0.0);
    }

    @Test
    void shouldReturnSecondsBaseTimeUnit() {
        assertThat(noop.baseTimeUnit()).isEqualTo(TimeUnit.SECONDS);
    }
}
```

#### File: `src/test/java/dev/linhvu/micrometer/integration/LongTaskTimerIntegrationTest.java` [NEW]

```java
package dev.linhvu.micrometer.integration;

import dev.linhvu.micrometer.LongTaskTimer;
import dev.linhvu.micrometer.Meter;
import dev.linhvu.micrometer.MockClock;
import dev.linhvu.micrometer.Tags;
import dev.linhvu.micrometer.config.MeterFilter;
import dev.linhvu.micrometer.noop.NoopLongTaskTimer;
import dev.linhvu.micrometer.simple.SimpleMeterRegistry;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.util.concurrent.TimeUnit;
import java.util.concurrent.atomic.AtomicReference;

import static org.assertj.core.api.Assertions.*;

/**
 * Integration tests for LongTaskTimer with MeterRegistry, MeterFilter, and
 * the visitor pattern.
 */
class LongTaskTimerIntegrationTest {

    private MockClock clock;
    private SimpleMeterRegistry registry;

    @BeforeEach
    void setUp() {
        clock = new MockClock();
        registry = new SimpleMeterRegistry(clock);
    }

    // -----------------------------------------------------------------------
    // Registry registration
    // -----------------------------------------------------------------------

    @Test
    void shouldRegisterViaBuilder_WhenRegisteredWithRegistry() {
        LongTaskTimer ltt = LongTaskTimer.builder("batch.import")
                .tag("source", "csv")
                .register(registry);

        assertThat(ltt).isNotNull();
        assertThat(ltt.getId().getName()).isEqualTo("batch.import");
        assertThat(ltt.activeTasks()).isEqualTo(0);
    }

    @Test
    void shouldDeduplicateRegistration_WhenSameNameAndTags() {
        LongTaskTimer first = LongTaskTimer.builder("batch.import").register(registry);
        LongTaskTimer second = LongTaskTimer.builder("batch.import").register(registry);

        assertThat(first).isSameAs(second);
    }

    @Test
    void shouldAppearInMeterList_WhenRegistered() {
        LongTaskTimer.builder("batch.import").register(registry);

        assertThat(registry.getMeters()).hasSize(1);
        assertThat(registry.getMeters().get(0)).isInstanceOf(LongTaskTimer.class);
    }

    @Test
    void shouldThrow_WhenNameConflictsWithDifferentType() {
        registry.counter("conflict");

        assertThatThrownBy(() -> LongTaskTimer.builder("conflict").register(registry))
                .isInstanceOf(IllegalArgumentException.class);
    }

    // -----------------------------------------------------------------------
    // MeterFilter integration
    // -----------------------------------------------------------------------

    @Test
    void shouldReturnNoop_WhenFilterDeniesRegistration() {
        registry.meterFilter(MeterFilter.deny());

        LongTaskTimer ltt = LongTaskTimer.builder("denied.timer").register(registry);

        assertThat(ltt).isInstanceOf(NoopLongTaskTimer.class);
    }

    @Test
    void shouldApplyCommonTags_WhenFilterAddsCommonTags() {
        registry.meterFilter(MeterFilter.commonTags(Tags.of("env", "prod")));

        LongTaskTimer ltt = LongTaskTimer.builder("batch.import").register(registry);

        assertThat(ltt.getId().getTag("env")).isEqualTo("prod");
    }

    @Test
    void shouldReturnNoop_WhenRegistryClosed() {
        registry.close();

        LongTaskTimer ltt = LongTaskTimer.builder("closed.timer").register(registry);

        assertThat(ltt).isInstanceOf(NoopLongTaskTimer.class);
    }

    // -----------------------------------------------------------------------
    // Visitor pattern
    // -----------------------------------------------------------------------

    @Test
    void shouldDispatchToLongTaskTimerVisitor_WhenMatchCalled() {
        LongTaskTimer ltt = LongTaskTimer.builder("batch.import").register(registry);

        String result = ltt.match(
                counter -> "counter",
                gauge -> "gauge",
                timer -> "timer",
                ds -> "distribution_summary",
                longTaskTimer -> "long_task_timer",
                fc -> "function_counter",
                ft -> "function_timer",
                m -> "other"
        );

        assertThat(result).isEqualTo("long_task_timer");
    }

    @Test
    void shouldDispatchToLongTaskTimerVisitor_WhenUseCalled() {
        LongTaskTimer ltt = LongTaskTimer.builder("batch.import").register(registry);
        AtomicReference<LongTaskTimer> captured = new AtomicReference<>();

        ltt.use(
                counter -> {},
                gauge -> {},
                timer -> {},
                ds -> {},
                captured::set,
                fc -> {},
                ft -> {},
                m -> {}
        );

        assertThat(captured.get()).isSameAs(ltt);
    }

    // -----------------------------------------------------------------------
    // End-to-end: typical usage pattern
    // -----------------------------------------------------------------------

    @Test
    void shouldTrackLongRunningTask_WhenUsedEndToEnd() {
        LongTaskTimer ltt = LongTaskTimer.builder("batch.import")
                .tag("source", "csv")
                .register(registry);

        // Start a long-running task
        LongTaskTimer.Sample sample = ltt.start();

        // Simulate 30 seconds of work
        clock.addSeconds(30);

        // While running, we can check progress
        assertThat(ltt.activeTasks()).isEqualTo(1);
        assertThat(ltt.duration(TimeUnit.SECONDS)).isEqualTo(30.0);
        assertThat(ltt.max(TimeUnit.SECONDS)).isEqualTo(30.0);

        // Start a second task
        LongTaskTimer.Sample sample2 = ltt.start();
        clock.addSeconds(10);

        // Both tasks active
        assertThat(ltt.activeTasks()).isEqualTo(2);
        assertThat(ltt.duration(TimeUnit.SECONDS)).isEqualTo(50.0); // 40 + 10

        // First task completes
        sample.stop();
        assertThat(ltt.activeTasks()).isEqualTo(1);
        assertThat(ltt.duration(TimeUnit.SECONDS)).isEqualTo(10.0);

        // Second task completes
        sample2.stop();
        assertThat(ltt.activeTasks()).isEqualTo(0);
        assertThat(ltt.duration(TimeUnit.SECONDS)).isEqualTo(0.0);
    }
}
```

---

## Summary

| Concept | What It Means |
|---------|---------------|
| **LongTaskTimer** | A meter that tracks *in-flight* operations — how many are running and how long they've been going |
| **Sample (live handle)** | A reference to an active task that can be queried for its current duration before being stopped |
| **ConcurrentSkipListSet** | O(log N) insert/remove with natural ordering — ideal for tasks that complete out of order |
| **ACTIVE_TASKS / DURATION** | The statistics emitted by LongTaskTimer, replacing Timer's COUNT / TOTAL_TIME |
| **SampleImplCounted** | Tiebreaker for the rare case when two tasks start at the exact same nanosecond |

**Next: Chapter 13 — MeterBinder** — A pattern for packaging reusable sets of related metrics that auto-register on a MeterRegistry. You'll build `JvmMemoryMetrics`, `JvmThreadMetrics`, and `UptimeMetrics` as examples of the idiomatic auto-instrumentation pattern.
