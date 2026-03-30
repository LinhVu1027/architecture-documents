# Chapter 1: Counter & MeterRegistry — Counting Things

> **Feature 1** · Tier 1 · Foundation · No dependencies

After this chapter you will have a working metrics library where clients can create
named, dimensionally-tagged counters, increment them, and read back the count — all
through the same API surface that real Micrometer exposes.

---

## 1. The API Contract

Before writing any implementation, let's define exactly what a client sees.

### What the client imports

```java
import simple.micrometer.api.Counter;
import simple.micrometer.api.MeterRegistry;
import simple.micrometer.internal.SimpleMeterRegistry;
```

### What the client writes

```java
// 1. Create a registry (the "home" for all meters)
MeterRegistry registry = new SimpleMeterRegistry();

// 2. Build and register a counter
Counter counter = Counter.builder("http.requests.total")
    .tag("method", "GET")
    .tag("status", "200")
    .description("Total HTTP requests")
    .baseUnit("requests")
    .register(registry);

// 3. Use it
counter.increment();
counter.increment(5.0);
System.out.println(counter.count()); // 6.0

// 4. Shorthand — skip the builder when you just need name + tags
Counter shorthand = registry.counter("cache.misses", "cache", "users");
```

### Behavioral contract

| Promise | Detail |
|---------|--------|
| Counters are monotonically increasing | `count()` never decreases (unless the JVM restarts) |
| Thread-safe | Multiple threads can call `increment()` concurrently |
| Deduplicated | Registering the same name + tags twice returns the *same* counter instance |
| Tag-sorted | Tags are always stored in key-sorted order for consistent identity |
| Live measurements | `measure()` returns a supplier-backed value that reflects current state |

---

## 2. Client Usage & Tests

These tests define the API's behavior — they were written **before** any implementation.

### Builder + Registration

```java
@Test
void shouldCreateCounter_WithBuilderAndRegister() {
    Counter counter = Counter.builder("http.requests.total")
        .tag("method", "GET")
        .tag("status", "200")
        .description("Total HTTP requests")
        .baseUnit("requests")
        .register(registry);

    assertThat(counter).isNotNull();
    assertThat(counter.getId().getName()).isEqualTo("http.requests.total");
}
```

### Increment & Count

```java
@Test
void shouldAccumulateMultipleIncrements() {
    Counter counter = Counter.builder("events").register(registry);

    counter.increment();       // +1
    counter.increment(5.0);    // +5
    counter.increment(2.5);    // +2.5

    assertThat(counter.count()).isEqualTo(8.5);
}
```

### Deduplication

```java
@Test
void shouldReturnSameCounter_WhenRegisteredTwiceWithSameId() {
    Counter counter1 = Counter.builder("requests")
        .tag("method", "GET")
        .register(registry);

    Counter counter2 = Counter.builder("requests")
        .tag("method", "GET")
        .register(registry);

    assertThat(counter1).isSameAs(counter2); // same object!
}
```

### Live Measurements

```java
@Test
void shouldReflectLiveCount_InMeasurement() {
    Counter counter = Counter.builder("events").register(registry);
    Iterable<Measurement> measurements = counter.measure();

    counter.increment(3.0);
    assertThat(measurements.iterator().next().getValue()).isEqualTo(3.0);

    counter.increment(2.0);
    assertThat(measurements.iterator().next().getValue()).isEqualTo(5.0);
}
```

---

## 3. Implementing the Call Chain — Top Down

### The call chain

```
Counter.builder("http.requests").tag("method","GET").register(registry)
  │
  ├─► [API Layer] Counter.Builder
  │     Stores name, tags, description, baseUnit
  │     Terminal op: register(registry) → creates Meter.Id, calls registry.counter(id)
  │
  ├─► [API Layer] MeterRegistry.counter(Meter.Id)
  │     Calls registerMeterIfNecessary() with double-checked locking
  │     Fast path: ConcurrentHashMap.get() — lock-free
  │     Slow path: synchronized block → call newCounter(id) → store in map
  │
  └─► [Implementation Layer] SimpleMeterRegistry.newCounter(id)
        Creates a CumulativeCounter backed by a DoubleAdder
```

### Layer 1: Foundation types — Tag, Tags, Statistic, Measurement, Clock

These are the atoms everything else is built from:

**Tag** — a single key-value dimension (`method=GET`). Implemented as an interface with a
`Tag.of()` factory method that returns an immutable record. Tags compare by key only,
which establishes the sorted order used by `Tags`.

**Tags** — an immutable, sorted, deduplicated collection of `Tag` instances. The merge
algorithm is a classic two-pointer sorted merge running in O(n+m) time. On key conflict,
the newer value wins.

**Statistic** — enum classifying what kind of value a measurement represents (`COUNT`,
`TOTAL`, `MAX`, `VALUE`, etc.).

**Measurement** — pairs a `DoubleSupplier` with a `Statistic`. The value is lazily
evaluated — each call to `getValue()` invokes the supplier, reflecting the meter's
current state. This is why measurements stay "live" without needing to be recreated.

**Clock** — abstracts `System.currentTimeMillis()` and `System.nanoTime()` for testability.
The `Clock.SYSTEM` constant is the production default.

### Layer 2: Meter and Meter.Id

**Meter** is the root interface for all meter types. It has two methods:
- `getId()` — returns the meter's identity
- `measure()` — returns current measurements

**Meter.Id** combines a name and a `Tags` collection to form the meter's unique identity.
It also carries non-identity metadata (description, base unit, type). Critically,
**equals/hashCode is based on name + tags ONLY** — two Ids with the same name and tags
but different descriptions are considered the same meter.

```java
// These two Ids are EQUAL — description is metadata, not identity
new Meter.Id("test", Tags.of("k","v"), null, "desc A", Type.COUNTER)
new Meter.Id("test", Tags.of("k","v"), null, "desc B", Type.COUNTER)
```

### Layer 3: Counter interface + Builder

**Counter** extends `Meter` with:
- `increment(double)` — the one method implementations must provide
- `increment()` — default method that calls `increment(1.0)`
- `count()` — returns the cumulative total
- `measure()` — default implementation wrapping `count()` in a `Measurement`

The **Builder** accumulates metadata but does NOT decide which Counter class to create.
That decision is deferred to the registry via `register(registry)`:

```java
public Counter register(MeterRegistry registry) {
    Meter.Id id = new Meter.Id(name, tags, baseUnit, description, Meter.Type.COUNTER);
    return registry.counter(id);
}
```

This separation is the key insight — the same builder produces different Counter
implementations depending on which registry you pass in.

### Layer 4: MeterRegistry — the registration engine

The abstract `MeterRegistry` provides the meter lifecycle:

```java
private <M extends Meter> M registerMeterIfNecessary(
        Class<M> meterClass, Meter.Id id, Function<Meter.Id, M> meterFactory) {

    // Fast path: lock-free ConcurrentHashMap lookup
    Meter existing = meterMap.get(id);
    if (existing != null) {
        return castOrThrow(meterClass, existing, id);
    }

    // Slow path: create under lock
    synchronized (meterMapLock) {
        existing = meterMap.get(id);  // double-check
        if (existing != null) {
            return castOrThrow(meterClass, existing, id);
        }
        M newMeter = meterFactory.apply(id);
        meterMap.put(id, newMeter);
        return newMeter;
    }
}
```

The double-checked locking pattern ensures:
1. **Hot path is fast** — reading an existing counter is just a `ConcurrentHashMap.get()`
2. **Creation is safe** — new meters are created inside `synchronized`, preventing races
3. **Deduplication is guaranteed** — same name + tags always returns the same instance

### Layer 5: SimpleMeterRegistry + CumulativeCounter

**SimpleMeterRegistry** is the simplest concrete registry — it always creates
`CumulativeCounter` instances:

```java
@Override
protected Counter newCounter(Meter.Id id) {
    return new CumulativeCounter(id);
}
```

**CumulativeCounter** stores its value in a `java.util.concurrent.atomic.DoubleAdder`:

```java
public class CumulativeCounter implements Counter {
    private final DoubleAdder value = new DoubleAdder();

    @Override
    public void increment(double amount) { value.add(amount); }

    @Override
    public double count() { return value.sum(); }
}
```

Why `DoubleAdder` over `AtomicLong`? Under high contention (many threads incrementing
the same counter), `DoubleAdder` maintains internal striped cells that reduce CAS
failures. It's the JDK's built-in solution for high-throughput counters.

---

## 4. Try It Yourself

1. **Add a `reset()` method to Counter** — What would break? (Hint: counters are
   supposed to be monotonically increasing. Monitoring systems rely on this.)

2. **Create a `StepCounter`** — instead of cumulative totals, it reports the count
   within the current time window. How would you use `Clock` for this?

3. **Add `registry.forEachCounter(Consumer<Counter>)`** — iterate over all registered
   counters. Should it filter by type? What about thread-safety during iteration?

---

## 5. Why This Works

### The Builder ↔ Registry separation

The Builder knows *what* to create (name, tags, metadata). The Registry knows *how* to
create it (CumulativeCounter vs StepCounter vs PrometheusCounter). This is the Template
Method pattern applied to object creation — the algorithm is fixed (build id → register →
store), but the factory step is abstract.

### Meter.Id equality

By making equality depend only on name + tags, the registry can deduplicate meters even
when different code paths register the same metric with slightly different descriptions.
This is intentional — in a large application, multiple libraries might register the same
counter without coordinating.

### Lazy Measurement suppliers

`Measurement` stores a `DoubleSupplier`, not a `double`. This means:
- Meters don't need to "push" updates to their measurements
- Monitoring backends can "pull" the current value at scrape time
- The `Measurement` object can be created once and reused forever

---

## 6. What We Enhanced

| Component | State after this feature | Will be enhanced by |
|-----------|------------------------|---------------------|
| `MeterRegistry` | Supports Counter only, no filters | F2: +Gauge, F3: +Timer, F5: +MeterFilter |
| `Meter.Id` | Basic identity | F5: filter mapping transforms Ids |
| `SimpleMeterRegistry` | Creates CumulativeCounter only | F2: +DefaultGauge, F3: +CumulativeTimer |
| `Tags` | Immutable sorted collection | Stable — no changes expected |

---

## 7. Connection to Real Framework

| Simplified Class | Real Framework Class | Key Simplification |
|-----------------|---------------------|--------------------|
| `api/Tag` | `io.micrometer.core.instrument.Tag` | Same API |
| `api/Tags` | `io.micrometer.core.instrument.Tags` | Simplified merge (no Spliterator) |
| `api/Statistic` | `io.micrometer.core.instrument.Statistic` | Same enum values |
| `api/Measurement` | `io.micrometer.core.instrument.Measurement` | Same lazy supplier pattern |
| `api/Clock` | `io.micrometer.core.instrument.Clock` | Same interface |
| `api/Meter` + `Meter.Id` | `io.micrometer.core.instrument.Meter` | No `match()`/`use()` visitor, no `MeterProvider`, no `NamingConvention` in Id |
| `api/Counter` | `io.micrometer.core.instrument.Counter` | No `MeterProvider.withRegistry()` |
| `api/MeterRegistry` | `io.micrometer.core.instrument.MeterRegistry` | No filters, no pre-filter cache, no listeners, no Config class |
| `internal/CumulativeCounter` | `...cumulative.CumulativeCounter` | No `AbstractMeter` base class |
| `internal/NoopCounter` | `...noop.NoopCounter` | No `NoopMeter` base class |
| `internal/SimpleMeterRegistry` | `...simple.SimpleMeterRegistry` | No `SimpleConfig`, no step mode |

---

## 8. Complete Code

All files created in this feature. Copy all `[NEW]` files to build the project from scratch.

### `src/main/java/simple/micrometer/api/Statistic.java` [NEW]

```java
package simple.micrometer.api;

public enum Statistic {

    TOTAL("total"),
    TOTAL_TIME("total"),
    COUNT("count"),
    MAX("max"),
    VALUE("value"),
    UNKNOWN("unknown"),
    ACTIVE_TASKS("active"),
    DURATION("duration");

    private final String tagValueRepresentation;

    Statistic(String tagValueRepresentation) {
        this.tagValueRepresentation = tagValueRepresentation;
    }

    public String getTagValueRepresentation() {
        return tagValueRepresentation;
    }

}
```

### `src/main/java/simple/micrometer/api/Clock.java` [NEW]

```java
package simple.micrometer.api;

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

    long wallTime();

    long monotonicTime();

}
```

### `src/main/java/simple/micrometer/api/Measurement.java` [NEW]

```java
package simple.micrometer.api;

import java.util.function.DoubleSupplier;

public class Measurement {

    private final DoubleSupplier valueFunction;
    private final Statistic statistic;

    public Measurement(DoubleSupplier valueFunction, Statistic statistic) {
        this.valueFunction = valueFunction;
        this.statistic = statistic;
    }

    public double getValue() {
        return valueFunction.getAsDouble();
    }

    public Statistic getStatistic() {
        return statistic;
    }

    @Override
    public String toString() {
        return "Measurement{" + statistic + "=" + getValue() + '}';
    }

}
```

### `src/main/java/simple/micrometer/api/Tag.java` [NEW]

```java
package simple.micrometer.api;

public interface Tag extends Comparable<Tag> {

    String getKey();

    String getValue();

    static Tag of(String key, String value) {
        return new ImmutableTag(key, value);
    }

    @Override
    default int compareTo(Tag o) {
        return getKey().compareTo(o.getKey());
    }

}
```

### `src/main/java/simple/micrometer/api/ImmutableTag.java` [NEW]

```java
package simple.micrometer.api;

import java.util.Objects;

record ImmutableTag(String key, String value) implements Tag {

    ImmutableTag {
        Objects.requireNonNull(key, "key must not be null");
        Objects.requireNonNull(value, "value must not be null");
    }

    @Override
    public String getKey() { return key; }

    @Override
    public String getValue() { return value; }

    @Override
    public String toString() { return "tag(" + key + "=" + value + ")"; }

}
```

### `src/main/java/simple/micrometer/api/Tags.java` [NEW]

```java
package simple.micrometer.api;

import java.util.Arrays;
import java.util.Iterator;
import java.util.NoSuchElementException;
import java.util.Objects;

public final class Tags implements Iterable<Tag> {

    private static final Tags EMPTY = new Tags(new Tag[0], 0);

    private final Tag[] sortedSet;
    private final int length;

    private Tags(Tag[] sortedSet, int length) {
        this.sortedSet = sortedSet;
        this.length = length;
    }

    public static Tags empty() { return EMPTY; }

    public static Tags of(String key, String value) {
        return new Tags(new Tag[] { Tag.of(key, value) }, 1);
    }

    public static Tags of(String... keyValues) {
        if (keyValues.length == 0) return EMPTY;
        if (keyValues.length % 2 != 0)
            throw new IllegalArgumentException("keyValues must have even length, got " + keyValues.length);
        Tag[] tags = new Tag[keyValues.length / 2];
        for (int i = 0; i < keyValues.length; i += 2)
            tags[i / 2] = Tag.of(keyValues[i], keyValues[i + 1]);
        return fromUnsorted(tags);
    }

    public static Tags of(Tag... tags) {
        if (tags.length == 0) return EMPTY;
        return fromUnsorted(tags.clone());
    }

    public static Tags of(Iterable<? extends Tag> tags) {
        if (tags instanceof Tags t) return t;
        Tag[] array = toArray(tags);
        if (array.length == 0) return EMPTY;
        return fromUnsorted(array);
    }

    public Tags and(String key, String value) {
        return and(new Tags(new Tag[] { Tag.of(key, value) }, 1));
    }

    public Tags and(String... keyValues) { return and(Tags.of(keyValues)); }
    public Tags and(Tag... tags) { return and(Tags.of(tags)); }

    public Tags and(Iterable<? extends Tag> tags) {
        if (tags instanceof Tags other) return merge(other);
        return merge(Tags.of(tags));
    }

    @Override
    public Iterator<Tag> iterator() { return new ArrayIterator(); }

    int size() { return length; }

    // ... equals, hashCode, toString, merge algorithm (see full source)

    private Tags merge(Tags other) {
        if (this.length == 0) return other;
        if (other.length == 0) return this;
        Tag[] merged = new Tag[this.length + other.length];
        int i = 0, j = 0, k = 0;
        while (i < this.length && j < other.length) {
            int cmp = this.sortedSet[i].compareTo(other.sortedSet[j]);
            if (cmp < 0) merged[k++] = this.sortedSet[i++];
            else if (cmp > 0) merged[k++] = other.sortedSet[j++];
            else { merged[k++] = other.sortedSet[j++]; i++; }
        }
        while (i < this.length) merged[k++] = this.sortedSet[i++];
        while (j < other.length) merged[k++] = other.sortedSet[j++];
        return new Tags(merged, k);
    }

    private static Tags fromUnsorted(Tag[] tags) {
        Arrays.sort(tags);
        int write = 0;
        for (int read = 0; read < tags.length; read++) {
            if (read + 1 < tags.length && tags[read].getKey().equals(tags[read + 1].getKey()))
                continue;
            tags[write++] = tags[read];
        }
        return new Tags(tags, write);
    }

    private static Tag[] toArray(Iterable<? extends Tag> tags) {
        Tag[] tmp = new Tag[8];
        int count = 0;
        for (Tag tag : tags) {
            if (count == tmp.length) tmp = Arrays.copyOf(tmp, tmp.length * 2);
            tmp[count++] = Objects.requireNonNull(tag);
        }
        return Arrays.copyOf(tmp, count);
    }

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof Tags other)) return false;
        if (this.length != other.length) return false;
        for (int i = 0; i < length; i++)
            if (!sortedSet[i].equals(other.sortedSet[i])) return false;
        return true;
    }

    @Override
    public int hashCode() {
        int result = 1;
        for (int i = 0; i < length; i++) result = 31 * result + sortedSet[i].hashCode();
        return result;
    }

    @Override
    public String toString() {
        StringBuilder sb = new StringBuilder("[");
        for (int i = 0; i < length; i++) {
            if (i > 0) sb.append(", ");
            sb.append(sortedSet[i]);
        }
        return sb.append("]").toString();
    }

    private class ArrayIterator implements Iterator<Tag> {
        private int index = 0;
        @Override public boolean hasNext() { return index < length; }
        @Override public Tag next() {
            if (!hasNext()) throw new NoSuchElementException();
            return sortedSet[index++];
        }
    }

}
```

### `src/main/java/simple/micrometer/api/Meter.java` [NEW]

```java
package simple.micrometer.api;

import java.util.ArrayList;
import java.util.Collections;
import java.util.List;
import java.util.Objects;

public interface Meter {

    Id getId();
    Iterable<Measurement> measure();

    enum Type {
        COUNTER, GAUGE, TIMER, DISTRIBUTION_SUMMARY, LONG_TASK_TIMER, OTHER
    }

    class Id {
        private final String name;
        private final Tags tags;
        private final String baseUnit;
        private final String description;
        private final Type type;

        public Id(String name, Tags tags, String baseUnit, String description, Type type) {
            this.name = Objects.requireNonNull(name, "name must not be null");
            this.tags = tags != null ? tags : Tags.empty();
            this.baseUnit = baseUnit;
            this.description = description;
            this.type = type;
        }

        public String getName() { return name; }
        public List<Tag> getTags() { return Collections.unmodifiableList(tagsAsList()); }
        public Iterable<Tag> getTagsAsIterable() { return tags; }

        public String getTag(String key) {
            for (Tag tag : tags)
                if (tag.getKey().equals(key)) return tag.getValue();
            return null;
        }

        public String getBaseUnit() { return baseUnit; }
        public String getDescription() { return description; }
        public Type getType() { return type; }

        public Id withName(String newName) {
            return new Id(newName, tags, baseUnit, description, type);
        }

        public Id withTag(Tag tag) {
            return new Id(name, tags.and(tag), baseUnit, description, type);
        }

        public Id withTags(Iterable<Tag> extraTags) {
            return new Id(name, tags.and(extraTags), baseUnit, description, type);
        }

        @Override
        public boolean equals(Object o) {
            if (this == o) return true;
            if (!(o instanceof Id other)) return false;
            return name.equals(other.name) && tags.equals(other.tags);
        }

        @Override
        public int hashCode() { return Objects.hash(name, tags); }

        @Override
        public String toString() {
            return "Meter.Id{name='" + name + "', tags=" + tags + '}';
        }

        private List<Tag> tagsAsList() {
            ArrayList<Tag> list = new ArrayList<>();
            for (Tag tag : tags) list.add(tag);
            return list;
        }
    }

}
```

### `src/main/java/simple/micrometer/api/Counter.java` [NEW]

```java
package simple.micrometer.api;

import java.util.Collections;

public interface Counter extends Meter {

    void increment(double amount);

    default void increment() { increment(1.0); }

    double count();

    @Override
    default Iterable<Measurement> measure() {
        return Collections.singletonList(new Measurement(this::count, Statistic.COUNT));
    }

    static Builder builder(String name) { return new Builder(name); }

    class Builder {
        private final String name;
        private Tags tags = Tags.empty();
        private String description;
        private String baseUnit;

        private Builder(String name) { this.name = name; }

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

        public Builder baseUnit(String unit) {
            this.baseUnit = unit;
            return this;
        }

        public Counter register(MeterRegistry registry) {
            Meter.Id id = new Meter.Id(name, tags, baseUnit, description, Meter.Type.COUNTER);
            return registry.counter(id);
        }
    }

}
```

### `src/main/java/simple/micrometer/api/MeterRegistry.java` [NEW]

```java
package simple.micrometer.api;

import simple.micrometer.internal.NoopCounter;

import java.util.ArrayList;
import java.util.Collections;
import java.util.List;
import java.util.Objects;
import java.util.concurrent.ConcurrentHashMap;
import java.util.function.Function;

public abstract class MeterRegistry {

    private final ConcurrentHashMap<Meter.Id, Meter> meterMap = new ConcurrentHashMap<>();
    private final Object meterMapLock = new Object();
    protected final Clock clock;

    protected MeterRegistry(Clock clock) {
        this.clock = Objects.requireNonNull(clock, "clock must not be null");
    }

    public Counter counter(String name, String... tags) {
        return counter(name, Tags.of(tags));
    }

    public Counter counter(String name, Iterable<Tag> tags) {
        Meter.Id id = new Meter.Id(name, Tags.of(tags), null, null, Meter.Type.COUNTER);
        return counter(id);
    }

    Counter counter(Meter.Id id) {
        return registerMeterIfNecessary(Counter.class, id, this::newCounter);
    }

    protected abstract Counter newCounter(Meter.Id id);

    public List<Meter> getMeters() {
        return Collections.unmodifiableList(new ArrayList<>(meterMap.values()));
    }

    public Clock getClock() { return clock; }

    private <M extends Meter> M registerMeterIfNecessary(
            Class<M> meterClass, Meter.Id id, Function<Meter.Id, M> meterFactory) {

        Meter existing = meterMap.get(id);
        if (existing != null) return castOrThrow(meterClass, existing, id);

        synchronized (meterMapLock) {
            existing = meterMap.get(id);
            if (existing != null) return castOrThrow(meterClass, existing, id);
            M newMeter = meterFactory.apply(id);
            meterMap.put(id, newMeter);
            return newMeter;
        }
    }

    private <M extends Meter> M castOrThrow(Class<M> meterClass, Meter existing, Meter.Id id) {
        if (!meterClass.isInstance(existing))
            throw new IllegalArgumentException(
                "Meter '" + id.getName() + "' exists as " + existing.getClass().getSimpleName()
                + " but expected " + meterClass.getSimpleName());
        return meterClass.cast(existing);
    }

}
```

### `src/main/java/simple/micrometer/internal/CumulativeCounter.java` [NEW]

```java
package simple.micrometer.internal;

import simple.micrometer.api.Counter;
import simple.micrometer.api.Meter;
import java.util.concurrent.atomic.DoubleAdder;

public class CumulativeCounter implements Counter {

    private final Meter.Id id;
    private final DoubleAdder value = new DoubleAdder();

    public CumulativeCounter(Meter.Id id) { this.id = id; }

    @Override public void increment(double amount) { value.add(amount); }
    @Override public double count() { return value.sum(); }
    @Override public Meter.Id getId() { return id; }

}
```

### `src/main/java/simple/micrometer/internal/NoopCounter.java` [NEW]

```java
package simple.micrometer.internal;

import simple.micrometer.api.Counter;
import simple.micrometer.api.Meter;

public class NoopCounter implements Counter {

    private final Meter.Id id;

    public NoopCounter(Meter.Id id) { this.id = id; }

    @Override public void increment(double amount) { /* no-op */ }
    @Override public double count() { return 0; }
    @Override public Meter.Id getId() { return id; }

}
```

### `src/main/java/simple/micrometer/internal/SimpleMeterRegistry.java` [NEW]

```java
package simple.micrometer.internal;

import simple.micrometer.api.Clock;
import simple.micrometer.api.Counter;
import simple.micrometer.api.Meter;
import simple.micrometer.api.MeterRegistry;

public class SimpleMeterRegistry extends MeterRegistry {

    public SimpleMeterRegistry() { super(Clock.SYSTEM); }
    public SimpleMeterRegistry(Clock clock) { super(clock); }

    @Override
    protected Counter newCounter(Meter.Id id) {
        return new CumulativeCounter(id);
    }

}
```

### `src/test/java/simple/micrometer/api/CounterTest.java` [NEW]

```java
package simple.micrometer.api;

import simple.micrometer.internal.SimpleMeterRegistry;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;

class CounterTest {

    private MeterRegistry registry;

    @BeforeEach
    void setUp() { registry = new SimpleMeterRegistry(); }

    @Test void shouldCreateCounter_WithBuilderAndRegister() { /* ... */ }
    @Test void shouldAttachTags_WhenBuiltWithMultipleTags() { /* ... */ }
    @Test void shouldAttachTags_WhenBuiltWithVarargsTags() { /* ... */ }
    @Test void shouldStartAtZero() { /* ... */ }
    @Test void shouldIncrementByOne() { /* ... */ }
    @Test void shouldIncrementByArbitraryAmount() { /* ... */ }
    @Test void shouldAccumulateMultipleIncrements() { /* ... */ }
    @Test void shouldCreateCounter_WithRegistryShorthand() { /* ... */ }
    @Test void shouldReturnSameCounter_WhenRegisteredTwiceWithSameId() { /* ... */ }
    @Test void shouldReturnDifferentCounters_WhenTagsDiffer() { /* ... */ }
    @Test void shouldProduceSingleCountMeasurement() { /* ... */ }
    @Test void shouldReflectLiveCount_InMeasurement() { /* ... */ }
}
```

*(Full test source: 12 tests covering builder, increment, dedup, measurements. See `src/test/` for complete code.)*

### `src/test/java/simple/micrometer/internal/SimpleMeterRegistryTest.java` [NEW]

```java
package simple.micrometer.internal;

import simple.micrometer.api.*;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;

class SimpleMeterRegistryTest {

    private SimpleMeterRegistry registry;

    @BeforeEach
    void setUp() { registry = new SimpleMeterRegistry(); }

    @Test void shouldStoreRegisteredMeters() { /* ... */ }
    @Test void shouldCreateCumulativeCounterImplementation() { /* ... */ }
    @Test void shouldDeduplicateByNameAndTags() { /* ... */ }
    @Test void shouldNotDeduplicateWhenTagsDiffer() { /* ... */ }
    @Test void shouldNotDeduplicateWhenNamesDiffer() { /* ... */ }
    @Test void shouldPreserveMeterIdProperties() { /* ... */ }
    @Test void shouldSortTagsByKey() { /* ... */ }
    @Test void shouldTreatIdAsEqual_WhenNameAndTagsMatch_EvenIfDescriptionDiffers() { /* ... */ }
    @Test void shouldNoopCounter_DiscardIncrements() { /* ... */ }
    @Test void shouldMergeTags_WithLastValueWinning() { /* ... */ }
    @Test void shouldCreateEmptyTags() { /* ... */ }
    @Test void shouldCreateTagsFromVarargs() { /* ... */ }
}
```

*(Full test source: 12 tests covering storage, dedup, Id equality, NoopCounter, Tags. See `src/test/` for complete code.)*
