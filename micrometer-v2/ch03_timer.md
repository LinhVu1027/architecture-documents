# Chapter 3: Timer — Measuring Durations

> **Feature 3** · Tier 1 · Depends on: Feature 1 (Counter & MeterRegistry)

After this chapter you will have a meter that measures **how long operations take** —
request latency, query duration, processing time — with automatic time unit conversion,
decaying max tracking, and a stopwatch API that lets you decide which timer to record
on *after* the work is done.

---

## 1. The API Contract

Before writing any implementation, let's define exactly what a client sees.

### What the client imports

```java
import simple.micrometer.api.Timer;
import simple.micrometer.api.MeterRegistry;
import simple.micrometer.internal.SimpleMeterRegistry;
```

### What the client writes

```java
// 1. Create a registry (reusing from Feature 1)
MeterRegistry registry = new SimpleMeterRegistry();

// 2. Create and use a timer
Timer timer = Timer.builder("http.request.duration")
    .tag("method", "GET")
    .description("HTTP request latency")
    .register(registry);

// 3. Record explicit duration
timer.record(Duration.ofMillis(250));

// 4. Record a block of code
timer.record(() -> handleRequest());

// 5. Record with return value
String result = timer.record(() -> computeResult());

// 6. Stopwatch pattern (deferred tag decision)
Timer.Sample sample = Timer.start(registry);
// ... do work ...
sample.stop(timer);

// 7. Read statistics
long count = timer.count();
double total = timer.totalTime(TimeUnit.MILLISECONDS);
double max = timer.max(TimeUnit.MILLISECONDS);
double mean = timer.mean(TimeUnit.MILLISECONDS);
```

### Behavioral contract

| Promise | Detail |
|---------|--------|
| Push-based recording | Client calls `record()` and the timer stores count/total/max internally |
| Three statistics | `count()`, `totalTime(unit)`, and `max(unit)` — all with time unit conversion |
| Decaying max | `max()` returns the worst recent case, not the all-time peak; decays to 0 over ~2 minutes |
| Negative-safe | Negative amounts are silently ignored |
| Code timing | `record(Runnable)`, `record(Supplier)`, `recordCallable(Callable)` measure execution time |
| Exception-safe | Duration is recorded even if the timed code throws |
| Stopwatch pattern | `Timer.start(registry)` → `sample.stop(timer)` allows deferred timer selection |
| Time unit conversion | Internal storage in nanoseconds; converted to any `TimeUnit` at query time |
| Deduplicated | Same name + tags returns the same timer instance |
| Three measurements | `measure()` returns COUNT, TOTAL_TIME, MAX (all in nanoseconds) |

---

## 2. Client Usage & Tests

These tests define the API's behavior — they were written **before** any implementation.

### Builder + Registration

```java
@Test
void shouldCreateTimer_WithBuilderAndRegister() {
    Timer timer = Timer.builder("http.request.duration")
            .tag("method", "GET")
            .description("HTTP request latency")
            .register(registry);

    assertThat(timer).isNotNull();
    assertThat(timer.getId().getName()).isEqualTo("http.request.duration");
    assertThat(timer.getId().getTag("method")).isEqualTo("GET");
    assertThat(timer.getId().getDescription()).isEqualTo("HTTP request latency");
    assertThat(timer.getId().getType()).isEqualTo(Meter.Type.TIMER);
}
```

### Explicit Duration Recording

```java
@Test
void shouldRecordExplicitDuration_InTimeUnit() {
    Timer timer = Timer.builder("latency").register(registry);

    timer.record(250, TimeUnit.MILLISECONDS);

    assertThat(timer.count()).isEqualTo(1);
    assertThat(timer.totalTime(TimeUnit.MILLISECONDS)).isCloseTo(250.0, within(0.01));
    assertThat(timer.max(TimeUnit.MILLISECONDS)).isCloseTo(250.0, within(0.01));
}
```

### Code Block Timing

```java
@Test
void shouldTimeSupplier_AndReturnResult() {
    Timer timer = Timer.builder("latency").register(registry);

    String result = timer.record(() -> {
        consumeCpu();
        return "hello";
    });

    assertThat(result).isEqualTo("hello");
    assertThat(timer.count()).isEqualTo(1);
    assertThat(timer.totalTime(TimeUnit.NANOSECONDS)).isGreaterThan(0);
}
```

### Exception Safety

```java
@Test
void shouldRecordDuration_EvenWhenRunnableThrows() {
    Timer timer = Timer.builder("latency").register(registry);

    try {
        timer.record((Runnable) () -> {
            throw new RuntimeException("boom");
        });
    } catch (RuntimeException ignored) {
    }

    assertThat(timer.count()).isEqualTo(1);  // recorded despite exception!
}
```

### Stopwatch Pattern

```java
@Test
void shouldUseSample_ForDeferredTagDecision() {
    Timer.Sample sample = Timer.start(registry);

    consumeCpu();

    // Decide tags AFTER the work is done (e.g., based on outcome)
    String status = "200";
    Timer timer = Timer.builder("http.requests")
            .tag("status", status)
            .register(registry);
    sample.stop(timer);

    assertThat(timer.count()).isEqualTo(1);
    assertThat(timer.getId().getTag("status")).isEqualTo("200");
}
```

### Mock Clock for Deterministic Tests

```java
@Test
void shouldUseMockClock_ForDeterministicTiming() {
    MockClock mockClock = new MockClock();
    MeterRegistry mockRegistry = new SimpleMeterRegistry(mockClock);

    Timer timer = Timer.builder("latency").register(mockRegistry);

    Timer.Sample sample = Timer.start(mockRegistry);
    mockClock.addNanos(500_000_000); // advance 500ms
    sample.stop(timer);

    assertThat(timer.count()).isEqualTo(1);
    assertThat(timer.totalTime(TimeUnit.MILLISECONDS)).isCloseTo(500.0, within(0.01));
}
```

---

## 3. Implementing the Call Chain — Top Down

### The call chain

```
Timer.builder("http.request.duration").tag("method","GET").register(registry)
  │
  ├─► [API Layer] Timer.Builder
  │     Stores name, tags, description. Terminal op: register(registry)
  │       → Creates Meter.Id with Type.TIMER
  │       → Calls registry.timer(id)
  │
  ├─► [API Layer] MeterRegistry.timer(Meter.Id)
  │     Calls registerMeterIfNecessary(Timer.class, id, this::newTimer)
  │     — reuses the same double-checked locking from Feature 1
  │
  └─► [Implementation Layer] SimpleMeterRegistry.newTimer(id)
        Creates CumulativeTimer(id, clock)
          └─► CumulativeTimer constructor creates TimeWindowMax(clock)
```

```
timer.record(() -> handleRequest())
  │
  ├─► [Implementation Layer] CumulativeTimer.record(Runnable)
  │     Captures clock.monotonicTime() before and after execution
  │     Calls record(durationNanos, NANOSECONDS)
  │
  └─► [Implementation Layer] CumulativeTimer.record(long, TimeUnit)
        if (amount >= 0):
          count.increment()           ← LongAdder
          totalNanos.add(nanos)       ← LongAdder
          max.record(nanos)           ← TimeWindowMax (CAS-based ring buffer)
```

### Counter vs Gauge vs Timer: Three Flavors of "Record"

```
Counter (Push, single stat)        Gauge (Pull, single stat)        Timer (Push, three stats)
──────────────────────────         ─────────────────────────        ──────────────────────────
increment(5)                       No mutation method               record(250, MILLISECONDS)
  └─► DoubleAdder.add(5)                                             ├─► LongAdder.increment()
                                   value()                            ├─► LongAdder.add(nanos)
count()                              └─► WeakRef.get()                └─► TimeWindowMax.record(nanos)
  └─► DoubleAdder.sum()                → List::size → 2.0
                                                                   count()      → LongAdder.longValue()
                                                                   totalTime(u) → nanosToUnit(sum, u)
                                                                   max(u)       → ringBuffer[current] → convert
```

Timer is the most complex meter so far: it's push-based like Counter, but tracks
three statistics instead of one, and uses a decaying ring buffer for max instead
of a simple accumulator.

### Layer 1: Timer interface + Builder + Sample

**Timer** extends `Meter` with recording methods (`record`), reading methods (`count`,
`totalTime`, `max`), and convenience methods (`mean`, `measure`).

Key design decisions:

1. **`record(long, TimeUnit)` is the core method** — all other recording methods
   ultimately delegate to it. `record(Duration)` converts to nanos and calls this.
   `record(Runnable)` measures elapsed clock time and calls this.

2. **`record(Runnable)` and `record(Supplier<T>)` are abstract**, not default methods.
   Why? Because they need the `Clock` to measure elapsed time, and interfaces can't
   hold a clock reference. Each implementation (CumulativeTimer, NoopTimer) provides
   its own version.

3. **`mean(TimeUnit)` is a default method** — it's pure math (`totalTime / count`)
   that works regardless of the underlying implementation.

4. **`measure()` returns three Measurements** in nanoseconds — COUNT, TOTAL_TIME, MAX.
   Monitoring backends use COUNT to compute rates, TOTAL_TIME for averages, and MAX
   for alerting on worst-case latency.

**Timer.Builder** follows the exact same pattern as Counter.Builder:
- Accumulates metadata (name, tags, description)
- Terminal `register(registry)` creates a `Meter.Id` with `Type.TIMER`
- Delegates to `registry.timer(id)` — the registry decides the implementation

**Timer.Sample** is the stopwatch:
```java
class Sample {
    private final long startTime;
    private final Clock clock;

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

The critical insight: **the Timer is not chosen at start time**. `Timer.start(registry)`
captures the clock and start time, but the Timer is only passed at `stop()`. This
enables a powerful pattern:

```java
Timer.Sample sample = Timer.start(registry);
Response response = callExternalService();
Timer timer = Timer.builder("external.call")
    .tag("status", String.valueOf(response.status()))  // decided AFTER the work
    .register(registry);
sample.stop(timer);
```

### Layer 2: MeterRegistry — adding timer support

The MeterRegistry gains timer methods following the same pattern as Counter and Gauge:

**Public convenience methods:**
```java
public Timer timer(String name, String... tags)
public Timer timer(String name, Iterable<Tag> tags)
```

**Package-private registration:**
```java
Timer timer(Meter.Id id) {
    return registerMeterIfNecessary(Timer.class, id, this::newTimer);
}
```

**Abstract factory:**
```java
protected abstract Timer newTimer(Meter.Id id);
```

This reuses the exact same `registerMeterIfNecessary()` double-checked locking from
Feature 1. The registration engine now handles Counter, Gauge, AND Timer without
any modification to the core logic.

### Layer 3: CumulativeTimer — three accumulators

CumulativeTimer holds three concurrent data structures:

```java
private final LongAdder count = new LongAdder();       // how many events
private final LongAdder totalNanos = new LongAdder();   // cumulative duration
private final TimeWindowMax max;                         // worst recent case
```

**Why `LongAdder` instead of `AtomicLong`?** Under high contention (many threads
recording concurrently), `LongAdder` uses striped cells — each thread updates its
own cell, avoiding CAS retry loops. The cost is slightly more expensive reads
(`sum()` aggregates all cells), but timers are recorded far more often than read.

**The core `record(long, TimeUnit)` method:**
```java
public void record(long amount, TimeUnit unit) {
    if (amount >= 0) {
        long nanos = unit.toNanos(amount);
        count.increment();
        totalNanos.add(nanos);
        max.record(nanos);
    }
}
```

Three operations, all lock-free. Negative amounts are silently dropped.

**The code-timing methods** all follow the same pattern:
```java
public void record(Runnable f) {
    long start = clock.monotonicTime();
    try {
        f.run();
    } finally {
        record(clock.monotonicTime() - start, TimeUnit.NANOSECONDS);
    }
}
```

The `finally` block ensures duration is recorded even if the code throws. This is
why NoopTimer must also execute the code — `timer.record(() -> doWork())` is a
contract that `doWork()` will be called regardless of meter state.

**Reading with unit conversion:**
```java
public double totalTime(TimeUnit unit) {
    return TimeWindowMax.nanosToUnit(totalNanos.doubleValue(), unit);
}
```

All internal storage is in nanoseconds. Conversion happens only at query time.
This keeps the hot path (recording) fast — no division, no floating-point math.

### Layer 4: TimeWindowMax — the decaying ring buffer

This is the most interesting internal class in Feature 3. A simple
`AtomicLong maxNanos` would track the all-time maximum, which is useless for
monitoring — you want "what was the worst latency in the last 2 minutes?", not
"what was the worst latency since the server started?"

TimeWindowMax uses a ring buffer:

```
Ring buffer (3 slots, 40 seconds each, total 2 minutes):

  Slot 0          Slot 1          Slot 2
┌──────────┐  ┌──────────┐  ┌──────────┐
│ max: 500 │  │ max: 300 │  │ max: 0   │  ← currentBucket
└──────────┘  └──────────┘  └──────────┘
  t-80s         t-40s         now (current)
```

**On record**: updates ALL slots:
```java
public void record(long nanoAmount) {
    rotate();
    for (AtomicLong slot : ringBuffer) {
        updateMax(slot, nanoAmount);
    }
}
```

Why all slots? Because a value recorded now is relevant to every time window that
includes "now". When slot 2 rotates out, slot 0 still remembers this value.

**On poll**: reads only the current slot:
```java
public double poll(TimeUnit unit) {
    rotate();
    long nanos = ringBuffer[currentBucket].get();
    return nanosToUnit((double) nanos, unit);
}
```

**On rotation**: zeroes out the new current slot and advances the pointer:
```java
currentBucket = (currentBucket + 1) % ringBuffer.length;
ringBuffer[currentBucket].set(0);
```

If more than 2 minutes pass without any recordings, all slots get zeroed — the max
naturally decays to 0.

**Lock-free max update** uses a CAS loop:
```java
private void updateMax(AtomicLong slot, long sample) {
    long curMax;
    do {
        curMax = slot.get();
    } while (curMax < sample && !slot.compareAndSet(curMax, sample));
}
```

If the sample is less than the current max, the loop exits on the first check
(`curMax < sample` is false). If greater, it attempts a CAS. Under contention,
the CAS may fail if another thread updated the max concurrently — but since we're
tracking the maximum, the retry will either succeed or discover an even larger
value (which means our sample is no longer the max, so we exit).

### Layer 5: SimpleMeterRegistry — one line

```java
@Override
protected Timer newTimer(Meter.Id id) {
    return new CumulativeTimer(id, clock);
}
```

The `clock` field (from the `MeterRegistry` superclass) is passed to CumulativeTimer,
which uses it for the `record(Runnable)` timing methods and passes it to TimeWindowMax
for rotation timing.

---

## 4. Try It Yourself

1. **Add `Timer.wrap(Runnable)`** — a method that returns a `Runnable` which, when
   executed, records its duration on this timer. The real Micrometer has this. Hint:
   `return () -> record(f);`

2. **What happens if you call `timer.record(0, TimeUnit.MILLISECONDS)`?** Is zero a
   valid recording? Check the guard in CumulativeTimer — `amount >= 0` means zero
   IS recorded. Is this correct? (Yes — a zero-duration event still counts.)

3. **Add a `Timer.ResourceSample`** that implements `AutoCloseable`. It would work
   with try-with-resources: `try (var s = Timer.resource(registry, "op")) { ... }`.
   The real Micrometer added this in 1.6.0.

---

## 5. Why This Works

### Everything stored in nanoseconds

By storing all durations as nanosecond longs, the recording path is pure integer
arithmetic — no floating-point, no division, no unit conversion overhead. The
conversion to the caller's desired `TimeUnit` happens only at query time, which
is called far less frequently than recording.

### Decaying max vs all-time max

An all-time max is useless for monitoring dashboards. If your server had one bad
request 3 months ago that took 30 seconds, the `max` metric would show 30s forever,
drowning out the fact that your current worst-case is 500ms. TimeWindowMax's ring
buffer ensures the max reflects *recent* behavior — old peaks expire, and if traffic
is healthy, max decays toward zero.

### The finally block guarantees recording

Every code-timing method uses `try { ... } finally { record(...) }`. This means:
- If the code succeeds, duration is recorded
- If the code throws, duration is still recorded
- The exception is propagated unchanged

This is critical for observability: you especially want to know the latency of
*failed* requests, not just successful ones.

### Timer.Sample enables deferred decisions

Without Sample, you'd need to choose the timer before starting the work:
```java
Timer timer = Timer.builder("http.requests").tag("status", ???).register(registry);
timer.record(() -> handleRequest());  // but we don't know the status yet!
```

With Sample, you start timing immediately and choose the timer based on the outcome:
```java
Timer.Sample sample = Timer.start(registry);
Response response = handleRequest();
Timer.builder("http.requests").tag("status", response.status()).register(registry);
sample.stop(timer);
```

---

## 6. What We Enhanced

| Component | State before (Feature 2) | Enhancement in Feature 3 | Will be enhanced by |
|-----------|-------------------------|-------------------------|---------------------|
| `MeterRegistry` | Supports Counter + Gauge | +Timer registration, +2 convenience methods, +`newTimer()` abstract | F4: +DistributionSummary, F5: +MeterFilter |
| `SimpleMeterRegistry` | Creates CumulativeCounter + DefaultGauge | +`newTimer()` → CumulativeTimer | F4: +CumulativeDistributionSummary |
| `Meter.Type` | Had `TIMER` enum value (unused) | Now used in Timer's Meter.Id | Stable |

---

## 7. Connection to Real Framework

| Simplified Class | Real Framework Class | Key Simplification |
|-----------------|---------------------|--------------------|
| `api/Timer` | `io.micrometer.core.instrument.Timer` | No `HistogramSupport` interface, no `@Timed` annotation builder, no `ResourceSample`, no `wrap()` methods |
| `api/Timer.Builder` | `io.micrometer.core.instrument.Timer.Builder` (extends `AbstractTimerBuilder`) | No `DistributionStatisticConfig` (percentiles, histograms, SLOs), no `PauseDetector`, no `minimumExpectedValue`/`maximumExpectedValue` |
| `api/Timer.Sample` | `io.micrometer.core.instrument.Timer.Sample` | Identical design — same start/stop pattern with deferred timer selection |
| `internal/CumulativeTimer` | `io.micrometer.core.instrument.cumulative.CumulativeTimer` | No `AbstractTimer` base class, no histogram recording, no pause detection. Uses `LongAdder` for total (real uses `LongAdder` too) |
| `internal/NoopTimer` | `io.micrometer.core.instrument.noop.NoopTimer` | No `NoopMeter` base class, no `HistogramSnapshot` |
| `internal/TimeWindowMax` | `io.micrometer.core.instrument.distribution.TimeWindowMax` | Hardcoded 2min/3-buffer defaults instead of `DistributionStatisticConfig`. No `AtomicIntegerFieldUpdater` for rotation guard (simplified to non-concurrent rotation). Uses `switch` expression for `nanosToUnit` instead of separate `TimeUtils` utility class |

---

## 8. Complete Code

All files created or modified in this feature. Copy all `[NEW]` files and replace
all `[MODIFIED]` files to build the project from Feature 1 + Feature 2 + Feature 3.

### `src/main/java/simple/micrometer/api/Timer.java` [NEW]

```java
package simple.micrometer.api;

import java.time.Duration;
import java.util.Arrays;
import java.util.concurrent.Callable;
import java.util.concurrent.TimeUnit;
import java.util.function.Supplier;

/**
 * A meter that measures the duration and frequency of events — request latency,
 * query time, processing duration. Timers track three statistics: count (how
 * many), totalTime (cumulative duration), and max (worst recent case).
 *
 * <p>Usage:
 * <pre>{@code
 * Timer timer = Timer.builder("http.request.duration")
 *     .tag("method", "GET")
 *     .description("HTTP request latency")
 *     .register(registry);
 *
 * // Record explicit duration
 * timer.record(Duration.ofMillis(250));
 *
 * // Record a block of code
 * timer.record(() -> handleRequest());
 *
 * // Stopwatch pattern (deferred tag decision)
 * Timer.Sample sample = Timer.start(registry);
 * // ... do work ...
 * sample.stop(timer);
 *
 * // Read statistics
 * long count = timer.count();
 * double total = timer.totalTime(TimeUnit.MILLISECONDS);
 * double max = timer.max(TimeUnit.MILLISECONDS);
 * }</pre>
 *
 * <p>Simplified from {@code io.micrometer.core.instrument.Timer}.
 */
public interface Timer extends Meter {

    // ── Recording ─────────────────────────────────────────────────────

    /**
     * Records a duration in the given time unit. This is the core recording
     * method — all other {@code record} overloads ultimately delegate here.
     * Negative amounts are ignored.
     */
    void record(long amount, TimeUnit unit);

    /**
     * Records a {@link Duration}.
     */
    default void record(Duration duration) {
        record(duration.toNanos(), TimeUnit.NANOSECONDS);
    }

    /**
     * Executes the runnable and records its execution time.
     */
    void record(Runnable f);

    /**
     * Executes the supplier, records its execution time, and returns the result.
     */
    <T> T record(Supplier<T> f);

    /**
     * Executes the callable, records its execution time, and returns the result.
     * Unlike {@link #record(Supplier)}, this propagates checked exceptions.
     */
    <T> T recordCallable(Callable<T> f) throws Exception;

    // ── Reading ───────────────────────────────────────────────────────

    /**
     * Returns the number of times {@code record} has been called.
     */
    long count();

    /**
     * Returns the cumulative total time of all recorded events, converted
     * to the requested unit.
     */
    double totalTime(TimeUnit unit);

    /**
     * Returns the maximum duration of a single event within the recent
     * time window, converted to the requested unit. The max decays over
     * time — if no new events are recorded, it gradually returns to 0.
     */
    double max(TimeUnit unit);

    /**
     * Returns the mean duration (totalTime / count), or 0 if no events
     * have been recorded.
     */
    default double mean(TimeUnit unit) {
        long n = count();
        return n == 0 ? 0 : totalTime(unit) / n;
    }

    // ── Measurement ───────────────────────────────────────────────────

    /**
     * Returns three measurements: COUNT, TOTAL_TIME (in nanoseconds), and MAX
     * (in nanoseconds). Backends use these to compute rates, averages, and peaks.
     */
    @Override
    default Iterable<Measurement> measure() {
        return Arrays.asList(
                new Measurement(this::count, Statistic.COUNT),
                new Measurement(() -> totalTime(TimeUnit.NANOSECONDS), Statistic.TOTAL_TIME),
                new Measurement(() -> max(TimeUnit.NANOSECONDS), Statistic.MAX)
        );
    }

    // ── Factory ───────────────────────────────────────────────────────

    /**
     * Creates a new Timer builder.
     */
    static Builder builder(String name) {
        return new Builder(name);
    }

    /**
     * Starts a stopwatch using the registry's clock. The Timer is chosen
     * later at {@link Sample#stop(Timer)}, enabling deferred tag decisions.
     */
    static Sample start(MeterRegistry registry) {
        return new Sample(registry.getClock());
    }

    /**
     * Starts a stopwatch using the given clock.
     */
    static Sample start(Clock clock) {
        return new Sample(clock);
    }

    // ── Sample (stopwatch) ────────────────────────────────────────────

    /**
     * A timing sample that records the duration between {@link Timer#start}
     * and {@link #stop(Timer)}. The Timer is not chosen until stop-time,
     * which allows tags to be decided based on the outcome of the timed
     * operation (e.g., HTTP status code).
     *
     * <p>Simplified from {@code io.micrometer.core.instrument.Timer.Sample}.
     */
    class Sample {

        private final long startTime;
        private final Clock clock;

        Sample(Clock clock) {
            this.clock = clock;
            this.startTime = clock.monotonicTime();
        }

        /**
         * Stops the sample and records the elapsed duration on the given timer.
         *
         * @return the elapsed time in nanoseconds
         */
        public long stop(Timer timer) {
            long durationNs = clock.monotonicTime() - startTime;
            timer.record(durationNs, TimeUnit.NANOSECONDS);
            return durationNs;
        }

    }

    // ── Builder ───────────────────────────────────────────────────────

    /**
     * Fluent builder for constructing and registering timers.
     * Follows the same pattern as {@link Counter.Builder} and
     * {@link Gauge.Builder}: accumulates metadata, then delegates to the
     * registry for concrete type creation.
     *
     * <p>Simplified from {@code io.micrometer.core.instrument.Timer.Builder}.
     * The real builder also supports {@code DistributionStatisticConfig}
     * (percentiles, histograms, SLOs) and {@code PauseDetector}.
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

        /**
         * Terminal operation: registers (or retrieves) a timer with the given
         * registry. The registry decides which concrete Timer to create.
         */
        public Timer register(MeterRegistry registry) {
            Meter.Id id = new Meter.Id(name, tags, baseUnit, description, Meter.Type.TIMER);
            return registry.timer(id);
        }

    }

}
```

### `src/main/java/simple/micrometer/internal/TimeWindowMax.java` [NEW]

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

### `src/main/java/simple/micrometer/internal/CumulativeTimer.java` [NEW]

```java
package simple.micrometer.internal;

import simple.micrometer.api.Clock;
import simple.micrometer.api.Meter;
import simple.micrometer.api.Timer;

import java.util.concurrent.Callable;
import java.util.concurrent.TimeUnit;
import java.util.concurrent.atomic.LongAdder;
import java.util.function.Supplier;

/**
 * A timer that tracks count, total time, and decaying max using lock-free
 * accumulators. All values are stored internally in nanoseconds and converted
 * to the requested unit at query time.
 *
 * <p>Thread-safe: {@link LongAdder} provides striped accumulation for count
 * and total, avoiding CAS contention under high throughput.
 * {@link TimeWindowMax} uses a CAS-based ring buffer for the decaying max.
 *
 * <p>Simplified from {@code io.micrometer.core.instrument.cumulative.CumulativeTimer}.
 * The real version extends {@code AbstractTimer} which adds histogram support
 * and pause detection. Our version implements {@link Timer} directly.
 */
public class CumulativeTimer implements Timer {

    private final Meter.Id id;
    private final Clock clock;
    private final LongAdder count = new LongAdder();
    private final LongAdder totalNanos = new LongAdder();
    private final TimeWindowMax max;

    public CumulativeTimer(Meter.Id id, Clock clock) {
        this.id = id;
        this.clock = clock;
        this.max = new TimeWindowMax(clock);
    }

    // ── Recording ─────────────────────────────────────────────────────

    @Override
    public void record(long amount, TimeUnit unit) {
        if (amount >= 0) {
            long nanos = unit.toNanos(amount);
            count.increment();
            totalNanos.add(nanos);
            max.record(nanos);
        }
    }

    @Override
    public void record(Runnable f) {
        long start = clock.monotonicTime();
        try {
            f.run();
        } finally {
            record(clock.monotonicTime() - start, TimeUnit.NANOSECONDS);
        }
    }

    @Override
    public <T> T record(Supplier<T> f) {
        long start = clock.monotonicTime();
        try {
            return f.get();
        } finally {
            record(clock.monotonicTime() - start, TimeUnit.NANOSECONDS);
        }
    }

    @Override
    public <T> T recordCallable(Callable<T> f) throws Exception {
        long start = clock.monotonicTime();
        try {
            return f.call();
        } finally {
            record(clock.monotonicTime() - start, TimeUnit.NANOSECONDS);
        }
    }

    // ── Reading ───────────────────────────────────────────────────────

    @Override
    public long count() {
        return count.longValue();
    }

    @Override
    public double totalTime(TimeUnit unit) {
        return TimeWindowMax.nanosToUnit(totalNanos.doubleValue(), unit);
    }

    @Override
    public double max(TimeUnit unit) {
        return max.poll(unit);
    }

    // ── Meter ─────────────────────────────────────────────────────────

    @Override
    public Meter.Id getId() {
        return id;
    }

}
```

### `src/main/java/simple/micrometer/internal/NoopTimer.java` [NEW]

```java
package simple.micrometer.internal;

import simple.micrometer.api.Meter;
import simple.micrometer.api.Timer;

import java.util.concurrent.Callable;
import java.util.concurrent.TimeUnit;
import java.util.function.Supplier;

/**
 * A no-op Timer that still executes the code passed to {@code record()} but
 * discards all timing data. Used when a meter is denied by a filter or when
 * the registry is not configured.
 *
 * <p>Important: the Runnable/Supplier/Callable IS still executed — only the
 * metrics are dropped. This ensures application behavior is unaffected by
 * meter filtering decisions.
 *
 * <p>Simplified from {@code io.micrometer.core.instrument.noop.NoopTimer}.
 */
public class NoopTimer implements Timer {

    private final Meter.Id id;

    public NoopTimer(Meter.Id id) {
        this.id = id;
    }

    @Override
    public void record(long amount, TimeUnit unit) {
        // no-op
    }

    @Override
    public void record(Runnable f) {
        f.run();
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
    public Meter.Id getId() {
        return id;
    }

}
```

### `src/main/java/simple/micrometer/api/MeterRegistry.java` [MODIFIED]

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

```java
package simple.micrometer.internal;

import simple.micrometer.api.Clock;
import simple.micrometer.api.Counter;
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

}
```

### `src/test/java/simple/micrometer/api/TimerTest.java` [NEW]

```java
package simple.micrometer.api;

import simple.micrometer.internal.SimpleMeterRegistry;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.time.Duration;
import java.util.concurrent.Callable;
import java.util.concurrent.TimeUnit;
import java.util.concurrent.atomic.AtomicLong;
import java.util.function.Supplier;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;
import static org.assertj.core.api.Assertions.within;

class TimerTest {

    private MeterRegistry registry;

    @BeforeEach
    void setUp() {
        registry = new SimpleMeterRegistry();
    }

    // ── Builder + Registration ────────────────────────────────────────

    @Test
    void shouldCreateTimer_WithBuilderAndRegister() {
        Timer timer = Timer.builder("http.request.duration")
                .tag("method", "GET")
                .description("HTTP request latency")
                .register(registry);

        assertThat(timer).isNotNull();
        assertThat(timer.getId().getName()).isEqualTo("http.request.duration");
        assertThat(timer.getId().getTag("method")).isEqualTo("GET");
        assertThat(timer.getId().getDescription()).isEqualTo("HTTP request latency");
        assertThat(timer.getId().getType()).isEqualTo(Meter.Type.TIMER);
    }

    @Test
    void shouldReturnSameTimer_WhenRegisteredTwiceWithSameId() {
        Timer timer1 = Timer.builder("latency").tag("k", "v").register(registry);
        Timer timer2 = Timer.builder("latency").tag("k", "v").register(registry);

        assertThat(timer1).isSameAs(timer2);
    }

    @Test
    void shouldReturnDifferentTimers_WhenTagsDiffer() {
        Timer timer1 = Timer.builder("latency").tag("env", "prod").register(registry);
        Timer timer2 = Timer.builder("latency").tag("env", "dev").register(registry);

        assertThat(timer1).isNotSameAs(timer2);
    }

    @Test
    void shouldCreateTimer_WithRegistryShorthand() {
        Timer timer = registry.timer("http.latency", "method", "POST");

        assertThat(timer).isNotNull();
        assertThat(timer.getId().getName()).isEqualTo("http.latency");
        assertThat(timer.getId().getTag("method")).isEqualTo("POST");
    }

    // ── Recording explicit durations ──────────────────────────────────

    @Test
    void shouldRecordExplicitDuration_InTimeUnit() {
        Timer timer = Timer.builder("latency").register(registry);

        timer.record(250, TimeUnit.MILLISECONDS);

        assertThat(timer.count()).isEqualTo(1);
        assertThat(timer.totalTime(TimeUnit.MILLISECONDS)).isCloseTo(250.0, within(0.01));
        assertThat(timer.max(TimeUnit.MILLISECONDS)).isCloseTo(250.0, within(0.01));
    }

    @Test
    void shouldRecordDuration_AsJavaDuration() {
        Timer timer = Timer.builder("latency").register(registry);

        timer.record(Duration.ofMillis(100));
        timer.record(Duration.ofMillis(300));

        assertThat(timer.count()).isEqualTo(2);
        assertThat(timer.totalTime(TimeUnit.MILLISECONDS)).isCloseTo(400.0, within(0.01));
    }

    @Test
    void shouldAccumulateMultipleRecordings() {
        Timer timer = Timer.builder("latency").register(registry);

        timer.record(100, TimeUnit.MILLISECONDS);
        timer.record(200, TimeUnit.MILLISECONDS);
        timer.record(300, TimeUnit.MILLISECONDS);

        assertThat(timer.count()).isEqualTo(3);
        assertThat(timer.totalTime(TimeUnit.MILLISECONDS)).isCloseTo(600.0, within(0.01));
        assertThat(timer.max(TimeUnit.MILLISECONDS)).isCloseTo(300.0, within(0.01));
    }

    @Test
    void shouldIgnoreNegativeAmounts() {
        Timer timer = Timer.builder("latency").register(registry);

        timer.record(-100, TimeUnit.MILLISECONDS);

        assertThat(timer.count()).isEqualTo(0);
        assertThat(timer.totalTime(TimeUnit.MILLISECONDS)).isEqualTo(0.0);
    }

    // ── Time unit conversion ──────────────────────────────────────────

    @Test
    void shouldConvertTimeUnits_WhenReading() {
        Timer timer = Timer.builder("latency").register(registry);

        timer.record(1, TimeUnit.SECONDS);

        assertThat(timer.totalTime(TimeUnit.SECONDS)).isCloseTo(1.0, within(0.001));
        assertThat(timer.totalTime(TimeUnit.MILLISECONDS)).isCloseTo(1000.0, within(0.1));
        assertThat(timer.totalTime(TimeUnit.NANOSECONDS)).isCloseTo(1_000_000_000.0, within(1.0));
    }

    // ── Recording code blocks ─────────────────────────────────────────

    @Test
    void shouldTimeRunnable() {
        Timer timer = Timer.builder("latency").register(registry);

        timer.record(() -> {
            // simulate work
            consumeCpu();
        });

        assertThat(timer.count()).isEqualTo(1);
        assertThat(timer.totalTime(TimeUnit.NANOSECONDS)).isGreaterThan(0);
    }

    @Test
    void shouldTimeSupplier_AndReturnResult() {
        Timer timer = Timer.builder("latency").register(registry);

        String result = timer.record(() -> {
            consumeCpu();
            return "hello";
        });

        assertThat(result).isEqualTo("hello");
        assertThat(timer.count()).isEqualTo(1);
        assertThat(timer.totalTime(TimeUnit.NANOSECONDS)).isGreaterThan(0);
    }

    @Test
    void shouldTimeCallable_AndReturnResult() throws Exception {
        Timer timer = Timer.builder("latency").register(registry);

        String result = timer.recordCallable(() -> {
            consumeCpu();
            return "computed";
        });

        assertThat(result).isEqualTo("computed");
        assertThat(timer.count()).isEqualTo(1);
    }

    @Test
    void shouldTimeCallable_AndPropagateCheckedException() {
        Timer timer = Timer.builder("latency").register(registry);

        assertThatThrownBy(() -> timer.recordCallable(() -> {
            throw new Exception("test error");
        })).isInstanceOf(Exception.class).hasMessage("test error");

        // Duration is still recorded even when exception is thrown
        assertThat(timer.count()).isEqualTo(1);
    }

    @Test
    void shouldRecordDuration_EvenWhenRunnableThrows() {
        Timer timer = Timer.builder("latency").register(registry);

        try {
            timer.record((Runnable) () -> {
                throw new RuntimeException("boom");
            });
        } catch (RuntimeException ignored) {
        }

        assertThat(timer.count()).isEqualTo(1);
    }

    // ── Timer.Sample (stopwatch pattern) ──────────────────────────────

    @Test
    void shouldMeasureDuration_WithSampleStopwatch() {
        Timer.Sample sample = Timer.start(registry);

        consumeCpu();

        Timer timer = Timer.builder("latency").register(registry);
        long durationNs = sample.stop(timer);

        assertThat(durationNs).isGreaterThan(0);
        assertThat(timer.count()).isEqualTo(1);
        assertThat(timer.totalTime(TimeUnit.NANOSECONDS)).isCloseTo((double) durationNs, within(1.0));
    }

    @Test
    void shouldStartSample_WithExplicitClock() {
        Timer.Sample sample = Timer.start(Clock.SYSTEM);
        Timer timer = Timer.builder("latency").register(registry);

        long durationNs = sample.stop(timer);

        assertThat(durationNs).isGreaterThanOrEqualTo(0);
        assertThat(timer.count()).isEqualTo(1);
    }

    @Test
    void shouldUseSample_ForDeferredTagDecision() {
        Timer.Sample sample = Timer.start(registry);

        consumeCpu();

        // Decide tags after the work is done (e.g., based on outcome)
        String status = "200";
        Timer timer = Timer.builder("http.requests")
                .tag("status", status)
                .register(registry);
        sample.stop(timer);

        assertThat(timer.count()).isEqualTo(1);
        assertThat(timer.getId().getTag("status")).isEqualTo("200");
    }

    // ── Mean ──────────────────────────────────────────────────────────

    @Test
    void shouldComputeMean() {
        Timer timer = Timer.builder("latency").register(registry);

        timer.record(100, TimeUnit.MILLISECONDS);
        timer.record(200, TimeUnit.MILLISECONDS);
        timer.record(300, TimeUnit.MILLISECONDS);

        assertThat(timer.mean(TimeUnit.MILLISECONDS)).isCloseTo(200.0, within(0.01));
    }

    @Test
    void shouldReturnZeroMean_WhenNoRecordings() {
        Timer timer = Timer.builder("latency").register(registry);

        assertThat(timer.mean(TimeUnit.MILLISECONDS)).isEqualTo(0.0);
    }

    // ── Measurements ──────────────────────────────────────────────────

    @Test
    void shouldProduceThreeMeasurements() {
        Timer timer = Timer.builder("latency").register(registry);
        timer.record(100, TimeUnit.MILLISECONDS);

        Iterable<Measurement> measurements = timer.measure();

        assertThat(measurements).hasSize(3);

        var iter = measurements.iterator();
        Measurement countM = iter.next();
        Measurement totalM = iter.next();
        Measurement maxM = iter.next();

        assertThat(countM.getStatistic()).isEqualTo(Statistic.COUNT);
        assertThat(countM.getValue()).isEqualTo(1.0);

        assertThat(totalM.getStatistic()).isEqualTo(Statistic.TOTAL_TIME);
        assertThat(totalM.getValue()).isCloseTo(100_000_000.0, within(1.0)); // nanos

        assertThat(maxM.getStatistic()).isEqualTo(Statistic.MAX);
        assertThat(maxM.getValue()).isCloseTo(100_000_000.0, within(1.0)); // nanos
    }

    // ── Mock clock for deterministic tests ────────────────────────────

    @Test
    void shouldUseMockClock_ForDeterministicTiming() {
        MockClock mockClock = new MockClock();
        MeterRegistry mockRegistry = new SimpleMeterRegistry(mockClock);

        Timer timer = Timer.builder("latency").register(mockRegistry);

        Timer.Sample sample = Timer.start(mockRegistry);
        mockClock.addNanos(500_000_000); // advance 500ms
        sample.stop(timer);

        assertThat(timer.count()).isEqualTo(1);
        assertThat(timer.totalTime(TimeUnit.MILLISECONDS)).isCloseTo(500.0, within(0.01));
    }

    // ── Coexistence with other meter types ────────────────────────────

    @Test
    void shouldCoexist_WithCounterAndGauge() {
        Counter counter = Counter.builder("requests").register(registry);
        Gauge gauge = Gauge.builder("queue.size", new java.util.ArrayList<>(), java.util.List::size)
                .register(registry);
        Timer timer = Timer.builder("latency").register(registry);

        counter.increment();
        timer.record(100, TimeUnit.MILLISECONDS);

        assertThat(registry.getMeters()).hasSize(3);
        assertThat(counter.count()).isEqualTo(1.0);
        assertThat(gauge.value()).isEqualTo(0.0);
        assertThat(timer.count()).isEqualTo(1);
    }

    // ── Zero state ────────────────────────────────────────────────────

    @Test
    void shouldReturnZeros_BeforeAnyRecording() {
        Timer timer = Timer.builder("latency").register(registry);

        assertThat(timer.count()).isEqualTo(0);
        assertThat(timer.totalTime(TimeUnit.MILLISECONDS)).isEqualTo(0.0);
        assertThat(timer.max(TimeUnit.MILLISECONDS)).isEqualTo(0.0);
        assertThat(timer.mean(TimeUnit.MILLISECONDS)).isEqualTo(0.0);
    }

    // ── Helpers ───────────────────────────────────────────────────────

    /**
     * Consume a small amount of CPU time so timing tests see non-zero durations.
     */
    private void consumeCpu() {
        long sum = 0;
        for (int i = 0; i < 10_000; i++) {
            sum += i;
        }
        // prevent dead-code elimination
        if (sum == -1) throw new AssertionError();
    }

    /**
     * A mock clock for deterministic timer tests. Wall time advances with
     * nanos to ensure TimeWindowMax rotation doesn't interfere.
     */
    private static class MockClock implements Clock {

        private final AtomicLong monotonicNanos = new AtomicLong(0);
        private final AtomicLong wallMillis = new AtomicLong(System.currentTimeMillis());

        @Override
        public long wallTime() {
            return wallMillis.get();
        }

        @Override
        public long monotonicTime() {
            return monotonicNanos.get();
        }

        void addNanos(long nanos) {
            monotonicNanos.addAndGet(nanos);
            wallMillis.addAndGet(nanos / 1_000_000);
        }

    }

}
```
