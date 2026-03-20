# Chapter 3: Counter

> Line references based on commit `2c8a4606c` of the Micrometer repository.

## Build Challenge

| Current State | Limitation | Objective |
|---------------|------------|-----------|
| `Counter` is a stub interface extending `Meter` with no methods of its own | Cannot count anything — no `increment()` or `count()` API, no concrete implementation | Build a full `Counter` interface with `increment()`/`count()`, a fluent `Builder`, and a lock-free `CumulativeCounter` implementation |

---

## 3.1 The Integration Point

Counter is the first meter type to gain real behavior. The integration point is the `measure()` default method on the `Counter` interface — it bridges the Counter-specific `count()` API into the generic `Meter.measure()` contract that all backends will consume.

**Modifying:** `src/main/java/dev/linhvu/micrometer/Counter.java`
**Change:** Replace stub interface with full Counter contract including `increment()`, `count()`, and `measure()`

```java
public interface Counter extends Meter {

    default void increment() {
        increment(1.0);
    }

    void increment(double amount);

    double count();

    @Override
    default Iterable<Measurement> measure() {
        return Collections.singletonList(new Measurement(this::count, Statistic.COUNT));
    }
}
```

Two key decisions here:

1. **`measure()` as a default method on the interface, not on the implementation.** Every Counter — whether cumulative, step-based, or composite — produces the same measurement shape: a single `COUNT` statistic. Defining it once on the interface means implementations only need to provide `count()`.
2. **`this::count` as a lazy supplier.** The `Measurement` captures a method reference, not a snapshot. When a backend calls `measurement.getValue()`, it gets the live count at that moment — not a stale value from registration time.

This connects **Counter's domain API** (`increment`/`count`) to **Meter's generic measurement pipeline** (`measure()`). To make it work, we need to build:
- A `Builder` inner class — the fluent API for constructing counter metadata (name, tags, description)
- `CumulativeCounter` — the concrete implementation using `DoubleAdder` for lock-free concurrency

## 3.2 The Builder

The Builder accumulates counter metadata. In the real Micrometer, the terminal operation is `register(MeterRegistry)` — the registry creates the actual Counter instance. Since MeterRegistry is Feature 4, we build the accumulation side now.

```java
class Builder {

    private final String name;
    private Tags tags = Tags.empty();
    private String description;
    private String baseUnit;

    private Builder(String name) {
        this.name = name;
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

    public Builder baseUnit(String unit) {
        this.baseUnit = unit;
        return this;
    }
}
```

Usage pattern (metadata accumulation — registration comes in ch04):

```java
Counter.Builder builder = Counter.builder("http.requests")
    .tag("method", "GET")
    .tags("status", "200", "uri", "/api")
    .description("Total HTTP requests")
    .baseUnit("requests");
```

## 3.3 CumulativeCounter

The concrete implementation lives in a `cumulative` subpackage — the same convention Micrometer uses to separate cumulative meters from step-based meters.

**New file:** `src/main/java/dev/linhvu/micrometer/cumulative/CumulativeCounter.java`

```java
package dev.linhvu.micrometer.cumulative;

import dev.linhvu.micrometer.AbstractMeter;
import dev.linhvu.micrometer.Counter;
import dev.linhvu.micrometer.Meter;

import java.util.concurrent.atomic.DoubleAdder;

public class CumulativeCounter extends AbstractMeter implements Counter {

    private final DoubleAdder value;

    public CumulativeCounter(Meter.Id id) {
        super(id);
        this.value = new DoubleAdder();
    }

    @Override
    public void increment(double amount) {
        value.add(amount);
    }

    @Override
    public double count() {
        return value.sum();
    }
}
```

The entire class is 10 lines of logic. `DoubleAdder` does the heavy lifting — it maintains per-thread accumulation cells internally and sums them lazily on `sum()`. This is why `CumulativeCounter` needs no locks, no `synchronized`, no CAS loops.

## 3.4 Try It Yourself

<details>
<summary>Challenge 1: Why does measure() use this::count instead of () -> count()?</summary>

They are functionally equivalent, but `this::count` is a method reference that makes the binding explicit — each call to `measurement.getValue()` invokes `count()` on this specific Counter instance. Try it: create a counter, capture `measure()`, increment the counter, and verify the measurement reflects the new value.

```java
Counter counter = new CumulativeCounter(id);
Iterable<Measurement> measurements = counter.measure();

counter.increment(10.0);

// The measurement was captured BEFORE the increment,
// but evaluates lazily via this::count
Measurement m = measurements.iterator().next();
assertThat(m.getValue()).isEqualTo(10.0);  // live value, not stale
```

</details>

<details>
<summary>Challenge 2: Implement CumulativeCounter yourself</summary>

Starting point — you need a class that:
1. Extends `AbstractMeter` (for identity/equality)
2. Implements `Counter` (for `increment`/`count`/`measure`)
3. Uses `DoubleAdder` for thread-safe accumulation

```java
public class CumulativeCounter extends AbstractMeter implements Counter {

    private final DoubleAdder value;

    public CumulativeCounter(Meter.Id id) {
        super(id);
        this.value = new DoubleAdder();
    }

    @Override
    public void increment(double amount) {
        value.add(amount);
    }

    @Override
    public double count() {
        return value.sum();
    }
}
```

</details>

## 3.5 Tests

### Unit Tests

**New file:** `src/test/java/dev/linhvu/micrometer/CounterTest.java`

```java
class CounterTest {

    @Test
    void shouldStartAtZero_WhenCreated() { ... }

    @Test
    void shouldIncrementByOne_WhenIncrementCalledWithNoArgs() { ... }

    @Test
    void shouldIncrementByAmount_WhenIncrementCalledWithValue() { ... }

    @Test
    void shouldAccumulateIncrements_WhenCalledMultipleTimes() { ... }

    @Test
    void shouldReturnSingleCountMeasurement_WhenMeasureCalled() { ... }

    @Test
    void shouldReflectLiveCount_WhenMeasurementEvaluatedLazily() { ... }

    @Test
    void shouldBuildWithName_WhenUsingBuilder() { ... }

    @Test
    void shouldAccumulateTags_WhenUsingBuilderTagMethods() { ... }

    @Test
    void shouldMatchAsCounter_WhenUsingVisitorPattern() { ... }
}
```

**New file:** `src/test/java/dev/linhvu/micrometer/cumulative/CumulativeCounterTest.java`

```java
class CumulativeCounterTest {

    @Test
    void shouldStartAtZero_WhenCreated() { ... }

    @Test
    void shouldAccumulate_WhenIncrementedMultipleTimes() { ... }

    @Test
    void shouldNeverDecrease_WhenOnlyPositiveAmountsAdded() { ... }

    @Test
    void shouldHandleConcurrentIncrements_WhenAccessedFromMultipleThreads() { ... }
}
```

The concurrency test verifies `DoubleAdder`'s correctness: 10 threads each increment 1,000 times, and the final count must be exactly 10,000. No lost updates.

**Modified file:** `src/test/java/dev/linhvu/micrometer/MeterVisitorTest.java`
**Change:** `TestCounter` now implements `increment(double)` and `count()` since Counter gained abstract methods.

**Run:** `./gradlew test` — expected: all 52 tests pass (including prior features' tests)

---

## 3.6 Why This Works

> ★ **Insight** -------------------------------------------
> - **Why `DoubleAdder` over `AtomicLong`?** `AtomicLong` uses a single CAS (compare-and-swap) variable — under high contention, threads spin-wait retrying CAS operations. `DoubleAdder` spreads updates across multiple cells, dramatically reducing contention. The trade-off: `sum()` is slightly more expensive (iterates all cells), but counters are write-heavy/read-light (incremented on every request, read only at scrape time).
> - **Why `double` instead of `long`?** Fractional increments matter. A counter tracking "bytes received" may need to add 1.5KB. A counter tracking "CPU seconds consumed" may add 0.003. The double type handles both without forcing callers to choose a unit multiplier.
> - **When NOT to use Counter:** If the value can go down (active connections, queue size), use a Gauge instead. Counters are monotonically increasing — their derivative (rate) is always non-negative. Monitoring systems rely on this invariant for rate calculations.
> -----------------------------------------------------------

> ★ **Insight** -------------------------------------------
> - **The Builder → Register pattern.** Every meter type in Micrometer follows the same lifecycle: `Builder` accumulates metadata → `register(registry)` creates the concrete meter. This separates *what to measure* (user's concern) from *how to store it* (registry's concern). A `SimpleMeterRegistry` creates `CumulativeCounter`; a `StepMeterRegistry` creates `StepCounter`; a `CompositeMeterRegistry` creates `CompositeCounter`. Same builder, different concrete types — classic Factory Method.
> - **Why the builder's constructor is private:** `Counter.builder("name")` is the only entry point. This prevents subclassing the builder and ensures every counter starts with a name — the one required field.
> -----------------------------------------------------------

## 3.7 What We Enhanced

| Aspect | Before (ch02) | Current (ch03) | Real Framework |
|--------|---------------|----------------|----------------|
| Counter interface | Empty stub — extends Meter with no methods | Full contract: `increment()`, `count()`, `measure()`, plus fluent `Builder` | Same contract plus `register(MeterRegistry)` on Builder (`Counter.java:143`) |
| Concrete implementation | None — no way to actually count anything | `CumulativeCounter` with `DoubleAdder` for lock-free concurrency | Same `CumulativeCounter` (`CumulativeCounter.java:23`) — our code is identical |
| Measurement bridge | Each meter type had to manually implement `measure()` | Counter provides a default `measure()` returning `Statistic.COUNT` | Same default method pattern (`Counter.java:54`) |

## 3.8 Connection to Real Framework

| Simplified Code | Real Framework Code | File:Line | Key Difference |
|----------------|--------------------|-----------| ---------------|
| `Counter` interface | `Counter` | `Counter.java:30` | Real adds `@Nullable` annotations via jspecify |
| `Counter.Builder` | `Counter.Builder` | `Counter.java:61` | Real has `register(MeterRegistry)` and `withRegistry()` for `MeterProvider` |
| `Counter.builder("name")` | `Counter.builder("name")` | `Counter.java:32` | Identical API |
| `increment()` / `increment(double)` | Same | `Counter.java:40,46` | Identical |
| `measure()` default | Same | `Counter.java:54` | Identical — `Collections.singletonList(new Measurement(this::count, COUNT))` |
| `CumulativeCounter` | `CumulativeCounter` | `CumulativeCounter.java:23` | Identical — extends `AbstractMeter`, uses `DoubleAdder` |

## 3.9 Complete Code

### Production Code

#### File: `src/main/java/dev/linhvu/micrometer/Counter.java` [MODIFIED]

```java
package dev.linhvu.micrometer;

import java.util.Collections;

/**
 * Counters monitor monotonically increasing values. Counters may never be reset to a
 * lesser value. If you need to track a value that goes up and down, use a {@link Gauge}.
 */
public interface Counter extends Meter {

    /**
     * Create a fluent builder for a counter.
     * @param name The counter's name.
     * @return A new builder.
     */
    static Builder builder(String name) {
        return new Builder(name);
    }

    /**
     * Update the counter by one.
     */
    default void increment() {
        increment(1.0);
    }

    /**
     * Update the counter by {@code amount}.
     * @param amount Amount to add to the counter.
     */
    void increment(double amount);

    /**
     * @return The cumulative count since this counter was created.
     */
    double count();

    @Override
    default Iterable<Measurement> measure() {
        return Collections.singletonList(new Measurement(this::count, Statistic.COUNT));
    }

    /**
     * Fluent builder for counters. Accumulates name, tags, description, and base unit.
     * The terminal operation {@code register(MeterRegistry)} will be available once
     * MeterRegistry is implemented (Feature 4).
     */
    class Builder {

        private final String name;

        private Tags tags = Tags.empty();

        private String description;

        private String baseUnit;

        private Builder(String name) {
            this.name = name;
        }

        /**
         * @param tags Must be an even number of arguments representing key/value pairs.
         * @return This builder with added tags.
         */
        public Builder tags(String... tags) {
            return tags(Tags.of(tags));
        }

        /**
         * @param tags Tags to add to the eventual counter.
         * @return This builder with added tags.
         */
        public Builder tags(Iterable<Tag> tags) {
            this.tags = this.tags.and(tags);
            return this;
        }

        /**
         * @param key The tag key.
         * @param value The tag value.
         * @return This builder with a single added tag.
         */
        public Builder tag(String key, String value) {
            this.tags = tags.and(key, value);
            return this;
        }

        /**
         * @param description Description text of the eventual counter.
         * @return This builder with added description.
         */
        public Builder description(String description) {
            this.description = description;
            return this;
        }

        /**
         * @param unit Base unit of the eventual counter.
         * @return This builder with added base unit.
         */
        public Builder baseUnit(String unit) {
            this.baseUnit = unit;
            return this;
        }

        // --- Accessors for MeterRegistry to use when creating the Id ---

        public String getName() {
            return name;
        }

        public Tags getTags() {
            return tags;
        }

        public String getDescription() {
            return description;
        }

        public String getBaseUnit() {
            return baseUnit;
        }

    }

}
```

#### File: `src/main/java/dev/linhvu/micrometer/cumulative/CumulativeCounter.java` [NEW]

```java
package dev.linhvu.micrometer.cumulative;

import dev.linhvu.micrometer.AbstractMeter;
import dev.linhvu.micrometer.Counter;
import dev.linhvu.micrometer.Meter;

import java.util.concurrent.atomic.DoubleAdder;

/**
 * A counter implementation that accumulates values using a lock-free {@link DoubleAdder}.
 * The count is cumulative — it only grows over the lifetime of this counter.
 */
public class CumulativeCounter extends AbstractMeter implements Counter {

    private final DoubleAdder value;

    public CumulativeCounter(Meter.Id id) {
        super(id);
        this.value = new DoubleAdder();
    }

    @Override
    public void increment(double amount) {
        value.add(amount);
    }

    @Override
    public double count() {
        return value.sum();
    }

}
```

### Test Code

#### File: `src/test/java/dev/linhvu/micrometer/CounterTest.java` [NEW]

```java
package dev.linhvu.micrometer;

import dev.linhvu.micrometer.cumulative.CumulativeCounter;
import org.junit.jupiter.api.Test;

import java.util.List;

import static org.assertj.core.api.Assertions.assertThat;

class CounterTest {

    private static Meter.Id counterId(String name) {
        return new Meter.Id(name, Tags.empty(), Meter.Type.COUNTER, null, null);
    }

    private static Meter.Id counterId(String name, Tags tags) {
        return new Meter.Id(name, tags, Meter.Type.COUNTER, null, null);
    }

    // --- increment / count ---

    @Test
    void shouldStartAtZero_WhenCreated() {
        Counter counter = new CumulativeCounter(counterId("requests"));
        assertThat(counter.count()).isEqualTo(0.0);
    }

    @Test
    void shouldIncrementByOne_WhenIncrementCalledWithNoArgs() {
        Counter counter = new CumulativeCounter(counterId("requests"));
        counter.increment();
        assertThat(counter.count()).isEqualTo(1.0);
    }

    @Test
    void shouldIncrementByAmount_WhenIncrementCalledWithValue() {
        Counter counter = new CumulativeCounter(counterId("requests"));
        counter.increment(5.0);
        assertThat(counter.count()).isEqualTo(5.0);
    }

    @Test
    void shouldAccumulateIncrements_WhenCalledMultipleTimes() {
        Counter counter = new CumulativeCounter(counterId("requests"));
        counter.increment(2.5);
        counter.increment(3.5);
        counter.increment();
        assertThat(counter.count()).isEqualTo(7.0);
    }

    @Test
    void shouldAcceptFractionalAmounts_WhenIncrementedByDouble() {
        Counter counter = new CumulativeCounter(counterId("bytes"));
        counter.increment(0.1);
        counter.increment(0.2);
        assertThat(counter.count()).isCloseTo(0.3, org.assertj.core.data.Offset.offset(1e-10));
    }

    // --- measure() ---

    @Test
    void shouldReturnSingleCountMeasurement_WhenMeasureCalled() {
        Counter counter = new CumulativeCounter(counterId("requests"));
        counter.increment(42.0);

        List<Measurement> measurements = (List<Measurement>) counter.measure();
        assertThat(measurements).hasSize(1);
        assertThat(measurements.get(0).getStatistic()).isEqualTo(Statistic.COUNT);
        assertThat(measurements.get(0).getValue()).isEqualTo(42.0);
    }

    @Test
    void shouldReflectLiveCount_WhenMeasurementEvaluatedLazily() {
        Counter counter = new CumulativeCounter(counterId("requests"));
        Iterable<Measurement> measurements = counter.measure();

        counter.increment(10.0);

        // The measurement was captured before increment, but evaluates lazily
        Measurement m = measurements.iterator().next();
        assertThat(m.getValue()).isEqualTo(10.0);
    }

    // --- Builder ---

    @Test
    void shouldBuildWithName_WhenUsingBuilder() {
        Counter.Builder builder = Counter.builder("http.requests");
        assertThat(builder.getName()).isEqualTo("http.requests");
    }

    @Test
    void shouldAccumulateTags_WhenUsingBuilderTagMethods() {
        Counter.Builder builder = Counter.builder("http.requests")
                .tag("method", "GET")
                .tags("status", "200", "uri", "/api");

        assertThat(builder.getTags()).containsExactly(
                Tag.of("method", "GET"),
                Tag.of("status", "200"),
                Tag.of("uri", "/api"));
    }

    @Test
    void shouldSetDescription_WhenUsingBuilder() {
        Counter.Builder builder = Counter.builder("http.requests")
                .description("Total HTTP requests received");

        assertThat(builder.getDescription()).isEqualTo("Total HTTP requests received");
    }

    @Test
    void shouldSetBaseUnit_WhenUsingBuilder() {
        Counter.Builder builder = Counter.builder("data.received")
                .baseUnit("bytes");

        assertThat(builder.getBaseUnit()).isEqualTo("bytes");
    }

    // --- Visitor pattern ---

    @Test
    void shouldMatchAsCounter_WhenUsingVisitorPattern() {
        Counter counter = new CumulativeCounter(counterId("requests"));
        counter.increment(5.0);

        String result = counter.match(
                c -> "counter:" + c.count(),
                gauge -> "gauge", timer -> "timer",
                ds -> "ds", ltt -> "ltt", fc -> "fc", ft -> "ft", m -> "default");

        assertThat(result).isEqualTo("counter:5.0");
    }

    // --- Identity ---

    @Test
    void shouldBeEqual_WhenCountersHaveSameId() {
        Counter a = new CumulativeCounter(counterId("requests", Tags.of("method", "GET")));
        Counter b = new CumulativeCounter(counterId("requests", Tags.of("method", "GET")));
        a.increment(10);
        assertThat(a).isEqualTo(b);
    }

    @Test
    void shouldNotBeEqual_WhenCountersHaveDifferentTags() {
        Counter a = new CumulativeCounter(counterId("requests", Tags.of("method", "GET")));
        Counter b = new CumulativeCounter(counterId("requests", Tags.of("method", "POST")));
        assertThat(a).isNotEqualTo(b);
    }

}
```

#### File: `src/test/java/dev/linhvu/micrometer/cumulative/CumulativeCounterTest.java` [NEW]

```java
package dev.linhvu.micrometer.cumulative;

import dev.linhvu.micrometer.Meter;
import dev.linhvu.micrometer.Tags;
import org.junit.jupiter.api.Test;

import java.util.concurrent.CountDownLatch;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

import static org.assertj.core.api.Assertions.assertThat;

class CumulativeCounterTest {

    private static Meter.Id id(String name) {
        return new Meter.Id(name, Tags.empty(), Meter.Type.COUNTER, null, null);
    }

    @Test
    void shouldStartAtZero_WhenCreated() {
        CumulativeCounter counter = new CumulativeCounter(id("test"));
        assertThat(counter.count()).isEqualTo(0.0);
    }

    @Test
    void shouldAccumulate_WhenIncrementedMultipleTimes() {
        CumulativeCounter counter = new CumulativeCounter(id("test"));
        counter.increment(1.0);
        counter.increment(2.0);
        counter.increment(3.0);
        assertThat(counter.count()).isEqualTo(6.0);
    }

    @Test
    void shouldNeverDecrease_WhenOnlyPositiveAmountsAdded() {
        CumulativeCounter counter = new CumulativeCounter(id("test"));
        double previous = 0.0;
        for (int i = 0; i < 100; i++) {
            counter.increment(1.0);
            double current = counter.count();
            assertThat(current).isGreaterThanOrEqualTo(previous);
            previous = current;
        }
    }

    @Test
    void shouldHandleConcurrentIncrements_WhenAccessedFromMultipleThreads() throws Exception {
        CumulativeCounter counter = new CumulativeCounter(id("concurrent"));
        int threadCount = 10;
        int incrementsPerThread = 1000;

        ExecutorService executor = Executors.newFixedThreadPool(threadCount);
        CountDownLatch latch = new CountDownLatch(threadCount);

        for (int t = 0; t < threadCount; t++) {
            executor.submit(() -> {
                for (int i = 0; i < incrementsPerThread; i++) {
                    counter.increment();
                }
                latch.countDown();
            });
        }

        latch.await();
        executor.shutdown();

        assertThat(counter.count()).isEqualTo(threadCount * incrementsPerThread);
    }

    @Test
    void shouldReturnCorrectId_WhenGetIdCalled() {
        Meter.Id expected = id("http.requests");
        CumulativeCounter counter = new CumulativeCounter(expected);
        assertThat(counter.getId()).isSameAs(expected);
    }

    @Test
    void shouldIncludeClassNameInToString() {
        CumulativeCounter counter = new CumulativeCounter(id("my.counter"));
        assertThat(counter.toString()).contains("CumulativeCounter");
        assertThat(counter.toString()).contains("my.counter");
    }

}
```

#### File: `src/test/java/dev/linhvu/micrometer/MeterVisitorTest.java` [MODIFIED]

```java
package dev.linhvu.micrometer;

import org.junit.jupiter.api.Test;

import java.util.List;
import java.util.concurrent.atomic.AtomicReference;

import static org.assertj.core.api.Assertions.assertThat;

class MeterVisitorTest {

    /**
     * A minimal Counter implementation for testing the visitor pattern.
     */
    static class TestCounter extends AbstractMeter implements Counter {

        TestCounter(Meter.Id id) {
            super(id);
        }

        @Override
        public void increment(double amount) {
            // no-op for visitor testing
        }

        @Override
        public double count() {
            return 0.0;
        }

    }

    /**
     * A minimal Gauge implementation for testing the visitor pattern.
     */
    static class TestGauge extends AbstractMeter implements Gauge {

        TestGauge(Meter.Id id) {
            super(id);
        }

        @Override
        public Iterable<Measurement> measure() {
            return List.of(new Measurement(() -> 0.0, Statistic.VALUE));
        }

    }

    /**
     * A custom meter that doesn't match any specific type — should hit the default.
     */
    static class CustomMeter extends AbstractMeter {

        CustomMeter(Meter.Id id) {
            super(id);
        }

        @Override
        public Iterable<Measurement> measure() {
            return List.of(new Measurement(() -> 0.0, Statistic.UNKNOWN));
        }

    }

    private static Meter.Id id(String name, Meter.Type type) {
        return new Meter.Id(name, Tags.empty(), type, null, null);
    }

    @Test
    void shouldMatchCounter_WhenMeterIsCounter() {
        Meter meter = new TestCounter(id("requests", Meter.Type.COUNTER));

        String result = meter.match(
                counter -> "counter",
                gauge -> "gauge",
                timer -> "timer",
                ds -> "ds",
                ltt -> "ltt",
                fc -> "fc",
                ft -> "ft",
                m -> "default"
        );

        assertThat(result).isEqualTo("counter");
    }

    @Test
    void shouldMatchGauge_WhenMeterIsGauge() {
        Meter meter = new TestGauge(id("temperature", Meter.Type.GAUGE));

        String result = meter.match(
                counter -> "counter",
                gauge -> "gauge",
                timer -> "timer",
                ds -> "ds",
                ltt -> "ltt",
                fc -> "fc",
                ft -> "ft",
                m -> "default"
        );

        assertThat(result).isEqualTo("gauge");
    }

    @Test
    void shouldMatchDefault_WhenMeterIsCustomType() {
        Meter meter = new CustomMeter(id("custom", Meter.Type.OTHER));

        String result = meter.match(
                counter -> "counter",
                gauge -> "gauge",
                timer -> "timer",
                ds -> "ds",
                ltt -> "ltt",
                fc -> "fc",
                ft -> "ft",
                m -> "default"
        );

        assertThat(result).isEqualTo("default");
    }

    @Test
    void shouldCallCorrectConsumer_WhenUsingUseVisitor() {
        Meter meter = new TestCounter(id("requests", Meter.Type.COUNTER));
        AtomicReference<String> visited = new AtomicReference<>("none");

        meter.use(
                counter -> visited.set("counter"),
                gauge -> visited.set("gauge"),
                timer -> visited.set("timer"),
                ds -> visited.set("ds"),
                ltt -> visited.set("ltt"),
                fc -> visited.set("fc"),
                ft -> visited.set("ft"),
                m -> visited.set("default")
        );

        assertThat(visited.get()).isEqualTo("counter");
    }

    @Test
    void shouldPassMeterInstance_WhenVisiting() {
        TestCounter counter = new TestCounter(id("requests", Meter.Type.COUNTER));

        Counter matched = counter.match(
                c -> c,
                gauge -> null,
                timer -> null,
                ds -> null,
                ltt -> null,
                fc -> null,
                ft -> null,
                m -> null
        );

        assertThat(matched).isSameAs(counter);
    }

}
```

---

## Summary

| Concept | What It Means |
|---------|---------------|
| **Counter** | A monotonically increasing metric — can only go up. Use for request counts, errors, bytes sent. |
| **`DoubleAdder`** | A JDK concurrent primitive optimized for write-heavy/read-light workloads — spreads updates across per-thread cells. |
| **`measure()` default method** | Bridges meter-specific APIs (like `count()`) to the generic `Measurement` pipeline that backends consume. |
| **Builder pattern** | Accumulates meter metadata (name, tags, description) fluently. Terminal `register()` will come with MeterRegistry. |
| **Cumulative semantics** | The count grows forever — never resets. Contrast with *step* semantics (Feature 16) where counts reset each interval. |

**Next: Chapter 4 — MeterRegistry & SimpleMeterRegistry** — The central registry that manages meter lifecycle: creation, deduplication, and lookup. The Builder's `register()` method will finally connect, and you'll see how the same `Counter.builder()` call can produce different Counter implementations depending on which registry it targets.
