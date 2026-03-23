# Chapter 20: Metrics Core

> Line references based on commit `2c8a4606c` of the Micrometer repository.

## Build Challenge

| Current State | Limitation | Objective |
|---------------|------------|-----------|
| The iris-boot application can serve HTTP requests, configure beans, and shut down gracefully — but has no way to measure what's happening at runtime | Cannot count requests, time operations, or track queue sizes — no quantitative visibility into application behavior | Build a standalone metrics library with Counter, Timer, Gauge, and a MeterRegistry that stores, deduplicates, and creates meters by name+tags |

---

## 20.1 The Foundation: MeterRegistry and Meter.Id

This is a **foundation chapter** — there is no existing system to integrate with. The `iris-micrometer` module is standalone (no Spring dependency), mirroring how real Micrometer is an independent library that Spring Boot later integrates via auto-configuration.

The core data structure is `MeterRegistry` — a `ConcurrentHashMap<Meter.Id, Meter>` that all future features (Observation API, Actuator MetricsEndpoint) will connect to.

**New file:** `iris-micrometer/src/main/java/com/iris/micrometer/core/MeterRegistry.java`

```java
package com.iris.micrometer.core;

import java.util.*;
import java.util.concurrent.ConcurrentHashMap;
import java.util.function.ToDoubleFunction;

public abstract class MeterRegistry {

    private final Map<Meter.Id, Meter> meterMap = new ConcurrentHashMap<>();
    private final Object meterMapLock = new Object();

    // --- Abstract factory methods (subclasses create concrete meter instances) ---
    protected abstract Counter newCounter(Meter.Id id);
    protected abstract Timer newTimer(Meter.Id id);
    protected abstract <T> Gauge newGauge(Meter.Id id, T obj, ToDoubleFunction<T> valueFunction);
    protected abstract java.util.concurrent.TimeUnit getBaseTimeUnit();

    // --- Registration with deduplication ---
    public Counter counter(Meter.Id id) {
        return getOrCreateMeter(id, Counter.class, this::newCounter);
    }

    public Counter counter(String name, String... tags) {
        return counter(new Meter.Id(name, Tags.of(tags), Meter.Type.COUNTER, null, null));
    }

    // ... timer() and gauge() follow the same pattern

    @SuppressWarnings("unchecked")
    private <M extends Meter> M getOrCreateMeter(Meter.Id id, Class<M> meterClass,
                                                  java.util.function.Function<Meter.Id, M> factory) {
        // Fast path — no lock
        Meter existing = meterMap.get(id);
        if (existing != null) {
            if (meterClass.isInstance(existing)) return (M) existing;
            throw meterTypeMismatch(id, meterClass, existing);
        }
        // Slow path — synchronized
        synchronized (meterMapLock) {
            existing = meterMap.get(id);
            if (existing != null) {
                if (meterClass.isInstance(existing)) return (M) existing;
                throw meterTypeMismatch(id, meterClass, existing);
            }
            M meter = factory.apply(id);
            meterMap.put(id, meter);
            return meter;
        }
    }
}
```

Two key decisions here:

1. **`ConcurrentHashMap` for reads, `synchronized` for writes** — allows lock-free meter lookup on the hot path (every `counter.increment()` first needs to find the counter), while protecting creation from races. This is the same double-checked locking pattern used in real Micrometer's `MeterRegistry.getOrCreateMeter()`.

2. **Abstract Factory pattern** — `MeterRegistry` defines the skeleton (storage, deduplication, convenience methods); subclasses implement `newCounter()`, `newTimer()`, `newGauge()` to choose concrete implementations. A `SimpleMeterRegistry` creates in-memory meters; a future `PrometheusRegistry` would create Prometheus-specific meters.

This registry is the thing all future features will connect to:
- **Observation API (ch21)** will bridge observations to this registry via a handler
- **Actuator MetricsEndpoint (ch24)** will expose `getMeters()` as JSON at `/actuator/metrics`
- **Auto-configuration** will create and wire a `MeterRegistry` bean

To make the registry work, we need:
- `Meter` interface with `Meter.Id` (the identity that drives deduplication)
- `Tag` / `Tags` (the dimension system that makes `Meter.Id` unique)
- Three concrete meter types: `Counter`, `Timer`, `Gauge`
- `SimpleMeterRegistry` (the in-memory concrete registry)

## 20.2 Tags — The Dimension System

Tags are key/value pairs that add context to metrics. Without tags, `http.requests` is a single counter; with tags like `method=GET, status=200`, it becomes a multi-dimensional metric.

**New file:** `iris-micrometer/src/main/java/com/iris/micrometer/core/Tag.java`

```java
package com.iris.micrometer.core;

public interface Tag extends Comparable<Tag> {

    String getKey();

    String getValue();

    static Tag of(String key, String value) {
        return new ImmutableTag(key, value);
    }

    @Override
    default int compareTo(Tag o) {
        return getKey().compareTo(o.getKey());
    }
}
```

**New file:** `iris-micrometer/src/main/java/com/iris/micrometer/core/ImmutableTag.java`

```java
package com.iris.micrometer.core;

import java.util.Objects;

final class ImmutableTag implements Tag {

    private final String key;
    private final String value;

    ImmutableTag(String key, String value) {
        Objects.requireNonNull(key, "tag key must not be null");
        Objects.requireNonNull(value, "tag value must not be null");
        this.key = key;
        this.value = value;
    }

    @Override
    public String getKey() { return key; }

    @Override
    public String getValue() { return value; }

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof Tag tag)) return false;
        return key.equals(tag.getKey()) && value.equals(tag.getValue());
    }

    @Override
    public int hashCode() {
        return 31 * key.hashCode() + value.hashCode();
    }

    @Override
    public String toString() {
        return "tag(" + key + "=" + value + ")";
    }
}
```

**New file:** `iris-micrometer/src/main/java/com/iris/micrometer/core/Tags.java`

```java
package com.iris.micrometer.core;

import java.util.*;
import java.util.stream.Stream;
import java.util.stream.StreamSupport;

public final class Tags implements Iterable<Tag> {

    private static final Tag[] EMPTY_ARRAY = new Tag[0];
    private static final Tags EMPTY = new Tags(EMPTY_ARRAY);

    private final Tag[] tags;

    private Tags(Tag[] tags) {
        this.tags = tags;
    }

    public static Tags empty() {
        return EMPTY;
    }

    public static Tags of(String... keyValues) {
        if (keyValues.length == 0) return EMPTY;
        if (keyValues.length % 2 != 0) {
            throw new IllegalArgumentException(
                "Tags must be key/value pairs, got odd number: " + keyValues.length);
        }
        Tag[] result = new Tag[keyValues.length / 2];
        for (int i = 0; i < keyValues.length; i += 2) {
            result[i / 2] = Tag.of(keyValues[i], keyValues[i + 1]);
        }
        return toTags(result);
    }

    public static Tags of(Tag... tags) {
        if (tags.length == 0) return EMPTY;
        return toTags(tags.clone());
    }

    public static Tags of(Iterable<Tag> tags) {
        if (tags instanceof Tags t) return t;
        Tag[] array = StreamSupport.stream(tags.spliterator(), false).toArray(Tag[]::new);
        if (array.length == 0) return EMPTY;
        return toTags(array);
    }

    /** Merges two sorted Tags in O(n+m). On key conflict, other's value wins. */
    public Tags and(Tags other) {
        if (this.tags.length == 0) return other;
        if (other.tags.length == 0) return this;

        List<Tag> merged = new ArrayList<>(this.tags.length + other.tags.length);
        int i = 0, j = 0;
        while (i < this.tags.length && j < other.tags.length) {
            int cmp = this.tags[i].getKey().compareTo(other.tags[j].getKey());
            if (cmp < 0) merged.add(this.tags[i++]);
            else if (cmp > 0) merged.add(other.tags[j++]);
            else { merged.add(other.tags[j++]); i++; } // other wins
        }
        while (i < this.tags.length) merged.add(this.tags[i++]);
        while (j < other.tags.length) merged.add(other.tags[j++]);
        return new Tags(merged.toArray(EMPTY_ARRAY));
    }

    /** Sorts and deduplicates. On duplicate keys, last value wins. */
    private static Tags toTags(Tag[] raw) {
        Arrays.sort(raw);
        int write = 0;
        for (int read = 0; read < raw.length; read++) {
            if (read + 1 < raw.length && raw[read].getKey().equals(raw[read + 1].getKey())) {
                continue;
            }
            raw[write++] = raw[read];
        }
        if (write == raw.length) return new Tags(raw);
        return new Tags(Arrays.copyOf(raw, write));
    }
    // ... equals, hashCode, iterator, toString
}
```

## 20.3 Meter and Meter.Id — The Identity Contract

**New file:** `iris-micrometer/src/main/java/com/iris/micrometer/core/Meter.java`

```java
package com.iris.micrometer.core;

import java.util.Objects;

public interface Meter {

    Id getId();

    enum Type { COUNTER, GAUGE, TIMER, OTHER }

    class Id {
        private final String name;
        private final Tags tags;
        private final Type type;
        private final String description;
        private final String baseUnit;

        public Id(String name, Tags tags, Type type, String description, String baseUnit) {
            Objects.requireNonNull(name, "meter name must not be null");
            this.name = name;
            this.tags = (tags != null) ? tags : Tags.empty();
            this.type = type;
            this.description = description;
            this.baseUnit = baseUnit;
        }

        // Only name and tags define identity — NOT type, description, or baseUnit
        @Override
        public boolean equals(Object o) {
            if (this == o) return true;
            if (!(o instanceof Id id)) return false;
            return name.equals(id.name) && tags.equals(id.tags);
        }

        @Override
        public int hashCode() {
            int result = name.hashCode();
            result = 31 * result + tags.hashCode();
            return result;
        }
        // ... getters, withTags(), toString()
    }
}
```

## 20.4 Three Meter Types: Counter, Timer, Gauge

### Counter — monotonically increasing

**New file:** `iris-micrometer/src/main/java/com/iris/micrometer/core/Counter.java`

```java
public interface Counter extends Meter {
    default void increment() { increment(1.0); }
    void increment(double amount);
    double count();

    static Builder builder(String name) { return new Builder(name); }

    class Builder {
        // name, tags, description, baseUnit — fluent setters
        public Counter register(MeterRegistry registry) {
            return registry.counter(new Meter.Id(name, tags, Meter.Type.COUNTER, description, baseUnit));
        }
    }
}
```

**New file:** `iris-micrometer/src/main/java/com/iris/micrometer/core/DefaultCounter.java`

```java
package com.iris.micrometer.core;

import java.util.concurrent.atomic.DoubleAdder;

public class DefaultCounter implements Counter {
    private final Id id;
    private final DoubleAdder value = new DoubleAdder();

    public DefaultCounter(Id id) { this.id = id; }

    @Override
    public void increment(double amount) {
        if (amount > 0) value.add(amount);
    }

    @Override
    public double count() { return value.sum(); }

    @Override
    public Id getId() { return id; }
}
```

### Timer — short-duration events with count, total, and max

**New file:** `iris-micrometer/src/main/java/com/iris/micrometer/core/Timer.java`

```java
public interface Timer extends Meter {
    void record(long amount, TimeUnit unit);
    default void record(Duration duration) { record(duration.toNanos(), TimeUnit.NANOSECONDS); }

    default void record(Runnable runnable) {
        long start = System.nanoTime();
        try { runnable.run(); }
        finally { record(System.nanoTime() - start, TimeUnit.NANOSECONDS); }
    }

    default <T> T record(Supplier<T> supplier) { /* same pattern */ }

    long count();
    double totalTime(TimeUnit unit);
    default double mean(TimeUnit unit) { return count() == 0 ? 0 : totalTime(unit) / count(); }
    double max(TimeUnit unit);

    /** Captures start time; timer chosen at stop-time for deferred tag decisions. */
    static Sample start() { return new Sample(System.nanoTime()); }

    class Sample {
        private final long startNanos;
        public long stop(Timer timer) {
            long elapsed = System.nanoTime() - startNanos;
            timer.record(elapsed, TimeUnit.NANOSECONDS);
            return elapsed;
        }
    }
}
```

**New file:** `iris-micrometer/src/main/java/com/iris/micrometer/core/DefaultTimer.java`

```java
public class DefaultTimer implements Timer {
    private final Id id;
    private final LongAdder count = new LongAdder();
    private final DoubleAdder totalNanos = new DoubleAdder();
    private final AtomicLong maxNanos = new AtomicLong(0);

    @Override
    public void record(long amount, TimeUnit unit) {
        if (amount >= 0) {
            long nanos = unit.toNanos(amount);
            count.increment();
            totalNanos.add(nanos);
            updateMax(nanos);
        }
    }

    private void updateMax(long nanos) {
        long currentMax;
        do {
            currentMax = maxNanos.get();
            if (nanos <= currentMax) return;
        } while (!maxNanos.compareAndSet(currentMax, nanos));
    }
}
```

### Gauge — instantaneous pull-based value

**New file:** `iris-micrometer/src/main/java/com/iris/micrometer/core/Gauge.java`

```java
public interface Gauge extends Meter {
    double value();

    static <T> Builder<T> builder(String name, T obj, ToDoubleFunction<T> valueFunction) {
        return new Builder<>(name, obj, valueFunction);
    }

    static Builder<Supplier<Number>> builder(String name, Supplier<Number> supplier) {
        return new Builder<>(name, supplier, s -> {
            Number val = s.get();
            return (val != null) ? val.doubleValue() : Double.NaN;
        });
    }
}
```

**New file:** `iris-micrometer/src/main/java/com/iris/micrometer/core/DefaultGauge.java`

```java
public class DefaultGauge<T> implements Gauge {
    private final Id id;
    private final T obj;
    private final ToDoubleFunction<T> valueFunction;

    @Override
    public double value() {
        try { return valueFunction.applyAsDouble(obj); }
        catch (Exception e) { return Double.NaN; }
    }
}
```

## 20.5 SimpleMeterRegistry — The Concrete Registry

**New file:** `iris-micrometer/src/main/java/com/iris/micrometer/core/simple/SimpleMeterRegistry.java`

```java
package com.iris.micrometer.core.simple;

import com.iris.micrometer.core.*;
import java.util.concurrent.TimeUnit;
import java.util.function.ToDoubleFunction;

public class SimpleMeterRegistry extends MeterRegistry {

    @Override
    protected Counter newCounter(Meter.Id id) {
        return new DefaultCounter(id);
    }

    @Override
    protected Timer newTimer(Meter.Id id) {
        return new DefaultTimer(id);
    }

    @Override
    protected <T> Gauge newGauge(Meter.Id id, T obj, ToDoubleFunction<T> valueFunction) {
        return new DefaultGauge<>(id, obj, valueFunction);
    }

    @Override
    protected TimeUnit getBaseTimeUnit() {
        return TimeUnit.SECONDS;
    }
}
```

## 20.6 Try It Yourself

<details>
<summary>Challenge 1: Implement a thread-safe max tracker without locks</summary>

The `DefaultTimer` needs to track the maximum recorded duration. Using `synchronized` would work but creates contention. How can you use `AtomicLong.compareAndSet()` to implement a lock-free max?

```java
private void updateMax(long nanos) {
    long currentMax;
    do {
        currentMax = maxNanos.get();
        if (nanos <= currentMax) return;
    } while (!maxNanos.compareAndSet(currentMax, nanos));
}
```

This is a **compare-and-swap (CAS) loop**: read the current max, bail if our value is smaller, otherwise attempt to set it. If another thread changed the max between our read and our CAS attempt, the CAS fails and we retry. This is the same pattern used in `java.util.concurrent` lock-free data structures.

</details>

<details>
<summary>Challenge 2: Implement Timer.Sample for deferred timer selection</summary>

The key insight: you want to start timing before an operation, but choose which timer to record to AFTER the operation completes (because tags like `outcome=success` vs `outcome=error` aren't known until then).

```java
class Sample {
    private final long startNanos;

    private Sample(long startNanos) {
        this.startNanos = startNanos;
    }

    public long stop(Timer timer) {
        long elapsed = System.nanoTime() - startNanos;
        timer.record(elapsed, TimeUnit.NANOSECONDS);
        return elapsed;
    }
}
```

Usage pattern:
```java
Timer.Sample sample = Timer.start();
try {
    doWork();
    sample.stop(registry.timer("request", "outcome", "success"));
} catch (Exception e) {
    sample.stop(registry.timer("request", "outcome", "error"));
    throw e;
}
```

</details>

## 20.7 Tests

### Unit Tests

**New file:** `iris-micrometer/src/test/java/com/iris/micrometer/core/TagsTest.java`

```java
class TagsTest {
    @Test void shouldCreateEmptyTags()
    @Test void shouldCreateTagsFromKeyValuePairs()
    @Test void shouldSortTagsByKey()
    @Test void shouldDeduplicateByKey_LastValueWins()
    @Test void shouldMergeTags_OtherWinsOnConflict()
    @Test void shouldMergeWithEmptyTags()
    @Test void shouldBeEqualForSameContent()
    @Test void shouldThrowOnOddNumberOfKeyValues()
    @Test void shouldCreateFromTagInstances()
    @Test void shouldCreateFromIterable()
    @Test void shouldAndWithKeyValuePairs()
}
```

**New file:** `iris-micrometer/src/test/java/com/iris/micrometer/core/MeterIdTest.java`

```java
class MeterIdTest {
    @Test void shouldBeEqualByNameAndTagsOnly()      // type, desc, unit ignored
    @Test void shouldNotBeEqualWithDifferentNames()
    @Test void shouldNotBeEqualWithDifferentTags()
    @Test void shouldMergeAdditionalTagsViaWithTags()
    @Test void shouldDefaultToEmptyTags_WhenNullProvided()
}
```

**New file:** `iris-micrometer/src/test/java/com/iris/micrometer/core/CounterTest.java`

```java
class CounterTest {
    @Test void shouldStartAtZero()
    @Test void shouldIncrementByOne()
    @Test void shouldIncrementByAmount()
    @Test void shouldAccumulateMultipleIncrements()
    @Test void shouldIgnoreNegativeIncrements()
    @Test void shouldCreateViaFluentBuilder()
    @Test void shouldCreateDistinctCountersForDifferentTags()
    @Test void shouldReturnSameInstance_WhenSameNameAndTags()
}
```

**New file:** `iris-micrometer/src/test/java/com/iris/micrometer/core/TimerTest.java`

```java
class TimerTest {
    @Test void shouldStartWithZeroCounts()
    @Test void shouldRecordDuration()
    @Test void shouldRecordDurationObject()
    @Test void shouldAccumulateMultipleRecordings()
    @Test void shouldTrackMaxDuration()
    @Test void shouldCalculateMean()
    @Test void shouldReturnZeroMean_WhenNoRecordings()
    @Test void shouldTimeRunnable()
    @Test void shouldTimeSupplierAndReturnResult()
    @Test void shouldTimeCallableAndReturnResult()
    @Test void shouldUseSampleForDeferredTimerChoice()
    @Test void shouldCreateViaFluentBuilder()
    @Test void shouldReturnSameInstance_WhenSameNameAndTags()
}
```

**New file:** `iris-micrometer/src/test/java/com/iris/micrometer/core/GaugeTest.java`

```java
class GaugeTest {
    @Test void shouldTrackCollectionSize()
    @Test void shouldTrackAtomicValue()
    @Test void shouldTrackNumberSupplier()
    @Test void shouldReturnNaN_WhenFunctionThrows()
    @Test void shouldCreateWithDescriptionAndUnit()
    @Test void shouldReturnSameInstance_WhenSameNameAndTags()
    @Test void shouldTrackNumberDirectly()
}
```

**New file:** `iris-micrometer/src/test/java/com/iris/micrometer/core/MeterRegistryTest.java`

```java
class MeterRegistryTest {
    @Test void shouldDeduplicateMeters_WhenSameNameAndTags()
    @Test void shouldCreateDistinctMeters_WhenDifferentTags()
    @Test void shouldThrowOnTypeMismatch_WhenSameNameAndTags()
    @Test void shouldListAllRegisteredMeters()
    @Test void shouldFindMeterById()
    @Test void shouldReturnNull_WhenMeterNotFound()
    @Test void shouldRemoveMeterById()
    @Test void shouldClearAllMeters()
    @Test void shouldSupportMultipleMeterTypes()
}
```

**Run:** `./gradlew test` — expected: all tests pass (including prior features' tests)

---

## 20.8 Why This Works

> ★ **Insight** -------------------------------------------
> - **Why identity = name + tags (not type)?** This is Micrometer's most important design decision. Two Ids with the same name and tags are "the same meter" regardless of type — if you try to register a Counter and a Timer with `http.requests` + `method=GET`, you get an `IllegalArgumentException`. This prevents ambiguity in monitoring systems where `http.requests{method=GET}` must refer to exactly one time series. The trade-off: you can't have both a counter and a timer with the same metric name, even if they measure different things. Choose your names carefully.
> - **When this breaks down:** In systems with very high tag cardinality (unique user IDs, request UUIDs), each unique tag combination creates a new meter in memory. This is called "cardinality explosion" — the `ConcurrentHashMap` grows unbounded. Real Micrometer addresses this with `MeterFilter.maximumAllowableTags()`. Our simplified version doesn't have filters, so be mindful of tag cardinality.
> -----------------------------------------------------------

> ★ **Insight** -------------------------------------------
> - **Why Abstract Factory over direct instantiation?** `MeterRegistry` is abstract because different monitoring backends need different meter implementations. A Prometheus registry stores counters as monotonic doubles with specific histogram bucket configurations. A StatsD registry doesn't store values at all — it fires UDP packets on every record. By making `newCounter()`, `newTimer()`, `newGauge()` abstract, the deduplication and convenience logic lives once in `MeterRegistry`, while each backend only implements creation. This is the same pattern as `java.util.AbstractList` — get the common behavior for free, implement only the specifics.
> - **When to use concrete:** If you'll never have more than one registry type, the abstraction adds complexity for no benefit. Real Micrometer has 15+ registry implementations (Prometheus, StatsD, Atlas, CloudWatch, Datadog, ...), which justifies the pattern.
> -----------------------------------------------------------

> ★ **Insight** -------------------------------------------
> - **Push vs. Pull: Counter/Timer vs. Gauge.** Counter and Timer are **push-based**: application code calls `increment()` or `record()`, and the meter accumulates the value internally. Gauge is **pull-based**: it holds a reference to a state object and a function that extracts the current value when observed. This means `Gauge.value()` always returns the *current* instantaneous state — there's no accumulation, no history. This is exactly the right model for things like "current queue depth" or "available memory" where the value changes outside the metrics system's control.
> - **Real Micrometer's WeakReference:** Real `DefaultGauge` uses a `WeakReference` to the state object, so if the object is garbage-collected, the gauge returns `NaN` instead of leaking memory. Our simplified version uses a strong reference for clarity — this means a gauge will keep its state object alive forever.
> -----------------------------------------------------------

## 20.9 Connection to Real Framework

| Simplified Code | Real Framework Code | File:Line | Key Difference |
|----------------|--------------------|-----------| ---------------|
| `Tag` | `Tag` | `Tag.java:26` | Real version is identical in design |
| `ImmutableTag` | `ImmutableTag` | `ImmutableTag.java:29` | Real version is identical |
| `Tags` | `Tags` | `Tags.java:36` | Real uses `Tag[]` with length field for sorted-set array; supports more overloads |
| `Meter.Id` | `Meter.Id` | `Meter.java:185` | Real adds `syntheticAssociation`, `NamingConvention` support, more `with*()` methods |
| `Meter.Id.equals()` | `Meter.Id.equals()` | `Meter.java:354` | Identical: only name + tags in equality |
| `MeterRegistry` | `MeterRegistry` | `MeterRegistry.java:86` | Real adds `MeterFilter` pipeline, `preFilterIdToMeterMap`, listener callbacks, `isClosed()` state, noop meters for denied registrations |
| `MeterRegistry.getOrCreateMeter()` | `MeterRegistry.getOrCreateMeter()` | `MeterRegistry.java:688` | Same double-checked locking pattern; real adds filter mapping/acceptance/configuration |
| `Counter` | `Counter` | `Counter.java:31` | Real adds `MeterProvider` for dynamic tagging |
| `DefaultCounter` | `CumulativeCounter` | `CumulativeCounter.java` | Identical: `DoubleAdder` for thread-safe accumulation |
| `Timer` | `Timer` | `Timer.java:49` | Real adds `wrap()` decorators, `ResourceSample` (AutoCloseable), distribution statistics |
| `DefaultTimer` | `CumulativeTimer` | `CumulativeTimer.java` | Real adds histogram support, `PauseDetector` integration |
| `Timer.Sample` | `Timer.Sample` | `Timer.java:310` | Real takes a `Clock` parameter for testability |
| `Gauge` | `Gauge` | `Gauge.java:43` | Real uses `WeakReference` by default; `strongReference(true)` opt-in |
| `DefaultGauge` | `DefaultGauge` | `DefaultGauge.java:38` | Real uses `WeakReference<T>` — returns `NaN` when GC'd |
| `SimpleMeterRegistry` | `SimpleMeterRegistry` | `SimpleMeterRegistry.java:46` | Real supports CUMULATIVE vs STEP modes, histogram gauges |

## 20.10 Complete Code

### Production Code

#### File: `iris-micrometer/src/main/java/com/iris/micrometer/core/Tag.java` [NEW]

```java
package com.iris.micrometer.core;

/**
 * A key/value dimension for a meter. Tags add context to metrics — e.g., "method"="GET", "status"="200".
 *
 * <p>Maps to: {@code io.micrometer.core.instrument.Tag}</p>
 */
public interface Tag extends Comparable<Tag> {

    String getKey();

    String getValue();

    static Tag of(String key, String value) {
        return new ImmutableTag(key, value);
    }

    @Override
    default int compareTo(Tag o) {
        return getKey().compareTo(o.getKey());
    }
}
```

#### File: `iris-micrometer/src/main/java/com/iris/micrometer/core/ImmutableTag.java` [NEW]

```java
package com.iris.micrometer.core;

import java.util.Objects;

/**
 * Immutable implementation of {@link Tag}. Created via {@link Tag#of(String, String)}.
 */
final class ImmutableTag implements Tag {

    private final String key;
    private final String value;

    ImmutableTag(String key, String value) {
        Objects.requireNonNull(key, "tag key must not be null");
        Objects.requireNonNull(value, "tag value must not be null");
        this.key = key;
        this.value = value;
    }

    @Override
    public String getKey() {
        return key;
    }

    @Override
    public String getValue() {
        return value;
    }

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof Tag tag)) return false;
        return key.equals(tag.getKey()) && value.equals(tag.getValue());
    }

    @Override
    public int hashCode() {
        return 31 * key.hashCode() + value.hashCode();
    }

    @Override
    public String toString() {
        return "tag(" + key + "=" + value + ")";
    }
}
```

#### File: `iris-micrometer/src/main/java/com/iris/micrometer/core/Tags.java` [NEW]

```java
package com.iris.micrometer.core;

import java.util.*;
import java.util.stream.Stream;
import java.util.stream.StreamSupport;

/**
 * An immutable, sorted collection of {@link Tag} instances. Duplicate keys are deduplicated
 * (last value wins). Tags are sorted by key for deterministic identity.
 *
 * <p>Maps to: {@code io.micrometer.core.instrument.Tags}</p>
 */
public final class Tags implements Iterable<Tag> {

    private static final Tag[] EMPTY_ARRAY = new Tag[0];
    private static final Tags EMPTY = new Tags(EMPTY_ARRAY);

    private final Tag[] tags;

    private Tags(Tag[] tags) {
        this.tags = tags;
    }

    /**
     * Returns an empty Tags instance (flyweight singleton).
     */
    public static Tags empty() {
        return EMPTY;
    }

    /**
     * Creates Tags from key/value pairs: {@code Tags.of("method", "GET", "status", "200")}.
     */
    public static Tags of(String... keyValues) {
        if (keyValues.length == 0) return EMPTY;
        if (keyValues.length % 2 != 0) {
            throw new IllegalArgumentException("Tags must be key/value pairs, got odd number: " + keyValues.length);
        }
        Tag[] result = new Tag[keyValues.length / 2];
        for (int i = 0; i < keyValues.length; i += 2) {
            result[i / 2] = Tag.of(keyValues[i], keyValues[i + 1]);
        }
        return toTags(result);
    }

    /**
     * Creates Tags from Tag instances.
     */
    public static Tags of(Tag... tags) {
        if (tags.length == 0) return EMPTY;
        return toTags(tags.clone());
    }

    /**
     * Creates Tags from an iterable of Tag instances.
     */
    public static Tags of(Iterable<Tag> tags) {
        if (tags instanceof Tags t) return t;
        Tag[] array = StreamSupport.stream(tags.spliterator(), false).toArray(Tag[]::new);
        if (array.length == 0) return EMPTY;
        return toTags(array);
    }

    /**
     * Returns a new Tags with additional tags merged in. On key conflict, the new tags win.
     */
    public Tags and(String... keyValues) {
        return and(Tags.of(keyValues));
    }

    /**
     * Returns a new Tags with additional tags merged in.
     */
    public Tags and(Tag... moreTags) {
        return and(Tags.of(moreTags));
    }

    /**
     * Merges two sorted Tags in O(n+m). On key conflict, {@code other}'s value wins.
     */
    public Tags and(Tags other) {
        if (this.tags.length == 0) return other;
        if (other.tags.length == 0) return this;

        List<Tag> merged = new ArrayList<>(this.tags.length + other.tags.length);
        int i = 0, j = 0;
        while (i < this.tags.length && j < other.tags.length) {
            int cmp = this.tags[i].getKey().compareTo(other.tags[j].getKey());
            if (cmp < 0) {
                merged.add(this.tags[i++]);
            } else if (cmp > 0) {
                merged.add(other.tags[j++]);
            } else {
                // Same key — other wins
                merged.add(other.tags[j++]);
                i++;
            }
        }
        while (i < this.tags.length) merged.add(this.tags[i++]);
        while (j < other.tags.length) merged.add(other.tags[j++]);
        return new Tags(merged.toArray(EMPTY_ARRAY));
    }

    public Stream<Tag> stream() {
        return Arrays.stream(tags);
    }

    @Override
    public Iterator<Tag> iterator() {
        return List.of(tags).iterator();
    }

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof Tags other)) return false;
        return Arrays.equals(tags, other.tags);
    }

    @Override
    public int hashCode() {
        return Arrays.hashCode(tags);
    }

    @Override
    public String toString() {
        return Arrays.toString(tags);
    }

    /**
     * Sorts and deduplicates the array. On duplicate keys, last value wins.
     */
    private static Tags toTags(Tag[] raw) {
        Arrays.sort(raw);
        // Deduplicate: on same key, keep last (which after sort is the higher-index one)
        int write = 0;
        for (int read = 0; read < raw.length; read++) {
            // If next tag has the same key, skip this one (keep the later one)
            if (read + 1 < raw.length && raw[read].getKey().equals(raw[read + 1].getKey())) {
                continue;
            }
            raw[write++] = raw[read];
        }
        if (write == raw.length) return new Tags(raw);
        return new Tags(Arrays.copyOf(raw, write));
    }
}
```

#### File: `iris-micrometer/src/main/java/com/iris/micrometer/core/Meter.java` [NEW]

```java
package com.iris.micrometer.core;

import java.util.Objects;

/**
 * Base interface for all metric instruments. Every meter has a unique {@link Id}
 * composed of a name and tags.
 *
 * <p>Maps to: {@code io.micrometer.core.instrument.Meter}</p>
 */
public interface Meter {

    /**
     * Returns this meter's unique identity.
     */
    Id getId();

    /**
     * The types of meters supported by the registry.
     */
    enum Type {
        COUNTER,
        GAUGE,
        TIMER,
        OTHER
    }

    /**
     * The unique identity of a meter, consisting of a name and a set of tags.
     *
     * <p>Key design decision from real Micrometer: <b>only name and tags participate
     * in equality</b>. Type, description, and base unit are metadata — not identity.
     * This means you cannot register a Counter and a Timer with the same name+tags.</p>
     *
     * <p>Maps to: {@code io.micrometer.core.instrument.Meter.Id}</p>
     */
    class Id {

        private final String name;
        private final Tags tags;
        private final Type type;
        private final String description;
        private final String baseUnit;

        public Id(String name, Tags tags, Type type, String description, String baseUnit) {
            Objects.requireNonNull(name, "meter name must not be null");
            this.name = name;
            this.tags = (tags != null) ? tags : Tags.empty();
            this.type = type;
            this.description = description;
            this.baseUnit = baseUnit;
        }

        public String getName() {
            return name;
        }

        public Tags getTags() {
            return tags;
        }

        public Type getType() {
            return type;
        }

        public String getDescription() {
            return description;
        }

        public String getBaseUnit() {
            return baseUnit;
        }

        /**
         * Returns a new Id with additional tags merged in.
         */
        public Id withTags(Tags additionalTags) {
            return new Id(name, tags.and(additionalTags), type, description, baseUnit);
        }

        /**
         * Only name and tags define identity — mirrors real Micrometer's design.
         */
        @Override
        public boolean equals(Object o) {
            if (this == o) return true;
            if (!(o instanceof Id id)) return false;
            return name.equals(id.name) && tags.equals(id.tags);
        }

        @Override
        public int hashCode() {
            int result = name.hashCode();
            result = 31 * result + tags.hashCode();
            return result;
        }

        @Override
        public String toString() {
            return "Meter.Id{name='" + name + "', tags=" + tags + "}";
        }
    }
}
```

#### File: `iris-micrometer/src/main/java/com/iris/micrometer/core/Counter.java` [NEW]

```java
package com.iris.micrometer.core;

/**
 * A monotonically increasing counter. Counters track things like request count, errors,
 * or bytes sent — values that only ever go up (or reset to zero on process restart).
 *
 * <p>Maps to: {@code io.micrometer.core.instrument.Counter}</p>
 */
public interface Counter extends Meter {

    /**
     * Increment by one.
     */
    default void increment() {
        increment(1.0);
    }

    /**
     * Increment by the given amount (must be positive).
     */
    void increment(double amount);

    /**
     * Returns the cumulative count since creation.
     */
    double count();

    /**
     * Fluent builder entry point.
     */
    static Builder builder(String name) {
        return new Builder(name);
    }

    /**
     * Fluent builder for Counter registration.
     */
    class Builder {
        private final String name;
        private Tags tags = Tags.empty();
        private String description;
        private String baseUnit;

        private Builder(String name) {
            this.name = name;
        }

        public Builder tags(String... keyValues) {
            this.tags = this.tags.and(keyValues);
            return this;
        }

        public Builder tags(Tags tags) {
            this.tags = this.tags.and(tags);
            return this;
        }

        public Builder tag(String key, String value) {
            this.tags = this.tags.and(Tag.of(key, value));
            return this;
        }

        public Builder description(String description) {
            this.description = description;
            return this;
        }

        public Builder baseUnit(String baseUnit) {
            this.baseUnit = baseUnit;
            return this;
        }

        public Counter register(MeterRegistry registry) {
            return registry.counter(new Meter.Id(name, tags, Meter.Type.COUNTER, description, baseUnit));
        }
    }
}
```

#### File: `iris-micrometer/src/main/java/com/iris/micrometer/core/Timer.java` [NEW]

```java
package com.iris.micrometer.core;

import java.time.Duration;
import java.util.concurrent.Callable;
import java.util.concurrent.TimeUnit;
import java.util.function.Supplier;

/**
 * Tracks a large number of short-running events, recording count, total time, and max.
 * Use for latency measurements (HTTP requests, DB calls, etc.).
 *
 * <p>Maps to: {@code io.micrometer.core.instrument.Timer}</p>
 */
public interface Timer extends Meter {

    /**
     * Record a duration.
     */
    void record(long amount, TimeUnit unit);

    /**
     * Record a Duration.
     */
    default void record(Duration duration) {
        record(duration.toNanos(), TimeUnit.NANOSECONDS);
    }

    /**
     * Time a Runnable and record the duration.
     */
    default void record(Runnable runnable) {
        long start = System.nanoTime();
        try {
            runnable.run();
        } finally {
            record(System.nanoTime() - start, TimeUnit.NANOSECONDS);
        }
    }

    /**
     * Time a Supplier and record the duration.
     */
    default <T> T record(Supplier<T> supplier) {
        long start = System.nanoTime();
        try {
            return supplier.get();
        } finally {
            record(System.nanoTime() - start, TimeUnit.NANOSECONDS);
        }
    }

    /**
     * Time a Callable and record the duration.
     */
    default <T> T recordCallable(Callable<T> callable) throws Exception {
        long start = System.nanoTime();
        try {
            return callable.call();
        } finally {
            record(System.nanoTime() - start, TimeUnit.NANOSECONDS);
        }
    }

    /**
     * Number of recorded events.
     */
    long count();

    /**
     * Total recorded duration in the given time unit.
     */
    double totalTime(TimeUnit unit);

    /**
     * Average duration per event.
     */
    default double mean(TimeUnit unit) {
        long c = count();
        return (c == 0) ? 0.0 : totalTime(unit) / c;
    }

    /**
     * Maximum single-event duration recorded.
     */
    double max(TimeUnit unit);

    /**
     * Start a timing sample. Call {@link Sample#stop(Timer)} to record the elapsed duration.
     * The target Timer is chosen at stop-time, allowing deferred tag decisions.
     */
    static Sample start() {
        return new Sample(System.nanoTime());
    }

    /**
     * A timing sample — captures a start time and records the elapsed duration when stopped.
     *
     * <p>Key design: Timer is chosen at stop-time, not start-time. This allows the caller
     * to decide which tags to apply after the operation completes (e.g., success vs error).</p>
     */
    class Sample {
        private final long startNanos;

        private Sample(long startNanos) {
            this.startNanos = startNanos;
        }

        /**
         * Stop the sample and record the elapsed duration to the given timer.
         * @return elapsed time in nanoseconds
         */
        public long stop(Timer timer) {
            long elapsed = System.nanoTime() - startNanos;
            timer.record(elapsed, TimeUnit.NANOSECONDS);
            return elapsed;
        }
    }

    /**
     * Fluent builder entry point.
     */
    static Builder builder(String name) {
        return new Builder(name);
    }

    /**
     * Fluent builder for Timer registration.
     */
    class Builder {
        private final String name;
        private Tags tags = Tags.empty();
        private String description;

        private Builder(String name) {
            this.name = name;
        }

        public Builder tags(String... keyValues) {
            this.tags = this.tags.and(keyValues);
            return this;
        }

        public Builder tags(Tags tags) {
            this.tags = this.tags.and(tags);
            return this;
        }

        public Builder tag(String key, String value) {
            this.tags = this.tags.and(Tag.of(key, value));
            return this;
        }

        public Builder description(String description) {
            this.description = description;
            return this;
        }

        public Timer register(MeterRegistry registry) {
            return registry.timer(new Meter.Id(name, tags, Meter.Type.TIMER, description, "seconds"));
        }
    }
}
```

#### File: `iris-micrometer/src/main/java/com/iris/micrometer/core/Gauge.java` [NEW]

```java
package com.iris.micrometer.core;

import java.util.function.Supplier;
import java.util.function.ToDoubleFunction;

/**
 * Tracks a value that can go up and down — queue size, memory usage, active threads, etc.
 * Unlike Counter and Timer, Gauge does not accept data pushes. Instead, it holds a reference
 * to a state object and a function that extracts the current value when observed.
 *
 * <p>Maps to: {@code io.micrometer.core.instrument.Gauge}</p>
 */
public interface Gauge extends Meter {

    /**
     * Returns the current value.
     */
    double value();

    /**
     * Builder for gauging an object + function.
     */
    static <T> Builder<T> builder(String name, T obj, ToDoubleFunction<T> valueFunction) {
        return new Builder<>(name, obj, valueFunction);
    }

    /**
     * Builder for gauging a number supplier.
     */
    static Builder<Supplier<Number>> builder(String name, Supplier<Number> supplier) {
        return new Builder<>(name, supplier, s -> {
            Number val = s.get();
            return (val != null) ? val.doubleValue() : Double.NaN;
        });
    }

    /**
     * Fluent builder for Gauge registration.
     */
    class Builder<T> {
        private final String name;
        private final T obj;
        private final ToDoubleFunction<T> valueFunction;
        private Tags tags = Tags.empty();
        private String description;
        private String baseUnit;

        private Builder(String name, T obj, ToDoubleFunction<T> valueFunction) {
            this.name = name;
            this.obj = obj;
            this.valueFunction = valueFunction;
        }

        public Builder<T> tags(String... keyValues) {
            this.tags = this.tags.and(keyValues);
            return this;
        }

        public Builder<T> tags(Tags tags) {
            this.tags = this.tags.and(tags);
            return this;
        }

        public Builder<T> tag(String key, String value) {
            this.tags = this.tags.and(Tag.of(key, value));
            return this;
        }

        public Builder<T> description(String description) {
            this.description = description;
            return this;
        }

        public Builder<T> baseUnit(String baseUnit) {
            this.baseUnit = baseUnit;
            return this;
        }

        public Gauge register(MeterRegistry registry) {
            return registry.gauge(new Meter.Id(name, tags, Meter.Type.GAUGE, description, baseUnit), obj, valueFunction);
        }
    }
}
```

#### File: `iris-micrometer/src/main/java/com/iris/micrometer/core/DefaultCounter.java` [NEW]

```java
package com.iris.micrometer.core;

import java.util.concurrent.atomic.DoubleAdder;

/**
 * Thread-safe cumulative Counter backed by a {@link DoubleAdder}.
 *
 * <p>Maps to: {@code io.micrometer.core.instrument.cumulative.CumulativeCounter}</p>
 */
public class DefaultCounter implements Counter {

    private final Id id;
    private final DoubleAdder value = new DoubleAdder();

    public DefaultCounter(Id id) {
        this.id = id;
    }

    @Override
    public void increment(double amount) {
        if (amount > 0) {
            value.add(amount);
        }
    }

    @Override
    public double count() {
        return value.sum();
    }

    @Override
    public Id getId() {
        return id;
    }
}
```

#### File: `iris-micrometer/src/main/java/com/iris/micrometer/core/DefaultTimer.java` [NEW]

```java
package com.iris.micrometer.core;

import java.util.concurrent.TimeUnit;
import java.util.concurrent.atomic.AtomicLong;
import java.util.concurrent.atomic.DoubleAdder;
import java.util.concurrent.atomic.LongAdder;

/**
 * Thread-safe cumulative Timer tracking count, total time, and max.
 *
 * <p>Maps to: {@code io.micrometer.core.instrument.cumulative.CumulativeTimer}</p>
 */
public class DefaultTimer implements Timer {

    private final Id id;
    private final LongAdder count = new LongAdder();
    private final DoubleAdder totalNanos = new DoubleAdder();
    private final AtomicLong maxNanos = new AtomicLong(0);

    public DefaultTimer(Id id) {
        this.id = id;
    }

    @Override
    public void record(long amount, TimeUnit unit) {
        if (amount >= 0) {
            long nanos = unit.toNanos(amount);
            count.increment();
            totalNanos.add(nanos);
            updateMax(nanos);
        }
    }

    @Override
    public long count() {
        return count.sum();
    }

    @Override
    public double totalTime(TimeUnit unit) {
        return unit.convert((long) totalNanos.sum(), TimeUnit.NANOSECONDS);
    }

    @Override
    public double max(TimeUnit unit) {
        return unit.convert(maxNanos.get(), TimeUnit.NANOSECONDS);
    }

    @Override
    public Id getId() {
        return id;
    }

    private void updateMax(long nanos) {
        long currentMax;
        do {
            currentMax = maxNanos.get();
            if (nanos <= currentMax) return;
        } while (!maxNanos.compareAndSet(currentMax, nanos));
    }
}
```

#### File: `iris-micrometer/src/main/java/com/iris/micrometer/core/DefaultGauge.java` [NEW]

```java
package com.iris.micrometer.core;

import java.util.function.ToDoubleFunction;

/**
 * Default Gauge implementation that holds a reference to an object and a function
 * that extracts the current value.
 *
 * <p>Simplification: real Micrometer uses a WeakReference by default (to avoid memory leaks
 * from gauges keeping objects alive). We use a strong reference for clarity.</p>
 *
 * <p>Maps to: {@code io.micrometer.core.instrument.internal.DefaultGauge}</p>
 */
public class DefaultGauge<T> implements Gauge {

    private final Id id;
    private final T obj;
    private final ToDoubleFunction<T> valueFunction;

    public DefaultGauge(Id id, T obj, ToDoubleFunction<T> valueFunction) {
        this.id = id;
        this.obj = obj;
        this.valueFunction = valueFunction;
    }

    @Override
    public double value() {
        try {
            return valueFunction.applyAsDouble(obj);
        } catch (Exception e) {
            return Double.NaN;
        }
    }

    @Override
    public Id getId() {
        return id;
    }
}
```

#### File: `iris-micrometer/src/main/java/com/iris/micrometer/core/MeterRegistry.java` [NEW]

```java
package com.iris.micrometer.core;

import java.util.*;
import java.util.concurrent.ConcurrentHashMap;
import java.util.function.ToDoubleFunction;

/**
 * Abstract base for meter registries. Stores meters in a {@link ConcurrentHashMap},
 * deduplicates by {@link Meter.Id} (name + tags), and provides fluent convenience methods.
 *
 * <p>This is the core data structure of the metrics system. Future features (Observation API,
 * Actuator MetricsEndpoint) will connect to this registry to read/write metrics.</p>
 *
 * <p>Maps to: {@code io.micrometer.core.instrument.MeterRegistry}</p>
 */
public abstract class MeterRegistry {

    private final Map<Meter.Id, Meter> meterMap = new ConcurrentHashMap<>();
    private final Object meterMapLock = new Object();

    // --- Abstract factory methods (subclasses create concrete meter instances) ---

    /**
     * Create a new Counter for this registry type.
     */
    protected abstract Counter newCounter(Meter.Id id);

    /**
     * Create a new Timer for this registry type.
     */
    protected abstract Timer newTimer(Meter.Id id);

    /**
     * Create a new Gauge for this registry type.
     */
    protected abstract <T> Gauge newGauge(Meter.Id id, T obj, ToDoubleFunction<T> valueFunction);

    /**
     * The base time unit for this registry (e.g., SECONDS for SimpleMeterRegistry).
     */
    protected abstract java.util.concurrent.TimeUnit getBaseTimeUnit();

    // --- Registration with deduplication ---

    /**
     * Get or create a Counter with the given Id. If a meter with the same name+tags already
     * exists, it must be a Counter — otherwise an IllegalArgumentException is thrown.
     */
    public Counter counter(Meter.Id id) {
        return getOrCreateMeter(id, Counter.class, this::newCounter);
    }

    /**
     * Convenience: get or create a Counter by name and tags.
     */
    public Counter counter(String name, String... tags) {
        return counter(new Meter.Id(name, Tags.of(tags), Meter.Type.COUNTER, null, null));
    }

    /**
     * Get or create a Timer with the given Id.
     */
    public Timer timer(Meter.Id id) {
        return getOrCreateMeter(id, Timer.class, this::newTimer);
    }

    /**
     * Convenience: get or create a Timer by name and tags.
     */
    public Timer timer(String name, String... tags) {
        return timer(new Meter.Id(name, Tags.of(tags), Meter.Type.TIMER, null, "seconds"));
    }

    /**
     * Register a Gauge with the given Id, object, and value function.
     */
    public <T> Gauge gauge(Meter.Id id, T obj, ToDoubleFunction<T> valueFunction) {
        Meter existing = meterMap.get(id);
        if (existing != null) {
            if (existing instanceof Gauge g) return g;
            throw meterTypeMismatch(id, Gauge.class, existing);
        }
        synchronized (meterMapLock) {
            existing = meterMap.get(id);
            if (existing != null) {
                if (existing instanceof Gauge g) return g;
                throw meterTypeMismatch(id, Gauge.class, existing);
            }
            Gauge gauge = newGauge(id, obj, valueFunction);
            meterMap.put(id, gauge);
            return gauge;
        }
    }

    /**
     * Convenience: register a Gauge by name, tags, object, and function.
     */
    public <T> Gauge gauge(String name, Tags tags, T obj, ToDoubleFunction<T> valueFunction) {
        return gauge(new Meter.Id(name, tags, Meter.Type.GAUGE, null, null), obj, valueFunction);
    }

    /**
     * Convenience: register a Gauge tracking a Number supplier.
     */
    public <T extends Number> Gauge gauge(String name, Tags tags, T number) {
        return gauge(name, tags, number, Number::doubleValue);
    }

    // --- Query ---

    /**
     * Returns all registered meters (unmodifiable view).
     */
    public Collection<Meter> getMeters() {
        return Collections.unmodifiableCollection(meterMap.values());
    }

    /**
     * Find a meter by its Id.
     */
    public Meter find(Meter.Id id) {
        return meterMap.get(id);
    }

    /**
     * Remove a meter by its Id.
     */
    public Meter remove(Meter.Id id) {
        synchronized (meterMapLock) {
            return meterMap.remove(id);
        }
    }

    /**
     * Remove all meters.
     */
    public void clear() {
        synchronized (meterMapLock) {
            meterMap.clear();
        }
    }

    // --- Internal helpers ---

    /**
     * Core deduplication logic: check ConcurrentHashMap first (no lock), then double-check
     * under lock before creating. This mirrors real Micrometer's {@code getOrCreateMeter()}.
     */
    @SuppressWarnings("unchecked")
    private <M extends Meter> M getOrCreateMeter(Meter.Id id, Class<M> meterClass,
                                                  java.util.function.Function<Meter.Id, M> factory) {
        // Fast path — no lock
        Meter existing = meterMap.get(id);
        if (existing != null) {
            if (meterClass.isInstance(existing)) return (M) existing;
            throw meterTypeMismatch(id, meterClass, existing);
        }

        // Slow path — synchronized
        synchronized (meterMapLock) {
            existing = meterMap.get(id);
            if (existing != null) {
                if (meterClass.isInstance(existing)) return (M) existing;
                throw meterTypeMismatch(id, meterClass, existing);
            }
            M meter = factory.apply(id);
            meterMap.put(id, meter);
            return meter;
        }
    }

    private IllegalArgumentException meterTypeMismatch(Meter.Id id, Class<?> expected, Meter actual) {
        return new IllegalArgumentException(
                "There is already a meter registered with name '" + id.getName()
                        + "' and tags " + id.getTags() + " but it is a "
                        + actual.getClass().getSimpleName() + ", not a "
                        + expected.getSimpleName());
    }
}
```

#### File: `iris-micrometer/src/main/java/com/iris/micrometer/core/simple/SimpleMeterRegistry.java` [NEW]

```java
package com.iris.micrometer.core.simple;

import com.iris.micrometer.core.*;

import java.util.concurrent.TimeUnit;
import java.util.function.ToDoubleFunction;

/**
 * In-memory meter registry for testing and simple use cases.
 * All meters live in memory with no external publishing.
 *
 * <p>Maps to: {@code io.micrometer.core.instrument.simple.SimpleMeterRegistry}</p>
 */
public class SimpleMeterRegistry extends MeterRegistry {

    @Override
    protected Counter newCounter(Meter.Id id) {
        return new DefaultCounter(id);
    }

    @Override
    protected Timer newTimer(Meter.Id id) {
        return new DefaultTimer(id);
    }

    @Override
    protected <T> Gauge newGauge(Meter.Id id, T obj, ToDoubleFunction<T> valueFunction) {
        return new DefaultGauge<>(id, obj, valueFunction);
    }

    @Override
    protected TimeUnit getBaseTimeUnit() {
        return TimeUnit.SECONDS;
    }
}
```

### Test Code

#### File: `iris-micrometer/src/test/java/com/iris/micrometer/core/TagsTest.java` [NEW]

```java
package com.iris.micrometer.core;

import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.*;

class TagsTest {

    @Test
    void shouldCreateEmptyTags() {
        Tags tags = Tags.empty();
        assertThat(tags).isEmpty();
    }

    @Test
    void shouldCreateTagsFromKeyValuePairs() {
        Tags tags = Tags.of("method", "GET", "status", "200");
        assertThat(tags).containsExactly(
                Tag.of("method", "GET"),
                Tag.of("status", "200")
        );
    }

    @Test
    void shouldSortTagsByKey() {
        Tags tags = Tags.of("status", "200", "method", "GET");
        assertThat(tags).containsExactly(
                Tag.of("method", "GET"),
                Tag.of("status", "200")
        );
    }

    @Test
    void shouldDeduplicateByKey_LastValueWins() {
        Tags tags = Tags.of("method", "GET", "method", "POST");
        assertThat(tags).containsExactly(Tag.of("method", "POST"));
    }

    @Test
    void shouldMergeTags_OtherWinsOnConflict() {
        Tags base = Tags.of("method", "GET", "uri", "/api");
        Tags extra = Tags.of("method", "POST", "status", "200");

        Tags merged = base.and(extra);

        assertThat(merged).containsExactly(
                Tag.of("method", "POST"),   // extra wins
                Tag.of("status", "200"),
                Tag.of("uri", "/api")
        );
    }

    @Test
    void shouldMergeWithEmptyTags() {
        Tags base = Tags.of("method", "GET");
        assertThat(base.and(Tags.empty())).isEqualTo(base);
        assertThat(Tags.empty().and(base)).isEqualTo(base);
    }

    @Test
    void shouldBeEqualForSameContent() {
        Tags a = Tags.of("method", "GET", "status", "200");
        Tags b = Tags.of("status", "200", "method", "GET");
        assertThat(a).isEqualTo(b);
        assertThat(a.hashCode()).isEqualTo(b.hashCode());
    }

    @Test
    void shouldThrowOnOddNumberOfKeyValues() {
        assertThatIllegalArgumentException()
                .isThrownBy(() -> Tags.of("method"))
                .withMessageContaining("odd number");
    }

    @Test
    void shouldCreateFromTagInstances() {
        Tags tags = Tags.of(Tag.of("method", "GET"), Tag.of("status", "200"));
        assertThat(tags).containsExactly(
                Tag.of("method", "GET"),
                Tag.of("status", "200")
        );
    }

    @Test
    void shouldCreateFromIterable() {
        var list = java.util.List.of(Tag.of("b", "2"), Tag.of("a", "1"));
        Tags tags = Tags.of(list);
        assertThat(tags).containsExactly(Tag.of("a", "1"), Tag.of("b", "2"));
    }

    @Test
    void shouldAndWithKeyValuePairs() {
        Tags tags = Tags.of("a", "1").and("b", "2");
        assertThat(tags).containsExactly(Tag.of("a", "1"), Tag.of("b", "2"));
    }
}
```

#### File: `iris-micrometer/src/test/java/com/iris/micrometer/core/MeterIdTest.java` [NEW]

```java
package com.iris.micrometer.core;

import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.*;

class MeterIdTest {

    @Test
    void shouldBeEqualByNameAndTagsOnly() {
        Meter.Id a = new Meter.Id("http.requests", Tags.of("method", "GET"),
                Meter.Type.COUNTER, "desc1", "unit1");
        Meter.Id b = new Meter.Id("http.requests", Tags.of("method", "GET"),
                Meter.Type.TIMER, "desc2", "unit2");

        assertThat(a).isEqualTo(b);
        assertThat(a.hashCode()).isEqualTo(b.hashCode());
    }

    @Test
    void shouldNotBeEqualWithDifferentNames() {
        Meter.Id a = new Meter.Id("http.requests", Tags.empty(), Meter.Type.COUNTER, null, null);
        Meter.Id b = new Meter.Id("http.errors", Tags.empty(), Meter.Type.COUNTER, null, null);

        assertThat(a).isNotEqualTo(b);
    }

    @Test
    void shouldNotBeEqualWithDifferentTags() {
        Meter.Id a = new Meter.Id("http.requests", Tags.of("method", "GET"), Meter.Type.COUNTER, null, null);
        Meter.Id b = new Meter.Id("http.requests", Tags.of("method", "POST"), Meter.Type.COUNTER, null, null);

        assertThat(a).isNotEqualTo(b);
    }

    @Test
    void shouldMergeAdditionalTagsViaWithTags() {
        Meter.Id original = new Meter.Id("http.requests", Tags.of("method", "GET"),
                Meter.Type.COUNTER, null, null);
        Meter.Id extended = original.withTags(Tags.of("status", "200"));

        assertThat(extended.getTags()).containsExactly(
                Tag.of("method", "GET"),
                Tag.of("status", "200")
        );
        // Original unchanged (immutable)
        assertThat(original.getTags()).containsExactly(Tag.of("method", "GET"));
    }

    @Test
    void shouldDefaultToEmptyTags_WhenNullProvided() {
        Meter.Id id = new Meter.Id("test", null, Meter.Type.COUNTER, null, null);
        assertThat(id.getTags()).isEqualTo(Tags.empty());
    }
}
```

#### File: `iris-micrometer/src/test/java/com/iris/micrometer/core/CounterTest.java` [NEW]

```java
package com.iris.micrometer.core;

import com.iris.micrometer.core.simple.SimpleMeterRegistry;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.*;

class CounterTest {

    private MeterRegistry registry;

    @BeforeEach
    void setUp() {
        registry = new SimpleMeterRegistry();
    }

    @Test
    void shouldStartAtZero() {
        Counter counter = registry.counter("test.counter");
        assertThat(counter.count()).isEqualTo(0.0);
    }

    @Test
    void shouldIncrementByOne() {
        Counter counter = registry.counter("test.counter");
        counter.increment();
        assertThat(counter.count()).isEqualTo(1.0);
    }

    @Test
    void shouldIncrementByAmount() {
        Counter counter = registry.counter("test.counter");
        counter.increment(5.0);
        assertThat(counter.count()).isEqualTo(5.0);
    }

    @Test
    void shouldAccumulateMultipleIncrements() {
        Counter counter = registry.counter("test.counter");
        counter.increment(3.0);
        counter.increment(7.0);
        counter.increment();
        assertThat(counter.count()).isEqualTo(11.0);
    }

    @Test
    void shouldIgnoreNegativeIncrements() {
        Counter counter = registry.counter("test.counter");
        counter.increment(5.0);
        counter.increment(-3.0);
        assertThat(counter.count()).isEqualTo(5.0);
    }

    @Test
    void shouldCreateViaFluentBuilder() {
        Counter counter = Counter.builder("http.requests")
                .tag("method", "GET")
                .tag("status", "200")
                .description("Total HTTP requests")
                .register(registry);

        counter.increment();
        assertThat(counter.count()).isEqualTo(1.0);
        assertThat(counter.getId().getName()).isEqualTo("http.requests");
        assertThat(counter.getId().getTags()).containsExactly(
                Tag.of("method", "GET"),
                Tag.of("status", "200")
        );
    }

    @Test
    void shouldCreateDistinctCountersForDifferentTags() {
        Counter get = registry.counter("http.requests", "method", "GET");
        Counter post = registry.counter("http.requests", "method", "POST");

        get.increment(10);
        post.increment(3);

        assertThat(get.count()).isEqualTo(10.0);
        assertThat(post.count()).isEqualTo(3.0);
    }

    @Test
    void shouldReturnSameInstance_WhenSameNameAndTags() {
        Counter c1 = registry.counter("test", "k", "v");
        Counter c2 = registry.counter("test", "k", "v");
        assertThat(c1).isSameAs(c2);
    }
}
```

#### File: `iris-micrometer/src/test/java/com/iris/micrometer/core/TimerTest.java` [NEW]

```java
package com.iris.micrometer.core;

import com.iris.micrometer.core.simple.SimpleMeterRegistry;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.time.Duration;
import java.util.concurrent.TimeUnit;

import static org.assertj.core.api.Assertions.*;

class TimerTest {

    private MeterRegistry registry;

    @BeforeEach
    void setUp() {
        registry = new SimpleMeterRegistry();
    }

    @Test
    void shouldStartWithZeroCounts() {
        Timer timer = registry.timer("test.timer");
        assertThat(timer.count()).isZero();
        assertThat(timer.totalTime(TimeUnit.NANOSECONDS)).isZero();
        assertThat(timer.max(TimeUnit.NANOSECONDS)).isZero();
    }

    @Test
    void shouldRecordDuration() {
        Timer timer = registry.timer("test.timer");
        timer.record(100, TimeUnit.MILLISECONDS);

        assertThat(timer.count()).isEqualTo(1);
        assertThat(timer.totalTime(TimeUnit.MILLISECONDS)).isEqualTo(100.0);
    }

    @Test
    void shouldRecordDurationObject() {
        Timer timer = registry.timer("test.timer");
        timer.record(Duration.ofMillis(250));

        assertThat(timer.count()).isEqualTo(1);
        assertThat(timer.totalTime(TimeUnit.MILLISECONDS)).isGreaterThanOrEqualTo(250.0);
    }

    @Test
    void shouldAccumulateMultipleRecordings() {
        Timer timer = registry.timer("test.timer");
        timer.record(100, TimeUnit.MILLISECONDS);
        timer.record(200, TimeUnit.MILLISECONDS);
        timer.record(150, TimeUnit.MILLISECONDS);

        assertThat(timer.count()).isEqualTo(3);
        assertThat(timer.totalTime(TimeUnit.MILLISECONDS)).isEqualTo(450.0);
        assertThat(timer.max(TimeUnit.MILLISECONDS)).isEqualTo(200.0);
    }

    @Test
    void shouldTrackMaxDuration() {
        Timer timer = registry.timer("test.timer");
        timer.record(50, TimeUnit.MILLISECONDS);
        timer.record(200, TimeUnit.MILLISECONDS);
        timer.record(100, TimeUnit.MILLISECONDS);

        assertThat(timer.max(TimeUnit.MILLISECONDS)).isEqualTo(200.0);
    }

    @Test
    void shouldCalculateMean() {
        Timer timer = registry.timer("test.timer");
        timer.record(100, TimeUnit.MILLISECONDS);
        timer.record(200, TimeUnit.MILLISECONDS);

        assertThat(timer.mean(TimeUnit.MILLISECONDS)).isEqualTo(150.0);
    }

    @Test
    void shouldReturnZeroMean_WhenNoRecordings() {
        Timer timer = registry.timer("test.timer");
        assertThat(timer.mean(TimeUnit.MILLISECONDS)).isEqualTo(0.0);
    }

    @Test
    void shouldTimeRunnable() {
        Timer timer = registry.timer("test.timer");
        timer.record(() -> {
            // Simulate work
            try { Thread.sleep(10); } catch (InterruptedException ignored) {}
        });

        assertThat(timer.count()).isEqualTo(1);
        assertThat(timer.totalTime(TimeUnit.MILLISECONDS)).isGreaterThan(0);
    }

    @Test
    void shouldTimeSupplierAndReturnResult() {
        Timer timer = registry.timer("test.timer");
        String result = timer.record(() -> "hello");

        assertThat(result).isEqualTo("hello");
        assertThat(timer.count()).isEqualTo(1);
    }

    @Test
    void shouldTimeCallableAndReturnResult() throws Exception {
        Timer timer = registry.timer("test.timer");
        Integer result = timer.recordCallable(() -> 42);

        assertThat(result).isEqualTo(42);
        assertThat(timer.count()).isEqualTo(1);
    }

    @Test
    void shouldUseSampleForDeferredTimerChoice() {
        Timer.Sample sample = Timer.start();

        // Simulate work
        try { Thread.sleep(10); } catch (InterruptedException ignored) {}

        // Choose timer at stop-time (e.g., based on operation outcome)
        Timer successTimer = registry.timer("request", "outcome", "success");
        long elapsed = sample.stop(successTimer);

        assertThat(elapsed).isGreaterThan(0);
        assertThat(successTimer.count()).isEqualTo(1);
        assertThat(successTimer.totalTime(TimeUnit.NANOSECONDS)).isGreaterThan(0);
    }

    @Test
    void shouldCreateViaFluentBuilder() {
        Timer timer = Timer.builder("http.server.requests")
                .tag("method", "GET")
                .tag("uri", "/api/users")
                .description("HTTP server request duration")
                .register(registry);

        timer.record(100, TimeUnit.MILLISECONDS);
        assertThat(timer.count()).isEqualTo(1);
        assertThat(timer.getId().getName()).isEqualTo("http.server.requests");
    }

    @Test
    void shouldReturnSameInstance_WhenSameNameAndTags() {
        Timer t1 = registry.timer("test", "k", "v");
        Timer t2 = registry.timer("test", "k", "v");
        assertThat(t1).isSameAs(t2);
    }
}
```

#### File: `iris-micrometer/src/test/java/com/iris/micrometer/core/GaugeTest.java` [NEW]

```java
package com.iris.micrometer.core;

import com.iris.micrometer.core.simple.SimpleMeterRegistry;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.util.ArrayList;
import java.util.List;
import java.util.concurrent.atomic.AtomicInteger;

import static org.assertj.core.api.Assertions.*;

class GaugeTest {

    private MeterRegistry registry;

    @BeforeEach
    void setUp() {
        registry = new SimpleMeterRegistry();
    }

    @Test
    void shouldTrackCollectionSize() {
        List<String> queue = new ArrayList<>();
        Gauge gauge = Gauge.builder("queue.size", queue, List::size)
                .register(registry);

        assertThat(gauge.value()).isEqualTo(0.0);

        queue.add("a");
        queue.add("b");
        assertThat(gauge.value()).isEqualTo(2.0);

        queue.clear();
        assertThat(gauge.value()).isEqualTo(0.0);
    }

    @Test
    void shouldTrackAtomicValue() {
        AtomicInteger active = new AtomicInteger(0);
        Gauge gauge = Gauge.builder("active.connections", active, AtomicInteger::get)
                .tag("pool", "default")
                .register(registry);

        active.set(5);
        assertThat(gauge.value()).isEqualTo(5.0);

        active.set(3);
        assertThat(gauge.value()).isEqualTo(3.0);
    }

    @Test
    void shouldTrackNumberSupplier() {
        AtomicInteger value = new AtomicInteger(42);
        Gauge gauge = Gauge.builder("dynamic.value", value::get)
                .register(registry);

        assertThat(gauge.value()).isEqualTo(42.0);

        value.set(99);
        assertThat(gauge.value()).isEqualTo(99.0);
    }

    @Test
    void shouldReturnNaN_WhenFunctionThrows() {
        Gauge gauge = Gauge.builder("broken", null, obj -> {
            throw new RuntimeException("boom");
        }).register(registry);

        assertThat(gauge.value()).isNaN();
    }

    @Test
    void shouldCreateWithDescriptionAndUnit() {
        AtomicInteger heapUsed = new AtomicInteger(1024);
        Gauge gauge = Gauge.builder("jvm.memory.used", heapUsed, AtomicInteger::get)
                .description("Current memory usage")
                .baseUnit("bytes")
                .register(registry);

        assertThat(gauge.getId().getName()).isEqualTo("jvm.memory.used");
        assertThat(gauge.getId().getDescription()).isEqualTo("Current memory usage");
        assertThat(gauge.getId().getBaseUnit()).isEqualTo("bytes");
        assertThat(gauge.value()).isEqualTo(1024.0);
    }

    @Test
    void shouldReturnSameInstance_WhenSameNameAndTags() {
        AtomicInteger v = new AtomicInteger(1);
        Gauge g1 = registry.gauge("test", Tags.of("k", "v"), v, AtomicInteger::doubleValue);
        Gauge g2 = registry.gauge("test", Tags.of("k", "v"), v, AtomicInteger::doubleValue);
        assertThat(g1).isSameAs(g2);
    }

    @Test
    void shouldTrackNumberDirectly() {
        AtomicInteger value = new AtomicInteger(42);
        Gauge gauge = registry.gauge("simple", Tags.empty(), value);

        assertThat(gauge.value()).isEqualTo(42.0);
        value.set(10);
        assertThat(gauge.value()).isEqualTo(10.0);
    }
}
```

#### File: `iris-micrometer/src/test/java/com/iris/micrometer/core/MeterRegistryTest.java` [NEW]

```java
package com.iris.micrometer.core;

import com.iris.micrometer.core.simple.SimpleMeterRegistry;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.util.concurrent.atomic.AtomicInteger;

import static org.assertj.core.api.Assertions.*;

class MeterRegistryTest {

    private MeterRegistry registry;

    @BeforeEach
    void setUp() {
        registry = new SimpleMeterRegistry();
    }

    @Test
    void shouldDeduplicateMeters_WhenSameNameAndTags() {
        Counter c1 = registry.counter("test", "method", "GET");
        Counter c2 = registry.counter("test", "method", "GET");

        assertThat(c1).isSameAs(c2);
    }

    @Test
    void shouldCreateDistinctMeters_WhenDifferentTags() {
        Counter get = registry.counter("requests", "method", "GET");
        Counter post = registry.counter("requests", "method", "POST");

        assertThat(get).isNotSameAs(post);
        get.increment(5);
        assertThat(post.count()).isEqualTo(0.0);
    }

    @Test
    void shouldThrowOnTypeMismatch_WhenSameNameAndTags() {
        registry.counter("test.metric");

        assertThatIllegalArgumentException()
                .isThrownBy(() -> registry.timer("test.metric"))
                .withMessageContaining("test.metric");
    }

    @Test
    void shouldListAllRegisteredMeters() {
        registry.counter("counter1");
        registry.timer("timer1");

        assertThat(registry.getMeters()).hasSize(2);
    }

    @Test
    void shouldFindMeterById() {
        Counter counter = registry.counter("find.me", "tag", "value");

        Meter found = registry.find(counter.getId());
        assertThat(found).isSameAs(counter);
    }

    @Test
    void shouldReturnNull_WhenMeterNotFound() {
        Meter found = registry.find(new Meter.Id("nonexistent", Tags.empty(),
                Meter.Type.COUNTER, null, null));
        assertThat(found).isNull();
    }

    @Test
    void shouldRemoveMeterById() {
        Counter counter = registry.counter("remove.me");
        Meter.Id id = counter.getId();

        Meter removed = registry.remove(id);
        assertThat(removed).isSameAs(counter);
        assertThat(registry.find(id)).isNull();
        assertThat(registry.getMeters()).isEmpty();
    }

    @Test
    void shouldClearAllMeters() {
        registry.counter("c1");
        registry.counter("c2");
        registry.timer("t1");

        registry.clear();
        assertThat(registry.getMeters()).isEmpty();
    }

    @Test
    void shouldSupportMultipleMeterTypes() {
        Counter counter = registry.counter("test.counter");
        Timer timer = registry.timer("test.timer");
        AtomicInteger val = new AtomicInteger(42);
        Gauge gauge = registry.gauge("test.gauge", Tags.empty(), val, AtomicInteger::doubleValue);

        counter.increment(5);
        timer.record(100, java.util.concurrent.TimeUnit.MILLISECONDS);
        val.set(99);

        assertThat(counter.count()).isEqualTo(5.0);
        assertThat(timer.count()).isEqualTo(1);
        assertThat(gauge.value()).isEqualTo(99.0);
        assertThat(registry.getMeters()).hasSize(3);
    }
}
```

---

## Summary

| Concept | What It Means |
|---------|---------------|
| **Meter.Id** | The unique identity of a metric: name + sorted tags. Type, description, and unit are metadata — not part of identity. |
| **MeterRegistry** | Abstract base that stores meters in a `ConcurrentHashMap`, deduplicates by Id, and delegates creation to subclasses via abstract factory methods. |
| **Counter** | Monotonically increasing value (e.g., request count). Push-based: call `increment()`. |
| **Timer** | Tracks short-duration events with count, total time, and max. Push-based: call `record()`. |
| **Gauge** | Instantaneous value that goes up and down (e.g., queue size). Pull-based: holds a reference + function, evaluated when observed. |
| **Timer.Sample** | Captures start time; timer chosen at stop-time for deferred tag decisions (success vs error). |
| **Tags** | Immutable, sorted, deduplicated collection of key/value dimensions. Enables multi-dimensional metrics. |
| **Abstract Factory** | `MeterRegistry` defines the skeleton; `SimpleMeterRegistry` chooses concrete implementations. Different backends (Prometheus, StatsD) would use different implementations. |
| **Double-Checked Locking** | Fast lock-free reads via `ConcurrentHashMap.get()`, synchronized writes only when creating new meters. |

**Next: Chapter 21 — Observation API** — The unified observability abstraction that instruments once and automatically produces metrics (and potentially traces) via pluggable handlers. It bridges observations to this `MeterRegistry`, connecting the "instrument" side with the "record" side.
