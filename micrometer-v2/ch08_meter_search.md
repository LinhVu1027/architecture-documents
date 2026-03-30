# Chapter 8: Meter Search & Discovery

> **Feature 8** · Tier 2 · Depends on: Feature 1 (MeterRegistry, meter storage)

After this chapter you will be able to **query registered meters by name and tags** —
useful for test assertions, building dashboards, or introspecting what's been registered
in a registry at runtime.

---

## 1. The API Contract

Before writing any implementation, let's define exactly what a client sees.

### What the client imports

```java
import simple.micrometer.api.Search;
import simple.micrometer.api.RequiredSearch;
import simple.micrometer.api.MeterNotFoundException;
import simple.micrometer.api.MeterRegistry;
import simple.micrometer.api.Counter;
```

### What the client writes

```java
MeterRegistry registry = new SimpleMeterRegistry();
registry.counter("http.requests", "method", "GET").increment(10);
registry.counter("http.requests", "method", "POST").increment(5);
registry.timer("http.duration", "method", "GET");

// Find meters by name and tags (returns null if not found)
Counter getCounter = registry.find("http.requests")
    .tag("method", "GET")
    .counter();
// getCounter.count() == 10.0

// Get all meters matching a name
Collection<Meter> meters = registry.find("http.requests").meters();

// Required search — throws MeterNotFoundException if not found
Counter required = registry.get("http.requests")
    .tag("method", "GET")
    .counter();
```

### Behavioral contract

| Promise | Detail |
|---------|--------|
| Name filtering | `find("name")` / `get("name")` restricts to meters with that exact name |
| Tag filtering | `.tag("key", "value")` requires meters to have that exact tag |
| Multi-tag filtering | `.tags("k1", "v1", "k2", "v2")` requires ALL tags present |
| Tag key filtering | `.tagKeys("k1", "k2")` requires keys to exist (values unconstrained) |
| Lenient search | `find()` returns null (single) or empty collection (multi) on miss |
| Required search | `get()` throws `MeterNotFoundException` on miss |
| Type-safe terminals | `.counter()` only returns Counter instances, `.timer()` only Timer, etc. |
| Linear scan | Search iterates `getMeters()` — no secondary index, no caching |

---

## 2. Client Usage & Tests

These tests define the API's behavior — they were written **before** any implementation.

### Find a counter by name and tag

```java
@Test
void shouldFindCounterByNameAndTag() {
    registry.counter("http.requests", "method", "GET").increment(10);
    registry.counter("http.requests", "method", "POST").increment(5);

    Counter getCounter = registry.find("http.requests")
            .tag("method", "GET")
            .counter();

    assertThat(getCounter).isNotNull();
    assertThat(getCounter.count()).isEqualTo(10.0);
}
```

### Return null when not found (lenient)

```java
@Test
void shouldReturnNullWhenCounterNotFound() {
    Counter found = registry.find("nonexistent").counter();

    assertThat(found).isNull();
}
```

### Return null when type doesn't match

```java
@Test
void shouldReturnNullWhenNameMatchesButTypeDoesNot() {
    registry.counter("http.requests", "method", "GET");

    Timer found = registry.find("http.requests").timer();

    assertThat(found).isNull();
}
```

### Find all meters matching a name

```java
@Test
void shouldFindAllMetersMatchingName() {
    registry.counter("http.requests", "method", "GET").increment(10);
    registry.counter("http.requests", "method", "POST").increment(5);
    registry.timer("http.duration", "method", "GET");

    Collection<Meter> meters = registry.find("http.requests").meters();

    assertThat(meters).hasSize(2);
}
```

### Required search throws on miss

```java
@Test
void shouldThrowWhenCounterNotFound() {
    assertThatThrownBy(() -> registry.get("nonexistent").counter())
            .isInstanceOf(MeterNotFoundException.class)
            .hasMessageContaining("nonexistent");
}
```

### Required search throws with descriptive message

```java
@Test
void shouldThrowWithDescriptiveMessage() {
    registry.counter("http.requests", "method", "GET");

    assertThatThrownBy(() ->
            registry.get("http.requests")
                    .tag("method", "PUT")
                    .counter())
            .isInstanceOf(MeterNotFoundException.class)
            .hasMessageContaining("http.requests")
            .hasMessageContaining("Counter")
            .hasMessageContaining("method");
}
```

### Filter by tag keys

```java
@Test
void shouldFilterByTagKeys() {
    registry.counter("http.requests", "method", "GET", "region", "us-east");
    registry.counter("http.requests", "method", "POST");

    Collection<Counter> counters = Search.in(registry)
            .name("http.requests")
            .tagKeys("method", "region")
            .counters();

    assertThat(counters).hasSize(1);
}
```

---

## 3. Implementing the Call Chain

The search feature has a thin call chain — it's purely API-layer with no dispatch or
infrastructure layers:

```
Client calls: registry.find("http.requests").tag("method", "GET").counter()
  │
  ├─► [API Layer] MeterRegistry.find("http.requests")
  │     Creates Search.in(this).name("http.requests")
  │
  ├─► [API Layer] Search.tag("method", "GET")
  │     Adds Tag("method", "GET") to requiredTags list
  │
  └─► [API Layer] Search.counter()
        Calls meterStream() → filters by name → filters by tags
        → filters by Counter.class::isInstance → findAny → cast or null
```

For `RequiredSearch`, the only difference is the terminal:

```
Client calls: registry.get("http.requests").tag("method", "GET").counter()
  │
  └─► [API Layer] RequiredSearch.counter()
        Same filtering as Search
        → orElseThrow(MeterNotFoundException)
```

### Layer 1: MeterRegistry — the entry points

Two new methods on `MeterRegistry` provide the fluent entry points:

```java
public Search find(String name) {
    return Search.in(this).name(name);
}

public RequiredSearch get(String name) {
    return RequiredSearch.in(this).name(name);
}
```

These are convenience methods — clients could also use `Search.in(registry)` directly.
The `find(name)` / `get(name)` pattern is more readable and matches the real framework's
API.

### Layer 2: Search — the lenient query builder

The core of Search is the `meterStream()` method, which builds a filtering pipeline
over the registry's meter list:

```java
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
```

Terminal methods then add a type filter and choose between single vs. collection results:

```java
// Single result: null on miss
private <M extends Meter> M findOne(Class<M> clazz) {
    return meterStream()
            .filter(clazz::isInstance)
            .findAny()
            .map(clazz::cast)
            .orElse(null);
}

// Collection result: empty on miss
private <M extends Meter> Collection<M> findAll(Class<M> clazz) {
    return meterStream()
            .filter(clazz::isInstance)
            .map(clazz::cast)
            .collect(Collectors.toList());
}
```

### Layer 3: RequiredSearch — the strict query builder

RequiredSearch has the **exact same** filtering pipeline as Search (it's an independent
implementation, not a wrapper). The only difference is the terminal methods:

```java
// Single result: throws on miss
private <M extends Meter> M getOne(Class<M> clazz) {
    return meterStream()
            .filter(clazz::isInstance)
            .findAny()
            .map(clazz::cast)
            .orElseThrow(() -> buildException(clazz));
}
```

The `buildException` method creates a descriptive `MeterNotFoundException` that includes
what was searched for and what meters are actually in the registry — essential for
debugging test failures:

```
MeterNotFoundException: No Counter found with name 'http.requests' with tags
[Tag{key='method', value='PUT'}] in the registry. Available meters:
http.requests[Tag{key='method', value='GET'}]
```

### Layer 4: MeterNotFoundException — the diagnostic exception

A simple `RuntimeException` subclass. The real framework's version is much more
sophisticated — it re-runs individual search criteria with `Search` to produce
per-criterion OK/FAIL diagnostics. Our simplified version just lists what was
searched for and what's available.

---

## 4. Try It Yourself

1. **Search without name**: Use `Search.in(registry).meters()` to get ALL meters
   in the registry without any name filter. Verify the count matches what you registered.

2. **Tag-only search**: Register several meters with different names but shared tags.
   Search by tag only (no name filter) using `Search.in(registry).tag("env", "prod").meters()`.
   Verify it returns meters across different names.

3. **Mixed type search**: Register a counter, timer, and gauge all named "app.metric"
   (with different tags so they don't collide). Use `registry.find("app.metric").meters()`
   to get all three, then use `.counter()`, `.timer()`, `.gauge()` to get each individually.

4. **Required search in tests**: Write a test that registers metrics, then uses
   `registry.get("name").counter()` to assert the metric exists. Break it on purpose
   (change the name) and observe the descriptive `MeterNotFoundException`.

---

## 5. Why This Works

### Linear scan is acceptable here

The search operates by streaming `registry.getMeters()` — a linear scan with no secondary
index. This seems inefficient, but it's the correct tradeoff for Micrometer:
- Registries rarely hold more than a few hundred meters
- Searches happen during test assertions or dashboard queries, not hot paths
- Adding a secondary index would increase memory and complicate meter registration

The real framework makes the same choice — `Search.meterStream()` does a linear scan.

### Independent implementations (Search vs. RequiredSearch)

Why doesn't RequiredSearch simply wrap Search? Three reasons:
1. **No null boxing**: RequiredSearch can use `orElseThrow()` directly instead of
   checking null return values from Search
2. **Field access**: In the real framework, `MeterNotFoundException` reads RequiredSearch's
   fields directly (package-private) for its diagnostics
3. **API surface differs**: Search supports predicate-based matching and `acceptFilter()`;
   RequiredSearch doesn't (it's for precise, assertion-style lookups)

### Tag filtering uses `containsAll`

The `requiredTags` filter uses `List.containsAll()`, which works because `Tag` implements
`equals()` based on key AND value. A meter with tags `[method=GET, status=200]` will pass
a filter requiring `[method=GET]` because the meter's tag list contains that element.

### `findAny()` not `findFirst()`

Single-meter terminal methods use `findAny()` rather than `findFirst()`. This is
intentional — there's no meaningful ordering guarantee in a `ConcurrentHashMap`, so
`findAny()` avoids the overhead of preserving encounter order in the stream.

---

## 6. What We Enhanced

| Enhancement | Description | Enabled by |
|-------------|-------------|------------|
| Name-based lookup | Find meters by exact name without iterating manually | Search.name() / RequiredSearch.name() |
| Tag filtering | Filter by exact key+value pairs or key-only | Search.tag() / tags() / tagKeys() |
| Type-safe terminals | Get counters, timers, gauges, summaries as their specific types | Search.counter(), timer(), etc. |
| Lenient vs. strict | Choose between null-on-miss and throw-on-miss semantics | Search vs. RequiredSearch |
| Registry entry points | Fluent `registry.find()` / `registry.get()` shortcuts | MeterRegistry.find() / get() |
| Descriptive errors | MeterNotFoundException lists what was searched for and what's available | RequiredSearch.buildException() |

---

## 7. Connection to Real Framework

| Simplified Class | Real Framework Class | Key Simplifications |
|------------------|---------------------|---------------------|
| `Search` | `io.micrometer.core.instrument.search.Search` | No predicate-based name matching, no predicate tag value matching, no `acceptFilter()` conversion |
| `RequiredSearch` | `io.micrometer.core.instrument.search.RequiredSearch` | No predicate-based matching, simplified exception diagnostics |
| `MeterNotFoundException` | `io.micrometer.core.instrument.search.MeterNotFoundException` | Simple message instead of per-criterion OK/FAIL diagnostics |

**What the real framework adds:**
- **`name(Predicate<String>)`**: Search by name pattern, not just exact match. Useful
  for finding all meters starting with "jvm." or matching a regex.
- **`tag(String key, Predicate<String> valueMatches)`**: Filter by tag value pattern —
  e.g., find all meters where the "status" tag starts with "5" (server errors).
- **`acceptFilter()`**: Convert a Search's criteria into a `MeterFilter` that can be
  added to a registry's filter chain — reusing query definitions as filters.
- **Per-criterion diagnostics**: `MeterNotFoundException` re-runs individual criteria
  using `Search` to report which specific condition (name? type? tag?) failed and
  what values were available. This produces messages like:
  ```
  FAIL: name = 'http.requests' (found 2 meters)
  FAIL: type = Timer (found 0, available types: Counter)
  OK: tag 'method' = 'GET' (found 1 meter)
  ```
- **`tagKeys(Collection)`**: An overload accepting a Collection (since 1.7.0).

---

## 8. Complete Code

All files created or modified in this feature. Copy all `[NEW]` files and replace all
`[MODIFIED]` files to get a compiling project with all tests passing.

### `src/main/java/simple/micrometer/api/Search.java` [NEW]

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
            throw new IllegalArgumentException(
                    "keyValues must have even length, got " + keyValues.length);
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

### `src/main/java/simple/micrometer/api/MeterNotFoundException.java` [NEW]

```java
package simple.micrometer.api;

/**
 * Thrown by {@link RequiredSearch} when no meter matches the search criteria.
 * Contains a descriptive message indicating what was searched for and what
 * type of meter was expected.
 *
 * <p>Simplified from {@code io.micrometer.core.instrument.search.MeterNotFoundException}.
 * The real framework includes per-criterion OK/FAIL diagnostics to help
 * pinpoint which search condition failed.
 */
public class MeterNotFoundException extends RuntimeException {

    public MeterNotFoundException(String message) {
        super(message);
    }

}
```

### `src/main/java/simple/micrometer/api/RequiredSearch.java` [NEW]

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
            throw new IllegalArgumentException(
                    "keyValues must have even length, got " + keyValues.length);
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
        return requireNonEmpty(findAll(DistributionSummary.class),
                DistributionSummary.class);
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

    private <M extends Meter> Collection<M> requireNonEmpty(
            Collection<M> result, Class<M> clazz) {
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
    private MeterNotFoundException buildException(
            Class<? extends Meter> expectedType) {
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

### `src/main/java/simple/micrometer/api/MeterRegistry.java` [MODIFIED]

Two new methods added to the Query section:

```java
// ── Query ────────────────────────────────────────────────────────

public List<Meter> getMeters() {
    return Collections.unmodifiableList(new ArrayList<>(meterMap.values()));
}

// NEW: Lenient search — returns null/empty on miss
public Search find(String name) {
    return Search.in(this).name(name);
}

// NEW: Required search — throws MeterNotFoundException on miss
public RequiredSearch get(String name) {
    return RequiredSearch.in(this).name(name);
}

public Clock getClock() {
    return clock;
}
```

### `src/test/java/simple/micrometer/api/SearchTest.java` [NEW]

```java
package simple.micrometer.api;

import simple.micrometer.internal.SimpleMeterRegistry;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Nested;
import org.junit.jupiter.api.Test;

import java.time.Duration;
import java.util.Collection;

import static org.assertj.core.api.Assertions.*;

/**
 * Client-perspective tests for {@link Search} and {@link RequiredSearch}.
 * These tests demonstrate how a client queries registered meters by name
 * and tags — useful for test assertions, building dashboards, or
 * introspecting what's been registered.
 */
class SearchTest {

    MeterRegistry registry;

    @BeforeEach
    void setUp() {
        registry = new SimpleMeterRegistry();
    }

    // ── Search (lenient — returns null/empty on miss) ──────────────────

    @Nested
    class LenientSearchTests {

        @Test
        void shouldFindCounterByName() {
            registry.counter("http.requests", "method", "GET").increment(10);

            Counter found = registry.find("http.requests").counter();

            assertThat(found).isNotNull();
            assertThat(found.count()).isEqualTo(10.0);
        }

        @Test
        void shouldFindCounterByNameAndTag() {
            registry.counter("http.requests", "method", "GET").increment(10);
            registry.counter("http.requests", "method", "POST").increment(5);

            Counter getCounter = registry.find("http.requests")
                    .tag("method", "GET")
                    .counter();

            assertThat(getCounter).isNotNull();
            assertThat(getCounter.count()).isEqualTo(10.0);
        }

        @Test
        void shouldReturnNullWhenCounterNotFound() {
            Counter found = registry.find("nonexistent").counter();

            assertThat(found).isNull();
        }

        @Test
        void shouldReturnNullWhenNameMatchesButTypeDoesNot() {
            registry.counter("http.requests", "method", "GET");

            Timer found = registry.find("http.requests").timer();

            assertThat(found).isNull();
        }

        @Test
        void shouldFindTimerByName() {
            Timer timer = registry.timer("http.duration", "method", "GET");
            timer.record(Duration.ofMillis(250));

            Timer found = registry.find("http.duration").timer();

            assertThat(found).isNotNull();
            assertThat(found.count()).isEqualTo(1);
        }

        @Test
        void shouldFindGaugeByName() {
            java.util.concurrent.atomic.AtomicInteger connections =
                    new java.util.concurrent.atomic.AtomicInteger(42);
            Gauge.builder("active.connections", () -> connections)
                    .register(registry);

            Gauge found = registry.find("active.connections").gauge();

            assertThat(found).isNotNull();
            assertThat(found.value()).isEqualTo(42.0);
        }

        @Test
        void shouldFindDistributionSummaryByName() {
            DistributionSummary summary =
                    registry.summary("payload.size", "uri", "/api");
            summary.record(1024);

            DistributionSummary found = registry.find("payload.size")
                    .tag("uri", "/api")
                    .summary();

            assertThat(found).isNotNull();
            assertThat(found.totalAmount()).isEqualTo(1024.0);
        }

        @Test
        void shouldFindAllMetersMatchingName() {
            registry.counter("http.requests", "method", "GET").increment(10);
            registry.counter("http.requests", "method", "POST").increment(5);
            registry.timer("http.duration", "method", "GET");

            Collection<Meter> meters =
                    registry.find("http.requests").meters();

            assertThat(meters).hasSize(2);
        }

        @Test
        void shouldReturnEmptyCollectionWhenNoMetersMatch() {
            Collection<Meter> meters =
                    registry.find("nonexistent").meters();

            assertThat(meters).isEmpty();
        }

        @Test
        void shouldFindAllCountersMatchingName() {
            registry.counter("http.requests", "method", "GET");
            registry.counter("http.requests", "method", "POST");
            registry.timer("http.requests.duration");

            Collection<Counter> counters =
                    registry.find("http.requests").counters();

            assertThat(counters).hasSize(2);
        }

        @Test
        void shouldFilterByMultipleTagsUsingVarargs() {
            registry.counter("http.requests",
                    "method", "GET", "status", "200").increment(10);
            registry.counter("http.requests",
                    "method", "GET", "status", "500").increment(3);

            Counter found = Search.in(registry)
                    .name("http.requests")
                    .tags("method", "GET", "status", "200")
                    .counter();

            assertThat(found).isNotNull();
            assertThat(found.count()).isEqualTo(10.0);
        }

        @Test
        void shouldFilterByTagKeys() {
            registry.counter("http.requests",
                    "method", "GET", "region", "us-east");
            registry.counter("http.requests", "method", "POST");

            Collection<Counter> counters = Search.in(registry)
                    .name("http.requests")
                    .tagKeys("method", "region")
                    .counters();

            assertThat(counters).hasSize(1);
        }

        @Test
        void shouldFindAnyMeterByName() {
            registry.counter("my.metric");
            registry.timer("my.metric.timer");

            Meter found = registry.find("my.metric").meter();

            assertThat(found).isNotNull();
            assertThat(found.getId().getName()).isEqualTo("my.metric");
        }

        @Test
        void shouldReturnAllTimersMatchingCriteria() {
            registry.timer("http.duration", "method", "GET");
            registry.timer("http.duration", "method", "POST");

            Collection<Timer> timers =
                    registry.find("http.duration").timers();

            assertThat(timers).hasSize(2);
        }

        @Test
        void shouldReturnAllGaugesMatchingCriteria() {
            java.util.concurrent.atomic.AtomicInteger v1 =
                    new java.util.concurrent.atomic.AtomicInteger(1);
            java.util.concurrent.atomic.AtomicInteger v2 =
                    new java.util.concurrent.atomic.AtomicInteger(2);
            Gauge.builder("pool.size", () -> v1)
                    .tag("pool", "a").register(registry);
            Gauge.builder("pool.size", () -> v2)
                    .tag("pool", "b").register(registry);

            Collection<Gauge> gauges =
                    registry.find("pool.size").gauges();

            assertThat(gauges).hasSize(2);
        }

        @Test
        void shouldReturnAllSummariesMatchingCriteria() {
            registry.summary("response.size", "uri", "/a");
            registry.summary("response.size", "uri", "/b");

            Collection<DistributionSummary> summaries =
                    registry.find("response.size").summaries();

            assertThat(summaries).hasSize(2);
        }

        @Test
        void shouldWorkWithSearchInStaticEntryPoint() {
            registry.counter("cache.hits", "cache", "users").increment(100);

            Counter found = Search.in(registry)
                    .name("cache.hits")
                    .tag("cache", "users")
                    .counter();

            assertThat(found).isNotNull();
            assertThat(found.count()).isEqualTo(100.0);
        }

        @Test
        void shouldFindMetersWithoutNameFilter() {
            registry.counter("a.counter");
            registry.timer("b.timer");
            registry.summary("c.summary");

            Collection<Meter> all = Search.in(registry).meters();

            assertThat(all).hasSize(3);
        }

        @Test
        void shouldFilterByTagsUsingIterable() {
            registry.counter("http.requests",
                    "method", "GET", "status", "200").increment();

            Counter found = Search.in(registry)
                    .name("http.requests")
                    .tags(Tags.of("method", "GET", "status", "200"))
                    .counter();

            assertThat(found).isNotNull();
        }
    }

    // ── RequiredSearch (strict — throws on miss) ──────────────────────

    @Nested
    class RequiredSearchTests {

        @Test
        void shouldFindCounterOrThrow() {
            registry.counter("http.requests", "method", "GET").increment(10);

            Counter found = registry.get("http.requests")
                    .tag("method", "GET")
                    .counter();

            assertThat(found.count()).isEqualTo(10.0);
        }

        @Test
        void shouldThrowWhenCounterNotFound() {
            assertThatThrownBy(() ->
                    registry.get("nonexistent").counter())
                    .isInstanceOf(MeterNotFoundException.class)
                    .hasMessageContaining("nonexistent");
        }

        @Test
        void shouldThrowWhenNameMatchesButTypeDoesNot() {
            registry.counter("http.requests");

            assertThatThrownBy(() ->
                    registry.get("http.requests").timer())
                    .isInstanceOf(MeterNotFoundException.class)
                    .hasMessageContaining("Timer");
        }

        @Test
        void shouldThrowWhenTagDoesNotMatch() {
            registry.counter("http.requests", "method", "GET");

            assertThatThrownBy(() ->
                    registry.get("http.requests")
                            .tag("method", "DELETE")
                            .counter())
                    .isInstanceOf(MeterNotFoundException.class);
        }

        @Test
        void shouldReturnAllMetersOrThrow() {
            registry.counter("http.requests", "method", "GET");
            registry.counter("http.requests", "method", "POST");

            Collection<Meter> meters =
                    registry.get("http.requests").meters();

            assertThat(meters).hasSize(2);
        }

        @Test
        void shouldThrowWhenMetersCollectionWouldBeEmpty() {
            assertThatThrownBy(() ->
                    registry.get("nonexistent").meters())
                    .isInstanceOf(MeterNotFoundException.class);
        }

        @Test
        void shouldFindTimerOrThrow() {
            registry.timer("db.query.time", "table", "users");

            Timer found = registry.get("db.query.time")
                    .tag("table", "users")
                    .timer();

            assertThat(found).isNotNull();
        }

        @Test
        void shouldFindGaugeOrThrow() {
            java.util.concurrent.atomic.AtomicInteger value =
                    new java.util.concurrent.atomic.AtomicInteger(99);
            Gauge.builder("active.threads", () -> value)
                    .register(registry);

            Gauge found = registry.get("active.threads").gauge();

            assertThat(found.value()).isEqualTo(99.0);
        }

        @Test
        void shouldFindSummaryOrThrow() {
            registry.summary("batch.size", "job", "import").record(500);

            DistributionSummary found = registry.get("batch.size")
                    .tag("job", "import")
                    .summary();

            assertThat(found.totalAmount()).isEqualTo(500.0);
        }

        @Test
        void shouldThrowWithDescriptiveMessage() {
            registry.counter("http.requests", "method", "GET");

            assertThatThrownBy(() ->
                    registry.get("http.requests")
                            .tag("method", "PUT")
                            .counter())
                    .isInstanceOf(MeterNotFoundException.class)
                    .hasMessageContaining("http.requests")
                    .hasMessageContaining("Counter")
                    .hasMessageContaining("method");
        }

        @Test
        void shouldWorkWithRequiredSearchInStaticEntryPoint() {
            registry.counter("cache.hits").increment(50);

            Counter found = RequiredSearch.in(registry)
                    .name("cache.hits")
                    .counter();

            assertThat(found.count()).isEqualTo(50.0);
        }

        @Test
        void shouldFindAllCountersOrThrow() {
            registry.counter("ops", "type", "read");
            registry.counter("ops", "type", "write");

            Collection<Counter> counters =
                    registry.get("ops").counters();

            assertThat(counters).hasSize(2);
        }

        @Test
        void shouldThrowWhenCountersCollectionEmpty() {
            registry.timer("ops");

            assertThatThrownBy(() ->
                    registry.get("ops").counters())
                    .isInstanceOf(MeterNotFoundException.class);
        }

        @Test
        void shouldFindMeterOfAnyType() {
            registry.counter("my.metric");

            Meter found = registry.get("my.metric").meter();

            assertThat(found.getId().getName()).isEqualTo("my.metric");
        }

        @Test
        void shouldFilterByVarargsTags() {
            registry.counter("http.requests",
                    "method", "GET", "status", "200").increment();

            Counter found = RequiredSearch.in(registry)
                    .name("http.requests")
                    .tags("method", "GET", "status", "200")
                    .counter();

            assertThat(found).isNotNull();
        }

        @Test
        void shouldFilterByTagKeys() {
            registry.counter("http.requests",
                    "method", "GET", "region", "us-east");
            registry.counter("http.requests", "method", "POST");

            Collection<Counter> counters = RequiredSearch.in(registry)
                    .name("http.requests")
                    .tagKeys("region")
                    .counters();

            assertThat(counters).hasSize(1);
        }

        @Test
        void shouldReturnAllTimersOrThrow() {
            registry.timer("db.query", "table", "users");
            registry.timer("db.query", "table", "orders");

            Collection<Timer> timers =
                    registry.get("db.query").timers();

            assertThat(timers).hasSize(2);
        }

        @Test
        void shouldReturnAllGaugesOrThrow() {
            java.util.concurrent.atomic.AtomicInteger v1 =
                    new java.util.concurrent.atomic.AtomicInteger(1);
            java.util.concurrent.atomic.AtomicInteger v2 =
                    new java.util.concurrent.atomic.AtomicInteger(2);
            Gauge.builder("pool.size", () -> v1)
                    .tag("pool", "a").register(registry);
            Gauge.builder("pool.size", () -> v2)
                    .tag("pool", "b").register(registry);

            Collection<Gauge> gauges =
                    registry.get("pool.size").gauges();

            assertThat(gauges).hasSize(2);
        }

        @Test
        void shouldReturnAllSummariesOrThrow() {
            registry.summary("response.size", "uri", "/a").record(100);
            registry.summary("response.size", "uri", "/b").record(200);

            Collection<DistributionSummary> summaries =
                    registry.get("response.size").summaries();

            assertThat(summaries).hasSize(2);
        }
    }

}
```
