# Chapter 10: Distribution Statistics

> Line references based on commit `2c8a4606c` of the Micrometer repository.

## Build Challenge

| Current State | Limitation | Objective |
|---------------|------------|-----------|
| Timer and DistributionSummary record count, total, and a decaying max — but nothing about the *shape* of the data | No way to answer "how many requests completed under 500ms?" or "what does the latency distribution look like?" without an external system | Build a histogram system with fixed-boundary buckets, a time-windowed ring buffer for decay, cascading configuration, and integration into Timer/DistributionSummary via `takeSnapshot()` |

---

## 10.1 The Integration Point

Distribution Statistics connects to the existing system in **two places**:

1. **`AbstractTimer.record()` / `AbstractDistributionSummary.record()`** — the `final` template methods gain a `histogram.recordLong()` / `histogram.recordDouble()` call *before* delegating to `recordNonNegative()`. This is the data path.

2. **`MeterRegistry`** — the filter chain gains a third stage: `MeterFilter.configure()` processes `DistributionStatisticConfig` through a cascading merge pipeline: Builder → Filter → Registry defaults.

**Modifying:** `src/main/java/dev/linhvu/micrometer/AbstractTimer.java`
**Change:** Add a `Histogram` field, record into it in `record()`, add `takeSnapshot()`

```java
import dev.linhvu.micrometer.distribution.Histogram;
import dev.linhvu.micrometer.distribution.HistogramSnapshot;
import dev.linhvu.micrometer.distribution.NoopHistogram;

// ... in AbstractTimer:

private final Histogram histogram;

// New constructor accepting a Histogram
protected AbstractTimer(Meter.Id id, Clock clock, TimeUnit baseTimeUnit, Histogram histogram) {
    super(id);
    this.clock = clock;
    this.baseTimeUnit = baseTimeUnit;
    this.histogram = (histogram != null) ? histogram : NoopHistogram.INSTANCE;
}

@Override
public final void record(long amount, TimeUnit unit) {
    if (amount >= 0) {
        histogram.recordLong(TimeUnit.NANOSECONDS.convert(amount, unit));  // NEW
        recordNonNegative(amount, unit);
    }
}

@Override
public HistogramSnapshot takeSnapshot() {
    return histogram.takeSnapshot(count(), totalTime(TimeUnit.NANOSECONDS), max(TimeUnit.NANOSECONDS));
}
```

**Modifying:** `src/main/java/dev/linhvu/micrometer/MeterRegistry.java`
**Change:** Add `configureDistributionStatistics()` pipeline, update `newTimer()`/`newDistributionSummary()` signatures to accept `DistributionStatisticConfig`

```java
// Updated abstract factories:
protected abstract Timer newTimer(Meter.Id id, DistributionStatisticConfig distributionStatisticConfig);
protected abstract DistributionSummary newDistributionSummary(Meter.Id id, double scale,
        DistributionStatisticConfig distributionStatisticConfig);

// New configure pipeline:
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
```

Three key decisions at the integration point:

1. **Histogram records BEFORE `recordNonNegative()`** — the histogram needs the raw nanosecond/scaled value for accurate bucketing. The subclass might store in different units internally.
2. **`record()` stays `final`** — every recording path (direct, Runnable, Supplier, Callable, Sample) automatically gets histogram tracking without subclass changes.
3. **Config flows through the registry, not the meter** — this enables per-meter configuration via `MeterFilter.configure()` without the meter knowing about filters.

**Direction:** This connects **the histogram subsystem** to **the Timer/DistributionSummary recording pipeline** and **the MeterRegistry filter chain**. To make it work, we need to build:
- `DistributionStatisticConfig` — cascading configuration with merge
- `Histogram` interface, `NoopHistogram`, and `HistogramSnapshot` — the histogram API
- `FixedBoundaryHistogram` — single-slot bucket array with binary search
- `AbstractTimeWindowHistogram` — ring buffer rotation engine
- `TimeWindowFixedBoundaryHistogram` — concrete histogram combining both
- `ValueAtPercentile` and `CountAtBucket` — snapshot data types
- Updates to Builders, MeterFilter, and all existing meter classes

---

## 10.2 The Configuration Layer

**New file:** `src/main/java/dev/linhvu/micrometer/distribution/DistributionStatisticConfig.java`

The config uses a **nullable-field cascading merge** pattern — each field is nullable, where `null` means "not configured at this level." The `merge()` method fills in nulls from a parent:

```java
package dev.linhvu.micrometer.distribution;

import java.time.Duration;
import java.util.NavigableSet;
import java.util.TreeSet;

public class DistributionStatisticConfig {

    public static final DistributionStatisticConfig DEFAULT = builder()
            .minimumExpectedValue(1.0)
            .maximumExpectedValue(Double.POSITIVE_INFINITY)
            .expiry(Duration.ofMinutes(2))
            .bufferLength(3)
            .build();

    public static final DistributionStatisticConfig NONE = builder().build();

    private final double[] percentiles;
    private final double[] serviceLevelObjectives;
    private final Double minimumExpectedValue;
    private final Double maximumExpectedValue;
    private final Duration expiry;
    private final Integer bufferLength;

    // ... constructor from Builder ...

    public DistributionStatisticConfig merge(DistributionStatisticConfig parent) {
        return builder()
                .percentiles(percentiles != null ? percentiles : parent.percentiles)
                .serviceLevelObjectives(
                        serviceLevelObjectives != null ? serviceLevelObjectives : parent.serviceLevelObjectives)
                .minimumExpectedValue(minimumExpectedValue != null ? minimumExpectedValue : parent.minimumExpectedValue)
                .maximumExpectedValue(maximumExpectedValue != null ? maximumExpectedValue : parent.maximumExpectedValue)
                .expiry(expiry != null ? expiry : parent.expiry)
                .bufferLength(bufferLength != null ? bufferLength : parent.bufferLength)
                .build();
    }

    public boolean isPublishingHistogram() {
        return serviceLevelObjectives != null && serviceLevelObjectives.length > 0;
    }

    public NavigableSet<Double> getHistogramBuckets() {
        NavigableSet<Double> buckets = new TreeSet<>();
        if (serviceLevelObjectives != null) {
            for (double slo : serviceLevelObjectives) {
                buckets.add(slo);
            }
        }
        return buckets;
    }

    // ... Builder with validation ...
}
```

The three-level cascade works like this:
```
Timer.builder("http.requests")
     .serviceLevelObjectives(Duration.ofMillis(100))  // ← Builder config (user)
     .register(registry)
         ↓
MeterFilter.configure(id, config)                      // ← Filter overrides
         ↓
config.merge(defaultHistogramConfig())                  // ← Registry defaults (DEFAULT)
```

Each layer only sets what it needs. `merge()` fills in the gaps.

---

## 10.3 The Histogram Interface and Snapshot Types

**New file:** `src/main/java/dev/linhvu/micrometer/distribution/Histogram.java`

```java
package dev.linhvu.micrometer.distribution;

public interface Histogram {
    void recordLong(long value);
    void recordDouble(double value);
    HistogramSnapshot takeSnapshot(long count, double total, double max);
}
```

**New file:** `src/main/java/dev/linhvu/micrometer/distribution/NoopHistogram.java`

```java
package dev.linhvu.micrometer.distribution;

public enum NoopHistogram implements Histogram {
    INSTANCE;

    @Override public void recordLong(long value) { }
    @Override public void recordDouble(double value) { }
    @Override public HistogramSnapshot takeSnapshot(long count, double total, double max) {
        return HistogramSnapshot.empty(count, total, max);
    }
}
```

**New file:** `src/main/java/dev/linhvu/micrometer/distribution/HistogramSnapshot.java`

An immutable point-in-time snapshot combining basic stats (count, total, max) from the meter with distribution data (percentile values, bucket counts) from the histogram:

```java
package dev.linhvu.micrometer.distribution;

public final class HistogramSnapshot {
    private final long count;
    private final double total;
    private final double max;
    private final ValueAtPercentile[] percentileValues;
    private final CountAtBucket[] histogramCounts;

    public static HistogramSnapshot empty(long count, double total, double max) {
        return new HistogramSnapshot(count, total, max, EMPTY_VALUES, EMPTY_COUNTS);
    }

    // ... accessors with TimeUnit conversion (total(TimeUnit), max(TimeUnit), mean(TimeUnit)) ...
}
```

**New files:** `ValueAtPercentile.java` and `CountAtBucket.java`

```java
// ValueAtPercentile — e.g., "the p95 latency is 142ms"
public final class ValueAtPercentile {
    private final double percentile;  // e.g., 0.95
    private final double value;       // e.g., 142000000 (nanoseconds)
}

// CountAtBucket — e.g., "47 requests completed in ≤ 100ms"
public final class CountAtBucket {
    private final double bucket;  // e.g., 100000000 (nanoseconds)
    private final double count;   // e.g., 47
}
```

---

## 10.4 The Ring Buffer Histogram

This is the heart of the feature — a time-windowed ring buffer that gives greater weight to recent samples.

**New file:** `src/main/java/dev/linhvu/micrometer/distribution/FixedBoundaryHistogram.java`

A single histogram slot with fixed bucket boundaries, backed by `AtomicLongArray`:

```java
class FixedBoundaryHistogram {
    private final double[] buckets;
    private final AtomicLongArray values;
    private final boolean isCumulativeBucketCounts;

    void record(long value) {
        int index = leastLessThanOrEqualTo(value);
        if (index >= 0 && index < values.length()) {
            values.incrementAndGet(index);
        }
    }

    // Binary search: find smallest bucket boundary ≥ value
    private int leastLessThanOrEqualTo(long value) {
        int low = 0;
        int high = buckets.length - 1;
        while (low <= high) {
            int mid = (low + high) >>> 1;
            if (buckets[mid] < value) low = mid + 1;
            else if (buckets[mid] > value) high = mid - 1;
            else return mid;
        }
        return low;
    }
}
```

**New file:** `src/main/java/dev/linhvu/micrometer/distribution/AbstractTimeWindowHistogram.java`

The ring buffer engine — multiple `FixedBoundaryHistogram` slots rotate based on wall-clock time:

```
Ring buffer: [ bucket0 | bucket1 | bucket2 ]
                           ↑ currentBucket

  expiry = 2 minutes
  bufferLength = 3 sub-windows
  rotation = 40 seconds each
```

**Write-all, read-one:** Every `record()` writes to ALL buckets. When a sub-window expires, its bucket is reset. On `takeSnapshot()`, only the current bucket is read.

```java
public abstract class AbstractTimeWindowHistogram<T> implements Histogram {
    private final T[] ringBuffer;
    private int currentBucket;
    private final long durationBetweenRotatesMillis;

    @Override
    public void recordLong(long value) {
        rotate();
        for (T bucket : ringBuffer) {
            recordLong(bucket, value);
        }
    }

    @Override
    public synchronized HistogramSnapshot takeSnapshot(long count, double total, double max) {
        rotate();
        CountAtBucket[] counts = countsAtBuckets(ringBuffer[currentBucket]);
        return new HistogramSnapshot(count, total, max, null, counts);
    }

    private void rotate() {
        // ... CAS-guarded rotation logic ...
        // Resets expired buckets, advances currentBucket
    }
}
```

**New file:** `src/main/java/dev/linhvu/micrometer/distribution/TimeWindowFixedBoundaryHistogram.java`

Concrete implementation combining the ring buffer with fixed-boundary buckets:

```java
public class TimeWindowFixedBoundaryHistogram
        extends AbstractTimeWindowHistogram<FixedBoundaryHistogram> {

    private final double[] buckets;
    private final boolean isCumulativeBucketCounts;

    public TimeWindowFixedBoundaryHistogram(Clock clock, DistributionStatisticConfig config) {
        super(clock, config, FixedBoundaryHistogram.class);
        NavigableSet<Double> bucketSet = config.getHistogramBuckets();
        this.buckets = bucketSet.stream().mapToDouble(Double::doubleValue).toArray();
        initRingBuffer();
    }

    @Override protected FixedBoundaryHistogram newBucket() {
        return new FixedBoundaryHistogram(buckets, isCumulativeBucketCounts);
    }
    // ... other abstract method implementations ...
}
```

---

## 10.5 Try It Yourself

Before looking at the complete code, try implementing these pieces:

<details><summary>Challenge 1: Implement the binary search in FixedBoundaryHistogram</summary>

Given sorted bucket boundaries `[100, 500, 1000]` and a value `250`, the binary search should return index `1` (the 500 bucket — the smallest boundary ≥ 250).

```java
private int leastLessThanOrEqualTo(long value) {
    int low = 0;
    int high = buckets.length - 1;
    while (low <= high) {
        int mid = (low + high) >>> 1;
        if (buckets[mid] < value) {
            low = mid + 1;
        } else if (buckets[mid] > value) {
            high = mid - 1;
        } else {
            return mid; // exact match
        }
    }
    return low; // first bucket >= value, or buckets.length if none
}
```

</details>

<details><summary>Challenge 2: Implement the three-level config merge</summary>

The merge cascade: Builder → Filter → Defaults. Each level only sets what it needs:

```java
// Builder: user sets SLOs
DistributionStatisticConfig builder = DistributionStatisticConfig.builder()
        .serviceLevelObjectives(100, 500)
        .build();

// Filter: adds buffer length
DistributionStatisticConfig filter = DistributionStatisticConfig.builder()
        .bufferLength(5)
        .build();

// Merge cascade:
DistributionStatisticConfig afterFilter = filter.merge(builder);
DistributionStatisticConfig finalConfig = afterFilter.merge(DistributionStatisticConfig.DEFAULT);

// Result: SLOs from builder, bufferLength from filter, expiry/min/max from DEFAULT
```

</details>

<details><summary>Challenge 3: Wire the histogram into AbstractTimer.record()</summary>

The key insight: record into the histogram BEFORE delegating to the subclass, and use nanoseconds for the histogram regardless of the caller's time unit:

```java
@Override
public final void record(long amount, TimeUnit unit) {
    if (amount >= 0) {
        histogram.recordLong(TimeUnit.NANOSECONDS.convert(amount, unit));
        recordNonNegative(amount, unit);
    }
}
```

</details>

---

## 10.6 Tests

### Unit Tests

**`DistributionStatisticConfigTest`** — validates the cascading merge pattern:

```java
@Test
void shouldSupportThreeLevelCascade_WhenMergedTwice() {
    DistributionStatisticConfig builder = DistributionStatisticConfig.builder()
            .serviceLevelObjectives(100, 500)
            .build();
    DistributionStatisticConfig filter = DistributionStatisticConfig.builder()
            .bufferLength(5)
            .build();

    DistributionStatisticConfig afterFilter = filter.merge(builder);
    DistributionStatisticConfig afterDefaults = afterFilter.merge(DistributionStatisticConfig.DEFAULT);

    assertThat(afterDefaults.getServiceLevelObjectives()).containsExactly(100, 500);
    assertThat(afterDefaults.getBufferLength()).isEqualTo(5);
    assertThat(afterDefaults.getExpiry()).isEqualTo(Duration.ofMinutes(2));
}
```

**`FixedBoundaryHistogramTest`** — validates binary search and cumulative counting:

```java
@Test
void shouldReturnCumulativeCounts_WhenCumulativeModeEnabled() {
    FixedBoundaryHistogram hist = new FixedBoundaryHistogram(
            new double[]{ 100, 500, 1000 }, true);

    hist.record(50);    // → bucket 100
    hist.record(200);   // → bucket 500
    hist.record(800);   // → bucket 1000

    CountAtBucket[] counts = hist.getCountAtBuckets();
    assertThat(counts[0].count()).isEqualTo(1);  // ≤ 100: 1
    assertThat(counts[1].count()).isEqualTo(2);  // ≤ 500: 1 + 1
    assertThat(counts[2].count()).isEqualTo(3);  // ≤ 1000: 1 + 1 + 1
}
```

**`TimeWindowFixedBoundaryHistogramTest`** — validates ring buffer decay:

```java
@Test
void shouldDecayOldValues_WhenTimeWindowExpires() {
    TimeWindowFixedBoundaryHistogram hist = new TimeWindowFixedBoundaryHistogram(
            clock, configWithSlos(100, 500, 1000));

    hist.recordLong(50);
    clock.add(Duration.ofMinutes(3));  // past the 2-minute expiry

    HistogramSnapshot snapshot = hist.takeSnapshot(1, 50, 50);
    CountAtBucket[] counts = snapshot.histogramCounts();
    assertThat(counts[0].count()).isEqualTo(0);  // decayed
}
```

### Integration Tests

**`DistributionStatisticsIntegrationTest`** — validates the full pipeline from Builder through Registry to Snapshot:

```java
@Test
void shouldProduceHistogramBuckets_WhenTimerHasSlos() {
    Timer timer = Timer.builder("http.requests")
            .serviceLevelObjectives(
                    Duration.ofMillis(100), Duration.ofMillis(500), Duration.ofSeconds(1))
            .register(registry);

    timer.record(50, TimeUnit.MILLISECONDS);
    timer.record(200, TimeUnit.MILLISECONDS);
    timer.record(800, TimeUnit.MILLISECONDS);

    HistogramSnapshot snapshot = timer.takeSnapshot();
    CountAtBucket[] counts = snapshot.histogramCounts();
    assertThat(counts[0].count()).isEqualTo(1);  // ≤ 100ms
    assertThat(counts[1].count()).isEqualTo(2);  // ≤ 500ms
    assertThat(counts[2].count()).isEqualTo(3);  // ≤ 1s
}

@Test
void shouldApplyFilterConfig_WhenFilterSetsSloBoundaries() {
    registry.meterFilter(new MeterFilter() {
        @Override
        public DistributionStatisticConfig configure(Meter.Id id, DistributionStatisticConfig config) {
            if (id.getName().startsWith("http")) {
                return DistributionStatisticConfig.builder()
                        .serviceLevelObjectives(
                                Duration.ofMillis(100).toNanos(),
                                Duration.ofMillis(500).toNanos(),
                                Duration.ofSeconds(1).toNanos())
                        .build();
            }
            return config;
        }
    });

    Timer timer = Timer.builder("http.requests").register(registry);
    timer.record(50, TimeUnit.MILLISECONDS);

    HistogramSnapshot snapshot = timer.takeSnapshot();
    assertThat(snapshot.histogramCounts()).hasSize(3);
    assertThat(snapshot.histogramCounts()[0].count()).isEqualTo(1);
}
```

Run all tests:

```bash
./gradlew test
```

All 385 tests pass — both new and all prior features' tests.

---

## 10.7 Why This Works

`★ Insight ─────────────────────────────────────`
**Write-all, read-one decay:** The ring buffer's key trick is that every `record()` writes to ALL slots, not just the current one. When a slot expires and is reset, its counts restart from zero. The `takeSnapshot()` reads only the current slot. Since each slot was reset at a different time, the current slot holds samples from at most one rotation interval — giving a histogram that naturally "forgets" old values without any explicit eviction logic. This is the same pattern used by `TimeWindowMax` (Feature 6), applied to an entire histogram.

**Why this is better than a simple reset:** A naive approach would be to reset the entire histogram every N minutes. But that creates gaps: the moment after reset, you have no data. The ring buffer ensures there's always at least `bufferLength - 1` slots with accumulated data, so the "current" slot always has a meaningful sample even right after rotation.
`─────────────────────────────────────────────────`

`★ Insight ─────────────────────────────────────`
**Nullable-field cascading merge:** The `DistributionStatisticConfig` uses nullable fields (not default values) specifically to support three-level configuration: Builder → MeterFilter → Registry defaults. If fields had default values, you couldn't distinguish "the user explicitly set bufferLength=3" from "the field has its default value of 3" — and a filter couldn't know whether to override. The null semantics mean "not configured at this level," enabling clean layered overrides. This pattern is common in Spring configuration (`@Nullable` properties merged via `BeanDefinition`) and CSS cascading.
`─────────────────────────────────────────────────`

`★ Insight ─────────────────────────────────────`
**Fixed-boundary vs. HdrHistogram trade-off:** Our simplified implementation uses fixed-boundary histograms (explicit bucket boundaries from SLO values). The real Micrometer also supports `TimeWindowPercentileHistogram` backed by HdrHistogram for precomputed percentiles (p50, p95, p99). The trade-off: fixed-boundary histograms are **aggregable across instances** (you can sum bucket counts from 100 pods), while precomputed percentiles are NOT (you can't average p99s). This is why Prometheus recommends histogram buckets over summary percentiles for distributed systems.
`─────────────────────────────────────────────────`

---

## 10.8 What We Enhanced

| File | Enhancement | Why |
|------|-------------|-----|
| `AbstractTimer` | Added `Histogram` field, records into it in `record()`, added `takeSnapshot()` | Centralizes histogram recording in the template method — every recording path gets histogram tracking |
| `AbstractDistributionSummary` | Same histogram integration as AbstractTimer | Mirrors the Timer pattern for non-time distributions |
| `CumulativeTimer` | New constructor accepting `DistributionStatisticConfig` | Creates the appropriate histogram via `defaultHistogram()` |
| `CumulativeDistributionSummary` | New constructor accepting `DistributionStatisticConfig` | Same pattern as CumulativeTimer |
| `Timer` interface | Added `takeSnapshot()` default method, Builder gains SLO/expiry/bufferLength config | Users can access histogram data; Builder configures the first cascade level |
| `DistributionSummary` interface | Added `takeSnapshot()` default method, Builder gains SLO/expiry/bufferLength config | Same as Timer |
| `MeterRegistry` | `newTimer()`/`newDistributionSummary()` gain `DistributionStatisticConfig` parameter; added `configureDistributionStatistics()` pipeline | Config flows through registry with filter chain processing |
| `SimpleMeterRegistry` | Updated factory methods to pass config | Passes merged config to CumulativeTimer/CumulativeDistributionSummary |
| `MeterFilter` | Added `configure(Meter.Id, DistributionStatisticConfig)` default method | Third stage in the filter chain — per-meter histogram configuration |
| `NoopTimer` / `NoopDistributionSummary` | Added `takeSnapshot()` returning empty snapshot | No-op meters still satisfy the interface contract |

---

## 10.9 Connection to Real Framework

| Simplified | Real Micrometer | File:Line (commit `2c8a4606c`) |
|-----------|-----------------|-------------------------------|
| `DistributionStatisticConfig` | `DistributionStatisticConfig` | `micrometer-core/.../distribution/DistributionStatisticConfig.java` |
| `Histogram` | `Histogram` | `micrometer-core/.../distribution/Histogram.java` |
| `HistogramSnapshot` | `HistogramSnapshot` | `micrometer-core/.../distribution/HistogramSnapshot.java` |
| `ValueAtPercentile` | `ValueAtPercentile` | `micrometer-core/.../distribution/ValueAtPercentile.java` |
| `CountAtBucket` | `CountAtBucket` | `micrometer-core/.../distribution/CountAtBucket.java` |
| `FixedBoundaryHistogram` | `FixedBoundaryHistogram` | `micrometer-core/.../distribution/FixedBoundaryHistogram.java` |
| `AbstractTimeWindowHistogram<T>` | `AbstractTimeWindowHistogram<T, U>` | `micrometer-core/.../distribution/AbstractTimeWindowHistogram.java` |
| `TimeWindowFixedBoundaryHistogram` | `TimeWindowFixedBoundaryHistogram` | `micrometer-core/.../distribution/TimeWindowFixedBoundaryHistogram.java` |
| `NoopHistogram` | `NoopHistogram` | `micrometer-core/.../distribution/NoopHistogram.java` |
| `MeterFilter.configure()` | `MeterFilter.configure()` | `micrometer-core/.../config/MeterFilter.java` |

**What we simplified:**
- Removed the `U` type parameter from `AbstractTimeWindowHistogram` (no accumulation needed without HdrHistogram)
- No `PercentileHistogramBuckets` (auto-generated bucket boundaries for aggregable percentile approximation)
- No `TimeWindowPercentileHistogram` (HdrHistogram-based precomputed percentiles)
- No pause detection / coordinated omission compensation
- No `percentileHistogram` boolean toggle (we only support explicit SLO boundaries)
- `Histogram` does not extend `AutoCloseable` (no resource cleanup needed)

---

## 10.10 Complete Code

### Production Code

#### [NEW] `src/main/java/dev/linhvu/micrometer/distribution/DistributionStatisticConfig.java`

```java
package dev.linhvu.micrometer.distribution;

import java.time.Duration;
import java.util.NavigableSet;
import java.util.TreeSet;

/**
 * Configuration for distribution statistics — percentiles, histogram bucket boundaries,
 * expected value ranges, and ring buffer settings.
 * <p>
 * Uses a <b>nullable-field cascading merge</b> pattern: each field is nullable, where
 * {@code null} means "not configured at this level." The {@link #merge(DistributionStatisticConfig)}
 * method fills in nulls from a parent config. This enables a three-layer cascade:
 * <ol>
 *   <li><b>Builder config</b> — user-specified settings</li>
 *   <li><b>MeterFilter.configure()</b> — per-meter overrides from the filter chain</li>
 *   <li><b>Registry defaults</b> — {@link #DEFAULT} fills any remaining gaps</li>
 * </ol>
 * <p>
 * <b>Simplification:</b> The real Micrometer also supports {@code percentileHistogram}
 * (auto-generated bucket boundaries for aggregable percentile approximation via
 * {@code PercentileHistogramBuckets}) and precomputed percentiles via HdrHistogram.
 * We only support fixed-boundary histograms from explicit SLO boundaries.
 */
public class DistributionStatisticConfig {

    /**
     * Default configuration — sensible values for all settings. Used as the final
     * fallback in the merge cascade.
     */
    public static final DistributionStatisticConfig DEFAULT = builder()
            .minimumExpectedValue(1.0)
            .maximumExpectedValue(Double.POSITIVE_INFINITY)
            .expiry(Duration.ofMinutes(2))
            .bufferLength(3)
            .build();

    /**
     * Empty configuration — all fields null. Represents "no settings configured."
     * Used as the starting point for builders.
     */
    public static final DistributionStatisticConfig NONE = builder().build();

    private final double[] percentiles;

    private final double[] serviceLevelObjectives;

    private final Double minimumExpectedValue;

    private final Double maximumExpectedValue;

    private final Duration expiry;

    private final Integer bufferLength;

    private DistributionStatisticConfig(Builder builder) {
        this.percentiles = builder.percentiles;
        this.serviceLevelObjectives = builder.serviceLevelObjectives;
        this.minimumExpectedValue = builder.minimumExpectedValue;
        this.maximumExpectedValue = builder.maximumExpectedValue;
        this.expiry = builder.expiry;
        this.bufferLength = builder.bufferLength;
    }

    /**
     * Creates a new config that prefers this config's values, falling back to the
     * parent's values for any field that is null in this config.
     */
    public DistributionStatisticConfig merge(DistributionStatisticConfig parent) {
        return builder()
                .percentiles(percentiles != null ? percentiles : parent.percentiles)
                .serviceLevelObjectives(
                        serviceLevelObjectives != null ? serviceLevelObjectives : parent.serviceLevelObjectives)
                .minimumExpectedValue(minimumExpectedValue != null ? minimumExpectedValue : parent.minimumExpectedValue)
                .maximumExpectedValue(maximumExpectedValue != null ? maximumExpectedValue : parent.maximumExpectedValue)
                .expiry(expiry != null ? expiry : parent.expiry)
                .bufferLength(bufferLength != null ? bufferLength : parent.bufferLength)
                .build();
    }

    public double[] getPercentiles() {
        return percentiles;
    }

    public double[] getServiceLevelObjectives() {
        return serviceLevelObjectives;
    }

    public Double getMinimumExpectedValue() {
        return minimumExpectedValue;
    }

    public Double getMaximumExpectedValue() {
        return maximumExpectedValue;
    }

    public Duration getExpiry() {
        return expiry;
    }

    public Integer getBufferLength() {
        return bufferLength;
    }

    public boolean isPublishingPercentiles() {
        return percentiles != null && percentiles.length > 0;
    }

    public boolean isPublishingHistogram() {
        return serviceLevelObjectives != null && serviceLevelObjectives.length > 0;
    }

    public NavigableSet<Double> getHistogramBuckets() {
        NavigableSet<Double> buckets = new TreeSet<>();
        if (serviceLevelObjectives != null) {
            for (double slo : serviceLevelObjectives) {
                buckets.add(slo);
            }
        }
        return buckets;
    }

    public static Builder builder() {
        return new Builder();
    }

    public static class Builder {

        private double[] percentiles;
        private double[] serviceLevelObjectives;
        private Double minimumExpectedValue;
        private Double maximumExpectedValue;
        private Duration expiry;
        private Integer bufferLength;

        Builder() {
        }

        public Builder percentiles(double... percentiles) {
            this.percentiles = percentiles;
            return this;
        }

        public Builder serviceLevelObjectives(double... slos) {
            this.serviceLevelObjectives = slos;
            return this;
        }

        public Builder minimumExpectedValue(Double value) {
            this.minimumExpectedValue = value;
            return this;
        }

        public Builder maximumExpectedValue(Double value) {
            this.maximumExpectedValue = value;
            return this;
        }

        public Builder expiry(Duration expiry) {
            this.expiry = expiry;
            return this;
        }

        public Builder bufferLength(Integer bufferLength) {
            this.bufferLength = bufferLength;
            return this;
        }

        public DistributionStatisticConfig build() {
            if (bufferLength != null && bufferLength <= 0) {
                throw new IllegalArgumentException("bufferLength must be positive");
            }
            if (minimumExpectedValue != null && minimumExpectedValue <= 0) {
                throw new IllegalArgumentException("minimumExpectedValue must be positive");
            }
            if (maximumExpectedValue != null && maximumExpectedValue <= 0) {
                throw new IllegalArgumentException("maximumExpectedValue must be positive");
            }
            if (minimumExpectedValue != null && maximumExpectedValue != null
                    && minimumExpectedValue > maximumExpectedValue) {
                throw new IllegalArgumentException("minimumExpectedValue must be <= maximumExpectedValue");
            }
            if (percentiles != null) {
                for (double p : percentiles) {
                    if (p < 0 || p > 1) {
                        throw new IllegalArgumentException(
                                "percentiles must be in [0, 1], got " + p);
                    }
                }
            }
            if (serviceLevelObjectives != null) {
                for (double slo : serviceLevelObjectives) {
                    if (slo <= 0) {
                        throw new IllegalArgumentException(
                                "SLO boundaries must be positive, got " + slo);
                    }
                }
            }
            return new DistributionStatisticConfig(this);
        }

    }

}
```

#### [NEW] `src/main/java/dev/linhvu/micrometer/distribution/Histogram.java`

```java
package dev.linhvu.micrometer.distribution;

/**
 * Records values into a histogram data structure and produces snapshots.
 */
public interface Histogram {

    void recordLong(long value);

    void recordDouble(double value);

    HistogramSnapshot takeSnapshot(long count, double total, double max);

}
```

#### [NEW] `src/main/java/dev/linhvu/micrometer/distribution/NoopHistogram.java`

```java
package dev.linhvu.micrometer.distribution;

public enum NoopHistogram implements Histogram {

    INSTANCE;

    @Override
    public void recordLong(long value) {
    }

    @Override
    public void recordDouble(double value) {
    }

    @Override
    public HistogramSnapshot takeSnapshot(long count, double total, double max) {
        return HistogramSnapshot.empty(count, total, max);
    }

}
```

#### [NEW] `src/main/java/dev/linhvu/micrometer/distribution/HistogramSnapshot.java`

```java
package dev.linhvu.micrometer.distribution;

import java.util.concurrent.TimeUnit;

public final class HistogramSnapshot {

    private static final ValueAtPercentile[] EMPTY_VALUES = new ValueAtPercentile[0];

    private static final CountAtBucket[] EMPTY_COUNTS = new CountAtBucket[0];

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
        this.percentileValues = percentileValues != null ? percentileValues : EMPTY_VALUES;
        this.histogramCounts = histogramCounts != null ? histogramCounts : EMPTY_COUNTS;
    }

    public static HistogramSnapshot empty(long count, double total, double max) {
        return new HistogramSnapshot(count, total, max, EMPTY_VALUES, EMPTY_COUNTS);
    }

    public long count() {
        return count;
    }

    public double total() {
        return total;
    }

    public double total(TimeUnit unit) {
        return total / unit.toNanos(1);
    }

    public double max() {
        return max;
    }

    public double max(TimeUnit unit) {
        return max / unit.toNanos(1);
    }

    public double mean() {
        return count == 0 ? 0 : total / count;
    }

    public double mean(TimeUnit unit) {
        return count == 0 ? 0 : total(unit) / count;
    }

    public ValueAtPercentile[] percentileValues() {
        return percentileValues;
    }

    public CountAtBucket[] histogramCounts() {
        return histogramCounts;
    }

    @Override
    public String toString() {
        StringBuilder sb = new StringBuilder("HistogramSnapshot{count=").append(count)
                .append(", total=").append(total)
                .append(", mean=").append(mean())
                .append(", max=").append(max);
        if (percentileValues.length > 0) {
            sb.append(", percentiles=[");
            for (int i = 0; i < percentileValues.length; i++) {
                if (i > 0)
                    sb.append(", ");
                sb.append(percentileValues[i]);
            }
            sb.append("]");
        }
        if (histogramCounts.length > 0) {
            sb.append(", histogramCounts=[");
            for (int i = 0; i < histogramCounts.length; i++) {
                if (i > 0)
                    sb.append(", ");
                sb.append(histogramCounts[i]);
            }
            sb.append("]");
        }
        sb.append("}");
        return sb.toString();
    }

}
```

#### [NEW] `src/main/java/dev/linhvu/micrometer/distribution/ValueAtPercentile.java`

```java
package dev.linhvu.micrometer.distribution;

import java.util.concurrent.TimeUnit;

public final class ValueAtPercentile {

    private final double percentile;

    private final double value;

    public ValueAtPercentile(double percentile, double value) {
        this.percentile = percentile;
        this.value = value;
    }

    public double percentile() {
        return percentile;
    }

    public double value() {
        return value;
    }

    public double value(TimeUnit unit) {
        return value / unit.toNanos(1);
    }

    @Override
    public String toString() {
        return String.format("(%.1f at %.1f%%)", value, percentile * 100);
    }

    @Override
    public boolean equals(Object o) {
        if (this == o)
            return true;
        if (!(o instanceof ValueAtPercentile other))
            return false;
        return Double.compare(other.percentile, percentile) == 0
                && Double.compare(other.value, value) == 0;
    }

    @Override
    public int hashCode() {
        long temp = Double.doubleToLongBits(percentile);
        int result = (int) (temp ^ (temp >>> 32));
        temp = Double.doubleToLongBits(value);
        result = 31 * result + (int) (temp ^ (temp >>> 32));
        return result;
    }

}
```

#### [NEW] `src/main/java/dev/linhvu/micrometer/distribution/CountAtBucket.java`

```java
package dev.linhvu.micrometer.distribution;

import java.util.concurrent.TimeUnit;

public final class CountAtBucket {

    private final double bucket;

    private final double count;

    public CountAtBucket(double bucket, double count) {
        this.bucket = bucket;
        this.count = count;
    }

    public double bucket() {
        return bucket;
    }

    public double bucket(TimeUnit unit) {
        return bucket / unit.toNanos(1);
    }

    public double count() {
        return count;
    }

    @Override
    public String toString() {
        return String.format("(%.1f at %.1f)", count, bucket);
    }

    @Override
    public boolean equals(Object o) {
        if (this == o)
            return true;
        if (!(o instanceof CountAtBucket other))
            return false;
        return Double.compare(other.bucket, bucket) == 0
                && Double.compare(other.count, count) == 0;
    }

    @Override
    public int hashCode() {
        long temp = Double.doubleToLongBits(bucket);
        int result = (int) (temp ^ (temp >>> 32));
        temp = Double.doubleToLongBits(count);
        result = 31 * result + (int) (temp ^ (temp >>> 32));
        return result;
    }

}
```

#### [NEW] `src/main/java/dev/linhvu/micrometer/distribution/FixedBoundaryHistogram.java`

```java
package dev.linhvu.micrometer.distribution;

import java.util.concurrent.atomic.AtomicLongArray;

class FixedBoundaryHistogram {

    private final double[] buckets;

    private final AtomicLongArray values;

    private final boolean isCumulativeBucketCounts;

    FixedBoundaryHistogram(double[] buckets, boolean isCumulativeBucketCounts) {
        this.buckets = buckets;
        this.values = new AtomicLongArray(buckets.length);
        this.isCumulativeBucketCounts = isCumulativeBucketCounts;
    }

    void record(long value) {
        int index = leastLessThanOrEqualTo(value);
        if (index >= 0 && index < values.length()) {
            values.incrementAndGet(index);
        }
    }

    CountAtBucket[] getCountAtBuckets() {
        CountAtBucket[] counts = new CountAtBucket[buckets.length];
        long cumulativeCount = 0;
        for (int i = 0; i < buckets.length; i++) {
            long count = values.get(i);
            if (isCumulativeBucketCounts) {
                cumulativeCount += count;
                counts[i] = new CountAtBucket(buckets[i], cumulativeCount);
            }
            else {
                counts[i] = new CountAtBucket(buckets[i], count);
            }
        }
        return counts;
    }

    void reset() {
        for (int i = 0; i < values.length(); i++) {
            values.set(i, 0);
        }
    }

    private int leastLessThanOrEqualTo(long value) {
        int low = 0;
        int high = buckets.length - 1;
        while (low <= high) {
            int mid = (low + high) >>> 1;
            if (buckets[mid] < value) {
                low = mid + 1;
            }
            else if (buckets[mid] > value) {
                high = mid - 1;
            }
            else {
                return mid;
            }
        }
        return low;
    }

}
```

#### [NEW] `src/main/java/dev/linhvu/micrometer/distribution/AbstractTimeWindowHistogram.java`

```java
package dev.linhvu.micrometer.distribution;

import dev.linhvu.micrometer.Clock;

import java.lang.reflect.Array;

public abstract class AbstractTimeWindowHistogram<T> implements Histogram {

    final DistributionStatisticConfig distributionStatisticConfig;

    private final Clock clock;

    private final T[] ringBuffer;

    private int currentBucket;

    private final long durationBetweenRotatesMillis;

    private volatile long lastRotateTimestampMillis;

    private volatile boolean rotating;

    @SuppressWarnings("unchecked")
    protected AbstractTimeWindowHistogram(Clock clock,
            DistributionStatisticConfig config,
            Class<T> bucketType) {
        this.distributionStatisticConfig = config;
        this.clock = clock;

        int bufferLength = config.getBufferLength();
        this.ringBuffer = (T[]) Array.newInstance(bucketType, bufferLength);

        long expiryMillis = config.getExpiry().toMillis();
        this.durationBetweenRotatesMillis = expiryMillis / bufferLength;
        if (durationBetweenRotatesMillis <= 0) {
            throw new IllegalArgumentException(
                    "expiry / bufferLength must result in a positive rotation interval, "
                            + "got expiry=" + config.getExpiry() + " bufferLength=" + bufferLength);
        }

        this.currentBucket = 0;
        this.lastRotateTimestampMillis = clock.wallTime();
    }

    protected void initRingBuffer() {
        for (int i = 0; i < ringBuffer.length; i++) {
            ringBuffer[i] = newBucket();
        }
    }

    protected abstract T newBucket();

    protected abstract void recordLong(T bucket, long value);

    protected abstract void recordDouble(T bucket, double value);

    protected abstract void resetBucket(T bucket);

    protected abstract CountAtBucket[] countsAtBuckets(T currentBucket);

    @Override
    public void recordLong(long value) {
        rotate();
        for (T bucket : ringBuffer) {
            recordLong(bucket, value);
        }
    }

    @Override
    public void recordDouble(double value) {
        rotate();
        for (T bucket : ringBuffer) {
            recordDouble(bucket, value);
        }
    }

    @Override
    public synchronized HistogramSnapshot takeSnapshot(long count, double total, double max) {
        rotate();
        CountAtBucket[] counts = countsAtBuckets(ringBuffer[currentBucket]);
        return new HistogramSnapshot(count, total, max, null, counts);
    }

    protected T currentHistogram() {
        return ringBuffer[currentBucket];
    }

    private void rotate() {
        long wallTime = clock.wallTime();
        long timeSinceLastRotateMillis = wallTime - lastRotateTimestampMillis;
        if (timeSinceLastRotateMillis < durationBetweenRotatesMillis) {
            return;
        }

        if (rotating) {
            return;
        }
        rotating = true;

        try {
            synchronized (this) {
                timeSinceLastRotateMillis = wallTime - lastRotateTimestampMillis;
                if (timeSinceLastRotateMillis < durationBetweenRotatesMillis) {
                    return;
                }

                if (timeSinceLastRotateMillis >= durationBetweenRotatesMillis * ringBuffer.length) {
                    for (T bucket : ringBuffer) {
                        resetBucket(bucket);
                    }
                    currentBucket = 0;
                    lastRotateTimestampMillis = wallTime
                            - timeSinceLastRotateMillis % durationBetweenRotatesMillis;
                    return;
                }

                int iterations = 0;
                do {
                    resetBucket(ringBuffer[currentBucket]);
                    if (++currentBucket >= ringBuffer.length) {
                        currentBucket = 0;
                    }
                    timeSinceLastRotateMillis -= durationBetweenRotatesMillis;
                    lastRotateTimestampMillis += durationBetweenRotatesMillis;
                }
                while (timeSinceLastRotateMillis >= durationBetweenRotatesMillis
                        && ++iterations < ringBuffer.length);
            }
        }
        finally {
            rotating = false;
        }
    }

}
```

#### [NEW] `src/main/java/dev/linhvu/micrometer/distribution/TimeWindowFixedBoundaryHistogram.java`

```java
package dev.linhvu.micrometer.distribution;

import dev.linhvu.micrometer.Clock;

import java.util.NavigableSet;

public class TimeWindowFixedBoundaryHistogram extends AbstractTimeWindowHistogram<FixedBoundaryHistogram> {

    private final double[] buckets;

    private final boolean isCumulativeBucketCounts;

    public TimeWindowFixedBoundaryHistogram(Clock clock, DistributionStatisticConfig config) {
        this(clock, config, true);
    }

    public TimeWindowFixedBoundaryHistogram(Clock clock, DistributionStatisticConfig config,
            boolean isCumulativeBucketCounts) {
        super(clock, config, FixedBoundaryHistogram.class);
        this.isCumulativeBucketCounts = isCumulativeBucketCounts;

        NavigableSet<Double> bucketSet = config.getHistogramBuckets();
        this.buckets = bucketSet.stream().mapToDouble(Double::doubleValue).toArray();

        initRingBuffer();
    }

    @Override
    protected FixedBoundaryHistogram newBucket() {
        return new FixedBoundaryHistogram(buckets, isCumulativeBucketCounts);
    }

    @Override
    protected void recordLong(FixedBoundaryHistogram bucket, long value) {
        bucket.record(value);
    }

    @Override
    protected void recordDouble(FixedBoundaryHistogram bucket, double value) {
        recordLong(bucket, (long) Math.ceil(value));
    }

    @Override
    protected void resetBucket(FixedBoundaryHistogram bucket) {
        bucket.reset();
    }

    @Override
    protected CountAtBucket[] countsAtBuckets(FixedBoundaryHistogram currentBucket) {
        return currentBucket.getCountAtBuckets();
    }

}
```

#### [MODIFIED] `src/main/java/dev/linhvu/micrometer/AbstractTimer.java`

```java
package dev.linhvu.micrometer;

import dev.linhvu.micrometer.distribution.DistributionStatisticConfig;
import dev.linhvu.micrometer.distribution.Histogram;
import dev.linhvu.micrometer.distribution.HistogramSnapshot;
import dev.linhvu.micrometer.distribution.NoopHistogram;
import dev.linhvu.micrometer.distribution.TimeWindowFixedBoundaryHistogram;

import java.util.concurrent.Callable;
import java.util.concurrent.TimeUnit;
import java.util.function.Supplier;

public abstract class AbstractTimer extends AbstractMeter implements Timer {

    protected final Clock clock;

    private final TimeUnit baseTimeUnit;

    private final Histogram histogram;

    protected AbstractTimer(Meter.Id id, Clock clock, TimeUnit baseTimeUnit) {
        this(id, clock, baseTimeUnit, NoopHistogram.INSTANCE);
    }

    protected AbstractTimer(Meter.Id id, Clock clock, TimeUnit baseTimeUnit, Histogram histogram) {
        super(id);
        this.clock = clock;
        this.baseTimeUnit = baseTimeUnit;
        this.histogram = (histogram != null) ? histogram : NoopHistogram.INSTANCE;
    }

    protected static Histogram defaultHistogram(Clock clock, DistributionStatisticConfig config) {
        if (config.isPublishingHistogram()) {
            return new TimeWindowFixedBoundaryHistogram(clock, config);
        }
        return NoopHistogram.INSTANCE;
    }

    @Override
    public final void record(long amount, TimeUnit unit) {
        if (amount >= 0) {
            histogram.recordLong(TimeUnit.NANOSECONDS.convert(amount, unit));
            recordNonNegative(amount, unit);
        }
    }

    @Override
    public HistogramSnapshot takeSnapshot() {
        return histogram.takeSnapshot(count(), totalTime(TimeUnit.NANOSECONDS), max(TimeUnit.NANOSECONDS));
    }

    protected abstract void recordNonNegative(long amount, TimeUnit unit);

    @Override
    public <T> T record(Supplier<T> f) {
        final long s = clock.monotonicTime();
        try {
            return f.get();
        }
        finally {
            final long e = clock.monotonicTime();
            record(e - s, TimeUnit.NANOSECONDS);
        }
    }

    @Override
    public <T> T recordCallable(Callable<T> f) throws Exception {
        final long s = clock.monotonicTime();
        try {
            return f.call();
        }
        finally {
            final long e = clock.monotonicTime();
            record(e - s, TimeUnit.NANOSECONDS);
        }
    }

    @Override
    public void record(Runnable f) {
        final long s = clock.monotonicTime();
        try {
            f.run();
        }
        finally {
            final long e = clock.monotonicTime();
            record(e - s, TimeUnit.NANOSECONDS);
        }
    }

    @Override
    public TimeUnit baseTimeUnit() {
        return baseTimeUnit;
    }
}
```

#### [MODIFIED] `src/main/java/dev/linhvu/micrometer/AbstractDistributionSummary.java`

```java
package dev.linhvu.micrometer;

import dev.linhvu.micrometer.distribution.DistributionStatisticConfig;
import dev.linhvu.micrometer.distribution.Histogram;
import dev.linhvu.micrometer.distribution.HistogramSnapshot;
import dev.linhvu.micrometer.distribution.NoopHistogram;
import dev.linhvu.micrometer.distribution.TimeWindowFixedBoundaryHistogram;

public abstract class AbstractDistributionSummary extends AbstractMeter implements DistributionSummary {

    private final double scale;

    private final Histogram histogram;

    protected AbstractDistributionSummary(Meter.Id id, double scale) {
        this(id, scale, NoopHistogram.INSTANCE);
    }

    protected AbstractDistributionSummary(Meter.Id id, double scale, Histogram histogram) {
        super(id);
        this.scale = scale;
        this.histogram = (histogram != null) ? histogram : NoopHistogram.INSTANCE;
    }

    protected static Histogram defaultHistogram(Clock clock, DistributionStatisticConfig config) {
        if (config.isPublishingHistogram()) {
            return new TimeWindowFixedBoundaryHistogram(clock, config);
        }
        return NoopHistogram.INSTANCE;
    }

    @Override
    public final void record(double amount) {
        if (amount >= 0) {
            double scaledAmount = this.scale * amount;
            histogram.recordDouble(scaledAmount);
            recordNonNegative(scaledAmount);
        }
    }

    @Override
    public HistogramSnapshot takeSnapshot() {
        return histogram.takeSnapshot(count(), totalAmount(), max());
    }

    protected abstract void recordNonNegative(double amount);

}
```

#### [MODIFIED] `src/main/java/dev/linhvu/micrometer/cumulative/CumulativeTimer.java`

```java
package dev.linhvu.micrometer.cumulative;

import dev.linhvu.micrometer.AbstractTimer;
import dev.linhvu.micrometer.Clock;
import dev.linhvu.micrometer.Meter;
import dev.linhvu.micrometer.distribution.DistributionStatisticConfig;
import dev.linhvu.micrometer.distribution.TimeWindowMax;

import java.util.concurrent.TimeUnit;
import java.util.concurrent.atomic.LongAdder;

public class CumulativeTimer extends AbstractTimer {

    private final LongAdder count;

    private final LongAdder total;

    private final TimeWindowMax max;

    public CumulativeTimer(Meter.Id id, Clock clock, TimeUnit baseTimeUnit) {
        super(id, clock, baseTimeUnit);
        this.count = new LongAdder();
        this.total = new LongAdder();
        this.max = new TimeWindowMax(clock);
    }

    public CumulativeTimer(Meter.Id id, Clock clock, TimeUnit baseTimeUnit,
            DistributionStatisticConfig config) {
        super(id, clock, baseTimeUnit, AbstractTimer.defaultHistogram(clock, config));
        this.count = new LongAdder();
        this.total = new LongAdder();
        this.max = new TimeWindowMax(clock);
    }

    @Override
    protected void recordNonNegative(long amount, TimeUnit unit) {
        long nanoAmount = unit.toNanos(amount);
        count.increment();
        total.add(nanoAmount);
        max.record((double) nanoAmount, TimeUnit.NANOSECONDS);
    }

    @Override
    public long count() {
        return count.longValue();
    }

    @Override
    public double totalTime(TimeUnit unit) {
        return nanosToUnit(total.doubleValue(), unit);
    }

    @Override
    public double max(TimeUnit unit) {
        return max.poll(unit);
    }

    private static double nanosToUnit(double nanos, TimeUnit unit) {
        return nanos / unit.toNanos(1);
    }
}
```

#### [MODIFIED] `src/main/java/dev/linhvu/micrometer/cumulative/CumulativeDistributionSummary.java`

```java
package dev.linhvu.micrometer.cumulative;

import dev.linhvu.micrometer.AbstractDistributionSummary;
import dev.linhvu.micrometer.Clock;
import dev.linhvu.micrometer.Measurement;
import dev.linhvu.micrometer.Meter;
import dev.linhvu.micrometer.Statistic;
import dev.linhvu.micrometer.distribution.DistributionStatisticConfig;
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

    public CumulativeDistributionSummary(Meter.Id id, Clock clock, double scale,
            DistributionStatisticConfig config) {
        super(id, scale, AbstractDistributionSummary.defaultHistogram(clock, config));
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

#### [MODIFIED] `src/main/java/dev/linhvu/micrometer/config/MeterFilter.java`

Added `configure()` default method (other methods unchanged):

```java
/**
 * Configures the distribution statistics for a meter.
 */
default DistributionStatisticConfig configure(Meter.Id id, DistributionStatisticConfig config) {
    return config;
}
```

#### [MODIFIED] `src/main/java/dev/linhvu/micrometer/MeterRegistry.java`

Key changes — updated `timer()` and `summary()` registration methods, updated abstract factory signatures, added `configureDistributionStatistics()` pipeline. See the full source file for complete code.

#### [MODIFIED] `src/main/java/dev/linhvu/micrometer/simple/SimpleMeterRegistry.java`

Updated factory methods:

```java
@Override
protected Timer newTimer(Meter.Id id, DistributionStatisticConfig distributionStatisticConfig) {
    return new CumulativeTimer(id, clock, getBaseTimeUnit(), distributionStatisticConfig);
}

@Override
protected DistributionSummary newDistributionSummary(Meter.Id id, double scale,
        DistributionStatisticConfig distributionStatisticConfig) {
    return new CumulativeDistributionSummary(id, clock, scale, distributionStatisticConfig);
}
```

### Test Code

#### `src/test/java/dev/linhvu/micrometer/distribution/DistributionStatisticConfigTest.java`

Tests cascading merge, publishing flags, histogram bucket generation, and builder validation. (16 tests)

#### `src/test/java/dev/linhvu/micrometer/distribution/HistogramSnapshotTest.java`

Tests basic stats, time unit conversions, and data inclusion. (7 tests)

#### `src/test/java/dev/linhvu/micrometer/distribution/FixedBoundaryHistogramTest.java`

Tests binary search placement, cumulative/non-cumulative modes, reset, and concurrency. (9 tests)

#### `src/test/java/dev/linhvu/micrometer/distribution/TimeWindowFixedBoundaryHistogramTest.java`

Tests ring buffer rotation, write-all/read-one decay, and edge cases. (8 tests)

#### `src/test/java/dev/linhvu/micrometer/integration/DistributionStatisticsIntegrationTest.java`

Tests the full pipeline: Timer+histogram, Summary+histogram, MeterFilter.configure(), decay through the stack, and deduplication. (8 tests)

---

## Summary

| What | Details |
|------|---------|
| New classes | 9 (`DistributionStatisticConfig`, `Histogram`, `NoopHistogram`, `HistogramSnapshot`, `ValueAtPercentile`, `CountAtBucket`, `FixedBoundaryHistogram`, `AbstractTimeWindowHistogram`, `TimeWindowFixedBoundaryHistogram`) |
| Modified classes | 11 (`AbstractTimer`, `AbstractDistributionSummary`, `CumulativeTimer`, `CumulativeDistributionSummary`, `Timer`, `DistributionSummary`, `MeterRegistry`, `SimpleMeterRegistry`, `MeterFilter`, `NoopTimer`, `NoopDistributionSummary`) |
| Tests | 48 new tests across 5 test files |
| Total tests | 385 (all passing) |
| Key patterns | Nullable-field cascading merge, write-all/read-one ring buffer, Template Method histogram integration, binary search bucket placement |

## Next Chapter Preview

**Chapter 11: Function Meters** — wraps existing counters/timers from other libraries via `FunctionCounter`, `FunctionTimer`, and `TimeGauge`. These meters derive values from external objects via `WeakReference` + `ToDoubleFunction`, enabling instrumentation of libraries that already maintain their own statistics (JMX beans, Kafka metrics, connection pools).
