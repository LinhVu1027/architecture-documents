# Chapter 21: Observation API

> Line references based on commit `2c8a4606c` of the Micrometer repository.

## Build Challenge

| Current State | Limitation | Objective |
|---------------|------------|-----------|
| We have a metrics library (`MeterRegistry`, `Counter`, `Timer`, `Gauge`) that requires manual instrumentation — you must explicitly create and manage each meter | Every place you want metrics requires explicit Timer/Counter code; if you later want tracing or logging from the same point, you need to add separate instrumentation for each concern | Build an `Observation` API that lets you instrument once and get metrics (and potentially traces/logs) automatically via pluggable `ObservationHandler`s |

---

## 21.1 The Integration Point

The Observation API connects two independent subsystems: the **vendor-neutral observation lifecycle** (new) and the **metrics core** (existing from Chapter 20). The bridge is `DefaultMeterObservationHandler` — an `ObservationHandler` that translates observation lifecycle events into `MeterRegistry` operations.

**New file:** `iris-micrometer/src/main/java/com/iris/micrometer/core/observation/DefaultMeterObservationHandler.java`

```java
package com.iris.micrometer.core.observation;

import com.iris.micrometer.core.MeterRegistry;
import com.iris.micrometer.core.Timer;
import com.iris.micrometer.observation.KeyValue;
import com.iris.micrometer.observation.Observation;
import com.iris.micrometer.observation.ObservationHandler;

import java.util.ArrayList;
import java.util.List;

public class DefaultMeterObservationHandler implements ObservationHandler<Observation.Context> {

    private final MeterRegistry meterRegistry;

    public DefaultMeterObservationHandler(MeterRegistry meterRegistry) {
        this.meterRegistry = meterRegistry;
    }

    @Override
    public void onStart(Observation.Context context) {
        Timer.Sample sample = Timer.start();
        context.put(Timer.Sample.class, sample);
    }

    @Override
    public void onStop(Observation.Context context) {
        Timer.Sample sample = context.getRequired(Timer.Sample.class);

        List<String> tagParts = new ArrayList<>();
        for (KeyValue kv : context.getLowCardinalityKeyValues()) {
            tagParts.add(kv.key());
            tagParts.add(kv.value());
        }

        tagParts.add("error");
        tagParts.add(context.getError() != null
                ? context.getError().getClass().getSimpleName()
                : KeyValue.NONE_VALUE);

        Timer timer = meterRegistry.timer(
                context.getName(),
                tagParts.toArray(new String[0]));

        sample.stop(timer);
    }

    @Override
    public void onError(Observation.Context context) {
        String exceptionName = context.getError() != null
                ? context.getError().getClass().getSimpleName()
                : "unknown";

        List<String> tagParts = new ArrayList<>();
        for (KeyValue kv : context.getLowCardinalityKeyValues()) {
            tagParts.add(kv.key());
            tagParts.add(kv.value());
        }
        tagParts.add("exception");
        tagParts.add(exceptionName);

        meterRegistry.counter(
                context.getName() + ".error",
                tagParts.toArray(new String[0])
        ).increment();
    }
}
```

**Direction:** This handler references types that don't exist yet — `Observation`, `Observation.Context`, `ObservationHandler`, `KeyValue`. These are what we need to build:

Two key decisions here:

1. **Why bridge at the handler level, not inside each observation?** Because the observation itself is vendor-neutral — it has no dependency on `MeterRegistry`. Different handlers can produce metrics, traces, or logs from the same lifecycle events. The handler is the adapter between the abstract observation and the concrete concern.
2. **Why use `Timer.Sample` stored in Context?** The `Sample` captures start-time at `onStart()`, and the `Timer` is chosen at `onStop()`. This means tags can be set after the observation starts (e.g., the HTTP status code isn't known until the response is written), and the timer still records the full duration.

This **connects the Observation lifecycle to MeterRegistry metrics**. To make it work, we need to build:
- `KeyValue` — immutable key-value pairs for dimensional metadata
- `Observation` — the main interface with lifecycle methods, `Context`, and `Scope`
- `ObservationHandler` — the callback interface for lifecycle events
- `ObservationRegistry` — holds handlers and tracks the current scope per thread
- `SimpleObservation` / `SimpleObservationRegistry` — the implementations

---

## 21.2 KeyValue — The Dimensional Metadata

Each observation can carry **key-value pairs** that describe what is being observed. These are split into two categories:

- **Low cardinality** — bounded set of values (e.g., HTTP method: GET/POST/PUT). These become **metric tags** because the set of possible combinations is manageable.
- **High cardinality** — unbounded values (e.g., request URL `/api/users/42`, request ID `abc-123`). These are used for **trace span attributes** but must NOT become metric tags (they would cause cardinality explosion in time series databases).

**New file:** `iris-micrometer/src/main/java/com/iris/micrometer/observation/KeyValue.java`

```java
package com.iris.micrometer.observation;

import java.util.Objects;

public record KeyValue(String key, String value) implements Comparable<KeyValue> {

    public static final String NONE_VALUE = "none";

    public KeyValue {
        Objects.requireNonNull(key, "Key must not be null");
        Objects.requireNonNull(value, "Value must not be null");
    }

    public static KeyValue of(String key, String value) {
        return new KeyValue(key, value);
    }

    @Override
    public int compareTo(KeyValue other) {
        return this.key.compareTo(other.key);
    }

    @Override
    public String toString() {
        return "KeyValue{" + key + "=" + value + "}";
    }
}
```

We use a Java 17 `record` — the real Micrometer uses an interface (`KeyValue`) with an `ImmutableKeyValue` implementation. The record gives us `equals`, `hashCode`, and accessors for free.

---

## 21.3 ObservationHandler — The Extension Point

Handlers react to observation lifecycle events. This is where metrics, tracing, and logging plug in — each as a separate handler, all receiving the same events.

**New file:** `iris-micrometer/src/main/java/com/iris/micrometer/observation/ObservationHandler.java`

```java
package com.iris.micrometer.observation;

public interface ObservationHandler<T extends Observation.Context> {

    default void onStart(T context) {
    }

    default void onStop(T context) {
    }

    default void onError(T context) {
    }

    default void onScopeOpened(T context) {
    }

    default void onScopeClosed(T context) {
    }

    default boolean supportsContext(Observation.Context context) {
        return true;
    }
}
```

All methods have empty defaults — handlers override only the events they care about. The `supportsContext` method acts as a type filter: handlers can choose to process only specific context types.

**Handler dispatch order:**

```
Forward order:  onStart → onScopeOpened → onError
Reverse order:  onScopeClosed → onStop
```

The reverse order on close/stop mirrors stack-based cleanup — handlers that set up first tear down last (like AOP around advice or servlet filters).

---

## 21.4 ObservationRegistry — Handler Management + Scope Tracking

The registry serves two roles: (1) hold the registered handlers, and (2) track the current observation scope per thread via a `ThreadLocal`.

**New file:** `iris-micrometer/src/main/java/com/iris/micrometer/observation/ObservationRegistry.java`

```java
package com.iris.micrometer.observation;

import java.util.List;

public interface ObservationRegistry {

    ObservationRegistry NOOP = new ObservationRegistry() {
        private final ObservationConfig noopConfig = new ObservationConfig() {
            @Override
            public ObservationConfig observationHandler(ObservationHandler<?> handler) {
                return this;
            }
            @Override
            public List<ObservationHandler<?>> getObservationHandlers() {
                return List.of();
            }
        };

        @Override
        public Observation.Scope getCurrentObservationScope() { return null; }
        @Override
        public void setCurrentObservationScope(Observation.Scope scope) { }
        @Override
        public ObservationConfig observationConfig() { return noopConfig; }
        @Override
        public boolean isNoop() { return true; }
    };

    static ObservationRegistry create() {
        return new SimpleObservationRegistry();
    }

    Observation.Scope getCurrentObservationScope();
    void setCurrentObservationScope(Observation.Scope scope);
    ObservationConfig observationConfig();
    boolean isNoop();

    default Observation getCurrentObservation() {
        Observation.Scope scope = getCurrentObservationScope();
        return scope != null ? scope.getCurrentObservation() : null;
    }

    interface ObservationConfig {
        ObservationConfig observationHandler(ObservationHandler<?> handler);
        List<ObservationHandler<?>> getObservationHandlers();
    }
}
```

**New file:** `iris-micrometer/src/main/java/com/iris/micrometer/observation/SimpleObservationRegistry.java`

```java
package com.iris.micrometer.observation;

import java.util.List;
import java.util.concurrent.CopyOnWriteArrayList;

public class SimpleObservationRegistry implements ObservationRegistry {

    private final ThreadLocal<Observation.Scope> currentScope = new ThreadLocal<>();
    private final SimpleObservationConfig config = new SimpleObservationConfig();

    @Override
    public Observation.Scope getCurrentObservationScope() {
        return currentScope.get();
    }

    @Override
    public void setCurrentObservationScope(Observation.Scope scope) {
        currentScope.set(scope);
    }

    @Override
    public ObservationConfig observationConfig() {
        return config;
    }

    @Override
    public boolean isNoop() {
        return config.getObservationHandlers().isEmpty();
    }

    private static class SimpleObservationConfig implements ObservationConfig {
        private final List<ObservationHandler<?>> handlers = new CopyOnWriteArrayList<>();

        @Override
        public ObservationConfig observationHandler(ObservationHandler<?> handler) {
            handlers.add(handler);
            return this;
        }

        @Override
        public List<ObservationHandler<?>> getObservationHandlers() {
            return handlers;
        }
    }
}
```

A critical optimization: `isNoop()` returns `true` when no handlers are registered. This means all observations created from an empty registry become `Observation.NOOP` — avoiding context/scope allocation when nobody is listening.

---

## 21.5 Observation — The Unified Interface

This is the main type that users interact with. It defines the lifecycle, holds the `Context` (mutable state bag), and provides `Scope` (ThreadLocal tracking).

**New file:** `iris-micrometer/src/main/java/com/iris/micrometer/observation/Observation.java`

```java
package com.iris.micrometer.observation;

import java.util.*;
import java.util.concurrent.ConcurrentHashMap;
import java.util.function.Supplier;

public interface Observation {

    Observation NOOP = new Observation() {
        @Override public Observation start() { return this; }
        @Override public Observation stop() { return this; }
        @Override public Observation error(Throwable error) { return this; }
        @Override public Observation lowCardinalityKeyValue(String key, String value) { return this; }
        @Override public Observation highCardinalityKeyValue(String key, String value) { return this; }
        @Override public Scope openScope() { return Scope.NOOP; }
        @Override public Context getContext() { return new Context(); }
    };

    static Observation createNotStarted(String name, ObservationRegistry registry) {
        return createNotStarted(name, Context::new, registry);
    }

    static Observation createNotStarted(String name,
                                        Supplier<? extends Context> contextSupplier,
                                        ObservationRegistry registry) {
        if (registry == null || registry.isNoop()) {
            return NOOP;
        }
        Context context = contextSupplier.get();
        return new SimpleObservation(name, registry, context);
    }

    static Observation start(String name, ObservationRegistry registry) {
        return createNotStarted(name, registry).start();
    }

    Observation start();
    Observation stop();
    Observation error(Throwable error);
    Observation lowCardinalityKeyValue(String key, String value);
    Observation highCardinalityKeyValue(String key, String value);
    Scope openScope();
    Context getContext();

    default void observe(Runnable task) {
        start();
        try (Scope scope = openScope()) {
            task.run();
        } catch (Exception e) {
            error(e);
            throw e;
        } finally {
            stop();
        }
    }

    default <T> T observe(Supplier<T> supplier) {
        start();
        try (Scope scope = openScope()) {
            return supplier.get();
        } catch (Exception e) {
            error(e);
            throw e;
        } finally {
            stop();
        }
    }

    // ─── Context ────────────────────────────────────────────────

    class Context {
        private String name;
        private String contextualName;
        private Throwable error;
        private final Map<String, KeyValue> lowCardinalityKeyValues = new ConcurrentHashMap<>();
        private final Map<String, KeyValue> highCardinalityKeyValues = new ConcurrentHashMap<>();
        private final Map<Object, Object> data = new ConcurrentHashMap<>();

        public String getName() { return name; }
        public void setName(String name) { this.name = name; }
        public String getContextualName() { return contextualName; }
        public void setContextualName(String contextualName) { this.contextualName = contextualName; }
        public Throwable getError() { return error; }
        public void setError(Throwable error) { this.error = error; }

        public Context addLowCardinalityKeyValue(KeyValue keyValue) {
            lowCardinalityKeyValues.put(keyValue.key(), keyValue);
            return this;
        }
        public Context addHighCardinalityKeyValue(KeyValue keyValue) {
            highCardinalityKeyValues.put(keyValue.key(), keyValue);
            return this;
        }
        public Collection<KeyValue> getLowCardinalityKeyValues() {
            List<KeyValue> sorted = new ArrayList<>(lowCardinalityKeyValues.values());
            Collections.sort(sorted);
            return Collections.unmodifiableList(sorted);
        }
        public Collection<KeyValue> getHighCardinalityKeyValues() {
            List<KeyValue> sorted = new ArrayList<>(highCardinalityKeyValues.values());
            Collections.sort(sorted);
            return Collections.unmodifiableList(sorted);
        }
        public Collection<KeyValue> getAllKeyValues() {
            List<KeyValue> all = new ArrayList<>(lowCardinalityKeyValues.values());
            all.addAll(highCardinalityKeyValues.values());
            Collections.sort(all);
            return Collections.unmodifiableList(all);
        }

        public void put(Object key, Object value) { data.put(key, value); }
        @SuppressWarnings("unchecked")
        public <T> T get(Object key) { return (T) data.get(key); }
        public <T> T getRequired(Object key) {
            T value = get(key);
            if (value == null) {
                throw new IllegalStateException(
                        "Context does not contain a required value for key [" + key + "]");
            }
            return value;
        }
    }

    // ─── Scope ──────────────────────────────────────────────────

    interface Scope extends AutoCloseable {
        Scope NOOP = new Scope() {
            @Override public void close() { }
            @Override public Observation getCurrentObservation() { return Observation.NOOP; }
        };

        @Override void close();
        Observation getCurrentObservation();
    }
}
```

The `observe()` methods encode the full lifecycle template:

```
start() → openScope() → execute → [error()] → scope.close() → stop()
```

Note the order when an exception occurs: `scope.close()` happens first (try-with-resources), then `error()` (catch block), then `stop()` (finally block). This is important because `onScopeClosed` should restore the ThreadLocal before error/stop processing.

---

## 21.6 SimpleObservation — The Implementation

**New file:** `iris-micrometer/src/main/java/com/iris/micrometer/observation/SimpleObservation.java`

```java
package com.iris.micrometer.observation;

import java.util.ArrayList;
import java.util.List;
import java.util.ListIterator;

class SimpleObservation implements Observation {

    private final ObservationRegistry registry;
    private final Context context;
    private final List<ObservationHandler<Context>> handlers;

    @SuppressWarnings("unchecked")
    SimpleObservation(String name, ObservationRegistry registry, Context context) {
        this.registry = registry;
        this.context = context;
        this.context.setName(name);

        this.handlers = new ArrayList<>();
        for (ObservationHandler<?> handler : registry.observationConfig().getObservationHandlers()) {
            if (handler.supportsContext(context)) {
                handlers.add((ObservationHandler<Context>) handler);
            }
        }
    }

    @Override
    public Observation start() {
        for (ObservationHandler<Context> handler : handlers) {
            handler.onStart(context);
        }
        return this;
    }

    @Override
    public Observation stop() {
        ListIterator<ObservationHandler<Context>> it =
                handlers.listIterator(handlers.size());
        while (it.hasPrevious()) {
            it.previous().onStop(context);
        }
        return this;
    }

    @Override
    public Observation error(Throwable error) {
        context.setError(error);
        for (ObservationHandler<Context> handler : handlers) {
            handler.onError(context);
        }
        return this;
    }

    @Override
    public Observation lowCardinalityKeyValue(String key, String value) {
        context.addLowCardinalityKeyValue(KeyValue.of(key, value));
        return this;
    }

    @Override
    public Observation highCardinalityKeyValue(String key, String value) {
        context.addHighCardinalityKeyValue(KeyValue.of(key, value));
        return this;
    }

    @Override
    public Scope openScope() {
        SimpleScope scope = new SimpleScope(registry, this);
        for (ObservationHandler<Context> handler : handlers) {
            handler.onScopeOpened(context);
        }
        return scope;
    }

    @Override
    public Context getContext() { return context; }

    private class SimpleScope implements Scope {
        private final ObservationRegistry registry;
        private final Observation observation;
        private final Scope previousScope;

        SimpleScope(ObservationRegistry registry, Observation observation) {
            this.registry = registry;
            this.observation = observation;
            this.previousScope = registry.getCurrentObservationScope();
            registry.setCurrentObservationScope(this);
        }

        @Override
        public void close() {
            ListIterator<ObservationHandler<Context>> it =
                    handlers.listIterator(handlers.size());
            while (it.hasPrevious()) {
                it.previous().onScopeClosed(context);
            }
            registry.setCurrentObservationScope(previousScope);
        }

        @Override
        public Observation getCurrentObservation() { return observation; }
    }
}
```

The `SimpleScope` forms a **linked list** through `previousScope` — each scope saves the previous one on construction and restores it on close. This enables nested observations (e.g., an HTTP request containing a database call) to maintain proper parent-child ordering through the ThreadLocal stack.

---

## 21.7 Try It Yourself

<details>
<summary>Challenge 1: Implement a logging ObservationHandler that prints lifecycle events</summary>

Create a handler that prints observation lifecycle events to `System.out`:

```java
public class LoggingObservationHandler implements ObservationHandler<Observation.Context> {

    @Override
    public void onStart(Observation.Context context) {
        System.out.println("[START] " + context.getName());
    }

    @Override
    public void onStop(Observation.Context context) {
        System.out.println("[STOP]  " + context.getName()
                + (context.getError() != null ? " ERROR: " + context.getError().getMessage() : " OK"));
    }

    @Override
    public void onError(Observation.Context context) {
        System.out.println("[ERROR] " + context.getName() + ": " + context.getError().getMessage());
    }

    @Override
    public void onScopeOpened(Observation.Context context) {
        System.out.println("[SCOPE] " + context.getName() + " opened");
    }

    @Override
    public void onScopeClosed(Observation.Context context) {
        System.out.println("[SCOPE] " + context.getName() + " closed");
    }
}
```

Register it alongside the metrics handler:
```java
registry.observationConfig()
    .observationHandler(new DefaultMeterObservationHandler(meterRegistry))
    .observationHandler(new LoggingObservationHandler());
```

Both handlers receive events from the same observation — demonstrating the "instrument once, observe many ways" pattern.

</details>

<details>
<summary>Challenge 2: Use nested observations to track an HTTP request that makes a database call</summary>

```java
ObservationRegistry registry = ObservationRegistry.create();
registry.observationConfig()
    .observationHandler(new DefaultMeterObservationHandler(meterRegistry));

// Outer: HTTP request
Observation httpObs = Observation.createNotStarted("http.server.requests", registry)
        .lowCardinalityKeyValue("method", "GET")
        .lowCardinalityKeyValue("uri", "/api/users");

httpObs.start();
try (Observation.Scope httpScope = httpObs.openScope()) {
    // The HTTP request is now "current" on this thread
    assert registry.getCurrentObservation() == httpObs;

    // Inner: database query (within the HTTP request)
    Observation dbObs = Observation.createNotStarted("db.query", registry)
            .lowCardinalityKeyValue("db.system", "postgresql");
    dbObs.start();
    try (Observation.Scope dbScope = dbObs.openScope()) {
        // The DB query is now "current"
        assert registry.getCurrentObservation() == dbObs;
        // ... execute query ...
    }
    dbObs.stop();

    // HTTP request is restored as current
    assert registry.getCurrentObservation() == httpObs;
}
httpObs.lowCardinalityKeyValue("status", "200");
httpObs.stop();

// Both observations produced independent Timer metrics
```

</details>

---

## 21.8 Tests

### Unit Tests

**New file:** `iris-micrometer/src/test/java/com/iris/micrometer/observation/KeyValueTest.java`

```java
class KeyValueTest {
    @Test void shouldCreateKeyValue()
    @Test void shouldRejectNullKey()
    @Test void shouldRejectNullValue()
    @Test void shouldBeEqualForSameKeyAndValue()
    @Test void shouldNotBeEqualForDifferentValues()
    @Test void shouldSortByKey()
    @Test void shouldHaveNoneValueSentinel()
}
```

**New file:** `iris-micrometer/src/test/java/com/iris/micrometer/observation/ObservationTest.java`

Tests organized in `@Nested` groups:

- **Lifecycle** — `shouldCallHandlerOnStart`, `shouldCallHandlerOnStop_InReverseOrder`, `shouldCallHandlerOnError`, `shouldCallFullLifecycle_WhenUsingObserveRunnable`, `shouldCallErrorBeforeStop_WhenObserveThrows`, `shouldReturnResult_WhenUsingObserveSupplier`
- **ContextTests** — `shouldSetNameFromCreation`, `shouldStoreLowCardinalityKeyValues`, `shouldStoreHighCardinalityKeyValues`, `shouldReturnAllKeyValuesSorted`, `shouldReplaceDuplicateKeys`, `shouldStoreAndRetrieveArbitraryData`, `shouldThrowOnMissingRequiredData`, `shouldUseCustomContext`
- **ScopeTracking** — `shouldSetCurrentObservation_WhenScopeOpened`, `shouldSupportNestedScopes`, `shouldCallScopeHandlers_InCorrectOrder`
- **NoopBehavior** — `shouldReturnNoop_WhenRegistryIsNoop`, `shouldReturnNoop_WhenNoHandlersRegistered`, `shouldReturnNoop_WhenRegistryIsNull`, `shouldSafelyChainNoopCalls`, `shouldReturnNoopScope`
- **HandlerFiltering** — `shouldOnlyDispatchToSupportingHandlers`

**New file:** `iris-micrometer/src/test/java/com/iris/micrometer/core/observation/DefaultMeterObservationHandlerTest.java`

```java
class DefaultMeterObservationHandlerTest {
    @Test void shouldRecordTimerOnStop_WhenObservationCompletes()
    @Test void shouldIncludeErrorTag_WhenErrorOccurred()
    @Test void shouldIncludeNoneErrorTag_WhenNoError()
    @Test void shouldIncludeLowCardinalityTags_OnTimer()
    @Test void shouldIncrementErrorCounter_WhenErrorSignaled()
    @Test void shouldRecordMultipleObservations_AsSeparateTimerRecordings()
    @Test void shouldCreateDistinctTimers_ForDifferentTags()
    @Test void shouldNotCreateMetrics_WhenRegistryIsNoop()
    @Test void shouldWorkWithObserveConvenience()
}
```

### Integration Tests

**New file:** `iris-micrometer/src/test/java/com/iris/micrometer/observation/integration/ObservationMetricsIntegrationTest.java`

Tests the full observation-to-metrics pipeline:

```java
class ObservationMetricsIntegrationTest {
    @Test void shouldRecordSuccessfulHttpRequest()
    @Test void shouldRecordFailedHttpRequest_WithErrorMetrics()
    @Test void shouldSupportMultipleHandlers()
    @Test void shouldTrackNestedObservations()
    @Test void shouldHandleObserveWithError_RecordingBothTimerAndCounter()
    @Test void shouldReturnSupplierResult_WithMetricsRecorded()
    @Test void shouldAccumulateMetrics_AcrossMultipleObservations()
    @Test void shouldUseHighCardinalityForTracing_NotForMetrics()
}
```

**Run:** `./gradlew :iris-micrometer:test` — expected: all 99 tests pass (52 from ch20 + 47 new)

---

## 21.9 Why This Works

> ★ **Insight** -------------------------------------------
> **Instrument Once, Observe Many Ways.** The Observation API inverts the traditional instrumentation model. Instead of each concern (metrics, tracing, logging) requiring its own instrumentation call, a single `Observation` lifecycle drives all of them through pluggable handlers. This is the Observer pattern applied to observability — and it's exactly how Spring Boot 3.x instruments all of its internal components. The `DefaultMeterObservationHandler` produces `Timer` and `Counter` metrics; a tracing handler (not built here) would produce spans; a logging handler would emit structured log entries — all from the same `observe(() -> doWork())` call.
> -----------------------------------------------------------

> ★ **Insight** -------------------------------------------
> **ThreadLocal Scope Chain for Nested Observations.** Scopes form a linked list through `previousScope`, stored in a `ThreadLocal` on the registry. When you open a nested scope (e.g., a database call inside an HTTP request), the inner scope saves the outer scope and becomes current. When the inner scope closes, it restores the outer. This is the same pattern as servlet request attributes or MDC contexts — and it's what enables distributed tracing to build parent-child span hierarchies. The trade-off: ThreadLocal-based context doesn't cross thread boundaries automatically (reactive/async code needs the separate `context-propagation` library for that).
> -----------------------------------------------------------

> ★ **Insight** -------------------------------------------
> **Low vs. High Cardinality — The Cardinality Wall.** Key values are split into low and high cardinality because metrics and traces have fundamentally different storage models. A time series database stores one series per unique tag combination — if `uri` is a tag with 10,000 unique values and `method` has 5, you get 50,000 time series (and growing). This is called "cardinality explosion" and it crashes monitoring systems. By separating key values, `DefaultMeterObservationHandler` only uses low-cardinality values for metric tags, while a tracing handler can use both. The `NONE_VALUE` sentinel ("none") is used for the `error` tag on successful operations — a fixed, bounded value that doesn't increase cardinality.
> -----------------------------------------------------------

---

## 21.10 What We Enhanced

| Aspect | Before (ch20) | Current (ch21) | Real Framework |
|--------|---------------|----------------|----------------|
| Instrumentation model | Manual: create `Timer`/`Counter` explicitly at each instrumentation point | Unified: create an `Observation`, handlers produce metrics automatically | Same — `Observation.observe()` is the primary instrumentation API since Micrometer 1.10 |
| Dimensional metadata | `Tags` — flat list of key-value pairs for metrics only | `KeyValue` with low/high cardinality separation — metrics use low, traces use both | Same — `KeyValue` in `micrometer-commons`, `KeyValue.java:30` |
| Metrics module scope | Standalone metrics library (Counter, Timer, Gauge, MeterRegistry) | + Observation API + handler bridge — observation is a higher-level abstraction that drives the metrics library | Same — `micrometer-observation` depends on `micrometer-commons`, while `micrometer-core` provides the bridge handler |
| Thread context | None — no concept of "current" meter | `Scope` + `ThreadLocal` — track the current observation per thread for nested parent-child relationships | Same — `SimpleObservationRegistry` uses ThreadLocal, `SimpleObservationRegistry.java:28` |

---

## 21.11 Connection to Real Framework

| Simplified Code | Real Framework Code | File:Line | Key Difference |
|----------------|--------------------|-----------| ---------------|
| `KeyValue` (record) | `KeyValue` (interface + `ImmutableKeyValue`) | `KeyValue.java:30` | Real has `KeyName` typed keys, `ValidatedKeyValue`, and factory methods for predicates |
| `Observation` | `Observation` | `Observation.java:49` | Real has `ObservationConvention` (naming strategy), `ObservationFilter` (mutate context before stop), `Event` signaling, `ObservationPredicate` (enable/disable observations) |
| `Observation.Context` | `Observation.Context` | `Observation.java:993` | Real has `parentObservation` auto-set from current scope, separate `contextualName`, and `toString()` showing all key values |
| `Observation.Scope` | `Observation.Scope` | `Observation.java:901` | Real has `reset()` (clear all scopes) and `makeCurrent()` (rebuild scope chain) for context propagation |
| `observe(Runnable)` | `observe(Runnable)` | `Observation.java:561` | Real also has `observeChecked()` for checked exceptions and `scoped()` for scope-only operations |
| `ObservationHandler` | `ObservationHandler` | `ObservationHandler.java:35` | Real adds `onEvent(Event, Context)`, `onScopeReset()`, and composite handlers (`FirstMatching`, `AllMatching`) |
| `ObservationRegistry` | `ObservationRegistry` | `ObservationRegistry.java:35` | Real has `ObservationPredicate`, `ObservationFilter`, `ObservationConvention` on config |
| `SimpleObservationRegistry` | `SimpleObservationRegistry` | `SimpleObservationRegistry.java:28` | Real `isNoop()` also checks parent no-op state, line 71 |
| `SimpleObservation` | `SimpleObservation` | `SimpleObservation.java:42` | Real applies conventions at start+stop, applies filters before stop handlers, and stores scopes per-thread in a Map |
| `DefaultMeterObservationHandler` | `DefaultMeterObservationHandler` | `DefaultMeterObservationHandler.java:42` | Real also creates a `LongTaskTimer` for tracking in-flight operations, and handles `onEvent` with Counters |

---

## 21.12 Complete Code

### Production Code

#### File: `iris-micrometer/src/main/java/com/iris/micrometer/observation/KeyValue.java` [NEW]

```java
package com.iris.micrometer.observation;

import java.util.Objects;

/**
 * An immutable key-value pair used to add dimensional metadata to observations.
 * <p>
 * Key values are split into two categories on {@link Observation.Context}:
 * <ul>
 *   <li><b>Low cardinality</b> — bounded set of possible values (used for metric
 *       tags/dimensions, e.g., HTTP method, status code template)</li>
 *   <li><b>High cardinality</b> — unbounded values (used for span attributes/log
 *       fields, e.g., full URL, request ID)</li>
 * </ul>
 * Only low cardinality key values become metric tags — high cardinality would
 * cause cardinality explosion in time series databases.
 *
 * @see Observation.Context#addLowCardinalityKeyValue(KeyValue)
 * @see Observation.Context#addHighCardinalityKeyValue(KeyValue)
 */
public record KeyValue(String key, String value) implements Comparable<KeyValue> {

    /** Sentinel value indicating "no value" or "not applicable". */
    public static final String NONE_VALUE = "none";

    public KeyValue {
        Objects.requireNonNull(key, "Key must not be null");
        Objects.requireNonNull(value, "Value must not be null");
    }

    public static KeyValue of(String key, String value) {
        return new KeyValue(key, value);
    }

    @Override
    public int compareTo(KeyValue other) {
        return this.key.compareTo(other.key);
    }

    @Override
    public String toString() {
        return "KeyValue{" + key + "=" + value + "}";
    }
}
```

#### File: `iris-micrometer/src/main/java/com/iris/micrometer/observation/Observation.java` [NEW]

```java
package com.iris.micrometer.observation;

import java.util.*;
import java.util.concurrent.ConcurrentHashMap;
import java.util.function.Supplier;

/**
 * The unified observability abstraction — instrument once, get metrics, traces,
 * and logs via pluggable {@link ObservationHandler}s.
 * <p>
 * An observation has a clear lifecycle:
 * <pre>
 *   Observation obs = Observation.createNotStarted("my.operation", registry);
 *   obs.lowCardinalityKeyValue("method", "GET");
 *   obs.start();                    // handlers.onStart()
 *   try (Scope scope = obs.openScope()) {  // handlers.onScopeOpened()
 *       // ... do work ...
 *   }                               // handlers.onScopeClosed()
 *   obs.stop();                     // handlers.onStop()
 * </pre>
 * Or use the convenience method:
 * <pre>
 *   Observation.createNotStarted("my.operation", registry)
 *       .observe(() -&gt; doWork());
 * </pre>
 */
public interface Observation {

    /** A no-op observation that silently ignores all lifecycle calls. */
    Observation NOOP = new Observation() {
        @Override
        public Observation start() {
            return this;
        }

        @Override
        public Observation stop() {
            return this;
        }

        @Override
        public Observation error(Throwable error) {
            return this;
        }

        @Override
        public Observation lowCardinalityKeyValue(String key, String value) {
            return this;
        }

        @Override
        public Observation highCardinalityKeyValue(String key, String value) {
            return this;
        }

        @Override
        public Scope openScope() {
            return Scope.NOOP;
        }

        @Override
        public Context getContext() {
            return new Context();
        }
    };

    /**
     * Creates a not-yet-started observation with a default {@link Context}.
     * Returns {@link #NOOP} if the registry is a no-op.
     */
    static Observation createNotStarted(String name, ObservationRegistry registry) {
        return createNotStarted(name, Context::new, registry);
    }

    /**
     * Creates a not-yet-started observation with a custom {@link Context}.
     * Returns {@link #NOOP} if the registry is a no-op.
     */
    static Observation createNotStarted(String name,
                                        Supplier<? extends Context> contextSupplier,
                                        ObservationRegistry registry) {
        if (registry == null || registry.isNoop()) {
            return NOOP;
        }
        Context context = contextSupplier.get();
        return new SimpleObservation(name, registry, context);
    }

    /**
     * Creates and immediately starts an observation with a default context.
     */
    static Observation start(String name, ObservationRegistry registry) {
        return createNotStarted(name, registry).start();
    }

    /**
     * Creates and immediately starts an observation with a custom context.
     */
    static Observation start(String name,
                             Supplier<? extends Context> contextSupplier,
                             ObservationRegistry registry) {
        return createNotStarted(name, contextSupplier, registry).start();
    }

    /** Starts the observation — notifies handlers via {@code onStart}. */
    Observation start();

    /** Stops the observation — notifies handlers via {@code onStop} in reverse order. */
    Observation stop();

    /** Signals that an error occurred — notifies handlers via {@code onError}. */
    Observation error(Throwable error);

    /** Adds a low-cardinality key-value pair (used for metric tags). */
    Observation lowCardinalityKeyValue(String key, String value);

    /** Adds a high-cardinality key-value pair (used for trace span attributes). */
    Observation highCardinalityKeyValue(String key, String value);

    /** Opens a scope — sets this observation as current on the calling thread. */
    Scope openScope();

    /** Returns the context holding this observation's state. */
    Context getContext();

    /**
     * Convenience method: start, open scope, run the task, close scope, stop.
     * If the task throws, {@link #error(Throwable)} is called before rethrowing.
     */
    default void observe(Runnable task) {
        start();
        try (Scope scope = openScope()) {
            task.run();
        }
        catch (Exception e) {
            error(e);
            throw e;
        }
        finally {
            stop();
        }
    }

    /**
     * Convenience method: start, open scope, run the supplier, close scope, stop.
     * If the supplier throws, {@link #error(Throwable)} is called before rethrowing.
     */
    default <T> T observe(Supplier<T> supplier) {
        start();
        try (Scope scope = openScope()) {
            return supplier.get();
        }
        catch (Exception e) {
            error(e);
            throw e;
        }
        finally {
            stop();
        }
    }

    // ─── Inner types ────────────────────────────────────────────────

    /**
     * Mutable bag of state for an observation — holds the name, error, and
     * low/high-cardinality key-value pairs.
     * <p>
     * Handlers can also store arbitrary data (e.g., a {@code Timer.Sample})
     * via {@link #put(Object, Object)} and {@link #get(Object)}.
     */
    class Context {

        private String name;
        private String contextualName;
        private Throwable error;

        private final Map<String, KeyValue> lowCardinalityKeyValues = new ConcurrentHashMap<>();
        private final Map<String, KeyValue> highCardinalityKeyValues = new ConcurrentHashMap<>();

        /** Arbitrary handler-specific data (e.g., Timer.Sample, Span). */
        private final Map<Object, Object> data = new ConcurrentHashMap<>();

        // ── Name ──

        public String getName() {
            return name;
        }

        public void setName(String name) {
            this.name = name;
        }

        public String getContextualName() {
            return contextualName;
        }

        public void setContextualName(String contextualName) {
            this.contextualName = contextualName;
        }

        // ── Error ──

        public Throwable getError() {
            return error;
        }

        public void setError(Throwable error) {
            this.error = error;
        }

        // ── Key values ──

        public Context addLowCardinalityKeyValue(KeyValue keyValue) {
            lowCardinalityKeyValues.put(keyValue.key(), keyValue);
            return this;
        }

        public Context addHighCardinalityKeyValue(KeyValue keyValue) {
            highCardinalityKeyValues.put(keyValue.key(), keyValue);
            return this;
        }

        public Collection<KeyValue> getLowCardinalityKeyValues() {
            List<KeyValue> sorted = new ArrayList<>(lowCardinalityKeyValues.values());
            Collections.sort(sorted);
            return Collections.unmodifiableList(sorted);
        }

        public Collection<KeyValue> getHighCardinalityKeyValues() {
            List<KeyValue> sorted = new ArrayList<>(highCardinalityKeyValues.values());
            Collections.sort(sorted);
            return Collections.unmodifiableList(sorted);
        }

        public Collection<KeyValue> getAllKeyValues() {
            List<KeyValue> all = new ArrayList<>(lowCardinalityKeyValues.values());
            all.addAll(highCardinalityKeyValues.values());
            Collections.sort(all);
            return Collections.unmodifiableList(all);
        }

        // ── Arbitrary data ──

        public void put(Object key, Object value) {
            data.put(key, value);
        }

        @SuppressWarnings("unchecked")
        public <T> T get(Object key) {
            return (T) data.get(key);
        }

        public <T> T getRequired(Object key) {
            T value = get(key);
            if (value == null) {
                throw new IllegalStateException(
                        "Context does not contain a required value for key [" + key + "]");
            }
            return value;
        }

        @Override
        public String toString() {
            return "Context{name='" + name + "'}";
        }
    }

    /**
     * A scope that tracks the current observation on the calling thread.
     * <p>
     * Scopes are {@link AutoCloseable} — use them in try-with-resources to
     * ensure proper cleanup. Scopes form a linked list via the registry's
     * ThreadLocal, enabling nested observations.
     */
    interface Scope extends AutoCloseable {

        /** A no-op scope that does nothing on close. */
        Scope NOOP = new Scope() {
            @Override
            public void close() {
            }

            @Override
            public Observation getCurrentObservation() {
                return Observation.NOOP;
            }
        };

        /**
         * Closes this scope — restores the previous scope on the current thread
         * and notifies handlers via {@code onScopeClosed}.
         */
        @Override
        void close();

        /**
         * Returns the observation that this scope belongs to.
         */
        Observation getCurrentObservation();
    }
}
```

#### File: `iris-micrometer/src/main/java/com/iris/micrometer/observation/ObservationHandler.java` [NEW]

```java
package com.iris.micrometer.observation;

/**
 * Handler that reacts to {@link Observation} lifecycle events.
 * <p>
 * This is the extension point where metrics, tracing, and logging plug in.
 * Register handlers on {@link ObservationRegistry.ObservationConfig} and they
 * will be notified of every observation lifecycle transition.
 * <p>
 * <b>Call order convention:</b>
 * <ul>
 *   <li>{@code onStart}, {@code onScopeOpened}, {@code onError} — called in
 *       <em>forward</em> (registration) order</li>
 *   <li>{@code onScopeClosed}, {@code onStop} — called in <em>reverse</em>
 *       order (stack-based cleanup, like AOP around advice)</li>
 * </ul>
 *
 * @param <T> the context type this handler can process
 */
public interface ObservationHandler<T extends Observation.Context> {

    /**
     * Called when an observation is started.
     */
    default void onStart(T context) {
    }

    /**
     * Called when an observation is stopped (final lifecycle event).
     */
    default void onStop(T context) {
    }

    /**
     * Called when an error is signaled on the observation.
     */
    default void onError(T context) {
    }

    /**
     * Called when a scope is opened (ThreadLocal set to this observation).
     */
    default void onScopeOpened(T context) {
    }

    /**
     * Called when a scope is closed (ThreadLocal restored to previous).
     */
    default void onScopeClosed(T context) {
    }

    /**
     * Determines whether this handler supports the given context type.
     * Only contexts for which this returns {@code true} will be dispatched
     * to this handler.
     *
     * @param context the observation context
     * @return {@code true} if this handler should receive events for the context
     */
    default boolean supportsContext(Observation.Context context) {
        return true;
    }
}
```

#### File: `iris-micrometer/src/main/java/com/iris/micrometer/observation/ObservationRegistry.java` [NEW]

```java
package com.iris.micrometer.observation;

import java.util.List;

/**
 * Registry that manages {@link ObservationHandler} instances and tracks the
 * current {@link Observation.Scope} via a ThreadLocal.
 * <p>
 * The registry serves two roles:
 * <ol>
 *   <li><b>Configuration</b> — holds the list of registered handlers via
 *       {@link #observationConfig()}</li>
 *   <li><b>Scope tracking</b> — maintains the current observation scope per
 *       thread, enabling nested observations to form a parent-child chain</li>
 * </ol>
 */
public interface ObservationRegistry {

    /** A no-op registry that produces no-op observations. */
    ObservationRegistry NOOP = new ObservationRegistry() {

        private final ObservationConfig noopConfig = new ObservationConfig() {
            @Override
            public ObservationConfig observationHandler(ObservationHandler<?> handler) {
                return this;
            }

            @Override
            public List<ObservationHandler<?>> getObservationHandlers() {
                return List.of();
            }
        };

        @Override
        public Observation.Scope getCurrentObservationScope() {
            return null;
        }

        @Override
        public void setCurrentObservationScope(Observation.Scope scope) {
        }

        @Override
        public ObservationConfig observationConfig() {
            return noopConfig;
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
     * Returns the current observation scope for the calling thread.
     */
    Observation.Scope getCurrentObservationScope();

    /**
     * Sets the current observation scope for the calling thread.
     */
    void setCurrentObservationScope(Observation.Scope scope);

    /**
     * Returns the configuration for this registry.
     */
    ObservationConfig observationConfig();

    /**
     * Returns {@code true} if this registry is a no-op (either the NOOP
     * singleton or a registry with no handlers registered).
     */
    boolean isNoop();

    /**
     * Returns the currently active observation on this thread, or {@code null}
     * if no observation scope is open.
     */
    default Observation getCurrentObservation() {
        Observation.Scope scope = getCurrentObservationScope();
        return scope != null ? scope.getCurrentObservation() : null;
    }

    /**
     * Configuration for an {@link ObservationRegistry} — holds handlers.
     */
    interface ObservationConfig {

        /**
         * Registers a handler that will receive observation lifecycle events.
         *
         * @param handler the handler to register
         * @return this config for fluent chaining
         */
        ObservationConfig observationHandler(ObservationHandler<?> handler);

        /**
         * Returns all registered handlers.
         */
        List<ObservationHandler<?>> getObservationHandlers();
    }
}
```

#### File: `iris-micrometer/src/main/java/com/iris/micrometer/observation/SimpleObservation.java` [NEW]

```java
package com.iris.micrometer.observation;

import java.util.ArrayList;
import java.util.List;
import java.util.ListIterator;

/**
 * Default implementation of {@link Observation} that dispatches lifecycle
 * events to matching {@link ObservationHandler}s.
 * <p>
 * <b>Handler dispatch order:</b>
 * <ul>
 *   <li>{@code onStart}, {@code onScopeOpened}, {@code onError} — forward order</li>
 *   <li>{@code onScopeClosed}, {@code onStop} — reverse order (stack unwinding)</li>
 * </ul>
 */
class SimpleObservation implements Observation {

    private final ObservationRegistry registry;
    private final Context context;
    private final List<ObservationHandler<Context>> handlers;

    @SuppressWarnings("unchecked")
    SimpleObservation(String name, ObservationRegistry registry, Context context) {
        this.registry = registry;
        this.context = context;
        this.context.setName(name);

        // Collect handlers that support this context type
        this.handlers = new ArrayList<>();
        for (ObservationHandler<?> handler : registry.observationConfig().getObservationHandlers()) {
            if (handler.supportsContext(context)) {
                handlers.add((ObservationHandler<Context>) handler);
            }
        }
    }

    @Override
    public Observation start() {
        for (ObservationHandler<Context> handler : handlers) {
            handler.onStart(context);
        }
        return this;
    }

    @Override
    public Observation stop() {
        // Reverse order — stack-based cleanup
        ListIterator<ObservationHandler<Context>> it =
                handlers.listIterator(handlers.size());
        while (it.hasPrevious()) {
            it.previous().onStop(context);
        }
        return this;
    }

    @Override
    public Observation error(Throwable error) {
        context.setError(error);
        for (ObservationHandler<Context> handler : handlers) {
            handler.onError(context);
        }
        return this;
    }

    @Override
    public Observation lowCardinalityKeyValue(String key, String value) {
        context.addLowCardinalityKeyValue(KeyValue.of(key, value));
        return this;
    }

    @Override
    public Observation highCardinalityKeyValue(String key, String value) {
        context.addHighCardinalityKeyValue(KeyValue.of(key, value));
        return this;
    }

    @Override
    public Scope openScope() {
        SimpleScope scope = new SimpleScope(registry, this);
        for (ObservationHandler<Context> handler : handlers) {
            handler.onScopeOpened(context);
        }
        return scope;
    }

    @Override
    public Context getContext() {
        return context;
    }

    // ─── Inner scope implementation ─────────────────────────────────

    /**
     * A scope that saves the previous scope on creation and restores it on
     * close, forming a linked list through the registry's ThreadLocal.
     */
    private class SimpleScope implements Scope {

        private final ObservationRegistry registry;
        private final Observation observation;
        private final Scope previousScope;

        SimpleScope(ObservationRegistry registry, Observation observation) {
            this.registry = registry;
            this.observation = observation;
            this.previousScope = registry.getCurrentObservationScope();
            registry.setCurrentObservationScope(this);
        }

        @Override
        public void close() {
            // Reverse order — mirror of onScopeOpened
            ListIterator<ObservationHandler<Context>> it =
                    handlers.listIterator(handlers.size());
            while (it.hasPrevious()) {
                it.previous().onScopeClosed(context);
            }
            registry.setCurrentObservationScope(previousScope);
        }

        @Override
        public Observation getCurrentObservation() {
            return observation;
        }
    }
}
```

#### File: `iris-micrometer/src/main/java/com/iris/micrometer/observation/SimpleObservationRegistry.java` [NEW]

```java
package com.iris.micrometer.observation;

import java.util.List;
import java.util.concurrent.CopyOnWriteArrayList;

/**
 * Default implementation of {@link ObservationRegistry} that uses a
 * {@link ThreadLocal} to track the current observation scope per thread.
 * <p>
 * <b>No-op optimization:</b> If no handlers are registered, the registry
 * reports itself as {@linkplain #isNoop() no-op}. This means all observations
 * created from it will be the {@link Observation#NOOP} singleton — a fast path
 * that avoids allocating context and scope objects when nobody is listening.
 */
public class SimpleObservationRegistry implements ObservationRegistry {

    private final ThreadLocal<Observation.Scope> currentScope = new ThreadLocal<>();
    private final SimpleObservationConfig config = new SimpleObservationConfig();

    @Override
    public Observation.Scope getCurrentObservationScope() {
        return currentScope.get();
    }

    @Override
    public void setCurrentObservationScope(Observation.Scope scope) {
        currentScope.set(scope);
    }

    @Override
    public ObservationConfig observationConfig() {
        return config;
    }

    /**
     * Returns {@code true} if no handlers are registered — meaning all
     * observations will be no-ops.
     */
    @Override
    public boolean isNoop() {
        return config.getObservationHandlers().isEmpty();
    }

    // ─── Inner config ───────────────────────────────────────────────

    private static class SimpleObservationConfig implements ObservationConfig {

        private final List<ObservationHandler<?>> handlers = new CopyOnWriteArrayList<>();

        @Override
        public ObservationConfig observationHandler(ObservationHandler<?> handler) {
            handlers.add(handler);
            return this;
        }

        @Override
        public List<ObservationHandler<?>> getObservationHandlers() {
            return handlers;
        }
    }
}
```

#### File: `iris-micrometer/src/main/java/com/iris/micrometer/core/observation/DefaultMeterObservationHandler.java` [NEW]

```java
package com.iris.micrometer.core.observation;

import com.iris.micrometer.core.MeterRegistry;
import com.iris.micrometer.core.Timer;
import com.iris.micrometer.observation.KeyValue;
import com.iris.micrometer.observation.Observation;
import com.iris.micrometer.observation.ObservationHandler;

import java.util.ArrayList;
import java.util.List;

/**
 * Bridges the {@link Observation} API to the {@link MeterRegistry} metrics system.
 * <p>
 * This is the key integration point — it demonstrates the "instrument once, get
 * metrics automatically" pattern:
 * <ul>
 *   <li><b>{@code onStart}</b> — creates a {@link Timer.Sample} and stores it
 *       in the observation context</li>
 *   <li><b>{@code onStop}</b> — stops the sample against a {@link Timer}
 *       registered with the observation's name and low-cardinality tags (plus
 *       an {@code error} tag indicating success/failure)</li>
 *   <li><b>{@code onError}</b> — increments a Counter named
 *       {@code "{name}.error"} for immediate error signaling</li>
 * </ul>
 * <p>
 * The Timer records the full operation duration (count + total time + max),
 * while the error Counter provides a quick signal for alerting.
 */
public class DefaultMeterObservationHandler implements ObservationHandler<Observation.Context> {

    private final MeterRegistry meterRegistry;

    public DefaultMeterObservationHandler(MeterRegistry meterRegistry) {
        this.meterRegistry = meterRegistry;
    }

    @Override
    public void onStart(Observation.Context context) {
        Timer.Sample sample = Timer.start();
        context.put(Timer.Sample.class, sample);
    }

    @Override
    public void onStop(Observation.Context context) {
        Timer.Sample sample = context.getRequired(Timer.Sample.class);

        // Build tag key-value pairs from low-cardinality key values
        List<String> tagParts = new ArrayList<>();
        for (KeyValue kv : context.getLowCardinalityKeyValues()) {
            tagParts.add(kv.key());
            tagParts.add(kv.value());
        }

        // Add error tag: exception class name or "none"
        tagParts.add("error");
        tagParts.add(context.getError() != null
                ? context.getError().getClass().getSimpleName()
                : KeyValue.NONE_VALUE);

        Timer timer = meterRegistry.timer(
                context.getName(),
                tagParts.toArray(new String[0]));

        sample.stop(timer);
    }

    @Override
    public void onError(Observation.Context context) {
        String exceptionName = context.getError() != null
                ? context.getError().getClass().getSimpleName()
                : "unknown";

        // Build tags from low-cardinality key values
        List<String> tagParts = new ArrayList<>();
        for (KeyValue kv : context.getLowCardinalityKeyValues()) {
            tagParts.add(kv.key());
            tagParts.add(kv.value());
        }
        tagParts.add("exception");
        tagParts.add(exceptionName);

        meterRegistry.counter(
                context.getName() + ".error",
                tagParts.toArray(new String[0])
        ).increment();
    }
}
```

### Test Code

#### File: `iris-micrometer/src/test/java/com/iris/micrometer/observation/KeyValueTest.java` [NEW]

```java
package com.iris.micrometer.observation;

import org.junit.jupiter.api.Test;

import java.util.Arrays;
import java.util.List;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

class KeyValueTest {

    @Test
    void shouldCreateKeyValue() {
        KeyValue kv = KeyValue.of("method", "GET");

        assertThat(kv.key()).isEqualTo("method");
        assertThat(kv.value()).isEqualTo("GET");
    }

    @Test
    void shouldRejectNullKey() {
        assertThatThrownBy(() -> KeyValue.of(null, "value"))
                .isInstanceOf(NullPointerException.class)
                .hasMessageContaining("Key");
    }

    @Test
    void shouldRejectNullValue() {
        assertThatThrownBy(() -> KeyValue.of("key", null))
                .isInstanceOf(NullPointerException.class)
                .hasMessageContaining("Value");
    }

    @Test
    void shouldBeEqualForSameKeyAndValue() {
        KeyValue kv1 = KeyValue.of("method", "GET");
        KeyValue kv2 = KeyValue.of("method", "GET");

        assertThat(kv1).isEqualTo(kv2);
        assertThat(kv1.hashCode()).isEqualTo(kv2.hashCode());
    }

    @Test
    void shouldNotBeEqualForDifferentValues() {
        KeyValue kv1 = KeyValue.of("method", "GET");
        KeyValue kv2 = KeyValue.of("method", "POST");

        assertThat(kv1).isNotEqualTo(kv2);
    }

    @Test
    void shouldSortByKey() {
        KeyValue b = KeyValue.of("b.status", "200");
        KeyValue a = KeyValue.of("a.method", "GET");
        KeyValue c = KeyValue.of("c.uri", "/api");

        List<KeyValue> sorted = Arrays.asList(b, a, c);
        sorted.sort(null); // uses Comparable

        assertThat(sorted).containsExactly(a, b, c);
    }

    @Test
    void shouldHaveNoneValueSentinel() {
        assertThat(KeyValue.NONE_VALUE).isEqualTo("none");
    }
}
```

#### File: `iris-micrometer/src/test/java/com/iris/micrometer/observation/ObservationTest.java` [NEW]

```java
package com.iris.micrometer.observation;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Nested;
import org.junit.jupiter.api.Test;

import java.util.ArrayList;
import java.util.List;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

class ObservationTest {

    private ObservationRegistry registry;

    @BeforeEach
    void setUp() {
        registry = ObservationRegistry.create();
    }

    // ── Lifecycle ────────────────────────────────────────────────────

    @Nested
    class Lifecycle {

        @Test
        void shouldCallHandlerOnStart() {
            List<String> events = new ArrayList<>();
            registry.observationConfig().observationHandler(new ObservationHandler<>() {
                @Override
                public void onStart(Observation.Context context) {
                    events.add("start:" + context.getName());
                }
            });

            Observation.createNotStarted("my.op", registry).start();

            assertThat(events).containsExactly("start:my.op");
        }

        @Test
        void shouldCallHandlerOnStop_InReverseOrder() {
            List<String> events = new ArrayList<>();
            registry.observationConfig()
                    .observationHandler(new ObservationHandler<>() {
                        @Override
                        public void onStop(Observation.Context context) {
                            events.add("stop:A");
                        }
                    })
                    .observationHandler(new ObservationHandler<>() {
                        @Override
                        public void onStop(Observation.Context context) {
                            events.add("stop:B");
                        }
                    });

            Observation obs = Observation.start("my.op", registry);
            obs.stop();

            // B registered second, but called first on stop (reverse order)
            assertThat(events).containsExactly("stop:B", "stop:A");
        }

        @Test
        void shouldCallHandlerOnError() {
            List<String> events = new ArrayList<>();
            registry.observationConfig().observationHandler(new ObservationHandler<>() {
                @Override
                public void onError(Observation.Context context) {
                    events.add("error:" + context.getError().getMessage());
                }
            });

            Observation obs = Observation.start("my.op", registry);
            obs.error(new RuntimeException("boom"));

            assertThat(events).containsExactly("error:boom");
        }

        @Test
        void shouldCallFullLifecycle_WhenUsingObserveRunnable() {
            List<String> events = new ArrayList<>();
            registry.observationConfig().observationHandler(new RecordingHandler(events));

            Observation.createNotStarted("my.op", registry)
                    .observe(() -> events.add("task"));

            assertThat(events).containsExactly(
                    "start", "scopeOpened", "task", "scopeClosed", "stop");
        }

        @Test
        void shouldCallErrorBeforeStop_WhenObserveThrows() {
            List<String> events = new ArrayList<>();
            registry.observationConfig().observationHandler(new RecordingHandler(events));

            assertThatThrownBy(() ->
                    Observation.createNotStarted("my.op", registry)
                            .observe((Runnable) () -> {
                                events.add("task");
                                throw new RuntimeException("boom");
                            })
            ).isInstanceOf(RuntimeException.class);

            // scope closes (try-with-resources), then error, then stop
            assertThat(events).containsExactly(
                    "start", "scopeOpened", "task", "scopeClosed", "error", "stop");
        }

        @Test
        void shouldReturnResult_WhenUsingObserveSupplier() {
            registry.observationConfig().observationHandler(new RecordingHandler(new ArrayList<>()));

            String result = Observation.createNotStarted("my.op", registry)
                    .observe(() -> "hello");

            assertThat(result).isEqualTo("hello");
        }
    }

    // ── Context ──────────────────────────────────────────────────────

    @Nested
    class ContextTests {

        @Test
        void shouldSetNameFromCreation() {
            registry.observationConfig().observationHandler(new RecordingHandler(new ArrayList<>()));
            Observation obs = Observation.createNotStarted("my.op", registry);

            assertThat(obs.getContext().getName()).isEqualTo("my.op");
        }

        @Test
        void shouldStoreLowCardinalityKeyValues() {
            registry.observationConfig().observationHandler(new RecordingHandler(new ArrayList<>()));
            Observation obs = Observation.createNotStarted("my.op", registry)
                    .lowCardinalityKeyValue("method", "GET")
                    .lowCardinalityKeyValue("status", "200");

            assertThat(obs.getContext().getLowCardinalityKeyValues())
                    .containsExactly(
                            KeyValue.of("method", "GET"),
                            KeyValue.of("status", "200"));
        }

        @Test
        void shouldStoreHighCardinalityKeyValues() {
            registry.observationConfig().observationHandler(new RecordingHandler(new ArrayList<>()));
            Observation obs = Observation.createNotStarted("my.op", registry)
                    .highCardinalityKeyValue("url", "/api/users/42")
                    .highCardinalityKeyValue("requestId", "abc-123");

            assertThat(obs.getContext().getHighCardinalityKeyValues())
                    .containsExactly(
                            KeyValue.of("requestId", "abc-123"),
                            KeyValue.of("url", "/api/users/42"));
        }

        @Test
        void shouldReturnAllKeyValuesSorted() {
            registry.observationConfig().observationHandler(new RecordingHandler(new ArrayList<>()));
            Observation obs = Observation.createNotStarted("my.op", registry)
                    .lowCardinalityKeyValue("method", "GET")
                    .highCardinalityKeyValue("url", "/api");

            assertThat(obs.getContext().getAllKeyValues())
                    .containsExactly(
                            KeyValue.of("method", "GET"),
                            KeyValue.of("url", "/api"));
        }

        @Test
        void shouldReplaceDuplicateKeys() {
            registry.observationConfig().observationHandler(new RecordingHandler(new ArrayList<>()));
            Observation obs = Observation.createNotStarted("my.op", registry)
                    .lowCardinalityKeyValue("status", "200")
                    .lowCardinalityKeyValue("status", "500"); // replace

            assertThat(obs.getContext().getLowCardinalityKeyValues())
                    .containsExactly(KeyValue.of("status", "500"));
        }

        @Test
        void shouldStoreAndRetrieveArbitraryData() {
            Observation.Context context = new Observation.Context();
            context.put("myKey", 42);

            Integer value = context.get("myKey");
            assertThat(value).isEqualTo(42);
        }

        @Test
        void shouldThrowOnMissingRequiredData() {
            Observation.Context context = new Observation.Context();

            assertThatThrownBy(() -> context.getRequired("missing"))
                    .isInstanceOf(IllegalStateException.class)
                    .hasMessageContaining("missing");
        }

        @Test
        void shouldUseCustomContext() {
            registry.observationConfig().observationHandler(new RecordingHandler(new ArrayList<>()));

            Observation.Context custom = new Observation.Context();
            custom.setContextualName("custom-context");

            Observation obs = Observation.createNotStarted("my.op", () -> custom, registry);

            assertThat(obs.getContext()).isSameAs(custom);
            assertThat(obs.getContext().getContextualName()).isEqualTo("custom-context");
        }
    }

    // ── Scope tracking ───────────────────────────────────────────────

    @Nested
    class ScopeTracking {

        @Test
        void shouldSetCurrentObservation_WhenScopeOpened() {
            registry.observationConfig().observationHandler(new RecordingHandler(new ArrayList<>()));
            Observation obs = Observation.start("my.op", registry);

            assertThat(registry.getCurrentObservation()).isNull();

            try (Observation.Scope scope = obs.openScope()) {
                assertThat(registry.getCurrentObservation()).isSameAs(obs);
            }

            assertThat(registry.getCurrentObservation()).isNull();
            obs.stop();
        }

        @Test
        void shouldSupportNestedScopes() {
            registry.observationConfig().observationHandler(new RecordingHandler(new ArrayList<>()));
            Observation outer = Observation.start("outer", registry);
            Observation inner = Observation.start("inner", registry);

            try (Observation.Scope outerScope = outer.openScope()) {
                assertThat(registry.getCurrentObservation()).isSameAs(outer);

                try (Observation.Scope innerScope = inner.openScope()) {
                    assertThat(registry.getCurrentObservation()).isSameAs(inner);
                }

                // After inner scope closes, outer is restored
                assertThat(registry.getCurrentObservation()).isSameAs(outer);
            }

            assertThat(registry.getCurrentObservation()).isNull();
            outer.stop();
            inner.stop();
        }

        @Test
        void shouldCallScopeHandlers_InCorrectOrder() {
            List<String> events = new ArrayList<>();
            registry.observationConfig()
                    .observationHandler(new ObservationHandler<>() {
                        @Override
                        public void onScopeOpened(Observation.Context context) {
                            events.add("opened:A");
                        }

                        @Override
                        public void onScopeClosed(Observation.Context context) {
                            events.add("closed:A");
                        }
                    })
                    .observationHandler(new ObservationHandler<>() {
                        @Override
                        public void onScopeOpened(Observation.Context context) {
                            events.add("opened:B");
                        }

                        @Override
                        public void onScopeClosed(Observation.Context context) {
                            events.add("closed:B");
                        }
                    });

            Observation obs = Observation.start("my.op", registry);
            try (Observation.Scope scope = obs.openScope()) {
                // handlers called
            }
            obs.stop();

            // opened in forward order, closed in reverse
            assertThat(events).containsExactly(
                    "opened:A", "opened:B", "closed:B", "closed:A");
        }
    }

    // ── NOOP ─────────────────────────────────────────────────────────

    @Nested
    class NoopBehavior {

        @Test
        void shouldReturnNoop_WhenRegistryIsNoop() {
            Observation obs = Observation.createNotStarted("my.op", ObservationRegistry.NOOP);

            assertThat(obs).isSameAs(Observation.NOOP);
        }

        @Test
        void shouldReturnNoop_WhenNoHandlersRegistered() {
            // Empty registry (no handlers) is considered noop
            ObservationRegistry empty = ObservationRegistry.create();

            Observation obs = Observation.createNotStarted("my.op", empty);

            assertThat(obs).isSameAs(Observation.NOOP);
        }

        @Test
        void shouldReturnNoop_WhenRegistryIsNull() {
            Observation obs = Observation.createNotStarted("my.op", (ObservationRegistry) null);

            assertThat(obs).isSameAs(Observation.NOOP);
        }

        @Test
        void shouldSafelyChainNoopCalls() {
            // NOOP observation should not throw on any operation
            Observation.NOOP
                    .lowCardinalityKeyValue("k", "v")
                    .highCardinalityKeyValue("k2", "v2")
                    .start()
                    .error(new RuntimeException("test"))
                    .stop();
        }

        @Test
        void shouldReturnNoopScope() {
            Observation.Scope scope = Observation.NOOP.openScope();

            assertThat(scope).isSameAs(Observation.Scope.NOOP);
            scope.close(); // should not throw
        }
    }

    // ── Handler filtering ────────────────────────────────────────────

    @Nested
    class HandlerFiltering {

        @Test
        void shouldOnlyDispatchToSupportingHandlers() {
            List<String> events = new ArrayList<>();

            registry.observationConfig()
                    .observationHandler(new ObservationHandler<>() {
                        @Override
                        public void onStart(Observation.Context context) {
                            events.add("handler-A");
                        }

                        @Override
                        public boolean supportsContext(Observation.Context context) {
                            return true; // supports all
                        }
                    })
                    .observationHandler(new ObservationHandler<>() {
                        @Override
                        public void onStart(Observation.Context context) {
                            events.add("handler-B");
                        }

                        @Override
                        public boolean supportsContext(Observation.Context context) {
                            return false; // supports nothing
                        }
                    });

            Observation.start("my.op", registry).stop();

            assertThat(events).containsExactly("handler-A");
        }
    }

    // ── Helper ───────────────────────────────────────────────────────

    /**
     * Records lifecycle events as strings for test assertions.
     */
    private static class RecordingHandler implements ObservationHandler<Observation.Context> {
        private final List<String> events;

        RecordingHandler(List<String> events) {
            this.events = events;
        }

        @Override
        public void onStart(Observation.Context context) {
            events.add("start");
        }

        @Override
        public void onStop(Observation.Context context) {
            events.add("stop");
        }

        @Override
        public void onError(Observation.Context context) {
            events.add("error");
        }

        @Override
        public void onScopeOpened(Observation.Context context) {
            events.add("scopeOpened");
        }

        @Override
        public void onScopeClosed(Observation.Context context) {
            events.add("scopeClosed");
        }
    }
}
```

#### File: `iris-micrometer/src/test/java/com/iris/micrometer/core/observation/DefaultMeterObservationHandlerTest.java` [NEW]

```java
package com.iris.micrometer.core.observation;

import com.iris.micrometer.core.Counter;
import com.iris.micrometer.core.Meter;
import com.iris.micrometer.core.Timer;
import com.iris.micrometer.core.simple.SimpleMeterRegistry;
import com.iris.micrometer.observation.Observation;
import com.iris.micrometer.observation.ObservationRegistry;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.util.concurrent.TimeUnit;

import static org.assertj.core.api.Assertions.assertThat;

class DefaultMeterObservationHandlerTest {

    private SimpleMeterRegistry meterRegistry;
    private ObservationRegistry observationRegistry;

    @BeforeEach
    void setUp() {
        meterRegistry = new SimpleMeterRegistry();
        observationRegistry = ObservationRegistry.create();
        observationRegistry.observationConfig()
                .observationHandler(new DefaultMeterObservationHandler(meterRegistry));
    }

    @Test
    void shouldRecordTimerOnStop_WhenObservationCompletes() {
        Observation obs = Observation.start("http.request", observationRegistry);
        // Simulate some work
        obs.stop();

        Timer timer = meterRegistry.timer("http.request", "error", "none");
        assertThat(timer.count()).isEqualTo(1);
        assertThat(timer.totalTime(TimeUnit.NANOSECONDS)).isGreaterThan(0);
    }

    @Test
    void shouldIncludeErrorTag_WhenErrorOccurred() {
        Observation obs = Observation.start("http.request", observationRegistry);
        obs.error(new IllegalArgumentException("bad input"));
        obs.stop();

        // Timer should have error tag with exception class name
        Timer timer = meterRegistry.timer("http.request",
                "error", "IllegalArgumentException");
        assertThat(timer.count()).isEqualTo(1);
    }

    @Test
    void shouldIncludeNoneErrorTag_WhenNoError() {
        Observation obs = Observation.start("http.request", observationRegistry);
        obs.stop();

        Timer timer = meterRegistry.timer("http.request", "error", "none");
        assertThat(timer.count()).isEqualTo(1);
    }

    @Test
    void shouldIncludeLowCardinalityTags_OnTimer() {
        Observation obs = Observation.createNotStarted("http.request", observationRegistry)
                .lowCardinalityKeyValue("method", "GET")
                .lowCardinalityKeyValue("status", "200");
        obs.start();
        obs.stop();

        Timer timer = meterRegistry.timer("http.request",
                "method", "GET", "status", "200", "error", "none");
        assertThat(timer.count()).isEqualTo(1);
    }

    @Test
    void shouldIncrementErrorCounter_WhenErrorSignaled() {
        Observation obs = Observation.start("http.request", observationRegistry);
        obs.lowCardinalityKeyValue("method", "POST");
        obs.error(new RuntimeException("connection refused"));
        obs.stop();

        Counter errorCounter = meterRegistry.counter("http.request.error",
                "method", "POST", "exception", "RuntimeException");
        assertThat(errorCounter.count()).isEqualTo(1);
    }

    @Test
    void shouldRecordMultipleObservations_AsSeparateTimerRecordings() {
        for (int i = 0; i < 3; i++) {
            Observation obs = Observation.start("db.query", observationRegistry);
            obs.stop();
        }

        Timer timer = meterRegistry.timer("db.query", "error", "none");
        assertThat(timer.count()).isEqualTo(3);
    }

    @Test
    void shouldCreateDistinctTimers_ForDifferentTags() {
        // Success
        Observation.createNotStarted("http.request", observationRegistry)
                .lowCardinalityKeyValue("method", "GET")
                .start().stop();

        // Different method
        Observation.createNotStarted("http.request", observationRegistry)
                .lowCardinalityKeyValue("method", "POST")
                .start().stop();

        Timer getTimer = meterRegistry.timer("http.request",
                "method", "GET", "error", "none");
        Timer postTimer = meterRegistry.timer("http.request",
                "method", "POST", "error", "none");

        assertThat(getTimer.count()).isEqualTo(1);
        assertThat(postTimer.count()).isEqualTo(1);
    }

    @Test
    void shouldNotCreateMetrics_WhenRegistryIsNoop() {
        ObservationRegistry noopRegistry = ObservationRegistry.NOOP;
        // NOOP observation — no handlers, no metrics
        Observation.createNotStarted("http.request", noopRegistry)
                .start().stop();

        assertThat(meterRegistry.getMeters()).isEmpty();
    }

    @Test
    void shouldWorkWithObserveConvenience() {
        Observation.createNotStarted("my.task", observationRegistry)
                .lowCardinalityKeyValue("type", "batch")
                .observe(() -> {
                    // simulate work
                    try { Thread.sleep(10); } catch (InterruptedException e) { /* ignore */ }
                });

        Timer timer = meterRegistry.timer("my.task",
                "type", "batch", "error", "none");
        assertThat(timer.count()).isEqualTo(1);
        assertThat(timer.totalTime(TimeUnit.MILLISECONDS)).isGreaterThanOrEqualTo(5);
    }
}
```

#### File: `iris-micrometer/src/test/java/com/iris/micrometer/observation/integration/ObservationMetricsIntegrationTest.java` [NEW]

```java
package com.iris.micrometer.observation.integration;

import com.iris.micrometer.core.Counter;
import com.iris.micrometer.core.Timer;
import com.iris.micrometer.core.observation.DefaultMeterObservationHandler;
import com.iris.micrometer.core.simple.SimpleMeterRegistry;
import com.iris.micrometer.observation.Observation;
import com.iris.micrometer.observation.ObservationHandler;
import com.iris.micrometer.observation.ObservationRegistry;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.util.ArrayList;
import java.util.List;
import java.util.concurrent.TimeUnit;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

/**
 * Integration tests verifying the full observation-to-metrics pipeline:
 * {@code Observation → ObservationHandler → MeterRegistry → Timer/Counter}.
 */
class ObservationMetricsIntegrationTest {

    private SimpleMeterRegistry meterRegistry;
    private ObservationRegistry observationRegistry;

    @BeforeEach
    void setUp() {
        meterRegistry = new SimpleMeterRegistry();
        observationRegistry = ObservationRegistry.create();
        observationRegistry.observationConfig()
                .observationHandler(new DefaultMeterObservationHandler(meterRegistry));
    }

    @Test
    void shouldRecordSuccessfulHttpRequest() {
        // Simulate an HTTP request observation
        Observation obs = Observation.createNotStarted("http.server.requests", observationRegistry)
                .lowCardinalityKeyValue("method", "GET")
                .lowCardinalityKeyValue("uri", "/api/users")
                .lowCardinalityKeyValue("status", "200");

        obs.start();
        // simulate request processing
        try { Thread.sleep(10); } catch (InterruptedException e) { /* ignore */ }
        obs.stop();

        Timer timer = meterRegistry.timer("http.server.requests",
                "method", "GET", "status", "200", "uri", "/api/users", "error", "none");
        assertThat(timer.count()).isEqualTo(1);
        assertThat(timer.totalTime(TimeUnit.MILLISECONDS)).isGreaterThanOrEqualTo(5);
    }

    @Test
    void shouldRecordFailedHttpRequest_WithErrorMetrics() {
        Observation obs = Observation.createNotStarted("http.server.requests", observationRegistry)
                .lowCardinalityKeyValue("method", "POST")
                .lowCardinalityKeyValue("uri", "/api/orders");

        obs.start();
        obs.error(new IllegalStateException("out of stock"));
        obs.lowCardinalityKeyValue("status", "500");
        obs.stop();

        // Timer should include the error tag
        Timer timer = meterRegistry.timer("http.server.requests",
                "method", "POST", "status", "500", "uri", "/api/orders",
                "error", "IllegalStateException");
        assertThat(timer.count()).isEqualTo(1);

        // Error counter should be incremented
        Counter errorCounter = meterRegistry.counter("http.server.requests.error",
                "method", "POST", "uri", "/api/orders",
                "exception", "IllegalStateException");
        assertThat(errorCounter.count()).isEqualTo(1);
    }

    @Test
    void shouldSupportMultipleHandlers() {
        // Add a second handler alongside the metrics handler
        List<String> logEntries = new ArrayList<>();
        observationRegistry.observationConfig()
                .observationHandler(new ObservationHandler<>() {
                    @Override
                    public void onStart(Observation.Context context) {
                        logEntries.add("START: " + context.getName());
                    }

                    @Override
                    public void onStop(Observation.Context context) {
                        logEntries.add("STOP: " + context.getName());
                    }
                });

        Observation.createNotStarted("db.query", observationRegistry)
                .observe(() -> {
                    // simulate query
                });

        // Metrics handler should have recorded
        Timer timer = meterRegistry.timer("db.query", "error", "none");
        assertThat(timer.count()).isEqualTo(1);

        // Logging handler should also have been called
        assertThat(logEntries).containsExactly("START: db.query", "STOP: db.query");
    }

    @Test
    void shouldTrackNestedObservations() {
        Observation parent = Observation.start("http.request", observationRegistry);

        try (Observation.Scope parentScope = parent.openScope()) {
            // The parent is now the current observation
            assertThat(observationRegistry.getCurrentObservation()).isSameAs(parent);

            // Start a nested observation (e.g., a database call within the HTTP request)
            Observation child = Observation.start("db.query", observationRegistry);

            try (Observation.Scope childScope = child.openScope()) {
                // The child is now current
                assertThat(observationRegistry.getCurrentObservation()).isSameAs(child);
            }

            // After child scope closes, parent is restored
            assertThat(observationRegistry.getCurrentObservation()).isSameAs(parent);
            child.stop();
        }

        parent.stop();

        // Both observations should have recorded timers
        Timer httpTimer = meterRegistry.timer("http.request", "error", "none");
        Timer dbTimer = meterRegistry.timer("db.query", "error", "none");
        assertThat(httpTimer.count()).isEqualTo(1);
        assertThat(dbTimer.count()).isEqualTo(1);
    }

    @Test
    void shouldHandleObserveWithError_RecordingBothTimerAndCounter() {
        assertThatThrownBy(() ->
                Observation.createNotStarted("my.operation", observationRegistry)
                        .lowCardinalityKeyValue("type", "critical")
                        .observe((Runnable) () -> {
                            throw new RuntimeException("connection lost");
                        })
        ).isInstanceOf(RuntimeException.class).hasMessage("connection lost");

        // Timer should be recorded with error tag
        Timer timer = meterRegistry.timer("my.operation",
                "type", "critical", "error", "RuntimeException");
        assertThat(timer.count()).isEqualTo(1);

        // Error counter should be incremented
        Counter errorCounter = meterRegistry.counter("my.operation.error",
                "type", "critical", "exception", "RuntimeException");
        assertThat(errorCounter.count()).isEqualTo(1);
    }

    @Test
    void shouldReturnSupplierResult_WithMetricsRecorded() {
        String result = Observation.createNotStarted("compute", observationRegistry)
                .lowCardinalityKeyValue("algo", "fibonacci")
                .observe(() -> "result-42");

        assertThat(result).isEqualTo("result-42");

        Timer timer = meterRegistry.timer("compute",
                "algo", "fibonacci", "error", "none");
        assertThat(timer.count()).isEqualTo(1);
    }

    @Test
    void shouldAccumulateMetrics_AcrossMultipleObservations() {
        for (int i = 0; i < 5; i++) {
            Observation.createNotStarted("batch.job", observationRegistry)
                    .lowCardinalityKeyValue("job", "etl")
                    .observe(() -> {
                        // work
                    });
        }

        Timer timer = meterRegistry.timer("batch.job",
                "job", "etl", "error", "none");
        assertThat(timer.count()).isEqualTo(5);
    }

    @Test
    void shouldUseHighCardinalityForTracing_NotForMetrics() {
        Observation.createNotStarted("http.request", observationRegistry)
                .lowCardinalityKeyValue("method", "GET")     // → metric tag
                .highCardinalityKeyValue("requestId", "abc") // → NOT a metric tag
                .observe(() -> {});

        // The timer should only have the low-cardinality "method" tag + error tag
        Timer timer = meterRegistry.timer("http.request",
                "method", "GET", "error", "none");
        assertThat(timer.count()).isEqualTo(1);

        // No timer should exist with requestId as a tag
        assertThat(meterRegistry.getMeters().stream()
                .filter(m -> m.getId().getTags().stream()
                        .anyMatch(t -> t.getKey().equals("requestId")))
                .count()).isZero();
    }
}
```

---

## Summary

| Concept | What It Means |
|---------|---------------|
| **Observation** | A lifecycle abstraction (`start` → `openScope` → `stop`) that unifies metrics, tracing, and logging instrumentation |
| **Observation.Context** | A mutable bag holding the observation's name, error, low/high-cardinality key-value pairs, and arbitrary handler data |
| **Observation.Scope** | An `AutoCloseable` that tracks the current observation per thread via a `ThreadLocal`, forming a linked list for nesting |
| **ObservationHandler** | A callback interface (`onStart`, `onStop`, `onError`, `onScopeOpened`, `onScopeClosed`) that pluggable concerns implement |
| **ObservationRegistry** | Holds registered handlers and tracks the current scope per thread; returns `NOOP` observations when no handlers are registered |
| **DefaultMeterObservationHandler** | The bridge between observations and metrics — starts a `Timer.Sample` on `onStart`, records duration on `onStop`, increments a Counter on `onError` |
| **Low vs High cardinality** | Low-cardinality key values become metric tags (bounded set); high-cardinality become span attributes only (unbounded) |
| **Instrument once, observe many** | The core pattern: a single `observe()` call drives all observability concerns through pluggable handlers |

**Next: Chapter 22 — SSL Bundles** — Named, reusable SSL configurations that wrap keystores, trust stores, and `SSLContext` creation into a single abstraction any component can consume, including the embedded Tomcat from Chapter 9.
