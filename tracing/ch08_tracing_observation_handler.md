# Chapter 8: TracingObservationHandler (Local Spans)

> Line references based on commit `dceab1b9` of the Micrometer Tracing repository.

## Build Challenge

| Current State | Limitation | Objective |
|---------------|------------|-----------|
| We have a working tracer (Features 1-3), export pipeline (Feature 4), propagator (Feature 5), baggage (Feature 6), and transport contexts (Feature 7) — but observations and tracing are separate worlds | Creating a span requires manual `tracer.nextSpan().start()` / `span.end()` calls; instrumented libraries must depend directly on the tracing API | Build a `TracingObservationHandler` that automatically creates and manages spans from observation lifecycle events, so any `Observation.start()` produces a real span without the library knowing about tracing |

---

## 8.1 The Integration Point

The integration point for this feature is the `ObservationHandler` interface from simple-micrometer. We extend it with a new interface — `TracingObservationHandler` — that maps every observation lifecycle event to a tracing operation. This handler is then registered with the `ObservationRegistry`, and from that moment on, every `Observation.start()` automatically creates a span.

```
ObservationRegistry
    │
    │ registers
    ▼
┌───────────────────────────────────────────────┐
│  TracingObservationHandler<T>                  │
│  extends ObservationHandler<T>                 │
│                                                │
│  onStart()  → create span                      │
│  onScope()  → open/close CurrentTraceContext   │
│  onEvent()  → span.event()                     │
│  onError()  → span.error()                     │
│  onStop()   → tag + end span                   │
│                                                │
│  TracingContext (inner class)                   │
│    ├── span: Span                              │
│    └── scopes: Map<Thread, Scope>              │
└───────────────────────────────────────────────┘
```

Two key decisions here:

1. **Interface with default methods, not abstract class** — This lets `DefaultTracingObservationHandler` (local spans), `PropagatingSenderTracingObservationHandler`, and `PropagatingReceiverTracingObservationHandler` (Feature 9) share all the scope/error/event logic while only overriding how spans are *created* and *finished*.

2. **TracingContext stored in the observation's generic map** — The observation's `Context` has a `Map<Object, Object>` for handler data. We use `context.computeIfAbsent(TracingContext.class, ...)` to store the span and scope, keeping the observation completely unaware of tracing.

This connects **Observation API** to **Tracing API**. To make it work, we need to build:
- `TracingObservationHandler<T>` — the interface with all default methods and the `TracingContext` inner class
- `DefaultTracingObservationHandler` — the concrete handler for local spans

## 8.2 TracingObservationHandler Interface

**New file:** `src/main/java/dev/linhvu/tracing/handler/TracingObservationHandler.java`

```java
package dev.linhvu.tracing.handler;

import dev.linhvu.micrometer.observation.KeyValue;
import dev.linhvu.micrometer.observation.Observation;
import dev.linhvu.micrometer.observation.ObservationHandler;
import dev.linhvu.tracing.CurrentTraceContext;
import dev.linhvu.tracing.Span;
import dev.linhvu.tracing.TraceContext;
import dev.linhvu.tracing.Tracer;

import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;

public interface TracingObservationHandler<T extends Observation.Context>
        extends ObservationHandler<T> {

    default void tagSpan(T context, Span span) {
        for (KeyValue keyValue : context.getAllKeyValues()) {
            if (!keyValue.getKey().equalsIgnoreCase("ERROR")) {
                span.tag(keyValue.getKey(), keyValue.getValue());
            }
            else {
                span.error(new RuntimeException(keyValue.getValue()));
            }
        }
    }

    default String getSpanName(T context) {
        String name = context.getName();
        if (context.getContextualName() != null && !context.getContextualName().isBlank()) {
            name = context.getContextualName();
        }
        return name;
    }

    @Override
    default void onScopeOpened(T context) {
        TracingContext tracingContext = getTracingContext(context);
        Span span = tracingContext.getSpan();
        TraceContext newContext = span != null ? span.context() : null;
        CurrentTraceContext.Scope scope = getTracer().currentTraceContext()
                .maybeScope(newContext);
        tracingContext.setSpanAndScope(span, scope);
    }

    @Override
    default void onScopeClosed(T context) {
        TracingContext tracingContext = getTracingContext(context);
        CurrentTraceContext.Scope scope = tracingContext.getScope();
        if (scope != null) {
            scope.close();
        }
    }

    @Override
    default void onEvent(Observation.Event event, T context) {
        getRequiredSpan(context).event(event.getContextualName());
    }

    @Override
    default void onError(T context) {
        if (context.getError() != null) {
            getRequiredSpan(context).error(context.getError());
        }
    }

    default Span getParentSpan(Observation.Context context) {
        TracingContext tracingContext = context.get(TracingContext.class);
        if (tracingContext != null) {
            return tracingContext.getSpan();
        }

        Observation parentObservation = context.getParentObservation();
        if (parentObservation != null) {
            TracingContext parentTracingCtx = parentObservation.getContext()
                    .get(TracingContext.class);
            if (parentTracingCtx != null) {
                return parentTracingCtx.getSpan();
            }
        }

        return getTracer().currentSpan();
    }

    default TracingContext getTracingContext(T context) {
        return context.computeIfAbsent(TracingContext.class,
                clazz -> new TracingContext());
    }

    @Override
    default boolean supportsContext(Observation.Context context) {
        return context != null;
    }

    default Span getRequiredSpan(T context) {
        Span span = getTracingContext(context).getSpan();
        if (span == null) {
            throw new IllegalStateException(
                    "Span wasn't started - an observation must be started "
                            + "(not only created)");
        }
        return span;
    }

    default void endSpan(T context, Span span) {
        getTracingContext(context).close();
        span.end();
    }

    Tracer getTracer();

    class TracingContext implements AutoCloseable {

        private Span span;

        private final Map<Thread, CurrentTraceContext.Scope> scopes =
                new ConcurrentHashMap<>();

        public Span getSpan() {
            return this.span;
        }

        public void setSpan(Span span) {
            this.span = span;
        }

        public CurrentTraceContext.Scope getScope() {
            return this.scopes.get(Thread.currentThread());
        }

        public void setScope(CurrentTraceContext.Scope scope) {
            if (scope == null) {
                this.scopes.remove(Thread.currentThread());
            }
            else {
                this.scopes.put(Thread.currentThread(), scope);
            }
        }

        public void setSpanAndScope(Span span, CurrentTraceContext.Scope scope) {
            setSpan(span);
            setScope(scope);
        }

        @Override
        public void close() {
        }

        @Override
        public String toString() {
            return "TracingContext{span="
                    + (span != null ? span.context().toString() : "null") + '}';
        }

    }

}
```

The interface is split into three categories of default methods:

- **Lifecycle mapping** (`onScopeOpened`, `onScopeClosed`, `onEvent`, `onError`) — these directly map each observation event to a tracing call
- **Span helpers** (`tagSpan`, `getSpanName`, `getParentSpan`, `getRequiredSpan`, `endSpan`) — reusable logic for creating and finishing spans
- **Context management** (`getTracingContext`, `supportsContext`) — accessing the `TracingContext` stored in the observation

The only abstract method is `getTracer()` — every concrete handler must supply the tracer it works with.

## 8.3 DefaultTracingObservationHandler

**New file:** `src/main/java/dev/linhvu/tracing/handler/DefaultTracingObservationHandler.java`

```java
package dev.linhvu.tracing.handler;

import dev.linhvu.micrometer.observation.Observation;
import dev.linhvu.tracing.Span;
import dev.linhvu.tracing.Tracer;

public class DefaultTracingObservationHandler
        implements TracingObservationHandler<Observation.Context> {

    private final Tracer tracer;

    public DefaultTracingObservationHandler(Tracer tracer) {
        this.tracer = tracer;
    }

    @Override
    public void onStart(Observation.Context context) {
        Span parentSpan = getParentSpan(context);
        Span childSpan = parentSpan != null
                ? getTracer().nextSpan(parentSpan)
                : getTracer().nextSpan();
        childSpan.start();
        getTracingContext(context).setSpan(childSpan);
    }

    @Override
    public void onStop(Observation.Context context) {
        Span span = getRequiredSpan(context);
        span.name(getSpanName(context));
        tagSpan(context, span);
        endSpan(context, span);
    }

    @Override
    public Tracer getTracer() {
        return this.tracer;
    }

}
```

Notice how small this class is — just `onStart`, `onStop`, and `getTracer()`. All the scope management, error recording, event recording, and tagging are inherited from `TracingObservationHandler`'s default methods.

The `onStart` flow:
1. Find the parent span (from parent observation, thread-local, or none)
2. Create a child span (or root span if no parent)
3. Start it and store it in the `TracingContext`

The `onStop` flow:
1. Get the span (throws if not started)
2. Set the name (contextualName if available, else base name)
3. Copy all key-values as tags
4. End the span

## 8.4 Try It Yourself

<details>
<summary>Challenge: Implement the parent span resolution logic</summary>

The `getParentSpan` method needs to find the correct parent span for a new child span. Think about:
- Where might a parent span come from? (manually set on context, parent observation, thread-local)
- Why does it use `context.get()` instead of `context.computeIfAbsent()` for the first check?

```java
default Span getParentSpan(Observation.Context context) {
    // 1. Check if a TracingContext was manually placed on THIS context
    TracingContext tracingContext = context.get(TracingContext.class);
    if (tracingContext != null) {
        return tracingContext.getSpan();
    }

    // 2. Walk to the parent observation's TracingContext
    Observation parentObservation = context.getParentObservation();
    if (parentObservation != null) {
        TracingContext parentTracingCtx = parentObservation.getContext()
                .get(TracingContext.class);
        if (parentTracingCtx != null) {
            return parentTracingCtx.getSpan();
        }
    }

    // 3. Fall back to whatever is current on this thread
    return getTracer().currentSpan();
}
```

Key insight: `context.get()` returns `null` if not present — this is intentional because at this point in `onStart`, the `TracingContext` hasn't been created yet for THIS observation. We only want to check if the USER manually placed one. The `computeIfAbsent` variant is used later in `getTracingContext()` when we're ready to create and store one.

</details>

<details>
<summary>Challenge: Wire the handler into the observation registry</summary>

Given a `SimpleTracer` and an `ObservationRegistry`, register the handler and create an observation that automatically produces a span:

```java
SimpleTracer tracer = new SimpleTracer();
ObservationRegistry registry = ObservationRegistry.create();
registry.observationConfig()
        .observationHandler(new DefaultTracingObservationHandler(tracer));

// This observation now automatically creates and manages a span!
Observation observation = Observation.createNotStarted("my.operation", registry)
        .lowCardinalityKeyValue("status", "200")
        .start();
try (Observation.Scope scope = observation.openScope()) {
    // tracer.currentSpan() is now the span for this observation
    observation.event(Observation.Event.of("checkpoint.reached"));
}
observation.stop();

// Verify: tracer.onlySpan() has name, tags, events, timestamps
```

</details>

## 8.5 Tests

### Unit Tests

**New file:** `src/test/java/dev/linhvu/tracing/handler/TracingObservationHandlerTest.java`

```java
package dev.linhvu.tracing.handler;

import dev.linhvu.micrometer.observation.KeyValue;
import dev.linhvu.micrometer.observation.Observation;
import dev.linhvu.tracing.CurrentTraceContext;
import dev.linhvu.tracing.Span;
import dev.linhvu.tracing.Tracer;
import dev.linhvu.tracing.simple.SimpleSpan;
import dev.linhvu.tracing.simple.SimpleTracer;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.util.Map;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

class TracingObservationHandlerTest {

    private SimpleTracer tracer;
    private TestTracingObservationHandler handler;

    @BeforeEach
    void setUp() {
        tracer = new SimpleTracer();
        handler = new TestTracingObservationHandler(tracer);
    }

    @Test
    void shouldTagSpan_WhenKeyValuesPresent() {
        Observation.Context context = new Observation.Context();
        context.addLowCardinalityKeyValue(KeyValue.of("http.method", "GET"));
        context.addHighCardinalityKeyValue(KeyValue.of("http.url", "/users"));
        SimpleSpan span = tracer.nextSpan().start();

        handler.tagSpan(context, span);

        assertThat(span.getTags()).containsEntry("http.method", "GET");
        assertThat(span.getTags()).containsEntry("http.url", "/users");
    }

    @Test
    void shouldRecordErrorFromKeyValue_WhenErrorKeyPresent() { ... }

    @Test
    void shouldReturnContextualName_WhenContextualNameIsSet() { ... }

    @Test
    void shouldReturnBaseName_WhenContextualNameIsBlank() { ... }

    @Test
    void shouldOpenScope_WhenScopeOpened() { ... }

    @Test
    void shouldCloseScope_WhenScopeClosed() { ... }

    @Test
    void shouldRecordEvent_WhenEventFired() { ... }

    @Test
    void shouldRecordError_WhenErrorPresent() { ... }

    @Test
    void shouldNotRecordError_WhenErrorIsNull() { ... }

    @Test
    void shouldReturnParentSpan_WhenParentObservationHasSpan() { ... }

    @Test
    void shouldReturnCurrentSpan_WhenNoParentObservation() { ... }

    @Test
    void shouldReturnNull_WhenNoParentAndNoCurrentSpan() { ... }

    @Test
    void shouldCreateTracingContext_WhenNotAlreadyPresent() { ... }

    @Test
    void shouldReuseTracingContext_WhenAlreadyPresent() { ... }

    @Test
    void shouldThrowException_WhenSpanNotStarted() { ... }

    @Test
    void shouldSupportAnyNonNullContext() { ... }

    @Test
    void shouldNotSupportNullContext() { ... }
}
```

**New file:** `src/test/java/dev/linhvu/tracing/handler/DefaultTracingObservationHandlerTest.java`

```java
package dev.linhvu.tracing.handler;

import dev.linhvu.micrometer.observation.KeyValue;
import dev.linhvu.micrometer.observation.Observation;
import dev.linhvu.tracing.simple.SimpleSpan;
import dev.linhvu.tracing.simple.SimpleTracer;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;

class DefaultTracingObservationHandlerTest {

    private SimpleTracer tracer;
    private DefaultTracingObservationHandler handler;

    @BeforeEach
    void setUp() {
        tracer = new SimpleTracer();
        handler = new DefaultTracingObservationHandler(tracer);
    }

    @Test
    void shouldCreateRootSpan_WhenNoParentExists() { ... }

    @Test
    void shouldCreateChildSpan_WhenParentSpanExistsInScope() { ... }

    @Test
    void shouldNameSpanWithBaseName_WhenStopped() { ... }

    @Test
    void shouldNameSpanWithContextualName_WhenStopped() { ... }

    @Test
    void shouldTagSpan_WhenStopped() { ... }

    @Test
    void shouldEndSpan_WhenStopped() { ... }

    @Test
    void shouldRecordErrorAndEndSpan_WhenErrorThenStop() { ... }

    @Test
    void shouldReturnTracer() { ... }
}
```

### Integration Tests

**New file:** `src/test/java/dev/linhvu/tracing/integration/TracingObservationIntegrationTest.java`

```java
package dev.linhvu.tracing.integration;

import dev.linhvu.micrometer.observation.KeyValue;
import dev.linhvu.micrometer.observation.Observation;
import dev.linhvu.micrometer.observation.ObservationRegistry;
import dev.linhvu.tracing.handler.DefaultTracingObservationHandler;
import dev.linhvu.tracing.simple.SimpleSpan;
import dev.linhvu.tracing.simple.SimpleTracer;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.util.Map;

import static org.assertj.core.api.Assertions.assertThat;

class TracingObservationIntegrationTest {

    private SimpleTracer tracer;
    private ObservationRegistry registry;

    @BeforeEach
    void setUp() {
        tracer = new SimpleTracer();
        registry = ObservationRegistry.create();
        registry.observationConfig()
                .observationHandler(new DefaultTracingObservationHandler(tracer));
    }

    @Test
    void shouldCreateSpan_WhenObservationStartedAndStopped() { ... }

    @Test
    void shouldUseContextualName_WhenSetDuringObservation() { ... }

    @Test
    void shouldCreateNestedSpans_WhenObservationsAreNested() { ... }

    @Test
    void shouldPropagateScope_WhenScopeOpened() { ... }

    @Test
    void shouldRecordError_WhenObservationHasError() { ... }

    @Test
    void shouldRecordEvent_WhenEventFiredDuringObservation() { ... }

    @Test
    void shouldCombineAllLifecycleEvents_WhenFullObservation() { ... }
}
```

**Run:** `./gradlew test` — expected: all tests pass (including all prior features' tests)

---

## 8.6 Why This Works

> ★ **Insight** -------------------------------------------
> - **Why an interface with default methods instead of an abstract class?** Java interfaces can be composed via multiple implementation, while abstract classes cannot. This means `PropagatingSenderTracingObservationHandler` (Feature 9) can implement `TracingObservationHandler` while also being parameterized on `SenderContext` — no diamond inheritance problem. The interface provides the *behavior*, while each concrete class only adds the *policy* (what kind of span to create, when to inject/extract).
> - **When this breaks down:** If the default methods need mutable shared state (e.g., caching), an abstract class with protected fields would be better. Here, all state lives in the `TracingContext` stored in the observation, so the interface stays stateless.
> -----------------------------------------------------------

> ★ **Insight** -------------------------------------------
> - **Why `computeIfAbsent` for TracingContext but `get` for parent lookup?** These are two different lifecycle moments. `getTracingContext()` is called when you *want* a TracingContext (creating or using the span) — `computeIfAbsent` creates it lazily. `getParentSpan()` is called to *check* if someone else already placed a TracingContext — `get` returns null without side effects. This distinction prevents `getParentSpan` from accidentally creating a TracingContext on the current context before `onStart` has a chance to set one up properly.
> -----------------------------------------------------------

> ★ **Insight** -------------------------------------------
> - **Why per-thread scopes (`ConcurrentHashMap<Thread, Scope>`)?** In a reactive pipeline, `onScopeOpened` and `onScopeClosed` might be called on different threads for the same observation. A single scope field would lose track of which thread needs restoration. The thread-keyed map ensures each thread's scope is independently managed. In our simplified imperative model this is rarely needed, but it's a small cost for correctness in concurrent scenarios.
> -----------------------------------------------------------

## 8.7 What We Enhanced

| Aspect | Before (ch01-ch07) | Current (ch08) | Real Framework |
|--------|---------------------|----------------|----------------|
| Span creation | Manual `tracer.nextSpan().start()` / `span.end()` — libraries must depend on the tracing API | Automatic via `Observation.start()` / `observation.stop()` — libraries only need the observation API | Same pattern: `TracingObservationHandler.java:50` default methods, `DefaultTracingObservationHandler.java:41` |
| Scope management | Manual `tracer.withSpan(span)` in try-with-resources | Automatic via `observation.openScope()` — handler calls `maybeScope()` internally | `TracingObservationHandler.java:79` uses `setMaybeScopeOnTracingContext` with `RevertingScope` for baggage support |
| Error recording | Manual `span.error(throwable)` | Automatic when `observation.error(throwable)` is called | Same: `TracingObservationHandler.java:128` |

## 8.8 Connection to Real Framework

| Simplified Code | Real Framework Code | File:Line | Key Difference |
|----------------|--------------------|-----------| ---------------|
| `TracingObservationHandler` | `TracingObservationHandler` | `TracingObservationHandler.java:38` | Real adds `setMaybeScopeOnTracingContext` with `RevertingScope` for baggage-aware scope management, `onScopeReset` for reactive cleanup |
| `TracingContext` (inner class) | `TracingContext` | `TracingObservationHandler.java:232` | Real has deprecated `getBaggage`/`setBaggage` methods and a `context` field linking back to the observation |
| `tagSpan()` | `tagSpan()` | `TracingObservationHandler.java:50` | Identical logic — iterate key-values, tag span, handle "error" key specially |
| `getSpanName()` | `getSpanName()` | `TracingObservationHandler.java:66` | Real uses `StringUtils.isNotBlank()` from `micrometer-commons`; we use `!isBlank()` |
| `getParentSpan()` | `getParentSpan()` | `TracingObservationHandler.java:152` | Real handles the "user manually created a span between observations" edge case by comparing `currentSpan` with `spanFromParentObservation` |
| `DefaultTracingObservationHandler` | `DefaultTracingObservationHandler` | `DefaultTracingObservationHandler.java:28` | Real adds a null check on `childSpan` (defensive for NOOP tracers) |
| `onStart()` | `onStart()` | `DefaultTracingObservationHandler.java:41` | Identical flow: getParentSpan → nextSpan → start → store |
| `onStop()` | `onStop()` | `DefaultTracingObservationHandler.java:51` | Identical flow: name → tag → endSpan |

## 8.9 Complete Code

### Production Code

#### File: `src/main/java/dev/linhvu/tracing/handler/TracingObservationHandler.java` [NEW]

```java
package dev.linhvu.tracing.handler;

import dev.linhvu.micrometer.observation.KeyValue;
import dev.linhvu.micrometer.observation.Observation;
import dev.linhvu.micrometer.observation.ObservationHandler;
import dev.linhvu.tracing.CurrentTraceContext;
import dev.linhvu.tracing.Span;
import dev.linhvu.tracing.TraceContext;
import dev.linhvu.tracing.Tracer;

import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;

/**
 * Base interface that bridges the Observation lifecycle to the Tracing API.
 * Extends {@link ObservationHandler} with default methods that map each observation
 * event to a tracing operation (scope management, error recording, event recording).
 *
 * <p>Concrete implementations only need to implement {@link #onStart}, {@link #onStop},
 * and {@link #getTracer()} — all other lifecycle methods are handled by default.
 *
 * <p>Uses the inner {@link TracingContext} class to store the span and per-thread scope
 * inside the observation's generic map ({@code context.put/get/computeIfAbsent}).
 * This means the observation itself doesn't need to know about tracing.
 *
 * @param <T> the observation context type this handler operates on
 */
public interface TracingObservationHandler<T extends Observation.Context>
        extends ObservationHandler<T> {

    /**
     * Copies all key-values from the observation context to span tags.
     * Key-values with key "error" (case-insensitive) are recorded as span errors instead.
     */
    default void tagSpan(T context, Span span) {
        for (KeyValue keyValue : context.getAllKeyValues()) {
            if (!keyValue.getKey().equalsIgnoreCase("ERROR")) {
                span.tag(keyValue.getKey(), keyValue.getValue());
            }
            else {
                span.error(new RuntimeException(keyValue.getValue()));
            }
        }
    }

    /**
     * Resolves the span name from the context. Prefers {@code contextualName}
     * (e.g., "GET /users") over the base name (e.g., "http.server.requests").
     */
    default String getSpanName(T context) {
        String name = context.getName();
        if (context.getContextualName() != null && !context.getContextualName().isBlank()) {
            name = context.getContextualName();
        }
        return name;
    }

    /**
     * Opens a {@link CurrentTraceContext.Scope} for the span's trace context,
     * making it the "current" context on this thread.
     */
    @Override
    default void onScopeOpened(T context) {
        TracingContext tracingContext = getTracingContext(context);
        Span span = tracingContext.getSpan();
        TraceContext newContext = span != null ? span.context() : null;
        CurrentTraceContext.Scope scope = getTracer().currentTraceContext()
                .maybeScope(newContext);
        tracingContext.setSpanAndScope(span, scope);
    }

    /**
     * Closes the scope for this observation's span, restoring the previous
     * trace context on this thread.
     */
    @Override
    default void onScopeClosed(T context) {
        TracingContext tracingContext = getTracingContext(context);
        CurrentTraceContext.Scope scope = tracingContext.getScope();
        if (scope != null) {
            scope.close();
        }
    }

    /**
     * Records an observation event as a span event.
     */
    @Override
    default void onEvent(Observation.Event event, T context) {
        getRequiredSpan(context).event(event.getContextualName());
    }

    /**
     * Records the observation's error on the span.
     */
    @Override
    default void onError(T context) {
        if (context.getError() != null) {
            getRequiredSpan(context).error(context.getError());
        }
    }

    /**
     * Finds the parent span for a new child span. Resolution order:
     * <ol>
     *   <li>If a {@link TracingContext} was manually placed on this context, use its span</li>
     *   <li>Walk to the parent observation and use its span</li>
     *   <li>Fall back to the current span on this thread</li>
     * </ol>
     *
     * @return the parent span, or {@code null} if none exists
     */
    default Span getParentSpan(Observation.Context context) {
        // Check if a TracingContext was manually put on this context
        TracingContext tracingContext = context.get(TracingContext.class);
        if (tracingContext != null) {
            return tracingContext.getSpan();
        }

        // Walk to the parent observation to find its span
        Observation parentObservation = context.getParentObservation();
        if (parentObservation != null) {
            TracingContext parentTracingCtx = parentObservation.getContext()
                    .get(TracingContext.class);
            if (parentTracingCtx != null) {
                return parentTracingCtx.getSpan();
            }
        }

        // Fall back to whatever span is current on this thread
        return getTracer().currentSpan();
    }

    /**
     * Gets or creates the {@link TracingContext} stored in the observation context.
     * Uses {@code computeIfAbsent} so the first access creates it and subsequent
     * accesses reuse the same instance.
     */
    default TracingContext getTracingContext(T context) {
        return context.computeIfAbsent(TracingContext.class,
                clazz -> new TracingContext());
    }

    /**
     * Accepts all non-null contexts by default. Subclasses override to filter
     * for specific context types (e.g., {@code SenderContext}).
     */
    @Override
    default boolean supportsContext(Observation.Context context) {
        return context != null;
    }

    /**
     * Returns the span from the tracing context, or throws if the span hasn't
     * been started yet (i.e., {@code onStart} wasn't called).
     */
    default Span getRequiredSpan(T context) {
        Span span = getTracingContext(context).getSpan();
        if (span == null) {
            throw new IllegalStateException(
                    "Span wasn't started - an observation must be started "
                            + "(not only created)");
        }
        return span;
    }

    /**
     * Ends the span and closes the tracing context. Called at the end of
     * the observation lifecycle ({@code onStop}).
     */
    default void endSpan(T context, Span span) {
        getTracingContext(context).close();
        span.end();
    }

    /**
     * Returns the {@link Tracer} used by this handler.
     */
    Tracer getTracer();

    // =========================================================================
    // Inner class: TracingContext
    // =========================================================================

    /**
     * Holds tracing state (span + per-thread scope) inside an observation context.
     * Stored via {@code context.computeIfAbsent(TracingContext.class, ...)} so
     * each observation gets exactly one TracingContext shared across all handlers.
     *
     * <p>The scope map is keyed by {@link Thread} to support observations that
     * open scopes on different threads (e.g., reactive pipelines).
     */
    class TracingContext implements AutoCloseable {

        private Span span;

        private final Map<Thread, CurrentTraceContext.Scope> scopes =
                new ConcurrentHashMap<>();

        /** Returns the span, or {@code null} if not yet set. */
        public Span getSpan() {
            return this.span;
        }

        /** Sets the span. */
        public void setSpan(Span span) {
            this.span = span;
        }

        /** Returns the scope for the current thread, or {@code null}. */
        public CurrentTraceContext.Scope getScope() {
            return this.scopes.get(Thread.currentThread());
        }

        /** Sets (or removes) the scope for the current thread. */
        public void setScope(CurrentTraceContext.Scope scope) {
            if (scope == null) {
                this.scopes.remove(Thread.currentThread());
            }
            else {
                this.scopes.put(Thread.currentThread(), scope);
            }
        }

        /** Convenience method to set both span and scope. */
        public void setSpanAndScope(Span span, CurrentTraceContext.Scope scope) {
            setSpan(span);
            setScope(scope);
        }

        @Override
        public void close() {
            // Cleanup hook — currently no-op
        }

        @Override
        public String toString() {
            return "TracingContext{span="
                    + (span != null ? span.context().toString() : "null") + '}';
        }

    }

}
```

#### File: `src/main/java/dev/linhvu/tracing/handler/DefaultTracingObservationHandler.java` [NEW]

```java
package dev.linhvu.tracing.handler;

import dev.linhvu.micrometer.observation.Observation;
import dev.linhvu.tracing.Span;
import dev.linhvu.tracing.Tracer;

/**
 * The default tracing observation handler for <b>local/in-process</b> spans.
 * Creates a child span when an observation starts and ends it when the observation stops.
 *
 * <p>This handler is suitable for operations that stay within a single process
 * (e.g., a database call, a method invocation). For cross-process communication
 * (HTTP calls, messaging), use the propagating handlers from Feature 9.
 *
 * <p>All scope management, error recording, and event recording are inherited
 * from the {@link TracingObservationHandler} default methods.
 */
public class DefaultTracingObservationHandler
        implements TracingObservationHandler<Observation.Context> {

    private final Tracer tracer;

    public DefaultTracingObservationHandler(Tracer tracer) {
        this.tracer = tracer;
    }

    /**
     * Creates a child span from the parent (or a root span if no parent exists)
     * and stores it in the tracing context.
     */
    @Override
    public void onStart(Observation.Context context) {
        Span parentSpan = getParentSpan(context);
        Span childSpan = parentSpan != null
                ? getTracer().nextSpan(parentSpan)
                : getTracer().nextSpan();
        childSpan.start();
        getTracingContext(context).setSpan(childSpan);
    }

    /**
     * Names the span (using contextual or base name), copies all key-values
     * as tags, and ends the span.
     */
    @Override
    public void onStop(Observation.Context context) {
        Span span = getRequiredSpan(context);
        span.name(getSpanName(context));
        tagSpan(context, span);
        endSpan(context, span);
    }

    @Override
    public Tracer getTracer() {
        return this.tracer;
    }

}
```

### Test Code

#### File: `src/test/java/dev/linhvu/tracing/handler/TracingObservationHandlerTest.java` [NEW]

```java
package dev.linhvu.tracing.handler;

import dev.linhvu.micrometer.observation.KeyValue;
import dev.linhvu.micrometer.observation.Observation;
import dev.linhvu.tracing.CurrentTraceContext;
import dev.linhvu.tracing.Span;
import dev.linhvu.tracing.Tracer;
import dev.linhvu.tracing.simple.SimpleSpan;
import dev.linhvu.tracing.simple.SimpleTracer;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.util.Map;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

/**
 * Tests the default methods of {@link TracingObservationHandler} via
 * a minimal concrete implementation that delegates to SimpleTracer.
 */
class TracingObservationHandlerTest {

    private SimpleTracer tracer;
    private TestTracingObservationHandler handler;

    @BeforeEach
    void setUp() {
        tracer = new SimpleTracer();
        handler = new TestTracingObservationHandler(tracer);
    }

    // --- tagSpan ---

    @Test
    void shouldTagSpan_WhenKeyValuesPresent() {
        // Arrange
        Observation.Context context = new Observation.Context();
        context.addLowCardinalityKeyValue(KeyValue.of("http.method", "GET"));
        context.addHighCardinalityKeyValue(KeyValue.of("http.url", "/users"));
        SimpleSpan span = tracer.nextSpan().start();

        // Act
        handler.tagSpan(context, span);

        // Assert
        assertThat(span.getTags()).containsEntry("http.method", "GET");
        assertThat(span.getTags()).containsEntry("http.url", "/users");
    }

    @Test
    void shouldRecordErrorFromKeyValue_WhenErrorKeyPresent() {
        // Arrange
        Observation.Context context = new Observation.Context();
        context.addLowCardinalityKeyValue(KeyValue.of("error", "something broke"));
        SimpleSpan span = tracer.nextSpan().start();

        // Act
        handler.tagSpan(context, span);

        // Assert
        assertThat(span.getError()).isNotNull();
        assertThat(span.getError()).hasMessage("something broke");
        assertThat(span.getTags()).doesNotContainKey("error");
    }

    // --- getSpanName ---

    @Test
    void shouldReturnContextualName_WhenContextualNameIsSet() {
        // Arrange
        Observation.Context context = new Observation.Context();
        context.setName("http.server.requests");
        context.setContextualName("GET /users");

        // Act & Assert
        assertThat(handler.getSpanName(context)).isEqualTo("GET /users");
    }

    @Test
    void shouldReturnBaseName_WhenContextualNameIsBlank() {
        // Arrange
        Observation.Context context = new Observation.Context();
        context.setName("http.server.requests");

        // Act & Assert
        assertThat(handler.getSpanName(context)).isEqualTo("http.server.requests");
    }

    // --- onScopeOpened / onScopeClosed ---

    @Test
    void shouldOpenScope_WhenScopeOpened() {
        // Arrange
        Observation.Context context = new Observation.Context();
        SimpleSpan span = tracer.nextSpan().start();
        handler.getTracingContext(context).setSpan(span);

        // Act
        handler.onScopeOpened(context);

        // Assert — span's trace context is now current
        assertThat(tracer.currentTraceContext().context())
                .isEqualTo(span.context());
    }

    @Test
    void shouldCloseScope_WhenScopeClosed() {
        // Arrange
        Observation.Context context = new Observation.Context();
        SimpleSpan span = tracer.nextSpan().start();
        handler.getTracingContext(context).setSpan(span);
        handler.onScopeOpened(context);

        // Act
        handler.onScopeClosed(context);

        // Assert — previous context is restored (null since no prior span)
        assertThat(tracer.currentTraceContext().context()).isNull();
    }

    // --- onEvent ---

    @Test
    void shouldRecordEvent_WhenEventFired() {
        // Arrange
        Observation.Context context = new Observation.Context();
        SimpleSpan span = tracer.nextSpan().start();
        handler.getTracingContext(context).setSpan(span);

        // Act
        handler.onEvent(Observation.Event.of("message.received"), context);

        // Assert
        assertThat(span.getEvents())
                .extracting(Map.Entry::getValue)
                .containsExactly("message.received");
    }

    // --- onError ---

    @Test
    void shouldRecordError_WhenErrorPresent() {
        // Arrange
        Observation.Context context = new Observation.Context();
        context.setError(new RuntimeException("connection timeout"));
        SimpleSpan span = tracer.nextSpan().start();
        handler.getTracingContext(context).setSpan(span);

        // Act
        handler.onError(context);

        // Assert
        assertThat(span.getError()).hasMessage("connection timeout");
    }

    @Test
    void shouldNotRecordError_WhenErrorIsNull() {
        // Arrange
        Observation.Context context = new Observation.Context();
        SimpleSpan span = tracer.nextSpan().start();
        handler.getTracingContext(context).setSpan(span);

        // Act — no error set on context
        handler.onError(context);

        // Assert
        assertThat(span.getError()).isNull();
    }

    // --- getParentSpan ---

    @Test
    void shouldReturnParentSpan_WhenParentObservationHasSpan() {
        // Arrange — simulate a parent observation with a TracingContext
        Observation.Context parentContext = new Observation.Context();
        SimpleSpan parentSpan = tracer.nextSpan().start();
        TracingObservationHandler.TracingContext parentTracingCtx =
                new TracingObservationHandler.TracingContext();
        parentTracingCtx.setSpan(parentSpan);
        parentContext.put(TracingObservationHandler.TracingContext.class,
                parentTracingCtx);

        // Simulate parent observation via a stub
        Observation parentObs = new StubObservation(parentContext);

        Observation.Context childContext = new Observation.Context();
        childContext.setParentObservation(parentObs);

        // Act
        Span result = handler.getParentSpan(childContext);

        // Assert
        assertThat(result).isSameAs(parentSpan);
    }

    @Test
    void shouldReturnCurrentSpan_WhenNoParentObservation() {
        // Arrange — put a span in scope
        SimpleSpan scopedSpan = tracer.nextSpan().start();
        try (Tracer.SpanInScope ws = tracer.withSpan(scopedSpan)) {
            Observation.Context context = new Observation.Context();

            // Act
            Span result = handler.getParentSpan(context);

            // Assert
            assertThat(result).isSameAs(scopedSpan);
        }
    }

    @Test
    void shouldReturnNull_WhenNoParentAndNoCurrentSpan() {
        // Arrange
        Observation.Context context = new Observation.Context();

        // Act
        Span result = handler.getParentSpan(context);

        // Assert
        assertThat(result).isNull();
    }

    // --- getTracingContext ---

    @Test
    void shouldCreateTracingContext_WhenNotAlreadyPresent() {
        // Arrange
        Observation.Context context = new Observation.Context();

        // Act
        TracingObservationHandler.TracingContext tc = handler.getTracingContext(context);

        // Assert
        assertThat(tc).isNotNull();
        assertThat(tc.getSpan()).isNull();
    }

    @Test
    void shouldReuseTracingContext_WhenAlreadyPresent() {
        // Arrange
        Observation.Context context = new Observation.Context();
        TracingObservationHandler.TracingContext first = handler.getTracingContext(context);

        // Act
        TracingObservationHandler.TracingContext second = handler.getTracingContext(context);

        // Assert
        assertThat(second).isSameAs(first);
    }

    // --- getRequiredSpan ---

    @Test
    void shouldThrowException_WhenSpanNotStarted() {
        // Arrange
        Observation.Context context = new Observation.Context();

        // Act & Assert
        assertThatThrownBy(() -> handler.getRequiredSpan(context))
                .isInstanceOf(IllegalStateException.class)
                .hasMessageContaining("Span wasn't started");
    }

    // --- supportsContext ---

    @Test
    void shouldSupportAnyNonNullContext() {
        assertThat(handler.supportsContext(new Observation.Context())).isTrue();
    }

    @Test
    void shouldNotSupportNullContext() {
        assertThat(handler.supportsContext(null)).isFalse();
    }

    // =========================================================================
    // Test helpers
    // =========================================================================

    /**
     * Minimal concrete implementation that exposes all default methods for testing.
     */
    static class TestTracingObservationHandler
            implements TracingObservationHandler<Observation.Context> {

        private final Tracer tracer;

        TestTracingObservationHandler(Tracer tracer) {
            this.tracer = tracer;
        }

        @Override
        public void onStart(Observation.Context context) {
            // not tested here — tested via DefaultTracingObservationHandler
        }

        @Override
        public void onStop(Observation.Context context) {
            // not tested here
        }

        @Override
        public Tracer getTracer() {
            return tracer;
        }
    }

    /**
     * Stub observation that just returns a pre-set context.
     */
    static class StubObservation implements Observation {

        private final Context context;

        StubObservation(Context context) {
            this.context = context;
        }

        @Override
        public Context getContext() {
            return context;
        }

        @Override
        public Observation start() {
            return this;
        }

        @Override
        public Observation lowCardinalityKeyValue(
                dev.linhvu.micrometer.observation.KeyValue keyValue) {
            return this;
        }

        @Override
        public Observation highCardinalityKeyValue(
                dev.linhvu.micrometer.observation.KeyValue keyValue) {
            return this;
        }

        @Override
        public Observation lowCardinalityKeyValues(
                dev.linhvu.micrometer.observation.KeyValues keyValues) {
            return this;
        }

        @Override
        public Observation highCardinalityKeyValues(
                dev.linhvu.micrometer.observation.KeyValues keyValues) {
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
        public boolean isNoop() {
            return false;
        }
    }

}
```

#### File: `src/test/java/dev/linhvu/tracing/handler/DefaultTracingObservationHandlerTest.java` [NEW]

```java
package dev.linhvu.tracing.handler;

import dev.linhvu.micrometer.observation.KeyValue;
import dev.linhvu.micrometer.observation.Observation;
import dev.linhvu.tracing.simple.SimpleSpan;
import dev.linhvu.tracing.simple.SimpleTracer;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;

/**
 * Tests for {@link DefaultTracingObservationHandler} — the concrete handler
 * for local/in-process spans.
 */
class DefaultTracingObservationHandlerTest {

    private SimpleTracer tracer;
    private DefaultTracingObservationHandler handler;

    @BeforeEach
    void setUp() {
        tracer = new SimpleTracer();
        handler = new DefaultTracingObservationHandler(tracer);
    }

    @Test
    void shouldCreateRootSpan_WhenNoParentExists() {
        // Arrange
        Observation.Context context = new Observation.Context();
        context.setName("db.query");

        // Act
        handler.onStart(context);

        // Assert
        SimpleSpan span = tracer.lastSpan();
        assertThat(span.getStartTimestamp().toEpochMilli()).isGreaterThan(0);
        assertThat(span.context().traceId()).isNotEmpty();
        assertThat(span.context().parentId()).isEmpty();
    }

    @Test
    void shouldCreateChildSpan_WhenParentSpanExistsInScope() {
        // Arrange — put a parent span in scope
        SimpleSpan parentSpan = tracer.nextSpan().start();
        try (var ws = tracer.withSpan(parentSpan)) {
            Observation.Context context = new Observation.Context();
            context.setName("child-op");

            // Act
            handler.onStart(context);

            // Assert
            SimpleSpan childSpan = tracer.lastSpan();
            assertThat(childSpan.context().traceId())
                    .isEqualTo(parentSpan.context().traceId());
            assertThat(childSpan.context().parentId())
                    .isEqualTo(parentSpan.context().spanId());
            assertThat(childSpan.context().spanId())
                    .isNotEqualTo(parentSpan.context().spanId());
        }
    }

    @Test
    void shouldNameSpanWithBaseName_WhenStopped() {
        // Arrange
        Observation.Context context = new Observation.Context();
        context.setName("http.server.requests");
        handler.onStart(context);

        // Act
        handler.onStop(context);

        // Assert
        SimpleSpan span = tracer.onlySpan();
        assertThat(span.getName()).isEqualTo("http.server.requests");
    }

    @Test
    void shouldNameSpanWithContextualName_WhenStopped() {
        // Arrange
        Observation.Context context = new Observation.Context();
        context.setName("http.server.requests");
        context.setContextualName("GET /users");
        handler.onStart(context);

        // Act
        handler.onStop(context);

        // Assert
        SimpleSpan span = tracer.onlySpan();
        assertThat(span.getName()).isEqualTo("GET /users");
    }

    @Test
    void shouldTagSpan_WhenStopped() {
        // Arrange
        Observation.Context context = new Observation.Context();
        context.setName("db.query");
        context.addLowCardinalityKeyValue(KeyValue.of("db.type", "postgresql"));
        context.addHighCardinalityKeyValue(KeyValue.of("db.statement", "SELECT 1"));
        handler.onStart(context);

        // Act
        handler.onStop(context);

        // Assert
        SimpleSpan span = tracer.onlySpan();
        assertThat(span.getTags()).containsEntry("db.type", "postgresql");
        assertThat(span.getTags()).containsEntry("db.statement", "SELECT 1");
    }

    @Test
    void shouldEndSpan_WhenStopped() {
        // Arrange
        Observation.Context context = new Observation.Context();
        context.setName("my-op");
        handler.onStart(context);

        // Act
        handler.onStop(context);

        // Assert
        SimpleSpan span = tracer.onlySpan();
        assertThat(span.getEndTimestamp().toEpochMilli()).isGreaterThan(0);
    }

    @Test
    void shouldRecordErrorAndEndSpan_WhenErrorThenStop() {
        // Arrange
        Observation.Context context = new Observation.Context();
        context.setName("failing-op");
        handler.onStart(context);
        context.setError(new RuntimeException("boom"));

        // Act
        handler.onError(context);
        handler.onStop(context);

        // Assert
        SimpleSpan span = tracer.onlySpan();
        assertThat(span.getError()).hasMessage("boom");
        assertThat(span.getEndTimestamp().toEpochMilli()).isGreaterThan(0);
    }

    @Test
    void shouldReturnTracer() {
        assertThat(handler.getTracer()).isSameAs(tracer);
    }

}
```

#### File: `src/test/java/dev/linhvu/tracing/integration/TracingObservationIntegrationTest.java` [NEW]

```java
package dev.linhvu.tracing.integration;

import dev.linhvu.micrometer.observation.KeyValue;
import dev.linhvu.micrometer.observation.Observation;
import dev.linhvu.micrometer.observation.ObservationRegistry;
import dev.linhvu.tracing.handler.DefaultTracingObservationHandler;
import dev.linhvu.tracing.simple.SimpleSpan;
import dev.linhvu.tracing.simple.SimpleTracer;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.util.Map;

import static org.assertj.core.api.Assertions.assertThat;

/**
 * Integration tests verifying the full observation → tracing bridge:
 * ObservationRegistry + DefaultTracingObservationHandler + SimpleTracer.
 */
class TracingObservationIntegrationTest {

    private SimpleTracer tracer;
    private ObservationRegistry registry;

    @BeforeEach
    void setUp() {
        tracer = new SimpleTracer();
        registry = ObservationRegistry.create();
        registry.observationConfig()
                .observationHandler(new DefaultTracingObservationHandler(tracer));
    }

    @Test
    void shouldCreateSpan_WhenObservationStartedAndStopped() {
        // Act
        Observation observation = Observation.createNotStarted("test.op", registry)
                .lowCardinalityKeyValue("status", "200")
                .start();
        observation.stop();

        // Assert
        SimpleSpan span = tracer.onlySpan();
        assertThat(span.getName()).isEqualTo("test.op");
        assertThat(span.getTags()).containsEntry("status", "200");
        assertThat(span.getStartTimestamp().toEpochMilli()).isGreaterThan(0);
        assertThat(span.getEndTimestamp().toEpochMilli()).isGreaterThan(0);
    }

    @Test
    void shouldUseContextualName_WhenSetDuringObservation() {
        // Act
        Observation observation = Observation.createNotStarted("http.server", registry)
                .start();
        observation.contextualName("GET /api/users");
        observation.stop();

        // Assert
        SimpleSpan span = tracer.onlySpan();
        assertThat(span.getName()).isEqualTo("GET /api/users");
    }

    @Test
    void shouldCreateNestedSpans_WhenObservationsAreNested() {
        // Act — outer observation
        Observation outer = Observation.createNotStarted("outer", registry).start();
        try (Observation.Scope outerScope = outer.openScope()) {
            // Inner observation — its parent is the outer observation's span
            Observation inner = Observation.createNotStarted("inner", registry)
                    .start();
            inner.stop();
        }
        outer.stop();

        // Assert
        assertThat(tracer.getSpans()).hasSize(2);

        SimpleSpan outerSpan = tracer.getSpans().getFirst();
        SimpleSpan innerSpan = tracer.getSpans().getLast();

        // Same trace
        assertThat(innerSpan.context().traceId())
                .isEqualTo(outerSpan.context().traceId());

        // Parent-child relationship
        assertThat(innerSpan.context().parentId())
                .isEqualTo(outerSpan.context().spanId());
    }

    @Test
    void shouldPropagateScope_WhenScopeOpened() {
        // Act
        Observation observation = Observation.createNotStarted("scoped.op", registry)
                .start();

        // Before scope is opened, the span is NOT current on this thread
        assertThat(tracer.currentSpan()).isNull();

        try (Observation.Scope scope = observation.openScope()) {
            // Inside scope, the span IS current
            assertThat(tracer.currentSpan()).isNotNull();
            assertThat(tracer.currentSpan().context().traceId())
                    .isEqualTo(tracer.lastSpan().context().traceId());
        }

        // After scope is closed, no span is current
        assertThat(tracer.currentSpan()).isNull();
        observation.stop();
    }

    @Test
    void shouldRecordError_WhenObservationHasError() {
        // Act
        Observation observation = Observation.createNotStarted("error.op", registry)
                .start();
        observation.error(new RuntimeException("connection failed"));
        observation.stop();

        // Assert
        SimpleSpan span = tracer.onlySpan();
        assertThat(span.getError()).hasMessage("connection failed");
    }

    @Test
    void shouldRecordEvent_WhenEventFiredDuringObservation() {
        // Act
        Observation observation = Observation.createNotStarted("event.op", registry)
                .start();
        observation.event(Observation.Event.of("cache.miss"));
        observation.stop();

        // Assert
        SimpleSpan span = tracer.onlySpan();
        assertThat(span.getEvents())
                .extracting(Map.Entry::getValue)
                .contains("cache.miss");
    }

    @Test
    void shouldCombineAllLifecycleEvents_WhenFullObservation() {
        // Act — full lifecycle: start, scope, event, error, stop
        Observation observation = Observation.createNotStarted("full.op", registry)
                .lowCardinalityKeyValue("method", "POST")
                .highCardinalityKeyValue(KeyValue.of("request.id", "abc-123"))
                .start();

        try (Observation.Scope scope = observation.openScope()) {
            observation.event(Observation.Event.of("payload.parsed"));
            observation.contextualName("POST /api/orders");
            observation.error(new IllegalStateException("validation failed"));
        }

        observation.stop();

        // Assert
        SimpleSpan span = tracer.onlySpan();
        assertThat(span.getName()).isEqualTo("POST /api/orders");
        assertThat(span.getTags()).containsEntry("method", "POST");
        assertThat(span.getTags()).containsEntry("request.id", "abc-123");
        assertThat(span.getEvents())
                .extracting(Map.Entry::getValue)
                .contains("payload.parsed");
        assertThat(span.getError()).hasMessage("validation failed");
    }

}
```

---

## Summary

| Concept | What It Means |
|---------|---------------|
| **`TracingObservationHandler<T>`** | Interface that bridges observation lifecycle events to tracing operations — the single abstraction that connects the two worlds |
| **`TracingContext`** | Inner class holding span + per-thread scope, stored in the observation's generic map via `computeIfAbsent` |
| **`DefaultTracingObservationHandler`** | Concrete handler for local spans — only implements `onStart` (create span) and `onStop` (name + tag + end) |
| **`getParentSpan()`** | Resolution chain: manual TracingContext → parent observation → thread-local current span |
| **`tagSpan()`** | Copies observation key-values to span tags, with special handling for "error" key |
| **`getSpanName()`** | Prefers contextualName ("GET /users") over base name ("http.server.requests") |

**Next: Chapter 9 — Propagating Observation Handlers** — Specialized handlers for cross-process communication: `PropagatingSenderTracingObservationHandler` injects trace context into outgoing carriers, `PropagatingReceiverTracingObservationHandler` extracts it from incoming carriers, enabling automatic distributed trace propagation across service boundaries.
