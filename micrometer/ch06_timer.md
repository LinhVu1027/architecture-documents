# Chapter 6: Timer

> Line references based on commit `2c8a4606c` of the Micrometer repository.

## Build Challenge

| Current State | Limitation | Objective |
|---------------|------------|-----------|
| Registry supports Counter (monotonically increasing) and Gauge (point-in-time sampling), but cannot measure **duration** of events | No way to record how long operations take — the most common metric in production systems (HTTP latency, DB query time, etc.) | Build a `Timer` that records duration and frequency with three statistics (COUNT, TOTAL_TIME, MAX), a decaying maximum via ring buffer, and a start/stop `Sample` pattern for cross-method timing |

---

## 6.1 The Integration Point

The Timer plugs into the registry exactly like Counter and Gauge did — through the Builder → `registry.timer(Id)` → `newTimer()` template method chain.

**Modifying:** `src/main/java/dev/linhvu/micrometer/MeterRegistry.java`
**Change:** Add public `timer()` convenience methods and a package-private `timer(Meter.Id)` registration method

```java
import dev.linhvu.micrometer.noop.NoopTimer;

// ... existing code ...

// --- Public convenience registration methods (add after gaugeMapSize) ---

public Timer timer(String name, String... tags) {
    return Timer.builder(name).tags(tags).register(this);
}

public Timer timer(String name, Iterable<Tag> tags) {
    return Timer.builder(name).tags(tags).register(this);
}

// --- Package-private registration (add after gauge(Meter.Id, ...)) ---

Timer timer(Meter.Id id) {
    return registerMeterIfNecessary(Timer.class, id, this::newTimer, NoopTimer::new);
}
```

Two key decisions here:

1. **Same pattern as Counter/Gauge** — the registration lifecycle (deduplication, type checking, noop fallback) is identical. Timer just plugs into the existing `registerMeterIfNecessary()` infrastructure.
2. **No base unit in Builder** — unlike Counter which allows a custom `baseUnit`, a timer's time unit is determined by the *registry*, not the user. This is why `Timer.Builder.register()` passes `null` for baseUnit — `SimpleMeterRegistry` decides it's `SECONDS`, while Prometheus would use `SECONDS` too, and other backends might use `MILLISECONDS`.

This integration connects **the Timer API** to **the MeterRegistry lifecycle**. To make it work, we need to build:
- `TimeWindowMax` — a ring-buffer-based decaying maximum (novel data structure)
- `Timer` interface — with record methods, Sample, and Builder
- `AbstractTimer` — template method for recording with validation
- `CumulativeTimer` — concrete implementation with LongAdder + TimeWindowMax
- `NoopTimer` — silent fallback for closed registries

---

## 6.2 TimeWindowMax — The Decaying Maximum

Before building the Timer itself, we need the data structure that tracks its maximum. Unlike COUNT and TOTAL (which grow forever), MAX should "forget" old spikes — a 10-second request from an hour ago shouldn't dominate your dashboard.

**New file:** `src/main/java/dev/linhvu/micrometer/distribution/TimeWindowMax.java`

```java
package dev.linhvu.micrometer.distribution;

import dev.linhvu.micrometer.Clock;

import java.util.concurrent.TimeUnit;
import java.util.concurrent.atomic.AtomicLong;

public class TimeWindowMax {

    private static final long DEFAULT_ROTATE_FREQUENCY_MILLIS = 40_000;
    private static final int DEFAULT_BUFFER_LENGTH = 3;

    private final Clock clock;
    private final long durationBetweenRotatesMillis;
    private final AtomicLong[] ringBuffer;
    private int currentBucket;
    private volatile long lastRotateTimestampMillis;
    private volatile boolean rotating;

    public TimeWindowMax(Clock clock) {
        this(clock, DEFAULT_ROTATE_FREQUENCY_MILLIS, DEFAULT_BUFFER_LENGTH);
    }

    public TimeWindowMax(Clock clock, long rotateFrequencyMillis, int bufferLength) {
        if (rotateFrequencyMillis <= 0) {
            throw new IllegalArgumentException("Rotate frequency must be positive");
        }
        this.clock = clock;
        this.durationBetweenRotatesMillis = rotateFrequencyMillis;
        this.lastRotateTimestampMillis = clock.wallTime();
        this.currentBucket = 0;
        this.ringBuffer = new AtomicLong[bufferLength];
        for (int i = 0; i < bufferLength; i++) {
            this.ringBuffer[i] = new AtomicLong();
        }
    }

    // Timer mode: store as nanos
    public void record(double sample, TimeUnit timeUnit) {
        record(timeUnit.toNanos((long) sample));
    }

    // Distribution summary mode: store via doubleToLongBits
    public void record(double sample) {
        record(Double.doubleToLongBits(sample));
    }

    private void record(long sample) {
        rotate();
        for (AtomicLong max : ringBuffer) {
            updateMax(max, sample);
        }
    }

    public double poll(TimeUnit timeUnit) {
        rotate();
        synchronized (this) {
            double nanos = (double) ringBuffer[currentBucket].get();
            return nanos / timeUnit.toNanos(1);
        }
    }

    public double poll() {
        rotate();
        synchronized (this) {
            return Double.longBitsToDouble(ringBuffer[currentBucket].get());
        }
    }

    private void updateMax(AtomicLong max, long sample) {
        long curMax;
        do {
            curMax = max.get();
        }
        while (curMax < sample && !max.compareAndSet(curMax, sample));
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
                    for (AtomicLong bufferItem : ringBuffer) {
                        bufferItem.set(0);
                    }
                    currentBucket = 0;
                    lastRotateTimestampMillis = wallTime - timeSinceLastRotateMillis % durationBetweenRotatesMillis;
                    return;
                }
                int iterations = 0;
                do {
                    ringBuffer[currentBucket].set(0);
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

The "write-all, read-one" pattern is the core insight:

```
Ring buffer: [ Bucket0 ] [ Bucket1 ] [ Bucket2 ]
                                        ↑ currentBucket

record(500ms):  updates ALL →  [500]     [500]     [500]
record(200ms):  updates ALL →  [500]     [500]     [500]  (200 < 500, no update)

After 1 rotation:              [  0 ]    [500]     [500]
                                  ↑ currentBucket moves

poll() reads only currentBucket → 500  (still available in rotated-to bucket)

After ALL rotations:           [  0 ]    [  0 ]    [  0 ]
poll() → 0  (max has decayed away)
```

---

## 6.3 Timer Interface, AbstractTimer, and CumulativeTimer

### Timer Interface

**Modifying:** `src/main/java/dev/linhvu/micrometer/Timer.java`
**Change:** Replace the empty stub interface with the full Timer API

```java
package dev.linhvu.micrometer;

import java.time.Duration;
import java.util.Arrays;
import java.util.concurrent.Callable;
import java.util.concurrent.TimeUnit;
import java.util.function.Supplier;

public interface Timer extends Meter {

    static Sample start() { return start(Clock.SYSTEM); }
    static Sample start(MeterRegistry registry) { return start(registry.getClock()); }
    static Sample start(Clock clock) { return new Sample(clock); }
    static Builder builder(String name) { return new Builder(name); }

    void record(long amount, TimeUnit unit);

    default void record(Duration duration) {
        record(duration.toNanos(), TimeUnit.NANOSECONDS);
    }

    <T> T record(Supplier<T> f);
    <T> T recordCallable(Callable<T> f) throws Exception;
    void record(Runnable f);

    long count();
    double totalTime(TimeUnit unit);
    double max(TimeUnit unit);
    TimeUnit baseTimeUnit();

    default double mean(TimeUnit unit) {
        long count = count();
        return count == 0 ? 0 : totalTime(unit) / count;
    }

    @Override
    default Iterable<Measurement> measure() {
        return Arrays.asList(
                new Measurement(() -> (double) count(), Statistic.COUNT),
                new Measurement(() -> totalTime(baseTimeUnit()), Statistic.TOTAL_TIME),
                new Measurement(() -> max(baseTimeUnit()), Statistic.MAX));
    }

    class Sample {
        private final Clock clock;
        private final long startTime;

        Sample(Clock clock) {
            this.clock = clock;
            this.startTime = clock.monotonicTime();
        }

        public long stop(Timer timer) {
            long durationNs = clock.monotonicTime() - startTime;
            timer.record(durationNs, TimeUnit.NANOSECONDS);
            return durationNs;
        }
    }

    class Builder {
        private final String name;
        private Tags tags = Tags.empty();
        private String description;

        private Builder(String name) { this.name = name; }

        public Builder tags(String... tags) { return tags(Tags.of(tags)); }
        public Builder tags(Iterable<Tag> tags) { this.tags = this.tags.and(tags); return this; }
        public Builder tag(String key, String value) { this.tags = tags.and(key, value); return this; }
        public Builder description(String description) { this.description = description; return this; }

        public Timer register(MeterRegistry registry) {
            return registry.timer(
                    new Meter.Id(name, tags, Meter.Type.TIMER, description, null));
        }

        public String getName() { return name; }
        public Tags getTags() { return tags; }
        public String getDescription() { return description; }
    }
}
```

### AbstractTimer

**New file:** `src/main/java/dev/linhvu/micrometer/AbstractTimer.java`

```java
package dev.linhvu.micrometer;

import java.util.concurrent.Callable;
import java.util.concurrent.TimeUnit;
import java.util.function.Supplier;

public abstract class AbstractTimer extends AbstractMeter implements Timer {

    protected final Clock clock;
    private final TimeUnit baseTimeUnit;

    protected AbstractTimer(Meter.Id id, Clock clock, TimeUnit baseTimeUnit) {
        super(id);
        this.clock = clock;
        this.baseTimeUnit = baseTimeUnit;
    }

    @Override
    public final void record(long amount, TimeUnit unit) {
        if (amount >= 0) {
            recordNonNegative(amount, unit);
        }
    }

    protected abstract void recordNonNegative(long amount, TimeUnit unit);

    @Override
    public <T> T record(Supplier<T> f) {
        final long s = clock.monotonicTime();
        try {
            return f.get();
        } finally {
            final long e = clock.monotonicTime();
            record(e - s, TimeUnit.NANOSECONDS);
        }
    }

    @Override
    public <T> T recordCallable(Callable<T> f) throws Exception {
        final long s = clock.monotonicTime();
        try {
            return f.call();
        } finally {
            final long e = clock.monotonicTime();
            record(e - s, TimeUnit.NANOSECONDS);
        }
    }

    @Override
    public void record(Runnable f) {
        final long s = clock.monotonicTime();
        try {
            f.run();
        } finally {
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

### CumulativeTimer

**New file:** `src/main/java/dev/linhvu/micrometer/cumulative/CumulativeTimer.java`

```java
package dev.linhvu.micrometer.cumulative;

import dev.linhvu.micrometer.AbstractTimer;
import dev.linhvu.micrometer.Clock;
import dev.linhvu.micrometer.Meter;
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

    @Override
    protected void recordNonNegative(long amount, TimeUnit unit) {
        long nanoAmount = unit.toNanos(amount);
        count.increment();
        total.add(nanoAmount);
        max.record((double) nanoAmount, TimeUnit.NANOSECONDS);
    }

    @Override
    public long count() { return count.longValue(); }

    @Override
    public double totalTime(TimeUnit unit) {
        return total.doubleValue() / unit.toNanos(1);
    }

    @Override
    public double max(TimeUnit unit) { return max.poll(unit); }
}
```

## 6.4 NoopTimer and Wiring

### NoopTimer

**New file:** `src/main/java/dev/linhvu/micrometer/noop/NoopTimer.java`

```java
package dev.linhvu.micrometer.noop;

import dev.linhvu.micrometer.Meter;
import dev.linhvu.micrometer.Timer;
import java.util.concurrent.Callable;
import java.util.concurrent.TimeUnit;
import java.util.function.Supplier;

public class NoopTimer extends NoopMeter implements Timer {

    public NoopTimer(Meter.Id id) { super(id); }

    @Override public void record(long amount, TimeUnit unit) { }
    @Override public <T> T record(Supplier<T> f) { return f.get(); }
    @Override public <T> T recordCallable(Callable<T> f) throws Exception { return f.call(); }
    @Override public void record(Runnable f) { f.run(); }
    @Override public long count() { return 0; }
    @Override public double totalTime(TimeUnit unit) { return 0; }
    @Override public double max(TimeUnit unit) { return 0; }
    @Override public TimeUnit baseTimeUnit() { return TimeUnit.SECONDS; }
}
```

### SimpleMeterRegistry Update

**Modifying:** `src/main/java/dev/linhvu/micrometer/simple/SimpleMeterRegistry.java`
**Change:** Replace the `newTimer()` stub with a real implementation

```java
// Add import:
import dev.linhvu.micrometer.cumulative.CumulativeTimer;

// Replace newTimer():
@Override
protected Timer newTimer(Meter.Id id) {
    return new CumulativeTimer(id, clock, getBaseTimeUnit());
}
```

**Run:** `./gradlew test` — expected: all tests pass (including all prior features' tests)

---

## 6.5 Try It Yourself

<details>
<summary>Challenge: Implement the Timer.Sample start/stop pattern</summary>

The Sample captures a start time from the clock, then when `stop()` is called, it computes the elapsed duration and records it to a timer. The trick: the Timer isn't provided until `stop()`, allowing you to decide the timer's tags late (e.g., based on the HTTP response status).

```java
class Sample {
    private final Clock clock;
    private final long startTime;

    Sample(Clock clock) {
        this.clock = clock;
        this.startTime = clock.monotonicTime();
    }

    public long stop(Timer timer) {
        long durationNs = clock.monotonicTime() - startTime;
        timer.record(durationNs, TimeUnit.NANOSECONDS);
        return durationNs;
    }
}
```

Why `monotonicTime()` and not `wallTime()`? Wall-clock time can jump backwards (NTP adjustments), which would give negative durations. Monotonic time only moves forward.

</details>

<details>
<summary>Challenge: Implement the lock-free CAS maximum update in TimeWindowMax</summary>

The `updateMax` method needs to atomically update an `AtomicLong` to the larger of its current value and a new sample — without locks.

```java
private void updateMax(AtomicLong max, long sample) {
    long curMax;
    do {
        curMax = max.get();
    }
    while (curMax < sample && !max.compareAndSet(curMax, sample));
}
```

The CAS loop terminates in two cases:
1. `curMax >= sample` — the existing max is already larger, nothing to do
2. `compareAndSet` succeeds — we updated the value atomically

If another thread modifies `max` between `get()` and `compareAndSet()`, the CAS fails and we retry with the new value. This is classic lock-free programming.

</details>

<details>
<summary>Challenge: Make record(long, TimeUnit) final in AbstractTimer</summary>

Why should `record(long, TimeUnit)` be `final`? Because it's the validation gateway — negative values must be dropped regardless of the subclass implementation.

```java
@Override
public final void record(long amount, TimeUnit unit) {
    if (amount >= 0) {
        recordNonNegative(amount, unit);
    }
}

protected abstract void recordNonNegative(long amount, TimeUnit unit);
```

Making it `final` means ALL recording paths (direct, Runnable, Supplier, Callable) flow through the same validation. Subclasses implement only `recordNonNegative()`, which they can trust receives valid input.

</details>

---

## 6.6 Tests

### Unit Tests

**New file:** `src/test/java/dev/linhvu/micrometer/TimerTest.java`

```java
class TimerTest {
    @Test void shouldCreateTimerViaBuilder_WhenRegisteredOnRegistry() { ... }
    @Test void shouldReturnSameInstance_WhenRegisteredTwiceWithSameId() { ... }
    @Test void shouldThrowException_WhenNameCollidesWithDifferentType() { ... }
    @Test void shouldRecordDuration_WhenGivenAmountAndUnit() { ... }
    @Test void shouldRecordDuration_WhenGivenDurationObject() { ... }
    @Test void shouldDropNegativeValues_WhenRecordedDirectly() { ... }
    @Test void shouldRecordZero_WhenAmountIsZero() { ... }
    @Test void shouldRecordRunnable_WhenExecutedViaTimer() { ... }
    @Test void shouldRecordSupplier_WhenExecutedViaTimer() { ... }
    @Test void shouldRecordCallable_WhenExecutedViaTimer() { ... }
    @Test void shouldPropagateException_WhenCallableThrows() { ... }
    @Test void shouldPropagateException_WhenRunnableThrows() { ... }
    @Test void shouldCalculateMean_WhenMultipleRecordings() { ... }
    @Test void shouldReturnZeroMean_WhenNoRecordings() { ... }
    @Test void shouldConvertTimeUnits_WhenQueried() { ... }
    @Test void shouldReportBaseTimeUnit_WhenQueried() { ... }
    @Test void shouldRecordViaSample_WhenStartedAndStopped() { ... }
    @Test void shouldStartSampleFromRegistry_WhenRegistryProvided() { ... }
    @Test void shouldProduceThreeMeasurements_WhenMeasured() { ... }
    @Test void shouldHaveTimerType_WhenRegistered() { ... }
}
```

**New file:** `src/test/java/dev/linhvu/micrometer/AbstractTimerTest.java`

```java
class AbstractTimerTest {
    @Test void shouldDropNegativeValues_WhenRecordCalled() { ... }
    @Test void shouldAcceptZero_WhenRecordCalled() { ... }
    @Test void shouldUseMonotonicClock_WhenRecordingRunnable() { ... }
    @Test void shouldUseMonotonicClock_WhenRecordingSupplier() { ... }
    @Test void shouldUseMonotonicClock_WhenRecordingCallable() { ... }
    @Test void shouldStillRecordDuration_WhenRunnableThrows() { ... }
    @Test void shouldStillRecordDuration_WhenCallableThrowsCheckedException() { ... }
    @Test void shouldReturnBaseTimeUnit_WhenQueried() { ... }
}
```

**New file:** `src/test/java/dev/linhvu/micrometer/cumulative/CumulativeTimerTest.java`

```java
class CumulativeTimerTest {
    @Test void shouldStartWithZeroCounts_WhenCreated() { ... }
    @Test void shouldAccumulateCount_WhenMultipleRecordings() { ... }
    @Test void shouldAccumulateTotalTime_WhenMultipleRecordings() { ... }
    @Test void shouldTrackMaximum_WhenMultipleRecordings() { ... }
    @Test void shouldConvertUnitsOnRecord_WhenDifferentUnitsUsed() { ... }
    @Test void shouldStoreInternallyAsNanoseconds_WhenRecordedInVariousUnits() { ... }
    @Test void shouldDecayMaximum_WhenTimeWindowExpires() { ... }
    @Test void shouldKeepMaximum_WhenWithinTimeWindow() { ... }
    @Test void shouldImplementTimerInterface_WhenCreated() { ... }
}
```

**New file:** `src/test/java/dev/linhvu/micrometer/distribution/TimeWindowMaxTest.java`

```java
class TimeWindowMaxTest {
    // Timer mode
    @Test void shouldReturnZero_WhenNothingRecorded() { ... }
    @Test void shouldTrackMaximum_WhenMultipleSamplesRecorded() { ... }
    @Test void shouldDecayToZero_WhenEntireWindowExpires() { ... }
    @Test void shouldRetainMaximum_WhenWithinWindow() { ... }
    @Test void shouldConvertUnitsOnPoll_WhenTimerModeUsed() { ... }
    @Test void shouldPartiallyRotate_WhenSomeWindowsExpire() { ... }
    @Test void shouldReplaceWithNewMax_WhenHigherValueRecordedAfterRotation() { ... }
    // Distribution summary mode
    @Test void shouldTrackRawDoubleMaximum_WhenDistributionSummaryMode() { ... }
    @Test void shouldDecayRawDouble_WhenWindowExpires() { ... }
    // Edge cases
    @Test void shouldThrowException_WhenRotateFrequencyNotPositive() { ... }
}
```

**New file:** `src/test/java/dev/linhvu/micrometer/noop/NoopTimerTest.java`

```java
class NoopTimerTest {
    @Test void shouldReturnZeroForAllStatistics_WhenQueried() { ... }
    @Test void shouldNotRecordAnything_WhenRecordCalled() { ... }
    @Test void shouldExecuteRunnable_WhenRecordCalled() { ... }
    @Test void shouldExecuteSupplier_WhenRecordCalled() { ... }
    @Test void shouldExecuteCallable_WhenRecordCallableCalled() { ... }
    @Test void shouldReturnSecondsBaseTimeUnit_WhenQueried() { ... }
    @Test void shouldBeInstanceOfTimer_WhenCreated() { ... }
}
```

### Integration Tests

**New file:** `src/test/java/dev/linhvu/micrometer/integration/TimerIntegrationTest.java`

```java
class TimerIntegrationTest {
    @Test void shouldCoexistWithCountersAndGauges_WhenRegisteredOnSameRegistry() { ... }
    @Test void shouldRegisterViaConvenienceMethod_WhenCalledOnRegistry() { ... }
    @Test void shouldDeduplicateTimers_WhenRegisteredMultipleTimes() { ... }
    @Test void shouldReturnNoopTimer_WhenRegistryIsClosed() { ... }
    @Test void shouldSupportTimerSampleWithRegistryClock_WhenUsedEnd2End() { ... }
    @Test void shouldAppearInMeterVisitor_WhenMatchCalled() { ... }
    @Test void shouldThrowOnTypeConflict_WhenTimerNameMatchesExistingCounter() { ... }
    @Test void shouldReportInBaseTimeUnit_WhenMeasureCalled() { ... }
}
```

**Run:** `./gradlew test` — all 52+ tests pass (including all prior features)

---

## 6.7 Why This Works

> ★ **Insight** -------------------------------------------
> - **Why a decaying max instead of an all-time max?** An all-time max is dominated by outliers that may have happened hours ago. A 10-second timeout from startup would permanently skew your dashboard. TimeWindowMax's ring buffer gives a "max over the last 2 minutes" that reflects current system behavior. The default 3-bucket, 40-second-rotation setup (total 2 minutes) balances responsiveness with stability.
> - **The `doubleToLongBits` trick:** TimeWindowMax stores both timer nanos (raw longs) and distribution summary values (doubles) in the same `AtomicLong[]`. For doubles, it uses `Double.doubleToLongBits()` which preserves ordering for non-negative values — if `a > b >= 0`, then `doubleToLongBits(a) > doubleToLongBits(b)`. This lets the same CAS-based `updateMax()` work for both modes without any branching.
> - **When NOT to use decaying max:** If you need a true all-time maximum (e.g., for regulatory compliance reporting), use a simple `AtomicLong` with CAS instead. Decaying max is a monitoring optimization, not a general-purpose statistic.
> -----------------------------------------------------------

> ★ **Insight** -------------------------------------------
> - **Why `record(long, TimeUnit)` is `final`:** This is the Template Method pattern within the Template Method pattern. `MeterRegistry.newTimer()` is one template method (subclasses choose the implementation). `AbstractTimer.record()` is another (subclasses only implement `recordNonNegative()`). Making `record()` final ensures negative value rejection can never be bypassed — even if a future subclass forgets validation, the base class enforces it.
> - **Why `finally` blocks for timing:** The wrapped recording methods (`record(Runnable)`, `record(Supplier)`) use `try/finally` to ensure the duration is recorded even if the operation throws. This means you can always trust the timer's count — it reflects attempted operations, not just successful ones. The exception propagates normally, so application error handling is unaffected.
> -----------------------------------------------------------

> ★ **Insight** -------------------------------------------
> - **Why internal storage is nanoseconds:** All durations are converted to nanoseconds on write and converted back on read. This avoids precision loss from intermediate conversions. If you recorded `1 second` and stored it as `1.0` seconds, then asked for milliseconds, you'd get `1000.0`. But if you recorded `333 milliseconds` and stored as `0.333` seconds, floating-point imprecision might give `332.99999...` milliseconds. Storing as `333000000` nanos and dividing by `1000000` gives exactly `333.0`.
> - **Why `LongAdder` instead of `AtomicLong`:** Under contention (many threads recording simultaneously), `AtomicLong.addAndGet()` causes a CAS retry storm — every failed CAS forces a re-read and retry. `LongAdder` maintains per-stripe counters that different threads can update independently, then `sum()` aggregates them. The trade-off: `sum()` is slightly slower (iterates stripes), but `add()` is dramatically faster under contention. Since timers are written far more than read, this is the right trade-off.
> -----------------------------------------------------------

---

## 6.8 What We Enhanced

| Aspect | Before (ch04) | Current (ch06) | Real Framework |
|--------|--------------|----------------|----------------|
| Timer support | `Timer` was an empty stub interface; `newTimer()` threw `UnsupportedOperationException` | Full Timer with record/count/totalTime/max, Sample pattern, Builder, AbstractTimer template, CumulativeTimer with LongAdder+TimeWindowMax | Adds `HistogramSupport`, `PauseDetector` integration, `ResourceSample` (try-with-resources), primitive supplier overloads (`AbstractTimer.java:130-160`) |
| Decaying statistics | No concept of time-windowed statistics | `TimeWindowMax` ring buffer with configurable window and CAS-based lock-free updates | Uses `AtomicIntegerFieldUpdater` instead of `volatile boolean` for rotation guard; integrates with `DistributionStatisticConfig` for window/buffer settings (`TimeWindowMax.java:40-50`) |
| Registry registration | Counter and Gauge only | Counter, Gauge, and Timer — same deduplication, type-checking, and noop-fallback pattern | All 7 meter types plus custom meters via `Meter.builder()` (`MeterRegistry.java:400-450`) |

---

## 6.9 Connection to Real Framework

| Simplified Code | Real Framework Code | File:Line | Key Difference |
|----------------|--------------------|-----------| ---------------|
| `Timer` interface | `Timer` | `Timer.java:50` | Real adds `HistogramSupport`, `ResourceSample`, `@Timed` builder, primitive supplier overloads |
| `Timer.Sample` | `Timer.Sample` | `Timer.java:240` | Identical design — start with clock, stop with timer |
| `Timer.Builder` | `Timer.Builder` extends `AbstractTimerBuilder` | `Timer.java:290` | Real adds `DistributionStatisticConfig`, `PauseDetector`, `MeterProvider`, SLO configuration |
| `AbstractTimer` | `AbstractTimer` | `AbstractTimer.java:55` | Real integrates `Histogram`, `PauseDetector` with LatencyUtils, and `WarnThenDebugLogger` for negative values |
| `AbstractTimer.record(long, TimeUnit)` final | Same pattern | `AbstractTimer.java:155` | Real also records to `histogram.recordLong()` and `IntervalEstimator` |
| `CumulativeTimer` | `CumulativeTimer` | `CumulativeTimer.java:30` | Real takes `DistributionStatisticConfig` and `PauseDetector` in constructor |
| `CumulativeTimer.recordNonNegative()` | Same | `CumulativeTimer.java:57` | Identical: convert to nanos, increment count, add total, record max |
| `TimeWindowMax` | `TimeWindowMax` | `TimeWindowMax.java:35` | Real uses `AtomicIntegerFieldUpdater` for `rotating` flag (more memory-efficient than our `volatile boolean`) |
| `TimeWindowMax.updateMax()` | Same CAS loop | `TimeWindowMax.java:130` | Identical lock-free CAS maximum pattern |
| `NoopTimer` | `NoopTimer` | `NoopTimer.java:25` | Real also implements `HistogramSupport.takeSnapshot()` returning empty snapshot |

---

## 6.10 Complete Code

### Production Code

#### File: `src/main/java/dev/linhvu/micrometer/distribution/TimeWindowMax.java` [NEW]

```java
package dev.linhvu.micrometer.distribution;

import dev.linhvu.micrometer.Clock;

import java.util.concurrent.TimeUnit;
import java.util.concurrent.atomic.AtomicLong;

/**
 * A decaying maximum based on a configurable ring buffer.
 * <p>
 * The ring buffer contains {@code bufferLength} buckets, each covering a time window
 * of {@code rotateFrequencyMillis}. The total time window is
 * {@code rotateFrequencyMillis × bufferLength}.
 * <p>
 * <b>Key insight — write-all, read-one:</b> On {@link #record}, the sample is compared
 * against ALL buckets (updating each one's maximum). On {@link #poll}, only the
 * {@code currentBucket} is read. When a bucket's time window expires, it is zeroed
 * during rotation. This means the current bucket always holds the maximum of all
 * samples recorded since it was last zeroed — giving a maximum that naturally
 * "forgets" old values as buckets rotate out.
 * <p>
 * Two recording modes are supported:
 * <ul>
 *   <li><b>Timer mode:</b> {@code record(double, TimeUnit)} stores nanosecond longs;
 *       {@code poll(TimeUnit)} reads and converts back.</li>
 *   <li><b>Distribution summary mode:</b> {@code record(double)} stores via
 *       {@link Double#doubleToLongBits}; {@code poll()} reads via
 *       {@link Double#longBitsToDouble}. This works because {@code doubleToLongBits}
 *       preserves ordering for non-negative doubles.</li>
 * </ul>
 */
public class TimeWindowMax {

    private static final long DEFAULT_ROTATE_FREQUENCY_MILLIS = 40_000; // 2 minutes / 3 buckets

    private static final int DEFAULT_BUFFER_LENGTH = 3;

    private final Clock clock;

    private final long durationBetweenRotatesMillis;

    private final AtomicLong[] ringBuffer;

    private int currentBucket;

    private volatile long lastRotateTimestampMillis;

    private volatile boolean rotating;

    /**
     * Creates a TimeWindowMax with default settings: 2-minute window, 3 buckets.
     */
    public TimeWindowMax(Clock clock) {
        this(clock, DEFAULT_ROTATE_FREQUENCY_MILLIS, DEFAULT_BUFFER_LENGTH);
    }

    public TimeWindowMax(Clock clock, long rotateFrequencyMillis, int bufferLength) {
        if (rotateFrequencyMillis <= 0) {
            throw new IllegalArgumentException("Rotate frequency must be positive");
        }
        this.clock = clock;
        this.durationBetweenRotatesMillis = rotateFrequencyMillis;
        this.lastRotateTimestampMillis = clock.wallTime();
        this.currentBucket = 0;

        this.ringBuffer = new AtomicLong[bufferLength];
        for (int i = 0; i < bufferLength; i++) {
            this.ringBuffer[i] = new AtomicLong();
        }
    }

    /**
     * For use by timer implementations. Records a sample in the given time unit.
     * Internally stored as nanoseconds.
     */
    public void record(double sample, TimeUnit timeUnit) {
        record(timeUnit.toNanos((long) sample));
    }

    /**
     * For use by distribution summary implementations. Records a raw double value
     * using {@link Double#doubleToLongBits} to preserve ordering in the AtomicLong.
     */
    public void record(double sample) {
        record(Double.doubleToLongBits(sample));
    }

    private void record(long sample) {
        rotate();
        for (AtomicLong max : ringBuffer) {
            updateMax(max, sample);
        }
    }

    /**
     * For use by timer implementations.
     * @param timeUnit The base unit of time to scale the max to.
     * @return The maximum value scaled to the given time unit.
     */
    public double poll(TimeUnit timeUnit) {
        rotate();
        synchronized (this) {
            double nanos = (double) ringBuffer[currentBucket].get();
            return nanos / timeUnit.toNanos(1);
        }
    }

    /**
     * For use by distribution summary implementations.
     * @return The unscaled maximum value.
     */
    public double poll() {
        rotate();
        synchronized (this) {
            return Double.longBitsToDouble(ringBuffer[currentBucket].get());
        }
    }

    /**
     * Lock-free maximum update using compare-and-swap.
     * Spins until either: (a) the sample is stored, or (b) the current max is already ≥ sample.
     */
    private void updateMax(AtomicLong max, long sample) {
        long curMax;
        do {
            curMax = max.get();
        }
        while (curMax < sample && !max.compareAndSet(curMax, sample));
    }

    /**
     * Rotates the ring buffer if enough wall-clock time has elapsed.
     * Uses a volatile boolean as a CAS-like guard to ensure only one thread
     * rotates at a time.
     */
    private void rotate() {
        long wallTime = clock.wallTime();
        long timeSinceLastRotateMillis = wallTime - lastRotateTimestampMillis;
        if (timeSinceLastRotateMillis < durationBetweenRotatesMillis) {
            return;
        }

        // Simple guard: only one thread rotates at a time
        if (rotating) {
            return;
        }
        rotating = true;

        try {
            synchronized (this) {
                // Re-check after acquiring lock
                timeSinceLastRotateMillis = wallTime - lastRotateTimestampMillis;
                if (timeSinceLastRotateMillis < durationBetweenRotatesMillis) {
                    return;
                }

                if (timeSinceLastRotateMillis >= durationBetweenRotatesMillis * ringBuffer.length) {
                    // Enough time to clear the whole ring buffer
                    for (AtomicLong bufferItem : ringBuffer) {
                        bufferItem.set(0);
                    }
                    currentBucket = 0;
                    lastRotateTimestampMillis = wallTime - timeSinceLastRotateMillis % durationBetweenRotatesMillis;
                    return;
                }

                int iterations = 0;
                do {
                    ringBuffer[currentBucket].set(0);
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

#### File: `src/main/java/dev/linhvu/micrometer/Timer.java` [MODIFIED]

```java
package dev.linhvu.micrometer;

import java.time.Duration;
import java.util.Arrays;
import java.util.concurrent.Callable;
import java.util.concurrent.TimeUnit;
import java.util.function.Supplier;

/**
 * Records the duration and frequency of events — the most-used meter type for
 * measuring latency. Intended for tracking short-running events (under a minute).
 * <p>
 * A timer produces three measurements:
 * <ul>
 *   <li>{@link Statistic#COUNT} — total number of recorded events</li>
 *   <li>{@link Statistic#TOTAL_TIME} — cumulative duration of all events</li>
 *   <li>{@link Statistic#MAX} — decaying maximum duration (resets over time)</li>
 * </ul>
 * <p>
 * Two ways to record:
 * <ol>
 *   <li><b>Direct:</b> {@code timer.record(duration)} or {@code timer.record(amount, unit)}</li>
 *   <li><b>Wrapped:</b> {@code timer.record(() -> doWork())} — brackets the call with timing</li>
 *   <li><b>Sample:</b> {@code Timer.Sample s = Timer.start(clock); ... s.stop(timer);} —
 *       start and stop can cross method boundaries</li>
 * </ol>
 */
public interface Timer extends Meter {

    /**
     * Start a timing sample using the system clock.
     */
    static Sample start() {
        return start(Clock.SYSTEM);
    }

    /**
     * Start a timing sample using the registry's clock.
     */
    static Sample start(MeterRegistry registry) {
        return start(registry.getClock());
    }

    /**
     * Start a timing sample using the given clock.
     */
    static Sample start(Clock clock) {
        return new Sample(clock);
    }

    static Builder builder(String name) {
        return new Builder(name);
    }

    /**
     * Records a duration in the given time unit. Negative values are silently dropped.
     */
    void record(long amount, TimeUnit unit);

    /**
     * Records a duration.
     */
    default void record(Duration duration) {
        record(duration.toNanos(), TimeUnit.NANOSECONDS);
    }

    /**
     * Executes the supplier and records the time taken.
     * @return The supplier's return value.
     */
    <T> T record(Supplier<T> f);

    /**
     * Executes the callable and records the time taken.
     * @return The callable's return value.
     * @throws Exception Any exception from the callable bubbles up.
     */
    <T> T recordCallable(Callable<T> f) throws Exception;

    /**
     * Executes the runnable and records the time taken.
     */
    void record(Runnable f);

    /**
     * @return The number of completed recordings.
     */
    long count();

    /**
     * @param unit The time unit to scale the total to.
     * @return The cumulative duration of all recordings.
     */
    double totalTime(TimeUnit unit);

    /**
     * @param unit The time unit to scale the mean to.
     * @return The average duration per recording, or 0 if no recordings.
     */
    default double mean(TimeUnit unit) {
        long count = count();
        return count == 0 ? 0 : totalTime(unit) / count;
    }

    /**
     * @param unit The time unit to scale the max to.
     * @return The maximum duration of a single recording within the decay window.
     */
    double max(TimeUnit unit);

    /**
     * @return The base time unit that this timer reports in when exported.
     */
    TimeUnit baseTimeUnit();

    @Override
    default Iterable<Measurement> measure() {
        return Arrays.asList(
                new Measurement(() -> (double) count(), Statistic.COUNT),
                new Measurement(() -> totalTime(baseTimeUnit()), Statistic.TOTAL_TIME),
                new Measurement(() -> max(baseTimeUnit()), Statistic.MAX));
    }

    /**
     * Maintains state on the clock's start position for a latency sample.
     * Complete the timing by calling {@link #stop(Timer)}.
     * <p>
     * The timer isn't provided until stop — this allows you to determine
     * the timer's tags at the last minute (e.g., based on the response status).
     */
    class Sample {

        private final Clock clock;
        private final long startTime;

        Sample(Clock clock) {
            this.clock = clock;
            this.startTime = clock.monotonicTime();
        }

        /**
         * Records the duration since this sample was started.
         * @param timer The timer to record to.
         * @return The total duration in nanoseconds.
         */
        public long stop(Timer timer) {
            long durationNs = clock.monotonicTime() - startTime;
            timer.record(durationNs, TimeUnit.NANOSECONDS);
            return durationNs;
        }
    }

    /**
     * Fluent builder for timers.
     */
    class Builder {

        private final String name;
        private Tags tags = Tags.empty();
        private String description;

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

        /**
         * Registers this timer on the given registry, or returns an existing timer
         * with the same name and tags.
         */
        public Timer register(MeterRegistry registry) {
            // base unit for a timer is determined by the registry, not the builder
            return registry.timer(
                    new Meter.Id(name, tags, Meter.Type.TIMER, description, null));
        }

        public String getName() { return name; }
        public Tags getTags() { return tags; }
        public String getDescription() { return description; }
    }
}
```

#### File: `src/main/java/dev/linhvu/micrometer/AbstractTimer.java` [NEW]

```java
package dev.linhvu.micrometer;

import java.util.concurrent.Callable;
import java.util.concurrent.TimeUnit;
import java.util.function.Supplier;

/**
 * Base class for {@link Timer} implementations. Provides the shared recording
 * logic that all concrete timers need.
 * <p>
 * The recording pipeline works in two stages:
 * <ol>
 *   <li><b>Wrapping methods</b> ({@code record(Runnable)}, {@code record(Supplier)}, etc.)
 *       bracket the execution with {@link Clock#monotonicTime()} and delegate to
 *       {@code record(long, TimeUnit)}.</li>
 *   <li>{@code record(long, TimeUnit)} is {@code final}: it validates non-negative
 *       and delegates to the abstract {@link #recordNonNegative(long, TimeUnit)}.
 *       This is the Template Method pattern — subclasses only implement the storage.</li>
 * </ol>
 * <p>
 * <b>Simplification:</b> The real Micrometer's AbstractTimer also integrates a Histogram
 * and PauseDetector. Those are added in later features (Distribution Statistics and are
 * out of scope, respectively).
 */
public abstract class AbstractTimer extends AbstractMeter implements Timer {

    protected final Clock clock;

    private final TimeUnit baseTimeUnit;

    protected AbstractTimer(Meter.Id id, Clock clock, TimeUnit baseTimeUnit) {
        super(id);
        this.clock = clock;
        this.baseTimeUnit = baseTimeUnit;
    }

    /**
     * The final recording method. Rejects negative values and delegates to
     * {@link #recordNonNegative(long, TimeUnit)}.
     * <p>
     * Making this final ensures that ALL recording paths (direct, Runnable, Supplier,
     * Callable) go through the same validation — subclasses can't accidentally bypass it.
     */
    @Override
    public final void record(long amount, TimeUnit unit) {
        if (amount >= 0) {
            recordNonNegative(amount, unit);
        }
    }

    /**
     * Subclasses implement this to store the validated, non-negative duration.
     * The amount is in the given time unit — convert to nanoseconds for internal storage.
     */
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

#### File: `src/main/java/dev/linhvu/micrometer/cumulative/CumulativeTimer.java` [NEW]

```java
package dev.linhvu.micrometer.cumulative;

import dev.linhvu.micrometer.AbstractTimer;
import dev.linhvu.micrometer.Clock;
import dev.linhvu.micrometer.Meter;
import dev.linhvu.micrometer.distribution.TimeWindowMax;

import java.util.concurrent.TimeUnit;
import java.util.concurrent.atomic.LongAdder;

/**
 * A cumulative timer implementation that tracks count, total duration, and
 * a decaying maximum.
 * <p>
 * <b>Internal storage is always nanoseconds.</b> Conversion to other units happens
 * on read (in {@link #totalTime(TimeUnit)} and {@link #max(TimeUnit)}).
 * <p>
 * Uses lock-free concurrent accumulators:
 * <ul>
 *   <li>{@link LongAdder} for count — increment-only, O(1) contended writes</li>
 *   <li>{@link LongAdder} for total nanoseconds — add-only</li>
 *   <li>{@link TimeWindowMax} for maximum — ring-buffer-based decaying max</li>
 * </ul>
 */
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

#### File: `src/main/java/dev/linhvu/micrometer/noop/NoopTimer.java` [NEW]

```java
package dev.linhvu.micrometer.noop;

import dev.linhvu.micrometer.Meter;
import dev.linhvu.micrometer.Timer;

import java.util.concurrent.Callable;
import java.util.concurrent.TimeUnit;
import java.util.function.Supplier;

/**
 * A timer that does nothing. Recording methods execute the supplied function but
 * record no metrics. All statistics return zero.
 * <p>
 * Used as a fallback when the registry is closed or a MeterFilter denies registration.
 * The function is still executed so that application logic is not affected by
 * metering being disabled.
 */
public class NoopTimer extends NoopMeter implements Timer {

    public NoopTimer(Meter.Id id) {
        super(id);
    }

    @Override
    public void record(long amount, TimeUnit unit) {
        // no-op
    }

    @Override
    public <T> T record(Supplier<T> f) {
        return f.get();
    }

    @Override
    public <T> T recordCallable(Callable<T> f) throws Exception {
        return f.call();
    }

    @Override
    public void record(Runnable f) {
        f.run();
    }

    @Override
    public long count() {
        return 0;
    }

    @Override
    public double totalTime(TimeUnit unit) {
        return 0;
    }

    @Override
    public double max(TimeUnit unit) {
        return 0;
    }

    @Override
    public TimeUnit baseTimeUnit() {
        return TimeUnit.SECONDS;
    }
}
```

#### File: `src/main/java/dev/linhvu/micrometer/MeterRegistry.java` [MODIFIED]

```java
package dev.linhvu.micrometer;

import dev.linhvu.micrometer.noop.NoopCounter;
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

    /**
     * Creates or retrieves a counter with the given name and tags.
     * @param name The counter name.
     * @param tags Must be an even number of arguments representing key/value pairs.
     * @return A new or existing Counter.
     */
    public Counter counter(String name, String... tags) {
        return Counter.builder(name).tags(tags).register(this);
    }

    /**
     * Creates or retrieves a counter with the given name and tags.
     * @param name The counter name.
     * @param tags The tag collection.
     * @return A new or existing Counter.
     */
    public Counter counter(String name, Iterable<Tag> tags) {
        return Counter.builder(name).tags(tags).register(this);
    }

    /**
     * Register a gauge that reports the value of the {@code obj} as determined by the
     * function {@code valueFunction}. The state object is held via a WeakReference.
     *
     * @param name          The gauge name.
     * @param tags          Tags for the gauge.
     * @param stateObject   The state object to observe.
     * @param valueFunction Function that extracts a double value from the state object.
     * @param <T>           The type of the state object.
     * @return The state object, for inline assignment.
     */
    public <T> T gauge(String name, Iterable<Tag> tags, T stateObject, ToDoubleFunction<T> valueFunction) {
        Gauge.builder(name, stateObject, valueFunction).tags(tags).register(this);
        return stateObject;
    }

    /**
     * Register a gauge that reports the value of the {@link Number}.
     *
     * @param name   The gauge name.
     * @param tags   Tags for the gauge.
     * @param number The Number whose {@code doubleValue()} to report.
     * @param <T>    The type of the number.
     * @return The number, for inline assignment.
     */
    public <T extends Number> T gauge(String name, Iterable<Tag> tags, T number) {
        return gauge(name, tags, number, Number::doubleValue);
    }

    /**
     * Register a gauge that reports the value of the {@link Number}.
     *
     * @param name   The gauge name.
     * @param number The Number whose {@code doubleValue()} to report.
     * @param <T>    The type of the number.
     * @return The number, for inline assignment.
     */
    public <T extends Number> T gauge(String name, T number) {
        return gauge(name, Collections.emptyList(), number);
    }

    /**
     * Register a gauge that reports the size of the {@link Collection}.
     *
     * @param name       The gauge name.
     * @param tags       Tags for the gauge.
     * @param collection The collection whose size to report.
     * @param <T>        The type of the collection.
     * @return The collection, for inline assignment.
     */
    public <T extends Collection<?>> T gaugeCollectionSize(String name, Iterable<Tag> tags, T collection) {
        return gauge(name, tags, collection, Collection::size);
    }

    /**
     * Register a gauge that reports the size of the {@link Map}.
     *
     * @param name The gauge name.
     * @param tags Tags for the gauge.
     * @param map  The map whose size to report.
     * @param <T>  The type of the map.
     * @return The map, for inline assignment.
     */
    public <T extends Map<?, ?>> T gaugeMapSize(String name, Iterable<Tag> tags, T map) {
        return gauge(name, tags, map, Map::size);
    }

    /**
     * Creates or retrieves a timer with the given name and tags.
     * @param name The timer name.
     * @param tags Must be an even number of arguments representing key/value pairs.
     * @return A new or existing Timer.
     */
    public Timer timer(String name, String... tags) {
        return Timer.builder(name).tags(tags).register(this);
    }

    /**
     * Creates or retrieves a timer with the given name and tags.
     * @param name The timer name.
     * @param tags The tag collection.
     * @return A new or existing Timer.
     */
    public Timer timer(String name, Iterable<Tag> tags) {
        return Timer.builder(name).tags(tags).register(this);
    }

    // -----------------------------------------------------------------------
    // Package-private registration (called by Builder.register())
    // -----------------------------------------------------------------------

    /**
     * Registers a counter using the pre-built ID. Called by {@link Counter.Builder#register(MeterRegistry)}.
     */
    Counter counter(Meter.Id id) {
        return registerMeterIfNecessary(Counter.class, id, this::newCounter, NoopCounter::new);
    }

    /**
     * Registers a gauge using the pre-built ID. Called by {@link Gauge.Builder#register(MeterRegistry)}.
     * The gauge factory is a lambda because {@code newGauge()} requires additional parameters
     * (the state object and value function) that standard method references cannot capture.
     */
    <T> Gauge gauge(Meter.Id id, T obj, ToDoubleFunction<T> valueFunction) {
        return registerMeterIfNecessary(Gauge.class, id,
                mappedId -> newGauge(mappedId, obj, valueFunction), NoopGauge::new);
    }

    /**
     * Registers a timer using the pre-built ID. Called by {@link Timer.Builder#register(MeterRegistry)}.
     */
    Timer timer(Meter.Id id) {
        return registerMeterIfNecessary(Timer.class, id, this::newTimer, NoopTimer::new);
    }

    // -----------------------------------------------------------------------
    // Template Method factories — subclasses implement these
    // -----------------------------------------------------------------------

    /**
     * Create a new counter implementation for this registry.
     * Called only when no counter with this ID exists yet.
     */
    protected abstract Counter newCounter(Meter.Id id);

    /**
     * Create a new gauge implementation for this registry.
     * Will be functional after Feature 5 (Gauge) is implemented.
     */
    protected abstract <T> Gauge newGauge(Meter.Id id, T obj, ToDoubleFunction<T> valueFunction);

    /**
     * Create a new timer implementation for this registry.
     * Will be functional after Feature 6 (Timer) is implemented.
     */
    protected abstract Timer newTimer(Meter.Id id);

    /**
     * Create a new distribution summary implementation for this registry.
     * Will be functional after Feature 7 (DistributionSummary) is implemented.
     */
    protected abstract DistributionSummary newDistributionSummary(Meter.Id id);

    /**
     * Returns the base time unit for this registry. Backends that work in
     * seconds (e.g., Prometheus) return {@link TimeUnit#SECONDS}; backends
     * that work in milliseconds return {@link TimeUnit#MILLISECONDS}.
     */
    protected abstract TimeUnit getBaseTimeUnit();

    // -----------------------------------------------------------------------
    // Registration lifecycle — the heart of deduplication
    // -----------------------------------------------------------------------

    /**
     * Registers a meter if one with the same ID does not already exist. If a meter
     * with the same name+tags already exists but is a different type, throws
     * {@link IllegalArgumentException}.
     *
     * @param meterClass    The expected meter type (e.g., Counter.class)
     * @param id            The meter's identity (name + tags)
     * @param meterFactory  Factory to create the real meter (calls a template method)
     * @param noopFactory   Factory to create a no-op meter (for closed registries)
     * @return The existing or newly created meter, cast to M
     */
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

    /**
     * The core deduplication algorithm. Uses double-check locking:
     * <ol>
     *     <li>Lock-free read from ConcurrentHashMap (fast path — no contention)</li>
     *     <li>Synchronized creation block with re-check (slow path — only on first registration)</li>
     * </ol>
     *
     * When the registry is closed, returns a no-op meter instead of creating a real one.
     * This ensures that late registrations after shutdown are silently absorbed rather
     * than throwing exceptions.
     */
    private Meter getOrCreateMeter(Meter.Id id,
            Function<Meter.Id, ? extends Meter> meterFactory,
            Function<Meter.Id, ? extends Meter> noopFactory) {

        // Fast path: ConcurrentHashMap.get() is lock-free
        Meter existing = meterMap.get(id);
        if (existing != null) {
            return existing;
        }

        // Slow path: synchronized creation to prevent duplicate meters
        synchronized (meterMapLock) {
            // Double-check: another thread may have created it while we waited
            existing = meterMap.get(id);
            if (existing != null) {
                return existing;
            }

            // Registry is closed → return a no-op meter
            if (closed) {
                return noopFactory.apply(id);
            }

            // Create the meter via the template method
            Meter newMeter = meterFactory.apply(id);
            meterMap.put(id, newMeter);
            return newMeter;
        }
    }

    // -----------------------------------------------------------------------
    // Query methods
    // -----------------------------------------------------------------------

    /**
     * Returns a snapshot of all registered meters. The returned list is a copy;
     * modifications to it do not affect the registry.
     */
    public List<Meter> getMeters() {
        return Collections.unmodifiableList(new ArrayList<>(meterMap.values()));
    }

    /**
     * Returns the clock used by this registry for time measurements.
     */
    public Clock getClock() {
        return clock;
    }

    // -----------------------------------------------------------------------
    // Lifecycle
    // -----------------------------------------------------------------------

    /**
     * Returns true if this registry has been closed. A closed registry returns
     * no-op meters for all new registrations.
     */
    public boolean isClosed() {
        return closed;
    }

    /**
     * Closes this registry. After this call, any new meter registration will
     * return a no-op meter. Existing meters remain functional.
     */
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
import dev.linhvu.micrometer.Gauge;
import dev.linhvu.micrometer.Meter;
import dev.linhvu.micrometer.MeterRegistry;
import dev.linhvu.micrometer.Timer;
import dev.linhvu.micrometer.cumulative.CumulativeCounter;
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
 * {@link DefaultGauge}), and Timer (via {@link CumulativeTimer}).
 * DistributionSummary factory will be implemented in Feature 7.
 */
public class SimpleMeterRegistry extends MeterRegistry {

    /**
     * Creates a SimpleMeterRegistry using the system clock.
     */
    public SimpleMeterRegistry() {
        this(Clock.SYSTEM);
    }

    /**
     * Creates a SimpleMeterRegistry with the given clock.
     * Use {@link dev.linhvu.micrometer.MockClock} for testing.
     */
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
    protected DistributionSummary newDistributionSummary(Meter.Id id) {
        throw new UnsupportedOperationException("DistributionSummary not yet supported. See Feature 7.");
    }

    @Override
    protected TimeUnit getBaseTimeUnit() {
        return TimeUnit.SECONDS;
    }

}
```

### Test Code

#### File: `src/test/java/dev/linhvu/micrometer/TimerTest.java` [NEW]

```java
package dev.linhvu.micrometer;

import dev.linhvu.micrometer.simple.SimpleMeterRegistry;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.time.Duration;
import java.util.concurrent.TimeUnit;
import java.util.concurrent.atomic.AtomicInteger;

import static org.assertj.core.api.Assertions.*;

/**
 * Tests for the {@link Timer} interface — covers the API contract, Builder,
 * Sample pattern, measure() output, and default methods like mean() and
 * record(Duration).
 */
class TimerTest {

    private MockClock clock;
    private SimpleMeterRegistry registry;

    @BeforeEach
    void setUp() {
        clock = new MockClock();
        registry = new SimpleMeterRegistry(clock);
    }

    // --- Builder and registration ---

    @Test
    void shouldCreateTimerViaBuilder_WhenRegisteredOnRegistry() {
        Timer timer = Timer.builder("http.requests")
                .tag("method", "GET")
                .description("HTTP request latency")
                .register(registry);

        assertThat(timer).isNotNull();
        assertThat(timer.getId().getName()).isEqualTo("http.requests");
        assertThat(timer.getId().getTag("method")).isEqualTo("GET");
        assertThat(timer.getId().getDescription()).isEqualTo("HTTP request latency");
    }

    @Test
    void shouldReturnSameInstance_WhenRegisteredTwiceWithSameId() {
        Timer timer1 = Timer.builder("my.timer").register(registry);
        Timer timer2 = Timer.builder("my.timer").register(registry);

        assertThat(timer1).isSameAs(timer2);
    }

    @Test
    void shouldThrowException_WhenNameCollidesWithDifferentType() {
        registry.counter("shared.name");

        assertThatThrownBy(() -> Timer.builder("shared.name").register(registry))
                .isInstanceOf(IllegalArgumentException.class)
                .hasMessageContaining("shared.name");
    }

    // --- Direct recording ---

    @Test
    void shouldRecordDuration_WhenGivenAmountAndUnit() {
        Timer timer = Timer.builder("test").register(registry);

        timer.record(100, TimeUnit.MILLISECONDS);
        timer.record(200, TimeUnit.MILLISECONDS);

        assertThat(timer.count()).isEqualTo(2);
        assertThat(timer.totalTime(TimeUnit.MILLISECONDS)).isEqualTo(300.0);
    }

    @Test
    void shouldRecordDuration_WhenGivenDurationObject() {
        Timer timer = Timer.builder("test").register(registry);

        timer.record(Duration.ofMillis(150));

        assertThat(timer.count()).isEqualTo(1);
        assertThat(timer.totalTime(TimeUnit.MILLISECONDS)).isEqualTo(150.0);
    }

    @Test
    void shouldDropNegativeValues_WhenRecordedDirectly() {
        Timer timer = Timer.builder("test").register(registry);

        timer.record(-1, TimeUnit.SECONDS);

        assertThat(timer.count()).isEqualTo(0);
        assertThat(timer.totalTime(TimeUnit.SECONDS)).isEqualTo(0.0);
    }

    @Test
    void shouldRecordZero_WhenAmountIsZero() {
        Timer timer = Timer.builder("test").register(registry);

        timer.record(0, TimeUnit.SECONDS);

        assertThat(timer.count()).isEqualTo(1);
        assertThat(timer.totalTime(TimeUnit.SECONDS)).isEqualTo(0.0);
    }

    // --- Wrapped recording ---

    @Test
    void shouldRecordRunnable_WhenExecutedViaTimer() {
        Timer timer = Timer.builder("test").register(registry);
        AtomicInteger counter = new AtomicInteger();

        timer.record(counter::incrementAndGet);

        assertThat(counter.get()).isEqualTo(1);
        assertThat(timer.count()).isEqualTo(1);
    }

    @Test
    void shouldRecordSupplier_WhenExecutedViaTimer() {
        Timer timer = Timer.builder("test").register(registry);

        String result = timer.record(() -> "hello");

        assertThat(result).isEqualTo("hello");
        assertThat(timer.count()).isEqualTo(1);
    }

    @Test
    void shouldRecordCallable_WhenExecutedViaTimer() throws Exception {
        Timer timer = Timer.builder("test").register(registry);

        Integer result = timer.recordCallable(() -> 42);

        assertThat(result).isEqualTo(42);
        assertThat(timer.count()).isEqualTo(1);
    }

    @Test
    void shouldPropagateException_WhenCallableThrows() {
        Timer timer = Timer.builder("test").register(registry);

        assertThatThrownBy(() -> timer.recordCallable(() -> {
            throw new RuntimeException("boom");
        }))
                .isInstanceOf(RuntimeException.class)
                .hasMessage("boom");

        // Duration is still recorded even though it threw
        assertThat(timer.count()).isEqualTo(1);
    }

    @Test
    void shouldPropagateException_WhenRunnableThrows() {
        Timer timer = Timer.builder("test").register(registry);

        assertThatThrownBy(() -> timer.record((Runnable) () -> {
            throw new RuntimeException("boom");
        }))
                .isInstanceOf(RuntimeException.class)
                .hasMessage("boom");

        assertThat(timer.count()).isEqualTo(1);
    }

    // --- Statistics ---

    @Test
    void shouldCalculateMean_WhenMultipleRecordings() {
        Timer timer = Timer.builder("test").register(registry);

        timer.record(100, TimeUnit.MILLISECONDS);
        timer.record(200, TimeUnit.MILLISECONDS);
        timer.record(300, TimeUnit.MILLISECONDS);

        assertThat(timer.mean(TimeUnit.MILLISECONDS)).isEqualTo(200.0);
    }

    @Test
    void shouldReturnZeroMean_WhenNoRecordings() {
        Timer timer = Timer.builder("test").register(registry);

        assertThat(timer.mean(TimeUnit.MILLISECONDS)).isEqualTo(0.0);
    }

    @Test
    void shouldConvertTimeUnits_WhenQueried() {
        Timer timer = Timer.builder("test").register(registry);

        timer.record(1, TimeUnit.SECONDS);

        assertThat(timer.totalTime(TimeUnit.SECONDS)).isEqualTo(1.0);
        assertThat(timer.totalTime(TimeUnit.MILLISECONDS)).isEqualTo(1000.0);
        assertThat(timer.totalTime(TimeUnit.NANOSECONDS)).isEqualTo(1_000_000_000.0);
    }

    @Test
    void shouldReportBaseTimeUnit_WhenQueried() {
        Timer timer = Timer.builder("test").register(registry);

        // SimpleMeterRegistry uses SECONDS as base time unit
        assertThat(timer.baseTimeUnit()).isEqualTo(TimeUnit.SECONDS);
    }

    // --- Sample pattern ---

    @Test
    void shouldRecordViaSample_WhenStartedAndStopped() {
        Timer.Sample sample = Timer.start(clock);

        clock.add(250, TimeUnit.MILLISECONDS);

        Timer timer = Timer.builder("test").register(registry);
        long durationNs = sample.stop(timer);

        assertThat(durationNs).isEqualTo(TimeUnit.MILLISECONDS.toNanos(250));
        assertThat(timer.count()).isEqualTo(1);
        assertThat(timer.totalTime(TimeUnit.MILLISECONDS)).isEqualTo(250.0);
    }

    @Test
    void shouldStartSampleFromRegistry_WhenRegistryProvided() {
        Timer.Sample sample = Timer.start(registry);

        clock.add(100, TimeUnit.MILLISECONDS);

        Timer timer = Timer.builder("test").register(registry);
        sample.stop(timer);

        assertThat(timer.count()).isEqualTo(1);
        assertThat(timer.totalTime(TimeUnit.MILLISECONDS)).isEqualTo(100.0);
    }

    // --- measure() ---

    @Test
    void shouldProduceThreeMeasurements_WhenMeasured() {
        Timer timer = Timer.builder("test").register(registry);
        timer.record(500, TimeUnit.MILLISECONDS);

        var measurements = timer.measure();
        var measurementList = new java.util.ArrayList<Measurement>();
        measurements.forEach(measurementList::add);

        assertThat(measurementList).hasSize(3);

        // COUNT
        assertThat(measurementList.get(0).getStatistic()).isEqualTo(Statistic.COUNT);
        assertThat(measurementList.get(0).getValue()).isEqualTo(1.0);

        // TOTAL_TIME (in base time unit = seconds)
        assertThat(measurementList.get(1).getStatistic()).isEqualTo(Statistic.TOTAL_TIME);
        assertThat(measurementList.get(1).getValue()).isEqualTo(0.5);

        // MAX (in base time unit = seconds)
        assertThat(measurementList.get(2).getStatistic()).isEqualTo(Statistic.MAX);
        assertThat(measurementList.get(2).getValue()).isEqualTo(0.5);
    }

    // --- Meter type ---

    @Test
    void shouldHaveTimerType_WhenRegistered() {
        Timer timer = Timer.builder("test").register(registry);

        assertThat(timer.getId().getType()).isEqualTo(Meter.Type.TIMER);
    }
}
```

#### File: `src/test/java/dev/linhvu/micrometer/AbstractTimerTest.java` [NEW]

```java
package dev.linhvu.micrometer;

import dev.linhvu.micrometer.cumulative.CumulativeTimer;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.util.concurrent.Callable;
import java.util.concurrent.TimeUnit;
import java.util.concurrent.atomic.AtomicBoolean;

import static org.assertj.core.api.Assertions.*;

/**
 * Tests for {@link AbstractTimer} — specifically the template method pattern
 * where record(long, TimeUnit) is final, the wrapping methods bracket with
 * clock timing, and negative values are dropped.
 */
class AbstractTimerTest {

    private MockClock clock;
    private Timer timer;

    @BeforeEach
    void setUp() {
        clock = new MockClock();
        Meter.Id id = new Meter.Id("test", Tags.empty(), Meter.Type.TIMER, null, null);
        timer = new CumulativeTimer(id, clock, TimeUnit.SECONDS);
    }

    @Test
    void shouldDropNegativeValues_WhenRecordCalled() {
        timer.record(-5, TimeUnit.SECONDS);

        assertThat(timer.count()).isEqualTo(0);
        assertThat(timer.totalTime(TimeUnit.SECONDS)).isEqualTo(0.0);
    }

    @Test
    void shouldAcceptZero_WhenRecordCalled() {
        timer.record(0, TimeUnit.SECONDS);

        assertThat(timer.count()).isEqualTo(1);
    }

    @Test
    void shouldUseMonotonicClock_WhenRecordingRunnable() {
        clock.add(0, TimeUnit.NANOSECONDS); // clock starts at 1ms

        timer.record(() -> clock.add(100, TimeUnit.MILLISECONDS));

        assertThat(timer.count()).isEqualTo(1);
        assertThat(timer.totalTime(TimeUnit.MILLISECONDS)).isEqualTo(100.0);
    }

    @Test
    void shouldUseMonotonicClock_WhenRecordingSupplier() {
        String result = timer.record(() -> {
            clock.add(50, TimeUnit.MILLISECONDS);
            return "done";
        });

        assertThat(result).isEqualTo("done");
        assertThat(timer.totalTime(TimeUnit.MILLISECONDS)).isEqualTo(50.0);
    }

    @Test
    void shouldUseMonotonicClock_WhenRecordingCallable() throws Exception {
        Integer result = timer.recordCallable(() -> {
            clock.add(75, TimeUnit.MILLISECONDS);
            return 42;
        });

        assertThat(result).isEqualTo(42);
        assertThat(timer.totalTime(TimeUnit.MILLISECONDS)).isEqualTo(75.0);
    }

    @Test
    void shouldStillRecordDuration_WhenRunnableThrows() {
        AtomicBoolean entered = new AtomicBoolean(false);

        assertThatThrownBy(() -> timer.record((Runnable) () -> {
            entered.set(true);
            clock.add(30, TimeUnit.MILLISECONDS);
            throw new RuntimeException("fail");
        })).isInstanceOf(RuntimeException.class);

        assertThat(entered.get()).isTrue();
        assertThat(timer.count()).isEqualTo(1);
        assertThat(timer.totalTime(TimeUnit.MILLISECONDS)).isEqualTo(30.0);
    }

    @Test
    void shouldStillRecordDuration_WhenCallableThrowsCheckedException() {
        assertThatThrownBy(() -> timer.recordCallable((Callable<String>) () -> {
            clock.add(20, TimeUnit.MILLISECONDS);
            throw new Exception("checked");
        })).isInstanceOf(Exception.class);

        assertThat(timer.count()).isEqualTo(1);
        assertThat(timer.totalTime(TimeUnit.MILLISECONDS)).isEqualTo(20.0);
    }

    @Test
    void shouldReturnBaseTimeUnit_WhenQueried() {
        assertThat(timer.baseTimeUnit()).isEqualTo(TimeUnit.SECONDS);
    }
}
```

#### File: `src/test/java/dev/linhvu/micrometer/cumulative/CumulativeTimerTest.java` [NEW]

```java
package dev.linhvu.micrometer.cumulative;

import dev.linhvu.micrometer.Meter;
import dev.linhvu.micrometer.MockClock;
import dev.linhvu.micrometer.Tags;
import dev.linhvu.micrometer.Timer;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.util.concurrent.TimeUnit;

import static org.assertj.core.api.Assertions.*;

/**
 * Tests for {@link CumulativeTimer} — the concrete cumulative implementation
 * that tracks count, total time, and a decaying maximum.
 */
class CumulativeTimerTest {

    private MockClock clock;
    private CumulativeTimer timer;

    @BeforeEach
    void setUp() {
        clock = new MockClock();
        Meter.Id id = new Meter.Id("test.timer", Tags.empty(), Meter.Type.TIMER, null, null);
        timer = new CumulativeTimer(id, clock, TimeUnit.SECONDS);
    }

    @Test
    void shouldStartWithZeroCounts_WhenCreated() {
        assertThat(timer.count()).isEqualTo(0);
        assertThat(timer.totalTime(TimeUnit.SECONDS)).isEqualTo(0.0);
        assertThat(timer.max(TimeUnit.SECONDS)).isEqualTo(0.0);
    }

    @Test
    void shouldAccumulateCount_WhenMultipleRecordings() {
        timer.record(100, TimeUnit.MILLISECONDS);
        timer.record(200, TimeUnit.MILLISECONDS);
        timer.record(300, TimeUnit.MILLISECONDS);

        assertThat(timer.count()).isEqualTo(3);
    }

    @Test
    void shouldAccumulateTotalTime_WhenMultipleRecordings() {
        timer.record(1, TimeUnit.SECONDS);
        timer.record(2, TimeUnit.SECONDS);

        assertThat(timer.totalTime(TimeUnit.SECONDS)).isEqualTo(3.0);
        assertThat(timer.totalTime(TimeUnit.MILLISECONDS)).isEqualTo(3000.0);
    }

    @Test
    void shouldTrackMaximum_WhenMultipleRecordings() {
        timer.record(100, TimeUnit.MILLISECONDS);
        timer.record(500, TimeUnit.MILLISECONDS);
        timer.record(200, TimeUnit.MILLISECONDS);

        assertThat(timer.max(TimeUnit.MILLISECONDS)).isEqualTo(500.0);
    }

    @Test
    void shouldConvertUnitsOnRecord_WhenDifferentUnitsUsed() {
        timer.record(1, TimeUnit.SECONDS);
        timer.record(500, TimeUnit.MILLISECONDS);

        // Total should be 1.5 seconds
        assertThat(timer.totalTime(TimeUnit.SECONDS)).isEqualTo(1.5);
        assertThat(timer.totalTime(TimeUnit.MILLISECONDS)).isEqualTo(1500.0);
    }

    @Test
    void shouldStoreInternallyAsNanoseconds_WhenRecordedInVariousUnits() {
        timer.record(1_000_000, TimeUnit.NANOSECONDS); // 1 ms
        timer.record(1, TimeUnit.MILLISECONDS);        // 1 ms

        assertThat(timer.totalTime(TimeUnit.MILLISECONDS)).isEqualTo(2.0);
        assertThat(timer.count()).isEqualTo(2);
    }

    @Test
    void shouldDecayMaximum_WhenTimeWindowExpires() {
        timer.record(500, TimeUnit.MILLISECONDS);
        assertThat(timer.max(TimeUnit.MILLISECONDS)).isEqualTo(500.0);

        // Advance past the full decay window (default: 3 × 40s = 120s = 2 minutes)
        clock.add(3, TimeUnit.MINUTES);

        assertThat(timer.max(TimeUnit.MILLISECONDS)).isEqualTo(0.0);
    }

    @Test
    void shouldKeepMaximum_WhenWithinTimeWindow() {
        timer.record(500, TimeUnit.MILLISECONDS);

        // Advance only a little — within the window
        clock.add(10, TimeUnit.SECONDS);

        assertThat(timer.max(TimeUnit.MILLISECONDS)).isEqualTo(500.0);
    }

    @Test
    void shouldImplementTimerInterface_WhenCreated() {
        assertThat(timer).isInstanceOf(Timer.class);
    }
}
```

#### File: `src/test/java/dev/linhvu/micrometer/distribution/TimeWindowMaxTest.java` [NEW]

```java
package dev.linhvu.micrometer.distribution;

import dev.linhvu.micrometer.MockClock;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.util.concurrent.TimeUnit;

import static org.assertj.core.api.Assertions.*;

/**
 * Tests for {@link TimeWindowMax} — the ring-buffer-based decaying maximum
 * used by Timer (nanosecond mode) and DistributionSummary (raw double mode).
 */
class TimeWindowMaxTest {

    private MockClock clock;

    @BeforeEach
    void setUp() {
        clock = new MockClock();
    }

    // --- Timer mode (nanosecond storage) ---

    @Test
    void shouldReturnZero_WhenNothingRecorded() {
        TimeWindowMax max = new TimeWindowMax(clock);

        assertThat(max.poll(TimeUnit.SECONDS)).isEqualTo(0.0);
    }

    @Test
    void shouldTrackMaximum_WhenMultipleSamplesRecorded() {
        TimeWindowMax max = new TimeWindowMax(clock);

        max.record(100.0, TimeUnit.MILLISECONDS);
        max.record(500.0, TimeUnit.MILLISECONDS);
        max.record(200.0, TimeUnit.MILLISECONDS);

        assertThat(max.poll(TimeUnit.MILLISECONDS)).isEqualTo(500.0);
    }

    @Test
    void shouldDecayToZero_WhenEntireWindowExpires() {
        // 3 buckets, 1 second per rotation = 3 second total window
        TimeWindowMax max = new TimeWindowMax(clock, 1000, 3);

        max.record(100.0, TimeUnit.MILLISECONDS);
        assertThat(max.poll(TimeUnit.MILLISECONDS)).isEqualTo(100.0);

        // Advance past entire window
        clock.add(4, TimeUnit.SECONDS);

        assertThat(max.poll(TimeUnit.MILLISECONDS)).isEqualTo(0.0);
    }

    @Test
    void shouldRetainMaximum_WhenWithinWindow() {
        TimeWindowMax max = new TimeWindowMax(clock, 1000, 3);

        max.record(250.0, TimeUnit.MILLISECONDS);

        // Advance 1 second — one rotation, but the sample is still in 2 remaining buckets
        clock.add(1, TimeUnit.SECONDS);

        assertThat(max.poll(TimeUnit.MILLISECONDS)).isEqualTo(250.0);
    }

    @Test
    void shouldConvertUnitsOnPoll_WhenTimerModeUsed() {
        TimeWindowMax max = new TimeWindowMax(clock);

        max.record(1_000_000.0, TimeUnit.NANOSECONDS); // 1 ms

        assertThat(max.poll(TimeUnit.NANOSECONDS)).isEqualTo(1_000_000.0);
        assertThat(max.poll(TimeUnit.MILLISECONDS)).isEqualTo(1.0);
        assertThat(max.poll(TimeUnit.SECONDS)).isCloseTo(0.001, within(1e-9));
    }

    @Test
    void shouldPartiallyRotate_WhenSomeWindowsExpire() {
        // 3 buckets, 1 second each
        TimeWindowMax max = new TimeWindowMax(clock, 1000, 3);

        max.record(100.0, TimeUnit.MILLISECONDS);

        // Advance 2 seconds — two rotations. Sample was in all 3 buckets;
        // two are cleared, one remains
        clock.add(2, TimeUnit.SECONDS);

        assertThat(max.poll(TimeUnit.MILLISECONDS)).isEqualTo(100.0);

        // Advance one more second — last bucket cleared
        clock.add(1, TimeUnit.SECONDS);

        assertThat(max.poll(TimeUnit.MILLISECONDS)).isEqualTo(0.0);
    }

    @Test
    void shouldReplaceWithNewMax_WhenHigherValueRecordedAfterRotation() {
        TimeWindowMax max = new TimeWindowMax(clock, 1000, 3);

        max.record(100.0, TimeUnit.MILLISECONDS);
        clock.add(2, TimeUnit.SECONDS);

        max.record(200.0, TimeUnit.MILLISECONDS);

        assertThat(max.poll(TimeUnit.MILLISECONDS)).isEqualTo(200.0);
    }

    // --- Distribution summary mode (raw double storage) ---

    @Test
    void shouldTrackRawDoubleMaximum_WhenDistributionSummaryMode() {
        TimeWindowMax max = new TimeWindowMax(clock);

        max.record(1.5);
        max.record(3.7);
        max.record(2.1);

        assertThat(max.poll()).isEqualTo(3.7);
    }

    @Test
    void shouldDecayRawDouble_WhenWindowExpires() {
        TimeWindowMax max = new TimeWindowMax(clock, 1000, 3);

        max.record(5.0);
        assertThat(max.poll()).isEqualTo(5.0);

        clock.add(4, TimeUnit.SECONDS);

        assertThat(max.poll()).isEqualTo(0.0);
    }

    // --- Edge cases ---

    @Test
    void shouldThrowException_WhenRotateFrequencyNotPositive() {
        assertThatThrownBy(() -> new TimeWindowMax(clock, 0, 3))
                .isInstanceOf(IllegalArgumentException.class);

        assertThatThrownBy(() -> new TimeWindowMax(clock, -1, 3))
                .isInstanceOf(IllegalArgumentException.class);
    }
}
```

#### File: `src/test/java/dev/linhvu/micrometer/noop/NoopTimerTest.java` [NEW]

```java
package dev.linhvu.micrometer.noop;

import dev.linhvu.micrometer.Meter;
import dev.linhvu.micrometer.Tags;
import dev.linhvu.micrometer.Timer;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.util.concurrent.TimeUnit;
import java.util.concurrent.atomic.AtomicInteger;

import static org.assertj.core.api.Assertions.*;

/**
 * Tests for {@link NoopTimer} — verifies that it executes supplied functions
 * but records no metrics.
 */
class NoopTimerTest {

    private NoopTimer timer;

    @BeforeEach
    void setUp() {
        timer = new NoopTimer(new Meter.Id("noop", Tags.empty(), Meter.Type.TIMER, null, null));
    }

    @Test
    void shouldReturnZeroForAllStatistics_WhenQueried() {
        assertThat(timer.count()).isEqualTo(0);
        assertThat(timer.totalTime(TimeUnit.SECONDS)).isEqualTo(0.0);
        assertThat(timer.max(TimeUnit.SECONDS)).isEqualTo(0.0);
    }

    @Test
    void shouldNotRecordAnything_WhenRecordCalled() {
        timer.record(100, TimeUnit.MILLISECONDS);

        assertThat(timer.count()).isEqualTo(0);
        assertThat(timer.totalTime(TimeUnit.MILLISECONDS)).isEqualTo(0.0);
    }

    @Test
    void shouldExecuteRunnable_WhenRecordCalled() {
        AtomicInteger counter = new AtomicInteger();

        timer.record(counter::incrementAndGet);

        assertThat(counter.get()).isEqualTo(1);
        assertThat(timer.count()).isEqualTo(0);
    }

    @Test
    void shouldExecuteSupplier_WhenRecordCalled() {
        String result = timer.record(() -> "hello");

        assertThat(result).isEqualTo("hello");
        assertThat(timer.count()).isEqualTo(0);
    }

    @Test
    void shouldExecuteCallable_WhenRecordCallableCalled() throws Exception {
        Integer result = timer.recordCallable(() -> 42);

        assertThat(result).isEqualTo(42);
        assertThat(timer.count()).isEqualTo(0);
    }

    @Test
    void shouldReturnSecondsBaseTimeUnit_WhenQueried() {
        assertThat(timer.baseTimeUnit()).isEqualTo(TimeUnit.SECONDS);
    }

    @Test
    void shouldBeInstanceOfTimer_WhenCreated() {
        assertThat(timer).isInstanceOf(Timer.class);
    }
}
```

#### File: `src/test/java/dev/linhvu/micrometer/integration/TimerIntegrationTest.java` [NEW]

```java
package dev.linhvu.micrometer.integration;

import dev.linhvu.micrometer.Counter;
import dev.linhvu.micrometer.Gauge;
import dev.linhvu.micrometer.Meter;
import dev.linhvu.micrometer.MockClock;
import dev.linhvu.micrometer.Timer;
import dev.linhvu.micrometer.noop.NoopTimer;
import dev.linhvu.micrometer.simple.SimpleMeterRegistry;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.util.concurrent.TimeUnit;

import static org.assertj.core.api.Assertions.*;

/**
 * Integration tests for Timer working with the registry and other meter types.
 * Verifies the full lifecycle: Builder → registry.timer(Id) → newTimer() → CumulativeTimer.
 */
class TimerIntegrationTest {

    private MockClock clock;
    private SimpleMeterRegistry registry;

    @BeforeEach
    void setUp() {
        clock = new MockClock();
        registry = new SimpleMeterRegistry(clock);
    }

    @Test
    void shouldCoexistWithCountersAndGauges_WhenRegisteredOnSameRegistry() {
        Counter counter = registry.counter("events");
        Timer timer = registry.timer("latency");
        registry.gauge("queue.size", java.util.Collections.emptyList(), 42.0, d -> d);

        counter.increment();
        timer.record(100, TimeUnit.MILLISECONDS);

        assertThat(registry.getMeters()).hasSize(3);
        assertThat(counter.count()).isEqualTo(1.0);
        assertThat(timer.count()).isEqualTo(1);
    }

    @Test
    void shouldRegisterViaConvenienceMethod_WhenCalledOnRegistry() {
        Timer timer = registry.timer("http.requests", "method", "GET", "status", "200");

        timer.record(50, TimeUnit.MILLISECONDS);

        assertThat(timer.count()).isEqualTo(1);
        assertThat(timer.getId().getTag("method")).isEqualTo("GET");
        assertThat(timer.getId().getTag("status")).isEqualTo("200");
    }

    @Test
    void shouldDeduplicateTimers_WhenRegisteredMultipleTimes() {
        Timer timer1 = registry.timer("api.latency", "endpoint", "/users");
        Timer timer2 = registry.timer("api.latency", "endpoint", "/users");

        timer1.record(100, TimeUnit.MILLISECONDS);
        timer2.record(200, TimeUnit.MILLISECONDS);

        assertThat(timer1).isSameAs(timer2);
        assertThat(timer1.count()).isEqualTo(2);
    }

    @Test
    void shouldReturnNoopTimer_WhenRegistryIsClosed() {
        registry.close();

        Timer timer = registry.timer("late.timer");

        assertThat(timer).isInstanceOf(NoopTimer.class);
        timer.record(100, TimeUnit.MILLISECONDS);
        assertThat(timer.count()).isEqualTo(0);
    }

    @Test
    void shouldSupportTimerSampleWithRegistryClock_WhenUsedEnd2End() {
        Timer.Sample sample = Timer.start(registry);
        clock.add(200, TimeUnit.MILLISECONDS);

        Timer timer = Timer.builder("request")
                .tag("method", "POST")
                .register(registry);

        long durationNs = sample.stop(timer);

        assertThat(durationNs).isEqualTo(TimeUnit.MILLISECONDS.toNanos(200));
        assertThat(timer.totalTime(TimeUnit.MILLISECONDS)).isEqualTo(200.0);
    }

    @Test
    void shouldAppearInMeterVisitor_WhenMatchCalled() {
        Timer timer = registry.timer("test.timer");
        timer.record(100, TimeUnit.MILLISECONDS);

        String result = timer.match(
                c -> "counter",
                g -> "gauge",
                t -> "timer:" + t.count(),
                ds -> "summary",
                ltt -> "longTaskTimer",
                fc -> "functionCounter",
                ft -> "functionTimer",
                m -> "other"
        );

        assertThat(result).isEqualTo("timer:1");
    }

    @Test
    void shouldThrowOnTypeConflict_WhenTimerNameMatchesExistingCounter() {
        registry.counter("shared");

        assertThatThrownBy(() -> registry.timer("shared"))
                .isInstanceOf(IllegalArgumentException.class);
    }

    @Test
    void shouldReportInBaseTimeUnit_WhenMeasureCalled() {
        Timer timer = registry.timer("test");
        timer.record(1, TimeUnit.SECONDS);

        // SimpleMeterRegistry base time unit is SECONDS
        var measurements = new java.util.ArrayList<dev.linhvu.micrometer.Measurement>();
        timer.measure().forEach(measurements::add);

        // COUNT = 1
        assertThat(measurements.get(0).getValue()).isEqualTo(1.0);
        // TOTAL_TIME in seconds = 1.0
        assertThat(measurements.get(1).getValue()).isEqualTo(1.0);
        // MAX in seconds = 1.0
        assertThat(measurements.get(2).getValue()).isEqualTo(1.0);
    }
}
```

---

## Summary

| Concept | What It Means |
|---------|---------------|
| **Timer** | Records duration + frequency of events; produces COUNT, TOTAL_TIME, and MAX measurements |
| **Timer.Sample** | Start/stop pattern where the Timer isn't known until `stop()` — enables late tag binding |
| **AbstractTimer** | Template method base class; `record(long, TimeUnit)` is `final` (validates non-negative), delegates to abstract `recordNonNegative()` |
| **CumulativeTimer** | Concrete implementation using `LongAdder` (count, total) and `TimeWindowMax` (decaying max) |
| **TimeWindowMax** | Ring-buffer-based decaying maximum; write-all/read-one pattern; CAS-based lock-free max updates |
| **NoopTimer** | Executes functions but records nothing; returned when registry is closed |
| **`monotonicTime()` vs `wallTime()`** | Monotonic for durations (never jumps backwards); wall-clock for timestamps and rotation scheduling |

**Next: Chapter 7 — DistributionSummary** — Records the distribution of non-time values (request sizes, payload lengths). Similar to Timer but without time-unit conversion — will reuse TimeWindowMax and follow the same AbstractDistributionSummary template pattern.
