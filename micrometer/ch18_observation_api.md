# Chapter 18: Observation API

> Line references based on commit `2c8a4606c` of the Micrometer repository.

## Build Challenge

| Current State | Limitation | Objective |
|---------------|------------|-----------|
| 17 chapters of metrics infrastructure — counters, timers, gauges, registries, push/step export, filters, naming conventions. Each meter type has its own builder pattern and requires direct interaction with `MeterRegistry`. | Every instrumentation point requires separate calls for metrics, tracing, and logging. Adding a timer means touching `MeterRegistry` directly; adding a span means touching a tracing library; adding a log means touching a logger. Three instrumentation calls for one operation. Changing what you measure means editing three places. | Build an `Observation` API — a single, higher-level instrumentation point with a lifecycle (`start` → `openScope` → `stop`) and pluggable handlers. Instrument once; let handlers decide what to do with the data. |

---

## 18.1 The Integration Point: A New Parallel API Layer

Unlike previous features that plugged INTO the existing `MeterRegistry`, the Observation API sits **above** the entire meter system as a parallel API layer. The integration comes in Feature 19 (the bridge) — for now, the Observation API is self-contained.

The foundation of this feature is the **`ObservationHandler` interface** — the extension point that all future bridges (metrics, tracing, logging) will implement:

**New file:** `src/main/java/dev/linhvu/micrometer/observation/ObservationHandler.java`

```java
public interface ObservationHandler<T extends Observation.Context> {

    default void onStart(T context) { }
    default void onError(T context) { }
    default void onEvent(Observation.Event event, T context) { }
    default void onScopeOpened(T context) { }
    default void onScopeClosed(T context) { }
    default void onStop(T context) { }

    boolean supportsContext(Observation.Context context);
}
```

**Direction:** The handler is the seam where this API connects to EVERYTHING else. A metrics handler will create `Timer.Sample`s on `onStart()` and stop them on `onStop()`. A tracing handler will create Spans. A logging handler will write log entries. The observation itself doesn't know or care — it just announces lifecycle events.

Two key decisions:

1. **Why type-parameterize on Context?** Different observation types carry different data (HTTP requests vs. messaging vs. database calls). A handler for `HttpContext` shouldn't receive `MessagingContext`. The `supportsContext()` method enables this type-safe dispatch — handlers only receive lifecycle calls for contexts they understand.

2. **Why are all lifecycle methods default no-ops?** A handler only needs to implement the events it cares about. A metrics handler may only need `onStart()`/`onStop()` to bracket a timer. A tracing handler may also need `onScopeOpened()`/`onScopeClosed()` for ThreadLocal propagation. Neither needs to implement all seven methods.

To make this work, we also need:
- `KeyValue` / `KeyValues` — dimensional labels with low/high cardinality separation
- `Observation` interface — the lifecycle API with Context, Scope, and Event inner types
- `ObservationRegistry` — manages handlers, predicates, conventions, and filters
- `SimpleObservation` — the lifecycle orchestrator
- `ObservationConvention` — naming/key-value strategy pattern

## 18.2 KeyValue and KeyValues — Dimensional Labels for Observations

**New file:** `src/main/java/dev/linhvu/micrometer/observation/KeyValue.java`

```java
public interface KeyValue extends Comparable<KeyValue> {

    String NONE_VALUE = "none";

    String getKey();
    String getValue();

    static KeyValue of(String key, String value) {
        return new ImmutableKeyValue(key, value);
    }

    @Override
    default int compareTo(KeyValue o) {
        return getKey().compareTo(o.getKey());
    }
}
```

**New file:** `src/main/java/dev/linhvu/micrometer/observation/ImmutableKeyValue.java`

```java
public record ImmutableKeyValue(String key, String value) implements KeyValue {

    public ImmutableKeyValue {
        if (key == null || key.isBlank()) {
            throw new IllegalArgumentException("KeyValue key must not be null or blank");
        }
        if (value == null) {
            throw new IllegalArgumentException("KeyValue value must not be null");
        }
    }

    @Override
    public String getKey() { return key; }

    @Override
    public String getValue() { return value; }
}
```

**New file:** `src/main/java/dev/linhvu/micrometer/observation/KeyValues.java`

`KeyValues` is an immutable, sorted, deduplicated collection — the same design as `Tags` from Chapter 1. Sorted by key, last-value-wins on conflicts, O(n+m) merge.

```java
public final class KeyValues implements Iterable<KeyValue> {

    private static final KeyValues EMPTY = new KeyValues(new KeyValue[0], 0);

    private final KeyValue[] sortedSet;
    private final int length;

    public static KeyValues empty() { return EMPTY; }
    public static KeyValues of(String key, String value) { /* ... */ }
    public static KeyValues of(KeyValue... keyValues) { /* sort + dedup */ }
    public static KeyValues of(Iterable<? extends KeyValue> keyValues) { /* ... */ }

    public KeyValues and(String key, String value) { /* merge */ }
    public KeyValues and(KeyValue... keyValues) { /* merge */ }
    public boolean isEmpty() { return length == 0; }
    public Stream<KeyValue> stream() { /* ... */ }
}
```

Why separate `KeyValue`/`KeyValues` from `Tag`/`Tags`? They live in different conceptual layers. Tags are metrics-specific (they become Prometheus labels or Datadog tags). KeyValues are observation-level — they carry additional metadata like cardinality level that Tags don't have. The separation keeps the observation layer independent of the metrics layer.

## 18.3 Observation Interface — The Lifecycle API

**New file:** `src/main/java/dev/linhvu/micrometer/observation/Observation.java`

The `Observation` interface is the primary API. It defines the lifecycle, factory methods, key-value methods, and inner types.

### Factory Methods

```java
public interface Observation {

    Observation NOOP = new Observation() { /* all methods are no-ops */ };

    static Observation createNotStarted(String name, ObservationRegistry registry) {
        return createNotStarted(name, Context::new, registry);
    }

    static <T extends Context> Observation createNotStarted(
            String name, Supplier<T> contextSupplier, ObservationRegistry registry) {
        if (registry == null || registry.isNoop()) {
            return NOOP;  // Avoids context creation — performance optimization
        }
        T context = contextSupplier.get();
        context.setParentFromCurrentObservation(registry);
        if (!registry.observationConfig().isObservationEnabled(name, context)) {
            return NOOP;
        }
        return new SimpleObservation(name, registry, context);
    }

    // Convention-aware factory with 3-tier resolution
    static <T extends Context> Observation createNotStarted(
            ObservationConvention<T> customConvention,
            ObservationConvention<T> defaultConvention,
            Supplier<T> contextSupplier,
            ObservationRegistry registry) { /* ... */ }

    static Observation start(String name, ObservationRegistry registry) {
        return createNotStarted(name, registry).start();
    }
}
```

### Lifecycle Methods

```java
    Observation start();
    Scope openScope();
    void stop();
    Observation error(Throwable error);
    Observation event(Event event);
```

### Key Value Methods (Fluent API)

```java
    Observation lowCardinalityKeyValue(KeyValue keyValue);
    default Observation lowCardinalityKeyValue(String key, String value) { /* ... */ }
    Observation highCardinalityKeyValue(KeyValue keyValue);
    default Observation highCardinalityKeyValue(String key, String value) { /* ... */ }
    Observation contextualName(String contextualName);
    Observation parentObservation(Observation parentObservation);
```

### Convenience Methods

```java
    default void observe(Runnable runnable) {
        start();
        try (Scope scope = openScope()) {
            runnable.run();
        } catch (Throwable error) {
            error(error);
            throw error;
        } finally {
            stop();
        }
    }

    default <T> T observe(Supplier<T> supplier) {
        start();
        try (Scope scope = openScope()) {
            return supplier.get();
        } catch (Throwable error) {
            error(error);
            throw error;
        } finally {
            stop();
        }
    }
```

### Inner Types

**Scope** — represents the observation being "in scope" (current on this thread):

```java
    interface Scope extends AutoCloseable {
        Scope NOOP = new Scope() { /* no-ops */ };
        Observation getCurrentObservation();
        void close();
        default boolean isNoop() { return this == NOOP; }
    }
```

**Event** — an arbitrary event during the observation's lifetime:

```java
    interface Event {
        static Event of(String name) { return of(name, name); }
        static Event of(String name, String contextualName) {
            return new SimpleEvent(name, contextualName);
        }
        String getName();
        String getContextualName();
    }
```

**Context** — mutable data holder that handlers read and write:

```java
    class Context {
        private String name;
        private String contextualName;
        private Throwable error;
        private Observation parentObservation;
        private final Map<String, KeyValue> lowCardinalityKeyValues = new ConcurrentHashMap<>();
        private final Map<String, KeyValue> highCardinalityKeyValues = new ConcurrentHashMap<>();
        private final Map<Object, Object> map = new HashMap<>();

        // Name, error, parent accessors...

        public Context addLowCardinalityKeyValue(KeyValue keyValue) {
            lowCardinalityKeyValues.put(keyValue.getKey(), keyValue);
            return this;
        }

        public KeyValues getLowCardinalityKeyValues() {
            return KeyValues.of(lowCardinalityKeyValues.values());
        }

        // Generic map for handler data: put(), get(), getRequired(), computeIfAbsent()
        public <V> Context put(Object key, V value) {
            map.put(key, value);
            return this;
        }

        public <V> V get(Object key) {
            return (V) map.get(key);
        }
    }
```

## 18.4 ObservationRegistry, SimpleObservationRegistry, and Supporting Types

### ObservationRegistry

**New file:** `src/main/java/dev/linhvu/micrometer/observation/ObservationRegistry.java`

```java
public interface ObservationRegistry {

    ObservationRegistry NOOP = new ObservationRegistry() { /* no-ops */ };

    static ObservationRegistry create() {
        return new SimpleObservationRegistry();
    }

    Observation getCurrentObservation();
    Observation.Scope getCurrentObservationScope();
    void setCurrentObservationScope(Observation.Scope scope);
    ObservationConfig observationConfig();

    default boolean isNoop() { return this == NOOP; }
}
```

### ObservationConfig (inner class)

The configuration hub with four CopyOnWriteArrayList-backed registries:

```java
    class ObservationConfig {
        private final List<ObservationHandler<?>> handlers = new CopyOnWriteArrayList<>();
        private final List<ObservationPredicate> predicates = new CopyOnWriteArrayList<>();
        private final List<GlobalObservationConvention<?>> conventions = new CopyOnWriteArrayList<>();
        private final List<ObservationFilter> filters = new CopyOnWriteArrayList<>();

        public ObservationConfig observationHandler(ObservationHandler<?> handler) { /* ... */ }
        public ObservationConfig observationPredicate(ObservationPredicate predicate) { /* ... */ }
        public ObservationConfig observationConvention(GlobalObservationConvention<?> convention) { /* ... */ }
        public ObservationConfig observationFilter(ObservationFilter filter) { /* ... */ }

        public boolean isObservationEnabled(String name, Observation.Context context) {
            for (ObservationPredicate predicate : predicates) {
                if (!predicate.test(name, context)) return false;
            }
            return true;
        }

        public <T extends Observation.Context> ObservationConvention<T> getObservationConvention(
                T context, ObservationConvention<T> defaultConvention) {
            for (GlobalObservationConvention<?> convention : conventions) {
                if (convention.supportsContext(context)) return (ObservationConvention<T>) convention;
            }
            return defaultConvention;
        }
    }
```

### SimpleObservationRegistry

**New file:** `src/main/java/dev/linhvu/micrometer/observation/SimpleObservationRegistry.java`

```java
public class SimpleObservationRegistry implements ObservationRegistry {

    private final ThreadLocal<Observation.Scope> localObservationScope = new ThreadLocal<>();
    private final ObservationConfig config = new ObservationConfig();

    @Override
    public Observation getCurrentObservation() {
        Observation.Scope scope = localObservationScope.get();
        return scope != null ? scope.getCurrentObservation() : null;
    }

    @Override
    public Observation.Scope getCurrentObservationScope() {
        return localObservationScope.get();
    }

    @Override
    public void setCurrentObservationScope(Observation.Scope scope) {
        localObservationScope.set(scope);
    }

    @Override
    public boolean isNoop() {
        return config.getObservationHandlers().isEmpty();
    }
}
```

### ObservationConvention and GlobalObservationConvention

**New files:** `ObservationConvention.java` and `GlobalObservationConvention.java`

```java
public interface ObservationConvention<T extends Observation.Context> {
    default KeyValues getLowCardinalityKeyValues(T context) { return KeyValues.empty(); }
    default KeyValues getHighCardinalityKeyValues(T context) { return KeyValues.empty(); }
    boolean supportsContext(Observation.Context context);
    default String getName() { return null; }
    default String getContextualName(T context) { return null; }
}

// Marker interface for conventions registered globally
public interface GlobalObservationConvention<T extends Observation.Context>
        extends ObservationConvention<T> { }
```

### ObservationFilter and ObservationPredicate

**New files:** `ObservationFilter.java` and `ObservationPredicate.java`

```java
@FunctionalInterface
public interface ObservationFilter {
    Observation.Context map(Observation.Context context);
}

@FunctionalInterface
public interface ObservationPredicate extends BiPredicate<String, Observation.Context> {
    boolean test(String name, Observation.Context context);
}
```

## 18.5 SimpleObservation — The Lifecycle Orchestrator

**New file:** `src/main/java/dev/linhvu/micrometer/observation/SimpleObservation.java`

```java
public class SimpleObservation implements Observation {

    private final ObservationRegistry registry;
    private final Context context;
    private final ObservationConvention<Context> convention;
    private final Deque<ObservationHandler<Context>> handlers;
    private final Collection<ObservationFilter> filters;

    SimpleObservation(String name, ObservationRegistry registry, Context context) {
        this.registry = registry;
        this.context = context;
        this.context.setName(name);
        this.convention = findConventionFromConfig(context, registry);
        this.handlers = getHandlersFromConfig(context, registry);
        this.filters = registry.observationConfig().getObservationFilters();
    }

    @Override
    public Observation start() {
        if (convention != null) {
            context.addLowCardinalityKeyValues(convention.getLowCardinalityKeyValues(context));
            context.addHighCardinalityKeyValues(convention.getHighCardinalityKeyValues(context));
        }
        notifyOnObservationStarted();  // Forward order
        return this;
    }

    @Override
    public void stop() {
        if (convention != null) {
            // Convention may produce different key values at stop time
            context.addLowCardinalityKeyValues(convention.getLowCardinalityKeyValues(context));
            context.addHighCardinalityKeyValues(convention.getHighCardinalityKeyValues(context));
        }
        Context ctx = context;
        for (ObservationFilter filter : filters) {
            ctx = filter.map(ctx);
        }
        notifyOnObservationStopped(ctx);  // REVERSE order
    }

    @Override
    public Scope openScope() {
        SimpleScope scope = new SimpleScope(registry, this);
        notifyOnScopeOpened();
        return scope;
    }
}
```

### SimpleScope — Linked-List Scope Stack

```java
    static class SimpleScope implements Scope {
        private final ObservationRegistry registry;
        private final SimpleObservation observation;
        private final Scope previousObservationScope;

        SimpleScope(ObservationRegistry registry, SimpleObservation observation) {
            this.registry = registry;
            this.observation = observation;
            this.previousObservationScope = registry.getCurrentObservationScope();
            registry.setCurrentObservationScope(this);
        }

        @Override
        public void close() {
            observation.notifyOnScopeClosed();
            registry.setCurrentObservationScope(previousObservationScope);
        }
    }
```

### Composite Handlers

```java
    // Delegates to FIRST matching handler only
    class FirstMatchingCompositeObservationHandler
            implements ObservationHandler<Observation.Context> {
        private final List<ObservationHandler<Observation.Context>> handlers;

        @Override
        public void onStart(Observation.Context context) {
            ObservationHandler<Observation.Context> handler = firstMatching(context);
            if (handler != null) handler.onStart(context);
        }
        // ... same pattern for all lifecycle methods
    }

    // Delegates to ALL matching handlers
    class AllMatchingCompositeObservationHandler
            implements ObservationHandler<Observation.Context> {
        private final List<ObservationHandler<Observation.Context>> handlers;

        @Override
        public void onStart(Observation.Context context) {
            for (ObservationHandler<Observation.Context> handler : handlers) {
                if (handler.supportsContext(context)) handler.onStart(context);
            }
        }
        // ... same pattern for all lifecycle methods
    }
```

---

## 18.6 Try It Yourself

<details><summary>Challenge 1: Implement the observe(Runnable) convenience method</summary>

The `observe()` method wraps the full lifecycle — start, open scope, run, handle errors, stop:

```java
default void observe(Runnable runnable) {
    start();
    try (Scope scope = openScope()) {
        runnable.run();
    } catch (Throwable error) {
        error(error);
        throw error;
    } finally {
        stop();
    }
}
```

Key points:
- `start()` before `openScope()` — scope requires a started observation
- `openScope()` in try-with-resources — guarantees cleanup
- `error()` before re-throwing — handlers see the error before stop
- `stop()` in `finally` — always runs, even on error

</details>

<details><summary>Challenge 2: Implement the SimpleScope linked-list stack</summary>

The scope creates a linked list of scopes via the registry's ThreadLocal:

```java
static class SimpleScope implements Scope {
    private final ObservationRegistry registry;
    private final SimpleObservation observation;
    private final Scope previousObservationScope;

    SimpleScope(ObservationRegistry registry, SimpleObservation observation) {
        this.registry = registry;
        this.observation = observation;
        // Save what was current BEFORE us
        this.previousObservationScope = registry.getCurrentObservationScope();
        // Set ourselves as current
        registry.setCurrentObservationScope(this);
    }

    @Override
    public void close() {
        observation.notifyOnScopeClosed();
        // Restore what was current before us
        registry.setCurrentObservationScope(previousObservationScope);
    }
}
```

</details>

<details><summary>Challenge 3: Implement the 3-tier convention resolution</summary>

Convention resolution order: custom > global > default:

```java
static <T extends Context> Observation createNotStarted(
        ObservationConvention<T> customConvention,
        ObservationConvention<T> defaultConvention,
        Supplier<T> contextSupplier,
        ObservationRegistry registry) {
    if (registry == null || registry.isNoop()) return NOOP;
    T context = contextSupplier.get();
    context.setParentFromCurrentObservation(registry);

    ObservationConvention<T> resolved;
    if (customConvention != null) {
        resolved = customConvention;                    // 1. Custom wins
    } else {
        resolved = registry.observationConfig()
                .getObservationConvention(context, defaultConvention);  // 2. Global, then 3. Default
    }
    // ... create SimpleObservation with resolved convention
}
```

</details>

---

## 18.7 Tests

### Unit Tests

| Test | Verifies |
|------|----------|
| `shouldCreateKeyValue_WithKeyAndValue` | KeyValue factory method |
| `shouldCompareByKey_IgnoringValue` | Natural ordering by key only |
| `shouldRejectNullKey` | Validation on construction |
| `shouldSortByKey` | KeyValues sorts entries alphabetically |
| `shouldDeduplicateByKey_LastWins` | Last-value-wins deduplication |
| `shouldMergeWithAnd` | Immutable merge of two KeyValues |
| `shouldNotifyHandler_WhenObservationStarted` | Handler receives onStart |
| `shouldNotifyInReverseOrder_WhenStopped` | Symmetric cleanup (3→2→1) |
| `shouldOpenAndCloseScope` | Scope sets current observation |
| `shouldRestorePreviousScope_WhenClosingNestedScope` | Linked-list scope stack |
| `shouldRecordError` | Error stored on context, handler notified |
| `shouldSignalEvent` | Event forwarded to handlers |
| `shouldAddLowCardinalityKeyValues` | Low-cardinality key values stored on context |
| `shouldAddHighCardinalityKeyValues` | High-cardinality key values stored on context |
| `shouldSetContextualName` | Contextual name overrides display name |
| `shouldAutoWireParent_FromCurrentScope` | Parent auto-wired from scope |
| `shouldRunObserveRunnable` | Full lifecycle via observe(Runnable) |
| `shouldRunObserveSupplier` | Full lifecycle via observe(Supplier) |
| `shouldRecordError_WhenObserveThrows` | Error handling in observe() |
| `shouldReturnNoop_WhenRegistryIsNull` | Null-safe factory |
| `shouldReturnNoop_WhenRegistryHasNoHandlers` | Empty registry optimization |
| `shouldOnlyCallFirstMatching_WhenUsingFirstMatchingComposite` | First-match composite |
| `shouldCallAllMatching_WhenUsingAllMatchingComposite` | All-match composite |
| `shouldFilterByContextType_WhenUsingTypedHandler` | Type-safe handler dispatch |
| `shouldApplyConventionKeyValues_WhenStarted` | Convention key values applied |
| `shouldApplyContextualName_FromConventionAtStopTime` | Convention contextual name at stop |
| `shouldDisableObservation_WhenPredicateReturnsFalse` | Predicate-based disable |
| `shouldApplyFilter_BeforeStop` | Filter mutates context before stop |
| `shouldResolveGlobalConvention_WhenRegistered` | Global convention resolution |
| `shouldUseCustomConvention_WhenProvidedOverGlobal` | Custom beats global |

---

## 18.8 Why This Works

> `★ Insight ─────────────────────────────────────`
> **The Context as a Handler Mailbox:**
> The generic `put()`/`get()` map on `Context` is what makes the handler pattern work across unrelated concerns. A metrics handler stores `Timer.Sample` under a `TimerSample.class` key on `onStart()`, then retrieves it on `onStop()` to record the duration. A tracing handler stores a `Span` under a `Span.class` key. Neither handler knows about the other — they communicate with themselves across time via the context. This is essentially the "Carrier" pattern from distributed tracing, applied within a single process.
> `─────────────────────────────────────────────────`

> `★ Insight ─────────────────────────────────────`
> **Low vs. High Cardinality — Why It Matters for Metrics:**
> An HTTP endpoint that records `user.id` as a metric tag would create a unique time series for every user — potentially millions. At $0.01/series/month (typical monitoring pricing), that's a bill in the tens of thousands. The low/high cardinality split forces this decision at instrumentation time: `lowCardinalityKeyValue("method", "GET")` becomes a metric tag (bounded, safe), while `highCardinalityKeyValue("user.id", "12345")` becomes only a trace attribute (one-off, stored per-request). The Observation API makes this invisible to handler authors — the metrics bridge only reads low-cardinality values.
> `─────────────────────────────────────────────────`

> `★ Insight ─────────────────────────────────────`
> **Why Reverse-Order Stop Notification:**
> If handlers A, B, C are notified on `start()` in order A→B→C, they are notified on `stop()` in order C→B→A. This matches try-with-resources semantics: the last resource opened is the first to close. It matters when handlers have ordering dependencies — a tracing handler might need to close its span (handler C) before the metrics handler records the final timer value (handler A), because the span's duration affects what gets recorded.
> `─────────────────────────────────────────────────`

---

## 18.9 What We Enhanced

| Component | Enhancement | Why |
|-----------|-------------|-----|
| (none) | N/A — this feature is a new parallel API layer | The Observation API sits above the metrics system. No existing files were modified. The bridge to metrics comes in Feature 19. |

---

## 18.10 Connection to Real Framework

| Simplified | Real Micrometer | Location |
|-----------|-----------------|----------|
| `Observation` interface | `Observation` interface | `micrometer-observation/.../observation/Observation.java:1` |
| `Observation.Context` inner class | `Observation.Context` inner class | `Observation.java:992` |
| `Observation.Scope` inner interface | `Observation.Scope` inner interface | `Observation.java:901` |
| `Observation.Event` inner interface | `Observation.Event` inner interface | `Observation.java:849` |
| `Observation.NOOP` field | `Observation.NOOP` field | `Observation.java:65` |
| `ObservationRegistry` interface | `ObservationRegistry` interface | `micrometer-observation/.../observation/ObservationRegistry.java:1` |
| `ObservationConfig` inner class | `ObservationConfig` inner class | `ObservationRegistry.java` |
| `SimpleObservationRegistry` | `SimpleObservationRegistry` | `micrometer-observation/.../observation/SimpleObservationRegistry.java:1` |
| `ObservationHandler` interface | `ObservationHandler` interface | `micrometer-observation/.../observation/ObservationHandler.java:1` |
| `FirstMatchingCompositeObservationHandler` | `FirstMatchingCompositeObservationHandler` | `ObservationHandler.java` inner class |
| `AllMatchingCompositeObservationHandler` | `AllMatchingCompositeObservationHandler` | `ObservationHandler.java` inner class |
| `SimpleObservation` | `SimpleObservation` | `micrometer-observation/.../observation/SimpleObservation.java:1` |
| `SimpleObservation.SimpleScope` inner class | `SimpleObservation.SimpleScope` inner class | `SimpleObservation.java` inner class |
| `ObservationConvention` interface | `ObservationConvention` interface | `micrometer-observation/.../observation/ObservationConvention.java:1` |
| `GlobalObservationConvention` | `GlobalObservationConvention` | `micrometer-observation/.../observation/GlobalObservationConvention.java:1` |
| `ObservationFilter` | `ObservationFilter` | `micrometer-observation/.../observation/ObservationFilter.java:1` |
| `ObservationPredicate` | `ObservationPredicate` | `micrometer-observation/.../observation/ObservationPredicate.java:1` |
| `KeyValue` interface | `KeyValue` interface | `micrometer-commons/.../common/KeyValue.java:1` |
| `ImmutableKeyValue` record | `ImmutableKeyValue` record | `micrometer-commons/.../common/ImmutableKeyValue.java:1` |
| `KeyValues` collection | `KeyValues` collection | `micrometer-commons/.../common/KeyValues.java:1` |
| No `ObservationView`/`ContextView` | `ObservationView` / `ContextView` interfaces | Real version has read-only view interfaces for safety |
| No `NoopButScopeHandlingObservation` | `NoopButScopeHandlingObservation` | Real version handles scopes even when observation is disabled, for context propagation |
| No `scope.reset()`/`makeCurrent()` | `Scope.reset()` / `Scope.makeCurrent()` | Real version supports cross-thread context propagation |
| Per-instance ThreadLocal | Static ThreadLocal | Real version shares ThreadLocal across all registry instances |
| No checked exception variants | `CheckedRunnable`, `CheckedCallable` | Real version supports `observe()` with checked exceptions |

> Commit: `2c8a4606c`

---

## 18.11 Complete Code

### Production Code

#### File: `src/main/java/dev/linhvu/micrometer/observation/KeyValue.java` [NEW]

```java
package dev.linhvu.micrometer.observation;

/**
 * A key-value pair used as a dimensional label on an {@link Observation}.
 * <p>
 * Similar to {@link dev.linhvu.micrometer.Tag} in the metrics world, but lives in the
 * observation layer. Key values are split into two cardinality levels:
 * <ul>
 *   <li><b>Low cardinality:</b> bounded set of values (e.g., HTTP method: GET/POST).
 *       These become metric tags/dimensions.</li>
 *   <li><b>High cardinality:</b> unbounded set of values (e.g., user ID, request URL).
 *       These become trace span attributes only — never metric tags.</li>
 * </ul>
 * <p>
 * Natural ordering is by {@link #getKey()} only, matching the behavior of {@code Tag}.
 * <p>
 * <b>Simplification:</b> The real Micrometer's {@code KeyValue} lives in
 * {@code micrometer-commons} and supports {@code KeyName} enums for type-safe keys.
 * We keep it simple with string keys only.
 */
public interface KeyValue extends Comparable<KeyValue> {

    /**
     * Sentinel value used when no value is available.
     */
    String NONE_VALUE = "none";

    String getKey();

    String getValue();

    static KeyValue of(String key, String value) {
        return new ImmutableKeyValue(key, value);
    }

    @Override
    default int compareTo(KeyValue o) {
        return getKey().compareTo(o.getKey());
    }

}
```

#### File: `src/main/java/dev/linhvu/micrometer/observation/ImmutableKeyValue.java` [NEW]

```java
package dev.linhvu.micrometer.observation;

/**
 * Immutable implementation of {@link KeyValue} as a Java record.
 *
 * @param key   The key name (e.g., "http.method", "user.id").
 * @param value The value (e.g., "GET", "12345").
 */
public record ImmutableKeyValue(String key, String value) implements KeyValue {

    public ImmutableKeyValue {
        if (key == null || key.isBlank()) {
            throw new IllegalArgumentException("KeyValue key must not be null or blank");
        }
        if (value == null) {
            throw new IllegalArgumentException("KeyValue value must not be null");
        }
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
    public String toString() {
        return "tag(" + key + "=" + value + ")";
    }

}
```

#### File: `src/main/java/dev/linhvu/micrometer/observation/KeyValues.java` [NEW]

```java
package dev.linhvu.micrometer.observation;

import java.util.ArrayList;
import java.util.Arrays;
import java.util.Collection;
import java.util.Iterator;
import java.util.List;
import java.util.NoSuchElementException;
import java.util.Spliterator;
import java.util.Spliterators;
import java.util.stream.Collectors;
import java.util.stream.Stream;
import java.util.stream.StreamSupport;

/**
 * An immutable, sorted, deduplicated collection of {@link KeyValue}s.
 * <p>
 * Mirrors the design of {@link dev.linhvu.micrometer.Tags}: sorted by key,
 * last-value-wins on key conflicts, O(n+m) merge. The same sorted-array
 * backing gives memory efficiency and predictable iteration order.
 * <p>
 * <b>Simplification:</b> The real {@code KeyValues} is in {@code micrometer-commons}
 * and supports additional factory methods. We keep the essential operations.
 */
public final class KeyValues implements Iterable<KeyValue> {

    private static final KeyValue[] EMPTY_ARRAY = new KeyValue[0];

    private static final KeyValues EMPTY = new KeyValues(EMPTY_ARRAY, 0);

    private final KeyValue[] sortedSet;

    private final int length;

    private KeyValues(KeyValue[] sortedSet, int length) {
        this.sortedSet = sortedSet;
        this.length = length;
    }

    // -----------------------------------------------------------------------
    // Factory methods
    // -----------------------------------------------------------------------

    public static KeyValues empty() {
        return EMPTY;
    }

    public static KeyValues of(String key, String value) {
        return new KeyValues(new KeyValue[] { KeyValue.of(key, value) }, 1);
    }

    public static KeyValues of(KeyValue... keyValues) {
        if (keyValues == null || keyValues.length == 0) {
            return EMPTY;
        }
        return toKeyValues(keyValues.clone());
    }

    public static KeyValues of(Iterable<? extends KeyValue> keyValues) {
        if (keyValues instanceof KeyValues kvs) {
            return kvs;
        }
        if (keyValues instanceof Collection<? extends KeyValue> c) {
            if (c.isEmpty()) {
                return EMPTY;
            }
            return toKeyValues(c.toArray(EMPTY_ARRAY));
        }
        List<KeyValue> list = new ArrayList<>();
        keyValues.forEach(list::add);
        if (list.isEmpty()) {
            return EMPTY;
        }
        return toKeyValues(list.toArray(EMPTY_ARRAY));
    }

    // -----------------------------------------------------------------------
    // Merge methods
    // -----------------------------------------------------------------------

    public KeyValues and(String key, String value) {
        return and(KeyValue.of(key, value));
    }

    public KeyValues and(KeyValue... keyValues) {
        if (keyValues == null || keyValues.length == 0) {
            return this;
        }
        return merge(KeyValues.of(keyValues));
    }

    public KeyValues and(Iterable<? extends KeyValue> keyValues) {
        if (keyValues instanceof KeyValues other) {
            return merge(other);
        }
        return merge(KeyValues.of(keyValues));
    }

    // -----------------------------------------------------------------------
    // Query methods
    // -----------------------------------------------------------------------

    public boolean isEmpty() {
        return length == 0;
    }

    public Stream<KeyValue> stream() {
        return StreamSupport.stream(Spliterators.spliterator(sortedSet, 0, length,
                Spliterator.IMMUTABLE | Spliterator.ORDERED | Spliterator.DISTINCT
                        | Spliterator.NONNULL | Spliterator.SORTED),
                false);
    }

    @Override
    public Iterator<KeyValue> iterator() {
        return new ArrayIterator(sortedSet, length);
    }

    // -----------------------------------------------------------------------
    // equals / hashCode / toString
    // -----------------------------------------------------------------------

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof KeyValues other)) return false;
        if (this.length != other.length) return false;
        for (int i = 0; i < length; i++) {
            if (!sortedSet[i].equals(other.sortedSet[i])) return false;
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
        return stream().map(kv -> kv.getKey() + "=" + kv.getValue())
                .collect(Collectors.joining(",", "[", "]"));
    }

    // -----------------------------------------------------------------------
    // Internal algorithms (same as Tags)
    // -----------------------------------------------------------------------

    private static KeyValues toKeyValues(KeyValue[] kvs) {
        int len = kvs.length;
        if (len == 0) return EMPTY;
        if (len == 1) return new KeyValues(kvs, 1);
        if (isSortedSet(kvs, len)) return new KeyValues(kvs, len);
        Arrays.sort(kvs);
        return dedup(kvs);
    }

    private static boolean isSortedSet(KeyValue[] kvs, int length) {
        for (int i = 0; i < length - 1; i++) {
            if (kvs[i].compareTo(kvs[i + 1]) >= 0) return false;
        }
        return true;
    }

    private static KeyValues dedup(KeyValue[] sorted) {
        int len = sorted.length;
        int writeIdx = 0;
        for (int readIdx = 0; readIdx < len; readIdx++) {
            while (readIdx < len - 1 && sorted[readIdx].compareTo(sorted[readIdx + 1]) == 0) {
                readIdx++;
            }
            sorted[writeIdx++] = sorted[readIdx];
        }
        return new KeyValues(sorted, writeIdx);
    }

    private KeyValues merge(KeyValues other) {
        if (other.length == 0) return this;
        if (this.length == 0) return other;

        KeyValue[] merged = new KeyValue[this.length + other.length];
        int thisIdx = 0, otherIdx = 0, writeIdx = 0;

        while (thisIdx < this.length && otherIdx < other.length) {
            KeyValue thisKv = this.sortedSet[thisIdx];
            KeyValue otherKv = other.sortedSet[otherIdx];
            int cmp = thisKv.compareTo(otherKv);
            if (cmp < 0) {
                merged[writeIdx++] = thisKv;
                thisIdx++;
            } else if (cmp > 0) {
                merged[writeIdx++] = otherKv;
                otherIdx++;
            } else {
                merged[writeIdx++] = otherKv; // other wins
                thisIdx++;
                otherIdx++;
            }
        }
        while (thisIdx < this.length) merged[writeIdx++] = this.sortedSet[thisIdx++];
        while (otherIdx < other.length) merged[writeIdx++] = other.sortedSet[otherIdx++];

        return new KeyValues(merged, writeIdx);
    }

    private static class ArrayIterator implements Iterator<KeyValue> {

        private final KeyValue[] kvs;
        private final int length;
        private int index = 0;

        ArrayIterator(KeyValue[] kvs, int length) {
            this.kvs = kvs;
            this.length = length;
        }

        @Override
        public boolean hasNext() {
            return index < length;
        }

        @Override
        public KeyValue next() {
            if (!hasNext()) throw new NoSuchElementException();
            return kvs[index++];
        }

    }

}
```

#### File: `src/main/java/dev/linhvu/micrometer/observation/ObservationFilter.java` [NEW]

```java
package dev.linhvu.micrometer.observation;

/**
 * A filter that can mutate an {@link Observation.Context} just before
 * {@link Observation#stop()} notifies handlers.
 * <p>
 * Use this to add computed key values at the last moment — for example,
 * deriving a "status" key value from the error field on the context.
 * <p>
 * Filters run in registration order, each receiving the context returned
 * by the previous filter.
 */
@FunctionalInterface
public interface ObservationFilter {

    /**
     * Mutates or replaces the context before stop handlers are called.
     *
     * @param context The context to transform.
     * @return The (possibly modified) context.
     */
    Observation.Context map(Observation.Context context);

}
```

#### File: `src/main/java/dev/linhvu/micrometer/observation/ObservationPredicate.java` [NEW]

```java
package dev.linhvu.micrometer.observation;

import java.util.function.BiPredicate;

/**
 * A predicate that determines whether an observation should be created
 * or replaced with a noop.
 * <p>
 * When any registered predicate returns {@code false} for a given name
 * and context, the observation is disabled and {@link Observation#NOOP}
 * is returned instead.
 * <p>
 * Use cases:
 * <ul>
 *   <li>Disable observations for health check endpoints</li>
 *   <li>Disable observations below a certain sampling threshold</li>
 *   <li>Disable observations in specific contexts (e.g., internal-only)</li>
 * </ul>
 */
@FunctionalInterface
public interface ObservationPredicate extends BiPredicate<String, Observation.Context> {

    /**
     * Whether the observation with the given name and context should be created.
     *
     * @param name    The observation name.
     * @param context The observation context.
     * @return {@code true} to create the observation, {@code false} to disable it.
     */
    @Override
    boolean test(String name, Observation.Context context);

}
```

#### File: `src/main/java/dev/linhvu/micrometer/observation/ObservationConvention.java` [NEW]

```java
package dev.linhvu.micrometer.observation;

/**
 * Provides naming and key-value strategies for observations.
 * <p>
 * Conventions separate "what to name and tag" from "what to do with
 * the observation." A library author defines a default convention;
 * users can override it with a custom convention or register a
 * {@link GlobalObservationConvention} on the registry.
 * <p>
 * <b>3-tier resolution order:</b>
 * <ol>
 *   <li>Custom convention (passed to {@code createNotStarted()})</li>
 *   <li>Global convention (registered on {@code ObservationConfig})</li>
 *   <li>Default convention (passed to {@code createNotStarted()})</li>
 * </ol>
 *
 * @param <T> The type of context this convention applies to.
 */
public interface ObservationConvention<T extends Observation.Context> {

    /**
     * Returns low-cardinality key values for the given context.
     * These become metric tags/dimensions.
     */
    default KeyValues getLowCardinalityKeyValues(T context) {
        return KeyValues.empty();
    }

    /**
     * Returns high-cardinality key values for the given context.
     * These become trace span attributes only — never metric tags.
     */
    default KeyValues getHighCardinalityKeyValues(T context) {
        return KeyValues.empty();
    }

    /**
     * Whether this convention supports the given context type.
     * Used for type-safe dispatch when multiple conventions are registered.
     */
    boolean supportsContext(Observation.Context context);

    /**
     * Returns the observation name. When non-null and non-blank, overrides
     * the name passed to {@code createNotStarted()}.
     *
     * @return The name, or {@code null} to keep the original name.
     */
    default String getName() {
        return null;
    }

    /**
     * Returns a contextual name derived from the context (e.g., from an HTTP
     * request path). Applied at stop time.
     *
     * @return The contextual name, or {@code null} to keep the original.
     */
    default String getContextualName(T context) {
        return null;
    }

}
```

#### File: `src/main/java/dev/linhvu/micrometer/observation/GlobalObservationConvention.java` [NEW]

```java
package dev.linhvu.micrometer.observation;

/**
 * Marker interface for conventions that can be registered globally on
 * {@link ObservationRegistry.ObservationConfig}.
 * <p>
 * Only {@code GlobalObservationConvention} instances are accepted by
 * {@code config.observationConvention()}. This type distinction prevents
 * accidentally registering an inline convention as a global one.
 * <p>
 * In the 3-tier resolution order, global conventions sit between the
 * custom convention (highest priority) and the default convention
 * (lowest priority).
 *
 * @param <T> The type of context this convention applies to.
 */
public interface GlobalObservationConvention<T extends Observation.Context>
        extends ObservationConvention<T> {

}
```

#### File: `src/main/java/dev/linhvu/micrometer/observation/ObservationHandler.java` [NEW]

```java
package dev.linhvu.micrometer.observation;

import java.util.Arrays;
import java.util.List;

/**
 * Hooks into the {@link Observation} lifecycle to perform work at each stage.
 * <p>
 * This is the Strategy pattern: different handler implementations decide
 * WHAT happens at each lifecycle point (create a timer, start a span,
 * write a log entry), while the observation itself controls WHEN these
 * events occur.
 * <p>
 * Handlers are type-parameterized on {@code Context} — a handler for
 * {@code HttpContext} won't be called for a {@code MessagingContext}.
 * The {@link #supportsContext(Observation.Context)} method enables this
 * type-safe dispatch.
 * <p>
 * All lifecycle methods default to no-op, so implementations only need
 * to override the methods they care about.
 *
 * @param <T> The context type this handler operates on.
 */
public interface ObservationHandler<T extends Observation.Context> {

    /**
     * Called when {@link Observation#start()} is invoked.
     */
    default void onStart(T context) {
    }

    /**
     * Called when {@link Observation#error(Throwable)} is invoked.
     */
    default void onError(T context) {
    }

    /**
     * Called when {@link Observation#event(Observation.Event)} is invoked.
     */
    default void onEvent(Observation.Event event, T context) {
    }

    /**
     * Called when {@link Observation#openScope()} is invoked.
     */
    default void onScopeOpened(T context) {
    }

    /**
     * Called when {@link Observation.Scope#close()} is invoked.
     */
    default void onScopeClosed(T context) {
    }

    /**
     * Called when {@link Observation#stop()} is invoked.
     */
    default void onStop(T context) {
    }

    /**
     * Whether this handler supports the given context type.
     * Only handlers that return {@code true} will receive lifecycle callbacks
     * for that observation.
     */
    boolean supportsContext(Observation.Context context);

    // -----------------------------------------------------------------------
    // Composite handlers
    // -----------------------------------------------------------------------

    /**
     * Delegates each lifecycle call to the <b>first</b> handler whose
     * {@link #supportsContext(Observation.Context)} returns {@code true}.
     * <p>
     * Use this when you want exactly one handler to handle a given observation
     * (e.g., pick one metrics handler from several candidates).
     */
    class FirstMatchingCompositeObservationHandler
            implements ObservationHandler<Observation.Context> {

        private final List<ObservationHandler<Observation.Context>> handlers;

        @SafeVarargs
        public FirstMatchingCompositeObservationHandler(
                ObservationHandler<? extends Observation.Context>... handlers) {
            this(Arrays.asList(handlers));
        }

        @SuppressWarnings("unchecked")
        public FirstMatchingCompositeObservationHandler(
                List<ObservationHandler<? extends Observation.Context>> handlers) {
            this.handlers = handlers.stream()
                    .map(h -> (ObservationHandler<Observation.Context>) h)
                    .toList();
        }

        @Override
        public void onStart(Observation.Context context) {
            ObservationHandler<Observation.Context> handler = firstMatching(context);
            if (handler != null) handler.onStart(context);
        }

        @Override
        public void onError(Observation.Context context) {
            ObservationHandler<Observation.Context> handler = firstMatching(context);
            if (handler != null) handler.onError(context);
        }

        @Override
        public void onEvent(Observation.Event event, Observation.Context context) {
            ObservationHandler<Observation.Context> handler = firstMatching(context);
            if (handler != null) handler.onEvent(event, context);
        }

        @Override
        public void onScopeOpened(Observation.Context context) {
            ObservationHandler<Observation.Context> handler = firstMatching(context);
            if (handler != null) handler.onScopeOpened(context);
        }

        @Override
        public void onScopeClosed(Observation.Context context) {
            ObservationHandler<Observation.Context> handler = firstMatching(context);
            if (handler != null) handler.onScopeClosed(context);
        }

        @Override
        public void onStop(Observation.Context context) {
            ObservationHandler<Observation.Context> handler = firstMatching(context);
            if (handler != null) handler.onStop(context);
        }

        @Override
        public boolean supportsContext(Observation.Context context) {
            return firstMatching(context) != null;
        }

        private ObservationHandler<Observation.Context> firstMatching(
                Observation.Context context) {
            for (ObservationHandler<Observation.Context> handler : handlers) {
                if (handler.supportsContext(context)) {
                    return handler;
                }
            }
            return null;
        }

    }

    /**
     * Delegates each lifecycle call to <b>all</b> handlers whose
     * {@link #supportsContext(Observation.Context)} returns {@code true}.
     * <p>
     * Use this when you want multiple handlers to process the same observation
     * (e.g., both a metrics handler and a tracing handler).
     */
    class AllMatchingCompositeObservationHandler
            implements ObservationHandler<Observation.Context> {

        private final List<ObservationHandler<Observation.Context>> handlers;

        @SafeVarargs
        public AllMatchingCompositeObservationHandler(
                ObservationHandler<? extends Observation.Context>... handlers) {
            this(Arrays.asList(handlers));
        }

        @SuppressWarnings("unchecked")
        public AllMatchingCompositeObservationHandler(
                List<ObservationHandler<? extends Observation.Context>> handlers) {
            this.handlers = handlers.stream()
                    .map(h -> (ObservationHandler<Observation.Context>) h)
                    .toList();
        }

        @Override
        public void onStart(Observation.Context context) {
            for (ObservationHandler<Observation.Context> handler : handlers) {
                if (handler.supportsContext(context)) handler.onStart(context);
            }
        }

        @Override
        public void onError(Observation.Context context) {
            for (ObservationHandler<Observation.Context> handler : handlers) {
                if (handler.supportsContext(context)) handler.onError(context);
            }
        }

        @Override
        public void onEvent(Observation.Event event, Observation.Context context) {
            for (ObservationHandler<Observation.Context> handler : handlers) {
                if (handler.supportsContext(context)) handler.onEvent(event, context);
            }
        }

        @Override
        public void onScopeOpened(Observation.Context context) {
            for (ObservationHandler<Observation.Context> handler : handlers) {
                if (handler.supportsContext(context)) handler.onScopeOpened(context);
            }
        }

        @Override
        public void onScopeClosed(Observation.Context context) {
            for (ObservationHandler<Observation.Context> handler : handlers) {
                if (handler.supportsContext(context)) handler.onScopeClosed(context);
            }
        }

        @Override
        public void onStop(Observation.Context context) {
            for (ObservationHandler<Observation.Context> handler : handlers) {
                if (handler.supportsContext(context)) handler.onStop(context);
            }
        }

        @Override
        public boolean supportsContext(Observation.Context context) {
            for (ObservationHandler<Observation.Context> handler : handlers) {
                if (handler.supportsContext(context)) return true;
            }
            return false;
        }

    }

}
```

#### File: `src/main/java/dev/linhvu/micrometer/observation/ObservationRegistry.java` [NEW]

```java
package dev.linhvu.micrometer.observation;

import java.util.Collections;
import java.util.List;
import java.util.concurrent.CopyOnWriteArrayList;

/**
 * Registry that manages observation state — the current scope (via ThreadLocal)
 * and configuration (handlers, predicates, conventions, filters).
 * <p>
 * Create a registry with {@link #create()}, register handlers on its
 * {@link #observationConfig()}, then use it in {@link Observation#createNotStarted}.
 * <p>
 * <b>Noop semantics:</b> A registry with no handlers registered is treated as noop
 * — all observations will return {@link Observation#NOOP} without even creating a
 * context. This is a performance optimization for applications that haven't set up
 * observation yet.
 * <p>
 * <b>Simplification:</b> The real Micrometer has a static ThreadLocal shared across
 * all registry instances and separate noop config. We use a per-instance ThreadLocal
 * for simplicity, and a single static NOOP registry.
 */
public interface ObservationRegistry {

    /**
     * A no-op registry. All observations created against this registry will be
     * {@link Observation#NOOP}.
     */
    ObservationRegistry NOOP = new ObservationRegistry() {

        private final ObservationConfig config = new ObservationConfig();

        @Override
        public Observation getCurrentObservation() {
            return null;
        }

        @Override
        public Observation.Scope getCurrentObservationScope() {
            return null;
        }

        @Override
        public void setCurrentObservationScope(Observation.Scope scope) {
        }

        @Override
        public ObservationConfig observationConfig() {
            return config;
        }

        @Override
        public boolean isNoop() {
            return true;
        }
    };

    /**
     * Creates a new {@link SimpleObservationRegistry}.
     */
    static ObservationRegistry create() {
        return new SimpleObservationRegistry();
    }

    /**
     * Returns the observation currently in scope on this thread, or null if none.
     */
    Observation getCurrentObservation();

    /**
     * Returns the current observation scope on this thread.
     */
    Observation.Scope getCurrentObservationScope();

    /**
     * Sets the current observation scope on this thread.
     * Called internally by scope open/close — not part of the public API.
     */
    void setCurrentObservationScope(Observation.Scope scope);

    /**
     * Returns the configuration for this registry.
     */
    ObservationConfig observationConfig();

    /**
     * Whether this registry is a noop. When true, all observations will be
     * {@link Observation#NOOP}.
     */
    default boolean isNoop() {
        return this == NOOP;
    }

    // -----------------------------------------------------------------------
    // ObservationConfig
    // -----------------------------------------------------------------------

    /**
     * Configuration hub for the observation registry. Manages four lists:
     * <ul>
     *   <li>{@link ObservationHandler}s — lifecycle callbacks</li>
     *   <li>{@link ObservationPredicate}s — enable/disable observations</li>
     *   <li>{@link GlobalObservationConvention}s — naming/key-value strategies</li>
     *   <li>{@link ObservationFilter}s — pre-stop context mutation</li>
     * </ul>
     * <p>
     * All lists use {@link CopyOnWriteArrayList} for thread-safe registration.
     */
    class ObservationConfig {

        private final List<ObservationHandler<?>> handlers = new CopyOnWriteArrayList<>();

        private final List<ObservationPredicate> predicates = new CopyOnWriteArrayList<>();

        private final List<GlobalObservationConvention<?>> conventions = new CopyOnWriteArrayList<>();

        private final List<ObservationFilter> filters = new CopyOnWriteArrayList<>();

        // -- Registration --

        public ObservationConfig observationHandler(ObservationHandler<?> handler) {
            handlers.add(handler);
            return this;
        }

        public ObservationConfig observationPredicate(ObservationPredicate predicate) {
            predicates.add(predicate);
            return this;
        }

        public ObservationConfig observationConvention(GlobalObservationConvention<?> convention) {
            conventions.add(convention);
            return this;
        }

        public ObservationConfig observationFilter(ObservationFilter filter) {
            filters.add(filter);
            return this;
        }

        // -- Query --

        public List<ObservationHandler<?>> getObservationHandlers() {
            return Collections.unmodifiableList(handlers);
        }

        public List<ObservationFilter> getObservationFilters() {
            return Collections.unmodifiableList(filters);
        }

        /**
         * Returns the first matching global convention for the given context,
         * or falls back to the default convention.
         */
        @SuppressWarnings("unchecked")
        public <T extends Observation.Context> ObservationConvention<T> getObservationConvention(
                T context, ObservationConvention<T> defaultConvention) {
            for (GlobalObservationConvention<?> convention : conventions) {
                if (convention.supportsContext(context)) {
                    return (ObservationConvention<T>) convention;
                }
            }
            return defaultConvention;
        }

        /**
         * Checks all predicates. Returns false if any predicate disables
         * the observation.
         */
        public boolean isObservationEnabled(String name, Observation.Context context) {
            for (ObservationPredicate predicate : predicates) {
                if (!predicate.test(name, context)) {
                    return false;
                }
            }
            return true;
        }

    }

}
```

#### File: `src/main/java/dev/linhvu/micrometer/observation/SimpleObservationRegistry.java` [NEW]

```java
package dev.linhvu.micrometer.observation;

/**
 * Default implementation of {@link ObservationRegistry}.
 * <p>
 * Uses a {@link ThreadLocal} to track the current {@link Observation.Scope}
 * per thread. When an observation opens a scope, it becomes "current" on
 * that thread; closing the scope restores the previous one (linked-list stack).
 * <p>
 * <b>Noop optimization:</b> {@link #isNoop()} returns true if no handlers
 * are registered. This means all {@code Observation.createNotStarted()} calls
 * will return {@link Observation#NOOP} without even creating a context —
 * a key performance optimization for applications that haven't configured
 * observation yet.
 * <p>
 * <b>Simplification:</b> The real Micrometer uses a <b>static</b> ThreadLocal
 * shared across all registry instances (so the NOOP registry can also participate
 * in scope management for context propagation). We use a per-instance ThreadLocal
 * since context propagation is out of scope.
 */
public class SimpleObservationRegistry implements ObservationRegistry {

    private final ThreadLocal<Observation.Scope> localObservationScope = new ThreadLocal<>();

    private final ObservationConfig config = new ObservationConfig();

    @Override
    public Observation getCurrentObservation() {
        Observation.Scope scope = localObservationScope.get();
        if (scope != null) {
            return scope.getCurrentObservation();
        }
        return null;
    }

    @Override
    public Observation.Scope getCurrentObservationScope() {
        return localObservationScope.get();
    }

    @Override
    public void setCurrentObservationScope(Observation.Scope scope) {
        localObservationScope.set(scope);
    }

    @Override
    public ObservationConfig observationConfig() {
        return config;
    }

    /**
     * Returns true if no handlers are registered — a registry with no handlers
     * is effectively a noop (no one is listening to lifecycle events).
     */
    @Override
    public boolean isNoop() {
        return config.getObservationHandlers().isEmpty();
    }

}
```

#### File: `src/main/java/dev/linhvu/micrometer/observation/Observation.java` [NEW]

```java
package dev.linhvu.micrometer.observation;

import java.util.HashMap;
import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;
import java.util.function.Supplier;

/**
 * A higher-level instrumentation API that unifies metrics, tracing, and logging
 * under a single observation lifecycle.
 * <p>
 * <b>Lifecycle:</b>
 * <pre>
 * Observation obs = Observation.createNotStarted("my.operation", registry);
 * obs.lowCardinalityKeyValue("method", "GET");
 * obs.start();
 * try (Observation.Scope scope = obs.openScope()) {
 *     // ... do work ...
 * } catch (Exception e) {
 *     obs.error(e);
 *     throw e;
 * } finally {
 *     obs.stop();
 * }
 * </pre>
 * <p>
 * Or use the convenience method:
 * <pre>
 * Observation.createNotStarted("my.operation", registry)
 *     .observe(() -> doWork());
 * </pre>
 * <p>
 * Key values are split into {@link #lowCardinalityKeyValue low cardinality}
 * (become metric tags) and {@link #highCardinalityKeyValue high cardinality}
 * (become trace attributes only).
 * <p>
 * <b>Simplification:</b> The real Micrometer's {@code Observation} extends
 * {@code ObservationView} (read-only interface) and supports checked exceptions,
 * context propagation (scope reset/makeCurrent), and convention-aware factory
 * methods. We keep the core lifecycle and key-value API.
 */
public interface Observation {

    /**
     * A no-op observation. Returned when the registry is null, noop, or when
     * predicates disable the observation. All methods are safe to call but do nothing.
     */
    Observation NOOP = new Observation() {
        @Override
        public Observation start() {
            return this;
        }

        @Override
        public Observation lowCardinalityKeyValue(KeyValue keyValue) {
            return this;
        }

        @Override
        public Observation highCardinalityKeyValue(KeyValue keyValue) {
            return this;
        }

        @Override
        public Observation lowCardinalityKeyValues(KeyValues keyValues) {
            return this;
        }

        @Override
        public Observation highCardinalityKeyValues(KeyValues keyValues) {
            return this;
        }

        @Override
        public Observation contextualName(String contextualName) {
            return this;
        }

        @Override
        public Observation parentObservation(Observation parentObservation) {
            return this;
        }

        @Override
        public Observation error(Throwable error) {
            return this;
        }

        @Override
        public Observation event(Event event) {
            return this;
        }

        @Override
        public Scope openScope() {
            return Scope.NOOP;
        }

        @Override
        public void stop() {
        }

        @Override
        public Context getContext() {
            return new Context();
        }

        @Override
        public boolean isNoop() {
            return true;
        }

        @Override
        public String toString() {
            return "Observation.NOOP";
        }
    };

    // -----------------------------------------------------------------------
    // Factory methods
    // -----------------------------------------------------------------------

    /**
     * Creates an observation that has not yet been started.
     * <p>
     * If the registry is null or noop, returns {@link #NOOP} immediately
     * (avoids context creation — a key performance optimization).
     *
     * @param name     The observation name (e.g., "http.server.requests").
     * @param registry The observation registry.
     * @return A new observation, or {@link #NOOP} if the registry is inactive.
     */
    static Observation createNotStarted(String name, ObservationRegistry registry) {
        return createNotStarted(name, Context::new, registry);
    }

    /**
     * Creates an observation with a custom context.
     *
     * @param name            The observation name.
     * @param contextSupplier Supplier for the context (called only if observation is active).
     * @param registry        The observation registry.
     * @return A new observation, or {@link #NOOP}.
     */
    static <T extends Context> Observation createNotStarted(
            String name, Supplier<T> contextSupplier, ObservationRegistry registry) {
        if (registry == null || registry.isNoop()) {
            return NOOP;
        }
        T context = contextSupplier.get();
        context.setParentFromCurrentObservation(registry);
        if (!registry.observationConfig().isObservationEnabled(name, context)) {
            return NOOP;
        }
        return new SimpleObservation(name, registry, context);
    }

    /**
     * Creates an observation using the convention resolution chain.
     * <p>
     * Resolution order:
     * <ol>
     *   <li>{@code customConvention} if non-null</li>
     *   <li>Matching {@link GlobalObservationConvention} from the registry config</li>
     *   <li>{@code defaultConvention}</li>
     * </ol>
     *
     * @param customConvention  User-provided override (nullable).
     * @param defaultConvention Library-provided default.
     * @param contextSupplier   Supplier for the context.
     * @param registry          The observation registry.
     * @return A new observation, or {@link #NOOP}.
     */
    @SuppressWarnings("unchecked")
    static <T extends Context> Observation createNotStarted(
            ObservationConvention<T> customConvention,
            ObservationConvention<T> defaultConvention,
            Supplier<T> contextSupplier,
            ObservationRegistry registry) {
        if (registry == null || registry.isNoop()) {
            return NOOP;
        }
        T context = contextSupplier.get();
        context.setParentFromCurrentObservation(registry);

        // 3-tier convention resolution
        ObservationConvention<T> resolved;
        if (customConvention != null) {
            resolved = customConvention;
        } else {
            ObservationConvention<T> global = (ObservationConvention<T>)
                    registry.observationConfig().getObservationConvention(context, defaultConvention);
            resolved = global;
        }

        String name = resolved.getName();
        if (name == null || name.isBlank()) {
            name = defaultConvention.getName();
        }
        if (name == null || name.isBlank()) {
            name = "observation";
        }

        if (!registry.observationConfig().isObservationEnabled(name, context)) {
            return NOOP;
        }
        return new SimpleObservation(resolved, registry, context);
    }

    /**
     * Creates and immediately starts an observation.
     */
    static Observation start(String name, ObservationRegistry registry) {
        return createNotStarted(name, registry).start();
    }

    /**
     * Creates and immediately starts an observation with a custom context.
     */
    static <T extends Context> Observation start(
            String name, Supplier<T> contextSupplier, ObservationRegistry registry) {
        return createNotStarted(name, contextSupplier, registry).start();
    }

    // -----------------------------------------------------------------------
    // Lifecycle methods
    // -----------------------------------------------------------------------

    /**
     * Starts the observation. Notifies all matching handlers via {@code onStart()}.
     * Must be called before {@link #openScope()} or {@link #stop()}.
     *
     * @return This observation for chaining.
     */
    Observation start();

    /**
     * Opens a scope for this observation. The observation becomes "current"
     * in the thread-local context, enabling automatic parent-child wiring.
     * <p>
     * Must be used with try-with-resources:
     * <pre>
     * try (Scope scope = observation.openScope()) {
     *     // observation is "in scope" here
     * }
     * </pre>
     *
     * @return A scope that must be closed.
     */
    Scope openScope();

    /**
     * Stops the observation. Applies filters, then notifies handlers in
     * reverse order via {@code onStop()}.
     */
    void stop();

    /**
     * Records an error on this observation.
     *
     * @param error The error that occurred.
     * @return This observation for chaining.
     */
    Observation error(Throwable error);

    /**
     * Signals an arbitrary event during the observation.
     *
     * @param event The event to signal.
     * @return This observation for chaining.
     */
    Observation event(Event event);

    // -----------------------------------------------------------------------
    // Key value methods (fluent API)
    // -----------------------------------------------------------------------

    Observation lowCardinalityKeyValue(KeyValue keyValue);

    default Observation lowCardinalityKeyValue(String key, String value) {
        return lowCardinalityKeyValue(KeyValue.of(key, value));
    }

    Observation highCardinalityKeyValue(KeyValue keyValue);

    default Observation highCardinalityKeyValue(String key, String value) {
        return highCardinalityKeyValue(KeyValue.of(key, value));
    }

    Observation lowCardinalityKeyValues(KeyValues keyValues);

    Observation highCardinalityKeyValues(KeyValues keyValues);

    /**
     * Sets a contextual name for this observation (e.g., derived from an HTTP path).
     */
    Observation contextualName(String contextualName);

    /**
     * Manually sets the parent observation. Alternative to scope-based auto-parenting.
     */
    Observation parentObservation(Observation parentObservation);

    // -----------------------------------------------------------------------
    // Query methods
    // -----------------------------------------------------------------------

    /**
     * Returns the context associated with this observation.
     */
    Context getContext();

    /**
     * Whether this observation is a noop (does nothing).
     */
    default boolean isNoop() {
        return this == NOOP;
    }

    // -----------------------------------------------------------------------
    // Convenience methods
    // -----------------------------------------------------------------------

    /**
     * Runs the full observation lifecycle around the given runnable:
     * start → openScope → run → stop (with error handling).
     */
    default void observe(Runnable runnable) {
        start();
        try (Scope scope = openScope()) {
            runnable.run();
        } catch (Throwable error) {
            error(error);
            throw error;
        } finally {
            stop();
        }
    }

    /**
     * Runs the full observation lifecycle around the given supplier:
     * start → openScope → get → stop (with error handling).
     *
     * @return The supplier's result.
     */
    default <T> T observe(Supplier<T> supplier) {
        start();
        try (Scope scope = openScope()) {
            return supplier.get();
        } catch (Throwable error) {
            error(error);
            throw error;
        } finally {
            stop();
        }
    }

    // -----------------------------------------------------------------------
    // Inner types
    // -----------------------------------------------------------------------

    /**
     * Represents the observation being "in scope" — the observation is the
     * current one on this thread.
     * <p>
     * Implements {@link AutoCloseable} for try-with-resources. When closed,
     * restores the previous observation scope (forming a linked-list stack).
     */
    interface Scope extends AutoCloseable {

        Scope NOOP = new Scope() {
            @Override
            public Observation getCurrentObservation() {
                return Observation.NOOP;
            }

            @Override
            public void close() {
            }

            @Override
            public boolean isNoop() {
                return true;
            }
        };

        /**
         * Returns the observation that opened this scope.
         */
        Observation getCurrentObservation();

        /**
         * Closes this scope and restores the previous observation scope.
         * Notifies handlers via {@code onScopeClosed()}.
         */
        @Override
        void close();

        /**
         * Whether this scope is a noop.
         */
        default boolean isNoop() {
            return this == NOOP;
        }

    }

    /**
     * An arbitrary event that can be signaled during an observation.
     * <p>
     * Events are used for things like "message received", "connection established",
     * etc. — discrete occurrences within the observation's lifetime.
     */
    interface Event {

        static Event of(String name) {
            return of(name, name);
        }

        static Event of(String name, String contextualName) {
            return new SimpleEvent(name, contextualName);
        }

        /**
         * The event name (used as a technical identifier).
         */
        String getName();

        /**
         * A human-readable contextual name for this event.
         */
        String getContextualName();

    }

    /**
     * Mutable data holder that {@link ObservationHandler} implementations read and write.
     * <p>
     * Each observation has exactly one context. Handlers use it to:
     * <ul>
     *   <li>Read key values and metadata to create metrics/spans</li>
     *   <li>Store their own data (e.g., a Timer.Sample, a Span) via the generic
     *       {@link #put}/{@link #get} map</li>
     * </ul>
     * <p>
     * Key values are stored in maps keyed by key name — adding a key value with
     * the same key replaces the previous one (last-writer-wins deduplication).
     */
    class Context {

        private String name;

        private String contextualName;

        private Throwable error;

        private Observation parentObservation;

        private final Map<String, KeyValue> lowCardinalityKeyValues = new ConcurrentHashMap<>();

        private final Map<String, KeyValue> highCardinalityKeyValues = new ConcurrentHashMap<>();

        private final Map<Object, Object> map = new HashMap<>();

        // -- Name --

        public String getName() {
            return name;
        }

        public Context setName(String name) {
            this.name = name;
            return this;
        }

        public String getContextualName() {
            return contextualName;
        }

        public Context setContextualName(String contextualName) {
            this.contextualName = contextualName;
            return this;
        }

        /**
         * Returns the contextual name if set, otherwise the regular name.
         */
        public String getNameForDisplay() {
            return (contextualName != null && !contextualName.isBlank())
                    ? contextualName : name;
        }

        // -- Error --

        public Throwable getError() {
            return error;
        }

        public Context setError(Throwable error) {
            this.error = error;
            return this;
        }

        // -- Parent --

        public Observation getParentObservation() {
            return parentObservation;
        }

        public Context setParentObservation(Observation parentObservation) {
            this.parentObservation = parentObservation;
            return this;
        }

        /**
         * Auto-wires the parent from the registry's current observation.
         * Only sets the parent if none is already set.
         */
        void setParentFromCurrentObservation(ObservationRegistry registry) {
            if (this.parentObservation == null) {
                Observation current = registry.getCurrentObservation();
                if (current != null) {
                    this.parentObservation = current;
                }
            }
        }

        // -- Key values --

        public Context addLowCardinalityKeyValue(KeyValue keyValue) {
            lowCardinalityKeyValues.put(keyValue.getKey(), keyValue);
            return this;
        }

        public Context addHighCardinalityKeyValue(KeyValue keyValue) {
            highCardinalityKeyValues.put(keyValue.getKey(), keyValue);
            return this;
        }

        public Context addLowCardinalityKeyValues(KeyValues keyValues) {
            for (KeyValue kv : keyValues) {
                lowCardinalityKeyValues.put(kv.getKey(), kv);
            }
            return this;
        }

        public Context addHighCardinalityKeyValues(KeyValues keyValues) {
            for (KeyValue kv : keyValues) {
                highCardinalityKeyValues.put(kv.getKey(), kv);
            }
            return this;
        }

        /**
         * Returns sorted low-cardinality key values.
         */
        public KeyValues getLowCardinalityKeyValues() {
            return KeyValues.of(lowCardinalityKeyValues.values());
        }

        /**
         * Returns sorted high-cardinality key values.
         */
        public KeyValues getHighCardinalityKeyValues() {
            return KeyValues.of(highCardinalityKeyValues.values());
        }

        /**
         * Returns all key values (both low and high cardinality), sorted.
         */
        public KeyValues getAllKeyValues() {
            return getLowCardinalityKeyValues().and(getHighCardinalityKeyValues());
        }

        // -- Generic map for handler data --

        /**
         * Stores arbitrary data (e.g., a Timer.Sample, a Span object)
         * for use by handlers.
         */
        public <V> Context put(Object key, V value) {
            map.put(key, value);
            return this;
        }

        /**
         * Retrieves data stored by a handler.
         */
        @SuppressWarnings("unchecked")
        public <V> V get(Object key) {
            return (V) map.get(key);
        }

        /**
         * Retrieves data or throws if missing.
         */
        @SuppressWarnings("unchecked")
        public <V> V getRequired(Object key) {
            V value = (V) map.get(key);
            if (value == null) {
                throw new IllegalStateException("Context does not contain key: " + key);
            }
            return value;
        }

        /**
         * Retrieves data, computing and storing it if absent.
         */
        @SuppressWarnings("unchecked")
        public <V> V computeIfAbsent(Object key, java.util.function.Function<Object, V> function) {
            return (V) map.computeIfAbsent(key, function);
        }

        @Override
        public String toString() {
            return "Context{name=" + name + ", contextualName=" + contextualName
                    + ", error=" + error + ", lowCardinalityKeyValues="
                    + getLowCardinalityKeyValues() + ", highCardinalityKeyValues="
                    + getHighCardinalityKeyValues() + "}";
        }

    }

    // -----------------------------------------------------------------------
    // Simple event implementation
    // -----------------------------------------------------------------------

    /**
     * Default implementation of {@link Event}.
     */
    record SimpleEvent(String name, String contextualName) implements Event {

        @Override
        public String getName() {
            return name;
        }

        @Override
        public String getContextualName() {
            return contextualName;
        }

        @Override
        public String toString() {
            return "Event{" + name + "}";
        }

    }

}
```

#### File: `src/main/java/dev/linhvu/micrometer/observation/SimpleObservation.java` [NEW]

```java
package dev.linhvu.micrometer.observation;

import java.util.ArrayDeque;
import java.util.Collection;
import java.util.Deque;
import java.util.Iterator;

/**
 * The default (and only non-noop) implementation of {@link Observation}.
 * <p>
 * Orchestrates the full observation lifecycle:
 * <ul>
 *   <li>{@code start()} — applies convention key values, notifies handlers forward</li>
 *   <li>{@code openScope()} — creates a {@link SimpleScope}, notifies handlers forward</li>
 *   <li>{@code stop()} — applies convention key values again, runs filters, notifies
 *       handlers in <b>reverse</b> order (symmetric cleanup)</li>
 *   <li>{@code error()} / {@code event()} — notifies handlers immediately</li>
 * </ul>
 * <p>
 * <b>Handler filtering:</b> At construction time, only handlers whose
 * {@code supportsContext()} returns true are kept. This avoids repeated
 * checks during lifecycle events.
 * <p>
 * <b>Reverse notification on stop/close:</b> If handlers 1, 2, 3 were notified
 * on start/open, they get stopped/closed in order 3, 2, 1. This ensures symmetric
 * cleanup (last resource opened is first to close).
 * <p>
 * <b>Simplification:</b> The real implementation tracks the last scope per thread
 * in a ConcurrentHashMap for multi-thread scope handling. We simplify to a single-
 * thread model where each thread manages its own scope stack via the registry's
 * ThreadLocal.
 */
public class SimpleObservation implements Observation {

    private final ObservationRegistry registry;

    private final Context context;

    private final ObservationConvention<Context> convention;

    private final Deque<ObservationHandler<Context>> handlers;

    private final Collection<ObservationFilter> filters;

    /**
     * Creates an observation with a name (no convention).
     */
    @SuppressWarnings("unchecked")
    SimpleObservation(String name, ObservationRegistry registry, Context context) {
        this.registry = registry;
        this.context = context;
        this.context.setName(name);
        this.convention = findConventionFromConfig(context, registry);
        this.handlers = getHandlersFromConfig(context, registry);
        this.filters = registry.observationConfig().getObservationFilters();
    }

    /**
     * Creates an observation with a resolved convention.
     */
    @SuppressWarnings("unchecked")
    SimpleObservation(ObservationConvention<? extends Context> convention,
            ObservationRegistry registry, Context context) {
        this.registry = registry;
        this.context = context;
        this.convention = (ObservationConvention<Context>) convention;

        // Set name from convention
        String conventionName = convention.getName();
        if (conventionName != null && !conventionName.isBlank()) {
            this.context.setName(conventionName);
        }

        this.handlers = getHandlersFromConfig(context, registry);
        this.filters = registry.observationConfig().getObservationFilters();
    }

    // -----------------------------------------------------------------------
    // Lifecycle
    // -----------------------------------------------------------------------

    @Override
    public Observation start() {
        if (convention != null) {
            context.addLowCardinalityKeyValues(convention.getLowCardinalityKeyValues(context));
            context.addHighCardinalityKeyValues(convention.getHighCardinalityKeyValues(context));
            String name = convention.getName();
            if (name != null && !name.isBlank()) {
                context.setName(name);
            }
        }
        notifyOnObservationStarted();
        return this;
    }

    @Override
    public Scope openScope() {
        SimpleScope scope = new SimpleScope(registry, this);
        notifyOnScopeOpened();
        return scope;
    }

    @Override
    public void stop() {
        if (convention != null) {
            // Convention may produce different key values at stop time
            context.addLowCardinalityKeyValues(convention.getLowCardinalityKeyValues(context));
            context.addHighCardinalityKeyValues(convention.getHighCardinalityKeyValues(context));
            String contextualName = convention.getContextualName(context);
            if (contextualName != null && !contextualName.isBlank()) {
                context.setContextualName(contextualName);
            }
        }
        // Run filters before notifying handlers
        Context ctx = context;
        for (ObservationFilter filter : filters) {
            ctx = filter.map(ctx);
        }
        notifyOnObservationStopped(ctx);
    }

    @Override
    public Observation error(Throwable error) {
        context.setError(error);
        notifyOnError();
        return this;
    }

    @Override
    public Observation event(Event event) {
        notifyOnEvent(event);
        return this;
    }

    // -----------------------------------------------------------------------
    // Key values
    // -----------------------------------------------------------------------

    @Override
    public Observation lowCardinalityKeyValue(KeyValue keyValue) {
        context.addLowCardinalityKeyValue(keyValue);
        return this;
    }

    @Override
    public Observation highCardinalityKeyValue(KeyValue keyValue) {
        context.addHighCardinalityKeyValue(keyValue);
        return this;
    }

    @Override
    public Observation lowCardinalityKeyValues(KeyValues keyValues) {
        context.addLowCardinalityKeyValues(keyValues);
        return this;
    }

    @Override
    public Observation highCardinalityKeyValues(KeyValues keyValues) {
        context.addHighCardinalityKeyValues(keyValues);
        return this;
    }

    @Override
    public Observation contextualName(String contextualName) {
        context.setContextualName(contextualName);
        return this;
    }

    @Override
    public Observation parentObservation(Observation parentObservation) {
        context.setParentObservation(parentObservation);
        return this;
    }

    // -----------------------------------------------------------------------
    // Query
    // -----------------------------------------------------------------------

    @Override
    public Context getContext() {
        return context;
    }

    // -----------------------------------------------------------------------
    // Handler notifications
    // -----------------------------------------------------------------------

    private void notifyOnObservationStarted() {
        for (ObservationHandler<Context> handler : handlers) {
            handler.onStart(context);
        }
    }

    private void notifyOnObservationStopped(Context ctx) {
        // Reverse order — symmetric cleanup
        Iterator<ObservationHandler<Context>> it = handlers.descendingIterator();
        while (it.hasNext()) {
            it.next().onStop(ctx);
        }
    }

    void notifyOnScopeOpened() {
        for (ObservationHandler<Context> handler : handlers) {
            handler.onScopeOpened(context);
        }
    }

    void notifyOnScopeClosed() {
        // Reverse order
        Iterator<ObservationHandler<Context>> it = handlers.descendingIterator();
        while (it.hasNext()) {
            it.next().onScopeClosed(context);
        }
    }

    private void notifyOnError() {
        for (ObservationHandler<Context> handler : handlers) {
            handler.onError(context);
        }
    }

    private void notifyOnEvent(Event event) {
        for (ObservationHandler<Context> handler : handlers) {
            handler.onEvent(event, context);
        }
    }

    // -----------------------------------------------------------------------
    // Internals
    // -----------------------------------------------------------------------

    /**
     * Filters the registry's handlers to only those that support this context.
     */
    @SuppressWarnings("unchecked")
    private static Deque<ObservationHandler<Observation.Context>> getHandlersFromConfig(
            Observation.Context context, ObservationRegistry registry) {
        Deque<ObservationHandler<Observation.Context>> matching = new ArrayDeque<>();
        for (ObservationHandler<?> handler : registry.observationConfig().getObservationHandlers()) {
            if (handler.supportsContext(context)) {
                matching.add((ObservationHandler<Observation.Context>) handler);
            }
        }
        return matching;
    }

    /**
     * Looks for a matching convention in the registry's global config.
     */
    @SuppressWarnings("unchecked")
    private static ObservationConvention<Observation.Context> findConventionFromConfig(
            Observation.Context context, ObservationRegistry registry) {
        // Check if any global convention supports this context
        ObservationConvention<Observation.Context> found =
                registry.observationConfig().getObservationConvention(context, null);
        return found;
    }

    // -----------------------------------------------------------------------
    // SimpleScope
    // -----------------------------------------------------------------------

    /**
     * Default scope implementation. Creates a linked-list stack of scopes
     * via the registry's ThreadLocal.
     * <p>
     * On creation: saves the previous scope, sets itself as current.
     * On close: restores the previous scope, notifies handlers.
     */
    static class SimpleScope implements Scope {

        private final ObservationRegistry registry;

        private final SimpleObservation observation;

        private final Scope previousObservationScope;

        SimpleScope(ObservationRegistry registry, SimpleObservation observation) {
            this.registry = registry;
            this.observation = observation;
            this.previousObservationScope = registry.getCurrentObservationScope();
            registry.setCurrentObservationScope(this);
        }

        @Override
        public Observation getCurrentObservation() {
            return observation;
        }

        @Override
        public void close() {
            observation.notifyOnScopeClosed();
            registry.setCurrentObservationScope(previousObservationScope);
        }

    }

}
```

### Test Code

#### File: `src/test/java/dev/linhvu/micrometer/observation/KeyValueTest.java` [NEW]

```java
package dev.linhvu.micrometer.observation;

import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

class KeyValueTest {

    @Test
    void shouldCreateKeyValue_WithKeyAndValue() {
        KeyValue kv = KeyValue.of("method", "GET");
        assertThat(kv.getKey()).isEqualTo("method");
        assertThat(kv.getValue()).isEqualTo("GET");
    }

    @Test
    void shouldCompareByKey_IgnoringValue() {
        KeyValue a = KeyValue.of("a", "z");
        KeyValue b = KeyValue.of("b", "a");
        assertThat(a.compareTo(b)).isLessThan(0);
    }

    @Test
    void shouldRejectNullKey() {
        assertThatThrownBy(() -> KeyValue.of(null, "value"))
                .isInstanceOf(IllegalArgumentException.class);
    }

    @Test
    void shouldRejectBlankKey() {
        assertThatThrownBy(() -> KeyValue.of("  ", "value"))
                .isInstanceOf(IllegalArgumentException.class);
    }

    @Test
    void shouldRejectNullValue() {
        assertThatThrownBy(() -> KeyValue.of("key", null))
                .isInstanceOf(IllegalArgumentException.class);
    }

    @Test
    void shouldHaveNoneSentinelValue() {
        assertThat(KeyValue.NONE_VALUE).isEqualTo("none");
    }

}
```

#### File: `src/test/java/dev/linhvu/micrometer/observation/KeyValuesTest.java` [NEW]

```java
package dev.linhvu.micrometer.observation;

import org.junit.jupiter.api.Test;

import java.util.List;
import java.util.stream.StreamSupport;

import static org.assertj.core.api.Assertions.assertThat;

class KeyValuesTest {

    @Test
    void shouldCreateEmpty() {
        KeyValues kvs = KeyValues.empty();
        assertThat(kvs.isEmpty()).isTrue();
    }

    @Test
    void shouldCreateFromKeyAndValue() {
        KeyValues kvs = KeyValues.of("method", "GET");
        assertThat(asList(kvs)).hasSize(1);
        assertThat(asList(kvs).get(0).getKey()).isEqualTo("method");
        assertThat(asList(kvs).get(0).getValue()).isEqualTo("GET");
    }

    @Test
    void shouldSortByKey() {
        KeyValues kvs = KeyValues.of(
                KeyValue.of("z", "1"),
                KeyValue.of("a", "2"),
                KeyValue.of("m", "3")
        );
        List<String> keys = asList(kvs).stream().map(KeyValue::getKey).toList();
        assertThat(keys).containsExactly("a", "m", "z");
    }

    @Test
    void shouldDeduplicateByKey_LastWins() {
        KeyValues kvs = KeyValues.of(
                KeyValue.of("method", "GET"),
                KeyValue.of("method", "POST")
        );
        assertThat(asList(kvs)).hasSize(1);
        assertThat(asList(kvs).get(0).getValue()).isEqualTo("POST");
    }

    @Test
    void shouldMergeWithAnd() {
        KeyValues a = KeyValues.of("a", "1");
        KeyValues b = KeyValues.of("b", "2");
        KeyValues merged = a.and(b);
        List<String> keys = asList(merged).stream().map(KeyValue::getKey).toList();
        assertThat(keys).containsExactly("a", "b");
    }

    @Test
    void shouldOverrideOnKeyConflict_WhenMerging() {
        KeyValues a = KeyValues.of("key", "old");
        KeyValues b = KeyValues.of("key", "new");
        KeyValues merged = a.and(b);
        assertThat(asList(merged)).hasSize(1);
        assertThat(asList(merged).get(0).getValue()).isEqualTo("new");
    }

    @Test
    void shouldBeImmutable_AndReturnsNewInstance() {
        KeyValues original = KeyValues.of("a", "1");
        KeyValues merged = original.and("b", "2");
        assertThat(asList(original)).hasSize(1);
        assertThat(asList(merged)).hasSize(2);
    }

    @Test
    void shouldSupportStreamOperations() {
        KeyValues kvs = KeyValues.of(
                KeyValue.of("a", "1"),
                KeyValue.of("b", "2")
        );
        List<String> values = kvs.stream().map(KeyValue::getValue).toList();
        assertThat(values).containsExactly("1", "2");
    }

    private List<KeyValue> asList(KeyValues kvs) {
        return StreamSupport.stream(kvs.spliterator(), false).toList();
    }

}
```

#### File: `src/test/java/dev/linhvu/micrometer/observation/SimpleObservationTest.java` [NEW]

```java
package dev.linhvu.micrometer.observation;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.util.ArrayList;
import java.util.List;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

/**
 * Tests for the core observation lifecycle: create → start → scope → stop,
 * key values, error handling, and event signaling.
 */
class SimpleObservationTest {

    private ObservationRegistry registry;
    private TestHandler handler;

    @BeforeEach
    void setUp() {
        registry = ObservationRegistry.create();
        handler = new TestHandler();
        registry.observationConfig().observationHandler(handler);
    }

    @Test
    void shouldNotifyHandler_WhenObservationStarted() {
        Observation observation = Observation.createNotStarted("test.op", registry);
        observation.start();
        assertThat(handler.started).isTrue();
        assertThat(handler.lastContext.getName()).isEqualTo("test.op");
    }

    @Test
    void shouldNotifyHandler_WhenObservationStopped() {
        Observation observation = Observation.createNotStarted("test.op", registry);
        observation.start();
        observation.stop();
        assertThat(handler.stopped).isTrue();
    }

    @Test
    void shouldNotifyInReverseOrder_WhenStopped() {
        List<String> order = new ArrayList<>();

        ObservationRegistry orderedRegistry = ObservationRegistry.create();
        orderedRegistry.observationConfig().observationHandler(new TestHandler() {
            @Override public void onStart(Observation.Context context) { order.add("A-start"); }
            @Override public void onStop(Observation.Context context) { order.add("A-stop"); }
        });
        orderedRegistry.observationConfig().observationHandler(new TestHandler() {
            @Override public void onStart(Observation.Context context) { order.add("B-start"); }
            @Override public void onStop(Observation.Context context) { order.add("B-stop"); }
        });

        Observation obs = Observation.createNotStarted("test.op", orderedRegistry);
        obs.start();
        obs.stop();
        assertThat(order).containsExactly("A-start", "B-start", "B-stop", "A-stop");
    }

    @Test
    void shouldOpenAndCloseScope() {
        Observation observation = Observation.createNotStarted("test.op", registry);
        observation.start();
        try (Observation.Scope scope = observation.openScope()) {
            assertThat(handler.scopeOpened).isTrue();
            assertThat(registry.getCurrentObservation()).isSameAs(observation);
        }
        assertThat(handler.scopeClosed).isTrue();
        assertThat(registry.getCurrentObservation()).isNull();
    }

    @Test
    void shouldRestorePreviousScope_WhenClosingNestedScope() {
        Observation outer = Observation.createNotStarted("outer", registry);
        Observation inner = Observation.createNotStarted("inner", registry);

        outer.start();
        try (Observation.Scope outerScope = outer.openScope()) {
            assertThat(registry.getCurrentObservation()).isSameAs(outer);

            inner.start();
            try (Observation.Scope innerScope = inner.openScope()) {
                assertThat(registry.getCurrentObservation()).isSameAs(inner);
            }
            assertThat(registry.getCurrentObservation()).isSameAs(outer);
        }
        assertThat(registry.getCurrentObservation()).isNull();
        outer.stop();
        inner.stop();
    }

    @Test
    void shouldRecordError() {
        Observation observation = Observation.createNotStarted("test.op", registry);
        observation.start();
        RuntimeException error = new RuntimeException("boom");
        observation.error(error);
        assertThat(handler.errorRecorded).isTrue();
        assertThat(observation.getContext().getError()).isSameAs(error);
    }

    @Test
    void shouldSignalEvent() {
        Observation observation = Observation.createNotStarted("test.op", registry);
        observation.start();
        Observation.Event event = Observation.Event.of("message.received");
        observation.event(event);
        assertThat(handler.lastEvent).isNotNull();
        assertThat(handler.lastEvent.getName()).isEqualTo("message.received");
    }

    @Test
    void shouldAddLowCardinalityKeyValues() {
        Observation observation = Observation.createNotStarted("test.op", registry);
        observation.lowCardinalityKeyValue("method", "GET");
        observation.lowCardinalityKeyValue("status", "200");
        observation.start();

        KeyValues kvs = observation.getContext().getLowCardinalityKeyValues();
        assertThat(kvs.stream().map(KeyValue::getKey).toList())
                .containsExactly("method", "status");
    }

    @Test
    void shouldAddHighCardinalityKeyValues() {
        Observation observation = Observation.createNotStarted("test.op", registry);
        observation.highCardinalityKeyValue("user.id", "12345");
        observation.start();

        KeyValues kvs = observation.getContext().getHighCardinalityKeyValues();
        assertThat(kvs.stream().map(KeyValue::getKey).toList())
                .containsExactly("user.id");
    }

    @Test
    void shouldSetContextualName() {
        Observation observation = Observation.createNotStarted("http.request", registry);
        observation.contextualName("GET /api/users");
        observation.start();

        assertThat(observation.getContext().getContextualName()).isEqualTo("GET /api/users");
        assertThat(observation.getContext().getNameForDisplay()).isEqualTo("GET /api/users");
    }

    @Test
    void shouldReturnName_WhenNoContextualNameSet() {
        Observation observation = Observation.createNotStarted("http.request", registry);
        observation.start();
        assertThat(observation.getContext().getNameForDisplay()).isEqualTo("http.request");
    }

    @Test
    void shouldSetParentObservation() {
        Observation parent = Observation.createNotStarted("parent.op", registry);
        Observation child = Observation.createNotStarted("child.op", registry);
        child.parentObservation(parent);
        child.start();

        assertThat(child.getContext().getParentObservation()).isSameAs(parent);
    }

    @Test
    void shouldAutoWireParent_FromCurrentScope() {
        Observation parent = Observation.createNotStarted("parent.op", registry);
        parent.start();
        try (Observation.Scope scope = parent.openScope()) {
            Observation child = Observation.createNotStarted("child.op", registry);
            assertThat(child.getContext().getParentObservation()).isSameAs(parent);
        }
        parent.stop();
    }

    @Test
    void shouldRunObserveRunnable() {
        Observation.createNotStarted("test.op", registry)
                .observe(() -> {
                    assertThat(registry.getCurrentObservation()).isNotNull();
                });

        assertThat(handler.started).isTrue();
        assertThat(handler.scopeOpened).isTrue();
        assertThat(handler.scopeClosed).isTrue();
        assertThat(handler.stopped).isTrue();
    }

    @Test
    void shouldRunObserveSupplier() {
        String result = Observation.createNotStarted("test.op", registry)
                .observe(() -> "hello");

        assertThat(result).isEqualTo("hello");
        assertThat(handler.started).isTrue();
        assertThat(handler.stopped).isTrue();
    }

    @Test
    void shouldRecordError_WhenObserveThrows() {
        RuntimeException error = new RuntimeException("boom");
        assertThatThrownBy(() ->
                Observation.createNotStarted("test.op", registry)
                        .observe(() -> { throw error; })
        ).isSameAs(error);

        assertThat(handler.errorRecorded).isTrue();
        assertThat(handler.stopped).isTrue();
    }

    @Test
    void shouldReturnNoop_WhenRegistryIsNull() {
        Observation obs = Observation.createNotStarted("test.op", (ObservationRegistry) null);
        assertThat(obs.isNoop()).isTrue();
    }

    @Test
    void shouldReturnNoop_WhenRegistryIsNoop() {
        Observation obs = Observation.createNotStarted("test.op", ObservationRegistry.NOOP);
        assertThat(obs.isNoop()).isTrue();
    }

    @Test
    void shouldReturnNoop_WhenRegistryHasNoHandlers() {
        ObservationRegistry emptyRegistry = ObservationRegistry.create();
        Observation obs = Observation.createNotStarted("test.op", emptyRegistry);
        assertThat(obs.isNoop()).isTrue();
    }

    @Test
    void shouldCreateEvent_WithName() {
        Observation.Event event = Observation.Event.of("test.event");
        assertThat(event.getName()).isEqualTo("test.event");
        assertThat(event.getContextualName()).isEqualTo("test.event");
    }

    @Test
    void shouldCreateEvent_WithNameAndContextualName() {
        Observation.Event event = Observation.Event.of("test.event", "Test Event Occurred");
        assertThat(event.getName()).isEqualTo("test.event");
        assertThat(event.getContextualName()).isEqualTo("Test Event Occurred");
    }

    // -----------------------------------------------------------------------
    // Test handler
    // -----------------------------------------------------------------------

    static class TestHandler implements ObservationHandler<Observation.Context> {

        boolean started;
        boolean stopped;
        boolean scopeOpened;
        boolean scopeClosed;
        boolean errorRecorded;
        Observation.Context lastContext;
        Observation.Event lastEvent;

        @Override
        public void onStart(Observation.Context context) {
            started = true;
            lastContext = context;
        }

        @Override
        public void onStop(Observation.Context context) {
            stopped = true;
        }

        @Override
        public void onScopeOpened(Observation.Context context) {
            scopeOpened = true;
        }

        @Override
        public void onScopeClosed(Observation.Context context) {
            scopeClosed = true;
        }

        @Override
        public void onError(Observation.Context context) {
            errorRecorded = true;
        }

        @Override
        public void onEvent(Observation.Event event, Observation.Context context) {
            lastEvent = event;
        }

        @Override
        public boolean supportsContext(Observation.Context context) {
            return true;
        }

    }

}
```

#### File: `src/test/java/dev/linhvu/micrometer/observation/ObservationRegistryTest.java` [NEW]

```java
package dev.linhvu.micrometer.observation;

import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;

/**
 * Tests for {@link ObservationRegistry} and {@link ObservationRegistry.ObservationConfig}.
 */
class ObservationRegistryTest {

    @Test
    void shouldCreateRegistryWithCreate() {
        ObservationRegistry registry = ObservationRegistry.create();
        assertThat(registry).isInstanceOf(SimpleObservationRegistry.class);
    }

    @Test
    void shouldBeNoop_WhenNoHandlersRegistered() {
        ObservationRegistry registry = ObservationRegistry.create();
        assertThat(registry.isNoop()).isTrue();
    }

    @Test
    void shouldNotBeNoop_WhenHandlerRegistered() {
        ObservationRegistry registry = ObservationRegistry.create();
        registry.observationConfig().observationHandler(new SimpleObservationTest.TestHandler());
        assertThat(registry.isNoop()).isFalse();
    }

    @Test
    void shouldReturnNullCurrentObservation_WhenNoScopeOpen() {
        ObservationRegistry registry = ObservationRegistry.create();
        assertThat(registry.getCurrentObservation()).isNull();
    }

    @Test
    void shouldTrackCurrentObservation_WhenScopeOpened() {
        ObservationRegistry registry = ObservationRegistry.create();
        registry.observationConfig().observationHandler(new SimpleObservationTest.TestHandler());

        Observation obs = Observation.createNotStarted("test", registry);
        obs.start();
        try (Observation.Scope scope = obs.openScope()) {
            assertThat(registry.getCurrentObservation()).isSameAs(obs);
        }
        assertThat(registry.getCurrentObservation()).isNull();
        obs.stop();
    }

    @Test
    void shouldDisableObservation_WhenPredicateReturnsFalse() {
        ObservationRegistry registry = ObservationRegistry.create();
        registry.observationConfig().observationHandler(new SimpleObservationTest.TestHandler());
        registry.observationConfig().observationPredicate(
                (name, context) -> !name.startsWith("internal."));

        Observation enabled = Observation.createNotStarted("http.request", registry);
        Observation disabled = Observation.createNotStarted("internal.debug", registry);

        assertThat(enabled.isNoop()).isFalse();
        assertThat(disabled.isNoop()).isTrue();
    }

    @Test
    void shouldApplyFilter_BeforeStop() {
        ObservationRegistry registry = ObservationRegistry.create();
        SimpleObservationTest.TestHandler handler = new SimpleObservationTest.TestHandler();
        registry.observationConfig().observationHandler(handler);
        registry.observationConfig().observationFilter(context -> {
            context.addLowCardinalityKeyValue(KeyValue.of("filtered", "true"));
            return context;
        });

        Observation obs = Observation.createNotStarted("test", registry);
        obs.start();
        obs.stop();

        assertThat(obs.getContext().getLowCardinalityKeyValues().stream()
                .anyMatch(kv -> kv.getKey().equals("filtered"))).isTrue();
    }

    @Test
    void shouldResolveGlobalConvention_WhenRegistered() {
        ObservationRegistry registry = ObservationRegistry.create();
        SimpleObservationTest.TestHandler handler = new SimpleObservationTest.TestHandler();
        registry.observationConfig().observationHandler(handler);

        GlobalObservationConvention<Observation.Context> globalConvention =
                new GlobalObservationConvention<>() {
                    @Override
                    public KeyValues getLowCardinalityKeyValues(Observation.Context context) {
                        return KeyValues.of("global.key", "global.value");
                    }

                    @Override
                    public boolean supportsContext(Observation.Context context) {
                        return true;
                    }

                    @Override
                    public String getName() {
                        return "global.name";
                    }
                };
        registry.observationConfig().observationConvention(globalConvention);

        ObservationConvention<Observation.Context> defaultConvention =
                new ObservationConvention<>() {
                    @Override
                    public boolean supportsContext(Observation.Context context) {
                        return true;
                    }

                    @Override
                    public String getName() {
                        return "default.name";
                    }
                };

        Observation obs = Observation.createNotStarted(
                null, defaultConvention, Observation.Context::new, registry);
        obs.start();

        assertThat(obs.getContext().getName()).isEqualTo("global.name");
        assertThat(obs.getContext().getLowCardinalityKeyValues().stream()
                .anyMatch(kv -> kv.getKey().equals("global.key"))).isTrue();
        obs.stop();
    }

    @Test
    void shouldUseCustomConvention_WhenProvidedOverGlobal() {
        ObservationRegistry registry = ObservationRegistry.create();
        SimpleObservationTest.TestHandler handler = new SimpleObservationTest.TestHandler();
        registry.observationConfig().observationHandler(handler);

        GlobalObservationConvention<Observation.Context> globalConvention =
                new GlobalObservationConvention<>() {
                    @Override
                    public boolean supportsContext(Observation.Context context) {
                        return true;
                    }

                    @Override
                    public String getName() {
                        return "global.name";
                    }
                };
        registry.observationConfig().observationConvention(globalConvention);

        ObservationConvention<Observation.Context> customConvention =
                new ObservationConvention<>() {
                    @Override
                    public boolean supportsContext(Observation.Context context) {
                        return true;
                    }

                    @Override
                    public String getName() {
                        return "custom.name";
                    }
                };

        ObservationConvention<Observation.Context> defaultConvention =
                new ObservationConvention<>() {
                    @Override
                    public boolean supportsContext(Observation.Context context) {
                        return true;
                    }

                    @Override
                    public String getName() {
                        return "default.name";
                    }
                };

        Observation obs = Observation.createNotStarted(
                customConvention, defaultConvention, Observation.Context::new, registry);
        obs.start();

        assertThat(obs.getContext().getName()).isEqualTo("custom.name");
        obs.stop();
    }

    @Test
    void shouldNoopRegistryAlwaysReturnNoop() {
        Observation obs = Observation.createNotStarted("test", ObservationRegistry.NOOP);
        assertThat(obs.isNoop()).isTrue();
        assertThat(ObservationRegistry.NOOP.isNoop()).isTrue();
    }

}
```

#### File: `src/test/java/dev/linhvu/micrometer/observation/ObservationHandlerTest.java` [NEW]

```java
package dev.linhvu.micrometer.observation;

import org.junit.jupiter.api.Test;

import java.util.ArrayList;
import java.util.List;

import static org.assertj.core.api.Assertions.assertThat;

/**
 * Tests for {@link ObservationHandler} composite handlers:
 * {@link ObservationHandler.FirstMatchingCompositeObservationHandler} and
 * {@link ObservationHandler.AllMatchingCompositeObservationHandler}.
 */
class ObservationHandlerTest {

    @Test
    void shouldOnlyCallFirstMatching_WhenUsingFirstMatchingComposite() {
        List<String> calls = new ArrayList<>();

        ObservationHandler<Observation.Context> handlerA = new TrackingHandler("A", calls, true);
        ObservationHandler<Observation.Context> handlerB = new TrackingHandler("B", calls, true);

        var composite = new ObservationHandler.FirstMatchingCompositeObservationHandler(
                handlerA, handlerB);

        Observation.Context context = new Observation.Context();
        composite.onStart(context);

        assertThat(calls).containsExactly("A-start");
    }

    @Test
    void shouldCallAllMatching_WhenUsingAllMatchingComposite() {
        List<String> calls = new ArrayList<>();

        ObservationHandler<Observation.Context> handlerA = new TrackingHandler("A", calls, true);
        ObservationHandler<Observation.Context> handlerB = new TrackingHandler("B", calls, true);

        var composite = new ObservationHandler.AllMatchingCompositeObservationHandler(
                handlerA, handlerB);

        Observation.Context context = new Observation.Context();
        composite.onStart(context);

        assertThat(calls).containsExactly("A-start", "B-start");
    }

    @Test
    void shouldSkipNonMatchingHandlers_InComposite() {
        List<String> calls = new ArrayList<>();

        ObservationHandler<Observation.Context> matchingHandler =
                new TrackingHandler("A", calls, true);
        ObservationHandler<Observation.Context> nonMatchingHandler =
                new TrackingHandler("B", calls, false);

        var composite = new ObservationHandler.AllMatchingCompositeObservationHandler(
                matchingHandler, nonMatchingHandler);

        Observation.Context context = new Observation.Context();
        composite.onStart(context);

        assertThat(calls).containsExactly("A-start");
    }

    @Test
    void shouldSkipFirstNonMatching_AndCallFirstMatching() {
        List<String> calls = new ArrayList<>();

        ObservationHandler<Observation.Context> nonMatchingHandler =
                new TrackingHandler("A", calls, false);
        ObservationHandler<Observation.Context> matchingHandler =
                new TrackingHandler("B", calls, true);

        var composite = new ObservationHandler.FirstMatchingCompositeObservationHandler(
                nonMatchingHandler, matchingHandler);

        Observation.Context context = new Observation.Context();
        composite.onStart(context);

        assertThat(calls).containsExactly("B-start");
    }

    @Test
    void shouldSupportContext_WhenAnyChildSupports() {
        var matching = new TrackingHandler("A", new ArrayList<>(), true);
        var nonMatching = new TrackingHandler("B", new ArrayList<>(), false);

        var composite = new ObservationHandler.AllMatchingCompositeObservationHandler(
                nonMatching, matching);

        assertThat(composite.supportsContext(new Observation.Context())).isTrue();
    }

    @Test
    void shouldNotSupportContext_WhenNoChildSupports() {
        var nonMatching1 = new TrackingHandler("A", new ArrayList<>(), false);
        var nonMatching2 = new TrackingHandler("B", new ArrayList<>(), false);

        var composite = new ObservationHandler.AllMatchingCompositeObservationHandler(
                nonMatching1, nonMatching2);

        assertThat(composite.supportsContext(new Observation.Context())).isFalse();
    }

    @Test
    void shouldCallAllLifecycleMethods_OnAllMatchingComposite() {
        List<String> calls = new ArrayList<>();
        ObservationHandler<Observation.Context> handler = new TrackingHandler("A", calls, true);

        var composite = new ObservationHandler.AllMatchingCompositeObservationHandler(handler);

        Observation.Context context = new Observation.Context();
        composite.onStart(context);
        composite.onScopeOpened(context);
        composite.onEvent(Observation.Event.of("test"), context);
        composite.onError(context);
        composite.onScopeClosed(context);
        composite.onStop(context);

        assertThat(calls).containsExactly(
                "A-start", "A-scopeOpened", "A-event", "A-error", "A-scopeClosed", "A-stop");
    }

    @Test
    void shouldFilterByContextType_WhenUsingTypedHandler() {
        ObservationRegistry registry = ObservationRegistry.create();
        List<String> calls = new ArrayList<>();

        ObservationHandler<CustomContext> typedHandler = new ObservationHandler<>() {
            @Override
            public void onStart(CustomContext context) {
                calls.add("typed-start:" + context.customField);
            }

            @Override
            public boolean supportsContext(Observation.Context context) {
                return context instanceof CustomContext;
            }
        };

        ObservationHandler<Observation.Context> genericHandler = new ObservationHandler<>() {
            @Override
            public void onStart(Observation.Context context) {
                calls.add("generic-start");
            }

            @Override
            public boolean supportsContext(Observation.Context context) {
                return true;
            }
        };

        registry.observationConfig().observationHandler(typedHandler);
        registry.observationConfig().observationHandler(genericHandler);

        Observation obs1 = Observation.createNotStarted("test", CustomContext::new, registry);
        obs1.start();
        assertThat(calls).containsExactly("typed-start:custom-data", "generic-start");

        calls.clear();

        Observation obs2 = Observation.createNotStarted("test", Observation.Context::new, registry);
        obs2.start();
        assertThat(calls).containsExactly("generic-start");
    }

    // -----------------------------------------------------------------------
    // Helpers
    // -----------------------------------------------------------------------

    static class CustomContext extends Observation.Context {
        String customField = "custom-data";
    }

    static class TrackingHandler implements ObservationHandler<Observation.Context> {

        private final String name;
        private final List<String> calls;
        private final boolean supports;

        TrackingHandler(String name, List<String> calls, boolean supports) {
            this.name = name;
            this.calls = calls;
            this.supports = supports;
        }

        @Override
        public void onStart(Observation.Context context) {
            calls.add(name + "-start");
        }

        @Override
        public void onStop(Observation.Context context) {
            calls.add(name + "-stop");
        }

        @Override
        public void onScopeOpened(Observation.Context context) {
            calls.add(name + "-scopeOpened");
        }

        @Override
        public void onScopeClosed(Observation.Context context) {
            calls.add(name + "-scopeClosed");
        }

        @Override
        public void onError(Observation.Context context) {
            calls.add(name + "-error");
        }

        @Override
        public void onEvent(Observation.Event event, Observation.Context context) {
            calls.add(name + "-event");
        }

        @Override
        public boolean supportsContext(Observation.Context context) {
            return supports;
        }

    }

}
```

#### File: `src/test/java/dev/linhvu/micrometer/observation/ObservationConventionTest.java` [NEW]

```java
package dev.linhvu.micrometer.observation;

import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;

/**
 * Tests for {@link ObservationConvention} and the 3-tier convention resolution.
 */
class ObservationConventionTest {

    @Test
    void shouldApplyConventionKeyValues_WhenStarted() {
        ObservationRegistry registry = ObservationRegistry.create();
        registry.observationConfig().observationHandler(
                new SimpleObservationTest.TestHandler());

        ObservationConvention<Observation.Context> convention = new ObservationConvention<>() {
            @Override
            public KeyValues getLowCardinalityKeyValues(Observation.Context context) {
                return KeyValues.of("method", "GET");
            }

            @Override
            public KeyValues getHighCardinalityKeyValues(Observation.Context context) {
                return KeyValues.of("request.id", "abc-123");
            }

            @Override
            public boolean supportsContext(Observation.Context context) {
                return true;
            }

            @Override
            public String getName() {
                return "http.request";
            }
        };

        Observation obs = Observation.createNotStarted(convention, convention,
                Observation.Context::new, registry);
        obs.start();

        Observation.Context ctx = obs.getContext();
        assertThat(ctx.getName()).isEqualTo("http.request");
        assertThat(ctx.getLowCardinalityKeyValues().stream()
                .anyMatch(kv -> kv.getKey().equals("method") && kv.getValue().equals("GET")))
                .isTrue();
        assertThat(ctx.getHighCardinalityKeyValues().stream()
                .anyMatch(kv -> kv.getKey().equals("request.id")))
                .isTrue();
        obs.stop();
    }

    @Test
    void shouldApplyContextualName_FromConventionAtStopTime() {
        ObservationRegistry registry = ObservationRegistry.create();
        registry.observationConfig().observationHandler(
                new SimpleObservationTest.TestHandler());

        ObservationConvention<Observation.Context> convention = new ObservationConvention<>() {
            @Override
            public boolean supportsContext(Observation.Context context) {
                return true;
            }

            @Override
            public String getName() {
                return "http.request";
            }

            @Override
            public String getContextualName(Observation.Context context) {
                return "GET /api/users";
            }
        };

        Observation obs = Observation.createNotStarted(convention, convention,
                Observation.Context::new, registry);
        obs.start();
        obs.stop();

        assertThat(obs.getContext().getContextualName()).isEqualTo("GET /api/users");
    }

    @Test
    void shouldDefaultToEmptyKeyValues() {
        ObservationConvention<Observation.Context> convention = new ObservationConvention<>() {
            @Override
            public boolean supportsContext(Observation.Context context) {
                return true;
            }
        };

        Observation.Context context = new Observation.Context();
        assertThat(convention.getLowCardinalityKeyValues(context).isEmpty()).isTrue();
        assertThat(convention.getHighCardinalityKeyValues(context).isEmpty()).isTrue();
        assertThat(convention.getName()).isNull();
        assertThat(convention.getContextualName(context)).isNull();
    }

}
```

---

## Summary

| Concept | What It Means |
|---------|---------------|
| **Observation** | Higher-level instrumentation point with a lifecycle: `createNotStarted()` → `start()` → `openScope()` → `stop()` |
| **Observation.Context** | Mutable data holder that handlers read/write — stores name, error, key values, and arbitrary handler data |
| **Observation.Scope** | AutoCloseable that makes an observation "current" on the thread via ThreadLocal — enables automatic parent-child wiring |
| **Observation.Event** | Arbitrary event signaled during an observation's lifetime (e.g., "message received") |
| **ObservationHandler** | Strategy interface that hooks into lifecycle events — the extension point for metrics, tracing, logging |
| **ObservationRegistry** | Manages handlers, predicates, conventions, filters, and current scope state |
| **ObservationConvention** | Separates "what to name and tag" from "what to do with the observation" — 3-tier resolution (custom > global > default) |
| **ObservationFilter** | Mutates context just before `stop()` — last chance to add computed key values |
| **ObservationPredicate** | Enables/disables observations by name or context — disabled observations return `NOOP` |
| **Low vs. High Cardinality** | Low-cardinality key values become metric tags (bounded), high-cardinality become trace attributes only (unbounded) |
| **FirstMatchingComposite** | Delegates to first handler that supports the context — "pick one" pattern |
| **AllMatchingComposite** | Delegates to all handlers that support the context — "notify all" pattern |
| **Reverse stop notification** | Handlers notified on `stop()` in reverse of `start()` order — symmetric cleanup like try-with-resources |

**Next: Chapter 19 — Observation-Metrics Bridge** — Translates observation lifecycle events into Timer, Counter, and LongTaskTimer recordings, making "instrument once, export anywhere" a reality.
