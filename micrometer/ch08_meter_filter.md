# Chapter 8: MeterFilter

> Line references based on commit `2c8a4606c` of the Micrometer repository.

## Build Challenge

| Current State | Limitation | Objective |
|---------------|------------|-----------|
| MeterRegistry creates and stores any meter that's registered — all meters are accepted unconditionally | Cannot add common tags (like `env=prod`) to all meters, cannot block unwanted metrics, cannot rename tag keys — every meter passes straight through to the registry unchanged | Build a filter chain that transforms, accepts/denies, and controls meter registration before the meter is created |

---

## 8.1 The Integration Point

The filter chain plugs into `MeterRegistry.registerMeterIfNecessary()` — the single method that every meter registration (Counter, Gauge, Timer, DistributionSummary) flows through. This is where we intercept the ID **before** deduplication and creation.

**Modifying:** `src/main/java/dev/linhvu/micrometer/MeterRegistry.java`
**Change:** Add filter storage, `meterFilter()` method, and two-stage pipeline (map → accept) inside `registerMeterIfNecessary()`

```java
/**
 * Volatile array of filters — enables lock-free iteration during meter registration
 * while still allowing thread-safe mutation via {@link #meterFilter(MeterFilter)}.
 * Copy-on-write: adding a filter creates a new array.
 */
private volatile MeterFilter[] filters = new MeterFilter[0];

public synchronized MeterRegistry meterFilter(MeterFilter filter) {
    Objects.requireNonNull(filter, "filter must not be null");
    MeterFilter[] newFilters = Arrays.copyOf(filters, filters.length + 1);
    newFilters[filters.length] = filter;
    filters = newFilters;
    return this;
}
```

The registration method now runs two stages before hitting `getOrCreateMeter()`:

```java
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
    // ... type checking unchanged ...
}
```

Two key decisions here:

1. **Why volatile array instead of `CopyOnWriteArrayList`?** The volatile array enables lock-free reads on the hot path (every meter registration iterates filters) while the `synchronized` add method handles the rare mutation path. This is the same pattern the real Micrometer uses at `MeterRegistry.java:90`.

2. **Why does map run before accept?** A filter might add a tag (via `commonTags`) that another filter uses in its accept decision. Running map first ensures accept filters see the fully-transformed ID.

This connects the **filter chain** to the **registration pipeline**. To make it work, we need to build:
- `MeterFilterReply` — the three-way decision enum (DENY/NEUTRAL/ACCEPT)
- `MeterFilter` — the interface with `map()` and `accept()` default methods plus built-in static factories
- `mapId()` and `accept()` — the pipeline methods in MeterRegistry

```
Builder.register(registry)
    │
    ▼
registerMeterIfNecessary(id)
    │
    ├──> mapId(id)              ← Stage 1: transform (all filters, in order)
    │    ├── filter[0].map(id)
    │    ├── filter[1].map(id')
    │    └── filter[N].map(id'')
    │
    ├──> accept(mappedId)       ← Stage 2: accept/deny (short-circuits)
    │    ├── filter[0].accept(mappedId) → DENY? return noop
    │    ├── filter[1].accept(mappedId) → ACCEPT? proceed
    │    └── all NEUTRAL? → accept by default
    │
    └──> getOrCreateMeter(mappedId)  ← unchanged deduplication
```

## 8.2 MeterFilterReply

The accept decision needs three states — not just a boolean. A filter that doesn't care should defer to the next filter, not force-accept.

**New file:** `src/main/java/dev/linhvu/micrometer/config/MeterFilterReply.java`

```java
package dev.linhvu.micrometer.config;

public enum MeterFilterReply {

    DENY,

    NEUTRAL,

    ACCEPT

}
```

- **DENY** — exclude the meter; a no-op meter is returned to the caller
- **NEUTRAL** — this filter has no opinion; defer to the next filter in the chain
- **ACCEPT** — include the meter; short-circuits the chain

If every filter returns NEUTRAL, the meter is accepted by default.

## 8.3 MeterFilter Interface

The core interface has two default methods and a rich set of static factory methods.

**New file:** `src/main/java/dev/linhvu/micrometer/config/MeterFilter.java`

```java
package dev.linhvu.micrometer.config;

import dev.linhvu.micrometer.Meter;
import dev.linhvu.micrometer.Tag;
import dev.linhvu.micrometer.Tags;

import java.util.ArrayList;
import java.util.List;
import java.util.Set;
import java.util.concurrent.ConcurrentHashMap;
import java.util.function.Predicate;

public interface MeterFilter {

    default Meter.Id map(Meter.Id id) {
        return id;
    }

    default MeterFilterReply accept(Meter.Id id) {
        return MeterFilterReply.NEUTRAL;
    }

    // --- Tag manipulation (override map) ---

    static MeterFilter commonTags(Iterable<Tag> tags) {
        return new MeterFilter() {
            @Override
            public Meter.Id map(Meter.Id id) {
                Tags merged = Tags.of(tags).and(id.getTagsAsIterable());
                return id.replaceTags(merged);
            }
        };
    }

    static MeterFilter renameTag(String meterNamePrefix, String fromTagKey, String toTagKey) {
        return new MeterFilter() {
            @Override
            public Meter.Id map(Meter.Id id) {
                if (!id.getName().startsWith(meterNamePrefix)) {
                    return id;
                }
                List<Tag> newTags = new ArrayList<>();
                for (Tag tag : id.getTagsAsIterable()) {
                    if (tag.getKey().equals(fromTagKey)) {
                        newTags.add(Tag.of(toTagKey, tag.getValue()));
                    } else {
                        newTags.add(tag);
                    }
                }
                return id.replaceTags(newTags);
            }
        };
    }

    static MeterFilter ignoreTags(String... tagKeys) {
        Set<String> ignoreSet = Set.of(tagKeys);
        return new MeterFilter() {
            @Override
            public Meter.Id map(Meter.Id id) {
                List<Tag> filtered = new ArrayList<>();
                for (Tag tag : id.getTagsAsIterable()) {
                    if (!ignoreSet.contains(tag.getKey())) {
                        filtered.add(tag);
                    }
                }
                return id.replaceTags(filtered);
            }
        };
    }

    // --- Accept/deny (override accept) ---

    static MeterFilter accept(Predicate<Meter.Id> predicate) { /* ... */ }
    static MeterFilter accept() { return accept(id -> true); }
    static MeterFilter deny(Predicate<Meter.Id> predicate) { /* ... */ }
    static MeterFilter deny() { return deny(id -> true); }
    static MeterFilter denyUnless(Predicate<Meter.Id> predicate) { /* ... */ }
    static MeterFilter acceptNameStartsWith(String prefix) { /* ... */ }
    static MeterFilter denyNameStartsWith(String prefix) { /* ... */ }

    // --- Cardinality control (stateful) ---

    static MeterFilter maximumAllowableMetrics(int maximumMetrics) { /* ... */ }
}
```

Key factories:

| Factory | Overrides | Behavior |
|---------|-----------|----------|
| `commonTags(tags)` | `map` | Adds tags to all meters; meter's own tags win on conflicts |
| `renameTag(prefix, from, to)` | `map` | Renames a tag key on meters matching the prefix |
| `ignoreTags(keys...)` | `map` | Removes specified tag keys from all meters |
| `accept(predicate)` | `accept` | Returns ACCEPT when predicate matches, NEUTRAL otherwise |
| `deny(predicate)` | `accept` | Returns DENY when predicate matches, NEUTRAL otherwise |
| `denyUnless(predicate)` | `accept` | Whitelist: DENY unless predicate matches |
| `maximumAllowableMetrics(n)` | `accept` | Global cap on unique meter count |

## 8.4 The Pipeline Methods

Back in `MeterRegistry`, the two private methods that drive the filter chain:

**Modifying:** `src/main/java/dev/linhvu/micrometer/MeterRegistry.java`
**Change:** Add `mapId()` and `accept()` pipeline methods

```java
private Meter.Id mapId(Meter.Id id) {
    Meter.Id mappedId = id;
    for (MeterFilter filter : filters) {
        mappedId = filter.map(mappedId);
    }
    return mappedId;
}

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
    return true; // all NEUTRAL → accept by default
}
```

## 8.5 Try It Yourself

<details>
<summary>Challenge 1: Implement a filter that adds a "version" tag to all meters</summary>

Using the `MeterFilter` interface, create a filter that adds `version=2.0` to every meter:

```java
MeterFilter versionFilter = new MeterFilter() {
    @Override
    public Meter.Id map(Meter.Id id) {
        return id.withTag(Tag.of("version", "2.0"));
    }
};

registry.meterFilter(versionFilter);
Counter counter = registry.counter("requests");
assertThat(counter.getId().getTag("version")).isEqualTo("2.0");
```

</details>

<details>
<summary>Challenge 2: Implement a whitelist that only allows "http" and "db" metrics</summary>

Use `denyUnless` with a compound predicate:

```java
registry.meterFilter(MeterFilter.denyUnless(
    id -> id.getName().startsWith("http.") || id.getName().startsWith("db.")
));

Counter http = registry.counter("http.requests");   // real counter
Counter db = registry.counter("db.queries");         // real counter
Counter jvm = registry.counter("jvm.memory");        // NoopCounter

assertThat(http).isNotInstanceOf(NoopCounter.class);
assertThat(db).isNotInstanceOf(NoopCounter.class);
assertThat(jvm).isInstanceOf(NoopCounter.class);
```

</details>

<details>
<summary>Challenge 3: Combine map and deny — add common tags, then deny based on tag presence</summary>

Create a pipeline where:
1. A `commonTags` filter adds `env=prod` to all meters
2. A custom filter denies meters where `env` is NOT `prod`

Since map runs before accept, the common tag is visible to the deny filter:

```java
registry.meterFilter(MeterFilter.commonTags(Tags.of("env", "prod")));
registry.meterFilter(MeterFilter.deny(id -> !"prod".equals(id.getTag("env"))));

Counter counter = registry.counter("requests");
assertThat(counter).isNotInstanceOf(NoopCounter.class); // accepted — env=prod was added
```

</details>

## 8.6 Tests

### Unit Tests

**New file:** `src/test/java/dev/linhvu/micrometer/config/MeterFilterReplyTest.java`

```java
class MeterFilterReplyTest {
    @Test
    void shouldHaveThreeValues() {
        assertThat(MeterFilterReply.values()).hasSize(3);
    }

    @Test
    void shouldHaveExpectedOrdering() {
        MeterFilterReply[] values = MeterFilterReply.values();
        assertThat(values[0]).isEqualTo(MeterFilterReply.DENY);
        assertThat(values[1]).isEqualTo(MeterFilterReply.NEUTRAL);
        assertThat(values[2]).isEqualTo(MeterFilterReply.ACCEPT);
    }
}
```

**New file:** `src/test/java/dev/linhvu/micrometer/config/MeterFilterTest.java`

Tests each static factory in isolation — `commonTags`, `renameTag`, `ignoreTags`, `accept`, `deny`, `denyUnless`, `acceptNameStartsWith`, `denyNameStartsWith`, `maximumAllowableMetrics`.

Key behaviors verified:
- `shouldNotOverrideExistingTag_WhenCommonTagHasSameKey` — meter's own tags take precedence
- `shouldNotRenameTag_WhenNameDoesNotMatchPrefix` — prefix matching scopes the rename
- `shouldAllowAlreadySeenMetric_WhenOverLimit` — cardinality cap doesn't block existing meters

### Integration Tests

**New file:** `src/test/java/dev/linhvu/micrometer/integration/MeterFilterIntegrationTest.java`

Tests the full pipeline: filter → registry → meter creation across all meter types:
- Common tags applied to Counter, Timer, Gauge, DistributionSummary
- Deny returns noop for all meter types
- Filter chain ordering: short-circuit on DENY/ACCEPT
- `denyUnless` whitelist pattern
- `maximumAllowableMetrics` cardinality cap
- Noop meters still execute wrapped business logic
- Deduplication works correctly after ID mapping

**Run:** `./gradlew test` — expected: all tests pass (including prior features' tests)

---

## 8.7 Why This Works

> ★ **Insight** -------------------------------------------
> - **Why Chain of Responsibility over a single filter?** Multiple filters compose cleanly — you can add common tags in one filter, deny internal metrics in another, and cap cardinality in a third. Each filter is a single-responsibility unit. Adding a new concern doesn't require modifying existing filters.
> - **When NOT to use it:** If you only need one filter, the chain adds unnecessary iteration. But in practice, real applications typically have 3-5 filters (common tags + deny lists + cardinality guards), making the chain worthwhile.
> - **Real-world parallel:** Micrometer's filter chain is directly analogous to servlet filters and Spring interceptors — the same pattern of sequential processing with short-circuit semantics.
> -----------------------------------------------------------

> ★ **Insight** -------------------------------------------
> - **Why volatile array instead of a List?** The volatile array gives lock-free reads on the hot path (every `counter.increment()` triggers filter iteration) with synchronized writes on the cold path (filters are typically configured once at startup). The `CopyOnWriteArrayList` would work too, but the raw array avoids iterator allocation.
> - **Trade-off:** This makes the filter array immutable from the reader's perspective — readers see a consistent snapshot. But it means adding a filter after meters are registered doesn't retroactively transform existing meters (only new registrations see the new filter). The real Micrometer warns about this at `MeterRegistry.java:911`.
> -----------------------------------------------------------

> ★ **Insight** -------------------------------------------
> - **Why three-value enum (DENY/NEUTRAL/ACCEPT) instead of boolean?** NEUTRAL is crucial — it means "I don't care, ask the next filter." Without it, a filter that doesn't match would have to choose between force-accepting (blocking downstream deny filters) or force-denying (overriding downstream accept filters). The three-way design enables composable whitelist/blacklist patterns like `acceptNameStartsWith("http.") → deny()`, where the first filter short-circuits for matching meters and the second catches everything else.
> -----------------------------------------------------------

## 8.8 What We Enhanced

| Aspect | Before (ch04) | Current (ch08) | Real Framework |
|--------|---------------|----------------|----------------|
| Registration pipeline | ID passes straight from Builder to ConcurrentHashMap — no interception | Two-stage filter chain (map → accept) runs before deduplication | Three-stage pipeline: map → accept → configure (`MeterRegistry.java:677-729`) |
| Tag management | Tags set only by the Builder — no way to add global tags | `commonTags()`, `renameTag()`, `ignoreTags()` transform IDs via filters | Same filters plus `replaceTagValues()` and conditional delegation via `forMeters()` (`MeterFilter.java:122,425`) |
| Meter admission | All meters accepted unconditionally (no way to block) | `deny()`, `accept()`, `denyUnless()`, `denyNameStartsWith()`, `maximumAllowableMetrics()` control admission | Same plus `maximumAllowableTags()` for per-tag-key cardinality limits (`MeterFilter.java:241`) |
| Distribution config | Not configurable via filters | Deferred to Feature 10 | `configure(Id, DistributionStatisticConfig)` sets histograms/percentiles/SLO boundaries (`MeterFilter.java:477`) |

## 8.9 Connection to Real Framework

| Simplified Code | Real Framework Code | File:Line | Key Difference |
|----------------|--------------------|-----------| ---------------|
| `MeterFilter` interface | `MeterFilter` | `MeterFilter.java:45` | Real adds `configure()` method for distribution statistics, `replaceTagValues()`, `maximumAllowableTags()`, and `forMeters()` conditional delegation |
| `MeterFilterReply` enum | `MeterFilterReply` | `MeterFilterReply.java:18` | Identical — same three values |
| `MeterRegistry.filters` | `MeterRegistry.filters` | `MeterRegistry.java:90` | Real uses same volatile array pattern |
| `MeterRegistry.meterFilter()` | `Config.meterFilter()` | `MeterRegistry.java:908` | Real nests this inside a `Config` inner class and warns if meters already exist |
| `mapId()` | `mapId()` | `MeterRegistry.java:677` | Real skips synthetic associations; otherwise identical chaining |
| `accept()` | `accept()` | `MeterRegistry.java:785` | Identical short-circuit logic |
| `commonTags()` | `commonTags()` | `MeterFilter.java:55` | Real also guards against overriding previously configured common tags with the same key |
| `maximumAllowableMetrics()` | `maximumAllowableMetrics()` | `MeterFilter.java:217` | Identical `ConcurrentHashMap.newKeySet()` pattern |

## 8.10 Complete Code

### Production Code

#### File: `src/main/java/dev/linhvu/micrometer/config/MeterFilterReply.java` [NEW]

```java
package dev.linhvu.micrometer.config;

/**
 * The result of a {@link MeterFilter#accept(dev.linhvu.micrometer.Meter.Id)} decision.
 * <p>
 * The filter chain iterates all filters in order. The first non-{@code NEUTRAL}
 * reply wins (short-circuit). If all filters return {@code NEUTRAL}, the meter
 * is accepted.
 *
 * <ul>
 *   <li>{@code DENY} — exclude the meter; a no-op meter is returned to the caller</li>
 *   <li>{@code NEUTRAL} — this filter has no opinion; defer to the next filter</li>
 *   <li>{@code ACCEPT} — include the meter; short-circuits the chain</li>
 * </ul>
 */
public enum MeterFilterReply {

    DENY,

    NEUTRAL,

    ACCEPT

}
```

#### File: `src/main/java/dev/linhvu/micrometer/config/MeterFilter.java` [NEW]

```java
package dev.linhvu.micrometer.config;

import dev.linhvu.micrometer.Meter;
import dev.linhvu.micrometer.Tag;
import dev.linhvu.micrometer.Tags;

import java.util.ArrayList;
import java.util.List;
import java.util.Set;
import java.util.concurrent.ConcurrentHashMap;
import java.util.function.Predicate;

/**
 * A filter that transforms, accepts/denies, and configures meters during registration.
 * Filters are applied as a chain — the output of one filter feeds into the next.
 * <p>
 * The filter chain has two stages:
 * <ol>
 *   <li><b>Map</b> — transforms the {@link Meter.Id} (rename, add/remove tags)</li>
 *   <li><b>Accept</b> — decides whether to register a real meter or return a no-op</li>
 * </ol>
 * <p>
 * A third stage ({@code configure}) for histogram/distribution settings will be added
 * in Feature 10 (Distribution Statistics).
 * <p>
 * Each method has a sensible default (identity for map, NEUTRAL for accept), so
 * filters only need to override the stage they care about.
 */
public interface MeterFilter {

    /**
     * Transforms the meter's identity before registration. Enables renaming metrics,
     * adding common tags, or removing tag keys.
     * <p>
     * Each filter's output becomes the next filter's input. The default implementation
     * returns the ID unchanged.
     *
     * @param id The meter identity to potentially transform.
     * @return A (possibly new) Meter.Id.
     */
    default Meter.Id map(Meter.Id id) {
        return id;
    }

    /**
     * Determines whether a meter with the given (already-mapped) ID should be registered.
     * <p>
     * The chain short-circuits on the first non-{@code NEUTRAL} reply:
     * {@code DENY} → return a no-op meter; {@code ACCEPT} → register immediately.
     * If all filters return {@code NEUTRAL}, the meter is accepted.
     *
     * @param id The (already-mapped) meter identity.
     * @return DENY, NEUTRAL, or ACCEPT.
     */
    default MeterFilterReply accept(Meter.Id id) {
        return MeterFilterReply.NEUTRAL;
    }

    // -----------------------------------------------------------------------
    // Built-in tag manipulation filters (override map)
    // -----------------------------------------------------------------------

    /**
     * Adds common tags to every meter. Existing tags with the same key are NOT
     * overridden — the meter's own tags take precedence over common tags.
     * <p>
     * This is achieved by prepending common tags so that the merge gives precedence
     * to the meter's existing tags (since Tags.and() lets the "other" side win on
     * key conflicts, we put common tags first so meter tags override).
     *
     * @param tags Tags to add to all meters.
     * @return A filter that adds common tags.
     */
    static MeterFilter commonTags(Iterable<Tag> tags) {
        return new MeterFilter() {
            @Override
            public Meter.Id map(Meter.Id id) {
                // Prepend common tags, then merge with the meter's own tags.
                // Tags.of(tags).and(id.getTagsAsIterable()) gives precedence to
                // the meter's own tags on key conflicts.
                Tags merged = Tags.of(tags).and(id.getTagsAsIterable());
                return id.replaceTags(merged);
            }
        };
    }

    /**
     * Renames a tag key for meters whose name starts with the given prefix.
     *
     * @param meterNamePrefix Meter name prefix to match (empty string matches all).
     * @param fromTagKey      The current tag key.
     * @param toTagKey        The new tag key.
     * @return A filter that renames the tag key.
     */
    static MeterFilter renameTag(String meterNamePrefix, String fromTagKey, String toTagKey) {
        return new MeterFilter() {
            @Override
            public Meter.Id map(Meter.Id id) {
                if (!id.getName().startsWith(meterNamePrefix)) {
                    return id;
                }
                List<Tag> newTags = new ArrayList<>();
                for (Tag tag : id.getTagsAsIterable()) {
                    if (tag.getKey().equals(fromTagKey)) {
                        newTags.add(Tag.of(toTagKey, tag.getValue()));
                    } else {
                        newTags.add(tag);
                    }
                }
                return id.replaceTags(newTags);
            }
        };
    }

    /**
     * Removes tags with the specified keys from all meters.
     *
     * @param tagKeys Tag keys to remove.
     * @return A filter that strips the specified tag keys.
     */
    static MeterFilter ignoreTags(String... tagKeys) {
        Set<String> ignoreSet = Set.of(tagKeys);
        return new MeterFilter() {
            @Override
            public Meter.Id map(Meter.Id id) {
                List<Tag> filtered = new ArrayList<>();
                for (Tag tag : id.getTagsAsIterable()) {
                    if (!ignoreSet.contains(tag.getKey())) {
                        filtered.add(tag);
                    }
                }
                return id.replaceTags(filtered);
            }
        };
    }

    // -----------------------------------------------------------------------
    // Built-in accept/deny filters (override accept)
    // -----------------------------------------------------------------------

    /**
     * Returns ACCEPT for meters matching the predicate, NEUTRAL otherwise.
     */
    static MeterFilter accept(Predicate<Meter.Id> predicate) {
        return new MeterFilter() {
            @Override
            public MeterFilterReply accept(Meter.Id id) {
                return predicate.test(id) ? MeterFilterReply.ACCEPT : MeterFilterReply.NEUTRAL;
            }
        };
    }

    /**
     * Unconditionally accepts all meters. Useful to short-circuit after more
     * specific deny filters.
     */
    static MeterFilter accept() {
        return accept(id -> true);
    }

    /**
     * Returns DENY for meters matching the predicate, NEUTRAL otherwise.
     */
    static MeterFilter deny(Predicate<Meter.Id> predicate) {
        return new MeterFilter() {
            @Override
            public MeterFilterReply accept(Meter.Id id) {
                return predicate.test(id) ? MeterFilterReply.DENY : MeterFilterReply.NEUTRAL;
            }
        };
    }

    /**
     * Unconditionally denies all meters.
     */
    static MeterFilter deny() {
        return deny(id -> true);
    }

    /**
     * Whitelist approach: returns NEUTRAL for meters matching the predicate (allowing
     * subsequent filters to accept them), and DENY for everything else.
     */
    static MeterFilter denyUnless(Predicate<Meter.Id> predicate) {
        return new MeterFilter() {
            @Override
            public MeterFilterReply accept(Meter.Id id) {
                return predicate.test(id) ? MeterFilterReply.NEUTRAL : MeterFilterReply.DENY;
            }
        };
    }

    /**
     * Convenience: accepts meters whose name starts with the given prefix.
     */
    static MeterFilter acceptNameStartsWith(String prefix) {
        return accept(id -> id.getName().startsWith(prefix));
    }

    /**
     * Convenience: denies meters whose name starts with the given prefix.
     */
    static MeterFilter denyNameStartsWith(String prefix) {
        return deny(id -> id.getName().startsWith(prefix));
    }

    // -----------------------------------------------------------------------
    // Cardinality control filters (stateful)
    // -----------------------------------------------------------------------

    /**
     * Limits the total number of unique meters that can be registered. Once the
     * cap is reached, all new meters are denied.
     * <p>
     * Uses a {@link ConcurrentHashMap}-backed set to track seen meter IDs with
     * O(1) lookup.
     *
     * @param maximumMetrics The maximum number of unique meters allowed.
     * @return A filter that enforces the cap.
     */
    static MeterFilter maximumAllowableMetrics(int maximumMetrics) {
        Set<Meter.Id> seen = ConcurrentHashMap.newKeySet();
        return new MeterFilter() {
            @Override
            public MeterFilterReply accept(Meter.Id id) {
                if (seen.contains(id)) {
                    return MeterFilterReply.NEUTRAL;
                }
                if (seen.size() >= maximumMetrics) {
                    return MeterFilterReply.DENY;
                }
                seen.add(id);
                return MeterFilterReply.NEUTRAL;
            }
        };
    }

}
```

#### File: `src/main/java/dev/linhvu/micrometer/MeterRegistry.java` [MODIFIED]

```java
package dev.linhvu.micrometer;

import dev.linhvu.micrometer.config.MeterFilter;
import dev.linhvu.micrometer.config.MeterFilterReply;
import dev.linhvu.micrometer.noop.NoopCounter;
import dev.linhvu.micrometer.noop.NoopDistributionSummary;
import dev.linhvu.micrometer.noop.NoopGauge;
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

    private volatile boolean closed = false;

    protected MeterRegistry(Clock clock) {
        this.clock = Objects.requireNonNull(clock, "clock must not be null");
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
     * Registers a timer using the pre-built ID. Called by {@link Timer.Builder#register(MeterRegistry)}.
     */
    Timer timer(Meter.Id id) {
        return registerMeterIfNecessary(Timer.class, id, this::newTimer, NoopTimer::new);
    }

    /**
     * Registers a distribution summary using the pre-built ID and scale.
     * Called by {@link DistributionSummary.Builder#register(MeterRegistry)}.
     */
    DistributionSummary summary(Meter.Id id, double scale) {
        return registerMeterIfNecessary(DistributionSummary.class, id,
                mappedId -> newDistributionSummary(mappedId, scale), NoopDistributionSummary::new);
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
     * @param id    The meter identity.
     * @param scale Factor to scale each recorded value by.
     */
    protected abstract DistributionSummary newDistributionSummary(Meter.Id id, double scale);

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

### Test Code

#### File: `src/test/java/dev/linhvu/micrometer/config/MeterFilterReplyTest.java` [NEW]

```java
package dev.linhvu.micrometer.config;

import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;

/**
 * Tests for {@link MeterFilterReply} — verifies that the enum values exist
 * and have the expected ordering (DENY, NEUTRAL, ACCEPT).
 */
class MeterFilterReplyTest {

    @Test
    void shouldHaveThreeValues() {
        assertThat(MeterFilterReply.values()).hasSize(3);
    }

    @Test
    void shouldHaveExpectedOrdering() {
        MeterFilterReply[] values = MeterFilterReply.values();
        assertThat(values[0]).isEqualTo(MeterFilterReply.DENY);
        assertThat(values[1]).isEqualTo(MeterFilterReply.NEUTRAL);
        assertThat(values[2]).isEqualTo(MeterFilterReply.ACCEPT);
    }

}
```

#### File: `src/test/java/dev/linhvu/micrometer/config/MeterFilterTest.java` [NEW]

```java
package dev.linhvu.micrometer.config;

import dev.linhvu.micrometer.Counter;
import dev.linhvu.micrometer.Meter;
import dev.linhvu.micrometer.Tag;
import dev.linhvu.micrometer.Tags;
import dev.linhvu.micrometer.noop.NoopCounter;
import dev.linhvu.micrometer.simple.SimpleMeterRegistry;
import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;

/**
 * Tests for {@link MeterFilter} static factory methods and default behavior.
 * Uses SimpleMeterRegistry to verify end-to-end filter integration.
 */
class MeterFilterTest {

    // -----------------------------------------------------------------------
    // Default behavior
    // -----------------------------------------------------------------------

    @Test
    void shouldReturnIdUnchanged_WhenMapNotOverridden() {
        MeterFilter filter = new MeterFilter() {};
        Meter.Id id = new Meter.Id("test", Tags.of("k", "v"), Meter.Type.COUNTER, null, null);

        assertThat(filter.map(id)).isSameAs(id);
    }

    @Test
    void shouldReturnNeutral_WhenAcceptNotOverridden() {
        MeterFilter filter = new MeterFilter() {};
        Meter.Id id = new Meter.Id("test", Tags.empty(), Meter.Type.COUNTER, null, null);

        assertThat(filter.accept(id)).isEqualTo(MeterFilterReply.NEUTRAL);
    }

    // -----------------------------------------------------------------------
    // commonTags
    // -----------------------------------------------------------------------

    @Test
    void shouldAddCommonTags_WhenCommonTagsFilter() {
        MeterFilter filter = MeterFilter.commonTags(Tags.of("env", "prod", "region", "us-east"));
        Meter.Id id = new Meter.Id("requests", Tags.empty(), Meter.Type.COUNTER, null, null);

        Meter.Id mapped = filter.map(id);

        assertThat(mapped.getTag("env")).isEqualTo("prod");
        assertThat(mapped.getTag("region")).isEqualTo("us-east");
    }

    @Test
    void shouldNotOverrideExistingTag_WhenCommonTagHasSameKey() {
        MeterFilter filter = MeterFilter.commonTags(Tags.of("env", "prod"));
        Meter.Id id = new Meter.Id("requests", Tags.of("env", "staging"),
                Meter.Type.COUNTER, null, null);

        Meter.Id mapped = filter.map(id);

        // Meter's own tag takes precedence over common tag
        assertThat(mapped.getTag("env")).isEqualTo("staging");
    }

    @Test
    void shouldPreserveName_WhenCommonTagsAdded() {
        MeterFilter filter = MeterFilter.commonTags(Tags.of("env", "prod"));
        Meter.Id id = new Meter.Id("http.requests", Tags.empty(), Meter.Type.COUNTER, "desc", "unit");

        Meter.Id mapped = filter.map(id);

        assertThat(mapped.getName()).isEqualTo("http.requests");
    }

    // -----------------------------------------------------------------------
    // renameTag
    // -----------------------------------------------------------------------

    @Test
    void shouldRenameTag_WhenNameMatchesPrefix() {
        MeterFilter filter = MeterFilter.renameTag("http", "status", "statusCode");
        Meter.Id id = new Meter.Id("http.requests", Tags.of("status", "200"),
                Meter.Type.COUNTER, null, null);

        Meter.Id mapped = filter.map(id);

        assertThat(mapped.getTag("status")).isNull();
        assertThat(mapped.getTag("statusCode")).isEqualTo("200");
    }

    @Test
    void shouldNotRenameTag_WhenNameDoesNotMatchPrefix() {
        MeterFilter filter = MeterFilter.renameTag("http", "status", "statusCode");
        Meter.Id id = new Meter.Id("db.queries", Tags.of("status", "ok"),
                Meter.Type.COUNTER, null, null);

        Meter.Id mapped = filter.map(id);

        assertThat(mapped.getTag("status")).isEqualTo("ok");
        assertThat(mapped.getTag("statusCode")).isNull();
    }

    // -----------------------------------------------------------------------
    // ignoreTags
    // -----------------------------------------------------------------------

    @Test
    void shouldRemoveIgnoredTags_WhenIgnoreTagsFilter() {
        MeterFilter filter = MeterFilter.ignoreTags("host", "instance");
        Meter.Id id = new Meter.Id("requests",
                Tags.of("host", "server1", "instance", "i-123", "method", "GET"),
                Meter.Type.COUNTER, null, null);

        Meter.Id mapped = filter.map(id);

        assertThat(mapped.getTag("host")).isNull();
        assertThat(mapped.getTag("instance")).isNull();
        assertThat(mapped.getTag("method")).isEqualTo("GET");
    }

    @Test
    void shouldReturnUnchangedId_WhenIgnoredTagNotPresent() {
        MeterFilter filter = MeterFilter.ignoreTags("host");
        Meter.Id id = new Meter.Id("requests", Tags.of("method", "GET"),
                Meter.Type.COUNTER, null, null);

        Meter.Id mapped = filter.map(id);

        assertThat(mapped.getTag("method")).isEqualTo("GET");
    }

    // -----------------------------------------------------------------------
    // accept / deny
    // -----------------------------------------------------------------------

    @Test
    void shouldReturnAccept_WhenAcceptPredicateMatches() {
        MeterFilter filter = MeterFilter.accept(id -> id.getName().startsWith("http"));
        Meter.Id id = new Meter.Id("http.requests", Tags.empty(), Meter.Type.COUNTER, null, null);

        assertThat(filter.accept(id)).isEqualTo(MeterFilterReply.ACCEPT);
    }

    @Test
    void shouldReturnNeutral_WhenAcceptPredicateDoesNotMatch() {
        MeterFilter filter = MeterFilter.accept(id -> id.getName().startsWith("http"));
        Meter.Id id = new Meter.Id("db.queries", Tags.empty(), Meter.Type.COUNTER, null, null);

        assertThat(filter.accept(id)).isEqualTo(MeterFilterReply.NEUTRAL);
    }

    @Test
    void shouldReturnAcceptForAll_WhenUnconditionalAccept() {
        MeterFilter filter = MeterFilter.accept();
        Meter.Id id = new Meter.Id("anything", Tags.empty(), Meter.Type.COUNTER, null, null);

        assertThat(filter.accept(id)).isEqualTo(MeterFilterReply.ACCEPT);
    }

    @Test
    void shouldReturnDeny_WhenDenyPredicateMatches() {
        MeterFilter filter = MeterFilter.deny(id -> id.getName().contains("internal"));
        Meter.Id id = new Meter.Id("app.internal.cache", Tags.empty(), Meter.Type.COUNTER, null, null);

        assertThat(filter.accept(id)).isEqualTo(MeterFilterReply.DENY);
    }

    @Test
    void shouldReturnNeutral_WhenDenyPredicateDoesNotMatch() {
        MeterFilter filter = MeterFilter.deny(id -> id.getName().contains("internal"));
        Meter.Id id = new Meter.Id("http.requests", Tags.empty(), Meter.Type.COUNTER, null, null);

        assertThat(filter.accept(id)).isEqualTo(MeterFilterReply.NEUTRAL);
    }

    @Test
    void shouldReturnDenyForAll_WhenUnconditionalDeny() {
        MeterFilter filter = MeterFilter.deny();
        Meter.Id id = new Meter.Id("anything", Tags.empty(), Meter.Type.COUNTER, null, null);

        assertThat(filter.accept(id)).isEqualTo(MeterFilterReply.DENY);
    }

    // -----------------------------------------------------------------------
    // denyUnless
    // -----------------------------------------------------------------------

    @Test
    void shouldReturnNeutral_WhenDenyUnlessPredicateMatches() {
        MeterFilter filter = MeterFilter.denyUnless(id -> id.getName().startsWith("http"));
        Meter.Id id = new Meter.Id("http.requests", Tags.empty(), Meter.Type.COUNTER, null, null);

        assertThat(filter.accept(id)).isEqualTo(MeterFilterReply.NEUTRAL);
    }

    @Test
    void shouldReturnDeny_WhenDenyUnlessPredicateDoesNotMatch() {
        MeterFilter filter = MeterFilter.denyUnless(id -> id.getName().startsWith("http"));
        Meter.Id id = new Meter.Id("jvm.memory", Tags.empty(), Meter.Type.COUNTER, null, null);

        assertThat(filter.accept(id)).isEqualTo(MeterFilterReply.DENY);
    }

    // -----------------------------------------------------------------------
    // Name-prefix convenience methods
    // -----------------------------------------------------------------------

    @Test
    void shouldAcceptNameStartsWith_WhenMatches() {
        MeterFilter filter = MeterFilter.acceptNameStartsWith("http.");
        Meter.Id id = new Meter.Id("http.requests", Tags.empty(), Meter.Type.COUNTER, null, null);

        assertThat(filter.accept(id)).isEqualTo(MeterFilterReply.ACCEPT);
    }

    @Test
    void shouldDenyNameStartsWith_WhenMatches() {
        MeterFilter filter = MeterFilter.denyNameStartsWith("jvm.");
        Meter.Id id = new Meter.Id("jvm.memory.used", Tags.empty(), Meter.Type.COUNTER, null, null);

        assertThat(filter.accept(id)).isEqualTo(MeterFilterReply.DENY);
    }

    // -----------------------------------------------------------------------
    // maximumAllowableMetrics
    // -----------------------------------------------------------------------

    @Test
    void shouldAllowMetrics_WhenUnderLimit() {
        MeterFilter filter = MeterFilter.maximumAllowableMetrics(3);

        Meter.Id id1 = new Meter.Id("a", Tags.empty(), Meter.Type.COUNTER, null, null);
        Meter.Id id2 = new Meter.Id("b", Tags.empty(), Meter.Type.COUNTER, null, null);
        Meter.Id id3 = new Meter.Id("c", Tags.empty(), Meter.Type.COUNTER, null, null);

        assertThat(filter.accept(id1)).isEqualTo(MeterFilterReply.NEUTRAL);
        assertThat(filter.accept(id2)).isEqualTo(MeterFilterReply.NEUTRAL);
        assertThat(filter.accept(id3)).isEqualTo(MeterFilterReply.NEUTRAL);
    }

    @Test
    void shouldDenyMetrics_WhenOverLimit() {
        MeterFilter filter = MeterFilter.maximumAllowableMetrics(2);

        Meter.Id id1 = new Meter.Id("a", Tags.empty(), Meter.Type.COUNTER, null, null);
        Meter.Id id2 = new Meter.Id("b", Tags.empty(), Meter.Type.COUNTER, null, null);
        Meter.Id id3 = new Meter.Id("c", Tags.empty(), Meter.Type.COUNTER, null, null);

        filter.accept(id1); // seen: {a}
        filter.accept(id2); // seen: {a, b}

        assertThat(filter.accept(id3)).isEqualTo(MeterFilterReply.DENY);
    }

    @Test
    void shouldAllowAlreadySeenMetric_WhenOverLimit() {
        MeterFilter filter = MeterFilter.maximumAllowableMetrics(2);

        Meter.Id id1 = new Meter.Id("a", Tags.empty(), Meter.Type.COUNTER, null, null);
        Meter.Id id2 = new Meter.Id("b", Tags.empty(), Meter.Type.COUNTER, null, null);

        filter.accept(id1);
        filter.accept(id2);

        // id1 was already seen — it should still be allowed
        assertThat(filter.accept(id1)).isEqualTo(MeterFilterReply.NEUTRAL);
    }

}
```

#### File: `src/test/java/dev/linhvu/micrometer/integration/MeterFilterIntegrationTest.java` [NEW]

```java
package dev.linhvu.micrometer.integration;

import dev.linhvu.micrometer.Counter;
import dev.linhvu.micrometer.DistributionSummary;
import dev.linhvu.micrometer.Gauge;
import dev.linhvu.micrometer.Tags;
import dev.linhvu.micrometer.Timer;
import dev.linhvu.micrometer.config.MeterFilter;
import dev.linhvu.micrometer.noop.NoopCounter;
import dev.linhvu.micrometer.noop.NoopDistributionSummary;
import dev.linhvu.micrometer.noop.NoopGauge;
import dev.linhvu.micrometer.noop.NoopTimer;
import dev.linhvu.micrometer.simple.SimpleMeterRegistry;
import org.junit.jupiter.api.Test;

import java.util.concurrent.atomic.AtomicInteger;

import static org.assertj.core.api.Assertions.assertThat;

/**
 * Integration tests for MeterFilter with MeterRegistry. Verifies that the
 * filter chain correctly transforms, accepts, and denies meters across
 * all meter types (Counter, Gauge, Timer, DistributionSummary).
 */
class MeterFilterIntegrationTest {

    // -----------------------------------------------------------------------
    // Common tags integration
    // -----------------------------------------------------------------------

    @Test
    void shouldAddCommonTagsToCounter_WhenRegistered() {
        SimpleMeterRegistry registry = new SimpleMeterRegistry();
        registry.meterFilter(MeterFilter.commonTags(Tags.of("app", "myapp", "env", "prod")));

        Counter counter = registry.counter("http.requests", "method", "GET");

        assertThat(counter.getId().getTag("app")).isEqualTo("myapp");
        assertThat(counter.getId().getTag("env")).isEqualTo("prod");
        assertThat(counter.getId().getTag("method")).isEqualTo("GET");
    }

    @Test
    void shouldAddCommonTagsToTimer_WhenRegistered() {
        SimpleMeterRegistry registry = new SimpleMeterRegistry();
        registry.meterFilter(MeterFilter.commonTags(Tags.of("region", "us-east")));

        Timer timer = registry.timer("http.latency", "method", "GET");

        assertThat(timer.getId().getTag("region")).isEqualTo("us-east");
        assertThat(timer.getId().getTag("method")).isEqualTo("GET");
    }

    @Test
    void shouldAddCommonTagsToGauge_WhenRegistered() {
        SimpleMeterRegistry registry = new SimpleMeterRegistry();
        registry.meterFilter(MeterFilter.commonTags(Tags.of("env", "prod")));

        AtomicInteger value = new AtomicInteger(42);
        Gauge.builder("pool.size", value, AtomicInteger::doubleValue)
                .register(registry);

        // Find the gauge in the registry
        assertThat(registry.getMeters()).hasSize(1);
        assertThat(registry.getMeters().get(0).getId().getTag("env")).isEqualTo("prod");
    }

    @Test
    void shouldAddCommonTagsToDistributionSummary_WhenRegistered() {
        SimpleMeterRegistry registry = new SimpleMeterRegistry();
        registry.meterFilter(MeterFilter.commonTags(Tags.of("env", "prod")));

        DistributionSummary summary = registry.summary("response.size", "endpoint", "/api");

        assertThat(summary.getId().getTag("env")).isEqualTo("prod");
        assertThat(summary.getId().getTag("endpoint")).isEqualTo("/api");
    }

    // -----------------------------------------------------------------------
    // Deny filter integration
    // -----------------------------------------------------------------------

    @Test
    void shouldReturnNoopCounter_WhenFilterDenies() {
        SimpleMeterRegistry registry = new SimpleMeterRegistry();
        registry.meterFilter(MeterFilter.denyNameStartsWith("internal."));

        Counter counter = registry.counter("internal.debug.count");

        assertThat(counter).isInstanceOf(NoopCounter.class);
        assertThat(registry.getMeters()).isEmpty(); // noop not stored
    }

    @Test
    void shouldReturnNoopTimer_WhenFilterDenies() {
        SimpleMeterRegistry registry = new SimpleMeterRegistry();
        registry.meterFilter(MeterFilter.deny());

        Timer timer = registry.timer("http.latency");

        assertThat(timer).isInstanceOf(NoopTimer.class);
        assertThat(registry.getMeters()).isEmpty();
    }

    @Test
    void shouldReturnNoopGauge_WhenFilterDenies() {
        SimpleMeterRegistry registry = new SimpleMeterRegistry();
        registry.meterFilter(MeterFilter.deny());

        AtomicInteger value = new AtomicInteger(42);
        Gauge gauge = Gauge.builder("pool.size", value, AtomicInteger::doubleValue)
                .register(registry);

        assertThat(gauge).isInstanceOf(NoopGauge.class);
        assertThat(registry.getMeters()).isEmpty();
    }

    @Test
    void shouldReturnNoopDistributionSummary_WhenFilterDenies() {
        SimpleMeterRegistry registry = new SimpleMeterRegistry();
        registry.meterFilter(MeterFilter.deny());

        DistributionSummary summary = registry.summary("response.size");

        assertThat(summary).isInstanceOf(NoopDistributionSummary.class);
        assertThat(registry.getMeters()).isEmpty();
    }

    @Test
    void shouldAllowNonMatchingMeters_WhenDenyNameStartsWith() {
        SimpleMeterRegistry registry = new SimpleMeterRegistry();
        registry.meterFilter(MeterFilter.denyNameStartsWith("internal."));

        Counter allowed = registry.counter("http.requests");
        Counter denied = registry.counter("internal.debug");

        assertThat(allowed).isNotInstanceOf(NoopCounter.class);
        assertThat(denied).isInstanceOf(NoopCounter.class);
        assertThat(registry.getMeters()).hasSize(1);
    }

    // -----------------------------------------------------------------------
    // Filter chain ordering
    // -----------------------------------------------------------------------

    @Test
    void shouldApplyFiltersInOrder_WhenMultipleMapFilters() {
        SimpleMeterRegistry registry = new SimpleMeterRegistry();

        // First: add env=prod
        registry.meterFilter(MeterFilter.commonTags(Tags.of("env", "prod")));
        // Second: add region=us-east
        registry.meterFilter(MeterFilter.commonTags(Tags.of("region", "us-east")));

        Counter counter = registry.counter("requests");

        assertThat(counter.getId().getTag("env")).isEqualTo("prod");
        assertThat(counter.getId().getTag("region")).isEqualTo("us-east");
    }

    @Test
    void shouldShortCircuitOnDeny_WhenDenyBeforeAccept() {
        SimpleMeterRegistry registry = new SimpleMeterRegistry();

        // First: deny everything
        registry.meterFilter(MeterFilter.deny());
        // Second: accept everything (never reached due to short-circuit)
        registry.meterFilter(MeterFilter.accept());

        Counter counter = registry.counter("requests");

        assertThat(counter).isInstanceOf(NoopCounter.class);
    }

    @Test
    void shouldShortCircuitOnAccept_WhenAcceptBeforeDeny() {
        SimpleMeterRegistry registry = new SimpleMeterRegistry();

        // First: accept http metrics
        registry.meterFilter(MeterFilter.acceptNameStartsWith("http."));
        // Second: deny everything else
        registry.meterFilter(MeterFilter.deny());

        Counter httpCounter = registry.counter("http.requests");
        Counter otherCounter = registry.counter("jvm.memory");

        // http.requests matched ACCEPT first → registered
        assertThat(httpCounter).isNotInstanceOf(NoopCounter.class);
        // jvm.memory didn't match the first filter (NEUTRAL), hit deny → noop
        assertThat(otherCounter).isInstanceOf(NoopCounter.class);
    }

    // -----------------------------------------------------------------------
    // denyUnless (whitelist pattern)
    // -----------------------------------------------------------------------

    @Test
    void shouldOnlyAllowWhitelistedMeters_WhenDenyUnlessUsed() {
        SimpleMeterRegistry registry = new SimpleMeterRegistry();
        registry.meterFilter(MeterFilter.denyUnless(id -> id.getName().startsWith("http.")));

        Counter httpCounter = registry.counter("http.requests");
        Counter jvmCounter = registry.counter("jvm.memory");

        assertThat(httpCounter).isNotInstanceOf(NoopCounter.class);
        assertThat(jvmCounter).isInstanceOf(NoopCounter.class);
    }

    // -----------------------------------------------------------------------
    // maximumAllowableMetrics
    // -----------------------------------------------------------------------

    @Test
    void shouldDenyNewMeters_WhenMaximumReached() {
        SimpleMeterRegistry registry = new SimpleMeterRegistry();
        registry.meterFilter(MeterFilter.maximumAllowableMetrics(2));

        Counter c1 = registry.counter("a");
        Counter c2 = registry.counter("b");
        Counter c3 = registry.counter("c");

        assertThat(c1).isNotInstanceOf(NoopCounter.class);
        assertThat(c2).isNotInstanceOf(NoopCounter.class);
        assertThat(c3).isInstanceOf(NoopCounter.class);
        assertThat(registry.getMeters()).hasSize(2);
    }

    // -----------------------------------------------------------------------
    // Map + deny combined
    // -----------------------------------------------------------------------

    @Test
    void shouldMapBeforeDenyCheck_WhenBothFiltersPresent() {
        SimpleMeterRegistry registry = new SimpleMeterRegistry();

        // First: rename by adding a prefix tag
        registry.meterFilter(MeterFilter.commonTags(Tags.of("env", "prod")));
        // Second: deny meters that don't have a specific tag value
        registry.meterFilter(MeterFilter.deny(id -> !"prod".equals(id.getTag("env"))));

        Counter counter = registry.counter("requests");

        // The common tag was added first, so env=prod exists when deny checks
        assertThat(counter).isNotInstanceOf(NoopCounter.class);
        assertThat(counter.getId().getTag("env")).isEqualTo("prod");
    }

    // -----------------------------------------------------------------------
    // Noop meters are functional (don't throw)
    // -----------------------------------------------------------------------

    @Test
    void shouldSilentlyIgnoreOperations_WhenDeniedNoopCounter() {
        SimpleMeterRegistry registry = new SimpleMeterRegistry();
        registry.meterFilter(MeterFilter.deny());

        Counter counter = registry.counter("requests");
        counter.increment();
        counter.increment(100);

        assertThat(counter.count()).isEqualTo(0.0);
    }

    @Test
    void shouldStillExecuteFunction_WhenDeniedNoopTimer() {
        SimpleMeterRegistry registry = new SimpleMeterRegistry();
        registry.meterFilter(MeterFilter.deny());

        Timer timer = registry.timer("latency");
        String result = timer.record(() -> "hello");

        assertThat(result).isEqualTo("hello");
        assertThat(timer.count()).isEqualTo(0);
    }

    // -----------------------------------------------------------------------
    // Fluent chaining
    // -----------------------------------------------------------------------

    @Test
    void shouldSupportFluentChaining_WhenAddingMultipleFilters() {
        SimpleMeterRegistry registry = new SimpleMeterRegistry();
        registry.meterFilter(MeterFilter.commonTags(Tags.of("env", "prod")))
                .meterFilter(MeterFilter.denyNameStartsWith("internal."));

        Counter allowed = registry.counter("http.requests");
        Counter denied = registry.counter("internal.debug");

        assertThat(allowed.getId().getTag("env")).isEqualTo("prod");
        assertThat(denied).isInstanceOf(NoopCounter.class);
    }

    // -----------------------------------------------------------------------
    // Deduplication still works after mapping
    // -----------------------------------------------------------------------

    @Test
    void shouldDeduplicateByMappedId_WhenFiltersTransformId() {
        SimpleMeterRegistry registry = new SimpleMeterRegistry();
        registry.meterFilter(MeterFilter.commonTags(Tags.of("env", "prod")));

        Counter first = registry.counter("requests");
        Counter second = registry.counter("requests");

        assertThat(first).isSameAs(second);
        assertThat(registry.getMeters()).hasSize(1);
    }

    // -----------------------------------------------------------------------
    // ignoreTags integration
    // -----------------------------------------------------------------------

    @Test
    void shouldRemoveSpecifiedTags_WhenIgnoreTagsFilter() {
        SimpleMeterRegistry registry = new SimpleMeterRegistry();
        registry.meterFilter(MeterFilter.ignoreTags("host"));

        Counter counter = Counter.builder("requests")
                .tag("method", "GET")
                .tag("host", "server-1")
                .register(registry);

        assertThat(counter.getId().getTag("host")).isNull();
        assertThat(counter.getId().getTag("method")).isEqualTo("GET");
    }

    // -----------------------------------------------------------------------
    // renameTag integration
    // -----------------------------------------------------------------------

    @Test
    void shouldRenameTagOnMatchingMeters_WhenRenameTagFilter() {
        SimpleMeterRegistry registry = new SimpleMeterRegistry();
        registry.meterFilter(MeterFilter.renameTag("http", "status", "http.status"));

        Counter counter = Counter.builder("http.requests")
                .tag("status", "200")
                .register(registry);

        assertThat(counter.getId().getTag("status")).isNull();
        assertThat(counter.getId().getTag("http.status")).isEqualTo("200");
    }

}
```

---

## Summary

| Concept | What It Means |
|---------|---------------|
| **MeterFilter** | An interface with `map()` and `accept()` default methods that intercepts meter registration |
| **MeterFilterReply** | Three-value enum (DENY/NEUTRAL/ACCEPT) enabling composable whitelist/blacklist patterns |
| **Chain of Responsibility** | Filters are applied sequentially — each filter's output feeds the next; accept short-circuits |
| **Map stage** | Transforms the meter's ID (common tags, rename, remove) before deduplication |
| **Accept stage** | Decides whether to create a real meter or return a no-op; first non-NEUTRAL reply wins |
| **Volatile array** | Copy-on-write pattern for lock-free reads on the hot path with synchronized writes on the cold path |
| **Noop meter** | A silent fallback returned when a filter denies registration — ignores data but still executes wrapped functions |

**Next: Chapter 9 — NamingConvention** — How "http.server.requests" becomes "http_server_requests" for Prometheus or "httpServerRequests" for another backend. Each registry applies a naming convention when exporting, translating names and tag keys to match the backend's format.
