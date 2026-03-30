# Chapter 12: Function-Tracking Meters — Wrapping External Statistics

> **Feature 12** · Tier 3 · Depends on: Features 1–3 (Counter, Gauge, Timer)

After this chapter you will have three new meter types that wrap **externally-managed
statistics** — `FunctionCounter`, `FunctionTimer`, and `TimeGauge`. These are the meters
you reach for when the source of truth for a measurement lives in *another library's API*
(cache stats, connection pool metrics, JMX beans) and you can't call `increment()` or
`record()` yourself.

---

## 1. The API Contract

Before writing any implementation, let's define exactly what a client sees.

### What the client imports

```java
import simple.micrometer.api.FunctionCounter;
import simple.micrometer.api.FunctionTimer;
import simple.micrometer.api.TimeGauge;
import simple.micrometer.api.MeterRegistry;
import simple.micrometer.internal.SimpleMeterRegistry;
```

### What the client writes

```java
MeterRegistry registry = new SimpleMeterRegistry();

// 1. FunctionCounter — wrap an external monotonic count
Cache cache = createCache();
FunctionCounter hitCounter = FunctionCounter
    .builder("cache.hits", cache, Cache::getHitCount)
    .tag("cache", "users")
    .register(registry);
double hits = hitCounter.count(); // delegates to cache.getHitCount()

// 2. FunctionTimer — wrap external count + total time
FunctionTimer getTimer = FunctionTimer
    .builder("cache.gets", cache,
        Cache::getRequestCount,        // count function (ToLongFunction)
        Cache::getTotalLoadTime,       // total time function (ToDoubleFunction)
        TimeUnit.NANOSECONDS)          // unit of the total time function
    .register(registry);
double count = getTimer.count();
double total = getTimer.totalTime(TimeUnit.MILLISECONDS);
double avg   = getTimer.mean(TimeUnit.MILLISECONDS);

// 3. TimeGauge — observe a time value with automatic unit conversion
RuntimeMXBean runtime = ManagementFactory.getRuntimeMXBean();
TimeGauge uptime = TimeGauge
    .builder("process.uptime", runtime,
        TimeUnit.MILLISECONDS,
        RuntimeMXBean::getUptime)
    .register(registry);
double uptimeSeconds = uptime.value(TimeUnit.SECONDS);
```

### Behavioral contract

| Promise | Detail |
|---------|--------|
| Pull-based | The framework evaluates the provided function when it needs a reading — no push API |
| WeakReference | The observed object is held via `WeakReference`; if GC'd, the meter freezes at its last value |
| Deduplication | Same name + tags returns the same meter instance |
| Filter-aware | Denied meters return noop implementations (zero values, no side effects) |
| Type-safe generics | `Builder<T>` ensures the function matches the observed object's type at compile time |
| Time unit conversion | `FunctionTimer` and `TimeGauge` automatically convert between time units |
| IS-A relationships | `FunctionCounter` IS-A `Meter`, `FunctionTimer` IS-A `Meter`, `TimeGauge` IS-A `Gauge` IS-A `Meter` |
| Measurements | `FunctionCounter` → 1 (COUNT), `FunctionTimer` → 2 (COUNT + TOTAL_TIME), `TimeGauge` → 1 (VALUE) |

### Push vs Pull: When to use which

```
Push-based (regular meters)          Pull-based (function-tracking meters)
──────────────────────────           ────────────────────────────────────
You own the data source              Another library owns the data
counter.increment()                  FunctionCounter wraps cache.getHitCount()
timer.record(duration)               FunctionTimer wraps cache.getRequestCount() + getTotalTime()
Gauge.builder(obj, fn)               TimeGauge.builder(obj, MILLIS, fn)
Write API (increment/record)         Read-only API (count/totalTime/value)
Data stored in meter internals       Data stored in external object
```

---

## 2. Client Usage & Tests

These tests define the API's behavior — they were written **before** any implementation.

### FunctionCounter: Track an external count

```java
@Test
void shouldTrackExternalCount() {
    var cache = new FakeCache();
    FunctionCounter hitCounter = FunctionCounter
            .builder("cache.hits", cache, FakeCache::getHitCount)
            .register(registry);

    assertThat(hitCounter.count()).isEqualTo(0);

    cache.simulateHits(7);
    assertThat(hitCounter.count()).isEqualTo(7);

    cache.simulateHits(3);
    assertThat(hitCounter.count()).isEqualTo(10);
}
```

### FunctionTimer: Track external count + total time

```java
@Test
void shouldTrackExternalCountAndTotalTime() {
    var cache = new FakeCache();
    FunctionTimer timer = FunctionTimer
            .builder("cache.gets", cache,
                    FakeCache::getRequestCount,
                    FakeCache::getTotalLoadTimeNanos,
                    TimeUnit.NANOSECONDS)
            .register(registry);

    assertThat(timer.count()).isEqualTo(0);
    assertThat(timer.totalTime(TimeUnit.NANOSECONDS)).isEqualTo(0);

    cache.simulateRequests(5, 100_000_000L); // 5 requests, 100ms total
    assertThat(timer.count()).isEqualTo(5);
    assertThat(timer.totalTime(TimeUnit.MILLISECONDS))
            .isCloseTo(100.0, within(0.1));
}
```

### FunctionTimer: Mean computation and zero-division guard

```java
@Test
void shouldComputeMeanTime() {
    var cache = new FakeCache();
    cache.simulateRequests(10, 500_000_000L); // 10 requests, 500ms total

    FunctionTimer timer = FunctionTimer
            .builder("cache.gets", cache,
                    FakeCache::getRequestCount,
                    FakeCache::getTotalLoadTimeNanos,
                    TimeUnit.NANOSECONDS)
            .register(registry);

    assertThat(timer.mean(TimeUnit.MILLISECONDS))
            .isCloseTo(50.0, within(0.1)); // 500ms / 10 = 50ms avg
}

@Test
void shouldReturnZeroMeanWhenCountIsZero() {
    var cache = new FakeCache();
    FunctionTimer timer = FunctionTimer
            .builder("cache.gets", cache,
                    FakeCache::getRequestCount,
                    FakeCache::getTotalLoadTimeNanos,
                    TimeUnit.NANOSECONDS)
            .register(registry);

    assertThat(timer.mean(TimeUnit.MILLISECONDS)).isEqualTo(0);
}
```

### TimeGauge: Time-valued gauge with unit conversion

```java
@Test
void shouldTrackTimeValue() {
    var runtime = new FakeRuntime();
    runtime.setUptime(5000); // 5000 ms

    TimeGauge uptime = TimeGauge
            .builder("process.uptime", runtime,
                    TimeUnit.MILLISECONDS,
                    FakeRuntime::getUptime)
            .register(registry);

    assertThat(uptime.value(TimeUnit.SECONDS))
            .isCloseTo(5.0, within(0.001));
    assertThat(uptime.value(TimeUnit.MILLISECONDS))
            .isCloseTo(5000.0, within(0.1));
}

@Test
void shouldBeAGauge() {
    var runtime = new FakeRuntime();
    runtime.setUptime(1000);

    TimeGauge uptime = TimeGauge
            .builder("process.uptime", runtime,
                    TimeUnit.MILLISECONDS,
                    FakeRuntime::getUptime)
            .register(registry);

    // TimeGauge IS-A Gauge — value() without TimeUnit returns
    // the raw value in the source function's unit (milliseconds)
    assertThat(uptime.value()).isCloseTo(1000.0, within(0.1));
}
```

### MeterFilter: Noop behavior when denied

```java
@Test
void shouldReturnNoopFunctionCounterWhenDenied() {
    registry.config().meterFilter(MeterFilter.deny(id ->
            id.getName().startsWith("cache")));

    var cache = new FakeCache();
    cache.simulateHits(100);

    FunctionCounter counter = FunctionCounter
            .builder("cache.hits", cache, FakeCache::getHitCount)
            .register(registry);

    // Noop: returns 0 regardless of external state
    assertThat(counter.count()).isEqualTo(0);
    assertThat(registry.getMeters()).isEmpty();
}
```

---

## 3. Implementing the Call Chain

### Call chain diagram

```
FunctionCounter.builder("cache.hits", cache, Cache::getHitCount).register(registry)
  │
  ├─► [API Layer] FunctionCounter.Builder<T>
  │     Stores name, obj, function, tags. Terminal op: register(registry)
  │
  ├─► [API Layer] MeterRegistry.More.counter(Meter.Id, obj, f)
  │     Calls registerMeterIfNecessary(FunctionCounter.class, id, factory, noopFactory)
  │
  ├─► [Registration Layer] MeterRegistry.registerMeterIfNecessary()
  │     1. Apply map/accept filter chain (reused from Feature 5)
  │     2. Double-checked locking for deduplication (reused from Feature 1)
  │     3. If denied → NoopFunctionCounter
  │     4. If accepted → newFunctionCounter(id, obj, f)
  │
  └─► [Implementation Layer] SimpleMeterRegistry.newFunctionCounter()
        Creates CumulativeFunctionCounter backed by WeakReference + volatile cache
```

### 3a. API Layer — FunctionCounter interface

The interface defines the contract and houses the Builder:

```java
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
        // ... tag(), description(), baseUnit() methods ...

        public FunctionCounter register(MeterRegistry registry) {
            return registry.more().counter(
                    new Meter.Id(name, tags, baseUnit, description, Type.COUNTER), obj, f);
        }
    }
}
```

**Key design**: The Builder holds three final fields — `name`, `obj`, and `f`. The generic
`<T>` flows from `builder()` through the builder to `register()`, ensuring type safety:
`FunctionCounter.builder("hits", cache, Cache::getHitCount)` — the compiler verifies
that `getHitCount` returns a number from a `Cache`.

### 3b. API Layer — FunctionTimer interface

FunctionTimer wraps **two** functions — count and total time:

```java
public interface FunctionTimer extends Meter {

    static <T> Builder<T> builder(String name, T obj, ToLongFunction<T> countFunction,
                                  ToDoubleFunction<T> totalTimeFunction,
                                  TimeUnit totalTimeFunctionUnit) { ... }

    double count();
    double totalTime(TimeUnit unit);
    TimeUnit baseTimeUnit();

    default double mean(TimeUnit unit) {
        double count = count();
        return count == 0 ? 0 : totalTime(unit) / count;
    }

    @Override
    default Iterable<Measurement> measure() {
        return Arrays.asList(
                new Measurement(this::count, Statistic.COUNT),
                new Measurement(() -> totalTime(baseTimeUnit()), Statistic.TOTAL_TIME));
    }
}
```

**Key design**: The count function is `ToLongFunction<T>` (returns `long`) while the
total time function is `ToDoubleFunction<T>` (returns `double`). This matches real-world
APIs where counts are integral but time accumulations may have fractional parts. The
`mean()` default method guards against division by zero.

### 3c. API Layer — TimeGauge interface

TimeGauge extends Gauge, adding time-unit awareness:

```java
public interface TimeGauge extends Gauge {

    static <T> Builder<T> builder(String name, T obj, TimeUnit fUnits,
                                  ToDoubleFunction<T> f) { ... }

    TimeUnit baseTimeUnit();

    default double value(TimeUnit unit) {
        return convertTime(value(), baseTimeUnit(), unit);
    }

    static double convertTime(double value, TimeUnit from, TimeUnit to) {
        if (from == to) return value;
        return value * ((double) from.toNanos(1) / to.toNanos(1));
    }
}
```

**Key design**: `TimeGauge` extends `Gauge` (not `Meter` directly). This means a
`TimeGauge` IS-A `Gauge` — it inherits `value()` (no-arg, returns raw value) and adds
`value(TimeUnit)` (with conversion). The `convertTime()` static utility does
double-precision time conversion, because Java's `TimeUnit.convert()` only works with
`long` values.

### 3d. Implementation Layer — CumulativeFunctionCounter

```java
public class CumulativeFunctionCounter<T> implements FunctionCounter {

    private final Meter.Id id;
    private final WeakReference<T> ref;
    private final ToDoubleFunction<T> f;
    private volatile double last;

    @Override
    public double count() {
        T obj = ref.get();
        return obj != null ? (last = f.applyAsDouble(obj)) : last;
    }
}
```

**The WeakReference + volatile cache pattern** is the heart of function-tracking meters:

1. **`WeakReference<T> ref`**: Holds the observed object without preventing GC
2. **`volatile double last`**: Caches the most recent reading
3. **`count()` logic**: If the object is alive → evaluate function, cache result, return it.
   If the object is GC'd → return the cached value (frozen at last known state)

This is only 3 fields and 1 method. The simplicity is intentional — function-tracking
meters are thin wrappers, not data structures.

### 3e. Implementation Layer — CumulativeFunctionTimer

```java
public class CumulativeFunctionTimer<T> implements FunctionTimer {

    private final WeakReference<T> ref;
    private final ToLongFunction<T> countFunction;
    private final ToDoubleFunction<T> totalTimeFunction;
    private final TimeUnit totalTimeFunctionUnit;
    private final TimeUnit baseTimeUnit;
    private volatile long lastCount;
    private volatile double lastTime;

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
                    TimeGauge.convertTime(totalTimeFunction.applyAsDouble(obj),
                            totalTimeFunctionUnit, baseTimeUnit),
                    0);
        }
        return TimeGauge.convertTime(lastTime, baseTimeUnit, unit);
    }
}
```

**Two-stage time conversion in `totalTime()`**:
1. Convert raw value from `totalTimeFunctionUnit` → `baseTimeUnit` (for consistent caching)
2. Convert cached value from `baseTimeUnit` → caller's requested `unit`

The `Math.max(..., 0)` guard prevents negative values from malformed external sources.

### 3f. Implementation Layer — DefaultTimeGauge

```java
public class DefaultTimeGauge<T> implements TimeGauge {

    private final WeakReference<T> ref;
    private final ToDoubleFunction<T> f;
    private final TimeUnit fUnits;

    @Override
    public double value() {
        T obj = ref.get();
        if (obj != null) {
            try { return f.applyAsDouble(obj); }
            catch (Exception ex) { return Double.NaN; }
        }
        return Double.NaN;
    }

    @Override
    public TimeUnit baseTimeUnit() { return fUnits; }
}
```

Structurally identical to `DefaultGauge` from Feature 2, but implements `TimeGauge`
instead of `Gauge`. The `baseTimeUnit()` returns the source function's unit; the inherited
`value(TimeUnit)` default method handles conversion.

### 3g. Registration Layer — Extending MeterRegistry.More

We extended the existing `More` inner class with three new methods:

```java
// Added to MeterRegistry.More:
<T> FunctionCounter counter(Meter.Id id, T obj, ToDoubleFunction<T> countFunction) {
    return registerMeterIfNecessary(FunctionCounter.class, id,
            i -> newFunctionCounter(i, obj, countFunction), NoopFunctionCounter::new);
}

<T> FunctionTimer timer(Meter.Id id, T obj, ToLongFunction<T> countFunction,
                        ToDoubleFunction<T> totalTimeFunction, TimeUnit totalTimeFunctionUnit) {
    return registerMeterIfNecessary(FunctionTimer.class, id,
            i -> newFunctionTimer(i, obj, countFunction, totalTimeFunction, totalTimeFunctionUnit),
            NoopFunctionTimer::new);
}

<T> TimeGauge timeGauge(Meter.Id id, T obj, TimeUnit fUnits, ToDoubleFunction<T> f) {
    return registerMeterIfNecessary(TimeGauge.class, id,
            i -> newTimeGauge(i, obj, fUnits, f), NoopTimeGauge::new);
}
```

These reuse the same `registerMeterIfNecessary()` infrastructure from Feature 1 — filter
chain, double-checked locking, deduplication — all for free.

### 3h. Factory Layer — SimpleMeterRegistry additions

```java
// Added to SimpleMeterRegistry:
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
            totalTimeFunctionUnit, TimeUnit.NANOSECONDS);
}

@Override
protected <T> TimeGauge newTimeGauge(Meter.Id id, T obj,
        TimeUnit fUnits, ToDoubleFunction<T> f) {
    return new DefaultTimeGauge<>(id, obj, fUnits, f);
}
```

Note: `SimpleMeterRegistry` always uses `TimeUnit.NANOSECONDS` as the base time unit
for `CumulativeFunctionTimer`. This is consistent with how `CumulativeTimer` stores
all values in nanoseconds internally.

---

## 4. Try It Yourself

**Challenge 1**: Write a `FunctionCounter` that wraps a `ConcurrentHashMap` and tracks
the number of entries. Hint: `FunctionCounter.builder("map.size", map, m -> m.size())`.
Wait — is `size()` monotonically increasing? Think about what happens when entries are
removed. Is `FunctionCounter` the right choice, or should you use a `Gauge`?

**Challenge 2**: Write a `FunctionTimer` that wraps a `ThreadPoolExecutor`. The count is
`getCompletedTaskCount()` and the total time is... well, `ThreadPoolExecutor` doesn't
expose total time. How would you handle this? Consider wrapping the executor with a
decorator that accumulates task durations.

**Challenge 3**: Modify `CumulativeFunctionCounter` to support `strongReference` mode
(like `Gauge.Builder` does). When would a user want this?

---

## 5. Why This Works

**Pull vs push is about ownership**: When your code *owns* the data (counting requests,
timing operations), use push-based meters. When *another library* owns the data (cache
stats, pool sizes, JMX beans), use pull-based meters. This isn't just API convenience —
it's about who is responsible for updating the metric.

**WeakReference prevents memory leaks in long-lived registries**: Consider a connection
pool that is closed and garbage collected. If the `FunctionCounter` held a strong reference,
the pool object would stay in memory forever. With `WeakReference`, the meter gracefully
degrades to returning the last known value.

**TimeGauge extends Gauge, not Meter**: This isn't just IS-A semantics — it means anywhere
Micrometer expects a `Gauge` (exports, search queries, filters), a `TimeGauge` just works.
The time unit awareness is layered on top without breaking the existing type hierarchy.

**`More` as progressive disclosure**: The main `MeterRegistry` API surface has convenience
methods for the 80% case (Counter, Timer, Gauge, Summary). Function-tracking meters live
behind `registry.more()` to keep the common API clean while making advanced meters
discoverable for users who need them.

---

## 6. What We Enhanced

| Previous Feature | What Changed | Why |
|-----------------|-------------|-----|
| Feature 1 (MeterRegistry) | Added 3 abstract factory methods: `newFunctionCounter`, `newFunctionTimer`, `newTimeGauge` | Each function-tracking meter needs a registry-specific implementation |
| Feature 1 (MeterRegistry.More) | Added 3 registration methods: `counter()`, `timer()`, `timeGauge()` | Function-tracking meters register through `More`, not top-level convenience methods |
| Feature 1 (SimpleMeterRegistry) | Implemented 3 factory methods | Wires factories to cumulative/default implementations |
| Feature 7 (CompositeMeterRegistry) | Implemented 3 factory methods | Uses cumulative implementations directly (pull-based meters don't need composite fan-out) |

---

## 7. Connection to Real Framework

| Simplified Class | Real Framework Class | Key Simplification |
|-----------------|---------------------|-------------------|
| `FunctionCounter` | `io.micrometer.core.instrument.FunctionCounter` | No `@Nullable` annotations |
| `FunctionTimer` | `io.micrometer.core.instrument.FunctionTimer` | Identical structure |
| `TimeGauge` | `io.micrometer.core.instrument.TimeGauge` | No `strongReference` support, no `@Incubating` supplier overload |
| `CumulativeFunctionCounter` | `io.micrometer.core.instrument.cumulative.CumulativeFunctionCounter` | Doesn't extend `AbstractMeter` (we have no abstract base) |
| `CumulativeFunctionTimer` | `io.micrometer.core.instrument.cumulative.CumulativeFunctionTimer` | Uses `TimeGauge.convertTime()` instead of `TimeUtils.convert()` |
| `DefaultTimeGauge` | Reuses `DefaultGauge` in real Micrometer | Separate class for clarity |
| `NoopFunctionCounter` | `io.micrometer.core.instrument.noop.NoopFunctionCounter` | Identical structure |
| `NoopFunctionTimer` | `io.micrometer.core.instrument.noop.NoopFunctionTimer` | Identical structure |
| `NoopTimeGauge` | `io.micrometer.core.instrument.noop.NoopTimeGauge` | Identical structure |

---

## 8. Complete Code

### `[NEW]` `src/main/java/simple/micrometer/api/FunctionCounter.java`

```java
package simple.micrometer.api;

import java.util.Collections;
import java.util.function.ToDoubleFunction;

/**
 * A counter that tracks a monotonically increasing function — used when the
 * source of truth for a count lives in another library's API and you can't
 * call {@code increment()} yourself.
 *
 * <p>Example: wrapping a cache's hit count that is maintained by the cache library:
 * <pre>{@code
 * Cache cache = createCache();
 * FunctionCounter hitCounter = FunctionCounter
 *     .builder("cache.hits", cache, Cache::getHitCount)
 *     .register(registry);
 *
 * // Later, just read:
 * double hits = hitCounter.count(); // delegates to cache.getHitCount()
 * }</pre>
 *
 * <p>Unlike {@link Counter} (push-based: you call {@code increment()}), a
 * FunctionCounter is pull-based: the framework calls the provided function
 * when it needs a reading.
 *
 * <p>The observed object is held via a {@link java.lang.ref.WeakReference}.
 * If nothing else holds a strong reference to it, it can be garbage collected,
 * and the counter freezes at its last known value.
 *
 * <p>Simplified from {@code io.micrometer.core.instrument.FunctionCounter}.
 */
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

        private Builder(String name, T obj, ToDoubleFunction<T> f) {
            this.name = name;
            this.obj = obj;
            this.f = f;
        }

        public Builder<T> tags(String... tags) {
            return tags(Tags.of(tags));
        }

        public Builder<T> tags(Iterable<Tag> tags) {
            this.tags = this.tags.and(tags);
            return this;
        }

        public Builder<T> tag(String key, String value) {
            this.tags = tags.and(key, value);
            return this;
        }

        public Builder<T> description(String description) {
            this.description = description;
            return this;
        }

        public Builder<T> baseUnit(String unit) {
            this.baseUnit = unit;
            return this;
        }

        public FunctionCounter register(MeterRegistry registry) {
            return registry.more().counter(
                    new Meter.Id(name, tags, baseUnit, description, Type.COUNTER), obj, f);
        }
    }
}
```

### `[NEW]` `src/main/java/simple/micrometer/api/FunctionTimer.java`

```java
package simple.micrometer.api;

import java.util.Arrays;
import java.util.concurrent.TimeUnit;
import java.util.function.ToDoubleFunction;
import java.util.function.ToLongFunction;

/**
 * A timer that tracks two monotonically increasing functions: one for the count
 * of events and one for the total time spent — used when timing data is
 * maintained externally and you can't call {@code record()} yourself.
 *
 * <p>Example: wrapping a cache's request count and total load time:
 * <pre>{@code
 * Cache cache = createCache();
 * FunctionTimer functionTimer = FunctionTimer
 *     .builder("cache.gets", cache,
 *         Cache::getRequestCount,
 *         Cache::getTotalLoadTime,
 *         TimeUnit.NANOSECONDS)
 *     .register(registry);
 *
 * // Later, just read:
 * double count = functionTimer.count();
 * double total = functionTimer.totalTime(TimeUnit.MILLISECONDS);
 * double avg   = functionTimer.mean(TimeUnit.MILLISECONDS);
 * }</pre>
 *
 * <p>Unlike {@link Timer} (push-based: you call {@code record()}), a
 * FunctionTimer is pull-based: the framework calls the provided functions
 * when it needs readings.
 *
 * <p>Simplified from {@code io.micrometer.core.instrument.FunctionTimer}.
 */
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

        public Builder<T> tags(String... tags) {
            return tags(Tags.of(tags));
        }

        public Builder<T> tags(Iterable<Tag> tags) {
            this.tags = this.tags.and(tags);
            return this;
        }

        public Builder<T> tag(String key, String value) {
            this.tags = tags.and(key, value);
            return this;
        }

        public Builder<T> description(String description) {
            this.description = description;
            return this;
        }

        public FunctionTimer register(MeterRegistry registry) {
            return registry.more().timer(
                    new Meter.Id(name, tags, null, description, Type.TIMER),
                    obj, countFunction, totalTimeFunction, totalTimeFunctionUnit);
        }
    }
}
```

### `[NEW]` `src/main/java/simple/micrometer/api/TimeGauge.java`

```java
package simple.micrometer.api;

import java.util.concurrent.TimeUnit;
import java.util.function.ToDoubleFunction;

/**
 * A specialized gauge that tracks a time-valued measurement, with automatic
 * unit conversion — used when you need to observe an externally-maintained
 * time value (uptime, elapsed time, TTL) and have the registry normalize it
 * to its preferred base time unit.
 *
 * <p>Example: wrapping the JVM's uptime from RuntimeMXBean:
 * <pre>{@code
 * RuntimeMXBean runtime = ManagementFactory.getRuntimeMXBean();
 * TimeGauge uptime = TimeGauge
 *     .builder("process.uptime", runtime,
 *         TimeUnit.MILLISECONDS,
 *         RuntimeMXBean::getUptime)
 *     .register(registry);
 *
 * // Read in any unit — automatic conversion:
 * double uptimeSeconds = uptime.value(TimeUnit.SECONDS);
 * }</pre>
 *
 * <p>TimeGauge extends {@link Gauge}, so anywhere a Gauge is expected,
 * a TimeGauge fits. The additional {@link #value(TimeUnit)} method provides
 * time-unit-aware access.
 *
 * <p>Simplified from {@code io.micrometer.core.instrument.TimeGauge}.
 */
public interface TimeGauge extends Gauge {

    static <T> Builder<T> builder(String name, T obj, TimeUnit fUnits, ToDoubleFunction<T> f) {
        return new Builder<>(name, obj, fUnits, f);
    }

    TimeUnit baseTimeUnit();

    default double value(TimeUnit unit) {
        return convertTime(value(), baseTimeUnit(), unit);
    }

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

        public Builder<T> tags(String... tags) {
            return tags(Tags.of(tags));
        }

        public Builder<T> tags(Iterable<Tag> tags) {
            this.tags = this.tags.and(tags);
            return this;
        }

        public Builder<T> tag(String key, String value) {
            this.tags = tags.and(key, value);
            return this;
        }

        public Builder<T> description(String description) {
            this.description = description;
            return this;
        }

        public TimeGauge register(MeterRegistry registry) {
            return registry.more().timeGauge(
                    new Meter.Id(name, tags, null, description, Meter.Type.GAUGE),
                    obj, fUnits, f);
        }
    }

    static double convertTime(double value, TimeUnit from, TimeUnit to) {
        if (from == to) return value;
        return value * ((double) from.toNanos(1) / to.toNanos(1));
    }
}
```

### `[NEW]` `src/main/java/simple/micrometer/internal/CumulativeFunctionCounter.java`

```java
package simple.micrometer.internal;

import simple.micrometer.api.FunctionCounter;
import simple.micrometer.api.Meter;

import java.lang.ref.WeakReference;
import java.util.function.ToDoubleFunction;

public class CumulativeFunctionCounter<T> implements FunctionCounter {

    private final Meter.Id id;
    private final WeakReference<T> ref;
    private final ToDoubleFunction<T> f;
    private volatile double last;

    public CumulativeFunctionCounter(Meter.Id id, T obj, ToDoubleFunction<T> f) {
        this.id = id;
        this.ref = new WeakReference<>(obj);
        this.f = f;
    }

    @Override
    public double count() {
        T obj = ref.get();
        return obj != null ? (last = f.applyAsDouble(obj)) : last;
    }

    @Override
    public Meter.Id getId() {
        return id;
    }
}
```

### `[NEW]` `src/main/java/simple/micrometer/internal/CumulativeFunctionTimer.java`

```java
package simple.micrometer.internal;

import simple.micrometer.api.FunctionTimer;
import simple.micrometer.api.Meter;
import simple.micrometer.api.TimeGauge;

import java.lang.ref.WeakReference;
import java.util.concurrent.TimeUnit;
import java.util.function.ToDoubleFunction;
import java.util.function.ToLongFunction;

public class CumulativeFunctionTimer<T> implements FunctionTimer {

    private final Meter.Id id;
    private final WeakReference<T> ref;
    private final ToLongFunction<T> countFunction;
    private final ToDoubleFunction<T> totalTimeFunction;
    private final TimeUnit totalTimeFunctionUnit;
    private final TimeUnit baseTimeUnit;

    private volatile long lastCount;
    private volatile double lastTime;

    public CumulativeFunctionTimer(Meter.Id id, T obj,
                                   ToLongFunction<T> countFunction,
                                   ToDoubleFunction<T> totalTimeFunction,
                                   TimeUnit totalTimeFunctionUnit,
                                   TimeUnit baseTimeUnit) {
        this.id = id;
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
                    TimeGauge.convertTime(totalTimeFunction.applyAsDouble(obj),
                            totalTimeFunctionUnit, baseTimeUnit),
                    0);
        }
        return TimeGauge.convertTime(lastTime, baseTimeUnit, unit);
    }

    @Override
    public TimeUnit baseTimeUnit() {
        return baseTimeUnit;
    }

    @Override
    public Meter.Id getId() {
        return id;
    }
}
```

### `[NEW]` `src/main/java/simple/micrometer/internal/DefaultTimeGauge.java`

```java
package simple.micrometer.internal;

import simple.micrometer.api.Meter;
import simple.micrometer.api.TimeGauge;

import java.lang.ref.WeakReference;
import java.util.concurrent.TimeUnit;
import java.util.function.ToDoubleFunction;

public class DefaultTimeGauge<T> implements TimeGauge {

    private final Meter.Id id;
    private final WeakReference<T> ref;
    private final ToDoubleFunction<T> f;
    private final TimeUnit fUnits;

    public DefaultTimeGauge(Meter.Id id, T obj, TimeUnit fUnits, ToDoubleFunction<T> f) {
        this.id = id;
        this.ref = new WeakReference<>(obj);
        this.f = f;
        this.fUnits = fUnits;
    }

    @Override
    public double value() {
        T obj = ref.get();
        if (obj != null) {
            try {
                return f.applyAsDouble(obj);
            } catch (Exception ex) {
                return Double.NaN;
            }
        }
        return Double.NaN;
    }

    @Override
    public TimeUnit baseTimeUnit() {
        return fUnits;
    }

    @Override
    public Meter.Id getId() {
        return id;
    }
}
```

### `[NEW]` `src/main/java/simple/micrometer/internal/NoopFunctionCounter.java`

```java
package simple.micrometer.internal;

import simple.micrometer.api.FunctionCounter;
import simple.micrometer.api.Meter;

public class NoopFunctionCounter implements FunctionCounter {

    private final Meter.Id id;

    public NoopFunctionCounter(Meter.Id id) {
        this.id = id;
    }

    @Override
    public double count() {
        return 0;
    }

    @Override
    public Meter.Id getId() {
        return id;
    }
}
```

### `[NEW]` `src/main/java/simple/micrometer/internal/NoopFunctionTimer.java`

```java
package simple.micrometer.internal;

import simple.micrometer.api.FunctionTimer;
import simple.micrometer.api.Meter;

import java.util.concurrent.TimeUnit;

public class NoopFunctionTimer implements FunctionTimer {

    private final Meter.Id id;

    public NoopFunctionTimer(Meter.Id id) {
        this.id = id;
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

    @Override
    public Meter.Id getId() {
        return id;
    }
}
```

### `[NEW]` `src/main/java/simple/micrometer/internal/NoopTimeGauge.java`

```java
package simple.micrometer.internal;

import simple.micrometer.api.Meter;
import simple.micrometer.api.TimeGauge;

import java.util.concurrent.TimeUnit;

public class NoopTimeGauge implements TimeGauge {

    private final Meter.Id id;

    public NoopTimeGauge(Meter.Id id) {
        this.id = id;
    }

    @Override
    public double value() {
        return 0;
    }

    @Override
    public TimeUnit baseTimeUnit() {
        return TimeUnit.NANOSECONDS;
    }

    @Override
    public Meter.Id getId() {
        return id;
    }
}
```

### `[MODIFIED]` `src/main/java/simple/micrometer/api/MeterRegistry.java`

**Changes**: Added 3 abstract factory methods (`newFunctionCounter`, `newFunctionTimer`,
`newTimeGauge`), added 3 registration methods to the `More` inner class, added imports
for the new types.

**Specific additions to abstract factory methods** (after `newLongTaskTimer`):

```java
protected abstract <T> FunctionCounter newFunctionCounter(Meter.Id id, T obj, ToDoubleFunction<T> countFunction);

protected abstract <T> FunctionTimer newFunctionTimer(Meter.Id id, T obj,
        ToLongFunction<T> countFunction, ToDoubleFunction<T> totalTimeFunction,
        TimeUnit totalTimeFunctionUnit);

protected abstract <T> TimeGauge newTimeGauge(Meter.Id id, T obj, TimeUnit fUnits, ToDoubleFunction<T> f);
```

**Specific additions to `More` inner class**:

```java
<T> FunctionCounter counter(Meter.Id id, T obj, ToDoubleFunction<T> countFunction) {
    return registerMeterIfNecessary(FunctionCounter.class, id,
            i -> newFunctionCounter(i, obj, countFunction), NoopFunctionCounter::new);
}

<T> FunctionTimer timer(Meter.Id id, T obj, ToLongFunction<T> countFunction,
                        ToDoubleFunction<T> totalTimeFunction, TimeUnit totalTimeFunctionUnit) {
    return registerMeterIfNecessary(FunctionTimer.class, id,
            i -> newFunctionTimer(i, obj, countFunction, totalTimeFunction, totalTimeFunctionUnit),
            NoopFunctionTimer::new);
}

<T> TimeGauge timeGauge(Meter.Id id, T obj, TimeUnit fUnits, ToDoubleFunction<T> f) {
    return registerMeterIfNecessary(TimeGauge.class, id,
            i -> newTimeGauge(i, obj, fUnits, f), NoopTimeGauge::new);
}
```

### `[MODIFIED]` `src/main/java/simple/micrometer/internal/SimpleMeterRegistry.java`

**Changes**: Implemented the 3 new abstract factory methods.

```java
@Override
protected <T> FunctionCounter newFunctionCounter(Meter.Id id, T obj, ToDoubleFunction<T> countFunction) {
    return new CumulativeFunctionCounter<>(id, obj, countFunction);
}

@Override
protected <T> FunctionTimer newFunctionTimer(Meter.Id id, T obj,
        ToLongFunction<T> countFunction, ToDoubleFunction<T> totalTimeFunction,
        TimeUnit totalTimeFunctionUnit) {
    return new CumulativeFunctionTimer<>(id, obj, countFunction, totalTimeFunction,
            totalTimeFunctionUnit, TimeUnit.NANOSECONDS);
}

@Override
protected <T> TimeGauge newTimeGauge(Meter.Id id, T obj, TimeUnit fUnits, ToDoubleFunction<T> f) {
    return new DefaultTimeGauge<>(id, obj, fUnits, f);
}
```

### `[MODIFIED]` `src/main/java/simple/micrometer/api/CompositeMeterRegistry.java`

**Changes**: Implemented the 3 new abstract factory methods. Uses cumulative/default
implementations directly since function-tracking meters are pull-based (no composite
fan-out needed).

```java
@Override
protected <T> FunctionCounter newFunctionCounter(Meter.Id id, T obj, ToDoubleFunction<T> countFunction) {
    return new CumulativeFunctionCounter<>(id, obj, countFunction);
}

@Override
protected <T> FunctionTimer newFunctionTimer(Meter.Id id, T obj,
        ToLongFunction<T> countFunction, ToDoubleFunction<T> totalTimeFunction,
        TimeUnit totalTimeFunctionUnit) {
    return new CumulativeFunctionTimer<>(id, obj, countFunction, totalTimeFunction,
            totalTimeFunctionUnit, TimeUnit.NANOSECONDS);
}

@Override
protected <T> TimeGauge newTimeGauge(Meter.Id id, T obj, TimeUnit fUnits, ToDoubleFunction<T> f) {
    return new DefaultTimeGauge<>(id, obj, fUnits, f);
}
```

### `[NEW]` `src/test/java/simple/micrometer/api/FunctionMeterTest.java`

```java
package simple.micrometer.api;

import simple.micrometer.internal.SimpleMeterRegistry;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Nested;
import org.junit.jupiter.api.Test;

import java.util.concurrent.TimeUnit;
import java.util.concurrent.atomic.AtomicLong;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.within;

class FunctionMeterTest {

    MeterRegistry registry;

    @BeforeEach
    void setUp() {
        registry = new SimpleMeterRegistry();
    }

    @Nested
    class FunctionCounterTests {

        @Test
        void shouldTrackExternalCount() {
            var cache = new FakeCache();
            FunctionCounter hitCounter = FunctionCounter
                    .builder("cache.hits", cache, FakeCache::getHitCount)
                    .register(registry);

            assertThat(hitCounter.count()).isEqualTo(0);

            cache.simulateHits(7);
            assertThat(hitCounter.count()).isEqualTo(7);

            cache.simulateHits(3);
            assertThat(hitCounter.count()).isEqualTo(10);
        }

        @Test
        void shouldSupportTags() {
            var cache = new FakeCache();
            FunctionCounter counter = FunctionCounter
                    .builder("cache.hits", cache, FakeCache::getHitCount)
                    .tag("cache", "users")
                    .description("Cache hit count")
                    .baseUnit("hits")
                    .register(registry);

            assertThat(counter.getId().getName()).isEqualTo("cache.hits");
            assertThat(counter.getId().getTag("cache")).isEqualTo("users");
            assertThat(counter.getId().getDescription()).isEqualTo("Cache hit count");
            assertThat(counter.getId().getBaseUnit()).isEqualTo("hits");
        }

        @Test
        void shouldDeduplicateByNameAndTags() {
            var cache = new FakeCache();
            FunctionCounter first = FunctionCounter
                    .builder("cache.hits", cache, FakeCache::getHitCount)
                    .tag("cache", "users")
                    .register(registry);

            FunctionCounter second = FunctionCounter
                    .builder("cache.hits", cache, FakeCache::getHitCount)
                    .tag("cache", "users")
                    .register(registry);

            assertThat(first).isSameAs(second);
        }

        @Test
        void shouldProduceMeasurements() {
            var cache = new FakeCache();
            cache.simulateHits(42);
            FunctionCounter counter = FunctionCounter
                    .builder("cache.hits", cache, FakeCache::getHitCount)
                    .register(registry);

            Iterable<Measurement> measurements = counter.measure();
            assertThat(measurements).hasSize(1);

            Measurement m = measurements.iterator().next();
            assertThat(m.getValue()).isEqualTo(42);
            assertThat(m.getStatistic()).isEqualTo(Statistic.COUNT);
        }

        @Test
        void shouldBeDiscoverableViaRegistrySearch() {
            var cache = new FakeCache();
            FunctionCounter.builder("cache.hits", cache, FakeCache::getHitCount)
                    .tag("cache", "users")
                    .register(registry);

            assertThat(registry.getMeters()).hasSize(1);
            Meter meter = registry.getMeters().get(0);
            assertThat(meter).isInstanceOf(FunctionCounter.class);
        }
    }

    @Nested
    class FunctionTimerTests {

        @Test
        void shouldTrackExternalCountAndTotalTime() {
            var cache = new FakeCache();
            FunctionTimer timer = FunctionTimer
                    .builder("cache.gets", cache,
                            FakeCache::getRequestCount,
                            FakeCache::getTotalLoadTimeNanos,
                            TimeUnit.NANOSECONDS)
                    .register(registry);

            assertThat(timer.count()).isEqualTo(0);
            assertThat(timer.totalTime(TimeUnit.NANOSECONDS)).isEqualTo(0);

            cache.simulateRequests(5, 100_000_000L);
            assertThat(timer.count()).isEqualTo(5);
            assertThat(timer.totalTime(TimeUnit.MILLISECONDS))
                    .isCloseTo(100.0, within(0.1));
        }

        @Test
        void shouldComputeMeanTime() {
            var cache = new FakeCache();
            cache.simulateRequests(10, 500_000_000L);

            FunctionTimer timer = FunctionTimer
                    .builder("cache.gets", cache,
                            FakeCache::getRequestCount,
                            FakeCache::getTotalLoadTimeNanos,
                            TimeUnit.NANOSECONDS)
                    .register(registry);

            assertThat(timer.mean(TimeUnit.MILLISECONDS))
                    .isCloseTo(50.0, within(0.1));
        }

        @Test
        void shouldReturnZeroMeanWhenCountIsZero() {
            var cache = new FakeCache();
            FunctionTimer timer = FunctionTimer
                    .builder("cache.gets", cache,
                            FakeCache::getRequestCount,
                            FakeCache::getTotalLoadTimeNanos,
                            TimeUnit.NANOSECONDS)
                    .register(registry);

            assertThat(timer.mean(TimeUnit.MILLISECONDS)).isEqualTo(0);
        }

        @Test
        void shouldConvertTimeUnits() {
            var cache = new FakeCache();
            cache.simulateRequests(1, 1_000_000_000L);

            FunctionTimer timer = FunctionTimer
                    .builder("cache.gets", cache,
                            FakeCache::getRequestCount,
                            FakeCache::getTotalLoadTimeNanos,
                            TimeUnit.NANOSECONDS)
                    .register(registry);

            assertThat(timer.totalTime(TimeUnit.SECONDS))
                    .isCloseTo(1.0, within(0.001));
            assertThat(timer.totalTime(TimeUnit.MILLISECONDS))
                    .isCloseTo(1000.0, within(0.1));
        }

        @Test
        void shouldSupportTags() {
            var cache = new FakeCache();
            FunctionTimer timer = FunctionTimer
                    .builder("cache.gets", cache,
                            FakeCache::getRequestCount,
                            FakeCache::getTotalLoadTimeNanos,
                            TimeUnit.NANOSECONDS)
                    .tag("cache", "users")
                    .description("Cache get latency")
                    .register(registry);

            assertThat(timer.getId().getName()).isEqualTo("cache.gets");
            assertThat(timer.getId().getTag("cache")).isEqualTo("users");
        }

        @Test
        void shouldProduceMeasurements() {
            var cache = new FakeCache();
            cache.simulateRequests(3, 300_000_000L);

            FunctionTimer timer = FunctionTimer
                    .builder("cache.gets", cache,
                            FakeCache::getRequestCount,
                            FakeCache::getTotalLoadTimeNanos,
                            TimeUnit.NANOSECONDS)
                    .register(registry);

            Iterable<Measurement> measurements = timer.measure();
            assertThat(measurements).hasSize(2);
        }

        @Test
        void shouldAcceptMillisecondSourceUnit() {
            var cache = new FakeCache();
            cache.simulateRequests(2, 0);

            FunctionTimer timer = FunctionTimer
                    .builder("cache.gets", cache,
                            FakeCache::getRequestCount,
                            c -> 500.0,
                            TimeUnit.MILLISECONDS)
                    .register(registry);

            assertThat(timer.totalTime(TimeUnit.SECONDS))
                    .isCloseTo(0.5, within(0.001));
        }
    }

    @Nested
    class TimeGaugeTests {

        @Test
        void shouldTrackTimeValue() {
            var runtime = new FakeRuntime();
            runtime.setUptime(5000);

            TimeGauge uptime = TimeGauge
                    .builder("process.uptime", runtime,
                            TimeUnit.MILLISECONDS,
                            FakeRuntime::getUptime)
                    .register(registry);

            assertThat(uptime.value(TimeUnit.SECONDS))
                    .isCloseTo(5.0, within(0.001));
            assertThat(uptime.value(TimeUnit.MILLISECONDS))
                    .isCloseTo(5000.0, within(0.1));
        }

        @Test
        void shouldAutoConvertUnits() {
            var runtime = new FakeRuntime();
            runtime.setUptime(120_000);

            TimeGauge uptime = TimeGauge
                    .builder("process.uptime", runtime,
                            TimeUnit.MILLISECONDS,
                            FakeRuntime::getUptime)
                    .register(registry);

            assertThat(uptime.value(TimeUnit.MINUTES))
                    .isCloseTo(2.0, within(0.001));
        }

        @Test
        void shouldBeAGauge() {
            var runtime = new FakeRuntime();
            runtime.setUptime(1000);

            TimeGauge uptime = TimeGauge
                    .builder("process.uptime", runtime,
                            TimeUnit.MILLISECONDS,
                            FakeRuntime::getUptime)
                    .register(registry);

            assertThat(uptime.value()).isCloseTo(1000.0, within(0.1));
        }

        @Test
        void shouldSupportTags() {
            var runtime = new FakeRuntime();
            TimeGauge uptime = TimeGauge
                    .builder("process.uptime", runtime,
                            TimeUnit.MILLISECONDS,
                            FakeRuntime::getUptime)
                    .tag("host", "prod-1")
                    .description("JVM uptime")
                    .register(registry);

            assertThat(uptime.getId().getName()).isEqualTo("process.uptime");
            assertThat(uptime.getId().getTag("host")).isEqualTo("prod-1");
        }

        @Test
        void shouldDeduplicateByNameAndTags() {
            var runtime = new FakeRuntime();
            TimeGauge first = TimeGauge
                    .builder("process.uptime", runtime,
                            TimeUnit.MILLISECONDS,
                            FakeRuntime::getUptime)
                    .register(registry);

            TimeGauge second = TimeGauge
                    .builder("process.uptime", runtime,
                            TimeUnit.MILLISECONDS,
                            FakeRuntime::getUptime)
                    .register(registry);

            assertThat(first).isSameAs(second);
        }

        @Test
        void shouldReflectChangingValues() {
            var runtime = new FakeRuntime();
            runtime.setUptime(1000);

            TimeGauge uptime = TimeGauge
                    .builder("process.uptime", runtime,
                            TimeUnit.MILLISECONDS,
                            FakeRuntime::getUptime)
                    .register(registry);

            assertThat(uptime.value(TimeUnit.SECONDS))
                    .isCloseTo(1.0, within(0.001));

            runtime.setUptime(10_000);
            assertThat(uptime.value(TimeUnit.SECONDS))
                    .isCloseTo(10.0, within(0.001));
        }
    }

    @Nested
    class FilterIntegrationTests {

        @Test
        void shouldReturnNoopFunctionCounterWhenDenied() {
            registry.config().meterFilter(MeterFilter.deny(id ->
                    id.getName().startsWith("cache")));

            var cache = new FakeCache();
            cache.simulateHits(100);

            FunctionCounter counter = FunctionCounter
                    .builder("cache.hits", cache, FakeCache::getHitCount)
                    .register(registry);

            assertThat(counter.count()).isEqualTo(0);
            assertThat(registry.getMeters()).isEmpty();
        }

        @Test
        void shouldReturnNoopFunctionTimerWhenDenied() {
            registry.config().meterFilter(MeterFilter.deny(id ->
                    id.getName().startsWith("cache")));

            var cache = new FakeCache();
            cache.simulateRequests(10, 1_000_000_000L);

            FunctionTimer timer = FunctionTimer
                    .builder("cache.gets", cache,
                            FakeCache::getRequestCount,
                            FakeCache::getTotalLoadTimeNanos,
                            TimeUnit.NANOSECONDS)
                    .register(registry);

            assertThat(timer.count()).isEqualTo(0);
            assertThat(timer.totalTime(TimeUnit.MILLISECONDS)).isEqualTo(0);
        }

        @Test
        void shouldReturnNoopTimeGaugeWhenDenied() {
            registry.config().meterFilter(MeterFilter.deny(id ->
                    id.getName().startsWith("process")));

            var runtime = new FakeRuntime();
            runtime.setUptime(5000);

            TimeGauge uptime = TimeGauge
                    .builder("process.uptime", runtime,
                            TimeUnit.MILLISECONDS,
                            FakeRuntime::getUptime)
                    .register(registry);

            assertThat(uptime.value()).isEqualTo(0);
            assertThat(uptime.value(TimeUnit.SECONDS)).isEqualTo(0);
        }
    }

    static class FakeCache {

        private final AtomicLong hitCount = new AtomicLong();
        private final AtomicLong requestCount = new AtomicLong();
        private final AtomicLong totalLoadTimeNanos = new AtomicLong();

        void simulateHits(int hits) {
            hitCount.addAndGet(hits);
        }

        void simulateRequests(int count, long totalNanos) {
            requestCount.addAndGet(count);
            totalLoadTimeNanos.addAndGet(totalNanos);
        }

        double getHitCount() {
            return hitCount.get();
        }

        long getRequestCount() {
            return requestCount.get();
        }

        double getTotalLoadTimeNanos() {
            return totalLoadTimeNanos.get();
        }
    }

    static class FakeRuntime {

        private volatile double uptimeMillis;

        void setUptime(double millis) {
            this.uptimeMillis = millis;
        }

        double getUptime() {
            return uptimeMillis;
        }
    }
}
```
