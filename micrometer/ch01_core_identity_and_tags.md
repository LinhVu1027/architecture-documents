# Chapter 1: Core Identity & Tags

## Build Challenge

| Current State | Limitation | Objective |
|---|---|---|
| No metrics library exists | We have no way to name, label, or identify a metric | Build the core identity data structures — `Tag`, `Tags`, `Meter.Id` — that every future meter and registry will depend on |

---

## 1.1 Foundation Point: `Meter.Id`

Every feature we build from here on rests on one data structure: **`Meter.Id`**.

A `Meter.Id` is the lookup key that a `MeterRegistry` will use to store and retrieve meters. It consists of a **name** and a sorted set of **tags** (key-value dimensional labels). Two IDs with the same name and tags are considered equal — regardless of type, description, or base unit.

Here is how future features will plug into this foundation:

| Future Feature | How It Uses `Meter.Id` |
|---|---|
| **Ch02: Meter Interface & Clock** | Every `Meter` exposes a `getId()` that returns a `Meter.Id` |
| **Ch03: MeterRegistry** | The registry stores meters in a `ConcurrentHashMap<Meter.Id, Meter>` |
| **Ch04: Counter** | Created with a `Meter.Id` containing name + tags |
| **Ch05: Gauge** | Same identity system — different `Type` enum value |
| **Ch06: Timer & DistributionSummary** | `Id.withTag(Statistic)` creates per-measurement sub-IDs for export |
| **Ch07: MeterFilter** | Transforms and filters based on `Id` name and tags |

To build `Meter.Id`, we first need its building blocks: `Tag`, `ImmutableTag`, and `Tags`.

---

## 1.2 Tag & ImmutableTag

### Code

**`Tag.java`** — the interface:

```java
package dev.linhvu.micrometer;

/**
 * Key-value pair used as a dimensional label on a meter.
 * Tags enable filtering and grouping of metrics (e.g., "method"="GET", "status"="200").
 * <p>
 * Natural ordering is by {@link #getKey()} only — this is intentional so that
 * {@link Tags} can sort and deduplicate by key.
 */
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

**`ImmutableTag.java`** — the implementation:

```java
package dev.linhvu.micrometer;

import java.util.Objects;

/**
 * An immutable implementation of {@link Tag}.
 * <p>
 * Note: {@code equals}/{@code hashCode} use both key and value,
 * while {@code compareTo} (inherited from Tag) uses only the key.
 * This intentional inconsistency allows Tags to sort by key while
 * still distinguishing tags with different values.
 */
public class ImmutableTag implements Tag {

    private final String key;

    private final String value;

    public ImmutableTag(String key, String value) {
        Objects.requireNonNull(key, "key must not be null");
        Objects.requireNonNull(value, "value must not be null");
        this.key = key;
        this.value = value;
    }

    @Override
    public String getKey() {
        return key;
    }

    @Override
    public String getValue() {
        return value;
    }

    @Override
    public boolean equals(Object o) {
        if (this == o)
            return true;
        if (!(o instanceof ImmutableTag that))
            return false;
        return key.equals(that.getKey()) && value.equals(that.getValue());
    }

    @Override
    public int hashCode() {
        int result = key.hashCode();
        result = 31 * result + value.hashCode();
        return result;
    }

    @Override
    public String toString() {
        return "tag(" + key + "=" + value + ")";
    }

}
```

### Explanation

A `Tag` is a key-value pair that acts as a dimensional label. Think `"method"="GET"` or `"status"="200"`. Tags are the foundation of dimensional metrics — they let you slice and dice data along any axis.

The interface defines three things:

1. **`getKey()` / `getValue()`** — accessors for the two halves of the pair.
2. **`Tag.of(key, value)`** — a static factory that returns an `ImmutableTag`.
3. **`compareTo(Tag)`** — natural ordering by **key only**.

`ImmutableTag` is the concrete implementation. Its fields are `final`, its constructor rejects nulls, and it provides `equals`/`hashCode`/`toString`.

Notice the deliberate inconsistency: `compareTo` uses the key only, but `equals` uses both key and value. This is not a bug. It serves two distinct purposes:

- **`compareTo` by key only** lets the `Tags` collection sort tags alphabetically by key and detect duplicates (same key = `compareTo` returns 0).
- **`equals` by key+value** lets you distinguish `tag(env=prod)` from `tag(env=staging)` when checking identity — two tags with the same key but different values are *not* equal.

---

## 1.3 Tags — The Sorted Collection

### Code

```java
package dev.linhvu.micrometer;

import java.util.ArrayList;
import java.util.Arrays;
import java.util.Collection;
import java.util.Collections;
import java.util.Iterator;
import java.util.List;
import java.util.NoSuchElementException;
import java.util.Spliterator;
import java.util.Spliterators;
import java.util.stream.Collectors;
import java.util.stream.Stream;
import java.util.stream.StreamSupport;

/**
 * An immutable, sorted, deduplicated collection of {@link Tag}s.
 * <p>
 * Tags are sorted by key. When duplicate keys are present, the last value wins.
 * Merging two Tags collections ({@link #and}) gives precedence to the "other"
 * (newer) collection on key conflicts.
 * <p>
 * Backed by a sorted array for memory efficiency and O(n+m) merge performance.
 */
public final class Tags implements Iterable<Tag> {

    private static final Tag[] EMPTY_TAG_ARRAY = new Tag[0];

    private static final Tags EMPTY = new Tags(EMPTY_TAG_ARRAY, 0);

    private final Tag[] sortedSet;

    private final int length;

    private Tags(Tag[] sortedSet, int length) {
        this.sortedSet = sortedSet;
        this.length = length;
    }

    // -----------------------------------------------------------------------
    // Factory methods
    // -----------------------------------------------------------------------

    public static Tags empty() {
        return EMPTY;
    }

    public static Tags of(String key, String value) {
        return new Tags(new Tag[] { Tag.of(key, value) }, 1);
    }

    public static Tags of(Tag... tags) {
        if (tags == null || tags.length == 0) {
            return EMPTY;
        }
        return toTags(tags.clone());
    }

    public static Tags of(String... keyValues) {
        if (keyValues == null || keyValues.length == 0) {
            return EMPTY;
        }
        if (keyValues.length % 2 != 0) {
            throw new IllegalArgumentException("size must be even, it is a set of key=value pairs");
        }
        Tag[] tags = new Tag[keyValues.length / 2];
        for (int i = 0; i < keyValues.length; i += 2) {
            tags[i / 2] = Tag.of(keyValues[i], keyValues[i + 1]);
        }
        return toTags(tags);
    }

    @SuppressWarnings("unchecked")
    public static Tags of(Iterable<? extends Tag> tags) {
        if (tags instanceof Tags t) {
            return t;
        }
        if (tags instanceof Collection<? extends Tag> c) {
            if (c.isEmpty()) {
                return EMPTY;
            }
            return toTags(c.toArray(EMPTY_TAG_ARRAY));
        }
        return toTags(iterableToArray(tags));
    }

    // -----------------------------------------------------------------------
    // Merge methods
    // -----------------------------------------------------------------------

    public Tags and(String key, String value) {
        return and(Tag.of(key, value));
    }

    public Tags and(Tag... tags) {
        if (tags == null || tags.length == 0) {
            return this;
        }
        return merge(Tags.of(tags));
    }

    public Tags and(Iterable<? extends Tag> tags) {
        if (tags instanceof Tags other) {
            return merge(other);
        }
        return merge(Tags.of(tags));
    }

    public static Tags concat(Iterable<? extends Tag> tagsA, Iterable<? extends Tag> tagsB) {
        return Tags.of(tagsA).and(tagsB);
    }

    // -----------------------------------------------------------------------
    // Query methods
    // -----------------------------------------------------------------------

    public Stream<Tag> stream() {
        return StreamSupport.stream(Spliterators.spliterator(sortedSet, 0, length,
                Spliterator.IMMUTABLE | Spliterator.ORDERED | Spliterator.DISTINCT | Spliterator.NONNULL
                        | Spliterator.SORTED),
                false);
    }

    @Override
    public Iterator<Tag> iterator() {
        return new ArrayIterator(sortedSet, length);
    }

    // -----------------------------------------------------------------------
    // equals / hashCode / toString
    // -----------------------------------------------------------------------

    @Override
    public boolean equals(Object o) {
        if (this == o)
            return true;
        if (!(o instanceof Tags other))
            return false;
        if (this.length != other.length)
            return false;
        for (int i = 0; i < length; i++) {
            if (!sortedSet[i].equals(other.sortedSet[i]))
                return false;
        }
        return true;
    }

    @Override
    public int hashCode() {
        int result = 1;
        for (int i = 0; i < length; i++) {
            result = 31 * result + sortedSet[i].hashCode();
        }
        return result;
    }

    @Override
    public String toString() {
        return stream().map(Tag::toString).collect(Collectors.joining(",", "[", "]"));
    }

    // -----------------------------------------------------------------------
    // Internal algorithms
    // -----------------------------------------------------------------------

    private static Tags toTags(Tag[] tags) {
        int len = tags.length;
        if (len == 0) {
            return EMPTY;
        }
        if (len == 1) {
            return new Tags(tags, 1);
        }
        // Fast path: already a sorted set (no duplicates, strictly ascending by key)
        if (isSortedSet(tags, len)) {
            return new Tags(tags, len);
        }
        Arrays.sort(tags);
        return dedup(tags);
    }

    /**
     * Returns true if the first {@code length} elements are strictly ordered
     * by key — no duplicate keys allowed.
     */
    private static boolean isSortedSet(Tag[] tags, int length) {
        for (int i = 0; i < length - 1; i++) {
            if (tags[i].compareTo(tags[i + 1]) >= 0) {
                return false;
            }
        }
        return true;
    }

    /**
     * Removes duplicate keys from a sorted array. When the same key appears
     * multiple times, the LAST occurrence wins (it appears latest in sort order).
     */
    private static Tags dedup(Tag[] sortedTags) {
        int len = sortedTags.length;
        int writeIdx = 0;
        for (int readIdx = 0; readIdx < len; readIdx++) {
            // Skip forward to the last tag with the same key
            while (readIdx < len - 1 && sortedTags[readIdx].compareTo(sortedTags[readIdx + 1]) == 0) {
                readIdx++;
            }
            sortedTags[writeIdx++] = sortedTags[readIdx];
        }
        return new Tags(sortedTags, writeIdx);
    }

    /**
     * Classic sorted-merge of two sorted sets. On key conflict, the "other"
     * tag wins — this is why {@code tags.and(newTags)} gives precedence to newTags.
     */
    private Tags merge(Tags other) {
        if (other.length == 0)
            return this;
        if (this.length == 0)
            return other;

        Tag[] merged = new Tag[this.length + other.length];
        int thisIdx = 0, otherIdx = 0, writeIdx = 0;

        while (thisIdx < this.length && otherIdx < other.length) {
            Tag thisTag = this.sortedSet[thisIdx];
            Tag otherTag = other.sortedSet[otherIdx];
            int cmp = thisTag.compareTo(otherTag);
            if (cmp < 0) {
                merged[writeIdx++] = thisTag;
                thisIdx++;
            }
            else if (cmp > 0) {
                merged[writeIdx++] = otherTag;
                otherIdx++;
            }
            else {
                // Same key — other wins (newer overrides older)
                merged[writeIdx++] = otherTag;
                thisIdx++;
                otherIdx++;
            }
        }
        while (thisIdx < this.length) {
            merged[writeIdx++] = this.sortedSet[thisIdx++];
        }
        while (otherIdx < other.length) {
            merged[writeIdx++] = other.sortedSet[otherIdx++];
        }

        return new Tags(merged, writeIdx);
    }

    private static Tag[] iterableToArray(Iterable<? extends Tag> tags) {
        List<Tag> list = new ArrayList<>();
        tags.forEach(list::add);
        return list.toArray(EMPTY_TAG_ARRAY);
    }

    // -----------------------------------------------------------------------
    // Inner classes
    // -----------------------------------------------------------------------

    private static class ArrayIterator implements Iterator<Tag> {

        private final Tag[] tags;

        private final int length;

        private int index = 0;

        ArrayIterator(Tag[] tags, int length) {
            this.tags = tags;
            this.length = length;
        }

        @Override
        public boolean hasNext() {
            return index < length;
        }

        @Override
        public Tag next() {
            if (!hasNext()) {
                throw new NoSuchElementException();
            }
            return tags[index++];
        }

        @Override
        public void remove() {
            throw new UnsupportedOperationException("Tags are immutable");
        }

    }

}
```

### Explanation

`Tags` is where the real engineering happens. It is an immutable, sorted, deduplicated collection of `Tag` objects backed by a plain array.

**Why a sorted array instead of a `TreeMap` or `LinkedHashMap`?**

Micrometer creates and compares tag sets on every metric recording — potentially millions of times per second. A sorted array gives us:

- **Cache-friendly iteration** — contiguous memory, no pointer-chasing through `Map.Entry` nodes.
- **O(n+m) merge** — two sorted arrays can be merged in a single linear pass.
- **Minimal memory overhead** — no per-entry objects, no load-factor waste.

**Construction pipeline:** Every factory method feeds into `toTags(Tag[])`, which:
1. Checks for the fast path: if the array is already a sorted set (strictly ascending keys, no duplicates), wrap it directly.
2. Otherwise, `Arrays.sort(tags)` sorts by key (using `Tag.compareTo`), then `dedup()` collapses duplicate keys.

**Deduplication — last value wins:** When you write `Tags.of(Tag.of("env", "staging"), Tag.of("env", "prod"))`, both tags have key `"env"`. After sorting, they are adjacent. The `dedup()` method walks forward through consecutive same-key entries and keeps only the *last* one. Since Java's `Arrays.sort` is stable, the last occurrence in the original input is the one that survives. This gives predictable "last value wins" semantics.

**Merge — other wins:** The `merge()` method is a classic sorted-merge of two already-sorted arrays. When both sides have a tag with the same key, the "other" (right-hand) tag wins. This is why `base.and(overrides)` gives precedence to `overrides`.

**Immutability contract:** The constructor is `private`. Every public method returns a new `Tags` instance (or `this` / `other` when merging with an empty set — an optimization the tests verify). The `ArrayIterator` throws `UnsupportedOperationException` on `remove()`.

---

## 1.4 Meter.Id, Statistic & Measurement

### Code

**`Statistic.java`** — enum of measurement types:

```java
package dev.linhvu.micrometer;

/**
 * Enumerates the types of statistics a {@link Measurement} can represent.
 * <p>
 * Each statistic has a tag-value representation used when exporting individual
 * measurements as separate time series (e.g., a Timer emits COUNT, TOTAL_TIME, and MAX).
 */
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

    /**
     * Returns the string used as the value of a "statistic" tag when this
     * measurement is exported. For example, {@code COUNT} becomes {@code "count"}.
     */
    public String getTagValueRepresentation() {
        return tagValueRepresentation;
    }

}
```

**`Measurement.java`** — lazy value + statistic pair:

```java
package dev.linhvu.micrometer;

import java.util.function.DoubleSupplier;
import java.util.function.Supplier;

/**
 * A sampled value paired with the {@link Statistic} it represents.
 * <p>
 * Measurements use lazy evaluation — the value is computed fresh on every call
 * to {@link #getValue()} via a {@link DoubleSupplier}. This means measurements
 * are always live snapshots rather than stale stored values.
 */
public class Measurement {

    private final DoubleSupplier f;

    private final Statistic statistic;

    public Measurement(DoubleSupplier valueFunction, Statistic statistic) {
        this.f = valueFunction;
        this.statistic = statistic;
    }

    public Measurement(Supplier<Double> valueFunction, Statistic statistic) {
        this.f = valueFunction::get;
        this.statistic = statistic;
    }

    public double getValue() {
        return f.getAsDouble();
    }

    public Statistic getStatistic() {
        return statistic;
    }

    @Override
    public String toString() {
        return "Measurement{statistic='" + statistic + "', value=" + getValue() + '}';
    }

}
```

**`Meter.java`** — the interface with inner `Id` class and `Type` enum:

```java
package dev.linhvu.micrometer;

import java.util.ArrayList;
import java.util.Collections;
import java.util.List;
import java.util.Objects;

/**
 * A named and tagged set of measurements. This is the base type for all meters.
 * <p>
 * Feature 1 defines the identity system ({@link Id}, {@link Type}).
 * The full Meter interface ({@code getId}, {@code measure}, visitor methods)
 * will be added in Feature 2.
 */
public interface Meter {

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

### Explanation

**`Statistic`** enumerates the kinds of values a meter can produce. A Counter produces `COUNT`; a Timer produces `COUNT`, `TOTAL_TIME`, and `MAX`. Each constant carries a `tagValueRepresentation` — the string used when exporting individual measurements as tagged time series.

Note that `TOTAL` and `TOTAL_TIME` both map to `"total"`. This is intentional: `TOTAL` represents an amount (like bytes transferred) while `TOTAL_TIME` represents a duration. They share the same tag value because the meter type already disambiguates them. A DistributionSummary's "total" is an amount; a Timer's "total" is a time.

**`Measurement`** is a lazy pair: a `DoubleSupplier` and a `Statistic`. The key design decision is that values are *never stored* — they are computed on every call to `getValue()`. This means a `Measurement` is always a live snapshot of the current state of its meter. When an exporter asks "what is the count right now?", the supplier reads the meter's atomic counter at that instant.

The two constructors accept either a `DoubleSupplier` (primitive, no boxing) or a `Supplier<Double>` (for convenience). The `Supplier<Double>` variant converts via method reference `valueFunction::get`.

**`Meter.Type`** enumerates the kinds of meters: `COUNTER`, `GAUGE`, `TIMER`, `LONG_TASK_TIMER`, `DISTRIBUTION_SUMMARY`, and `OTHER`. This is metadata — it tells exporters how to interpret the data — but it is *not* part of identity.

**`Meter.Id`** is the heart of this chapter. Its constructor takes five parameters, but only two define identity:

```java
public Id(String name, Tags tags, Type type, String description, String baseUnit)
```

Look at `equals` and `hashCode`: they use only `name` and `tags`. The `type`, `description`, and `baseUnit` are metadata that travels with the ID but does not affect equality. This means:

```java
new Meter.Id("http.requests", Tags.of("method", "GET"), COUNTER, "Request count", null)
    .equals(
new Meter.Id("http.requests", Tags.of("method", "GET"), TIMER, "Different desc", "ms")
    )  // true!
```

Why? Because the registry maps `name+tags` to a single meter. If you accidentally create a Counter and a Timer with the same name and tags, the registry should detect the conflict — not silently store two separate entries.

All `with*` methods return new `Id` instances. This immutability is critical because IDs are used as HashMap keys — if an ID mutated after insertion, its hash bucket would be wrong and lookups would fail silently.

---

## 1.5 Try It Yourself

Before reading the tests, try these challenges:

<details>
<summary>Challenge 1: What happens when you sort tags with duplicate keys?</summary>

Create these tags and predict the result:

```java
Tags tags = Tags.of(
    Tag.of("env", "staging"),
    Tag.of("host", "server1"),
    Tag.of("env", "prod")
);
```

**Answer:** The result contains two tags: `[tag(env=prod), tag(host=server1)]`. The key `"env"` appears twice. After sorting, the two `"env"` tags are adjacent. `dedup()` walks forward to the last `"env"` entry (which is `"prod"` because the input order is preserved by stable sort) and keeps only that one.
</details>

<details>
<summary>Challenge 2: What does merge do when both sides have the same key?</summary>

```java
Tags base = Tags.of("env", "prod", "host", "server1");
Tags overrides = Tags.of("env", "staging", "region", "us-east");
Tags merged = base.and(overrides);
```

**Answer:** The result is `[tag(env=staging), tag(host=server1), tag(region=us-east)]`. The merge walks both sorted arrays in parallel. When both have key `"env"`, the "other" (overrides) wins, so `"staging"` replaces `"prod"`. Keys unique to either side are kept.
</details>

<details>
<summary>Challenge 3: Are these two Meter.Ids equal?</summary>

```java
Meter.Id a = new Meter.Id("http.requests", Tags.of("method", "GET"),
    Meter.Type.COUNTER, "Request count", null);
Meter.Id b = new Meter.Id("http.requests", Tags.of("method", "GET"),
    Meter.Type.TIMER, "Latency", "ms");
```

**Answer:** Yes, they are equal. `Meter.Id.equals` only checks `name` and `tags`. The type (`COUNTER` vs `TIMER`), description, and base unit are metadata — not identity. This is by design: the registry uses `name+tags` as its lookup key.
</details>

<details>
<summary>Challenge 4: Why does Measurement use a DoubleSupplier instead of storing a double?</summary>

```java
double[] counter = { 0.0 };
Measurement m = new Measurement(() -> counter[0], Statistic.COUNT);
counter[0] = 42.0;
System.out.println(m.getValue()); // What prints?
```

**Answer:** It prints `42.0`. The measurement does not store a value — it calls the supplier every time. This means it always reflects the current state of the underlying meter, not a stale snapshot from creation time.
</details>

---

## 1.6 Tests

Run the full test suite:

```bash
./gradlew test
```

All 45 tests should pass. Here is what each test file covers:

### TagTest (4 tests)

```java
package dev.linhvu.micrometer;

import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;

class TagTest {

    @Test
    void shouldCreateTag_WhenUsingStaticFactory() {
        Tag tag = Tag.of("env", "prod");

        assertThat(tag.getKey()).isEqualTo("env");
        assertThat(tag.getValue()).isEqualTo("prod");
    }

    @Test
    void shouldReturnZero_WhenComparingTagsWithSameKey() {
        Tag a = Tag.of("host", "server1");
        Tag b = Tag.of("host", "server2");

        // compareTo uses key only — same key means compareTo == 0
        assertThat(a.compareTo(b)).isEqualTo(0);
    }

    @Test
    void shouldOrderAlphabetically_WhenComparingByKey() {
        Tag a = Tag.of("alpha", "1");
        Tag b = Tag.of("beta", "2");

        assertThat(a.compareTo(b)).isLessThan(0);
        assertThat(b.compareTo(a)).isGreaterThan(0);
    }

    @Test
    void shouldReturnImmutableTag_WhenUsingStaticFactory() {
        Tag tag = Tag.of("key", "value");

        assertThat(tag).isInstanceOf(ImmutableTag.class);
    }

}
```

### ImmutableTagTest (6 tests)

```java
package dev.linhvu.micrometer;

import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatNullPointerException;

class ImmutableTagTest {

    @Test
    void shouldBeEqual_WhenKeyAndValueMatch() {
        ImmutableTag a = new ImmutableTag("env", "prod");
        ImmutableTag b = new ImmutableTag("env", "prod");

        assertThat(a).isEqualTo(b);
        assertThat(a.hashCode()).isEqualTo(b.hashCode());
    }

    @Test
    void shouldNotBeEqual_WhenValuesDiffer() {
        ImmutableTag a = new ImmutableTag("env", "prod");
        ImmutableTag b = new ImmutableTag("env", "staging");

        assertThat(a).isNotEqualTo(b);
    }

    @Test
    void shouldNotBeEqual_WhenKeysDiffer() {
        ImmutableTag a = new ImmutableTag("env", "prod");
        ImmutableTag b = new ImmutableTag("region", "prod");

        assertThat(a).isNotEqualTo(b);
    }

    @Test
    void shouldRejectNullKey() {
        assertThatNullPointerException().isThrownBy(() -> new ImmutableTag(null, "value"));
    }

    @Test
    void shouldRejectNullValue() {
        assertThatNullPointerException().isThrownBy(() -> new ImmutableTag("key", null));
    }

    @Test
    void shouldProduceReadableToString() {
        assertThat(new ImmutableTag("env", "prod").toString()).isEqualTo("tag(env=prod)");
    }

}
```

### TagsTest (17 tests)

```java
package dev.linhvu.micrometer;

import org.junit.jupiter.api.Test;

import java.util.List;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatIllegalArgumentException;

class TagsTest {

    @Test
    void shouldCreateEmptyTags() {
        Tags tags = Tags.empty();

        assertThat(tags).isEmpty();
    }

    @Test
    void shouldCreateFromSingleKeyValue() {
        Tags tags = Tags.of("env", "prod");

        assertThat(tags).containsExactly(Tag.of("env", "prod"));
    }

    @Test
    void shouldSortTagsByKey() {
        Tags tags = Tags.of(Tag.of("zone", "us-east"), Tag.of("app", "web"), Tag.of("env", "prod"));

        List<String> keys = tags.stream().map(Tag::getKey).toList();
        assertThat(keys).containsExactly("app", "env", "zone");
    }

    @Test
    void shouldDeduplicateByKey_LastValueWins() {
        Tags tags = Tags.of(Tag.of("env", "staging"), Tag.of("env", "prod"));

        // Both have key "env" — after sorting by key they're adjacent,
        // and the last occurrence ("prod") wins
        assertThat(tags).containsExactly(Tag.of("env", "prod"));
    }

    @Test
    void shouldCreateFromAlternatingKeyValueStrings() {
        Tags tags = Tags.of("a", "1", "b", "2");

        assertThat(tags).containsExactly(Tag.of("a", "1"), Tag.of("b", "2"));
    }

    @Test
    void shouldRejectOddNumberOfStrings() {
        assertThatIllegalArgumentException().isThrownBy(() -> Tags.of("a", "1", "orphan"));
    }

    @Test
    void shouldMergeTags_OtherWinsOnConflict() {
        Tags base = Tags.of("env", "prod", "host", "server1");
        Tags override = Tags.of("env", "staging", "region", "us-east");

        Tags merged = base.and(override);

        assertThat(merged).containsExactly(Tag.of("env", "staging"), // overridden by "other"
                Tag.of("host", "server1"), // kept from base
                Tag.of("region", "us-east") // added from other
        );
    }

    @Test
    void shouldSupportAndWithSingleKeyValue() {
        Tags tags = Tags.of("a", "1").and("b", "2");

        assertThat(tags).containsExactly(Tag.of("a", "1"), Tag.of("b", "2"));
    }

    @Test
    void shouldConcatTwoIterables() {
        Tags a = Tags.of("x", "1");
        Tags b = Tags.of("y", "2");

        Tags result = Tags.concat(a, b);

        assertThat(result).containsExactly(Tag.of("x", "1"), Tag.of("y", "2"));
    }

    @Test
    void shouldStreamTags() {
        Tags tags = Tags.of("a", "1", "b", "2");

        List<String> keys = tags.stream().map(Tag::getKey).toList();
        assertThat(keys).containsExactly("a", "b");
    }

    @Test
    void shouldBeEqual_WhenContainingSameTags() {
        Tags a = Tags.of("env", "prod", "host", "s1");
        Tags b = Tags.of("host", "s1", "env", "prod"); // different insertion order

        assertThat(a).isEqualTo(b);
        assertThat(a.hashCode()).isEqualTo(b.hashCode());
    }

    @Test
    void shouldNotBeEqual_WhenTagsDiffer() {
        Tags a = Tags.of("env", "prod");
        Tags b = Tags.of("env", "staging");

        assertThat(a).isNotEqualTo(b);
    }

    @Test
    void shouldReturnSameInstance_WhenMergingWithEmpty() {
        Tags tags = Tags.of("a", "1");

        assertThat(tags.and(Tags.empty())).isSameAs(tags);
    }

    @Test
    void shouldReturnOther_WhenThisIsEmpty() {
        Tags other = Tags.of("a", "1");

        assertThat(Tags.empty().and(other)).isSameAs(other);
    }

    @Test
    void shouldHandleAlreadySortedInput() {
        // Fast path: input is already sorted and has no duplicates
        Tags tags = Tags.of(Tag.of("a", "1"), Tag.of("b", "2"), Tag.of("c", "3"));

        assertThat(tags).containsExactly(Tag.of("a", "1"), Tag.of("b", "2"), Tag.of("c", "3"));
    }

    @Test
    void shouldCreateFromIterable() {
        List<Tag> tagList = List.of(Tag.of("b", "2"), Tag.of("a", "1"));

        Tags tags = Tags.of(tagList);

        assertThat(tags).containsExactly(Tag.of("a", "1"), Tag.of("b", "2"));
    }

    @Test
    void shouldReturnSameInstance_WhenCreatingFromTags() {
        Tags original = Tags.of("a", "1");

        assertThat(Tags.of(original)).isSameAs(original);
    }

}
```

### MeterIdTest (12 tests)

```java
package dev.linhvu.micrometer;

import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;

class MeterIdTest {

    @Test
    void shouldBeEqual_WhenNameAndTagsMatch() {
        Meter.Id a = new Meter.Id("http.requests", Tags.of("method", "GET"), Meter.Type.COUNTER, "Request count",
                null);
        Meter.Id b = new Meter.Id("http.requests", Tags.of("method", "GET"), Meter.Type.TIMER,
                "Different description", "ms");

        // Type, description, and baseUnit are NOT part of identity
        assertThat(a).isEqualTo(b);
        assertThat(a.hashCode()).isEqualTo(b.hashCode());
    }

    @Test
    void shouldNotBeEqual_WhenNamesDiffer() {
        Meter.Id a = new Meter.Id("http.requests", Tags.empty(), Meter.Type.COUNTER, null, null);
        Meter.Id b = new Meter.Id("http.responses", Tags.empty(), Meter.Type.COUNTER, null, null);

        assertThat(a).isNotEqualTo(b);
    }

    @Test
    void shouldNotBeEqual_WhenTagsDiffer() {
        Meter.Id a = new Meter.Id("http.requests", Tags.of("method", "GET"), Meter.Type.COUNTER, null, null);
        Meter.Id b = new Meter.Id("http.requests", Tags.of("method", "POST"), Meter.Type.COUNTER, null, null);

        assertThat(a).isNotEqualTo(b);
    }

    @Test
    void shouldCreateNewId_WhenWithNameCalled() {
        Meter.Id original = new Meter.Id("old.name", Tags.of("a", "1"), Meter.Type.COUNTER, "desc", "unit");
        Meter.Id renamed = original.withName("new.name");

        assertThat(renamed.getName()).isEqualTo("new.name");
        assertThat(renamed.getTagsAsIterable()).isEqualTo(original.getTagsAsIterable());
        assertThat(renamed.getType()).isEqualTo(original.getType());
        assertThat(renamed.getDescription()).isEqualTo(original.getDescription());
    }

    @Test
    void shouldAddTag_WhenWithTagCalled() {
        Meter.Id original = new Meter.Id("metric", Tags.of("a", "1"), Meter.Type.GAUGE, null, null);
        Meter.Id withTag = original.withTag(Tag.of("b", "2"));

        assertThat(withTag.getTags()).containsExactly(Tag.of("a", "1"), Tag.of("b", "2"));
        // Original is unchanged (immutable)
        assertThat(original.getTags()).containsExactly(Tag.of("a", "1"));
    }

    @Test
    void shouldAddStatisticTag_WhenWithTagStatisticCalled() {
        Meter.Id id = new Meter.Id("metric", Tags.empty(), Meter.Type.COUNTER, null, null);
        Meter.Id withStat = id.withTag(Statistic.COUNT);

        assertThat(withStat.getTag("statistic")).isEqualTo("count");
    }

    @Test
    void shouldLookUpTagValue_WhenGetTagCalledWithExistingKey() {
        Meter.Id id = new Meter.Id("metric", Tags.of("env", "prod", "region", "us"), Meter.Type.GAUGE, null, null);

        assertThat(id.getTag("env")).isEqualTo("prod");
        assertThat(id.getTag("region")).isEqualTo("us");
    }

    @Test
    void shouldReturnNull_WhenGetTagCalledWithMissingKey() {
        Meter.Id id = new Meter.Id("metric", Tags.of("env", "prod"), Meter.Type.GAUGE, null, null);

        assertThat(id.getTag("nonexistent")).isNull();
    }

    @Test
    void shouldReplaceAllTags_WhenReplaceTagsCalled() {
        Meter.Id original = new Meter.Id("metric", Tags.of("a", "1", "b", "2"), Meter.Type.GAUGE, null, null);
        Meter.Id replaced = original.replaceTags(Tags.of("c", "3"));

        assertThat(replaced.getTags()).containsExactly(Tag.of("c", "3"));
    }

    @Test
    void shouldDefaultToEmptyTags_WhenNullTagsProvided() {
        Meter.Id id = new Meter.Id("metric", null, Meter.Type.COUNTER, null, null);

        assertThat(id.getTagsAsIterable()).isEqualTo(Tags.empty());
    }

    @Test
    void shouldMergeTags_WhenWithTagsCalled() {
        Meter.Id id = new Meter.Id("metric", Tags.of("a", "1"), Meter.Type.GAUGE, null, null);
        Meter.Id merged = id.withTags(Tags.of("b", "2", "c", "3"));

        assertThat(merged.getTags()).containsExactly(Tag.of("a", "1"), Tag.of("b", "2"), Tag.of("c", "3"));
    }

    @Test
    void shouldUpdateBaseUnit_WhenWithBaseUnitCalled() {
        Meter.Id id = new Meter.Id("metric", Tags.empty(), Meter.Type.TIMER, null, null);
        Meter.Id withUnit = id.withBaseUnit("ms");

        assertThat(withUnit.getBaseUnit()).isEqualTo("ms");
        assertThat(id.getBaseUnit()).isNull(); // original unchanged
    }

}
```

### StatisticTest (2 tests)

```java
package dev.linhvu.micrometer;

import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;

class StatisticTest {

    @Test
    void shouldHaveCorrectTagRepresentations() {
        assertThat(Statistic.TOTAL.getTagValueRepresentation()).isEqualTo("total");
        assertThat(Statistic.TOTAL_TIME.getTagValueRepresentation()).isEqualTo("total");
        assertThat(Statistic.COUNT.getTagValueRepresentation()).isEqualTo("count");
        assertThat(Statistic.MAX.getTagValueRepresentation()).isEqualTo("max");
        assertThat(Statistic.VALUE.getTagValueRepresentation()).isEqualTo("value");
        assertThat(Statistic.UNKNOWN.getTagValueRepresentation()).isEqualTo("unknown");
        assertThat(Statistic.ACTIVE_TASKS.getTagValueRepresentation()).isEqualTo("active");
        assertThat(Statistic.DURATION.getTagValueRepresentation()).isEqualTo("duration");
    }

    @Test
    void shouldShareTagRepresentation_BetweenTotalAndTotalTime() {
        // TOTAL (for amounts like bytes) and TOTAL_TIME (for durations)
        // both export as "total" — the meter type disambiguates them
        assertThat(Statistic.TOTAL.getTagValueRepresentation())
            .isEqualTo(Statistic.TOTAL_TIME.getTagValueRepresentation());
    }

}
```

### MeasurementTest (4 tests)

```java
package dev.linhvu.micrometer;

import org.junit.jupiter.api.Test;

import java.util.function.Supplier;

import static org.assertj.core.api.Assertions.assertThat;

class MeasurementTest {

    @Test
    void shouldEvaluateLazily_WhenGetValueCalled() {
        double[] value = { 0.0 };
        Measurement m = new Measurement(() -> value[0], Statistic.COUNT);

        assertThat(m.getValue()).isEqualTo(0.0);

        value[0] = 42.0;
        assertThat(m.getValue()).isEqualTo(42.0);
    }

    @Test
    void shouldAcceptSupplierOfDouble() {
        Supplier<Double> supplier = () -> 3.14;
        Measurement m = new Measurement(supplier, Statistic.VALUE);

        assertThat(m.getValue()).isEqualTo(3.14);
    }

    @Test
    void shouldReturnStatistic() {
        Measurement m = new Measurement(() -> 0.0, Statistic.MAX);

        assertThat(m.getStatistic()).isEqualTo(Statistic.MAX);
    }

    @Test
    void shouldProduceReadableToString() {
        Measurement m = new Measurement(() -> 5.0, Statistic.COUNT);

        assertThat(m.toString()).contains("COUNT").contains("5.0");
    }

}
```

---

## 1.7 Why This Works

> **Insight 1: The intentional `compareTo` vs `equals` inconsistency in Tag**
>
> Java's `Comparable` contract recommends that `compareTo` be consistent with `equals`. Micrometer deliberately breaks this rule. `Tag.compareTo` sorts by key only so that `Tags` can detect duplicate keys (same key = `compareTo` returns 0) and keep just one. But `ImmutableTag.equals` checks both key and value because you need to distinguish `tag(env=prod)` from `tag(env=staging)` in identity checks (like `Tags.equals` and `Meter.Id.equals`).
>
> The trade-off: you cannot safely put `Tag` objects into a `TreeSet` or use them as `TreeMap` keys, because `TreeSet` uses `compareTo` for equality. But Micrometer never does that — it uses its own sorted-array container (`Tags`). The benefit is a clean separation: sorting logic lives in the interface, identity logic lives in the implementation.

> **Insight 2: Sorted array over Map for tag storage**
>
> A typical metrics meter has 2-5 tags. At this scale, a sorted array beats `HashMap` and `TreeMap` on every axis:
>
> | Operation | Sorted Array | HashMap | TreeMap |
> |---|---|---|---|
> | Iteration | O(n), cache-friendly | O(capacity), pointer-chasing | O(n), pointer-chasing |
> | Merge two sets | O(n+m), single pass | O(n+m), hashing + collisions | O(n+m log n), insertions |
> | Memory per tag | 1 object reference | Entry + hash + next pointer | Entry + left/right/parent/color |
> | Equality check | O(n), array walk | O(n), bucket walk | O(n), tree walk |
>
> For the small cardinalities Micrometer deals with, the sorted array's cache locality and zero per-entry overhead make it the clear winner. The cost is that you must keep the invariant (sorted, deduplicated) maintained — which `toTags()` and `merge()` handle.

> **Insight 3: Identity = name + tags, not name + tags + type**
>
> `Meter.Id.equals` deliberately ignores `type`, `description`, and `baseUnit`. This is a registry-level design decision: the registry maps `name+tags` to exactly one meter. If two pieces of code try to register a Counter and a Timer with the same name and tags, the registry can detect the conflict.
>
> If type were part of identity, the registry would silently store both, and you would end up with two time series that look identical to your monitoring backend (same metric name, same tag labels) but contain incompatible data. By making identity strict, Micrometer forces conflicts to surface early.
>
> The trade-off: metadata like `description` and `baseUnit` must be resolved separately (the registry typically keeps the first description it sees). But this is a small price for avoiding silent data corruption in your monitoring pipeline.

---

## 1.8 Connection to Real Micrometer

Each class in our simplified implementation maps to its real counterpart in the Micrometer codebase at commit `2c8a4606c`:

| Our Class | Real Micrometer Class | File | Key Difference |
|---|---|---|---|
| `Tag` | `io.micrometer.common.Tag` | [`micrometer-commons/src/main/java/io/micrometer/common/Tag.java`](https://github.com/micrometer-metrics/micrometer/blob/2c8a4606c/micrometer-commons/src/main/java/io/micrometer/common/Tag.java) | Real version is in `micrometer-commons` module, shared across Micrometer and Micrometer Tracing |
| `ImmutableTag` | `io.micrometer.common.ImmutableTag` | [`micrometer-commons/src/main/java/io/micrometer/common/ImmutableTag.java`](https://github.com/micrometer-metrics/micrometer/blob/2c8a4606c/micrometer-commons/src/main/java/io/micrometer/common/ImmutableTag.java) | Real version is identical in design |
| `Tags` | `io.micrometer.core.instrument.Tags` | [`micrometer-core/src/main/java/io/micrometer/core/instrument/Tags.java`](https://github.com/micrometer-metrics/micrometer/blob/2c8a4606c/micrometer-core/src/main/java/io/micrometer/core/instrument/Tags.java) | Real version has the same sorted-array design with `dedup()` and `merge()` |
| `Statistic` | `io.micrometer.core.instrument.Statistic` | [`micrometer-core/src/main/java/io/micrometer/core/instrument/Statistic.java`](https://github.com/micrometer-metrics/micrometer/blob/2c8a4606c/micrometer-core/src/main/java/io/micrometer/core/instrument/Statistic.java) | Real version is identical — same enum constants, same `tagValueRepresentation` |
| `Measurement` | `io.micrometer.core.instrument.Measurement` | [`micrometer-core/src/main/java/io/micrometer/core/instrument/Measurement.java`](https://github.com/micrometer-metrics/micrometer/blob/2c8a4606c/micrometer-core/src/main/java/io/micrometer/core/instrument/Measurement.java) | Real version is identical — same lazy `DoubleSupplier` pattern |
| `Meter` (interface) | `io.micrometer.core.instrument.Meter` | [`micrometer-core/src/main/java/io/micrometer/core/instrument/Meter.java`](https://github.com/micrometer-metrics/micrometer/blob/2c8a4606c/micrometer-core/src/main/java/io/micrometer/core/instrument/Meter.java) | Real version has `getId()`, `measure()`, `match()`/`use()` visitor methods, and a fluent `Builder` — we add those in Chapter 2 |
| `Meter.Id` | `io.micrometer.core.instrument.Meter.Id` | Same file as above | Real version has additional `syntheticAssociation` field and convention-based name transformations — same `equals`/`hashCode` using name+tags only |
| `Meter.Type` | `io.micrometer.core.instrument.Meter.Type` | Same file as above | Real version is identical |

Notable design choices preserved from real Micrometer:
- `Tag.compareTo` by key only, `ImmutableTag.equals` by key+value
- `Tags` backed by sorted array with O(n+m) merge
- `Meter.Id` identity = name+tags (type/description/baseUnit excluded from `equals`)
- `Measurement` lazy evaluation via `DoubleSupplier`

---

## 1.9 Complete Code

Below is the full content of every source file, exactly as it exists in the repository.

### Production Code

#### `src/main/java/dev/linhvu/micrometer/Tag.java`

```java
package dev.linhvu.micrometer;

/**
 * Key-value pair used as a dimensional label on a meter.
 * Tags enable filtering and grouping of metrics (e.g., "method"="GET", "status"="200").
 * <p>
 * Natural ordering is by {@link #getKey()} only — this is intentional so that
 * {@link Tags} can sort and deduplicate by key.
 */
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

#### `src/main/java/dev/linhvu/micrometer/ImmutableTag.java`

```java
package dev.linhvu.micrometer;

import java.util.Objects;

/**
 * An immutable implementation of {@link Tag}.
 * <p>
 * Note: {@code equals}/{@code hashCode} use both key and value,
 * while {@code compareTo} (inherited from Tag) uses only the key.
 * This intentional inconsistency allows Tags to sort by key while
 * still distinguishing tags with different values.
 */
public class ImmutableTag implements Tag {

    private final String key;

    private final String value;

    public ImmutableTag(String key, String value) {
        Objects.requireNonNull(key, "key must not be null");
        Objects.requireNonNull(value, "value must not be null");
        this.key = key;
        this.value = value;
    }

    @Override
    public String getKey() {
        return key;
    }

    @Override
    public String getValue() {
        return value;
    }

    @Override
    public boolean equals(Object o) {
        if (this == o)
            return true;
        if (!(o instanceof ImmutableTag that))
            return false;
        return key.equals(that.getKey()) && value.equals(that.getValue());
    }

    @Override
    public int hashCode() {
        int result = key.hashCode();
        result = 31 * result + value.hashCode();
        return result;
    }

    @Override
    public String toString() {
        return "tag(" + key + "=" + value + ")";
    }

}
```

#### `src/main/java/dev/linhvu/micrometer/Tags.java`

```java
package dev.linhvu.micrometer;

import java.util.ArrayList;
import java.util.Arrays;
import java.util.Collection;
import java.util.Collections;
import java.util.Iterator;
import java.util.List;
import java.util.NoSuchElementException;
import java.util.Spliterator;
import java.util.Spliterators;
import java.util.stream.Collectors;
import java.util.stream.Stream;
import java.util.stream.StreamSupport;

/**
 * An immutable, sorted, deduplicated collection of {@link Tag}s.
 * <p>
 * Tags are sorted by key. When duplicate keys are present, the last value wins.
 * Merging two Tags collections ({@link #and}) gives precedence to the "other"
 * (newer) collection on key conflicts.
 * <p>
 * Backed by a sorted array for memory efficiency and O(n+m) merge performance.
 */
public final class Tags implements Iterable<Tag> {

    private static final Tag[] EMPTY_TAG_ARRAY = new Tag[0];

    private static final Tags EMPTY = new Tags(EMPTY_TAG_ARRAY, 0);

    private final Tag[] sortedSet;

    private final int length;

    private Tags(Tag[] sortedSet, int length) {
        this.sortedSet = sortedSet;
        this.length = length;
    }

    // -----------------------------------------------------------------------
    // Factory methods
    // -----------------------------------------------------------------------

    public static Tags empty() {
        return EMPTY;
    }

    public static Tags of(String key, String value) {
        return new Tags(new Tag[] { Tag.of(key, value) }, 1);
    }

    public static Tags of(Tag... tags) {
        if (tags == null || tags.length == 0) {
            return EMPTY;
        }
        return toTags(tags.clone());
    }

    public static Tags of(String... keyValues) {
        if (keyValues == null || keyValues.length == 0) {
            return EMPTY;
        }
        if (keyValues.length % 2 != 0) {
            throw new IllegalArgumentException("size must be even, it is a set of key=value pairs");
        }
        Tag[] tags = new Tag[keyValues.length / 2];
        for (int i = 0; i < keyValues.length; i += 2) {
            tags[i / 2] = Tag.of(keyValues[i], keyValues[i + 1]);
        }
        return toTags(tags);
    }

    @SuppressWarnings("unchecked")
    public static Tags of(Iterable<? extends Tag> tags) {
        if (tags instanceof Tags t) {
            return t;
        }
        if (tags instanceof Collection<? extends Tag> c) {
            if (c.isEmpty()) {
                return EMPTY;
            }
            return toTags(c.toArray(EMPTY_TAG_ARRAY));
        }
        return toTags(iterableToArray(tags));
    }

    // -----------------------------------------------------------------------
    // Merge methods
    // -----------------------------------------------------------------------

    public Tags and(String key, String value) {
        return and(Tag.of(key, value));
    }

    public Tags and(Tag... tags) {
        if (tags == null || tags.length == 0) {
            return this;
        }
        return merge(Tags.of(tags));
    }

    public Tags and(Iterable<? extends Tag> tags) {
        if (tags instanceof Tags other) {
            return merge(other);
        }
        return merge(Tags.of(tags));
    }

    public static Tags concat(Iterable<? extends Tag> tagsA, Iterable<? extends Tag> tagsB) {
        return Tags.of(tagsA).and(tagsB);
    }

    // -----------------------------------------------------------------------
    // Query methods
    // -----------------------------------------------------------------------

    public Stream<Tag> stream() {
        return StreamSupport.stream(Spliterators.spliterator(sortedSet, 0, length,
                Spliterator.IMMUTABLE | Spliterator.ORDERED | Spliterator.DISTINCT | Spliterator.NONNULL
                        | Spliterator.SORTED),
                false);
    }

    @Override
    public Iterator<Tag> iterator() {
        return new ArrayIterator(sortedSet, length);
    }

    // -----------------------------------------------------------------------
    // equals / hashCode / toString
    // -----------------------------------------------------------------------

    @Override
    public boolean equals(Object o) {
        if (this == o)
            return true;
        if (!(o instanceof Tags other))
            return false;
        if (this.length != other.length)
            return false;
        for (int i = 0; i < length; i++) {
            if (!sortedSet[i].equals(other.sortedSet[i]))
                return false;
        }
        return true;
    }

    @Override
    public int hashCode() {
        int result = 1;
        for (int i = 0; i < length; i++) {
            result = 31 * result + sortedSet[i].hashCode();
        }
        return result;
    }

    @Override
    public String toString() {
        return stream().map(Tag::toString).collect(Collectors.joining(",", "[", "]"));
    }

    // -----------------------------------------------------------------------
    // Internal algorithms
    // -----------------------------------------------------------------------

    private static Tags toTags(Tag[] tags) {
        int len = tags.length;
        if (len == 0) {
            return EMPTY;
        }
        if (len == 1) {
            return new Tags(tags, 1);
        }
        // Fast path: already a sorted set (no duplicates, strictly ascending by key)
        if (isSortedSet(tags, len)) {
            return new Tags(tags, len);
        }
        Arrays.sort(tags);
        return dedup(tags);
    }

    /**
     * Returns true if the first {@code length} elements are strictly ordered
     * by key — no duplicate keys allowed.
     */
    private static boolean isSortedSet(Tag[] tags, int length) {
        for (int i = 0; i < length - 1; i++) {
            if (tags[i].compareTo(tags[i + 1]) >= 0) {
                return false;
            }
        }
        return true;
    }

    /**
     * Removes duplicate keys from a sorted array. When the same key appears
     * multiple times, the LAST occurrence wins (it appears latest in sort order).
     */
    private static Tags dedup(Tag[] sortedTags) {
        int len = sortedTags.length;
        int writeIdx = 0;
        for (int readIdx = 0; readIdx < len; readIdx++) {
            // Skip forward to the last tag with the same key
            while (readIdx < len - 1 && sortedTags[readIdx].compareTo(sortedTags[readIdx + 1]) == 0) {
                readIdx++;
            }
            sortedTags[writeIdx++] = sortedTags[readIdx];
        }
        return new Tags(sortedTags, writeIdx);
    }

    /**
     * Classic sorted-merge of two sorted sets. On key conflict, the "other"
     * tag wins — this is why {@code tags.and(newTags)} gives precedence to newTags.
     */
    private Tags merge(Tags other) {
        if (other.length == 0)
            return this;
        if (this.length == 0)
            return other;

        Tag[] merged = new Tag[this.length + other.length];
        int thisIdx = 0, otherIdx = 0, writeIdx = 0;

        while (thisIdx < this.length && otherIdx < other.length) {
            Tag thisTag = this.sortedSet[thisIdx];
            Tag otherTag = other.sortedSet[otherIdx];
            int cmp = thisTag.compareTo(otherTag);
            if (cmp < 0) {
                merged[writeIdx++] = thisTag;
                thisIdx++;
            }
            else if (cmp > 0) {
                merged[writeIdx++] = otherTag;
                otherIdx++;
            }
            else {
                // Same key — other wins (newer overrides older)
                merged[writeIdx++] = otherTag;
                thisIdx++;
                otherIdx++;
            }
        }
        while (thisIdx < this.length) {
            merged[writeIdx++] = this.sortedSet[thisIdx++];
        }
        while (otherIdx < other.length) {
            merged[writeIdx++] = other.sortedSet[otherIdx++];
        }

        return new Tags(merged, writeIdx);
    }

    private static Tag[] iterableToArray(Iterable<? extends Tag> tags) {
        List<Tag> list = new ArrayList<>();
        tags.forEach(list::add);
        return list.toArray(EMPTY_TAG_ARRAY);
    }

    // -----------------------------------------------------------------------
    // Inner classes
    // -----------------------------------------------------------------------

    private static class ArrayIterator implements Iterator<Tag> {

        private final Tag[] tags;

        private final int length;

        private int index = 0;

        ArrayIterator(Tag[] tags, int length) {
            this.tags = tags;
            this.length = length;
        }

        @Override
        public boolean hasNext() {
            return index < length;
        }

        @Override
        public Tag next() {
            if (!hasNext()) {
                throw new NoSuchElementException();
            }
            return tags[index++];
        }

        @Override
        public void remove() {
            throw new UnsupportedOperationException("Tags are immutable");
        }

    }

}
```

#### `src/main/java/dev/linhvu/micrometer/Statistic.java`

```java
package dev.linhvu.micrometer;

/**
 * Enumerates the types of statistics a {@link Measurement} can represent.
 * <p>
 * Each statistic has a tag-value representation used when exporting individual
 * measurements as separate time series (e.g., a Timer emits COUNT, TOTAL_TIME, and MAX).
 */
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

    /**
     * Returns the string used as the value of a "statistic" tag when this
     * measurement is exported. For example, {@code COUNT} becomes {@code "count"}.
     */
    public String getTagValueRepresentation() {
        return tagValueRepresentation;
    }

}
```

#### `src/main/java/dev/linhvu/micrometer/Measurement.java`

```java
package dev.linhvu.micrometer;

import java.util.function.DoubleSupplier;
import java.util.function.Supplier;

/**
 * A sampled value paired with the {@link Statistic} it represents.
 * <p>
 * Measurements use lazy evaluation — the value is computed fresh on every call
 * to {@link #getValue()} via a {@link DoubleSupplier}. This means measurements
 * are always live snapshots rather than stale stored values.
 */
public class Measurement {

    private final DoubleSupplier f;

    private final Statistic statistic;

    public Measurement(DoubleSupplier valueFunction, Statistic statistic) {
        this.f = valueFunction;
        this.statistic = statistic;
    }

    public Measurement(Supplier<Double> valueFunction, Statistic statistic) {
        this.f = valueFunction::get;
        this.statistic = statistic;
    }

    public double getValue() {
        return f.getAsDouble();
    }

    public Statistic getStatistic() {
        return statistic;
    }

    @Override
    public String toString() {
        return "Measurement{statistic='" + statistic + "', value=" + getValue() + '}';
    }

}
```

#### `src/main/java/dev/linhvu/micrometer/Meter.java`

```java
package dev.linhvu.micrometer;

import java.util.ArrayList;
import java.util.Collections;
import java.util.List;
import java.util.Objects;

/**
 * A named and tagged set of measurements. This is the base type for all meters.
 * <p>
 * Feature 1 defines the identity system ({@link Id}, {@link Type}).
 * The full Meter interface ({@code getId}, {@code measure}, visitor methods)
 * will be added in Feature 2.
 */
public interface Meter {

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

### Test Code

#### `src/test/java/dev/linhvu/micrometer/TagTest.java`

```java
package dev.linhvu.micrometer;

import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;

class TagTest {

    @Test
    void shouldCreateTag_WhenUsingStaticFactory() {
        Tag tag = Tag.of("env", "prod");

        assertThat(tag.getKey()).isEqualTo("env");
        assertThat(tag.getValue()).isEqualTo("prod");
    }

    @Test
    void shouldReturnZero_WhenComparingTagsWithSameKey() {
        Tag a = Tag.of("host", "server1");
        Tag b = Tag.of("host", "server2");

        // compareTo uses key only — same key means compareTo == 0
        assertThat(a.compareTo(b)).isEqualTo(0);
    }

    @Test
    void shouldOrderAlphabetically_WhenComparingByKey() {
        Tag a = Tag.of("alpha", "1");
        Tag b = Tag.of("beta", "2");

        assertThat(a.compareTo(b)).isLessThan(0);
        assertThat(b.compareTo(a)).isGreaterThan(0);
    }

    @Test
    void shouldReturnImmutableTag_WhenUsingStaticFactory() {
        Tag tag = Tag.of("key", "value");

        assertThat(tag).isInstanceOf(ImmutableTag.class);
    }

}
```

#### `src/test/java/dev/linhvu/micrometer/ImmutableTagTest.java`

```java
package dev.linhvu.micrometer;

import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatNullPointerException;

class ImmutableTagTest {

    @Test
    void shouldBeEqual_WhenKeyAndValueMatch() {
        ImmutableTag a = new ImmutableTag("env", "prod");
        ImmutableTag b = new ImmutableTag("env", "prod");

        assertThat(a).isEqualTo(b);
        assertThat(a.hashCode()).isEqualTo(b.hashCode());
    }

    @Test
    void shouldNotBeEqual_WhenValuesDiffer() {
        ImmutableTag a = new ImmutableTag("env", "prod");
        ImmutableTag b = new ImmutableTag("env", "staging");

        assertThat(a).isNotEqualTo(b);
    }

    @Test
    void shouldNotBeEqual_WhenKeysDiffer() {
        ImmutableTag a = new ImmutableTag("env", "prod");
        ImmutableTag b = new ImmutableTag("region", "prod");

        assertThat(a).isNotEqualTo(b);
    }

    @Test
    void shouldRejectNullKey() {
        assertThatNullPointerException().isThrownBy(() -> new ImmutableTag(null, "value"));
    }

    @Test
    void shouldRejectNullValue() {
        assertThatNullPointerException().isThrownBy(() -> new ImmutableTag("key", null));
    }

    @Test
    void shouldProduceReadableToString() {
        assertThat(new ImmutableTag("env", "prod").toString()).isEqualTo("tag(env=prod)");
    }

}
```

#### `src/test/java/dev/linhvu/micrometer/TagsTest.java`

```java
package dev.linhvu.micrometer;

import org.junit.jupiter.api.Test;

import java.util.List;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatIllegalArgumentException;

class TagsTest {

    @Test
    void shouldCreateEmptyTags() {
        Tags tags = Tags.empty();

        assertThat(tags).isEmpty();
    }

    @Test
    void shouldCreateFromSingleKeyValue() {
        Tags tags = Tags.of("env", "prod");

        assertThat(tags).containsExactly(Tag.of("env", "prod"));
    }

    @Test
    void shouldSortTagsByKey() {
        Tags tags = Tags.of(Tag.of("zone", "us-east"), Tag.of("app", "web"), Tag.of("env", "prod"));

        List<String> keys = tags.stream().map(Tag::getKey).toList();
        assertThat(keys).containsExactly("app", "env", "zone");
    }

    @Test
    void shouldDeduplicateByKey_LastValueWins() {
        Tags tags = Tags.of(Tag.of("env", "staging"), Tag.of("env", "prod"));

        // Both have key "env" — after sorting by key they're adjacent,
        // and the last occurrence ("prod") wins
        assertThat(tags).containsExactly(Tag.of("env", "prod"));
    }

    @Test
    void shouldCreateFromAlternatingKeyValueStrings() {
        Tags tags = Tags.of("a", "1", "b", "2");

        assertThat(tags).containsExactly(Tag.of("a", "1"), Tag.of("b", "2"));
    }

    @Test
    void shouldRejectOddNumberOfStrings() {
        assertThatIllegalArgumentException().isThrownBy(() -> Tags.of("a", "1", "orphan"));
    }

    @Test
    void shouldMergeTags_OtherWinsOnConflict() {
        Tags base = Tags.of("env", "prod", "host", "server1");
        Tags override = Tags.of("env", "staging", "region", "us-east");

        Tags merged = base.and(override);

        assertThat(merged).containsExactly(Tag.of("env", "staging"), // overridden by "other"
                Tag.of("host", "server1"), // kept from base
                Tag.of("region", "us-east") // added from other
        );
    }

    @Test
    void shouldSupportAndWithSingleKeyValue() {
        Tags tags = Tags.of("a", "1").and("b", "2");

        assertThat(tags).containsExactly(Tag.of("a", "1"), Tag.of("b", "2"));
    }

    @Test
    void shouldConcatTwoIterables() {
        Tags a = Tags.of("x", "1");
        Tags b = Tags.of("y", "2");

        Tags result = Tags.concat(a, b);

        assertThat(result).containsExactly(Tag.of("x", "1"), Tag.of("y", "2"));
    }

    @Test
    void shouldStreamTags() {
        Tags tags = Tags.of("a", "1", "b", "2");

        List<String> keys = tags.stream().map(Tag::getKey).toList();
        assertThat(keys).containsExactly("a", "b");
    }

    @Test
    void shouldBeEqual_WhenContainingSameTags() {
        Tags a = Tags.of("env", "prod", "host", "s1");
        Tags b = Tags.of("host", "s1", "env", "prod"); // different insertion order

        assertThat(a).isEqualTo(b);
        assertThat(a.hashCode()).isEqualTo(b.hashCode());
    }

    @Test
    void shouldNotBeEqual_WhenTagsDiffer() {
        Tags a = Tags.of("env", "prod");
        Tags b = Tags.of("env", "staging");

        assertThat(a).isNotEqualTo(b);
    }

    @Test
    void shouldReturnSameInstance_WhenMergingWithEmpty() {
        Tags tags = Tags.of("a", "1");

        assertThat(tags.and(Tags.empty())).isSameAs(tags);
    }

    @Test
    void shouldReturnOther_WhenThisIsEmpty() {
        Tags other = Tags.of("a", "1");

        assertThat(Tags.empty().and(other)).isSameAs(other);
    }

    @Test
    void shouldHandleAlreadySortedInput() {
        // Fast path: input is already sorted and has no duplicates
        Tags tags = Tags.of(Tag.of("a", "1"), Tag.of("b", "2"), Tag.of("c", "3"));

        assertThat(tags).containsExactly(Tag.of("a", "1"), Tag.of("b", "2"), Tag.of("c", "3"));
    }

    @Test
    void shouldCreateFromIterable() {
        List<Tag> tagList = List.of(Tag.of("b", "2"), Tag.of("a", "1"));

        Tags tags = Tags.of(tagList);

        assertThat(tags).containsExactly(Tag.of("a", "1"), Tag.of("b", "2"));
    }

    @Test
    void shouldReturnSameInstance_WhenCreatingFromTags() {
        Tags original = Tags.of("a", "1");

        assertThat(Tags.of(original)).isSameAs(original);
    }

}
```

#### `src/test/java/dev/linhvu/micrometer/MeterIdTest.java`

```java
package dev.linhvu.micrometer;

import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;

class MeterIdTest {

    @Test
    void shouldBeEqual_WhenNameAndTagsMatch() {
        Meter.Id a = new Meter.Id("http.requests", Tags.of("method", "GET"), Meter.Type.COUNTER, "Request count",
                null);
        Meter.Id b = new Meter.Id("http.requests", Tags.of("method", "GET"), Meter.Type.TIMER,
                "Different description", "ms");

        // Type, description, and baseUnit are NOT part of identity
        assertThat(a).isEqualTo(b);
        assertThat(a.hashCode()).isEqualTo(b.hashCode());
    }

    @Test
    void shouldNotBeEqual_WhenNamesDiffer() {
        Meter.Id a = new Meter.Id("http.requests", Tags.empty(), Meter.Type.COUNTER, null, null);
        Meter.Id b = new Meter.Id("http.responses", Tags.empty(), Meter.Type.COUNTER, null, null);

        assertThat(a).isNotEqualTo(b);
    }

    @Test
    void shouldNotBeEqual_WhenTagsDiffer() {
        Meter.Id a = new Meter.Id("http.requests", Tags.of("method", "GET"), Meter.Type.COUNTER, null, null);
        Meter.Id b = new Meter.Id("http.requests", Tags.of("method", "POST"), Meter.Type.COUNTER, null, null);

        assertThat(a).isNotEqualTo(b);
    }

    @Test
    void shouldCreateNewId_WhenWithNameCalled() {
        Meter.Id original = new Meter.Id("old.name", Tags.of("a", "1"), Meter.Type.COUNTER, "desc", "unit");
        Meter.Id renamed = original.withName("new.name");

        assertThat(renamed.getName()).isEqualTo("new.name");
        assertThat(renamed.getTagsAsIterable()).isEqualTo(original.getTagsAsIterable());
        assertThat(renamed.getType()).isEqualTo(original.getType());
        assertThat(renamed.getDescription()).isEqualTo(original.getDescription());
    }

    @Test
    void shouldAddTag_WhenWithTagCalled() {
        Meter.Id original = new Meter.Id("metric", Tags.of("a", "1"), Meter.Type.GAUGE, null, null);
        Meter.Id withTag = original.withTag(Tag.of("b", "2"));

        assertThat(withTag.getTags()).containsExactly(Tag.of("a", "1"), Tag.of("b", "2"));
        // Original is unchanged (immutable)
        assertThat(original.getTags()).containsExactly(Tag.of("a", "1"));
    }

    @Test
    void shouldAddStatisticTag_WhenWithTagStatisticCalled() {
        Meter.Id id = new Meter.Id("metric", Tags.empty(), Meter.Type.COUNTER, null, null);
        Meter.Id withStat = id.withTag(Statistic.COUNT);

        assertThat(withStat.getTag("statistic")).isEqualTo("count");
    }

    @Test
    void shouldLookUpTagValue_WhenGetTagCalledWithExistingKey() {
        Meter.Id id = new Meter.Id("metric", Tags.of("env", "prod", "region", "us"), Meter.Type.GAUGE, null, null);

        assertThat(id.getTag("env")).isEqualTo("prod");
        assertThat(id.getTag("region")).isEqualTo("us");
    }

    @Test
    void shouldReturnNull_WhenGetTagCalledWithMissingKey() {
        Meter.Id id = new Meter.Id("metric", Tags.of("env", "prod"), Meter.Type.GAUGE, null, null);

        assertThat(id.getTag("nonexistent")).isNull();
    }

    @Test
    void shouldReplaceAllTags_WhenReplaceTagsCalled() {
        Meter.Id original = new Meter.Id("metric", Tags.of("a", "1", "b", "2"), Meter.Type.GAUGE, null, null);
        Meter.Id replaced = original.replaceTags(Tags.of("c", "3"));

        assertThat(replaced.getTags()).containsExactly(Tag.of("c", "3"));
    }

    @Test
    void shouldDefaultToEmptyTags_WhenNullTagsProvided() {
        Meter.Id id = new Meter.Id("metric", null, Meter.Type.COUNTER, null, null);

        assertThat(id.getTagsAsIterable()).isEqualTo(Tags.empty());
    }

    @Test
    void shouldMergeTags_WhenWithTagsCalled() {
        Meter.Id id = new Meter.Id("metric", Tags.of("a", "1"), Meter.Type.GAUGE, null, null);
        Meter.Id merged = id.withTags(Tags.of("b", "2", "c", "3"));

        assertThat(merged.getTags()).containsExactly(Tag.of("a", "1"), Tag.of("b", "2"), Tag.of("c", "3"));
    }

    @Test
    void shouldUpdateBaseUnit_WhenWithBaseUnitCalled() {
        Meter.Id id = new Meter.Id("metric", Tags.empty(), Meter.Type.TIMER, null, null);
        Meter.Id withUnit = id.withBaseUnit("ms");

        assertThat(withUnit.getBaseUnit()).isEqualTo("ms");
        assertThat(id.getBaseUnit()).isNull(); // original unchanged
    }

}
```

#### `src/test/java/dev/linhvu/micrometer/StatisticTest.java`

```java
package dev.linhvu.micrometer;

import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;

class StatisticTest {

    @Test
    void shouldHaveCorrectTagRepresentations() {
        assertThat(Statistic.TOTAL.getTagValueRepresentation()).isEqualTo("total");
        assertThat(Statistic.TOTAL_TIME.getTagValueRepresentation()).isEqualTo("total");
        assertThat(Statistic.COUNT.getTagValueRepresentation()).isEqualTo("count");
        assertThat(Statistic.MAX.getTagValueRepresentation()).isEqualTo("max");
        assertThat(Statistic.VALUE.getTagValueRepresentation()).isEqualTo("value");
        assertThat(Statistic.UNKNOWN.getTagValueRepresentation()).isEqualTo("unknown");
        assertThat(Statistic.ACTIVE_TASKS.getTagValueRepresentation()).isEqualTo("active");
        assertThat(Statistic.DURATION.getTagValueRepresentation()).isEqualTo("duration");
    }

    @Test
    void shouldShareTagRepresentation_BetweenTotalAndTotalTime() {
        // TOTAL (for amounts like bytes) and TOTAL_TIME (for durations)
        // both export as "total" — the meter type disambiguates them
        assertThat(Statistic.TOTAL.getTagValueRepresentation())
            .isEqualTo(Statistic.TOTAL_TIME.getTagValueRepresentation());
    }

}
```

#### `src/test/java/dev/linhvu/micrometer/MeasurementTest.java`

```java
package dev.linhvu.micrometer;

import org.junit.jupiter.api.Test;

import java.util.function.Supplier;

import static org.assertj.core.api.Assertions.assertThat;

class MeasurementTest {

    @Test
    void shouldEvaluateLazily_WhenGetValueCalled() {
        double[] value = { 0.0 };
        Measurement m = new Measurement(() -> value[0], Statistic.COUNT);

        assertThat(m.getValue()).isEqualTo(0.0);

        value[0] = 42.0;
        assertThat(m.getValue()).isEqualTo(42.0);
    }

    @Test
    void shouldAcceptSupplierOfDouble() {
        Supplier<Double> supplier = () -> 3.14;
        Measurement m = new Measurement(supplier, Statistic.VALUE);

        assertThat(m.getValue()).isEqualTo(3.14);
    }

    @Test
    void shouldReturnStatistic() {
        Measurement m = new Measurement(() -> 0.0, Statistic.MAX);

        assertThat(m.getStatistic()).isEqualTo(Statistic.MAX);
    }

    @Test
    void shouldProduceReadableToString() {
        Measurement m = new Measurement(() -> 5.0, Statistic.COUNT);

        assertThat(m.toString()).contains("COUNT").contains("5.0");
    }

}
```

---

## Summary

| Class | Role | Key Design Decision |
|---|---|---|
| `Tag` | Interface for key-value dimensional label | `compareTo` by key only |
| `ImmutableTag` | Immutable `Tag` implementation | `equals` by key+value (intentionally inconsistent with `compareTo`) |
| `Tags` | Sorted, deduplicated tag collection | Sorted array with O(n+m) merge; last-value-wins dedup; other-wins merge |
| `Statistic` | Enum of measurement statistic types | `TOTAL` and `TOTAL_TIME` share `"total"` tag representation |
| `Measurement` | Lazy value + statistic pair | `DoubleSupplier` for live snapshots, never stale |
| `Meter.Type` | Enum of meter kinds | Metadata, not part of identity |
| `Meter.Id` | Identity = name + tags | `equals`/`hashCode` ignore type, description, and baseUnit |

**What we built:** The complete identity and tagging system that every meter, registry, and exporter will depend on. A `Meter.Id` is the key to the metrics universe — it tells you *what* is being measured and *along which dimensions*.

**Next chapter: Meter Interface & Clock** — We will add `getId()` and `measure()` methods to the `Meter` interface, introduce `Clock` for time-based meters, and build the abstract base that Counter, Gauge, and Timer will extend.
