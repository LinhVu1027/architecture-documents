# Chapter 4: Distribution Summary — Value Distributions

> **Feature 4** · Tier 2 · Depends on: Feature 1 (Counter & MeterRegistry), Feature 3 (Timer — reuses TimeWindowMax)

After this chapter you will have a meter that tracks the **distribution of arbitrary values** —
response payload sizes, batch sizes, queue depths — with count, total, max, and mean statistics.
Think of it as a Timer for non-time quantities.

---

## 1. The API Contract

Before writing any implementation, let's define exactly what a client sees.

### What the client imports

```java
import simple.micrometer.api.DistributionSummary;
import simple.micrometer.api.MeterRegistry;
import simple.micrometer.internal.SimpleMeterRegistry;
```

### What the client writes

```java
// 1. Create a registry (reusing from Feature 1)
MeterRegistry registry = new SimpleMeterRegistry();

// 2. Create and use a distribution summary
DistributionSummary summary = DistributionSummary.builder("http.response.size")
    .tag("uri", "/api/users")
    .description("Response payload sizes")
    .baseUnit("bytes")
    .register(registry);

// 3. Record values
summary.record(1024);
summary.record(2048);

// 4. Read statistics
long count = summary.count();       // 2
double total = summary.totalAmount(); // 3072.0
double max = summary.max();          // 2048.0
double mean = summary.mean();        // 1536.0

// 5. Registry shorthand
DistributionSummary shorthand = registry.summary("payload.size", "uri", "/api");
```

### Behavioral contract

| Promise | Detail |
|---------|--------|
| Push-based recording | Client calls `record(double)` and the summary stores count/total/max internally |
| Three statistics | `count()`, `totalAmount()`, and `max()` — raw values, no unit conversion |
| Computed mean | `mean()` = `totalAmount() / count()`, computed on-the-fly |
| Decaying max | `max()` returns the highest recent value, not the all-time peak; decays to 0 over ~2 minutes |
| Negative-safe | Negative amounts are silently ignored |
| Zero-valid | Recording 0 is valid — it increments count |
| Deduplicated | Same name + tags returns the same summary instance |
| Three measurements | `measure()` returns COUNT, TOTAL, MAX |

---

## 2. Client Usage & Tests

These tests define the API's behavior — they were written **before** any implementation.

### Builder + Registration

```java
@Test
void shouldCreateSummary_WithBuilderAndRegister() {
    DistributionSummary summary = DistributionSummary.builder("http.response.size")
            .tag("uri", "/api/users")
            .description("Response payload sizes")
            .baseUnit("bytes")
            .register(registry);

    assertThat(summary).isNotNull();
    assertThat(summary.getId().getName()).isEqualTo("http.response.size");
    assertThat(summary.getId().getTag("uri")).isEqualTo("/api/users");
    assertThat(summary.getId().getDescription()).isEqualTo("Response payload sizes");
    assertThat(summary.getId().getBaseUnit()).isEqualTo("bytes");
    assertThat(summary.getId().getType()).isEqualTo(Meter.Type.DISTRIBUTION_SUMMARY);
}
```

### Recording and reading

```java
@Test
void shouldRecordValues() {
    DistributionSummary summary = DistributionSummary.builder("payload.size")
            .register(registry);

    summary.record(1024);
    summary.record(2048);

    assertThat(summary.count()).isEqualTo(2);
    assertThat(summary.totalAmount()).isCloseTo(3072.0, within(0.01));
    assertThat(summary.max()).isCloseTo(2048.0, within(0.01));
}
```

### Negative values are ignored

```java
@Test
void shouldIgnoreNegativeValues() {
    DistributionSummary summary = DistributionSummary.builder("payload.size")
            .register(registry);

    summary.record(-100);

    assertThat(summary.count()).isEqualTo(0);
    assertThat(summary.totalAmount()).isEqualTo(0.0);
}
```

### Mean is computed on-the-fly

```java
@Test
void shouldComputeMean() {
    DistributionSummary summary = DistributionSummary.builder("payload.size")
            .register(registry);

    summary.record(1024);
    summary.record(2048);

    assertThat(summary.mean()).isCloseTo(1536.0, within(0.01));
}
```

### Max tracks the highest value, not the last

```java
@Test
void shouldTrackMaxAcrossRecordings() {
    DistributionSummary summary = DistributionSummary.builder("payload.size")
            .register(registry);

    summary.record(100);
    summary.record(500);
    summary.record(200);

    // Max should be 500, not the last recorded value
    assertThat(summary.max()).isCloseTo(500.0, within(0.01));
}
```

### Coexistence with all other meter types

```java
@Test
void shouldCoexist_WithCounterGaugeAndTimer() {
    Counter counter = Counter.builder("requests").register(registry);
    Gauge gauge = Gauge.builder("queue.size", new ArrayList<>(), List::size).register(registry);
    Timer timer = Timer.builder("latency").register(registry);
    DistributionSummary summary = DistributionSummary.builder("payload.size").register(registry);

    counter.increment();
    timer.record(Duration.ofMillis(100));
    summary.record(1024);

    assertThat(registry.getMeters()).hasSize(4);
    assertThat(summary.count()).isEqualTo(1);
    assertThat(summary.totalAmount()).isCloseTo(1024.0, within(0.01));
}
```

---

## 3. Implementing the Call Chain

### Layer 1: DistributionSummary interface — the API surface

The interface follows the same pattern as `Counter` and `Timer`: abstract recording/reading
methods, default `mean()` and `measure()`, a static `builder()` factory, and a nested `Builder`.

```java
public interface DistributionSummary extends Meter {

    void record(double amount);

    long count();
    double totalAmount();
    double max();

    default double mean() {
        long n = count();
        return n == 0 ? 0 : totalAmount() / n;
    }

    default Iterable<Measurement> measure() {
        return Arrays.asList(
                new Measurement(this::count, Statistic.COUNT),
                new Measurement(this::totalAmount, Statistic.TOTAL),
                new Measurement(this::max, Statistic.MAX)
        );
    }
}
```

**Key difference from Timer**: `measure()` uses `Statistic.TOTAL` (not `TOTAL_TIME`) and
`max()` returns a raw double (no `TimeUnit` parameter). There are no code-timing convenience
methods like `record(Runnable)` or `Timer.Sample` — those only make sense for time-based meters.

### Layer 2: The Builder — identical pattern

```java
class Builder {
    private final String name;
    private Tags tags = Tags.empty();
    private String description;
    private String baseUnit;

    public DistributionSummary register(MeterRegistry registry) {
        Meter.Id id = new Meter.Id(name, tags, baseUnit, description,
                Meter.Type.DISTRIBUTION_SUMMARY);
        return registry.summary(id);
    }
}
```

The `baseUnit` field is particularly useful for distribution summaries — it records
what unit the values are in (bytes, items, etc.) so monitoring backends can label
the axis correctly.

### Layer 3: MeterRegistry — extending the shared registration infrastructure

We add the same three pieces we added for Timer:

**Convenience methods** (public):
```java
public DistributionSummary summary(String name, String... tags) {
    return summary(name, Tags.of(tags));
}

public DistributionSummary summary(String name, Iterable<Tag> tags) {
    Meter.Id id = new Meter.Id(name, Tags.of(tags), null, null,
            Meter.Type.DISTRIBUTION_SUMMARY);
    return summary(id);
}
```

**Registration entry point** (package-private):
```java
DistributionSummary summary(Meter.Id id) {
    return registerMeterIfNecessary(DistributionSummary.class, id,
            this::newDistributionSummary);
}
```

**Abstract factory method**:
```java
protected abstract DistributionSummary newDistributionSummary(Meter.Id id);
```

This is the fourth time we've extended MeterRegistry with this pattern. By now the
mechanism should feel familiar: convenience → entry point → abstract factory → concrete
implementation. The `registerMeterIfNecessary` method handles deduplication for all meter
types uniformly.

### Layer 4: CumulativeDistributionSummary — count/total/max tracking

This is structurally very similar to `CumulativeTimer`, with two key differences:

1. **`DoubleAdder` for total** instead of `LongAdder` — because values are arbitrary doubles,
   not nanosecond longs
2. **No time unit conversion** — `record(double)` stores the raw value, `totalAmount()` and
   `max()` return raw values

```java
public class CumulativeDistributionSummary implements DistributionSummary {

    private final Meter.Id id;
    private final LongAdder count = new LongAdder();
    private final DoubleAdder total = new DoubleAdder();
    private final TimeWindowMax max;

    public CumulativeDistributionSummary(Meter.Id id, Clock clock) {
        this.id = id;
        this.max = new TimeWindowMax(clock);
    }

    @Override
    public void record(double amount) {
        if (amount >= 0) {
            count.increment();
            total.add(amount);
            max.record(amount);
        }
    }

    @Override
    public long count() { return count.longValue(); }

    @Override
    public double totalAmount() { return total.sum(); }

    @Override
    public double max() { return max.poll(); }
}
```

### Layer 4b: TimeWindowMax — extending for raw doubles

The `TimeWindowMax` from Feature 3 stores nanosecond longs via `record(long)` / `poll(TimeUnit)`.
For distribution summaries, we need to store arbitrary doubles. We add a second API pair:

```java
public void record(double amount) {
    rotate();
    long bits = Double.doubleToLongBits(amount);
    for (AtomicLong slot : ringBuffer) {
        updateMax(slot, bits);
    }
}

public double poll() {
    rotate();
    long bits = ringBuffer[currentBucket].get();
    return Double.longBitsToDouble(bits);
}
```

**The IEEE 754 trick**: `Double.doubleToLongBits()` converts a double to its 64-bit IEEE 754
representation as a long. For **positive** doubles (which is all we store, since negatives are
filtered out), the bit ordering preserves comparison semantics: if `a > b`, then
`doubleToLongBits(a) > doubleToLongBits(b)`. This means the existing CAS-based `updateMax`
comparison works correctly without any changes.

The initial ring buffer value of 0L corresponds to `Double.longBitsToDouble(0L) = 0.0`, which
is the correct initial max.

### Layer 5: SimpleMeterRegistry — one line

```java
@Override
protected DistributionSummary newDistributionSummary(Meter.Id id) {
    return new CumulativeDistributionSummary(id, clock);
}
```

### Layer 6: NoopDistributionSummary — the null-object

```java
public class NoopDistributionSummary implements DistributionSummary {
    private final Meter.Id id;

    public NoopDistributionSummary(Meter.Id id) { this.id = id; }

    @Override public void record(double amount) { }
    @Override public long count() { return 0; }
    @Override public double totalAmount() { return 0; }
    @Override public double max() { return 0; }
    @Override public Meter.Id getId() { return id; }
}
```

---

## 4. Try It Yourself

1. **Add `DistributionSummary.builder(...).scale(double)`** — a multiplier applied to all
   recorded values before storing. The real Micrometer has this. Hint: multiply `amount` by
   `scale` in `record()` before passing to count/total/max.

2. **What happens if you call `summary.record(Double.MAX_VALUE)` twice?** The `DoubleAdder`
   total would overflow to `Infinity`. How does the real Micrometer handle this?
   (It doesn't — overflow protection is the caller's responsibility.)

3. **Verify the IEEE 754 ordering trick**: write a test that records several doubles,
   converts each to `doubleToLongBits`, and asserts that the long comparison matches
   the double comparison. Does it work for negative doubles? (No — and that's why
   `record()` filters them out.)

---

## 5. Why This Works

### DoubleAdder for total, LongAdder for count

Timer uses `LongAdder` for both count and total (nanosecond longs). DistributionSummary
uses `DoubleAdder` for total because values can be fractional (1.5 bytes per request,
0.7 items per batch). Internally, `DoubleAdder` stores values as long bits via
`Double.doubleToRawLongBits` and uses the same striped accumulation strategy as `LongAdder`
— so the performance characteristics are identical.

### No unit conversion

Timers need `totalTime(TimeUnit)` and `max(TimeUnit)` because the caller wants to choose
their preferred time unit at read time. Distribution summaries don't need this — the values
are unitless doubles. The `baseUnit` string is purely metadata for monitoring backends to
label their axes correctly.

### Same TimeWindowMax, different storage format

Rather than creating a separate `TimeWindowMaxDouble` class, we added `record(double)` /
`poll()` methods to the existing `TimeWindowMax`. This is the Reuse Rule in action — Feature 4
extends an internal from Feature 3 rather than duplicating it. The key insight is that
IEEE 754 bit representation preserves ordering for positive doubles, so the same `updateMax`
CAS loop works for both formats.

### Mean is not stored

`mean()` is computed as `totalAmount() / count()` each time it's called. This avoids the
complexity of maintaining a running average (which has floating-point precision issues) and
ensures the mean is always consistent with the current count and total.

---

## 6. What We Enhanced

| Component | State before (Feature 3) | Enhancement in Feature 4 | Will be enhanced by |
|-----------|-------------------------|--------------------------|---------------------|
| `MeterRegistry` | Supports Counter + Gauge + Timer | +DistributionSummary registration, +2 convenience methods, +`newDistributionSummary()` abstract | F5: +MeterFilter |
| `SimpleMeterRegistry` | Creates CumulativeCounter + DefaultGauge + CumulativeTimer | +`newDistributionSummary()` → CumulativeDistributionSummary | Stable for basic meters |
| `TimeWindowMax` | Only `record(long)` / `poll(TimeUnit)` for nanosecond values | +`record(double)` / `poll()` for raw double values via IEEE 754 bit encoding | Stable |
| `Meter.Type` | Had `DISTRIBUTION_SUMMARY` enum value (unused) | Now used in DistributionSummary's Meter.Id | Stable |

---

## 7. Connection to Real Framework

| Simplified Class | Real Framework Class | Key Simplification |
|-----------------|---------------------|--------------------|
| `api/DistributionSummary` | `io.micrometer.core.instrument.DistributionSummary` | No `HistogramSupport` interface (no percentiles/histograms/SLOs), no `scale` in interface, no `withRegistry()` for dynamic tagging |
| `api/DistributionSummary.Builder` | `io.micrometer.core.instrument.DistributionSummary.Builder` | No `DistributionStatisticConfig` (percentiles, histogram boundaries, SLOs, expiry), no `scale(double)`, no `publishPercentiles()`, no `serviceLevelObjectives()` |
| `internal/CumulativeDistributionSummary` | `io.micrometer.core.instrument.cumulative.CumulativeDistributionSummary` | No `AbstractDistributionSummary` base class, no histogram recording via `Histogram`, no `scale` factor. Uses `DoubleAdder` for total (real uses `DoubleAdder` too) |
| `internal/NoopDistributionSummary` | `io.micrometer.core.instrument.noop.NoopDistributionSummary` | No `NoopMeter` base class, no `HistogramSnapshot` |
| `internal/TimeWindowMax` (extended) | `io.micrometer.core.instrument.distribution.TimeWindowMax` | Real version always stores doubles as long bits (even for timers). Our version has dual APIs: `record(long)/poll(TimeUnit)` for timers and `record(double)/poll()` for summaries |

---

## 8. Complete Code

All files created or modified in this feature. Copy all `[NEW]` files and replace
all `[MODIFIED]` files to build the project from Feature 1 + Feature 2 + Feature 3 + Feature 4.

### `src/main/java/simple/micrometer/api/DistributionSummary.java` [NEW]

```java
package simple.micrometer.api;

import java.util.Arrays;

/**
 * A meter that tracks the distribution of values — response sizes, batch sizes,
 * queue depths, or any non-time quantity. Like a {@link Timer} but for arbitrary
 * values instead of durations: tracks count, total, max, and computed mean.
 *
 * <p>Usage:
 * <pre>{@code
 * DistributionSummary summary = DistributionSummary.builder("http.response.size")
 *     .tag("uri", "/api/users")
 *     .description("Response payload sizes")
 *     .baseUnit("bytes")
 *     .register(registry);
 *
 * summary.record(1024);
 * summary.record(2048);
 *
 * long count = summary.count();       // 2
 * double total = summary.totalAmount(); // 3072.0
 * double max = summary.max();          // 2048.0
 * double mean = summary.mean();        // 1536.0
 * }</pre>
 *
 * <p>Simplified from {@code io.micrometer.core.instrument.DistributionSummary}.
 */
public interface DistributionSummary extends Meter {

    // ── Recording ─────────────────────────────────────────────────────

    /**
     * Records a value. Negative values are silently ignored.
     */
    void record(double amount);

    // ── Reading ───────────────────────────────────────────────────────

    /**
     * Returns the number of times {@code record} has been called with
     * a non-negative value.
     */
    long count();

    /**
     * Returns the cumulative total of all recorded values.
     */
    double totalAmount();

    /**
     * Returns the maximum recorded value within the recent time window.
     * The max decays over time — if no new values are recorded, it
     * gradually returns to 0.
     */
    double max();

    /**
     * Returns the mean value (totalAmount / count), or 0 if no values
     * have been recorded.
     */
    default double mean() {
        long n = count();
        return n == 0 ? 0 : totalAmount() / n;
    }

    // ── Measurement ───────────────────────────────────────────────────

    /**
     * Returns three measurements: COUNT, TOTAL, and MAX.
     * Backends use these to compute rates, averages, and peaks.
     */
    @Override
    default Iterable<Measurement> measure() {
        return Arrays.asList(
                new Measurement(this::count, Statistic.COUNT),
                new Measurement(this::totalAmount, Statistic.TOTAL),
                new Measurement(this::max, Statistic.MAX)
        );
    }

    // ── Factory ───────────────────────────────────────────────────────

    /**
     * Creates a new DistributionSummary builder.
     */
    static Builder builder(String name) {
        return new Builder(name);
    }

    // ── Builder ───────────────────────────────────────────────────────

    /**
     * Fluent builder for constructing and registering distribution summaries.
     * Follows the same pattern as {@link Counter.Builder} and
     * {@link Timer.Builder}: accumulates metadata, then delegates to the
     * registry for concrete type creation.
     *
     * <p>Simplified from {@code io.micrometer.core.instrument.DistributionSummary.Builder}.
     * The real builder also supports {@code DistributionStatisticConfig}
     * (percentiles, histograms, SLOs) and a {@code scale} factor.
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

        public Builder tags(Iterable<Tag> tags) {
            this.tags = this.tags.and(tags);
            return this;
        }

        public Builder tag(String key, String value) {
            this.tags = this.tags.and(key, value);
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

        /**
         * Terminal operation: registers (or retrieves) a distribution summary
         * with the given registry. The registry decides which concrete
         * DistributionSummary to create.
         */
        public DistributionSummary register(MeterRegistry registry) {
            Meter.Id id = new Meter.Id(name, tags, baseUnit, description, Meter.Type.DISTRIBUTION_SUMMARY);
            return registry.summary(id);
        }

    }

}
```

### `src/main/java/simple/micrometer/internal/CumulativeDistributionSummary.java` [NEW]

```java
package simple.micrometer.internal;

import simple.micrometer.api.Clock;
import simple.micrometer.api.DistributionSummary;
import simple.micrometer.api.Meter;

import java.util.concurrent.atomic.DoubleAdder;
import java.util.concurrent.atomic.LongAdder;

/**
 * A distribution summary that tracks count, total, and decaying max using
 * lock-free accumulators. All values are stored as raw doubles — no time
 * unit conversion (unlike {@link CumulativeTimer}).
 *
 * <p>Thread-safe: {@link LongAdder} and {@link DoubleAdder} provide striped
 * accumulation for high-throughput concurrent recording without contention.
 * {@link TimeWindowMax} uses a CAS-based ring buffer for the decaying max.
 *
 * <p>Simplified from {@code io.micrometer.core.instrument.cumulative.CumulativeDistributionSummary}.
 * The real version extends {@code AbstractDistributionSummary} which adds
 * histogram support and a scale factor. Our version implements
 * {@link DistributionSummary} directly.
 */
public class CumulativeDistributionSummary implements DistributionSummary {

    private final Meter.Id id;
    private final LongAdder count = new LongAdder();
    private final DoubleAdder total = new DoubleAdder();
    private final TimeWindowMax max;

    public CumulativeDistributionSummary(Meter.Id id, Clock clock) {
        this.id = id;
        this.max = new TimeWindowMax(clock);
    }

    // ── Recording ─────────────────────────────────────────────────────

    @Override
    public void record(double amount) {
        if (amount >= 0) {
            count.increment();
            total.add(amount);
            max.record(amount);
        }
    }

    // ── Reading ───────────────────────────────────────────────────────

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

    // ── Meter ─────────────────────────────────────────────────────────

    @Override
    public Meter.Id getId() {
        return id;
    }

}
```

### `src/main/java/simple/micrometer/internal/NoopDistributionSummary.java` [NEW]

```java
package simple.micrometer.internal;

import simple.micrometer.api.DistributionSummary;
import simple.micrometer.api.Meter;

/**
 * A no-op DistributionSummary that silently discards all recorded values.
 * Used when a meter is denied by a filter or when the registry is not
 * configured to create real meters.
 *
 * <p>Simplified from {@code io.micrometer.core.instrument.noop.NoopDistributionSummary}.
 */
public class NoopDistributionSummary implements DistributionSummary {

    private final Meter.Id id;

    public NoopDistributionSummary(Meter.Id id) {
        this.id = id;
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

    @Override
    public Meter.Id getId() {
        return id;
    }

}
```

### `src/test/java/simple/micrometer/api/DistributionSummaryTest.java` [NEW]

```java
package simple.micrometer.api;

import simple.micrometer.internal.SimpleMeterRegistry;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.util.concurrent.atomic.AtomicLong;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.within;

class DistributionSummaryTest {

    private MeterRegistry registry;

    @BeforeEach
    void setUp() {
        registry = new SimpleMeterRegistry();
    }

    // ── Builder + Registration ────────────────────────────────────────

    @Test
    void shouldCreateSummary_WithBuilderAndRegister() {
        DistributionSummary summary = DistributionSummary.builder("http.response.size")
                .tag("uri", "/api/users")
                .description("Response payload sizes")
                .baseUnit("bytes")
                .register(registry);

        assertThat(summary).isNotNull();
        assertThat(summary.getId().getName()).isEqualTo("http.response.size");
        assertThat(summary.getId().getTag("uri")).isEqualTo("/api/users");
        assertThat(summary.getId().getDescription()).isEqualTo("Response payload sizes");
        assertThat(summary.getId().getBaseUnit()).isEqualTo("bytes");
        assertThat(summary.getId().getType()).isEqualTo(Meter.Type.DISTRIBUTION_SUMMARY);
    }

    @Test
    void shouldReturnSameSummary_WhenRegisteredTwiceWithSameId() {
        DistributionSummary s1 = DistributionSummary.builder("payload.size")
                .tag("k", "v").register(registry);
        DistributionSummary s2 = DistributionSummary.builder("payload.size")
                .tag("k", "v").register(registry);

        assertThat(s1).isSameAs(s2);
    }

    @Test
    void shouldReturnDifferentSummaries_WhenTagsDiffer() {
        DistributionSummary s1 = DistributionSummary.builder("payload.size")
                .tag("env", "prod").register(registry);
        DistributionSummary s2 = DistributionSummary.builder("payload.size")
                .tag("env", "dev").register(registry);

        assertThat(s1).isNotSameAs(s2);
    }

    @Test
    void shouldCreateSummary_WithRegistryShorthand() {
        DistributionSummary summary = registry.summary("http.response.size",
                "uri", "/api/users");

        assertThat(summary).isNotNull();
        assertThat(summary.getId().getName()).isEqualTo("http.response.size");
        assertThat(summary.getId().getTag("uri")).isEqualTo("/api/users");
    }

    // ── Recording ─────────────────────────────────────────────────────

    @Test
    void shouldRecordValues() {
        DistributionSummary summary = DistributionSummary.builder("payload.size")
                .register(registry);

        summary.record(1024);
        summary.record(2048);

        assertThat(summary.count()).isEqualTo(2);
        assertThat(summary.totalAmount()).isCloseTo(3072.0, within(0.01));
        assertThat(summary.max()).isCloseTo(2048.0, within(0.01));
    }

    @Test
    void shouldAccumulateMultipleRecordings() {
        DistributionSummary summary = DistributionSummary.builder("batch.size")
                .register(registry);

        summary.record(10);
        summary.record(20);
        summary.record(30);

        assertThat(summary.count()).isEqualTo(3);
        assertThat(summary.totalAmount()).isCloseTo(60.0, within(0.01));
        assertThat(summary.max()).isCloseTo(30.0, within(0.01));
    }

    @Test
    void shouldIgnoreNegativeValues() {
        DistributionSummary summary = DistributionSummary.builder("payload.size")
                .register(registry);

        summary.record(-100);

        assertThat(summary.count()).isEqualTo(0);
        assertThat(summary.totalAmount()).isEqualTo(0.0);
    }

    @Test
    void shouldRecordZeroValue() {
        DistributionSummary summary = DistributionSummary.builder("payload.size")
                .register(registry);

        summary.record(0);

        assertThat(summary.count()).isEqualTo(1);
        assertThat(summary.totalAmount()).isEqualTo(0.0);
        assertThat(summary.max()).isEqualTo(0.0);
    }

    @Test
    void shouldRecordFractionalValues() {
        DistributionSummary summary = DistributionSummary.builder("latency.score")
                .register(registry);

        summary.record(1.5);
        summary.record(2.7);

        assertThat(summary.count()).isEqualTo(2);
        assertThat(summary.totalAmount()).isCloseTo(4.2, within(0.01));
        assertThat(summary.max()).isCloseTo(2.7, within(0.01));
    }

    // ── Mean ──────────────────────────────────────────────────────────

    @Test
    void shouldComputeMean() {
        DistributionSummary summary = DistributionSummary.builder("payload.size")
                .register(registry);

        summary.record(1024);
        summary.record(2048);

        assertThat(summary.mean()).isCloseTo(1536.0, within(0.01));
    }

    @Test
    void shouldReturnZeroMean_WhenNoRecordings() {
        DistributionSummary summary = DistributionSummary.builder("payload.size")
                .register(registry);

        assertThat(summary.mean()).isEqualTo(0.0);
    }

    // ── Measurements ──────────────────────────────────────────────────

    @Test
    void shouldProduceThreeMeasurements() {
        DistributionSummary summary = DistributionSummary.builder("payload.size")
                .register(registry);
        summary.record(1024);

        Iterable<Measurement> measurements = summary.measure();

        assertThat(measurements).hasSize(3);

        var iter = measurements.iterator();
        Measurement countM = iter.next();
        Measurement totalM = iter.next();
        Measurement maxM = iter.next();

        assertThat(countM.getStatistic()).isEqualTo(Statistic.COUNT);
        assertThat(countM.getValue()).isEqualTo(1.0);

        assertThat(totalM.getStatistic()).isEqualTo(Statistic.TOTAL);
        assertThat(totalM.getValue()).isCloseTo(1024.0, within(0.01));

        assertThat(maxM.getStatistic()).isEqualTo(Statistic.MAX);
        assertThat(maxM.getValue()).isCloseTo(1024.0, within(0.01));
    }

    // ── Zero state ────────────────────────────────────────────────────

    @Test
    void shouldReturnZeros_BeforeAnyRecording() {
        DistributionSummary summary = DistributionSummary.builder("payload.size")
                .register(registry);

        assertThat(summary.count()).isEqualTo(0);
        assertThat(summary.totalAmount()).isEqualTo(0.0);
        assertThat(summary.max()).isEqualTo(0.0);
        assertThat(summary.mean()).isEqualTo(0.0);
    }

    // ── Coexistence with other meter types ────────────────────────────

    @Test
    void shouldCoexist_WithCounterGaugeAndTimer() {
        Counter counter = Counter.builder("requests").register(registry);
        Gauge gauge = Gauge.builder("queue.size", new java.util.ArrayList<>(), java.util.List::size)
                .register(registry);
        Timer timer = Timer.builder("latency").register(registry);
        DistributionSummary summary = DistributionSummary.builder("payload.size")
                .register(registry);

        counter.increment();
        timer.record(java.time.Duration.ofMillis(100));
        summary.record(1024);

        assertThat(registry.getMeters()).hasSize(4);
        assertThat(counter.count()).isEqualTo(1.0);
        assertThat(gauge.value()).isEqualTo(0.0);
        assertThat(timer.count()).isEqualTo(1);
        assertThat(summary.count()).isEqualTo(1);
        assertThat(summary.totalAmount()).isCloseTo(1024.0, within(0.01));
    }

    // ── Large values ──────────────────────────────────────────────────

    @Test
    void shouldHandleLargeValues() {
        DistributionSummary summary = DistributionSummary.builder("file.size")
                .baseUnit("bytes")
                .register(registry);

        summary.record(1_000_000_000.0); // 1 GB
        summary.record(2_000_000_000.0); // 2 GB

        assertThat(summary.count()).isEqualTo(2);
        assertThat(summary.totalAmount()).isCloseTo(3_000_000_000.0, within(1.0));
        assertThat(summary.max()).isCloseTo(2_000_000_000.0, within(1.0));
    }

    // ── Max tracks the highest value ──────────────────────────────────

    @Test
    void shouldTrackMaxAcrossRecordings() {
        DistributionSummary summary = DistributionSummary.builder("payload.size")
                .register(registry);

        summary.record(100);
        summary.record(500);
        summary.record(200);

        // Max should be 500, not the last recorded value
        assertThat(summary.max()).isCloseTo(500.0, within(0.01));
    }

}
```

### `src/main/java/simple/micrometer/api/MeterRegistry.java` [MODIFIED]

Added DistributionSummary support: 2 public convenience methods, 1 package-private registration
entry point, 1 abstract factory method.

```java
package simple.micrometer.api;

import simple.micrometer.internal.NoopCounter;

import java.util.ArrayList;
import java.util.Collections;
import java.util.List;
import java.util.Objects;
import java.util.concurrent.ConcurrentHashMap;
import java.util.function.Function;
import java.util.function.ToDoubleFunction;

/**
 * The central hub that manages meter lifecycle — creation, deduplication,
 * storage, and retrieval. Concrete subclasses (like {@code SimpleMeterRegistry})
 * decide which implementation classes to instantiate for each meter type.
 *
 * <p>Registration uses double-checked locking: the first lookup is lock-free
 * via a {@link ConcurrentHashMap}, and the synchronized block is only entered
 * when creating a new meter.
 *
 * <p>Simplified from {@code io.micrometer.core.instrument.MeterRegistry}.
 */
public abstract class MeterRegistry {

    private final ConcurrentHashMap<Meter.Id, Meter> meterMap = new ConcurrentHashMap<>();
    private final Object meterMapLock = new Object();

    protected final Clock clock;

    protected MeterRegistry(Clock clock) {
        this.clock = Objects.requireNonNull(clock, "clock must not be null");
    }

    // ── Public counter convenience methods ───────────────────────────

    /**
     * Shorthand: creates or retrieves a counter with the given name and
     * alternating key-value tag strings.
     *
     * <pre>{@code
     * Counter c = registry.counter("cache.misses", "cache", "users");
     * }</pre>
     */
    public Counter counter(String name, String... tags) {
        return counter(name, Tags.of(tags));
    }

    /**
     * Creates or retrieves a counter with the given name and tags.
     */
    public Counter counter(String name, Iterable<Tag> tags) {
        Meter.Id id = new Meter.Id(name, Tags.of(tags), null, null, Meter.Type.COUNTER);
        return counter(id);
    }

    // ── Public timer convenience methods ──────────────────────────────

    /**
     * Shorthand: creates or retrieves a timer with the given name and
     * alternating key-value tag strings.
     *
     * <pre>{@code
     * Timer t = registry.timer("http.latency", "method", "GET");
     * }</pre>
     */
    public Timer timer(String name, String... tags) {
        return timer(name, Tags.of(tags));
    }

    /**
     * Creates or retrieves a timer with the given name and tags.
     */
    public Timer timer(String name, Iterable<Tag> tags) {
        Meter.Id id = new Meter.Id(name, Tags.of(tags), null, null, Meter.Type.TIMER);
        return timer(id);
    }

    // ── Public summary convenience methods ──────────────────────────

    /**
     * Shorthand: creates or retrieves a distribution summary with the given
     * name and alternating key-value tag strings.
     *
     * <pre>{@code
     * DistributionSummary s = registry.summary("payload.size", "uri", "/api");
     * }</pre>
     */
    public DistributionSummary summary(String name, String... tags) {
        return summary(name, Tags.of(tags));
    }

    /**
     * Creates or retrieves a distribution summary with the given name and tags.
     */
    public DistributionSummary summary(String name, Iterable<Tag> tags) {
        Meter.Id id = new Meter.Id(name, Tags.of(tags), null, null, Meter.Type.DISTRIBUTION_SUMMARY);
        return summary(id);
    }

    // ── Public gauge convenience methods ─────────────────────────────

    /**
     * Creates or retrieves a gauge that observes {@code stateObject} through
     * {@code valueFunction}. Returns the state object (not the Gauge) for
     * fluent assignment patterns:
     *
     * <pre>{@code
     * AtomicInteger active = registry.gauge("connections", Tags.empty(),
     *     new AtomicInteger(0), AtomicInteger::doubleValue);
     * }</pre>
     *
     * @return the {@code stateObject} passed in (for assignment convenience)
     */
    public <T> T gauge(String name, Iterable<Tag> tags, T stateObject, ToDoubleFunction<T> valueFunction) {
        Gauge.builder(name, stateObject, valueFunction).tags(tags).register(this);
        return stateObject;
    }

    /**
     * Shorthand for gauging a {@link Number} subclass (AtomicInteger, AtomicLong, etc.)
     * using its {@link Number#doubleValue()}.
     */
    public <T extends Number> T gauge(String name, Iterable<Tag> tags, T number) {
        return gauge(name, tags, number, Number::doubleValue);
    }

    /**
     * Shorthand: gauge a Number without tags.
     */
    public <T extends Number> T gauge(String name, T number) {
        return gauge(name, Collections.emptyList(), number);
    }

    /**
     * Shorthand: gauge an object without tags.
     */
    public <T> T gauge(String name, T stateObject, ToDoubleFunction<T> valueFunction) {
        return gauge(name, Collections.emptyList(), stateObject, valueFunction);
    }

    // ── Package-private registration entry points ─────────────────────

    /**
     * Called by {@link Counter.Builder#register} and the convenience methods above.
     */
    Counter counter(Meter.Id id) {
        return registerMeterIfNecessary(Counter.class, id, this::newCounter);
    }

    /**
     * Called by {@link Gauge.Builder#register} and the convenience methods above.
     */
    <T> Gauge gauge(Meter.Id id, T obj, ToDoubleFunction<T> valueFunction) {
        return registerMeterIfNecessary(Gauge.class, id, i -> newGauge(i, obj, valueFunction));
    }

    /**
     * Called by {@link Timer.Builder#register} and the convenience methods above.
     */
    Timer timer(Meter.Id id) {
        return registerMeterIfNecessary(Timer.class, id, this::newTimer);
    }

    /**
     * Called by {@link DistributionSummary.Builder#register} and the convenience methods above.
     */
    DistributionSummary summary(Meter.Id id) {
        return registerMeterIfNecessary(DistributionSummary.class, id, this::newDistributionSummary);
    }

    // ── Abstract factory methods (subclasses implement these) ────────

    /**
     * Creates a new Counter implementation for the given id.
     * Called once per unique meter id, inside a synchronized block.
     */
    protected abstract Counter newCounter(Meter.Id id);

    /**
     * Creates a new Gauge implementation for the given id.
     * Called once per unique meter id, inside a synchronized block.
     *
     * @param id             the gauge's identity
     * @param obj            the state object to observe (may be held via WeakReference)
     * @param valueFunction  extracts a double from the state object
     */
    protected abstract <T> Gauge newGauge(Meter.Id id, T obj, ToDoubleFunction<T> valueFunction);

    /**
     * Creates a new Timer implementation for the given id.
     * Called once per unique meter id, inside a synchronized block.
     */
    protected abstract Timer newTimer(Meter.Id id);

    /**
     * Creates a new DistributionSummary implementation for the given id.
     * Called once per unique meter id, inside a synchronized block.
     */
    protected abstract DistributionSummary newDistributionSummary(Meter.Id id);

    // ── Query ────────────────────────────────────────────────────────

    /**
     * Returns an unmodifiable snapshot of all registered meters.
     */
    public List<Meter> getMeters() {
        return Collections.unmodifiableList(new ArrayList<>(meterMap.values()));
    }

    /**
     * Returns the clock used by this registry.
     */
    public Clock getClock() {
        return clock;
    }

    // ── Core registration logic ──────────────────────────────────────

    /**
     * Double-checked locking registration. If a meter with the same Id already
     * exists, it is returned (with a type-safety check). Otherwise, the factory
     * is invoked inside a synchronized block to create a new meter.
     *
     * @param meterClass  expected concrete type (for type-safety cast)
     * @param id          the meter's identity
     * @param meterFactory  creates the concrete meter (e.g., {@code this::newCounter})
     * @return the registered (or pre-existing) meter, cast to M
     * @throws IllegalArgumentException if an existing meter has the same id
     *                                  but a different type
     */
    private <M extends Meter> M registerMeterIfNecessary(
            Class<M> meterClass, Meter.Id id, Function<Meter.Id, M> meterFactory) {

        // Fast path: lock-free lookup
        Meter existing = meterMap.get(id);
        if (existing != null) {
            return castOrThrow(meterClass, existing, id);
        }

        // Slow path: create under lock
        synchronized (meterMapLock) {
            existing = meterMap.get(id);
            if (existing != null) {
                return castOrThrow(meterClass, existing, id);
            }

            M newMeter = meterFactory.apply(id);
            meterMap.put(id, newMeter);
            return newMeter;
        }
    }

    private <M extends Meter> M castOrThrow(Class<M> meterClass, Meter existing, Meter.Id id) {
        if (!meterClass.isInstance(existing)) {
            throw new IllegalArgumentException(
                    "Meter with id '" + id.getName() + "' already exists with type "
                            + existing.getClass().getSimpleName()
                            + " but was expected to be " + meterClass.getSimpleName());
        }
        return meterClass.cast(existing);
    }

}
```

### `src/main/java/simple/micrometer/internal/SimpleMeterRegistry.java` [MODIFIED]

Added `newDistributionSummary()` factory method.

```java
package simple.micrometer.internal;

import simple.micrometer.api.Clock;
import simple.micrometer.api.Counter;
import simple.micrometer.api.DistributionSummary;
import simple.micrometer.api.Gauge;
import simple.micrometer.api.Meter;
import simple.micrometer.api.MeterRegistry;
import simple.micrometer.api.Timer;

import java.util.function.ToDoubleFunction;

/**
 * A minimal, in-memory meter registry that creates cumulative meter
 * implementations. Primarily intended for testing and simple use cases.
 *
 * <p>In the real Micrometer, {@code SimpleMeterRegistry} supports both
 * cumulative and step modes via a {@code SimpleConfig}. Our simplified
 * version always uses cumulative mode.
 *
 * <p>Simplified from {@code io.micrometer.core.instrument.simple.SimpleMeterRegistry}.
 */
public class SimpleMeterRegistry extends MeterRegistry {

    public SimpleMeterRegistry() {
        super(Clock.SYSTEM);
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
        return new CumulativeTimer(id, clock);
    }

    @Override
    protected DistributionSummary newDistributionSummary(Meter.Id id) {
        return new CumulativeDistributionSummary(id, clock);
    }

}
```

### `src/main/java/simple/micrometer/internal/TimeWindowMax.java` [MODIFIED]

Added `record(double)` / `poll()` API for raw double values used by DistributionSummary.

```java
package simple.micrometer.internal;

import simple.micrometer.api.Clock;

import java.util.concurrent.TimeUnit;
import java.util.concurrent.atomic.AtomicLong;

/**
 * Tracks the maximum recorded value over a rotating time window, so old
 * peaks naturally decay away. Uses a ring buffer of {@link AtomicLong} slots
 * with lock-free CAS-based updates.
 *
 * <p>Design:
 * <ul>
 *   <li><b>On record</b>: updates ALL slots with the new max — so the value
 *       is visible in every overlapping time window.</li>
 *   <li><b>On poll</b>: reads only the current slot — giving the max within
 *       the most recent rotation window.</li>
 *   <li><b>On rotation</b>: zeroes out the oldest slot and advances the
 *       current-bucket pointer. If enough time passes without any recordings,
 *       all slots decay to 0.</li>
 * </ul>
 *
 * <p>Default configuration: 2-minute expiry across 3 buffer slots, so each
 * slot covers 40 seconds.
 *
 * <p>Simplified from {@code io.micrometer.core.instrument.distribution.TimeWindowMax}.
 * The real version reads expiry/bufferLength from {@code DistributionStatisticConfig};
 * our version uses hardcoded defaults.
 */
public class TimeWindowMax {

    private static final long DEFAULT_EXPIRY_MS = 120_000; // 2 minutes
    private static final int DEFAULT_BUFFER_LENGTH = 3;

    private final Clock clock;
    private final long durationBetweenRotatesMillis;
    private final AtomicLong[] ringBuffer;
    private int currentBucket;
    private volatile long lastRotateTimestampMillis;

    public TimeWindowMax(Clock clock) {
        this(clock, DEFAULT_EXPIRY_MS, DEFAULT_BUFFER_LENGTH);
    }

    TimeWindowMax(Clock clock, long expiryMillis, int bufferLength) {
        this.clock = clock;
        this.durationBetweenRotatesMillis = expiryMillis / bufferLength;
        this.ringBuffer = new AtomicLong[bufferLength];
        for (int i = 0; i < bufferLength; i++) {
            this.ringBuffer[i] = new AtomicLong(0);
        }
        this.currentBucket = 0;
        this.lastRotateTimestampMillis = clock.wallTime();
    }

    /**
     * Records a sample value (in nanoseconds). Updates all ring buffer slots
     * so the max is visible in every active time window.
     */
    public void record(long nanoAmount) {
        rotate();
        for (AtomicLong slot : ringBuffer) {
            updateMax(slot, nanoAmount);
        }
    }

    /**
     * Records a raw double sample value (for non-time meters like
     * {@code DistributionSummary}). Uses {@link Double#doubleToLongBits}
     * for storage — IEEE 754 bit ordering preserves comparison semantics
     * for positive doubles, so the CAS-based max update works correctly.
     *
     * <p>A given {@code TimeWindowMax} instance should use either the
     * {@code record(long)} / {@code poll(TimeUnit)} API (for timers) or
     * the {@code record(double)} / {@code poll()} API (for summaries),
     * but not both.
     */
    public void record(double amount) {
        rotate();
        long bits = Double.doubleToLongBits(amount);
        for (AtomicLong slot : ringBuffer) {
            updateMax(slot, bits);
        }
    }

    /**
     * Returns the current maximum as a raw double (no unit conversion).
     * Intended for non-time meters like {@code DistributionSummary}.
     */
    public double poll() {
        rotate();
        long bits = ringBuffer[currentBucket].get();
        return Double.longBitsToDouble(bits);
    }

    /**
     * Returns the current maximum in the requested time unit.
     * Only reads the current bucket — values in rotated-out buckets
     * have already been zeroed by {@link #rotate()}.
     */
    public double poll(TimeUnit unit) {
        rotate();
        long nanos = ringBuffer[currentBucket].get();
        return nanosToUnit((double) nanos, unit);
    }

    private void rotate() {
        long timeSinceLastRotate = clock.wallTime() - lastRotateTimestampMillis;
        if (timeSinceLastRotate < durationBetweenRotatesMillis) {
            return;
        }

        // How many rotations have we missed?
        long rotationsToPerform = timeSinceLastRotate / durationBetweenRotatesMillis;

        if (rotationsToPerform >= ringBuffer.length) {
            // Entire buffer is stale — zero everything
            for (AtomicLong slot : ringBuffer) {
                slot.set(0);
            }
        } else {
            for (long i = 0; i < rotationsToPerform; i++) {
                currentBucket = (currentBucket + 1) % ringBuffer.length;
                ringBuffer[currentBucket].set(0);
            }
        }

        lastRotateTimestampMillis = clock.wallTime();
    }

    /**
     * Lock-free CAS loop to update a slot if the new sample exceeds
     * the current maximum.
     */
    private void updateMax(AtomicLong slot, long sample) {
        long curMax;
        do {
            curMax = slot.get();
        } while (curMax < sample && !slot.compareAndSet(curMax, sample));
    }

    static double nanosToUnit(double nanos, TimeUnit unit) {
        return switch (unit) {
            case NANOSECONDS -> nanos;
            case MICROSECONDS -> nanos / 1_000;
            case MILLISECONDS -> nanos / 1_000_000;
            case SECONDS -> nanos / 1_000_000_000;
            case MINUTES -> nanos / 60_000_000_000L;
            case HOURS -> nanos / 3_600_000_000_000L;
            case DAYS -> nanos / 86_400_000_000_000L;
        };
    }

}
```
