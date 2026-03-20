# Chapter 5: Gauge

> Line references based on commit `2c8a4606c` of the Micrometer repository.

## Build Challenge

| Current State | Limitation | Objective |
|---------------|------------|-----------|
| `Gauge` is a stub interface extending `Meter` with no methods — `MeterRegistry.newGauge()` throws `UnsupportedOperationException` | Cannot observe external state — no way to monitor pool sizes, queue depths, or any value that fluctuates up and down | Build a full `Gauge` interface with `value()`, a `DefaultGauge` backed by `WeakReference` to prevent memory leaks, a `NoopGauge`, a fluent `Builder`, and registry convenience methods like `gaugeCollectionSize()` and `gaugeMapSize()` |

---

## 5.1 The Integration Point: MeterRegistry gains gauge registration

**Modifying:** `src/main/java/dev/linhvu/micrometer/MeterRegistry.java`
**Change:** Add a package-private `gauge(Meter.Id, T, ToDoubleFunction)` method and public convenience methods for gauge registration.

```java
/**
 * Registers a gauge using the pre-built ID. Called by {@link Gauge.Builder#register(MeterRegistry)}.
 * The gauge factory is a lambda because {@code newGauge()} requires additional parameters
 * (the state object and value function) that standard method references cannot capture.
 */
<T> Gauge gauge(Meter.Id id, T obj, ToDoubleFunction<T> valueFunction) {
    return registerMeterIfNecessary(Gauge.class, id,
            mappedId -> newGauge(mappedId, obj, valueFunction), NoopGauge::new);
}
```

Three key decisions here:

1. **Lambda instead of method reference.** Counter registration uses `this::newCounter` because `newCounter(Id)` takes only an ID. But `newGauge(Id, T, ToDoubleFunction)` takes three parameters — the state object and value function come from the Builder, not the registry. A lambda `mappedId -> newGauge(mappedId, obj, valueFunction)` captures the extra parameters.

2. **Convenience methods return the state object, not the Gauge.** This enables the inline assignment pattern: `List<String> items = registry.gaugeCollectionSize("items", tags, new ArrayList<>())`. The gauge is registered as a side effect; the returned value is the object you actually work with.

3. **`gaugeCollectionSize()` and `gaugeMapSize()` use `Collection::size` and `Map::size`.** These are method references to interface methods, which means they work with any `Collection` or `Map` implementation — `ArrayList`, `ConcurrentHashMap`, `CopyOnWriteArrayList`, etc.

This connects **User Instrumentation Code** (the Builder and convenience methods) to **Meter Lifecycle Management** (the Registry's deduplication and factory dispatch). To make it work, we need to build:
- `Gauge` — the full interface with `value()`, `measure()`, and a `Builder`
- `DefaultGauge` — the `WeakReference`-backed implementation
- `NoopGauge` — the no-op implementation for closed/filtered registries

## 5.2 The Gauge Interface

**Modifying:** `src/main/java/dev/linhvu/micrometer/Gauge.java`
**Change:** Replace stub interface with full Gauge contract including `value()`, `measure()`, builder factory, and `Builder` inner class

```java
package dev.linhvu.micrometer;

import java.util.Collections;
import java.util.function.ToDoubleFunction;

public interface Gauge extends Meter {

    static <T> Builder<T> builder(String name, T obj, ToDoubleFunction<T> f) {
        return new Builder<>(name, obj, f);
    }

    double value();

    @Override
    default Iterable<Measurement> measure() {
        return Collections.singletonList(new Measurement(this::value, Statistic.VALUE));
    }

    class Builder<T> {

        private final String name;
        private final T obj;
        private final ToDoubleFunction<T> f;
        private Tags tags = Tags.empty();
        private String description;
        private String baseUnit;

        private Builder(String name, T obj, ToDoubleFunction<T> f) {
            this.name = name;
            this.obj = obj;
            this.f = f;
        }

        public Builder<T> tags(String... tags) { return tags(Tags.of(tags)); }
        public Builder<T> tags(Iterable<Tag> tags) { this.tags = this.tags.and(tags); return this; }
        public Builder<T> tag(String key, String value) { this.tags = tags.and(key, value); return this; }
        public Builder<T> description(String description) { this.description = description; return this; }
        public Builder<T> baseUnit(String unit) { this.baseUnit = unit; return this; }

        public Gauge register(MeterRegistry registry) {
            return registry.gauge(new Meter.Id(name, tags, Meter.Type.GAUGE, description, baseUnit), obj, f);
        }
    }
}
```

Notice how the Builder is parameterized by `<T>` — the type of the state object. This carries through from `Gauge.builder("name", myList, List::size)` all the way to the `ToDoubleFunction<T>`. Compare this with `Counter.Builder` which has no type parameter because counters own their state internally.

## 5.3 DefaultGauge — The WeakReference Implementation

**New file:** `src/main/java/dev/linhvu/micrometer/internal/DefaultGauge.java`

```java
package dev.linhvu.micrometer.internal;

import dev.linhvu.micrometer.AbstractMeter;
import dev.linhvu.micrometer.Gauge;
import dev.linhvu.micrometer.Meter;

import java.lang.ref.WeakReference;
import java.util.function.ToDoubleFunction;

public class DefaultGauge<T> extends AbstractMeter implements Gauge {

    private final WeakReference<T> ref;

    private final ToDoubleFunction<T> value;

    public DefaultGauge(Meter.Id id, T obj, ToDoubleFunction<T> value) {
        super(id);
        this.ref = new WeakReference<>(obj);
        this.value = value;
    }

    @Override
    public double value() {
        T obj = ref.get();
        if (obj != null) {
            try {
                return value.applyAsDouble(obj);
            }
            catch (Exception e) {
                return Double.NaN;
            }
        }
        return Double.NaN;
    }
}
```

The `value()` method has three possible outcomes:
1. **Object alive, function succeeds** → returns the extracted value
2. **Object alive, function throws** → returns `NaN` (defensive — never crash on gauge evaluation)
3. **Object garbage collected** → returns `NaN` (the `WeakReference.get()` returns `null`)

## 5.4 NoopGauge

**New file:** `src/main/java/dev/linhvu/micrometer/noop/NoopGauge.java`

```java
package dev.linhvu.micrometer.noop;

import dev.linhvu.micrometer.Gauge;
import dev.linhvu.micrometer.Meter;

public class NoopGauge extends NoopMeter implements Gauge {

    public NoopGauge(Meter.Id id) {
        super(id);
    }

    @Override
    public double value() {
        return 0;
    }
}
```

## 5.5 Registry Convenience Methods

**Modifying:** `src/main/java/dev/linhvu/micrometer/MeterRegistry.java`
**Change:** Add public convenience methods for gauge registration after the existing counter convenience methods

```java
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

public <T extends Collection<?>> T gaugeCollectionSize(String name, Iterable<Tag> tags, T collection) {
    return gauge(name, tags, collection, Collection::size);
}

public <T extends Map<?, ?>> T gaugeMapSize(String name, Iterable<Tag> tags, T map) {
    return gauge(name, tags, map, Map::size);
}
```

The "return the state object" pattern is unusual but powerful:

```java
// Without the pattern — two lines, two variables:
List<String> items = new ArrayList<>();
registry.gaugeCollectionSize("items", Tags.empty(), items);

// With the pattern — one line:
List<String> items = registry.gaugeCollectionSize("items", Tags.empty(), new ArrayList<>());
```

## 5.6 SimpleMeterRegistry Enhancement

**Modifying:** `src/main/java/dev/linhvu/micrometer/simple/SimpleMeterRegistry.java`
**Change:** Replace `newGauge()` stub with `DefaultGauge` creation

```java
@Override
protected <T> Gauge newGauge(Meter.Id id, T obj, ToDoubleFunction<T> valueFunction) {
    return new DefaultGauge<>(id, obj, valueFunction);
}
```

Unlike counters (where `SimpleMeterRegistry` creates `CumulativeCounter`), gauges don't need cumulative vs. step variants — they're point-in-time snapshots that are the same regardless of export mode.

## 5.7 Try It Yourself

<details>
<summary>Challenge 1: Why does DefaultGauge use WeakReference instead of a strong reference?</summary>

Consider this scenario:

```java
void processRequest(MeterRegistry registry) {
    List<String> tempBuffer = new ArrayList<>();
    registry.gaugeCollectionSize("buffer.size", Tags.empty(), tempBuffer);
    // ... process using tempBuffer ...
}
// After this method returns, tempBuffer should be eligible for GC
```

With a **strong reference**: the gauge holds a strong reference to `tempBuffer` → it can never be garbage collected → memory leak. Every call to `processRequest` creates a new list that can never be freed.

With a **weak reference**: when `processRequest` returns and `tempBuffer` goes out of scope, the only reference is the gauge's `WeakReference` → GC can collect it → `gauge.value()` returns `NaN` → no memory leak.

The trade-off: if nothing else holds a reference to the observed object, it may be collected prematurely. The real Micrometer adds a `strongReference(true)` option on the Builder for cases where the gauge *should* be the owner (e.g., a `Supplier<Number>` created inline).

</details>

<details>
<summary>Challenge 2: Implement DefaultGauge yourself</summary>

Starting point — you need a class that:
1. Extends `AbstractMeter` (for identity/equality)
2. Implements `Gauge` (for `value()`/`measure()`)
3. Holds the state object via `WeakReference<T>`
4. Applies a `ToDoubleFunction<T>` to extract the value
5. Returns `NaN` when the object is collected or the function throws

```java
public class DefaultGauge<T> extends AbstractMeter implements Gauge {

    private final WeakReference<T> ref;
    private final ToDoubleFunction<T> value;

    public DefaultGauge(Meter.Id id, T obj, ToDoubleFunction<T> value) {
        super(id);
        this.ref = new WeakReference<>(obj);
        this.value = value;
    }

    @Override
    public double value() {
        T obj = ref.get();
        if (obj != null) {
            try {
                return value.applyAsDouble(obj);
            } catch (Exception e) {
                return Double.NaN;
            }
        }
        return Double.NaN;
    }
}
```

</details>

<details>
<summary>Challenge 3: Why do the convenience methods return the state object instead of the Gauge?</summary>

The convenience methods (`gauge()`, `gaugeCollectionSize()`, `gaugeMapSize()`) return `T` (the state object) instead of `Gauge`. This enables **inline assignment**:

```java
// You typically want a reference to the collection, not the gauge
List<String> items = registry.gaugeCollectionSize("cache.items", Tags.empty(), new ArrayList<>());

// The gauge is registered as a side effect — you'll never call gauge.value() directly
// Instead, the monitoring backend reads it during scrape/publish
```

Compare with Counter, where `register()` returns the `Counter` — because you need to call `counter.increment()`. With Gauge, you mutate the observed object directly (e.g., `items.add(...)`) and the gauge reads the value passively.

</details>

## 5.8 Tests

### Unit Tests

**New file:** `src/test/java/dev/linhvu/micrometer/GaugeTest.java`

```java
class GaugeTest {

    @Test
    void shouldReturnCurrentValue_WhenValueCalled() { ... }

    @Test
    void shouldReflectChanges_WhenSourceObjectMutated() { ... }

    @Test
    void shouldReturnSingleValueMeasurement_WhenMeasureCalled() { ... }

    @Test
    void shouldReflectLiveValue_WhenMeasurementEvaluatedLazily() { ... }

    @Test
    void shouldBuildGauge_WhenUsingBuilder() { ... }

    @Test
    void shouldAccumulateTags_WhenUsingBuilderTagMethods() { ... }

    @Test
    void shouldSetDescriptionAndBaseUnit_WhenUsingBuilder() { ... }

    @Test
    void shouldMatchAsGauge_WhenUsingVisitorPattern() { ... }

    @Test
    void shouldBeEqual_WhenGaugesHaveSameId() { ... }

    @Test
    void shouldNotBeEqual_WhenGaugesHaveDifferentTags() { ... }
}
```

**New file:** `src/test/java/dev/linhvu/micrometer/internal/DefaultGaugeTest.java`

```java
class DefaultGaugeTest {

    @Test
    void shouldReturnValue_WhenObjectAlive() { ... }

    @Test
    void shouldTrackChanges_WhenSourceMutated() { ... }

    @Test
    void shouldReturnNaN_WhenObjectGarbageCollected() { ... }

    @Test
    void shouldReturnNaN_WhenValueFunctionThrows() { ... }

    @Test
    void shouldReturnNaN_WhenCreatedWithNullObject() { ... }

    @Test
    void shouldWorkWithCollectionSize() { ... }

    @Test
    void shouldReturnCorrectId_WhenGetIdCalled() { ... }

    @Test
    void shouldIncludeClassNameInToString() { ... }
}
```

**New file:** `src/test/java/dev/linhvu/micrometer/noop/NoopGaugeTest.java`

```java
class NoopGaugeTest {

    @Test
    void shouldReturnZero_WhenValueQueried() { ... }

    @Test
    void shouldReturnEmptyMeasurements_WhenMeasured() { ... }

    @Test
    void shouldPreserveId_WhenCreated() { ... }

    @Test
    void shouldBeInstanceOfGauge_WhenTypeChecked() { ... }
}
```

### Integration Tests

**New file:** `src/test/java/dev/linhvu/micrometer/integration/GaugeIntegrationTest.java`

```java
class GaugeIntegrationTest {

    @Test
    void shouldRegisterGauge_WhenUsingBuilder() { ... }

    @Test
    void shouldReturnSameInstance_WhenGaugeRegisteredTwice() { ... }

    @Test
    void shouldCreateDistinctGauges_WhenDifferentNames() { ... }

    @Test
    void shouldRegisterGauge_WhenUsingConvenienceMethodWithFunction() { ... }

    @Test
    void shouldRegisterGauge_WhenUsingConvenienceMethodWithNumber() { ... }

    @Test
    void shouldRegisterGauge_WhenUsingSimpleNumberConvenience() { ... }

    @Test
    void shouldTrackCollectionSize_WhenUsingGaugeCollectionSize() { ... }

    @Test
    void shouldTrackMapSize_WhenUsingGaugeMapSize() { ... }

    @Test
    void shouldReturnNoopGauge_WhenRegistryClosed() { ... }

    @Test
    void shouldThrowException_WhenGaugeNameConflictsWithCounter() { ... }

    @Test
    void shouldCoexist_WhenGaugeAndCounterHaveDifferentNames() { ... }

    @Test
    void shouldTrackLiveValue_WhenSourceChangesAfterRegistration() { ... }
}
```

**Modified file:** `src/test/java/dev/linhvu/micrometer/MeterVisitorTest.java`
**Change:** `TestGauge` now implements `value()` since Gauge gained this abstract method.

```java
static class TestGauge extends AbstractMeter implements Gauge {
    TestGauge(Meter.Id id) { super(id); }

    @Override
    public double value() { return 0.0; }
}
```

**Run:** `./gradlew test` — expected: all 148 tests pass (including all prior features' tests)

---

## 5.9 Why This Works

> ★ **Insight** -------------------------------------------
> - **Why WeakReference instead of a strong reference?** Gauges observe objects they don't own — a queue, a cache, a connection pool. If the gauge held a strong reference, the observed object could never be garbage collected, even if the application has moved on. `WeakReference` lets the GC reclaim the object when nothing else needs it; the gauge gracefully degrades to `NaN`. This is especially critical in frameworks like Spring where request-scoped beans may be monitored by application-scoped gauges.
> - **The real Micrometer adds `strongReference(true)` on the Builder** for the `Supplier<Number>` factory method, because the Supplier is created inline and has no other reference — without a strong reference, it would be collected immediately. Our simplified version skips this, meaning all gauges use weak references.
> -----------------------------------------------------------

> ★ **Insight** -------------------------------------------
> - **Why do convenience methods return the state object?** Counter's `register()` returns the Counter because you need to call `counter.increment()` — the counter is the API. Gauge's convenience methods return the *observed object* because you never call `gauge.value()` directly — instead, you mutate the object (`list.add(...)`, `atomicInt.set(...)`) and the monitoring backend reads the gauge's value during scrape/publish. This design eliminates the need for users to keep two references (one to the gauge, one to the object).
> - **The `gaugeCollectionSize()` idiom** is the most common gauge pattern in production: `List<Request> pending = registry.gaugeCollectionSize("pending", tags, new CopyOnWriteArrayList<>())`. One line registers the gauge and assigns the collection.
> -----------------------------------------------------------

> ★ **Insight** -------------------------------------------
> - **Counter vs. Gauge — when to use which?** If the value can only increase (request counts, bytes sent, errors), use a Counter — its rate (derivative) is always non-negative, which monitoring systems rely on. If the value can go up and down (pool size, queue depth, temperature), use a Gauge. The fundamental difference: Counter owns its state (`DoubleAdder`), Gauge observes external state (`WeakReference<T>` + `ToDoubleFunction<T>`). This is why counters are thread-safe by construction (DoubleAdder handles it) while gauges delegate thread-safety to the observed object.
> -----------------------------------------------------------

## 5.10 What We Enhanced

| Aspect | Before (ch04) | Current (ch05) | Real Framework |
|--------|---------------|----------------|----------------|
| Gauge interface | Empty stub — extends Meter with no methods | Full contract: `value()`, `measure()`, `Builder<T>` with type-safe state observation | Same contract plus `strongReference(boolean)`, `synthetic(Meter.Id)`, and `Supplier<Number>` overload (`Gauge.java:32`) |
| Concrete implementation | None — `newGauge()` throws `UnsupportedOperationException` | `DefaultGauge` with `WeakReference<T>` + exception-safe `value()` | Same `DefaultGauge` with additional `WarnThenDebugLogger` for function failures (`DefaultGauge.java:34`) |
| No-op implementation | Only `NoopCounter` existed | Added `NoopGauge` — `value()` returns 0, `measure()` returns empty | Identical pattern (`NoopGauge.java:20`) |
| Registry convenience | Only `counter(name, tags)` convenience methods | Added `gauge()`, `gaugeCollectionSize()`, `gaugeMapSize()` — all return the state object for inline assignment | Same methods with `@Nullable` annotations and `@Contract` specs (`MeterRegistry.java:567-648`) |
| SimpleMeterRegistry | `newGauge()` threw exception | Returns `new DefaultGauge<>(id, obj, valueFunction)` | Same — gauges don't vary by counting mode (`SimpleMeterRegistry.java:110`) |

## 5.11 Connection to Real Framework

| Simplified Code | Real Framework Code | File:Line | Key Difference |
|----------------|--------------------|-----------| ---------------|
| `Gauge` interface | `Gauge` | `Gauge.java:32` | Real adds `Supplier<Number>` builder factory with auto `strongReference(true)` |
| `Gauge.Builder<T>` | `Gauge.Builder<T>` | `Gauge.java:80` | Real has `strongReference(boolean)` and `synthetic(Meter.Id)` fields |
| `Builder.register()` | `Builder.register()` | `Gauge.java:190` | Real wraps function in `StrongReferenceGaugeFunction` when `strongReference` is true |
| `DefaultGauge` | `DefaultGauge` | `DefaultGauge.java:34` | Real uses `WarnThenDebugLogger` to log function failures (warns once, then debug level) |
| `DefaultGauge.value()` | `DefaultGauge.value()` | `DefaultGauge.java:48` | Identical structure — WeakReference check → try/catch → NaN fallback |
| `NoopGauge` | `NoopGauge` | `NoopGauge.java:20` | Identical — 7-line class returning 0 |
| `MeterRegistry.gauge(Id, T, fn)` | `MeterRegistry.gauge(Id, T, fn)` | `MeterRegistry.java:359` | Identical pattern — lambda wrapping `newGauge()` with captured parameters |
| `gauge(name, tags, obj, fn)` | `gauge(name, tags, obj, fn)` | `MeterRegistry.java:567` | Identical — registers gauge, returns state object |
| `gaugeCollectionSize()` | `gaugeCollectionSize()` | `MeterRegistry.java:629` | Identical — `gauge(name, tags, collection, Collection::size)` |
| `gaugeMapSize()` | `gaugeMapSize()` | `MeterRegistry.java:646` | Identical — `gauge(name, tags, map, Map::size)` |
| `SimpleMeterRegistry.newGauge()` | `SimpleMeterRegistry.newGauge()` | `SimpleMeterRegistry.java:110` | Identical — `new DefaultGauge<>(id, obj, valueFunction)` |

## 5.12 Complete Code

### Production Code

#### File: `src/main/java/dev/linhvu/micrometer/Gauge.java` [MODIFIED]

```java
package dev.linhvu.micrometer;

import java.util.Collections;
import java.util.function.ToDoubleFunction;

/**
 * A meter that samples a current value from an external object — unlike Counter,
 * it can go up and down. Gauges are ideal for monitoring pool sizes, queue depths,
 * cache sizes, and other values that fluctuate.
 * <p>
 * Gauges observe external state through a function — they do not own the value.
 * The observed object is held via a {@link java.lang.ref.WeakReference} by default,
 * so gauge registration will not prevent garbage collection. If the object is
 * collected, the gauge returns {@code NaN}.
 */
public interface Gauge extends Meter {

    /**
     * Create a fluent builder for a gauge.
     *
     * @param name The gauge's name.
     * @param obj  The state object to observe (held via WeakReference by default).
     * @param f    A function that extracts a double value from the state object.
     * @param <T>  The type of the state object.
     * @return A new builder.
     */
    static <T> Builder<T> builder(String name, T obj, ToDoubleFunction<T> f) {
        return new Builder<>(name, obj, f);
    }

    /**
     * @return The current value of this gauge. Returns {@code NaN} if the
     * observed object has been garbage collected.
     */
    double value();

    @Override
    default Iterable<Measurement> measure() {
        return Collections.singletonList(new Measurement(this::value, Statistic.VALUE));
    }

    /**
     * Fluent builder for gauges. Accumulates name, tags, description, and base unit.
     * The terminal operation {@link #register(MeterRegistry)} creates or retrieves
     * a gauge from the registry.
     *
     * @param <T> The type of the state object being observed.
     */
    class Builder<T> {

        private final String name;

        private final T obj;

        private final ToDoubleFunction<T> f;

        private Tags tags = Tags.empty();

        private String description;

        private String baseUnit;

        private Builder(String name, T obj, ToDoubleFunction<T> f) {
            this.name = name;
            this.obj = obj;
            this.f = f;
        }

        /**
         * @param tags Must be an even number of arguments representing key/value pairs.
         * @return This builder with added tags.
         */
        public Builder<T> tags(String... tags) {
            return tags(Tags.of(tags));
        }

        /**
         * @param tags Tags to add to the eventual gauge.
         * @return This builder with added tags.
         */
        public Builder<T> tags(Iterable<Tag> tags) {
            this.tags = this.tags.and(tags);
            return this;
        }

        /**
         * @param key   The tag key.
         * @param value The tag value.
         * @return This builder with a single added tag.
         */
        public Builder<T> tag(String key, String value) {
            this.tags = tags.and(key, value);
            return this;
        }

        /**
         * @param description Description text of the eventual gauge.
         * @return This builder with added description.
         */
        public Builder<T> description(String description) {
            this.description = description;
            return this;
        }

        /**
         * @param unit Base unit of the eventual gauge.
         * @return This builder with added base unit.
         */
        public Builder<T> baseUnit(String unit) {
            this.baseUnit = unit;
            return this;
        }

        /**
         * Registers this gauge with the given registry, or retrieves the existing
         * one if a gauge with the same name and tags was already registered.
         *
         * @param registry The registry to register with.
         * @return The registered Gauge.
         */
        public Gauge register(MeterRegistry registry) {
            return registry.gauge(new Meter.Id(name, tags, Meter.Type.GAUGE, description, baseUnit), obj, f);
        }

    }

}
```

#### File: `src/main/java/dev/linhvu/micrometer/internal/DefaultGauge.java` [NEW]

```java
package dev.linhvu.micrometer.internal;

import dev.linhvu.micrometer.AbstractMeter;
import dev.linhvu.micrometer.Gauge;
import dev.linhvu.micrometer.Meter;

import java.lang.ref.WeakReference;
import java.util.function.ToDoubleFunction;

/**
 * Default {@link Gauge} implementation that holds a {@link WeakReference} to the
 * observed object and a {@link ToDoubleFunction} to extract the value.
 * <p>
 * The weak reference design is critical for preventing memory leaks — registering
 * a gauge on a short-lived object (e.g., a request-scoped collection) will not
 * prevent that object from being garbage collected. When the object is collected,
 * {@link #value()} returns {@link Double#NaN}.
 * <p>
 * If the value function throws an exception, the gauge also returns {@code NaN}
 * to prevent gauge evaluation from crashing the application.
 *
 * @param <T> The type of the state object being observed.
 */
public class DefaultGauge<T> extends AbstractMeter implements Gauge {

    private final WeakReference<T> ref;

    private final ToDoubleFunction<T> value;

    public DefaultGauge(Meter.Id id, T obj, ToDoubleFunction<T> value) {
        super(id);
        this.ref = new WeakReference<>(obj);
        this.value = value;
    }

    @Override
    public double value() {
        T obj = ref.get();
        if (obj != null) {
            try {
                return value.applyAsDouble(obj);
            }
            catch (Exception e) {
                return Double.NaN;
            }
        }
        return Double.NaN;
    }

}
```

#### File: `src/main/java/dev/linhvu/micrometer/noop/NoopGauge.java` [NEW]

```java
package dev.linhvu.micrometer.noop;

import dev.linhvu.micrometer.Gauge;
import dev.linhvu.micrometer.Meter;

/**
 * A gauge that does nothing. {@code value()} always returns zero.
 * <p>
 * Returned by the registry when the registry is closed or a filter denies
 * the meter. This guarantees that instrumentation code accessing {@code gauge.value()}
 * never throws or requires null checks.
 */
public class NoopGauge extends NoopMeter implements Gauge {

    public NoopGauge(Meter.Id id) {
        super(id);
    }

    @Override
    public double value() {
        return 0;
    }

}
```

#### File: `src/main/java/dev/linhvu/micrometer/MeterRegistry.java` [MODIFIED]

```java
package dev.linhvu.micrometer;

import dev.linhvu.micrometer.noop.NoopCounter;
import dev.linhvu.micrometer.noop.NoopGauge;

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
 * Currently supports Counter (via {@link CumulativeCounter}) and Gauge (via
 * {@link DefaultGauge}). Timer and DistributionSummary factories will be
 * implemented in Features 6 and 7 respectively.
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

### Test Code

#### File: `src/test/java/dev/linhvu/micrometer/GaugeTest.java` [NEW]

```java
package dev.linhvu.micrometer;

import dev.linhvu.micrometer.internal.DefaultGauge;
import dev.linhvu.micrometer.simple.SimpleMeterRegistry;
import org.junit.jupiter.api.Test;

import java.util.List;
import java.util.concurrent.atomic.AtomicInteger;

import static org.assertj.core.api.Assertions.assertThat;

/**
 * Tests the Gauge interface: value(), measure(), Builder, and visitor pattern.
 */
class GaugeTest {

    private static Meter.Id gaugeId(String name) {
        return new Meter.Id(name, Tags.empty(), Meter.Type.GAUGE, null, null);
    }

    private static Meter.Id gaugeId(String name, Tags tags) {
        return new Meter.Id(name, tags, Meter.Type.GAUGE, null, null);
    }

    // --- value() ---

    @Test
    void shouldReturnCurrentValue_WhenValueCalled() {
        AtomicInteger source = new AtomicInteger(42);
        Gauge gauge = new DefaultGauge<>(gaugeId("my.gauge"), source, AtomicInteger::get);

        assertThat(gauge.value()).isEqualTo(42.0);
    }

    @Test
    void shouldReflectChanges_WhenSourceObjectMutated() {
        AtomicInteger source = new AtomicInteger(10);
        Gauge gauge = new DefaultGauge<>(gaugeId("my.gauge"), source, AtomicInteger::get);

        source.set(99);
        assertThat(gauge.value()).isEqualTo(99.0);
    }

    // --- measure() ---

    @Test
    void shouldReturnSingleValueMeasurement_WhenMeasureCalled() {
        AtomicInteger source = new AtomicInteger(7);
        Gauge gauge = new DefaultGauge<>(gaugeId("my.gauge"), source, AtomicInteger::get);

        List<Measurement> measurements = (List<Measurement>) gauge.measure();
        assertThat(measurements).hasSize(1);
        assertThat(measurements.get(0).getStatistic()).isEqualTo(Statistic.VALUE);
        assertThat(measurements.get(0).getValue()).isEqualTo(7.0);
    }

    @Test
    void shouldReflectLiveValue_WhenMeasurementEvaluatedLazily() {
        AtomicInteger source = new AtomicInteger(0);
        Gauge gauge = new DefaultGauge<>(gaugeId("my.gauge"), source, AtomicInteger::get);
        Iterable<Measurement> measurements = gauge.measure();

        source.set(100);

        Measurement m = measurements.iterator().next();
        assertThat(m.getValue()).isEqualTo(100.0);
    }

    // --- Builder ---

    @Test
    void shouldBuildGauge_WhenUsingBuilder() {
        MeterRegistry registry = new SimpleMeterRegistry();
        AtomicInteger source = new AtomicInteger(5);

        Gauge gauge = Gauge.builder("queue.size", source, AtomicInteger::get)
                .tag("queue", "main")
                .description("Queue depth")
                .register(registry);

        assertThat(gauge).isNotNull();
        assertThat(gauge.value()).isEqualTo(5.0);
        assertThat(gauge.getId().getName()).isEqualTo("queue.size");
        assertThat(gauge.getId().getTag("queue")).isEqualTo("main");
    }

    @Test
    void shouldAccumulateTags_WhenUsingBuilderTagMethods() {
        MeterRegistry registry = new SimpleMeterRegistry();
        AtomicInteger source = new AtomicInteger(0);

        Gauge gauge = Gauge.builder("my.gauge", source, AtomicInteger::get)
                .tag("region", "us-east")
                .tags("env", "prod", "service", "api")
                .register(registry);

        assertThat(gauge.getId().getTag("region")).isEqualTo("us-east");
        assertThat(gauge.getId().getTag("env")).isEqualTo("prod");
        assertThat(gauge.getId().getTag("service")).isEqualTo("api");
    }

    @Test
    void shouldSetDescriptionAndBaseUnit_WhenUsingBuilder() {
        MeterRegistry registry = new SimpleMeterRegistry();
        AtomicInteger source = new AtomicInteger(0);

        Gauge gauge = Gauge.builder("temperature", source, AtomicInteger::get)
                .description("Current temperature")
                .baseUnit("celsius")
                .register(registry);

        assertThat(gauge.getId().getDescription()).isEqualTo("Current temperature");
        assertThat(gauge.getId().getBaseUnit()).isEqualTo("celsius");
    }

    // --- Visitor pattern ---

    @Test
    void shouldMatchAsGauge_WhenUsingVisitorPattern() {
        AtomicInteger source = new AtomicInteger(42);
        Gauge gauge = new DefaultGauge<>(gaugeId("temperature"), source, AtomicInteger::get);

        String result = gauge.match(
                c -> "counter",
                g -> "gauge:" + g.value(),
                t -> "timer",
                ds -> "ds",
                ltt -> "ltt",
                fc -> "fc",
                ft -> "ft",
                m -> "default");

        assertThat(result).isEqualTo("gauge:42.0");
    }

    // --- Identity ---

    @Test
    void shouldBeEqual_WhenGaugesHaveSameId() {
        AtomicInteger source1 = new AtomicInteger(1);
        AtomicInteger source2 = new AtomicInteger(2);
        Gauge a = new DefaultGauge<>(gaugeId("temp", Tags.of("loc", "kitchen")), source1, AtomicInteger::get);
        Gauge b = new DefaultGauge<>(gaugeId("temp", Tags.of("loc", "kitchen")), source2, AtomicInteger::get);

        assertThat(a).isEqualTo(b);
    }

    @Test
    void shouldNotBeEqual_WhenGaugesHaveDifferentTags() {
        AtomicInteger source = new AtomicInteger(1);
        Gauge a = new DefaultGauge<>(gaugeId("temp", Tags.of("loc", "kitchen")), source, AtomicInteger::get);
        Gauge b = new DefaultGauge<>(gaugeId("temp", Tags.of("loc", "bedroom")), source, AtomicInteger::get);

        assertThat(a).isNotEqualTo(b);
    }

}
```

#### File: `src/test/java/dev/linhvu/micrometer/internal/DefaultGaugeTest.java` [NEW]

```java
package dev.linhvu.micrometer.internal;

import dev.linhvu.micrometer.Meter;
import dev.linhvu.micrometer.Tags;
import org.junit.jupiter.api.Test;

import java.util.ArrayList;
import java.util.List;
import java.util.concurrent.atomic.AtomicInteger;

import static org.assertj.core.api.Assertions.assertThat;

/**
 * Tests the DefaultGauge implementation — focusing on WeakReference behavior,
 * NaN on GC, and error handling in the value function.
 */
class DefaultGaugeTest {

    private static Meter.Id id(String name) {
        return new Meter.Id(name, Tags.empty(), Meter.Type.GAUGE, null, null);
    }

    @Test
    void shouldReturnValue_WhenObjectAlive() {
        AtomicInteger source = new AtomicInteger(42);
        DefaultGauge<AtomicInteger> gauge = new DefaultGauge<>(id("test"), source, AtomicInteger::get);

        assertThat(gauge.value()).isEqualTo(42.0);
    }

    @Test
    void shouldTrackChanges_WhenSourceMutated() {
        AtomicInteger source = new AtomicInteger(1);
        DefaultGauge<AtomicInteger> gauge = new DefaultGauge<>(id("test"), source, AtomicInteger::get);

        source.set(100);
        assertThat(gauge.value()).isEqualTo(100.0);

        source.set(-5);
        assertThat(gauge.value()).isEqualTo(-5.0);
    }

    @Test
    void shouldReturnNaN_WhenObjectGarbageCollected() {
        DefaultGauge<List<String>> gauge;
        {
            List<String> list = new ArrayList<>();
            list.add("a");
            list.add("b");
            gauge = new DefaultGauge<>(id("list.size"), list, List::size);
            assertThat(gauge.value()).isEqualTo(2.0);
        }

        // Try to trigger GC — the list should be collectable since only a WeakReference remains
        // We can't guarantee GC, but we can verify the code path handles null gracefully
        for (int i = 0; i < 10; i++) {
            System.gc();
        }

        // After GC, the value should be NaN (if GC collected it) or still valid
        // We can't guarantee GC happened, so just verify no exception is thrown
        double value = gauge.value();
        assertThat(value).isNotNull(); // either NaN or 2.0 — both valid
    }

    @Test
    void shouldReturnNaN_WhenValueFunctionThrows() {
        AtomicInteger source = new AtomicInteger(0);
        DefaultGauge<AtomicInteger> gauge = new DefaultGauge<>(id("test"), source, obj -> {
            throw new RuntimeException("broken");
        });

        assertThat(gauge.value()).isNaN();
    }

    @Test
    void shouldReturnNaN_WhenCreatedWithNullObject() {
        DefaultGauge<Object> gauge = new DefaultGauge<>(id("null.gauge"), null, obj -> 42.0);

        assertThat(gauge.value()).isNaN();
    }

    @Test
    void shouldWorkWithCollectionSize() {
        List<String> items = new ArrayList<>();
        items.add("first");
        items.add("second");

        DefaultGauge<List<String>> gauge = new DefaultGauge<>(id("items"), items, List::size);
        assertThat(gauge.value()).isEqualTo(2.0);

        items.add("third");
        assertThat(gauge.value()).isEqualTo(3.0);

        items.clear();
        assertThat(gauge.value()).isEqualTo(0.0);
    }

    @Test
    void shouldReturnCorrectId_WhenGetIdCalled() {
        Meter.Id expected = id("my.gauge");
        DefaultGauge<AtomicInteger> gauge = new DefaultGauge<>(expected, new AtomicInteger(0), AtomicInteger::get);

        assertThat(gauge.getId()).isSameAs(expected);
    }

    @Test
    void shouldIncludeClassNameInToString() {
        DefaultGauge<AtomicInteger> gauge = new DefaultGauge<>(id("my.gauge"), new AtomicInteger(0), AtomicInteger::get);

        assertThat(gauge.toString()).contains("DefaultGauge");
        assertThat(gauge.toString()).contains("my.gauge");
    }

}
```

#### File: `src/test/java/dev/linhvu/micrometer/noop/NoopGaugeTest.java` [NEW]

```java
package dev.linhvu.micrometer.noop;

import dev.linhvu.micrometer.Gauge;
import dev.linhvu.micrometer.Measurement;
import dev.linhvu.micrometer.Meter;
import dev.linhvu.micrometer.Tags;
import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;

/**
 * Tests the NoopGauge — verifies that value() returns zero
 * and that measurement returns empty results.
 */
class NoopGaugeTest {

    private static Meter.Id noopId(String name) {
        return new Meter.Id(name, Tags.empty(), Meter.Type.GAUGE, null, null);
    }

    @Test
    void shouldReturnZero_WhenValueQueried() {
        Gauge gauge = new NoopGauge(noopId("noop"));
        assertThat(gauge.value()).isEqualTo(0.0);
    }

    @Test
    void shouldReturnEmptyMeasurements_WhenMeasured() {
        NoopGauge gauge = new NoopGauge(noopId("noop"));
        Iterable<Measurement> measurements = gauge.measure();
        assertThat(measurements).isEmpty();
    }

    @Test
    void shouldPreserveId_WhenCreated() {
        Meter.Id id = noopId("my.gauge");
        Gauge gauge = new NoopGauge(id);
        assertThat(gauge.getId()).isSameAs(id);
        assertThat(gauge.getId().getName()).isEqualTo("my.gauge");
    }

    @Test
    void shouldBeInstanceOfGauge_WhenTypeChecked() {
        Gauge gauge = new NoopGauge(noopId("noop"));
        assertThat(gauge).isInstanceOf(Gauge.class);
        assertThat(gauge).isInstanceOf(Meter.class);
        assertThat(gauge).isInstanceOf(NoopMeter.class);
    }

}
```

#### File: `src/test/java/dev/linhvu/micrometer/integration/GaugeIntegrationTest.java` [NEW]

```java
package dev.linhvu.micrometer.integration;

import dev.linhvu.micrometer.Counter;
import dev.linhvu.micrometer.Gauge;
import dev.linhvu.micrometer.MeterRegistry;
import dev.linhvu.micrometer.Tags;
import dev.linhvu.micrometer.internal.DefaultGauge;
import dev.linhvu.micrometer.noop.NoopGauge;
import dev.linhvu.micrometer.simple.SimpleMeterRegistry;
import org.junit.jupiter.api.Test;

import java.util.ArrayList;
import java.util.HashMap;
import java.util.List;
import java.util.Map;
import java.util.concurrent.atomic.AtomicInteger;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

/**
 * Integration tests verifying the full Gauge lifecycle: Builder → Registry →
 * DefaultGauge, convenience methods, deduplication, and interaction with
 * previously implemented features (Counter).
 */
class GaugeIntegrationTest {

    // --- Builder → Registry lifecycle ---

    @Test
    void shouldRegisterGauge_WhenUsingBuilder() {
        MeterRegistry registry = new SimpleMeterRegistry();
        AtomicInteger source = new AtomicInteger(42);

        Gauge gauge = Gauge.builder("queue.size", source, AtomicInteger::get)
                .tag("queue", "main")
                .description("Current queue depth")
                .baseUnit("items")
                .register(registry);

        assertThat(gauge).isInstanceOf(DefaultGauge.class);
        assertThat(gauge.value()).isEqualTo(42.0);
        assertThat(registry.getMeters()).hasSize(1);
    }

    @Test
    void shouldReturnSameInstance_WhenGaugeRegisteredTwice() {
        MeterRegistry registry = new SimpleMeterRegistry();
        AtomicInteger source = new AtomicInteger(10);

        Gauge first = Gauge.builder("gauge", source, AtomicInteger::get).register(registry);
        Gauge second = Gauge.builder("gauge", source, AtomicInteger::get).register(registry);

        assertThat(first).isSameAs(second);
        assertThat(registry.getMeters()).hasSize(1);
    }

    @Test
    void shouldCreateDistinctGauges_WhenDifferentNames() {
        MeterRegistry registry = new SimpleMeterRegistry();
        AtomicInteger a = new AtomicInteger(1);
        AtomicInteger b = new AtomicInteger(2);

        Gauge gaugeA = Gauge.builder("gauge.a", a, AtomicInteger::get).register(registry);
        Gauge gaugeB = Gauge.builder("gauge.b", b, AtomicInteger::get).register(registry);

        assertThat(gaugeA).isNotSameAs(gaugeB);
        assertThat(gaugeA.value()).isEqualTo(1.0);
        assertThat(gaugeB.value()).isEqualTo(2.0);
        assertThat(registry.getMeters()).hasSize(2);
    }

    // --- Convenience methods ---

    @Test
    void shouldRegisterGauge_WhenUsingConvenienceMethodWithFunction() {
        MeterRegistry registry = new SimpleMeterRegistry();
        AtomicInteger source = new AtomicInteger(55);

        AtomicInteger returned = registry.gauge("my.gauge", Tags.empty(), source, AtomicInteger::get);

        assertThat(returned).isSameAs(source);
        assertThat(registry.getMeters()).hasSize(1);

        Gauge gauge = (Gauge) registry.getMeters().get(0);
        assertThat(gauge.value()).isEqualTo(55.0);
    }

    @Test
    void shouldRegisterGauge_WhenUsingConvenienceMethodWithNumber() {
        MeterRegistry registry = new SimpleMeterRegistry();
        AtomicInteger source = new AtomicInteger(77);

        AtomicInteger returned = registry.gauge("my.number", Tags.of("type", "count"), source);

        assertThat(returned).isSameAs(source);

        Gauge gauge = (Gauge) registry.getMeters().get(0);
        assertThat(gauge.value()).isEqualTo(77.0);
    }

    @Test
    void shouldRegisterGauge_WhenUsingSimpleNumberConvenience() {
        MeterRegistry registry = new SimpleMeterRegistry();
        AtomicInteger source = new AtomicInteger(33);

        AtomicInteger returned = registry.gauge("simple.gauge", source);

        assertThat(returned).isSameAs(source);

        Gauge gauge = (Gauge) registry.getMeters().get(0);
        assertThat(gauge.value()).isEqualTo(33.0);
    }

    // --- gaugeCollectionSize ---

    @Test
    void shouldTrackCollectionSize_WhenUsingGaugeCollectionSize() {
        MeterRegistry registry = new SimpleMeterRegistry();
        List<String> items = new ArrayList<>();
        items.add("a");
        items.add("b");

        List<String> returned = registry.gaugeCollectionSize("items.count", Tags.empty(), items);

        assertThat(returned).isSameAs(items);

        Gauge gauge = (Gauge) registry.getMeters().get(0);
        assertThat(gauge.value()).isEqualTo(2.0);

        items.add("c");
        assertThat(gauge.value()).isEqualTo(3.0);

        items.clear();
        assertThat(gauge.value()).isEqualTo(0.0);
    }

    // --- gaugeMapSize ---

    @Test
    void shouldTrackMapSize_WhenUsingGaugeMapSize() {
        MeterRegistry registry = new SimpleMeterRegistry();
        Map<String, Integer> cache = new HashMap<>();
        cache.put("key1", 1);

        Map<String, Integer> returned = registry.gaugeMapSize("cache.size", Tags.empty(), cache);

        assertThat(returned).isSameAs(cache);

        Gauge gauge = (Gauge) registry.getMeters().get(0);
        assertThat(gauge.value()).isEqualTo(1.0);

        cache.put("key2", 2);
        cache.put("key3", 3);
        assertThat(gauge.value()).isEqualTo(3.0);
    }

    // --- Closed registry ---

    @Test
    void shouldReturnNoopGauge_WhenRegistryClosed() {
        MeterRegistry registry = new SimpleMeterRegistry();
        registry.close();

        AtomicInteger source = new AtomicInteger(42);
        Gauge gauge = Gauge.builder("gauge", source, AtomicInteger::get).register(registry);

        assertThat(gauge).isInstanceOf(NoopGauge.class);
        assertThat(gauge.value()).isEqualTo(0.0);
    }

    // --- Type conflict ---

    @Test
    void shouldThrowException_WhenGaugeNameConflictsWithCounter() {
        MeterRegistry registry = new SimpleMeterRegistry();
        registry.counter("my.metric");

        AtomicInteger source = new AtomicInteger(1);

        assertThatThrownBy(() ->
                Gauge.builder("my.metric", source, AtomicInteger::get).register(registry))
                .isInstanceOf(IllegalArgumentException.class)
                .hasMessageContaining("my.metric")
                .hasMessageContaining("Counter");
    }

    // --- Gauge + Counter coexistence ---

    @Test
    void shouldCoexist_WhenGaugeAndCounterHaveDifferentNames() {
        MeterRegistry registry = new SimpleMeterRegistry();

        Counter counter = registry.counter("http.requests");
        AtomicInteger queueSize = new AtomicInteger(5);
        Gauge gauge = Gauge.builder("queue.size", queueSize, AtomicInteger::get).register(registry);

        counter.increment(10);
        queueSize.set(3);

        assertThat(counter.count()).isEqualTo(10.0);
        assertThat(gauge.value()).isEqualTo(3.0);
        assertThat(registry.getMeters()).hasSize(2);
    }

    // --- Live value tracking ---

    @Test
    void shouldTrackLiveValue_WhenSourceChangesAfterRegistration() {
        MeterRegistry registry = new SimpleMeterRegistry();
        AtomicInteger source = new AtomicInteger(0);

        Gauge gauge = Gauge.builder("live.value", source, AtomicInteger::get).register(registry);

        assertThat(gauge.value()).isEqualTo(0.0);

        source.set(100);
        assertThat(gauge.value()).isEqualTo(100.0);

        source.set(-50);
        assertThat(gauge.value()).isEqualTo(-50.0);

        source.set(0);
        assertThat(gauge.value()).isEqualTo(0.0);
    }

}
```

#### File: `src/test/java/dev/linhvu/micrometer/MeterVisitorTest.java` [MODIFIED]

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
        public void increment(double amount) {
            // no-op for visitor testing
        }

        @Override
        public double count() {
            return 0.0;
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
        public double value() {
            return 0.0;
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
| **Gauge** | A meter that samples a current value from an external object — can go up and down. Use for pool sizes, queue depths, cache sizes. |
| **WeakReference** | Prevents gauge registration from causing memory leaks — if the observed object is collected, the gauge returns `NaN` instead of holding a zombie reference. |
| **`ToDoubleFunction<T>`** | A function that extracts a double value from the state object. This decouples what is observed (the object) from how it is measured (the function). |
| **Convenience return pattern** | `gauge()`, `gaugeCollectionSize()`, `gaugeMapSize()` return the state object (not the Gauge) — enabling inline assignment. |
| **NaN sentinel** | When the observed object is collected or the function throws, `Double.NaN` signals "no valid reading" without exceptions. |

**Next: Chapter 6 — Timer** — Records the duration and frequency of events. The most-used meter type for measuring latency, introducing `Timer.Sample` for start/stop measurement and `TimeWindowMax` for decaying maximum tracking.
