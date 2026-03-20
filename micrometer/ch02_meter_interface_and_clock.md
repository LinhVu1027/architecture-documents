# Chapter 2: Meter Interface & Clock

> Line references based on commit `2c8a4606c` of the Micrometer repository.

## Build Challenge

| Current State | Limitation | Objective |
|---------------|------------|-----------|
| We have `Meter.Id` and `Tags` for identity, but `Meter` is an empty shell — no instance methods, no way to read measurements from a meter | Cannot create actual meter objects, measure them, or dispatch across meter types; no time abstraction for testability | Add `getId()`, `measure()`, and a visitor pattern to `Meter`; create `Clock` for time abstraction and `AbstractMeter` as the concrete base class |

---

## 2.1 The Integration Point

This feature is unusual: the integration point is the `Meter` interface itself. It already exists from Chapter 1, but it's purely a container for inner types (`Id`, `Type`). We need to transform it into the root abstraction that every meter type implements.

**Modifying:** `src/main/java/dev/linhvu/micrometer/Meter.java`
**Change:** Add `getId()`, `measure()`, and `match()`/`use()` visitor methods to the interface

```java
public interface Meter {

    /**
     * Returns the unique identity of this meter — its name and tags.
     */
    Id getId();

    /**
     * Returns the current set of measurements for this meter.
     * A counter returns one measurement (COUNT); a timer returns three
     * (COUNT, TOTAL_TIME, MAX). The number and order of measurements
     * for a given meter never changes.
     */
    Iterable<Measurement> measure();

    /**
     * Type-safe visitor that returns a value.
     */
    default <T> T match(
            Function<Counter, T> visitCounter,
            Function<Gauge, T> visitGauge,
            Function<Timer, T> visitTimer,
            Function<DistributionSummary, T> visitDistributionSummary,
            Function<LongTaskTimer, T> visitLongTaskTimer,
            Function<FunctionCounter, T> visitFunctionCounter,
            Function<FunctionTimer, T> visitFunctionTimer,
            Function<Meter, T> defaultFunction) {
        if (this instanceof Counter c) return visitCounter.apply(c);
        if (this instanceof Timer t) return visitTimer.apply(t);
        // ... more specific types before more general ones
        if (this instanceof Gauge g) return visitGauge.apply(g);
        return defaultFunction.apply(this);
    }

    // ... existing Id, Type inner types unchanged ...
}
```

Two key decisions here:

1. **`match` uses `instanceof` dispatch, not a `Type` enum switch.** The enum exists for metadata, but a meter's Java type is the authoritative identity. A class implementing both `Counter` and some custom interface would match as a Counter — the runtime type wins.
2. **Gauge is checked LAST among the standard types** because `TimeGauge extends Gauge` (in later features). Checking Gauge first would swallow TimeGauges. Order matters.

This connects **the identity system (Chapter 1)** to **the measurement pipeline (all future meters)**. To make it work, we need to build:
- **Stub interfaces** (`Counter`, `Gauge`, `Timer`, etc.) — empty markers that future features will flesh out
- **`Clock`** — time abstraction so registries and timers don't depend on `System.nanoTime()` directly
- **`AbstractMeter`** — base class holding the `Id` and providing identity-based equality

## 2.2 Stub Meter Type Interfaces

The visitor references types that don't exist yet. We create minimal marker interfaces that future features will expand:

**New file:** `src/main/java/dev/linhvu/micrometer/Counter.java`

```java
package dev.linhvu.micrometer;

public interface Counter extends Meter {
}
```

**New file:** `src/main/java/dev/linhvu/micrometer/Gauge.java`

```java
package dev.linhvu.micrometer;

public interface Gauge extends Meter {
}
```

**New file:** `src/main/java/dev/linhvu/micrometer/Timer.java`

```java
package dev.linhvu.micrometer;

public interface Timer extends Meter {
}
```

**New file:** `src/main/java/dev/linhvu/micrometer/DistributionSummary.java`

```java
package dev.linhvu.micrometer;

public interface DistributionSummary extends Meter {
}
```

**New file:** `src/main/java/dev/linhvu/micrometer/LongTaskTimer.java`

```java
package dev.linhvu.micrometer;

public interface LongTaskTimer extends Meter {
}
```

**New file:** `src/main/java/dev/linhvu/micrometer/FunctionCounter.java`

```java
package dev.linhvu.micrometer;

public interface FunctionCounter extends Meter {
}
```

**New file:** `src/main/java/dev/linhvu/micrometer/FunctionTimer.java`

```java
package dev.linhvu.micrometer;

public interface FunctionTimer extends Meter {
}
```

These are empty now — each will gain its real API when its feature is implemented. The visitor pattern will compile and work immediately because it dispatches on `instanceof`, not on the interface's methods.

## 2.3 Clock

**New file:** `src/main/java/dev/linhvu/micrometer/Clock.java`

```java
package dev.linhvu.micrometer;

public interface Clock {

    Clock SYSTEM = new Clock() {
        @Override
        public long wallTime() {
            return System.currentTimeMillis();
        }

        @Override
        public long monotonicTime() {
            return System.nanoTime();
        }
    };

    /**
     * Current wall-clock time in milliseconds since the epoch.
     */
    long wallTime();

    /**
     * Current monotonic time in nanoseconds.
     */
    long monotonicTime();
}
```

Two methods, two time domains:
- **`wallTime()`** — milliseconds, for timestamps. Can jump (NTP adjustments).
- **`monotonicTime()`** — nanoseconds, for durations. Never goes backwards.

`Clock.SYSTEM` is the production default. Tests use `MockClock` (next section) to control time explicitly.

## 2.4 MockClock

**New file:** `src/main/java/dev/linhvu/micrometer/MockClock.java`

```java
package dev.linhvu.micrometer;

import java.time.Duration;
import java.util.concurrent.TimeUnit;

public class MockClock implements Clock {

    private long timeNanos = TimeUnit.MILLISECONDS.toNanos(1);

    @Override
    public long wallTime() {
        return TimeUnit.NANOSECONDS.toMillis(timeNanos);
    }

    @Override
    public long monotonicTime() {
        return timeNanos;
    }

    public long add(long amount, TimeUnit unit) {
        timeNanos += unit.toNanos(amount);
        return timeNanos;
    }

    public long add(Duration duration) {
        return add(duration.toNanos(), TimeUnit.NANOSECONDS);
    }

    public long addSeconds(long seconds) {
        return add(seconds, TimeUnit.SECONDS);
    }
}
```

The initial value is 1 millisecond (not zero) to avoid divide-by-zero in rate calculations. Both `wallTime()` and `monotonicTime()` derive from the same internal counter — in production they're independent sources, but for testing, having them correlated simplifies assertions.

## 2.5 AbstractMeter

**New file:** `src/main/java/dev/linhvu/micrometer/AbstractMeter.java`

```java
package dev.linhvu.micrometer;

public abstract class AbstractMeter implements Meter {

    private final Meter.Id id;

    protected AbstractMeter(Meter.Id id) {
        this.id = id;
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
}
```

The equality check uses `instanceof Meter` (not `instanceof AbstractMeter`) — this is deliberate. A Counter and a Timer with the same name+tags are `equals()`. The registry relies on this to detect type-mismatch collisions when someone tries to register a Counter and a Timer under the same name.

## 2.6 Try It Yourself

<details>
<summary>Challenge: Implement a custom meter that reports two measurements — a VALUE and a COUNT</summary>

Create a class `HitRateMeter` that extends `AbstractMeter` and reports both how many times something was checked and the current hit rate:

```java
public class HitRateMeter extends AbstractMeter {

    private int checks = 0;
    private int hits = 0;

    public HitRateMeter(Meter.Id id) {
        super(id);
    }

    public void recordCheck(boolean hit) {
        checks++;
        if (hit) hits++;
    }

    @Override
    public Iterable<Measurement> measure() {
        double rate = checks == 0 ? 0.0 : (double) hits / checks;
        return List.of(
                new Measurement(() -> (double) checks, Statistic.COUNT),
                new Measurement(() -> rate, Statistic.VALUE)
        );
    }
}
```

</details>

<details>
<summary>Challenge: Use the visitor pattern to format different meter types as strings</summary>

Write a method that produces a human-readable format: `"Counter[requests] = 42"` or `"Gauge[temperature] = 72.5"`:

```java
public static String format(Meter meter) {
    return meter.match(
            counter -> "Counter[" + counter.getId().getName() + "]",
            gauge -> "Gauge[" + gauge.getId().getName() + "]",
            timer -> "Timer[" + timer.getId().getName() + "]",
            ds -> "DistributionSummary[" + ds.getId().getName() + "]",
            ltt -> "LongTaskTimer[" + ltt.getId().getName() + "]",
            fc -> "FunctionCounter[" + fc.getId().getName() + "]",
            ft -> "FunctionTimer[" + ft.getId().getName() + "]",
            m -> "Meter[" + m.getId().getName() + "]"
    );
}
```

</details>

## 2.7 Tests

### Unit Tests

**New file:** `src/test/java/dev/linhvu/micrometer/ClockTest.java`

```java
class ClockTest {

    @Test
    void shouldReturnSystemMillis_WhenUsingSystemClock() {
        long before = System.currentTimeMillis();
        long wallTime = Clock.SYSTEM.wallTime();
        long after = System.currentTimeMillis();
        assertThat(wallTime).isBetween(before, after);
    }

    @Test
    void shouldReturnSystemNanos_WhenUsingSystemClock() {
        long before = System.nanoTime();
        long monotonicTime = Clock.SYSTEM.monotonicTime();
        long after = System.nanoTime();
        assertThat(monotonicTime).isBetween(before, after);
    }
}
```

**New file:** `src/test/java/dev/linhvu/micrometer/MockClockTest.java`

```java
class MockClockTest {

    @Test
    void shouldStartAtOneMillisecond_WhenCreated() { ... }

    @Test
    void shouldAdvanceTime_WhenAddingWithTimeUnit() { ... }

    @Test
    void shouldAdvanceTime_WhenAddingDuration() { ... }

    @Test
    void shouldAdvanceTime_WhenAddingSeconds() { ... }

    @Test
    void shouldKeepWallAndMonotonicInSync_WhenAdvancingTime() { ... }

    @Test
    void shouldReturnNewMonotonicTime_WhenAddReturnsValue() { ... }
}
```

**New file:** `src/test/java/dev/linhvu/micrometer/AbstractMeterTest.java`

```java
class AbstractMeterTest {

    @Test
    void shouldReturnId_WhenGetIdCalled() { ... }

    @Test
    void shouldReturnMeasurements_WhenMeasureCalled() { ... }

    @Test
    void shouldBeEqual_WhenMetersHaveSameNameAndTags() { ... }

    @Test
    void shouldNotBeEqual_WhenMetersHaveDifferentNames() { ... }

    @Test
    void shouldNotBeEqual_WhenMetersHaveDifferentTags() { ... }

    @Test
    void shouldBeEqual_WhenTypesDifferButNameAndTagsMatch() { ... }

    @Test
    void shouldIncludeIdInToString() { ... }
}
```

**New file:** `src/test/java/dev/linhvu/micrometer/MeterVisitorTest.java`

```java
class MeterVisitorTest {

    @Test
    void shouldMatchCounter_WhenMeterIsCounter() { ... }

    @Test
    void shouldMatchGauge_WhenMeterIsGauge() { ... }

    @Test
    void shouldMatchDefault_WhenMeterIsCustomType() { ... }

    @Test
    void shouldCallCorrectConsumer_WhenUsingUseVisitor() { ... }

    @Test
    void shouldPassMeterInstance_WhenVisiting() { ... }
}
```

**Run:** `./gradlew test` — expected: all 65 tests pass (38 from Chapter 1 + 27 new)

---

## 2.8 Why This Works

> ★ **Insight** -------------------------------------------
> - **Why two time domains?** Wall-clock time (`System.currentTimeMillis()`) is for timestamps and scheduling — "when did this step start?" But it's unreliable for measuring durations because NTP can jump it forward or backward. Monotonic time (`System.nanoTime()`) never goes backwards, making it safe for duration measurement, but it has no relation to real-world time. By keeping them as separate methods on `Clock`, Micrometer uses the right tool for each job. The real `Clock.java` (`Clock.java:47-56`) has exactly this same two-method design.
> - **Trade-off:** Unifying them in MockClock (single nanosecond counter for both) is a simplification. In production, `wallTime` and `monotonicTime` drift apart. If you need to test NTP-jump behavior, you'd need separate counters.
> -----------------------------------------------------------

> ★ **Insight** -------------------------------------------
> - **Why identity-based equality across meter types?** `AbstractMeter.equals()` checks `instanceof Meter`, not `instanceof AbstractMeter`. This means a Counter and a Timer with the same name+tags are considered equal. This seems wrong until you consider the registry: it stores meters in a `ConcurrentHashMap` keyed by `Meter.Id`. If someone accidentally registers `counter("http.requests")` and then `timer("http.requests")` with the same tags, the registry MUST detect the collision. Identity-based equality makes this collision detection automatic — the map lookup finds the existing entry, and the registry can throw a meaningful error rather than silently creating a duplicate.
> - **Real Micrometer's approach:** `MeterEquivalence.java:28-37` extracts this same logic into a utility class, but the principle is identical.
> -----------------------------------------------------------

> ★ **Insight** -------------------------------------------
> - **Why a functional visitor instead of the classic Visitor pattern?** The classic Gang-of-Four visitor requires an `accept(Visitor)` method on every concrete class and a `Visitor` interface with `visit(Counter)`, `visit(Timer)`, etc. Micrometer's approach uses a single `match(Function, Function, ...)` default method. The advantage: it's a one-liner at the call site — just pass lambdas. The cost: adding a new meter type changes the method signature and breaks every call site. But Micrometer treats this as a *feature* — a compile error is better than silently ignoring a new type.
> - **When NOT to use this:** If meter types change frequently, this pattern creates painful churn. It works for Micrometer because the set of meter types is very stable (hasn't changed since 1.0).
> -----------------------------------------------------------

## 2.9 What We Enhanced

| Aspect | Before (ch01) | Current (ch02) | Real Framework |
|--------|---------------|----------------|----------------|
| `Meter` interface | Empty shell — only contains `Id` and `Type` inner types | Full interface with `getId()`, `measure()`, `match()`/`use()` visitor | Same methods plus `close()`, `MeterProvider<T>`, and `Builder` inner class (`Meter.java:40-524`) |
| Time abstraction | None | `Clock` interface with wall/monotonic time, `MockClock` for testing | Identical design (`Clock.java:26-56`, `MockClock.java:25-54`) |
| Meter base class | None | `AbstractMeter` with identity-based equality | Same, but equality is externalized to `MeterEquivalence` utility (`AbstractMeter.java:27-49`, `MeterEquivalence.java:28-37`) |

## 2.10 Connection to Real Framework

| Simplified Code | Real Framework Code | File:Line | Key Difference |
|----------------|--------------------|-----------| ---------------|
| `Meter.getId()` | `Meter.getId()` | `Meter.java:49` | Identical contract |
| `Meter.measure()` | `Meter.measure()` | `Meter.java:57` | Identical contract |
| `Meter.match()` (8 params) | `Meter.match()` (9 params) | `Meter.java:95-126` | Real version also handles `TimeGauge` separately from `Gauge` |
| `Meter.use()` | `Meter.use()` | `Meter.java:149-180` | Real version delegates to `match()` identically |
| `Clock.SYSTEM` | `Clock.SYSTEM` | `Clock.java:28-38` | Identical implementation |
| `Clock.wallTime()` | `Clock.wallTime()` | `Clock.java:47` | Identical — milliseconds |
| `Clock.monotonicTime()` | `Clock.monotonicTime()` | `Clock.java:56` | Identical — nanoseconds |
| `AbstractMeter` | `AbstractMeter` | `AbstractMeter.java:27-49` | Real version delegates equality to `MeterEquivalence` utility class |
| `MockClock` | `MockClock` | `MockClock.java:25-54` | Real version adds `clock(MeterRegistry)` static factory for extraction |

## 2.11 Complete Code

### Production Code

#### File: `src/main/java/dev/linhvu/micrometer/Meter.java` [MODIFIED]

```java
package dev.linhvu.micrometer;

import java.util.ArrayList;
import java.util.Collections;
import java.util.List;
import java.util.Objects;
import java.util.function.Consumer;
import java.util.function.Function;

/**
 * A named and tagged set of measurements. This is the base type for all meters.
 * <p>
 * Every meter has a unique {@link Id} (name + tags) and produces an {@link Iterable}
 * of {@link Measurement}s. The {@code match/use} visitor methods provide type-safe
 * dispatching across the concrete meter types without instanceof chains.
 */
public interface Meter {

    /**
     * Returns the unique identity of this meter — its name and tags.
     */
    Id getId();

    /**
     * Returns the current set of measurements for this meter.
     * A counter returns one measurement (COUNT); a timer returns three
     * (COUNT, TOTAL_TIME, MAX). The number and order of measurements
     * for a given meter never changes.
     */
    Iterable<Measurement> measure();

    /**
     * Type-safe visitor that returns a value. One function per meter type, plus a
     * fallback for custom meters. If new meter types are added, this method signature
     * will change — intentionally forcing a compile error at every call site so nothing
     * is silently missed.
     * <p>
     * Ordering matters: TimeGauge is checked before Gauge because TimeGauge extends Gauge.
     */
    @SuppressWarnings("unchecked")
    default <T> T match(
            Function<Counter, T> visitCounter,
            Function<Gauge, T> visitGauge,
            Function<Timer, T> visitTimer,
            Function<DistributionSummary, T> visitDistributionSummary,
            Function<LongTaskTimer, T> visitLongTaskTimer,
            Function<FunctionCounter, T> visitFunctionCounter,
            Function<FunctionTimer, T> visitFunctionTimer,
            Function<Meter, T> defaultFunction) {
        // Order matters: more specific types must be checked first
        if (this instanceof Counter c) return visitCounter.apply(c);
        if (this instanceof Timer t) return visitTimer.apply(t);
        if (this instanceof LongTaskTimer ltt) return visitLongTaskTimer.apply(ltt);
        if (this instanceof DistributionSummary ds) return visitDistributionSummary.apply(ds);
        if (this instanceof FunctionCounter fc) return visitFunctionCounter.apply(fc);
        if (this instanceof FunctionTimer ft) return visitFunctionTimer.apply(ft);
        if (this instanceof Gauge g) return visitGauge.apply(g);
        return defaultFunction.apply(this);
    }

    /**
     * Type-safe visitor for side-effect operations (no return value).
     */
    default void use(
            Consumer<Counter> visitCounter,
            Consumer<Gauge> visitGauge,
            Consumer<Timer> visitTimer,
            Consumer<DistributionSummary> visitDistributionSummary,
            Consumer<LongTaskTimer> visitLongTaskTimer,
            Consumer<FunctionCounter> visitFunctionCounter,
            Consumer<FunctionTimer> visitFunctionTimer,
            Consumer<Meter> defaultConsumer) {
        match(
                counter -> { visitCounter.accept(counter); return null; },
                gauge -> { visitGauge.accept(gauge); return null; },
                timer -> { visitTimer.accept(timer); return null; },
                ds -> { visitDistributionSummary.accept(ds); return null; },
                ltt -> { visitLongTaskTimer.accept(ltt); return null; },
                fc -> { visitFunctionCounter.accept(fc); return null; },
                ft -> { visitFunctionTimer.accept(ft); return null; },
                meter -> { defaultConsumer.accept(meter); return null; }
        );
    }

    /**
     * Enumerates the different types of meters.
     */
    enum Type {

        COUNTER,
        GAUGE,
        LONG_TASK_TIMER,
        TIMER,
        DISTRIBUTION_SUMMARY,
        OTHER

    }

    /**
     * Uniquely identifies a meter by its name and tags.
     * <p>
     * Two IDs with the same name and tags are considered equal, regardless of
     * type, description, or base unit. This is intentional — the registry uses
     * name+tags as the lookup key, and metadata is supplementary.
     * <p>
     * All {@code with*} methods return new instances — Id is immutable.
     */
    class Id {

        private final String name;

        private final Tags tags;

        private final Type type;

        private final String description;

        private final String baseUnit;

        public Id(String name, Tags tags, Type type, String description, String baseUnit) {
            Objects.requireNonNull(name, "name must not be null");
            this.name = name;
            this.tags = (tags != null) ? tags : Tags.empty();
            this.type = type;
            this.description = description;
            this.baseUnit = baseUnit;
        }

        // --- Accessors ---

        public String getName() {
            return name;
        }

        public Tags getTagsAsIterable() {
            return tags;
        }

        /**
         * Returns an unmodifiable list of tags. Prefer {@link #getTagsAsIterable()}
         * for iteration to avoid the copy.
         */
        public List<Tag> getTags() {
            List<Tag> list = new ArrayList<>();
            tags.forEach(list::add);
            return Collections.unmodifiableList(list);
        }

        /**
         * Looks up a tag value by key via linear scan. Returns null if not found.
         */
        public String getTag(String key) {
            for (Tag tag : tags) {
                if (tag.getKey().equals(key)) {
                    return tag.getValue();
                }
            }
            return null;
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

        // --- Immutable copy methods ---

        public Id withName(String newName) {
            return new Id(newName, tags, type, description, baseUnit);
        }

        public Id withTag(Tag tag) {
            return new Id(name, tags.and(tag), type, description, baseUnit);
        }

        public Id withTags(Iterable<Tag> newTags) {
            return new Id(name, tags.and(newTags), type, description, baseUnit);
        }

        public Id replaceTags(Iterable<Tag> newTags) {
            return new Id(name, Tags.of(newTags), type, description, baseUnit);
        }

        /**
         * Adds a "statistic" tag with the given statistic's tag-value representation.
         * Used when exporting individual measurements as separate time series.
         */
        public Id withTag(Statistic statistic) {
            return withTag(Tag.of("statistic", statistic.getTagValueRepresentation()));
        }

        public Id withBaseUnit(String newBaseUnit) {
            return new Id(name, tags, type, description, newBaseUnit);
        }

        // --- Identity: name + tags only ---

        @Override
        public boolean equals(Object o) {
            if (this == o)
                return true;
            if (!(o instanceof Id other))
                return false;
            return name.equals(other.name) && tags.equals(other.tags);
        }

        @Override
        public int hashCode() {
            int result = name.hashCode();
            result = 31 * result + tags.hashCode();
            return result;
        }

        @Override
        public String toString() {
            return "Id{name='" + name + "', tags=" + tags + '}';
        }

    }

}
```

#### File: `src/main/java/dev/linhvu/micrometer/Clock.java` [NEW]

```java
package dev.linhvu.micrometer;

/**
 * Abstraction for time measurement, separating two distinct time domains:
 * <ul>
 *   <li>{@link #wallTime()} — wall-clock time in <b>milliseconds</b>, for timestamps and
 *       step boundaries. Can jump due to NTP adjustments.</li>
 *   <li>{@link #monotonicTime()} — monotonic time in <b>nanoseconds</b>, for measuring
 *       durations. Never goes backwards, but has no relation to wall-clock.</li>
 * </ul>
 * <p>
 * Injecting Clock into the registry makes all time-dependent code testable —
 * tests use {@link MockClock} to control time explicitly.
 */
public interface Clock {

    /**
     * Default implementation backed by {@link System#currentTimeMillis()}
     * and {@link System#nanoTime()}.
     */
    Clock SYSTEM = new Clock() {
        @Override
        public long wallTime() {
            return System.currentTimeMillis();
        }

        @Override
        public long monotonicTime() {
            return System.nanoTime();
        }
    };

    /**
     * Current wall-clock time in milliseconds since the epoch.
     * Used for timestamping exports and determining step boundaries.
     */
    long wallTime();

    /**
     * Current monotonic time in nanoseconds.
     * Used for measuring durations (e.g., Timer recordings).
     */
    long monotonicTime();

}
```

#### File: `src/main/java/dev/linhvu/micrometer/AbstractMeter.java` [NEW]

```java
package dev.linhvu.micrometer;

/**
 * Base class for all concrete meter implementations.
 * <p>
 * Provides the shared plumbing that every meter needs: storing the {@link Meter.Id}
 * and implementing identity-based {@code equals}/{@code hashCode}. Concrete meters
 * (Counter, Timer, Gauge, etc.) extend this and only need to implement
 * {@link Meter#measure()}.
 * <p>
 * Equality is based solely on the meter's Id (name + tags). This means a Counter and
 * a Timer with the same name and tags are considered {@code equals()} — which is
 * deliberate, since the registry uses name+tags as its map key and must detect
 * such collisions.
 */
public abstract class AbstractMeter implements Meter {

    private final Meter.Id id;

    protected AbstractMeter(Meter.Id id) {
        this.id = id;
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

}
```

#### File: `src/main/java/dev/linhvu/micrometer/MockClock.java` [NEW]

```java
package dev.linhvu.micrometer;

import java.time.Duration;
import java.util.concurrent.TimeUnit;

/**
 * A controllable {@link Clock} for testing. Both {@link #wallTime()} and
 * {@link #monotonicTime()} are derived from a single internal nanosecond counter,
 * so advancing time moves both clocks in lockstep.
 * <p>
 * Initialized to 1 millisecond (not zero) to avoid divide-by-zero in rate
 * calculations that divide by elapsed time.
 */
public class MockClock implements Clock {

    private long timeNanos = TimeUnit.MILLISECONDS.toNanos(1);

    @Override
    public long wallTime() {
        return TimeUnit.NANOSECONDS.toMillis(timeNanos);
    }

    @Override
    public long monotonicTime() {
        return timeNanos;
    }

    /**
     * Advance time by the given amount.
     *
     * @return the new monotonic time in nanoseconds
     */
    public long add(long amount, TimeUnit unit) {
        timeNanos += unit.toNanos(amount);
        return timeNanos;
    }

    /**
     * Advance time by the given duration.
     *
     * @return the new monotonic time in nanoseconds
     */
    public long add(Duration duration) {
        return add(duration.toNanos(), TimeUnit.NANOSECONDS);
    }

    /**
     * Convenience: advance time by the given number of seconds.
     *
     * @return the new monotonic time in nanoseconds
     */
    public long addSeconds(long seconds) {
        return add(seconds, TimeUnit.SECONDS);
    }

}
```

#### File: `src/main/java/dev/linhvu/micrometer/Counter.java` [NEW]

```java
package dev.linhvu.micrometer;

/**
 * A monotonically increasing value. The simplest meter type — it can only go up.
 * <p>
 * Stub interface: will be fully implemented in Feature 3 (Counter).
 */
public interface Counter extends Meter {

}
```

#### File: `src/main/java/dev/linhvu/micrometer/Gauge.java` [NEW]

```java
package dev.linhvu.micrometer;

/**
 * A meter that samples a current value — unlike Counter, it can go up and down.
 * <p>
 * Stub interface: will be fully implemented in Feature 5 (Gauge).
 */
public interface Gauge extends Meter {

}
```

#### File: `src/main/java/dev/linhvu/micrometer/Timer.java` [NEW]

```java
package dev.linhvu.micrometer;

/**
 * Records the duration and frequency of events — the most-used meter type for
 * measuring latency.
 * <p>
 * Stub interface: will be fully implemented in Feature 6 (Timer).
 */
public interface Timer extends Meter {

}
```

#### File: `src/main/java/dev/linhvu/micrometer/DistributionSummary.java` [NEW]

```java
package dev.linhvu.micrometer;

/**
 * Records the distribution of values (like request sizes or payload lengths).
 * Similar to Timer but for non-time measurements.
 * <p>
 * Stub interface: will be fully implemented in Feature 7 (DistributionSummary).
 */
public interface DistributionSummary extends Meter {

}
```

#### File: `src/main/java/dev/linhvu/micrometer/LongTaskTimer.java` [NEW]

```java
package dev.linhvu.micrometer;

/**
 * Tracks the duration of long-running tasks that are still in progress.
 * <p>
 * Stub interface: will be fully implemented in Feature 12 (LongTaskTimer).
 */
public interface LongTaskTimer extends Meter {

}
```

#### File: `src/main/java/dev/linhvu/micrometer/FunctionCounter.java` [NEW]

```java
package dev.linhvu.micrometer;

/**
 * A counter that derives its value from an external object via a function.
 * <p>
 * Stub interface: will be fully implemented in Feature 11 (Function Meters).
 */
public interface FunctionCounter extends Meter {

}
```

#### File: `src/main/java/dev/linhvu/micrometer/FunctionTimer.java` [NEW]

```java
package dev.linhvu.micrometer;

/**
 * A timer that derives count and total time from an external object via functions.
 * <p>
 * Stub interface: will be fully implemented in Feature 11 (Function Meters).
 */
public interface FunctionTimer extends Meter {

}
```

### Test Code

#### File: `src/test/java/dev/linhvu/micrometer/ClockTest.java` [NEW]

```java
package dev.linhvu.micrometer;

import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;

class ClockTest {

    @Test
    void shouldReturnSystemMillis_WhenUsingSystemClock() {
        long before = System.currentTimeMillis();
        long wallTime = Clock.SYSTEM.wallTime();
        long after = System.currentTimeMillis();

        assertThat(wallTime).isBetween(before, after);
    }

    @Test
    void shouldReturnSystemNanos_WhenUsingSystemClock() {
        long before = System.nanoTime();
        long monotonicTime = Clock.SYSTEM.monotonicTime();
        long after = System.nanoTime();

        assertThat(monotonicTime).isBetween(before, after);
    }

}
```

#### File: `src/test/java/dev/linhvu/micrometer/MockClockTest.java` [NEW]

```java
package dev.linhvu.micrometer;

import org.junit.jupiter.api.Test;

import java.time.Duration;
import java.util.concurrent.TimeUnit;

import static org.assertj.core.api.Assertions.assertThat;

class MockClockTest {

    @Test
    void shouldStartAtOneMillisecond_WhenCreated() {
        MockClock clock = new MockClock();

        assertThat(clock.wallTime()).isEqualTo(1L);
        assertThat(clock.monotonicTime()).isEqualTo(TimeUnit.MILLISECONDS.toNanos(1));
    }

    @Test
    void shouldAdvanceTime_WhenAddingWithTimeUnit() {
        MockClock clock = new MockClock();

        clock.add(5, TimeUnit.SECONDS);

        assertThat(clock.monotonicTime())
                .isEqualTo(TimeUnit.MILLISECONDS.toNanos(1) + TimeUnit.SECONDS.toNanos(5));
        assertThat(clock.wallTime()).isEqualTo(5001L);
    }

    @Test
    void shouldAdvanceTime_WhenAddingDuration() {
        MockClock clock = new MockClock();

        clock.add(Duration.ofSeconds(3));

        assertThat(clock.wallTime()).isEqualTo(3001L);
    }

    @Test
    void shouldAdvanceTime_WhenAddingSeconds() {
        MockClock clock = new MockClock();

        clock.addSeconds(10);

        assertThat(clock.wallTime()).isEqualTo(10_001L);
    }

    @Test
    void shouldKeepWallAndMonotonicInSync_WhenAdvancingTime() {
        MockClock clock = new MockClock();
        clock.add(2, TimeUnit.SECONDS);

        // Both should reflect the same underlying nanosecond counter
        long expectedNanos = TimeUnit.MILLISECONDS.toNanos(1) + TimeUnit.SECONDS.toNanos(2);
        assertThat(clock.monotonicTime()).isEqualTo(expectedNanos);
        assertThat(clock.wallTime()).isEqualTo(TimeUnit.NANOSECONDS.toMillis(expectedNanos));
    }

    @Test
    void shouldReturnNewMonotonicTime_WhenAddReturnsValue() {
        MockClock clock = new MockClock();

        long returned = clock.add(1, TimeUnit.SECONDS);

        assertThat(returned).isEqualTo(clock.monotonicTime());
    }

}
```

#### File: `src/test/java/dev/linhvu/micrometer/AbstractMeterTest.java` [NEW]

```java
package dev.linhvu.micrometer;

import org.junit.jupiter.api.Test;

import java.util.List;

import static org.assertj.core.api.Assertions.assertThat;

class AbstractMeterTest {

    /**
     * Concrete test implementation of AbstractMeter that returns a fixed measurement.
     */
    static class TestMeter extends AbstractMeter {

        private final double value;

        TestMeter(Meter.Id id, double value) {
            super(id);
            this.value = value;
        }

        @Override
        public Iterable<Measurement> measure() {
            return List.of(new Measurement(() -> value, Statistic.VALUE));
        }

    }

    private static Meter.Id id(String name) {
        return new Meter.Id(name, Tags.empty(), Meter.Type.OTHER, null, null);
    }

    private static Meter.Id id(String name, Tags tags) {
        return new Meter.Id(name, tags, Meter.Type.OTHER, null, null);
    }

    @Test
    void shouldReturnId_WhenGetIdCalled() {
        Meter.Id meterId = id("requests");
        TestMeter meter = new TestMeter(meterId, 42.0);

        assertThat(meter.getId()).isSameAs(meterId);
    }

    @Test
    void shouldReturnMeasurements_WhenMeasureCalled() {
        TestMeter meter = new TestMeter(id("cpu.usage"), 0.75);

        List<Measurement> measurements = (List<Measurement>) meter.measure();

        assertThat(measurements).hasSize(1);
        assertThat(measurements.get(0).getValue()).isEqualTo(0.75);
        assertThat(measurements.get(0).getStatistic()).isEqualTo(Statistic.VALUE);
    }

    @Test
    void shouldBeEqual_WhenMetersHaveSameNameAndTags() {
        TestMeter meter1 = new TestMeter(id("requests"), 1.0);
        TestMeter meter2 = new TestMeter(id("requests"), 99.0);

        assertThat(meter1).isEqualTo(meter2);
        assertThat(meter1.hashCode()).isEqualTo(meter2.hashCode());
    }

    @Test
    void shouldNotBeEqual_WhenMetersHaveDifferentNames() {
        TestMeter meter1 = new TestMeter(id("requests"), 1.0);
        TestMeter meter2 = new TestMeter(id("errors"), 1.0);

        assertThat(meter1).isNotEqualTo(meter2);
    }

    @Test
    void shouldNotBeEqual_WhenMetersHaveDifferentTags() {
        Tags tags1 = Tags.of("method", "GET");
        Tags tags2 = Tags.of("method", "POST");
        TestMeter meter1 = new TestMeter(id("requests", tags1), 1.0);
        TestMeter meter2 = new TestMeter(id("requests", tags2), 1.0);

        assertThat(meter1).isNotEqualTo(meter2);
    }

    @Test
    void shouldBeEqual_WhenTypesDifferButNameAndTagsMatch() {
        // This is deliberate — the registry detects collisions via equals()
        Meter.Id counterId = new Meter.Id("requests", Tags.empty(), Meter.Type.COUNTER, null, null);
        Meter.Id timerId = new Meter.Id("requests", Tags.empty(), Meter.Type.TIMER, null, null);

        TestMeter meter1 = new TestMeter(counterId, 1.0);
        TestMeter meter2 = new TestMeter(timerId, 1.0);

        assertThat(meter1).isEqualTo(meter2);
    }

    @Test
    void shouldIncludeIdInToString() {
        TestMeter meter = new TestMeter(id("http.requests"), 1.0);

        assertThat(meter.toString()).contains("TestMeter");
        assertThat(meter.toString()).contains("http.requests");
    }

}
```

#### File: `src/test/java/dev/linhvu/micrometer/MeterVisitorTest.java` [NEW]

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
        public Iterable<Measurement> measure() {
            return List.of(new Measurement(() -> 0.0, Statistic.COUNT));
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
| **`Meter`** | Root interface: every meter has an `Id` and produces `Measurement`s |
| **`Meter.Id`** | Name + tags = identity; type/description/baseUnit are metadata only |
| **`measure()`** | Returns the current set of measurements — always live, never stale |
| **`match()`/`use()`** | Functional visitor pattern for type-safe dispatching across meter types |
| **`Clock`** | Separates wall-clock time (ms) from monotonic time (ns) for testability |
| **`MockClock`** | Test double — single nanosecond counter drives both time domains |
| **`AbstractMeter`** | Base class: stores `Id`, implements identity-based `equals`/`hashCode` |

**Next: Chapter 3 — Counter** — The simplest meter type: a monotonically increasing value backed by `DoubleAdder` for lock-free concurrent increments. Introduces the Builder-to-register pattern that all meter types follow.
