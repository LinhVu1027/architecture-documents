# Chapter 4: MeterRegistry & SimpleMeterRegistry

> Line references based on commit `2c8a4606c` of the Micrometer repository.

## Build Challenge

| Current State | Limitation | Objective |
|---------------|------------|-----------|
| Counter, Gauge, Timer exist as interfaces with a `Builder` that accumulates configuration, but `Builder.register()` has no implementation — there's no central place to create, store, or deduplicate meters | Counters must be manually constructed via `new CumulativeCounter(id)` — there's no lifecycle management, no deduplication (same name+tags creates duplicate objects), and no way to query what meters exist | Build an abstract `MeterRegistry` with a ConcurrentHashMap-backed deduplication lifecycle, a `SimpleMeterRegistry` in-memory implementation, and wire `Counter.Builder.register()` as the bridge between user code and the registry |

---

## 4.1 The Integration Point: Counter.Builder gains `register()`

**Modifying:** `src/main/java/dev/linhvu/micrometer/Counter.java`
**Change:** Add `register(MeterRegistry)` method to the inner `Builder` class — the terminal operation that bridges the Builder pattern to the Registry lifecycle.

```java
/**
 * Registers this counter with the given registry, or retrieves the existing
 * one if a counter with the same name and tags was already registered.
 * <p>
 * This is the terminal operation of the builder pattern — it bridges
 * the Builder (user-facing API) to the MeterRegistry (lifecycle management).
 *
 * @param registry The registry to register with.
 * @return A new or existing Counter.
 */
public Counter register(MeterRegistry registry) {
    return registry.counter(new Meter.Id(name, tags, Meter.Type.COUNTER, description, baseUnit));
}
```

Two key decisions here:

1. **The Builder creates the `Meter.Id`, not the Registry.** The Builder knows its name, tags, type, description, and base unit — it assembles the identity and hands it to the registry. The registry's job is lifecycle management (deduplication, creation), not identity construction.

2. **`register()` calls a package-private `registry.counter(Meter.Id)` method.** This keeps the internal registration API hidden from users while allowing Builder (in the same package) to invoke it. Users call `Counter.builder("x").register(registry)` — they never see the internal `counter(Id)` method.

This connects **User Instrumentation Code** (the Builder) to **Meter Lifecycle Management** (the Registry). To make it work, we need to build:
- `MeterRegistry` — the abstract lifecycle manager with deduplication
- `SimpleMeterRegistry` — an in-memory implementation
- `NoopMeter` / `NoopCounter` — silent no-op meters for closed registries

## 4.2 NoopMeter and NoopCounter

Before building the registry, we need the "null object" meters it returns when the registry is closed.

**New file:** `src/main/java/dev/linhvu/micrometer/noop/NoopMeter.java`

```java
package dev.linhvu.micrometer.noop;

import dev.linhvu.micrometer.AbstractMeter;
import dev.linhvu.micrometer.Measurement;
import dev.linhvu.micrometer.Meter;

import java.util.Collections;

public class NoopMeter extends AbstractMeter {

    public NoopMeter(Meter.Id id) {
        super(id);
    }

    @Override
    public Iterable<Measurement> measure() {
        return Collections.emptyList();
    }

}
```

**New file:** `src/main/java/dev/linhvu/micrometer/noop/NoopCounter.java`

```java
package dev.linhvu.micrometer.noop;

import dev.linhvu.micrometer.Counter;
import dev.linhvu.micrometer.Meter;

public class NoopCounter extends NoopMeter implements Counter {

    public NoopCounter(Meter.Id id) {
        super(id);
    }

    @Override
    public void increment(double amount) {
        // no-op
    }

    @Override
    public double count() {
        return 0;
    }

}
```

The Null Object pattern ensures callers always get a non-null meter. Instrumentation code like `counter.increment()` never needs null checks — when the registry is closed, operations are silently absorbed.

## 4.3 MeterRegistry — The Heart of Micrometer

**New file:** `src/main/java/dev/linhvu/micrometer/MeterRegistry.java`

```java
package dev.linhvu.micrometer;

import dev.linhvu.micrometer.noop.NoopCounter;

import java.util.ArrayList;
import java.util.Collections;
import java.util.List;
import java.util.Objects;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.TimeUnit;
import java.util.function.Function;
import java.util.function.ToDoubleFunction;

public abstract class MeterRegistry {

    protected final Clock clock;

    private final ConcurrentHashMap<Meter.Id, Meter> meterMap = new ConcurrentHashMap<>();

    private final Object meterMapLock = new Object();

    private volatile boolean closed = false;

    protected MeterRegistry(Clock clock) {
        this.clock = Objects.requireNonNull(clock, "clock must not be null");
    }
```

The registry's fields tell the whole story:
- **`ConcurrentHashMap<Meter.Id, Meter>`** — meters keyed by identity (name+tags). ConcurrentHashMap gives lock-free reads.
- **`Object meterMapLock`** — dedicated lock for synchronized creation (not `this`, not the map — a private lock prevents external interference).
- **`volatile boolean closed`** — volatile ensures visibility across threads without synchronization.

### Public Convenience Methods

```java
    public Counter counter(String name, String... tags) {
        return Counter.builder(name).tags(tags).register(this);
    }

    public Counter counter(String name, Iterable<Tag> tags) {
        return Counter.builder(name).tags(tags).register(this);
    }
```

### Package-Private Registration Bridge

```java
    Counter counter(Meter.Id id) {
        return registerMeterIfNecessary(Counter.class, id, this::newCounter, NoopCounter::new);
    }
```

This is the method `Counter.Builder.register()` calls. It passes four things:
1. `Counter.class` — the expected type (for conflict detection)
2. `id` — the meter identity
3. `this::newCounter` — the template method for creating a real counter
4. `NoopCounter::new` — the factory for creating a no-op counter

### Template Method Factories

```java
    protected abstract Counter newCounter(Meter.Id id);

    protected abstract <T> Gauge newGauge(Meter.Id id, T obj, ToDoubleFunction<T> valueFunction);

    protected abstract Timer newTimer(Meter.Id id);

    protected abstract DistributionSummary newDistributionSummary(Meter.Id id);

    protected abstract TimeUnit getBaseTimeUnit();
```

This is the Template Method pattern: `MeterRegistry` defines the registration lifecycle, subclasses only implement factory methods. When Feature 5 adds Gauge, only `newGauge()` needs a real implementation.

### The Deduplication Algorithm

```java
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
```

The double-check locking pattern:
1. **Fast path** — lock-free `ConcurrentHashMap.get()`. After the first registration, this is all that runs — O(1) with no contention.
2. **Slow path** — `synchronized` block with a re-check. Multiple threads racing to create the same meter will serialize here, but only one creates it; the rest find it on re-check.
3. **Closed check** — inside the synchronized block so that `close()` + late `register()` don't race.

### Query and Lifecycle

```java
    public List<Meter> getMeters() {
        return Collections.unmodifiableList(new ArrayList<>(meterMap.values()));
    }

    public Clock getClock() {
        return clock;
    }

    public boolean isClosed() {
        return closed;
    }

    public void close() {
        closed = true;
    }

}
```

## 4.4 SimpleMeterRegistry

**New file:** `src/main/java/dev/linhvu/micrometer/simple/SimpleMeterRegistry.java`

```java
package dev.linhvu.micrometer.simple;

import dev.linhvu.micrometer.*;
import dev.linhvu.micrometer.cumulative.CumulativeCounter;

import java.util.concurrent.TimeUnit;
import java.util.function.ToDoubleFunction;

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
        throw new UnsupportedOperationException("Gauge not yet supported. See Feature 5.");
    }

    @Override
    protected Timer newTimer(Meter.Id id) {
        throw new UnsupportedOperationException("Timer not yet supported. See Feature 6.");
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

SimpleMeterRegistry is intentionally boring — that's the point of the Template Method pattern. The abstract base class handles all the complexity (deduplication, concurrency, lifecycle). This subclass just says "a counter is a `CumulativeCounter`" and "my time unit is seconds."

## 4.5 Try It Yourself

<details>
<summary>Challenge: Trace the full registration flow from builder to counter</summary>

Starting from this code:
```java
MeterRegistry registry = new SimpleMeterRegistry();
Counter counter = Counter.builder("http.requests")
    .tag("method", "GET")
    .register(registry);
```

Trace every method call in order. What happens when you call `register(registry)` a second time with the same name and tags?

**Answer:**

1. `Counter.Builder.register(registry)` creates `new Meter.Id("http.requests", Tags.of("method","GET"), COUNTER, null, null)`
2. Calls `registry.counter(id)` (package-private)
3. Calls `registerMeterIfNecessary(Counter.class, id, this::newCounter, NoopCounter::new)`
4. Calls `getOrCreateMeter(id, ...)`
5. **Fast path:** `meterMap.get(id)` returns `null` (first time)
6. **Slow path:** enters `synchronized(meterMapLock)`, double-checks → still null
7. Calls `meterFactory.apply(id)` → `SimpleMeterRegistry.newCounter(id)` → `new CumulativeCounter(id)`
8. Puts in `meterMap`, returns `CumulativeCounter`
9. `registerMeterIfNecessary` verifies `Counter.class.isInstance(counter)` → true, returns

On second call with same name+tags:
1-4. Same as above
5. **Fast path:** `meterMap.get(id)` returns the existing `CumulativeCounter` → returns immediately (never enters synchronized block)

</details>

<details>
<summary>Challenge: What happens if you try to register a Timer with the same name as an existing Counter?</summary>

When Feature 6 (Timer) is implemented, this will be the flow:

```java
registry.counter("my.metric");  // registers a CumulativeCounter
registry.timer("my.metric");    // tries to register a Timer with same name+tags
```

`registerMeterIfNecessary` would call `getOrCreateMeter` which returns the existing `CumulativeCounter`. Then it checks `Timer.class.isInstance(counter)` → **false** → throws `IllegalArgumentException`:

```
There is already a meter registered with name 'my.metric' and tags []
but with type CumulativeCounter. Attempted to register type Timer.
```

This is possible because `Meter.Id.equals()` only uses name+tags (ignoring type). So a Counter and Timer with the same name+tags have equal IDs → collision detected.

</details>

## 4.6 Tests

### Unit Tests

**New file:** `src/test/java/dev/linhvu/micrometer/MeterRegistryTest.java`

```java
class MeterRegistryTest {

    @Test
    void shouldRegisterCounter_WhenUsingBuilder() { ... }

    @Test
    void shouldReturnSameInstance_WhenCounterRegisteredTwice() { ... }

    @Test
    void shouldReturnSameInstance_WhenCounterRegisteredWithSameTags() { ... }

    @Test
    void shouldCreateDistinctCounters_WhenDifferentNames() { ... }

    @Test
    void shouldCreateDistinctCounters_WhenDifferentTags() { ... }

    @Test
    void shouldRegisterCounter_WhenUsingConvenienceMethodWithVarargs() { ... }

    @Test
    void shouldRegisterCounter_WhenUsingConvenienceMethodWithTags() { ... }

    @Test
    void shouldReturnSameInstance_WhenConvenienceAndBuilderUseSameNameAndTags() { ... }

    @Test
    void shouldReturnEmptyList_WhenNoMetersRegistered() { ... }

    @Test
    void shouldReturnAllRegisteredMeters_WhenGetMetersCalled() { ... }

    @Test
    void shouldReturnUnmodifiableList_WhenGetMetersCalled() { ... }

    @Test
    void shouldNotBeClosed_WhenNewlyCreated() { ... }

    @Test
    void shouldBeClosed_AfterClose() { ... }

    @Test
    void shouldReturnNoopCounter_WhenRegistryClosed() { ... }

    @Test
    void shouldReturnExistingCounter_WhenRegisteredBeforeClose() { ... }

    @Test
    void shouldExposeClock_WhenGetClockCalled() { ... }
}
```

### Integration Tests

**New file:** `src/test/java/dev/linhvu/micrometer/integration/RegistryIntegrationTest.java`

```java
class RegistryIntegrationTest {

    @Test
    void shouldIncrementRegisteredCounter_WhenAccessedByBuilder() { ... }

    @Test
    void shouldShareState_WhenCounterRegisteredMultipleTimesAndIncremented() { ... }

    @Test
    void shouldRegisterMultipleCounters_WhenDifferentNamesUsed() { ... }

    @Test
    void shouldDeduplicateCorrectly_WhenConcurrentRegistration() { ... }

    @Test
    void shouldWorkWithMockClock_WhenTestingTimeDependentBehavior() { ... }
}
```

**Run:** `./gradlew test` — expected: all 116 tests pass (including all prior features' tests)

---

## 4.7 Why This Works

> ★ **Insight** -------------------------------------------
> - **Why Template Method over Strategy?** The Template Method pattern (abstract `newCounter()` in a base class) works perfectly here because the registration lifecycle is always the same — only the _creation_ of meters differs across backends. Strategy would let you swap meter creation at runtime, but registries don't change their backend type after construction. Template Method gives compile-time enforcement: if you add a new meter type, the abstract method forces every registry subclass to handle it.
> - **When Template Method hurts:** If you needed multiple orthogonal variation points (e.g., different deduplication strategies AND different meter types), Template Method creates a combinatorial explosion of subclasses. At that point, compose with Strategy. Micrometer avoids this by keeping deduplication fixed and only varying meter creation.
> -----------------------------------------------------------

> ★ **Insight** -------------------------------------------
> - **Why double-check locking instead of `ConcurrentHashMap.computeIfAbsent()`?** The simpler approach would be `meterMap.computeIfAbsent(id, this::newCounter)`. But `computeIfAbsent` holds a bucket lock during the lambda execution — if the factory method triggers another `computeIfAbsent` on the same map (e.g., creating a synthetic histogram gauge), you get a deadlock. The explicit `synchronized(meterMapLock)` block avoids this because it's a separate lock from the map's internal locks.
> - **The real Micrometer hit this exact bug.** That's why `getOrCreateMeter` at `MeterRegistry.java:688` uses a dedicated lock object rather than relying on ConcurrentHashMap's built-in atomicity.
> -----------------------------------------------------------

> ★ **Insight** -------------------------------------------
> - **Why does the Null Object pattern matter here?** Without `NoopCounter`, closing a registry would force a choice: return `null` (every call site needs null checks) or throw (instrumentation crashes the app). Neither is acceptable for a metrics library — metrics should never affect application behavior. The Null Object pattern means `counter.increment()` is always safe to call, even after shutdown.
> - **Real-world impact:** In production, applications may shut down gracefully by closing the registry before all threads have finished. Without noop meters, late `counter.increment()` calls would throw NPE. With them, metrics silently drain away.
> -----------------------------------------------------------

## 4.8 What We Enhanced

| Aspect | Before (ch03) | Current (ch04) | Real Framework |
|--------|---------------|----------------|----------------|
| Counter creation | Manual: `new CumulativeCounter(id)` — user constructs directly | `Counter.builder("x").register(registry)` — registry manages lifecycle | Same builder→register pattern (`Counter.java:143`) |
| Deduplication | None — each `new CumulativeCounter(id)` creates a separate object | ConcurrentHashMap keyed by `Meter.Id` — same name+tags always returns same instance | Three-layer lookup: pre-filter cache → post-filter map → synchronized creation (`MeterRegistry.java:688`) |
| Closed state | N/A — no registry concept | `close()` sets flag; new registrations return `NoopCounter` | Same + graceful shutdown with final publish for push registries (`MeterRegistry.java:225`) |
| Thread safety | `CumulativeCounter` uses `DoubleAdder` (thread-safe recording) | Registry adds thread-safe registration via double-check locking | Same pattern with additional stale-ID handling for late-added filters (`MeterRegistry.java:692`) |

## 4.9 Connection to Real Framework

| Simplified Code | Real Framework Code | File:Line | Key Difference |
|----------------|--------------------|-----------| ---------------|
| `MeterRegistry` | `MeterRegistry` | `MeterRegistry.java:84` | Real has 1339 lines with MeterFilter chain, NamingConvention, PauseDetector, listener callbacks, synthetic associations, stale-ID tracking, and a `More` inner class for advanced meter types |
| `getOrCreateMeter()` | `getOrCreateMeter()` | `MeterRegistry.java:688` | Real has 3-layer lookup (pre-filter map → filtered map → synchronized creation) and applies `DistributionStatisticConfig` merging |
| `registerMeterIfNecessary()` | `registerMeterIfNecessary()` | `MeterRegistry.java:650` | Real has two overloads — one for simple meters, one with `DistributionStatisticConfig` and `PauseDetector` |
| `meterMap` (single `ConcurrentHashMap`) | `meterMap` + `preFilterIdToMeterMap` + `meterToPreFilterIdMap` | `MeterRegistry.java:107,114,119` | Real uses three maps: primary (post-filter ID → Meter), pre-filter cache (original ID → Meter), and reverse lookup (Meter → original ID) |
| `SimpleMeterRegistry` | `SimpleMeterRegistry` | `SimpleMeterRegistry.java:44` | Real supports both CUMULATIVE and STEP modes via `SimpleConfig`, registers `HistogramGauges` for timers/summaries |
| `NoopCounter` | `NoopCounter` | `NoopCounter.java:20` | Identical pattern — the real version is the same 11-line class |
| `Counter.Builder.register()` | `Counter.Builder.register()` | `Counter.java:143` | Real calls `registry.createId()` which applies common tags from config; our version creates the ID directly |

## 4.10 Complete Code

### Production Code

#### File: `src/main/java/dev/linhvu/micrometer/MeterRegistry.java` [NEW]

```java
package dev.linhvu.micrometer;

import dev.linhvu.micrometer.noop.NoopCounter;

import java.util.ArrayList;
import java.util.Collections;
import java.util.List;
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

    // -----------------------------------------------------------------------
    // Package-private registration (called by Builder.register())
    // -----------------------------------------------------------------------

    /**
     * Registers a counter using the pre-built ID. Called by {@link Counter.Builder#register(MeterRegistry)}.
     */
    Counter counter(Meter.Id id) {
        return registerMeterIfNecessary(Counter.class, id, this::newCounter, NoopCounter::new);
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

#### File: `src/main/java/dev/linhvu/micrometer/simple/SimpleMeterRegistry.java` [NEW]

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
 * Currently supports Counter (via {@link CumulativeCounter}). Gauge, Timer, and
 * DistributionSummary factories will be implemented in Features 5, 6, and 7
 * respectively.
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
        throw new UnsupportedOperationException("Gauge not yet supported. See Feature 5.");
    }

    @Override
    protected Timer newTimer(Meter.Id id) {
        throw new UnsupportedOperationException("Timer not yet supported. See Feature 6.");
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

#### File: `src/main/java/dev/linhvu/micrometer/noop/NoopMeter.java` [NEW]

```java
package dev.linhvu.micrometer.noop;

import dev.linhvu.micrometer.AbstractMeter;
import dev.linhvu.micrometer.Measurement;
import dev.linhvu.micrometer.Meter;

import java.util.Collections;

/**
 * Base class for all no-op meter implementations. Returns empty measurements
 * and silently ignores all recording operations.
 * <p>
 * No-op meters are returned in two situations:
 * <ul>
 *     <li>When the registry has been {@linkplain dev.linhvu.micrometer.MeterRegistry#close() closed}</li>
 *     <li>When a {@code MeterFilter} denies the meter (Feature 8)</li>
 * </ul>
 * This ensures callers always receive a non-null meter and can call recording
 * methods without null checks — the operations simply do nothing.
 */
public class NoopMeter extends AbstractMeter {

    public NoopMeter(Meter.Id id) {
        super(id);
    }

    @Override
    public Iterable<Measurement> measure() {
        return Collections.emptyList();
    }

}
```

#### File: `src/main/java/dev/linhvu/micrometer/noop/NoopCounter.java` [NEW]

```java
package dev.linhvu.micrometer.noop;

import dev.linhvu.micrometer.Counter;
import dev.linhvu.micrometer.Meter;

/**
 * A counter that does nothing. {@code increment()} is silently ignored and
 * {@code count()} always returns zero.
 * <p>
 * Returned by the registry when the registry is closed or a filter denies
 * the meter. This guarantees that instrumentation code like
 * {@code counter.increment()} never throws or requires null checks.
 */
public class NoopCounter extends NoopMeter implements Counter {

    public NoopCounter(Meter.Id id) {
        super(id);
    }

    @Override
    public void increment(double amount) {
        // no-op
    }

    @Override
    public double count() {
        return 0;
    }

}
```

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
     * The terminal operation {@link #register(MeterRegistry)} creates or retrieves a
     * counter from the registry.
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

        /**
         * Registers this counter with the given registry, or retrieves the existing
         * one if a counter with the same name and tags was already registered.
         * <p>
         * This is the terminal operation of the builder pattern — it bridges
         * the Builder (user-facing API) to the MeterRegistry (lifecycle management).
         *
         * @param registry The registry to register with.
         * @return A new or existing Counter.
         */
        public Counter register(MeterRegistry registry) {
            return registry.counter(new Meter.Id(name, tags, Meter.Type.COUNTER, description, baseUnit));
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

### Test Code

#### File: `src/test/java/dev/linhvu/micrometer/MeterRegistryTest.java` [NEW]

```java
package dev.linhvu.micrometer;

import dev.linhvu.micrometer.cumulative.CumulativeCounter;
import dev.linhvu.micrometer.noop.NoopCounter;
import dev.linhvu.micrometer.simple.SimpleMeterRegistry;
import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

/**
 * Tests the abstract MeterRegistry lifecycle: deduplication, type-conflict detection,
 * closed-state behavior, and convenience registration methods. Uses SimpleMeterRegistry
 * as the concrete implementation.
 */
class MeterRegistryTest {

    // --- Counter registration via Builder ---

    @Test
    void shouldRegisterCounter_WhenUsingBuilder() {
        MeterRegistry registry = new SimpleMeterRegistry();

        Counter counter = Counter.builder("http.requests")
                .tag("method", "GET")
                .description("Total HTTP requests")
                .register(registry);

        assertThat(counter).isNotNull();
        assertThat(counter).isInstanceOf(CumulativeCounter.class);
        assertThat(counter.getId().getName()).isEqualTo("http.requests");
        assertThat(counter.getId().getTag("method")).isEqualTo("GET");
    }

    @Test
    void shouldReturnSameInstance_WhenCounterRegisteredTwice() {
        MeterRegistry registry = new SimpleMeterRegistry();

        Counter first = Counter.builder("requests").register(registry);
        Counter second = Counter.builder("requests").register(registry);

        assertThat(first).isSameAs(second);
    }

    @Test
    void shouldReturnSameInstance_WhenCounterRegisteredWithSameTags() {
        MeterRegistry registry = new SimpleMeterRegistry();

        Counter first = Counter.builder("requests").tag("method", "GET").register(registry);
        Counter second = Counter.builder("requests").tag("method", "GET").register(registry);

        assertThat(first).isSameAs(second);
    }

    @Test
    void shouldCreateDistinctCounters_WhenDifferentNames() {
        MeterRegistry registry = new SimpleMeterRegistry();

        Counter a = Counter.builder("requests").register(registry);
        Counter b = Counter.builder("errors").register(registry);

        assertThat(a).isNotSameAs(b);
        a.increment();
        assertThat(a.count()).isEqualTo(1.0);
        assertThat(b.count()).isEqualTo(0.0);
    }

    @Test
    void shouldCreateDistinctCounters_WhenDifferentTags() {
        MeterRegistry registry = new SimpleMeterRegistry();

        Counter get = Counter.builder("requests").tag("method", "GET").register(registry);
        Counter post = Counter.builder("requests").tag("method", "POST").register(registry);

        assertThat(get).isNotSameAs(post);
        get.increment(5);
        post.increment(3);
        assertThat(get.count()).isEqualTo(5.0);
        assertThat(post.count()).isEqualTo(3.0);
    }

    // --- Convenience registration ---

    @Test
    void shouldRegisterCounter_WhenUsingConvenienceMethodWithVarargs() {
        MeterRegistry registry = new SimpleMeterRegistry();

        Counter counter = registry.counter("requests", "method", "GET", "status", "200");

        assertThat(counter).isNotNull();
        assertThat(counter.getId().getName()).isEqualTo("requests");
        assertThat(counter.getId().getTag("method")).isEqualTo("GET");
        assertThat(counter.getId().getTag("status")).isEqualTo("200");
    }

    @Test
    void shouldRegisterCounter_WhenUsingConvenienceMethodWithTags() {
        MeterRegistry registry = new SimpleMeterRegistry();
        Tags tags = Tags.of("region", "us-east");

        Counter counter = registry.counter("requests", tags);

        assertThat(counter).isNotNull();
        assertThat(counter.getId().getTag("region")).isEqualTo("us-east");
    }

    @Test
    void shouldReturnSameInstance_WhenConvenienceAndBuilderUseSameNameAndTags() {
        MeterRegistry registry = new SimpleMeterRegistry();

        Counter fromBuilder = Counter.builder("requests").tag("method", "GET").register(registry);
        Counter fromConvenience = registry.counter("requests", "method", "GET");

        assertThat(fromBuilder).isSameAs(fromConvenience);
    }

    // --- getMeters() ---

    @Test
    void shouldReturnEmptyList_WhenNoMetersRegistered() {
        MeterRegistry registry = new SimpleMeterRegistry();
        assertThat(registry.getMeters()).isEmpty();
    }

    @Test
    void shouldReturnAllRegisteredMeters_WhenGetMetersCalled() {
        MeterRegistry registry = new SimpleMeterRegistry();
        Counter a = registry.counter("requests");
        Counter b = registry.counter("errors");

        assertThat(registry.getMeters()).hasSize(2);
        assertThat(registry.getMeters()).contains(a, b);
    }

    @Test
    void shouldReturnUnmodifiableList_WhenGetMetersCalled() {
        MeterRegistry registry = new SimpleMeterRegistry();
        registry.counter("requests");

        assertThatThrownBy(() -> registry.getMeters().add(null))
                .isInstanceOf(UnsupportedOperationException.class);
    }

    // --- Closed state ---

    @Test
    void shouldNotBeClosed_WhenNewlyCreated() {
        MeterRegistry registry = new SimpleMeterRegistry();
        assertThat(registry.isClosed()).isFalse();
    }

    @Test
    void shouldBeClosed_AfterClose() {
        MeterRegistry registry = new SimpleMeterRegistry();
        registry.close();
        assertThat(registry.isClosed()).isTrue();
    }

    @Test
    void shouldReturnNoopCounter_WhenRegistryClosed() {
        MeterRegistry registry = new SimpleMeterRegistry();
        registry.close();

        Counter counter = Counter.builder("requests").register(registry);

        assertThat(counter).isInstanceOf(NoopCounter.class);
        // Noop counter should silently ignore operations
        counter.increment(100);
        assertThat(counter.count()).isEqualTo(0.0);
    }

    @Test
    void shouldReturnExistingCounter_WhenRegisteredBeforeClose() {
        MeterRegistry registry = new SimpleMeterRegistry();

        Counter before = Counter.builder("requests").register(registry);
        before.increment(5);

        registry.close();

        // Registering the same counter again returns the existing one (not noop)
        Counter after = Counter.builder("requests").register(registry);
        assertThat(after).isSameAs(before);
        assertThat(after.count()).isEqualTo(5.0);
    }

    // --- Clock ---

    @Test
    void shouldExposeClock_WhenGetClockCalled() {
        MockClock clock = new MockClock();
        MeterRegistry registry = new SimpleMeterRegistry(clock);

        assertThat(registry.getClock()).isSameAs(clock);
    }

}
```

#### File: `src/test/java/dev/linhvu/micrometer/simple/SimpleMeterRegistryTest.java` [NEW]

```java
package dev.linhvu.micrometer.simple;

import dev.linhvu.micrometer.Clock;
import dev.linhvu.micrometer.Counter;
import dev.linhvu.micrometer.MockClock;
import dev.linhvu.micrometer.cumulative.CumulativeCounter;
import org.junit.jupiter.api.Test;

import java.util.concurrent.TimeUnit;

import static org.assertj.core.api.Assertions.assertThat;

/**
 * Tests SimpleMeterRegistry-specific behavior: the factory method returns
 * CumulativeCounter, uses seconds as the base time unit, and supports
 * both no-arg and Clock-arg construction.
 */
class SimpleMeterRegistryTest {

    @Test
    void shouldCreateCumulativeCounter_WhenCounterRegistered() {
        SimpleMeterRegistry registry = new SimpleMeterRegistry();
        Counter counter = Counter.builder("test.counter").register(registry);

        assertThat(counter).isInstanceOf(CumulativeCounter.class);
    }

    @Test
    void shouldUseSecondsAsBaseTimeUnit() {
        SimpleMeterRegistry registry = new SimpleMeterRegistry();
        assertThat(registry.getBaseTimeUnit()).isEqualTo(TimeUnit.SECONDS);
    }

    @Test
    void shouldUseSystemClock_WhenNoArgConstructor() {
        SimpleMeterRegistry registry = new SimpleMeterRegistry();
        assertThat(registry.getClock()).isSameAs(Clock.SYSTEM);
    }

    @Test
    void shouldUseProvidedClock_WhenClockArgConstructor() {
        MockClock clock = new MockClock();
        SimpleMeterRegistry registry = new SimpleMeterRegistry(clock);
        assertThat(registry.getClock()).isSameAs(clock);
    }

}
```

#### File: `src/test/java/dev/linhvu/micrometer/noop/NoopCounterTest.java` [NEW]

```java
package dev.linhvu.micrometer.noop;

import dev.linhvu.micrometer.Counter;
import dev.linhvu.micrometer.Measurement;
import dev.linhvu.micrometer.Meter;
import dev.linhvu.micrometer.Tags;
import org.junit.jupiter.api.Test;

import java.util.List;

import static org.assertj.core.api.Assertions.assertThat;

/**
 * Tests the NoopCounter — verifies that all operations are silently ignored
 * and that measurement returns empty results.
 */
class NoopCounterTest {

    private static Meter.Id noopId(String name) {
        return new Meter.Id(name, Tags.empty(), Meter.Type.COUNTER, null, null);
    }

    @Test
    void shouldDoNothing_WhenIncremented() {
        Counter counter = new NoopCounter(noopId("noop"));
        counter.increment();
        counter.increment(100.0);
        assertThat(counter.count()).isEqualTo(0.0);
    }

    @Test
    void shouldReturnZero_WhenCountQueried() {
        Counter counter = new NoopCounter(noopId("noop"));
        assertThat(counter.count()).isEqualTo(0.0);
    }

    @Test
    void shouldReturnEmptyMeasurements_WhenMeasured() {
        NoopMeter meter = new NoopMeter(noopId("noop"));
        Iterable<Measurement> measurements = meter.measure();
        assertThat(measurements).isEmpty();
    }

    @Test
    void shouldPreserveId_WhenCreated() {
        Meter.Id id = noopId("my.counter");
        Counter counter = new NoopCounter(id);
        assertThat(counter.getId()).isSameAs(id);
        assertThat(counter.getId().getName()).isEqualTo("my.counter");
    }

    @Test
    void shouldBeInstanceOfCounter_WhenTypeChecked() {
        Counter counter = new NoopCounter(noopId("noop"));
        assertThat(counter).isInstanceOf(Counter.class);
        assertThat(counter).isInstanceOf(Meter.class);
        assertThat(counter).isInstanceOf(NoopMeter.class);
    }

    @Test
    void shouldReturnCountStatistic_WhenNoopCounterMeasured() {
        // NoopCounter inherits measure() from Counter interface (returns COUNT),
        // but NoopMeter overrides measure() to return empty — NoopCounter gets NoopMeter's
        // empty measure() because it extends NoopMeter which has the concrete override
        Counter counter = new NoopCounter(noopId("noop"));
        List<Measurement> measurements = (List<Measurement>) counter.measure();
        assertThat(measurements).isEmpty();
    }

}
```

#### File: `src/test/java/dev/linhvu/micrometer/integration/RegistryIntegrationTest.java` [NEW]

```java
package dev.linhvu.micrometer.integration;

import dev.linhvu.micrometer.Counter;
import dev.linhvu.micrometer.MeterRegistry;
import dev.linhvu.micrometer.MockClock;
import dev.linhvu.micrometer.simple.SimpleMeterRegistry;
import org.junit.jupiter.api.Test;

import java.util.ArrayList;
import java.util.List;
import java.util.concurrent.CountDownLatch;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

import static org.assertj.core.api.Assertions.assertThat;

/**
 * Integration tests verifying end-to-end behavior of the registry with counters,
 * including the Builder→Registry→CumulativeCounter lifecycle and thread safety.
 */
class RegistryIntegrationTest {

    @Test
    void shouldIncrementRegisteredCounter_WhenAccessedByBuilder() {
        MeterRegistry registry = new SimpleMeterRegistry();

        Counter counter = Counter.builder("http.requests")
                .tag("method", "GET")
                .tag("status", "200")
                .description("Total HTTP requests")
                .baseUnit("requests")
                .register(registry);

        counter.increment();
        counter.increment();
        counter.increment(3.0);

        assertThat(counter.count()).isEqualTo(5.0);
        assertThat(registry.getMeters()).hasSize(1);
    }

    @Test
    void shouldShareState_WhenCounterRegisteredMultipleTimesAndIncremented() {
        MeterRegistry registry = new SimpleMeterRegistry();

        // Two different code paths register the same counter
        Counter counter1 = Counter.builder("requests").tag("method", "GET").register(registry);
        Counter counter2 = Counter.builder("requests").tag("method", "GET").register(registry);

        counter1.increment(10);
        counter2.increment(5);

        // Both references point to the same counter — total is 15
        assertThat(counter1.count()).isEqualTo(15.0);
        assertThat(counter2.count()).isEqualTo(15.0);
        assertThat(counter1).isSameAs(counter2);
        assertThat(registry.getMeters()).hasSize(1);
    }

    @Test
    void shouldRegisterMultipleCounters_WhenDifferentNamesUsed() {
        MeterRegistry registry = new SimpleMeterRegistry();

        Counter requests = registry.counter("http.requests");
        Counter errors = registry.counter("http.errors");
        Counter latency = registry.counter("processing.count");

        requests.increment(100);
        errors.increment(5);
        latency.increment(50);

        assertThat(registry.getMeters()).hasSize(3);
        assertThat(requests.count()).isEqualTo(100.0);
        assertThat(errors.count()).isEqualTo(5.0);
        assertThat(latency.count()).isEqualTo(50.0);
    }

    @Test
    void shouldDeduplicateCorrectly_WhenConcurrentRegistration() throws InterruptedException {
        MeterRegistry registry = new SimpleMeterRegistry();
        int threadCount = 10;
        CountDownLatch latch = new CountDownLatch(threadCount);
        List<Counter> counters = new ArrayList<>();

        ExecutorService executor = Executors.newFixedThreadPool(threadCount);
        for (int i = 0; i < threadCount; i++) {
            executor.submit(() -> {
                try {
                    Counter c = Counter.builder("concurrent.counter").register(registry);
                    synchronized (counters) {
                        counters.add(c);
                    }
                    c.increment();
                } finally {
                    latch.countDown();
                }
            });
        }

        latch.await();
        executor.shutdown();

        // All threads should get the same counter instance
        assertThat(counters).hasSize(threadCount);
        Counter first = counters.get(0);
        for (Counter c : counters) {
            assertThat(c).isSameAs(first);
        }

        // Total increments = threadCount
        assertThat(first.count()).isEqualTo((double) threadCount);
        assertThat(registry.getMeters()).hasSize(1);
    }

    @Test
    void shouldWorkWithMockClock_WhenTestingTimeDependentBehavior() {
        MockClock clock = new MockClock();
        MeterRegistry registry = new SimpleMeterRegistry(clock);

        Counter counter = Counter.builder("test").register(registry);
        counter.increment();

        assertThat(counter.count()).isEqualTo(1.0);
        assertThat(registry.getClock()).isSameAs(clock);
    }

}
```

---

## Summary

| Concept | What It Means |
|---------|---------------|
| **Template Method** | Abstract class defines the algorithm skeleton (registration lifecycle); subclasses implement specific steps (`newCounter()`, `newGauge()`, etc.) |
| **Double-check locking** | Lock-free fast path (ConcurrentHashMap.get) + synchronized slow path with re-check — gives thread-safe deduplication without contention on reads |
| **Null Object (Noop meters)** | Silent no-op implementations returned when the registry is closed — instrumentation code never needs null checks |
| **Builder → Registry bridge** | `Counter.Builder.register(registry)` creates the `Meter.Id` and calls the registry's package-private `counter(Id)` method — clean separation between user API and lifecycle management |

**Next: Chapter 5 — Gauge** — A meter that samples a current value from an external object via `WeakReference`, preventing memory leaks while monitoring collection sizes, queue depths, and live object counts.
