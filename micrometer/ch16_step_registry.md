# Chapter 16: Step Registry

> Line references based on commit `2c8a4606c` of the Micrometer repository.

## Build Challenge

| Current State | Limitation | Objective |
|---------------|------------|-----------|
| `PushMeterRegistry` periodically calls `publish()` to export metrics to a backend. But the meters it uses are cumulative — counters grow forever, timers accumulate total time | Push-based backends (Datadog, InfluxDB, OTLP-delta) expect **per-interval rates**, not ever-growing cumulative totals. A counter that reads 1,000,000 tells you nothing about the last minute. | Build step-based meters that accumulate values during a time interval and **reset at step boundaries**, plus a `StepMeterRegistry` with a **second scheduling loop** that triggers the reset |

---

## 16.1 The Integration Point: Overriding Factory Methods in PushMeterRegistry

The step registry connects to the existing system by **overriding every `new*()` factory method** in `MeterRegistry` to create step-based meter variants instead of cumulative ones. It also overrides `start()` and `close()` to add the meter polling lifecycle.

**Modifying:** `src/main/java/dev/linhvu/micrometer/push/PushMeterRegistry.java`
**Change:** Make `publishSafelyOrSkipIfInProgress()` `protected` (was package-private) so `StepMeterRegistry` in the `step` package can call it during shutdown.

```java
// Before (package-private):
void publishSafelyOrSkipIfInProgress() {

// After (protected):
protected void publishSafelyOrSkipIfInProgress() {
```

The core integration is in `StepMeterRegistry.close()`:

```java
@Override
public void close() {
    if (meterPollingService != null) {
        meterPollingService.shutdown();           // 1. Stop polling
    }
    if (config.enabled() && !isClosed()) {
        if (shouldPublishDataForLastStep()) {
            publishSafelyOrSkipIfInProgress();    // 2. Publish unpublished rollover data
        }
        closingRolloverStepMeters();              // 3. Drain partial-step accumulators
    }
    super.close();                                // 4. Final publish + set closed flag
}
```

Two key decisions:

1. **Why a second scheduling loop instead of polling inside `publish()`?** The polling loop fires 1ms into each step boundary, while publishing fires at a random offset (2ms+). If we polled inside `publish()`, we'd have a race condition: the publish might fire before the step boundary, reading stale data from the previous-previous step.

2. **Why force a closing rollover?** When `close()` is called mid-step, the current accumulator holds partial data that hasn't rolled over yet. `closingRolloverStepMeters()` drains it into "previous" so the final `publish()` (in `super.close()`) can see it.

This connects **step-based meter semantics** to the **push registry lifecycle**. To make it work, we need:
- `StepValue<V>` — the core accumulate-and-drain primitive
- `StepDouble` / `StepLong` / `StepTuple2` — concrete accumulators
- `StepCounter`, `StepTimer`, `StepDistributionSummary` — step-based meter implementations
- `StepFunctionCounter`, `StepFunctionTimer` — step-based function meter adapters
- `StepMeterRegistry` — the registry with two scheduling loops

## 16.2 StepValue — The Core Step Primitive

**New file:** `src/main/java/dev/linhvu/micrometer/step/StepValue.java`

```java
abstract class StepValue<V> {

    private final Clock clock;
    private final long stepMillis;
    private final AtomicLong lastInitPos;
    private volatile V previous;

    StepValue(Clock clock, long stepMillis) {
        this.clock = clock;
        this.stepMillis = stepMillis;
        this.previous = noValue();
        this.lastInitPos = new AtomicLong(clock.wallTime() / stepMillis);
    }

    abstract Supplier<V> valueSupplier();  // reads-and-resets the accumulator
    abstract V noValue();                  // "zero" value

    private void rollCount(long now) {
        long stepTime = now / stepMillis;
        long lastInit = lastInitPos.get();
        if (lastInit < stepTime && lastInitPos.compareAndSet(lastInit, stepTime)) {
            V value = valueSupplier().get();
            if (lastInit == stepTime - 1) {
                previous = value;           // immediate predecessor — use drained value
            } else {
                previous = noValue();       // gap — discard stale data
            }
        }
    }

    V poll() {
        rollCount(clock.wallTime());
        return previous;
    }

    void _closingRollover() {
        lastInitPos.set(Long.MAX_VALUE);    // prevent future rollovers
        previous = valueSupplier().get();   // drain current accumulator
    }
}
```

The CAS on `lastInitPos` is the key to thread safety. Multiple threads may call `poll()` simultaneously, but `compareAndSet` ensures exactly one thread performs the drain. The gap detection (`lastInit == stepTime - 1`) prevents stale data from leaking across non-adjacent steps — if a step was missed, the data is discarded rather than attributed to the wrong interval.

## 16.3 StepDouble, StepLong, and StepTuple2

**New file:** `src/main/java/dev/linhvu/micrometer/step/StepDouble.java`

```java
class StepDouble extends StepValue<Double> {

    private final DoubleAdder current = new DoubleAdder();

    StepDouble(Clock clock, long stepMillis) {
        super(clock, stepMillis);
    }

    @Override Supplier<Double> valueSupplier() { return current::sumThenReset; }
    @Override Double noValue() { return 0.0; }
    DoubleAdder getCurrent() { return current; }
}
```

**New file:** `src/main/java/dev/linhvu/micrometer/step/StepLong.java`

```java
class StepLong extends StepValue<Long> {

    private final LongAdder current = new LongAdder();

    StepLong(Clock clock, long stepMillis) {
        super(clock, stepMillis);
    }

    @Override Supplier<Long> valueSupplier() { return current::sumThenReset; }
    @Override Long noValue() { return 0L; }
    LongAdder getCurrent() { return current; }
}
```

**New file:** `src/main/java/dev/linhvu/micrometer/step/StepTuple2.java`

`StepTuple2` rolls over **two values atomically** — critical for Timer (count + total time) and DistributionSummary (count + total amount):

```java
class StepTuple2<T1, T2> {

    private final Clock clock;
    private final long stepMillis;
    private final AtomicLong lastInitPos;
    private final T1 t1NoValue;
    private final T2 t2NoValue;
    private final Supplier<T1> t1Supplier;
    private final Supplier<T2> t2Supplier;
    private volatile T1 t1Previous;
    private volatile T2 t2Previous;

    // Same CAS-based rollover as StepValue, but drains both suppliers together
    private void rollCount(long now) {
        long stepTime = now / stepMillis;
        long lastInit = lastInitPos.get();
        if (lastInit < stepTime && lastInitPos.compareAndSet(lastInit, stepTime)) {
            T1 v1 = t1Supplier.get();
            T2 v2 = t2Supplier.get();
            if (lastInit == stepTime - 1) {
                t1Previous = v1;
                t2Previous = v2;
            } else {
                t1Previous = t1NoValue;
                t2Previous = t2NoValue;
            }
        }
    }

    T1 poll1() { rollCount(clock.wallTime()); return t1Previous; }
    T2 poll2() { rollCount(clock.wallTime()); return t2Previous; }
}
```

## 16.4 Step Meter Implementations

### StepMeter marker interface

**New file:** `src/main/java/dev/linhvu/micrometer/step/StepMeter.java`

```java
interface StepMeter {
    void _closingRollover();
}
```

### StepCounter

**New file:** `src/main/java/dev/linhvu/micrometer/step/StepCounter.java`

```java
public class StepCounter extends AbstractMeter implements Counter, StepMeter {

    private final StepDouble value;

    public StepCounter(Meter.Id id, Clock clock, long stepMillis) {
        super(id);
        this.value = new StepDouble(clock, stepMillis);
    }

    @Override public void increment(double amount) { value.getCurrent().add(amount); }
    @Override public double count() { return value.poll(); }
    @Override public void _closingRollover() { value._closingRollover(); }
}
```

### StepTimer

**New file:** `src/main/java/dev/linhvu/micrometer/step/StepTimer.java`

```java
public class StepTimer extends AbstractTimer implements StepMeter {

    private final LongAdder count = new LongAdder();
    private final LongAdder total = new LongAdder();
    private final StepTuple2<Long, Long> countTotal;
    private final TimeWindowMax max;

    public StepTimer(Meter.Id id, Clock clock, TimeUnit baseTimeUnit, long stepMillis,
            DistributionStatisticConfig config) {
        super(id, clock, baseTimeUnit, AbstractTimer.defaultHistogram(clock, config));
        this.countTotal = new StepTuple2<>(clock, stepMillis,
                0L, 0L, count::sumThenReset, total::sumThenReset);
        this.max = new TimeWindowMax(clock);
    }

    @Override
    protected void recordNonNegative(long amount, TimeUnit unit) {
        long nanoAmount = unit.toNanos(amount);
        count.increment();
        total.add(nanoAmount);
        max.record((double) nanoAmount, TimeUnit.NANOSECONDS);
    }

    @Override public long count() { return countTotal.poll1(); }
    @Override public double totalTime(TimeUnit unit) { return nanosToUnit(countTotal.poll2(), unit); }
    @Override public double max(TimeUnit unit) { return max.poll(unit); }
    @Override public void _closingRollover() { countTotal._closingRollover(); }
}
```

### StepDistributionSummary

**New file:** `src/main/java/dev/linhvu/micrometer/step/StepDistributionSummary.java`

Same pattern as StepTimer but with `DoubleAdder` for total (arbitrary amounts are doubles, not nanoseconds):

```java
public class StepDistributionSummary extends AbstractDistributionSummary implements StepMeter {

    private final LongAdder count = new LongAdder();
    private final DoubleAdder total = new DoubleAdder();
    private final StepTuple2<Long, Double> countTotal;
    private final TimeWindowMax max;

    @Override
    protected void recordNonNegative(double amount) {
        count.increment();
        total.add(amount);
        max.record(amount);
    }

    @Override public long count() { return countTotal.poll1(); }
    @Override public double totalAmount() { return countTotal.poll2(); }
    @Override public double max() { return max.poll(); }
    @Override public void _closingRollover() { countTotal._closingRollover(); }
}
```

### StepFunctionCounter

**New file:** `src/main/java/dev/linhvu/micrometer/step/StepFunctionCounter.java`

Adapts a monotonically increasing external counter into per-step deltas:

```java
public class StepFunctionCounter<T> extends AbstractMeter implements FunctionCounter, StepMeter {

    private final WeakReference<T> ref;
    private final ToDoubleFunction<T> f;
    private volatile double last;
    private final StepDouble value;

    @Override
    public double count() {
        T obj = ref.get();
        if (obj != null) {
            double current = f.applyAsDouble(obj);
            double delta = current - last;
            last = current;
            if (delta > 0) { value.getCurrent().add(delta); }
        }
        return value.poll();
    }
}
```

### StepFunctionTimer

**New file:** `src/main/java/dev/linhvu/micrometer/step/StepFunctionTimer.java`

Same delta pattern but for two values (count + total time) using `StepTuple2`:

```java
public class StepFunctionTimer<T> implements FunctionTimer, StepMeter {

    private final WeakReference<T> ref;
    private final ToLongFunction<T> countFunction;
    private final ToDoubleFunction<T> totalTimeFunction;
    private volatile long lastCount;
    private volatile double lastTime;
    private final LongAdder count = new LongAdder();
    private final DoubleAdder total = new DoubleAdder();
    private final StepTuple2<Long, Double> countTotal;

    private void accumulateCountAndTotal() {
        T obj = ref.get();
        if (obj != null) {
            long currentCount = countFunction.applyAsLong(obj);
            double currentTime = totalTimeFunction.applyAsDouble(obj);
            long countDelta = Math.max(currentCount - lastCount, 0);
            double timeDelta = Math.max(currentTime - lastTime, 0);
            lastCount = currentCount;
            lastTime = currentTime;
            if (countDelta > 0) count.add(countDelta);
            if (timeDelta > 0) total.add(timeDelta);
        }
    }

    @Override public double count() { accumulateCountAndTotal(); return countTotal.poll1(); }
    @Override public double totalTime(TimeUnit unit) {
        accumulateCountAndTotal();
        return TimeUtils.convert(countTotal.poll2(), totalTimeFunctionUnit, unit);
    }
}
```

Note: `StepFunctionTimer` does NOT extend `AbstractMeter` — it directly implements `FunctionTimer` and holds its own `Meter.Id`, matching the real Micrometer implementation.

## 16.5 StepMeterRegistry and StepRegistryConfig

**New file:** `src/main/java/dev/linhvu/micrometer/step/StepRegistryConfig.java`

```java
public interface StepRegistryConfig extends PushRegistryConfig {
    StepRegistryConfig DEFAULT = new StepRegistryConfig() {};
}
```

**New file:** `src/main/java/dev/linhvu/micrometer/step/StepMeterRegistry.java`

The registry with two scheduling loops:

```java
public abstract class StepMeterRegistry extends PushMeterRegistry {

    private final StepRegistryConfig config;
    private volatile ScheduledExecutorService meterPollingService;
    private volatile long lastMeterRolloverStartTime = -1;

    // Factory methods — all return step-based variants
    @Override protected Counter newCounter(Meter.Id id) {
        return new StepCounter(id, clock, config.step().toMillis());
    }
    @Override protected Timer newTimer(Meter.Id id, DistributionStatisticConfig config) {
        return new StepTimer(id, clock, getBaseTimeUnit(), this.config.step().toMillis(), config);
    }
    // ... same pattern for all meter types
    // Gauges and LongTaskTimers use non-step implementations (DefaultGauge, DefaultLongTaskTimer)

    @Override
    public void start(ThreadFactory threadFactory) {
        super.start(threadFactory);  // starts the publish loop
        if (config.enabled()) {
            meterPollingService = Executors.newSingleThreadScheduledExecutor(threadFactory);
            long stepMillis = config.step().toMillis();
            long initialDelay = stepMillis - (clock.wallTime() % stepMillis) + 1;  // 1ms into next step
            meterPollingService.scheduleAtFixedRate(
                    this::pollMetersToRollover, initialDelay, stepMillis, TimeUnit.MILLISECONDS);
        }
    }

    void pollMetersToRollover() {
        lastMeterRolloverStartTime = clock.wallTime();
        for (Meter meter : getMeters()) {
            meter.match(
                    Counter::count,               // triggers rollover
                    gauge -> null,                // skip
                    Timer::count,                 // triggers rollover
                    DistributionSummary::count,   // triggers rollover
                    longTaskTimer -> null,         // skip
                    FunctionCounter::count,       // triggers rollover
                    FunctionTimer::count,         // triggers rollover
                    m -> null                     // skip
            );
        }
    }
}
```

---

## 16.6 Try It Yourself

<details><summary>Challenge 1: Observe step vs. cumulative behavior with a counter</summary>

```java
MockClock clock = new MockClock();
StepRegistryConfig config = new StepRegistryConfig() {
    @Override public Duration step() { return Duration.ofSeconds(10); }
};

// Create a step registry
TestStepRegistry registry = new TestStepRegistry(config, clock);
Counter counter = registry.counter("http.requests");

// Increment during step 1
counter.increment(5);
System.out.println("Before step boundary: " + counter.count()); // 0.0 (current step not visible)

clock.add(10, TimeUnit.SECONDS);  // cross step boundary
System.out.println("After step boundary: " + counter.count());  // 5.0 (previous step's value)

// Increment during step 2
counter.increment(3);
clock.add(10, TimeUnit.SECONDS);
System.out.println("Step 2: " + counter.count());               // 3.0 (not 8.0!)
```

</details>

<details><summary>Challenge 2: Demonstrate gap detection — what happens when steps are skipped?</summary>

```java
MockClock clock = new MockClock();
StepRegistryConfig config = new StepRegistryConfig() {
    @Override public Duration step() { return Duration.ofSeconds(10); }
};

TestStepRegistry registry = new TestStepRegistry(config, clock);
Counter counter = registry.counter("events");

counter.increment(100);

// Skip 2 full steps (20 seconds instead of 10)
clock.add(20, TimeUnit.SECONDS);
System.out.println("After gap: " + counter.count());  // 0.0 (stale data discarded)
```

This is correct! If you increment 100 but don't poll for 20 seconds, the system can't know which step the increments belong to. Rather than misattribute data, it reports zero.

</details>

<details><summary>Challenge 3: Closing rollover captures partial-step data</summary>

```java
MockClock clock = new MockClock();
StepRegistryConfig config = new StepRegistryConfig() {
    @Override public Duration step() { return Duration.ofSeconds(60); }
};

TestStepRegistry registry = new TestStepRegistry(config, clock);
Counter counter = registry.counter("events");
counter.increment(42);

// Only 10 seconds have passed — we're mid-step
clock.add(10, TimeUnit.SECONDS);
System.out.println("Mid-step: " + counter.count());  // 0.0

// Close the registry — closing rollover drains the accumulator
registry.close();
// The publish() inside close() will see count=42
```

</details>

---

## 16.7 Tests

### Unit Tests

#### StepValueTest — Core rollover behavior

```java
@Test
void shouldReturnAccumulatedValue_WhenStepBoundaryCrossed() {
    stepDouble.getCurrent().add(5.0);
    stepDouble.getCurrent().add(3.0);
    clock.add(STEP_MILLIS, TimeUnit.MILLISECONDS);
    assertThat(stepDouble.poll()).isEqualTo(8.0);
}

@Test
void shouldReturnNoValue_WhenStepGapExceedsOne() {
    stepDouble.getCurrent().add(5.0);
    clock.add(2 * STEP_MILLIS, TimeUnit.MILLISECONDS);  // skip a step
    assertThat(stepDouble.poll()).isEqualTo(0.0);         // stale data discarded
}

@Test
void shouldDrainCurrentAccumulator_WhenClosingRollover() {
    stepDouble.getCurrent().add(7.0);
    stepDouble._closingRollover();
    assertThat(stepDouble.poll()).isEqualTo(7.0);
}
```

#### StepTuple2Test — Atomic pair rollover

```java
@Test
void shouldRolloverAtomically_WhenBothValuesPresent() {
    count.add(5);
    total.add(250.0);
    clock.add(STEP_MILLIS, TimeUnit.MILLISECONDS);

    Long c = tuple.poll1();
    Double t = tuple.poll2();
    assertThat(c).isEqualTo(5L);
    assertThat(t).isEqualTo(250.0);
    assertThat(t / c).isEqualTo(50.0);  // consistent average
}
```

#### StepCounterTest

```java
@Test
void shouldReturnStepCount_WhenStepBoundaryCrossed() {
    counter.increment();
    counter.increment();
    counter.increment();
    clock.add(STEP_MILLIS, TimeUnit.MILLISECONDS);
    assertThat(counter.count()).isEqualTo(3.0);
}

@Test
void shouldResetBetweenSteps_WhenMultipleStepsCrossed() {
    counter.increment(5.0);
    clock.add(STEP_MILLIS, TimeUnit.MILLISECONDS);
    assertThat(counter.count()).isEqualTo(5.0);

    counter.increment(2.0);
    clock.add(STEP_MILLIS, TimeUnit.MILLISECONDS);
    assertThat(counter.count()).isEqualTo(2.0);  // 2, not 7
}
```

#### StepTimerTest

```java
@Test
void shouldComputeConsistentMean_WhenCountAndTotalRolloverAtomically() {
    timer.record(100, TimeUnit.MILLISECONDS);
    timer.record(200, TimeUnit.MILLISECONDS);
    timer.record(300, TimeUnit.MILLISECONDS);
    clock.add(STEP_MILLIS, TimeUnit.MILLISECONDS);
    assertThat(timer.mean(TimeUnit.MILLISECONDS)).isEqualTo(200.0);
}
```

#### StepMeterRegistryTest

```java
@Test
void shouldCreateStepCounter_WhenCounterRegistered() {
    Counter counter = reg.counter("test.counter");
    assertThat(counter).isInstanceOf(StepCounter.class);
}

@Test
void shouldReportPerStepCount_WhenStepBoundaryCrossed() {
    Counter counter = reg.counter("test.counter");
    counter.increment(5);
    clock.add(10, TimeUnit.SECONDS);
    assertThat(counter.count()).isEqualTo(5.0);

    counter.increment(2);
    clock.add(10, TimeUnit.SECONDS);
    assertThat(counter.count()).isEqualTo(2.0);
}
```

### Integration Tests

```java
@Test
void shouldPublishPerStepValues_WhenFullLifecycleCompletes() {
    Counter counter = reg.counter("http.requests");
    Timer timer = reg.timer("http.latency");

    counter.increment(10);
    timer.record(100, TimeUnit.MILLISECONDS);
    timer.record(200, TimeUnit.MILLISECONDS);

    clock.add(STEP_SECONDS, TimeUnit.SECONDS);

    assertThat(counter.count()).isEqualTo(10.0);
    assertThat(timer.count()).isEqualTo(2);
    assertThat(timer.totalTime(TimeUnit.MILLISECONDS)).isEqualTo(300.0);
}

@Test
void shouldReturnZero_WhenStepIsSkipped() {
    Counter counter = reg.counter("requests");
    counter.increment(5);
    clock.add(2 * STEP_SECONDS, TimeUnit.SECONDS);
    assertThat(counter.count()).isEqualTo(0.0);  // gap detection
}
```

**Run:** `./gradlew test` — expected: all tests pass (including all prior features' tests)

---

## 16.8 Why This Works

> ★ **Insight** -------------------------------------------
> **Why `sumThenReset()` instead of `sum()` + manual reset?** `DoubleAdder.sumThenReset()` is an atomic operation that reads the total and resets to zero in one step. If we did `sum()` followed by `reset()`, increments that arrive between the two calls would be lost. This matters because `increment()` calls can arrive from any thread at any time — the rollover must not lose or double-count any increment.
>
> **Trade-off:** `sumThenReset()` requires the caller to use the returned value immediately. If the caller discards it (e.g., a concurrent `poll()` that lost the CAS race), those values are gone. That's why the CAS on `lastInitPos` is essential — it ensures exactly one thread drains the accumulator per step.
> -----------------------------------------------------------

> ★ **Insight** -------------------------------------------
> **Why two scheduling loops instead of one?** The step registry runs a **polling loop** (fires 1ms into each step) and a **publishing loop** (fires at a random offset 2ms+ into the step). This separation solves a subtle timing problem: if publishing fires *before* the step boundary, `poll()` would return the previous-previous step's data (the current step hasn't rolled over yet). By polling first and publishing second, the publish always sees freshly rolled-over data. This is the same architecture the real Micrometer uses — see `StepMeterRegistry.java:96-115`.
>
> **Trade-off:** Two executors consume two threads. In production with many registry instances, this adds up. The real Micrometer mitigates this by sharing thread pools and using daemon threads.
> -----------------------------------------------------------

> ★ **Insight** -------------------------------------------
> **Why do `StepFunctionCounter` and `StepFunctionTimer` compute deltas?** Monotonically increasing external counters (like JMX's `TotalCompilationTime`) report cumulative totals. A step-based backend needs per-interval deltas. The function meters track the last raw value and compute `current - last` to convert cumulative into per-step. This is the **adapter pattern** — adapting one interface (monotonic cumulative) to another (periodic delta). The `Math.max(..., 0)` guard handles the edge case where an external counter wraps or is reset.
> -----------------------------------------------------------

## 16.9 What We Enhanced

| Aspect | Before (ch15) | Current (ch16) | Real Framework |
|--------|---------------|----------------|----------------|
| Meter semantics | Cumulative only — counters grow forever, totals accumulate | Step-based — values reset at step boundaries, report per-interval deltas | Same — `StepCounter.java`, `StepTimer.java` |
| Rollover primitive | Not available | `StepValue<V>` with CAS-based drain and gap detection | Same — `StepValue.java:30-96` |
| Concurrent accumulation | `DoubleAdder.sum()` (cumulative read) | `DoubleAdder.sumThenReset()` (read-and-reset) | Same — `StepDouble.java:36` |
| Paired value rollover | Not needed (cumulative can read independently) | `StepTuple2<T1,T2>` for atomic count+total rollover | Same — `StepTuple2.java:27-100` |
| Function meter adaptation | `CumulativeFunctionCounter` reports lifetime total | `StepFunctionCounter` computes per-step deltas from monotonic source | Same — `StepFunctionCounter.java:44-68` |
| Meter polling | Not available | Second `ScheduledExecutorService` polls meters at step boundaries | Same — `StepMeterRegistry.java:96-115` |
| Shutdown protocol | `PushMeterRegistry.close()`: stop → final publish → wait | Extended: stop polling → publish unpublished → closing rollover → `super.close()` | Same — `StepMeterRegistry.java:128-159` |
| Histogram expiry | Default 2 minutes | Overridden to match step duration | Same — `StepMeterRegistry.java:162-165` |
| `publishSafelyOrSkipIfInProgress` visibility | Package-private in `PushMeterRegistry` | Made `protected` for subclass access | Already `protected` in real Micrometer |

## 16.10 Connection to Real Framework

| Simplified Code | Real Framework Code | File:Line | Key Difference |
|----------------|--------------------|-----------| ---------------|
| `StepValue<V>` | `StepValue<V>` | `StepValue.java:30-96` | Real has additional `lastInitPos` initialization via `wallTime / stepMillis`. Our version matches. |
| `StepDouble` | `StepDouble` | `StepDouble.java:25-48` | Identical structure — wraps `DoubleAdder` with `sumThenReset` |
| `StepLong` | `StepLong` | `StepLong.java:25-48` | Identical structure — wraps `LongAdder` with `sumThenReset` |
| `StepTuple2<T1,T2>` | `StepTuple2<T1,T2>` | `StepTuple2.java:27-100` | Real is a standalone class (not extending StepValue) — same approach in our version |
| `StepCounter` | `StepCounter` | `StepCounter.java:27-56` | Identical — wraps `StepDouble`, implements `Counter` and `StepMeter` |
| `StepTimer` | `StepTimer` | `StepTimer.java:29-86` | Real uses `StepTuple2<Long, Long>` for count+total — same approach |
| `StepDistributionSummary` | `StepDistributionSummary` | `StepDistributionSummary.java:28-78` | Real uses `StepTuple2<Long, Double>` — same approach |
| `StepFunctionCounter<T>` | `StepFunctionCounter<T>` | `StepFunctionCounter.java:28-70` | Real reads delta in `count()` then polls `StepDouble` — same approach |
| `StepFunctionTimer<T>` | `StepFunctionTimer<T>` | `StepFunctionTimer.java:29-118` | Real adds a 1ms debounce (`lastUpdateTime`) to avoid redundant accumulation — we omit this simplification |
| `StepMeterRegistry.start()` | `StepMeterRegistry.start()` | `StepMeterRegistry.java:96-115` | Real aligns polling to 1ms into the next step boundary — same approach |
| `StepMeterRegistry.close()` | `StepMeterRegistry.close()` | `StepMeterRegistry.java:128-159` | Real has `shouldPublishDataForLastStep()` check using `lastMeterRolloverStartTime` — same approach |
| `StepMeterRegistry.pollMetersToRollover()` | `StepMeterRegistry.pollMetersToRollover()` | `StepMeterRegistry.java:117-126` | Real iterates meters and calls `count()` to trigger rollover — same approach |
| `StepMeter._closingRollover()` | `StepMeter._closingRollover()` | `StepMeter.java:25-27` | Real uses package-private interface — same approach |

## 16.11 Complete Code

### Production Code

#### File: `src/main/java/dev/linhvu/micrometer/push/PushMeterRegistry.java` [MODIFIED]

Change on line 93: `void publishSafelyOrSkipIfInProgress()` → `protected void publishSafelyOrSkipIfInProgress()`

```java
package dev.linhvu.micrometer.push;

import dev.linhvu.micrometer.Clock;
import dev.linhvu.micrometer.MeterRegistry;

import java.util.concurrent.Executors;
import java.util.concurrent.ScheduledExecutorService;
import java.util.concurrent.Semaphore;
import java.util.concurrent.ThreadFactory;
import java.util.concurrent.TimeUnit;

/**
 * Base class for registries that periodically push/export metrics to a remote
 * backend on a schedule.
 * <p>
 * The publishing lifecycle:
 * <ol>
 *   <li>{@link #start(ThreadFactory)} — creates a {@link ScheduledExecutorService}
 *       that calls {@link #publish()} at the configured {@link PushRegistryConfig#step()}</li>
 *   <li>{@link #publishSafelyOrSkipIfInProgress()} — wraps {@link #publish()} with
 *       a {@link Semaphore} guard (prevents overlapping publishes) and exception handling
 *       (prevents the scheduler from dying)</li>
 *   <li>{@link #close()} — stops the scheduler, does one final publish, waits for
 *       any in-progress publish to complete, then marks the registry as closed</li>
 * </ol>
 * <p>
 * Subclasses implement:
 * <ul>
 *   <li>{@link #publish()} — serialize and send all meters to the backend</li>
 *   <li>{@code newCounter()}, {@code newGauge()}, etc. — create backend-specific meter implementations</li>
 *   <li>{@link #getBaseTimeUnit()} — return the backend's expected time unit</li>
 * </ul>
 * <p>
 * <b>Simplification:</b> The real Micrometer uses an internal logging framework
 * ({@code InternalLogger}) for warnings. We use {@code System.err} for simplicity.
 */
public abstract class PushMeterRegistry extends MeterRegistry {

    /**
     * Schedule publishing in the beginning 80% of the step to avoid spill-over
     * into the next step. The remaining 20% provides buffer for clock skew and
     * publish duration.
     */
    private static final double PERCENT_RANGE_OF_RANDOM_PUBLISHING_OFFSET = 0.8;

    private final PushRegistryConfig config;

    /**
     * Binary semaphore that prevents overlapping publishes. {@code tryAcquire()}
     * is used during scheduled publishing (non-blocking skip if busy), while
     * {@code acquire()} is used during {@code close()} (blocking wait for
     * in-progress publish to complete).
     */
    private final Semaphore publishingSemaphore = new Semaphore(1);

    private volatile ScheduledExecutorService scheduledExecutorService;

    protected PushMeterRegistry(PushRegistryConfig config, Clock clock) {
        super(clock);
        this.config = config;
    }

    // -----------------------------------------------------------------------
    // Abstract publish method — subclasses implement the export logic
    // -----------------------------------------------------------------------

    /**
     * Serializes and sends all registered meters to the backend. Called periodically
     * by the scheduler and once on graceful shutdown.
     * <p>
     * Implementations should iterate {@link #getMeters()} and format each meter
     * for the target system. Use {@link PushRegistryConfig#batchSize()} to chunk
     * large meter lists into multiple requests.
     */
    protected abstract void publish();

    // -----------------------------------------------------------------------
    // Safe publishing with semaphore guard
    // -----------------------------------------------------------------------

    /**
     * Calls {@link #publish()} with two safety guards:
     * <ol>
     *   <li><b>Semaphore guard:</b> If another publish is already in progress,
     *       this call is skipped (non-blocking). This prevents slow publishes
     *       from causing a backlog of waiting threads.</li>
     *   <li><b>Exception handling:</b> All exceptions from {@code publish()} are
     *       caught and logged. This is critical because
     *       {@link ScheduledExecutorService#scheduleAtFixedRate} silently stops
     *       future executions if a task throws an uncaught exception.</li>
     * </ol>
     */
    protected void publishSafelyOrSkipIfInProgress() {
        if (publishingSemaphore.tryAcquire()) {
            try {
                publish();
            }
            catch (Throwable e) {
                System.err.println("Unexpected exception while publishing metrics for "
                        + getClass().getSimpleName() + ": " + e.getMessage());
            }
            finally {
                publishingSemaphore.release();
            }
        }
    }

    /**
     * Returns whether a publish is currently in progress.
     */
    protected boolean isPublishing() {
        return publishingSemaphore.availablePermits() == 0;
    }

    // -----------------------------------------------------------------------
    // Scheduler lifecycle: start / stop / close
    // -----------------------------------------------------------------------

    /**
     * Starts the scheduled publisher using the default thread factory.
     */
    public void start() {
        start(Executors.defaultThreadFactory());
    }

    /**
     * Starts the scheduled publisher. If publishing is already running, it is
     * stopped first. If {@link PushRegistryConfig#enabled()} is {@code false},
     * this method does nothing.
     * <p>
     * The scheduler calls {@link #publishSafelyOrSkipIfInProgress()} at a fixed
     * rate of {@link PushRegistryConfig#step()}. A randomized initial delay
     * prevents thundering-herd effects when many instances start simultaneously.
     *
     * @param threadFactory Factory for creating the scheduler thread.
     */
    public void start(ThreadFactory threadFactory) {
        if (scheduledExecutorService != null) {
            stop();
        }

        if (config.enabled()) {
            scheduledExecutorService = Executors.newSingleThreadScheduledExecutor(threadFactory);
            long stepMillis = config.step().toMillis();
            long initialDelayMillis = calculateInitialDelay();
            scheduledExecutorService.scheduleAtFixedRate(
                    this::publishSafelyOrSkipIfInProgress,
                    initialDelayMillis, stepMillis, TimeUnit.MILLISECONDS);
        }
    }

    /**
     * Stops the scheduled publisher. Does not trigger a final publish — use
     * {@link #close()} for graceful shutdown.
     */
    public void stop() {
        if (scheduledExecutorService != null) {
            scheduledExecutorService.shutdown();
            scheduledExecutorService = null;
        }
    }

    /**
     * Graceful shutdown:
     * <ol>
     *   <li>Stop the scheduler (no more triggered publishes)</li>
     *   <li>If enabled, do one final publish (or skip if one is in progress)</li>
     *   <li>Wait for any in-progress publish to complete</li>
     *   <li>Mark the registry as closed (no-op meters for new registrations)</li>
     * </ol>
     */
    @Override
    public void close() {
        stop();
        if (config.enabled() && !isClosed()) {
            // Do a final publish on close, or skip if one is in progress
            publishSafelyOrSkipIfInProgress();
            waitForInProgressScheduledPublish();
        }
        super.close();
    }

    /**
     * Blocks until any in-progress publish completes. Used during {@link #close()}
     * to ensure data is not lost.
     */
    protected void waitForInProgressScheduledPublish() {
        try {
            // Block by acquiring the semaphore (will wait if publish is in progress),
            // then immediately release it
            publishingSemaphore.acquire();
            publishingSemaphore.release();
        }
        catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
    }

    // -----------------------------------------------------------------------
    // Initial delay calculation
    // -----------------------------------------------------------------------

    /**
     * Calculates a randomized initial delay for the first publish.
     * <p>
     * The delay places the first publish within the first 80% of the next step
     * boundary, at least 2ms into the step. This serves two purposes:
     * <ol>
     *   <li><b>Thundering-herd prevention:</b> When many instances start at the
     *       same time, each gets a different random delay, spreading the publish
     *       load across the step interval.</li>
     *   <li><b>Step alignment:</b> The 2ms minimum offset ensures the publish runs
     *       after the step boundary has been crossed, giving step-based meters
     *       time to roll over their values.</li>
     * </ol>
     *
     * @return The initial delay in milliseconds.
     */
    long calculateInitialDelay() {
        long stepMillis = config.step().toMillis();
        // Random offset within [0, 80% of step - 2)
        long randomOffset = Math.max(0,
                (long) (stepMillis * Math.random() * PERCENT_RANGE_OF_RANDOM_PUBLISHING_OFFSET) - 2);
        // Offset to the start of the next step boundary
        long offsetToStartOfNextStep = stepMillis - (clock.wallTime() % stepMillis);
        // At least 2ms into the step
        return offsetToStartOfNextStep + 2 + randomOffset;
    }

}
```

#### File: `src/main/java/dev/linhvu/micrometer/step/StepMeter.java` [NEW]

```java
package dev.linhvu.micrometer.step;

/**
 * Marker interface for meters that use step-based semantics — values accumulate
 * during a step interval and are drained (rolled over) at step boundaries.
 * <p>
 * The single method {@link #_closingRollover()} is called during
 * {@link StepMeterRegistry#close()} to drain whatever is in the current
 * accumulator as a final partial-step value, ensuring no data is lost on shutdown.
 * <p>
 * The leading underscore in the method name signals that this is an internal
 * lifecycle method, not part of the public meter API.
 */
interface StepMeter {

    /**
     * Performs a final drain of the current accumulator into the "previous" value,
     * and prevents any future rollovers by setting the rollover position to
     * {@code Long.MAX_VALUE}.
     * <p>
     * Called once during registry shutdown — after this call, {@code poll()} will
     * return the drained value from the in-progress (partial) step.
     */
    void _closingRollover();

}
```

#### File: `src/main/java/dev/linhvu/micrometer/step/StepValue.java` [NEW]

```java
package dev.linhvu.micrometer.step;

import dev.linhvu.micrometer.Clock;

import java.util.concurrent.atomic.AtomicLong;
import java.util.function.Supplier;

/**
 * The core primitive that accumulates values in time-bounded steps, where
 * {@link #poll()} returns the <b>previous completed step's value</b> — not
 * the running total.
 * <p>
 * <b>How step semantics work:</b>
 * <ol>
 *   <li>Values accumulate into a "current" accumulator during a step interval.</li>
 *   <li>When {@code poll()} is called and the wall clock has crossed a step boundary,
 *       {@link #rollCount(long)} fires: it drains the accumulator (via
 *       {@link #valueSupplier()}'s {@code sumThenReset}) into {@code previous},
 *       and the accumulator restarts at zero.</li>
 *   <li>Subsequent calls to {@code poll()} within the same step just return
 *       {@code previous} without rolling over again.</li>
 *   <li>If a step boundary is crossed without any poll (a "gap"), the next poll
 *       detects it and sets {@code previous = noValue()} — effectively reporting
 *       zero for the missed interval and discarding stale data.</li>
 * </ol>
 * <p>
 * Thread safety: {@link #lastInitPos} uses CAS to ensure that exactly one thread
 * performs the drain when multiple threads call {@code poll()} simultaneously.
 *
 * @param <V> The value type (e.g., {@code Double} for counters, {@code Long} for counts).
 */
abstract class StepValue<V> {

    private final Clock clock;

    private final long stepMillis;

    /**
     * The step number (wallTime / stepMillis) at which the last rollover happened.
     * CAS-updated to ensure only one thread performs the drain.
     */
    private final AtomicLong lastInitPos;

    /**
     * The value from the last completed step interval. This is what {@link #poll()}
     * returns. Volatile for visibility across threads.
     */
    private volatile V previous;

    StepValue(Clock clock, long stepMillis) {
        this.clock = clock;
        this.stepMillis = stepMillis;
        this.previous = noValue();
        this.lastInitPos = new AtomicLong(clock.wallTime() / stepMillis);
    }

    /**
     * Returns a supplier that reads-and-resets the current accumulator.
     * For example, {@code DoubleAdder::sumThenReset}.
     */
    abstract Supplier<V> valueSupplier();

    /**
     * The "zero" or default value returned when no activity occurred in the previous step.
     */
    abstract V noValue();

    /**
     * The core rollover logic. If the wall clock has crossed a step boundary since
     * the last rollover, drains the accumulator into {@code previous}.
     * <p>
     * If the last rollover was the immediately preceding step (no gap), the drained
     * value becomes {@code previous}. If there was a gap of more than one step,
     * {@code previous} is set to {@code noValue()} — stale data is discarded.
     */
    private void rollCount(long now) {
        long stepTime = now / stepMillis;
        long lastInit = lastInitPos.get();
        if (lastInit < stepTime && lastInitPos.compareAndSet(lastInit, stepTime)) {
            // We won the CAS — drain the accumulator
            V value = valueSupplier().get();
            // Only use the drained value if it's from the immediately preceding step
            if (lastInit == stepTime - 1) {
                previous = value;
            }
            else {
                // Gap of more than one step — discard stale data
                previous = noValue();
            }
        }
    }

    /**
     * Returns the value from the last completed step, triggering a rollover if
     * a step boundary has been crossed.
     */
    V poll() {
        rollCount(clock.wallTime());
        return previous;
    }

    /**
     * Final drain for shutdown. Sets {@code lastInitPos} to {@code Long.MAX_VALUE}
     * to prevent any future rollovers, then drains whatever is in the accumulator
     * into {@code previous}.
     */
    void _closingRollover() {
        lastInitPos.set(Long.MAX_VALUE);
        previous = valueSupplier().get();
    }

}
```

#### File: `src/main/java/dev/linhvu/micrometer/step/StepDouble.java` [NEW]

```java
package dev.linhvu.micrometer.step;

import dev.linhvu.micrometer.Clock;

import java.util.concurrent.atomic.DoubleAdder;
import java.util.function.Supplier;

/**
 * A step-aware double accumulator. Uses {@link DoubleAdder} for high-throughput
 * concurrent writes with minimal contention (internally striped).
 * <p>
 * During a step interval, values are added via {@link #getCurrent()}.{@code add()}.
 * On rollover, {@link DoubleAdder#sumThenReset()} atomically reads the total
 * and resets to zero.
 * <p>
 * Used by {@link StepCounter} and {@link StepFunctionCounter}.
 */
class StepDouble extends StepValue<Double> {

    private final DoubleAdder current = new DoubleAdder();

    StepDouble(Clock clock, long stepMillis) {
        super(clock, stepMillis);
    }

    @Override
    Supplier<Double> valueSupplier() {
        return current::sumThenReset;
    }

    @Override
    Double noValue() {
        return 0.0;
    }

    /**
     * Returns the underlying accumulator for callers (like {@link StepCounter})
     * to add values to.
     */
    DoubleAdder getCurrent() {
        return current;
    }

}
```

#### File: `src/main/java/dev/linhvu/micrometer/step/StepLong.java` [NEW]

```java
package dev.linhvu.micrometer.step;

import dev.linhvu.micrometer.Clock;

import java.util.concurrent.atomic.LongAdder;
import java.util.function.Supplier;

/**
 * A step-aware long accumulator. Exact parallel of {@link StepDouble} but for
 * {@code Long} values using {@link LongAdder}.
 * <p>
 * Used internally by step meters that need to accumulate integer counts
 * (e.g., the count field in {@link StepTimer}).
 */
class StepLong extends StepValue<Long> {

    private final LongAdder current = new LongAdder();

    StepLong(Clock clock, long stepMillis) {
        super(clock, stepMillis);
    }

    @Override
    Supplier<Long> valueSupplier() {
        return current::sumThenReset;
    }

    @Override
    Long noValue() {
        return 0L;
    }

    /**
     * Returns the underlying accumulator for callers to add values to.
     */
    LongAdder getCurrent() {
        return current;
    }

}
```

#### File: `src/main/java/dev/linhvu/micrometer/step/StepTuple2.java` [NEW]

```java
package dev.linhvu.micrometer.step;

import dev.linhvu.micrometer.Clock;

import java.util.concurrent.atomic.AtomicLong;
import java.util.function.Supplier;

/**
 * A step-aware pair of values that roll over atomically together. This ensures
 * both values always come from the same step interval.
 * <p>
 * This is critical for {@link StepTimer} (count + total time) and
 * {@link StepDistributionSummary} (count + total amount) — you can't have
 * count from step N and total from step N-1 because that would produce
 * incorrect averages.
 * <p>
 * Implements the same CAS-based rollover algorithm as {@link StepValue}, but
 * for two values simultaneously. This is a separate class rather than a
 * subclass of {@code StepValue} because the dual-drain cannot be expressed
 * through a single {@code valueSupplier()}.
 *
 * @param <T1> Type of the first value (e.g., {@code Long} for count).
 * @param <T2> Type of the second value (e.g., {@code Long} or {@code Double} for total).
 */
class StepTuple2<T1, T2> {

    private final Clock clock;

    private final long stepMillis;

    private final AtomicLong lastInitPos;

    private final T1 t1NoValue;

    private final T2 t2NoValue;

    private final Supplier<T1> t1Supplier;

    private final Supplier<T2> t2Supplier;

    private volatile T1 t1Previous;

    private volatile T2 t2Previous;

    StepTuple2(Clock clock, long stepMillis,
            T1 t1NoValue, T2 t2NoValue,
            Supplier<T1> t1Supplier, Supplier<T2> t2Supplier) {
        this.clock = clock;
        this.stepMillis = stepMillis;
        this.t1NoValue = t1NoValue;
        this.t2NoValue = t2NoValue;
        this.t1Supplier = t1Supplier;
        this.t2Supplier = t2Supplier;
        this.t1Previous = t1NoValue;
        this.t2Previous = t2NoValue;
        this.lastInitPos = new AtomicLong(clock.wallTime() / stepMillis);
    }

    /**
     * Same CAS-based rollover as {@link StepValue#poll()}, but drains both
     * suppliers in the same atomic operation.
     */
    private void rollCount(long now) {
        long stepTime = now / stepMillis;
        long lastInit = lastInitPos.get();
        if (lastInit < stepTime && lastInitPos.compareAndSet(lastInit, stepTime)) {
            T1 v1 = t1Supplier.get();
            T2 v2 = t2Supplier.get();
            if (lastInit == stepTime - 1) {
                t1Previous = v1;
                t2Previous = v2;
            }
            else {
                t1Previous = t1NoValue;
                t2Previous = t2NoValue;
            }
        }
    }

    /**
     * Returns the first value from the last completed step.
     */
    T1 poll1() {
        rollCount(clock.wallTime());
        return t1Previous;
    }

    /**
     * Returns the second value from the last completed step.
     */
    T2 poll2() {
        rollCount(clock.wallTime());
        return t2Previous;
    }

    /**
     * Final drain for shutdown — same pattern as {@link StepValue#_closingRollover()}.
     */
    void _closingRollover() {
        lastInitPos.set(Long.MAX_VALUE);
        t1Previous = t1Supplier.get();
        t2Previous = t2Supplier.get();
    }

}
```

#### File: `src/main/java/dev/linhvu/micrometer/step/StepCounter.java` [NEW]

```java
package dev.linhvu.micrometer.step;

import dev.linhvu.micrometer.AbstractMeter;
import dev.linhvu.micrometer.Counter;
import dev.linhvu.micrometer.Meter;

/**
 * A counter with step-based semantics: {@link #count()} returns the count from
 * the <b>last completed step interval</b>, not the cumulative lifetime count.
 * <p>
 * Values accumulate in a {@link StepDouble} during the step. When a step boundary
 * is crossed, the accumulated count becomes the "previous" value and the accumulator
 * resets to zero.
 * <p>
 * This is the fundamental behavioral difference from {@link dev.linhvu.micrometer.cumulative.CumulativeCounter}:
 * <ul>
 *   <li><b>Cumulative:</b> count() → 1, 5, 12, 20 (grows forever)</li>
 *   <li><b>Step:</b> count() → 0, 4, 7, 8 (resets each step — reports per-interval delta)</li>
 * </ul>
 */
public class StepCounter extends AbstractMeter implements Counter, StepMeter {

    private final StepDouble value;

    public StepCounter(Meter.Id id, dev.linhvu.micrometer.Clock clock, long stepMillis) {
        super(id);
        this.value = new StepDouble(clock, stepMillis);
    }

    @Override
    public void increment(double amount) {
        value.getCurrent().add(amount);
    }

    /**
     * Returns the count from the last completed step interval, triggering a
     * rollover if a step boundary has been crossed.
     */
    @Override
    public double count() {
        return value.poll();
    }

    @Override
    public void _closingRollover() {
        value._closingRollover();
    }

}
```

#### File: `src/main/java/dev/linhvu/micrometer/step/StepTimer.java` [NEW]

```java
package dev.linhvu.micrometer.step;

import dev.linhvu.micrometer.AbstractTimer;
import dev.linhvu.micrometer.Clock;
import dev.linhvu.micrometer.Meter;
import dev.linhvu.micrometer.distribution.DistributionStatisticConfig;
import dev.linhvu.micrometer.distribution.TimeWindowMax;

import java.util.concurrent.TimeUnit;
import java.util.concurrent.atomic.LongAdder;

/**
 * A timer with step-based semantics: {@link #count()} and {@link #totalTime(TimeUnit)}
 * return values from the <b>last completed step interval</b>.
 * <p>
 * Uses two raw {@link LongAdder} accumulators (count and total nanoseconds) wrapped
 * in a {@link StepTuple2} to ensure they roll over atomically together. The max
 * uses {@link TimeWindowMax} with its own independent time-windowing.
 * <p>
 * The count and total are accumulated as longs (nanoseconds) for precision,
 * then converted to the requested time unit on read.
 */
public class StepTimer extends AbstractTimer implements StepMeter {

    private final LongAdder count = new LongAdder();

    private final LongAdder total = new LongAdder();

    private final StepTuple2<Long, Long> countTotal;

    private final TimeWindowMax max;

    public StepTimer(Meter.Id id, Clock clock, TimeUnit baseTimeUnit, long stepMillis,
            DistributionStatisticConfig distributionStatisticConfig) {
        super(id, clock, baseTimeUnit, AbstractTimer.defaultHistogram(clock, distributionStatisticConfig));
        this.countTotal = new StepTuple2<>(clock, stepMillis,
                0L, 0L,
                count::sumThenReset, total::sumThenReset);
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
        return countTotal.poll1();
    }

    @Override
    public double totalTime(TimeUnit unit) {
        return nanosToUnit(countTotal.poll2(), unit);
    }

    @Override
    public double max(TimeUnit unit) {
        return max.poll(unit);
    }

    @Override
    public void _closingRollover() {
        countTotal._closingRollover();
    }

    private static double nanosToUnit(double nanos, TimeUnit unit) {
        return nanos / unit.toNanos(1);
    }

}
```

#### File: `src/main/java/dev/linhvu/micrometer/step/StepDistributionSummary.java` [NEW]

```java
package dev.linhvu.micrometer.step;

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

/**
 * A distribution summary with step-based semantics: {@link #count()} and
 * {@link #totalAmount()} return values from the <b>last completed step interval</b>.
 * <p>
 * Nearly identical structure to {@link StepTimer} but without time-unit conversion.
 * Uses {@link DoubleAdder} for the total (arbitrary amounts can be fractional)
 * rather than {@link LongAdder} (nanoseconds are always longs).
 */
public class StepDistributionSummary extends AbstractDistributionSummary implements StepMeter {

    private final LongAdder count = new LongAdder();

    private final DoubleAdder total = new DoubleAdder();

    private final StepTuple2<Long, Double> countTotal;

    private final TimeWindowMax max;

    public StepDistributionSummary(Meter.Id id, Clock clock, double scale, long stepMillis,
            DistributionStatisticConfig distributionStatisticConfig) {
        super(id, scale, AbstractDistributionSummary.defaultHistogram(clock, distributionStatisticConfig));
        this.countTotal = new StepTuple2<>(clock, stepMillis,
                0L, 0.0,
                count::sumThenReset, total::sumThenReset);
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
        return countTotal.poll1();
    }

    @Override
    public double totalAmount() {
        return countTotal.poll2();
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

    @Override
    public void _closingRollover() {
        countTotal._closingRollover();
    }

}
```

#### File: `src/main/java/dev/linhvu/micrometer/step/StepFunctionCounter.java` [NEW]

```java
package dev.linhvu.micrometer.step;

import dev.linhvu.micrometer.AbstractMeter;
import dev.linhvu.micrometer.Clock;
import dev.linhvu.micrometer.FunctionCounter;
import dev.linhvu.micrometer.Meter;

import java.lang.ref.WeakReference;
import java.util.function.ToDoubleFunction;

/**
 * A function counter with step-based semantics. Adapts a monotonically
 * increasing external counter into a per-step delta.
 * <p>
 * On each {@link #count()} call:
 * <ol>
 *   <li>Reads the current monotonic value from the function</li>
 *   <li>Computes the delta from the last read</li>
 *   <li>Accumulates the delta into a {@link StepDouble}</li>
 *   <li>Returns the {@code StepDouble}'s poll() — the delta for the last completed step</li>
 * </ol>
 * <p>
 * The observed object is held via a {@link WeakReference} so it can be
 * garbage collected. If the object is GC'd, no more deltas are computed.
 */
public class StepFunctionCounter<T> extends AbstractMeter implements FunctionCounter, StepMeter {

    private final WeakReference<T> ref;

    private final ToDoubleFunction<T> f;

    private volatile double last;

    private final StepDouble value;

    public StepFunctionCounter(Meter.Id id, Clock clock, long stepMillis,
            T obj, ToDoubleFunction<T> f) {
        super(id);
        this.ref = new WeakReference<>(obj);
        this.f = f;
        this.value = new StepDouble(clock, stepMillis);
    }

    @Override
    public double count() {
        T obj = ref.get();
        if (obj != null) {
            double current = f.applyAsDouble(obj);
            double delta = current - last;
            last = current;
            if (delta > 0) {
                value.getCurrent().add(delta);
            }
        }
        return value.poll();
    }

    @Override
    public void _closingRollover() {
        count(); // capture any final delta
        value._closingRollover();
    }

}
```

#### File: `src/main/java/dev/linhvu/micrometer/step/StepFunctionTimer.java` [NEW]

```java
package dev.linhvu.micrometer.step;

import dev.linhvu.micrometer.Clock;
import dev.linhvu.micrometer.FunctionTimer;
import dev.linhvu.micrometer.Meter;
import dev.linhvu.micrometer.TimeUtils;

import java.lang.ref.WeakReference;
import java.util.concurrent.TimeUnit;
import java.util.concurrent.atomic.DoubleAdder;
import java.util.concurrent.atomic.LongAdder;
import java.util.function.ToDoubleFunction;
import java.util.function.ToLongFunction;

/**
 * A function timer with step-based semantics. Adapts monotonically increasing
 * count and total-time functions into per-step deltas.
 * <p>
 * Like {@link StepFunctionCounter}, but for two values (count + total time)
 * that must roll over atomically via {@link StepTuple2}.
 * <p>
 * <b>Note:</b> This class does NOT extend {@code AbstractMeter} — it directly
 * implements {@link FunctionTimer} and holds its own {@link Meter.Id}. This
 * matches the real Micrometer implementation.
 */
public class StepFunctionTimer<T> implements FunctionTimer, StepMeter {

    private final Meter.Id id;

    private final WeakReference<T> ref;

    private final ToLongFunction<T> countFunction;

    private final ToDoubleFunction<T> totalTimeFunction;

    private final TimeUnit totalTimeFunctionUnit;

    private final TimeUnit baseTimeUnit;

    private volatile long lastCount;

    private volatile double lastTime;

    private final LongAdder count = new LongAdder();

    private final DoubleAdder total = new DoubleAdder();

    private final StepTuple2<Long, Double> countTotal;

    public StepFunctionTimer(Meter.Id id, Clock clock, long stepMillis,
            T obj, ToLongFunction<T> countFunction,
            ToDoubleFunction<T> totalTimeFunction, TimeUnit totalTimeFunctionUnit,
            TimeUnit baseTimeUnit) {
        this.id = id;
        this.ref = new WeakReference<>(obj);
        this.countFunction = countFunction;
        this.totalTimeFunction = totalTimeFunction;
        this.totalTimeFunctionUnit = totalTimeFunctionUnit;
        this.baseTimeUnit = baseTimeUnit;
        this.countTotal = new StepTuple2<>(clock, stepMillis,
                0L, 0.0,
                count::sumThenReset, total::sumThenReset);
    }

    /**
     * Reads the external functions, computes deltas, and accumulates them
     * into the step-aware accumulators.
     */
    private void accumulateCountAndTotal() {
        T obj = ref.get();
        if (obj != null) {
            long currentCount = countFunction.applyAsLong(obj);
            double currentTime = totalTimeFunction.applyAsDouble(obj);
            long countDelta = Math.max(currentCount - lastCount, 0);
            double timeDelta = Math.max(currentTime - lastTime, 0);
            lastCount = currentCount;
            lastTime = currentTime;
            if (countDelta > 0) {
                count.add(countDelta);
            }
            if (timeDelta > 0) {
                total.add(timeDelta);
            }
        }
    }

    @Override
    public double count() {
        accumulateCountAndTotal();
        return countTotal.poll1();
    }

    @Override
    public double totalTime(TimeUnit unit) {
        accumulateCountAndTotal();
        double totalInSourceUnit = countTotal.poll2();
        return TimeUtils.convert(totalInSourceUnit, totalTimeFunctionUnit, unit);
    }

    @Override
    public TimeUnit baseTimeUnit() {
        return baseTimeUnit;
    }

    @Override
    public Id getId() {
        return id;
    }

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof Meter other)) return false;
        return id.equals(other.getId());
    }

    @Override
    public int hashCode() {
        return id.hashCode();
    }

    @Override
    public String toString() {
        return getClass().getSimpleName() + "{" + id + '}';
    }

    @Override
    public void _closingRollover() {
        accumulateCountAndTotal();
        countTotal._closingRollover();
    }

}
```

#### File: `src/main/java/dev/linhvu/micrometer/step/StepRegistryConfig.java` [NEW]

```java
package dev.linhvu.micrometer.step;

import dev.linhvu.micrometer.push.PushRegistryConfig;

/**
 * Configuration for step-based meter registries.
 * <p>
 * Extends {@link PushRegistryConfig} — all push configuration (step interval,
 * enabled, batch size) applies. A step registry uses the same {@code step()}
 * interval for both meter rollover and publishing.
 * <p>
 * <b>Simplification:</b> The real Micrometer's step config is identical to
 * push config (no additional properties). This interface exists for type
 * clarity and to parallel the real codebase structure.
 */
public interface StepRegistryConfig extends PushRegistryConfig {

    /**
     * Default configuration instance: step = 1 minute, enabled = true, batchSize = 10,000.
     */
    StepRegistryConfig DEFAULT = new StepRegistryConfig() {
    };

}
```

#### File: `src/main/java/dev/linhvu/micrometer/step/StepMeterRegistry.java` [NEW]

```java
package dev.linhvu.micrometer.step;

import dev.linhvu.micrometer.Clock;
import dev.linhvu.micrometer.Counter;
import dev.linhvu.micrometer.DistributionSummary;
import dev.linhvu.micrometer.FunctionCounter;
import dev.linhvu.micrometer.FunctionTimer;
import dev.linhvu.micrometer.Gauge;
import dev.linhvu.micrometer.LongTaskTimer;
import dev.linhvu.micrometer.Meter;
import dev.linhvu.micrometer.Timer;
import dev.linhvu.micrometer.distribution.DistributionStatisticConfig;
import dev.linhvu.micrometer.internal.DefaultGauge;
import dev.linhvu.micrometer.internal.DefaultLongTaskTimer;
import dev.linhvu.micrometer.push.PushMeterRegistry;
import dev.linhvu.micrometer.push.PushRegistryConfig;

import java.util.concurrent.Executors;
import java.util.concurrent.ScheduledExecutorService;
import java.util.concurrent.ThreadFactory;
import java.util.concurrent.TimeUnit;
import java.util.function.ToDoubleFunction;
import java.util.function.ToLongFunction;

/**
 * Abstract registry with step/interval-based meter semantics. Extends
 * {@link PushMeterRegistry} to add a <b>second scheduling loop</b> that polls
 * meters at step boundaries to trigger value rollover.
 * <p>
 * <b>Two scheduling loops:</b>
 * <ol>
 *   <li><b>Polling loop</b> ({@code meterPollingService}): Fires 1ms into each
 *       new step boundary. Reads each step meter's value, triggering rollover.
 *       This captures the data from the just-completed step into "previous".</li>
 *   <li><b>Publishing loop</b> (inherited from {@link PushMeterRegistry}): Fires at a
 *       random offset within the step (at least 2ms in). Calls {@link #publish()}
 *       which reads "previous" values and sends them to the backend.</li>
 * </ol>
 * <p>
 * Because the polling loop fires first (1ms) and publishing fires later (2ms+),
 * the publish always sees freshly rolled-over data.
 * <p>
 * <b>Shutdown protocol:</b>
 * <ol>
 *   <li>Stop the polling service</li>
 *   <li>If data was rolled over but not yet published, do an extra publish</li>
 *   <li>Force a closing rollover on all step meters (drains partial-step data)</li>
 *   <li>Delegate to {@link PushMeterRegistry#close()} for the final publish</li>
 * </ol>
 */
public abstract class StepMeterRegistry extends PushMeterRegistry {

    private final StepRegistryConfig config;

    private volatile ScheduledExecutorService meterPollingService;

    /**
     * Wall clock time when the last scheduled rollover started. Used to determine
     * if data for the last step was published.
     */
    private volatile long lastMeterRolloverStartTime = -1;

    protected StepMeterRegistry(StepRegistryConfig config, Clock clock) {
        super(config, clock);
        this.config = config;
    }

    // -----------------------------------------------------------------------
    // Factory methods — create step-based meter variants
    // -----------------------------------------------------------------------

    @Override
    protected Counter newCounter(Meter.Id id) {
        return new StepCounter(id, clock, config.step().toMillis());
    }

    @Override
    protected <T> Gauge newGauge(Meter.Id id, T obj, ToDoubleFunction<T> valueFunction) {
        // Gauges don't need step semantics — they sample a current value
        return new DefaultGauge<>(id, obj, valueFunction);
    }

    @Override
    protected Timer newTimer(Meter.Id id, DistributionStatisticConfig distributionStatisticConfig) {
        return new StepTimer(id, clock, getBaseTimeUnit(), config.step().toMillis(),
                distributionStatisticConfig);
    }

    @Override
    protected DistributionSummary newDistributionSummary(Meter.Id id, double scale,
            DistributionStatisticConfig distributionStatisticConfig) {
        return new StepDistributionSummary(id, clock, scale, config.step().toMillis(),
                distributionStatisticConfig);
    }

    @Override
    protected LongTaskTimer newLongTaskTimer(Meter.Id id,
            DistributionStatisticConfig distributionStatisticConfig) {
        // LongTaskTimers track active tasks — they don't use step semantics
        return new DefaultLongTaskTimer(id, clock, getBaseTimeUnit());
    }

    @Override
    protected <T> FunctionCounter newFunctionCounter(Meter.Id id, T obj,
            ToDoubleFunction<T> countFunction) {
        return new StepFunctionCounter<>(id, clock, config.step().toMillis(), obj, countFunction);
    }

    @Override
    protected <T> FunctionTimer newFunctionTimer(Meter.Id id, T obj,
            ToLongFunction<T> countFunction, ToDoubleFunction<T> totalTimeFunction,
            TimeUnit totalTimeFunctionUnit) {
        return new StepFunctionTimer<>(id, clock, config.step().toMillis(),
                obj, countFunction, totalTimeFunction, totalTimeFunctionUnit, getBaseTimeUnit());
    }

    // -----------------------------------------------------------------------
    // Scheduling lifecycle
    // -----------------------------------------------------------------------

    /**
     * Starts both the publishing loop (via {@code super.start()}) and the
     * meter polling loop that triggers step rollovers.
     */
    @Override
    public void start(ThreadFactory threadFactory) {
        super.start(threadFactory);

        if (config.enabled()) {
            if (meterPollingService != null) {
                meterPollingService.shutdown();
            }
            meterPollingService = Executors.newSingleThreadScheduledExecutor(threadFactory);
            long stepMillis = config.step().toMillis();
            // Fire 1ms into the next step boundary
            long initialDelay = stepMillis - (clock.wallTime() % stepMillis) + 1;
            meterPollingService.scheduleAtFixedRate(
                    this::pollMetersToRollover,
                    initialDelay, stepMillis, TimeUnit.MILLISECONDS);
        }
    }

    /**
     * Iterates all registered meters and reads their count/value, which triggers
     * the rollover logic inside the {@link StepValue}/{@link StepTuple2} instances.
     * <p>
     * Gauges and LongTaskTimers are skipped — they don't have step semantics.
     */
    void pollMetersToRollover() {
        lastMeterRolloverStartTime = clock.wallTime();
        for (Meter meter : getMeters()) {
            meter.match(
                    Counter::count,        // triggers StepCounter rollover
                    gauge -> null,         // gauges don't need rollover
                    Timer::count,          // triggers StepTimer rollover
                    DistributionSummary::count,  // triggers StepDistributionSummary rollover
                    longTaskTimer -> null,  // LTTs don't need rollover
                    FunctionCounter::count, // triggers StepFunctionCounter rollover
                    FunctionTimer::count,  // triggers StepFunctionTimer rollover
                    m -> null              // custom meters — skip
            );
        }
    }

    // -----------------------------------------------------------------------
    // Shutdown
    // -----------------------------------------------------------------------

    @Override
    public void close() {
        if (meterPollingService != null) {
            meterPollingService.shutdown();
            meterPollingService = null;
        }

        if (config.enabled() && !isClosed()) {
            // If data was rolled over but not yet published, publish it now
            if (shouldPublishDataForLastStep()) {
                publishSafelyOrSkipIfInProgress();
            }
            // Force a final rollover on all step meters to drain partial-step data
            closingRolloverStepMeters();
        }

        // Delegate to PushMeterRegistry.close() which does final publish + sets closed
        super.close();
    }

    /**
     * Checks if the last meter rollover happened in a more recent step than the
     * last publish. If so, there's data that was rolled over but not yet published.
     */
    private boolean shouldPublishDataForLastStep() {
        if (lastMeterRolloverStartTime < 0) {
            return false;
        }
        long stepMillis = config.step().toMillis();
        long lastRolloverStep = lastMeterRolloverStartTime / stepMillis;
        long currentStep = clock.wallTime() / stepMillis;
        // If the rollover happened in a step before the current one, the data
        // should have been published; if it happened in the current step, it
        // may not have been published yet
        return lastRolloverStep >= currentStep;
    }

    /**
     * Calls {@code _closingRollover()} on every step-based meter, draining whatever
     * is in the current accumulator as final partial-step data.
     */
    private void closingRolloverStepMeters() {
        for (Meter meter : getMeters()) {
            if (meter instanceof StepMeter stepMeter) {
                stepMeter._closingRollover();
            }
        }
    }

    /**
     * Override to set the histogram expiry to match the step duration,
     * so histogram data aligns with the step interval.
     */
    @Override
    protected DistributionStatisticConfig defaultHistogramConfig() {
        return DistributionStatisticConfig.builder()
                .expiry(config.step())
                .build()
                .merge(DistributionStatisticConfig.DEFAULT);
    }

}
```

### Test Code

#### File: `src/test/java/dev/linhvu/micrometer/step/StepValueTest.java` [NEW]

```java
package dev.linhvu.micrometer.step;

import dev.linhvu.micrometer.MockClock;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.util.concurrent.TimeUnit;

import static org.assertj.core.api.Assertions.assertThat;

/**
 * Tests for {@link StepValue} — the core step-rollover primitive.
 * Uses a concrete {@link StepDouble} to test the abstract behavior.
 */
class StepValueTest {

    private MockClock clock;
    private StepDouble stepDouble;

    private static final long STEP_MILLIS = 10_000; // 10 seconds

    @BeforeEach
    void setUp() {
        clock = new MockClock();
        stepDouble = new StepDouble(clock, STEP_MILLIS);
    }

    @Test
    void shouldReturnZero_WhenNoValueRecordedAndPolled() {
        assertThat(stepDouble.poll()).isEqualTo(0.0);
    }

    @Test
    void shouldReturnZero_WhenValueRecordedButStepNotCrossed() {
        stepDouble.getCurrent().add(5.0);
        assertThat(stepDouble.poll()).isEqualTo(0.0);
    }

    @Test
    void shouldReturnAccumulatedValue_WhenStepBoundaryCrossed() {
        stepDouble.getCurrent().add(5.0);
        stepDouble.getCurrent().add(3.0);
        clock.add(STEP_MILLIS, TimeUnit.MILLISECONDS);
        assertThat(stepDouble.poll()).isEqualTo(8.0);
    }

    @Test
    void shouldResetAccumulator_AfterRollover() {
        stepDouble.getCurrent().add(5.0);
        clock.add(STEP_MILLIS, TimeUnit.MILLISECONDS);
        stepDouble.poll();

        stepDouble.getCurrent().add(2.0);
        clock.add(STEP_MILLIS, TimeUnit.MILLISECONDS);
        assertThat(stepDouble.poll()).isEqualTo(2.0);
    }

    @Test
    void shouldReturnNoValue_WhenStepGapExceedsOne() {
        stepDouble.getCurrent().add(5.0);
        clock.add(2 * STEP_MILLIS, TimeUnit.MILLISECONDS);
        assertThat(stepDouble.poll()).isEqualTo(0.0);
    }

    @Test
    void shouldReturnSameValue_WhenPolledMultipleTimesWithinSameStep() {
        stepDouble.getCurrent().add(5.0);
        clock.add(STEP_MILLIS, TimeUnit.MILLISECONDS);
        assertThat(stepDouble.poll()).isEqualTo(5.0);
        assertThat(stepDouble.poll()).isEqualTo(5.0);
        assertThat(stepDouble.poll()).isEqualTo(5.0);
    }

    @Test
    void shouldDrainCurrentAccumulator_WhenClosingRollover() {
        stepDouble.getCurrent().add(7.0);
        stepDouble._closingRollover();
        assertThat(stepDouble.poll()).isEqualTo(7.0);
    }

    @Test
    void shouldPreventFutureRollovers_AfterClosingRollover() {
        stepDouble.getCurrent().add(7.0);
        stepDouble._closingRollover();
        clock.add(STEP_MILLIS, TimeUnit.MILLISECONDS);
        assertThat(stepDouble.poll()).isEqualTo(7.0);
    }

}
```

#### File: `src/test/java/dev/linhvu/micrometer/step/StepTuple2Test.java` [NEW]

```java
package dev.linhvu.micrometer.step;

import dev.linhvu.micrometer.MockClock;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.util.concurrent.TimeUnit;
import java.util.concurrent.atomic.DoubleAdder;
import java.util.concurrent.atomic.LongAdder;

import static org.assertj.core.api.Assertions.assertThat;

class StepTuple2Test {

    private MockClock clock;
    private LongAdder count;
    private DoubleAdder total;
    private StepTuple2<Long, Double> tuple;

    private static final long STEP_MILLIS = 10_000;

    @BeforeEach
    void setUp() {
        clock = new MockClock();
        count = new LongAdder();
        total = new DoubleAdder();
        tuple = new StepTuple2<>(clock, STEP_MILLIS,
                0L, 0.0,
                count::sumThenReset, total::sumThenReset);
    }

    @Test
    void shouldReturnNoValues_WhenNothingRecorded() {
        assertThat(tuple.poll1()).isEqualTo(0L);
        assertThat(tuple.poll2()).isEqualTo(0.0);
    }

    @Test
    void shouldReturnBothValues_WhenStepBoundaryCrossed() {
        count.add(3);
        total.add(150.0);
        clock.add(STEP_MILLIS, TimeUnit.MILLISECONDS);
        assertThat(tuple.poll1()).isEqualTo(3L);
        assertThat(tuple.poll2()).isEqualTo(150.0);
    }

    @Test
    void shouldRolloverAtomically_WhenBothValuesPresent() {
        count.add(5);
        total.add(250.0);
        clock.add(STEP_MILLIS, TimeUnit.MILLISECONDS);
        Long c = tuple.poll1();
        Double t = tuple.poll2();
        assertThat(c).isEqualTo(5L);
        assertThat(t).isEqualTo(250.0);
        assertThat(t / c).isEqualTo(50.0);
    }

    @Test
    void shouldReturnNoValues_WhenStepGapExceedsOne() {
        count.add(5);
        total.add(250.0);
        clock.add(2 * STEP_MILLIS, TimeUnit.MILLISECONDS);
        assertThat(tuple.poll1()).isEqualTo(0L);
        assertThat(tuple.poll2()).isEqualTo(0.0);
    }

    @Test
    void shouldDrainBothValues_WhenClosingRollover() {
        count.add(3);
        total.add(100.0);
        tuple._closingRollover();
        assertThat(tuple.poll1()).isEqualTo(3L);
        assertThat(tuple.poll2()).isEqualTo(100.0);
    }

}
```

#### File: `src/test/java/dev/linhvu/micrometer/step/StepCounterTest.java` [NEW]

```java
package dev.linhvu.micrometer.step;

import dev.linhvu.micrometer.Meter;
import dev.linhvu.micrometer.MockClock;
import dev.linhvu.micrometer.Tags;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.util.concurrent.TimeUnit;

import static org.assertj.core.api.Assertions.assertThat;

class StepCounterTest {

    private MockClock clock;
    private StepCounter counter;

    private static final long STEP_MILLIS = 10_000;

    @BeforeEach
    void setUp() {
        clock = new MockClock();
        counter = new StepCounter(
                new Meter.Id("test.counter", Tags.empty(), Meter.Type.COUNTER, null, null),
                clock, STEP_MILLIS);
    }

    @Test
    void shouldReturnZero_WhenNoIncrementsInPreviousStep() {
        assertThat(counter.count()).isEqualTo(0.0);
    }

    @Test
    void shouldReturnStepCount_WhenStepBoundaryCrossed() {
        counter.increment();
        counter.increment();
        counter.increment();
        clock.add(STEP_MILLIS, TimeUnit.MILLISECONDS);
        assertThat(counter.count()).isEqualTo(3.0);
    }

    @Test
    void shouldReturnZero_WhenValueRecordedButStepNotCrossed() {
        counter.increment(5.0);
        assertThat(counter.count()).isEqualTo(0.0);
    }

    @Test
    void shouldResetBetweenSteps_WhenMultipleStepsCrossed() {
        counter.increment(5.0);
        clock.add(STEP_MILLIS, TimeUnit.MILLISECONDS);
        assertThat(counter.count()).isEqualTo(5.0);
        counter.increment(2.0);
        clock.add(STEP_MILLIS, TimeUnit.MILLISECONDS);
        assertThat(counter.count()).isEqualTo(2.0);
    }

    @Test
    void shouldSupportFractionalIncrements() {
        counter.increment(0.5);
        counter.increment(0.3);
        clock.add(STEP_MILLIS, TimeUnit.MILLISECONDS);
        assertThat(counter.count()).isCloseTo(0.8, org.assertj.core.data.Offset.offset(0.001));
    }

    @Test
    void shouldDrainPartialStep_WhenClosingRollover() {
        counter.increment(7.0);
        counter._closingRollover();
        assertThat(counter.count()).isEqualTo(7.0);
    }

}
```

#### File: `src/test/java/dev/linhvu/micrometer/step/StepTimerTest.java` [NEW]

```java
package dev.linhvu.micrometer.step;

import dev.linhvu.micrometer.Meter;
import dev.linhvu.micrometer.MockClock;
import dev.linhvu.micrometer.Tags;
import dev.linhvu.micrometer.distribution.DistributionStatisticConfig;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.util.concurrent.TimeUnit;

import static org.assertj.core.api.Assertions.assertThat;

class StepTimerTest {

    private MockClock clock;
    private StepTimer timer;

    private static final long STEP_MILLIS = 10_000;

    @BeforeEach
    void setUp() {
        clock = new MockClock();
        timer = new StepTimer(
                new Meter.Id("test.timer", Tags.empty(), Meter.Type.TIMER, null, null),
                clock, TimeUnit.SECONDS, STEP_MILLIS,
                DistributionStatisticConfig.DEFAULT);
    }

    @Test
    void shouldReturnZero_WhenNoRecordingsInPreviousStep() {
        assertThat(timer.count()).isEqualTo(0);
        assertThat(timer.totalTime(TimeUnit.SECONDS)).isEqualTo(0.0);
    }

    @Test
    void shouldReturnStepValues_WhenStepBoundaryCrossed() {
        timer.record(100, TimeUnit.MILLISECONDS);
        timer.record(200, TimeUnit.MILLISECONDS);
        clock.add(STEP_MILLIS, TimeUnit.MILLISECONDS);
        assertThat(timer.count()).isEqualTo(2);
        assertThat(timer.totalTime(TimeUnit.MILLISECONDS)).isEqualTo(300.0);
    }

    @Test
    void shouldResetBetweenSteps() {
        timer.record(100, TimeUnit.MILLISECONDS);
        clock.add(STEP_MILLIS, TimeUnit.MILLISECONDS);
        assertThat(timer.count()).isEqualTo(1);
        timer.record(50, TimeUnit.MILLISECONDS);
        timer.record(50, TimeUnit.MILLISECONDS);
        clock.add(STEP_MILLIS, TimeUnit.MILLISECONDS);
        assertThat(timer.count()).isEqualTo(2);
        assertThat(timer.totalTime(TimeUnit.MILLISECONDS)).isEqualTo(100.0);
    }

    @Test
    void shouldComputeConsistentMean_WhenCountAndTotalRolloverAtomically() {
        timer.record(100, TimeUnit.MILLISECONDS);
        timer.record(200, TimeUnit.MILLISECONDS);
        timer.record(300, TimeUnit.MILLISECONDS);
        clock.add(STEP_MILLIS, TimeUnit.MILLISECONDS);
        assertThat(timer.mean(TimeUnit.MILLISECONDS)).isEqualTo(200.0);
    }

    @Test
    void shouldTrackMax_IndependentOfStepRollover() {
        timer.record(500, TimeUnit.MILLISECONDS);
        assertThat(timer.max(TimeUnit.MILLISECONDS)).isGreaterThan(0);
    }

    @Test
    void shouldDrainPartialStep_WhenClosingRollover() {
        timer.record(100, TimeUnit.MILLISECONDS);
        timer._closingRollover();
        assertThat(timer.count()).isEqualTo(1);
        assertThat(timer.totalTime(TimeUnit.MILLISECONDS)).isEqualTo(100.0);
    }

    @Test
    void shouldConvertTimeUnits_WhenRequested() {
        timer.record(1, TimeUnit.SECONDS);
        clock.add(STEP_MILLIS, TimeUnit.MILLISECONDS);
        assertThat(timer.totalTime(TimeUnit.SECONDS)).isEqualTo(1.0);
        assertThat(timer.totalTime(TimeUnit.MILLISECONDS)).isEqualTo(1000.0);
    }

}
```

#### File: `src/test/java/dev/linhvu/micrometer/step/StepDistributionSummaryTest.java` [NEW]

```java
package dev.linhvu.micrometer.step;

import dev.linhvu.micrometer.Meter;
import dev.linhvu.micrometer.MockClock;
import dev.linhvu.micrometer.Tags;
import dev.linhvu.micrometer.distribution.DistributionStatisticConfig;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.util.concurrent.TimeUnit;

import static org.assertj.core.api.Assertions.assertThat;

class StepDistributionSummaryTest {

    private MockClock clock;
    private StepDistributionSummary summary;

    private static final long STEP_MILLIS = 10_000;

    @BeforeEach
    void setUp() {
        clock = new MockClock();
        summary = new StepDistributionSummary(
                new Meter.Id("test.summary", Tags.empty(), Meter.Type.DISTRIBUTION_SUMMARY, null, null),
                clock, 1.0, STEP_MILLIS,
                DistributionStatisticConfig.DEFAULT);
    }

    @Test
    void shouldReturnZero_WhenNoRecordingsInPreviousStep() {
        assertThat(summary.count()).isEqualTo(0);
        assertThat(summary.totalAmount()).isEqualTo(0.0);
    }

    @Test
    void shouldReturnStepValues_WhenStepBoundaryCrossed() {
        summary.record(100.0);
        summary.record(200.0);
        summary.record(300.0);
        clock.add(STEP_MILLIS, TimeUnit.MILLISECONDS);
        assertThat(summary.count()).isEqualTo(3);
        assertThat(summary.totalAmount()).isEqualTo(600.0);
    }

    @Test
    void shouldResetBetweenSteps() {
        summary.record(100.0);
        clock.add(STEP_MILLIS, TimeUnit.MILLISECONDS);
        assertThat(summary.count()).isEqualTo(1);
        assertThat(summary.totalAmount()).isEqualTo(100.0);
        summary.record(50.0);
        clock.add(STEP_MILLIS, TimeUnit.MILLISECONDS);
        assertThat(summary.count()).isEqualTo(1);
        assertThat(summary.totalAmount()).isEqualTo(50.0);
    }

    @Test
    void shouldComputeConsistentMean() {
        summary.record(100.0);
        summary.record(200.0);
        clock.add(STEP_MILLIS, TimeUnit.MILLISECONDS);
        assertThat(summary.mean()).isEqualTo(150.0);
    }

    @Test
    void shouldDrainPartialStep_WhenClosingRollover() {
        summary.record(42.0);
        summary._closingRollover();
        assertThat(summary.count()).isEqualTo(1);
        assertThat(summary.totalAmount()).isEqualTo(42.0);
    }

}
```

#### File: `src/test/java/dev/linhvu/micrometer/step/StepFunctionCounterTest.java` [NEW]

```java
package dev.linhvu.micrometer.step;

import dev.linhvu.micrometer.Meter;
import dev.linhvu.micrometer.MockClock;
import dev.linhvu.micrometer.Tags;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.util.concurrent.TimeUnit;
import java.util.concurrent.atomic.AtomicLong;

import static org.assertj.core.api.Assertions.assertThat;

class StepFunctionCounterTest {

    private MockClock clock;
    private AtomicLong source;
    private StepFunctionCounter<AtomicLong> counter;

    private static final long STEP_MILLIS = 10_000;

    @BeforeEach
    void setUp() {
        clock = new MockClock();
        source = new AtomicLong(0);
        counter = new StepFunctionCounter<>(
                new Meter.Id("test.fn.counter", Tags.empty(), Meter.Type.COUNTER, null, null),
                clock, STEP_MILLIS, source, AtomicLong::get);
    }

    @Test
    void shouldReturnZero_WhenNoChanges() {
        assertThat(counter.count()).isEqualTo(0.0);
    }

    @Test
    void shouldReturnDelta_WhenStepBoundaryCrossed() {
        source.set(10);
        counter.count();
        clock.add(STEP_MILLIS, TimeUnit.MILLISECONDS);
        assertThat(counter.count()).isEqualTo(10.0);
    }

    @Test
    void shouldReturnPerStepDelta_NotCumulativeTotal() {
        source.set(10);
        counter.count();
        clock.add(STEP_MILLIS, TimeUnit.MILLISECONDS);
        counter.count();
        source.set(15);
        counter.count();
        clock.add(STEP_MILLIS, TimeUnit.MILLISECONDS);
        assertThat(counter.count()).isEqualTo(5.0);
    }

    @Test
    void shouldIgnoreNegativeDeltas_WhenSourceDecreases() {
        source.set(10);
        counter.count();
        clock.add(STEP_MILLIS, TimeUnit.MILLISECONDS);
        counter.count();
        source.set(5);
        counter.count();
        clock.add(STEP_MILLIS, TimeUnit.MILLISECONDS);
        assertThat(counter.count()).isEqualTo(0.0);
    }

    @Test
    void shouldDrainPartialStep_WhenClosingRollover() {
        source.set(10);
        counter._closingRollover();
        assertThat(counter.count()).isEqualTo(10.0);
    }

}
```

#### File: `src/test/java/dev/linhvu/micrometer/step/StepFunctionTimerTest.java` [NEW]

```java
package dev.linhvu.micrometer.step;

import dev.linhvu.micrometer.Meter;
import dev.linhvu.micrometer.MockClock;
import dev.linhvu.micrometer.Tags;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.util.concurrent.TimeUnit;
import java.util.concurrent.atomic.AtomicLong;

import static org.assertj.core.api.Assertions.assertThat;

class StepFunctionTimerTest {

    private MockClock clock;
    private AtomicLong sourceCount;
    private AtomicLong sourceTotalMillis;
    private StepFunctionTimer<Object[]> timer;

    private static final long STEP_MILLIS = 10_000;

    @BeforeEach
    void setUp() {
        clock = new MockClock();
        sourceCount = new AtomicLong(0);
        sourceTotalMillis = new AtomicLong(0);
        Object[] sources = { sourceCount, sourceTotalMillis };
        timer = new StepFunctionTimer<>(
                new Meter.Id("test.fn.timer", Tags.empty(), Meter.Type.TIMER, null, null),
                clock, STEP_MILLIS, sources,
                obj -> ((AtomicLong) obj[0]).get(),
                obj -> ((AtomicLong) obj[1]).get(),
                TimeUnit.MILLISECONDS, TimeUnit.SECONDS);
    }

    @Test
    void shouldReturnZero_WhenNoChanges() {
        assertThat(timer.count()).isEqualTo(0.0);
        assertThat(timer.totalTime(TimeUnit.SECONDS)).isEqualTo(0.0);
    }

    @Test
    void shouldReturnPerStepDeltas_WhenStepBoundaryCrossed() {
        sourceCount.set(5);
        sourceTotalMillis.set(500);
        timer.count();
        clock.add(STEP_MILLIS, TimeUnit.MILLISECONDS);
        assertThat(timer.count()).isEqualTo(5.0);
        assertThat(timer.totalTime(TimeUnit.MILLISECONDS)).isEqualTo(500.0);
    }

    @Test
    void shouldConvertTimeUnits_WhenQueried() {
        sourceCount.set(1);
        sourceTotalMillis.set(1000);
        timer.count();
        clock.add(STEP_MILLIS, TimeUnit.MILLISECONDS);
        assertThat(timer.totalTime(TimeUnit.SECONDS)).isEqualTo(1.0);
    }

    @Test
    void shouldComputeMean_WhenCountAndTotalPresent() {
        sourceCount.set(4);
        sourceTotalMillis.set(400);
        timer.count();
        clock.add(STEP_MILLIS, TimeUnit.MILLISECONDS);
        timer.count();
        assertThat(timer.mean(TimeUnit.MILLISECONDS)).isEqualTo(100.0);
    }

    @Test
    void shouldDrainPartialStep_WhenClosingRollover() {
        sourceCount.set(3);
        sourceTotalMillis.set(300);
        timer._closingRollover();
        assertThat(timer.count()).isEqualTo(3.0);
        assertThat(timer.totalTime(TimeUnit.MILLISECONDS)).isEqualTo(300.0);
    }

}
```

#### File: `src/test/java/dev/linhvu/micrometer/step/StepMeterRegistryTest.java` [NEW]

```java
package dev.linhvu.micrometer.step;

import dev.linhvu.micrometer.Clock;
import dev.linhvu.micrometer.Counter;
import dev.linhvu.micrometer.DistributionSummary;
import dev.linhvu.micrometer.FunctionCounter;
import dev.linhvu.micrometer.FunctionTimer;
import dev.linhvu.micrometer.Gauge;
import dev.linhvu.micrometer.MockClock;
import dev.linhvu.micrometer.Timer;
import org.junit.jupiter.api.AfterEach;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.time.Duration;
import java.util.ArrayList;
import java.util.Collections;
import java.util.List;
import java.util.concurrent.TimeUnit;
import java.util.concurrent.atomic.AtomicInteger;
import java.util.concurrent.atomic.AtomicLong;

import static org.assertj.core.api.Assertions.assertThat;

class StepMeterRegistryTest {

    private MockClock clock;
    private AtomicInteger publishCount;
    private List<String> publishLog;
    private TestStepMeterRegistry registry;

    @BeforeEach
    void setUp() {
        clock = new MockClock();
        publishCount = new AtomicInteger(0);
        publishLog = Collections.synchronizedList(new ArrayList<>());
    }

    @AfterEach
    void tearDown() {
        if (registry != null && !registry.isClosed()) {
            registry.close();
        }
    }

    private TestStepMeterRegistry createRegistry(StepRegistryConfig config) {
        registry = new TestStepMeterRegistry(config, clock, publishCount, publishLog);
        return registry;
    }

    @Test
    void shouldCreateStepCounter_WhenCounterRegistered() {
        TestStepMeterRegistry reg = createRegistry(StepRegistryConfig.DEFAULT);
        Counter counter = reg.counter("test.counter");
        assertThat(counter).isInstanceOf(StepCounter.class);
    }

    @Test
    void shouldCreateStepTimer_WhenTimerRegistered() {
        TestStepMeterRegistry reg = createRegistry(StepRegistryConfig.DEFAULT);
        Timer timer = reg.timer("test.timer");
        assertThat(timer).isInstanceOf(StepTimer.class);
    }

    @Test
    void shouldCreateStepDistributionSummary_WhenSummaryRegistered() {
        TestStepMeterRegistry reg = createRegistry(StepRegistryConfig.DEFAULT);
        DistributionSummary summary = reg.summary("test.summary");
        assertThat(summary).isInstanceOf(StepDistributionSummary.class);
    }

    @Test
    void shouldCreateStepFunctionCounter_WhenFunctionCounterRegistered() {
        TestStepMeterRegistry reg = createRegistry(StepRegistryConfig.DEFAULT);
        AtomicLong source = new AtomicLong(0);
        FunctionCounter fc = FunctionCounter.builder("test.fn.counter", source, AtomicLong::get)
                .register(reg);
        assertThat(fc).isInstanceOf(StepFunctionCounter.class);
    }

    @Test
    void shouldCreateStepFunctionTimer_WhenFunctionTimerRegistered() {
        TestStepMeterRegistry reg = createRegistry(StepRegistryConfig.DEFAULT);
        AtomicLong source = new AtomicLong(0);
        FunctionTimer ft = FunctionTimer.builder("test.fn.timer", source,
                        AtomicLong::get, AtomicLong::doubleValue, TimeUnit.MILLISECONDS)
                .register(reg);
        assertThat(ft).isInstanceOf(StepFunctionTimer.class);
    }

    @Test
    void shouldReportPerStepCount_WhenStepBoundaryCrossed() {
        StepRegistryConfig config = new StepRegistryConfig() {
            @Override public Duration step() { return Duration.ofSeconds(10); }
        };
        TestStepMeterRegistry reg = createRegistry(config);
        Counter counter = reg.counter("test.counter");
        counter.increment(5);
        clock.add(10, TimeUnit.SECONDS);
        assertThat(counter.count()).isEqualTo(5.0);
        counter.increment(2);
        clock.add(10, TimeUnit.SECONDS);
        assertThat(counter.count()).isEqualTo(2.0);
    }

    @Test
    void shouldReturnZeroForCurrentStep_WhenValueRecordedButStepNotCrossed() {
        StepRegistryConfig config = new StepRegistryConfig() {
            @Override public Duration step() { return Duration.ofSeconds(10); }
        };
        TestStepMeterRegistry reg = createRegistry(config);
        Counter counter = reg.counter("test.counter");
        counter.increment(5);
        assertThat(counter.count()).isEqualTo(0.0);
    }

    @Test
    void shouldTriggerRollover_WhenPollMetersCalledAfterStepBoundary() {
        StepRegistryConfig config = new StepRegistryConfig() {
            @Override public Duration step() { return Duration.ofSeconds(10); }
        };
        TestStepMeterRegistry reg = createRegistry(config);
        Counter counter = reg.counter("test.counter");
        counter.increment(5);
        clock.add(10, TimeUnit.SECONDS);
        reg.pollMetersToRollover();
        assertThat(counter.count()).isEqualTo(5.0);
    }

    @Test
    void shouldDrainPartialStepData_WhenClosed() {
        StepRegistryConfig config = new StepRegistryConfig() {
            @Override public Duration step() { return Duration.ofSeconds(10); }
        };
        TestStepMeterRegistry reg = createRegistry(config);
        Counter counter = reg.counter("test.counter");
        counter.increment(7);
        reg.close();
        assertThat(publishCount.get()).isGreaterThanOrEqualTo(1);
    }

    @Test
    void shouldPublishOnClose_WhenEnabled() {
        TestStepMeterRegistry reg = createRegistry(StepRegistryConfig.DEFAULT);
        reg.counter("test.counter").increment(5);
        reg.close();
        assertThat(publishCount.get()).isGreaterThanOrEqualTo(1);
    }

    @Test
    void shouldNotPublishOnClose_WhenDisabled() {
        StepRegistryConfig config = new StepRegistryConfig() {
            @Override public boolean enabled() { return false; }
        };
        TestStepMeterRegistry reg = createRegistry(config);
        reg.counter("test.counter").increment(5);
        reg.close();
        assertThat(publishCount.get()).isEqualTo(0);
    }

    @Test
    void shouldCreateNonStepGauge_WhenGaugeRegistered() {
        TestStepMeterRegistry reg = createRegistry(StepRegistryConfig.DEFAULT);
        AtomicLong value = new AtomicLong(42);
        reg.gauge("test.gauge", value);
        Gauge gauge = (Gauge) reg.getMeters().stream()
                .filter(m -> m.getId().getName().equals("test.gauge"))
                .findFirst().orElseThrow();
        assertThat(gauge.value()).isEqualTo(42.0);
    }

    @Test
    void shouldSetHistogramExpiry_ToStepDuration() {
        StepRegistryConfig config = new StepRegistryConfig() {
            @Override public Duration step() { return Duration.ofSeconds(30); }
        };
        TestStepMeterRegistry reg = createRegistry(config);
        assertThat(reg.defaultHistogramConfig().getExpiry()).isEqualTo(Duration.ofSeconds(30));
    }

    static class TestStepMeterRegistry extends StepMeterRegistry {
        private final AtomicInteger publishCount;
        private final List<String> publishLog;

        TestStepMeterRegistry(StepRegistryConfig config, Clock clock,
                AtomicInteger publishCount, List<String> publishLog) {
            super(config, clock);
            this.publishCount = publishCount;
            this.publishLog = publishLog;
        }

        @Override protected void publish() {
            publishCount.incrementAndGet();
            publishLog.add("publish@" + System.currentTimeMillis());
        }

        @Override protected TimeUnit getBaseTimeUnit() { return TimeUnit.SECONDS; }

        @Override
        public dev.linhvu.micrometer.distribution.DistributionStatisticConfig defaultHistogramConfig() {
            return super.defaultHistogramConfig();
        }
    }

}
```

#### File: `src/test/java/dev/linhvu/micrometer/step/integration/StepRegistryIntegrationTest.java` [NEW]

```java
package dev.linhvu.micrometer.step.integration;

import dev.linhvu.micrometer.Clock;
import dev.linhvu.micrometer.Counter;
import dev.linhvu.micrometer.DistributionSummary;
import dev.linhvu.micrometer.FunctionCounter;
import dev.linhvu.micrometer.FunctionTimer;
import dev.linhvu.micrometer.MockClock;
import dev.linhvu.micrometer.Tags;
import dev.linhvu.micrometer.Timer;
import dev.linhvu.micrometer.config.MeterFilter;
import dev.linhvu.micrometer.step.StepMeterRegistry;
import dev.linhvu.micrometer.step.StepRegistryConfig;
import org.junit.jupiter.api.AfterEach;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.time.Duration;
import java.util.ArrayList;
import java.util.Collections;
import java.util.List;
import java.util.concurrent.TimeUnit;
import java.util.concurrent.atomic.AtomicInteger;
import java.util.concurrent.atomic.AtomicLong;

import static org.assertj.core.api.Assertions.assertThat;

class StepRegistryIntegrationTest {

    private MockClock clock;
    private AtomicInteger publishCount;
    private List<List<MeterSnapshot>> publishedSnapshots;
    private TestStepRegistry registry;

    private static final long STEP_SECONDS = 10;

    @BeforeEach
    void setUp() {
        clock = new MockClock();
        publishCount = new AtomicInteger(0);
        publishedSnapshots = Collections.synchronizedList(new ArrayList<>());
    }

    @AfterEach
    void tearDown() {
        if (registry != null && !registry.isClosed()) { registry.close(); }
    }

    private TestStepRegistry createRegistry() {
        StepRegistryConfig config = new StepRegistryConfig() {
            @Override public Duration step() { return Duration.ofSeconds(STEP_SECONDS); }
        };
        registry = new TestStepRegistry(config, clock, publishCount, publishedSnapshots);
        return registry;
    }

    @Test
    void shouldPublishPerStepValues_WhenFullLifecycleCompletes() {
        TestStepRegistry reg = createRegistry();
        Counter counter = reg.counter("http.requests");
        Timer timer = reg.timer("http.latency");
        counter.increment(10);
        timer.record(100, TimeUnit.MILLISECONDS);
        timer.record(200, TimeUnit.MILLISECONDS);
        clock.add(STEP_SECONDS, TimeUnit.SECONDS);
        assertThat(counter.count()).isEqualTo(10.0);
        assertThat(timer.count()).isEqualTo(2);
        assertThat(timer.totalTime(TimeUnit.MILLISECONDS)).isEqualTo(300.0);
        counter.increment(3);
        timer.record(50, TimeUnit.MILLISECONDS);
        clock.add(STEP_SECONDS, TimeUnit.SECONDS);
        assertThat(counter.count()).isEqualTo(3.0);
        assertThat(timer.count()).isEqualTo(1);
        assertThat(timer.totalTime(TimeUnit.MILLISECONDS)).isEqualTo(50.0);
    }

    @Test
    void shouldApplyFilters_WhenStepMetersRegistered() {
        TestStepRegistry reg = createRegistry();
        reg.meterFilter(MeterFilter.commonTags(Tags.of("env", "prod")));
        Counter counter = reg.counter("http.requests");
        assertThat(counter.getId().getTagsAsIterable())
                .anyMatch(tag -> tag.getKey().equals("env") && tag.getValue().equals("prod"));
    }

    @Test
    void shouldReturnNoopMeter_WhenFilterDeniesStepMeter() {
        TestStepRegistry reg = createRegistry();
        reg.meterFilter(MeterFilter.deny());
        Counter counter = reg.counter("denied.counter");
        counter.increment(5);
        assertThat(reg.getMeters()).isEmpty();
    }

    @Test
    void shouldHandleMixedMeterTypes_InSameRegistry() {
        TestStepRegistry reg = createRegistry();
        Counter counter = reg.counter("requests");
        Timer timer = reg.timer("latency");
        DistributionSummary summary = reg.summary("payload.size");
        AtomicLong source = new AtomicLong(100);
        FunctionCounter fc = FunctionCounter.builder("cache.hits", source, AtomicLong::get)
                .register(reg);
        counter.increment(5);
        timer.record(100, TimeUnit.MILLISECONDS);
        summary.record(1024.0);
        fc.count();
        clock.add(STEP_SECONDS, TimeUnit.SECONDS);
        assertThat(counter.count()).isEqualTo(5.0);
        assertThat(timer.count()).isEqualTo(1);
        assertThat(summary.count()).isEqualTo(1);
        assertThat(fc.count()).isEqualTo(100.0);
    }

    @Test
    void shouldCapturePartialStepData_OnGracefulShutdown() {
        TestStepRegistry reg = createRegistry();
        Counter counter = reg.counter("requests");
        counter.increment(7);
        reg.close();
        assertThat(publishCount.get()).isGreaterThanOrEqualTo(1);
    }

    @Test
    void shouldTrackFunctionTimerDeltas_AcrossSteps() {
        TestStepRegistry reg = createRegistry();
        AtomicLong callCount = new AtomicLong(0);
        AtomicLong totalTimeMs = new AtomicLong(0);
        FunctionTimer ft = FunctionTimer.builder("db.query", new Object[] { callCount, totalTimeMs },
                        obj -> ((AtomicLong) ((Object[]) obj)[0]).get(),
                        obj -> ((AtomicLong) ((Object[]) obj)[1]).doubleValue(),
                        TimeUnit.MILLISECONDS)
                .register(reg);
        callCount.set(10);
        totalTimeMs.set(500);
        ft.count();
        ft.totalTime(TimeUnit.MILLISECONDS);
        clock.add(STEP_SECONDS, TimeUnit.SECONDS);
        assertThat(ft.count()).isEqualTo(10.0);
        assertThat(ft.totalTime(TimeUnit.MILLISECONDS)).isEqualTo(500.0);
        callCount.set(15);
        totalTimeMs.set(700);
        ft.count();
        ft.totalTime(TimeUnit.MILLISECONDS);
        clock.add(STEP_SECONDS, TimeUnit.SECONDS);
        assertThat(ft.count()).isEqualTo(5.0);
        assertThat(ft.totalTime(TimeUnit.MILLISECONDS)).isEqualTo(200.0);
    }

    @Test
    void shouldReturnZero_WhenStepIsSkipped() {
        TestStepRegistry reg = createRegistry();
        Counter counter = reg.counter("requests");
        counter.increment(5);
        clock.add(2 * STEP_SECONDS, TimeUnit.SECONDS);
        assertThat(counter.count()).isEqualTo(0.0);
    }

    record MeterSnapshot(String name, double value) {}

    static class TestStepRegistry extends StepMeterRegistry {
        private final AtomicInteger publishCount;
        private final List<List<MeterSnapshot>> publishedSnapshots;

        TestStepRegistry(StepRegistryConfig config, Clock clock,
                AtomicInteger publishCount, List<List<MeterSnapshot>> publishedSnapshots) {
            super(config, clock);
            this.publishCount = publishCount;
            this.publishedSnapshots = publishedSnapshots;
        }

        @Override protected void publish() {
            publishCount.incrementAndGet();
            List<MeterSnapshot> snapshot = new ArrayList<>();
            for (var meter : getMeters()) {
                for (var measurement : meter.measure()) {
                    snapshot.add(new MeterSnapshot(
                            meter.getId().getName() + "/" + measurement.getStatistic(),
                            measurement.getValue()));
                }
            }
            publishedSnapshots.add(snapshot);
        }

        @Override protected TimeUnit getBaseTimeUnit() { return TimeUnit.SECONDS; }
    }

}
```

---

## Summary

| Concept | What It Means |
|---------|---------------|
| **StepValue\<V\>** | Core primitive: accumulates values during a step, drains via `sumThenReset()` at step boundaries, returns previous step's value on `poll()` |
| **StepTuple2\<T1,T2\>** | Rolls over two values atomically — ensures count and total always come from the same step interval |
| **CAS-based rollover** | `AtomicLong.compareAndSet` on `lastInitPos` ensures exactly one thread drains the accumulator, even under concurrent reads |
| **Gap detection** | If `lastInit < stepTime - 1`, a step was missed — previous is set to `noValue()` to prevent stale data |
| **StepCounter / StepTimer / StepDistributionSummary** | Step-based meter implementations: `count()` returns per-interval delta, not lifetime cumulative |
| **StepFunctionCounter / StepFunctionTimer** | Adapts monotonically increasing external counters into per-step deltas via `current - last` |
| **Two scheduling loops** | Polling loop (1ms into step) triggers rollover; publishing loop (2ms+ into step) reads rolled-over data |
| **Closing rollover** | `_closingRollover()` drains partial-step data on shutdown, sets `lastInitPos = MAX_VALUE` to prevent future rollovers |
| **StepMeterRegistry** | Abstract registry that creates step-based meters and manages the two-loop lifecycle |

**Next: Chapter 17 — Logging Registry** — A concrete push registry that publishes metrics by logging them — serves as both a debugging tool and a reference push implementation.
