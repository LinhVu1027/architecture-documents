# Chapter 11: Function Meters

> Line references based on commit `2c8a4606c` of the Micrometer repository.

## Build Challenge

| Current State | Limitation | Objective |
|---------------|------------|-----------|
| The registry supports Counter, Gauge, Timer, and DistributionSummary — all of which own their internal state and require explicit recording calls (`increment()`, `record()`) | Cannot instrument libraries that already maintain their own statistics (Kafka consumer metrics, connection pool stats, JMX beans) — you'd have to forward every update to a Micrometer meter, duplicating bookkeeping | Build `FunctionCounter`, `FunctionTimer`, and `TimeGauge` — meters that derive values from external objects via functions — plus a `More` inner class on `MeterRegistry` to provide a clean namespace for these less common meter types |

---

## 11.1 The Integration Point: MeterRegistry gains `More` inner class

**Modifying:** `src/main/java/dev/linhvu/micrometer/MeterRegistry.java`
**Change:** Add `more()` accessor and the `More` inner class that provides registration methods for function meters, plus abstract factory methods `newFunctionCounter`, `newFunctionTimer`, and `newTimeGauge`.

```java
// --- New abstract factory methods (alongside existing newCounter, newGauge, etc.) ---

protected abstract <T> FunctionCounter newFunctionCounter(Meter.Id id, T obj,
        ToDoubleFunction<T> countFunction);

protected abstract <T> FunctionTimer newFunctionTimer(Meter.Id id, T obj,
        ToLongFunction<T> countFunction, ToDoubleFunction<T> totalTimeFunction,
        TimeUnit totalTimeFunctionUnit);

protected <T> TimeGauge newTimeGauge(Meter.Id id, T obj, TimeUnit timeFunctionUnit,
        ToDoubleFunction<T> f) {
    return new DefaultTimeGauge<>(id, obj, timeFunctionUnit, f, getBaseTimeUnit());
}

// --- The More accessor and inner class ---

private final More more = new More();

public More more() {
    return more;
}

public class More {

    <T> FunctionCounter counter(Meter.Id id, T obj, ToDoubleFunction<T> countFunction) {
        return registerMeterIfNecessary(FunctionCounter.class, id,
                mappedId -> newFunctionCounter(mappedId, obj, countFunction),
                NoopFunctionCounter::new);
    }

    <T> FunctionTimer timer(Meter.Id id, T obj, ToLongFunction<T> countFunction,
            ToDoubleFunction<T> totalTimeFunction, TimeUnit totalTimeFunctionUnit) {
        return registerMeterIfNecessary(FunctionTimer.class,
                id.withBaseUnit(getBaseTimeUnit().name().toLowerCase()),
                mappedId -> newFunctionTimer(mappedId, obj, countFunction,
                        totalTimeFunction, totalTimeFunctionUnit),
                NoopFunctionTimer::new);
    }

    <T> TimeGauge timeGauge(Meter.Id id, T obj, TimeUnit timeFunctionUnit,
            ToDoubleFunction<T> timeFunction) {
        return registerMeterIfNecessary(TimeGauge.class, id,
                mappedId -> newTimeGauge(mappedId, obj, timeFunctionUnit, timeFunction),
                NoopTimeGauge::new);
    }

    // + public convenience methods with (name, tags, obj, function) signatures
}
```

Three key decisions here:

1. **The `More` namespace.** Rather than cluttering `MeterRegistry` with `functionCounter()`, `functionTimer()`, `timeGauge()` methods at the top level, Micrometer groups them behind `registry.more()`. This keeps the common API surface clean (counter/gauge/timer/summary) while providing a discoverable path to advanced types.

2. **Package-private + public methods.** Each function meter type has two registration paths: a package-private method (called by `Builder.register()`) that takes a pre-built `Meter.Id`, and a public convenience method (called by users) that takes name + tags + object + function. This mirrors the pattern used by Counter and Gauge.

3. **`newTimeGauge` has a default implementation.** Unlike `newFunctionCounter` and `newFunctionTimer` which are abstract, `newTimeGauge` provides a default that creates a `DefaultTimeGauge`. Most registries don't need a custom TimeGauge implementation.

This connects **User Instrumentation Code** (Builders and `registry.more()` calls) to **Registry Lifecycle Management** (deduplication, filtering, factory dispatch). To make it work, we need to build:
- `FunctionCounter` — interface with `count()` and a `Builder`
- `FunctionTimer` — interface with `count()`, `totalTime()`, `mean()`, and a `Builder`
- `TimeGauge` — interface extending `Gauge` with time-unit awareness and a `Builder`
- `TimeUtils` — double-precision time conversion utility
- `CumulativeFunctionCounter` — WeakReference-backed implementation
- `CumulativeFunctionTimer` — WeakReference-backed implementation with time conversion
- `DefaultTimeGauge` — WeakReference-backed time gauge with unit conversion
- Noop implementations for all three types

## 11.2 TimeUtils — Double-Precision Time Conversion

**New file:** `src/main/java/dev/linhvu/micrometer/TimeUtils.java`

```java
package dev.linhvu.micrometer;

import java.util.concurrent.TimeUnit;

public final class TimeUtils {

    private TimeUtils() {
    }

    public static double convert(double value, TimeUnit from, TimeUnit to) {
        if (from == to) {
            return value;
        }
        double nanos = value * nanosPerUnit(from);
        return nanos / nanosPerUnit(to);
    }

    private static double nanosPerUnit(TimeUnit unit) {
        return switch (unit) {
            case NANOSECONDS -> 1.0;
            case MICROSECONDS -> 1_000.0;
            case MILLISECONDS -> 1_000_000.0;
            case SECONDS -> 1_000_000_000.0;
            case MINUTES -> 60_000_000_000.0;
            case HOURS -> 3_600_000_000_000.0;
            case DAYS -> 86_400_000_000_000.0;
        };
    }

}
```

JDK's `TimeUnit.convert(long, TimeUnit)` truncates because it operates on `long` values — converting 500 milliseconds to seconds yields 0, not 0.5. This utility converts through nanoseconds as a common denominator to preserve fractional precision.

## 11.3 FunctionCounter

**Modifying:** `src/main/java/dev/linhvu/micrometer/FunctionCounter.java`
**Change:** Replace stub interface with full contract including `count()`, `measure()`, builder factory, and `Builder<T>` inner class.

```java
package dev.linhvu.micrometer;

import java.util.Collections;
import java.util.function.ToDoubleFunction;

public interface FunctionCounter extends Meter {

    static <T> Builder<T> builder(String name, T obj, ToDoubleFunction<T> f) {
        return new Builder<>(name, obj, f);
    }

    double count();

    @Override
    default Iterable<Measurement> measure() {
        return Collections.singletonList(new Measurement(this::count, Statistic.COUNT));
    }

    class Builder<T> {
        private final String name;
        private final T obj;
        private final ToDoubleFunction<T> f;
        private Tags tags = Tags.empty();
        private String description;
        private String baseUnit;

        // tags(), tag(), description(), baseUnit() fluent methods...

        public FunctionCounter register(MeterRegistry registry) {
            return registry.more().counter(
                    new Meter.Id(name, tags, Meter.Type.COUNTER, description, baseUnit), obj, f);
        }
    }
}
```

The Builder calls `registry.more().counter()` — routing through the `More` inner class that we set up in the integration point.

## 11.4 FunctionTimer

**Modifying:** `src/main/java/dev/linhvu/micrometer/FunctionTimer.java`
**Change:** Replace stub interface with full contract including `count()`, `totalTime()`, `mean()`, `baseTimeUnit()`, `measure()`, and `Builder<T>`.

```java
package dev.linhvu.micrometer;

import java.util.Arrays;
import java.util.concurrent.TimeUnit;
import java.util.function.ToDoubleFunction;
import java.util.function.ToLongFunction;

public interface FunctionTimer extends Meter {

    static <T> Builder<T> builder(String name, T obj, ToLongFunction<T> countFunction,
            ToDoubleFunction<T> totalTimeFunction, TimeUnit totalTimeFunctionUnit) {
        return new Builder<>(name, obj, countFunction, totalTimeFunction, totalTimeFunctionUnit);
    }

    double count();

    double totalTime(TimeUnit unit);

    default double mean(TimeUnit unit) {
        double count = count();
        return count == 0 ? 0 : totalTime(unit) / count;
    }

    TimeUnit baseTimeUnit();

    @Override
    default Iterable<Measurement> measure() {
        return Arrays.asList(
                new Measurement(this::count, Statistic.COUNT),
                new Measurement(() -> totalTime(baseTimeUnit()), Statistic.TOTAL_TIME));
    }

    class Builder<T> {
        // countFunction (ToLongFunction) + totalTimeFunction (ToDoubleFunction) + totalTimeFunctionUnit

        public FunctionTimer register(MeterRegistry registry) {
            return registry.more().timer(
                    new Meter.Id(name, tags, Meter.Type.TIMER, description, null),
                    obj, countFunction, totalTimeFunction, totalTimeFunctionUnit);
        }
    }
}
```

Note that the count function is `ToLongFunction` (not `ToDoubleFunction`) — event counts are always whole numbers. The total time function is `ToDoubleFunction` because time durations can be fractional.

## 11.5 TimeGauge

**New file:** `src/main/java/dev/linhvu/micrometer/TimeGauge.java`

```java
package dev.linhvu.micrometer;

import java.util.concurrent.TimeUnit;
import java.util.function.ToDoubleFunction;

public interface TimeGauge extends Gauge {

    static <T> Builder<T> builder(String name, T obj, TimeUnit fUnits, ToDoubleFunction<T> f) {
        return new Builder<>(name, obj, fUnits, f);
    }

    TimeUnit baseTimeUnit();

    default double value(TimeUnit unit) {
        return TimeUtils.convert(value(), baseTimeUnit(), unit);
    }

    class Builder<T> {
        // fUnits — the time unit of the source function's output

        public TimeGauge register(MeterRegistry registry) {
            return registry.more().timeGauge(
                    new Meter.Id(name, tags, Meter.Type.GAUGE, description, null),
                    obj, fUnits, f);
        }
    }
}
```

`TimeGauge extends Gauge` — it IS-A Gauge with additional time-unit awareness. `value()` (from Gauge) returns the value in the registry's base time unit. `value(TimeUnit)` converts from the base unit to any requested unit.

## 11.6 Cumulative Implementations

**New file:** `src/main/java/dev/linhvu/micrometer/cumulative/CumulativeFunctionCounter.java`

```java
package dev.linhvu.micrometer.cumulative;

import dev.linhvu.micrometer.AbstractMeter;
import dev.linhvu.micrometer.FunctionCounter;
import dev.linhvu.micrometer.Meter;

import java.lang.ref.WeakReference;
import java.util.function.ToDoubleFunction;

public class CumulativeFunctionCounter<T> extends AbstractMeter implements FunctionCounter {

    private final WeakReference<T> ref;
    private final ToDoubleFunction<T> f;
    private volatile double last;

    public CumulativeFunctionCounter(Meter.Id id, T obj, ToDoubleFunction<T> f) {
        super(id);
        this.ref = new WeakReference<>(obj);
        this.f = f;
    }

    @Override
    public double count() {
        T obj = ref.get();
        return obj != null ? (last = f.applyAsDouble(obj)) : last;
    }
}
```

When the source object is GC'd, the counter returns the last known value (not NaN like DefaultGauge). This is because a counter's last value is still meaningful — it represents the total count up to the point the source disappeared.

**New file:** `src/main/java/dev/linhvu/micrometer/cumulative/CumulativeFunctionTimer.java`

```java
package dev.linhvu.micrometer.cumulative;

import dev.linhvu.micrometer.AbstractMeter;
import dev.linhvu.micrometer.FunctionTimer;
import dev.linhvu.micrometer.TimeUtils;

import java.lang.ref.WeakReference;
import java.util.concurrent.TimeUnit;
import java.util.function.ToDoubleFunction;
import java.util.function.ToLongFunction;

public class CumulativeFunctionTimer<T> extends AbstractMeter implements FunctionTimer {

    private final WeakReference<T> ref;
    private final ToLongFunction<T> countFunction;
    private final ToDoubleFunction<T> totalTimeFunction;
    private final TimeUnit totalTimeFunctionUnit;
    private final TimeUnit baseTimeUnit;
    private volatile long lastCount;
    private volatile double lastTime;

    // constructor...

    @Override
    public double count() {
        T obj = ref.get();
        return obj != null ? (lastCount = Math.max(countFunction.applyAsLong(obj), 0)) : lastCount;
    }

    @Override
    public double totalTime(TimeUnit unit) {
        T obj = ref.get();
        if (obj != null) {
            lastTime = Math.max(
                    TimeUtils.convert(totalTimeFunction.applyAsDouble(obj),
                            totalTimeFunctionUnit, baseTimeUnit()), 0);
        }
        return TimeUtils.convert(lastTime, baseTimeUnit(), unit);
    }

    @Override
    public TimeUnit baseTimeUnit() {
        return baseTimeUnit;
    }
}
```

Negative values are clamped to 0 via `Math.max` — counts and durations are inherently non-negative. The total time is stored in the registry's base time unit and converted to the requested unit on read.

**New file:** `src/main/java/dev/linhvu/micrometer/internal/DefaultTimeGauge.java`

```java
package dev.linhvu.micrometer.internal;

import dev.linhvu.micrometer.AbstractMeter;
import dev.linhvu.micrometer.TimeGauge;
import dev.linhvu.micrometer.TimeUtils;

import java.lang.ref.WeakReference;
import java.util.concurrent.TimeUnit;
import java.util.function.ToDoubleFunction;

public class DefaultTimeGauge<T> extends AbstractMeter implements TimeGauge {

    private final WeakReference<T> ref;
    private final ToDoubleFunction<T> f;
    private final TimeUnit fUnits;
    private final TimeUnit baseTimeUnit;

    // constructor...

    @Override
    public TimeUnit baseTimeUnit() {
        return baseTimeUnit;
    }

    @Override
    public double value() {
        T obj = ref.get();
        if (obj != null) {
            try {
                return TimeUtils.convert(f.applyAsDouble(obj), fUnits, baseTimeUnit);
            } catch (Exception e) {
                return Double.NaN;
            }
        }
        return Double.NaN;
    }
}
```

Like `DefaultGauge`, if the source is GC'd or the function throws, the gauge returns `NaN`.

## 11.7 Noop Implementations and SimpleMeterRegistry Wiring

**New files:**
- `src/main/java/dev/linhvu/micrometer/noop/NoopFunctionCounter.java` — extends `NoopMeter`, `count()` returns 0
- `src/main/java/dev/linhvu/micrometer/noop/NoopFunctionTimer.java` — extends `NoopMeter`, all methods return 0, base unit is `NANOSECONDS`
- `src/main/java/dev/linhvu/micrometer/noop/NoopTimeGauge.java` — extends `NoopMeter`, `value()` returns 0, base unit is `NANOSECONDS`

**Modifying:** `src/main/java/dev/linhvu/micrometer/simple/SimpleMeterRegistry.java`
**Change:** Implement `newFunctionCounter` and `newFunctionTimer` factory methods.

```java
@Override
protected <T> FunctionCounter newFunctionCounter(Meter.Id id, T obj,
        ToDoubleFunction<T> countFunction) {
    return new CumulativeFunctionCounter<>(id, obj, countFunction);
}

@Override
protected <T> FunctionTimer newFunctionTimer(Meter.Id id, T obj,
        ToLongFunction<T> countFunction, ToDoubleFunction<T> totalTimeFunction,
        TimeUnit totalTimeFunctionUnit) {
    return new CumulativeFunctionTimer<>(id, obj, countFunction, totalTimeFunction,
            totalTimeFunctionUnit, getBaseTimeUnit());
}
```

`newTimeGauge` is inherited from the base `MeterRegistry` — it uses `DefaultTimeGauge` by default.

## 11.8 Try It Yourself

<details>
<summary>Challenge: Wrap a simulated Kafka consumer's metrics using FunctionCounter and FunctionTimer</summary>

Imagine you have a `KafkaConsumerMetrics` object that tracks messages consumed and total fetch latency:

```java
class KafkaConsumerMetrics {
    private long messagesConsumed = 0;
    private long fetchCount = 0;
    private double totalFetchLatencyMs = 0;

    void consume(int messages, double fetchMs) {
        messagesConsumed += messages;
        fetchCount++;
        totalFetchLatencyMs += fetchMs;
    }

    long messagesConsumed() { return messagesConsumed; }
    long fetchCount() { return fetchCount; }
    double totalFetchLatencyMs() { return totalFetchLatencyMs; }
}
```

Register function meters to expose these statistics:

```java
var kafka = new KafkaConsumerMetrics();
var registry = new SimpleMeterRegistry();

FunctionCounter.builder("kafka.messages.consumed", kafka,
        KafkaConsumerMetrics::messagesConsumed)
    .tag("topic", "orders")
    .register(registry);

FunctionTimer.builder("kafka.fetch", kafka,
        KafkaConsumerMetrics::fetchCount,
        KafkaConsumerMetrics::totalFetchLatencyMs,
        TimeUnit.MILLISECONDS)
    .tag("topic", "orders")
    .register(registry);

// Simulate activity
kafka.consume(100, 250.0);
kafka.consume(50, 150.0);

// Query
FunctionCounter counter = (FunctionCounter) registry.getMeters().stream()
        .filter(m -> m.getId().getName().equals("kafka.messages.consumed"))
        .findFirst().orElseThrow();
assertThat(counter.count()).isEqualTo(150.0);

FunctionTimer timer = (FunctionTimer) registry.getMeters().stream()
        .filter(m -> m.getId().getName().equals("kafka.fetch"))
        .findFirst().orElseThrow();
assertThat(timer.count()).isEqualTo(2.0);
assertThat(timer.mean(TimeUnit.MILLISECONDS)).isCloseTo(200.0, within(0.1));
```

</details>

<details>
<summary>Challenge: Implement a TimeGauge that tracks JVM uptime</summary>

```java
var registry = new SimpleMeterRegistry();
var runtimeMXBean = java.lang.management.ManagementFactory.getRuntimeMXBean();

TimeGauge.builder("jvm.uptime", runtimeMXBean,
        TimeUnit.MILLISECONDS,
        bean -> (double) bean.getUptime())
    .description("JVM uptime")
    .register(registry);

// The gauge automatically converts from milliseconds to the registry's
// base unit (seconds for SimpleMeterRegistry)
TimeGauge gauge = (TimeGauge) registry.getMeters().get(0);
double uptimeSeconds = gauge.value(); // in seconds
double uptimeMinutes = gauge.value(TimeUnit.MINUTES); // in minutes
```

</details>

## 11.9 Tests

### Unit Tests

**New file:** `src/test/java/dev/linhvu/micrometer/FunctionCounterTest.java`

```java
@Test
void shouldTrackMonotonicallyIncreasingCount() {
    AtomicLong source = new AtomicLong(0);
    FunctionCounter counter = FunctionCounter.builder("requests", source, AtomicLong::doubleValue)
            .register(registry);
    source.set(42);
    assertThat(counter.count()).isEqualTo(42.0);
}

@Test
void shouldReturnLastValue_WhenSourceIsGarbageCollected() {
    AtomicLong source = new AtomicLong(99);
    FunctionCounter counter = FunctionCounter.builder("gc.test", source, AtomicLong::doubleValue)
            .register(registry);
    assertThat(counter.count()).isEqualTo(99.0);
    source = null;
    System.gc();
    assertThat(counter.count()).isEqualTo(99.0); // last cached, not NaN
}
```

**New file:** `src/test/java/dev/linhvu/micrometer/FunctionTimerTest.java`

```java
@Test
void shouldTrackCountAndTotalTime() {
    TestTimerSource source = new TestTimerSource(10, 5000.0);
    FunctionTimer timer = FunctionTimer.builder("http.requests", source,
                    TestTimerSource::count, TestTimerSource::totalTimeMs, TimeUnit.MILLISECONDS)
            .register(registry);
    assertThat(timer.count()).isEqualTo(10.0);
    assertThat(timer.totalTime(TimeUnit.SECONDS)).isCloseTo(5.0, within(0.001));
}

@Test
void shouldCalculateMean() {
    TestTimerSource source = new TestTimerSource(4, 2000.0);
    FunctionTimer timer = FunctionTimer.builder("ops.duration", source,
                    TestTimerSource::count, TestTimerSource::totalTimeMs, TimeUnit.MILLISECONDS)
            .register(registry);
    assertThat(timer.mean(TimeUnit.MILLISECONDS)).isCloseTo(500.0, within(0.1));
}
```

**New file:** `src/test/java/dev/linhvu/micrometer/TimeGaugeTest.java`

```java
@Test
void shouldConvertTimeValueToRegistryBaseUnit() {
    AtomicLong uptimeMs = new AtomicLong(60_000);
    TimeGauge gauge = TimeGauge.builder("jvm.uptime", uptimeMs,
                    TimeUnit.MILLISECONDS, AtomicLong::doubleValue)
            .register(registry);
    assertThat(gauge.value()).isCloseTo(60.0, within(0.001)); // 60s
}

@Test
void shouldExtendGaugeInterface() {
    TimeGauge timeGauge = TimeGauge.builder("is.a.gauge", source,
                    TimeUnit.MILLISECONDS, AtomicLong::doubleValue)
            .register(registry);
    assertThat(timeGauge).isInstanceOf(Gauge.class);
}
```

### Integration Tests

**New file:** `src/test/java/dev/linhvu/micrometer/integration/FunctionMetersIntegrationTest.java`

```java
@Test
void shouldReturnNoopFunctionCounter_WhenFilterDenies() {
    registry.meterFilter(MeterFilter.deny(id -> id.getName().startsWith("denied")));
    FunctionCounter counter = FunctionCounter.builder("denied.counter", source, AtomicLong::doubleValue)
            .register(registry);
    assertThat(counter).isInstanceOf(NoopFunctionCounter.class);
}

@Test
void shouldThrow_WhenRegisteringFunctionCounterWithSameNameAsCounter() {
    registry.counter("conflict", "type", "regular");
    assertThatThrownBy(() ->
            FunctionCounter.builder("conflict", source, AtomicLong::doubleValue)
                    .tag("type", "regular").register(registry))
            .isInstanceOf(IllegalArgumentException.class);
}

@Test
void shouldWrapExternalMetricsSource() {
    // Full connection pool wrapping test — see src/test for details
}
```

**Run:** `./gradlew test` — expected: all tests pass (including prior features' tests)

---

## 11.10 Why This Works

> ★ **Insight** -------------------------------------------
> - **Why WeakReference with last-value caching (not NaN)?** `DefaultGauge` returns `NaN` when its source is GC'd because a gauge samples a *current* state — there's no meaningful "last" value when the source no longer exists. But `CumulativeFunctionCounter` returns the *last known count* because counters track cumulative totals — the final count is the historical truth, not stale data. This asymmetry is intentional and mirrors the semantic difference between gauges (instantaneous) and counters (cumulative).
> - **When NOT to use function meters:** If you control the instrumentation code, prefer regular Counter/Timer — they're simpler, more efficient (DoubleAdder is lock-free), and don't require you to maintain a separate state object. Function meters exist specifically for wrapping third-party libraries where you *cannot* inject Micrometer's recording API.
> -----------------------------------------------------------

> ★ **Insight** -------------------------------------------
> - **Why the `More` namespace?** The registry's main API (counter/gauge/timer/summary) covers 90%+ of use cases. Function meters are specialized — used primarily for library wrappers. By grouping them behind `more()`, Micrometer keeps autocomplete clean and signals to users that these are advanced types. The real Micrometer also puts `longTaskTimer()`, custom `Meter` registration, and other rare operations in `More`.
> - **Alternative considered: top-level methods.** Putting `functionCounter()` directly on `MeterRegistry` would make the API surface wider without adding value. Most users would see methods they never need, making the "which counter method?" question harder.
> -----------------------------------------------------------

> ★ **Insight** -------------------------------------------
> - **Why `TimeGauge extends Gauge`?** TimeGauge IS-A Gauge — it's a gauge that happens to measure time. This means the visitor pattern's `visitGauge` handler catches TimeGauge instances, so existing code that iterates over gauges automatically picks up time gauges. The only difference is that registries *aware* of TimeGauge can apply time-unit conversion during export. If TimeGauge were a separate type, every visitor/exporter would need a new branch.
> - **The `TimeUtils.convert` necessity:** JDK's `TimeUnit.convert(long, TimeUnit)` truncates: `TimeUnit.SECONDS.convert(500, TimeUnit.MILLISECONDS)` returns 0. Micrometer needs fractional precision (0.5 seconds), hence the double-based conversion utility.
> -----------------------------------------------------------

## 11.11 What We Enhanced

| Aspect | Before (ch04) | Current (ch11) | Real Framework |
|--------|---------------|----------------|----------------|
| Meter types in registry | Counter, Gauge, Timer, DistributionSummary only — all require explicit recording calls | + FunctionCounter, FunctionTimer, TimeGauge — meters that derive values from external objects via functions | Same types + `LongTaskTimer`, custom `Meter` registration via `More` (`MeterRegistry.java:1084-1261`) |
| `FunctionCounter` interface | Empty stub extending `Meter` | Full interface with `count()`, `measure()`, and `Builder` that routes through `registry.more()` | Same API + `@Nullable` annotations and `@Incubating` supplier convenience (`FunctionCounter.java:27-120`) |
| `FunctionTimer` interface | Empty stub extending `Meter` | Full interface with `count()`, `totalTime(TimeUnit)`, `mean(TimeUnit)`, `baseTimeUnit()`, `measure()`, and `Builder` | Same API + `@Nullable` annotations (`FunctionTimer.java:30-160`) |
| Time conversion | None — all conversions done via `TimeUnit.convert(long)` with truncation | `TimeUtils.convert(double, TimeUnit, TimeUnit)` for fractional precision | Full `TimeUtils` with per-unit conversion methods and duration parsing (`TimeUtils.java:54-170`) |
| Registry API surface | Only top-level `counter()`/`gauge()`/`timer()`/`summary()` | + `more()` accessor providing `counter()`/`timer()`/`timeGauge()` for function meter types | Same `More` inner class also hosts `longTaskTimer()`, custom `Meter`, gauge `register()` (`MeterRegistry.java:1084`) |

## 11.12 Connection to Real Framework

| Simplified Code | Real Framework Code | File:Line | Key Difference |
|----------------|--------------------|-----------| ---------------|
| `FunctionCounter` | `FunctionCounter` | `FunctionCounter.java:27` | Real adds `@Nullable` annotations via jspecify |
| `FunctionCounter.Builder` | `FunctionCounter.Builder` | `FunctionCounter.java:51` | Real adds `@Nullable` on description/baseUnit fields |
| `FunctionTimer` | `FunctionTimer` | `FunctionTimer.java:30` | Real adds `@Nullable` annotations |
| `FunctionTimer.Builder` | `FunctionTimer.Builder` | `FunctionTimer.java:76` | Same structure |
| `TimeGauge` | `TimeGauge` | `TimeGauge.java:29` | Real adds `@Incubating` supplier convenience builder and `StrongReferenceGaugeFunction` |
| `TimeGauge.Builder` | `TimeGauge.Builder` | `TimeGauge.java:80` | Real has `strongReference(boolean)` option |
| `CumulativeFunctionCounter` | `CumulativeFunctionCounter` | `CumulativeFunctionCounter.java:21` | Identical design — WeakReference + volatile last |
| `CumulativeFunctionTimer` | `CumulativeFunctionTimer` | `CumulativeFunctionTimer.java:26` | Identical design — same lastCount/lastTime caching |
| `DefaultTimeGauge` | (anonymous in `MeterRegistry.newTimeGauge`) | `MeterRegistry.java:252-274` | Real creates anonymous `TimeGauge` inline; we extract to a named class for clarity |
| `TimeUtils.convert` | `TimeUtils.convert` | `TimeUtils.java:54` | Real has per-unit switch methods for precision; we use nanoseconds as common denominator |
| `MeterRegistry.More` | `MeterRegistry.More` | `MeterRegistry.java:1084` | Real also hosts `longTaskTimer()`, custom `Meter` registration, gauge convenience |
| `NoopFunctionCounter` | `NoopFunctionCounter` | `NoopFunctionCounter.java:19` | Identical |
| `NoopFunctionTimer` | `NoopFunctionTimer` | `NoopFunctionTimer.java:19` | Identical |
| `NoopTimeGauge` | `NoopTimeGauge` | `NoopTimeGauge.java:19` | Identical |

## 11.13 Complete Code

### Production Code

#### File: `src/main/java/dev/linhvu/micrometer/TimeUtils.java` [NEW]

```java
package dev.linhvu.micrometer;

import java.util.concurrent.TimeUnit;

/**
 * Utility for converting time values between {@link TimeUnit}s using
 * {@code double} precision, avoiding the truncation that
 * {@link TimeUnit#convert(long, TimeUnit)} causes with {@code long} values.
 * <p>
 * JDK's {@code TimeUnit.convert()} operates on longs and truncates toward zero:
 * converting 500 milliseconds to seconds yields 0, not 0.5. This utility
 * converts through nanoseconds as a common denominator to preserve fractional
 * precision.
 */
public final class TimeUtils {

    private TimeUtils() {
    }

    /**
     * Convert a time value from one unit to another with double precision.
     *
     * @param value The time value to convert.
     * @param from  The source time unit.
     * @param to    The destination time unit.
     * @return The converted value.
     */
    public static double convert(double value, TimeUnit from, TimeUnit to) {
        if (from == to) {
            return value;
        }
        // Convert to nanoseconds, then to the target unit
        double nanos = value * nanosPerUnit(from);
        return nanos / nanosPerUnit(to);
    }

    /**
     * Returns the number of nanoseconds per one unit of the given TimeUnit.
     */
    private static double nanosPerUnit(TimeUnit unit) {
        return switch (unit) {
            case NANOSECONDS -> 1.0;
            case MICROSECONDS -> 1_000.0;
            case MILLISECONDS -> 1_000_000.0;
            case SECONDS -> 1_000_000_000.0;
            case MINUTES -> 60_000_000_000.0;
            case HOURS -> 3_600_000_000_000.0;
            case DAYS -> 86_400_000_000_000.0;
        };
    }

}
```

#### File: `src/main/java/dev/linhvu/micrometer/FunctionCounter.java` [MODIFIED]

```java
package dev.linhvu.micrometer;

import java.util.Collections;
import java.util.function.ToDoubleFunction;

/**
 * A counter that tracks a monotonically increasing function.
 * <p>
 * Unlike {@link Counter}, where you explicitly call {@code increment()}, a
 * {@code FunctionCounter} derives its count from an external object via a
 * {@link ToDoubleFunction}. This is essential for instrumenting libraries that
 * already maintain their own counters (JMX beans, Kafka consumer metrics, etc.)
 * — you wrap the existing counter rather than duplicating the bookkeeping.
 * <p>
 * The observed object is held via a {@link java.lang.ref.WeakReference} by
 * default, so function counter registration will not prevent garbage collection.
 * When the object is collected, {@link #count()} returns the last known value.
 */
public interface FunctionCounter extends Meter {

    /**
     * Create a fluent builder for a function counter.
     *
     * @param name The counter's name.
     * @param obj  The state object to observe (held via WeakReference).
     * @param f    A function that extracts a monotonically increasing count.
     * @param <T>  The type of the state object.
     * @return A new builder.
     */
    static <T> Builder<T> builder(String name, T obj, ToDoubleFunction<T> f) {
        return new Builder<>(name, obj, f);
    }

    /**
     * @return The cumulative count since this counter was created.
     */
    double count();

    @Override
    default Iterable<Measurement> measure() {
        return Collections.singletonList(new Measurement(this::count, Statistic.COUNT));
    }

    /**
     * Fluent builder for function counters.
     *
     * @param <T> The type of the state object from which the counter value is extracted.
     */
    class Builder<T> {

        private final String name;

        private final T obj;

        private final ToDoubleFunction<T> f;

        private Tags tags = Tags.empty();

        private String description;

        private String baseUnit;

        private Builder(String name, T obj, ToDoubleFunction<T> f) {
            this.name = name;
            this.obj = obj;
            this.f = f;
        }

        /**
         * @param tags Must be an even number of arguments representing key/value pairs.
         * @return This builder with added tags.
         */
        public Builder<T> tags(String... tags) {
            return tags(Tags.of(tags));
        }

        /**
         * @param tags Tags to add to the eventual function counter.
         * @return This builder with added tags.
         */
        public Builder<T> tags(Iterable<Tag> tags) {
            this.tags = this.tags.and(tags);
            return this;
        }

        /**
         * @param key   The tag key.
         * @param value The tag value.
         * @return This builder with a single added tag.
         */
        public Builder<T> tag(String key, String value) {
            this.tags = tags.and(key, value);
            return this;
        }

        /**
         * @param description Description text of the eventual function counter.
         * @return This builder with added description.
         */
        public Builder<T> description(String description) {
            this.description = description;
            return this;
        }

        /**
         * @param unit Base unit of the eventual counter.
         * @return This builder with added base unit.
         */
        public Builder<T> baseUnit(String unit) {
            this.baseUnit = unit;
            return this;
        }

        /**
         * Registers this function counter with the given registry, or retrieves
         * the existing one if a function counter with the same name and tags was
         * already registered.
         *
         * @param registry The registry to register with.
         * @return The registered FunctionCounter.
         */
        public FunctionCounter register(MeterRegistry registry) {
            return registry.more().counter(
                    new Meter.Id(name, tags, Meter.Type.COUNTER, description, baseUnit), obj, f);
        }

    }

}
```

#### File: `src/main/java/dev/linhvu/micrometer/FunctionTimer.java` [MODIFIED]

```java
package dev.linhvu.micrometer;

import java.util.Arrays;
import java.util.concurrent.TimeUnit;
import java.util.function.ToDoubleFunction;
import java.util.function.ToLongFunction;

/**
 * A timer that tracks two monotonically increasing functions: one representing
 * the count of events and one representing the total time spent in every event.
 * <p>
 * Unlike {@link Timer}, where you explicitly call {@code record(duration)}, a
 * {@code FunctionTimer} derives both count and total time from an external object.
 * This is essential for wrapping timers from other libraries (e.g., Kafka's
 * {@code fetch-latency-avg} or connection pool wait times) that already maintain
 * their own aggregated statistics.
 * <p>
 * The observed object is held via a {@link java.lang.ref.WeakReference} by default,
 * so function timer registration will not prevent garbage collection.
 */
public interface FunctionTimer extends Meter {

    /**
     * Create a fluent builder for a function timer.
     *
     * @param name                  The timer's name.
     * @param obj                   The state object to observe.
     * @param countFunction         Extracts a monotonically increasing event count.
     * @param totalTimeFunction     Extracts a monotonically increasing total time.
     * @param totalTimeFunctionUnit The time unit of the total time function's output.
     * @param <T>                   The type of the state object.
     * @return A new builder.
     */
    static <T> Builder<T> builder(String name, T obj, ToLongFunction<T> countFunction,
            ToDoubleFunction<T> totalTimeFunction, TimeUnit totalTimeFunctionUnit) {
        return new Builder<>(name, obj, countFunction, totalTimeFunction, totalTimeFunctionUnit);
    }

    /**
     * @return The total number of occurrences of the timed event.
     */
    double count();

    /**
     * The total time of all occurrences of the timed event.
     *
     * @param unit The base unit of time to scale the total to.
     * @return The total time of all occurrences of the timed event.
     */
    double totalTime(TimeUnit unit);

    /**
     * @param unit The base unit of time to scale the mean to.
     * @return The distribution average for all recorded events.
     */
    default double mean(TimeUnit unit) {
        double count = count();
        return count == 0 ? 0 : totalTime(unit) / count;
    }

    /**
     * @return The base time unit of the timer to which all published metrics
     * will be scaled.
     */
    TimeUnit baseTimeUnit();

    @Override
    default Iterable<Measurement> measure() {
        return Arrays.asList(
                new Measurement(this::count, Statistic.COUNT),
                new Measurement(() -> totalTime(baseTimeUnit()), Statistic.TOTAL_TIME));
    }

    /**
     * Fluent builder for function timers.
     *
     * @param <T> The type of the state object from which the timer values are extracted.
     */
    class Builder<T> {

        private final String name;

        private final T obj;

        private final ToLongFunction<T> countFunction;

        private final ToDoubleFunction<T> totalTimeFunction;

        private final TimeUnit totalTimeFunctionUnit;

        private Tags tags = Tags.empty();

        private String description;

        private Builder(String name, T obj, ToLongFunction<T> countFunction,
                ToDoubleFunction<T> totalTimeFunction, TimeUnit totalTimeFunctionUnit) {
            this.name = name;
            this.obj = obj;
            this.countFunction = countFunction;
            this.totalTimeFunction = totalTimeFunction;
            this.totalTimeFunctionUnit = totalTimeFunctionUnit;
        }

        /**
         * @param tags Must be an even number of arguments representing key/value pairs.
         * @return This builder with added tags.
         */
        public Builder<T> tags(String... tags) {
            return tags(Tags.of(tags));
        }

        /**
         * @param tags Tags to add to the eventual function timer.
         * @return This builder with added tags.
         */
        public Builder<T> tags(Iterable<Tag> tags) {
            this.tags = this.tags.and(tags);
            return this;
        }

        /**
         * @param key   The tag key.
         * @param value The tag value.
         * @return This builder with a single added tag.
         */
        public Builder<T> tag(String key, String value) {
            this.tags = tags.and(key, value);
            return this;
        }

        /**
         * @param description Description text of the eventual function timer.
         * @return This builder with added description.
         */
        public Builder<T> description(String description) {
            this.description = description;
            return this;
        }

        /**
         * Registers this function timer with the given registry, or retrieves
         * the existing one if a function timer with the same name and tags was
         * already registered.
         *
         * @param registry The registry to register with.
         * @return The registered FunctionTimer.
         */
        public FunctionTimer register(MeterRegistry registry) {
            return registry.more().timer(
                    new Meter.Id(name, tags, Meter.Type.TIMER, description, null),
                    obj, countFunction, totalTimeFunction, totalTimeFunctionUnit);
        }

    }

}
```

#### File: `src/main/java/dev/linhvu/micrometer/TimeGauge.java` [NEW]

```java
package dev.linhvu.micrometer;

import java.util.concurrent.TimeUnit;
import java.util.function.ToDoubleFunction;

/**
 * A specialized {@link Gauge} that tracks a time value, to be scaled to the
 * base unit of time expected by each registry implementation.
 * <p>
 * When you record a value in milliseconds but the registry works in seconds,
 * the TimeGauge handles the conversion automatically. This is useful for
 * monitoring JVM uptime, connection pool wait times, or any other time-based
 * gauge where the source and destination units may differ.
 * <p>
 * TimeGauge extends Gauge — it IS-A Gauge with additional time-unit awareness.
 * This means it can be used anywhere a Gauge is expected, but registries that
 * know about TimeGauge can apply time-unit conversion during export.
 */
public interface TimeGauge extends Gauge {

    /**
     * Create a fluent builder for a time gauge.
     *
     * @param name   The time gauge's name.
     * @param obj    The state object to observe (held via WeakReference).
     * @param fUnits The time unit of the function's output.
     * @param f      A function that extracts a time value from the state object.
     * @param <T>    The type of the state object.
     * @return A new builder.
     */
    static <T> Builder<T> builder(String name, T obj, TimeUnit fUnits, ToDoubleFunction<T> f) {
        return new Builder<>(name, obj, fUnits, f);
    }

    /**
     * @return The base time unit of the time gauge to which all published
     * metrics will be scaled.
     */
    TimeUnit baseTimeUnit();

    /**
     * The act of observing the value by calling this method triggers sampling
     * of the underlying number or user-defined function that defines the value
     * for the gauge.
     *
     * @param unit The base unit of time to scale the value to.
     * @return The current value, scaled to the appropriate base unit.
     */
    default double value(TimeUnit unit) {
        return TimeUtils.convert(value(), baseTimeUnit(), unit);
    }

    /**
     * Fluent builder for time gauges.
     *
     * @param <T> The type of the state object being observed.
     */
    class Builder<T> {

        private final String name;

        private final T obj;

        private final TimeUnit fUnits;

        private final ToDoubleFunction<T> f;

        private Tags tags = Tags.empty();

        private String description;

        private Builder(String name, T obj, TimeUnit fUnits, ToDoubleFunction<T> f) {
            this.name = name;
            this.obj = obj;
            this.fUnits = fUnits;
            this.f = f;
        }

        /**
         * @param tags Must be an even number of arguments representing key/value pairs.
         * @return This builder with added tags.
         */
        public Builder<T> tags(String... tags) {
            return tags(Tags.of(tags));
        }

        /**
         * @param tags Tags to add to the eventual time gauge.
         * @return This builder with added tags.
         */
        public Builder<T> tags(Iterable<Tag> tags) {
            this.tags = this.tags.and(tags);
            return this;
        }

        /**
         * @param key   The tag key.
         * @param value The tag value.
         * @return This builder with a single added tag.
         */
        public Builder<T> tag(String key, String value) {
            this.tags = tags.and(key, value);
            return this;
        }

        /**
         * @param description Description text of the eventual time gauge.
         * @return This builder with added description.
         */
        public Builder<T> description(String description) {
            this.description = description;
            return this;
        }

        /**
         * Registers this time gauge with the given registry, or retrieves
         * the existing one if a time gauge with the same name and tags was
         * already registered.
         *
         * @param registry The registry to register with.
         * @return The registered TimeGauge.
         */
        public TimeGauge register(MeterRegistry registry) {
            return registry.more().timeGauge(
                    new Meter.Id(name, tags, Meter.Type.GAUGE, description, null),
                    obj, fUnits, f);
        }

    }

}
```

#### File: `src/main/java/dev/linhvu/micrometer/cumulative/CumulativeFunctionCounter.java` [NEW]

```java
package dev.linhvu.micrometer.cumulative;

import dev.linhvu.micrometer.AbstractMeter;
import dev.linhvu.micrometer.FunctionCounter;
import dev.linhvu.micrometer.Meter;

import java.lang.ref.WeakReference;
import java.util.function.ToDoubleFunction;

/**
 * A {@link FunctionCounter} that reads the current count from an external object
 * via a {@link ToDoubleFunction}, holding the object via a {@link WeakReference}.
 * <p>
 * When the observed object is garbage collected, this counter returns the last
 * successfully read value. This differs from {@code DefaultGauge} (which returns
 * {@code NaN}) because a counter's last value is still meaningful — it represents
 * the total count up to the point the source disappeared.
 */
public class CumulativeFunctionCounter<T> extends AbstractMeter implements FunctionCounter {

    private final WeakReference<T> ref;

    private final ToDoubleFunction<T> f;

    private volatile double last;

    public CumulativeFunctionCounter(Meter.Id id, T obj, ToDoubleFunction<T> f) {
        super(id);
        this.ref = new WeakReference<>(obj);
        this.f = f;
    }

    @Override
    public double count() {
        T obj = ref.get();
        return obj != null ? (last = f.applyAsDouble(obj)) : last;
    }

}
```

#### File: `src/main/java/dev/linhvu/micrometer/cumulative/CumulativeFunctionTimer.java` [NEW]

```java
package dev.linhvu.micrometer.cumulative;

import dev.linhvu.micrometer.AbstractMeter;
import dev.linhvu.micrometer.FunctionTimer;
import dev.linhvu.micrometer.TimeUtils;

import java.lang.ref.WeakReference;
import java.util.concurrent.TimeUnit;
import java.util.function.ToDoubleFunction;
import java.util.function.ToLongFunction;

/**
 * A {@link FunctionTimer} that reads count and total time from an external
 * object via functions, holding the object via a {@link WeakReference}.
 * <p>
 * Both values are stored in the registry's base time unit. When the observed
 * object is garbage collected, this timer returns the last successfully read
 * values for both count and total time.
 * <p>
 * Negative values are clamped to 0 (via {@code Math.max}) because counts
 * and durations are inherently non-negative.
 */
public class CumulativeFunctionTimer<T> extends AbstractMeter implements FunctionTimer {

    private final WeakReference<T> ref;

    private final ToLongFunction<T> countFunction;

    private final ToDoubleFunction<T> totalTimeFunction;

    private final TimeUnit totalTimeFunctionUnit;

    private final TimeUnit baseTimeUnit;

    private volatile long lastCount;

    private volatile double lastTime;

    public CumulativeFunctionTimer(Meter.Id id, T obj, ToLongFunction<T> countFunction,
            ToDoubleFunction<T> totalTimeFunction, TimeUnit totalTimeFunctionUnit,
            TimeUnit baseTimeUnit) {
        super(id);
        this.ref = new WeakReference<>(obj);
        this.countFunction = countFunction;
        this.totalTimeFunction = totalTimeFunction;
        this.totalTimeFunctionUnit = totalTimeFunctionUnit;
        this.baseTimeUnit = baseTimeUnit;
    }

    @Override
    public double count() {
        T obj = ref.get();
        return obj != null ? (lastCount = Math.max(countFunction.applyAsLong(obj), 0)) : lastCount;
    }

    @Override
    public double totalTime(TimeUnit unit) {
        T obj = ref.get();
        if (obj != null) {
            lastTime = Math.max(
                    TimeUtils.convert(totalTimeFunction.applyAsDouble(obj),
                            totalTimeFunctionUnit, baseTimeUnit()),
                    0);
        }
        return TimeUtils.convert(lastTime, baseTimeUnit(), unit);
    }

    @Override
    public TimeUnit baseTimeUnit() {
        return baseTimeUnit;
    }

}
```

#### File: `src/main/java/dev/linhvu/micrometer/internal/DefaultTimeGauge.java` [NEW]

```java
package dev.linhvu.micrometer.internal;

import dev.linhvu.micrometer.AbstractMeter;
import dev.linhvu.micrometer.Meter;
import dev.linhvu.micrometer.TimeGauge;
import dev.linhvu.micrometer.TimeUtils;

import java.lang.ref.WeakReference;
import java.util.concurrent.TimeUnit;
import java.util.function.ToDoubleFunction;

/**
 * Default {@link TimeGauge} implementation that holds a {@link WeakReference}
 * to the observed object and a {@link ToDoubleFunction} to extract a time value.
 * <p>
 * The value is stored in the source function's time unit ({@code fUnits}). When
 * {@link #value()} is called, the raw value is converted to the registry's base
 * time unit. When {@link #value(TimeUnit)} is called, the conversion goes through
 * the base time unit.
 * <p>
 * Like {@link DefaultGauge}, if the observed object is garbage collected,
 * the time gauge returns {@code NaN}.
 *
 * @param <T> The type of the state object being observed.
 */
public class DefaultTimeGauge<T> extends AbstractMeter implements TimeGauge {

    private final WeakReference<T> ref;

    private final ToDoubleFunction<T> f;

    private final TimeUnit fUnits;

    private final TimeUnit baseTimeUnit;

    public DefaultTimeGauge(Meter.Id id, T obj, TimeUnit fUnits, ToDoubleFunction<T> f,
            TimeUnit baseTimeUnit) {
        super(id);
        this.ref = new WeakReference<>(obj);
        this.f = f;
        this.fUnits = fUnits;
        this.baseTimeUnit = baseTimeUnit;
    }

    @Override
    public TimeUnit baseTimeUnit() {
        return baseTimeUnit;
    }

    @Override
    public double value() {
        T obj = ref.get();
        if (obj != null) {
            try {
                return TimeUtils.convert(f.applyAsDouble(obj), fUnits, baseTimeUnit);
            }
            catch (Exception e) {
                return Double.NaN;
            }
        }
        return Double.NaN;
    }

}
```

#### File: `src/main/java/dev/linhvu/micrometer/noop/NoopFunctionCounter.java` [NEW]

```java
package dev.linhvu.micrometer.noop;

import dev.linhvu.micrometer.FunctionCounter;
import dev.linhvu.micrometer.Meter;

/**
 * No-op {@link FunctionCounter} that always returns 0. Returned when
 * the registry is closed or a {@code MeterFilter} denies registration.
 */
public class NoopFunctionCounter extends NoopMeter implements FunctionCounter {

    public NoopFunctionCounter(Meter.Id id) {
        super(id);
    }

    @Override
    public double count() {
        return 0;
    }

}
```

#### File: `src/main/java/dev/linhvu/micrometer/noop/NoopFunctionTimer.java` [NEW]

```java
package dev.linhvu.micrometer.noop;

import dev.linhvu.micrometer.FunctionTimer;
import dev.linhvu.micrometer.Meter;

import java.util.concurrent.TimeUnit;

/**
 * No-op {@link FunctionTimer} that always returns 0. Returned when
 * the registry is closed or a {@code MeterFilter} denies registration.
 */
public class NoopFunctionTimer extends NoopMeter implements FunctionTimer {

    public NoopFunctionTimer(Meter.Id id) {
        super(id);
    }

    @Override
    public double count() {
        return 0;
    }

    @Override
    public double totalTime(TimeUnit unit) {
        return 0;
    }

    @Override
    public TimeUnit baseTimeUnit() {
        return TimeUnit.NANOSECONDS;
    }

}
```

#### File: `src/main/java/dev/linhvu/micrometer/noop/NoopTimeGauge.java` [NEW]

```java
package dev.linhvu.micrometer.noop;

import dev.linhvu.micrometer.Meter;
import dev.linhvu.micrometer.TimeGauge;

import java.util.concurrent.TimeUnit;

/**
 * No-op {@link TimeGauge} that always returns 0. Returned when
 * the registry is closed or a {@code MeterFilter} denies registration.
 */
public class NoopTimeGauge extends NoopMeter implements TimeGauge {

    public NoopTimeGauge(Meter.Id id) {
        super(id);
    }

    @Override
    public TimeUnit baseTimeUnit() {
        return TimeUnit.NANOSECONDS;
    }

    @Override
    public double value() {
        return 0;
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

/**
 * Central registry that manages meter lifecycle — creation, deduplication, and lookup.
 * <p>
 * Uses the <b>Template Method</b> pattern: this class defines the complete registration
 * lifecycle (ID construction, deduplication via ConcurrentHashMap, factory dispatch),
 * while subclasses implement factory methods like {@code newCounter()} and {@code newGauge()}.
 * <p>
 * The deduplication algorithm uses <b>double-check locking</b>:
 * <ol>
 *     <li>Lock-free fast path: read from ConcurrentHashMap</li>
 *     <li>Synchronized slow path: re-check, create, and store the new meter</li>
 * </ol>
 * This ensures that {@code Counter.builder("x").register(registry)} called from multiple
 * threads always returns the same Counter instance.
 * <p>
 * If a meter is already registered with a given name+tags but as a different type
 * (e.g., trying to register a Timer when a Counter already exists), an
 * {@link IllegalArgumentException} is thrown.
 */
public abstract class MeterRegistry {

    protected final Clock clock;

    private final ConcurrentHashMap<Meter.Id, Meter> meterMap = new ConcurrentHashMap<>();

    private final Object meterMapLock = new Object();

    /**
     * Volatile array of filters — enables lock-free iteration during meter registration
     * while still allowing thread-safe mutation via {@link #meterFilter(MeterFilter)}.
     * Copy-on-write: adding a filter creates a new array.
     */
    private volatile MeterFilter[] filters = new MeterFilter[0];

    private volatile NamingConvention namingConvention = NamingConvention.identity;

    private volatile boolean closed = false;

    protected MeterRegistry(Clock clock) {
        this.clock = Objects.requireNonNull(clock, "clock must not be null");
    }

    // -----------------------------------------------------------------------
    // NamingConvention configuration
    // -----------------------------------------------------------------------

    public NamingConvention getNamingConvention() {
        return namingConvention;
    }

    public MeterRegistry namingConvention(NamingConvention namingConvention) {
        this.namingConvention = Objects.requireNonNull(namingConvention, "namingConvention must not be null");
        return this;
    }

    // -----------------------------------------------------------------------
    // Filter configuration
    // -----------------------------------------------------------------------

    public synchronized MeterRegistry meterFilter(MeterFilter filter) {
        Objects.requireNonNull(filter, "filter must not be null");
        MeterFilter[] newFilters = Arrays.copyOf(filters, filters.length + 1);
        newFilters[filters.length] = filter;
        filters = newFilters;
        return this;
    }

    // -----------------------------------------------------------------------
    // Public convenience registration methods
    // -----------------------------------------------------------------------

    public Counter counter(String name, String... tags) {
        return Counter.builder(name).tags(tags).register(this);
    }

    public Counter counter(String name, Iterable<Tag> tags) {
        return Counter.builder(name).tags(tags).register(this);
    }

    public <T> T gauge(String name, Iterable<Tag> tags, T stateObject, ToDoubleFunction<T> valueFunction) {
        Gauge.builder(name, stateObject, valueFunction).tags(tags).register(this);
        return stateObject;
    }

    public <T extends Number> T gauge(String name, Iterable<Tag> tags, T number) {
        return gauge(name, tags, number, Number::doubleValue);
    }

    public <T extends Number> T gauge(String name, T number) {
        return gauge(name, Collections.emptyList(), number);
    }

    public <T extends Collection<?>> T gaugeCollectionSize(String name, Iterable<Tag> tags, T collection) {
        return gauge(name, tags, collection, Collection::size);
    }

    public <T extends Map<?, ?>> T gaugeMapSize(String name, Iterable<Tag> tags, T map) {
        return gauge(name, tags, map, Map::size);
    }

    public Timer timer(String name, String... tags) {
        return Timer.builder(name).tags(tags).register(this);
    }

    public Timer timer(String name, Iterable<Tag> tags) {
        return Timer.builder(name).tags(tags).register(this);
    }

    public DistributionSummary summary(String name, String... tags) {
        return DistributionSummary.builder(name).tags(tags).register(this);
    }

    public DistributionSummary summary(String name, Iterable<Tag> tags) {
        return DistributionSummary.builder(name).tags(tags).register(this);
    }

    // -----------------------------------------------------------------------
    // Package-private registration (called by Builder.register())
    // -----------------------------------------------------------------------

    Counter counter(Meter.Id id) {
        return registerMeterIfNecessary(Counter.class, id, this::newCounter, NoopCounter::new);
    }

    <T> Gauge gauge(Meter.Id id, T obj, ToDoubleFunction<T> valueFunction) {
        return registerMeterIfNecessary(Gauge.class, id,
                mappedId -> newGauge(mappedId, obj, valueFunction), NoopGauge::new);
    }

    Timer timer(Meter.Id id, DistributionStatisticConfig distributionStatisticConfig) {
        return registerMeterIfNecessary(Timer.class, id,
                mappedId -> newTimer(mappedId,
                        configureDistributionStatistics(mappedId, distributionStatisticConfig)),
                NoopTimer::new);
    }

    DistributionSummary summary(Meter.Id id, double scale,
            DistributionStatisticConfig distributionStatisticConfig) {
        return registerMeterIfNecessary(DistributionSummary.class, id,
                mappedId -> newDistributionSummary(mappedId, scale,
                        configureDistributionStatistics(mappedId, distributionStatisticConfig)),
                NoopDistributionSummary::new);
    }

    // -----------------------------------------------------------------------
    // Template Method factories — subclasses implement these
    // -----------------------------------------------------------------------

    protected abstract Counter newCounter(Meter.Id id);

    protected abstract <T> Gauge newGauge(Meter.Id id, T obj, ToDoubleFunction<T> valueFunction);

    protected abstract Timer newTimer(Meter.Id id,
            DistributionStatisticConfig distributionStatisticConfig);

    protected abstract DistributionSummary newDistributionSummary(Meter.Id id, double scale,
            DistributionStatisticConfig distributionStatisticConfig);

    protected abstract <T> FunctionCounter newFunctionCounter(Meter.Id id, T obj,
            ToDoubleFunction<T> countFunction);

    protected abstract <T> FunctionTimer newFunctionTimer(Meter.Id id, T obj,
            ToLongFunction<T> countFunction, ToDoubleFunction<T> totalTimeFunction,
            TimeUnit totalTimeFunctionUnit);

    protected <T> TimeGauge newTimeGauge(Meter.Id id, T obj, TimeUnit timeFunctionUnit,
            ToDoubleFunction<T> f) {
        return new dev.linhvu.micrometer.internal.DefaultTimeGauge<>(id, obj, timeFunctionUnit,
                f, getBaseTimeUnit());
    }

    protected abstract TimeUnit getBaseTimeUnit();

    protected DistributionStatisticConfig defaultHistogramConfig() {
        return DistributionStatisticConfig.DEFAULT;
    }

    private DistributionStatisticConfig configureDistributionStatistics(Meter.Id id,
            DistributionStatisticConfig config) {
        for (MeterFilter filter : filters) {
            DistributionStatisticConfig filterConfig = filter.configure(id, config);
            if (filterConfig != null) {
                config = filterConfig.merge(config);
            }
        }
        return config.merge(defaultHistogramConfig());
    }

    // -----------------------------------------------------------------------
    // Registration lifecycle — the heart of deduplication
    // -----------------------------------------------------------------------

    private <M extends Meter> M registerMeterIfNecessary(Class<M> meterClass, Meter.Id id,
            Function<Meter.Id, ? extends Meter> meterFactory,
            Function<Meter.Id, ? extends Meter> noopFactory) {

        Meter.Id mappedId = mapId(id);

        if (!accept(mappedId)) {
            return meterClass.cast(noopFactory.apply(mappedId));
        }

        Meter meter = getOrCreateMeter(mappedId, meterFactory, noopFactory);

        if (!meterClass.isInstance(meter)) {
            throw new IllegalArgumentException(
                    "There is already a meter registered with name '" + mappedId.getName()
                            + "' and tags " + mappedId.getTagsAsIterable()
                            + " but with type " + meter.getClass().getSimpleName()
                            + ". Attempted to register type " + meterClass.getSimpleName() + ".");
        }

        return meterClass.cast(meter);
    }

    private Meter.Id mapId(Meter.Id id) {
        Meter.Id mappedId = id;
        for (MeterFilter filter : filters) {
            mappedId = filter.map(mappedId);
        }
        return mappedId;
    }

    private boolean accept(Meter.Id id) {
        for (MeterFilter filter : filters) {
            MeterFilterReply reply = filter.accept(id);
            if (reply == MeterFilterReply.DENY) {
                return false;
            }
            if (reply == MeterFilterReply.ACCEPT) {
                return true;
            }
        }
        return true;
    }

    private Meter getOrCreateMeter(Meter.Id id,
            Function<Meter.Id, ? extends Meter> meterFactory,
            Function<Meter.Id, ? extends Meter> noopFactory) {

        Meter existing = meterMap.get(id);
        if (existing != null) {
            return existing;
        }

        synchronized (meterMapLock) {
            existing = meterMap.get(id);
            if (existing != null) {
                return existing;
            }

            if (closed) {
                return noopFactory.apply(id);
            }

            Meter newMeter = meterFactory.apply(id);
            meterMap.put(id, newMeter);
            return newMeter;
        }
    }

    // -----------------------------------------------------------------------
    // "More" accessor — less common meter types
    // -----------------------------------------------------------------------

    private final More more = new More();

    /**
     * Access less common meter types through a dedicated namespace.
     * <p>
     * This keeps the main registry API clean (counter/gauge/timer/summary) while
     * providing a discoverable path to advanced types:
     * <pre>
     *     registry.more().counter("cache.evictions", cache, Cache::evictionCount);
     *     registry.more().timer("pool.wait", pool, Pool::waitCount, Pool::totalWaitTime, MILLISECONDS);
     *     registry.more().timeGauge("jvm.uptime", runtime, MILLISECONDS, RuntimeMXBean::getUptime);
     * </pre>
     */
    public More more() {
        return more;
    }

    /**
     * Provides registration methods for less common meter types: {@link FunctionCounter},
     * {@link FunctionTimer}, and {@link TimeGauge}.
     * <p>
     * These meters derive their values from external objects via functions — they are
     * essential for instrumenting libraries that already maintain their own statistics.
     */
    public class More {

        <T> FunctionCounter counter(Meter.Id id, T obj, ToDoubleFunction<T> countFunction) {
            return registerMeterIfNecessary(FunctionCounter.class, id,
                    mappedId -> newFunctionCounter(mappedId, obj, countFunction),
                    NoopFunctionCounter::new);
        }

        public <T> FunctionCounter counter(String name, Iterable<Tag> tags, T obj,
                ToDoubleFunction<T> countFunction) {
            return FunctionCounter.builder(name, obj, countFunction).tags(tags).register(MeterRegistry.this);
        }

        <T> FunctionTimer timer(Meter.Id id, T obj, ToLongFunction<T> countFunction,
                ToDoubleFunction<T> totalTimeFunction, TimeUnit totalTimeFunctionUnit) {
            return registerMeterIfNecessary(FunctionTimer.class,
                    id.withBaseUnit(getBaseTimeUnit().name().toLowerCase()),
                    mappedId -> newFunctionTimer(mappedId, obj, countFunction,
                            totalTimeFunction, totalTimeFunctionUnit),
                    NoopFunctionTimer::new);
        }

        public <T> FunctionTimer timer(String name, Iterable<Tag> tags, T obj,
                ToLongFunction<T> countFunction, ToDoubleFunction<T> totalTimeFunction,
                TimeUnit totalTimeFunctionUnit) {
            return FunctionTimer.builder(name, obj, countFunction, totalTimeFunction,
                    totalTimeFunctionUnit).tags(tags).register(MeterRegistry.this);
        }

        <T> TimeGauge timeGauge(Meter.Id id, T obj, TimeUnit timeFunctionUnit,
                ToDoubleFunction<T> timeFunction) {
            return registerMeterIfNecessary(TimeGauge.class, id,
                    mappedId -> newTimeGauge(mappedId, obj, timeFunctionUnit, timeFunction),
                    NoopTimeGauge::new);
        }

        public <T> TimeGauge timeGauge(String name, Iterable<Tag> tags, T obj,
                TimeUnit timeFunctionUnit, ToDoubleFunction<T> timeFunction) {
            return TimeGauge.builder(name, obj, timeFunctionUnit, timeFunction)
                    .tags(tags).register(MeterRegistry.this);
        }

    }

    // -----------------------------------------------------------------------
    // Query methods
    // -----------------------------------------------------------------------

    public List<Meter> getMeters() {
        return Collections.unmodifiableList(new ArrayList<>(meterMap.values()));
    }

    public Clock getClock() {
        return clock;
    }

    // -----------------------------------------------------------------------
    // Lifecycle
    // -----------------------------------------------------------------------

    public boolean isClosed() {
        return closed;
    }

    public void close() {
        closed = true;
    }

}
```

#### File: `src/main/java/dev/linhvu/micrometer/simple/SimpleMeterRegistry.java` [MODIFIED]

```java
package dev.linhvu.micrometer.simple;

import dev.linhvu.micrometer.Clock;
import dev.linhvu.micrometer.Counter;
import dev.linhvu.micrometer.DistributionSummary;
import dev.linhvu.micrometer.FunctionCounter;
import dev.linhvu.micrometer.FunctionTimer;
import dev.linhvu.micrometer.Gauge;
import dev.linhvu.micrometer.Meter;
import dev.linhvu.micrometer.MeterRegistry;
import dev.linhvu.micrometer.Timer;
import dev.linhvu.micrometer.cumulative.CumulativeCounter;
import dev.linhvu.micrometer.cumulative.CumulativeDistributionSummary;
import dev.linhvu.micrometer.cumulative.CumulativeFunctionCounter;
import dev.linhvu.micrometer.cumulative.CumulativeFunctionTimer;
import dev.linhvu.micrometer.cumulative.CumulativeTimer;
import dev.linhvu.micrometer.distribution.DistributionStatisticConfig;
import dev.linhvu.micrometer.internal.DefaultGauge;

import java.util.concurrent.TimeUnit;
import java.util.function.ToDoubleFunction;
import java.util.function.ToLongFunction;

/**
 * The simplest useful {@link MeterRegistry} — an in-memory implementation backed
 * by cumulative meter types.
 * <p>
 * All meters are cumulative: counters grow forever, timers accumulate total time
 * and count. This is ideal for testing, local development, and as the base
 * implementation for composite registries.
 * <p>
 * Currently supports Counter (via {@link CumulativeCounter}), Gauge (via
 * {@link DefaultGauge}), Timer (via {@link CumulativeTimer}),
 * DistributionSummary (via {@link CumulativeDistributionSummary}),
 * FunctionCounter (via {@link CumulativeFunctionCounter}), and
 * FunctionTimer (via {@link CumulativeFunctionTimer}).
 */
public class SimpleMeterRegistry extends MeterRegistry {

    public SimpleMeterRegistry() {
        this(Clock.SYSTEM);
    }

    public SimpleMeterRegistry(Clock clock) {
        super(clock);
    }

    @Override
    protected Counter newCounter(Meter.Id id) {
        return new CumulativeCounter(id);
    }

    @Override
    protected <T> Gauge newGauge(Meter.Id id, T obj, ToDoubleFunction<T> valueFunction) {
        return new DefaultGauge<>(id, obj, valueFunction);
    }

    @Override
    protected Timer newTimer(Meter.Id id, DistributionStatisticConfig distributionStatisticConfig) {
        return new CumulativeTimer(id, clock, getBaseTimeUnit(), distributionStatisticConfig);
    }

    @Override
    protected DistributionSummary newDistributionSummary(Meter.Id id, double scale,
            DistributionStatisticConfig distributionStatisticConfig) {
        return new CumulativeDistributionSummary(id, clock, scale, distributionStatisticConfig);
    }

    @Override
    protected <T> FunctionCounter newFunctionCounter(Meter.Id id, T obj,
            ToDoubleFunction<T> countFunction) {
        return new CumulativeFunctionCounter<>(id, obj, countFunction);
    }

    @Override
    protected <T> FunctionTimer newFunctionTimer(Meter.Id id, T obj,
            ToLongFunction<T> countFunction, ToDoubleFunction<T> totalTimeFunction,
            TimeUnit totalTimeFunctionUnit) {
        return new CumulativeFunctionTimer<>(id, obj, countFunction, totalTimeFunction,
                totalTimeFunctionUnit, getBaseTimeUnit());
    }

    @Override
    protected TimeUnit getBaseTimeUnit() {
        return TimeUnit.SECONDS;
    }

}
```

### Test Code

#### File: `src/test/java/dev/linhvu/micrometer/TimeUtilsTest.java` [NEW]

```java
package dev.linhvu.micrometer;

import org.junit.jupiter.api.Test;

import java.util.concurrent.TimeUnit;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.within;

class TimeUtilsTest {

    @Test
    void shouldReturnSameValue_WhenUnitsAreEqual() {
        assertThat(TimeUtils.convert(42.0, TimeUnit.SECONDS, TimeUnit.SECONDS))
                .isEqualTo(42.0);
    }

    @Test
    void shouldConvertMillisecondsToSeconds() {
        assertThat(TimeUtils.convert(5000.0, TimeUnit.MILLISECONDS, TimeUnit.SECONDS))
                .isCloseTo(5.0, within(0.0001));
    }

    @Test
    void shouldConvertSecondsToMilliseconds() {
        assertThat(TimeUtils.convert(2.5, TimeUnit.SECONDS, TimeUnit.MILLISECONDS))
                .isCloseTo(2500.0, within(0.01));
    }

    @Test
    void shouldPreserveFractionalPrecision() {
        assertThat(TimeUtils.convert(500.0, TimeUnit.MILLISECONDS, TimeUnit.SECONDS))
                .isCloseTo(0.5, within(0.0001));
    }

    @Test
    void shouldConvertNanosecondsToMinutes() {
        double nanos = 120_000_000_000.0;
        assertThat(TimeUtils.convert(nanos, TimeUnit.NANOSECONDS, TimeUnit.MINUTES))
                .isCloseTo(2.0, within(0.0001));
    }

    @Test
    void shouldConvertHoursToDays() {
        assertThat(TimeUtils.convert(48.0, TimeUnit.HOURS, TimeUnit.DAYS))
                .isCloseTo(2.0, within(0.0001));
    }

}
```

#### File: `src/test/java/dev/linhvu/micrometer/FunctionCounterTest.java` [NEW]

```java
package dev.linhvu.micrometer;

import dev.linhvu.micrometer.simple.SimpleMeterRegistry;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.util.concurrent.atomic.AtomicLong;

import static org.assertj.core.api.Assertions.assertThat;

class FunctionCounterTest {

    private SimpleMeterRegistry registry;

    @BeforeEach
    void setUp() {
        registry = new SimpleMeterRegistry();
    }

    @Test
    void shouldTrackMonotonicallyIncreasingCount() {
        AtomicLong source = new AtomicLong(0);
        FunctionCounter counter = FunctionCounter.builder("requests", source, AtomicLong::doubleValue)
                .register(registry);
        assertThat(counter.count()).isEqualTo(0.0);
        source.set(5);
        assertThat(counter.count()).isEqualTo(5.0);
        source.set(42);
        assertThat(counter.count()).isEqualTo(42.0);
    }

    @Test
    void shouldDeduplicateByNameAndTags() {
        AtomicLong source = new AtomicLong(10);
        FunctionCounter first = FunctionCounter.builder("cache.evictions", source, AtomicLong::doubleValue)
                .tag("cache", "users").register(registry);
        FunctionCounter second = FunctionCounter.builder("cache.evictions", source, AtomicLong::doubleValue)
                .tag("cache", "users").register(registry);
        assertThat(first).isSameAs(second);
    }

    @Test
    void shouldSupportTagsAndDescription() {
        AtomicLong source = new AtomicLong(0);
        FunctionCounter counter = FunctionCounter.builder("cache.hits", source, AtomicLong::doubleValue)
                .tags("cache", "sessions", "region", "us-east")
                .description("Total cache hits").baseUnit("hits").register(registry);
        assertThat(counter.getId().getName()).isEqualTo("cache.hits");
        assertThat(counter.getId().getTag("cache")).isEqualTo("sessions");
        assertThat(counter.getId().getTag("region")).isEqualTo("us-east");
        assertThat(counter.getId().getDescription()).isEqualTo("Total cache hits");
        assertThat(counter.getId().getBaseUnit()).isEqualTo("hits");
    }

    @Test
    void shouldProduceSingleCountMeasurement() {
        AtomicLong source = new AtomicLong(7);
        FunctionCounter counter = FunctionCounter.builder("ops", source, AtomicLong::doubleValue)
                .register(registry);
        var measurements = counter.measure();
        assertThat(measurements).hasSize(1);
        Measurement m = measurements.iterator().next();
        assertThat(m.getStatistic()).isEqualTo(Statistic.COUNT);
        assertThat(m.getValue()).isEqualTo(7.0);
    }

    @Test
    void shouldReturnLastValue_WhenSourceIsGarbageCollected() {
        AtomicLong source = new AtomicLong(99);
        FunctionCounter counter = FunctionCounter.builder("gc.test", source, AtomicLong::doubleValue)
                .register(registry);
        assertThat(counter.count()).isEqualTo(99.0);
        source = null;
        System.gc();
        assertThat(counter.count()).isEqualTo(99.0);
    }

    @Test
    void shouldRegisterViaMoreConvenience() {
        AtomicLong source = new AtomicLong(5);
        FunctionCounter counter = registry.more().counter("direct.counter",
                Tags.of("env", "test"), source, AtomicLong::doubleValue);
        assertThat(counter.count()).isEqualTo(5.0);
        assertThat(counter.getId().getName()).isEqualTo("direct.counter");
        assertThat(counter.getId().getTag("env")).isEqualTo("test");
    }

}
```

#### File: `src/test/java/dev/linhvu/micrometer/FunctionTimerTest.java` [NEW]

```java
package dev.linhvu.micrometer;

import dev.linhvu.micrometer.simple.SimpleMeterRegistry;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.util.concurrent.TimeUnit;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.within;

class FunctionTimerTest {

    private SimpleMeterRegistry registry;

    @BeforeEach
    void setUp() {
        registry = new SimpleMeterRegistry();
    }

    @Test
    void shouldTrackCountAndTotalTime() {
        TestTimerSource source = new TestTimerSource(10, 5000.0);
        FunctionTimer timer = FunctionTimer.builder("http.requests", source,
                        TestTimerSource::count, TestTimerSource::totalTimeMs, TimeUnit.MILLISECONDS)
                .register(registry);
        assertThat(timer.count()).isEqualTo(10.0);
        assertThat(timer.totalTime(TimeUnit.SECONDS)).isCloseTo(5.0, within(0.001));
    }

    @Test
    void shouldConvertTotalTimeToRequestedUnit() {
        TestTimerSource source = new TestTimerSource(1, 2000.0);
        FunctionTimer timer = FunctionTimer.builder("latency", source,
                        TestTimerSource::count, TestTimerSource::totalTimeMs, TimeUnit.MILLISECONDS)
                .register(registry);
        assertThat(timer.totalTime(TimeUnit.SECONDS)).isCloseTo(2.0, within(0.001));
        assertThat(timer.totalTime(TimeUnit.MILLISECONDS)).isCloseTo(2000.0, within(0.1));
    }

    @Test
    void shouldCalculateMean() {
        TestTimerSource source = new TestTimerSource(4, 2000.0);
        FunctionTimer timer = FunctionTimer.builder("ops.duration", source,
                        TestTimerSource::count, TestTimerSource::totalTimeMs, TimeUnit.MILLISECONDS)
                .register(registry);
        assertThat(timer.mean(TimeUnit.SECONDS)).isCloseTo(0.5, within(0.001));
        assertThat(timer.mean(TimeUnit.MILLISECONDS)).isCloseTo(500.0, within(0.1));
    }

    @Test
    void shouldReturnZeroMean_WhenCountIsZero() {
        TestTimerSource source = new TestTimerSource(0, 0.0);
        FunctionTimer timer = FunctionTimer.builder("empty.timer", source,
                        TestTimerSource::count, TestTimerSource::totalTimeMs, TimeUnit.MILLISECONDS)
                .register(registry);
        assertThat(timer.mean(TimeUnit.SECONDS)).isEqualTo(0.0);
    }

    @Test
    void shouldDeduplicateByNameAndTags() {
        TestTimerSource source = new TestTimerSource(1, 100.0);
        FunctionTimer first = FunctionTimer.builder("dedup.timer", source,
                        TestTimerSource::count, TestTimerSource::totalTimeMs, TimeUnit.MILLISECONDS)
                .tag("service", "api").register(registry);
        FunctionTimer second = FunctionTimer.builder("dedup.timer", source,
                        TestTimerSource::count, TestTimerSource::totalTimeMs, TimeUnit.MILLISECONDS)
                .tag("service", "api").register(registry);
        assertThat(first).isSameAs(second);
    }

    @Test
    void shouldProduceCountAndTotalTimeMeasurements() {
        TestTimerSource source = new TestTimerSource(3, 1500.0);
        FunctionTimer timer = FunctionTimer.builder("measured", source,
                        TestTimerSource::count, TestTimerSource::totalTimeMs, TimeUnit.MILLISECONDS)
                .register(registry);
        var measurements = timer.measure();
        assertThat(measurements).hasSize(2);
        var list = new java.util.ArrayList<Measurement>();
        measurements.forEach(list::add);
        Measurement countMeasurement = list.stream()
                .filter(m -> m.getStatistic() == Statistic.COUNT).findFirst().orElseThrow();
        assertThat(countMeasurement.getValue()).isEqualTo(3.0);
        Measurement totalMeasurement = list.stream()
                .filter(m -> m.getStatistic() == Statistic.TOTAL_TIME).findFirst().orElseThrow();
        assertThat(totalMeasurement.getValue()).isCloseTo(1.5, within(0.001));
    }

    @Test
    void shouldReturnBaseTimeUnit() {
        TestTimerSource source = new TestTimerSource(1, 100.0);
        FunctionTimer timer = FunctionTimer.builder("unit.timer", source,
                        TestTimerSource::count, TestTimerSource::totalTimeMs, TimeUnit.MILLISECONDS)
                .register(registry);
        assertThat(timer.baseTimeUnit()).isEqualTo(TimeUnit.SECONDS);
    }

    @Test
    void shouldClampNegativeValues() {
        TestTimerSource source = new TestTimerSource(-5, -100.0);
        FunctionTimer timer = FunctionTimer.builder("clamped", source,
                        TestTimerSource::count, TestTimerSource::totalTimeMs, TimeUnit.MILLISECONDS)
                .register(registry);
        assertThat(timer.count()).isEqualTo(0.0);
        assertThat(timer.totalTime(TimeUnit.SECONDS)).isEqualTo(0.0);
    }

    @Test
    void shouldRegisterViaMoreConvenience() {
        TestTimerSource source = new TestTimerSource(3, 900.0);
        FunctionTimer timer = registry.more().timer("direct.timer",
                Tags.of("type", "db"), source,
                TestTimerSource::count, TestTimerSource::totalTimeMs, TimeUnit.MILLISECONDS);
        assertThat(timer.count()).isEqualTo(3.0);
        assertThat(timer.totalTime(TimeUnit.MILLISECONDS)).isCloseTo(900.0, within(0.1));
        assertThat(timer.getId().getTag("type")).isEqualTo("db");
    }

    @Test
    void shouldReturnLastValue_WhenSourceIsGarbageCollected() {
        TestTimerSource source = new TestTimerSource(10, 5000.0);
        FunctionTimer timer = FunctionTimer.builder("gc.timer", source,
                        TestTimerSource::count, TestTimerSource::totalTimeMs, TimeUnit.MILLISECONDS)
                .register(registry);
        assertThat(timer.count()).isEqualTo(10.0);
        assertThat(timer.totalTime(TimeUnit.SECONDS)).isCloseTo(5.0, within(0.001));
        source = null;
        System.gc();
        assertThat(timer.count()).isEqualTo(10.0);
        assertThat(timer.totalTime(TimeUnit.SECONDS)).isCloseTo(5.0, within(0.001));
    }

    static class TestTimerSource {
        private final long count;
        private final double totalTimeMs;
        TestTimerSource(long count, double totalTimeMs) {
            this.count = count;
            this.totalTimeMs = totalTimeMs;
        }
        long count() { return count; }
        double totalTimeMs() { return totalTimeMs; }
    }

}
```

#### File: `src/test/java/dev/linhvu/micrometer/TimeGaugeTest.java` [NEW]

```java
package dev.linhvu.micrometer;

import dev.linhvu.micrometer.simple.SimpleMeterRegistry;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.util.concurrent.TimeUnit;
import java.util.concurrent.atomic.AtomicLong;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.within;

class TimeGaugeTest {

    private SimpleMeterRegistry registry;

    @BeforeEach
    void setUp() {
        registry = new SimpleMeterRegistry();
    }

    @Test
    void shouldConvertTimeValueToRegistryBaseUnit() {
        AtomicLong uptimeMs = new AtomicLong(60_000);
        TimeGauge gauge = TimeGauge.builder("jvm.uptime", uptimeMs,
                        TimeUnit.MILLISECONDS, AtomicLong::doubleValue)
                .register(registry);
        assertThat(gauge.value()).isCloseTo(60.0, within(0.001));
    }

    @Test
    void shouldConvertToRequestedUnit() {
        AtomicLong uptimeMs = new AtomicLong(5000);
        TimeGauge gauge = TimeGauge.builder("uptime", uptimeMs,
                        TimeUnit.MILLISECONDS, AtomicLong::doubleValue)
                .register(registry);
        assertThat(gauge.value(TimeUnit.SECONDS)).isCloseTo(5.0, within(0.001));
        assertThat(gauge.value(TimeUnit.MILLISECONDS)).isCloseTo(5000.0, within(0.1));
        assertThat(gauge.value(TimeUnit.MINUTES)).isCloseTo(5.0 / 60, within(0.001));
    }

    @Test
    void shouldTrackChangingValues() {
        AtomicLong waitTimeMs = new AtomicLong(100);
        TimeGauge gauge = TimeGauge.builder("pool.wait", waitTimeMs,
                        TimeUnit.MILLISECONDS, AtomicLong::doubleValue)
                .register(registry);
        assertThat(gauge.value(TimeUnit.MILLISECONDS)).isCloseTo(100.0, within(0.1));
        waitTimeMs.set(500);
        assertThat(gauge.value(TimeUnit.MILLISECONDS)).isCloseTo(500.0, within(0.1));
    }

    @Test
    void shouldReturnNaN_WhenSourceIsGarbageCollected() {
        AtomicLong source = new AtomicLong(1000);
        TimeGauge gauge = TimeGauge.builder("gc.test", source,
                        TimeUnit.MILLISECONDS, AtomicLong::doubleValue)
                .register(registry);
        assertThat(gauge.value()).isCloseTo(1.0, within(0.001));
        source = null;
        System.gc();
        assertThat(gauge.value()).isNaN();
    }

    @Test
    void shouldReturnBaseTimeUnit() {
        AtomicLong source = new AtomicLong(100);
        TimeGauge gauge = TimeGauge.builder("unit.test", source,
                        TimeUnit.MILLISECONDS, AtomicLong::doubleValue)
                .register(registry);
        assertThat(gauge.baseTimeUnit()).isEqualTo(TimeUnit.SECONDS);
    }

    @Test
    void shouldDeduplicateByNameAndTags() {
        AtomicLong source = new AtomicLong(100);
        TimeGauge first = TimeGauge.builder("dedup", source,
                        TimeUnit.MILLISECONDS, AtomicLong::doubleValue)
                .tag("pool", "main").register(registry);
        TimeGauge second = TimeGauge.builder("dedup", source,
                        TimeUnit.MILLISECONDS, AtomicLong::doubleValue)
                .tag("pool", "main").register(registry);
        assertThat(first).isSameAs(second);
    }

    @Test
    void shouldSupportTagsAndDescription() {
        AtomicLong source = new AtomicLong(100);
        TimeGauge gauge = TimeGauge.builder("conn.wait", source,
                        TimeUnit.MILLISECONDS, AtomicLong::doubleValue)
                .tags("pool", "hikari", "db", "primary")
                .description("Connection pool wait time").register(registry);
        assertThat(gauge.getId().getName()).isEqualTo("conn.wait");
        assertThat(gauge.getId().getTag("pool")).isEqualTo("hikari");
        assertThat(gauge.getId().getTag("db")).isEqualTo("primary");
        assertThat(gauge.getId().getDescription()).isEqualTo("Connection pool wait time");
    }

    @Test
    void shouldProduceSingleValueMeasurement() {
        AtomicLong source = new AtomicLong(3000);
        TimeGauge gauge = TimeGauge.builder("measured", source,
                        TimeUnit.MILLISECONDS, AtomicLong::doubleValue)
                .register(registry);
        var measurements = gauge.measure();
        assertThat(measurements).hasSize(1);
        Measurement m = measurements.iterator().next();
        assertThat(m.getStatistic()).isEqualTo(Statistic.VALUE);
        assertThat(m.getValue()).isCloseTo(3.0, within(0.001));
    }

    @Test
    void shouldRegisterViaMoreConvenience() {
        AtomicLong source = new AtomicLong(2000);
        TimeGauge gauge = registry.more().timeGauge("direct.gauge",
                Tags.of("env", "test"), source,
                TimeUnit.MILLISECONDS, AtomicLong::doubleValue);
        assertThat(gauge.value(TimeUnit.SECONDS)).isCloseTo(2.0, within(0.001));
        assertThat(gauge.getId().getTag("env")).isEqualTo("test");
    }

    @Test
    void shouldExtendGaugeInterface() {
        AtomicLong source = new AtomicLong(1000);
        TimeGauge timeGauge = TimeGauge.builder("is.a.gauge", source,
                        TimeUnit.MILLISECONDS, AtomicLong::doubleValue)
                .register(registry);
        assertThat(timeGauge).isInstanceOf(Gauge.class);
        Gauge gauge = timeGauge;
        assertThat(gauge.value()).isCloseTo(1.0, within(0.001));
    }

}
```

#### File: `src/test/java/dev/linhvu/micrometer/cumulative/CumulativeFunctionCounterTest.java` [NEW]

```java
package dev.linhvu.micrometer.cumulative;

import dev.linhvu.micrometer.Meter;
import dev.linhvu.micrometer.Tags;
import org.junit.jupiter.api.Test;

import java.util.concurrent.atomic.AtomicLong;

import static org.assertj.core.api.Assertions.assertThat;

class CumulativeFunctionCounterTest {

    @Test
    void shouldReadCountFromSourceObject() {
        AtomicLong source = new AtomicLong(42);
        var counter = new CumulativeFunctionCounter<>(
                new Meter.Id("test", Tags.empty(), Meter.Type.COUNTER, null, null),
                source, AtomicLong::doubleValue);
        assertThat(counter.count()).isEqualTo(42.0);
        source.set(100);
        assertThat(counter.count()).isEqualTo(100.0);
    }

    @Test
    void shouldReturnLastValue_WhenWeakReferenceIsCleared() {
        AtomicLong source = new AtomicLong(55);
        var counter = new CumulativeFunctionCounter<>(
                new Meter.Id("weak.test", Tags.empty(), Meter.Type.COUNTER, null, null),
                source, AtomicLong::doubleValue);
        assertThat(counter.count()).isEqualTo(55.0);
        source = null;
        System.gc();
        assertThat(counter.count()).isEqualTo(55.0);
    }

    @Test
    void shouldReturnZero_WhenNeverReadAndWeakReferenceCleared() {
        AtomicLong source = new AtomicLong(10);
        var counter = new CumulativeFunctionCounter<>(
                new Meter.Id("never.read", Tags.empty(), Meter.Type.COUNTER, null, null),
                source, AtomicLong::doubleValue);
        source = null;
        System.gc();
        assertThat(counter.count()).isEqualTo(0.0);
    }

}
```

#### File: `src/test/java/dev/linhvu/micrometer/cumulative/CumulativeFunctionTimerTest.java` [NEW]

```java
package dev.linhvu.micrometer.cumulative;

import dev.linhvu.micrometer.Meter;
import dev.linhvu.micrometer.Tags;
import org.junit.jupiter.api.Test;

import java.util.concurrent.TimeUnit;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.within;

class CumulativeFunctionTimerTest {

    @Test
    void shouldReadCountAndTotalTimeFromSource() {
        var source = new TimerSource(10, 5000.0);
        var timer = createTimer(source, TimeUnit.MILLISECONDS, TimeUnit.SECONDS);
        assertThat(timer.count()).isEqualTo(10.0);
        assertThat(timer.totalTime(TimeUnit.SECONDS)).isCloseTo(5.0, within(0.001));
    }

    @Test
    void shouldConvertTotalTimeToRequestedUnit() {
        var source = new TimerSource(1, 3.0);
        var timer = createTimer(source, TimeUnit.SECONDS, TimeUnit.SECONDS);
        assertThat(timer.totalTime(TimeUnit.SECONDS)).isCloseTo(3.0, within(0.001));
        assertThat(timer.totalTime(TimeUnit.MILLISECONDS)).isCloseTo(3000.0, within(0.1));
    }

    @Test
    void shouldClampNegativeCountToZero() {
        var source = new TimerSource(-5, 100.0);
        var timer = createTimer(source, TimeUnit.MILLISECONDS, TimeUnit.SECONDS);
        assertThat(timer.count()).isEqualTo(0.0);
    }

    @Test
    void shouldClampNegativeTotalTimeToZero() {
        var source = new TimerSource(1, -100.0);
        var timer = createTimer(source, TimeUnit.MILLISECONDS, TimeUnit.SECONDS);
        assertThat(timer.totalTime(TimeUnit.SECONDS)).isEqualTo(0.0);
    }

    @Test
    void shouldReturnBaseTimeUnit() {
        var source = new TimerSource(1, 100.0);
        var timer = createTimer(source, TimeUnit.MILLISECONDS, TimeUnit.SECONDS);
        assertThat(timer.baseTimeUnit()).isEqualTo(TimeUnit.SECONDS);
    }

    @Test
    void shouldReturnLastValues_WhenWeakReferenceIsCleared() {
        var source = new TimerSource(10, 5000.0);
        var timer = createTimer(source, TimeUnit.MILLISECONDS, TimeUnit.SECONDS);
        assertThat(timer.count()).isEqualTo(10.0);
        assertThat(timer.totalTime(TimeUnit.SECONDS)).isCloseTo(5.0, within(0.001));
        source = null;
        System.gc();
        assertThat(timer.count()).isEqualTo(10.0);
        assertThat(timer.totalTime(TimeUnit.SECONDS)).isCloseTo(5.0, within(0.001));
    }

    private CumulativeFunctionTimer<TimerSource> createTimer(TimerSource source,
            TimeUnit sourceUnit, TimeUnit baseUnit) {
        return new CumulativeFunctionTimer<>(
                new Meter.Id("test.timer", Tags.empty(), Meter.Type.TIMER, null, null),
                source, TimerSource::count, TimerSource::totalTime, sourceUnit, baseUnit);
    }

    record TimerSource(long count, double totalTime) {
    }

}
```

#### File: `src/test/java/dev/linhvu/micrometer/internal/DefaultTimeGaugeTest.java` [NEW]

```java
package dev.linhvu.micrometer.internal;

import dev.linhvu.micrometer.Meter;
import dev.linhvu.micrometer.Tags;
import org.junit.jupiter.api.Test;

import java.util.concurrent.TimeUnit;
import java.util.concurrent.atomic.AtomicLong;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.within;

class DefaultTimeGaugeTest {

    @Test
    void shouldConvertFromSourceUnitToBaseUnit() {
        AtomicLong source = new AtomicLong(5000);
        var gauge = new DefaultTimeGauge<>(
                new Meter.Id("test", Tags.empty(), Meter.Type.GAUGE, null, null),
                source, TimeUnit.MILLISECONDS, AtomicLong::doubleValue, TimeUnit.SECONDS);
        assertThat(gauge.value()).isCloseTo(5.0, within(0.001));
    }

    @Test
    void shouldReturnBaseTimeUnit() {
        AtomicLong source = new AtomicLong(100);
        var gauge = new DefaultTimeGauge<>(
                new Meter.Id("test", Tags.empty(), Meter.Type.GAUGE, null, null),
                source, TimeUnit.MILLISECONDS, AtomicLong::doubleValue, TimeUnit.SECONDS);
        assertThat(gauge.baseTimeUnit()).isEqualTo(TimeUnit.SECONDS);
    }

    @Test
    void shouldReturnNaN_WhenSourceIsGarbageCollected() {
        AtomicLong source = new AtomicLong(100);
        var gauge = new DefaultTimeGauge<>(
                new Meter.Id("gc.test", Tags.empty(), Meter.Type.GAUGE, null, null),
                source, TimeUnit.MILLISECONDS, AtomicLong::doubleValue, TimeUnit.SECONDS);
        assertThat(gauge.value()).isNotNaN();
        source = null;
        System.gc();
        assertThat(gauge.value()).isNaN();
    }

    @Test
    void shouldReturnNaN_WhenFunctionThrows() {
        Object source = new Object();
        var gauge = new DefaultTimeGauge<>(
                new Meter.Id("error.test", Tags.empty(), Meter.Type.GAUGE, null, null),
                source, TimeUnit.MILLISECONDS,
                obj -> { throw new RuntimeException("boom"); }, TimeUnit.SECONDS);
        assertThat(gauge.value()).isNaN();
    }

}
```

#### File: `src/test/java/dev/linhvu/micrometer/noop/NoopFunctionCounterTest.java` [NEW]

```java
package dev.linhvu.micrometer.noop;

import dev.linhvu.micrometer.Meter;
import dev.linhvu.micrometer.Tags;
import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;

class NoopFunctionCounterTest {

    @Test
    void shouldAlwaysReturnZeroCount() {
        var counter = new NoopFunctionCounter(
                new Meter.Id("noop", Tags.empty(), Meter.Type.COUNTER, null, null));
        assertThat(counter.count()).isEqualTo(0.0);
    }

    @Test
    void shouldReturnEmptyMeasurements() {
        var counter = new NoopFunctionCounter(
                new Meter.Id("noop", Tags.empty(), Meter.Type.COUNTER, null, null));
        assertThat(counter.measure()).isEmpty();
    }

}
```

#### File: `src/test/java/dev/linhvu/micrometer/noop/NoopFunctionTimerTest.java` [NEW]

```java
package dev.linhvu.micrometer.noop;

import dev.linhvu.micrometer.Meter;
import dev.linhvu.micrometer.Tags;
import org.junit.jupiter.api.Test;

import java.util.concurrent.TimeUnit;

import static org.assertj.core.api.Assertions.assertThat;

class NoopFunctionTimerTest {

    @Test
    void shouldAlwaysReturnZero() {
        var timer = new NoopFunctionTimer(
                new Meter.Id("noop", Tags.empty(), Meter.Type.TIMER, null, null));
        assertThat(timer.count()).isEqualTo(0.0);
        assertThat(timer.totalTime(TimeUnit.SECONDS)).isEqualTo(0.0);
        assertThat(timer.mean(TimeUnit.SECONDS)).isEqualTo(0.0);
    }

    @Test
    void shouldReturnNanosecondsAsBaseUnit() {
        var timer = new NoopFunctionTimer(
                new Meter.Id("noop", Tags.empty(), Meter.Type.TIMER, null, null));
        assertThat(timer.baseTimeUnit()).isEqualTo(TimeUnit.NANOSECONDS);
    }

}
```

#### File: `src/test/java/dev/linhvu/micrometer/noop/NoopTimeGaugeTest.java` [NEW]

```java
package dev.linhvu.micrometer.noop;

import dev.linhvu.micrometer.Meter;
import dev.linhvu.micrometer.Tags;
import org.junit.jupiter.api.Test;

import java.util.concurrent.TimeUnit;

import static org.assertj.core.api.Assertions.assertThat;

class NoopTimeGaugeTest {

    @Test
    void shouldAlwaysReturnZero() {
        var gauge = new NoopTimeGauge(
                new Meter.Id("noop", Tags.empty(), Meter.Type.GAUGE, null, null));
        assertThat(gauge.value()).isEqualTo(0.0);
        assertThat(gauge.value(TimeUnit.SECONDS)).isEqualTo(0.0);
    }

    @Test
    void shouldReturnNanosecondsAsBaseUnit() {
        var gauge = new NoopTimeGauge(
                new Meter.Id("noop", Tags.empty(), Meter.Type.GAUGE, null, null));
        assertThat(gauge.baseTimeUnit()).isEqualTo(TimeUnit.NANOSECONDS);
    }

}
```

#### File: `src/test/java/dev/linhvu/micrometer/integration/FunctionMetersIntegrationTest.java` [NEW]

```java
package dev.linhvu.micrometer.integration;

import dev.linhvu.micrometer.Counter;
import dev.linhvu.micrometer.FunctionCounter;
import dev.linhvu.micrometer.FunctionTimer;
import dev.linhvu.micrometer.Gauge;
import dev.linhvu.micrometer.Meter;
import dev.linhvu.micrometer.Tags;
import dev.linhvu.micrometer.TimeGauge;
import dev.linhvu.micrometer.config.MeterFilter;
import dev.linhvu.micrometer.noop.NoopFunctionCounter;
import dev.linhvu.micrometer.noop.NoopFunctionTimer;
import dev.linhvu.micrometer.noop.NoopTimeGauge;
import dev.linhvu.micrometer.simple.SimpleMeterRegistry;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.util.concurrent.TimeUnit;
import java.util.concurrent.atomic.AtomicLong;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;
import static org.assertj.core.api.Assertions.within;

class FunctionMetersIntegrationTest {

    private SimpleMeterRegistry registry;

    @BeforeEach
    void setUp() {
        registry = new SimpleMeterRegistry();
    }

    @Test
    void shouldReturnNoopFunctionCounter_WhenFilterDenies() {
        registry.meterFilter(MeterFilter.deny(id -> id.getName().startsWith("denied")));
        AtomicLong source = new AtomicLong(100);
        FunctionCounter counter = FunctionCounter.builder("denied.counter", source, AtomicLong::doubleValue)
                .register(registry);
        assertThat(counter).isInstanceOf(NoopFunctionCounter.class);
        assertThat(counter.count()).isEqualTo(0.0);
    }

    @Test
    void shouldReturnNoopFunctionTimer_WhenFilterDenies() {
        registry.meterFilter(MeterFilter.deny(id -> id.getName().startsWith("denied")));
        FunctionTimer timer = FunctionTimer.builder("denied.timer",
                        new Object(), obj -> 0L, obj -> 0.0, TimeUnit.MILLISECONDS)
                .register(registry);
        assertThat(timer).isInstanceOf(NoopFunctionTimer.class);
    }

    @Test
    void shouldReturnNoopTimeGauge_WhenFilterDenies() {
        registry.meterFilter(MeterFilter.deny(id -> id.getName().startsWith("denied")));
        AtomicLong source = new AtomicLong(100);
        TimeGauge gauge = TimeGauge.builder("denied.gauge", source,
                        TimeUnit.MILLISECONDS, AtomicLong::doubleValue)
                .register(registry);
        assertThat(gauge).isInstanceOf(NoopTimeGauge.class);
    }

    @Test
    void shouldApplyCommonTagsToFunctionMeters() {
        registry.meterFilter(MeterFilter.commonTags(Tags.of("env", "prod")));
        AtomicLong source = new AtomicLong(42);
        FunctionCounter counter = FunctionCounter.builder("tagged.counter", source, AtomicLong::doubleValue)
                .register(registry);
        assertThat(counter.getId().getTag("env")).isEqualTo("prod");
    }

    @Test
    void shouldReturnNoopMeters_WhenRegistryIsClosed() {
        registry.close();
        AtomicLong source = new AtomicLong(100);
        FunctionCounter counter = FunctionCounter.builder("closed.counter", source, AtomicLong::doubleValue)
                .register(registry);
        assertThat(counter).isInstanceOf(NoopFunctionCounter.class);
        FunctionTimer timer = FunctionTimer.builder("closed.timer", source,
                        AtomicLong::longValue, AtomicLong::doubleValue, TimeUnit.MILLISECONDS)
                .register(registry);
        assertThat(timer).isInstanceOf(NoopFunctionTimer.class);
        TimeGauge gauge = TimeGauge.builder("closed.gauge", source,
                        TimeUnit.MILLISECONDS, AtomicLong::doubleValue)
                .register(registry);
        assertThat(gauge).isInstanceOf(NoopTimeGauge.class);
    }

    @Test
    void shouldThrow_WhenRegisteringFunctionCounterWithSameNameAsCounter() {
        registry.counter("conflict", "type", "regular");
        AtomicLong source = new AtomicLong(1);
        assertThatThrownBy(() ->
                FunctionCounter.builder("conflict", source, AtomicLong::doubleValue)
                        .tag("type", "regular").register(registry))
                .isInstanceOf(IllegalArgumentException.class)
                .hasMessageContaining("already a meter registered");
    }

    @Test
    void shouldDispatchFunctionCounterViaVisitor() {
        AtomicLong source = new AtomicLong(5);
        FunctionCounter counter = FunctionCounter.builder("visitor.fc", source, AtomicLong::doubleValue)
                .register(registry);
        String result = counter.match(
                c -> "counter", g -> "gauge", t -> "timer", ds -> "summary",
                ltt -> "longTaskTimer", fc -> "functionCounter:" + fc.count(),
                ft -> "functionTimer", m -> "other");
        assertThat(result).isEqualTo("functionCounter:5.0");
    }

    @Test
    void shouldDispatchFunctionTimerViaVisitor() {
        var source = new Object() {
            long count() { return 3; }
            double totalMs() { return 900.0; }
        };
        FunctionTimer timer = FunctionTimer.builder("visitor.ft", source,
                        obj -> obj.count(), obj -> obj.totalMs(), TimeUnit.MILLISECONDS)
                .register(registry);
        String result = timer.match(
                c -> "counter", g -> "gauge", t -> "timer", ds -> "summary",
                ltt -> "longTaskTimer", fc -> "functionCounter",
                ft -> "functionTimer:" + ft.count(), m -> "other");
        assertThat(result).isEqualTo("functionTimer:3.0");
    }

    @Test
    void shouldDispatchTimeGaugeViaGaugeVisitor() {
        AtomicLong source = new AtomicLong(1000);
        TimeGauge gauge = TimeGauge.builder("visitor.tg", source,
                        TimeUnit.MILLISECONDS, AtomicLong::doubleValue)
                .register(registry);
        String result = gauge.match(
                c -> "counter", g -> "gauge:" + g.value(), t -> "timer", ds -> "summary",
                ltt -> "longTaskTimer", fc -> "functionCounter",
                ft -> "functionTimer", m -> "other");
        assertThat(result).startsWith("gauge:");
    }

    @Test
    void shouldIncludeFunctionMetersInGetMeters() {
        AtomicLong source = new AtomicLong(1);
        FunctionCounter.builder("fn.counter", source, AtomicLong::doubleValue).register(registry);
        FunctionTimer.builder("fn.timer", source, AtomicLong::longValue,
                AtomicLong::doubleValue, TimeUnit.MILLISECONDS).register(registry);
        TimeGauge.builder("fn.gauge", source, TimeUnit.MILLISECONDS,
                AtomicLong::doubleValue).register(registry);
        assertThat(registry.getMeters()).hasSize(3);
        assertThat(registry.getMeters()).anyMatch(m -> m instanceof FunctionCounter);
        assertThat(registry.getMeters()).anyMatch(m -> m instanceof FunctionTimer);
        assertThat(registry.getMeters()).anyMatch(m -> m instanceof TimeGauge);
    }

    @Test
    void shouldWrapExternalMetricsSource() {
        var connectionPool = new SimulatedConnectionPool();
        FunctionCounter.builder("pool.connections.created", connectionPool,
                        SimulatedConnectionPool::totalCreated)
                .tag("pool", "primary").register(registry);
        FunctionTimer.builder("pool.acquire", connectionPool,
                        SimulatedConnectionPool::acquireCount,
                        SimulatedConnectionPool::totalAcquireTimeMs, TimeUnit.MILLISECONDS)
                .tag("pool", "primary").register(registry);
        TimeGauge.builder("pool.idle.time", connectionPool,
                        TimeUnit.MILLISECONDS, SimulatedConnectionPool::avgIdleTimeMs)
                .tag("pool", "primary").register(registry);
        connectionPool.simulateActivity(50, 2500.0, 150.0);
        Meter fnCounter = registry.getMeters().stream()
                .filter(m -> m.getId().getName().equals("pool.connections.created"))
                .findFirst().orElseThrow();
        assertThat(((FunctionCounter) fnCounter).count()).isEqualTo(50.0);
        Meter fnTimer = registry.getMeters().stream()
                .filter(m -> m.getId().getName().equals("pool.acquire"))
                .findFirst().orElseThrow();
        assertThat(((FunctionTimer) fnTimer).count()).isEqualTo(50.0);
        assertThat(((FunctionTimer) fnTimer).totalTime(TimeUnit.MILLISECONDS))
                .isCloseTo(2500.0, within(0.1));
        assertThat(((FunctionTimer) fnTimer).mean(TimeUnit.MILLISECONDS))
                .isCloseTo(50.0, within(0.1));
        Meter tGauge = registry.getMeters().stream()
                .filter(m -> m.getId().getName().equals("pool.idle.time"))
                .findFirst().orElseThrow();
        assertThat(((TimeGauge) tGauge).value(TimeUnit.MILLISECONDS))
                .isCloseTo(150.0, within(0.1));
    }

    static class SimulatedConnectionPool {
        private double totalCreated = 0;
        private long acquireCount = 0;
        private double totalAcquireTimeMs = 0;
        private double avgIdleTimeMs = 0;

        void simulateActivity(long connections, double acquireMs, double idleMs) {
            this.totalCreated = connections;
            this.acquireCount = connections;
            this.totalAcquireTimeMs = acquireMs;
            this.avgIdleTimeMs = idleMs;
        }

        double totalCreated() { return totalCreated; }
        long acquireCount() { return acquireCount; }
        double totalAcquireTimeMs() { return totalAcquireTimeMs; }
        double avgIdleTimeMs() { return avgIdleTimeMs; }
    }

}
```

---

## Summary

| Concept | What It Means |
|---------|---------------|
| **FunctionCounter** | A counter that derives its value from an external object via a `ToDoubleFunction` — wraps existing counters from other libraries |
| **FunctionTimer** | A timer that derives count and total time from an external object — wraps existing timers from other libraries |
| **TimeGauge** | A Gauge with time-unit awareness — automatically converts between the source unit and the registry's base unit |
| **`More` inner class** | A namespace on `MeterRegistry` for less common meter types, keeping the main API surface clean |
| **WeakReference with last-value** | Function counters/timers return the last known value when GC'd (unlike gauges which return NaN) because cumulative values remain meaningful |
| **TimeUtils** | Double-precision time conversion utility that avoids `TimeUnit.convert(long)` truncation |

**Next: Chapter 12 — LongTaskTimer** — Tracks the duration of long-running tasks that are still in progress, reporting active task count and cumulative duration. Unlike regular Timer which records completed events, LongTaskTimer answers "how many tasks are running right now, and how long have they been running?"
