# Chapter 6: Baggage

> Line references based on commit `dceab1b9` of the Micrometer Tracing repository.

## Build Challenge

| Current State | Limitation | Objective |
|---------------|------------|-----------|
| SimpleTracer creates spans with IDs and scope management, but all BaggageManager methods return NOOP — baggage entries are silently ignored | No way to attach custom metadata (user IDs, request IDs, tenant IDs) to a trace and have it propagate across service boundaries alongside the trace context | Build `SimpleBaggageInScope` (implementing both `Baggage` and `BaggageInScope`), `SimpleBaggageManager`, and wire them into `SimpleTracer` so baggage entries can be created, scoped, retrieved, and collected — with full lifecycle management |

---

## 6.1 The Integration Point

The integration point is **SimpleTracer's BaggageManager stubs** — lines 161-186 in the previous version, where every baggage method returned `Baggage.NOOP` or `Collections.emptyMap()`. These stubs are the seam where baggage plugs into the existing tracer.

This connects **Baggage management ↔ Tracer lifecycle**. To make it work, we need to build:
- `SimpleBaggageInScope` — a single class implementing both `Baggage` (mutable) and `BaggageInScope` (scoped lifecycle)
- `SimpleBaggageManager` — stores and retrieves baggage entries per trace context
- Wire `SimpleTracer` to delegate all `BaggageManager` methods to `SimpleBaggageManager`

Two key decisions here:

1. **Why one class for both `Baggage` and `BaggageInScope`?** Because baggage has two phases — you *configure* it (`set(value)`) then *activate* it (`makeCurrent()`). Having one object serve both roles means `makeCurrent()` just flips an `inScope` flag and returns `this` — no wrapper objects, no state copying. The `get()` method gates on this flag: value is only visible when in scope.

2. **Why store baggage per `TraceContext` in a `ConcurrentHashMap`?** Because different traces running concurrently on the same JVM should have independent baggage. A global flat map would leak baggage between unrelated traces. Keying by `TraceContext` isolates each trace's baggage.

**Modifying:** `src/main/java/dev/linhvu/tracing/simple/SimpleTracer.java`
**Change:** Add `SimpleBaggageManager` field and replace NOOP stubs with delegation

```java
// New field (after spanReporters)
final SimpleBaggageManager simpleBaggageManager;

// In constructor — initialize the baggage manager
this.simpleBaggageManager = new SimpleBaggageManager(this);

// Replace NOOP stubs with delegation:
@Override
public Map<String, String> getAllBaggage() {
    return simpleBaggageManager.getAllBaggage();
}

@Override
public Baggage getBaggage(String name) {
    return simpleBaggageManager.getBaggage(name);
}

@Override
public Baggage createBaggage(String name) {
    return simpleBaggageManager.createBaggage(name);
}
// ... (all BaggageManager methods delegate similarly)
```

## 6.2 SimpleBaggageInScope — One Object, Two Interfaces

The core insight: `SimpleBaggageInScope` implements **both** `Baggage` (mutable configuration) and `BaggageInScope` (scoped lifecycle). The lifecycle is:

```
create → set(value) → makeCurrent() → [value visible via get()] → close() → [value invisible]
```

**New file:** `src/main/java/dev/linhvu/tracing/simple/SimpleBaggageInScope.java`

```java
package dev.linhvu.tracing.simple;

import dev.linhvu.tracing.Baggage;
import dev.linhvu.tracing.BaggageInScope;
import dev.linhvu.tracing.CurrentTraceContext;
import dev.linhvu.tracing.TraceContext;

import java.util.Objects;

public class SimpleBaggageInScope implements Baggage, BaggageInScope {

    private final String name;

    private volatile TraceContext traceContext;

    private volatile String value;

    private volatile boolean inScope = false;

    private final CurrentTraceContext currentTraceContext;

    public SimpleBaggageInScope(CurrentTraceContext currentTraceContext, String name) {
        this.currentTraceContext = currentTraceContext;
        this.name = name;
        this.traceContext = currentTraceContext.context();
    }

    public SimpleBaggageInScope(CurrentTraceContext currentTraceContext, String name,
            TraceContext traceContext) {
        this.currentTraceContext = currentTraceContext;
        this.name = name;
        this.traceContext = traceContext;
    }

    @Override
    public String name() {
        return name;
    }

    @Override
    public String get() {
        if (currentTraceContext.context() == null) {
            return null;
        }
        return get(currentTraceContext.context());
    }

    @Override
    public String get(TraceContext traceContext) {
        if (!inScope) {
            return null;
        }
        if (this.traceContext != null && traceContext != null
                && this.traceContext == traceContext) {
            return value;
        }
        return null;
    }

    @Override
    public Baggage set(String value) {
        this.value = value;
        this.traceContext = currentTraceContext.context();
        return this;
    }

    @Override
    public Baggage set(TraceContext traceContext, String value) {
        this.value = value;
        this.traceContext = traceContext;
        return this;
    }

    @Override
    public BaggageInScope makeCurrent() {
        this.inScope = true;
        return this;
    }

    @Override
    public void close() {
        this.inScope = false;
    }

    // Package-private accessors for SimpleBaggageManager
    boolean isInScope() { return inScope; }
    String getValue() { return value; }
    TraceContext getTraceContext() { return traceContext; }

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (o == null || getClass() != o.getClass()) return false;
        SimpleBaggageInScope that = (SimpleBaggageInScope) o;
        return Objects.equals(name, that.name);
    }

    @Override
    public int hashCode() {
        return Objects.hash(name);
    }
}
```

The `get()` method is the gatekeeper — it checks two conditions:
1. **`inScope` must be `true`** — if you closed the scope, the value disappears
2. **`traceContext` must match** (reference equality `==`) — prevents cross-trace leakage

## 6.3 SimpleBaggageManager — The Registry

**New file:** `src/main/java/dev/linhvu/tracing/simple/SimpleBaggageManager.java`

```java
package dev.linhvu.tracing.simple;

import dev.linhvu.tracing.Baggage;
import dev.linhvu.tracing.BaggageInScope;
import dev.linhvu.tracing.BaggageManager;
import dev.linhvu.tracing.TraceContext;

import java.util.ArrayList;
import java.util.Collections;
import java.util.LinkedHashMap;
import java.util.List;
import java.util.Map;
import java.util.Set;
import java.util.concurrent.ConcurrentHashMap;

public class SimpleBaggageManager implements BaggageManager {

    private final Map<TraceContext, Set<SimpleBaggageInScope>> baggagesByContext =
            new ConcurrentHashMap<>();

    private final SimpleTracer tracer;

    private final List<String> remoteFields;

    SimpleBaggageManager(SimpleTracer tracer) {
        this(tracer, List.of());
    }

    SimpleBaggageManager(SimpleTracer tracer, List<String> remoteFields) {
        this.tracer = tracer;
        this.remoteFields = new ArrayList<>(remoteFields);
    }

    @Override
    public Map<String, String> getAllBaggage() {
        TraceContext current = tracer.currentTraceContext().context();
        if (current == null) {
            return Collections.emptyMap();
        }
        return getAllBaggageForCtx(current);
    }

    Map<String, String> getAllBaggageForCtx(TraceContext traceContext) {
        Set<SimpleBaggageInScope> entries = baggagesByContext.get(traceContext);
        if (entries == null || entries.isEmpty()) {
            return Collections.emptyMap();
        }
        Map<String, String> result = new LinkedHashMap<>();
        for (SimpleBaggageInScope entry : entries) {
            if (entry.isInScope()) {
                String val = entry.getValue();
                if (val != null) {
                    result.put(entry.name(), val);
                }
            }
        }
        return result;
    }

    @Override
    public Baggage getBaggage(String name) {
        TraceContext current = tracer.currentTraceContext().context();
        if (current == null) {
            return Baggage.NOOP;
        }
        Baggage found = getBaggage(current, name);
        return found != null ? found : Baggage.NOOP;
    }

    @Override
    public Baggage getBaggage(TraceContext traceContext, String name) {
        return baggageForName(traceContext, name);
    }

    @Override
    public Baggage createBaggage(String name) {
        return createSimpleBaggage(name);
    }

    @Override
    public Baggage createBaggage(String name, String value) {
        SimpleBaggageInScope baggage = createSimpleBaggage(name);
        baggage.set(value);
        return baggage;
    }

    private SimpleBaggageInScope createSimpleBaggage(String name) {
        TraceContext current = tracer.currentTraceContext().context();
        SimpleBaggageInScope existing = baggageForName(current, name);
        if (existing != null) {
            return existing;
        }
        SimpleBaggageInScope baggage = new SimpleBaggageInScope(
                tracer.currentTraceContext(), name, current);
        if (current != null) {
            baggagesByContext.computeIfAbsent(current, k -> ConcurrentHashMap.newKeySet())
                    .add(baggage);
        }
        return baggage;
    }

    private SimpleBaggageInScope baggageForName(TraceContext traceContext, String name) {
        if (traceContext == null || name == null) {
            return null;
        }
        Set<SimpleBaggageInScope> entries = baggagesByContext.get(traceContext);
        if (entries == null) {
            return null;
        }
        return entries.stream()
                .filter(b -> b.name().equalsIgnoreCase(name))
                .findFirst()
                .orElse(null);
    }
}
```

Key design choices:
- **`ConcurrentHashMap<TraceContext, Set<SimpleBaggageInScope>>`** — each trace context has its own set of baggage entries, isolated from other traces
- **Case-insensitive lookup** — `equalsIgnoreCase(name)` matches HTTP header convention (baggage often propagates as HTTP headers)
- **`createSimpleBaggage` deduplicates** — if baggage with the same name already exists for this context, return it instead of creating a duplicate

## 6.4 Try It Yourself

<details>
<summary>Challenge: Implement the get() method with scope and context checks</summary>

The `get()` method must return the baggage value only when two conditions are met. Implement it:

```java
@Override
public String get() {
    if (currentTraceContext.context() == null) {
        return null;
    }
    return get(currentTraceContext.context());
}

@Override
public String get(TraceContext traceContext) {
    if (!inScope) {
        return null;
    }
    // Only return value if the context matches — prevents cross-trace leakage
    if (this.traceContext != null && traceContext != null
            && this.traceContext == traceContext) {
        return value;
    }
    return null;
}
```

</details>

<details>
<summary>Challenge: Implement getAllBaggageForCtx — collect all in-scope entries into a Map</summary>

Given a `Set<SimpleBaggageInScope>` for a trace context, collect all in-scope entries with non-null values into a `Map<String, String>`:

```java
Map<String, String> getAllBaggageForCtx(TraceContext traceContext) {
    Set<SimpleBaggageInScope> entries = baggagesByContext.get(traceContext);
    if (entries == null || entries.isEmpty()) {
        return Collections.emptyMap();
    }
    Map<String, String> result = new LinkedHashMap<>();
    for (SimpleBaggageInScope entry : entries) {
        if (entry.isInScope()) {
            String val = entry.getValue();
            if (val != null) {
                result.put(entry.name(), val);
            }
        }
    }
    return result;
}
```

</details>

<details>
<summary>Challenge: Implement createSimpleBaggage — deduplicate by name, register with context</summary>

Create a new baggage entry or return an existing one with the same name. Register it with the current trace context:

```java
private SimpleBaggageInScope createSimpleBaggage(String name) {
    TraceContext current = tracer.currentTraceContext().context();
    SimpleBaggageInScope existing = baggageForName(current, name);
    if (existing != null) {
        return existing;
    }
    SimpleBaggageInScope baggage = new SimpleBaggageInScope(
            tracer.currentTraceContext(), name, current);
    if (current != null) {
        baggagesByContext.computeIfAbsent(current, k -> ConcurrentHashMap.newKeySet())
                .add(baggage);
    }
    return baggage;
}
```

</details>

## 6.5 Tests

### Unit Tests

**New file:** `src/test/java/dev/linhvu/tracing/simple/SimpleBaggageInScopeTest.java`

```java
class SimpleBaggageInScopeTest {

    @Test
    void shouldReturnName_WhenCreated() { ... }

    @Test
    void shouldReturnNullFromGet_WhenNotInScope() { ... }

    @Test
    void shouldReturnValue_WhenInScope() { ... }

    @Test
    void shouldReturnNullFromGet_WhenClosedAfterMakeCurrent() { ... }

    @Test
    void shouldReturnValueFromGetWithContext_WhenContextMatches() { ... }

    @Test
    void shouldReturnNullFromGetWithContext_WhenContextDoesNotMatch() { ... }

    @Test
    void shouldUpdateValue_WhenSetCalledMultipleTimes() { ... }

    @Test
    void shouldReturnSelfFromSet() { ... }

    @Test
    void shouldReturnSelfFromMakeCurrent() { ... }

    @Test
    void shouldSupportMakeCurrentWithValue_AsConvenienceMethod() { ... }

    @Test
    void shouldSupportMakeCurrentWithContextAndValue() { ... }

    @Test
    void shouldBeEqualByName() { ... }

    @Test
    void shouldNotBeEqual_WhenDifferentNames() { ... }

    @Test
    void shouldReturnNullFromGet_WhenNoCurrentContext() { ... }
}
```

**New file:** `src/test/java/dev/linhvu/tracing/simple/SimpleBaggageManagerTest.java`

```java
class SimpleBaggageManagerTest {

    // createBaggage
    @Test
    void shouldCreateBaggage_WhenInScope() { ... }

    @Test
    void shouldCreateBaggageWithValue() { ... }

    @Test
    void shouldReturnExistingBaggage_WhenCreatedTwiceWithSameName() { ... }

    // createBaggageInScope
    @Test
    void shouldCreateBaggageInScope_WhenUsingConvenienceMethod() { ... }

    @Test
    void shouldCreateBaggageInScopeWithContext() { ... }

    // getBaggage
    @Test
    void shouldReturnNoopBaggage_WhenNoSpanInScope() { ... }

    @Test
    void shouldReturnBaggageByName_WhenCreatedAndInScope() { ... }

    @Test
    void shouldReturnBaggageByNameCaseInsensitive() { ... }

    @Test
    void shouldReturnBaggageByNameAndContext() { ... }

    @Test
    void shouldReturnNull_WhenBaggageNotFoundForContext() { ... }

    // getAllBaggage
    @Test
    void shouldReturnEmptyMap_WhenNoSpanInScope() { ... }

    @Test
    void shouldReturnAllInScopeBaggage() { ... }

    @Test
    void shouldNotIncludeClosedBaggage_WhenGettingAll() { ... }

    @Test
    void shouldReturnAllBaggageForSpecificContext() { ... }

    @Test
    void shouldDelegateToGetAllBaggage_WhenContextIsNull() { ... }

    @Test
    void shouldReturnEmptyBaggageFields_WhenDefault() { ... }
}
```

### Integration Tests

**New file:** `src/test/java/dev/linhvu/tracing/integration/BaggageIntegrationTest.java`

```java
class BaggageIntegrationTest {

    @Test
    void shouldManageBaggageLifecycle_WhenUsingTryWithResources() { ... }

    @Test
    void shouldSupportMultipleBaggageEntries_WhenInSameScope() { ... }

    @Test
    void shouldUpdateBaggageValue_WhenSetCalledAgain() { ... }

    @Test
    void shouldRetrieveBaggageByName_WhenUsingGetBaggage() { ... }

    @Test
    void shouldWorkWithNestedSpanScopes() { ... }

    @Test
    void shouldSupportBaggageWithExplicitTraceContext() { ... }

    @Test
    void shouldHandleClosingBaggageGracefully_WhenClosedMultipleTimes() { ... }

    @Test
    void shouldReturnNoopBaggage_WhenNoSpanInScope() { ... }

    @Test
    void shouldReturnEmptyBaggage_WhenNoSpanInScope() { ... }

    @Test
    void shouldSupportBaggageAcrossSpanPattern_WhenCreatedAndUsedInDifferentScopes() { ... }
}
```

**Run:** `./gradlew test` — expected: all tests pass (including all prior features' tests)

---

## 6.6 Why This Works

> ★ **Insight** -------------------------------------------
> - **Why does `get()` use reference equality (`==`) for context matching instead of `equals()`?** Because `TraceContext` instances in the simple tracer are mutable (`SimpleTraceContext` has setters). Two contexts could be `.equals()` at one moment but differ later. Reference equality ensures that `get()` only returns the value for the *exact* context this baggage was associated with — not a copy or lookalike. The real framework (`SimpleBaggageInScope.java:93-101`) uses the same reference check.
> - **Why is `inScope` a `volatile boolean` instead of using `AtomicBoolean`?** Because we only need visibility guarantees (one thread writes, another reads), not compare-and-swap atomicity. `volatile` is sufficient and lighter. The real framework uses the same approach.
> -----------------------------------------------------------

> ★ **Insight** -------------------------------------------
> - **Baggage vs. Span Tags — when to use which?** Span tags are *local* to a single span and don't propagate. Baggage *propagates* across service boundaries via HTTP headers (or message headers). Use span tags for span-specific metadata (like `http.status_code`). Use baggage for cross-cutting concerns (like `user-id` or `tenant-id`) that should be visible to every service in the trace. The trade-off: baggage adds overhead to every outgoing request (it's injected into headers), so keep it small.
> - **The `BaggageManager` is part of `Tracer` for a reason.** Making `Tracer extends BaggageManager` means any code that has a `Tracer` can create and manage baggage without needing a separate dependency. This is the **Facade pattern** — the tracer is the single entry point for all tracing concerns (spans, scope, and baggage).
> -----------------------------------------------------------

## 6.7 What We Enhanced

| Aspect | Before (ch01-05) | Current (ch06) | Real Framework |
|--------|-------------------|----------------|----------------|
| BaggageManager methods | All returned `Baggage.NOOP` / `Collections.emptyMap()` — baggage silently ignored | `SimpleTracer` delegates to `SimpleBaggageManager` with real `ConcurrentHashMap`-backed storage per `TraceContext` | Same delegation pattern — `SimpleTracer` creates `SimpleBaggageManager(this)` and delegates all baggage methods to it (`SimpleTracer.java:148-191`) |
| Baggage implementation | Only stub interfaces with NOOP singletons | `SimpleBaggageInScope` implements both `Baggage` and `BaggageInScope` in one class — `inScope` flag gates `get()` visibility | Same single-class design (`SimpleBaggageInScope.java:31`). Real version also tracks `closed` flag and has `CurrentTraceContext` reference |
| Baggage storage | None | `ConcurrentHashMap<TraceContext, Set<SimpleBaggageInScope>>` keyed by context for trace isolation | `ConcurrentHashMap<TraceContext, ThreadLocal<Set<SimpleBaggageInScope>>>` — adds per-thread isolation on top of per-context, for concurrent request handling |
| Baggage lookup | N/A | Case-insensitive name matching via `equalsIgnoreCase()` | Same case-insensitive lookup (`SimpleBaggageManager.java:100-110`) |

## 6.8 Connection to Real Framework

| Simplified Code | Real Framework Code | File:Line | Key Difference |
|----------------|--------------------|-----------| ---------------|
| `SimpleBaggageInScope` | `SimpleBaggageInScope` | `SimpleBaggageInScope.java:31` | Real version adds `closed` boolean flag (separate from `inScope`) and `CurrentTraceContext` initialization to `CurrentTraceContext.NOOP` |
| `SimpleBaggageInScope.get(TraceContext)` | `SimpleBaggageInScope.get(TraceContext)` | `SimpleBaggageInScope.java:93-101` | Same reference equality check for context matching |
| `SimpleBaggageInScope.makeCurrent()` | `SimpleBaggageInScope.makeCurrent()` | `SimpleBaggageInScope.java:133-136` | Identical — sets `inScope = true`, returns `this` |
| `SimpleBaggageInScope.close()` | `SimpleBaggageInScope.close()` | `SimpleBaggageInScope.java:139-142` | Real also sets `closed = true` alongside `inScope = false` |
| `SimpleBaggageManager` | `SimpleBaggageManager` | `SimpleBaggageManager.java:33` | Real uses `Map<TraceContext, ThreadLocal<Set<>>>` for per-thread isolation; simplified uses `Map<TraceContext, Set<>>` |
| `SimpleBaggageManager.getAllBaggageForCtx` | `SimpleBaggageManager.getAllBaggageForCtx` | `SimpleBaggageManager.java:69-78` | Real also merges `baggageFromParent()` stored on `SimpleTraceContext` — parent baggage propagation to child spans |
| `SimpleTracer` delegation | `SimpleTracer` delegation | `SimpleTracer.java:148-191` | Real also propagates parent baggage during child span creation and overrides `getAllBaggage(TraceContext)` |

## 6.9 Complete Code

### Production Code

#### File: `src/main/java/dev/linhvu/tracing/simple/SimpleBaggageInScope.java` [NEW]

```java
package dev.linhvu.tracing.simple;

import dev.linhvu.tracing.Baggage;
import dev.linhvu.tracing.BaggageInScope;
import dev.linhvu.tracing.CurrentTraceContext;
import dev.linhvu.tracing.TraceContext;

import java.util.Objects;

/**
 * A single class that implements both {@link Baggage} (mutable configuration) and
 * {@link BaggageInScope} (scoped lifecycle). This mirrors the real framework's design
 * where one object serves both roles.
 *
 * <p>Lifecycle:
 * <ol>
 *   <li>{@link #set(String)} — stores the value</li>
 *   <li>{@link #makeCurrent()} — places this entry in scope ({@code inScope = true})</li>
 *   <li>{@link #get()} — returns value only when in scope</li>
 *   <li>{@link #close()} — removes from scope ({@code inScope = false})</li>
 * </ol>
 *
 * <p>The {@link #get(TraceContext)} method adds a context check: it only returns the
 * value when the supplied context matches the one this baggage was associated with.
 * This prevents baggage from leaking across independent traces.
 */
public class SimpleBaggageInScope implements Baggage, BaggageInScope {

    private final String name;

    private volatile TraceContext traceContext;

    private volatile String value;

    private volatile boolean inScope = false;

    private final CurrentTraceContext currentTraceContext;

    /**
     * Creates a baggage entry associated with the current trace context.
     */
    public SimpleBaggageInScope(CurrentTraceContext currentTraceContext, String name) {
        this.currentTraceContext = currentTraceContext;
        this.name = name;
        this.traceContext = currentTraceContext.context();
    }

    /**
     * Creates a baggage entry associated with a specific trace context.
     */
    public SimpleBaggageInScope(CurrentTraceContext currentTraceContext, String name,
            TraceContext traceContext) {
        this.currentTraceContext = currentTraceContext;
        this.name = name;
        this.traceContext = traceContext;
    }

    @Override
    public String name() {
        return name;
    }

    @Override
    public String get() {
        if (currentTraceContext.context() == null) {
            return null;
        }
        return get(currentTraceContext.context());
    }

    @Override
    public String get(TraceContext traceContext) {
        if (!inScope) {
            return null;
        }
        // Only return value if the context matches — prevents cross-trace leakage
        if (this.traceContext != null && traceContext != null
                && this.traceContext == traceContext) {
            return value;
        }
        return null;
    }

    @Override
    public Baggage set(String value) {
        this.value = value;
        this.traceContext = currentTraceContext.context();
        return this;
    }

    @Override
    public Baggage set(TraceContext traceContext, String value) {
        this.value = value;
        this.traceContext = traceContext;
        return this;
    }

    @Override
    public BaggageInScope makeCurrent() {
        this.inScope = true;
        return this;
    }

    @Override
    public void close() {
        this.inScope = false;
    }

    // --- Accessors for SimpleBaggageManager ---

    boolean isInScope() {
        return inScope;
    }

    String getValue() {
        return value;
    }

    TraceContext getTraceContext() {
        return traceContext;
    }

    @Override
    public boolean equals(Object o) {
        if (this == o) {
            return true;
        }
        if (o == null || getClass() != o.getClass()) {
            return false;
        }
        SimpleBaggageInScope that = (SimpleBaggageInScope) o;
        return Objects.equals(name, that.name);
    }

    @Override
    public int hashCode() {
        return Objects.hash(name);
    }

    @Override
    public String toString() {
        return "SimpleBaggageInScope{name='" + name + "', value='" + value
                + "', inScope=" + inScope + "}";
    }

}
```

#### File: `src/main/java/dev/linhvu/tracing/simple/SimpleBaggageManager.java` [NEW]

```java
package dev.linhvu.tracing.simple;

import dev.linhvu.tracing.Baggage;
import dev.linhvu.tracing.BaggageInScope;
import dev.linhvu.tracing.BaggageManager;
import dev.linhvu.tracing.TraceContext;

import java.util.ArrayList;
import java.util.Collections;
import java.util.LinkedHashMap;
import java.util.List;
import java.util.Map;
import java.util.Set;
import java.util.concurrent.ConcurrentHashMap;

/**
 * An in-memory implementation of {@link BaggageManager} that stores baggage entries
 * in a {@link ConcurrentHashMap} keyed by trace context.
 *
 * <p>Each trace context maps to a thread-local set of {@link SimpleBaggageInScope} entries.
 * This allows different traces to maintain independent baggage while being thread-safe.
 *
 * <p>Baggage lookup by name is <strong>case-insensitive</strong>, matching the real
 * framework's behavior where HTTP header names (which carry baggage) are case-insensitive.
 */
public class SimpleBaggageManager implements BaggageManager {

    private final Map<TraceContext, Set<SimpleBaggageInScope>> baggagesByContext =
            new ConcurrentHashMap<>();

    private final SimpleTracer tracer;

    private final List<String> remoteFields;

    SimpleBaggageManager(SimpleTracer tracer) {
        this(tracer, List.of());
    }

    SimpleBaggageManager(SimpleTracer tracer, List<String> remoteFields) {
        this.tracer = tracer;
        this.remoteFields = new ArrayList<>(remoteFields);
    }

    @Override
    public Map<String, String> getAllBaggage() {
        TraceContext current = tracer.currentTraceContext().context();
        if (current == null) {
            return Collections.emptyMap();
        }
        return getAllBaggageForCtx(current);
    }

    @Override
    public Map<String, String> getAllBaggage(TraceContext traceContext) {
        if (traceContext == null) {
            return getAllBaggage();
        }
        return getAllBaggageForCtx(traceContext);
    }

    /**
     * Collects all in-scope baggage entries for a given trace context into a map.
     */
    Map<String, String> getAllBaggageForCtx(TraceContext traceContext) {
        Set<SimpleBaggageInScope> entries = baggagesByContext.get(traceContext);
        if (entries == null || entries.isEmpty()) {
            return Collections.emptyMap();
        }

        Map<String, String> result = new LinkedHashMap<>();
        for (SimpleBaggageInScope entry : entries) {
            if (entry.isInScope()) {
                String val = entry.getValue();
                if (val != null) {
                    result.put(entry.name(), val);
                }
            }
        }
        return result;
    }

    @Override
    public Baggage getBaggage(String name) {
        TraceContext current = tracer.currentTraceContext().context();
        if (current == null) {
            return Baggage.NOOP;
        }
        Baggage found = getBaggage(current, name);
        return found != null ? found : Baggage.NOOP;
    }

    @Override
    public Baggage getBaggage(TraceContext traceContext, String name) {
        return baggageForName(traceContext, name);
    }

    @Override
    public Baggage createBaggage(String name) {
        return createSimpleBaggage(name);
    }

    @Override
    public Baggage createBaggage(String name, String value) {
        SimpleBaggageInScope baggage = createSimpleBaggage(name);
        baggage.set(value);
        return baggage;
    }

    @Override
    public BaggageInScope createBaggageInScope(String name, String value) {
        return createSimpleBaggage(name).makeCurrent(value);
    }

    @Override
    public BaggageInScope createBaggageInScope(TraceContext traceContext, String name,
            String value) {
        return createSimpleBaggage(name).makeCurrent(traceContext, value);
    }

    @Override
    public List<String> getBaggageFields() {
        return remoteFields;
    }

    // --- Internal helpers ---

    /**
     * Creates a new baggage entry or returns an existing one with the same name.
     */
    private SimpleBaggageInScope createSimpleBaggage(String name) {
        TraceContext current = tracer.currentTraceContext().context();

        // Look for existing baggage with this name in the current context
        SimpleBaggageInScope existing = baggageForName(current, name);
        if (existing != null) {
            return existing;
        }

        // Create new baggage entry
        SimpleBaggageInScope baggage = new SimpleBaggageInScope(
                tracer.currentTraceContext(), name, current);

        // Register with the current context
        if (current != null) {
            baggagesByContext.computeIfAbsent(current, k -> ConcurrentHashMap.newKeySet())
                    .add(baggage);
        }

        return baggage;
    }

    /**
     * Finds a baggage entry by name (case-insensitive) for a given trace context.
     */
    private SimpleBaggageInScope baggageForName(TraceContext traceContext, String name) {
        if (traceContext == null || name == null) {
            return null;
        }
        Set<SimpleBaggageInScope> entries = baggagesByContext.get(traceContext);
        if (entries == null) {
            return null;
        }
        return entries.stream()
                .filter(b -> b.name().equalsIgnoreCase(name))
                .findFirst()
                .orElse(null);
    }

}
```

#### File: `src/main/java/dev/linhvu/tracing/simple/SimpleTracer.java` [MODIFIED]

```java
package dev.linhvu.tracing.simple;

import dev.linhvu.tracing.Baggage;
import dev.linhvu.tracing.BaggageInScope;
import dev.linhvu.tracing.ScopedSpan;
import dev.linhvu.tracing.Span;
import dev.linhvu.tracing.SpanCustomizer;
import dev.linhvu.tracing.TraceContext;
import dev.linhvu.tracing.Tracer;
import dev.linhvu.tracing.exporter.FinishedSpan;
import dev.linhvu.tracing.exporter.SpanExportingPredicate;
import dev.linhvu.tracing.exporter.SpanFilter;
import dev.linhvu.tracing.exporter.SpanReporter;

import java.util.ArrayList;
import java.util.Deque;
import java.util.List;
import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.LinkedBlockingDeque;

/**
 * An in-memory implementation of {@link Tracer} that creates real spans with
 * generated hex IDs, manages scope via ThreadLocal, and collects all created
 * spans in a deque for test inspection.
 *
 * <p>Key design decisions:
 * <ul>
 *   <li><strong>Instance-based state</strong> — unlike the real framework's static fields,
 *       all state is per-instance so tests can create isolated tracers without interference.</li>
 *   <li><strong>TraceContext→Span map</strong> — when {@code newScope(context)} is called,
 *       the tracer looks up which SimpleSpan owns that context to make it "current".</li>
 *   <li><strong>Baggage delegation</strong> — all BaggageManager methods delegate to
 *       a {@link SimpleBaggageManager} instance for real in-memory baggage management.</li>
 * </ul>
 */
public class SimpleTracer implements Tracer {

    private final Map<TraceContext, SimpleSpan> contextToSpan = new ConcurrentHashMap<>();

    private final ThreadLocal<SimpleSpan> currentSpan = new ThreadLocal<>();

    private final SimpleCurrentTraceContext currentTraceContext;

    private final Deque<SimpleSpan> spans = new LinkedBlockingDeque<>();

    private final List<SpanExportingPredicate> exportingPredicates;

    private final List<SpanFilter> spanFilters;

    private final List<SpanReporter> spanReporters;

    final SimpleBaggageManager simpleBaggageManager;

    public SimpleTracer() {
        this(List.of(), List.of(), List.of());
    }

    public SimpleTracer(List<SpanExportingPredicate> exportingPredicates,
            List<SpanFilter> spanFilters, List<SpanReporter> spanReporters) {
        this.currentTraceContext = new SimpleCurrentTraceContext(this);
        this.exportingPredicates = new ArrayList<>(exportingPredicates);
        this.spanFilters = new ArrayList<>(spanFilters);
        this.spanReporters = new ArrayList<>(spanReporters);
        this.simpleBaggageManager = new SimpleBaggageManager(this);
    }

    @Override
    public SimpleSpan nextSpan() {
        SimpleSpan span = createSpan(currentSpan());
        return span;
    }

    @Override
    public SimpleSpan nextSpan(Span parent) {
        SimpleSpan span = createSpan(parent);
        return span;
    }

    private SimpleSpan createSpan(Span parent) {
        SimpleSpan span = new SimpleSpan();
        SimpleTraceContext ctx = span.context();

        if (parent != null) {
            // Child span: inherit traceId, record parent's spanId
            ctx.setTraceId(parent.context().traceId());
            ctx.setParentId(parent.context().spanId());
            ctx.setSpanId(ctx.generateId());
        }
        else {
            // Root span: traceId = spanId (new trace)
            String id = ctx.generateId();
            ctx.setTraceId(id);
            ctx.setSpanId(id);
        }

        contextToSpan.put(ctx, span);
        spans.add(span);
        span.setOnSpanEnded(() -> exportSpan(span));
        return span;
    }

    /**
     * Runs the finished span through the export pipeline:
     * predicates (gate) → filters (transform) → reporters (sink).
     */
    private void exportSpan(SimpleSpan span) {
        // Gate: if any predicate says no, drop the span
        for (SpanExportingPredicate predicate : exportingPredicates) {
            if (!predicate.isExportable(span)) {
                return;
            }
        }

        // Transform: apply each filter in order
        FinishedSpan result = span;
        for (SpanFilter filter : spanFilters) {
            result = filter.map(result);
        }

        // Sink: deliver to each reporter
        for (SpanReporter reporter : spanReporters) {
            reporter.report(result);
        }
    }

    @Override
    public SimpleSpanInScope withSpan(Span span) {
        return new SimpleSpanInScope(
                currentTraceContext.newScope(span != null ? span.context() : null));
    }

    @Override
    public ScopedSpan startScopedSpan(String name) {
        return new SimpleScopedSpan(this).name(name);
    }

    @Override
    public SimpleSpanBuilder spanBuilder() {
        return new SimpleSpanBuilder(this);
    }

    @Override
    public TraceContext.Builder traceContextBuilder() {
        return new SimpleTraceContextBuilder();
    }

    @Override
    public SimpleCurrentTraceContext currentTraceContext() {
        return currentTraceContext;
    }

    @Override
    public SpanCustomizer currentSpanCustomizer() {
        return new SimpleSpanCustomizer(this);
    }

    @Override
    public SimpleSpan currentSpan() {
        return currentSpan.get();
    }

    // --- BaggageManager (delegates to SimpleBaggageManager) ---

    @Override
    public Map<String, String> getAllBaggage() {
        return simpleBaggageManager.getAllBaggage();
    }

    @Override
    public Map<String, String> getAllBaggage(TraceContext traceContext) {
        if (traceContext == null) {
            return getAllBaggage();
        }
        return simpleBaggageManager.getAllBaggageForCtx(traceContext);
    }

    @Override
    public Baggage getBaggage(String name) {
        return simpleBaggageManager.getBaggage(name);
    }

    @Override
    public Baggage getBaggage(TraceContext traceContext, String name) {
        return simpleBaggageManager.getBaggage(traceContext, name);
    }

    @Override
    public Baggage createBaggage(String name) {
        return simpleBaggageManager.createBaggage(name);
    }

    @Override
    public Baggage createBaggage(String name, String value) {
        return simpleBaggageManager.createBaggage(name, value);
    }

    @Override
    public BaggageInScope createBaggageInScope(String name, String value) {
        return simpleBaggageManager.createBaggageInScope(name, value);
    }

    @Override
    public BaggageInScope createBaggageInScope(TraceContext traceContext, String name,
            String value) {
        return simpleBaggageManager.createBaggageInScope(traceContext, name, value);
    }

    @Override
    public List<String> getBaggageFields() {
        return simpleBaggageManager.getBaggageFields();
    }

    // --- Convenience methods for test inspection ---

    /**
     * Returns all created spans (in creation order).
     */
    public Deque<SimpleSpan> getSpans() {
        return spans;
    }

    /**
     * Returns the single created span. Asserts exactly one span exists and
     * that it has been started and ended.
     */
    public SimpleSpan onlySpan() {
        if (spans.size() != 1) {
            throw new AssertionError("Expected 1 span, got " + spans.size());
        }
        SimpleSpan span = spans.getFirst();
        if (span.getStartTimestamp().toEpochMilli() == 0) {
            throw new AssertionError("Span must be started");
        }
        if (span.getEndTimestamp().toEpochMilli() == 0) {
            throw new AssertionError("Span must be finished");
        }
        return span;
    }

    /**
     * Returns the most recently created span.
     */
    public SimpleSpan lastSpan() {
        if (spans.isEmpty()) {
            throw new AssertionError("No spans created");
        }
        return spans.getLast();
    }

    // --- Package-private scope management (used by SimpleCurrentTraceContext) ---

    SimpleSpan getCurrentSpanInternal() {
        return currentSpan.get();
    }

    void setCurrentSpanDirect(SimpleSpan span) {
        currentSpan.set(span);
    }

    void resetCurrentSpan() {
        currentSpan.remove();
    }

    void setCurrentSpan(TraceContext context) {
        SimpleSpan span = contextToSpan.get(context);
        if (span != null) {
            currentSpan.set(span);
        }
        else {
            currentSpan.remove();
        }
    }

    void registerSpan(SimpleSpan span) {
        contextToSpan.put(span.context(), span);
        spans.add(span);
        span.setOnSpanEnded(() -> exportSpan(span));
    }

}
```

### Test Code

#### File: `src/test/java/dev/linhvu/tracing/simple/SimpleBaggageInScopeTest.java` [NEW]

```java
package dev.linhvu.tracing.simple;

import dev.linhvu.tracing.BaggageInScope;
import dev.linhvu.tracing.Span;
import dev.linhvu.tracing.Tracer;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;

/**
 * Tests {@link SimpleBaggageInScope} — the single class implementing both
 * {@link dev.linhvu.tracing.Baggage} and {@link BaggageInScope}.
 */
class SimpleBaggageInScopeTest {

    private SimpleTracer tracer;

    private SimpleSpan span;

    @BeforeEach
    void setUp() {
        tracer = new SimpleTracer();
        span = tracer.nextSpan().name("test-span").start();
    }

    @Test
    void shouldReturnName_WhenCreated() {
        SimpleBaggageInScope baggage = new SimpleBaggageInScope(
                tracer.currentTraceContext(), "user-id");

        assertThat(baggage.name()).isEqualTo("user-id");
    }

    @Test
    void shouldReturnNullFromGet_WhenNotInScope() {
        try (Tracer.SpanInScope ws = tracer.withSpan(span)) {
            SimpleBaggageInScope baggage = new SimpleBaggageInScope(
                    tracer.currentTraceContext(), "user-id", span.context());
            baggage.set("user-123");

            // Not in scope yet — get() should return null
            assertThat(baggage.get()).isNull();
        }
    }

    @Test
    void shouldReturnValue_WhenInScope() {
        try (Tracer.SpanInScope ws = tracer.withSpan(span)) {
            SimpleBaggageInScope baggage = new SimpleBaggageInScope(
                    tracer.currentTraceContext(), "user-id", span.context());
            baggage.set("user-123");
            baggage.makeCurrent();

            assertThat(baggage.get()).isEqualTo("user-123");
        }
    }

    @Test
    void shouldReturnNullFromGet_WhenClosedAfterMakeCurrent() {
        try (Tracer.SpanInScope ws = tracer.withSpan(span)) {
            SimpleBaggageInScope baggage = new SimpleBaggageInScope(
                    tracer.currentTraceContext(), "user-id", span.context());
            baggage.set("user-123").makeCurrent();

            // Close the scope
            baggage.close();

            assertThat(baggage.get()).isNull();
        }
    }

    @Test
    void shouldReturnValueFromGetWithContext_WhenContextMatches() {
        try (Tracer.SpanInScope ws = tracer.withSpan(span)) {
            SimpleBaggageInScope baggage = new SimpleBaggageInScope(
                    tracer.currentTraceContext(), "user-id", span.context());
            baggage.set("user-123").makeCurrent();

            assertThat(baggage.get(span.context())).isEqualTo("user-123");
        }
    }

    @Test
    void shouldReturnNullFromGetWithContext_WhenContextDoesNotMatch() {
        try (Tracer.SpanInScope ws = tracer.withSpan(span)) {
            SimpleBaggageInScope baggage = new SimpleBaggageInScope(
                    tracer.currentTraceContext(), "user-id", span.context());
            baggage.set("user-123").makeCurrent();

            // Different context — should not return the value
            SimpleSpan otherSpan = tracer.nextSpan().start();
            assertThat(baggage.get(otherSpan.context())).isNull();
        }
    }

    @Test
    void shouldUpdateValue_WhenSetCalledMultipleTimes() {
        try (Tracer.SpanInScope ws = tracer.withSpan(span)) {
            SimpleBaggageInScope baggage = new SimpleBaggageInScope(
                    tracer.currentTraceContext(), "user-id", span.context());
            baggage.set("user-123").makeCurrent();
            assertThat(baggage.get()).isEqualTo("user-123");

            baggage.set("user-456");
            assertThat(baggage.get()).isEqualTo("user-456");
        }
    }

    @Test
    void shouldReturnSelfFromSet() {
        SimpleBaggageInScope baggage = new SimpleBaggageInScope(
                tracer.currentTraceContext(), "user-id");

        assertThat(baggage.set("value")).isSameAs(baggage);
    }

    @Test
    void shouldReturnSelfFromMakeCurrent() {
        SimpleBaggageInScope baggage = new SimpleBaggageInScope(
                tracer.currentTraceContext(), "user-id");

        BaggageInScope result = baggage.makeCurrent();

        assertThat(result).isSameAs(baggage);
    }

    @Test
    void shouldSupportMakeCurrentWithValue_AsConvenienceMethod() {
        try (Tracer.SpanInScope ws = tracer.withSpan(span)) {
            SimpleBaggageInScope baggage = new SimpleBaggageInScope(
                    tracer.currentTraceContext(), "user-id", span.context());

            // makeCurrent(value) should set AND put in scope
            BaggageInScope bis = baggage.makeCurrent("user-789");

            assertThat(bis.get()).isEqualTo("user-789");
            assertThat(bis).isSameAs(baggage);
        }
    }

    @Test
    void shouldSupportMakeCurrentWithContextAndValue() {
        try (Tracer.SpanInScope ws = tracer.withSpan(span)) {
            SimpleBaggageInScope baggage = new SimpleBaggageInScope(
                    tracer.currentTraceContext(), "user-id");

            BaggageInScope bis = baggage.makeCurrent(span.context(), "user-789");

            assertThat(bis.get(span.context())).isEqualTo("user-789");
        }
    }

    @Test
    void shouldBeEqualByName() {
        SimpleBaggageInScope a = new SimpleBaggageInScope(
                tracer.currentTraceContext(), "user-id");
        SimpleBaggageInScope b = new SimpleBaggageInScope(
                tracer.currentTraceContext(), "user-id");

        assertThat(a).isEqualTo(b);
        assertThat(a.hashCode()).isEqualTo(b.hashCode());
    }

    @Test
    void shouldNotBeEqual_WhenDifferentNames() {
        SimpleBaggageInScope a = new SimpleBaggageInScope(
                tracer.currentTraceContext(), "user-id");
        SimpleBaggageInScope b = new SimpleBaggageInScope(
                tracer.currentTraceContext(), "request-id");

        assertThat(a).isNotEqualTo(b);
    }

    @Test
    void shouldReturnNullFromGet_WhenNoCurrentContext() {
        // No span in scope → no current context
        SimpleBaggageInScope baggage = new SimpleBaggageInScope(
                tracer.currentTraceContext(), "user-id");
        baggage.makeCurrent();

        assertThat(baggage.get()).isNull();
    }

}
```

#### File: `src/test/java/dev/linhvu/tracing/simple/SimpleBaggageManagerTest.java` [NEW]

```java
package dev.linhvu.tracing.simple;

import dev.linhvu.tracing.Baggage;
import dev.linhvu.tracing.BaggageInScope;
import dev.linhvu.tracing.Span;
import dev.linhvu.tracing.Tracer;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.util.Map;

import static org.assertj.core.api.Assertions.assertThat;

/**
 * Tests {@link SimpleBaggageManager} through the {@link SimpleTracer} which delegates
 * all BaggageManager methods to it.
 */
class SimpleBaggageManagerTest {

    private SimpleTracer tracer;

    private SimpleSpan span;

    @BeforeEach
    void setUp() {
        tracer = new SimpleTracer();
        span = tracer.nextSpan().name("test-span").start();
    }

    // --- createBaggage ---

    @Test
    void shouldCreateBaggage_WhenInScope() {
        try (Tracer.SpanInScope ws = tracer.withSpan(span)) {
            Baggage baggage = tracer.createBaggage("user-id");

            assertThat(baggage).isNotNull();
            assertThat(baggage.name()).isEqualTo("user-id");
        }
    }

    @Test
    void shouldCreateBaggageWithValue() {
        try (Tracer.SpanInScope ws = tracer.withSpan(span)) {
            Baggage baggage = tracer.createBaggage("user-id", "user-123");

            assertThat(baggage.name()).isEqualTo("user-id");
            // Value is set but not in scope yet — get() returns null
            try (BaggageInScope bis = baggage.makeCurrent()) {
                assertThat(baggage.get()).isEqualTo("user-123");
            }
        }
    }

    @Test
    void shouldReturnExistingBaggage_WhenCreatedTwiceWithSameName() {
        try (Tracer.SpanInScope ws = tracer.withSpan(span)) {
            Baggage first = tracer.createBaggage("user-id");
            Baggage second = tracer.createBaggage("user-id");

            assertThat(first).isSameAs(second);
        }
    }

    // --- createBaggageInScope ---

    @Test
    void shouldCreateBaggageInScope_WhenUsingConvenienceMethod() {
        try (Tracer.SpanInScope ws = tracer.withSpan(span)) {
            try (BaggageInScope bis = tracer.createBaggageInScope("user-id", "user-123")) {
                assertThat(bis.name()).isEqualTo("user-id");
                assertThat(bis.get()).isEqualTo("user-123");
            }
        }
    }

    @Test
    void shouldCreateBaggageInScopeWithContext() {
        try (Tracer.SpanInScope ws = tracer.withSpan(span)) {
            try (BaggageInScope bis = tracer.createBaggageInScope(
                    span.context(), "request-id", "req-456")) {
                assertThat(bis.get(span.context())).isEqualTo("req-456");
            }
        }
    }

    // --- getBaggage ---

    @Test
    void shouldReturnNoopBaggage_WhenNoSpanInScope() {
        // No span in scope
        Baggage baggage = tracer.getBaggage("user-id");

        assertThat(baggage).isSameAs(Baggage.NOOP);
    }

    @Test
    void shouldReturnBaggageByName_WhenCreatedAndInScope() {
        try (Tracer.SpanInScope ws = tracer.withSpan(span)) {
            tracer.createBaggage("user-id", "user-123");

            Baggage found = tracer.getBaggage("user-id");

            assertThat(found).isNotNull();
            assertThat(found.name()).isEqualTo("user-id");
        }
    }

    @Test
    void shouldReturnBaggageByNameCaseInsensitive() {
        try (Tracer.SpanInScope ws = tracer.withSpan(span)) {
            tracer.createBaggage("User-Id", "user-123");

            Baggage found = tracer.getBaggage("user-id");

            assertThat(found).isNotNull();
            assertThat(found.name()).isEqualTo("User-Id");
        }
    }

    @Test
    void shouldReturnBaggageByNameAndContext() {
        try (Tracer.SpanInScope ws = tracer.withSpan(span)) {
            tracer.createBaggage("user-id", "user-123");

            Baggage found = tracer.getBaggage(span.context(), "user-id");

            assertThat(found).isNotNull();
            assertThat(found.name()).isEqualTo("user-id");
        }
    }

    @Test
    void shouldReturnNull_WhenBaggageNotFoundForContext() {
        try (Tracer.SpanInScope ws = tracer.withSpan(span)) {
            Baggage found = tracer.getBaggage(span.context(), "nonexistent");

            assertThat(found).isNull();
        }
    }

    // --- getAllBaggage ---

    @Test
    void shouldReturnEmptyMap_WhenNoSpanInScope() {
        Map<String, String> all = tracer.getAllBaggage();

        assertThat(all).isEmpty();
    }

    @Test
    void shouldReturnAllInScopeBaggage() {
        try (Tracer.SpanInScope ws = tracer.withSpan(span)) {
            try (BaggageInScope uid = tracer.createBaggageInScope("user-id", "user-123");
                    BaggageInScope rid = tracer.createBaggageInScope("request-id", "req-456")) {

                Map<String, String> all = tracer.getAllBaggage();

                assertThat(all)
                        .containsEntry("user-id", "user-123")
                        .containsEntry("request-id", "req-456")
                        .hasSize(2);
            }
        }
    }

    @Test
    void shouldNotIncludeClosedBaggage_WhenGettingAll() {
        try (Tracer.SpanInScope ws = tracer.withSpan(span)) {
            BaggageInScope uid = tracer.createBaggageInScope("user-id", "user-123");
            try (BaggageInScope rid = tracer.createBaggageInScope("request-id", "req-456")) {
                // Close one of the two
                uid.close();

                Map<String, String> all = tracer.getAllBaggage();

                assertThat(all)
                        .doesNotContainKey("user-id")
                        .containsEntry("request-id", "req-456")
                        .hasSize(1);
            }
        }
    }

    @Test
    void shouldReturnAllBaggageForSpecificContext() {
        try (Tracer.SpanInScope ws = tracer.withSpan(span)) {
            try (BaggageInScope uid = tracer.createBaggageInScope("user-id", "user-123")) {
                Map<String, String> all = tracer.getAllBaggage(span.context());

                assertThat(all).containsEntry("user-id", "user-123");
            }
        }
    }

    @Test
    void shouldDelegateToGetAllBaggage_WhenContextIsNull() {
        try (Tracer.SpanInScope ws = tracer.withSpan(span)) {
            try (BaggageInScope uid = tracer.createBaggageInScope("user-id", "user-123")) {
                Map<String, String> all = tracer.getAllBaggage(null);

                assertThat(all).containsEntry("user-id", "user-123");
            }
        }
    }

    // --- getBaggageFields ---

    @Test
    void shouldReturnEmptyBaggageFields_WhenDefault() {
        assertThat(tracer.getBaggageFields()).isEmpty();
    }

}
```

#### File: `src/test/java/dev/linhvu/tracing/integration/BaggageIntegrationTest.java` [NEW]

```java
package dev.linhvu.tracing.integration;

import dev.linhvu.tracing.Baggage;
import dev.linhvu.tracing.BaggageInScope;
import dev.linhvu.tracing.Span;
import dev.linhvu.tracing.Tracer;
import dev.linhvu.tracing.simple.SimpleSpan;
import dev.linhvu.tracing.simple.SimpleTracer;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.util.Map;

import static org.assertj.core.api.Assertions.assertThat;

/**
 * Integration tests verifying that baggage works correctly with span scope management,
 * parent-child relationships, and the full tracer lifecycle.
 */
class BaggageIntegrationTest {

    private SimpleTracer tracer;

    @BeforeEach
    void setUp() {
        tracer = new SimpleTracer();
    }

    @Test
    void shouldManageBaggageLifecycle_WhenUsingTryWithResources() {
        SimpleSpan span = tracer.nextSpan().name("request").start();
        try (Tracer.SpanInScope ws = tracer.withSpan(span)) {
            // Create baggage and place in scope
            try (BaggageInScope userId = tracer.createBaggageInScope("user-id", "user-123")) {
                // Baggage is visible
                assertThat(userId.get()).isEqualTo("user-123");
                assertThat(tracer.getAllBaggage()).containsEntry("user-id", "user-123");
            }

            // After closing, baggage is no longer visible
            assertThat(tracer.getAllBaggage()).isEmpty();
        }
        finally {
            span.end();
        }
    }

    @Test
    void shouldSupportMultipleBaggageEntries_WhenInSameScope() {
        SimpleSpan span = tracer.nextSpan().name("request").start();
        try (Tracer.SpanInScope ws = tracer.withSpan(span)) {
            try (BaggageInScope userId = tracer.createBaggageInScope("user-id", "user-123");
                    BaggageInScope requestId = tracer.createBaggageInScope("request-id", "req-456");
                    BaggageInScope tenantId = tracer.createBaggageInScope("tenant-id", "tenant-A")) {

                Map<String, String> all = tracer.getAllBaggage();
                assertThat(all).hasSize(3)
                        .containsEntry("user-id", "user-123")
                        .containsEntry("request-id", "req-456")
                        .containsEntry("tenant-id", "tenant-A");
            }
        }
        finally {
            span.end();
        }
    }

    @Test
    void shouldUpdateBaggageValue_WhenSetCalledAgain() {
        SimpleSpan span = tracer.nextSpan().name("request").start();
        try (Tracer.SpanInScope ws = tracer.withSpan(span)) {
            Baggage userId = tracer.createBaggage("user-id");
            try (BaggageInScope bis = userId.makeCurrent("user-123")) {
                assertThat(bis.get()).isEqualTo("user-123");

                // Update the value
                userId.set("user-456");
                assertThat(bis.get()).isEqualTo("user-456");
            }
        }
        finally {
            span.end();
        }
    }

    @Test
    void shouldRetrieveBaggageByName_WhenUsingGetBaggage() {
        SimpleSpan span = tracer.nextSpan().name("request").start();
        try (Tracer.SpanInScope ws = tracer.withSpan(span)) {
            tracer.createBaggageInScope("user-id", "user-123");

            // Retrieve by name
            Baggage found = tracer.getBaggage("user-id");
            assertThat(found).isNotNull();
            assertThat(found.name()).isEqualTo("user-id");
        }
        finally {
            span.end();
        }
    }

    @Test
    void shouldWorkWithNestedSpanScopes() {
        // Parent span with baggage
        SimpleSpan parentSpan = tracer.nextSpan().name("parent").start();
        try (Tracer.SpanInScope parentScope = tracer.withSpan(parentSpan)) {
            try (BaggageInScope userId = tracer.createBaggageInScope("user-id", "user-123")) {

                assertThat(tracer.getAllBaggage()).containsEntry("user-id", "user-123");

                // Child span — baggage is still associated with parent context
                SimpleSpan childSpan = tracer.nextSpan().name("child").start();
                try (Tracer.SpanInScope childScope = tracer.withSpan(childSpan)) {
                    // The parent's baggage is retrievable through getBaggage
                    Baggage found = tracer.getBaggage(parentSpan.context(), "user-id");
                    assertThat(found).isNotNull();
                    assertThat(found.name()).isEqualTo("user-id");
                }
                finally {
                    childSpan.end();
                }
            }
        }
        finally {
            parentSpan.end();
        }
    }

    @Test
    void shouldSupportBaggageWithExplicitTraceContext() {
        SimpleSpan span = tracer.nextSpan().name("request").start();
        try (Tracer.SpanInScope ws = tracer.withSpan(span)) {
            // Create baggage explicitly for this context
            try (BaggageInScope bis = tracer.createBaggageInScope(
                    span.context(), "session-id", "sess-789")) {

                assertThat(bis.get(span.context())).isEqualTo("sess-789");
            }
        }
        finally {
            span.end();
        }
    }

    @Test
    void shouldHandleClosingBaggageGracefully_WhenClosedMultipleTimes() {
        SimpleSpan span = tracer.nextSpan().name("request").start();
        try (Tracer.SpanInScope ws = tracer.withSpan(span)) {
            BaggageInScope bis = tracer.createBaggageInScope("user-id", "user-123");

            // Close twice — should not throw
            bis.close();
            bis.close();

            assertThat(bis.get()).isNull();
        }
        finally {
            span.end();
        }
    }

    @Test
    void shouldReturnNoopBaggage_WhenNoSpanInScope() {
        Baggage baggage = tracer.getBaggage("user-id");

        assertThat(baggage).isSameAs(Baggage.NOOP);
    }

    @Test
    void shouldReturnEmptyBaggage_WhenNoSpanInScope() {
        Map<String, String> all = tracer.getAllBaggage();

        assertThat(all).isEmpty();
    }

    @Test
    void shouldSupportBaggageAcrossSpanPattern_WhenCreatedAndUsedInDifferentScopes() {
        SimpleSpan span = tracer.nextSpan().name("request").start();
        Baggage userId;

        // Create baggage in first scope
        try (Tracer.SpanInScope ws = tracer.withSpan(span)) {
            userId = tracer.createBaggage("user-id", "user-123");
            try (BaggageInScope bis = userId.makeCurrent()) {
                assertThat(userId.get()).isEqualTo("user-123");
            }
        }
        finally {
            span.end();
        }

        // After scope is closed, value is no longer accessible via get()
        assertThat(userId.get()).isNull();
    }

}
```

---

## Summary

| Concept | What It Means |
|---------|---------------|
| **Baggage** | A mutable key-value entry that rides alongside trace context — extends `BaggageView` with `set()` and `makeCurrent()` |
| **BaggageInScope** | A baggage entry that's been activated via `makeCurrent()` — must be `close()`d to prevent scope leakage |
| **SimpleBaggageInScope** | One class implementing both `Baggage` and `BaggageInScope` — `makeCurrent()` flips `inScope=true`, `close()` flips it back |
| **SimpleBaggageManager** | Registry storing baggage per `TraceContext` in a `ConcurrentHashMap` — supports case-insensitive lookup, deduplication by name |
| **BaggageManager** | Factory interface for creating/retrieving baggage — `Tracer` extends it, so every tracer is also a baggage manager |
| **Scope gating** | `get()` only returns the value when `inScope == true` AND context matches — prevents cross-trace leakage |

**Next: Chapter 7 — Transport Contexts** — `SenderContext` and `ReceiverContext` observation contexts that carry the carrier object and propagation metadata for cross-process communication. These bridge the Micrometer Observation world to the tracing world.
