# Chapter 10: Distribution Statistics & Histograms

> **Feature 10** · Tier 3 · Depends on: Features 3–4 (Timer and DistributionSummary)

After this chapter you will be able to **configure rich distribution statistics** on Timers and
DistributionSummaries — client-side percentile computation, SLO boundary histograms, and
full percentile histograms — then read those statistics through a unified snapshot API.

---

## 1. The API Contract

Before writing any implementation, let's define exactly what a client sees.

### What the client imports

```java
import simple.micrometer.api.Timer;
import simple.micrometer.api.DistributionSummary;
import simple.micrometer.api.HistogramSnapshot;
import simple.micrometer.api.ValueAtPercentile;
import simple.micrometer.api.CountAtBucket;
import simple.micrometer.api.DistributionStatisticConfig;
import simple.micrometer.api.MeterRegistry;
import simple.micrometer.internal.SimpleMeterRegistry;
```

### What the client writes

```java
MeterRegistry registry = new SimpleMeterRegistry();

// Timer with client-side percentiles
Timer timer = Timer.builder("http.request.duration")
    .publishPercentiles(0.5, 0.95, 0.99)
    .register(registry);

timer.record(Duration.ofMillis(100));
timer.record(Duration.ofMillis(200));
timer.record(Duration.ofMillis(300));

// Read histogram snapshot
HistogramSnapshot snapshot = timer.takeSnapshot();
ValueAtPercentile[] percentiles = snapshot.percentileValues();
// percentiles[0] ≈ {percentile=0.5, value≈200ms}

// DistributionSummary with SLO boundaries
DistributionSummary summary = DistributionSummary.builder("http.response.size")
    .serviceLevelObjectives(1024, 4096, 16384)
    .publishPercentileHistogram()
    .register(registry);

summary.record(512);
summary.record(2048);

HistogramSnapshot sloSnapshot = summary.takeSnapshot();
CountAtBucket[] buckets = sloSnapshot.histogramCounts();
// buckets[0] = {bucket=1024, count=1}  (512 <= 1024)
```

### Behavioral contract

| Promise | Detail |
|---------|--------|
| Percentile computation | `publishPercentiles(0.5, 0.95, 0.99)` causes `takeSnapshot()` to return pre-computed percentile values using histogram interpolation |
| SLO bucket counting | `serviceLevelObjectives(...)` causes `takeSnapshot()` to return cumulative counts at each SLO boundary |
| Percentile histogram | `publishPercentileHistogram()` publishes logarithmically-spaced bucket counts for aggregable percentile computation by the backend |
| Combined support | Percentiles AND SLOs can be configured simultaneously on the same meter |
| Duration-based SLOs | Timer's builder accepts `Duration` for SLOs, min, and max — automatically converts to nanoseconds |
| Raw SLOs | DistributionSummary's builder accepts raw `double` values for SLOs, min, and max |
| Empty when unconfigured | If no distribution stats are configured, `takeSnapshot()` returns basic stats with empty percentile/histogram arrays |
| No overhead when disabled | When `DistributionStatisticConfig.NONE` is used, no histogram is created — recording is zero-overhead |
| Approximate percentiles | Percentiles are computed by interpolating within fixed bucket boundaries — approximate but efficient |
| Unified snapshot | `HistogramSnapshot` provides count, total, max, mean, percentiles, and bucket counts in one object |

---

## 2. Client Usage & Tests

These tests define the API's behavior — they were written **before** any implementation.

### Timer with percentiles

```java
@Test
void shouldComputePercentiles_ForTimer() {
    Timer timer = Timer.builder("http.request.duration")
            .publishPercentiles(0.5, 0.95, 0.99)
            .register(registry);

    // Record 100 values: 1ms, 2ms, ..., 100ms
    for (int i = 1; i <= 100; i++) {
        timer.record(Duration.ofMillis(i));
    }

    HistogramSnapshot snapshot = timer.takeSnapshot();
    assertThat(snapshot.count()).isEqualTo(100);

    ValueAtPercentile[] percentiles = snapshot.percentileValues();
    assertThat(percentiles).hasSize(3);
    assertThat(percentiles[0].percentile()).isEqualTo(0.5);

    // p50 should be around 50ms (approximate)
    assertThat(percentiles[0].value(TimeUnit.MILLISECONDS))
            .isCloseTo(50.0, within(15.0));
}
```

### DistributionSummary with SLO boundaries

```java
@Test
void shouldTrackSLOBuckets_ForDistributionSummary() {
    DistributionSummary summary = DistributionSummary.builder("http.response.size")
            .serviceLevelObjectives(1024, 4096, 16384)
            .register(registry);

    summary.record(512);    // below 1024
    summary.record(1024);   // at 1024
    summary.record(2048);   // between 1024 and 4096
    summary.record(4096);   // at 4096
    summary.record(8192);   // between 4096 and 16384
    summary.record(32768);  // above 16384

    HistogramSnapshot snapshot = summary.takeSnapshot();
    CountAtBucket[] buckets = snapshot.histogramCounts();
    assertThat(buckets).hasSize(3);

    // <= 1024: 512, 1024 → cumulative count 2
    assertThat(buckets[0].bucket()).isEqualTo(1024);
    assertThat(buckets[0].count()).isEqualTo(2);

    // <= 4096: 512, 1024, 2048, 4096 → cumulative count 4
    assertThat(buckets[1].bucket()).isEqualTo(4096);
    assertThat(buckets[1].count()).isEqualTo(4);
}
```

### Empty snapshot when no config

```java
@Test
void shouldReturnEmptySnapshot_WhenNoDistributionConfigured() {
    Timer timer = Timer.builder("latency").register(registry);
    timer.record(Duration.ofMillis(100));

    HistogramSnapshot snapshot = timer.takeSnapshot();
    assertThat(snapshot.count()).isEqualTo(1);
    assertThat(snapshot.percentileValues()).isEmpty();
    assertThat(snapshot.histogramCounts()).isEmpty();
}
```

### Combined percentiles + SLOs

```java
@Test
void shouldSupportBothPercentilesAndSLOs() {
    DistributionSummary summary = DistributionSummary.builder("payload.size")
            .publishPercentiles(0.5, 0.99)
            .serviceLevelObjectives(1000, 5000)
            .register(registry);

    for (int i = 1; i <= 100; i++) {
        summary.record(i * 100);
    }

    HistogramSnapshot snapshot = summary.takeSnapshot();
    assertThat(snapshot.percentileValues()).hasSize(2);
    assertThat(snapshot.histogramCounts()).hasSize(2);

    // <= 1000: values 100-1000 → 10 values
    assertThat(snapshot.histogramCounts()[0].count()).isEqualTo(10);
}
```

---

## 3. Implementing the Call Chain — API Layer

### 3a. DistributionStatisticConfig — the configuration object

The config holds all distribution statistics settings. It uses a Builder pattern with sensible
defaults. The key insight: all fields are eagerly resolved (not nullable) — a simplification
from the real Micrometer, which uses nullable fields for layered merging.

```java
public class DistributionStatisticConfig {

    public static final DistributionStatisticConfig NONE = builder().build();

    private final double[] percentiles;
    private final double[] serviceLevelObjectives;
    private final boolean percentileHistogram;
    private final double minimumExpectedValue;
    private final double maximumExpectedValue;

    // Key methods:
    public boolean isPublishingPercentiles()  // percentiles.length > 0
    public boolean isPublishingHistogram()    // percentileHistogram || slos.length > 0
    public boolean isActive()                 // either of the above
}
```

### 3b. Value types — ValueAtPercentile and CountAtBucket

These are Java records — immutable value objects that carry snapshot data.

```java
// The value at a specific percentile rank
public record ValueAtPercentile(double percentile, double value) {
    public double value(TimeUnit unit) {
        return value / unit.toNanos(1);  // Convert nanos → requested unit
    }
}

// The cumulative count at a histogram bucket boundary
public record CountAtBucket(double bucket, double count) {
    public double bucket(TimeUnit unit) {
        return bucket / unit.toNanos(1);
    }
}
```

### 3c. HistogramSnapshot — the unified readout

An immutable snapshot capturing all distribution statistics at a point in time:

```java
public final class HistogramSnapshot {
    private final long count;
    private final double total;
    private final double max;
    private final ValueAtPercentile[] percentileValues;
    private final CountAtBucket[] histogramCounts;

    public static HistogramSnapshot empty(long count, double total, double max) { ... }
}
```

### 3d. HistogramSupport — the takeSnapshot() interface

```java
public interface HistogramSupport {
    HistogramSnapshot takeSnapshot();
}
```

Both `Timer` and `DistributionSummary` now extend this interface:

```java
public interface Timer extends Meter, HistogramSupport { ... }
public interface DistributionSummary extends Meter, HistogramSupport { ... }
```

### 3e. Builder enhancements

Timer.Builder and DistributionSummary.Builder gain distribution config methods.
Timer's builder converts `Duration` to nanoseconds; DistributionSummary uses raw doubles:

```java
// Timer.Builder:
public Builder publishPercentiles(double... percentiles) { ... }
public Builder serviceLevelObjectives(Duration... slos) {
    double[] nanos = Stream.of(slos).mapToDouble(Duration::toNanos).toArray();
    distributionConfigBuilder.serviceLevelObjectives(nanos);
    return this;
}
public Builder publishPercentileHistogram() { ... }

// DistributionSummary.Builder:
public Builder serviceLevelObjectives(double... slos) {
    distributionConfigBuilder.serviceLevelObjectives(slos);
    return this;
}
```

The `register()` method now passes the built config to the registry:

```java
public Timer register(MeterRegistry registry) {
    Meter.Id id = new Meter.Id(name, tags, baseUnit, description, Meter.Type.TIMER);
    return registry.timer(id, distributionConfigBuilder.build());
}
```

---

## 4. Implementing the Call Chain — Registration Layer

### MeterRegistry changes

The abstract factory methods now accept the distribution config:

```java
// Old:
protected abstract Timer newTimer(Meter.Id id);

// New:
protected abstract Timer newTimer(Meter.Id id, DistributionStatisticConfig distributionConfig);
```

Convenience methods (e.g., `registry.timer("name", "key", "value")`) pass `NONE`:

```java
Timer timer(Meter.Id id) {
    return timer(id, DistributionStatisticConfig.NONE);
}

Timer timer(Meter.Id id, DistributionStatisticConfig distributionConfig) {
    return registerMeterIfNecessary(Timer.class, id,
            i -> this.newTimer(i, distributionConfig), NoopTimer::new);
}
```

---

## 5. Implementing the Call Chain — Implementation Layer

### FixedBoundaryHistogram — the core data structure

This is where the magic happens. A FixedBoundaryHistogram uses sorted bucket boundaries
and atomic counters to track the distribution of observed values:

```
Boundaries:  [ 10 ]  [ 50 ]  [ 100 ]  [ 500 ]  [ 1000 ]  [ MAX ]
Counts:      [  5 ]  [ 15 ]  [  60 ]  [  15 ]  [    3 ]  [   2 ]
Cumulative:  [  5 ]  [ 20 ]  [  80 ]  [  95 ]  [   98 ]  [ 100 ]
```

**Recording**: Binary search for the smallest boundary ≥ value, increment that counter.

**Percentile interpolation**: To find the p50 (target = 50):
1. Walk cumulative counts: 5, 20, **80** — first ≥ 50 is at boundary 100
2. Previous boundary is 50, previous cumulative is 20
3. Interpolate: 50 + (50 - 20) / (80 - 20) × (100 - 50) = 50 + 25 = 75

**Bucket boundaries are chosen by the factory method:**
- `publishPercentiles` or `publishPercentileHistogram` → 73 log-spaced boundaries
- `serviceLevelObjectives` → exact SLO values inserted
- Always includes `Double.MAX_VALUE` to capture all values

```java
public static FixedBoundaryHistogram create(DistributionStatisticConfig config) {
    if (!config.isActive()) return null;

    TreeSet<Double> boundaries = new TreeSet<>();
    if (config.isPublishingPercentiles() || config.isPercentileHistogram()) {
        boundaries.addAll(generateLogBuckets(min, max, 73));
    }
    for (double slo : config.getServiceLevelObjectives()) {
        boundaries.add(slo);
    }
    boundaries.add(Double.MAX_VALUE);
    return new FixedBoundaryHistogram(boundaries.toArray());
}
```

### CumulativeTimer — recording to the histogram

When a CumulativeTimer has distribution stats configured, every `record()` call also
feeds the histogram:

```java
@Override
public void record(long amount, TimeUnit unit) {
    if (amount >= 0) {
        long nanos = unit.toNanos(amount);
        count.increment();
        totalNanos.add(nanos);
        max.record(nanos);
        if (histogram != null) {
            histogram.record((double) nanos);
        }
    }
}
```

### takeSnapshot() — assembling the snapshot

```java
@Override
public HistogramSnapshot takeSnapshot() {
    long c = count();
    double total = totalTime(TimeUnit.NANOSECONDS);
    double m = max(TimeUnit.NANOSECONDS);

    if (histogram == null) {
        return HistogramSnapshot.empty(c, total, m);
    }

    // Compute percentiles from histogram data
    ValueAtPercentile[] percentiles = histogram.getPercentileValues(
            distributionConfig.getPercentiles(), c);

    // Build bucket counts — only for published boundaries
    CountAtBucket[] buckets;
    if (distributionConfig.isPercentileHistogram()) {
        buckets = histogram.getCountAtBuckets();  // all log-spaced boundaries
    } else if (distributionConfig.isPublishingHistogram()) {
        // Only SLO boundaries
        double[] slos = distributionConfig.getServiceLevelObjectives();
        buckets = new CountAtBucket[slos.length];
        for (int i = 0; i < slos.length; i++) {
            buckets[i] = new CountAtBucket(slos[i],
                    histogram.cumulativeCountAtBoundary(slos[i]));
        }
    } else {
        buckets = new CountAtBucket[0];
    }

    return new HistogramSnapshot(c, total, m, percentiles, buckets);
}
```

---

## 6. Try It Yourself

1. **Add percentile-based alerting**: Create a Timer with `publishPercentiles(0.5, 0.99)`.
   Record random durations and verify that the p99 is always ≥ the p50.

2. **SLO compliance tracking**: Create a DistributionSummary with
   `serviceLevelObjectives(100, 500, 1000)`. Record 1000 random values and compute
   what percentage of requests meet each SLO from the bucket counts.

3. **Compare accuracy**: Record 10,000 sorted values. Compare the histogram-interpolated
   p99 to the true p99 (value at index 9900). How close is the approximation?

---

## 7. Why This Works

### Logarithmic bucket spacing gives uniform error bounds

With linearly-spaced buckets, you get high resolution at the low end and poor resolution at
the high end. Logarithmic spacing ensures that the *relative* error is approximately constant
across the entire range. This is why Prometheus, Atlas, and other monitoring systems use
exponential bucket boundaries.

### The interpolation trick avoids heavy data structures

The real Micrometer uses HdrHistogram (a specialized data structure from Gil Tene) for
precise percentile computation. Our simplified version achieves similar practical accuracy
with a much simpler approach: fixed buckets + linear interpolation. With 73 log-spaced
buckets, the error is typically within a few percent — more than adequate for operational
monitoring.

### The "null histogram" pattern is zero-overhead

When no distribution statistics are configured, `FixedBoundaryHistogram.create()` returns
`null`. The recording path checks `if (histogram != null)` — a branch that the CPU's
branch predictor optimizes away after the first few calls. This means that meters without
distribution config pay essentially zero overhead.

### SLO boundaries are exact because they're included as boundaries

When SLO values are specified, they're inserted directly into the bucket boundary array.
This means `cumulativeCountAtBoundary(sloValue)` returns an *exact* count (not an
interpolation), which is critical for SLO compliance reporting.

---

## 8. What We Enhanced

| Component | Before (Features 3–4) | After (Feature 10) |
|-----------|----------------------|---------------------|
| `Timer` interface | `extends Meter` | `extends Meter, HistogramSupport` — adds `takeSnapshot()` |
| `Timer.Builder` | name, tags, description | + `publishPercentiles()`, `serviceLevelObjectives(Duration...)`, `publishPercentileHistogram()`, `minimumExpectedValue()`, `maximumExpectedValue()` |
| `DistributionSummary` interface | `extends Meter` | `extends Meter, HistogramSupport` — adds `takeSnapshot()` |
| `DistributionSummary.Builder` | name, tags, description, baseUnit | + `publishPercentiles()`, `serviceLevelObjectives(double...)`, `publishPercentileHistogram()`, `minimumExpectedValue()`, `maximumExpectedValue()` |
| `MeterRegistry` | `newTimer(id)`, `newDistributionSummary(id)` | `newTimer(id, config)`, `newDistributionSummary(id, config)` — distribution config flows from builder through registry |
| `SimpleMeterRegistry` | Creates plain CumulativeTimer/Summary | Creates CumulativeTimer/Summary with histogram based on config |
| `CumulativeTimer` | `(id, clock)` | `(id, clock, config)` — records to histogram, implements `takeSnapshot()` |
| `CumulativeDistributionSummary` | `(id, clock)` | `(id, clock, config)` — records to histogram, implements `takeSnapshot()` |
| `NoopTimer` | No snapshot | Implements `takeSnapshot()` → empty snapshot |
| `NoopDistributionSummary` | No snapshot | Implements `takeSnapshot()` → empty snapshot |
| `CompositeTimer` | No snapshot | Implements `takeSnapshot()` → delegates to first child |
| `CompositeDistributionSummary` | No snapshot | Implements `takeSnapshot()` → delegates to first child |

---

## 9. Connection to Real Framework

| Simplified Class | Real Framework Class | Key Simplification |
|-----------------|---------------------|-------------------|
| `DistributionStatisticConfig` | `io.micrometer.core.instrument.distribution.DistributionStatisticConfig` | Direct fields vs. nullable layered merge; no `Mergeable` interface |
| `HistogramSupport` | `io.micrometer.core.instrument.distribution.HistogramSupport` | Interface vs. abstract class extending Meter |
| `HistogramSnapshot` | `io.micrometer.core.instrument.distribution.HistogramSnapshot` | No `outputSummary` function; simpler constructors |
| `ValueAtPercentile` | `io.micrometer.core.instrument.distribution.ValueAtPercentile` | Java record vs. class with equals/hashCode |
| `CountAtBucket` | `io.micrometer.core.instrument.distribution.CountAtBucket` | Java record vs. class; no `isPositiveInf()` |
| `FixedBoundaryHistogram` | `FixedBoundaryHistogram` + `AbstractTimeWindowHistogram` + `TimeWindowPercentileHistogram` | Single class replaces three-class hierarchy; no time-windowed ring buffer for histogram decay; uses interpolation instead of HdrHistogram for percentile computation |

### What the real Micrometer does differently

1. **Layered nullable config merge**: Real `DistributionStatisticConfig` has all nullable fields.
   `merge(parent)` resolves each field: user value → filter value → registry default → global default.
   This enables `MeterFilter.configure()` to inject per-meter histogram config.

2. **Time-windowed histograms**: `AbstractTimeWindowHistogram` uses a ring buffer of histograms,
   rotating them on a configurable expiry schedule. This causes old data to decay, so snapshots
   reflect recent behavior. Our version accumulates all-time.

3. **HdrHistogram for percentiles**: `TimeWindowPercentileHistogram` uses Gil Tene's HdrHistogram
   library (`DoubleRecorder` / `DoubleHistogram`) for precise, memory-efficient percentile
   computation. Our version uses fixed-bucket interpolation.

4. **Separate histogram strategies**: The real framework chooses between `TimeWindowPercentileHistogram`
   (when percentiles needed) and `TimeWindowFixedBoundaryHistogram` (when only bucket counts needed)
   based on config. Our version uses a single `FixedBoundaryHistogram` for everything.

5. **276 pre-defined percentile histogram buckets**: `PercentileHistogramBuckets` contains a
   carefully tuned set of boundaries. Our version generates 73 log-spaced boundaries.

---

## 10. Complete Code

All files created or modified in this feature, read back from `src/`.

### `api/ValueAtPercentile.java` [NEW]

```java
package simple.micrometer.api;

import java.util.concurrent.TimeUnit;

public record ValueAtPercentile(double percentile, double value) {

    public double value(TimeUnit unit) {
        return value / unit.toNanos(1);
    }

    @Override
    public String toString() {
        return "ValueAtPercentile{percentile=" + percentile + ", value=" + value + '}';
    }

}
```

### `api/CountAtBucket.java` [NEW]

```java
package simple.micrometer.api;

import java.util.concurrent.TimeUnit;

public record CountAtBucket(double bucket, double count) {

    public double bucket(TimeUnit unit) {
        return bucket / unit.toNanos(1);
    }

    @Override
    public String toString() {
        return "CountAtBucket{bucket=" + bucket + ", count=" + count + '}';
    }

}
```

### `api/DistributionStatisticConfig.java` [NEW]

```java
package simple.micrometer.api;

public class DistributionStatisticConfig {

    public static final DistributionStatisticConfig NONE = builder().build();

    private final double[] percentiles;
    private final double[] serviceLevelObjectives;
    private final boolean percentileHistogram;
    private final double minimumExpectedValue;
    private final double maximumExpectedValue;

    private DistributionStatisticConfig(Builder builder) {
        this.percentiles = builder.percentiles != null ? builder.percentiles.clone() : new double[0];
        this.serviceLevelObjectives = builder.serviceLevelObjectives != null
                ? builder.serviceLevelObjectives.clone() : new double[0];
        this.percentileHistogram = builder.percentileHistogram;
        this.minimumExpectedValue = builder.minimumExpectedValue;
        this.maximumExpectedValue = builder.maximumExpectedValue;
    }

    public double[] getPercentiles() { return percentiles.clone(); }
    public double[] getServiceLevelObjectives() { return serviceLevelObjectives.clone(); }
    public boolean isPercentileHistogram() { return percentileHistogram; }
    public double getMinimumExpectedValue() { return minimumExpectedValue; }
    public double getMaximumExpectedValue() { return maximumExpectedValue; }
    public boolean isPublishingPercentiles() { return percentiles.length > 0; }
    public boolean isPublishingHistogram() { return percentileHistogram || serviceLevelObjectives.length > 0; }
    public boolean isActive() { return isPublishingPercentiles() || isPublishingHistogram(); }

    public static Builder builder() { return new Builder(); }

    public static class Builder {
        private double[] percentiles;
        private double[] serviceLevelObjectives;
        private boolean percentileHistogram;
        private double minimumExpectedValue = 1.0;
        private double maximumExpectedValue = Double.POSITIVE_INFINITY;
        private Builder() {}
        public Builder percentiles(double... percentiles) { this.percentiles = percentiles; return this; }
        public Builder serviceLevelObjectives(double... slos) { this.serviceLevelObjectives = slos; return this; }
        public Builder percentileHistogram(boolean enabled) { this.percentileHistogram = enabled; return this; }
        public Builder minimumExpectedValue(double min) { this.minimumExpectedValue = min; return this; }
        public Builder maximumExpectedValue(double max) { this.maximumExpectedValue = max; return this; }
        public DistributionStatisticConfig build() { return new DistributionStatisticConfig(this); }
    }
}
```

### `api/HistogramSnapshot.java` [NEW]

```java
package simple.micrometer.api;

import java.util.concurrent.TimeUnit;

public final class HistogramSnapshot {

    private final long count;
    private final double total;
    private final double max;
    private final ValueAtPercentile[] percentileValues;
    private final CountAtBucket[] histogramCounts;

    public HistogramSnapshot(long count, double total, double max,
                             ValueAtPercentile[] percentileValues,
                             CountAtBucket[] histogramCounts) {
        this.count = count;
        this.total = total;
        this.max = max;
        this.percentileValues = percentileValues != null ? percentileValues : new ValueAtPercentile[0];
        this.histogramCounts = histogramCounts != null ? histogramCounts : new CountAtBucket[0];
    }

    public static HistogramSnapshot empty(long count, double total, double max) {
        return new HistogramSnapshot(count, total, max, new ValueAtPercentile[0], new CountAtBucket[0]);
    }

    public long count() { return count; }
    public double total() { return total; }
    public double total(TimeUnit unit) { return total / unit.toNanos(1); }
    public double max() { return max; }
    public double max(TimeUnit unit) { return max / unit.toNanos(1); }
    public double mean() { return count == 0 ? 0 : total / count; }
    public double mean(TimeUnit unit) { return count == 0 ? 0 : total(unit) / count; }
    public ValueAtPercentile[] percentileValues() { return percentileValues.clone(); }
    public CountAtBucket[] histogramCounts() { return histogramCounts.clone(); }
}
```

### `api/HistogramSupport.java` [NEW]

```java
package simple.micrometer.api;

public interface HistogramSupport {
    HistogramSnapshot takeSnapshot();
}
```

### `internal/FixedBoundaryHistogram.java` [NEW]

```java
package simple.micrometer.internal;

import simple.micrometer.api.CountAtBucket;
import simple.micrometer.api.DistributionStatisticConfig;
import simple.micrometer.api.ValueAtPercentile;

import java.util.Arrays;
import java.util.TreeSet;
import java.util.concurrent.atomic.AtomicLong;

public class FixedBoundaryHistogram {

    private static final int DEFAULT_PERCENTILE_BUCKET_COUNT = 73;

    private final double[] boundaries;
    private final AtomicLong[] counts;

    public FixedBoundaryHistogram(double[] boundaries) {
        this.boundaries = boundaries.clone();
        Arrays.sort(this.boundaries);
        this.counts = new AtomicLong[this.boundaries.length];
        for (int i = 0; i < this.counts.length; i++) {
            this.counts[i] = new AtomicLong(0);
        }
    }

    public void record(double value) {
        int idx = findBucket(value);
        counts[idx].incrementAndGet();
    }

    public CountAtBucket[] getCountAtBuckets() {
        CountAtBucket[] result = new CountAtBucket[boundaries.length];
        long cumulative = 0;
        for (int i = 0; i < boundaries.length; i++) {
            cumulative += counts[i].get();
            result[i] = new CountAtBucket(boundaries[i], cumulative);
        }
        return result;
    }

    public long cumulativeCountAtBoundary(double boundary) {
        int idx = Arrays.binarySearch(boundaries, boundary);
        if (idx < 0) idx = -idx - 2;
        if (idx < 0) return 0;
        long cum = 0;
        for (int i = 0; i <= idx; i++) cum += counts[i].get();
        return cum;
    }

    public ValueAtPercentile[] getPercentileValues(double[] percentiles, long totalCount) {
        if (totalCount == 0 || percentiles.length == 0) return new ValueAtPercentile[0];
        long[] cumulative = new long[boundaries.length];
        cumulative[0] = counts[0].get();
        for (int i = 1; i < boundaries.length; i++)
            cumulative[i] = cumulative[i - 1] + counts[i].get();
        ValueAtPercentile[] result = new ValueAtPercentile[percentiles.length];
        for (int p = 0; p < percentiles.length; p++) {
            double target = percentiles[p] * totalCount;
            result[p] = new ValueAtPercentile(percentiles[p], interpolatePercentile(target, cumulative));
        }
        return result;
    }

    public double[] getBoundaries() { return boundaries.clone(); }

    public static FixedBoundaryHistogram create(DistributionStatisticConfig config) {
        if (!config.isActive()) return null;
        TreeSet<Double> all = new TreeSet<>();
        if (config.isPublishingPercentiles() || config.isPercentileHistogram()) {
            double min = config.getMinimumExpectedValue();
            double max = config.getMaximumExpectedValue();
            if (max == Double.POSITIVE_INFINITY) max = min * 1_000_000_000_000.0;
            for (double b : generateLogBuckets(min, max, DEFAULT_PERCENTILE_BUCKET_COUNT)) all.add(b);
        }
        for (double slo : config.getServiceLevelObjectives()) all.add(slo);
        all.add(Double.MAX_VALUE);
        return new FixedBoundaryHistogram(all.stream().mapToDouble(Double::doubleValue).toArray());
    }

    private int findBucket(double value) {
        int idx = Arrays.binarySearch(boundaries, value);
        if (idx >= 0) return idx;
        return Math.min(-idx - 1, boundaries.length - 1);
    }

    private double interpolatePercentile(double target, long[] cumulative) {
        for (int i = 0; i < cumulative.length; i++) {
            if (cumulative[i] >= target) {
                if (i == 0) return boundaries[0];
                double lower = boundaries[i - 1], upper = boundaries[i];
                long lowerCum = cumulative[i - 1], bucketCount = cumulative[i] - lowerCum;
                if (bucketCount == 0) return upper;
                return lower + (target - lowerCum) / bucketCount * (upper - lower);
            }
        }
        return boundaries[boundaries.length - 1];
    }

    static double[] generateLogBuckets(double min, double max, int count) {
        double logMin = Math.log(min), logMax = Math.log(max);
        double[] buckets = new double[count];
        for (int i = 0; i < count; i++)
            buckets[i] = Math.exp(logMin + (logMax - logMin) * i / (count - 1));
        return buckets;
    }
}
```

### `api/Timer.java` [MODIFIED]

**Changes**: Interface now `extends Meter, HistogramSupport`. Builder gains
`publishPercentiles()`, `serviceLevelObjectives(Duration...)`, `publishPercentileHistogram()`,
`minimumExpectedValue(Duration)`, `maximumExpectedValue(Duration)`. `register()` passes
`DistributionStatisticConfig` to the registry.

### `api/DistributionSummary.java` [MODIFIED]

**Changes**: Interface now `extends Meter, HistogramSupport`. Builder gains
`publishPercentiles()`, `serviceLevelObjectives(double...)`, `publishPercentileHistogram()`,
`minimumExpectedValue(double)`, `maximumExpectedValue(double)`. `register()` passes
`DistributionStatisticConfig` to the registry.

### `api/MeterRegistry.java` [MODIFIED]

**Changes**: Abstract methods `newTimer(id)` → `newTimer(id, config)` and
`newDistributionSummary(id)` → `newDistributionSummary(id, config)`. Added overloaded
`timer(id, config)` and `summary(id, config)` registration methods. Convenience methods
pass `DistributionStatisticConfig.NONE`.

### `api/CompositeMeterRegistry.java` [MODIFIED]

**Changes**: Updated `newTimer` and `newDistributionSummary` to accept
`DistributionStatisticConfig` parameter (config is not forwarded to children in the
simplified version).

### `internal/SimpleMeterRegistry.java` [MODIFIED]

**Changes**: `newTimer(id, config)` passes config to `CumulativeTimer`.
`newDistributionSummary(id, config)` passes config to `CumulativeDistributionSummary`.

### `internal/CumulativeTimer.java` [MODIFIED]

**Changes**: Constructor accepts `DistributionStatisticConfig`. Creates
`FixedBoundaryHistogram` from config. `record()` forwards to histogram. Implements
`takeSnapshot()`.

### `internal/CumulativeDistributionSummary.java` [MODIFIED]

**Changes**: Same as CumulativeTimer — constructor accepts config, creates histogram,
records to it, implements `takeSnapshot()`.

### `internal/NoopTimer.java` [MODIFIED]

**Changes**: Implements `takeSnapshot()` → returns `HistogramSnapshot.empty(0, 0, 0)`.

### `internal/NoopDistributionSummary.java` [MODIFIED]

**Changes**: Implements `takeSnapshot()` → returns `HistogramSnapshot.empty(0, 0, 0)`.

### `internal/CompositeTimer.java` [MODIFIED]

**Changes**: Implements `takeSnapshot()` → delegates to `firstChild().takeSnapshot()`.

### `internal/CompositeDistributionSummary.java` [MODIFIED]

**Changes**: Implements `takeSnapshot()` → delegates to `firstChild().takeSnapshot()`.

### `api/DistributionStatisticTest.java` [NEW]

Client-perspective tests covering: Timer percentiles, DistributionSummary SLOs,
percentile histograms, Timer SLOs with Duration, empty snapshots, basic stats in
snapshots, combined percentiles + SLOs, config queries, and backward compatibility.
