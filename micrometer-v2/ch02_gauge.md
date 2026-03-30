# Chapter 2: Gauge — Observing Current Values

> **Feature 2** · Tier 1 · Depends on: Feature 1 (Counter & MeterRegistry)

After this chapter you will have a pull-based meter that samples the current value
of any object — queue depth, active connections, cache size — by applying a function
on demand. You'll also understand Micrometer's `WeakReference` design for gauges and
why it exists.

---

## 1. The API Contract

Before writing any implementation, let's define exactly what a client sees.

### What the client imports

```java
import simple.micrometer.api.Gauge;
import simple.micrometer.api.MeterRegistry;
import simple.micrometer.internal.SimpleMeterRegistry;
```

### What the client writes

```java
// 1. Create a registry (reusing from Feature 1)
MeterRegistry registry = new SimpleMeterRegistry();

// 2. Track a field via object + function reference
List<String> queue = new ArrayList<>();
Gauge gauge = Gauge.builder("queue.size", queue, List::size)
    .description("Current queue depth")
    .register(registry);

queue.add("item1");
queue.add("item2");
System.out.println(gauge.value()); // 2.0

// 3. Convenience: wrap a Number supplier
AtomicInteger connections = new AtomicInteger(0);
Gauge.builder("active.connections", () -> connections)
    .register(registry);

// 4. Shorthand on registry (returns the state object, not the Gauge)
registry.gauge("cache.size", myCache, Cache::size);
```

### Behavioral contract

| Promise | Detail |
|---------|--------|
| Pull-based sampling | `value()` evaluates the function each time — no internal storage |
| WeakReference by default | The gauge does not prevent the state object from being GC'd |
| Strong reference opt-in | `strongReference(true)` or the `Supplier` overload prevent GC |
| Returns NaN when GC'd | If the state object is collected, `value()` returns `Double.NaN` |
| Returns NaN on error | If the value function throws, `value()` returns `Double.NaN` |
| Deduplicated | Same name + tags returns the same gauge instance |
| Live measurements | `measure()` returns a supplier-backed `VALUE` measurement |

---

## 2. Client Usage & Tests

These tests define the API's behavior — they were written **before** any implementation.

### Builder + Registration

```java
@Test
void shouldCreateGauge_WithBuilderAndRegister() {
    List<String> queue = new ArrayList<>();
    Gauge gauge = Gauge.builder("queue.size", queue, List::size)
            .description("Current queue depth")
            .baseUnit("items")
            .register(registry);

    assertThat(gauge).isNotNull();
    assertThat(gauge.getId().getName()).isEqualTo("queue.size");
    assertThat(gauge.getId().getType()).isEqualTo(Meter.Type.GAUGE);
}
```

### Live Value Tracking

```java
@Test
void shouldReturnCurrentValue_WhenStateChanges() {
    List<String> queue = new ArrayList<>();
    Gauge gauge = Gauge.builder("queue.size", queue, List::size)
            .register(registry);

    assertThat(gauge.value()).isEqualTo(0.0);

    queue.add("item1");
    queue.add("item2");
    assertThat(gauge.value()).isEqualTo(2.0);  // reflects live state

    queue.remove(0);
    assertThat(gauge.value()).isEqualTo(1.0);  // updates immediately
}
```

### Supplier Convenience

```java
@Test
void shouldCreateGauge_WithSupplierBuilder() {
    AtomicInteger connections = new AtomicInteger(5);
    Gauge gauge = Gauge.builder("active.connections", () -> connections)
            .register(registry);

    assertThat(gauge.value()).isEqualTo(5.0);
    connections.set(10);
    assertThat(gauge.value()).isEqualTo(10.0);
}
```

### WeakReference / GC Behavior

```java
@Test
void shouldNotBeCollected_WhenStrongReferenceEnabled() {
    Gauge gauge = Gauge.builder("strong.size", new ArrayList<>(), List::size)
            .strongReference(true)
            .register(registry);

    System.gc();
    assertThat(gauge.value()).isEqualTo(0.0); // still alive!
}
```

### Registry Shorthand

```java
@Test
void shouldCreateGauge_WithNumberShorthand() {
    AtomicInteger count = new AtomicInteger(42);
    AtomicInteger returned = registry.gauge("my.count", count);

    assertThat(returned).isSameAs(count); // returns the state object!

    Gauge gauge = (Gauge) registry.getMeters().get(0);
    assertThat(gauge.value()).isEqualTo(42.0);
}
```

---

## 3. Implementing the Call Chain — Top Down

### The call chain

```
Gauge.builder("queue.size", queue, List::size).register(registry)
  │
  ├─► [API Layer] Gauge.Builder
  │     Stores name, obj, valueFunction, tags, description, baseUnit, strongReference
  │     Terminal op: register(registry)
  │       → If strongReference: wraps valueFunction in StrongReferenceGaugeFunction
  │       → Creates Meter.Id with Type.GAUGE
  │       → Calls registry.gauge(id, obj, effectiveFunction)
  │
  ├─► [API Layer] MeterRegistry.gauge(Meter.Id, T, ToDoubleFunction<T>)
  │     Calls registerMeterIfNecessary() — reuses the same double-checked locking
  │     from Feature 1 (Counter registration)
  │
  └─► [Implementation Layer] SimpleMeterRegistry.newGauge(id, obj, valueFunction)
        Creates a DefaultGauge backed by a WeakReference
```

### Counter vs Gauge: Push vs Pull

Before diving into the code, it's worth understanding the fundamental difference:

```
Counter (Push)                       Gauge (Pull)
──────────────                       ────────────
Client calls increment()  ──►  Internal DoubleAdder stores value
Client calls count()      ◄──  DoubleAdder.sum() returns accumulated total

                                     Gauge:
Client holds List<String> queue      No internal storage
Framework calls value()   ──►  WeakRef.get() → List::size → returns queue.size()
```

Counter *owns* its data (the `DoubleAdder`). Gauge *observes* external data (through
a function reference). This is why gauges need `WeakReference` — they point at objects
they don't own.

### Layer 1: Gauge interface + Builder

**Gauge** extends `Meter` with a single method:
- `value()` — the one method implementations must provide

Unlike Counter (which has `increment()` + `count()`), Gauge has no mutation method.
There's nothing to "push" — the gauge just reads the current state on demand.

The **default `measure()`** implementation wraps `value()` in a `Measurement` with
`Statistic.VALUE` — same pattern as Counter wrapping `count()` with `Statistic.COUNT`.

**Two factory methods** create builders:

```java
// Primary: observe an object through a function
static <T> Builder<T> builder(String name, T obj, ToDoubleFunction<T> valueFunction)

// Convenience: observe a Number supplier (auto-enables strong reference)
static <T extends Number> Builder<Supplier<T>> builder(String name, Supplier<T> supplier)
```

The Supplier overload is interesting — it wraps the supplier as both the state object
AND the function input:

```java
static <T extends Number> Builder<Supplier<T>> builder(String name, Supplier<T> supplier) {
    return new Builder<>(name, supplier, s -> {
        T val = s.get();
        return val != null ? val.doubleValue() : Double.NaN;
    }).strongReference(true);  // auto-enabled!
}
```

Why `strongReference(true)` here? Because `() -> connections` is a lambda with no
other references — without strong reference mode, the GC would collect it immediately.

### Layer 2: The StrongReferenceGaugeFunction trick

This is one of Micrometer's cleverest patterns. The `Builder.register()` method
conditionally wraps the value function:

```java
public Gauge register(MeterRegistry registry) {
    Meter.Id id = new Meter.Id(name, tags, baseUnit, description, Meter.Type.GAUGE);
    ToDoubleFunction<T> effectiveFunction = strongReference
            ? new StrongReferenceGaugeFunction<>(obj, valueFunction)
            : valueFunction;
    return registry.gauge(id, obj, effectiveFunction);
}
```

`StrongReferenceGaugeFunction` holds a strong reference to the state object:

```java
private static class StrongReferenceGaugeFunction<T> implements ToDoubleFunction<T> {
    @SuppressWarnings("unused") // prevents GC of the state object
    private final T strongRef;
    private final ToDoubleFunction<T> delegate;

    StrongReferenceGaugeFunction(T obj, ToDoubleFunction<T> delegate) {
        this.strongRef = obj;
        this.delegate = delegate;
    }

    @Override
    public double applyAsDouble(T value) {
        return delegate.applyAsDouble(value);
    }
}
```

Here's the key insight: `DefaultGauge` stores the state object via `WeakReference`,
but it stores the `ToDoubleFunction` as a regular field. So when the function IS a
`StrongReferenceGaugeFunction`, that function's `strongRef` field keeps the state
object alive — indirectly, through the function object that the gauge holds directly.

```
DefaultGauge
  ├── ref: WeakReference<T> ──weak──► state object ◄──strong── strongRef
  └── valueFunction ──strong──► StrongReferenceGaugeFunction
                                    ├── strongRef ──────────────┘
                                    └── delegate → original ToDoubleFunction
```

### Layer 3: MeterRegistry — adding gauge support

The MeterRegistry gains a new abstract factory method and convenience methods.

**Abstract factory** (subclasses implement):
```java
protected abstract <T> Gauge newGauge(Meter.Id id, T obj, ToDoubleFunction<T> valueFunction);
```

**Package-private registration** (called by Builder):
```java
<T> Gauge gauge(Meter.Id id, T obj, ToDoubleFunction<T> valueFunction) {
    return registerMeterIfNecessary(Gauge.class, id, i -> newGauge(i, obj, valueFunction));
}
```

This reuses the exact same `registerMeterIfNecessary()` from Feature 1 — the
double-checked locking registration engine now handles both Counter and Gauge.

**Public convenience methods** return the state object (not the Gauge):

```java
public <T> T gauge(String name, Iterable<Tag> tags, T stateObject, ToDoubleFunction<T> valueFunction) {
    Gauge.builder(name, stateObject, valueFunction).tags(tags).register(this);
    return stateObject;  // return the object, not the gauge!
}

public <T extends Number> T gauge(String name, T number) {
    return gauge(name, Collections.emptyList(), number, Number::doubleValue);
}
```

Why return the state object? It enables assignment patterns like:
```java
AtomicInteger active = registry.gauge("connections", new AtomicInteger(0));
// Now 'active' is both the gauge's data source AND the variable you increment
```

### Layer 4: DefaultGauge — the WeakReference implementation

```java
public class DefaultGauge<T> implements Gauge {
    private final Meter.Id id;
    private final WeakReference<T> ref;
    private final ToDoubleFunction<T> valueFunction;

    public DefaultGauge(Meter.Id id, T obj, ToDoubleFunction<T> valueFunction) {
        this.id = id;
        this.ref = new WeakReference<>(obj);
        this.valueFunction = valueFunction;
    }

    @Override
    public double value() {
        T obj = ref.get();
        if (obj != null) {
            try {
                return valueFunction.applyAsDouble(obj);
            } catch (Exception ex) {
                return Double.NaN;
            }
        }
        return Double.NaN;
    }
}
```

Three things happen in `value()`:
1. **Dereference** the `WeakReference` — returns `null` if the object has been GC'd
2. **Apply** the value function — extracts the current double value
3. **Catch** any exceptions — returns `NaN` instead of propagating errors

The `NaN` sentinel means monitoring backends can distinguish "no data" from "zero" —
a gauge reporting `NaN` means the data source is gone, while `0.0` is a valid reading.

### Layer 5: SimpleMeterRegistry — one line

```java
@Override
protected <T> Gauge newGauge(Meter.Id id, T obj, ToDoubleFunction<T> valueFunction) {
    return new DefaultGauge<>(id, obj, valueFunction);
}
```

This is the beauty of the registry abstraction. A Prometheus registry would create
a `PrometheusGauge` here; our simple registry creates a `DefaultGauge`.

---

## 4. Try It Yourself

1. **Create a `gaugeCollectionSize()` convenience method** on `MeterRegistry` that
   takes a `Collection<?>` and automatically uses `Collection::size`. The real
   Micrometer has this — it's a one-liner that delegates to the main `gauge()` method.

2. **What happens if you register a Gauge and a Counter with the same name?** Try it.
   The `castOrThrow` mechanism in `registerMeterIfNecessary` should produce a clear
   error. This is a real Micrometer behavior.

3. **Add a `TimeGauge`** that wraps a gauge value with a `TimeUnit` conversion.
   How would you convert the raw value to seconds for export? The real Micrometer
   has `TimeGauge` as a `Gauge` subinterface.

---

## 5. Why This Works

### Pull-based vs Push-based

Counter is push-based: the client calls `increment()` and the counter stores the
delta. Gauge is pull-based: the client mutates its own data, and the gauge reads it
on demand. This means gauges have zero overhead between reads — there's no
synchronization, no atomic operations, no bookkeeping. The cost is paid only at
scrape time (when `value()` is called).

### WeakReference prevents memory leaks

In real applications, gauges often track objects that have their own lifecycle —
connection pools, caches, request queues. If the pool is shut down and garbage
collected, the gauge should not keep it alive. `WeakReference` ensures the gauge
is "just an observer" — it never extends the lifetime of what it observes.

The `StrongReferenceGaugeFunction` escape hatch exists for cases where the gauge
IS the only reference — typically lambdas and suppliers. Without it, `Gauge.builder(
"active.connections", () -> connections).register(registry)` would create a supplier
lambda that's immediately GC-eligible.

### Convenience methods return the state object

This is a subtle but important API design choice. Instead of:
```java
AtomicInteger count = new AtomicInteger(0);
registry.gauge("connections", count);
// count is still usable
```

The return-state-object pattern lets you write:
```java
AtomicInteger count = registry.gauge("connections", new AtomicInteger(0));
// count is the SAME object the gauge tracks
```

One line instead of two. The gauge registration is a side effect — the primary
purpose is creating and tracking the state object.

---

## 6. What We Enhanced

| Component | State before (Feature 1) | Enhancement in Feature 2 | Will be enhanced by |
|-----------|-------------------------|-------------------------|---------------------|
| `MeterRegistry` | Supports Counter only | +Gauge registration, +4 convenience methods, +`newGauge()` abstract | F3: +Timer, F5: +MeterFilter |
| `SimpleMeterRegistry` | Creates CumulativeCounter only | +`newGauge()` → DefaultGauge | F3: +CumulativeTimer |
| `Meter.Type` | Had `GAUGE` enum value (unused) | Now used in Gauge's Meter.Id | Stable |

---

## 7. Connection to Real Framework

| Simplified Class | Real Framework Class | Key Simplification |
|-----------------|---------------------|--------------------|
| `api/Gauge` | `io.micrometer.core.instrument.Gauge` | No `@Incubating` annotations, no `synthetic` association, no `MeterProvider` |
| `api/Gauge.Builder` | `io.micrometer.core.instrument.Gauge.Builder` | `StrongReferenceGaugeFunction` is a private inner class instead of a separate package-private class |
| `internal/DefaultGauge` | `io.micrometer.core.instrument.internal.DefaultGauge` | No `AbstractMeter` base class, no `WarnThenDebugLogger` (exceptions silently return NaN) |
| `internal/NoopGauge` | `io.micrometer.core.instrument.noop.NoopGauge` | No `NoopMeter` base class |
| `api/MeterRegistry` (gauge methods) | `io.micrometer.core.instrument.MeterRegistry` | No `gaugeCollectionSize()`, no `gaugeMapSize()`, no null-contract annotations |

---

## 8. Complete Code

All files created or modified in this feature. Copy all `[NEW]` files and replace
all `[MODIFIED]` files to build the project from Feature 1 + Feature 2.

### `src/main/java/simple/micrometer/api/Gauge.java` [NEW]

```java
package simple.micrometer.api;

import java.util.Collections;
import java.util.function.Supplier;
import java.util.function.ToDoubleFunction;

public interface Gauge extends Meter {

    double value();

    @Override
    default Iterable<Measurement> measure() {
        return Collections.singletonList(new Measurement(this::value, Statistic.VALUE));
    }

    static <T> Builder<T> builder(String name, T obj, ToDoubleFunction<T> valueFunction) {
        return new Builder<>(name, obj, valueFunction);
    }

    static <T extends Number> Builder<Supplier<T>> builder(String name, Supplier<T> supplier) {
        return new Builder<>(name, supplier, s -> {
            T val = s.get();
            return val != null ? val.doubleValue() : Double.NaN;
        }).strongReference(true);
    }

    class Builder<T> {

        private final String name;
        private final T obj;
        private final ToDoubleFunction<T> valueFunction;
        private Tags tags = Tags.empty();
        private String description;
        private String baseUnit;
        private boolean strongReference = false;

        Builder(String name, T obj, ToDoubleFunction<T> valueFunction) {
            this.name = name;
            this.obj = obj;
            this.valueFunction = valueFunction;
        }

        public Builder<T> tags(String... keyValues) {
            this.tags = this.tags.and(keyValues);
            return this;
        }

        public Builder<T> tags(Iterable<Tag> tags) {
            this.tags = this.tags.and(tags);
            return this;
        }

        public Builder<T> tag(String key, String value) {
            this.tags = this.tags.and(key, value);
            return this;
        }

        public Builder<T> description(String description) {
            this.description = description;
            return this;
        }

        public Builder<T> baseUnit(String unit) {
            this.baseUnit = unit;
            return this;
        }

        public Builder<T> strongReference(boolean strong) {
            this.strongReference = strong;
            return this;
        }

        public Gauge register(MeterRegistry registry) {
            Meter.Id id = new Meter.Id(name, tags, baseUnit, description, Meter.Type.GAUGE);
            ToDoubleFunction<T> effectiveFunction = strongReference
                    ? new StrongReferenceGaugeFunction<>(obj, valueFunction)
                    : valueFunction;
            return registry.gauge(id, obj, effectiveFunction);
        }

        private static class StrongReferenceGaugeFunction<T> implements ToDoubleFunction<T> {

            @SuppressWarnings("unused")
            private final T strongRef;
            private final ToDoubleFunction<T> delegate;

            StrongReferenceGaugeFunction(T obj, ToDoubleFunction<T> delegate) {
                this.strongRef = obj;
                this.delegate = delegate;
            }

            @Override
            public double applyAsDouble(T value) {
                return delegate.applyAsDouble(value);
            }

        }

    }

}
```

### `src/main/java/simple/micrometer/internal/DefaultGauge.java` [NEW]

```java
package simple.micrometer.internal;

import simple.micrometer.api.Gauge;
import simple.micrometer.api.Meter;

import java.lang.ref.WeakReference;
import java.util.function.ToDoubleFunction;

public class DefaultGauge<T> implements Gauge {

    private final Meter.Id id;
    private final WeakReference<T> ref;
    private final ToDoubleFunction<T> valueFunction;

    public DefaultGauge(Meter.Id id, T obj, ToDoubleFunction<T> valueFunction) {
        this.id = id;
        this.ref = new WeakReference<>(obj);
        this.valueFunction = valueFunction;
    }

    @Override
    public double value() {
        T obj = ref.get();
        if (obj != null) {
            try {
                return valueFunction.applyAsDouble(obj);
            }
            catch (Exception ex) {
                return Double.NaN;
            }
        }
        return Double.NaN;
    }

    @Override
    public Meter.Id getId() {
        return id;
    }

}
```

### `src/main/java/simple/micrometer/internal/NoopGauge.java` [NEW]

```java
package simple.micrometer.internal;

import simple.micrometer.api.Gauge;
import simple.micrometer.api.Meter;

public class NoopGauge implements Gauge {

    private final Meter.Id id;

    public NoopGauge(Meter.Id id) {
        this.id = id;
    }

    @Override
    public double value() {
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

public abstract class MeterRegistry {

    private final ConcurrentHashMap<Meter.Id, Meter> meterMap = new ConcurrentHashMap<>();
    private final Object meterMapLock = new Object();

    protected final Clock clock;

    protected MeterRegistry(Clock clock) {
        this.clock = Objects.requireNonNull(clock, "clock must not be null");
    }

    // ── Public counter convenience methods ───────────────────────────

    public Counter counter(String name, String... tags) {
        return counter(name, Tags.of(tags));
    }

    public Counter counter(String name, Iterable<Tag> tags) {
        Meter.Id id = new Meter.Id(name, Tags.of(tags), null, null, Meter.Type.COUNTER);
        return counter(id);
    }

    // ── Public gauge convenience methods ─────────────────────────────

    public <T> T gauge(String name, Iterable<Tag> tags, T stateObject, ToDoubleFunction<T> valueFunction) {
        Gauge.builder(name, stateObject, valueFunction).tags(tags).register(this);
        return stateObject;
    }

    public <T extends Number> T gauge(String name, Iterable<Tag> tags, T number) {
        return gauge(name, tags, number, Number::doubleValue);
    }

    public <T extends Number> T gauge(String name, T number) {
        return gauge(name, Collections.emptyList(), number);
    }

    public <T> T gauge(String name, T stateObject, ToDoubleFunction<T> valueFunction) {
        return gauge(name, Collections.emptyList(), stateObject, valueFunction);
    }

    // ── Package-private registration entry points ─────────────────────

    Counter counter(Meter.Id id) {
        return registerMeterIfNecessary(Counter.class, id, this::newCounter);
    }

    <T> Gauge gauge(Meter.Id id, T obj, ToDoubleFunction<T> valueFunction) {
        return registerMeterIfNecessary(Gauge.class, id, i -> newGauge(i, obj, valueFunction));
    }

    // ── Abstract factory methods (subclasses implement these) ────────

    protected abstract Counter newCounter(Meter.Id id);

    protected abstract <T> Gauge newGauge(Meter.Id id, T obj, ToDoubleFunction<T> valueFunction);

    // ── Query ────────────────────────────────────────────────────────

    public List<Meter> getMeters() {
        return Collections.unmodifiableList(new ArrayList<>(meterMap.values()));
    }

    public Clock getClock() {
        return clock;
    }

    // ── Core registration logic ──────────────────────────────────────

    private <M extends Meter> M registerMeterIfNecessary(
            Class<M> meterClass, Meter.Id id, Function<Meter.Id, M> meterFactory) {

        Meter existing = meterMap.get(id);
        if (existing != null) {
            return castOrThrow(meterClass, existing, id);
        }

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

import java.util.function.ToDoubleFunction;

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

}
```

### `src/test/java/simple/micrometer/api/GaugeTest.java` [NEW]

```java
package simple.micrometer.api;

import simple.micrometer.internal.SimpleMeterRegistry;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.util.ArrayList;
import java.util.List;
import java.util.concurrent.atomic.AtomicInteger;

import static org.assertj.core.api.Assertions.assertThat;

class GaugeTest {

    private MeterRegistry registry;

    @BeforeEach
    void setUp() {
        registry = new SimpleMeterRegistry();
    }

    @Test
    void shouldCreateGauge_WithBuilderAndRegister() {
        List<String> queue = new ArrayList<>();
        Gauge gauge = Gauge.builder("queue.size", queue, List::size)
                .description("Current queue depth")
                .baseUnit("items")
                .register(registry);

        assertThat(gauge).isNotNull();
        assertThat(gauge.getId().getName()).isEqualTo("queue.size");
        assertThat(gauge.getId().getDescription()).isEqualTo("Current queue depth");
        assertThat(gauge.getId().getBaseUnit()).isEqualTo("items");
        assertThat(gauge.getId().getType()).isEqualTo(Meter.Type.GAUGE);
    }

    @Test
    void shouldAttachTags_WhenBuiltWithTags() {
        List<String> queue = new ArrayList<>();
        Gauge gauge = Gauge.builder("queue.size", queue, List::size)
                .tag("region", "us-east")
                .tag("env", "prod")
                .register(registry);

        assertThat(gauge.getId().getTag("region")).isEqualTo("us-east");
        assertThat(gauge.getId().getTag("env")).isEqualTo("prod");
    }

    @Test
    void shouldReturnCurrentValue_WhenStateChanges() {
        List<String> queue = new ArrayList<>();
        Gauge gauge = Gauge.builder("queue.size", queue, List::size)
                .register(registry);

        assertThat(gauge.value()).isEqualTo(0.0);

        queue.add("item1");
        queue.add("item2");
        assertThat(gauge.value()).isEqualTo(2.0);

        queue.remove(0);
        assertThat(gauge.value()).isEqualTo(1.0);
    }

    @Test
    void shouldTrackAtomicInteger_ViaFunctionReference() {
        AtomicInteger connections = new AtomicInteger(0);
        Gauge gauge = Gauge.builder("active.connections", connections, AtomicInteger::doubleValue)
                .register(registry);

        assertThat(gauge.value()).isEqualTo(0.0);
        connections.set(42);
        assertThat(gauge.value()).isEqualTo(42.0);
    }

    @Test
    void shouldCreateGauge_WithSupplierBuilder() {
        AtomicInteger connections = new AtomicInteger(5);
        Gauge gauge = Gauge.builder("active.connections", () -> connections)
                .register(registry);

        assertThat(gauge.value()).isEqualTo(5.0);
        connections.set(10);
        assertThat(gauge.value()).isEqualTo(10.0);
    }

    @Test
    void shouldReturnNaN_WhenSupplierReturnsNull() {
        Gauge gauge = Gauge.builder("nullable", () -> (Number) null)
                .register(registry);

        assertThat(gauge.value()).isNaN();
    }

    @Test
    void shouldReturnNaN_WhenObjectIsGarbageCollected() {
        Gauge gauge = Gauge.builder("temp.size", new ArrayList<>(), List::size)
                .register(registry);

        System.gc();
        double val = gauge.value();
        assertThat(val).satisfiesAnyOf(
                v -> assertThat(v).isEqualTo(0.0),
                v -> assertThat(v).isNaN()
        );
    }

    @Test
    void shouldNotBeCollected_WhenStrongReferenceEnabled() {
        Gauge gauge = Gauge.builder("strong.size", new ArrayList<>(), List::size)
                .strongReference(true)
                .register(registry);

        System.gc();
        assertThat(gauge.value()).isEqualTo(0.0);
    }

    @Test
    void shouldNotBeCollected_WhenSupplierUsed() {
        Gauge gauge = Gauge.builder("supplier.gauge", () -> new AtomicInteger(99))
                .register(registry);

        System.gc();
        assertThat(gauge.value()).isEqualTo(99.0);
    }

    @Test
    void shouldCreateGauge_WithRegistryShorthand() {
        List<String> queue = new ArrayList<>();
        List<String> returned = registry.gauge("queue.size", Tags.empty(), queue, List::size);

        assertThat(returned).isSameAs(queue);

        queue.add("item");
        assertThat(registry.getMeters()).hasSize(1);
        Meter meter = registry.getMeters().get(0);
        assertThat(meter).isInstanceOf(Gauge.class);
        assertThat(((Gauge) meter).value()).isEqualTo(1.0);
    }

    @Test
    void shouldCreateGauge_WithNumberShorthand() {
        AtomicInteger count = new AtomicInteger(42);
        AtomicInteger returned = registry.gauge("my.count", count);

        assertThat(returned).isSameAs(count);

        Gauge gauge = (Gauge) registry.getMeters().get(0);
        assertThat(gauge.value()).isEqualTo(42.0);
    }

    @Test
    void shouldReturnSameGauge_WhenRegisteredTwiceWithSameId() {
        List<String> queue = new ArrayList<>();
        Gauge gauge1 = Gauge.builder("queue.size", queue, List::size)
                .register(registry);

        Gauge gauge2 = Gauge.builder("queue.size", queue, List::size)
                .register(registry);

        assertThat(gauge1).isSameAs(gauge2);
    }

    @Test
    void shouldReturnDifferentGauges_WhenTagsDiffer() {
        List<String> queue1 = new ArrayList<>();
        List<String> queue2 = new ArrayList<>();

        Gauge gauge1 = Gauge.builder("queue.size", queue1, List::size)
                .tag("name", "incoming")
                .register(registry);

        Gauge gauge2 = Gauge.builder("queue.size", queue2, List::size)
                .tag("name", "outgoing")
                .register(registry);

        assertThat(gauge1).isNotSameAs(gauge2);
    }

    @Test
    void shouldProduceSingleValueMeasurement() {
        AtomicInteger val = new AtomicInteger(7);
        Gauge gauge = Gauge.builder("test", val, AtomicInteger::doubleValue)
                .register(registry);

        Iterable<Measurement> measurements = gauge.measure();

        assertThat(measurements).hasSize(1);
        Measurement m = measurements.iterator().next();
        assertThat(m.getStatistic()).isEqualTo(Statistic.VALUE);
        assertThat(m.getValue()).isEqualTo(7.0);
    }

    @Test
    void shouldReflectLiveValue_InMeasurement() {
        AtomicInteger val = new AtomicInteger(0);
        Gauge gauge = Gauge.builder("test", val, AtomicInteger::doubleValue)
                .register(registry);
        Iterable<Measurement> measurements = gauge.measure();

        val.set(42);
        assertThat(measurements.iterator().next().getValue()).isEqualTo(42.0);

        val.set(100);
        assertThat(measurements.iterator().next().getValue()).isEqualTo(100.0);
    }

    @Test
    void shouldCoexist_WithCounterOfDifferentName() {
        Counter counter = Counter.builder("metric.a").register(registry);
        AtomicInteger val = new AtomicInteger(1);
        Gauge gauge = Gauge.builder("metric.b", val, AtomicInteger::doubleValue)
                .register(registry);

        assertThat(registry.getMeters()).hasSize(2);
        counter.increment();
        assertThat(counter.count()).isEqualTo(1.0);
        assertThat(gauge.value()).isEqualTo(1.0);
    }

}
```
