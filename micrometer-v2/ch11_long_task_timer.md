# Chapter 11: LongTaskTimer — Tracking In-Flight Work

> **Feature 11** · Tier 2 · Depends on: Feature 1 (Counter & MeterRegistry)

After this chapter you will have a meter that tracks **in-flight long-running operations** —
database migrations, batch processing jobs, large file transfers — reporting how many tasks
are active, how long they have been running, and which is the longest. Unlike `Timer`, which
records durations of *completed* events, `LongTaskTimer` reports on tasks that have not
finished yet.

---

## 1. The API Contract

Before writing any implementation, let's define exactly what a client sees.

### What the client imports

```java
import simple.micrometer.api.LongTaskTimer;
import simple.micrometer.api.MeterRegistry;
import simple.micrometer.internal.SimpleMeterRegistry;
```

### What the client writes

```java
// 1. Create a registry (reusing from Feature 1)
MeterRegistry registry = new SimpleMeterRegistry();

// 2. Create a long task timer — via Builder
LongTaskTimer longTimer = LongTaskTimer.builder("data.migration")
    .tag("table", "users")
    .description("Tracks active data migrations")
    .register(registry);

// 3. Start tracking a long-running task
LongTaskTimer.Sample task = longTimer.start();

// 4. While the task is running, observe:
int active = longTimer.activeTasks();        // 1
double elapsed = longTimer.duration(TimeUnit.SECONDS);
double longest = longTimer.max(TimeUnit.SECONDS);
double average = longTimer.mean(TimeUnit.SECONDS);

// 5. Query the sample itself
double taskElapsed = task.duration(TimeUnit.SECONDS);

// 6. When done:
long durationNanos = task.stop();  // active tasks -> 0

// 7. Alternative: create via More convenience
LongTaskTimer ltt = registry.more().longTaskTimer("batch.job", "type", "export");

// 8. Convenience recording methods
ltt.record(() -> performMigration());
String result = ltt.record(() -> computeResult());
```

### Behavioral contract

| Promise | Detail |
|---------|--------|
| In-flight tracking | Reports on tasks that have *not* finished yet — active count, cumulative duration, max duration |
| Start/stop lifecycle | `start()` returns a `Sample`; `Sample.stop()` removes the task from the active set |
| Three statistics | `activeTasks()`, `duration(unit)`, `max(unit)` — all live, queried at any time |
| Mean from defaults | `mean(unit)` is a default method: `duration / activeTasks` (returns 0 when idle) |
| Sample duration | Each `Sample` knows its own elapsed time via `duration(unit)`; returns `-1` after stop |
| Exception-safe | Convenience recording methods (`record(Runnable)`, etc.) stop the task even if the code throws |
| Convenience methods | `record(Runnable)`, `record(Supplier)`, `recordCallable(Callable)` wrap start/stop automatically |
| Three measurements | `measure()` returns ACTIVE_TASKS, DURATION, MAX (all in nanoseconds) |
| Deduplicated | Same name + tags returns the same long task timer instance |
| More registration | Registered through `registry.more().longTaskTimer(...)`, not a top-level convenience |
| Noop pattern | Denied meters return a `NoopLongTaskTimer` with inert samples, but still execute user code |

### Timer vs LongTaskTimer

```
Timer (completed events)              LongTaskTimer (in-flight work)
────────────────────────              ─────────────────────────────
Records AFTER work finishes           Tracks WHILE work is running
count / totalTime / max               activeTasks / duration / max
Push: timer.record(250, MS)           Push: sample = ltt.start()
Stopwatch: Sample → stop(timer)       Handle: sample.stop()
Sample chooses Timer at stop-time     Sample is bound to LongTaskTimer at start-time
Measures latency distributions        Measures concurrency and staleness
```

---

## 2. Client Usage & Tests

These tests define the API's behavior — they were written **before** any implementation.

### Builder + Registration

```java
@Test
void shouldCreateLongTaskTimer_WithBuilderAndRegister() {
    LongTaskTimer ltt = LongTaskTimer.builder("data.migration")
            .tag("table", "users")
            .description("Tracks active data migrations")
            .register(registry);

    assertThat(ltt).isNotNull();
    assertThat(ltt.getId().getName()).isEqualTo("data.migration");
    assertThat(ltt.getId().getTag("table")).isEqualTo("users");
    assertThat(ltt.getId().getDescription()).isEqualTo("Tracks active data migrations");
    assertThat(ltt.getId().getType()).isEqualTo(Meter.Type.LONG_TASK_TIMER);
}
```

### More Convenience Registration

```java
@Test
void shouldCreateLongTaskTimer_ViaMoreConvenience() {
    LongTaskTimer ltt = registry.more().longTaskTimer("batch.job", "type", "export");

    assertThat(ltt).isNotNull();
    assertThat(ltt.getId().getName()).isEqualTo("batch.job");
    assertThat(ltt.getId().getTag("type")).isEqualTo("export");
}
```

### Start / Stop Lifecycle

```java
@Test
void shouldTrackActiveTasks_AfterStart() {
    LongTaskTimer ltt = LongTaskTimer.builder("migration").register(registry);

    assertThat(ltt.activeTasks()).isEqualTo(0);

    LongTaskTimer.Sample task = ltt.start();
    assertThat(ltt.activeTasks()).isEqualTo(1);

    task.stop();
    assertThat(ltt.activeTasks()).isEqualTo(0);
}
```

### Multiple Active Tasks with Out-of-Order Completion

```java
@Test
void shouldTrackMultipleActiveTasks() {
    LongTaskTimer ltt = LongTaskTimer.builder("batch").register(registry);

    LongTaskTimer.Sample task1 = ltt.start();
    clock.addNanos(TimeUnit.SECONDS.toNanos(1));

    LongTaskTimer.Sample task2 = ltt.start();
    clock.addNanos(TimeUnit.SECONDS.toNanos(1));

    LongTaskTimer.Sample task3 = ltt.start();

    assertThat(ltt.activeTasks()).isEqualTo(3);

    task2.stop();  // middle task finishes first
    assertThat(ltt.activeTasks()).isEqualTo(2);

    task1.stop();
    assertThat(ltt.activeTasks()).isEqualTo(1);

    task3.stop();
    assertThat(ltt.activeTasks()).isEqualTo(0);
}
```

### Duration Queries

```java
@Test
void shouldReportTotalDuration_OfAllActiveTasks() {
    LongTaskTimer ltt = LongTaskTimer.builder("migration").register(registry);

    // Task 1 starts at t=0
    LongTaskTimer.Sample task1 = ltt.start();
    clock.addNanos(TimeUnit.SECONDS.toNanos(3));

    // Task 2 starts at t=3s
    LongTaskTimer.Sample task2 = ltt.start();
    clock.addNanos(TimeUnit.SECONDS.toNanos(2));

    // At t=5s: task1 has been running 5s, task2 has been running 2s
    // Total duration = 5 + 2 = 7 seconds
    assertThat(ltt.duration(TimeUnit.SECONDS)).isCloseTo(7.0, within(0.01));

    task1.stop();
    task2.stop();
}
```

### Max Query

```java
@Test
void shouldReportMax_AsLongestRunningTask() {
    LongTaskTimer ltt = LongTaskTimer.builder("migration").register(registry);

    // Task 1 starts at t=0 (will be the longest)
    LongTaskTimer.Sample task1 = ltt.start();
    clock.addNanos(TimeUnit.SECONDS.toNanos(5));

    // Task 2 starts at t=5s (shorter)
    LongTaskTimer.Sample task2 = ltt.start();
    clock.addNanos(TimeUnit.SECONDS.toNanos(2));

    // At t=7s: task1=7s, task2=2s -> max should be 7s
    assertThat(ltt.max(TimeUnit.SECONDS)).isCloseTo(7.0, within(0.01));

    task1.stop();

    // After task1 stops, max should be task2's duration (now 2s)
    assertThat(ltt.max(TimeUnit.SECONDS)).isCloseTo(2.0, within(0.01));

    task2.stop();
}
```

### Sample Duration and Post-Stop Behavior

```java
@Test
void shouldReportSampleDuration_WhileRunning() {
    LongTaskTimer ltt = LongTaskTimer.builder("task").register(registry);

    LongTaskTimer.Sample task = ltt.start();
    clock.addNanos(TimeUnit.SECONDS.toNanos(3));

    assertThat(task.duration(TimeUnit.SECONDS)).isCloseTo(3.0, within(0.01));

    clock.addNanos(TimeUnit.SECONDS.toNanos(2));
    assertThat(task.duration(TimeUnit.SECONDS)).isCloseTo(5.0, within(0.01));

    task.stop();
}

@Test
void shouldReturnNegativeOne_FromStoppedSampleDuration() {
    LongTaskTimer ltt = LongTaskTimer.builder("task").register(registry);

    LongTaskTimer.Sample task = ltt.start();
    clock.addNanos(TimeUnit.SECONDS.toNanos(1));
    task.stop();

    assertThat(task.duration(TimeUnit.SECONDS)).isEqualTo(-1.0);
}
```

### Exception Safety

```java
@Test
void shouldStopTask_EvenWhenRunnableThrows() {
    LongTaskTimer ltt = LongTaskTimer.builder("failing").register(registry);

    assertThatThrownBy(() -> ltt.record((Runnable) () -> {
        throw new RuntimeException("boom");
    })).isInstanceOf(RuntimeException.class);

    assertThat(ltt.activeTasks()).isEqualTo(0);
}
```

### Noop Behavior

```java
@Test
void shouldReturnNoopLongTaskTimer_WhenDeniedByFilter() {
    registry.config().meterFilter(MeterFilter.deny());

    LongTaskTimer ltt = LongTaskTimer.builder("denied.timer").register(registry);

    assertThat(ltt).isInstanceOf(NoopLongTaskTimer.class);
    assertThat(ltt.activeTasks()).isEqualTo(0);
    assertThat(ltt.duration(TimeUnit.SECONDS)).isEqualTo(0.0);
    assertThat(ltt.max(TimeUnit.SECONDS)).isEqualTo(0.0);

    // Noop sample should be inert
    LongTaskTimer.Sample sample = ltt.start();
    assertThat(sample.stop()).isEqualTo(0);
    assertThat(sample.duration(TimeUnit.SECONDS)).isEqualTo(0.0);
}

@Test
void shouldExecuteRunnable_EvenWhenNoop() {
    registry.config().meterFilter(MeterFilter.deny());
    LongTaskTimer ltt = LongTaskTimer.builder("denied").register(registry);

    AtomicLong counter = new AtomicLong(0);
    ltt.record(counter::incrementAndGet);
    assertThat(counter.get()).isEqualTo(1);
}
```

---

## 3. Implementing the Call Chain — Top Down

### The call chain

```
LongTaskTimer.builder("data.migration").tag("table","users").register(registry)
  |
  +-> [API Layer] LongTaskTimer.Builder
  |     Stores name, tags, description. Terminal op: register(registry)
  |       -> Creates Meter.Id with Type.LONG_TASK_TIMER
  |       -> Calls registry.more().longTaskTimer(id)
  |
  +-> [API Layer] MeterRegistry.More.longTaskTimer(Meter.Id)
  |     Calls registerMeterIfNecessary(LongTaskTimer.class, id,
  |         MeterRegistry.this::newLongTaskTimer, NoopLongTaskTimer::new)
  |     -- reuses the same double-checked locking from Feature 1
  |
  +-> [Implementation Layer] SimpleMeterRegistry.newLongTaskTimer(id)
        Creates DefaultLongTaskTimer(id, clock)
```

```
LongTaskTimer.Sample task = longTimer.start()
  |
  +-> [Implementation Layer] DefaultLongTaskTimer.start()
  |     Captures clock.monotonicTime() as startTime
  |     Creates SampleImpl(startTime) and adds to ConcurrentSkipListSet
  |     If collision (same nanosecond), creates SampleImplCounted with counter
  |
  +-> Returns Sample handle to caller

task.stop()
  |
  +-> [Implementation Layer] SampleImpl.stop()
        Removes this sample from the active set
        Returns elapsed nanoseconds
        Sets stopped = true
```

```
longTimer.max(TimeUnit.SECONDS)
  |
  +-> [Implementation Layer] DefaultLongTaskTimer.max()
        Calls activeTasks.first().duration(unit)
        first() = oldest start time = longest-running task
        O(1) because ConcurrentSkipListSet is ordered
```

### Timer vs LongTaskTimer: Two Flavors of Duration Tracking

```
Timer (completed events)               LongTaskTimer (in-flight work)
──────────────────────────             ──────────────────────────────
record(250, MILLISECONDS)              task = start()
  +-> count.increment()                  +-> activeTasks.add(sample)
  +-> totalNanos.add(nanos)
  +-> max.record(nanos)                task.stop()
                                         +-> activeTasks.remove(sample)
count()      -> LongAdder.longValue()
totalTime(u) -> nanosToUnit(sum, u)    activeTasks() -> skipList.size()
max(u)       -> ringBuffer[current]    duration(u)   -> sum(now - startTime)
                                       max(u)        -> skipList.first().duration()
```

Timer records after the fact and stores aggregated statistics. LongTaskTimer tracks
live handles and computes statistics on demand from the active task set.

### Layer 1: LongTaskTimer interface + Sample + Builder

**LongTaskTimer** extends `Meter` with four core methods: `start()`, `duration(unit)`,
`activeTasks()`, and `max(unit)`.

Key design decisions:

1. **`start()` is the only recording method** — unlike Timer which has `record(long, TimeUnit)`
   as its core, LongTaskTimer's fundamental operation is creating a tracked task handle.
   There is no `record(long, TimeUnit)` because you cannot retroactively report in-flight
   work.

2. **`record(Runnable)`, `record(Supplier)`, `recordCallable(Callable)` are default methods** —
   they all follow the same pattern: `start()`, execute, `stop()` in a `finally` block.
   Unlike Timer's recording methods which need a Clock reference, these only need `start()`
   and `stop()`, which the interface already provides.

3. **`mean(TimeUnit)` is a default method** — pure math (`duration / activeTasks`), works
   regardless of implementation.

4. **`measure()` returns three Measurements** — ACTIVE_TASKS, DURATION, MAX (all in
   nanoseconds). These are the statistics monitoring backends use for LongTaskTimer
   exposition.

**LongTaskTimer.Sample** is an abstract class (not a concrete class like Timer.Sample):
```java
abstract class Sample {
    public abstract long stop();
    public abstract double duration(TimeUnit unit);
}
```

The critical difference from Timer.Sample: **the LongTaskTimer is chosen at start-time,
not stop-time**. The Sample is already bound to a specific LongTaskTimer and actively
participates in its active-task accounting. Timer.Sample, by contrast, is a free-floating
stopwatch that records onto a Timer chosen at `stop()`.

**LongTaskTimer.Builder** follows the same pattern as other builders, with one twist:
```java
public LongTaskTimer register(MeterRegistry registry) {
    Meter.Id id = new Meter.Id(name, tags, null, description, Meter.Type.LONG_TASK_TIMER);
    return registry.more().longTaskTimer(id);
}
```

The builder calls `registry.more().longTaskTimer(id)` instead of a top-level method like
`registry.timer(id)`. This is the `More` accessor pattern — secondary meter types go through
`more()` to keep the `MeterRegistry` API surface focused.

### Layer 2: MeterRegistry — the More inner class

The `More` inner class is the registration gateway for secondary meter types:

```java
public class More {
    public LongTaskTimer longTaskTimer(String name, String... tags)
    public LongTaskTimer longTaskTimer(String name, Iterable<Tag> tags)
    LongTaskTimer longTaskTimer(Meter.Id id)  // package-private
}
```

The package-private method delegates to the existing `registerMeterIfNecessary()`:
```java
LongTaskTimer longTaskTimer(Meter.Id id) {
    return registerMeterIfNecessary(LongTaskTimer.class, id,
            MeterRegistry.this::newLongTaskTimer, NoopLongTaskTimer::new);
}
```

This reuses the exact same double-checked locking, filter application, and noop fallback
from Feature 1. The `More` class also introduces the `more()` accessor on MeterRegistry:

```java
private final More more = new More();

public More more() {
    return more;
}
```

And the new abstract factory method:
```java
protected abstract LongTaskTimer newLongTaskTimer(Meter.Id id);
```

### Layer 3: DefaultLongTaskTimer — ConcurrentSkipListSet

DefaultLongTaskTimer tracks active tasks in a `ConcurrentSkipListSet<SampleImpl>` ordered
by start time.

**Why a skip list?** Tasks complete out of order (a task started first may finish last).
A queue would be O(N) for arbitrary removal. A skip list gives O(log N) for both insert
and removal, plus natural ordering by start time means `max()` is O(1) — the oldest
task (smallest start time = longest duration) is always `first()` in the set.

**The start() method and collision handling:**
```java
public Sample start() {
    long startTime = clock.monotonicTime();
    SampleImpl sample = new SampleImpl(startTime);
    if (!activeTasks.add(sample)) {
        // Collision: two tasks started at the exact same nanosecond.
        sample = new SampleImplCounted(startTime, nextNonZeroCounter());
        activeTasks.add(sample);
    }
    return sample;
}
```

Since `ConcurrentSkipListSet` uses `compareTo` for equality, two tasks with the same
start time would collide. The solution is a two-class hierarchy:

- **`SampleImpl`** (counter=0): the base case for the first task at a given nanosecond
- **`SampleImplCounted`** (counter>0): disambiguates when a second task starts at the
  exact same nanosecond

The `compareTo` breaks ties using the counter:
```java
public int compareTo(SampleImpl that) {
    if (this == that) return 0;
    int cmp = Long.compare(this.startTime, that.startTime);
    if (cmp != 0) return cmp;
    return Integer.compare(this.counter(), that.counter());
}
```

**The duration() method** computes the sum on-demand:
```java
public double duration(TimeUnit unit) {
    long now = clock.monotonicTime();
    double totalNanos = 0.0;
    for (SampleImpl task : activeTasks) {
        totalNanos += (now - task.startTime);
    }
    return nanosToUnit(totalNanos, unit);
}
```

This is O(N) in the number of active tasks, but active tasks are typically few (tens, not
millions), and this is a read path called infrequently for monitoring exposition.

**The max() method** is O(1):
```java
public double max(TimeUnit unit) {
    try {
        return activeTasks.first().duration(unit);
    } catch (NoSuchElementException e) {
        return 0.0;
    }
}
```

`first()` returns the element with the smallest start time, which has been running the
longest. The `NoSuchElementException` catch handles the empty case.

### Layer 4: NoopLongTaskTimer — inert but functional

```java
public Sample start() {
    return new Sample() {
        public long stop() { return 0; }
        public double duration(TimeUnit unit) { return 0; }
    };
}
```

The noop returns anonymous `Sample` instances that do nothing. But the convenience
recording methods (`record(Runnable)`, etc.) are inherited from the `LongTaskTimer`
interface's default methods — they still call `start()` and `stop()`, but crucially,
**they still execute the user's code**. A `ltt.record(() -> doWork())` is a contract
that `doWork()` will be called regardless of meter state.

### Layer 5: SimpleMeterRegistry — one line

```java
@Override
protected LongTaskTimer newLongTaskTimer(Meter.Id id) {
    return new DefaultLongTaskTimer(id, clock);
}
```

### Layer 6: CompositeMeterRegistry — also one line

```java
@Override
protected LongTaskTimer newLongTaskTimer(Meter.Id id) {
    return new DefaultLongTaskTimer(id, clock);
}
```

Unlike other meter types (Counter, Gauge, Timer) which create composite wrappers to
fan out to child registries, LongTaskTimer tracks inherently local active-task state.
A composite version is not needed — a single DefaultLongTaskTimer suffices.

---

## 4. Try It Yourself

1. **What happens if you start 1,000 tasks at the exact same nanosecond?** The first
   uses `SampleImpl` (counter=0). The second triggers a collision and gets
   `SampleImplCounted` with counter=1. The third gets counter=2, and so on. Each is
   unique in the skip list because `compareTo` considers both `startTime` and `counter`.
   Trace through the `nextNonZeroCounter()` method.

2. **Add a `LongTaskTimer.wrap(Runnable)`** — a method that returns a `Runnable` which,
   when executed, starts a task, runs the delegate, and stops. Hint:
   `return () -> record(f);`

3. **What happens if you call `stop()` twice on the same Sample?** The first `stop()`
   removes it from the active set and sets `stopped = true`. The second `stop()` calls
   `activeTasks.remove(this)` on a sample that is already gone (harmless no-op), then
   reads `duration()` which returns `-1` because `stopped` is already true. The return
   value is `-1` cast to `long`. Is this a bug? The real Micrometer returns `-1` too.

4. **Why is `stopped` volatile?** The `stop()` call may happen on one thread while
   another thread calls `duration()`. The `volatile` keyword ensures the `stopped = true`
   write is visible to the reading thread without needing synchronization.

---

## 5. Why This Works

### In-flight vs completed: complementary views

Timer tells you "how long did past requests take?" LongTaskTimer tells you "how many
requests are still in progress, and how long have they been running?" These are
complementary — Timer is for latency histograms and SLAs, while LongTaskTimer is for
detecting stuck operations and monitoring concurrency. A database migration that has been
running for 3 hours shows up clearly in a LongTaskTimer but is invisible to a Timer until
it completes.

### ConcurrentSkipListSet for natural ordering

The key insight is that the oldest task is always the longest-running one. By ordering
tasks by start time, `max()` becomes O(1) — just read `first()`. This is more efficient
than iterating all tasks to find the maximum, and the skip list also gives O(log N) for
insert and removal, compared to O(N) for an ArrayList.

### The More accessor pattern

Not every meter type deserves a top-level convenience method on `MeterRegistry`. Counter,
Gauge, Timer, and DistributionSummary are the "primary four" — they cover 90% of use cases.
LongTaskTimer, FunctionCounter, FunctionTimer, and TimeGauge are secondary types that live
behind `registry.more()`. This keeps the main API surface clean while still providing
discoverable access to specialized meters.

### Collision handling as a subclass

Instead of adding a `counter` field to every `SampleImpl` (wasting memory in the 99.99%
case where there is no collision), the design uses a subclass `SampleImplCounted` only
when needed. The base `SampleImpl.counter()` returns 0; the subclass overrides it with the
actual value. This is a space-efficient approach that keeps the common path lean.

### Default recording methods on the interface

Unlike Timer, where `record(Runnable)` must be abstract because it needs a `Clock` to
measure elapsed time, LongTaskTimer's convenience methods can be default methods on the
interface. They only need `start()` and `stop()`, both of which are interface methods.
This means NoopLongTaskTimer inherits the same recording behavior automatically — user
code always executes, even when the meter is denied.

---

## 6. What We Enhanced

| Component | State before (Feature 10) | Enhancement in Feature 11 | Will be enhanced by |
|-----------|--------------------------|--------------------------|---------------------|
| `MeterRegistry` | Supports Counter, Gauge, Timer, DistributionSummary, MeterFilter, NamingConvention | +`More` inner class, +`more()` accessor, +`newLongTaskTimer()` abstract | Future: +FunctionCounter, +FunctionTimer, +TimeGauge in More |
| `SimpleMeterRegistry` | Creates all primary meter implementations | +`newLongTaskTimer()` -> DefaultLongTaskTimer | Stable |
| `CompositeMeterRegistry` | Creates composite wrappers for primary meters | +`newLongTaskTimer()` -> DefaultLongTaskTimer | Future: +composite long task timer if needed |
| `Search` | Finds Counter, Gauge, Timer, DistributionSummary | +`longTaskTimer()` and `longTaskTimers()` | Stable |
| `RequiredSearch` | Finds Counter, Gauge, Timer, DistributionSummary | +`longTaskTimer()` and `longTaskTimers()` | Stable |

---

## 7. Connection to Real Framework

| Simplified Class | Real Framework Class | Key Simplification |
|-----------------|---------------------|--------------------|
| `api/LongTaskTimer` | `io.micrometer.core.instrument.LongTaskTimer` | No `HistogramSupport` interface, no distribution statistics, no `wrap()` methods, no `fluent()` builder |
| `api/LongTaskTimer.Sample` | `io.micrometer.core.instrument.LongTaskTimer.Sample` | Identical abstract design — same `stop()` and `duration()` pattern |
| `api/LongTaskTimer.Builder` | `io.micrometer.core.instrument.LongTaskTimer.Builder` | No `DistributionStatisticConfig`, no `minimumExpectedValue`/`maximumExpectedValue`, no `serviceLevelObjectives` |
| `internal/DefaultLongTaskTimer` | `io.micrometer.core.instrument.internal.DefaultLongTaskTimer` | Same `ConcurrentSkipListSet` approach with `SampleImpl`/`SampleImplCounted` collision handling. No `AbstractMeter` base class, no histogram recording |
| `internal/NoopLongTaskTimer` | `io.micrometer.core.instrument.noop.NoopLongTaskTimer` | No `NoopMeter` base class. Identical inert-sample approach |
| `api/MeterRegistry.More` | `io.micrometer.core.instrument.MeterRegistry.More` | Only `longTaskTimer()` — real version also has `counter()` (FunctionCounter), `timer()` (FunctionTimer), `timeGauge()`, and `gauge()` for function-based meters |

---

## 8. Complete Code

All files created or modified in this feature. Copy all `[NEW]` files and replace
all `[MODIFIED]` files to build the project incrementally.

### `src/main/java/simple/micrometer/api/LongTaskTimer.java` [NEW]

```java
package simple.micrometer.api;

import java.util.Arrays;
import java.util.concurrent.Callable;
import java.util.concurrent.TimeUnit;
import java.util.function.Supplier;

/**
 * A meter that tracks the duration of <em>in-flight</em> long-running operations.
 * Unlike {@link Timer}, which records durations of completed events,
 * {@code LongTaskTimer} reports on tasks that haven't finished yet —
 * how many are active, how long they've been running, and which is the longest.
 *
 * <p>Typical use cases: database migrations, batch processing jobs, large file
 * transfers, or any operation where you need alerts on work that is still
 * in progress rather than historical latency.
 *
 * <p>Usage:
 * <pre>{@code
 * LongTaskTimer longTimer = LongTaskTimer.builder("data.migration")
 *     .tag("table", "users")
 *     .register(registry);
 *
 * // Start tracking a long-running task
 * LongTaskTimer.Sample task = longTimer.start();
 *
 * // While the task is running, observe:
 * int active = longTimer.activeTasks();        // 1
 * double elapsed = longTimer.duration(TimeUnit.SECONDS);
 * double longest = longTimer.max(TimeUnit.SECONDS);
 *
 * // When done:
 * long durationNanos = task.stop();  // active tasks → 0
 * }</pre>
 *
 * <p>Simplified from {@code io.micrometer.core.instrument.LongTaskTimer}.
 */
public interface LongTaskTimer extends Meter {

    // ── Core operations ──────────────────────────────────────────────

    /**
     * Begins timing a task. The returned {@link Sample} handle is used to
     * stop the task when work completes. The task is counted as "active"
     * until {@link Sample#stop()} is called.
     *
     * @return a sample representing the in-flight task
     */
    Sample start();

    /**
     * Returns the total cumulative duration of all currently active tasks,
     * converted to the given time unit.
     */
    double duration(TimeUnit unit);

    /**
     * Returns the number of currently active (in-flight) tasks.
     */
    int activeTasks();

    /**
     * Returns the duration of the longest-running active task, converted
     * to the given time unit. Returns 0 if no tasks are active.
     */
    double max(TimeUnit unit);

    // ── Default query methods ────────────────────────────────────────

    /**
     * Returns the mean duration of active tasks (duration / activeTasks),
     * or 0 if no tasks are active.
     */
    default double mean(TimeUnit unit) {
        int active = activeTasks();
        return active == 0 ? 0.0 : duration(unit) / active;
    }

    // ── Measurement ──────────────────────────────────────────────────

    /**
     * Returns three measurements: ACTIVE_TASKS, DURATION (in nanoseconds),
     * and MAX (in nanoseconds). These are the statistics monitoring backends
     * use for LongTaskTimer exposition.
     */
    @Override
    default Iterable<Measurement> measure() {
        return Arrays.asList(
                new Measurement(() -> (double) activeTasks(), Statistic.ACTIVE_TASKS),
                new Measurement(() -> duration(TimeUnit.NANOSECONDS), Statistic.DURATION),
                new Measurement(() -> max(TimeUnit.NANOSECONDS), Statistic.MAX)
        );
    }

    // ── Convenience recording methods ────────────────────────────────

    /**
     * Wraps the given runnable: starts a task, executes, then stops.
     * The task duration is recorded even if the runnable throws.
     */
    default void record(Runnable f) {
        Sample sample = start();
        try {
            f.run();
        } finally {
            sample.stop();
        }
    }

    /**
     * Wraps the given supplier: starts a task, executes, stops, and
     * returns the result. The task duration is recorded even if the
     * supplier throws.
     */
    default <T> T record(Supplier<T> f) {
        Sample sample = start();
        try {
            return f.get();
        } finally {
            sample.stop();
        }
    }

    /**
     * Wraps the given callable: starts a task, executes, stops, and
     * returns the result. Propagates checked exceptions.
     */
    default <T> T recordCallable(Callable<T> f) throws Exception {
        Sample sample = start();
        try {
            return f.call();
        } finally {
            sample.stop();
        }
    }

    // ── Factory ──────────────────────────────────────────────────────

    /**
     * Creates a new LongTaskTimer builder.
     */
    static Builder builder(String name) {
        return new Builder(name);
    }

    // ── Sample (task handle) ─────────────────────────────────────────

    /**
     * Represents a single in-flight task being tracked by a {@link LongTaskTimer}.
     * Each call to {@link LongTaskTimer#start()} returns a new Sample. The
     * caller holds this handle and calls {@link #stop()} when the work completes.
     *
     * <p>Unlike {@link Timer.Sample} which is a stopwatch that records onto a
     * Timer chosen at stop-time, this Sample is already bound to a specific
     * LongTaskTimer and actively participates in its active-task accounting.
     *
     * <p>Simplified from {@code io.micrometer.core.instrument.LongTaskTimer.Sample}.
     */
    abstract class Sample {

        /**
         * Stops timing this task and removes it from the active set.
         *
         * @return the task's total duration in nanoseconds
         */
        public abstract long stop();

        /**
         * Returns the current elapsed duration of this task (time since
         * {@link LongTaskTimer#start()} was called), converted to the
         * given unit. Returns {@code -1} if the task has already been stopped.
         */
        public abstract double duration(TimeUnit unit);
    }

    // ── Builder ──────────────────────────────────────────────────────

    /**
     * Fluent builder for constructing and registering long task timers.
     * Follows the same pattern as {@link Timer.Builder}: accumulates
     * metadata, then delegates to the registry for concrete creation.
     *
     * <p>Unlike regular Timer builders, LongTaskTimer builders do not
     * support distribution statistics in this simplified version.
     *
     * <p>Simplified from {@code io.micrometer.core.instrument.LongTaskTimer.Builder}.
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
         * Terminal operation: registers (or retrieves) a long task timer
         * with the given registry. Uses the {@link MeterRegistry.More}
         * accessor because LongTaskTimer is a less common meter type.
         */
        public LongTaskTimer register(MeterRegistry registry) {
            Meter.Id id = new Meter.Id(name, tags, null, description, Meter.Type.LONG_TASK_TIMER);
            return registry.more().longTaskTimer(id);
        }
    }

}
```

### `src/main/java/simple/micrometer/internal/DefaultLongTaskTimer.java` [NEW]

```java
package simple.micrometer.internal;

import simple.micrometer.api.Clock;
import simple.micrometer.api.LongTaskTimer;
import simple.micrometer.api.Meter;

import java.util.NoSuchElementException;
import java.util.concurrent.ConcurrentSkipListSet;
import java.util.concurrent.TimeUnit;
import java.util.concurrent.atomic.AtomicInteger;

/**
 * Default implementation of {@link LongTaskTimer} that tracks in-flight tasks
 * using a {@link ConcurrentSkipListSet} ordered by start time.
 *
 * <h3>Data structure choice</h3>
 * Tasks are added at {@link #start()} and removed at {@link Sample#stop()}.
 * Removal is out-of-order (tasks don't complete in FIFO order), so a queue
 * would be O(N) for removal. A skip list gives O(log N) for both insert and
 * removal, plus natural ordering by start time means {@link #max(TimeUnit)}
 * is O(1) — the oldest task (smallest start time = longest duration) is
 * always {@code first()} in the set.
 *
 * <h3>Collision handling</h3>
 * Since the set requires unique elements via {@code compareTo}, two tasks
 * starting at the exact same nanosecond would collide. The first task uses
 * counter=0; if that collides, a disambiguating counter is assigned via
 * {@link AtomicInteger}.
 *
 * <p>Simplified from {@code io.micrometer.core.instrument.internal.DefaultLongTaskTimer}.
 */
public class DefaultLongTaskTimer implements LongTaskTimer {

    private final Meter.Id id;
    private final Clock clock;

    /**
     * Active tasks ordered by start time (ascending). The first element
     * is always the oldest = longest-running task.
     */
    private final ConcurrentSkipListSet<SampleImpl> activeTasks = new ConcurrentSkipListSet<>();

    /**
     * Monotonic counter for disambiguating tasks that start at the exact
     * same nanosecond.
     */
    private final AtomicInteger counter = new AtomicInteger();

    public DefaultLongTaskTimer(Meter.Id id, Clock clock) {
        this.id = id;
        this.clock = clock;
    }

    // ── LongTaskTimer interface ──────────────────────────────────────

    @Override
    public Sample start() {
        long startTime = clock.monotonicTime();
        SampleImpl sample = new SampleImpl(startTime);
        if (!activeTasks.add(sample)) {
            // Collision: two tasks started at the exact same nanosecond.
            // Create a new sample with a unique non-zero counter.
            sample = new SampleImplCounted(startTime, nextNonZeroCounter());
            activeTasks.add(sample);
        }
        return sample;
    }

    @Override
    public double duration(TimeUnit unit) {
        long now = clock.monotonicTime();
        double totalNanos = 0.0;
        for (SampleImpl task : activeTasks) {
            totalNanos += (now - task.startTime);
        }
        return nanosToUnit(totalNanos, unit);
    }

    @Override
    public int activeTasks() {
        return activeTasks.size();
    }

    @Override
    public double max(TimeUnit unit) {
        try {
            // first() = oldest start time = longest-running task
            return activeTasks.first().duration(unit);
        } catch (NoSuchElementException e) {
            return 0.0;
        }
    }

    // ── Meter interface ──────────────────────────────────────────────

    @Override
    public Id getId() {
        return id;
    }

    // ── Internal helpers ─────────────────────────────────────────────

    /**
     * Returns the next non-zero counter value. Counter=0 is reserved for
     * the base {@link SampleImpl} (no collision), so disambiguating samples
     * always get a non-zero value.
     */
    private int nextNonZeroCounter() {
        int next;
        while ((next = counter.incrementAndGet()) == 0) {
            // extremely unlikely: counter wrapped around to exactly 0
        }
        return next;
    }

    private static double nanosToUnit(double nanos, TimeUnit unit) {
        return nanos / TimeUnit.NANOSECONDS.convert(1, unit);
    }

    // ── SampleImpl (base case: counter=0) ────────────────────────────

    /**
     * A sample representing a single in-flight task. Ordered by start time
     * for the {@link ConcurrentSkipListSet}. The base implementation has
     * counter=0; the {@link SampleImplCounted} subclass provides non-zero
     * counters for collision disambiguation.
     */
    private class SampleImpl extends LongTaskTimer.Sample implements Comparable<SampleImpl> {

        final long startTime;
        private volatile boolean stopped;

        SampleImpl(long startTime) {
            this.startTime = startTime;
        }

        @Override
        public long stop() {
            activeTasks.remove(this);
            long durationNanos = (long) duration(TimeUnit.NANOSECONDS);
            stopped = true;
            return durationNanos;
        }

        @Override
        public double duration(TimeUnit unit) {
            return stopped ? -1 : nanosToUnit(clock.monotonicTime() - startTime, unit);
        }

        /**
         * Subclass hook for collision counter. Base case returns 0.
         */
        int counter() {
            return 0;
        }

        @Override
        public int compareTo(SampleImpl that) {
            if (this == that) return 0;
            int cmp = Long.compare(this.startTime, that.startTime);
            if (cmp != 0) return cmp;
            return Integer.compare(this.counter(), that.counter());
        }

        @Override
        public boolean equals(Object o) {
            if (this == o) return true;
            if (!(o instanceof SampleImpl that)) return false;
            return this.compareTo(that) == 0;
        }

        @Override
        public int hashCode() {
            return Long.hashCode(startTime) * 31 + counter();
        }
    }

    // ── SampleImplCounted (collision case: non-zero counter) ─────────

    /**
     * A sample with a non-zero counter, used only when two tasks start at
     * the exact same nanosecond. The counter ensures uniqueness in the
     * {@link ConcurrentSkipListSet}.
     */
    private class SampleImplCounted extends SampleImpl {

        private final int counterValue;

        SampleImplCounted(long startTime, int counterValue) {
            super(startTime);
            this.counterValue = counterValue;
        }

        @Override
        int counter() {
            return counterValue;
        }
    }

}
```

### `src/main/java/simple/micrometer/internal/NoopLongTaskTimer.java` [NEW]

```java
package simple.micrometer.internal;

import simple.micrometer.api.LongTaskTimer;
import simple.micrometer.api.Meter;

import java.util.concurrent.TimeUnit;

/**
 * A no-op LongTaskTimer returned when a meter is denied by a filter.
 * The {@link #start()} method still returns a usable (but inert) Sample,
 * so callers never need null checks. Convenience recording methods
 * ({@link #record(Runnable)}, etc.) still execute the provided function —
 * only the tracking is discarded.
 *
 * <p>Simplified from {@code io.micrometer.core.instrument.noop.NoopLongTaskTimer}.
 */
public class NoopLongTaskTimer implements LongTaskTimer {

    private final Meter.Id id;

    public NoopLongTaskTimer(Meter.Id id) {
        this.id = id;
    }

    @Override
    public Sample start() {
        return new Sample() {
            @Override
            public long stop() {
                return 0;
            }

            @Override
            public double duration(TimeUnit unit) {
                return 0;
            }
        };
    }

    @Override
    public double duration(TimeUnit unit) {
        return 0;
    }

    @Override
    public int activeTasks() {
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
import simple.micrometer.internal.NoopDistributionSummary;
import simple.micrometer.internal.NoopGauge;
import simple.micrometer.internal.NoopLongTaskTimer;
import simple.micrometer.internal.NoopTimer;

import java.util.ArrayList;
import java.util.Collections;
import java.util.List;
import java.util.Objects;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.CopyOnWriteArrayList;
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
 * <p>Before a meter is created, any registered {@link MeterFilter}s are applied:
 * first {@link MeterFilter#map(Meter.Id)} transforms the identity (adding common
 * tags, renaming, etc.), then {@link MeterFilter#accept(Meter.Id)} decides whether
 * the meter should be real or a noop.
 *
 * <p>Simplified from {@code io.micrometer.core.instrument.MeterRegistry}.
 */
public abstract class MeterRegistry {

    private final ConcurrentHashMap<Meter.Id, Meter> meterMap = new ConcurrentHashMap<>();
    private final Object meterMapLock = new Object();
    private final List<MeterFilter> filters = new CopyOnWriteArrayList<>();
    private volatile NamingConvention namingConvention = NamingConvention.dot;
    private final Config config = new Config();
    private final More more = new More();

    protected final Clock clock;

    protected MeterRegistry(Clock clock) {
        this.clock = Objects.requireNonNull(clock, "clock must not be null");
    }

    // ── Configuration ─────────────────────────────────────────────────

    /**
     * Returns this registry's configuration handle, through which
     * {@link MeterFilter}s and common tags can be added.
     */
    public Config config() {
        return config;
    }

    /**
     * Returns this registry's {@link More} handle, through which less
     * common meter types (like {@link LongTaskTimer}) can be registered.
     */
    public More more() {
        return more;
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
        return registerMeterIfNecessary(Counter.class, id, this::newCounter, NoopCounter::new);
    }

    /**
     * Called by {@link Gauge.Builder#register} and the convenience methods above.
     */
    <T> Gauge gauge(Meter.Id id, T obj, ToDoubleFunction<T> valueFunction) {
        return registerMeterIfNecessary(Gauge.class, id,
                i -> newGauge(i, obj, valueFunction), NoopGauge::new);
    }

    /**
     * Called by convenience methods (no distribution config).
     */
    Timer timer(Meter.Id id) {
        return timer(id, DistributionStatisticConfig.NONE);
    }

    /**
     * Called by {@link Timer.Builder#register} to pass the distribution config.
     */
    Timer timer(Meter.Id id, DistributionStatisticConfig distributionConfig) {
        return registerMeterIfNecessary(Timer.class, id,
                i -> this.newTimer(i, distributionConfig), NoopTimer::new);
    }

    /**
     * Called by convenience methods (no distribution config).
     */
    DistributionSummary summary(Meter.Id id) {
        return summary(id, DistributionStatisticConfig.NONE);
    }

    /**
     * Called by {@link DistributionSummary.Builder#register} to pass the distribution config.
     */
    DistributionSummary summary(Meter.Id id, DistributionStatisticConfig distributionConfig) {
        return registerMeterIfNecessary(DistributionSummary.class, id,
                i -> this.newDistributionSummary(i, distributionConfig), NoopDistributionSummary::new);
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
     * Creates a new Timer implementation for the given id and distribution config.
     * Called once per unique meter id, inside a synchronized block.
     *
     * @param id                 the timer's identity
     * @param distributionConfig histogram/percentile configuration
     */
    protected abstract Timer newTimer(Meter.Id id, DistributionStatisticConfig distributionConfig);

    /**
     * Creates a new DistributionSummary implementation for the given id and distribution config.
     * Called once per unique meter id, inside a synchronized block.
     *
     * @param id                 the summary's identity
     * @param distributionConfig histogram/percentile configuration
     */
    protected abstract DistributionSummary newDistributionSummary(Meter.Id id, DistributionStatisticConfig distributionConfig);

    /**
     * Creates a new LongTaskTimer implementation for the given id.
     * Called once per unique meter id, inside a synchronized block.
     *
     * @param id the long task timer's identity
     */
    protected abstract LongTaskTimer newLongTaskTimer(Meter.Id id);

    // ── Query ────────────────────────────────────────────────────────

    /**
     * Returns an unmodifiable snapshot of all registered meters.
     */
    public List<Meter> getMeters() {
        return Collections.unmodifiableList(new ArrayList<>(meterMap.values()));
    }

    /**
     * Starts a lenient search for meters by name. Returns a {@link Search}
     * whose terminal methods return {@code null} or empty collections when
     * no meters match.
     *
     * <pre>{@code
     * Counter counter = registry.find("http.requests")
     *     .tag("method", "GET")
     *     .counter(); // null if not found
     * }</pre>
     */
    public Search find(String name) {
        return Search.in(this).name(name);
    }

    /**
     * Starts a required search for meters by name. Returns a
     * {@link RequiredSearch} whose terminal methods throw
     * {@link MeterNotFoundException} when no meters match.
     *
     * <pre>{@code
     * Counter counter = registry.get("http.requests")
     *     .tag("method", "GET")
     *     .counter(); // throws if not found
     * }</pre>
     */
    public RequiredSearch get(String name) {
        return RequiredSearch.in(this).name(name);
    }

    /**
     * Returns the clock used by this registry.
     */
    public Clock getClock() {
        return clock;
    }

    /**
     * Returns the naming convention configured for this registry.
     */
    public NamingConvention getNamingConvention() {
        return namingConvention;
    }

    // ── Core registration logic ──────────────────────────────────────

    /**
     * Applies the filter chain and then performs double-checked locking
     * registration. The flow is:
     * <ol>
     *   <li>Apply all {@link MeterFilter#map} transformations to the id</li>
     *   <li>Apply all {@link MeterFilter#accept} decisions — if denied,
     *       return a noop meter</li>
     *   <li>Look up the mapped id in the meter map</li>
     *   <li>If not found, create under lock via the meter factory</li>
     * </ol>
     *
     * @param meterClass    expected concrete type (for type-safety cast)
     * @param id            the meter's identity (pre-filter)
     * @param meterFactory  creates the concrete meter (e.g., {@code this::newCounter})
     * @param noopFactory   creates a noop meter for denied registrations
     * @return the registered (or pre-existing) meter, cast to M
     * @throws IllegalArgumentException if an existing meter has the same id
     *                                  but a different type
     */
    private <M extends Meter> M registerMeterIfNecessary(
            Class<M> meterClass, Meter.Id id,
            Function<Meter.Id, M> meterFactory,
            Function<Meter.Id, M> noopFactory) {

        // Step 1: Apply map() filters to transform the id
        Meter.Id mappedId = applyMapFilters(id);

        // Step 2: Apply accept() filters — denied meters get a noop
        if (!applyAcceptFilters(mappedId)) {
            return noopFactory.apply(mappedId);
        }

        // Step 3: Fast path — lock-free lookup with the mapped id
        Meter existing = meterMap.get(mappedId);
        if (existing != null) {
            return castOrThrow(meterClass, existing, mappedId);
        }

        // Step 4: Slow path — create under lock
        synchronized (meterMapLock) {
            existing = meterMap.get(mappedId);
            if (existing != null) {
                return castOrThrow(meterClass, existing, mappedId);
            }

            M newMeter = meterFactory.apply(mappedId);
            meterMap.put(mappedId, newMeter);
            return newMeter;
        }
    }

    /**
     * Runs every filter's {@link MeterFilter#map} in insertion order.
     * Each filter receives the output of the previous one.
     */
    private Meter.Id applyMapFilters(Meter.Id id) {
        Meter.Id mappedId = id;
        for (MeterFilter filter : filters) {
            mappedId = filter.map(mappedId);
        }
        return mappedId;
    }

    /**
     * Runs every filter's {@link MeterFilter#accept} in insertion order.
     * The first non-{@link MeterFilterReply#NEUTRAL} reply wins.
     * If all filters return NEUTRAL, the meter is accepted by default.
     *
     * @return true if the meter should be registered, false if denied
     */
    private boolean applyAcceptFilters(Meter.Id id) {
        for (MeterFilter filter : filters) {
            MeterFilterReply reply = filter.accept(id);
            if (reply == MeterFilterReply.DENY) {
                return false;
            }
            if (reply == MeterFilterReply.ACCEPT) {
                return true;
            }
            // NEUTRAL → continue to next filter
        }
        return true; // default: accept if all filters are neutral
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

    // ── Config inner class ──────────────────────────────────────────

    /**
     * Configuration handle for a {@link MeterRegistry}. Provides methods
     * to add {@link MeterFilter}s and common tags.
     *
     * <pre>{@code
     * registry.config()
     *     .commonTags("app", "order-service", "env", "prod")
     *     .meterFilter(MeterFilter.denyNameStartsWith("jvm.gc"));
     * }</pre>
     */
    public class Config {

        /**
         * Adds a meter filter to this registry. Filters are applied in
         * insertion order during meter registration.
         *
         * @param filter the filter to add
         * @return this Config for chaining
         */
        public Config meterFilter(MeterFilter filter) {
            filters.add(Objects.requireNonNull(filter, "filter must not be null"));
            return this;
        }

        /**
         * Adds common tags that will be applied to every meter registered
         * with this registry. Meter-specific tags with the same key take
         * precedence over common tags.
         *
         * <p>This is a shorthand for
         * {@code meterFilter(MeterFilter.commonTags(Tags.of(tags)))}.
         *
         * @param tags alternating key-value strings
         * @return this Config for chaining
         */
        public Config commonTags(String... tags) {
            return meterFilter(MeterFilter.commonTags(Tags.of(tags)));
        }

        /**
         * Adds common tags that will be applied to every meter registered
         * with this registry.
         *
         * @param tags the common tags to add
         * @return this Config for chaining
         */
        public Config commonTags(Iterable<Tag> tags) {
            return meterFilter(MeterFilter.commonTags(tags));
        }

        /**
         * Sets the naming convention used by this registry to format meter
         * names and tag keys for a specific monitoring backend.
         *
         * @param convention the naming convention to use
         * @return this Config for chaining
         */
        public Config namingConvention(NamingConvention convention) {
            namingConvention = Objects.requireNonNull(convention, "convention must not be null");
            return this;
        }

    }

    // ── More inner class ─────────────────────────────────────────────

    /**
     * Provides registration methods for less common meter types.
     * While the primary types (Counter, Gauge, Timer, DistributionSummary)
     * have top-level convenience methods on {@link MeterRegistry}, secondary
     * types like {@link LongTaskTimer} are registered through this handle.
     *
     * <pre>{@code
     * LongTaskTimer ltt = registry.more().longTaskTimer("data.migration", "table", "users");
     * }</pre>
     *
     * <p>Simplified from {@code io.micrometer.core.instrument.MeterRegistry.More}.
     */
    public class More {

        /**
         * Creates or retrieves a long task timer with the given name and
         * alternating key-value tag strings.
         */
        public LongTaskTimer longTaskTimer(String name, String... tags) {
            return longTaskTimer(name, Tags.of(tags));
        }

        /**
         * Creates or retrieves a long task timer with the given name and tags.
         */
        public LongTaskTimer longTaskTimer(String name, Iterable<Tag> tags) {
            return longTaskTimer(new Meter.Id(name, Tags.of(tags), null, null, Meter.Type.LONG_TASK_TIMER));
        }

        /**
         * Package-private entry point called by {@link LongTaskTimer.Builder#register}
         * and the convenience methods above.
         */
        LongTaskTimer longTaskTimer(Meter.Id id) {
            return registerMeterIfNecessary(LongTaskTimer.class, id,
                    MeterRegistry.this::newLongTaskTimer, NoopLongTaskTimer::new);
        }
    }

}
```

### `src/main/java/simple/micrometer/internal/SimpleMeterRegistry.java` [MODIFIED]

```java
package simple.micrometer.internal;

import simple.micrometer.api.Clock;
import simple.micrometer.api.Counter;
import simple.micrometer.api.DistributionStatisticConfig;
import simple.micrometer.api.DistributionSummary;
import simple.micrometer.api.Gauge;
import simple.micrometer.api.LongTaskTimer;
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
    protected Timer newTimer(Meter.Id id, DistributionStatisticConfig distributionConfig) {
        return new CumulativeTimer(id, clock, distributionConfig);
    }

    @Override
    protected DistributionSummary newDistributionSummary(Meter.Id id, DistributionStatisticConfig distributionConfig) {
        return new CumulativeDistributionSummary(id, clock, distributionConfig);
    }

    @Override
    protected LongTaskTimer newLongTaskTimer(Meter.Id id) {
        return new DefaultLongTaskTimer(id, clock);
    }

}
```

### `src/main/java/simple/micrometer/api/CompositeMeterRegistry.java` [MODIFIED]

```java
package simple.micrometer.api;

import simple.micrometer.internal.CompositeMeter;
import simple.micrometer.internal.CompositeCounter;
import simple.micrometer.internal.CompositeDistributionSummary;
import simple.micrometer.internal.CompositeGauge;
import simple.micrometer.internal.CompositeTimer;
import simple.micrometer.internal.DefaultLongTaskTimer;

import java.util.Collections;
import java.util.LinkedHashSet;
import java.util.Set;
import java.util.concurrent.CopyOnWriteArrayList;
import java.util.function.ToDoubleFunction;

/**
 * A meter registry that delegates to multiple child registries, enabling
 * metrics to be published to multiple monitoring backends simultaneously.
 *
 * <p>Usage:
 * <pre>{@code
 * SimpleMeterRegistry simple = new SimpleMeterRegistry();
 * // PrometheusRegistry prometheus = new PrometheusRegistry();
 *
 * CompositeMeterRegistry composite = new CompositeMeterRegistry();
 * composite.add(simple);
 * // composite.add(prometheus);
 *
 * // Meters registered on composite are mirrored to all children
 * Counter counter = composite.counter("http.requests", "method", "GET");
 * counter.increment();
 *
 * // All children see the data
 * simple.counter("http.requests", "method", "GET").count(); // 1.0
 * }</pre>
 *
 * <p><b>How it works:</b> When a meter is created on this registry, it creates
 * a <em>composite meter</em> (e.g., {@link CompositeCounter}) that wraps
 * concrete meters from each child. Write operations (increment, record) fan
 * out to all children. Read operations (count, value) delegate to the first
 * child.
 *
 * <p><b>Dynamic children:</b> Child registries can be added or removed at any
 * time. Adding a child wires all existing composite meters to create concrete
 * meters in the new child. Removing a child unwires them.
 *
 * <p><b>No children:</b> If no child registries are present, composite meters
 * delegate to noop implementations — operations silently do nothing.
 *
 * <p>Simplified from {@code io.micrometer.core.instrument.composite.CompositeMeterRegistry}.
 * The real version supports nested composites (flattening), parent tracking,
 * and CAS spinlocks for higher concurrency.
 */
public class CompositeMeterRegistry extends MeterRegistry {

    /**
     * Thread-safe list of child registries. Uses {@link CopyOnWriteArrayList}
     * so that the {@code newXxx()} factory methods can safely iterate without
     * holding a lock, while {@link #add}/{@link #remove} mutate atomically.
     */
    private final CopyOnWriteArrayList<MeterRegistry> childRegistries = new CopyOnWriteArrayList<>();

    public CompositeMeterRegistry() {
        this(Clock.SYSTEM);
    }

    public CompositeMeterRegistry(Clock clock) {
        super(clock);
    }

    // ── Child registry management ──────────────────────────────────────

    /**
     * Adds a child registry. All existing composite meters in this registry
     * will create concrete meters in the new child. Future meters will
     * automatically include this child.
     *
     * @throws IllegalArgumentException if the registry is this composite itself
     */
    public CompositeMeterRegistry add(MeterRegistry registry) {
        if (registry == this) {
            throw new IllegalArgumentException(
                    "Cannot add a CompositeMeterRegistry to itself");
        }
        if (childRegistries.addIfAbsent(registry)) {
            // Wire all existing composite meters to the new child
            for (Meter meter : getMeters()) {
                if (meter instanceof CompositeMeter cm) {
                    cm.add(registry);
                }
            }
        }
        return this;
    }

    /**
     * Removes a child registry. All existing composite meters will stop
     * delegating to it.
     */
    public CompositeMeterRegistry remove(MeterRegistry registry) {
        if (childRegistries.remove(registry)) {
            // Unwire all existing composite meters from the removed child
            for (Meter meter : getMeters()) {
                if (meter instanceof CompositeMeter cm) {
                    cm.remove(registry);
                }
            }
        }
        return this;
    }

    /**
     * Returns an unmodifiable view of the current child registries.
     */
    public Set<MeterRegistry> getRegistries() {
        return Collections.unmodifiableSet(new LinkedHashSet<>(childRegistries));
    }

    // ── Factory methods: create composite meters ───────────────────────

    @Override
    protected Counter newCounter(Meter.Id id) {
        CompositeCounter counter = new CompositeCounter(id);
        for (MeterRegistry child : childRegistries) {
            counter.add(child);
        }
        return counter;
    }

    @Override
    protected <T> Gauge newGauge(Meter.Id id, T obj, ToDoubleFunction<T> valueFunction) {
        CompositeGauge<T> gauge = new CompositeGauge<>(id, obj, valueFunction);
        for (MeterRegistry child : childRegistries) {
            gauge.add(child);
        }
        return gauge;
    }

    @Override
    protected Timer newTimer(Meter.Id id, DistributionStatisticConfig distributionConfig) {
        CompositeTimer timer = new CompositeTimer(id, clock);
        for (MeterRegistry child : childRegistries) {
            timer.add(child);
        }
        return timer;
    }

    @Override
    protected DistributionSummary newDistributionSummary(Meter.Id id, DistributionStatisticConfig distributionConfig) {
        CompositeDistributionSummary summary = new CompositeDistributionSummary(id);
        for (MeterRegistry child : childRegistries) {
            summary.add(child);
        }
        return summary;
    }

    /**
     * Creates a {@link DefaultLongTaskTimer} for the composite registry.
     * Unlike other meter types, LongTaskTimer tracks inherently local
     * active-task state, so a composite version is not needed — a single
     * DefaultLongTaskTimer suffices.
     */
    @Override
    protected LongTaskTimer newLongTaskTimer(Meter.Id id) {
        return new DefaultLongTaskTimer(id, clock);
    }

}
```

### `src/main/java/simple/micrometer/api/Search.java` [MODIFIED]

```java
package simple.micrometer.api;

import java.util.ArrayList;
import java.util.Collection;
import java.util.HashSet;
import java.util.List;
import java.util.Set;
import java.util.stream.Collectors;
import java.util.stream.Stream;

/**
 * A fluent query builder for finding meters in a {@link MeterRegistry}.
 * Returns {@code null} for single-meter lookups or empty collections when
 * no meters match — use {@link RequiredSearch} when a match is mandatory.
 *
 * <p>Usage:
 * <pre>{@code
 * // Find a counter by name and tag
 * Counter counter = registry.find("http.requests")
 *     .tag("method", "GET")
 *     .counter();
 *
 * // Get all meters matching a name
 * Collection<Meter> meters = registry.find("http.requests").meters();
 * }</pre>
 *
 * <p>The search operates by streaming {@link MeterRegistry#getMeters()} and
 * applying name, tag, and tag-key filters. This is a linear scan — there is
 * no secondary index.
 *
 * <p>Simplified from {@code io.micrometer.core.instrument.search.Search}.
 * The real framework also supports predicate-based name matching,
 * predicate-based tag value matching, and conversion to a {@link MeterFilter}.
 */
public final class Search {

    private final MeterRegistry registry;
    private String exactName;
    private final List<Tag> requiredTags = new ArrayList<>();
    private final Set<String> requiredTagKeys = new HashSet<>();

    private Search(MeterRegistry registry) {
        this.registry = registry;
    }

    // ── Static entry point ─────────────────────────────────────────────

    /**
     * Creates a new search scoped to the given registry.
     */
    public static Search in(MeterRegistry registry) {
        return new Search(registry);
    }

    // ── Fluent filter methods ──────────────────────────────────────────

    /**
     * Restricts the search to meters whose name exactly equals the given string.
     */
    public Search name(String exactName) {
        this.exactName = exactName;
        return this;
    }

    /**
     * Requires meters to have a tag with the given key and value.
     */
    public Search tag(String tagKey, String tagValue) {
        this.requiredTags.add(Tag.of(tagKey, tagValue));
        return this;
    }

    /**
     * Requires meters to have all tags specified as alternating key-value strings.
     *
     * @throws IllegalArgumentException if the array has odd length
     */
    public Search tags(String... keyValues) {
        if (keyValues.length % 2 != 0) {
            throw new IllegalArgumentException("keyValues must have even length, got " + keyValues.length);
        }
        for (int i = 0; i < keyValues.length; i += 2) {
            this.requiredTags.add(Tag.of(keyValues[i], keyValues[i + 1]));
        }
        return this;
    }

    /**
     * Requires meters to have all of the given tags (exact key+value match).
     */
    public Search tags(Iterable<Tag> tags) {
        for (Tag tag : tags) {
            this.requiredTags.add(tag);
        }
        return this;
    }

    /**
     * Requires meters to have tags with the given keys (values are unconstrained).
     */
    public Search tagKeys(String... tagKeys) {
        for (String key : tagKeys) {
            this.requiredTagKeys.add(key);
        }
        return this;
    }

    // ── Terminal: single-meter finders (return null on miss) ───────────

    /**
     * Finds a single {@link Counter} matching the search criteria, or
     * {@code null} if none matches.
     */
    public Counter counter() {
        return findOne(Counter.class);
    }

    /**
     * Finds a single {@link Gauge} matching the search criteria, or
     * {@code null} if none matches.
     */
    public Gauge gauge() {
        return findOne(Gauge.class);
    }

    /**
     * Finds a single {@link Timer} matching the search criteria, or
     * {@code null} if none matches.
     */
    public Timer timer() {
        return findOne(Timer.class);
    }

    /**
     * Finds a single {@link DistributionSummary} matching the search criteria,
     * or {@code null} if none matches.
     */
    public DistributionSummary summary() {
        return findOne(DistributionSummary.class);
    }

    /**
     * Finds a single {@link LongTaskTimer} matching the search criteria,
     * or {@code null} if none matches.
     */
    public LongTaskTimer longTaskTimer() {
        return findOne(LongTaskTimer.class);
    }

    /**
     * Finds a single {@link Meter} of any type matching the search criteria,
     * or {@code null} if none matches.
     */
    public Meter meter() {
        return meterStream().findAny().orElse(null);
    }

    // ── Terminal: collection finders (return empty on miss) ────────────

    /**
     * Returns all meters matching the search criteria, or an empty collection.
     */
    public Collection<Meter> meters() {
        return meterStream().collect(Collectors.toList());
    }

    /**
     * Returns all counters matching the search criteria, or an empty collection.
     */
    public Collection<Counter> counters() {
        return findAll(Counter.class);
    }

    /**
     * Returns all gauges matching the search criteria, or an empty collection.
     */
    public Collection<Gauge> gauges() {
        return findAll(Gauge.class);
    }

    /**
     * Returns all timers matching the search criteria, or an empty collection.
     */
    public Collection<Timer> timers() {
        return findAll(Timer.class);
    }

    /**
     * Returns all distribution summaries matching the search criteria, or
     * an empty collection.
     */
    public Collection<DistributionSummary> summaries() {
        return findAll(DistributionSummary.class);
    }

    /**
     * Returns all long task timers matching the search criteria, or
     * an empty collection.
     */
    public Collection<LongTaskTimer> longTaskTimers() {
        return findAll(LongTaskTimer.class);
    }

    // ── Internal filtering ─────────────────────────────────────────────

    private <M extends Meter> M findOne(Class<M> clazz) {
        return meterStream()
                .filter(clazz::isInstance)
                .findAny()
                .map(clazz::cast)
                .orElse(null);
    }

    private <M extends Meter> Collection<M> findAll(Class<M> clazz) {
        return meterStream()
                .filter(clazz::isInstance)
                .map(clazz::cast)
                .collect(Collectors.toList());
    }

    /**
     * The core search pipeline: stream all meters, filter by name, then by tags.
     */
    private Stream<Meter> meterStream() {
        Stream<Meter> stream = registry.getMeters().stream();

        // Filter by name (exact match)
        if (exactName != null) {
            stream = stream.filter(m -> exactName.equals(m.getId().getName()));
        }

        // Filter by required tags (exact key+value pairs)
        if (!requiredTags.isEmpty()) {
            stream = stream.filter(m -> {
                List<Tag> meterTags = m.getId().getTags();
                return meterTags.containsAll(requiredTags);
            });
        }

        // Filter by required tag keys (key must exist, value unconstrained)
        if (!requiredTagKeys.isEmpty()) {
            stream = stream.filter(m -> {
                Set<String> meterTagKeys = new HashSet<>();
                for (Tag tag : m.getId().getTagsAsIterable()) {
                    meterTagKeys.add(tag.getKey());
                }
                return meterTagKeys.containsAll(requiredTagKeys);
            });
        }

        return stream;
    }

}
```

### `src/main/java/simple/micrometer/api/RequiredSearch.java` [MODIFIED]

```java
package simple.micrometer.api;

import java.util.ArrayList;
import java.util.Collection;
import java.util.HashSet;
import java.util.List;
import java.util.Set;
import java.util.stream.Collectors;
import java.util.stream.Stream;

/**
 * A fluent query builder for finding meters in a {@link MeterRegistry} that
 * <b>throws {@link MeterNotFoundException}</b> when no meter matches. Use
 * this in tests or anywhere a meter is expected to exist.
 *
 * <p>Usage:
 * <pre>{@code
 * // Throws MeterNotFoundException if not found
 * Counter counter = registry.get("http.requests")
 *     .tag("method", "GET")
 *     .counter();
 *
 * // All matching meters (throws if empty)
 * Collection<Meter> meters = registry.get("http.requests").meters();
 * }</pre>
 *
 * <p>This is an independent implementation parallel to {@link Search} — it
 * does NOT wrap or delegate to Search. The difference is purely in the terminal
 * operations: Search returns null/empty, RequiredSearch throws.
 *
 * <p>Simplified from {@code io.micrometer.core.instrument.search.RequiredSearch}.
 */
public final class RequiredSearch {

    private final MeterRegistry registry;
    private String exactName;
    private final List<Tag> requiredTags = new ArrayList<>();
    private final Set<String> requiredTagKeys = new HashSet<>();

    private RequiredSearch(MeterRegistry registry) {
        this.registry = registry;
    }

    // ── Static entry point ─────────────────────────────────────────────

    /**
     * Creates a new required search scoped to the given registry.
     */
    public static RequiredSearch in(MeterRegistry registry) {
        return new RequiredSearch(registry);
    }

    // ── Fluent filter methods ──────────────────────────────────────────

    /**
     * Restricts the search to meters whose name exactly equals the given string.
     */
    public RequiredSearch name(String exactName) {
        this.exactName = exactName;
        return this;
    }

    /**
     * Requires meters to have a tag with the given key and value.
     */
    public RequiredSearch tag(String tagKey, String tagValue) {
        this.requiredTags.add(Tag.of(tagKey, tagValue));
        return this;
    }

    /**
     * Requires meters to have all tags specified as alternating key-value strings.
     *
     * @throws IllegalArgumentException if the array has odd length
     */
    public RequiredSearch tags(String... keyValues) {
        if (keyValues.length % 2 != 0) {
            throw new IllegalArgumentException("keyValues must have even length, got " + keyValues.length);
        }
        for (int i = 0; i < keyValues.length; i += 2) {
            this.requiredTags.add(Tag.of(keyValues[i], keyValues[i + 1]));
        }
        return this;
    }

    /**
     * Requires meters to have all of the given tags (exact key+value match).
     */
    public RequiredSearch tags(Iterable<Tag> tags) {
        for (Tag tag : tags) {
            this.requiredTags.add(tag);
        }
        return this;
    }

    /**
     * Requires meters to have tags with the given keys (values are unconstrained).
     */
    public RequiredSearch tagKeys(String... tagKeys) {
        for (String key : tagKeys) {
            this.requiredTagKeys.add(key);
        }
        return this;
    }

    // ── Terminal: single-meter finders (throw on miss) ─────────────────

    /**
     * Finds a single {@link Counter} matching the search criteria.
     *
     * @throws MeterNotFoundException if no matching counter is found
     */
    public Counter counter() {
        return getOne(Counter.class);
    }

    /**
     * Finds a single {@link Gauge} matching the search criteria.
     *
     * @throws MeterNotFoundException if no matching gauge is found
     */
    public Gauge gauge() {
        return getOne(Gauge.class);
    }

    /**
     * Finds a single {@link Timer} matching the search criteria.
     *
     * @throws MeterNotFoundException if no matching timer is found
     */
    public Timer timer() {
        return getOne(Timer.class);
    }

    /**
     * Finds a single {@link DistributionSummary} matching the search criteria.
     *
     * @throws MeterNotFoundException if no matching distribution summary is found
     */
    public DistributionSummary summary() {
        return getOne(DistributionSummary.class);
    }

    /**
     * Finds a single {@link LongTaskTimer} matching the search criteria.
     *
     * @throws MeterNotFoundException if no matching long task timer is found
     */
    public LongTaskTimer longTaskTimer() {
        return getOne(LongTaskTimer.class);
    }

    /**
     * Finds a single {@link Meter} of any type matching the search criteria.
     *
     * @throws MeterNotFoundException if no matching meter is found
     */
    public Meter meter() {
        return meterStream().findAny()
                .orElseThrow(() -> buildException(Meter.class));
    }

    // ── Terminal: collection finders (throw on empty) ──────────────────

    /**
     * Returns all meters matching the search criteria.
     *
     * @throws MeterNotFoundException if no meters match
     */
    public Collection<Meter> meters() {
        Collection<Meter> result = meterStream().collect(Collectors.toList());
        if (result.isEmpty()) {
            throw buildException(Meter.class);
        }
        return result;
    }

    /**
     * Returns all counters matching the search criteria.
     *
     * @throws MeterNotFoundException if no counters match
     */
    public Collection<Counter> counters() {
        return requireNonEmpty(findAll(Counter.class), Counter.class);
    }

    /**
     * Returns all gauges matching the search criteria.
     *
     * @throws MeterNotFoundException if no gauges match
     */
    public Collection<Gauge> gauges() {
        return requireNonEmpty(findAll(Gauge.class), Gauge.class);
    }

    /**
     * Returns all timers matching the search criteria.
     *
     * @throws MeterNotFoundException if no timers match
     */
    public Collection<Timer> timers() {
        return requireNonEmpty(findAll(Timer.class), Timer.class);
    }

    /**
     * Returns all distribution summaries matching the search criteria.
     *
     * @throws MeterNotFoundException if no distribution summaries match
     */
    public Collection<DistributionSummary> summaries() {
        return requireNonEmpty(findAll(DistributionSummary.class), DistributionSummary.class);
    }

    /**
     * Returns all long task timers matching the search criteria.
     *
     * @throws MeterNotFoundException if no long task timers match
     */
    public Collection<LongTaskTimer> longTaskTimers() {
        return requireNonEmpty(findAll(LongTaskTimer.class), LongTaskTimer.class);
    }

    // ── Internal filtering ─────────────────────────────────────────────

    private <M extends Meter> M getOne(Class<M> clazz) {
        return meterStream()
                .filter(clazz::isInstance)
                .findAny()
                .map(clazz::cast)
                .orElseThrow(() -> buildException(clazz));
    }

    private <M extends Meter> Collection<M> findAll(Class<M> clazz) {
        return meterStream()
                .filter(clazz::isInstance)
                .map(clazz::cast)
                .collect(Collectors.toList());
    }

    private <M extends Meter> Collection<M> requireNonEmpty(Collection<M> result, Class<M> clazz) {
        if (result.isEmpty()) {
            throw buildException(clazz);
        }
        return result;
    }

    private Stream<Meter> meterStream() {
        Stream<Meter> stream = registry.getMeters().stream();

        if (exactName != null) {
            stream = stream.filter(m -> exactName.equals(m.getId().getName()));
        }

        if (!requiredTags.isEmpty()) {
            stream = stream.filter(m -> {
                List<Tag> meterTags = m.getId().getTags();
                return meterTags.containsAll(requiredTags);
            });
        }

        if (!requiredTagKeys.isEmpty()) {
            stream = stream.filter(m -> {
                Set<String> meterTagKeys = new HashSet<>();
                for (Tag tag : m.getId().getTagsAsIterable()) {
                    meterTagKeys.add(tag.getKey());
                }
                return meterTagKeys.containsAll(requiredTagKeys);
            });
        }

        return stream;
    }

    /**
     * Builds a descriptive exception message showing what was searched for.
     */
    private MeterNotFoundException buildException(Class<? extends Meter> expectedType) {
        StringBuilder msg = new StringBuilder("No ");
        msg.append(expectedType.getSimpleName()).append(" found");

        if (exactName != null) {
            msg.append(" with name '").append(exactName).append("'");
        }

        if (!requiredTags.isEmpty()) {
            msg.append(" with tags ").append(requiredTags);
        }

        if (!requiredTagKeys.isEmpty()) {
            msg.append(" with tag keys ").append(requiredTagKeys);
        }

        msg.append(" in the registry. ");
        msg.append("Available meters: ");

        List<Meter> all = registry.getMeters();
        if (all.isEmpty()) {
            msg.append("(none)");
        } else {
            for (int i = 0; i < all.size(); i++) {
                if (i > 0) msg.append(", ");
                Meter.Id id = all.get(i).getId();
                msg.append(id.getName());
                List<Tag> tags = id.getTags();
                if (!tags.isEmpty()) {
                    msg.append(tags);
                }
            }
        }

        return new MeterNotFoundException(msg.toString());
    }

}
```

### `src/test/java/simple/micrometer/api/LongTaskTimerTest.java` [NEW]

```java
package simple.micrometer.api;

import simple.micrometer.internal.NoopLongTaskTimer;
import simple.micrometer.internal.SimpleMeterRegistry;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.util.concurrent.TimeUnit;
import java.util.concurrent.atomic.AtomicLong;
import java.util.concurrent.atomic.AtomicReference;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;
import static org.assertj.core.api.Assertions.within;

/**
 * Client-perspective tests for {@link LongTaskTimer}.
 * These tests define the API's behavior — they were written before
 * any implementation code.
 */
class LongTaskTimerTest {

    private MockClock clock;
    private MeterRegistry registry;

    @BeforeEach
    void setUp() {
        clock = new MockClock();
        registry = new SimpleMeterRegistry(clock);
    }

    // ── Builder + Registration ───────────────────────────────────────

    @Test
    void shouldCreateLongTaskTimer_WithBuilderAndRegister() {
        LongTaskTimer ltt = LongTaskTimer.builder("data.migration")
                .tag("table", "users")
                .description("Tracks active data migrations")
                .register(registry);

        assertThat(ltt).isNotNull();
        assertThat(ltt.getId().getName()).isEqualTo("data.migration");
        assertThat(ltt.getId().getTag("table")).isEqualTo("users");
        assertThat(ltt.getId().getDescription()).isEqualTo("Tracks active data migrations");
        assertThat(ltt.getId().getType()).isEqualTo(Meter.Type.LONG_TASK_TIMER);
    }

    @Test
    void shouldCreateLongTaskTimer_ViaMoreConvenience() {
        LongTaskTimer ltt = registry.more().longTaskTimer("batch.job", "type", "export");

        assertThat(ltt).isNotNull();
        assertThat(ltt.getId().getName()).isEqualTo("batch.job");
        assertThat(ltt.getId().getTag("type")).isEqualTo("export");
    }

    @Test
    void shouldDeduplicateSameNameAndTags() {
        LongTaskTimer ltt1 = LongTaskTimer.builder("migration")
                .tag("table", "users")
                .register(registry);
        LongTaskTimer ltt2 = LongTaskTimer.builder("migration")
                .tag("table", "users")
                .register(registry);

        assertThat(ltt1).isSameAs(ltt2);
    }

    // ── Start / Stop Lifecycle ───────────────────────────────────────

    @Test
    void shouldTrackActiveTasks_AfterStart() {
        LongTaskTimer ltt = LongTaskTimer.builder("migration").register(registry);

        assertThat(ltt.activeTasks()).isEqualTo(0);

        LongTaskTimer.Sample task = ltt.start();
        assertThat(ltt.activeTasks()).isEqualTo(1);

        task.stop();
        assertThat(ltt.activeTasks()).isEqualTo(0);
    }

    @Test
    void shouldTrackMultipleActiveTasks() {
        LongTaskTimer ltt = LongTaskTimer.builder("batch").register(registry);

        LongTaskTimer.Sample task1 = ltt.start();
        clock.addNanos(TimeUnit.SECONDS.toNanos(1));

        LongTaskTimer.Sample task2 = ltt.start();
        clock.addNanos(TimeUnit.SECONDS.toNanos(1));

        LongTaskTimer.Sample task3 = ltt.start();

        assertThat(ltt.activeTasks()).isEqualTo(3);

        task2.stop();
        assertThat(ltt.activeTasks()).isEqualTo(2);

        task1.stop();
        assertThat(ltt.activeTasks()).isEqualTo(1);

        task3.stop();
        assertThat(ltt.activeTasks()).isEqualTo(0);
    }

    @Test
    void shouldReturnDurationInNanos_FromStop() {
        LongTaskTimer ltt = LongTaskTimer.builder("task").register(registry);

        LongTaskTimer.Sample task = ltt.start();
        clock.addNanos(TimeUnit.MILLISECONDS.toNanos(500));

        long durationNanos = task.stop();

        assertThat(durationNanos).isEqualTo(TimeUnit.MILLISECONDS.toNanos(500));
    }

    // ── Duration Queries ─────────────────────────────────────────────

    @Test
    void shouldReportTotalDuration_OfAllActiveTasks() {
        LongTaskTimer ltt = LongTaskTimer.builder("migration").register(registry);

        // Task 1 starts at t=0
        LongTaskTimer.Sample task1 = ltt.start();
        clock.addNanos(TimeUnit.SECONDS.toNanos(3));

        // Task 2 starts at t=3s
        LongTaskTimer.Sample task2 = ltt.start();
        clock.addNanos(TimeUnit.SECONDS.toNanos(2));

        // At t=5s: task1 has been running 5s, task2 has been running 2s
        // Total duration = 5 + 2 = 7 seconds
        assertThat(ltt.duration(TimeUnit.SECONDS)).isCloseTo(7.0, within(0.01));

        task1.stop();
        task2.stop();
    }

    @Test
    void shouldReportZeroDuration_WhenNoActiveTasks() {
        LongTaskTimer ltt = LongTaskTimer.builder("idle").register(registry);
        assertThat(ltt.duration(TimeUnit.SECONDS)).isEqualTo(0.0);
    }

    @Test
    void shouldConvertDuration_ToRequestedTimeUnit() {
        LongTaskTimer ltt = LongTaskTimer.builder("task").register(registry);

        LongTaskTimer.Sample task = ltt.start();
        clock.addNanos(TimeUnit.MILLISECONDS.toNanos(1500));

        assertThat(ltt.duration(TimeUnit.SECONDS)).isCloseTo(1.5, within(0.01));
        assertThat(ltt.duration(TimeUnit.MILLISECONDS)).isCloseTo(1500.0, within(1.0));

        task.stop();
    }

    // ── Max Query ────────────────────────────────────────────────────

    @Test
    void shouldReportMax_AsLongestRunningTask() {
        LongTaskTimer ltt = LongTaskTimer.builder("migration").register(registry);

        // Task 1 starts at t=0 (will be the longest)
        LongTaskTimer.Sample task1 = ltt.start();
        clock.addNanos(TimeUnit.SECONDS.toNanos(5));

        // Task 2 starts at t=5s (shorter)
        LongTaskTimer.Sample task2 = ltt.start();
        clock.addNanos(TimeUnit.SECONDS.toNanos(2));

        // At t=7s: task1=7s, task2=2s → max should be 7s
        assertThat(ltt.max(TimeUnit.SECONDS)).isCloseTo(7.0, within(0.01));

        task1.stop();

        // After task1 stops, max should be task2's duration (now 2s)
        assertThat(ltt.max(TimeUnit.SECONDS)).isCloseTo(2.0, within(0.01));

        task2.stop();
    }

    @Test
    void shouldReportZeroMax_WhenNoActiveTasks() {
        LongTaskTimer ltt = LongTaskTimer.builder("idle").register(registry);
        assertThat(ltt.max(TimeUnit.SECONDS)).isEqualTo(0.0);
    }

    // ── Mean Query ───────────────────────────────────────────────────

    @Test
    void shouldComputeMean_AsDurationDividedByActiveTasks() {
        LongTaskTimer ltt = LongTaskTimer.builder("migration").register(registry);

        LongTaskTimer.Sample task1 = ltt.start();
        clock.addNanos(TimeUnit.SECONDS.toNanos(4));

        LongTaskTimer.Sample task2 = ltt.start();
        clock.addNanos(TimeUnit.SECONDS.toNanos(2));

        // At t=6s: task1=6s, task2=2s → total=8s, active=2, mean=4s
        assertThat(ltt.mean(TimeUnit.SECONDS)).isCloseTo(4.0, within(0.01));

        task1.stop();
        task2.stop();
    }

    @Test
    void shouldReturnZeroMean_WhenNoActiveTasks() {
        LongTaskTimer ltt = LongTaskTimer.builder("idle").register(registry);
        assertThat(ltt.mean(TimeUnit.SECONDS)).isEqualTo(0.0);
    }

    // ── Sample.duration() ────────────────────────────────────────────

    @Test
    void shouldReportSampleDuration_WhileRunning() {
        LongTaskTimer ltt = LongTaskTimer.builder("task").register(registry);

        LongTaskTimer.Sample task = ltt.start();
        clock.addNanos(TimeUnit.SECONDS.toNanos(3));

        assertThat(task.duration(TimeUnit.SECONDS)).isCloseTo(3.0, within(0.01));

        clock.addNanos(TimeUnit.SECONDS.toNanos(2));
        assertThat(task.duration(TimeUnit.SECONDS)).isCloseTo(5.0, within(0.01));

        task.stop();
    }

    @Test
    void shouldReturnNegativeOne_FromStoppedSampleDuration() {
        LongTaskTimer ltt = LongTaskTimer.builder("task").register(registry);

        LongTaskTimer.Sample task = ltt.start();
        clock.addNanos(TimeUnit.SECONDS.toNanos(1));
        task.stop();

        assertThat(task.duration(TimeUnit.SECONDS)).isEqualTo(-1.0);
    }

    // ── Measurements ─────────────────────────────────────────────────

    @Test
    void shouldProduceThreeMeasurements() {
        LongTaskTimer ltt = LongTaskTimer.builder("migration").register(registry);

        LongTaskTimer.Sample task = ltt.start();
        clock.addNanos(TimeUnit.SECONDS.toNanos(5));

        var measurements = new java.util.ArrayList<Measurement>();
        ltt.measure().forEach(measurements::add);

        assertThat(measurements).hasSize(3);

        // ACTIVE_TASKS
        Measurement activeMeasurement = measurements.stream()
                .filter(m -> m.getStatistic() == Statistic.ACTIVE_TASKS)
                .findFirst().orElseThrow();
        assertThat(activeMeasurement.getValue()).isEqualTo(1.0);

        // DURATION (in nanoseconds)
        Measurement durationMeasurement = measurements.stream()
                .filter(m -> m.getStatistic() == Statistic.DURATION)
                .findFirst().orElseThrow();
        assertThat(durationMeasurement.getValue())
                .isCloseTo(TimeUnit.SECONDS.toNanos(5), within(1_000_000.0));

        // MAX (in nanoseconds)
        Measurement maxMeasurement = measurements.stream()
                .filter(m -> m.getStatistic() == Statistic.MAX)
                .findFirst().orElseThrow();
        assertThat(maxMeasurement.getValue())
                .isCloseTo(TimeUnit.SECONDS.toNanos(5), within(1_000_000.0));

        task.stop();
    }

    // ── Convenience Recording Methods ────────────────────────────────

    @Test
    void shouldRecordRunnable() {
        LongTaskTimer ltt = LongTaskTimer.builder("job").register(registry);

        ltt.record(() -> {
            // While inside the runnable, there should be 1 active task
            assertThat(ltt.activeTasks()).isEqualTo(1);
        });

        // After completion, no active tasks
        assertThat(ltt.activeTasks()).isEqualTo(0);
    }

    @Test
    void shouldRecordSupplier_AndReturnResult() {
        LongTaskTimer ltt = LongTaskTimer.builder("compute").register(registry);

        String result = ltt.record(() -> {
            assertThat(ltt.activeTasks()).isEqualTo(1);
            return "computed";
        });

        assertThat(result).isEqualTo("computed");
        assertThat(ltt.activeTasks()).isEqualTo(0);
    }

    @Test
    void shouldRecordCallable_AndReturnResult() throws Exception {
        LongTaskTimer ltt = LongTaskTimer.builder("query").register(registry);

        String result = ltt.recordCallable(() -> {
            assertThat(ltt.activeTasks()).isEqualTo(1);
            return "queried";
        });

        assertThat(result).isEqualTo("queried");
        assertThat(ltt.activeTasks()).isEqualTo(0);
    }

    @Test
    void shouldStopTask_EvenWhenRunnableThrows() {
        LongTaskTimer ltt = LongTaskTimer.builder("failing").register(registry);

        assertThatThrownBy(() -> ltt.record((Runnable) () -> {
            throw new RuntimeException("boom");
        })).isInstanceOf(RuntimeException.class);

        assertThat(ltt.activeTasks()).isEqualTo(0);
    }

    @Test
    void shouldStopTask_EvenWhenCallableThrows() {
        LongTaskTimer ltt = LongTaskTimer.builder("failing").register(registry);

        assertThatThrownBy(() -> ltt.recordCallable(() -> {
            throw new Exception("checked");
        })).isInstanceOf(Exception.class);

        assertThat(ltt.activeTasks()).isEqualTo(0);
    }

    // ── Noop Behavior ────────────────────────────────────────────────

    @Test
    void shouldReturnNoopLongTaskTimer_WhenDeniedByFilter() {
        registry.config().meterFilter(MeterFilter.deny());

        LongTaskTimer ltt = LongTaskTimer.builder("denied.timer").register(registry);

        assertThat(ltt).isInstanceOf(NoopLongTaskTimer.class);
        assertThat(ltt.activeTasks()).isEqualTo(0);
        assertThat(ltt.duration(TimeUnit.SECONDS)).isEqualTo(0.0);
        assertThat(ltt.max(TimeUnit.SECONDS)).isEqualTo(0.0);

        // Noop sample should be inert
        LongTaskTimer.Sample sample = ltt.start();
        assertThat(sample.stop()).isEqualTo(0);
        assertThat(sample.duration(TimeUnit.SECONDS)).isEqualTo(0.0);
    }

    @Test
    void shouldExecuteRunnable_EvenWhenNoop() {
        registry.config().meterFilter(MeterFilter.deny());
        LongTaskTimer ltt = LongTaskTimer.builder("denied").register(registry);

        AtomicLong counter = new AtomicLong(0);
        ltt.record(counter::incrementAndGet);
        assertThat(counter.get()).isEqualTo(1);
    }

    // ── More convenience registration ────────────────────────────────

    @Test
    void shouldRegisterViaMore_WithNameOnly() {
        LongTaskTimer ltt = registry.more().longTaskTimer("simple.timer");

        assertThat(ltt).isNotNull();
        assertThat(ltt.getId().getName()).isEqualTo("simple.timer");
        assertThat(ltt.getId().getTags()).isEmpty();
    }

    @Test
    void shouldRegisterViaMore_WithIterableTags() {
        LongTaskTimer ltt = registry.more().longTaskTimer("tagged",
                Tags.of("env", "prod", "region", "us-east"));

        assertThat(ltt.getId().getTag("env")).isEqualTo("prod");
        assertThat(ltt.getId().getTag("region")).isEqualTo("us-east");
    }

    // ── MeterFilter integration ──────────────────────────────────────

    @Test
    void shouldApplyCommonTags_ToLongTaskTimer() {
        registry.config().commonTags("app", "myapp");

        LongTaskTimer ltt = LongTaskTimer.builder("migration").register(registry);

        assertThat(ltt.getId().getTag("app")).isEqualTo("myapp");
    }

    // ── Type conflict ────────────────────────────────────────────────

    @Test
    void shouldThrow_WhenSameIdRegisteredAsDifferentType() {
        registry.counter("conflict.name");

        assertThatThrownBy(() ->
                registry.more().longTaskTimer("conflict.name"))
                .isInstanceOf(IllegalArgumentException.class);
    }

    // ── MockClock ────────────────────────────────────────────────────

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
