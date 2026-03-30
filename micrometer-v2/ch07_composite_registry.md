# Chapter 7: Composite Registry — Multi-Backend Publishing

> **Feature 7** · Tier 2 · Depends on: Features 1–4 (needs all meter types to wrap)

After this chapter you will be able to **publish metrics to multiple monitoring backends
simultaneously** — Prometheus AND Datadog, or SimpleMeterRegistry AND your custom exporter —
by registering meters on a composite registry that delegates to all child registries.

---

## 1. The API Contract

Before writing any implementation, let's define exactly what a client sees.

### What the client imports

```java
import simple.micrometer.api.CompositeMeterRegistry;
import simple.micrometer.api.Counter;
import simple.micrometer.api.MeterRegistry;
import simple.micrometer.internal.SimpleMeterRegistry;
```

### What the client writes

```java
SimpleMeterRegistry simpleRegistry = new SimpleMeterRegistry();
// Imagine: PrometheusRegistry prometheusRegistry = new PrometheusRegistry();

CompositeMeterRegistry composite = new CompositeMeterRegistry();
composite.add(simpleRegistry);
// composite.add(prometheusRegistry);

// Meters registered on composite are mirrored to all child registries
Counter counter = composite.counter("http.requests", "method", "GET");
counter.increment();

// Both child registries see the counter
double simpleCount = simpleRegistry.counter("http.requests", "method", "GET").count(); // 1.0
```

### Behavioral contract

| Promise | Detail |
|---------|--------|
| Write fan-out | `increment()`, `record()` are forwarded to ALL child registries |
| Read first-child | `count()`, `value()`, `totalTime()`, `max()` read from the FIRST child only |
| Dynamic children | Child registries can be added or removed at any time |
| Add wires existing | Adding a child wires ALL existing composite meters to the new child |
| Remove unwires | Removing a child stops delegation to it |
| No-child safety | With no children, operations are no-ops (no exceptions) |
| Self-reference prevention | `composite.add(composite)` throws `IllegalArgumentException` |
| Child deduplication | Adding the same child twice has no effect |

---

## 2. Client Usage & Tests

These tests define the API's behavior — they were written **before** any implementation.

### Counter increments are mirrored to all backends

```java
@Test
void shouldMirrorIncrementToAllChildren() {
    composite.add(backend1);
    composite.add(backend2);

    Counter counter = composite.counter("http.requests", "method", "GET");
    counter.increment();
    counter.increment(5.0);

    // Both backends see the full count
    assertThat(backend1.counter("http.requests", "method", "GET").count())
            .isEqualTo(6.0);
    assertThat(backend2.counter("http.requests", "method", "GET").count())
            .isEqualTo(6.0);
}
```

### Timer recordings are mirrored with consistent timing

```java
@Test
void shouldMirrorRecordToAllChildren() {
    composite.add(backend1);
    composite.add(backend2);

    Timer timer = composite.timer("http.latency", "method", "GET");
    timer.record(Duration.ofMillis(100));
    timer.record(Duration.ofMillis(200));

    Timer b1Timer = backend1.timer("http.latency", "method", "GET");
    Timer b2Timer = backend2.timer("http.latency", "method", "GET");

    assertThat(b1Timer.count()).isEqualTo(2);
    assertThat(b2Timer.count()).isEqualTo(2);
    assertThat(b1Timer.totalTime(TimeUnit.MILLISECONDS)).isEqualTo(300.0);
    assertThat(b2Timer.totalTime(TimeUnit.MILLISECONDS)).isEqualTo(300.0);
}
```

### Existing meters wire to newly-added children

```java
@Test
void shouldWireExistingMetersWhenChildAdded() {
    // Register meter BEFORE adding any children
    Counter counter = composite.counter("early.counter");
    counter.increment(10.0);

    // Now add a child — existing meter should be wired
    composite.add(backend1);
    counter.increment(5.0);

    // backend1 only sees increments after it was added
    assertThat(backend1.counter("early.counter").count()).isEqualTo(5.0);
}
```

### Operations are safe with no children

```java
@Test
void shouldNotThrowWhenIncrementingWithNoChildren() {
    Counter counter = composite.counter("orphan.counter");
    assertThatCode(() -> counter.increment(5.0)).doesNotThrowAnyException();
    assertThat(counter.count()).isEqualTo(0.0);
}
```

### Self-reference is prevented

```java
@Test
void shouldPreventSelfReference() {
    assertThatThrownBy(() -> composite.add(composite))
            .isInstanceOf(IllegalArgumentException.class)
            .hasMessageContaining("itself");
}
```

---

## 3. Implementing the Call Chain

The composite pattern introduces a delegation layer between the client's meter and the
concrete meters in each backend:

```
Client calls: counter.increment(5.0)
  │
  ├─► [Composite Layer] CompositeCounter.increment(5.0)
  │     Iterates all child counters via getChildren()
  │     Calls child.increment(5.0) on EACH one
  │
  ├─► [Child 1] CumulativeCounter.increment(5.0)
  │     Adds to DoubleAdder in SimpleMeterRegistry #1
  │
  └─► [Child 2] CumulativeCounter.increment(5.0)
        Adds to DoubleAdder in SimpleMeterRegistry #2
```

For reads, the chain is shorter:

```
Client calls: counter.count()
  │
  ├─► [Composite Layer] CompositeCounter.count()
  │     Calls firstChild().count()
  │
  └─► [Child 1] CumulativeCounter.count()
        Returns DoubleAdder value from the FIRST child only
```

### Layer 1: CompositeMeter — the internal contract

Every composite meter must support dynamic wiring:

```java
public interface CompositeMeter {
    void add(MeterRegistry registry);
    void remove(MeterRegistry registry);
}
```

When `CompositeMeterRegistry.add(child)` is called, it iterates all existing meters
and calls `CompositeMeter.add(child)` on each one. This is how meters registered
*before* the child was added still get wired to it.

### Layer 2: AbstractCompositeMeter — the shared plumbing

The base class manages a **copy-on-write map** from child registries to their concrete
meters:

```java
public abstract class AbstractCompositeMeter<T extends Meter> implements CompositeMeter {
    private volatile Map<MeterRegistry, T> children = Collections.emptyMap();
    private final Object lock = new Object();
    private volatile T noopMeter;

    abstract T newNoopMeter();
    abstract T registerNewMeter(MeterRegistry registry);

    @Override
    public void add(MeterRegistry registry) {
        T meter = registerNewMeter(registry);
        if (meter == null) return;
        synchronized (lock) {
            Map<MeterRegistry, T> newChildren = new LinkedHashMap<>(children);
            newChildren.put(registry, meter);
            children = Collections.unmodifiableMap(newChildren);
        }
    }

    protected T firstChild() {
        Map<MeterRegistry, T> snapshot = children;
        if (snapshot.isEmpty()) {
            T noop = noopMeter;
            if (noop == null) { noopMeter = noop = newNoopMeter(); }
            return noop;
        }
        return snapshot.values().iterator().next();
    }

    protected Collection<T> getChildren() {
        return children.values();
    }
}
```

**Why copy-on-write?** Write operations (`increment`, `record`) iterate `getChildren()`.
If the children map were modified during iteration, we'd get `ConcurrentModificationException`.
Copy-on-write ensures readers always see a consistent snapshot — reads are lock-free,
and only mutations (which are rare — adding/removing backends) take a lock.

**Why LinkedHashMap?** It preserves insertion order, making `firstChild()` deterministic.
The first child is always the first backend you added.

**Why volatile?** The `children` field must be volatile so that the new map published inside
`synchronized(lock)` is immediately visible to threads calling `getChildren()` or
`firstChild()` without holding the lock.

### Layer 3: CompositeCounter — the simplest composite

```java
public class CompositeCounter extends AbstractCompositeMeter<Counter> implements Counter {

    public CompositeCounter(Meter.Id id) { super(id); }

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
    Counter registerNewMeter(MeterRegistry registry) {
        return Counter.builder(getId().getName())
                .tags(getId().getTagsAsIterable())
                .description(getId().getDescription())
                .baseUnit(getId().getBaseUnit())
                .register(registry);
    }
}
```

The `registerNewMeter` method uses the **builder API** to create a concrete counter in the
child registry. This means each child applies its own filters — the composite's filters were
already applied when the composite meter's ID was created.

### Layer 3: CompositeTimer — measure-once semantics

The timer is the most interesting composite because of its lambda recording methods:

```java
@Override
public void record(Runnable f) {
    long start = clock.monotonicTime();
    try {
        f.run();
    } finally {
        record(clock.monotonicTime() - start, TimeUnit.NANOSECONDS);
    }
}
```

**Why measure once?** If each child timer independently timed the lambda, they'd measure
slightly different durations (due to iteration overhead). By measuring once at the composite
level, all backends get exactly the same duration — a subtle but important consistency
guarantee.

### Layer 3: CompositeGauge — WeakReference awareness

```java
@Override
Gauge registerNewMeter(MeterRegistry registry) {
    T obj = ref.get();
    if (obj == null) return null;  // object was GC'd — skip this child
    return Gauge.builder(getId().getName(), obj, valueFunction)
            .tags(getId().getTagsAsIterable())
            .register(registry);
}
```

CompositeGauge is the only composite meter where `registerNewMeter` can return `null`.
If the observed object has been garbage collected before a new child is added, there's
nothing to observe — so the child is silently skipped.

### Layer 4: CompositeMeterRegistry — the orchestrator

```java
public class CompositeMeterRegistry extends MeterRegistry {
    private final CopyOnWriteArrayList<MeterRegistry> childRegistries = new CopyOnWriteArrayList<>();

    public CompositeMeterRegistry add(MeterRegistry registry) {
        if (registry == this) {
            throw new IllegalArgumentException("Cannot add a CompositeMeterRegistry to itself");
        }
        if (childRegistries.addIfAbsent(registry)) {
            for (Meter meter : getMeters()) {
                if (meter instanceof CompositeMeter cm) {
                    cm.add(registry);
                }
            }
        }
        return this;
    }

    @Override
    protected Counter newCounter(Meter.Id id) {
        CompositeCounter counter = new CompositeCounter(id);
        for (MeterRegistry child : childRegistries) {
            counter.add(child);
        }
        return counter;
    }
    // ... same pattern for newGauge, newTimer, newDistributionSummary
}
```

There are two scenarios to handle:

1. **Meter created after child already exists**: `newCounter()` iterates `childRegistries`
   and wires the new composite meter to all existing children.

2. **Child added after meter already exists**: `add()` iterates `getMeters()` and wires
   all existing composite meters to the new child.

Together, these ensure every meter is wired to every child, regardless of ordering.

---

## 4. Try It Yourself

1. **Add a third backend**: Create three `SimpleMeterRegistry` instances, add all three
   to a composite, increment a counter, and verify all three see the value.

2. **Staggered adds**: Register a timer, add backend1, record 100ms, add backend2,
   record 200ms. Verify backend1 has count=2 (300ms total) and backend2 has count=1
   (200ms total).

3. **Remove and re-add**: Add a child, increment, remove it, increment more, re-add it.
   What count does the child have? (Hint: the child's counter persists — re-adding
   wires the composite to the *existing* counter in the child.)

4. **Composite with filters**: Add a `MeterFilter.denyNameStartsWith("internal.")`
   to the composite, and a different filter to a child. Verify that both filter
   chains apply independently.

---

## 5. Why This Works

### The Composite Pattern (GoF)
This is a textbook application of the Gang-of-Four Composite pattern. The client works
with a single `MeterRegistry` reference and doesn't know (or care) whether it's a
`SimpleMeterRegistry`, a `CompositeMeterRegistry`, or something else entirely. The
composite transparently delegates to multiple backends.

### Asymmetric delegation
Write operations fan out to ALL children. Read operations delegate to the FIRST child
only. This asymmetry is intentional:
- **Writes must fan out** because every backend needs to receive all data points.
- **Reads should not aggregate** because summing counts across backends would
  double-count (each backend has an independent copy of the same data).

### Copy-on-write for safe iteration
The `children` map uses copy-on-write semantics. This means `getChildren()` during
`increment()` is lock-free and never throws `ConcurrentModificationException`, even if
a child is being added or removed concurrently. The tradeoff: adding/removing a child
allocates a new map. Since children are modified rarely (startup/shutdown), this is
an excellent tradeoff.

### Builder API for child registration
Composite meters use the builder API (`Counter.builder(...).register(childRegistry)`)
rather than calling child internals directly. This means each child registry applies its
own filter chain independently — the composite doesn't need to know about the child's
configuration.

---

## 6. What We Enhanced

| Enhancement | Description | Enabled by |
|-------------|-------------|------------|
| Multi-backend publishing | A single counter/timer/gauge/summary publishes to multiple monitoring systems | CompositeMeterRegistry + 4 composite meters |
| Dynamic backend management | Child registries can be added/removed at runtime | CompositeMeter.add/remove + wiring in add() |
| Noop fallback | Composite meters work safely with zero children | AbstractCompositeMeter.firstChild() lazy noop |
| Copy-on-write thread safety | Lock-free reads for write operations, synchronized writes for mutations | AbstractCompositeMeter children map |

---

## 7. Connection to Real Framework

| Simplified Class | Real Framework Class | Key Simplifications |
|------------------|---------------------|---------------------|
| `CompositeMeterRegistry` | `io.micrometer.core.instrument.composite.CompositeMeterRegistry` | No nested composite flattening, no parent tracking, no CAS spinlock |
| `CompositeMeter` | `io.micrometer.core.instrument.composite.CompositeMeter` | Same interface — add/remove contract |
| `AbstractCompositeMeter` | `io.micrometer.core.instrument.composite.AbstractCompositeMeter` | `synchronized` instead of CAS spinlock; `LinkedHashMap` instead of `IdentityHashMap` |
| `CompositeCounter` | `io.micrometer.core.instrument.composite.CompositeCounter` | Same delegation pattern |
| `CompositeGauge` | `io.micrometer.core.instrument.composite.CompositeGauge` | Same WeakReference handling |
| `CompositeTimer` | `io.micrometer.core.instrument.composite.CompositeTimer` | No DistributionStatisticConfig forwarding, no PauseDetector |
| `CompositeDistributionSummary` | `io.micrometer.core.instrument.composite.CompositeDistributionSummary` | No DistributionStatisticConfig forwarding, no scale |

**What the real framework adds:**
- **Nested composite flattening**: Adding a `CompositeMeterRegistry` as a child flattens
  its descendants into `nonCompositeDescendants`, so composite meters only wrap leaf registries.
- **Parent tracking**: Each composite tracks its parents so that `updateDescendants()` can
  propagate changes upward through the tree.
- **CAS spinlock**: Uses `AtomicBoolean` compare-and-set in a busy loop instead of
  `synchronized`, avoiding thread parking under extreme contention.
- **IdentityHashMap**: Uses reference equality (`==`) for registry comparison instead of
  `equals()`, which is the correct semantic for registry identity.
- **Cycle detection**: `forbidSelfContainingComposite()` recursively walks the composite
  tree to prevent transitive self-reference.

---

## 8. Complete Code

All files created or modified in this feature. Copy all `[NEW]` files and replace all
`[MODIFIED]` files to get a compiling project with all tests passing.

### `src/main/java/simple/micrometer/internal/CompositeMeter.java` [NEW]

```java
package simple.micrometer.internal;

import simple.micrometer.api.MeterRegistry;

/**
 * Internal marker interface for meters created by {@code CompositeMeterRegistry}.
 * Each composite meter delegates to concrete meters in child registries.
 *
 * <p>When a child registry is added to or removed from the composite, every
 * existing composite meter is notified so it can create or discard the
 * corresponding concrete meter in that child.
 *
 * <p>Simplified from {@code io.micrometer.core.instrument.composite.CompositeMeter}.
 */
public interface CompositeMeter {

    /**
     * Creates a concrete meter in the given child registry and starts
     * delegating to it.
     */
    void add(MeterRegistry registry);

    /**
     * Removes the concrete meter associated with the given child registry
     * and stops delegating to it.
     */
    void remove(MeterRegistry registry);

}
```

### `src/main/java/simple/micrometer/internal/AbstractCompositeMeter.java` [NEW]

```java
package simple.micrometer.internal;

import simple.micrometer.api.Meter;
import simple.micrometer.api.MeterRegistry;

import java.util.Collection;
import java.util.Collections;
import java.util.LinkedHashMap;
import java.util.Map;

/**
 * Base class for composite meters that delegate to concrete meters in child
 * registries. Manages a copy-on-write map from child registries to their
 * concrete meter instances.
 *
 * <p><b>Thread safety:</b> The {@code children} map uses copy-on-write semantics.
 * Mutations ({@link #add}/{@link #remove}) create a new map under a synchronized
 * block, then publish it via a volatile write. Reads ({@link #firstChild()},
 * {@link #getChildren()}) are lock-free and see a consistent snapshot.
 *
 * <p><b>Write fan-out vs. read first-child:</b> Subclasses use {@link #getChildren()}
 * for write operations (increment, record) to fan out to ALL children, and
 * {@link #firstChild()} for read operations (count, value) to delegate to a
 * single child. This avoids double-counting while ensuring every backend
 * receives all data.
 *
 * <p>Simplified from {@code io.micrometer.core.instrument.composite.AbstractCompositeMeter}.
 * The real version uses a CAS spinlock instead of {@code synchronized} for
 * higher throughput under extreme contention.
 *
 * @param <T> the meter type this composite wraps (Counter, Gauge, Timer, etc.)
 */
public abstract class AbstractCompositeMeter<T extends Meter> implements CompositeMeter {

    private final Meter.Id id;

    /**
     * Copy-on-write map from child registries to their concrete meters.
     * Published via volatile write; read lock-free.
     * Uses {@link LinkedHashMap} to preserve insertion order so that
     * {@link #firstChild()} is deterministic.
     */
    private volatile Map<MeterRegistry, T> children = Collections.emptyMap();

    private final Object lock = new Object();

    /** Lazily created noop meter for when there are no children. */
    private volatile T noopMeter;

    protected AbstractCompositeMeter(Meter.Id id) {
        this.id = id;
    }

    // ── Abstract template methods ──────────────────────────────────────

    /**
     * Creates a noop meter to use as a fallback when no children exist.
     * Called lazily, at most once.
     */
    abstract T newNoopMeter();

    /**
     * Registers a concrete meter in the given child registry and returns it.
     * Returns {@code null} if the meter cannot be registered (e.g., a gauge
     * whose observed object has been garbage collected).
     */
    abstract T registerNewMeter(MeterRegistry registry);

    // ── CompositeMeter ─────────────────────────────────────────────────

    @Override
    public void add(MeterRegistry registry) {
        T meter = registerNewMeter(registry);
        if (meter == null) {
            return;
        }
        synchronized (lock) {
            Map<MeterRegistry, T> newChildren = new LinkedHashMap<>(children);
            newChildren.put(registry, meter);
            children = Collections.unmodifiableMap(newChildren);
        }
    }

    @Override
    public void remove(MeterRegistry registry) {
        synchronized (lock) {
            Map<MeterRegistry, T> newChildren = new LinkedHashMap<>(children);
            if (newChildren.remove(registry) != null) {
                children = newChildren.isEmpty()
                        ? Collections.emptyMap()
                        : Collections.unmodifiableMap(newChildren);
            }
        }
    }

    // ── Protected helpers for subclasses ────────────────────────────────

    /**
     * Returns the first child meter, or a lazily-created noop meter if no
     * children exist. Used by read operations (count, value, max, etc.).
     */
    protected T firstChild() {
        Map<MeterRegistry, T> snapshot = children;
        if (snapshot.isEmpty()) {
            T noop = noopMeter;
            if (noop == null) {
                noopMeter = noop = newNoopMeter();
            }
            return noop;
        }
        return snapshot.values().iterator().next();
    }

    /**
     * Returns all child meters. Used by write operations (increment, record)
     * to fan out to every child. The returned collection is a snapshot —
     * safe to iterate without synchronization.
     */
    protected Collection<T> getChildren() {
        return children.values();
    }

    public Meter.Id getId() {
        return id;
    }

}
```

### `src/main/java/simple/micrometer/internal/CompositeCounter.java` [NEW]

```java
package simple.micrometer.internal;

import simple.micrometer.api.Counter;
import simple.micrometer.api.Meter;
import simple.micrometer.api.MeterRegistry;

/**
 * A composite counter that delegates to concrete counters in child registries.
 *
 * <p><b>Write fan-out:</b> {@link #increment(double)} forwards the amount to
 * every child counter, ensuring all monitoring backends see the increment.
 *
 * <p><b>Read delegation:</b> {@link #count()} reads from the first child only,
 * avoiding double-counting across backends.
 *
 * <p>Simplified from {@code io.micrometer.core.instrument.composite.CompositeCounter}.
 */
public class CompositeCounter extends AbstractCompositeMeter<Counter> implements Counter {

    public CompositeCounter(Meter.Id id) {
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
    Counter newNoopMeter() {
        return new NoopCounter(getId());
    }

    /**
     * Registers a new counter in the child registry using the builder API.
     * The child registry applies its own filters and creates its own concrete
     * counter implementation (e.g., CumulativeCounter).
     */
    @Override
    Counter registerNewMeter(MeterRegistry registry) {
        return Counter.builder(getId().getName())
                .tags(getId().getTagsAsIterable())
                .description(getId().getDescription())
                .baseUnit(getId().getBaseUnit())
                .register(registry);
    }

}
```

### `src/main/java/simple/micrometer/internal/CompositeGauge.java` [NEW]

```java
package simple.micrometer.internal;

import simple.micrometer.api.Gauge;
import simple.micrometer.api.Meter;
import simple.micrometer.api.MeterRegistry;

import java.lang.ref.WeakReference;
import java.util.function.ToDoubleFunction;

/**
 * A composite gauge that delegates to concrete gauges in child registries.
 *
 * <p><b>WeakReference handling:</b> Like {@link DefaultGauge}, this composite
 * holds the observed object via a {@link WeakReference}. If the object is
 * garbage collected before a new child registry is added, {@link #registerNewMeter}
 * returns {@code null} and that child is skipped — the only composite meter
 * type that can return null from registration.
 *
 * <p><b>Read-only delegation:</b> Since gauges have no write operations (they
 * passively observe a value), only {@link #value()} exists, delegating to the
 * first child.
 *
 * <p>Simplified from {@code io.micrometer.core.instrument.composite.CompositeGauge}.
 *
 * @param <T> the type of the state object being observed
 */
public class CompositeGauge<T> extends AbstractCompositeMeter<Gauge> implements Gauge {

    private final WeakReference<T> ref;
    private final ToDoubleFunction<T> valueFunction;

    public CompositeGauge(Meter.Id id, T obj, ToDoubleFunction<T> valueFunction) {
        super(id);
        this.ref = new WeakReference<>(obj);
        this.valueFunction = valueFunction;
    }

    @Override
    public double value() {
        return firstChild().value();
    }

    @Override
    Gauge newNoopMeter() {
        return new NoopGauge(getId());
    }

    /**
     * Registers a new gauge in the child registry. Returns {@code null} if the
     * observed object has been garbage collected — the child is then skipped.
     */
    @Override
    Gauge registerNewMeter(MeterRegistry registry) {
        T obj = ref.get();
        if (obj == null) {
            return null;
        }
        return Gauge.builder(getId().getName(), obj, valueFunction)
                .tags(getId().getTagsAsIterable())
                .description(getId().getDescription())
                .baseUnit(getId().getBaseUnit())
                .register(registry);
    }

}
```

### `src/main/java/simple/micrometer/internal/CompositeTimer.java` [NEW]

```java
package simple.micrometer.internal;

import simple.micrometer.api.Clock;
import simple.micrometer.api.Meter;
import simple.micrometer.api.MeterRegistry;
import simple.micrometer.api.Timer;

import java.util.concurrent.Callable;
import java.util.concurrent.TimeUnit;
import java.util.function.Supplier;

/**
 * A composite timer that delegates to concrete timers in child registries.
 *
 * <p><b>Write fan-out:</b> {@link #record(long, TimeUnit)} forwards the
 * duration to every child timer.
 *
 * <p><b>Measure-once semantics:</b> For lambda/runnable recording methods
 * ({@link #record(Runnable)}, {@link #record(Supplier)}, etc.), the duration
 * is measured <em>once</em> at the composite level using the composite's clock,
 * then the raw duration is fanned out to all children. This ensures consistent
 * timing across all backends, regardless of each child's internal clock.
 *
 * <p><b>Read delegation:</b> {@link #count()}, {@link #totalTime(TimeUnit)},
 * and {@link #max(TimeUnit)} read from the first child only.
 *
 * <p>Simplified from {@code io.micrometer.core.instrument.composite.CompositeTimer}.
 */
public class CompositeTimer extends AbstractCompositeMeter<Timer> implements Timer {

    private final Clock clock;

    public CompositeTimer(Meter.Id id, Clock clock) {
        super(id);
        this.clock = clock;
    }

    // ── Write: fan out to all children ─────────────────────────────────

    @Override
    public void record(long amount, TimeUnit unit) {
        for (Timer child : getChildren()) {
            child.record(amount, unit);
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

    // ── Read: delegate to first child ──────────────────────────────────

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

    // ── Composite wiring ───────────────────────────────────────────────

    @Override
    Timer newNoopMeter() {
        return new NoopTimer(getId());
    }

    @Override
    Timer registerNewMeter(MeterRegistry registry) {
        return Timer.builder(getId().getName())
                .tags(getId().getTagsAsIterable())
                .description(getId().getDescription())
                .register(registry);
    }

}
```

### `src/main/java/simple/micrometer/internal/CompositeDistributionSummary.java` [NEW]

```java
package simple.micrometer.internal;

import simple.micrometer.api.DistributionSummary;
import simple.micrometer.api.Meter;
import simple.micrometer.api.MeterRegistry;

/**
 * A composite distribution summary that delegates to concrete summaries in
 * child registries.
 *
 * <p><b>Write fan-out:</b> {@link #record(double)} forwards the value to
 * every child summary, ensuring all monitoring backends receive the data.
 *
 * <p><b>Read delegation:</b> {@link #count()}, {@link #totalAmount()}, and
 * {@link #max()} read from the first child only.
 *
 * <p>Simplified from {@code io.micrometer.core.instrument.composite.CompositeDistributionSummary}.
 */
public class CompositeDistributionSummary extends AbstractCompositeMeter<DistributionSummary>
        implements DistributionSummary {

    public CompositeDistributionSummary(Meter.Id id) {
        super(id);
    }

    // ── Write: fan out to all children ─────────────────────────────────

    @Override
    public void record(double amount) {
        for (DistributionSummary child : getChildren()) {
            child.record(amount);
        }
    }

    // ── Read: delegate to first child ──────────────────────────────────

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

    // ── Composite wiring ───────────────────────────────────────────────

    @Override
    DistributionSummary newNoopMeter() {
        return new NoopDistributionSummary(getId());
    }

    @Override
    DistributionSummary registerNewMeter(MeterRegistry registry) {
        return DistributionSummary.builder(getId().getName())
                .tags(getId().getTagsAsIterable())
                .description(getId().getDescription())
                .baseUnit(getId().getBaseUnit())
                .register(registry);
    }

}
```

### `src/main/java/simple/micrometer/api/CompositeMeterRegistry.java` [NEW]

```java
package simple.micrometer.api;

import simple.micrometer.internal.CompositeMeter;
import simple.micrometer.internal.CompositeCounter;
import simple.micrometer.internal.CompositeDistributionSummary;
import simple.micrometer.internal.CompositeGauge;
import simple.micrometer.internal.CompositeTimer;

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
    protected Timer newTimer(Meter.Id id) {
        CompositeTimer timer = new CompositeTimer(id, clock);
        for (MeterRegistry child : childRegistries) {
            timer.add(child);
        }
        return timer;
    }

    @Override
    protected DistributionSummary newDistributionSummary(Meter.Id id) {
        CompositeDistributionSummary summary = new CompositeDistributionSummary(id);
        for (MeterRegistry child : childRegistries) {
            summary.add(child);
        }
        return summary;
    }

}
```

### `src/test/java/simple/micrometer/api/CompositeMeterRegistryTest.java` [NEW]

```java
package simple.micrometer.api;

import simple.micrometer.internal.SimpleMeterRegistry;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Nested;
import org.junit.jupiter.api.Test;

import java.time.Duration;
import java.util.ArrayList;
import java.util.List;
import java.util.concurrent.TimeUnit;
import java.util.concurrent.atomic.AtomicInteger;

import static org.assertj.core.api.Assertions.*;

/**
 * Client-perspective tests for {@link CompositeMeterRegistry}.
 * These tests demonstrate how a client uses the composite registry to
 * publish metrics to multiple backends simultaneously.
 */
class CompositeMeterRegistryTest {

    CompositeMeterRegistry composite;
    SimpleMeterRegistry backend1;
    SimpleMeterRegistry backend2;

    @BeforeEach
    void setUp() {
        composite = new CompositeMeterRegistry();
        backend1 = new SimpleMeterRegistry();
        backend2 = new SimpleMeterRegistry();
    }

    // ── Counter ────────────────────────────────────────────────────────

    @Nested
    class CounterTests {

        @Test
        void shouldMirrorIncrementToAllChildren() {
            composite.add(backend1);
            composite.add(backend2);

            Counter counter = composite.counter("http.requests", "method", "GET");
            counter.increment();
            counter.increment(5.0);

            // Both backends see the full count
            assertThat(backend1.counter("http.requests", "method", "GET").count())
                    .isEqualTo(6.0);
            assertThat(backend2.counter("http.requests", "method", "GET").count())
                    .isEqualTo(6.0);
        }

        @Test
        void shouldReturnCountFromFirstChild() {
            composite.add(backend1);
            composite.add(backend2);

            Counter counter = composite.counter("ops.count");
            counter.increment(3.0);

            // Composite reads from the first child
            assertThat(counter.count()).isEqualTo(3.0);
        }

        @Test
        void shouldUseBuilderApi() {
            composite.add(backend1);

            Counter counter = Counter.builder("cache.misses")
                    .tag("cache", "users")
                    .description("Cache miss count")
                    .baseUnit("misses")
                    .register(composite);

            counter.increment(10.0);

            Counter fromBackend = backend1.counter("cache.misses", "cache", "users");
            assertThat(fromBackend.count()).isEqualTo(10.0);
        }
    }

    // ── Gauge ──────────────────────────────────────────────────────────

    @Nested
    class GaugeTests {

        @Test
        void shouldMirrorGaugeValueToChildren() {
            composite.add(backend1);

            List<String> queue = new ArrayList<>();
            Gauge gauge = Gauge.builder("queue.size", queue, List::size)
                    .register(composite);

            queue.add("item1");
            queue.add("item2");

            // Composite gauge reads the current value
            assertThat(gauge.value()).isEqualTo(2.0);
        }

        @Test
        void shouldObserveSameObjectFromChild() {
            composite.add(backend1);

            AtomicInteger connections = new AtomicInteger(5);
            Gauge.builder("active.connections", () -> connections)
                    .register(composite);

            connections.set(42);

            // Read from composite — delegates to backend1's gauge
            Gauge compositeGauge = composite.getMeters().stream()
                    .filter(m -> m.getId().getName().equals("active.connections"))
                    .map(m -> (Gauge) m)
                    .findFirst()
                    .orElseThrow();
            assertThat(compositeGauge.value()).isEqualTo(42.0);
        }
    }

    // ── Timer ──────────────────────────────────────────────────────────

    @Nested
    class TimerTests {

        @Test
        void shouldMirrorRecordToAllChildren() {
            composite.add(backend1);
            composite.add(backend2);

            Timer timer = composite.timer("http.latency", "method", "GET");
            timer.record(Duration.ofMillis(100));
            timer.record(Duration.ofMillis(200));

            Timer b1Timer = backend1.timer("http.latency", "method", "GET");
            Timer b2Timer = backend2.timer("http.latency", "method", "GET");

            assertThat(b1Timer.count()).isEqualTo(2);
            assertThat(b2Timer.count()).isEqualTo(2);
            assertThat(b1Timer.totalTime(TimeUnit.MILLISECONDS)).isEqualTo(300.0);
            assertThat(b2Timer.totalTime(TimeUnit.MILLISECONDS)).isEqualTo(300.0);
        }

        @Test
        void shouldRecordRunnableAndFanOutDuration() {
            composite.add(backend1);

            Timer timer = composite.timer("task.duration");
            timer.record(() -> {
                // simulate work
            });

            assertThat(timer.count()).isEqualTo(1);
            assertThat(backend1.timer("task.duration").count()).isEqualTo(1);
        }

        @Test
        void shouldRecordSupplierAndReturnResult() {
            composite.add(backend1);

            Timer timer = composite.timer("compute.duration");
            String result = timer.record(() -> "hello");

            assertThat(result).isEqualTo("hello");
            assertThat(timer.count()).isEqualTo(1);
        }

        @Test
        void shouldRecordCallableAndReturnResult() throws Exception {
            composite.add(backend1);

            Timer timer = composite.timer("query.duration");
            String result = timer.recordCallable(() -> "world");

            assertThat(result).isEqualTo("world");
            assertThat(timer.count()).isEqualTo(1);
        }
    }

    // ── DistributionSummary ────────────────────────────────────────────

    @Nested
    class DistributionSummaryTests {

        @Test
        void shouldMirrorRecordToAllChildren() {
            composite.add(backend1);
            composite.add(backend2);

            DistributionSummary summary = composite.summary("payload.size", "uri", "/api");
            summary.record(1024);
            summary.record(2048);

            DistributionSummary b1 = backend1.summary("payload.size", "uri", "/api");
            DistributionSummary b2 = backend2.summary("payload.size", "uri", "/api");

            assertThat(b1.count()).isEqualTo(2);
            assertThat(b2.count()).isEqualTo(2);
            assertThat(b1.totalAmount()).isEqualTo(3072.0);
            assertThat(b2.totalAmount()).isEqualTo(3072.0);
        }

        @Test
        void shouldReturnStatsFromFirstChild() {
            composite.add(backend1);

            DistributionSummary summary = DistributionSummary.builder("response.size")
                    .tag("uri", "/users")
                    .baseUnit("bytes")
                    .register(composite);

            summary.record(100);
            summary.record(200);
            summary.record(300);

            assertThat(summary.count()).isEqualTo(3);
            assertThat(summary.totalAmount()).isEqualTo(600.0);
            assertThat(summary.mean()).isEqualTo(200.0);
        }
    }

    // ── Dynamic child management ───────────────────────────────────────

    @Nested
    class DynamicChildTests {

        @Test
        void shouldWireExistingMetersWhenChildAdded() {
            // Register meter BEFORE adding any children
            Counter counter = composite.counter("early.counter");
            counter.increment(10.0);

            // Now add a child — existing meter should be wired
            composite.add(backend1);
            counter.increment(5.0);

            // backend1 only sees increments after it was added
            assertThat(backend1.counter("early.counter").count()).isEqualTo(5.0);
        }

        @Test
        void shouldStopDelegatingAfterChildRemoved() {
            composite.add(backend1);

            Counter counter = composite.counter("tracked.count");
            counter.increment(3.0);
            assertThat(backend1.counter("tracked.count").count()).isEqualTo(3.0);

            // Remove the child
            composite.remove(backend1);
            counter.increment(7.0);

            // backend1 still has 3.0 — the 7.0 was NOT forwarded
            assertThat(backend1.counter("tracked.count").count()).isEqualTo(3.0);
        }

        @Test
        void shouldWorkWithMultipleChildrenAddedAtDifferentTimes() {
            composite.add(backend1);
            Counter counter = composite.counter("phased.count");
            counter.increment(1.0);

            // backend1 sees 1.0, backend2 not yet added
            assertThat(backend1.counter("phased.count").count()).isEqualTo(1.0);

            composite.add(backend2);
            counter.increment(2.0);

            // backend1 sees 1+2=3, backend2 sees only 2
            assertThat(backend1.counter("phased.count").count()).isEqualTo(3.0);
            assertThat(backend2.counter("phased.count").count()).isEqualTo(2.0);
        }
    }

    // ── No children (noop fallback) ────────────────────────────────────

    @Nested
    class NoChildrenTests {

        @Test
        void shouldNotThrowWhenIncrementingWithNoChildren() {
            Counter counter = composite.counter("orphan.counter");
            assertThatCode(() -> counter.increment(5.0)).doesNotThrowAnyException();
            assertThat(counter.count()).isEqualTo(0.0);
        }

        @Test
        void shouldNotThrowWhenRecordingTimerWithNoChildren() {
            Timer timer = composite.timer("orphan.timer");
            assertThatCode(() -> timer.record(Duration.ofMillis(100)))
                    .doesNotThrowAnyException();
            assertThat(timer.count()).isEqualTo(0);
        }

        @Test
        void shouldNotThrowWhenRecordingSummaryWithNoChildren() {
            DistributionSummary summary = composite.summary("orphan.summary");
            assertThatCode(() -> summary.record(42.0)).doesNotThrowAnyException();
            assertThat(summary.count()).isEqualTo(0);
        }

        @Test
        void shouldReturnZeroForGaugeWithNoChildren() {
            List<String> items = new ArrayList<>();
            items.add("test");
            Gauge gauge = Gauge.builder("orphan.gauge", items, List::size)
                    .register(composite);
            assertThat(gauge.value()).isEqualTo(0.0);
        }
    }

    // ── Edge cases ─────────────────────────────────────────────────────

    @Nested
    class EdgeCaseTests {

        @Test
        void shouldPreventSelfReference() {
            assertThatThrownBy(() -> composite.add(composite))
                    .isInstanceOf(IllegalArgumentException.class)
                    .hasMessageContaining("itself");
        }

        @Test
        void shouldDeduplicateChildRegistries() {
            composite.add(backend1);
            composite.add(backend1); // add again

            Counter counter = composite.counter("dedup.test");
            counter.increment();

            // Should only be counted once (not doubled)
            assertThat(counter.count()).isEqualTo(1.0);
            assertThat(composite.getRegistries()).hasSize(1);
        }

        @Test
        void shouldReturnRegistriesAsUnmodifiableSet() {
            composite.add(backend1);
            composite.add(backend2);

            var registries = composite.getRegistries();
            assertThat(registries).hasSize(2);
            assertThatThrownBy(() -> registries.add(new SimpleMeterRegistry()))
                    .isInstanceOf(UnsupportedOperationException.class);
        }

        @Test
        void shouldShareMeterIdentityAcrossCompositeAndChild() {
            composite.add(backend1);

            // Register via composite and then query via child
            Counter compositeCounter = composite.counter("shared.id", "env", "test");
            compositeCounter.increment(42.0);

            Counter childCounter = backend1.counter("shared.id", "env", "test");
            assertThat(childCounter.count()).isEqualTo(42.0);
        }
    }

}
```
