# Chapter 7: DistributionSummary

> Line references based on commit `2c8a4606c` of the Micrometer repository.

## Build Challenge

| Current State | Limitation | Objective |
|---------------|------------|-----------|
| Registry supports Counter, Gauge, and Timer, but all value recording is either monotonic (Counter) or time-based (Timer) | No way to record the distribution of non-time values — response sizes, batch counts, payload lengths, queue depths | Build a `DistributionSummary` that records arbitrary double values with COUNT, TOTAL, and decaying MAX, plus a scale factor for unit conversion |

---

## 7.1 The Integration Point

DistributionSummary plugs into the registry through the same Builder → `registry.summary(Id, scale)` → `newDistributionSummary()` template method chain used by Counter, Gauge, and Timer.

**Modifying:** `src/main/java/dev/linhvu/micrometer/MeterRegistry.java`
**Change:** Add public `summary()` convenience methods, a package-private `summary(Meter.Id, double)` registration method, and update the `newDistributionSummary()` abstract factory to accept a `scale` parameter

```java
import dev.linhvu.micrometer.noop.NoopDistributionSummary;

// ... existing code ...

// --- Public convenience registration methods (add after timer methods) ---

public DistributionSummary summary(String name, String... tags) {
    return DistributionSummary.builder(name).tags(tags).register(this);
}

public DistributionSummary summary(String name, Iterable<Tag> tags) {
    return DistributionSummary.builder(name).tags(tags).register(this);
}

// --- Package-private registration (add after timer(Meter.Id)) ---

DistributionSummary summary(Meter.Id id, double scale) {
    return registerMeterIfNecessary(DistributionSummary.class, id,
            mappedId -> newDistributionSummary(mappedId, scale), NoopDistributionSummary::new);
}

// --- Update the abstract factory signature ---

protected abstract DistributionSummary newDistributionSummary(Meter.Id id, double scale);
```

Two key decisions here:

1. **Scale travels through the registry** — unlike Timer where the time unit is determined by the *registry*, DistributionSummary's scale is determined by the *user* via the Builder. The registry passes it through to the factory method because the scale must be applied in the abstract layer (before concrete storage).
2. **Same deduplication pattern** — the registration uses the existing `registerMeterIfNecessary()` infrastructure. A DistributionSummary with the same name+tags returns the same instance; a name collision with a different meter type throws `IllegalArgumentException`.

This integration connects **the DistributionSummary API** to **the MeterRegistry lifecycle**. To make it work, we need to build:
- `DistributionSummary` interface — with record, count, totalAmount, max, mean, and Builder
- `AbstractDistributionSummary` — template method for recording with validation and scaling
- `CumulativeDistributionSummary` — concrete implementation with DoubleAdder + TimeWindowMax
- `NoopDistributionSummary` — silent fallback for closed registries

---

## 7.2 The DistributionSummary Interface

**Modifying:** `src/main/java/dev/linhvu/micrometer/DistributionSummary.java`
**Change:** Replace the stub interface with the full API — recording methods, statistics, `measure()`, and a fluent `Builder`

```java
package dev.linhvu.micrometer;

import java.util.Arrays;

public interface DistributionSummary extends Meter {

    static Builder builder(String name) {
        return new Builder(name);
    }

    void record(double amount);

    long count();

    double totalAmount();

    default double mean() {
        long count = count();
        return count == 0 ? 0 : totalAmount() / count;
    }

    double max();

    @Override
    default Iterable<Measurement> measure() {
        return Arrays.asList(
                new Measurement(() -> (double) count(), Statistic.COUNT),
                new Measurement(this::totalAmount, Statistic.TOTAL),
                new Measurement(this::max, Statistic.MAX));
    }

    class Builder {

        private final String name;
        private Tags tags = Tags.empty();
        private String description;
        private String baseUnit;
        private double scale = 1.0;

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

        public Builder scale(double scale) {
            this.scale = scale;
            return this;
        }

        public DistributionSummary register(MeterRegistry registry) {
            return registry.summary(
                    new Meter.Id(name, tags, Meter.Type.DISTRIBUTION_SUMMARY, description, baseUnit),
                    scale);
        }
    }
}
```

Notice how `measure()` uses `Statistic.TOTAL` (not `TOTAL_TIME`). This is the fundamental distinction from Timer — the same three-measurement structure (COUNT, total, MAX) but the "total" statistic carries different semantics: accumulated amount vs. accumulated duration.

---

## 7.3 AbstractDistributionSummary — The Template Method

**New file:** `src/main/java/dev/linhvu/micrometer/AbstractDistributionSummary.java`

```java
package dev.linhvu.micrometer;

public abstract class AbstractDistributionSummary extends AbstractMeter implements DistributionSummary {

    private final double scale;

    protected AbstractDistributionSummary(Meter.Id id, double scale) {
        super(id);
        this.scale = scale;
    }

    @Override
    public final void record(double amount) {
        if (amount >= 0) {
            double scaledAmount = this.scale * amount;
            recordNonNegative(scaledAmount);
        }
    }

    protected abstract void recordNonNegative(double amount);
}
```

The pattern is identical to `AbstractTimer.record(long, TimeUnit)`:
- `record()` is `final` — no subclass can bypass validation
- Negative values are silently dropped
- Positive values are scaled, then forwarded to the abstract `recordNonNegative()`

The key difference: no time-unit conversion. Timer converts to nanoseconds for internal storage; DistributionSummary keeps raw doubles.

---

## 7.4 CumulativeDistributionSummary — The Concrete Implementation

**New file:** `src/main/java/dev/linhvu/micrometer/cumulative/CumulativeDistributionSummary.java`

```java
package dev.linhvu.micrometer.cumulative;

import dev.linhvu.micrometer.AbstractDistributionSummary;
import dev.linhvu.micrometer.Clock;
import dev.linhvu.micrometer.Measurement;
import dev.linhvu.micrometer.Meter;
import dev.linhvu.micrometer.Statistic;
import dev.linhvu.micrometer.distribution.TimeWindowMax;

import java.util.Arrays;
import java.util.concurrent.atomic.DoubleAdder;
import java.util.concurrent.atomic.LongAdder;

public class CumulativeDistributionSummary extends AbstractDistributionSummary {

    private final LongAdder count;
    private final DoubleAdder total;
    private final TimeWindowMax max;

    public CumulativeDistributionSummary(Meter.Id id, Clock clock, double scale) {
        super(id, scale);
        this.count = new LongAdder();
        this.total = new DoubleAdder();
        this.max = new TimeWindowMax(clock);
    }

    @Override
    protected void recordNonNegative(double amount) {
        count.increment();
        total.add(amount);
        max.record(amount);
    }

    @Override
    public long count() { return count.longValue(); }

    @Override
    public double totalAmount() { return total.sum(); }

    @Override
    public double max() { return max.poll(); }

    @Override
    public Iterable<Measurement> measure() {
        return Arrays.asList(
                new Measurement(() -> (double) count(), Statistic.COUNT),
                new Measurement(this::totalAmount, Statistic.TOTAL),
                new Measurement(this::max, Statistic.MAX));
    }
}
```

Compare with `CumulativeTimer`:

| Aspect | CumulativeTimer | CumulativeDistributionSummary |
|--------|----------------|-------------------------------|
| Count storage | `LongAdder` | `LongAdder` |
| Total storage | `LongAdder` (nanoseconds) | `DoubleAdder` (raw doubles) |
| Max recording | `max.record(double, TimeUnit)` | `max.record(double)` |
| Max reading | `max.poll(TimeUnit)` | `max.poll()` |
| Total statistic | `TOTAL_TIME` | `TOTAL` |

The `DoubleAdder` vs `LongAdder` for total is the critical difference — durations are always whole nanoseconds (longs), but amounts like "1.5 KB" are fractional (doubles).

---

## 7.5 NoopDistributionSummary — The Silent Fallback

**New file:** `src/main/java/dev/linhvu/micrometer/noop/NoopDistributionSummary.java`

```java
package dev.linhvu.micrometer.noop;

import dev.linhvu.micrometer.DistributionSummary;
import dev.linhvu.micrometer.Meter;

public class NoopDistributionSummary extends NoopMeter implements DistributionSummary {

    public NoopDistributionSummary(Meter.Id id) {
        super(id);
    }

    @Override
    public void record(double amount) { }

    @Override
    public long count() { return 0; }

    @Override
    public double totalAmount() { return 0; }

    @Override
    public double max() { return 0; }
}
```

---

## 7.6 Wire Into SimpleMeterRegistry

**Modifying:** `src/main/java/dev/linhvu/micrometer/simple/SimpleMeterRegistry.java`
**Change:** Replace the stub `newDistributionSummary()` with the real implementation

```java
import dev.linhvu.micrometer.cumulative.CumulativeDistributionSummary;

// ... existing code ...

@Override
protected DistributionSummary newDistributionSummary(Meter.Id id, double scale) {
    return new CumulativeDistributionSummary(id, clock, scale);
}
```

---

## 7.7 Try It Yourself

Before looking at the solutions above, try implementing these yourself:

<details><summary>Challenge 1: Implement the DistributionSummary interface</summary>

The interface needs:
- `record(double)` — the main recording method
- `count()`, `totalAmount()`, `max()` — statistics
- `mean()` — default method computed from count and totalAmount
- `measure()` — default method returning COUNT, TOTAL, MAX measurements
- A `Builder` inner class with `tags()`, `description()`, `baseUnit()`, `scale()`, `register()`

Key question: Which `Statistic` should the total use — `TOTAL` or `TOTAL_TIME`?

Answer: `TOTAL`. This is not a timer. The distinction matters when backends format the metric name (e.g., Prometheus appends `_total` vs `_seconds_total`).

</details>

<details><summary>Challenge 2: Implement AbstractDistributionSummary with scale</summary>

The abstract class needs:
- Store the `scale` factor
- Make `record(double)` final — reject negatives, apply scale, delegate to `recordNonNegative(double)`
- Declare `recordNonNegative(double)` as abstract

Key question: Should scale be applied before or after the negative check?

Answer: Before in the sense that scale is applied to the value, but the negative check happens on the *original* amount. If the user records -5.0, it's dropped regardless of scale. This prevents confusion where `record(-1.0)` with `scale(0.001)` would silently record -0.001.

</details>

<details><summary>Challenge 3: Implement CumulativeDistributionSummary</summary>

The concrete class needs:
- `LongAdder` for count (integer, increment-only)
- `DoubleAdder` for total (floating-point)
- `TimeWindowMax` for max (reuse from Timer, but using the `record(double)` / `poll()` API)

Key question: Why `DoubleAdder` for total and not `LongAdder`?

Answer: Unlike Timer (which converts everything to nanosecond longs), DistributionSummary records raw doubles. Values like 1.5 KB or 0.003 MB are fractional. `LongAdder` would silently truncate them.

</details>

---

## 7.8 Tests

### Unit Tests

**`DistributionSummaryTest`** — Tests the interface API contract:

```java
@Test
void shouldRecordAmount_WhenPositiveValue() {
    DistributionSummary summary = DistributionSummary.builder("test").register(registry);
    summary.record(100.0);
    summary.record(200.0);
    assertThat(summary.count()).isEqualTo(2);
    assertThat(summary.totalAmount()).isEqualTo(300.0);
}

@Test
void shouldApplyScaleFactor_WhenScaleConfigured() {
    DistributionSummary summary = DistributionSummary.builder("test")
            .scale(1.0 / 1024)
            .register(registry);
    summary.record(2048.0);
    assertThat(summary.totalAmount()).isEqualTo(2.0);  // 2 KB
}

@Test
void shouldProduceThreeMeasurements_WhenMeasured() {
    DistributionSummary summary = DistributionSummary.builder("test").register(registry);
    summary.record(100.0);
    summary.record(300.0);
    var measurementList = new ArrayList<Measurement>();
    summary.measure().forEach(measurementList::add);
    assertThat(measurementList).hasSize(3);
    assertThat(measurementList.get(0).getStatistic()).isEqualTo(Statistic.COUNT);
    assertThat(measurementList.get(1).getStatistic()).isEqualTo(Statistic.TOTAL);  // NOT TOTAL_TIME
    assertThat(measurementList.get(2).getStatistic()).isEqualTo(Statistic.MAX);
}
```

**`AbstractDistributionSummaryTest`** — Tests the template method contract:

```java
@Test
void shouldApplyScale_WhenScaleIsNotOne() {
    var summary = new TestDistributionSummary(0.001);
    summary.record(2000.0);
    assertThat(summary.recorded).containsExactly(2.0);
}

@Test
void shouldDropNegativeValues_WhenRecordCalled() {
    var summary = new TestDistributionSummary(1.0);
    summary.record(-1.0);
    assertThat(summary.recorded).isEmpty();
}
```

**`CumulativeDistributionSummaryTest`** — Tests the concrete implementation:

```java
@Test
void shouldDecayMax_WhenTimeWindowExpires() {
    var summary = createSummary();
    summary.record(999.0);
    clock.add(121, TimeUnit.SECONDS);  // past full decay window
    assertThat(summary.max()).isEqualTo(0.0);
}

@Test
void shouldBeThreadSafe_WhenRecordedConcurrently() throws Exception {
    var summary = createSummary();
    int threads = 8, recordsPerThread = 1000;
    // ... concurrent recording ...
    assertThat(summary.count()).isEqualTo(8000L);
    assertThat(summary.totalAmount()).isEqualTo(8000.0);
}
```

### Integration Tests

**`DistributionSummaryIntegrationTest`** — Tests the full lifecycle with the registry:

```java
@Test
void shouldCoexistWithOtherMeterTypes_WhenRegisteredOnSameRegistry() {
    Counter counter = registry.counter("events");
    Timer timer = registry.timer("latency");
    DistributionSummary summary = registry.summary("response.size");
    assertThat(registry.getMeters()).hasSize(4);  // +gauge
}

@Test
void shouldReturnNoopSummary_WhenRegistryIsClosed() {
    registry.close();
    DistributionSummary summary = registry.summary("late.summary");
    assertThat(summary).isInstanceOf(NoopDistributionSummary.class);
}

@Test
void shouldAppearInMeterVisitor_WhenMatchCalled() {
    DistributionSummary summary = registry.summary("test.summary");
    summary.record(42.0);
    String result = summary.match(
            c -> "counter", g -> "gauge", t -> "timer",
            ds -> "summary:" + ds.count(),
            ltt -> "longTaskTimer", fc -> "functionCounter",
            ft -> "functionTimer", m -> "other");
    assertThat(result).isEqualTo("summary:1");
}
```

Run all tests:

```bash
./gradlew test
```

All 242 tests pass — including 42 new DistributionSummary tests across 5 test classes.

---

## 7.9 Why This Works

`★ Insight — Timer's Sibling, Not Its Copy ─────`
DistributionSummary and Timer share the same *architecture* (interface → abstract template → cumulative impl → noop) but differ in *data types*. Timer uses `long` nanoseconds internally because time is fundamentally discrete at the nanosecond level. DistributionSummary uses `double` throughout because amounts can be fractional. This ripples into every layer: `DoubleAdder` vs `LongAdder` for total, `TimeWindowMax.record(double)` vs `record(double, TimeUnit)`, and `Statistic.TOTAL` vs `TOTAL_TIME`. The real Micrometer source at `AbstractDistributionSummary.java:69-74` shows the exact same `record()` → scale → `recordNonNegative()` pipeline.
`─────────────────────────────────────────────────`

`★ Insight — Scale Factor as a Design Choice ───`
The `scale` factor lives in the abstract layer (not the interface or concrete class) because it's a *recording-time* concern, not a *storage* concern. Scaling happens once, immediately when `record()` is called — the concrete implementation never sees the original value. This means `count()` counts scaled recordings, `totalAmount()` sums scaled values, and `max()` tracks the scaled maximum. In the real Micrometer, this is used for unit conversion (e.g., recording bytes but reporting kilobytes) without requiring the caller to do the math. See `AbstractDistributionSummary.java:71` in the real source.
`─────────────────────────────────────────────────`

`★ Insight — DoubleAdder and Double.doubleToLongBits ──`
`TimeWindowMax` uses `AtomicLong` internally, so how does it store doubles? Via `Double.doubleToLongBits()`, which converts a double to a 64-bit long representation. The critical property: for non-negative doubles, `doubleToLongBits` preserves ordering — `a > b` implies `doubleToLongBits(a) > doubleToLongBits(b)`. This means the CAS-based `updateMax()` loop works correctly for doubles without any floating-point comparison. Timer doesn't need this trick because it stores nanosecond longs directly.
`─────────────────────────────────────────────────`

---

## 7.10 What We Enhanced

| File | What Changed | Why |
|------|-------------|-----|
| `MeterRegistry.java` | Added `summary(String, String...)`, `summary(String, Iterable<Tag>)`, and `summary(Meter.Id, double)` methods; changed `newDistributionSummary(Id)` to `newDistributionSummary(Id, double)` | DistributionSummary needs a scale parameter passed through the registry to the abstract layer |
| `SimpleMeterRegistry.java` | Replaced `UnsupportedOperationException` stub with `CumulativeDistributionSummary` creation | Enables DistributionSummary in the in-memory registry |

---

## 7.11 Connection to Real Framework

| Simplified | Real Micrometer | Location (commit `2c8a4606c`) |
|-----------|----------------|-------------------------------|
| `DistributionSummary` interface | `DistributionSummary` | `micrometer-core/.../instrument/DistributionSummary.java:35` |
| `DistributionSummary.Builder` | `DistributionSummary.Builder` | `micrometer-core/.../instrument/DistributionSummary.java:116-419` |
| `AbstractDistributionSummary` | `AbstractDistributionSummary` | `micrometer-core/.../instrument/AbstractDistributionSummary.java:21` |
| `AbstractDistributionSummary.record()` | `AbstractDistributionSummary.record()` | `micrometer-core/.../instrument/AbstractDistributionSummary.java:68-75` |
| `CumulativeDistributionSummary` | `CumulativeDistributionSummary` | `micrometer-core/.../instrument/cumulative/CumulativeDistributionSummary.java:37` |
| `NoopDistributionSummary` | `NoopDistributionSummary` | `micrometer-core/.../instrument/noop/NoopDistributionSummary.java:21` |

**What we simplified:**
- **No Histogram integration** — the real `AbstractDistributionSummary` holds a `Histogram` field and calls `histogram.recordDouble(scaledAmount)` before `recordNonNegative()`. We skip this (Feature 10 adds it).
- **No `HistogramSupport` interface** — the real `DistributionSummary` extends `HistogramSupport` with `takeSnapshot()`. We omit this until Distribution Statistics.
- **No `DistributionStatisticConfig`** — the real Builder accepts percentile/SLO/bucket configuration. We strip this out as it belongs to Feature 10.
- **No `MeterProvider`** — the real Builder has `withRegistry()` for dynamic tagging. We omit this advanced pattern.
- **No deprecated compatibility methods** — the real interface has `histogramCountAtValue()` and `percentile()` deprecated methods.

---

## 7.12 Complete Code

### Production Code

#### `[MODIFIED]` `src/main/java/dev/linhvu/micrometer/DistributionSummary.java`

```java
package dev.linhvu.micrometer;

import java.util.Arrays;

/**
 * Records the distribution of values — like request sizes, payload lengths, or batch counts.
 * Similar to {@link Timer} but for non-time measurements: no time-unit conversion.
 * <p>
 * A distribution summary produces three measurements:
 * <ul>
 *   <li>{@link Statistic#COUNT} — total number of recorded events</li>
 *   <li>{@link Statistic#TOTAL} — cumulative amount of all events</li>
 *   <li>{@link Statistic#MAX} — decaying maximum value (resets over time)</li>
 * </ul>
 * <p>
 * An optional {@code scale} factor can be applied to all recorded values — for example,
 * recording bytes but scaling to kilobytes.
 */
public interface DistributionSummary extends Meter {

    static Builder builder(String name) {
        return new Builder(name);
    }

    /**
     * Updates the statistics kept by the summary with the specified amount.
     * If the amount is less than 0, the value will be dropped.
     *
     * @param amount Amount for an event being measured (e.g., response size in bytes).
     */
    void record(double amount);

    /**
     * @return The number of times that record has been called since this summary was created.
     */
    long count();

    /**
     * @return The total amount of all recorded events.
     */
    double totalAmount();

    /**
     * @return The distribution average for all recorded events, or 0 if no recordings.
     */
    default double mean() {
        long count = count();
        return count == 0 ? 0 : totalAmount() / count;
    }

    /**
     * @return The maximum value of a single event within the decay window.
     */
    double max();

    @Override
    default Iterable<Measurement> measure() {
        return Arrays.asList(
                new Measurement(() -> (double) count(), Statistic.COUNT),
                new Measurement(this::totalAmount, Statistic.TOTAL),
                new Measurement(this::max, Statistic.MAX));
    }

    /**
     * Fluent builder for distribution summaries.
     */
    class Builder {

        private final String name;

        private Tags tags = Tags.empty();

        private String description;

        private String baseUnit;

        private double scale = 1.0;

        private Builder(String name) {
            this.name = name;
        }

        /**
         * @param tags Must be an even number of arguments representing key/value pairs.
         * @return This builder.
         */
        public Builder tags(String... tags) {
            return tags(Tags.of(tags));
        }

        /**
         * @param tags Tags to add to the eventual distribution summary.
         * @return This builder.
         */
        public Builder tags(Iterable<Tag> tags) {
            this.tags = this.tags.and(tags);
            return this;
        }

        /**
         * @param key   The tag key.
         * @param value The tag value.
         * @return This builder.
         */
        public Builder tag(String key, String value) {
            this.tags = tags.and(key, value);
            return this;
        }

        /**
         * @param description Description text of the eventual distribution summary.
         * @return This builder.
         */
        public Builder description(String description) {
            this.description = description;
            return this;
        }

        /**
         * @param unit Base unit of the eventual distribution summary (e.g., "bytes").
         * @return This builder.
         */
        public Builder baseUnit(String unit) {
            this.baseUnit = unit;
            return this;
        }

        /**
         * Multiply values recorded to the distribution summary by a scaling factor.
         * For example, a scale of {@code 1.0/1024} converts bytes to kilobytes.
         *
         * @param scale Factor to scale each recorded value by.
         * @return This builder.
         */
        public Builder scale(double scale) {
            this.scale = scale;
            return this;
        }

        /**
         * Registers this distribution summary on the given registry, or returns
         * an existing one with the same name and tags.
         */
        public DistributionSummary register(MeterRegistry registry) {
            return registry.summary(
                    new Meter.Id(name, tags, Meter.Type.DISTRIBUTION_SUMMARY, description, baseUnit),
                    scale);
        }

        public String getName() { return name; }
        public Tags getTags() { return tags; }
        public String getDescription() { return description; }
        public String getBaseUnit() { return baseUnit; }
        public double getScale() { return scale; }
    }

}
```

#### `[NEW]` `src/main/java/dev/linhvu/micrometer/AbstractDistributionSummary.java`

```java
package dev.linhvu.micrometer;

/**
 * Base class for {@link DistributionSummary} implementations. Provides the shared
 * recording logic with scaling and non-negative validation.
 * <p>
 * The recording pipeline mirrors {@link AbstractTimer}:
 * <ol>
 *   <li>{@code record(double)} is {@code final}: it rejects negatives, applies
 *       the {@code scale} factor, and delegates to {@link #recordNonNegative(double)}.</li>
 *   <li>Subclasses implement only {@code recordNonNegative(double)} — the storage.</li>
 * </ol>
 * <p>
 * <b>Key difference from Timer:</b> No time-unit conversion. Values are raw doubles
 * throughout (bytes, items, etc.), not nanoseconds. This is why
 * {@code recordNonNegative} takes a {@code double}, not a {@code long + TimeUnit}.
 */
public abstract class AbstractDistributionSummary extends AbstractMeter implements DistributionSummary {

    private final double scale;

    protected AbstractDistributionSummary(Meter.Id id, double scale) {
        super(id);
        this.scale = scale;
    }

    /**
     * The final recording method. Rejects negative values, applies the scale factor,
     * and delegates to {@link #recordNonNegative(double)}.
     * <p>
     * Making this final ensures that all recording paths go through the same
     * validation — subclasses can't accidentally bypass it.
     */
    @Override
    public final void record(double amount) {
        if (amount >= 0) {
            double scaledAmount = this.scale * amount;
            recordNonNegative(scaledAmount);
        }
    }

    /**
     * Subclasses implement this to store the validated, non-negative, scaled amount.
     */
    protected abstract void recordNonNegative(double amount);

}
```

#### `[NEW]` `src/main/java/dev/linhvu/micrometer/cumulative/CumulativeDistributionSummary.java`

```java
package dev.linhvu.micrometer.cumulative;

import dev.linhvu.micrometer.AbstractDistributionSummary;
import dev.linhvu.micrometer.Clock;
import dev.linhvu.micrometer.Measurement;
import dev.linhvu.micrometer.Meter;
import dev.linhvu.micrometer.Statistic;
import dev.linhvu.micrometer.distribution.TimeWindowMax;

import java.util.Arrays;
import java.util.concurrent.atomic.DoubleAdder;
import java.util.concurrent.atomic.LongAdder;

/**
 * Cumulative distribution summary — values accumulate forever (count grows,
 * total grows), while max decays over a time window.
 * <p>
 * Uses lock-free concurrent accumulators:
 * <ul>
 *   <li>{@link LongAdder} for count — increment-only, integer</li>
 *   <li>{@link DoubleAdder} for total — add-only, floating-point</li>
 *   <li>{@link TimeWindowMax} for maximum — ring-buffer-based decaying max</li>
 * </ul>
 * <p>
 * <b>Key difference from CumulativeTimer:</b> Timer uses {@code LongAdder} for total
 * (nanoseconds are always longs) and calls {@code TimeWindowMax.record(double, TimeUnit)}.
 * DistributionSummary uses {@code DoubleAdder} for total (amounts can be fractional)
 * and calls {@code TimeWindowMax.record(double)} — the double-bits path that preserves
 * ordering for non-negative values.
 */
public class CumulativeDistributionSummary extends AbstractDistributionSummary {

    private final LongAdder count;

    private final DoubleAdder total;

    private final TimeWindowMax max;

    public CumulativeDistributionSummary(Meter.Id id, Clock clock, double scale) {
        super(id, scale);
        this.count = new LongAdder();
        this.total = new DoubleAdder();
        this.max = new TimeWindowMax(clock);
    }

    @Override
    protected void recordNonNegative(double amount) {
        count.increment();
        total.add(amount);
        max.record(amount);
    }

    @Override
    public long count() {
        return count.longValue();
    }

    @Override
    public double totalAmount() {
        return total.sum();
    }

    @Override
    public double max() {
        return max.poll();
    }

    @Override
    public Iterable<Measurement> measure() {
        return Arrays.asList(
                new Measurement(() -> (double) count(), Statistic.COUNT),
                new Measurement(this::totalAmount, Statistic.TOTAL),
                new Measurement(this::max, Statistic.MAX));
    }

}
```

#### `[NEW]` `src/main/java/dev/linhvu/micrometer/noop/NoopDistributionSummary.java`

```java
package dev.linhvu.micrometer.noop;

import dev.linhvu.micrometer.DistributionSummary;
import dev.linhvu.micrometer.Meter;

/**
 * A distribution summary that does nothing. All statistics return zero.
 * <p>
 * Used as a fallback when the registry is closed or a MeterFilter denies registration.
 */
public class NoopDistributionSummary extends NoopMeter implements DistributionSummary {

    public NoopDistributionSummary(Meter.Id id) {
        super(id);
    }

    @Override
    public void record(double amount) {
        // no-op
    }

    @Override
    public long count() {
        return 0;
    }

    @Override
    public double totalAmount() {
        return 0;
    }

    @Override
    public double max() {
        return 0;
    }

}
```

#### `[MODIFIED]` `src/main/java/dev/linhvu/micrometer/MeterRegistry.java`

```java
package dev.linhvu.micrometer;

import dev.linhvu.micrometer.noop.NoopCounter;
import dev.linhvu.micrometer.noop.NoopDistributionSummary;
import dev.linhvu.micrometer.noop.NoopGauge;
import dev.linhvu.micrometer.noop.NoopTimer;

import java.util.ArrayList;
import java.util.Collection;
import java.util.Collections;
import java.util.List;
import java.util.Map;
import java.util.Objects;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.TimeUnit;
import java.util.function.Function;
import java.util.function.ToDoubleFunction;

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

    private volatile boolean closed = false;

    protected MeterRegistry(Clock clock) {
        this.clock = Objects.requireNonNull(clock, "clock must not be null");
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

    Timer timer(Meter.Id id) {
        return registerMeterIfNecessary(Timer.class, id, this::newTimer, NoopTimer::new);
    }

    DistributionSummary summary(Meter.Id id, double scale) {
        return registerMeterIfNecessary(DistributionSummary.class, id,
                mappedId -> newDistributionSummary(mappedId, scale), NoopDistributionSummary::new);
    }

    // -----------------------------------------------------------------------
    // Template Method factories — subclasses implement these
    // -----------------------------------------------------------------------

    protected abstract Counter newCounter(Meter.Id id);

    protected abstract <T> Gauge newGauge(Meter.Id id, T obj, ToDoubleFunction<T> valueFunction);

    protected abstract Timer newTimer(Meter.Id id);

    protected abstract DistributionSummary newDistributionSummary(Meter.Id id, double scale);

    protected abstract TimeUnit getBaseTimeUnit();

    // -----------------------------------------------------------------------
    // Registration lifecycle — the heart of deduplication
    // -----------------------------------------------------------------------

    private <M extends Meter> M registerMeterIfNecessary(Class<M> meterClass, Meter.Id id,
            Function<Meter.Id, ? extends Meter> meterFactory,
            Function<Meter.Id, ? extends Meter> noopFactory) {

        Meter meter = getOrCreateMeter(id, meterFactory, noopFactory);

        if (!meterClass.isInstance(meter)) {
            throw new IllegalArgumentException(
                    "There is already a meter registered with name '" + id.getName()
                            + "' and tags " + id.getTagsAsIterable()
                            + " but with type " + meter.getClass().getSimpleName()
                            + ". Attempted to register type " + meterClass.getSimpleName() + ".");
        }

        return meterClass.cast(meter);
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

#### `[MODIFIED]` `src/main/java/dev/linhvu/micrometer/simple/SimpleMeterRegistry.java`

```java
package dev.linhvu.micrometer.simple;

import dev.linhvu.micrometer.Clock;
import dev.linhvu.micrometer.Counter;
import dev.linhvu.micrometer.DistributionSummary;
import dev.linhvu.micrometer.Gauge;
import dev.linhvu.micrometer.Meter;
import dev.linhvu.micrometer.MeterRegistry;
import dev.linhvu.micrometer.Timer;
import dev.linhvu.micrometer.cumulative.CumulativeCounter;
import dev.linhvu.micrometer.cumulative.CumulativeDistributionSummary;
import dev.linhvu.micrometer.cumulative.CumulativeTimer;
import dev.linhvu.micrometer.internal.DefaultGauge;

import java.util.concurrent.TimeUnit;
import java.util.function.ToDoubleFunction;

/**
 * The simplest useful {@link MeterRegistry} — an in-memory implementation backed
 * by cumulative meter types.
 * <p>
 * All meters are cumulative: counters grow forever, timers accumulate total time
 * and count. This is ideal for testing, local development, and as the base
 * implementation for composite registries.
 * <p>
 * Currently supports Counter (via {@link CumulativeCounter}), Gauge (via
 * {@link DefaultGauge}), Timer (via {@link CumulativeTimer}), and
 * DistributionSummary (via {@link CumulativeDistributionSummary}).
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
    protected Timer newTimer(Meter.Id id) {
        return new CumulativeTimer(id, clock, getBaseTimeUnit());
    }

    @Override
    protected DistributionSummary newDistributionSummary(Meter.Id id, double scale) {
        return new CumulativeDistributionSummary(id, clock, scale);
    }

    @Override
    protected TimeUnit getBaseTimeUnit() {
        return TimeUnit.SECONDS;
    }

}
```

### Test Code

#### `[NEW]` `src/test/java/dev/linhvu/micrometer/DistributionSummaryTest.java`

```java
package dev.linhvu.micrometer;

import dev.linhvu.micrometer.simple.SimpleMeterRegistry;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.*;

class DistributionSummaryTest {

    private MockClock clock;
    private SimpleMeterRegistry registry;

    @BeforeEach
    void setUp() {
        clock = new MockClock();
        registry = new SimpleMeterRegistry(clock);
    }

    @Test
    void shouldCreateSummaryViaBuilder_WhenRegisteredOnRegistry() {
        DistributionSummary summary = DistributionSummary.builder("http.response.size")
                .tag("method", "GET")
                .description("Response payload size")
                .baseUnit("bytes")
                .register(registry);

        assertThat(summary).isNotNull();
        assertThat(summary.getId().getName()).isEqualTo("http.response.size");
        assertThat(summary.getId().getTag("method")).isEqualTo("GET");
        assertThat(summary.getId().getDescription()).isEqualTo("Response payload size");
        assertThat(summary.getId().getBaseUnit()).isEqualTo("bytes");
    }

    @Test
    void shouldReturnSameInstance_WhenRegisteredTwiceWithSameId() {
        DistributionSummary s1 = DistributionSummary.builder("my.summary").register(registry);
        DistributionSummary s2 = DistributionSummary.builder("my.summary").register(registry);
        assertThat(s1).isSameAs(s2);
    }

    @Test
    void shouldThrowException_WhenNameCollidesWithDifferentType() {
        registry.counter("shared.name");
        assertThatThrownBy(() -> DistributionSummary.builder("shared.name").register(registry))
                .isInstanceOf(IllegalArgumentException.class)
                .hasMessageContaining("shared.name");
    }

    @Test
    void shouldRecordAmount_WhenPositiveValue() {
        DistributionSummary summary = DistributionSummary.builder("test").register(registry);
        summary.record(100.0);
        summary.record(200.0);
        assertThat(summary.count()).isEqualTo(2);
        assertThat(summary.totalAmount()).isEqualTo(300.0);
    }

    @Test
    void shouldDropNegativeValues_WhenRecorded() {
        DistributionSummary summary = DistributionSummary.builder("test").register(registry);
        summary.record(-1.0);
        assertThat(summary.count()).isEqualTo(0);
        assertThat(summary.totalAmount()).isEqualTo(0.0);
    }

    @Test
    void shouldRecordZero_WhenAmountIsZero() {
        DistributionSummary summary = DistributionSummary.builder("test").register(registry);
        summary.record(0.0);
        assertThat(summary.count()).isEqualTo(1);
        assertThat(summary.totalAmount()).isEqualTo(0.0);
    }

    @Test
    void shouldRecordFractionalAmounts_WhenGivenDoubles() {
        DistributionSummary summary = DistributionSummary.builder("test").register(registry);
        summary.record(1.5);
        summary.record(2.7);
        assertThat(summary.count()).isEqualTo(2);
        assertThat(summary.totalAmount()).isCloseTo(4.2, within(0.001));
    }

    @Test
    void shouldApplyScaleFactor_WhenScaleConfigured() {
        DistributionSummary summary = DistributionSummary.builder("test")
                .scale(1.0 / 1024)
                .register(registry);
        summary.record(2048.0);
        assertThat(summary.count()).isEqualTo(1);
        assertThat(summary.totalAmount()).isEqualTo(2.0);
    }

    @Test
    void shouldApplyDefaultScale_WhenNoScaleConfigured() {
        DistributionSummary summary = DistributionSummary.builder("test").register(registry);
        summary.record(42.0);
        assertThat(summary.totalAmount()).isEqualTo(42.0);
    }

    @Test
    void shouldCalculateMean_WhenMultipleRecordings() {
        DistributionSummary summary = DistributionSummary.builder("test").register(registry);
        summary.record(100.0);
        summary.record(200.0);
        summary.record(300.0);
        assertThat(summary.mean()).isEqualTo(200.0);
    }

    @Test
    void shouldReturnZeroMean_WhenNoRecordings() {
        DistributionSummary summary = DistributionSummary.builder("test").register(registry);
        assertThat(summary.mean()).isEqualTo(0.0);
    }

    @Test
    void shouldTrackMax_WhenMultipleRecordings() {
        DistributionSummary summary = DistributionSummary.builder("test").register(registry);
        summary.record(100.0);
        summary.record(500.0);
        summary.record(200.0);
        assertThat(summary.max()).isEqualTo(500.0);
    }

    @Test
    void shouldProduceThreeMeasurements_WhenMeasured() {
        DistributionSummary summary = DistributionSummary.builder("test").register(registry);
        summary.record(100.0);
        summary.record(300.0);
        var measurementList = new java.util.ArrayList<Measurement>();
        summary.measure().forEach(measurementList::add);
        assertThat(measurementList).hasSize(3);
        assertThat(measurementList.get(0).getStatistic()).isEqualTo(Statistic.COUNT);
        assertThat(measurementList.get(0).getValue()).isEqualTo(2.0);
        assertThat(measurementList.get(1).getStatistic()).isEqualTo(Statistic.TOTAL);
        assertThat(measurementList.get(1).getValue()).isEqualTo(400.0);
        assertThat(measurementList.get(2).getStatistic()).isEqualTo(Statistic.MAX);
        assertThat(measurementList.get(2).getValue()).isEqualTo(300.0);
    }

    @Test
    void shouldHaveDistributionSummaryType_WhenRegistered() {
        DistributionSummary summary = DistributionSummary.builder("test").register(registry);
        assertThat(summary.getId().getType()).isEqualTo(Meter.Type.DISTRIBUTION_SUMMARY);
    }
}
```

#### `[NEW]` `src/test/java/dev/linhvu/micrometer/AbstractDistributionSummaryTest.java`

```java
package dev.linhvu.micrometer;

import org.junit.jupiter.api.Test;

import java.util.ArrayList;
import java.util.List;

import static org.assertj.core.api.Assertions.*;

class AbstractDistributionSummaryTest {

    static class TestDistributionSummary extends AbstractDistributionSummary {
        final List<Double> recorded = new ArrayList<>();

        TestDistributionSummary(double scale) {
            super(id("test.summary"), scale);
        }

        @Override
        protected void recordNonNegative(double amount) { recorded.add(amount); }
        @Override
        public long count() { return recorded.size(); }
        @Override
        public double totalAmount() { return recorded.stream().mapToDouble(d -> d).sum(); }
        @Override
        public double max() { return recorded.stream().mapToDouble(d -> d).max().orElse(0); }

        private static Meter.Id id(String name) {
            return new Meter.Id(name, Tags.empty(), Meter.Type.DISTRIBUTION_SUMMARY, null, null);
        }
    }

    @Test
    void shouldDelegateToRecordNonNegative_WhenAmountIsPositive() {
        var summary = new TestDistributionSummary(1.0);
        summary.record(42.0);
        assertThat(summary.recorded).containsExactly(42.0);
    }

    @Test
    void shouldDropNegativeValues_WhenRecordCalled() {
        var summary = new TestDistributionSummary(1.0);
        summary.record(-1.0);
        assertThat(summary.recorded).isEmpty();
    }

    @Test
    void shouldAcceptZero_WhenRecordCalled() {
        var summary = new TestDistributionSummary(1.0);
        summary.record(0.0);
        assertThat(summary.recorded).containsExactly(0.0);
    }

    @Test
    void shouldApplyScale_WhenScaleIsNotOne() {
        var summary = new TestDistributionSummary(0.001);
        summary.record(2000.0);
        assertThat(summary.recorded).containsExactly(2.0);
    }

    @Test
    void shouldApplyScaleBeforeDelegate_WhenNegativeAfterScale() {
        var summary = new TestDistributionSummary(2.0);
        summary.record(-5.0);
        assertThat(summary.recorded).isEmpty();
    }

    @Test
    void shouldBeFinal_WhenRecordCalled() {
        var summary = new TestDistributionSummary(3.0);
        summary.record(10.0);
        summary.record(20.0);
        summary.record(-1.0);
        assertThat(summary.recorded).containsExactly(30.0, 60.0);
    }
}
```

#### `[NEW]` `src/test/java/dev/linhvu/micrometer/cumulative/CumulativeDistributionSummaryTest.java`

```java
package dev.linhvu.micrometer.cumulative;

import dev.linhvu.micrometer.Measurement;
import dev.linhvu.micrometer.Meter;
import dev.linhvu.micrometer.MockClock;
import dev.linhvu.micrometer.Statistic;
import dev.linhvu.micrometer.Tags;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.util.ArrayList;
import java.util.List;
import java.util.concurrent.CountDownLatch;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.TimeUnit;

import static org.assertj.core.api.Assertions.*;

class CumulativeDistributionSummaryTest {

    private MockClock clock;

    @BeforeEach
    void setUp() { clock = new MockClock(); }

    private CumulativeDistributionSummary createSummary() { return createSummary(1.0); }

    private CumulativeDistributionSummary createSummary(double scale) {
        return new CumulativeDistributionSummary(id("test"), clock, scale);
    }

    private static Meter.Id id(String name) {
        return new Meter.Id(name, Tags.empty(), Meter.Type.DISTRIBUTION_SUMMARY, null, null);
    }

    @Test
    void shouldTrackCountAndTotal_WhenMultipleRecordings() {
        var summary = createSummary();
        summary.record(100.0);
        summary.record(200.0);
        summary.record(50.0);
        assertThat(summary.count()).isEqualTo(3);
        assertThat(summary.totalAmount()).isEqualTo(350.0);
    }

    @Test
    void shouldTrackDecayingMax_WhenRecorded() {
        var summary = createSummary();
        summary.record(100.0);
        summary.record(500.0);
        summary.record(200.0);
        assertThat(summary.max()).isEqualTo(500.0);
    }

    @Test
    void shouldDecayMax_WhenTimeWindowExpires() {
        var summary = createSummary();
        summary.record(999.0);
        clock.add(121, TimeUnit.SECONDS);
        assertThat(summary.max()).isEqualTo(0.0);
    }

    @Test
    void shouldHandleFractionalValues_WhenRecorded() {
        var summary = createSummary();
        summary.record(1.5);
        summary.record(2.7);
        assertThat(summary.count()).isEqualTo(2);
        assertThat(summary.totalAmount()).isCloseTo(4.2, within(0.001));
        assertThat(summary.max()).isEqualTo(2.7);
    }

    @Test
    void shouldApplyScaleFactor_WhenScaleIsNotOne() {
        var summary = createSummary(1.0 / 1024);
        summary.record(2048.0);
        assertThat(summary.count()).isEqualTo(1);
        assertThat(summary.totalAmount()).isEqualTo(2.0);
        assertThat(summary.max()).isEqualTo(2.0);
    }

    @Test
    void shouldReturnZeros_WhenNoRecordings() {
        var summary = createSummary();
        assertThat(summary.count()).isEqualTo(0);
        assertThat(summary.totalAmount()).isEqualTo(0.0);
        assertThat(summary.max()).isEqualTo(0.0);
    }

    @Test
    void shouldProduceCorrectMeasurements_WhenMeasureCalled() {
        var summary = createSummary();
        summary.record(100.0);
        summary.record(300.0);
        List<Measurement> measurements = new ArrayList<>();
        summary.measure().forEach(measurements::add);
        assertThat(measurements).hasSize(3);
        assertThat(measurements.get(0).getStatistic()).isEqualTo(Statistic.COUNT);
        assertThat(measurements.get(0).getValue()).isEqualTo(2.0);
        assertThat(measurements.get(1).getStatistic()).isEqualTo(Statistic.TOTAL);
        assertThat(measurements.get(1).getValue()).isEqualTo(400.0);
        assertThat(measurements.get(2).getStatistic()).isEqualTo(Statistic.MAX);
        assertThat(measurements.get(2).getValue()).isEqualTo(300.0);
    }

    @Test
    void shouldBeThreadSafe_WhenRecordedConcurrently() throws Exception {
        var summary = createSummary();
        int threads = 8, recordsPerThread = 1000;
        ExecutorService executor = Executors.newFixedThreadPool(threads);
        CountDownLatch latch = new CountDownLatch(threads);
        for (int t = 0; t < threads; t++) {
            executor.submit(() -> {
                try { for (int i = 0; i < recordsPerThread; i++) summary.record(1.0); }
                finally { latch.countDown(); }
            });
        }
        latch.await(5, TimeUnit.SECONDS);
        executor.shutdown();
        assertThat(summary.count()).isEqualTo(8000L);
        assertThat(summary.totalAmount()).isEqualTo(8000.0);
    }
}
```

#### `[NEW]` `src/test/java/dev/linhvu/micrometer/noop/NoopDistributionSummaryTest.java`

```java
package dev.linhvu.micrometer.noop;

import dev.linhvu.micrometer.Meter;
import dev.linhvu.micrometer.Tags;
import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.*;

class NoopDistributionSummaryTest {

    private static Meter.Id id(String name) {
        return new Meter.Id(name, Tags.empty(), Meter.Type.DISTRIBUTION_SUMMARY, null, null);
    }

    @Test
    void shouldAcceptRecordings_WhenCalledOnNoop() {
        var noop = new NoopDistributionSummary(id("test"));
        noop.record(100.0);
        noop.record(200.0);
    }

    @Test
    void shouldReturnZeroCount_WhenQueried() {
        var noop = new NoopDistributionSummary(id("test"));
        noop.record(100.0);
        assertThat(noop.count()).isEqualTo(0);
    }

    @Test
    void shouldReturnZeroTotalAmount_WhenQueried() {
        var noop = new NoopDistributionSummary(id("test"));
        noop.record(100.0);
        assertThat(noop.totalAmount()).isEqualTo(0.0);
    }

    @Test
    void shouldReturnZeroMax_WhenQueried() {
        var noop = new NoopDistributionSummary(id("test"));
        noop.record(100.0);
        assertThat(noop.max()).isEqualTo(0.0);
    }

    @Test
    void shouldReturnZeroMean_WhenQueried() {
        var noop = new NoopDistributionSummary(id("test"));
        noop.record(100.0);
        assertThat(noop.mean()).isEqualTo(0.0);
    }
}
```

#### `[NEW]` `src/test/java/dev/linhvu/micrometer/integration/DistributionSummaryIntegrationTest.java`

```java
package dev.linhvu.micrometer.integration;

import dev.linhvu.micrometer.Counter;
import dev.linhvu.micrometer.DistributionSummary;
import dev.linhvu.micrometer.Gauge;
import dev.linhvu.micrometer.Meter;
import dev.linhvu.micrometer.MockClock;
import dev.linhvu.micrometer.Timer;
import dev.linhvu.micrometer.noop.NoopDistributionSummary;
import dev.linhvu.micrometer.simple.SimpleMeterRegistry;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.util.Collections;
import java.util.concurrent.TimeUnit;

import static org.assertj.core.api.Assertions.*;

class DistributionSummaryIntegrationTest {

    private MockClock clock;
    private SimpleMeterRegistry registry;

    @BeforeEach
    void setUp() {
        clock = new MockClock();
        registry = new SimpleMeterRegistry(clock);
    }

    @Test
    void shouldCoexistWithOtherMeterTypes_WhenRegisteredOnSameRegistry() {
        Counter counter = registry.counter("events");
        Timer timer = registry.timer("latency");
        registry.gauge("queue.size", Collections.emptyList(), 42.0, d -> d);
        DistributionSummary summary = registry.summary("response.size");
        counter.increment();
        timer.record(100, TimeUnit.MILLISECONDS);
        summary.record(1024.0);
        assertThat(registry.getMeters()).hasSize(4);
    }

    @Test
    void shouldRegisterViaConvenienceMethod_WhenCalledOnRegistry() {
        DistributionSummary summary = registry.summary("http.response.size", "method", "GET", "status", "200");
        summary.record(512.0);
        assertThat(summary.count()).isEqualTo(1);
        assertThat(summary.getId().getTag("method")).isEqualTo("GET");
    }

    @Test
    void shouldDeduplicateSummaries_WhenRegisteredMultipleTimes() {
        DistributionSummary s1 = registry.summary("payload.size", "endpoint", "/api");
        DistributionSummary s2 = registry.summary("payload.size", "endpoint", "/api");
        s1.record(100.0);
        s2.record(200.0);
        assertThat(s1).isSameAs(s2);
        assertThat(s1.count()).isEqualTo(2);
    }

    @Test
    void shouldReturnNoopSummary_WhenRegistryIsClosed() {
        registry.close();
        DistributionSummary summary = registry.summary("late.summary");
        assertThat(summary).isInstanceOf(NoopDistributionSummary.class);
    }

    @Test
    void shouldAppearInMeterVisitor_WhenMatchCalled() {
        DistributionSummary summary = registry.summary("test.summary");
        summary.record(42.0);
        String result = summary.match(
                c -> "counter", g -> "gauge", t -> "timer",
                ds -> "summary:" + ds.count(),
                ltt -> "longTaskTimer", fc -> "functionCounter",
                ft -> "functionTimer", m -> "other");
        assertThat(result).isEqualTo("summary:1");
    }

    @Test
    void shouldThrowOnTypeConflict_WhenSummaryNameMatchesExistingCounter() {
        registry.counter("shared");
        assertThatThrownBy(() -> registry.summary("shared"))
                .isInstanceOf(IllegalArgumentException.class);
    }

    @Test
    void shouldThrowOnTypeConflict_WhenSummaryNameMatchesExistingTimer() {
        registry.timer("shared");
        assertThatThrownBy(() -> registry.summary("shared"))
                .isInstanceOf(IllegalArgumentException.class);
    }

    @Test
    void shouldSupportScaleViaBuilder_WhenUsedEndToEnd() {
        DistributionSummary summary = DistributionSummary.builder("payload.size")
                .baseUnit("kilobytes").scale(1.0 / 1024)
                .tag("endpoint", "/upload").register(registry);
        summary.record(4096.0);
        assertThat(summary.totalAmount()).isEqualTo(4.0);
        assertThat(summary.max()).isEqualTo(4.0);
        assertThat(summary.getId().getBaseUnit()).isEqualTo("kilobytes");
    }

    @Test
    void shouldDifferentiateBySummaryTags_WhenSameNameDifferentTags() {
        DistributionSummary s1 = registry.summary("response.size", "status", "200");
        DistributionSummary s2 = registry.summary("response.size", "status", "500");
        s1.record(1000.0);
        s2.record(50.0);
        assertThat(s1).isNotSameAs(s2);
        assertThat(s1.totalAmount()).isEqualTo(1000.0);
        assertThat(s2.totalAmount()).isEqualTo(50.0);
    }
}
```

---

## Summary

| What | File | Lines |
|------|------|-------|
| DistributionSummary interface | `DistributionSummary.java` | 157 |
| Abstract template | `AbstractDistributionSummary.java` | 47 |
| Cumulative impl | `CumulativeDistributionSummary.java` | 76 |
| Noop fallback | `NoopDistributionSummary.java` | 37 |
| Registry wiring | `MeterRegistry.java` (modified) | +32 lines |
| SimpleMeterRegistry | `SimpleMeterRegistry.java` (modified) | +3 lines |
| Tests | 5 test classes | 42 tests |

**Next chapter:** [Chapter 8: MeterFilter](ch08_meter_filter.md) — a filter chain that transforms, accepts/denies, and configures meters during registration. Filters can rename metrics, add common tags, deny high-cardinality meters, and set histogram boundaries — all without changing application code.
