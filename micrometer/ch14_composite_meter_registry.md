# Chapter 14: CompositeMeterRegistry & Global Facade

> Line references based on commit `2c8a4606c` of the Micrometer repository.

## Build Challenge

| Current State | Limitation | Objective |
|---------------|------------|-----------|
| The registry supports all meter types, filters, naming conventions, distribution statistics, function meters, LongTaskTimer, and MeterBinder — but each `MeterRegistry` is a standalone island. Applications that export to Datadog AND Prometheus need two registries and must duplicate every `counter.increment()` call to both | No way to register a meter once and have it automatically appear in multiple monitoring backends | Build a `CompositeMeterRegistry` that fans out writes to multiple child registries, plus a static `Metrics` facade for use where dependency injection is unavailable |

---

## 14.1 The Integration Point: `meterAdded()` Hook in MeterRegistry

The composite pattern requires two-way coordination between the registry and its meters. When a user registers a counter on a composite that already has child registries, the composite needs to create a real counter in each child. This requires a lifecycle hook in `MeterRegistry`.

**Modifying:** `src/main/java/dev/linhvu/micrometer/MeterRegistry.java`
**Change:** Add a `meterAdded(Meter)` hook method called from `getOrCreateMeter()` after a new meter is stored, plus a `removeByPreFilterId()` method for cleanup.

```java
// Inside getOrCreateMeter(), after creating the meter:
Meter newMeter = meterFactory.apply(id);
meterMap.put(id, newMeter);
meterAdded(newMeter);  // ← NEW: lifecycle hook
return newMeter;
```

```java
// New protected hook method:
protected void meterAdded(Meter meter) {
    // Default: do nothing
}

// New removal method:
public Meter removeByPreFilterId(Meter.Id id) {
    return meterMap.remove(id);
}
```

Two key decisions here:

1. **Why a protected template method instead of a callback list?** The real Micrometer uses `onMeterAdded(Consumer<Meter>)` callbacks configured via `config()`. We use a simpler protected method — `CompositeMeterRegistry` overrides it to propagate new meters to all children. Same behavior, less machinery.

2. **Called inside the synchronized block.** The hook fires while holding `meterMapLock`, which is safe because the hook's work targets *different* registries (each child has its own lock). This guarantees that a meter is never visible to `getMeters()` before it's been propagated to all children.

This connects the **MeterRegistry lifecycle** to the **CompositeMeterRegistry**. To make it work, we need to build:
- `CompositeMeter` — interface that composite meters implement
- `AbstractCompositeMeter<T>` — copy-on-write base with children map and fan-out logic
- `CompositeCounter/Timer/Gauge/...` — one per meter type
- `CompositeMeterRegistry` — orchestrates the fan-out
- `Metrics` — static global facade

## 14.2 CompositeMeter Interface

**New file:** `src/main/java/dev/linhvu/micrometer/composite/CompositeMeter.java`

```java
package dev.linhvu.micrometer.composite;

import dev.linhvu.micrometer.Meter;
import dev.linhvu.micrometer.MeterRegistry;

public interface CompositeMeter extends Meter {

    void add(MeterRegistry registry);

    void remove(MeterRegistry registry);

}
```

Two methods — `add()` creates a real child meter in a registry, `remove()` cleans it up. Every composite meter type implements this.

## 14.3 AbstractCompositeMeter — The Copy-on-Write Base

**New file:** `src/main/java/dev/linhvu/micrometer/composite/AbstractCompositeMeter.java`

This is the core of the composite pattern. It manages a map from `MeterRegistry` to child meter using copy-on-write for thread safety:

```java
public abstract class AbstractCompositeMeter<T extends Meter> extends AbstractMeter
        implements CompositeMeter {

    private volatile Map<MeterRegistry, T> children = Collections.emptyMap();
    private volatile T noopMeter;

    protected T firstChild() {
        Collection<T> values = children.values();
        if (values.isEmpty()) {
            if (noopMeter == null) {
                noopMeter = newNoopMeter();
            }
            return noopMeter;
        }
        return values.iterator().next();
    }

    protected Collection<T> getChildren() {
        return children.values();
    }

    protected abstract T newNoopMeter();
    protected abstract T registerNewMeter(MeterRegistry registry);

    @Override
    public synchronized void add(MeterRegistry registry) {
        T meter = registerNewMeter(registry);
        if (meter != null) {
            Map<MeterRegistry, T> newChildren = new IdentityHashMap<>(children);
            newChildren.put(registry, meter);
            children = Collections.unmodifiableMap(newChildren);
        }
    }

    @Override
    public synchronized void remove(MeterRegistry registry) {
        Map<MeterRegistry, T> newChildren = new IdentityHashMap<>(children);
        newChildren.remove(registry);
        children = newChildren.isEmpty()
                ? Collections.emptyMap()
                : Collections.unmodifiableMap(newChildren);
    }
}
```

Three critical design choices:

1. **`IdentityHashMap`** — Registry identity (not equality) determines uniqueness. Two registries that happen to be `equals()` are separate children.
2. **`volatile` + copy-on-write** — Write methods (`add`/`remove`) are `synchronized` and create a new map. Read methods (`firstChild`/`getChildren`) just read the volatile field — no locking.
3. **Two abstract template methods** — `newNoopMeter()` creates a fallback for when no children exist, `registerNewMeter()` uses the builder API to create a real meter in a child registry.

## 14.4 Composite Meter Implementations

Each meter type gets a composite implementation. They all follow the same pattern: **writes fan out to all children, reads come from `firstChild()`**.

### CompositeCounter

**New file:** `src/main/java/dev/linhvu/micrometer/composite/CompositeCounter.java`

```java
class CompositeCounter extends AbstractCompositeMeter<Counter> implements Counter {

    CompositeCounter(Meter.Id id) {
        super(id);
    }

    @Override
    public void increment(double amount) {
        for (Counter child : getChildren()) {  // fan out writes
            child.increment(amount);
        }
    }

    @Override
    public double count() {
        return firstChild().count();  // read from one child
    }

    @Override
    protected Counter newNoopMeter() {
        return new NoopCounter(getId());
    }

    @Override
    protected Counter registerNewMeter(MeterRegistry registry) {
        return Counter.builder(getId().getName())
                .tags(getId().getTagsAsIterable())
                .description(getId().getDescription())
                .baseUnit(getId().getBaseUnit())
                .register(registry);
    }
}
```

### CompositeGauge

**New file:** `src/main/java/dev/linhvu/micrometer/composite/CompositeGauge.java`

Uses `WeakReference` to the observed object. If the object is garbage collected, `registerNewMeter()` returns null and the child is not added.

```java
class CompositeGauge<S> extends AbstractCompositeMeter<Gauge> implements Gauge {

    private final WeakReference<S> objRef;
    private final ToDoubleFunction<S> f;

    CompositeGauge(Meter.Id id, S obj, ToDoubleFunction<S> f) {
        super(id);
        this.objRef = new WeakReference<>(obj);
        this.f = f;
    }

    @Override
    public double value() {
        return firstChild().value();
    }

    @Override
    protected Gauge registerNewMeter(MeterRegistry registry) {
        S obj = objRef.get();
        if (obj == null) return null;  // GC'd — skip
        return Gauge.builder(getId().getName(), obj, f)
                .tags(getId().getTagsAsIterable())
                .register(registry);
    }
}
```

### CompositeTimer

**New file:** `src/main/java/dev/linhvu/micrometer/composite/CompositeTimer.java`

The most complex composite meter. For convenience methods (`record(Runnable)`, `record(Supplier)`), the timing is done once at the composite level, then the recorded duration is fanned out. This avoids each child independently timing the same operation.

```java
class CompositeTimer extends AbstractCompositeMeter<Timer> implements Timer {

    private final Clock clock;
    private final DistributionStatisticConfig distributionStatisticConfig;

    @Override
    public void record(long amount, TimeUnit unit) {
        for (Timer child : getChildren()) {
            child.record(amount, unit);
        }
    }

    @Override
    public <T> T record(Supplier<T> f) {
        long start = clock.monotonicTime();
        try {
            return f.get();
        } finally {
            long durationNanos = clock.monotonicTime() - start;
            record(durationNanos, TimeUnit.NANOSECONDS);  // single timing, fan out once
        }
    }
    // ... similar for recordCallable, record(Runnable)
}
```

### CompositeLongTaskTimer

**New file:** `src/main/java/dev/linhvu/micrometer/composite/CompositeLongTaskTimer.java`

Unlike other composites, `start()` creates a `Sample` in EVERY child timer and wraps them in a `CompositeSample`. Stopping the composite sample stops all children.

```java
@Override
public Sample start() {
    Collection<LongTaskTimer> children = getChildren();
    List<Sample> childSamples = new ArrayList<>(children.size());
    for (LongTaskTimer child : children) {
        childSamples.add(child.start());
    }
    return new CompositeSample(childSamples);
}
```

### CompositeFunctionCounter, CompositeFunctionTimer, CompositeTimeGauge

These follow the same pattern as `CompositeGauge` — hold a `WeakReference` to the state object, and if GC'd, `registerNewMeter()` returns null. `CompositeFunctionCounter.count()` reads directly from the weak-referenced object rather than from the first child.

## 14.5 CompositeMeterRegistry — The Orchestrator

**New file:** `src/main/java/dev/linhvu/micrometer/composite/CompositeMeterRegistry.java`

The central class that ties everything together:

```java
public class CompositeMeterRegistry extends MeterRegistry {

    private final Set<MeterRegistry> registries = new LinkedHashSet<>();

    public CompositeMeterRegistry(Clock clock) {
        super(clock);
        namingConvention(NamingConvention.identity);  // each child uses its own
    }

    public CompositeMeterRegistry add(MeterRegistry registry) {
        synchronized (registries) { registries.add(registry); }
        // Retroactively propagate existing meters
        for (Meter meter : getMeters()) {
            if (meter instanceof CompositeMeter composite) {
                composite.add(registry);
            }
        }
        return this;
    }

    @Override
    protected void meterAdded(Meter meter) {
        // Propagate new meter to all existing children
        if (meter instanceof CompositeMeter composite) {
            Set<MeterRegistry> snapshot;
            synchronized (registries) { snapshot = Set.copyOf(registries); }
            for (MeterRegistry registry : snapshot) {
                composite.add(registry);
            }
        }
    }

    @Override
    protected Counter newCounter(Meter.Id id) {
        return new CompositeCounter(id);
    }
    // ... factory methods for all other meter types

    @Override
    protected DistributionStatisticConfig defaultHistogramConfig() {
        return DistributionStatisticConfig.NONE;  // don't inject defaults
    }
}
```

Two propagation paths handle all timing scenarios:
1. **Meter registered first, then child added** — `add(MeterRegistry)` iterates existing meters
2. **Child already present, then new meter registered** — `meterAdded()` propagates to existing children

## 14.6 Metrics — The Static Global Facade

**New file:** `src/main/java/dev/linhvu/micrometer/Metrics.java`

A static utility class backed by a global `CompositeMeterRegistry`:

```java
public final class Metrics {

    public static final CompositeMeterRegistry globalRegistry = new CompositeMeterRegistry();

    public static void addRegistry(MeterRegistry registry) {
        globalRegistry.add(registry);
    }

    public static Counter counter(String name, String... tags) {
        return globalRegistry.counter(name, tags);
    }

    public static Timer timer(String name, String... tags) {
        return globalRegistry.timer(name, tags);
    }
    // ... delegates for all meter types
}
```

Meters created before any registry is added are composite meters with no children — reads return zero. As soon as a registry is added, those meters are retroactively propagated to it.

---

## 14.7 Try It Yourself

<details><summary>Challenge 1: Build a composite that fans out to three registries and verify all receive increments</summary>

```java
CompositeMeterRegistry composite = new CompositeMeterRegistry();
SimpleMeterRegistry r1 = new SimpleMeterRegistry();
SimpleMeterRegistry r2 = new SimpleMeterRegistry();
SimpleMeterRegistry r3 = new SimpleMeterRegistry();

composite.add(r1).add(r2).add(r3);

Counter counter = composite.counter("http.requests", "method", "GET");
counter.increment(100);

// All three should have 100
assertThat(r1.counter("http.requests", "method", "GET").count()).isEqualTo(100);
assertThat(r2.counter("http.requests", "method", "GET").count()).isEqualTo(100);
assertThat(r3.counter("http.requests", "method", "GET").count()).isEqualTo(100);
```

</details>

<details><summary>Challenge 2: Register meters before adding children, then verify retroactive propagation</summary>

```java
CompositeMeterRegistry composite = new CompositeMeterRegistry();

// Register BEFORE any child
Counter counter = composite.counter("early.counter");
Timer timer = composite.timer("early.timer");
counter.increment(50);
timer.record(200, TimeUnit.MILLISECONDS);

// Now add a child — should retroactively get both meters
SimpleMeterRegistry child = new SimpleMeterRegistry();
composite.add(child);

// Future writes go to child
counter.increment(10);
timer.record(100, TimeUnit.MILLISECONDS);

assertThat(child.counter("early.counter").count()).isEqualTo(10);
assertThat(child.timer("early.timer").count()).isEqualTo(1);
```

Note: the pre-child increments (50 and 200ms) are lost — the child only sees writes that happen after it was added.

</details>

<details><summary>Challenge 3: Use the Metrics facade to instrument a service method</summary>

```java
Metrics.addRegistry(new SimpleMeterRegistry());

public String processRequest(String input) {
    return Metrics.timer("request.processing").record(() -> {
        Metrics.counter("requests.total").increment();
        // Process input...
        return "processed: " + input;
    });
}

// The timer records duration, the counter tracks invocations
// Both are backed by the same SimpleMeterRegistry
```

</details>

---

## 14.8 Tests

### Unit Tests

#### CompositeMeterRegistryTest

```java
@Test
void shouldFanOutCounterIncrements_WhenMultipleChildRegistries() {
    composite.add(child1).add(child2);
    Counter counter = composite.counter("test.counter");
    counter.increment(5.0);
    assertThat(child1.counter("test.counter").count()).isEqualTo(5.0);
    assertThat(child2.counter("test.counter").count()).isEqualTo(5.0);
}

@Test
void shouldPropagateExistingMeters_WhenChildAddedAfterRegistration() {
    Counter counter = composite.counter("test.counter");
    counter.increment(7.0);
    composite.add(child1);
    counter.increment(3.0);
    assertThat(child1.counter("test.counter").count()).isEqualTo(3.0);
}

@Test
void shouldStopFanningOutToRemovedChild_WhenChildRemoved() {
    composite.add(child1).add(child2);
    Counter counter = composite.counter("test.counter");
    counter.increment(5.0);
    composite.remove(child2);
    counter.increment(3.0);
    assertThat(child1.counter("test.counter").count()).isEqualTo(8.0);
    assertThat(child2.counter("test.counter").count()).isEqualTo(5.0);
}
```

#### CompositeCounterTest, CompositeTimerTest, CompositeGaugeTest

Individual tests for each composite meter type verifying fan-out writes, single-source reads, tag preservation, and noop behavior when no children exist.

#### MetricsTest

```java
@Test
void shouldCreateCounterViaStaticMethod_WhenCalled() {
    Metrics.addRegistry(testRegistry);
    Counter counter = Metrics.counter("global.counter", "env", "test");
    counter.increment(3.0);
    assertThat(testRegistry.counter("global.counter", "env", "test").count()).isEqualTo(3.0);
}

@Test
void shouldPropagateToNewRegistry_WhenAddedAfterMeterCreation() {
    Counter counter = Metrics.counter("early.counter");
    counter.increment(10.0);
    SimpleMeterRegistry lateRegistry = new SimpleMeterRegistry();
    Metrics.addRegistry(lateRegistry);
    counter.increment(5.0);
    assertThat(lateRegistry.counter("early.counter").count()).isEqualTo(5.0);
}
```

### Integration Tests

```java
@Test
void shouldApplyFiltersAtCompositeLevel_WhenFilterConfigured() {
    composite.meterFilter(MeterFilter.commonTags(Tags.of("app", "myapp")));
    composite.add(child1);
    Counter counter = composite.counter("test.counter");
    assertThat(counter.getId().getTag("app")).isEqualTo("myapp");
}

@Test
void shouldPropagateBoundMetersToLateChild_WhenChildAddedAfterBind() {
    new UptimeMetrics().bindTo(composite);
    composite.add(child1);
    assertThat(child1.getMeters().stream().map(m -> m.getId().getName()))
            .contains("process.uptime", "process.start.time");
}
```

**Run:** `./gradlew test` — expected: all 570 tests pass (including all prior features' tests)

---

## 14.9 Why This Works

> ★ **Insight** -------------------------------------------
> **The Composite Pattern operates at two levels.** `CompositeMeterRegistry` is a composite of *registries* (it manages a set of child registries). Each `CompositeCounter`/`CompositeTimer`/etc. is a composite of *individual meters* (it manages a map of child meters). The registry orchestrates the lifecycle — when a child registry is added, the composite propagates to each composite meter; when a meter is created, the composite propagates to each child registry. This two-level design cleanly separates "which backends exist" from "which meters exist."
>
> **Trade-off:** The read path always returns data from the first child only. If child registries have different retention windows or processing pipelines, the composite's read values reflect only one backend's view. For most use cases this is fine — the composite is a write multiplexer, not a read aggregator.
> -----------------------------------------------------------

> ★ **Insight** -------------------------------------------
> **Copy-on-write gives the write path zero-cost reads.** The children map is `volatile` and updated by creating a new `IdentityHashMap`. Write operations (`increment()`, `record()`) iterate the volatile reference without locking — they see a consistent snapshot even if `add()`/`remove()` is happening concurrently. The cost is paid on mutation (creating a new map), which is rare (only when registries are added/removed). The hot path — recording metrics — pays no synchronization cost at all.
>
> The real Micrometer uses a CAS spin-lock (`AtomicBoolean.compareAndSet`) instead of `synchronized` for even lower overhead. The principle is the same: optimize for the read/write path, accept higher cost on the rare mutation path.
> -----------------------------------------------------------

> ★ **Insight** -------------------------------------------
> **The Metrics facade enables "instrument once, decide later."** Library authors can call `Metrics.counter("lib.calls").increment()` without knowing which monitoring backend the application uses. If no registry is added, the composite silently absorbs all operations (noop). When the application adds a Prometheus registry at startup, all existing meters are retroactively propagated — the library's metrics instantly appear in Prometheus without code changes. This is the core value proposition of Micrometer: *decouple instrumentation from export*.
> -----------------------------------------------------------

## 14.10 What We Enhanced

| Aspect | Before (ch13) | Current (ch14) | Real Framework |
|--------|---------------|----------------|----------------|
| Multi-backend export | Not possible — each `MeterRegistry` is standalone | `CompositeMeterRegistry.add()` fans out to multiple children | Same — `CompositeMeterRegistry.java:137-151`, plus nested composite support |
| Meter lifecycle | No hook after creation | `meterAdded(Meter)` template method called after storage | Real uses `config().onMeterAdded(Consumer)` callback — `MeterRegistry.java:615-620` |
| Static global API | Not available | `Metrics.counter()`/`timer()` backed by `globalRegistry` | Same pattern — `Metrics.java:40-260` |
| Identity naming | Always uses configured naming convention | Composite uses `NamingConvention.identity`; children apply their own | Same — `CompositeMeterRegistry.java:66` |
| Histogram config cascade | Composite would inject registry defaults | `defaultHistogramConfig()` returns `NONE` — children apply their own | Same — `CompositeMeterRegistry.java:100` |

## 14.11 Connection to Real Framework

| Simplified Code | Real Framework Code | File:Line | Key Difference |
|----------------|--------------------|-----------| ---------------|
| `CompositeMeter` | `CompositeMeter` | `CompositeMeter.java:28-30` | Identical interface |
| `AbstractCompositeMeter` | `AbstractCompositeMeter` | `AbstractCompositeMeter.java:30-107` | Real uses `AtomicBoolean` CAS spin-lock instead of `synchronized`; caches noop via `Supplier<T>` |
| `CompositeMeterRegistry` | `CompositeMeterRegistry` | `CompositeMeterRegistry.java:42-231` | Real supports nested composites (tree flattening via `updateDescendants()`), parent-child tracking, and `forbidSelfContainingComposite()` circular reference detection |
| `CompositeMeterRegistry.meterAdded()` | `config().onMeterAdded()` callback | `CompositeMeterRegistry.java:63-76` | Real uses callback registration via `Config` class; we use protected template method |
| `CompositeTimer` | `CompositeTimer` | `CompositeTimer.java:33-149` | Real also passes `PauseDetector` through builder; handles `percentileHistogram` config |
| `CompositeLongTaskTimer.CompositeSample` | `CompositeLongTaskTimer.CompositeSample` | `CompositeLongTaskTimer.java:55-82` | Same pattern — wraps child samples, stop fans out to all |
| `Metrics` | `Metrics` | `Metrics.java:40-260` | Real has more overloads and includes `More` as a static field rather than a static inner class |
| `CompositeMeterRegistry.remove()` | `CompositeMeterRegistry.remove()` | `CompositeMeterRegistry.java:153-170` | Real also calls `removeByPreFilterId()` to clean up child registries; we leave child meters intact |

## 14.12 Complete Code

### Production Code

#### File: `src/main/java/dev/linhvu/micrometer/MeterRegistry.java` [MODIFIED]

```java
package dev.linhvu.micrometer;

import dev.linhvu.micrometer.config.MeterFilter;
import dev.linhvu.micrometer.config.MeterFilterReply;
import dev.linhvu.micrometer.config.NamingConvention;
import dev.linhvu.micrometer.distribution.DistributionStatisticConfig;
import dev.linhvu.micrometer.noop.NoopCounter;
import dev.linhvu.micrometer.noop.NoopDistributionSummary;
import dev.linhvu.micrometer.noop.NoopFunctionCounter;
import dev.linhvu.micrometer.noop.NoopFunctionTimer;
import dev.linhvu.micrometer.noop.NoopGauge;
import dev.linhvu.micrometer.noop.NoopLongTaskTimer;
import dev.linhvu.micrometer.noop.NoopTimeGauge;
import dev.linhvu.micrometer.noop.NoopTimer;

import java.util.ArrayList;
import java.util.Arrays;
import java.util.Collection;
import java.util.Collections;
import java.util.List;
import java.util.Map;
import java.util.Objects;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.TimeUnit;
import java.util.function.Function;
import java.util.function.ToDoubleFunction;
import java.util.function.ToLongFunction;

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

    /**
     * Volatile array of filters — enables lock-free iteration during meter registration
     * while still allowing thread-safe mutation via {@link #meterFilter(MeterFilter)}.
     * Copy-on-write: adding a filter creates a new array.
     */
    private volatile MeterFilter[] filters = new MeterFilter[0];

    private volatile NamingConvention namingConvention = NamingConvention.identity;

    private volatile boolean closed = false;

    protected MeterRegistry(Clock clock) {
        this.clock = Objects.requireNonNull(clock, "clock must not be null");
    }

    // -----------------------------------------------------------------------
    // NamingConvention configuration
    // -----------------------------------------------------------------------

    /**
     * Returns the naming convention used by this registry when exporting metric
     * names and tag keys/values.
     */
    public NamingConvention getNamingConvention() {
        return namingConvention;
    }

    /**
     * Sets the naming convention for this registry. Subclasses (backend registries)
     * typically set a default in their constructor (e.g., snakeCase for Prometheus).
     *
     * @param namingConvention The convention to use.
     * @return This registry, for chaining.
     */
    public MeterRegistry namingConvention(NamingConvention namingConvention) {
        this.namingConvention = Objects.requireNonNull(namingConvention, "namingConvention must not be null");
        return this;
    }

    // -----------------------------------------------------------------------
    // Filter configuration
    // -----------------------------------------------------------------------

    /**
     * Adds a meter filter to this registry. Filters are applied in the order they
     * are added, forming a chain:
     * <ol>
     *   <li>{@link MeterFilter#map(Meter.Id)} — transforms the ID (all filters)</li>
     *   <li>{@link MeterFilter#accept(Meter.Id)} — accept/deny decision (short-circuits)</li>
     * </ol>
     * <p>
     * Filters should be configured before any meters are registered. Filters added
     * after meters are registered will only affect new registrations.
     *
     * @param filter The filter to add.
     * @return This registry, for chaining.
     */
    public synchronized MeterRegistry meterFilter(MeterFilter filter) {
        Objects.requireNonNull(filter, "filter must not be null");
        MeterFilter[] newFilters = Arrays.copyOf(filters, filters.length + 1);
        newFilters[filters.length] = filter;
        filters = newFilters;
        return this;
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

    /**
     * Creates or retrieves a distribution summary with the given name and tags.
     * @param name The summary name.
     * @param tags Must be an even number of arguments representing key/value pairs.
     * @return A new or existing DistributionSummary.
     */
    public DistributionSummary summary(String name, String... tags) {
        return DistributionSummary.builder(name).tags(tags).register(this);
    }

    /**
     * Creates or retrieves a distribution summary with the given name and tags.
     * @param name The summary name.
     * @param tags The tag collection.
     * @return A new or existing DistributionSummary.
     */
    public DistributionSummary summary(String name, Iterable<Tag> tags) {
        return DistributionSummary.builder(name).tags(tags).register(this);
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
     * Registers a timer using the pre-built ID and distribution config.
     * Called by {@link Timer.Builder#register(MeterRegistry)}.
     * <p>
     * The config goes through three stages:
     * <ol>
     *   <li>Builder-provided config (from user)</li>
     *   <li>MeterFilter.configure() chain (per-meter overrides)</li>
     *   <li>Merge with {@link #defaultHistogramConfig()} (registry defaults)</li>
     * </ol>
     */
    Timer timer(Meter.Id id, DistributionStatisticConfig distributionStatisticConfig) {
        return registerMeterIfNecessary(Timer.class, id,
                mappedId -> newTimer(mappedId,
                        configureDistributionStatistics(mappedId, distributionStatisticConfig)),
                NoopTimer::new);
    }

    /**
     * Registers a distribution summary using the pre-built ID, scale, and distribution config.
     * Called by {@link DistributionSummary.Builder#register(MeterRegistry)}.
     */
    DistributionSummary summary(Meter.Id id, double scale,
            DistributionStatisticConfig distributionStatisticConfig) {
        return registerMeterIfNecessary(DistributionSummary.class, id,
                mappedId -> newDistributionSummary(mappedId, scale,
                        configureDistributionStatistics(mappedId, distributionStatisticConfig)),
                NoopDistributionSummary::new);
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
     *
     * @param id                          The meter identity.
     * @param distributionStatisticConfig The fully-merged distribution config.
     */
    protected abstract Timer newTimer(Meter.Id id,
            DistributionStatisticConfig distributionStatisticConfig);

    /**
     * Create a new distribution summary implementation for this registry.
     *
     * @param id                          The meter identity.
     * @param scale                       Factor to scale each recorded value by.
     * @param distributionStatisticConfig The fully-merged distribution config.
     */
    protected abstract DistributionSummary newDistributionSummary(Meter.Id id, double scale,
            DistributionStatisticConfig distributionStatisticConfig);

    /**
     * Create a new long task timer implementation for this registry.
     *
     * @param id                          The meter identity.
     * @param distributionStatisticConfig The fully-merged distribution config.
     */
    protected abstract LongTaskTimer newLongTaskTimer(Meter.Id id,
            DistributionStatisticConfig distributionStatisticConfig);

    /**
     * Create a new function counter implementation for this registry.
     * Called only when no function counter with this ID exists yet.
     */
    protected abstract <T> FunctionCounter newFunctionCounter(Meter.Id id, T obj,
            ToDoubleFunction<T> countFunction);

    /**
     * Create a new function timer implementation for this registry.
     * Called only when no function timer with this ID exists yet.
     */
    protected abstract <T> FunctionTimer newFunctionTimer(Meter.Id id, T obj,
            ToLongFunction<T> countFunction, ToDoubleFunction<T> totalTimeFunction,
            TimeUnit totalTimeFunctionUnit);

    /**
     * Create a new time gauge implementation for this registry.
     * <p>
     * Default implementation delegates to {@code newGauge()} with time-unit
     * conversion built in. Subclasses can override for richer support.
     */
    protected <T> TimeGauge newTimeGauge(Meter.Id id, T obj, TimeUnit timeFunctionUnit,
            ToDoubleFunction<T> f) {
        return new dev.linhvu.micrometer.internal.DefaultTimeGauge<>(id, obj, timeFunctionUnit,
                f, getBaseTimeUnit());
    }

    /**
     * Returns the base time unit for this registry. Backends that work in
     * seconds (e.g., Prometheus) return {@link TimeUnit#SECONDS}; backends
     * that work in milliseconds return {@link TimeUnit#MILLISECONDS}.
     */
    protected abstract TimeUnit getBaseTimeUnit();

    /**
     * Returns the default distribution statistic configuration for this registry.
     * Used as the final fallback in the cascading merge.
     * <p>
     * Subclasses can override this to provide backend-specific defaults
     * (e.g., a Prometheus registry might enable percentile histograms by default).
     */
    protected DistributionStatisticConfig defaultHistogramConfig() {
        return DistributionStatisticConfig.DEFAULT;
    }

    /**
     * Stage 3 of the filter pipeline: configures distribution statistics.
     * <p>
     * Each filter can modify the config (e.g., set SLO boundaries for specific meters).
     * The filter's result is merged with the existing config — filter values take
     * priority over previously-set values. Finally, the result is merged with
     * {@link #defaultHistogramConfig()} to fill in any remaining nulls.
     */
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

    // -----------------------------------------------------------------------
    // Registration lifecycle — the heart of deduplication
    // -----------------------------------------------------------------------

    /**
     * Registers a meter if one with the same ID does not already exist. If a meter
     * with the same name+tags already exists but is a different type, throws
     * {@link IllegalArgumentException}.
     * <p>
     * The filter chain runs here: the ID is mapped (transformed), then the accept
     * decision is checked. If denied, a no-op meter is returned immediately without
     * being stored in the registry.
     *
     * @param meterClass    The expected meter type (e.g., Counter.class)
     * @param id            The meter's identity (name + tags)
     * @param meterFactory  Factory to create the real meter (calls a template method)
     * @param noopFactory   Factory to create a no-op meter (for closed/denied meters)
     * @return The existing or newly created meter, cast to M
     */
    private <M extends Meter> M registerMeterIfNecessary(Class<M> meterClass, Meter.Id id,
            Function<Meter.Id, ? extends Meter> meterFactory,
            Function<Meter.Id, ? extends Meter> noopFactory) {

        // Stage 1: MAP — transform the ID through the filter chain
        Meter.Id mappedId = mapId(id);

        // Stage 2: ACCEPT — check if the meter should be registered
        if (!accept(mappedId)) {
            return meterClass.cast(noopFactory.apply(mappedId));
        }

        Meter meter = getOrCreateMeter(mappedId, meterFactory, noopFactory);

        if (!meterClass.isInstance(meter)) {
            throw new IllegalArgumentException(
                    "There is already a meter registered with name '" + mappedId.getName()
                            + "' and tags " + mappedId.getTagsAsIterable()
                            + " but with type " + meter.getClass().getSimpleName()
                            + ". Attempted to register type " + meterClass.getSimpleName() + ".");
        }

        return meterClass.cast(meter);
    }

    /**
     * Stage 1 of the filter pipeline: transforms the meter ID through all filters.
     * Each filter's output becomes the next filter's input.
     */
    private Meter.Id mapId(Meter.Id id) {
        Meter.Id mappedId = id;
        for (MeterFilter filter : filters) {
            mappedId = filter.map(mappedId);
        }
        return mappedId;
    }

    /**
     * Stage 2 of the filter pipeline: determines whether to create a real meter
     * or a no-op. Uses short-circuit evaluation — the first non-NEUTRAL reply wins.
     * If all filters return NEUTRAL, the meter is accepted.
     */
    private boolean accept(Meter.Id id) {
        for (MeterFilter filter : filters) {
            MeterFilterReply reply = filter.accept(id);
            if (reply == MeterFilterReply.DENY) {
                return false;
            }
            if (reply == MeterFilterReply.ACCEPT) {
                return true;
            }
        }
        // All filters returned NEUTRAL → accept by default
        return true;
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
            meterAdded(newMeter);
            return newMeter;
        }
    }

    // -----------------------------------------------------------------------
    // Meter lifecycle hook
    // -----------------------------------------------------------------------

    /**
     * Called after a new meter is successfully created and stored in the registry.
     * Subclasses can override to react to new meter registrations.
     * <p>
     * Used by {@code CompositeMeterRegistry} to propagate new meters to child registries.
     *
     * @param meter The newly created meter.
     */
    protected void meterAdded(Meter meter) {
        // Default: do nothing
    }

    // -----------------------------------------------------------------------
    // Meter removal
    // -----------------------------------------------------------------------

    /**
     * Removes a meter from this registry by its pre-filter (raw) ID.
     * Used by {@code CompositeMeterRegistry} when a child registry is removed,
     * to clean up meters that were propagated to it.
     *
     * @param id The meter ID to remove.
     * @return The removed meter, or null if not found.
     */
    public Meter removeByPreFilterId(Meter.Id id) {
        return meterMap.remove(id);
    }

    // -----------------------------------------------------------------------
    // "More" accessor — less common meter types
    // -----------------------------------------------------------------------

    private final More more = new More();

    /**
     * Access less common meter types through a dedicated namespace.
     * <p>
     * This keeps the main registry API clean (counter/gauge/timer/summary) while
     * providing a discoverable path to advanced types:
     * <pre>
     *     registry.more().counter("cache.evictions", cache, Cache::evictionCount);
     *     registry.more().timer("pool.wait", pool, Pool::waitCount, Pool::totalWaitTime, MILLISECONDS);
     *     registry.more().timeGauge("jvm.uptime", runtime, MILLISECONDS, RuntimeMXBean::getUptime);
     * </pre>
     */
    public More more() {
        return more;
    }

    /**
     * Provides registration methods for less common meter types: {@link FunctionCounter},
     * {@link FunctionTimer}, and {@link TimeGauge}.
     * <p>
     * These meters derive their values from external objects via functions — they are
     * essential for instrumenting libraries that already maintain their own statistics.
     */
    public class More {

        /**
         * Register a function counter that monitors a monotonically increasing
         * function on the given object.
         *
         * @param id            The meter identity.
         * @param obj           The state object to observe.
         * @param countFunction Function that extracts the count.
         * @param <T>           The type of the state object.
         * @return A new or existing FunctionCounter.
         */
        <T> FunctionCounter counter(Meter.Id id, T obj, ToDoubleFunction<T> countFunction) {
            return registerMeterIfNecessary(FunctionCounter.class, id,
                    mappedId -> newFunctionCounter(mappedId, obj, countFunction),
                    NoopFunctionCounter::new);
        }

        /**
         * Register a function counter that monitors a monotonically increasing
         * function on the given object.
         *
         * @param name          The counter name.
         * @param tags          Tags for the counter.
         * @param obj           The state object to observe.
         * @param countFunction Function that extracts the count.
         * @param <T>           The type of the state object.
         * @return A new or existing FunctionCounter.
         */
        public <T> FunctionCounter counter(String name, Iterable<Tag> tags, T obj,
                ToDoubleFunction<T> countFunction) {
            return FunctionCounter.builder(name, obj, countFunction).tags(tags).register(MeterRegistry.this);
        }

        /**
         * Register a function timer that monitors count and total time functions
         * on the given object.
         *
         * @param id                    The meter identity.
         * @param obj                   The state object to observe.
         * @param countFunction         Function that extracts the event count.
         * @param totalTimeFunction     Function that extracts the total time.
         * @param totalTimeFunctionUnit The time unit of the total time function.
         * @param <T>                   The type of the state object.
         * @return A new or existing FunctionTimer.
         */
        <T> FunctionTimer timer(Meter.Id id, T obj, ToLongFunction<T> countFunction,
                ToDoubleFunction<T> totalTimeFunction, TimeUnit totalTimeFunctionUnit) {
            return registerMeterIfNecessary(FunctionTimer.class,
                    id.withBaseUnit(getBaseTimeUnit().name().toLowerCase()),
                    mappedId -> newFunctionTimer(mappedId, obj, countFunction,
                            totalTimeFunction, totalTimeFunctionUnit),
                    NoopFunctionTimer::new);
        }

        /**
         * Register a function timer that monitors count and total time functions
         * on the given object.
         *
         * @param name                  The timer name.
         * @param tags                  Tags for the timer.
         * @param obj                   The state object to observe.
         * @param countFunction         Function that extracts the event count.
         * @param totalTimeFunction     Function that extracts the total time.
         * @param totalTimeFunctionUnit The time unit of the total time function.
         * @param <T>                   The type of the state object.
         * @return A new or existing FunctionTimer.
         */
        public <T> FunctionTimer timer(String name, Iterable<Tag> tags, T obj,
                ToLongFunction<T> countFunction, ToDoubleFunction<T> totalTimeFunction,
                TimeUnit totalTimeFunctionUnit) {
            return FunctionTimer.builder(name, obj, countFunction, totalTimeFunction,
                    totalTimeFunctionUnit).tags(tags).register(MeterRegistry.this);
        }

        /**
         * Register a time gauge that monitors a time-valued function on the given object.
         *
         * @param id               The meter identity.
         * @param obj              The state object to observe.
         * @param timeFunctionUnit The time unit of the function's output.
         * @param timeFunction     Function that extracts a time value.
         * @param <T>              The type of the state object.
         * @return A new or existing TimeGauge.
         */
        <T> TimeGauge timeGauge(Meter.Id id, T obj, TimeUnit timeFunctionUnit,
                ToDoubleFunction<T> timeFunction) {
            return registerMeterIfNecessary(TimeGauge.class, id,
                    mappedId -> newTimeGauge(mappedId, obj, timeFunctionUnit, timeFunction),
                    NoopTimeGauge::new);
        }

        /**
         * Register a time gauge that monitors a time-valued function on the given object.
         *
         * @param name             The time gauge name.
         * @param tags             Tags for the time gauge.
         * @param obj              The state object to observe.
         * @param timeFunctionUnit The time unit of the function's output.
         * @param timeFunction     Function that extracts a time value.
         * @param <T>              The type of the state object.
         * @return A new or existing TimeGauge.
         */
        public <T> TimeGauge timeGauge(String name, Iterable<Tag> tags, T obj,
                TimeUnit timeFunctionUnit, ToDoubleFunction<T> timeFunction) {
            return TimeGauge.builder(name, obj, timeFunctionUnit, timeFunction)
                    .tags(tags).register(MeterRegistry.this);
        }

        /**
         * Register a long task timer using a pre-built ID and distribution config.
         * Called by {@link LongTaskTimer.Builder#register(MeterRegistry)}.
         *
         * @param id                          The meter identity.
         * @param distributionStatisticConfig The distribution config from the builder.
         * @return A new or existing LongTaskTimer.
         */
        LongTaskTimer longTaskTimer(Meter.Id id, DistributionStatisticConfig distributionStatisticConfig) {
            return registerMeterIfNecessary(LongTaskTimer.class, id,
                    mappedId -> newLongTaskTimer(mappedId,
                            configureDistributionStatistics(mappedId, distributionStatisticConfig)),
                    NoopLongTaskTimer::new);
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

#### File: `src/main/java/dev/linhvu/micrometer/composite/CompositeMeter.java` [NEW]

```java
package dev.linhvu.micrometer.composite;

import dev.linhvu.micrometer.Meter;
import dev.linhvu.micrometer.MeterRegistry;

/**
 * A meter that fans out operations to child meters in multiple registries.
 * <p>
 * When a {@link CompositeMeterRegistry} adds a new child registry, it calls
 * {@link #add(MeterRegistry)} on every existing composite meter, which creates
 * a real meter in that child registry. When a child is removed, {@link #remove(MeterRegistry)}
 * cleans up the corresponding child meter.
 */
public interface CompositeMeter extends Meter {

    /**
     * Register a real child meter in the given registry.
     *
     * @param registry The child registry to add a meter to.
     */
    void add(MeterRegistry registry);

    /**
     * Remove the child meter for the given registry.
     *
     * @param registry The child registry to remove from.
     */
    void remove(MeterRegistry registry);

}
```

#### File: `src/main/java/dev/linhvu/micrometer/composite/AbstractCompositeMeter.java` [NEW]

```java
package dev.linhvu.micrometer.composite;

import dev.linhvu.micrometer.AbstractMeter;
import dev.linhvu.micrometer.Meter;
import dev.linhvu.micrometer.MeterRegistry;

import java.util.Collection;
import java.util.Collections;
import java.util.IdentityHashMap;
import java.util.Map;

/**
 * Abstract base for composite meters that fan out operations to child meters
 * in multiple registries.
 * <p>
 * Uses <b>copy-on-write</b> on a volatile {@link IdentityHashMap} for thread safety:
 * {@link #add(MeterRegistry)} and {@link #remove(MeterRegistry)} create a new map
 * snapshot, while read operations ({@link #firstChild()}, {@link #getChildren()})
 * see a consistent snapshot without locking.
 * <p>
 * {@link IdentityHashMap} is used so registry identity (not equality) determines
 * uniqueness — two registries that happen to be {@code equals()} are still treated
 * as separate children.
 * <p>
 * <b>Simplification:</b> The real Micrometer uses a CAS spin-lock
 * ({@code AtomicBoolean.compareAndSet}) for non-blocking concurrency. We use
 * {@code synchronized} for clarity.
 *
 * @param <T> The concrete meter type this composite wraps (e.g., Counter, Timer).
 */
public abstract class AbstractCompositeMeter<T extends Meter> extends AbstractMeter implements CompositeMeter {

    /**
     * Maps each child registry to the real meter registered within it.
     * Volatile + copy-on-write ensures safe concurrent reads during iteration.
     */
    private volatile Map<MeterRegistry, T> children = Collections.emptyMap();

    private volatile T noopMeter;

    protected AbstractCompositeMeter(Meter.Id id) {
        super(id);
    }

    /**
     * Returns the first child meter for read operations, or a cached noop meter
     * if no children exist. This is the read path — queries like {@code count()}
     * or {@code value()} delegate here.
     */
    protected T firstChild() {
        Collection<T> values = children.values();
        if (values.isEmpty()) {
            if (noopMeter == null) {
                noopMeter = newNoopMeter();
            }
            return noopMeter;
        }
        return values.iterator().next();
    }

    /**
     * Returns an unmodifiable snapshot of all child meters for write operations.
     * This is the write path — operations like {@code increment()} or {@code record()}
     * iterate this collection to fan out to all children.
     */
    protected Collection<T> getChildren() {
        return children.values();
    }

    /**
     * Create a noop meter to use as the read fallback when no children exist.
     * Subclasses return the appropriate noop type (NoopCounter, NoopTimer, etc.).
     */
    protected abstract T newNoopMeter();

    /**
     * Register a real meter in the given child registry using the builder API.
     * May return {@code null} if the backing object has been garbage collected
     * (for WeakReference-based meters like Gauge, FunctionCounter, etc.).
     *
     * @param registry The child registry to register in.
     * @return The registered child meter, or null if registration failed.
     */
    protected abstract T registerNewMeter(MeterRegistry registry);

    @Override
    public synchronized void add(MeterRegistry registry) {
        T meter = registerNewMeter(registry);
        if (meter != null) {
            Map<MeterRegistry, T> newChildren = new IdentityHashMap<>(children);
            newChildren.put(registry, meter);
            children = Collections.unmodifiableMap(newChildren);
        }
    }

    @Override
    public synchronized void remove(MeterRegistry registry) {
        Map<MeterRegistry, T> newChildren = new IdentityHashMap<>(children);
        newChildren.remove(registry);
        children = newChildren.isEmpty()
                ? Collections.emptyMap()
                : Collections.unmodifiableMap(newChildren);
    }

}
```

#### File: `src/main/java/dev/linhvu/micrometer/composite/CompositeCounter.java` [NEW]

```java
package dev.linhvu.micrometer.composite;

import dev.linhvu.micrometer.Counter;
import dev.linhvu.micrometer.Meter;
import dev.linhvu.micrometer.MeterRegistry;
import dev.linhvu.micrometer.noop.NoopCounter;

/**
 * A composite counter that fans out {@link #increment(double)} to all child counters
 * and reads {@link #count()} from the first child.
 */
class CompositeCounter extends AbstractCompositeMeter<Counter> implements Counter {

    CompositeCounter(Meter.Id id) {
        super(id);
    }

    @Override
    public void increment(double amount) {
        for (Counter child : getChildren()) {
            child.increment(amount);
        }
    }

    @Override
    public double count() {
        return firstChild().count();
    }

    @Override
    protected Counter newNoopMeter() {
        return new NoopCounter(getId());
    }

    @Override
    protected Counter registerNewMeter(MeterRegistry registry) {
        return Counter.builder(getId().getName())
                .tags(getId().getTagsAsIterable())
                .description(getId().getDescription())
                .baseUnit(getId().getBaseUnit())
                .register(registry);
    }

}
```

#### File: `src/main/java/dev/linhvu/micrometer/composite/CompositeGauge.java` [NEW]

```java
package dev.linhvu.micrometer.composite;

import dev.linhvu.micrometer.Gauge;
import dev.linhvu.micrometer.Meter;
import dev.linhvu.micrometer.MeterRegistry;
import dev.linhvu.micrometer.noop.NoopGauge;

import java.lang.ref.WeakReference;
import java.util.function.ToDoubleFunction;

/**
 * A composite gauge that reads {@link #value()} from the first child.
 * <p>
 * Uses a {@link WeakReference} to the observed object, just like
 * {@link dev.linhvu.micrometer.internal.DefaultGauge}. If the object is
 * garbage collected, {@link #registerNewMeter(MeterRegistry)} returns null
 * and the child is not added.
 *
 * @param <S> The type of the state object being observed.
 */
class CompositeGauge<S> extends AbstractCompositeMeter<Gauge> implements Gauge {

    private final WeakReference<S> objRef;

    private final ToDoubleFunction<S> f;

    CompositeGauge(Meter.Id id, S obj, ToDoubleFunction<S> f) {
        super(id);
        this.objRef = new WeakReference<>(obj);
        this.f = f;
    }

    @Override
    public double value() {
        return firstChild().value();
    }

    @Override
    protected Gauge newNoopMeter() {
        return new NoopGauge(getId());
    }

    @Override
    protected Gauge registerNewMeter(MeterRegistry registry) {
        S obj = objRef.get();
        if (obj == null) {
            return null;
        }
        return Gauge.builder(getId().getName(), obj, f)
                .tags(getId().getTagsAsIterable())
                .description(getId().getDescription())
                .baseUnit(getId().getBaseUnit())
                .register(registry);
    }

}
```

#### File: `src/main/java/dev/linhvu/micrometer/composite/CompositeTimer.java` [NEW]

```java
package dev.linhvu.micrometer.composite;

import dev.linhvu.micrometer.Clock;
import dev.linhvu.micrometer.Meter;
import dev.linhvu.micrometer.MeterRegistry;
import dev.linhvu.micrometer.Timer;
import dev.linhvu.micrometer.distribution.DistributionStatisticConfig;
import dev.linhvu.micrometer.distribution.HistogramSnapshot;
import dev.linhvu.micrometer.noop.NoopTimer;

import java.time.Duration;
import java.util.Arrays;
import java.util.concurrent.Callable;
import java.util.concurrent.TimeUnit;
import java.util.function.Supplier;

/**
 * A composite timer that fans out {@link #record(long, TimeUnit)} to all child timers
 * and reads from the first child.
 * <p>
 * For convenience methods ({@code record(Supplier)}, {@code record(Runnable)}), the
 * timing is done once at the composite level using the composite's clock, then the
 * recorded duration is fanned out. This avoids each child independently timing the
 * same operation.
 */
class CompositeTimer extends AbstractCompositeMeter<Timer> implements Timer {

    private final Clock clock;

    private final DistributionStatisticConfig distributionStatisticConfig;

    CompositeTimer(Meter.Id id, Clock clock, DistributionStatisticConfig distributionStatisticConfig) {
        super(id);
        this.clock = clock;
        this.distributionStatisticConfig = distributionStatisticConfig;
    }

    @Override
    public void record(long amount, TimeUnit unit) {
        for (Timer child : getChildren()) {
            child.record(amount, unit);
        }
    }

    @Override
    public <T> T record(Supplier<T> f) {
        long start = clock.monotonicTime();
        try {
            return f.get();
        } finally {
            long durationNanos = clock.monotonicTime() - start;
            record(durationNanos, TimeUnit.NANOSECONDS);
        }
    }

    @Override
    public <T> T recordCallable(Callable<T> f) throws Exception {
        long start = clock.monotonicTime();
        try {
            return f.call();
        } finally {
            long durationNanos = clock.monotonicTime() - start;
            record(durationNanos, TimeUnit.NANOSECONDS);
        }
    }

    @Override
    public void record(Runnable f) {
        long start = clock.monotonicTime();
        try {
            f.run();
        } finally {
            long durationNanos = clock.monotonicTime() - start;
            record(durationNanos, TimeUnit.NANOSECONDS);
        }
    }

    @Override
    public long count() {
        return firstChild().count();
    }

    @Override
    public double totalTime(TimeUnit unit) {
        return firstChild().totalTime(unit);
    }

    @Override
    public double max(TimeUnit unit) {
        return firstChild().max(unit);
    }

    @Override
    public TimeUnit baseTimeUnit() {
        return firstChild().baseTimeUnit();
    }

    @Override
    public HistogramSnapshot takeSnapshot() {
        return firstChild().takeSnapshot();
    }

    @Override
    protected Timer newNoopMeter() {
        return new NoopTimer(getId());
    }

    @Override
    protected Timer registerNewMeter(MeterRegistry registry) {
        Timer.Builder builder = Timer.builder(getId().getName())
                .tags(getId().getTagsAsIterable())
                .description(getId().getDescription());

        // Pass through distribution config settings to the child
        if (distributionStatisticConfig.getServiceLevelObjectives() != null) {
            Duration[] slos = Arrays.stream(distributionStatisticConfig.getServiceLevelObjectives())
                    .mapToObj(nanos -> Duration.ofNanos((long) nanos))
                    .toArray(Duration[]::new);
            builder.serviceLevelObjectives(slos);
        }
        if (distributionStatisticConfig.getMinimumExpectedValue() != null) {
            builder.minimumExpectedValue(
                    Duration.ofNanos(distributionStatisticConfig.getMinimumExpectedValue().longValue()));
        }
        if (distributionStatisticConfig.getMaximumExpectedValue() != null
                && distributionStatisticConfig.getMaximumExpectedValue() != Double.POSITIVE_INFINITY) {
            builder.maximumExpectedValue(
                    Duration.ofNanos(distributionStatisticConfig.getMaximumExpectedValue().longValue()));
        }
        if (distributionStatisticConfig.getExpiry() != null) {
            builder.distributionStatisticExpiry(distributionStatisticConfig.getExpiry());
        }
        if (distributionStatisticConfig.getBufferLength() != null) {
            builder.distributionStatisticBufferLength(distributionStatisticConfig.getBufferLength());
        }

        return builder.register(registry);
    }

}
```

#### File: `src/main/java/dev/linhvu/micrometer/composite/CompositeDistributionSummary.java` [NEW]

```java
package dev.linhvu.micrometer.composite;

import dev.linhvu.micrometer.DistributionSummary;
import dev.linhvu.micrometer.Meter;
import dev.linhvu.micrometer.MeterRegistry;
import dev.linhvu.micrometer.distribution.DistributionStatisticConfig;
import dev.linhvu.micrometer.distribution.HistogramSnapshot;
import dev.linhvu.micrometer.noop.NoopDistributionSummary;

/**
 * A composite distribution summary that fans out {@link #record(double)} to all children
 * and reads from the first child.
 */
class CompositeDistributionSummary extends AbstractCompositeMeter<DistributionSummary>
        implements DistributionSummary {

    private final double scale;

    private final DistributionStatisticConfig distributionStatisticConfig;

    CompositeDistributionSummary(Meter.Id id, double scale,
            DistributionStatisticConfig distributionStatisticConfig) {
        super(id);
        this.scale = scale;
        this.distributionStatisticConfig = distributionStatisticConfig;
    }

    @Override
    public void record(double amount) {
        for (DistributionSummary child : getChildren()) {
            child.record(amount);
        }
    }

    @Override
    public long count() {
        return firstChild().count();
    }

    @Override
    public double totalAmount() {
        return firstChild().totalAmount();
    }

    @Override
    public double max() {
        return firstChild().max();
    }

    @Override
    public HistogramSnapshot takeSnapshot() {
        return firstChild().takeSnapshot();
    }

    @Override
    protected DistributionSummary newNoopMeter() {
        return new NoopDistributionSummary(getId());
    }

    @Override
    protected DistributionSummary registerNewMeter(MeterRegistry registry) {
        DistributionSummary.Builder builder = DistributionSummary.builder(getId().getName())
                .tags(getId().getTagsAsIterable())
                .description(getId().getDescription())
                .baseUnit(getId().getBaseUnit())
                .scale(scale);

        if (distributionStatisticConfig.getServiceLevelObjectives() != null) {
            builder.serviceLevelObjectives(distributionStatisticConfig.getServiceLevelObjectives());
        }
        if (distributionStatisticConfig.getMinimumExpectedValue() != null) {
            builder.minimumExpectedValue(distributionStatisticConfig.getMinimumExpectedValue());
        }
        if (distributionStatisticConfig.getMaximumExpectedValue() != null
                && distributionStatisticConfig.getMaximumExpectedValue() != Double.POSITIVE_INFINITY) {
            builder.maximumExpectedValue(distributionStatisticConfig.getMaximumExpectedValue());
        }
        if (distributionStatisticConfig.getExpiry() != null) {
            builder.distributionStatisticExpiry(distributionStatisticConfig.getExpiry());
        }
        if (distributionStatisticConfig.getBufferLength() != null) {
            builder.distributionStatisticBufferLength(distributionStatisticConfig.getBufferLength());
        }

        return builder.register(registry);
    }

}
```

#### File: `src/main/java/dev/linhvu/micrometer/composite/CompositeLongTaskTimer.java` [NEW]

```java
package dev.linhvu.micrometer.composite;

import dev.linhvu.micrometer.Clock;
import dev.linhvu.micrometer.LongTaskTimer;
import dev.linhvu.micrometer.Meter;
import dev.linhvu.micrometer.MeterRegistry;
import dev.linhvu.micrometer.distribution.DistributionStatisticConfig;
import dev.linhvu.micrometer.distribution.HistogramSnapshot;
import dev.linhvu.micrometer.noop.NoopLongTaskTimer;

import java.time.Duration;
import java.util.ArrayList;
import java.util.Arrays;
import java.util.Collection;
import java.util.List;
import java.util.concurrent.TimeUnit;

/**
 * A composite long task timer that starts samples in ALL child timers simultaneously.
 * <p>
 * Unlike other composite meters where write operations iterate children directly,
 * {@link #start()} must coordinate samples across all children. A {@link CompositeSample}
 * wraps multiple child samples and fans out {@code stop()} and {@code duration()}.
 */
class CompositeLongTaskTimer extends AbstractCompositeMeter<LongTaskTimer> implements LongTaskTimer {

    private final Clock clock;

    private final DistributionStatisticConfig distributionStatisticConfig;

    CompositeLongTaskTimer(Meter.Id id, Clock clock,
            DistributionStatisticConfig distributionStatisticConfig) {
        super(id);
        this.clock = clock;
        this.distributionStatisticConfig = distributionStatisticConfig;
    }

    @Override
    public Sample start() {
        Collection<LongTaskTimer> children = getChildren();
        List<Sample> childSamples = new ArrayList<>(children.size());
        for (LongTaskTimer child : children) {
            childSamples.add(child.start());
        }
        return new CompositeSample(childSamples);
    }

    @Override
    public double duration(TimeUnit unit) {
        return firstChild().duration(unit);
    }

    @Override
    public int activeTasks() {
        return firstChild().activeTasks();
    }

    @Override
    public double max(TimeUnit unit) {
        return firstChild().max(unit);
    }

    @Override
    public TimeUnit baseTimeUnit() {
        return TimeUnit.SECONDS;
    }

    @Override
    public HistogramSnapshot takeSnapshot() {
        return firstChild().takeSnapshot();
    }

    @Override
    protected LongTaskTimer newNoopMeter() {
        return new NoopLongTaskTimer(getId());
    }

    @Override
    protected LongTaskTimer registerNewMeter(MeterRegistry registry) {
        LongTaskTimer.Builder builder = LongTaskTimer.builder(getId().getName())
                .tags(getId().getTagsAsIterable())
                .description(getId().getDescription());

        if (distributionStatisticConfig.getServiceLevelObjectives() != null) {
            Duration[] slos = Arrays.stream(distributionStatisticConfig.getServiceLevelObjectives())
                    .mapToObj(nanos -> Duration.ofNanos((long) nanos))
                    .toArray(Duration[]::new);
            builder.serviceLevelObjectives(slos);
        }
        if (distributionStatisticConfig.getMinimumExpectedValue() != null) {
            builder.minimumExpectedValue(
                    Duration.ofNanos(distributionStatisticConfig.getMinimumExpectedValue().longValue()));
        }
        if (distributionStatisticConfig.getMaximumExpectedValue() != null
                && distributionStatisticConfig.getMaximumExpectedValue() != Double.POSITIVE_INFINITY) {
            builder.maximumExpectedValue(
                    Duration.ofNanos(distributionStatisticConfig.getMaximumExpectedValue().longValue()));
        }

        return builder.register(registry);
    }

    /**
     * A sample that wraps multiple child samples. Stopping this sample stops all
     * children; duration queries delegate to the first child.
     */
    private static class CompositeSample extends Sample {

        private final List<Sample> childSamples;

        CompositeSample(List<Sample> childSamples) {
            this.childSamples = childSamples;
        }

        @Override
        public long stop() {
            long lastDuration = -1;
            for (Sample sample : childSamples) {
                lastDuration = sample.stop();
            }
            return lastDuration;
        }

        @Override
        public double duration(TimeUnit unit) {
            if (childSamples.isEmpty()) {
                return -1;
            }
            return childSamples.get(0).duration(unit);
        }

    }

}
```

#### File: `src/main/java/dev/linhvu/micrometer/composite/CompositeFunctionCounter.java` [NEW]

```java
package dev.linhvu.micrometer.composite;

import dev.linhvu.micrometer.FunctionCounter;
import dev.linhvu.micrometer.Meter;
import dev.linhvu.micrometer.MeterRegistry;
import dev.linhvu.micrometer.noop.NoopFunctionCounter;

import java.lang.ref.WeakReference;
import java.util.function.ToDoubleFunction;

/**
 * A composite function counter. Unlike other composite meters, {@link #count()}
 * reads directly from the weak-referenced object rather than from the first child.
 * This is because function-based meters derive their value from an external object,
 * not from accumulated state.
 *
 * @param <T> The type of the state object being observed.
 */
class CompositeFunctionCounter<T> extends AbstractCompositeMeter<FunctionCounter> implements FunctionCounter {

    private final WeakReference<T> objRef;

    private final ToDoubleFunction<T> f;

    CompositeFunctionCounter(Meter.Id id, T obj, ToDoubleFunction<T> f) {
        super(id);
        this.objRef = new WeakReference<>(obj);
        this.f = f;
    }

    @Override
    public double count() {
        T obj = objRef.get();
        return obj != null ? f.applyAsDouble(obj) : 0;
    }

    @Override
    protected FunctionCounter newNoopMeter() {
        return new NoopFunctionCounter(getId());
    }

    @Override
    protected FunctionCounter registerNewMeter(MeterRegistry registry) {
        T obj = objRef.get();
        if (obj == null) {
            return null;
        }
        return FunctionCounter.builder(getId().getName(), obj, f)
                .tags(getId().getTagsAsIterable())
                .description(getId().getDescription())
                .baseUnit(getId().getBaseUnit())
                .register(registry);
    }

}
```

#### File: `src/main/java/dev/linhvu/micrometer/composite/CompositeFunctionTimer.java` [NEW]

```java
package dev.linhvu.micrometer.composite;

import dev.linhvu.micrometer.FunctionTimer;
import dev.linhvu.micrometer.Meter;
import dev.linhvu.micrometer.MeterRegistry;
import dev.linhvu.micrometer.noop.NoopFunctionTimer;

import java.lang.ref.WeakReference;
import java.util.concurrent.TimeUnit;
import java.util.function.ToDoubleFunction;
import java.util.function.ToLongFunction;

/**
 * A composite function timer. Like {@link CompositeFunctionCounter}, the read
 * methods delegate to the first child (which itself reads from the weak-referenced
 * object via the count and total-time functions).
 *
 * @param <T> The type of the state object being observed.
 */
class CompositeFunctionTimer<T> extends AbstractCompositeMeter<FunctionTimer> implements FunctionTimer {

    private final WeakReference<T> objRef;

    private final ToLongFunction<T> countFunction;

    private final ToDoubleFunction<T> totalTimeFunction;

    private final TimeUnit totalTimeFunctionUnit;

    CompositeFunctionTimer(Meter.Id id, T obj, ToLongFunction<T> countFunction,
            ToDoubleFunction<T> totalTimeFunction, TimeUnit totalTimeFunctionUnit) {
        super(id);
        this.objRef = new WeakReference<>(obj);
        this.countFunction = countFunction;
        this.totalTimeFunction = totalTimeFunction;
        this.totalTimeFunctionUnit = totalTimeFunctionUnit;
    }

    @Override
    public double count() {
        return firstChild().count();
    }

    @Override
    public double totalTime(TimeUnit unit) {
        return firstChild().totalTime(unit);
    }

    @Override
    public TimeUnit baseTimeUnit() {
        return firstChild().baseTimeUnit();
    }

    @Override
    protected FunctionTimer newNoopMeter() {
        return new NoopFunctionTimer(getId());
    }

    @Override
    protected FunctionTimer registerNewMeter(MeterRegistry registry) {
        T obj = objRef.get();
        if (obj == null) {
            return null;
        }
        return FunctionTimer.builder(getId().getName(), obj, countFunction,
                        totalTimeFunction, totalTimeFunctionUnit)
                .tags(getId().getTagsAsIterable())
                .description(getId().getDescription())
                .register(registry);
    }

}
```

#### File: `src/main/java/dev/linhvu/micrometer/composite/CompositeTimeGauge.java` [NEW]

```java
package dev.linhvu.micrometer.composite;

import dev.linhvu.micrometer.Meter;
import dev.linhvu.micrometer.MeterRegistry;
import dev.linhvu.micrometer.TimeGauge;
import dev.linhvu.micrometer.noop.NoopTimeGauge;

import java.lang.ref.WeakReference;
import java.util.concurrent.TimeUnit;
import java.util.function.ToDoubleFunction;

/**
 * A composite time gauge. Reads {@link #value()} and {@link #baseTimeUnit()}
 * from the first child.
 *
 * @param <T> The type of the state object being observed.
 */
class CompositeTimeGauge<T> extends AbstractCompositeMeter<TimeGauge> implements TimeGauge {

    private final WeakReference<T> objRef;

    private final ToDoubleFunction<T> f;

    private final TimeUnit fUnit;

    CompositeTimeGauge(Meter.Id id, T obj, TimeUnit fUnit, ToDoubleFunction<T> f) {
        super(id);
        this.objRef = new WeakReference<>(obj);
        this.f = f;
        this.fUnit = fUnit;
    }

    @Override
    public double value() {
        return firstChild().value();
    }

    @Override
    public TimeUnit baseTimeUnit() {
        return firstChild().baseTimeUnit();
    }

    @Override
    protected TimeGauge newNoopMeter() {
        return new NoopTimeGauge(getId());
    }

    @Override
    protected TimeGauge registerNewMeter(MeterRegistry registry) {
        T obj = objRef.get();
        if (obj == null) {
            return null;
        }
        return TimeGauge.builder(getId().getName(), obj, fUnit, f)
                .tags(getId().getTagsAsIterable())
                .description(getId().getDescription())
                .register(registry);
    }

}
```

#### File: `src/main/java/dev/linhvu/micrometer/composite/CompositeMeterRegistry.java` [NEW]

```java
package dev.linhvu.micrometer.composite;

import dev.linhvu.micrometer.Clock;
import dev.linhvu.micrometer.Counter;
import dev.linhvu.micrometer.DistributionSummary;
import dev.linhvu.micrometer.FunctionCounter;
import dev.linhvu.micrometer.FunctionTimer;
import dev.linhvu.micrometer.Gauge;
import dev.linhvu.micrometer.LongTaskTimer;
import dev.linhvu.micrometer.Meter;
import dev.linhvu.micrometer.MeterRegistry;
import dev.linhvu.micrometer.TimeGauge;
import dev.linhvu.micrometer.Timer;
import dev.linhvu.micrometer.config.NamingConvention;
import dev.linhvu.micrometer.distribution.DistributionStatisticConfig;

import java.util.LinkedHashSet;
import java.util.Set;
import java.util.concurrent.TimeUnit;
import java.util.function.ToDoubleFunction;
import java.util.function.ToLongFunction;

/**
 * A fan-out registry that broadcasts operations to multiple child registries.
 * <p>
 * When a meter is registered, it creates a composite meter (e.g., {@link CompositeCounter})
 * that maintains a child meter in each registered child registry. Writes (increment,
 * record) fan out to all children; reads (count, value) come from the first child.
 * <p>
 * Adding a new child registry retroactively registers all existing meters on it.
 * Removing a child registry cleans up the corresponding child meters.
 * <p>
 * Uses {@link NamingConvention#identity} — no name/tag transformation at the composite
 * level. Each child registry applies its own naming convention when it creates the
 * real meter.
 * <p>
 * <b>Simplification:</b> The real Micrometer supports nested composite registries
 * (a tree that flattens to non-composite leaf registries) with parent-child tracking
 * and recursive descendant propagation. We support a single level of fan-out — the
 * children must be non-composite registries.
 */
public class CompositeMeterRegistry extends MeterRegistry {

    private final Set<MeterRegistry> registries = new LinkedHashSet<>();

    /**
     * Creates a CompositeMeterRegistry with no child registries using the system clock.
     */
    public CompositeMeterRegistry() {
        this(Clock.SYSTEM);
    }

    /**
     * Creates a CompositeMeterRegistry with the given clock.
     */
    public CompositeMeterRegistry(Clock clock) {
        super(clock);
        // Identity naming: the composite doesn't transform names/tags.
        // Each child applies its own naming convention.
        namingConvention(NamingConvention.identity);
    }

    // -----------------------------------------------------------------------
    // Child registry management
    // -----------------------------------------------------------------------

    /**
     * Adds a child registry. All existing meters in this composite are retroactively
     * registered on the new child.
     *
     * @param registry The child registry to add.
     * @return This composite, for chaining.
     */
    public CompositeMeterRegistry add(MeterRegistry registry) {
        synchronized (registries) {
            registries.add(registry);
        }
        // Propagate all existing meters to the new registry
        for (Meter meter : getMeters()) {
            if (meter instanceof CompositeMeter composite) {
                composite.add(registry);
            }
        }
        return this;
    }

    /**
     * Removes a child registry. The composite stops fanning out writes to
     * this registry. Meters already created in the child registry remain —
     * they simply stop receiving new recordings.
     *
     * @param registry The child registry to remove.
     * @return This composite, for chaining.
     */
    public CompositeMeterRegistry remove(MeterRegistry registry) {
        synchronized (registries) {
            registries.remove(registry);
        }
        for (Meter meter : getMeters()) {
            if (meter instanceof CompositeMeter composite) {
                composite.remove(registry);
            }
        }
        return this;
    }

    /**
     * Returns an unmodifiable snapshot of the child registries.
     */
    public Set<MeterRegistry> getRegistries() {
        synchronized (registries) {
            return Set.copyOf(registries);
        }
    }

    // -----------------------------------------------------------------------
    // Meter lifecycle hook
    // -----------------------------------------------------------------------

    /**
     * Called when a new meter is created. Propagates the composite meter to all
     * existing child registries so that each child gets a real meter.
     */
    @Override
    protected void meterAdded(Meter meter) {
        if (meter instanceof CompositeMeter composite) {
            Set<MeterRegistry> snapshot;
            synchronized (registries) {
                snapshot = Set.copyOf(registries);
            }
            for (MeterRegistry registry : snapshot) {
                composite.add(registry);
            }
        }
    }

    // -----------------------------------------------------------------------
    // Template method factories — create composite meters
    // -----------------------------------------------------------------------

    @Override
    protected Counter newCounter(Meter.Id id) {
        return new CompositeCounter(id);
    }

    @Override
    protected <T> Gauge newGauge(Meter.Id id, T obj, ToDoubleFunction<T> valueFunction) {
        return new CompositeGauge<>(id, obj, valueFunction);
    }

    @Override
    protected Timer newTimer(Meter.Id id, DistributionStatisticConfig distributionStatisticConfig) {
        return new CompositeTimer(id, clock, distributionStatisticConfig);
    }

    @Override
    protected DistributionSummary newDistributionSummary(Meter.Id id, double scale,
            DistributionStatisticConfig distributionStatisticConfig) {
        return new CompositeDistributionSummary(id, scale, distributionStatisticConfig);
    }

    @Override
    protected LongTaskTimer newLongTaskTimer(Meter.Id id,
            DistributionStatisticConfig distributionStatisticConfig) {
        return new CompositeLongTaskTimer(id, clock, distributionStatisticConfig);
    }

    @Override
    protected <T> FunctionCounter newFunctionCounter(Meter.Id id, T obj,
            ToDoubleFunction<T> countFunction) {
        return new CompositeFunctionCounter<>(id, obj, countFunction);
    }

    @Override
    protected <T> FunctionTimer newFunctionTimer(Meter.Id id, T obj,
            ToLongFunction<T> countFunction, ToDoubleFunction<T> totalTimeFunction,
            TimeUnit totalTimeFunctionUnit) {
        return new CompositeFunctionTimer<>(id, obj, countFunction, totalTimeFunction,
                totalTimeFunctionUnit);
    }

    @Override
    protected <T> TimeGauge newTimeGauge(Meter.Id id, T obj, TimeUnit timeFunctionUnit,
            ToDoubleFunction<T> f) {
        return new CompositeTimeGauge<>(id, obj, timeFunctionUnit, f);
    }

    // -----------------------------------------------------------------------
    // Configuration
    // -----------------------------------------------------------------------

    /**
     * The composite's base time unit is seconds. Each child registry uses its own
     * base time unit — the composite converts to nanoseconds when fanning out.
     */
    @Override
    protected TimeUnit getBaseTimeUnit() {
        return TimeUnit.SECONDS;
    }

    /**
     * Returns {@link DistributionStatisticConfig#NONE} — the composite doesn't inject
     * histogram defaults. Each child registry applies its own defaults when the
     * composite meter's {@code registerNewMeter()} creates a real meter via the builder.
     */
    @Override
    protected DistributionStatisticConfig defaultHistogramConfig() {
        return DistributionStatisticConfig.NONE;
    }

}
```

#### File: `src/main/java/dev/linhvu/micrometer/Metrics.java` [NEW]

```java
package dev.linhvu.micrometer;

import dev.linhvu.micrometer.composite.CompositeMeterRegistry;

import java.util.Collections;
import java.util.concurrent.TimeUnit;
import java.util.function.ToDoubleFunction;
import java.util.function.ToLongFunction;

/**
 * Static global facade for Micrometer metrics, backed by a
 * {@link CompositeMeterRegistry}.
 * <p>
 * Use this when dependency injection is not available (e.g., static methods,
 * library code, or legacy systems). The global registry starts with no children —
 * calling {@link #addRegistry(MeterRegistry)} plugs in a real backend.
 * <p>
 * Example:
 * <pre>{@code
 * // At application startup, plug in a real registry
 * Metrics.addRegistry(new SimpleMeterRegistry());
 *
 * // Anywhere in the application
 * Metrics.counter("requests.total", "method", "GET").increment();
 * Metrics.timer("request.duration", "method", "GET").record(duration);
 * }</pre>
 * <p>
 * Meters created before any registry is added are backed by composite meters
 * that initially have no children (reads return zero/noop). As soon as a
 * registry is added, those meters are retroactively propagated to it.
 */
public final class Metrics {

    /**
     * The global composite registry. All static methods delegate to this.
     */
    public static final CompositeMeterRegistry globalRegistry = new CompositeMeterRegistry();

    private Metrics() {
        // static utility class
    }

    // -----------------------------------------------------------------------
    // Registry management
    // -----------------------------------------------------------------------

    /**
     * Add a registry to the global composite. All existing and future metrics
     * will be propagated to it.
     */
    public static void addRegistry(MeterRegistry registry) {
        globalRegistry.add(registry);
    }

    /**
     * Remove a registry from the global composite.
     */
    public static void removeRegistry(MeterRegistry registry) {
        globalRegistry.remove(registry);
    }

    // -----------------------------------------------------------------------
    // Counter
    // -----------------------------------------------------------------------

    /**
     * Creates or retrieves a counter with the given name and tags.
     */
    public static Counter counter(String name, String... tags) {
        return globalRegistry.counter(name, tags);
    }

    /**
     * Creates or retrieves a counter with the given name and tags.
     */
    public static Counter counter(String name, Iterable<Tag> tags) {
        return globalRegistry.counter(name, tags);
    }

    // -----------------------------------------------------------------------
    // Timer
    // -----------------------------------------------------------------------

    /**
     * Creates or retrieves a timer with the given name and tags.
     */
    public static Timer timer(String name, String... tags) {
        return globalRegistry.timer(name, tags);
    }

    /**
     * Creates or retrieves a timer with the given name and tags.
     */
    public static Timer timer(String name, Iterable<Tag> tags) {
        return globalRegistry.timer(name, tags);
    }

    // -----------------------------------------------------------------------
    // DistributionSummary
    // -----------------------------------------------------------------------

    /**
     * Creates or retrieves a distribution summary with the given name and tags.
     */
    public static DistributionSummary summary(String name, String... tags) {
        return globalRegistry.summary(name, tags);
    }

    /**
     * Creates or retrieves a distribution summary with the given name and tags.
     */
    public static DistributionSummary summary(String name, Iterable<Tag> tags) {
        return globalRegistry.summary(name, tags);
    }

    // -----------------------------------------------------------------------
    // Gauge
    // -----------------------------------------------------------------------

    /**
     * Register a gauge backed by a state object and value function.
     */
    public static <T> T gauge(String name, Iterable<Tag> tags, T stateObject,
            ToDoubleFunction<T> valueFunction) {
        return globalRegistry.gauge(name, tags, stateObject, valueFunction);
    }

    /**
     * Register a gauge backed by a {@link Number}.
     */
    public static <T extends Number> T gauge(String name, Iterable<Tag> tags, T number) {
        return globalRegistry.gauge(name, tags, number);
    }

    /**
     * Register a gauge backed by a {@link Number} with no tags.
     */
    public static <T extends Number> T gauge(String name, T number) {
        return globalRegistry.gauge(name, number);
    }

    /**
     * Register a gauge that reports the size of a collection.
     */
    public static <T extends java.util.Collection<?>> T gaugeCollectionSize(
            String name, Iterable<Tag> tags, T collection) {
        return globalRegistry.gaugeCollectionSize(name, tags, collection);
    }

    /**
     * Register a gauge that reports the size of a map.
     */
    public static <T extends java.util.Map<?, ?>> T gaugeMapSize(
            String name, Iterable<Tag> tags, T map) {
        return globalRegistry.gaugeMapSize(name, tags, map);
    }

    // -----------------------------------------------------------------------
    // More (less common meter types)
    // -----------------------------------------------------------------------

    /**
     * Access to less common meter types through the global registry.
     */
    public static class More {

        private More() {
        }

        /**
         * Register a long task timer.
         */
        public static LongTaskTimer longTaskTimer(String name, String... tags) {
            return LongTaskTimer.builder(name).tags(tags).register(globalRegistry);
        }

        /**
         * Register a function counter.
         */
        public static <T> FunctionCounter counter(String name, Iterable<Tag> tags,
                T obj, ToDoubleFunction<T> countFunction) {
            return globalRegistry.more().counter(name, tags, obj, countFunction);
        }

        /**
         * Register a time gauge.
         */
        public static <T> TimeGauge timeGauge(String name, Iterable<Tag> tags,
                T obj, TimeUnit fUnits, ToDoubleFunction<T> f) {
            return globalRegistry.more().timeGauge(name, tags, obj, fUnits, f);
        }

        /**
         * Register a function timer.
         */
        public static <T> FunctionTimer timer(String name, Iterable<Tag> tags,
                T obj, ToLongFunction<T> countFunction,
                ToDoubleFunction<T> totalTimeFunction,
                TimeUnit totalTimeFunctionUnit) {
            return globalRegistry.more().timer(name, tags, obj, countFunction,
                    totalTimeFunction, totalTimeFunctionUnit);
        }

    }

}
```

### Test Code

Test files are extensive — see the actual source files for the complete test code:

- `src/test/java/dev/linhvu/micrometer/composite/CompositeMeterRegistryTest.java` [NEW] — 24 tests covering fan-out, retroactive propagation, removal, all meter types
- `src/test/java/dev/linhvu/micrometer/composite/CompositeCounterTest.java` [NEW] — 7 tests for counter fan-out, tags, measurements
- `src/test/java/dev/linhvu/micrometer/composite/CompositeTimerTest.java` [NEW] — 9 tests for timer recording, Runnable/Supplier timing
- `src/test/java/dev/linhvu/micrometer/composite/CompositeGaugeTest.java` [NEW] — 3 tests for gauge value reading, live value observation
- `src/test/java/dev/linhvu/micrometer/MetricsTest.java` [NEW] — 10 tests for the static global facade
- `src/test/java/dev/linhvu/micrometer/integration/CompositeMeterRegistryIntegrationTest.java` [NEW] — 8 tests for filter/binder/dedup interaction

---

## Summary

| Concept | What It Means |
|---------|---------------|
| **CompositeMeterRegistry** | A fan-out registry that broadcasts writes to multiple child registries and reads from the first child |
| **CompositeMeter** | A meter implementation that maintains a child meter per registry, using copy-on-write for thread safety |
| **Write fan-out / Read from first** | The asymmetric design: writes (increment, record) go to ALL children; reads (count, value) come from ONE child |
| **Retroactive propagation** | Adding a child registry after meters exist causes all existing meters to be registered in the new child |
| **Metrics facade** | A static global `CompositeMeterRegistry` for use where dependency injection is unavailable |
| **`meterAdded()` hook** | Protected template method in MeterRegistry, called after meter creation, overridden by CompositeMeterRegistry to propagate |
| **`DistributionStatisticConfig.NONE`** | The composite's default — it doesn't inject histogram defaults, letting each child apply its own |

**Next: Chapter 15 — Push Registry** — An abstract base for registries that periodically push/export metrics to a remote backend on a schedule, introducing scheduled publishing with a semaphore guard and graceful shutdown.
