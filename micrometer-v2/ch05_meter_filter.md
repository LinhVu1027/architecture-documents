# Chapter 5: Meter Filtering & Transformation

> **Feature 5** · Tier 2 · Depends on: Features 1–4 (filters apply to all meter types during registration)

After this chapter you will be able to **globally control** which meters get registered and
**transform** their identities at registration time — adding common tags, denying metrics by
name prefix, renaming tag keys, and stripping high-cardinality tags — all configured centrally
on the registry, applied automatically to every meter.

---

## 1. The API Contract

Before writing any implementation, let's define exactly what a client sees.

### What the client imports

```java
import simple.micrometer.api.MeterFilter;
import simple.micrometer.api.MeterFilterReply;
import simple.micrometer.api.MeterRegistry;
import simple.micrometer.internal.SimpleMeterRegistry;
```

### What the client writes

```java
MeterRegistry registry = new SimpleMeterRegistry();

// Add common tags to every meter
registry.config().commonTags("app", "order-service", "env", "prod");

// Deny metrics by name prefix
registry.config().meterFilter(MeterFilter.denyNameStartsWith("jvm.gc"));

// Rename a tag across matching meters
registry.config().meterFilter(MeterFilter.renameTag("http", "status", "http.status"));

// Custom accept/deny logic
registry.config().meterFilter(new MeterFilter() {
    @Override
    public MeterFilterReply accept(Meter.Id id) {
        if (id.getName().startsWith("internal.")) return MeterFilterReply.DENY;
        return MeterFilterReply.NEUTRAL;
    }
});

// Counter registered after filters are applied
Counter counter = registry.counter("http.requests", "method", "GET");
// counter.getId().getTags() includes "app"="order-service", "env"="prod"
```

### Behavioral contract

| Promise | Detail |
|---------|--------|
| Two-phase filtering | `map()` transforms the id first, then `accept()` decides whether to register |
| Insertion-order execution | Filters run in the order they were added via `config().meterFilter()` |
| First non-NEUTRAL wins | DENY short-circuits to a noop; ACCEPT short-circuits to registration; NEUTRAL defers to the next filter |
| Default accept | If all filters return NEUTRAL, the meter is accepted |
| Common tags don't override | Meter-specific tags with the same key take precedence over common tags |
| Denied = noop | Denied meters return a noop implementation (NoopCounter, NoopTimer, etc.) |
| Denied meters not stored | Denied meters do not appear in `registry.getMeters()` |
| Applies to all meter types | Counters, Gauges, Timers, and DistributionSummaries all pass through the filter chain |

---

## 2. Client Usage & Tests

These tests define the API's behavior — they were written **before** any implementation.

### Common tags are applied to every meter

```java
@Test
void shouldAddCommonTags_ToEveryMeter() {
    registry.config().commonTags("app", "order-service", "env", "prod");

    Counter counter = registry.counter("http.requests", "method", "GET");

    assertThat(counter.getId().getTag("app")).isEqualTo("order-service");
    assertThat(counter.getId().getTag("env")).isEqualTo("prod");
    assertThat(counter.getId().getTag("method")).isEqualTo("GET");
}
```

### Meter-specific tags override common tags

```java
@Test
void shouldNotOverrideMeterSpecificTags_WhenCommonTagHasSameKey() {
    registry.config().commonTags("env", "prod");

    Counter counter = Counter.builder("http.requests")
            .tag("env", "staging")
            .register(registry);

    // Meter-specific tag wins over common tag
    assertThat(counter.getId().getTag("env")).isEqualTo("staging");
}
```

### Deny by name prefix

```java
@Test
void shouldDenyByNamePrefix() {
    registry.config().meterFilter(MeterFilter.denyNameStartsWith("jvm.gc"));

    Counter denied = registry.counter("jvm.gc.pause");
    Counter allowed = registry.counter("http.requests");

    assertThat(denied).isInstanceOf(NoopCounter.class);
    assertThat(allowed).isNotInstanceOf(NoopCounter.class);
}
```

### Denied meters don't appear in getMeters()

```java
@Test
void shouldNotStoreDeniedMeters_InRegistry() {
    registry.config().meterFilter(MeterFilter.denyNameStartsWith("jvm"));

    registry.counter("jvm.gc.pause");
    registry.counter("http.requests");

    assertThat(registry.getMeters()).hasSize(1);
    assertThat(registry.getMeters().get(0).getId().getName()).isEqualTo("http.requests");
}
```

### Accept short-circuits the chain

```java
@Test
void shouldShortCircuit_OnFirstAccept() {
    registry.config()
            .meterFilter(MeterFilter.accept())
            .meterFilter(MeterFilter.deny()); // never reached

    Counter counter = registry.counter("anything");

    assertThat(counter).isNotInstanceOf(NoopCounter.class);
}
```

### Map runs before accept

```java
@Test
void shouldApplyMapBeforeAccept() {
    registry.config()
            .meterFilter(new MeterFilter() {
                @Override
                public Meter.Id map(Meter.Id id) {
                    if (id.getName().equals("old.name")) {
                        return id.withName("internal.old.name");
                    }
                    return id;
                }
            })
            .meterFilter(MeterFilter.denyNameStartsWith("internal."));

    Counter counter = registry.counter("old.name");

    // Was renamed to "internal.old.name", then denied
    assertThat(counter).isInstanceOf(NoopCounter.class);
}
```

### Whitelist with denyUnless

```java
@Test
void shouldOnlyAllowMatchingMeters() {
    registry.config().meterFilter(
            MeterFilter.denyUnless(id -> id.getName().startsWith("http.")));

    Counter allowed = registry.counter("http.requests");
    Counter denied = registry.counter("jvm.gc.pause");

    assertThat(allowed).isNotInstanceOf(NoopCounter.class);
    assertThat(denied).isInstanceOf(NoopCounter.class);
}
```

### Rename tag key

```java
@Test
void shouldRenameTagKey_WhenNameMatchesPrefix() {
    registry.config().meterFilter(
            MeterFilter.renameTag("http", "status", "http.status"));

    Counter counter = registry.counter("http.requests",
            "status", "200", "method", "GET");

    assertThat(counter.getId().getTag("http.status")).isEqualTo("200");
    assertThat(counter.getId().getTag("status")).isNull();
    assertThat(counter.getId().getTag("method")).isEqualTo("GET");
}
```

### Ignore (strip) tags

```java
@Test
void shouldStripSpecifiedTags() {
    registry.config().meterFilter(MeterFilter.ignoreTags("host", "instance"));

    Counter counter = registry.counter("http.requests",
            "method", "GET", "host", "server-1", "instance", "i-123");

    assertThat(counter.getId().getTag("method")).isEqualTo("GET");
    assertThat(counter.getId().getTag("host")).isNull();
    assertThat(counter.getId().getTag("instance")).isNull();
}
```

---

## 3. Implementing the Call Chain

This feature is unique among what we've built so far: it doesn't add a new meter type.
Instead, it **modifies the registration pipeline** that all four existing meter types
pass through. The call chain becomes:

```
Client calls: registry.counter("http.requests", "method", "GET")
  │
  ├─► [API Layer] MeterRegistry.counter(String, String...)
  │     Constructs Meter.Id, delegates to counter(Meter.Id)
  │
  ├─► [Registration Layer] MeterRegistry.registerMeterIfNecessary()
  │     NEW: 1. applyMapFilters(id) — runs each filter's map() in order
  │     NEW: 2. applyAcceptFilters(mappedId) — first non-NEUTRAL wins
  │     NEW: 3. If denied → return noopFactory.apply(mappedId)
  │          4. Look up mappedId in ConcurrentHashMap
  │          5. If not found, create under lock
  │
  └─► [Implementation Layer] SimpleMeterRegistry.newCounter(mappedId)
        Creates a CumulativeCounter with the transformed identity
```

### Layer 1: MeterFilterReply — the three-state vote

This is the simplest piece — a three-value enum:

```java
public enum MeterFilterReply {
    DENY,     // reject → noop meter
    NEUTRAL,  // no opinion → continue to next filter
    ACCEPT    // accept → short-circuit, register real meter
}
```

The three-state design is deliberate. A two-state (accept/deny) system would force every
filter to make a definitive decision. NEUTRAL allows filters to be composed: a `commonTags`
filter that only transforms IDs returns NEUTRAL from `accept()`, letting later filters
decide whether to accept or deny.

### Layer 2: MeterFilter interface — two default methods + static factories

The interface has just two default methods, both pass-through by default:

```java
public interface MeterFilter {

    default Meter.Id map(Meter.Id id) {
        return id;  // no transformation
    }

    default MeterFilterReply accept(Meter.Id id) {
        return MeterFilterReply.NEUTRAL;  // no opinion
    }
}
```

This design means a filter only needs to override the method it cares about. A `commonTags`
filter only overrides `map()`. A `deny()` filter only overrides `accept()`. A complex filter
could override both.

**Static factory: commonTags**

The most commonly used filter. The subtle part is merge order — common tags are the
**base**, and meter-specific tags **override** on key conflict:

```java
static MeterFilter commonTags(Iterable<Tag> tags) {
    Tags commonTagSet = Tags.of(tags);
    return new MeterFilter() {
        @Override
        public Meter.Id map(Meter.Id id) {
            // Common tags are the base; meter-specific tags override on key conflict
            Tags merged = commonTagSet.and(id.getTagsAsIterable());
            return new Meter.Id(id.getName(), merged,
                    id.getBaseUnit(), id.getDescription(), id.getType());
        }
    };
}
```

Why `commonTagSet.and(meterTags)` and not the reverse? Because `Tags.and()` gives the
**argument** priority on key conflict. So `commonTags.and(meterTags)` means: start with
common tags, then let meter tags override. This prevents `commonTags("env", "prod")` from
silently overwriting a meter that explicitly set `env=staging`.

**Static factories: deny and accept**

These factories compose naturally. Each returns a filter that overrides only `accept()`:

```java
static MeterFilter deny(Predicate<Meter.Id> iff) {
    return new MeterFilter() {
        @Override
        public MeterFilterReply accept(Meter.Id id) {
            return iff.test(id) ? MeterFilterReply.DENY : MeterFilterReply.NEUTRAL;
        }
    };
}

static MeterFilter denyNameStartsWith(String prefix) {
    return deny(id -> id.getName().startsWith(prefix));
}
```

Note the layered delegation: `denyNameStartsWith` → `deny(Predicate)` → anonymous
`MeterFilter`. Higher-level factories delegate to lower-level ones, reducing code
duplication.

**denyUnless — the whitelist pattern**

This inverts the logic: NEUTRAL for matching meters (letting later filters or the default
decide), DENY for everything else:

```java
static MeterFilter denyUnless(Predicate<Meter.Id> iff) {
    return new MeterFilter() {
        @Override
        public MeterFilterReply accept(Meter.Id id) {
            return iff.test(id) ? MeterFilterReply.NEUTRAL : MeterFilterReply.DENY;
        }
    };
}
```

This is a subtle but important distinction from `accept(predicate)`. With `accept(predicate)`,
non-matching meters get NEUTRAL (they might still be accepted by default). With `denyUnless`,
non-matching meters get DENY — they are actively rejected.

**Static factory: renameTag**

Tag manipulation requires rebuilding the tag list:

```java
static MeterFilter renameTag(String meterNamePrefix, String fromTagKey, String toTagKey) {
    return new MeterFilter() {
        @Override
        public Meter.Id map(Meter.Id id) {
            if (!id.getName().startsWith(meterNamePrefix)) return id;
            String value = id.getTag(fromTagKey);
            if (value == null) return id;
            List<Tag> newTags = new ArrayList<>();
            for (Tag tag : id.getTagsAsIterable()) {
                if (tag.getKey().equals(fromTagKey)) {
                    newTags.add(Tag.of(toTagKey, tag.getValue()));
                } else {
                    newTags.add(tag);
                }
            }
            return new Meter.Id(id.getName(), Tags.of(newTags),
                    id.getBaseUnit(), id.getDescription(), id.getType());
        }
    };
}
```

The `Tags.of(newTags)` call re-sorts the tags, which is important because the renamed key
may sort differently than the original.

### Layer 3: MeterRegistry — integrating the filter chain

This is the core modification. Three things change:

**1. Filter storage and Config inner class**

```java
private final List<MeterFilter> filters = new CopyOnWriteArrayList<>();
private final Config config = new Config();

public Config config() { return config; }

public class Config {
    public Config meterFilter(MeterFilter filter) {
        filters.add(Objects.requireNonNull(filter));
        return this;
    }
    public Config commonTags(String... tags) {
        return meterFilter(MeterFilter.commonTags(Tags.of(tags)));
    }
}
```

`CopyOnWriteArrayList` is chosen because filters are added rarely (during startup) but
read on every meter registration. Copy-on-write gives lock-free reads with safe concurrent
writes.

**2. Registration entry points now pass a noop factory**

```java
Counter counter(Meter.Id id) {
    return registerMeterIfNecessary(Counter.class, id,
            this::newCounter, NoopCounter::new);
}
```

Each registration method now provides a `noopFactory` — a function that creates the
appropriate noop implementation when a filter denies the meter.

**3. The filter pipeline in registerMeterIfNecessary**

```java
private <M extends Meter> M registerMeterIfNecessary(
        Class<M> meterClass, Meter.Id id,
        Function<Meter.Id, M> meterFactory,
        Function<Meter.Id, M> noopFactory) {

    // Step 1: Apply map() filters to transform the id
    Meter.Id mappedId = applyMapFilters(id);

    // Step 2: Apply accept() filters — denied meters get a noop
    if (!applyAcceptFilters(mappedId)) {
        return noopFactory.apply(mappedId);
    }

    // Step 3: Fast path — lock-free lookup with the mapped id
    Meter existing = meterMap.get(mappedId);
    if (existing != null) {
        return castOrThrow(meterClass, existing, mappedId);
    }

    // Step 4: Slow path — create under lock
    synchronized (meterMapLock) {
        existing = meterMap.get(mappedId);
        if (existing != null) {
            return castOrThrow(meterClass, existing, mappedId);
        }
        M newMeter = meterFactory.apply(mappedId);
        meterMap.put(mappedId, newMeter);
        return newMeter;
    }
}
```

Key design decisions:

- **Filters run before the meterMap lookup** — the mapped Id is used for deduplication.
  Two calls with the same pre-filter Id correctly resolve to the same meter.
- **Denied meters are not cached** — a fresh noop is returned each time. This keeps
  denied meters out of `getMeters()` and avoids polluting the meter map.
- **The noop factory receives the mapped Id** — so a denied meter's Id still reflects
  the filter transformations (e.g., common tags are present even on denied meters).

The two helper methods are straightforward pipeline loops:

```java
private Meter.Id applyMapFilters(Meter.Id id) {
    Meter.Id mappedId = id;
    for (MeterFilter filter : filters) {
        mappedId = filter.map(mappedId);
    }
    return mappedId;
}

private boolean applyAcceptFilters(Meter.Id id) {
    for (MeterFilter filter : filters) {
        MeterFilterReply reply = filter.accept(id);
        if (reply == MeterFilterReply.DENY) return false;
        if (reply == MeterFilterReply.ACCEPT) return true;
    }
    return true; // default: accept if all NEUTRAL
}
```

---

## 4. Try It Yourself

1. **Add `MeterFilter.maximumAllowableMetrics(int max)`** — a filter that accepts
   the first `max` unique meter Ids, then denies all subsequent ones. The real
   Micrometer has this for cardinality protection. Hint: use a `ConcurrentHashMap.newKeySet()`
   to track seen Ids.

2. **Add `MeterFilter.replaceTagValues(String tagKey, Function<String, String> replacement)`**
   — a filter that applies a function to all values of a given tag key. Useful for
   normalizing URL paths (e.g., `/users/123` → `/users/{id}`).

3. **What happens if you add a filter after meters are already registered?** In our
   simplified version, existing meters are unaffected — the filter only applies to
   future registrations. The real Micrometer marks existing meters as "stale" and
   re-evaluates them through the new filter chain on the next access.

---

## 5. Why This Works

### Two default methods, not three

The real Micrometer's `MeterFilter` has a third method: `configure(Meter.Id, DistributionStatisticConfig)`,
which controls histogram boundaries, percentiles, and SLOs for timers and distribution
summaries. We omit this because our simplified meters don't support configurable distribution
statistics yet. This is a natural enhancement point for a future feature.

### NEUTRAL enables composition

A two-state (ACCEPT/DENY) system would force every filter to make a definitive decision.
With NEUTRAL, filters compose cleanly:
- A `commonTags` filter only transforms — it returns NEUTRAL from `accept()`.
- A `denyNameStartsWith("jvm")` filter denies JVM metrics and returns NEUTRAL for
  everything else, letting later filters decide.
- Only an explicit `accept()` or `deny()` (without predicate) makes a definitive call.

This three-state design is a form of the **Chain of Responsibility** pattern — each handler
can process, pass, or reject.

### CopyOnWriteArrayList vs volatile array

The real Micrometer uses a `volatile MeterFilter[]` with manual copy-on-write (creating a
new array for each `add`). We use `CopyOnWriteArrayList`, which does the same thing
internally but wraps it in a standard `List` interface. Both are optimized for the same
access pattern: filters are added rarely (during startup) but read frequently (on every
meter registration). The performance difference is negligible for this use case.

### Filters run once, not on every access

A critical performance property: filters transform the Id **at registration time**, not on
every `increment()` or `record()` call. Once a meter is created with its mapped Id, every
subsequent operation on that meter is zero-overhead. This is why Micrometer can add common
tags to 10,000 metrics without measurable latency impact on the hot path.

### No pre-filter cache

The real Micrometer maintains a `preFilterIdToMeterMap` that caches the mapping from
original (pre-filter) Ids to meters. This avoids re-running the filter chain when the
same meter is looked up repeatedly. Our simplified version re-runs the filter chain each
time, which is correct but slightly slower for repeated lookups. The performance difference
is negligible because the ConcurrentHashMap lookup after filtering is still O(1).

---

## 6. What We Enhanced

| Component | State before (Feature 4) | Enhancement in Feature 5 | Will be enhanced by |
|-----------|-------------------------|--------------------------|---------------------|
| `MeterRegistry` | Direct registration — no filter chain; single `registerMeterIfNecessary` parameter set | +`filters` list, +`Config` inner class, +`config()` method, +`applyMapFilters()`, +`applyAcceptFilters()`, +`noopFactory` parameter, +noop imports for all meter types | F6: +NamingConvention, F7: +CompositeMeterRegistry |
| Registration entry points | `counter(id)`, `gauge(id,...)`, `timer(id)`, `summary(id)` passed only a meter factory | Now also pass a noop factory (`NoopCounter::new`, `NoopGauge::new`, etc.) | Stable |
| Existing Noop classes | Used only for type completeness | Now actively returned by the filter chain when meters are denied | Stable |

---

## 7. Connection to Real Framework

| Simplified Class | Real Framework Class | Key Simplification |
|-----------------|---------------------|--------------------|
| `api/MeterFilter` | `io.micrometer.core.instrument.config.MeterFilter` | No `configure(Id, DistributionStatisticConfig)` method (no histogram/percentile/SLO config), no `maximumAllowableMetrics()`, no `maximumAllowableTags()`, no `replaceTagValues()`, no `maxExpected()`/`minExpected()`, no `forMeters()` conditional delegation |
| `api/MeterFilterReply` | `io.micrometer.core.instrument.config.MeterFilterReply` | Identical — same three values (DENY, NEUTRAL, ACCEPT) |
| `api/MeterRegistry` (modified) | `io.micrometer.core.instrument.MeterRegistry` | No `preFilterIdToMeterMap` cache, no stale-Id tracking for late-added filters, no `meterAddedListeners`, filters stored in `CopyOnWriteArrayList` instead of `volatile MeterFilter[]`, `Config` is simpler (no `onMeterAdded`, `onMeterRemoved`, `namingConvention`) |
| `api/MeterRegistry.Config` | `io.micrometer.core.instrument.MeterRegistry.Config` | No `namingConvention()`, no `onMeterAdded()`, no `onMeterRemoved()`, no `pause detector` configuration |

---

## 8. Complete Code

All files created or modified in this feature. Copy all `[NEW]` files and replace
all `[MODIFIED]` files to build the project from Features 1–4 + Feature 5.

### `src/main/java/simple/micrometer/api/MeterFilterReply.java` [NEW]

```java
package simple.micrometer.api;

/**
 * The outcome of a {@link MeterFilter#accept(Meter.Id)} decision.
 *
 * <p>Filters are evaluated in order. The first non-{@link #NEUTRAL} reply wins:
 * <ul>
 *   <li>{@link #DENY} — reject the meter (a noop implementation is returned)</li>
 *   <li>{@link #ACCEPT} — accept the meter (short-circuits remaining filters)</li>
 *   <li>{@link #NEUTRAL} — no opinion; pass to the next filter</li>
 * </ul>
 * If all filters return {@code NEUTRAL}, the meter is accepted by default.
 *
 * <p>Simplified from {@code io.micrometer.core.instrument.config.MeterFilterReply}.
 */
public enum MeterFilterReply {

    /**
     * The meter should be excluded — a noop meter is returned instead.
     */
    DENY,

    /**
     * No opinion — defer to the next filter in the chain.
     */
    NEUTRAL,

    /**
     * The meter should be included — short-circuits remaining filters.
     */
    ACCEPT

}
```

### `src/main/java/simple/micrometer/api/MeterFilter.java` [NEW]

```java
package simple.micrometer.api;

import java.util.ArrayList;
import java.util.List;
import java.util.Set;
import java.util.function.Predicate;

/**
 * A mechanism for globally transforming meter identities and controlling which
 * meters are registered. Filters are added to a registry via
 * {@link MeterRegistry.Config#meterFilter(MeterFilter)} and are applied in
 * insertion order during meter registration.
 *
 * <p>Each filter can:
 * <ul>
 *   <li>{@link #map(Meter.Id)} — transform the meter's name and/or tags</li>
 *   <li>{@link #accept(Meter.Id)} — accept or deny meter registration</li>
 * </ul>
 *
 * <p>The interface also provides static factory methods for common filter
 * patterns (adding common tags, denying by name prefix, renaming tags, etc.).
 *
 * <p>Simplified from {@code io.micrometer.core.instrument.config.MeterFilter}.
 */
public interface MeterFilter {

    // ── Default instance methods (the contract) ─────────────────────

    /**
     * Transforms the meter's identity during registration. Called for every
     * filter in order; each receives the output of the previous filter.
     *
     * <p>The default implementation returns the id unchanged.
     *
     * @param id the meter identity (possibly already transformed by earlier filters)
     * @return the (possibly transformed) identity
     */
    default Meter.Id map(Meter.Id id) {
        return id;
    }

    /**
     * Determines whether a meter should be registered (real) or rejected (noop).
     * Called after all {@link #map} transformations have been applied.
     *
     * <p>The default implementation returns {@link MeterFilterReply#NEUTRAL}.
     *
     * @param id the (post-map) meter identity
     * @return ACCEPT, DENY, or NEUTRAL
     */
    default MeterFilterReply accept(Meter.Id id) {
        return MeterFilterReply.NEUTRAL;
    }

    // ── Static factory: common tags ─────────────────────────────────

    /**
     * Returns a filter that prepends common tags to every meter. Meter-specific
     * tags with the same key are NOT overridden — they take precedence.
     *
     * <pre>{@code
     * registry.config().meterFilter(MeterFilter.commonTags(Tags.of("app", "my-service")));
     * }</pre>
     */
    static MeterFilter commonTags(Iterable<Tag> tags) {
        Tags commonTagSet = Tags.of(tags);
        return new MeterFilter() {
            @Override
            public Meter.Id map(Meter.Id id) {
                // Common tags are the base; meter-specific tags override on key conflict
                Tags merged = commonTagSet.and(id.getTagsAsIterable());
                return new Meter.Id(id.getName(), merged,
                        id.getBaseUnit(), id.getDescription(), id.getType());
            }
        };
    }

    // ── Static factories: accept ────────────────────────────────────

    /**
     * Returns a filter that unconditionally accepts all meters.
     * Useful for short-circuiting the filter chain.
     */
    static MeterFilter accept() {
        return new MeterFilter() {
            @Override
            public MeterFilterReply accept(Meter.Id id) {
                return MeterFilterReply.ACCEPT;
            }
        };
    }

    /**
     * Returns a filter that accepts meters matching the predicate;
     * returns NEUTRAL for non-matching meters.
     */
    static MeterFilter accept(Predicate<Meter.Id> iff) {
        return new MeterFilter() {
            @Override
            public MeterFilterReply accept(Meter.Id id) {
                return iff.test(id) ? MeterFilterReply.ACCEPT : MeterFilterReply.NEUTRAL;
            }
        };
    }

    /**
     * Returns a filter that accepts meters whose name starts with the
     * given prefix; returns NEUTRAL for others.
     */
    static MeterFilter acceptNameStartsWith(String prefix) {
        return accept(id -> id.getName().startsWith(prefix));
    }

    // ── Static factories: deny ──────────────────────────────────────

    /**
     * Returns a filter that unconditionally denies all meters.
     */
    static MeterFilter deny() {
        return new MeterFilter() {
            @Override
            public MeterFilterReply accept(Meter.Id id) {
                return MeterFilterReply.DENY;
            }
        };
    }

    /**
     * Returns a filter that denies meters matching the predicate;
     * returns NEUTRAL for non-matching meters.
     */
    static MeterFilter deny(Predicate<Meter.Id> iff) {
        return new MeterFilter() {
            @Override
            public MeterFilterReply accept(Meter.Id id) {
                return iff.test(id) ? MeterFilterReply.DENY : MeterFilterReply.NEUTRAL;
            }
        };
    }

    /**
     * Returns a filter that denies meters whose name starts with the
     * given prefix; returns NEUTRAL for others.
     *
     * <pre>{@code
     * registry.config().meterFilter(MeterFilter.denyNameStartsWith("jvm.gc"));
     * }</pre>
     */
    static MeterFilter denyNameStartsWith(String prefix) {
        return deny(id -> id.getName().startsWith(prefix));
    }

    /**
     * Whitelist filter: returns NEUTRAL when predicate matches (allowing
     * later filters or the default to decide), and DENY for everything else.
     *
     * <pre>{@code
     * // Only allow "http." metrics
     * registry.config().meterFilter(MeterFilter.denyUnless(
     *     id -> id.getName().startsWith("http.")));
     * }</pre>
     */
    static MeterFilter denyUnless(Predicate<Meter.Id> iff) {
        return new MeterFilter() {
            @Override
            public MeterFilterReply accept(Meter.Id id) {
                return iff.test(id) ? MeterFilterReply.NEUTRAL : MeterFilterReply.DENY;
            }
        };
    }

    // ── Static factories: tag manipulation ──────────────────────────

    /**
     * Returns a filter that renames a tag key for meters whose name starts
     * with {@code meterNamePrefix}. The tag value is preserved.
     *
     * <pre>{@code
     * registry.config().meterFilter(
     *     MeterFilter.renameTag("http", "status", "http.status"));
     * }</pre>
     */
    static MeterFilter renameTag(String meterNamePrefix, String fromTagKey, String toTagKey) {
        return new MeterFilter() {
            @Override
            public Meter.Id map(Meter.Id id) {
                if (!id.getName().startsWith(meterNamePrefix)) {
                    return id;
                }
                String value = id.getTag(fromTagKey);
                if (value == null) {
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
                return new Meter.Id(id.getName(), Tags.of(newTags),
                        id.getBaseUnit(), id.getDescription(), id.getType());
            }
        };
    }

    /**
     * Returns a filter that strips tags with the specified keys from every meter.
     *
     * <pre>{@code
     * registry.config().meterFilter(MeterFilter.ignoreTags("host", "instance"));
     * }</pre>
     */
    static MeterFilter ignoreTags(String... tagKeys) {
        Set<String> keysToIgnore = Set.of(tagKeys);
        return new MeterFilter() {
            @Override
            public Meter.Id map(Meter.Id id) {
                List<Tag> filtered = new ArrayList<>();
                for (Tag tag : id.getTagsAsIterable()) {
                    if (!keysToIgnore.contains(tag.getKey())) {
                        filtered.add(tag);
                    }
                }
                return new Meter.Id(id.getName(), Tags.of(filtered),
                        id.getBaseUnit(), id.getDescription(), id.getType());
            }
        };
    }

}
```

### `src/main/java/simple/micrometer/api/MeterRegistry.java` [MODIFIED]

```java
package simple.micrometer.api;

import simple.micrometer.internal.NoopCounter;
import simple.micrometer.internal.NoopDistributionSummary;
import simple.micrometer.internal.NoopGauge;
import simple.micrometer.internal.NoopTimer;

import java.util.ArrayList;
import java.util.Collections;
import java.util.List;
import java.util.Objects;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.CopyOnWriteArrayList;
import java.util.function.Function;
import java.util.function.ToDoubleFunction;

/**
 * The central hub that manages meter lifecycle — creation, deduplication,
 * storage, and retrieval. Concrete subclasses (like {@code SimpleMeterRegistry})
 * decide which implementation classes to instantiate for each meter type.
 *
 * <p>Registration uses double-checked locking: the first lookup is lock-free
 * via a {@link ConcurrentHashMap}, and the synchronized block is only entered
 * when creating a new meter.
 *
 * <p>Before a meter is created, any registered {@link MeterFilter}s are applied:
 * first {@link MeterFilter#map(Meter.Id)} transforms the identity (adding common
 * tags, renaming, etc.), then {@link MeterFilter#accept(Meter.Id)} decides whether
 * the meter should be real or a noop.
 *
 * <p>Simplified from {@code io.micrometer.core.instrument.MeterRegistry}.
 */
public abstract class MeterRegistry {

    private final ConcurrentHashMap<Meter.Id, Meter> meterMap = new ConcurrentHashMap<>();
    private final Object meterMapLock = new Object();
    private final List<MeterFilter> filters = new CopyOnWriteArrayList<>();
    private final Config config = new Config();

    protected final Clock clock;

    protected MeterRegistry(Clock clock) {
        this.clock = Objects.requireNonNull(clock, "clock must not be null");
    }

    // ── Configuration ─────────────────────────────────────────────────

    /**
     * Returns this registry's configuration handle, through which
     * {@link MeterFilter}s and common tags can be added.
     */
    public Config config() {
        return config;
    }

    // ── Public counter convenience methods ───────────────────────────

    /**
     * Shorthand: creates or retrieves a counter with the given name and
     * alternating key-value tag strings.
     *
     * <pre>{@code
     * Counter c = registry.counter("cache.misses", "cache", "users");
     * }</pre>
     */
    public Counter counter(String name, String... tags) {
        return counter(name, Tags.of(tags));
    }

    /**
     * Creates or retrieves a counter with the given name and tags.
     */
    public Counter counter(String name, Iterable<Tag> tags) {
        Meter.Id id = new Meter.Id(name, Tags.of(tags), null, null, Meter.Type.COUNTER);
        return counter(id);
    }

    // ── Public timer convenience methods ──────────────────────────────

    /**
     * Shorthand: creates or retrieves a timer with the given name and
     * alternating key-value tag strings.
     *
     * <pre>{@code
     * Timer t = registry.timer("http.latency", "method", "GET");
     * }</pre>
     */
    public Timer timer(String name, String... tags) {
        return timer(name, Tags.of(tags));
    }

    /**
     * Creates or retrieves a timer with the given name and tags.
     */
    public Timer timer(String name, Iterable<Tag> tags) {
        Meter.Id id = new Meter.Id(name, Tags.of(tags), null, null, Meter.Type.TIMER);
        return timer(id);
    }

    // ── Public summary convenience methods ──────────────────────────

    /**
     * Shorthand: creates or retrieves a distribution summary with the given
     * name and alternating key-value tag strings.
     *
     * <pre>{@code
     * DistributionSummary s = registry.summary("payload.size", "uri", "/api");
     * }</pre>
     */
    public DistributionSummary summary(String name, String... tags) {
        return summary(name, Tags.of(tags));
    }

    /**
     * Creates or retrieves a distribution summary with the given name and tags.
     */
    public DistributionSummary summary(String name, Iterable<Tag> tags) {
        Meter.Id id = new Meter.Id(name, Tags.of(tags), null, null, Meter.Type.DISTRIBUTION_SUMMARY);
        return summary(id);
    }

    // ── Public gauge convenience methods ─────────────────────────────

    /**
     * Creates or retrieves a gauge that observes {@code stateObject} through
     * {@code valueFunction}. Returns the state object (not the Gauge) for
     * fluent assignment patterns:
     *
     * <pre>{@code
     * AtomicInteger active = registry.gauge("connections", Tags.empty(),
     *     new AtomicInteger(0), AtomicInteger::doubleValue);
     * }</pre>
     *
     * @return the {@code stateObject} passed in (for assignment convenience)
     */
    public <T> T gauge(String name, Iterable<Tag> tags, T stateObject, ToDoubleFunction<T> valueFunction) {
        Gauge.builder(name, stateObject, valueFunction).tags(tags).register(this);
        return stateObject;
    }

    /**
     * Shorthand for gauging a {@link Number} subclass (AtomicInteger, AtomicLong, etc.)
     * using its {@link Number#doubleValue()}.
     */
    public <T extends Number> T gauge(String name, Iterable<Tag> tags, T number) {
        return gauge(name, tags, number, Number::doubleValue);
    }

    /**
     * Shorthand: gauge a Number without tags.
     */
    public <T extends Number> T gauge(String name, T number) {
        return gauge(name, Collections.emptyList(), number);
    }

    /**
     * Shorthand: gauge an object without tags.
     */
    public <T> T gauge(String name, T stateObject, ToDoubleFunction<T> valueFunction) {
        return gauge(name, Collections.emptyList(), stateObject, valueFunction);
    }

    // ── Package-private registration entry points ─────────────────────

    /**
     * Called by {@link Counter.Builder#register} and the convenience methods above.
     */
    Counter counter(Meter.Id id) {
        return registerMeterIfNecessary(Counter.class, id, this::newCounter, NoopCounter::new);
    }

    /**
     * Called by {@link Gauge.Builder#register} and the convenience methods above.
     */
    <T> Gauge gauge(Meter.Id id, T obj, ToDoubleFunction<T> valueFunction) {
        return registerMeterIfNecessary(Gauge.class, id,
                i -> newGauge(i, obj, valueFunction), NoopGauge::new);
    }

    /**
     * Called by {@link Timer.Builder#register} and the convenience methods above.
     */
    Timer timer(Meter.Id id) {
        return registerMeterIfNecessary(Timer.class, id, this::newTimer, NoopTimer::new);
    }

    /**
     * Called by {@link DistributionSummary.Builder#register} and the convenience methods above.
     */
    DistributionSummary summary(Meter.Id id) {
        return registerMeterIfNecessary(DistributionSummary.class, id,
                this::newDistributionSummary, NoopDistributionSummary::new);
    }

    // ── Abstract factory methods (subclasses implement these) ────────

    /**
     * Creates a new Counter implementation for the given id.
     * Called once per unique meter id, inside a synchronized block.
     */
    protected abstract Counter newCounter(Meter.Id id);

    /**
     * Creates a new Gauge implementation for the given id.
     * Called once per unique meter id, inside a synchronized block.
     *
     * @param id             the gauge's identity
     * @param obj            the state object to observe (may be held via WeakReference)
     * @param valueFunction  extracts a double from the state object
     */
    protected abstract <T> Gauge newGauge(Meter.Id id, T obj, ToDoubleFunction<T> valueFunction);

    /**
     * Creates a new Timer implementation for the given id.
     * Called once per unique meter id, inside a synchronized block.
     */
    protected abstract Timer newTimer(Meter.Id id);

    /**
     * Creates a new DistributionSummary implementation for the given id.
     * Called once per unique meter id, inside a synchronized block.
     */
    protected abstract DistributionSummary newDistributionSummary(Meter.Id id);

    // ── Query ────────────────────────────────────────────────────────

    /**
     * Returns an unmodifiable snapshot of all registered meters.
     */
    public List<Meter> getMeters() {
        return Collections.unmodifiableList(new ArrayList<>(meterMap.values()));
    }

    /**
     * Returns the clock used by this registry.
     */
    public Clock getClock() {
        return clock;
    }

    // ── Core registration logic ──────────────────────────────────────

    /**
     * Applies the filter chain and then performs double-checked locking
     * registration. The flow is:
     * <ol>
     *   <li>Apply all {@link MeterFilter#map} transformations to the id</li>
     *   <li>Apply all {@link MeterFilter#accept} decisions — if denied,
     *       return a noop meter</li>
     *   <li>Look up the mapped id in the meter map</li>
     *   <li>If not found, create under lock via the meter factory</li>
     * </ol>
     *
     * @param meterClass    expected concrete type (for type-safety cast)
     * @param id            the meter's identity (pre-filter)
     * @param meterFactory  creates the concrete meter (e.g., {@code this::newCounter})
     * @param noopFactory   creates a noop meter for denied registrations
     * @return the registered (or pre-existing) meter, cast to M
     * @throws IllegalArgumentException if an existing meter has the same id
     *                                  but a different type
     */
    private <M extends Meter> M registerMeterIfNecessary(
            Class<M> meterClass, Meter.Id id,
            Function<Meter.Id, M> meterFactory,
            Function<Meter.Id, M> noopFactory) {

        // Step 1: Apply map() filters to transform the id
        Meter.Id mappedId = applyMapFilters(id);

        // Step 2: Apply accept() filters — denied meters get a noop
        if (!applyAcceptFilters(mappedId)) {
            return noopFactory.apply(mappedId);
        }

        // Step 3: Fast path — lock-free lookup with the mapped id
        Meter existing = meterMap.get(mappedId);
        if (existing != null) {
            return castOrThrow(meterClass, existing, mappedId);
        }

        // Step 4: Slow path — create under lock
        synchronized (meterMapLock) {
            existing = meterMap.get(mappedId);
            if (existing != null) {
                return castOrThrow(meterClass, existing, mappedId);
            }

            M newMeter = meterFactory.apply(mappedId);
            meterMap.put(mappedId, newMeter);
            return newMeter;
        }
    }

    /**
     * Runs every filter's {@link MeterFilter#map} in insertion order.
     * Each filter receives the output of the previous one.
     */
    private Meter.Id applyMapFilters(Meter.Id id) {
        Meter.Id mappedId = id;
        for (MeterFilter filter : filters) {
            mappedId = filter.map(mappedId);
        }
        return mappedId;
    }

    /**
     * Runs every filter's {@link MeterFilter#accept} in insertion order.
     * The first non-{@link MeterFilterReply#NEUTRAL} reply wins.
     * If all filters return NEUTRAL, the meter is accepted by default.
     *
     * @return true if the meter should be registered, false if denied
     */
    private boolean applyAcceptFilters(Meter.Id id) {
        for (MeterFilter filter : filters) {
            MeterFilterReply reply = filter.accept(id);
            if (reply == MeterFilterReply.DENY) {
                return false;
            }
            if (reply == MeterFilterReply.ACCEPT) {
                return true;
            }
            // NEUTRAL → continue to next filter
        }
        return true; // default: accept if all filters are neutral
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

    // ── Config inner class ──────────────────────────────────────────

    /**
     * Configuration handle for a {@link MeterRegistry}. Provides methods
     * to add {@link MeterFilter}s and common tags.
     *
     * <pre>{@code
     * registry.config()
     *     .commonTags("app", "order-service", "env", "prod")
     *     .meterFilter(MeterFilter.denyNameStartsWith("jvm.gc"));
     * }</pre>
     */
    public class Config {

        /**
         * Adds a meter filter to this registry. Filters are applied in
         * insertion order during meter registration.
         *
         * @param filter the filter to add
         * @return this Config for chaining
         */
        public Config meterFilter(MeterFilter filter) {
            filters.add(Objects.requireNonNull(filter, "filter must not be null"));
            return this;
        }

        /**
         * Adds common tags that will be applied to every meter registered
         * with this registry. Meter-specific tags with the same key take
         * precedence over common tags.
         *
         * <p>This is a shorthand for
         * {@code meterFilter(MeterFilter.commonTags(Tags.of(tags)))}.
         *
         * @param tags alternating key-value strings
         * @return this Config for chaining
         */
        public Config commonTags(String... tags) {
            return meterFilter(MeterFilter.commonTags(Tags.of(tags)));
        }

        /**
         * Adds common tags that will be applied to every meter registered
         * with this registry.
         *
         * @param tags the common tags to add
         * @return this Config for chaining
         */
        public Config commonTags(Iterable<Tag> tags) {
            return meterFilter(MeterFilter.commonTags(tags));
        }

    }

}
```

### `src/test/java/simple/micrometer/api/MeterFilterTest.java` [NEW]

```java
package simple.micrometer.api;

import simple.micrometer.internal.NoopCounter;
import simple.micrometer.internal.NoopDistributionSummary;
import simple.micrometer.internal.NoopGauge;
import simple.micrometer.internal.NoopTimer;
import simple.micrometer.internal.SimpleMeterRegistry;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Nested;
import org.junit.jupiter.api.Test;

import java.time.Duration;
import java.util.concurrent.atomic.AtomicInteger;

import static org.assertj.core.api.Assertions.assertThat;

/**
 * Client-perspective tests for the MeterFilter API.
 * These tests exercise the filter chain exactly as a framework user would —
 * through the public API surface of MeterRegistry.Config and MeterFilter.
 */
class MeterFilterTest {

    private MeterRegistry registry;

    @BeforeEach
    void setUp() {
        registry = new SimpleMeterRegistry();
    }

    // ── Common tags ─────────────────────────────────────────────────

    @Nested
    class CommonTags {

        @Test
        void shouldAddCommonTags_ToEveryMeter() {
            registry.config().commonTags("app", "order-service", "env", "prod");

            Counter counter = registry.counter("http.requests", "method", "GET");

            assertThat(counter.getId().getTag("app")).isEqualTo("order-service");
            assertThat(counter.getId().getTag("env")).isEqualTo("prod");
            assertThat(counter.getId().getTag("method")).isEqualTo("GET");
        }

        @Test
        void shouldNotOverrideMeterSpecificTags_WhenCommonTagHasSameKey() {
            registry.config().commonTags("env", "prod");

            Counter counter = Counter.builder("http.requests")
                    .tag("env", "staging")
                    .register(registry);

            // Meter-specific tag wins over common tag
            assertThat(counter.getId().getTag("env")).isEqualTo("staging");
        }

        @Test
        void shouldApplyCommonTags_ToTimers() {
            registry.config().commonTags("region", "us-east-1");

            Timer timer = registry.timer("http.latency", "method", "GET");

            assertThat(timer.getId().getTag("region")).isEqualTo("us-east-1");
            assertThat(timer.getId().getTag("method")).isEqualTo("GET");
        }

        @Test
        void shouldApplyCommonTags_ToGauges() {
            registry.config().commonTags("region", "us-east-1");

            AtomicInteger value = new AtomicInteger(42);
            Gauge gauge = Gauge.builder("queue.size", value, AtomicInteger::doubleValue)
                    .register(registry);

            assertThat(gauge.getId().getTag("region")).isEqualTo("us-east-1");
        }

        @Test
        void shouldApplyCommonTags_ToDistributionSummaries() {
            registry.config().commonTags("region", "us-east-1");

            DistributionSummary summary = registry.summary("payload.size", "uri", "/api");

            assertThat(summary.getId().getTag("region")).isEqualTo("us-east-1");
            assertThat(summary.getId().getTag("uri")).isEqualTo("/api");
        }

        @Test
        void shouldStackMultipleCommonTagFilters() {
            registry.config()
                    .commonTags("app", "order-service")
                    .commonTags("env", "prod");

            Counter counter = registry.counter("events");

            assertThat(counter.getId().getTag("app")).isEqualTo("order-service");
            assertThat(counter.getId().getTag("env")).isEqualTo("prod");
        }

        @Test
        void shouldDeduplicateMeters_WithCommonTags() {
            registry.config().commonTags("app", "my-app");

            Counter c1 = registry.counter("requests", "method", "GET");
            Counter c2 = registry.counter("requests", "method", "GET");

            assertThat(c1).isSameAs(c2);
        }
    }

    // ── Deny filters ────────────────────────────────────────────────

    @Nested
    class DenyFilters {

        @Test
        void shouldDenyAllMeters_WithDenyFilter() {
            registry.config().meterFilter(MeterFilter.deny());

            Counter counter = registry.counter("http.requests");

            assertThat(counter).isInstanceOf(NoopCounter.class);
            assertThat(counter.count()).isEqualTo(0.0);
            counter.increment(); // should be a no-op
            assertThat(counter.count()).isEqualTo(0.0);
        }

        @Test
        void shouldDenyByNamePrefix() {
            registry.config().meterFilter(MeterFilter.denyNameStartsWith("jvm.gc"));

            Counter denied = registry.counter("jvm.gc.pause");
            Counter allowed = registry.counter("http.requests");

            assertThat(denied).isInstanceOf(NoopCounter.class);
            assertThat(allowed).isNotInstanceOf(NoopCounter.class);
        }

        @Test
        void shouldDenyByPredicate() {
            registry.config().meterFilter(
                    MeterFilter.deny(id -> id.getName().contains("internal")));

            Counter denied = registry.counter("internal.cache.hits");
            Counter allowed = registry.counter("http.requests");

            assertThat(denied).isInstanceOf(NoopCounter.class);
            assertThat(allowed).isNotInstanceOf(NoopCounter.class);
        }

        @Test
        void shouldDenyTimers() {
            registry.config().meterFilter(MeterFilter.denyNameStartsWith("internal"));

            Timer denied = registry.timer("internal.processing");

            assertThat(denied).isInstanceOf(NoopTimer.class);
        }

        @Test
        void shouldDenyGauges() {
            registry.config().meterFilter(MeterFilter.denyNameStartsWith("internal"));

            AtomicInteger value = new AtomicInteger(42);
            Gauge denied = Gauge.builder("internal.queue", value, AtomicInteger::doubleValue)
                    .register(registry);

            assertThat(denied).isInstanceOf(NoopGauge.class);
        }

        @Test
        void shouldDenyDistributionSummaries() {
            registry.config().meterFilter(MeterFilter.denyNameStartsWith("internal"));

            DistributionSummary denied = registry.summary("internal.payload.size");

            assertThat(denied).isInstanceOf(NoopDistributionSummary.class);
        }

        @Test
        void shouldNotStoreDeniedMeters_InRegistry() {
            registry.config().meterFilter(MeterFilter.denyNameStartsWith("jvm"));

            registry.counter("jvm.gc.pause");
            registry.counter("http.requests");

            assertThat(registry.getMeters()).hasSize(1);
            assertThat(registry.getMeters().get(0).getId().getName()).isEqualTo("http.requests");
        }
    }

    // ── Accept filters ──────────────────────────────────────────────

    @Nested
    class AcceptFilters {

        @Test
        void shouldAcceptAll_WithAcceptFilter() {
            // Even with a later deny filter, accept short-circuits
            registry.config()
                    .meterFilter(MeterFilter.accept())
                    .meterFilter(MeterFilter.deny());

            Counter counter = registry.counter("anything");

            assertThat(counter).isNotInstanceOf(NoopCounter.class);
        }

        @Test
        void shouldAcceptByNamePrefix() {
            registry.config()
                    .meterFilter(MeterFilter.acceptNameStartsWith("http."))
                    .meterFilter(MeterFilter.deny());

            Counter accepted = registry.counter("http.requests");
            Counter denied = registry.counter("jvm.gc");

            assertThat(accepted).isNotInstanceOf(NoopCounter.class);
            assertThat(denied).isInstanceOf(NoopCounter.class);
        }

        @Test
        void shouldAcceptByPredicate() {
            registry.config()
                    .meterFilter(MeterFilter.accept(id -> id.getTag("critical") != null))
                    .meterFilter(MeterFilter.deny());

            Counter accepted = registry.counter("low.level", "critical", "true");
            Counter denied = registry.counter("low.level.other");

            assertThat(accepted).isNotInstanceOf(NoopCounter.class);
            assertThat(denied).isInstanceOf(NoopCounter.class);
        }
    }

    // ── denyUnless (whitelist) ───────────────────────────────────────

    @Nested
    class DenyUnless {

        @Test
        void shouldOnlyAllowMatchingMeters() {
            registry.config().meterFilter(
                    MeterFilter.denyUnless(id -> id.getName().startsWith("http.")));

            Counter allowed = registry.counter("http.requests");
            Counter denied = registry.counter("jvm.gc.pause");

            assertThat(allowed).isNotInstanceOf(NoopCounter.class);
            assertThat(denied).isInstanceOf(NoopCounter.class);
        }
    }

    // ── Rename tag ──────────────────────────────────────────────────

    @Nested
    class RenameTag {

        @Test
        void shouldRenameTagKey_WhenNameMatchesPrefix() {
            registry.config().meterFilter(
                    MeterFilter.renameTag("http", "status", "http.status"));

            Counter counter = registry.counter("http.requests",
                    "status", "200", "method", "GET");

            assertThat(counter.getId().getTag("http.status")).isEqualTo("200");
            assertThat(counter.getId().getTag("status")).isNull();
            assertThat(counter.getId().getTag("method")).isEqualTo("GET");
        }

        @Test
        void shouldNotRenameTag_WhenNameDoesNotMatchPrefix() {
            registry.config().meterFilter(
                    MeterFilter.renameTag("http", "status", "http.status"));

            Counter counter = registry.counter("cache.requests", "status", "hit");

            // "cache.requests" doesn't start with "http", so no rename
            assertThat(counter.getId().getTag("status")).isEqualTo("hit");
            assertThat(counter.getId().getTag("http.status")).isNull();
        }

        @Test
        void shouldNotRenameTag_WhenTagKeyNotPresent() {
            registry.config().meterFilter(
                    MeterFilter.renameTag("http", "status", "http.status"));

            Counter counter = registry.counter("http.requests", "method", "GET");

            // No "status" tag → no rename
            assertThat(counter.getId().getTag("method")).isEqualTo("GET");
            assertThat(counter.getId().getTag("http.status")).isNull();
        }
    }

    // ── Ignore tags ─────────────────────────────────────────────────

    @Nested
    class IgnoreTags {

        @Test
        void shouldStripSpecifiedTags() {
            registry.config().meterFilter(MeterFilter.ignoreTags("host", "instance"));

            Counter counter = registry.counter("http.requests",
                    "method", "GET", "host", "server-1", "instance", "i-123");

            assertThat(counter.getId().getTag("method")).isEqualTo("GET");
            assertThat(counter.getId().getTag("host")).isNull();
            assertThat(counter.getId().getTag("instance")).isNull();
        }

        @Test
        void shouldNotAffectUnspecifiedTags() {
            registry.config().meterFilter(MeterFilter.ignoreTags("host"));

            Counter counter = registry.counter("http.requests",
                    "method", "GET", "status", "200");

            assertThat(counter.getId().getTag("method")).isEqualTo("GET");
            assertThat(counter.getId().getTag("status")).isEqualTo("200");
        }
    }

    // ── Custom filter (implementing the interface) ──────────────────

    @Nested
    class CustomFilter {

        @Test
        void shouldSupportCustomAcceptLogic() {
            registry.config().meterFilter(new MeterFilter() {
                @Override
                public MeterFilterReply accept(Meter.Id id) {
                    if (id.getName().startsWith("internal.")) {
                        return MeterFilterReply.DENY;
                    }
                    return MeterFilterReply.NEUTRAL;
                }
            });

            Counter denied = registry.counter("internal.cache");
            Counter allowed = registry.counter("http.requests");

            assertThat(denied).isInstanceOf(NoopCounter.class);
            assertThat(allowed).isNotInstanceOf(NoopCounter.class);
        }

        @Test
        void shouldSupportCustomMapLogic() {
            registry.config().meterFilter(new MeterFilter() {
                @Override
                public Meter.Id map(Meter.Id id) {
                    // Prefix all meter names with "myapp."
                    return id.withName("myapp." + id.getName());
                }
            });

            Counter counter = registry.counter("requests");

            assertThat(counter.getId().getName()).isEqualTo("myapp.requests");
        }
    }

    // ── Filter ordering ─────────────────────────────────────────────

    @Nested
    class FilterOrdering {

        @Test
        void shouldApplyMapFilters_InInsertionOrder() {
            registry.config()
                    .meterFilter(new MeterFilter() {
                        @Override
                        public Meter.Id map(Meter.Id id) {
                            return id.withName(id.getName() + ".first");
                        }
                    })
                    .meterFilter(new MeterFilter() {
                        @Override
                        public Meter.Id map(Meter.Id id) {
                            return id.withName(id.getName() + ".second");
                        }
                    });

            Counter counter = registry.counter("base");

            assertThat(counter.getId().getName()).isEqualTo("base.first.second");
        }

        @Test
        void shouldShortCircuit_OnFirstDeny() {
            registry.config()
                    .meterFilter(MeterFilter.deny())
                    .meterFilter(MeterFilter.accept()); // never reached

            Counter counter = registry.counter("anything");

            assertThat(counter).isInstanceOf(NoopCounter.class);
        }

        @Test
        void shouldShortCircuit_OnFirstAccept() {
            registry.config()
                    .meterFilter(MeterFilter.accept())
                    .meterFilter(MeterFilter.deny()); // never reached

            Counter counter = registry.counter("anything");

            assertThat(counter).isNotInstanceOf(NoopCounter.class);
        }

        @Test
        void shouldApplyMapBeforeAccept() {
            // Rename first, then deny based on the renamed name
            registry.config()
                    .meterFilter(new MeterFilter() {
                        @Override
                        public Meter.Id map(Meter.Id id) {
                            if (id.getName().equals("old.name")) {
                                return id.withName("internal.old.name");
                            }
                            return id;
                        }
                    })
                    .meterFilter(MeterFilter.denyNameStartsWith("internal."));

            Counter counter = registry.counter("old.name");

            // Was renamed to "internal.old.name", then denied
            assertThat(counter).isInstanceOf(NoopCounter.class);
        }

        @Test
        void shouldApplyCommonTagsBeforeDenyFilter() {
            // Add common tags, then deny based on a tag value
            registry.config()
                    .commonTags("env", "test")
                    .meterFilter(MeterFilter.deny(
                            id -> "test".equals(id.getTag("env"))));

            Counter counter = registry.counter("http.requests");

            // Common tag "env=test" was added, then the deny filter matched it
            assertThat(counter).isInstanceOf(NoopCounter.class);
        }
    }

    // ── Meters still function correctly after filtering ─────────────

    @Nested
    class FilteredMeterBehavior {

        @Test
        void shouldCountCorrectly_WhenCommonTagsApplied() {
            registry.config().commonTags("app", "my-service");

            Counter counter = registry.counter("requests");
            counter.increment();
            counter.increment(4.0);

            assertThat(counter.count()).isEqualTo(5.0);
        }

        @Test
        void shouldTimeCorrectly_WhenCommonTagsApplied() {
            registry.config().commonTags("app", "my-service");

            Timer timer = registry.timer("latency");
            timer.record(Duration.ofMillis(100));

            assertThat(timer.count()).isEqualTo(1);
            assertThat(timer.totalTime(java.util.concurrent.TimeUnit.MILLISECONDS))
                    .isCloseTo(100.0, org.assertj.core.data.Offset.offset(0.5));
        }

        @Test
        void shouldRecordDistribution_WhenCommonTagsApplied() {
            registry.config().commonTags("app", "my-service");

            DistributionSummary summary = registry.summary("payload");
            summary.record(100);
            summary.record(200);

            assertThat(summary.count()).isEqualTo(2);
            assertThat(summary.totalAmount()).isEqualTo(300.0);
        }
    }

}
```
